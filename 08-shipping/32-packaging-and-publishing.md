# Lesson 32 — Packages: exports, publishing & monorepos

> **Why this lesson exists:** consuming packages is daily work, and it hides how little most people know about `package.json`. Publishing one correctly exposes all of it: types that resolve for every consumer, both module systems happy, no deep imports that break on a minor release, no `.env` in the tarball. Ledger now ships two artefacts other developers depend on, the `@ledger/node` SDK and the `ledger` CLI, so a mistake here becomes their outage, not yours.

**Time:** ~80 minutes · **Prereq:** Lessons 04, 30

---

## 1. The idea in one sentence

> **`package.json` is a contract read by three different resolvers, namely Node, TypeScript and your package manager, and every publishing bug is one of them reading it differently from how you meant it.**

---

## 2. Why publishing exposes what you never learned

| What the consumer sees | Which resolver disagreed | Section |
|---|---|---|
| "Could not find a declaration file for module '@ledger/node'" | TypeScript walked the `exports` conditions in a different order from Node | §3 |
| `ERR_PACKAGE_PATH_NOT_EXPORTED` after a minor upgrade | Node enforced an `exports` map the consumer's deep import ignored | §3 |
| Works with npm, `ERR_MODULE_NOT_FOUND` with pnpm | The package manager's `node_modules` layout | §6 |
| "Your package published our `.env`" | npm's file-selection rules | §3, §7 |
| `^0.2.3` never picks up `0.3.0` | semver's special case for `0.x` | §4 |

[TypeScript Lesson 14](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/14-modules-and-declarations.md) covers how TypeScript resolves types and why `types` comes first in a conditions block. This lesson is the Node and npm side, and the whole publishing workflow.

---

## 3. The fields that matter

### `exports`: the gate and the switchboard

```jsonc
// packages/ledger-node/package.json
{
  "name": "@ledger/node",
  "version": "1.2.0",
  "type": "module",
  "exports": {
    ".":          { "types": "./dist/index.d.ts",    "default": "./dist/index.js" },
    "./webhooks": { "types": "./dist/webhooks.d.ts", "default": "./dist/webhooks.js" },
    "./package.json": "./package.json"
  },
  "files": ["dist", "src"],
  "engines": { "node": ">=22.12" },
  "sideEffects": false
}
```

**It's a gate.** Once `exports` exists, **only the listed subpaths exist.** `import "@ledger/node/dist/client.js"` throws `ERR_PACKAGE_PATH_NOT_EXPORTED`, even though the file is right there. That's the point: your internal file layout stops being public API, so you can refactor it in a patch release. Subpath patterns (`"./resources/*": "./dist/resources/*.js"`) expose a family; a `null` target (`"./internal/*": null`) blocks one explicitly.

**It's a switchboard.** Each subpath maps to a *conditions object*, and resolution works like this:

```text
import { verify } from "@ledger/node/webhooks"
  1. find node_modules/@ledger/node/package.json
  2. "exports" exists → only listed subpaths are reachable; no fallback to raw files
  3. match "./webhooks" → { "types": …, "default": … }
  4. walk the keys IN OBJECT ORDER; the first key in the resolver's active set wins
       Node's set:        node, import | require, default   (+ any --conditions you pass)
       TypeScript's set:  types, import | require, node, default
  5. Node → ./dist/webhooks.js           TypeScript → ./dist/webhooks.d.ts
```

Three rules fall straight out of step 4:

- **`types` first.** If `import` or `default` comes first, TypeScript matches it, lands on the `.js` file, and looks for a `.d.ts` *next to it*. When your declarations happen to sit beside the JavaScript, that works by accident; move them to `dist/types/` and every consumer gets an untyped module.
- **`default` last.** It matches everything, so anything after it is dead.
- **For an ESM-only package that CJS consumers can `require()`, use `default`, not `import`.** A `require()` doesn't match the `import` condition; with only `import` listed, CJS consumers get `ERR_PACKAGE_PATH_NOT_EXPORTED` instead of `require(esm)`.

