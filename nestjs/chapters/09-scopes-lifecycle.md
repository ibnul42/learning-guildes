# Chapter 9 — Provider Scopes and Lifecycle

## Learning Objectives

- Choose between `DEFAULT`, `REQUEST`, and `TRANSIENT` scope from the requirement
- Predict which providers become request-scoped when one dependency is, and bound the damage
- Explain why lifecycle hooks never fire on request-scoped classes
- Order the initialization and termination hooks, and say what `enableShutdownHooks()` changes
- Use durable providers to make multi-tenant request scope affordable

---

## 9.1 The Three Scopes

Scope decides how many instances exist and how long each lives.

```ts
@Injectable()                                        // DEFAULT — one per application
export class TasksService {}

@Injectable({ scope: Scope.REQUEST })                // one per incoming request
export class RequestContextService {}

@Injectable({ scope: Scope.TRANSIENT })              // one per consumer
export class LoggerService {}
```

| Scope | Instances | Lifetime | Lifecycle hooks fire? |
|---|---|---|---|
| `DEFAULT` | one per application | application | Yes |
| `REQUEST` | one per request | the request | **No** |
| `TRANSIENT` | one per injecting consumer | its consumer's | Init only, not termination |

`DEFAULT` is right for almost everything. It is the only scope where a provider is constructed once, at startup, so wiring failures surface at boot and per-request cost is zero.

`TRANSIENT` is easy to misread. It does not mean "new instance per call" — it means **each consumer gets its own instance**, created when that consumer is created. A singleton injecting a transient provider gets one dedicated instance for the application's lifetime, not a fresh one per method call. Its main use is a provider that needs to know who injected it:

```ts
@Injectable({ scope: Scope.TRANSIENT })
export class ContextLogger {
  private context = 'App';
  setContext(name: string) { this.context = name; }
  log(msg: string) { console.log(`[${this.context}] ${msg}`); }
}

@Injectable()
export class TasksService {
  constructor(private readonly logger: ContextLogger) {
    this.logger.setContext(TasksService.name);   // safe: this instance is ours alone
  }
}
```

With `DEFAULT` scope that `setContext` call would be a bug — the last service constructed would win, and every other service would log under the wrong name.

---

## 9.2 Scope Bubbling

This is the section that matters most.

**Request scope propagates upward.** If a provider is `REQUEST`-scoped, every provider that injects it becomes request-scoped too, and so does anything injecting *those* — all the way up to the controller.

```ts
@Injectable({ scope: Scope.REQUEST })
export class TenantContext {}                    // explicitly request-scoped

@Injectable()                                     // looks like a singleton…
export class BillingService {
  constructor(private readonly tenant: TenantContext) {}
}                                                 // …but is now request-scoped

@Controller('invoices')                           // …and so is this
export class InvoicesController {
  constructor(private readonly billing: BillingService) {}
}
```

Nothing warns you. `BillingService` still reads `@Injectable()` with no scope, and it is now constructed on every request along with its entire dependency subtree. In a large application, one request-scoped provider near the bottom of the graph can promote most of the application.

**Transient scope does not bubble.** A singleton can inject a transient provider and stay a singleton — it just receives its own dedicated instance. This asymmetry is a common interview question and the reason `TRANSIENT` is the cheaper tool when you only need per-consumer identity.

The practical consequence: **keep request-scoped providers at the edges**. A request-scoped provider injected by one narrow service is fine. One injected by a service that half the application depends on is an architecture-wide change made by one line.

> ⚠️ **Gotcha** — A `REQUEST`-scoped provider cannot be injected into a singleton without promoting it. If you need request data inside a genuinely long-lived object — a scheduler, a queue consumer — the answer is not request scope. Pass the data as a parameter, or use `AsyncLocalStorage` for ambient context.

> 💼 **Interview** — "What happens if you inject a request-scoped provider into a singleton?" The expected answer is that the singleton silently becomes request-scoped, along with everything above it, and that the cost is per-request instantiation of that whole subtree.

---

## 9.3 Working With Request Scope

A request-scoped provider can inject the request object itself:

