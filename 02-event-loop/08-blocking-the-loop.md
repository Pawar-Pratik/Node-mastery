# Lesson 08 — Blocking the loop — and measuring it

> **Why this lesson exists:** "everything got slow at once and CPU was low" is the signature Node incident. No errors, no crash, one core busy, p99 up on every endpoint, and a dashboard that says the box is mostly idle. You need three skills to deal with it: **prove** the loop is blocked with a metric, **find** the blocking frame with a profile, and **fix** it at the layer that allowed the work to get big. You'll also build the ten-line loop meter that every later experiment in this track runs with.

**Time:** ~85 minutes · **Prereq:** Lessons 05–07

---

## 1. The idea in one sentence

> **A blocked event loop doesn't show up as errors or high CPU; it shows up as event-loop delay. Prove it with a delay histogram and ELU, find it with a CPU profile, and fix it at the layer that let one callback's work grow without bound.**

---

## 2. Why it exists: the incident nobody's dashboard catches

A 4-core pod runs a Node API. One endpoint starts receiving bigger payloads. Then:

- **p99 triples on every endpoint**, including `/healthz`, because every request shares the one loop thread ([Lesson 05](05-the-event-loop.md)).
- **CPU reads about 25–30%.** The loop thread is pinned at 100% of *one* core, and one core out of four averages to 25%. The dashboard says "plenty of headroom".
- **No errors.** Nothing throws. Requests just wait, then time out at the load balancer.
- **Health checks start failing**, the orchestrator restarts the pod, the traffic moves to the next pod, and the same payload blocks it too.

In Tomcat, a CPU-heavy handler costs one of ~200 threads and the rest keep serving. **In Node it costs the process.** Throughput is how fast each callback hands the thread back, so the only metric that sees this incident directly is how late the loop is running.

---

## 3. What blocks

| Category | Examples | Why it blocks |
|---|---|---|
| **`*Sync` APIs** | `readFileSync`, `pbkdf2Sync`, `gzipSync`, `execSync` | The loop thread does the wait or the work itself |
| **Large JSON** | `JSON.parse` of a big body, `JSON.stringify` of a big result | Both are synchronous and proportional to size: tens of MB costs tens to hundreds of milliseconds |
| **CPU-heavy JavaScript** | Sorting or deduplicating huge arrays, an accidental O(n²) loop over request data, template rendering, big `Buffer` transforms | Runs to completion, no yield points |
| **Synchronous crypto on big inputs** | `createHash().update(hugeBuffer)`, sync signing in a loop | Hashing a few bytes is free; hashing hundreds of MB isn't |
| **Regular expressions** | ReDoS: catastrophic backtracking | One `test()` call can run for seconds or minutes |
| **Microtask chains** | An `async` loop over already-resolved values | Never yields to poll ([Lesson 06](06-promises-and-microtasks.md)) |

### ReDoS, mechanically

V8's regex engine **backtracks**: when a match fails, it goes back and tries every other way the pattern could have matched the input so far. Most patterns have one or two ways. Nested quantifiers have exponentially many:

```ts
const words = /^(\w+\s?)*$/;                       // "words separated by optional spaces"
words.test("a".repeat(25) + "!");                   // ❌ seconds of one CPU core, then false
```

With `\s?` optional, a run of `n` letters can be split between iterations of the group in 2^(n−1) ways (`aaa` is `a|a|a`, `aa|a`, `a|aa` or `aaa`). The trailing `!` makes every split fail at the `$`, so the engine tries all of them. **Each extra character doubles the time**: around 20 characters you can measure it, before 30 you're waiting seconds to minutes, and the whole time the loop is frozen.

The shapes to recognise: **nested quantifiers** (`(a+)+`, `(\w+\s?)*`), **alternations that overlap** (`(\w|\d)+`), and **a quantified run followed by something that can fail**. That last one doesn't need nesting to hurt:

```ts
/\s+$/.test(" ".repeat(20_000) + "x");   // n + (n−1) + … + 1 ≈ 200 million steps
```

The engine tries a match starting at each of the 20,000 spaces, scans to the `x`, fails, and moves one character on. That's **quadratic**, not exponential, and still enough to take a site down. **Stack Overflow's July 2016 outage** was a whitespace-trimming regex backtracking across a post containing ~20,000 whitespace characters. **Cloudflare's July 2019 outage** was a WAF rule's regex backtracking until CPUs were exhausted: not Node, but the same mechanism.

