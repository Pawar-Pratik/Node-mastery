# Lesson 14 — Composing streams: Transform, generators & Web Streams

> **Why this lesson exists:** real pipelines don't just copy bytes. They split, parse, validate, batch and route bad records somewhere useful, and every one of those stages has to keep the backpressure contract from Lesson 13. Async generators make each stage a ten-line function that `pipeline` can compose. Meanwhile Web Streams have arrived through `fetch`, edge runtimes and the browser, so you need to bridge both worlds. "How would you parse a 2 GB CSV in Node?" is a standard interview question, and this lesson is the answer.

**Time:** ~80 minutes · **Prereq:** Lesson 13

---

## 1. The idea in one sentence

> **A pipeline stage is a function from an async iterable to an async iterable: write each one as an async generator, connect them with `pipeline`, and backpressure, error propagation and cancellation come with it.**

---

## 2. Why composition matters: a 2 GB payout file

Ledger merchants pay out to thousands of recipients by uploading a CSV:

```text
recipient_id,amount_minor,currency,reference
rcp_8f2k1m9q,250000,inr,"Invoice 2026-10, batch A"
rcp_x71b4e0z,99900,inr,Refund ₹999
```

A large merchant's file is 2 GB, around 30 million rows. Between the socket and Postgres the data changes shape four times:

```text
  upload socket        splitLines()        parseCsv()          validate()          batch(1000)        insertBatches()
  ───────────▶ bytes ─────────────▶ lines ───────────▶ fields ───────────▶ rows ──────────▶ rows[] ───────────▶ Postgres
   Buffer chunks       strings             string[]            PayoutRow   │       PayoutRow[1000]    1 txn per batch
   (16–64 KiB)                                                             │
                                                                           └──▶ rejects.csv (side output)
```

The naive versions each fail differently:

| Attempt | What breaks |
|---|---|
| `express.text({ limit: "3gb" })` then `split("\n")` | 2 GB string in memory; V8's string cap is about 512 MiB, so it can't even be built |
| `req.on("data", c => c.toString().split("\n").forEach(insertRow))` | Lines are cut at chunk boundaries, `₹` is split across chunks, and `'data'` doesn't wait for your inserts |
| `for await (const line of lines) await insertRow(line)` | Correct, but 30M round trips to Postgres. Hours instead of minutes |
| Everything in one big `Transform` class | Works, but it's untestable, and every concern is tangled with stream mechanics |

The right shape is **small stages, each with one job, connected by `pipeline`**. Each stage is testable on a plain array, and the composition inherits the backpressure from [Lesson 13](13-streams-and-backpressure.md): a slow `INSERT` pauses the upload socket.

---

## 3. Implementing a Transform

The classic way to write a stage is a `Transform`. You implement two callbacks:

```ts
import { Transform, type TransformCallback } from "node:stream";

const upper = new Transform({
  transform(chunk: Buffer, _enc: BufferEncoding, cb: TransformCallback) {
    cb(null, chunk.toString("utf8").toUpperCase());   // push one output and signal "ready for more"
  },
  flush(cb: TransformCallback) {                       // input ended: emit anything you held back
    cb(null, "\n-- end --\n");
  },
});
```

The rules:

- **Call `cb` exactly once per chunk.** Calling it twice fails the stream with `ERR_MULTIPLE_CALLBACK`. Never calling it stalls the pipeline forever.
- **`cb(err)` fails the stream.** `pipeline` then destroys everything. Never `throw` from inside an async callback in a Transform, because nothing catches it.
- **Emit zero, one or many outputs.** `this.push(x)` as many times as you like, then `cb()`. A line splitter pushes many lines per chunk; a filter pushes none.
- **`flush` runs once at the end**: the place to emit a buffered remainder.
- **Object mode per side:** `readableObjectMode: true` emits objects, and `writableObjectMode: true` accepts them. Bytes → lines needs object mode on the readable side only.

**A Transform applies backpressure for you, even when one chunk produces many outputs.** When its readable side is at or over its hwm, it withholds your `cb` from its writable side, so it doesn't accept the next chunk until downstream reads. You can push 500 lines from one chunk safely; the overshoot is bounded by one chunk's worth.