```ts
import { REQUEST } from '@nestjs/core';

@Injectable({ scope: Scope.REQUEST })
export class TenantContext {
  readonly tenantId: string;

  constructor(@Inject(REQUEST) private readonly request: Request) {
    this.tenantId = request.headers['x-tenant-id'] as string;
  }
}
```

The official guidance is that a well-designed application using request scope should lose no more than roughly 5% in latency. That number is easy to blow past, because the cost is not the scoped provider — it is everything bubbling above it.

To reach a scoped provider from a singleton without promoting it, resolve it explicitly (→ [Ch 7](07-dependency-injection.md)):

```ts
@Injectable()                                     // stays a singleton
export class JobDispatcher {
  constructor(private readonly moduleRef: ModuleRef) {}

  async run(payload: Payload) {
    const contextId = ContextIdFactory.create();
    const handler = await this.moduleRef.resolve(ScopedHandler, contextId);
    return handler.handle(payload);
  }
}
```

`moduleRef.get()` throws for a scoped provider — there is no single instance to return. `resolve()` creates one tied to a context identifier, and passing the same `contextId` twice returns the same instance.

---

## 9.4 Durable Providers

Multi-tenancy is the case where request scope is genuinely justified and genuinely expensive: every request rebuilds the tenant's subtree, even though there are twenty tenants and thousands of requests.

**Durable providers** let Nest reuse a dependency subtree keyed by something other than the request — usually the tenant:

```ts
// src/tenancy/tenant-context.strategy.ts
import { ContextIdFactory, ContextIdStrategy, HostComponentInfo } from '@nestjs/core';

const tenants = new Map<string, ContextId>();

export class TenantContextIdStrategy implements ContextIdStrategy {
  attach(contextId: ContextId, request: Request) {
    const tenantId = request.headers['x-tenant-id'] as string;
    if (!tenantId) return () => contextId;         // no tenant: normal per-request behaviour

    let tenantSubTreeId = tenants.get(tenantId);
    if (!tenantSubTreeId) {
      tenantSubTreeId = ContextIdFactory.create();
      tenants.set(tenantId, tenantSubTreeId);
    }

    // Durable providers reuse the tenant's subtree; everything else stays per-request.
    return (info: HostComponentInfo) =>
      info.isTreeDurable ? tenantSubTreeId! : contextId;
  }
}

// src/main.ts
ContextIdFactory.apply(new TenantContextIdStrategy());
```

```ts
@Injectable({ scope: Scope.REQUEST, durable: true })
export class TenantConnection {
  constructor(@Inject(REQUEST) private readonly payload: { tenantId: string }) {}
}
```

Now twenty tenants mean twenty subtrees rather than one per request.

The trade-off is real: a durable provider **outlives the request**, so it is a singleton per tenant, and the stateless rule from [Chapter 6](06-providers-services.md) applies again. Storing anything request-specific on a durable provider reintroduces exactly the cross-request corruption request scope was meant to prevent. The `Map` above also grows without bound — a production implementation needs eviction.

---

## 9.5 Lifecycle Hooks

Five hooks, in two phases. Implement the matching interface:

```ts
@Injectable()
export class DbService
  implements OnModuleInit, OnApplicationBootstrap, OnModuleDestroy, OnApplicationShutdown
{
  async onModuleInit() { await this.connect(); }            // 1
  async onApplicationBootstrap() { await this.warmCache(); } // 2
  async onModuleDestroy() { await this.stopAcceptingWork(); }// 3
  async onApplicationShutdown(signal?: string) {             // 5
    await this.pool.end();
  }
}
```

**Initialization order:**

1. `onModuleInit` — once the host module's dependencies are resolved. Modules initialise dependencies-first.
2. `onApplicationBootstrap` — once *every* module has initialised, before the server listens.

The distinction matters: in `onModuleInit` you may only rely on your own module's dependencies. Anything needing the whole application to exist belongs in `onApplicationBootstrap`.

**Termination order:**

3. `onModuleDestroy` — after a termination signal.
4. `beforeApplicationShutdown` — after every `onModuleDestroy` completes.
5. `onApplicationShutdown` — after connections close.

All five may return a promise, and Nest awaits them.

> ⚠️ **Gotcha** — **Lifecycle hooks never fire on request-scoped providers.** The docs are explicit: request-scoped classes are not tied to the application lifecycle, are created per request, and are garbage-collected after the response. An `onModuleDestroy` on a request-scoped class that closes a handle will never run, and the handle leaks once per request.

