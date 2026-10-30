# Lesson 18 — Outbound HTTP: fetch, undici & connection pools

> **Why this lesson exists:** much of production latency, and a fair share of outages, comes from calls you *make*: no timeout on a slow dependency, a new connection for every request, a response body nobody read that quietly pins a pooled socket, and a user-supplied URL that points at your cloud metadata service. Ledger's webhook dispatcher hits every one of these at once, because it sends payment data to thousands of URLs that merchants typed in.

**Time:** ~75 minutes · **Prereq:** Lessons 10, 17

---

## 1. The idea in one sentence

> **An outbound request borrows a pooled connection and a place in a queue, and it must give both back, by consuming its body and by finishing before a deadline, or your own pool becomes the outage.**

---

## 2. Why outbound is where things break

The remote server is rarely the whole story. The failures that page you are on your side of the socket:

| Failure | What you see | Section |
|---|---|---|
| No deadline on a slow dependency | Requests pile up, memory climbs, then every endpoint times out | §5 |
| A new connection per request | Latency triples; `EADDRNOTAVAIL` at volume; DNS saturating the thread pool | §3, §7 |
| An error path that never reads the body | Requests start hanging after N failures, intermittently | §6 |
| No per-origin cap | One slow merchant holds hundreds of sockets | §4 |
| A user-supplied URL fetched as-is | Your server fetches `169.254.169.254` for an attacker | §9 |

The protocol side (round trips per handshake, why reuse matters) is in [API L02](https://github.com/Pawar-Pratik/API-mastery/blob/main/01-foundations/02-journey-of-a-request.md). This lesson is what Node does underneath.

---

## 3. The client stack: fetch, undici and dispatchers

Node's global `fetch` is **undici**, bundled into Node. The `undici` npm package is the same library with its full API: `request()` (lower overhead, Node-only, the body is a Node `Readable`) and the dispatcher classes that do the pooling.

```text
 fetch(url, { dispatcher })          undici.request(url, { dispatcher })
          └────────────────┬──────────────────────┘
                     Dispatcher   (if you pass none: the global dispatcher, an Agent)
                           │ one Pool per origin (scheme + host + port)
          ┌────────────────┼────────────────┐
     Pool(a.example)  Pool(b.example)  Pool(c.example)
      ├ Client ─ one keep-alive socket          `connections` caps the Clients per Pool
      ├ Client ─ one keep-alive socket
      └ queue: requests waiting for a free Client   ← in memory, and their deadlines are ticking
```

| Class | What it is | You use it when |
|---|---|---|
| `Client` | One origin, one connection | Rarely directly |
| `Pool` | One origin, many `Client`s | A single dependency you call a lot (the processor) |
| `Agent` | A `Pool` per origin, created on demand | Many origins: webhooks, the default |
| global dispatcher | The `Agent` that `fetch` uses when you pass none | Set once at boot with `setGlobalDispatcher`, if at all |

Which API to use:

- **`fetch`**: the standard. Portable code, and the `@ledger/node` SDK uses it so it runs anywhere ([L32](../08-shipping/32-packaging-and-publishing.md)).
- **`undici.request`**: inside a Node service where you want the lower overhead, `statusCode` and a `Readable` body. Ledger's webhook sender uses it.
- **`http.request` + `http.Agent`**: the classic module. Libraries built on it (axios uses it on Node) take an `agent` option (§7).

If you install `undici`, use its `fetch`/`request` with its own `Agent`. Mixing an npm `Agent` with the bundled global `fetch` only works while the two versions happen to be compatible.

### What keep-alive saves you, in Node terms

Reusing a connection skips more than round trips. **Each new connection costs a `dns.lookup` on the libuv thread pool** ([L07](../02-event-loop/07-the-thread-pool.md)), a TCP handshake, a TLS handshake (real CPU in your process) and, when it eventually closes, a port in `TIME_WAIT` (§7). An `Agent` you keep for the life of the process amortises all of that across thousands of requests. **One dispatcher per purpose, created at boot. Never one per request, never a `Map` of them per merchant** ([L27](../07-production/27-memory-and-gc.md) hunts exactly that leak).

---

## 4. Connections per origin, the queue, and pipelining

An `Agent`'s per-origin `connections` limit is **unlimited by default**. That's fine for one fast dependency and dangerous for webhooks: a slow merchant plus a burst of events means hundreds of sockets to one endpoint, each a descriptor and buffers in your process, and a flood to a receiver that's already struggling.

Cap it, and understand what the cap does: **excess requests wait in the pool's in-memory queue**. That queue has no bound, and a request waiting in it is spending its deadline doing nothing. So pair the dispatcher cap with a concurrency limit *in front of it* ([L10](../02-event-loop/10-cancellation-and-concurrency.md)'s `keyedLimit`), sized the same, so requests wait in your limiter where you can measure the queue instead of invisibly inside undici.

