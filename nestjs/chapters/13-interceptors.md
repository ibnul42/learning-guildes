# Chapter 13 — Interceptors

## Learning Objectives

- Implement `NestInterceptor` and explain what `CallHandler` returns
- Use the handful of RxJS operators that cover almost every interceptor
- Transform, wrap, and serialize responses without touching controllers
- Add timeouts, caching, and error mapping at the right layer
- Recognise the two ways an interceptor silently does nothing

---

## 13.1 An Interceptor Wraps the Handler

An interceptor is the only component that runs on both sides of the route handler. That is what it is for.

```ts
// src/common/logging.interceptor.ts
import { CallHandler, ExecutionContext, Injectable, NestInterceptor } from '@nestjs/common';
import { Observable, tap } from 'rxjs';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const start = Date.now();               // ── before the handler

    return next.handle().pipe(              // ── next.handle() invokes it
      tap(() => console.log(`took ${Date.now() - start}ms`)),   // ── after
    );
  }
}
```

`next.handle()` returns an `Observable` that emits the handler's return value. Everything before that call runs on the way in; everything piped onto it runs on the way out.

Interceptors nest. With a global and a route interceptor, the global one is the outer layer — it enters first and exits last (→ [Ch 10](10-request-lifecycle.md)).

Two things follow, and both are exploited constantly:

**You can skip the handler entirely** by returning your own observable instead of calling `next.handle()`. That is how caching works.

**You can modify the response**, because the value flows through your operators before it is serialized. No other component can do this except a filter, and a filter only runs on errors.

> ⚠️ **Gotcha** — An interceptor cannot transform the response of a handler that injected bare `@Res()`. That handler writes the response itself, so nothing flows back through the observable and your `map` never runs. The symptom is one endpoint mysteriously exempt from a global envelope (→ [Ch 5](05-controllers-routing.md)).

---

## 13.2 The RxJS You Actually Need

Interceptors are the one place Nest requires RxJS, and six operators cover nearly everything.

| Operator | Does | Typical use |
|---|---|---|
| `map` | transforms the emitted value | response envelopes |
| `tap` | side effect, value untouched | logging, metrics |
| `catchError` | intercepts errors | mapping domain errors |
| `timeout` | errors if too slow | request deadlines |
| `finalize` | runs on success **and** error | cleanup, releasing resources |
| `of` | creates an observable from a value | returning a cached response |

The distinction that causes the most bugs is `tap` versus `finalize`:

```ts
// ❌ Fires only on success — a failed request logs nothing
return next.handle().pipe(tap(() => this.metrics.record()));

// ✅ Both paths, with different handling
return next.handle().pipe(
  tap({
    next: (value) => this.metrics.success(),
    error: (err) => this.metrics.failure(err),
  }),
);

// ✅ Or finalize, which runs regardless of outcome
return next.handle().pipe(finalize(() => this.release()));
```

`tap(fn)` with a single function subscribes only to the next notification. Any interceptor doing measurement, logging, or metrics that uses it will report exclusively on successful requests — producing dashboards that look healthy during an outage.

---

## 13.3 Response Envelopes

Wrapping every response in a consistent shape is the canonical `map` use:

```ts
// src/common/transform.interceptor.ts
export interface ApiResponse<T> {
  data: T;
  meta: { timestamp: string; correlationId: string };
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, ApiResponse<T>> {
  constructor(private readonly context: RequestContext) {}

  intercept(_ctx: ExecutionContext, next: CallHandler<T>): Observable<ApiResponse<T>> {
    return next.handle().pipe(
      map((data) => ({
        data,
        meta: {
          timestamp: new Date().toISOString(),
          correlationId: this.context.correlationId,
        },
      })),
    );
  }
}
```

Every controller keeps returning plain objects; the envelope is applied in one place.

The catch is that it applies to *everything*, including responses that should not be wrapped — a file download, a health check a load balancer parses, a redirect. Exempt them with metadata rather than by removing the global registration:

```ts
export const NO_ENVELOPE = 'noEnvelope';
export const RawResponse = () => SetMetadata(NO_ENVELOPE, true);

// in intercept()
const raw = this.reflector.getAllAndOverride<boolean>(NO_ENVELOPE, [
  ctx.getHandler(),
  ctx.getClass(),
]);
if (raw) return next.handle();
```

---

## 13.4 Serialization

`ClassSerializerInterceptor` applies `class-transformer` decorators to whatever the handler returns. It is how sensitive fields get stripped declaratively:

```ts
// src/users/dto/user.dto.ts
export class UserDto {
  id: string;
  email: string;

  @Exclude()                    // never serialized
  passwordHash: string;

  @Expose()
  get displayName(): string {   // computed, included
    return this.email.split('@')[0];
  }
}
```

```ts
@UseInterceptors(ClassSerializerInterceptor)
@Get(':id')
findOne(@Param('id') id: string): Promise<UserDto> {
  return this.users.findOne(id);
}
```

Two conditions must both hold, and missing either produces silent failure:

**The returned value must be an instance of the decorated class.** A plain object from an ORM has no decorators attached, so nothing is excluded:

```ts
// ❌ A plain object — @Exclude() has nothing to act on, passwordHash leaks
return await this.repo.findOne(id);

// ✅ A real instance
return plainToInstance(UserDto, await this.repo.findOne(id));
```

**The interceptor must be applied.** Without it the decorators are inert metadata.

Enabling `excludeExtraneousValues` makes the DTO an allowlist rather than a denylist, which is the safer default — a new database column is not exposed until someone adds `@Expose()`:

```ts
providers: [
  {
    provide: APP_INTERCEPTOR,
    useFactory: (reflector: Reflector) =>
      new ClassSerializerInterceptor(reflector, {
        excludeExtraneousValues: true,     // only @Expose()d fields survive
        excludePrefixes: ['_'],
      }),
    inject: [Reflector],
  },
]
```

> 💼 **Interview** — "How do you stop a password hash reaching the client?" A weak answer deletes the field in the controller. A strong one uses `@Exclude()` with `ClassSerializerInterceptor` and explains that `excludeExtraneousValues` inverts the default so future columns are private until explicitly exposed.

---

## 13.5 Timeouts, Caching, and Error Mapping

**Timeouts** must be interceptors, because the deadline should cover validation and the handler together:

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

The `catchError` is required — a bare `timeout()` emits `TimeoutError`, which is not an `HttpException` and becomes a 500.

**Caching** exploits the ability to skip the handler:

```ts
@Injectable()
export class HttpCacheInterceptor implements NestInterceptor {
  constructor(private readonly cache: CacheService, private readonly reflector: Reflector) {}

  async intercept(ctx: ExecutionContext, next: CallHandler): Promise<Observable<unknown>> {
    const request = ctx.switchToHttp().getRequest();
    if (request.method !== 'GET') return next.handle();

    // Identity MUST be in the key, or one user's response is served to another.
    const key = `${request.user?.id ?? 'anon'}:${request.originalUrl}`;

    const hit = await this.cache.get(key);
    if (hit) return of(hit);                    // handler and pipes both skipped

    return next.handle().pipe(tap((value) => this.cache.set(key, value, 60)));
  }
}
```

Because interceptors run before pipes, the key is built from unvalidated input. Omitting the principal from the key is the single most damaging cache bug there is — it leaks data across users.

**Error mapping** belongs in a filter, not an interceptor. `catchError` in an interceptor is for errors you want to *transform in flight* — converting a `TimeoutError`, retrying a transient failure — not for producing HTTP responses (→ [Ch 15](15-exception-filters.md)).

---

## 13.6 Common Mistakes and Best Practices

**Mistake: `tap(fn)` for metrics or logging.** It only fires on success, so failures vanish from your instrumentation.

**Mistake: forgetting to return `next.handle()`.** Returning nothing, or not returning the piped observable, hangs the request.

**Mistake: expecting an interceptor to transform a `@Res()` handler's response.** That handler writes the response itself; nothing flows back.

**Mistake: `ClassSerializerInterceptor` on plain objects.** The decorators live on a class, so a raw ORM row is serialized untouched and excluded fields leak.

**Mistake: cache keys without the principal.** Two users hitting the same URL share an entry, which leaks data.

**Mistake: bare `timeout()` with no `catchError`.** `TimeoutError` is not an `HttpException` and surfaces as a 500.

**Mistake: building HTTP responses in `catchError`.** Error-to-status mapping belongs in a filter, where it is applied consistently.

**Best practice: use `tap({ next, error })` or `finalize` for anything observational.** Instrumentation that ignores failures is worse than none.

**Best practice: set `excludeExtraneousValues: true`.** It makes DTOs an allowlist, so new columns are private by default.

**Best practice: exempt responses from a global envelope with metadata.** A decorator keeps the exception next to the route rather than in the interceptor.

---

## 13.7 Real-World Example: A Response Envelope With Metrics and Timeouts

Three interceptors registered globally, each doing one job, ordered so their nesting is correct.

