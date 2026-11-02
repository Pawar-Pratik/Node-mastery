# Lesson 19 — Frameworks by mechanism: Express, Fastify, NestJS

> **Why this lesson exists:** "Express or Fastify? Why is Fastify faster? What about NestJS?" comes up in nearly every Node interview, and the strong answer is mechanical: what each framework does on every request, what that costs, and what it buys you. You also need NestJS mapped onto your Spring knowledge, including the parts of the analogy that break.

**Time:** ~75 minutes · **Prereq:** Lesson 17

---

## 1. The idea in one sentence

> **A framework is the code that runs between llhttp handing you `(req, res)` and your handler (a router, a chain of functions, a validator and a serializer), so compare frameworks by what that costs per request next to everything else the request does, never by a hello-world benchmark.**

---

## 2. Five jobs, four ways

In [L17](17-http-from-scratch.md) you wrote all of it by hand. Every framework packages the same five jobs, with different mechanics:

| Job | `node:http` (L17) | Express 5 | Fastify 5 | NestJS |
|---|---|---|---|---|
| **Route** | your linear table | layer stack, regex per layer, **scanned in order** | **radix tree** (find-my-way) | decorators → the underlying platform's router |
| **Validate input** | your code | bring your own (zod) | JSON Schema, **compiled by ajv at startup** | pipes (`ValidationPipe`) |
| **Cross-cutting** (auth, logging) | wrap the handler | middleware + `next()` | **hooks** at named lifecycle points | middleware, guards, interceptors |
| **Map errors** | top-level `catch` | 4-argument error middleware | `setErrorHandler` | exception filters |
| **Serialize** | `JSON.stringify` | `JSON.stringify` (`res.json`) | **fast-json-stringify, compiled from the response schema** | the platform's |

The bold cells are where the performance differences come from, and §7 is about how much they matter.

---

## 3. Express: an array and an index

Express's core is small enough to rewrite in fifteen lines, and doing that once explains every Express bug you'll meet:

```ts
import type { IncomingMessage, ServerResponse } from "node:http";
type Layer = { matches(req: IncomingMessage): boolean; fn: (...args: unknown[]) => unknown };

function dispatch(stack: Layer[], req: IncomingMessage, res: ServerResponse, done: (err?: unknown) => void) {
  let i = 0;
  const next = (err?: unknown): void => {
    const layer = stack[i++];
    if (!layer) return done(err);                       // fell off the end: Express's final handler (404 or 500)
    if (!layer.matches(req)) return next(err);          // ← linear scan: every layer's path regex, in order
    const isErrorHandler = layer.fn.length === 4;       // ← ARITY decides what an error handler is
    if ((err !== undefined) !== isErrorHandler) return next(err);   // errors skip normal layers and vice versa
    try {
      const out = isErrorHandler ? layer.fn(err, req, res, next) : layer.fn(req, res, next);
      if (out instanceof Promise) out.catch(next);      // ← the Express 5 change, in one line
    } catch (e) { next(e); }                            // synchronous throws were always caught
  };
  next();
}
```

What falls out of that:

- **Middleware is an array plus an index.** `app.use()` and `app.get()` push layers; a request walks them in registration order. **Order is the design**: auth before parsing, error handler last.
- **Route matching is a linear scan of regexes.** The 300th route tests up to 300 patterns per request. Netflix's "Node.js in Flames" (2014) is the canonical evidence: flame graphs showed Express route lookups getting slower over time because a reload bug kept appending handlers to the route array.
- **An error handler is any function with exactly four parameters.** Delete the "unused" `next` from `(err, req, res, next)` to satisfy a linter and it silently becomes ordinary middleware that errors skip.
- **Per request, Express swaps the prototypes of Node's `req` and `res`** to add `res.json()`, `req.params` and friends. Convenient, and not free.
- **Express ships no validation, serialization or logging.** That's its flexibility and its cost: each is a middleware you choose, order and pay for.

### What Express 5 changed (the parts you can rely on)

| Change | Consequence |
|---|---|
| A rejected promise from a handler or middleware goes to `next(err)` | `async` handlers no longer need a wrapper; in Express 4 the rejection was unhandled and, since Node 15, **crashed the process** |
| Path syntax (path-to-regexp v8): wildcards must be named (`/*splat`), optional segments use braces (`/:file{.:ext}`), no inline regex | Old route strings throw at startup, which beats silently mismatching |
| `req.body` is `undefined` when no parser ran (no longer `{}`) | Code that assumed an object fails loudly |
| The default query parser is `"simple"` (no nested objects) | `?a[b]=1` no longer becomes `{ a: { b: "1" } }`: less type confusion from query strings |
| Removed: `req.param()`, `app.del()`, `res.sendfile()` | Use `req.params`/`req.query`/`req.body`, `app.delete()`, `res.sendFile()` |

