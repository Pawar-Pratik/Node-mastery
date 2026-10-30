# Lesson 16 — TCP & the `net` module

> **Why this lesson exists:** every HTTP request, Postgres query and Redis command Ledger makes is bytes on a TCP socket. Once you know that a socket is a Duplex stream with **no message boundaries**, a lot of things stop being mysterious: why protocols need framing, what `ECONNRESET` and `EPIPE` actually report, why an idle connection can be dead without anyone noticing, and how every database driver you use works inside. Interviewers use "does TCP preserve message boundaries?" to find out whether you understand the layer under your frameworks.

**Time:** ~70 minutes · **Prereq:** Lesson 13

---

## 1. The idea in one sentence

> **A TCP socket is an ordered, reliable stream of bytes with no message boundaries. Every protocol you use is a framing rule on top of it, and every driver is a parser plus a queue.**

---

## 2. Why this matters: the bug that works on localhost

Here's an internal service that sends JSON events between two Ledger processes over a raw socket:

```ts
// ❌ Assumes one write on one side = one 'data' event on the other.
socket.on("data", (chunk) => {
  const event = JSON.parse(chunk.toString());   // 💥 sometimes; silently wrong other times
  handle(event);
});
```

On a laptop with small messages and gaps between sends, every `'data'` event happens to hold exactly one message, and the tests pass. In production two things happen, and this test on Node 24 shows both on loopback:

- **Coalescing.** Two quick writes of 13 bytes each arrived as **one** 26-byte `'data'` event. `JSON.parse` throws on `{"type":"a"}\n{"type":"b"}\n`.
- **Splitting.** One 1 MB write arrived as **seventeen** `'data'` events of up to 64 KiB each. `JSON.parse` throws on the first fragment.

TCP never promised anything else. It delivers bytes in order, with nothing marking where one `write()` ended. **Message boundaries are something your protocol has to add.** HTTP, Postgres, Redis and gRPC each do it differently, and once you can write a framer yourself, those drivers are no longer black boxes.

---

## 3. `net` in twenty lines

```ts
import net from "node:net";

const server = net.createServer((socket) => {          // one call per accepted connection
  socket.on("data", (chunk) => socket.write(chunk));   // echo (no framing yet)
  socket.on("error", (err) => console.warn({ code: (err as NodeJS.ErrnoException).code }, "socket error"));
});
server.listen(7000);

const client = net.connect({ port: 7000, host: "127.0.0.1" });
client.on("connect", () => client.write("hello"));
client.on("data", (chunk) => { console.log(chunk.toString()); client.end(); });
```

A `net.Socket` is a **Duplex stream** ([L13](../03-data-and-io/13-streams-and-backpressure.md)): the readable side carries bytes from the peer, the writable side carries bytes to it, and the two are independent. So everything from Module 3 applies: `for await` over a socket, `pipeline(socket, …, socket)`, and `write()` returning `false`.

Sockets don't use the libuv thread pool. Reads and writes are readiness-driven, through epoll on Linux, kqueue on macOS and IOCP on Windows, which is why one process can hold tens of thousands of connections. **Connecting to a hostname is the exception:** `net.connect({ host: "redis" })` resolves it with `dns.lookup`, which runs on the pool ([L07](../02-event-loop/07-the-thread-pool.md)).

**The lifecycle, in event order:**

