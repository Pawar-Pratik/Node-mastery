# Lesson 37 — Output prediction & build-from-scratch challenges

> **Why this lesson exists:** two live formats dominate Node coding rounds: *"what does this print?"* with nested ticks, promises and timers, and *"implement `p-limit` / `EventEmitter` / `retry` on this shared screen."* Both are performance skills. Knowing the event loop isn't the same as narrating it phase by phase under observation, and knowing what `p-limit` does isn't the same as writing one that frees its slot on a synchronous throw. This lesson gives you a method for each format, sixteen puzzles verified on Node 24, and ten challenges with tested reference solutions.

**Time:** ~2.5 hours, then practise · **Prereq:** Modules 2–5 (especially [Lesson 05](../02-event-loop/05-the-event-loop.md) and [Lesson 06](../02-event-loop/06-promises-and-microtasks.md))

---

## 1. The method for output puzzles

Say this out loud when you're handed a snippet. It buys thinking time and shows process.

```text
1. CONTEXT     Main module or inside a callback? CommonJS or ES module?
2. CLASSIFY    Tag every line: sync · nextTick · microtask · timer · immediate · I/O · emit
3. SYNC        Write the synchronous output first. It always comes first.
4. DRAIN       nextTick queue to empty, then microtask queue to empty; repeat until both are empty
5. PHASE       Move to the next phase that has work: timers → poll → check → close → timers …
6. PER CALLBACK  After EACH callback (every timer, every immediate), go back to step 4
7. NEW WORK    Anything scheduled during a callback joins the back of its own queue
```

```text
              ┌──────────── after EVERY single callback ────────────┐
              │  1. nextTick queue  → drain fully (incl. new ticks)  │
              │  2. microtask queue → drain fully (promises,         │
              │     queueMicrotask, await continuations)             │
              │  3. repeat until both are empty                      │
              └──────────────────────────────────────────────────────┘
   ┌──► timers ──► pending ──► idle/prepare ──► poll ──► check ──► close ──┐
   └────────────────────────────────────────────────────────────────────────┘
```

**The convention for this lesson:** every puzzle was run on Node 24. Each is a `.ts` file run with `node pN.ts` in a project with `"type": "module"`, so it's an **ES module**, which is how this track runs code. Puzzle 1 is the one exception (`p1.cjs`), to show the difference. Only top-level code that mixes `nextTick` with promises cares about the module system, and puzzles 1 and 2 are the only ones that do.

**Narrate the queues, not the output.** *"`setTimeout` goes to the timers list, the `.then` to the microtask queue, the tick to the nextTick queue; the sync part prints A and E…"* is what the interviewer is scoring. The output is just where you end up.

---

## 2. Part 1 — Sixteen output puzzles

Cover everything below each snippet. Write your answer down, then compare. Tiers: **warm-up** (1–4), **core** (5–10), **hard** (11–16).

### Puzzle 1 — The warm-up (CommonJS)

```ts
// p1.cjs → node p1.cjs
console.log("A");
setTimeout(() => console.log("B"), 0);
Promise.resolve().then(() => console.log("C"));
process.nextTick(() => console.log("D"));
console.log("E");
```

**Your answer:** ____________

```text
A E D C B
```

**Why, phase by phase:**
1. Sync: A, E. Queued: timer B (0 coerced to 1 ms), microtask C, tick D.
2. The main script was the first "callback". When it returns, Node drains the nextTick queue (**D**), then the microtask queue (**C**).
3. The loop starts; the timers phase runs **B**. Even if 1 ms hadn't elapsed on the first check, nothing else is queued, so B is still last.

### Puzzle 2 — The same five lines, as an ES module

```ts
// p2.ts → node p2.ts   (identical body; "type": "module")
console.log("A");
setTimeout(() => console.log("B"), 0);
Promise.resolve().then(() => console.log("C"));
process.nextTick(() => console.log("D"));
console.log("E");
```

**Your answer:** ____________

```text
A E C D B
```

**Why:** the ESM loader is asynchronous, so Node evaluates the module body **from inside a promise job**. When the body ends you're mid-way through draining the microtask queue: V8 keeps going and runs **C** (and anything C queues; a three-link `.then` chain all runs) before control returns to Node, which then processes the nextTick queue (**D**). Timers come last as before.

> **The level-3 answer:** *"In CommonJS D prints before C; in an ES module C prints first, because the module body itself runs in a microtask. Inside any later callback the classic nextTick-first order returns."* Interviewers usually expect the CommonJS answer; volunteering both is the strong signal.

### Puzzle 3 — The famous race

```ts
// p3a.ts
setTimeout(() => console.log("timeout"), 0);
setImmediate(() => console.log("immediate"));
```

```ts
// p3b.ts
import { readFile } from "node:fs";

readFile(import.meta.filename, () => {
  setTimeout(() => console.log("timeout"), 0);
  setImmediate(() => console.log("immediate"));
});
```

**Your answer (for each):** ____________

```text
p3a: either order. Run it 40 times and you'll see both.
p3b: immediate, timeout. Every time.
```

**Why:**
- **p3a:** the timer's 0 becomes 1 ms. When the loop enters the timers phase for the first time, the timer fires only if 1 ms has already passed since it was scheduled, which depends on how long bootstrap took. If it hasn't, the loop moves on: poll doesn't block (an immediate is pending), check runs the immediate, and the timer fires next iteration. **The correct answer is "it depends", with that reason.**
- **p3b:** the `readFile` callback runs in the **poll** phase. The next phase is **check**, so the immediate runs before the loop wraps around to timers. Deterministic.

### Puzzle 4 — `await` and the caller

```ts
async function a() {
  console.log("a1");
  await b();
  console.log("a2");
}
async function b() {
  console.log("b");
}

console.log("start");
a();
Promise.resolve().then(() => console.log("p"));
console.log("end");
```