**What Express 5 still doesn't catch:** an error thrown inside a callback the handler scheduled (a `setTimeout`, an EventEmitter listener, a stream `'data'` handler). That never touches the returned promise; it's an uncaught exception and the process exits ([L09](../02-event-loop/09-errors-and-crashing.md)).

---

## 4. Fastify: compile everything at startup

Fastify's design principle is to move work from request time to boot time.

**Routing: a radix tree.** Routes are stored by shared prefixes (`/v1/payments`, `/v1/payments/:id`, `/v1/payouts` share `/v1/pa`), so a lookup walks the path's characters, and its cost depends on the path's length, not on how many routes you have.

**Validation: ajv compiles each route's JSON Schema into a JavaScript function** when the app starts. At request time, validating a body is a call to a specialised function, not an interpretation of a schema.

**Serialization: often the real bottleneck.** `JSON.stringify` knows nothing about your object, so it discovers every key and inspects every value's type at runtime, for every object in a 500-row list. fast-json-stringify compiles the response schema into code that writes the known fields in a known order. It has a second, underrated property: **fields not in the response schema are not written.** That's the output allowlist the API track insists on, enforced by the serializer, so an internal `processor_ref` added to a row can't leak.

**Encapsulation: plugins form a tree.** `register()` creates a child context (managed by avvio, Fastify's plugin loader). Decorators, hooks and plugins registered inside it are visible to that subtree only:

```text
root ── decorate("db") , decorateRequest("merchantId", null)    ← visible everywhere
 ├── register(health)                                          ← no auth hook here
 └── register(v1, { prefix: "/v1" })
       addHook("onRequest", authenticate)                     ← applies ONLY inside this subtree
       ├── register(paymentRoutes)
       └── register(refundRoutes)
```

That's how you get "auth on `/v1`, not on `/healthz`" without ordering tricks. Wrapping a plugin in `fastify-plugin` opts it out of encapsulation, so its decorators reach the parent; that's how shared things like the DB pool are registered.

**Hooks: named points in a fixed lifecycle**, instead of one ordered array:

```text
onRequest → preParsing → (parse) → preValidation → (validate) → preHandler → handler
          → preSerialization → (serialize) → onSend → (write) → onResponse          onError on failure
```

`onRequest` runs *before the body is parsed*, so authentication there means anonymous clients never cost you a parse. Add request properties with `decorateRequest("merchantId", null)` up front rather than attaching them ad hoc: every request object then has the same shape, which V8 rewards ([L28](../07-production/28-performance-and-v8.md)). Logging is built in: `Fastify({ logger: true })` gives you pino and a per-request child logger at `request.log` ([L29](../07-production/29-observability.md)). The default `bodyLimit` is 1 MiB.

**The gotcha: Fastify's default ajv options coerce types and remove additional properties.** With `additionalProperties: false`, an unknown key is silently *stripped*, not rejected, and `"amount_minor": "500"` is coerced to `500`. If your contract is strict-in (the API track's `.strict()` schemas), set `ajv.customOptions` to `removeAdditional: false` and `coerceTypes: false`.

---

## 5. Hono: the Web-standard option

Hono is written against the Fetch API's `Request` and `Response`, so the same app runs on Cloudflare Workers, Deno, Bun and Node. On Node it runs through an adapter (`@hono/node-server`) that turns each `IncomingMessage` into a `Request` and each `Response` back into a `ServerResponse`. That conversion is overhead a Node-native framework doesn't pay. **Choose it for portability** (edge plus Node, one codebase), not to be faster on Node.

---

## 6. NestJS: Spring's architecture on Node's runtime

NestJS gives you modules, providers, a DI container, decorators, and a per-request pipeline that will feel familiar:

```text
middleware → guards → interceptors (before) → pipes → handler → interceptors (after)
                                                                 └ exception filters on any throw
```

It runs **on top of** Express (the default) or Fastify, so its per-request cost is the platform's plus its own layers.

