# Lesson 17 — `node:http` with no framework

> **Why this lesson exists:** Express and Fastify are thin layers over `node:http`. The settings that cause incidents (timeouts, keep-alive, body limits, disconnect handling) belong to Node, not to the framework, and they're behind the intermittent load-balancer 502s, the slowloris exposure and the work that keeps running after the client has gone. Build one server with nothing but `node:http` and every framework afterwards becomes a set of conveniences you can see through.

**Time:** ~85 minutes · **Prereq:** Lessons 13, 16

---

## 1. The idea in one sentence

> **An HTTP server in Node is a TCP socket, a C parser and two streams: your handler runs the moment the headers arrive, before the body exists, and every limit on size, time and liveness is yours to enforce on those streams.**

---

## 2. Why build one by hand

Four production incidents, none of them caused by a framework:

| Symptom | Real cause | Where it lives |
|---|---|---|
| 502s from the load balancer, only on quiet endpoints | Node closed an idle keep-alive connection just as the LB reused it | `server.keepAliveTimeout` |
| Memory spikes on one endpoint, then OOMKilled | A body read into memory with no limit | your body reader |
| A few hundred sockets open that never send a request | Slowloris, or a client bug, with the timeouts disabled | `server.headersTimeout` / `requestTimeout` |
| Database load from requests nobody is waiting for | The client disconnected and the handler kept going | `res.on("close")` |

Every one is configured on the `http.Server` object, and **Express's `app.listen()` returns exactly that object.** So nothing here is academic for Ledger: the Express API sets these same properties on the server `app.listen()` hands back.

---

## 3. The anatomy of a request

```text
  TCP socket (net.Socket, L16)
        │  bytes
        ▼
  ┌──────────────┐  headers complete   ┌────────────────────────────────────────┐
  │ llhttp       │ ──────────────────▶ │ 'request' event → your handler(req, res)│
  │ (C parser)   │                     │  req: IncomingMessage  (Readable)       │
  └──────┬───────┘                     │  res: ServerResponse   (Writable)       │
         │ body bytes, later           └────────────────────────────────────────┘
         └──────────────▶ req.push(chunk) … req.push(null)
  res.write()/res.end() ──▶ headers + body onto the same socket
  after res.end(): the socket waits for the next request (keep-alive)
```

```ts
import { createServer } from "node:http";

const server = createServer((req, res) => {
  // req: headers parsed, body NOT read yet. res: nothing sent yet.
  res.writeHead(200, { "content-type": "text/plain; charset=utf-8" });
  res.end("ok\n");
});
server.listen(3000);
```

**The handler runs before the body has arrived.** A 50 MB upload is still on the wire when your code starts. That single fact is why body limits must be enforced *while streaming*, not after.

### What Node does to headers

| You read | What you get |
|---|---|
| `req.headers` | Keys **lowercased**. Duplicates: `set-cookie` becomes an array; a fixed list (`content-type`, `content-length`, `authorization`, `host`, `user-agent` and a few more) **keeps the first and drops the rest**; everything else is joined with `", "` |
| `req.rawHeaders` | A flat `[name, value, name, value, …]` array in original case and order. The only faithful view |
| `req.headersDistinct` | Every header as an array, no joining or dropping |
| `req.url` | Path plus query, **raw and undecoded**. Not a URL object |

The joining rule bites with `x-forwarded-for`: two proxies' values arrive as one comma-separated string, and which entry you trust is a security decision ([API L19](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/19-rate-limiting.md)).

### Writing a response, and how it's framed

`res.setHeader()` stages a header; `res.writeHead()` merges the staged ones and commits the status line. Nothing hits the socket until the first body write or `end()`. After that, `setHeader` throws `ERR_HTTP_HEADERS_SENT`, which is the classic "I tried to send an error after streaming started" bug.

The client has to know where the body ends ([L16](16-tcp-and-net.md): TCP has no boundaries). Node picks the framing for you:

| You do | Node sends |
|---|---|
| `res.end(body)` with nothing written before | `Content-Length` computed for you: it has the whole body |
| `res.write(a); res.write(b); res.end()` with no `Content-Length` | `Transfer-Encoding: chunked` (each write becomes a sized chunk; `0\r\n\r\n` ends it) |
| You set `content-length` yourself | Exactly that: compute it with `Buffer.byteLength`, never `str.length` ([L12](../03-data-and-io/12-buffers-and-encodings.md)) |

```text
$ curl.exe --raw -s localhost:3000/stream
6           ← chunk size in hex
hello       ← 6 bytes, including the newline

0           ← terminator: the response is complete
```

**`res.end()` is not optional.** A handler that forgets it leaves the client waiting and the socket busy until someone's timeout fires. And `res.write()` returning `false` is backpressure from the client's socket, exactly as in [L13](../03-data-and-io/13-streams-and-backpressure.md).

---

## 4. Reading a body safely

The naive version is in a lot of production code:

```ts
// ❌ Unbounded: a 2 GB upload becomes 2 GB of Buffers, then one giant JSON.parse that blocks the loop
const chunks: Buffer[] = [];
for await (const chunk of req) chunks.push(chunk);
const body = JSON.parse(Buffer.concat(chunks).toString());   // 💥 a bad body throws inside your handler
```

Three defences, in order of cost:

1. **Refuse on `Content-Length`** before reading a byte.
2. **Count while streaming.** Chunked uploads carry no length, so the header alone proves nothing.
3. **Parse in a `try`/`catch`** and answer 400; check `content-type` and answer 415.

```ts
// http-raw/body.ts
import type { IncomingMessage } from "node:http";
import { HttpError } from "./errors.ts";

const tooLarge = (limit: number) =>
  new HttpError(413, "payload_too_large", `Body exceeds ${limit} bytes`);

export function readBody(req: IncomingMessage, opts: { limit: number; signal: AbortSignal }): Promise<Buffer> {
  const declared = Number(req.headers["content-length"]);        // NaN when absent → check skipped
  if (declared > opts.limit) return Promise.reject(tooLarge(opts.limit));   // 1. before reading anything

  return new Promise((resolve, reject) => {
    const chunks: Buffer[] = [];
    let size = 0;
    const finish = (err?: Error) => {
      req.off("data", onData).off("end", onEnd).off("error", finish);
      opts.signal.removeEventListener("abort", onAbort);
      if (err) { req.pause(); reject(err); }                      // stop pulling; TCP backpressure stalls the sender
      else resolve(Buffer.concat(chunks, size));
    };
    const onData = (chunk: Buffer) => {
      size += chunk.length;
      if (size > opts.limit) return finish(tooLarge(opts.limit)); // 2. counted while streaming
      chunks.push(chunk);
    };
    const onEnd = () => finish();
    const onAbort = () => finish(opts.signal.reason as Error);
    if (opts.signal.aborted) return onAbort();
    opts.signal.addEventListener("abort", onAbort, { once: true });
    req.on("data", onData).on("end", onEnd).on("error", finish);  // 'error' fires if the client drops mid-upload
  });
}

export function parseJson(raw: Buffer, contentType: string | undefined): unknown {
  if (!contentType?.toLowerCase().startsWith("application/json"))
    throw new HttpError(415, "unsupported_media_type", "Send application/json");
  try { return JSON.parse(raw.toString("utf8")); }                // 3. bad JSON is the client's fault: 400
  catch { throw new HttpError(400, "invalid_json", "Body is not valid JSON"); }
}
```

The return type is `unknown` on purpose: the next step is a schema parse ([TS L15](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/15-the-boundary-and-parsing.md)).

**How to reject well.** Answer 413 with `Connection: close` and stop reading (`req.pause()` above). Don't `req.destroy()` first: that kills the socket and the client never sees the 413. Even so, many clients only read the response after they finish uploading, so a large rejected upload can still surface on their side as a write error. HTTP has an answer for that, `Expect: 100-continue`, and Node supports it through the server's `'checkContinue'` event.

