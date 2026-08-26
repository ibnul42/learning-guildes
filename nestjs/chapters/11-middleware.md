# Chapter 11 — Middleware

## Learning Objectives

- Write class-based and functional middleware, and choose between them
- Bind middleware to routes with `configure()`, `forRoutes()`, and `exclude()`
- Explain what middleware can do that guards cannot, and what it fundamentally cannot
- Control execution order across several middleware and across modules
- Recognise when a requirement belongs in a guard or interceptor instead

---

## 11.1 Middleware Is the Platform Layer

Middleware is the one pipeline component that is not really Nest's — it is Express (or Fastify) middleware, running on the raw request before Nest has decided anything.

```ts
// src/common/logger.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction): void {
    console.log(`${req.method} ${req.originalUrl}`);
    next();                      // MUST be called, or the request hangs forever
  }
}
```

Three properties define it, and all three follow from running before route resolution:

**It has no `ExecutionContext`.** Middleware cannot know which controller or handler will serve the request, so it cannot read `@Public()`, `@Roles()`, or any other metadata (→ [Ch 10](10-request-lifecycle.md)).

**It works with the raw platform objects.** `req` and `res` are Express's, not an abstraction — which makes middleware the least portable component if you ever move to Fastify.

**It must call `next()`.** Forgetting is not an error; the request simply hangs until the client times out. This is the single most common middleware bug.

```ts
use(req: Request, res: Response, next: NextFunction): void {
  if (!req.headers['x-api-key']) {
    res.status(401).json({ message: 'Missing API key' });
    return;                     // ← ending the response instead of calling next()
  }
  next();
}
```

Ending the response without calling `next()` is legitimate — that is how middleware short-circuits. The bug is doing *neither*.

> 💼 **Interview** — "When would you use middleware over a guard?" The answer is: when the concern is genuinely handler-agnostic and platform-level — parsing cookies, correlation ids, raw-body capture, request logging. Anything that needs to know *which* route is being called belongs in a guard.

---

## 11.2 Class vs Functional Middleware

Functional middleware is a plain function:

```ts
// src/common/correlation.middleware.ts
import { Request, Response, NextFunction } from 'express';

export function correlationId(req: Request, res: Response, next: NextFunction): void {
  const id = (req.headers['x-correlation-id'] as string) ?? randomUUID();
  res.setHeader('x-correlation-id', id);
  next();
}
```

| | Class middleware | Functional middleware |
|---|---|---|
| Dependency injection | Yes | **No** |
| Lifecycle hooks | Yes | No |
| Boilerplate | More | Minimal |
| Use when | it needs a service | it is self-contained |

The rule is simple: **if it needs an injected dependency, it must be a class.** A functional middleware cannot reach `ConfigService` or a logger provider — it is not constructed by the container.

Both are bound the same way:

```ts
consumer.apply(LoggerMiddleware, correlationId).forRoutes('*splat');
```

---

## 11.3 Binding Middleware

Middleware cannot be bound with a decorator. It is configured in a module implementing `NestModule`:

```ts
// src/app.module.ts
import { MiddlewareConsumer, Module, NestModule, RequestMethod } from '@nestjs/common';

@Module({ controllers: [TasksController], providers: [LoggerMiddleware] })
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer): void {
    consumer
      .apply(LoggerMiddleware)
      .exclude(
        { path: 'health', method: RequestMethod.GET },
        { path: 'metrics', method: RequestMethod.ALL },
      )
      .forRoutes('*splat');
  }
}
```

`forRoutes()` accepts three shapes, and the third is the most maintainable:

```ts
.forRoutes('tasks')                                        // a path prefix
.forRoutes({ path: 'tasks', method: RequestMethod.POST })  // path + method
.forRoutes(TasksController)                                // a controller class
```

Passing the controller class means the binding follows the controller if its prefix changes — a string path silently stops matching.

Several middleware can be applied at once, and **they run in the order listed**:

```ts
consumer.apply(correlationId, LoggerMiddleware, rateLimiter).forRoutes('*splat');
// correlationId → LoggerMiddleware → rateLimiter
```

Chained `apply()` calls run in the order the `configure` method executes them, and across modules, in module-resolution order — which is why order-sensitive middleware belongs in one `configure` call rather than scattered.

