# Chapter 4 — Modules

## Learning Objectives

- Explain what each of a module's four arrays controls, and which one defines the public surface
- Predict whether a given provider is visible from a given module, and why resolution fails when it is not
- Decide between a shared module, a global module, and a re-export
- Build a dynamic module with `forRoot`/`forFeature`, including an async configuration variant
- Resolve a circular module dependency, and recognise when it signals a design problem instead

---

## 4.1 What a Module Is

A module is a class with a decorator and, usually, an empty body. That body stays empty — a module is configuration, not behaviour.

```ts
// src/tasks/tasks.module.ts
@Module({
  imports: [UsersModule],          // what this module may use from elsewhere
  controllers: [TasksController],  // instantiated, routes registered
  providers: [TasksService],       // instantiated, injectable inside this module
  exports: [TasksService],         // ...and injectable by whoever imports this module
})
export class TasksModule {}
```

Modules do two jobs. They **group** related code, which is organisational and could be achieved with folders. And they **encapsulate** it, which cannot — `providers` and `exports` are a genuine visibility boundary enforced at startup.

That second job is the one that matters. In an Express codebase, any file can `require` any other file; the only thing preventing `orders/handler.js` from reaching into `users/internals.js` is that nobody has needed to yet. A Nest module makes that reach a startup error.

Every application has at least one module, `AppModule`, and Nest builds a graph from it by following `imports`. A module not reachable from that graph does not exist as far as the framework is concerned — its controllers register no routes and its providers are never constructed.

---

## 4.2 The Four Arrays

| Array | Contains | Effect |
|---|---|---|
| `imports` | other modules | this module can inject whatever they **export** |
| `controllers` | controller classes | instantiated; their route metadata is registered |
| `providers` | injectable classes/tokens | instantiated; injectable **within this module only** |
| `exports` | providers or modules | makes them injectable by importing modules |

Two rules cover almost every resolution error you will hit.

**A provider must be listed in `providers` somewhere.** Being `@Injectable()` is not registration. An `@Injectable()` class nobody registered is an ordinary class.

**A provider is only visible where it is exported to.** If `TasksService` is in `TasksModule`'s `providers` but not its `exports`, then a controller in `ReportsModule` cannot inject it — even with `imports: [TasksModule]`.

```ts
// ❌ Fails at startup: "Nest can't resolve dependencies of ReportsService (?)"
@Module({ providers: [TasksService] })            // not exported
export class TasksModule {}

@Module({ imports: [TasksModule], providers: [ReportsService] })
export class ReportsModule {}
// ReportsService constructor asks for TasksService → no visible token
```

The fix is `exports: [TasksService]`. The instinct to reach for `@Global()` instead is what §4.4 is about, and it is usually wrong.

> ⚠️ **Gotcha** — Listing a provider in two modules' `providers` arrays does not share it. It creates **two independent instances**, each with its own state. Caches diverge, counters disagree, and connection pools double. To share one instance, register it once and export it.

> 💼 **Interview** — "How do you share a service between two modules?" The answer is: export it from the module that owns it and import that module. If the candidate says "add it to both modules' providers", they have just described a subtle duplicate-instance bug.

---

## 4.3 Modules Are Singletons

A module is instantiated once per application, no matter how many modules import it.

```ts
@Module({ providers: [DbConnection], exports: [DbConnection] })
export class DatabaseModule {}

@Module({ imports: [DatabaseModule] })  export class UsersModule {}
@Module({ imports: [DatabaseModule] })  export class TasksModule {}
// One DbConnection instance, shared by both. Not one per importer.
```

This is what makes `imports` cheap. Importing a module in fifteen places costs nothing extra — you are declaring visibility, not requesting a copy.

A module can also **re-export** a module it imports, which is how you build a curated bundle:

```ts
@Module({
  imports: [DatabaseModule, CacheModule, ConfigModule],
  exports: [DatabaseModule, CacheModule, ConfigModule],   // pass the whole surface through
})
export class InfrastructureModule {}

// Now one import gets all three, and the set can change in one place.
@Module({ imports: [InfrastructureModule] })
export class TasksModule {}
```

Re-exporting a module exports everything *that module* exports — not its private providers. Encapsulation still holds one level down.

