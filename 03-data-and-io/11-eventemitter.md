# Lesson 11 — EventEmitter: the pattern under everything

> **Why this lesson exists:** sockets, HTTP servers, requests, every stream, child processes and `process` itself are EventEmitters. Its rules are few and surprising: `emit` is synchronous, an `'error'` nobody listens to kills the process, and a listener added per request is a memory leak with a warning attached. Those three rules explain a whole class of production bugs, and "is `emit` async?" is a question people answer wrong with total confidence.

**Time:** ~55 minutes · **Prereq:** Lessons 05, 09

---

## 1. The idea in one sentence

> **`emit()` is a synchronous loop over an array of functions: nothing about it is queued, deferred or isolated, so every listener runs on the caller's stack, in order, before `emit` returns.**

---

## 2. Why it exists: many values, many listeners

A callback delivers one result to one caller. A promise delivers one value to anyone who asks. Neither fits a TCP server, which produces an unbounded number of connections over its lifetime, or a socket, which produces data, then an end, then a close, and maybe an error. Node needed **one-to-many, many-times** delivery from day one, and EventEmitter is that primitive.

Because it sits under everything, its behaviour leaks into everything:

```text
EventEmitter
 ├─ net.Server ── http.Server         'connection', 'request', 'listening', 'error', 'close'
 ├─ net.Socket (Duplex stream)        'data', 'end', 'error', 'close', 'timeout'
 ├─ stream.Readable / Writable        'data', 'end', 'drain', 'finish', 'error', 'close'
 │    └─ http.IncomingMessage, fs.ReadStream, zlib streams, …
 ├─ ChildProcess                      'exit', 'close', 'error', 'spawn'
 └─ process                           'exit', 'warning', 'uncaughtException', signals
```

Three bugs come straight from its rules:

- A slow listener (an audit write done synchronously) **stalls the request that emitted the event**, because emit waits for it.
- A socket's `'error'` with no listener **kills the process** ([L09](../02-event-loop/09-errors-and-crashing.md)).
- `config.on("change", …)` inside a request handler **leaks one closure per request**, holding each request's objects forever.

---

## 3. What `emit` actually does

Strip away the edge cases and Node's implementation is this:

```ts
// The mechanism, simplified (Node's lib/events.js does the same with more edge cases)
class MiniEmitter {
  #events = new Map<string | symbol, Function[]>();

  on(name: string | symbol, fn: Function) {
    const list = this.#events.get(name) ?? [];
    this.#events.set(name, [...list, fn]);         // append: registration order is call order
    return this;
  }

  emit(name: string | symbol, ...args: unknown[]): boolean {
    const list = this.#events.get(name);
    if (!list || list.length === 0) {
      if (name === "error") throw args[0];          // the special case (§5)
      return false;
    }
    for (const fn of [...list]) fn.apply(this, args); // a snapshot copy, then plain calls
    return true;
  }
}
```

Every surprising rule falls out of that loop:

1. **It's synchronous.** Listeners run before `emit` returns, on the emitter's call stack. There is no queue and no event-loop phase involved.
2. **Order is registration order.** `prependListener` puts a listener at the front; nothing else reorders.
3. **A throwing listener aborts the loop.** Listeners after it don't run, and the exception propagates out of `emit` into whoever called it. If that caller was a libuv callback (a socket delivering data), it's now an uncaught exception.
4. **`this` is the emitter** inside a `function` listener, and lexical inside an arrow. The classic `emitter.on("data", this.handle)` bug lives in [L02](../01-foundations/02-the-javascript-node-assumes.md).
5. **Snapshot semantics.** `emit` iterates a copy. A listener removed during an emit **still runs in that emit**; a listener added during an emit **doesn't run until the next one**.
6. **`emit` returns `true` if anyone was listening**, `false` otherwise: useful for "was this event handled?" checks.

```ts
const events = new EventEmitter();
events.on("payment.succeeded", () => console.log("2. metrics"));
events.on("payment.succeeded", () => console.log("3. audit"));

console.log("1. before emit");
events.emit("payment.succeeded");
console.log("4. after emit");          // always last: emit did all the work inline
```

```text
handler ──▶ emit("payment.succeeded") ──┬─▶ metrics listener   (runs now)
   ▲                                    ├─▶ audit listener     (runs now; 200 ms here = 200 ms added to the request)
   └──────────── returns ◀──────────────┘
```