---

## 4. Splitting lines: chunk boundaries never align

Chunks are cut wherever the kernel and the stream buffers happen to cut them. They don't respect lines, and they don't respect characters:

```text
  chunk 1                                   chunk 2
  ┌──────────────────────────────────────┐ ┌──────────────────────────────────┐
  │ ...,inr,Invoice 7\r\nrcp_x71b4,99900,i│ │nr,Refund ₹999\r\nrcp_...          │
  └──────────────────────────────────────┘ └──────────────────────────────────┘
                     line ends ▲    ▲ this line continues in the next chunk
  and "₹" is 3 bytes in UTF-8 (E2 82 B9): a chunk can end after E2 or after E2 82
```

The naive `chunk.toString().split("\n")` makes two mistakes. It treats the fragment at the end of each chunk as a complete line, so you get one broken record per chunk boundary, and it decodes each chunk alone, so a split `₹` becomes `�` mojibake ([L12](12-buffers-and-encodings.md)). The fix has three parts: **keep the trailing fragment** for the next chunk, **decode with a `StringDecoder`**, which holds back incomplete characters, and **flush both at the end**.

```ts
// src/ingest/split-lines.ts
import { StringDecoder } from "node:string_decoder";

export function splitLines({ maxLine = 64 * 1024, maxBytes = Infinity } = {}) {
  return async function* (source: AsyncIterable<Uint8Array>): AsyncGenerator<string> {
    const decoder = new StringDecoder("utf8");      // never splits a multi-byte character
    let tail = "", bytes = 0;
    for await (const chunk of source) {
      if ((bytes += chunk.length) > maxBytes) throw new Error(`upload exceeds ${maxBytes} bytes`);
      const parts = (tail + decoder.write(chunk)).split("\n");
      tail = parts.pop() ?? "";                     // the last piece may be half a line
      if (tail.length > maxLine) throw new Error(`line longer than ${maxLine} chars`);
      for (const line of parts) yield line.endsWith("\r") ? line.slice(0, -1) : line;
    }
    tail += decoder.end();
    if (tail.length > 0) yield tail;                // files often lack a final newline
  };
}
```

Three details that matter:

- **`\r\n` is handled by stripping a trailing `\r`**, which also works when `\r` and `\n` land in different chunks.
- **`maxLine` is a security control.** A 2 GB file with no newline would otherwise grow `tail` until the process dies. Any parser of untrusted input needs a cap on every buffer it grows.
- **`maxBytes` enforces the upload limit while streaming.** A `Content-Length` check up front is cheaper, but chunked uploads don't send one.

**`readline` is the built-in alternative:**

```ts
import { createInterface } from "node:readline";
for await (const line of createInterface({ input: req, crlfDelay: Infinity })) { /* … */ }
```

| | `readline` | Your `splitLines` stage |
|---|---|---|
| Correctness on chunk splits and `\r\n` | ✅ (`crlfDelay: Infinity` treats `\r\n` as one break) | ✅ |
| Input errors | Rejects the `for await` in current Node | Rejects through `pipeline` |
| Maximum line length | **None**: a file with no newlines buffers until OOM | Your cap |
| Backpressure | Coarse: it pauses the input once about a thousand lines are queued (an implementation detail) | Exact: pull-based |
| Fits in `pipeline` as a stage | No; you wrap it | Yes |

`readline` is fine for trusted, line-oriented files like logs or fixtures. For untrusted uploads, use a stage with caps. **Neither one parses real CSV**: RFC 4180 lets a quoted field contain a newline, so any line splitter breaks that record in two. Ledger's payout format forbids embedded newlines and rejects such rows. If you must accept arbitrary spreadsheet exports, use a streaming CSV parser such as `csv-parse`, which tracks quote state across chunks.

---

## 5. Async generators as pipeline stages

`pipeline` accepts functions as stages:

```ts
await pipeline(
  source,                                          // a Readable, an iterable, or a generator function
  async function* (input, { signal }) {            // a transform stage
    for await (const x of input) yield f(x);
  },
  async function (input, { signal }) {             // the destination: an async function
    for await (const x of input) await save(x);
    return "done";                                 // pipeline resolves with this value
  },
);
```

