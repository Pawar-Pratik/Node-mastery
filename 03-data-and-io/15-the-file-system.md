# Lesson 15 — The file system, honestly

> **Why this lesson exists:** file code that works on a laptop corrupts data when the process is killed mid-write, runs out of descriptors under load, and hands an attacker `/etc/passwd` through a `../` in a filename. You develop on Windows, which adds locked files, a different path syntax and different limits. This lesson is the short list of rules that make file code survive crashes, load and hostile input, and the reasons behind each rule, which is what the interview question "how do you write a file atomically?" is really testing.

**Time:** ~65 minutes · **Prereq:** Lessons 07, 13

---

## 1. The idea in one sentence

> **Treat the file system as a shared database with no transactions: write to a temp file and rename it into place, treat every user-supplied path as hostile, and treat every open descriptor as a finite resource you must give back.**

---

## 2. Why "just call `writeFile`" fails

Three incidents, all from code that passed its tests:

1. **The truncated statement.** Ledger's exporter writes `stmt_2026_10.csv.gz` with `fs.writeFile(target, data)`. A deploy sends SIGTERM, the grace period expires, SIGKILL lands mid-write, and the file on disk is now 40% of a gzip stream. The next upload job ships it to the merchant. `writeFile` opens the target with `O_TRUNC` first, so **the old good file was destroyed before the new one existed.**
2. **`EMFILE` at 2 a.m.** A reconciliation job does `Promise.all(files.map((f) => readFile(f)))` over 20,000 settlement files. Every open happens at once, the process hits its descriptor limit, and every later `open()`, including the API's sockets on Linux, fails with `EMFILE: too many open files`.
3. **The traversal.** `GET /v1/statements/:name` does `res.sendFile(path.join(STATEMENTS_DIR, req.params.name))`. Express decodes `%2F` in the parameter, so `..%2F..%2F..%2Fetc%2Fpasswd` becomes `../../../etc/passwd`, and `path.join` happily walks out of the directory.

None of these is a Node bug. They're the semantics of the file system meeting code that assumed it was simpler.

---

## 3. Three API styles, and when each is right

```ts
import { readFile } from "node:fs";                            // callback
import { readFile as readFileP } from "node:fs/promises";      // promise
import { readFileSync } from "node:fs";                        // synchronous
```

| Style | Runs on | Use it for |
|---|---|---|
| **Promise** (`node:fs/promises`) | The libuv thread pool ([L07](../02-event-loop/07-the-thread-pool.md)) | **The default** in application code |
| **Callback** (`node:fs`) | The thread pool | Hot paths where allocation matters. The fs docs say the callback API is preferable when maximal performance (time and memory) is required |
| **Sync** (`readFileSync`, …) | **The event loop itself** | Startup (config, TLS certs, migrations list), CLIs, build scripts, tests. Never in a request path |

Two consequences of "runs on the thread pool": **async file I/O competes with DNS lookups, async crypto and zlib** for the pool's four threads, and a burst of large reads can make unrelated `dns.lookup` calls wait. And **sync calls block every request in the process** for the duration of a disk read, which on a cold network volume can be tens of milliseconds ([L08](../02-event-loop/08-blocking-the-loop.md)). `readFileSync` at boot is fine because nothing is being served yet.

---

## 4. Descriptors, `FileHandle`, and `readFile` vs streams

`fs.promises.open()` returns a `FileHandle`, a wrapper around an OS file descriptor. **You must close it.** If one is garbage-collected while still open, Node tries to close it and emits a warning, and the docs say plainly not to rely on that: it can be unreliable and the file may not be closed.

```ts
import { open } from "node:fs/promises";

// ✅ Classic: close in finally, on every path.
const fh = await open(path, "r");
try {
  const { size } = await fh.stat();
  // …
} finally {
  await fh.close();
}

// ✅ Node 24: FileHandle implements Symbol.asyncDispose, and V8 supports `await using`.
{
  await using fh2 = await open(path, "r");
  const { size } = await fh2.stat();
}   // fh2.close() runs here, even if the block throws
```

