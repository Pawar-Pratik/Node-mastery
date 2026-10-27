# Lesson 05 — The event loop, phase by phase

> **Why this lesson exists:** every Node performance question and every "what does this print?" question comes down to the event loop, and most engineers carry the wrong picture of it: one call stack plus one "callback queue". The real loop has six phases, a poll phase that sleeps inside the kernel, and two priority queues that are drained between *every* pair of callbacks. Once you hold the real picture you can predict execution order from first principles, explain why a server froze with CPU at one core, and say why timers are a bad place to keep durable work.

**Time:** ~100 minutes · **Prereq:** Lessons 01–04

---

## 1. The idea in one sentence

> **The event loop isn't a queue. It's a cycle of phases, each with its own queue, plus a poll phase that sleeps in the kernel, and after every single callback Node drains the `nextTick` queue and then the microtask queue. Execution order comes from where you are in that cycle, not from when you scheduled something.**

---

## 2. Why "a callback queue" isn't enough

The picture in most blog posts is a call stack, a queue, and an arrow between them. It can't answer the questions interviewers actually ask:

- Why does `setTimeout(fn, 0)` sometimes run before `setImmediate(fn)` and sometimes after, but *always* after it inside an `fs` callback?
- Why does a promise callback scheduled *after* a timer run *before* it?
- Why does recursing on `process.nextTick` freeze an HTTP server, while recursing on `setImmediate` only pegs the CPU?
- Why does an idle Node process with 10,000 open sockets use 0% CPU?

The loop has four jobs, and one queue can't do all of them. It must **wait on thousands of file descriptors at once without spinning**, **fire timers no earlier than requested**, **give "run this after the current I/O" work a place to run**, and **never interrupt a running callback**. That last one is the rule everything else in this module rests on: **JavaScript callbacks run to completion**. The loop can only reorder *between* callbacks, so between callbacks is where all the interesting machinery lives.

---

## 3. The cycle: six phases

The loop is libuv's `uv_run`. Node runs your main module synchronously first, then enters it:

```text
        main module runs to completion (synchronously)
                          │
                          ▼
   ┌────────────────────────────────────┐
┌─▶│ timers            expired setTimeout / setInterval callbacks
│  └────────────────────────────────────┘
│  ┌────────────────────────────────────┐
│  │ pending callbacks  I/O callbacks deferred from the previous iteration
│  └────────────────────────────────────┘
│  ┌────────────────────────────────────┐
│  │ idle, prepare      internal: libuv and Node housekeeping
│  └────────────────────────────────────┘
│  ┌────────────────────────────────────┐     ┌─────────────────────────────┐
│  │ poll               block for I/O    │◀────│ epoll (Linux)               │
│  │                    run I/O callbacks│     │ kqueue (macOS) / IOCP (Win) │
│  └────────────────────────────────────┘     │ + thread-pool completions   │
│  ┌────────────────────────────────────┐     └─────────────────────────────┘
│  │ check              setImmediate callbacks
│  └────────────────────────────────────┘
│  ┌────────────────────────────────────┐
│  │ close callbacks    'close' after an abrupt destroy()
│  └────────────────────────────────────┘
│                       │
│     any referenced handles or pending requests?
└───────── yes ─────────┘          no ──▶ 'beforeExit' → 'exit'

  After EVERY callback, in EVERY phase:
  drain the process.nextTick queue → drain the microtask queue → repeat until both are empty
```