```ts
// ⚠️ NOT runnable with `node file.ts`. Decorators and emitDecoratorMetadata are not erasable syntax,
// and the constructor's `private readonly` is a parameter property: this needs tsc or SWC first.
@Controller("v1/payments/:id/refunds")
@UseGuards(ApiKeyGuard, new ScopeGuard("refunds:write"))
export class RefundsController {
  constructor(private readonly refunds: RefundsService) {}     // resolved from design:paramtypes metadata

  @Post()
  @HttpCode(201)
  create(@Param("id") id: string, @Body(new ZodPipe(RefundBody)) body: RefundInput, @MerchantId() merchantId: string) {
    return this.refunds.create(merchantId, id, body);
  }
}
```

**How the DI works:** with `emitDecoratorMetadata`, the TypeScript compiler writes each decorated class's constructor parameter types into runtime metadata (`design:paramtypes`), and Nest reads it through `reflect-metadata` to know that `RefundsController` needs a `RefundsService`. That's why Nest needs a build step: Node's type stripping removes types and refuses decorators, so there are no types left to emit ([TS L01](https://github.com/Pawar-Pratik/TypeScript-mastery/blob/main/01-foundations/01-why-typescript-and-erasure.md)).

| Spring | NestJS | Express 5 | Fastify 5 |
|---|---|---|---|
| Component scan + `@Configuration` | `@Module({ imports, providers, controllers, exports })` | plain modules | plugins |
| `@Service` bean, singleton | `@Injectable()` provider, singleton by default | a module-level instance | `decorate()` |
| Constructor injection | Constructor injection via `design:paramtypes` | `import`, or pass arguments | plugin options, decorators |
| `@RestController` + `@PostMapping` | `@Controller` + `@Post` | `router.post` | `app.post` with a schema |
| Servlet `Filter` | Middleware | `app.use` | `onRequest` hook |
| Security filter chain / `HandlerInterceptor.preHandle` | Guard | middleware before the route | `onRequest` / `preHandler` |
| AOP `@Around` | Interceptor (RxJS `Observable` around the handler) | wrap the handler | paired hooks (`preHandler` + `onSend`) |
| `@Valid` + Bean Validation | Pipe (`ValidationPipe`, or a zod pipe) | zod middleware | JSON Schema, compiled |
| `@ControllerAdvice` / `@ExceptionHandler` | Exception filter | 4-argument error middleware | `setErrorHandler` |
| `@Scope("request")` | `Scope.REQUEST` | — | — |
| `@Transactional` (a proxy) | **nothing built in** | nothing | nothing |

Where the analogy breaks, which is what interviewers probe:

- **Request scope bubbles up.** A provider that depends on a request-scoped provider becomes request-scoped too, so one request-scoped logger can turn a whole dependency chain into per-request allocations. Carry request context with `AsyncLocalStorage` instead ([L25](../06-data-and-jobs/25-databases-from-node.md), [L29](../07-production/29-observability.md)).
- **No transactional proxies.** You still pass a client or use ALS, exactly as without Nest.
- **The runtime is still one event loop.** Nest's structure doesn't change anything in Modules 2 and 3: a blocking guard blocks every request in the process.
- **The cost:** a compile step (no native `.ts`), reflection metadata, deep stack traces through framework layers, and startup time spent building the module graph. **What it buys:** enforced structure and a DI-friendly testing module, which pays off for large teams.

