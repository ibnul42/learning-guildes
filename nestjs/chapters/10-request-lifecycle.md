# Chapter 10 — The Request Lifecycle

## Learning Objectives

- Recite the exact order in which middleware, guards, interceptors, pipes, handlers, and filters run
- Explain why global components run before controller-scoped ones, and where that reverses
- Predict what still executes after a guard denies a request or a pipe throws
- Choose the right component for a requirement instead of reaching for the familiar one
- Debug an ordering problem by reasoning about the pipeline rather than by trial and error

---

## 10.1 The Canonical Order

This is the chapter every other chapter in Part III points at. Learn this list and most NestJS ordering questions answer themselves.

```
1.  Incoming request
2.  Middleware              global (app.use) → module-bound (configure), in binding order
3.  Guards                  global → controller → route
4.  Interceptors (pre)      global → controller → route
5.  Pipes                   global → controller → route → route parameter
6.  Route handler           your controller method
7.  Service layer           whatever the handler calls
8.  Interceptors (post)     route → controller → global      ← reversed
9.  Exception filters       route → controller → global      ← most specific first
10. Response sent
```

Two properties of this list do most of the work.

**Steps 3–5 run outermost-first.** A global guard runs before a controller guard, which runs before a route guard. The rationale is that broad checks should reject early — there is no reason to run a route-specific permission check on a request that failed authentication.

**Step 8 reverses.** Interceptors wrap the handler, so they unwind in the opposite order to the one they entered. A global interceptor is the outermost layer: it sees the request first and the response last.

```ts
// Global LoggingInterceptor + route TransformInterceptor
// →  Logging (pre)
//      →  Transform (pre)
//          →  handler
//      ←  Transform (post)
//    ←  Logging (post)
```

> 💼 **Interview** — "Walk me through what happens between the request arriving and your controller method running" is the most common NestJS interview question at any level. The strong answer names the six stages in order *and* explains why guards precede pipes: there is no point validating a body for a caller who is not allowed to be here.

---

## 10.2 Why Guards Run Before Pipes

The ordering is not arbitrary, and understanding the reasoning makes it memorable.

**Middleware first** because it is platform-level. It runs on the raw Express or Fastify request before Nest has decided which handler will serve it — so it has no idea what controller or method is involved (→ [Ch 11](11-middleware.md)).

**Guards next** because authorization is the cheapest rejection. A guard has an `ExecutionContext`, so it *does* know the target handler and can read its metadata. Rejecting here means no body parsing, no validation, and no handler work (→ [Ch 12](12-guards.md)).

**Interceptors before pipes** because an interceptor's job is to wrap the whole operation, including validation. A timeout interceptor must cover the time spent validating; a logging interceptor should record requests that fail validation.

**Pipes last** because validation is the most expensive per-request step and the most specific. By the time a pipe runs, the caller is authenticated, authorized, and known to be hitting a real handler.

**Filters are not in the chain at all.** They are the catch handler wrapped around everything from step 3 onward. Nothing calls a filter; a filter runs because something threw (→ [Ch 15](15-exception-filters.md)).

| Component | Knows the handler? | Can stop the request? | Sees the response? |
|---|---|---|---|
| Middleware | No | Yes (`res.end()`) | Only via raw `res` |
| Guard | Yes | Yes (return `false`) | No |
| Interceptor | Yes | Yes (return early) | **Yes** |
| Pipe | Yes | Yes (throw) | No |
| Filter | Yes | — | Yes (it writes it) |

That table is the decision tool. **If a component needs to see or modify the response, only an interceptor or a filter will do.** If it needs to know the target handler, middleware is out.

---

## 10.3 What Happens When Something Stops the Request

Short-circuiting is where most confusion lives.

**A guard returns `false`.** Nest throws `ForbiddenException`. Pipes and the handler never run. Interceptors that already ran their pre-half do **not** run their post-half, because there is no response value flowing back — the exception path takes over and filters handle it.

```ts
@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(ctx: ExecutionContext): boolean {
    return false;             // → 403 Forbidden, nothing downstream executes
  }
}
```

Throwing your own exception from a guard gives a better message than returning `false`:

```ts
if (!user) throw new UnauthorizedException('Missing bearer token');   // 401, not 403
```

**A pipe throws.** The handler never runs. `ValidationPipe` throws `BadRequestException` with per-field messages.

**An interceptor returns early.** This is the one that surprises people — an interceptor can skip the handler entirely by returning an observable instead of calling `next.handle()`:

```ts
@Injectable()
export class CacheInterceptor implements NestInterceptor {
  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    const cached = this.cache.get(this.keyFor(ctx));
    if (cached) return of(cached);      // handler never runs — and neither do pipes
    return next.handle();
  }
}
```