---

## 4.4 Shared vs Global Modules

Some things genuinely belong everywhere: configuration, logging, the database connection. There are two ways to provide them, and the choice is a real trade-off.

**A shared module** is an ordinary module that exports its providers, imported explicitly wherever needed. Verbose, and honest — every module's dependencies are visible in its own `imports`.

**A global module** registers itself once and becomes injectable everywhere:

```ts
@Global()                                  // only meaningful on the module where it is declared
@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule {}
```

`@Global()` still requires `exports` — it changes who can see the exports, not whether there are any. And it must still be imported **once**, typically in `AppModule`; `@Global()` on a module nobody imports does nothing.

| | Shared module | Global module |
|---|---|---|
| Dependencies visible in `imports` | Yes | No |
| Verbosity | Higher | Lower |
| Testability | Explicit — import what you need | Implicit — easy to forget in a test module |
| Refactoring safety | Removing it breaks compilation | Removing it breaks at runtime |
| Right for | Almost everything | Config, logging, and little else |

Prefer shared modules. `@Global()` trades a small amount of typing for the loss of the one property that makes a large Nest codebase navigable: being able to read a module's `imports` and know what it depends on. Two or three global modules in an application is reasonable; ten means the encapsulation has been abandoned.

---

## 4.5 Dynamic Modules

A static module is fixed at compile time. A **dynamic module** is a module configured by its consumer — the mechanism behind `TypeOrmModule.forRoot()`, `JwtModule.register()`, and `ConfigModule.forRoot()`.

A dynamic module is just a static method returning a `DynamicModule` object:

```ts
// src/storage/storage.module.ts
export interface StorageOptions {
  bucket: string;
  region: string;
}

export const STORAGE_OPTIONS = 'STORAGE_OPTIONS';

@Module({})
export class StorageModule {
  static forRoot(options: StorageOptions): DynamicModule {
    return {
      module: StorageModule,
      providers: [
        { provide: STORAGE_OPTIONS, useValue: options },   // config as a provider
        StorageService,
      ],
      exports: [StorageService],
      global: false,
    };
  }
}

// src/app.module.ts
@Module({
  imports: [StorageModule.forRoot({ bucket: 'uploads', region: 'eu-west-1' })],
})
export class AppModule {}
```

The naming is convention, and following it matters because every Nest developer reads it as a contract:

| Method | Meaning |
|---|---|
| `forRoot()` | Configure once, at the application root |
| `forRootAsync()` | Same, but options come from injected providers |
| `forFeature()` | Register a scoped slice — one entity, one queue, one namespace |
| `register()` | Configure per-import, with no global state implied |
| `registerAsync()` | The async form of `register()` |

`forRootAsync` exists because configuration usually comes from `ConfigService`, which is itself a provider — so the options cannot be known when `AppModule` is written:

```ts
static forRootAsync(options: {
  imports?: any[];
  inject?: any[];
  useFactory: (...args: any[]) => Promise<StorageOptions> | StorageOptions;
}): DynamicModule {
  return {
    module: StorageModule,
    imports: options.imports ?? [],
    providers: [
      {
        provide: STORAGE_OPTIONS,
        useFactory: options.useFactory,     // resolved by the container, after its deps
        inject: options.inject ?? [],
      },
      StorageService,
    ],
    exports: [StorageService],
  };
}

// Usage
StorageModule.forRootAsync({
  imports: [ConfigModule],
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    bucket: config.getOrThrow('S3_BUCKET'),
    region: config.getOrThrow('AWS_REGION'),
  }),
});
```

Writing both variants by hand for every module is tedious, so Nest provides `ConfigurableModuleBuilder`, which generates them:

```ts
// src/storage/storage.module-definition.ts
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<StorageOptions>()
    .setClassMethodName('forRoot')          // generates forRoot AND forRootAsync
    .build();

// src/storage/storage.module.ts
@Module({ providers: [StorageService], exports: [StorageService] })
export class StorageModule extends ConfigurableModuleClass {}
```

That is the whole implementation. `StorageModule.forRoot({...})` and `StorageModule.forRootAsync({...})` both exist, and `MODULE_OPTIONS_TOKEN` is injectable inside `StorageService`.

