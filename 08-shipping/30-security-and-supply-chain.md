# Lesson 30 — Security & the supply chain

> **Why this lesson exists:** in Node, the largest attack surface is the code you didn't write: hundreds of transitive packages, any of which can run a script on your laptop or your CI runner the moment you type `npm install`. After that come the bug classes JavaScript makes uniquely easy, where data turns into behaviour: prototype pollution, `eval`, and the shell. "A dependency was compromised overnight: what protected you?" is now a standard senior question, and "we run `npm audit`" is the answer that ends the interview badly.

**Time:** ~85 minutes · **Prereq:** Lessons 18, 22

---

## 1. The idea in one sentence

> **Every install runs strangers' code with your privileges, so the first defence is controlling exactly what gets installed and what it may execute, and the second is never letting input reach the JavaScript features that turn data into behaviour.**

---

## 2. The attack surface you inherited

Count yours before reading on: `npm ls --all --omit=dev | wc -l` in the Ledger API. Every line is a package whose maintainer account, build machine and future intentions you're trusting.

What a dependency can do, and when:

| When | What runs | With what access |
|---|---|---|
| **Install** | `preinstall` / `install` / `postinstall` scripts of **every transitive dependency** | Your user, your shell, your env: **on CI that means the npm token and cloud credentials** |
| **Import** | Top-level module code, on the first `import` | The whole process: `process.env`, the file system, the network |
| **Runtime** | Any function you call | Same, plus every request's data that flows through it |

The incidents worth knowing, each with the mechanism and the control that would have helped:

| When | What | Mechanism | What would have helped |
|---|---|---|---|
| March 2016 | **left-pad** | A maintainer unpublished a tiny package; builds broke worldwide | Lockfile + a registry proxy that caches what you use; npm tightened its unpublish policy afterwards |
| November 2018 | **event-stream / flatmap-stream** | The maintainer handed the package to a stranger, who added a dependency carrying an encrypted payload aimed at the **Copay** bitcoin wallet | Reviewing lockfile diffs: the attack arrived as a **new transitive dependency** in a release that `^` ranges accepted |
| October 2021 | **ua-parser-js** | A hijacked maintainer account published malicious versions that ran at install time | Not running install scripts; a release-age delay |
| January 2022 | **colors, faker** | The maintainer sabotaged his own packages | Lockfiles and `npm ci`: existing builds were safe until someone updated |
| September 2025 | **chalk, debug** and other very popular packages | A **phished** maintainer account published malicious versions | Release-age delay; lockfile; verifying provenance |
| September 2025 | **Shai-Hulud** | A self-replicating worm: install scripts stole tokens and secrets, then used stolen npm tokens to publish itself into more packages | No install scripts; no long-lived publish tokens; trusted publishing |

npm's response to that year moved the defaults: **trusted publishing** (CI publishes via short-lived OIDC credentials, no stored token), **deprecating classic tokens** in favour of short-lived granular ones, and **enforced 2FA for publishing**. The lesson for consumers is the other half: **none of this protects you from a legitimate maintainer's account publishing malware; your own pipeline has to.**

---

## 3. Defence in layers

No single control stops all six incidents. Layers do:

```text
  ┌─ PIN ────────── lockfile + npm ci           → nothing new arrives without a reviewed diff
  ├─ DON'T EXECUTE ─ --ignore-scripts / pnpm 10 → install-time payloads never run
  ├─ DELAY ──────── release-age cooldown        → fast, noisy compromises get pulled first
  ├─ VERIFY ─────── signatures + provenance     → "built from this repo by this workflow"
  ├─ SCAN ───────── audit / Socket / Snyk       → known-bad and suspicious-behaviour signals
  └─ REDUCE ─────── fewer deps, a registry proxy, scoped names, least-privilege CI tokens
```

### Pin: the lockfile and `npm ci`

`package.json` says `"pg": "^8.13.0"`, a range. `package-lock.json` records the **exact** version, tarball URL and **integrity hash** of every package in the tree. `npm ci`:

- installs exactly what the lockfile says and **verifies each tarball's hash**;
- **fails** if `package.json` and the lockfile disagree, instead of "fixing" the lockfile;
- deletes `node_modules` first, so there's no leftover state;
- never writes the lockfile.

`npm install` resolves ranges and may rewrite the lockfile, so CI and Docker builds use `npm ci`, always. Updates happen in reviewed pull requests whose lockfile diff you actually read: **a new transitive package in a patch bump is exactly what event-stream looked like.** [Lesson 32](32-packaging-and-publishing.md) covers what the lockfile pins in depth.

