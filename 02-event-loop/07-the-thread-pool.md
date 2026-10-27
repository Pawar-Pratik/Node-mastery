# Lesson 07 — The thread pool: what actually runs in parallel

> **Why this lesson exists:** "Node is single-threaded" hides a second, smaller truth: file I/O, `dns.lookup`, async crypto and async compression all queue for a pool of **four** threads that nobody sized. It's invisible until it saturates, and then unrelated operations slow down together, with low CPU and a healthy event loop, so the usual suspects all look innocent. Being able to say which APIs use the pool, prove it's saturated and size it with arithmetic is a genuinely senior skill, and it's the answer to "why did file reads slow down when DNS got slow?"

**Time:** ~60 minutes · **Prereq:** Lesson 05

---

## 1. The idea in one sentence

> **Network I/O waits in the kernel and scales with connections; file I/O, `dns.lookup`, async crypto and async zlib queue for a pool of four threads per process, so when any one of them saturates the pool, all of them slow down together.**

---

## 2. Why a pool exists at all

[Lesson 05](05-the-event-loop.md) showed poll waiting on sockets with epoll, kqueue or IOCP. That works because the kernel can tell you when a *socket* is ready. Three kinds of work don't fit that model:

- **Regular files.** Linux's epoll doesn't accept regular files at all; to the kernel they're always "ready", even when a read will block on a slow disk. libuv runs file operations on worker threads on every platform, Windows included.
- **`getaddrinfo`.** The system resolver is a blocking libc call that reads `/etc/hosts`, `nsswitch` and the configured nameservers. There's no portable async version.
- **CPU work in C++.** `pbkdf2`, `scrypt` and `gzip` are pure computation. "Async" for them means *on another thread*.

So libuv keeps a small pool of threads and a work queue:

```text
 JS thread (the loop)                       libuv thread pool (default 4 threads)
 ─────────────────────                      ──────────────────────────────────────
 fs.readFile(...)   ──┐                     ┌─▶ [thread 1] read()         ─┐
 dns.lookup(...)    ──┼─▶ work queue (FIFO) ┼─▶ [thread 2] getaddrinfo()   │ blocking calls
 crypto.scrypt(...) ──┤   unbounded         ├─▶ [thread 3] scrypt()        │ and CPU work
 zlib.gzip(...)     ──┘                     └─▶ [thread 4] deflate()      ─┘
                                                          │
 poll phase ◀── async wake-up ◀── completion queue ◀─────┘
 your callback runs HERE, on the loop thread

 Sockets never enter this picture: the kernel watches them, the loop is told when they're ready.
```

**The queue is unbounded and first-in, first-out.** A fifth request doesn't fail; it waits behind the four running ones, whatever they are. That's the whole failure mode.

And one call is often several trips. `fs.readFile` is an `open`, an `fstat`, one `read` per chunk and a `close`, each a separate pool request that rejoins the back of the queue. **Under saturation, file-heavy code degrades faster than the wait for a single slot suggests.**

---

## 3. What runs where

| API | Runs on | Note |
|---|---|---|
| TCP/UDP sockets, HTTP request and response bytes, pipes | **Kernel** (epoll / kqueue / IOCP) | Scales to many thousands of connections with no threads |
| Timers, `setImmediate` | **Loop** | No pool involvement |
| `fs.*` callbacks and `node:fs/promises` | **Pool** | Every async file operation, including `stat` and `open` |
| `fs.watch` | **Kernel** notifications | inotify, FSEvents/kqueue, `ReadDirectoryChangesW` |
| `dns.lookup()` | **Pool** (`getaddrinfo`) | **Used by default by `http`, `https`, `net` and `fetch` whenever you pass a hostname** |
| `dns.resolve*()`, `dns.Resolver` | **Loop** (c-ares, over the network) | Doesn't consult `/etc/hosts` |
| `crypto.pbkdf2`, `scrypt`, `randomBytes`, `randomFill`, `generateKeyPair` (async forms) | **Pool** | CPU-bound: each call occupies a thread for its full cost |
| `crypto.createHash`, `createHmac`, `timingSafeEqual` | **Loop**, synchronously | Microseconds for small inputs |
| `zlib.gzip`, `deflate`, `brotliCompress`, and their streams | **Pool** | Compression middleware lives here |
| `readFileSync`, `pbkdf2Sync`, `gzipSync`, every `*Sync` | **Loop**, blocking it | No pool contention, but nothing else runs ([Lesson 08](08-blocking-the-loop.md)) |
| Native addons using N-API async work (`bcrypt`, `argon2`) | **Pool** | The same four threads |
| `worker_threads` | Their **own** threads and loops | But their `fs` and DNS calls share the process's one pool |

