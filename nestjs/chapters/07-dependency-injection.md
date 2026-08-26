# Chapter 7 — Dependency Injection Deep Dive

## Learning Objectives

- Describe what a DI token is and which values can serve as one
- Trace the container's resolution algorithm from a constructor parameter to an instance
- Read a "Nest can't resolve dependencies" error and name the cause without guessing
- Use `@Inject`, `@Optional`, and `useExisting` deliberately rather than by trial and error
- Escape the container with `ModuleRef` when static injection genuinely cannot work

---

## 7.1 Everything Is a Token

The container is a map from **tokens** to **instances**. A token is the key you register under and the key you look up by. Three kinds exist:

```ts
// 1. A class — the common case. The class itself is the token.
@Injectable() export class TasksService {}
providers: [TasksService]                      // token: TasksService

// 2. A string.
providers: [{ provide: 'CONFIG', useValue: { retries: 3 } }]

// 3. A symbol.
export const CONFIG = Symbol('CONFIG');
providers: [{ provide: CONFIG, useValue: { retries: 3 } }]
```

`providers: [TasksService]` is shorthand. It expands to:

```ts
providers: [{ provide: TasksService, useClass: TasksService }]
```

Seeing that expansion is what makes Chapter 8 obvious: the token and the implementation are separate, and the shorthand just happens to use the same class for both.

When you write a constructor, Nest reads the parameter's type from `design:paramtypes` metadata (→ [Ch 2](02-typescript-decorators.md)) and uses that class as the token:

```ts
constructor(private readonly tasks: TasksService) {}
// metadata says [TasksService] → look up the token TasksService
```

This is why an interface cannot be injected. It leaves no runtime value, so the metadata records `Object`, and the container has no meaningful key.

**String tokens collide; symbol tokens cannot.** Two libraries both registering `'CONFIG'` silently overwrite each other, and the winner depends on module order. A `Symbol` is unique by construction:

```ts
// ❌ Collides with anyone else's 'CONFIG'
providers: [{ provide: 'CONFIG', useValue: cfg }]

// ✅ Unique, and exported so consumers reference the same identity
export const APP_CONFIG = Symbol('APP_CONFIG');
providers: [{ provide: APP_CONFIG, useValue: cfg }]
```

---

## 7.2 The Resolution Algorithm

When the container needs a token for a provider inside module M, it looks in this order:

1. **M's own `providers`.**
2. **The `exports` of every module M `imports`** — and transitively, whatever those modules re-export.
3. **Any `@Global()` module's exports.**
4. Otherwise: **fail at startup**.

Three properties of this algorithm cause most of the confusion.

**Visibility is per-module, not per-application.** A provider registered in `UsersModule` is invisible in `TasksModule` unless `UsersModule` exports it *and* `TasksModule` imports `UsersModule`. Both halves are required.

**Only exports cross the boundary.** Importing a module gives you its public surface, not its internals (→ [Ch 4](04-modules.md)).

**Resolution is depth-first and eager.** To construct `A`, the container first constructs everything `A` needs, and so on. The entire graph is built at startup, in dependency order, before the server listens — which is why a missing token is a boot failure, not a request failure.

```ts
@Module({ providers: [Db], exports: [Db] })
export class DbModule {}

@Module({ imports: [DbModule], providers: [UsersService], exports: [UsersService] })
export class UsersModule {}

@Module({ imports: [UsersModule], providers: [ReportsService] })
export class ReportsModule {}
```

`ReportsService` can inject `UsersService` — exported by `UsersModule`, which `ReportsModule` imports. It **cannot** inject `Db`, even though `Db` is reachable two hops away, because `UsersModule` does not re-export `DbModule`. Reachability is not visibility.

---

## 7.3 Reading the Error

Nest's resolution error is more informative than it looks. Learn its anatomy once and it becomes a diagnosis.

```
Nest can't resolve dependencies of the TasksService (?, NotificationsService).
Please make sure that the argument TaskRepository at index [0] is available
in the TasksModule context.
```

