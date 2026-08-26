# Chapter 8 — Custom Providers

## Learning Objectives

- Choose between `useValue`, `useClass`, `useFactory`, and `useExisting` from the requirement
- Inject dependencies into a factory provider, and explain why `inject` order matters
- Write an async provider without hanging application startup
- Register global guards, pipes, filters, and interceptors as providers so they can inject
- Recognise when a custom provider is the wrong tool

---

## 8.1 The Four Recipes

Chapter 7 established that `providers: [TasksService]` expands to `{ provide: TasksService, useClass: TasksService }`. Once the token and the implementation are separate, four recipes become available:

| Recipe | Supplies | Reach for it when |
|---|---|---|
| `useValue` | an existing object | constants, config, third-party clients, test doubles |
| `useClass` | a class for Nest to construct | swapping implementations behind one token |
| `useFactory` | the return value of a function | construction needs logic or other providers |
| `useExisting` | an alias to another token | renaming or deprecating a token |

Every recipe registers under a `provide` token, and every one produces a singleton by default.

```ts
@Module({
  providers: [
    { provide: APP_CONFIG,     useValue: { retries: 3 } },
    { provide: PaymentGateway, useClass: StripeGateway },
    { provide: 'CLOCK',        useFactory: () => new Clock() },
    { provide: 'LEGACY_MAIL',  useExisting: MailService },
  ],
})
export class AppModule {}
```

---

## 8.2 `useValue`

The simplest recipe: hand the container an object you already have.

```ts
export const APP_CONFIG = Symbol('APP_CONFIG');

providers: [
  { provide: APP_CONFIG, useValue: { retries: 3, timeoutMs: 5_000 } },
]
```

Its three real uses:

**Configuration constants**, as above.

**Third-party clients** that Nest cannot construct because they are not classes with injectable constructors:

```ts
import Stripe from 'stripe';

{
  provide: 'STRIPE',
  useValue: new Stripe(process.env.STRIPE_KEY!, { apiVersion: '2024-06-20' }),
}
```

**Test doubles**, which is where `useValue` earns its keep:

```ts
const module = await Test.createTestingModule({
  providers: [
    TasksService,
    { provide: TaskRepository, useValue: { findById: jest.fn(), save: jest.fn() } },
  ],
}).compile();
```

Non-class tokens cannot be injected by type annotation — there is no class for the metadata to record — so they always need `@Inject()`:

```ts
constructor(@Inject(APP_CONFIG) private readonly config: AppConfig) {}
```

> ⚠️ **Gotcha** — `useValue` is evaluated **when the module is defined**, not when the container resolves it. `new Stripe(process.env.STRIPE_KEY!)` at module scope reads the environment at import time, which is before `ConfigModule` has loaded a `.env` file. The value silently becomes `undefined`. Anything depending on configuration belongs in `useFactory`, not `useValue`.

---

## 8.3 `useClass`

`useClass` tells Nest which class to construct for a token. Its point is that the two need not match:

```ts
export abstract class PaymentGateway {
  abstract charge(amountCents: number): Promise<Receipt>;
}

providers: [{ provide: PaymentGateway, useClass: StripeGateway }]
```

Consumers depend on `PaymentGateway`; the module decides the implementation. Swapping to `PaypalGateway` touches one line and no consumer.

Because a module is code, the class can be chosen conditionally:

```ts
providers: [
  {
    provide: PaymentGateway,
    useClass: process.env.NODE_ENV === 'test' ? FakeGateway : StripeGateway,
  },
]
```

That works and it is worth being uneasy about it: the condition is evaluated at import time, is invisible to anyone reading the consumer, and is exactly how a test double reaches production when an environment variable is misspelled. Prefer configuring it through a factory with an explicit guard (§8.4), and if you do branch on `NODE_ENV`, make the unsafe branch throw rather than fall through.

Unlike `useValue`, the class is **constructed by the container**, so its own dependencies are injected and its lifecycle hooks fire.

---

## 8.4 `useFactory`

`useFactory` runs a function and registers its return value. It is the recipe that can depend on other providers:

