# Lesson 23 — Scaling out: cluster, containers & cores

> **Why this lesson exists:** one Node process runs your JavaScript on one core. Using a 16-core machine, or a Kubernetes cluster, means choosing between the `cluster` module, PM2 and more replicas, and each choice has CPU-limit, memory and correctness consequences that people get wrong in production: worker counts read from the wrong number, heaps that add up past the container limit, and rate limits that silently multiply. This lesson makes the choice mechanical.

**Time:** ~65 minutes · **Prereq:** Lessons 21, 22

---

## 1. The idea in one sentence

> **Node scales request handling across cores by running more processes, not more threads: in containers that means one process per container and more replicas, each sized against the CPU and memory it's actually allowed, and every piece of in-process state becomes wrong the moment there are two of you.**

---

## 2. The one-core ceiling

A Node process handles requests on one JavaScript thread. When that thread is busy all the time (ELU ≈ 1, [L08](../02-event-loop/08-blocking-the-loop.md)), the process is saturated, however many cores the machine has. The libuv pool and V8's GC threads use other cores a little, but **request handling doesn't spread out on its own.**

Worker threads ([L21](21-worker-threads.md)) don't solve this. They parallelize a *task* you hand them; the HTTP server, its sockets and your handlers stay on the main thread. To handle more requests in parallel, you need more event loops, each owning connections. That means more processes, and there are three ways to run them:

```text
 (a) cluster / PM2: one machine, one primary     (b) replicas: the orchestrator is the primary
┌────────────────────────────────────┐           ┌──────────┐ ┌──────────┐ ┌──────────┐
│ primary (supervisor, owns :3000)   │           │ pod: node│ │ pod: node│ │ pod: node│
│   ├─ worker 1 (node, own heap)     │           │ 1 process│ │ 1 process│ │ 1 process│
│   ├─ worker 2 (node, own heap)     │           └────▲─────┘ └────▲─────┘ └────▲─────┘
│   └─ worker N (node, own heap)     │                └───── Service / load balancer ┘
└────────────────────────────────────┘
```

One warning before any sizing. **Ten replicas don't fix a blocked event loop**: every replica blocks the same way on the same request, and you've only spread the damage. Extra processes add capacity for work that's already well-behaved. Fix the blocking first ([L08](../02-event-loop/08-blocking-the-loop.md)), then scale.

---

## 3. The `cluster` module

`cluster` is `child_process.fork` ([L22](22-child-processes.md)) plus one trick: **the workers can share a listening socket.** The same script runs in the primary and in each worker, and `cluster.isPrimary` tells them apart.

### How connections reach workers

When a worker calls `server.listen(3000)`, it doesn't bind the port itself. It asks the primary over IPC, and then one of two policies applies:

| Policy | Default on | What happens |
|---|---|---|
| **`SCHED_RR`** (round-robin) | **Every platform except Windows** | The primary owns the listening socket, `accept()`s each connection and **sends the connection handle to the next worker** over IPC |
| **`SCHED_NONE`** | **Windows** | The primary shares the listening handle; every worker calls `accept()` and the OS decides who wins |

```text
SCHED_RR, step by step
  client ──SYN──▶ :3000  (the listening socket lives in the PRIMARY)
                    │ primary's event loop: accept() → a new connected socket
                    │ pick the next worker in the rotation
                    ▼
          primary ──IPC: "here's a handle" (the fd is passed through the IPC channel)──▶ worker 2
                    │ the primary drops its copy; worker 2 owns the socket from now on
                    ▼
          worker 2 ──reads the request, runs your handler, writes the response──▶ client
```

`SCHED_NONE` looks more efficient, but Node's own documentation notes that in practice the OS distributes very unevenly, with a few workers taking most connections, which is why round-robin became the default. Override it with `cluster.schedulingPolicy` before forking, or `NODE_CLUSTER_SCHED_POLICY=rr|none`.

**Distribution is per connection, not per request.** A keep-alive connection lives on one worker for its whole life. A load balancer that holds a handful of long-lived connections to your machine can pin most of the traffic to a few workers, however fair the round-robin.

