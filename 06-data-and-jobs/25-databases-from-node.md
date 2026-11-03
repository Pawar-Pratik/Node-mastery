# Lesson 25 — Databases from Node: pools, transactions & context

> **Why this lesson exists:** the most common Node database outage doesn't look like a database problem. The app hangs, requests time out, and Postgres sits idle, because every pooled connection is checked out by code that forgot to give it back or is holding it across a slow `await`. Add transactions that silently spread across three connections and a tenant setting that leaks into the next request, and you have the failures this lesson makes impossible. Interviewers love the topic because the symptoms mislead and the fix is mechanical.

**Time:** ~85 minutes · **Prereq:** Lessons 10, 13

---

## 1. The idea in one sentence

> **A pooled connection is a scarce, stateful resource on loan: everything that must happen together (a transaction, a `SET LOCAL`, a cursor) happens on one client, and that client goes back to the pool on every path, quickly.**

---

## 2. Why pools exist, and why Node fails them in its own way

Opening a Postgres connection is expensive before you've run a single query:

```text
TCP handshake                               1 round trip
TLS handshake                               1–2 round trips
StartupMessage + SCRAM-SHA-256 auth         2+ round trips
postmaster fork()s a backend process        a whole OS process, with its own memory
```

**Postgres is process-per-connection.** Every connection is a dedicated backend process, so connections are expensive to open *and* expensive to hold. `max_connections` defaults to 100 for that reason. A pool pays the setup cost once and caps how many backends you occupy.

In Node, the pool is also your **database concurrency limiter**. One process can have thousands of requests in flight, because waiting on I/O costs a promise and not a thread. Only `max` of them can talk to Postgres at once. The rest queue inside `pg.Pool`:

```text
  1,000 in-flight requests on one event loop
          │  pool.connect()
          ▼
 ┌──────────────────── pg.Pool (max 10) ─────────────────────┐
 │  [c1 busy] [c2 busy] [c3 busy] … [c10 busy]                │
 │  waiting: ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ (990 promises)  │ ← connectionTimeoutMillis: 0
 └──────────────────────────┬────────────────────────────────┘    means they wait forever
                            ▼  10 TCP connections
                 Postgres: 10 backend processes
```

This is where Node differs from Spring. In Tomcat, a request waiting for a Hikari connection occupies one of ~200 threads, so the backlog is naturally bounded. **In Node, waiting is free, so nothing bounds the queue**: 50,000 requests can pile up behind 10 connections without any thread pressure to warn you. The process looks healthy, CPU is low, Postgres is idle, and every request times out at the load balancer. That's the "app hangs, DB idle" incident.

| `pg.Pool` option | Default | What to do |
|---|---|---|
| `max` | 10 | Derive it from the budget in §4 |
| `idleTimeoutMillis` | 10000 | Fine; idle clients close after 10 s |
| `connectionTimeoutMillis` | **0 = wait forever** | **Always set it** (1–3 s), below the request deadline |

---

## 3. Configuring the pool

```ts
// src/lib/db.ts
import pg from "pg";
import { config } from "../config.ts";
import { logger } from "./logger.ts";

pg.types.setTypeParser(20, (v) => BigInt(v));            // int8 → bigint (see §7)

export const pool = new pg.Pool({
  connectionString: config.DATABASE_URL,
  max: config.PG_POOL_MAX,                               // from the budget, never a guess
  connectionTimeoutMillis: 2_000,                        // ✅ fail fast instead of queueing forever
  idleTimeoutMillis: 10_000,
  statement_timeout: 5_000,                              // Postgres cancels runaway statements
  idle_in_transaction_session_timeout: 10_000,           // Postgres kills a leaked open transaction
  application_name: "ledger-api",                        // visible in pg_stat_activity
});

// ✅ Required. An idle client whose backend dies emits 'error' through the pool.
// With no listener that's an unhandled 'error' event, and the process crashes (L11).
pool.on("error", (err) => logger.error({ err }, "idle pg client error"));

export function poolStats() {
  return { total: pool.totalCount, idle: pool.idleCount, waiting: pool.waitingCount };
}
```