> **Spring equivalent:** DI resolution and constructor injection in [Spring L07](https://github.com/Pawar-Pratik/SpringBoot/blob/main/02-spring-core/07-dependency-injection-deep.md); filters vs interceptors (the middleware vs hooks vs guards question) in [Spring L17](https://github.com/Pawar-Pratik/SpringBoot/blob/main/05-web/17-openapi-cors-filters-interceptors.md).

---

## 7. Benchmarks, honestly

A hello-world benchmark measures one thing: **framework overhead when the handler does nothing**, because then the framework *is* the CPU per request. Fastify wins those clearly, for the reasons in §4. Then ask what your request actually spends CPU on: body parsing, validation, auth crypto, the DB driver encoding and decoding, business logic, serialization, logging, and the framework.

Illustrative arithmetic (made-up but plausible numbers; measure your own with [L28](../07-production/28-performance-and-v8.md)):

```text
                      framework   everything else   CPU / request   max req/s on one core
Framework A           40 µs       300 µs            340 µs          ≈ 2,940
Framework B           10 µs       300 µs            310 µs          ≈ 3,230      → ~10%, not 4×
```

And often CPU isn't the ceiling at all. **Little's law** ([L10](../02-event-loop/10-cancellation-and-concurrency.md)): with `pg.Pool({ max: 10 })` and a 5 ms query per request, the pool completes at most `10 / 0.005 s = 2,000` queries per second per process. No framework serves more than 2,000 such requests per second from that process; they all queue for a connection.

**When the framework does matter:** high request rates with tiny handlers (a cache-hit read, a BFF fan-out, a proxy), large list responses where serialization dominates, and any time a flame graph shows routing, validation or serialization as a wide plateau. Then Fastify's compiled validators and serializers are a real, measurable win.

---

## 8. How to choose

| Situation | Choose | Because |
|---|---|---|
| Existing Express codebase and middleware you depend on | **Express 5** | The upgrade is small; a rewrite buys ~nothing until a profile says otherwise |
| New service, high RPS, schema-first contract | **Fastify** | Compiled validation and serialization, encapsulation, built-in logging |
| Must run on edge runtimes and Node from one codebase | **Hono** | Web-standard `Request`/`Response` |
| Large team, Spring/Angular background, wants enforced structure | **NestJS** | Modules and DI make big codebases uniform; accept the build step |
| A handful of internal routes, minimal dependencies | **`node:http`** | You know exactly what it does (L17) |

---

## 9. Ledger: one route, two frameworks, one decision

`POST /v1/payments/:id/refunds` with auth, validation, idempotency and error mapping, written both ways.

```ts
// routes/refunds.express.ts (Express 5)
import express, { Router } from "express";
import { z } from "zod";
import { requireScope } from "../middleware/auth.ts";
import { idempotent } from "../middleware/idempotency.ts";
import { refundService, toRefundDto } from "../services/refunds.ts";

const RefundBody = z.object({
  amount_minor: z.number().int().positive(),
  reason: z.enum(["requested_by_customer", "duplicate", "fraudulent"]).optional(),
}).strict();                                                  // strict-in: unknown keys → 422

export const refunds = Router();
refunds.post("/payments/:id/refunds",
  requireScope("refunds:write"),
  express.json({ limit: "100kb" }),                           // parser scoped to the route that needs it
  idempotent("POST /payments/:id/refunds", { required: true }),   // API L18; needs the parsed body
  async (req, res) => {                                       // a rejection here reaches problemHandler
    const input = RefundBody.parse(req.body);                 // ZodError → 422 in problemHandler
    const refund = await refundService.create(req.merchantId, req.params.id, input);  // req.merchantId: augmented (TS L14)
    res.status(201).location(`/v1/refunds/${refund.id}`).json(toRefundDto(refund));
  });

// app.ts: the order IS the design
app.use(requestId(), httpLogger(), authenticate);             // cheap auth BEFORE any body is parsed
app.use("/v1", refunds);
app.use(problemHandler);                                      // last, and declared with all 4 parameters
```

```ts
// routes/refunds.fastify.ts (Fastify 5)
import type { FastifyInstance } from "fastify";
import { requireScope } from "../hooks/auth.ts";
import { idempotent } from "../hooks/idempotency.ts";
import { refundService, toRefundDto } from "../services/refunds.ts";
import type { RefundInput } from "../services/refunds.ts";

const RefundBody = {
  type: "object", additionalProperties: false, required: ["amount_minor"],
  properties: {
    amount_minor: { type: "integer", minimum: 1 },
    reason: { enum: ["requested_by_customer", "duplicate", "fraudulent"] },
  },
} as const;
const RefundDto = {
  type: "object",
  properties: { id: { type: "string" }, amount_minor: { type: "integer" },
                status: { type: "string" }, created_at: { type: "string" } },
} as const;

export async function refundRoutes(app: FastifyInstance) {
  app.post<{ Params: { id: string }; Body: RefundInput }>("/payments/:id/refunds", {
    schema: { body: RefundBody, response: { 201: RefundDto } },  // compiled at boot: validator + serializer
    preHandler: [requireScope("refunds:write"), idempotent("POST /payments/:id/refunds")],
  }, async (req, reply) => {
    const refund = await refundService.create(req.merchantId, req.params.id, req.body);
    reply.code(201).header("location", `/v1/refunds/${refund.id}`);
    return toRefundDto(refund);                                // undeclared fields are never written
  });
}

// app.ts
const app = Fastify({ logger: true, bodyLimit: 100 * 1024,
  ajv: { customOptions: { removeAdditional: false, coerceTypes: false } } });   // match zod's strictness
app.decorateRequest("merchantId", null);
app.register(async (v1) => {
  v1.addHook("onRequest", authenticate);                       // before parsing, and only under /v1
  await v1.register(refundRoutes);
}, { prefix: "/v1" });
app.setErrorHandler(problemHandler);                           // err.validation is set for schema failures → 422
```

Same behaviour, different mechanics: Express gets its order from registration and its validation from a library you call; Fastify gets its order from the lifecycle and its validation and serialization from schemas compiled at startup.

**The decision: Ledger stays on Express 5.** The API track's middleware (idempotency, raw-body capture for HMAC, the OpenAPI validator) is Express middleware; continuity with that track is worth more than a framework speedup nobody has measured; and in Ledger's request profile the database, crypto and logging dominate.

**The triggers that would justify moving to Fastify,** named in advance so the decision isn't a matter of taste:

1. A CPU profile of `POST /v1/payments` under load ([L28](../07-production/28-performance-and-v8.md)) shows routing, validation or serialization among the widest plateaus.
2. A hot, cheap endpoint (a cached balance read, say) needs more requests per second per pod, and CPU is what limits it.
3. The team wants one schema per route to drive validation, serialization and the OpenAPI document.

Migration would go route by route: `@fastify/express` can mount the existing middleware while routes move.

---

## 10. Production rules

| Rule | Why |
|---|---|
| **Choose by a profile and the constraints, not a hello-world benchmark** | Framework overhead is one slice of CPU per request, and often the pool is the ceiling |
| **Authenticate before parsing bodies** (Express order; Fastify `onRequest`) | Anonymous clients shouldn't be able to make you buffer and parse |
| **Scope body parsers to the routes that need them, with limits** | A global parser parses every body for every route |
| **Register the error handler last, with all four parameters** | Express identifies error middleware by arity and position |
| **Don't rely on Express 5 for errors thrown in callbacks** | Only the returned promise is caught; a throw in a timer or listener crashes the process |
| **Fastify: set `removeAdditional: false` and `coerceTypes: false` for strict-in contracts** | The defaults strip unknown fields and coerce types silently |
| **Fastify: declare response schemas** | Faster serialization and an enforced output allowlist |
| **Fastify: `decorateRequest` with an initial value; share via `fastify-plugin` deliberately** | Stable object shapes; encapsulation is the default, sharing is the exception |
| **NestJS: avoid `Scope.REQUEST`; carry context with ALS** | Request scope propagates up the dependency chain |
| **NestJS: budget for the build step** | Decorators and metadata aren't erasable; no native `.ts` |
| **Keep handlers thin: parse → service → DTO** | The framework becomes swappable, which is what makes the triggers above actionable |

---

## 11. Interview traps

**Q1. "Why is Fastify faster than Express?"**
Mechanically: a radix-tree router whose lookup cost doesn't grow with the route count, versus Express's in-order scan of path regexes; validation compiled by ajv at startup; response serialization compiled from schemas by fast-json-stringify instead of `JSON.stringify` discovering each object at runtime; and no per-request prototype swapping on Node's `req`/`res`. Then the honest part: that gap is clearest in hello-world benchmarks, and shrinks as the rest of the request's CPU grows.
**Follow-up:** "So should we migrate?" Only if a profile shows framework work is a large share of CPU, or the RPS target is CPU-bound on cheap endpoints. Name the trigger.

**Q2. "How does Express middleware work?"**
An array of layers and an index. Each request walks the array in registration order; `next()` advances the index; a layer runs if its path and method match. `next(err)` skips to the next layer that declares four parameters. That's why order is semantics, and why an error handler missing its `next` parameter stops being one.

**Q3. "How do errors propagate in Express 5?"**
Synchronous throws in a handler are caught and passed to `next(err)`, as in Express 4. New in 5: a rejected promise returned by a handler or middleware is too, so `async` handlers work without wrappers. Not covered: errors thrown inside callbacks the handler scheduled (timers, emitters, stream events). Those are uncaught exceptions and take the process down. In Express 4, an `async` handler's rejection was unhandled and crashed the process under Node's default.

**Q4. "NestJS vs Spring?"**
Same architecture: modules, singleton providers, constructor injection, guards/interceptors/pipes/filters mapping to filters, AOP, validation and `@ControllerAdvice`. Differences: DI reads TypeScript-emitted metadata, so it needs a build step; there's no `@Transactional` proxy; request scope bubbles up the dependency chain; and underneath is one event loop, so a blocking guard blocks everything. Nest is a structure choice, not a performance choice.

**Q5. "Does the framework matter for performance?"**
Less than people think. Requests per second per core ≈ 1 / CPU per request, and the framework is one term in that sum next to parsing, crypto, the driver, serialization and logging. And a 10-connection pool with 5 ms queries caps you at 2,000 queries per second per process whatever the framework. It matters for high-RPS tiny handlers and serialization-heavy responses, which a profile will show you.

**Q6. "What is Fastify's encapsulation?"**
Each `register()` creates a child context; decorators, hooks and plugins added inside are visible only to that subtree. It lets you scope an auth hook to `/v1` without touching `/healthz`, and prevents one plugin's decorations leaking into another's. `fastify-plugin` opts a plugin out, so shared infrastructure (the DB pool) reaches the parent.

**Q7. "My Express error handler never runs. Why?"**
Three usual causes: it's registered before the routes (the request never reaches it), it declares three parameters (so it's ordinary middleware), or the error was thrown in a callback outside the handler's promise and became an uncaught exception instead of reaching `next`.

