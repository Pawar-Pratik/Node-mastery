# Lesson 10 — Cancellation, timeouts & bounded concurrency

> **Why this lesson exists:** a Node service rarely dies from too little capacity. It dies from work nobody is waiting for (a query still running for a client who left ten seconds ago) and from work nobody limited (`Promise.all` over ten thousand webhook sends). `AbortSignal` is Node's universal way to stop work, and a concurrency limit is the other half. Together they're what keeps one slow dependency or one big batch from turning into a self-inflicted outage, and "implement p-limit" is a standard live-coding question.

**Time:** ~80 minutes · **Prereq:** Lessons 06, 09

---

## 1. The idea in one sentence

> **Every unit of work needs a way to be stopped and a limit on how many run at once: an `AbortSignal` for the first, a semaphore for the second. Node gives you neither by default.**

---

## 2. Work nobody is waiting for, and work nobody limited

**Scenario 1: the abandoned report.** A Console user opens a statement summary that runs a 30-second aggregate. After 2 seconds they close the tab. The socket closes, and your handler keeps going: the query runs to completion holding a pool client, the result is serialized, and the write goes to a dead socket. Refresh-happy users multiply it; with a pool of 10, ten abandoned reports and the API stops answering.

In Java you'd at least have a thread to interrupt. **Node has no thread per request to kill, and a promise can't be cancelled: it's a value representing a result, not a handle to the work.** The only code that can stop the work is the code doing it, and only if something tells it to.

**Scenario 2: the batch.** The webhook dispatcher claims 5,000 due deliveries and does this:

```ts
// ❌ 5,000 sends start in the same tick
await Promise.all(due.map((d) => send(d)));
```

`map` calls `send` 5,000 times **synchronously, before the first `await` yields**. That's 5,000 sockets opening at once (the default `fetch` dispatcher doesn't cap connections per origin), 5,000 sets of buffers, file descriptors heading for `EMFILE` ([L09](09-errors-and-crashing.md)), and merchant endpoints receiving hundreds of concurrent requests and answering with 429s, which your retry logic then multiplies. And if one send rejects, `Promise.all` rejects immediately while the other 4,999 keep running unobserved.

A Tomcat server caps concurrency implicitly: 200 threads, so at most 200 things in flight. **Promises cost almost nothing to create, so in Node concurrency is unbounded unless you bound it.**

---

## 3. `AbortController` and `AbortSignal`