| Phase | What runs there | Who put it there |
|---|---|---|
| **timers** | Callbacks of timers whose delay has elapsed | `setTimeout`, `setInterval` |
| **pending callbacks** | I/O callbacks libuv deferred to the next iteration (Node's docs give the example of some systems reporting a TCP `ECONNREFUSED` this way) | libuv internals |
| **idle, prepare** | Nothing of yours. Node uses an idle handle to stop poll from sleeping while immediates are queued | Node, libuv |
| **poll** | Incoming connections, socket data, completed `fs`/DNS/crypto work from the thread pool | the kernel, the pool |
| **check** | `setImmediate` callbacks | `setImmediate` |
| **close callbacks** | `'close'` for handles destroyed abruptly (`socket.destroy()`); graceful closes emit via `nextTick` | `uv_close` |

Two details that matter later:

- **The loop has a cached clock.** libuv caches "now" in milliseconds once per iteration, and timer expiry is checked against that loop time, not against a fresh high-resolution reading. That's why timers have millisecond granularity.
- **If you read libuv's source,** you'll find timers run once *before* the `while` loop starts and then at the *bottom* of each iteration, after close callbacks. It's the same circle cut at a different point: timers → pending → idle/prepare → poll → check → close → timers.

---

## 4. The poll phase: where Node sleeps

Poll is where the process spends most of its life, and it's why Node can hold 10,000 idle connections on one thread. Before polling, libuv computes a **timeout**:

```text
poll timeout =
  0         if setImmediate callbacks are queued (Node's idle handle is active),
            or pending callbacks or closing handles exist         → don't sleep
  T         = time until the nearest timer expires                → sleep at most T
  infinite  if there are no timers but other handles are active   → sleep until I/O arrives
```

Then it calls the kernel: `epoll_wait` on Linux, `kevent` on macOS, `GetQueuedCompletionStatusEx` on Windows. **The thread is blocked inside the kernel, consuming no CPU,** until a socket becomes readable, a connection arrives, the timeout expires, or another thread wakes it.

That last case is how the thread pool reports back ([Lesson 07](07-the-thread-pool.md)). A pool thread that finishes an `fs.readFile` posts its result to a queue and signals a wake-up handle that poll is watching. Your callback then runs **on the loop thread, in the poll phase**, never on the pool thread.

Poll runs the callbacks for whatever is ready, in a bounded batch, and moves on. **An idle Node server is a process parked in `epoll_wait` with no timeout**, which is exactly what you want to see in a profiler.

---

## 5. Timers: a lower bound, not an appointment

```ts
setTimeout(fn, 0);            // coerced to 1 ms
setTimeout(fn, 2 ** 31);      // > 2,147,483,647 ms: TimeoutOverflowWarning, coerced to 1 ms 💥
```

- **`setTimeout(fn, 0)` is `setTimeout(fn, 1)`.** Any delay below 1, or above `2^31 − 1` ms (about 24.8 days), becomes 1 ms. Schedule a "30-day reminder" with `setTimeout` and it fires a few milliseconds later, with only a warning on stderr.
- **The delay is a minimum.** A timer becomes *eligible* after `delay` ms. It runs at the next timers phase after that, and the next timers phase only comes when the current callback, and everything else queued ahead of it, finishes. A 50 ms timer behind a 200 ms synchronous callback fires at 200 ms.
- **Node doesn't keep one heap node per timer.** It keeps **one linked list per duration** (every 100 ms timer is appended to the same list, so each list is already sorted by expiry), a priority queue of those lists keyed by their earliest expiry, and **a single libuv timer** set to the soonest one. Creating a timer with a common duration costs an allocation and a list append.

A timer holds the process open while it's pending. `timer.unref()` releases it, which is how you write a metrics flush interval that doesn't stop the process from exiting.

---

## 6. `setImmediate` and close callbacks

`setImmediate(fn)` queues `fn` for the **check phase**, which runs right after poll. Its real meaning is "run this after the current round of I/O callbacks", and that gives it two properties worth memorising:

1. **Inside an I/O callback it always beats a timer.** You're in poll, check is next, and timers come only after the loop wraps around.
2. **It can't starve I/O.** The check phase runs only the immediates that were queued when the phase began. One queued *during* check waits for the next iteration, and poll runs in between.

> **The names are backwards.** `process.nextTick` runs before the loop moves at all, so it's the "immediate" one. `setImmediate` waits for the next check phase, so it's the "next tick" one. Node's docs admit this; the names can't change without breaking npm.

**Close callbacks** run last in each iteration: when a handle is closed abruptly (`socket.destroy()`), its `'close'` event fires here. The order inside a socket callback is deterministic: a `nextTick` first, then an immediate (check phase), then `'close'` (close phase).

---

## 7. Between every callback: `nextTick`, then microtasks

Two queues aren't phases at all. They're drained **after each callback**, wherever it ran:

```text
 run ONE callback (a timer, an I/O callback, an immediate, …)
        │
        ▼
 ┌─▶ drain the process.nextTick queue  (including ticks added while draining)
 │   drain the microtask queue         (promise reactions, await continuations,
 │      │                               queueMicrotask; including ones added while draining)
 │      ▼
 └── either queue non-empty again? ── no ──▶ check for unhandled rejections ──▶ next callback
```

- **`process.nextTick`** is Node's own queue, implemented in JavaScript (`processTicksAndRejections` in Node's source).
- **The microtask queue** belongs to V8. Promise `.then` callbacks, the continuation after an `await`, and `queueMicrotask` all go here ([Lesson 06](06-promises-and-microtasks.md) covers exactly what `await` enqueues).
- **The order is all ticks, then all microtasks, then repeat.** A microtask scheduled from inside a tick waits until the tick queue is empty. A tick scheduled from inside a microtask waits until the microtask queue is empty.
- **Since Node 11 this happens between individual callbacks** inside the timers and check phases, matching browsers. Before Node 11, two timers due in the same phase both ran before either one's microtasks did.