---

## 4.6 Circular Dependencies

Two modules that import each other will fail to resolve, because neither can be constructed first. `forwardRef()` defers the reference:

```ts
@Module({
  imports: [forwardRef(() => TasksModule)],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}

@Module({
  imports: [forwardRef(() => UsersModule)],
  providers: [TasksService],
  exports: [TasksService],
})
export class TasksModule {}
```

Providers in a cycle need it too, at the injection site:

```ts
constructor(
  @Inject(forwardRef(() => TasksService))
  private readonly tasks: TasksService,
) {}
```

`forwardRef` works, and reaching for it should still prompt a design question. A cycle means two modules each know something about the other, which usually indicates a third concept that belongs in its own module — or logic sitting on the wrong side of a boundary. The common fix is to extract the shared piece:

```
UsersModule ─┐
             ├─► MembershipModule   (owns the relationship both cared about)
TasksModule ─┘
```

Use `forwardRef` when the cycle is genuine and small. Treat three or more mutually-referencing modules as a signal to redraw the boundaries, because `forwardRef` will keep the application booting long after the structure has stopped making sense.

---

## 4.7 Common Mistakes and Best Practices

**Mistake: registering the same provider in two modules.** You get two instances with separate state, and the symptom — a cache that is sometimes stale — is far removed from the cause.

**Mistake: forgetting `exports`.** The provider is constructed but invisible outside its module, and the error names the *consumer*, not the missing export.

**Mistake: reaching for `@Global()` to fix a resolution error.** It works, and it deletes the boundary you were being told about.

**Mistake: `@Global()` without `exports`.** Global affects visibility of exports; with nothing exported there is nothing to see.

**Mistake: importing a module for a provider it does not export.** Importing grants access to the public surface only, not to everything inside.

**Mistake: using `forwardRef` as a first resort.** It silences the symptom of a boundary problem and leaves the problem in place.

**Mistake: putting logic in the module class.** A module is configuration; behaviour there is untestable and runs at an unpredictable time.

**Best practice: one module per bounded concept, not per layer.** `UsersModule` containing its controller, service, and entity beats `ControllersModule` + `ServicesModule`.

**Best practice: export the smallest surface that works.** Every export is a promise; a service exported "just in case" is a dependency someone will take.

**Best practice: use `ConfigurableModuleBuilder` for anything with options.** It generates both the sync and async variants correctly, and the hand-written async factory is easy to get subtly wrong.

---

## 4.8 Real-World Example: A Configurable Notifications Module

A module other teams install and configure, supporting synchronous and asynchronous options, with a swappable transport and a private implementation detail that must not leak.

```ts
// src/notifications/notifications.types.ts
export interface NotificationsOptions {
  defaultFrom: string;
  provider: 'ses' | 'smtp';
  retryAttempts?: number;
}

/** Public contract. An abstract class, not an interface, so it works as a DI token. */
export abstract class NotificationTransport {
  abstract send(to: string, subject: string, body: string): Promise<void>;
}
```

```ts
// src/notifications/notifications.module-definition.ts
import { ConfigurableModuleBuilder } from '@nestjs/common';
import { NotificationsOptions } from './notifications.types';

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<NotificationsOptions>()
    .setClassMethodName('forRoot')        // yields forRoot() and forRootAsync()
    .setExtras({ isGlobal: false }, (definition, extras) => ({
      ...definition,
      global: extras.isGlobal,            // consumer opts in to global, we do not decide
    }))
    .build();
```

```ts
// src/notifications/notifications.module.ts
import { Module, Provider } from '@nestjs/common';
import { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } from './notifications.module-definition';
import { NotificationsOptions, NotificationTransport } from './notifications.types';
import { NotificationsService } from './notifications.service';
import { SesTransport } from './transports/ses.transport';
import { SmtpTransport } from './transports/smtp.transport';
import { TemplateRenderer } from './template-renderer';

/**
 * The transport is chosen at runtime from the resolved options, so consumers
 * configure behaviour with data rather than by importing a different module.
 */
const transportProvider: Provider = {
  provide: NotificationTransport,
  inject: [MODULE_OPTIONS_TOKEN],
  useFactory: (options: NotificationsOptions): NotificationTransport =>
    options.provider === 'ses' ? new SesTransport(options) : new SmtpTransport(options),
};

@Module({
  providers: [
    transportProvider,
    TemplateRenderer,        // deliberately NOT exported — an internal detail
    NotificationsService,
  ],
  exports: [NotificationsService],   // the entire public surface, on purpose
})
export class NotificationsModule extends ConfigurableModuleClass {}
```