**Pull-based, one item at a time.** A generator chain hands values along by `yield`: stage 2 asks stage 1 for its next value only when it's ready for one. Tracing a three-stage chain with a slow sink shows each item travelling the whole chain before the next one starts, while only the source Readable reads a little ahead into its own buffer. **Backpressure is just "nobody calls `next()`."** Memory in flight is the source's buffer plus about one item per stage.

**Errors and cleanup.** A `throw` in any stage rejects the `pipeline` and destroys every stream in it. When a downstream stage fails, upstream generators are closed, so their `finally` blocks run. That makes `finally` inside a generator the right place to release a stage's own resources.

**Stage factories, not stages with options.** `pipeline` calls each stage as `fn(source, { signal })`, so the second parameter is taken. Configuration goes in a factory that returns the stage, which is why the code above is `splitLines({ maxLine: 4096 })` and not `splitLines(source, { maxLine })`. A factory also gives each pipeline fresh state (`tail`, `bytes`).

**`Readable.from(iterable)`** turns any iterable or async iterable into a Readable. It's in **object mode by default**, and a string isn't split into characters: `Readable.from("abc")` emits one chunk, `"abc"`. It's mostly useful in tests: `Readable.from([buf1, buf2])` feeds a stage the exact chunk boundaries you want to test.

**Two TypeScript frictions with `@types/node`:**

- Its `pipeline` overloads type **at most four transform stages** between source and destination. Node accepts more, but TypeScript rejects the call. Merge stages, or group some with `compose` (§8).
- The stage function types don't include the `{ signal }` argument. Declare it optional (`opts?: { signal?: AbortSignal }`) and the stage typechecks.

**Async generator or Transform?**

| | Async generator stage | `Transform` |
|---|---|---|
| Code | A `for await` loop: reads like ordinary code | Callbacks, `push`, `cb`, `flush` |
| Async work per item | Natural: `await` inside the loop | Possible, but you juggle `cb` timing |
| Errors | `throw` | `cb(err)`, and a stray `throw` in async code escapes |
| Testing | Call it on an array: `for await (const x of stage(arr))` | Needs a stream harness |
| Reusable as a stream object | Only via `Duplex.from(fn)` | It already is one |
| Raw throughput on tiny chunks | Pays a promise per item; batch to amortise | Slightly cheaper per chunk |

Default to generators inside `pipeline`. Write a `Transform` when you need a reusable stream object to hand to APIs that expect one, like `toCsv()` in [Lesson 13](13-streams-and-backpressure.md).

---

## 6. Parse, validate, batch

**Parsing** turns a line into fields. Ledger's format allows quoted fields with commas and doubled quotes:

```ts
// src/ingest/parse-csv.ts
export type Fields = { lineNo: number; raw: string; fields: string[] | null };   // null = unparseable

export function parseCsvLine(line: string): string[] | null {
  const out: string[] = [];
  let field = "", i = 0, quoted = false;
  while (i < line.length) {
    const ch = line[i++];
    if (quoted) {
      if (ch !== '"') field += ch;
      else if (line[i] === '"') { field += '"'; i++; }   // "" inside quotes is a literal quote
      else quoted = false;
    } else if (ch === '"' && field === "") quoted = true;
    else if (ch === ",") { out.push(field); field = ""; }
    else field += ch;
  }
  if (quoted) return null;                               // unterminated quote
  out.push(field);
  return out;
}

export function parseCsv() {
  return async function* (lines: AsyncIterable<string>): AsyncGenerator<Fields> {
    let lineNo = 0;
    for await (const raw of lines) {
      lineNo++;
      if (lineNo === 1 || raw === "") continue;          // header and blank lines
      yield { lineNo, raw, fields: parseCsvLine(raw) };
    }
  };
}
```

**Validating** applies the schema at the boundary ([TS L15](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/15-the-boundary-and-parsing.md) covers zod). Note that `amountMinor` stays a string: it goes straight into a `bigint` column, and never through a JavaScript number ([L02](../01-foundations/02-the-javascript-node-assumes.md)). The stage's rejects routing is in §7.