Node's own docs recommend `queueMicrotask()` over `process.nextTick()` for most application code: it's portable, and it doesn't jump ahead of promises. `nextTick` earns its place in library code that must emit an event after a constructor returns but before any I/O, which is the pattern in [Lesson 11](../03-data-and-io/11-eventemitter.md).

### The ESM top-level exception

This is the detail that catches experts. Run the same three lines as `.cjs` and as ESM (every file in this track is ESM):

```ts
process.nextTick(() => console.log("tick"));
Promise.resolve().then(() => console.log("promise"));
console.log("sync");
// CommonJS:        sync, tick, promise
// ES module (.ts): sync, promise, tick
```

The CommonJS loader runs your main module as an ordinary synchronous call; when it returns, Node drains ticks first, as the rule says. The ESM loader evaluates your entry module **from inside its own async code, so the module body runs during a microtask checkpoint**. V8 drains the whole microtask queue, your promise included, before control returns to Node, and only then does Node get to the tick queue. **Inside any callback the rule is the same in both formats.** Interview puzzles are almost always written as CommonJS scripts; say which you're assuming, and you'll stand out.

---

## 8. Ordering rules, worked

| Where you schedule them | `setTimeout(fn, 0)` vs `setImmediate(fn)` | Why |
|---|---|---|
| Main module | **Nondeterministic** | Depends on whether 1 ms has passed by the first timers step |
| Inside an I/O callback | **Immediate first, always** | You're in poll; check comes before the next timers phase |
| Inside a timer callback | **Immediate first, always** | Check comes before the timers phase comes round again |
| Inside an immediate | **Nondeterministic** | The new immediate waits a full iteration, racing the 1 ms timer again |

### Worked example 1: the main module

```ts
setTimeout(() => console.log("timeout"), 0);
setImmediate(() => console.log("immediate"));
```

Phase by phase: the main module schedules a timer due 1 ms from now on the loop clock and queues an immediate. The loop starts with timers. **If at least 1 ms has elapsed since the timer was created**, it's due: `timeout` prints, then poll (which doesn't sleep, because an immediate is queued), then check prints `immediate`. **If less than 1 ms has elapsed**, nothing's due: poll doesn't sleep, check prints `immediate`, and `timeout` prints on the next pass through timers. Whether that 1 ms has passed depends on process start-up work and machine load, so you'll see both orders. Run it 20 times in the Break section.

### Worked example 2: inside an I/O callback

```ts
import { readFile } from "node:fs";

readFile(import.meta.filename, () => {
  setTimeout(() => console.log("timeout"), 0);
  setImmediate(() => console.log("immediate"));
  process.nextTick(() => console.log("tick"));
  Promise.resolve().then(() => console.log("promise"));
  console.log("sync");
});
// sync, tick, promise, immediate, timeout  (every time)
```