```ts
{
  provide: 'DATABASE_CONNECTION',
  inject: [ConfigService],                       // resolved first, in this order
  useFactory: (config: ConfigService) => createPool({
    host: config.getOrThrow('DB_HOST'),
    max: config.get('DB_POOL_SIZE') ?? 10,
  }),
}
```

**`inject` is positional.** The array's order is the factory's parameter order — there is no name matching, so swapping two entries silently passes the wrong object:

```ts
// ❌ Arguments are transposed. Both are objects, so it may not throw for a while.
inject: [ConfigService, Logger],
useFactory: (logger: Logger, config: ConfigService) => …

// ✅
inject: [ConfigService, Logger],
useFactory: (config: ConfigService, logger: Logger) => …
```

Use a factory whenever construction requires a decision, a dependency, or anything beyond a literal:

```ts
{
  provide: CacheStore,
  inject: [ConfigService],
  useFactory: (config: ConfigService): CacheStore => {
    const url = config.get('REDIS_URL');
    // Falling back is a deliberate choice for local development; in production
    // a missing Redis URL should be a startup failure, not a silent downgrade.
    if (!url) {
      if (config.get('NODE_ENV') === 'production') {
        throw new Error('REDIS_URL is required in production');
      }
      return new InMemoryCacheStore();
    }
    return new RedisCacheStore(url);
  },
}
```

To inject an optional dependency into a factory, wrap the token:

```ts
inject: [ConfigService, { token: METRICS, optional: true }],
useFactory: (config: ConfigService, metrics?: MetricsClient) => …
```

---

## 8.5 Async Providers

A factory may return a promise. Nest awaits it before continuing bootstrap, so the provider is fully ready before anything can inject it:

```ts
{
  provide: 'DATABASE_CONNECTION',
  inject: [ConfigService],
  useFactory: async (config: ConfigService) => {
    const connection = await createConnection(config.getOrThrow('DB_URL'));
    await connection.ping();                 // fail fast, at boot, not on request one
    return connection;
  },
}
```

This is the right place to verify a connection. A database that is unreachable should stop the deploy, not surface as a 500 on the first request that needs it.

The cost is that **bootstrap blocks on it**:

> ⚠️ **Gotcha** — An async factory with no timeout hangs startup indefinitely if the dependency is unreachable. There is no error and no log line — the process simply never listens, and health checks report nothing because the server was never started. Always bound it.

```ts
useFactory: async (config: ConfigService) => {
  const connect = createConnection(config.getOrThrow('DB_URL'));
  const timeout = new Promise<never>((_, reject) =>
    setTimeout(() => reject(new Error('DB connection timed out after 10s')), 10_000),
  );
  return Promise.race([connect, timeout]);    // fails loudly instead of hanging
}
```

Providers whose factories do not depend on each other are resolved concurrently, so several async providers do not necessarily serialise — but the slowest one still gates `listen()`.

---

## 8.6 Providers That Configure the Framework

Four special tokens let you register global pipeline components as providers. This matters because `app.useGlobalGuards()` in `main.ts` **cannot inject anything**:

```ts
// ❌ RolesGuard needs Reflector and ConfigService — neither is available here
app.useGlobalGuards(new RolesGuard(/* ??? */));

// ✅ Registered as a provider: constructed by the container, dependencies injected
import { APP_GUARD, APP_PIPE, APP_FILTER, APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  providers: [
    { provide: APP_GUARD,       useClass: RolesGuard },
    { provide: APP_PIPE,        useClass: ValidationPipe },
    { provide: APP_FILTER,      useClass: AllExceptionsFilter },
    { provide: APP_INTERCEPTOR, useClass: LoggingInterceptor },
  ],
})
export class AppModule {}
```

These tokens are **multi-provider**: registering `APP_GUARD` several times adds guards rather than replacing them, and they run in registration order.

The other advantage is testability — a testing module builds the same graph, so global components are present in tests exactly as in production, which is not true of anything configured in `main.ts` (→ [Ch 29](29-unit-testing.md)).

---

## 8.7 Common Mistakes and Best Practices