```ts
// src/common/interceptors/metrics.interceptor.ts
@Injectable()
export class MetricsInterceptor implements NestInterceptor {
  constructor(private readonly metrics: MetricsService) {}

  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    if (ctx.getType() !== 'http') return next.handle();

    const request = ctx.switchToHttp().getRequest();
    // The route pattern, not the URL — otherwise /tasks/1 and /tasks/2 become
    // separate metric series and cardinality explodes.
    const route = `${request.method} ${request.route?.path ?? 'unknown'}`;
    const start = process.hrtime.bigint();

    const record = (outcome: 'success' | 'error', status: number) => {
      const ms = Number(process.hrtime.bigint() - start) / 1e6;
      this.metrics.histogram('http.duration', ms, { route, outcome, status });
    };

    return next.handle().pipe(
      tap({
        next: () => record('success', ctx.switchToHttp().getResponse().statusCode),
        // Without this branch every failed request would be invisible.
        error: (err) => record('error', err instanceof HttpException ? err.getStatus() : 500),
      }),
    );
  }
}
```

```ts
// src/common/interceptors/timeout.interceptor.ts
@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  constructor(private readonly reflector: Reflector) {}

  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    // Per-route override, because a report endpoint legitimately takes longer
    // than the default and should not need a separate interceptor.
    const ms =
      this.reflector.getAllAndOverride<number>(TIMEOUT_KEY, [
        ctx.getHandler(),
        ctx.getClass(),
      ]) ?? 5_000;

    return next.handle().pipe(
      timeout(ms),
      catchError((err) =>
        err instanceof TimeoutError
          ? throwError(() => new GatewayTimeoutException(`Exceeded ${ms}ms`))
          : throwError(() => err),
      ),
    );
  }
}
```

```ts
// src/common/interceptors/transform.interceptor.ts
export const NO_ENVELOPE = 'noEnvelope';
export const RawResponse = () => SetMetadata(NO_ENVELOPE, true);

@Injectable()
export class TransformInterceptor implements NestInterceptor {
  constructor(
    private readonly reflector: Reflector,
    private readonly context: RequestContext,
  ) {}

  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    const raw = this.reflector.getAllAndOverride<boolean>(NO_ENVELOPE, [
      ctx.getHandler(),
      ctx.getClass(),
    ]);
    if (raw || ctx.getType() !== 'http') return next.handle();

    return next.handle().pipe(
      map((data) => ({
        data,
        meta: {
          correlationId: this.context.correlationId,
          timestamp: new Date().toISOString(),
        },
      })),
    );
  }
}
```

```ts
// src/app.module.ts
@Module({
  providers: [
    // Order defines nesting. Metrics is outermost so it measures everything,
    // including time spent inside the timeout and transform layers.
    { provide: APP_INTERCEPTOR, useClass: MetricsInterceptor },
    { provide: APP_INTERCEPTOR, useClass: TimeoutInterceptor },
    { provide: APP_INTERCEPTOR, useClass: TransformInterceptor },
  ],
})
export class AppModule {}
```

```ts
// src/reports/reports.controller.ts
@Controller('reports')
export class ReportsController {
  @Timeout(30_000)                    // this one legitimately takes longer
  @Get('annual')
  annual() {
    return this.reports.buildAnnual();
  }

  @RawResponse()                      // a probe parses this; no envelope
  @Get('health')
  health() {
    return { status: 'ok' };
  }
}
```

The decision worth defending is registering `MetricsInterceptor` first so it is the outermost layer. The intuitive ordering is the reverse — measure the handler, not the framework overhead — and it produces numbers that are wrong in the way that matters. As the outermost layer, metrics observe what the *client* experiences: the timeout interceptor's deadline, the transform layer's serialization cost, and crucially the `GatewayTimeoutException` itself, which an inner metrics interceptor would never see because the timeout above it converts the error before it propagates down. Placed innermost, a slow endpoint that times out records nothing at all, so the very requests you most need to see are the ones missing from the histogram. The rule generalises: instrumentation belongs outside the components whose behaviour it is meant to capture, and "measure only my code" is the wrong instinct when the question is why users are waiting.

---

## 13.8 Questions

**1.** MCQ — `next.handle()` returns: a) the value b) a promise c) an Observable d) void
<details><summary>Answer</summary>**c)** An Observable that emits the handler's return value.</details>

**2.** MCQ — Which operator runs on both success and error? a) `map` b) `tap(fn)` c) `finalize` d) `of`
<details><summary>Answer</summary>**c)** `finalize`. `tap(fn)` with one function fires only on success.</details>

**3.** MCQ — An interceptor that never calls `next.handle()`: a) errors b) skips the handler c) hangs d) runs it anyway
<details><summary>Answer</summary>**b)** Returning your own observable skips the handler — and the pipes.</details>