The 1 MB limit isn't tidiness. A body you never buffer can't exhaust memory, and a body you never parse can't block the loop ([L08](../02-event-loop/08-blocking-the-loop.md)). **Body limits are availability controls.**

---

## 5. Routing by hand

```ts
// http-raw/router.ts
import type { Handler } from "./types.ts";

type Route = { method: string; segments: string[]; handler: Handler };
const routes: Route[] = [];

export function route(method: string, pattern: string, handler: Handler): void {
  routes.push({ method, segments: pattern.split("/").filter(Boolean), handler });
}

export function match(method: string, path: string) {
  const parts = path.split("/").filter(Boolean);
  const allowed: string[] = [];
  for (const r of routes) {                                       // linear scan, first match wins
    if (r.segments.length !== parts.length) continue;
    const params: Record<string, string> = {};
    let ok = true;
    for (let i = 0; i < parts.length && ok; i++) {
      const seg = r.segments[i]!, part = parts[i]!;
      if (seg.startsWith(":")) params[seg.slice(1)] = decodeURIComponent(part);   // throws URIError on bad %-escapes → 400
      else ok = seg === part;
    }
    if (!ok) continue;
    if (r.method === method) return { handler: r.handler, params, allowed };
    allowed.push(r.method);                                       // path exists, method doesn't → 405 + Allow
  }
  return { handler: undefined, params: {}, allowed };
}
```

Two details frameworks handle for you, and that you now know to check:

- **404 vs 405.** A path that exists with the wrong method is `405 Method Not Allowed` with an `Allow` header, not 404.
- **Parse the URL with a fixed base:** `new URL(req.url ?? "/", "http://internal")`. The `Host` header is client-controlled; never build absolute URLs (redirects, emailed links) from it.

This is Express's router with the conveniences removed: a list scanned in order. [L19](19-frameworks-by-mechanism.md) shows what that linear scan costs and what Fastify's radix tree does instead.

---

## 6. Server timeouts: what each one defends

| Setting | Default | Clock runs from → to | Defends against | On expiry |
|---|---|---|---|---|
| `headersTimeout` | 60 000 ms | first byte of a request → headers complete | **Slowloris**: headers dribbled one byte at a time | 408, connection closed |
| `requestTimeout` | 300 000 ms | first byte → the whole request (headers + body) received | **Slow bodies**: a POST trickled for hours | 408, connection closed |
| `keepAliveTimeout` | 5 000 ms | response finished → next request's first byte | **Idle sockets** hoarding descriptors | socket closed silently |
| `connectionsCheckingInterval` | 30 000 ms | — | How often the first two are checked | — |

Three things people get wrong:

1. **`requestTimeout` is about receiving the request, not sending the response.** A two-minute export or an hour-long SSE stream ([L20](20-realtime-servers.md)) isn't affected by it.
2. **The first two are enforced by a periodic sweep**, so they're coarse: a 60 s `headersTimeout` fires somewhere between 60 and 90 s with the default interval.
3. **Never set them to 0 to "fix" long uploads.** Raise them for the upload route's needs. Zero means a client that never finishes its headers holds a socket forever.

**Slowloris, mechanically.** The attacker opens thousands of connections and sends a partial header line every few seconds. Each connection is legal, so nothing errors. A thread-per-connection server dies at its thread limit; Node doesn't run out of threads, but without `headersTimeout` it keeps every socket (one descriptor plus buffers each) until it hits `ulimit -n` ([L15](../03-data-and-io/15-the-file-system.md)) and stops accepting real users. Behind an L7 load balancer, header-dribbling hits the balancer first, because it has to read the headers before it can route; Node's timeouts are your defence when you're exposed directly or behind an L4 balancer, and defence in depth otherwise.

---

## 7. The load-balancer 502: a race you configure

Both ends of a keep-alive connection can close it when it's idle. The LB keeps idle upstream connections in a pool for up to its idle timeout (ALB: 60 s by default). Node closes an idle connection after `keepAliveTimeout` (5 s). So Node is always the one that closes first, and sometimes it closes at the exact moment the LB picks that connection for the next request:

```text
 ALB (idle timeout 60 s)                         Node (keepAliveTimeout 5 s)
   │── request 1 ───────────────────────────────────▶│
   │◀────────────────────────────────── response 1 ──│
   │      connection idle, back in the ALB's pool    │
   │                                                 │ t = 5.000 s  idle timer fires → FIN
   │── request 2 (t = 5.001 s, reuses the conn) ────▶│ ✗ socket already closed → RST
   │      FIN and request 2 crossed on the wire
   ▼
 ALB: "upstream reset before response" → 502 to the client
```

**Why only quiet endpoints:** a busy connection never sits idle for 5 s, so it never gets closed. Low-traffic routes reuse connections at random intervals, and some land in the window.

**The fix is an ordering, not a value: the side that reuses the connection must be the side that gives up on it first.**

```ts
const server = app.listen(3000);      // Express returns the http.Server
server.keepAliveTimeout = 65_000;     // > ALB idle timeout (60 s): the ALB always closes first
server.headersTimeout   = 66_000;     // > keepAliveTimeout
server.requestTimeout   = 120_000;    // ≥ headersTimeout
```

Why `headersTimeout` above `keepAliveTimeout`? On older Node lines the headers timer could fire on an idle keep-alive connection, which silently capped your 65 s keep-alive at the headers timeout and brought the 502s back. Modern Node measures it per request, but the ordering costs nothing and is correct on every line.

The rule as a production checklist item is in [API L18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md); the race above is why it exists. The same race exists for your outbound calls, with you as the reusing side, which is why undici's client idle timeout sits below Node's server default ([L18](18-outbound-http.md)).

> **Spring equivalent:** Tomcat's connector has the same knobs: `server.tomcat.connection-timeout` (how long to wait for the request after accepting), `server.tomcat.keep-alive-timeout` and the max header size. Same race, same ordering rule. [Spring L14](https://github.com/Pawar-Pratik/SpringBoot/blob/main/05-web/14-servlet-and-dispatcherservlet.md)

---

## 8. Client disconnects → an AbortSignal

A client that gives up (a timeout, a closed tab, a cancelled `fetch`) doesn't stop your handler. It keeps querying, calling the processor and serialising a response into a dead socket. Detect it and turn it into a signal ([L10](../02-event-loop/10-cancellation-and-concurrency.md)):

```ts
// http-raw/signal.ts
import type { ServerResponse } from "node:http";

export function requestSignal(res: ServerResponse, timeoutMs: number): AbortSignal {
  const gone = new AbortController();
  res.once("close", () => {
    if (!res.writableFinished) gone.abort(new Error("client disconnected"));   // closed before we finished
  });
  return AbortSignal.any([gone.signal, AbortSignal.timeout(timeoutMs)]);
}
```

**Listen on `res`, not `req`.** `req`'s `'close'` means the *request stream* is done, which can happen as soon as its body has been consumed, long before the client leaves. `res` closing while `writableFinished` is still `false` is the unambiguous "the response never completed" signal. Then pass the signal into every downstream call and the body reader.

---

## 9. Limits you don't write: header size, llhttp and smuggling

- **`maxHeaderSize`** defaults to 16 KiB; above it Node answers `431 Request Header Fields Too Large`. You can raise it (`--max-http-header-size`, or the `createServer` option), but the usual cause is a bloated cookie or JWT. Fix that instead.
- **Why you never hand-parse HTTP.** You framed messages on raw TCP in [L16](16-tcp-and-net.md); HTTP/1.1 framing is the same problem with decades of edge cases. **Request smuggling** happens when two parsers disagree about where a request ends (`Content-Length` vs `Transfer-Encoding`, duplicate lengths, malformed chunk sizes). The front proxy sees one request, the backend sees two, and the attacker's second request is glued onto the next user's connection. llhttp rejects ambiguous framing, and Node has shipped several security releases tightening exactly this. `--insecure-http-parser` turns the strictness off. **Never in production.**
- **`'clientError'`** fires when parsing fails. The default handler answers 400 and destroys the socket; add a listener only to log (rate-limited), and still destroy the socket.

