# Lesson 04 — Modules: CommonJS vs ESM, properly

> **Why this lesson exists:** module errors burn hours: `ERR_REQUIRE_ESM`, `ERR_PACKAGE_PATH_NOT_EXPORTED`, "Cannot use import statement outside a module", a singleton that somehow exists twice. They burn hours because people don't know that Node has two loading algorithms. Once you know both, every one of these errors names the exact step that failed. And since `require()` learned to load ES modules, much of the advice still repeated about modules is out of date.

**Time:** ~75 minutes · **Prereq:** Lesson 03

---

## 1. The idea in one sentence

> **CommonJS runs a file when you `require` it and hands back whatever it exported, synchronously, one call at a time. ESM parses the whole import graph first, links every import to a live binding, and only then runs anything. Every module error is a consequence of which algorithm is in charge.**

---

## 2. Why there are two systems

In 2009 JavaScript had no module system, so Node adopted **CommonJS**: `require()` and `module.exports`. Synchronous loading was fine, because the files were on local disk.

In 2015 the language standardised **ES modules**: `import` and `export`. They're designed for browsers, so the syntax is **static** (tools and engines can see the whole dependency graph without running code) and loading is **asynchronous by design** (the browser fetches modules over the network, in parallel).

Node has supported both for years, and the npm registry is full of both. Each error below comes from the gap between them:

| Error | What happened | Section |
|---|---|---|
| `Cannot use import statement outside a module` | ESM syntax in a file Node decided is CommonJS | 5 |
| `require is not defined in ES module scope` | The inverse | 5 |
| `ERR_REQUIRE_ESM` | `require()` of ESM on an old Node line | 6 |
| `ERR_REQUIRE_ASYNC_MODULE` | `require()` of an ESM graph that uses top-level `await` | 6 |
| `Named export 'x' not found … is a CommonJS module` | Node's static guess at a CJS module's exports failed | 6 |
| `ERR_PACKAGE_PATH_NOT_EXPORTED` | A deep import past a package's `exports` gate | 8 |
| `instanceof` is false for an object whose class name matches | Two copies of the same module | 9 |

TypeScript's view of resolution (how it finds *types*, `--traceResolution`, `.d.ts` files) is [TS L14](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/14-modules-and-declarations.md). This lesson is what Node does at runtime.

---

## 3. CommonJS: how `require` works

### The module wrapper

Node never runs a CommonJS file as-is. It wraps the source in a function and calls it:

```js
(function (exports, require, module, __filename, __dirname) {
  // your file's code
});
```

That's printed by `require("node:module").wrapper`, and it explains most of CommonJS:

- **Top-level variables are module-scoped**, because they're local variables of the wrapper function.
- **`require`, `module`, `__filename` and `__dirname` aren't globals.** They're parameters. That's why they don't exist in ESM, which has no wrapper (section 7).
- **`exports` is just a parameter pointing at `module.exports`**, and `require` returns `module.exports`:

```js
exports.formatMinor = fn;              // ✅ adds a property to the shared object
module.exports = { formatMinor: fn };  // ✅ replaces the object that require() returns
exports = { formatMinor: fn };         // ❌ rebinds the local parameter; the module exports {}
```

### The resolution algorithm

```text
require(X) from /app/src/payments/service.js

1. X is a core module ("fs", "node:fs")       → return the built-in
2. X starts with "./", "../" or "/"           → as a file:      X, X.js, X.json, X.node
                                                as a directory: its package.json "main", then index.js
3. Otherwise X is a bare specifier ("pg", "@ledger/money/format") → walk UP the directory tree:
        /app/src/payments/node_modules/pg
        /app/src/node_modules/pg
        /app/node_modules/pg                   ← found
        /node_modules/pg
   Inside the package: "exports" if present (and then it's a gate, section 8),
   otherwise "main", otherwise index.js
4. Nothing found → Error: Cannot find module 'pg'   (code: MODULE_NOT_FOUND)
```

Every step is filesystem lookups, done synchronously. That's part of the startup cost in section 10.