**`readFile` vs streams** is the [Lesson 13](13-streams-and-backpressure.md) decision applied to disk:

| | `readFile` | `createReadStream` / `for await` |
|---|---|---|
| Memory | The whole file, in one Buffer | Chunks of 64 KiB (its own default) |
| Size limit | Files over 2 GiB fail with `ERR_FS_FILE_TOO_LARGE` | None |
| Good for | Config, templates, small JSON (kilobytes to a few MB) | Uploads, exports, logs, anything a user controls the size of |

Streams also release their descriptor automatically: `autoClose` (default `true`) closes it on end or error, and `pipeline` destroys the stream on failure.

---

## 5. Atomic writes: temp file, `fsync`, rename

What `writeFile(target, data)` does, step by step, and where a crash leaves you:

```text
  open(target, O_WRONLY | O_CREAT | O_TRUNC)   ← old contents gone HERE
  write(...)  write(...)  write(...)           ← kill -9 here: a truncated file
  close()
```

The safe sequence writes somewhere else and swaps:

```text
  open(".stmt.csv.gz.<random>.tmp", "wx")      same directory as the target
  write(...) ...                               a crash here leaves a stray temp file; the target is untouched
  fsync(tmp)                                   the data is on disk BEFORE the name points at it
  close(tmp)
  rename(tmp, "stmt.csv.gz")                   atomic: readers see the old file or the new one, never half
  fsync(directory)                             (POSIX) make the rename itself durable
```

Each step has a reason:

- **`rename` is the atomic step.** On POSIX, `rename(2)` within one filesystem atomically replaces the target: a concurrent reader opens either the complete old file or the complete new one. A reader that already had the old file open keeps reading the old contents.
- **Same directory, not `os.tmpdir()`.** `rename` across filesystems fails with `EXDEV`, and `/tmp` is often a different filesystem (tmpfs, or a separate volume in a container).
- **`fsync` before `rename` is about power loss, not process crashes.** If only the *process* dies, the kernel's page cache survives, and temp-plus-rename alone is enough. If the *machine* loses power, the rename can reach disk before the data does, leaving a correctly named file full of zeros or nothing. That's the classic "zero-length files after a crash" complaint about delayed allocation on ext4. `fsync(tmp)` forces the data out first.
- **`fsync` of the directory** persists the directory entry change, which is what makes the rename itself survive power loss on POSIX. On Windows, opening the directory works but syncing it fails with `EPERM`, so skip that step there.

```ts
// src/lib/fs-safe.ts
import { open, rename, rm } from "node:fs/promises";
import { randomBytes } from "node:crypto";
import path from "node:path";
import { setTimeout as sleep } from "node:timers/promises";

const isWindows = process.platform === "win32";

export async function writeFileAtomic(target: string, data: string | Uint8Array): Promise<void> {
  const dir = path.dirname(target);
  const tmp = path.join(dir, `.${path.basename(target)}.${randomBytes(6).toString("hex")}.tmp`);
  const fh = await open(tmp, "wx");                       // same directory → same filesystem
  try {
    await fh.writeFile(data);
    await fh.sync();                                      // data on disk BEFORE the rename
  } catch (err) {
    await fh.close();
    await rm(tmp, { force: true });
    throw err;
  }
  await fh.close();
  try {
    await renameWithRetry(tmp, target);                   // the atomic step
  } catch (err) {
    await rm(tmp, { force: true });
    throw err;
  }
  if (!isWindows) {                                       // make the rename itself durable
    const d = await open(dir, "r");
    try { await d.sync(); } finally { await d.close(); }
  }
}

async function renameWithRetry(from: string, to: string): Promise<void> {
  for (let attempt = 0; ; attempt++) {
    try { return await rename(from, to); }
    catch (err) {
      const code = (err as NodeJS.ErrnoException).code;
      const transient = isWindows && (code === "EPERM" || code === "EBUSY" || code === "EACCES");
      if (!transient || attempt === 8) throw err;         // ~2.5 s of backoff in total
      await sleep(10 * 2 ** attempt);
    }
  }
}
```