| Fragment | Meaning |
|---|---|
| `TasksService` | the class being constructed |
| `(?, NotificationsService)` | its constructor parameters; **`?` is the one that failed** |
| `at index [0]` | which parameter position |
| `TaskRepository` | the token it looked for |
| `in the TasksModule context` | the module whose visibility was searched |

The parenthesised list is the most useful part: everything resolved except the `?`. If you see `(?, ?, ?)`, none resolved, which usually means the class has no decorator at all.

Four causes, in the order worth checking:

| Symptom | Cause | Fix |
|---|---|---|
| All parameters are `?` | no `@Injectable()`, so no metadata | add the decorator |
| Token named, module named | not exported by its owner | add to `exports` |
| Token named, module named | owner not imported | add to `imports` |
| Token is `Object` | you injected an interface or type alias | use `@Inject(TOKEN)` or an abstract class |

> 💼 **Interview** — Being handed this error and walking it methodically — "index 0 is `TaskRepository`, the context is `TasksModule`, so either `TasksModule` doesn't import the owner or the owner doesn't export it" — is a much stronger signal than knowing the API surface.

> ⚠️ **Gotcha** — The error names the **consumer**, not the module that should have exported the token. Developers habitually open `TasksModule` and add the provider to its `providers` array, which "fixes" it by creating a second instance (→ [Ch 4](04-modules.md)) instead of importing the module that owns it.

---

## 7.4 `@Inject`, `@Optional`, and `useExisting`

`@Inject()` supplies a token explicitly, which you need whenever the type annotation is not a usable one:

```ts
@Injectable()
export class MailService {
  constructor(
    @Inject(APP_CONFIG) private readonly config: AppConfig,   // symbol token
    @Inject('MAIL_TRANSPORT') private readonly transport: Transport,
  ) {}
}
```

`@Optional()` makes a missing dependency `undefined` instead of a startup failure:

```ts
constructor(
  @Optional() @Inject(METRICS) private readonly metrics?: MetricsClient,
) {}

record(name: string) {
  this.metrics?.increment(name);      // must handle absence at every use
}
```

Use it for genuinely optional collaborators — telemetry, a cache in front of a working fallback. Do not use it to silence a resolution error you have not understood: you have converted a loud startup failure into a silent runtime one.

`useExisting` creates an **alias** — a second token pointing at the same instance:

```ts
providers: [
  LoggerService,
  { provide: 'LOGGER', useExisting: LoggerService },   // alias, NOT a copy
]
```

Contrast with `useClass`, which constructs a separate instance:

| | `useExisting` | `useClass` |
|---|---|---|
| Instances | one, shared | a new one per token |
| Use for | aliasing, deprecating a token | providing an implementation |

Aliasing is how you rename a token without breaking consumers: register the new one, alias the old, migrate, then delete the alias.

---

## 7.5 `ModuleRef`: Escaping the Container

Constructor injection is static — resolved once at startup. When the thing you need is only known at runtime, `ModuleRef` gives you programmatic access:

```ts
@Injectable()
export class PaymentDispatcher {
  constructor(private readonly moduleRef: ModuleRef) {}

  getGateway(name: 'stripe' | 'paypal'): PaymentGateway {
    const token = name === 'stripe' ? StripeGateway : PaypalGateway;
    return this.moduleRef.get(token, { strict: false });
  }
}
```

Three methods, and the difference between them matters:

| Method | Returns | Use for |
|---|---|---|
| `get(token)` | the existing instance, synchronously | default-scoped providers |
| `resolve(token)` | a **new** instance, as a promise | request/transient-scoped providers (→ [Ch 9](09-scopes-lifecycle.md)) |
| `create(SomeClass)` | a new instance of an unregistered class | one-off construction with injection |

`{ strict: false }` searches the whole application rather than just the current module. It is convenient and it deliberately bypasses encapsulation, so treat it as a marked exception.

> ⚠️ **Gotcha** — `moduleRef.get()` on a request-scoped provider throws. Scoped providers have no single instance to return; you must use `await moduleRef.resolve()`, which creates one and ties it to a context identifier.