### Don't execute: install scripts off by default

```bash
npm ci --ignore-scripts      # no lifecycle scripts from ANY dependency
npm rebuild argon2           # the allowlist: packages you reviewed that need a build step
```

Most packages need no install script at all; the ones that do are usually native addons. Make the list explicit and short. pnpm 10 does this by default: dependencies' lifecycle scripts don't run unless allowlisted, and pnpm tells you what it skipped.

```jsonc
// package.json (pnpm): the allowlist lives in the repo, reviewed like code
{ "pnpm": { "onlyBuiltDependencies": ["argon2"] } }
```

Remember what this does **not** cover: code that runs at import or call time. Install-script blocking stops the worm-at-install class of attack, not a malicious function you later call.

### Delay: a release-age cooldown

Most account-takeover attacks are noisy: the malicious version is spotted and pulled quickly. A policy of **not adopting any version younger than a few days** skips that window entirely. Renovate supports it directly (`minimumReleaseAge`), and package managers have been adding equivalents, so check yours. The cooldown does **not** stop patient attacks (event-stream went unnoticed for weeks), and it delays security fixes too, so allow a manual fast path for an advisory you've read.

### Verify: signatures, provenance and trusted publishing

- **Registry signatures**: `npm audit signatures` verifies that installed packages were signed by the registry, and checks provenance attestations where they exist.
- **Provenance** (`npm publish --provenance` from a supported CI) attaches a signed statement: *this tarball was built from commit X of repo Y by workflow Z*. It turns "a stolen token published from someone's laptop" into a detectable anomaly, because that version has no provenance while its predecessors did.
- **Trusted publishing** removes the long-lived token from CI entirely: the registry trusts a specific repository and workflow, and CI exchanges an OIDC token for a short-lived publish credential. There's no token for a worm to steal.

**What provenance doesn't prove: that the code is benign.** A compromised maintainer whose CI is also compromised produces perfectly valid provenance for malware. It proves origin, not intent.

### Scan, and know each scanner's blind spot

| Tool | Sees | Misses |
|---|---|---|
| `npm audit` | Packages with a **published advisory** (known CVEs) | Brand-new malware (no advisory yet); reachability: it can't tell whether the vulnerable code sees attacker input |
| Socket and similar behavioural scanners | What a new version *does*: install scripts added, network or filesystem access, obfuscation | Subtle logic changes |
| Snyk, Dependabot alerts | Known vulnerabilities, with fix PRs | The same zero-day gap as `npm audit` |

### Reduce

- **Fewer dependencies.** A ten-line helper you own beats a package whose maintainer you don't know. Ledger's SDK ships with **zero runtime dependencies** for exactly this reason ([Lesson 32](32-packaging-and-publishing.md)).
- **A private registry proxy** (Verdaccio, Artifactory, Nexus) caches what you use, survives unpublishes, and can quarantine new versions.
- **Scoped internal names** (`@ledger/*`) with the scope pinned to your private registry in `.npmrc`. Otherwise an attacker can publish a public package with your internal package's name and a higher version, and a misconfigured install picks it up (the *dependency confusion* pattern).
- **`npx some-tool`** downloads and runs the latest version of whatever that name points to today; typosquats live one letter away. Pin tools as devDependencies.
- **CI actions are dependencies too.** Pin third-party GitHub Actions to a commit SHA: a tag can be moved to point at new code.

---

## 4. Prototype pollution

### The mechanism

Every ordinary object inherits from `Object.prototype`. `obj.__proto__` is not a normal property: it's an **accessor** on `Object.prototype` that reads and writes an object's prototype. Now combine two facts:

1. `JSON.parse('{"__proto__": {"isAdmin": true}}')` creates an **own, enumerable** property literally named `"__proto__"`. Harmless so far.
2. A recursive merge that copies keys with `target[key] = …` hits the **accessor** when `key === "__proto__"`, and `target.__proto__` *is* `Object.prototype`.

```ts
// ❌ The deep merge every codebase has written once
function merge(target: Record<string, any>, source: Record<string, any>): Record<string, any> {
  for (const key in source) {
    if (typeof source[key] === "object" && source[key] !== null) {
      target[key] ??= {};
      merge(target[key], source[key]);   // key "__proto__" → target["__proto__"] is Object.prototype
    } else {
      target[key] = source[key];
    }
  }
  return target;
}

merge(user, JSON.parse('{"prefs":{"__proto__":{"isAdmin":true}}}'));
({} as { isAdmin?: boolean }).isAdmin;   // true: for EVERY object in the process
```