**The Windows retry exists because Windows rename isn't POSIX rename.** If any process has the target open (an antivirus scanner, the search indexer, a backup agent, or your own download handler streaming the old statement), replacing it fails with `EPERM`. Tested on Node 24 on Windows, a rename over a file that the same process merely has open for reading fails with `EPERM`, while deleting that open file succeeds. That's why `graceful-fs` and similar libraries retry renames on Windows. **The deeper fix is to avoid replacing files people read:** give each statement an immutable, versioned name (`stmt_<ulid>.csv.gz`) and point the database row at the new one.

---

## 6. File descriptors and `EMFILE`

Every open file is a descriptor, and on Linux so is every socket, pipe and `fs.watch` handle. The per-process limit is `ulimit -n`: the soft limit is often 1024 in a Linux shell, and containers inherit whatever the runtime sets, so check it rather than assume it. On Windows, Node's file descriptors come from the C runtime's table: in a test on Node 24, opening one file 20,000 times concurrently succeeded about 8,190 times and the rest failed with `EMFILE`.

`EMFILE` is almost always **unbounded concurrency**, not a limit set too low:

```ts
// ❌ 20,000 simultaneous opens. Also blocks the thread pool behind 20,000 queued reads.
const contents = await Promise.all(files.map((f) => readFile(f)));

// ✅ Bounded: at most 32 descriptors at once, from L10's mapLimit.
const contents = await mapLimit(files, 32, (f) => readFile(f));
```

The full fix list, in order:

1. **Bound concurrency** ([L10](../02-event-loop/10-cancellation-and-concurrency.md)). 32 parallel reads saturate the 4-thread pool anyway.
2. **Close on every path**: `finally`, `await using`, or streams with `pipeline`.
3. **Find leaks**: on Linux, `ls /proc/<pid>/fd | wc -l` growing over time means something isn't closed (run this in WSL2).
4. **Only then raise the limit**, as a deliberate capacity decision. Raising it first hides the leak until it's bigger.

`graceful-fs` queues opens that fail with `EMFILE` and retries them later. It's a band-aid that turns a crash into latency; bounded concurrency is the fix.

---

## 7. Watching files: `fs.watch` is a hint, not a feed

`fs.watch` uses the OS's notification API (inotify on Linux, FSEvents on macOS, `ReadDirectoryChangesW` on Windows), and inherits each one's quirks:

- **Only two event types**, `'rename'` and `'change'`, and the `filename` argument isn't guaranteed on every platform.
- **One save produces several events**, and several changes can arrive as one.
- **Editors save by writing a temp file and renaming it over the original.** On Linux, a watch on the *file* is attached to its inode, so after the rename you're watching a deleted file and later edits are silent. Watch the *directory* and filter by name.
- **Network file systems and some container bind mounts** don't deliver events at all, which is common with Docker Desktop mounts from a Windows or macOS host.
- `recursive` support has varied by platform and Node version.

`fs.watchFile` polls with `stat` instead (default interval 5,007 ms). It's portable and slow, which is acceptable for a config file.

**The rule: treat an event as "something may have changed".** Debounce, then re-read or re-scan to find out what actually changed. For real tooling, use `chokidar`, which normalises much of this. For Ledger, nothing important is triggered by watching a directory: files arrive through the upload API ([L14](14-composing-streams.md)) or a queue, both of which say exactly what arrived.

---

## 8. `path`: join, resolve, and two operating systems