**Nest 11 reversed the traversal order within the termination phases.** The sequence of the three hooks is unchanged, but the order modules are visited inside each phase is now the opposite of Nest 10. Code that assumed a dependency module was still alive during its own teardown can now find it already destroyed — audit anything doing real work in `onModuleDestroy` when upgrading.

---

## 9.6 Graceful Shutdown

Termination hooks do not run unless you opt in:

```ts
// src/main.ts
app.enableShutdownHooks();     // listens for SIGTERM, SIGINT, SIGBREAK, SIGHUP
```

Without it, `SIGTERM` kills the process outright: no pool drain, no in-flight request completion, and no hook you wrote ever executes. Under a rolling deploy this produces a burst of dropped connections every release (→ [Ch 3](03-first-application.md)).

There is a cost to enabling it — Nest attaches listeners to those signals for the process lifetime — which is why it is opt-in rather than default. For any long-running service, enable it.

---

## 9.7 Common Mistakes and Best Practices

**Mistake: injecting a request-scoped provider deep in the graph.** It promotes every consumer above it, silently, and the per-request cost is the whole subtree.

**Mistake: expecting `TRANSIENT` to mean per-call.** It means per-consumer; a singleton's transient dependency lives as long as the singleton.

**Mistake: lifecycle hooks on request-scoped classes.** They never fire, so any cleanup written there leaks once per request.

**Mistake: forgetting `enableShutdownHooks()`.** Every termination hook you wrote is dead code.

**Mistake: request state on a durable provider.** Durable means per-tenant and long-lived, so the cross-request corruption from Chapter 6 returns.

**Mistake: an unbounded tenant map in a `ContextIdStrategy`.** It grows forever; entries need eviction.

**Mistake: heavy work in `onModuleInit` that needs another module.** Only your own module's dependencies are guaranteed ready — use `onApplicationBootstrap`.

**Best practice: default to `DEFAULT`.** Reach for request scope only when per-request construction is genuinely required, not merely per-request *data*.

**Best practice: keep request-scoped providers at the leaves.** Bounding what sits above one bounds the blast radius.

**Best practice: prefer `AsyncLocalStorage` for ambient request context.** It gives per-request data without changing any provider's scope.

---

## 9.8 Real-World Example: Per-Request Context Without Request Scope

A correlation id and authenticated user available anywhere in the call stack, with **every provider remaining a singleton**. This is the pattern most production Nest services converge on, precisely because it avoids scope bubbling.

```ts
// src/context/request-context.ts
import { AsyncLocalStorage } from 'node:async_hooks';
import { Injectable } from '@nestjs/common';

export interface RequestStore {
  correlationId: string;
  userId?: string;
  tenantId?: string;
}

@Injectable()
export class RequestContext {
  // One ALS instance for the process. Node keeps a separate store per async
  // execution chain, which is exactly one HTTP request here.
  private readonly als = new AsyncLocalStorage<RequestStore>();

  run<T>(store: RequestStore, fn: () => T): T {
    return this.als.run(store, fn);
  }

  get(): RequestStore | undefined {
    return this.als.getStore();
  }

  get correlationId(): string {
    return this.als.getStore()?.correlationId ?? 'no-context';
  }

  /**
   * Mutating the existing store rather than replacing it: guards run after
   * middleware, so the user is only known once the store already exists.
   */
  setUser(userId: string, tenantId?: string): void {
    const store = this.als.getStore();
    if (store) {
      store.userId = userId;
      store.tenantId = tenantId;
    }
  }
}
```

```ts
// src/context/request-context.middleware.ts
@Injectable()
export class RequestContextMiddleware implements NestMiddleware {
  constructor(private readonly context: RequestContext) {}

  use(req: Request, res: Response, next: NextFunction): void {
    const correlationId =
      (req.headers['x-correlation-id'] as string) ?? randomUUID();

    res.setHeader('x-correlation-id', correlationId);

    // Everything downstream — guards, pipes, the handler, services, even
    // callbacks scheduled inside them — runs inside this store.
    this.context.run({ correlationId }, () => next());
  }
}
```

