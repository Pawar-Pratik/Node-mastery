# Lesson 28 — Performance: profiling & how V8 runs your code

> **Why this lesson exists:** performance work without measurement is superstition. Most "optimisations" target code that wasn't slow, while the real cost (a serializer, a log line, a schema rebuilt on every request) sits in plain sight in a flame graph nobody captured. You need to load-test honestly, profile, read a flame graph, and know enough about V8's JIT to recognise the code shapes that defeat it. "How would you find what's slow?" is asked in every senior loop.

**Time:** ~85 minutes · **Prereq:** Lessons 08, 27

---

## 1. The idea in one sentence

> **Measure under realistic load, profile to find where the time actually goes, and fix the widest frame at the right layer; V8 rewards boring, consistently shaped code and punishes surprises with deoptimization.**

---

## 2. Why guessing fails

**Amdahl's law** caps every optimisation by the share of time it touches: overall speedup = 1 / ((1 − p) + p / s), where `p` is the fraction of time in the part you improve and `s` is how much faster it gets. Make a function that takes 5% of request time 10× faster and the request gets about 4.7% faster. Make the 40% frame 2× faster and you win 25%. **Profiling exists to find `p` before you spend effort on `s`.**

Node adds its own arithmetic. I/O wait is free (the loop serves other requests meanwhile), but **CPU time on the main thread is the hard ceiling**: if each request costs 2 ms of JavaScript, one process can never exceed 500 requests/s, however much I/O concurrency you have. CPU per request on the loop is the number that decides how many replicas you pay for.

**Latency and throughput are different goals.** At low load, latency is just service time. As utilisation approaches 100%, requests queue and latency climbs steeply: that bend is **the knee**. For an API, optimise p99 at the load you expect; throughput sets the cost. A change that raises maximum throughput by 10% but makes p99 worse at normal load is usually a regression.

The evidence that measurement beats intuition is Netflix's **"Node.js in Flames"** (2014). Their latency crept up over time, and flame graphs showed the time going into Express's route lookup: a reload bug kept appending routes to the route array, so the linear scan ([L19](../04-networking/19-frameworks-by-mechanism.md)) grew longer. Nobody would have guessed that cause; a flame graph made it obvious.

---

## 3. Measure first: load testing

```bash
npx autocannon -c 50 -d 30 http://localhost:3000/v1/payments/pi_123 \
  -H "authorization: Bearer $LEDGER_TEST_KEY"
```

autocannon prints a latency table (2.5%, 50%, 97.5%, 99%, average, max) and requests per second. One run tells you little. **Find the knee:** repeat at `-c 10, 25, 50, 100, 200` and plot p99 against throughput. Throughput rises and then flattens while p99 starts climbing; the knee is your real capacity, and it's the number to put in the capacity plan. Run the [L08](../02-event-loop/08-blocking-the-loop.md) loop-delay meter alongside: when loop delay rises with load, the bottleneck is CPU on the loop; when it stays flat while latency climbs, look at the pool, the database or a downstream.

Three honesty rules:

- **Benchmark the real path.** A `POST /v1/payments` load test that reuses one `Idempotency-Key` measures the replay path after the first request. Give every request a fresh key (autocannon's programmatic API lets you rewrite each request), or you're measuring the wrong code.
- **Coordinated omission.** autocannon is a closed-loop tool by default: each connection sends its next request only after the previous response. When the server stalls for a second, the tool politely stops sending, so the stall shows up as a handful of slow requests instead of the thousands real users (who don't wait for each other) would have sent during it. **Closed-loop tests under-report tail latency.** Use them to find hot frames and compare before/after; use an open-model, constant-arrival-rate test (k6's `constant-arrival-rate` executor, from [API L26](https://github.com/Pawar-Pratik/API-mastery/blob/main/06-craft/26-testing-apis.md)) for SLO numbers.
- **Same machine, relative numbers.** On a laptop the load generator competes with the server for CPU. That's fine for before/after comparisons, never for absolute capacity claims.

---

## 4. CPU profiling

### `--cpu-prof`: a profile of a run

```bash
node --cpu-prof --cpu-prof-dir=./profiles src/main.ts
# run the load, then stop the server cleanly → profiles/CPU.<date>.<pid>.<…>.cpuprofile
```

V8's sampling profiler records the JavaScript stack about every millisecond (`--cpu-prof-interval`, in microseconds). Overhead is low, and the profile is written **when the process exits cleanly**, so the server needs a SIGINT/SIGTERM handler that ends in `process.exit()` (L24's lifecycle does). Ctrl+C in a Windows terminal delivers SIGINT, so this works natively.

A CPU profile shows **CPU on the JavaScript thread, not waiting**. An `await` on a 300 ms query contributes nothing to it; that's what traces are for ([L29](29-observability.md)). If a profile under load is mostly `(idle)`, CPU isn't your bottleneck, and no amount of code tuning will help.

One bonus from running TypeScript natively: type stripping replaces types with whitespace, so **line and column numbers in profiles match your `.ts` files** with no source maps.

Open the `.cpuprofile` in Chrome DevTools (Performance panel → load profile), in VS Code, or in speedscope.

### `--inspect`: profile a live process

`node --inspect src/main.ts`, then `chrome://inspect` → "Open dedicated DevTools for Node" → record a profile while the load runs. The inspector binds to `127.0.0.1:9229` by default. **Never expose it beyond localhost**: an inspector connection can run arbitrary code in your process.

### Reading a flame graph

```text
                       ┌────────────────┐
                       │ canonicalize   │ ◄── wide with a flat top: SELF time. Look here first
 ┌───────────┐ ┌───────┴────────────────┴──┐ ┌───────────────────────┐
 │ zod parse │ │ hashIdempotencyBody        │ │ JSON.stringify        │ ◄── flat top: self time
 ├───────────┴─┴────────────────────────────┤ ├───────────────────────┤
 │ createPayment                             │ │ res.json              │
 ├───────────────────────────────────────────┴─┴───────────────────────┤
 │ express router and middleware frames                                │
 └──────────────────────────────────────────────────────────────────────┘
   y: stack depth (callers below, callees above) · width: share of samples = time on CPU
   x: NOT time order; identical stacks are merged and siblings are sorted
```

- **Width is total time**: the frame plus everything it called.
- **Self time is the part of a frame's top edge that nothing sits on.** Wide plateaus, frames with a lot of exposed top, are where CPU is actually burned.
- **Total vs self:** `createPayment` is wide but has almost no self time; it's a messenger. `canonicalize` is narrower but all self: that's the target.
- A **flame graph** aggregates and sorts stacks; a **flame chart** (DevTools' timeline view) keeps time order, which is better for finding one long task. The **bottom-up** (heavy) view sorts functions by self time and is the fastest route to the hot function.
- `(garbage collector)` taking a visible share means allocation pressure ([L27](27-memory-and-gc.md)); `(program)` is time in V8 and native code outside your frames.

0x turns a run into an interactive flame graph, Clinic.js bundles several diagnostics (doctor, flame, bubbleprof), and the older `--prof` + `node --prof-process` produces a V8 tick report. They all read the same samples.

### On-demand profiling in production

`--cpu-prof` only writes at exit, which is useless for a pod that's slow *right now*. The `inspector` module can profile the running process from inside:

```ts
import { Session } from "node:inspector/promises";   // the promise API is experimental; node:inspector is the stable callback form
import { writeFile } from "node:fs/promises";
import { setTimeout as delay } from "node:timers/promises";

export async function captureCpuProfile(ms: number, path: string): Promise<void> {
  const session = new Session();
  session.connect();
  try {
    await session.post("Profiler.enable");
    await session.post("Profiler.start");
    await delay(ms);
    const { profile } = await session.post("Profiler.stop");
    await writeFile(path, JSON.stringify(profile));
  } finally {
    session.disconnect();
  }
}
```

Expose it on the same authenticated admin port as `/debug/heap`, one capture at a time, capped at a minute. Sampling has a small but non-zero cost, so profile one instance, not the fleet. A CPU profile contains function names and file paths, not request data, so it's far less sensitive than a heap snapshot.

> **Spring equivalent:** JFR for always-on, low-overhead recording and async-profiler for flame graphs, including its wall-clock mode for I/O-bound code, which is the JVM version of "a CPU profile shows nothing while you wait". See [Spring L36](https://github.com/Pawar-Pratik/SpringBoot/blob/main/09-production/36-performance-and-jvm-tuning.md).

---

## 5. How V8 runs your code

```text
 source ─parse─▶ bytecode ──warm──▶ Sparkplug ──hot + type feedback──▶ Maglev ──very hot──▶ TurboFan
                 Ignition           baseline compiler:                 mid-tier            top-tier
                 interpreter;       bytecode → machine code,           optimizing          optimizing
                 records feedback   no optimization, very fast         compiler, quick     compiler, best
                 in inline caches   to compile                         to compile          code, slow to compile
                    ▲                                                                            │
                    └──────────────── deoptimization: a speculative assumption failed ──────────┘
```

Everything starts in **Ignition**, the interpreter, which runs bytecode and records **type feedback**: which shapes and types each operation has actually seen. Warm functions get compiled by **Sparkplug** straight from bytecode to machine code with no optimization, cheaply. Hot functions go to **Maglev**, a mid-tier optimizing compiler that's quick to compile, and the hottest to **TurboFan**, which produces the best code and takes the longest to compile.

The optimizing tiers **speculate**. If the feedback says `payment.amount_minor` has always been a small integer on objects of one shape, they emit code specialised for exactly that, guarded by cheap checks. When a check fails (a new shape, a string where a number always was, an array index out of bounds) V8 **deoptimizes**: it throws the optimized code away, rebuilds the interpreter frame and carries on in Ignition. It may re-optimize later with broader assumptions; a function that deopts repeatedly can stay slow. `node --trace-opt --trace-deopt` prints these events with a reason, usually a shape or type the code hadn't seen. The output is noisy, so grep for your function.

---

## 6. Hidden classes and inline caches

V8 doesn't store an object as a hash table if it can avoid it. Objects get a **hidden class** (V8 calls it a *Map*) describing which properties exist and at which offsets. Objects built the same way share one; adding a property moves an object to a new hidden class. **Order matters:**

```ts
const a = { id: "pi_1", amount_minor: 4999 };     // hidden class H1: id, amount_minor
const b = { amount_minor: 4999, id: "pi_2" };     // hidden class H2: same keys, different order
```

Each property access site in your code has an **inline cache** (IC) that remembers the hidden classes it has seen and where the property lives in each:

| IC state | Hidden classes seen at that site | Cost |
|---|---|---|
| Monomorphic | 1 | One shape check, then a load at a known offset; the optimizer can inline it |
| Polymorphic | 2–4 | A short chain of shape checks |
| Megamorphic | More than 4 | A generic lookup through a global cache: much slower, and it blocks optimization of the surrounding code |

The way real code goes megamorphic is mundane:

```ts
// ❌ Conditional properties: "payment DTOs" with many different hidden classes
function toDto(p: PaymentRow) {
  const dto: Record<string, unknown> = { id: p.id, amount_minor: p.amount_minor };
  if (p.customer_id) dto.customer = p.customer_id;        // some DTOs gain `customer`…
  if (p.failure_code) dto.failure_code = p.failure_code;  // …some `failure_code`, some both
  return dto;
}

// ✅ Same properties, same order, every time; absent values are null
function toDto(p: PaymentRow) {
  return {
    id: p.id,
    amount_minor: p.amount_minor,
    customer: p.customer_id ?? null,
    failure_code: p.failure_code ?? null,
  };
}
```

The ✅ version is also the better API contract: a stable response shape for clients. The other rules follow from the same mechanism:

- **Initialise every field in the constructor** (class fields do this) and don't bolt properties on later.
- **Don't `delete` properties on hot objects.** It usually drops the object into dictionary mode, a slow hash-table representation. Set the field to `undefined` or build a new object.
- **Don't change a field's type.** A field that holds numbers and occasionally a string forces a more general representation for every object of that shape.
- **Keep arrays homogeneous and hole-free.** Arrays have *elements kinds*: small integers → doubles → anything, each with a "holey" variant. Transitions only go towards the more general kind, never back. Pushing one string into an array of numbers, or `new Array(n)` with holes, makes every later access pay for it.

**Keep this in proportion.** These rules matter in hot loops: validating every request, transforming 4M rows in the export, matching 1M lines in the reconciler. They don't matter in a handler that runs 100 times a second and then waits 5 ms for Postgres. The profile tells you which situation you're in.

---

## 7. JSON costs

`JSON.stringify` is generic: for every object it walks every property, checks types, looks for `toJSON`, and escapes strings, all synchronously on the loop. For a large response it shows up as a wide flat-topped frame, and for a huge one it's the loop blocking from [L08](../02-event-loop/08-blocking-the-loop.md). `JSON.parse` of a large body is the same on the way in.

**Schema-based serialization** (fast-json-stringify, which Fastify uses for response schemas) compiles a serializer function from a JSON Schema ahead of time. It knows the keys and their types, so it skips the discovery work per object, and it emits only the properties in the schema, which doubles as an allowlist against leaking internal fields. It pays off for large or high-volume responses with a known shape; measure it on yours. The cheaper fixes come first: return less (pagination caps, sparse fields), and stream big outputs instead of stringifying them in one call ([L13](../03-data-and-io/13-streams-and-backpressure.md)).

---

## 8. Micro-benchmarks lie

```ts
// ❌ Measures nothing: the result is unused, so the optimizer may delete the work
const start = performance.now();
for (let i = 0; i < 1e7; i++) Math.sqrt(i);
console.log(performance.now() - start);

// ✅ Consume the result (and let a harness handle warm-up and many samples)
let sink = 0;
const t0 = performance.now();
for (let i = 0; i < 1e7; i++) sink += Math.sqrt(i);
console.log(performance.now() - t0, sink);
```

| Trap | What happens |
|---|---|
| Dead-code elimination | Unused results let the optimizer remove the work you think you're timing |
| Constant folding | Inputs known at compile time get computed once |
| JIT warm-up | Early iterations run in the interpreter; short benchmarks time the compiler, not your code |
| Unrealistic shapes | The benchmark is monomorphic; production calls the same function with six shapes |
| GC and noise | A major GC lands in one variant; CPU frequency scaling and other processes skew runs |

Use a harness (tinybench, mitata) that warms up, runs many samples and reports variance, keep your inputs as varied as production's, and **confirm every micro-win with a load test on the real endpoint**. A function made 3× faster that accounts for 2% of request time doesn't matter (§2).

---

## 9. Measuring inside the process

```ts
import { performance, PerformanceObserver } from "node:perf_hooks";

performance.mark("idem:start");
const hash = hashIdempotencyBody(body);
performance.mark("idem:end");
performance.measure("idempotency-hash", "idem:start", "idem:end");

export const validatePaymentTimed = performance.timerify(validatePayment);   // a 'function' entry per call

new PerformanceObserver((list) => {
  for (const e of list.getEntries()) stageMs.observe(e.duration, { stage: e.name });
  performance.clearMarks();
  performance.clearMeasures();
}).observe({ entryTypes: ["measure", "function"] });
```

User-timing marks and measures accumulate in a global buffer until you clear them, so clear them or you've built the leak from [L27](27-memory-and-gc.md). Use this to keep a permanent per-stage histogram on the few stages a profile told you matter, not to time every function.

---

## 10. Ledger: profiling `POST /v1/payments`

The procedure is the lesson; the numbers are yours to measure.

1. **Baseline.** autocannon at a concurrency below the knee for 60 s, with a fresh `Idempotency-Key` per request. Record RPS, p50, p99 and loop-delay p99.
2. **Profile** under the same load (`--cpu-prof`, or `captureCpuProfile` for 30 s).
3. **Read the bottom-up view.** In a payment create, the usual suspects are: zod parsing, canonicalizing and hashing the body for the idempotency key, `JSON.stringify` of the response, pino serializing log objects, HMAC/crypto, and middleware.
4. **Fix the widest frame at the right layer:**

| Hot frame | Tempting wrong fix | Right layer |
|---|---|---|
| A zod schema built inside the handler | Hand-rolled validation | Hoist schemas to module scope: building them per request is the bug |
| `canonicalize` (recursive key sort, many allocations) | Micro-optimise the sort | Canonicalize the small validated DTO, not the raw body; or hash the raw bytes you already capture for HMAC ([L12](../03-data-and-io/12-buffers-and-encodings.md)), a deliberate semantic change to the idempotency contract |
| pino serializing `req.body` on every line | Turn logging off | Log ids, not bodies; child-logger bindings ([L29](29-observability.md)) |
| `JSON.stringify` of a big response | Nothing | A smaller response, or schema-based serialization |
| Synchronous crypto | — | The async API on the thread pool, if it's large ([L07](../02-event-loop/07-the-thread-pool.md)) |

5. **Re-measure with the identical load**, and record it:

```text
                        RPS      p50      p99      loop p99
before                  ____     ____     ____     ____
after (fix: ________)   ____     ____     ____     ____
```

If p99 didn't move, the frame you fixed wasn't on the critical path. Revert it and go back to the profile.

---

## 11. Production rules

| Rule | Why |
|---|---|
| **No optimisation without a profile under realistic load** | Amdahl's law: you must know `p` before you spend effort on `s` |
| **Find the knee, and plan capacity below it** | Past the knee, queueing makes p99 explode |
| **Treat closed-loop latency numbers as relative** | Coordinated omission hides tail latency; use open-model tests for SLOs |
| **Benchmark the real code path** | Reused idempotency keys, warm caches and toy payloads measure something else |
| **Look for wide plateaus (self time), not tall stacks** | Self time is where CPU burns |
| **A mostly idle profile means look elsewhere** | CPU profiles don't show waiting; use traces and pool metrics |
| **Keep object shapes consistent in hot paths** | Megamorphic sites and deopts cost far more than the code suggests |
| **Never `delete` on hot objects; never mix types in a field** | Dictionary mode and generalized fields slow every access |
| **Hoist schemas, regexes and serializers to module scope** | Per-request construction is pure waste on the loop |
| **Inspector bound to localhost only; on-demand profiling behind admin auth** | The inspector is remote code execution |
| **Confirm every micro-benchmark win with a load test** | Micro-benchmarks measure the JIT, not your service |

---

## 12. Interview traps

**Q1. "How do you find what's slow in a Node service?"**
Start from the symptom and a metric: which endpoint, which percentile, and whether loop delay rises with it. If loop delay rises, it's CPU on the loop: reproduce under load, take a CPU profile, and read the bottom-up view for the widest self-time frames. If loop delay is flat, the time is waiting, so look at traces, pool `waiting` counts and downstream latency instead. Then fix at the right layer and re-measure with identical load. The weak answer is "add caching" or "add replicas" before measuring anything.

**Q2. "How do you read a flame graph?"**
Each box is a function, callers below and callees above; width is the share of samples, so time on CPU. The x-axis isn't time: stacks are merged and sorted. Look for wide plateaus, frames with lots of exposed top edge, because that's self time; wide frames with everything stacked on top are just callers. Mention the flame chart (time-ordered) for finding a single long task.
**Follow-up:** "The profile is mostly idle but latency is bad?" Then CPU isn't the bottleneck; the time is spent waiting, which a CPU profile can't see.

**Q3. "What is a hidden class?"**
V8's internal description of an object's layout: which properties it has and at which offsets, shared by objects built the same way, in the same order. Property access sites cache hidden classes in inline caches: one shape is monomorphic and fast, two to four is polymorphic, more than four is megamorphic and falls back to slow generic lookups. That's why consistent object construction matters in hot code.

**Q4. "What causes deoptimization?"**
Optimizing compilers specialise code using type feedback, behind guards. A guard failing (a new hidden class, a string in a number field, an out-of-bounds index, a property deleted) makes V8 discard the optimized code and fall back to the interpreter. Repeated deopts can leave a hot function slow. `--trace-deopt` shows the function and the reason.

**Q5. "Why are micro-benchmarks misleading?"**
Dead-code elimination and constant folding can delete the work, JIT warm-up means short runs time the compiler, benchmarks feed one monomorphic shape where production feeds several, and GC or CPU-frequency noise lands unevenly. Use a harness with warm-up and many samples, consume results, and confirm with a load test on the real endpoint, because Amdahl's law decides whether the win matters at all.

**Q6. "Latency or throughput: which do you optimize?"**
For a user-facing API, tail latency at the expected load, because that's what users feel; throughput determines cost. They trade off around the knee: pushing utilisation up raises throughput and p99 together. In Node, CPU per request on the main thread caps one process's throughput, so reducing it improves both.

**Q7. "What is coordinated omission?"**
A closed-loop load generator waits for each response before sending the next request, so when the server stalls, the generator stops sending and records a few slow requests instead of the many that real independent users would have sent during the stall. Tail latency is under-reported, sometimes massively. Fix it with open-model, constant-arrival-rate tests.

**Q8. "Why can Fastify serialize responses faster than `JSON.stringify`?"**
`JSON.stringify` discovers each object's keys and types at runtime; fast-json-stringify compiles a function from the response schema, so the structure is known in advance and it emits only schema properties. That's less work per object, plus an allowlist against leaking fields. It only matters when serialization is a meaningful share of the profile.

**Q9. "Is it safe to profile in production?"**
CPU profiling, yes, with care: sampling overhead is small, so capture on one instance for a bounded time through an authenticated admin endpoint using the `inspector` module. Never expose `--inspect` beyond localhost, because the inspector protocol can execute code. Heap snapshots are the riskier tool ([L27](27-memory-and-gc.md)).

**Q10. "What does Amdahl's law tell you about optimization?"**
That the overall gain is capped by the fraction of time the optimized part takes: making a 5% frame infinitely fast gains at most about 5%. So the first job is finding the biggest `p` with a profile, and the second is checking the end-to-end number moved.

---

## 13. Build & break

### Build — `bench/` and a profiling script
- `bench/payments-get.sh` and `bench/payments-create.ts`: autocannon runs with fixed duration and concurrency; the create script uses autocannon's programmatic API to put a fresh `Idempotency-Key` on every request.
- `bench/knee.sh`: loops concurrency over 10, 25, 50, 100, 200 and appends RPS and p99 to a CSV you can plot.
- `bench/profile.sh`: starts the server, waits for `/healthz`, starts the load in the background, calls the authenticated `POST /debug/cpu-profile?ms=20000` endpoint (built from §4's `captureCpuProfile`), waits for the load to end and prints the profile path. Using the endpoint instead of signals makes it work in Git Bash on Windows; in WSL2 you can use `--cpu-prof` and `kill -INT` instead.
- Run §10 end to end and keep the filled-in before/after table.

### Break — 4 experiments
1. **Megamorphic.** Write `function total(xs: { amount_minor: number }[]) { let s = 0; for (const x of xs) s += x.amount_minor; return s; }`. Feed it 1M objects of one shape, then 1M objects spread over six shapes (same `amount_minor`, but different extra properties or key orders). Warm up both, run each many times, print the sums, and compare timings. Expect the six-shape version to be clearly slower; record the ratio on your machine. Add `--trace-deopt` and grep for `total`.
2. **`delete` in a hot loop.** In a plain `.mjs` file (the `%` natives aren't valid TypeScript), run with `node --allow-natives-syntax`: create an object, print `%HasFastProperties(o)`, `delete` a property, and print it again. Then benchmark a loop that reads fields from objects with a deleted property against objects where the field was set to `undefined`.
3. **The benchmark the JIT deleted.** Time the ❌ `Math.sqrt` loop from §8 at `1e7` and `1e8` iterations. If the time barely changes with 10× the work, the work isn't happening. Add the `sink` and watch the time scale with `n`.
4. **The `JSON.stringify` plateau.** Add a debug route that loads 50,000 payments and returns them with `res.json`. Profile it under `npx autocannon -c 10`: a wide flat-topped `JSON.stringify` frame dominates and loop-delay p99 jumps. Cap the page size (or stream it, L13), profile again, and confirm the plateau is gone.

### Explain out loud (60–120 seconds)
1. Amdahl's law, and why CPU per request on the loop caps a Node process.
2. Finding the knee, and why closed-loop tests under-report tail latency.
3. How to read a flame graph: width, self time, plateaus, and what an idle profile means.
4. Ignition → Sparkplug → Maglev → TurboFan, speculation and deoptimization.
5. Hidden classes and inline caches: monomorphic to megamorphic, and the code that causes it.

---

## What's next

You can now measure memory and CPU on demand. Production needs them all the time, cheaply, correlated by request: logs that don't block the loop, context that survives every `await`, runtime metrics like loop delay and GC pauses, and tracing that actually loads before your app does. That's the last lesson of Module 7.

Next → **[Lesson 29: Observability from the runtime](29-observability.md)**