Because interceptors run *before* pipes, a cache hit skips validation too. That is usually what you want for a GET, and it is a genuine hazard if the cache key is built from unvalidated input.

**The handler throws.** Filters take over. Post-interceptors are bypassed unless they explicitly handle errors with `catchError`.

> ⚠️ **Gotcha** — Post-interceptor logic does not run on the error path. A logging interceptor written as `tap(() => log('done'))` silently logs nothing for failed requests, which is exactly when you wanted the log. Use `tap({ next, error })` or add `catchError`.

---

## 10.4 Global, Controller, and Route Scope

Every pipeline component can be bound at three levels, and the binding method differs:

```ts
// Global — in main.ts (cannot inject dependencies)
app.useGlobalGuards(new AuthGuard());

// Global — as a provider (CAN inject; preferred, → Ch 8)
{ provide: APP_GUARD, useClass: AuthGuard }

// Controller-scoped
@UseGuards(RolesGuard)
@Controller('tasks')
export class TasksController {}

// Route-scoped
@UseGuards(OwnerGuard)
@Delete(':id')
remove() {}
```

All three apply cumulatively — a route with a global, a controller, and a route guard runs all three, in that order. Binding at a narrower scope does not replace the broader one.

**Middleware is the exception.** It cannot be bound with a decorator; it is configured in a module:

```ts
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer): void {
    consumer
      .apply(LoggerMiddleware)
      .exclude({ path: 'health', method: RequestMethod.GET })
      .forRoutes('*splat');                     // Express 5 named wildcard
  }
}
```

Multiple `APP_GUARD` registrations accumulate rather than replace, and run in registration order (→ [Ch 8](08-custom-providers.md)).

---

## 10.5 Choosing the Right Component

Most "how do I do X in NestJS" questions are really "which pipeline component owns X".

| Requirement | Component | Why |
|---|---|---|
| Parse cookies, add security headers | Middleware | platform-level, handler-agnostic |
| Reject unauthenticated requests | Guard | knows the handler, rejects cheapest |
| Check the caller owns this record | Guard | authorization, needs route metadata |
| Validate and coerce the body | Pipe | operates on a single argument |
| Wrap every response in an envelope | Interceptor | only interceptors see the response |
| Add a request timeout | Interceptor | must wrap the whole operation |
| Cache GET responses | Interceptor | can skip the handler entirely |
| Strip `passwordHash` from output | Interceptor | `ClassSerializerInterceptor` |
| Map a domain error to a status code | Filter | runs on the throw path |
| Log every request with its duration | Interceptor | sees both ends |

Two common miscategorisations are worth calling out.

**Authentication in middleware.** It works, and it cannot read route metadata — so `@Public()` on one route is invisible to it, and you end up with a path allowlist that drifts from the decorators. Use a global guard.

**Response shaping in middleware.** Middleware only has the raw `res`, so wrapping a response means monkey-patching `res.json`. An interceptor receives the value directly.

---

## 10.6 Common Mistakes and Best Practices

**Mistake: expecting decorator order to set pipeline order.** Nest fixes guard → interceptor → pipe; stacking decorators differently changes nothing except ordering within one kind.

**Mistake: assuming post-interceptor code runs on errors.** `tap(fn)` fires only on success; failed requests skip it silently.

**Mistake: authentication in middleware.** It cannot read `@Public()` or role metadata, so the allowlist drifts from the decorators.

**Mistake: caching before validating.** Interceptors precede pipes, so a cache key built from raw input can be poisoned by a request that would have failed validation.

**Mistake: `useGlobalGuards(new Guard())` for a guard with dependencies.** Registration in `main.ts` cannot inject; use `APP_GUARD`.

**Mistake: assuming a narrower binding replaces a broader one.** All scopes apply cumulatively.

**Mistake: returning `false` from a guard when you mean 401.** Bare `false` always produces 403; throw `UnauthorizedException` for a missing credential.

**Best practice: put cross-cutting concerns at the global level as providers.** They gain injection and appear in tests, which `main.ts` registration does not.

**Best practice: let filters own error-to-status mapping.** Services should throw domain errors; only the filter should know about HTTP codes.

**Best practice: keep guards fast and side-effect-free.** They run on every request to the routes they cover, ahead of everything else.

---

## 10.7 Real-World Example: One Request Through Every Stage

A single endpoint instrumented at every stage, so the ordering is observable rather than memorised. Running this and reading the log teaches the chapter better than the list does.