```ts
// src/tasks/tasks.service.ts — a plain singleton with per-request awareness
@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  constructor(
    private readonly repo: TaskRepository,
    private readonly context: RequestContext,   // singleton, injected normally
  ) {}

  async create(dto: CreateTaskDto): Promise<Task> {
    const { correlationId, userId } = this.context.get() ?? {};

    this.logger.log(`Creating task [${correlationId}] for user ${userId}`);
    return this.repo.save(Task.create({ ...dto, ownerId: userId! }));
  }
}
```

```ts
// src/app.module.ts
@Module({
  providers: [RequestContext],
  exports: [RequestContext],
})
export class ContextModule implements NestModule {
  configure(consumer: MiddlewareConsumer): void {
    consumer.apply(RequestContextMiddleware).forRoutes('*splat');   // Express 5 syntax
  }
}
```

The decision worth defending is using `AsyncLocalStorage` instead of a request-scoped `TenantContext`, which is the more idiomatic-looking Nest answer. A request-scoped provider would give the same data with better type safety and explicit dependencies — genuine advantages. It would also promote `TasksService`, its repository, its controller, and everything else in that chain to request scope, so a single line about correlation ids would change the instantiation strategy of most of the application. `AsyncLocalStorage` keeps every provider a singleton and confines the cost to Node's async context tracking, which is a fraction of rebuilding a dependency subtree per request. The honest downside is that the dependency is *implicit*: nothing in `TasksService`'s constructor reveals that it reads ambient state, and a call made outside the middleware's `run` — a scheduled job, a queue consumer — silently gets `undefined`. That is why `correlationId` returns `'no-context'` rather than throwing: the fallback makes the boundary visible in logs instead of turning a background job into a crash.

---

## 9.9 Questions

**1.** MCQ — The default scope is: a) REQUEST b) TRANSIENT c) DEFAULT d) SINGLETON
<details><summary>Answer</summary>**c)** `Scope.DEFAULT`, which is a singleton. "SINGLETON" is not a Nest scope name.</details>

**2.** MCQ — `TRANSIENT` means a new instance per: a) call b) request c) consumer d) module
<details><summary>Answer</summary>**c)** Per injecting consumer — not per method call.</details>

**3.** MCQ — Injecting a REQUEST-scoped provider into a singleton makes the singleton: a) throw b) request-scoped c) transient d) unchanged
<details><summary>Answer</summary>**b)** Request scope bubbles upward through the whole chain.</details>

**4.** MCQ — Which scope does NOT bubble upward? a) REQUEST b) TRANSIENT c) both d) neither
<details><summary>Answer</summary>**b)** A singleton can inject a transient provider and remain a singleton.</details>

**5.** MCQ — Lifecycle hooks on a request-scoped provider: a) fire per request b) fire once c) never fire d) fire on shutdown
<details><summary>Answer</summary>**c)** Request-scoped classes are not tied to the application lifecycle.</details>

**6.** MCQ — `onApplicationBootstrap` runs: a) before `onModuleInit` b) after all modules initialise c) after `listen()` d) on shutdown
<details><summary>Answer</summary>**b)** After every module has initialised, and before the server listens.</details>

**7.** Short — Why is scope bubbling dangerous in a large application?
<details><summary>Answer</summary>One request-scoped provider low in the graph silently promotes every consumer above it, so a large subtree is reconstructed on every request with no warning and no change visible at those classes.</details>

**8.** Short — Why can't `moduleRef.get()` return a request-scoped provider?
<details><summary>Answer</summary>There is no single instance — one exists per request context — so you must use `await moduleRef.resolve(token, contextId)`, which creates one.</details>

**9.** Short — What problem do durable providers solve?
<details><summary>Answer</summary>Multi-tenant request scope rebuilding the same subtree per request. Durable providers key the subtree on something coarser, usually the tenant, so twenty tenants mean twenty subtrees rather than one per request.</details>

**10.** Short — What is the risk of state on a durable provider?
<details><summary>Answer</summary>It is long-lived and shared by every request for that tenant, so request-specific state on it corrupts across concurrent requests exactly as it would on a singleton.</details>

**11.** Short — What does `enableShutdownHooks()` change?
<details><summary>Answer</summary>It attaches listeners for SIGTERM, SIGINT, SIGBREAK, and SIGHUP so the termination hooks run. Without it, the process dies immediately and no hook executes.</details>

