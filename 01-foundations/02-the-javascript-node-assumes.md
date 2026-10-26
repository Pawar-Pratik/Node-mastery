# Lesson 02 — The JavaScript Node assumes

> **Why this lesson exists:** many Node bugs are JavaScript-semantics bugs in disguise: a leak through a closure, `this` lost inside a callback, money that drifts by a cent, a refund of `0` that silently became a full refund. TypeScript type-checks every one of them. This lesson covers only the JavaScript that bites on a server, and skips what the TypeScript track already taught.

**Time:** ~70 minutes · **Prereq:** Lesson 01

---

## 1. The idea in one sentence

> **TypeScript checks the shapes of your values; JavaScript decides what they do at runtime. References, closures, `this` and doubles behave identically with or without types, and each has a server-side failure mode.**

---

## 2. Why it exists: types don't change semantics

Four lines from a plausible Ledger codebase. All four compile under `strict`.

```ts
const amount = body.amount_minor || payment.amount_minor;   // ❌ 0 → refunds the full payment
emitter.on("row", this.onRow);                              // ❌ `this` is the emitter, not you
const total = rows.reduce((s, r) => s + Number(r.amount_minor), 0);  // ❌ wrong past 2^53
pending.set(id, () => log(`export was ${csv.length} bytes`));        // ❌ keeps all of `csv` alive
```

None of them throws. Each produces a wrong answer, a silent no-op or a slow leak. That's the category this lesson is about: **behaviour the compiler can't see because it's decided by how JavaScript evaluates code, not by what types the code has.**

---

## 3. Values, references, and state shared between requests

Primitives (`number`, `string`, `boolean`, `bigint`, `symbol`, `undefined`, `null`) are copied by value. Everything else (objects, arrays, functions, `Map`s, `Buffer`s, class instances) is passed **by reference**: assignment copies the pointer, not the object. `const` freezes the *binding*, not the object behind it.

On a server this matters more than in a browser, for one reason: **module-level state is shared by every concurrent request in the process.** There's no request scope unless you build one.

```ts
// src/routes/payments.ts
const DEFAULT_LIST = { limit: 20, expand: [] as string[] };

// ❌ Mutates the module-level object. Request A's `expand` leaks into request B's.
function listOptions(q: { limit?: string; expand?: string }) {
  const opts = DEFAULT_LIST;
  if (q.limit) opts.limit = Number(q.limit);
  if (q.expand) opts.expand.push(q.expand);
  return opts;
}

// ❌ Still wrong: spread is shallow, so `expand` is the same array as before.
const opts = { ...DEFAULT_LIST };

// ✅ Build a fresh object per request.
function listOptionsFixed(q: { limit?: string; expand?: string }) {
  return { limit: q.limit ? Number(q.limit) : 20, expand: q.expand ? [q.expand] : [] };
}
```

In a multi-tenant system, **shared mutable state is a cross-tenant data leak waiting for the right interleaving**. The same goes for middleware that mutates `req.body` in place: every later middleware and handler sees the mutation, including code that cached a reference to the original.

When you genuinely need a deep copy, `structuredClone` is built in:

| `structuredClone(x)` | Behaviour |
|---|---|
| Nested objects, arrays, cycles | Deep-copied |
| `Date`, `Map`, `Set`, `RegExp`, `BigInt`, typed arrays | Supported |
| Class instances | **Copied as plain objects: the prototype is dropped**, so methods vanish and `instanceof` is false |
| Functions | **Throws `DataCloneError`** |
| Cost | A full traversal and copy, so don't clone large payloads per request |

`Object.freeze` is shallow too. Freeze your validated config object ([L03](03-the-process.md)) and know that nested objects inside it stay mutable unless you freeze them as well.

---

## 4. Closures: the unit of state in Node

A closure is a function plus the variables of the scope it was created in, **kept alive for as long as the function is reachable**. Node code is closures all the way down: every callback, every middleware factory, and every suspended `async` function (its local variables live on the heap until it resumes).