**Your answer:** ____________

```text
start a1 b end a2 p
```

**Why:**
1. `a()` runs synchronously up to its first `await`: prints a1, then calls `b()` **synchronously** (b prints and returns an already-fulfilled promise).
2. `await` on a fulfilled native promise queues a's continuation immediately: microtask 1. `a()` returns to the caller.
3. The `.then` queues microtask 2. "end" prints.
4. Drain: a2 (microtask 1), then p.

**The two traps:** thinking `b` runs later (it runs synchronously; only the code *after* `await` is deferred), and thinking a2 prints before "end" (the continuation always waits for the caller's synchronous code).

### Puzzle 5 — Closures in loops

```ts
for (var i = 0; i < 3; i++) setTimeout(() => console.log("var", i), 0);
for (let j = 0; j < 3; j++) setTimeout(() => console.log("let", j), 0);
for (let k = 0; k < 3; k++) {
  setTimeout(() => console.log("let+mutate", k), 0);
  k++;
}
```

**Your answer:** ____________

```text
var 3
var 3
var 3
let 0
let 1
let 2
let+mutate 1
let+mutate 3
```

**Why:**
- `var` is one binding for the whole module; the timers run after the loop finished, so all three closures read 3.
- `let` in a `for` head creates **a fresh binding per iteration**, so each closure captures its own `j`.
- The third loop is the level-3 part. Each iteration's binding is a copy of the previous one, made **before** the increment. Iteration 1's binding starts at 0, the body's `k++` makes it 1, and that's what its closure prints. The copy for iteration 2 starts at 1, the loop's increment makes it 2, the body's `k++` makes it 3. The next copy (3) is incremented to 4 and fails the test: two timers, printing 1 and 3.
- Timers with the same delay fire in insertion order, so the groups never interleave.

### Puzzle 6 — Emitting from a constructor

```ts
import { EventEmitter } from "node:events";

class Connection extends EventEmitter {
  constructor() {
    super();
    this.emit("ready");                                  // synchronous
    process.nextTick(() => this.emit("ready-deferred")); // after the caller's code
  }
}

const conn = new Connection();
conn.on("ready", () => console.log("ready"));
conn.on("ready-deferred", () => console.log("ready-deferred"));
console.log("constructed");
```

**Your answer:** ____________

```text
constructed
ready-deferred
```

**Why:** `emit` is synchronous. During the constructor nobody is listening yet, so `"ready"` goes nowhere (`emit` returns `false`) and is **lost forever**. The tick runs after the module body, by which time both listeners exist. This is exactly why Node core defers events like `'listening'` and `'connect'`, and why your own evented APIs must too ([Lesson 11](../03-data-and-io/11-eventemitter.md)).

### Puzzle 7 — Snapshots, return values and `'error'`

```ts
import { EventEmitter } from "node:events";

const e = new EventEmitter();
e.on("x", () => {
  console.log("A");
  e.on("x", () => console.log("C"));      // added DURING an emit
});
e.on("x", () => console.log("B"));

console.log("emit 1 →", e.emit("x"));
console.log("emit 2 →", e.emit("x"));
console.log("emit y →", e.emit("y"));
try {
  e.emit("error", new Error("nobody listening"));
} catch (err) {
  console.log("thrown:", (err as Error).message);
}
```

**Your answer:** ____________

```text
A
B
emit 1 → true
A
B
C
emit 2 → true
emit y → false
thrown: nobody listening
```

**Why:**
1. `console.log`'s arguments are evaluated first, and `emit` runs every listener **before it returns**, so A and B print before "emit 1 → true".
2. `emit` iterates over **a copy of the listener array taken when it started**. The C added during emit 1 isn't in that copy.
3. Emit 2's copy is [A, B, C]: all three run. (A adds a second C, which waits for emit 3.)
4. `emit` returns whether anyone listened. An `'error'` with no listener **throws the error itself**, synchronously, out of `emit`.

### Puzzle 8 — Between every callback

```ts
setTimeout(() => {
  console.log("timer 1");
  queueMicrotask(() => console.log("micro 1"));
  process.nextTick(() => console.log("tick 1"));
}, 0);

setTimeout(() => {
  console.log("timer 2");
  setImmediate(() => console.log("immediate"));
  setTimeout(() => console.log("timer 3"), 0);
  queueMicrotask(() => console.log("micro 2"));
}, 0);
```

**Your answer:** ____________

```text
timer 1
tick 1
micro 1
timer 2
micro 2
immediate
timer 3
```

**Why:**
1. After timer 1's callback, Node drains nextTick (**tick 1**, even though `queueMicrotask` was called first) and then microtasks (**micro 1**) **before timer 2 runs**. That's the Node 11 change; before it, all expired timers ran first.
2. Timer 2 queues an immediate (this iteration's check phase), timer 3 (it can't fire in the timers phase that's already running) and micro 2 (drained right after timer 2).
3. Poll doesn't block because an immediate is pending; check prints **immediate**; the next iteration prints **timer 3**.

> **Why the immediate is scheduled from timer 2, not timer 1.** Node reads the clock for each `setTimeout` call, so two `setTimeout(…, 0)` calls on consecutive lines can get start times a millisecond apart and fire in **different loop iterations**. A version of this puzzle that scheduled the immediate from timer 1 printed it *between* the timers in about one run in thirty. Deterministic puzzles need you to know what isn't deterministic.

### Puzzle 9 — `this` in callbacks

```ts
import { EventEmitter } from "node:events";

class Counter {
  count = 0;
  inc() { this.count++; console.log("inc:", this.count); }
  incArrow = () => { this.count++; console.log("arrow:", this.count); };
}

const c = new Counter();
const bus = new EventEmitter();
bus.on("tick", c.inc);
bus.on("tick", c.incArrow);
bus.emit("tick");

setTimeout(c.inc, 0);
setTimeout(() => console.log("c.count:", c.count), 0);

const detached = c.inc;
try { detached(); } catch (e) { console.log("detached:", (e as Error).name); }
```

**Your answer:** ____________

```text
inc: NaN
arrow: 1
detached: TypeError
inc: NaN
c.count: 1
```

**Why:**
1. `bus.on("tick", c.inc)` passes the function, not the binding. **EventEmitter calls listeners with `this` set to the emitter**, so `this.count` is `undefined` on the bus, `++` makes it `NaN`, and nothing crashes.
2. The arrow field captured `this` = `c` when the instance was built: **1**.
3. `detached()` is a plain call in class code, which is strict mode, so `this` is `undefined` and reading `.count` throws a `TypeError`.
4. **Node calls timer callbacks with `this` set to the `Timeout` object** (browsers don't), so the timer version also prints `NaN`, onto the Timeout.
5. Only the arrow touched the real counter.

**The point to say out loud:** a lost receiver in Node often doesn't crash; it silently writes `NaN` onto some other object. That's worse than the `TypeError`.

### Puzzle 10 — Nested queues inside an I/O callback

```ts
import { readFile } from "node:fs";

readFile(import.meta.filename, () => {
  setTimeout(() => console.log("timeout"), 0);
  setImmediate(() => console.log("immediate"));
  process.nextTick(() => {
    console.log("tick 1");
    Promise.resolve().then(() => console.log("promise from tick"));
    process.nextTick(() => console.log("tick from tick"));
  });
  Promise.resolve().then(() => {
    console.log("promise 1");
    process.nextTick(() => console.log("tick from promise"));
  });
  console.log("sync");
});
```

**Your answer:** ____________

```text
sync
tick 1
tick from tick
promise 1
promise from tick
tick from promise
immediate
timeout
```

**Why, queue by queue:**
1. The callback runs in poll and prints **sync**. Queues: nextTick [tick 1], microtasks [promise 1], check [immediate], timers [timeout].
2. Drain nextTick: **tick 1** adds "promise from tick" to the microtasks (behind promise 1) and "tick from tick" to the nextTick queue, which is still draining, so **tick from tick** runs next. The nextTick queue drains fully, including ticks added while draining.
3. Drain microtasks in order: **promise 1** (adds a tick), then **promise from tick**.
4. Both queues must be empty before moving on, so back to nextTick: **tick from promise**.
5. Poll ends → check: **immediate** → next iteration's timers: **timeout**.

### Puzzle 11 — Awaiting a thenable

```ts
const thenable = {
  then(resolve: (v: string) => void) {
    console.log("then called");
    resolve("thenable value");
  },
};

async function main() {
  console.log("before await");
  const v = await thenable;
  console.log(v);
}

main();
Promise.resolve()
  .then(() => console.log("p1"))
  .then(() => console.log("p2"));
console.log("sync end");
```

**Your answer:** ____________

```text
before await
sync end
then called
p1
thenable value
p2
```

**Why:**
1. `await thenable` wraps it in a new promise resolved with a thenable. The spec doesn't call `then` right away: it queues a job to call it (**job 1**). `main` suspends.
2. The `.then(p1)` is **job 2**. "sync end" prints.
3. Job 1 calls `thenable.then`: prints **then called**, and `resolve(...)` fulfils the wrapper, queueing main's continuation (**job 3**).
4. Job 2 prints **p1** and queues p2 (**job 4**). Job 3 prints **thenable value**. Job 4 prints **p2**.

**Why the spec defers calling `then`:** so foreign code can never run synchronously inside your stack (protection from Zalgo). The cost: a thenable takes one extra tick compared with a native promise.

### Puzzle 12 — Returning a promise from an async function

```ts
async function direct() { return "direct"; }
async function wrapped() { return Promise.resolve("wrapped"); }

wrapped().then(console.log);
direct().then(console.log);
Promise.resolve()
  .then(() => console.log("tick 1"))
  .then(() => console.log("tick 2"))
  .then(() => console.log("tick 3"));
```

**Your answer:** ____________

```text
direct
tick 1
tick 2
wrapped
tick 3
```

**Why:** `direct()`'s promise is fulfilled at once, so its `.then` runs in the first round. `wrapped()` returns a *promise*, so the async function's own promise is resolved with a thenable: one job to call `inner.then` (W1), one job to resolve the outer promise (W2), and only then the `.then` callback (W3). The tick chain advances one link per round alongside it: W1 ‖ direct ‖ tick 1 → W2 ‖ tick 2 → wrapped ‖ tick 3.

**The follow-up:** *"So is `return await p` slower?"* No. With V8's await optimisation, `return await p` resolves **one tick earlier** than `return p` (measured on Node 24: it prints between tick 1 and tick 2). Use `return await` inside `try/catch` for correctness ([Lesson 06](../02-event-loop/06-promises-and-microtasks.md)); never write code whose correctness depends on tick counts.

### Puzzle 13 — Backpressure in slow motion

```ts
import { Writable } from "node:stream";

const sink = new Writable({
  highWaterMark: 3,                       // bytes
  write(chunk, _enc, cb) {
    console.log(`write ${chunk}`);
    setImmediate(cb);                     // a destination that takes a moment
  },
});
sink.on("drain", () => { console.log("drain"); sink.end(); });
sink.on("finish", () => console.log("finish"));
sink.on("close", () => console.log("close"));

console.log(sink.write("ab"));
console.log(sink.write("cd"));
console.log("sync done");
```

**Your answer:** ____________

```text
write ab
true
false
sync done
write cd
drain
finish
close
```

**Why:**
1. `write("ab")`: nothing is in flight, so `_write` is called **synchronously inside `write()`**: "write ab" prints before `write` returns. Buffered length 2 < 3 → **true**.
2. `write("cd")`: a write is in flight, so "cd" is **buffered, not rejected**; length 4 ≥ 3 → **false**. `highWaterMark` is a threshold, not a limit.
3. First check phase: ab's callback runs; the buffer still holds "cd", so `_write("cd")` starts. Length is 2 again, not zero: no drain yet.
4. Next check phase: cd's callback empties the buffer, and since `write()` returned `false` earlier, **drain** fires. The handler calls `end()`: everything is flushed, so **finish**, then autoDestroy releases the stream: **close**.

### Puzzle 14 — Starving the loop with promises

```ts
setTimeout(() => console.log("timeout"), 0);
setImmediate(() => console.log("immediate"));

let n = 0;
function spin(): void {
  if (++n < 1_000_000) Promise.resolve().then(spin);
  else console.log("spun", n);
}
spin();
console.log("sync");
```

**Your answer:** ____________

```text
sync
spun 1000000
timeout
immediate
```

**Why:** every microtask queues the next, and the microtask queue must be empty before the loop can move, so **a million "async" steps run back to back**: no timers, no I/O, no immediates. This is the microtask starvation from [Lesson 06](../02-event-loop/06-promises-and-microtasks.md): async is not the same as yielding. Bonus point: when the loop finally starts, far more than 1 ms has passed, so the timer is already due on the first timers check. **The starvation made the famous race deterministic.** The fix is to yield with `setImmediate` between chunks.

### Puzzle 15 — The late catch

```ts
import { setTimeout as sleep } from "node:timers/promises";

async function main(wait: () => Promise<unknown>) {
  const p = Promise.reject(new Error("boom"));
  await wait();
  try {
    await p;
  } catch (e) {
    console.log("caught", (e as Error).message);
  }
}

await main(() => Promise.resolve());   // variant A: wait one microtask
console.log("A done");
await main(() => sleep(10));           // variant B: wait for a timer
console.log("B done");
```

**Your answer:** ____________

```text
caught boom
A done
<source line with a caret>
Error: boom
    at main (…/p15.ts:4:28)
    …
Node.js v24.x            ← exit code 1; "B done" never prints
```

**Why:** Node checks for unhandled rejections **after the microtask queue drains**. In variant A the continuation is a microtask, so `await p` attaches a handler within the same drain, before the check. In variant B the queue drains while `sleep(10)` is pending: `p` has no handler, the default `throw` mode turns it into an uncaught exception, and the process exits with code 1. The `try/catch` that would have handled it 10 ms later never runs. Fix: attach handlers when you create the promise (`await Promise.all([sleep(10), p])`), or don't start work you can't await yet.

### Puzzle 16 — `process.exit()` mid-flight

```ts
process.on("exit", (code) => {
  console.log("exit handler, code", code);
  setTimeout(() => console.log("timer in exit handler"), 0);
});
process.on("beforeExit", () => console.log("beforeExit"));

setTimeout(() => console.log("timer"), 0);
Promise.resolve()
  .then(() => {
    console.log("promise 1");
    process.exit(2);
  })
  .then(() => console.log("promise 2"));
queueMicrotask(() => console.log("microtask"));
console.log("sync");
```

**Your answer:** ____________

```text
sync
promise 1
exit handler, code 2         ← exit code 2
```

**Why:** the first microtask calls `process.exit(2)`, which emits `'exit'` **synchronously** and then terminates. The microtask queued right behind it, "promise 2", the timer, and the timer created inside the `'exit'` handler never run, because the loop never turns again. `'beforeExit'` isn't emitted on an explicit exit.

**The follow-up:** replace `process.exit(2)` with `process.exitCode = 2` and everything drains: `sync, promise 1, microtask, promise 2, timer, beforeExit, exit handler, code 2`, still with exit code 2. That's why `exitCode` is the default and `exit()` the exception ([Lesson 03](../01-foundations/03-the-process.md)).

---

## 3. The method for build-from-scratch challenges

```text
1. RESTATE      The contract: inputs, outputs, error behaviour, concurrency
2. SIGNATURE    Types first, said out loud
3. HAPPY PATH   The smallest version that works
4. FAILURE      Sync throws, rejections, timeouts, callbacks called twice
5. RESOURCES    Timers cleared, listeners removed, memory bounded
6. CANCEL       Where does the AbortSignal go?
7. TEST         One test per edge you named
```

**Steps 4 and 5 are where Node rounds are decided.** Any candidate can write the happy path of `withTimeout`; the one who says *"and I clear the timer on settle, or 10k calls leave 10k live timers"* gets the offer. Every solution below is erasable TypeScript, was run against tests on Node 24, and typechecks under the track's `tsconfig`.

---

## 4. Part 2 — Ten challenges

### Challenge 1 — `promisify`

**Prompt:** *"Implement `promisify(fn)` for an error-first callback API."*
**Constraints they'll add:** preserve `this`; a synchronous throw must become a rejection; typed.

```ts
type Callback<T> = (err: unknown, value?: T) => void;

export function promisify<A extends unknown[], T>(
  fn: (...args: [...A, Callback<T>]) => void,
): (...args: A) => Promise<T> {
  return function (this: unknown, ...args: A): Promise<T> {
    return new Promise<T>((resolve, reject) => {
      const callback: Callback<T> = (err, value) => {
        if (err) reject(err);                 // error-first: any truthy err wins
        else resolve(value as T);
      };
      Reflect.apply(fn, this, [...args, callback]); // a sync throw here rejects the promise
    });
  };
}
```

**What they're looking for:** the Promise executor **converts a synchronous throw into a rejection**, so callers never see Zalgo; settle-once semantics come free (a callback invoked twice is ignored); a `function`, not an arrow, so `promisify(obj.method).call(obj)` keeps its receiver.
**Common mistakes:** returning an arrow function (loses `this`); `try/catch` around the call instead of trusting the executor; resolving with an array of all values. Then say the production answer: use `util.promisify`, or better, the native promise API (`node:fs/promises`).

### Challenge 2 — A minimal `EventEmitter`

**Prompt:** *"Implement `on`, `once`, `off` and `emit`, with Node's `'error'` semantics."*
**Constraints:** typed events; `off` must remove a `once` listener; listeners added or removed during an emit don't affect that emit.

```ts
type Listener = (...args: any[]) => void;
type Entry = { fn: Listener; once: boolean };

export class Emitter<Events extends Record<string, unknown[]>> {
  #listeners = new Map<keyof Events, Entry[]>();

  on<K extends keyof Events>(name: K, fn: (...args: Events[K]) => void): this {
    return this.#add(name, fn, false);
  }

  once<K extends keyof Events>(name: K, fn: (...args: Events[K]) => void): this {
    return this.#add(name, fn, true);
  }

  off<K extends keyof Events>(name: K, fn: (...args: Events[K]) => void): this {
    const list = this.#listeners.get(name);
    if (!list) return this;
    const i = list.findIndex((e) => e.fn === fn);   // works for once() too: we stored the original
    if (i !== -1) list.splice(i, 1);
    if (list.length === 0) this.#listeners.delete(name);
    return this;
  }

  emit<K extends keyof Events>(name: K, ...args: Events[K]): boolean {
    const list = this.#listeners.get(name);
    if (!list || list.length === 0) {
      if (name === "error") {                       // Node's rule: an unheard error is fatal
        const err = args[0];
        throw err instanceof Error ? err : new Error(`Unhandled 'error' event: ${String(err)}`);
      }
      return false;
    }
    for (const entry of [...list]) {                // snapshot: changes during emit apply next time
      if (entry.once) this.off(name, entry.fn);     // remove BEFORE calling (re-entrancy, throws)
      entry.fn.apply(this, args);                   // synchronous; a throw propagates to emit's caller
    }
    return true;
  }

  #add<K extends keyof Events>(name: K, fn: Listener, once: boolean): this {
    const list = this.#listeners.get(name) ?? [];
    list.push({ fn, once });
    this.#listeners.set(name, list);
    return this;
  }
}

// const events = new Emitter<{ paid: [paymentId: string, amountMinor: bigint]; error: [Error] }>();
```

**What they're looking for:** synchronous emit; **iterating a snapshot**; removing a `once` listener *before* calling it, so a re-entrant emit or a throw can't fire it twice; `this` = the emitter; `'error'` throwing.
**Common mistakes:** iterating the live array while a `once` listener splices itself out, which **silently skips the next listener**; wrapping `once` in a new function so `off(original)` can't find it (Node solves that by storing the original on the wrapper); making `emit` async; catching listener errors.

### Challenge 3 — `pLimit`

**Prompt:** *"Implement `pLimit(n)`: `limit(fn)` runs at most n functions concurrently."*
**Constraints:** FIFO; each call gets its own result or error; a failure must not leak a slot.

```ts
export function pLimit(concurrency: number) {
  if (!Number.isInteger(concurrency) || concurrency < 1) {
    throw new RangeError("concurrency must be a positive integer");
  }
  const queue: Array<() => void> = [];
  let active = 0;

  const next = (): void => {
    if (active < concurrency) queue.shift()?.();
  };

  return function limit<T>(fn: () => Promise<T>): Promise<T> {
    return new Promise<T>((resolve, reject) => {
      const run = async (): Promise<void> => {
        active++;
        try {
          resolve(await fn());                // a sync throw inside fn lands in catch too
        } catch (err) {
          reject(err);
        } finally {
          active--;
          next();                             // one slot freed → start exactly one waiter
        }
      };
      queue.push(() => void run());
      next();
    });
  };
}

// await Promise.all(endpoints.map((ep) => limit(() => deliver(ep))));
```

**What they're looking for:** the slot is released in `finally`, so success, rejection and a synchronous throw all free it; **one finish starts exactly one waiter**; no unhandled rejections (each caller owns its promise). Follow-ups: `activeCount`/`pendingCount`, and an `AbortSignal` that removes a queued task.
**Common mistakes:** batching (`Promise.all` on chunks of n), where the slowest item of each batch holds up the next; decrementing only on success, so one failure leaks a slot forever; a `next()` that starts every queued task.

### Challenge 4 — `retry` with full jitter and an `AbortSignal`

**Prompt:** *"Retry an async operation with exponential backoff. Make it cancellable."*
**Constraints:** full jitter; don't retry what can't succeed; shutdown must interrupt the wait.

```ts
import { setTimeout as sleep } from "node:timers/promises";

export interface RetryOptions {
  attempts?: number;                          // total tries, including the first
  baseMs?: number;
  maxMs?: number;
  retryable?: (err: unknown) => boolean;
  signal?: AbortSignal;
}

export async function retry<T>(
  fn: (attempt: number, signal?: AbortSignal) => Promise<T>,
  { attempts = 4, baseMs = 100, maxMs = 5_000, retryable = () => true, signal }: RetryOptions = {},
): Promise<T> {
  for (let attempt = 1; ; attempt++) {
    signal?.throwIfAborted();
    try {
      return await fn(attempt, signal);
    } catch (err) {
      if (attempt >= attempts || !retryable(err) || signal?.aborted) throw err;
      const cap = Math.min(maxMs, baseMs * 2 ** (attempt - 1));
      await sleep(Math.random() * cap, undefined, { signal });   // FULL jitter; abortable
    }
  }
}
```

**What they're looking for:** `random(0, cap)` and why (synchronised retries re-kill a recovering service; the policy is [API L18](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/18-reliability-and-idempotency.md)); a `retryable` predicate so 4xx and programmer errors fail fast; **an abortable sleep**, so a SIGTERM during a 5 s backoff stops now; the signal passed into `fn` too.
**Common mistakes:** `delay + random(0, 100)`; a plain `setTimeout` sleep that ignores the signal (shutdown waits out the backoff); retrying a non-idempotent call without an idempotency key; retrying the `AbortError` itself. Follow-ups: honour `Retry-After`, and a retry budget.

### Challenge 5 — A line-splitting `Transform`

**Prompt:** *"Turn a byte stream into a stream of lines."*
**Constraints:** chunk boundaries anywhere (including mid-character and between `\r` and `\n`); bounded memory.

```ts
import { Transform, type TransformCallback } from "node:stream";
import { StringDecoder } from "node:string_decoder";

export function splitLines(maxLineLength = 1_000_000): Transform {
  const decoder = new StringDecoder("utf8");   // holds a split multi-byte char until it's whole
  let carry = "";

  return new Transform({
    readableObjectMode: true,                   // bytes in, one string per line out
    transform(chunk: Buffer, _enc: BufferEncoding, cb: TransformCallback) {
      carry += decoder.write(chunk);
      const lines = carry.split(/\r?\n/);
      carry = lines.pop() ?? "";                // the last piece may be half a line
      if (carry.length > maxLineLength) {
        cb(new Error(`line longer than ${maxLineLength} characters`));
        return;
      }
      for (const line of lines) this.push(line);
      cb();
    },
    flush(cb: TransformCallback) {
      carry += decoder.end();
      if (carry.length > 0) this.push(carry.endsWith("\r") ? carry.slice(0, -1) : carry);
      cb();
    },
  });
}

// await pipeline(createReadStream(path), splitLines(), async function* (lines) { … });
```

**What they're looking for:** the carry; `StringDecoder` for a `₹` split across chunks; splitting *after* concatenation, so a `\r` at the end of one chunk and `\n` at the start of the next still form one break; `flush` for a last line with no newline; **a cap**, because one 2 GB "line" is otherwise 2 GB of string. Pushing several lines per chunk is fine: the Transform holds the write callback until the reader catches up, so backpressure survives.
**Common mistakes:** `chunk.toString().split("\n")` per chunk (lines lost at every boundary, mojibake); forgetting `flush`; using it for CSV, where quoted fields can contain newlines and you need a real parser ([Lesson 14](../03-data-and-io/14-composing-streams.md)). Mention `readline` as the built-in alternative.

### Challenge 6 — An LRU cache with TTL

**Prompt:** *"Build an in-memory LRU cache with a max size and a TTL."*
**Constraints:** O(1) get and set; testable without sleeping.

```ts
type Entry<V> = { value: V; expiresAt: number };

export class LruCache<K, V> {
  readonly #max: number;
  readonly #ttlMs: number;
  readonly #now: () => number;
  readonly #map = new Map<K, Entry<V>>();     // insertion order = recency order

  constructor(opts: { max: number; ttlMs: number; now?: () => number }) {
    if (opts.max < 1) throw new RangeError("max must be >= 1");
    this.#max = opts.max;
    this.#ttlMs = opts.ttlMs;
    this.#now = opts.now ?? (() => performance.now());   // monotonic; injectable for tests
  }

  get(key: K): V | undefined {
    const entry = this.#map.get(key);
    if (entry === undefined) return undefined;
    if (entry.expiresAt <= this.#now()) {      // lazy expiry: no timer per entry
      this.#map.delete(key);
      return undefined;
    }
    this.#map.delete(key);                     // move to the most-recent end, O(1)
    this.#map.set(key, entry);
    return entry.value;
  }

  set(key: K, value: V): void {
    this.#map.delete(key);
    this.#map.set(key, { value, expiresAt: this.#now() + this.#ttlMs });
    while (this.#map.size > this.#max) {
      const oldest = this.#map.keys().next();  // first key = least recently used
      if (oldest.done) break;
      this.#map.delete(oldest.value);
    }
  }
}
```

**What they're looking for:** **`Map` iterates in insertion order**, so delete-and-reinsert is an O(1) recency bump and the first key is the eviction victim; lazy expiry; a hard size bound (an unbounded Map is leak pattern #1 in [Lesson 27](../07-production/27-memory-and-gc.md)); a monotonic, injectable clock.
**Common mistakes:** an array with `indexOf` (O(n)); a `setTimeout` per entry (a live timer per key, and handles that keep a CLI alive); `Date.now()` (wall-clock jumps expire everything or nothing). Then the judgement line: an in-process cache is per replica, so with N replicas you have N inconsistent caches.

### Challenge 7 — `withTimeout(promise, ms, signal)`

**Prompt:** *"Reject if the promise doesn't settle within `ms`, or when `signal` aborts."*
**Constraints:** no leaked timers or listeners; a distinguishable timeout error.

```ts
export class TimeoutError extends Error {
  name = "TimeoutError";
}

export function withTimeout<T>(promise: Promise<T>, ms: number, signal?: AbortSignal): Promise<T> {
  return new Promise<T>((resolve, reject) => {
    if (signal?.aborted) {
      reject(signal.reason);
      return;
    }
    const onAbort = (): void => finish(() => reject(signal?.reason));
    const timer = setTimeout(() => finish(() => reject(new TimeoutError(`timed out after ${ms} ms`))), ms);
    signal?.addEventListener("abort", onAbort, { once: true });

    function finish(settle: () => void): void {
      clearTimeout(timer);                               // don't keep the loop alive for ms
      signal?.removeEventListener("abort", onAbort);     // don't leak listeners on a long-lived signal
      settle();
    }

    promise.then(
      (value) => finish(() => resolve(value)),
      (err: unknown) => finish(() => reject(err)),
    );
  });
}

// The better shape: let the timeout cancel the WORK, not just the waiting.
export function withDeadline<T>(
  fn: (signal: AbortSignal) => Promise<T>,
  ms: number,
  signal?: AbortSignal,
): Promise<T> {
  const timeout = AbortSignal.timeout(ms);
  return fn(signal ? AbortSignal.any([signal, timeout]) : timeout);
}
```

**What they're looking for:** clearing the timer on every path (10k calls with a 30 s timeout otherwise leave 10k live timers); removing the abort listener (a per-call listener on a process-lifetime signal is a `MaxListenersExceededWarning` and a leak); the pre-aborted check. Then **the insight that wins the round**: you can't cancel a promise, so `withTimeout` stops the *waiting* while the work carries on holding its socket. `withDeadline` passes the signal into the work itself.
**Common mistakes:** `Promise.race([p, sleep(ms).then(fail)])` with the timer never cleared; ignoring the signal; wrapping the original rejection and losing its stack (use `{ cause }` if you wrap). A testing trap worth knowing: `AbortSignal.timeout()`'s timer doesn't keep the event loop alive, so a script whose only pending work is an `await withDeadline(...)` exits with code 13 before the timeout fires.

### Challenge 8 — A token-bucket rate limiter

**Prompt:** *"In-memory rate limiter: bursts up to `capacity`, refilling at `refillPerSec`."*
**Constraints:** no timers; tell the caller when to retry.

```ts
export type TakeResult = { ok: true } | { ok: false; retryAfterMs: number };

export class TokenBucket {
  readonly #capacity: number;
  readonly #refillPerMs: number;
  readonly #now: () => number;
  #tokens: number;
  #last: number;

  constructor(opts: { capacity: number; refillPerSec: number; now?: () => number }) {
    this.#capacity = opts.capacity;
    this.#refillPerMs = opts.refillPerSec / 1000;
    this.#now = opts.now ?? (() => performance.now());   // monotonic: wall-clock jumps don't mint tokens
    this.#tokens = opts.capacity;                         // start full: allow an initial burst
    this.#last = this.#now();
  }

  take(cost = 1): TakeResult {
    if (cost > this.#capacity) return { ok: false, retryAfterMs: Number.POSITIVE_INFINITY };
    const now = this.#now();
    this.#tokens = Math.min(this.#capacity, this.#tokens + (now - this.#last) * this.#refillPerMs);
    this.#last = now;                                     // lazy refill: no timers at all
    if (this.#tokens >= cost) {
      this.#tokens -= cost;                               // check + update with no await between them
      return { ok: true };
    }
    return { ok: false, retryAfterMs: Math.ceil((cost - this.#tokens) / this.#refillPerMs) };
  }
}
```

**What they're looking for:** lazy refill computed from elapsed time; `retryAfterMs` feeding a `Retry-After` header; and the Node-specific sentence: **the check and the update are atomic because there's no `await` between them** on the one JS thread, so no lock is needed. Per-key buckets live in a bounded map (Challenge 6), or an attacker rotating keys grows it forever.
**Common mistakes:** a `setInterval` refilling every bucket (O(keys) work per tick); `Date.now()`; forgetting that N replicas means N× the limit: the shared version is Redis + Lua ([API L19](https://github.com/Pawar-Pratik/API-mastery/blob/main/04-production/19-rate-limiting.md)).

### Challenge 9 — `singleflight` (deduplicate concurrent calls)

**Prompt:** *"100 concurrent requests miss the cache for the same merchant config. Make them share one load."*
**Constraints:** failures aren't cached; the next call after settling loads again.

```ts
export function singleflight<K, V>(fn: (key: K) => Promise<V>): (key: K) => Promise<V> {
  const inflight = new Map<K, Promise<V>>();

  return (key: K): Promise<V> => {
    const existing = inflight.get(key);
    if (existing) return existing;                       // join the call already in flight

    const p = (async () => fn(key))()                    // a sync throw becomes a rejection
      .finally(() => {
        if (inflight.get(key) === p) inflight.delete(key);   // forget it once settled: no stale cache
      });
    inflight.set(key, p);
    return p;
  };
}

// const loadConfig = singleflight((merchantId: string) => db.merchantConfig(merchantId));
```

**What they're looking for:** the map holds *in-flight promises*, so its size is bounded by concurrency, not by key cardinality; deleting on settle, success **or** failure, so an error isn't served forever; the get-check-set has **no `await` in the middle**, which is the only reason two callers can't both miss. Pair it with Challenge 6 for stampede protection on cache misses.
**Common mistakes:** keeping resolved values (now it's an unbounded cache); deleting only in `.then` (a failure sticks); one caller's `AbortSignal` cancelling the shared load for everyone: each caller should race its own signal against the shared promise. And the scale caveat: it dedupes per process, so N replicas still make N calls.

### Challenge 10 — A minimal worker pool

**Prompt:** *"Run CPU-heavy tasks on N worker threads without blocking the event loop."*
**Constraints:** a fixed pool; a task's error goes to its caller; a crashed worker is replaced.

```ts
import { Worker } from "node:worker_threads";

type Job = { input: unknown; resolve: (value: unknown) => void; reject: (err: unknown) => void };
type Reply = { ok: true; value: unknown } | { ok: false; error: unknown };

export class WorkerPool {
  readonly #file: URL;
  readonly #idle: Worker[] = [];
  readonly #running = new Map<Worker, Job>();
  readonly #queue: Job[] = [];
  #closed = false;

  constructor(file: URL, size: number) {
    this.#file = file;
    for (let i = 0; i < size; i++) this.#spawn();
  }

  run<R>(input: unknown): Promise<R> {
    if (this.#closed) return Promise.reject(new Error("pool is closed"));
    return new Promise<R>((resolve, reject) => {
      this.#queue.push({ input, resolve: resolve as (value: unknown) => void, reject });
      this.#dispatch();
    });
  }

  async close(): Promise<void> {
    this.#closed = true;
    for (const job of this.#queue.splice(0)) job.reject(new Error("pool closed"));
    await Promise.all([...this.#idle, ...this.#running.keys()].map((w) => w.terminate()));
  }

  #spawn(): void {
    const worker = new Worker(this.#file);
    worker.on("message", (reply: Reply) => {
      const job = this.#running.get(worker);
      this.#running.delete(worker);
      if (reply.ok) job?.resolve(reply.value);
      else job?.reject(reply.error);
      this.#idle.push(worker);
      this.#dispatch();
    });
    worker.on("error", (err) => {                      // uncaught inside the worker: it will exit
      this.#running.get(worker)?.reject(err);
      this.#running.delete(worker);
    });
    worker.on("exit", () => {                          // crash, process.exit(), OOM, terminate()
      this.#running.get(worker)?.reject(new Error("worker exited mid-task"));
      this.#running.delete(worker);
      const i = this.#idle.indexOf(worker);
      if (i !== -1) this.#idle.splice(i, 1);
      if (!this.#closed) {
        this.#spawn();                                 // keep the pool at full strength
        this.#dispatch();
      }
    });
    this.#idle.push(worker);
  }

  #dispatch(): void {
    while (this.#idle.length > 0 && this.#queue.length > 0) {
      const worker = this.#idle.pop()!;
      const job = this.#queue.shift()!;
      this.#running.set(worker, job);
      worker.postMessage(job.input);                   // structured clone (or transfer a buffer)
    }
  }
}
```

```ts
// reconcile-worker.ts: errors travel back as messages, so a bad task doesn't cost a worker
import { parentPort } from "node:worker_threads";
import { matchSettlements } from "./match.ts";        // the CPU-bound function

parentPort?.on("message", (input: unknown) => {
  try {
    parentPort?.postMessage({ ok: true, value: matchSettlements(input) });
  } catch (error) {
    parentPort?.postMessage({ ok: false, error });     // Errors survive structured clone
  }
});

// const pool = new WorkerPool(new URL("./reconcile-worker.ts", import.meta.url), config.reconcilerThreads);
```

**What they're looking for:** workers created once; one task per worker, so no task IDs are needed (multiplexing would need them); task errors returned as messages so a worker survives a bad input; `'exit'` handled, the in-flight task rejected and the worker replaced; `close()` for graceful shutdown; size from config, not `os.cpus()` ([Lesson 23](../05-concurrency/23-cluster-and-scaling.md)).
**Common mistakes:** `new Worker()` per task; ignoring `'exit'`, so the pool silently shrinks to zero and every later task hangs; an unbounded queue (cap it and shed load); posting a 200 MB object, whose structured clone runs on the main thread you were trying to protect. Then the production answer: piscina ([Lesson 21](../05-concurrency/21-worker-threads.md)).

---

## 5. Interview delivery

### What to say while you work
- *"Let me tag each line with its queue before I write any output."*
- *"Is this file CommonJS or an ES module? It changes the top-level order of ticks and promises."*
- *"That's inside an I/O callback, so the immediate wins."*
- *"I release the slot in `finally`, so a synchronous throw can't leak it."*
- *"I clear the timer on every path; otherwise each call leaves one behind."*
- *"There's no `await` between the check and the set, so this is atomic on one thread."*

### What losing candidates do
| Mistake | Instead |
|---|---|
| One "callback queue" in their head | Name the phase and the two drain queues every time |
| Forgetting that `emit` and `await b()`'s call are synchronous | Mark sync lines first |
| Claiming certainty about `setTimeout(0)` vs `setImmediate` at top level | "Nondeterministic, and here's why" |
| Happy path only | Name sync throws, cleanup, bounds and cancellation unprompted |
| Silent typing | Narrate; the reasoning is what's scored |
| No test | One assertion per edge you named takes a minute |

---

## 6. How to practise

**Week 1 — Part 1, cold.** Two puzzles a day: answer on paper, narrate the queues out loud, *then* run the file. Every wrong answer goes in `RECALL.md` with the rule you missed ("ticks added while draining run in the same drain").

**Week 2 — write your own puzzles.** Mutate each one: move a line into a `readFile` callback, swap `nextTick` for `queueMicrotask`, rename `.ts` to `.cjs`. Predict, then run. **Generating puzzles is what makes you unpuzzleable.**

**Week 3 — Part 2, timed.** 20 minutes per challenge on a blank file, tests first, out loud. Then compare with the reference: what did you miss in steps 4–6?

**Week 4 — wire them into Ledger.** `pLimit` and `retry` into the webhook sender, `singleflight` + LRU into merchant-config loading, the worker pool into the reconciler. The ones you ship are the ones you'll remember.

### The self-check after each
```text
[ ] Did I tag every line with its queue before writing output?
[ ] Did I ask CommonJS or ESM, main module or callback?
[ ] Did I write the types before the body?
[ ] Did I handle sync throws, double settles and cleanup unprompted?
[ ] Did I place the AbortSignal?
[ ] Could I explain it to a colleague in 60 seconds?
```

---

## What's next

You can predict and produce code under observation. The last format is the senior one: *"it's 3am and p99 tripled; what do you do?"* and *"design a webhook delivery service for 10k events a second, with the capacity maths."* The final lesson covers 35 Node production scenarios and the Node system design round.

Next → **[Lesson 38: Production scenarios & Node system design](38-scenarios-and-system-design.md)**