**Mistake: `useValue` for anything derived from configuration.** It is evaluated at import time, before `ConfigModule` has loaded anything, so the value is silently `undefined`.

**Mistake: transposing the `inject` array.** It is positional; mismatched order passes the wrong argument and often fails far from the cause.

**Mistake: an async factory with no timeout.** An unreachable dependency hangs bootstrap with no error and no listening port.

**Mistake: `useClass` where `useExisting` was meant.** You get two instances instead of an alias, and any state inside them diverges (→ [Ch 7](07-dependency-injection.md)).

**Mistake: injecting a non-class token by type.** String and symbol tokens have no metadata; they always need `@Inject()`.

**Mistake: `app.useGlobalGuards(new SomeGuard())` for a guard with dependencies.** It cannot inject, so you end up hand-constructing its collaborators.

**Mistake: branching on `NODE_ENV` inside `useClass`.** The decision is invisible at the consumer and a typo in the variable ships a test double.

**Best practice: use a factory whenever construction needs a decision.** It keeps the logic in one readable place with its dependencies declared.

**Best practice: validate in the factory and throw.** A missing production credential should fail the deploy, not degrade silently at runtime.

**Best practice: register global pipeline components with the `APP_*` tokens.** They gain injection and appear in tests.

---

## 8.8 Real-World Example: A Redis Provider That Fails Correctly

A cache client that is configured from the environment, verified at startup, bounded so it cannot hang a deploy, and closed cleanly on shutdown.

```ts
// src/cache/cache.tokens.ts
export const REDIS_CLIENT = Symbol('REDIS_CLIENT');
export const CACHE_OPTIONS = Symbol('CACHE_OPTIONS');

export interface CacheOptions {
  url?: string;
  connectTimeoutMs: number;
  keyPrefix: string;
}
```

```ts
// src/cache/redis.provider.ts
import { Provider, Logger } from '@nestjs/common';
import { createClient, RedisClientType } from 'redis';
import { ConfigService } from '@nestjs/config';
import { REDIS_CLIENT, CACHE_OPTIONS, CacheOptions } from './cache.tokens';

/** Options are their own provider so tests can override them without touching the client. */
export const cacheOptionsProvider: Provider = {
  provide: CACHE_OPTIONS,
  inject: [ConfigService],
  useFactory: (config: ConfigService): CacheOptions => ({
    url: config.get<string>('REDIS_URL'),
    connectTimeoutMs: config.get<number>('REDIS_TIMEOUT_MS') ?? 10_000,
    keyPrefix: config.get<string>('CACHE_PREFIX') ?? 'app:',
  }),
};

export const redisProvider: Provider = {
  provide: REDIS_CLIENT,
  // Positional: CACHE_OPTIONS first, ConfigService second — must match the signature.
  inject: [CACHE_OPTIONS, ConfigService],
  useFactory: async (
    options: CacheOptions,
    config: ConfigService,
  ): Promise<RedisClientType | null> => {
    const logger = new Logger('RedisProvider');
    const isProduction = config.get('NODE_ENV') === 'production';

    if (!options.url) {
      // Degrading silently is acceptable locally and dangerous in production,
      // where a missing cache usually means a misconfigured deploy.
      if (isProduction) throw new Error('REDIS_URL is required in production');
      logger.warn('REDIS_URL not set — cache disabled for this process');
      return null;
    }

    const client: RedisClientType = createClient({ url: options.url });
    client.on('error', (err) => logger.error('Redis client error', err));

    // Bounded: an unreachable Redis must fail the boot, never hang it silently.
    const timeout = new Promise<never>((_, reject) =>
      setTimeout(
        () => reject(new Error(`Redis connect timed out after ${options.connectTimeoutMs}ms`)),
        options.connectTimeoutMs,
      ),
    );

    await Promise.race([client.connect(), timeout]);
    await client.ping();                       // prove it actually answers
    logger.log('Redis connected');
    return client;
  },
};
```