**Global middleware** bypasses `configure()` entirely:

```ts
// src/main.ts — runs before ALL module-bound middleware
app.use(helmet());
app.use(cookieParser());
```

> ⚠️ **Gotcha** — NestJS 11 runs Express 5, where `forRoutes('*')` is invalid. Use `forRoutes('*splat')` or `forRoutes({ path: '*splat', method: RequestMethod.ALL })`. This is a common upgrade failure, and the error names `path-to-regexp` rather than your module (→ [Ch 1](01-what-nestjs-is.md)).

---

## 11.4 What Middleware Is Uniquely Good At

Despite its limitations, four jobs genuinely belong here.

**Raw body capture**, needed for webhook signature verification. The signature is computed over the exact bytes, and by the time a pipe sees the body it has been parsed and re-serialised:

```ts
app.use(
  '/webhooks/stripe',
  express.raw({ type: 'application/json' }),   // Buffer, not parsed JSON
);
```

**Correlation ids and request context**, which must be established before anything else runs so every downstream log carries them (→ [Ch 9](09-scopes-lifecycle.md)).

**Third-party Express middleware** — Helmet, `cookie-parser`, `compression`, session handling. These are written against Express, and middleware is where they fit.

**Anything needing the response stream itself**, such as compression, which must wrap `res.write`.

What middleware is *bad* at is anything requiring knowledge of the target route. Authentication is the classic mistake:

```ts
// ❌ Cannot see @Public(); the allowlist drifts from the decorators
const PUBLIC_PATHS = ['/auth/login', '/auth/register', '/health'];
export function auth(req: Request, res: Response, next: NextFunction) {
  if (PUBLIC_PATHS.includes(req.path)) return next();
  // …
}
```

Every new public route requires editing a list in a different file from the route. A global guard reading `@Public()` with `Reflector` keeps the decision next to the route (→ [Ch 12](12-guards.md)).

---

## 11.5 Common Mistakes and Best Practices

**Mistake: forgetting `next()`.** The request hangs until the client times out, with no error logged anywhere.

**Mistake: calling `next()` after sending a response.** Express warns about setting headers after they are sent, and the downstream chain runs against a finished response.

**Mistake: authentication in middleware.** It cannot read route metadata, so `@Public()` is invisible and you maintain a parallel allowlist.

**Mistake: `forRoutes('*')` on Nest 11.** Express 5 requires a named wildcard — `'*splat'`.

**Mistake: expecting DI in functional middleware.** It is not container-constructed; convert it to a class.

**Mistake: heavy work in middleware.** It runs on every matched request before any authorization check, so an expensive lookup here is done for callers who will be rejected.

**Mistake: binding by string path when a controller class would do.** Renaming the controller's prefix silently unbinds the middleware.

**Best practice: keep order-sensitive middleware in one `apply()` call.** Ordering across modules depends on resolution order, which is easy to change accidentally.

**Best practice: exclude health and metrics endpoints.** Probes run constantly and should not be logged, rate-limited, or authenticated.

**Best practice: prefer a guard whenever the decision depends on the route.** Middleware is for handler-agnostic, platform-level work.

---

## 11.6 Real-World Example: Request Context and Raw-Body Capture

Two middleware that genuinely cannot be anything else: one establishing per-request context before the pipeline runs, and one preserving the exact bytes a webhook signature is computed over.

```ts
// src/context/context.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { randomUUID } from 'node:crypto';
import { RequestContext } from './request-context';

@Injectable()
export class ContextMiddleware implements NestMiddleware {
  // A class, not a function, because it needs the injected RequestContext.
  constructor(private readonly context: RequestContext) {}

  use(req: Request, res: Response, next: NextFunction): void {
    const correlationId =
      (req.headers['x-correlation-id'] as string) ?? randomUUID();

    // Echo it back so a client can correlate its own logs with ours.
    res.setHeader('x-correlation-id', correlationId);

    // next() is called INSIDE run(), so the AsyncLocalStorage store covers
    // every guard, pipe, handler, and service for this request. Calling
    // run() and then next() outside it would leave the store empty.
    this.context.run({ correlationId, startedAt: Date.now() }, () => next());
  }
}
```

