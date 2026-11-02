# Lesson 24 — Graceful shutdown & the process lifecycle

> **Why this lesson exists:** every deploy is a shutdown, and so is every scale-down and every node drain. Get it wrong and each rollout drops in-flight requests, half-finishes jobs, and leaves the load balancer sending traffic to a pod that's already closing. Get the container setup wrong and the process never even sees the signal: it hangs for 30 seconds and is killed with no cleanup at all. "Walk me through graceful shutdown in Kubernetes" is a standard senior question, and the answer is a precise sequence.

**Time:** ~80 minutes · **Prereq:** Lessons 17, 23

---

## 1. The idea in one sentence

> **Shutdown is a sequence, not an event: stop new work from arriving, finish the work you've already accepted under a deadline, then release resources in reverse dependency order, and make sure the signal that starts all this actually reaches your process.**

---

## 2. Every deploy is a shutdown

A rolling update starts new pods and terminates old ones. For each old pod, here's what happens with no plan:

```text
SIGTERM arrives → Node's default: exit immediately (code 143)
  ├─ 37 requests in flight      → sockets reset; clients see ECONNRESET, the LB returns 502
  ├─ the LB hasn't caught up    → new requests for ~1–2 s go to a closed port: more 502s
  ├─ a webhook send in progress → the delivery row stays claimed until its lease expires
  └─ pino's buffered log lines  → gone, including the one that would explain the 502s
```

Ledger survives each of these, but only because of other lessons: a dropped `POST /v1/payments` is retried with its idempotency key ([API L18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md)), and a half-sent webhook is redelivered because delivery is at-least-once ([L26](../06-data-and-jobs/26-background-jobs-and-queues.md)). **Crash-only design ([L09](../02-event-loop/09-errors-and-crashing.md)) is what makes a hard kill safe; graceful shutdown is what makes it rare.** You need both: SIGKILL, the OOM killer and a dead node never run your handler.

---

## 3. Signals

| Signal | No. | Sent by | Catchable? | Exit code if it kills you |
|---|---|---|---|---|
| **SIGTERM** | 15 | `docker stop`, Kubernetes, systemd, `kill <pid>` | ✅ the polite request | 143 |
| **SIGINT** | 2 | Ctrl+C in a terminal | ✅ | 130 |
| **SIGKILL** | 9 | The end of a grace period, the OOM killer, `kill -9` | ❌ never: no handler, no cleanup, no flush | 137 |
| **SIGHUP** | 1 | A closed terminal; often repurposed as "reload" | ✅ | 129 |

**Node's default for SIGTERM and SIGINT** is to reset the terminal and exit with 128 + the signal number. **The moment you add a listener, that default is gone:** Node no longer exits on that signal, and finishing the shutdown and exiting become your job. A handler that logs "shutting down" and does nothing else produces a process that ignores SIGTERM until SIGKILL arrives.

### Windows

Windows has no POSIX signal delivery between processes. Ctrl+C raises `SIGINT` and Ctrl+Break raises `SIGBREAK`, and closing the console window raises `SIGHUP`, after which Windows terminates the process shortly afterwards. You can *listen* for `SIGTERM`, but nothing ever sends it: `process.kill(pid, "SIGTERM")` from another process, or `child.kill()`, terminates the target unconditionally and no handler runs. Practical rules:
- **Handle `SIGINT` and `SIGTERM` with the same function**, so Ctrl+C on Windows exercises the same shutdown path you'll run in production.
- **Test real SIGTERM behaviour in WSL2 or in a container.** Docker Desktop runs Linux containers, so `docker stop` sends a genuine SIGTERM inside the VM.

---

## 4. The Kubernetes termination sequence

