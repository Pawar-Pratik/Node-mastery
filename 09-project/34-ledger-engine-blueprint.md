# Lesson 34 — Ledger Engine blueprint

> **Why this lesson exists:** thirty-three lessons of runtime mechanism need one system to land in. **Ledger Engine** is the set of Node processes that run Ledger in production, plus the SDK and CLI that developers install. This blueprint fixes every process, data flow, budget and decision with its arithmetic and its reason, so that when an interviewer says *"walk me through something you built"*, every follow-up question has an answer you've already measured.

**Time:** ~60 minutes to read and internalise · **Prereq:** Modules 1–8

---

## 1. What Ledger Engine is

The [API track](https://github.com/Pawar-Pratik/API-mastery/blob/main/07-project/28-ledger-blueprint.md) designed Ledger's contract and schema, and the [React track](https://github.com/Pawar-Pratik/React-mastery/blob/main/08-project/26-console-blueprint.md) built the Console on top of it. Ledger Engine is **the runtime layer underneath**: the api process understood below Express, five worker and pipeline processes, and two published artefacts, `@ledger/node` (a typed ESM SDK) and `ledger` (a CLI with `ledger listen --forward-to`). The domain, the tables and the HTTP contract are the API track's; everything here is about what the Node processes do with them.

| The engine forces… | …which teaches |
|---|---|
| Exporting 4M payments to one merchant | **Constant-memory streams, cancelled on disconnect** ([L13](../03-data-and-io/13-streams-and-backpressure.md), [L10](../02-event-loop/10-cancellation-and-concurrency.md)) |
| 2 GB payout files uploaded over HTTP | **Composed pipelines, backpressure to the socket** ([L14](../03-data-and-io/14-composing-streams.md), [L15](../03-data-and-io/15-the-file-system.md)) |
| Merchant endpoints that are slow, dead or hostile | **Bounded concurrency, pooled undici, an SSRF guard** ([L10](../02-event-loop/10-cancellation-and-concurrency.md), [L18](../04-networking/18-outbound-http.md)) |
| Webhooks that must survive restarts | **A `SKIP LOCKED` queue with leases, at-least-once** ([L26](../06-data-and-jobs/26-background-jobs-and-queues.md)) |
| Console tabs watching payments live | **SSE at scale, fan-out with `LISTEN/NOTIFY`** ([L20](../04-networking/20-realtime-servers.md)) |
| CPU-bound settlement matching | **Worker threads, bytes transferred** ([L21](../05-concurrency/21-worker-threads.md)) |
| Deploys several times a day | **Graceful shutdown, PID 1** ([L24](../05-concurrency/24-graceful-shutdown.md), [L33](../08-shipping/33-shipping-node.md)) |
| One process per pod, with hard limits | **Pool, thread and memory budgets with arithmetic** ([L07](../02-event-loop/07-the-thread-pool.md), [L23](../05-concurrency/23-cluster-and-scaling.md), [L25](../06-data-and-jobs/25-databases-from-node.md), [L27](../07-production/27-memory-and-gc.md)) |
| Developers integrating against you | **A published ESM SDK and a CLI** ([L32](../08-shipping/32-packaging-and-publishing.md)) |
| Thousands of transitive packages | **Supply-chain posture** ([L30](../08-shipping/30-security-and-supply-chain.md)) |

The card processor stays mocked (the API track's `FakeProcessor`), for the reasons that blueprint gives.

---

## 2. Process topology

```text
  Console (React)          Merchant servers             Payout files
  SSE feed + REST          REST + Idempotency-Key       PUT …/content, 2 GB
         │                        │                             │
         └───────────┬────────────┘                             │
                     ▼                                          ▼
  ┌──────────────────────────────────────┐        ┌────────────────────────────┐
  │ api ×4 (Express 5)                   │        │ ingester ×2                │
  │ REST · idempotency · rate limits     │        │ split → parse → validate   │
  │ exports: cursor → CSV → gzip → res   │        │ → batch(1000) → insert     │
  │ SSE hub · LISTEN ledger_events       │        │ rejects.csv in /tmp        │
  └──────┬───────────────────────┬───────┘        └─────────────┬──────────────┘
         │ tx: payment,          │ charge, capture              │ tx per batch
         │ entries, event,       ▼                              │
         │ outbox; NOTIFY   ┌───────────────┐                   │
         │                  │ FakeProcessor │                   │
         │                  └───────────────┘                   │
         ▼                                                      ▼
  ┌────────────────────────────────────────────────────────────────────────────┐
  │ Postgres 16 (RLS): payments · ledger_entries · events · outbox ·           │
  │ webhook_deliveries · idempotency_keys · payout_items · jobs                │
  └──────▲─────────────────▲─────────────────────▲────────────────────▲────────┘
         │                 │                     │                    │
  ┌──────┴───────┐  ┌──────┴────────┐  ┌─────────┴──────────┐  ┌──────┴────────┐
  │ outbox-relay │  │ dispatcher ×2 │  │ reconciler ×1      │  │ sweeper ×2    │
  │ ×2           │  │ undici Agent  │  │ main thread +      │  │ advisory lock │
  │ deliveries + │  │ SSRF lookup   │  │ 3 worker threads   │  │ one active    │
  │ NOTIFY       │  │ HMAC · 10 s   │  │ (piscina)          │  │               │
  └──────────────┘  └──────┬────────┘  └────────────────────┘  └───────────────┘
                           │ signed POST, at least once
                           ▼
                  Merchant webhook endpoints
```

Redis 7 holds the api's rate-limit buckets. A nightly CronJob runs the backup (`pg_dump` through L22's safe runner).

| Process | Kind | Entrypoint | Replicas | Owns |
|---|---|---|---|---|
| **api** | HTTP | `apps/api/src/main.ts` | 4, HPA 4–8 on ELU | REST, idempotency, statement exports, the SSE hub, one `LISTEN` connection |
| **ingester** | HTTP | `apps/api/src/ingest-main.ts` | 2 | Payout-file uploads |
| **outbox-relay** | Worker | `apps/workers/src/outbox-relay.ts` | 2 (`SKIP LOCKED` makes two safe) | Outbox → deliveries, plus the live-feed `NOTIFY` |
| **dispatcher** | Worker | `apps/workers/src/dispatcher.ts` | 2, HPA 2–4 on oldest-due age | Webhook sends |
| **reconciler** | Worker + threads | `apps/workers/src/reconciler.ts` | 1, `strategy: Recreate` | Settlement matching |
| **sweeper** | Worker | `apps/workers/src/sweeper.ts` | 2, one active via an advisory lock | Idempotency-key TTL, delivery retention |

**One image, six process types.** Every Deployment runs the same digest and picks its entrypoint with `args` ([L33](../08-shipping/33-shipping-node.md)), so the api and the workers can never run different versions of the schema code. **The ingester is a separate Deployment** even though it's HTTP: a 2 GB upload holds a socket and a database client for minutes and pushes batch inserts, and isolating it keeps the api's pool, event loop and memory for payments. It also gets its own grace period (75 s) without slowing api rollouts.

---

## 3. Data flows

### 3.1 Create payment → outbox → webhook

```text
merchant ─POST /v1/payments (Idempotency-Key)─► api
  withTenantTx(merchantId, fn):  SET LOCAL app.merchant_id, client carried by ALS
    INSERT payments · ledger_entries (balanced) · events · outbox
    SELECT pg_notify('outbox', '')                  ← delivered only if the transaction commits
  COMMIT ─► 201
outbox-relay  (woken by NOTIFY, 5 s fallback poll)
  BEGIN; SELECT … FROM outbox WHERE processed_at IS NULL … FOR UPDATE SKIP LOCKED
    INSERT webhook_deliveries per matching endpoint   (UNIQUE (endpoint_id, event_id))
    UPDATE outbox SET processed_at = now()
    SELECT pg_notify('ledger_events', '{"m":"<merchant>","seq":<highest sequence>}')
  COMMIT
dispatcher
  claim:    UPDATE … SET next_attempt_at = now() + 60 s … FOR UPDATE SKIP LOCKED   ← the lease
  send:     keyedLimit(5) inside limit(200) → undici, SSRF-safe lookup, HMAC, 10 s deadline, 2 KB capture
  complete: fenced on the attempt number (incremented at claim time)
            → delivered | failed (the receiver rejected it) | pending again after BACKOFF_SEC[attempt]
              with jitter | dead once attempt 8 fails, + auto-disable
```

L26's schedule is `BACKOFF_SEC = [60, 300, 1_800, 7_200, 21_600, 86_400, 172_800]`: seven gaps between eight attempts, about 3.4 days end to end.

**`NOTIFY` inside the transaction is delivered at commit or never**, so no listener ever hears about a payment that rolled back. The relay holds its transaction for the whole batch because its work is database-only; the dispatcher uses a lease because its work is network I/O, and **no database client is held across a send** ([L26](../06-data-and-jobs/26-background-jobs-and-queues.md), [L25](../06-data-and-jobs/25-databases-from-node.md)). The claim's `LIMIT` is the dispatcher's free capacity, never more.

### 3.2 The statement export

```text
GET /v1/exports/payments.csv?created[gte]=… ─► api
  try-acquire an exportPool client (max 4 per replica = the export cap) → 503 + Retry-After if none
  BEGIN READ ONLY; SET LOCAL app.merchant_id; SET LOCAL statement_timeout
  pipeline(QueryStream(cursor, 1,000 rows) → toCsv (Transform) → createGzip() → res, { signal })
  signal = AbortSignal.any([client disconnected, export deadline, shutdown])
  any error or abort → every stage destroyed → cursor closed → release(err) → ROLLBACK
```

Memory is the sum of the stage buffers, independent of row count. **A slow client pauses the cursor through backpressure; a gone client kills the query** ([L13](../03-data-and-io/13-streams-and-backpressure.md)). Pre-generated statements take a different path: files written atomically to disk and served from `GET /v1/statements/:id` through L15's `safeJoin` ([L15](../03-data-and-io/15-the-file-system.md)).

### 3.3 The payout-file ingest

```text
POST /v1/payout-files ─► ingester: payout_files row (receiving) ─► 201 { id }
PUT  /v1/payout-files/:id/content (2 GB, streamed) ─► ingester
  pipeline(req → splitLines → parseCsv → validate (zod) ──bad rows──► rejects.csv (withTempDir)
                → batch(1000) → insertBatch: one tx, ON CONFLICT (file_id, line_no) DO NOTHING,
                  rows_inserted and last_line_no updated in the same tx)
  slow inserts → the batch stage awaits → buffers fill → req pauses → TCP window closes → the client slows
```

The upload is a `PUT` to a resource that already exists, and every line is keyed by `UNIQUE (file_id, line_no)`, so **re-sending a file after an abort is idempotent**: committed batches are skipped, and the result is identical to a clean run ([L14](../03-data-and-io/14-composing-streams.md), [L15](../03-data-and-io/15-the-file-system.md)).

### 3.4 The live feed

```text
relay COMMIT ─NOTIFY ledger_events {m, seq}─► every api instance's dedicated LISTEN client
  hub: subscribers for merchant m here?  no → ignore
       yes → SELECT … FROM events WHERE merchant_id = m AND sequence > lastRead ORDER BY sequence (paged)
           → write "id: <seq>" / "event: <type>" / "data: <json>" frames to each subscriber
  per subscriber: writableLength > 256 KB → evict; it reconnects with Last-Event-ID and replays
```

The payload carries IDs, never event bodies (the ~8,000-byte limit), and **eviction is safe because resume is lossless**: the database is the buffer ([L20](../04-networking/20-realtime-servers.md)). One edge to know: a `bigserial` is assigned at insert, not at commit, so an event can commit after a higher sequence was already sent. The hub re-reads a small overlap behind `lastRead`, and the Console dedupes by event ID ([React L19](https://github.com/Pawar-Pratik/React-mastery/blob/main/05-data/19-realtime.md)).

### 3.5 Reconciliation

```text
nightly: a `reconcile` job on L26's queue (leased; heartbeats extend the lease)
reconciler main thread
  expected entries for the window via a pg cursor → Map<processor_ref, { amount_minor: bigint, currency }>
  settlement.csv in 1 MiB chunks → cut at the last '\n' → transfer the ArrayBuffer to piscina
workers: decode → parse → validate → normalise → small records back
main thread: join against the Map → matched · missing · extra · amount_mismatch
             → reconciliation_results (one tx per batch, ON CONFLICT DO NOTHING)
≤ 2 × threads chunks in flight → the file read pauses when the workers fall behind
```

State lives in one place (the `Map` on the main thread), bytes move by transfer, and a restarted run rewrites nothing twice ([L21](../05-concurrency/21-worker-threads.md)).

### The Node-track schema additions

```sql
-- the ingester's tables, as L14 defines them (abbreviated)
CREATE TABLE payout_files (
  id            text PRIMARY KEY,
  merchant_id   uuid NOT NULL REFERENCES merchants(id),
  status        text NOT NULL,                    -- receiving|completed|failed
  rows_inserted int  NOT NULL DEFAULT 0,
  last_line_no  int  NOT NULL DEFAULT 0,          -- progress, committed with each batch
  created_at    timestamptz NOT NULL DEFAULT now()
);
CREATE TABLE payout_items (
  file_id       text   NOT NULL REFERENCES payout_files(id),
  line_no       int    NOT NULL,
  merchant_id   uuid   NOT NULL,                  -- RLS, like every tenant table
  amount_minor  bigint NOT NULL CHECK (amount_minor > 0),
  currency      char(3) NOT NULL,
  -- … destination and the remaining columns from L14
  UNIQUE (file_id, line_no)                       -- makes a re-upload idempotent
);
CREATE TABLE reconciliation_results (
  job_id        text NOT NULL,
  processor_ref text NOT NULL,
  outcome       text NOT NULL,                    -- matched|missing|extra|amount_mismatch
  detail        jsonb,
  PRIMARY KEY (job_id, processor_ref)             -- makes a restarted run idempotent
);
```

---

## 4. Budgets, with the arithmetic

The numbers below are **planning targets you replace with measurements** in the build log; the arithmetic is the part you defend.

### Postgres connections

| Process | Replicas | Main pool `max` | Other connections | Per replica | Total |
|---|---|---|---|---|---|
| api | 4 | 10 | exportPool 4 + 1 `LISTEN` | 15 | 60 |
| ingester | 2 | 4 | – | 4 | 8 |
| outbox-relay | 2 | 2 | 1 `LISTEN` | 3 | 6 |
| dispatcher | 2 | 5 | – | 5 | 10 |
| reconciler | 1 | 4 | – | 4 | 4 |
| sweeper | 2 | 2 | – | 2 | 4 |
| **Steady state** | | | | | **92** |

```text
steady state                                                    92
+ api at its HPA max (8 replicas × 15)                         +60   → 152
+ dispatcher at its HPA max (4 replicas × 5)                   +10   → 162
+ a simultaneous rollout surge (maxSurge 1: 15 + 4 + 3 + 5 + 2) +29   → 191
usable: max_connections 250 − 3 superuser-reserved − 20 for the migrate Job,
        psql sessions, the metrics exporter and the nightly pg_dump  = 227     191 ≤ 227 ✅
```

**Why not bigger pools?** Little's law. At a fleet target of 1,000 req/s with a mean database hold of 8 ms, 1,000 × 0.008 = **8 connections are busy on average across the whole fleet**; per api replica that's 250 × 0.008 = 2, so `max: 10` is five times headroom. Postgres does the work with about eight active backends, and the rest is idle capacity, which is why larger pools usually make things slower ([L25](../06-data-and-jobs/25-databases-from-node.md)). Every pool sets `connectionTimeoutMillis: 2000` (a 503, never a hang), `statement_timeout` 5 s and `idle_in_transaction_session_timeout` 10 s. The export pool is separate because **an export holds its client for the whole stream**, so four exports can never take request-path connections.

### The libuv thread pool

| Process | `UV_THREADPOOL_SIZE` | What uses the pool there | Arithmetic |
|---|---|---|---|
| api | 6 | gzip for exports, scrypt for Console logins, `dns.lookup`, a little `fs` | 4 export streams, each compressing one chunk at a time, plus 2 so logins, DNS and fs never queue behind gzip |
| dispatcher | 16 | `dns.lookup` for merchant hosts, behind a TTL-bounded cache | 40 uncached lookups/s × 0.3 s worst case = 12 threads busy, plus headroom |
| everything else | 4 (default) | `fs` streams | The default is enough |

The api's pool work is mostly CPU (gzip, scrypt), so threads beyond its CPU allowance buy nothing, and **that CPU comes out of the same cgroup quota as the event loop**; that's why exports are capped at four per replica. The dispatcher's pool work is pure waiting, so threads beyond cores are cheap, and they're needed because `getaddrinfo` can't be cancelled: a lookup that has already timed out from your point of view still holds its thread until the resolver gives up ([L07](../02-event-loop/07-the-thread-pool.md)). `ndots: 2` in its pod spec stops each lookup from walking the cluster search domains first ([L33](../08-shipping/33-shipping-node.md)).

### The reconciler's threads and heap

```text
CPU limit 4 → 1 main thread (file reads, the join, DB I/O, /healthz, lease heartbeats) + 3 worker threads
RECONCILE_THREADS=3 from config — never os.cpus().length, which reports the node's cores, not the quota
in flight: ≤ 2 × 3 = 6 chunks of 1 MiB
heap: 4 isolates × --max-old-space-size=384 = 1,536 MiB = 75% of the 2 GiB limit
```

The flag applies to every isolate and overrides `resourceLimits`, so the budget counts four ceilings even though the workers use little of theirs. The main thread's `Map` of expected entries is the big consumer: **measure its heap per entry in Phase 7**, and if it outgrows 384 MiB, raise the container limit and the flag together.

### Dispatcher concurrency

```text
slots: 2 replicas × global 200 = 400; per endpoint 5; per attempt 10 s
planning peak: 500 deliveries/s × 0.3 s mean endpoint latency = 150 slots busy (37%)
ceiling at normal latency: 400 / 0.3 s ≈ 1,300 deliveries/s
a dead endpoint holds ≤ 5 slots for ≤ 10 s → 400 / 5 = 80 endpoints must time out at once to saturate the fleet
lease 60 s ≫ the 10 s attempt deadline → a crashed dispatcher's rows come back after 60 s, never during a send
```

### Memory per process

| Process | Limit | `--max-old-space-size` | Share | What lives off-heap |
|---|---|---|---|---|
| api | 1Gi | 768 | 75% | Export stream buffers, request bodies (1 MB cap), socket buffers |
| ingester | 768Mi | 512 | 67% | Upload chunks, the rejects file's write buffer |
| outbox-relay | 256Mi | 192 | 75% | Little |
| dispatcher | 512Mi | 384 | 75% | 200 in-flight bodies and 2 KB captures |
| reconciler | 2Gi | 384 per isolate × 4 | 75% | 1 MiB file chunks in transit |
| sweeper | 256Mi | 192 | 75% | Little |

Memory requests equal limits, and every entrypoint logs a warning at boot if the heap limit exceeds 80% of the container limit ([L33](../08-shipping/33-shipping-node.md), [L27](../07-production/27-memory-and-gc.md)).

### SSE connections per api instance

```text
per subscriber: ≤ 256 KB buffered before eviction (L20) + a planning assumption of ≤ 16 KB idle heap
budget for SSE per instance: ~300 MB of the 768 MB heap
cap: 300 MB / 272 KB ≈ 1,100 → 1,000 connections per instance; beyond it, 503 + Retry-After
fleet: 4 × 1,000 = 4,000 concurrent Console tabs; 8,000 at the HPA max
one heartbeat timer per hub (every 20 s, unref'd), never one per connection
```

The Console's client reconnects with jittered backoff and resumes ([React L19](https://github.com/Pawar-Pratik/React-mastery/blob/main/05-data/19-realtime.md)), so a 503 just moves it to another instance. **If you need more connections, lower the per-client byte cap before you raise memory**: eviction costs a replay, not data.

---

## 5. The lifecycle

Every process uses the same `lifecycle.ts` from [L24](../05-concurrency/24-graceful-shutdown.md): closers registered in dependency order, an `unref`'d deadline timer, and a second signal that forces an exit. What differs is the order and the deadline.

| Process | preStop | On SIGTERM, in order | Drain deadline | Grace period |
|---|---|---|---|---|
| api | sleep 5 s | readiness → 503; `server.close()` and close idle keep-alive; `Connection: close` on in-flight responses; send `retry:` and end each SSE stream; abort running exports (the client retries); in-flight requests finish; close the `LISTEN` client; end both pools and Redis; flush pino and OTel | 25 s | 40 s |
| ingester | sleep 5 s | stop accepting; in-flight uploads continue; at the deadline abort them (temp dirs removed, committed batches kept, a re-`PUT` is idempotent); end the pool; flush | 60 s | 75 s |
| outbox-relay | – | stop the fallback timer and `UNLISTEN`; finish the current batch transaction; end the pool; flush | 5 s | 15 s |
| dispatcher | – | stop claiming; in-flight sends finish under their 10 s deadlines; abort the rest (their leases expire and they're redelivered); end the pool; flush | 15 s | 30 s |
| reconciler | – | stop reading the file; await in-flight tasks; `pool.destroy()`; release the job or let its lease expire (the rerun is idempotent); end the pool; flush | 20 s | 30 s |
| sweeper | – | finish the current delete batch; `pg_advisory_unlock`; end the pool; flush | 5 s | 15 s |

The api and ingester get a preStop sleep because a Service routes to them; the workers don't, because nothing does. **Every grace period exceeds preStop plus the drain deadline**, and tini is PID 1 in all six ([L33](../08-shipping/33-shipping-node.md)).

---

## 6. Observability: SLOs and alerts

Wiring, from [L29](../07-production/29-observability.md): pino to stdout with redaction, `AsyncLocalStorage` carrying the request ID and merchant ID, OpenTelemetry loaded with `--import` before any app code, `prom-client` default metrics plus the loop monitor from [L08](../02-event-loop/08-blocking-the-loop.md). The RED metrics and tracing concepts are [API L20](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/20-observability.md)'s; these are the Node-specific signals:

| Signal | Target | Alert | Why this signal |
|---|---|---|---|
| api event-loop delay p99 | < 50 ms | > 100 ms for 5 min | The Node vital sign: blocking shows here first |
| api ELU | < 0.7 (the HPA target) | > 0.85 for 10 min | Saturation before latency, unlike CPU% |
| Requests shed with 503 | rare | > 1% for 5 min | Load shedding is working, but capacity isn't |
| Oldest due webhook delivery (now − oldest due `next_attempt_at`) | < 30 s | > 2 min | Queue health is age, not depth |
| Outbox lag (age of the oldest unprocessed row) | < 5 s | > 60 s | Feeds both webhooks and the live feed |
| Heap used after GC / heap limit | < 70% | > 85% for 15 min | Leak suspicion, before the OOM |
| Restarts and `OOMKilled` | 0 | any OOMKilled; > 2 restarts in 15 min | Crash loops and memory budgets |
| pg pool waiting count | 0 | > 0 for 1 min | Pool starvation looks like "the app hangs" |
| SSE connections per instance | < 800 | > 900 | The 1,000 cap is near |
| Reconciler main-thread loop p99 during a run | < 50 ms | > 500 ms | Proves the threads, not the main loop, do the work |

---

## 7. The SDK and the CLI

```ts
import { Ledger, LedgerError } from "@ledger/node";

const ledger = new Ledger({ apiKey: config.ledgerApiKey, maxRetries: 2, timeoutMs: 30_000 });

const payment = await ledger.payments.create(
  { amount_minor: 4999, currency: "usd", customer: "cus_9s2k" },
  { idempotencyKey: order.id, signal },          // the key is generated once per call if omitted
);

for await (const p of ledger.payments.list({ status: "succeeded" })) {   // auto-pagination
  await writeRow(p);
}

const event = ledger.webhooks.verify(rawBody, req.headers["ledger-signature"], endpointSecret);
```

- **Resources:** `payments`, `refunds`, `events`, `webhookEndpoints`, each method taking `{ signal }`. List methods return an async generator that follows `next_cursor`, so the caller never sees a cursor ([TS L16](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/16-async-and-errors.md)).
- **Retries:** network errors, timeouts, 429 and 5xx, with full jitter and `Retry-After` honoured, all inside the caller's signal. A `POST` is retried only with an idempotency key, which the SDK generates once per logical call and reuses across attempts ([API L18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md)).
- **Errors:** `LedgerError` with `status`, `code`, `requestId` and `retryable`, mapped from the API's problem+json.
- **`webhooks.verify()`:** HMAC over `timestamp.rawBody` with `timingSafeEqual` on equal-length digests and a tolerance window; it takes the raw `Buffer`, never re-serialized JSON ([L12](../03-data-and-io/12-buffers-and-encodings.md)).
- **Packaging:** ESM-only, an `exports` map with `types` first, `engines: { "node": ">=22.12" }` because that's where CJS consumers can `require(esm)`, no runtime dependencies (global `fetch` is undici), built with `tsc` to `.js` + `.d.ts`, published with provenance ([L32](../08-shipping/32-packaging-and-publishing.md)).

```text
$ ledger listen --forward-to http://localhost:4000/webhooks --events "payment.*"
Ready. This session's webhook signing secret: whsec_… (Ctrl+C to stop)
10:15:02  payment.succeeded  evt_01J…  → POST /webhooks [200] 41 ms
```

`ledger listen` opens `GET /v1/events/stream` with the API key over `fetch`, parses SSE frames from the web stream, re-signs each event with a per-session secret using the same `Ledger-Signature` scheme, forwards it with a 10 s deadline, and on a dropped stream reconnects with full jitter and `Last-Event-ID`. Ctrl+C aborts the stream and any in-flight forward and exits with 130. It's built with `util.parseArgs`, a shebang and a `bin` entry, and also ships as a single executable for users without Node ([L33](../08-shipping/33-shipping-node.md)).

---

## 8. Project layout

```text
ledger-engine/
├── package.json              private root: "type": "module", engines, packageManager
├── pnpm-workspace.yaml · pnpm-lock.yaml · .nvmrc (24) · tsconfig.json (the README's)
├── apps/
│   ├── api/src/
│   │   ├── main.ts           api entrypoint: config → telemetry → app → listen → lifecycle
│   │   ├── ingest-main.ts    ingester entrypoint (its own HTTP server and Deployment)
│   │   ├── http/             app.ts, middleware/, routes/ (from the API track)
│   │   ├── export/           payments-csv.ts: cursor → CSV → gzip pipeline
│   │   ├── ingest/           split-lines, parse-csv, validate, batch, insert-batches
│   │   └── realtime/         sse.ts (the hub), listen-bridge.ts
│   └── workers/src/
│       ├── outbox-relay.ts · dispatcher.ts · reconciler.ts · sweeper.ts   (entrypoints)
│       ├── webhooks/sender.ts
│       ├── reconcile/run.ts · reconcile/worker.ts
│       └── jobs/queue.ts · jobs/backup.ts
├── packages/
│   ├── shared/src/           config, lifecycle, db (withTenantTx + ALS), errors, process-guards,
│   │                         loop-monitor, concurrency, http-client, telemetry, fs-safe,
│   │                         money, hmac, memory-guard, domain-events  (consumed as .ts via the workspace symlink)
│   ├── ledger-node/          @ledger/node: src/ → dist/ (.js + .d.ts), published
│   └── ledger-cli/           ledger: bin, util.parseArgs, depends on @ledger/node
├── labs/                     understanding builds that never ship: the event-loop tracer (L05),
│                             resp-client (L16), http-raw server (L17), Express vs Fastify (L19),
│                             the cluster primary (L23), the hand-rolled WorkerPool (L21)
├── deploy/                   Dockerfile · compose.yaml · k8s/ (six Deployments, migrate Job, backup CronJob)
├── bench/                    autocannon scripts, profile.sh (L28)
└── test/                     unit/ · integration/ (testcontainers) · drills/ (shutdown, export memory, leak)
```

**`packages/shared` is never published and never copied into `node_modules`**: it's consumed through the workspace symlink, so its real path stays outside `node_modules` and Node will strip its types. `labs/` keeps the from-scratch builds out of production while keeping them in the repo you'll show.

---

## 9. The decisions, with justifications

This table **is** your interview answer: every row is a choice with a reason, not a default.

| Decision | Choice | Why |
|---|---|---|
| Module format | ESM-only, `"type": "module"` everywhere | Every supported line can `require(esm)`, so dual packages buy only the dual-package hazard ([L04](../01-foundations/04-modules-cjs-and-esm.md), [L32](../08-shipping/32-packaging-and-publishing.md)) |
| TypeScript at runtime | Native `.ts` for apps and `shared`; `tsc`-built JS + `.d.ts` for the SDK and CLI | Long-lived processes pay stripping once and keep exact stack traces; Node won't strip under `node_modules` ([L03](../01-foundations/03-the-process.md), [L33](../08-shipping/33-shipping-node.md)) |
| Process model | One image, one process type per Deployment | One artefact to scan and promote; api and workers can't drift ([L33](../08-shipping/33-shipping-node.md)) |
| Scaling | Replicas × 1 process, not cluster or PM2 | The orchestrator already supervises, and signals and limits stay per process ([L23](../05-concurrency/23-cluster-and-scaling.md)) |
| HTTP framework | Express 5 stays | Continuity with the API track; framework time is small next to DB time. Trigger to move to Fastify: serialization showing as a hot frame at high RPS ([L19](../04-networking/19-frameworks-by-mechanism.md)) |
| Server timeouts | `keepAliveTimeout` 65 s > the LB's 60 s; `headersTimeout` above it | Closes the idle-connection 502 race ([L17](../04-networking/17-http-from-scratch.md)) |
| Load protection | 503 on loop delay, not readiness failures | Shedding requests doesn't cascade; draining pods does ([L08](../02-event-loop/08-blocking-the-loop.md)) |
| Errors | Operational vs programmer; crash on programmer errors outside a request | Limping on with unknown state is worse than a restart ([L09](../02-event-loop/09-errors-and-crashing.md)) |
| Job queue | Postgres `SKIP LOCKED` on the existing tables, not BullMQ | Enqueue in the same transaction as the business write; no second datastore to operate. Trigger: throughput Postgres can't absorb ([L26](../06-data-and-jobs/26-background-jobs-and-queues.md)) |
| Lease | Push `next_attempt_at` forward | A visibility timeout with no new column, on the existing partial index ([L26](../06-data-and-jobs/26-background-jobs-and-queues.md)) |
| Recurring jobs | `pg_try_advisory_lock`, not "run on one replica" | Rollouts overlap replicas; a lock doesn't ([L26](../06-data-and-jobs/26-background-jobs-and-queues.md)) |
| Outbound HTTP | One undici `Agent` per process: per-origin caps, deadlines, capped reads, an SSRF-safe lookup | Reuse, bounded sockets, no unconsumed-body leak, no metadata-endpoint fetches ([L18](../04-networking/18-outbound-http.md)) |
| Send concurrency | Global 200, 5 per endpoint, 10 s per attempt | One dead merchant can't starve the rest ([L10](../02-event-loop/10-cancellation-and-concurrency.md)) |
| Bulk data | `pipeline` with cursors and generator stages | Constant memory and real cancellation ([L13](../03-data-and-io/13-streams-and-backpressure.md), [L14](../03-data-and-io/14-composing-streams.md)) |
| Live feed | SSE, not WebSocket | One-way, plain HTTP, resumable with `Last-Event-ID` ([L20](../04-networking/20-realtime-servers.md), [API L23](https://github.com/Pawar-Pratik/API-mastery/blob/main/05-beyond-rest/23-realtime-and-webhooks.md)) |
| Fan-out | `LISTEN/NOTIFY` carrying IDs, not Redis pub/sub | Postgres is already there and `NOTIFY` is transactional. Trigger: notification volume or a move to PgBouncer transaction mode ([L20](../04-networking/20-realtime-servers.md)) |
| CPU work | piscina in production; L21's hand-rolled pool in `labs/` | A hardened queue, idle shrink, `signal` and `transferList`; the hand-rolled one is how you explain it ([L21](../05-concurrency/21-worker-threads.md)) |
| Tenancy in the DB | `withTenantTx` with `SET LOCAL` and ALS | `SET` leaks across pooled sessions; `SET LOCAL` dies with the transaction ([L25](../06-data-and-jobs/25-databases-from-node.md)) |
| Domain events | A typed in-process emitter for metrics and audit only | It isn't the outbox: lost on crash, single process ([L11](../03-data-and-io/11-eventemitter.md)) |
| Logging and tracing | pino; OTel via `--import` | Fast structured logs; instrumentation that loads before the modules it patches ([L29](../07-production/29-observability.md)) |
| Base image | `node:24-slim` by digest + tini, non-root | glibc for addons and DNS; signals and reaping ([L33](../08-shipping/33-shipping-node.md)) |
| Supply chain | Frozen lockfile, scripts allowlisted, provenance, Renovate with a delay, `--permission` on the reconciler | The code you didn't write is the biggest risk ([L30](../08-shipping/30-security-and-supply-chain.md)) |
| Tests | `node:test` with mock timers; testcontainers for Postgres | No sleeps; a real database ([L31](../08-shipping/31-testing-node.md)) |

---

## 10. The five things you'll say in an interview

1. *"The statement export streams 4 million rows from a Postgres cursor through a CSV transform and gzip into the response with `pipeline`. RSS stays flat for the whole export — here's the graph — and when the client disconnects, the pipeline destroys the cursor and the query disappears from `pg_stat_activity` within a second."*
2. *"I sized every pool with arithmetic: 92 connections steady, 191 worst case with autoscaling and a rollout surge, against 227 usable. Little's law says each api replica keeps about two connections busy, which is why I didn't make the pools bigger, and the trigger for PgBouncer is written down."*
3. *"Webhooks run on a Postgres `SKIP LOCKED` queue. The claim is a lease, pushing `next_attempt_at` 60 seconds out, so a `kill -9` mid-send means redelivery after a minute, never a lost event. Concurrency is 200 global and 5 per endpoint, so it takes 80 dead endpoints at once to saturate the fleet."*
4. *"Rollouts lose zero requests, and I can show it: autocannon running through `kubectl rollout restart`, zero non-2xx. That's a preStop sleep, readiness failing first, `server.close()` with keep-alive handling, SSE clients told to reconnect, and tini as PID 1. Without them the same test drops requests, and I have that number too."*
5. *"I set heap flags at 75% of each container limit and counted every isolate, so memory failures are V8 heap errors I can diagnose, not silent OOMKills. And I found a real leak with three heap snapshots: a per-merchant `Agent` map that never evicted. Here's the retaining path and the before/after heap graph."*

Each is a **mechanism, plus a trade-off, plus evidence**. That's what senior sounds like.

---

## 11. Deliberate scope boundaries

| Not built | Reason |
|---|---|
| Multi-region, active-active | A database replication problem, not a Node runtime one |
| Kafka or another broker | The Postgres queue covers this scale; the trigger is throughput or many independent consumers |
| Lambda deployment | Long-lived connections and warm pools fit containers ([L33](../08-shipping/33-shipping-node.md)) |
| A WebSocket server | The feed is one-way |
| Your own native addon | Node-API C++ is a separate skill |
| A read replica for exports and reconciliation | Trigger: export load visible in the primary's p99 |
| A full custom-metrics autoscaling pipeline | The HPA targets are specified; the adapter plumbing is ops, not Node |

**If asked "what would you add next?": resumable exports to object storage**, streaming the same pipeline into an S3 multipart upload behind a job row and a signed download URL. It would force streaming to a remote writable with backpressure, part-level retries and lifecycle cleanup, and it reuses everything already built.

---

## 12. Definition of done

```text
FUNCTIONAL
[ ] All six process types run from one image, locally (compose) and in a cluster (kind or minikube)
[ ] Statement export: cursor → CSV → gzip → response, cancelled on disconnect
[ ] Payout-file ingest: 2 GB, validated, batched, rejects.csv, idempotent re-upload
[ ] Webhooks: outbox relay, leased dispatcher, 8 attempts over ~3.4 days with jitter, dead + auto-disable, sweeper
[ ] Live feed: SSE hub, LISTEN/NOTIFY fan-out, Last-Event-ID resume, slow-consumer eviction
[ ] Reconciler: settlement file on a worker pool, idempotent restart
[ ] @ledger/node and ledger published (Verdaccio is fine), ledger listen --forward-to working

QUALITY
[ ] tsc --noEmit clean with the README's tsconfig; erasable syntax only
[ ] node:test suites: backoff schedule (mock timers), export row count + memory bound,
    ingester bad-row routing, HMAC (property-based), shutdown with zero failed requests
[ ] publint and attw clean; npm pack --dry-run reviewed
[ ] Supply-chain policy in CI: frozen lockfile, script allowlist, provenance

OPERATIONS
[ ] Manifests with probes, preStop, grace periods, budgets from §4, NODE_OPTIONS beside the limits
[ ] Runtime metrics, alerts from §6, a dashboard, OTel traces through api → Postgres → FakeProcessor
[ ] Every process passes the shutdown drill

EVIDENCE (for interviews)
[ ] Export RSS flat across 4M rows, next to the buffered version's climb to a crash
[ ] Client disconnect → query gone from pg_stat_activity (timestamped)
[ ] Ingest at constant memory, with the upload rate dropping when inserts are slowed
[ ] Reconciler main-loop p99 with and without the pool; api p99 unchanged during a run
[ ] Zero failed requests during a rollout under load, and the "before" number
[ ] docs/leak-hunt.md: three snapshots, the retaining path, the fix, before/after heap
[ ] The OOM pair: a V8 heap error vs an OOMKill, same leak, with exit codes
[ ] docs/budgets.md: §4 with measured numbers replacing the planning targets
[ ] docs/engine-architecture.md: decisions, rejections, and the triggers that would change them
```

That **EVIDENCE** section is what turns a project into interview leverage. Anyone can say "I know Node streams." Very few can hand over an RSS graph that stays flat across 4 million rows and the `pg_stat_activity` screenshot showing the query die with the client.

---

## What's next

The blueprint is the *what*. The build log is the *how*: ten phases in an order where something works after every one, each mapped to the lessons it uses, and each ending in an incident you cause on purpose and recover from.

Next → **[Lesson 35: Ledger Engine build log](35-ledger-engine-build-log.md)**
