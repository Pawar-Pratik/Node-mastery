# Lesson 35 — Ledger Engine build log

> **Why this lesson exists:** the blueprint ([Lesson 34](34-ledger-engine-blueprint.md)) says what to build, and thirty-three lessons say how each piece works. What's left is the order. Build it wrong and you spend a week on infrastructure before anything runs, or you bolt shutdown and metrics onto six processes at the end and find out none of your evidence can be measured. This log is ordered so **something works after every phase**, each phase uses only lessons you've read, and each one ends with **an incident you cause on purpose and recover from**, because the drills are what you'll talk about in interviews.

**Time:** 10 phases · roughly 55–70 hours · **Prereq:** Lesson 34

---

## How to use this

- **One phase at a time, finished.** A phase is done when every "Done when" item works on your machine, not when the code compiles.
- **Never skip the incident drill.** It's the part that turns a feature into a story: "I killed it mid-job and here's what happened" beats "I implemented leases" in every interview.
- **Record every number with the command that produced it**, in `docs/evidence.md`. Loop p99, RSS, failed-request counts, timings. A number without its command can't be defended.
- **Keep `DECISIONS.md`**: one line per choice and its reason. At the end it becomes `docs/engine-architecture.md`, honest rather than reconstructed.
- **Commit per phase**, naming what now works, so your git log reads as a build narrative.
- **Shell:** run commands in Git Bash (its `curl` is the real one; in PowerShell use `curl.exe`). Drills that send POSIX signals run in WSL2, as each one says.