```text
user.prefs ──► { theme }  ──__proto__──►  Object.prototype  ◄── every other object
                                            isAdmin: true  ← written here
```

### Pollution needs a gadget

Writing `isAdmin` onto `Object.prototype` does nothing by itself. The damage comes from a **gadget**: code that reads a property which is normally *absent* and so falls through to the prototype.

```ts
function canRefund(user: { role: string; isAdmin?: boolean }): boolean {
  return user.isAdmin === true || user.role === "owner";   // 💥 inherited isAdmin: true for everyone
}
```

Options objects are the classic gadgets: `{ ...defaults }` lookups, `options.shell`, template-engine settings. The pollution is **process-wide and persistent**: every request after the attack sees it until the process restarts.

The second route doesn't use `__proto__` at all: `{"constructor":{"prototype":{"isAdmin":true}}}`. `target.constructor` is inherited (`Object`), its `.prototype` is `Object.prototype`, and a merge that recurses into whatever it finds ends up in the same place. That's why blocking only `__proto__` isn't enough.

Query-string parsers that build nested objects from `a[b][c]=1` syntax have had the same bug class, and so have older versions of popular merge utilities. Assume any code that sets properties by **dynamic, attacker-influenced keys** is a candidate.

### Defences

| Defence | Stops | Limit |
|---|---|---|
| **Reject `__proto__`, `constructor`, `prototype` keys at the boundary** | All of it, at the source | Must be on every path that sets properties by dynamic key |
| **`Object.create(null)` or `Map` for dictionaries** | Writes reaching `Object.prototype` through those objects | Only objects you create |
| **Schema validation with `.strict()` objects** ([TS L15](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/15-the-boundary-and-parsing.md)) | Unknown keys on fixed-shape input | Free-form maps such as `metadata` still need explicit key rules |
| **`Object.freeze(Object.prototype)` at boot** | Any write to `Object.prototype` | The "override mistake": assigning `obj.toString = fn` on ordinary objects starts failing (it throws in strict mode); test your dependencies |
| **`node --disable-proto=delete`** (or `=throw`) | The `__proto__` accessor route | **The `constructor.prototype` route still works** |
| **`Object.hasOwn(user, "isAdmin")` in security checks** | Gadgets reading inherited flags | Must be everywhere it matters |

Ledger's `metadata` is the exposed surface: a merchant-controlled map updated with JSON merge-patch semantics ([API L09](https://github.com/Pawar-Pratik/API-mastery/blob/main/02-rest-design/09-writes-patch-and-bulk.md)). The innocent-looking key rule `^[a-zA-Z0-9_]{1,40}$` **accepts `__proto__`**, so the forbidden keys are checked explicitly:

```ts
// domain/merge-patch.ts — RFC 7396 merge patch, pollution-safe
import { ValidationError } from "../errors.ts";             // L09's taxonomy → 422

const FORBIDDEN = new Set(["__proto__", "constructor", "prototype"]);
const MAX_DEPTH = 5;

const isPlainObject = (v: unknown): v is Record<string, unknown> => {
  if (typeof v !== "object" || v === null || Array.isArray(v)) return false;
  const proto = Object.getPrototypeOf(v);
  return proto === Object.prototype || proto === null;
};

export function applyMergePatch(target: unknown, patch: unknown, depth = 0): unknown {
  if (!isPlainObject(patch)) return patch;                       // non-objects replace
  if (depth > MAX_DEPTH) throw new ValidationError("metadata_too_deep");
  const out: Record<string, unknown> = Object.create(null);      // nothing to pollute
  if (isPlainObject(target)) for (const k of Object.keys(target)) out[k] = target[k];
  for (const key of Object.keys(patch)) {                        // own keys only
    if (FORBIDDEN.has(key)) throw new ValidationError("metadata_forbidden_key");
    const value = patch[key];
    if (value === null) delete out[key];                         // null deletes (RFC 7396)
    else out[key] = applyMergePatch(out[key], value, depth + 1);
  }
  return out;
}
```

Throwing on a forbidden key is better than silently skipping it: no legitimate client sends `__proto__`, so its arrival is a signal worth a 422 and a log line.