The standard (from the web platform, used across Node's APIs):

```text
 AbortController ──owns──▶ AbortSignal  (an EventTarget)
   .abort(reason)            .aborted       false → true, once, forever
                             .reason        why: an Error / DOMException
                             .throwIfAborted()
                             'abort' event  fires synchronously inside abort()
                                 │
            ┌────────────────────┼─────────────────────┬───────────────────┐
            ▼                    ▼                     ▼                   ▼
      fetch(url,{signal})   setTimeout(ms,v,{signal})  pipeline(...,{signal})  your code
      closes the socket     clears the timer           destroys every stream   checks & listens
```

**Abort is cooperative.** `abort()` doesn't stop anything by itself; it flips a flag and fires an event. Every piece of work has to be listening.

### Node APIs that take a `signal`

| API | What abort actually does |
|---|---|
| `fetch(url, { signal })` | Cancels the request/connection; the promise rejects with `signal.reason` |
| `fs.promises.readFile(p, { signal })` (and `writeFile`) | Stops `readFile`'s internal chunked reading; a read already submitted to the thread pool still completes ([L07](07-the-thread-pool.md)) |
| `setTimeout` / `setInterval` / `setImmediate` from `node:timers/promises` | Clears the timer; rejects with an `AbortError` |
| `events.once(emitter, name, { signal })`, `events.on(…)` | Stops waiting and removes the listeners |
| `pipeline(…streams, { signal })` from `node:stream/promises` | Destroys every stream in the pipeline ([L13](../03-data-and-io/13-streams-and-backpressure.md)) |
| `spawn` / `exec` / `execFile` / `fork` with `signal` | Kills the child with `killSignal` ([L22](../05-concurrency/22-child-processes.md)) |
| `readline.createInterface({ signal })`, `rl.question(q, { signal })` | Closes the interface / cancels the question |
| `http.request({ signal })` | Destroys the request |

### Building signals

- **`AbortSignal.timeout(ms)`** aborts after `ms` with a `DOMException` named `TimeoutError`.
- **`AbortSignal.any([a, b])`** aborts as soon as any input does, with that input's reason. This is how you combine "the client left" with "the deadline passed".

### What the rejection looks like

The error you catch depends on the API, which is why matching on error shapes is fragile:

| Situation | `fetch` rejects with | Node APIs (`timers/promises`, `fs`, …) reject with |
|---|---|---|
| `controller.abort()` with no reason | `DOMException` `AbortError` (the default reason) | `AbortError` (`code: "ABORT_ERR"`), `cause` = the reason |
| `AbortSignal.timeout(ms)` fired | `DOMException` `TimeoutError` | `AbortError`, `cause` = the `TimeoutError` |
| `controller.abort(new Error("shutdown"))` | **that exact `Error`** | `AbortError`, `cause` = that `Error` |

**The robust test isn't the error's shape; it's the signal.** You know whether you asked for the abort:

```ts
try {
  await work(signal);
} catch (err) {
  if (signal.aborted) return reportCancelled(signal.reason);   // ✅ we asked for this; not a failure
  throw err;                                                    // a real failure
}
```

This matters for metrics and retries: an aborted call is not a failed dependency, must not trip a circuit breaker, and must never be retried.

---

## 4. Writing abortable functions

Five rules:

1. **Check before starting:** `signal.throwIfAborted()`. Don't open a connection for a request that's already dead.
2. **Pass the signal down** to every API that accepts one. One signal, threaded through the whole call tree.
3. **For work that doesn't take a signal, listen and cancel it yourself**: destroy a socket, cancel a Postgres backend, stop a chunked CPU loop between chunks with `throwIfAborted()` ([L08](08-blocking-the-loop.md)).
4. **Remove your listener when the work finishes**, in `finally`.
5. **Reject with `signal.reason`**, so the caller sees *why* (timeout vs client gone vs shutdown).

Rule 4 is the one everyone breaks:

```ts
/** Run fn on abort. Returns the cleanup; always call it. */
export function onAbort(signal: AbortSignal, fn: () => void): () => void {
  if (signal.aborted) { fn(); return () => {}; }
  signal.addEventListener("abort", fn, { once: true });
  return () => signal.removeEventListener("abort", fn);
}

export async function queryCancellable(client: PgClient, sql: string, params: unknown[], signal: AbortSignal) {
  signal.throwIfAborted();
  const off = onAbort(signal, () => void cancelBackend(client));   // L25: pg_cancel_backend / destroy
  try {
    return await client.query(sql, params);
  } finally {
    off();      // ✅ the signal may live for hours; this call must not stay attached to it
  }
}
```

`{ once: true }` only removes the listener **if abort fires**. If the work finishes normally, which is almost always, the listener stays registered. On a short-lived per-request signal that's harmless; it's collected with the signal. On a **long-lived signal** (a worker's shutdown signal lives as long as the process) every call adds one more listener, and each listener's closure keeps its `client`, `sql` and `params` reachable. That's a memory leak that grows with traffic, and an abort that now runs 400,000 stale callbacks.

### The leak detector you have to switch on

EventTargets print `MaxListenersExceededWarning` past 10 listeners per event type, the same leak detector EventEmitter has ([L11](../03-data-and-io/11-eventemitter.md)). **But in current Node an `AbortSignal`'s limit defaults to 0 (unlimited), so a leaking signal stays silent**: check with `events.getMaxListeners(signal)`. Switch the detector on for the long-lived signals you own, at least in development and tests, and measure directly:

```ts
import { setMaxListeners, getEventListeners } from "node:events";

setMaxListeners(10, shutdown.signal);                       // warn past 10: a leak detector, not a limit
// …in a test, after 1,000 calls:
assert.equal(getEventListeners(shutdown.signal, "abort").length, 1);   // only the worker loop itself
```

---

## 5. Deadline propagation: one signal per request

A request can become pointless for three reasons, and all three should stop its work:

| Source | Detect it with | Typical reason |
|---|---|---|
| **The client disconnected** | `res.on("close")` while `!res.writableFinished` | Tab closed, client timeout, LB gave up |
| **The request deadline passed** | `AbortSignal.timeout(budget)` | You promised a response within, say, 15 s |
| **The process is shutting down** | Your lifecycle's shutdown hook ([L24](../05-concurrency/24-graceful-shutdown.md)) | SIGTERM during a rollout |

**Use `res`, not `req`, to detect a disconnect.** In modern Node, an `IncomingMessage` emits `'close'` as soon as its body has been consumed, which for a typical POST is milliseconds into the request, long before you respond. `res` emits `'close'` when the underlying connection is done; if the response hadn't finished by then, the client left.

```ts
import type { ServerResponse } from "node:http";

const inFlight = new Set<AbortController>();

export function requestSignal(res: ServerResponse, timeoutMs: number): AbortSignal {
  const ac = new AbortController();
  inFlight.add(ac);
  res.once("close", () => {
    inFlight.delete(ac);
    if (!res.writableFinished) ac.abort(new DOMException("client closed the connection", "AbortError"));
  });
  return AbortSignal.any([ac.signal, AbortSignal.timeout(timeoutMs)]);
}

/** Called by the lifecycle after the drain deadline (L24): abort whatever is still running. */
export function abortAllRequests(reason: unknown): void {
  for (const ac of inFlight) ac.abort(reason);
}

// Express wiring: one line of middleware
app.use((req, res, next) => { res.locals.signal = requestSignal(res, 15_000); next(); });
```

Why a registry for shutdown instead of `AbortSignal.any([clientGone, timeout, shutdownSignal])`? Folding a process-lifetime signal into a fresh `any()` on every request makes that one long-lived signal track every request ever created. In a quick test on Node 24, a loop of such calls retained memory and got progressively slower. **Keep per-request signals made only of per-request parts, and fan shutdown out from a set you clean up on `close`.**

Then every downstream call in the handler gets the request's signal, optionally tightened with its own budget:

```ts
const signal = res.locals.signal as AbortSignal;
const payment = await repo.getPayment(id, { signal });
const risk = await fetch(riskUrl, { signal: AbortSignal.any([signal, AbortSignal.timeout(2_000)]) });
```

```text
t=0     request arrives; signal created (15 s budget)
t=0.1   query #1 starts ─────────────┐
t=2.0   client closes the tab        │  res 'close', !writableFinished → abort
        ├─ query #1 cancelled ◀──────┘  backend cancelled, client released to the pool
        ├─ risk fetch never starts (throwIfAborted)
        └─ log: client_aborted, status 499 (nginx's convention), 0 work wasted after t=2.0
```

Passing the *remaining budget* to the next service (a deadline header) is the protocol half, covered in [API L18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md). The signal is the in-process half.

---

## 6. Bounded concurrency

### A semaphore from scratch (this is "implement p-limit")

```ts
export type Limiter = {
  run<T>(fn: () => Promise<T>, signal?: AbortSignal): Promise<T>;
  readonly active: number;
  readonly pending: number;
};

export function limit(max: number): Limiter {
  if (!Number.isInteger(max) || max < 1) throw new RangeError(`max must be a positive integer, got ${max}`);
  let active = 0;
  const waiters: Array<() => void> = [];

  function acquire(signal?: AbortSignal): Promise<void> {
    signal?.throwIfAborted();
    if (active < max) { active++; return Promise.resolve(); }
    return new Promise<void>((resolve, reject) => {
      let off = () => {};
      const grant = () => { off(); resolve(); };          // slot handed over: `active` unchanged
      waiters.push(grant);
      if (signal) off = onAbort(signal, () => {           // gave up while queued: leave the queue
        const i = waiters.indexOf(grant);
        if (i !== -1) waiters.splice(i, 1);
        reject(signal.reason);
      });
    });
  }

  function release(): void {
    const next = waiters.shift();
    if (next) next();                                     // FIFO hand-off to the next waiter
    else active--;
  }

  return {
    async run(fn, signal) {
      await acquire(signal);
      try { return await fn(); } finally { release(); }   // released on success AND failure
    },
    get active() { return active; },
    get pending() { return waiters.length; },
  };
}
```

What the interviewer is checking, line by line:

- **`release` in `finally`.** A rejected task that doesn't release leaks a slot; after `max` failures the limiter deadlocks and everything queued waits forever.
- **Direct hand-off.** `release` gives the slot straight to the next waiter instead of decrementing and letting everyone race for it. FIFO order, no thundering herd, and `active` never overshoots `max`.
- **Abort while queued** removes the waiter, so a dead request doesn't consume a slot when its turn comes. (A *running* task is aborted through its own signal, not by the limiter.)
- **`active` and `pending` are exported** because they're your metrics: a `pending` count that only grows says the limit or the dependency is the bottleneck.
- **It's synchronous up to the first `await`.** `acquire` claims a free slot in the same tick, so there's no window where two callers both see `active < max`. Single-threaded JavaScript makes the check-then-increment atomic, as long as no `await` sits between them.

### Per-key limits under a global limit

The dispatcher needs two caps: at most 5 concurrent sends to any one merchant endpoint (don't hammer one receiver) and at most 200 overall (protect your own sockets and memory).

```ts
export function keyedLimit(perKey: number) {
  const limiters = new Map<string, Limiter>();
  return async function run<T>(key: string, fn: () => Promise<T>, signal?: AbortSignal): Promise<T> {
    let l = limiters.get(key);
    if (!l) { l = limit(perKey); limiters.set(key, l); }
    try { return await l.run(fn, signal); }
    finally { if (l.active === 0 && l.pending === 0) limiters.delete(key); }   // no Map that grows forever
  };
}
```

**Acquire the per-key slot first, then the global one.** Reverse it and a slow merchant with 300 due deliveries grabs global slots, then sits on them waiting for its 5 per-endpoint slots. Within seconds most of the 200 global slots belong to one dead endpoint and every other merchant's webhooks stall: head-of-line blocking you built yourself. Per-key first means a slow endpoint can only ever hold 5 global slots.

### N workers pulling from one iterator

A limiter wraps individual calls. When you're consuming a source (a cursor of 4M rows, a stream of ingested records), the cleaner shape is **N workers sharing one iterator**: only N items are ever in flight, so memory stays bounded however long the source is. ([TS L16](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/16-async-and-errors.md) has the array-only `mapLimit` and its typing; this is the iterator version with cancellation.)

```ts
export async function forEachLimit<T>(
  source: Iterable<T> | AsyncIterable<T>,
  concurrency: number,
  fn: (item: T, signal: AbortSignal) => Promise<void>,
  signal?: AbortSignal,
): Promise<void> {
  const iterator = (async function* () { yield* source; })();   // one shared iterator; concurrent next() calls queue
  const failed = new AbortController();
  const stop = signal ? AbortSignal.any([signal, failed.signal]) : failed.signal;

  const worker = async () => {
    try {
      for (let r = await iterator.next(); !r.done; r = await iterator.next()) {
        stop.throwIfAborted();
        await fn(r.value, stop);
      }
    } catch (err) {
      if (!failed.signal.aborted) failed.abort(stop.aborted ? stop.reason : err);   // first failure stops all
    }
  };

  try {
    await Promise.all(Array.from({ length: concurrency }, worker));   // workers never reject
  } finally {
    await iterator.return(undefined);       // close the source: releases a cursor, destroys a stream
  }
  if (failed.signal.aborted) throw failed.signal.reason;
}
```

Unlike `Promise.all(items.map(fn))`, a failure here **stops the other workers** (they see `stop` at their next item, and their in-flight call gets the signal), **waits for them to finish**, then throws the first error. No orphaned work keeps running after the function has returned.

### Little's law: how big should N be?

**Concurrency = throughput × latency** (L = λW, for a system in steady state). It turns "how many in flight?" into arithmetic:

```text
Target: 1,000 deliveries/s.  Mean merchant latency: 200 ms.
  L = 1,000/s × 0.2 s = 200 in flight            ← the global cap

Merchants slow down to 2 s:
  with cap 200 → throughput = 200 / 2 s = 100/s  ← the queue grows in Postgres, which is fine
  without a cap → 2,000 concurrent sockets       ← memory, fds and merchants' rate limits

One dead endpoint, per-key cap 5, 10 s timeout:
  holds at most 5 slots; drains at 5 / 10 s = 0.5 attempts/s; can't starve anyone else
```

The cap is a decision about **where waiting happens**: in a durable queue you can see and drain, instead of in sockets, buffers and heap you can't. The same law sizes pool connections ([L25](../06-data-and-jobs/25-databases-from-node.md)) and the blueprint's budgets ([L34](../09-project/34-ledger-engine-blueprint.md)).

---

## 7. Backoff that respects the signal

The retry *policy* (full jitter, budgets, only idempotent operations) is in [API L18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md), and the typed `retry` helper is in [TS L16](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/16-async-and-errors.md). The Node mechanics are three lines you must not forget:

```ts
import { setTimeout as sleep } from "node:timers/promises";

for (let attempt = 1; ; attempt++) {
  try { return await fn(signal); }
  catch (err) {
    if (signal.aborted) throw signal.reason;                       // 1. an abort is never retried
    if (!isRetryable(err) || attempt >= max) throw err;
    await sleep(Math.random() * Math.min(capMs, baseMs * 2 ** attempt), undefined, { signal });  // 2. abortable sleep
  }
}
```

The abortable sleep is what makes shutdown fast. A worker sitting in a 30-second backoff without a signal holds the process past Kubernetes' 30-second grace period and gets SIGKILLed mid-job. With the signal, SIGTERM ends the sleep immediately and the job goes back to the queue ([L24](../05-concurrency/24-graceful-shutdown.md)).

---

## 8. Ledger: the dispatcher's send loop

The spec: **a global cap of 200, a per-endpoint cap of 5, a 10 s timeout per attempt, and the whole batch abortable on shutdown.**

```ts
const global = limit(200);
const perEndpoint = keyedLimit(5);

export async function dispatchBatch(batch: Delivery[], shutdown: AbortSignal): Promise<void> {
  // A short-lived signal per batch, fed by the process-lifetime shutdown signal through ONE
  // listener that is removed when the batch ends. Per-attempt any() calls combine short-lived parts only.
  const batchAc = new AbortController();
  const off = onAbort(shutdown, () => batchAc.abort(shutdown.reason));
  const stop = batchAc.signal;
  try {
    // The batch is bounded by the claim query (LIMIT 500), so 500 waiting promises are fine:
    // waiting in the limiter costs a closure, not a socket.
    await Promise.all(batch.map((d) =>
      perEndpoint(d.endpointId, () =>                            // per-key first…
        global.run(() =>                                         // …then global
          attempt(d, AbortSignal.any([stop, AbortSignal.timeout(10_000)])),
        stop),
      stop)
        .then((result) => recordResult(d, result))
        .catch((err) => stop.aborted
          ? releaseClaim(d)                                      // shutdown isn't the merchant's failure
          : recordFailure(d, classify(err, "merchant_endpoint")))));
  } finally {
    off();
  }
}
```

The details that make it correct:

- **`stop.aborted` decides the bookkeeping.** An attempt cut short by SIGTERM goes back to `pending` with its lease released (or simply left to expire, [L26](../06-data-and-jobs/26-background-jobs-and-queues.md)) and does **not** count toward the 8 attempts or the endpoint's auto-disable.
- **The per-attempt timeout** comes from the signal, so the socket is actually closed at 10 s, not merely ignored.
- **`.catch` on every element** means one failure doesn't reject `Promise.all` and abandon the batch; each delivery records its own outcome.
- **The process-lifetime `shutdown` signal gets exactly one listener per batch, removed in `finally`.** Thousands of attempts per second combine only the short-lived batch signal with their own timeouts, so nothing long-lived accumulates listeners or dependants (§4 and §5).

---

## 9. Production rules

| Rule | Why |
|---|---|
| **Every async function you write accepts a `signal` and passes it down** | Cancellation only works if the whole call tree listens |
| **Every outbound call has a deadline: `AbortSignal.timeout` or a combined signal** | An unbounded call holds a socket and a closure forever |
| **Detect disconnects with `res.on("close")` + `!res.writableFinished`, never `req.on("close")`** | `req` closes when its body is consumed, not when the client leaves |
| **Remove abort listeners in `finally`** | `{ once: true }` only cleans up if abort fires; long-lived signals accumulate listeners and their closures |
| **Turn on `setMaxListeners(10, signal)` for long-lived signals you own** | `AbortSignal` doesn't warn by default, so leaks are silent |
| **Decide "was this an abort?" with `signal.aborted`, not the error's shape** | Abort errors differ between `fetch` and Node APIs |
| **Never retry or count an abort as a dependency failure** | It's your decision, not their outage; it would trip breakers and auto-disable endpoints |
| **Never `Promise.all` an unbounded array of I/O** | Every element starts at once: sockets, fds, memory, downstream rate limits |
| **Bound concurrency with a semaphore or N workers on one iterator; release in `finally`** | A leaked slot eventually deadlocks the limiter |
| **Acquire per-key limits before the global limit** | Otherwise one slow key holds global slots and blocks everyone |
| **Size limits with Little's law and let excess wait in a durable queue** | Waiting in Postgres is visible and survives restarts; waiting in sockets isn't |
| **Sleep through backoff with an abortable timer** | Shutdown must not wait out a 30-second backoff |

> **Spring equivalent:** Resilience4j's `@Bulkhead` (semaphore type) is `limit(n)`, and `@TimeLimiter` is the per-call `AbortSignal.timeout` ([SpringBoot L34](https://github.com/Pawar-Pratik/SpringBoot/blob/main/09-production/34-resilience-and-integration.md)). The difference: a servlet container's thread pool is an implicit global bulkhead you get for free, while Node has no implicit cap at all. Cancellation differs too: Java can interrupt a blocked thread; Node can only ask code to stop, so the code has to listen.

---

## 10. Interview traps

**Q1. "How do you cancel an in-flight operation in Node?"**
You can't cancel a promise; it's a result, not a handle. You create an `AbortController`, pass its `signal` to every API in the call tree (`fetch`, `fs`, timers, `pipeline`, child processes), and call `abort(reason)`. Abort is cooperative: each API stops its own work, and your own functions must check `throwIfAborted()` and listen for `'abort'` to cancel things like a database query.
**Follow-up:** "What does abort do to `fs.readFile`?" It stops `readFile`'s chunked reading and rejects; an OS read already handed to the thread pool still completes.

**Q2. "How do you stop work when the client disconnects?"**
Listen for `res.on("close")` and check `!res.writableFinished`; if the response hadn't finished, the client left, so abort a per-request controller whose signal every downstream call received. Not `req.on("close")`: it fires once the request body is consumed. The weak answer stops writing to the socket and lets the query run to completion.
**Follow-up:** "What about the database query already running?" Abort has to reach it: cancel the backend or destroy the connection, and set `statement_timeout` as the backstop ([L25](../06-data-and-jobs/25-databases-from-node.md)).

**Q3. "Implement p-limit."**
A counter and a FIFO queue of resolvers. `run` acquires (increment if under the limit, otherwise enqueue and wait), awaits the task, and releases in `finally`; release hands the slot directly to the next waiter or decrements. Points that separate answers: release on rejection, FIFO hand-off without a race, removing a waiter whose signal aborts, and exposing `active`/`pending` as metrics.

**Q4. "Why is `Promise.all(items.map(send))` over 10,000 items dangerous?"**
`map` starts all 10,000 synchronously before anything yields. You get 10,000 sockets (the default `fetch` dispatcher doesn't cap per origin), 10,000 buffers, `EMFILE` when descriptors run out, and a downstream flooded into rate limiting. And `Promise.all` rejects on the first failure while the rest keep running unobserved. Use a limit, or N workers on one iterator.

**Q5. "How do you combine a timeout with a user cancel, and tell which one fired?"**
`AbortSignal.any([userSignal, AbortSignal.timeout(ms)])`. The combined signal's `reason` is whichever input fired first: a `DOMException` named `TimeoutError` for the timeout, or the user's reason. Branch on `signal.reason`, not the caught error's shape, because `fetch` and Node APIs wrap aborts differently.

**Q6. "What is Little's law and how does it size concurrency?"**
L = λW: average in-flight work equals throughput times average latency. 1,000 deliveries/s at 200 ms needs 200 concurrent; if latency rises to 2 s, a cap of 200 holds throughput at 100/s and the backlog queues durably instead of exploding sockets and memory. It tells you both the cap and what happens when the dependency slows down.

**Q7. "Does `Promise.race([work(), timeout(1000)])` implement a timeout?"**
It stops *waiting*, not *working*. The losing `work()` keeps running, holding its socket or pool client, and its eventual rejection is unobserved. The race timer also keeps the process alive unless cleared. A real timeout passes `AbortSignal.timeout(ms)` into the work so it actually stops.

**Q8. "Heap snapshots show thousands of closures retained by one `AbortSignal`. What happened?"**
Code adds an `'abort'` listener per call to a long-lived signal (a shutdown signal) and never removes it. `{ once: true }` doesn't help because abort never fires. Remove listeners in `finally`, and enable `setMaxListeners(10, signal)` in tests, because `AbortSignal` doesn't warn by default.

**Q9. "You have a global limit and a per-merchant limit. Which do you acquire first?"**
Per-merchant first. Taking the global slot first lets a slow merchant's queued tasks sit on global slots while they wait for per-merchant slots, starving everyone else. Per-key first caps any one key's share of the global pool at its per-key limit.

**Q10. "Why not just raise concurrency until throughput stops improving?"**
Because past the knee you're not adding throughput, only queueing inside the dependency and inside your process: latency rises, memory rises, the downstream starts rejecting, and retries amplify it. Size from Little's law and the dependency's real capacity (its rate limits, its pool), and leave headroom.

---

## 11. Build & break

### Build — `src/lib/concurrency.ts`
- `onAbort(signal, fn)`, `limit(n)`, `keyedLimit(n)` and `forEachLimit(source, n, fn, signal)` from this lesson, plus `mapLimit(items, n, fn, signal)` implemented on `forEachLimit` (iterate `items.entries()` and write results by index).
- `withDeadline(signal, ms)` returning `AbortSignal.any([signal, AbortSignal.timeout(ms)])`.
- Tests: peak concurrency never exceeds `n` (track a counter inside the task); a rejected task releases its slot; aborting a queued waiter removes it and leaves `pending` at 0; `keyedLimit`'s map is empty after all keys drain; `forEachLimit` stops starting new items after the first failure and closes the source iterator; no abort listeners remain on the caller's signal afterwards (`getEventListeners`).

### Build — the request-signal middleware
- `requestSignal(res, timeoutMs)` and `abortAllRequests(reason)` from §5, `res.locals.signal` set for every route, and the lifecycle calling `abortAllRequests` after the drain deadline.
- Every repository and outbound client in the API accepts `{ signal }`. Log `client_aborted` (status 499) and count it as its own metric, separate from errors.

### Break — four experiments
1. **The self-DoS.** Start a local server that holds every request for 2 s. Fire `Promise.all` over 5,000 `fetch`es to it and print `process.memoryUsage().rss` every 500 ms. Count connections with `netstat -an | grep -c ":3999"` (Git Bash). Then run it in WSL2 after `ulimit -n 1024` and watch `EMFILE` appear in the `cause` of the failures. Replace with `mapLimit(urls, 50, …)`: connections stay at ~50, memory flat, no `EMFILE`, and it finishes in ~200 s of wall time instead of failing.
2. **The silent listener leak.** Create one long-lived `AbortController`, then call a function 10,000 times that adds an `'abort'` listener with `{ once: true }` and never removes it. Print `getEventListeners(signal, "abort").length` (10,000) and note there was no warning. Add `setMaxListeners(10, signal)` and rerun: the warning appears at 11. Fix with the `finally` cleanup from §4 and confirm the count is 0.
3. **The abandoned request.** Write a route that runs ten sequential fake queries (`await sleep(1000, undefined, { signal })`, logging each). Call it with `curl.exe --max-time 2 http://localhost:3000/report` (PowerShell's `curl` is an alias; use `curl.exe`). Without the signal: all ten "queries" log, for a client that left at 2 s. Wire `res.locals.signal` through: logging stops at the disconnect and you log `client_aborted`.
4. **The race that doesn't cancel.** `await Promise.race([fetch(slowUrl), sleep(1000).then(() => { throw new Error("timeout"); })])` against an endpoint that takes 10 s, with a `.then(() => console.log("still finished"))` on the fetch. The "timeout" fires at 1 s, then "still finished" prints at 10 s, and the process can't exit before then. Replace with `fetch(slowUrl, { signal: AbortSignal.timeout(1000) })`: the request is gone at 1 s.

### Explain out loud (2 minutes)
1. Why a promise can't be cancelled, and what `AbortSignal` does instead.
2. The three sources of cancellation for a request, and why you watch `res` and not `req`.
3. Writing an abortable function: the five rules, especially listener removal.
4. Why `Promise.all` over a big array is self-DoS, and the semaphore's four correctness points.
5. Little's law with the dispatcher's numbers, and why per-key limits come before the global one.

---

## Module 2 complete — checkpoint

- [ ] The six libuv phases in order, and what runs in each
- [ ] Why an idle Node process uses no CPU: the poll phase blocks with a timeout from the nearest timer
- [ ] `process.nextTick` and the microtask queue drain after every callback, nextTick first
- [ ] `setTimeout(0)` vs `setImmediate`: nondeterministic in the main module, immediate first inside I/O
- [ ] How recursive `nextTick` or promise chains starve I/O
- [ ] What keeps the loop alive, and what `unref()` does
- [ ] Why Ledger's webhook retries live in `next_attempt_at`, not in `setTimeout`
- [ ] What `await` does to execution order: suspend, resume as a microtask
- [ ] Unhandled rejections exit with code 1, and the "late catch" that triggers it
- [ ] `return await` vs `return` inside `try/catch`
- [ ] Which APIs use the libuv thread pool, and why slow DNS slows file reads
- [ ] Why `UV_THREADPOOL_SIZE` must be set at launch, and how to size it
- [ ] bcrypt vs bcryptjs, and the thread pool vs worker threads
- [ ] What blocks the loop: `*Sync` calls, giant JSON, ReDoS, CPU loops over request data
- [ ] Measuring blocking with `monitorEventLoopDelay` and ELU, and why ELU beats CPU%
- [ ] Fixes by layer, and shedding load on loop pressure
- [ ] Operational vs programmer errors, and the four error channels
- [ ] Compare `err.code` through the `cause` chain, never messages
- [ ] Why you exit after `uncaughtException`, and the fatal handler's steps
- [ ] Crash-only design and the request-scope boundary
- [ ] `AbortSignal`: cooperative cancellation, `timeout()`, `any()`, removing listeners
- [ ] Detecting a client disconnect with `res.on("close")` + `!res.writableFinished`
- [ ] Implementing p-limit; why `Promise.all` over 10k items is self-DoS
- [ ] Little's law, and per-key limits before the global one

---

## What's next

Module 3 moves from scheduling to data. It starts with the object underneath every socket, stream, server and child process: the EventEmitter. Its rules are few and surprising (emit is synchronous, an unheard `'error'` kills the process, a listener per request is a leak), and they explain a whole class of bugs you've already met.

Next → **[Lesson 11: EventEmitter: the pattern under everything](../03-data-and-io/11-eventemitter.md)**