```ts
// src/cache/cache.module.ts
@Module({
  imports: [ConfigModule],
  providers: [cacheOptionsProvider, redisProvider, CacheService],
  exports: [CacheService],                     // the client stays private
})
export class CacheModule implements OnApplicationShutdown {
  constructor(@Inject(REDIS_CLIENT) private readonly client: RedisClientType | null) {}

  async onApplicationShutdown(): Promise<void> {
    await this.client?.quit();                 // drain rather than drop the socket
  }
}
```

```ts
// src/cache/cache.service.ts
@Injectable()
export class CacheService {
  constructor(
    @Inject(REDIS_CLIENT) private readonly client: RedisClientType | null,
    @Inject(CACHE_OPTIONS) private readonly options: CacheOptions,
  ) {}

  /** A null client means caching is disabled, so every read is a miss. */
  async get<T>(key: string): Promise<T | null> {
    if (!this.client) return null;
    const raw = await this.client.get(this.options.keyPrefix + key);
    return raw ? (JSON.parse(raw) as T) : null;
  }

  async set(key: string, value: unknown, ttlSeconds: number): Promise<void> {
    if (!this.client) return;
    await this.client.set(this.options.keyPrefix + key, JSON.stringify(value), {
      EX: ttlSeconds,
    });
  }
}
```

The decision worth defending is returning `null` for a missing URL rather than a no-op client object implementing the same interface. A null-object implementation would remove the `if (!this.client)` checks and read more cleanly, and it would also make a disabled cache indistinguishable from a working one at every call site — which is precisely the thing you want to be able to see. Making absence explicit in the type (`RedisClientType | null`) forces every consumer to acknowledge the case at compile time, and the two branches are one line each. The related decision is throwing in production while warning locally: the same missing variable means "I haven't started Redis" on a laptop and "this deploy is broken" in production, and a provider that treats those identically will always pick the wrong one.

---

## 8.9 Questions

**1.** MCQ — Which recipe can depend on other providers? a) `useValue` b) `useClass` c) `useFactory` d) both b and c
<details><summary>Answer</summary>**d)** `useClass` gets constructor injection; `useFactory` gets the `inject` array. `useValue` receives an already-built object.</details>

**2.** MCQ — The `inject` array is matched to factory parameters by: a) name b) type c) position d) token order in the module
<details><summary>Answer</summary>**c)** Position — transposing two entries silently passes the wrong argument.</details>

**3.** MCQ — An async factory that never resolves causes: a) a startup error b) a hang with no listening port c) a 500 on first request d) a retry
<details><summary>Answer</summary>**b)** Bootstrap blocks silently. Always bound it with a timeout.</details>

**4.** MCQ — `APP_GUARD` registered three times results in: a) the last one winning b) an error c) all three running d) the first one winning
<details><summary>Answer</summary>**c)** The `APP_*` tokens are multi-provider; all run, in registration order.</details>

**5.** MCQ — A symbol-token provider is injected with: a) the type annotation b) `@Inject(TOKEN)` c) `@Optional()` d) `useExisting`
<details><summary>Answer</summary>**b)** Non-class tokens have no metadata, so the token must be explicit.</details>

**6.** MCQ — `useValue: new Client(process.env.KEY)` at module scope is evaluated: a) at resolution b) at import time c) per request d) lazily
<details><summary>Answer</summary>**b)** At import time — before `ConfigModule` has loaded a `.env` file.</details>

**7.** Short — Why register a global pipe with `APP_PIPE` rather than `useGlobalPipes`?
<details><summary>Answer</summary>`APP_PIPE` makes it a provider, so the container constructs it with its dependencies injected and it appears in testing modules; `useGlobalPipes` in `main.ts` cannot inject and is absent from tests.</details>

**8.** Short — When is `useValue` the wrong choice for a third-party client?
<details><summary>Answer</summary>Whenever its construction reads configuration, because the value is built at import time, before configuration is loaded. Use `useFactory` with `ConfigService` injected.</details>

**9.** Short — What does an async provider buy you over connecting lazily?
<details><summary>Answer</summary>Failure at boot rather than on the first request that needs the dependency, so a broken deploy never starts serving.</details>