One more property worth knowing: a package with `exports` can **import itself by name**, so `import { Ledger } from "@ledger/node"` inside its own test suite exercises the real exports map rather than a relative path.

### `imports`: private `#` aliases that Node understands

Type stripping ignores `tsconfig.json`, so `paths` aliases don't exist at runtime ([Lesson 03](../01-foundations/03-the-process.md)). The `imports` field is the runtime-native replacement, scoped to one package and always prefixed with `#`:

```jsonc
// apps/api/package.json: an app that runs .ts natively
{ "imports": { "#lib/*": "./src/lib/*" } }
```

```ts
import { pool } from "#lib/db.ts";       // Node and TypeScript (nodenext) both resolve it
```

Use it in apps. Inside a published library, plain relative imports keep the build simple.

### The rest of the fields

| Field | What it does | The trap |
|---|---|---|
| `type: "module"` | `.js` files in the package are ESM ([L04](../01-foundations/04-modules-cjs-and-esm.md)) | Omit it and your `.js` output is treated as CommonJS |
| `bin` | `{ "ledger": "./dist/cli.js" }`: npm links a command on install, with a `.cmd`/PowerShell shim on Windows | The file needs `#!/usr/bin/env node` as its first line |
| `files` | **An allowlist** of what goes in the tarball (`package.json`, README and LICENSE are always included) | Without it, npm falls back to `.npmignore`, then `.gitignore` (below) |
| `engines` | Declares supported Node versions | **Advisory**: npm warns, it doesn't refuse, unless the consumer enables `engine-strict` |
| `sideEffects: false` | Tells **bundlers** that unused modules can be dropped | Node ignores it entirely |
| `private: true` | npm refuses to publish | Put it on every workspace root and app |

**The `.npmignore` trap.** With no `files` field and no `.npmignore`, npm excludes what `.gitignore` excludes, so a gitignored `.env` stays out. **The moment someone adds an `.npmignore`** (say, to exclude `src/`), **npm stops reading `.gitignore`**, and `.env` ships. A `files` allowlist fails closed; ignore files fail open. That's the same positive-list argument as output DTOs in [API L16](https://github.com/Pawar-Pratik/API-mastery/blob/main/03-security/16-owasp-and-hardening.md).

### Dependency kinds

| Field | Installed for your consumers? | Use it for |
|---|---|---|
| `dependencies` | Yes | What your runtime code imports |
| `devDependencies` | No | Build, test and lint tools |
| `peerDependencies` | Provided by the host app (npm 7+ installs missing peers) | Plugins that must share **one copy** of the host: a future `@ledger/express` middleware peers on `express`, because two copies of Express means two `Router` classes and broken `instanceof` |
| `optionalDependencies` | Yes, and an install failure is tolerated | Platform-specific binaries selected by `os`/`cpu` fields |
| `overrides` (npm), `pnpm.overrides` | Root project only | Forcing a patched transitive version ([L30](30-security-and-supply-chain.md)) |

Every runtime dependency of an SDK becomes a dependency of every merchant who installs it: one more maintainer account in *their* supply chain. **`@ledger/node` has zero runtime dependencies**: `fetch`, `crypto` and `AbortSignal` are built in. Its public types take `Uint8Array | string` rather than `Buffer`, so the `.d.ts` files don't require `@types/node` either (a `Buffer` *is* a `Uint8Array`, so Node callers pass one unchanged).

---

## 4. semver and ranges

| Range | Means | Accepts from 1.2.3 |
|---|---|---|
| `1.2.3` | Exactly that version | Nothing else |
| `~1.2.3` | `>=1.2.3 <1.3.0` | Patches |
| `^1.2.3` | `>=1.2.3 <2.0.0` | Minors and patches |
| **`^0.2.3`** | **`>=0.2.3 <0.3.0`** | **Patches only** |
| **`^0.0.3`** | **`>=0.0.3 <0.0.4`** | **Only 0.0.3** |
| `*`, `latest` | Anything | Never use in a real project |