```text
t=0   Pod gets a deletionTimestamp → "Terminating". Two things start AT THE SAME TIME:

  (A) control plane                              (B) kubelet on the pod's node
  EndpointSlice: pod marked not ready             run the preStop hook, if any
     ↓ asynchronously                                ↓ when it returns
  kube-proxy on EVERY node rewrites its rules     SIGTERM → PID 1 of each container
  ingress controllers / cloud LBs update             ↓
     (milliseconds to seconds; not yours)         your shutdown sequence runs

t=terminationGracePeriodSeconds (default 30 s, counted from t=0, preStop included)
      → SIGKILL to anything still running
```

**The race:** (A) and (B) are concurrent, so SIGTERM routinely arrives *before* every proxy has stopped sending you traffic. If your handler closes the listening socket immediately, the requests still being routed to you hit a closed port, and the load balancer turns them into 502s. **"We have a SIGTERM handler and still drop requests on every deploy" is this race.**

The fix is to keep serving normally for a few seconds after SIGTERM, until propagation has finished. Two ways to get that window:

| | preStop sleep | In-process drain delay |
|---|---|---|
| How | `lifecycle.preStop` runs `sleep 5`, and SIGTERM arrives after it | On SIGTERM, mark not-ready and wait `DRAIN_DELAY_MS` before closing anything |
| Needs | A `sleep` binary in the image (distroless has none), or the built-in `sleep` action in newer Kubernetes versions | Nothing |
| Covers | Kubernetes only | Kubernetes, Docker, VMs behind any load balancer |
| Testable | In a cluster | In a unit test |

Ledger uses the in-process delay. Either way, **`terminationGracePeriodSeconds` must exceed the delay plus your shutdown deadline**, or the grace period ends mid-drain with a SIGKILL.

**Where readiness fits:** Kubernetes removes a terminating pod from its Service endpoints whatever its readiness probe says. Failing `/readyz` during the drain still matters for everything that discovers you by health-checking you directly (a cloud load balancer's target checks, a mesh, a deployment that isn't Kubernetes), and it costs one line, so do both. That's [API L20](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/20-observability.md)'s rule; this is the mechanism behind it. **`/healthz` (liveness) must keep passing while you drain**, or the kubelet restarts you mid-shutdown.

---

## 5. The ordered shutdown

```text
SIGTERM / SIGINT
 1. state = draining          /readyz → 503 · new responses carry `Connection: close`
 2. drain delay (5 s)         keep serving normally while load balancers catch up
 3. stop accepting            server.close() · SSE clients told to reconnect elsewhere
                              idle keep-alive sockets closed, and swept every second
 4. finish in-flight work     requests (each has its own deadline, L10) · the job loop stops
                              claiming and finishes, or releases, its current job
 5. at the deadline           closeAllConnections() · abort what's left (it will be retried)
 6. close resources           Postgres pool.end() · Redis quit() — only after nothing uses them
 7. flush                     logs, traces, metrics
 8. exit                      the loop empties → exit 0; an unref'd timer exits 1 if it doesn't
 any second signal            exit 1 immediately: a human or the orchestrator has run out of patience
```

**The order is reverse dependency order.** In-flight requests need the database, so the pool closes after HTTP has drained, never before; that ordering bug shows up as a burst of "Cannot use a pool after calling end" errors on every deploy. Logs close last because every other step logs.

**Let the process exit by itself.** If every resource was really closed, the event loop is empty and Node exits with `process.exitCode` ([L03](../01-foundations/03-the-process.md)). If it doesn't exit, something leaked: an interval, a socket, a client nobody ended. The safety-net timer is `unref()`'d so it can't keep the process alive itself, and when it fires it tells you shutdown was incomplete. `process.getActiveResourcesInfo()` (experimental) lists what's still holding the loop; log it there.

The budget, as Ledger configures it:

```text
t=0s     SIGTERM · draining · /readyz 503
t=0–5s   drain delay: business as usual
t=5s     server.close() · SSE told to reconnect · idle sockets closed
t=5–30s  in-flight requests finish (each ≤ 20 s) · dispatcher finishes current sends
t=30s    shutdown deadline: closeAllConnections() · abort the rest · pools · flush · exit 0
t=32s    hard deadline: process.exit(1) if anything still holds the loop
t=45s    terminationGracePeriodSeconds: Kubernetes would SIGKILL here; never reached
```