**Two corrections to common answers.** "Node uses a thread pool for I/O" is half wrong: network I/O, the thing most servers mostly do, never touches it. And "async crypto is free" is wrong: it moves the cost off the loop, onto four shared threads.

---

## 4. The pbkdf2 demo: four together, the fifth waits

```ts
// pool-demo.ts — run: node pool-demo.ts 5
import { pbkdf2 } from "node:crypto";

const n = Number(process.argv[2] ?? 5);
const start = performance.now();
for (let i = 1; i <= n; i++) {
  pbkdf2("secret", "salt", 300_000, 64, "sha512", (err) => {
    if (err) throw err;
    console.log(`hash ${i}: ${(performance.now() - start).toFixed(0)} ms`);
  });
}
```

```text
4 hashes:  ████████ ████████ ████████ ████████        all four finish together
5 hashes:  ████████ ████████ ████████ ████████
                                              ████████  the fifth starts when a thread frees up
           0                                  T       ≈2T
```

With four calls, all four finish at about the same time T. With five, the first four finish at about T and **the fifth at roughly 2T**: it sat in the queue. With eight, two waves. Raise the pool with `UV_THREADPOOL_SIZE=8 node pool-demo.ts 8` and the eight finish together again, *if you have eight cores to run them on*. The order in which they finish varies; the waves don't.

---

## 5. The `dns.lookup` trap

Ledger's webhook dispatcher calls merchant URLs: `https://hooks.merchant.example/ledger`. Every new connection to a hostname starts with `dns.lookup`, which runs `getaddrinfo` **on a pool thread**, and that thread is held until the resolver answers or gives up.

```text
Resolver healthy:  lookup ▏2 ms        pool threads mostly free; fs.readFile ≈ 1 ms

Resolver slow:     lookup ████████████ several seconds (timeouts and retries)
                   lookup ████████████
                   lookup ████████████   ← 4 lookups hold all 4 threads
                   lookup ████████████
                   fs.readFile ░░░░░░░░░░░░ waits in the queue behind them
                   crypto.scrypt ░░░░░░░░░░ waits too
```

**That's the incident from the README: file reads slow down when DNS gets slow, with CPU near zero and loop delay normal.** Nothing in the code that reads files changed. And you can't escape by timing out the HTTP request: aborting a `fetch` abandons the *request*, but a `getaddrinfo` that has started can't be cancelled, so the thread stays occupied until the resolver gives up.

Mitigations, in order of impact:

1. **Reuse connections.** A keep-alive connection does one lookup, then carries hundreds of requests. `http.Agent` has had `keepAlive` on by default since Node 19, and undici pools connections per origin ([Lesson 18](../04-networking/18-outbound-http.md)). Creating a new agent per request multiplies lookups.
2. **Cache lookups.** Put a caching layer in front of `lookup` with a TTL, either undici's DNS-caching interceptor or a small cache of your own. Respect the record's TTL and cap it, or a failover to a new IP will take minutes to reach you.
3. **Use `dns.resolve4`/`resolve6` for external hostnames** via a custom `lookup` option. c-ares queries nameservers over sockets on the loop: no pool thread. The trade-off is semantics: it skips `/etc/hosts` and the system resolver configuration, so don't use it for names that only exist there.
4. **Size the pool** for the DNS you actually do (§6).