```ts
// src/observability/trace.interceptor.ts
@Injectable()
export class TraceInterceptor implements NestInterceptor {
  constructor(private readonly label: string) {}

  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    const started = Date.now();
    console.log(`  [interceptor:${this.label}] pre`);

    return next.handle().pipe(
      // `tap` with both handlers so the error path is instrumented too —
      // tap(fn) alone would log nothing for a failed request.
      tap({
        next: () => console.log(`  [interceptor:${this.label}] post (${Date.now() - started}ms)`),
        error: (e) => console.log(`  [interceptor:${this.label}] error: ${e.message}`),
      }),
    );
  }
}
```

```ts
// src/observability/trace.guard.ts
@Injectable()
export class TraceGuard implements CanActivate {
  constructor(private readonly label: string) {}

  canActivate(ctx: ExecutionContext): boolean {
    // A guard knows the target, which is exactly what middleware cannot see.
    const handler = ctx.getHandler().name;
    console.log(`  [guard:${this.label}] checking ${handler}`);
    return true;
  }
}
```

```ts
// src/observability/trace.pipe.ts
@Injectable()
export class TracePipe implements PipeTransform {
  transform(value: unknown, metadata: ArgumentMetadata): unknown {
    console.log(`  [pipe] ${metadata.type}:${metadata.data ?? '-'}`);
    return value;
  }
}
```

```ts
// src/observability/trace.filter.ts
@Catch()
export class TraceFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost): void {
    console.log(`  [filter] caught ${(exception as Error).constructor.name}`);

    const ctx = host.switchToHttp();
    const status =
      exception instanceof HttpException ? exception.getStatus() : 500;

    ctx.getResponse<Response>().status(status).json({
      statusCode: status,
      message: exception instanceof HttpException ? exception.message : 'Internal error',
    });
  }
}
```

```ts
// src/tasks/tasks.controller.ts
@Controller('tasks')
@UseGuards(new TraceGuard('controller'))
@UseInterceptors(new TraceInterceptor('controller'))
export class TasksController {
  @Get(':id')
  @UseGuards(new TraceGuard('route'))
  @UseInterceptors(new TraceInterceptor('route'))
  @UseFilters(TraceFilter)
  findOne(@Param('id', TracePipe, ParseIntPipe) id: number) {
    console.log('  [handler] running');
    if (id === 0) throw new NotFoundException('Task 0 does not exist');
    return { id };
  }
}
```

```ts
// src/app.module.ts
@Module({
  providers: [
    { provide: APP_GUARD,       useValue: new TraceGuard('global') },
    { provide: APP_INTERCEPTOR, useValue: new TraceInterceptor('global') },
  ],
})
export class AppModule {}
```

Requesting `GET /tasks/7` prints the pipeline in order:

```
  [guard:global] checking findOne
  [guard:controller] checking findOne
  [guard:route] checking findOne
  [interceptor:global] pre
  [interceptor:controller] pre
  [interceptor:route] pre
  [pipe] param:id
  [handler] running
  [interceptor:route] post (2ms)
  [interceptor:controller] post (2ms)
  [interceptor:global] post (3ms)
```

Requesting `GET /tasks/0` shows the error path — note that the post-halves report an error rather than being skipped, precisely because `tap` was given both handlers:

```
  [guard:global] checking findOne
  [guard:controller] checking findOne
  [guard:route] checking findOne
  [interceptor:global] pre
  [interceptor:controller] pre
  [interceptor:route] pre
  [pipe] param:id
  [handler] running
  [interceptor:route] error: Task 0 does not exist
  [interceptor:controller] error: Task 0 does not exist
  [interceptor:global] error: Task 0 does not exist
  [filter] caught NotFoundException
```

The decision worth defending is instrumenting `tap` with both `next` and `error` rather than the shorter `tap(() => …)`. The short form is what almost everyone writes, and it produces a monitoring system that reports only successes — latency percentiles computed from requests that did not fail, and a "requests completed" counter that silently ignores every 500. The failure mode is dangerous because the dashboard looks healthy during an outage. Any interceptor doing measurement or logging must handle the error path explicitly, and the fact that this requires remembering an option rather than being the default is the single most common instrumentation bug in Nest applications.

---

## 10.8 Questions

**1.** MCQ — Which runs first? a) pipe b) guard c) interceptor d) filter
<details><summary>Answer</summary>**b)** Guards, then interceptors, then pipes. Filters run only on a throw.</details>

**2.** MCQ — Post-interceptor logic runs in which order? a) global → route b) route → global c) same as pre d) undefined
<details><summary>Answer</summary>**b)** Interceptors wrap the handler, so they unwind in reverse.</details>

