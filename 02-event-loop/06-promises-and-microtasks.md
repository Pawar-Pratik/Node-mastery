# Lesson 06 — Promises, async/await & the microtask queue

> **Why this lesson exists:** `async`/`await` makes asynchronous code read like synchronous code, and that's exactly what hides the scheduling. Engineers who write it daily can't say what `await` does to execution order, why a rejection that was "inside a try/catch" killed a production process, or why "async" code still froze the server. This lesson turns `await` back into what it is, a microtask with precise rules, so you can predict it, debug it, and explain it in an interview.

**Time:** ~80 minutes · **Prereq:** Lesson 05

---

## 1. The idea in one sentence

> **`await` doesn't wait: it suspends the function, hands the thread back to whoever called it, and schedules the rest of the function as a microtask. Every `await` is a scheduling point, and any rejection still unhandled when the microtask queue empties takes the process down.**

---

## 2. Before promises: callbacks, error-first, and Zalgo

Node's original contract is the **error-first callback**: the last argument is a function whose first parameter is an error or `null`.

```ts
import { readFile } from "node:fs";

readFile("settlement.csv", "utf8", (err, text) => {
  if (err) return handle(err);      // operational errors arrive here, never as a throw
  parse(text);
});
```

It works, but every caller re-implements error handling, nothing composes ("do these three in parallel, then this"), and a buggy API can call you twice or never. Worse is the bug Isaac Schlueter named in his 2013 post "Designing APIs for Asynchrony": **releasing Zalgo**, an API that is *sometimes* synchronous and sometimes asynchronous.

```ts
const cache = new Map<string, Merchant>();

// ❌ Synchronous on a cache hit, asynchronous on a miss.
function getMerchant(id: string, cb: (err: Error | null, m?: Merchant) => void): void {
  const hit = cache.get(id);
  if (hit) return cb(null, hit);                       // runs BEFORE the caller's next line
  db.findMerchant(id).then((m) => { cache.set(id, m); cb(null, m); }, cb);
}

// ✅ Always asynchronous: the callback never runs before the caller's next line.
//    if (hit) return queueMicrotask(() => cb(null, hit));
```

Call that twice with `let ready = false; getMerchant(id, () => use(ready)); ready = true;` and the first call sees `ready === true` (miss, async) while the second sees `ready === false` (hit, sync). The same call site runs in two different orders depending on the cache. **Node core never does this.** Invalid arguments throw synchronously because they're programmer errors, but every operational result is delivered asynchronously. That's why `net.connect()` reports a failure through an `'error'` event on a later tick, never by throwing from the call: you always get to attach the listener first.

**Promises make Zalgo impossible by construction:** a `.then` callback always runs as a microtask, even when the promise is already settled.

---

## 3. Promises, mechanically

A promise is **pending**, then settles once: **fulfilled** with a value or **rejected** with a reason. "Resolved" is a different word: a promise resolved *with another promise* is locked to it but still pending.

Three mechanics matter in production:

1. **The executor runs synchronously.** `new Promise(executor)` calls `executor` immediately; so does calling an `async` function, up to its first `await`. **A promise is eager:** the work starts when you create it, whether or not anyone ever awaits it. That's the root of floating promises and the late-catch crash in §7.
2. **Reactions are microtasks.** `.then`, `.catch`, `.finally` and `await` continuations go on V8's microtask queue, drained after every callback ([Lesson 05](05-the-event-loop.md)).
3. **Resolving with a thenable costs ticks.** `resolve(x)` checks whether `x` has a `then` method. If it does, it doesn't use `x` directly; it queues a job that calls `x.then(resolve, reject)`, and *that* settles one tick later. For a native promise, that's **two extra microtask ticks**.

```ts
async function a() { return Promise.resolve(1); }   // resolved WITH a promise
async function b() { return 1; }                     // resolved with a value

a().then(() => console.log("a"));
b().then(() => console.log("b"));
Promise.resolve()
  .then(() => console.log("1")).then(() => console.log("2"))
  .then(() => console.log("3")).then(() => console.log("4"));
// b, 1, 2, a, 3, 4
```

`b`'s promise is already fulfilled, so its reaction is queued at once. `a`'s promise has to adopt the inner promise: one tick to call its `then`, a second to settle, and only then is `a`'s reaction queued. It loses two places to the counter chain.

