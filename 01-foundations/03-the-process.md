# Lesson 03 — The process: your program's contract with the OS

> **Why this lesson exists:** a Node program is a process, and how it starts, reads configuration, writes output and exits decides whether it behaves under Docker, Kubernetes and CI. "Why won't my script exit?" and "why is my log output truncated?" are both answered here. So is running TypeScript with no build step, which is how every file in this track runs.

**Time:** ~65 minutes · **Prereq:** Lesson 01

---

## 1. The idea in one sentence

> **A Node process runs your main module once, then lives exactly as long as the event loop has referenced work, and it talks to the outside world through four channels only: argv and env in; stdout/stderr and an exit code out.**

---

## 2. Why it exists: the process is all your platform sees

Kubernetes, Docker, systemd and your CI runner never call your functions or read your source. They see a command line, environment variables, bytes on stdout and stderr, the signals they send, and an exit code. Get those wrong and the platform makes the wrong decision on your behalf:

| Symptom | Process-level cause | Section |
|---|---|---|
| A migration script finishes its work but CI times out an hour later | Something still holds the event loop open: a DB pool, an interval | 3 |
| The service "starts fine" and 500s at 2am on one endpoint | An env var is missing or malformed and is read lazily | 5 |
| The last log lines before a crash are missing | `process.exit()` raced asynchronous writes to stdout | 6 |
| A Kubernetes Job is marked successful although it failed | It exited 0 | 7 |
| Every deploy shows containers exiting 143 or 137 | SIGTERM with no handler; SIGKILL after the grace period | 7, 8 |
| `node main.ts` throws on an `enum` | Type stripping only removes erasable syntax | 9 |

---

## 3. The lifecycle, and why a script doesn't exit

```text
 node src/main.ts
   │
   ├─ 1. Bootstrap: V8 and libuv start, flags are parsed, --env-file is loaded
   ├─ 2. Run the main module synchronously, top to bottom
   │       ESM: the whole import graph is loaded and linked first (L04);
   │       top-level await may pause here
   ├─ 3. The event loop: run callbacks while anything referenced is alive
   │       handles:  long-lived  (a listening server, a socket, a timer, an interval,
   │                              a child process, stdin being read)
   │       requests: short-lived (an fs read in flight, a DNS lookup, a pending write)
   ├─ 4. Nothing referenced is left → 'beforeExit' (a listener may schedule more → back to 3)
   └─ 5. 'exit' (synchronous listeners only) → the process ends with process.exitCode ?? 0
```

Step 3 is the whole story behind "why won't my script exit". **The process doesn't end when your code reaches its last line; it ends when the loop has nothing referenced left.** The usual suspects:

| Still open | Close it with |
|---|---|
| A listening HTTP server | `server.close()` ([L24](../05-concurrency/24-graceful-shutdown.md) does it gracefully) |
| A `setInterval` (a metrics flush, a sweeper) | `clearInterval(t)`, or `t.unref()` |
| A `pg.Pool` or Redis client with open connections | `await pool.end()`, `await redis.quit()` |
| A socket or a child process | `socket.end()` / `child.kill()` |
| `process.stdin` in flowing mode | `process.stdin.pause()` |

**`.unref()`** tells libuv "don't keep the process alive just for this handle". Timers, intervals, sockets, servers and child processes all have it. It's right for housekeeping that should die with the process (a periodic metrics flush in a CLI) and wrong for work that must complete.

To find what's holding the loop open, `process.getActiveResourcesInfo()` lists the live resource types (it's marked experimental, but it's invaluable for exactly this):

```ts
const t = setInterval(() => {}, 60_000);
console.log(process.getActiveResourcesInfo());   // [ 'Timeout' ]
t.unref();
console.log(process.getActiveResourcesInfo());   // []
```

**Never "fix" a hanging script with `process.exit()`.** It hides the leak, and section 6 shows what else it throws away.

---

## 4. `process.argv` and `util.parseArgs`