```ts
// src/webhooks/raw-body.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import * as express from 'express';

/**
 * Stripe signs the exact bytes it sent. Parsing to JSON and re-serialising
 * changes key order and whitespace, so the signature will never verify —
 * the raw buffer must be captured before any body parser runs.
 */
@Injectable()
export class RawBodyMiddleware implements NestMiddleware {
  private readonly parser = express.raw({ type: 'application/json', limit: '1mb' });

  use(req: Request, res: Response, next: NextFunction): void {
    this.parser(req, res, (err?: unknown) => {
      if (err) return next(err);
      // Keep the buffer for signature verification, and hand the pipeline
      // parsed JSON so DTO validation still works downstream.
      (req as Request & { rawBody?: Buffer }).rawBody = req.body as Buffer;
      try {
        req.body = JSON.parse((req.body as Buffer).toString('utf8'));
      } catch {
        req.body = {};                       // let ValidationPipe produce the 400
      }
      next();
    });
  }
}
```

```ts
// src/app.module.ts
@Module({
  imports: [ContextModule, WebhooksModule],
  controllers: [TasksController],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer): void {
    // Order matters and both are declared in one call, so it cannot drift
    // with module resolution order.
    consumer
      .apply(ContextMiddleware)
      .exclude(
        { path: 'health', method: RequestMethod.ALL },
        { path: 'metrics', method: RequestMethod.ALL },
      )
      .forRoutes('*splat');                  // Express 5 named wildcard

    // Bound to the controller class, so renaming its prefix cannot unbind it.
    consumer.apply(RawBodyMiddleware).forRoutes(StripeWebhookController);
  }
}
```

```ts
// src/webhooks/stripe-webhook.controller.ts
@Controller('webhooks/stripe')
export class StripeWebhookController {
  constructor(private readonly stripe: StripeService) {}

  @Post()
  @HttpCode(200)                              // Stripe treats non-2xx as a retry
  handle(
    @Req() req: Request & { rawBody?: Buffer },
    @Headers('stripe-signature') signature: string,
  ) {
    if (!req.rawBody) throw new BadRequestException('Raw body unavailable');
    const event = this.stripe.constructEvent(req.rawBody, signature);
    return this.stripe.process(event);
  }
}
```

The decision worth defending is calling `next()` *inside* `context.run()` rather than after it. Written the other way round the code looks equivalent and is silently broken: `AsyncLocalStorage.run` only keeps the store alive for the synchronous execution of its callback and anything asynchronously chained from within it, so invoking `run` and then calling `next()` outside would establish a store that has already exited by the time any handler executes. Every downstream `getStore()` would return `undefined`, and because the correlation-id fallback returns a placeholder rather than throwing, the symptom would be logs that all read `no-context` rather than an error anyone would notice. This is a general property of `AsyncLocalStorage` worth internalising: the store covers the callback's async subtree, so whatever continues the request must be invoked from inside it.

---

## 11.7 Questions

**1.** MCQ — Middleware runs: a) after guards b) before guards c) after pipes d) after the handler
<details><summary>Answer</summary>**b)** It is the first stage, before guards.</details>

**2.** MCQ — Functional middleware supports: a) dependency injection b) lifecycle hooks c) neither d) both
<details><summary>Answer</summary>**c)** Neither — it is not constructed by the container.</details>

**3.** MCQ — Forgetting `next()` causes: a) a 500 b) a 404 c) the request to hang d) the next middleware to run anyway
<details><summary>Answer</summary>**c)** It hangs until the client times out, with nothing logged.</details>

**4.** MCQ — Middleware is bound using: a) `@UseMiddleware()` b) `configure()` c) `@Injectable()` d) `APP_MIDDLEWARE`
<details><summary>Answer</summary>**b)** `configure(consumer)` in a module implementing `NestModule`. There is no middleware decorator.</details>

**5.** MCQ — On Nest 11, the correct catch-all route is: a) `'*'` b) `'**'` c) `'*splat'` d) `'/*'`
<details><summary>Answer</summary>**c)** Express 5 requires named wildcards.</details>