**4.** MCQ — `ClassSerializerInterceptor` on a plain object: a) works b) throws c) silently does nothing d) returns null
<details><summary>Answer</summary>**c)** Decorators live on the class, so a plain object passes through untouched.</details>

**5.** MCQ — Bare `timeout(5000)` with no `catchError` produces: a) 504 b) 408 c) 500 d) 503
<details><summary>Answer</summary>**c)** `TimeoutError` is not an `HttpException`, so it becomes 500.</details>

**6.** MCQ — With interceptors A then B registered, exit order is: a) A then B b) B then A c) parallel d) undefined
<details><summary>Answer</summary>**b)** They nest, so they unwind in reverse.</details>

**7.** Short — Why must a timeout be an interceptor rather than middleware?
<details><summary>Answer</summary>The deadline must cover validation and the handler together, and only an interceptor wraps the whole operation while being able to convert the result into an HTTP error.</details>

**8.** Short — Why does `excludeExtraneousValues: true` make serialization safer?
<details><summary>Answer</summary>It turns the DTO into an allowlist, so a newly added database column is not serialized until someone explicitly adds `@Expose()`.</details>

**9.** Short — Why is omitting the principal from a cache key so dangerous?
<details><summary>Answer</summary>Two users requesting the same URL share one entry, so one user's response is served to another — a data leak rather than a broken feature.</details>

**10.** Short — Why can't an interceptor transform a `@Res()` handler's output?
<details><summary>Answer</summary>That handler writes the response directly, so no value flows back through the observable for the interceptor's operators to act on.</details>

**11.** Short — Where should error-to-status mapping live, and why not in `catchError`?
<details><summary>Answer</summary>In an exception filter, so the mapping is applied consistently to every error regardless of which interceptors a route happens to have.</details>

**12.** Output — What is logged when the handler throws?
```ts
return next.handle().pipe(tap(() => this.log('done')));
```
<details><summary>Answer</summary>Nothing. `tap` with a single function only observes the next notification.</details>

**13.** Output — Global interceptor A and route interceptor B both log `pre`/`post`. Order for a success?
<details><summary>Answer</summary>`A pre`, `B pre`, handler, `B post`, `A post`.</details>

**14.** Output — An interceptor returns `of({ cached: true })`. Does `ValidationPipe` run?
<details><summary>Answer</summary>No. Interceptors precede pipes, so returning early skips validation and the handler.</details>

**15.** Output — A DTO has `@Exclude() passwordHash`, the interceptor is applied, but the service returns `repo.findOne()` directly. What does the client see?
<details><summary>Answer</summary>The `passwordHash`. The returned value is a plain object, not a `UserDto` instance, so there are no decorators to apply.</details>

**16.** Debug — Latency dashboards show p99 of 40ms while users report timeouts. Cause?
<details><summary>Answer</summary>The metrics interceptor uses `tap(fn)`, so only successful requests are recorded — the slow and failing ones are absent entirely.</details>

**17.** Debug — One endpoint is missing the global response envelope. What do you check first?
<details><summary>Answer</summary>Whether its handler injects bare `@Res()`. That opts the route out of Nest's response handling, so `map` never sees a value.</details>

**18.** Debug — A metric series has millions of distinct labels and the backend is rejecting writes. Cause?
<details><summary>Answer</summary>The route label uses `request.originalUrl` rather than `request.route.path`, so every id becomes its own series. Use the route pattern.</details>

**19.** Debug — A slow endpoint returns 504 correctly but never appears in the duration histogram. Why?
<details><summary>Answer</summary>The metrics interceptor is registered inside the timeout interceptor, so the timeout converts the error above it and metrics never observes the request. Register metrics outermost.</details>

**20.** Debug — After enabling `excludeExtraneousValues`, every response is `{}`. Fix?
<details><summary>Answer</summary>No properties carry `@Expose()`. With extraneous values excluded, only explicitly exposed fields survive, so each DTO field needs the decorator.</details>

**21.** Code — Write an interceptor that retries a failed request twice with backoff, but only for GETs.
<details><summary>Answer</summary>

```ts
@Injectable()
export class RetryInterceptor implements NestInterceptor {
  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    if (ctx.switchToHttp().getRequest().method !== 'GET') return next.handle();

    return next.handle().pipe(
      retry({ count: 2, delay: (_e, n) => timer(2 ** n * 100) }),
    );
  }
}
```
Restricting to GET matters — retrying a non-idempotent POST can duplicate a charge.
</details>