| Call | Result (POSIX) | Behaviour |
|---|---|---|
| `path.join("/srv/st", "a/b.csv")` | `/srv/st/a/b.csv` | Concatenates segments, then normalises `.` and `..` |
| `path.join("/srv/st", "../../etc/passwd")` | `/etc/passwd` | **`..` walks out of the base** |
| `path.join("/srv/st", "/etc/passwd")` | `/srv/st/etc/passwd` | An absolute segment is just another segment |
| `path.resolve("/srv/st", "/etc/passwd")` | `/etc/passwd` | Processes right to left; **an absolute segment discards everything before it** |
| `path.resolve("rel/x")` | `<cwd>/rel/x` | Always returns an absolute path |

So `join` and `resolve` fail differently on hostile input, and neither one is a security check.

**`path` is platform-specific.** `path` is `path.posix` on Linux and `path.win32` on Windows. Use `path.posix` explicitly for things that are always forward-slashed (URLs, S3 keys, zip entry names), and the platform default for real files. Paths that differ in ways that bite:

- **Separators.** Windows accepts both `\` and `/`. On POSIX, `\` is an ordinary filename character, so `..\..\x` is a traversal on Windows and a strange filename on Linux. Test path code on both.
- **Case.** NTFS is case-insensitive by default: `C:\Srv\St` and `c:\srv\st` are the same directory. `path.win32.relative` compares case-insensitively; a hand-written `startsWith` check doesn't.
- **Drives.** `path.win32.relative("C:\\srv\\st", "D:\\x")` returns `D:\x`, an absolute path, because there's no relative route between drives.
- **Reserved names and oddities.** `CON`, `NUL`, `COM1` and friends are devices, trailing dots and spaces are stripped, and `name:stream` addresses an NTFS alternate data stream. Server-generated names avoid all of it.
- **Long paths.** The classic 260-character `MAX_PATH` still trips tools in your chain. Node's own fs calls generally cope, but Git needs `core.longpaths` for deep trees, for example.

In ESM there's no `__dirname`. Use `import.meta.dirname` (and `import.meta.filename`) to locate files next to your module, and never resolve data files relative to `process.cwd()`, which is wherever the process was launched from ([L04](../01-foundations/04-modules-cjs-and-esm.md)).

---

## 9. Path traversal: contain every user-supplied path

The payloads an attacker tries, and what stops each:

| Payload | Why naive code falls for it |
|---|---|
| `../../etc/passwd` | `join` normalises `..` out of the base |
| `..%2F..%2Fetc%2Fpasswd` | Express decodes route parameters before your handler sees them |
| `/etc/passwd`, `C:\Windows\win.ini` | `resolve` lets an absolute path replace the base |
| `..\..\x` | A traversal on Windows only |
| `stmt.csv\0.png` | Node rejects any path containing a NUL byte with `ERR_INVALID_ARG_VALUE`, so this classic is dead in Node, but log it as an attack |
| `../statements-evil/x` | Defeats a `target.startsWith(base)` check when a sibling directory shares the prefix |
| A symlink inside the base pointing outside it | Every lexical check passes; the OS follows the link |

The lexical check: resolve, then prove containment with `path.relative`:

```ts
// src/lib/fs-safe.ts (continued)
import { realpath } from "node:fs/promises";

export class PathEscapeError extends Error { readonly code = "PATH_ESCAPE"; }

function isInside(root: string, target: string): boolean {
  const rel = path.relative(root, target);
  return rel !== "" && rel !== ".." && !rel.startsWith(".." + path.sep) && !path.isAbsolute(rel);
}

export function safeJoin(base: string, userPath: string): string {
  const root = path.resolve(base);
  const target = path.resolve(root, userPath);              // an absolute userPath wins here…
  if (!isInside(root, target)) throw new PathEscapeError(`path escapes ${root}`);   // …and dies here
  return target;
}