### HTTP/2 and `server.close()`, briefly

`node:http2` exists and works. Usually you don't need it on the Node side: the load balancer speaks HTTP/2 (or 3) to browsers, which removes the six-connection limit where it matters ([API L02](https://github.com/Pawar-Pratik/API-mastery/blob/main/01-foundations/02-journey-of-a-request.md)), and talks HTTP/1.1 over pooled keep-alive connections to your pods. Serve HTTP/2 from Node when there's no L7 balancer in front, or for gRPC.

`server.close()` stops accepting connections, lets in-flight requests finish, and (since Node 19) also closes idle keep-alive connections; `closeIdleConnections()` and `closeAllConnections()` exist since 18.2. The ordered drain that loses zero requests is [L24](../05-concurrency/24-graceful-shutdown.md).

---

## 10. Production rules

| Rule | Why |
|---|---|
| **`keepAliveTimeout` > the load balancer's idle timeout** | The reusing side must close first, or idle reuse races the FIN and you get 502s |
| **`headersTimeout` > `keepAliveTimeout`, and `requestTimeout` ≥ `headersTimeout`** | Keeps the keep-alive window intact on every Node line |
| **Never set `headersTimeout` or `requestTimeout` to 0** | One slow client per socket, forever: slowloris with no defence |
| **Check `Content-Length`, then count bytes while streaming** | Chunked bodies have no length; the header alone proves nothing |
| **Answer 413 with `Connection: close`; pause, don't destroy** | The client can only see the 413 if the socket survives long enough to send it |
| **`JSON.parse` inside `try`/`catch` → 400; wrong `content-type` → 415** | A malformed body is the client's error, not a 500 and not a crash |
| **Parse `req.url` against a fixed base; never trust `Host` to build URLs** | `Host` is attacker-controlled (host-header injection) |
| **Catch the handler's promise at the top level and always end the response** | A rejection escaping the request listener is an unhandled rejection: exit code 1 |
| **Turn `res` `'close'` + `!writableFinished` into an `AbortSignal`** | Work for disconnected clients is pure waste under load |
| **`Content-Length` from `Buffer.byteLength`** | `str.length` counts UTF-16 units, not bytes |
| **If headers are already sent when an error happens, destroy the socket** | A truncated chunked body is detectable; a "successful" half-response is not |
| **Never run with `--insecure-http-parser`** | It re-opens request smuggling |
| **Terminate HTTP/2 at the load balancer unless you have a reason not to** | The browser connection limit is solved at the edge; HTTP/1.1 keep-alive is fine inside |

---

## 11. Interview traps

**Q1. "We get intermittent 502s from the ALB, only on idle connections. Why?"**
Keep-alive timeouts in the wrong order. The ALB keeps idle upstream connections for 60 s; Node closes them after 5 s. When the ALB reuses a connection at the instant Node's FIN is in flight, Node answers with a reset and the ALB returns 502. It only shows on quiet endpoints because busy connections never sit idle long enough to be closed. Fix: `keepAliveTimeout` above the LB idle timeout (65 s for an ALB) and `headersTimeout` above that. A weak answer says "increase the timeout" without naming which side must close first.
**Follow-up:** "How would you prove it?" Correlate the 502s with the ALB's backend-connection-reset metric or access-log reason, and show they're absent on hot endpoints; then reproduce with a proxy that reuses connections at exactly the server's keep-alive interval.

**Q2. "What do `keepAliveTimeout`, `headersTimeout` and `requestTimeout` each do?"**
`keepAliveTimeout` bounds the idle gap between requests on one connection. `headersTimeout` bounds the time from a request's first byte to complete headers (slowloris). `requestTimeout` bounds the time to receive the whole request including the body (slow uploads). None of them limits how long your *response* takes. The first two are checked by a sweep every `connectionsCheckingInterval`, so they're coarse.

**Q3. "How do you limit request body size without buffering it?"**
Reject on `Content-Length` before reading, then count bytes as they stream in and stop at the limit, because chunked bodies have no length. Answer 413 with `Connection: close` and stop reading instead of destroying the socket. The strong addition: the limit protects the event loop too, since you never parse what you never accepted.