Export `poolStats()` as gauges every few seconds. **`waiting > 0` for more than a moment is the pool-exhaustion alarm**, and it fires minutes before the latency graphs make sense. When `connectionTimeoutMillis` fires, map the error to `503` with `Retry-After`: it's load shedding, not a bug.

> **Spring equivalent:** HikariCP, with the same sizing maths. Hikari's `maximumPoolSize` also defaults to 10, but its `connectionTimeout` defaults to 30 s: **Hikari gives up; `pg` waits forever.** See [Spring L19](https://github.com/Pawar-Pratik/SpringBoot/blob/main/06-data/19-jdbc-datasource-pooling.md).

---

## 4. Sizing: arithmetic, not vibes

**Little's law sizes one pool:** connections busy = query rate × how long each query holds a connection. One Ledger API replica at 300 req/s with ~2 queries per request at ~4 ms each needs `600/s × 0.004 s = 2.4` connections on average. A `max` of 6 covers bursts.

"Hold time" means everything that happens while the client is checked out. Inside a transaction, that's the whole transaction, including every `await` in it. Keep that in mind for §6.

**The fleet budget sizes all the pools together.** Every replica of every process has its own pool, and they all count against one `max_connections`:

```text
Postgres max_connections (default)                 100
− superuser_reserved_connections (default)          −3
− headroom: migrations, psql, monitoring           −10
= budget                                             87

api            (6 replicas + 2 during a rollout) × 6   =  48
api LISTEN     8 × 1 dedicated client (live feed, L20)  =   8
dispatcher     2 × 5                                    =  10
outbox-relay   1 × 2  (+1 LISTEN client)                =   3
sweeper        1 × 1                                    =   1
ingester       1 × 4                                    =   4
reconciler     1 × 4                                    =   4
                                                total   =  78   ✅ 9 to spare
```

This is a worked example against Postgres's *default* limit, to show the method. Ledger Engine's real budget (a larger `max_connections`, the separate export pool, HPA maximums plus surge) is in the [blueprint](../09-project/34-ledger-engine-blueprint.md); the arithmetic is the same.

**Count the rollout surge.** During a rolling deploy the old and new pods coexist (`maxSurge: 25%` of 6 rounds up to 2 extra pods), and the budget that fit at steady state overflows at exactly the moment you're deploying. Then new pods fail with `too many clients already` and the rollout stalls.

**Why bigger pools are often slower.** Postgres does the work on a fixed number of cores and disks. Past that parallelism, extra active connections only add context switches and contention for locks, buffers and CPU caches. In Node there's a second reason: a big pool moves the queue from `pg.Pool` (cheap, observable, with a timeout) into Postgres (where every waiting query is a process). Keep the queue on your side.

### When the budget doesn't fit: PgBouncer

When replicas × `max` outgrows the budget, put PgBouncer in front in **transaction mode**: a server connection is lent to a client only for the length of one transaction, so hundreds of client connections share a few dozen backends. The price is that **session state stops belonging to you**:

| Feature | Transaction mode | Why |
|---|---|---|
| `SET` (session level) | ❌ leaks to whoever gets that server connection next | It lives on the server connection |
| `SET LOCAL` / `set_config(…, true)` | ✅ | Scoped to the transaction |
| Session advisory locks | ❌ | The lock stays on a backend you no longer own |
| `LISTEN` | ❌ | Notifications go to a backend, not to you |
| Named prepared statements | ❌ on older versions; recent PgBouncer can track protocol-level ones (`max_prepared_statements`) | The statement exists on one backend |
| Temp tables, `WITH HOLD` cursors | ❌ | Session state |

So Ledger's LISTEN clients and the sweeper's advisory-lock client (L26) connect to Postgres directly or through a session-mode pool.

---

## 5. The transaction bug: `pool.query("BEGIN")`