**The rotation is also health-blind.** The primary doesn't know a worker's event loop is blocked or its pool is exhausted; it keeps handing that worker every Nth new connection, and those requests stall. Kubernetes can at least take a pod out of rotation when its readiness probe fails. A blocked cluster worker inside a pod has no such escape: the pod stays "ready" because the other workers answer the probe. And the accept loop itself runs on the primary's single thread, so the primary is one more process whose loop must stay healthy.

### Supervision and rolling restarts

```ts
// cluster-primary.ts: for understanding only; Ledger runs replicas instead (§9)
import cluster from "node:cluster";
import { once } from "node:events";

const WORKERS = Number(process.env.WEB_CONCURRENCY ?? 2);    // from config, never os.cpus()

if (cluster.isPrimary) {
  let restarts = 0;
  for (let i = 0; i < WORKERS; i++) cluster.fork();

  cluster.on("exit", (worker, code, signal) => {
    if (worker.exitedAfterDisconnect) return;                // we asked it to leave
    log.error({ pid: worker.process.pid, code, signal }, "worker died");
    setTimeout(() => cluster.fork(), Math.min(30_000, 1_000 * 2 ** restarts++));  // crash-loop backoff
  });

  process.on("SIGHUP", () => void rollingRestart());         // POSIX only: run it in WSL2
} else {
  await import("./main.ts");     // each worker runs the real server; listen() goes via the primary
}

async function rollingRestart(): Promise<void> {
  for (const old of Object.values(cluster.workers ?? {})) {
    if (!old) continue;
    const fresh = cluster.fork();
    await once(fresh, "listening");                          // the new one accepts first...
    old.disconnect();                                        // ...then the old one stops accepting and drains
    const kill = setTimeout(() => old.process.kill("SIGKILL"), 30_000);
    await new Promise((resolve) => old.on("exit", resolve));
    clearTimeout(kill);
  }
}
```

`worker.disconnect()` closes the worker's servers so it takes no new connections, lets existing ones finish, and the worker exits once nothing keeps it alive. That's [L24](24-graceful-shutdown.md)'s drain, one level down. Note what you just built: **a supervisor with restarts, backoff and rolling deploys. In Kubernetes, that's the orchestrator's job, and doing it twice hides failures.**

**Every worker is a full process.** Its own heap, its own `pg.Pool` (so `WORKERS × pool.max` connections, [L25](../06-data-and-jobs/25-databases-from-node.md)), its own caches and its own timers. Nothing in a module is shared between workers.

### PM2

PM2 is a process manager built on the same mechanism: its cluster mode (`exec_mode: "cluster"`, `instances: N`) uses the `cluster` module and adds automatic restarts, log handling, `pm2 reload` for rolling restarts and `pm2-runtime` for containers. It's a reasonable choice on a VM or bare metal with no orchestrator. Watch `instances: "max"`: it means "one per CPU the machine reports", which is the container trap in §5.

---

## 4. Containers: one process per container

On Kubernetes or ECS, the modern default is **one Node process per container, and more replicas for more throughput.** Each reason is mechanical:

| Concern | Cluster inside a pod | One process per pod |
|---|---|---|
| **Supervision** | The primary restarts workers, and a crash loop hides inside a pod that still looks healthy | The orchestrator restarts the container, and restarts are visible in metrics and alerts |
| **Signals** | SIGTERM reaches the primary, which must forward it and coordinate N drains | SIGTERM reaches the process that holds the connections ([L24](24-graceful-shutdown.md)) |
| **Memory limit** | N heaps share one limit; an OOM kill takes out one worker or the whole pod | One heap, one limit, one simple calculation (§6) |
| **Metrics** | Per-pod numbers are sums over N processes; per-process loop delay needs extra work | Loop delay, ELU and heap are per pod by construction ([L29](../07-production/29-observability.md)) |
| **Scaling** | In steps of a whole pod's worth of workers | One replica at a time, by the HPA |

The costs are real but small: a baseline RSS per process, and one connection pool per replica. **Cluster still earns its place where there's no orchestrator:** a single VM, an on-premises box, a legacy deployment. In Kubernetes, adding cluster or PM2 inside the pod is a second supervisor fighting the first.

---

## 5. CPU: the cores you see vs the cores you're allowed