### Synchronous loading, the cache, and singletons

`require(X)` resolves X to an absolute filename, then checks `require.cache`. On a miss, it creates the module object, **puts it in the cache before running it**, runs the wrapper, and returns `module.exports`. On a hit, it returns the cached `module.exports` without running anything.

- **That cache is why a module is a singleton.** Your `db.js` exporting one pool gives every `require("./db.js")` the same pool.
- **It's a singleton per resolved filename, not per package name.** Two copies of a package at two paths in `node_modules` are two modules, with two independent states (section 9).
- **`require` blocks the event loop** while it reads and compiles. Fine at startup; a `require()` inside a request handler does synchronous I/O on the first call.

### Cycles return a partially filled `exports`

Because a module is cached *before* it runs, a cycle hands back whatever the module had exported so far:

```js
// a.cjs
const b = require("./b.cjs");                  // a starts, then requires b...
module.exports = { greet: () => "hello from a" };

// b.cjs
const a = require("./a.cjs");                  // ...b requires a, which is mid-execution
console.log(a);                                // {}  ← a hasn't assigned module.exports yet
module.exports = { name: "b" };
```

Run `node a.cjs`. `b` gets `{}`, and Node warns: `Accessing non-existent property 'greet' of module exports inside circular dependency`. Worse, because `a` *replaced* `module.exports`, **`b` holds the old empty object forever**; `a.greet` stays `undefined` even after `a` finishes. With the `exports.greet = …` style, `b` holds the same object that later fills in, so the property appears by the time anything calls it. Either way, the fix is structural: break the cycle by moving the shared piece into a third module.

---

## 4. ESM: construct, link, evaluate

### The three phases

```text
 1. CONSTRUCTION   resolve every specifier, load every file, parse each into a module record
                   → every import and export in the graph is known, and no code has run
 2. LINKING        create each module's export bindings and point every import at the
                   exporter's binding: wiring, not copying. Function declarations are
                   initialised here; let/const/class bindings exist but are uninitialised
 3. EVALUATION     run module bodies, dependencies first (post-order);
                   a module that uses top-level await can pause, and its importers wait
```

The consequences you can observe:

- **Imports are static.** `import` declarations must be at the top level with string-literal specifiers, and they're hoisted: every import is resolved and evaluated before the first line of the importing module runs, wherever the `import` appears in the file.
- **A wrong named import fails before any code runs.** `import { Payment } from "./types.ts"` when `types.ts` has no runtime export `Payment` throws `SyntaxError: The requested module './types.ts' does not provide an export named 'Payment'` at link time. CommonJS destructuring would silently give you `undefined`. (This is the error behind [L03](03-the-process.md)'s `import type` rule.)
- **ESM is always strict mode**, top-level `this` is `undefined`, and there's no `require`, `module`, `exports`, `__filename` or `__dirname`.
- **There's no cache you can delete from.** A module instance lives as long as the process. `import("./x.ts?v=2")` creates a *new* instance (the module map is keyed by URL) and keeps the old one, which is how naive ESM hot-reloading leaks memory.

### Live bindings

```ts
// counter.ts
export let count = 0;
export function increment(): void { count++; }

// main.ts
import { count, increment } from "./counter.ts";
console.log(count);      // 0
increment();
console.log(count);      // 1  ← the import is a live view of counter.ts's variable
count = 5;               // 💥 TypeError: Assignment to constant variable.
```

The CommonJS equivalent, `const { count, increment } = require("./counter.cjs")`, prints `0` and `0`: destructuring **copied** the value at require time. In ESM **an imported name is a read-only window onto the exporter's binding.** The exporter can change it, and every importer sees the change; importers can't assign to it. This is why ESM cycles work better than CommonJS ones (below), and why tests can't monkey-patch an ES module's exports from outside, so mocking ESM needs loader-level support ([L31](../08-shipping/31-testing-node.md)).

### Top-level `await`

An ES module can `await` at the top level:

```ts
// src/main.ts
const config = loadConfig();
const secrets = await fetchSigningKeys(config);      // the module pauses here; its importers wait
```

The cost is that **everything importing that module waits for it**, and the module can no longer be loaded with `require()` (section 6). If the promise never settles and nothing keeps the loop alive, the process exits with code 13 ([L03](03-the-process.md)). The rule: **top-level `await` in entry points, never in library modules.** A library that needs async setup should export an `init()` function instead.

### Cycles in ESM: the temporal dead zone

```ts
// x.ts
import { y } from "./y.ts";
export const x = "x";

// y.ts
import { x } from "./x.ts";
console.log(x);          // 💥 ReferenceError: Cannot access 'x' before initialization
export const y = "y";
```

Linking succeeded: `y.ts`'s `x` points at `x.ts`'s binding. But `y.ts` runs first (it's a dependency of `x.ts`), and the binding is in its temporal dead zone until `x.ts` executes `export const x`. **ESM cycles fail loudly with a `ReferenceError` for `let`, `const` and `class`. Function declarations work across a cycle**, because they're initialised during linking. If you've heard "ESM cycles give `undefined`", that's CommonJS, or TypeScript compiled to CommonJS. Same fix as before: break the cycle, or make the cross-references type-only with `import type`.

---

## 5. How Node decides which algorithm to use

```text
.mjs, .mts   → ESM
.cjs, .cts   → CommonJS
.js,  .ts    → look at the nearest package.json "type":
                 "module"    → ESM
                 "commonjs"  → CommonJS (ESM syntax is a SyntaxError)
                 absent      → try CommonJS; if the file contains ESM syntax
                               (import/export, import.meta, top-level await),
                               re-parse it as ESM and warn
```

That last branch is **syntax detection**, on by default since 22.7. It works, but it parses the file twice and prints `[MODULE_TYPELESS_PACKAGE_JSON] Warning: … Reparsing as ES module because module syntax was detected. This incurs a performance overhead.` **Always set `"type"` explicitly.** Ledger's `package.json` says `"type": "module"`, so every `.ts` and `.js` file is ESM, and the rare CommonJS file has to say so with `.cjs`.

Now the two classic errors read as plain facts:

- **"Cannot use import statement outside a module"**: Node decided this file is CommonJS (an explicit `"type": "commonjs"`, or a `.cjs` extension) and found `import`.
- **"require is not defined in ES module scope, you can use import instead"**: Node decided this file is ESM and found `require`.

---

## 6. Interop in both directions

### ESM importing CommonJS

```ts
import pg from "pg";                     // ✅ always works: the default export IS module.exports
import { Pool } from "pg";               // works only if Node can detect `Pool` statically
```

**The default import is always `module.exports`.** Named imports are a best effort. A CommonJS module's exports only exist after it runs, but ESM must link before anything runs, so Node scans the CommonJS source with a static analyser (cjs-module-lexer) looking for patterns like `exports.x = …`, `module.exports.x = …`, `Object.defineProperty(exports, "x", …)` and `module.exports = { a, b }`. Exports computed at runtime aren't detectable:

```js
// dynamic.cjs
const api = {};
for (const k of ["formatMinor", "currency"]) api[k] = k;
module.exports = api;
```

`import { currency } from "./dynamic.cjs"` fails at link time with `Named export 'currency' not found. The requested module './dynamic.cjs' is a CommonJS module, which may not support all module.exports as named exports`, and the message tells you the fix: import the default and destructure.

### CommonJS requiring ESM

**`require()` of an ES module works unflagged since 22.12 and 20.19.** In 2026, that means every supported Node line (22, 24, 26). It returns the module namespace object:

```js
const money = require("./money.mjs");
money.formatMinor(4999n, "usd");         // named exports are properties
money.default;                           // the default export, if there is one
```

When the ES module has a default export, Node also sets `__esModule: true` on the namespace, so code compiled by older TypeScript or Babel, which checks that flag, still finds `.default`. On Node lines before 22.12 the same `require()` threw `ERR_REQUIRE_ESM`, and that's the reason so many packages shipped two builds.

The one hard rule: `require()` is synchronous, so **it can't load an ESM graph that uses top-level `await`**:

```text
Error [ERR_REQUIRE_ASYNC_MODULE]: require() cannot be used on an ESM graph with top-level await.
Use import() instead. To see where the top-level await comes from, use --experimental-print-required-tla.
```

The `await` can be anywhere in the imported graph, not just the file you named, and that flag finds it.

### `import()` from anywhere

Dynamic `import()` works in both CommonJS and ESM, returns a promise of the namespace, and handles top-level `await`. It's the escape hatch for CommonJS code that needs an async ES module, and the tool for lazy loading (section 10). Going the other way, `createRequire(import.meta.url)` from `node:module` gives ESM code a real `require`, which you need only for unusual cases.

---

## 7. ESM's replacements for CommonJS conveniences

| CommonJS | ESM |
|---|---|
| `__dirname` | `import.meta.dirname` (since 20.11) |
| `__filename` | `import.meta.filename` |
| `require("./fixtures.json")` | `import fixtures from "./fixtures.json" with { type: "json" }` (default export only) |
| `require.resolve("pkg")` | `import.meta.resolve("pkg")` (returns a `file://` URL string) |
| `module.exports = x` | `export default x` |
| `require(path)` anywhere, synchronously | `await import(path)` anywhere, asynchronously |

**Why `__dirname` doesn't exist in ESM:** it was never a global. It was a parameter of the CommonJS wrapper, and ESM has no wrapper. An ES module's identity is a URL (`import.meta.url`, `file:///…`), because the same module system runs in browsers, where there's no directory to speak of.

### Always use the `node:` prefix

`import { readFile } from "node:fs/promises"`, never `"fs/promises"`. A bare specifier is a resolution that **happens** to hit a built-in. `node:` can only ever mean a built-in. Some built-ins exist only with the prefix: `require("test")` looks in `node_modules` for an npm package called `test`, while `require("node:test")` is Node's test runner (`node:sqlite` works the same way). The prefix makes a module unshadowable, makes the dependency obvious to readers and tools, and protects you when Node adds a built-in whose name an npm package already uses.

---

## 8. The `exports` field is a gate

```jsonc
// node_modules/@ledger/money/package.json
{
  "name": "@ledger/money",
  "type": "module",
  "exports": {
    ".": "./dist/index.js",              // import "@ledger/money"
    "./format": "./dist/format.js",      // import "@ledger/money/format"
    "./package.json": "./package.json"
  }
}
```

**Once a package has `exports`, that map is its entire public surface.** Anything not listed is unreachable:

```text
import "@ledger/money/dist/internal.js"
Error [ERR_PACKAGE_PATH_NOT_EXPORTED]: Package subpath './dist/internal.js' is not defined by "exports"
```

That's deliberate. Before `exports`, any file in a package was importable, so internal refactors broke consumers who had reached into `lib/whatever.js`. With `exports`, the author decides what's public and can reorganise everything else.

Each entry can also be a set of **conditions** that pick a file depending on who's asking: `"import"`, `"require"`, `"types"`, `"node"`, `"default"`. Conditions and publishing are [L32](../08-shipping/32-packaging-and-publishing.md); the rule that `"types"` goes first is in [TS L14](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/14-modules-and-declarations.md).

`exports` has a sibling worth knowing now: **`"imports"` defines private aliases starting with `#`.** Node ignores tsconfig `paths` ([L03](03-the-process.md)), and this is the Node-native replacement:

```jsonc
// Ledger's package.json
{ "type": "module", "imports": { "#lib/*": "./src/lib/*" } }
```
```ts
import { pool } from "#lib/db.ts";       // resolved by Node itself, and understood by tsc's nodenext
```

---

## 9. The dual-package hazard

To support both module systems before `require(esm)` existed, many packages shipped two builds selected by conditions: `"require"` → `dist/index.cjs`, `"import"` → `dist/index.js`. Now picture one application that loads both. Your code does `import { Money } from "@ledger/money"` and gets the ESM build; a dependency does `require("@ledger/money")` and gets the CommonJS build. **They're two different files, so they're two different modules with two different `Money` classes.**

```ts
import { Money } from "@ledger/money";                       // the ESM build
import { createRequire } from "node:module";
const require = createRequire(import.meta.url);
const { make } = require("./make.cjs");                      // requires the CJS build inside

const m = make();
console.log(m instanceof Money, m.constructor.name);         // false Money
```

`false Money` is the signature: the class name matches, the identity doesn't. In real systems it looks like this:

- **Error handling breaks.** `err instanceof ApiError` is false in your error middleware, so a 404 becomes a 500.
- **Singletons split.** A registry, cache or connection pool exists twice, each with half the state: a plugin registered in one copy can't be found in the other, and a metrics library exports only the metrics recorded in "its" copy.
- **Nothing errors.** You get wrong answers, and the debugger shows two objects with the same name.

The same thing happens without dual builds when npm installs two *versions* of one package at different paths: two resolved filenames, two module instances. `npm ls @ledger/money` shows the duplicates.

**The 2026 answer is ESM-only packages.** Every supported Node line can `require()` an ES module, so a CommonJS consumer and an ESM consumer get the same single ESM file, and there's one copy of every class. The condition is the one from section 6: no top-level `await` anywhere in the package's graph. Where you must defend against duplication you don't control, check for a brand (a `Symbol.for("ledger.money")` property) instead of relying on `instanceof`.

> **Spring equivalent:** the JVM classpath is one flat, ordered search path where the first match wins, so Maven must pick a single version of every library, and conflicts surface as `NoSuchMethodError`: [Spring L01 — The JVM, the classpath & build tools](https://github.com/Pawar-Pratik/SpringBoot/blob/main/01-java-for-spring/01-jvm-build-and-classpath.md). `node_modules` is the opposite trade. Resolution walks up from the *requiring file*, so each package can get its own version nested beneath it. You rarely get Java's version conflict; you get duplicate module instances instead, and broken `instanceof` and split singletons are Node's version of classpath hell.

---

## 10. What modules cost at startup

Every module costs a resolution (filesystem lookups walking `node_modules`), a read, a compile and an evaluation, and **all of it happens before your server's first line of real work**: CommonJS `require` is synchronous, and static ESM imports are loaded and linked before the entry module runs. A large dependency tree (a cloud SDK, an ORM, a framework plus its plugins) shows up as slow cold starts. That matters for CLIs (every `ledger` command pays it), for serverless, and for how fast a crashed pod is back in service during an incident.

**Measure before optimising.** `node --cpu-prof src/main.ts` writes a profile you can open in Chrome DevTools, and module loading shows up as its own frames ([L28](../07-production/28-performance-and-v8.md)). For a quick number, wrap an `await import()` in `performance.now()` calls.

Then fix it where it matters:

- **Lazy-load rarely used paths with `import()`.** Ledger's CSV exporter, with its CSV and gzip machinery, is needed only when someone requests a statement:

  ```ts
  // src/routes/exports.ts
  import type { Request, Response } from "express";

  type CsvExporter = typeof import("../exports/csv-exporter.ts");
  let exporter: Promise<CsvExporter> | undefined;

  export async function getStatementCsv(req: Request, res: Response): Promise<void> {
    exporter ??= import("../exports/csv-exporter.ts");   // loaded once, on first use
    const { streamStatementCsv } = await exporter;
    await streamStatementCsv(req, res);                   // the streaming itself is L13
  }
  ```

  The trade-off: lazy loading moves load errors from boot to the first request. A typo in `csv-exporter.ts` no longer stops the deploy; it fails the first export. Cover lazy paths in tests, or warm them up after `listen()`.
- **Avoid internal barrel files.** Importing one name from a barrel loads every module behind it ([TS L14](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/14-modules-and-declarations.md)).
- **CLI subcommands import their own dependencies**, so `ledger --version` doesn't load the HTTP client.
- **Node can cache compiled code on disk** (`module.enableCompileCache()`), which helps short-lived processes that start often.

---

## 11. Production rules

| Rule | Why |
|---|---|
| **Set `"type"` explicitly in every `package.json`** | Syntax detection parses twice and warns; an explicit type makes every file's format obvious |
| **Write ESM; use `.cjs` only for the rare file that must be CommonJS** | One algorithm, static imports, link-time errors, live bindings |
| **Always use the `node:` prefix for built-ins** | It can't be shadowed by a package, and some built-ins exist only with it |
| **Top-level `await` in entry points only, never in libraries** | It makes the module un-`require()`-able and delays every importer |
| **Publish ESM-only packages** | Every supported Node line can `require(esm)`; one build means one copy of every class |
| **Give every package an `exports` map** | It defines the public surface and makes internals refactorable |
| **Use `"imports"` (`#lib/*`) for internal aliases, never tsconfig `paths`** | Node ignores `paths`; `imports` is resolved by Node and understood by tsc |
| **Break value cycles by extracting shared code** | CJS cycles return partial exports; ESM cycles throw TDZ `ReferenceError`s |
| **Check `npm ls <pkg>` when `instanceof` mysteriously fails** | Duplicate module instances are the usual cause |
| **Lazy-load rarely used, heavy modules with `import()`, and test those paths** | Faster boot, but load errors move from deploy time to first use |
| **Never `require()` inside a request handler** | Synchronous I/O on the event loop on first use |

---

## 12. Interview traps

**Q1. "CommonJS vs ESM: three real differences."**
Loading: CommonJS loads and runs synchronously, one `require` at a time; ESM constructs and links the whole graph before any code runs. Bindings: CommonJS gives you a copy of whatever `module.exports` held when you destructured it; ESM imports are live, read-only bindings. Analysis: ESM imports are static, so a missing export fails at link time and tools can see the graph; `require` is a runtime function call. Then: ESM supports top-level `await`; CommonJS has `__dirname` and a deletable cache. The weak answer is "different syntax".

**Q2. "What are live bindings?"**
An ES module import isn't a copy; it's a read-only reference to the exporter's variable. If the exporter does `count++`, every importer sees the new value. Importers can't assign to it (`TypeError: Assignment to constant variable.`). It's set up in the linking phase, before any code runs, which is also why function declarations work across ESM cycles.

**Q3. "Can `require` load an ES module?"**
Yes, unflagged since 22.12 and 20.19, so on every supported line in 2026. It returns the namespace object, with the default export on `.default`. The exception: if anything in the imported graph uses top-level `await`, it throws `ERR_REQUIRE_ASYNC_MODULE`, because `require` must be synchronous; use `import()` there.
**Follow-up:** "What changed for package authors?" Dual CJS/ESM builds are no longer necessary: ESM-only reaches both kinds of consumer.

**Q4. "What's the dual-package hazard?"**
A package ships CommonJS and ESM builds, and one app loads both, so two module instances exist: two copies of each class, two copies of module state. `instanceof` fails across them and singletons split, all silently. Fix: ship ESM-only now that `require(esm)` is universal; defensively, use brand checks instead of `instanceof`.

**Q5. "Why does `__dirname` not exist in ESM?"**
It was never a global. It's a parameter of the function CommonJS wraps every file in. ESM has no wrapper, and its module identity is a URL, because it's the same system browsers use. Use `import.meta.dirname` and `import.meta.filename`.

**Q6. "What does `exports` in package.json do?"**
It declares the package's public entry points and blocks everything else: a deep import gets `ERR_PACKAGE_PATH_NOT_EXPORTED`. It can also select different files by condition (`import`, `require`, `types`, `node`, `default`). Authors use it to keep internals private and to serve the right build to each consumer.

**Q7. "How does `require` find a module?"**
Core modules first. Then relative or absolute paths, trying the exact name, then `.js`, `.json` and `.node`, then a directory's `main` or `index.js`. Bare names walk up the directory tree checking each `node_modules`, and inside the package use `exports` if present, otherwise `main`. The result is cached by resolved filename, which is why modules are singletons, but only per path.

**Q8. "What happens with circular dependencies, in each system?"**
CommonJS caches a module before running it, so a cycle returns the partially filled `exports`: `{}` if the module later replaces `module.exports`, and that stale object is kept forever. Node warns when you read a missing property. ESM links bindings first, so a cycle that reads a `let`/`const`/`class` before its module has run throws a TDZ `ReferenceError`; function declarations work. Fix both by extracting the shared code.

**Q9. "'Cannot use import statement outside a module.' What decided that?"**
Node's format rules: the file is `.cjs`, or it's `.js`/`.ts` under a `package.json` with `"type": "commonjs"`. Syntax detection only applies when `"type"` is absent. Set `"type": "module"` or rename the file to `.mjs`.

**Q10. "Why use the `node:` prefix?"**
A bare name is a lookup that happens to hit a built-in; `node:` can only mean the built-in. Some modules (`node:test`, `node:sqlite`) exist only with the prefix: bare `test` resolves to an npm package. It also documents intent and survives Node adding built-ins that collide with package names.

---

## 13. Build & break

### Build — convert a CommonJS utility to ESM and install it into Ledger

Start from a CommonJS package next to Ledger:

```js
// ledger-money/index.js — BEFORE (package.json has no "type", so CommonJS)
const MINOR_DIGITS = { usd: 2, eur: 2, inr: 2, jpy: 0 };

function formatMinor(amountMinor, currency) {
  const digits = MINOR_DIGITS[currency];
  if (digits === undefined) throw new RangeError(`unknown currency: ${currency}`);
  const s = (amountMinor < 0n ? -amountMinor : amountMinor).toString().padStart(digits + 1, "0");
  const sign = amountMinor < 0n ? "-" : "";
  return digits === 0 ? sign + s : `${sign}${s.slice(0, -digits)}.${s.slice(-digits)}`;
}
module.exports = { formatMinor, MINOR_DIGITS };
```

Convert it:

1. `package.json`: `"name": "@ledger/money"`, `"type": "module"`, and an `exports` map with `"."`, `"./package.json"` and nothing else. Add a hand-written `index.d.ts` with a `"types"` condition listed first ([TS L14](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/14-modules-and-declarations.md)).
2. Replace `module.exports` with `export function` / `export const`. Move the currency table into `internal.js` and import it, so there's a deep path for the gate to block.
3. In Ledger: `npm install ../ledger-money`. npm creates a symlink in `node_modules` (a junction on Windows).
4. Verify three things: `import { formatMinor } from "@ledger/money"` works from a `.ts` file; a `.cjs` file can `require("@ledger/money")` (that's `require(esm)`, and it's why one ESM build is enough); `import "@ledger/money/internal.js"` fails with `ERR_PACKAGE_PATH_NOT_EXPORTED`.

One detail worth knowing: because the linked package is a symlink and Node resolves real paths, a *linked* local package could even contain `.ts` files and still be stripped. A package installed from the registry lives under `node_modules` for real, and Node refuses to strip it, so a published package must ship JavaScript ([L32](../08-shipping/32-packaging-and-publishing.md)).

### Build — lazy-load the CSV exporter

Add the `getStatementCsv` route from section 10, with a stub `csv-exporter.ts` that imports something heavy. Log `performance.now()` when the server starts listening, once with a static import and once with the lazy `import()`, and compare the two startup times. Then put a syntax error in the exporter and confirm that the server still boots and the first export fails. Decide where you'd catch that (a test that hits the route).

### Break — five experiments

1. **The CommonJS cycle.** Build `a.cjs` and `b.cjs` from section 3. Run `node a.cjs`: `b` logs `{}`, Node warns about accessing a non-existent property inside a circular dependency, and `a.greet` stays `undefined` in `b` forever. Switch `a` to `exports.greet = …` and see what changes. Then fix it properly by moving `greet` into `greeting.cjs`.
2. **The dual-package hazard.** Create `node_modules/@ledger/money-dual/` by hand with `"exports": { ".": { "import": "./index.js", "require": "./index.cjs" } }` and a `class Money` defined in each file. From an ES module, import `Money`, and also `createRequire` a CommonJS helper that constructs a `Money` through `require`. `instanceof` prints `false` while `constructor.name` prints `Money`. Delete the CommonJS build, point both conditions at the ESM file, and re-run: `true`.
3. **`require()` meets top-level `await`.** Write `keys.mjs` with `export const keys = await Promise.resolve(["k1"])` and `require()` it from a `.cjs` file. Read the `ERR_REQUIRE_ASYNC_MODULE` message, run again with `--experimental-print-required-tla` to see where the `await` lives, then fix it two ways: `await import()` from the CommonJS side, or export an async `loadKeys()` instead of awaiting at the top level.
4. **The `exports` gate.** Import `@ledger/money/internal.js` and read `ERR_PACKAGE_PATH_NOT_EXPORTED`. Add `"./internal": "./internal.js"` to the map and it works, which is the point: the author decided.
5. **A live binding, observed.** Build `counter.ts` and `main.ts` from section 4. Watch the value change through the import, then try `count = 5` and read the `TypeError`. Rewrite the pair as CommonJS with destructuring and watch the copy stay at `0`. Finally, build the `x.ts`/`y.ts` cycle and read the TDZ `ReferenceError`; change `export const x` to `export function x()` and see the cycle work.

### Explain out loud (60–120 seconds)

1. The CommonJS wrapper, and what it explains: module scope, `__dirname`, `exports` versus `module.exports`.
2. The `require` resolution steps and the cache, and why a singleton exists once per path.
3. ESM's three phases, and what "live binding" means.
4. How Node decides a file's format, and what the two classic errors mean.
5. Interop both ways, including `require(esm)` and its top-level-`await` limit.
6. The dual-package hazard, and why ESM-only is the 2026 answer.

---

## Module 1 complete — checkpoint

Before Module 2, check you can do each of these **without looking**:

- [ ] Draw Node's four layers and say what V8 and libuv each own, and what each knows nothing about
- [ ] Trace `fs.readFile` through the thread pool and `http.get` through the kernel, and say why they differ
- [ ] Answer "is Node single-threaded?" precisely, with the thread counts you measured
- [ ] Compare thread-per-request, the event loop and virtual threads: what waiting costs and what CPU work costs in each
- [ ] Name where Node wins, where it loses, and what you do with CPU-bound work
- [ ] Explain why production runs LTS, and how the release model changes from Node 27
- [ ] Explain why module-level state leaks between requests, and what `structuredClone` drops
- [ ] Explain how closures retain memory, including V8's shared context
- [ ] Give the five `this` rules, and why Node makes the detached-method bug silent
- [ ] Explain why `pg` returns `bigint` as a string, and how Ledger parses and serialises money
- [ ] Explain `??` versus `||` with the refund bug
- [ ] List the four process lifecycle steps, and say why a script won't exit and how you find the culprit
- [ ] Explain why piped output gets truncated, and choose between `process.exitCode` and `process.exit()`
- [ ] Decode exit codes 0, 1, 13, 130, 137 and 143
- [ ] Validate config at boot, including the env-boolean trap
- [ ] Say what type stripping does and doesn't do, and justify every line of the track's tsconfig
- [ ] Explain the CommonJS wrapper, the `require` algorithm and partial exports in cycles
- [ ] Explain ESM's three phases, live bindings and TDZ cycles
- [ ] Explain `require(esm)` and its top-level-`await` limit
- [ ] Explain the dual-package hazard and the ESM-only answer

Any unticked box: re-read that section now. Module 2 assumes all of them.

---

## What's next

Module 1 gave you the map: the threads, the language, the process and the modules. Module 2 is the heart of the track, and it starts with the most important lesson in it: the event loop, phase by phase. Not one "callback queue", but six phases, two priority queues drained between every callback, and a poll phase that sleeps in the kernel.

Next → **[Lesson 05: The event loop, phase by phase](../02-event-loop/05-the-event-loop.md)**