---

## 5. The other code-level classes, by reference

Each has its own lesson. One paragraph here so the threat model is complete:

- **Command injection.** `exec(\`convert ${file}\`)` hands user input to a shell. Use `execFile`/`spawn` with an argument array, never `shell: true` with input, and remember the Windows `.bat`/`.cmd` rules. → [Lesson 22](../05-concurrency/22-child-processes.md)
- **ReDoS.** A regex with nested quantifiers backtracks exponentially and freezes the only JavaScript thread. Cap input lengths, avoid ambiguous patterns, use a linear-time engine for user-supplied patterns. → [Lesson 08](../02-event-loop/08-blocking-the-loop.md)
- **SSRF.** A webhook URL pointed at `169.254.169.254` or your Redis. Resolve, validate the IP, connect to that IP, re-validate redirects. → [Lesson 18](../04-networking/18-outbound-http.md), and the protocol view in [API L16](https://github.com/Pawar-Pratik/API-mastery/blob/main/03-security/16-owasp-and-hardening.md)
- **Path traversal.** `path.join(base, "../../etc/passwd")`. Resolve, then verify containment with `path.relative`; `realpath` for symlinks. → [Lesson 15](../03-data-and-io/15-the-file-system.md)
- **HTTP hardening** (helmet, CORS, body limits, the OWASP API Top 10) is protocol-level and lives in [API L16](https://github.com/Pawar-Pratik/API-mastery/blob/main/03-security/16-owasp-and-hardening.md).

---

## 6. Strings that become code: `eval`, `new Function`, `vm`

`eval` and `new Function` compile a string in **your** realm with **your** privileges: `process`, the environment, the file system, the network. There is no partial version of that.

**The `vm` module is not a security mechanism, and Node's documentation says so.** It gives code a different global object, not a different security boundary:

```ts
import vm from "node:vm";

const sandbox = {};                      // created in YOUR realm, so its prototype chain is yours
vm.runInNewContext(
  `this.constructor.constructor("return process")().env.DATABASE_URL`,
  sandbox,
);
// → your production database URL: Object → Function → a function compiled in the host realm
```

`vm2`, the popular wrapper that promised a real sandbox, was discontinued after repeated escapes. When Ledger merchants ask for custom rules ("only send webhooks where `amount_minor > 10000`"), the options, in order of preference:

1. **A restricted expression language you parse yourself** (a JSON rule format or an expression grammar evaluated by your own interpreter). Data in, boolean out; nothing to escape from.
2. **A separate isolate** (`isolated-vm`): a different V8 heap, but the same process, so a V8 bug is the boundary.
3. **A separate process or container with OS-level isolation** for genuinely arbitrary code.

`node --disallow-code-generation-from-strings` makes `eval` and `new Function` throw. It's good defence in depth, but test it first: schema compilers such as ajv and fast-json-stringify generate code with `new Function` at runtime.

---

## 7. Secrets and randomness

**Where secrets leak from a Node process**, and the control for each:

| Leak path | Control |
|---|---|
| Committed `.env` files, secrets baked into images | `.env` in `.gitignore`; inject at runtime from a secret manager; secret scanning in CI |
| Log lines | Redaction plus allowlisted serializers, with a sentinel test ([Lesson 29](../07-production/29-observability.md)) |
| Diagnostic reports | They include the environment: restricted storage only ([L29](../07-production/29-observability.md)) |
| Child processes | They **inherit `process.env` by default**: pass an explicit `env` to `spawn` ([L22](../05-concurrency/22-child-processes.md)) |
| Published packages | Source maps with embedded sources, stray `.env` files: a `files` allowlist ([L32](32-packaging-and-publishing.md)) |
| `docker inspect`, `/proc/<pid>/environ` | Least-privilege access to hosts and the container runtime |

**Randomness.** V8's `Math.random` is xorshift128+: fast, statistically fine, and **not a cryptographic generator**. Researchers have shown its internal state can be recovered from a handful of outputs, after which every future "random" token is predictable. Tokens, secrets, IDs that grant access and webhook signing secrets come from the OS CSPRNG:

```ts
import { createHash, randomBytes, randomUUID, timingSafeEqual } from "node:crypto";

const bad = Math.random().toString(36).slice(2);              // ❌ predictable, variable length
const secret = `whsec_${randomBytes(32).toString("base64url")}`; // ✅ 256 bits
const requestId = randomUUID();                               // ✅ v4, CSPRNG-backed

// Constant-time comparison needs equal lengths, so compare fixed-size digests (L12)
export function safeEqual(a: string, b: string): boolean {
  const da = createHash("sha256").update(a).digest();
  const db = createHash("sha256").update(b).digest();
  return timingSafeEqual(da, db);
}
```

---

## 8. The Node permission model

```bash
node --permission \
  --allow-fs-read=/app --allow-fs-read=/data/settlements \
  --allow-fs-write=/data/reports \
  --allow-worker \
  apps/workers/reconciler.ts
```

With `--permission`, a process may touch only what you granted: file-system reads and writes (`--allow-fs-read`, `--allow-fs-write`), child processes (`--allow-child-process`), worker threads (`--allow-worker`), native addons (`--allow-addons`) and WASI (`--allow-wasi`). Anything else throws:

```text
Error: ... { code: 'ERR_ACCESS_DENIED', permission: 'FileSystemWrite', resource: '/tmp/x' }   (abridged)
```

Code can check before acting with `process.permission.has("fs.write", "/data/reports")`. In Node 24 the flag is `--permission`; earlier lines spelled it `--experimental-permission`. Paths should be absolute; check your version's docs for wildcard and relative-path rules.

**What it protects against:** your own code, or a dependency, *opportunistically* reaching outside the process's job. A compromised package in the reconciler that tries to read `~/.npmrc`, write a cron entry or spawn a shell gets `ERR_ACCESS_DENIED` instead of succeeding.

**What it doesn't:**
- **It's a seat belt, not a sandbox.** Node's docs describe it as protection for *trusted* code against unintended access, with no guarantees against malicious code that sets out to bypass it.
- **Granting addons or child processes grants everything**: native code and child processes aren't bound by it.
- **Install time is untouched**: lifecycle scripts run under npm, not under your flags.
- **Don't count on it for network egress**: use network policy for that.
- The docs list further caveats (symbolic links, already-open file descriptors). Read them for your version.

Use it where the grant list is short and stable, which describes batch workers far better than an API server.

---

## 9. Staying patched

- **Run a supported LTS line.** Node security releases are announced ahead of time and ship for every supported line together; **end-of-life lines (18 since April 2025, 20 since April 2026) get nothing**, whatever the CVE. The upgrade policy is in [Lesson 33](33-shipping-node.md).
- **Triage `npm audit` by reachability.** `npm audit --omit=dev` shows the runtime surface. For each finding ask one question: *can attacker-controlled input reach the vulnerable code?* A ReDoS in a build tool's CLI argument parser can't; a prototype pollution in your query parser can.
- **Force a patched transitive version** with `overrides` when the direct dependency is slow to release ([L32](32-packaging-and-publishing.md)), and remove the override once it catches up.

---

## 10. Ledger's supply-chain policy

```yaml
# .github/workflows/ci.yml — the supply-chain steps
permissions:
  contents: read                          # least privilege by default
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4         # pin third-party actions to a SHA in practice
      - uses: actions/setup-node@v4
        with: { node-version-file: .nvmrc }
      - run: npm ci --ignore-scripts      # exact lockfile, hash-verified, no install scripts
      - run: npm rebuild argon2           # the reviewed allowlist
      - run: npm audit signatures         # registry signatures + provenance
      - run: npm audit --omit=dev --audit-level=high
      - run: npm test
```

```jsonc
// renovate.json
{
  "extends": ["config:recommended"],
  "minimumReleaseAge": "7 days"           // no version younger than a week; fast-path advisories by hand
}
```

The rest of `docs/security.md`: lockfile diffs are reviewed like code (new transitive packages called out in the PR); the SDK is published with provenance via trusted publishing and has no runtime dependencies; `metadata` patches go through `applyMergePatch`; the reconciler runs under `--permission` with the grants in §8. When the workspace moves to pnpm in [Lesson 32](32-packaging-and-publishing.md), the allowlist becomes `onlyBuiltDependencies`, and the policy is unchanged.

---

## 11. Production rules

| Rule | Why |
|---|---|
| **`npm ci` in CI and Docker, never `npm install`** | Exact, hash-verified, fails on drift |
| **Review lockfile diffs; flag new transitive packages** | event-stream arrived as a new dependency inside an accepted range |
| **Install scripts off; an explicit, reviewed allowlist** | Install-time payloads run with your CI's secrets |
| **A release-age cooldown, with a manual fast path for advisories** | Most takeovers are pulled before the cooldown ends |
| **Publish with provenance via trusted publishing; no long-lived tokens** | Nothing for a worm to steal; anomalies become detectable |
| **Pin CI actions to SHAs; least-privilege `permissions`** | CI is part of the supply chain |
| **Reject `__proto__`/`constructor`/`prototype` keys at the boundary** | Blocking only `__proto__` leaves the `constructor.prototype` route |
| **`Object.create(null)` or `Map` for dynamic-key dictionaries** | No prototype to pollute |
| **Never `vm` for untrusted code; never `eval` input** | `vm` isn't a boundary; `eval` runs with your privileges |
| **CSPRNG for every token; `timingSafeEqual` on equal-length digests** | `Math.random` is predictable; string `===` leaks timing |
| **Explicit `env` for child processes** | Children inherit every secret by default |
| **`--permission` for workers with short, stable grant lists** | A seat belt that turns opportunistic access into an error |
| **Stay on a supported LTS; triage audit findings by reachability** | EOL lines get no fixes; unreachable findings are noise |

> **Spring equivalent:** `dependency-check-maven` failing the build on high CVSS, Dependabot, and the dependency-incident analogue: Log4Shell was a dependency you didn't write. See [Spring Boot Lesson 28](https://github.com/Pawar-Pratik/SpringBoot/blob/main/07-security/28-authorization-and-hardening.md). The difference that matters: Maven doesn't run arbitrary dependency code at download time; npm does unless you stop it.

---

## 12. Interview traps

**Q1. "A dependency was compromised on npm overnight. What protected you?"**
Answer in layers, not with one tool. The lockfile and `npm ci` mean nothing new entered a build without a reviewed diff; install scripts are off, so an install-time payload didn't run; a release-age cooldown means automated updates hadn't proposed the version yet; signatures and provenance checks flag a version published outside the usual workflow. Then the response: search lockfiles across repos for the affected versions, and **rotate every secret that was reachable** from any machine that installed it (CI tokens, cloud credentials, env vars), because you assume exfiltration.
**Follow-up:** "Would `npm audit` have caught it?" Not on day one: it reports published advisories, and a fresh compromise has none.

**Q2. "What is prototype pollution?"**
Attacker-controlled keys like `__proto__` reach code that sets properties by dynamic key, typically a deep merge or a nested query parser, and the write lands on `Object.prototype`, so every object in the process inherits it. It needs a gadget to matter: code reading a normally-absent property such as `user.isAdmin` or `options.shell`. Fix at the boundary by rejecting `__proto__`, `constructor` and `prototype`, and use null-prototype objects or `Map` for dictionaries.
**Follow-up:** "Does `--disable-proto` fix it?" Only the `__proto__` accessor route; `{"constructor":{"prototype":…}}` still reaches `Object.prototype`.

**Q3. "Is `vm` a sandbox?"**
No, and the Node docs say so. It gives code a different global object in the same process, and objects passed in carry your realm's prototype chain: `this.constructor.constructor("return process")()` escapes to the host's `process`. For untrusted logic, parse a restricted expression language, or isolate in a separate process with OS-level controls.

**Q4. "What does `npm ci` do differently from `npm install`?"**
It installs exactly what the lockfile records and verifies integrity hashes, fails if `package.json` and the lockfile disagree, removes `node_modules` first, and never writes the lockfile. `npm install` resolves ranges and may update the lockfile, which makes a build depend on what the registry served that minute.

**Q5. "What is package provenance, and what doesn't it prove?"**
A signed attestation linking a published tarball to the source commit, repository and CI workflow that built it. It makes a token stolen and used from a laptop detectable, because that version lacks the provenance its predecessors had. It doesn't prove the code is safe: a compromised repository or workflow produces valid provenance for malicious code.

**Q6. "What does Node's permission model protect against?"**
Trusted code, including dependencies, reaching outside its declared needs: file reads and writes, child processes, workers, addons. It turns opportunistic access into `ERR_ACCESS_DENIED`. It isn't a sandbox for malicious code, granting addons or child processes grants everything, and it does nothing at install time. Defence in depth for a worker with a short grant list.

**Q7. "Why not `Math.random()` for tokens?"**
It's a fast statistical generator (xorshift128+ in V8) whose state can be recovered from a few outputs, so future values are predictable; its string forms also have variable, low entropy. Use `crypto.randomBytes` or `randomUUID`, and compare secrets with `timingSafeEqual` on fixed-length digests.

**Q8. "`npm audit` reports 37 vulnerabilities. What do you do?"**
Don't run `npm audit fix --force` and hope. Scope to runtime with `--omit=dev`, then triage each by reachability: does attacker input reach this code path? Fix the reachable ones by upgrading the direct dependency, or with an `overrides` pin for a transitive one, and record why the rest are accepted. The weak answer treats the count as the risk.

**Q9. "Merchants want custom webhook filter rules. How do you run them safely?"**
Never `eval` and never `vm`. Define a small rule language (field, operator, value), validate it with a schema, and evaluate it with your own interpreter, so merchant input is data. Bound its cost too: rule count, nesting depth, string lengths, and no regex operator unless it's linear-time.

**Q10. "Where can secrets leak from a Node process?"**
Logs, diagnostic reports (they embed the environment), child processes (they inherit `process.env`), error messages that echo connection strings, published packages (stray `.env` files, source maps with embedded sources), and host access such as `docker inspect`. Each has a specific control, and naming them is what makes the answer concrete.

---

## 13. Build & break

### Build — `docs/security.md` and the CI steps
Write Ledger's policy from §10 as a document a reviewer can check: lockfile rules, the install-script allowlist with a reason per entry, the Renovate cooldown, provenance for the SDK, secret handling, the permission flags per worker. Wire the CI steps.

### Build — `applyMergePatch` and its tests
Implement §4's function and test it: `__proto__`, `constructor`, `prototype` at depth 0 and depth 3 are rejected with 422; `null` deletes; arrays replace; depth 6 is rejected; and after every malicious attempt, `({} as Record<string, unknown>).isAdmin === undefined`.

### Break — four experiments
1. **Flip an authorization check with a JSON body.** A tiny Express app with `PATCH /me` that calls the naive `merge` from §4, and `GET /admin` guarded by `canRefund`-style logic. Send `curl.exe -X PATCH localhost:3000/me -H "content-type: application/json" -d '{"prefs":{"__proto__":{"isAdmin":true}}}'` as user A, then call `/admin` as user B: **200**, for every user, until restart. Restart with `--disable-proto=delete` and repeat: blocked. Now send `{"prefs":{"constructor":{"prototype":{"isAdmin":true}}}}`: **200 again.** Replace `merge` with `applyMergePatch`: both rejected.
2. **An install script runs your code.** Make `evil-pkg/` with `"scripts": { "postinstall": "node postinstall.cjs" }`, where `postinstall.cjs` writes `ledger-postinstall-was-here.txt` to `os.homedir()` containing only `Object.keys(process.env).length` (a count, never values). `npm pack` it, then `npm install ../evil-pkg/evil-pkg-1.0.0.tgz` in a scratch project: the file appears in your home directory, written with your full environment in reach. Delete it, reinstall with `--ignore-scripts`: nothing. Try `pnpm add` with pnpm 10 and read what it reports skipping.
3. **The permission model says no.** Run the reconciler with §8's flags (simplest in WSL2, where the paths match the container's). Add a debug line that writes to `os.tmpdir()`: `ERR_ACCESS_DENIED`. Remove `--allow-worker`: the worker pool fails to start. Put both back and confirm `process.permission.has("fs.write", "/data/reports")` is `true`.
4. **`Math.random` tokens collide.** Generate 1,000,000 tokens with `Math.random().toString(36).slice(2, 8)` (six base-36 characters, about 31 bits) and count duplicates in a `Set`. The birthday bound predicts roughly n²/2N = 10¹² / (2 × 36⁶) ≈ 230; compare with what you get. Repeat with `randomBytes(16).toString("base64url")`: zero. Collisions are the visible problem; predictability is the invisible one.

### Explain out loud (60–120 seconds)
1. What a dependency can do at install, import and run time.
2. The six layers, and which incident each would have stopped.
3. Prototype pollution: the accessor, the gadget, both routes, and the boundary fix.
4. Why `vm` isn't a sandbox and what you'd use instead.
5. What the permission model is for, and its three biggest limits.

---

## What's next

Your dependencies are pinned, your merges are safe and your secrets stay put. Next: proving your code works, specifically the parts that are hard to test in Node: timers, events, streams, shutdown, and test runners that never exit because something is still holding the loop open.

Next → **[Lesson 31: Testing Node](31-testing-node.md)**