`ModuleRef` is a service locator, and service locators hide dependencies — a class taking `ModuleRef` in its constructor tells you nothing about what it actually uses. Reach for it only when the token is genuinely dynamic, and prefer a factory provider (→ [Ch 8](08-custom-providers.md)) when the set of implementations is known at startup.

---

## 7.6 Circular Dependencies

Two providers that need each other cannot both be constructed first. Nest reports a cycle, and `forwardRef` defers the reference until both classes exist:

```ts
@Injectable()
export class UsersService {
  constructor(
    @Inject(forwardRef(() => TasksService))
    private readonly tasks: TasksService,
  ) {}
}

@Injectable()
export class TasksService {
  constructor(
    @Inject(forwardRef(() => UsersService))
    private readonly users: UsersService,
  ) {}
}
```

If the two live in different modules, both the module imports **and** the injection sites need it (→ [Ch 4](04-modules.md)).

`forwardRef` resolves the mechanical problem and leaves the design problem. A cycle means two classes each depend on the other's behaviour, and there are usually three better fixes:

- **Extract the shared concern.** If both need "can this user be assigned a task", that rule belongs in a third service both depend on.
- **Invert with an event.** `UsersService` emits `user.deleted`; `TasksService` listens. The dependency becomes one-directional (→ [Ch 28](28-cqrs-events.md)).
- **Move the method.** Often one of the two calls is on the wrong class.

Reach for `forwardRef` for a genuine, small, intentional cycle. Three or more mutually referencing providers is a boundary problem that `forwardRef` will hide indefinitely.

---

## 7.7 Common Mistakes and Best Practices

**Mistake: fixing a resolution error by adding the provider to the consumer's `providers`.** It resolves and creates a second instance with separate state.

**Mistake: injecting an interface.** The metadata records `Object`; use an abstract class or an explicit `@Inject()` token.

**Mistake: string tokens in shared code.** Two libraries registering `'CONFIG'` overwrite each other, and which one wins depends on module order.

**Mistake: `@Optional()` to silence an error.** It converts a startup failure into a runtime `undefined`, usually far from the cause.

**Mistake: `moduleRef.get()` on a scoped provider.** It throws — scoped providers require `resolve()`.

**Mistake: `{ strict: false }` as a default.** It bypasses module encapsulation everywhere, which removes the boundary the error was reporting.

**Mistake: `ModuleRef` where a factory would do.** If the implementations are known at startup, a `useFactory` provider keeps the dependency visible.

**Best practice: export tokens as `Symbol` constants from one file.** Unique by construction, and the import makes the dependency traceable.

**Best practice: read the error's parenthesised parameter list first.** It tells you immediately whether one dependency failed or the class has no metadata at all.

**Best practice: prefer an abstract class to a string token for a contract.** You get a real token, an enforced shape, and editor navigation in one move.

---

## 7.8 Real-World Example: A Pluggable Gateway With a Deprecated Token

A payments module supporting several gateways chosen by configuration, exposing a stable contract, while keeping an old token alive for consumers that have not migrated.

```ts
// src/payments/payment-gateway.ts
/** The contract. An abstract class, so it is both a type and a DI token. */
export abstract class PaymentGateway {
  abstract charge(amountCents: number, token: string): Promise<Receipt>;
  abstract refund(receiptId: string): Promise<void>;
}

export const PAYMENT_OPTIONS = Symbol('PAYMENT_OPTIONS');
export interface PaymentOptions {
  provider: 'stripe' | 'paypal' | 'fake';
  apiKey: string;
}
```