```ts
import { z } from "zod";

export const PayoutRow = z.object({
  lineNo: z.number().int(),
  recipientId: z.string().regex(/^rcp_[A-Za-z0-9]{8,32}$/),
  amountMinor: z.string().regex(/^[1-9][0-9]{0,17}$/),   // positive, fits in bigint, no float
  currency: z.enum(["inr", "usd", "eur", "gbp"]),
  reference: z.string().max(140),
});
export type PayoutRow = z.infer<typeof PayoutRow>;
```

**Batching** groups rows so you pay one database round trip per thousand:

```ts
// src/ingest/batch.ts
export function batch<T>(size: number) {
  return async function* (source: AsyncIterable<T>): AsyncGenerator<T[]> {
    let buf: T[] = [];
    for await (const item of source) {
      buf.push(item);
      if (buf.length === size) { yield buf; buf = []; }   // suspends here until the insert is done
    }
    if (buf.length > 0) yield buf;                        // the final partial batch
  };
}
```

**Why batching keeps backpressure:** the `yield` suspends the generator until the sink asks for the next batch, which happens only after the previous transaction commits. While Postgres is busy, every stage above sits suspended, the upload Readable's buffer fills to its hwm, Node stops reading from the socket, TCP's window closes, and **the merchant's upload slows to the speed of your database**. Memory in flight is at most about one batch per stage plus a socket buffer, whether the file is 2 MB or 2 GB.

Batching a live, trickling source needs one more rule: flush a partial batch after a time limit, or the last few rows wait forever. Files end, so the ingester doesn't need it, but a Kafka or queue consumer does.

**The insert** sends a batch as arrays and lets Postgres unnest them. That's one statement with six parameters instead of 6,000, which also stays clear of the protocol's 65,535-parameter limit:

```ts
// src/ingest/insert.ts
import { withTenantTx } from "../lib/db.ts";             // L25: BEGIN; SET LOCAL app.merchant_id; COMMIT

const INSERT = `
  INSERT INTO payout_items (file_id, line_no, recipient_id, amount_minor, currency, reference)
  SELECT $1::text, * FROM unnest($2::int[], $3::text[], $4::bigint[], $5::text[], $6::text[])
  ON CONFLICT (file_id, line_no) DO NOTHING`;

export function insertBatches(merchantId: string, fileId: string, progress: Progress) {
  return async function (batches: AsyncIterable<PayoutRow[]>, opts?: { signal?: AbortSignal }) {
    for await (const rows of batches) {
      opts?.signal?.throwIfAborted();
      await withTenantTx(merchantId, async (tx) => {
        const r = await tx.query(INSERT, [fileId, rows.map((x) => x.lineNo), rows.map((x) => x.recipientId),
          rows.map((x) => x.amountMinor), rows.map((x) => x.currency), rows.map((x) => x.reference)]);
        await tx.query("UPDATE payout_files SET rows_inserted = rows_inserted + $2, last_line_no = $3 WHERE id = $1",
          [fileId, r.rowCount ?? 0, rows.at(-1)!.lineNo]);
      });
      progress.inserted += rows.length;
    }
  };
}
```

Each design choice is load-bearing. **One transaction per batch** commits the rows and the progress counter together, so `payout_files` never claims rows that aren't there. **`ON CONFLICT (file_id, line_no) DO NOTHING`** makes a re-run after a crash safe: the ingester can be killed at any point and restarted from the beginning, which is crash-only design ([L09](../02-event-loop/09-errors-and-crashing.md)). For raw speed, `COPY … FROM STDIN` through `pg-copy-streams` is faster still, but it makes per-row conflict handling harder, so use it for staging tables.

---

## 7. Side outputs: routing bad rows

A 30-million-row file always has bad rows. Failing the whole upload for one typo is useless, and silently dropping rows is worse. Ledger writes each reject to `rejects.csv` with its line number and reason, and the merchant downloads it.

A side output is a second Writable that one stage writes to. **It needs the same backpressure discipline as the main path, plus error handling of its own:**

