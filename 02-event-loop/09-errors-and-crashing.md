# Lesson 09 — Errors: operational vs programmer, and crash-only design

> **Why this lesson exists:** Node delivers errors through four different channels, and each one fails differently when nobody is listening. Handle them inconsistently and you get the two classic Node incidents: a process that limps on with corrupted state after an exception it should have died from, and a process that crash-loops because of a typo in a log line. This lesson gives you the classification (expected failure vs bug), the mechanics of every channel, and the design that makes crashing safe instead of scary.

**Time:** ~70 minutes · **Prereq:** Lesson 06

---

## 1. The idea in one sentence

> **Handle the failures you expected, crash on the ones you didn't, and build the system so that a crash costs one retry instead of an outage.**

---

## 2. Two kinds of failure, one `catch` block

Here is the handler most codebases start with:

```ts
// ❌ One catch for everything
try {
  await chargeAndRecord(payment);
} catch (err) {
  logger.warn({ err }, "charge failed, will retry");
  await scheduleRetry(payment);
}
```

That block treats two completely different situations the same way:

- The processor returned `ECONNREFUSED` because it's restarting. **The world failed.** Retrying in a few seconds is exactly right.
- `chargeAndRecord` threw `TypeError: Cannot read properties of undefined (reading 'currency')` because a refactor broke it. **Your program is wrong.** Retrying runs the same broken code again, forever, and every attempt may leave something half-done.

Joyent's classic guide, *Error Handling in Node.js*, named the distinction, and it's still the one interviewers want:

| | **Operational error** | **Programmer error** |
|---|---|---|
| What it is | An expected runtime failure of the environment | A bug: the code is wrong |
| Examples | `ECONNREFUSED`, a timeout, invalid input, `ENOENT` on a user-supplied path, a 429 from a merchant | Reading a property of `undefined`, passing a string where a number belongs, a broken invariant, an unawaited promise |
| Can correct code prevent it? | No. It will happen in production | Yes. Fix the code |
| What you do | **Handle it**: retry, fall back, return 4xx/5xx, alert | **Crash** (or fail the request, if the damage is provably contained). Then fix the bug |

**The same error code can be either kind.** `ENOENT` when opening a path the merchant gave you is operational. `ENOENT` on a temp file your own code wrote one line earlier is a bug. Classify by whether the program *expected* the failure, not by its type.

Why "crash" and not "log and continue"? Because a programmer error means the program is in a state you didn't design for. Handling code can't repair a state it doesn't know about. **The only reliable recovery from an unknown state is a known state, and a fresh process is the cheapest known state you have.** That's the fourth of the track's five ideas: the process is the unit of failure.

---

## 3. The four channels

Node delivers errors four ways. Each has its own "how to catch" and its own "what happens if nobody does".

| Channel | Example | How you catch it | If nobody catches it |
|---|---|---|---|
| **`throw`** (synchronous) | `JSON.parse("{")`, `fs.readFile(123)` → `ERR_INVALID_ARG_TYPE` | `try/catch` **on the same stack** | `'uncaughtException'` → stderr trace → exit 1 |
| **Callback `err`** | `fs.readFile(p, (err, data) => …)` | `if (err) return …` as the first line | **Nothing.** It's silently ignored |
| **Promise rejection** | `await fetch(url)` | `try/catch` around `await`, or `.catch()` | `'unhandledRejection'` → (default mode) treated as uncaught → exit 1 ([L06](06-promises-and-microtasks.md)) |
| **`'error'` event** | `socket.on("error", …)`, a stream, a server | A listener for `'error'` | `emit("error")` **throws**, usually from a libuv callback → uncaught → exit 1 |

### `try/catch` only sees its own stack

```ts
// ❌ The catch never runs. The timer callback executes on a later, different stack.
try {
  setTimeout(() => { throw new Error("late"); }, 10);
} catch (err) {
  console.log("never printed");
}
```

By the time the callback runs, the `try` block finished long ago: its stack frame is gone ([L05](05-the-event-loop.md)). The throw unwinds a stack whose bottom frame is the event loop itself, so there's nothing to catch it but the process-level handler. **This is why every async API reports errors through a channel (callback, rejection, event) instead of throwing.** `await` is what lets `try/catch` work again, because the continuation after `await` is resumed *inside* the `try`.

### The silent channel

The callback convention is the only one where ignoring the error is silent:

```ts
fs.writeFile(path, data, () => {});        // ❌ disk full? permission denied? You'll never know
fs.writeFile(path, data, (err) => {        // ✅
  if (err) return done(err);
  done(null);
});
```