If you want a listener's work off the caller's path, the listener has to defer it itself (queue a job, `setImmediate`, or start async work without awaiting it). The emitter will never do it for you.

---

## 4. The API surface

| Method | What it does | The detail that bites |
|---|---|---|
| `on(name, fn)` / `addListener` | Append a listener | The same function added twice runs twice |
| `once(name, fn)` | Listener removed just before its first call | `off(name, fn)` still works: Node keeps a reference from the wrapper to your function (`rawListeners` shows the wrappers) |
| `off(name, fn)` / `removeListener` | Remove one registration (the most recent match) | Needs the *same function reference*: an inline arrow can never be removed |
| `prependListener` / `prependOnceListener` | Insert at the front | Rarely needed; a smell if order matters a lot |
| `removeAllListeners(name?)` | Remove everything | **Never on an emitter you don't own**: it strips Node core's own internal listeners from sockets and streams |
| `listenerCount(name)`, `listeners`, `eventNames()` | Introspection | Your leak-hunting tools |
| `'newListener'` / `'removeListener'` | Meta-events about registrations | `'newListener'` fires *before* the listener is added |
| `setMaxListeners(n)` | Change the leak-warning threshold | §6: a detector, not a limit |

---

## 5. `'error'` is special

`emit("error", err)` with no `'error'` listener **throws `err`** (a non-`Error` value is wrapped in `ERR_UNHANDLED_ERROR`, message `Unhandled error. ('…')`). Every other event name with no listener just returns `false`. [L09](../02-event-loop/09-errors-and-crashing.md) covers why the throw becomes a crash when it comes from a libuv callback.

The rule that follows: **you own the `'error'` event of every emitter you create or receive.** A socket from `net.connect`, a stream from `fs.createReadStream`, a child from `spawn`, a server from `createServer`: each needs an `'error'` listener, or must be consumed through something that attaches one (`pipeline()`, `events.once()`, `for await`).

To *observe* errors without handling them (a metrics counter), listen on `events.errorMonitor`:

```ts
import { errorMonitor } from "node:events";

socket.on(errorMonitor, (err) => metrics.inc("socket_errors", { code: systemCode(err) }));
// still crashes if nobody listens on "error": errorMonitor never counts as handling
socket.on("error", (err) => logger.warn({ err }, "socket error"));
```

---

## 6. Listener leaks and `MaxListenersExceededWarning`

Add an 11th listener for the same event on one emitter and Node prints, once per event name:

```text
(node:7592) MaxListenersExceededWarning: Possible EventEmitter memory leak detected.
11 tick listeners added to [EventEmitter]. MaxListeners is 10. Use emitter.setMaxListeners() to increase limit
```

**It's a leak detector, not a limit.** The 11th listener is added and works; Node is guessing that nobody legitimately registers that many, so you're probably registering in a loop or per request. The guess is usually right:

```ts
// ❌ One listener per request on a process-lifetime emitter
app.get("/v1/balance", (req, res) => {
  configEvents.on("rates.updated", () => res.locals.staleRates = true);   // never removed
  // …
});
```

Each request leaves a listener behind, and **each listener's closure keeps that request's `res` (and everything it references) reachable forever**. Memory climbs with traffic, and every `rates.updated` emit now runs one more stale callback than the last, so CPU climbs too. Heap snapshots show it as thousands of closures retained by the emitter's listener array ([L27](../07-production/27-memory-and-gc.md)).

Three fixes, in order of preference:

1. **Don't subscribe per request.** One process-level listener updates shared state that requests read.
2. **If you must, unsubscribe on the request's end:** `res.once("close", () => configEvents.off("rates.updated", onRates))`, with `onRates` a named reference.
3. **For legitimately many subscribers** (SSE clients per merchant, [L20](../04-networking/20-realtime-servers.md)), use one emitter listener that fans out to a `Set` you add to and delete from.

Raising the threshold (`setMaxListeners(50)`) is right only when the count is genuinely bounded and you know the bound. `setMaxListeners(0)` switches the detector off, which is almost never what you want. Run with `--trace-warnings` to get the stack of the registration that crossed the line, and ship `process.on("warning")` to your logs so production warnings aren't lost in stderr. (EventTargets have the same detector, except that `AbortSignal`s default to no warning at all: see [L10](../02-event-loop/10-cancellation-and-concurrency.md).)

---

## 7. Async listeners and `captureRejections`

`emit` ignores listeners' return values. An `async` listener returns a promise, so its rejection goes nowhere:

```ts
events.on("payment.succeeded", async (e) => {
  await auditLog.write(e);           // rejects when the audit sink is down
});
events.emit("payment.succeeded", e); // 💥 unhandled rejection → process exits with code 1
```

An audit-log outage just took down the API. Two fixes:

- **`captureRejections`.** `new EventEmitter({ captureRejections: true })` (or `events.captureRejections = true` globally) attaches a rejection handler to promises returned by listeners. The rejection goes to the emitter's `[Symbol.for("nodejs.rejection")](err, eventName, ...args)` method if it defines one, otherwise it's emitted as `'error'`, and with no `'error'` listener **that** throws. Don't make your `'error'` listener itself `async`: its rejections aren't captured, by design, to avoid error loops.
- **Wrap at registration** (below): the listener can never throw into or reject past the emitter.

Neither makes `emit` wait. Each async listener still runs **synchronously up to its first `await`**, then `emit` moves on to the next listener and returns. Code before the first `await` adds to the caller's latency; code after it doesn't.

---

## 8. Awaiting events: `once()` and `on()`

**`events.once(emitter, name, { signal })`** returns a promise of the event's argument array. It **rejects if the emitter emits `'error'` while you wait** (unless you're waiting for `'error'` itself), and the `signal` lets you stop waiting ([L10](../02-event-loop/10-cancellation-and-concurrency.md)):

```ts
import { once } from "node:events";

server.listen(port);
await once(server, "listening");                                  // EADDRINUSE rejects here instead of crashing
const [code] = await once(child, "exit", { signal: AbortSignal.timeout(30_000) });
```

The trap: **events aren't sticky.** A promise remembers its result; an event that already fired is gone. `await once(stream, "end")` on a stream that ended a moment ago waits forever. Check the state first (`stream.readableEnded`, `server.listening`), or attach before triggering.

**`events.on(emitter, name, { signal, close })`** turns events into an async iterator of argument arrays:

```ts
import { on } from "node:events";

for await (const [event] of on(domainEvents, "payment.succeeded", { signal, close: ["shutdown"] })) {
  await projectToReadModel(event);
}
```

Two properties you must know before using it:

- **It never ends by itself.** It stops on `break`, an abort, or one of the `close` event names. An event iterator without one of those is an infinite loop by design.
- **It buffers.** Events arriving while your loop body is awaiting are queued, and by default the queue has no limit: a fast emitter and a slow consumer grow memory without bound. Recent versions add `highWaterMark`/`lowWaterMark` options that pause emitters implementing `pause()`/`resume()`; check your Node version's docs. For real flow control, use a stream ([L13](13-streams-and-backpressure.md)).

---

## 9. EventTarget vs EventEmitter

Node also ships the web platform's `EventTarget` (it's what `AbortSignal` is). Same idea, different rules:

| | **EventEmitter** | **EventTarget** |
|---|---|---|
| Payload | Any number of arguments | One `Event` object |
| Listener throws | Propagates out of `emit` to the caller; later listeners skipped | **Reported as an uncaught exception on the next tick**; `dispatchEvent` returns normally |
| `'error'` | Special: throws if unheard | Not special |
| Same function added twice | Runs twice | **Deduplicated**: runs once |
| Options | `once()` | `{ once, signal, capture, passive }` |
| Leak warning | Past 10 per event | Past 10 per type (`AbortSignal`: off by default) |
| Ecosystem | All of Node core | Web APIs: `AbortSignal`, `fetch`-related, cross-runtime libraries |

Use EventEmitter in Node-side code; it's what the ecosystem expects. Use EventTarget when you're writing a library that must run unchanged in browsers, Deno or Bun. Note the error asymmetry: **a throwing EventTarget listener can't be caught by the dispatcher at all**; it goes straight to the process.

---

## 10. Designing evented APIs

### Never emit synchronously from the constructor

```ts
// ❌ The event fires before the caller can attach a listener
class FeedConnection extends EventEmitter {
  constructor(url: string) {
    super();
    this.emit("connect", url);                      // nobody is listening yet
  }
}
const feed = new FeedConnection(url);
feed.on("connect", () => console.log("never printed"));

// ✅ Defer to just after the caller's synchronous code
class FeedConnection extends EventEmitter {
  constructor(url: string) {
    super();
    process.nextTick(() => this.emit("connect", url));
  }
}
```