1. The callback runs in **poll**. It schedules four things and prints `sync`.
2. The callback returns, so Node drains the tick queue (`tick`) and then the microtask queue (`promise`).
3. Poll finishes, and **check** runs the immediate: `immediate`.
4. Close callbacks has nothing to do. The loop wraps to **timers**: `timeout` (or one iteration later if the 1 ms hasn't elapsed yet, which still leaves it last).

**Moving work inside an I/O callback is how you make `setImmediate` ordering deterministic.**

---

## 9. Starvation: when the loop never reaches poll

Because ticks and microtasks are drained *until empty*, a callback that keeps adding to either queue means the loop never gets past the current callback:

```ts
import { createServer } from "node:http";

createServer((_req, res) => res.end("ok\n")).listen(3000);

setTimeout(() => {
  // ❌ The tick queue is never empty, so poll never runs: the server stops answering.
  const spin = () => process.nextTick(spin);
  // ❌ Same with microtasks:  const spin = () => { void Promise.resolve().then(spin); };
  // ✅ Busy, but fair: each immediate waits for the next check phase, and poll runs in between.
  // const spin = () => setImmediate(spin);
  spin();
}, 5_000);
```

With `nextTick` or the promise version, memory stays flat, one core sits at 100%, and every connection hangs: **no exception, no log line, no crash.** With `setImmediate`, the CPU is still pegged but requests keep being answered, because poll gets a turn on every iteration. That's why the yield-to-the-loop idiom in [Lesson 08](08-blocking-the-loop.md) uses `setImmediate`, never `nextTick` or `await Promise.resolve()`.

---

## 10. Liveness: handles, requests, `ref` and `unref`

The loop keeps going while there's something that could produce a callback. libuv tracks two kinds of thing:

| Kind | Lifetime | Examples | Keeps the process alive while… |
|---|---|---|---|
| **Handle** | Long-lived | TCP server, socket, timer, child process, signal listener | it's active **and referenced** |
| **Request** | One operation | an `fs.readFile`, a `dns.lookup`, a socket write | it's pending |

`.unref()` on a server, socket, timer or child process tells the loop not to count it. A request can't be unref'd; a pending file read always holds the process. To see what's keeping a process alive, `process.getActiveResourcesInfo()` (experimental) lists resource types, for example `['FSReqCallback', 'GetAddrInfoReqWrap', 'Timeout']`. [Lesson 03](../01-foundations/03-the-process.md) covers the "why won't my script exit?" side of this.

---

## 11. The browser's loop, compared

| | Node | Browser (HTML spec) |
|---|---|---|
| Structure | Six fixed phases | Several task queues; the browser picks one task per iteration |
| Microtasks | Drained after every callback (since Node 11) | Drained after every task: the same rule |
| `nextTick` | Yes, ahead of microtasks | Doesn't exist |
| `setImmediate` | Yes (check phase) | Non-standard; don't use it |
| Rendering | None | Rendering steps between tasks: `requestAnimationFrame`, style, layout, paint |
| Timer floor | 1 ms | 4 ms once timers nest more than 5 levels deep |

React's scheduler shows the difference: to yield back to the browser without the timer clamp it posts a task through `MessageChannel`, and when `setImmediate` exists (Node, test environments) it uses that instead. **Same idea, different primitive: a macrotask is how you let everything else run.**

---

## 12. What "non-blocking" actually guarantees

**Guaranteed:** waiting doesn't occupy the JavaScript thread. 10,000 sockets waiting for bytes cost file descriptors and memory, not threads. The kernel and the pool do the waiting.

**Not guaranteed:**
- that your callbacks are short. One 200 ms callback delays every timer, every request and every I/O callback in the process by 200 ms ([Lesson 08](08-blocking-the-loop.md)).
- that `async` code yields. An `async` function that never awaits real I/O runs straight through, as microtasks ([Lesson 06](06-promises-and-microtasks.md)).
- that "async" APIs avoid shared bottlenecks. `fs`, `dns.lookup` and crypto queue for four pool threads ([Lesson 07](07-the-thread-pool.md)).
- any order between different sources. Two sockets' data arrive in whatever order the kernel reports them.

**Throughput in Node is how quickly each callback gives the thread back.** Everything in Module 2 is a consequence of that sentence.

---

## 13. Ledger: why webhook retries don't live in `setTimeout`

Ledger's webhook dispatcher retries a failed delivery 8 times over about 3 days with jitter ([API Lesson 23](https://github.com/Pawar-Pratik/API-mastery/blob/main/05-beyond-rest/23-realtime-and-webhooks.md) has the policy). The tempting implementation is a timer per retry:

```ts
// ❌ Retry state lives in one process's heap.
function scheduleRetry(delivery: Delivery, attempt: number): void {
  setTimeout(() => void deliver(delivery, attempt), backoffMs(attempt));
}
```

Everything about that is wrong for durable work:

- **Every deploy loses every scheduled retry.** A restart, a crash or an OOM kill empties the timer lists. Rolling deploys happen daily; a 3-day schedule never survives one.
- **Memory grows with the backlog.** Each pending retry pins a `Timeout` object, a closure and the delivery payload. A merchant endpoint that's down for a day holds a day's worth of events in your heap.
- **It's invisible and unshareable.** You can't query "what's pending for merchant X", and a second dispatcher replica can't pick up the work.
- **It overflows past 24.8 days**, as §5 showed, if anyone ever lengthens the schedule.

```ts
// ✅ Retry state lives in Postgres; the process only polls for due rows.
await db.query(
  `UPDATE webhook_deliveries
      SET attempt = attempt + 1,
          next_attempt_at = now() + make_interval(secs => $2)
    WHERE id = $1`,
  [delivery.id, backoffMs(attempt) / 1000],
);
```

**The database is the timer wheel.** `webhook_deliveries.next_attempt_at` plus the partial index `WHERE status = 'pending'` lets the dispatcher fetch due rows cheaply, survive restarts, and run as several replicas. The only in-process timer left is the short polling interval. The queue mechanics (`FOR UPDATE SKIP LOCKED`, leases, at-least-once) are [Lesson 26](../06-data-and-jobs/26-background-jobs-and-queues.md).

---

## 14. Production rules

| Rule | Why |
|---|---|
| **Keep every callback short** | The loop is shared by every request; one long callback delays all of them |
| **Yield with `setImmediate`, never with `nextTick` or `await Promise.resolve()`** | Ticks and microtasks drain before the loop moves; only a macrotask lets poll run |
| **Never depend on `setTimeout(0)` vs `setImmediate` order in the main module** | It's nondeterministic; schedule from an I/O callback if order matters |
| **Treat timer delays as minimums and measure elapsed time with `performance.now()`** | A busy loop delays timers; a deadline check must compare clocks, not trust the timer |
| **Never keep durable work in in-memory timers** | Lost on restart, memory per pending item, invisible, overflows past 24.8 days |
| **`unref()` background intervals and timers** | A metrics flush shouldn't keep a finished process alive |
| **Never recurse on `process.nextTick` or promise chains** | It starves I/O with no error and no log line |
| **Prefer `queueMicrotask` to `nextTick` in application code** | Portable, and doesn't jump ahead of promises; Node's docs say the same |
| **State the module format when you reason about top-level ordering** | ESM runs top-level promises before ticks; CommonJS doesn't |

> **Spring equivalent:** WebFlux runs on Netty's event loops: roughly one loop thread per core, each running a select loop and a task queue. The rule is identical, and so is the failure mode: one blocking call on a loop thread stalls every connection assigned to it. Node has a single loop, so it stalls *everything*. See [MVC vs WebFlux vs virtual threads](https://github.com/Pawar-Pratik/SpringBoot/blob/main/05-web/18-mvc-vs-webflux-virtual-threads.md).

---

## 15. Interview traps

**Q1. "Explain the event loop."**
The weak answer is "a call stack and a callback queue": it can't explain a single ordering question. The strong answer: libuv runs a cycle of phases (timers, pending callbacks, idle/prepare, poll, check, close); poll computes a timeout from the nearest timer and blocks in epoll, kqueue or IOCP; and after every callback Node drains the `nextTick` queue, then the microtask queue, until both are empty. Callbacks run to completion, so ordering happens only between callbacks. **Follow-up:** "Where does the thread pool fit in?" Pool threads do the blocking work and signal the loop; the callback runs on the loop thread in the poll phase.

**Q2. "`process.nextTick` vs `setImmediate`?"**
The names are backwards. `nextTick` runs before the loop continues: after the current callback, ahead of promises (outside ESM top level). `setImmediate` runs in the check phase, after the current poll. Recursive `nextTick` starves I/O; recursive `setImmediate` doesn't, because immediates queued during check wait for the next iteration. **Follow-up:** "When would you use `nextTick`?" To defer an event until after a constructor returns but before any I/O, as Node core does; otherwise `queueMicrotask`.

**Q3. "`setTimeout(fn, 0)` or `setImmediate(fn)`: which runs first?"**
"It depends" is only half an answer; name the contexts. In the main module it's nondeterministic, because the timer needs 1 ms (0 is coerced to 1) to have elapsed when the loop first checks timers. Inside an I/O callback or a timer callback, the immediate always wins, because check comes before the next timers phase.

**Q4. "Where do promise callbacks run? Which phase?"**
None. They're microtasks, drained after every callback in every phase, after the `nextTick` queue. The exception is the top level of an ES module: the module body runs during a microtask checkpoint, so its promise callbacks run before its ticks. Since Node 11 the draining happens between individual timers and immediates too, matching browsers.

**Q5. "Can a timer fire early? Late?"**
Not early: the delay is a lower bound on the loop's millisecond clock. Late is normal: a timer runs at the first timers phase after it's due, so any long callback delays it. The real trap is overflow: a delay above `2^31 − 1` ms becomes 1 ms, so a "30-day" timer fires immediately with only a warning.

**Q6. "What does Node do in the poll phase when there's nothing to do?"**
It sleeps in the kernel (`epoll_wait`, `kevent`, `GetQueuedCompletionStatusEx`) with a timeout: zero if immediates or pending callbacks are queued, the time to the nearest timer if there is one, infinite otherwise. That's why an idle server uses no CPU. If there are no referenced handles or pending requests at all, the loop exits instead.

**Q7. "How is Node's loop different from the browser's?"**
The browser has task queues, one task per iteration, a microtask checkpoint after each task, and rendering steps (rAF, style, layout, paint) between tasks. Node has fixed phases, no rendering, `nextTick` ahead of microtasks, and `setImmediate`. The microtask rule is the same since Node 11. Browsers clamp nested timers to 4 ms; Node's floor is 1 ms.

**Q8. "A server stopped answering, CPU was at one core, and there was no error. How can that happen without a long synchronous function?"**
Starvation: something keeps re-filling the `nextTick` or microtask queue (a recursive `nextTick`, a promise chain that schedules itself, an `async` loop over already-resolved values). Each callback is short, but the queues never empty, so poll never runs. A CPU profile shows the same small function on every sample, called from the tick or microtask machinery rather than from an I/O callback. The fix is to yield with `setImmediate`.

**Q9. "What does this print?"**
```ts
setTimeout(() => {
  console.log("t1");
  Promise.resolve().then(() => console.log("p1"));
  process.nextTick(() => console.log("n1"));
}, 0);
setTimeout(() => {
  console.log("t2");
  Promise.resolve().then(() => console.log("p2"));
}, 0);
```
`t1, n1, p1, t2, p2`. Both timers are in the same 1 ms list. `t1` runs; before the next timer, Node drains ticks (`n1`) then microtasks (`p1`); then `t2` runs and its microtask (`p2`) drains. If `t2` happens not to be due yet, it runs one iteration later, and the output is the same. On Node 10 and earlier, with both due, you'd get `t1, t2, n1, p1, p2`, because draining happened only between phases. **Follow-up:** "Why did Node change it?" To match browsers, so code that relies on microtasks running between timers behaves the same in both places.

**Q10. "What does this print?"**
```ts
setImmediate(() => {
  process.nextTick(() => {
    console.log("tick 1");
    process.nextTick(() => console.log("tick 3"));
    Promise.resolve().then(() => console.log("promise 2"));
  });
  Promise.resolve().then(() => {
    console.log("promise 1");
    process.nextTick(() => console.log("tick 4"));
  });
  process.nextTick(() => console.log("tick 2"));
});
```
`tick 1, tick 2, tick 3, promise 1, promise 2, tick 4`. After the immediate returns, the tick queue is `[tick 1, tick 2]` and the microtask queue is `[promise 1]`. Draining ticks: `tick 1` appends `tick 3` to the tick queue and `promise 2` to the microtasks; `tick 2`; `tick 3`. Tick queue empty, so drain microtasks: `promise 1` (which queues `tick 4`), then `promise 2`. The tick queue is non-empty again, so `tick 4`. **The tick queue is drained to empty before any microtask runs, and vice versa.**

**Q11. "What does this print?"**
```ts
setImmediate(() => {
  console.log("A");
  process.nextTick(() => console.log("B"));
  setImmediate(() => console.log("C"));
});
setImmediate(() => {
  console.log("D");
  Promise.resolve().then(() => console.log("E"));
});
```
`A, B, D, E, C`. Both outer immediates are in the check queue when the phase starts. `A` runs, then its tick `B` drains before the next immediate. `D` runs, then its microtask `E`. `C` was queued *during* check, so it waits for the next iteration's check phase. That rule is why `setImmediate` can't starve I/O.

**Q12. "What does this print, and does the answer change in an `.mjs` file?"**
```ts
process.nextTick(() => console.log("tick"));
Promise.resolve().then(() => console.log("promise"));
console.log("sync");
```
As a CommonJS script: `sync, tick, promise`, because the main module returns and Node drains ticks first. As an ES module: `sync, promise, tick`, because the ESM loader evaluates the module during a microtask checkpoint, and V8 drains every microtask before Node processes ticks. Wrap the same code in any callback and both formats print `sync, tick, promise`.

---

## 16. Build & break

### Build — an event-loop tracer

`tools/trace.ts`: a helper that labels every callback with the queue it came from, plus scenario scripts that use it. **Write your predicted output as a comment at the top of each scenario before you run it.**

```ts
// tools/trace.ts
import { readFile } from "node:fs";

let seq = 0;
const log = (kind: string, label: string): void =>
  console.log(`${String(++seq).padStart(2)}  ${kind.padEnd(10)} ${label}`);

type Then = () => void;
export const on = {
  timeout:   (label: string, then?: Then) => setTimeout(() => { log("timers", label); then?.(); }, 0),
  immediate: (label: string, then?: Then) => setImmediate(() => { log("check", label); then?.(); }),
  tick:      (label: string, then?: Then) => process.nextTick(() => { log("nextTick", label); then?.(); }),
  micro:     (label: string, then?: Then) => queueMicrotask(() => { log("microtask", label); then?.(); }),
  io:        (label: string, then?: Then) => readFile(import.meta.filename, () => { log("poll", label); then?.(); }),
};
```

Write at least four scenarios: everything scheduled from the main module; the same set from inside `on.io(...)`; from inside `on.timeout(...)`; and ticks and microtasks nested three levels deep. For each mismatch between your prediction and the output, write one sentence naming the rule you missed. Stretch goal: an `async_hooks.createHook` version that prints resource types (`Timeout`, `Immediate`, `TickObject`, `PROMISE`, `FSREQCALLBACK`). Write with `fs.writeSync(1, …)` inside hooks, because `console.log` there creates async resources and recurses forever, and keep it out of production: Node's docs discourage `createHook`.

### Break — five experiments

1. **The flip.** Save the two-line puzzle from worked example 1 as `classic.ts` and run it 20 times: `for i in $(seq 1 20); do node classic.ts | head -1; done | sort | uniq -c`. You'll get a mix of `timeout` and `immediate`. Explain the 1 ms race out loud.
2. **The fix.** Wrap both calls in `readFile(import.meta.filename, () => { … })` and repeat the loop. Twenty `immediate`s. Then wrap them in a `setTimeout` callback instead: still twenty.
3. **Starvation.** Run the server from §9 with the `nextTick` spin. Before 5 s, `curl -m 3 http://localhost:3000/` answers (use `curl.exe` in PowerShell); after 5 s it times out, and Task Manager shows one core pegged with flat memory. Switch to the promise spin: same result. Switch to `setImmediate`: CPU still pegged, but `curl` answers. That difference is the whole of §6 and §9.
4. **The late timer.** Schedule a 50 ms timer, then busy-wait 200 ms (`const end = Date.now() + 200; while (Date.now() < end) {}`), and print `performance.now()` deltas. It fires at about 200 ms. Now you know why loop delay is the metric [Lesson 08](08-blocking-the-loop.md) builds a meter for.
5. **The 30-day timer.** `setTimeout(fire, 30 * 24 * 3600 * 1000)`. Read the `TimeoutOverflowWarning` and watch it fire within milliseconds. Then write the one-line guard you'd put in a scheduling helper (reject delays above `2 ** 31 - 1`), and the sentence for a code review explaining why the real fix is a `next_attempt_at` column.

### Explain out loud (60–120 seconds)

1. The six phases in order, and what runs in each.
2. How poll computes its timeout, and why an idle server uses no CPU.
3. The between-every-callback rule: all ticks, then all microtasks, repeat; changed in Node 11.
4. `setTimeout(0)` vs `setImmediate` in the main module, in an I/O callback, and in a timer callback, with reasons.
5. Why recursive `nextTick` starves I/O and recursive `setImmediate` doesn't.
6. The ESM top-level exception.
7. Why Ledger's webhook retries live in `next_attempt_at`, not in timers.

---

## What's next

You know *where* promise callbacks run. Next is *what* `async`/`await` actually enqueues: how many microtask ticks an `await` costs, why `return await` changes which `catch` runs, why a rejection handled 40 ms too late kills the process, and how an `async` loop can freeze a server without a single blocking call.

Next → **[Lesson 06: Promises, async/await & the microtask queue](06-promises-and-microtasks.md)**
