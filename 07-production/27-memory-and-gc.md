# Lesson 27 — Memory & garbage collection

> **Why this lesson exists:** memory incidents in Node come in two shapes that look nothing alike: a loud `JavaScript heap out of memory` crash, and a silent OOMKill (exit 137) while the heap graph looks healthy. Leaks hide in closures, caches, listeners and timers, and they show up as latency long before they show up as a crash. Explaining both shapes, and finding a leak with heap snapshots instead of guessing, is a core senior skill and a staple interview question.

**Time:** ~90 minutes · **Prereq:** Lessons 12, 13

---

## 1. The idea in one sentence

> **The garbage collector frees only what's unreachable, and the kernel counts more than the heap: a leak is something still reachable that shouldn't be, and an OOMKill is RSS crossing a limit the heap never knew about.**

---

## 2. Two incidents that look nothing alike

**Incident A.** Latency climbs for twenty minutes, CPU sits near 100%, and then stderr prints:

```text
<--- Last few GCs --->
… Mark-Compact … (repeated, each freeing almost nothing)
FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory
```

**Incident B.** The pod restarts. Your logs end mid-sentence. The heap graph shows 300 MB of a 1 GB limit. `kubectl describe pod` says `Reason: OOMKilled` and `Exit Code: 137`.

| | Heap OOM (A) | Kernel OOMKill (B) |
|---|---|---|
| Who kills the process | V8: it aborts itself | The kernel's cgroup OOM killer, with SIGKILL |
| Trigger | Old space can't grow past V8's heap limit | The container's memory (RSS) crosses its cgroup limit |
| Evidence | `FATAL ERROR` and the last GCs on stderr; exit typically 134 (SIGABRT) | Nothing in your logs; exit 137 (128 + 9); `OOMKilled` in the pod status |
| Usual cause | JavaScript objects: a leak, or one enormous parse | Off-heap memory (Buffers, native libraries), several processes per container, or a heap limit set above the container limit |
| Warning signs | GC thrash: latency and CPU up, troughs in `heapUsed` rising | Often none: RSS climbs while the heap looks flat |

Knowing which one you have tells you where to look. The rest of this lesson is the map.

---

## 3. The V8 heap

```text
 ┌──────────────────────────────── V8 heap (one per isolate) ──────────────────────────────┐
 │  New space (young generation)          Old space (old generation)      Large-object     │
 │  ┌──────────┐  ┌──────────┐            ┌─────────────────────────┐     space            │
 │  │ semi A   │  │ semi B   │──promote──▶│ objects that survived    │     ┌────────────┐  │
 │  │ allocate │  │ (empty)  │            │ mark → sweep → compact   │     │ huge arrays │  │
 │  └──────────┘  └──────────┘            └─────────────────────────┘     │ and strings │  │
 │  Scavenger: copy the live objects into the other semi-space              └────────────┘  │
 │  Code space: JIT-compiled machine code                                                   │
 └──────────────────────────────────────────────────────────────────────────────────────────┘
  Outside the heap, but inside RSS: Buffers and ArrayBuffers, native allocations (zlib, TLS,
  crypto), thread stacks, the node binary and shared libraries, allocator fragmentation
```

**The generational hypothesis: most objects die young.** In a server that's literal: request objects, parsed bodies and DTOs live for milliseconds. So V8 splits the heap by age.

**New space** is small. Allocation is a pointer bump, so it's very cheap. When it fills, a **Scavenge** copies the *live* objects into the other semi-space and reclaims everything else at once. Its cost is proportional to what survives, not to the garbage, which is why scavenges are frequent and short. Objects that survive a couple of scavenges are **promoted** to old space.

**Old space** is collected by **Mark-Compact**: mark everything reachable from the roots, sweep the dead, compact fragmented pages. **Orinoco** is V8's project that made this parallel (helper threads), incremental (marking in small slices between JavaScript tasks) and concurrent (marking and sweeping on background threads while your code runs). The pauses are now far shorter than the total GC work, but **never zero**: there's still a stop-the-world finish, and every pause stops the event loop.

**Large-object space** holds objects too big for a regular page; they're never moved. **Code space** holds JIT output ([L28](28-performance-and-v8.md)). `v8.getHeapSpaceStatistics()` lists the spaces by name with their sizes.

### What "reachable" means

The roots are the active stack, globals, **module-scope variables** (a module lives for the life of the process), and **active handles**: a live timer, socket or server, and everything their callbacks close over. If a chain of references leads from a root to an object, the object stays, forever. **A leak in a garbage-collected language is always a reference you forgot you were holding.**

