# Lesson 21 — Worker threads

> **Why this lesson exists:** CPU-bound work on the main thread blocks every request in the process, and [Lesson 08](../02-event-loop/08-blocking-the-loop.md) showed you how to prove it. Worker threads are the only way to run *your JavaScript* in parallel inside one Node process, and their rules are the opposite of Java's: nothing is shared unless you share it, and every message is copied unless you transfer it. Interviewers use them to test whether you can tell I/O concurrency from CPU parallelism, and whether you know what parallelism costs.

**Time:** ~80 minutes · **Prereq:** Lessons 05, 08

---

## 1. The idea in one sentence

> **A worker thread is a second V8 isolate with its own heap and event loop inside the same process: it buys you CPU parallelism, it shares nothing by default, and every message you send it is copied, so pool your workers, keep messages small, and transfer anything big.**

---

## 2. The CPU wall, and what workers are for

The event loop runs each callback to completion ([L05](../02-event-loop/05-the-event-loop.md)). A handler that spends 800 ms parsing a file holds the only JavaScript thread for 800 ms, and every other request waits behind it:

```text
main thread ─┬─[ req A: parse 800 ms ████████████████ ]─[B]─[C]─[D]─▶
             │   B, C, D arrived at t=10ms and waited ~790 ms each
```

[L08](../02-event-loop/08-blocking-the-loop.md) gave you the fixes by layer. Partitioning with `setImmediate` makes the loop *fair* (other callbacks get turns), but the work still runs on one core, so it finishes no sooner. **When the work is genuinely CPU-bound and has to happen, the fix is another thread running JavaScript.** That's a worker.

| Work | Worker thread? | Why |
|---|---|---|
| Parsing/validating a 500 MB CSV, diffing large datasets, matching | ✅ | Pure JS CPU time |
| Pure-JS hashing, compression or image transforms of big inputs | ✅ | Pure JS CPU time |
| Rendering a huge template or PDF in JS | ✅ | Pure JS CPU time |
| `await db.query()`, `fetch()`, reading files | ❌ | Already off-thread: kernel or libuv pool ([L07](../02-event-loop/07-the-thread-pool.md)) |
| `crypto.scrypt`, async `zlib` | ❌ | Already on the libuv pool; wrapping them adds a copy and nothing else |
| Native addons that manage their own threads | ❌ | They're already parallel |

**The test is ELU, not intuition:** if event-loop utilization goes to ~1 and loop delay climbs while the work runs ([L08](../02-event-loop/08-blocking-the-loop.md)), it's CPU-bound on your thread. If ELU stays low and the work is still slow, you're waiting on I/O, and a worker will only add copying.

---

## 3. What a worker actually is

```text
 one OS process (one PID, one RSS, one memory limit)
┌───────────────────────────────────────────────────────────────┐
│  main thread               worker 1             worker 2      │
│  ┌──────────────────┐   ┌──────────────┐   ┌──────────────┐   │
│  │ V8 isolate + heap│   │ isolate+heap │   │ isolate+heap │   │
│  │ event loop       │   │ event loop   │   │ event loop   │   │
│  │ modules, globals │   │ OWN modules  │   │ OWN modules  │   │
│  └────────┬─────────┘   └──────┬───────┘   └──────┬───────┘   │
│           └──── MessagePort ───┴──── MessagePort ─┘           │
│  shared: the libuv thread pool · file descriptors · native    │
│  memory · SharedArrayBuffers you explicitly hand over         │
└───────────────────────────────────────────────────────────────┘
```

Each worker boots a complete Node environment: an isolate with its own heap and GC, its own event loop, and **its own copy of every module it imports**. A `pg.Pool` created in a worker is a separate pool with separate connections; a `Map` cache in a module is a different `Map` in every thread. `process.env` is a copy. `process.exit()` inside a worker ends that thread only, with that exit code, and the process carries on.

What *is* shared is less obvious and matters more. **The libuv pool is per process**, so a worker's `fs` and async-crypto calls queue on the same 4 threads as the main thread's ([L07](../02-event-loop/07-the-thread-pool.md)). And everything lands in one RSS against one container limit ([L27](../07-production/27-memory-and-gc.md)).