**Why `0.x` is special:** semver declares `0.y.z` "initial development: anything may change", so the caret treats the **leftmost non-zero** component as the breaking one. `^0.2.3` accepts `0.2.9` and refuses `0.3.0`, because `0.3.0` is allowed to break you. That's why your `0.x` dependency "never updates": it's working as designed.

Pre-releases have their own rule: `1.3.0-beta.1` doesn't satisfy `^1.2.3`. A range only matches a pre-release if one of its comparators names a pre-release on the same `major.minor.patch`, so nobody gets a beta by accident.

Check any range without guessing: `npx semver -r "^0.2.3" 0.2.3 0.2.9 0.3.0 1.0.0` prints the versions that match.

---

## 5. Lockfiles: what they pin, and who reads them

`package-lock.json` records, for **every** package in the tree, the exact version, the tarball URL and an **integrity hash**. `npm ci` installs exactly that and verifies every hash ([L30](30-security-and-supply-chain.md)). Two consequences people miss:

- **Your library's lockfile is not published, and consumers never read it.** They resolve your *ranges* against the registry on the day they install. So your CI, which installs from your lockfile, tests transitive versions your consumers may never get, and misses the ones they will. Add a CI job that installs without the lockfile (`npm install --no-package-lock`) so you see what a fresh consumer sees.
- **`npm-shrinkwrap.json` is the exception**: it *is* published and honoured. It's for applications distributed through npm (a globally installed CLI); never for libraries, because it blocks deduplication and stops consumers from receiving patches.

---

## 6. npm vs pnpm vs yarn: what `node_modules` looks like

```text
npm (hoisted, flat)                       pnpm (isolated, symlinked)
node_modules/                             node_modules/
├── pg/                                   ├── pg ──► .pnpm/pg@8.x/node_modules/pg
├── pg-types/   ← pg's dependency,        └── .pnpm/
├── pg-pool/       hoisted to the top:        ├── pg@8.x/node_modules/
└── …              YOU can import it           │   ├── pg/          (hard links into the store)
                   without declaring it         │   ├── pg-types ──► ../../pg-types@…/…
                                               │   └── pg-pool  ──► …
                                               └── pg-types@…/node_modules/pg-types/
```

- **npm** hoists everything it can to the top level, so any package can resolve any other.
- **pnpm** keeps one copy of every file in a global **content-addressable store** and hard-links it into projects (fast installs, little disk). The top level holds **only your declared dependencies**; each package sees only *its* declared dependencies through symlinks. pnpm 10 also skips dependencies' install scripts by default ([L30](30-security-and-supply-chain.md)).
- **yarn** (Berry) can skip `node_modules` altogether with Plug'n'Play, a generated resolution map that's similarly strict.

**A phantom dependency** is a package you import but never declared, which works only because npm hoisted it. The Ledger-flavoured example: `import types from "pg-types"` to register a bigint type parser ([L25](../06-data-and-jobs/25-databases-from-node.md)). It works under npm because `pg` depends on `pg-types`; it fails under pnpm with `ERR_MODULE_NOT_FOUND`; and it breaks under npm too the day `pg` changes its own dependencies. The fix is `pg.types.setTypeParser(...)` (pg re-exports it) or declaring the dependency. **pnpm's strictness is a feature: it turns a latent bug into an install-time error.**

---

## 7. Publishing a TypeScript library

### Ship JavaScript, because Node won't strip types in `node_modules`

Node's type stripping refuses files under `node_modules` (`ERR_UNSUPPORTED_NODE_MODULES_TYPE_STRIPPING`), and your consumers may run bundlers or other runtimes anyway. So the Engine runs `.ts` natively, while the packages build to `.js` + `.d.ts`:

```jsonc
// packages/ledger-node/tsconfig.build.json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "noEmit": false,
    "rootDir": "src",
    "outDir": "dist",
    "declaration": true,
    "declarationMap": true,                    // go-to-definition lands in src/ (which we publish)
    "allowImportingTsExtensions": false,       // that option is for noEmit builds
    "rewriteRelativeImportExtensions": true    // TS 5.7: "./client.ts" is emitted as "./client.js"
  },
  "include": ["src"]
}
```

Source keeps the `.ts` specifiers Node needs when running it directly; the emitted JavaScript gets the `.js` specifiers consumers need. The rewrite applies to **relative** imports only. Tests live in `test/`, outside `src/`, so `"files": ["dist", "src"]` never ships them. Publishing `src/` makes declaration maps useful, and it also means **anything in your source ships**: comments, internal URLs, a test key someone hard-coded. Review the file list, every release.

### ESM-only or dual, in 2026

| | **ESM-only** | Dual (separate ESM and CJS builds) |
|---|---|---|
| CJS consumers | `require()` works on every supported line (22.12+, 24, 26) through `require(esm)`, provided your graph has **no top-level await** (`ERR_REQUIRE_ASYNC_MODULE`) | Native |
| Dual-package hazard | Impossible: one copy | Two copies can load, so `instanceof` checks and singletons break ([L04](../01-foundations/04-modules-cjs-and-esm.md)) |
| Build | One emit | Two emits, two sets of declarations, a condition per format |
| Who's left out | Node 20 (end-of-life) and consumers on TypeScript's old `node10` resolution | Nobody, at the cost of complexity and the hazard |

**Ledger publishes ESM-only, with `engines: >=22.12`.** TypeScript 5.8+ understands `require()` of ESM under `module: nodenext`, so typed CJS consumers are covered too. And never use top-level await in a library entry point: it's the one thing that turns `require(esm)` into an error.

### Verify before you publish

```bash
npm pack --dry-run                    # the exact file list and size that would ship
npx publint                           # package.json lint: exports targets exist, formats match, files cover them
npx @arethetypeswrong/cli --pack .    # how TypeScript resolves your types under each resolution mode
```

`npm pack --dry-run` is the one that catches the `.env`. `attw` checks the node10, node16 (CJS and ESM) and bundler views of your types; tell it the package is ESM-only (it has a profile for that) so it doesn't flag the view you chose not to support. Run all three in CI.

---

## 8. Versioning and releasing

**Changesets** keeps versioning honest in a multi-package repo:

```bash
pnpm changeset            # choose packages and bump types, write a summary → .changeset/*.md (committed with the PR)
pnpm changeset version    # apply them: bump versions, update dependents, write CHANGELOG.md
pnpm changeset publish    # publish every package whose new version isn't on the registry yet, and tag it
```

The bump decision is made by the person who wrote the change, in the PR where a reviewer can challenge it, not by someone guessing at release time.

```yaml
# .github/workflows/release.yml
on: { push: { branches: [main] } }
permissions:
  contents: write          # the version PR and tags
  pull-requests: write
  id-token: write          # OIDC: trusted publishing and provenance, no NPM_TOKEN secret
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version-file: .nvmrc, registry-url: "https://registry.npmjs.org" }
      - run: pnpm install --frozen-lockfile
      - run: pnpm -r build && pnpm -r test && pnpm -r run verify    # verify = pack --dry-run + publint + attw
      - uses: changesets/action@v1
        with: { publish: pnpm changeset publish }
```

Trusted publishing ([L30](30-security-and-supply-chain.md)) needs two things: the package configured on npmjs.com to trust this repository and workflow, and a publishing client recent enough to perform the OIDC exchange. Support landed in npm, pnpm and Changesets at different times, so check the current docs for the versions you pin. The scoped package's first publish also needs `--access public` (or `publishConfig.access`), since scoped packages default to restricted.

---

## 9. Workspaces and monorepos

```yaml
# pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"
```

```jsonc
// packages/ledger-cli/package.json
{ "dependencies": { "@ledger/node": "workspace:^" } }   // a symlink locally; "^1.2.0" in the published tarball
```