---

## 4. Reading `process.memoryUsage()`

```ts
const m = process.memoryUsage();    // all values in bytes
// { rss, heapTotal, heapUsed, external, arrayBuffers }
```

| Field | What it is | Moves when |
|---|---|---|
| `rss` | Resident set size: physical memory the process occupies. **What the kernel and your container limit count** | Anything grows |
| `heapTotal` | Memory V8 has reserved for its heap | The heap grows or shrinks |
| `heapUsed` | Memory occupied by JavaScript objects, live and not-yet-collected | Up with allocation, down after each GC |
| `external` | Memory of C++ objects tied to JavaScript objects, **including** `arrayBuffers` | Buffers, native resources |
| `arrayBuffers` | ArrayBuffers and SharedArrayBuffers, including every Node `Buffer` | Buffers ([L12](../03-data-and-io/12-buffers-and-encodings.md)) |

**RSS is bigger than the heap, by design.** On top of `heapTotal` it contains Buffers, native allocations (zlib contexts, TLS sessions), the stacks of the main thread, the libuv pool and V8's platform threads, the mapped node binary, and memory the allocator has freed but not returned to the OS. A service that streams files or compresses responses can sit at a 200 MB heap and a 900 MB RSS.

`v8.getHeapStatistics().heap_size_limit` tells you the limit V8 is actually using. **Graph `heapUsed` against it**, not against RSS.

The shape of `heapUsed` over time is the diagnosis:

```text
healthy: the troughs (after each major GC) stay flat
  │     /|    /|    /|    /|    /|
  │    / |   / |   / |   / |   / |
  │   /  |  /  |  /  |  /  |  /  |
  └──────────────────────────────────── time

leak: the troughs rise; each GC frees less than the last cycle allocated
  │                          /|  /|
  │                /|  /|  / |/ |
  │      /|  /|  / |/ |/
  │  /| / |/ |/
  └──────────────────────────────────── time
```

---

## 5. The heap limit, and setting it in a container

V8's default old-space limit is a heuristic derived from the memory it thinks the machine has. Whether that heuristic sees your **container's** cgroup limit, rather than the host's RAM, depends on the Node and libuv version, so don't rely on it: **set `--max-old-space-size` (in MiB) explicitly, to about 75% of the container limit.**

```yaml
# 1 GiB container → 768 MiB of old space; the other 25% is for new space, Buffers,
# native memory, stacks and code
resources:
  limits: { memory: "1Gi" }
env:
  - name: NODE_OPTIONS
    value: "--max-old-space-size=768"
```

The direction of the mistake decides which incident you get:

- **Limit at or above the container limit:** the heap is allowed to grow into memory the container doesn't have. The kernel kills the process before V8 ever runs a last-ditch GC or prints anything. **You've converted a debuggable heap OOM into a silent exit 137.**
- **Limit at ~75%:** a leak ends in a heap OOM with a message, and you can arm `--heapsnapshot-near-heap-limit` to capture evidence.
- **Limit far too low:** the service spends its life in major GCs (thrash) under normal load.

It's per heap, so multiply. Every `cluster` worker is a separate process with its own heap ([L23](../05-concurrency/23-cluster-and-scaling.md)), and every worker thread is a separate isolate with its own heap, capped through `resourceLimits` ([L21](../05-concurrency/21-worker-threads.md)). Four workers × 768 MiB in a 1 GiB container is the OOMKill from incident B.