```ts
// hash.worker.ts
import { parentPort, workerData } from "node:worker_threads";
import { createHash } from "node:crypto";

const port = parentPort;
if (!port) throw new Error("hash.worker.ts must run as a worker");
const { rounds } = workerData as { rounds: number };

port.on("message", (data: string) => {
  let digest = data;
  for (let i = 0; i < rounds; i++) digest = createHash("sha256").update(digest).digest("hex");
  port.postMessage(digest);
});
```

```ts
// main.ts
import { Worker } from "node:worker_threads";

// ❌ new Worker("./hash.worker.ts"): a relative string resolves against process.cwd(), not this file
// ✅ resolve it against this module, so it works wherever the process was started
const worker = new Worker(new URL("./hash.worker.ts", import.meta.url), { workerData: { rounds: 100_000 } });
worker.on("error", (err) => log.error({ err }, "hash worker crashed"));    // never omit this (§6)
worker.postMessage("pi_01HQ8ZK");
worker.once("message", (digest: string) => log.info({ digest }));
```

On Node 24 the worker file can be `.ts`: the worker uses the same loader, so type stripping applies to it too.

**The cost is real:** booting an isolate and a Node environment takes milliseconds to tens of milliseconds and megabytes of memory per worker (measure yours in Break 1). Paying that per request turns a CPU fix into a CPU and memory problem. **Workers are a pool you create at startup, never a per-request resource.**

---

## 4. Messages: copy, transfer, or channel

### Structured clone: the default, and its cost