**Any object with a `then` method is treated as a promise.** That's how query builders like Knex's and Mongoose's queries execute when you `await` them. It's also a trap: return an object with a `then` field from an `async` function, and the runtime *calls* it, and your caller receives whatever it resolves to, not your object.

---

## 4. What `await` actually does

```ts
const value = await x;
// ≈ 1. Suspend this function and return control to the caller
//      (on the first await, the caller receives this function's pending promise).
//   2. Convert x to a promise: native promises are used as-is, anything else is wrapped.
//   3. Register a reaction on it: on fulfil, resume with the value; on reject, throw inside the function.
//   4. The resumption runs as a microtask, after x settles.
```

| You `await`… | Ticks before the function resumes | Why |
|---|---|---|
| A native promise | **1** after it settles | Used directly; one reaction job |
| A non-promise (`42`, `null`, an object) | **1** | Wrapped in an already-fulfilled promise |
| A non-native thenable | **2** | The thenable-adoption job from §3 runs first |

**`await` on a non-promise still yields.** `await null` is a real scheduling point: everything already queued as a microtask runs before your function continues. The first segment of an `async` function, up to its first `await`, runs synchronously inside the caller's callback.

Worked example, inside an I/O callback so the rules from [Lesson 05](05-the-event-loop.md) apply unchanged:

```ts
import { readFile } from "node:fs";

async function work() {
  console.log("work start");
  await null;
  console.log("work resumed");
}

readFile(import.meta.filename, () => {
  console.log("callback");
  setImmediate(() => console.log("immediate"));
  setTimeout(() => console.log("timeout"), 0);
  work();
  process.nextTick(() => console.log("tick"));
  Promise.resolve().then(() => console.log("promise"));
  console.log("callback end");
});
// callback, work start, callback end, tick, work resumed, promise, immediate, timeout
```

`work()` prints synchronously until `await null`, which queues its resumption as the *first* microtask. The `.then` is queued second. When the callback returns, ticks drain (`tick`), then microtasks in order (`work resumed`, `promise`), then check (`immediate`), then timers (`timeout`).

**The consequence that bites in services:** every `await` is a point where other requests run. A check-then-act split by an `await` (`if (!(await keys.exists(k))) await keys.insert(k)`) is a race between two concurrent requests, even with one thread. Ledger's idempotency claim is a single `INSERT … ON CONFLICT` for exactly this reason ([API Lesson 18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md)).

---

## 5. `async` functions, `return await`, and stack traces

An `async` function **always returns a promise**. A `throw` inside it, even before the first `await`, becomes a rejection, not a synchronous exception.

`return promise` and `return await promise` differ inside `try`:

```ts
async function getBalance(merchantId: string) {
  const client = await pool.connect();
  try {
    return client.query(BALANCE_SQL, [merchantId]);         // ❌ leaves the try immediately:
                                                             //    a rejection skips the catch,
  } catch (err) {                                            //    and finally releases the client
    throw new LedgerError("balance_unavailable", { cause: err }); //    while the query is in flight
  } finally {
    client.release();
  }
}
// ✅ return await client.query(BALANCE_SQL, [merchantId]);   // settles inside the try
```

With plain `return`, the function hands the pending promise to its caller and leaves the `try` block at once: **`finally` runs before the query has finished, and a rejection goes straight to the caller, bypassing your `catch`.** With `return await`, the function stays suspended inside the `try` until the promise settles. Outside a `try`, the two behave the same apart from stack traces. The ESLint rule that enforces this (`@typescript-eslint/return-await` set to `in-try-catch`) is configured in [TypeScript Lesson 16](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/16-async-and-errors.md).

**Stack traces:** V8's zero-cost async stack traces rebuild the async part of a trace by walking the chain of `async` functions *suspended at an `await`*, using information the promises already hold. That gives you `at async handler` frames for free, but it explains the gaps: a function that does `return p` without awaiting isn't suspended, so it's missing; callbacks attached with raw `.then()` aren't async functions, so their callers are missing. **Prefer `await` over `.then` chains when you want readable production traces**, and chain errors with `cause` (TypeScript Lesson 16).

---

## 6. Node's promise toolkit