---

## 6. Draining HTTP: `server.close()` and keep-alive

`server.close()` stops accepting new connections and calls back when **every** connection has closed. Since Node 19 it also closes connections that are idle at that moment. What it does *not* do is the part that bites:

1. **A busy keep-alive connection keeps serving new requests after `close()`.** Node stops accepting *connections*, not requests. A client with an open keep-alive socket can send request after request on it, each answered with `Connection: keep-alive`, and `close()` never completes while it does. Under load from a load balancer that pools connections, that can mean forever.
2. **A connection that finishes its last response after `close()` is not closed right away.** It sits idle until the keep-alive timeout expires, several seconds later, delaying the callback for nothing.
3. **Long-lived responses never finish on their own.** An SSE stream is an in-flight response ([L20](../04-networking/20-realtime-servers.md)); `close()` waits for it indefinitely.

The fixes, all small:

```ts
// ✅ first middleware: once draining, every response is the last one on its connection
app.use((_req, res, next) => {
  if (isDraining()) res.setHeader("Connection", "close");   // Node closes the socket after this response
  next();
});
```

`Connection: close` tells the peer this connection is finished after this response, so a load balancer opens its next connection to another pod. In a quick experiment, the same slow request let `close()` finish in about 1.5 s with the header and about 7.5 s without it. Then in the closer: sweep idle connections every second (requests already past the middleware didn't get the header), end SSE streams with a `retry:` hint so clients reconnect elsewhere, and keep `closeAllConnections()` for the deadline, because it resets sockets that are mid-request.

**Why the drain delay comes first, again:** closing an idle keep-alive socket at the exact moment a load balancer reuses it is the 502 race from [L17](../04-networking/17-http-from-scratch.md). After the drain delay, the load balancer has stopped picking you, so the race window is empty. HTTP/2 has a cleaner mechanism for the same thing, a `GOAWAY` frame (`session.goaway()` / `session.close()`), which tells the client to finish its streams and open new ones elsewhere.

---

## 7. Draining queue workers

A worker's "in-flight requests" are its claimed jobs ([L26](../06-data-and-jobs/26-background-jobs-and-queues.md)):

1. **Stop claiming.** The claim loop checks a shutdown `AbortSignal` before every batch, so it never takes work it can't finish.
2. **Finish what's running, within the deadline.** A webhook send has its own 10 s timeout ([L18](../04-networking/18-outbound-http.md)), which fits easily.
3. **At the deadline, abort and release.** Mark the job back to pending, or simply let its lease expire. Either way it runs again elsewhere.
4. **Make the "runs again" safe.** At-least-once delivery means every job handler must be idempotent: the receiver dedupes on the event ID, and the payout processor checks the payout's state before calling the bank. **A graceful shutdown narrows the window for duplicates; only idempotency closes it.**

---

## 8. The PID 1 trap

Inside a container, the process your `CMD` starts is **PID 1** of its PID namespace, and the kernel treats PID 1 specially: **it won't apply a signal's *default* action to it.** A signal is delivered to PID 1 only if PID 1 has installed a handler for it (SIGKILL from outside the container still works).

Now combine that with Node's defaults. With no `SIGTERM` listener, Node's built-in behaviour ends in the default action, which is exactly what the kernel refuses to apply to PID 1. So **Node as PID 1 without a SIGTERM handler ignores SIGTERM**: `docker stop` waits its 10 seconds and sends SIGKILL (exit 137), and Kubernetes waits the full grace period. Your deploys get slow and every pod dies hard, with no drain, no pool shutdown and no log flush.

Two more ways to lose the signal:
- **Shell form:** `CMD node src/main.ts` runs `/bin/sh -c "node src/main.ts"`. The shell is PID 1 and typically doesn't forward SIGTERM to its child; Docker's own documentation warns about this. (Some shells `exec` a lone command and make it work by accident. Don't depend on it.)
- **`npm start` as PID 1:** npm → a shell → node. Whether the signal survives the chain depends on the npm version and the shell, you get npm's exit code instead of yours, and it costs an extra process's memory.

PID 1 has a second job, too: **reaping orphaned zombies** ([L22](22-child-processes.md)). Node only reaps the children it spawned itself.

```dockerfile
# ❌ the shell is PID 1 and node never sees SIGTERM
CMD node src/main.ts

# ❌ npm → sh → node: forwarding depends on versions, and the exit code is npm's
CMD ["npm", "start"]

# ✅ exec form: node is PID 1 and receives SIGTERM directly, so it MUST handle it
CMD ["node", "src/main.ts"]

# ✅ best: tini is PID 1, forwards signals and reaps zombies; node runs as its child
RUN apt-get update && apt-get install -y --no-install-recommends tini && rm -rf /var/lib/apt/lists/*
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["node", "src/main.ts"]
```

`docker run --init` injects tini for local runs; Kubernetes has no such flag, so bake it into the image (Alpine installs it at `/sbin/tini`). Handling the signals in Node is required either way: tini only delivers SIGTERM, and what happens next is your shutdown sequence. The full production Dockerfile is [L33](../08-shipping/33-shipping-node.md)'s.

---

## 9. Ledger: `lifecycle.ts`

One module, used by the API and by every worker process. Each subsystem registers its closer where it's created, with an order number; the module owns signals, readiness, the deadline and the exit.

```ts
// lib/lifecycle.ts
import { setTimeout as sleep } from "node:timers/promises";
import type { Logger } from "./logger.ts";

type Closer = { name: string; order: number; fn: (deadline: AbortSignal) => Promise<void> };
type State = "starting" | "ready" | "draining" | "stopped";

const closers: Closer[] = [];
let state: State = "starting";

export const isReady = (): boolean => state === "ready";
export const isDraining = (): boolean => state === "draining" || state === "stopped";
export function markReady(): void { if (state === "starting") state = "ready"; }

/** Lower `order` closes first; equal orders close in parallel. */
export function onShutdown(name: string, fn: Closer["fn"], order: number): void {
  closers.push({ name, fn, order });
}

export function installSignalHandlers(opts: { drainDelayMs: number; deadlineMs: number; log: Logger }): void {
  let started = false;
  const handle = (signal: NodeJS.Signals): void => {
    if (started) {                                        // second signal: stop waiting
      opts.log.warn({ signal }, "second signal, forcing exit");
      process.exit(1);
    }
    started = true;
    void shutdown(signal, opts);
  };
  process.on("SIGTERM", handle);
  process.on("SIGINT", handle);                           // Ctrl+C: the one Windows really delivers
}

async function shutdown(signal: NodeJS.Signals,
    { drainDelayMs, deadlineMs, log }: { drainDelayMs: number; deadlineMs: number; log: Logger }): Promise<void> {
  const t0 = performance.now();
  state = "draining";                                     // /readyz → 503, Connection: close from now on
  log.info({ signal }, "shutdown: draining");

  const deadline = new AbortController();
  setTimeout(() => deadline.abort(new Error("shutdown deadline")), deadlineMs).unref();
  setTimeout(() => {                                      // the safety net: never keeps the process alive
    log.error({ active: process.getActiveResourcesInfo() }, "shutdown incomplete, forcing exit");
    process.exit(1);
  }, deadlineMs + 2_000).unref();

  await sleep(drainDelayMs);                              // let load balancers stop routing to us

  const orders = [...new Set(closers.map((c) => c.order))].sort((a, b) => a - b);
  for (const order of orders) {
    const group = closers.filter((c) => c.order === order);
    const results = await Promise.allSettled(group.map(async (c) => {
      const start = performance.now();
      await c.fn(deadline.signal);
      log.info({ closer: c.name, ms: Math.round(performance.now() - start) }, "closed");
    }));
    results.forEach((r, i) => {
      if (r.status === "rejected") {
        process.exitCode = 1;                             // keep closing the rest, but report failure
        log.error({ err: r.reason, closer: group[i]?.name }, "closer failed");
      }
    });
  }
  state = "stopped";
  log.info({ ms: Math.round(performance.now() - t0) }, "shutdown complete");
  // No process.exit() here: an empty loop exits by itself, and a leak trips the safety net above
}
```

Wiring in the API (the workers register their own claim loops the same way):

```ts
// main.ts (API): wiring excerpt
const server = app.listen(config.port);
server.keepAliveTimeout = 65_000;                         // > the load balancer's idle timeout (L17)
server.headersTimeout = 66_000;

onShutdown("http", async (deadline) => {
  const closed = new Promise<void>((resolve) => server.close(() => resolve()));   // stop accepting
  sse.closeAll();                                         // `retry:` hint, then end each stream (L20)
  const sweep = setInterval(() => server.closeIdleConnections(), 1_000);
  deadline.addEventListener("abort", () => server.closeAllConnections(), { once: true });
  try { await closed; } finally { clearInterval(sweep); }
}, 10);
onShutdown("outbox-relay", (deadline) => relay.stop(deadline), 20);              // stop claiming, finish batch
onShutdown("postgres", () => pool.end(), 30);
onShutdown("redis", async () => { await redis.quit(); }, 30);
onShutdown("telemetry", async () => { await tracing.shutdown(); await flushLogger(); }, 90);

installSignalHandlers({ drainDelayMs: config.drainDelayMs, deadlineMs: 30_000, log });
markReady();                                              // only after every dependency is up
```

```yaml
# ledger-api pod spec (excerpt)
terminationGracePeriodSeconds: 45        # > 5 s drain + 30 s deadline + margin
containers:
  - name: api
    readinessProbe:  { httpGet: { path: /readyz, port: 3000 }, periodSeconds: 2, failureThreshold: 1 }
    livenessProbe:   { httpGet: { path: /healthz, port: 3000 }, periodSeconds: 10 }   # stays 200 while draining
```

---

## 10. Testing shutdown

A shutdown you haven't tested is a hope. The test spawns the real server as a child process ([L22](22-child-processes.md)), puts requests in flight, sends a real SIGTERM and asserts on what clients saw. **Run it on POSIX** (WSL2 or a Linux container): on Windows, `child.kill("SIGTERM")` is a hard kill and the test proves nothing.

```ts
// test/shutdown.test.ts
test("SIGTERM under load: every accepted request completes, exit code 0", async () => {
  const child = spawn(process.execPath, ["src/main.ts"],
    { env: { ...process.env, PORT: "3999", DRAIN_DELAY_MS: "500" }, stdio: "inherit" });
  await waitForReady("http://localhost:3999/readyz");
  const inFlight = Array.from({ length: 20 }, () =>
    fetch("http://localhost:3999/test/slow?ms=3000").then(async (r) => { await r.text(); return r.status; }));
  await sleep(200);
  child.kill("SIGTERM");
  assert.deepEqual(await Promise.all(inFlight), Array(20).fill(200));   // nothing reset mid-request
  const [code] = await once(child, "exit");
  assert.equal(code, 0);                                                // drained and exited by itself
});
```

Add two more assertions in the same style: `/readyz` returns 503 during the drain delay while `/healthz` still returns 200, and a client holding a keep-alive connection gets `Connection: close` on its next response.

---

## 11. Production rules

| Rule | Why |
|---|---|
| **Handle SIGTERM and SIGINT with one idempotent function** | Adding a listener removes Node's default exit; Ctrl+C on Windows then tests the same path |
| **Fail readiness first, then wait a drain delay before closing anything** | Endpoint removal propagates asynchronously; closing at once turns the stragglers into 502s |
| **Keep liveness passing while draining** | A failing liveness probe gets the pod restarted mid-shutdown |
| **Send `Connection: close` while draining and sweep idle sockets** | Busy keep-alive connections keep serving after `close()` and linger after their last response |
| **End long-lived streams (SSE, WebSockets) explicitly, with a reconnect hint** | They never finish on their own, so `close()` never completes |
| **Close in reverse dependency order: HTTP and jobs, then pools, then telemetry** | In-flight work still needs the database; every step still needs the logger |
| **Put every step under one deadline; `closeAllConnections()` at the deadline** | One stuck connection must not hold the whole pod |
| **Keep an `unref()`'d hard-deadline timer that exits 1** | A leaked handle otherwise hangs until SIGKILL, with no clue why |
| **Exit on the second signal** | The operator or orchestrator has decided not to wait |
| **Set `terminationGracePeriodSeconds` above drain delay + deadline + margin** | Otherwise SIGKILL lands mid-drain |
| **Use exec-form `CMD ["node", …]`, with tini as PID 1** | Shell form and `npm start` lose the signal; PID 1 ignores default actions and doesn't reap zombies |
| **Make every job idempotent anyway** | SIGKILL, OOM kills and node failures skip your shutdown entirely |
| **Test shutdown under load with a real SIGTERM** | It's the code path that runs on every deploy and is never exercised locally |

> **Spring equivalent:** `server.shutdown=graceful` and `spring.lifecycle.timeout-per-shutdown-phase` are steps 3–5 of §5 packaged by the framework, and the preStop sleep plus `terminationGracePeriodSeconds` arithmetic is identical ([SpringBoot L40](https://github.com/Pawar-Pratik/SpringBoot/blob/main/11-deployment/40-docker-cicd-deployment.md)). The ordering of closers is Spring's lifecycle phases and `@PreDestroy`, and the same warning applies: they only run on a clean shutdown, so never rely on them for data integrity ([SpringBoot L08](https://github.com/Pawar-Pratik/SpringBoot/blob/main/02-spring-core/08-scopes-lifecycle-conditional.md)). In Node, nothing is packaged: you write the sequence.

---

## 12. Interview traps

**Q1. "Walk me through graceful shutdown in Kubernetes."**
The pod is marked Terminating; endpoint removal and the preStop/SIGTERM path start concurrently. On SIGTERM: fail readiness and keep serving for a drain delay while proxies catch up; then `server.close()`, `Connection: close` on responses, idle sockets closed and SSE streams ended; in-flight requests finish under a deadline; job loops stop claiming and finish or release their work; pools close; logs and traces flush; the process exits by itself, with a hard-deadline timer as the safety net. All of it fits inside `terminationGracePeriodSeconds` (default 30 s) before SIGKILL. **Follow-up:** "And in the Dockerfile?" Exec-form `CMD` and tini, so the signal reaches Node at all.

**Q2. "Why does my container take 30 seconds to stop?"** (or 10 with `docker stop`)
The process never handled SIGTERM. Usually it's Node running as PID 1 with no handler: the kernel won't apply default signal actions to PID 1, and Node's default ends in exactly that, so the signal is ignored until SIGKILL. Or a shell-form `CMD` or `npm start` swallowed it. Fix: exec-form `CMD`, tini, and a real handler.

**Q3. "Why do requests fail during deploys even though we handle SIGTERM?"**
Endpoint propagation lag. SIGTERM and endpoint removal start at the same moment, so for a second or two proxies keep sending traffic; if the handler closes the listener immediately, those requests hit a closed port. Keep serving for a few seconds after SIGTERM (an in-process drain delay or a preStop sleep). The weak answer is "increase the grace period", which changes nothing about the first two seconds.

**Q4. "What does `server.close()` do with keep-alive connections?"**
It stops accepting new connections and waits for every existing one to close. Idle ones are closed immediately (since Node 19), but a busy keep-alive connection can keep sending new requests and `close()` never completes, and a connection that finishes its last response lingers until the keep-alive timeout. Send `Connection: close` while draining, sweep idle connections, and `closeAllConnections()` at the deadline.

**Q5. "How do you shut down a queue worker safely?"**
Stop claiming new jobs, let running ones finish within a deadline, and abort and release whatever doesn't finish (or let the lease expire). Then close pools. Because delivery is at-least-once, an interrupted job will run again, so the handler must be idempotent; graceful shutdown only makes duplicates rarer.

**Q6. "SIGTERM vs SIGKILL?"**
SIGTERM asks the process to stop and can be caught, so you can drain (exit code 143 if it kills you). SIGKILL can't be caught or ignored: the kernel removes the process, with no handler, no flush and no cleanup (137). Kubernetes sends SIGTERM first and SIGKILL when the grace period ends; the OOM killer only ever sends SIGKILL.

**Q7. "Should `/healthz` fail during shutdown?"**
No. Liveness answers "is this process wedged?"; failing it during a drain gets the container restarted mid-shutdown. Only readiness fails, to take the instance out of rotation.

**Q8. "How do you choose `terminationGracePeriodSeconds`?"**
Drain delay plus your shutdown deadline plus a margin, where the deadline covers your longest legitimate request or job step. Ledger: 5 s + 30 s + margin = 45 s. If a job step can take minutes, split it into resumable steps instead of stretching the grace period.

**Q9. "What's wrong with `process.on('SIGTERM', () => { server.close(); process.exit(0); })`?"**
`server.close()` is asynchronous: `process.exit(0)` runs immediately and kills every in-flight request, drops buffered logs and skips closing pools. It also closes the listener with no drain delay. Wait for the close, under a deadline, and let the process exit by itself.

**Q10. "preStop sleep or an in-process drain delay?"**
Same effect: keep serving while endpoint removal propagates. preStop keeps the app simple but needs a `sleep` binary (or the newer built-in sleep action) and only works on Kubernetes. The in-process delay works anywhere, with distroless images, and can be unit-tested. Either way, budget it into the grace period.

**Q11. "Does graceful shutdown make deploys safe on its own?"**
No. It only runs when you're asked politely. OOM kills, node failures and SIGKILL after a stuck drain skip it entirely, so correctness has to come from idempotency, transactions and at-least-once processing. Graceful shutdown is the fast path; crash-only design is the guarantee.

---

## 13. Build & break

### Build — `lib/lifecycle.ts`
Implement §9 and wire it into the API and the webhook dispatcher. Unit-test it without real signals by exporting the shutdown function for tests: closers run in order, equal orders run in parallel, a failing closer sets `exitCode = 1` and the rest still run, a hanging closer is cut off by the deadline, and a second call forces the exit.

### Build — the shutdown test under load
Implement §10. Then the full version: run `npx autocannon -c 50 -d 20 http://localhost:3999/v1/payments/pi_1` against the spawned server, send SIGTERM at second 5, and record autocannon's error and non-2xx counts. In-flight requests must all complete, and the only failures allowed are connection-refused errors *after* `close()`, which a real load balancer would no longer send you after the drain delay.

### Break — 5 experiments
Run 1 and 2 with Docker Desktop (real Linux signals) and 3–5 in WSL2 or a container.
1. **PID 1 with no handler.** Build an image with exec-form `CMD ["node", "server.ts"]` and no signal handlers. `time docker stop t`: about 10 s, then `docker inspect -f '{{.State.ExitCode}}' t` prints 137. Run it again with `docker run --init`: it stops at once with 143. Then add the handler: it stops at once with 0.
2. **`CMD npm start`.** Same image with `CMD ["npm", "start"]` and your handler logging "shutdown: draining". Does the log line appear on `docker stop`? Note how long the stop takes and which exit code you get. Switch to exec form and compare.
3. **`server.close()` vs keep-alive.** Run the server under `npx autocannon -c 10 -d 60` (keep-alive connections) and call `server.close()` on SIGTERM with no other changes. Throughput doesn't drop and the close callback never fires: the existing connections keep carrying new requests. Add the `Connection: close` middleware and the idle sweep: connections drain within a second or two and the callback fires.
4. **Exit immediately on SIGTERM.** Make the handler `process.exit(0)` and run autocannon with a 200 ms handler: count the errors and timeouts at the signal (every in-flight request). Switch to `lifecycle.ts`: zero errors for requests in flight at SIGTERM.
5. **A worker killed mid-job.** Give the dispatcher a fake 5 s webhook send and `kill -9` it during a send. Restart it: once the lease expires the same delivery is sent again, and your test receiver logs the same event ID twice. Make the receiver dedupe on the event ID ([L26](../06-data-and-jobs/26-background-jobs-and-queues.md)) and repeat: two deliveries, one effect.

### Explain out loud (60–120 seconds)
1. The Kubernetes sequence: concurrent endpoint removal and SIGTERM, the grace period, SIGKILL.
2. Why a drain delay comes first, and where readiness and liveness fit.
3. The ordered shutdown, in reverse dependency order, under one deadline with a safety net.
4. What `server.close()` doesn't do with keep-alive connections, and the three fixes.
5. The PID 1 trap, shell form and `npm start`, and why tini fixes both signals and zombies.

---

## Module 5 complete — checkpoint

- [ ] A worker thread is an isolate with its own heap and loop; I/O doesn't need one, CPU-bound JS does
- [ ] Workers are pooled at startup, never spawned per request, and sized from config
- [ ] Structured clone copies on both threads, drops error subclasses and custom fields; transfer moves an `ArrayBuffer` in O(1) and detaches it
- [ ] `SharedArrayBuffer` without `Atomics` loses updates; logical races at `await` are the common kind
- [ ] A `Worker` with no `'error'` listener takes the process down; running JS is stopped only cooperatively or by `terminate()`
- [ ] `exec` is the only one of the four functions with a shell; args arrays plus `--` and allowlists stop injection
- [ ] A child hangs when nobody reads its pipe; `pipeline` turns the pipe into backpressure
- [ ] `'close'` comes after `'exit'` and after stdio closes; a non-zero exit is not an `'error'`
- [ ] Kill trees with process groups (POSIX) or `taskkill /T` (Windows); orphans, zombies, and BatBadBut
- [ ] Request handling scales with processes: replicas on an orchestrator, cluster or PM2 on a bare VM
- [ ] Cluster's round-robin is per connection and blind to worker health; Windows defaults to `SCHED_NONE`
- [ ] `os.cpus()` reports the host; CPU limits are CFS quotas, and throttling shows up as p99 steps
- [ ] Heaps add up: `--max-old-space-size` per process, all of them within ~75% of the limit
- [ ] In-process state (rate limits, cron, caches, sessions, subscribers) breaks at N > 1
- [ ] The Kubernetes termination race, and the drain delay or preStop sleep that closes it
- [ ] The ordered shutdown, `Connection: close` while draining, and the unref'd safety net
- [ ] PID 1 ignores default signal actions; exec-form `CMD` plus tini

---

## What's next

You can now run Node across threads, processes and replicas, and stop it without losing work. Module 6 turns to the resource every one of those processes fights over: the database. Next is pools, the transaction bug that `pool.query` makes easy to write, and carrying a tenant and a transaction through your code with `AsyncLocalStorage`.

Next → **[Lesson 25: Databases from Node: pools, transactions & context](../06-data-and-jobs/25-databases-from-node.md)**