**10.** Short — Why is branching on `NODE_ENV` inside `useClass` risky?
<details><summary>Answer</summary>The decision is evaluated at import time and is invisible from the consumer, so a misspelled or unset variable can ship a test double to production with nothing failing.</details>

**11.** Short — How do you make one factory dependency optional?
<details><summary>Answer</summary>Use the object form in `inject`: `{ token: METRICS, optional: true }`, and type the corresponding parameter as optional.</details>

**12.** Output — What does the factory receive?
```ts
inject: [ConfigService, Logger],
useFactory: (logger: Logger, config: ConfigService) => {
  return logger.constructor.name;
}
```
<details><summary>Answer</summary>`'ConfigService'` — `inject` is positional, so the first parameter (named `logger`) is actually the `ConfigService` instance. The names are misleading, not binding.</details>

**13.** Output — Two providers use `useClass: LoggerService` under different tokens. How many instances?
<details><summary>Answer</summary>Two. `useClass` constructs per token; `useExisting` is what shares one instance.</details>

**14.** Output — `{ provide: 'X', useValue: undefined }` is registered and injected. Does resolution fail?
<details><summary>Answer</summary>No. The token exists and resolves to `undefined`, so the failure appears later at first use — a reason to validate inside a factory instead.</details>

**15.** Output — An async factory takes 8 seconds. When does `app.listen()` complete?
<details><summary>Answer</summary>After at least 8 seconds. Bootstrap awaits async providers before the server listens, which is why they need timeouts.</details>

**16.** Debug — A Stripe client provider works locally but has an undefined API key in production, though the variable is set. Cause?
<details><summary>Answer</summary>It is a `useValue` reading `process.env` at import time, before configuration loading. Convert it to a `useFactory` with `ConfigService` injected.</details>

**17.** Debug — A deploy hangs with no logs after "Starting Nest application". What do you suspect?
<details><summary>Answer</summary>An async provider awaiting an unreachable dependency. The process never reaches `listen()`, so nothing logs and no port opens. Add a timeout to the factory.</details>

**18.** Debug — A global guard works when registered in `main.ts` but its injected `Reflector` is undefined. Explain.
<details><summary>Answer</summary>It was hand-constructed with `new`, so nothing was injected. Register it with `APP_GUARD` so the container builds it.</details>

**19.** Debug — After adding a second `APP_INTERCEPTOR`, the first one's behaviour changed. Why?
<details><summary>Answer</summary>Both now run, in registration order, and interceptors nest — so the second wraps inside the first and the response passes through both on the way out.</details>

**20.** Debug — A factory's dependency resolves to the wrong object, but only sometimes. What is the first thing to check?
<details><summary>Answer</summary>The `inject` array's order against the factory's parameter list. Positional mismatches between two objects of similar shape can go unnoticed until a specific field is touched.</details>

**21.** Code — Provide a database pool whose size comes from configuration, verified at startup and bounded to 5 seconds.
<details><summary>Answer</summary>

```ts
{
  provide: DB_POOL,
  inject: [ConfigService],
  useFactory: async (config: ConfigService) => {
    const pool = createPool({
      url: config.getOrThrow('DB_URL'),
      max: config.get<number>('DB_POOL_SIZE') ?? 10,
    });
    const timeout = new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error('DB connect timed out')), 5_000),
    );
    await Promise.race([pool.query('SELECT 1'), timeout]);
    return pool;
  },
}
```
</details>

**22.** Code — Register `ValidationPipe` and a `RolesGuard` that needs `Reflector`, both globally and both injectable.
<details><summary>Answer</summary>

```ts
@Module({
  providers: [
    {
      provide: APP_PIPE,
      useValue: new ValidationPipe({ whitelist: true, transform: true }),
    },
    { provide: APP_GUARD, useClass: RolesGuard },   // useClass so Reflector is injected
  ],
})
export class AppModule {}
```
`ValidationPipe` takes no injected dependencies, so `useValue` is fine; `RolesGuard` needs the container, so it must be `useClass`.
</details>