```ts
// src/ingest/validate.ts
import { once } from "node:events";
import type { Writable } from "node:stream";

export function validate(rejects: Writable, progress: Progress) {
  return async function* (source: AsyncIterable<Fields>, opts?: { signal?: AbortSignal }): AsyncGenerator<PayoutRow> {
    for await (const { lineNo, raw, fields } of source) {
      const [recipientId, amountMinor, currency, reference] = fields ?? [];
      const parsed = fields?.length === 4
        ? PayoutRow.safeParse({ lineNo, recipientId, amountMinor, currency, reference })
        : undefined;
      if (parsed?.success) { yield parsed.data; continue; }

      progress.rejected++;
      const reason = parsed ? parsed.error.issues[0]?.message ?? "invalid" : "expected 4 fields";
      if (rejects.destroyed) throw rejects.errored ?? new Error("rejects output closed");
      if (!rejects.write(`${lineNo},${JSON.stringify(reason)},${JSON.stringify(raw)}\n`)) {
        await once(rejects, "drain", { signal: opts?.signal });   // side outputs have backpressure too
      }
    }
  };
}
```

Three failure modes are covered:

- **The side output is slow.** `write()` returns `false` and the stage waits for `'drain'`, which stalls the main path too. That's correct: a bounded system has to slow down somewhere.
- **The side output dies** (disk full, `EACCES`). The `destroyed` check stops writing into a dead stream, and the caller (§10) aborts the whole pipeline when the side output fails.
- **The pipeline is cancelled while waiting.** The `signal` makes `once` reject instead of waiting for a `'drain'` that will never come ([L13](13-streams-and-backpressure.md) showed that hang).

---

## 8. `compose`, `Duplex.from` and iterator helpers

Three newer APIs package the same ideas:

- **`Duplex.from(asyncGeneratorFn)`** wraps a generator stage as a real Duplex stream, for APIs that insist on a stream object.
- **`stream.compose(a, b, c)`** bundles several stages into one Duplex. Write is the first stage's input, read is the last stage's output. It's useful for shipping `parsePayoutFile = compose(splitLines(), parseCsv(), validate(...))` as one reusable unit, and for the `@types/node` four-stage limit. It was **experimental for most of the Node 24 line** and was marked stable only in a late 24.x release, so check the docs for your exact version before relying on it.
- **Iterator helpers on Readable**: `readable.map(fn, { concurrency })`, `.filter()`, `.take()`, `.toArray()` and friends are **still experimental**. `map` with `concurrency: 4` is a neat way to run four async calls in flight while keeping backpressure, but don't build production code on an experimental API. A bounded-concurrency stage from [L10](../02-event-loop/10-cancellation-and-concurrency.md) does the same job.

---

## 9. Web Streams, and when to use which

Node ships the WHATWG Streams standard as globals: `ReadableStream`, `WritableStream`, `TransformStream`, plus `TextDecoderStream` and `CompressionStream`. They're the same API as in the browser, Deno, Bun and edge runtimes.

| | Node streams | Web Streams |
|---|---|---|
| Read side | `for await`, `'data'`, `pipe` | `getReader().read()`, `for await` (async iterable in Node), `pipeTo` / `pipeThrough` |
| Backpressure signal | `write()` → `false`, `'drain'` | `writer.desiredSize`, `await writer.ready` |
| Default buffer | 16 objects; 64 KiB of bytes (16 KiB on Windows) | **One chunk** (the default queuing strategy counts chunks, `highWaterMark: 1`) |
| Errors | `'error'` event; `pipeline` rejects | Promises reject; the stream becomes `errored` |
| Where you meet them | `fs`, `net`, `http`, `zlib`, `pg`, almost every npm stream | **`fetch` response bodies**, `Request` in Hono and edge handlers, `CompressionStream` |

The bridges are static methods, **stable in Node 24**:

```ts
import { Readable, Writable } from "node:stream";

const nodeReadable = Readable.fromWeb(webReadable);     // a fetch body → a Node Readable
const webReadable2 = Readable.toWeb(nodeReadable);      // a Node Readable → for a Web API or Hono
const webWritable = Writable.toWeb(fileWriteStream);
```

`pipeline` also accepts Web Streams directly, so you can stream a fetched file into the ingester stages without converting it:

```ts
const res = await fetch(settlementUrl, { signal });
if (!res.ok || !res.body) { await res.body?.cancel(); throw new Error(`download failed: ${res.status}`); }
await pipeline(res.body, splitLines({ maxLine: 4096 }), parseCsv(), countRows, { signal });
```