| Need | Use | Note |
|---|---|---|
| Convert an error-first API | `util.promisify(fn)` | Respects `util.promisify.custom` for APIs whose callbacks don't fit |
| Files | `node:fs/promises` | Same thread pool as callback `fs` ([Lesson 07](07-the-thread-pool.md)) |
| Sleep / timeouts | `setTimeout(ms, value, { signal })` from `node:timers/promises` | Aborting rejects with an `AbortError`; with `AbortSignal.timeout()` its `cause` is a `TimeoutError` |
| Yield to the loop | `setImmediate()` from `node:timers/promises` | A macrotask: lets poll run |
| Wait for one event | `events.once(emitter, "ready", { signal })` | Resolves with the event's args array; rejects if `'error'` fires first |
| Stream composition | `pipeline` from `node:stream/promises` | Covered in [Lesson 13](../03-data-and-io/13-streams-and-backpressure.md) |
| DNS | `node:dns/promises` | `lookup` uses the pool, `resolve*` doesn't ([Lesson 07](07-the-thread-pool.md)) |

---

## 7. Unhandled rejections

**When Node checks:** after each callback, once the tick and microtask queues have drained, Node walks the promises that were rejected with no handler attached. Any still unhandled is reported.

**What happens then** depends on `--unhandled-rejections`:

| Mode | Behaviour |
|---|---|
| **`throw`** (the default since Node 15) | Emit `'unhandledRejection'`; if nobody listens, raise it as an uncaught exception: **the process exits with code 1** |
| `strict` | Raise as an uncaught exception even if an `'unhandledRejection'` listener exists |
| `warn` | Print a warning and keep running |
| `warn-with-error-code` | Warn, and set the exit code to 1 |
| `none` | Silence |

A rejection with a non-`Error` reason prints `UnhandledPromiseRejection: This error originated either by throwing inside of an async function without a catch block, or by rejecting a promise which was not handled with .catch()…` with code `ERR_UNHANDLED_REJECTION`. **The default is right:** a rejection nobody handles is an error path nobody wrote, so the process's state is unknown. [Lesson 09](09-errors-and-crashing.md) covers what a process-level handler may and may not do; "swallow and continue" isn't on the list.

### The late catch

This handler has a `try/catch` around everything, and it still kills the process:

```ts
async function paymentWithFx(id: string) {
  try {
    const paymentP = ledger.getPayment(id);    // starts now; rejects after 10 ms (a 404 from the DB layer)
    const fx = await fxRates.current();        // takes 50 ms of real I/O
    const payment = await paymentP;            // the first handler is attached HERE, at 50 ms
    return { payment, fx };
  } catch (err) {
    return null;                               // never reached: exit code 1 at ~10 ms
  }
}
```

At 10 ms, `paymentP` rejects. The function is suspended at `await fxRates.current()`, so nothing is listening to `paymentP` yet. That timer callback ends, the queues drain, Node finds an unhandled rejection, and the process dies. **The `try` block attaches a handler only when execution reaches `await paymentP`.** If the gap were only a microtask hop (`await Promise.resolve()`), the handler would be attached before the check runs and nothing would crash, which is why this bug survives local testing and appears under production latency.

```ts
// ✅ Start them together AND attach handlers together.
const [payment, fx] = await Promise.all([ledger.getPayment(id), fxRates.current()]);
```

`Promise.all` attaches a handler to every input immediately. Use `Promise.allSettled` when one failure shouldn't discard the others. The typing of the combinators is in [TypeScript Lesson 16](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/16-async-and-errors.md).

### Floating promises

```ts
audit.record(event);                                    // ❌ nobody awaits: a rejection exits the process
await audit.record(event);                              // ✅ part of the request
void audit.record(event).catch((err) => log.error({ err }, "audit write failed")); // ✅ deliberate
```

A promise nobody awaits and nobody catches is a crash waiting for its first failure. Enable `@typescript-eslint/no-floating-promises`, and give every deliberate fire-and-forget a `.catch` (the `background()` helper in Build does it with context).

---

## 8. Sequential, parallel, and loops

The traps (`forEach` with an `async` callback, consecutive independent `await`s) are covered with their typing in [TypeScript Lesson 16](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/16-async-and-errors.md). The runtime view in one block:

```ts
for (const id of ids) await refund(id);              // sequential: total ≈ sum of latencies
await Promise.all(ids.map((id) => refund(id)));      // parallel: ≈ the slowest, but UNBOUNDED
ids.forEach(async (id) => { await refund(id); });    // ❌ forEach ignores the promises: no waiting,
                                                     //    and each rejection is unhandled → exit 1
```

**Parallel isn't free.** `Promise.all` over 10,000 items opens 10,000 requests at once: sockets, memory and a downstream rate limit all hit together. Bounded concurrency is [Lesson 10](10-cancellation-and-concurrency.md).

---

## 9. Microtask starvation: async isn't non-blocking

```ts
// ❌ Looks asynchronous. Never yields to the loop.
async function sumEntries(entries: readonly LedgerEntry[]): Promise<bigint> {
  let total = 0n;
  for (const e of entries) total += await toMinor(e);   // toMinor is async but does no I/O
  return total;
}
```

Each `await` on an already-resolved promise queues a microtask, and the loop drains microtasks *until the queue is empty* ([Lesson 05](05-the-event-loop.md)). The resumption queues the next one, so the queue never empties until the loop finishes. **Poll doesn't run, so no request is accepted, no socket is read, no timer fires.** A loop of 10 million such awaits freezes a server for seconds, with every `curl` waiting the whole time (Break 3 measures yours). The CPU profile shows no single heavy function, just millions of short microtasks.

The realistic shapes: an `async` mapper or validator over a large in-memory array; a cache that returns `Promise.resolve(hit)`; an async generator over data that's already in memory. The fixes, in order of preference:

```ts
import { setImmediate as yieldToLoop } from "node:timers/promises";

// ✅ Yield every N items: poll runs between chunks.
for (let i = 0; i < entries.length; i++) {
  total += toMinorSync(entries[i]!);
  if (i % 10_000 === 9_999) await yieldToLoop();
}
```

Better still: don't make CPU work `async` at all (make it sync and chunk it, [Lesson 08](08-blocking-the-loop.md)), or move it off the loop entirely ([Lesson 21](../05-concurrency/21-worker-threads.md)). **`async` describes how a function returns, not whether it shares the thread.**

---

## 10. `for await` in one paragraph

`for await (const x of source)` calls `source[Symbol.asyncIterator]()` and awaits each `next()`. Readable streams, `readline` interfaces and `events.on(emitter, name)` are all async iterables. Two behaviours to know now: a `break`, `return` or `throw` inside the loop calls the iterator's `return()`, which **destroys a stream** (good: no leaked file handle); and a stream `'error'` surfaces as an exception inside the loop. Async generators as pipeline stages are [Lesson 14](../03-data-and-io/14-composing-streams.md).

---

## 11. Production rules

| Rule | Why |
|---|---|
| **Never call back synchronously sometimes and asynchronously other times** | Zalgo: one call site, two execution orders |
| **Start concurrent work with `Promise.all`/`allSettled`, not "create now, await later"** | A rejection before the `await` is unhandled → exit 1 |
| **Every promise is awaited, returned, or given a `.catch`** | Floating promises crash on their first failure; enforce with `no-floating-promises` |
| **`return await` inside `try`/`catch`/`finally`** | Otherwise the catch is skipped and `finally` runs before the work ends |
| **Leave `--unhandled-rejections` at `throw`** | An unhandled rejection is an unwritten error path; limping on corrupts state |
| **Don't put check-then-act across an `await`** | Other requests run at every `await`; use one atomic statement |
| **Don't make CPU-bound code `async`; chunk it with `setImmediate`** | Resolved-promise loops starve the loop like synchronous code |
| **Prefer `await` to `.then` chains** | Async stack traces only cover suspended `async` functions |
| **Use `node:timers/promises` and `events.once` with a `signal`** | Abortable waits; no hand-rolled `new Promise(setTimeout)` |
| **Never return an object with a `then` field from an async function** | The runtime calls it and adopts its result |

> **Spring equivalent:** a `CompletableFuture` is eager like a promise, but its continuations run on whichever thread completes it or on an `Executor` you choose; in Node every continuation runs on the one loop thread. Reactor's `Mono` is *lazy*: nothing runs until someone subscribes, and disposing the subscription cancels it. Promises have no built-in cancellation, which is why Node threads an `AbortSignal` through everything ([Lesson 10](10-cancellation-and-concurrency.md)). See [Async, scheduling & events](https://github.com/Pawar-Pratik/SpringBoot/blob/main/09-production/32-async-scheduling-events.md).