**Q4. "How do you detect that a client disconnected?"**
`res.on("close")` with `res.writableFinished === false`. Then turn it into an `AbortSignal`, combined with a deadline via `AbortSignal.any`, and pass it to every downstream call. The common weak answer is `req.on("close")`, which fires when the request stream finishes, not when the client goes away.

**Q5. "Why not parse HTTP yourself? It's text."**
Because framing is the hard part, and disagreement about framing is an attack. Request smuggling exploits two parsers that disagree on where a request ends. llhttp is strict about ambiguous `Content-Length`/`Transfer-Encoding` combinations and has been hardened through security releases; a hand-written parser behind a proxy is a smuggling vulnerability waiting for a researcher.

**Q6. "What is slowloris, and what in Node stops it?"**
Many connections, each sending headers so slowly that none completes. It exhausts connection slots (threads in thread-per-connection servers; descriptors and memory in Node). `headersTimeout` closes each one with a 408, and `requestTimeout` is the backstop. An L7 load balancer absorbs most of it in practice, since it must read headers before routing.

**Q7. "When does Node send `Content-Length` and when does it use chunked encoding?"**
If you call `res.end(body)` without writing first, Node knows the full body and sets `Content-Length`. If you `write()` before `end()` without setting a length, HTTP/1.1 responses go out chunked, which is how streaming responses work. If you set `Content-Length` yourself, compute it with `Buffer.byteLength`.

**Q8. "Your handler throws after it has already written part of the response. What does the client see?"**
The status and headers are gone (200 already sent), so you can't send a 500. The right move is `res.destroy()`: a chunked response without its terminating zero-length chunk, or a body shorter than its `Content-Length`, is detectable by the client as a failed response. "Finishing" it with `res.end()` turns a failure into a truncated success, which is far worse for an export or a payment list.

**Q9. "Two `Authorization` headers arrive. What does `req.headers.authorization` contain?"**
The first one; Node drops duplicates for a fixed set of headers where joining makes no sense. `set-cookie` becomes an array, most others are joined with `", "`. `rawHeaders` and `headersDistinct` show what was really sent, which matters when proxies append to headers like `x-forwarded-for`.

**Q10. "Should your Node service speak HTTP/2?"**
Usually not directly. The load balancer gives browsers HTTP/2 or 3 (which removes the per-origin connection limit) and uses pooled HTTP/1.1 keep-alive to the pods, where multiplexing buys little. Serve it from Node when there's no L7 balancer, or for gRPC.

---

## 12. Build & break

### Build — `http-raw/`, a framework-free mini-Ledger

```text
http-raw/
├── server.ts     createServer, tuned timeouts, top-level catch, listen
├── router.ts     route(), match() with params, 404/405 + Allow
├── body.ts       readBody(req, { limit, signal }), parseJson()
├── signal.ts     requestSignal(res, timeoutMs)
├── errors.ts     HttpError + sendProblem(res, err): application/problem+json, request_id
├── types.ts      Ctx { req, res, url, params, signal } and Handler = (ctx: Ctx) => Promise<void>
└── routes.ts     /healthz, /readyz, POST /v1/payments
```

