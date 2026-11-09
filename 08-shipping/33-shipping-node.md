# Lesson 33 — Shipping Node: containers, config & runtime flags

> **Why this lesson exists:** everything you've built behaves only as well as the last mile lets it. The base image decides whether your native addon loads, PID 1 decides whether SIGTERM ever arrives, one environment variable decides whether a memory problem is a clean heap error or a silent `exit 137`, and moving to Lambda quietly breaks every assumption a long-lived process makes about timers and connection pools. Interviewers ask "walk me through your Dockerfile" because the answer shows whether you understand the runtime or copied a template.

**Time:** ~70 minutes · **Prereq:** Lessons 24, 27, 32

---

## 1. The idea in one sentence

> **Node's defaults were chosen for a laptop: inside a container, every runtime decision (who is PID 1, how big the heap may grow, how many threads to start, which mode your libraries run in) must be made explicitly from the container's limits, or the kernel makes it for you with SIGKILL.**

---

## 2. What changes between your laptop and production

| On your laptop | In production | What goes wrong | Where |
|---|---|---|---|
| `node_modules` built for Windows | A Linux image | Native addons copied into the image fail to load | §3 |
| You press Ctrl+C | The orchestrator sends SIGTERM to PID 1 | A shell or `npm` as PID 1 swallows it; SIGKILL follows 30 s later | §3, [L24](../05-concurrency/24-graceful-shutdown.md) |
| The heap grows until 16 GB of RAM says no | A 1 GiB cgroup limit | V8 doesn't respect the limit; the kernel kills the process with no log line | §5 |
| `NODE_ENV` unset | Should be `production` | Express sends stack traces to clients; libraries run their debug builds | §4 |
| One long-lived process | Maybe Lambda | Timers freeze, sockets die, every invocation opens a pool | §9 |

None of these show up in tests, and all of them show up in the first week of production. **So you decide each one explicitly, in the image or the manifest, where a reviewer can see the decision.**

---

## 3. The production Dockerfile

```dockerfile
# syntax=docker/dockerfile:1

# ── Stage 1: production dependencies, built where compilers exist ───────────
FROM node:24-slim AS deps
WORKDIR /app
# Compilers only for native addons without a prebuilt binary. They never reach the runtime image.
RUN apt-get update \
 && apt-get install -y --no-install-recommends python3 make g++ \
 && rm -rf /var/lib/apt/lists/*
# Manifests first: this layer and the install below stay cached until the manifests change.
COPY package.json package-lock.json ./
# No dev dependencies and no install scripts; then build the one allowlisted addon (L30).
RUN --mount=type=cache,target=/root/.npm \
    npm ci --omit=dev --ignore-scripts \
 && npm rebuild argon2

# ── Stage 2: runtime ────────────────────────────────────────────────────────
FROM node:24-slim
RUN apt-get update \
 && apt-get install -y --no-install-recommends tini \
 && rm -rf /var/lib/apt/lists/*
WORKDIR /app
ENV NODE_ENV=production
COPY --from=deps /app/node_modules ./node_modules
# package.json carries "type": "module", which decides how the .ts files load.
COPY package.json ./
COPY src ./src
# Code stays root-owned, so the process can read it but never rewrite it.
# The official image ships a `node` user (uid 1000).
USER node
EXPOSE 3000
# tini is PID 1: it forwards signals and reaps orphans. Exec form: no shell in between.
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["node", "src/main.ts"]
```

Comments sit on their own lines on purpose: **a Dockerfile `#` only starts a comment at the beginning of a line**, so `USER node # comment` passes the extra words as arguments.

**Two stages, because build tools are weight and attack surface.** The deps stage has python3, make and g++ so a native addon can compile. The runtime stage receives only the compiled `node_modules`, so the image you scan and ship has no compiler for an attacker to use and nothing your process doesn't need.

**Manifests before source, because layers are cached by content.** A source edit invalidates only the `COPY src` layer, so the install layer is reused and a rebuild takes seconds. The `--mount=type=cache` keeps npm's download cache between builds without baking it into a layer.

**`npm ci --omit=dev --ignore-scripts`, then an explicit rebuild.** `npm ci` installs exactly the lockfile and fails if `package.json` disagrees with it ([L32](32-packaging-and-publishing.md)). `--omit=dev` keeps your test runner and `typescript` out of production. `--ignore-scripts` followed by `npm rebuild <addon>` is [L30](30-security-and-supply-chain.md)'s allowlist applied inside the image: only the addon you named gets to run its install script.

**No TypeScript build stage.** Ledger runs `.ts` natively (§6), so there's nothing to compile. Typechecking (`tsc --noEmit`) is a CI step that gates `docker build`, not a layer inside it. If you build to JavaScript, add a stage that installs dev dependencies and runs the build, and copy only its output.

