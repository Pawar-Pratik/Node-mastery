# Lesson 38 — Production scenarios & Node system design

> **Why this lesson exists:** the senior Node round asks two things the rest of the track prepared you for but never rehearsed: *"it's 3am, what do you do?"* and *"design this service, with numbers."* Both reward a repeatable shape far more than recall. This lesson gives you 35 Node-specific scenarios answered in that shape, a design playbook built on capacity arithmetic, three worked designs, and the meta-answers that tie them together.

**Time:** ~2.5 hours, then practise · **Prereq:** Lessons 01–37

---

## 1. The answer shape, with Node's instruments

The shape is the API track's, and it doesn't change for Node ([API L32](https://github.com/Pawar-Pratik/API-mastery/blob/main/08-interview/32-scenario-gauntlet.md)):

```text
① CONTAIN    Stop the bleeding before you understand it (roll back, drain, shed, disable)
② DIAGNOSE   Narrow it with telemetry: which instance, which endpoint, since when
③ FIX        The smallest change that resolves it
④ VERIFY     Name the metric that proves it
⑤ PREVENT    The structural fix + the regression test + the alert that would have caught it
```

What's Node-specific is **②: which number you look at first.** Juniors say "I'd check the logs." Seniors name the metric that splits the hypothesis space in half:

| Symptom | First number | Then | Lessons |
|---|---|---|---|
| Everything slow, CPU low | **Event-loop delay p99 and ELU** | A CPU profile under load | [L08](../02-event-loop/08-blocking-the-loop.md), [L28](../07-production/28-performance-and-v8.md) |
| Slow, but ELU low | **Pool waiting count**, libuv pool queueing, downstream latency | A trace of one slow request | [L07](../02-event-loop/07-the-thread-pool.md), [L25](../06-data-and-jobs/25-databases-from-node.md) |
| Memory climbing | **`rss` vs `heapUsed` vs `external`** | Three heap snapshots, or the off-heap suspects | [L27](../07-production/27-memory-and-gc.md) |
| Restarts | **Exit code and reason** (137 + OOMKilled, 1, 143) | The last log lines, `kubectl describe pod` | [L03](../01-foundations/03-the-process.md), [L09](../02-event-loop/09-errors-and-crashing.md) |
| Hangs, nothing errors | **Pool waiting, open handles, oldest job age** | A diagnostic report (`process.report.writeReport()`) | [L25](../06-data-and-jobs/25-databases-from-node.md), [L29](../07-production/29-observability.md) |
| 502 / `ECONNRESET` | **Which hop logged it** (LB? app? neither?) | Timeouts per hop, in order | [L17](../04-networking/17-http-from-scratch.md), [L18](../04-networking/18-outbound-http.md) |

> **The habit that makes these answers land:** say the number *and what each value would mean*. *"If loop delay p99 is high, it's our own synchronous work; if it's flat and the pool's waiting count is up, we're waiting on Postgres."* One sentence, two branches, no guessing.

---

## 2. Latency & the event loop (6)

**1. "p99 tripled overnight. CPU is at 30%."**
② Loop delay p99 and ELU per instance. A blocked loop pegs one core, which reads as ~30% on a multi-core host. If delay is high: take a CPU profile under load and read the widest self-time plateau, usually a large `JSON.stringify`, a `*Sync` call, a backtracking regex, or a new O(n²) over request data. If delay is flat, it isn't the loop: check the pool's waiting count and downstream latency. ① Contain by rolling back if it lines up with a deploy; shed load on loop delay so healthy requests still get through. ⑤ Alert on loop-delay p99 and ELU, not CPU%. [L08, L28]

**2. "One reporting endpoint got slow, and now every endpoint is slow."**
The signature of **shared-thread contention**: the report does CPU work on the loop (serializing 50k rows, sorting in memory), and every other request queues behind each call. ① Rate-limit or disable that endpoint. ③ Cap its page size, stream the output, or move the computation to a worker pool. ④ Loop-delay p99 back to baseline *while the report runs*. ⑤ A load test that runs the report alongside normal traffic; Spring engineers expect one slow endpoint to cost one thread, and in Node it costs everyone. [L08, L21]

**3. "File uploads and password checks slowed down at the same time DNS got slow."**
They share **the libuv pool**: `getaddrinfo` for every new outbound connection held all four threads, so `fs` and async crypto queued behind it. ② Correlate DNS latency with fs and scrypt latency; ELU stays low because the JS thread is idle, waiting. ③ Reuse connections (fewer lookups), cache resolutions, raise `UV_THREADPOOL_SIZE` at launch. ⑤ A metric for DNS lookup time, and a pool-pressure test in CI. [L07]

**4. "Latency spikes every few minutes, then recovers."**
Find the period. Line it up with GC (`--trace-gc` or perf_hooks `'gc'` entries): long major GCs on a large old space pause the thread. Or a `setInterval` doing synchronous work (a cache rebuild, a metrics flush), or a cron that runs on every replica. ③ For GC: allocate less in hot paths, shrink the heap the process keeps alive, consider `--max-semi-space-size` if young-generation collections dominate. ④ Spikes gone from the loop-delay max, not just the average. [L27, L08]

**5. "Latency creeps up for hours and resets on every deploy."**
Something grows per request and makes each request slower: a leak adding GC pressure, or a structure that's scanned per request. Netflix's "Node.js in Flames" (2014) is the canonical case: flame graphs showed Express route lookups slowing because a reload bug kept appending to the route array. ② Two CPU profiles hours apart; compare the plateaus. Heap usage trend. ⑤ A soak test: steady load for hours with p99 and heap plotted. [L28, L27]

**6. "Throughput plateaus at 600 rps no matter how many replicas we add."**
If replicas don't help, the bottleneck is **shared**: the database (pool sizes summed across replicas against `max_connections`, lock contention), a single Redis, a downstream's rate limit, or one partition everyone writes to. ② ELU per replica is low; pool waiting is high; the DB's active sessions are maxed. Adding replicas here *adds* contention. ③ Fix the shared resource: query plans, fewer round trips, a cache, smaller pools with PgBouncer. [L25, L23]

---

## 3. Memory (4)

**7. "Memory grows 50 MB an hour."**
② Split it: if `heapUsed` trends up, it's the JS heap; if `rss` and `external` grow with a flat heap, it's Buffers or native memory. For the heap: take three snapshots (baseline, after load, after more load) on an instance drained from traffic, compare objects allocated between the first two that survive into the third, and read their retaining paths. The usual answer is an unbounded Map, a listener added per request, an interval per connection never cleared, or a closure holding a request. ① Meanwhile, a scheduled restart buys time; it isn't the fix. ⑤ A memory metric with a slope alert and an LRU with max + TTL everywhere you cache. [L27]

**8. "Pods are OOMKilled (exit 137) but there's never a 'heap out of memory' error."**
The kernel killed the process because **RSS crossed the cgroup limit before V8 reached its heap limit**. ② `rss` vs `heapUsed` vs `external` from your metrics: a large gap means Buffers (unbounded body buffering, retained slices), native addons, or `--max-old-space-size` set above the container limit so V8 never feels pressure. ③ Set the heap to ~75% of the limit; bound the off-heap source. ④ RSS stays flat under the same load. [L27, L12, L33]

**9. "Memory spikes to 3 GB whenever a merchant exports their statement."**
The export **buffers**: `rows = await query(all)` then `rows.map(toCsv).join("\n")`, a 4M-row result set plus a 300 MB string plus its gzip. ① Cap concurrent exports. ③ Stream it: a cursor → CSV Transform → gzip → response in one `pipeline`, aborted when the client disconnects. ④ RSS flat during a 4M-row export. ⑤ A CI test that exports a large fixture under a memory ceiling. [L13]

**10. "The process crashes with 'Reached heap limit' on one specific request."**
Unbounded input: a 200 MB JSON body with no limit, `?limit=500000`, or a bulk endpoint with no item cap. ③ Body-size limits enforced while streaming, page-size caps, item caps. ⑤ Every input has a documented maximum, and `--heapsnapshot-near-heap-limit` captures what was on the heap next time. [L08, L17]

---

## 4. Crashes & restarts (4)

**11. "The process exits with no log line at all."**
② The exit code tells you which kind of silence. **137** with reason OOMKilled: the kernel, so the app never got to log. **1**: an uncaught exception or unhandled rejection, whose log was lost because the logger's asynchronous buffer wasn't flushed before exit. **143**: SIGTERM with nobody handling it. ③ Fatal handlers that log synchronously or flush the logger before exiting, with a timeout. ⑤ Alert on restarts by exit code. [L03, L09, L29]

**12. "Every pod goes into CrashLoopBackOff after a deploy."**
① Roll back first. ② The first log lines of a crashing pod. The good case: config validation failing at boot with a readable error, exactly as designed. The others: an ESM/CJS mismatch (`ERR_REQUIRE_ESM`, a missing `.ts` extension, a deep import blocked by `exports`), a native addon built against glibc running on alpine, or a liveness probe killing a slow-starting process (use a startup probe). ⑤ Boot the container in CI and hit `/healthz` before the image can ship. [L03, L04, L33]

**13. "The process crashed when the database restarted, from a library we don't call."**
An **`'error'` event with no listener**. `pg.Pool` emits `'error'` when an idle client's connection dies; with no `pool.on("error")` listener, the emit throws and takes the process down. ③ Listen for `'error'` on every long-lived emitter you own (pools, sockets, streams, child processes), log it, and let the pool replace the client. ⑤ A chaos drill: restart Postgres under load and assert zero process restarts. [L09, L11, L25]

**14. "Someone added an `uncaughtException` handler that logs and continues. Now we serve wrong balances."**
The throw unwound through code that had **half-applied an invariant**: a cache entry updated, the matching DB write never issued. Continuing serves corrupted state. ① Remove the swallow; ③ the handler logs with context, flushes, sets `exitCode = 1` and exits after a timeout; the supervisor restarts a clean process. ⑤ Crash-only design: idempotent operations and transactions make the restart safe, and a lint rule or review checklist bans catch-and-continue at process level. [L09]

---

## 5. Networking (5)

**15. "Intermittent 502s from the ALB, about 1 in 500, never in our app logs."**
"Not in our logs" means the request never reached a handler. **The ALB's idle timeout (60 s) is longer than Node's `keepAliveTimeout` (5 s)**, so Node closes an idle connection just as the ALB sends a request on it. ③ `keepAliveTimeout` above the LB's idle timeout (e.g. 65 s), `headersTimeout` above that. ④ 502 rate to zero on quiet endpoints, which were the ones with idle connections. [L17]

**16. "`ECONNRESET` spikes on calls to an internal service."**
The client-side mirror of #15: **our agent keeps idle sockets longer than the server does**, so we reuse a socket the server already closed. ② Errors cluster on the first request after an idle gap. ③ Set the client's idle socket timeout below the server's keep-alive timeout, and retry idempotent requests once on a reset of a reused socket. ⑤ Document keep-alive timeouts per hop as a chain: each client shorter than its server. [L18, L16]

**17. "Outbound calls start failing with `EADDRNOTAVAIL`."**
Ephemeral ports are exhausted by sockets in **`TIME_WAIT`**: something creates a new Agent per request, or disabled keep-alive, so every call opens and closes a connection. ② `ss -s` / `netstat` shows tens of thousands in `TIME_WAIT`. ③ One shared dispatcher with keep-alive and per-origin connection caps. ⑤ A connection-count metric per origin. [L18]

**18. "Webhooks to one merchant time out, and now deliveries to everyone are delayed."**
One slow endpoint is **consuming the dispatcher's global concurrency**: every attempt holds a slot for the full 10 s timeout. ① Per-endpoint concurrency cap (5) so the slow merchant queues against itself; open a per-endpoint breaker and back off. ② Their latency and status distribution; also rule out our side (DNS, the SSRF guard's lookup). ⑤ After the final attempt the endpoint is auto-disabled and the merchant notified; deliveries stay in the event log for replay. [L10, L18, L26]

**19. "Outbound requests start hanging forever; restarts fix it for an hour."**
Two Node classics, often together: **no total deadline** on the call, and **response bodies not consumed** on error paths, so undici holds each connection until the pool is exhausted and new requests queue in the dispatcher. ③ `AbortSignal.timeout` on every call; read or `cancel()` every body, including 4xx/5xx. ④ Pool connection counts stay below the cap; no request older than its deadline. [L18]

---

## 6. Deploys & lifecycle (4)

**20. "Every rolling deploy drops a few dozen requests."**
② Which ones: connection resets mid-request (we exited while serving), or 502/503 right after SIGTERM (the LB was still routing to us). Endpoint removal propagates **asynchronously**, so traffic arrives for seconds after SIGTERM. ③ The ordered shutdown: fail readiness, keep serving a few seconds (a preStop sleep), `server.close()`, close idle keep-alive, finish in-flight under a deadline, close pools, flush, exit before the grace period. ⑤ A CI test that runs load during SIGTERM and asserts zero failures. [L24]

**21. "Pods take exactly 30 seconds to terminate."**
Exactly the grace period means **nobody handled SIGTERM**, and SIGKILL ended it. Usually Node as PID 1 with no handler (the kernel applies no default action to PID 1), or `npm start` as PID 1 not forwarding the signal. ③ Exec-form `CMD ["node", "main.ts"]`, a SIGTERM handler, `--init`/tini. ④ Termination in seconds, exit code 0, not 137. [L24, L33]

**22. "The nightly settlement job ran three times."**
Three replicas, each with an in-process scheduler. ① Make the job idempotent if it isn't (it must be anyway). ③ Leader election with `pg_try_advisory_lock`, or move scheduling to a Kubernetes CronJob. ⑤ A job-run table with a unique key per (job, date), so a duplicate run is rejected by the database rather than by discipline. [L23, L26]

**23. "After scaling from one replica to four, rate limiting became four times too lenient."**
The token buckets live **in process memory**, one set per replica. The same class of bug as #22: in-memory caches, sessions and WebSocket subscriptions all break at N. ③ Shared state in Redis with an atomic Lua script, or per-instance limits at `global / N` if approximation is acceptable. ⑤ Run the test suite against two instances behind a proxy at least once. [L23]

---

## 7. Data & jobs (4)

**24. "The app hangs. The database is idle."**
**Pool exhaustion**: every client is checked out and nobody returns them, and the default `connectionTimeoutMillis: 0` makes new requests wait forever. ② The pool's waiting count is high, total = max, the DB shows sessions `idle` or `idle in transaction`. The causes: `release()` missing on an error path, or a client held across a slow `await` (an HTTP call inside a transaction). ① Restart buys an hour. ③ `withTransaction` with `release()` in `finally`, no outbound calls inside transactions, a connection timeout. ⑤ Alert on pool waiting > 0 for a minute. [L25]

**25. "Merchant A saw one of merchant B's payments."**
① Treat it as a breach: contain, preserve logs, involve security. ② The Node-specific suspect: `SET app.merchant_id` (session-scoped) instead of `SET LOCAL` inside the transaction, so a pooled connection carried B's tenant into A's next request, and RLS trusted it. Or tenancy read from module-level state instead of the request's `AsyncLocalStorage` store. ③ `SET LOCAL` inside `withTenantTx`. ⑤ A test that interleaves two tenants on a pool of size 1 and asserts isolation. [L25, L29]

**26. "Merchants report duplicate webhooks for the same event."**
At-least-once delivery working as designed: a dispatcher was killed after sending and before recording the delivery, or **a slow send outlived its lease** and another worker reclaimed it. ② Check for lease expiries and restarts at those timestamps. ③ Leases longer than the send timeout plus margin, heartbeats for long work. ⑤ Make sure merchants dedupe on the event ID, which is in your docs and in the SDK's `webhooks.verify()` helpers. Exactly-once delivery doesn't exist; say so. [L26]

**27. "The job backlog grows, but the workers look idle."**
② Oldest-job age is climbing while ELU is near zero: workers aren't claiming. Suspects: a claim query without `SKIP LOCKED` waiting on locks; jobs stuck `running` whose lease never expires after a crash; one poison message at the head of a strictly ordered queue; or the claim query doing a sequential scan because the partial index doesn't match its `WHERE`. ③ Fix the claim, add lease expiry and a DLQ after N attempts. ⑤ Alert on oldest-job age, not depth. [L26]

---

## 8. Streams (3)

**28. "The export sometimes never finishes: the download stalls forever at some percentage."**
A stream in the chain is waiting on something that will never happen: a Transform that doesn't call its callback on some branch, a `flush` that never calls `cb`, a stage connected with `.pipe()` whose source errored (pipe doesn't propagate it, so the destination waits forever), or a cursor waiting for a pool client. ② Log `finished()` per stage and the buffered lengths (`readableLength`, `writableLength`): the stage with a full input and an empty output is the culprit. ③ `pipeline()` everywhere, every code path calls back. ⑤ A test that injects an error in each stage and asserts the pipeline rejects. [L13, L14]