Spec:
1. **Timeouts:** `keepAliveTimeout` 65 s, `headersTimeout` 66 s, `requestTimeout` 120 s, set on the server object.
2. **Top level:** the request listener calls `handle(req, res).catch(err => sendProblem(res, err))`. `sendProblem` destroys the socket if `res.headersSent`, otherwise writes RFC 9457 with a `request_id` ([API L10](https://github.com/Pawar-Pratik/API-mastery/blob/main/02-rest-design/10-errors-and-problem-details.md)). `URIError` from the router maps to 400.
3. **`/healthz`** returns 200 with no dependency checks; **`/readyz`** checks Postgres with a 1 s deadline ([API L20](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/20-observability.md) for the split).
4. **`POST /v1/payments`:** `requestSignal(res, 10_000)` → `readBody` with a 1 MB limit → keep the raw `Buffer` (the HMAC needs those exact bytes, [L12](../03-data-and-io/12-buffers-and-encodings.md), and the idempotency layer hashes the request, [API L18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md)) → `parseJson` → schema parse → a fake `createPayment(input, signal)` that awaits `setTimeout(300, undefined, { signal })` from `node:timers/promises` → 201 with `Location`.
5. **Responses** set `content-length` from `Buffer.byteLength` and `content-type: application/json; charset=utf-8`.

### Break — five experiments

1. **Slowloris.** Write `slowloris.ts` with `node:net`: open 500 sockets, send `GET / HTTP/1.1\r\nHost: localhost\r\n`, then `X-a: b\r\n` every 10 s. In the server, log `server.getConnections()` every 5 s. With defaults, the count holds near 500 and then collapses somewhere between 60 and 90 s as each socket gets a 408 (print what the sockets receive). Now set `server.headersTimeout = 0`: they survive until `requestTimeout` cleans up at ~300 s. Set both to 0: they never die. Restore the defaults. Note that `keepAliveTimeout` never helped: it only governs the gap *between* requests.
2. **The 502 race.** Server: `keepAliveTimeout = 1000`. Proxy (`proxy.ts`): `http.createServer` forwarding with `http.request`, answering 502 when the upstream request errors. Recent Node `http.Agent`s read the server's `Keep-Alive: timeout=` hint and retire sockets early, which makes them too polite to race, so behave like a load balancer instead: subclass `http.Agent` with `keepAlive: true` and override `keepSocketAlive(socket)` with the documented minimal behaviour (`socket.setKeepAlive(true); socket.unref(); return true;`). Its pooled sockets now never expire, an idle timeout effectively longer than the server's. Client: a few hundred sequential requests through the proxy, sleeping `1000 ± 5` ms between them. Count the 502s and log the upstream error code (`ECONNRESET`). The window is small on localhost, so if you see none, narrow the jitter; that smallness is exactly why the bug is intermittent in production. Then set the server's `keepAliveTimeout = 65_000` and rerun: zero 502s.
3. **The unbounded body.** Add a `/naive` route using the `for await` + `Buffer.concat` version, and log `process.memoryUsage().rss` every second. Create a test file in Git Bash with `head -c 50000000 /dev/urandom > big.bin` and send it with `curl.exe -X POST --data-binary @big.bin localhost:3000/naive`. Watch RSS jump; send five in parallel and watch it multiply. Then hit `/v1/payments`: an immediate 413 from the `Content-Length` check. Add `-H "Transfer-Encoding: chunked"` so there's no length header, and watch the streaming counter trip at 1 MB instead.
4. **Work for nobody.** Temporarily drop the `signal` from `createPayment` and raise its delay to 5 s, logging "payment created" at the end. Call it with `curl.exe --max-time 1 ...`: the client gives up at 1 s and the server logs "payment created" at 5 s. Restore the signal: the handler aborts at ~1 s and nothing is created.
5. **The header limit.** `curl.exe -i -H "X-Big: $(head -c 20000 /dev/zero | tr '\0' a)" localhost:3000/healthz` returns 431 without your handler ever running. Note that the parser rejected it, not your code.

### Explain out loud (60–120 seconds)

1. The anatomy: socket → llhttp → `req`/`res` streams, and why the handler runs before the body exists.
2. The three server timeouts, what each defends, and that none limits the response.
3. The 502 race, drawn: who closes, who reuses, and the ordering fix.
4. How you read a body with a limit, and how you reject it so the client sees the 413.
5. How a client disconnect becomes an `AbortSignal`, and why you listen on `res`.

---

## What's next

You can now serve HTTP without a framework and know which knobs are Node's. The other half of networking is outbound: every webhook Ledger sends, every processor call. Next comes connection pools, the timeouts an outbound call needs, the unconsumed-body leak, and an SSRF guard that holds up against DNS rebinding.

Next → **[Lesson 18: Outbound HTTP: fetch, undici & connection pools](18-outbound-http.md)**