**Q8. "A Fastify route has `additionalProperties: false`, yet a body with extra fields gets a 201. Why?"**
Fastify's default ajv configuration uses `removeAdditional`, so unknown properties are stripped rather than rejected, and `coerceTypes`, so strings become numbers. If the contract is strict-in, turn both off.

**Q9. "Express or NestJS for a new team of 15 people coming from Spring?"**
Name the trade: Nest gives uniform structure and DI-based testing that a large team benefits from, at the cost of a build step, metadata reflection, and a layer of indirection when debugging. Express with a firm layering convention (routes → services → repos) gets most of the structure with fewer moving parts. Either is defensible; what isn't is choosing Nest because "it's faster" or Express because "it's simpler" without naming what you give up.

---

## 12. Build & break

### Build — the refund route in both frameworks, one test suite

1. `apps/api-express/` and `apps/api-fastify/` serving the §9 route from the **same** `services/refunds.ts`, with `authenticate`, `requireScope`, `idempotent` and `problemHandler` written once per framework.
2. One `node:test` suite that starts each app on an ephemeral port (`listen(0)`) and runs identical cases against both: 201 with `Location`; 422 for an unknown field; 422 for `amount_minor: 0`; 422 for `"500"` as a string; 401 without a key, **with a counter proving the body was never parsed**; 403 for a key without `refunds:write`; an idempotent replay returning the stored response; and byte-identical `application/problem+json` error shapes.
3. When both pass, write down the triggers from §9 in Ledger's decision doc.