export async function safeRealJoin(base: string, userPath: string): Promise<string> {
  const root = await realpath(base);
  const target = await realpath(safeJoin(root, userPath)); // resolves symlinks; ENOENT if missing
  if (!isInside(root, target)) throw new PathEscapeError("symlink escapes the base directory");
  return target;
}
```

Why each clause: `rel === ""` is the base directory itself; `".."` and `"../…"` are outside; `path.isAbsolute(rel)` catches a different Windows drive; and checking `".." + path.sep` rather than `".."` lets a legitimate file named `..foo` through. Because it uses `path.relative`, the check is case-insensitive on Windows. Tested on Node 24 on Windows, `safeJoin` rejects every lexical payload in the table and allows `sub/ok.csv` and `..foo`. The NUL byte is rejected by `fs` itself, and the symlink needs the `realpath` step.

**`realpath` closes the symlink hole, but not perfectly.** Between your check and your `open`, someone with write access to the directory could swap a file for a symlink (a time-of-check to time-of-use race). That matters only if untrusted parties can write into the directory.

**The best fix uses no user-supplied path at all.** Ledger's download endpoint is `GET /v1/statements/:id`: it looks up the statement row by ID (with RLS proving the merchant owns it), and the row holds a server-generated name like `stmt_01J9X….csv.gz`. `safeJoin` stays as defence in depth. [API L16](https://github.com/Pawar-Pratik/API-mastery/blob/main/03-security/16-owasp-and-hardening.md) has the policy-level rule; this is the mechanism under it.

---

## 10. Temp directories, cleanup, and the rest of the toolkit

```ts
import { mkdtemp, rm } from "node:fs/promises";
import { tmpdir } from "node:os";