- The `workspace:` protocol links the local package and **is rewritten to a real range when pnpm packs or publishes**. Publishing with a tool that doesn't rewrite it ships a dependency nobody can install.
- `pnpm -r build` runs in **topological order**: `ledger-node` builds before `ledger-cli`. `pnpm --filter @ledger/node test` targets one package.
- npm workspaces do the linking too, with ordinary version ranges instead of a protocol.
- **Turborepo** and **Nx** add a task graph with caching keyed by input hashes, so CI rebuilds only what changed. Worth it at a dozen packages; overhead at four.
- Type-checking across packages incrementally is TypeScript's job: project references, in [TS Lesson 18](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/05-ecosystem/18-tooling-and-performance.md).

---

## 10. Ledger: `@ledger/node` and `ledger`

### The SDK

```ts
// packages/ledger-node/src/client.ts
import { randomUUID } from "node:crypto";
import { LedgerError } from "./errors.ts";
import { retryDelayMs, sleep } from "./retry.ts";
import type { Page, Payment, PaymentCreateParams } from "./types.ts";

export type LedgerOptions = { apiKey: string; baseUrl?: string; timeoutMs?: number; maxRetries?: number };
export type RequestOptions = { signal?: AbortSignal; idempotencyKey?: string };
const RETRYABLE = new Set([408, 429, 500, 502, 503, 504]);

export class Ledger {
  readonly payments: Payments;
  readonly #o: Required<LedgerOptions>;

  constructor(options: LedgerOptions) {
    this.#o = { baseUrl: "https://api.ledger.example", timeoutMs: 30_000, maxRetries: 2, ...options };
    this.payments = new Payments(this);
  }

  async request<T>(method: "GET" | "POST", path: string, body?: unknown, opts: RequestOptions = {}): Promise<T> {
    const key = method === "POST" ? (opts.idempotencyKey ?? randomUUID()) : undefined; // ONE key, every attempt
    for (let attempt = 0; ; attempt++) {
      const perAttempt = AbortSignal.timeout(this.#o.timeoutMs);
      const res = await fetch(`${this.#o.baseUrl}${path}`, {
        method,
        signal: opts.signal ? AbortSignal.any([opts.signal, perAttempt]) : perAttempt,
        headers: {
          authorization: `Bearer ${this.#o.apiKey}`,
          "content-type": "application/json",
          ...(key ? { "idempotency-key": key } : {}),
        },
        body: body === undefined ? undefined : JSON.stringify(body),
      });
      if (res.ok) return (await res.json()) as T;
      if (!RETRYABLE.has(res.status) || attempt >= this.#o.maxRetries) throw await LedgerError.from(res);
      await res.body?.cancel();                                  // free the pooled connection (L18)
      await sleep(retryDelayMs(res, attempt), opts.signal);      // Retry-After, else full jitter
    }
  }
}