`process.argv` is `[pathToNode, pathToScript, ...yourArgs]`. Node's own flags (`--watch`, `--inspect`) are in `process.execArgv`, not `argv`. Parsing by hand is how `--merchant=mrc_1` versus `--merchant mrc_1` bugs happen; `util.parseArgs` is built in and stable:

```ts
// scripts/reconcile.ts — run: node scripts/reconcile.ts mrc_1 --since 2026-10-01 --dry-run
import { parseArgs } from "node:util";

const { values, positionals } = parseArgs({
  options: {
    since: { type: "string" },
    "dry-run": { type: "boolean", default: false },
    concurrency: { type: "string", short: "c", default: "4" },
  },
  allowPositionals: true,
  strict: true,               // unknown flags throw ERR_PARSE_ARGS_UNKNOWN_OPTION
});

const merchantId = positionals[0];          // string | undefined (noUncheckedIndexedAccess)
if (!merchantId) throw new Error("usage: reconcile <merchantId> [--since YYYY-MM-DD] [--dry-run]");
const concurrency = Number(values.concurrency);
if (!Number.isInteger(concurrency) || concurrency < 1) throw new Error("--concurrency must be a positive integer");
```

Values are only strings or booleans: **numbers and dates are still your job to parse and validate**. The full Ledger CLI (`ledger listen --forward-to`) is built in [L32](../08-shipping/32-packaging-and-publishing.md).

---

## 5. `process.env`, and validating config once

**Every value in `process.env` is a string or `undefined`.** That's the source of a whole family of bugs:

```ts
if (process.env.SHED_LOAD) { … }              // ❌ "false" is a non-empty string, so truthy
const port = Number(process.env.PORT);        // ❌ "" → 0, unset → NaN
const port2 = parseInt(process.env.PORT ?? "") || 3000;   // ❌ "abc" → silently 3000
```

Loading a `.env` file is built in:

- **`node --env-file=.env main.ts`** loads the file before your code runs. **Variables already set in the real environment win over the file**, which is what you want: the deploy platform's values override the developer's file. (`--env-file-if-exists` doesn't fail when the file is missing.)
- **`process.loadEnvFile(".env")`** does the same from code.

Loading isn't validating. The rule: **parse the whole environment once, at boot, into a typed and frozen object, and crash immediately if it's wrong.** A misconfigured service that refuses to start is a two-minute fix. One that starts and fails on the first webhook three hours later is an incident. [TS L15](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/15-the-boundary-and-parsing.md) teaches zod; here's Ledger's config with the Node-specific details:

```ts
// src/config.ts
import { z } from "zod";

// z.coerce.boolean() is WRONG for env vars: Boolean("false") === true.
const flag = (def: "true" | "false") =>
  z.enum(["true", "false"]).default(def).transform((v) => v === "true");

const EnvSchema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  PORT: z.coerce.number().int().min(1).max(65_535).default(3000),   // "" → 0 → rejected
  DATABASE_URL: z.string().regex(/^postgres(ql)?:\/\//, "must be a postgres:// URL"),
  REDIS_URL: z.string().regex(/^rediss?:\/\//, "must be a redis:// URL"),
  PG_POOL_MAX: z.coerce.number().int().min(1).max(100).default(10),
  WEBHOOK_TIMEOUT_MS: z.coerce.number().int().min(100).max(30_000).default(10_000),
  WEBHOOK_SIGNING_KEY: z.string().min(32, "must be at least 32 characters"),
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),
  SHED_ON_LOOP_DELAY: flag("true"),
});

export type Config = Readonly<z.infer<typeof EnvSchema>>;

/** Takes env as a parameter so tests never have to mutate process.env. */
export function loadConfig(env: NodeJS.ProcessEnv = process.env): Config {
  const parsed = EnvSchema.safeParse(env);
  if (!parsed.success) {
    const lines = parsed.error.issues.map((i) => `  ${i.path.join(".")}: ${i.message}`);
    throw new Error(`Invalid configuration:\n${lines.join("\n")}`);
  }
  return Object.freeze(parsed.data);
}
```

Then **nothing else in the codebase reads `process.env`**; it receives `Config`. (ESLint's `n/no-process-env` rule enforces that.) Secrets in config must never reach a log line in clear text: the Build redacts them.

> **Spring equivalent:** `@ConfigurationProperties` plus `@Validated` turns a bad property into a startup failure with a precise message: [Spring L12 — Configuration, profiles & secrets](https://github.com/Pawar-Pratik/SpringBoot/blob/main/04-spring-boot/12-configuration-and-profiles.md). Same principle; Node just doesn't do it unless you write the ten lines above.

---

## 6. stdio: three streams with a platform-dependent quirk

`process.stdin`, `process.stdout` and `process.stderr` are streams ([L13](../03-data-and-io/13-streams-and-backpressure.md)). `console.log` writes to stdout, `console.error` to stderr. In a container both are collected by the runtime, and structured JSON logs go to stdout as an event stream ([L29](../07-production/29-observability.md)).

They differ from every other Node stream in one way that matters: **writes may be synchronous or asynchronous depending on the platform and on what the stream is connected to.** Files are synchronous everywhere. The case that bites: **pipes are asynchronous on POSIX** (and synchronous on Windows). In a Linux container your stdout is usually a pipe to the container runtime, so in production, writes to stdout are asynchronous.

"Asynchronous" here means: Node writes what the kernel's pipe buffer will accept (64 KiB by default on Linux) and queues the rest in memory, to be flushed over later loop turns. Two consequences:

1. **`process.exit()` straight after a big write truncates the output.** The queued bytes are in Node's memory, and `process.exit()` doesn't wait for them. Small writes usually survive because they fit in the pipe buffer, which is why the bug appears only with big outputs, or under load, or at the worst moment: the final lines before a crash, which explain the crash.
2. **Logging faster than the consumer reads grows memory.** The queue is unbounded unless you honour backpressure (`write()` returning `false`, [L13](../03-data-and-io/13-streams-and-backpressure.md)). Async log libraries buffer for the same reason, and the same `process.exit()` truncation applies to their buffers.

On Windows, pipes are synchronous, so you **won't** reproduce the truncation in Git Bash: run that experiment in WSL2.

---

## 7. Exit codes, and why `process.exitCode` usually beats `process.exit()`

| Code | Meaning |
|---|---|
| **0** | Success |
| **1** | An uncaught exception, an unhandled rejection, or you set it. The generic failure |
| **13** | Top-level `await` on a promise that never settled, and nothing else kept the loop alive. Node prints `Warning: Detected unsettled top-level await` |
| **128 + N** | Killed by signal N. **130** = SIGINT (2), **137** = SIGKILL (9), **143** = SIGTERM (15) |

Node documents a dozen more codes (9 is an invalid command-line argument); you'll rarely see the rest. The 128 + N codes come from the shell and the container runtime: the process didn't return 137, **it was killed and never got to return anything.** 137 in Kubernetes means one of three things: the kernel's OOM killer (the pod shows `OOMKilled`, [L27](../07-production/27-memory-and-gc.md)), the grace period expiring after SIGTERM ([L24](../05-concurrency/24-graceful-shutdown.md)), or someone running `kill -9`. **Exit codes are how orchestrators decide what happened**: a Kubernetes Job retries on non-zero and is marked complete on 0, so a worker that catches a fatal error and exits 0 lies to the platform.

### `process.exit()` vs `process.exitCode`

- **`process.exit(code)`** runs the `'exit'` listeners and terminates immediately. It abandons everything pending: queued stdout writes, a `COMMIT` in flight, buffered log lines, requests your server is halfway through answering.
- **`process.exitCode = code`** records the code and lets the process end naturally once the loop drains, after pending writes have flushed. Node's documentation recommends this over calling `process.exit()`.

```ts
// ❌ A CLI that prints a report and exits. The report is cut off when stdout is a pipe.
printReport(rows);
process.exit(failures > 0 ? 1 : 0);

// ✅ Same intent; output flushes first.
printReport(rows);
process.exitCode = failures > 0 ? 1 : 0;
```

`process.exit()` has legitimate uses, all of them as the *last* step, never as a substitute for closing things: the end of a graceful shutdown with a hard timeout ([L24](../05-concurrency/24-graceful-shutdown.md)), after an uncaught exception ([L09](../02-event-loop/09-errors-and-crashing.md)), and a CLI that must exit even though a dependency leaked a handle, after it has flushed its output.

### `'beforeExit'` vs `'exit'`

| | `'beforeExit'` | `'exit'` |
|---|---|---|
| **When** | The loop has run out of work | The process is about to end, on almost any path |
| **Async work allowed?** | Yes: scheduling some keeps the process alive, and `'beforeExit'` fires again later | **No**: listeners must be synchronous; anything they schedule is abandoned |
| **On `process.exit()` or an uncaught exception?** | **Not emitted** | Emitted (with code 1 after an uncaught exception) |
| **On SIGKILL?** | No | No: nothing runs, because SIGKILL can't be caught |
| **Good for** | Flushing a final batch in a short-lived script | A last synchronous log line or a `writeFileSync` of a marker |

A server never empties its loop (the listening socket keeps it alive), so `'beforeExit'` is a script-and-CLI tool, not a server one.

---

## 8. Signals, briefly

- **SIGINT** is Ctrl+C in a terminal.
- **SIGTERM** is what `docker stop` and Kubernetes send first. If the process is still alive after the grace period (10 s for `docker stop`, 30 s by default in Kubernetes), **SIGKILL** follows, and it can't be caught.

With no listener, Node exits on SIGINT and SIGTERM with 128 + the signal number (130, 143). **Adding a listener removes that default: Node no longer exits, and you own the shutdown.**

```ts
process.once("SIGTERM", () => {
  console.log("SIGTERM: stop accepting work, drain, then exit");   // L24 writes the real one
  server.close(() => { process.exitCode = 0; });
});
```

Windows differs: Ctrl+C delivers SIGINT, Ctrl+Break delivers SIGBREAK, and SIGTERM can be listened for but isn't delivered by other processes in the POSIX sense; `process.kill(pid, "SIGTERM")` on Windows terminates the target unconditionally. **Run signal experiments in WSL2.** Draining, readiness and the PID 1 trap in containers are [L24](../05-concurrency/24-graceful-shutdown.md).

---

## 9. Running TypeScript natively

`node src/main.ts` runs TypeScript directly. **Type stripping** arrived behind a flag in 22.6, has been on by default since 23.6 and 22.18, and is **stable since 25.2 and 24.12**. It replaces type syntax with whitespace, so line and column numbers in stack traces still match your source with no source maps.

| What it does | What it doesn't do |
|---|---|
| Removes annotations, interfaces, type aliases, generics, `as`, `satisfies`, `declare`, `abstract`, `private`/`readonly` modifiers, type-only namespaces | **No type checking at all.** A file full of type errors runs |
| Respects `"type"` in `package.json` for `.ts` (and `.mts`/`.cts` like `.mjs`/`.cjs`) | **Ignores `tsconfig.json`:** no `paths`, no downlevelling, no JSX |
| Erases `import type` statements entirely | **Refuses non-erasable syntax:** `enum`, parameter properties, namespaces with values throw `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX` (unless you opt into `--experimental-transform-types`) |
| | **Refuses files under `node_modules`** (`ERR_UNSUPPORTED_NODE_MODULES_TYPE_STRIPPING`): published packages must ship JavaScript |
| | **No extension guessing:** `import "./db"` fails with `ERR_MODULE_NOT_FOUND`; write `./db.ts` |

Two consequences you'll hit in week one. First, `enum` is out; use a const object and a derived union:

```ts
// ❌ enum PaymentStatus { Pending = "pending", Succeeded = "succeeded" }   → ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX
export const PaymentStatus = { Pending: "pending", Succeeded: "succeeded", Failed: "failed" } as const;
export type PaymentStatus = (typeof PaymentStatus)[keyof typeof PaymentStatus];   // ✅ erasable
```

Second, **type-only imports must say so.** Node strips `import type` lines, but a plain `import { Payment } from "./types.ts"` survives stripping, and at runtime the module has no export called `Payment`: `SyntaxError: The requested module './types.ts' does not provide an export named 'Payment'`. That's why the track's tsconfig turns on `verbatimModuleSyntax`.

**Type checking is a separate step:** `npx tsc --noEmit` in CI as a required check, and `npx tsc --noEmit --watch` (or your editor) locally. The compilation model itself is [TS L01](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/01-foundations/01-why-typescript-and-erasure.md).

### The track's `tsconfig.json`, line by line

The README promised every line would be explained. Flag mechanics in general are [TS L02](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/01-foundations/02-tsconfig-deeply.md); this is why each line is right *for code that Node runs directly*.

| Line | What it does | Why it's in this config |
|---|---|---|
| `"target": "esnext"` | The syntax level tsc assumes, and the default `lib` | Nothing is ever downlevelled: Node runs exactly what you wrote. (If the `esnext` built-in types ever offer an API your Node lacks, pin `lib` to an ES year) |
| `"module": "nodenext"` | Model Node's real module system | Each file is ESM or CJS by its extension and the nearest `"type"`, exactly as Node decides ([L04](04-modules-cjs-and-esm.md)); imports need extensions |
| `"moduleResolution": "nodenext"` | Resolve specifiers the way Node does | Understands `exports`/`imports` maps and `import` vs `require` conditions. It's implied by `module: nodenext`; it's written out for readers |
| `"noEmit": true` | tsc never writes `.js` | Node runs the `.ts`; tsc is purely a checker |
| `"allowImportingTsExtensions": true` | Permits `import "./db.ts"` | That's the file Node actually loads. tsc allows this flag only with `noEmit` (or declaration-only emit) |
| `"erasableSyntaxOnly": true` | Errors on `enum`, parameter properties, namespaces with code, `import x = require()` (TS 5.8+) | Catches what Node refuses to strip **at check time**, not at runtime |
| `"verbatimModuleSyntax": true` | Imports and exports are kept exactly as written, so type-only imports must use `import type` | Without it, a type-only import survives stripping and crashes at runtime (above). It also flags ESM syntax in a file Node will treat as CommonJS |
| `"strict": true` | The eight strict checks | Non-negotiable ([TS L02](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/01-foundations/02-tsconfig-deeply.md)) |
| `"noUncheckedIndexedAccess": true` | Index access returns `T \| undefined` | `positionals[0]`, `rows[0]` and `process.argv[2]` genuinely may be missing |
| `"types": ["node"]` | Include only `@types/node`'s globals | Gives you `process`, `Buffer` and `NodeJS.*`; stops stray `@types/*` packages leaking globals in. Requires `npm i -D @types/node` |
| `"skipLibCheck": true` | Don't type-check `.d.ts` files in dependencies | You can't fix their types, and it makes checks much faster |

---

## 10. The everyday toolbox

| Tool | What it does | Note |
|---|---|---|
| `node --watch src/main.ts` | Restarts when an imported file changes | Stable since 22; `--watch-path` adds paths |
| `node --run dev` | Runs a `package.json` script | Skips npm's startup cost; doesn't run `pre`/`post` scripts |
| `node` (the REPL) | Interactive prompt | `.help`; `_` is the last result; top-level `await` works |
| `globalThis` | The global object | Same name as in browsers; `global` is the Node-only alias |
| `process.memoryUsage()` | `rss`, `heapTotal`, `heapUsed`, `external`, `arrayBuffers` | What each one means is [L27](../07-production/27-memory-and-gc.md) |
| `process.hrtime.bigint()` | Monotonic time in nanoseconds, as a `bigint` | For precise durations |
| `performance.now()` | Monotonic milliseconds (fractional) since process start | For durations you'll print |
| `Date.now()` | Wall-clock milliseconds | For timestamps only: NTP can move it backwards mid-measurement |

---

## 11. Production rules

| Rule | Why |
|---|---|
| **Close what you open; `.unref()` only housekeeping timers** | The process lives while referenced handles exist; leaked handles hang scripts and CI |
| **Never use `process.exit()` to fix a hang** | It hides the leaked handle and truncates pending output |
| **Set `process.exitCode`; call `process.exit()` only as the final step after flushing** | Lets stdout and log buffers drain; on POSIX pipes, writes are asynchronous |
| **Exit non-zero on every failure path** | Orchestrators and CI decide retries and success from the exit code alone |
| **Parse all env vars once, at boot, into a frozen typed object** | Env values are strings; a bad value must fail at startup, not on the first request that reads it |
| **Never use `z.coerce.boolean()` or truthiness on env flags** | `"false"` is truthy; only an explicit `"true"`/`"false"` enum is safe |
| **Only `config.ts` reads `process.env`** | One place to validate, document and redact |
| **Never log secrets; redact config before printing it** | Logs are copied to many systems with weaker access controls |
| **Erasable TypeScript only: no `enum`, no parameter properties** | Node refuses to strip them; `erasableSyntaxOnly` catches them at check time |
| **`import type` for type-only imports** | A surviving type-only import crashes at runtime |
| **`tsc --noEmit` is a required CI check** | Node runs code with type errors in it |
| **Measure durations with `performance.now()` or `hrtime`, never `Date.now()`** | Wall-clock time can jump |

---

## 12. Interview traps

**Q1. "Why doesn't my Node script exit?"**
Because the process exits only when the event loop has no referenced handles or requests left, not when your code reaches its last line. Something is still open: a server, an interval, a socket, a database pool. Find it with `process.getActiveResourcesInfo()`, then close it, or `unref()` it if it's housekeeping. The weak answer is "add `process.exit()`", which hides the leak and truncates output.

**Q2. "`process.exit()` vs `process.exitCode`?"**
`process.exit()` terminates immediately after the synchronous `'exit'` listeners, abandoning pending async work: queued stdout writes, in-flight DB commits, buffered logs. `process.exitCode` sets the code and lets the process end naturally once that work drains. Use `exitCode` by default; `exit()` only as the last step of a shutdown that has already flushed.
**Follow-up:** "Why can stdout lose data at all?" Because writes to pipes are asynchronous on POSIX: what doesn't fit in the pipe buffer is queued in Node's memory.

**Q3. "What does exit code 137 mean?"**
128 + 9: the process was killed with SIGKILL and never ran another instruction. In Kubernetes it's usually the OOM killer (`OOMKilled` in the pod status) or the grace period expiring after SIGTERM; otherwise a manual `kill -9`. 143 is SIGTERM with no handler. Saying "the app crashed with 137" is the weak answer: the app didn't exit, it was killed.

**Q4. "Can Node run TypeScript? What doesn't work?"**
Yes: type stripping is on by default and stable since 24.12. It replaces types with whitespace and does no type checking. It ignores `tsconfig` (so no `paths`), refuses non-erasable syntax like `enum` and parameter properties, refuses files under `node_modules`, needs explicit `.ts` extensions, and requires `import type` for type-only imports. Type checking is a separate `tsc --noEmit` step.

**Q5. "How do you validate config?"**
Parse `process.env` once at boot with a schema (zod), coercing strings to numbers explicitly and treating booleans as an explicit `"true"`/`"false"` enum. Freeze the result, export it as the only source of config, and crash with a message naming every bad variable. Take the env as a parameter so tests don't mutate `process.env`.

**Q6. "Why is my piped output truncated?"**
On Linux and macOS, writes to a pipe are asynchronous: bytes beyond the kernel's buffer queue in Node's memory and flush over later loop turns. If the code calls `process.exit()` right after a large write, the queue is discarded. Set `process.exitCode` instead. You won't reproduce it on Windows, where pipe writes are synchronous.

**Q7. "`'exit'` vs `'beforeExit'`?"**
`'beforeExit'` fires when the loop runs out of work; listeners can schedule more, which keeps the process alive. It's not emitted on `process.exit()` or after an uncaught exception. `'exit'` fires on almost every path, but its listeners must be synchronous, because the process ends right after them. Neither runs on SIGKILL.

**Q8. "Every deploy, our containers exit 143. Problem?"**
It means SIGTERM arrived and nothing handled it, so Node exited with the default. Any in-flight requests were dropped. It's the signature of a service with no graceful shutdown: install a handler that stops accepting work, drains, closes pools and then exits ([L24](../05-concurrency/24-graceful-shutdown.md)).

**Q9. "What's wrong with `if (process.env.FEATURE_X)`?"**
Env values are strings, and every non-empty string is truthy, so `FEATURE_X=false` enables the feature. Parse flags explicitly at boot into booleans.

**Q10. "Node strips types anyway. Why do you need `import type` and `verbatimModuleSyntax`?"**
Because stripping is syntactic. Node erases `import type` lines, but a plain import of a name that's only a type survives, and at runtime the ESM loader fails with "does not provide an export named …". `verbatimModuleSyntax` makes tsc flag those imports at check time.

---

## 13. Build & break

### Build — the Ledger Engine skeleton

```text
ledger-engine/
├── .nvmrc              24
├── .env.example        every variable, with fake values, committed
├── package.json
├── tsconfig.json       the track's, copied from the README
└── src/
    ├── config.ts       section 5
    ├── lib/redact.ts
    └── main.ts
```

`package.json`:

```jsonc
{
  "name": "ledger-engine",
  "private": true,
  "type": "module",                       // every .ts file is ESM (L04)
  "engines": { "node": ">=24.12" },       // type stripping is stable from here
  "scripts": {
    "start": "node src/main.ts",
    "dev": "node --watch --env-file=.env src/main.ts",
    "typecheck": "tsc --noEmit"
  }
}
```

Then `npm i zod` and `npm i -D typescript @types/node` (TypeScript 5.8 or newer, for `erasableSyntaxOnly`).

`src/lib/redact.ts` prints config without secrets:

```ts
import type { Config } from "../config.ts";

export function redact(c: Config): Record<string, unknown> {
  const mask = (url: string) => {
    const u = new URL(url);
    if (u.password) u.password = "***";
    return u.toString();
  };
  return { ...c, DATABASE_URL: mask(c.DATABASE_URL), REDIS_URL: mask(c.REDIS_URL), WEBHOOK_SIGNING_KEY: "***" };
}
```

`src/main.ts` boots, logs the redacted config, and exits correctly on failure:

```ts
import { createServer } from "node:http";
import { loadConfig } from "./config.ts";
import { redact } from "./lib/redact.ts";

async function main(): Promise<void> {
  const config = loadConfig();                       // throws before anything is opened
  console.log(JSON.stringify({ msg: "config loaded", config: redact(config) }));

  const server = createServer((_req, res) => res.end("ok\n"));   // L17 builds the real one
  await new Promise<void>((resolve) => server.listen(config.PORT, resolve));
  console.log(JSON.stringify({ msg: "listening", port: config.PORT }));
}

main().catch((err: unknown) => {
  console.error(err instanceof Error ? err.message : err);
  process.exitCode = 1;            // ✅ nothing is open yet, so the process ends and stderr flushes
});
```

That `catch` is only correct because a boot failure happens before any handle is opened. If boot can fail after the database pool is connected, close what's open first, or the process won't exit at all. That's part of [L24](../05-concurrency/24-graceful-shutdown.md).

Verify: `npm run typecheck` passes; `node --run start` with no env prints every missing variable and `echo $?` shows `1`; with a valid `.env`, `npm run dev` restarts on save.

### Break — five experiments

1. **A script that never exits.** In a one-off script, start the sweeper from [L02](02-the-javascript-node-assumes.md) (`setInterval(sweep, 60_000)`), run one sweep, and log "done". The process hangs. Print `process.getActiveResourcesInfo()`: `[ 'Timeout' ]`. Fix with `clearInterval` after the work (or `.unref()` if the timer is genuinely housekeeping) and watch it exit by itself.

2. **Truncated output (run in WSL2).**
   ```ts
   // big.ts
   const line = "x".repeat(99) + "\n";
   process.stdout.write(line.repeat(100_000));   // 10 MB
   process.exit(0);                               // ❌
   ```
   `node big.ts | wc -c` prints **less than 10000000**; `node big.ts > out.txt && wc -c out.txt` prints exactly 10000000, because files are written synchronously. Replace `process.exit(0)` with `process.exitCode = 0` and the pipe gets every byte. In Git Bash on Windows, both print 10000000 because pipe writes are synchronous there.

3. **A config error that surfaces late.** Without validation, read config where it's used: `server.listen(Number(process.env.PORT))` at the end of a simulated 3-second warm-up, and `AbortSignal.timeout(Number(process.env.WEBHOOK_TIMEOUT_MS))` inside a "send webhook" function called after 10 seconds. Run with `PORT=abc`: the crash comes after the warm-up, as `ERR_SOCKET_BAD_PORT … Received type number (NaN)`, and never names the env var. Fix `PORT` and set `WEBHOOK_TIMEOUT_MS=abc`: now it runs fine until the first webhook, then `ERR_OUT_OF_RANGE`. Switch to `loadConfig()` and both fail in the first millisecond, naming the variable.

4. **An enum, run natively.** Put `enum PaymentStatus { Pending, Succeeded }` in a `.ts` file and run it. Read the whole error: `SyntaxError [ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX]: TypeScript enum is not supported in strip-only mode`. Run `npx tsc --noEmit`: with `erasableSyntaxOnly`, tsc reports `error TS1294: This syntax is not allowed when 'erasableSyntaxOnly' is enabled.` at check time, before anything runs. Replace it with the `as const` object from section 9.

5. **Exit code 13.**
   ```ts
   // ready.ts
   import { EventEmitter, once } from "node:events";
   const bus = new EventEmitter();
   await once(bus, "ready");          // nothing will ever emit "ready", and nothing keeps the loop alive
   console.log("never printed");
   ```
   Run it, then `echo $?`: **13**, with `Warning: Detected unsettled top-level await`. Now add `setInterval(() => {}, 1000)` above the `await` and run it again: no exit code at all, the process hangs forever. Same bug, two symptoms, depending on whether something keeps the loop alive.

### Explain out loud (60–120 seconds)

1. The five lifecycle steps, and handles versus requests.
2. Why a script doesn't exit, how you find out what's holding it, and what `.unref()` means.
3. Why env vars need parsing, the boolean trap, and the fail-fast config pattern.
4. Why stdout can lose data, which platforms, and the fix.
5. Exit codes 0, 1, 13, 130, 137 and 143, and `exitCode` versus `exit()`.
6. What type stripping does and doesn't do, and four lines of the tsconfig with their reasons.

---

## What's next

Every file you just wrote was an ES module because of one line in `package.json`. Next: what that line actually switches on. Two module systems with two loading algorithms, live bindings, `require()` of ES modules, and the dual-package hazard that gives you two copies of the same class.

Next → **[Lesson 04: Modules: CommonJS vs ESM, properly](04-modules-cjs-and-esm.md)**