Prefer the promise APIs (`node:fs/promises`): an ignored rejection at least crashes loudly.

### The event channel throws when unheard

```ts
import { EventEmitter } from "node:events";

new EventEmitter().emit("error", new Error("db down"));  // 💥 throws Error: db down
new EventEmitter().emit("error", "db down");             // 💥 throws ERR_UNHANDLED_ERROR: Unhandled error. ('db down')
```

An `'error'` with no listener is thrown, and a non-`Error` value is wrapped in `ERR_UNHANDLED_ERROR` first. Since sockets, servers, streams and child processes emit `'error'` from inside libuv callbacks, "thrown" in practice means `uncaughtException`. **Every emitter you create or receive needs an `'error'` listener**, or must be consumed through an API that attaches one for you (`pipeline()`, `events.once()`). `events.errorMonitor` lets you observe errors (for metrics) without counting as a handler: the throw still happens. [L11](../03-data-and-io/11-eventemitter.md) covers the emitter rules in full.

### One function, one channel

A function must report failure through exactly one channel. Code that sometimes throws and sometimes calls back (or rejects *and* emits) forces every caller to guard both, and the one they forget is the one that fires in production. This is the error-side version of Zalgo ([L06](06-promises-and-microtasks.md)). **Validate arguments synchronously (a throw is a programmer error), and report runtime failures asynchronously through your one channel.**

---

## 4. Compare codes, never messages

Errors from the OS (through libuv) arrive as `SystemError`s with structured fields:

```ts
fs.readFile("missing.json", (err) => {
  // err.code    "ENOENT"          ← stable across versions and platforms. Branch on this.
  // err.errno   -2 on Linux, -4058 on Windows   ← platform-specific. Never compare it.
  // err.syscall "open"
  // err.path    "/app/missing.json"
  // err.message "ENOENT: no such file or directory, open '/app/missing.json'"  ← contains data; changes
});
```

**Messages are for humans and change between Node versions; they also embed paths and addresses, so string-matching them is both brittle and wrong.** Branch on `code`.

Node uses two code families, and they map neatly onto the classification:

- **`ERR_*`** (`ERR_INVALID_ARG_TYPE`, `ERR_INVALID_URL`, …): Node's own argument validation, thrown synchronously. **Almost always a programmer error.**
- **`E*`** (`ENOENT`, `ECONNRESET`, …): the operating system reporting a runtime failure. **Usually operational.**

### The codes you must know

| Code | Meaning | Typical cause | Retry? | Depth in |
|---|---|---|---|---|
| `ENOENT` | No such file or directory | Wrong path, wrong cwd, file not created yet | No | [L15](../03-data-and-io/15-the-file-system.md) |
| `EACCES` / `EPERM` | Permission denied / not permitted | File modes; binding port < 1024 unprivileged on Linux; Windows file locks | No | [L15](../03-data-and-io/15-the-file-system.md) |
| `EADDRINUSE` | Address already in use | A second instance, or the old process hasn't exited yet | No: fail boot, let the supervisor retry | [L17](../04-networking/17-http-from-scratch.md) |
| `ECONNREFUSED` | Nothing listening at that address | Dependency down or restarting | Yes, with backoff | [L18](../04-networking/18-outbound-http.md) |
| `ECONNRESET` | Peer reset an established connection | Peer crashed, idle connection cut by an LB, keep-alive race | Only if idempotent (it may have been processed) | [L16](../04-networking/16-tcp-and-net.md) |
| `EPIPE` | Wrote to a pipe/socket the peer closed | Client went away mid-response; `node app.ts \| head` | No: stop writing | [L16](../04-networking/16-tcp-and-net.md) |
| `ETIMEDOUT` | An OS-level operation timed out | Blackholed host, dropped packets | Yes, with backoff | [L18](../04-networking/18-outbound-http.md) |
| `EMFILE` | Too many open files in this process | Unbounded concurrency, leaked sockets/handles | No: fix the code (bound it) | [L10](10-cancellation-and-concurrency.md) |
| `ENOTFOUND` | DNS name doesn't resolve | Typo, missing record (`EAI_AGAIN` is the *temporary* DNS failure: retry that one) | No | [L07](07-the-thread-pool.md) |

### Codes hide inside wrappers

Higher-level APIs wrap the system error. You have to look underneath:

