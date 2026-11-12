# Node.js: Zero → Absolute Authority

> The goal is not "I can build an Express server."
> The goal is: **you know what the event loop was doing when p99 spiked, why the process used 3 GB when the heap was 400 MB, and why the deploy dropped requests** — and you can fix each at the right layer instead of adding replicas and hoping.

**Part of [The Legend Track](https://github.com/Pawar-Pratik/The-Legend-Track)** — the five-track path.
Sibling tracks: [APIs & REST](https://github.com/Pawar-Pratik/API-mastery) · [TypeScript](https://github.com/Pawar-Pratik/TypeScript-mastery) · [React](https://github.com/Pawar-Pratik/React-mastery) · [Spring Boot](https://github.com/Pawar-Pratik/SpringBoot)

---

## Why most Node knowledge is shallow

Node is easy to start and hard to run. Most engineers learn it as "JavaScript on the server": install Express, write handlers, `await` everything, ship. It works on a laptop, and then production asks questions the tutorials never did:

- *"Why did every request get slow at once, while CPU sat at 30%?"* (one synchronous call blocked the only JavaScript thread)
- *"Why did file reads slow down when DNS got slow?"* (they share a 4-thread pool)
- *"Why did the export endpoint take the pod down?"* (it buffered 4 million rows instead of streaming them)
- *"Why does the process die with no log line?"* (an unhandled rejection, or the kernel's OOM killer)
- *"Why do we get 502s from the load balancer, but only on quiet endpoints?"* (keep-alive timeouts in the wrong order)
- *"Why does every deploy drop a few requests?"* (SIGTERM went to the wrong process, or nobody drained)
- *"Why does memory grow 50 MB an hour?"* (something reachable that should have been garbage)

Every one has a precise, mechanical answer. This course teaches the mechanisms, so Node stops surprising you.

### The five ideas everything else derives from

1. **One JavaScript thread, many runtime threads.** Your code never runs in parallel with itself. I/O does, in the kernel and in libuv's thread pool. Saying "Node is single-threaded" without that distinction is the first wrong answer.
2. **The event loop runs callbacks to completion, and never interrupts one.** Anything that doesn't yield blocks *every* request in the process. Throughput in Node is a function of how quickly each callback gives the thread back.
3. **Data moves as streams and events, and backpressure is a contract.** A fast producer and a slow consumer will exhaust memory unless someone honours `write()` returning `false`.
4. **The process is the unit of failure.** After a programmer error the process's state is unknown: crash, let the supervisor restart it, and shut down gracefully when asked. Never limp on.
5. **Memory is managed, not unlimited.** Anything still reachable is kept forever, and the heap is only part of what the kernel counts.

Internalise those five and Node becomes predictable.

---

## What you'll be able to do at the end

- Predict the output of any `nextTick` / promise / `setImmediate` / timer puzzle, and explain it phase by phase
- Prove the event loop is blocked from metrics, find the blocking frame in a CPU profile, and fix it at the right layer
- Stream 4 million rows to a client in constant memory, and cancel the database query when the client disconnects
- Choose between `worker_threads`, `child_process`, `cluster` and more replicas, with reasons
- Write a graceful shutdown that loses zero requests during a Kubernetes rollout, including the PID 1 trap
- Find a memory leak with heap snapshots, and explain why a pod was OOMKilled when the heap looked fine
- Size a Postgres pool, the libuv thread pool and a worker pool against each other, with arithmetic
- Publish a typed ESM package with a correct `exports` map, and defend your supply-chain posture after an npm compromise
- Answer "is Node single-threaded?" in a way that ends the follow-up questions instead of inviting them

---

## Coming from Spring

You already know how a server runtime works. Here's where Node agrees and where it doesn't:

| Spring / JVM | Node | The difference that matters |
|---|---|---|
| Tomcat thread pool, one thread per request | One event loop + libuv thread pool | Blocking a thread costs one request in Tomcat, and every request in Node ([L05](02-event-loop/05-the-event-loop.md)) |
| Virtual threads (Java 21) | async/await on the event loop | Same economics, different syntax. Pinning in Loom ≈ blocking in Node ([L01](01-foundations/01-what-node-really-is.md)) |
| `ThreadLocal` / MDC | `AsyncLocalStorage` | Context follows the async call chain, not the thread ([L29](07-production/29-observability.md)) |
| HikariCP | `pg.Pool` | Same pool-sizing maths. Different leak shape: a client never released ([L25](06-data-and-jobs/25-databases-from-node.md)) |
| `@Transactional` | An explicit client + `BEGIN`/`COMMIT`, often carried by `AsyncLocalStorage` | No proxy magic, so no self-invocation trap, and no safety net either ([L25](06-data-and-jobs/25-databases-from-node.md)) |
| `@Async`, `ExecutorService` | Worker threads for CPU work, queues for deferred work | Workers share nothing by default ([L21](05-concurrency/21-worker-threads.md)) |
| JVM heap, G1/ZGC, `-Xmx` | V8 heap, Orinoco GC, `--max-old-space-size` | RSS includes memory outside the heap: Buffers and native code ([L27](07-production/27-memory-and-gc.md)) |
| Spring Boot Actuator | Health endpoints + `perf_hooks` runtime metrics | Event-loop delay is the Node-specific vital sign ([L29](07-production/29-observability.md)) |
| Maven / Gradle, Maven Central | npm / pnpm, the npm registry | Installs run third-party scripts by default ([L30](08-shipping/30-security-and-supply-chain.md)) |
| NestJS feels like… | …Spring, deliberately | DI and decorators, at a cost you should be able to name ([L19](04-networking/19-frameworks-by-mechanism.md)) |

> **Spring equivalent** callouts throughout the track point at the matching Spring Boot lesson. The same idea in two runtimes is what makes you framework-independent, and that's what senior interviews probe.

---

## The project: **Ledger Engine**

The [API track](https://github.com/Pawar-Pratik/API-mastery) designed Ledger's contract and the [React track](https://github.com/Pawar-Pratik/React-mastery) built the Console on top of it. This track builds **the Node processes that run Ledger in production**, plus the two artefacts every payments platform ships to developers:

- **The API process**, understood below Express: keep-alive tuned to the load balancer, request deadlines, load shedding on event-loop lag, and a graceful drain
- **The webhook dispatcher**: a Postgres `SKIP LOCKED` queue with per-endpoint and global concurrency limits, pooled `undici` clients, an SSRF guard, and 8 attempts over ~3 days
- **The statement exporter**: 4M payments → CSV → gzip → HTTP response in constant memory, cancelled the moment the client disconnects
- **The payout-file ingester**: a 2 GB CSV upload parsed, validated and batch-inserted with backpressure end to end
- **The reconciler**: CPU-heavy settlement matching on a worker-thread pool, so the API never feels it
- **The live feed**: an SSE endpoint fed by Postgres `LISTEN/NOTIFY`, the server half of the Console's live payments ([React L19](https://github.com/Pawar-Pratik/React-mastery/blob/main/05-data/19-realtime.md))
- **`@ledger/node` and `ledger`**: a published, typed, ESM SDK (auto-pagination with async iterators, retries with idempotency keys, webhook verification) and a CLI with `ledger listen --forward-to`, like Stripe's

**Why this and not a chat app:** a payments engine makes every runtime concern real. Constant memory matters because the export is 4 million rows. Backpressure matters because the upload is 2 GB. Graceful shutdown matters because a dropped request might be a payment. Worker threads matter because reconciliation is CPU-bound and the API can't stall. You can't fake your way through any of it.

Blueprint: **[Ledger Engine blueprint](09-project/34-ledger-engine-blueprint.md)** · Build log: **[Ledger Engine build log](09-project/35-ledger-engine-build-log.md)**

---

## Setup

| Tool | Notes |
|---|---|
| **Node 24 LTS or newer** | Everything here targets Node 24+. Node 22 is in maintenance until April 2027; Node 20 reached end-of-life in April 2026. Node 26 becomes LTS in late October 2026, and from Node 27 there's one major release a year and every release becomes LTS |
| **A version manager** | `fnm` (works on Windows) or `nvm-windows`, plus an `.nvmrc` per project. Never trust whatever `node` happens to be on the PATH |
| **TypeScript 5.8+** | Node runs the `.ts` files directly (type stripping is stable since Node 24.12). `tsc` only *checks*: `npx tsc --noEmit`. The [TypeScript track](https://github.com/Pawar-Pratik/TypeScript-mastery) is a prerequisite |
| **Chrome DevTools** | `node --inspect` → `chrome://inspect`. The CPU profiler and heap snapshots are the primary learning tools from Module 2 on |
| **Docker Desktop** | Postgres 16 + Redis 7 from Module 4 on: the same Compose file as the API track |
| **`autocannon`** | `npx autocannon` for load. You'll measure before and after every fix |
| **WSL2** *(recommended)* | Lessons 22–24 send real POSIX signals. Windows has no `SIGTERM` in the POSIX sense, and each lesson says where Windows differs |

The track's `tsconfig.json`. Every line is explained in [Lesson 03](01-foundations/03-the-process.md):

```jsonc
{
  "compilerOptions": {
    "target": "esnext",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "noEmit": true,                       // Node runs the .ts; tsc only checks
    "allowImportingTsExtensions": true,   // import "./db.ts", exactly what Node loads
    "erasableSyntaxOnly": true,           // no enums, no parameter properties: Node can't strip those
    "verbatimModuleSyntax": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "types": ["node"],
    "skipLibCheck": true
  }
}
```

> **The habit that matters most: watch the loop.** Every experiment in this track runs with a ten-line event-loop delay meter printing p50/p99 (built in [Lesson 08](02-event-loop/08-blocking-the-loop.md)). People who become fluent in Node look at loop delay and heap graphs constantly. People who stay stuck reason about them in their heads.

---

## Table of contents

### Module 1 — Foundations: what Node actually is
> Don't skip this because you can already write JavaScript. Nearly every production surprise in Node comes from a gap in this module.

| # | Lesson | What you'll be able to do |
|---|---|---|
| 01 | [What Node really is](01-foundations/01-what-node-really-is.md) | Explain V8, libuv and the bindings; answer "is Node single-threaded?" precisely; know when Node is the wrong tool |
| 02 | [The JavaScript Node assumes](01-foundations/02-the-javascript-node-assumes.md) | Closures, `this`, prototypes, and why `pg` returns your `bigint` money column as a string |
| 03 | [The process: your program's contract with the OS](01-foundations/03-the-process.md) | Explain why a script won't exit; handle argv, env, stdio and exit codes correctly; run TypeScript natively |
| 04 | [Modules: CommonJS vs ESM, properly](01-foundations/04-modules-cjs-and-esm.md) | Explain the loading algorithms, live bindings, `require(esm)`, and the dual-package hazard |

### Module 2 — The event loop & async
> **The heart of the track. Go slowest here.** Every later lesson is an application of these six.

| # | Lesson | What you'll be able to do |
|---|---|---|
| 05 | [The event loop, phase by phase](02-event-loop/05-the-event-loop.md) | **The most important lesson.** Predict execution order from first principles; explain `setImmediate` vs `setTimeout(0)` |
| 06 | [Promises, async/await & the microtask queue](02-event-loop/06-promises-and-microtasks.md) | Explain what `await` compiles to, when rejections crash the process, and microtask starvation |
| 07 | [The thread pool: what actually runs in parallel](02-event-loop/07-the-thread-pool.md) | Name every API that uses the libuv pool; diagnose the DNS-slows-file-reads incident |
| 08 | [Blocking the loop — and measuring it](02-event-loop/08-blocking-the-loop.md) | Measure loop delay and utilisation; find and fix synchronous work, giant JSON and ReDoS |
| 09 | [Errors: operational vs programmer, and crash-only design](02-event-loop/09-errors-and-crashing.md) | Handle each failure channel correctly; explain why you exit after `uncaughtException` |
| 10 | [Cancellation, timeouts & bounded concurrency](02-event-loop/10-cancellation-and-concurrency.md) | Thread `AbortSignal` through everything; stop `Promise.all` from DoS-ing your own dependencies |

### Module 3 — Data & I/O
| # | Lesson | What you'll be able to do |
|---|---|---|
| 11 | [EventEmitter: the pattern under everything](03-data-and-io/11-eventemitter.md) | Explain synchronous emit, the `'error'` crash rule, and listener leaks |
| 12 | [Buffers, bytes & encodings](03-data-and-io/12-buffers-and-encodings.md) | Handle binary data, multi-byte splits and raw bodies for HMAC; avoid the pooled-slice leak |
| 13 | [Streams & backpressure](03-data-and-io/13-streams-and-backpressure.md) | **The second most important lesson.** Move unbounded data in bounded memory; explain why `pipeline` beats `pipe` |
| 14 | [Composing streams: Transform, generators & Web Streams](03-data-and-io/14-composing-streams.md) | Build parsing and batching pipelines with async generators; bridge Node and Web streams |
| 15 | [The file system, honestly](03-data-and-io/15-the-file-system.md) | Write files atomically and durably; prevent path traversal; survive `EMFILE` and Windows |

### Module 4 — Networking & HTTP from the metal
| # | Lesson | What you'll be able to do |
|---|---|---|
| 16 | [TCP & the `net` module](04-networking/16-tcp-and-net.md) | Frame messages on a byte stream; speak Redis's protocol over a raw socket |
| 17 | [`node:http` with no framework](04-networking/17-http-from-scratch.md) | Build a server with body limits and correct timeouts; fix the load-balancer 502 |
| 18 | [Outbound HTTP: fetch, undici & connection pools](04-networking/18-outbound-http.md) | Pool and time-bound outbound calls; avoid the unconsumed-body leak; block SSRF |
| 19 | [Frameworks by mechanism: Express, Fastify, NestJS](04-networking/19-frameworks-by-mechanism.md) | Explain what each framework does per request, and choose by constraint |
| 20 | [Real-time servers: SSE & WebSockets](04-networking/20-realtime-servers.md) | Serve thousands of live connections with heartbeats, backpressure and cross-instance fan-out |

### Module 5 — Concurrency & scaling out
| # | Lesson | What you'll be able to do |
|---|---|---|
| 21 | [Worker threads](05-concurrency/21-worker-threads.md) | Move CPU work off the loop with a pool; use transfer and `SharedArrayBuffer` correctly |
| 22 | [Child processes](05-concurrency/22-child-processes.md) | Choose `spawn`/`exec`/`execFile`/`fork`; never ship a shell injection |
| 23 | [Scaling out: cluster, containers & cores](05-concurrency/23-cluster-and-scaling.md) | Decide cluster vs replicas; size processes against container CPU and memory limits |
| 24 | [Graceful shutdown & the process lifecycle](05-concurrency/24-graceful-shutdown.md) | Lose zero requests in a rolling deploy; explain the PID 1 signal trap |

### Module 6 — Data & background work
| # | Lesson | What you'll be able to do |
|---|---|---|
| 25 | [Databases from Node: pools, transactions & context](06-data-and-jobs/25-databases-from-node.md) | Size pools with arithmetic; never leak a client; carry tenant and transaction with `AsyncLocalStorage` |
| 26 | [Background jobs & queues](06-data-and-jobs/26-background-jobs-and-queues.md) | Build an at-least-once job system on Postgres `SKIP LOCKED`; know when to reach for BullMQ or SQS |

### Module 7 — Production internals
| # | Lesson | What you'll be able to do |
|---|---|---|
| 27 | [Memory & garbage collection](07-production/27-memory-and-gc.md) | Read `memoryUsage()`; explain heap OOM vs OOMKilled; find a leak with three heap snapshots |
| 28 | [Performance: profiling & how V8 runs your code](07-production/28-performance-and-v8.md) | Read a flame graph; explain hidden classes, inline caches and deopts; benchmark honestly |
| 29 | [Observability from the runtime](07-production/29-observability.md) | Fast structured logs, `AsyncLocalStorage` context, runtime metrics, OpenTelemetry without guesswork |

### Module 8 — Shipping safely
| # | Lesson | What you'll be able to do |
|---|---|---|
| 30 | [Security & the supply chain](08-shipping/30-security-and-supply-chain.md) | Defend against compromised packages, prototype pollution and injection; use the permission model honestly |
| 31 | [Testing Node](08-shipping/31-testing-node.md) | Test async, evented, streaming and time-dependent code with `node:test`, without sleeps |
| 32 | [Packages: exports, publishing & monorepos](08-shipping/32-packaging-and-publishing.md) | Publish a typed ESM package with a correct `exports` map; run a workspace monorepo |
| 33 | [Shipping Node: containers, config & runtime flags](08-shipping/33-shipping-node.md) | Write a production Dockerfile; set memory flags against limits; know what serverless changes |

### Module 9 — The project
| # | Lesson | What you'll be able to do |
|---|---|---|
| 34 | [Ledger Engine blueprint](09-project/34-ledger-engine-blueprint.md) | The full system: processes, budgets, data flows, decisions and their justifications |
| 35 | [Ledger Engine build log](09-project/35-ledger-engine-build-log.md) | Build it in phases, each mapped to lessons, each ending in an incident drill |

### Module 10 — Interview mastery
| # | Lesson | What you'll be able to do |
|---|---|---|
| 36 | [Rapid-fire master Q&A](10-interview/36-rapid-fire-master-qa.md) | Revise the whole track; drill until answers are reflexes |
| 37 | [Output prediction & build-from-scratch challenges](10-interview/37-output-and-implementation-challenges.md) | Solve "what does this print?" and "implement `promisify` / `EventEmitter` / `p-limit`" live |
| 38 | [Production scenarios & Node system design](10-interview/38-scenarios-and-system-design.md) | Handle 3am scenarios with a repeatable method; design Node services with capacity maths |

---

## The 20 questions that define mastery

Self-test now, note your score, retest at the end.

1. Is Node single-threaded? Answer precisely enough that there's no follow-up.
2. Name the event-loop phases in order. Where do `process.nextTick` and promise callbacks run?
3. `setTimeout(fn, 0)` vs `setImmediate(fn)`: which runs first? Answer for the main module and for inside an `fs` callback.
4. Which Node APIs use the libuv thread pool, and what goes wrong when it saturates?
5. p99 latency tripled but CPU is at 30%. How do you prove the event loop is blocked, and find what's blocking it?
6. What is backpressure, and what happens when you ignore `write()` returning `false`?
7. Why `pipeline()` instead of `.pipe()`?
8. What does Node do with an unhandled promise rejection, and why is that the right default?
9. Operational vs programmer errors: what do you do with each? Why exit after `uncaughtException`?
10. CommonJS vs ESM: three real differences. Can `require()` load an ES module today?
11. `worker_threads` vs `child_process` vs `cluster` vs more replicas: when do you use each?
12. Walk me through a graceful shutdown in Kubernetes, step by step, including the PID 1 problem.
13. Intermittent 502s from the load balancer, only on idle connections. Cause and fix.
14. Memory grows 50 MB an hour. How do you find the leak?
15. The pod was OOMKilled (exit 137) but there was no "heap out of memory" error. Why?
16. Why does `pool.query("BEGIN")` … `pool.query("COMMIT")` corrupt your transactions?
17. How does `AsyncLocalStorage` work, and where does context get lost?
18. Stream 4M rows as CSV to a client in constant memory, and stop the query if they disconnect. How?
19. A dependency was compromised on npm overnight. What protected you, and what would have?
20. `exec(\`convert ${file}\`)` with user input: what's the attack, and what's the fix?

Every one is answered in the track, with the lesson named in [Lesson 36](10-interview/36-rapid-fire-master-qa.md).

---

## Pacing

| Intensity | Duration |
|---|---|
| **Sprint** (4 h/day) | ~4 weeks |
| **Serious** (2 h/day) | ~8 weeks — the sweet spot |
| **Steady** (1 h/day) | ~16 weeks (keep Ledger Engine moving weekly or you'll lose context) |

Spend the most time on **Module 2** and **Lesson 13**. The event loop and backpressure are where every later problem comes from, and they're the two things most Node developers never learn properly.

---

## One promise

By the end, Node will have no magic in it. When a process is slow, leaking, crashing or dropping requests, you'll know which of the five ideas it's violating, which metric proves it, and which line to change.

Start here → **[Lesson 01: What Node really is](01-foundations/01-what-node-really-is.md)**