| Call | Returns | Inside a container with a CPU limit |
|---|---|---|
| `os.cpus().length` | The machine's logical CPUs | **Still the host's count:** 64 on a 64-core node with a 1-CPU limit |
| `os.availableParallelism()` | libuv's estimate of usable parallelism | Respects the CPU *affinity* mask (`--cpuset-cpus`); whether it reflects a CPU *quota* depends on the libuv in your Node, so don't build on it |

Check what your image actually reports: `docker run --rm --cpus=1 node:24 node -p "[os.cpus().length, os.availableParallelism()]"`. Whatever it prints, **take worker and thread counts from config**, set them from the pod spec, and log all three numbers at boot so a mismatch is visible:

```ts
log.info({ configured: config.webConcurrency, availableParallelism: os.availableParallelism(),
  cpus: os.cpus().length }, "cpu budget");
```

### How a CPU limit actually works: CFS quota and throttling

In Kubernetes, `requests.cpu` is a scheduling weight; `limits.cpu` becomes a **CFS quota**: CPU time per period, for all the container's threads combined. The default period is 100 ms.

```text
limits.cpu: 1  →  100 ms of CPU time per 100 ms period, across every thread in the container

1 busy thread    ████████████████████ (100 ms)                    → never throttled
8 busy threads   ██ 12.5 ms ── quota spent ── frozen for 87.5 ms ──→ every request in flight waits
```

A Node process is never just one thread: the main thread, 4 libuv pool threads, V8's GC and compiler helpers, plus any worker threads. **A GC burst or a few pool threads can spend a 1-CPU quota in a fraction of the period, and then the whole container is frozen until the next one.** Average CPU looks fine (40%), but p99 latency has 50–90 ms steps in it. Spawn `os.cpus().length` workers in that container and you make it much worse: dozens of threads drain the quota almost instantly.

You can see it directly: `/sys/fs/cgroup/cpu.stat` (cgroup v2) has `nr_throttled` and `throttled_usec`, and cAdvisor exports `container_cpu_cfs_throttled_periods_total`. Rules that follow:
- **One API process per container, with a CPU limit of at least 1** (many teams set CPU requests and no CPU limit at all, precisely to avoid throttling; that's a cluster-policy decision).
- **Worker-thread pools sized to the limit**, minus one if the main thread is busy ([L21](21-worker-threads.md)).
- **Alert on throttling**, not just on CPU utilization.

---

## 6. Memory: every process brings its own heap

Each process has its own V8 heap, its own off-heap memory (Buffers, [L27](../07-production/27-memory-and-gc.md)) and a baseline RSS. The heap ceiling is per process, and the default depends on the system, so you set it explicitly with `--max-old-space-size`, usually through `NODE_OPTIONS`.

**`NODE_OPTIONS` is inherited by every forked worker**, so a cluster of N applies the same limit N times:

```text
container limit 2 GiB, 4 cluster workers, NODE_OPTIONS=--max-old-space-size=1536
  allowed heap: 4 × 1.5 GiB = 6 GiB  ≫  2 GiB  → the kernel OOM-kills a worker (exit 137) long before V8 complains

correct: total heaps ≈ 75% of the limit → 2048 × 0.75 / 4 ≈ 384 MiB each
one process per container, 1 GiB limit → --max-old-space-size=768
```

Don't trust that the flag arrived; print the limit V8 actually applied, in every process, at boot:

```ts
import { getHeapStatistics } from "node:v8";

const heapLimitMiB = Math.round(getHeapStatistics().heap_size_limit / 1024 / 1024);
log.info({ heapLimitMiB, pid: process.pid }, "heap budget");   // compare with the pod's memory limit
```

Expect a number well above the flag: `heap_size_limit` adds the young generation's maximum on top of old space. On a 64-bit Node 24.11, `--max-old-space-size=768` reports 960 MiB, a fixed 192 MiB more at every size tried. **That young generation lives inside the 25% you left**, so in small containers (1 GiB and under) the headroom is thinner than it looks: watch RSS against the limit, and lower the flag if it gets close.

The 25% you leave is for everything that isn't the old-space heap: Buffers, new space, code, thread stacks, the primary. Worker threads count too: each isolate has its own heap inside the same limit, so give them explicit `resourceLimits` ([L21](21-worker-threads.md)). The full sizing method lives in [L27](../07-production/27-memory-and-gc.md) and [L33](../08-shipping/33-shipping-node.md).

---

## 7. What breaks when there are N of you

Everything a process keeps in memory is now one of N divergent copies. **This is the real cost of scaling out, and it's the same whether N comes from cluster, PM2 or replicas.**

| In-process state | What goes wrong at N > 1 | Fix |
|---|---|---|
| **Rate-limit counters** | Each process allows the full limit: the effective limit is ×N | A shared atomic counter: Redis + Lua ([API L19](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/19-rate-limiting.md)) |
| **In-process schedulers** (`setInterval` cron) | The job runs N times | Leader election with `pg_try_advisory_lock` ([L26](../06-data-and-jobs/26-background-jobs-and-queues.md)), or a separate single-replica deployment / CronJob |
| **In-memory caches** | Instances disagree; invalidation reaches one of N | Short TTLs, Redis, or invalidation broadcast over pub/sub |
| **Sessions in memory** | The next request lands elsewhere and the user is logged out | Redis sessions or stateless tokens |
| **SSE/WebSocket subscribers** | An event published on instance A never reaches clients on B | Fan-out via Postgres `LISTEN/NOTIFY` or Redis pub/sub ([L20](../04-networking/20-realtime-servers.md)) |
| **In-memory locks and dedupe** | Two instances both "hold the lock" | Database constraints, `SKIP LOCKED`, idempotency keys ([API L18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md)) |
| **Local disk** | A file uploaded to pod A doesn't exist on pod B | Object storage |

### Sticky sessions

Sticky routing sends a client back to the same instance. **You need it only when state lives in the process**: a WebSocket server keeping per-connection state that other instances can't reach, or a protocol whose session spans several HTTP requests (Socket.IO's long-polling transport is the classic case). It costs you: uneven load, scaling that doesn't rebalance existing clients, and a deploy that drops every session on the replaced pod at once. **Prefer externalizing the state.** SSE with `Last-Event-ID` resume needs no stickiness at all ([L20](../04-networking/20-realtime-servers.md)), and plain WebSockets are connection-bound anyway: a single connection never moves. The `cluster` module's round-robin is per connection and offers no stickiness.