```ts
// ❌ Four statements, possibly four different clients
await pool.query("BEGIN");
await pool.query("INSERT INTO payments (id, merchant_id, amount_minor) VALUES ($1, $2, $3)", p);
await pool.query("INSERT INTO outbox (event_id) VALUES ($1)", [eventId]);
await pool.query("COMMIT");
```

`pool.query()` checks out a client, runs one statement and releases it. Under concurrency, each call can get a different client:

```text
pool.query("BEGIN")          → c1: BEGIN               c1 returns to the pool *inside* a transaction
pool.query("INSERT payment") → c2: INSERT              autocommit: committed on its own
pool.query("INSERT outbox")  → c1: INSERT              lands in the stray transaction on c1
pool.query("COMMIT")         → c3: COMMIT              WARNING: there is no transaction in progress
```

With one request at a time, the pool tends to hand back the same idle client, so **this passes every test and fails under load**. The damage is worse than "not atomic": c1 went back to the pool with a transaction open, so the next request to borrow c1 runs inside someone else's transaction, and a later error there rolls back work that isn't even theirs.

The correct shape, which every transaction in Ledger goes through:

```ts
import type { PoolClient } from "pg";

export async function withTransaction<T>(fn: (client: PoolClient) => Promise<T>): Promise<T> {
  const client = await pool.connect();
  let broken: Error | undefined;
  try {
    await client.query("BEGIN");
    const result = await fn(client);
    await client.query("COMMIT");
    return result;
  } catch (err) {
    try {
      await client.query("ROLLBACK");
    } catch (rollbackErr) {
      broken = rollbackErr as Error;            // the connection itself is unusable
    }
    throw err;
  } finally {
    client.release(broken);                     // truthy → destroy it instead of reusing it
  }
}
```

Three details carry the weight:

1. **`ROLLBACK` on every failure.** After any error inside a transaction, Postgres rejects every further statement with `current transaction is aborted, commands ignored until end of transaction block`. A client released without `ROLLBACK` poisons the next borrower: all of their queries fail.
2. **`release(err)` destroys a broken client.** Passing a truthy value tells the pool to disconnect that client rather than hand a dead or mid-transaction connection to the next request. The pool opens a fresh one when it needs it.
3. **Exactly one `release`, in `finally`.** Forgetting it leaks the client forever; calling it twice throws. `finally` is the only place that runs on every path.

> **Spring equivalent:** `@Transactional` does this for you through a proxy, so a call from inside the same class silently skips it (the self-invocation trap). Node has no proxy, **so there's no self-invocation trap, and no safety net either**: the boundary is exactly where you call `withTransaction`. See [Spring L23](https://github.com/Pawar-Pratik/SpringBoot/blob/main/06-data/23-transactions-deep-dive.md), whose `TransactionTemplate` is the closest Java analogue.

---

## 6. Pool starvation: holding a client across a slow `await`

```ts
// ❌ A 3-second processor call inside a transaction
await withTransaction(async (tx) => {
  const { rows } = await tx.query("SELECT * FROM payments WHERE id = $1 FOR UPDATE", [id]);
  const result = await processor.charge(rows[0]);           // 3 s: client AND row lock held
  await tx.query("UPDATE payments SET status = $2 WHERE id = $1", [id, result.status]);
});
```

Run Little's law on it: 50 charges/s × 3 s of hold time = **150 connections needed**, against a pool of 6. Within a fraction of a second every client is parked inside a charge, and `GET /v1/payments/:id`, which needs one 2 ms query, waits in the queue behind them. In `pg_stat_activity` the signature is a wall of sessions in state **`idle in transaction`**: connected, inside a transaction, doing nothing. The open transaction also holds the row lock and stops vacuum from cleaning up behind it.

**The fix is structural: never hold a client across network I/O to anything that isn't the database.** Split the work into short transactions either side of the slow call, with the state machine recording where you are:

```ts
// ✅ Two 2 ms transactions; no client is held during the 3 s call
const payment = await withTransaction((tx) => markProcessing(tx, id));     // status → processing
const result  = await processor.charge(payment, { signal });              // no client held
await withTransaction((tx) => recordResult(tx, id, result));              // status → succeeded|failed
```

A crash between the steps leaves the payment in `processing`, which the reconciler resolves by asking the processor (idempotently) what happened. That's the API track's state-machine-plus-idempotency design doing its job.

---

## 7. Types: why your numbers come back as strings

By default `pg` parses `int8`/`bigint` (OID 20) and `numeric` (OID 1700) as **strings**, because a 64-bit integer can exceed `Number.MAX_SAFE_INTEGER` and `numeric` has arbitrary precision. [L02](../01-foundations/02-the-javascript-node-assumes.md) covers the JavaScript side. The production consequences:

| Column | Default in JS | The bug it causes | Ledger's choice |
|---|---|---|---|
| `amount_minor bigint` | `"4999"` | `"4999" + 1 === "49991"` | `setTypeParser(20, BigInt)`, then serialise at the DTO boundary (`JSON.stringify(1n)` throws) |
| `count(*)` (returns `bigint`) | `"42"` | `total + 1 === "421"` | Cast in SQL: `count(*)::int` when it's guaranteed small |
| `numeric` | `"12.50"` | `parseFloat` reintroduces float error | Keep the string, or use a decimal library; never `Number` |
| `timestamptz` | `Date` | Postgres stores **microseconds**, `Date` holds milliseconds | Build pagination cursors from the raw text or from `(created_at, id)` in SQL ([API L08](https://github.com/Pawar-Pratik/API-mastery/blob/main/02-rest-design/08-collections-and-pagination.md)), or rows get skipped or repeated |

**Large results don't go through `query()` at all.** `query()` buffers every row in memory. The 4M-row statement export streams through `pg-query-stream` on a dedicated client ([L13](../03-data-and-io/13-streams-and-backpressure.md)), and that client is held for the whole stream, so the export concurrency cap is part of the pool budget too.

---

## 8. RLS tenancy: `SET LOCAL`, never `SET`

The policy itself is [API L15](https://github.com/Pawar-Pratik/API-mastery/blob/main/03-security/15-authorization-and-multitenancy.md): `USING (merchant_id = current_setting('app.merchant_id', true)::uuid)`. The Node side is about *where the setting lives*:

```ts
// ❌ Session level: it survives release(), and it's string-built (SET can't take $1)
await client.query(`SET app.merchant_id = '${merchantId}'`);

// ✅ Transaction-local AND parameterised: set_config(name, value, is_local = true)
await client.query("SELECT set_config('app.merchant_id', $1, true)", [merchantId]);
```

`SET` is a utility statement, so it can't take bind parameters, which tempts people into string interpolation. `set_config(..., true)` is the parameterisable `SET LOCAL`. **Session-level settings outlive the request** because the connection outlives the request:

```text
req A (merchant A): checkout c4 → SET app.merchant_id = A → queries → release c4
req B (merchant B): checkout c4 → a new code path forgets to set the tenant
                    SELECT * FROM payments  → RLS sees merchant A → B gets A's payments ❌

with set_config(…, true): COMMIT/ROLLBACK resets it, so req B on c4 has no tenant
                    → current_setting(…, true) is empty → the policy matches zero rows ✅
```

Two consequences. `SET LOCAL` outside a transaction does nothing except emit a warning, so the setting must go inside `withTransaction`. And fail closed: once a custom setting has been set on a session, reading it later can return an empty string rather than `NULL`, so write the policy as `nullif(current_setting('app.merchant_id', true), '')::uuid` to get zero rows instead of a cast error.

---

## 9. `AsyncLocalStorage`: carrying the tenant and the transaction

Threading `tx` and `merchantId` through every repository function is noisy, and every function that takes them as optional is a place to forget them. `AsyncLocalStorage` (ALS) carries both implicitly, the way Spring binds the current connection to a `ThreadLocal`:

```ts
import { AsyncLocalStorage } from "node:async_hooks";
import type { PoolClient, QueryResultRow } from "pg";

type DbContext = { merchantId: string; client: PoolClient; open: boolean };
const als = new AsyncLocalStorage<DbContext>();

export async function withTenantTx<T>(merchantId: string, fn: () => Promise<T>): Promise<T> {
  const outer = als.getStore();
  if (outer?.open) {
    if (outer.merchantId !== merchantId) throw new Error("tenant switch inside a transaction");
    return fn();                                        // join the outer transaction
  }
  return withTransaction(async (client) => {
    await client.query("SELECT set_config('app.merchant_id', $1, true)", [merchantId]);
    const ctx: DbContext = { merchantId, client, open: true };
    try {
      return await als.run(ctx, fn);
    } finally {
      ctx.open = false;                                 // stale references now fail loudly
    }
  });
}

export async function q<R extends QueryResultRow>(text: string, values: unknown[] = []) {
  const ctx = als.getStore();
  if (!ctx) throw new Error("query outside a tenant transaction");   // no unscoped path exists
  if (!ctx.open) throw new Error("query after its transaction ended");
  return ctx.client.query<R>(text, values);
}
```

Repositories call `q()` and never see a client; handlers wrap the unit of work once:

```ts
export const payments = {
  findById: async (id: string) =>
    (await q<PaymentRow>("SELECT * FROM payments WHERE id = $1", [id])).rows[0] ?? null,
};

const payment = await withTenantTx(req.principal.merchantId, () => payments.findById(id));
```

**How ALS works, briefly:** `als.run(store, fn)` makes `store` the current context, and every promise continuation, timer and I/O callback created inside `fn` captures that context and restores it when it runs. In Node 24 the implementation is AsyncContextFrame by default. The full mechanics, and the places context gets lost (callback-based libraries, emitters created outside the request), are in [L29](../07-production/29-observability.md).

**Why the `open` flag exists.** A fire-and-forget promise started inside the transaction (`void audit.record(…)`) inherits the store. If it queries after `COMMIT`, its client is back in the pool, possibly lent to another request that has set *another tenant*. Nothing in `pg` stops a stale reference from querying a released client. The flag turns silent cross-request corruption into a loud error.

---

## 10. Timeouts, cancellation, injection and N+1

| Timeout | Lives in | What it does |
|---|---|---|
| `connectionTimeoutMillis` | `pg.Pool` | Bounds the wait for a client (0 = forever) |
| `statement_timeout` | Postgres (config, role, or `SET LOCAL` per transaction) | **The server cancels the statement.** The one that actually bounds work |
| `idle_in_transaction_session_timeout` | Postgres | Terminates a session left idle inside a transaction: the leak detector of last resort |
| `lock_timeout` | Postgres | Bounds waiting for a row or table lock |
| `query_timeout` | `pg` client | The client stops waiting; **the server keeps executing** |

`pg` doesn't take an `AbortSignal` for a query. To cancel on client disconnect ([L10](../02-event-loop/10-cancellation-and-concurrency.md)), you send `SELECT pg_cancel_backend(pid)` over a *different* connection, using the pid you read with `pg_backend_pid()` at checkout. Destroying the client isn't enough on its own: a backend busy computing a result only notices its client vanished when it tries to send rows (Postgres 14 added `client_connection_check_interval` to check sooner). Use cancellation for long reads like exports; for everything else, a short `statement_timeout` is the reliable bound.

**SQL injection: values only ever travel as parameters.** `pg` sends them in a separate protocol message (Bind), so they're never parsed as SQL:

```ts
// ❌ Injectable
await q(`SELECT * FROM payments WHERE status = '${req.query.status}'`);
// ✅ Parameterised
await q("SELECT * FROM payments WHERE status = $1", [status]);

// Identifiers (columns, sort keys) can't be parameters: allowlist them
const SORT_COLUMNS = { created: "created_at", amount: "amount_minor" } as const;
```

When an allowlist is impossible, `pg-format`'s `%I` quotes identifiers safely.

**N+1 in Node loops** is the same bug as in JPA, written by hand:

```ts
// ❌ 1 + N round trips and N checkouts: 100 payments × 1 ms = 100 ms of pure latency
for (const p of page) p.refunds = (await q("SELECT * FROM refunds WHERE payment_id = $1", [p.id])).rows;

// ✅ One round trip: pg turns the JS array into a Postgres array
const { rows } = await q("SELECT * FROM refunds WHERE payment_id = ANY($1)", [page.map((p) => p.id)]);
```

`Promise.all` over the N queries isn't a fix. On a pool it fans out N checkouts; on one transaction client the queries just queue, because **a Postgres connection runs one statement at a time**.

---

## 11. ORMs, query builders and raw SQL

| | Raw `pg` | Kysely | Drizzle | Prisma |
|---|---|---|---|---|
| What it is | The driver | Type-safe query builder over your driver | TypeScript schema + SQL-like builder | Schema DSL + generated client + its own runtime layer |
| SQL visibility | Total | High | High | Lower: the query is generated for you |
| Connection/transaction control | Total | High | High | Through its own transaction API |
| Fits RLS via `set_config` + ALS | ✅ | ✅ | ✅ | Possible: `set_config` and the queries must run in the same transaction |
| Migrations | Bring your own | Bring your own | Included | Included |

The deciding question for Ledger isn't ergonomics, it's **"can I control which connection runs which statement?"** RLS with `SET LOCAL`, ALS-carried transactions and cursor streaming all depend on it. Ledger uses raw `pg` behind thin typed repositories; Kysely is the upgrade if hand-written types start drifting from the schema. Picking an ORM is fine when you've checked that it doesn't hide the connection from you.

---

## 12. Redis from Node: one connection, with exceptions

Redis needs a different mental model from Postgres. A Redis client (`ioredis`, `node-redis`) **pipelines** commands over one TCP connection and matches replies in order: the framing from [L16](../04-networking/16-tcp-and-net.md) plus a queue. Redis executes commands one at a time, so a pool of connections doesn't make it faster. One shared connection per process is normal.

The exceptions each need a **dedicated connection** (both clients offer `duplicate()`):

- **Blocking commands** (`BLPOP`, `XREAD BLOCK`) park the connection. Every other command pipelined behind them waits.
- **Pub/sub:** a connection in subscriber mode can only run subscription commands.

Two more traps: a slow command (`KEYS *`, a huge `SMEMBERS`) blocks the **whole Redis server** for every client, so use `SCAN`; and both popular clients queue commands while reconnecting by default, which is an unbounded in-memory buffer during a Redis outage. Bound it, and fail fast.

---

## 13. Production rules

| Rule | Why |
|---|---|
| **Set `connectionTimeoutMillis`; map its error to 503** | The default waits forever, so an exhausted pool hangs every request |
| **Every transaction goes through `withTransaction`** | `pool.query("BEGIN")` spreads one transaction over several clients |
| **`ROLLBACK` on failure; `release(err)` if the rollback fails** | An aborted or broken client poisons the next borrower |
| **Never hold a client across non-database I/O** | Hold time multiplies the connections you need (Little's law) |
| **Size every pool from the fleet budget, including rollout surge** | Replicas × `max` must fit `max_connections` at the worst moment |
| **Tenant via `set_config(…, true)` inside the transaction** | Session `SET` leaks to the next borrower and can't be parameterised |
| **Repositories query through the ALS context, never the bare pool** | No unscoped query path exists to forget |
| **Mark the ALS context closed after commit** | Floating promises would otherwise query a client lent to someone else |
| **`statement_timeout` and `idle_in_transaction_session_timeout` on the server** | The client can't bound server-side work by itself |
| **`pool.on("error")` registered at startup** | An idle client error with no listener crashes the process |
| **Parameters for values, an allowlist for identifiers** | The only complete defence against SQL injection |
| **Export `total/idle/waiting`; alert on sustained `waiting > 0`** | It's the earliest signal of pool exhaustion |

---

## 14. Interview traps

**Q1. "Why can't you run a transaction with `pool.query`?"**
Because a transaction is per *connection*, and `pool.query()` checks out a client, runs one statement and releases it, so `BEGIN`, the statements and `COMMIT` can land on different connections. The inserts autocommit individually, `COMMIT` warns that there's no transaction, and a client goes back to the pool with an open transaction that the next borrower inherits. The weak answer is "use `pool.connect()`"; the strong one adds `ROLLBACK` in the catch and `release(err)` in a `finally`.
**Follow-up:** "What does `release(err)` do?" It destroys the client instead of returning it, which is what you want when the connection itself may be broken.

**Q2. "The app hangs, latency is through the roof, and the database is idle. What's happening?"**
Pool exhaustion. Every client is checked out, either leaked by a path that never releases or parked across a slow `await` inside a transaction, and new requests wait in the pool's queue, forever with the default `connectionTimeoutMillis: 0`. Prove it with `waitingCount` and with `pg_stat_activity` showing `idle in transaction` sessions or fewer connections than `max` doing work. Fix the leak, add the timeout, and alert on `waiting`.

**Q3. "How do you size a connection pool?"**
Two constraints. Per pool, Little's law: query rate × hold time gives average busy connections, plus burst headroom. Across the fleet: every replica of every process × its `max`, plus LISTEN clients and rollout surge, must fit under `max_connections` minus reserved and admin headroom. Then say that bigger isn't better: past the database's real parallelism, more connections add contention, and in Node a big pool just moves the queue into Postgres.

**Q4. "`SET` vs `SET LOCAL` with a connection pool?"**
`SET` changes the session, and the session belongs to a pooled connection that outlives your request, so the next borrower inherits your tenant: with RLS, that's a cross-tenant data leak. `SET LOCAL` (or `set_config(name, value, true)`) reverts at commit or rollback. Under PgBouncer transaction mode, `SET` is worse still, because the server connection moves between clients after every transaction.

**Q5. "How do you pass a transaction through layers without parameters?"**
`AsyncLocalStorage`. `withTenantTx` opens the transaction, sets the tenant, and runs the unit of work inside `als.run({ client, merchantId })`; repositories read the client from the store. It's Spring's `ThreadLocal`-bound connection, keyed by async context instead of thread. Mention the trap: a floating promise inherits the context and can query after commit, so mark the context closed.

**Q6. "Why does `pg` return `numeric` and `bigint` as strings?"**
Because JavaScript numbers are doubles, exact only up to 2^53 − 1, while `bigint` goes to 2^63 − 1 and `numeric` has arbitrary precision. Returning a lossy number would corrupt money silently, so the driver gives you the exact text and makes you choose: a `BigInt` parser for `int8`, a decimal library or string for `numeric`. The classic follow-on bug is `count(*)` + 1 producing `"421"`.

**Q7. "What's wrong with calling a payment processor inside a database transaction?"**
The client and any row locks are held for the processor's latency, so hold time jumps from milliseconds to seconds and the pool starves, taking unrelated endpoints down with it. You also can't roll back a charge, so the transaction doesn't give you atomicity anyway. Use short transactions on both sides of the call, with the state machine (`processing`) and an idempotent processor call making a crash in between recoverable.

**Q8. "You set `query_timeout` to 5 seconds. Is the query bounded?"**
No: `query_timeout` is client-side. The promise rejects, but Postgres keeps executing the statement and holding its resources. `statement_timeout` is the server-side bound that actually cancels work; to cancel on demand, send `pg_cancel_backend(pid)` from another connection.

**Q9. "ORM or query builder?"**
Answer with the constraint, not taste: if you rely on transaction-scoped session state (RLS via `set_config`), ALS-carried transactions or streaming cursors, you need full control over which connection runs which statement. A query builder (Kysely, Drizzle) keeps that control and adds type safety; an ORM like Prisma adds productivity and migrations but hides more. Either is defensible once you've checked that requirement.

**Q10. "Do you need a Redis connection pool in Node?"**
Usually not. Redis executes commands sequentially and clients pipeline many in-flight commands over one connection, so one shared connection per process is normal. You need dedicated connections for blocking commands and for pub/sub, because both monopolise the connection. And no client-side pool helps if one `KEYS *` blocks the server for everyone.

---

## 15. Build & break

### Build — `src/lib/db.ts`
- The pool from §3: `connectionTimeoutMillis`, server timeouts, `application_name`, `pool.on("error")`, the `int8` parser.
- `withTransaction(fn)` with `ROLLBACK` and `release(broken)`; `withTenantTx(merchantId, fn)` with `set_config(…, true)` and the ALS context; `q()` that refuses to run outside a context or after commit.
- `poolStats()` logged every 5 s (exported as metrics in L29), plus a `pool_waiting` warning when it stays above zero.
- One scoped repository (`payments`) that uses only `q()`, and an ESLint `no-restricted-imports` rule so handlers can't import `pool`.

### Build — the budget
Write Ledger's §4 table for your own deployment plan, with each process's `max` in config rather than code. Recompute it with one more API replica and decide where PgBouncer would go.

### Break — 4 experiments
1. **Transactions via `pool.query`.** Run 20 concurrent "transfers" that use `pool.query("BEGIN")` … `pool.query("COMMIT")`. While they run, query `SELECT pid, state, left(query, 60) FROM pg_stat_activity WHERE application_name = 'ledger-api';` in `psql` (`docker compose exec postgres psql -U postgres`). You'll see one transfer's statements on different `pid`s, sessions stuck in `idle in transaction`, and `there is no transaction in progress` warnings in the Postgres log. Switch to `withTransaction` and repeat.
2. **The leaked client.** Add a route that does `await pool.connect()`, then throws before releasing. Hit it 10 times with `curl.exe`, then call any healthy route: **the 11th request hangs forever** while `poolStats()` shows `idle: 0, waiting: 1` and Postgres is idle. Set `connectionTimeoutMillis: 2000` and watch it fail in 2 s instead; then fix the route with `finally`.
3. **Starvation by a slow `await`.** Start a local HTTP server that answers after 3 s, and call it from inside `withTransaction` in `POST /v1/payments`. Run `npx autocannon -c 50 -d 20` against it, and meanwhile time `GET /v1/payments/:id` with `curl.exe -w "%{time_total}"`: the read waits behind the charges. Restructure into two short transactions and re-measure both.
4. **The tenant leak.** Set the pool `max` to 1 so every request reuses the same client. Request A runs `SET app.merchant_id = '<A>'`; request B takes a code path that doesn't set the tenant and lists payments: **B receives A's rows.** Replace `SET` with `set_config(…, true)` inside `withTenantTx` and repeat: B gets zero rows.

### Explain out loud (60–120 seconds)
1. Why Postgres connections are expensive, and why Node's pool queue has no natural bound.
2. The `pool.query("BEGIN")` bug, drawn as three statements on three clients, and the correct `withTransaction`.
3. Little's law for one pool and the fleet budget for all of them, including rollout surge.
4. `SET` vs `SET LOCAL`, and what PgBouncer transaction mode breaks.
5. How ALS carries the tenant and the transaction, and why the context is closed after commit.

---

## What's next

The request path now stays short: it writes the payment and an outbox row in one transaction and returns. Everything slow (webhook delivery, retries over three days, sweeping old idempotency keys) has to happen somewhere else, survive crashes and deploys, and run exactly as often as intended across N replicas. That's a job system, and Postgres can be one.

Next → **[Lesson 26: Background jobs & queues](26-background-jobs-and-queues.md)**