**29. "Uploads of large payout files make the ingester's memory climb until it dies."**
Something upstream ignores backpressure: a custom loop calling `write()` without honouring `false`, a generator stage that collects rows into an array "for batching" without ever yielding, or the body read with `await req.arrayBuffer()`. ③ One `pipeline` from the socket to the database, with batching that yields every N rows. ④ RSS flat while a 2 GB file ingests. [L13, L14]

**30. "The ingester reports fewer rows than the file has, and a few names come out garbled."**
Chunk boundaries. **Lines split across chunks** are dropped by a naive `chunk.toString().split("\n")`, and **multi-byte characters split across chunks** (a "₹" or an accented name) decode as replacement characters. ③ A carry-over line splitter with a `StringDecoder`, or a real streaming CSV parser. ⑤ A test that feeds the same file in 1-byte, 7-byte and 64 KiB chunks and asserts identical output. [L12, L14]

---

## 9. Concurrency & scale (2)

**31. "Whenever reconciliation runs, API p99 doubles."**
② Is the reconciler in the API process? Then its CPU work is on the API's loop (loop delay spikes during the run). If it's a worker pool in the same pod, check CPU throttling: a pool sized from `os.cpus()` (host cores) inside a 2-CPU limit burns the quota and **CFS throttling stalls the API thread too**. ③ A separate deployment for the reconciler, its pool sized from config. ④ API loop-delay p99 unchanged during a run: that's the evidence in the blueprint. [L21, L23]