> **Spring equivalent:** `-Xmx` vs the container limit, and the same rule: never set the heap equal to the limit, because metaspace, thread stacks and direct buffers live outside it. Java has `-XX:MaxRAMPercentage`; in Node, derive `--max-old-space-size` from the limit in the same deployment template so the two change together. See [Spring L36](https://github.com/Pawar-Pratik/SpringBoot/blob/main/09-production/36-performance-and-jvm-tuning.md).

---

## 6. GC pauses and tail latency

Every GC pause is time the event loop isn't running, so GC shows up in p99, not p50. Two ways to see it:

```bash
node --trace-gc src/main.ts      # one line per GC: Scavenge or Mark-Compact, heap before → after, pause ms
```

```ts
import { PerformanceObserver, constants } from "node:perf_hooks";

const gcObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    const kind = (entry.detail as { kind?: number } | undefined)?.kind;
    const major = kind === constants.NODE_PERFORMANCE_GC_MAJOR;
    gcPauseMs.observe(entry.duration, { kind: major ? "major" : "minor" });   // a histogram (L29)
  }
});
gcObserver.observe({ entryTypes: ["gc"] });
```

**GC frequency follows your allocation rate.** A handler that builds large intermediate arrays scavenges constantly. One big transient allocation, like a 20 MB `JSON.parse`, fills new space so fast that short-lived objects get promoted before they die, and the old space then needs extra major GCs to clean up garbage that should never have reached it. Allocation-heavy services sometimes raise `--max-semi-space-size` (MiB) to make new space bigger: fewer scavenges and fewer premature promotions, at the cost of memory. Measure with `--trace-gc` before and after; don't cargo-cult it.

**Near the limit, GC becomes the workload.** Each mark-compact frees almost nothing, the next one starts immediately, and the loop delay ([L08](../02-event-loop/08-blocking-the-loop.md)) explodes. This phase can last many minutes before the fatal error. **A leak usually appears as a latency incident first**, which is why heap-used-vs-limit belongs on the same dashboard as loop delay.

---

## 7. Leak patterns

| Pattern | What holds the reference | Fix |
|---|---|---|
| **Unbounded `Map` cache** | A module-scope Map that only grows | LRU with `max` (the memory bound) and TTL (freshness) |
| **Closure capturing a big object** | A long-lived callback's context | Copy what you need; don't capture the big thing |
| **Listener never removed** | The emitter's listener array | `once`, or remove on `close`; `MaxListenersExceededWarning` is the smoke alarm ([L11](../03-data-and-io/11-eventemitter.md)) |
| **Timer never cleared** | An active timer is a root | `clearInterval` on every exit path |
| **Requests held globally** | A "recent requests" debug array | Store ids, bounded |
| **Promises that never settle, held in a map** | A pending-reply map whose replies never arrive | Timeouts that delete the entry |
| **Pooled-buffer slice retention** | A small `subarray` keeps its 8 KiB pool slab alive ([L12](../03-data-and-io/12-buffers-and-encodings.md)) | `Buffer.from(slice)` to copy |
| **Per-key resources that never expire** | A map of Agents, clients or sockets per tenant | One shared resource, or an LRU that closes on eviction |

The closure case deserves its own example, because the retention isn't visible in the code you're reading:

```ts
// ❌ Retains the whole 50 MB file for as long as the listener is registered
const file = await readFile(statementPath);          // 50 MB Buffer
const header = file.subarray(0, 64);                 // a VIEW: it shares file's memory
const size = file.length;
logger.info({ size }, "loaded");
events.on("statement.requested", () => sendHeader(header));

// ✅ Copy the 64 bytes you need; the 50 MB becomes unreachable when this scope ends
const header = Buffer.from(file.subarray(0, 64));
```

Two mechanisms stack up there. `subarray` shares memory, so the 64-byte view keeps the 50 MB allocation alive. And V8 gives each scope **one shared context object** for all the variables that any closure in that scope captures, so a long-lived closure can keep alive a variable that only a *different* closure in the same scope used. Narrow scopes and copy small things out.

### Ledger's three leaks

1. **The idempotency response cache.** Someone "optimised" the idempotency middleware with `const responses = new Map<string, StoredResponse>()` in front of Postgres. Every key ever seen stays forever, and Postgres is the source of truth anyway. Fix: delete it, or bound it.
2. **A per-merchant `Agent` map.** `agents.get(merchantId) ?? new Agent(…)` for webhook sends creates an Agent with its own sockets per merchant and never closes one. Memory *and* file descriptors grow with the merchant count. Fix: one shared Agent with per-origin connection limits ([L18](../04-networking/18-outbound-http.md)).
3. **A per-SSE-connection heartbeat.** `setInterval(() => res.write(":\n\n"), 15_000)` with no `clearInterval` on `close`. Each disconnected client leaves a live timer holding its `res`. Fix: clear it in the `close` handler ([L20](../04-networking/20-realtime-servers.md)).

The bounded cache, as Ledger's `lib/lru.ts`:

```ts
export class LruCache<K, V> {
  readonly #map = new Map<K, { value: V; expiresAt: number }>();
  readonly #max: number;
  readonly #ttlMs: number;

  constructor(opts: { max: number; ttlMs: number }) {
    this.#max = opts.max;
    this.#ttlMs = opts.ttlMs;
  }

  get(key: K): V | undefined {
    const hit = this.#map.get(key);
    if (!hit) return undefined;
    this.#map.delete(key);
    if (hit.expiresAt <= Date.now()) return undefined;      // expired: dropped
    this.#map.set(key, hit);                                // re-insert = most recently used
    return hit.value;
  }

  set(key: K, value: V): void {
    this.#map.delete(key);
    this.#map.set(key, { value, expiresAt: Date.now() + this.#ttlMs });
    if (this.#map.size > this.#max) this.#map.delete(this.#map.keys().next().value!);   // evict oldest
  }
}
```

A `Map` iterates in insertion order, so deleting and re-inserting on every hit keeps the least recently used key first. **`max` is what bounds memory**; TTL only bounds staleness, because an expired entry that's never read again still sits there until size pushes it out.

---

## 8. Finding a leak: three heap snapshots

| Tool | How | Cost |
|---|---|---|
| `v8.writeHeapSnapshot()` | From code, behind an admin endpoint | **Synchronous: blocks the loop** for seconds on a big heap, and needs memory on the order of the heap size again |
| `--heapsnapshot-signal=SIGUSR2` | `kill -USR2 <pid>` (POSIX only: WSL2 or Linux) | Same |
| `--heapsnapshot-near-heap-limit=2` | Writes snapshots automatically as the heap approaches the limit | Same, at the worst possible moment |
| DevTools Memory tab via `--inspect` | Heap snapshots, allocation timeline, allocation sampling | Development and staging |
| `--heap-prof` | Sampling heap profiler, writes a `.heapprofile` on exit | Low; shows *where* memory is allocated |

**The three-snapshot technique:**

```text
warm up (JIT, pools, legitimate caches fill) ──▶ snapshot 1
run a fixed load, e.g. 10,000 requests         ──▶ snapshot 2
run the same load again                        ──▶ snapshot 3

In snapshot 3, Summary view, filter: "Objects allocated between Snapshot 1 and Snapshot 2"
```

Each snapshot forces a full GC first, so it contains only reachable objects. The objects allocated during the first load run that are *still alive* in snapshot 3 have survived a whole second run and several full GCs. Request-scoped objects should all be gone by then. **What remains is your leak**, grouped by constructor. Two snapshots aren't enough: everything in flight or warming up at the second snapshot looks like a leak too.

Then read the evidence:

- **Shallow size** is the object's own bytes. **Retained size** is what would be freed if that object became unreachable: everything reachable only through it. A `Map` with a shallow size of a few hundred bytes can retain 400 MB. Sort by retained size.
- The **Retainers** panel shows the **retaining path**: the chain of references from a GC root down to your object, like `(GC roots) → … → Module (idempotency.ts) → responses: Map → table → StoredResponse`. The retaining path names the variable to fix. That's the whole point of the exercise.

**Snapshots in production need care.** The process freezes while the snapshot is written, and the extra memory can push a pod that's already near its limit into the very OOMKill you're investigating. Fail the instance's readiness first so it stops taking traffic ([L24](../05-concurrency/24-graceful-shutdown.md)), check the memory headroom, and write to a volume. And **a heap snapshot contains everything in memory**: API keys, webhook secrets, customer data. Handle it like a database dump.

---

## 9. Weak references

| Tool | Semantics | Right use |
|---|---|---|
| `WeakMap` | Keys are held weakly; the entry vanishes once the key object is otherwise unreachable. Keys must be objects | Attaching data to objects you don't own (per-request metadata keyed by `req`) |
| `WeakRef` | `ref.deref()` returns the object or `undefined` once collected | A cache of large, recomputable objects |
| `FinalizationRegistry` | A callback some time after an object is collected, **or never** | Diagnostics, or a backstop that logs "you forgot to close this" |

A `WeakMap` is **not** a cache for string keys: strings can't be keys, and the lifetime is tied to the key object, not to time or size. And `FinalizationRegistry` is not for cleanup logic: the callback timing is unspecified, it may never run (for example at process exit), and GC is triggered by memory pressure, not by how many file handles or database clients you've leaked. **Resources that matter are closed explicitly**, in a `finally` or a shutdown hook.

---

## 10. Production rules

| Rule | Why |
|---|---|
| **Set `--max-old-space-size` to ~75% of the container limit** | Leaves room for off-heap memory and turns silent OOMKills into heap errors with evidence |
| **Multiply heaps by processes and worker threads** | Each has its own heap against one container limit |
| **Graph `heapUsed` against `heap_size_limit`, and RSS against the container limit** | They fail differently; you need both |
| **Watch the troughs of `heapUsed`, not its peaks** | Rising troughs after major GCs are the leak signature |
| **Every in-process cache has a `max`** | TTL alone doesn't bound memory |
| **Clear every timer and remove every listener on every exit path** | Active timers and emitters are GC roots |
| **Copy small slices out of big Buffers you keep** | A view retains its whole parent allocation |
| **Export GC pause durations alongside loop delay** | GC is a source of tail latency and an early leak warning |
| **Arm `--heapsnapshot-near-heap-limit` where you can afford the disk and memory** | Evidence from the moment it matters |
| **Take snapshots out of rotation, with headroom, and treat them as secret** | They freeze the process, cost memory, and contain everything |
| **Close resources explicitly; never rely on finalizers** | Finalization timing is unspecified and may never happen |

---

## 11. Interview traps

**Q1. "Memory grows 50 MB an hour. How do you find the leak?"**
First confirm it's a leak and locate it: does `heapUsed` grow (rising troughs after major GCs), or only RSS and `external`? If it's the heap, reproduce under load and use three heap snapshots, filtering snapshot 3 to objects allocated between 1 and 2, sort by retained size, and follow the retaining path to the variable holding it. If only RSS grows, look off-heap: Buffers, native modules, allocator behaviour. The weak answer is "restart it nightly" or "increase the memory limit": that hides the bug and moves the crash.
**Follow-up:** "Why three snapshots and not two?" Two can't distinguish a leak from in-flight or warming-up objects; three show what keeps surviving.

**Q2. "The pod was OOMKilled, exit 137, but there's no heap error. Why?"**
The kernel killed it because the container's RSS crossed its cgroup limit, and RSS includes far more than the V8 heap: Buffers, native allocations, stacks, code, other processes or worker threads in the same container. Either off-heap memory grew, or the heap limit was set above what the container can hold, so the kernel struck before V8 could fail with a message. Check `external`/`arrayBuffers` and the heap limit against the container limit.

**Q3. "What does `--max-old-space-size` control, and how do you set it in a container?"**
It caps V8's old generation, in MiB; total heap also includes new space. Set it explicitly to about 75% of the container's memory limit, via `NODE_OPTIONS` in the same manifest as the limit, leaving the rest for off-heap memory. Too high gives silent OOMKills; too low gives GC thrash.

**Q4. "Explain V8's generational GC."**
Most objects die young, so the heap is split. New space is small, allocated by pointer bump, and collected by the Scavenger, which copies survivors to the other semi-space: cost proportional to live objects, so frequent and short. Survivors get promoted to old space, collected by Mark-Compact, which Orinoco runs incrementally, concurrently and in parallel to keep pauses short but not zero. Large objects have their own space and are never moved.

**Q5. "`rss` vs `heapUsed` vs `external`?"**
`heapUsed` is JavaScript objects in the V8 heap; `external` is memory owned by C++ objects tied to JavaScript objects, including all Buffers (`arrayBuffers`); `rss` is everything resident: heap, external, native allocations, stacks, code and the binary. The container limit counts RSS, which is why a small heap can still be OOMKilled.

**Q6. "Name four leak patterns in Node."**
Unbounded Map caches, listeners added per request and never removed, intervals never cleared (an active timer is a root), and closures retaining large objects, including through V8's shared scope context. Bonus points: a small `subarray` retaining a big Buffer, pending-promise maps whose replies never come, and per-tenant resource maps such as one HTTP Agent per merchant.

**Q7. "What is a retaining path, and what's retained size?"**
The retaining path is the chain of references from a GC root to an object: the reason it's still alive, and therefore the line of code to change. Retained size is how much memory would be freed if that object became unreachable, as opposed to its shallow size. Leaks are found by sorting on retained size and reading the retainers.

**Q8. "Is it safe to take a heap snapshot in production?"**
Carefully. Writing one is synchronous, so the process freezes for seconds on a large heap, and it needs substantial extra memory, which can trigger an OOMKill on a pod near its limit. Take the instance out of rotation, confirm headroom, write to a volume, and treat the file as sensitive because it contains every secret and record in memory. Often the better move is reproducing in staging with the same traffic.

**Q9. "Can you use `FinalizationRegistry` to close database connections that callers forget?"**
No. Finalizers may run late or never, and GC is triggered by heap pressure, not by connection exhaustion, so you'd run out of connections long before anything got collected. Close resources explicitly in `finally` blocks and shutdown hooks; a finalizer is at most a diagnostic that logs a leak.

**Q10. "p99 spikes every few seconds and the heap graph is a steep sawtooth. What's going on?"**
A high allocation rate is driving frequent GCs, and the spikes are GC pauses stopping the event loop. Confirm with `--trace-gc` or perf_hooks `gc` entries correlated with the latency spikes. Reduce allocation in the hot path first (big intermediate arrays, per-request schema construction, buffering instead of streaming); raising `--max-semi-space-size` can help an allocation-heavy service, measured before and after.

---

## 12. Build & break

### Build — memory tooling for Ledger
- `GET /debug/heap` on the internal admin port: authenticated, one snapshot at a time (a simple in-flight flag), logs `memoryUsage()` and the heap limit before writing, calls `v8.writeHeapSnapshot(path)` into a configured directory, and returns the filename. Document the "take it out of rotation first" step in its handler comment.
- Memory metrics every 10 s: `rss`, `heapUsed`, `heapTotal`, `external`, `arrayBuffers`, `heap_size_limit`, plus the GC pause histogram from §6 (exported properly in [L29](29-observability.md)).
- `lib/lru.ts` from §7, with tests for eviction order, TTL expiry and the `max` bound.

### Build — Ledger's leak hunt
Plant the three leaks from §7 behind a `LEAKY=1` flag. For each one: reproduce under `npx autocannon`, take three snapshots, record the retaining path you found, apply the fix, and repeat the snapshots to show the troughs go flat. Keep the write-up: it's evidence for the blueprint's definition of done.

### Break — 4 experiments
1. **The global Map.** Add `const seen = new Map<string, unknown>()` that stores every request body under its request id. Run `npx autocannon -c 50 -d 60`, taking snapshots with `/debug/heap` after warm-up, after the first run and after a second identical run. Load all three in DevTools (Memory tab → Load), select snapshot 3, filter "Objects allocated between Snapshot 1 and Snapshot 2", sort by retained size, and follow the retainers to `seen`. Replace it with `LruCache` and repeat.
2. **OOMKilled with a small heap.** Write `oom.ts` that every 100 ms pushes `Buffer.allocUnsafe(8 * 1024 * 1024).fill(1)` into an array and logs `memoryUsage()` (the `fill` touches every page so it becomes resident). In Git Bash run `MSYS_NO_PATHCONV=1 docker run --name oom -m 256m --memory-swap 256m -v "$(pwd -W)":/app -w /app node:24 node oom.ts`. Watch `heapUsed` stay tiny while `rss` and `arrayBuffers` climb, until the output just stops. Then `docker inspect oom --format '{{.State.OOMKilled}} {{.State.ExitCode}}'` prints `true 137`. Clean up with `docker rm oom`.
3. **The heap limit.** Run `node --max-old-space-size=64 --trace-gc heap-oom.ts`, where the script keeps pushing small objects into an array. Watch scavenges turn into back-to-back mark-compacts that free almost nothing, then the `FATAL ERROR … JavaScript heap out of memory`. Run it in WSL2 or Docker to see exit code 134; on Windows the exit code differs. Add `--heapsnapshot-near-heap-limit=1` and open the snapshot it leaves behind.
4. **A listener per request.** In a handler, add `domainEvents.on("payment.succeeded", () => res.locals.count++)` and never remove it. Under load you'll see `MaxListenersExceededWarning` after 10 listeners. Take a snapshot and find the emitter's listener array in the retainers of a `ServerResponse` that should have died long ago. Fix it with `once` plus removal on `res.on("close")`.

### Explain out loud (60–120 seconds)
1. Heap OOM vs OOMKilled: who kills, why, and the evidence each leaves.
2. New space and the Scavenger, old space and Mark-Compact, and why pauses are short but not zero.
3. The five `memoryUsage()` fields, and why RSS is bigger than the heap.
4. How to set `--max-old-space-size` in a container, and what goes wrong in each direction.
5. Four leak patterns, and the three-snapshot technique with retaining paths.

---

## What's next

Memory is one half of "why is the process struggling?"; the other is CPU. Next you'll load-test properly, read a flame graph, and learn enough about V8's compiler pipeline (Ignition, Sparkplug, Maglev, TurboFan) to recognise the code shapes that make it give up on optimizing.

Next → **[Lesson 28: Performance: profiling & how V8 runs your code](28-performance-and-v8.md)**