`postMessage(value)` runs the structured clone algorithm (V8's serializer). **The sending thread serializes synchronously inside `postMessage`, and the receiving thread deserializes before your `'message'` handler runs.** Both are CPU work on those threads' loops, proportional to the size and shape of the value. Posting a 200 MB object graph from the main thread blocks the main loop while it serializes, which is the exact problem you were trying to escape.

| Value | Survives the clone? |
|---|---|
| Primitives, plain objects, arrays | ✅ copied |
| `Map`, `Set`, `Date`, `RegExp`, typed arrays, `ArrayBuffer` | ✅ copied |
| Built-in errors (`Error`, `TypeError`, …) | ✅ name, message, `stack` and `cause` |
| **Your error subclass** (`class LedgerError extends Error { code = … }`) | ❌ arrives as a plain `Error`; **`code` and other own fields are gone** |
| Class instances | ❌ arrive as plain objects; the prototype (and its methods) is lost |
| Functions | ❌ `DataCloneError` thrown by `postMessage` |

The error row bites everyone who builds a pool: the worker throws `LedgerError` with `code: "recon_bad_line"`, the main thread receives `Error` with no code, and your retry logic that branches on `err.code` silently takes the wrong path. **Serialize errors yourself, as `{ name, message, code, stack }`** (§7 does).

### Transfer: move ownership instead of copying

```ts
const bytes = new Uint8Array(200 * 1024 * 1024);
worker.postMessage({ bytes: bytes.buffer }, [bytes.buffer]);   // the transferList
console.log(bytes.byteLength);                                 // 0: the sender's view is detached
```

A transfer hands the `ArrayBuffer`'s backing memory to the other thread in O(1), whatever its size. **The sender's buffer is detached** (its `byteLength` becomes 0), so the bytes exist in exactly one thread. That's the real value: not only speed, but a single owner, so no race is possible. `MessagePort`s and `FileHandle`s are transferable too.

**The pooled-Buffer trap:** a small `Buffer.from(...)` or `Buffer.allocUnsafe(...)` is a view into Node's shared 8 KiB pool ([L12](../03-data-and-io/12-buffers-and-encodings.md)). Node marks that pool untransferable, so putting `smallBuf.buffer` in a transferList throws `DataCloneError`, and even if it didn't, you'd be handing over 8 KiB of other people's bytes. **Transfer only buffers you allocated whole:** `new Uint8Array(n)`, `new ArrayBuffer(n)`, or a Buffer where `byteOffset === 0` and `byteLength === buffer.byteLength`.

### Channels

`new MessageChannel()` gives you two connected ports. Transfer `port2` to a worker and you have a private channel, for example between two workers without routing every message through the main thread. `receiveMessageOnPort(port)` reads one message synchronously, and `BroadcastChannel` fans one message out to every thread that opened the same name. You rarely need more than `parentPort`.

---

## 5. Shared memory: `SharedArrayBuffer` and `Atomics`

A `SharedArrayBuffer` (SAB) is the one way two threads see **the same bytes**: no copy, no transfer, both can read and write. **Shared memory brings back data races**, the thing Node's model otherwise rules out:

```ts
// sab-worker.ts: 4 of these run at once against one Int32Array over a SAB
const counter = new Int32Array(workerData.sab as SharedArrayBuffer);
for (let i = 0; i < 1_000_000; i++) {
  counter[0]!++;                    // ❌ load, add, store: three steps another thread can interleave
  // Atomics.add(counter, 0, 1);    // ✅ one indivisible read-modify-write
}
```

Four workers × 1,000,000 increments with `++` finish far below 4,000,000: updates are lost whenever two threads load the same value before either stores. `Atomics.add` gives exactly 4,000,000. You'll run this in Break 3.

| `Atomics` call | What it does |
|---|---|
| `add`, `sub`, `and`, `or`, `exchange`, `compareExchange` | Indivisible read-modify-write; returns the old value |
| `load`, `store` | Reads and writes with ordering guarantees across threads |
| `wait(arr, i, expected, timeoutMs)` | Sleeps the thread until notified or timed out. **Synchronous**: allowed on Node's main thread, and it blocks the loop exactly like a busy-wait would, so never call it there |
| `notify(arr, i, count)` | Wakes threads waiting on that index |

SABs hold bytes, not objects, so they don't help you share an object graph: you'd serialize into them anyway. **They're worth it for small, hot, numeric state:** progress counters, a ring buffer of samples, or a cancellation flag a long task polls:

```ts
// main thread                                     // worker, inside its hot loop
const cancel = new Int32Array(new SharedArrayBuffer(4));
worker.postMessage({ cancel });                     if (i % 10_000 === 0 && Atomics.load(cancel, 0) === 1) return;
Atomics.store(cancel, 0, 1);    // ask it to stop
```

That flag is the only way to stop a running task *without* killing its thread: JavaScript can't be interrupted, so cancellation inside a worker is cooperative or it's `terminate()`.

**"Are there race conditions in Node?"** Yes, two kinds. Data races on a SAB without `Atomics`. And logical races on the main thread with no shared memory at all: `const b = await getBalance(); await setBalance(b + x)` interleaves with another request at the `await`. The second kind is far more common, and it's why Ledger's balance is an append-only `SUM`, not a read-modify-write.

---

## 6. Lifecycle: errors, exits, limits

| Event on the `Worker` | When |
|---|---|
| `'online'` | The worker started executing |
| `'message'` | It posted something |
| `'error'` | **An uncaught exception inside the worker.** The worker then exits with code 1 |
| `'messageerror'` | A message failed to deserialize |
| `'exit'` | The thread stopped; you get the exit code |

Three facts, each a production bug when you don't know it:

1. **No `'error'` listener on the `Worker` means the whole process crashes.** The worker's uncaught exception is re-emitted as `'error'` on the `Worker` object, which is an EventEmitter, and an unhandled `'error'` throws ([L11](../03-data-and-io/11-eventemitter.md)). One bad input line kills the API process along with the worker.
2. **`await once(worker, "exit")` rejects if `'error'` fires first**, because `events.once` rejects on `'error'`. Wait for exit with `new Promise((r) => worker.on("exit", r))` when you also expect crashes.
3. **`worker.terminate()` stops even a tight synchronous loop** and returns a promise for the exit code. It's the only hard stop for a running task, and it throws away the isolate, so you pay startup again.

`resourceLimits` caps a worker's V8 heap (`maxOldGenerationSizeMb`, `maxYoungGenerationSizeMb`, `codeRangeSizeMb`, `stackSizeMb`). Exceed it and the worker dies with `ERR_WORKER_OUT_OF_MEMORY` while the process survives, which is exactly what you want for parsing merchant-supplied files of unknown shape. **It caps the JS heap only:** Buffers and ArrayBuffers live off-heap and aren't counted ([L27](../07-production/27-memory-and-gc.md)).

**The trap that combines with L27's advice:** if the process starts with `--max-old-space-size` (and in a container it should), that flag **overrides `maxOldGenerationSizeMb` for every worker**. Each worker silently gets the main thread's ceiling, and your 32 MiB cap is gone. Check it inside a worker with `v8.getHeapStatistics().heap_size_limit`. On Node 24.11, a worker given `maxOldGenerationSizeMb: 32` under `--max-old-space-size=384` reports the same limit as the main thread. You can't undo it per worker: `execArgv` doesn't accept V8 flags. So budget memory as **(1 + workers) × the flag**, which is what the [blueprint](../09-project/34-ledger-engine-blueprint.md) does, and bound untrusted inputs by bytes and rows before parsing rather than relying on the heap cap.

**A worker is not a sandbox.** It runs in your process with your environment, your file system access and your credentials. Never run untrusted code in one.

---

## 7. A worker pool from scratch

The design: N workers created up front, a FIFO queue, one task per worker at a time, a numeric task ID on every message, errors propagated as data, an `AbortSignal` per task, crashed workers replaced, and **a bounded queue that rejects instead of growing**, because an unbounded queue in front of a CPU pool is a memory leak with extra steps ([L10](../02-event-loop/10-cancellation-and-concurrency.md)).

```ts
// lib/worker-pool.ts
import { Worker } from "node:worker_threads";
import type { TransferListItem } from "node:worker_threads";

export type SerializedError = { name: string; message: string; code?: string; stack?: string };
type Reply = { id: number; ok: true; result: unknown } | { id: number; ok: false; error: SerializedError };
type Job = {
  id: number; payload: unknown; transfer: readonly TransferListItem[];
  resolve: (value: unknown) => void; reject: (reason: unknown) => void;
  signal?: AbortSignal; onAbort?: () => void;
};
type Slot = { worker: Worker; job: Job | null };

export class WorkerPool {
  #file: URL;
  #slots: Slot[] = [];
  #queue: Job[] = [];
  #maxQueue: number;
  #nextId = 1;
  #closed = false;

  constructor(file: URL, size: number, maxQueue = size * 4) {
    this.#file = file;
    this.#maxQueue = maxQueue;
    for (let i = 0; i < size; i++) this.#slots.push(this.#spawn());
  }

  run<T>(payload: unknown, opts: { transfer?: TransferListItem[]; signal?: AbortSignal } = {}): Promise<T> {
    if (this.#closed) return Promise.reject(new Error("pool is closed"));
    if (opts.signal?.aborted) return Promise.reject(opts.signal.reason);
    if (this.#queue.length >= this.#maxQueue) return Promise.reject(new Error("pool queue full")); // ✅ backpressure
    return new Promise<T>((resolve, reject) => {
      const job: Job = { id: this.#nextId++, payload, transfer: opts.transfer ?? [],
        resolve: resolve as (value: unknown) => void, reject, signal: opts.signal };
      if (job.signal) {
        job.onAbort = () => this.#abort(job);
        job.signal.addEventListener("abort", job.onAbort, { once: true });
      }
      this.#queue.push(job);
      this.#dispatch();
    });
  }

  async close(): Promise<void> {
    this.#closed = true;
    for (const job of this.#queue.splice(0)) this.#settle(job, new Error("pool closed"));
    await Promise.all(this.#slots.map((s) => s.worker.terminate()));   // running jobs fail via 'exit'
  }

  #spawn(): Slot {
    const slot: Slot = { worker: new Worker(this.#file), job: null };
    slot.worker.on("message", (msg: Reply) => {
      const job = slot.job;
      if (!job || job.id !== msg.id) return;            // a late reply for an aborted job
      slot.job = null;
      if (msg.ok) this.#settle(job, null, msg.result);
      else this.#settle(job, Object.assign(new Error(msg.error.message), msg.error));
      this.#dispatch();
    });
    slot.worker.on("error", (err) => this.#retire(slot, err));      // ✅ uncaught throw in the worker
    slot.worker.on("exit", (code) => this.#retire(slot, new Error(`worker exited with code ${code}`)));
    return slot;
  }

  #retire(slot: Slot, reason: unknown): void {           // fail its job, replace the thread
    const i = this.#slots.indexOf(slot);
    if (i === -1) return;                                 // 'error' is always followed by 'exit'
    this.#slots.splice(i, 1);
    if (slot.job) this.#settle(slot.job, reason);
    void slot.worker.terminate();
    if (!this.#closed) { this.#slots.push(this.#spawn()); this.#dispatch(); }
  }

  #abort(job: Job): void {
    const q = this.#queue.indexOf(job);
    if (q !== -1) { this.#queue.splice(q, 1); this.#settle(job, job.signal?.reason); return; }
    const slot = this.#slots.find((s) => s.job === job);
    if (slot) this.#retire(slot, job.signal?.reason);    // running JS can't be interrupted: kill the thread
  }

  #dispatch(): void {
    for (const slot of this.#slots) {
      while (!slot.job && this.#queue.length > 0) {
        const job = this.#queue.shift()!;
        try { slot.worker.postMessage({ id: job.id, payload: job.payload }, job.transfer); slot.job = job; }
        catch (err) { this.#settle(job, err); }           // DataCloneError: a function, a pooled Buffer…
      }
    }
  }

  #settle(job: Job, err: unknown, result?: unknown): void {
    if (job.onAbort) job.signal?.removeEventListener("abort", job.onAbort);   // no leak on long-lived signals
    if (err) job.reject(err); else job.resolve(result);
  }
}
```

The worker side owns the error protocol:

```ts
// reconcile/worker.ts
import { parentPort } from "node:worker_threads";
import { parseSettlement } from "./parse.ts";

const port = parentPort;
if (!port) throw new Error("reconcile/worker.ts must run in a worker thread");

port.on("message", ({ id, payload }: { id: number; payload: { bytes: ArrayBuffer } }) => {
  try {
    port.postMessage({ id, ok: true, result: parseSettlement(new Uint8Array(payload.bytes)) });
  } catch (e) {
    const err = e instanceof Error ? e : new Error(String(e));
    // ❌ postMessage({ error: err }) drops `code` and the subclass; ✅ send the fields you branch on
    port.postMessage({ id, ok: false, error: { name: err.name, message: err.message,
      stack: err.stack, code: (err as { code?: string }).code } });
  }
});
```

**In production, use piscina.** It's this design hardened: min and max threads, idle timeouts that shrink the pool, `maxQueue`, per-task `signal` and `transferList`, and years of edge cases you haven't hit yet. The worker file default-exports a function and `await pool.run(task, { signal, transferList })` returns its result. Build the pool once by hand so you can explain piscina, then use piscina.

---

## 8. Sizing, granularity and measuring the win

- **Threads = the cores you're allowed, not the cores you see.** CPU-bound tasks gain nothing past one thread per core; beyond that the OS just time-slices them and you add context switches. Leave one core for the main thread when it has real I/O to do. Inside a container, the CPU *limit* is what counts, and `os.cpus()` doesn't know it ([L23](23-cluster-and-scaling.md)). **Take the number from config.**
- **Memory = N × (isolate baseline + the task's working set)**, all inside one container limit. Give each worker `resourceLimits` so one pathological input can't take the process down.
- **Granularity:** a task should be milliseconds to tens of milliseconds of CPU. Tiny tasks drown in per-message overhead (two serializations, a thread hop); huge tasks give bad latency and hold memory. Batch lines, rows or images into chunks.
- **Speedup flattens** at your core count and earlier if the main thread has serial work (reading the file, joining results). That's Amdahl's law: the serial part caps you.

**Measure before and after, every time.** Run the [L08](../02-event-loop/08-blocking-the-loop.md) loop meter on the main thread: p99 delay should collapse from "the task's duration" to near zero. `worker.performance.eventLoopUtilization()` tells you whether each worker is actually busy: workers sitting at low ELU mean the main thread can't feed them fast enough.

---

## 9. Ledger: the reconciler on a pool

Every night a processor sends a settlement file: ~500 MB, ~1M lines of `processor_ref,amount_minor,currency,settled_at`. Reconciliation matches each line to the ledger and reports what's missing, extra or mismatched. Parsing and normalising 1M lines is pure CPU; the join against expected entries is cheap `Map` lookups.

```text
settlement.csv ──stream (1 MiB chunks)──► main thread: cut at the last '\n'
                                           │  transfer the ArrayBuffer (zero copy)
                                           ▼
                         WorkerPool (RECONCILE_THREADS, e.g. limit − 1)
                           worker: decode → parse → validate → normalise
                                           │  small records back
                                           ▼
                main thread: join with expected entries (Map by processor_ref,
                             amount_minor as bigint, loaded via a pg cursor)
                                           │
                                           ▼
                   matched · missing · amount_mismatch → report, alerts
```

```ts
// reconcile/run.ts (the core loop)
const pool = new WorkerPool(new URL("./worker.ts", import.meta.url), config.reconcileThreads);
const maxInFlight = config.reconcileThreads * 2;       // enough to keep every thread fed
const inFlight = new Set<Promise<void>>();
let failure: unknown;
let carry = Buffer.alloc(0);

for await (const chunk of createReadStream(file, { highWaterMark: 1 << 20, signal }) as AsyncIterable<Buffer>) {
  if (failure) break;                                   // break destroys the read stream
  const buf = carry.length > 0 ? Buffer.concat([carry, chunk]) : chunk;
  const cut = buf.lastIndexOf(0x0a) + 1;                // only whole lines leave this thread
  carry = Buffer.from(buf.subarray(cut));               // copy the tail: don't pin the whole chunk
  const bytes = new Uint8Array(cut);                    // a fresh ArrayBuffer we own outright...
  bytes.set(buf.subarray(0, cut));
  const task: Promise<void> = pool
    .run<Settled[]>({ bytes: bytes.buffer }, { transfer: [bytes.buffer], signal })  // ...so it can move
    .then((rows) => matcher.add(rows))
    .catch((err: unknown) => { failure ??= err; })
    .finally(() => inFlight.delete(task));
  inFlight.add(task);
  if (inFlight.size >= maxInFlight) await Promise.race(inFlight);   // ✅ backpressure: stop reading the file
}
await Promise.all(inFlight);
if (failure) throw failure;
// the final `carry` (a last line with no '\n') goes through the same path
```

Three decisions worth defending:

1. **Bytes go to the workers, not strings.** Decoding on the main thread would be CPU work on the thread you're protecting, and a string can't be transferred. The worker decodes.
2. **Backpressure is end to end.** At most `2 × threads` chunks are in flight, so when workers fall behind, the `for await` stops pulling, the file stream stops reading, and memory stays flat ([L13](../03-data-and-io/13-streams-and-backpressure.md)).
3. **State lives in one place.** The expected-entries `Map` stays on the main thread. Cloning it into every worker would multiply its memory by N; partitioning it would complicate everything for a join that costs microseconds per line.

**Why this matters even though the reconciler is its own process:** its main thread still serves `/healthz`, extends its job lease ([L26](../06-data-and-jobs/26-background-jobs-and-queues.md)) and streams results to Postgres. Run the matching on that thread and the liveness probe times out, the lease expires and the job runs twice. On the pool, its loop p99 stays flat, which is also the proof you'd need before anyone suggests running it inside the API.

---

## 10. Production rules

| Rule | Why |
|---|---|
| **Use workers for CPU-bound JavaScript only** | I/O is already parallel; a worker adds two serializations and a thread hop |
| **Create a pool at startup; never a worker per request** | Each worker boots an isolate: milliseconds and megabytes every time |
| **Size the pool from config, not `os.cpus()`** | Containers limit CPU below what the machine reports ([L23](23-cluster-and-scaling.md)) |
| **Bound the queue and reject when it's full** | An unbounded queue in front of a CPU pool is a memory leak under load |
| **Always attach `'error'` (and `'exit'`) listeners to every `Worker`** | An unhandled `'error'` crashes the whole process |
| **Serialize errors explicitly as `{ name, message, code, stack }`** | Structured clone drops subclasses and custom fields like `code` |
| **Transfer big binary payloads; send bytes, not object graphs** | Cloning costs CPU on both threads; transfer is O(1) and leaves one owner |
| **Transfer only buffers you allocated whole** | The pooled Buffer is untransferable, and a slice's `.buffer` may hold other data |
| **Set `resourceLimits` on workers that parse untrusted input** | One pathological file kills the worker, not the process |
| **Use `SharedArrayBuffer` only with `Atomics`, and only for small numeric state** | Plain reads and writes race; object graphs don't fit in bytes anyway |
| **Make long tasks cancellable cooperatively; `terminate()` as the last resort** | JS can't be interrupted, and terminating discards a warm isolate |
| **Measure loop p99 before and after** | Proves the main thread got its time back, and shows if workers are starved |

> **Spring equivalent:** `ExecutorService` and `@Async` ([SpringBoot L32](https://github.com/Pawar-Pratik/SpringBoot/blob/main/09-production/32-async-scheduling-events.md)): a bounded pool, a bounded queue and a rejection policy are the same three decisions. The difference is the memory model. Java threads share one heap, so passing an object is free and every shared field needs `synchronized` or `volatile`. Node workers share nothing, so passing an object is a copy and races only exist in a `SharedArrayBuffer`. A Node worker is closer to an actor, or to a process with a fast pipe, than to a Java thread.

---

## 11. Interview traps

**Q1. "When would you use worker threads?"**
For CPU-bound JavaScript that has to run and would otherwise block the event loop: parsing and validating huge files, matching or diffing large datasets, pure-JS compression or image work. The weak answer is "to make Node multi-threaded" or "for heavy I/O": I/O is already concurrent through the kernel and the libuv pool, so a worker only adds copying. **Follow-up:** "How do you know the work is CPU-bound?" ELU near 1 and rising loop delay while it runs; if ELU stays low, you're waiting on I/O.

**Q2. "Worker threads vs child processes vs cluster?"**
Workers are threads in one process: cheapest to start, cheapest to message, can transfer and share memory, but a native crash or OOM takes the whole process. Child processes are separate OS processes: full isolation, any program, killable with signals, but slower to start and message over pipes. Cluster is child processes that share a listening socket, for spreading *connections* across cores. **Workers parallelize a task, cluster parallelizes request handling**, and in Kubernetes you'd usually use replicas instead of cluster ([L23](23-cluster-and-scaling.md)).

**Q3. "How do workers communicate, and what does it cost?"**
`postMessage` over a `MessagePort`, using structured clone: the sender serializes synchronously, the receiver deserializes before the handler runs, so a large message costs CPU on both loops. Transferring an `ArrayBuffer` moves it in O(1) and detaches the sender's copy. A `SharedArrayBuffer` avoids messages entirely but needs `Atomics`.

**Q4. "What is a transferable?"**
An object whose ownership moves between threads instead of being copied: `ArrayBuffer`, `MessagePort`, `FileHandle`. After the transfer the sender's buffer is detached, with `byteLength` 0, so exactly one thread owns the bytes. **Follow-up:** "Why did transferring `buf.buffer` throw?" It was a small pooled Buffer; Node marks the shared pool untransferable. Allocate a fresh `Uint8Array` for anything you'll transfer.

**Q5. "Are there race conditions in Node?"**
Yes. Data races on a `SharedArrayBuffer` written without `Atomics` (four workers doing `++` lose updates). And logical races on a single thread: two requests interleave at an `await` inside a read-modify-write. The weak answer is "no, it's single-threaded"; the interleaving at `await` is the one you'll actually ship.

**Q6. "Why pool workers instead of spawning one per task?"**
Each worker boots an isolate, a heap and a Node environment: milliseconds and megabytes. Per-request spawning turns a CPU problem into a startup-latency and memory problem, and gives you unbounded parallelism. A pool bounds threads to cores and queues the rest.

**Q7. "Can a worker share the main thread's database pool?"**
No. Each worker loads its own module instances, so `import { pool } from "./db.ts"` in a worker creates a second pool with its own connections. Either keep DB access on the main thread and send workers pure data (Ledger's choice), or count `threads × pool.max` against Postgres's `max_connections` ([L25](../06-data-and-jobs/25-databases-from-node.md)).

**Q8. "How do you cancel a task that's running in a worker?"**
You can't interrupt running JavaScript. Either cooperate (the task polls a flag in a `SharedArrayBuffer` with `Atomics.load` every N iterations) or `terminate()` the worker and replace it. Queued tasks are simply removed. A good pool does all three behind an `AbortSignal`.

**Q9. "A worker threw, and the whole API process died. Why?"**
The uncaught exception is re-emitted as `'error'` on the `Worker` object in the parent, and an EventEmitter `'error'` with no listener throws. Attach `'error'` and `'exit'` handlers, fail the task, and replace the worker.

**Q10. "Will moving my slow, database-heavy endpoint into a worker speed it up?"**
No. The time is spent waiting on Postgres, which doesn't occupy the event loop. You'd add a serialization round trip and a second connection pool. Fix the query, the pool size or the concurrency instead.

**Q11. "Java threads vs Node workers?"**
Java threads share the heap: passing data is free and everything needs synchronization. Node workers share nothing by default: passing data is a copy or a transfer, and races exist only in explicitly shared memory. The Node model trades message cost for the near-absence of locking bugs.

---

## 12. Build & break

### Build — `lib/worker-pool.ts`
Implement §7 and test it with `node:test`. The tests are the deliverable:
- 20 tasks on a 4-worker pool all resolve, and never more than 4 run at once (count inside the worker with a SAB and `Atomics.add`).
- A worker throwing `LedgerError` with `code: "recon_bad_line"` rejects with an error whose `code` is intact.
- Aborting a *queued* task rejects it without touching a worker; aborting a *running* task terminates that worker, the pool replaces it, and the next task succeeds.
- A worker that calls `process.exit(2)` mid-task fails that task and is replaced.
- Task 13 on a pool with `maxQueue: 8` and all workers busy rejects with "pool queue full".

### Build — `reconcile/worker.ts` and the measurement
Generate a fake 1M-line settlement file with a script. Implement `parseSettlement` (decode with `TextDecoder`, split, validate, normalise `amount_minor` to `bigint`) and the §9 loop. In the same process, run a tiny `node:http` server with the L08 loop meter and hammer it with `npx autocannon -c 20 -d 30 http://localhost:3000/healthz` while reconciling. Record a table: wall time, healthz p99 latency and loop p99, for (a) parsing on the main thread, (b) a pool of 1, 2, 4 and `availableParallelism() − 1`. Then swap your pool for piscina behind the same interface and check the numbers match.

### Break — 5 experiments
1. **A worker per request.** Write a handler that does `new Worker(...)` per request and waits for one message. Run `npx autocannon -c 50 -d 20` against it and watch RPS, p99 and RSS (Task Manager, or `ps -o rss -p <pid>` in WSL2). Measure the startup per worker with `performance.now()` around `new Worker` → `'online'`. Then switch to the pool and compare.
2. **Clone vs transfer.** Post a 200 MB `Uint8Array` and time the `postMessage` call and the main loop's delay. Now post it with `[bytes.buffer]` in the transferList: near-instant, and `bytes.byteLength` is `0` afterwards. Finally post an *array of 2M small objects*: notice an object graph costs far more to clone than the same number of bytes.
3. **The lost update.** Four workers increment one `Int32Array` over a SAB 1,000,000 times each with `counter[0]++`. Print the total: well below 4,000,000, and different every run. Switch to `Atomics.add(counter, 0, 1)`: exactly 4,000,000.
4. **The unhandled worker throw.** Remove the `'error'` listener and make a worker throw. The *parent* process dies with the worker's stack trace. Put the listener back: you see exit code 1 and the process lives. Then write `await once(worker, "exit")` and watch it reject with the worker's error instead of resolving.
5. **The poisoned input.** Give a worker `resourceLimits: { maxOldGenerationSizeMb: 32 }` and feed it a "line" that makes the parser build a huge array. Observe `ERR_WORKER_OUT_OF_MEMORY` on `'error'`, the main thread's heap untouched, and your pool replacing the worker. Then rerun with `node --max-old-space-size=384`: the cap vanishes and the worker grows to the main thread's limit. That's the trap above, seen once.

### Explain out loud (60–120 seconds)
1. What a worker is: an isolate, a heap and an event loop, sharing the process and the libuv pool.
2. When workers help (CPU-bound JS) and when they don't (I/O, `scrypt`), and how ELU tells you.
3. Clone vs transfer vs `SharedArrayBuffer`, what each costs, and why custom errors lose their `code`.
4. The pool design: bounded queue, task IDs, crash replacement, cancellation by flag or `terminate()`.
5. The reconciler: bytes transferred out, small records back, backpressure from the pool to the file.

---

## What's next

Workers run *your* JavaScript in parallel. Often what you need to run isn't JavaScript at all: `pg_dump`, `ffmpeg`, `git`. That means child processes, where the big risks are a shell parsing your input, a pipe nobody reads, and a process tree you never killed.

Next → **[Lesson 22: Child processes](22-child-processes.md)**