**12.** Output — How many `Logger` instances exist?
```ts
@Injectable({ scope: Scope.TRANSIENT }) class Logger {}
@Injectable() class A { constructor(private l: Logger) {} }
@Injectable() class B { constructor(private l: Logger) {} }
```
<details><summary>Answer</summary>Two — one per consumer. `A` and `B` are still singletons.</details>

**13.** Output — `TenantContext` is REQUEST-scoped and injected by `BillingService`, which `InvoicesController` injects. How many `BillingService` instances exist across 100 requests?
<details><summary>Answer</summary>100. It was promoted to request scope by bubbling, along with the controller.</details>

**14.** Output — A REQUEST-scoped provider implements `onModuleDestroy` closing a file handle. After 1,000 requests, how many handles were closed?
<details><summary>Answer</summary>Zero. Lifecycle hooks never fire for request-scoped classes, so all 1,000 leaked.</details>

**15.** Output — With `enableShutdownHooks()` never called, what does `onApplicationShutdown` log on SIGTERM?
<details><summary>Answer</summary>Nothing — the process exits without running termination hooks.</details>

**16.** Debug — After adding one small request-scoped provider, p99 latency rose sharply across unrelated endpoints. Explain.
<details><summary>Answer</summary>Bubbling. Whatever injects it — and everything above that — is now constructed per request, so endpoints that never touch the new provider still pay for rebuilding the shared subtree they have in common.</details>

**17.** Debug — A transient `ContextLogger` logs the wrong class name after a refactor changed it to default scope. Why?
<details><summary>Answer</summary>With `DEFAULT` scope there is one shared instance, so every consumer's `setContext` overwrites the previous one and the last constructor to run wins.</details>

**18.** Debug — `onModuleInit` in module A calls a service from module B and gets `undefined`. Fix?
<details><summary>Answer</summary>`onModuleInit` only guarantees the host module's own dependencies. Move the work to `onApplicationBootstrap`, which runs after every module has initialised.</details>

**19.** Debug — After upgrading to Nest 11, a module's `onModuleDestroy` now throws because a dependency is already destroyed. Cause?
<details><summary>Answer</summary>Nest 11 reversed the module traversal order within the termination phases. Code that relied on a dependency module still being alive during its own teardown now runs after that module has been destroyed. Move the work to `beforeApplicationShutdown`, or stop depending on another module during teardown.</details>

**20.** Debug — A queue consumer reads `AsyncLocalStorage` context and gets `undefined`, though the same code works in HTTP handlers. Why?
<details><summary>Answer</summary>The store is established by HTTP middleware, and a queue consumer never passes through it. Wrap the consumer's handler in its own `als.run(...)` with a generated correlation id.</details>

**21.** Code — Resolve a request-scoped provider from a singleton without promoting the singleton.
<details><summary>Answer</summary>

```ts
@Injectable()
export class Dispatcher {
  constructor(private readonly moduleRef: ModuleRef) {}

  async handle(payload: Payload) {
    const contextId = ContextIdFactory.create();
    const scoped = await this.moduleRef.resolve(ScopedHandler, contextId);
    return scoped.run(payload);
  }
}
```
`resolve` creates an instance per context id; passing the same id again returns the same instance.
</details>

**22.** Code — Write a `ContextIdStrategy` that shares a DI subtree per tenant, with bounded growth.
<details><summary>Answer</summary>

```ts
export class TenantStrategy implements ContextIdStrategy {
  private readonly tenants = new LRUCache<string, ContextId>({ max: 500 });

  attach(contextId: ContextId, request: Request) {
    const tenantId = request.headers['x-tenant-id'] as string;
    if (!tenantId) return () => contextId;

    let subTree = this.tenants.get(tenantId);
    if (!subTree) {
      subTree = ContextIdFactory.create();
      this.tenants.set(tenantId, subTree);
    }
    return (info: HostComponentInfo) => (info.isTreeDurable ? subTree! : contextId);
  }
}
```
The LRU bound is the part usually omitted; an unbounded `Map` keyed by tenant is a slow memory leak.
</details>