---

## 8. The decision table

| You need… | Use | Why |
|---|---|---|
| More request throughput, on Kubernetes/ECS | **More replicas, one process each** | The orchestrator already supervises, routes, restarts and scales |
| More request throughput on one VM, no orchestrator | **PM2 cluster mode** (or `cluster`, or N systemd services behind nginx) | Something has to supervise and share the port |
| One CPU-heavy task off the event loop | **A worker-thread pool** ([L21](21-worker-threads.md)) | Parallelizes the task; the loop keeps serving |
| A non-JS program, or isolation from a crashy native library | **A child process** ([L22](22-child-processes.md)) | Separate process, separate failure domain |
| Background work with its own scaling signal | **A separate deployment** of a worker process | Scale it on queue depth, independently of HTTP traffic |

### What to scale on

Replicas only help if the autoscaler adds them for the right reason, and each kind of Node process saturates differently:

| Process | Saturates when… | Scale on | Misleading signal |
|---|---|---|---|
| **API** | The event loop is busy most of the time | CPU as a start; better, ELU or loop-delay p99 exported as a custom metric ([L29](../07-production/29-observability.md)) | Memory (it grows with caches, not load) |
| **Queue worker** | Jobs arrive faster than they finish | Queue depth, or **the age of the oldest due job** ([L26](../06-data-and-jobs/26-background-jobs-and-queues.md)) | CPU (a worker waiting on slow webhooks is idle and behind) |
| **CPU-pool process** (the reconciler) | Every pool thread is busy | Usually nothing: it's a batch job; give it a fixed size and a deadline | Request rate (it has none) |

CPU is a decent first proxy for an API because a saturated loop burns a core, but it lags, it's blurred by GC threads, and **a 1-CPU-limited pod being throttled reports high CPU for the wrong reason** (§5). Whatever you scale on, remember the floor: each new replica brings a new connection pool, and the database doesn't autoscale with you.

---

## 9. Ledger: the deployment plan