export async function withTempDir<T>(prefix: string, fn: (dir: string) => Promise<T>): Promise<T> {
  const dir = await mkdtemp(path.join(tmpdir(), `${prefix}-`));   // appends 6 random characters
  try {
    return await fn(dir);
  } finally {
    await rm(dir, { recursive: true, force: true, maxRetries: 3 });  // retries help with Windows locks
  }
}
```

`finally` covers errors, aborts and early returns. **It doesn't cover SIGKILL, OOMKill or power loss.** Crash-only design ([L09](../02-event-loop/09-errors-and-crashing.md)) means cleanup can't depend on a clean exit: give temp directories a recognisable prefix and **sweep stale ones at boot** (anything with your prefix older than, say, a day). Node 24.4 added `fsPromises.mkdtempDisposable()`, which pairs with `await using`, but the boot-time sweep is still needed.

The rest of the toolkit, stable in Node 24:

| API | Note |
|---|---|
| `fs.rm(p, { recursive: true, force: true })` | `rm -rf`. `force` ignores a missing path. Also takes `maxRetries` for transient `EBUSY`/`EPERM` |
| `fs.cp(src, dst, { recursive: true })` | Recursive copy (no longer experimental since Node 22) |
| `fs.glob(pattern)` / `fsPromises.glob` | Built-in globbing; marked stable in Node 24.0 |
| `fs.mkdir(p, { recursive: true })` | No error if the directory already exists |
| `fs.copyFile(src, dst, constants.COPYFILE_EXCL)` | Fails instead of overwriting |
| `open(p, "wx")` | Create exclusively: fails with `EEXIST`, which makes it a cheap lock or a guard against two writers |

---

## 11. Ledger: files in the export and ingest paths

**The async statement export** (the large-merchant path from [L13](13-streams-and-backpressure.md)) writes to disk before uploading to object storage. The stream ends in a temp file in the statements directory, gets `fsync`ed, and is renamed to its final, immutable name, and only then does the job insert the `statements` row that makes it downloadable. A crash at any point leaves either nothing visible or a complete file. The boot sweep collects the orphaned temp files.

**The payout ingester** ([L14](14-composing-streams.md)) writes `rejects.csv` inside `withTempDir("ledger-ingest", …)` with `{ flags: "wx" }`. On success, the file is moved into place for the merchant to download (it's in a different directory, so it's a copy or an upload, not a `rename`, because tmp may be another filesystem). On failure or abort, `finally` deletes the directory, and the boot sweep handles the SIGKILL case.

**The download path** never touches user input: `GET /v1/statements/:id` → an RLS-scoped row lookup → `safeRealJoin(STATEMENTS_DIR, row.file_name)` → `pipeline(createReadStream(p), res)`.

> **Spring equivalent:** Java's `Files.move(tmp, target, StandardCopyOption.ATOMIC_MOVE)` after `FileChannel.force(true)` is the same temp-fsync-rename recipe, and `Path.normalize()` plus `startsWith(base)` (on `Path` objects, which compare by segment, unlike Node's string `startsWith`) is the JVM's containment check. The OS semantics underneath, including Windows' locked-file rename, are identical.

---

## 12. Production rules

| Rule | Why |
|---|---|
| **Never `writeFile` straight onto a file that matters** | It truncates first; a crash mid-write destroys the old and the new |
| **Write temp → `fsync` → `rename` in the same directory** | `rename` is atomic within one filesystem; `fsync` orders data before the name |
| **`fsync` the directory after the rename on POSIX; skip it on Windows** | Makes the rename durable on POSIX; Windows refuses with `EPERM` |
| **Prefer immutable, versioned filenames over replacing files** | Windows can't replace a file that anyone has open |
| **Retry `EPERM`/`EBUSY`/`EACCES` on Windows renames and deletes, briefly** | Antivirus and indexers hold files open for short periods |
| **Bound fs concurrency; never `Promise.all` over a file list** | `EMFILE`, plus a thread pool starved for DNS and crypto |
| **Close every `FileHandle` in `finally` or with `await using`** | GC-time closing is a warning, not a guarantee |
| **`readFile` only for small files; stream the rest** | Whole-file memory, and a 2 GiB hard limit |
| **Sync fs APIs only at startup and in CLIs** | They block the event loop for every request |
| **Never build a path from user input; look up server-generated names by ID** | Removes path traversal as a class |
| **When you must, `safeJoin` with `path.relative`, plus `realpath` for symlinks** | `startsWith` and `join` alone are bypassable |
| **Treat `fs.watch` events as hints; debounce and re-scan** | Events are duplicated, coalesced, platform-specific, or missing |
| **Sweep stale temp files at boot** | `finally` doesn't run on SIGKILL or OOMKill |

---

## 13. Interview traps

**Q1. "How do you write a file atomically?"**
Write the full contents to a temp file in the same directory, `fsync` it, close it, and `rename` it over the target. `rename` within one filesystem is atomic on POSIX, so readers see the old file or the new one, never a half-written one. Then `fsync` the directory so the rename survives power loss. The weak answer is "use `writeFile`", which truncates first. The common half-answer forgets that the temp file must be on the same filesystem (`EXDEV`), or that `fsync` is what protects against power loss rather than process crashes.
**Follow-up:** "Does that work on Windows?" Mostly. The rename replaces the target, but fails with `EPERM` if any process has the target open, so you retry briefly, or better, write immutable versioned names and never replace a file someone might be reading.

**Q2. "What is `EMFILE`, and how do you fix it?"**
The process ran out of file descriptors: every open file, socket and pipe uses one, and the limit is `ulimit -n` on Linux. It's nearly always unbounded concurrency (`Promise.all` over thousands of files) or a leak (handles not closed on an error path). Fix the cause: bound concurrency, close in `finally`, use streams with `pipeline`. Raise the limit only afterwards, deliberately. Raising it first just delays the same failure at a bigger number.

**Q3. "How do you prevent path traversal?"**
Best: don't put user input in paths. Look up a server-generated filename by ID, with authorization on the lookup. When you must accept a name, resolve it against the base and check containment with `path.relative`: reject an empty result, `..` or anything starting with `../`, and an absolute result (a different drive on Windows). Then `realpath` it and check again, so a symlink can't point outside. `startsWith(base)` is the classic wrong answer: `/srv/statements-evil` starts with `/srv/statements`.

**Q4. "`path.join` vs `path.resolve`?"**
`join` concatenates segments and normalises, so an absolute segment is just another piece, and `..` can still climb out. `resolve` works right to left until it has an absolute path, prepending the working directory if needed, so an absolute segment discards everything before it. On hostile input, `join` lets `../` escape and `resolve` lets `/etc/passwd` replace the base. Neither is a security control; containment checking is.

**Q5. "Is `fs.watch` reliable?"**
No. It wraps inotify, FSEvents or `ReadDirectoryChangesW`, so behaviour differs by platform: events are duplicated or coalesced, the filename isn't guaranteed, editors' rename-on-save breaks a watch on a single file under inotify, and network mounts and some container bind mounts deliver nothing. Treat events as hints, debounce, then re-scan, or use `chokidar`. Don't build a data pipeline on it.

**Q6. "When is `readFileSync` fine?"**
When nothing else is waiting on the event loop: at startup (reading config, TLS certificates, a migrations list), in CLIs and build scripts, and in tests. In a request handler it blocks every concurrent request for the length of the read, which on a slow or network-attached disk can be long. The async version runs on the thread pool instead.

**Q7. "Why might deleting or renaming a file fail on Windows when it works on Linux?"**
Windows opens files with sharing modes, and replacing or deleting a file that another handle has open can fail with `EPERM`, `EBUSY` or `EACCES`. The other handle is often an antivirus scanner, the search indexer or your own process. On Linux, open handles don't block unlink or rename: the inode lives on until the last handle closes. Hence short retries with backoff on Windows, and designs that don't replace in-use files.

**Q8. "Your service crashed during a job. How do you make sure its temp files are cleaned up?"**
`try/finally` covers errors and aborts, but not SIGKILL, an OOMKill or a power cut. So give temp files a recognisable prefix in a known location, and sweep stale ones at startup, using a time threshold so you never delete a file another running instance is using. That's crash-only design: recovery runs on every start, so a crash isn't a special case.

**Q9. "Promise, callback or sync fs: which do you use?"**
Promises by default. Callbacks are measurably cheaper when you need every last bit of performance, and the docs say so. Sync only where blocking the loop doesn't matter. All the async forms run on the libuv thread pool, so heavy file I/O competes with `dns.lookup`, async crypto and zlib, which is worth knowing when DNS gets mysteriously slow during a big batch job.

---

## 14. Build & break

### Build — `src/lib/fs-safe.ts`

Implement `writeFileAtomic`, `renameWithRetry`, `safeJoin`, `safeRealJoin` and `withTempDir` from this lesson, plus `sweepStaleTemp(dir, prefix, maxAgeMs)` for boot. Then test them with `node:test`:

- `safeJoin` against every payload in §9's table: on Windows and in WSL2, since the separators differ.
- `writeFileAtomic` leaves no `.tmp` file behind on success, and none after a write that you force to fail.
- `withTempDir` removes its directory when `fn` throws.

### Break — 4 experiments

1. **Kill mid-write.** Write a script that does `await writeFile("stmt.csv", hugeString)` with a 500 MB string, start it, and kill it while it's writing (`taskkill /F /PID <pid>` on Windows, `kill -9` in WSL2). Check the file's size: it's truncated, and the previous good copy is gone. Repeat with `writeFileAtomic`: after the kill, `stmt.csv` is either the complete old file or the complete new one, plus at most a stray `.tmp` for the sweeper.
2. **`EMFILE`.** `await Promise.all(Array.from({ length: 20_000 }, () => open("one.txt", "r")))`. On Windows, a few thousand succeed and the rest fail with `EMFILE`; in WSL2, it fails near `ulimit -n`. Switch to `mapLimit(…, 32, …)`, close each handle, and it completes. Bonus: in WSL2, leak handles in a loop and watch `ls /proc/<pid>/fd | wc -l` climb.
3. **Traversal.** Build the naive route: `GET /v1/statements/:name` → `pipeline(createReadStream(path.join(STATEMENTS_DIR, req.params.name)), res)`. Create `secret.txt` one directory above `STATEMENTS_DIR` and request `curl.exe http://localhost:3000/v1/statements/..%2Fsecret.txt`: the file is served, because Express matched `..%2Fsecret.txt` as one segment and then decoded it. Try `..%5Csecret.txt` (a backslash) on Windows and in WSL2 and compare. Replace the join with `safeRealJoin`, map `PathEscapeError` to 404, and every variant fails.
4. **Rename an open file on Windows.** Open `stmt.csv` for reading in one process (`const fh = await open("stmt.csv", "r")`, then keep it alive with a long timer). From another, run `writeFileAtomic("stmt.csv", "v2")` without the retry: it fails with `EPERM`. Close the reader and it succeeds. Run the same pair in WSL2: the rename succeeds while the file is open, and the reader still sees the old contents.