```ts
// src/payments/payments.module.ts
import { Module, Provider, Logger } from '@nestjs/common';

/**
 * One factory chooses the implementation from resolved configuration.
 * The set of gateways is known at startup, so this is a factory rather than
 * a ModuleRef lookup — the dependency stays visible in the provider list.
 */
const gatewayProvider: Provider = {
  provide: PaymentGateway,
  inject: [PAYMENT_OPTIONS, Logger],
  useFactory: (options: PaymentOptions, logger: Logger): PaymentGateway => {
    switch (options.provider) {
      case 'stripe':
        return new StripeGateway(options.apiKey, logger);
      case 'paypal':
        return new PaypalGateway(options.apiKey, logger);
      case 'fake':
        // Guard the test double so it can never be selected in production.
        if (process.env.NODE_ENV === 'production') {
          throw new Error('FakeGateway cannot be used in production');
        }
        return new FakeGateway();
    }
  },
};

@Module({
  providers: [
    gatewayProvider,
    PaymentsService,
    // Deprecated alias. useExisting means both tokens resolve to ONE instance,
    // so migrating consumers can happen gradually with no behaviour change.
    { provide: 'PAYMENT_GATEWAY', useExisting: PaymentGateway },
  ],
  exports: [PaymentsService, PaymentGateway, 'PAYMENT_GATEWAY'],
})
export class PaymentsModule {}
```

```ts
// src/payments/payments.service.ts
@Injectable()
export class PaymentsService {
  constructor(
    private readonly gateway: PaymentGateway,          // resolved by class token
    @Inject(PAYMENT_OPTIONS) private readonly options: PaymentOptions,
    // Genuinely optional: metrics are nice to have, and the service must work
    // in a test module that never registers them.
    @Optional() @Inject(METRICS) private readonly metrics?: MetricsClient,
  ) {}

  async charge(amountCents: number, cardToken: string): Promise<Receipt> {
    const started = Date.now();
    try {
      const receipt = await this.gateway.charge(amountCents, cardToken);
      this.metrics?.timing('payment.charge.ok', Date.now() - started);
      return receipt;
    } catch (err) {
      this.metrics?.increment('payment.charge.failed');
      throw err;
    }
  }
}
```

The decision worth defending is `useExisting` rather than registering the old token with its own `useFactory`. Duplicating the factory would resolve both tokens and produce **two gateway instances** — two HTTP connection pools, two circuit-breaker states, and a rate limiter that counts half the traffic. Because the bug only appears under load and only in whichever half of the code uses the old token, it is close to undiagnosable from a stack trace. `useExisting` guarantees one instance behind both names, which is exactly what a deprecation alias should mean: a second name, never a second object. The other decision worth noting is throwing on `fake` in production rather than silently falling back — a payment gateway that quietly accepts every charge is the worst possible default, so it fails at boot where someone will notice.

---

## 7.9 Questions

**1.** MCQ — `providers: [TasksService]` is shorthand for: a) `useValue` b) `useClass` with the same token c) `useExisting` d) `useFactory`
<details><summary>Answer</summary>**b)** `{ provide: TasksService, useClass: TasksService }`.</details>

**2.** MCQ — In `(?, NotificationsService)`, the `?` marks: a) an optional dependency b) the parameter that failed to resolve c) a circular reference d) a scoped provider
<details><summary>Answer</summary>**b)** The unresolvable parameter; everything else in the list resolved fine.</details>

**3.** MCQ — Which token type cannot collide across libraries? a) string b) class c) symbol d) both b and c
<details><summary>Answer</summary>**d)** Classes and symbols are both unique by identity; only strings collide.</details>

**4.** MCQ — `useExisting` produces: a) a new instance b) an alias to one instance c) a factory d) a scoped provider
<details><summary>Answer</summary>**b)** A second token resolving to the same instance.</details>

**5.** MCQ — `moduleRef.get()` on a request-scoped provider: a) returns the instance b) returns undefined c) throws d) creates one
<details><summary>Answer</summary>**c)** It throws; scoped providers need `await moduleRef.resolve()`.</details>

**6.** MCQ — Resolution searches, in order: a) global, then local b) own providers, then imports' exports, then globals c) imports only d) alphabetically
<details><summary>Answer</summary>**b)**</details>

**7.** Short — Why can't you inject an interface?
<details><summary>Answer</summary>Interfaces are erased at compile time, so `design:paramtypes` records `Object` and there is no distinct token to look up. Use an abstract class or an explicit `@Inject()` token.</details>