- **API:** a Deployment of N replicas × 1 process, scaled by the HPA from 4 to 8 on event-loop utilisation. No cluster, no PM2.
- **Webhook dispatcher:** its own Deployment, scaled on queue health (the age of the oldest due delivery, [L26](../06-data-and-jobs/26-background-jobs-and-queues.md)); its concurrency limits come from config ([L10](../02-event-loop/10-cancellation-and-concurrency.md)).
- **Idempotency-key sweeper and outbox relay:** small Deployments of their own (2 replicas each, for availability), with each tick guarded by an advisory lock or `SKIP LOCKED`, so N replicas never duplicate work.
- **Reconciler:** a single replica (or a Job per settlement file) with a worker pool sized from `RECONCILE_THREADS`, never from `cpus()`.

```yaml
# ledger-api Deployment (excerpt)
spec:
  replicas: 4                                   # HPA: 4..8
  template:
    spec:
      terminationGracePeriodSeconds: 45         # L24
      containers:
        - name: api
          image: ledger/api:1.42.0
          command: ["node", "src/main.ts"]       # one process: no cluster, no PM2
          env:
            - { name: NODE_OPTIONS, value: "--max-old-space-size=768" }   # ~75% of 1 GiB
            - { name: PG_POOL_MAX, value: "10" }                          # 8 × 10 = 80 at the HPA max, before export pools
          resources:
            requests: { cpu: "1", memory: 1Gi }
            limits:   { memory: 1Gi }           # CPU limit per cluster policy; ≥ 1 if set
---
# ledger-reconciler (excerpt)
          env:
            - { name: RECONCILE_THREADS, value: "3" }                     # CPU limit − 1 for the main thread
            - { name: NODE_OPTIONS, value: "--max-old-space-size=384" }   # applies to EVERY isolate: main + 3 workers
          resources:
            requests: { cpu: "4", memory: 2Gi }
            limits:   { cpu: "4", memory: 2Gi } # 4 isolates × 384 MiB = 1.5 GiB = 75%
```