The defences, cheapest first:

1. **Cap the input length before any regex runs.** Bounding `n` bounds 2^n. This alone defuses most real cases.
2. **Write unambiguous patterns.** `/^\w+(\s\w+)*$/` accepts the same sentences as the pattern above, except an empty string and a trailing space, with the separator mandatory, so there's only one way to match; it rejects 100,000 characters in well under a millisecond. Decide deliberately about the cases that changed.
3. **Use a linear-time engine for anything you don't fully control.** The `re2` package binds Google's RE2: no backtracking, linear time, and in exchange no backreferences or lookaround.
4. **Never compile a user-supplied pattern** with `new RegExp(input)`. That's ReDoS as a feature.

The OWASP framing and body limits are in [API Lesson 16](https://github.com/Pawar-Pratik/API-mastery/blob/main/03-security/16-owasp-and-hardening.md); this lesson is the runtime mechanism.

---

## 4. Measuring

### Event-loop delay: `monitorEventLoopDelay()`

```ts
import { monitorEventLoopDelay } from "node:perf_hooks";

const h = monitorEventLoopDelay({ resolution: 10 });   // sample every 10 ms
h.enable();
// later: h.percentile(50), h.percentile(99), h.max, h.mean (all in NANOSECONDS); h.reset()
```

It runs an internal timer every `resolution` ms and records **how long each gap between ticks actually was** in an HDR histogram. Two consequences for reading it:

- **An idle loop never reads zero.** Each sample includes the sampling interval itself, so an idle process reads roughly the resolution. On Windows, coarse OS timers make that floor higher and noisier; judge spikes, not the floor, or measure in WSL2.
- **A blocked loop reads the block.** If a callback runs for 300 ms, one sample is ≈300 ms, and p99 and max show it. **p99 and max are the numbers that matter;** the mean hides a 300 ms stall among hundreds of 10 ms samples.

### Event-loop utilisation: `performance.eventLoopUtilization()`

ELU is the fraction of wall-clock time the loop spent **outside** the kernel wait in poll, that is, running something.

```ts
import { performance } from "node:perf_hooks";

const before = performance.eventLoopUtilization();
// … 5 seconds later …
const after = performance.eventLoopUtilization();
const { utilization } = performance.eventLoopUtilization(after, before);   // 0.0 – 1.0 over the window
```

**Why ELU beats CPU% for Node:**

| | CPU% | ELU |
|---|---|---|
| Scope | Every thread in the process (loop, GC helpers, libuv pool), averaged over cores | **The loop thread only** |
| A pinned loop on a 4-core box | ~25% | **~1.0** |
| `readFileSync` waiting on a slow disk | Low (the thread is asleep in a syscall) | **High** (the loop isn't available) |
| Pool threads busy with `scrypt` | High | Unaffected (correctly: the loop is free) |

Use both metrics together. **Delay tells you the latency impact now; ELU tells you how much headroom is left.** A healthy server at high throughput can show ELU at 0.8 with low delay: busy, not blocked. ELU near 1.0 with delay climbing means saturated.

### The no-API version: interval drift

```ts
let expected = performance.now() + 100;
setInterval(() => {
  const lag = performance.now() - expected;         // how late this tick ran
  expected = performance.now() + 100;
  if (lag > 50) console.warn(`loop lag ${lag.toFixed(0)} ms`);
}, 100).unref();
```

It's what `monitorEventLoopDelay` does natively, minus the histogram. It works in any JavaScript runtime, and it's the version to write on a whiteboard.

### The ten-line loop meter

This is the meter the README promised. Every experiment from here on runs with it preloaded:

```ts
// tools/loop-meter.ts — run any script with: node --import ./tools/loop-meter.ts app.ts
import { monitorEventLoopDelay, performance } from "node:perf_hooks";

const h = monitorEventLoopDelay({ resolution: 10 });
h.enable();
let last = performance.eventLoopUtilization();
const ms = (ns: number) => (ns / 1e6).toFixed(1);
setInterval(() => {
  const now = performance.eventLoopUtilization();
  const { utilization } = performance.eventLoopUtilization(now, last);
  last = now;
  console.error(`[loop] p50=${ms(h.percentile(50))}ms p99=${ms(h.percentile(99))}ms max=${ms(h.max)}ms elu=${utilization.toFixed(2)}`);
  h.reset();
}, 1000).unref();
```

`--import` preloads it before your entry module, so you never edit the code under test. It writes to stderr so it doesn't mix with program output, and `unref()` means it never keeps a finished script alive. An idle line looks like `p99=11ms elu=0.01`; a blocked one like `p99=2240ms elu=0.80`.

---

## 5. Finding the frame: `--cpu-prof`

The metrics prove the loop is blocked. A CPU profile tells you **which function** blocked it.

```bash
node --cpu-prof --cpu-prof-dir=./prof src/main.ts
# drive load at it, then stop it; a CPU.<date>.<pid>….cpuprofile file appears in ./prof
```

The profile is written when the process exits normally, which includes `process.exit()`. Give the server a `process.on("SIGINT", () => process.exit(0))` handler so Ctrl+C produces a file. Open it in Chrome DevTools (the Performance panel loads `.cpuprofile` files) and look for **the widest plateau under a request handler**: `JSON.stringify`, a `RegExp` exec, a `*Sync` call, your own sort. Width is time. Reading flame graphs properly, sampling in production with the `inspector` module, and the V8 side are [Lesson 28](../07-production/28-performance-and-v8.md).

---

## 6. Fixes, by layer

The right fix depends on **why the work was allowed to get big**. Work up the list:

| Layer | Fix | Example |
|---|---|---|
| **Input** | Cap sizes before doing work | `express.json({ limit: "100kb" })`; `limit` max 100 ([API Lesson 08](https://github.com/Pawar-Pratik/API-mastery/blob/main/02-rest-design/08-collections-and-pagination.md)); length caps before regexes |
| **Data flow** | Stream instead of buffering | A cursor → CSV → gzip pipeline instead of building one huge string ([Lesson 13](../03-data-and-io/13-streams-and-backpressure.md)) |
| **Algorithm** | A better complexity or engine | A `Set` instead of nested `includes`; RE2 or an unambiguous pattern |
| **Scheduling** | Partition the work and yield between slices | Below |
| **Thread** | Move it off the loop | A worker-thread pool for genuinely CPU-bound jobs ([Lesson 21](../05-concurrency/21-worker-threads.md)) |

**Partitioning** keeps CPU work on the loop but gives poll a turn between slices. Yield on a time budget rather than an item count, so slices stay short whatever each item costs:

```ts
import { setImmediate as yieldToLoop } from "node:timers/promises";

// ✅ Runs fn over every item, handing the loop back every ~10 ms.
export async function forEachYielding<T>(items: Iterable<T>, fn: (item: T) => void, budgetMs = 10): Promise<void> {
  let sliceStart = performance.now();
  for (const item of items) {
    fn(item);
    if (performance.now() - sliceStart > budgetMs) {
      await yieldToLoop();            // a macrotask: poll runs, other requests are served
      sliceStart = performance.now();
    }
  }
}
```

It must be `setImmediate`. `await Promise.resolve()` and `process.nextTick` don't let poll run ([Lesson 05](05-the-event-loop.md)), and `setTimeout(0)` waits at least 1 ms per slice. Two costs to say out loud: the job's total time grows slightly, and **other requests now run between slices**, so any shared state the job reads can change under it. Partitioning improves fairness; it doesn't reduce work. When the job is seconds of CPU, use a worker.

One trap with workers: parsing a huge JSON string in a worker and posting the *object* back doesn't help much, because the main thread has to deserialise the structured clone, which costs about as much as the parse. **Move the work that needs the parsed data into the worker, and post back a small result.**

---

## 7. Load shedding on loop pressure

When the loop is saturated, accepting more work makes everything worse: every queued request waits, most time out at the load balancer anyway, and clients retry. **Rejecting fast with a cheap 503 keeps the process partly available** ([API Lesson 18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md) has the general argument). In Node the pressure signal is the loop itself: shed when recent loop-delay p99 or ELU crosses a threshold. Fastify's `@fastify/under-pressure` plugin is the packaged version of this pattern; it checks loop delay, ELU, heap and RSS.

Three details separate a shedder that helps from one that hurts:

- **It can't run during a block.** Nothing runs then, the shedder included. What it does is answer the backlog cheaply once the block ends, instead of starting more heavy work.
- **Hysteresis.** Start shedding above one threshold and stop below a lower one, or the process oscillates between shedding and overload.
- **Never shed liveness.** Mount the shedder after `/healthz`. If liveness fails under load, Kubernetes restarts a pod that was merely busy, its traffic lands on the others, and they fail too. Readiness may reflect pressure; liveness must not.

---

## 8. Ledger: three blockers waiting to happen

1. **`GET /v1/payments?limit=500000`.** If the `limit` cap from [API Lesson 08](https://github.com/Pawar-Pratik/API-mastery/blob/main/02-rest-design/08-collections-and-pagination.md) is ever missing, one request loads 500,000 rows and `JSON.stringify`s a response of hundreds of megabytes: a multi-second block, plus the memory spike [Lesson 27](../07-production/27-memory-and-gc.md) will explain. **Fix at the input layer:** the zod schema caps `limit` at 100, and bulk access goes through the export.
2. **The naive statement export.** `rows.map(toCsvLine).join("\n")` over 4M payments builds a ~300 MB string on the loop thread, then sends it. It blocks while building, and the string itself is close to V8's maximum string length (about 512 MiB on 64-bit; beyond it you get `ERR_STRING_TOO_LONG`). **Fix at the data-flow layer:** a cursor → CSV → gzip → response pipeline with backpressure ([Lesson 13](../03-data-and-io/13-streams-and-backpressure.md)).
3. **A merchant-supplied regex filter** for searching payment `metadata`. Any merchant could freeze the API with one crafted pattern. **Don't build it.** Offer structured filters (equals, prefix) that Postgres evaluates with an index; if pattern search is ever unavoidable, use RE2 with a pattern-length cap and an input-length cap.

---

## 9. Production rules

| Rule | Why |
|---|---|
| **Alert on loop-delay p99 and ELU, not CPU%** | A pinned loop on a 4-core box reads as 25% CPU |
| **Run the loop meter in every experiment and every service** | Blocking is invisible in every other metric |
| **Cap every input before processing it: body size, page size, string length** | Work proportional to input is a blocker waiting for a big enough input |
| **Never put `*Sync` I/O or crypto in a request path** | It stops every request for its duration; startup and CLIs are fine |
| **Length-cap input before any regex; never compile user patterns** | ReDoS turns 30 characters into a frozen process |
| **Stream large responses and files; never build them as one string** | Building blocks the loop and spikes memory |
| **Partition long CPU work with `setImmediate` on a time budget** | Microtask yields don't let poll run |
| **Move CPU-bound jobs to worker threads, and send back small results** | Cloning a large result back costs the main thread again |
| **Shed load on loop pressure with hysteresis, never on liveness** | Partial availability beats a restart cascade |
| **Profile before fixing** | The frame you'd guess is often not the widest one |

> **Spring equivalent:** the profiling half maps directly: `--cpu-prof` flame graphs are async-profiler or JFR flame graphs, read the same way ([Performance & JVM tuning](https://github.com/Pawar-Pratik/SpringBoot/blob/main/09-production/36-performance-and-jvm-tuning.md)). The blast radius doesn't: a CPU-heavy handler in Tomcat ties up one of its ~200 threads, while in Node it ties up the only thread that runs JavaScript. That's why Node needs a loop-delay metric and the JVM doesn't have one.

---

## 10. Interview traps

**Q1. "CPU is at 30%, but p99 latency tripled across every endpoint. What's happening?"**
The weak answer is "the database is slow" or "add replicas". Every endpoint slowing together, with low CPU and no errors, is the signature of a blocked event loop: one core pinned by the loop thread reads as about 25% on a 4-core machine. Prove it with loop-delay p99 and ELU near 1.0, find the frame with a CPU profile, then fix at the input, data-flow or algorithm layer. **Follow-up:** "What if loop delay is normal?" Then look at the thread pool ([Lesson 07](07-the-thread-pool.md)) or downstream calls.

**Q2. "How do you measure event-loop blocking?"**
`perf_hooks.monitorEventLoopDelay()` gives a histogram of how late an internal timer fires, in nanoseconds; watch p99 and max per window, then reset. `performance.eventLoopUtilization()` gives the fraction of time the loop was busy. Without either, an interval that measures its own drift does the same job. Export them as metrics and alert on them ([Lesson 29](../07-production/29-observability.md)).

**Q3. "What is ELU, and why is it better than CPU% for Node?"**
The fraction of time the loop spent running, rather than waiting in the kernel in poll. It measures the one thread that matters: CPU% averages over all threads and cores, so a pinned loop looks like 25%, and a loop stuck in `readFileSync` on a slow disk looks idle. ELU near 1.0 means no headroom. Pair it with delay, because high ELU with low delay just means busy.

**Q4. "What is ReDoS, and how do you prevent it?"**
A backtracking regex engine trying exponentially (or quadratically) many ways to match an input that ultimately fails, as with nested quantifiers like `(\w+\s?)*` against a crafted string. In Node it freezes the whole process, not one request. Prevent it with input length caps first, then unambiguous patterns, RE2 for untrusted cases, and no user-supplied patterns. Stack Overflow's 2016 outage is the textbook case.

**Q5. "How would you process a 1 GB JSON file?"**
Not with `JSON.parse`: reading it as one string fails outright, because V8's maximum string length is about 512 MiB, and parsing anything close to that would block the loop for seconds. Stream it: prefer NDJSON (one record per line, split and parse line by line), or use a streaming JSON parser, and process records in a `pipeline` with backpressure, moving CPU-heavy per-record work to a worker pool.

**Q6. "When is `readFileSync` acceptable?"**
At startup (loading config, certificates, templates once), in CLIs and build scripts, and anywhere nothing else is waiting on the loop. Never in a request path, because every concurrent request waits for the disk.

**Q7. "Why chunk CPU work with `setImmediate` rather than `await Promise.resolve()`?"**
Promise and `nextTick` continuations are drained before the loop moves on, so poll never runs and no request is served between chunks. `setImmediate` is a macrotask in the check phase, so a full poll happens between slices. `setTimeout(0)` works but costs at least 1 ms per slice.

**Q8. "You add a load shedder that returns 503 when loop delay is high. What can go wrong?"**
It can't act during a block, only afterwards. Without hysteresis it oscillates. If it sheds `/healthz`, Kubernetes restarts busy pods and the load cascades onto the rest. And if it uses a window that's too long, it keeps shedding after the pressure has gone. Mount it after liveness, use two thresholds and a short window.

**Q9. "Would moving `JSON.parse` of a 50 MB body to a worker thread fix the blocking?"**
Partly at best. The parse moves off the loop, but posting the resulting object back means the main thread deserialises a structured clone of the same size. Either do the work that needs the parsed data inside the worker and return a small result, or avoid the giant body altogether with a size limit and a streaming format.

**Q10. "How do you find what's blocking the loop in production?"**
Correlate loop-delay spikes with request logs to find the endpoint and payload, then take a CPU profile under representative load (`--cpu-prof` in staging, or an on-demand `inspector` session in production) and look for the widest frames under the handler. A blocking callback usually has a recognisable name: `JSON.stringify`, a regex, a `*Sync` call, or a sort.

---

## 11. Build & break

### Build — `src/lib/loop-monitor.ts` and a load shedder

```ts
// src/lib/loop-monitor.ts
import { monitorEventLoopDelay, performance, type EventLoopUtilization } from "node:perf_hooks";

export type LoopSnapshot = { p50Ms: number; p99Ms: number; maxMs: number; elu: number };

const histogram = monitorEventLoopDelay({ resolution: 10 });
let lastElu: EventLoopUtilization = performance.eventLoopUtilization();
let latest: LoopSnapshot = { p50Ms: 0, p99Ms: 0, maxMs: 0, elu: 0 };
const toMs = (ns: number) => Math.round(ns / 1e4) / 100;          // ns → ms, 2 decimals

function sample(): LoopSnapshot {
  const now = performance.eventLoopUtilization();
  const { utilization } = performance.eventLoopUtilization(now, lastElu);
  lastElu = now;
  latest = {
    p50Ms: toMs(histogram.percentile(50)),
    p99Ms: toMs(histogram.percentile(99)),
    maxMs: toMs(histogram.max),
    elu: Math.round(utilization * 1000) / 1000,
  };
  histogram.reset();
  return latest;
}

/** Samples every `everyMs`; the shedder reads `loopSnapshot()`, the logger gets every sample. */
export function startLoopMonitor(onSample: (s: LoopSnapshot) => void, everyMs = 1_000): () => void {
  histogram.enable();
  const timer = setInterval(() => onSample(sample()), everyMs);
  timer.unref();
  return () => { clearInterval(timer); histogram.disable(); };
}

export const loopSnapshot = (): LoopSnapshot => latest;
```

Wire it in `main.ts`: sample every second, and log one line every 5 s (`log.info({ loop: snapshot }, "loop")`). Lesson 29 exports the same numbers as metrics.

Then `src/middleware/load-shedder.ts`, an Express middleware with this contract:

- Reads `loopSnapshot()` on every request: it's a property read, so the shedder itself is free.
- Starts shedding when `p99Ms > 200` **or** `elu > 0.9`; stops only when `p99Ms < 100` **and** `elu < 0.9` (hysteresis). Make the thresholds configuration, validated at boot.
- While shedding: `503` with `Retry-After: 2` and an RFC 9457 body (`code: "overloaded"`, `request_id`), following [API Lesson 10](https://github.com/Pawar-Pratik/API-mastery/blob/main/02-rest-design/10-errors-and-problem-details.md). Count shed requests in a metric.
- Mounted **after** `/healthz` and before the body parser, so a shed request costs no parsing.

```ts
// src/middleware/load-shedder.ts
import type { RequestHandler } from "express";
import { loopSnapshot } from "../lib/loop-monitor.ts";

export type ShedConfig = { startP99Ms: number; stopP99Ms: number; maxElu: number };

export function loadShedder(cfg: ShedConfig): RequestHandler {
  let shedding = false;
  return (req, res, next) => {
    const { p99Ms, elu } = loopSnapshot();
    if (!shedding && (p99Ms > cfg.startP99Ms || elu > cfg.maxElu)) shedding = true;
    else if (shedding && p99Ms < cfg.stopP99Ms && elu < cfg.maxElu) shedding = false;   // hysteresis
    if (!shedding) return next();
    res.status(503).set("Retry-After", "2").type("application/problem+json").json({
      type: "about:blank", title: "Service Unavailable", status: 503,
      code: "overloaded", request_id: req.get("x-request-id"),
    });
  };
}
```

### Break — four experiments

Run every server with `--import ./tools/loop-meter.ts`.

1. **The giant body.** Add `POST /import` that does `JSON.parse` on a raw ~20 MB body (raise the body limit to allow it), plus a trivial `GET /ping`. Run `npx autocannon -c 10 -d 10 http://localhost:3000/ping` while a second autocannon sends the 20 MB body in a loop. Watch the meter's p99 jump to roughly the parse time, and `/ping`'s latency follow it. Then restore a `100kb` limit and see the 413s and a flat meter.
2. **ReDoS.** Time `/^(\w+\s?)*$/.test("a".repeat(n) + "!")` for n = 20, 21, 22, … in fresh processes, stopping once a run takes several seconds. Plot it: each step roughly doubles. Then run the same inputs against `/^\w+(\s\w+)*$/` and against a 100-character length cap. Finally, write down which strings the rewrite treats differently.
3. **`readFileSync` in the hot path.** A handler that reads a template with `readFileSync` on every request. Under autocannon, compare p99 and the meter with the version that reads the file once at startup. On a fast SSD with a warm cache the difference is small, which is the point: the next disk hiccup turns it into a stall of every request.
4. **Partition it.** A `GET /reconcile-preview` handler that runs 50 million iterations of `Math.sqrt` synchronously. Hit `/ping` during it and record the meter's max. Rewrite it with `forEachYielding` (or a time-budgeted loop) and repeat: `/ping` stays fast and loop max drops to around the budget. Record the total job time before and after, and explain why it went up.

### Explain out loud (60–120 seconds)

1. Why a blocked loop shows as 25% CPU and climbing p99, with no errors.
2. What `monitorEventLoopDelay` measures and why you read p99 and max; what ELU measures and why it beats CPU%.
3. ReDoS: why `(\w+\s?)*` is exponential, why `\s+$` is quadratic, and the four defences in order.
4. How to find the blocking frame with `--cpu-prof`.
5. The fix layers: input, data flow, algorithm, scheduling, thread, with one Ledger example each.
6. Load shedding on loop pressure, and the liveness trap.

---

## What's next

You can now keep the loop free and prove it. Next is what happens when something *does* go wrong: the four channels an error can arrive through, the difference between an operational error you handle and a programmer error you crash on, and why the right response to an `uncaughtException` is a clean exit.

Next → **[Lesson 09: Errors: operational vs programmer, and crash-only design](09-errors-and-crashing.md)**