**Pipelining** (`pipelining: 1`, the default, means one request at a time per connection). HTTP/1.1 pipelining sends several requests before the first response returns. Responses must come back in order, so one slow response blocks every request behind it on that connection (head-of-line blocking); a connection that dies mid-pipeline leaves you unsure which requests ran, which is unsafe for anything non-idempotent; and plenty of servers and proxies handle it badly. Leave it at 1 unless you control the server and the requests are idempotent. HTTP/2 multiplexing is the protocol's real answer to the same problem.

---

## 5. Timeouts: every outbound call gets a deadline

| Timeout | undici option | Default | Bounds | Doesn't bound |
|---|---|---|---|---|
| Connect | `connect: { timeout }` | 10 s | TCP + TLS handshake | anything after |
| Headers | `headersTimeout` | 300 s | request sent → complete response headers | time queued in the pool |
| Body | `bodyTimeout` | 300 s | **the gap between body chunks** | total body time |
| Total | `signal: AbortSignal.timeout(ms)` | **none** | everything, including the queue | — |

Two facts decide the design:

1. **`fetch` has no timeout option at all.** Without a `signal` it's bounded only by those minutes-long defaults.
2. **`bodyTimeout` is an idle timeout.** A server that sends one byte every 4 s never trips a 5 s `bodyTimeout`, and holds your connection for as long as it likes. Spring has the same gap: a read timeout is per read, not per request ([Spring L34](https://github.com/Pawar-Pratik/SpringBoot/blob/main/09-production/34-resilience-and-integration.md)). **Only a total deadline bounds a call.**

```ts
// lib/http-client.ts
import { Agent, request } from "undici";
import { safeLookup } from "./ssrf.ts";

export type Failure =
  | { kind: "timeout"; phase: "connect" | "headers" | "body" | "deadline" }
  | { kind: "network" | "blocked"; code: string }
  | { kind: "aborted" };
export type Outcome =
  | { kind: "response"; status: number; retryAfterMs: number | undefined;
      body: { text: string; truncated: boolean }; ms: number }
  | (Failure & { ms: number });

export const webhookAgent = new Agent({
  connections: 5,                    // per origin: one merchant can't take 500 sockets
  keepAliveTimeout: 4_000,           // our idle limit stays BELOW typical server idle (Node's is 5 s): L17's race, mirrored
  headersTimeout: 8_000,
  bodyTimeout: 5_000,
  connect: { timeout: 3_000, lookup: safeLookup },   // §9: every connection resolves through the SSRF guard
});

export async function postCapped(
  url: string,
  init: { headers: Record<string, string>; body: string },
  opts: { deadlineMs: number; captureBytes: number; signal?: AbortSignal },
): Promise<Outcome> {
  const deadline = AbortSignal.timeout(opts.deadlineMs);
  const signal = opts.signal ? AbortSignal.any([opts.signal, deadline]) : deadline;
  const started = performance.now();
  try {
    const res = await request(url, { method: "POST", ...init, dispatcher: webhookAgent, signal });
    const body = await readCapped(res.body, opts.captureBytes);   // §6: consumed on EVERY path
    return { kind: "response", status: res.statusCode, retryAfterMs: retryAfterMs(res.headers["retry-after"]),
             body, ms: Math.round(performance.now() - started) };
  } catch (err) {
    return { ...classify(err, deadline, opts.signal), ms: Math.round(performance.now() - started) };
  }
}
```

Deadlines compose downward: the dispatcher's own timeouts are tighter than the 10 s total, and the total is tighter than whatever is waiting on it ([API L18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md) on budgets). The undici `keepAliveTimeout` of 4 s is the client half of [L17](17-http-from-scratch.md)'s 502 race: now *you* are the side that reuses connections, so you must give up on them before the server does. undici also honours a server's `Keep-Alive: timeout=` hint, but not every server sends one.

---

## 6. The unconsumed-body leak, and reading only what you need

A response body is the rest of the bytes on that socket. **The connection can't go back to the pool until the body has been read to the end or cancelled**, because the next response's bytes come after it. Skip the body and the connection stays checked out.

```ts
// ❌ The error path never touches the body. Each failure pins one connection.
const res = await fetch(url, { signal });
if (!res.ok) throw new UpstreamError(res.status);
return await res.json();

// ✅ Consume or cancel on every path, including errors and the bodies you don't care about.
const res = await fetch(url, { signal });
if (!res.ok) {
  const detail = await res.text().catch(() => "");   // or: await res.body?.cancel()
  throw new UpstreamError(res.status, detail.slice(0, 500));
}
return await res.json();
```

With a cap of 5 connections, five unread error bodies exhaust the pool and every later request queues until its deadline. Without a cap, each one leaks a socket instead. For `fetch`, undici eventually cancels the bodies of responses that get garbage-collected, **which is what makes this bug intermittent**: it hangs under load and clears up after a GC. For `undici.request`, consume with `body.text()`/`body.json()` or discard with `body.dump()`.

Sometimes you want part of a body. Ledger stores at most 2 KB of a merchant's response, because the response is the evidence in their support ticket ([API L23](https://github.com/Pawar-Pratik/API-mastery/blob/main/05-beyond-rest/23-realtime-and-webhooks.md)), but a misbehaving endpoint might stream 5 GB:

```ts
export async function readCapped(body: AsyncIterable<Uint8Array>, cap: number) {
  const chunks: Uint8Array[] = [];
  let size = 0, truncated = false;
  for await (const chunk of body) {
    const room = cap - size;
    if (chunk.byteLength > room) {
      chunks.push(chunk.subarray(0, room)); size = cap; truncated = true;
      break;                                   // break destroys the stream: the socket is closed, not reused
    }
    chunks.push(chunk); size += chunk.byteLength;
  }
  return { text: new TextDecoder().decode(Buffer.concat(chunks, size)), truncated };  // a split UTF-8 tail → U+FFFD (L12)
}
```

**Stopping early costs the connection**: the unread remainder can't be skipped, so the socket is destroyed. That's the right trade for a rare oversized body, and the wrong one for a hot path, where you read to the end instead.

---

## 7. `http.Agent`, socket exhaustion and `TIME_WAIT`

On the classic `http`/`https` modules, the **global agent** has kept connections alive by default since Node 19. **An `Agent` you construct still defaults to `keepAlive: false`**, and its `maxSockets` per origin defaults to `Infinity`:

```ts
import { Agent } from "node:https";
export const processorAgent = new Agent({ keepAlive: true, maxSockets: 50, maxFreeSockets: 10 });
// pass it to http(s).request({ agent }), or to libraries built on them (axios: httpsAgent)
```

**`TIME_WAIT`.** The side that closes a TCP connection first keeps its end in `TIME_WAIT` for a while (60 s on Linux) so stray packets from the old connection can't corrupt a new one with the same addresses. Each such entry holds a local port. Outbound connections take an ephemeral port from a finite range, and a port can't be reused for the same destination while it's in `TIME_WAIT`:

```text
Linux default ephemeral range: 32768–60999  →  28,232 ports
TIME_WAIT on Linux: 60 s
Sustained NEW connections to one destination (ip:port) before exhaustion ≈ 28,232 / 60 ≈ 470 per second
Past that: connect() fails with EADDRNOTAVAIL, while CPU and the remote look fine
```

Windows' default dynamic range (49152–65535) is smaller still. A service that opens a fresh connection per request crosses 470/s against one dependency easily. Kernel knobs (`tcp_tw_reuse`) are a patch; **reusing connections is the fix**: one keep-alive connection carries thousands of requests and enters `TIME_WAIT` once.

**DNS, once more.** Every new connection runs `dns.lookup` (getaddrinfo) on the 4-thread pool, so thousands of short-lived connections to many merchant hostnames compete with `fs` and crypto ([L07](../02-event-loop/07-the-thread-pool.md)). Reuse cuts the lookups; beyond that, cache resolutions with a TTL in your `lookup` function, or use the DNS-caching interceptor that recent undici versions ship. `dns.resolve4/6` uses c-ares (no thread pool) and returns TTLs, but bypasses `/etc/hosts` and the OS resolver configuration, so choose it deliberately.

---

## 8. Retries: the Node mechanics

*Whether* to retry belongs to the idempotency design in [API L18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md), and it isn't repeated here. What Node adds is knowing, from the error, **whether the request could have reached the server**:

| Failure | Was the request sent? | Retry |
|---|---|---|
| `ECONNREFUSED`, `UND_ERR_CONNECT_TIMEOUT`, `EAI_AGAIN` | No: no connection existed | Yes, even for a `POST` |
| `ENOTFOUND` | No | Later; it may be permanent (a typo, a deleted record) |
| `ECONNRESET` / `UND_ERR_SOCKET` before any response | Maybe (a reused connection reset by the L17 race, or a crash mid-request) | Only with an idempotency key |
| `UND_ERR_HEADERS_TIMEOUT`, total deadline | Yes, outcome unknown | Only with an idempotency key |
| 429, 503 | Yes, refused | Yes, **after `Retry-After`** |
| Other 5xx | Yes | Only with an idempotency key |
| 4xx | Yes | Never |

```ts
export function retryAfterMs(h: string | string[] | undefined, now = Date.now()): number | undefined {
  const v = Array.isArray(h) ? h[0] : h;
  if (!v) return undefined;
  const secs = Number(v);
  if (Number.isFinite(secs)) return Math.max(0, secs * 1_000);   // "120"
  const at = Date.parse(v);                                         // "Wed, 21 Oct 2026 07:28:00 GMT"
  return Number.isNaN(at) ? undefined : Math.max(0, at - now);
}
```

Three mechanical rules: **cap `Retry-After`** (a hostile or buggy server can send a year); **never sleep past the deadline** (if the wait exceeds the remaining budget, fail now); and **spend from a retry budget** so retries can't multiply load during an incident (API L18 has the budget). Classify by *your own signal's* state, not by the error's shape, which differs between `fetch` (a `TypeError` with a `cause`) and `request`:

```ts
function errCode(e: unknown): string | undefined {
  let cur: unknown = e;
  for (let depth = 0; depth < 5 && cur instanceof Error; depth++) {
    const code = (cur as Error & { code?: unknown }).code;
    if (typeof code === "string") return code;
    cur = cur.cause;                                   // fetch wraps the real error: TypeError("fetch failed", { cause })
  }
  return undefined;
}

export function classify(err: unknown, deadline: AbortSignal, shutdown?: AbortSignal): Failure {
  if (shutdown?.aborted) return { kind: "aborted" };
  if (deadline.aborted) return { kind: "timeout", phase: "deadline" };
  const code = errCode(err) ?? "UNKNOWN";
  if (code === "UND_ERR_CONNECT_TIMEOUT") return { kind: "timeout", phase: "connect" };
  if (code === "UND_ERR_HEADERS_TIMEOUT") return { kind: "timeout", phase: "headers" };
  if (code === "UND_ERR_BODY_TIMEOUT")    return { kind: "timeout", phase: "body" };
  if (code === "ERR_SSRF_BLOCKED")        return { kind: "blocked", code };
  return { kind: "network", code };
}
```

For webhooks, the dispatcher doesn't retry in-process at all: each attempt is a row, and the next one is scheduled in `webhook_deliveries.next_attempt_at` ([L26](../06-data-and-jobs/26-background-jobs-and-queues.md)). `Retry-After` only stretches that schedule.

---

## 9. The SSRF guard: validate where the socket connects

The attack, the bypass list (decimal IPs, IPv6 forms, `nip.io`, redirects, rebinding) and the network-layer controls are in [API L16](https://github.com/Pawar-Pratik/API-mastery/blob/main/03-security/16-owasp-and-hardening.md). The Node question is *where* the check goes.

**Check-then-fetch is a race.** You resolve the hostname, see a public IP, approve it; then `fetch` resolves again to connect. An attacker who controls the DNS answers with TTL 0: public the first time, `10.0.0.5` the second. That's **DNS rebinding**: a time-of-check/time-of-use bug.

```text
❌ check, then fetch                           ✅ validate inside the connect
assertSafe(url): lookup → 93.184.x.x  ✓        request(url) → Agent → net.connect
request(url):    lookup → 10.0.0.5   (TTL 0)                        → safeLookup(host): resolve ONCE,
                 connects to 10.0.0.5  💥                              check EVERY address
                                                                     → connect to the address that was checked
```

undici passes `connect` options through to `net.connect`/`tls.connect`, which accept a `lookup` function. Put the check there, and the socket connects to exactly what you validated, while the URL, the `Host` header and TLS certificate verification all stay on the hostname:

```ts
// lib/ssrf.ts
import { lookup } from "node:dns";
import type { LookupAddress, LookupOptions } from "node:dns";
import { BlockList, isIP } from "node:net";

const blocked = new BlockList();
for (const [net, prefix] of [["0.0.0.0", 8], ["10.0.0.0", 8], ["100.64.0.0", 10], ["127.0.0.0", 8],
  ["169.254.0.0", 16],                                  // link-local, incl. the metadata service 169.254.169.254
  ["172.16.0.0", 12], ["192.168.0.0", 16], ["224.0.0.0", 4], ["240.0.0.0", 4]] as const)
  blocked.addSubnet(net, prefix, "ipv4");
for (const [net, prefix] of [["::", 128], ["::1", 128], ["::ffff:0:0", 96],   // unspecified, loopback, IPv4-mapped
  ["fc00::", 7], ["fe80::", 10], ["ff00::", 8]] as const)                      // unique-local, link-local, multicast
  blocked.addSubnet(net, prefix, "ipv6");

export const isBlocked = (ip: string) => blocked.check(ip, isIP(ip) === 6 ? "ipv6" : "ipv4");
const blockedError = (host: string, ip: string) =>
  Object.assign(new Error(`${host} resolves to blocked address ${ip}`), { code: "ERR_SSRF_BLOCKED" });

type LookupCallback = (err: NodeJS.ErrnoException | null, address: string | LookupAddress[], family?: number) => void;

export function safeLookup(hostname: string, options: LookupOptions, cb: LookupCallback): void {
  lookup(hostname, { ...options, all: true }, (err, addresses) => {
    if (err) return cb(err, []);
    const bad = addresses.find((a) => isBlocked(a.address));        // every address, not just the first
    if (bad) return cb(blockedError(hostname, bad.address), []);
    if (options.all) return cb(null, addresses);                     // net's happy-eyeballs path asks for all
    const first = addresses[0];
    if (!first) return cb(Object.assign(new Error(`no address for ${hostname}`), { code: "ENOTFOUND" }), []);
    cb(null, first.address, first.family);
  });
}
```

Two holes the lookup can't close, so the URL check closes them:

```ts
export class UrlRejected extends Error {}                  // message = a stable code for the API error

export function assertSafeUrl(raw: string): URL {
  const url = new URL(raw);                                // WHATWG parsing: "http://2130706433/" → hostname "127.0.0.1"
  if (url.protocol !== "https:") throw new UrlRejected("https_required");
  if (url.port && url.port !== "443") throw new UrlRejected("port_not_allowed");
  if (url.username || url.password) throw new UrlRejected("credentials_in_url");
  const host = url.hostname.replace(/^\[|\]$/g, "");      // IPv6 literals arrive bracketed
  if (isIP(host) && isBlocked(host)) throw new UrlRejected("blocked_address");
  return url;
}
```

1. **IP literals never reach `lookup`.** `net.connect` skips resolution when the host is already an IP, so `https://169.254.169.254/` must be caught here.
2. **Redirects.** `undici.request` doesn't follow them; `fetch` does by default, so pass `redirect: "manual"`. Any hop through the same `Agent` would hit `safeLookup` anyway, but for webhooks a 3xx is simply a failed delivery: the endpoint must answer 2xx itself.

Run `assertSafeUrl` at registration *and* before every delivery. And remember this is the application layer: egress rules that can't reach private ranges, plus IMDSv2, are the real controls (API L16 says why).

---

## 10. Ledger: the webhook sender

```ts
// webhooks/sender.ts
import { postCapped } from "../lib/http-client.ts";
import { assertSafeUrl } from "../lib/ssrf.ts";
import { sign } from "../lib/hmac.ts";                     // L12: HMAC over `${ts}.${rawBody}`
import type { Delivery } from "./types.ts";

export async function deliver(d: Delivery, shutdown: AbortSignal) {
  assertSafeUrl(d.url);                                    // DNS and URLs change after registration
  const body = JSON.stringify(d.event);                    // these exact bytes are signed AND sent
  const ts = Math.floor(Date.now() / 1_000);
  return postCapped(d.url, {
    headers: {
      "content-type": "application/json",
      "ledger-signature": sign(ts, body, d.secrets),       // both secrets during rotation (API L23)
      "ledger-event-id": d.event.id,
      "ledger-delivery-attempt": String(d.attempt + 1),
    },
    body,
  }, { deadlineMs: 10_000, captureBytes: 2_048, signal: shutdown });
}
```

The caller records the outcome in one statement; *when* to try again is [L26](../06-data-and-jobs/26-background-jobs-and-queues.md)'s job:

```sql
UPDATE webhook_deliveries
   SET attempt = attempt + 1,
       status = $2,              -- 'delivered' | 'pending' (retry scheduled) | 'dead'
       response_status = $3,     -- NULL on network errors and timeouts
       response_body = $4,       -- ≤ 2 KB, from readCapped
       duration_ms = $5,
       next_attempt_at = $6      -- NULL unless pending; never earlier than Retry-After
 WHERE id = $1;
```

Everything in this lesson is in those two blocks: one long-lived `Agent`, a per-origin cap of 5 matched by the dispatcher's per-endpoint limiter, a 10 s total deadline under tighter phase timeouts, a body read capped at 2 KB on every path, the SSRF-safe lookup, and a shutdown signal that aborts in-flight sends ([L24](../05-concurrency/24-graceful-shutdown.md)).

---

## 11. Production rules

| Rule | Why |
|---|---|
| **Every outbound call gets a total deadline (`AbortSignal.timeout`)** | Phase timeouts default to minutes, and `bodyTimeout` is idle-based, so a drip-feed never trips it |
| **One long-lived dispatcher per purpose, created at boot** | A new `Agent` per request means a handshake, a DNS lookup and a `TIME_WAIT` port per request |
| **Cap `connections` per origin, and limit concurrency in front of it** | Unlimited is the default; the cap alone just moves the pile-up into undici's invisible queue |
| **Client idle timeout below the server's** | You're the reusing side now: L17's race, with the roles swapped |
| **Consume or cancel every response body, on every path** | An unread body pins its connection; with a cap, N of them hang the pool |
| **Cap how much of an untrusted response you read** | A merchant endpoint can stream gigabytes at you |
| **Pass `keepAlive: true` to any `http.Agent` you construct** | Only the global agent defaults to keep-alive |
| **Retry without a key only when the request provably wasn't sent** | Connect failures are safe; a reset after sending has an unknown outcome |
| **Honour `Retry-After`, capped, and never past the deadline** | Respect the server without letting it park you forever |
| **Validate SSRF in the dispatcher's `lookup`, plus an IP-literal check on the URL** | Check-then-fetch loses to DNS rebinding; IP literals skip `lookup` |
| **Don't follow redirects for webhooks** | A 3xx is a failed delivery, and a redirect is an SSRF vector |
| **Watch DNS: reuse connections, cache resolutions deliberately** | Every new connection is a `getaddrinfo` on the shared 4-thread pool |

> **Spring equivalent:** `RestClient`/`WebClient` sit on a pooled client (Apache HttpClient, Reactor Netty) with the same knobs: connect timeout, response/read timeout, max connections per route, pending-acquire timeout, and the same per-read vs total gap. [Spring L34](https://github.com/Pawar-Pratik/SpringBoot/blob/main/09-production/34-resilience-and-integration.md)

---

## 12. Interview traps

**Q1. "Why does connection reuse matter so much?"**
A new connection costs a DNS lookup (on the libuv pool in Node), a TCP handshake, a TLS handshake that burns CPU in your process, and a port stuck in `TIME_WAIT` when it closes. Reuse pays those once per connection instead of once per request. The weak answer stops at "it's faster"; the strong one names port exhaustion and DNS pressure on the thread pool as the failure modes at volume.

**Q2. "What happens if you don't consume a fetch response body?"**
The socket can't return to the pool, because the remaining body bytes sit in front of the next response. With a connection cap, a handful of unread error bodies exhausts the pool and later requests hang until their deadlines; without one, you leak sockets. It's intermittent because undici eventually cancels bodies of garbage-collected responses. Fix: read or `cancel()` the body on every path.
**Follow-up:** "And if you only want the first 2 KB?" Read up to the cap and stop; the stream is destroyed and that connection is closed rather than reused, which is fine for rare cases.

**Q3. "What timeouts does an outbound call need?"**
Connect, headers and a body idle timeout at the dispatcher, plus a **total deadline** via `AbortSignal`. The total matters because `bodyTimeout` measures gaps between chunks (a drip-feed never trips it) and no phase timeout covers time spent queued for a connection. Deadlines must shrink as you go downstream.

**Q4. "How do you prevent SSRF in a webhook system?"**
Validate the URL (https, port, no credentials, no blocked IP literal) at registration and before every delivery, and enforce the IP check inside the dispatcher's `lookup`, so the socket connects only to an address you validated. Check every resolved address, don't follow redirects, and put egress controls and IMDSv2 underneath, because application checks are the mitigation, not the control.

**Q5. "What is DNS rebinding, and why does check-then-fetch fail?"**
The attacker's DNS answers with a public IP when you validate and a private one when you connect (TTL 0). Validation and connection do two separate resolutions, so the check proves nothing about the connection. Resolving once, inside the connect path, and connecting to the checked address removes the gap.

**Q6. "What's `TIME_WAIT`, and why would you run out of ports?"**
The side that closes first keeps the connection's port pair reserved for a while (60 s on Linux). With about 28k ephemeral ports by default, that caps sustained new connections to one destination at roughly 470 per second; past it, `connect()` fails with `EADDRNOTAVAIL` while everything else looks healthy. Fix: keep-alive reuse, not kernel tuning.

**Q7. "Why does undici's client keep-alive default to 4 seconds?"**
To close idle connections before a default Node server (5 s) does. Whoever reuses a connection must give up on it first, or a reused request races the server's FIN and gets a reset. It's the client side of the load-balancer 502.

**Q8. "When is it safe to retry a POST without an idempotency key?"**
Only when the request provably never reached the server: connection refused, connect timeout, DNS failure. A reset or timeout after the request was written has an unknown outcome, so it needs a key ([API L18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md)). Honour `Retry-After` on 429/503, capped and within the deadline.

**Q9. "Outbound calls take 800 ms, but the remote team swears they answer in 20 ms. Where's the time?"**
On your side: waiting in the pool's queue because the per-origin cap is saturated; new connections per request (DNS on a busy thread pool, TCP, TLS); or your own event loop delaying the callbacks ([L08](../02-event-loop/08-blocking-the-loop.md)). Measure queue time separately from request time, and check loop delay and thread-pool pressure before blaming them.

**Q10. "Why is an unlimited connection pool dangerous?"**
A slow dependency plus a burst gives you hundreds of sockets to one origin: descriptors, buffers and a flood against a receiver already in trouble. A cap bounds it, but then requests queue inside the client, so you also need a visible concurrency limit in front with the same size.

---

## 13. Build & break

### Build — `lib/http-client.ts` and `webhooks/sender.ts`

1. `lib/http-client.ts`: `webhookAgent` as in §5; `postCapped()` returning a discriminated `Outcome` (`response` with status, capped body, `retryAfterMs`, `ms`; or a `Failure` from `classify`); `readCapped`; `retryAfterMs`; and a `processorPool` (`new Pool(origin, …)`) for the FakeProcessor with its own deadlines.
2. `lib/ssrf.ts`: `BlockList`, `safeLookup`, `assertSafeUrl`, with table-driven tests: `https://169.254.169.254/`, `https://[::1]/`, `https://[::ffff:127.0.0.1]/`, `https://2130706433/`, `https://127.0.0.1.nip.io/` (blocked at connect), `https://user:pw@example.com/`, `http://example.com/`.
3. `webhooks/sender.ts`: `deliver()` as in §10, wired to L10's `keyedLimit(5)` per endpoint and a global limit of 200, plus the `UPDATE` above.
4. Metrics: time queued in the limiter, request duration by outcome kind, open sockets per origin.

### Break — four experiments

1. **An Agent per request.** Against a local `node:http` server, send 2,000 sequential requests with `request(url, { dispatcher: new Agent() })`, then run `netstat -an | grep -c TIME_WAIT` (works in Git Bash on Windows too) and time the loop. Repeat with one shared `Agent`: `TIME_WAIT` stays near zero and the loop is faster. Work out how long the first version would take to exhaust the port range at production rates.
2. **Unread error bodies.** Make the server answer 500 with a small body. Client: `import { Agent, fetch } from "undici"`, an `Agent({ connections: 2 })`, and a loop that does `if (!res.ok) continue;` without touching the body. Request 3 hangs. Run with `node --expose-gc` and call `globalThis.gc?.()` from a timer and it may come unstuck, which is the intermittency you'd see in production. Fix with `await res.body?.cancel()` and run 10,000 requests through two connections.
3. **No deadline.** A server with `createServer(() => {})` never answers. `await fetch(url)` and print the elapsed time when it finally fails (minutes, via the default headers timeout). Add `signal: AbortSignal.timeout(2_000)`: a `TimeoutError` at 2 s. Then a server that sends headers and one body byte every 4 s, with `bodyTimeout: 5_000` on your Agent: it never trips. Only the total deadline ends it.
4. **SSRF and rebinding.** Run the `ssrf.ts` test table and confirm every entry is rejected, either by `assertSafeUrl` or with `ERR_SSRF_BLOCKED` at connect. Then simulate rebinding deterministically: write `flipLookup`, a fake resolver that returns `93.184.215.14` on odd calls and `127.0.0.1` on even ones. Build a naive `checkThenSend` that validates with one `flipLookup` call and connects through an Agent whose `connect.lookup` calls it again: your local server on 127.0.0.1 logs a hit. Now wrap the same resolver in `safeLookup`'s logic so the check and the connect are one resolution: blocked.

### Explain out loud (60–120 seconds)

1. What a dispatcher is, and what reuse saves in Node: DNS on the pool, handshakes, `TIME_WAIT`.
2. The four timeouts, why `bodyTimeout` isn't enough, and why every call has a total deadline.
3. The unconsumed-body leak, and why it shows up intermittently.
4. The port-exhaustion arithmetic.
5. Why the SSRF check belongs in `lookup`, plus the two holes (IP literals, redirects) the URL check closes.

---

## What's next

You've now built the server side and the client side with no framework at all. Next, the frameworks themselves, judged by mechanism: what Express, Fastify and NestJS actually do on every request, what that costs next to a database call, and which one Ledger should run.

Next → **[Lesson 19: Frameworks by mechanism: Express, Fastify, NestJS](19-frameworks-by-mechanism.md)**