If TypeScript says the global `fetch` body's `ReadableStream` isn't assignable to the `node:stream/web` one, that's a known mismatch between the two type definitions in `@types/node`. Cast at that one line; the runtime objects are the same. The `res.body?.cancel()` on the error path matters: an unconsumed body holds the connection ([L18](../04-networking/18-outbound-http.md)).

**Which to use:** inside Node, **use Node streams.** They're what `fs`, `net`, `zlib` and the database drivers speak, they've had a decade of performance work, and `pipeline` composes them. Use Web Streams **at the boundaries** where an API hands you one (`fetch`), or in code that must run unchanged on an edge runtime. Convert once, at the edge of your code, and don't ping-pong between the two inside a pipeline.

---

## 10. Ledger: the payout-file ingester

The upload endpoint takes the CSV as a raw body (`Content-Type: text/csv`), so **no body-parser runs on this route**. A JSON or text parser would try to buffer 2 GB.

```ts
// src/routes/payout-files.ts  —  PUT /v1/payout-files/:id/content
import type { Request } from "express";
import { createWriteStream } from "node:fs";
import { finished, pipeline } from "node:stream/promises";
import { log } from "../lib/log.ts";

export type Progress = { inserted: number; rejected: number };
const MAX_BYTES = 2 * 1024 ** 3;

export async function ingestPayoutFile(req: Request, fileId: string, rejectsPath: string) {
  const progress: Progress = { inserted: 0, rejected: 0 };
  const failed = new AbortController();
  const rejects = createWriteStream(rejectsPath, { flags: "wx" });
  const rejectsDone = finished(rejects);
  rejectsDone.catch((err) => failed.abort(err));     // side output died → stop everything
  const ticker = setInterval(() => log.info({ fileId, ...progress }, "ingest progress"), 5_000);
  try {
    await pipeline(
      req,                                           // the upload socket, as a Readable
      splitLines({ maxLine: 4096, maxBytes: MAX_BYTES }),
      parseCsv(),
      validate(rejects, progress),
      batch<PayoutRow>(1000),
      insertBatches(req.merchantId, fileId, progress),
      { signal: AbortSignal.any([failed.signal, AbortSignal.timeout(60 * 60_000)]) },
    );
    rejects.end();
    await rejectsDone;                               // flushed to disk, or throws
    return progress;
  } finally {
    clearInterval(ticker);
    if (!rejects.writableFinished) rejects.destroy();
  }
}
```

Read it as a list of guarantees:

- **Constant memory.** About one batch per stage, plus the socket and file buffers, regardless of the file's size.
- **Backpressure from the database to the uploader.** A slow `INSERT` suspends `batch`, which suspends every stage above it and stops Node reading the socket. The client's upload slows down rather than your heap filling up.
- **Every failure stops everything.** A thrown error in any stage, a client disconnect (`req` closes early), the one-hour deadline, or the rejects file dying all reject the `pipeline` and destroy every stage.
- **`rejectsDone.catch(…)` is attached immediately.** If the rejects file fails while the pipeline is still running, its rejection must already have a handler, or Node treats it as unhandled and kills the process. That's the late-catch trap from [L06](../02-event-loop/06-promises-and-microtasks.md). The `await rejectsDone` later still sees the error.
- **`{ flags: "wx" }`** fails if the file already exists, so two ingests can't interleave into one rejects file. Where that file lives, and how it's cleaned up on abort, is [Lesson 15](15-the-file-system.md).
- **Restart-safe.** Each batch commits atomically with its progress row, and `ON CONFLICT DO NOTHING` makes a full re-run after a crash harmless.

---

## 11. Production rules

