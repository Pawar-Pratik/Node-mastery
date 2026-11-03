# Lesson 26 — Background jobs & queues

> **Why this lesson exists:** the request path has to stay fast, so slow work moves into jobs, and jobs fail, run twice, pile up behind one noisy tenant, and run N times when you scale to N replicas. A job system is a distributed-systems problem with a Node implementation. Ledger's webhook dispatcher, outbox relay and idempotency-key sweeper are all jobs, and "design a job queue" is one of the most common senior system-design prompts.

**Time:** ~85 minutes · **Prereq:** Lessons 24, 25

---

## 1. The idea in one sentence

> **A job system gives you at-least-once execution, never exactly-once: every claim is a lease that can expire, so some jobs will run twice, and every handler must make that harmless.**

---

## 2. Why jobs exist, and why `setTimeout` isn't a queue

A payment request writes two rows and returns in milliseconds. Delivering the webhook can take 10 seconds, can fail, and has to be retried for three days. That work can't live in the request, and it can't live in memory either:

```ts
// ❌ A retry "queue" that dies with the process
setTimeout(() => void send(delivery), 30 * 60_000);    // any deploy in the next 30 minutes deletes it
```

Every deploy is a process exit ([L24](../05-concurrency/24-graceful-shutdown.md)), so an in-memory timer is lost on every rollout as well as on every crash. Each pending timer also pins its closure in memory, N replicas don't coordinate, and nobody can see what's waiting. That's why [L05](../02-event-loop/05-the-event-loop.md) said Ledger keeps `next_attempt_at` in Postgres. **Durable work needs a durable queue.**

| Kind | Ledger example | What it needs |
|---|---|---|
| Fire-and-forget | Fan an event out to webhook endpoints | Durable enqueue, at-least-once |
| Scheduled | Retry delivery 6 at `next_attempt_at` | A due-time index and a claim |
| Recurring | Sweep idempotency keys older than 24 h | Exactly one runner across replicas |
| Long-running | Reconcile a 500 MB settlement file | Leases with heartbeats, resumability |

---

## 3. Where the queue lives

| | In-process (`setTimeout`, an array) | Postgres (`SKIP LOCKED`; pg-boss, graphile-worker) | Redis (BullMQ) | Managed (SQS) |
|---|---|---|---|---|
| Survives a crash | ❌ | ✅ | Depends on Redis persistence settings | ✅ |
| **Enqueue in the same transaction as the business write** | — | ✅ | ❌ (dual write → outbox) | ❌ (dual write → outbox) |
| Pick-up latency | Immediate | Poll interval, or near-immediate with `LISTEN/NOTIFY` | Near-immediate | Long polling |
| Throughput ceiling | — | Moderate: every claim is an `UPDATE`, so vacuum is the cost | High | High and elastic |
| Extra system to run | None | **None, if you already run Postgres** | A stateful Redis | None, but a vendor and per-request cost |
| Built-in features | None | What you build, or what pg-boss/graphile-worker give you | Delays, priorities, rate limits, repeatable jobs, flows | Visibility timeout, DLQ redrive, FIFO queues |

**Ledger uses Postgres**, and the reason is the second row. A `webhook_deliveries` row created in the same transaction as the event it delivers can't be lost and can't exist without its event: there's no dual write to protect. Postgres is already the most-watched system in the stack, and Ledger's volume is far below where queue-table churn hurts. The triggers that would change the decision: sustained job rates where the dead tuples from claim/complete `UPDATE`s outpace autovacuum, or many independent consumers of the same events (then the outbox feeds Kafka or SQS instead).

Under MVCC every claim and completion writes a new row version. Keep the **hot index small**, which is what the blueprint's partial index on `next_attempt_at WHERE status = 'pending'` does: delivered history stays in the table for the delivery log but drops out of the index the claim scans.

---

## 4. The claim: `FOR UPDATE SKIP LOCKED` and leases

`SKIP LOCKED` lets many workers pull from one table without coordinating:

```text
                  webhook_deliveries, due rows (status='pending', next_attempt_at <= now())
                  ┌────┬────┬────┬────┬────┬────┐
                  │ d1 │ d2 │ d3 │ d4 │ d5 │ d6 │
                  └────┴────┴────┴────┴────┴────┘
worker A: SELECT … LIMIT 3 FOR UPDATE SKIP LOCKED  → locks d1 d2 d3
worker B: SELECT … LIMIT 3 FOR UPDATE SKIP LOCKED  → d1–d3 are locked, skip them → locks d4 d5 d6

plain FOR UPDATE:     B blocks on d1 until A commits, so workers run one after another
no FOR UPDATE at all: A and B both read d1–d3, and both send them  → duplicates
```

What happens *after* the claim is the real design choice:

1. **Lock for the whole job.** Keep the transaction open while you work. A crash drops the connection and Postgres releases the locks, which is elegant, but every in-flight job holds a pooled connection and an open transaction: exactly the starvation pattern from [L25](25-databases-from-node.md). Fine for millisecond, database-only jobs like the outbox relay.
2. **Lease.** Claim in a short transaction that hides the row until a deadline, do the work with no transaction open, then complete it. A crash means the lease expires and another worker reclaims the job. **This is SQS's visibility timeout and BullMQ's job lock, written in SQL.** Anything that does network I/O, like a webhook send, uses a lease.

Ledger's dispatcher leases by pushing `next_attempt_at` forward, so it needs no extra column and uses the existing partial index:

```sql
WITH due AS (
  SELECT id
    FROM webhook_deliveries
   WHERE status = 'pending' AND next_attempt_at <= now()
   ORDER BY next_attempt_at
   LIMIT $1                                            -- this worker's free slots, not a fixed 100
   FOR UPDATE SKIP LOCKED
)
UPDATE webhook_deliveries d
   SET attempt = d.attempt + 1,                        -- counted at claim time (see §5)
       next_attempt_at = now() + make_interval(secs => $2)   -- the lease: invisible until it expires
  FROM due
 WHERE d.id = due.id
RETURNING d.id, d.endpoint_id, d.event_id, d.attempt;
```