---

## 12. Interview traps

**Q1. "What does `await` actually do?"**
The weak answer is "it waits for the promise". It suspends the function, returns control to the caller (handing back the function's pending promise on the first `await`), and registers a reaction on the awaited value; the rest of the function runs as a microtask once the value settles. A native promise costs one tick after settling, a plain value one tick, a foreign thenable two. Nothing blocks; other callbacks run in between. **Follow-up:** "So is code after an `await` atomic with code before it?" No: any number of other requests can run at that point, so check-then-act across an `await` is a race.

**Q2. "What does this print?"**
```ts
import { readFile } from "node:fs";
async function a() { console.log("a1"); await b(); console.log("a2"); }
async function b() { console.log("b"); }
readFile(import.meta.filename, () => {
  setTimeout(() => console.log("timeout"), 0);
  a();
  process.nextTick(() => console.log("tick"));
  Promise.resolve().then(() => console.log("promise"));
  console.log("sync");
});
```
`a1, b, sync, tick, a2, promise, timeout`. `a` runs synchronously and calls `b`, which prints and returns an already-fulfilled promise; `await` on it queues `a`'s resumption as the first microtask. The tick and the `.then` are queued, `sync` prints, the callback returns. Ticks drain (`tick`), then microtasks in order (`a2`, `promise`), and the timer fires on a later timers phase.

**Q3. "Why does `async function f() { return Promise.resolve(1) }` resolve later than `return 1`?"**
Resolving a promise with a thenable doesn't adopt it directly: it queues a job that calls the thenable's `then`, which then settles the outer promise. That's two extra microtask ticks for a native promise. It's harmless in application code, but it's why "return the promise" and "return the value" interleave differently with other microtasks.

**Q4. "What happens on an unhandled rejection in Node 24?"**
The default `throw` mode emits `'unhandledRejection'`; with no listener, the rejection is raised as an uncaught exception, and the process exits with code 1. The check runs after each callback once the microtask queue has drained, so attaching a handler within the same microtask drain is fine and attaching it after any I/O isn't. **Follow-up:** "Should you add a handler that logs and continues?" No: log with context, then exit and let the supervisor restart ([Lesson 09](09-errors-and-crashing.md)).

**Q5. "There's a try/catch around the whole handler, yet the process died with exit code 1. How?"**
The late catch: a promise was created, the function then awaited something slower, and the first promise rejected while nothing was attached to it. `try` only attaches a handler when execution reaches the `await`. Fix it by starting and awaiting together with `Promise.all` or `allSettled`. It hides in development because local latencies are tiny.

**Q6. "Why is `items.forEach(async (item) => { await save(item); })` broken?"**
`forEach` discards the returned promises: nothing waits, the code after it runs immediately, and every rejection is unhandled, which in Node means exit code 1, not a lost log line. Use `for…of` for sequential work, `Promise.all` for small parallel sets, and a concurrency limit for large ones.

**Q7. "`return` vs `return await`: when does it matter?"**
Inside `try`/`catch`/`finally`. Without `await`, the function leaves the `try` immediately: a rejection skips the `catch`, and `finally` runs while the work is still in flight (releasing a DB client mid-query, for example). Elsewhere it only affects stack traces: the non-awaiting function disappears from the async trace.

**Q8. "Is async code non-blocking?"**
Only the waiting is non-blocking. An `async` function runs on the loop thread like any other code, and a loop of awaits over already-resolved values is a chain of microtasks that the loop drains to completion before it polls again. Ten million such awaits freeze a server for seconds. `async` describes how a result is delivered, not where the CPU work happens.

**Q9. "What is Zalgo, and why does Node defer callbacks it could call immediately?"**
An API that calls back synchronously in some cases and asynchronously in others, so the same call site executes in two orders. Node always delivers operational results asynchronously (programmer errors like a bad argument type still throw synchronously), so callers can attach listeners and finish their own setup first. Promises enforce it: reactions are always microtasks.

**Q10. "Why do some frames go missing from async stack traces?"**
V8 rebuilds the async part of a trace by walking `async` functions suspended at `await`. A function that returns a promise without awaiting it isn't suspended, and callbacks attached with `.then` aren't async functions, so neither appears. Use `await`, and wrap errors with `cause` at boundaries.

**Q11. "Promise vs Java's `CompletableFuture` vs Reactor's `Mono`?"**
A promise is eager, single-valued and not cancellable, and every continuation runs on the one loop thread. `CompletableFuture` is also eager but lets you choose the executor its stages run on. `Mono` is lazy (nothing happens until subscription) and cancellable by disposing. Node gets cancellation from `AbortSignal` instead of from the promise itself.

---

## 13. Build & break

### Build — `src/lib/async.ts`

```ts
// src/lib/async.ts
import { setTimeout as delay, setImmediate as immediate } from "node:timers/promises";

/** Abortable sleep: rejects with an AbortError (cause: TimeoutError for AbortSignal.timeout). */
export function sleep(ms: number, signal?: AbortSignal): Promise<void> {
  return delay(ms, undefined, { signal });
}

/** A macrotask hop: the only correct way to let poll run inside a long loop. */
export function yieldToLoop(): Promise<void> {
  return immediate();
}

export type TaskContext = { task: string; requestId?: string; merchantId?: string };
type ErrorLog = { error(obj: object, msg: string): void };        // pino's logger fits

/** Deliberate fire-and-forget: the failure is logged WITH request context, never unhandled. */
export function background(ctx: TaskContext, work: () => Promise<unknown>, log: ErrorLog): void {
  const started = performance.now();
  let p: Promise<unknown>;
  try { p = work(); } catch (err) { p = Promise.reject(err); }   // a sync throw is handled too
  void p.catch((err: unknown) => {
    log.error({ err, ...ctx, ms: Math.round(performance.now() - started) }, "background task failed");
  });
}
```

Use it where Ledger fires work it doesn't wait for: `background({ task: "audit.record", requestId, merchantId }, () => audit.record(evt), log)`. Then write tests with `node:test`: `sleep` rejects with `name === "AbortError"` when its signal aborts; `background` logs exactly once with the context when `work` rejects, and also when `work` throws synchronously; the process survives both.

### Break — four experiments

1. **The late-catch crash.** Paste the `paymentWithFx` shape with two fake functions (`sleep(10)` then `throw`, and `sleep(50)` then return). Run it: the stack trace prints and `echo $?` shows `1`; the `catch` never runs. Replace the 50 ms sleep with `await Promise.resolve()` and watch it survive. Then fix the real version with `Promise.all`.
2. **The floating promise.** `async function audit() { throw "audit sink down"; } audit(); setTimeout(() => console.log("never"), 100);`. Read the `ERR_UNHANDLED_REJECTION` message, check the exit code, then fix it with `void audit().catch(…)` and with `background()`.
3. **The async freeze.** Start a `node:http` server, and 1 s later run `for (let i = 0; i < 10_000_000; i++) await Promise.resolve(i);`. While it runs, time a request: `time curl -s localhost:3000/`. It waits for the entire loop. Add `if (i % 10_000 === 0) await yieldToLoop();`, repeat, and compare the request latency and the loop's total runtime.
4. **The lost catch.** Write `getBalance` from §5 with a fake query that rejects after 20 ms and a `finally` that logs `"released"`. With `return`, the log prints before the rejection and the caller sees the raw error; with `return await`, the `catch` converts it and `"released"` prints after. Turn on `@typescript-eslint/return-await` and watch it flag the first version.

### Explain out loud (60–120 seconds)

1. Zalgo, and why promise reactions are always asynchronous.
2. What `await` does, step by step, and what each kind of awaited value costs in ticks.
3. Why returning a promise from an `async` function takes longer than returning a value.
4. When Node checks for unhandled rejections, what the default does, and the late-catch bug.
5. `return` vs `return await` inside `try`.
6. Why an `async` loop over resolved values freezes a server, and the fix.

---

## What's next

Everything so far happens on one thread. But `fs.readFile`, `dns.lookup`, `crypto.scrypt` and `zlib.gzip` *do* run in parallel, on four libuv pool threads that nobody sized, and that's how slow DNS ends up slowing your file reads.

Next → **[Lesson 07: The thread pool: what actually runs in parallel](07-the-thread-pool.md)**