| Rule | Why |
|---|---|
| **One job per stage; compose with `pipeline`** | Each stage is testable on an array; `pipeline` adds backpressure, errors and cancellation |
| **Write stages as factories returning async generators** | `pipeline` owns the second argument (`{ signal }`); factories also give each run fresh state |
| **Carry a remainder across chunks, and decode with `StringDecoder`** | Chunks split lines and multi-byte characters at arbitrary points |
| **Cap every buffer an untrusted input can grow** (line length, total bytes, frame size) | A file with no newline is a memory DoS against a naive splitter |
| **Never parse quoted CSV with a line splitter alone** | RFC 4180 allows newlines inside quoted fields |
| **Batch database writes; one transaction per batch** | 30M round trips take hours; the batch transaction keeps rows and progress consistent |
| **Make the batch insert idempotent (`ON CONFLICT DO NOTHING` on a natural key)** | A crashed ingest can simply be re-run |
| **Give side outputs backpressure, a destroyed check and an abort hook** | Otherwise a slow or dead rejects file leaks memory, hangs or crashes the process |
| **Attach `.catch` to a side output's `finished()` promise immediately** | A rejection with no handler yet kills the process |
| **No body-parser on streaming upload routes** | It buffers the whole body before your handler runs |
| **Node streams inside Node; Web Streams only at the edges** | Node's ecosystem speaks Node streams; convert once, not per stage |
| **Don't ship experimental stream APIs (`readable.map`, and `compose` on older 24.x)** | Their behaviour can still change in a minor release |

---

## 12. Interview traps

**Q1. "How would you parse a 2 GB CSV upload in Node?"**
Stream it, never buffer it: `pipeline(req, splitLines(), parseCsv(), validate(), batch(1000), insertBatches())`, with each stage an async generator. Memory stays at about a batch per stage. Backpressure runs from the database to the upload socket, so a slow insert slows the upload instead of filling the heap. Bad rows go to a side output with line numbers rather than failing the file, and batch inserts are idempotent so a crash means a re-run. The weak answer is "use a CSV library" with no mention of backpressure, batching or bad rows.
**Follow-up:** "What if fields can contain newlines?" Then a line splitter is wrong by construction; use a streaming CSV parser that tracks quote state across chunks.

**Q2. "How do you split a stream into lines correctly?"**
Keep the trailing fragment of each chunk and prepend it to the next, because chunk boundaries don't align with lines. Decode with a `StringDecoder` (or `TextDecoder` with `stream: true`), because they don't align with UTF-8 characters either. Strip a trailing `\r` for CRLF files, emit the remainder at the end, and cap the remainder's length so a newline-free file can't exhaust memory. `readline` does the first parts correctly but has no line-length cap.

**Q3. "Async generators vs Transform streams?"**
Both work inside `pipeline`. Generators read like ordinary code: `for await`, `await`, `throw` and `finally` all behave normally, they test against plain arrays, and they're pull-based, so backpressure is implicit. Transforms are callback-based (`push`, `cb`, `flush`) and are real stream objects you can hand to APIs that want one. Default to generators. Write a Transform for a reusable stream object, or wrap a generator with `Duplex.from`.

**Q4. "How do you batch inside a pipeline without losing backpressure?"**
Accumulate in a generator and `yield` the array when it's full. The `yield` suspends the generator until the consumer asks for the next batch, which it does only after the previous insert finishes, so nothing upstream runs ahead. Flush the partial batch at the end. For live sources, also flush on a timer. The mistake to name is collecting into an array and firing inserts without awaiting them: that's unbounded concurrency, and memory grows with the input.

**Q5. "Node streams vs Web Streams?"**
Node streams are what `fs`, `net`, `http`, `zlib` and the database drivers speak: event-based, with `write()` returning `false` and `'drain'`, and default buffers of 64 KiB (16 KiB on Windows) or 16 objects. Web Streams are the WHATWG standard shared with browsers and edge runtimes: promise-based, with `desiredSize` and `writer.ready`, and a default buffer of one chunk. You meet them in `fetch` bodies. Use Node streams inside Node, bridge with `Readable.fromWeb`/`toWeb` at the boundary, and remember that `pipeline` accepts Web Streams directly.

**Q6. "A stage throws halfway through. What happens to the other stages?"**
`pipeline` rejects with that error and destroys every stream in the chain: the source socket or file is closed, and generators are closed, so their `finally` blocks run. That's the reason to put a stage's own cleanup in `finally`. With `pipe()` chains, none of that happens.

