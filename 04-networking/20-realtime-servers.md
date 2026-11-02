# Lesson 20 — Real-time servers: SSE & WebSockets

> **Why this lesson exists:** the protocol choice is made in [API L23](https://github.com/Pawar-Pratik/API-mastery/blob/main/05-beyond-rest/23-realtime-and-webhooks.md) and the client is built in [React L19](https://github.com/Pawar-Pratik/React-mastery/blob/main/05-data/19-realtime.md). This lesson is the server: thousands of connections that stay open for hours in one process, each needing a heartbeat, a cap on what you'll buffer for it, cleanup when it dies, and fan-out from whichever instance committed the event to whichever instance holds the connection.

**Time:** ~75 minutes · **Prereq:** Lessons 13, 17

---

## 1. The idea in one sentence

> **A real-time connection is a long-lived Writable whose speed you don't control: it needs a heartbeat to prove it's alive, a byte cap on what you'll hold for it, cleanup when it closes, and, because instances share nothing, a bus that tells every instance about every event.**

---

## 2. What breaks when a response lasts for hours

Everything in [L17](17-http-from-scratch.md) assumed a response finishes in milliseconds. Hold one open and each assumption turns into a failure mode:

| Assumption | What happens to a long-lived response | The fix |
|---|---|---|
| Idle connections are rare | The LB closes a quiet stream after its idle timeout (ALB: 60 s) | Heartbeats (§3) |
| The client reads as fast as you write | A phone on a train stops reading; your buffer for it grows | Per-client byte cap and eviction (§4) |
| Request state dies with the request | Listeners and timers per connection outlive the connection unless removed | Cleanup on `close` (§5) |
| The instance that handles a write also answers the reader | The event is committed through instance B; the stream lives on instance A | Cross-instance fan-out (§6) |
| A deploy takes milliseconds to drain | Open streams never finish on their own | Tell clients to reconnect elsewhere (§9) |

---

## 3. An SSE endpoint on `node:http`

The format is a few text fields per event and a blank line between events ([API L23](https://github.com/Pawar-Pratik/API-mastery/blob/main/05-beyond-rest/23-realtime-and-webhooks.md) shows the wire format). What the server has to get right:

```ts
// realtime/sse.ts
import type { IncomingMessage, ServerResponse } from "node:http";
import { hub } from "./hub.ts";
import type { Client } from "./hub.ts";
import { replayAfter } from "./replay.ts";

const RESUME_ID = /^\d{1,19}$/;                        // a bigserial, as text: validate before BigInt()

export async function eventStream(req: IncomingMessage, res: ServerResponse, merchantId: string) {
  res.writeHead(200, {
    "content-type": "text/event-stream; charset=utf-8",
    "cache-control": "no-cache, no-transform",         // no-transform: intermediaries must not compress or rewrite it
    "x-accel-buffering": "no",                         // nginx: pass each write straight through
  });
  res.write("retry: 5000\n\n");                        // the client's reconnect delay, in ms; also flushes the headers

  const lastId = req.headers["last-event-id"];         // sent automatically by EventSource on reconnect
  const client: Client = {
    merchantId, res, live: false, pending: [],
    lastSent: typeof lastId === "string" && RESUME_ID.test(lastId) ? BigInt(lastId) : -1n,
  };
  const unsubscribe = hub.subscribe(client);           // 1. subscribe FIRST: live events queue in client.pending
  res.on("close", unsubscribe);                        // 2. cleanup registered before any await
  if (client.lastSent >= 0n) await replayAfter(client);   // 3. replay what they missed, from Postgres
  hub.goLive(client);                                  // 4. flush the queue (duplicates skipped), then write live
}
```

**Why subscribe before replay.** Replay is a query; it takes milliseconds. An event committed during those milliseconds isn't in the query result, and if you subscribe *after* the query you never hear about it either. Subscribing first and queueing live events during replay closes that gap; `send()` (§4) drops anything at or below `lastSent`, so the overlap is harmless.

**The `id:` field is `events.sequence`.** Each frame carries `id: <sequence>`, the browser remembers the last one, and on reconnect sends it as `Last-Event-ID`. Resume is then one indexed query on `(merchant_id, sequence)`:

```sql
SELECT sequence::text AS sequence,
       json_build_object('id', id, 'type', type, 'sequence', sequence::text, 'data', data)::text AS payload
  FROM events
 WHERE merchant_id = $1 AND sequence > $2
 ORDER BY sequence
 LIMIT 500;                -- paged: replayAfter awaits 'drain' between pages (L13), it's a producer like any other
```

Postgres renders the JSON text, so Node never parses or re-stringifies an event; it writes the string it was given. The `sequence` comes back as text, as `bigint` columns always do from `pg` ([L02](../01-foundations/02-the-javascript-node-assumes.md)), and stays a `bigint` in Node.

### Framing rules that bite

- **No `event:` line in Ledger's stream.** A frame with `event: payment.succeeded` is dispatched to `addEventListener("payment.succeeded")` only; `onmessage` never sees it. The Console's client uses `onmessage` and switches on `data.type` ([React L19](https://github.com/Pawar-Pratik/React-mastery/blob/main/05-data/19-realtime.md)), so the type travels inside the JSON. Server and client must agree on this, and it's a classic silent bug.
- **Newlines end fields.** `JSON.stringify` escapes them, which is why `data:` carries JSON. Interpolating a raw string that contains `\n\nid: 999` lets its author forge an event.
- **Heartbeats are comment lines** (`:\n\n`), which the browser ignores. Send one every 20 s, comfortably under the LB's 60 s idle timeout. They also detect dead clients: TCP never notices a peer that vanished silently ([L16](16-tcp-and-net.md)), but a write to it eventually fails and `'close'` fires.
- **`requestTimeout` doesn't apply.** It bounds *receiving* the request, which finished long ago (L17). The limits on a stream are the LB's idle timeout and your own heartbeat.

### Why your events arrive in bursts

Compression middleware buffers output so it has enough bytes to compress well. An SSE frame is 200 bytes, so frames sit in the gzip buffer until enough accumulate, then arrive together. **Exclude `text/event-stream` from compression** (Express's `compression` takes a `filter`), or flush after every write (that middleware adds `res.flush()`). A buffering proxy causes the same symptom, which is what `X-Accel-Buffering: no` and `no-transform` are for. Over HTTP/1.1 each open stream also holds one of the browser's six connections per origin, so serve the Console over HTTP/2 at the load balancer ([API L23](https://github.com/Pawar-Pratik/API-mastery/blob/main/05-beyond-rest/23-realtime-and-webhooks.md)).

### The commit-order trap

A `bigserial` value is assigned at `INSERT` and becomes visible at `COMMIT`. With concurrent writers, sequence 105 can commit and be read while 104's transaction is still open. A cursor that advances to 105 has skipped 104 for good, and so has a client resuming from `Last-Event-ID: 105`. Two layers keep this from mattering in Ledger: transactions that insert events are short (so the window is milliseconds), and the Console refetches its data on every reconnect ([React L19](https://github.com/Pawar-Pratik/React-mastery/blob/main/05-data/19-realtime.md)), so a rare skip heals itself. A strictly gap-free stream needs positions assigned by a single writer *after* commit; say so if an interviewer asks whether resume-by-sequence is exact.

---

## 4. Per-client backpressure: cap, then evict

`res.write()` returning `false` means this client's socket buffer is full ([L13](../03-data-and-io/13-streams-and-backpressure.md)). In a one-to-one pipeline you'd wait for `'drain'`. In fan-out you can't: **the hub writes one event to thousands of clients in a loop, and awaiting one slow client would stall all of them.** So the hub never awaits a client. It writes, and checks how much it's holding:

```ts
// realtime/hub.ts
import type { ServerResponse } from "node:http";
import { metrics } from "../lib/metrics.ts";

export type Client = {
  merchantId: string; res: ServerResponse; lastSent: bigint;
  live: boolean; pending: Array<[bigint, string]>;
};
const MAX_BUFFERED_BYTES = 256 * 1024;                  // what we'll hold for one client
const MAX_PENDING = 1_000;                              // live events queued during replay

export function send(c: Client, seq: bigint, payload: string): void {
  if (seq <= c.lastSent || c.res.destroyed) return;     // replay and live overlap: dedupe by sequence
  c.res.write(`id: ${seq}\ndata: ${payload}\n\n`);
  c.lastSent = seq;
  if (c.res.writableLength > MAX_BUFFERED_BYTES) evict(c, "slow_consumer");
}

function evict(c: Client, reason: string): void {
  metrics.sseEvictions.inc({ reason });
  c.res.destroy();                                      // 'close' → unsubscribe; the browser reconnects and resumes
}
```

**Eviction is safe because resume is lossless.** A slow client that gets disconnected reconnects with `Last-Event-ID` and replays from Postgres, at its own pace, through the paged replay that does respect backpressure. Without resume, eviction would lose events, and you'd be tempted to buffer forever, which is how one slow phone becomes an OOMKill ([L27](../07-production/27-memory-and-gc.md)). With resume, the database is the buffer.

The cap is your memory budget: 10,000 connections × 256 KB worst case is 2.5 GB, which tells you either the cap or the connections per instance must come down. Do that arithmetic for your own numbers before choosing either.

---

## 5. The hub, and cleanup that can't leak

```ts
// realtime/hub.ts (continued)
class Hub {
  #byMerchant = new Map<string, Set<Client>>();

  has(merchantId: string): boolean { return this.#byMerchant.has(merchantId); }

  subscribe(c: Client): () => void {
    const set = this.#byMerchant.get(c.merchantId) ?? new Set<Client>();
    this.#byMerchant.set(c.merchantId, set);
    set.add(c);
    return () => {                                      // idempotent: 'close' may fire after an eviction
      set.delete(c);
      if (set.size === 0 && this.#byMerchant.get(c.merchantId) === set) this.#byMerchant.delete(c.merchantId);
    };
  }

  publish(merchantId: string, seq: bigint, payload: string): void {    // formatted ONCE, written N times
    for (const c of this.#byMerchant.get(merchantId) ?? []) {
      if (c.live) send(c, seq, payload);
      else if (c.pending.push([seq, payload]) > MAX_PENDING) evict(c, "replay_too_slow");
    }
  }

  goLive(c: Client): void {
    for (const [seq, payload] of c.pending) send(c, seq, payload);
    c.pending.length = 0;
    c.live = true;
  }

  heartbeat(): void {                                   // ONE timer for every connection
    for (const set of this.#byMerchant.values())
      for (const c of set) if (c.live && !c.res.destroyed) c.res.write(":\n\n");
  }
}

export const hub = new Hub();
setInterval(() => hub.heartbeat(), 20_000).unref();     // unref: a heartbeat must not keep the process alive (L03)
```

**Per-connection state is the leak surface.** The textbook SSE handler creates a `setInterval` per connection and clears it on `close`; forget the `clearInterval` once and every closed connection leaves a timer firing forever, each closure holding its `res` and everything reachable from it. A single hub-level timer removes the whole class: there's nothing per connection to forget. The rule that remains is one line: **everything a connection registers is removed in its `close` handler, and that handler is attached before the first `await`.** Listen on `res`, not `req` (L17).

---

## 6. Fan-out across instances: LISTEN/NOTIFY

The hub is in-process memory. With three API replicas, a payment committed through instance B must reach a Console connected to instance A. Ledger already runs Postgres, so it uses `LISTEN/NOTIFY` as the bus:

```text
 API instance B                       Postgres                            API instance A
 POST /v1/payments ──tx──▶ payments + events + outbox ── COMMIT
                                   │
          outbox relay (L26): mark processed + pg_notify('ledger_events', 'mrc_1:4812')
                                   │                      (delivered when the relay's transaction commits)
            ┌──────────────────────┴───────────────────────┐
     LISTEN client (B)                               LISTEN client (A)
     no mrc_1 subscribers → ignore                   hub.has('mrc_1') → pump('mrc_1')
                                                           │
                                     SELECT … WHERE merchant_id = 'mrc_1' AND sequence > cursor
                                                           ▼
                                     hub.publish → byte-capped writes → the Console (React L19)
```

```ts
// realtime/listen.ts
import pg from "pg";
import { hub } from "./hub.ts";
import { fetchAfter } from "./replay.ts";               // the paged SELECT from §3, run under withTenantTx (L25)

const cursors = new Map<string, bigint>();              // per merchant: the highest sequence this instance published
const running = new Map<string, boolean>();             // merchant → "notified again while a pump was running"

async function pump(merchantId: string): Promise<void> {
  if (running.has(merchantId)) { running.set(merchantId, true); return; }   // coalesce bursts into one loop
  running.set(merchantId, false);
  try {
    do {
      running.set(merchantId, false);
      for (let full = true; full; ) {
        const rows = await fetchAfter(merchantId, cursors.get(merchantId) ?? 0n, 500);
        for (const r of rows) {
          const seq = BigInt(r.sequence);
          hub.publish(merchantId, seq, r.payload);
          cursors.set(merchantId, seq);
        }
        full = rows.length === 500;
      }
    } while (running.get(merchantId));                  // a NOTIFY arrived mid-query: go again
  } finally { running.delete(merchantId); }
}

export async function startListener(connectionString: string): Promise<pg.Client> {
  const client = new pg.Client({ connectionString });  // dedicated connection: LISTEN is session state, never pooled
  await client.connect();
  client.on("notification", (msg) => {
    const [merchantId, seqText] = (msg.payload ?? "").split(":");   // IDs, not data
    if (!merchantId || !seqText) return;
    if (!hub.has(merchantId)) { cursors.delete(merchantId); return; }
    if (!cursors.has(merchantId)) cursors.set(merchantId, BigInt(seqText) - 1n);
    pump(merchantId).catch((err) => console.error("sse pump failed", merchantId, err));   // pino in L29
  });
  await client.query("LISTEN ledger_events");
  return client;
}
```

What you must know about `NOTIFY` to use it safely:

| Property | Consequence |
|---|---|
| Payload limit is about 8,000 bytes by default | **Send IDs, not payloads.** The listener fetches the rows, so events of any size work and the data is read under RLS |
| Delivered at `COMMIT`, never for a rolled-back transaction | Listeners only hear about committed events |
| **Not durable**: a listener that's disconnected misses everything sent meanwhile | It's a wake-up signal, not a queue. On reconnect, `pump()` every merchant in `cursors`: the cursor query catches up whatever was missed |
| Needs a session | It won't work through PgBouncer in transaction mode ([L25](../06-data-and-jobs/25-databases-from-node.md)); budget one extra direct connection per instance |
| Every listening instance receives every notification | Fine at Ledger's volume; at larger scale, use a channel per merchant (`LISTEN` on first subscriber, `UNLISTEN` on last) or a dedicated bus |

Redis pub/sub is the common alternative: also fire-and-forget, and it needs a dedicated connection because a subscribed Redis connection can't run other commands (L25). If you need durability in the bus itself, that's Redis Streams or Kafka, which is more machinery than a live feed with database-backed resume needs.

### Sticky sessions: who needs them

**SSE with database-backed resume doesn't.** A reconnect can land on any instance, because the resume state is the `Last-Event-ID` the browser sends plus rows in Postgres. A WebSocket doesn't need stickiness to stay put either: it's one TCP connection and lives on one instance until it closes. Stickiness matters when a *reconnect* must reach the same instance (per-connection state kept in that instance's memory) or when a transport spreads one session over several HTTP requests (Socket.IO's long-polling fallback). Keep connection state out of instance memory and you can drop it.

---

## 7. WebSockets on Node

### The upgrade, by hand

A WebSocket starts as an HTTP request with `Upgrade: websocket`. Node doesn't route it to your request handler; it emits `'upgrade'` on the server and hands you the raw socket:

```ts
import { createHash } from "node:crypto";
const WS_GUID = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11";       // a constant fixed by RFC 6455

server.on("upgrade", (req, socket, head) => {
  const key = req.headers["sec-websocket-key"];
  if (req.headers.upgrade?.toLowerCase() !== "websocket" || typeof key !== "string") {
    socket.end("HTTP/1.1 400 Bad Request\r\n\r\n"); return;
  }
  const accept = createHash("sha1").update(key + WS_GUID).digest("base64");
  // RFC 6455's own example: key "dGhlIHNhbXBsZSBub25jZQ==" → accept "s3pPLMBiTxaQ9kYGzzhZRbK+xOo="
  socket.write("HTTP/1.1 101 Switching Protocols\r\nUpgrade: websocket\r\nConnection: Upgrade\r\n" +
               `Sec-WebSocket-Accept: ${accept}\r\n\r\n`);
  // The socket is now raw TCP carrying WebSocket frames. `head` holds bytes the HTTP parser read past the
  // headers, which may already be the first frame. This is the point where you hand over to a library.
});
```

**The accept hash is not security.** It proves the server understood the WebSocket handshake, so a confused HTTP server or cache can't accidentally "accept" one. Authentication and the `Origin` check are yours ([API L23](https://github.com/Pawar-Pratik/API-mastery/blob/main/05-beyond-rest/23-realtime-and-webhooks.md)).

**After the 101, the socket has left HTTP.** None of L17's server timeouts apply to it, it isn't an idle keep-alive connection that `server.close()` will tidy up, and the only liveness check is the one you build.

What comes next is framing: each frame has an opcode (text, binary, close, ping, pong), a 7-, 16- or 64-bit length, and client-to-server frames are masked with a 4-byte key so intermediaries can't be tricked into caching attacker-chosen bytes. That's L16's framing problem again with an RFC's worth of edge cases. **Use `ws`.** Node 22 made the global `WebSocket` *client* stable (handy in tests), but there's no built-in server.

### The Node-specific parts of a `ws` server

```ts
import { WebSocketServer } from "ws";
import type { WebSocket } from "ws";

const wss = new WebSocketServer({ noServer: true, maxPayload: 64 * 1024 });   // the default limit is 100 MiB
server.on("upgrade", (req, socket, head) => {
  if (!isAllowedOrigin(req.headers.origin)) { socket.end("HTTP/1.1 403 Forbidden\r\n\r\n"); return; }
  wss.handleUpgrade(req, socket, head, (ws) => wss.emit("connection", ws, req));
});

function sendWs(ws: WebSocket, payload: string): void {
  if (ws.bufferedAmount > 256 * 1024) { ws.terminate(); return; }   // same cap-then-evict policy as §4
  ws.send(payload);
}
```

- **`bufferedAmount` is the WebSocket version of `writableLength`.** Same policy: cap, then evict.
- **`terminate()`, not `close()`, for a slow client.** `close()` queues a close frame behind the data the client isn't reading and waits for a reply that won't come; `terminate()` destroys the socket now.
- **Heartbeats are protocol-level.** Ping every 30 s and terminate a connection that hasn't answered the previous ping. Browsers answer pings automatically (page JavaScript can't send them). The reaper loop is in API L23.
- **`perMessageDeflate` is off by default on a `ws` server.** Turning it on keeps compression state per connection, which multiplies by your connection count.
- **WebSockets have no resume.** There's no `Last-Event-ID`; a reconnecting client must tell you where it was, and you build the replay yourself. That's a large part of why Ledger's feed is SSE.

> **Spring equivalent:** `SseEmitter` (one async servlet response per connection) or WebFlux's `Flux<ServerSentEvent<T>>`, and Spring WebSocket with a STOMP broker relay for cross-instance fan-out. The servlet version spends a container connection per stream, where Node spends a descriptor and a buffer. [Spring L18](https://github.com/Pawar-Pratik/SpringBoot/blob/main/05-web/18-mvc-vs-webflux-virtual-threads.md)

---

## 8. Capacity: what each connection costs

Per connection you pay a file descriptor, kernel socket buffers, Node's per-socket objects, whatever your code attaches, and up to your byte cap in queued writes. **Measure, don't guess:** open 10,000 idle streams from a load script and divide the RSS growth by 10,000; then do it again with traffic flowing. Check `ulimit -n` against your target ([L15](../03-data-and-io/15-the-file-system.md)), add one LISTEN connection per instance to the pool budget (L25), and remember that a mass reconnect is 10,000 replay queries arriving at a 10-connection pool, which is the reason §9 jitters reconnects.

---

## 9. Shutting down long-lived connections

`server.close()` waits for in-flight responses, and an SSE response never finishes by itself, so a deploy that only calls `server.close()` hangs until it's killed. After readiness fails and the load balancer has stopped sending new connections ([L24](../05-concurrency/24-graceful-shutdown.md)), end the streams yourself:

```ts
export function drainStreams(clients: Iterable<Client>): void {
  for (const c of clients) {
    c.res.write(`retry: ${1_000 + Math.floor(Math.random() * 4_000)}\n\n`);   // reconnect in 1–5 s, spread out
    c.res.end();
  }
}
```

The jittered `retry:` spreads the reconnect herd across surviving instances and across the database's replay capacity, instead of 10,000 clients reconnecting in the same second. WebSocket clients get `ws.close(1001, "server shutting down")`, 1001 being the "going away" code, and then reconnect with their own jittered backoff ([React L19](https://github.com/Pawar-Pratik/React-mastery/blob/main/05-data/19-realtime.md)).

---

## 10. Production rules

| Rule | Why |
|---|---|
| **Heartbeat every ~20 s, below the LB idle timeout** | Quiet streams get closed by the LB, and heartbeats are how you notice dead clients |
| **Exclude `text/event-stream` from compression; send `X-Accel-Buffering: no`** | Buffering middleware and proxies turn a live feed into bursts |
| **Use `events.sequence` as the SSE `id:`; resume with one indexed query** | Reconnects become lossless, which makes eviction safe |
| **Subscribe before replaying; dedupe by sequence** | Otherwise events committed during the replay query vanish |
| **Never await a client inside fan-out; cap bytes per client and evict** | One slow consumer must not stall the hub or eat the heap |
| **One heartbeat timer for the hub, not one per connection** | Removes the "forgot `clearInterval`" leak entirely |
| **Attach the `close` cleanup before the first `await`, on `res`** | A client that leaves during replay must still be unsubscribed |
| **Format each event once, write it N times** | Serialization is per event, not per connection |
| **NOTIFY carries IDs; listeners read rows** | The ~8,000-byte payload limit, and the data is read under RLS |
| **Treat NOTIFY as a wake-up, not a queue: catch up from cursors on reconnect** | Notifications sent while a listener was down are gone |
| **A dedicated, unpooled connection for LISTEN** | It's session state; transaction pooling breaks it |
| **WebSockets: `maxPayload`, `bufferedAmount` cap, `terminate()` slow clients** | The defaults trust the client far too much |
| **On shutdown, send a jittered `retry:` and end every stream** | `server.close()` never finishes while streams are open, and a synchronized reconnect herd hurts the next instance |

---

## 11. Interview traps

**Q1. "How do you scale SSE across instances?"**
Every instance holds its own connections, so events need a bus every instance subscribes to. Ledger uses Postgres `LISTEN/NOTIFY`: the outbox relay notifies with `merchant:sequence`, each instance's dedicated LISTEN connection checks whether it has subscribers for that merchant, fetches the rows past its cursor, and writes them to local clients. Name the properties: IDs not payloads (8,000-byte limit), delivered on commit, not durable, so a reconnected listener catches up from its cursors. Redis pub/sub is the equivalent with the same caveats.
**Follow-up:** "What if the LISTEN connection drops for 30 seconds?" Those notifications are lost; on reconnect, run the cursor query for every merchant with subscribers. The bus wakes you up, and the table is the source of truth.

**Q2. "How does SSE resume after a disconnect?"**
The server sends `id:` with each event (Ledger: `events.sequence`); the browser stores the last one and sends `Last-Event-ID` when it reconnects, automatically. The server replays `sequence > last` in pages with backpressure, while already subscribed to live events, and dedupes the overlap by sequence. The caveat that makes it a strong answer: sequences become visible in commit order, not allocation order, so a strictly gap-free stream needs post-commit positions, and the client refetches on reconnect as a safety net.

**Q3. "What's the WebSocket handshake?"**
An HTTP/1.1 `GET` with `Upgrade: websocket`, `Connection: Upgrade` and a random `Sec-WebSocket-Key`. The server answers `101 Switching Protocols` with `Sec-WebSocket-Accept` = base64(SHA-1(key + a fixed GUID)). In Node that arrives as the server's `'upgrade'` event with the raw socket and a `head` buffer. The hash proves the server speaks WebSocket; it isn't authentication, and the `Origin` check is on you.

**Q4. "How do you handle a slow SSE or WebSocket consumer?"**
Never await one client during fan-out. Write, check the bytes queued for that client (`res.writableLength` or `ws.bufferedAmount`), and disconnect it past a cap. With SSE and database-backed resume, that's lossless: the client reconnects and replays at its own pace. The weak answer is "buffer until it catches up", which is how a slow client OOMs the process.

**Q5. "Why do my SSE events arrive in bursts?"**
Something is buffering the response: compression middleware waiting for enough bytes to compress, or a proxy such as nginx buffering upstream responses. Exclude `text/event-stream` from compression (or flush per write), send `X-Accel-Buffering: no` and `Cache-Control: no-transform`.

**Q6. "Do SSE connections need sticky sessions?"**
No, if resume state lives in the database: any instance can serve a reconnect from `Last-Event-ID`. A live connection is one TCP connection and stays on its instance anyway. Stickiness is needed when reconnects depend on one instance's memory or the transport uses several requests per session, as Socket.IO's polling fallback does.

**Q7. "The server sends `event: payment.succeeded`, but the client's `onmessage` never fires. Why?"**
Named events are dispatched only to listeners for that name; `onmessage` receives events without an `event:` field. Either the client adds `addEventListener` per type, or, as Ledger does, the server omits `event:` and puts `type` in the JSON.

**Q8. "Why does every SSE example add a heartbeat?"**
Two reasons. Load balancers and proxies close connections that are idle for their timeout (60 s on an ALB), and a feed can be quiet for minutes. And TCP doesn't notice a client that vanished without closing, so without writes the server keeps its subscription forever; a heartbeat write eventually fails and triggers cleanup.

**Q9. "Our deploys take minutes and then the database spikes. Why?"**
`server.close()` waits for responses that never finish, so pods wait for their kill timeout. When they die, every client reconnects at once and runs a replay query against a small pool. Fix: after readiness fails, send each stream a jittered `retry:` and end it, so reconnects spread over seconds.

**Q10. "How many SSE connections can one Node instance hold?"**
Measure it rather than quoting a number: RSS growth per idle connection, then with traffic; descriptors against `ulimit -n`; and the worst case of your per-client byte cap times the connection count. The loop isn't usually the limit for idle streams; memory, descriptors and the fan-out work per event are.

---

## 12. Build & break

### Build — `realtime/`: the live feed behind `GET /v1/events/stream`

1. `realtime/hub.ts`: `subscribe`/`unsubscribe` by merchant, `publish`, `goLive`, the single heartbeat timer, `send` with the byte cap and eviction metrics.
2. `realtime/sse.ts`: the route handler from §3 behind the Console's cookie auth, with resume from `Last-Event-ID` and a hard cap on how far back replay goes (older than the retention window → send one `{"type":"resync_required"}` event and let the Console refetch).
3. `realtime/replay.ts`: `fetchAfter(merchantId, after, limit)` under `withTenantTx` (L25), and `replayAfter(client)` paging with a wait for `'drain'` (or `'close'`) between pages.
4. `realtime/listen.ts`: the LISTEN bridge, with reconnect plus full jitter and a catch-up `pump()` for every cursor after reconnecting.
5. The relay side: the outbox relay runs `SELECT pg_notify('ledger_events', $1)` with `merchant_id || ':' || sequence` in the transaction that marks the outbox rows processed.
6. `drainStreams()` wired into the shutdown sequence (L24).
7. Tests: a client receives an event published through another instance; a reconnect with `Last-Event-ID` gets exactly the missed events; a non-reading client is evicted; closing 1,000 connections leaves the hub empty.

### Break — four experiments

1. **Bursts.** Put the API behind Express with `app.use(compression())` and publish one event per second. Watch `curl.exe -N --compressed localhost:3000/v1/events/stream`: events arrive in clumps, or not at all for a while. Add a `filter` that skips `text/event-stream`: one event per second again.
2. **The client that stops reading.** Write a `net` client that sends the stream request and then calls `socket.pause()` forever. Publish 1 KB events at 1,000/s with the eviction check removed, and log `res.writableLength` and RSS every second: both climb without limit. Restore the cap: the client is evicted at 256 KB and memory stays flat. (A real `EventSource` would now reconnect and replay.)
3. **10,000 leaked intervals.** Give each connection its own `setInterval` heartbeat and remove the `clearInterval` from `close`. A script opens and closes 10,000 streams. Count timers with `process.getActiveResourcesInfo().filter((r) => r === "Timeout").length` (an experimental API): about 10,000 are still running, each retaining its `res`. Take a heap snapshot to see them ([L27](../07-production/27-memory-and-gc.md)), then switch to the hub's single timer.
4. **Two instances, no bus.** Run two API processes on ports 3001 and 3002 with the hub fed only in-process. Connect a stream to 3001 and create a payment through 3002: nothing arrives. Add the LISTEN bridge: it arrives. Then kill instance 3001's LISTEN backend with `SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE query LIKE 'LISTEN%'`, create two payments while it's reconnecting, and confirm the catch-up `pump()` delivers both.

### Explain out loud (60–120 seconds)

1. What changes when a response lasts hours, and the five failure modes it creates.
2. The SSE endpoint: headers, `id:` as sequence, subscribe-then-replay, heartbeats, the named-event trap.
3. Why fan-out never awaits a client, and why eviction is lossless with resume.
4. LISTEN/NOTIFY: IDs not payloads, delivered on commit, not durable, cursors for catch-up.
5. The WebSocket handshake, what `ws` handles, and when sticky sessions are actually needed.

---

## Module 4 complete — checkpoint

- [ ] TCP has no message boundaries; framing by delimiter and by length prefix (L16)
- [ ] What `ECONNRESET` and `EPIPE` mean; half-open connections and `allowHalfOpen` (L16)
- [ ] `socket.setTimeout` only emits `'timeout'`: you destroy the socket yourself (L16)
- [ ] Nagle and `setNoDelay`; a database driver is framing plus pooling plus pipelining, as RESP showed (L16)
- [ ] The `node:http` anatomy, and why the handler runs before the body exists (L17)
- [ ] A body limit: `Content-Length` check, streaming count, 413 with `Connection: close` (L17)
- [ ] `headersTimeout`, `requestTimeout`, `keepAliveTimeout`: what each defends, and that none limits the response (L17)
- [ ] The load-balancer 502 race, drawn, and the ordering fix (L17)
- [ ] `res` `'close'` + `!writableFinished` → an `AbortSignal` (L17)
- [ ] Why you never hand-parse HTTP: request smuggling (L17)
- [ ] Dispatchers and per-origin pools; a connection cap plus a visible limiter; client idle below server idle (L18)
- [ ] The four outbound timeouts, why `bodyTimeout` isn't a total, and a deadline on every call (L18)
- [ ] The unconsumed-body leak, and why it's intermittent (L18)
- [ ] `TIME_WAIT` and the ephemeral-port arithmetic (L18)
- [ ] SSRF validated in the dispatcher's `lookup`, plus IP literals and redirects; DNS rebinding (L18)
- [ ] Express: an array, an index, arity; what Express 5 catches and what it still doesn't (L19)
- [ ] Fastify: radix routing, compiled validation and serialization, encapsulation, the ajv defaults (L19)
- [ ] NestJS mapped onto Spring, and the three places the analogy breaks (L19)
- [ ] Little's law as the argument about framework performance (L19)
- [ ] SSE: framing, heartbeats, `Last-Event-ID` resume, subscribe before replay (L20)
- [ ] Slow-consumer eviction, and why resume makes it lossless (L20)
- [ ] LISTEN/NOTIFY fan-out: IDs not payloads, delivered on commit, not durable, catch-up from cursors (L20)
- [ ] The WebSocket handshake, what `ws` handles, and when sticky sessions are needed (L20)

---

## What's next

Module 4 gave you every way bytes enter and leave a Node process. Module 5 is about doing more than one thing at a time: first, worker threads, Node's real parallelism for CPU-bound work like Ledger's reconciler, with rules very unlike Java's threads.

Next → **[Lesson 21: Worker threads](../05-concurrency/21-worker-threads.md)**