| Event | Meaning |
|---|---|
| `'connection'` (server) | A connection was accepted; you get the socket |
| `'connect'` (client) | The three-way handshake completed ([API L02](https://github.com/Pawar-Pratik/API-mastery/blob/main/01-foundations/02-journey-of-a-request.md)) |
| `'data'` | Some bytes arrived: any amount, at any boundary |
| `'drain'` | The write buffer emptied after `write()` returned `false` |
| `'end'` | The peer sent FIN: **it** won't send any more. You may still be able to write (§6) |
| `'timeout'` | No activity for the `setTimeout` duration. **Nothing else happens** (§7) |
| `'error'` | Something failed (`ECONNRESET`, `EPIPE`, …). It's followed by `'close'`, without `'end'` |
| `'close'` | The socket is fully closed; `hadError` says whether an error caused it. Always last |

---

## 4. Framing: putting boundaries back

There are only a few ways to mark where a message ends:

| Strategy | How the receiver knows the end | Used by |
|---|---|---|
| **Delimiter** | A byte sequence that can't appear in the message (or is escaped) | NDJSON (`\n`), HTTP/1.1 headers (`\r\n\r\n`), RESP's simple replies, SMTP |
| **Length prefix** | A fixed-size header carries the body length | Postgres messages (1-byte type + 4-byte length), gRPC messages (1-byte flag + 4-byte length), Kafka |
| **Self-describing** | The header line says how many bytes follow | RESP bulk strings (`$5\r\nhello\r\n`), HTTP bodies via `Content-Length` or chunked encoding |
| **Fixed size** | Every message is N bytes | Some binary sensor and market-data protocols |

**Delimiter framing** is [Lesson 14](../03-data-and-io/14-composing-streams.md)'s `splitLines` applied to a socket. NDJSON over TCP is then three lines:

```ts
await pipeline(socket, splitLines({ maxLine: 64 * 1024 }), async function (lines) {
  for await (const line of lines) handle(JSON.parse(line));
});
```

The weakness of delimiters is that the payload can't contain the delimiter, or must escape it. JSON escapes newlines inside strings, so NDJSON works; arbitrary binary data doesn't.

**Length-prefix framing** handles any payload, binary included. A frame is a 4-byte big-endian length followed by that many bytes ([L12](../03-data-and-io/12-buffers-and-encodings.md) covers `readUInt32BE`):

```ts
// src/lib/framing.ts
export function encodeFrame(payload: Buffer): Buffer {
  const header = Buffer.allocUnsafe(4);
  header.writeUInt32BE(payload.length, 0);
  return Buffer.concat([header, payload]);
}

export function lengthPrefixed({ maxFrame = 1024 * 1024 } = {}) {
  return async function* (source: AsyncIterable<Buffer>): AsyncGenerator<Buffer> {
    let buf: Buffer = Buffer.alloc(0);
    for await (const chunk of source) {
      buf = buf.length === 0 ? chunk : Buffer.concat([buf, chunk]);
      while (buf.length >= 4) {
        const len = buf.readUInt32BE(0);
        if (len > maxFrame) throw new Error(`frame of ${len} bytes exceeds ${maxFrame}`);
        if (buf.length < 4 + len) break;                   // wait for the rest of the frame
        yield Buffer.from(buf.subarray(4, 4 + len));       // copy: don't pin the big buffer (L12)
        buf = buf.subarray(4 + len);
      }
    }
    if (buf.length > 0) throw new Error("connection closed mid-frame");
  };
}
```

The details are where the production bugs are:

- **`maxFrame` is a security control.** Without it, a peer sends a header saying `0xFFFFFFFF` and you buffer data towards 4 GiB, waiting for a frame that will never complete.
- **The `while` loop** handles several frames in one chunk (coalescing). **The `break`** handles a frame spread over many chunks (splitting).
- **`Buffer.from(subarray)` copies.** A `subarray` shares memory with the accumulation buffer, so retaining a small frame would keep a large buffer alive.
- **Repeated `Buffer.concat` is quadratic for large frames.** A 16 MB frame arriving in 64 KiB chunks gets copied about 250 times. For large frames, keep a list of chunks and concatenate once when the total reaches `4 + len`.
- **Ending mid-frame is an error,** not a clean end of stream: the peer died or misbehaved.

The echo server is now a `pipeline` with the same socket at both ends:

```ts
// src/net/echo-server.ts
import net from "node:net";
import { pipeline } from "node:stream/promises";
import { encodeFrame, lengthPrefixed } from "../lib/framing.ts";

const server = net.createServer({ noDelay: true }, (socket) => {
  socket.setTimeout(30_000, () => socket.destroy());        // idle peers don't get to stay
  pipeline(
    socket,
    lengthPrefixed({ maxFrame: 64 * 1024 }),
    async function* (frames) { for await (const f of frames) yield encodeFrame(f); },
    socket,
  ).catch((err) => console.warn({ code: err.code, msg: err.message }, "connection failed"));
});
server.maxConnections = 10_000;
server.listen(7001);
```

Backpressure works with no extra code: if the client stops reading its replies, the server's `write` side fills, `pipeline` stops pulling frames, and the server stops reading from the socket. TCP then pushes back on the client's writes. Any error, whether an oversize frame, a reset or a timeout, destroys the socket through `pipeline`.

---

## 5. Nagle, delayed ACKs and keepalive

**Nagle's algorithm** makes the kernel hold back a small write while earlier data is still unacknowledged, so it can combine small writes into fewer packets. It saves bandwidth for interactive streams of tiny writes. It also interacts badly with **delayed ACKs**, where the receiver waits briefly hoping to piggyback its ACK on a response. A request written as two small writes (headers, then body) can wait on that delay before the second write goes out, and the result looks like unexplained latency of tens of milliseconds on small requests.

Request/response protocols want each message sent immediately, so **modern HTTP servers and database drivers disable Nagle.** A raw `net` socket leaves the OS default (Nagle on) unless you call `socket.setNoDelay(true)` or create it with `noDelay: true`, as the echo server above does. The complementary habit is to **write each message with one `write()` call** (build the frame in a Buffer, then write it), so you send full messages rather than fragments.

**TCP keepalive** (`socket.setKeepAlive(true, initialDelayMs)`) sends empty probes on an idle connection, so the kernel finds out when the peer has vanished. The initial delay is yours to set. The probe interval and count come from the OS, and the Linux default waits two hours before the first probe, far too long to rely on unconfigured. It does have a second use: probes keep NAT tables and load balancers from forgetting an idle connection. **Application-level heartbeats** (Redis `PING`, WebSocket ping/pong, SSE comments in [L20](20-realtime-servers.md)) are the reliable way to detect a dead peer, because they prove the other *application* is alive, not just its kernel.

---

## 6. Ending connections, and what the error codes mean

There are three ways to end a connection, and they're different on the wire:

| Call | On the wire | Meaning |
|---|---|---|
| `socket.end(data?)` | FIN after the remaining data | "I'm done sending." A **half-close**: you can still receive |
| `socket.destroy()` | FIN, **or RST if unread data is sitting in your receive buffer** | "Tear it down now." Queued writes are discarded |
| `socket.resetAndDestroy()` | RST | "Abort." The peer gets `ECONNRESET` |

**`allowHalfOpen`** (default `false`) decides what happens when the *peer* sends FIN. By default Node ends your side automatically too. With `allowHalfOpen: true` the socket stays writable, so you can send a reply after the peer has finished sending. That's the "send the request, `end()`, then read the response" pattern some protocols use. Tested on Node 24, a client that calls `end("request")` with `allowHalfOpen: true` still receives the server's reply sent after the FIN.

**The error codes**, as Node reports them:

| Code | What actually happened | Typical cause |
|---|---|---|
| `ECONNREFUSED` | The peer's kernel answered the SYN with RST: nothing is listening | Process down, wrong port, not started yet |
| `ECONNRESET` | The connection was **aborted with RST** | Peer crashed or was killed with unread data, peer called `resetAndDestroy`, a proxy or load balancer dropped the connection, an idle keep-alive connection the server had already closed ([L17](17-http-from-scratch.md)) |
| `EPIPE` | You wrote to a connection the peer has already closed | Writing after the peer went away. On Windows it typically shows up as `ECONNABORTED` or `ECONNRESET` instead |
| `ETIMEDOUT` | The kernel gave up retransmitting or connecting | Packets vanish: a firewall drop, a dead host, a partition |
| `ENOTFOUND` | DNS has no record for the name | Typo, missing service, DNS outage |

Two Node-specific facts make these errors show up as errors rather than crashes. **Node ignores `SIGPIPE`** at startup, so writing to a closed socket produces an `EPIPE` error event instead of the default action of killing the process. And **`'error'` on a socket with no listener crashes the process** ([L11](../03-data-and-io/11-eventemitter.md)), which is why every raw socket needs an error handler or a `pipeline`.

Both of these were observed on Node 24 on Windows: killing a server process mid-stream gave the client `ECONNRESET`, and the length-prefixed server destroying a socket that still had an unread oversize frame in its buffer made the kernel send RST, so the client saw `ECONNRESET` rather than a clean end.

**"Half-open" means two different things**, and a strong answer names both:

1. **Half-closed:** one side has sent FIN and the other is still sending. It's legitimate, and `allowHalfOpen` controls it.
2. **Half-open (dead peer):** one side thinks the connection is alive, and the other side is gone: it lost power, a NAT entry expired, a cable was pulled. No FIN or RST ever arrives. **A reader waits forever.** A writer finds out only when its retransmissions give up, which on Linux defaults takes on the order of 15 minutes. The defences are idle timeouts, keepalive and application heartbeats.

---

## 7. Timeouts: `setTimeout` only tells you

```ts
// ❌ Looks like a 30 s timeout. It isn't: the socket stays open forever.
socket.setTimeout(30_000);

// ✅ 'timeout' is a notification; deciding what it means is your job.
socket.setTimeout(30_000);
socket.on("timeout", () => socket.destroy(new Error("idle for 30 s")));
```

`socket.setTimeout(ms)` arms an **idle** timer, reset by any activity on the socket. When it fires, Node emits `'timeout'` and does nothing else. The docs say you must end or destroy the socket yourself. That's deliberate: for some protocols, idle means "send a heartbeat", not "hang up". The Redis client in §10 uses exactly that freedom. It treats idle as fatal only while it's waiting for a reply.

`setTimeout` is an idle timeout, so a peer that trickles one byte every 29 seconds never trips it. For a **total deadline** and a **connect timeout**, use an `AbortSignal`, which `net.connect` accepts:

```ts
const socket = net.connect({ host, port, signal: AbortSignal.timeout(2_000) });
await once(socket, "connect");    // rejects with AbortError after 2 s, or with ECONNREFUSED etc.
```

`events.once` rejects when the emitter emits `'error'`, so this single line handles refusal, DNS failure and the timeout.

---

## 8. Capacity: backpressure, connection limits, closing

**Backpressure on `socket.write()`** is [Lesson 13](../03-data-and-io/13-streams-and-backpressure.md) exactly: `false` past `writableHighWaterMark` (16 KiB on Windows, 64 KiB elsewhere, by default), then wait for `'drain'`. `socket.writableLength` shows what's queued in Node, and a server that broadcasts to thousands of sockets must watch it per socket, which is [L20](20-realtime-servers.md)'s slow-consumer problem.

**Every connection costs memory and a descriptor:** the socket object, its read and write buffers, and a file descriptor counted against `ulimit -n` ([L15](../03-data-and-io/15-the-file-system.md)). Bound it:

- `server.maxConnections = N` makes the server close new connections once N are open. That's crude but effective shedding.
- Idle timeouts reclaim connections from clients that went quiet.

**`server.close()`** stops accepting new connections and **leaves existing ones open**. Its callback fires only when the last one ends. A graceful shutdown therefore has to end or destroy the open connections itself, which [L24](../05-concurrency/24-graceful-shutdown.md) covers. `server.getConnections(cb)` counts what's still open.

**TLS** (`node:tls`) wraps a socket in encryption: `tls.connect({ host, port, servername })` and `tls.createServer({ key, cert })`. A `TLSSocket` is still a Duplex stream, so framing, backpressure and timeouts work the same way, and the handshake adds round trips before `'secureConnect'` ([API L02](https://github.com/Pawar-Pratik/API-mastery/blob/main/01-foundations/02-journey-of-a-request.md)). Always pass `servername` for SNI, and never set `rejectUnauthorized: false` outside a test.

**Local sockets** skip TCP entirely for processes on the same machine. `server.listen("/var/run/ledger.sock")` is a Unix domain socket on Linux and macOS, as used by Postgres's local connections and the Docker daemon. On Windows the same `net` API uses **named pipes**, and the path must be in the pipe namespace: `server.listen("\\\\.\\pipe\\ledger")`. The code is otherwise identical; only the address changes.

---

## 9. What a driver really is

A database or cache driver is four things on top of a socket:

1. **An encoder** that turns your call into protocol bytes.
2. **An incremental parser** that turns arriving chunks into whole replies (framing).
3. **A queue that matches replies to requests.** Replies on one connection come back in order, so a FIFO of pending promises is enough.
4. **Connection management:** pooling, reconnects, timeouts, TLS and authentication.

Building items 1–3 for Redis takes about a hundred lines, and doing it once explains a whole category of production behaviour.

---

## 10. Ledger: speaking RESP to Redis by hand

Ledger's rate limiter and caches talk to Redis 7. Redis's protocol, **RESP**, frames with both techniques: CRLF-terminated header lines, and explicit byte lengths for payloads.

| First byte | Type | Example on the wire |
|---|---|---|
| `+` | Simple string | `+OK\r\n`, `+PONG\r\n` |
| `-` | Error **reply** | `-ERR unknown command 'NOPE'\r\n` |
| `:` | Integer | `:42\r\n` |
| `$` | Bulk string: length, then bytes | `$5\r\nhello\r\n`; `$-1\r\n` is nil |
| `*` | Array of N replies | `*2\r\n$3\r\nGET\r\n$4\r\npi_1\r\n` |

**Commands are arrays of bulk strings,** and **the `$` length is in bytes, not characters**: `"₹4,999"` is 6 characters but 8 bytes, which is the `string.length` vs `Buffer.byteLength` bug from [L12](../03-data-and-io/12-buffers-and-encodings.md). Redis 7 speaks RESP2 unless the client asks for RESP3 with `HELLO 3`.

```ts
// src/net/resp-client.ts
import net from "node:net";
import { once } from "node:events";

export type RespValue = string | number | null | RespValue[] | RespError;
export class RespError extends Error { readonly code = "REDIS_ERROR"; }
export class ProtocolError extends Error { readonly code = "RESP_PROTOCOL"; }

export function encodeCommand(args: readonly string[]): Buffer {
  const parts: Buffer[] = [Buffer.from(`*${args.length}\r\n`)];
  for (const a of args) {
    const b = Buffer.from(a, "utf8");
    parts.push(Buffer.from(`$${b.length}\r\n`), b, Buffer.from("\r\n"));   // BYTE length (L12)
  }
  return Buffer.concat(parts);
}

/** Parses one reply at `pos`. Returns undefined if the buffer doesn't hold a whole reply yet. */
export function parseReply(buf: Buffer, pos = 0): { value: RespValue; next: number } | undefined {
  const eol = buf.indexOf("\r\n", pos);
  if (eol === -1) return undefined;                              // header line incomplete
  const type = String.fromCharCode(buf[pos]!);
  const line = buf.toString("utf8", pos + 1, eol);
  const after = eol + 2;
  switch (type) {
    case "+": return { value: line, next: after };               // delimiter-framed
    case "-": return { value: new RespError(line), next: after };
    case ":": return { value: Number(line), next: after };
    case "$": {                                                  // length-prefixed
      const len = Number(line);
      if (len === -1) return { value: null, next: after };
      if (!Number.isInteger(len) || len < 0 || len > 512 * 1024 * 1024) throw new ProtocolError(`bad length ${line}`);
      if (buf.length < after + len + 2) return undefined;        // body not here yet
      return { value: buf.toString("utf8", after, after + len), next: after + len + 2 };
    }
    case "*": {
      const n = Number(line);
      if (n === -1) return { value: null, next: after };
      const items: RespValue[] = [];
      let p = after;
      for (let i = 0; i < n; i++) {
        const r = parseReply(buf, p);
        if (!r) return undefined;
        items.push(r.value);
        p = r.next;
      }
      return { value: items, next: p };
    }
    default: throw new ProtocolError(`unknown type byte ${JSON.stringify(type)}`);
  }
}
```

The client is the parser plus a FIFO queue:

```ts
type Pending = { resolve: (v: RespValue) => void; reject: (e: Error) => void };

export class RespClient {
  #socket: net.Socket;
  #pending: Pending[] = [];                                      // replies arrive in request order
  #buf: Buffer = Buffer.alloc(0);

  private constructor(socket: net.Socket) {
    this.#socket = socket;
    socket.on("data", (chunk) => this.#onData(chunk));
    socket.on("error", (err) => this.#failAll(err));
    socket.on("close", () => this.#failAll(new Error("connection closed")));
    socket.setTimeout(5_000);                                    // only EMITS 'timeout'…
    socket.on("timeout", () => {                                 // …so we decide what it means
      if (this.#pending.length > 0) socket.destroy(new Error("no reply from redis in 5 s"));
    });
  }

  static async connect(port: number, host: string, timeoutMs = 2_000): Promise<RespClient> {
    const socket = net.connect({ port, host, noDelay: true, signal: AbortSignal.timeout(timeoutMs) });
    await once(socket, "connect");                               // rejects on refusal or timeout
    socket.setKeepAlive(true, 30_000);
    return new RespClient(socket);
  }

  send(...args: string[]): Promise<RespValue> {
    if (this.#socket.destroyed) return Promise.reject(new Error("connection closed"));
    return new Promise((resolve, reject) => {
      this.#pending.push({ resolve, reject });
      this.#socket.write(encodeCommand(args));                   // don't wait for the reply: pipelining
    });
  }

  close(): void { this.#socket.end(); }

  #onData(chunk: Buffer): void {
    this.#buf = this.#buf.length === 0 ? chunk : Buffer.concat([this.#buf, chunk]);
    let pos = 0;
    try {
      for (let r = parseReply(this.#buf, pos); r; r = parseReply(this.#buf, pos)) {
        pos = r.next;
        const waiter = this.#pending.shift();
        if (!waiter) throw new ProtocolError("reply with no pending command");
        if (r.value instanceof RespError) waiter.reject(r.value);   // a reply, not a broken connection
        else waiter.resolve(r.value);
      }
    } catch (err) {
      this.#socket.destroy(err as Error);                        // a corrupt stream can't be resynced
      return;
    }
    this.#buf = this.#buf.subarray(pos);                         // keep only the unparsed tail
  }

  #failAll(err: Error): void {
    for (const p of this.#pending.splice(0)) p.reject(err);
  }
}
```

```ts
const redis = await RespClient.connect(6379, "127.0.0.1");
await redis.send("PING");                                   // "PONG"
await redis.send("SET", "ratelimit:mrc_1", "17", "EX", "60"); // "OK"
await redis.send("GET", "ratelimit:mrc_1");                 // "17"
await redis.send("GET", "missing");                         // null  ($-1)
```

This client was tested against a fake server that sent every reply in random 1–3 byte fragments, including a `₹` cut mid-character, and against 1,000 pipelined `SET`s fired at once: every reply matched its command. What makes that work:

- **The parser is incremental.** It returns `undefined` for "not yet", so the client keeps the unparsed tail and waits for the next chunk. Parsing from a byte offset avoids re-copying the buffer for every reply.
- **The FIFO queue is the whole matching mechanism.** Redis answers commands on a connection in order, so the first reply belongs to the first pending command. That's why `Promise.all` of 1,000 sends on one connection works.
- **Pipelining is free.** `send` writes without waiting, so 1,000 commands cost about one round trip instead of 1,000. `ioredis` and `node-redis` do the same, and `ioredis` can also batch commands issued in the same tick into one write (`enableAutoPipelining`).
- **A `-ERR` is a reply, not a failure.** It rejects one command; the connection is fine. A *protocol* error destroys the connection, because once the byte stream is misparsed, you can't find the next reply boundary.
- **The socket closing rejects every pending command.** Otherwise they'd hang forever, which is the same "no silent hangs" rule as [L13](../03-data-and-io/13-streams-and-backpressure.md).

**What real drivers add on top:**

| Driver | Framing | Requests per connection | What else it adds |
|---|---|---|---|
| **`ioredis` / `node-redis`** | RESP (delimiter + length) | Many in flight, matched FIFO: **one connection multiplexes everything** | Reconnect with backoff, resend or reject on reconnect, RESP3, Cluster and Sentinel, TLS, auth |
| **`pg`** | Postgres messages: 1-byte type + 4-byte length | **One query at a time**: further queries queue on the client | The extended protocol, type parsers, `Pool` because one client can't run queries concurrently ([L25](../06-data-and-jobs/25-databases-from-node.md)) |

That contrast is the core of why Redis clients need one connection while Postgres needs a pool. One more consequence: **blocking commands (`BLPOP`) and pub/sub (`SUBSCRIBE`) take over a Redis connection**, so they need dedicated connections, because everything queued behind them waits.

> **Spring equivalent:** Lettuce, Spring Data Redis's default client ([SpringBoot L11](https://github.com/Pawar-Pratik/SpringBoot/blob/main/04-spring-boot/11-autoconfiguration.md)), is this same design on Netty: one thread-safe connection shared by all threads, commands pipelined and matched to replies in order. Netty's pipeline of decoders is the JVM's version of the framing stages above ([SpringBoot L18](https://github.com/Pawar-Pratik/SpringBoot/blob/main/05-web/18-mvc-vs-webflux-virtual-threads.md)).

---

## 11. Production rules

| Rule | Why |
|---|---|
| **Never treat one `'data'` event as one message** | TCP coalesces and splits writes; framing is the protocol's job |
| **Cap every frame and line length** | A hostile length header otherwise makes you buffer gigabytes |
| **Every socket has an `'error'` handler or lives inside `pipeline`** | An unhandled `'error'` crashes the process |
| **`setTimeout` + destroy in the `'timeout'` handler** | `'timeout'` only notifies; the socket stays open otherwise |
| **Use an `AbortSignal` for connect timeouts and total deadlines** | An idle timer never fires against a trickling or blackholed peer |
| **`setNoDelay(true)` for request/response protocols; one `write()` per message** | Nagle plus delayed ACKs adds latency to small messages |
| **Detect dead peers with application heartbeats, not just TCP keepalive** | Keepalive defaults are hours, and only prove the kernel is alive |
| **Reject every pending request when the connection closes** | Otherwise callers hang forever |
| **Destroy the connection on a protocol parse error** | A misparsed byte stream can't be resynchronised |
| **Treat `ECONNRESET` on idle pooled connections as retryable for idempotent requests** | The peer or a middlebox closed the connection while it sat idle |
| **Bound connections (`maxConnections`, idle timeouts) and watch `writableLength`** | Each connection costs memory and a descriptor; slow readers buffer |
| **Handle `server.close()` not closing existing connections** | Shutdown hangs on long-lived clients unless you end them ([L24](../05-concurrency/24-graceful-shutdown.md)) |

---

## 12. Interview traps

**Q1. "Does TCP preserve message boundaries?"**
No. TCP is a byte stream: it guarantees order and delivery, and nothing about where one `write()` ends. Two writes can arrive as one `'data'` event, and one write can arrive as many. The weak answer is "yes, TCP is reliable", which confuses reliability with framing. Every protocol adds boundaries itself, by delimiter, by length prefix or with self-describing headers.
**Follow-up:** "So how does UDP differ?" UDP preserves datagram boundaries (one send, one receive) but guarantees neither delivery nor order.

**Q2. "How do database drivers frame messages?"**
Postgres prefixes every message with a 1-byte type and a 4-byte length, so the parser reads 5 bytes, then waits for exactly that many more. Redis's RESP uses CRLF-terminated header lines, with explicit byte lengths for bulk strings. The parser is incremental either way: it keeps the unparsed tail and returns "need more" until a whole message has arrived. Replies are matched to requests by order, using a FIFO queue on the connection.

**Q3. "What does `ECONNRESET` mean?"**
The connection was aborted with a TCP RST rather than closed with FIN. Common causes are a peer process that crashed or was killed while unread data was in its receive buffer, a proxy or load balancer dropping a connection, a peer calling `resetAndDestroy`, or reusing an idle keep-alive connection the other side had already closed. It's different from `ECONNREFUSED` (nothing listening, at connect time) and `ETIMEDOUT` (packets vanished).
**Follow-up:** "Should you retry?" Only if the request is idempotent, because you can't tell whether the server processed it before the reset ([API L18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md)).

**Q4. "What does `socket.setTimeout` do?"**
It arms an idle timer that resets on any activity, and when it fires it emits `'timeout'`. That's all. The socket stays open until you `end()` or `destroy()` it in the handler. It's also only an idle timeout: a peer sending a byte every 29 seconds never trips a 30-second timer, so total deadlines need an `AbortSignal` or your own timer.

**Q5. "What is Nagle's algorithm, and why disable it?"**
The kernel holds back small writes while earlier data is unacknowledged, coalescing them into fewer packets. Combined with the receiver's delayed ACKs, a message written in two small pieces can stall waiting for an ACK, which adds latency to request/response traffic. HTTP servers and database drivers disable it, and a raw socket needs `setNoDelay(true)`. The complementary fix is to write each message in one `write()` call.

**Q6. "What's a half-open connection?"**
It has two meanings. Half-*closed* is legitimate: one side sent FIN and the other can keep sending, which `allowHalfOpen` controls in Node. Half-*open* in the failure sense is one side believing the connection is alive after the other side vanished without a FIN or RST (power loss, an expired NAT entry). A reader waits forever, and a writer learns only when retransmissions give up, after many minutes by default. Idle timeouts, keepalive and application heartbeats detect it.

**Q7. "What happens if you write to a socket after the peer has closed it?"**
On Linux, the first write may succeed into the kernel buffer. The peer's kernel answers with RST, and later writes fail with `EPIPE`, or `ECONNRESET` if the reset is seen first. Node ignores `SIGPIPE`, so it's an `'error'` event rather than the process dying, and an unhandled `'error'` still crashes it. On Windows the same situation usually reports `ECONNABORTED` or `ECONNRESET`.

**Q8. "Why does a Redis client use one connection while `pg` uses a pool?"**
Redis accepts many commands in flight on one connection and answers them in order, so one connection with a FIFO queue serves the whole process, and pipelining makes it fast. `pg` runs one query at a time per connection (later queries queue on the client), so concurrency needs several connections, which is a pool. The exceptions on the Redis side are blocking commands and pub/sub, which monopolise a connection and need their own.

**Q9. "How do you tell a Redis error reply from a connection failure?"**
An error reply (`-ERR …`) is a well-formed message: it rejects that one command, and the connection stays healthy. A connection failure (`ECONNRESET`, close, timeout) or a protocol parse error rejects *every* pending command and needs a reconnect. Mixing them up leads either to tearing down a healthy connection for a bad command, or to retrying a command that will fail the same way forever.

**Q10. "Your server's `close()` callback never fires. Why?"**
`server.close()` only stops accepting new connections. The callback waits until every existing connection has closed, and long-lived or keep-alive clients may never close on their own. You have to end idle connections, give active ones a deadline, then destroy what's left. That's the graceful shutdown in [L24](../05-concurrency/24-graceful-shutdown.md).

---

## 13. Build & break

### Build — `src/net/resp-client.ts` and a length-prefixed echo server

1. Start Redis 7 from the track's Compose file (or `docker run -p 6379:6379 redis:7`).
2. Implement `encodeCommand`, `parseReply` and `RespClient` from §10, and the framer and echo server from §4.
3. Unit-test `parseReply` by cutting a multi-reply buffer (`*3\r\n$4\r\nhé!\r\n:42\r\n$-1\r\n`) at **every** byte offset and asserting it returns `undefined` until the last byte arrives, then the full value.
4. Against real Redis: `PING`, `SET` with a `₹` value, `GET` it back, and `GET` a missing key (`null`). Then fire 10,000 `INCR`s with `Promise.all` on one connection, time it, and compare with awaiting them one by one: pipelining's win, measured.
5. Point `ioredis` at the same Redis with `MONITOR` running in `redis-cli`: the commands it sends are the arrays you're encoding.

### Break — 4 experiments

1. **One `'data'` = one message.** Write the naive JSON-per-`'data'` server from §2. Send two messages with back-to-back `write()`s (coalesced: `JSON.parse` throws on the second `{`), then one 1 MB message (split: it throws on the first fragment). Swap in `splitLines` or `lengthPrefixed` and both cases work.
2. **The timeout that doesn't time out.** Give a server socket `setTimeout(5_000)` with no `'timeout'` handler, connect with a client that sends nothing, and check `netstat -an | findstr 7001` (Windows) or `ss -tn` (WSL2) after a minute: the connection is still `ESTABLISHED`. Add `socket.destroy()` in the handler and it's gone after 5 seconds.
3. **Write after the peer closed.** Have the server `destroy()` each socket 50 ms after accepting it. The client, created with `allowHalfOpen: true` so it doesn't close itself on `'end'`, writes every 50 ms. In WSL2 you'll see `EPIPE` (or `ECONNRESET`), and on Windows `ECONNABORTED`. Then remove the client's `'error'` listener and watch the process crash. That's the unhandled `'error'` rule.
4. **Kill the server mid-stream.** Run a server in a separate process that streams data to the client while the client also streams to it, and kill the server process (`taskkill /F /PID <pid>`, or `kill -9` in WSL2). The client gets `ECONNRESET`, because the dead process's socket had unread data, so its kernel reset the connection. Do the same against `RespClient` with commands pending: every pending promise rejects with the error rather than hanging.

### Explain out loud (60–120 seconds)

1. Why TCP has no message boundaries, and the three ways protocols add them.
2. How an incremental parser plus a FIFO queue makes a driver, and why that gives pipelining for free.
3. `end` vs `destroy` vs `resetAndDestroy`, and what `ECONNRESET`, `EPIPE` and `ECONNREFUSED` each mean.
4. Why `setTimeout` alone doesn't close a socket, and the two meanings of half-open.
5. Why Redis needs one connection and Postgres needs a pool.

---

## What's next

You can now speak a protocol over a raw socket. HTTP is just another framing rule on TCP: header lines up to a blank line, then a body delimited by `Content-Length` or chunked encoding. Next you'll build a server on `node:http` with no framework: body limits without buffering, the three server timeouts and what each defends against, and the keep-alive race behind load-balancer 502s.

Next → **[Lesson 17: `node:http` with no framework](17-http-from-scratch.md)**