They're the idiomatic way to hold private state:

```ts
// A factory: `lastRunAt` is private, per instance, and needs no class.
export function createSweeper(opts: { intervalMs: number; sweep: () => Promise<number> }) {
  let lastRunAt: number | undefined;
  let timer: NodeJS.Timeout | undefined;
  return {
    start() {
      timer = setInterval(async () => {
        lastRunAt = Date.now();
        await opts.sweep().catch((err: unknown) => console.error("sweep failed", err));
      }, opts.intervalMs);
    },
    stop() { clearInterval(timer); },
    status: () => ({ lastRunAt }),
  };
}
```

### How closures keep memory alive

Anything a closure captures stays reachable as long as the closure does. That's the mechanism behind a whole family of leaks ([L27](../07-production/27-memory-and-gc.md) hunts them with heap snapshots):

```ts
const completed = new Map<string, () => void>();

async function finishExport(exportId: string, merchantId: string) {
  const csv = await buildStatementCsv(merchantId);          // a 50 MB string
  await upload(exportId, csv);

  // ❌ Only needs the length, but captures `csv`: 50 MB per entry, kept until deleted.
  completed.set(exportId, () => console.log(`export ${exportId}: ${csv.length} bytes`));

  // ✅ Capture the number, not the string. And bound the map, or delete entries.
  const bytes = csv.length;
  completed.set(exportId, () => console.log(`export ${exportId}: ${bytes} bytes`));
}
```

One V8 detail makes this worse than it looks: **closures created in the same scope share one context object.** If *any* closure in that scope references `csv`, the context holds `csv`, and *every* closure from that scope keeps the context alive. So a long-lived closure that never mentions `csv` can still retain it, because a short-lived sibling (a `.then` callback, say) did. When a heap snapshot shows a huge string retained by a closure that "doesn't use it", this is why.

### The loop-variable classic

```ts
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i), 0);   // 3 3 3
for (let i = 0; i < 3; i++) setTimeout(() => console.log(i), 0);   // 0 1 2
```

`var` has one binding for the whole function, so all three closures see its final value. `let` creates a fresh binding per iteration. You'll never write `var` in this track, but output-prediction questions still use it ([L37](../10-interview/37-output-and-implementation-challenges.md)).

---

## 5. `this`: decided at call time

`this` isn't a property of the function. **It's decided by how the function is called**, except for arrow functions, which have no `this` of their own.

| Call form | `this` inside | Example |
|---|---|---|
| **Default:** plain call | `undefined` in strict mode, which ESM and class bodies always are | `const f = obj.m; f()` |
| **Implicit:** method call | The object before the dot | `obj.m()` |
| **Explicit** | Whatever you pass | `m.call(x)`, `m.apply(x)`, `m.bind(x)` |
| **`new`** | The freshly created object | `new Payment()` |
| **Arrow function** | The enclosing scope's `this`, captured lexically, and can't be rebound | `() => this.count++` |

The textbook bug is passing a method as a callback, `setTimeout(obj.method)`, which detaches it from `obj`. **Node adds a twist that makes the bug silent instead of loud:** Node's APIs often call your callback with a `this` of their own.

- **`EventEmitter` calls listeners with `this` set to the emitter** (for ordinary functions; arrows keep their lexical `this`).
- **Node's `setTimeout` calls the callback with `this` set to the `Timeout` object.**

So the detached method doesn't crash with "cannot read properties of undefined". It writes to the wrong object:

```ts
import { EventEmitter } from "node:events";

class StatementExporter {
  rows = 0;
  onRow() { this.rows++; }                    // a prototype method: `this` depends on the caller
}

const parser = new EventEmitter();
const exporter = new StatementExporter();
parser.on("row", exporter.onRow);             // ❌ detached
parser.emit("row");
parser.emit("row");
console.log(exporter.rows);                   // 0
console.log((parser as any).rows);            // NaN: `undefined + 1`, written onto the emitter
```

No error, no warning: the count just never moves. The fixes:

```ts
class StatementExporter {
  rows = 0;
  onRow = (): void => { this.rows++; };       // ✅ arrow field: bound at construction
}
parser.on("row", (row) => exporter.onRow(row));   // ✅ or wrap at the call site
```

Arrow fields cost one closure per instance and can't be overridden by a subclass ([TS L13](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/13-classes-and-oop.md) covers the class trade-offs). One more Node-specific trap, previewing [L11](../03-data-and-io/11-eventemitter.md)'s listener leaks:

```ts
parser.on("row", exporter.onRow.bind(exporter));
parser.off("row", exporter.onRow.bind(exporter));   // ❌ removes nothing: bind() returns a NEW function
// Keep the bound reference, or use an arrow field, so on() and off() get the same function.
```

---

## 6. Prototypes: what `class` actually is

```ts
class Payment {
  status = "requires_payment_method";         // a field: created on EACH instance
  constructor(public readonly id: string) {}  // ❌ not erasable: Node can't strip this (L03)
  capture() { this.status = "succeeded"; }    // a method: ONE function, on Payment.prototype
}
```

Drop the parameter property and that's roughly:

```ts
function Payment(this: { id: string; status: string }, id: string) {
  this.status = "requires_payment_method";
  this.id = id;
}
Payment.prototype.capture = function () { this.status = "succeeded"; };
```

`typeof Payment` is `"function"`, and `Object.getOwnPropertyNames(Payment.prototype)` is `["constructor", "capture"]`. **A property lookup walks the chain:** the object's own properties, then its prototype, then the prototype's prototype, up to `Object.prototype`, then `null`. Methods live once, on the prototype; every instance shares them.

Why this matters on a server:

1. **Plain objects used as dictionaries inherit keys.** With user-controlled keys (merchant metadata, query parameters, CSV headers), `counts["constructor"]` is a function and `"toString" in counts` is `true`. Code that does `if (seen[key])` gives wrong answers for those keys.
2. **Prototype pollution.** If attacker-controlled input reaches a naive deep-merge, a key like `__proto__` writes onto `Object.prototype`, and every object in the process inherits the attacker's property ([L30](../08-shipping/30-security-and-supply-chain.md)).

```ts
const seen: Record<string, true> = {};
"constructor" in seen;                        // true ❌ inherited from Object.prototype

const safe = Object.create(null) as Record<string, true>;   // ✅ no prototype at all
"constructor" in safe;                        // false

const byKey = new Map<string, number>();      // ✅ usually better: a real dictionary
Object.hasOwn(obj, key);                      // ✅ own properties only
```

And since instances get their methods from the prototype, **anything that drops the prototype drops the methods**: `JSON.parse`, `structuredClone`, messages to worker threads. That's the argument for plain objects plus functions at boundaries, made in [TS L13](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/13-classes-and-oop.md).

---

## 7. Numbers: doubles, money, and why `pg` returns strings

Every JavaScript `number` is an IEEE-754 double: 52 explicit fraction bits plus one implicit bit, so **53 bits of integer precision**.

```ts
0.1 + 0.2;                       // 0.30000000000000004
let s = 0; for (let i = 0; i < 10; i++) s += 0.1;   // s === 0.9999999999999999
Number.MAX_SAFE_INTEGER;         // 9007199254740991, which is 2^53 − 1
2 ** 53 === 2 ** 53 + 1;         // true: past 2^53, adjacent integers collapse
Number("9007199254740993");      // 9007199254740992, silently a different number
```