**32. "We moved image processing to worker threads and throughput got worse."**
Either workers are created per task (isolate startup and memory per call), or the data crossing the boundary is large and **structured clone runs on the main thread**, so you moved the work and kept the cost. ③ A fixed pool; transfer ArrayBuffers with a `transferList` instead of cloning; send references (a file path, an object-store key) instead of payloads. ④ Measure ELU on the main thread before and after. [L21]

---

## 10. Supply chain & security (3)

**33. "A dependency we use was compromised on npm overnight."**
① Freeze deploys and dependency updates. ② Is the malicious version in any lockfile, and did any CI job or developer machine install it? Install scripts are the usual payload (the Shai-Hulud worm of September 2025 spread through them). ③ Pin to a known-good version, rebuild from a clean lock, and **rotate every secret the install step could read**: CI tokens, npm tokens, cloud credentials. ⑤ `npm ci --ignore-scripts` with an allowlist (or pnpm 10's default), a release-age delay before adopting new versions, provenance checks, no secrets in the install step's environment, and the permission model on processes that don't need fs or child processes. [L30]

**34. "A security report says you're vulnerable to prototype pollution in the metadata merge."**
① Verify with the reported payload (`{"__proto__": {"isAdmin": true}}`) against a non-production instance, then patch immediately because it can flip authorization checks process-wide. ③ Validate the body with a schema that strips unknown keys, merge into `Object.create(null)` or a `Map`, reject `__proto__`/`constructor`/`prototype` keys in the merge. ⑤ `--disable-proto=delete`, pollution tests in CI, and an audit of every other deep-merge and query parser. [L30]

**35. "A pentest found that the thumbnail endpoint runs ``exec(`convert ${file}`)``."**
That's remote code execution: a filename containing `; curl … | sh` runs as a command. ① Disable the endpoint or deploy the fix now, then check logs for exploitation (unusual filenames, outbound connections from the pod). ③ `execFile`/`spawn` with an args array and no shell, an allowlist for the value, `--` before user arguments, a generated storage name instead of the user's filename. ⑤ A lint rule banning `exec` with template strings, and egress restrictions so a shell that does get through can't phone home. [L22]

---

## 11. The Node system design round

API design rounds ask for a contract ([API L31](https://github.com/Pawar-Pratik/API-mastery/blob/main/08-interview/31-design-round-playbook.md)). A Node system design round asks for **processes, budgets and failure modes**: how many, how big, what saturates first, and what happens when it does. The differentiator is arithmetic, said out loud.

```text
 0–5   min  ① CLARIFY     Rates (avg and peak), sizes, latency targets, durability, fan-out, tenants
 5–12  min  ② TOPOLOGY    Which processes, what each owns, where CPU work runs, what's shared
12–25  min  ③ BUDGETS     The arithmetic: concurrency, CPU, memory, connections, pool, descriptors
25–35  min  ④ FAILURE     Slow dependency, crash mid-work, deploy, overload, noisy tenant
35–42  min  ⑤ OPERATE     SLOs, Node alerts, shutdown order, rollout, what I'd watch first
42–45  min     Questions for them
```

### The budget sheet

Fill it in on the whiteboard for every process type. **Every number is either given, assumed out loud, or derived.**

```text
Throughput        λ = ___ /s at peak                        (given or assumed)
Latency           W = ___ s mean, ___ s p99, timeout ___ s  (assumed; measure later)
Concurrency       L = λ × W = ___ in flight                 (Little's law)
CPU               c = ___ ms of JS per unit → one process ≈ 1000/c units/s at ELU 1.0
                  plan at ELU ≈ 0.6 → processes = λ / (600/c), plus headroom
Memory            per in-flight unit × L/processes + baseline → container limit
                  --max-old-space-size ≈ 75% of the limit (less if off-heap heavy)
DB connections    Σ (replicas × pool max) ≤ max_connections − headroom
libuv pool        which work hits it (fs, DNS, crypto, zlib)? lookups/s × lookup time ≤ threads
Descriptors       sockets + files per process ≤ ulimit -n
```

Three rules for the numbers:
- **Say "assume" and "I'd measure this".** *"I'll assume 1 ms of CPU per delivery; the first thing I'd do is profile one under load."* Invented precision is worse than a stated assumption.
- **Plan at ELU ~0.6, not 1.0.** Queueing delay explodes as utilization approaches 1, and you need headroom to absorb a node loss and a deploy.
- **Find the shared resource.** The bottleneck in a well-designed Node fleet is almost never Node: it's the database, a broker, a downstream, or the libuv pool.

---

## 12. Worked design A — Webhook delivery at 10k events/s

**① Clarify (assumptions said out loud):** 10k events/s at peak (3k average); each event matches 1.2 endpoints on average → **12k deliveries/s**; ~2% fail and retry → **~12.2k attempts/s**. Receiver latency p50 150 ms, p99 3 s, mean ~300 ms; 10 s timeout; 8 attempts over ~3 days ([API L23](https://github.com/Pawar-Pratik/API-mastery/blob/main/05-beyond-rest/23-realtime-and-webhooks.md)). At-least-once, HMAC-signed, no ordering guarantee.

**② Topology:**
```text
 API ──tx──► outbox ──► outbox-relay ──► delivery queue ──► dispatcher × N ──► merchant endpoints
                                              ▲                    │
                                              └── retries ◄────────┘ (next_attempt_at)
                                                                   └──► delivery log (batched inserts)
```

**③ Budgets:**
- **Concurrency:** 12.2k/s × 0.3 s ≈ **3,700 in flight**, plus timeouts: if 0.5% of attempts hit the 10 s timeout, 61/s × 10 s ≈ **600 more**. Budget ~4,500.
- **CPU:** assume ~1 ms of JS per delivery (HTTP client, signing, serialization, logging) → ~1,000/s per core at ELU 1.0, **~600/s at 0.6**. 12.2k ÷ 600 ≈ 21 processes; with headroom for a lost node and a rolling deploy, **24 dispatcher pods × 1 vCPU**.
- **Per process:** 4,500 ÷ 24 ≈ 190 in flight → a global cap of **250**. Memory: 250 × ~50 KB per in-flight request ≈ 12.5 MB, plus idle keep-alive sockets: a 512 MB limit with `--max-old-space-size=384` is generous.
- **Per-endpoint cap of 5** must hold across 24 pods. A per-process cap would allow 120 to one merchant. So **partition endpoints**: hash `endpoint_id` into 64 partitions leased to pods; each pod only claims its partitions, and the per-endpoint cap is local and exact.
- **DB connections:** dispatchers 24 × 3 = 72, API 8 × 10 = 80, other workers ~20 → **172 against Postgres's default `max_connections` of 100**. Either PgBouncer in transaction mode (claims are short transactions, so it fits) or a deliberate `max_connections` increase with its memory cost. Say which, and why.
- **The libuv pool (the Node-specific trap):** long-tail endpoints get one event a minute, so their idle sockets expire and ~30% of deliveries open a new connection → ~3,700 lookups/s cluster-wide, **~150/s per pod**. Four pool threads at 20 ms per `getaddrinfo` handle 200/s; a resolver slowing to 200 ms handles 20/s, and the pool saturates. The SSRF guard already resolves every hostname, so **cache the validated IPs** (respecting TTLs, clamped), resolve with c-ares (`dns.resolve4/6`, off the pool), and set `UV_THREADPOOL_SIZE=16` at launch.

**④ Failure modes:**
- *A merchant endpoint hangs:* holds at most 5 slots for 10 s each; a per-endpoint breaker backs it off; after 8 attempts → `dead`, auto-disable, notify.
- *A dispatcher dies mid-send:* the claim's lease expires and another pod redelivers. Receivers dedupe on the event ID.
- *A big merchant comes back after 6 hours:* its backlog drains at 5 concurrent, jittered, without starving anyone else.
- *Deploy:* stop claiming, finish in-flight (bounded by the 10 s timeout), release leases, exit; `terminationGracePeriodSeconds: 30`.

**What breaks first:** at a sustained 12k state changes/s, **the Postgres queue table's churn**: every attempt is an UPDATE, i.e. a dead tuple, so vacuum pressure and bloat on the `next_attempt_at` partial index grow without bound. Watch oldest-job age and vacuum lag. At this scale I'd put the hot queue on a broker partitioned by endpoint from day one, keeping Postgres for the outbox and an append-only, batched delivery log partitioned by day. **Naming the first thing to break is the senior signal.**

---

## 13. Worked design B — Multi-GB file processing

**① Clarify:** merchants upload settlement CSVs up to 5 GB; 200 files a day averaging 500 MB; ~200 bytes a row → up to **25M rows per file**, ~500M rows/day (~5.8k rows/s on average). Each row validated and inserted; a rejects report per file; the largest file processed within 30 minutes. Uploads must be resumable.

**② Topology:**
```text
 client ──presigned multipart PUT──► object storage ──"complete"──► API ──enqueue──► ingest_jobs
                                                                                       │
              ingest worker × 4:  GET (ranged) → gunzip? → CSV parse → validate → batch(1000) → COPY → staging
                                                     └──► rejects.csv (multipart upload) ──► merge into target
```
The 5 GB never passes through the API process: **pre-signed uploads** take bandwidth, memory and request timeouts off the Node tier entirely ([API L31](https://github.com/Pawar-Pratik/API-mastery/blob/main/08-interview/31-design-round-playbook.md)). If the brief insists on uploading through the API, it's the [Lesson 14](../03-data-and-io/14-composing-streams.md) pipeline straight from the request socket, with a byte limit.

**③ Budgets:**
- **CPU:** assume parse + validate at ~30 MB/s per core (measure it) → 5 GB ≈ **170 s**.
- **Database:** assume `COPY` into an unindexed staging table at ~50k rows/s per connection → 25M rows ≈ **500 s**. The DB is the bottleneck, and that's fine: `pipeline` backpressure throttles the object-storage read to the database's pace automatically. Total for the largest file ≈ 9–10 minutes: inside 30.
- **Concurrency:** 20 concurrent files × 50k rows/s = 1M rows/s of writes against the OLTP database: no. **Cap concurrent ingest jobs at 4** (~200k rows/s peak) and let the rest queue, with an oldest-job-age SLO.
- **Memory per job:** byte streams buffer to `highWaterMark` (64 KiB on Linux, 16 KiB on Windows), object mode to 16 records, a batch of 1,000 rows as objects ≈ 1 MB. Tens of MB per job: **1 job per pod, 1 vCPU, 512 MB limit**, heap capped at 384 MB.
- **The loop:** the worker's loop also renews the job lease and answers health probes. Parse stages yield at every chunk, so loop delay stays low; if heartbeats ever slip, move parsing into a worker thread and keep the main thread for I/O.

**④ Failure modes:**
- *Worker crashes at row 18M:* each batch commits with a checkpoint `(file_id, byte_offset, row_number)` in the same transaction; the restart resumes with a ranged GET from the last offset; a unique `(file_id, row_number)` with `ON CONFLICT DO NOTHING` makes replays harmless.
- *Hostile or broken input:* a "line" with no newline for 2 GB (cap line and field lengths), quoted fields containing newlines (a real CSV parser, not a line splitter), a BOM or split UTF-8 (`StringDecoder`), a gzip bomb (cap decompressed bytes), 100% invalid rows (abort the job after N rejects and say so), and formula injection in `rejects.csv` (prefix cells starting with `=`, `+`, `-`, `@`).
- *Deploy mid-file:* stop at a batch boundary, commit the checkpoint, release the lease, exit.

**What breaks first:** the database. WAL volume, index maintenance and I/O contention with payment traffic hit before Node does. Unindexed staging plus a batched merge, admission control on concurrent jobs, and off-peak scheduling for the biggest files; beyond that, ingestion gets its own database.

---

## 14. Worked design C — Real-time fan-out to 100k SSE clients

**① Clarify:** 100k concurrent connections (Console users, merchant dashboards); 2k events/s platform-wide, each for one merchant, ~3 connected subscribers on average → **6k event writes/s**; heartbeats every 20 s → **5k writes/s**; p99 delivery under 1 s; resume after a disconnect without gaps.

**② Topology:**
```text
 outbox-relay ──publish (event JSON)──► Redis pub/sub ──► SSE instance × 12 ──► Map<merchantId, Set<Client>> ──► res.write
                                                              │
                         resume: Last-Event-ID ──► Redis ring buffer (recent) ──► Postgres events (sequence) fallback
```

**③ Budgets:**
- **Connections per instance:** assume ~50 KB per idle SSE connection (socket, HTTP objects, buffers; measure with a load generator) → 10k connections ≈ 500 MB. 100k ÷ 10k = 10 instances; **12** so that losing one moves ~8.3k clients onto the rest without breaching the 10k plan.
- **Memory:** a 1 GB limit with the heap capped near 640 MB, **lower than the usual 75%**, because per-connection memory is partly off-heap. `ulimit -n` ≥ 65,536.
- **CPU:** 11k writes/s ÷ 12 instances ≈ **900 writes/s each**, well under 0.1 ELU. This service is bound by memory and connections, not CPU.
- **Fan-out choice, with arithmetic:** `LISTEN/NOTIFY` caps payloads near 8000 bytes, so you'd send IDs, and every instance holding a subscriber then reads the row: up to 3 instances × 2k events/s ≈ 6k reads/s for nothing. **Publish the serialized event once to Redis pub/sub**; each instance does one Map lookup per message (2k/s, trivial). Ledger's own scale uses `LISTEN/NOTIFY` ([Lesson 20](../04-networking/20-realtime-servers.md)); this is where that decision flips.
- **Load balancer:** heartbeats every 20 s beat the ALB's 60 s idle timeout; HTTP/2 to browsers at the LB; compression off and `X-Accel-Buffering: no` on the stream route.

**④ Failure modes:**
- *A slow consumer:* `res.write()` returns `false` → stop writing to that client, buffer up to ~256 KB, then disconnect; the client resumes from its last ID.
- *An instance dies or deploys:* **the reconnect storm.** 10k clients reconnect within seconds, each with a `Last-Event-ID` that becomes a resume query. Before closing, send each client `retry:` with a random 1–10 s value and close in batches over ~20 s; serve resumes from a per-merchant Redis ring buffer of recent events before touching Postgres; concurrency-limit resume queries and `singleflight` identical ones.
- *Auth expiry:* validate the token on connect and close the stream when it expires, so revocation applies to long-lived connections too.

**What breaks first:** the reconnect storm: resume queries against the database and TLS handshakes at the load balancer, all within seconds. Then memory, if slow consumers aren't evicted. Neither is the event loop.

---

## 15. The five meta-answers

**"How do you know a Node service is healthy?"**
The user-facing SLOs (RED per endpoint), plus the runtime vitals no JVM engineer would think to watch: **event-loop delay p99, ELU, heap used vs its limit, restarts by exit code, pool waiting count, oldest job age**. Liveness checks the process only; readiness checks dependencies and fails first during shutdown. *"CPU% is the one number I don't trust on its own."*

**"It's slow. Where do you look first?"**
Loop delay and ELU split the world in two. High: it's our own synchronous work, so profile under load. Low: we're waiting, so check pool waiting counts, the libuv pool, and downstream latency in a trace. **One metric, two branches, before any guessing.**

**"How do you debug a production problem you can't reproduce?"**
Capture evidence from the running process without restarting it: a diagnostic report (`process.report.writeReport()` from an authenticated admin endpoint; `--report-on-fatalerror` for crashes), a CPU profile through the `inspector` module for 30 seconds, a heap snapshot on an instance drained from traffic, `--heapsnapshot-near-heap-limit` for the crash you can't catch, and logs carrying request and merchant IDs from `AsyncLocalStorage`. **Set these up before the incident.**

**"Do you scale out or fix the code?"**
Look at ELU per replica and the shared resources. Replicas buy time when the loop is the bottleneck and the work parallelizes, so scale out to stop the bleeding and profile to fix it. When the bottleneck is shared (the database, a downstream), **replicas make it worse**, so fix the shared path. *"I scale to contain and fix to resolve, and I know which one I'm doing."*

**"How do you prevent this class of bug generally?"**
Make the safe path the only path: an HTTP client wrapper that can't be called without a deadline, `limit()` as the only way to fan out, `withTransaction` as the only way to hold a client, config validated at boot, `no-floating-promises` in lint, and CI tests for the invariants (export memory ceiling, zero failures during SIGTERM, tenant isolation on a pool of one). **Delete the ability to make the mistake rather than training people not to.**

---

## 16. Practice protocol

Cover the answers. For each scenario, say the five beats out loud in under 90 seconds:

```text
[ ] Did I contain before diagnosing?
[ ] Did I name the first metric, and what each value would mean?
[ ] Did I separate one instance / one endpoint / one tenant from "everything"?
[ ] Did I name the mechanism (the phase, the pool, the timeout, the buffer)?
[ ] Did I give a structural prevention, a regression test, and an alert?
```

**Over-rehearse these**, because they're asked most and reward depth: #1 (p99 with low CPU), #7 (the leak), #8 (OOMKilled with a small heap), #15 (the 502), #20 (dropped requests in a deploy), #24 (hangs with an idle DB), #33 (the compromised dependency).

For the design round: three timed 45-minute runs of §12–14 from a blank page, then three fresh prompts: a rate-limited outbound email service, a CSV export service for 10k merchants at month-end, a job scheduler with per-tenant fairness. Score each with the budget sheet: **did every number come from a stated assumption or from arithmetic?**

---

## The Node track is complete

You've covered 38 lessons: what Node is, the event loop and async, data and I/O, networking from the metal, concurrency, data and jobs, production internals, shipping, a full project, and interview performance.

### What you can now do
- Predict any `nextTick` / promise / `setImmediate` / timer ordering and explain it phase by phase, including the ES-module twist.
- Prove the loop is blocked from loop delay and ELU, find the frame in a flame graph, and fix it at the right layer.
- Stream 4 million rows in constant memory and stop the query when the client leaves.
- Choose between worker threads, child processes, cluster and replicas, with reasons.
- Ship a graceful shutdown that loses zero requests in a rolling deploy, PID 1 included.
- Find a leak with heap snapshots, and explain an OOMKill with a healthy-looking heap.
- Size pools, the libuv thread pool and a worker pool against each other, with arithmetic.
- Publish a typed ESM package with a correct `exports` map, and defend your supply-chain posture.
- Answer "is Node single-threaded?" so there's no follow-up.

### Before you call it done
1. **Retake the 20-question self-test** from the [README](../README.md), cold and timed, and compare it with the score you noted before Lesson 01. The model answers are in [Lesson 36](36-rapid-fire-master-qa.md).
2. **Empty `RECALL.md`.** Every question you hedged on is a debt; pay it while the context is fresh.
3. **Build Ledger Engine at least through the dispatcher phase** of the [build log](../09-project/35-ledger-engine-build-log.md), and run its incident drills. Reading 38 lessons without building gives you recognition, not skill, and interviewers can tell within two follow-up questions.
4. **Do the three timed design rounds** from §12–14 out loud, plus ten scenarios from §2–10.

### How to keep it
Re-test `RECALL.md` every Sunday, cold. Re-run three of Ledger's incident drills every quarter (a slow DNS, a slow client during export, a rolling deploy under load), because these skills decay without use. When a new LTS lands each October, read its release notes for changes to the defaults this track taught you (timeouts, `highWaterMark`, module loading) and re-check your numbers.

### How it connects
- **[API & REST](https://github.com/Pawar-Pratik/API-mastery)** designed Ledger's contract: idempotency, webhooks, pagination, the error model. This track built the runtime that keeps those promises under load.
- **[TypeScript](https://github.com/Pawar-Pratik/TypeScript-mastery)** gave you the types; Node now runs them directly, which is why every lesson here was erasable-only.
- **[React](https://github.com/Pawar-Pratik/React-mastery)** built the Console; its live feed is the SSE server from [Lesson 20](../04-networking/20-realtime-servers.md).
- **[Spring Boot](https://github.com/Pawar-Pratik/SpringBoot)** is the same set of ideas on another runtime: pools, transactions, shutdown, observability. Being able to say how each one maps between the JVM and Node is what makes you framework-independent, and that's what senior interviews probe.

> Node has no magic in it any more. When a process is slow, leaking, crashing or dropping requests, you know which of the five ideas it's violating, which metric proves it, and which line to change.

Back to the hub → **[The Legend Track](https://github.com/Pawar-Pratik/The-Legend-Track)**