```ts
// src/notifications/notifications.service.ts
@Injectable()
export class NotificationsService {
  constructor(
    @Inject(MODULE_OPTIONS_TOKEN) private readonly options: NotificationsOptions,
    private readonly transport: NotificationTransport,
    private readonly renderer: TemplateRenderer,
  ) {}

  async notify(to: string, template: string, data: Record<string, unknown>): Promise<void> {
    const { subject, body } = this.renderer.render(template, data);
    const attempts = this.options.retryAttempts ?? 3;

    for (let attempt = 1; attempt <= attempts; attempt++) {
      try {
        return await this.transport.send(to, subject, body);
      } catch (err) {
        if (attempt === attempts) throw err;
        await new Promise((r) => setTimeout(r, 2 ** attempt * 100));   // backoff
      }
    }
  }
}
```

```ts
// src/app.module.ts — async, because the credentials live in config
@Module({
  imports: [
    NotificationsModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService): NotificationsOptions => ({
        defaultFrom: config.getOrThrow('MAIL_FROM'),
        provider: config.get('MAIL_PROVIDER') === 'ses' ? 'ses' : 'smtp',
        retryAttempts: 3,
      }),
    }),
  ],
})
export class AppModule {}
```

The decision worth defending is exporting only `NotificationsService` while `NotificationTransport` and `TemplateRenderer` stay private. Exporting the transport would be convenient — a caller could send a raw message and skip templating — and that convenience is exactly the problem. Every export is a commitment: once a consumer injects `NotificationTransport`, the retry policy and the template layer can be bypassed, and swapping the transport implementation becomes a breaking change for code you cannot see. A module's public surface should be the smallest set of operations that satisfies the use case, because shrinking it later requires finding every consumer, while growing it later requires only adding a line.

---

## 4.9 Questions

**1.** MCQ — Which array makes a provider available to other modules? a) `providers` b) `imports` c) `exports` d) `controllers`
<details><summary>Answer</summary>**c)** `exports`. `providers` only registers it locally.</details>

**2.** MCQ — Listing the same provider class in two modules' `providers` creates: a) one shared instance b) two instances c) a startup error d) a warning
<details><summary>Answer</summary>**b)** Two independent instances with separate state.</details>

**3.** MCQ — A module imported by five other modules is instantiated: a) once b) five times c) once per request d) lazily
<details><summary>Answer</summary>**a)** Modules are singletons per application.</details>

**4.** MCQ — `@Global()` removes the need for: a) `exports` b) `providers` c) `imports` in consumers d) `@Injectable()`
<details><summary>Answer</summary>**c)** Consumers no longer import it, but it still needs `exports` and one import at the root.</details>

**5.** MCQ — Which convention means "configure once at the root, with injected dependencies"? a) `register()` b) `forFeature()` c) `forRootAsync()` d) `registerAsync()`
<details><summary>Answer</summary>**c)** `forRootAsync()`.</details>

**6.** MCQ — `ConfigurableModuleBuilder` primarily saves you from writing: a) providers b) the sync and async option variants c) controllers d) tests
<details><summary>Answer</summary>**b)** It generates `forRoot`/`forRootAsync` (or `register`/`registerAsync`) and the options token.</details>

**7.** Short — Why does exporting a module differ from exporting a provider?
<details><summary>Answer</summary>Exporting a module re-exports whatever *that* module exports, passing a whole public surface through; exporting a provider exposes exactly one token.</details>

**8.** Short — What is the strongest argument against `@Global()`?
<details><summary>Answer</summary>It removes dependencies from the consumer's `imports`, so you can no longer read a module and know what it depends on — and tests silently miss providers that production supplies.</details>