export class Payments {
  readonly #client: Ledger;
  constructor(client: Ledger) { this.#client = client; }

  create(params: PaymentCreateParams, opts?: RequestOptions): Promise<Payment> {
    return this.#client.request("POST", "/v1/payments", params, opts);
  }

  /** Auto-pagination: callers write `for await` and never see a cursor. */
  async *list(params: { limit?: number } = {}, opts: RequestOptions = {}): AsyncGenerator<Payment, void, undefined> {
    let cursor: string | null = null;
    do {
      const qs = new URLSearchParams({ limit: String(params.limit ?? 100), ...(cursor ? { cursor } : {}) });
      const page = await this.#client.request<Page<Payment>>("GET", `/v1/payments?${qs}`, undefined, opts);
      yield* page.data;
      cursor = page.has_more ? page.next_cursor : null;
    } while (cursor);
  }
}
```

The decisions a reviewer should see: **the idempotency key is generated once per logical call and reused on every retry**, which is what makes retrying a `POST` safe ([API L18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md)); 409 and other 4xx responses are never retried; the response body is cancelled before a retry; every call takes an `AbortSignal`. Network failures (where `fetch` rejects) retry under the same key, in a `try/catch` omitted here. `for await (const p of ledger.payments.list())` streams all 4 million payments in pages, in bounded memory ([TS L16](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/16-async-and-errors.md)).

`webhooks.verify(rawBody, header, secret)` lives at the `./webhooks` subpath: it parses `Ledger-Signature: t=…,v1=…`, rejects stale timestamps, computes HMAC-SHA256 over `` `${t}.` `` plus the raw bytes, compares each `v1` with `timingSafeEqual` after checking lengths, and **only then** parses the JSON. That's L12's `hmac.ts`, shipped to merchants.

### The CLI

```ts
#!/usr/bin/env node
// packages/ledger-cli/src/cli.ts   →   "bin": { "ledger": "./dist/cli.js" }
import { parseArgs } from "node:util";
import { randomBytes } from "node:crypto";
import { listen } from "./listen.ts";

async function main(): Promise<number> {
  const { positionals, values } = parseArgs({
    allowPositionals: true,
    options: {
      "forward-to": { type: "string" },
      "api-key":    { type: "string" },
      events:       { type: "string", multiple: true },
      help:         { type: "boolean", short: "h" },
    },
  });
  const forwardTo = values["forward-to"];
  const apiKey = values["api-key"] ?? process.env.LEDGER_API_KEY;
  if (values.help || positionals[0] !== "listen" || !forwardTo || !apiKey) {
    console.error("usage: ledger listen --forward-to <url> [--events payment.*]  (LEDGER_API_KEY or --api-key)");
    return values.help ? 0 : 2;
  }
  const secret = `whsec_${randomBytes(24).toString("base64url")}`;
  console.error(`Forwarding events to ${forwardTo}. Your local signing secret: ${secret}`);
  const ac = new AbortController();
  process.once("SIGINT", () => ac.abort());
  await listen({ apiKey, forwardTo, events: values.events ?? [], secret, signal: ac.signal });
  return 0;
}

process.exitCode = await main();          // top-level await: fine in a bin, never in a library entry
```

`listen()` opens `GET /v1/events/stream` (the SSE feed from [Lesson 20](../04-networking/20-realtime-servers.md)), parses frames with the line splitter from [Lesson 14](../03-data-and-io/14-composing-streams.md), and for each event re-signs the raw bytes with the local secret and POSTs them to `--forward-to` with a 10 s deadline, consuming each response and printing `payment.succeeded → 200`. A merchant developing against `localhost:3000` gets real, correctly signed webhooks without exposing their laptop to the internet.

`tsc` preserves the shebang; npm marks the file executable on install and generates the Windows shims. The CLI's `package.json` has `"type": "module"`, `"bin"`, `"files": ["dist"]`, `"engines"`, and depends on `@ledger/node` via `workspace:^`.

---

## 11. Production rules

| Rule | Why |
|---|---|
| **Always ship an `exports` map** | Internal files stop being public API; refactors stay patch releases |
| **`types` first, `default` last in every conditions object** | First match wins, and TypeScript and Node walk the same object |
| **ESM-only packages use `default`, not just `import`** | `require(esm)` never matches the `import` condition |
| **A `files` allowlist; never rely on `.npmignore`** | Adding `.npmignore` silently stops npm reading `.gitignore` |
| **`npm pack --dry-run`, publint and attw in CI** | They catch the `.env`, broken targets and wrong types before consumers do |
| **Ship `.js` + `.d.ts`; `rewriteRelativeImportExtensions` from `.ts` sources** | Node won't strip types under `node_modules` |
| **ESM-only in 2026; no top-level await in library entries** | Every supported line can `require(esm)`, except when TLA is in the graph |
| **Zero or minimal runtime dependencies in an SDK** | Each one joins every consumer's supply chain |
| **`peerDependencies` for anything that must be a single shared copy** | Duplicates break `instanceof` and singletons |
| **Declare every package you import; use pnpm to enforce it** | Phantom dependencies break when someone else's tree changes |
| **Test against fresh resolution, not only your lockfile** | Consumers never see your lockfile |
| **Changesets for bumps; provenance via trusted publishing for releases** | Bumps decided in review; origin verifiable without long-lived tokens |
| **`private: true` on workspace roots and apps** | Nobody publishes the monorepo by accident |

> **Spring equivalent:** Maven coordinates (`groupId:artifactId:version`) are `@scope/name@version`; Maven resolves exact versions with "nearest wins" and rarely uses ranges, where npm resolves semver ranges and can nest duplicate copies; a BOM curates the versions you *declare*, while a lockfile freezes the tree you *resolved*. See [Spring Boot Lesson 01](https://github.com/Pawar-Pratik/SpringBoot/blob/main/01-java-for-spring/01-jvm-build-and-classpath.md).

---

## 12. Interview traps

**Q1. "What does the `exports` field do?"**
Two jobs. It's a gate: once present, only the listed subpaths can be imported, and anything else throws `ERR_PACKAGE_PATH_NOT_EXPORTED`, which makes the internal layout private. And it's a switchboard: each subpath maps to conditions (`types`, `import`, `require`, `node`, `default`, custom ones) that pick a different target per resolver and module system, walked in object order with the first match winning.

**Q2. "Why must `types` come first?"**
Because TypeScript walks the same conditions object in order and stops at the first key it recognises. If `import` or `default` comes first, it resolves to the JavaScript file and looks for a declaration beside it. That can work by accident when `.d.ts` files sit next to the `.js`, which is why the bug survives until someone moves declarations into their own folder.

**Q3. "ESM-only or dual in 2026?"**
ESM-only for new packages. Every supported Node line can `require()` an ES module (22.12+, 24, 26; Node 20 is end-of-life), so CJS consumers are covered without a second build, and you avoid the dual-package hazard entirely. The conditions: point `exports` at `default` so `require` matches, set `engines`, and keep top-level await out of the graph, since it makes `require(esm)` throw `ERR_REQUIRE_ASYNC_MODULE`.

**Q4. "What is a phantom dependency?"**
A package your code imports without declaring, which resolves only because npm hoisted it from someone else's dependency tree. It breaks under pnpm's strict layout, and under npm the day the real owner drops or upgrades it. The fix is to declare it, or use the owner's re-export (`pg.types` instead of `pg-types`).

**Q5. "What does `^0.2.3` match?"**
`>=0.2.3 <0.3.0`: patches only. The caret treats the leftmost non-zero component as the breaking one, because semver says anything may change in `0.x`. `^0.0.3` matches only `0.0.3`. That's why a `0.x` dependency "never updates".

**Q6. "What's in a lockfile, and when is it ignored?"**
Every package in the tree with its exact version, tarball URL and integrity hash; `npm ci` installs exactly that and verifies the hashes. It's ignored by the consumers of a library: lockfiles aren't published, so consumers resolve your ranges fresh, and your lockfile-based CI tests a tree they may never get. `npm-shrinkwrap.json` is the published exception, for CLIs and apps, never libraries.

**Q7. "`peerDependencies`: when and why?"**
When your package plugs into a host that must exist exactly once: an Express middleware, a React component library, a plugin system. Declaring the host as a regular dependency can install a second copy, and two copies means two classes, failed `instanceof` checks and split singletons. As a peer, the app provides the one copy and the package manager checks the range.

**Q8. "How do you publish a TypeScript library that Node can run?"**
Build it: Node refuses to strip types under `node_modules`, so ship `.js`, `.d.ts` and declaration maps. Write `.ts` relative imports in source and let `rewriteRelativeImportExtensions` (TS 5.7) emit `.js` specifiers. Put `types` first in `exports`, allowlist `files`, and gate the release on `npm pack --dry-run`, publint and attw.

**Q9. "What happens to `workspace:^` when you publish?"**
pnpm rewrites it to a real range, such as `^1.2.0`, in the packed tarball, so consumers get an installable dependency while the monorepo links the local copy. Publishing through a tool that doesn't understand the protocol ships `workspace:^` verbatim, and nobody can install the package.

**Q10. "How do you stop a package from shipping secrets?"**
A `files` allowlist, because it fails closed; never `.npmignore`, whose mere existence turns off the `.gitignore` fallback. Then `npm pack --dry-run` in CI, reviewed or diffed against the previous release, plus remembering that published sources and source maps expose whatever the source contains.

**Q11. "Does Node use the `sideEffects` field?"**
No. It's a hint for bundlers that unused modules can be dropped during tree-shaking. Node loads whatever you import.

---

## 13. Build & break

### Build — the two packages
Turn the repo into a pnpm workspace (`apps/api`, `apps/workers`, `packages/ledger-node`, `packages/ledger-cli`, `packages/shared`). Implement `@ledger/node` (client, payments with `create` and auto-paginating `list`, `webhooks.verify` at `./webhooks`, `AbortSignal` everywhere, zero runtime dependencies) and `ledger` (`bin`, shebang, `parseArgs`, `listen --forward-to`). Build both with `tsc -p tsconfig.build.json`. Add a `verify` script to each: `npm pack --dry-run && publint && attw --pack .`. Add a changeset and run `changeset version` locally to see what it writes.

### Break — five experiments
1. **`types` last.** Move the declarations to `dist/types/` and write `".": { "default": "./dist/index.js", "types": "./dist/types/index.d.ts" }`. In a consumer project with `moduleResolution: nodenext` and `strict`, run `npx tsc --noEmit`: "Could not find a declaration file for module '@ledger/node'". Swap the order: clean. Run attw on both versions and read how it reports the first.
2. **The blocked deep import.** In the consumer, `import { verify } from "@ledger/node/dist/webhooks.js"`. Read the `ERR_PACKAGE_PATH_NOT_EXPORTED` message. Fix it by importing the `./webhooks` subpath, not by widening `exports`.
3. **The phantom dependency.** In an npm project, `npm install pg` and add `import types from "pg-types"` to register a parser: it runs. Convert the project to pnpm (`pnpm import` reads the npm lockfile, then `pnpm install`) and run it again: `ERR_MODULE_NOT_FOUND`. Fix it with `pg.types.setTypeParser`.
4. **`^0.2.3` and `0.3.0`.** Run `npx semver -r "^0.2.3" 0.2.3 0.2.9 0.3.0` and see `0.3.0` missing. Then find a real `0.x` package (`npm view <name> versions`), depend on `^0.<minor>.0`, and watch `npm install` stay inside that minor even when a newer one exists.
5. **The `.env` that shipped.** In `packages/ledger-node`, create `.env` (gitignored), delete the `files` field, and run `npm pack --dry-run`: `.env` absent, thanks to the `.gitignore` fallback. Add an `.npmignore` containing `src/` and run it again: **`.env` is in the tarball.** Delete `.npmignore`, restore `"files": ["dist", "src"]`, and confirm it's gone. Publish to a local Verdaccio if you want to see it on a registry page.

### Explain out loud (60–120 seconds)
1. `exports` as a gate and a switchboard, and why `types` goes first and `default` last.
2. ESM-only vs dual in 2026, and the top-level-await caveat.
3. `^0.x`, and why a library's lockfile doesn't protect its consumers.
4. Phantom dependencies, and what pnpm's layout changes.
5. The release pipeline: build to JS, verify, changesets, trusted publishing with provenance.

---

## What's next

The SDK and CLI are published. The last lesson of the module ships the Engine itself: a production Dockerfile, PID 1 and signals, memory flags against container limits, what `NODE_ENV=production` really changes, and what serverless does to your pools and timers.

Next → **[Lesson 33: Shipping Node: containers, config & runtime flags](33-shipping-node.md)**