**Non-root, with root-owned code.** Running as `node` limits what a remote-code-execution bug can do, and root-owned files mean the process can't modify its own source. The directories it must write (temp files for the exporter and ingester, [L15](../03-data-and-io/15-the-file-system.md)) come from a mounted volume, so the root filesystem can be read-only (§7).

**tini as PID 1, exec-form `CMD`.** [L24](../05-concurrency/24-graceful-shutdown.md) owns the PID 1 trap. Two consequences land here. Shell form (`CMD node src/main.ts`) makes `/bin/sh` PID 1, and SIGTERM never reaches Node. And even in exec form, **Node as PID 1 doesn't reap orphaned grandchildren**: if the `pg_dump` your backup job spawns ([L22](../05-concurrency/22-child-processes.md)) leaves children behind, they become zombies. `docker run --init` injects tini for local runs, but Kubernetes has no `--init`, so tini goes in the image.

### `.dockerignore`, especially on Windows

```text
node_modules
.git
.env*
coverage
dist
*.log
test/fixtures/large
```

Workspace images usually need `COPY . .`, and without this file it ships your Windows `node_modules` into a Linux image: **native `.node` binaries compiled for win32 that fail at load time**, plus every `.env` on your disk. A smaller build context also makes every build start faster.

### Base images: slim, alpine or distroless

| Base | libc | Shell / package manager | Choose when | The catch |
|---|---|---|---|---|
| `node:24-slim` | glibc (Debian) | ✅ | The default | Bigger than alpine |
| `node:24-alpine` | musl | ✅ (`apk`) | Size dominates and you have no native addons | glibc-built addons won't load; musl's resolver behaves differently; musl is a lower support tier than glibc in Node's own platform list |
| distroless Node | glibc | ❌ | Hardened production | No shell for `kubectl exec` (use `kubectl debug`); no `sleep` binary for a preStop exec; the entrypoint is already `node` |

**The musl point is mechanical, not a matter of taste.** A native addon is a shared library linked against a C library. A `.node` file built on glibc references glibc's dynamic loader and symbols, which don't exist on musl, so `require()` fails with a dynamic-linker error. Packages that download or select a prebuilt binary at install time choose one for the libc they detect, so installing on one libc and running on the other breaks the same way. And because `dns.lookup` calls libc's `getaddrinfo` ([L07](../02-event-loop/07-the-thread-pool.md)), every outbound connection by hostname inherits musl's resolver, which has historically handled `resolv.conf` options and truncated responses differently from glibc's. **Install and run on the same base: that's why both stages above start `FROM node:24-slim`.**

Ledger pins the base by digest (`node:24-slim@sha256:…`) and lets Renovate propose digest bumps, so security patches to the base still arrive weekly.

### The workspace version

Ledger is a pnpm workspace ([L32](32-packaging-and-publishing.md)), and **one image serves every process type**: the api, the ingester and each worker choose their entrypoint in the manifest. The shape stays the same with two changes. `pnpm fetch --prod` downloads packages using only `pnpm-lock.yaml`, so that layer survives source edits; then you copy the workspace and run `pnpm install --offline --frozen-lockfile --prod`. And `packages/shared` stays a workspace symlink rather than being copied into `node_modules`, because **Node refuses to strip types from files under `node_modules`**: an internal package consumed as `.ts` must keep a real path outside it.

---

## 4. `NODE_ENV=production`: what it actually changes

**Node itself never reads `NODE_ENV`.** It's a convention that libraries and tools check, so its effect is exactly the sum of what your dependencies do with it:

| Who reads it | What `production` changes |
|---|---|
| Node core | Nothing: not V8, not the GC, not error handling |
| npm | `npm install` and `npm ci` omit `devDependencies` by default |
| Express | `app.get("env")`; template caching on; the default error handler stops putting the stack trace in the response body |
| React on the server, and many libraries | They load their production builds, with development checks and warnings off |
| Your code | Whatever you wrote — which is where it usually goes wrong |

It doesn't make anything faster by itself, set the heap, or make the app secure. **And it isn't your environment's name.** Staging should run `NODE_ENV=production` too, or staging runs different library code paths from production and stops being a rehearsal. Put the environment's identity in its own variable (`LEDGER_ENV=staging`) and validate it with the rest of the config ([L03](../01-foundations/03-the-process.md)).

**The build-stage trap:** set `ENV NODE_ENV=production` above `npm ci` in a stage that builds, and npm silently skips `typescript`, `esbuild` and every other dev dependency. The build fails with `tsc: not found`, or worse, a package you need at runtime turns out to be listed as a dev dependency and vanishes. Set `NODE_ENV` in the runtime stage only, and write `--omit=dev` explicitly where you mean it.

---

## 5. Runtime flags: `NODE_OPTIONS`, memory and source maps