The wire-format rules (integer minor units plus a currency, large IDs as strings) are in [API L05](https://github.com/Pawar-Pratik/API-mastery/blob/main/01-foundations/05-data-formats.md); don't re-derive them. This section is about what happens **inside the Node process**.

### Why `pg` returns `bigint` columns as strings

Postgres `bigint` (`int8`) is a 64-bit integer: values go up to about 9.2 × 10^18, far past 2^53. If `node-postgres` converted `int8` to a JavaScript `number`, some values would silently become different numbers. So **its default parser returns the text unchanged**, and `numeric` comes back as a string for the same reason ([L25](../06-data-and-jobs/25-databases-from-node.md) covers the driver itself). Two consequences surprise everyone:

```ts
const { rows } = await pool.query("SELECT count(*) FROM payments WHERE merchant_id = $1", [id]);
rows[0].count + 1;               // "421": count(*) returns bigint, so it's the string "42"

const r = await pool.query("SELECT sum(amount_minor) AS total FROM ledger_entries WHERE …");
typeof r.rows[0].total;          // "string": sum(bigint) returns numeric, also a string
```

Ledger's `amount_minor` is `bigint`, so every amount arrives as a string. You have three honest options:

| Option | How | Gains | Costs |
|---|---|---|---|
| **Keep strings** | Treat amounts as opaque decimal strings until you need arithmetic | Zero precision risk; JSON just works | Every calculation needs a conversion; `+` concatenates silently |
| **`BigInt` in the domain** | Parse with `BigInt(s)` at the repository boundary; serialise deliberately | Exact at any size; TypeScript keeps `bigint` and `number` apart, so mixing them is a compile error | `JSON.stringify` throws; division truncates; some libraries don't accept it |
| **Cap at safe integers** | Parse to `number` and assert `Number.isSafeInteger` | Plain numbers everywhere | One missing assertion is silent corruption; IDs and platform-wide sums can exceed 2^53 |

**Ledger uses `BigInt` in the domain.** Amounts are parsed once, in the repositories, and every calculation is exact. The `BigInt` behaviours you must know:

```ts
1n + 1;                          // ❌ TypeError: Cannot mix BigInt and other types, use explicit conversions
BigInt("12.50");                 // ❌ SyntaxError: Cannot convert 12.50 to a BigInt
BigInt(1.5);                     // ❌ RangeError: … not an integer
7n / 2n;                         // 3n: truncates toward zero, so -7n / 2n is -3n
0n ? "truthy" : "falsy";         // "falsy"
JSON.stringify({ a: 10n });      // ❌ TypeError: Do not know how to serialize a BigInt
```

**Division truncates, so fee calculations must round explicitly.** A 2.9% fee is `amount * 290n / 10_000n`, which always rounds down. Decide the rounding rule (half-up, banker's) and write it as code, with a test. And **never fix the JSON problem by patching `BigInt.prototype.toJSON`**: that changes serialisation for every library in the process. Use a replacer at your own boundary (Build & break).

---

## 8. Equality and truthiness

Use `===`. The one `==` some style guides allow is `x == null`, which is true for both `null` and `undefined`.

The falsy values are `false`, `0`, `-0`, `0n`, `""`, `null`, `undefined` and `NaN`. Everything else is truthy, **including the strings `"0"` and `"false"`** (which is why `if (process.env.FEATURE_X)` is a bug, [L03](03-the-process.md)).

**`||` replaces any falsy left side; `??` replaces only `null` and `undefined`.** On money, the difference is a refund bug:

```ts
// The client sends { amount_minor: 0 }: invalid, and it should be rejected.
const amount = body.amount_minor || payment.amount_minor;   // ❌ 0 is falsy → refunds EVERYTHING
const amount2 = body.amount_minor ?? payment.amount_minor;  // ✅ 0 stays 0 → validation rejects it
```

The client meant "nothing" and got a full refund. `??` alone isn't the whole fix: validate the input (`amount_minor > 0`) so a zero is a `422`, not an amount ([TS L15](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/15-the-boundary-and-parsing.md)). The same applies to `||=` versus `??=`.

**Optional chaining** (`a?.b?.c`, `fn?.()`) short-circuits to `undefined`. Use it for things that are genuinely optional. On something that must exist, `?.` hides the bug: throw instead, so the broken invariant is loud.

Conversions that matter when parsing input:

| Expression | Result | Why it bites |
|---|---|---|
| `Number("")` | `0` | An empty env var becomes port 0 |
| `Number(" 42 ")` | `42` | Whitespace is ignored |
| `Number("3000abc")` | `NaN` | Strict, and `NaN !== NaN`: test with `Number.isNaN` |
| `parseInt("3000abc")` | `3000` | **Silently accepts trailing garbage** |
| `Number("0x10")` | `16` | Hex is accepted |

---

## 9. Iterators and generators

`for…of`, spread and destructuring all run on one protocol: an object with a `[Symbol.iterator]()` method that returns `{ next(): { value, done } }`. Arrays, strings, `Map`s and `Set`s implement it; plain objects don't.

A **generator** (`function*`) returns an iterator whose body pauses at each `yield` and resumes on the next `next()`. It's lazy: nothing runs until someone asks for a value.

```ts
// Split rows into batches for a multi-row INSERT. Lazy: one batch exists at a time.
export function* batches<T>(items: Iterable<T>, size: number): Generator<T[]> {
  let batch: T[] = [];
  try {
    for (const item of items) {
      batch.push(item);
      if (batch.length === size) { yield batch; batch = []; }
    }
    if (batch.length > 0) yield batch;
  } finally {
    // Runs on normal completion AND when the consumer stops early (break, return, throw).
    console.log("batching finished");
  }
}

for (const b of batches(rows, 500)) {
  await insertLedgerEntries(b);
}
```

The `finally` detail is the important one: **when a `for…of` loop exits early, it calls the iterator's `return()`, which runs the generator's `finally`.** That's how an async generator over a database cursor closes the cursor when the consumer stops reading. Async generators (`async function*` with `for await`) are the backbone of Ledger's ingester and SDK pagination; the depth is in [L14](../03-data-and-io/14-composing-streams.md), and the typing is in [TS L16](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/16-async-and-errors.md).

---

## 10. Error objects and stack traces

**Throw `Error` objects, never strings.** A thrown string has no stack, fails `instanceof Error`, and turns into `Error: undefined` in a careless logger. The typing side (`catch` is `unknown`, `Error.cause` chaining) is in [TS L16](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/04-real-code/16-async-and-errors.md).

Runtime facts that matter in Node:

- **The stack is captured when the `Error` is constructed**, not when it's thrown. Constructing errors isn't free, so don't use them for control flow on a hot path.
- **V8 keeps 10 frames by default** (`Error.stackTraceLimit`). Deep framework stacks can cut off before your code appears; raising the limit in development helps.
- **`Error.captureStackTrace(target, fn)`** is a V8 API that attaches a `.stack` to any object and can omit every frame above `fn`, so an `assertInvariant()` helper doesn't appear at the top of its own traces. Custom error classes use it.
- **Node's own errors carry a `code`:** `ENOENT`, `ECONNREFUSED`, `ERR_INVALID_ARG_TYPE`. System errors add `syscall`, `errno` and sometimes `path`. **Branch on `code`, never on `message`**: messages change between versions; codes are a contract.

```ts
try {
  await readFile(path);
} catch (err) {
  if (err instanceof Error && "code" in err && err.code === "ENOENT") return null;   // ✅
  throw err;                                                     // anything else is real
}
```

Which errors to catch, and which should crash the process, is [L09](../02-event-loop/09-errors-and-crashing.md).

---

## 11. Dates

A `Date` is a UTC millisecond timestamp (one number) with local-time-zone accessors bolted on. The pitfalls:

| Pitfall | Example |
|---|---|
| **Months are 0-indexed, and overflow silently** | `new Date(2026, 1, 31)` is **March 3rd**, not an error |
| **Date-only strings parse as UTC; date-time strings without an offset parse as local** | Covered in [API L05](https://github.com/Pawar-Pratik/API-mastery/blob/main/01-foundations/05-data-formats.md) |
| **`Date` is mutable** | `d.setDate(d.getDate() + 1)` changes it for every holder of the reference (section 3 again) |
| **The server's time zone leaks in** | Containers usually run UTC and laptops don't, so local-time code passes in CI and fails on your machine, or the reverse |
| **Millisecond precision only** | Postgres `timestamptz` has microseconds. `pg` returns a `Date`, so a round trip **truncates microseconds** |

That last row bites Ledger directly. Cursor pagination on `(created_at, id)` ([API L08](https://github.com/Pawar-Pratik/API-mastery/blob/main/02-rest-design/08-collections-and-pagination.md)) that round-trips `created_at` through a JS `Date` can skip or repeat rows that share a millisecond. Encode the cursor from the database's text representation, or cast in SQL, rather than from a `Date`.

The rules: store `timestamptz`; compute in UTC epoch milliseconds; serialise with `toISOString()` (ISO-8601 with `Z`); convert to local time only at the edge (the Console); run servers and tests with `TZ=UTC`; and measure durations with a monotonic clock (`performance.now()`, [L03](03-the-process.md)), never `Date.now()`. **Temporal**, the replacement date API, is arriving in JavaScript engines; check whether your Node version ships it before relying on it.

---

## 12. Production rules

| Rule | Why |
|---|---|
| **Never mutate module-level objects from request code** | Module state is shared by every concurrent request; mutation leaks data between them, and across tenants |
| **Copy deliberately: spread is shallow, `structuredClone` is deep and drops prototypes** | Half-copied objects share nested arrays; cloned class instances lose their methods |
| **Capture values, not big objects, in long-lived closures** | A closure keeps everything it captures reachable, and V8 shares one context across a scope's closures |
| **Never pass a bare method as a callback** | Node calls it with the emitter or the `Timeout` as `this`, so the bug is silent |
| **Keep the same function reference for `on()` and `off()`** | `bind()` creates a new function, so `off()` removes nothing and listeners leak |
| **`Map` or `Object.create(null)` for user-controlled keys** | Plain objects inherit `constructor`, `toString` and `__proto__` |
| **`??` for defaults, never `||`, when `0` or `""` are valid values** | `||` treats a real zero as missing, and on money that's a wrong amount |
| **Money is `bigint` minor units inside the process; parse `pg` strings at the repository boundary** | Exact arithmetic; doubles are wrong past 2^53 and for fractions |
| **Round explicitly in every division of money** | `BigInt` division truncates toward zero |
| **Serialise `bigint` with a replacer at your boundary, never `BigInt.prototype.toJSON`** | Patching a built-in prototype changes behaviour for every library |
| **Throw `Error` objects; branch on `err.code`, never `err.message`** | Strings have no stack; messages change between versions, codes don't |
| **Store and compute in UTC; run with `TZ=UTC`; never round-trip cursor timestamps through `Date`** | Local zones make tests environment-dependent; `Date` truncates microseconds |

---

## 13. Interview traps

**Q1. "Why does `pg` return `bigint` columns as strings?"**
Postgres `bigint` is 64-bit and JavaScript numbers are exact only up to 2^53 − 1, so converting automatically would silently corrupt large values. The driver returns the text and lets you choose. Then the details that show you've hit it: `count(*)` is `bigint`, so it arrives as `"42"` and `+ 1` concatenates; and `sum()` over `bigint` returns `numeric`, also a string.
**Follow-up:** "How would you handle it?" Parse to `BigInt` at the repository boundary for exact arithmetic, and serialise with a replacer that checks the safe range, or emit strings when values can exceed it.

**Q2. "Explain closures, and a memory leak they can cause."**
A closure is a function plus the scope it was created in, kept alive as long as the function is reachable. The leak: a long-lived closure (stored in a map, registered as a listener, held by a timer) captures a large object it barely uses, so the object can never be collected. The senior detail: V8 shares one context object between all closures created in a scope, so a closure can retain a variable it never references because a sibling closure did.

**Q3. "What's `this` inside a callback passed to an EventEmitter?"**
For an ordinary function, the emitter: Node sets it deliberately. For an arrow function, the enclosing scope's `this`. So `emitter.on("data", this.handle)` runs `handle` with `this` as the emitter, and writes like `this.count++` land on the emitter silently. Fix with an arrow class field or `(x) => this.handle(x)`. Node's `setTimeout` does the same thing with the `Timeout` object.

**Q4. "`??` vs `||`?"**
`||` falls back on any falsy value: `0`, `""`, `false`, `NaN`. `??` falls back only on `null` and `undefined`. Defaulting a fee, an amount or a count with `||` turns a legitimate zero into the default. The concrete bug: `body.amount_minor || payment.amount_minor` turns an invalid zero into a full refund.

**Q5. "How do you represent money in JavaScript?"**
Integer minor units with an explicit currency, never a floating-point decimal. Inside a Node process I'd hold them as `bigint`, parsed from `pg`'s strings at the boundary, so arithmetic is exact at any size and TypeScript stops me mixing them with `number`. I'd round explicitly on division and serialise through a replacer. The weak answer is "use `toFixed(2)`": that formats a double, it doesn't make it exact.

**Q6. "What does `class` compile to?"**
A constructor function plus methods on its `prototype`. Fields are created per instance in the constructor; methods exist once on the prototype and are found by walking the prototype chain. That's why a class instance that goes through `JSON` or `structuredClone` comes back as a plain object without its methods.

**Q7. "Integer minor units avoid `0.1 + 0.2`. Are you safe with `number` then?"**
Mostly, until 2^53 − 1, which is far beyond any single payment. The danger comes back three ways: aggregates and IDs that grow without bound (`bigserial`, platform-wide sums), any division or percentage (fees, splits), and `Number("…")` conversions that silently round. `bigint` with explicit rounding removes all three.

**Q8. "What does `structuredClone` do that spread doesn't, and what does it lose?"**
It copies deeply, handles cycles, and supports `Date`, `Map`, `Set` and `BigInt`. Spread copies one level, so nested objects stay shared. `structuredClone` drops prototypes (class instances become plain objects) and throws `DataCloneError` on functions.

**Q9. "Why never throw a string?"**
No stack trace, `instanceof Error` fails, and loggers that read `err.message` print `undefined`. A thrown string at 3am is an error with no location. Throw an `Error` (or a subclass) and chain causes.

**Q10. "Why would `seen[key]` give the wrong answer for some keys?"**
Plain objects inherit from `Object.prototype`, so keys like `constructor` and `toString` are always "present". With user-controlled keys that's a correctness bug, and with merges it's the entry point for prototype pollution. Use a `Map`, `Object.create(null)` or `Object.hasOwn`.

---

## 14. Build & break

### Build — Ledger's `money.ts`

Create `src/domain/money.ts`. It's pure (no I/O), so it gets fast unit tests.

```ts
export type Currency = "usd" | "eur" | "gbp" | "inr" | "jpy";
export type Money = { readonly amountMinor: bigint; readonly currency: Currency };

const INT8_TEXT = /^-?\d{1,19}$/;

/** Parse an int8 column value from pg (always a string) into Money. */
export function fromPg(amountMinor: string, currency: Currency): Money {
  if (!INT8_TEXT.test(amountMinor)) throw new TypeError(`not an int8 value: ${amountMinor}`);
  return { amountMinor: BigInt(amountMinor), currency };
}

export function add(a: Money, b: Money): Money {
  if (a.currency !== b.currency) {
    throw new RangeError(`currency mismatch: ${a.currency} + ${b.currency}`);
  }
  return { amountMinor: a.amountMinor + b.amountMinor, currency: a.currency };
}

/** Basis points (1 bp = 0.01%), rounded half-up. Non-negative amounts only. */
export function feeBps(m: Money, bps: bigint): Money {
  if (m.amountMinor < 0n) throw new RangeError("fees are computed on non-negative amounts");
  return { amountMinor: (m.amountMinor * bps + 5_000n) / 10_000n, currency: m.currency };
}

const MAX_JSON = BigInt(Number.MAX_SAFE_INTEGER);

/** JSON.stringify replacer: bigint → number when exactly representable, otherwise fail loudly. */
export function bigintReplacer(_key: string, value: unknown): unknown {
  if (typeof value !== "bigint") return value;
  if (value > MAX_JSON || value < -MAX_JSON) {
    throw new RangeError(`bigint ${value} exceeds the JSON-safe integer range`);
  }
  return Number(value);
}
```

Usage at the HTTP edge: `res.type("application/json").send(JSON.stringify(dto, bigintReplacer))`. The API contract sends `amount_minor` as a JSON integer ([API L05](https://github.com/Pawar-Pratik/API-mastery/blob/main/01-foundations/05-data-formats.md)), so the replacer converts when it's exact and **throws rather than corrupts** when it isn't. An API whose amounts can exceed 2^53 would emit strings instead.

Write these tests with `node:test` ([L31](../08-shipping/31-testing-node.md) covers it properly; `node --test` is enough for now):

- `add(fromPg("9007199254740993", "usd"), fromPg("1", "usd"))` is exactly `9007199254740994n`
- `feeBps` of 10,000 at 290 bp is `290n`; of 1 at 290 bp is `0n`; of 5 at 1,000 bp is `1n` (the half-up case: 0.5 rounds up)
- `add` across currencies throws `RangeError`
- `fromPg("12.50", "usd")` and `fromPg("", "usd")` throw `TypeError`
- `JSON.stringify({ amount: 4999n }, bigintReplacer)` is `{"amount":4999}`; `2n ** 60n` throws

### Break — six experiments

Run each with `node -e "…"` or a scratch `.ts` file.

1. **Float money.** Sum `0.1` ten times. You get `0.9999999999999999`, and `=== 1` is `false`. Fix: integer minor units, `10n * 10n === 100n`.
2. **BigInt and JSON.** `JSON.stringify({ a: 10n })` throws `TypeError: Do not know how to serialize a BigInt`. Fix: `bigintReplacer`. Then try `2n ** 60n` through it and read the `RangeError`: that's the loud failure you want.
3. **The silent rounding.** `Number("9007199254740993")` prints `9007199254740992`. No error, a different number. Fix: `BigInt("9007199254740993")`.
4. **Lose `this` in a timer.**
   ```ts
   class Sweeper {
     count = 0;
     tick() { this.count++; }
     start() { setTimeout(this.tick, 5); }        // ❌ detached method
   }
   const s = new Sweeper();
   s.start();
   setTimeout(() => console.log(s.count), 50);    // 0: the increment went onto the Timeout object
   ```
   No crash. Log `this` inside `tick` and you'll see a `Timeout`. Fix: `tick = (): void => { this.count++; }`, or `setTimeout(() => this.tick(), 5)`.
5. **The `||` refund.** Write `refundAmount(body, payment)` with `body.amount_minor || payment.amount_minor`, call it with `{ amount_minor: 0 }` and a 4,999 payment: it returns 4,999. Switch to `??` and validate `> 0`; the call now fails validation, which is the correct outcome.
6. **A closure leak you can watch.** In a loop, push closures that capture a fresh `new Array(5_000_000).fill(0)` (about 40 MB) and use only its `length`. Log `process.memoryUsage().heapUsed` each iteration and watch it climb. Capture `const n = arr.length` instead and watch it stay flat.

### Explain out loud (60–120 seconds)

1. Why module-level state is dangerous in a server, and what `structuredClone` does and doesn't copy.
2. What a closure retains, and the shared-context detail.
3. The five `this` rules, and why Node makes the detached-method bug silent.
4. What `class` desugars to, and why `Object.create(null)` exists.
5. Why `pg` returns `bigint` as a string, Ledger's choice, and how it serialises.
6. `??` versus `||`, using the refund bug.

---

## What's next

You know what your JavaScript does. Next, the box it runs in: the process. How it starts, why it sometimes refuses to exit, why piped output gets cut off, what exit code 137 means, and how Node runs TypeScript with no build step.

Next → **[Lesson 03: The process: your program's contract with the OS](03-the-process.md)**