**22.** Code — Write an envelope interceptor that can be disabled per route.
<details><summary>Answer</summary>

```ts
export const NO_ENVELOPE = 'noEnvelope';
export const RawResponse = () => SetMetadata(NO_ENVELOPE, true);

@Injectable()
export class TransformInterceptor implements NestInterceptor {
  constructor(private readonly reflector: Reflector) {}
  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    const raw = this.reflector.getAllAndOverride<boolean>(NO_ENVELOPE, [
      ctx.getHandler(), ctx.getClass(),
    ]);
    return raw ? next.handle() : next.handle().pipe(map((data) => ({ data })));
  }
}
```
</details>

**23.** Code — Serialize responses as DTO instances so `@Exclude()` actually applies.
<details><summary>Answer</summary>

```ts
@Injectable()
export class UsersService {
  async findOne(id: string): Promise<UserDto> {
    const row = await this.repo.findOne(id);
    if (!row) throw new NotFoundException();
    return plainToInstance(UserDto, row, { excludeExtraneousValues: true });
  }
}
```
Converting at the service boundary means every consumer — HTTP, queue, GraphQL — gets the safe shape, not just routes with the interceptor applied.
</details>

**24.** Scenario — A teammate proposes doing authorization in an interceptor because it can also shape the response based on the user's role. Evaluate it.
<details><summary>Answer</summary>Split the two concerns. The authorization decision belongs in a guard: guards run first, so a rejection costs nothing downstream, and a denial expressed as a guard is visible to anyone auditing which routes are protected — an interceptor that sometimes throws is far harder to reason about. Role-dependent response shaping genuinely does belong in an interceptor or in serialization groups, since only those layers see the response. Combining them means the same class decides both admission and presentation, so a change to output formatting sits in a security-critical file and a reviewer cannot tell which branches are load-bearing. Keep the guard for the yes/no and use `@SerializeOptions({ groups })` driven by the principal for the shaping.</details>

**25.** Scenario — Your team wants to cache expensive GET endpoints, and some responses depend on the user, some on the tenant, and some on nothing. How do you design it safely?
<details><summary>Answer</summary>Make the variance explicit and required rather than inferred. Default to not caching, and require an opt-in decorator that names what the response varies by — `@Cacheable({ ttl: 60, varyBy: ['user'] })` — so a developer adding caching must state the dimension and a reviewer can check it against the handler. Build the key in one shared function that refuses to run when `varyBy` is empty and the route is behind the auth guard, which turns the dangerous case into a startup or request-time error instead of a silent leak. Include the API version and the serialization group in the key too, since both change the response body without changing the URL. The reason for this ceremony is asymmetry of failure: a cache that misses is slow, and a cache that collides serves one customer's data to another, so the design should make the second mistake hard to express.</details>

---

## Chapter Summary

- Interceptors wrap the handler, so they are the only component that sees both the request and the response value, and they nest — outermost enters first and exits last.
- `tap(fn)` fires only on success; instrumentation must use `tap({ next, error })` or `finalize`, or it will report exclusively on requests that worked.
- Returning an observable without calling `next.handle()` skips the handler *and* the pipes, which is what makes caching possible and what makes cache keys built from unvalidated input.
- `ClassSerializerInterceptor` needs a real class instance and the interceptor applied; missing either fails silently and leaks excluded fields.
- `excludeExtraneousValues: true` inverts serialization from denylist to allowlist, so new columns are private until exposed.
- Instrumentation belongs outside the layers it measures — metrics registered inside a timeout interceptor never observe the timeouts.

---

## Practice Exercises

1. Write a metrics interceptor with `tap(fn)`, generate failures, then fix it with `tap({ next, error })` and compare the recorded data.
2. Build a caching interceptor without the user in the key and demonstrate the leak with two accounts.
3. Return an ORM entity through `ClassSerializerInterceptor` and confirm `@Exclude()` does nothing, then fix it with `plainToInstance`.
4. Register three interceptors and log entry and exit to observe the nesting.
5. Place a metrics interceptor inside and then outside a timeout interceptor, and compare what each records for a timing-out endpoint.

---

## Mini Project: An Observability Interceptor Suite

Build a reusable set of interceptors for a production API.

**Requirements:**

- Duration, status, and outcome recorded for every request, including failures and timeouts.
- Route labels use the route pattern so metric cardinality stays bounded.
- A per-route configurable timeout via decorator, converting to 504.
- A response envelope with an opt-out decorator for probes and downloads.
- A test proving that a timing-out request still produces exactly one metric with `outcome=error`.

---

**Next:** [Chapter 14 — Pipes and Validation →](14-pipes-validation.md)
