# Lesson 29 — Observability from the runtime

> **Why this lesson exists:** [API Lesson 20](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/20-observability.md) taught *what* to observe: correlation IDs, RED metrics, traces, liveness vs readiness. This lesson is *how Node produces it* cheaply and correctly: logging that doesn't block the loop, request context that survives every `await`, the runtime vital signs no JVM engineer would think to watch, and auto-instrumentation that actually loads. Get these wrong and your telemetry either lies (another request's ID on a log line, a trace with no database spans) or becomes the incident (a synchronous logger stalling every request).

**Time:** ~80 minutes · **Prereq:** Lessons 08, 25, 27

---

## 1. The idea in one sentence

> **Telemetry runs on the same thread as your requests, so it must cost almost nothing, it must follow the async call chain rather than a thread, and it must be installed before the code it observes is loaded.**

---

## 2. Why the runtime side is its own problem

API L20 gives you the vocabulary. Each of its concepts hits a Node-specific mechanism that can quietly break it:

| API L20 concept | Node mechanism underneath | How it fails in Node |
|---|---|---|
| Structured logs | A logger writing to `process.stdout` | **The write blocks the event loop**, so logging *is* the latency |
| Correlation IDs | `AsyncLocalStorage` | **The context is swapped**: a log line carries another request's ID |
| Traces | Module patching at load time | **pg was loaded first**: HTTP spans, no DB spans, nobody notices |
| Saturation (USE) | One JS thread | Thread counts mean nothing; **loop delay and ELU are the vital signs** |
| Health checks | The same loop serves `/healthz` | A blocked loop fails liveness and Kubernetes restarts a busy pod |

Every one of these is invisible on a laptop at 5 requests a second. The rest of the lesson takes them in order.

---

## 3. Logging without blocking the loop

### What `console.log` actually costs

`console.log(obj)` does two expensive things on your only JavaScript thread:

1. **It formats** with `util.inspect`, which walks the object graph, checks for cycles and builds a human-readable string. For a request-sized object that's real CPU per call.
2. **It writes to `process.stdout`**, and stdout isn't an ordinary stream. When it's a **file**, and on POSIX when it's a **terminal**, the write is **synchronous**: the loop waits for the kernel to accept the bytes. When it's a pipe (which it often is in a container, piped to the runtime's log collector), the write may be asynchronous, and `console.log` **ignores `write()`'s return value**. That's the backpressure violation from [Lesson 13](../03-data-and-io/13-streams-and-backpressure.md): a slow log consumer means log lines pile up in your process's memory.

```text
console.log({ payment })                 pino logger.info({ paymentId }, "created")
   │                                        │
   ├─ util.inspect: walk the object         ├─ level check (a no-op function below level)
   ├─ build a human string                  ├─ string concat: prefix + bindings + fields
   └─ write(2) to stdout ── loop WAITS      └─ append to an in-memory buffer
                            (file / TTY)       └─ one write(2) per ~4 KB, later
```

So `console.log` in a hot path either blocks every request or grows the heap. **Neither shows up as "logging" in your metrics; it shows up as p99 latency.**

### Why pino is fast

| Mechanism | Effect |
|---|---|
| **JSON lines built by string concatenation**, no `util.inspect` | Minimal CPU per line |
| **Child bindings serialized once**, when you call `logger.child({...})` | Per-request fields cost nothing per line |
| **Disabled levels are no-op functions** | `logger.debug(...)` in production is a call to an empty function |
| **Buffered destination** (`pino.destination({ sync: false, minLength })`) | One `write` syscall per few KB instead of per line |
| **No formatting in-process**: pretty-printing and shipping run as **transports** (`pino.transport()`) in a worker thread | Formatting CPU leaves the main loop ([Lesson 21](../05-concurrency/21-worker-threads.md)) |

Two caveats that matter. **Arguments are still evaluated**: `logger.debug(expensiveDump())` runs `expensiveDump()` even when debug is off, so guard it with `logger.isLevelEnabled("debug")`. And **a buffered destination can lose its last lines on a hard crash**, which is why the fatal handlers from [Lesson 09](../02-event-loop/09-errors-and-crashing.md) call `logger.flush()` before exiting.

```ts
// lib/telemetry/logger.ts
import pino from "pino";
import { config } from "../../config.ts";

export const logger = pino(
  {
    level: config.LOG_LEVEL,                                   // "info" in production
    base: { service: config.SERVICE_NAME, version: config.GIT_SHA }, // replaces pid + hostname
    timestamp: pino.stdTimeFunctions.isoTime,
    redact: {
      paths: [
        "req.headers.authorization", "req.headers.cookie", 'req.headers["idempotency-key"]',
        "*.password", "*.secret", "*.api_key", "*.card_number", "*.cvc",
      ],
      censor: "[REDACTED]",
    },
    serializers: { err: pino.stdSerializers.err },             // message, stack, cause chain
  },
  pino.destination({ dest: 1, sync: false, minLength: 4096 }), // buffered async writes to fd 1
);
```

### Redaction is path-based, not content-based

This is the detail that leaks card numbers. pino's `redact` replaces values **at the paths you listed**. It never scans values. `*.card_number` matches `card_number` one level down; it doesn't match `body.card.number`, and it can't match a merchant who typed a card number into `metadata.note`. **Redaction is a safety net under a policy, not the policy.** The policy is: **never log raw request bodies; log allowlisted fields through serializers**, and keep API L20's sentinel test that fails CI if a credential ever appears in captured output.

### Levels in production

`info` by default, `debug` off. For an incident, flip the level at runtime from an authenticated admin endpoint (`logger.level = "debug"`) and set a timer to flip it back. Children created after the change inherit it, and per-request children are short-lived, so it takes effect within a request. Never redeploy just to get debug logs.

---

## 4. `AsyncLocalStorage`: context that follows the call chain

API L20 showed *using* ALS for correlation. Here's what it does underneath, because "where does context get lost?" is the follow-up every interviewer asks.

### The mechanism in Node 24

In Node 24, ALS uses the **AsyncContextFrame** implementation by default. A *frame* is an immutable map from each `AsyncLocalStorage` instance to its current store. There is always exactly one current frame.

```text
als.run({ requestId: "r1" }, handler)
  │  current frame := copy of previous frame with { als → r1 }
  ▼
handler()
  │  await pool.query(...)  ──► V8 attaches the CURRENT frame to the promise reaction
  │                              (Node does the same for timers, nextTick and I/O requests:
  │                               the frame is captured when the work is scheduled)
  │        … loop runs other requests' callbacks, each with ITS frame restored …
  │  query resolves ─────────► reaction runs with the captured frame restored
  ▼
log().info(...)  →  als.getStore()  →  r1        (a lookup in the current frame)
```

- **`run(store, fn)`** makes a new frame with one entry changed, runs `fn`, then restores the previous frame.
- **`getStore()`** is a lookup in the current frame.
- **Propagation is free-riding**: V8 already tracks state for every promise reaction, and Node already wraps its own callbacks.

The older implementation built on `async_hooks`: an `init` hook copied the store onto every new async resource, and enabling promise hooks made *every promise* in the process pay. That's why older advice warned about ALS overhead. **It's cheap now, but not free**: do one `run()` per request or per job, not per log line.

### The causal rule

> A callback runs in the context that was current when it was **scheduled**: when the promise reaction, timer or I/O request was created. An event listener isn't scheduled at all: `emit` calls it synchronously, so **it runs in whatever context called `emit`**.

That one rule predicts every loss.

### Where context gets lost, or worse, swapped

| Situation | What happens | Fix |
|---|---|---|
| **A user-land queue**: a callback-style pool or batcher stores callbacks and calls them when *another* request releases a resource | The callback runs in the **releaser's** context: **the wrong request ID** | `AsyncResource.bind(cb)` at enqueue time |
| **An EventEmitter created at boot** with listeners added inside a request | The listener runs in the context of whoever emits | Bind the listener, use `EventEmitterAsyncResource`, or don't use global emitters for request work |
| **Middleware calling `next()` from a stream event** (some body parsers have done this) | Everything downstream runs in the socket's context | Register the context middleware *after* body parsing, and assert it in a test |
| **`setInterval` or a batch flusher created at boot** | Runs with the boot context (no store) | That's correct behaviour: carry IDs inside the batched items |
| **`als.enterWith(store)`** | Sets the store for the rest of the current execution and everything it schedules, including the next job in a loop | Use `run()`; treat `enterWith` as a smell |

A **lost** context is a missing field. A **swapped** context is a log line, a span, or an RLS `merchant_id` ([Lesson 25](../06-data-and-jobs/25-databases-from-node.md)) attributed to the wrong tenant. Here's the shape that causes it:

```ts
import { AsyncResource } from "node:async_hooks";

// A callback-style pool, the kind older client libraries ship
export class CallbackPool<C> {
  #free: C[] = [];
  #waiters: Array<(conn: C) => void> = [];

  acquire(cb: (conn: C) => void): void {
    const conn = this.#free.pop();
    if (conn !== undefined) { process.nextTick(cb, conn); return; } // ✅ nextTick captures the caller's frame
    this.#waiters.push(cb);                        // ❌ stored raw: the frame is forgotten
    // this.#waiters.push(AsyncResource.bind(cb)); // ✅ frozen to the acquirer's context
  }

  release(conn: C): void {
    const next = this.#waiters.shift();
    if (next) next(conn);                          // 💥 runs in the RELEASER's context
    else this.#free.push(conn);
  }
}
```

`AsyncResource.bind(fn)` is the long-standing fix. The newer static helpers do the same job: `AsyncLocalStorage.bind(fn)` binds a function to the current context, and `AsyncLocalStorage.snapshot()` returns a runner you can call later (`snap(() => work())`) to execute anything in the captured context.

### The Ledger context module

```ts
// lib/telemetry/context.ts
import { AsyncLocalStorage } from "node:async_hooks";
import type { Logger } from "pino";
import { logger } from "./logger.ts";

export type Ctx = { requestId: string; merchantId?: string; log: Logger };
const als = new AsyncLocalStorage<Ctx>();

export const runWithContext = <T>(ctx: Ctx, fn: () => T): T => als.run(ctx, fn);
export const currentContext = (): Ctx | undefined => als.getStore();
export const log = (): Logger => als.getStore()?.log ?? logger;

/** Called by auth once the merchant is known. Mutating is safe: this store belongs to one request. */
export function bindMerchant(merchantId: string): void {
  const ctx = als.getStore();
  if (!ctx) return;
  ctx.merchantId = merchantId;
  ctx.log = ctx.log.child({ merchant_id: merchantId });   // bindings serialized once, here
}
```

API L20 used pino's `mixin` to read the store on every line. Keeping a **child logger in the store** is the faster variant: the bindings are serialized once per request instead of merged per line, and `log()` still works outside a request (it falls back to the root logger), which matters for workers.

### ALS vs `ThreadLocal`

`ThreadLocal` (and MDC, which is built on it) binds state to a **thread**. That works when one request is one thread, and breaks the moment work hops threads: a `CompletableFuture` on another pool or a Reactor pipeline needs explicit context copying. ALS binds state to the **async call chain**, because there is no thread to bind to. Virtual threads make `ThreadLocal` workable again (each request owns its virtual thread), and Java's newer `ScopedValue` is the immutable, scope-bound analogue of `als.run()`.

> **Spring equivalent:** MDC is the thread-local analogue of ALS, Micrometer is `prom-client` plus OTel metrics, and Actuator's health groups are your `/healthz` and `/readyz`. See [Spring Boot Lesson 35](https://github.com/Pawar-Pratik/SpringBoot/blob/main/09-production/35-observability.md).

---

## 5. Runtime metrics: the vital signs

RED metrics tell you users are hurting. These tell you **why, in Node terms**:

| Metric | Source | What it tells you |
|---|---|---|
| **Event-loop delay** p50 / p99 / max | `perf_hooks.monitorEventLoopDelay()` (nanoseconds) | How long a ready callback waited. **The** Node vital sign ([L08](../02-event-loop/08-blocking-the-loop.md)) |
| **ELU** | `performance.eventLoopUtilization()` | Fraction of time the loop was busy. Saturation shows here before CPU% does |
| **Heap used / heap limit** | `v8.getHeapStatistics()` | Distance to a V8 "heap out of memory" ([L27](27-memory-and-gc.md)) |
| **RSS vs container limit** | `process.memoryUsage().rss` | Distance to OOMKilled (exit 137) |
| **GC pause durations** | `PerformanceObserver` on `"gc"` entries | A tail-latency source no request trace will show |
| **Active resources by type** | `process.getActiveResourcesInfo()` (experimental) | A climbing count of sockets or timers is a leak |
| **Pool total / idle / waiting** | `pool.totalCount`, `idleCount`, `waitingCount` | **Waiting > 0 means requests are queued for a connection** ([L25](../06-data-and-jobs/25-databases-from-node.md)) |

Note what's missing: **Node exposes no metric for the libuv thread-pool queue.** You infer pool saturation from `fs` and DNS latency ([L07](../02-event-loop/07-the-thread-pool.md)).

`prom-client`'s `collectDefaultMetrics()` already covers loop-delay percentiles, GC durations by kind, heap spaces, RSS and handle counts (names like `nodejs_eventloop_lag_p99_seconds` and `nodejs_gc_duration_seconds`; check your `/metrics` output for the exact set). Add what it doesn't:

```ts
// lib/telemetry/metrics.ts
import { performance } from "node:perf_hooks";
import { getHeapStatistics } from "node:v8";
import client from "prom-client";
import { pool } from "../db.ts";

client.collectDefaultMetrics();

let lastElu = performance.eventLoopUtilization();
new client.Gauge({
  name: "ledger_event_loop_utilization",
  help: "Fraction of time the event loop was busy since the previous scrape",
  collect() {
    this.set(performance.eventLoopUtilization(lastElu).utilization);  // delta since lastElu
    lastElu = performance.eventLoopUtilization();
  },
});

new client.Gauge({
  name: "ledger_heap_used_ratio",
  help: "V8 heap used divided by the heap size limit",
  collect() {
    const h = getHeapStatistics();
    this.set(h.used_heap_size / h.heap_size_limit);
  },
});

new client.Gauge({
  name: "ledger_pg_pool_clients",
  help: "pg pool clients by state",
  labelNames: ["state"] as const,
  collect() {
    this.set({ state: "total" }, pool.totalCount);
    this.set({ state: "idle" }, pool.idleCount);
    this.set({ state: "waiting" }, pool.waitingCount);   // > 0 means queued requests
  },
});
```

**Compute on scrape (`collect`), not on a `setInterval`**: it costs nothing between scrapes, and there's no timer to keep the process alive or to forget to clear. Serve `/metrics` on an internal port or behind internal auth (API L20).

---

## 6. OpenTelemetry: why load order is everything

### How auto-instrumentation works

OTel's Node auto-instrumentation **monkey-patches modules as they load**: it wraps `pg`'s `Client.prototype.query`, `http`'s `request`, Express's router and so on, so each call opens and closes a span. For CommonJS it hooks `require`; for ESM it installs a **loader hook** (registered through `node:module`'s `register()`) that intercepts `import` of the packages it knows. Either way, **a patch only applies to modules loaded after the hook is installed.** A module loaded earlier has already handed its unpatched exports to whoever imported them.

### Why it must be `--import`, not "the first line of main.ts"

```text
node main.ts                                     node --import ./instrumentation.ts main.ts
  1. resolve + load + link the WHOLE static        1. load and RUN instrumentation.ts to completion
     graph of main.ts  (pg, express, pino …)          → hooks installed, SDK started
  2. evaluate: pg first, main.ts body last         2. resolve + load + link main.ts's graph
     → main.ts line 1 starts the SDK                  → pg is intercepted as it loads   ✅
     → pg was ALREADY loaded                ❌     3. evaluate
```

In ESM, the entire static import graph is resolved, loaded and linked **before a single line of `main.ts` runs** ([Lesson 04](../01-foundations/04-modules-cjs-and-esm.md)). So no line of `main.ts` is early enough. `--import` runs a separate module graph to completion before Node even starts resolving your entry point.

```ts
// apps/api/instrumentation.ts   →   node --import ./instrumentation.ts main.ts
import { register } from "node:module";
import { NodeSDK } from "@opentelemetry/sdk-node";
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";

// ESM: install the import hook so packages imported LATER can be patched
register("@opentelemetry/instrumentation/hook.mjs", import.meta.url);

export const sdk = new NodeSDK({
  serviceName: "ledger-api",
  traceExporter: new OTLPTraceExporter(),   // endpoint from OTEL_EXPORTER_OTLP_ENDPOINT
  instrumentations: [getNodeAutoInstrumentations({
    "@opentelemetry/instrumentation-fs": { enabled: false },   // span-per-fs-call is noise
  })],
});
sdk.start();
```

The exact ESM hook wiring has changed across OTel JS releases, so follow the current OTel docs for your versions; **the mechanism, and the `--import` requirement, don't change.** Sampling is configuration, not code (`OTEL_TRACES_SAMPLER=parentbased_traceidratio`, `OTEL_TRACES_SAMPLER_ARG=0.1`; API L20 covers tail sampling). Register `sdk.shutdown()` in your lifecycle closers ([Lesson 24](../05-concurrency/24-graceful-shutdown.md)) so the last spans are flushed on SIGTERM.

Two ways to defeat module patching without noticing: **bundling** (esbuild inlines `pg` into one file, so there's no module load left to intercept) and **importing instrumentation inside the app**. Both produce traces that look fine and are missing the database.

### `diagnostics_channel`: instrumentation without patching

`node:diagnostics_channel` is a named publish/subscribe bus built into Node. Libraries publish events at interesting points; anyone can subscribe by name.

```ts
import diagnostics_channel from "node:diagnostics_channel";

// Subscribe: undici (and therefore global fetch) publishes request lifecycle events
diagnostics_channel.subscribe("undici:request:create", (message) => {
  const { request } = message as { request: { origin: string; method: string } };
  // ❌ label by origin: webhook destinations are merchant URLs, so cardinality is unbounded
  outboundRequests.inc({ target: request.origin === PROCESSOR_ORIGIN ? "processor" : "webhook" });
});

// Publish your own: the check costs a boolean when nobody listens
const created = diagnostics_channel.channel("ledger:payment:created");
if (created.hasSubscribers) created.publish({ paymentId, amountMinor });
```

**"Zero cost when nobody subscribes"** means exactly this: `hasSubscribers` is a boolean check, and well-written publishers don't even build the message object without a subscriber. Node's `http` module publishes channels too (`http.server.request.start`, `http.client.request.start` and friends). And because channels are global by name, **a subscriber doesn't care when the library was loaded**: instrumentation built on `diagnostics_channel` (OTel's undici instrumentation, for one) has no load-order trap at all.

---

## 7. Diagnostic reports

A diagnostic report is a JSON snapshot of the process: the JavaScript and native stacks, heap statistics, libuv handles, resource usage, loaded libraries, and **environment variables**.

```bash
node --report-on-fatalerror --report-uncaught-exception \
     --report-directory=/var/log/ledger --report-compact main.ts
```

| Trigger | Use |
|---|---|
| `--report-on-fatalerror` | V8 fatal errors such as heap OOM, where your JS handlers never run |
| `--report-uncaught-exception` | The state at the moment of the crash, alongside L09's guards |
| `--report-on-signal` (`SIGUSR2` by default; run it in WSL2, Windows has no `SIGUSR2`) | A live process that's wedged |
| `process.report.writeReport()` | An authenticated admin endpoint |

**Treat every report as a secret**: it contains `DATABASE_URL`, API keys, anything in the environment. Ship it to restricted storage, never to the general log stream. `--report-compact` writes it as a single JSON line.

---

## 8. The Node alert set, and Ledger's dashboard

API L20's rule stands: page on symptoms. These are the Node-specific signals that **explain** a symptom, plus the few worth paging on directly. Thresholds are starting points to tune against your SLO.

| Alert | Starting threshold | Why it's Node-specific |
|---|---|---|
| **Loop delay p99** | > 100 ms for 5 min | One slow callback delays every request in the process |
| **ELU** | > 0.9 for 10 min | The loop is saturated: add replicas or remove work, CPU% may still look fine |
| **Heap used / limit** | > 0.85 for 15 min | A heap OOM is coming; a leak is the usual cause ([L27](27-memory-and-gc.md)) |
| **Restarts** | Any `OOMKilled`; > 3 per hour otherwise | Exit 137 vs exit 1 tells you memory vs crash |
| **pg pool waiting** | > 0 for 2 min | Requests queued on connections: a leak, a slow query, or an undersized pool |
| **Oldest job age** | Above your queue SLO ([L26](../06-data-and-jobs/26-background-jobs-and-queues.md)) | The queue metric that catches a stuck dispatcher |

**Ledger's dashboard, one row per question:**

```text
Row 1  RED          rate, errors, p50/p99 by route              (API L20)
Row 2  Loop         delay p50/p99/max and ELU, per pod           ← look here first
Row 3  Memory       rss vs limit, heapUsed vs heap limit, external
Row 4  GC           pause p99, GC time per minute, by kind
Row 5  Pools        pg total/idle/waiting per process; outbound requests by target
Row 6  Queues       webhook_deliveries due, oldest pending age, DLQ size
Row 7  Process      restarts with last terminated reason, uptime, version per pod
```

Read it top to bottom: RED says users hurt, row 2 says whether the loop is the reason, rows 3–5 say what's eating it.

---

## 9. Production rules

| Rule | Why |
|---|---|
| **No `console.log` in request paths; pino with a buffered destination** | Synchronous writes to files and TTYs block the loop; pipes buffer unboundedly |
| **Pretty-printing and log shipping in transports, never in-process** | Formatting CPU belongs off the main thread |
| **Never log raw bodies; allowlist fields; keep the sentinel test** | Redaction only covers paths you predicted |
| **Flush the logger in fatal handlers** | A buffered destination loses its last lines on exit |
| **One `als.run()` per request or job; never `enterWith`** | `enterWith` leaks context into whatever runs next |
| **`AsyncResource.bind` every callback you store for later** | Otherwise it runs in someone else's context: the wrong tenant |
| **Load OTel with `--import`; don't bundle instrumented packages** | ESM links the whole graph before your first line runs |
| **Prefer `diagnostics_channel`-based instrumentation where it exists** | No load-order dependency, no patching |
| **Export loop delay, ELU, heap/limit, GC and pool waiting** | They explain RED symptoms in Node terms |
| **Compute metrics on scrape, not on intervals** | Free between scrapes; no timers to leak |
| **Metric labels stay bounded: no origins, IDs or URLs** | A label per merchant endpoint is a cardinality bomb |
| **Diagnostic reports go to restricted storage** | They contain the environment, secrets included |

---

## 10. Interview traps

**Q1. "Why is pino faster than `console.log`?"**
The weak answer is "because it's async." The strong answer names three mechanisms: `console.log` formats with `util.inspect` and writes synchronously when stdout is a file or a POSIX terminal, so the loop waits for the kernel; pino builds JSON by string concatenation with child bindings pre-serialized, buffers writes into few syscalls, and moves formatting and shipping into a worker-thread transport. Then add the cost of the alternative: when stdout is a pipe, `console.log` ignores backpressure, so a slow collector grows your heap.
**Follow-up:** "So is async logging free?" No: a crash can lose buffered lines, so fatal handlers must flush, and arguments to disabled log calls are still evaluated.

**Q2. "How does `AsyncLocalStorage` work?"**
In Node 24, through AsyncContextFrame: an immutable map of ALS instances to stores is the "current frame"; V8 attaches the current frame to each promise reaction when it's created and restores it when the reaction runs, and Node does the same for timers, `nextTick` and I/O callbacks. `run()` swaps in a new frame for the duration of a function; `getStore()` is a lookup. The older `async_hooks` implementation copied the store onto every async resource, which is why ALS used to have a reputation for overhead.

**Q3. "Where does ALS lose context, and what's worse than losing it?"**
Anything that stores a callback and invokes it later from another code path: callback-style pools, batchers, event emitters created outside the request, middleware that calls `next()` from a stream event. The rule is that callbacks run in the context current when they were scheduled, and listeners run in the emitter's caller context. Worse than losing context is **swapping** it: a pool that calls a waiter from the releasing request's context attaches request A's ID, or merchant A's RLS setting, to request B. Fix with `AsyncResource.bind` at the point you store the callback.

**Q4. "ALS vs `ThreadLocal`?"**
`ThreadLocal` follows the thread, so it works for thread-per-request and breaks when work hops pools (MDC needs explicit copying for `CompletableFuture` or Reactor). ALS follows the causal async chain, because Node has no per-request thread. Virtual threads make `ThreadLocal` viable again, and `ScopedValue` is the closer analogue to `als.run()`: immutable and bound to a scope.

**Q5. "Which Node-specific metrics would you alert on?"**
Event-loop delay p99, ELU, heap used over heap limit, restarts split by reason (OOMKilled vs crash), and pg pool waiting count, plus oldest-job age for workers. Then explain why CPU% isn't on the list: a Node process blocked on one long callback can show modest CPU across the node while every request waits, and ELU measures the thing that's actually saturated.

**Q6. "Why must tracing load before your app code?"**
Auto-instrumentation patches modules as they load, and a patch only applies to modules loaded after the hook is installed. In ESM the whole static graph is linked before the entry module's first line executes, so the SDK has to start in a `--import` preload. The symptom of getting it wrong is subtle: HTTP spans appear, database spans don't.
**Follow-up:** "We use `--import` and still have no pg spans." Check for bundling (the package was inlined, so there's no module load to hook) or a native driver the instrumentation doesn't cover.

**Q7. "What is `diagnostics_channel`?"**
A built-in, name-based publish/subscribe bus for diagnostics. Libraries such as undici and Node's own `http` publish lifecycle events; tools subscribe. It's zero cost without subscribers because publishing is guarded by a boolean `hasSubscribers` check, and since channels are global by name, subscribers work regardless of load order, unlike monkey-patching.

**Q8. "A trace shows 400 ms of nothing between two spans. What is it?"**
Uninstrumented waiting. In Node the usual suspects are the callback waiting for the event loop (check loop delay at that timestamp), waiting for a pool connection (pool waiting count), or CPU work in your own code between awaits (a CPU profile, [L28](28-performance-and-v8.md)). The gap is a clue, not an absence of data.

**Q9. "Should you log request bodies?"**
No. pino's redaction matches paths you listed, so a card number in a free-text field or a nested object you didn't predict sails through. Log allowlisted fields through serializers, and keep a test that sends a sentinel credential and asserts it never appears in captured logs.

**Q10. "What's in a Node diagnostic report, and why be careful with one?"**
Stacks (JS and native), heap statistics, libuv handles, resource usage, loaded libraries and the full environment. It's the best artefact for a fatal error your handlers never saw, and it contains every secret in `process.env`, so it goes to restricted storage.

---

## 11. Build & break

### Build — `lib/telemetry/`
1. `logger.ts` (§3): pino, buffered destination, redaction, `err` serializer; `pino-pretty` only as a dev transport.
2. `context.ts` (§4): `runWithContext`, `log()`, `bindMerchant()`; the middleware sits after body parsing and sets `x-request-id`.
3. `metrics.ts` (§5): defaults plus ELU, heap ratio and pool gauges; `/metrics` on an internal port.
4. `apps/api/instrumentation.ts` (§6), loaded with `--import`; `sdk.shutdown()` registered as a lifecycle closer.
5. `GET /admin/debug/report` (authenticated) that calls `process.report.writeReport()` into restricted storage.
6. The alert rules from §8 as a Prometheus rules file, and the dashboard rows as a JSON spec.

A test for the context: two concurrent requests through the full middleware stack, each asserting that every log line it produced carries its own `request_id`.

### Break — five experiments
1. **`console.log` vs pino.** Log five lines per request in a handler, run `node main.ts > out.log`, and load it with `npx autocannon -c 100 -d 20`. Record RPS and loop p99 from L08's meter. Switch to the pino logger and repeat. Then run the `console.log` version with output to a terminal. Write down all three; the gap is your logging tax.
2. **Swapped context.** Put §4's `CallbackPool` (size 1) in front of a fake query and fire two concurrent requests. Request B's log line from inside the acquire callback carries **request A's ID**. Wrap the stored callback in `AsyncResource.bind` and watch it correct itself.
3. **Instrumentation too late.** Move the `NodeSDK` start into the top of `main.ts` (after its imports) and run a payment. The trace has HTTP spans and no `pg` spans. Restore `--import` and watch them appear. Bonus: bundle with esbuild and watch them vanish again.
4. **The card number redaction missed.** Log the raw body of `POST /v1/payments` with `metadata: { note: "4242 4242 4242 4242" }`. It's in the log, `redact` notwithstanding. Replace body logging with an allowlisted serializer and add the sentinel test so it can't come back.
5. **The report is a secret.** Call `process.report.writeReport()` and search the file for `DATABASE_URL`. It's there in full. Point the report directory at restricted storage.

### Explain out loud (60–120 seconds)
1. Why `console.log` hurts in hot paths, and the three mechanisms that make pino fast.
2. How ALS propagates in Node 24, and the causal rule that predicts every loss.
3. Lost vs swapped context, and `AsyncResource.bind`.
4. The Node vital signs and why ELU beats CPU%.
5. Why OTel must load with `--import`, and why `diagnostics_channel` doesn't need to.

---

## Module 7 complete — checkpoint

Cold, no notes:

- [ ] V8's generational heap: new space and the Scavenger, old space and mark-sweep-compact, promotion
- [ ] `rss` vs `heapUsed` vs `external` vs `arrayBuffers`, and why RSS exceeds the heap
- [ ] Heap OOM vs OOMKilled (exit 137), and setting `--max-old-space-size` to ~75% of the container limit
- [ ] Five leak patterns, and the bounded LRU that fixes the most common one
- [ ] The three-snapshot technique, retaining paths, and what a snapshot costs in production
- [ ] Measure first: autocannon percentiles, the knee, and coordinated omission
- [ ] Reading a flame graph: width is time, self vs total, wide plateaus
- [ ] Ignition → Sparkplug → Maglev → TurboFan, and what causes a deopt
- [ ] Hidden classes and inline caches: monomorphic → polymorphic → megamorphic
- [ ] Why micro-benchmarks lie
- [ ] Why `console.log` blocks and pino doesn't; why redaction is a safety net, not a policy
- [ ] How ALS propagates (AsyncContextFrame), where it loses or swaps context, and the fix
- [ ] The Node vital signs: loop delay, ELU, heap/limit, GC pauses, pool waiting
- [ ] Why OTel must load via `--import`, and what `diagnostics_channel` changes
- [ ] What a diagnostic report contains, and why it's a secret

---

## What's next

You can now see inside a running Node process. Module 8 is about shipping one safely, starting with the code you didn't write: thousands of transitive packages that can run scripts on your machine at install time, plus the JavaScript-specific bug classes, like prototype pollution, that turn a JSON body into an admin flag.

Next → **[Lesson 30: Security & the supply chain](../08-shipping/30-security-and-supply-chain.md)**