> **Start from the API track's Ledger.** Its [build log](https://github.com/Pawar-Pratik/API-mastery/blob/main/07-project/29-ledger-build-log.md) Phases 0–5 give you the schema, the payment routes, idempotency and the outbox. If you haven't built it, apply the [blueprint's DDL](https://github.com/Pawar-Pratik/API-mastery/blob/main/07-project/28-ledger-blueprint.md) and build only the routes each phase below needs.

---

## The phases at a glance

| Phase | Builds | Incident drill | Evidence it leaves behind |
|---|---|---|---|
| 0 | Workspace, config, lifecycle | The closer that hangs | Ordered shutdown log, exit codes |
| 1 | Loop monitor, errors, guards | "CPU 30%, latency tripled" | p99 before/after, the blocking frame |
| 2 | Raw HTTP, tuned timeouts | The load-balancer 502 | 502 count before/after |
| 3 | Statement export | Slow client, vanishing client | Flat RSS vs the buffered crash; `pg_stat_activity` |
| 4 | Payout-file ingest | Abort at 60%, slow database | Identical row counts; upload rate vs insert speed |
| 5 | Queue, relay, dispatcher | `kill -9` mid-flight; slow DNS | Exactly 1,000 delivered; fs latency vs pool size |
| 6 | Live feed | Lose an instance | Zero missing event IDs |
| 7 | Reconciler on threads | Main thread, dying worker | Loop p99 with and without the pool |
| 8 | SDK and CLI | The bad release | `^0.x` behaviour; resumed `ledger listen` |
| 9 | Containers, telemetry, security | Rollout, leak hunt, OOM pair | Zero failed requests; `leak-hunt.md`; exit 134 vs 137 |

---

## Phase 0 — Workspace, config & lifecycle (4–5 h)

**Lessons:** [L03](../01-foundations/03-the-process.md), [L04](../01-foundations/04-modules-cjs-and-esm.md), [L24](../05-concurrency/24-graceful-shutdown.md)

**Goal:** an engine that boots, validates its config, and shuts down in order, before it does anything else.

- [ ] A pnpm workspace with the [L34 layout](34-ledger-engine-blueprint.md): `apps/api`, `apps/workers`, `packages/shared`, `packages/ledger-node`, `packages/ledger-cli`, `labs/`. Root `package.json` with `"type": "module"`, `engines` and `packageManager`; `.nvmrc` set to 24; the README's `tsconfig.json`.
- [ ] Move the API track's `src/` into `apps/api/src/` and make it run natively: `.ts` extensions on relative imports, `import type` for types, and no parameter properties or enums. `erasableSyntaxOnly` lists every offender; the API track's `CircuitBreaker` and `RetryBudget` both use parameter properties.
- [ ] `packages/shared/src/config.ts`: one zod schema, parsed once, fail fast, and a redacted boot log line.
- [ ] `packages/shared/src/lifecycle.ts`: `onShutdown(name, fn, order)`, SIGTERM and SIGINT wiring, a 25 s deadline on an `unref`'d timer, a second signal forcing exit.
- [ ] `apps/api/src/main.ts` and one stub worker entrypoint, both registering their closers.
- [ ] Scripts: `typecheck` (`tsc --noEmit`) and `test` (`node --test`), run with `node --run`.

```yaml
# pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"
```

**Done when:**
- `PORT=abc node apps/api/src/main.ts` exits with code 1 in under a second, naming `PORT`.
- Ctrl+C (Windows) or `kill -TERM <pid>` (WSL2) logs each closer in order and exits 0.
- An `enum` added anywhere fails `node --run typecheck`.
- `apps/api` imports `@ledger/shared` as `.ts` through the workspace symlink and runs without a build.

**Incident drill — the process that won't die.** Register a closer that never resolves and send SIGTERM: the deadline fires, the log names the closer that hung, and the process exits 1. Then start a script containing an un-`unref`'d `setInterval` and watch it never exit on its own. Those are the two halves of every stuck deploy, and you've now seen both from the inside.

---

## Phase 1 — Loop monitor, errors & process guards (3–4 h)

**Lessons:** [L08](../02-event-loop/08-blocking-the-loop.md), [L09](../02-event-loop/09-errors-and-crashing.md)

**Goal:** you can see the event loop, and every failure takes a deliberate path.

- [ ] `packages/shared/src/loop-monitor.ts`: `monitorEventLoopDelay()` p50/p99/max plus ELU, logged every 5 s and exposed for metrics later.
- [ ] A `loadShedder` middleware: 503 with `Retry-After` while the last second's p99 is above 200 ms.
- [ ] `errors.ts`: the taxonomy (domain → 4xx, operational downstream → 502/503 with `retryable`, programmer → 500) and `toProblem()`; the Express 5 error middleware maps everything through it.
- [ ] `process-guards.ts`: `uncaughtException` and `unhandledRejection` log with request context, flush pino, set `exitCode = 1` and exit after a short timeout.

**Done when:**
- The monitor's line appears every 5 s, and at idle p99 sits near the timer resolution.
- A dev-only `/debug/block?ms=2000` drives p99 above 2 s, and requests arriving in the next second get 503.
- A deliberately floating rejection in a handler produces one structured fatal log line, carrying the request ID, and exit code 1.

**Incident drill — "CPU is 30%, but latency tripled."** Run `npx autocannon -c 50 -d 30 localhost:3000/v1/payments` while a dev route runs a 20 MB `JSON.parse` once a second. Prove the diagnosis from the monitor (p99 and ELU, not CPU%), find the frame with `node --cpu-prof`, fix it at the right layer, and record p99 before and after.

---

## Phase 2 — `node:http` by hand, then the Express API tuned (5–6 h)

**Lessons:** [L17](../04-networking/17-http-from-scratch.md), [L19](../04-networking/19-frameworks-by-mechanism.md)

**Goal:** you know what Express sits on, and the api's server is tuned for a load balancer.

- [ ] `labs/http-raw/server.ts`: a method-and-path router, `readJson(req, { limit, signal })`, problem+json errors, every server timeout set explicitly.
- [ ] On the api's server: `keepAliveTimeout = 65_000`, `headersTimeout = 66_000`, and `requestTimeout` set deliberately. It bounds *receiving* the request, not sending the response, so it never cuts a long export short.
- [ ] A per-request signal middleware: `AbortSignal.any([client disconnected, 20 s deadline])` on `req`, passed into every downstream call.
- [ ] Raw-body capture with a 1 MB limit for idempotency hashing.
- [ ] `labs/frameworks/`: the refund route in Express 5 and in Fastify, passing the same tests.

**Done when:**
- A 2 MB body gets 413 without the server buffering it (RSS doesn't move).
- Closing `curl` mid-request aborts the handler's FakeProcessor call, and the log shows the abort reason.

**Incident drill — the load-balancer 502.** Put a small `node:http` proxy with a keep-alive agent and a 60 s idle timeout in front of the raw server with `keepAliveTimeout` at 5 s. Send bursts 6 s apart and watch the proxy return 502s on reused connections. Set 65 s and repeat: zero. Then open 500 slowloris sockets that drip one header byte every 10 s, and confirm `headersTimeout` closes them.

---

## Phase 3 — The statement export (5–6 h)

**Lessons:** [L13](../03-data-and-io/13-streams-and-backpressure.md), [L10](../02-event-loop/10-cancellation-and-concurrency.md), [L25](../06-data-and-jobs/25-databases-from-node.md)

**Goal:** 4 million rows to a client at constant memory, cancelled when they leave.

- [ ] Seed 4M payments for one merchant by streaming generated rows into `COPY`.
- [ ] `packages/shared/src/db.ts`: the main pool and `exportPool` (`max: 4`), `withTenantTx`, `connectionTimeoutMillis: 2000`, pool metrics (total, idle, waiting).
- [ ] `GET /v1/exports/payments.csv`: try-acquire an export client (503 + `Retry-After` when none is free), a read-only transaction with `SET LOCAL`, `pg-query-stream` → CSV Transform with correct escaping → `createGzip()` → `res`, all in `pipeline(…, { signal })`, and `release(err)` on every path.
- [ ] RSS and heap logged every second during an export.

**Done when:**
- `curl -s localhost:3000/v1/exports/payments.csv -H "Authorization: Bearer $KEY" | gunzip | wc -l` prints 4,000,001 (the header plus 4M rows).
- The per-second RSS log stays flat from the first row to the last; record the peak.
- A fifth concurrent export gets 503 with `Retry-After`, while normal requests keep their latency.

**Incident drill — the slow client and the vanishing client.** Export with `curl --limit-rate 50k`: RSS stays flat, because backpressure travels from the socket back to the cursor. Then press Ctrl+C on that `curl`: within a second the query is gone from `pg_stat_activity`, the export client is back in the pool, and the waiting count is 0.

```bash
# terminal 1: the slow client (Ctrl+C it after ~20 s)
curl -s --limit-rate 50k -H "Authorization: Bearer $KEY" \
  localhost:3000/v1/exports/payments.csv -o /dev/null
# terminal 2: before and after the Ctrl+C
psql "$DATABASE_URL" -c "SELECT pid, state, now() - query_start AS running, left(query, 50)
                         FROM pg_stat_activity WHERE datname = current_database()"
```

 Finally, swap in the buffered version (`rows = await query(all)` then `join`) and record RSS climbing to the crash. **That pair of graphs is your best evidence in the whole project.**

---

## Phase 4 — The payout-file ingest (6–7 h)

**Lessons:** [L14](../03-data-and-io/14-composing-streams.md), [L15](../03-data-and-io/15-the-file-system.md), [L12](../03-data-and-io/12-buffers-and-encodings.md)

**Goal:** a 2 GB upload parsed, validated and inserted with backpressure from the database to the uploader.

- [ ] `apps/api/src/ingest-main.ts`: the ingester's own HTTP server, config and lifecycle (60 s drain).
- [ ] `POST /v1/payout-files` → a `payout_files` row; `PUT /v1/payout-files/:id/content` streamed through `splitLines` (with leftover handling) → `parseCsv` → `validate` (zod) → `batch(1000)` → `insertBatches` (one transaction per batch, `ON CONFLICT (file_id, line_no) DO NOTHING`, with `rows_inserted` and `last_line_no` updated in the same transaction).
- [ ] Bad rows go to `rejects.csv` inside `withTempDir()`; `packages/shared/src/fs-safe.ts` provides `writeFileAtomic`, `safeJoin` and `withTempDir`.
- [ ] A generator script that writes a 2 GB file with known bad rows at known line numbers, including a multi-byte character split across a chunk boundary.

**Done when:**
- The 2 GB file ingests at flat RSS, and `rows_inserted` plus the lines in `rejects.csv` equals `wc -l` minus the header.
- `rejects.csv` lists exactly the planted bad lines, and the temp directory is gone afterwards.

**Incident drill — the abort and the slow database.** Press Ctrl+C on the upload at about 60%: the temp directory is removed, the file is marked `failed`, and committed batches stay. Re-`PUT` the same file and compare the final row count with a clean run: identical, because the re-upload is idempotent. Then add a 200 ms sleep inside `insertBatch` and upload again: `curl`'s progress meter shows the upload rate falling while the ingester's RSS stays flat. **The database just slowed down the client's TCP window, through five stages.**

---

## Phase 5 — The queue, outbox relay, dispatcher & sender (8–10 h)

**Lessons:** [L10](../02-event-loop/10-cancellation-and-concurrency.md), [L18](../04-networking/18-outbound-http.md), [L26](../06-data-and-jobs/26-background-jobs-and-queues.md), [L07](../02-event-loop/07-the-thread-pool.md)

**Goal:** webhooks that survive anything you do to the process sending them.

- [ ] `concurrency.ts`: `limit(n)`, `keyedLimit(n)`, `mapLimit`, `withDeadline(signal)`.
- [ ] `http-client.ts`: one undici `Agent` with per-origin connection caps and an idle timeout, connect, headers and body timeouts, a 10 s total deadline, a 2 KB capped body read that always consumes or cancels, and the SSRF-safe lookup with a TTL-bounded DNS cache.
- [ ] `webhooks/sender.ts`: HMAC over `timestamp.rawBody`, the `Ledger-Signature` header.
- [ ] `outbox-relay.ts`: `LISTEN outbox` with a 5 s fallback poll, a `SKIP LOCKED` batch, deliveries created, `NOTIFY ledger_events`.
- [ ] `dispatcher.ts`: the lease claim (`next_attempt_at` + 60 s), global 200 and 5 per endpoint, a fenced completion (the attempt number is incremented at claim time), L26's `BACKOFF_SEC` schedule with jitter (8 attempts, ~3.4 days), `failed` when the receiver rejects, `dead` plus auto-disable when attempts run out.
- [ ] `sweeper.ts` under `pg_try_advisory_lock`; `labs/receiver/`: a test receiver that verifies signatures and dedupes on event ID.

```sql
-- the claim: a lease, and the attempt number becomes the fencing token for completion (L26)
UPDATE webhook_deliveries
   SET attempt = attempt + 1, next_attempt_at = now() + interval '60 seconds'
 WHERE id IN (SELECT id FROM webhook_deliveries
               WHERE status = 'pending' AND next_attempt_at <= now()
               ORDER BY next_attempt_at
               LIMIT $1                          -- the dispatcher's free capacity, never more
               FOR UPDATE SKIP LOCKED)
RETURNING id, endpoint_id, event_id, attempt;
```

**Done when:**
- Creating a payment produces a signed webhook that the receiver verifies within about a second.
- With the receiver stopped, attempts follow the schedule; a `node:test` suite with mock timers asserts the schedule without sleeping ([L31](../08-shipping/31-testing-node.md)).
- An endpoint that hangs for 10 s doesn't slow delivery to any other endpoint.
- Three sweeper replicas log exactly one active leader.

**Incident drill — `kill -9` mid-flight (WSL2).** Queue 1,000 deliveries to a receiver that takes 2 s per request, then `docker kill -s KILL` the dispatcher mid-batch. Restart it: after the 60 s lease the in-flight rows are claimed again, the receiver logs duplicate event IDs and dedupes them, and the final `delivered` count is exactly 1,000. **Second drill, slow DNS:** give one hostname a custom `lookup` that waits 2 s, keep `UV_THREADPOOL_SIZE` at 4, and measure `fs.readFile` latency in the same process climbing; then add the DNS cache, set the pool to 16 and measure again.

---

## Phase 6 — The live feed (5–6 h)

**Lessons:** [L20](../04-networking/20-realtime-servers.md), [L11](../03-data-and-io/11-eventemitter.md)

**Goal:** the server half of the Console's live payments, correct across instances.

- [ ] `realtime/sse.ts`: a hub keyed by merchant; `id:`/`event:`/`data:` framing; one 20 s heartbeat timer per hub; eviction above 256 KB of `writableLength`; a cap of 1,000 connections with 503 beyond it; `X-Accel-Buffering: no`; the route excluded from compression.
- [ ] `realtime/listen-bridge.ts`: a dedicated `pg.Client` running `LISTEN ledger_events`, reconnecting and re-listening after an error, with paged reads per merchant and a small overlap behind `lastRead`.
- [ ] `Last-Event-ID` resume, paged and backpressure-aware.
- [ ] On shutdown, send `retry:` and end every stream.

**Done when:**
- `curl -N localhost:3000/v1/events/stream -H "Authorization: Bearer $KEY"` shows a new payment within about a second, and so does the Console.
- Reconnecting with `Last-Event-ID` replays exactly the gap.
- You've measured heap per idle subscriber with 1,000 connections open and written it into `docs/budgets.md`.

**Incident drill — lose an instance.** Run two api instances behind a local proxy with 200 SSE clients, then kill one. Clients reconnect to the survivor and resume; a checker script compares the event IDs each client received against the `events` table for its merchant and finds none missing. Then connect a client that stops reading: it's evicted at 256 KB, and RSS stays flat.

---

## Phase 7 — The reconciler on worker threads (6–7 h)

**Lessons:** [L21](../05-concurrency/21-worker-threads.md), [L26](../06-data-and-jobs/26-background-jobs-and-queues.md)

**Goal:** CPU-heavy matching that never stalls the process's own event loop.

- [ ] A generator for a 500 MB, 1M-line settlement file with planted mismatches (missing, extra, wrong amount).
- [ ] `labs/worker-pool/`: L21's hand-rolled pool, built once to understand it; `reconcile/run.ts` uses piscina with `RECONCILE_THREADS` from config (`0` runs the matching in-thread, for the drill only).
- [ ] `reconcile/worker.ts`: decode → parse → validate → normalise, returning small records.
- [ ] `reconcile/run.ts`: the expected-entries `Map` loaded through a cursor; 1 MiB chunks cut at the last newline and transferred; at most `2 × threads` in flight; results in a transaction per batch with `ON CONFLICT DO NOTHING`; lease heartbeats; `/healthz` and the loop monitor running.

**Done when:**
- Every planted mismatch is reported, and nothing else.
- The reconciler's main-thread loop p99 stays under 50 ms during a run; record it.
- The api's p99 under autocannon is unchanged while a run is in progress (two runs, side by side).
- You've measured heap per `Map` entry and checked the main thread's share against its 384 MiB.

**Incident drill — the main thread, then a dying worker.** Run the matching in-thread (`RECONCILE_THREADS=0`) and watch `curl --max-time 2 localhost:9100/healthz` time out while loop p99 reaches seconds: in Kubernetes the liveness probe would restart the pod mid-run and the lease would let it run twice. Switch to the pool and repeat. Then make a worker throw on line 600,000: the error reaches the main thread, the job is released, and the rerun produces exactly the same results table as a clean run.

---

## Phase 8 — The SDK and the CLI, published (6–7 h)

**Lessons:** [L32](../08-shipping/32-packaging-and-publishing.md), [L12](../03-data-and-io/12-buffers-and-encodings.md), [L14](../03-data-and-io/14-composing-streams.md)

**Goal:** two packages another developer could install today.

- [ ] `@ledger/node`: the client core (`fetch`, full-jitter retries honouring `Retry-After`, an idempotency key per logical call, `AbortSignal` on every method, `LedgerError`), the resources, auto-pagination as an async generator, and `webhooks.verify()`.
- [ ] `exports` with `types` first; `tsc` emitting `.js` + `.d.ts` with `rewriteRelativeImportExtensions`; `publint`, `@arethetypeswrong/cli` and `npm pack --dry-run` clean; changesets for versioning.
- [ ] `ledger`: `bin` and a shebang, `util.parseArgs`, `ledger listen --forward-to <url>` (SSE over `fetch`, re-signing with a session secret, reconnecting with `Last-Event-ID`), and `ledger trigger <event>` in test mode.
- [ ] A local registry (Verdaccio) to publish to.

**Done when:**
- A fresh ESM project and a CommonJS project on Node 22.12+ (`require(esm)`) both install `@ledger/node` from Verdaccio, and their types resolve.
- `ledger trigger payment.succeeded` arrives at a local receiver through `ledger listen --forward-to` and passes `webhooks.verify()`.

**Incident drill — the bad release.** Publish `0.3.0` with a deliberate bug to Verdaccio. A consumer on `^0.2.3` doesn't receive it (explain exactly why), while one on `^0.3.0` does. Publish `0.3.1` with the fix and `npm deprecate @ledger/node@0.3.0 "<reason>"`. Then restart the api under a running `ledger listen`: it reconnects, resumes and forwards each missed event exactly once.

---

## Phase 9 — Production: containers, telemetry, load, leaks, security (8–10 h)

**Lessons:** [L27](../07-production/27-memory-and-gc.md), [L28](../07-production/28-performance-and-v8.md), [L29](../07-production/29-observability.md), [L30](../08-shipping/30-security-and-supply-chain.md), [L33](../08-shipping/33-shipping-node.md)

**Goal:** deployed, observable, load-tested, and defended with numbers.

- [ ] `deploy/Dockerfile`, `deploy/compose.yaml` and `deploy/k8s/` for all six process types, with every budget from [L34](34-ledger-engine-blueprint.md) in the manifests; a local cluster with kind or minikube.
- [ ] `packages/shared/src/telemetry/`: pino with redaction, ALS context, OTel loaded with `--import`, runtime metrics, the L34 alert rules and a dashboard.
- [ ] `bench/`: autocannon scripts for the hot routes, and `profile.sh` running `--cpu-prof` under load. Fix one hot frame and record p99 and RPS before and after.
- [ ] Security pass: frozen install with an allowlist for install scripts; prototype-pollution tests on the `metadata` merge; the reconciler running under `--permission` with explicit allows; Renovate with a release-age delay; provenance on the SDK's publish workflow.

**Done when:**
- From a clean clone, `docker compose up` brings up the whole stack and every healthcheck goes green.
- The cluster deployment is healthy, and the dashboard shows loop p99, ELU, heap vs limit, pool waiting and queue age.

**Incident drills — this phase earns three.**
1. **A rolling deploy under load.** Run autocannon for 60 s against the api in kind while `kubectl rollout restart deployment/ledger-api` runs, and count non-2xx responses: zero. Remove the preStop sleep and the SIGTERM handler, repeat, and record the failures. Both numbers go into the evidence. Send the load through the ingress, never through `kubectl port-forward`, which pins one pod and dies with it:
   ```bash
   npx autocannon -c 50 -d 60 -H "Authorization=Bearer $KEY" http://ledger.localhost/v1/payments &
   kubectl rollout restart deployment/ledger-api && kubectl rollout status deployment/ledger-api
   wait    # autocannon's summary reports the non-2xx count
   ```
2. **A leak hunt.** Plant the per-merchant `Agent` map that never evicts ([L27](../07-production/27-memory-and-gc.md)), drive it with rotating merchant IDs for ten minutes, take three heap snapshots, follow the retaining path, fix it, and write `docs/leak-hunt.md` with the before/after heap graph.
3. **The OOM pair.** Give the reconciler `--max-old-space-size=1024` with four isolates in its 2Gi pod and feed it an oversized `Map`: `OOMKilled`, exit 137, an empty log. Restore the budgeted 384 and repeat: a V8 heap error with a stack and a diagnostic report.

---

## The build order, and why

| Phase | Why here |
|---|---|
| 0 Workspace & lifecycle | Every later process registers closers and reads config; retrofitting shutdown into six processes is an audit |
| 1 Loop monitor & errors | Without the monitor, nothing in Phases 3–9 can be proven, only claimed |
| 2 HTTP & timeouts | The export and the feed are long-lived responses on this server |
| 3 Export | The simplest stream with the most striking evidence; it needs only the pool and a cursor |
| 4 Ingest | The same stream skills in the other direction, plus files and idempotent writes |
| 5 Queue & dispatcher | Needs the outbox rows the api writes and the HTTP client discipline from Phase 2 |
| 6 Live feed | Needs the relay's `NOTIFY` from Phase 5 |
| 7 Reconciler | Needs L26's job queue from Phase 5 for its lease |
| 8 SDK & CLI | Needs a stable API and the live feed for `ledger listen` |
| 9 Production | Needs every process to exist before it can be deployed, measured and attacked |

**The two phases people get wrong:** they skip **Phase 0's lifecycle** ("I'll add shutdown later"), and every later drill that kills a process becomes unmeasurable; and they skip **Phase 1's monitor**, so in Phase 7 they *believe* the pool helped instead of showing the loop p99 that proves it.

---

## If you have less time

| Available | Build phases | You still get |
|---|---|---|
| **~15 h** | 0, 1, 3 | Boot and shutdown discipline, loop metrics, and the 4M-row export: **the most Node-specific evidence in the project** |
| **~30 h** | + 5 | The webhook dispatcher, with the `kill -9` drill |
| **~45 h** | + 4, 6, 7 | All three data pipelines and the live feed |
| **~65 h** | + 2, 8, 9 | Tuned HTTP, a published SDK and CLI, deployed with evidence |

**If you build only one thing, build Phase 3 with its drill.** A flat RSS graph across 4 million rows, next to the buffered version's climb to a crash, plus the `pg_stat_activity` proof that the query died with the client, answers two of the README's mastery questions with your own measurements.

---

## What's next

You've built it. Module 10 turns the track and the project into interview performance, starting with a rapid-fire question bank across every module, so the mechanisms come out as reflexes.

Next → **[Lesson 36: Rapid-fire master Q&A](../10-interview/36-rapid-fire-master-qa.md)**