**Q7. "Where can a side output go wrong?"**
Three ways. It can be slow: if you ignore `write()` returning `false`, the rejects accumulate in memory. It can die: writing to a destroyed stream doesn't help, and its `'error'` with no listener crashes the process. And a `'drain'` you're waiting for may never come once it's destroyed, so the wait needs a signal. The fix is to honour `write()`, check `destroyed`, observe it with `finished()` (handled immediately) and abort the main pipeline when it fails.

**Q8. "Why does `pipeline` pass `{ signal }` to stages, and what do you do with it?"**
So a stage can stop its own async work when the pipeline is aborted, by a deadline, a client disconnect or another stage's failure. Pass it to anything awaitable inside the stage (`once(..., { signal })`, `fetch`, a database query that accepts one) or call `signal.throwIfAborted()` between batches. Without it, a stage waiting on an external event can outlive the pipeline that owns it.

---

## 13. Build & break

### Build — the `ingest/` stages

Create `src/ingest/` with `splitLines()`, `parseCsv()`, `validate()`, `batch()` and `insertBatches()` exactly as above, and `ingestPayoutFile()` wiring them together. Then:

1. **Unit-test each stage on arrays.** For `splitLines`, feed `Readable.from([...])` with chunks cut at every byte position of `"a,b\r\nc,₹\r\nd"` and assert you always get the same three lines. For `batch(3)`, feed 7 items and assert `[3, 3, 1]`.
2. **Generate a 2 GB file:** write 30M rows with `fs.createWriteStream` in a loop that respects `write()`'s return value (a fitting test of [L13](13-streams-and-backpressure.md)), with one bad row in every 100,000.
3. **Upload it and watch memory:** `curl.exe -T payouts.csv -H "Content-Type: text/csv" -H "Authorization: Bearer $KEY" http://localhost:3000/v1/payout-files/pf_1/content`, logging `rss` every second. It stays flat.
4. **Prove the backpressure:** add a 200 ms sleep inside `insertBatches` and watch curl's upload speed fall to about 1,000 rows every 200 ms. The database is now setting the pace of the upload.

### Break — 4 experiments

1. **Naive split.** Replace `splitLines` with `async function* (src) { for await (const c of src) yield* c.toString().split("\n") }`. Ingest the 2 GB file and compare `progress.lines` with `wc -l payouts.csv`. The counts differ, and `rejects.csv` fills with half-lines at every chunk boundary and `�` wherever a `₹` was cut. Restore `splitLines` and the counts match.
2. **The collecting stage.** Write a stage that does `const all = []; for await (const r of src) all.push(r); yield* all;` before `batch`. RSS climbs with the file until the process dies, and nothing is inserted until it's read the whole file. Remove it, and memory is flat again.
3. **Throw mid-stream.** Make `validate` throw on line 5,000,000. Confirm that `pipeline` rejects, that `req.destroyed` is true (the upload connection is closed: curl reports an error), that a `finally` with a log line inside `batch` ran, and that `payout_files.rows_inserted` matches the rows actually in `payout_items`. Remove the throw and re-run the full file: the count ends exactly at the file's valid-row total, thanks to `ON CONFLICT`.
4. **Mismatched modes.** Put a stage that yields objects (`{ line }`) directly in front of a byte-mode `fs.createWriteStream`. `pipeline` rejects with `ERR_INVALID_ARG_TYPE` (*The "chunk" argument must be of type string or an instance of Buffer…*). Fix it with a stage that serializes each object to a string first.

### Explain out loud (60–120 seconds)

1. The four shape changes from bytes to batches, and why each stage is separate.
2. Why chunk boundaries break naive line splitting, and the three-part fix.
3. Why an async generator stage is pull-based, and how `yield` in `batch` carries backpressure from Postgres to the upload socket.
4. How the side output keeps backpressure and how its failure stops the main pipeline.
5. Node streams vs Web Streams, and where Ledger uses each.

---

## What's next

The ingester writes a rejects file, the exporter could write statements to disk before uploading them, and both need temporary files that get cleaned up when things fail. Next: the file system, honestly. You'll write files atomically so a crash can't leave a truncated statement, survive `EMFILE` and Windows' locked files, and stop path traversal before it reaches `fs`.

Next → **[Lesson 15: The file system, honestly](15-the-file-system.md)**