The protocol-level view of DNS cost is in [API Lesson 02](https://github.com/Pawar-Pratik/API-mastery/blob/main/01-foundations/02-journey-of-a-request.md); this lesson is the runtime half of that story.

---

## 6. `UV_THREADPOOL_SIZE`: when, how far, and how to set it

**Facts.** The default is 4 and the maximum is 1024. The pool belongs to the **process**: every worker thread shares it. libuv reads the variable **once, when the pool first starts**, which is the first time anything uses it.

**Set it at launch, from the environment:**

```bash
UV_THREADPOOL_SIZE=8 node src/main.ts         # Git Bash, WSL2, Linux, macOS
# PowerShell:  $env:UV_THREADPOOL_SIZE = "8"; node src/main.ts
# Dockerfile:  ENV UV_THREADPOOL_SIZE=8      Kubernetes: env: [{ name: UV_THREADPOOL_SIZE, value: "8" }]
```

Assigning `process.env.UV_THREADPOOL_SIZE` inside your program is unreliable: in ESM your assignment runs only after every static import has been evaluated, any earlier pool use has already fixed the size, and on Windows it's ignored even when nothing has touched the pool yet (Break 2 shows it). **Treat it as launch configuration, like `--max-old-space-size`.**

**How far to raise it** depends on what the pool is doing:

| Pool work | More threads help? | Why |
|---|---|---|
| CPU-bound (crypto, zlib) | **Only up to the cores you actually get** | Beyond that, threads just take turns and add context switches. In a container with a 2-CPU limit, four busy pool threads plus the loop already oversubscribe it |
| Wait-bound (`getaddrinfo` on a slow resolver, files on network storage) | **Yes, beyond the core count** | A waiting thread uses no CPU; more threads means more waits in parallel |

**Do the arithmetic.** By Little's law ([Lesson 10](10-cancellation-and-concurrency.md)), threads busy on average = arrival rate × time each holds a thread. If logins arrive at 40/s and each `scrypt` holds a thread for 100 ms, that's 40 × 0.1 = **4 threads busy on average**: the entire default pool, before a single file read or DNS lookup. Either raise the pool and give the pod the cores to back it, or cap concurrent hashes below the pool size so the rest of the process keeps a lane.

Each thread also costs memory for its stack, so 1024 is a ceiling, not a target. Raise it in small steps, each one justified by a measurement.

---

## 7. Sync variants, native addons and compression

**`*Sync` APIs skip the pool and block the loop instead.** `pbkdf2Sync` in a request handler doesn't contend with file reads; it stops *everything* for the duration of the hash. That's [Lesson 08](08-blocking-the-loop.md)'s problem, and it's always worse than pool contention in a server. (At startup and in CLIs, `readFileSync` is fine.)

**Native vs pure JavaScript.** `bcrypt` is a native addon whose async functions run on the libuv pool: off the loop, competing with `fs` and DNS. `bcryptjs` is pure JavaScript: it computes on the JavaScript thread itself, so whatever its API looks like, every CPU cycle of the hash comes out of the loop's budget. On a server, use the native one, or `crypto.scrypt`, which ships with Node.

**Compression.** Express's `compression` middleware and `zlib.createGzip()` streams compress on the pool, chunk by chunk. Under load, many compressing responses keep pool threads busy, and file reads and lookups queue behind them. Brotli is the sharp edge: **Node's default Brotli quality is 11, the maximum and slowest setting**, which is meant for static assets compressed once, not for dynamic responses. Set `BROTLI_PARAM_QUALITY` explicitly (a mid-range value) for anything compressed per request, or compress at the CDN or proxy instead.

---

## 8. Seeing saturation

There's no built-in metric for the pool's queue length. You infer it:

- **The signature:** `fs`, DNS and crypto latency rise together, while event-loop delay stays normal and endpoints that only touch the network (a cached response, a pure proxy) stay fast.
- **A canary:** every second, time a tiny pool operation (an `fs.stat` of a small file the OS has cached). Its normal latency is well under a millisecond, so anything above a few milliseconds is queue wait. Export it as a metric next to loop delay:

```ts
// src/lib/pool-canary.ts — pool queue wait as a metric
import { stat } from "node:fs/promises";

export function startPoolCanary(onSample: (ms: number) => void, everyMs = 1_000): () => void {
  const timer = setInterval(() => {
    const t0 = performance.now();
    stat(import.meta.filename).then(
      () => onSample(performance.now() - t0),      // normally well under 1 ms
      () => onSample(Number.POSITIVE_INFINITY),     // a failing stat deserves an alert too
    );
  }, everyMs);
  timer.unref();                                    // a metric must never keep the process alive
  return () => clearInterval(timer);
}
```
- **`process.getActiveResourcesInfo()`** (experimental) counts in-flight requests by type: many `FSReqCallback` or `GetAddrInfoReqWrap` entries means file or DNS work is piling up. Crypto jobs don't appear in it.

| Symptom | Loop delay | Pool canary | Likely cause |
|---|---|---|---|
| Everything slow | **High** | High too | The loop is blocked ([Lesson 08](08-blocking-the-loop.md)) |
| File/DNS/crypto slow, network-only fast | Normal | **High** | Pool saturated |
| Only outbound calls slow | Normal | Normal | Downstream or connection pool ([Lesson 18](../04-networking/18-outbound-http.md)) |

---

## 9. The pool vs worker threads

| | libuv thread pool | `worker_threads` |
|---|---|---|
| Runs | libuv's and Node's C/C++ work (`read`, `getaddrinfo`, `scrypt`, `deflate`) | **Your JavaScript** |
| You submit | Indirectly, by calling `fs`, `dns.lookup`, async crypto, zlib | Directly, with `postMessage` |
| Each unit has | A thread | A thread, a V8 isolate, a heap and an event loop |
| Sized by | `UV_THREADPOOL_SIZE`, once, at launch | You, by how many workers you create |
| Use it for | Nothing: it's used for you | CPU-heavy JavaScript (parsing, matching, transforms) |

You can't put your own JavaScript on the libuv pool. When the CPU-heavy work is yours, as with Ledger's reconciler, that's [Lesson 21](../05-concurrency/21-worker-threads.md).

---

## 10. Ledger: API keys vs passwords, and sizing on purpose

**API-key verification is SHA-256 on the loop, synchronously, and that's correct.** The key has 256 bits of entropy, so a fast hash is the right tool ([API Lesson 12](https://github.com/Pawar-Pratik/API-mastery/blob/main/03-security/12-authentication-landscape.md) explains why). Hashing a 40-byte secret takes microseconds; sending it to the pool would cost more in queueing and wake-ups than the hash itself.

```ts
import { createHash, timingSafeEqual } from "node:crypto";

export function keyMatches(secret: string, storedHash: Buffer): boolean {
  const digest = createHash("sha256").update(secret).digest();   // ✅ microseconds, on the loop
  return digest.length === storedHash.length && timingSafeEqual(digest, storedHash);
}
```

**Console password logins use `scrypt` on the pool, and that's where sizing matters.** In the API process the pool is shared by three unrelated jobs: login hashing, the statement exporter's gzip stream, and DNS for outbound calls. A credential-stuffing burst against the login endpoint would fill all four threads with `scrypt`, and exports and outbound calls would stall behind it. The decision, written down:

- **Cap concurrent password hashes** with a small semaphore (2 in an API pod with a 2-CPU limit), so hashing can never take the whole pool. The cap belongs next to the login rate limit ([API Lesson 19](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/19-rate-limiting.md)).
- **Set `UV_THREADPOOL_SIZE=6` in the API pod spec**: one thread per concurrent export (each gzip stream keeps one chunk on the pool, and exports are capped at four per replica) plus two, so logins, DNS and `fs` never queue behind compression. Keep the CPU limit consistent with the CPU-bound part. The whole fleet budget is in the [Ledger Engine blueprint](../09-project/34-ledger-engine-blueprint.md).
- **Export the pool canary** alongside loop delay, so the next saturation is a graph, not a mystery.

---

## 11. Production rules

| Rule | Why |
|---|---|
| **Know which of your calls use the pool** | `fs`, `dns.lookup`, async crypto and zlib share four threads by default |
| **Set `UV_THREADPOOL_SIZE` in the environment at launch** | It's read once at first use; in-code assignment is unreliable, and ignored on Windows |
| **Size CPU-bound pool work to real cores; wait-bound work can go higher** | Extra threads for CPU work just add context switches, especially under a container CPU limit |
| **Reuse connections and cache DNS** | Each new connection to a hostname is a `getaddrinfo` holding a pool thread |
| **Cap concurrency of expensive pool work (password hashing)** | One endpoint must not be able to take the whole pool |
| **Never use `*Sync` crypto or zlib in a request path** | It blocks the loop, which is worse than queueing |
| **Use native or built-in hashing (`crypto.scrypt`, `bcrypt`), not pure-JS hashing, on servers** | Pure JS hashing burns the loop thread |
| **Set Brotli quality explicitly for dynamic responses** | The default is 11, the slowest setting |
| **Run a pool canary metric next to loop delay** | Saturation is invisible in CPU and loop metrics |

> **Spring equivalent:** the closest analogue is a small, shared, default executor that unrelated work silently contends for: `CompletableFuture.supplyAsync` without an executor runs on the common `ForkJoinPool`, and `@Async` with the default executor has its own hazards. The fix is the same in both runtimes: know what's on the shared pool, size it deliberately, and give expensive work its own bound. See [Async, scheduling & events](https://github.com/Pawar-Pratik/SpringBoot/blob/main/09-production/32-async-scheduling-events.md).

---

## 12. Interview traps

**Q1. "Is Node's I/O handled by a thread pool?"**
Partly, and the distinction is the answer. Network sockets are handled by the kernel's readiness APIs (epoll, kqueue, IOCP) on the loop thread, with no pool. File system operations, `dns.lookup`, async crypto and async zlib run on libuv's pool, four threads by default. The weak answer, "yes, libuv has a thread pool for I/O", implies 10,000 connections need threads, which they don't.

**Q2. "Which operations use the libuv pool?"**
All async `fs` calls (callback and promise forms), `dns.lookup` (and therefore `http`, `net` and `fetch` when given a hostname), async `pbkdf2`, `scrypt`, `randomBytes`, `randomFill` and `generateKeyPair`, async zlib including streams, and native addons that use N-API async work. Not sockets, not timers, not `dns.resolve*`. **Follow-up:** "And `crypto.createHash`?" Synchronous, on the loop: fine for small inputs, a blocker for huge ones.

**Q3. "Why would slow DNS slow down file reads?"**
`http.get` with a hostname calls `dns.lookup`, which is `getaddrinfo` on a pool thread. A slow resolver holds each thread for seconds; four concurrent lookups occupy the whole default pool; `fs.readFile` queues behind them. CPU and loop delay look fine, which is why it's hard to spot. The fixes: connection reuse, DNS caching, `dns.resolve` for external names, and sizing the pool. **Follow-up:** "Doesn't a request timeout free the thread?" No: an in-progress `getaddrinfo` can't be cancelled, so the thread stays busy until the resolver gives up.

**Q4. "How do you size `UV_THREADPOOL_SIZE`?"**
From what the pool does. CPU-bound work (hashing, compression) gains nothing beyond the cores the container actually gets; wait-bound work (slow DNS, network file systems) can use more. Use Little's law: arrival rate × hold time = threads busy, so 40 logins/s × 100 ms of `scrypt` is 4 busy threads. Set it in the environment at launch, measure with a pool canary, and cap expensive work so it can't take every thread.

**Q5. "I set `process.env.UV_THREADPOOL_SIZE = '16'` at the top of `main.ts` and nothing changed. Why?"**
libuv reads it once, when the pool first starts. In ESM, static imports are evaluated before your first statement, so anything they did on the pool already fixed the size; on Windows the in-process assignment is ignored outright. Set it in the launch environment: the shell, the Dockerfile, or the pod spec.

**Q6. "`bcrypt` or `bcryptjs` in a Node service?"**
`bcrypt`: it's a native addon whose async API runs on the libuv pool, so the loop stays free. `bcryptjs` is pure JavaScript and computes on the loop thread, so every login steals CPU from every other request. Either way, password hashing occupies a pool thread or the loop for tens of milliseconds by design, so bound its concurrency.

**Q7. "Thread pool vs worker threads?"**
The pool runs libuv's C work on your behalf; you can't submit JavaScript to it. Worker threads run *your* JavaScript, each with its own isolate, heap and event loop, and you create and size them yourself. Workers share the process's single libuv pool for their own `fs` and DNS calls.

**Q8. "Five `pbkdf2` calls start at once on a default Node process. Describe the timings."**
Four start immediately on the four pool threads and finish together at about T; the fifth waits in the FIFO queue and finishes at about 2T. With `UV_THREADPOOL_SIZE=5` and at least five free cores, all five finish together. That two-wave pattern is the signature of pool saturation.

**Q9. "Why is SHA-256 fine synchronously for API keys, when password hashing has to be async?"**
SHA-256 over a short secret takes microseconds; the round trip to the pool would cost more than the work. Password hashes (`scrypt`, `bcrypt`, argon2) are deliberately slow, tens of milliseconds, so running them synchronously would block every request for that long. The threat model decides the hash, and the hash's cost decides the thread.

**Q10. "After enabling compression, static file serving got slower under load, though CPU wasn't maxed. Why?"**
Compression runs on the same pool as file reads. Many concurrent compressing responses keep the threads busy, so `fs` operations queue. Brotli at its default quality 11 makes it much worse. Lower the quality, compress at a proxy or CDN, pre-compress static assets, or raise the pool if you have cores to spare.

---

## 13. Build & break

### Build — a pool-pressure harness

`bench/pool-pressure.ts`: keep N `scrypt` calls in flight for 5 seconds while a canary times `fs.stat` every 20 ms, then print the canary's p50 and p99 and the number of hashes completed.

```ts
// bench/pool-pressure.ts — run: node bench/pool-pressure.ts 4
import { scrypt } from "node:crypto";
import { stat } from "node:fs/promises";

const inFlight = Number(process.argv[2] ?? 4);
const SCRYPT = { N: 2 ** 15, maxmem: 64 * 1024 * 1024 };
const deadline = performance.now() + 5_000;
const canary: number[] = [];
let hashes = 0;

const probe = setInterval(() => {
  const t0 = performance.now();
  stat(import.meta.filename).then(
    () => { canary.push(performance.now() - t0); },
    (err: unknown) => console.error("canary failed", err),
  );
}, 20);

const hashOnce = (i: number) =>
  new Promise<void>((resolve, reject) =>
    scrypt(`pw${i}`, "salt", 64, SCRYPT, (err) => (err ? reject(err) : resolve())));

async function hashLoop(i: number): Promise<void> {
  while (performance.now() < deadline) { await hashOnce(i); hashes++; }
}

await Promise.all(Array.from({ length: inFlight }, (_, i) => hashLoop(i)));
clearInterval(probe);

const sorted = canary.toSorted((a, b) => a - b);
const pct = (p: number) => (sorted[Math.min(sorted.length - 1, Math.floor(p * sorted.length))] ?? 0).toFixed(1);
console.log(`${inFlight} in flight: ${hashes} hashes, fs.stat p50=${pct(0.5)} ms p99=${pct(0.99)} ms`);
```

Run it with 1, 2, 3, 4, 6 and 8 in flight, and tabulate. Up to 3 the canary stays sub-millisecond, because one thread is always free; at 4 its p99 jumps towards the duration of one hash, because a `stat` now has to wait for a thread to free up. Then repeat with `UV_THREADPOOL_SIZE=8` and explain both what changes and what doesn't (hashes per second stops scaling once you run out of cores).

### Break — four experiments

1. **Waves.** Run `pool-demo.ts` from §4 with 4, 5 and 8. Write down the timings, then draw the waves.
2. **Sizing, three ways.** `UV_THREADPOOL_SIZE=8 node pool-demo.ts 8` finishes in one wave. Now try `=64` with 64 hashes and compare total time to `=8`: no gain once the cores are busy. Finally, delete the variable and put `process.env.UV_THREADPOOL_SIZE = "8"` on the first line of an `.mjs` file that `await import("node:crypto")`s afterwards. On Windows you still get two waves.
3. **Slow DNS without breaking your machine.** Give `http.get` a custom `lookup` that holds a pool thread before resolving, which is exactly what a slow `getaddrinfo` does:
   ```ts
   import { lookup } from "node:dns";
   import type { LookupFunction } from "node:net";
   import { pbkdf2 } from "node:crypto";

   // Holds a pool thread for about a second, then really resolves. A setTimeout delay would NOT
   // reproduce the incident: it delays the request without occupying a pool thread.
   export const slowLookup: LookupFunction = (host, opts, cb) => {
     pbkdf2("x", "y", 1_000_000, 32, "sha512", () => lookup(host, opts, cb));
   };
   ```
   Fire 8 requests at a local server with `{ host: "localhost", port, lookup: slowLookup }` and time an `fs.readFile` of a small file before and during: milliseconds before, seconds during. That comment is the whole lesson: what matters is *which thread is held*. For the real thing, run a container whose only nameserver is unroutable (`docker run --dns 10.255.255.1 …`, in Git Bash with `MSYS_NO_PATHCONV=1` before a `-v` mount) and watch `dns.lookup` calls hang until the resolver times out.
4. **Sync under load.** Write a server with `GET /login-sync` (`pbkdf2Sync`, 100k iterations), `GET /login` (async `pbkdf2`) and `GET /health`. Run `npx autocannon -c 20 -d 10 http://localhost:3000/login-sync` and, in a second terminal, time `curl localhost:3000/health`: the health check waits behind every queued hash. Repeat against `/login`: health stays fast, and login throughput is capped at four hashes at a time. Explain which of those two failure modes you'd rather have, and why a semaphore on hashing fixes the remaining one.

### Explain out loud (60–120 seconds)

1. Why the pool exists: regular files, `getaddrinfo` and C++ CPU work don't fit epoll.
2. The table: what's on the kernel, what's on the pool, what blocks the loop.
3. The slow-DNS-slows-file-reads incident, its signature in metrics, and four mitigations.
4. How to size `UV_THREADPOOL_SIZE` with Little's law, and why it must be set at launch.
5. Why API keys hash synchronously and passwords hash on the pool, with a concurrency cap.
6. Pool vs worker threads in one sentence each.

---

## What's next

You can now tell pool contention from a blocked loop. Next is the blocked loop itself, Node's signature incident: how to prove it with event-loop delay and utilisation, find the blocking frame in a profile, and fix giant JSON, ReDoS and synchronous work at the right layer. You'll also build the ten-line loop meter every later experiment runs with.

Next → **[Lesson 08: Blocking the loop — and measuring it](08-blocking-the-loop.md)**