**6.** MCQ — Which can middleware NOT access? a) headers b) the raw response c) route metadata d) the request body
<details><summary>Answer</summary>**c)** It runs before route resolution, so there is no `ExecutionContext`.</details>

**7.** Short — Why can't middleware read `@Public()`?
<details><summary>Answer</summary>It executes before Nest resolves which handler will serve the request, so there is no handler whose metadata could be read.</details>

**8.** Short — When must middleware be a class rather than a function?
<details><summary>Answer</summary>When it needs an injected dependency or a lifecycle hook — functional middleware is not constructed by the container.</details>

**9.** Short — Why capture a raw body in middleware rather than a pipe?
<details><summary>Answer</summary>Signature verification is computed over the exact bytes sent, and by the time a pipe runs the body has been parsed and re-serialised, changing key order and whitespace.</details>

**10.** Short — Why bind middleware with `forRoutes(SomeController)` rather than a path string?
<details><summary>Answer</summary>The binding follows the controller if its route prefix changes; a string path silently stops matching.</details>

**11.** Short — Why exclude health and metrics endpoints from middleware?
<details><summary>Answer</summary>Probes run constantly, so logging, rate-limiting, or authenticating them adds noise and can take a healthy instance out of rotation.</details>

**12.** Output — What does the client receive?
```ts
use(req: Request, res: Response, next: NextFunction) {
  console.log('middleware ran');
}
```
<details><summary>Answer</summary>Nothing — the log prints and the request hangs until the client times out, because `next()` was never called and no response was sent.</details>

**13.** Output — `consumer.apply(A, B, C).forRoutes('*splat')`. In what order do they run?
<details><summary>Answer</summary>A, then B, then C — the order listed in `apply()`.</details>

**14.** Output — `app.use(helmet())` in `main.ts` and `consumer.apply(LoggerMiddleware)` in a module. Which runs first?
<details><summary>Answer</summary>`helmet()`. Global middleware registered with `app.use` runs before module-bound middleware.</details>

**15.** Output — Middleware sends a 401 response and then calls `next()`. What happens?
<details><summary>Answer</summary>The chain continues against an already-finished response, and Express warns that headers cannot be set after they are sent. Return after responding.</details>

**16.** Debug — A webhook signature never verifies, though the secret is correct. Cause?
<details><summary>Answer</summary>The body was parsed before the signature was computed. Capture the raw buffer with `express.raw()` in middleware bound to that route, and verify against those bytes.</details>

**17.** Debug — After upgrading to Nest 11, startup fails in a module's `configure()`. Likely cause?
<details><summary>Answer</summary>`forRoutes('*')` — Express 5 rejects unnamed wildcards. Change it to `'*splat'`.</details>

**18.** Debug — Functional middleware needs `ConfigService` and every attempt to inject it yields `undefined`. Fix?
<details><summary>Answer</summary>Convert it to a class implementing `NestMiddleware` with `@Injectable()`, and register it so the container constructs it.</details>

**19.** Debug — Correlation ids are `undefined` in every service, though the middleware sets them. What is the likely mistake?
<details><summary>Answer</summary>`next()` is being called outside `AsyncLocalStorage.run()`, so the store has exited before any handler executes. Invoke `next()` inside the `run` callback.</details>

**20.** Debug — Middleware order changed after an unrelated module import was added. Why?
<details><summary>Answer</summary>Ordering across modules follows module resolution order. Put order-sensitive middleware in a single `apply()` call so it cannot drift.</details>

**21.** Code — Write class middleware that rejects requests without an API key, without hanging on the success path.
<details><summary>Answer</summary>

```ts
@Injectable()
export class ApiKeyMiddleware implements NestMiddleware {
  constructor(private readonly config: ConfigService) {}

  use(req: Request, res: Response, next: NextFunction): void {
    const key = req.headers['x-api-key'];
    if (key !== this.config.getOrThrow('API_KEY')) {
      res.status(401).json({ message: 'Invalid API key' });
      return;                       // respond OR next() — never neither, never both
    }
    next();
  }
}
```
In practice this belongs in a guard unless it must run before all route resolution.
</details>

**22.** Code — Apply logging to every route except `health`, and a body-size limit only to `POST /uploads`.
<details><summary>Answer</summary>

