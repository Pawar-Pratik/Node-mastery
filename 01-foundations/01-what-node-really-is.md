# Lesson 01 — What Node really is

> **Why this lesson exists:** everyone calls Node "single-threaded, non-blocking and event-driven", and almost nobody can say what each of those words means mechanically. Interviewers know this, which is why "is Node single-threaded?" opens so many Node interviews. This lesson builds the map every later lesson hangs on: which component does what, which thread runs what, and why that design wins for I/O and loses for CPU.

**Time:** ~60 minutes · **Prereq:** none (the [TypeScript track](https://github.com/Pawar-Pratik/TypeScript-mastery)'s Module 1 is recommended)

---

## 1. The idea in one sentence

> **Node is a C++ program that embeds V8 to run your JavaScript on one thread, and hands every slow operation to libuv, which waits for it in the kernel or on a small thread pool. Your JavaScript thread is never the one doing the waiting.**

---

## 2. Why Node exists: waiting is expensive if it costs a thread

### The C10K problem

Around the turn of the century, the standard server design was one thread (or one process) per connection. Dan Kegel's "C10K problem" asked the obvious next question: how do you serve 10,000 concurrent connections from one machine? Thread-per-connection hits three walls:

1. **Stack memory.** Every thread reserves a stack. Java reserves about 1 MB per platform thread by default, so 10,000 threads reserve about 10 GB of address space before doing any work.
2. **Context switches.** The kernel scheduler has to juggle thousands of threads. Each switch costs kernel time and evicts useful data from the CPU caches.
3. **Most of those threads are asleep.** A typical API request holds its thread for 250 ms and uses 2 ms of CPU. [Spring L18](https://github.com/Pawar-Pratik/SpringBoot/blob/main/05-web/18-mvc-vs-webflux-virtual-threads.md) works through exactly that arithmetic: the thread is idle 99% of the time.

The insight that fixes it: **waiting doesn't need a thread.** The kernel can already watch thousands of sockets at once and report which ones are ready: `epoll` on Linux, `kqueue` on macOS and BSD, IOCP on Windows. One thread can ask "which of these 10,000 sockets has data?", sleep until the kernel answers, handle the ready ones, and ask again.

```text
Thread per connection                          Event loop
10,000 connections                             10,000 connections
= 10,000 threads, nearly all asleep            = 10,000 sockets registered with epoll
= ~10 GB of reserved stack (at 1 MB each)      = kilobytes of objects per connection
= the scheduler juggles 10,000 threads         = one thread asks the kernel "who's ready?"
```

nginx was built on this evented model and showed that it holds enormous numbers of connections in little memory. **Ryan Dahl introduced Node in 2009** with the same model, plus one bet: JavaScript. JavaScript had no blocking I/O library to inherit, and browsers had already trained a generation of programmers to write callbacks, so every Node API could be asynchronous from day one.

---

## 3. The anatomy: what's inside the `node` binary

```text
┌───────────────────────────────────────────────────────────────┐
│  Your code + npm packages                          JavaScript │
├───────────────────────────────────────────────────────────────┤
│  Node's standard library (lib/*.js)                JavaScript │
│  node:fs  node:http  node:net  node:crypto  node:stream  ...  │
├───────────────────────────────────────────────────────────────┤
│  C++ bindings (src/*.cc): glue between JS and C           C++ │
├────────────────────────────────┬──────────────────────────────┤
│  V8                            │  libuv                     C │
│  parse, interpret, JIT compile │  the event loop              │
│  the heap + garbage collector  │  epoll / kqueue / IOCP       │
│                                │  thread pool (4 by default)  │
├────────────────────────────────┴──────────────────────────────┤
│  Also bundled: llhttp (HTTP parser), c-ares (DNS resolver),   │
│  OpenSSL (TLS, crypto), zlib, undici (fetch), ICU (Intl)      │
└───────────────────────────────────────────────────────────────┘
                         Operating system kernel
```

| Layer | What it owns | What it does *not* know about |
|---|---|---|
| **V8** | Parsing, the Ignition interpreter, optimising JIT compilers (TurboFan and intermediate tiers, [L28](../07-production/28-performance-and-v8.md)), the heap where every object, string and closure lives, the generational garbage collector ([L27](../07-production/27-memory-and-gc.md)) | Files, sockets, timers. `setTimeout` isn't in the ECMAScript spec; the host (a browser or Node) provides it |
| **libuv** | The event loop, cross-platform async I/O over epoll/kqueue/IOCP, timers, signals, child processes, and a thread pool for work the OS can't do asynchronously | JavaScript. It's a C library, written for Node and now used by other runtimes too |
| **C++ bindings** | Turning a JS call like `fs.read()` into a libuv request, and calling back into V8 when it completes. Also where `Buffer` memory is allocated, *outside* the V8 heap ([L12](../03-data-and-io/12-buffers-and-encodings.md)) | Your application |
| **Standard library** | The `node:*` modules, mostly written in JavaScript. `node:http` is JavaScript on top of llhttp, which is C | — |

**So is Node a language or a framework? Neither.** It's a *runtime*: V8 plus libuv plus a standard library. The language is JavaScript (or TypeScript with the types stripped, [L03](03-the-process.md)). The frameworks are Express, Fastify and NestJS ([L19](../04-networking/19-frameworks-by-mechanism.md)). In Java terms, Node is the JVM plus the JDK, and Express is Spring.

---

## 4. Two calls, traced end to end

### `fs.readFile`: the thread-pool path

```text
 JS thread (main)                     libuv thread pool              kernel / disk
 ────────────────                     ─────────────────              ─────────────
 readFile("statement.csv", cb)
   └─ binding queues uv_fs_open ──────▶ worker: open()    ──────────▶ may block on disk
 (returns at once: your code
  keeps running, the loop turns)              │ done
 loop picks up the completion ◀────── result posted to the loop
   └─ queue fstat ────────────────────▶ worker: fstat()
   └─ queue read  ────────────────────▶ worker: read() one chunk
   └─ queue read  ────────────────────▶ worker: read() next chunk   (repeat)
   └─ queue close ────────────────────▶ worker: close()
 loop runs cb(err, data) on the JS thread
```

Four things to take from this picture:

1. **The blocking syscalls run on pool threads.** The JS thread never waits for the disk.
2. **Files use the pool because there's no portable async file API.** Readiness APIs like `epoll` don't support regular files at all, so libuv runs ordinary blocking calls on worker threads.
3. **It's many trips, not one.** `readFile` reads one chunk at a time, and the loop schedules each next chunk. If your JavaScript is hogging the thread, the pool sits idle after the first chunk. Break experiment 2 proves it.
4. **The callback runs on the main thread, when the loop gets to it.** The pool can finish in 1 ms and your code still sees the result late.

### `http.get`: the kernel path

```text
 JS thread (main)                      libuv / pool                    kernel
 ────────────────                      ────────────                    ──────
 http.get("http://merchant.example.com/hooks")
   └─ dns.lookup ─────────────────────▶ pool: getaddrinfo() ─────────▶ resolver (can be slow!)
 loop ◀─────────────── address ─────────┘
   └─ non-blocking connect() ───────────────────────────────────────▶ TCP handshake
   └─ register the socket with epoll / kqueue / IOCP
 ... the JS thread is free; the loop sleeps in the poll phase ...
 kernel: socket writable ─▶ loop wakes ─▶ write the request bytes
 kernel: socket readable ─▶ loop wakes ─▶ read ─▶ llhttp parses ─▶ 'response' event in JS
```

**No pool thread is involved in the network part.** The socket is a file descriptor the kernel watches, and one thread can have tens of thousands of them registered. The one exception is the hostname lookup: `dns.lookup` calls the blocking `getaddrinfo`, so it goes to the pool. That detail causes the "slow DNS made file reads slow" incident in [L07](../02-event-loop/07-the-thread-pool.md). The network stages themselves (DNS, TCP, TLS) are covered in [API L02](https://github.com/Pawar-Pratik/API-mastery/blob/main/01-foundations/02-journey-of-a-request.md); this track goes deeper on the runtime side.

The loop's phases (timers, poll, check and the rest) are [Lesson 05](../02-event-loop/05-the-event-loop.md). For now, one fact: **when there's nothing to do, the loop blocks inside epoll/kqueue/IOCP with a timeout**, which is why an idle Node process uses no CPU.

---

## 5. "Single-threaded", precisely

The sentence to say in an interview:

> **"Your JavaScript runs on one thread, the main thread of a V8 isolate, and never runs in parallel with itself. The process has more threads: libuv's pool (4 by default, started lazily), V8's platform threads for concurrent garbage collection and background compilation, and some housekeeping threads. Network I/O is done by the kernel and needs no thread at all."**

You can see it. On Windows with Node 24, a hello-world `node:http` server showed **12 OS threads at idle and 16 right after the first file read**: the four pool threads start on first use. The idle count varies by platform and version; the jump of four doesn't. You'll measure it yourself in Build & break.

Three consequences, each of which comes up again later in the track:

1. **No data races in your JavaScript.** No locks, no `synchronized`, no torn writes. A callback runs to completion before anything else runs.
2. **But single-threaded is not race-free.** Every `await` is a point where other requests run. Read a balance, `await` something, write the balance: two requests can interleave between the read and the write exactly as two Java threads would. The interleaving points are just visible in the code. Ledger avoids this class of bug by guarding state transitions in SQL (`UPDATE … WHERE status = …`), not in JavaScript.
3. **One callback that doesn't yield blocks everything.** There's no preemption. A 5-second loop is 5 seconds of nothing for every other request in the process. That's [Lesson 08](../02-event-loop/08-blocking-the-loop.md)'s whole subject.

"One JavaScript thread *per isolate*" matters because worker threads ([L21](../05-concurrency/21-worker-threads.md)) create more isolates, each with its own thread, heap and event loop. So JavaScript *can* run in parallel inside one process, just never on shared objects by default.

---

## 6. The cost models compared

You know two of these from Spring. The table is the comparison interviewers want you to be able to draw.

| | Thread per request (Tomcat / Spring MVC) | Event loop (Node) | Virtual threads (Java 21+) |
|---|---|---|---|
| **Unit of concurrency** | An OS thread from a pool (Tomcat's default max is 200) | A callback, or a suspended `async` function | A JVM-managed thread, mounted on a few carrier threads |
| **Memory per waiting request** | A full thread stack (~1 MB reserved) | The suspended function's state on the heap | A heap-allocated stack chunk |
| **What waiting costs** | A blocked OS thread | A registered socket and a pending promise | The virtual thread unmounts; the carrier is free |
| **Code style** | Blocking, sequential | `async`/`await` | Blocking, sequential |
| **CPU work in a handler** | Occupies 1 of 200 threads; others carry on | **Stops every request in the process** | Occupies one carrier (≈ one per core); others carry on |
| **Typical failure** | Pool exhaustion: all 200 threads stuck waiting on a slow dependency | Loop blocked: one synchronous call freezes everything | Pinning: a virtual thread that can't unmount holds its carrier |
| **Uses all cores?** | Yes | One core for JavaScript per process (more processes or workers for the rest) | Yes |

Read the last two columns together. **Virtual threads give Java thread-per-request syntax with event-loop economics**: when a virtual thread blocks on a socket, the JDK parks it and registers the socket with the same epoll/kqueue machinery Node uses. The waiting side is the same deal in both. The difference is CPU: the JVM has roughly one carrier per core, and Node has one JavaScript thread per process. **Pinning in Loom is the closest analogue to blocking the loop in Node.** In Java 21, a virtual thread blocked inside `synchronized` (or in native code) can't unmount, so it holds a carrier hostage the way a synchronous call holds Node's only thread. Java 24 removed most `synchronized` pinning; native calls still pin.

> **Spring equivalent:** [Spring L18 — MVC vs WebFlux vs virtual threads](https://github.com/Pawar-Pratik/SpringBoot/blob/main/05-web/18-mvc-vs-webflux-virtual-threads.md). WebFlux runs on Netty's event loops, the same design as Node with one loop per core instead of one per process, and its "one blocking call poisons everything" rule is Node's rule word for word.

---

## 7. Where Node wins, and where it loses

**Node wins when a request mostly waits:**

- **JSON APIs and BFFs.** Parse a small body, call Postgres and Redis, serialise a small response. Almost all the wall-clock time is I/O.
- **Many concurrent, long-lived connections.** SSE and WebSockets ([L20](../04-networking/20-realtime-servers.md)): an idle connection costs memory, not a thread.
- **Streaming.** Moving bytes from a database cursor to a socket with backpressure ([L13](../03-data-and-io/13-streams-and-backpressure.md)).
- **Gateways and proxies.** Little computation, lots of forwarding.
- **Tooling, and one language across the stack.** Ledger's SDK, CLI and React Console share TypeScript types with the API.

**Node loses when a request mostly computes on the main thread:** image and video processing, large cryptographic loops, giant JSON transforms, numeric work, ML inference, matching millions of settlement rows. **Every millisecond of CPU in a handler is a millisecond every other in-flight request waits.**

What you do instead:

| Situation | Do this |
|---|---|
| An occasional heavy step (password hashing, compression) | Use the async API so it runs on the thread pool ([L07](../02-event-loop/07-the-thread-pool.md)) |
| A big synchronous chunk you can't avoid (huge `JSON.parse`) | Bound the input, or move it off the loop ([L08](../02-event-loop/08-blocking-the-loop.md)) |
| Steady CPU work written in JavaScript | A worker-thread pool ([L21](../05-concurrency/21-worker-threads.md)): Ledger's reconciler |
| CPU work some other program already does well | A child process ([L22](../05-concurrency/22-child-processes.md)) |
| Heavy computation is the product | A separate service in a language built for it |
| You need more total throughput | More processes: replicas or `cluster` ([L23](../05-concurrency/23-cluster-and-scaling.md)) |

---

## 8. Node vs Deno vs Bun, by mechanism

| | Node | Deno | Bun |
|---|---|---|---|
| **Engine** | V8 | V8 | JavaScriptCore (Safari's engine) |
| **Runtime written in** | C++, on libuv | Rust, on Tokio | Zig and C++ |
| **TypeScript** | Type stripping, erasable syntax only ([L03](03-the-process.md)) | Built in | Built in |
| **Default permissions** | Full access (an opt-in permission model exists, [L30](../08-shipping/30-security-and-supply-chain.md)) | Deny by default (`--allow-net`, `--allow-read`, …) | Full access |
| **npm compatibility** | It's the reference | High, through `npm:` specifiers and a Node compatibility layer | High; it aims to be a drop-in replacement |
| **Built-in tooling** | Test runner, watch mode; package management is separate | Formatter, linter, test runner | Package manager, bundler, test runner |

The honest summary: **Bun's pitch is speed** (fast startup, a very fast package manager). **Deno's pitch is secure defaults and integrated tooling.** Both are serious projects. Node stays the production default for reasons that have nothing to do with benchmarks: it's the implementation the npm ecosystem is tested against, so "Node-compatible" is the yardstick the others measure themselves by; it has a predictable LTS policy with published dates; and its operational tooling (the inspector, heap snapshots, diagnostic reports, support from every APM vendor and serverless platform) is the deepest. Pick an alternative for a concrete reason you can name, not because of a micro-benchmark.

---

## 9. The release model, and why production runs LTS

| Line | Status (October 2026) |
|---|---|
| **Node 26** | Shipped April 2026; promoted to LTS in late October 2026 |
| **Node 24** | LTS: the track's baseline |
| **Node 22** | Maintenance LTS until 2027-04-30 |
| **Node 20** | End-of-life since 2026-04-30 (Node 18 went in April 2025) |

How the model has worked so far: **even-numbered majors ship in April and become LTS in October**, then move to maintenance and reach end-of-life about three years after release (Node 20 shipped in 2023 and died in April 2026). Odd-numbered majors (like 25) live briefly and never become LTS. **From Node 27 that split ends: one major a year in April, promoted to LTS in October, and every release becomes LTS.**

Why production runs LTS:

- **Security fixes are backported for years.** A "Current" line gets fixes only until it's replaced.
- **Breaking changes arrive only at major versions**, on a published schedule you can plan around.
- **Your dependencies test against LTS lines.** Native addons and APM agents especially.
- **The upgrade becomes a planned project**, done one LTS at a time before maintenance ends, rather than an emergency when a CVE lands on an end-of-life line.

Pin the version in the repo (`.nvmrc`, plus `"engines"` in `package.json`) so laptops, CI and the container image all run the same Node ([L03](03-the-process.md), [L33](../08-shipping/33-shipping-node.md)).

---

## 10. How Ledger Engine maps onto this

Ledger runs as **several Node processes, one per container**, not one big process:

```text
                ┌──────────────────── container ─────────────────────┐
  HTTP ───────▶ │ api: Express + TypeScript                          │  scaled by
                │ payments, statement exports, payout-file ingest,   │  replicas (L23)
                │ the SSE live feed                                  │
                └────────────────────────────────────────────────────┘
  ┌──────────── container ───────────┐  ┌──────────── container ───────────┐
  │ webhook dispatcher               │  │ payout processor                 │
  └──────────────────────────────────┘  └──────────────────────────────────┘
  ┌──────────── container ───────────┐  ┌──────────── container ───────────┐
  │ reconciler + worker-thread pool  │  │ idempotency-key sweeper          │
  └──────────────────────────────────┘  └──────────────────────────────────┘
                 all share Postgres 16 (RLS, outbox) and Redis 7
```

Why separate processes and not one process doing everything:

1. **Each process has its own event loop.** The reconciler's CPU work, or a dispatcher stuck behind a merchant endpoint that takes 30 seconds, can't add a millisecond to an API request.
2. **The process is the unit of failure.** If the payout processor crashes, the supervisor restarts it; the API never notices ([L09](../02-event-loop/09-errors-and-crashing.md), [L24](../05-concurrency/24-graceful-shutdown.md)).
3. **Independent scaling.** The API scales with traffic; the dispatcher scales with webhook backlog.

The API process itself is exactly what this lesson describes: one JavaScript thread multiplexing thousands of sockets to clients, Postgres and Redis, with the thread pool underneath for files, DNS and crypto.

---

## 11. Production rules

| Rule | Why |
|---|---|
| **Never run CPU-heavy work on a server's main thread** | Its duration is added to every in-flight request, not just the one that asked for it |
| **Use the async API in request paths; keep `*Sync` calls for startup and CLIs** | A sync call blocks the only JavaScript thread; at boot nobody is waiting yet |
| **Treat every `await` as a point where other requests run** | Single-threaded isn't race-free; check-then-act across an `await` is a race |
| **Put latency-sensitive and CPU-heavy or batch work in different processes** | Each process has its own loop, so one can't stall the other |
| **Count capacity in sockets, memory and downstream pool slots, not threads** | Node's concurrency ceiling is file descriptors, heap and the database pool ([L25](../06-data-and-jobs/25-databases-from-node.md)) |
| **Run an LTS line in production; pin it with `.nvmrc` and `engines`** | Backported security fixes, scheduled breaking changes, and the same Node everywhere |
| **Upgrade one LTS at a time, before maintenance ends** | An end-of-life runtime gets no CVE fixes, and skipping lines stacks breaking changes |
| **Choose Deno or Bun for a stated reason, not a benchmark** | Compatibility and operational tooling decide production outcomes more than raw speed |

---

## 12. Interview traps

**Q1. "Is Node single-threaded?"**
The weak answers are "yes" and "no, it uses a thread pool for everything". The strong answer: your JavaScript runs on one thread per V8 isolate and never in parallel with itself, but the process has libuv's pool (4 threads by default) for file system work, `dns.lookup`, async crypto and zlib, plus V8's threads for GC and compilation. Network I/O uses no thread at all: sockets are registered with epoll/kqueue/IOCP and the kernel reports readiness.
**Follow-up:** "So can two requests' handlers run at the same time?" Never simultaneously in one isolate. They interleave at `await` and callback boundaries. True parallel JavaScript needs worker threads or more processes.

**Q2. "How does Node handle 10,000 connections with one thread?"**
Each connection is a non-blocking socket registered with the kernel's readiness API. The loop sleeps in the poll phase until the kernel reports which sockets are readable or writable, runs the callbacks for just those, and goes back to sleep. A connection costs some kernel buffers and kilobytes of JavaScript objects, not a thread and its stack.
**Follow-up:** "Then what's the limit?" File descriptors (`ulimit -n`), memory per connection, CPU time per message, and the downstream pools the requests end up waiting on.

**Q3. "Node vs Java virtual threads?"**
Same economics for waiting: both park the waiting computation on the heap and let the kernel watch the socket. Virtual threads keep blocking-style code and spread CPU work across all cores via carrier threads; Node needs `async`/`await` and runs all JavaScript on one thread, so CPU work in a handler stalls everyone. Pinning in Java 21 is the analogue of blocking Node's loop, except that it costs one carrier out of several rather than the whole process.

**Q4. "When would you *not* choose Node?"**
When the core workload is CPU-bound in-process: media processing, heavy numeric or ML work, large in-memory computation per request. Weak answer: "Node is slow." V8's JIT is fast; the problem is that CPU time on the main thread is shared by every request. Then give the mitigations: worker threads, child processes, or a separate service for the heavy part, with Node keeping the I/O-heavy edge.

**Q5. "What is libuv?"**
The C library underneath Node that provides the event loop and cross-platform asynchronous I/O over epoll, kqueue and IOCP, plus timers, signals, child processes and a thread pool for work the OS can't do asynchronously (files, `getaddrinfo`, and the crypto and zlib work Node offloads). The weak answer is "Node's thread pool", which describes only one of its jobs.

**Q6. "Is Node a language or a framework?"**
Neither: a runtime. V8 runs the JavaScript, libuv does the I/O, and Node adds a standard library and C++ bindings. Express is a framework running on it. The Java analogy is JVM plus JDK, versus Spring.

**Q7. "Why is an `await`ed database call not blocking?"**
`await` suspends the async function, saving its state on the heap, and returns control to the loop. The driver has already written the query to a socket registered with epoll, so the thread goes on serving other requests. When the reply's bytes arrive, the poll phase wakes, the driver parses the reply and resolves the promise, and your function resumes as a microtask. "Blocking" would be a driver waiting inside a syscall on the main thread.
**Follow-up:** "So what *does* block?" CPU work: `*Sync` APIs, huge `JSON.parse`, catastrophic regexes, long loops ([L08](../02-event-loop/08-blocking-the-loop.md)).

**Q8. "Node is non-blocking, so a slow database can't hurt the API, right?"**
Wrong. The *thread* isn't blocked, but requests still pile up: each waiting request holds memory, a socket, and a place in the pool's queue. `pg`'s pool defaults to 10 clients and waits forever for a free one, so a slow database turns into an unbounded queue in your process ([L25](../06-data-and-jobs/25-databases-from-node.md)). Non-blocking moves where the waiting happens; it doesn't make waiting free.

**Q9. "Your Node service runs on an 8-core machine. How much of it does it use?"**
One core for JavaScript per process; the pool threads, GC and the kernel use some of the others. To use all eight you run more processes (container replicas, or `cluster`), and move CPU-heavy work to worker threads ([L23](../05-concurrency/23-cluster-and-scaling.md)). Saying "Node can't use multiple cores" is the weak version; it can, just not from one JavaScript thread.

**Q10. "Bun benchmarks faster. Should we switch?"**
Ask what the bottleneck is first: most API latency is database and network time, which no runtime changes. Then weigh compatibility with your dependencies and native addons, LTS and security-patch policy, and operational tooling (profilers, heap snapshots, APM support). Switch for a measured, specific win (CLI startup time, say), not a headline number.

---

## 13. Build & break

### Build — a hello-world server, and count its threads

Create `scratch/hello.ts`:

```ts
import { createServer } from "node:http";
import { readFile } from "node:fs/promises";

const server = createServer(async (req, res) => {
  if (req.url === "/file") {
    const t0 = performance.now();
    const data = await readFile(import.meta.filename);       // goes through the pool
    res.end(`read ${data.length} bytes in ${(performance.now() - t0).toFixed(1)} ms\n`);
    return;
  }
  res.end("ok\n");
});

server.listen(3000, () => console.log(`pid ${process.pid} listening on :3000`));
```

Run it with `node scratch/hello.ts`, then count the process's OS threads:

- **Windows:** Process Explorer → the process → Properties → Threads tab. Or in PowerShell: `(Get-Process -Id <pid>).Threads.Count`.
- **WSL2:** `ls /proc/<pid>/task | wc -l`.

Count once at idle. Then run `curl.exe localhost:3000/file` and count again: **four more threads**, the libuv pool starting on first use. Write both numbers down; they're your answer to "is Node single-threaded?"

### Break — three experiments

1. **Block the loop.** Add a route that busy-waits:
   ```ts
   if (req.url === "/block") {
     const end = Date.now() + 5_000;
     while (Date.now() < end) {}                 // ❌ 5 s of CPU on the only JS thread
     res.end("done blocking\n");
     return;
   }
   ```
   In one terminal run `curl.exe localhost:3000/block`; immediately, in a second terminal, run `time curl.exe localhost:3000/`. **The trivial route takes about 5 seconds.** The kernel accepted the connection (it sits in the accept queue), but the JavaScript that would answer it can't run. Every route freezes, not just `/block`. The fix is never "a faster loop": it's moving CPU work off the main thread ([L08](../02-event-loop/08-blocking-the-loop.md), [L21](../05-concurrency/21-worker-threads.md)).

2. **A file read while the loop is busy.** Create a large file (`node -e "require('node:fs').writeFileSync('big.bin', Buffer.alloc(300 * 1024 * 1024))"`), then add:
   ```ts
   if (req.url === "/file-busy") {
     const t0 = performance.now();
     const pending = readFile("big.bin");        // the first chunk is queued on the pool now
     const end = Date.now() + 2_000;
     while (Date.now() < end) {}                 // the JS thread is busy for 2 s
     await pending;
     res.end(`took ${(performance.now() - t0).toFixed(0)} ms\n`);
     return;
   }
   ```
   First time a plain `readFile("big.bin")` on its own, then hit `/file-busy`. **The total is about 2 s plus the read time, not the larger of the two.** The pool finished the first chunk almost immediately, but each next chunk is scheduled from the loop, and the loop was busy. Blocking the loop doesn't just delay your callbacks; it starves the thread pool of work too.

3. **The same thing in Spring.** In a scratch Spring Boot app (MVC, platform threads), add:
   ```java
   @GetMapping("/block") String block() throws InterruptedException { Thread.sleep(5000); return "done"; }
   @GetMapping("/")      String ok()    { return "ok"; }
   ```
   Repeat experiment 1. **Only the `/block` request waits; `/` answers instantly**, because Tomcat served it on one of its other 199 threads. Same mistake, two blast radii: one request in Tomcat, every request in Node.

### Explain out loud (60–120 seconds)

1. The four layers inside `node`, and what V8 and libuv each do and don't know about.
2. The `fs.readFile` path versus the `http.get` path, and why one uses the pool and the other doesn't.
3. The precise answer to "is Node single-threaded?", with the thread counts you measured.
4. Thread-per-request versus event loop versus virtual threads: what waiting costs and what CPU work costs in each.
5. Where Node wins, where it loses, and what you do about CPU work.
6. Why production runs LTS, and how the release model changes from Node 27.

---

## What's next

You know which thread runs what. Next, the JavaScript itself, but only the parts that bite on a server: references shared between requests, closures that keep 50 MB alive, `this` lost inside a callback, and why `pg` hands you your `bigint` money column as a string.

Next → **[Lesson 02: The JavaScript Node assumes](02-the-javascript-node-assumes.md)**