The arithmetic you write next to that YAML is the deliverable: **replicas × pool `max` (plus the workers' pools) must fit Postgres's `max_connections` with headroom** ([L25](../06-data-and-jobs/25-databases-from-node.md)), the heaps must fit 75% of each limit, and thread counts must match CPU limits.

---

## 10. Production rules

| Rule | Why |
|---|---|
| **On an orchestrator, run one Node process per container and scale with replicas** | One supervisor, direct signals, one heap per memory limit, per-process metrics |
| **Never derive process or thread counts from `os.cpus()`** | It reports the host's CPUs, not your quota |
| **Set worker and thread counts from config, and log them next to `availableParallelism()` at boot** | A mismatch with the pod spec becomes visible on day one |
| **Keep CPU limits at ≥ 1 per process, or use requests without limits** | Multi-threaded processes burn small quotas fast and get throttled |
| **Alert on CFS throttling, not just CPU%** | Throttling shows up as p99 steps while average CPU looks healthy |
| **Size `--max-old-space-size` so all heaps fit ~75% of the limit** | `NODE_OPTIONS` applies to every forked process; heaps add up |
| **Count `replicas × pool.max` against `max_connections`** | Every process opens its own pool |
| **Move shared state out of the process before adding the second instance** | Rate limits multiply, cron runs N times, caches diverge |
| **Elect a leader for singleton jobs** | In-process schedulers run once per replica |
| **Avoid sticky sessions unless state truly can't leave the process** | They unbalance load and drop sessions on every deploy |
| **Fix event-loop blocking before scaling** | Every replica blocks the same way |

> **Spring equivalent:** a JVM uses every core from one process (Tomcat's thread pool), so Spring has no cluster module: scaling out is replicas from day one. The "cron runs N times" problem is exactly the one ShedLock solves for `@Scheduled` ([SpringBoot L32](https://github.com/Pawar-Pratik/SpringBoot/blob/main/09-production/32-async-scheduling-events.md)), and the container-memory arithmetic is the `-XX:MaxRAMPercentage` lesson in [SpringBoot L36](https://github.com/Pawar-Pratik/SpringBoot/blob/main/09-production/36-performance-and-jvm-tuning.md), except that in Node you compute `--max-old-space-size` yourself per process.

---

## 11. Interview traps

**Q1. "How do you use all the cores with Node?"**
Run more processes, because request handling lives on one thread per process. On Kubernetes that means more replicas with one process each; on a bare VM, the `cluster` module or PM2. Worker threads help with CPU-heavy *tasks*, not with spreading request handling. The weak answer is "use worker threads" or "Node can't".

**Q2. "`cluster` vs `worker_threads`?"**
`cluster` forks whole processes that share a listening socket, so it scales connection handling; each worker has its own heap and its own everything. `worker_threads` are isolates inside one process for offloading CPU work; they can't share a server socket and they share the process's fate. Different problems: throughput across cores vs one heavy computation.

**Q3. "`cluster` vs Kubernetes replicas?"**
Replicas, on an orchestrator. Cluster adds a second supervisor inside the pod: it hides crash loops, complicates signal handling, puts N heaps under one memory limit and blurs per-process metrics. Cluster is for machines with no orchestrator.

**Q4. "What breaks when you go from 1 process to N?"**
All in-process state: rate limits become ×N, in-process cron runs N times, caches diverge, in-memory sessions and WebSocket subscriptions only exist on one instance, local files aren't shared, and DB connections multiply by N. **Follow-up:** "How do you run a cron job once?" A leader per tick via `pg_try_advisory_lock`, or a separate single-replica job.

**Q5. "How many workers in a container with a 2-CPU limit?"**
Not `os.cpus().length`, which reports the host and would throttle you hard. In Kubernetes, one process per container and more replicas; if you must cluster inside it, two processes, set from config. A thread pool for CPU work would be about the limit minus one. **Follow-up:** "What happens if you spawn 64?" The CFS quota of 200 ms per 100 ms period is spent in a few milliseconds, the container is frozen for the rest of each period, and p99 explodes.

**Q6. "How does `cluster` distribute connections?"**
By default on everything except Windows, round-robin: the primary accepts each connection and passes its handle to a worker over IPC. On Windows the workers share the listening handle and the OS picks, which tends to be uneven. Distribution is per connection, so long-lived keep-alive connections can skew load.

**Q7. "Average CPU is 40%, but p99 has jumps of 50–90 ms. What do you check?"**
CFS throttling: `nr_throttled` and `throttled_usec` in `cpu.stat`, or `container_cpu_cfs_throttled_periods_total`. A multi-threaded process (libuv pool, GC helpers, workers) can spend a small quota early in each 100 ms period and then freeze. Fix the thread counts, raise or remove the CPU limit.

**Q8. "Four cluster workers in a 2 GiB container, each with `--max-old-space-size=1536`. What happens?"**
`NODE_OPTIONS` is inherited, so the heaps may grow to 6 GiB in total. The kernel OOM-kills a process (exit 137) before V8 ever reports a heap limit. Budget the heaps together: about 384 MiB each.

**Q9. "Do you need sticky sessions?"**
Only when state lives in the process across several connections or requests: a stateful WebSocket service, or a long-polling transport. Plain WebSockets are connection-bound, and SSE resumes with `Last-Event-ID`, so neither needs stickiness if the state is external. Stickiness costs balance and drops sessions on every deploy.

**Q10. "Will adding replicas fix our latency problem?"**
Only if the processes are healthy and simply busy. If the event loop is blocked by synchronous work, every replica blocks the same way on the same requests; you've spread the damage, not removed it. Check loop delay and ELU first.

**Q11. "One of four cluster workers has a blocked event loop. What happens to traffic?"**
The primary keeps handing it every fourth new connection, because round-robin knows nothing about worker health, so a quarter of new connections stall while the other workers sit partly idle. The pod's readiness probe probably still passes, since another worker answers it. With one process per pod, the blocked process fails its own probe and the orchestrator stops routing to it.

**Q12. "What should the autoscaler scale a Node API and a queue worker on?"**
The API on event-loop saturation: CPU as a first proxy, ELU or loop-delay p99 as a better custom metric. The queue worker on its backlog, ideally the age of the oldest due job, because a worker waiting on slow downstream calls shows low CPU while falling behind. In both cases, check the database can absorb the extra connection pools.

---

## 12. Build & break

### Build — a cluster primary, for understanding
Implement §3: N workers from `WEB_CONCURRENCY`, restart with exponential backoff on unexpected exits (reset the counter once a worker has stayed up for a minute), `SIGHUP` → rolling restart, `SIGTERM` → `disconnect()` every worker and exit when they're gone, with a hard deadline. Each worker serves `GET /whoami` returning its `process.pid`. Hit it 20 times with `curl.exe` (or `curl` in WSL2) and look at the distribution. On Windows you're seeing `SCHED_NONE`; rerun with `NODE_CLUSTER_SCHED_POLICY=rr` and compare. Then run it in WSL2 with `kill -HUP <primary pid>` while `npx autocannon -c 20 -d 30` runs, and confirm zero errors through the rolling restart.

### Build — the Ledger deployment plan
Write `deploy/PLAN.md` beside the manifests from §9: for each deployment, the replica range, CPU and memory requests/limits, `NODE_OPTIONS`, thread or concurrency counts, pool sizes, and the arithmetic (connections at peak vs `max_connections`, heaps vs 75% of limits, threads vs CPU limits). Add the scaling signal for each deployment and the reason it's that signal.

### Break — 4 experiments
1. **The ×4 rate limit.** Put an in-memory limiter (10 requests per minute per API key, a `Map`) behind your 4-worker cluster. Fire 40 requests with distinct connections (`npx autocannon -c 40 -a 40 -H "authorization: Bearer k1"`, or 40 `curl.exe` calls). About 40 succeed instead of 10. On Windows the `SCHED_NONE` default may skew it, so set `NODE_CLUSTER_SCHED_POLICY=rr` or run in WSL2. Fix: the Redis limiter from [API L19](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/19-rate-limiting.md); now exactly 10 succeed.
2. **The triple cron.** Start the API three times (`PORT=3001`, `3002`, `3003`), each with `setInterval(sweepExpiredIdempotencyKeys, 10_000)` that inserts a row into a `sweeper_runs` table. After a minute, `SELECT count(*), date_trunc('second', ran_at) FROM sweeper_runs GROUP BY 2`: three rows per tick. Wrap the tick in `pg_try_advisory_lock` ([L26](../06-data-and-jobs/26-background-jobs-and-queues.md)): one row per tick.
3. **The 1-CPU container that thinks it has 16.** On Docker Desktop: `docker run --cpus=1` your cluster with `WEB_CONCURRENCY` set to `os.cpus().length`. Print `os.cpus().length` and `os.availableParallelism()` inside, then run autocannon and record p99. Read `nr_throttled` from `/sys/fs/cgroup/cpu.stat` before and after. Repeat with `WEB_CONCURRENCY=1`: compare p99 and the throttled counts.
4. **Heaps that don't fit.** `docker run -m 1g` the cluster with 4 workers and `NODE_OPTIONS=--max-old-space-size=512`, plus a route that retains 10 MB per call. Load it until a worker dies: the primary's `'exit'` handler logs `signal: "SIGKILL"`, the kernel's OOM kill, with no V8 heap error anywhere. Recompute the heap budget from §6 and repeat: now V8's own limit is hit first, and you get a heap error you can diagnose ([L27](../07-production/27-memory-and-gc.md)).

### Explain out loud (60–120 seconds)
1. Why request handling needs more processes, and why worker threads don't provide them.
2. How `cluster` shares a port, round-robin vs the Windows default, and why distribution is per connection and blind to worker health.
3. Why one process per container beats cluster inside a pod.
4. CFS quotas: why `os.cpus()` lies in a container, and how throttling hurts p99 at low average CPU.
5. The heap arithmetic, and the list of in-process state that breaks at N > 1.

---

## What's next

Every scaling decision here assumed processes can be replaced safely. Every deploy, scale-down and node drain is a shutdown, and doing it wrong drops in-flight requests, half-finishes jobs and leaves the load balancer sending traffic to a dying pod. Next is the shutdown sequence, step by step, and the PID 1 trap behind most "why does my container take 30 seconds to stop?" questions.

Next → **[Lesson 24: Graceful shutdown & the process lifecycle](24-graceful-shutdown.md)**