**Put runtime flags in `NODE_OPTIONS` in the manifest, not in `CMD`.** An operator can then change the heap size without rebuilding and re-promoting the image. Two details make it sharper than it looks. Node accepts only an allowlisted set of flags there (V8's memory flags are on it). And **every Node process that inherits the environment gets the flags**: `npm`, `node --run`, child processes you spawn, and the `node scripts/migrate.ts` someone runs with `kubectl exec` inside a pod that's already near its limit. A second Node process in the same container shares the same cgroup limit, so one-off scripts belong in a Job, not in a running pod.

### The memory arithmetic

```text
container memory limit (cgroup)        1024 MiB  ← above this: kernel SIGKILL, exit 137, no log line
├─ V8 old space   --max-old-space-size=768       ← above this: V8 fails with a clean fatal error
├─ V8 new space, code space, large objects
├─ Buffers / ArrayBuffers ("external")           ← not counted against the heap flag (L12, L27)
├─ libuv thread stacks, zlib and OpenSSL state
└─ everything else in RSS
```

[L27](../07-production/27-memory-and-gc.md) explains the spaces; here's the operational rule. **Set `--max-old-space-size` explicitly to about 75% of the container limit.** The default is derived from the memory Node detects, and whether that reflects your cgroup limit depends on the version and the environment, so don't rely on it. The two orderings fail very differently:

| Heap flag vs container limit | When memory fills | What you're left with |
|---|---|---|
| Heap limit **below** the container limit | V8 prints `FATAL ERROR: Reached heap limit … JavaScript heap out of memory` and aborts | A log line, a stack, and whatever `--heapsnapshot-near-heap-limit` and `--report-on-fatalerror` wrote ([L27](../07-production/27-memory-and-gc.md)) |
| Heap limit **above** the container limit | RSS crosses the cgroup limit first, and the kernel's OOM killer sends SIGKILL | `OOMKilled: true`, exit 137, and nothing in the logs |

You want the first failure every time, because it's diagnosable. **Count every isolate, too.** Each worker thread has its own heap. The flag applies to each isolate, and the worker_threads docs state that `--max-old-space-size` overrides `resourceLimits.maxOldGenerationSizeMb` when both are set. So set both to the same number and budget `(1 + workers) × heap` against the limit; Ledger's reconciler does exactly that ([L34](../09-project/34-ledger-engine-blueprint.md)).

A boot-time check turns a misconfigured manifest into a log line before the first OOM:

```ts
// packages/shared/src/memory-guard.ts
import { getHeapStatistics } from "node:v8";
import type { Logger } from "pino";

export function warnIfHeapNearContainerLimit(log: Logger): void {
  const heapLimit = getHeapStatistics().heap_size_limit; // what V8 will allow this isolate
  const containerLimit = process.constrainedMemory() ?? 0; // the cgroup limit; 0 when unknown
  if (containerLimit > 0 && heapLimit > containerLimit * 0.8) {
    log.warn({ heapLimit, containerLimit }, "heap limit near container limit: expect OOMKilled, not a heap error");
  }
}
```

> **Spring equivalent:** the JVM reads the cgroup limit and sizes its heap with `-XX:MaxRAMPercentage=75`, passed through `JAVA_TOOL_OPTIONS` so operators can change it without a rebuild ([Spring L40](https://github.com/Pawar-Pratik/SpringBoot/blob/main/11-deployment/40-docker-cicd-deployment.md)). In Node you write an absolute number into `NODE_OPTIONS` next to the limit in the manifest, and keep the two lines adjacent so nobody changes one without the other.

### `UV_THREADPOOL_SIZE` lives here too

It must be in the environment at launch ([L07](../02-event-loop/07-the-thread-pool.md)), so it sits in the manifest beside the heap flag and is sized per process type: Ledger's api runs 6, its DNS-heavy dispatcher 16, everything else the default 4. The arithmetic is in [L34](../09-project/34-ledger-engine-blueprint.md).

### `--enable-source-maps` and what it costs

If you ship a bundle, stack traces point at `bundle.js:1:48213` unless Node maps them back. `--enable-source-maps` does that, but it isn't free: Node keeps the parsed maps in memory and does the mapping whenever a stack trace is generated, and its docs warn about latency when `Error.stack` is accessed frequently. **That cost peaks exactly when a downstream dies and you log ten thousand errors a second.** Native type stripping avoids it entirely: stripping replaces annotations with whitespace, so every line and column in a `.ts` file stays where it was and stack traces are already correct.

---

## 6. Run `.ts` natively, or build to JavaScript?

| | Native `.ts` (type stripping) | Build to JS (tsc emit / esbuild bundle) |
|---|---|---|
| Artefact | The reviewed source is the artefact | Compiled output, built in CI |
| Stack traces | Exact, no source maps | Need source maps and `--enable-source-maps` |
| Startup | Each file is read and stripped at load | A bundle is one file: the fastest cold start |
| Syntax | Erasable only; no tsconfig `paths` | Anything: enums, NestJS decorators with metadata, `paths` |
| Under `node_modules` | Refused: packages must ship JavaScript | Not applicable |
| Type safety | None at runtime either way: `tsc --noEmit` gates CI | The same |

**Ledger's choice is made per artefact.** The api and workers run `.ts` natively: they're long-lived, so stripping is paid once per pod start (measure it as the time from process start to `listening`, well inside the startup probe's budget), the image contains exactly what was reviewed, and stack traces need no maps. `@ledger/node` and the `ledger` CLI are built to JavaScript plus `.d.ts`, because consumers load them from `node_modules` ([L32](32-packaging-and-publishing.md)). Anything that would run on Lambda gets bundled with esbuild, because there cold start is paid per environment, not per deploy (§9). See [TS L18](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/05-ecosystem/18-tooling-and-performance.md) for the toolchain itself. **"Always build" and "never build" are both the weak answer; "per artefact, for this reason" is the strong one.**

---

## 7. Kubernetes manifests for a Node process

[API L20](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/20-observability.md) owns the liveness/readiness rule and [L24](../05-concurrency/24-graceful-shutdown.md) owns the shutdown sequence. This is the manifest that implements them for the Ledger api:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: ledger-api }
spec:
  replicas: 4
  strategy: { type: RollingUpdate, rollingUpdate: { maxSurge: 1, maxUnavailable: 0 } }
  selector: { matchLabels: { app: ledger-api } }
  template:
    metadata: { labels: { app: ledger-api } }
    spec:
      terminationGracePeriodSeconds: 40          # > preStop 5 s + in-app drain deadline 25 s, plus margin
      containers:
        - name: api
          image: registry.example.com/ledger@sha256:4f1c0d   # one image, promoted by digest
          args: ["node", "apps/api/src/main.ts"] # replaces CMD; tini stays as ENTRYPOINT
          ports: [{ containerPort: 3000 }]
          env:
            - { name: NODE_ENV, value: production }
            - { name: LEDGER_ENV, value: production }
            - { name: NODE_OPTIONS, value: "--max-old-space-size=768" }   # 75% of the 1Gi limit below
            - { name: UV_THREADPOOL_SIZE, value: "6" }
          envFrom: [{ secretRef: { name: ledger-api-secrets } }]
          resources:
            requests: { cpu: "1", memory: 1Gi }
            limits:   { cpu: "2", memory: 1Gi }  # memory request = limit: no overcommit surprises
          startupProbe:   { httpGet: { path: /healthz, port: 3000 }, periodSeconds: 2, failureThreshold: 15 }
          livenessProbe:  { httpGet: { path: /healthz, port: 3000 }, periodSeconds: 10, timeoutSeconds: 2, failureThreshold: 3 }
          readinessProbe: { httpGet: { path: /readyz, port: 3000 }, periodSeconds: 5, timeoutSeconds: 2, failureThreshold: 2 }
          lifecycle:
            preStop: { sleep: { seconds: 5 } }   # built-in sleep action; needs a recent cluster
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000                      # "node" is a name; the kubelet can only verify a number
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities: { drop: ["ALL"] }
          volumeMounts: [{ name: tmp, mountPath: /tmp }]
      volumes: [{ name: tmp, emptyDir: { sizeLimit: 2Gi } }]
```

The Node-specific reasoning in that file:

- **Liveness is answered by the event loop.** `/healthz` runs on the same JavaScript thread as every request, so a loop blocked longer than `timeoutSeconds` fails the probe. That's correct for a wedged loop, but **a restart throws away every in-flight request on the pod**, so liveness gets slack (three failures, ten seconds apart) and no dependency checks. Don't fail readiness on high loop delay either: pulling a busy pod out shifts its load onto the others, which then fail too. Shed individual requests with 503 instead ([L08](../02-event-loop/08-blocking-the-loop.md)).
- **The startup probe** covers boot (config validation, the pool's first connection, stripping every module) so liveness doesn't kill a pod that is merely starting.
- **preStop:** the five seconds let endpoint removal propagate before SIGTERM arrives ([L24](../05-concurrency/24-graceful-shutdown.md)). Recent Kubernetes versions have a built-in `sleep` action; on an older cluster you'd need `exec: { command: ["sleep", "5"] }`, which needs a `sleep` binary in the image, which distroless doesn't have. The portable alternative is L24's in-app delay: fail readiness, wait, then close.
- **`terminationGracePeriodSeconds` must exceed preStop plus the drain deadline**, or the kubelet SIGKILLs the pod mid-drain.
- **CPU:** the JavaScript thread uses at most one core; the limit of 2 leaves room for GC threads and the pool's gzip and scrypt work. Know what the limit costs: CFS throttling stalls the whole process for the rest of a scheduling period, which shows up as tail latency ([L23](../05-concurrency/23-cluster-and-scaling.md)).
- **`runAsUser: 1000`:** with `runAsNonRoot: true` and an image whose `USER` is the name `node`, the kubelet refuses to start the container because it can't verify that a name isn't root. Give it the number.
- **`readOnlyRootFilesystem` plus an `emptyDir` at `/tmp`:** `os.tmpdir()` stays writable and nothing else is, and `sizeLimit` caps how much disk a runaway temp file can take.
- **DNS:** pods default to `ndots:5`, so a lookup of `hooks.merchant.com` first tries every cluster search domain, all inside one `getaddrinfo` call holding one pool thread ([L07](../02-event-loop/07-the-thread-pool.md)). The dispatcher, which resolves merchant hostnames all day, sets `dnsConfig: { options: [{ name: ndots, value: "2" }] }`.

---

## 8. Config: twelve-factor, validated at boot

The image is identical in every environment and only the environment differs; that's what lets you promote one digest from staging to production and know you run what you tested (the same argument as [Spring L40](https://github.com/Pawar-Pratik/SpringBoot/blob/main/11-deployment/40-docker-cicd-deployment.md)). In practice:

- Config comes from environment variables (`env` for plain values, a Secret through `envFrom` for credentials), parsed and validated once at boot with the schema from [L03](../01-foundations/03-the-process.md). **A missing secret crashes the pod at start, where the rollout stops, not on the first payment.**
- `--env-file` and `process.loadEnvFile()` are for local development. `.env` files never enter the image (`.dockerignore`) and are never committed.
- The first log line of every pod states what's running: the config with secrets redacted, `GIT_SHA`, `process.version`, the heap limit and `UV_THREADPOOL_SIZE`.

---

## 9. Serverless Node: what Lambda changes

```text
cold start:  start runtime ─► INIT: your module scope runs once ─► invoke ─► FREEZE
warm:                                                       thaw ─► invoke ─► FREEZE ─► … ─► discarded
one environment runs one invocation at a time; concurrency means more environments
```

Lambda keeps the event loop but changes its lifecycle. **Between invocations the environment is frozen:** no timers fire, no callbacks run, and the process doesn't know that time passed. Every assumption of a long-lived process needs rechecking:

| Long-lived assumption | On Lambda |
|---|---|
| A `setInterval` flushes metrics every 10 s | It doesn't fire while frozen; flush before you return |
| A keep-alive socket stays usable | The peer or a NAT may drop it while you're frozen; the first call after a thaw can fail with `ECONNRESET` |
| A fire-and-forget promise finishes in the background | The invocation ends when the handler's promise settles; leftover work freezes and may run during a later invocation, or never |
| One pool serves many concurrent requests | Each environment serves one request at a time, so 500 concurrent requests means 500 environments, each with its own pool |
| SIGTERM triggers your graceful shutdown | Not a lifecycle you control: assume an environment can disappear |

**The connection storm** is the classic failure. A handler that runs `new Pool()` *inside* the handler opens new connections on every invocation and never closes them, so a warm environment accumulates connections until Postgres answers `too many clients`. The fix has three parts: create the pool at module scope (in INIT, reused while warm); set `max: 1`, because the environment never runs two invocations at once; and put RDS Proxy (or another pooler) in front, because a thousand environments must not become a thousand Postgres backends. Then bundle the function with esbuild into one minified file to cut cold-start load time, `import()` rarely used dependencies lazily, and pay for provisioned concurrency only where a cold start is unacceptable.

**Ledger stays on containers, and the reasons are in that table.** The live feed holds SSE connections for hours, each api pod owns a `LISTEN` connection, the dispatcher is a polling loop, the reconciler keeps a warm worker pool, and a 4M-row export can outlast Lambda's 15-minute maximum. Lambda fits the other end of the wire: a merchant's webhook receiver calling `@ledger/node`'s `webhooks.verify()`.

---

## 10. Single executable applications

For a CLI whose users may not have Node, a single executable application (SEA) packs the `node` binary and your script into one file. The shape: bundle the CLI into one script, describe it in a `sea-config.json`, generate a preparation blob with `node --experimental-sea-config sea-config.json`, inject the blob into a copy of the `node` binary with `postject`, and re-sign the result on macOS and Windows. **The feature is still marked as under active development, so follow your version's docs** (module-format support and the build commands have been changing), and expect a file about the size of the Node binary itself. Ledger ships the `ledger` CLI both ways: `npm i -g ledger` for Node users, and a SEA download for everyone else.

---

## 11. The upgrade policy

- **Production runs an LTS line.** Today Node 24 is LTS, Node 26 becomes LTS in late October 2026, Node 22 is in maintenance until 2027-04-30, and Node 20 has been end-of-life since 2026-04-30, which means it gets no security fixes. From Node 27 there's one major release a year, in April, promoted to LTS in October, and every release becomes LTS.
- **CI runs the next line from its release**, non-blocking until it reaches LTS. Ledger's matrix: 24 (blocking) and 26 (becoming blocking now that it's LTS). Production moves a few patch releases after promotion, once that matrix entry has stayed green.
- **Read the semver-major section of the release notes and the deprecation list**, and run the test suite with `--throw-deprecation` so a runtime deprecation fails CI instead of scrolling past in a log.
- **Native addons are built for one Node ABI** unless they use Node-API, which is ABI-stable across majors. That's one more reason the deps stage builds on the base you run.
- **One source of truth for the version:** `.nvmrc`, `engines` and the base image tag must agree, and a CI step checks that they do.

---

## 12. Production rules

| Rule | Why |
|---|---|
| **Multi-stage: compile in one stage, run from a clean one** | Compilers and dev dependencies are weight and attack surface you don't need at runtime |
| **Copy manifests before source; `npm ci`, never `npm install`** | Dependency layers cache across source edits, and the install matches the lockfile exactly |
| **Build `node_modules` on the same base and libc you run** | Native addons link against one libc; crossing OS or libc breaks them at load time |
| **`.dockerignore` `node_modules`, `.git` and `.env*`** | Otherwise Windows binaries and secrets end up in a Linux image |
| **Non-root, root-owned code, read-only root filesystem** | An RCE can't rewrite the app or write anywhere except the mounted temp volume |
| **tini as PID 1, exec-form `CMD`, never `npm start`** | Signals reach Node, and orphaned grandchildren get reaped |
| **`NODE_ENV=production` in every deployed environment; the environment's name in its own variable** | Staging must run the same library code paths as production |
| **Runtime flags in `NODE_OPTIONS` in the manifest** | Changing the heap shouldn't need a rebuild and a re-promotion |
| **`--max-old-space-size` ≈ 75% of the limit, budgeted per isolate** | A heap error is diagnosable; an OOMKill leaves nothing behind |
| **Liveness without dependencies and with slack; shed load instead of failing readiness** | A blip or a busy loop shouldn't restart or drain the fleet |
| **`terminationGracePeriodSeconds` > preStop + drain deadline** | Otherwise the kubelet SIGKILLs you mid-drain |
| **Pin the base image by digest; let a bot propose bumps** | Builds are reproducible, and base-image patches still arrive |
| **On Lambda: module-scope clients, `max: 1`, a pooler in front, bundled code** | Each environment is single-request, so connections multiply with concurrency |
| **LTS in production, the next LTS in CI, `--throw-deprecation` in tests** | Upgrades become routine instead of archaeology |

> **Spring equivalent:** Boot's layered-jar extraction is the same idea as copying manifests before source: keep the rarely changing dependency layer cached and push only the application layer. Actuator's liveness and readiness groups map onto `/healthz` and `/readyz`, and `server.shutdown=graceful` onto L24's lifecycle ([Spring L40](https://github.com/Pawar-Pratik/SpringBoot/blob/main/11-deployment/40-docker-cicd-deployment.md)).

---

## 13. Interview traps

**Q1. "Walk me through a production Node Dockerfile."**
A deps stage on the same base as the runtime, with compilers, that copies only the manifests and runs `npm ci --omit=dev` so the layer caches; a runtime stage that copies `node_modules` and the source, sets `NODE_ENV=production`, runs as the `node` user with root-owned code, and starts with tini as PID 1 and an exec-form `CMD ["node", "src/main.ts"]`; plus a `.dockerignore` and a digest-pinned base. The weak answer recites lines; the strong one gives each line's reason: cache, attack surface, libc, signals. **Follow-up:** "Why not `CMD npm start`?" npm becomes PID 1 and doesn't reliably forward SIGTERM, so Node never starts its graceful shutdown and the kubelet SIGKILLs it after the grace period ([L24](../05-concurrency/24-graceful-shutdown.md)).

**Q2. "What does `NODE_ENV=production` change?"**
In Node core, nothing. npm omits dev dependencies; Express caches templates and stops sending stack traces in error responses; React and other libraries load their production builds. It doesn't tune V8 or secure anything, so "it makes Node faster" is the weak answer. **Follow-up:** "What should staging use?" `production` as well, with the environment's name in a separate variable; otherwise staging runs different library code from production and stops being a rehearsal.

**Q3. "alpine or slim for Node?"**
slim by default. Alpine uses musl: glibc-built native addons fail to load, prebuilt binaries are chosen by libc at install time, `dns.lookup` inherits musl's resolver, and musl is a lower support tier in Node's own platform list. Alpine is fine for a pure-JavaScript service where size matters most, provided you install and run on alpine. Distroless is the hardening step, at the cost of a shell.

**Q4. "How do you set memory flags in a container?"**
`--max-old-space-size` at about 75% of the container limit, in `NODE_OPTIONS` next to the limit in the manifest, so the heap fills before the cgroup does and you get V8's fatal error instead of a silent `exit 137`. The other 25% covers new space, code space, Buffers and native allocations, none of which the flag limits. **Follow-up:** "And with worker threads?" Each isolate has its own heap, so the budget is `(1 + workers) × heap`; set `resourceLimits` and the flag to the same value, because the flag overrides `resourceLimits` when both are set.

**Q5. "The pod was OOMKilled, but the heap graph peaked at 400 MB of a 1 GB limit. Why?"**
RSS isn't the heap. Buffers, `external` memory, thread stacks and native allocations count against the cgroup and not against the heap flag, so the usual suspects are buffered bodies, a stream ignoring backpressure ([L13](../03-data-and-io/13-streams-and-backpressure.md)) or pooled-buffer retention ([L12](../03-data-and-io/12-buffers-and-encodings.md)). The other cause is a heap flag set above the limit, so the heap never reached its own ceiling. Graph `rss`, `heapUsed` and `external` side by side ([L27](../07-production/27-memory-and-gc.md)).

**Q6. "What breaks when you move a Node service to Lambda?"**
The frozen-environment model: timers don't fire between invocations, keep-alive sockets can be dead after a thaw, background promises freeze once the handler's promise settles, and concurrency comes from more single-request environments, each with its own pool. Long-lived connections (SSE, `LISTEN`) and in-process workers don't fit at all. **Follow-up:** "How many connections can a pool-per-environment design open?" Environments × pool `max`: 500 concurrent invocations with `max: 10` can mean 5,000 backends, which is why it's module scope, `max: 1`, and RDS Proxy.

**Q7. "Build to JS or run `.ts` natively in production?"**
Per artefact. Long-lived services can run natively: no build step, exact stack traces because stripping preserves positions, and stripping paid once per start. The constraints are erasable syntax only, no `paths`, and typechecking moved entirely to CI. Published packages must ship JavaScript because Node won't strip under `node_modules`; cold-start-sensitive code (Lambda, CLIs) gets bundled; NestJS needs a build because decorator metadata isn't erasable.

**Q8. "What should a Node liveness probe check?"**
Only that the process can answer: no database, no Redis ([API L20](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/20-observability.md)). The Node-specific part is that the probe runs on the event loop, so a blocked loop fails it. That's right for a wedged process, but a restart drops every in-flight request on the pod, so give liveness slack. And don't use readiness to react to load, because removing a busy pod moves its traffic onto the others; shed requests with 503 based on loop delay instead.

**Q9. "It works locally, but the container crashes with `invalid ELF header`. What happened?"**
A native addon compiled for another platform got into the image, usually the host's `node_modules` copied in by `COPY . .` without a `.dockerignore`, so a Windows or macOS binary is being loaded on Linux. Fix: ignore `node_modules` and install inside the image, on the base you run.

**Q10. "How do you upgrade Node majors safely?"**
Production on LTS; the next line in the CI matrix from its April release, blocking once it's LTS; read the semver-major notes and run tests with `--throw-deprecation`; rebuild native addons (Node-API ones survive across majors); move production a few patches after promotion; keep `.nvmrc`, `engines` and the base image in lockstep. Never stay on an end-of-life line: it gets no security releases.

---

## 14. Build & break

### Build — the image and the manifests
- `deploy/Dockerfile`: the §3 shape for the pnpm workspace. One image containing `apps/api`, `apps/workers` and `packages/shared` with production dependencies only, tini, `USER node`, and a default `CMD` that starts the api. Typechecking runs in CI before `docker build`, not inside it.
- `deploy/k8s/`: `api.yaml` (§7); `dispatcher.yaml` (no Service and no preStop, since nothing routes to it; `ndots: 2`; `UV_THREADPOOL_SIZE=16`; a 512Mi limit with `--max-old-space-size=384`; a 30 s grace period); `reconciler.yaml` (`strategy: Recreate`; requests equal to limits at 4 CPUs and 2Gi; `RECONCILE_THREADS=3`; 384 MiB per isolate); and a `migrate` Job that runs before each rollout. **Take every number from the [L34](../09-project/34-ledger-engine-blueprint.md) budgets.**
- Call `warnIfHeapNearContainerLimit()` at the start of every entrypoint.

### Build — the local stack
`deploy/compose.yaml`: Postgres 16 and Redis 7 with healthchecks, then the api, dispatcher, outbox relay and reconciler from the same image with different `command`s, `depends_on: { <service>: { condition: service_healthy } }`, the same env vars as the manifests, and `mem_limit` set so the memory flags get exercised locally. The api's healthcheck needs no curl in the image: `["CMD", "node", "-e", "fetch('http://localhost:3000/healthz').then(r => process.exit(r.ok ? 0 : 1), () => process.exit(1))"]`. Kubernetes ignores Dockerfile `HEALTHCHECK`s, so probes live in the manifests.

### Break — six experiments
1. **The fat image.** Build a single-stage image: `FROM node:24`, `COPY . .`, `npm install`. Compare its size with your multi-stage image in `docker images`, then run `docker run --rm <fat-image> ls node_modules/.bin` and find `tsc` and your test runner in "production". Fix: the two-stage build.
2. **A native addon across libcs.** In a scratch Dockerfile, install an addon that compiles or downloads a single binary at install time (`better-sqlite3` is convenient) in a `node:24-slim` stage, then copy that `node_modules` into a `node:24-alpine` stage. Run a script that imports it and read the dynamic-linker error. Fix: install in an alpine stage, or use slim on both sides.
3. **`NODE_ENV` unset.** A bare Express 5 app with one route that throws and no error middleware. With `NODE_ENV` unset, `curl -s localhost:3000/boom` (in Git Bash; in PowerShell use `curl.exe`) returns HTML containing the stack trace and your file paths. Restart with `NODE_ENV=production`: the body is just "Internal Server Error". **Every framework default you didn't override is a production setting you forgot.**
4. **The build-stage trap.** Put `ENV NODE_ENV=production` above `npm ci` in a stage that runs `npx tsc --noEmit`, and watch the build fail because `typescript` was never installed. Fix: move the `ENV` to the runtime stage.
5. **A pool per invocation.** Write `handler.ts` that creates a `new Pool()` inside the handler, runs `SELECT 1` and returns without `end()`. Call it 150 times in a loop (one warm environment over a busy minute) while watching `SELECT count(*) FROM pg_stat_activity` climb until Postgres answers `sorry, too many clients already` (the default `max_connections` is 100). Move the pool to module scope with `max: 1` and the count stays at one.
6. **A heap flag above the limit.** Write a script that keeps pushing `new Array(100_000).fill(1.5)` (about 800 KB of heap each) into an array. Run it with `docker run -m 256m -e NODE_OPTIONS=--max-old-space-size=1024 …`: it dies with exit 137, `docker inspect --format '{{.State.OOMKilled}}' <container>` prints `true`, and the logs say nothing. Rerun with `--max-old-space-size=192 --report-on-fatalerror --report-directory=/tmp`: you get V8's "JavaScript heap out of memory" with a stack, plus a diagnostic report. **Same leak, one diagnosable failure and one silent one.**

### Explain out loud (60–120 seconds)
1. The Dockerfile line by line, with the reason for each line: cache, attack surface, libc, signals.
2. What `NODE_ENV=production` changes and what it doesn't, and why staging uses it too.
3. The memory arithmetic: 75% of the limit, per isolate, and heap error vs OOMKilled.
4. Native `.ts` vs a build, decided per artefact.
5. The liveness probe runs on the event loop, and why load shouldn't fail readiness.
6. Three things Lambda's freeze breaks, and the connection-pool fix.

---

## Module 8 complete — checkpoint

- [ ] Why install scripts make every transitive dependency part of your attack surface, and what lockfiles, `npm ci --ignore-scripts`, provenance and a release-age delay each defend against
- [ ] How prototype pollution flips an authorization check, and the defences (`Object.create(null)`, `Map`, schema validation)
- [ ] Why `vm` isn't a sandbox, and why the permission model is defence in depth rather than isolation
- [ ] `randomBytes` vs `Math.random` for tokens; why `timingSafeEqual` needs equal lengths
- [ ] Testing timers and backoff schedules with mock timers instead of sleeps
- [ ] Testing backpressure with a slow writer, and shutdown by spawning, loading and signalling
- [ ] Why a test runner hangs (open handles), and how to find them
- [ ] The `exports` conditions, why `types` comes first, and how `exports` blocks deep imports
- [ ] What `^0.2.3` matches, what a lockfile pins, and why consumers ignore a library's lockfile
- [ ] Phantom dependencies, and pnpm's strict `node_modules`
- [ ] Why ESM-only is the 2026 default: every supported line can `require(esm)`
- [ ] Multi-stage, non-root, exec form, tini, and never copying `node_modules` across OS or libc
- [ ] What `NODE_ENV=production` changes (npm, frameworks) and what it doesn't (Node core)
- [ ] The heap flag at ~75% of the limit, per isolate; heap error vs OOMKilled
- [ ] Native `.ts` vs a build, decided per artefact
- [ ] What Lambda's freeze does to timers, sockets and connection pools
- [ ] The LTS upgrade policy and the CI matrix that enforces it

---

## What's next

That's the end of technique. Lesson 34 assembles thirty-three lessons into one system: the Ledger Engine blueprint, with every process, data flow, budget and decision written down along with its arithmetic and its reason.

Next → **[Lesson 34: Ledger Engine blueprint](../09-project/34-ledger-engine-blueprint.md)**
