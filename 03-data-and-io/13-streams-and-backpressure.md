# Lesson 13 — Streams & backpressure

> **Why this lesson exists:** streams are how Node moves unbounded data through bounded memory, and backpressure is the contract that makes that work. Ignore it and nothing fails in testing. Then a merchant exports 4 million payments over a slow connection and the pod gets OOMKilled. This is the second most important lesson in the track: "what is backpressure?" and "why `pipeline` over `pipe`?" are on every senior Node interview list, and the Ledger exporter depends on getting both right.

**Time:** ~100 minutes · **Prereq:** Lessons 11, 12

---

## 1. The idea in one sentence

> **A stream is a bounded buffer between a producer and a consumer, and backpressure is the producer agreeing to stop when that buffer is full. Every streaming memory incident is someone breaking that agreement.**

---

## 2. Why streams exist: the export that took down the pod

Here's the first version of Ledger's statement export. It works on the developer's 200-row test merchant:

```ts
// ❌ Fine at 200 rows. Fatal at 4,000,000.
app.get("/v1/exports/payments.csv", async (req, res) => {
  const { rows } = await pool.query("SELECT id, created_at, status, amount_minor, currency FROM payments");
  const csv = rows.map(toCsvLine).join("\n");     // one giant string
  res.type("text/csv").send(csv);
});
```

Follow the memory for a merchant with 4M payments:

1. **`pg` materialises every row** as a JavaScript object before `await` returns. Four million objects, each with five properties plus a `Date`, live on the V8 heap at once, comfortably past a gigabyte.
2. **`join` builds one string** from all of them. V8 caps a string at `buffer.constants.MAX_STRING_LENGTH` (536,870,888 characters on 64-bit, about 512 MiB), so a large enough export dies with `RangeError: Invalid string length` even before the heap runs out.
3. **Nothing reaches the client** until the whole query and the whole string exist. Time to first byte equals total time.
4. **While this runs, the process is unhealthy.** Building the string blocks the loop ([L08](../02-event-loop/08-blocking-the-loop.md)), and the GC thrashes on a heap near its limit, so every other request in the pod pays.

The streamed version keeps a few thousand rows in memory, no matter how many the merchant has:

| | Buffered | Streamed |
|---|---|---|
| **Memory** | O(rows): grows with the merchant | **O(buffer sizes)**: about 1,000 rows plus a few 64 KiB buffers, flat |
| **Time to first byte** | After the full query and serialization | After the first batch of rows |
| **A slow client** | Holds the whole result in memory until the client finishes | Pauses the query |
| **Failure mode** | OOM, or `Invalid string length` | A slow download (you'll fix that in §10) |

Streams buy you three things: **constant memory**, **early first bytes**, and **composition**, because every stage speaks the same contract, so a cursor, a CSV encoder, gzip and a socket snap together.

---

## 3. The four types

Every stream is an EventEmitter ([L11](11-eventemitter.md)), so `'data'`, `'end'`, `'finish'`, `'drain'`, `'error'` and `'close'` are ordinary events with ordinary rules: `emit` is synchronous, and an `'error'` with no listener crashes the process.

| Type | Role | You've used it as |
|---|---|---|
| **Readable** | A source you read from | `fs.createReadStream`, `req` (an `IncomingMessage`), `process.stdin`, a `pg-query-stream` |
| **Writable** | A sink you write to | `fs.createWriteStream`, `res` (a `ServerResponse`), `process.stdout` |
| **Duplex** | Both, and **independent**: what you write isn't what you read | `net.Socket` ([L16](../04-networking/16-tcp-and-net.md)), a TLS socket |
| **Transform** | A Duplex whose output is computed from its input | `zlib.createGzip()`, `crypto.createHash` used as a stream, your CSV encoder |

Streams run in one of two modes. **Byte mode** (the default) carries `Buffer`s or strings, and the buffer is measured in bytes. **Object mode** carries any JavaScript value and is measured in items. A Transform can be object mode on one side and byte mode on the other: Ledger's CSV encoder takes row objects and emits bytes.

---

## 4. The internal buffer and `highWaterMark`

Every Readable and every Writable owns a buffer. `highWaterMark` (hwm) is the size at which the stream **asks** its producer to stop.

```text
                  write(chunk)                         _write() takes one chunk at a time
  producer ─────────────────────▶ ┌──────────────────────────────┐ ───────────────────▶ fd / socket
                                  │ ██████████████░░░░░░░░░░░░░░ │
                                  └──────────────────────────────┘
                                  0          length       hwm
                     write() returns  (length < hwm)  ← after queuing this chunk
```

**It's a threshold, not a limit.** Watch:

```ts
import { Writable } from "node:stream";

const w = new Writable({ highWaterMark: 16, write(_chunk, _enc, _cb) { /* never finishes */ } });
w.write(Buffer.alloc(15));   // true:  length 15 < 16
w.write(Buffer.alloc(1));    // false: length 16, "please stop"
w.write(Buffer.alloc(100));  // false, and the 100 bytes were ACCEPTED: length is now 116
```

`false` means "I'm full; wait for `'drain'`." **Nothing enforces it.** The stream keeps queuing whatever you hand it, which is exactly why ignoring it is a memory leak instead of an error.

The Readable side mirrors it. The implementer calls `push(chunk)`, which returns `false` once the readable buffer reaches hwm. Node then stops calling `_read()` until a consumer drains the buffer below hwm. That pause is how a file read, a socket read or a database cursor stops.

**The defaults:**

| Stream | Default hwm |
|---|---|
| Byte streams | **64 KiB since Node 22 on Linux and macOS; 16 KiB on Windows**, where Node kept the old value |
| Object mode | **16 objects** |
| `fs.createReadStream` | 64 KiB on every platform (its own default) |
| `pg-query-stream` | `batchSize` rows if you set it, else 100 |

You're on Windows, so your experiments will show 16,384, while the Linux container in production uses 65,536. Check rather than assume: `node -p "require('node:stream').getDefaultHighWaterMark(false)"`. `stream.setDefaultHighWaterMark()` changes it process-wide; tune per stream instead.

---

## 5. Consuming a Readable: paused, flowing, and `for await`

A Readable starts **paused**: data accumulates in its buffer (up to hwm) and waits for someone to `read()` it. It becomes **flowing** when you attach a `'data'` listener, call `resume()`, or `pipe()` it. Then it emits chunks as fast as they arrive.

| `readable.readableFlowing` | Meaning |
|---|---|
| `null` | Nobody has consumed it yet; nothing flows |
| `true` | Flowing: chunks go out as `'data'` events |
| `false` | Explicitly paused (`pause()`, or `pipe` backing off) |

Flowing mode has a trap that catches almost everyone once:

```ts
// ❌ 'data' doesn't wait for your promise. 4M rows → 4M concurrent INSERTs.
rows.on("data", async (row) => {
  await insertAuditRow(row);
});
```

`emit` is synchronous and ignores return values ([L11](11-eventemitter.md)), so the stream fires the next `'data'` immediately. The stream's buffer stays small while your pending promises grow without bound, along with the pool's wait queue. **Backpressure didn't fail. You opted out of it.**

**The modern consumer is async iteration**, and it's pull-based:

```ts
// ✅ The next chunk is read only after the loop body finishes.
for await (const row of rows) {
  await insertAuditRow(row);       // slow body → the buffer fills → the source pauses
}
```

Three properties make `for await` the default way to consume a Readable:

- **Backpressure is automatic.** While your body awaits, nothing is read. The buffer fills to hwm and the source stops.
- **Errors become exceptions.** A stream `'error'` rejects the iteration, so `try/catch` works.
- **Leaving early cleans up.** `break`, `return` or a `throw` in the loop body calls `destroy()` on the stream, which closes the file descriptor or database cursor.

---

## 6. The backpressure loop, end to end

This is what happens when a merchant downloads the export over a slow mobile connection. Read it bottom-up as "who stops whom":

```text
  client stops reading
        │   its TCP receive buffer fills; it advertises a zero window
        ▼
  kernel send buffer fills ──────────── libuv can't write; socket.writableLength grows
        │
        ▼
  res.write() returns false ─────────── pipeline stops reading from gzip
        │
        ▼
  gzip's readable buffer reaches hwm ── gzip.write() returns false
        │
        ▼
  toCsv's readable side is full ─────── the Transform withholds its callback
        │
        ▼
  QueryStream's buffer is full ──────── push() returned false; _read() isn't called
        │
        ▼
  pg-cursor sends no more fetches ───── Postgres leaves the portal suspended

  Memory in flight ≈ the sum of the buffers: ~1,000 rows + a few 16–64 KiB buffers.
```

When the client reads again, the whole chain unwinds upwards: `'drain'` fires on `res`, pipeline resumes gzip, and the cursor fetches the next batch. **Nobody wrote a line of flow-control code.** Each stream honoured the contract, and the slowest consumer set the pace for the whole chain.

If you write to a Writable by hand, the contract is yours to honour:

```ts
import { once } from "node:events";
import { finished } from "node:stream/promises";
import type { Writable } from "node:stream";

// ✅ The correct manual loop. Use pipeline instead whenever you can.
async function writeAll(out: Writable, lines: AsyncIterable<string>, signal: AbortSignal) {
  for await (const line of lines) {
    if (!out.write(line)) await once(out, "drain", { signal });   // wait until it has room
  }
  out.end();
  await finished(out);                                           // flushed, or rejects on error
}
```

That `{ signal }` isn't decoration. **If `out` is destroyed without an error** (a client disconnect closes `res` silently), `'drain'` never fires and `once` waits forever. `once` rejects on `'error'`, not on `'close'`. That's one more reason to prefer `pipeline`, which watches `'close'` on every stream.

> **Spring equivalent:** Reactive Streams makes the same contract explicit: the subscriber calls `request(n)` and the publisher may send at most n items ([SpringBoot L18](https://github.com/Pawar-Pratik/SpringBoot/blob/main/05-web/18-mvc-vs-webflux-virtual-threads.md)). Node does it with a buffer threshold (`write()` → `false`, then `'drain'`) on the push side, and with `for await` on the pull side. Same idea, different signal.

---

## 7. What happens when you ignore it

```ts
// ❌ Never checks write()'s return value.
for await (const row of rows) {
  res.write(toCsvLine(row));
}
res.end();
```

Against a fast local client this looks fine. Against a client that has stopped reading, the cursor keeps fetching (nothing tells it to stop), every line lands in `res`'s buffer, and `res.writableLength` grows towards the size of the whole export, as fast as Postgres can produce rows.

Where that memory lives matters. Queued chunks are `Buffer`s, which sit **outside the V8 heap** ([L12](12-buffers-and-encodings.md)). So `heapUsed` looks healthy, `rss` climbs, and the kernel kills the container with exit 137 and no JavaScript error at all ([L27](../07-production/27-memory-and-gc.md)). **Ignored backpressure shows up as an OOMKill with a clean-looking heap.**

---

## 8. `pipe()` vs `pipeline()`

`src.pipe(dest)` implements backpressure correctly: it pauses `src` when `dest.write()` returns `false` and resumes it on `'drain'`. **It does nothing about errors or cleanup.** Verified on Node 24:

| Situation | `a.pipe(b).pipe(c)` | `await pipeline(a, b, c)` |
|---|---|---|
| `b` errors | `b` is destroyed. **`a` stays open and paused** with a full buffer; `c` is never ended | Every stream is destroyed; the promise rejects with `b`'s error |
| `c` closes early (client disconnect) | `a` is unpiped and paused **forever**: not destroyed, so its fd or cursor leaks | Rejects with `ERR_STREAM_PREMATURE_CLOSE`; every stream is destroyed |
| Error handling | You need an `'error'` listener on **each** stream, or the process crashes | One `try/catch` |
| Cancellation | None | `{ signal }` destroys everything and rejects with `AbortError` |
| Async generator stages | No | Yes ([L14](14-composing-streams.md)) |

```ts
// ❌ Backpressure yes. Errors and cleanup no. The QueryStream leaks on any failure.
rows.pipe(toCsv()).pipe(createGzip()).pipe(res);

// ✅ Backpressure, error propagation, destruction of every stage, cancellation.
import { pipeline } from "node:stream/promises";
await pipeline(rows, toCsv(), createGzip(), res, { signal });
```

In Ledger, "`a` stays open" means **a pooled Postgres client is never released.** The default pool has 10 clients and waits forever for a free one (`connectionTimeoutMillis: 0`, [L25](../06-data-and-jobs/25-databases-from-node.md)), so after ten failed exports the eleventh request of any kind that needs the database hangs. `pipe()` turns a CSV bug into an outage.

Two companions from `node:stream/promises`:

- **`finished(stream)`** resolves when one stream is done (`'end'` or `'finish'`) and rejects on an error or a premature close. Use it for a stream you're driving by hand, like `writeAll` above.
- The callback form `pipeline(a, b, cb)` from `node:stream` still exists. Prefer the promise form in new code.

---

## 9. `destroy()`, errors and `autoDestroy`

`stream.destroy(err?)` tears a stream down immediately. It calls the implementation's `_destroy()`, which releases the underlying resource (closes the fd, closes the cursor, frees the socket), emits `'error'` if you passed one, and then **always emits `'close'` last**. Nothing is emitted after `'close'`. Writing to a destroyed stream fails with `ERR_STREAM_DESTROYED`.

`autoDestroy` is `true` by default: a stream destroys itself after `'end'`/`'finish'` or after an error. That's why a file stream closes its descriptor without you calling `close()`.

`_destroy` is where resource ownership lives. `pg-query-stream`'s is essentially this:

```ts
_destroy(err, cb) {
  this.cursor.close((closeErr) => cb(closeErr || err));   // tell Postgres to drop the portal
}
```

So "destroy every stream on failure", which is what `pipeline` does, means "close the cursor". Code that never destroys a stream never closes it.

**`ERR_STREAM_PREMATURE_CLOSE`** means a stream emitted `'close'` before it finished. On an HTTP response, it almost always means the client hung up. That's an operational event: log it at `info` with the merchant and row count, and don't page anyone.

---

## 10. Network backpressure, and its bill

Backpressure crosses the network through **TCP flow control** ([API L02](https://github.com/Pawar-Pratik/API-mastery/blob/main/01-foundations/02-journey-of-a-request.md) covers the protocol). The receiver advertises how much buffer space it has left. When an app stops reading, its kernel's receive buffer fills and it advertises a zero window. The sender's kernel stops transmitting, its send buffer fills, libuv's writes stop completing, and Node's socket buffer grows past hwm. At that point `res.write()` returns `false`. **A slow phone on 3G pauses your Postgres cursor**, three machines away, without either side knowing about the other.

```ts
res.writableLength      // bytes queued in Node, not yet handed to the kernel
res.writableNeedDrain   // true after write() returned false and before 'drain'
```

**The bill: a paused stream still holds its resources.** A paused export holds an open transaction, a pooled Postgres client and a socket for as long as the client takes. Ten merchants downloading over slow links can hold Ledger's whole 10-client pool, and every other endpoint starts waiting. Backpressure protected your memory and moved the cost to your connection pool. The production answers:

- A **separate, small pool** for exports (`max: 4`), so exports can never starve payments.
- **One concurrent export per merchant**, and a global cap ([L10](../02-event-loop/10-cancellation-and-concurrency.md)); return 429 or 503 beyond it.
- A **deadline**: `AbortSignal.timeout()` on the pipeline, so an abandoned download can't hold a client for hours.
- Past a size threshold, an **async export job** that writes to object storage and hands back a link ([API L08](https://github.com/Pawar-Pratik/API-mastery/blob/main/02-rest-design/08-collections-and-pagination.md) explains why an export isn't pagination). The streamed endpoint stays for the common case.

---

## 11. Ledger: the statement export

`GET /v1/exports/payments.csv`: a Postgres cursor → CSV → gzip → the response, at constant memory, cancelled the moment the client disconnects.

```ts
// src/routes/exports.ts
import type { Request, Response } from "express";
import { Transform } from "node:stream";
import { pipeline } from "node:stream/promises";
import { createGzip } from "node:zlib";
import QueryStream from "pg-query-stream";
import { exportPool } from "../lib/db.ts";

type PaymentRow = { id: string; created_at: Date; status: string;
  amount_minor: string;                       // bigint arrives as a string: write it as-is, never Number() it
  currency: string; description: string | null };

const HEADER = "id,created_at,status,amount_minor,currency,description\n";
const SQL = `SELECT id, created_at, status, amount_minor, currency, description
               FROM payments ORDER BY created_at, id`;           // RLS scopes it to the merchant

export function csvField(v: string | null): string {
  if (v === null) return "";
  const s = /^[=+\-@\t\r]/.test(v) ? `'${v}` : v;                 // neutralise spreadsheet formulas
  return /[",\r\n]/.test(s) ? `"${s.replaceAll('"', '""')}"` : s;   // RFC 4180 quoting
}

export function toCsv(): Transform {
  let first = true;
  return new Transform({
    writableObjectMode: true,                                     // rows in, bytes out
    transform(row: PaymentRow, _enc, cb) {
      if (first) { first = false; this.push(HEADER); }
      cb(null, [row.id, row.created_at.toISOString(), row.status,
        row.amount_minor, row.currency, csvField(row.description)].join(",") + "\n");
    },
  });
}

export async function exportPayments(req: Request, res: Response): Promise<void> {
  const client = await exportPool.connect();
  let failure: unknown;
  try {
    await client.query("BEGIN");
    await client.query("SELECT set_config('app.merchant_id', $1, true)", [req.merchantId]);
    const rows = client.query(new QueryStream(SQL, [], { batchSize: 1000 }));

    res.setHeader("Content-Type", "text/csv; charset=utf-8");
    res.setHeader("Content-Encoding", "gzip");
    res.setHeader("Content-Disposition", 'attachment; filename="payments.csv"');

    await pipeline(rows, toCsv(), createGzip(), res, { signal: AbortSignal.timeout(30 * 60_000) });
    await client.query("COMMIT");
  } catch (err) {
    failure = err;
    if ((err as NodeJS.ErrnoException).code !== "ERR_STREAM_PREMATURE_CLOSE") throw err;
    // The client hung up. pipeline already destroyed every stage, including the cursor.
  } finally {
    client.release(failure !== undefined);    // true → destroy this connection instead of pooling it
  }
}
```

The decisions, each one an interview answer:

- **One `SELECT`, one snapshot.** A single statement sees a consistent snapshot, so the CSV can't skip or duplicate rows the way offset pagination does mid-export. The cost is a long transaction, which holds back vacuum's cleanup horizon for its duration. For very large merchants, run exports against a replica.
- **`BEGIN` + `set_config(..., true)`** is `SET LOCAL`: the tenant setting dies with the transaction, so it can't leak to the next user of this pooled client ([L25](../06-data-and-jobs/25-databases-from-node.md), [API L15](https://github.com/Pawar-Pratik/API-mastery/blob/main/03-security/15-authorization-and-multitenancy.md)).
- **`batchSize: 1000`** sets both the rows per fetch and the QueryStream's hwm. Bigger batches mean fewer round trips and more memory in flight.
- **The disconnect needs no extra code.** `res` emits `'close'` before `'finish'`, so `pipeline` rejects with `ERR_STREAM_PREMATURE_CLOSE` and destroys the QueryStream, whose `_destroy` closes the cursor.
- **`release(true)` on failure.** A connection that died mid-cursor is in an unknown state, so destroy it rather than hand it to the next request. Destroying it also rolls back the transaction.
- **Errors after the headers are sent can't become a 500.** Express's default error handler closes the connection when `res.headersSent` is true, and Ledger's error middleware must do the same. The client sees a truncated download: chunked encoding never sent its final empty chunk, and the gzip trailer is missing. `curl` reports the transfer as incomplete.
- **`created_at.toISOString()`** writes UTC ISO-8601, and `amount_minor` stays a string end to end, so the merchant gets the exact bigint ([L02](../01-foundations/02-the-javascript-node-assumes.md)).
- **No `Content-Length`**, so Node uses chunked transfer encoding ([L17](../04-networking/17-http-from-scratch.md)). zlib runs on the libuv pool ([L07](../02-event-loop/07-the-thread-pool.md)) and buffers internally, so the first compressed bytes leave after zlib has gathered enough input.

> **Spring equivalent:** `StreamingResponseBody` for the response ([SpringBoot L14](https://github.com/Pawar-Pratik/SpringBoot/blob/main/05-web/14-servlet-and-dispatcherservlet.md)) plus a JDBC fetch size or a `Stream<T>` query with `HINT_FETCH_SIZE` for the cursor ([SpringBoot L23](https://github.com/Pawar-Pratik/SpringBoot/blob/main/06-data/23-transactions-deep-dive.md)). The difference: a blocking `OutputStream.write` stalls a thread when the client is slow, while Node pauses the stream and the thread keeps serving everyone else.

---

## 12. Production rules

| Rule | Why |
|---|---|
| **Stream anything whose size the client controls** | Buffered size scales with the merchant; streamed size scales with your buffers |
| **Connect stages with `pipeline`, never `pipe`** | `pipe` leaks the source (a cursor, an fd, a pooled client) on any error or disconnect |
| **Consume with `for await`, not `'data'` + async handler** | `'data'` doesn't wait for promises, so it silently opts out of backpressure |
| **When writing by hand, `await once(out, "drain", { signal })` on `false`** | The only way to honour the contract manually; the signal saves you from the silent-close hang |
| **Every pipeline gets a deadline (`{ signal }`)** | A paused stream holds a transaction, a pool client and a socket until something ends it |
| **Give long streams their own small pool and a per-tenant cap** | Backpressure moves slow clients' cost into your connection pool |
| **Treat `ERR_STREAM_PREMATURE_CLOSE` as an operational event** | It's a client hanging up; log it at info with context, don't alert |
| **Release a client that failed mid-stream with `release(true)`** | Its protocol state is unknown; don't hand it to the next request |
| **Never `Number()` a bigint column on its way to CSV** | Values above 2^53 − 1 silently lose precision |
| **Escape CSV per RFC 4180 and neutralise leading `= + - @`** | Merchant-supplied text opened in a spreadsheet can execute formulas |
| **Watch `rss`, not just `heapUsed`, when testing streams** | Queued Buffers live off-heap; ignored backpressure shows up only in RSS |

---

## 13. Interview traps

**Q1. "What is backpressure?"**
It's the signal that flows upstream when a consumer is slower than its producer, telling the producer to stop until the consumer catches up, so memory stays bounded. In Node it's concrete: `write()` returns `false` once the Writable's buffer reaches `highWaterMark`, `'drain'` says "resume", and on the Readable side `push()` returning `false` stops Node calling `_read()`. The weak answer, "it slows the producer down", misses that **nothing enforces it**. Every producer has to cooperate, and `pipeline` and `for await` are how you get cooperation for free.
**Follow-up:** "Where does it come from at the end of the chain?" From TCP flow control: a client that stops reading shrinks its receive window to zero, the server's send buffer fills, and Node's socket starts returning `false`.

**Q2. "What happens if you ignore `write()` returning `false`?"**
The write is still accepted and queued, so the buffer grows without bound: by the size of the entire export, if the consumer is slow enough. Because the queued chunks are Buffers, the growth is off-heap: `heapUsed` looks fine while `rss` climbs until the container is OOMKilled with exit 137 and no JavaScript stack. That disconnect between heap and RSS is what makes the bug hard to diagnose.
**Follow-up:** "How would you spot it in production?" `writableLength` on the destination, and RSS growing during exports while the heap stays flat.

**Q3. "`pipe` vs `pipeline`?"**
Both implement backpressure. `pipe` doesn't propagate errors or clean up. If a middle stream fails, the source stays open and paused, and if the destination closes early, the source is unpiped and left paused forever. Either way you leak whatever the source owns (a descriptor, a cursor, a pooled DB client), and you need an `'error'` listener on every stream or the process crashes. `pipeline` destroys every stream on failure, rejects with the first error, waits for `'close'`, takes a `signal`, and accepts async generator stages.
**Follow-up:** "What does `pipeline` do when an HTTP client disconnects mid-download?" `res` closes before finishing, so `pipeline` rejects with `ERR_STREAM_PREMATURE_CLOSE` and destroys every stage, which closes the cursor.

**Q4. "Flowing vs paused mode?"**
Paused (the default) holds data in the buffer until someone calls `read()`. Flowing pushes chunks out as `'data'` events as fast as they arrive. Attaching `'data'`, calling `resume()` or `pipe()` switches to flowing. The trap is `on("data", async (x) => await save(x))`: `emit` ignores the returned promise, so you get unbounded concurrent saves. Modern code uses `for await`, which is pull-based. It reads the next chunk only after the body finishes, turns stream errors into exceptions, and destroys the stream if you `break`.

**Q5. "How do you stream 4M rows as CSV to a client in constant memory, and stop the query if they disconnect?"**
A server-side cursor (`pg-query-stream` over one `SELECT`) as the Readable, an object-to-bytes Transform for CSV, `createGzip()`, and `res`, all connected with `pipeline`. Memory is bounded by the buffers: a batch of rows plus a few kilobytes per stage. A slow client pauses the cursor through backpressure. A disconnect makes `res` close early, so `pipeline` rejects with a premature close and destroys the QueryStream, whose `_destroy` closes the cursor, and you release the client in a `finally`.
**Follow-up:** "What does that cost you?" A paused export holds a transaction and a pool client for as long as the client takes, so give exports their own small pool, a per-merchant cap and a deadline.

**Q6. "Is `highWaterMark` a limit or a threshold?"**
A threshold. It's the buffer level at which `write()` starts returning `false` (or `push()` does on a Readable). Writes past it are still accepted. The defaults are 16 objects in object mode and, for byte streams, 64 KiB since Node 22 except on Windows, which kept 16 KiB. Raising it trades memory for fewer, larger I/O operations. Lowering it doesn't cap anything unless every producer honours the signal.

**Q7. "How does backpressure cross the network?"**
Through TCP's receive window. The receiving application stops reading, its kernel buffer fills, and it advertises a zero window. The sender's kernel stops sending and its send buffer fills. libuv's writes stop completing, Node's socket buffer passes hwm, and `res.write()` returns `false`. `pipeline` stops pulling from upstream, and the pause propagates stage by stage to the source, in Ledger's case a Postgres portal that stops being fetched. No application-level protocol is involved.

**Q8. "After a few slow downloads, the whole API hangs, but memory and CPU are fine. Why?"**
Backpressure did its job: each slow download paused its stream, but each paused stream still holds a pooled Postgres client and an open transaction. A handful of slow clients holds the whole pool, and `pg`'s default `connectionTimeoutMillis: 0` makes every other request wait forever. The fix is isolation and bounds: a dedicated small pool for exports, one export per merchant, a deadline on the pipeline, and an async export job for the largest merchants.

**Q9. "The export returned 200, but the merchant's file is truncated. How?"**
The status and headers went out with the first byte. A failure after that (the database connection dropped, a timeout, a bug in the CSV encoder) can't become a 500, so the server can only destroy the connection. The client gets a chunked response with no terminating chunk and, with gzip, a missing trailer. Good clients detect that (`curl` reports the transfer as incomplete); naive ones save a partial file. Make it detectable: let the error middleware destroy the socket rather than write a body, and consider a final trailer row with the row count so the merchant can verify the file.

**Q10. "What does `destroy()` do, and why is `'close'` the event to trust?"**
`destroy(err)` calls `_destroy` to release the underlying resource, emits `'error'` if given one, and always emits `'close'` last. Nothing comes after it. `'end'` (a Readable finished) and `'finish'` (a Writable flushed) only happen on success, while `'close'` happens on every path, including errors and aborts. That's why cleanup keyed to `'close'`, or to `pipeline`/`finished`, which use it, never leaks.

---

## 14. Build & break

### Build — `GET /v1/exports/payments.csv`

1. Seed a merchant with 4M payments, as the `postgres` superuser (superusers bypass RLS):
   ```sql
   INSERT INTO payments (id, merchant_id, amount_minor, currency, status, description, created_at)
   SELECT 'pi_' || g, '<merchant-uuid>', 1 + (random() * 100000)::bigint, 'inr', 'succeeded',
          'order ' || g, now() - g * interval '1 second'
   FROM generate_series(1, 4000000) AS g;
   ```
2. Implement §11 with an `exportPool` (`max: 4`) separate from the main pool.
3. Log memory every second while you test:
   ```ts
   setInterval(() => {
     const m = process.memoryUsage(), mb = (n: number) => (n / 2 ** 20).toFixed(0);
     console.log(`rss=${mb(m.rss)}MB heap=${mb(m.heapUsed)}MB external=${mb(m.external)}MB`);
   }, 1000).unref();
   ```
4. Download it fast, then slowly, and confirm that RSS stays flat in both runs and that the slow one doesn't fetch ahead:
   ```bash
   curl.exe -H "Authorization: Bearer $KEY" -o export.csv.gz http://localhost:3000/v1/exports/payments.csv
   curl.exe --limit-rate 100k -H "Authorization: Bearer $KEY" -o slow.csv.gz http://localhost:3000/v1/exports/payments.csv
   ```
5. Verify the file: `gzip -dc export.csv.gz | wc -l` prints 4,000,001 (the rows plus the header).

### Break — 4 experiments

1. **The buffered export.** Swap in §2's version (`await pool.query(...)` then `join`). Watch `heap` climb by gigabytes, every other endpoint's latency spike while it serializes, and the process end in either V8's "heap out of memory" or `RangeError: Invalid string length`, depending on row width. Restore the stream and watch `heap` stay flat for the same 4M rows.
2. **Ignore `write()`.** Replace the pipeline with the `for await … res.write(line)` loop from §7 (with gzip removed) and download with `--limit-rate 50k`. `external` and `rss` climb by hundreds of megabytes while `heap` barely moves. Restore backpressure by adding `if (!res.write(line)) await once(res, "drain", { signal })` and re-run: memory is flat.
3. **`pipe()` with a failing middle stage.** Rewrite the handler pipe-era style: connect the stages with `.pipe()`, add an `'error'` listener to each so the process survives, and release the client in `rows.on("end", …)`, which is where that style of code always put it. Make `toCsv` call `cb(new Error("bad row"))` at row 10,000 and log `exportPool.totalCount`, `idleCount` and `waitingCount` after each request. `'end'` never fires on a failed export, so after four failures the pool is exhausted and the fifth export hangs forever. Switch back to `pipeline` and repeat: every failure releases its client.
4. **The disconnect nobody noticed.** With the pipe-style handler again (no failing row this time), start a download and press Ctrl+C on curl after a second. In `psql`, run `SELECT pid, state, query FROM pg_stat_activity WHERE query LIKE '%FROM payments%';`. The export's backend is still there with your `SELECT` (its `state` depends on how far the cursor got, but the row doesn't go away), and the pool client is still checked out, because the QueryStream was unpiped and paused, not destroyed. Switch to `pipeline`, repeat, and the backend leaves within milliseconds of the disconnect.

### Explain out loud (60–120 seconds)

1. Why the buffered export dies: row objects on the heap, the string length cap, time to first byte.
2. `highWaterMark` as a threshold: `write()` returning `false`, `'drain'`, `push()` returning `false`.
3. The backpressure chain from a slow client's TCP window to a suspended Postgres portal.
4. Exactly what `pipe` leaks and what `pipeline` destroys, including on client disconnect.
5. Why a paused stream still costs you a pool client, and the four ways Ledger bounds that.

---

## What's next

You can move bytes from one place to another at constant memory. Real pipelines also reshape the data: bytes into lines, lines into records, records into batches of 1,000 for the database. Next you'll build those stages as async generators, connect them with `pipeline`, and bridge to the Web Streams that `fetch` hands you.

Next → **[Lesson 14: Composing streams: Transform, generators & Web Streams](14-composing-streams.md)**