**3.** MCQ — Which component cannot know the target handler? a) guard b) interceptor c) middleware d) pipe
<details><summary>Answer</summary>**c)** Middleware runs before Nest resolves the route, so it has no `ExecutionContext`.</details>

**4.** MCQ — A guard returning `false` produces: a) 401 b) 403 c) 400 d) 500
<details><summary>Answer</summary>**b)** 403 Forbidden. Throw `UnauthorizedException` explicitly if you want 401.</details>

**5.** MCQ — Which components can see the response value? a) guards and pipes b) interceptors and filters c) middleware only d) all of them
<details><summary>Answer</summary>**b)** Only interceptors and filters.</details>

**6.** MCQ — Binding a guard at route scope when a global guard exists means: a) the route guard replaces it b) both run c) only the global runs d) an error
<details><summary>Answer</summary>**b)** Scopes are cumulative — global, then controller, then route.</details>

**7.** Short — Why do guards run before pipes?
<details><summary>Answer</summary>Authorization is the cheapest rejection. There is no reason to parse and validate a body for a caller who is not permitted to reach the handler at all.</details>

**8.** Short — Why can an interceptor implement caching but a pipe cannot?
<details><summary>Answer</summary>An interceptor wraps the handler and can return a value without calling `next.handle()`; a pipe only transforms one argument on the way in and cannot produce a response.</details>

**9.** Short — What is the risk of caching in an interceptor?
<details><summary>Answer</summary>Interceptors run before pipes, so the cache key is built from unvalidated input and a request that would have been rejected can still populate or read the cache.</details>

**10.** Short — Why is authentication in middleware a poor choice?
<details><summary>Answer</summary>Middleware cannot read route metadata, so `@Public()` and role decorators are invisible to it and you must maintain a path allowlist that drifts from the code.</details>

**11.** Short — Where does error-to-status mapping belong, and why?
<details><summary>Answer</summary>In an exception filter, so services throw domain errors and only one layer knows about HTTP status codes — keeping the business logic reusable outside HTTP.</details>

**12.** Output — With a global and a route interceptor both logging `pre` and `post`, what is the log order for a successful request?
<details><summary>Answer</summary>`global pre`, `route pre`, handler, `route post`, `global post`. Entry outermost-first, exit innermost-first.</details>

**13.** Output — A guard returns `false`. Which of these run: pipes, handler, post-interceptors, filters?
<details><summary>Answer</summary>Only filters. The guard's rejection throws `ForbiddenException`; pipes, the handler, and the interceptors' post-halves are all skipped.</details>

**14.** Output — An interceptor uses `tap(() => logger.log('done'))`. The handler throws. What is logged?
<details><summary>Answer</summary>Nothing. `tap` with a single function subscribes only to the next notification, so the error path produces no log at all.</details>

**15.** Output — An interceptor returns `of(cachedValue)` without calling `next.handle()`. Does `ValidationPipe` run?
<details><summary>Answer</summary>No. Interceptors run before pipes, so returning early skips validation and the handler entirely.</details>

**16.** Debug — A logging interceptor reports 100% success while users see 500s. Cause and fix?
<details><summary>Answer</summary>It uses `tap(fn)`, which only fires on success. Use `tap({ next, error })` or add `catchError` so failures are recorded.</details>

**17.** Debug — A `@Public()` decorator is ignored and every request is rejected, even public ones. Where is the auth check?
<details><summary>Answer</summary>Almost certainly in middleware, which has no access to handler metadata. Move it to a global guard that reads the decorator with `Reflector`.</details>

**18.** Debug — A route guard needs `ConfigService`, but it is `undefined`. The guard is bound with `@UseGuards(new AuthGuard())`. Why?
<details><summary>Answer</summary>Passing an instance bypasses the container, so nothing is injected. Pass the class — `@UseGuards(AuthGuard)` — and let Nest construct it.</details>

**19.** Debug — A validation error returns 500 instead of 400 after adding a global filter. What went wrong?
<details><summary>Answer</summary>The filter catches everything with `@Catch()` and maps unknown errors to 500 without re-checking for `HttpException`. Preserve the status when the exception is an `HttpException`.</details>

**20.** Debug — Two `APP_INTERCEPTOR` registrations behave differently after their order in the providers array was swapped. Expected?
<details><summary>Answer</summary>Yes. Multi-provider tokens run in registration order, and interceptors nest, so swapping them changes which one wraps the other.</details>

**21.** Code — Write an interceptor that times every request and records failures too.
<details><summary>Answer</summary>