**9.** Short — When is `forwardRef()` the right answer?
<details><summary>Answer</summary>When two modules or providers genuinely reference each other and the cycle is small and intentional. Beyond that it hides a boundary problem that should be fixed by extracting a third module.</details>

**10.** Short — Why should a module class body stay empty?
<details><summary>Answer</summary>Modules are configuration. Logic there is hard to test and runs at construction time, which is not a point in the lifecycle you should depend on.</details>

**11.** Short — What does `forFeature()` conventionally mean?
<details><summary>Answer</summary>Registering a scoped slice of an already-root-configured module — one entity, one queue, one namespace — rather than configuring the module globally again.</details>

**12.** Output — `TasksModule` has `providers: [TasksService]` and no `exports`. `ReportsModule` imports `TasksModule` and injects `TasksService`. What happens?
<details><summary>Answer</summary>Startup fails: Nest cannot resolve the dependency of `ReportsService` at index [0]. The error names the consumer, not the missing export — the usual source of confusion.</details>

**13.** Output — `CounterService` holds an in-memory count and is listed in both `UsersModule` and `TasksModule` providers. A request increments via Users, then reads via Tasks. What is read?
<details><summary>Answer</summary>`0`. There are two instances; the increment went to the other one.</details>

**14.** Output — A `@Global()` module is declared but never imported anywhere, including `AppModule`. Can its exports be injected?
<details><summary>Answer</summary>No. `@Global()` widens visibility for a module that is in the graph; a module nobody imports is not in the graph at all.</details>

**15.** Output — `InfrastructureModule` imports and re-exports `DatabaseModule`. `DatabaseModule` has a private provider `Migrator` (not exported). Can a module importing `InfrastructureModule` inject `Migrator`?
<details><summary>Answer</summary>No. Re-exporting passes along only what `DatabaseModule` itself exports; encapsulation still holds one level down.</details>

**16.** Debug — After splitting one module into two, an in-memory cache behaves as though it is randomly empty. What is the first thing to check?
<details><summary>Answer</summary>Whether the cache provider ended up in both modules' `providers`. Two instances means each consumer sees only its own writes. Register it once and export it.</details>

**17.** Debug — `ConfigService` resolves everywhere in production but is undefined in one unit test. Cause?
<details><summary>Answer</summary>`ConfigModule` is global, so production gets it implicitly, while `Test.createTestingModule` builds a fresh graph that never imported it. Import it explicitly in the test — the failure is the cost of `@Global()`.</details>

**18.** Debug — Startup fails with an error naming a module rather than a provider, mentioning a circular reference. Fix?
<details><summary>Answer</summary>Wrap each side's import in `forwardRef(() => OtherModule)`, and use `@Inject(forwardRef(() => OtherService))` at the injection sites. Then ask whether a third module should own the shared concept.</details>

**19.** Debug — A dynamic module's `useFactory` receives `undefined` for its `ConfigService` argument. What is missing?
<details><summary>Answer</summary>`ConfigModule` in the dynamic module's `imports`, and/or `ConfigService` in `inject`. The factory's parameters come from `inject` in order, and the providing module must be visible to the dynamic module itself.</details>

**20.** Debug — A newly added controller's routes 404, though its module is correct and the controller is listed. What else could it be?
<details><summary>Answer</summary>The module is not reachable from `AppModule` — it was created but never added to any `imports` array, so Nest never scanned it.</details>

**21.** Code — Convert this to share one `RedisClient` instance across `UsersModule` and `TasksModule`.
```ts
@Module({ providers: [RedisClient, UsersService] }) export class UsersModule {}
@Module({ providers: [RedisClient, TasksService] }) export class TasksModule {}
```
<details><summary>Answer</summary>

```ts
@Module({ providers: [RedisClient], exports: [RedisClient] })
export class RedisModule {}

@Module({ imports: [RedisModule], providers: [UsersService] })
export class UsersModule {}

@Module({ imports: [RedisModule], providers: [TasksService] })
export class TasksModule {}
```
One instance, owned by one module, shared by visibility rather than by duplication.
</details>

**22.** Code — Write a dynamic `RateLimitModule.forRoot({ windowMs, max })` exposing a `RateLimitService`.
<details><summary>Answer</summary>