**8.** Short — A module imports another module whose provider it still cannot inject. Why?
<details><summary>Answer</summary>The provider is not in the imported module's `exports`. Importing grants the public surface only.</details>

**9.** Short — Why is fixing a resolution error by adding the provider locally dangerous?
<details><summary>Answer</summary>It creates a second instance with independent state, so caches, counters, and pools diverge — and the error goes away, which makes it look correct.</details>

**10.** Short — When is `@Optional()` appropriate?
<details><summary>Answer</summary>When the collaborator is genuinely optional and every use site handles its absence — telemetry, an optional cache. Never to silence an error you have not diagnosed.</details>

**11.** Short — What does `{ strict: false }` change, and what does it cost?
<details><summary>Answer</summary>It searches the whole application instead of the current module, at the cost of bypassing module encapsulation — so it should be a marked exception, not a habit.</details>

**12.** Output — `ReportsModule` imports `UsersModule`, which imports `DbModule` and exports only `UsersService`. Can `ReportsService` inject `Db`?
<details><summary>Answer</summary>No. `Db` is reachable but not visible — `UsersModule` would have to re-export `DbModule`. Reachability is not visibility.</details>

**13.** Output — What is logged?
```ts
providers: [
  LoggerService,
  { provide: 'LOGGER', useExisting: LoggerService },
]
// elsewhere
console.log(logger === loggerByStringToken);
```
<details><summary>Answer</summary>`true` — one instance behind two tokens.</details>

**14.** Output — Same as above but with `useClass: LoggerService` instead of `useExisting`. Now what?
<details><summary>Answer</summary>`false` — `useClass` constructs a separate instance, so the two tokens hold different objects with independent state.</details>

**15.** Output — An error reads `Nest can't resolve dependencies of the AuthService (?, ?, ?)`. What does the all-`?` pattern indicate?
<details><summary>Answer</summary>No parameter resolved, which almost always means the class is missing `@Injectable()`, so no `design:paramtypes` metadata exists at all.</details>

**16.** Debug — Two modules each register `{ provide: 'CONFIG', useValue: … }`. One module reads the other's values. Cause and fix?
<details><summary>Answer</summary>String token collision — whichever registration the container resolved last wins for shared consumers. Replace both with exported `Symbol` tokens, which cannot collide.</details>

**17.** Debug — After adding `@Optional()` the app boots, but a method throws `Cannot read properties of undefined`. What happened?
<details><summary>Answer</summary>The dependency was never resolvable; `@Optional()` turned the startup failure into `undefined` and moved the crash to first use. Fix the wiring, or guard every use with `?.`.</details>

**18.** Debug — A provider's constructor runs twice with different dependency instances. Likely cause?
<details><summary>Answer</summary>It is registered in two modules, or a second token provides it via `useClass` instead of `useExisting`. Register once and export, or alias.</details>

**19.** Debug — Startup fails with a circular dependency between `UsersService` and `TasksService` in different modules. What are the two places to fix?
<details><summary>Answer</summary>Both module `imports` need `forwardRef(() => OtherModule)`, and both injection sites need `@Inject(forwardRef(() => OtherService))`. Fixing only one side leaves the cycle unresolved.</details>

**20.** Debug — `moduleRef.get(SomeService)` throws even though the service is registered and the app boots. What is the likely reason?
<details><summary>Answer</summary>Either it is scoped — needing `resolve()` — or it lives in another module and the call needs `{ strict: false }`.</details>

**21.** Code — Register a config object under a collision-proof token and inject it.
<details><summary>Answer</summary>

```ts
export const APP_CONFIG = Symbol('APP_CONFIG');

@Module({
  providers: [{ provide: APP_CONFIG, useValue: { retries: 3 } }],
  exports: [APP_CONFIG],
})
export class ConfigModule {}

@Injectable()
export class Svc {
  constructor(@Inject(APP_CONFIG) private readonly config: AppConfig) {}
}
```
</details>