```ts
configure(consumer: MiddlewareConsumer): void {
  consumer
    .apply(LoggerMiddleware)
    .exclude({ path: 'health', method: RequestMethod.ALL })
    .forRoutes('*splat');

  consumer
    .apply(express.json({ limit: '10mb' }))
    .forRoutes({ path: 'uploads', method: RequestMethod.POST });
}
```
</details>

**23.** Code — Write middleware that measures request duration and sets it as a response header.
<details><summary>Answer</summary>

```ts
export function timing(req: Request, res: Response, next: NextFunction): void {
  const start = process.hrtime.bigint();
  // The header must be set before headers are flushed, so hook the event
  // rather than writing after next() returns.
  res.on('header', () => {
    const ms = Number(process.hrtime.bigint() - start) / 1e6;
    res.setHeader('x-response-time', `${ms.toFixed(1)}ms`);
  });
  next();
}
```
An interceptor is the better tool if you only need to *log* the duration, since it sees the response value too.
</details>

**24.** Scenario — A teammate proposes moving JWT verification from a global guard into middleware, arguing it will be faster because it rejects earlier. Evaluate it.
<details><summary>Answer</summary>The performance claim is real and trivial — the saving is a few microseconds of route resolution, which is noise next to the signature verification itself. The cost is substantial: middleware cannot read `@Public()`, so every unauthenticated route must be listed in a path allowlist maintained separately from the routes, and the failure mode when someone forgets is an endpoint that is either unreachable or unintentionally open. It also cannot read role metadata, so authorization stays in a guard anyway and the auth logic ends up split across two layers. Keep it in a guard. The general principle: optimise the pipeline stage only when profiling says the stage is the problem, and never at the cost of moving a security decision away from the thing it protects.</details>

**25.** Scenario — Your API must support both JSON and a legacy XML content type on the same endpoints. Where does the conversion belong?
<details><summary>Answer</summary>Middleware, and only reluctantly. Content negotiation on the way in is genuinely platform-level — it operates on the raw body before any handler is chosen, which is exactly what middleware is for, so an XML-to-JSON translation middleware bound to the affected routes keeps every controller and DTO unaware of the legacy format. The response direction is different: converting outbound JSON to XML needs the response value, which middleware cannot see cleanly, so that half belongs in an interceptor keyed on the `Accept` header. Splitting a single concern across two components is unpleasant, and the alternative — a separate set of legacy controllers — duplicates every route. Prefer the split, keep both pieces in one module so the coupling is visible, and put a hard deprecation date on the XML path, since the real cost is that every future endpoint must be tested twice.</details>

---

## Chapter Summary

- Middleware is platform middleware running before route resolution, so it has no `ExecutionContext` and cannot read any route metadata.
- Forgetting `next()` hangs the request silently — respond or continue, never neither and never both.
- Functional middleware cannot inject anything; needing a dependency is what forces the class form.
- It is bound only through `configure()`, and binding to a controller class survives a prefix rename that a string path would not.
- Its genuine niches are raw-body capture, request context, third-party Express middleware, and anything wrapping the response stream.
- Authentication is the classic misuse: without route metadata, `@Public()` becomes a hand-maintained allowlist that drifts from the code.

---

## Practice Exercises

1. Write middleware that omits `next()` and observe exactly how the failure presents to a client and in the logs.
2. Convert a functional middleware into a class because it needs `ConfigService`, and note everything that changes.
3. Bind middleware by string path, rename the controller's prefix, and confirm it silently stops running.
4. Implement raw-body capture and verify an HMAC signature over the exact bytes.
5. Move an authentication middleware to a global guard and make `@Public()` work, then compare the two implementations.

---

## Mini Project: A Request Auditing Middleware

Build auditing that records every mutating request for compliance.

**Requirements:**

- Captures method, path, correlation id, actor, request size, status, and duration.
- Records the raw body for `POST`/`PUT`/`PATCH`, redacting configured sensitive fields before storage.
- Skips health and metrics endpoints entirely.
- Never blocks or fails a request — an auditing outage must not become an API outage.
- A test proving a hung audit sink does not delay the response.

---

**Next:** [Chapter 12 — Guards →](12-guards.md)