**23.** Code — Give a singleton per-request correlation ids without changing any provider's scope.
<details><summary>Answer</summary>

```ts
@Injectable()
export class RequestContext {
  private readonly als = new AsyncLocalStorage<{ correlationId: string }>();
  run<T>(store: { correlationId: string }, fn: () => T): T { return this.als.run(store, fn); }
  get correlationId(): string { return this.als.getStore()?.correlationId ?? 'no-context'; }
}

@Injectable()
export class ContextMiddleware implements NestMiddleware {
  constructor(private readonly ctx: RequestContext) {}
  use(req: Request, _res: Response, next: NextFunction) {
    this.ctx.run({ correlationId: randomUUID() }, () => next());
  }
}
```
</details>

**24.** Scenario — A teammate proposes making `TenantContext` request-scoped and injecting it into your repository base class, which every repository extends. Evaluate it.
<details><summary>Answer</summary>Reject it on blast radius. A base class every repository extends sits at the bottom of the dependency graph, so request scope would bubble through every repository, every service, and every controller — effectively converting the entire application to per-request instantiation to deliver one header value. The measurable cost is rebuilding the whole object graph on each request; the unmeasurable one is that no class involved shows any sign of having changed. If tenant data is genuinely needed that deep, `AsyncLocalStorage` supplies it without touching scope, and durable providers are the right answer only when you need per-tenant *construction* — a separate connection per tenant — rather than per-tenant data.</details>

**25.** Scenario — Your service must flush a 5,000-item in-memory buffer to a database on shutdown, and the platform sends SIGKILL 30 seconds after SIGTERM. How do you design it?
<details><summary>Answer</summary>Start by enabling shutdown hooks, since none of this runs otherwise. Put the flush in `onApplicationShutdown` so it happens after connections close and no new items can arrive, and bound it with a timeout well under the 30-second window — a flush that outlives SIGKILL loses the data anyway, and losing 5,000 items is better than losing them *and* corrupting a half-written batch. The deeper issue is that an in-memory buffer whose loss matters is the wrong design: shutdown is not the only way a process dies, and SIGKILL, an OOM, or a hardware failure skips hooks entirely. Treat the graceful flush as a best-effort optimisation and make the buffer durable — write-ahead to disk or to a queue — if the data genuinely cannot be lost.</details>

---

## Chapter Summary

- `DEFAULT` is one instance per application, `REQUEST` one per request, and `TRANSIENT` one per *consumer* — not per call.
- Request scope bubbles upward through every consumer, so one provider low in the graph can convert most of the application to per-request instantiation with no visible change at those classes.
- Transient scope does not bubble, which makes it the cheap way to give a provider per-consumer identity.
- Lifecycle hooks never fire on request-scoped classes, so cleanup written there leaks once per request.
- Initialization runs `onModuleInit` then `onApplicationBootstrap`; termination runs `onModuleDestroy`, `beforeApplicationShutdown`, then `onApplicationShutdown` — and none of it happens without `enableShutdownHooks()`.
- For per-request *data* rather than per-request *construction*, `AsyncLocalStorage` delivers the same result while every provider stays a singleton.

---

## Practice Exercises

1. Add a request-scoped provider at the bottom of a dependency chain and log every constructor. Count the instantiations per request before and after.
2. Prove that transient scope does not bubble by injecting a transient provider into a singleton and logging instance identity across requests.
3. Implement a request-scoped provider with `onModuleDestroy` and confirm it never runs.
4. Instrument all five lifecycle hooks, then start and stop the app with and without `enableShutdownHooks()`.
5. Benchmark the same endpoint with a singleton service and with the same service promoted to request scope, and measure the latency difference.

---

## Mini Project: A Multi-Tenant Request Context

Build tenant isolation for an API where every query must be scoped to a tenant.

**Requirements:**

- Tenant id resolved from a header, with requests lacking one rejected before any handler runs.
- Tenant context available in every service without promoting any provider to request scope.
- A durable-provider variant that gives each tenant its own database connection, with LRU-bounded subtrees.
- A test proving two concurrent requests for different tenants never observe each other's context.
- Graceful shutdown that closes every tenant connection exactly once.

---

**Next:** [Chapter 10 — The Request Lifecycle →](10-request-lifecycle.md)