Node core does exactly this: a `net.connect` to a port nobody listens on, or a `server.listen` on a port already in use, emits its `'error'` asynchronously, so the caller's `.on("error")` on the next line catches it. (Invalid *arguments*, a programmer error, still throw synchronously.) **`process.nextTick` is the right tool**: it runs after the current synchronous code (where listeners get attached) and before any I/O ([L05](../02-event-loop/05-the-event-loop.md)). The same rule applies to any method that might emit before returning: the error-reporting version of Zalgo ([L06](../02-event-loop/06-promises-and-microtasks.md)).

### Type the event map

`@types/node` makes `EventEmitter` generic over an event map, so `emit` and `on` check names and payloads. It's a type argument only, so it's fully erasable:

```ts
type DomainEventMap = {
  "payment.succeeded": [PaymentSucceeded];
  "refund.created": [RefundCreated];
};
const events = new EventEmitter<DomainEventMap>();
events.emit("payment.succeeded", { paymentId: 1 });   // ❌ compile error: wrong payload type
```

### Events vs callbacks vs promises

| Use | When |
|---|---|
| **A promise** | One result, or one failure, per call (a query, a request). Failure handling is mandatory, and an unhandled rejection is loud |
| **Events** | Many occurrences over time, optional observers, fan-out (connections, progress, domain notifications) |
| **A stream** | Many chunks where **the consumer must be able to slow the producer down** ([L13](13-streams-and-backpressure.md)) |

Exposing a raw emitter where callers need flow control is how the buffering problem from §8 reaches your users.

---

## 11. Ledger: in-process domain events, and why they're not the outbox

When a payment succeeds, several in-process things should react: a metrics counter, a structured audit log line, invalidating a cached balance. An emitter decouples the payment service from those concerns:

```ts
import { EventEmitter } from "node:events";

type Report = (err: unknown, where: string) => void;

export class DomainEvents extends EventEmitter<DomainEventMap> {
  readonly #report: Report;

  constructor(report: Report) {
    super({ captureRejections: true });             // backstop for raw on() with an async listener
    this.#report = report;
  }

  /** A listener that can never throw into, or reject past, the code that emitted. */
  listen<K extends keyof DomainEventMap & string>(
    event: K, name: string, handler: (...args: DomainEventMap[K]) => void | Promise<void>,
  ): () => void {
    const safe = (...args: DomainEventMap[K]) => {
      try {
        const r = handler(...args);
        if (r instanceof Promise) r.catch((err: unknown) => this.#report(err, `${event}:${name}`));
      } catch (err) {
        this.#report(err, `${event}:${name}`);       // a broken audit listener must not fail a payment
      }
    };
    // TypeScript can't resolve the listener type for a generic K; `safe` has exactly its parameters
    this.on(event, safe as never);
    return () => { this.off(event, safe as never); };
  }

  [Symbol.for("nodejs.rejection")](err: unknown, event: string): void {
    this.#report(err, event);
  }
}

// After the transaction commits, never before:
await withTenantTx(merchantId, (tx) => capture(tx, paymentId));   // payment row + outbox row, one transaction
domainEvents.emit("payment.succeeded", { paymentId, merchantId, amountMinor, currency });
```

**Emit after commit.** Emit inside the transaction and a listener may invalidate a cache or log an audit entry for a payment that then rolls back.

And this is **not** the outbox. The webhook to the merchant and the Console's live feed come from the `outbox` row written in the same transaction ([API L18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md)), never from this emitter:

| | **In-process emitter** | **Transactional outbox** |
|---|---|---|
| Survives a crash | No: the event lives in memory; die after commit and it's gone | Yes: it's a committed row |
| Seen by other processes | No: only this process's listeners | Yes: any relay or worker |
| Coupling to the request | Synchronous: listeners add latency and can throw into the caller | None: a relay publishes later |
| Delivery | At most once, best effort | At least once (consumers must be idempotent) |
| **Use for** | Metrics, logs, local cache invalidation: things you can afford to lose | Webhooks, the live feed, anything a merchant or another service must see |

---

## 12. Production rules