```ts
@Injectable()
export class TimingInterceptor implements NestInterceptor {
  private readonly logger = new Logger(TimingInterceptor.name);

  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    const start = Date.now();
    const { method, url } = ctx.switchToHttp().getRequest();

    return next.handle().pipe(
      tap({
        next: () => this.logger.log(`${method} ${url} ok ${Date.now() - start}ms`),
        error: (e) => this.logger.warn(`${method} ${url} failed ${Date.now() - start}ms: ${e.message}`),
      }),
    );
  }
}
```
</details>

**22.** Code — Register a global guard that can inject `Reflector` and a config service.
<details><summary>Answer</summary>

```ts
@Module({
  providers: [{ provide: APP_GUARD, useClass: JwtAuthGuard }],
})
export class AppModule {}

@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(
    private readonly reflector: Reflector,
    private readonly config: ConfigService,
  ) {}
  canActivate(ctx: ExecutionContext): boolean { /* … */ return true; }
}
```
`useClass` lets the container construct it; `useGlobalGuards(new JwtAuthGuard())` could not inject either dependency.
</details>

**23.** Code — Add a 5-second timeout to every request, converting the timeout into a 504.
<details><summary>Answer</summary>

```ts
@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(_ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    return next.handle().pipe(
      timeout(5_000),
      catchError((err) =>
        err instanceof TimeoutError
          ? throwError(() => new GatewayTimeoutException())
          : throwError(() => err),
      ),
    );
  }
}
```
It must be an interceptor, not middleware, so the timeout covers validation and the handler together.
</details>

**24.** Scenario — Your team wants to add tenant isolation so no query can ever read another tenant's rows. Where in the pipeline does it belong?
<details><summary>Answer</summary>Split it. A guard establishes and validates the tenant — it knows the handler, can read a `@SkipTenant()` decorator for admin routes, and rejects before any work happens. Enforcement, however, must not live in the pipeline at all: a guard that only checks a header is bypassed by any code path not going through HTTP, including queue consumers and cron jobs. The durable protection is at the data layer — a repository that always applies a tenant predicate, or row-level security in the database. Treat the guard as the fast rejection and the data layer as the guarantee, because a security control that lives only in the request pipeline protects only requests.</details>

**25.** Scenario — An interceptor caching GET responses has caused a customer to see another customer's data once. What went wrong and how do you prevent a recurrence?
<details><summary>Answer</summary>The cache key omitted identity — almost certainly built from the URL alone, so two users hitting the same path shared an entry. The immediate fix is to include the authenticated principal, and every other varying input, in the key. The structural lesson is that interceptors run before pipes and after guards, so the key is assembled from raw, unvalidated input while the user *is* known — meaning identity is available and easy to forget. Prevent recurrence by making the key derivation a single reviewed function rather than an inline template string, defaulting to not caching authenticated responses at all, and requiring an explicit opt-in decorator naming what the response varies by. Cache correctness bugs are silent and, unlike most bugs, leak data rather than breaking a feature.</details>

---

## Chapter Summary

- The order is middleware → guards → interceptors (pre) → pipes → handler → interceptors (post) → filters, with global-to-route on the way in and route-to-global on the way out.
- Guards precede pipes because rejecting an unauthorized caller should cost less than validating their input.
- Only interceptors and filters can see the response; only guards, interceptors, and pipes know the target handler.
- A guard returning `false` skips pipes, the handler, and every post-interceptor — bare `false` always yields 403, so throw explicitly for 401.
- Interceptors run before pipes, so early-returning for a cache hit skips validation entirely, and cache keys are built from unvalidated input.
- Post-interceptor code does not run on errors unless you handle them, which makes `tap(fn)` the most common source of dashboards that report only successes.

---

## Practice Exercises

1. Build the tracing setup from §10.7 and capture the log for a success, a guard rejection, a validation failure, and a handler throw.
2. Write an interceptor that skips the handler on a cache hit, then prove that `ValidationPipe` did not run.
3. Bind the same guard at global, controller, and route scope and confirm all three execute.
4. Convert an authentication middleware to a global guard and make `@Public()` work.
5. Instrument an interceptor with `tap(fn)`, generate failing requests, and observe the missing logs before fixing it.

---

## Mini Project: A Pipeline Visualiser

Build a development-only module that renders the pipeline for any route.

**Requirements:**

- `GET /_pipeline?route=/tasks/:id` lists every middleware, guard, interceptor, pipe, and filter that would run, in execution order.
- Each entry shows its binding scope (global, controller, route) and the class name.
- Reads the actual registered metadata rather than a hand-maintained list.
- Refuses to register when `NODE_ENV === 'production'`.
- A test asserting the reported order matches the documented pipeline.

---

**Next:** [Chapter 11 — Middleware →](11-middleware.md)