```ts
export const RATE_LIMIT_OPTIONS = Symbol('RATE_LIMIT_OPTIONS');

@Module({})
export class RateLimitModule {
  static forRoot(options: { windowMs: number; max: number }): DynamicModule {
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
`RateLimitService` injects the options with `@Inject(RATE_LIMIT_OPTIONS)`. A `Symbol` token cannot collide with another module's string token.
</details>

**23.** Code — Add an async variant so the limit can come from `ConfigService`.
<details><summary>Answer</summary>

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
In practice, prefer `ConfigurableModuleBuilder`, which generates both variants and their types.
</details>

**24.** Scenario — Your application has grown to forty modules and adding a feature means touching six of them. A senior engineer proposes making the five most-used modules `@Global()` to cut the boilerplate. Do you agree?
<details><summary>Answer</summary>No — it treats a symptom and destroys the diagnostic. If a feature touches six modules, the boundaries are drawn around layers or entities rather than around use cases, and hiding the imports makes that harder to see, not better. Global modules also break tests in a specific way: the test module builds a fresh graph without them, so a provider available everywhere in production is missing in isolation. The exception is genuine cross-cutting infrastructure with no domain meaning — configuration and logging — where two or three global modules are reasonable. For the rest, an `InfrastructureModule` that re-exports the common set gives the same one-line import while keeping the dependency visible.</details>

**25.** Scenario — For consistency, a teammate proposes giving every module a `forRoot()` so all modules are configured the same way. Do you agree?
<details><summary>Answer</summary>No. A dynamic module exists to let a *consumer* vary behaviour it does not own — a bucket name, a transport, a connection string. Most modules in an application have exactly one consumer and nothing to vary, so `forRoot()` adds an options object, a token, and an async variant to maintain, in exchange for indirection that hides what the module actually does. Consistency is the wrong axis: the meaningful distinction is between modules that are *reused with different configuration* and modules that are simply part of this application. Make a module dynamic when a second consumer needs different behaviour, or when you are packaging it for other teams — and read `forRoot()` in someone else's module as a signal that it was built to be configured, which is information you lose if everything has one.</details>

---

## Chapter Summary

- `providers` registers, `exports` publishes; the gap between them is the encapsulation boundary, and forgetting `exports` produces an error naming the consumer rather than the cause.
- Registering one class in two modules creates two instances — a state-divergence bug that surfaces far from where it was introduced.
- Modules are singletons, so importing one in fifteen places declares visibility rather than requesting copies.
- Prefer shared modules to `@Global()`: the cost of a global is that a module's `imports` no longer tells you what it depends on, and tests stop matching production.
- Dynamic modules configure behaviour with data; follow the `forRoot`/`forFeature`/`register` naming, and let `ConfigurableModuleBuilder` generate the async variants.
- `forwardRef` resolves a cycle without resolving the design problem that produced it — extract a third module when the cycle is more than a pair.

---

## Practice Exercises

1. Deliberately trigger "Nest can't resolve dependencies" four different ways — missing `@Injectable()`, missing registration, missing export, missing import — and record each error message.
2. Register one stateful provider in two modules and write a test that proves the instances are distinct.
3. Build an `InfrastructureModule` that re-exports three modules, then verify that a private provider inside one of them is still unreachable.
4. Convert a hand-written `forRoot`/`forRootAsync` pair to `ConfigurableModuleBuilder` and diff the public API.
5. Create a circular dependency between two modules, fix it with `forwardRef`, then fix it again by extracting a third module. Compare the two results.

---

## Mini Project: A Feature-Flag Module

Build a reusable `FeatureFlagModule` other teams could install.

**Requirements:**

- `forRoot()` and `forRootAsync()`, both generated by `ConfigurableModuleBuilder`.
- Options select a backend — in-memory for tests, Redis for production — chosen by factory at runtime.
- Only `FeatureFlagService` is exported; the backend and any cache are private.
- `forFeature(namespace)` returns a namespaced service for one module, without re-configuring the root.
- A test proving that two modules importing `forFeature` with the same namespace share one instance, and different namespaces do not.

---

**Next:** [Chapter 5 — Controllers and Routing →](05-controllers-routing.md)