Workers process every merchant by design, so they connect as a dedicated role with `BYPASSRLS`: the deliberate bypass from [API L15](https://github.com/Pawar-Pratik/API-mastery/blob/main/03-security/15-authorization-and-multitenancy.md), never used by the API process.

### Why jobs run twice

```text
t=0     A claims d7 with a 5 s lease               ← lease shorter than the job can take
t=0     A POSTs to a slow merchant endpoint (8 s)
t=5     the lease expires; d7 is due again
t=5.1   B claims d7 (attempt 2) and POSTs it too
t=8     A gets 200 → marks delivered
t=9     B gets 200 → marks delivered               the merchant received the event twice
```

Four layers of defence, and only the last one is complete:

1. **Lease ≫ job deadline.** The send has a 10 s total deadline from [L18](../04-networking/18-outbound-http.md), enforced with an `AbortSignal`, so it can't outlive a 60 s lease.
2. **Heartbeats for long jobs.** A job that legitimately runs for minutes extends its own lease periodically, as long as it's still the owner.
3. **Fencing on completion.** The attempt number taken at claim is a fencing token: completion only succeeds if the row is still on that attempt, so a worker whose lease was taken over learns it lost ownership instead of overwriting the new owner's result.
4. **An idempotent effect.** Even with 1–3, a crash *after* the POST but *before* the completion write re-runs the job. Merchants deduplicate on `Ledger-Event-Id` ([API L23](https://github.com/Pawar-Pratik/API-mastery/blob/main/05-beyond-rest/23-realtime-and-webhooks.md)); internal jobs use a unique constraint or a state-guarded `UPDATE` ([API L18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md)).

**At-least-once isn't a bug you can engineer away. Idempotency is the only complete answer**, and the other three layers just make duplicates rare.

```ts
export async function complete(id: string, attempt: number, r: SendResult): Promise<boolean> {
  const { rowCount } = await pool.query(
    `UPDATE webhook_deliveries
        SET status = 'delivered', response_status = $3, response_body = $4,
            duration_ms = $5, next_attempt_at = NULL
      WHERE id = $1 AND attempt = $2 AND status = 'pending'`,      -- the fence
    [id, attempt, r.status, r.bodyPreview, r.durationMs],
  );
  return rowCount === 1;            // false → our lease was taken over; count it as a metric
}
```

---

## 5. Retries, the DLQ and poison messages

The retry *policy* (which outcomes retry, the ~3-day schedule, jitter) is the API track's: [API L18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md) and [API L23](https://github.com/Pawar-Pratik/API-mastery/blob/main/05-beyond-rest/23-realtime-and-webhooks.md). The Node side is a pure function from an outcome to the next row state, which makes it trivially testable with fake time ([L31](../08-shipping/31-testing-node.md)):

```ts
// Ledger's published schedule (blueprint: 8 attempts over ~3 days): 7 gaps, ~3.4 days in total.
// API L23 shows the same shape with an immediate first attempt; publish whichever you choose.
export const BACKOFF_SEC = [60, 300, 1_800, 7_200, 21_600, 86_400, 172_800];

type Outcome =
  | { kind: "delivered" }
  | { kind: "retry"; reason: string }             // timeout, connection error, 5xx, 429
  | { kind: "reject"; reason: string };           // other 4xx: the receiver refused it

export const MAX_ATTEMPTS = 8;

export function nextState(attempt: number, o: Outcome, now = Date.now()) {
  if (o.kind === "delivered") return { status: "delivered", nextAttemptAt: null } as const;
  if (o.kind === "reject") return { status: "failed", nextAttemptAt: null } as const;
  if (attempt >= MAX_ATTEMPTS) return { status: "dead", nextAttemptAt: null } as const;
  const base = BACKOFF_SEC[attempt - 1]!;
  const jittered = base * (0.8 + Math.random() * 0.4);   // ±20%, so mass failures don't retry in lockstep
  return { status: "pending", nextAttemptAt: new Date(now + jittered * 1000) } as const;
}
```

In Ledger, the four `webhook_deliveries` statuses mean: `pending` (waiting for its next attempt), `delivered`, `failed` (the receiver rejected it; not retried), and `dead` (attempts exhausted). **In a Postgres queue, the DLQ is just `status = 'dead'`**, plus an alert on its rate and the blueprint's manual-retry endpoint, which resets the row to `pending` with `attempt = 0`. Every failed attempt also bumps `webhook_endpoints.consecutive_failures` (reset on success); past the threshold the endpoint is set to `disabled`, the merchant is emailed, and its pending deliveries are parked with `next_attempt_at = NULL` so the claim never sees them until it's re-enabled.

**Poison messages** come in two shapes. A *deterministic failure* (a payload the handler can never process) is contained by classification and the attempt cap. A *crash* (the handler throws an uncaught exception or exhausts memory, and the process dies, [L09](../02-event-loop/09-errors-and-crashing.md)) is nastier: if attempts were only counted when a failure is recorded, the job would crash the worker, get reclaimed after the lease, crash it again, forever. **That's why `attempt` is incremented at claim time**: a job that kills its worker still burns an attempt and lands in `dead` after eight.

---

## 6. Concurrency and per-tenant fairness

**Claim only what you can run.** If a worker claims 100 rows but can only send 10 at once, 90 sit leased in memory while their lease clocks run. The claim's `LIMIT` is the worker's free capacity, and inside the worker the [L10](../02-event-loop/10-cancellation-and-concurrency.md) limits apply: a global cap of 200, 5 per endpoint, a 10 s deadline per attempt.

```ts
import { setTimeout as delay } from "node:timers/promises";

const MAX_IN_FLIGHT = 200;
const global = limit(MAX_IN_FLIGHT);                    // L10
const perEndpoint = keyedLimit(5);                      // L10
let inFlight = 0;

export async function runDispatcher(stopClaiming: AbortSignal, abortSends: AbortSignal) {
  while (!stopClaiming.aborted) {
    const free = MAX_IN_FLIGHT - inFlight;
    const jobs = free > 0 ? await claim(free, LEASE_SEC) : [];
    for (const job of jobs) {
      inFlight++;
      void global(() => perEndpoint(job.endpointId, () => deliver(job, abortSends)))
        .catch((err) => logger.error({ err, deliveryId: job.id }, "deliver failed"))
        .finally(() => { inFlight--; });
    }
    if (jobs.length === 0) await delay(1_000, undefined, { signal: stopClaiming }).catch(() => {});
  }
}
```

**Fairness is a claim-time problem.** One merchant's integration bug enqueues 100,000 deliveries, all due now. `ORDER BY next_attempt_at LIMIT n` returns only theirs, every claim, for hours, and merchant B's webhooks wait behind a backlog B didn't create. It's the noisy-neighbour problem from [API L19](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/19-rate-limiting.md), moved from requests to jobs. Per-endpoint limits inside the worker don't fix it: the hot endpoint's rows still fill every batch and then queue in memory.

The fix is to **pick tenants first, then take a bounded share from each**:

```sql
WITH due AS (
  SELECT d.id
    FROM webhook_endpoints e
    CROSS JOIN LATERAL (
      SELECT id FROM webhook_deliveries
       WHERE endpoint_id = e.id AND status = 'pending' AND next_attempt_at <= now()
       ORDER BY next_attempt_at
       LIMIT 5                                   -- each endpoint's share of one claim
       FOR UPDATE SKIP LOCKED
    ) d
   WHERE e.status <> 'disabled'
   LIMIT $1
)
UPDATE webhook_deliveries … (the same lease UPDATE as §4)
```

It needs a second partial index the blueprint doesn't have yet, `(endpoint_id, next_attempt_at) WHERE status = 'pending'`. And because the outer `LIMIT` stops early, the same endpoints would fill every batch whenever more endpoints are busy than there are free slots, so rotate the starting point between claims (a cursor you advance, or `random()` ordering while the endpoints table is small). The alternatives scale differently: hash tenants into N lanes and round-robin the lanes, or weight shares by plan. All of them share the idea: **fairness means the scheduler chooses the tenant before it chooses the job.**

---

## 7. The outbox relay in Node

The API writes the payment, the event and an `outbox` row in one transaction ([API L18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md)). The relay turns outbox rows into deliveries. Its work is database-only, so it uses the lock-for-the-whole-job model:

```ts
export async function relayBatch(): Promise<number> {
  return withTransaction(async (tx) => {
    const { rows } = await tx.query<{ id: bigint; event_id: string }>(
      `SELECT id, event_id FROM outbox
        WHERE processed_at IS NULL
        ORDER BY id LIMIT 500
        FOR UPDATE SKIP LOCKED`);
    if (rows.length === 0) return 0;

    // Fan out to every subscribed endpoint. UNIQUE (endpoint_id, event_id) makes a re-run a no-op.
    // new_delivery_id() and event_matches() are small SQL helpers (ID generation, 'payment.*' globs).
    await tx.query(
      `INSERT INTO webhook_deliveries (id, endpoint_id, event_id, status, next_attempt_at)
       SELECT new_delivery_id(), e.id, ev.id, 'pending', now()
         FROM events ev
         JOIN webhook_endpoints e ON e.merchant_id = ev.merchant_id AND e.status <> 'disabled'
        WHERE ev.id = ANY($1) AND event_matches(ev.type, e.enabled_events)
       ON CONFLICT (endpoint_id, event_id) DO NOTHING`,
      [rows.map((r) => r.event_id)]);

    await tx.query("UPDATE outbox SET processed_at = now() WHERE id = ANY($1)", [rows.map((r) => r.id)]);
    await tx.query("SELECT pg_notify('ledger_events', '')");     // wakes the dispatcher and the live feed (L20)
    return rows.length;
  });
}
```

Two details an interviewer will probe. Because the fan-out and the `processed_at` update commit together **in one database**, this step is effectively exactly-once; if the relay published to SQS or Kafka instead, publish-then-mark can't be atomic and you're back to at-least-once. And **never track a "last processed id" high-water mark**: `bigserial` values are assigned at insert, not at commit, so a transaction that commits late with a lower id would be skipped forever. `processed_at IS NULL` picks a row up whenever it becomes visible.

### Waking up: `LISTEN/NOTIFY` instead of tight polling

Polling every 50 ms burns queries for nothing; polling every 5 s adds up to 5 s of latency to every webhook. Instead, the API's transaction runs `SELECT pg_notify('outbox', '')`, and the relay listens on a dedicated connection. `NOTIFY` has three properties you design around: **it's delivered only if and when the transaction commits**, identical notifications in one transaction are folded into one, and **it isn't durable**: a listener that's disconnected misses it. So keep a slow fallback poll, and drain on every reconnect.

```ts
const listener = new pg.Client({ connectionString: config.DATABASE_URL_DIRECT });  // not via PgBouncer
await listener.connect();
await listener.query("LISTEN outbox");
listener.on("notification", () => kick());

let running = false;
let again = false;
function kick(): void {
  if (running) { again = true; return; }                 // coalesce a burst of NOTIFYs into one drain
  running = true;
  void (async () => {
    try {
      do { again = false; while ((await relayBatch()) > 0); } while (again);
    } catch (err) {
      logger.error({ err }, "outbox relay failed");
    } finally {
      running = false;
    }
  })();
}
const fallback = setInterval(kick, 5_000);               // NOTIFY isn't durable; cleared on shutdown
```

---

## 8. Recurring jobs across N replicas: advisory locks

A `setInterval` sweeper inside a 3-replica deployment runs three times concurrently ([L23](../05-concurrency/23-cluster-and-scaling.md)). A separate `replicas: 1` deployment doesn't fully fix it either: during a rolling update the old and new pods overlap. Postgres gives you a lock without a table:

```ts
import { setInterval as every } from "node:timers/promises";

const SWEEPER_LOCK = 26_001;                             // an app-wide constant key

export async function runSweeper(signal: AbortSignal): Promise<void> {
  const session = new pg.Client({ connectionString: config.DATABASE_URL_DIRECT });  // a session you own
  await session.connect();
  try {
    for await (const _ of every(60_000, undefined, { signal })) {
      const { rows } = await session.query<{ leader: boolean }>(
        "SELECT pg_try_advisory_lock($1) AS leader", [SWEEPER_LOCK]);
      if (!rows[0]?.leader) continue;                    // another replica holds it this round
      try {
        await sweepExpiredKeys(signal);                  // DELETE in batches of 5,000, checking signal
      } finally {
        await session.query("SELECT pg_advisory_unlock($1)", [SWEEPER_LOCK]);
      }
    }
  } catch (err) {
    if (!signal.aborted) throw err;                      // abort is the normal way out
  } finally {
    await session.end();                                 // ending the session releases any lock it holds
  }
}
```

`pg_try_advisory_lock` returns `true` or `false` immediately, and a session-level lock belongs to the **connection**. Run it through `pool.query()` and the lock is taken on one client, which then goes back to the pool still holding it; your `pg_advisory_unlock` runs on a different client, returns `false` with a "you don't own a lock" warning, and the lock leaks to whichever request borrows that client next. PgBouncer transaction mode breaks it the same way ([L25](25-databases-from-node.md)). Hence the dedicated session. The alternative, `pg_try_advisory_xact_lock`, is released at commit and works through poolers, but then the whole run has to be one transaction.

**An advisory lock prevents concurrent runs, not repeated runs.** If replica A finishes the 02:00 reconciliation at 02:00:40 and releases the lock, replica B, whose tick fires at 02:00:50, takes it and runs the job again. For "once per period", record the run: `INSERT INTO job_runs (job, period) VALUES ('reconcile', current_date) ON CONFLICT DO NOTHING RETURNING job`, where the unique key is the actual guarantee. The sweeper's deletes are idempotent, so for it the lock only saves wasted work. And a leader that loses its connection silently keeps acting until it notices, while Postgres frees the lock once *it* notices the dead connection: another reason the work itself must be idempotent.

Delete in batches (`… WHERE (merchant_id, endpoint, key) IN (SELECT … LIMIT 5000)`) so each transaction stays short: one 3-million-row `DELETE` holds locks, bloats WAL and lags replicas.

> **Spring equivalent:** `@Scheduled` + ShedLock is the advisory-lock pattern ("don't run this twice", not a scheduler), and a Kafka consumer group is the distributed claim. [Spring L32](https://github.com/Pawar-Pratik/SpringBoot/blob/main/09-production/32-async-scheduling-events.md) already recommends `SKIP LOCKED` for distributing work across instances; [Spring L37](https://github.com/Pawar-Pratik/SpringBoot/blob/main/10-microservices/37-messaging-kafka.md) covers consumers and delivery semantics.

---

## 9. Shutting a worker down safely

The ordered shutdown from [L24](../05-concurrency/24-graceful-shutdown.md), applied to a worker:

```ts
onShutdown("dispatcher", async () => {
  stopClaiming.abort();                                              // 1. no new claims
  const drained = await waitUntil(() => inFlight === 0, 20_000);     // 2. let in-flight sends finish
  if (!drained) {
    abortSends.abort();                                              // 3. cut the stragglers
    await releaseLeases(inFlightJobs());                             //    next_attempt_at = now(), fenced by attempt
  }
}, 10);                                                              // 4. then LISTEN client, pool, logs (later order)
```

The 20 s drain must fit inside `terminationGracePeriodSeconds` with room for the steps after it. Releasing leases early means stragglers retry in seconds instead of waiting out a 60 s lease. An aborted send may already have reached the merchant, so its retry is a duplicate, which the contract already allows. And if SIGKILL arrives anyway, nothing is lost: the leases expire and the jobs run again. **At-least-once is what makes a crash during shutdown boring.**

---

## 10. Observability: what says a queue is sick

| Metric | Source | What it tells you |
|---|---|---|
| Due now | `count(*)` of due rows | Backlog size; ambiguous on its own |
| **Lateness of the oldest due job** | `now() − min(next_attempt_at)` over due rows | **The alert.** What merchants actually feel |
| Attempts at completion | Histogram from the worker | Rising means receivers are failing, or your sender is |
| DLQ size and rate | `count(*) WHERE status = 'dead'`, per hour | Permanent failures; alert on the rate |
| Job duration vs lease | Timing in the worker | Headroom before jobs start running twice |
| Lease takeovers | `complete()` returning `false` | Lease too short, or workers freezing (a blocked loop, L08) |
| Backlog per endpoint | Top 10 by due count | The noisy neighbour, by name |

```sql
SELECT count(*) AS due,
       coalesce(extract(epoch FROM now() - min(next_attempt_at)), 0) AS oldest_due_seconds
  FROM webhook_deliveries
 WHERE status = 'pending' AND next_attempt_at <= now();
```

**Why lateness beats depth:** 50,000 due jobs draining at 5,000/s is ten seconds behind and healthy; 20 due jobs with no worker running is a growing outage. Lateness rises for *every* failure mode: too few workers, a stuck claim, a crashed deployment, a slow downstream. Measure it from `next_attempt_at`, not `created_at`, or a retry legitimately scheduled three days out looks like a three-day backlog.

---

## 11. Production rules

| Rule | Why |
|---|---|
| **Never keep durable work in `setTimeout` or memory** | Every deploy and crash deletes it |
| **Enqueue in the same transaction as the business write** | No dual write; a job exists if and only if its cause committed |
| **Claim with `FOR UPDATE SKIP LOCKED`, limited to free capacity** | Disjoint batches with no waiting; nothing sits leased and idle |
| **Lease network jobs; never hold a transaction across a send** | Holding one starves the pool (L25) |
| **Lease ≫ job deadline, enforced by an `AbortSignal`** | A job that outlives its lease runs twice |
| **Fence completion on the attempt number** | A worker that lost its lease can't overwrite the new owner |
| **Increment attempts at claim time** | A job that crashes its worker still reaches `dead` |
| **Every handler idempotent** | At-least-once guarantees duplicates; nothing else makes them safe |
| **Claim per tenant, then per job** | One merchant's backlog can't starve the rest |
| **Wake with `LISTEN/NOTIFY`, keep a fallback poll** | Low latency without tight polling; NOTIFY isn't durable |
| **Advisory locks on a dedicated session; a run record for "once per period"** | Pooled sessions leak locks; locks stop overlap, not repeats |
| **Alert on the lateness of the oldest due job** | It rises for every failure mode, unlike depth |

---

## 12. Interview traps

**Q1. "Design a job queue on Postgres."**
A table with `status`, `run_at` (Ledger's `next_attempt_at`), `attempt` and a partial index on due rows. Claim with a CTE that selects due rows `FOR UPDATE SKIP LOCKED LIMIT free_slots` and updates them to push `run_at` out by a lease and increment `attempt`; process with no transaction open; complete with an `UPDATE` fenced on the attempt. Retries reschedule with jittered backoff; exhausted jobs go to `dead`. Then the parts that make it production-grade: idempotent handlers, `LISTEN/NOTIFY` wake-ups, per-tenant claims, and lateness as the alert.
**Follow-up:** "Why not just hold the row lock while processing?" Because every in-flight job then holds a pooled connection and an open transaction; that's fine for millisecond jobs and pool starvation for network jobs.

**Q2. "What does `SKIP LOCKED` do?"**
It makes a locking `SELECT` skip rows another transaction has locked instead of waiting for them, so N workers claim disjoint batches concurrently. Without it, `FOR UPDATE` serialises workers on the same first rows; without any lock, two workers read the same rows and both process them. It's deliberately inconsistent, which is right for a queue and wrong for a report.

**Q3. "Why do jobs run twice, and how do you make that safe?"**
A lease expires while a slow job is still running and another worker reclaims it, or a worker crashes after the side effect but before recording completion. Long leases, heartbeats and fencing make it rarer; only an idempotent effect makes it safe, because the crash-after-effect case can't be closed by any locking. The weak answer is "use exactly-once delivery": no queue can promise exactly-once side effects.

**Q4. "BullMQ vs Postgres vs SQS?"**
Postgres when you already run it, volume is moderate and you want the enqueue in the same transaction as the business write: it removes the dual write entirely. BullMQ when you need high throughput and rich features (priorities, rate limits, repeatable jobs) and you run Redis anyway, accepting a dual write handled by an outbox. SQS for elastic scale across services with no queue to operate, also behind an outbox. Name the transactional-enqueue property; it's what most answers miss.

**Q5. "How do you run a cron job once across N replicas?"**
`pg_try_advisory_lock` on a dedicated session, so only one replica runs at a time, plus a run record with a unique `(job, period)` key if "once per period" matters, because a lock stops concurrent runs but not a second replica running after the first finished. Mention that session locks don't survive `pool.query()` or PgBouncer transaction mode. Alternatives: a Kubernetes CronJob with `concurrencyPolicy: Forbid`, or ShedLock in the Spring world.

**Q6. "What metric tells you a queue is unhealthy?"**
The lateness of the oldest due job: now minus the due time of the oldest job that should already have run. Depth is ambiguous (big and fast is fine, small and stuck isn't); lateness rises for every failure mode and maps directly onto the customer's experience. Measure from the due time, not the creation time, or scheduled retries look like a backlog.

**Q7. "Why not retry with `setTimeout`?"**
It's lost on every crash and every deploy, it pins memory per pending retry, N replicas can't coordinate it, and nobody can see or replay what's waiting. Durable state belongs in the database: a `next_attempt_at` column survives everything the process doesn't.

**Q8. "What's a poison message, and how do you stop one from looping forever?"**
A job that fails every time, either deterministically or by crashing the worker. Classify non-retryable errors so they fail immediately, cap attempts, and increment the attempt counter *when the job is claimed*: otherwise a job that kills the process is reclaimed after its lease and kills the next worker, forever, without its counter ever moving. Then alert on the DLQ rate.

**Q9. "One merchant enqueues a million jobs. What happens to everyone else?"**
With `ORDER BY run_at LIMIT n`, every claim returns that merchant's jobs and everyone else waits behind a backlog they didn't create. Per-tenant limits inside the worker don't help, because the hot tenant still fills every batch. Fix it at claim time: choose tenants first and take a bounded share from each (a `LATERAL` join per endpoint, or tenant lanes), and watch backlog per tenant.

**Q10. "How does a worker shut down without losing jobs?"**
Stop claiming, let in-flight jobs finish under a deadline that fits the grace period, abort the stragglers and release their leases so they retry promptly, then close connections and exit. If it's killed instead, leases expire and the jobs run again, which is safe because handlers are idempotent. That's the payoff of at-least-once: a crash is just a slow release.

---

## 13. Build & break

### Build — `jobs/queue.ts` and three workers
- `jobs/queue.ts`: `claim(limit, leaseSec)`, `complete(id, attempt, result)` and `fail(id, attempt, outcome)` (both fenced), `releaseLease`, `extendLease` for heartbeats, and `nextState()` as a pure function with unit tests over all eight attempts.
- `workers/dispatcher.ts`: the claim loop from §6 with the L10 limits, the L18 sender, auto-disable on `consecutive_failures`, and the shutdown hook from §9.
- `workers/outbox-relay.ts`: `relayBatch()` plus the `LISTEN` wake-up, coalescing `kick()`, reconnect-and-drain, and the fallback poll.
- `workers/sweeper.ts`: the advisory-lock loop with batched deletes that check the signal between batches.
- Export due count, oldest-due lateness, DLQ count and lease takeovers every 10 s.

### Break — 5 experiments
1. **No `SKIP LOCKED`.** Change the claim to a plain `SELECT … LIMIT 100` followed by a separate `UPDATE`, start two dispatchers against a local receiver that counts requests per event id, and enqueue 1,000 deliveries: the receiver sees duplicates. Switch to `FOR UPDATE` without `SKIP LOCKED`: duplicates stop, but the second worker's claims stall behind the first (time them). Restore the real claim.
2. **The lease that's too short.** Set the lease to 2 s and make the receiver answer after 5 s. Watch `complete()` return `false` (a takeover) and the receiver log the same `Ledger-Event-Id` twice. Make the receiver deduplicate on that id and show the effect happens once; then fix the lease.
3. **The poison payload.** Insert a delivery whose handler throws on every attempt, and temporarily remove the attempt cap: it retries forever. Then make the handler call `process.exit(1)` instead, with attempts counted only on failure: the worker crash-loops on it. Move the increment to the claim and watch it reach `dead` after eight claims.
4. **Three sweepers.** Run three sweeper processes with the advisory lock removed and log each run with its pid: three concurrent runs a minute. Restore the lock, then deliberately take it through `pool.query()` and watch `pg_advisory_unlock` return `false`.
5. **The noisy neighbour.** Enqueue 100,000 due deliveries for merchant A's endpoint, then one for merchant B. Measure how long B's delivery waits with the plain claim, then with the per-endpoint `LATERAL` claim. Put both lateness numbers in your notes.

### Explain out loud (60–120 seconds)
1. Why durable work can't live in `setTimeout`, and why Ledger's queue is in Postgres (transactional enqueue).
2. The `SKIP LOCKED` claim, the lease, and the four defences against running twice, ending with idempotency.
3. Why attempts are counted at claim time.
4. Fairness: choosing the tenant before the job.
5. Advisory locks for recurring jobs, and why a lock prevents overlap but not repeats.
6. Why lateness of the oldest due job is the queue's vital sign.

---

## Module 6 complete — checkpoint

- [ ] Why Postgres connections are expensive, and why a Node pool's wait queue has no natural bound
- [ ] `connectionTimeoutMillis` defaults to waiting forever, and what to set instead
- [ ] Pool sizing: Little's law per pool, the fleet budget across pools, rollout surge
- [ ] Why `pool.query("BEGIN")` corrupts transactions, and the correct `withTransaction`
- [ ] `ROLLBACK` on failure and `release(err)` for a broken client
- [ ] Pool starvation from holding a client across a slow `await`, and the two-transaction fix
- [ ] Why `pg` returns `bigint` and `numeric` as strings, and the `timestamptz` precision trap
- [ ] `SET` vs `SET LOCAL` with pools and PgBouncer, and `set_config(…, true)` for RLS
- [ ] Carrying tenant and transaction in `AsyncLocalStorage`, and closing the context after commit
- [ ] `statement_timeout` bounds server work; `query_timeout` doesn't
- [ ] One Redis connection per process, except for blocking commands and pub/sub
- [ ] Postgres vs BullMQ vs SQS, and the transactional-enqueue argument
- [ ] The `SKIP LOCKED` claim, leases, fencing, and why jobs still run twice
- [ ] Attempts counted at claim time to contain poison messages
- [ ] Per-tenant fairness at claim time
- [ ] The outbox relay with `LISTEN/NOTIFY`, and why there's no high-water mark
- [ ] Advisory locks on a dedicated session, plus a run record for "once per period"
- [ ] Lateness of the oldest due job as the queue's alert

---

## What's next

Module 7 opens the runtime up. The first question every Node production incident eventually asks: where did the memory go? Next you'll take the V8 heap apart, learn why a pod can be OOMKilled while its heap looks fine, and find a leak with three heap snapshots.

Next → **[Lesson 27: Memory & garbage collection](../07-production/27-memory-and-gc.md)**
