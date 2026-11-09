# Lesson 31 — Testing Node

> **Why this lesson exists:** [API Lesson 26](https://github.com/Pawar-Pratik/API-mastery/blob/main/06-craft/26-testing-apis.md) covers what to test in an API: the pyramid, real databases, idempotency and concurrency properties. This lesson covers what makes testing *Node* hard: code that runs on timers and events, streams whose bugs only appear with a slow consumer, shutdown paths that only run on a signal, and a test runner that never exits because something is still holding the event loop open. Interviewers ask "how do you test code with timers?" precisely because the common answer, a `sleep`, is wrong.

**Time:** ~75 minutes · **Prereq:** Lessons 10, 13, 24

---

## 1. The idea in one sentence

> **A Node test is deterministic only when it controls what the runtime normally owns, namely time, scheduling and open handles: replace sleeps with events, the clock with an injected or fake one, and "the test finished" with "everything it opened is closed".**

---

## 2. The three ways Node tests go wrong

| Failure | What it looks like | Root cause |
|---|---|---|
| **Flaky** | Passes locally, fails on a loaded CI runner one run in twenty | A `setTimeout(…, 100)` guessing how long async work takes |
| **Hangs** | Every test passes, then the runner sits there until a timeout | An open handle (a pool, a server, an interval) keeps the loop alive, exactly as in [Lesson 03](../01-foundations/03-the-process.md) |
| **False pass** | The stream test is green; production OOMs | The test used a fast consumer, so backpressure was never exercised |

Every technique below removes one of these. None of them involves waiting longer.

---

## 3. `node:test`: the built-in runner

Node ships a test runner. With type stripping, it runs `.ts` test files directly: no transform, no config, no dependency.

```ts
// test/unit/money.test.ts
import { describe, it } from "node:test";
import assert from "node:assert/strict";
import { parseMinor, addMinor } from "../../src/domain/money.ts";

describe("money", () => {
  it("parses pg's bigint strings without precision loss", () => {
    assert.equal(parseMinor("9007199254740993"), 9007199254740993n);
  });
  it("adds minor units exactly", () => {
    assert.equal(addMinor(1n, 2n), 3n);
  });
});
```

```bash
node --test "test/unit/**/*.test.ts"     # quote the glob: Node expands it, not Git Bash
node --test --watch "test/unit/**/*.test.ts"
node --test --test-name-pattern="money" "test/**/*.test.ts"
node --test --test-reporter=junit --test-reporter-destination=junit.xml "test/**/*.test.ts"
```

What you need to know about how it runs:

| Behaviour | Consequence |
|---|---|
| **Each test file runs in its own child process** by default, several files in parallel | Module-level state is fresh per file but **shared by every test inside a file** |
| Tests inside a file run **sequentially** unless you pass `{ concurrency: true }` | Opt in only for tests that share nothing |
| Hooks: `before`, `after`, `beforeEach`, `afterEach`, plus `t.after(fn)` inside a test | Per-test cleanup lives next to the thing it cleans up |
| `t.mock.fn()`, `t.mock.method(obj, "name", impl)` | Restored automatically when the test ends |
| `t.mock.timers` | Fake `setTimeout`/`setInterval`/`setImmediate` and `Date` (§6) |
| Coverage | `--experimental-test-coverage`: **experimental**, so treat its numbers as indicative |
| Snapshots (`t.assert.snapshot`) and module mocks (`mock.module`) | Newer additions that arrived behind experimental flags; check your version's docs before depending on them |
| `{ skip: "reason" }`, `{ todo: true }`, `{ timeout: ms }` per test | Platform-specific tests skip themselves with a reason |

```ts
import { test } from "node:test";
import assert from "node:assert/strict";
import { deliver } from "../../src/workers/deliver.ts";

test("a send that throws records a failed attempt", async (t) => {
  const send = t.mock.fn(async () => { throw Object.assign(new Error("reset"), { code: "ECONNRESET" }); });
  const markFailed = t.mock.fn(async (_id: string, _attempt: number) => {});
  await deliver({ id: "wd_1", attempt: 2 }, { send, repo: { markFailed } });
  assert.equal(send.mock.callCount(), 1);
  assert.deepEqual(markFailed.mock.calls[0]?.arguments, ["wd_1", 3]);
});
```

Notice the shape: `deliver` takes its collaborators as a parameter. **Dependency injection, not module mocking, is the Node testing default**, because ESM imports are live bindings to a module you can't swap after linking ([Lesson 04](../01-foundations/04-modules-cjs-and-esm.md)). Code that takes `{ send, repo, clock }` is testable with any runner; code that imports a singleton needs runner magic.

---

## 4. `node:test` vs Vitest vs Jest

| | `node:test` | Vitest | Jest |
|---|---|---|---|
| Install | Built in | Dev dependency | Dev dependencies plus a transform |
| TypeScript | Native type stripping (erasable syntax only) | Transformed by Vite/esbuild | Transformed by babel-jest or ts-jest |
| ESM | Native | Native | Still requires Node's `--experimental-vm-modules` |
| Module mocking | Prefer DI; `mock.module` is newer and flagged | `vi.mock`, hoisted by the transform | `jest.mock` (CJS); `jest.unstable_mockModule` for ESM |
| Fake timers | `mock.timers` | `vi.useFakeTimers`, with async advance helpers | `jest.useFakeTimers`, with async advance helpers |
| Isolation | A process per file | Workers or forks per file | A module registry per file |
| What runs in the test | **Exactly the code that runs in production** | Transformed code | Transformed code |

**How to choose:**
- **`node:test`** for Node services and libraries written as ESM + erasable TypeScript. No transform means no gap between tested and shipped code, and no dependencies means nothing for [Lesson 30](30-security-and-supply-chain.md) to worry about. **Ledger Engine uses it.**
- **Vitest** when you need module mocking of code you can't restructure, when you share tooling with a Vite frontend (the [React track](https://github.com/Pawar-Pratik/React-mastery)), or when the watch-mode UX matters most.
- **Jest** when the codebase is already on it and CJS. Migrate when the ESM friction (the VM-modules flag, `unstable_mockModule`) starts costing real time.

---

## 5. Testing async and evented code without sleeps

**A sleep in a test is a guess about the scheduler.** Too short and it flakes under load; long enough to be safe and the suite is slow. Wait for the **event** instead.

### `events.once`, armed before you act

```ts
import { once } from "node:events";

test("a slow SSE subscriber is evicted", async () => {
  const hub = createHub({ maxBufferedEvents: 2 });
  const client = fakeSseClient({ writeReturns: false });          // write() always says "full"
  hub.subscribe("mrc_1", client);

  const evicted = once(hub, "evicted", { signal: AbortSignal.timeout(1_000) }); // ✅ listen FIRST
  for (let id = 1; id <= 3; id++) hub.publish("mrc_1", { id });

  const [who, reason] = await evicted;
  assert.equal(who, client);
  assert.equal(reason, "slow_consumer");
  assert.equal(hub.subscriberCount("mrc_1"), 0);                 // cleanup happened too
});
```

Two details make this correct. **`emit` is synchronous** ([Lesson 11](../03-data-and-io/11-eventemitter.md)), so if you publish first and call `once` second, the event has already fired and you wait forever: arm the listener before triggering. And the **signal** turns a hang into a clear failure. `once` also rejects if the emitter emits `'error'` while you wait, so `assert.rejects(once(stream, "finish"), /boom/)` tests the failure path.

### A deferred: control exactly when a dependency finishes

```ts
test("stop() waits for the in-flight send", async () => {
  const { promise: gate, resolve: finishSend } = Promise.withResolvers<void>();
  const send = mock.fn(() => gate);                    // the send hangs until WE say so
  const dispatcher = createDispatcher({ send, claim: async () => [aDelivery()] });

  void dispatcher.tick();
  await waitFor(() => send.mock.callCount() === 1);

  let stopped = false;
  const stopping = dispatcher.stop().then(() => { stopped = true; });
  await flushMicrotasks();
  assert.equal(stopped, false);                        // ✅ still draining

  finishSend();
  await stopping;
  assert.equal(stopped, true);
});
```

### `waitFor`: poll a condition, with a deadline

```ts
// test/helpers/async.ts
export async function waitFor(
  check: () => boolean | Promise<boolean>,
  { timeoutMs = 2_000, intervalMs = 10 } = {},
): Promise<void> {
  const deadline = performance.now() + timeoutMs;
  while (!(await check())) {
    if (performance.now() > deadline) throw new Error(`waitFor: not true within ${timeoutMs} ms`);
    await new Promise((r) => setTimeout(r, intervalMs));
  }
}

/** One trip around the loop: every queued promise continuation runs first. */
export const flushMicrotasks = (): Promise<void> => new Promise((r) => setImmediate(r));
```

`waitFor` is fast on a fast machine and patient on a slow one, which is strictly better than a sleep in both directions. For a sequence of events, iterate them: `for await (const [row] of on(emitter, "row", { signal }))`.

---

## 6. Time: injected clocks and fake timers

### First choice: inject the clock

Ledger's webhook backoff lives in Postgres as `next_attempt_at` ([Lesson 26](../06-data-and-jobs/26-background-jobs-and-queues.md)), so the schedule is a pure function of the attempt number, the current time and a random number. Make those parameters and the test needs no timers at all:

```ts
test("in-process retry uses full jitter: 0.5 × 200, 400, 800 ms", async () => {
  const delays: number[] = [];
  const deps = { sleep: async (ms: number) => { delays.push(ms); }, random: () => 0.5 };
  const fn = mock.fn(async () => { throw Object.assign(new Error("reset"), { code: "ECONNRESET" }); });

  await assert.rejects(retry(fn, { attempts: 4, baseMs: 200, maxMs: 10_000 }, deps), /reset/);
  assert.equal(fn.mock.callCount(), 4);
  assert.deepEqual(delays, [100, 200, 400]);           // records the schedule; waits for nothing
});
```

The dispatcher's schedule gets the same treatment: inject `now` and `random`, then assert every delay sits inside its documented bounds for attempts 1–8, and that the maximum total stays within the ~3-day window. With `random` at its extremes you test both edges of the jitter.

### When the timer is inside the code: `mock.timers`

Some timers are the behaviour under test: the job-lease heartbeat, the poll loop's idle sleep, the shutdown deadline. Fake them:

```ts
test("the lease heartbeat fires every 10 s and stops with the job", (t) => {
  t.mock.timers.enable({ apis: ["setInterval"] });
  const extend = t.mock.fn(async () => {});
  const stop = startHeartbeat(extend, 10_000);

  t.mock.timers.tick(30_000);
  assert.equal(extend.mock.callCount(), 3);
  stop();
  t.mock.timers.tick(30_000);
  assert.equal(extend.mock.callCount(), 3);            // ✅ cleared: no leaked interval
});
```

### The microtask gap: why fake-timer tests with `await` "hang"

`tick()` runs due timer callbacks **synchronously**. If a callback resolves a promise, the code awaiting it continues in a **microtask**, which hasn't run yet when `tick()` returns. So the next assertion sees the old state, and the next `tick()` finds no timer scheduled because the continuation that would schedule it never ran.

```text
tick(1000) ──► timer fires ──► resolve(sleepPromise)
                                   │ continuation queued as a microtask   (not run yet)
assert(claim called twice) ❌ ◄────┘
await flushMicrotasks() ──► continuation runs ──► claim() ──► next sleep scheduled
assert(claim called twice) ✅
```

```ts
test("the poll loop sleeps exactly 1 s when nothing is due", async (t) => {
  t.mock.timers.enable({ apis: ["setTimeout"] });        // NOT setImmediate: flushMicrotasks uses it
  const claim = t.mock.fn(async () => []);
  const loop = startPollLoop({ claim, idleMs: 1_000 });

  await flushMicrotasks();                  assert.equal(claim.mock.callCount(), 1);
  t.mock.timers.tick(999); await flushMicrotasks(); assert.equal(claim.mock.callCount(), 1);
  t.mock.timers.tick(1);   await flushMicrotasks(); assert.equal(claim.mock.callCount(), 2);
  await loop.stop();
});
```

Two rules follow. **Interleave `tick` with a flush** whenever async code sits between timers. And **fake only the APIs the code under test uses**: make sure it calls the timer API you enabled (a helper built on `node:timers/promises` may not be intercepted, depending on your version), and never fake the one your flush helper relies on.

---

## 7. Testing streams

### Collect the output

```ts
import { text } from "node:stream/consumers";
const csv = await text(exportStream);                        // whole output as a string
const rows = await Array.fromAsync(objectStream);            // every object chunk
```

Fine for correctness on small inputs. It says nothing about memory.

### Test backpressure by bounding the producer's lead

Backpressure means the producer never runs far ahead of the consumer. So measure exactly that, against a deliberately slow consumer:

```ts
test("the export never runs more than its buffers ahead of a slow client", async () => {
  let produced = 0, consumed = 0, maxLead = 0;
  const cursor = Readable.from((function* () {
    for (let i = 0; i < 50_000; i++) { produced++; yield { id: `pi_${i}`, amount_minor: "4999" }; }
  })());
  const slowClient = new Writable({
    objectMode: true, highWaterMark: 16,
    write(_line, _enc, cb) {
      consumed++;
      maxLead = Math.max(maxLead, produced - consumed);
      setImmediate(cb);                                     // a client that reads slowly
    },
  });

  await pipeline(cursor, toCsvLines(), slowClient);         // toCsvLines: object in, one line out
  assert.equal(consumed, 50_001);                           // header + every row
  assert.ok(maxLead < 100, `producer ran ${maxLead} rows ahead`);
});
```

**The bound is the sum of the `highWaterMark`s on the path** (16 objects per object-mode buffer by default; byte-mode buffers are 64 KiB on Linux and macOS, 16 KiB on Windows, [Lesson 13](../03-data-and-io/13-streams-and-backpressure.md)). Compute it from your pipeline; don't guess it. An exporter that ignores `write()`'s return value passes a correctness test with a fast sink and fails this one with `maxLead` near 50,000, which is the OOM you'd otherwise meet at 4 million rows. A secondary assertion on `process.memoryUsage().heapUsed` growth is fine, but it's noisy; the lead is exact.

### Test that errors tear everything down

```ts
test("a failing row destroys every stage and releases the DB client", async () => {
  const release = mock.fn();
  const cursor = fakeCursor({ rows: 1_000, failAt: 500, onDestroy: release });
  const sink = new PassThrough().resume();

  await assert.rejects(pipeline(cursor, toCsvLines(), sink), /row 500/);
  assert.equal(cursor.destroyed, true);
  assert.equal(sink.destroyed, true);
  assert.equal(release.mock.callCount(), 1);                // exactly once: no leak, no double release
});
```

The same test with `.pipe()` instead of `pipeline()` fails the `destroyed` assertions, which is the whole argument of L13 in one test.

---

## 8. Testing graceful shutdown

Shutdown code runs once per deploy and never in a unit test. Test it the way production exercises it: a real process, real load, a real signal.

```ts
// test/process/shutdown.test.ts
import { spawn } from "node:child_process";
import { once } from "node:events";
import { createInterface } from "node:readline";

test("SIGTERM under load: zero failed requests, exit code 0", {
  skip: process.platform === "win32" && "needs POSIX signals: run in WSL2",
  timeout: 60_000,
}, async () => {
  const api = spawn(process.execPath, ["apps/api/main.ts"], {
    env: { ...process.env, PORT: "4010", SHUTDOWN_DRAIN_MS: "500" },
    stdio: ["ignore", "pipe", "inherit"],
  });
  for await (const line of createInterface({ input: api.stdout })) {
    if (line.includes('"msg":"listening"')) break;
  }
  api.stdout.resume();                       // keep draining, or the child blocks on a full pipe (L22)

  const load = startLoad("http://127.0.0.1:4010", { concurrency: 50 });
  await waitFor(() => load.completed > 500);
  api.kill("SIGTERM");
  const [code] = await once(api, "exit");
  const result = await load.stop();

  assert.equal(code, 0);
  assert.equal(result.failed, 0, JSON.stringify(result.errors.slice(0, 5)));
});
```

`startLoad` plays the load balancer: N workers loop over `fetch`, and a poller reads `/readyz` every 100 ms; once readiness returns 503, workers stop *starting* requests, exactly as an LB stops routing to a draining pod ([Lesson 24](../05-concurrency/24-graceful-shutdown.md)). Every non-2xx response and every network error on a request that was started counts as a failure. A handful of `ECONNRESET`s means the keep-alive race from [Lesson 17](../04-networking/17-http-from-scratch.md): fix the server's drain order, not the test.

**Windows:** `child.kill("SIGTERM")` there terminates the process forcefully, with no handler running, so this test is meaningless on Windows. The `skip` says so; run it in WSL2 and in Linux CI.

---

## 9. Real Postgres, Redis and HTTP

[API L26](https://github.com/Pawar-Pratik/API-mastery/blob/main/06-craft/26-testing-apis.md) has the method: Testcontainers or a CI service container, real migrations, a **unique merchant per test**, third parties mocked at the HTTP boundary. Two `node:test` specifics:

- **Files are processes**, so a container started in each file's `before()` means one container per file. Start Postgres once (a CI service container or `docker compose up` before `node --test`) and pass `DATABASE_URL`.
- **Unique tenants make files parallel-safe.** If tests truncate tables instead, run that suite with `--test-concurrency=1`.

Start the app on port `0` (the OS picks a free port) and read the real port from `server.address()`, so parallel files never collide.

---

## 10. Open handles: why the runner hangs

A test file's process exits when its loop is empty, the same rule as any Node process. **Anything your tests opened and didn't close keeps it alive.**

| Symptom | Usual culprit | Close it with |
|---|---|---|
| The file finishes ~10 s after its last test | A `pg.Pool` nobody ended: idle clients live for `idleTimeoutMillis` (10 s by default) | `after(() => pool.end())` |
| Never exits | A listening HTTP server | Your app's shutdown, or `server.close()` + `server.closeAllConnections()` |
| Never exits | A Redis client | `await redis.quit()` |
| Never exits | A `setInterval` (heartbeat, metrics, cache sweeper) | Clear it; `.unref()` it only if it's genuinely background |
| Never exits | A fetch response body never read | Consume or `cancel()` it ([Lesson 18](../04-networking/18-outbound-http.md)) |

To find the culprit, print what's open at the end of the file:

```ts
after(() => {
  console.error("open at exit:", process.getActiveResourcesInfo());   // experimental API
});
```

You'll see a few entries belonging to the runner itself; look for the `TCPSocketWrap`, `TCPServerWrap` and `Timeout` entries you created. The `why-is-node-running` package goes further and records where each handle was created.

`--test-force-exit` makes the runner exit regardless. **It hides the leak instead of fixing it**, and the same leak in production is a process that won't shut down cleanly. Use it only as a temporary CI unblock, with the handle dump above logged.

---

## 11. Property-based tests

An example-based test checks the inputs you thought of. A property-based test states a rule that must hold for **all** inputs, generates hundreds of them, and, when one fails, **shrinks** it to a minimal counterexample and prints a seed to replay it. With `fast-check`:

```ts
import fc from "fast-check";
import { sign, verify } from "../../src/domain/hmac.ts";

test("verify accepts every signature sign produced, and rejects any one-byte change", () => {
  fc.assert(fc.property(fc.uint8Array({ maxLength: 4096 }), fc.nat({ max: 4095 }), (bytes, i) => {
    const raw = Buffer.from(bytes);
    const now = 1_760_000_000;
    const header = sign(raw, SECRET, now);
    if (!verify(raw, header, SECRET, { now })) return false;
    if (raw.length === 0) return true;
    const tampered = Buffer.from(raw);
    const k = i % raw.length;
    tampered[k] = (tampered[k] ?? 0) ^ 0x01;
    return !verify(tampered, header, SECRET, { now });
  }));
});
```

The properties worth having in Ledger:

| Code | Property |
|---|---|
| `hmac.ts` | Sign-then-verify always passes; any one-byte change, a different secret or a stale timestamp always fails |
| `money.ts` | `parseMinor(String(n)) === n` for every 64-bit `bigint`; addition is exact and commutative |
| `parseRow` (the ingester) | **Never throws** on arbitrary strings: it returns a record or a rejection with a reason. A parser that throws stops a 2 GB import at row 1,834,201 |
| `splitLines` | **The output doesn't depend on how the input is chunked**: generate random chunk boundaries over the same bytes and assert identical lines ([Lesson 14](../03-data-and-io/14-composing-streams.md)) |

That last one is the best property in the track: chunk boundaries are where every hand-written stream parser breaks, and no human writes the example that splits a `₹` between two chunks.

---

## 12. Test isolation in one process

| Shared thing | How it leaks between tests | Fix |
|---|---|---|
| **Module singletons** (`export const pool = new Pool()`) | ESM has no `require.cache` to clear, so every test in a file shares them | Factories: `createApp({ pool, clock })`; files are separate processes anyway |
| **`process.env`** | One test sets `FEATURE_X=1`, the next inherits it | Snapshot in `beforeEach`, restore in `afterEach` |
| **Global mocks** | `mock.method` on the module-level `mock` isn't auto-restored | Use `t.mock`, or `mock.restoreAll()` in `afterEach` |
| **ALS context** | Code that calls `als.enterWith()` leaves its store in place for whatever runs next in the chain | `als.run()` everywhere; assert `currentContext() === undefined` at the start of tests that care |
| **Database rows** | Test B sees test A's payments | A unique merchant per test (API L26) |

The ALS leak deserves a concrete picture, because it's a production bug wearing a test-isolation costume:

```ts
async function processJob(job: { merchantId?: string }): Promise<string | undefined> {
  if (job.merchantId) ctxStorage.enterWith({ merchantId: job.merchantId });   // ❌
  await Promise.resolve();
  return ctxStorage.getStore()?.merchantId;
}

test("a job without a merchant must not inherit the previous job's", async () => {
  assert.equal(await processJob({ merchantId: "mrc_A" }), "mrc_A");
  assert.equal(await processJob({}), undefined);    // 💥 "mrc_A": enterWith outlived its job
});
```

Two awaited calls in sequence are exactly what a worker loop does, and what a test runner does between tests. Replace `enterWith` with `ctxStorage.run(store, () => …)` and the second assertion passes.

---

## 13. Ledger Engine's suite

```text
test/
├── unit/          money, backoff schedule, hmac, csv, parseRow       pure, milliseconds each
├── streams/       export lead-bound, export teardown, ingest routing  fake cursors, slow sinks
├── properties/    fast-check: hmac, money, parseRow, splitLines
├── integration/   real Postgres + HTTP on port 0, unique merchant per test
├── process/       shutdown under load, crash guards, exit codes      spawn-based, POSIX only
└── helpers/       waitFor, flushMicrotasks, fakeCursor, startLoad, startApp
```

```jsonc
// package.json
"scripts": {
  "test":             "node --test \"test/unit/**/*.test.ts\" \"test/streams/**/*.test.ts\" \"test/properties/**/*.test.ts\"",
  "test:integration": "node --test \"test/integration/**/*.test.ts\"",
  "test:process":     "node --test \"test/process/**/*.test.ts\"",
  "test:coverage":    "node --test --experimental-test-coverage \"test/**/*.test.ts\""
}
```

The ingester test feeds a CSV with known bad rows (a missing column, a negative amount, a 2^63 amount) among valid ones, and asserts the inserted count, the `rejects.csv` line count, and that each rejection carries its source row number.

---

## 14. Production rules

| Rule | Why |
|---|---|
| **No sleeps in tests: wait for events or conditions with a deadline** | Sleeps are slow on fast machines and flaky on slow ones |
| **Arm `once()` before triggering** | `emit` is synchronous; a late listener waits forever |
| **Inject the clock, `sleep` and `random`; fake timers only when the timer is the behaviour** | Recording a schedule is exact and instant |
| **Interleave `tick()` with a microtask flush** | Promise continuations don't run inside `tick()` |
| **Test streams against a slow consumer and bound the producer's lead** | A fast consumer never exercises backpressure |
| **Assert teardown: `destroyed` on every stage, resources released exactly once** | Error paths are where streams leak |
| **Test shutdown with a real process, real load and a real signal, on POSIX** | It's the only way the code path runs |
| **Close every handle in `after`; never paper over with `--test-force-exit`** | A hang in tests is a shutdown bug in production |
| **Property tests for parsers, money and crypto** | They find the input you'd never write |
| **DI over module mocking** | ESM bindings can't be swapped; injected code works with any runner |
| **`als.run()`, never `enterWith()`** | `enterWith` leaks context into whatever runs next |

> **Spring equivalent:** Awaitility is `waitFor`, Mockito is `t.mock`, and the pyramid and CI staging are the same ideas. See [Spring Boot Lesson 29](https://github.com/Pawar-Pratik/SpringBoot/blob/main/08-testing/29-testing-pyramid-and-unit.md) and [Lesson 31](https://github.com/Pawar-Pratik/SpringBoot/blob/main/08-testing/31-advanced-testing-strategy.md). The Node-specific part is the event loop: there's no thread to block on with `Future.get()`, so every wait is a promise you must be able to resolve.

---

## 15. Interview traps

**Q1. "How do you test code that uses timers?"**
First, inject time: make `sleep`, `now` and `random` parameters, then record the schedule and assert it, with no waiting. When the timer is the behaviour (a heartbeat, a poll interval), use fake timers, and know the trap: `tick()` runs callbacks synchronously, but code awaiting them resumes in a microtask, so you interleave each `tick` with a flush. The weak answer is "a short `setTimeout` in the test", which is slow and flaky.

**Q2. "How do you test an EventEmitter-based API?"**
Arm `events.once(emitter, name, { signal })` *before* triggering, because `emit` is synchronous, then await it; the signal turns a missing event into a failure instead of a hang. `once` rejects on `'error'`, which covers failure paths. Also assert cleanup: subscriber counts or `listenerCount` back to zero after the client goes away, which is how you catch listener leaks.

**Q3. "Why does my test runner hang after all tests pass?"**
An open handle keeps the test process's loop alive: a server still listening, a pool not ended, a Redis client, an interval, an unconsumed fetch body. If the file finishes about ten seconds late, it's usually a `pg` pool, whose idle clients live for `idleTimeoutMillis`. Find it with `process.getActiveResourcesInfo()` or `why-is-node-running`, and close it in `after`. `--test-force-exit` hides the bug.

**Q4. "`node:test`, Vitest or Jest?"**
`node:test` for ESM + erasable TypeScript Node code: built in, no transform, so the tested code is the shipped code, and no dependencies. Vitest when you need transform-powered module mocking or share a Vite toolchain. Jest when you're already on it; its ESM support still runs through Node's experimental VM-modules flag. Then say the deciding factor: whether the code is written with injected dependencies.

**Q5. "How do you test backpressure?"**
Pipe the producer into a deliberately slow Writable with a small `highWaterMark`, count produced and consumed items, and assert that the producer's lead never exceeds the sum of the buffers on the path. A producer that ignores `write()`'s return value passes every correctness test and fails this one by tens of thousands of items.
**Follow-up:** "Why not just measure memory?" Heap measurements are noisy and GC-dependent; the lead is an exact count.

**Q6. "How do you test graceful shutdown?"**
Spawn the real server, wait for its ready line, keep draining its stdout, run load from a client that behaves like a load balancer (stop starting requests once `/readyz` fails), send SIGTERM, and assert exit code 0 and zero failed requests. On Windows `kill("SIGTERM")` is a forceful kill, so it runs on Linux or WSL2.

**Q7. "Why can't you mock an ES module the way you mock a CommonJS one?"**
CJS modules live in a mutable `require.cache` you can overwrite before the code under test requires them. ESM imports are bound at link time to the exporting module's live bindings, so after linking there's nothing to replace. Tools work around it with transforms (Vitest hoists `vi.mock`) or loader hooks (`mock.module`, Jest's `unstable_mockModule`). The robust answer is to design for injection.

**Q8. "What does property-based testing find that examples don't?"**
Inputs nobody thinks to write: empty buffers, multi-byte characters split across chunk boundaries, `bigint` values at 2^63, strings that are almost valid. It states an invariant ("verify accepts what sign produced", "line splitting is independent of chunking"), generates many inputs, and shrinks a failure to a minimal case. It's strongest on parsers, money and crypto.

**Q9. "A test passes alone and fails in the full suite. What do you check?"**
Shared state between tests in the same process: module singletons, `process.env`, un-restored global mocks, ALS context leaked by `enterWith`, and database rows from another test. Then order dependence: run the file with the failing test moved first. A unique merchant per test and factories instead of singletons remove most of it.

**Q10. "Your stream test is green but production OOMs on the same export. How?"**
The test's consumer was faster than the producer, so buffers never filled and the backpressure path never ran. Production clients are slow. Rewrite the test with a slow sink and a lead bound, and it fails the same way production did.

---

## 16. Build & break

### Build — the Ledger Engine test suite
Create the layout and scripts from §13 and the helpers (`waitFor`, `flushMicrotasks`, `fakeCursor`, `startLoad`, `startApp`). Write the five Ledger tests: the retry and backoff schedules with injected time plus the heartbeat with `mock.timers`; the export lead-bound and teardown tests; the ingester's bad-row routing; the HMAC and `splitLines` properties; and shutdown under load. `npm test` should finish in seconds and exit on its own.

### Break — five experiments
1. **The sleeping test.** Test `retry` by calling it with real timers and `await new Promise(r => setTimeout(r, 3000))` before asserting. Time the suite, then run it while another terminal pegs the CPU (`node -e "for(;;){}"`) and watch it fail intermittently. Rewrite with the injected `sleep`: milliseconds, and immune to load.
2. **The hanging runner.** In an integration test, create a `pg.Pool`, run one query, and don't end it. Watch the file finish about 10 s late. Set `idleTimeoutMillis: 0` and watch it never finish. Find it with the `getActiveResourcesInfo()` dump, then fix it with `after(() => pool.end())`.
3. **The false-pass stream.** Write an exporter that does `for await (const row of cursor) res.write(toCsv(row))`, ignoring the return value. Test it with an instant sink: green. Test it with §7's slow sink and lead bound: red, with `maxLead` in the tens of thousands. Fix it with `pipeline` and watch it go green.
4. **The ALS leak.** Run §12's `processJob` test as written and watch the second assertion receive `mrc_A`. Switch to `run()`, and add the `currentContext() === undefined` guard to the integration suite.
5. **The microtask gap.** Take §6's poll-loop test and delete the `flushMicrotasks()` after `tick(1)`. The count stays at 1 even though a second has "passed". Put it back, and explain why to yourself in one sentence.

### Explain out loud (60–120 seconds)
1. Why sleeps are wrong, and the three replacements: events, deferreds, `waitFor`.
2. Injected clocks vs fake timers, and the microtask gap.
3. How you prove backpressure works with one assertion.
4. How to test graceful shutdown, and why not on Windows.
5. Why runners hang, the ten-second clue, and why `--test-force-exit` is a smell.

---

## What's next

Your code is tested. Now it ships to other developers: the `@ledger/node` SDK and the `ledger` CLI. That means learning what `package.json` actually controls: `exports` maps that both Node and TypeScript must read the same way, semver's special case for `0.x`, lockfiles that consumers ignore, and the one `.npmignore` that publishes your `.env`.

Next → **[Lesson 32: Packages: exports, publishing & monorepos](32-packaging-and-publishing.md)**