### Explain out loud (60–120 seconds)

1. Why `writeFile` destroys data on a crash, and each step of temp → fsync → rename → fsync dir.
2. What `EMFILE` is, why it's a concurrency bug, and the fix order.
3. The traversal payloads, why `startsWith` fails, and the `path.relative` + `realpath` check, plus why IDs beat paths.
4. Three ways Windows differs: rename over open files, case-insensitivity, separators.

---

## Module 3 complete — checkpoint

- [ ] `emit` is synchronous: listeners run in order on the emitter's stack, and their exceptions reach the caller
- [ ] An `'error'` event with no listener throws and crashes the process
- [ ] `MaxListenersExceededWarning` is a leak detector, not a limit
- [ ] `events.once` and `events.on` turn events into promises and async iterators; in-process events are not an outbox
- [ ] A Buffer is a `Uint8Array` whose memory is outside the V8 heap; `allocUnsafe` can expose old memory
- [ ] A small `subarray` of a pooled Buffer pins the whole 8 KiB slab
- [ ] `string.length` ≠ byte length; `StringDecoder` handles characters split across chunks
- [ ] Webhook HMACs are computed over the raw body bytes and compared with `timingSafeEqual`
- [ ] `highWaterMark` is a threshold: 64 KiB for byte streams (16 KiB on Windows), 16 objects in object mode
- [ ] `write()` returning `false` means "wait for `'drain'`", and ignoring it grows RSS until an OOMKill
- [ ] `pipe` leaks the source on errors and disconnects; `pipeline` destroys every stage and takes a `signal`
- [ ] Backpressure crosses the network through TCP's receive window, all the way to a database cursor
- [ ] A paused stream still holds its pool client: isolate and bound long streams
- [ ] Async generator stages are pull-based; `yield` in `batch` carries backpressure from Postgres to the upload socket
- [ ] Line splitting needs a remainder, a `StringDecoder` and a length cap
- [ ] Node streams inside Node; Web Streams at the edges, bridged with `fromWeb`/`toWeb`
- [ ] Atomic writes are temp → `fsync` → `rename` in the same directory → `fsync` the directory
- [ ] `EMFILE` is a concurrency bug first and a limit second
- [ ] Path traversal is prevented by IDs, then `path.relative` containment and `realpath`
- [ ] Windows: renaming over an open file fails with `EPERM`; paths are case-insensitive; `\` is a separator

---

## What's next

Module 3 moved bytes through memory and onto disk. Module 4 puts them on the wire. Under every HTTP request, Postgres query and Redis command is a TCP socket, a Duplex stream with no message boundaries. Next you'll frame messages on raw bytes and speak Redis's protocol by hand, and see that every driver you use is a parser plus a queue.

Next → **[Lesson 16: TCP & the `net` module](../04-networking/16-tcp-and-net.md)**