**23.** Code — Provide a `Clock` that returns a fixed time in tests and the real time otherwise, without branching on `NODE_ENV`.
<details><summary>Answer</summary>

```ts
export abstract class Clock { abstract now(): Date; }

@Injectable() export class SystemClock extends Clock { now() { return new Date(); } }
export class FixedClock extends Clock {
  constructor(private readonly at: Date) { super(); }
  now() { return this.at; }
}

// production module
providers: [{ provide: Clock, useClass: SystemClock }]

// test module — the override is explicit at the call site, not hidden in a branch
Test.createTestingModule({ providers: [Svc] })
  .overrideProvider(Clock)
  .useValue(new FixedClock(new Date('2026-01-01')))
  .compile();
```
</details>

**24.** Scenario — A teammate wants one `useFactory` that builds three related clients and returns them as `{ a, b, c }`, to avoid repetition. Evaluate it.
<details><summary>Answer</summary>It works and it costs you granularity. One token means consumers inject the whole bundle and depend on all three even when they use one, tests must fake all three to exercise any, and a failure in the third client prevents the first two from being available at all. Registering three providers that each depend on a shared options provider keeps the repetition small and the failure domains separate. The bundle is defensible only when the three genuinely cannot exist independently — a driver and its two views over the same connection — in which case the shared lifetime is the point rather than an accident.</details>

**25.** Scenario — Your application needs a feature flag that changes which implementation a token resolves to, and it can change while the app is running. Can a custom provider do this?
<details><summary>Answer</summary>Not directly — providers are resolved once at startup, so a `useFactory` reading a flag captures the value at boot and never re-reads it. The workable pattern is to keep one provider that owns the decision at call time: inject both implementations into a small router class registered under the contract token, and have its methods consult the flag service per call. That keeps consumers depending on one stable token while the branch happens where it can actually observe change. Request-scoped providers (→ Ch 9) can re-evaluate per request, but promoting the whole dependency chain to request scope to read a boolean is a large cost for a small need, and restarting on flag change is often the honest answer for infrastructure-level switches.</details>

---

## Chapter Summary

- Four recipes cover every case: `useValue` for existing objects, `useClass` for container-constructed implementations, `useFactory` when construction needs logic or dependencies, `useExisting` for aliases.
- `useValue` evaluates at import time, which makes it the wrong recipe for anything derived from configuration — the value is built before config is loaded.
- `inject` is positional, so its order must match the factory signature exactly; a transposition passes the wrong object and fails far from the cause.
- Async factories let a broken dependency fail the boot instead of the first request, but an unbounded one hangs startup with no error and no open port.
- The `APP_GUARD`/`APP_PIPE`/`APP_FILTER`/`APP_INTERCEPTOR` tokens are multi-provider and give global components injection and test visibility, which `main.ts` registration cannot.
- Validate inside factories and throw: a missing production credential should stop a deploy rather than silently degrade.

---

## Practice Exercises

1. Register the same client with `useValue` reading `process.env` and with `useFactory` reading `ConfigService`. Load the variable from a `.env` file and observe which one is `undefined`.
2. Transpose a two-entry `inject` array and find how far from the provider the resulting failure appears.
3. Write an async provider against an unreachable host, confirm the silent hang, then add a timeout.
4. Convert a `useGlobalGuards` registration to `APP_GUARD` and give the guard an injected dependency.
5. Register three `APP_INTERCEPTOR`s that log on the way in and out, and record the nesting order.

---

## Mini Project: A Configurable Storage Provider

Build a storage module that resolves to a different backend per environment.

**Requirements:**

- One `StorageService` contract with S3, local-filesystem, and in-memory implementations.
- Selection by configuration through a factory — never by branching on `NODE_ENV` at import time.
- The production path validates its credentials in the factory and throws if any are missing.
- Connection verification is bounded by a timeout so an unreachable bucket fails the boot.
- The client is closed in `onApplicationShutdown`, and a test proves the in-memory backend is selected without any environment variables set.

---

**Next:** [Chapter 9 — Provider Scopes and Lifecycle →](09-scopes-lifecycle.md)