| Rule | Why |
|---|---|
| **Treat `emit` as a synchronous function call to every listener** | A slow listener is latency on the emitting request; a throwing one is an exception in the caller |
| **Every emitter you create or receive gets an `'error'` listener, or goes through `pipeline`/`once`** | An unheard `'error'` throws, usually from a libuv callback, and kills the process |
| **Listeners must not throw or reject; wrap them at registration** | One broken listener skips the rest and fails the emitter's caller |
| **Use `captureRejections` (or wrapping) for async listeners** | `emit` ignores returned promises, so a rejection is unhandled and fatal |
| **Never subscribe per request to a long-lived emitter; if you must, unsubscribe on `close`** | Each listener's closure retains its request; memory and emit cost grow with traffic |
| **Treat `MaxListenersExceededWarning` as a leak report; don't silence it with `setMaxListeners(0)`** | It's a detector, not a limit; the leak continues either way |
| **Remove listeners by named reference** | An inline arrow can never be removed |
| **Never `removeAllListeners()` on core objects** | It removes Node's internal listeners from sockets and streams |
| **Defer events from constructors with `process.nextTick`** | Synchronous emits fire before the caller can attach |
| **Check state before `await once(…)`** | Events aren't sticky; an already-fired event waits forever |
| **Give `events.on()` an exit (`signal`, `close`, `break`) and don't use it for high-volume flow** | It never ends on its own and buffers without bound by default |
| **Domain events after commit, and never for anything that must survive a crash** | In-memory and single-process; durable side effects belong in the outbox |

> **Spring equivalent:** `ApplicationEventPublisher.publishEvent` + `@EventListener` is the same pattern, and **synchronous by default too**: listeners run on the publisher's thread and an exception propagates to the publisher. `@Async` moves a listener off the thread; `@TransactionalEventListener(phase = AFTER_COMMIT)` is the "emit after commit" rule, with the same gap (the process dies after commit, the event is lost), which is why both tracks reach for an outbox ([SpringBoot L32](https://github.com/Pawar-Pratik/SpringBoot/blob/main/09-production/32-async-scheduling-events.md)).

---

## 13. Interview traps

**Q1. "Is `emit` asynchronous?"**
No. It loops over a snapshot of the listener array and calls each one synchronously, in registration order, on the caller's stack, then returns `true` if there were listeners. The confusion comes from events *originating* asynchronously (a socket's `'data'` comes from libuv), but the delivery is a plain function call.
**Follow-up:** "So how do you make a listener not slow down the emitter?" The listener defers its own work: enqueue a job, start an un-awaited async operation with a `.catch`, or `setImmediate`. The emitter won't.

**Q2. "What happens if `'error'` is emitted with no listener?"**
`emit` throws the error (non-`Error` values are wrapped in `ERR_UNHANDLED_ERROR`). Emitted from a socket or stream inside a libuv callback, that's an uncaught exception, so the process exits. `errorMonitor` listeners observe it but don't count as handling.

**Q3. "What does `MaxListenersExceededWarning` mean?"**
That one emitter has more than 10 listeners (by default) for one event, which Node treats as a likely leak, typically a listener added per request or per loop iteration without removal. It's a detector, not a limit: the listener is still added. The weak answer is "raise the limit"; the strong one finds the registration with `--trace-warnings` and fixes the lifecycle.

**Q4. "How do you await an event?"**
`await events.once(emitter, name, { signal })`, which resolves with the argument array, rejects if `'error'` fires meanwhile, and stops waiting on abort. For a sequence of events, `for await (const args of events.on(emitter, name, { signal }))`.
**Follow-up:** "What if the event already fired?" `once` waits forever: events aren't sticky. Check state first or attach before triggering.