### Break — three experiments

1. **Async errors, then and now.** In a scratch folder, install `express@4` and add an `async` handler that throws. Request it: no response, and the process exits with code 1 on the unhandled rejection, killing every other in-flight request. Run the same handler on Express 5: a 500 problem from your error middleware. Then, in Express 5, throw inside a `setTimeout` callback in the handler: the process dies again. Explain why to yourself before reading §3 again.
2. **Middleware order.** (a) Move `express.json({ limit: "100kb" })` to the app level, before `authenticate`. Send a 200 KB body with no API key: you get 413 instead of 401, proof that you buffered an anonymous client's body. Restore the order: 401, no parse. (b) Register `problemHandler` before the routes and throw from a handler: Express's default handler answers instead of your problem+json. (c) Delete `next` from `problemHandler`'s parameters: errors now skip it entirely.
3. **The benchmark that matters.** Add `GET /ping` returning `{ ok: true }` to both apps and run `npx autocannon -c 100 -d 20 http://localhost:3001/ping` against each; record requests per second and p99. Then make both handlers run `SELECT pg_sleep(0.005)` through a `pg.Pool({ max: 10 })`. **Predict before you run:** Little's law says about 2,000 req/s. Watch both frameworks pin near the same ceiling with the gap gone. Raise `max` to 50 (ceiling 10,000) and find out what limits you next.

### Explain out loud (60–120 seconds)

1. Express's dispatch loop: the array, the index, arity, and what Express 5 added.
2. The three things Fastify compiles at startup, and why serialization is often the real cost.
3. Encapsulation, and how it scopes an auth hook.
4. NestJS mapped onto Spring, and the three places the analogy breaks.
5. The Little's-law argument for why framework choice rarely sets your throughput, and when it does.

---

## What's next

Every request so far has been short-lived. The last lesson of the module is about connections that stay open for hours: an SSE feed for the Console and the WebSocket alternative, each needing heartbeats, per-client backpressure, cleanup, and fan-out across instances that don't share memory.

Next → **[Lesson 20: Real-time servers: SSE & WebSockets](20-realtime-servers.md)**