```text
fetch("http://127.0.0.1:59999/")  →  TypeError: fetch failed
                                       cause: Error { code: "ECONNREFUSED" }

fetch("http://localhost:59999/")  →  TypeError: fetch failed
                                       cause: AggregateError { code: "ECONNREFUSED",
                                                errors: [ …::1 refused, …127.0.0.1 refused ] }
```

`localhost` resolves to both `::1` and `127.0.0.1`, Node tries both, and reports the combined failure as an `AggregateError` with one entry per address. **A classifier that only reads `err.code` sees `undefined` for every `fetch` failure** and treats a restarting dependency as a bug. Walk the `cause` chain:

```ts
/** First string `code` on the error or anywhere down its cause chain. */
export function systemCode(err: unknown, depth = 0): string | undefined {
  if (!(err instanceof Error) || depth > 5) return undefined;
  const { code } = err as NodeJS.ErrnoException;
  return typeof code === "string" ? code : systemCode(err.cause, depth + 1);
}
```

---

## 5. Error classes that carry decisions

Callers need three things from an error: **what happened** (a stable `code`), **what to tell the client** (a status), and **whether trying again can help** (`retryable`). Put all three on the error, and chain the original with `cause`.

One Node-specific catch: the API track's `ApiError` declared its fields as constructor parameter properties (`constructor(readonly status: number, …)`). **Parameter properties aren't erasable, so `node errors.ts` refuses to run them.** Declare the fields and assign them:

```ts
export type ErrorKind = "domain" | "operational";
export type LedgerErrorOptions = { retryable?: boolean; cause?: unknown; extra?: Record<string, unknown> };

export class LedgerError extends Error {
  readonly kind: ErrorKind;
  readonly code: string;              // stable, snake_case, part of the API contract
  readonly status: number;
  readonly retryable: boolean;
  readonly extra: Record<string, unknown>;

  constructor(kind: ErrorKind, status: number, code: string, message: string, opts: LedgerErrorOptions = {}) {
    super(message, { cause: opts.cause });   // Error.cause: Node prints the whole chain
    this.name = new.target.name;             // "DomainError" / "UpstreamError" in stack traces
    this.kind = kind;
    this.status = status;
    this.code = code;
    this.retryable = opts.retryable ?? false;
    this.extra = opts.extra ?? {};
  }
}

/** Expected business outcome: 4xx, never retried unchanged. */
export class DomainError extends LedgerError {
  constructor(status: number, code: string, message: string, extra?: Record<string, unknown>) {
    super("domain", status, code, message, extra ? { extra } : {});
  }
}

/** A dependency failed: 502/503/504. Transient unless the caller says otherwise. */
export class UpstreamError extends LedgerError {
  constructor(status: 502 | 503 | 504, code: string, message: string, opts: LedgerErrorOptions = {}) {
    super("operational", status, code, message, { retryable: true, ...opts });
  }
}
```

There is deliberately **no `ProgrammerError` class**. A bug is whatever reaches the boundary *without* being a `LedgerError`: you don't get to predict your bugs.

Three tools worth knowing precisely:

- **`Error.cause`** keeps the original error and its stack; `console.error` and pino print the chain as `[cause]`. Never `new Error("failed: " + err)`, which flattens the stack you needed. The typing side (`catch` is `unknown`, `toError`, `Result` at boundaries) is in [TS L16](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/16-async-and-errors.md).
- **`AggregateError`** carries several failures in `errors`: `Promise.any` rejects with one, Node's multi-address connect produces one, and it's the right shape for "3 of 50 batch rows failed".
- **`Error.captureStackTrace(target, fn)`** is V8's API for recording a stack *starting above* `fn`. A subclass's `super()` already captures one, so you use this to hide helper frames (an `assert()` helper that shouldn't appear at the top of every trace) or to give a plain object a stack. `Error.stackTraceLimit` defaults to 10 frames; raising it in development is cheap and often decisive.

**And throw `Error` objects, never strings.** `throw "insufficient funds"` produces no stack at all: Node prints the bare string and suggests `--trace-uncaught`. You lose the one thing that tells you where it came from.

The classifier at a downstream boundary turns a raw failure into a decision, and **rethrows anything it can't name**:

```ts
const RETRYABLE = new Set(["ECONNREFUSED", "ECONNRESET", "ETIMEDOUT", "EAI_AGAIN", "UND_ERR_CONNECT_TIMEOUT"]);

export function classify(err: unknown, dependency: string): LedgerError {
  if (err instanceof LedgerError) return err;
  if (err instanceof DOMException && err.name === "TimeoutError")
    return new UpstreamError(504, "upstream_timeout", `${dependency} timed out`, { cause: err });
  const code = systemCode(err);
  if (code && RETRYABLE.has(code))
    return new UpstreamError(503, "upstream_unavailable", `${dependency} unavailable (${code})`, { cause: err });
  if (code)
    return new UpstreamError(502, "upstream_error", `${dependency} failed (${code})`, { cause: err, retryable: false });
  throw err;                            // no code and not ours: a bug. Don't dress it up as a 502.
}
```

`retryable` says the failure is *transient*. Whether you may actually retry also depends on whether the operation is safe to repeat: an `ECONNRESET` after the request was sent might mean the charge happened. That half of the decision is the idempotency design in [API L18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md).

---

## 6. The last line of defence: `uncaughtException` and `unhandledRejection`

An error reaches the process level when it escapes every other channel: a throw from a timer or I/O callback, an `'error'` event with no listener, or a rejection nobody handled (in the default `--unhandled-rejections=throw` mode, that's raised as an uncaught exception).

### Why you must exit

A throw unwinds the stack **in the middle of whatever was running**. Every function on that stack was partway through maintaining some invariant:

```text
applyRefund(paymentId, 500)
  ├─ cache.available.set(m, avail - 500)    ✅ ran
  ├─ formatAudit(refund.reason.trim())      💥 TypeError: reason is undefined
  └─ cache.pending.set(m, pend + 500)       ✗ never ran

Result: the in-memory balance cache now says 500 left "available"
and never arrived in "pending". The money vanished from this process's view.
```

Other half-applied states a throw leaves behind: a pool client checked out and never released, a lock or semaphore slot never freed, an in-flight counter incremented but never decremented, a stream half-written. **The `uncaughtException` handler receives the error, not the state, so it can't know which of these happened.** Node's documentation is explicit that it's a last-resort mechanism and that resuming normal operation afterwards is unsafe.

### The correct handler

Log with context, stop taking new work, flush, exit, and **never wait forever to do so**:

```ts
type FatalDeps = {
  log: (obj: Record<string, unknown>, msg: string) => void;
  flush: () => Promise<void>;           // wraps your logger's flush (pino buffers asynchronously)
  stopAccepting?: () => void;           // e.g. server.close(): no new requests on a broken process
  timeoutMs?: number;
};

let dying = false;

export function installProcessGuards({ log, flush, stopAccepting, timeoutMs = 3_000 }: FatalDeps): void {
  const die = (err: unknown, origin: string) => {
    if (dying) return;                    // a second fatal error while dying: already on it
    dying = true;
    process.exitCode = 1;
    log({ err, origin, pid: process.pid }, "fatal_error");
    setTimeout(() => process.exit(1), timeoutMs).unref();   // hard deadline; unref'd so it never holds the loop
    try { stopAccepting?.(); } catch { /* exiting regardless */ }
    flush().finally(() => process.exit(1));
  };
  process.on("uncaughtException", (err, origin) => die(err, origin));
  process.on("unhandledRejection", (reason) => die(reason, "unhandledRejection"));
}
```

Why each line exists:

- **`process.exit(1)` rather than just `exitCode`.** Normally `exitCode` is the polite choice ([L03](../01-foundations/03-the-process.md)), but a broken process may never drain: an open server keeps the loop alive indefinitely. After a fatal error you exit deliberately.
- **The unref'd timeout.** If `flush()` hangs (a dead log destination), the process must still die. This is the same safety net graceful shutdown uses ([L24](../05-concurrency/24-graceful-shutdown.md)).
- **The `dying` flag.** Fatal errors often cascade; the second one shouldn't restart the sequence.
- **Don't throw inside the handler.** If your `uncaughtException` handler itself throws, Node exits with **code 7** instead of 1, and whatever you were logging is lost.

Two related tools:

- **`process.on("uncaughtExceptionMonitor", (err, origin) => …)`** observes a fatal error *without* replacing the default behaviour: the process still prints and exits 1. `origin` tells you whether it came from `"uncaughtException"` or `"unhandledRejection"`. Use it for a metrics counter or a crash breadcrumb when you're happy with Node's default exit.
- **The rejection modes.** `--unhandled-rejections` accepts `throw` (default), `strict`, `warn`, `none` and `warn-with-error-code`. **Installing an `'unhandledRejection'` handler disables the default crash**, so a handler that only logs turns every floating promise into silent corruption. Either don't install one, or exit from it, as above.

### Why `domain` is deprecated

Node's `domain` module (marked deprecated in the docs for years; it still loads without a warning) tried to make "catch async errors per request and keep serving" work. Emitters, timers and callbacks created while a domain was active were implicitly bound to it, so an error anywhere in a request's async tree was routed to that request's domain handler.

It failed for three reasons, and they're the interview answer:

1. **Catching doesn't unwind state.** The domain handler had the same problem as `uncaughtException`: the half-applied invariants were still half-applied. Per-request error routing didn't make continuing any safer.
2. **Implicit binding leaked across requests.** Pooled, long-lived resources were bound to whichever domain was active when they were created, so a later request's error could be routed to an earlier request's handler.
3. **Overhead and magic** in every async path, for a guarantee it couldn't keep.

What replaced it is a split of responsibilities: proper error channels for expected failures, `AsyncLocalStorage` for per-request context ([L29](../07-production/29-observability.md)), and crash-only design for everything else.

---

## 7. Crash-only design

Candea and Fox's 2003 paper *Crash-Only Software* makes an argument that sounds backwards until it clicks: **your system must survive crashes anyway** (OOM kills, `kill -9`, node failures, power loss), so make crashing the *only* way to stop and recovery the *only* way to start. Then the crash path is exercised constantly instead of being the untested code that runs once a year at 3 a.m.

For Node, crash-only isn't a slogan; it's a list of properties Ledger must have so that "exit on a bug" is cheap:

| Property | How Ledger gets it | Why a crash is then safe |
|---|---|---|
| **Atomic state changes** | Payment row + `outbox` row in one Postgres transaction ([API L18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md)) | A crash mid-request leaves all of it or none of it |
| **Retryable requests** | Required `Idempotency-Key` on money-moving POSTs | The client's retry after the dropped connection replays or re-executes safely |
| **Durable work** | Deliveries in `webhook_deliveries` with `next_attempt_at`; jobs held by leases ([L26](../06-data-and-jobs/26-background-jobs-and-queues.md)) | A crashed worker's job is picked up again, never lost |
| **No state that matters in memory** | Caches rebuildable from Postgres; retries in the database, not in `setTimeout` ([L05](05-the-event-loop.md)) | The restarted process loses nothing it can't rebuild |
| **A supervisor** | Kubernetes `restartPolicy: Always` (restarts back off exponentially, capped at five minutes, which is `CrashLoopBackOff`); systemd `Restart=on-failure` | Something brings the process back without a human |
| **Redundancy** | N API replicas behind the load balancer | One crash drops that process's in-flight requests, not the service |

The trade is real: **a crash drops whatever that process had in flight.** Crash-only accepts that cost because the alternative, a zombie process confidently serving wrong balances, is far worse, and because idempotent retries turn "dropped" into "delayed".

### Don't crash-loop on poison

Crash-only needs one guard. If a worker crashes on a specific message (a malformed payload that triggers a bug), the message is redelivered after the restart and crashes the next worker too. Every replica ends up in `CrashLoopBackOff` over one row. The fix is an attempt counter and a dead-letter path ([L26](../06-data-and-jobs/26-background-jobs-and-queues.md)): **crash on the bug, but quarantine the input that triggers it.**

### Where's the boundary? Request scope vs process scope

Does every `TypeError` crash the API? No, and the reasoning is what interviewers probe. Inside an Express handler, the framework catches the throw (or, in Express 5, the rejection) and calls your error middleware. **If the request's code only touched request-scoped state** (its own variables, its own transaction that will roll back), the damage is contained: return a 500, log at `error`, alert, keep serving. **If the error escaped to the process** (a timer, an emitter listener, a worker loop, anything outside a request) or the code was mutating shared in-memory state, nothing contained it: crash.

That's also how you avoid the opposite failure from the opening of this lesson, the process that **crashes on a typo in a log line**. A bad `logger.info(\`paid ${payment.customer.email}\`)` inside a handler becomes one 500, not a dead process. The same line in a background worker loop with no boundary crash-loops the worker. Give every unit of work (request, job, message) a boundary that catches, records and fails *that unit*, and let only what escapes the boundaries reach the process guard.

---

## 8. Errors at the HTTP edge (Express 5)

In **Express 5, a promise rejected by an async handler reaches the error middleware**. In Express 4 it didn't: the rejection floated, and since Node 15 that meant the process crashed. ([L19](../04-networking/19-frameworks-by-mechanism.md) has the side-by-side.)

One error middleware maps the taxonomy to RFC 9457 problem+json. The format and the code catalogue are in [API L10](https://github.com/Pawar-Pratik/API-mastery/blob/main/02-rest-design/10-errors-and-problem-details.md); here's the runtime half:

```ts
import type { ErrorRequestHandler } from "express";
import { LedgerError, toProblem } from "./errors.ts";

export const errorMiddleware: ErrorRequestHandler = (err, req, res, next) => {
  if (res.headersSent) return next(err);   // mid-stream: let Express close the connection
  const problem = toProblem(err, String(res.locals.requestId));
  // req.log: the request-scoped pino child from pino-http (L29)
  if (!(err instanceof LedgerError)) req.log.error({ err }, "unhandled_error");   // a bug: page someone
  else if (err.status >= 500) req.log.error({ err, code: err.code }, "upstream_failure");
  if (err instanceof LedgerError && err.retryable) res.set("Retry-After", "2");
  res.status(problem.status).type("application/problem+json").json(problem);
};
```

`res.headersSent` matters for streaming responses: once bytes are on the wire you can't send a problem document, and delegating to Express's default handler makes it close the connection so the client sees a failure instead of a truncated "success".

### Ledger's error taxonomy

| Kind | Examples | HTTP | Retryable | Log | Process |
|---|---|---|---|---|---|
| **Domain** | `validation_failed`, `insufficient_funds`, `invalid_state_transition` | 4xx | No | `info`/`warn` | Continue |
| **Operational, downstream** | Processor `ECONNREFUSED`, timeouts, pool acquisition timeout | 502 / 503 / 504 + `Retry-After` | Yes (if the operation is idempotent) | `error` + breaker metrics | Continue |
| **Programmer, request scope** | `TypeError` in a handler that touched only request state | 500 `internal_error` | Client may retry with its idempotency key | `error` + alert | Continue |
| **Programmer, process scope** | Throw in a timer, listener or worker loop; any unhandled rejection | none | n/a | `fatal` | **Exit 1; supervisor restarts** |

---

## 9. Post-mortem tools: core dumps and diagnostic reports

When a crash's stack trace isn't enough, Node can capture more:

- **`--abort-on-uncaught-exception`** calls `abort()` instead of exiting, producing a core dump (on Linux, enable dumps with `ulimit -c unlimited`; the exit status is 134 = 128 + SIGABRT). The core holds the whole heap at the moment of the throw, for a debugger to inspect. **It bypasses your `uncaughtException` handler entirely**, so no graceful flush: use it for an investigation, not as the default.
- **Diagnostic reports.** `--report-uncaught-exception` writes a JSON report when an exception kills the process, and `--report-on-fatalerror` does it for V8 fatal errors such as running out of heap ([L27](../07-production/27-memory-and-gc.md)). `process.report.writeReport()` writes one on demand. A report contains the JavaScript and native stacks, heap statistics, active libuv handles (what was keeping the loop alive), resource usage, loaded shared libraries **and the environment variables**. Direct them with `--report-directory`, and treat the files as secrets.

---

## 10. Production rules

| Rule | Why |
|---|---|
| **Classify every failure as expected (operational) or a bug (programmer)** | They need opposite treatment: handling vs crashing |
| **Branch on `err.code`, walking `cause`; never on `message` or `errno`** | Messages change and embed data; `errno` differs per platform; wrappers like `fetch` hide the code one level down |
| **Exit after `uncaughtException` and unhandled rejections** | The handler gets the error, not the state; invariants may be half-applied |
| **Fatal handler: log, stop accepting, flush, exit, with an unref'd hard timeout** | A broken process must die even if flushing hangs |
| **Never install an `unhandledRejection` handler that only logs** | It disables the default crash and hides every floating promise |
| **Every emitter you own gets an `'error'` listener, or goes through `pipeline`/`once`** | An unheard `'error'` throws from a libuv callback and kills the process |
| **Throw `Error` subclasses with `code`, `status`, `retryable`, `cause`; never strings** | Strings have no stack and no code |
| **Declare error-class fields explicitly; no parameter properties** | Parameter properties aren't erasable, so native type stripping rejects them |
| **Give every unit of work a boundary (request, job, message)** | Bugs fail one unit; only what escapes reaches the process guard |
| **Make crashes cheap: transactions, idempotency keys, durable jobs, a supervisor, replicas** | Crash-only only works when restarting loses nothing that matters |
| **Dead-letter inputs that repeatedly crash a worker** | Otherwise one poison message crash-loops every replica |
| **Treat diagnostic reports as secrets** | They include environment variables |

> **Spring equivalent:** `@RestControllerAdvice` + `@ExceptionHandler` returning `ProblemDetail` is the same architecture as Express's error middleware: handlers throw typed exceptions, one place maps them to responses ([SpringBoot L16](https://github.com/Pawar-Pratik/SpringBoot/blob/main/05-web/16-api-design-dto-validation-errors.md)). The difference is what an *escaped* exception costs. In Tomcat it kills one request thread and the JVM keeps running; in Node it reaches the one thread every request shares, which is why Node's answer is to exit.

---

## 11. Interview traps

**Q1. "Operational vs programmer errors: what's the difference and what do you do with each?"**
Operational errors are expected failures of the environment (refused connections, timeouts, bad input, missing files) that correct code must handle: retry, fall back, or return the right status. Programmer errors are bugs: the code is wrong, and no handler can fix a state it doesn't understand, so you fail fast and fix the code. The weak answer classifies by error type; the strong one says the same `ENOENT` can be either, depending on whether the program expected it.
**Follow-up:** "So does every `TypeError` in an Express handler crash the server?" No: if the handler touched only request-scoped state, the framework boundary contains it and a 500 is correct. Crash when the error escapes to the process or corrupted shared state.

**Q2. "Why must you exit after `uncaughtException`?"**
Because the throw unwound the stack mid-operation, and any invariant in progress (a cache updated in two steps, a pool client checked out, a lock held, a counter incremented) may be half-applied. The handler receives the error, not the state, so it can't repair anything. Continuing means serving from a process in an undefined state. Log, flush, exit with a hard timeout, and let the supervisor start a known-good process.
**Follow-up:** "What if the handler itself throws?" Node exits with code 7, and anything you hadn't logged yet is gone.

**Q3. "What happens when an EventEmitter emits `'error'` with no listener?"**
`emit` throws the error (wrapping non-`Error` values in `ERR_UNHANDLED_ERROR`). Because sockets, streams and servers emit from libuv callbacks, that becomes an uncaught exception and the process exits. A listener on `events.errorMonitor` doesn't prevent it; only a real `'error'` listener does.

**Q4. "How should you compare errors: by message or by code?"**
By `code`. Messages change across versions and embed paths and addresses; `errno` is platform-specific (`ENOENT` is -2 on Linux and -4058 on Windows). And look beneath wrappers: `fetch` rejects with `TypeError: fetch failed` and puts the real code in `cause`, sometimes inside an `AggregateError` when several addresses were tried.

**Q5. "What is crash-only software?"**
Candea and Fox's idea: since a system must survive crashes anyway, make crashing the only shutdown and recovery the only startup, so the recovery path is constantly exercised. In practice: atomic transactions, idempotent retries, durable queues with leases, no important in-memory state, a supervisor and redundancy. Then "exit on a bug" costs a retry.
**Follow-up:** "Doesn't crashing drop in-flight requests?" Yes, on that one process. Replicas keep the service up, and idempotency keys let clients retry safely. A zombie process serving corrupted state is the worse outcome.

**Q6. "Why is the `domain` module deprecated?"**
It routed async errors to per-request handlers via implicit binding, which made "catch and keep serving" look safe. It wasn't: catching doesn't unwind half-applied state, implicit binding leaked across requests through pooled resources, and it added overhead everywhere. Its jobs were split between proper error channels, `AsyncLocalStorage` for context, and crash-only design.

**Q7. "Can you `try/catch` an error thrown inside a `setTimeout` callback?"**
Not from outside it. The callback runs later on a fresh stack whose bottom frame is the event loop, long after the `try` frame is gone, so it's an uncaught exception. Catch inside the callback, or use a promise API (`timers/promises`) and `await` it, which resumes the continuation inside the `try`.

**Q8. "Should you install an `unhandledRejection` handler?"**
Only if it exits. Installing one replaces the default crash, so a handler that just logs turns every floating promise into silently skipped work. Use `uncaughtExceptionMonitor` if you only want to observe, or a fatal handler that logs, flushes and exits.

**Q9. "Your Node process exits with code 1 and there's no log line in your log system. What happened?"**
Likely a fatal error whose log was buffered asynchronously (pino writes asynchronously by default) and never flushed because the process exited first, or a `process.exit()` called right after a write. Node's own trace went to stderr, so check the container's raw stderr. The fix is a fatal handler that flushes before exiting, under a deadline. If it's exit 137 instead, it was SIGKILL, probably the OOM killer ([L27](../07-production/27-memory-and-gc.md)).

**Q10. "Who decides whether an error is retryable?"**
The code that understands the failure sets `retryable` on the error: a connection refused before anything was sent is transient and safe; a 400 never is. The caller then combines that with whether the *operation* is idempotent, because a timeout after the request was sent may mean it succeeded. Retrying a programmer error is never right; it runs the same bug again.

**Q11. "Express 4 vs Express 5: what happens when an async handler throws?"**
In Express 5 the rejected promise is forwarded to the error middleware, so you get a clean 500. In Express 4 the router ignored the returned promise, so the rejection was unhandled and, since Node 15, crashed the process. That's why Express 4 code wrapped every async handler.

---

## 12. Build & break

### Build — `src/lib/errors.ts`
- `LedgerError`, `DomainError`, `UpstreamError` exactly as in §5, fields declared explicitly so `node` runs the file.
- `systemCode(err)` walking `cause`, and `classify(err, dependency)` that maps timeouts and system codes to `UpstreamError` and **rethrows anything unrecognised**.
- `toProblem(err, requestId)` returning RFC 9457 fields; unknown errors become a generic `internal_error` with no internals leaked.
- Tests: `classify` on a real refused `fetch` to `127.0.0.1` and to `localhost` (the `AggregateError` case) both yield `upstream_unavailable`; a plain `TypeError` is rethrown; `toProblem` on a `TypeError` contains no stack and no message.

### Build — `src/lib/process-guards.ts`
- `installProcessGuards({ log, flush, stopAccepting, timeoutMs })` from §6, called first thing in `main.ts` for the API and every worker.
- A test that spawns a child process which throws from a timer, and asserts: exit code 1, one `fatal_error` line with `origin: "uncaughtException"`, and exit within `timeoutMs` even when `flush` never resolves.

### Break — five experiments
1. **The unheard `'error'`.** `new EventEmitter().emit("error", new Error("db down"))`: the process dies with exit code 1. Emit a string instead and read `ERR_UNHANDLED_ERROR`. Add a listener on `events.errorMonitor` only: it still dies. Add a real `'error'` listener: it survives.
2. **The thrown string.** In a worker, `setTimeout(() => { throw "insufficient funds"; }, 10)`. Observe that the output is the bare string with no stack and a hint about `--trace-uncaught`. Replace it with `throw new DomainError(422, "insufficient_funds", "…")` and compare.
3. **The limping process.** Build a tiny in-memory balance cache with two maps (`available`, `pending`) and an `applyRefund` that updates `available`, then calls a formatter that throws on a missing `reason`, then updates `pending`. Install `process.on("uncaughtException", (e) => console.error(e))` and trigger the refund from a timer. **Observe the process keeps serving `GET /balance` with `available + pending` short by the refund amount: money missing, no crash, no alert.** Swap in `installProcessGuards`: the process exits, restarts, and rebuilds the cache from Postgres with correct totals.
4. **Retrying a bug forever.** Wrap a delivery in a retry loop that catches everything with exponential backoff, and make the payload builder read `event.data.customer.email` on an event without a customer. Watch the attempt counter climb and the logs fill with the same `TypeError`. Fix it with `classify()`: the `TypeError` is rethrown, the job fails once, and after N failures it goes to the dead-letter path.
5. **The report.** Run `node --report-uncaught-exception --report-directory=./reports crash.ts` with a script that throws from a timer while an HTTP server is listening. Open the JSON and find the JavaScript stack, the libuv handle for the listening TCP server, and the `environmentVariables` section. That last one is why the directory must never be world-readable.

### Explain out loud (90 seconds)
1. Operational vs programmer errors, with an example where the same code is both.
2. The four channels and what happens when each goes unheard.
3. Why you compare `code`, and why `fetch` errors need the `cause` chain.
4. Why you exit after `uncaughtException`, and the exact steps the handler takes.
5. Crash-only design: the five properties that make a crash cost a retry, and where the request-scope boundary sits.

---

## What's next

You can now tell a failure from a bug and fail correctly. Next comes the other half of reliability: stopping work nobody is waiting for any more, and capping how much work runs at once. `AbortSignal`, deadlines that follow a request, and the semaphore that keeps `Promise.all` from taking down your own dependencies.

Next → **[Lesson 10: Cancellation, timeouts & bounded concurrency](10-cancellation-and-concurrency.md)**