**Q5. "EventEmitter vs EventTarget?"**
EventEmitter: any arguments, a special throwing `'error'`, listener exceptions propagate to the caller, duplicates allowed; used throughout Node core. EventTarget: the web standard (it's what `AbortSignal` is), `Event` objects, options like `signal` and `once`, duplicate registrations deduplicated, and a throwing listener is reported as an uncaught exception rather than thrown from `dispatchEvent`.

**Q6. "Why defer events emitted in a constructor?"**
Because `emit` is synchronous, an event emitted inside the constructor fires before the caller has had a chance to call `.on()`. Deferring with `process.nextTick` runs it right after the caller's synchronous code and before any I/O. Node core does this for early errors, so `.on("error")` on the next line always works.

**Q7. "A listener throws. What happens to the other listeners and to the code that called `emit`?"**
The loop aborts: listeners registered after it don't run, and the exception propagates out of `emit` into the caller. In Ledger's case that would mean an audit-log bug failing a payment request *after* the transaction committed, so the client sees a 500 for a payment that succeeded. Wrap listeners so they can't throw.

**Q8. "Can a listener be an `async` function?"**
Yes, but `emit` ignores the returned promise, so a rejection is unhandled and, by default, fatal. Enable `captureRejections` (routing it to `Symbol.for("nodejs.rejection")` or `'error'`) or wrap the listener with a `.catch`. And remember it still runs synchronously up to its first `await`.

**Q9. "What happens if a listener removes another listener during an emit?"**
Nothing for the current emit: `emit` iterates a copy taken when it started, so the removed listener still runs this time and won't run next time. Likewise a listener added during an emit first runs on the next emit.

**Q10. "Why not use an EventEmitter to trigger webhooks after a payment?"**
It's in-memory, single-process and synchronous. If the process dies between commit and emit the webhook is lost forever; other replicas never see the event; and a slow or failing listener affects the request. Webhooks need the outbox: a row in the same transaction, published at least once by a relay. The emitter is fine for metrics and logs you can afford to lose.

---

## 14. Build & break

### Build — a typed `DomainEvents` emitter
- `src/lib/domain-events.ts`: the `DomainEvents` class from §11 with the typed event map, `listen(event, name, handler)` returning an unsubscribe function, `captureRejections`, and a `report` callback wired to pino and a `listener_failures_total{event,listener}` counter.
- Wire `payment.succeeded` to three listeners: metrics, an audit log line, and cache invalidation. Emit only after the capture transaction commits.
- Tests:
  - **A slow listener blocks `emit`:** register a listener that busy-waits 200 ms; measure `emit` with `performance.now()` and assert it took ≥ 200 ms. Then make the listener `async` with `await setImmediate()` (from `node:timers/promises`) before the busy-wait, and assert `emit` returns in under 5 ms.
  - A listener that throws synchronously doesn't stop the next listener and doesn't throw from `emit`.
  - An async listener that rejects is reported, and no `unhandledRejection` occurs.
  - `listen()`'s unsubscribe leaves `listenerCount` back where it started.

### Break — five experiments
1. **The constructor emit.** Build the ❌ `FeedConnection` from §10 and attach `.on("connect")` right after `new`. Nothing prints. Switch to `process.nextTick` and it prints. Then try `setImmediate`: it also prints here, so explain why `nextTick` is still the convention (it runs before the loop moves on, so no I/O callback can deliver a competing event first).
2. **The per-request listener.** Add the ❌ `/v1/balance` handler from §6 to the Ledger API and run `npx autocannon -d 30 http://localhost:3000/v1/balance`. Watch the warning appear at the 11th request, then log `configEvents.listenerCount("rates.updated")` and `process.memoryUsage().heapUsed` every 5 seconds: both climb for the whole run. Fix it with one process-level listener and rerun: flat.
3. **The throwing listener breaks the caller.** Register an audit listener that throws (`event.metadata.orderId.trim()` on a payment without metadata), then a metrics listener after it. Create a payment: the client gets a 500, the metrics listener never ran, and the payment row exists. Work out what happens next: [API L18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md)'s rule releases the idempotency key on a 5xx, so the client's retry executes again against a payment that already succeeded. Then use `listen()` and confirm a 201 plus one `listener_failures_total` increment.
4. **The async rejection.** `events.on("payment.succeeded", async () => { throw new Error("audit sink down"); })` on a plain `EventEmitter`, then emit: the process exits with code 1. Add `captureRejections: true` and an `'error'` listener: it survives and logs. Remove the `'error'` listener: it dies again, now via `uncaughtException`.
5. **The unbounded iterator.** Consume `on(emitter, "row")` with a loop body that awaits 10 ms, while a producer emits 200,000 `row` events as fast as it can. Print `heapUsed` as it runs and watch the buffered queue grow. Explain why a Readable stream with backpressure is the right tool for this flow ([L13](13-streams-and-backpressure.md)).

### Explain out loud (90 seconds)
1. What `emit` does, in five words or fewer per rule: synchronous, ordered, snapshot, throws through, returns a boolean.
2. The `'error'` rule and `errorMonitor`.
3. Why `MaxListenersExceededWarning` is a leak detector, and the per-request listener leak.
4. Async listeners: why rejections are unhandled, and the two fixes.
5. In-process domain events vs the outbox, and which Ledger side effects belong to each.

---

## What's next

Events tell you *that* something happened. The data that arrives with them, from sockets, files and HTTP bodies, is bytes. Next: what a Buffer really is, where its memory lives, why `chunk.toString()` corrupts text, and why Ledger's webhook signatures must be computed over raw bytes rather than parsed JSON.

Next → **[Lesson 12: Buffers, bytes & encodings](12-buffers-and-encodings.md)**
