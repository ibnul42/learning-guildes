# Appendix B — 112 Coding Challenges

Every challenge has a worked solution, the reasoning behind the approach, and the wrong answer people usually give first. Attempt each one before opening the solution — the value is in the attempt, and reading a solution feels like learning in a way that doesn't survive contact with an interview.

**Difficulty:** `easy` — you should get this in five minutes. `medium` — twenty, with an edge case you'll miss first time. `hard` — an hour, or a hint.

Unless stated otherwise, assume NestJS 11, TypeScript 5, and a global `ValidationPipe` with `whitelist` and `transform` enabled.

---

## Contents

| Section | Count |
|---|---|
| [1. Modules and Dependency Injection](#1-modules-and-dependency-injection) | 12 |
| [2. Providers and Factories](#2-providers-and-factories) | 10 |
| [3. Controllers and Routing](#3-controllers-and-routing) | 10 |
| [4. Guards and Interceptors](#4-guards-and-interceptors) | 12 |
| [5. Pipes and Validation](#5-pipes-and-validation) | 10 |
| [6. Exception Handling](#6-exception-handling) | 8 |
| [7. Database and Transactions](#7-database-and-transactions) | 12 |
| [8. Authentication and Authorization](#8-authentication-and-authorization) | 10 |
| [9. Async: Queues, Events, Scheduling](#9-async-queues-events-scheduling) | 10 |
| [10. Testing](#10-testing) | 10 |
| [11. Output Prediction and Debugging](#11-output-prediction-and-debugging) | 8 |

---

## 1. Modules and Dependency Injection

### 1. Share one provider between two modules  `easy`
`RedisClient` is registered in both `UsersModule` and `TasksModule`. Make them share one instance.
<details><summary>Solution</summary>

```ts
@Module({ providers: [RedisClient], exports: [RedisClient] })
export class RedisModule {}

@Module({ imports: [RedisModule], providers: [UsersService] })
export class UsersModule {}

@Module({ imports: [RedisModule], providers: [TasksService] })
export class TasksModule {}
```
**Why:** a provider is registered once, in the module that owns it, and shared by exporting.
**Common wrong answer:** leaving it in both `providers` arrays. It resolves, and creates two instances with independent state — the bug appears later as a cache that is sometimes stale.
</details>

### 2. Make a provider available application-wide  `easy`
`ConfigService` is needed in twenty modules. Avoid importing its module twenty times.
<details><summary>Solution</summary>

```ts
@Global()
@Module({ providers: [ConfigService], exports: [ConfigService] })
export class ConfigModule {}
// imported ONCE, in AppModule
```
**Why:** `@Global()` widens visibility of the module's exports.
**Common wrong answer:** omitting `exports`. Global affects who can see the exports, not whether there are any — with nothing exported there is nothing to see.
</details>

### 3. Inject an interface  `easy`
`Logger` is an interface. Make it injectable.
<details><summary>Solution</summary>

```ts
export abstract class Logger {
  abstract log(message: string): void;
}
providers: [{ provide: Logger, useClass: PinoLogger }]
// consumers: constructor(private readonly logger: Logger) {}
```
**Why:** an abstract class survives compilation, so it works as both a contract and a DI token.
**Common wrong answer:** keeping the interface and hoping. It is erased, `design:paramtypes` records `Object`, and resolution fails. The alternative is `@Inject('LOGGER')` with a string token, which works and loses type-directed navigation.
</details>

### 4. Break a circular dependency  `medium`
`UsersService` and `TasksService` need each other, in different modules.
<details><summary>Solution</summary>

```ts
// both modules
@Module({ imports: [forwardRef(() => TasksModule)], /* … */ })

// both services
constructor(
  @Inject(forwardRef(() => TasksService)) private readonly tasks: TasksService,
) {}
```
**Why:** both the module imports and the injection sites need deferring; fixing one side leaves the cycle.
**Common wrong answer:** treating `forwardRef` as the fix. It resolves the mechanism; the design cycle remains, and the real remedies are extracting the shared concept or inverting one direction with an event.
</details>

### 5. Build a dynamic module  `medium`
`RateLimitModule.forRoot({ windowMs, max })` exposing a `RateLimitService`.
<details><summary>Solution</summary>

```ts
export const RATE_LIMIT_OPTIONS = Symbol('RATE_LIMIT_OPTIONS');

@Module({})
export class RateLimitModule {
  static forRoot(options: RateLimitOptions): DynamicModule {
    return {
      module: RateLimitModule,
      providers: [
        { provide: RATE_LIMIT_OPTIONS, useValue: options },
        RateLimitService,
      ],
      exports: [RateLimitService],
    };
  }
}
```
**Why:** a `Symbol` token cannot collide with another library's `'OPTIONS'`.
**Common wrong answer:** a string token. Two modules registering `'OPTIONS'` overwrite each other, and which wins depends on module resolution order.
</details>

### 6. Add an async variant  `medium`
Let the previous module read its limit from `ConfigService`.
<details><summary>Solution</summary>

```ts
static forRootAsync(opts: {
  imports?: ModuleMetadata['imports'];
  inject?: any[];
  useFactory: (...args: any[]) => Promise<RateLimitOptions> | RateLimitOptions;
}): DynamicModule {
  return {
    module: RateLimitModule,
    imports: opts.imports ?? [],
    providers: [
      { provide: RATE_LIMIT_OPTIONS, useFactory: opts.useFactory, inject: opts.inject ?? [] },
      RateLimitService,
    ],
    exports: [RateLimitService],
  };
}
```
**Why:** the factory's parameters come from `inject`, positionally, and the providing module must be visible to the dynamic module itself.
**Common wrong answer:** omitting `imports`, so `ConfigService` is not resolvable and the factory receives `undefined`.
</details>

### 7. Replace both variants with a builder  `medium`
Generate `forRoot` and `forRootAsync` instead of writing them.
<details><summary>Solution</summary>

```ts
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<RateLimitOptions>()
    .setClassMethodName('forRoot')
    .build();

@Module({ providers: [RateLimitService], exports: [RateLimitService] })
export class RateLimitModule extends ConfigurableModuleClass {}
```
**Why:** both variants and the options token are generated, and the async form is the one that is easy to get subtly wrong by hand.
**Common wrong answer:** calling `.build()` before `.setClassMethodName()` and wondering why the methods are named `register`.
</details>

### 8. Deprecate a token without duplicating the instance  `medium`
`'MAILER'` should become `MailService`, with old consumers still working.
<details><summary>Solution</summary>

```ts
@Module({
  providers: [MailService, { provide: 'MAILER', useExisting: MailService }],
  exports: [MailService, 'MAILER'],
})
export class MailModule {}
```
**Why:** `useExisting` is an alias — one instance behind two names.
**Common wrong answer:** `useClass: MailService`, which constructs a *second* mailer. Two connection pools, two rate-limiter states, and a bug that only appears under load in whichever half of the code uses the old token.
</details>

### 9. Resolve a request-scoped provider from a singleton  `medium`
A singleton dispatcher needs a `Scope.REQUEST` handler, without becoming request-scoped itself.
<details><summary>Solution</summary>

```ts
@Injectable()
export class Dispatcher {
  constructor(private readonly moduleRef: ModuleRef) {}

  async handle(payload: Payload) {
    const contextId = ContextIdFactory.create();
    const handler = await this.moduleRef.resolve(ScopedHandler, contextId);
    return handler.run(payload);
  }
}
```
**Why:** `resolve` creates an instance per context id; the same id returns the same instance.
**Common wrong answer:** `moduleRef.get(ScopedHandler)`, which throws — there is no single instance to return.
</details>

### 10. Curate a bundle of infrastructure modules  `easy`
Three modules are imported together everywhere. Reduce the repetition without `@Global()`.
<details><summary>Solution</summary>

```ts
@Module({
  imports: [DatabaseModule, CacheModule, ConfigModule],
  exports: [DatabaseModule, CacheModule, ConfigModule],
})
export class InfrastructureModule {}
```
**Why:** re-exporting passes through each module's public surface while the dependency stays visible in `imports`.
**Common wrong answer:** making all three global. It removes the repetition and the ability to read a module and know what it depends on — and test modules stop matching production.
</details>

### 11. Diagnose from the error alone  `medium`
`Nest can't resolve dependencies of the ReportsService (?, MailService). Please make sure that the argument TaskRepository at index [0] is available in the ReportsModule context.` Name the two possible fixes.
<details><summary>Solution</summary>

Either `TaskRepository` is not in its owning module's `exports`, or `ReportsModule` does not import that module.

```ts
@Module({ providers: [TaskRepository], exports: [TaskRepository] })  // fix A
export class TasksModule {}

@Module({ imports: [TasksModule], providers: [ReportsService] })      // fix B
export class ReportsModule {}
```
**Why:** the `?` marks the failed parameter, `index [0]` locates it, and the named context is the module whose visibility was searched.
**Common wrong answer:** adding `TaskRepository` to `ReportsModule`'s own `providers`. The error goes away and a second instance appears.
</details>

### 12. Detect duplicate registrations at startup  `hard`
Write a provider that logs any token registered in more than one module.
<details><summary>Solution</summary>

```ts
@Injectable()
export class WiringAudit implements OnApplicationBootstrap {
  private readonly logger = new Logger(WiringAudit.name);
  constructor(private readonly modules: ModulesContainer) {}

  onApplicationBootstrap(): void {
    const seen = new Map<string, string[]>();

    for (const [, module] of this.modules) {
      for (const [token] of module.providers) {
        const key = typeof token === 'function' ? token.name : String(token);
        seen.set(key, [...(seen.get(key) ?? []), module.metatype?.name ?? 'unknown']);
      }
    }

    for (const [token, modules] of seen) {
      // A provider in two modules is two instances — almost always a bug.
      if (modules.length > 1) {
        this.logger.warn(`${token} registered in: ${modules.join(', ')}`);
      }
    }
  }
}
```
**Why:** `ModulesContainer` exposes the resolved graph, so this is a fact about the application rather than a lint heuristic.
**Common wrong answer:** grepping for the class name in `providers` arrays, which misses custom providers and dynamic modules.
</details>

---

## 2. Providers and Factories

### 13. Provide a value that depends on configuration  `easy`
`new Stripe(process.env.STRIPE_KEY)` is `undefined` in production. Fix it.
<details><summary>Solution</summary>

```ts
{
  provide: 'STRIPE',
  inject: [ConfigService],
  useFactory: (config: ConfigService) =>
    new Stripe(config.getOrThrow('STRIPE_KEY'), { apiVersion: '2024-06-20' }),
}
```
**Why:** `useValue` evaluates at *import* time, before `ConfigModule` has loaded `.env`.
**Common wrong answer:** keeping `useValue` and moving the import order around. Import order cannot fix it; the value must be produced lazily.
</details>

### 14. Bound an async provider  `medium`
A database provider hangs bootstrap when the host is unreachable. Make it fail instead.
<details><summary>Solution</summary>

```ts
{
  provide: DB_POOL,
  inject: [ConfigService],
  useFactory: async (config: ConfigService) => {
    const pool = createPool({ url: config.getOrThrow('DB_URL') });
    const timeout = new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error('DB connect timed out after 5s')), 5_000),
    );
    await Promise.race([pool.query('SELECT 1'), timeout]);
    return pool;
  },
}
```
**Why:** bootstrap awaits async providers, so an unbounded one produces a process that never listens, with no error and no open port.
**Common wrong answer:** relying on the driver's own timeout, which is often absent or far longer than a deployment's startup budget.
</details>

### 15. Select an implementation from configuration  `medium`
Choose between `StripeGateway` and `PaypalGateway` at runtime, behind one token.
<details><summary>Solution</summary>

```ts
{
  provide: PaymentGateway,
  inject: [PAYMENT_OPTIONS, Logger],
  useFactory: (options: PaymentOptions, logger: Logger): PaymentGateway =>
    options.provider === 'stripe'
      ? new StripeGateway(options.apiKey, logger)
      : new PaypalGateway(options.apiKey, logger),
}
```
**Why:** the decision is data-driven and visible in one place, with its dependencies declared.
**Common wrong answer:** `useClass: process.env.X === 'stripe' ? A : B`. It is evaluated at import time, invisible from the consumer, and a misspelled variable silently ships the wrong gateway.
</details>

### 16. Make a factory dependency optional  `easy`
A factory should work without `MetricsClient` registered.
<details><summary>Solution</summary>

```ts
{
  provide: SERVICE,
  inject: [ConfigService, { token: METRICS, optional: true }],
  useFactory: (config: ConfigService, metrics?: MetricsClient) =>
    new Service(config, metrics),
}
```
**Why:** the object form of an `inject` entry marks it optional.
**Common wrong answer:** `@Optional()` on the factory parameter — decorators do not apply to factory arguments.
</details>

### 17. Register a global pipe that needs injection  `easy`
`ValidationPipe` is fine, but a custom pipe needs `ConfigService`.
<details><summary>Solution</summary>

```ts
@Module({
  providers: [{ provide: APP_PIPE, useClass: TenantAwarePipe }],
})
export class AppModule {}
```
**Why:** `APP_PIPE` makes it a provider, so the container constructs it with dependencies — and it appears in testing modules.
**Common wrong answer:** `app.useGlobalPipes(new TenantAwarePipe(???))` in `main.ts`, which cannot inject and is absent from every test.
</details>

### 18. Order two global interceptors  `easy`
Metrics must observe timeouts. Which is registered first?
<details><summary>Solution</summary>

```ts
providers: [
  { provide: APP_INTERCEPTOR, useClass: MetricsInterceptor },   // outermost
  { provide: APP_INTERCEPTOR, useClass: TimeoutInterceptor },
]
```
**Why:** interceptors nest, and `APP_INTERCEPTOR` runs in registration order — so the first registered is the outer layer and sees the timeout error the inner one raises.
**Common wrong answer:** registering metrics innermost to "measure only my code". A request that times out then produces no metric at all, so the slowest requests are the ones missing from the histogram.
</details>

### 19. Provide a clock you can freeze in tests  `easy`
Remove `new Date()` from a service without branching on `NODE_ENV`.
<details><summary>Solution</summary>

```ts
export abstract class Clock { abstract now(): Date; }

@Injectable() export class SystemClock extends Clock { now() { return new Date(); } }
export class FixedClock extends Clock {
  constructor(private readonly at: Date) { super(); }
  now() { return this.at; }
}

// production
providers: [{ provide: Clock, useClass: SystemClock }]

// test
.overrideProvider(Clock).useValue(new FixedClock(new Date('2026-01-01')))
```
**Why:** the substitution is explicit at the call site rather than hidden in an environment check.
**Common wrong answer:** `jest.useFakeTimers()` everywhere, which also freezes timers you did not intend to affect.
</details>

### 20. Close a resource on shutdown  `easy`
A Redis client leaks connections across restarts.
<details><summary>Solution</summary>

```ts
@Module({ providers: [redisProvider], exports: [CacheService] })
export class CacheModule implements OnApplicationShutdown {
  constructor(@Inject(REDIS_CLIENT) private readonly client: RedisClientType | null) {}
  async onApplicationShutdown(): Promise<void> {
    await this.client?.quit();
  }
}
```
**Why:** `quit()` drains rather than dropping the socket.
**Common wrong answer:** implementing the hook and forgetting `app.enableShutdownHooks()` in `main.ts`, which makes the whole thing dead code.
</details>

### 21. Express "disabled" without a null-object  `medium`
Caching should be optional. Make absence visible at every call site.
<details><summary>Solution</summary>

```ts
useFactory: (options: CacheOptions): RedisClientType | null => {
  if (!options.url) {
    if (options.isProduction) throw new Error('REDIS_URL is required in production');
    return null;
  }
  return createClient({ url: options.url });
}

// consumer
async get<T>(key: string): Promise<T | null> {
  if (!this.client) return null;      // the compiler forced this branch
  …
}
```
**Why:** `RedisClientType | null` makes every consumer acknowledge the disabled case at compile time.
**Common wrong answer:** a no-op client implementing the same interface. It reads more cleanly and makes a disabled cache indistinguishable from a working one — which is exactly the thing you want to be able to see.
</details>

### 22. Fail closed on an unset environment  `medium`
A test gateway must never be selectable in production.
<details><summary>Solution</summary>

```ts
const LOCAL_ENVS = ['development', 'test'];

useFactory: (options: PaymentOptions): PaymentGateway => {
  if (options.provider === 'fake') {
    // Allowlist, not `!== 'production'` — an unset NODE_ENV must fail closed.
    if (!LOCAL_ENVS.includes(process.env.NODE_ENV ?? '')) {
      throw new Error('FakeGateway is not permitted outside development');
    }
    return new FakeGateway();
  }
  return new StripeGateway(options.apiKey);
}
```
**Why:** any code path whose only protection is `!== 'production'` will eventually run in production with the variable unset or misspelled.
**Common wrong answer:** `process.env.NODE_ENV !== 'production'`, which is true when the variable is missing entirely.
</details>

---

## 3. Controllers and Routing

### 23. Fix a route that is never reached  `easy`
`GET /users/me` returns "invalid UUID".
<details><summary>Solution</summary>

```ts
@Controller('users')
export class UsersController {
  @Get('me') me() {}                                        // static FIRST
  @Get(':id') findOne(@Param('id', ParseUUIDPipe) id: string) {}
}
```
**Why:** routes match in registration order, which follows method declaration order.
**Common wrong answer:** adding a regex constraint to `:id`. It works and leaves the ordering hazard in place for the next static route someone adds.
</details>

### 24. Coerce and bound pagination  `easy`
`?page=2&limit=10000` must not scan the table.
<details><summary>Solution</summary>

```ts
export class PageQueryDto {
  @IsOptional() @Type(() => Number) @IsInt() @Min(1)
  page: number = 1;

  @IsOptional() @Type(() => Number) @IsInt() @Min(1) @Max(100)
  limit: number = 20;
}

@Get() findAll(@Query() query: PageQueryDto) {}
```
**Why:** `@Type` performs the string→number conversion that `@IsInt` then validates, and `@Max` is the actual protection.
**Common wrong answer:** `@Query('limit') limit: number`. The annotation is erased; you get the string `'10000'`, and `Math.min` further down is a check someone will forget to add to the next endpoint.
</details>

### 25. Return 204 correctly  `easy`
A delete endpoint returns 200 with `{ deleted: true }`.
<details><summary>Solution</summary>

```ts
@Delete(':id')
@HttpCode(HttpStatus.NO_CONTENT)
remove(@Param('id', ParseUUIDPipe) id: string): Promise<void> {
  return this.tasks.remove(id);
}
```
**Why:** 204 must carry no body, and the return type documents that.
**Common wrong answer:** setting 204 while still returning an object. The body is discarded, so the code implies something the response cannot deliver.
</details>

### 26. Set a cookie and return a body  `easy`
Setting a cookie broke the global response envelope on one route.
<details><summary>Solution</summary>

```ts
@Post('login')
async login(@Res({ passthrough: true }) res: Response) {
  res.cookie('refresh_token', token, { httpOnly: true, secure: true, sameSite: 'strict' });
  return { accessToken };            // Nest still serializes this
}
```
**Why:** `passthrough: true` keeps Nest's response handling; bare `@Res()` hands it over entirely.
**Common wrong answer:** bare `@Res()` with `res.json(...)`. It works, and silently exempts that route from every response-transforming interceptor.
</details>

### 27. Redirect to a target decided at runtime  `easy`
<details><summary>Solution</summary>

```ts
@Get('go/:code')
@Redirect()                              // no default; the return value supplies it
async go(@Param('code') code: string) {
  const target = await this.links.resolve(code);
  if (!target) throw new NotFoundException();
  return { url: target, statusCode: 302 };
}
```
**Why:** returning `{ url, statusCode }` overrides the decorator's arguments.
**Common wrong answer:** injecting `@Res()` and calling `res.redirect()`, which opts the route out of the pipeline for no gain.
</details>

### 28. Prevent an open redirect  `medium`
`?next=` lets anyone bounce users off your domain.
<details><summary>Solution</summary>

```ts
const ALLOWED = new Set(['/dashboard', '/settings', '/tasks']);

if (!next?.startsWith('/') || next.startsWith('//') || !ALLOWED.has(next)) {
  throw new BadRequestException('Invalid redirect target');
}
```
**Why:** `//evil.example` is protocol-relative — it passes a naive `startsWith('/')` and navigates off-site.
**Common wrong answer:** checking only `startsWith('/')`, which is the near-miss the attacker uses.
</details>

### 29. Serve two API versions from one service  `medium`
v2 renames `title` to `name`.
<details><summary>Solution</summary>

```ts
@Controller({ path: 'tasks', version: '1' })
export class TasksV1Controller {
  constructor(private readonly tasks: TasksService) {}
  @Get() findAll() { return this.tasks.findAll(); }
}

@Controller({ path: 'tasks', version: '2' })
export class TasksV2Controller {
  constructor(private readonly tasks: TasksService) {}
  @Get() async findAll() {
    return (await this.tasks.findAll()).map(({ title, ...rest }) => ({ ...rest, name: title }));
  }
}
```
**Why:** duplicating the controller is cheap; duplicating the business logic is not.
**Common wrong answer:** branching on a version parameter inside one handler, which grows a conditional for every future change.
</details>

### 30. Normalise a query parameter that may repeat  `medium`
`?tags=a` yields a string; `?tags=a&tags=b` yields an array.
<details><summary>Solution</summary>

```ts
export class FilterDto {
  @IsOptional()
  @Transform(({ value }) => ([] as string[]).concat(value ?? []))
  @IsArray()
  @IsString({ each: true })
  tags: string[] = [];
}
```
**Why:** normalising in `@Transform` makes the shape stable regardless of how many values arrive.
**Common wrong answer:** `Array.isArray(tags) ? tags : [tags]` in the handler, repeated at every call site and forgotten at one.
</details>

### 31. Convert Express 4 routes to Express 5  `easy`
`GET /files/*` and `GET /report/:name.:ext?`.
<details><summary>Solution</summary>

```ts
@Get('files/*path')
findFile(@Param('path') path: string) {}

@Get('report/:name{.:ext}')
getReport(@Param('name') name: string, @Param('ext') ext?: string) {}
```
**Why:** Express 5 requires named wildcards and brace syntax for optional segments.
**Common wrong answer:** assuming the old syntax still works. It throws at startup, and the error names `path-to-regexp` rather than your controller.
</details>

### 32. Exclude health checks from the global prefix  `easy`
Probes must stay at `/health` while the API moves to `/api/v1`.
<details><summary>Solution</summary>

```ts
app.setGlobalPrefix('api', { exclude: ['health', 'metrics'] });
app.enableVersioning({ type: VersioningType.URI, defaultVersion: '1' });
```
**Why:** the exclusion removes both the prefix and versioning, which is what orchestrators and scrapers expect.
**Common wrong answer:** hard-coding `/api/v1/health` in the probe configuration, which breaks on the next version bump.
</details>

---

## 4. Guards and Interceptors

### 33. Return the right status for a missing credential  `easy`
A guard returns 403 when the token is absent.
<details><summary>Solution</summary>

```ts
if (!token) throw new UnauthorizedException('Missing bearer token');   // 401
```
**Why:** 401 tells the client to authenticate and retry; 403 tells it not to bother. Returning `false` always produces 403.
**Common wrong answer:** `return false`, which sends well-behaved clients into pointless retry loops or stops them retrying when they should.
</details>

### 34. Implement `@Public()`  `easy`
<details><summary>Solution</summary>

```ts
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);

canActivate(ctx: ExecutionContext): boolean {
  const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
    ctx.getHandler(),
    ctx.getClass(),
  ]);
  if (isPublic) return true;
  // …
}
```
**Why:** checking both handler and class lets a whole controller be marked public.
**Common wrong answer:** `reflector.get(IS_PUBLIC_KEY, ctx.getHandler())`, which ignores controller-level metadata.
</details>

### 35. Make a guard fail closed  `medium`
`RolesGuard` throws a `TypeError` when the auth guard did not run.
<details><summary>Solution</summary>

```ts
const { user } = ctx.switchToHttp().getRequest();
if (!user) throw new UnauthorizedException();     // explicit 401, not a crash
if (!required.some((r) => user.roles?.includes(r))) throw new ForbiddenException();
```
**Why:** a `TypeError` produces a 500, which is retried by clients and sometimes routed past authorization by an upstream proxy — a denial is the correct outcome.
**Common wrong answer:** trusting registration order. It is an invariant held in one providers array, and `@UseGuards(RolesGuard)` on a route bypasses it entirely.
</details>

### 36. Write a transport-agnostic guard  `medium`
A global HTTP guard breaks a new WebSocket gateway.
<details><summary>Solution</summary>

```ts
private getRequest(ctx: ExecutionContext) {
  switch (ctx.getType<GqlContextType>()) {
    case 'ws':      return ctx.switchToWs().getClient().handshake;
    case 'graphql': return GqlExecutionContext.create(ctx).getContext().req;
    default:        return ctx.switchToHttp().getRequest();
  }
}
```
**Why:** `switchToHttp()` yields nothing outside HTTP, so a globally registered guard must branch.
**Common wrong answer:** excluding the gateway from the guard, which leaves it unauthenticated.
</details>

### 37. Instrument both outcomes  `easy`
A metrics interceptor reports 100% success while users see 500s.
<details><summary>Solution</summary>

```ts
return next.handle().pipe(
  tap({
    next: () => record('success'),
    error: (err) => record('error', err),
  }),
);
```
**Why:** `tap(fn)` with a single function subscribes only to the next notification, so failures are invisible.
**Common wrong answer:** adding `catchError` that swallows the error to record it — the request then succeeds with an undefined body.
</details>

### 38. Add a per-route timeout  `medium`
Default 5s, overridable per handler, surfacing as 504.
<details><summary>Solution</summary>

```ts
const ms = this.reflector.getAllAndOverride<number>(TIMEOUT_KEY, [
  ctx.getHandler(), ctx.getClass(),
]) ?? 5_000;

return next.handle().pipe(
  timeout(ms),
  catchError((err) =>
    err instanceof TimeoutError
      ? throwError(() => new GatewayTimeoutException(`Exceeded ${ms}ms`))
      : throwError(() => err),
  ),
);
```
**Why:** a bare `timeout()` emits `TimeoutError`, which is not an `HttpException` and becomes a 500.
**Common wrong answer:** middleware with `setTimeout`, which cannot cover validation and cannot convert the result into a response.
</details>

### 39. Cache GET responses without leaking  `hard`
<details><summary>Solution</summary>

```ts
async intercept(ctx: ExecutionContext, next: CallHandler): Promise<Observable<unknown>> {
  const req = ctx.switchToHttp().getRequest();
  if (req.method !== 'GET') return next.handle();

  // Identity is not optional — omit it and one user's response is served to another.
  const key = ['v1', req.user?.tenantId ?? 'none', req.user?.id ?? 'anon', req.originalUrl].join(':');

  const hit = await this.cache.get(key);
  if (hit !== undefined) return of(hit);              // `undefined`, not falsy

  return next.handle().pipe(tap((v) => this.cache.set(key, v, 60_000)));
}
```
**Why:** interceptors run before pipes, so the key is built from unvalidated input while the user *is* known — identity is available and easy to forget.
**Common wrong answer:** keying on `req.originalUrl` alone. It works in every single-user test and leaks across accounts in production.
</details>

### 40. Wrap responses with an opt-out  `medium`
<details><summary>Solution</summary>

```ts
export const NO_ENVELOPE = 'noEnvelope';
export const RawResponse = () => SetMetadata(NO_ENVELOPE, true);

intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
  const raw = this.reflector.getAllAndOverride<boolean>(NO_ENVELOPE, [
    ctx.getHandler(), ctx.getClass(),
  ]);
  return raw ? next.handle() : next.handle().pipe(map((data) => ({ data })));
}
```
**Why:** probes, downloads, and redirects must not be wrapped, and a decorator keeps the exception next to the route.
**Common wrong answer:** listing exempt paths inside the interceptor, which drifts from the routes immediately.
</details>

### 41. Strip a field that must never ship  `medium`
`passwordHash` reaches clients despite `@Exclude()`.
<details><summary>Solution</summary>

```ts
// service — convert at the boundary so every transport benefits
return plainToInstance(UserDto, row, { excludeExtraneousValues: true });

// module — allowlist by default
{
  provide: APP_INTERCEPTOR,
  inject: [Reflector],
  useFactory: (r: Reflector) =>
    new ClassSerializerInterceptor(r, { excludeExtraneousValues: true }),
}
```
**Why:** decorators live on the class, so a plain ORM row has nothing to exclude — and `excludeExtraneousValues` makes new columns private until explicitly exposed.
**Common wrong answer:** destructuring the field out in the controller. Correct exactly as often as everyone remembers it.
</details>

### 42. Retry idempotent requests only  `medium`
<details><summary>Solution</summary>

```ts
intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
  if (ctx.switchToHttp().getRequest().method !== 'GET') return next.handle();
  return next.handle().pipe(retry({ count: 2, delay: (_e, n) => timer(2 ** n * 100) }));
}
```
**Why:** retrying a non-idempotent `POST` can duplicate a charge.
**Common wrong answer:** a global `retry(2)` with no method check, which turns one payment into three during a transient failure.
</details>

### 43. Rate-limit per user in a guard  `medium`
<details><summary>Solution</summary>

```ts
async canActivate(ctx: ExecutionContext): Promise<boolean> {
  const req = ctx.switchToHttp().getRequest();
  const key = `rl:${req.user?.id ?? req.ip}`;              // principal, IP fallback
  const count = await this.cache.increment(key, 60);
  if (count > 100) throw new HttpException('Rate limit exceeded', HttpStatus.TOO_MANY_REQUESTS);
  return true;
}
```
**Why:** keying on the user where present stops one shared NAT from exhausting everyone's quota.
**Common wrong answer:** IP-only, which both punishes shared networks and is forgeable if `trust proxy` is wrong.
</details>

### 44. Order guards so authorization can read the principal  `easy`
<details><summary>Solution</summary>

```ts
providers: [
  { provide: APP_GUARD, useClass: JwtAuthGuard },   // establishes request.user
  { provide: APP_GUARD, useClass: RolesGuard },     // reads it
  { provide: APP_GUARD, useClass: ScopesGuard },
]
```
**Why:** `APP_GUARD` is multi-provider and runs in registration order.
**Common wrong answer:** assuming order does not matter, which yields a 500 rather than a 401 when the order is wrong.
</details>

---

## 5. Pipes and Validation

### 45. Default a query parameter  `easy`
`?page` absent throws 400 instead of defaulting to 1.
<details><summary>Solution</summary>

```ts
@Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number
```
**Why:** pipes compose left to right, so the default must be applied before parsing.
**Common wrong answer:** the reverse order, where `ParseIntPipe` receives `undefined` and throws.
</details>

### 46. Close the nested-validation hole  `medium`
Invalid items inside an array are accepted.
<details><summary>Solution</summary>

```ts
export class CreateOrderDto {
  @IsArray()
  @ArrayMinSize(1)
  @ArrayMaxSize(50)
  @ValidateNested({ each: true })
  @Type(() => OrderItemDto)          // WITHOUT this, every item validates trivially
  items: OrderItemDto[];
}
```
**Why:** `design:type` records only `Array`; the element type is erased, so without `@Type()` the items stay plain objects with no class to validate against.
**Common wrong answer:** assuming `@ValidateNested()` is sufficient. It fails **open** — a 201 for input that should have been rejected.
</details>

### 47. Write a sortable-column pipe  `medium`
`?sort=field:asc` must not allow arbitrary columns.
<details><summary>Solution</summary>

```ts
@Injectable()
export class ParseSortPipe implements PipeTransform<string, Sort> {
  constructor(private readonly allowed: readonly string[]) {}

  transform(value: string): Sort {
    const [field, direction = 'asc'] = (value ?? '').split(':');
    if (!this.allowed.includes(field)) {
      throw new BadRequestException(`sort field must be one of: ${this.allowed.join(', ')}`);
    }
    if (direction !== 'asc' && direction !== 'desc') {
      throw new BadRequestException('sort direction must be asc or desc');
    }
    return { field, direction };
  }
}
```
**Why:** column names cannot be bound as SQL parameters, so an allowlist is the only safe approach.
**Common wrong answer:** escaping the value. Identifiers are not escapable the way values are — this is the injection vector that survives an ORM.
</details>

### 48. Derive an update DTO  `easy`
Every field optional, except an immutable one that must be rejected outright.
<details><summary>Solution</summary>

```ts
import { PartialType, OmitType } from '@nestjs/swagger';

export class UpdateUserDto extends PartialType(
  OmitType(CreateUserDto, ['email'] as const),
) {}
```
**Why:** composing `OmitType` inside `PartialType` removes the field entirely, so with `forbidNonWhitelisted` a client submitting it gets a 400.
**Common wrong answer:** duplicating the DTO by hand, which drifts silently the first time a rule changes.
</details>

### 49. Reject unknown properties  `easy`
A client misspells `emailAddress` and gets a 201.
<details><summary>Solution</summary>

```ts
new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true })
```
**Why:** `whitelist` alone strips silently, so the client believes a malformed request succeeded.
**Common wrong answer:** `whitelist` without `forbidNonWhitelisted`, which is also the configuration that hides a mass-assignment attempt instead of reporting it.
</details>

### 50. Validate a date range parameter  `medium`
`?range=2026-01-01..2026-02-01`.
<details><summary>Solution</summary>

```ts
@Injectable()
export class ParseDateRangePipe implements PipeTransform<string, DateRange> {
  transform(value: string, metadata: ArgumentMetadata): DateRange {
    const [fromRaw, toRaw] = (value ?? '').split('..');
    const from = new Date(fromRaw), to = new Date(toRaw);

    if (Number.isNaN(from.getTime()) || Number.isNaN(to.getTime())) {
      throw new BadRequestException(`${metadata.data ?? 'range'} must be "YYYY-MM-DD..YYYY-MM-DD"`);
    }
    if (from > to) throw new BadRequestException('range start must precede end');
    return { from, to };
  }
}
```
**Why:** `metadata.data` names the parameter in the error, so the message is actionable.
**Common wrong answer:** parsing in the service, which duplicates the check at every caller and returns a 500 for malformed input.
</details>

### 51. Replace `disableErrorMessages`  `medium`
Hide the schema without blinding clients.
<details><summary>Solution</summary>

```ts
new ValidationPipe({
  exceptionFactory: (errors) =>
    new BadRequestException({
      code: 'VALIDATION_FAILED',
      // Field paths and constraint names only — never the received value.
      errors: errors.map((e) => ({
        field: e.property,
        codes: Object.keys(e.constraints ?? {}),
      })),
    }),
})
```
**Why:** clients can still render form errors; nothing echoes back user input or constraint internals.
**Common wrong answer:** `disableErrorMessages: true`, which replaces every failure with a bare 400 and removes your own fastest diagnostic.
</details>

### 52. Enforce uniqueness correctly  `medium`
A `UniqueEmailPipe` queries the database.
<details><summary>Solution</summary>

```ts
// Service, not pipe — and enforced by a constraint, not a prior read.
async create(dto: CreateUserDto): Promise<User> {
  try {
    return await this.repo.save({ email: dto.email, name: dto.name });
  } catch (err) {
    if ((err as { code?: string }).code === '23505') {
      throw new ConflictException('Email already registered');
    }
    throw err;
  }
}
```
**Why:** uniqueness is a fact about system state, so it must apply to every entry point and be atomic — a check-then-insert has a race window.
**Common wrong answer:** the pipe. It only covers HTTP, races under concurrency, and duplicates a check the service needs anyway.
</details>

### 53. Coerce a boolean query flag  `easy`
`?active=false` arrives as the string `'false'`, which is truthy.
<details><summary>Solution</summary>

```ts
export class FilterDto {
  @IsOptional()
  @Transform(({ value }) => value === 'true' || value === true)
  @IsBoolean()
  active?: boolean;
}
```
**Why:** an explicit transform is unambiguous where `enableImplicitConversion` is not.
**Common wrong answer:** `@Type(() => Boolean)`, which uses `Boolean('false')` — `true`.
</details>

### 54. Validate a discriminated union  `hard`
A config object whose shape depends on a `type` field.
<details><summary>Solution</summary>

```ts
export class ConfigDto {
  @ValidateNested()
  @Type(() => BaseTarget, {
    discriminator: {
      property: 'type',
      subTypes: [
        { value: HttpTarget, name: 'http' },
        { value: QueueTarget, name: 'queue' },
      ],
    },
    keepDiscriminatorProperty: true,
  })
  target: HttpTarget | QueueTarget;
}
```
**Why:** each variant keeps its own rules, so `whitelist` stays meaningful — a single union-shaped DTO would have to permit the union of all fields, letting a payload mix keys from two variants.
**Common wrong answer:** one DTO with every field optional plus a manual `if (type === …)` check, which validates nothing the pipe can enforce.
</details>

---

## 6. Exception Handling

### 55. Return the intended status  `easy`
A service throws `new Error('Task not found')` and the client gets 500.
<details><summary>Solution</summary>

```ts
throw new NotFoundException(`Task ${id} not found`);
```
**Why:** only `HttpException` subclasses carry a status; anything else becomes a 500 with its message deliberately withheld.
**Common wrong answer:** catching in the controller and re-throwing, which works and duplicates the mapping at every call site.
</details>

### 56. Write a catch-all that preserves statuses  `medium`
<details><summary>Solution</summary>

```ts
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost): void {
    if (host.getType() !== 'http') throw exception;        // rethrow for other transports

    const status = exception instanceof HttpException ? exception.getStatus() : 500;
    host.switchToHttp().getResponse<Response>().status(status).json({
      code: status === 500 ? 'INTERNAL_ERROR' : 'ERROR',
      correlationId: this.context.correlationId,
    });
  }
}
```
**Why:** without the `instanceof` check every deliberate 400 and 401 collapses into 500.
**Common wrong answer:** `status(500)` unconditionally, which turns a well-behaved API into an uninformative one.
</details>

### 57. Normalise validation errors  `medium`
`message` is a string for most exceptions and an array for validation.
<details><summary>Solution</summary>

```ts
private extractErrors(body: unknown) {
  const message = (body as { message?: unknown })?.message;
  if (!Array.isArray(message)) return undefined;
  return message.map((m) => ({ field: String(m).split(' ')[0], message: String(m) }));
}
```
**Why:** any client parsing `message` must otherwise handle both types.
**Common wrong answer:** `String(message)`, which turns an array of field errors into one unusable comma-joined string.
</details>

### 58. Map a driver error to 409  `medium`
<details><summary>Solution</summary>

```ts
@Catch(QueryFailedError)
export class QueryFailedFilter implements ExceptionFilter {
  catch(exception: QueryFailedError & { code?: string }, host: ArgumentsHost): void {
    const isUnique = exception.code === '23505';          // PostgreSQL unique_violation
    const status = isUnique ? HttpStatus.CONFLICT : HttpStatus.INTERNAL_SERVER_ERROR;
    host.switchToHttp().getResponse<Response>().status(status).json({
      code: isUnique ? 'DUPLICATE' : 'DB_ERROR',
      message: isUnique ? 'Resource already exists' : 'Database error',
    });
  }
}
```
**Why:** the driver message names the table, column, and offending value — useful in logs, unacceptable in a response.
**Common wrong answer:** returning `exception.message`, which leaks schema details and sometimes the connection string.
</details>

### 59. Keep HTTP out of a service  `medium`
The same service serves a queue consumer.
<details><summary>Solution</summary>

```ts
export abstract class DomainError extends Error { abstract readonly code: string; }
export class ResourceNotFound extends DomainError { readonly code = 'RESOURCE_NOT_FOUND'; }

@Catch(DomainError)
export class DomainErrorFilter implements ExceptionFilter {
  private static readonly MAP: Record<string, number> = {
    RESOURCE_NOT_FOUND: 404, RULE_VIOLATED: 422, PERMISSION_DENIED: 403,
  };
  catch(e: DomainError, host: ArgumentsHost): void {
    const status = DomainErrorFilter.MAP[e.code] ?? 500;
    host.switchToHttp().getResponse<Response>().status(status).json({ code: e.code, message: e.message });
  }
}
```
**Why:** a 404 is meaningless to a queue consumer, and one filter owns the mapping.
**Common wrong answer:** throwing `NotFoundException` from the service, which is fine in a pure HTTP application and wrong as soon as a second caller exists.
</details>

### 60. Emit RFC 9457 problem details  `medium`
<details><summary>Solution</summary>

```ts
res.status(status).type('application/problem+json').json({
  type: `https://api.example.com/errors/${status}`,
  title: HttpStatus[status] ?? 'Error',
  status,
  detail,
  instance: req.url,
  code,
  correlationId,
});
```
**Why:** it is a standard format API gateways and client libraries already understand.
**Common wrong answer:** inventing a bespoke shape, which every consumer must be taught individually.
</details>

### 61. Make a filter safe on every transport  `medium`
A global filter crashes a WebSocket gateway.
<details><summary>Solution</summary>

```ts
catch(exception: unknown, host: ArgumentsHost): void {
  if (host.getType() !== 'http') throw exception;    // let the transport's default handle it
  // …HTTP handling
}
```
**Why:** a WS context has no `Response`, so `.status()` on `undefined` throws *inside the error handler*, where nothing can recover.
**Common wrong answer:** wrapping the body in `try/catch`, which hides the failure without producing a usable response.
</details>

### 62. Log at the right level  `easy`
<details><summary>Solution</summary>

```ts
if (status >= 500) {
  this.logger.error(`${req.method} ${req.url} → ${status} [${correlationId}]`, stack);
} else {
  this.logger.warn(`${req.method} ${req.url} → ${status} ${code} [${correlationId}]`);
}
```
**Why:** 4xx is the caller's problem and does not need a stack; 5xx is yours and does.
**Common wrong answer:** logging everything at `error`, which makes the error rate meaningless and buries real failures under validation noise.
</details>

---

## 7. Database and Transactions

### 63. Fix an N+1  `easy`
100 tasks trigger 101 queries.
<details><summary>Solution</summary>

```ts
// ❌ one query per row
const tasks = await this.repo.find();
for (const t of tasks) console.log(t.owner.name);

// ✅ one query
const tasks = await this.repo.find({ relations: { owner: true } });
```
**Why:** the relation is loaded in the same statement instead of lazily per row.
**Common wrong answer:** `eager: true` on the relation. It fixes this call site and makes every other query pay for the join, with no way to opt out.
</details>

### 64. Make a write atomic  `easy`
An audit row survives a rollback.
<details><summary>Solution</summary>

```ts
await this.dataSource.transaction(async (manager) => {
  const order = await manager.save(Order, dto);
  await manager.save(AuditEntry, { targetId: order.id });   // same manager
});
```
**Why:** an injected repository uses its own connection and runs outside the transaction.
**Common wrong answer:** `this.auditRepo.save(...)` inside the callback. Everything works until a rollback happens, and then the data that should have vanished is still there.
</details>

### 65. Prevent an oversell  `medium`
Two concurrent purchases both see stock of 1.
<details><summary>Solution</summary>

```ts
const result = await manager
  .createQueryBuilder()
  .update(Product)
  .set({ stock: () => 'stock - :qty' })
  .where('id = :id AND stock >= :qty', { id, qty })
  .setParameter('qty', qty)
  .execute();

if (result.affected === 0) throw new ConflictException('Insufficient stock');
```
**Why:** the condition and the write are one statement, so no window exists between them.
**Common wrong answer:** reading the row, comparing, then saving. It passes every sequential test and oversells under load.
</details>

### 66. Avoid a deadlock  `medium`
Two opposite transfers deadlock intermittently.
<details><summary>Solution</summary>

```ts
const [first, second] = [fromId, toId].sort();     // global lock ordering
await lock(manager, first);
await lock(manager, second);
```
**Why:** if every transaction acquires locks in the same order, a cycle cannot form.
**Common wrong answer:** retrying on deadlock only. Retries help, and without ordering the deadlock rate grows with traffic.
</details>

### 67. Write a safe `NOT NULL` migration  `medium`
A million rows, live traffic.
<details><summary>Solution</summary>

```ts
// Migration 1 — instant, metadata only
await q.query(`ALTER TABLE "tasks" ADD "priority" integer`);
// Deploy code that writes the column, THEN:
// Migration 2 — backfill
await q.query(`UPDATE "tasks" SET "priority" = 0 WHERE "priority" IS NULL`);
// Migration 3
await q.query(`ALTER TABLE "tasks" ALTER COLUMN "priority" SET NOT NULL`);
```
**Why:** each step is individually safe and reversible, and none holds a long exclusive lock.
**Common wrong answer:** one statement with a default, which rewrites the table while holding a lock.
</details>

### 68. Build an index without blocking writes  `easy`
<details><summary>Solution</summary>

```ts
export class AddOwnerIndex1735689600000 implements MigrationInterface {
  transaction = false;                       // CONCURRENTLY cannot run in a transaction

  public async up(q: QueryRunner): Promise<void> {
    await q.query(`CREATE INDEX CONCURRENTLY "idx_tasks_owner" ON "tasks" ("ownerId", "status")`);
  }
  public async down(q: QueryRunner): Promise<void> {
    await q.query(`DROP INDEX CONCURRENTLY "idx_tasks_owner"`);
  }
}
```
**Why:** a plain `CREATE INDEX` blocks writes for the whole build.
**Common wrong answer:** forgetting `transaction = false`, which makes the migration fail outright.
</details>

### 69. Paginate deeply without degrading  `medium`
Page 5,000 takes seconds and rows shift while scrolling.
<details><summary>Solution</summary>

```ts
const rows = await this.repo
  .createQueryBuilder('t')
  .where(cursor ? '(t.createdAt, t.id) < (:date, :id)' : '1=1', { date: cursor?.date, id: cursor?.id })
  .orderBy('t.createdAt', 'DESC')
  .addOrderBy('t.id', 'DESC')        // tiebreaker makes the ordering total
  .take(limit + 1)
  .getMany();
```
**Why:** `OFFSET` scans and discards every skipped row, and inserts shift subsequent pages. The unique tiebreaker is what makes the cursor exact.
**Common wrong answer:** keyset pagination ordered by timestamp alone, where rows sharing a millisecond have undefined order and can be skipped.
</details>

### 70. Group an OR without leaking rows  `medium`
A search endpoint started returning other users' records.
<details><summary>Solution</summary>

```ts
qb.where('task.ownerId = :ownerId', { ownerId })
  .andWhere(new Brackets((w) => {
    w.where('task.title ILIKE :q', { q }).orWhere('task.description ILIKE :q', { q });
  }));
```
**Why:** without `Brackets` the SQL becomes `owner = x AND title LIKE y OR description LIKE z`, and the last clause matches regardless of owner.
**Common wrong answer:** relying on the builder to parenthesise. It does not.
</details>

### 71. Choose the right write method  `easy`
Creation issues two queries.
<details><summary>Solution</summary>

```ts
await this.repo.insert(dto);        // one INSERT — no hooks, no cascades
```
**Why:** `save` first runs a `SELECT` to decide between insert and update.
**Common wrong answer:** `save(create(dto))` everywhere by habit, which doubles the query count on every write path.
</details>

### 72. Make creation idempotent atomically  `medium`
<details><summary>Solution</summary>

```ts
await this.repo.upsert(dto, {
  conflictPaths: ['idempotencyKey'],
  skipUpdateIfNoValuesChanged: true,
});
```
**Why:** a single `INSERT … ON CONFLICT` has no race window.
**Common wrong answer:** finding by key then saving, where two concurrent replays both find nothing and both insert.
</details>

### 73. Handle an unchecked filter value  `medium`
`findOne({ where: { id: req.params.id } })` returns an arbitrary user.
<details><summary>Solution</summary>

```ts
@Get(':id')
findOne(@Param('id', ParseUUIDPipe) id: string) {   // absent or malformed → 400
  return this.users.findOne(id);
}
```
**Why:** TypeORM *drops* a condition whose value is `undefined`, so the query matches everything and returns the first row.
**Common wrong answer:** an `if (!id)` guard inside the service. It works, and every new endpoint needs the same guard; a pipe applies at the boundary.
</details>

### 74. Scope a query by owner  `easy`
A non-owner gets 403, revealing that the id exists.
<details><summary>Solution</summary>

```ts
async findOwned(id: string, ownerId: string): Promise<Task> {
  const task = await this.repo.findOne({ where: { id, ownerId } });
  if (!task) throw new NotFoundException(`Task ${id} not found`);
  return task;
}
```
**Why:** identical responses for "missing" and "not yours" remove the enumeration oracle.
**Common wrong answer:** loading then comparing the owner id, which is correct and leaks existence through the status code.
</details>

---

## 8. Authentication and Authorization

### 75. Hash a password correctly  `easy`
<details><summary>Solution</summary>

```ts
await argon2.hash(plain, {
  type: argon2.argon2id,
  memoryCost: 19_456,        // memory hardness is the point
  timeCost: 2,
  parallelism: 1,
});
```
**Why:** a general-purpose hash is brute-forceable at billions of guesses per second regardless of salting.
**Common wrong answer:** SHA-256 with a hand-managed salt column, which is a hand-rolled scheme that should be replaced.
</details>

### 76. Remove the enumeration oracle  `medium`
Login is measurably faster for unknown emails.
<details><summary>Solution</summary>

```ts
async verifyCredentials(email: string, password: string): Promise<User | null> {
  const user = await this.users.findByEmail(email);
  const hash = user?.passwordHash ?? this.dummyHash;     // always hash
  const ok = await this.passwords.verify(hash, password);
  return ok && user ? user : null;                        // one outcome
}
```
**Why:** skipping the hash makes the missing-user path faster, and the timing alone enumerates accounts.
**Common wrong answer:** unifying only the message. The timing difference remains and is trivially measurable.
</details>

### 77. Detect refresh-token theft  `medium`
<details><summary>Solution</summary>

```ts
const record = await this.tokens.findByHash(sha256(presented));
if (!record || record.revokedAt) throw new UnauthorizedException();

if (record.usedAt) {
  // A single-use token presented twice means a copy exists somewhere.
  await this.tokens.revokeFamily(record.familyId);
  throw new UnauthorizedException('Refresh token reuse detected');
}
await this.tokens.markUsed(record.id);
return this.issueTokens(user, record.familyId);
```
**Why:** rotation is the only mechanism that makes theft observable, and you cannot tell victim from attacker — so both lose the session.
**Common wrong answer:** allowing reuse within a window without revoking. An attacker then keeps a renewable session indefinitely and invisibly.
</details>

### 78. Store refresh tokens safely  `easy`
<details><summary>Solution</summary>

```ts
tokenHash: createHash('sha256').update(token).digest('hex'),
```
**Why:** SHA-256 is correct here and wrong for passwords — the input is 48 random bytes, so there is nothing to brute-force and speed costs nothing.
**Common wrong answer:** storing the raw token. A database leak then compromises every live session.
</details>

### 79. Pin the JWT algorithm  `easy`
<details><summary>Solution</summary>

```ts
super({
  jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
  secretOrKey: config.secret,
  algorithms: ['HS256'],          // never trust the token's own `alg`
  ignoreExpiration: false,
});
```
**Why:** without pinning, a library may accept `alg: none` or treat an RS256 public key as an HMAC secret.
**Common wrong answer:** omitting `algorithms` because it "works", which it does — for forged tokens too.
</details>

### 80. Reject a deleted user's valid token  `easy`
<details><summary>Solution</summary>

```ts
async validate(payload: JwtPayload): Promise<AuthenticatedUser> {
  const user = await this.users.findActiveById(payload.sub);
  if (!user) throw new UnauthorizedException();
  return { id: user.id, email: user.email, roles: user.roles };
}
```
**Why:** a valid signature proves the token was issued, not that the account still exists or is still permitted.
**Common wrong answer:** returning the payload directly, so a banned or deleted user stays authenticated until expiry.
</details>

### 81. Set a refresh cookie  `easy`
<details><summary>Solution</summary>

```ts
res.cookie('refresh_token', token, {
  httpOnly: true,            // unreadable by script — XSS cannot exfiltrate it
  secure: true,
  sameSite: 'strict',
  path: '/auth/refresh',     // never sent to any other endpoint
  maxAge: 7 * 24 * 60 * 60 * 1000,
});
```
**Why:** `httpOnly` converts an XSS from account takeover into a much smaller problem.
**Common wrong answer:** `localStorage`, where any script on the origin — including a third-party one — reads it.
</details>

### 82. Check a CASL ability correctly  `medium`
A non-owner passes the update check.
<details><summary>Solution</summary>

```ts
const task = await this.repo.findById(id);
ForbiddenError.from(ability).throwUnlessCan('update', task);   // INSTANCE
```
**Why:** conditions are evaluated only against an instance.
**Common wrong answer:** checking against the subject type string. It asks "may update *some* task", returns `true`, type-checks, and is a silent authorization bypass.
</details>

### 83. Derive a list filter from permissions  `medium`
The list endpoint returns records the detail endpoint refuses.
<details><summary>Solution</summary>

```ts
return this.repo.find({
  where: { ...accessibleBy(ability, 'read').ofType(Task), ...filter },
  take: filter.limit ?? 20,
});
```
**Why:** one rule set drives both the instance check and the query, so they cannot drift.
**Common wrong answer:** a hand-written `WHERE` clause alongside the ability rules — two definitions, and the query is the one that leaks.
</details>

### 84. Separate readable from writable  `medium`
An auditor can read everything and must change nothing.
<details><summary>Solution</summary>

```ts
// Scoped read first — a record outside the readable set is 404, not 403.
const task = (await this.repo.findAccessible(ability, { id }))[0];
if (!task) throw new NotFoundException();

// Readable ≠ writable: check the write permission separately, on the instance.
ForbiddenError.from(ability).throwUnlessCan('update', task);
```
**Why:** a single "can access" check would grant the auditor writes.
**Common wrong answer:** one read gate covering both operations.
</details>

---

## 9. Async: Queues, Events, Scheduling

### 85. Enqueue without duplicating  `easy`
A retried API call sends two welcome emails.
<details><summary>Solution</summary>

```ts
await this.queue.add('welcome', { userId: user.id }, {
  jobId: `welcome:${user.id}`,        // adding an existing id is a no-op
  attempts: 3,
  backoff: { type: 'exponential', delay: 2_000 },
  removeOnComplete: { age: 3_600, count: 1_000 },
  removeOnFail: { age: 24 * 3_600 },
});
```
**Why:** a deterministic job id makes *enqueueing* idempotent.
**Common wrong answer:** stopping there. Redelivery reuses the same job, so effects still need their own guard.
</details>

### 86. Make job effects idempotent  `medium`
<details><summary>Solution</summary>

```ts
await this.dataSource.transaction(async (manager) => {
  await manager.insert(ProcessedJob, { jobId: job.id! });    // unique constraint
  await manager.save(Charge, Charge.from(job.data));
});
```
**Why:** the marker and the work share one atomic outcome, so a redelivery hits the constraint and rolls back entirely.
**Common wrong answer:** an existence check before the transaction and a record written after it — two windows where a crash causes double processing.
</details>

### 87. Stop retrying impossible work  `easy`
<details><summary>Solution</summary>

```ts
if (!user) throw new UnrecoverableError(`User ${job.data.userId} no longer exists`);
```
**Why:** a permanently failing job otherwise burns its whole attempt budget for the same outcome.
**Common wrong answer:** `attempts: 1` globally, which also removes retries from transient failures that would have succeeded.
</details>

### 88. Keep payloads small and fresh  `easy`
A job emails a stale address.
<details><summary>Solution</summary>

```ts
await queue.add('welcome', { userId: user.id });        // id only

// processor
const user = await this.users.findOne(job.data.userId); // re-read
```
**Why:** payloads are serialised into Redis, so an entity enqueued ten minutes ago is a snapshot — and any secret in it now sits in Redis.
**Common wrong answer:** enqueueing the whole entity "to save a query".
</details>

### 89. Prevent duplicate processing of a long job  `medium`
A report is occasionally generated twice, with no error.
<details><summary>Solution</summary>

```ts
@Processor('reports', {
  concurrency: 2,
  lockDuration: 300_000,      // must exceed the longest plausible runtime
})
```
**Why:** if blocking work starves lock renewal, the lock expires and a second worker takes the job while the first still runs it.
**Common wrong answer:** adding an idempotency check only. It prevents the duplicate *effect* after both workers have done the expensive work.
</details>

### 90. Run a cron once across replicas  `medium`
<details><summary>Solution</summary>

```ts
await this.queue.add('daily-report', {}, {
  repeat: { pattern: '0 3 * * *', tz: 'UTC' },
  jobId: 'daily-report',
});
```
**Why:** the schedule lives in Redis, so exactly one worker receives each occurrence — with retries and history included.
**Common wrong answer:** an unguarded `@Cron`, which runs on every replica: three reports, three billing runs.
</details>

### 91. Write a correct distributed lock  `hard`
<details><summary>Solution</summary>

```ts
const token = randomUUID();
const acquired = await redis.set(`joblock:${name}`, token, { NX: true, PX: ttlMs });
if (!acquired) return;

const renewal = setInterval(() => void renew(name, token, ttlMs), ttlMs / 3);
try {
  await work();
} finally {
  clearInterval(renewal);
  // Compare-and-delete in Lua: never remove a lock we no longer own.
  await redis.eval(
    `if redis.call("get",KEYS[1])==ARGV[1] then return redis.call("del",KEYS[1]) else return 0 end`,
    { keys: [`joblock:${name}`], arguments: [token] },
  );
}
```
**Why:** renewal means a crashed holder releases within a third of the TTL, and the Lua script closes the window where a plain delete removes someone else's lock.
**Common wrong answer:** a long TTL with a read-then-delete release. A crash blocks the job for the full TTL, and the two-step delete can free another instance's lock.
</details>

### 92. Guard against overlap  `easy`
A five-minute job takes six.
<details><summary>Solution</summary>

```ts
private running = false;

@Cron(CronExpression.EVERY_5_MINUTES, { name: 'sync' })
async sync(): Promise<void> {
  if (this.running) return;
  this.running = true;
  try { await this.doSync(); }
  finally { this.running = false; }     // MUST be finally
}
```
**Why:** without `finally`, one throw leaves the flag `true` and every subsequent run is skipped forever.
**Common wrong answer:** clearing the flag after the work, outside a `finally`.
</details>

### 93. Publish an event that cannot be lost  `medium`
<details><summary>Solution</summary>

```ts
await this.dataSource.transaction(async (manager) => {
  const task = await manager.save(Task, Task.create(cmd));
  await manager.insert(OutboxEvent, {
    id: randomUUID(),
    type: 'task.created',
    payload: { taskId: task.id },
  });
});
// A relay publishes committed rows and marks them sent.
```
**Why:** the state change and the event commit together — no window where one exists without the other.
**Common wrong answer:** publishing on the in-memory event bus after the transaction. A crash loses it with no record it was ever published.
</details>

### 94. Relay an outbox from several replicas  `medium`
<details><summary>Solution</summary>

```sql
SELECT * FROM outbox_events WHERE published_at IS NULL
ORDER BY created_at LIMIT 100 FOR UPDATE SKIP LOCKED
```
**Why:** each instance claims a disjoint batch without blocking on rows another is publishing.
**Common wrong answer:** a plain `FOR UPDATE`, where instances serialise behind each other — or no locking, where every instance publishes every row.
</details>

---

## 10. Testing

### 95. Test a service without the container  `easy`
<details><summary>Solution</summary>

```ts
const repo = new InMemoryTaskRepository();
const service = new TasksService(repo, new NoopNotifications(), { maxActiveTasksPerUser: 3 });
```
**Why:** constructor injection means the class is testable without the framework — no async setup, no module resolution.
**Common wrong answer:** building a testing module for every unit test, which adds setup time and no coverage.
</details>

### 96. Prove no write happened  `medium`
<details><summary>Solution</summary>

```ts
const before = repo.saveCount;
await service.create({ title: 'x', idempotencyKey: 'k1' }, 'u1');
expect(repo.saveCount).toBe(before);      // a fake can expose this; a stub cannot
```
**Why:** a counter on the fake asserts an outcome rather than a call.
**Common wrong answer:** asserting the mock was not called, which passes even if the service wrote through a different method.
</details>

### 97. Test a guard with no HTTP  `easy`
<details><summary>Solution</summary>

```ts
const ctx = {
  switchToHttp: () => ({ getRequest: () => ({ user: { roles: ['member'] } }) }),
  getHandler: () => () => undefined,
  getClass: () => class {},
} as unknown as ExecutionContext;

const guard = new RolesGuard({ getAllAndOverride: () => ['admin'] } as unknown as Reflector);
expect(() => guard.canActivate(ctx)).toThrow(ForbiddenException);
```
**Why:** implementing only the subset of the interface the guard uses keeps the test fast and readable.
**Common wrong answer:** driving it over HTTP through a full application, which is orders of magnitude slower for the same assertion.
</details>

### 98. Catch a fail-open validation bug  `medium`
<details><summary>Solution</summary>

```ts
const pipe = new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true });
const meta: ArgumentMetadata = { type: 'body', metatype: CreateOrderDto, data: '' };

await expect(pipe.transform({ items: [{ quantity: -1 }] }, meta))
  .rejects.toThrow(BadRequestException);
```
**Why:** the failure mode is *accepting* bad input, so only a rejection test detects a missing `@Type()`.
**Common wrong answer:** an end-to-end test posting a valid order and asserting 201, which passes identically whether or not the decorator is present.
</details>

### 99. Fix a test that always passes  `easy`
<details><summary>Solution</summary>

```ts
await expect(service.create(bad)).rejects.toThrow();   // note the await
```
**Why:** without `await` the assertion is never awaited and the test finishes first.
**Common wrong answer:** trusting a green suite. Nothing reports this, and coverage still counts the line as executed.
</details>

### 100. Override a global guard in a test  `easy`
<details><summary>Solution</summary>

```ts
const module = await Test.createTestingModule({ imports: [TasksModule] })
  .overrideGuard(JwtAuthGuard).useValue({ canActivate: () => true })
  .compile();
```
**Why:** an `APP_GUARD` provider is part of the module graph, so testing modules get it too.
**Common wrong answer:** doing this in *every* end-to-end test, which means no test ever proves a route is protected.
</details>

### 101. Set up and tear down an e2e suite  `medium`
<details><summary>Solution</summary>

```ts
beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16-alpine').start();
  process.env.DATABASE_URL = container.getConnectionUri();
  const ref = await Test.createTestingModule({ imports: [AppModule] }).compile();
  app = ref.createNestApplication();
  app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));  // main.ts never runs
  await app.init();
  await app.get(DataSource).runMigrations();
}, 120_000);

afterAll(async () => {
  await app?.close();        // drain the pool BEFORE the container disappears
  await container?.stop();
});
```
**Why:** closing the app first avoids connection errors during shutdown; omitting `close()` hangs the runner.
**Common wrong answer:** stopping the container first, or omitting `close()` and letting CI burn its timeout on a suite that passed.
</details>

### 102. Truncate between tests safely  `easy`
<details><summary>Solution</summary>

```ts
const tables = dataSource.entityMetadatas
  .filter((e) => e.tableName !== 'migrations')      // never wipe migration history
  .map((e) => `"${e.tableName}"`)
  .join(', ');
await dataSource.query(`TRUNCATE ${tables} RESTART IDENTITY CASCADE`);
```
**Why:** wiping the migrations table makes the framework re-apply migrations against an existing schema on the next run.
**Common wrong answer:** truncating everything, or omitting `RESTART IDENTITY` so sequence values drift and ids become unpredictable.
</details>

### 103. Test idempotency under concurrency  `medium`
<details><summary>Solution</summary>

```ts
const body = { title: 'Only once', idempotencyKey: randomUUID() };
const [a, b] = await Promise.all([
  http().post('/tasks').set('Authorization', auth).send(body),
  http().post('/tasks').set('Authorization', auth).send(body),
]);

expect(a.body.id).toBe(b.body.id);
expect(await repo.countBy({ ownerId })).toBe(1);
```
**Why:** a check-then-insert implementation passes when the requests run sequentially and fails here — which is the point.
**Common wrong answer:** awaiting the two requests in sequence, which tests nothing about concurrency.
</details>

### 104. Prove a detached side effect stays detached  `medium`
<details><summary>Solution</summary>

```ts
notifications.shouldFail = true;                     // a fake with a failure switch

const task = await service.create({ title: 'x' }, 'user-1');

await expect(repo.findById(task.id)).resolves.not.toBeNull();
expect(notifications.sent).toHaveLength(0);
```
**Why:** this is the only test that pins down a deliberately un-awaited side effect versus an awaited one.
**Common wrong answer:** a plain mock notifier, which resolves by default — so the suite stays green if someone adds `await`, silently turning a mail outage into a request outage.
</details>

---

## 11. Output Prediction and Debugging

### 105. What is logged?  `easy`
```ts
@Injectable() class A { constructor() { console.log('A built'); } }
// A is in providers; nothing injects it. App starts; no requests are sent.
```
<details><summary>Solution</summary>

`A built`, once, at startup.
**Why:** default-scoped providers are instantiated eagerly during `NestFactory.create()`, whether or not anything has injected them.
**Common wrong answer:** "nothing, because it is unused" — that would be lazy resolution, which Nest does not do for default scope.
</details>

### 106. How many instances?  `easy`
```ts
@Injectable({ scope: Scope.TRANSIENT }) class L {}
@Injectable() class A { constructor(private l: L) {} }
@Injectable() class B { constructor(private l: L) {} }
```
<details><summary>Solution</summary>

Two `L` instances; `A` and `B` remain singletons.
**Why:** transient means one per *consumer*, and it does not bubble.
**Common wrong answer:** assuming `A` and `B` become transient too — that is `REQUEST` scope's behaviour, not `TRANSIENT`'s.
</details>

### 107. What does this return?  `medium`
```ts
const cached = await this.cache.get<number>('hits');   // stored value is 0
if (cached) return cached;
return this.compute();
```
<details><summary>Solution</summary>

`this.compute()` — every time, forever.
**Why:** `0` is falsy, so a legitimate cached value is treated as a miss and re-cached as `0`.
**Common wrong answer:** assuming the cache is broken. Use `cached !== undefined`.
</details>

### 108. Which handler runs?  `easy`
```ts
@Controller('users')
export class UsersController {
  @Get(':id') findOne(@Param('id') id: string) { return { id }; }
  @Get('me')  me() { return { me: true }; }
}
```
<details><summary>Solution</summary>

`GET /users/me` hits `findOne` with `id = 'me'`.
**Why:** registration order follows declaration order, and no warning is emitted.
**Common wrong answer:** expecting the more specific route to win. Nest does not sort by specificity.
</details>

### 109. What persists?  `medium`
```ts
await this.dataSource.transaction(async (manager) => {
  await manager.save(Order, order);
  await this.auditRepo.save(audit);
  throw new Error('boom');
});
```
<details><summary>Solution</summary>

The audit row persists; the order does not.
**Why:** the injected repository runs on its own connection, outside the transaction.
**Common wrong answer:** "nothing persists" — which is exactly what makes this bug hard to spot in review.
</details>

### 110. What is the status?  `medium`
```ts
// RolesGuard is registered BEFORE JwtAuthGuard
canActivate(ctx) {
  const { user } = ctx.switchToHttp().getRequest();
  return user.roles.includes('admin');
}
```
<details><summary>Solution</summary>

500 — `user` is `undefined`, so reading `.roles` throws a `TypeError`, which is not an `HttpException`.
**Why:** a crash is worse than a denial: 500s are retried by clients and sometimes routed past authorization upstream.
**Common wrong answer:** 403. The guard never gets far enough to return `false`.
</details>

### 111. What does the client receive?  `easy`
```ts
use(req: Request, res: Response, next: NextFunction) {
  console.log('middleware ran');
}
```
<details><summary>Solution</summary>

Nothing — the log prints and the request hangs until the client times out.
**Why:** middleware must either send a response or call `next()`. Doing neither is not an error.
**Common wrong answer:** expecting a 500. Nothing throws, so nothing is reported anywhere.
</details>

### 112. Why is the dashboard green during an outage?  `medium`
```ts
return next.handle().pipe(tap(() => this.metrics.record(Date.now() - start)));
```
<details><summary>Solution</summary>

`tap` with a single function subscribes only to the next notification, so only successful requests are recorded — latency percentiles are computed from requests that did not fail, and errors are absent entirely.
**Why:** the failure mode is dangerous precisely because the dashboard looks healthy while users are not being served.
**Common wrong answer:** blaming the metrics backend. Use `tap({ next, error })` or `finalize`.
</details>

---

**Next:** [Appendix C — 20 Real-World Projects →](C-projects.md)