**22.** Code — Deprecate the token `'MAILER'` in favour of `MailService` without breaking existing consumers or duplicating the instance.
<details><summary>Answer</summary>

```ts
@Module({
  providers: [
    MailService,
    { provide: 'MAILER', useExisting: MailService },   // one instance, two names
  ],
  exports: [MailService, 'MAILER'],
})
export class MailModule {}
```
Migrate consumers to `MailService`, then delete the alias. `useClass` here would silently create a second mailer.
</details>

**23.** Code — Resolve a request-scoped provider from a singleton using `ModuleRef`.
<details><summary>Answer</summary>

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
`resolve` returns a new instance per context id; `get` would throw for a scoped provider (→ Ch 9).
</details>

**24.** Scenario — A junior developer keeps hitting resolution errors and has started adding every provider to every module's `providers` array. It works. What do you tell them?
<details><summary>Answer</summary>That the errors have stopped and the bugs have started. Each duplicate registration is a separate instance, so any provider holding a cache, a counter, a connection pool, or a circuit-breaker state is now inconsistent depending on which module the caller lives in — and the symptom will appear far from the cause. Walk them through the error anatomy instead: the token, the index, and the context module tell you whether the fix is an `exports` entry or an `imports` entry. The rule to internalise is that a provider is registered exactly once, in the module that owns it, and shared by exporting.</details>

**25.** Scenario — Your team wants to inject the current request's tenant id into a dozen services. Someone suggests a global mutable holder set by middleware. Evaluate it.
<details><summary>Answer</summary>Reject it — it is the singleton-state bug from Chapter 6 at application scope, and it will interleave tenants under concurrency at the first `await`. There are three defensible options. Passing the tenant id as a parameter is the most explicit and the most tedious. `Scope.REQUEST` providers give you a per-request instance, at the cost of promoting every consumer to request scope (→ Ch 9). `AsyncLocalStorage` gives ambient per-request context without changing signatures, which is what most production systems settle on for exactly this case, though it is implicit and harder to trace. Choose by blast radius: parameters for a handful of call sites, `AsyncLocalStorage` for a dozen, request scope only where you also need per-request construction rather than just per-request data.</details>

---

## Chapter Summary

- The container maps tokens to instances; `providers: [X]` is shorthand for `{ provide: X, useClass: X }`, and seeing that makes custom providers unremarkable.
- Resolution checks the module's own providers, then its imports' exports, then global modules — so both an `exports` and an `imports` entry are required to share a provider.
- The error message is a diagnosis: the `?` marks the failed parameter, the index locates it, and the named context is the module whose visibility was searched.
- It names the consumer rather than the owner, which is why the tempting fix — registering the provider locally — silently creates a second instance.
- `useExisting` aliases one instance under a second token; `useClass` builds a new one, which turns a deprecation alias into a duplicate-object bug.
- `ModuleRef` is a service locator that hides dependencies — use it only for genuinely dynamic tokens, and `resolve()` rather than `get()` for scoped providers.

---

## Practice Exercises

1. Trigger all four resolution failures deliberately and record the exact error text of each.
2. Build a three-module chain and prove that a provider two hops away is unreachable without a re-export.
3. Register the same stateful provider under `useExisting` and then `useClass`, and write a test that shows one shares state and the other does not.
4. Replace a `ModuleRef` lookup with a factory provider and compare what each reveals about the class's dependencies.
5. Create a string-token collision between two modules, observe which wins, then fix it with symbols.

---

## Mini Project: A Dependency Graph Inspector

Build a small diagnostic module that reports the application's wiring at startup.

**Requirements:**

- On `onApplicationBootstrap`, walk the container and log every token with its providing module.
- Flag any token registered in more than one module, with the modules named.
- Flag providers reachable only through `@Global()` modules, since those are invisible in `imports`.
- Print the resolution path for a token supplied on the command line.
- Output as a Mermaid diagram so the module graph can be viewed.

---

**Next:** [Chapter 8 — Custom Providers →](08-custom-providers.md)
