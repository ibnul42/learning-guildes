# Appendix A — 340 Interview Questions

Grouped by topic, and within each topic by level. Every answer includes a **Follow-up:** line stating what a good interviewer asks next — because the follow-up is usually where interviews are won or lost.

**Levels:** *Junior* — you should know this. *Mid* — you should be able to explain why. *Senior* — you should have an opinion and know the trade-off.

---

## Contents

| Section | Questions |
|---|---|
| [0. How to Answer](#0-how-to-answer) | — |
| [1. Rapid Fire Warm-Up](#1-rapid-fire-warm-up) | 20 |
| [2. Core Concepts and Architecture](#2-core-concepts-and-architecture) | 30 |
| [3. Modules and Dependency Injection](#3-modules-and-dependency-injection) | 40 |
| [4. The Request Pipeline](#4-the-request-pipeline) | 40 |
| [5. Validation and DTOs](#5-validation-and-dtos) | 20 |
| [6. Error Handling](#6-error-handling) | 15 |
| [7. Databases and Transactions](#7-databases-and-transactions) | 30 |
| [8. Authentication and Security](#8-authentication-and-security) | 30 |
| [9. Async: Queues, Events, Microservices](#9-async-queues-events-microservices) | 25 |
| [10. GraphQL and WebSockets](#10-graphql-and-websockets) | 20 |
| [11. Testing](#11-testing) | 25 |
| [12. Performance and Production](#12-performance-and-production) | 25 |
| [13. Architecture and Judgement](#13-architecture-and-judgement) | 20 |

---

## 0. How to Answer

**Technical questions.** Answer in three beats: the direct answer, the mechanism, then one concrete consequence. "Request scope bubbles upward. Anything injecting a request-scoped provider becomes request-scoped too, and so does anything injecting *that*. So one provider low in the graph can make most of the application rebuild its object tree on every request." That is fifteen seconds and it demonstrates you have debugged it, not read about it.

**When you don't know.** Say so, then reason out loud toward an answer. "I haven't used gRPC in Nest specifically, but the pattern is a transport with a `.proto` contract, so I'd expect a decorator equivalent to `@MessagePattern` and a client registered like any other transport." Being able to reason from what you do know is more valuable than the fact you were missing.

**Judgement questions.** These are the ones that separate levels. Commit to a recommendation, then name the condition under which the other choice wins. An answer with no opinion reads as inexperience; an answer with no trade-off reads as dogma.

**The one question to prepare cold.** "Walk me through what happens between the request arriving and your controller method running." It is asked in some form in nearly every NestJS interview. Know the order *and* the reasons.

**What interviewers are actually probing.** Not whether you can name decorators — whether you understand what the framework does at startup, what breaks under concurrency, and what breaks at more than one replica. Those three areas account for most of the questions below.

---

## 1. Rapid Fire Warm-Up

Twenty questions in five minutes. If any are shaky, start at the chapter it points to.

**Q1. What problem does NestJS solve that Express does not?**
<details><summary>Answer</summary>Inversion of control. Express classes import their dependencies, so swapping one for a test means intercepting the module system; Nest classes declare dependencies and a container supplies them. → Chapter 1

**Follow-up:** So what does that actually buy you day to day?  (Testability, and a consistent place for cross-cutting concerns.)</details>

**Q2. What are the three core building blocks?**
<details><summary>Answer</summary>Controllers (HTTP → method calls), providers (anything injectable), and modules (the encapsulation boundary). → Chapter 1

**Follow-up:** Is a guard a provider?  (Yes — "provider" is broader than "service".)</details>

**Q3. What HTTP platform does Nest use by default?**
<details><summary>Answer</summary>Express, via `@nestjs/platform-express`. Fastify is supported as an alternative. → Chapter 1

**Follow-up:** Name something that leaks through that abstraction.</details>

**Q4. What is the minimum Node version for NestJS 11?**
<details><summary>Answer</summary>Node 20. Running on 18 produces obscure native-binding failures rather than a clean version error. → Chapter 1

**Follow-up:** What else changed in Nest 11?  (Express 5, reversed termination hooks, cache-manager v6.)</details>

**Q5. What does `@Injectable()` actually do?**
<details><summary>Answer</summary>Very little directly — it marks the class and, crucially, triggers `emitDecoratorMetadata` so `design:paramtypes` is written. A dependency-free provider works without it. → Chapter 2

**Follow-up:** So why does adding one dependency break a class that has no decorator?</details>

**Q6. Why must a DTO be a class rather than an interface?**
<details><summary>Answer</summary>Interfaces are erased at compile time, so there is nothing at runtime for validation decorators to attach to or for `ValidationPipe` to inspect. → Chapter 2

**Follow-up:** What does `ValidationPipe` do with an interface-typed body?  (Nothing — it passes through.)</details>

**Q7. What is the default provider scope?**
<details><summary>Answer</summary>`Scope.DEFAULT` — a singleton, one instance per application, constructed at startup. → Chapter 9

**Follow-up:** What happens if you store the current user on `this`?</details>

**Q8. Name the request pipeline order.**
<details><summary>Answer</summary>Middleware → guards → interceptors (pre) → pipes → handler → interceptors (post) → filters. → Chapter 10

**Follow-up:** Why do guards run before pipes?</details>

**Q9. What is the difference between `providers` and `exports`?**
<details><summary>Answer</summary>`providers` registers a provider for use inside the module; `exports` makes it injectable by modules that import this one. Read them as private and public. → Chapter 4

**Follow-up:** What happens if you register the same provider in two modules?</details>

**Q10. What status does `POST` return by default?**
<details><summary>Answer</summary>201. Everything else defaults to 200. → Chapter 5

**Follow-up:** When would you override it?  (Login, search-with-a-body — any action that creates nothing.)</details>

**Q11. Is `@Param('id') id: number` a number?**
<details><summary>Answer</summary>No — a string. The type annotation is erased and enforces nothing; use `ParseIntPipe`. → Chapter 5

**Follow-up:** What about `@Query()` with a DTO and `transform: true`?</details>

**Q12. What does returning `false` from a guard produce?**
<details><summary>Answer</summary>403 Forbidden, always. For a missing credential you want 401, which requires throwing `UnauthorizedException`. → Chapter 12

**Follow-up:** Why does the distinction matter to a client?</details>

**Q13. Which pipeline components can see the response?**
<details><summary>Answer</summary>Only interceptors and exception filters. → Chapter 10

**Follow-up:** So where does a response envelope belong?</details>

**Q14. What does `throw new Error('not found')` return to the client?**
<details><summary>Answer</summary>500, with the message withheld. Only `HttpException` subclasses carry a status. → Chapter 15

**Follow-up:** Why is the message deliberately hidden?</details>

**Q15. What is `synchronize: true` and when should you use it?**
<details><summary>Answer</summary>A TypeORM option that alters the schema to match your entities, including dropping columns. Acceptable only in a disposable local database. → Chapter 17

**Follow-up:** What do you use instead?</details>

**Q16. What is the N+1 problem?**
<details><summary>Answer</summary>Fetching a collection then querying once per row for a relation — 101 queries for 100 rows. → Chapter 17

**Follow-up:** How would you detect one?</details>

**Q17. What does `enableShutdownHooks()` do?**
<details><summary>Answer</summary>Attaches listeners for SIGTERM, SIGINT, SIGBREAK, and SIGHUP so termination lifecycle hooks run. Without it they never fire. → Chapter 9

**Follow-up:** What does that break in a rolling deploy?</details>

**Q18. What is the difference between `useClass` and `useExisting`?**
<details><summary>Answer</summary>`useClass` constructs a new instance per token; `useExisting` is an alias to an existing one. → Chapter 8

**Follow-up:** What goes wrong if you use `useClass` for a deprecation alias?</details>

**Q19. Where should business logic live?**
<details><summary>Answer</summary>In services. The test is whether the logic would still be needed if the operation were triggered by a queue message or a CLI command. → Chapter 6

**Follow-up:** Give an example of logic that legitimately belongs in a controller.</details>

**Q20. Is NestJS faster than Express?**
<details><summary>Answer</summary>No. It adds a thin pipeline on top of Express or Fastify. Its benefits are structural — testability, consistency, encapsulation — not throughput. → Chapter 1

**Follow-up:** So when would you not use it?</details>

---

## 2. Core Concepts and Architecture

**Q21.** *Junior* — What is a decorator, mechanically?
<details><summary>Answer</summary>A function that receives what it decorates and usually records metadata about it. `@Controller('users')` is a function call whose result is applied to the class. In Nest, decorators almost never change behaviour directly — they record intent that the framework reads at startup. → Chapter 2

**Follow-up:** So what happens to `@Get()` on a controller no module declares?  (Nothing — inert metadata, no error.)</details>

**Q22.** *Mid* — How does the container know what to inject into a constructor?
<details><summary>Answer</summary>The TypeScript compiler emits the parameter types as `design:paramtypes` metadata on the decorated class — an array of the actual constructor functions. The container reads that array and looks each one up as a token. → Chapter 2

**Follow-up:** What is emitted for a parameter typed as an interface?  (`Object`.)</details>

**Q23.** *Mid* — Why does Nest require legacy decorators rather than the TC39 standard ones?
<details><summary>Answer</summary>The standard proposal has no parameter decorators, so `@Body()`, `@Param()`, and `@Inject()` could not exist. `experimentalDecorators` and `emitDecoratorMetadata` are both required. → Chapter 2

**Follow-up:** Does the ESM migration in v12 change that?  (No — module system and decorators are independent.)</details>

**Q24.** *Junior* — What happens between `NestFactory.create()` and `listen()`?
<details><summary>Answer</summary>`create()` scans the module graph, resolves dependencies, instantiates every default-scoped provider, runs `onModuleInit` and `onApplicationBootstrap`, and registers routes. `listen()` only binds the socket. → Chapter 3

**Follow-up:** Why can tests run an app without a port?</details>

**Q25.** *Mid* — Why does a dependency mistake crash at boot rather than on the first request?
<details><summary>Answer</summary>Resolution and instantiation happen eagerly during `create()`, before any request exists. → Chapter 3

**Follow-up:** Is that better or worse than lazy resolution?  (Better — a broken deploy never starts serving.)</details>

**Q26.** *Junior* — What does the Nest CLI's `generate` give you that copying a file does not?
<details><summary>Answer</summary>Registration. It adds the class to the nearest module's `providers` or `controllers` array, which is the step most commonly forgotten and the usual cause of a resolution error. → Chapter 3

**Follow-up:** What does `--dry-run` do?</details>

**Q27.** *Mid* — What is a hybrid application?
<details><summary>Answer</summary>An application serving HTTP *and* one or more microservice transports, connected with `connectMicroservice()` and started with `startAllMicroservices()` before `listen()`. → Chapter 26

**Follow-up:** What happens if you call `listen()` first?</details>

**Q28.** *Senior* — When is NestJS the wrong choice?
<details><summary>Answer</summary>A single-purpose serverless function, where booting the whole module graph is real cold-start latency for one endpoint. A throwaway prototype, where the structure is pure overhead. A team avoiding TypeScript, since the container works by reading emitted type metadata. And a service that is one hot path, where the pipeline costs more than it returns. → Chapter 1

**Follow-up:** What would you use instead for the webhook receiver case?</details>

**Q29.** *Mid* — What leaks through the platform adapter abstraction?
<details><summary>Answer</summary>The raw request and response types, middleware signatures, file-upload handling, and third-party middleware compatibility. → Chapter 1

**Follow-up:** Why does that argue for choosing the adapter on day one?</details>

**Q30.** *Junior* — Why type the app as `NestExpressApplication`?
<details><summary>Answer</summary>`NestFactory.create()` otherwise returns `INestApplication`, which does not declare platform-specific methods like `useStaticAssets()` or `set('trust proxy', …)`. → Chapter 1

**Follow-up:** What do people do instead when they hit that?  (Cast to `any` — which is the wrong fix.)</details>

**Q31.** *Mid* — What changed in NestJS 11 that breaks upgrades?
<details><summary>Answer</summary>Node 20 minimum, Express 5 by default (named wildcards and brace-syntax optional params), reversed termination hook traversal order, and `cache-manager` v6 switching TTL units from seconds to milliseconds. → Chapter 36

**Follow-up:** Which of those fails loudest?  (Express 5 — startup throws from `path-to-regexp`.)</details>

**Q32.** *Senior* — What are the four pillars of NestJS 12?
<details><summary>Answer</summary>Full ESM across official packages, Standard Schema support in route decorators, Vitest as the default runner for new ESM projects, and Rspack plus oxlint replacing Webpack and ESLint. → Chapter 36

**Follow-up:** Which one costs the most to migrate?  (ESM, by a wide margin.)</details>

**Q33.** *Mid* — What breaks when you move to ESM?
<details><summary>Answer</summary>`require`, `__dirname`, `__filename`, and extensionless relative imports — the last failing at runtime rather than compile time. Some dependencies are CJS-only, so named imports may not resolve. → Chapter 36

**Follow-up:** What can you do now, on v11, to reduce that?</details>

**Q34.** *Junior* — What is `bufferLogs` for?
<details><summary>Answer</summary>It holds log output produced during bootstrap until a custom logger is attached with `useLogger`, so startup messages are not lost or emitted in the wrong format. → Chapter 3

**Follow-up:** Why does that matter?  (Startup output is often where a broken deploy explains itself.)</details>

**Q35.** *Mid* — What is the difference between `@Global()` and a shared module?
<details><summary>Answer</summary>A shared module is imported explicitly wherever needed, so a module's dependencies are visible in its own `imports`. `@Global()` makes exports available everywhere, which is less typing and removes that visibility — and means testing modules silently miss providers production supplies. → Chapter 4

**Follow-up:** What is a legitimate use of `@Global()`?</details>

**Q36.** *Senior* — How do you decide where module boundaries go?
<details><summary>Answer</summary>Around bounded concepts rather than entities. Two tests: can you state the module's job in one sentence without "and", and does a typical feature change touch one module rather than five. One module per entity produces boundaries where everything imports everything. → Chapter 35

**Follow-up:** What does a cycle between two modules tell you?</details>

**Q37.** *Mid* — What does `forFeature` conventionally mean?
<details><summary>Answer</summary>Registering a scoped slice of a module that has already been configured at the root — one entity, one queue, one namespace. → Chapter 4

**Follow-up:** What does TypeORM's `forFeature` give you that Prisma does not?  (A real module boundary.)</details>

**Q38.** *Mid* — What is a dynamic module and when do you need one?
<details><summary>Answer</summary>A module configured by its consumer, via a static method returning a `DynamicModule`. Needed when a consumer must vary behaviour the module does not own — a bucket name, a transport, a connection string. → Chapter 4

**Follow-up:** Should every module have a `forRoot()`?  (No — most have one consumer and nothing to vary.)</details>

**Q39.** *Junior* — What does `ConfigurableModuleBuilder` generate?
<details><summary>Answer</summary>Both the sync and async configuration methods — `forRoot`/`forRootAsync` or `register`/`registerAsync` — plus the options injection token. → Chapter 4

**Follow-up:** Why is the async variant the one worth generating?  (It is easy to get subtly wrong by hand.)</details>

**Q40.** *Mid* — Why is layered architecture the right default?
<details><summary>Answer</summary>Each layer depends only on the one below, which gives testability and a predictable place to look. It degrades only when folders are grouped by *layer* rather than by feature, which makes every change span directories. → Chapter 35

**Follow-up:** When would you deviate?</details>

**Q41.** *Senior* — What does hexagonal architecture buy, and what does it cost?
<details><summary>Answer</summary>Buys: business rules testable with no framework or database, swappable infrastructure, and rules living on entities rather than scattered across services. Costs: roughly twice the files, a mapping layer to maintain, and slower navigation. Worth it where the domain has real invariants; ceremony for CRUD. → Chapter 35

**Follow-up:** What is the most common half-adoption?  (A domain class carrying an ORM decorator.)</details>

**Q42.** *Senior* — When is a monorepo justified?
<details><summary>Answer</summary>When several deployables genuinely share code — an API and a worker sharing modules and entities is the canonical case — so changes can be atomic across them. Its failure mode is a `common` library that imports from the features that import it. → Chapter 35

**Follow-up:** How do you stop that?  (Lint rules enforcing import direction.)</details>

**Q43.** *Mid* — What is the difference between a provider and a service?
<details><summary>Answer</summary>"Service" is a naming convention for the most common kind of provider. Providers also include repositories, guards, factory results, and plain values via `useValue`. → Chapter 6

**Follow-up:** Why does the distinction matter?  (It makes `useValue` and `useFactory` unremarkable rather than exotic.)</details>

**Q44.** *Junior* — Why declare injected dependencies `private readonly`?
<details><summary>Answer</summary>`readonly` prevents a method from reassigning a dependency at runtime, which is a real class of bug in long-lived singletons, and it costs nothing. → Chapter 6

**Follow-up:** When is property injection acceptable?  (A base class subclasses inherit from.)</details>

**Q45.** *Senior* — How do you know a service has grown too large?
<details><summary>Answer</summary>By reasons to change, not line count. A `UsersService` doing authentication, profile updates, avatar uploads, and billing has four unrelated reasons to change. A 400-line service doing one thing thoroughly is fine. More than five constructor dependencies is the reliable warning sign. → Chapter 6

**Follow-up:** What is the wrong split?  (One service per entity, always.)</details>

**Q46.** *Mid* — What is the repository seam and when is it worth it?
<details><summary>Answer</summary>Depending on an abstract repository rather than the ORM, so a test can supply a fake with no database. Worth it when the service has rules worth testing in isolation; not worth it when the service only forwards calls to the ORM. → Chapter 6

**Follow-up:** So do you always introduce it up front?  (No — the refactor is mechanical when you need it.)</details>

**Q47.** *Mid* — Why should a service not inject another module's repository?
<details><summary>Answer</summary>It bypasses the owning service's rules and invariants, so validation and limits enforced there are silently skipped. Go through the module's public surface. → Chapter 6

**Follow-up:** What if that service does not expose what you need?</details>

**Q48.** *Junior* — What is `ModuleRef` and when do you use it?
<details><summary>Answer</summary>A service locator giving programmatic access to the container: `get()` for singletons, `resolve()` for scoped providers, `create()` for unregistered classes. Use it only when the token is genuinely dynamic. → Chapter 7

**Follow-up:** Why is it a last resort?  (It hides dependencies — a constructor taking `ModuleRef` tells you nothing.)</details>

**Q49.** *Senior* — Why prefer an abstract class to an interface for a contract?
<details><summary>Answer</summary>An abstract class compiles to a real function, so it serves as both the type contract and the DI token, while still being implementation-free. An interface needs a separate string or symbol token. → Chapter 2

**Follow-up:** What do you lose?  (Nothing meaningful — it is strictly better for injectable contracts.)</details>

**Q50.** *Mid* — What is `AsyncLocalStorage` used for in a Nest application?
<details><summary>Answer</summary>Carrying per-request context — correlation ids, tenant, principal — through the whole async call chain without changing any provider's scope. It is what most production services use instead of request-scoped providers. → Chapter 9

**Follow-up:** What is its downside?  (The dependency is implicit — nothing in the constructor reveals it.)</details>

---

## 3. Modules and Dependency Injection

**Q51.** *Junior* — What are the four arrays in `@Module()`?
<details><summary>Answer</summary>`imports` (modules whose exports this may use), `controllers` (instantiated, routes registered), `providers` (instantiated, injectable within this module), `exports` (made available to importing modules). → Chapter 4

**Follow-up:** Which one is the public surface?</details>

**Q52.** *Mid* — What happens if you list the same provider in two modules' `providers`?
<details><summary>Answer</summary>Two independent instances, each with its own state. Caches diverge, counters disagree, connection pools double — and the symptom appears far from the cause. → Chapter 4

**Follow-up:** Why does the error message push people toward this mistake?</details>

**Q53.** *Junior* — How many times is a module instantiated if fifteen modules import it?
<details><summary>Answer</summary>Once. Modules are singletons per application, so importing declares visibility rather than requesting a copy. → Chapter 4

**Follow-up:** So what is the cost of importing a module?  (Effectively nothing.)</details>

**Q54.** *Mid* — What does re-exporting a module do?
<details><summary>Answer</summary>Passes through whatever *that* module exports, not its private providers — so encapsulation still holds one level down. It is how you build a curated bundle. → Chapter 4

**Follow-up:** How is that different from `@Global()`?</details>

**Q55.** *Mid* — Walk through the container's resolution algorithm.
<details><summary>Answer</summary>For a token needed inside module M: check M's own `providers`, then the `exports` of every module M imports (and transitively what those re-export), then any `@Global()` module's exports, then fail at startup. → Chapter 7

**Follow-up:** Is a provider two hops away reachable?  (No — reachability is not visibility.)</details>

**Q56.** *Mid* — Read this error: `Nest can't resolve dependencies of the TasksService (?, NotificationsService)… argument TaskRepository at index [0]… in the TasksModule context.`
<details><summary>Answer</summary>`TasksService` is being constructed; the `?` marks the parameter that failed; index [0] locates it; `TaskRepository` is the token searched for; `TasksModule` is the module whose visibility was searched. So either the owning module does not export it, or `TasksModule` does not import that module. → Chapter 7

**Follow-up:** What does `(?, ?, ?)` mean instead?  (No metadata at all — a missing `@Injectable()`.)</details>

**Q57.** *Senior* — Why is "add it to the consumer's providers" the wrong fix for that error?
<details><summary>Answer</summary>It resolves and creates a second instance. Any provider holding a cache, counter, pool, or circuit-breaker state becomes inconsistent depending on which module the caller lives in — and the error, which was pointing at a real boundary problem, disappears. → Chapter 7

**Follow-up:** How would you catch this in review?</details>

**Q58.** *Junior* — What is a DI token?
<details><summary>Answer</summary>The key a provider is registered and looked up under: a class, a string, or a symbol. `providers: [X]` is shorthand for `{ provide: X, useClass: X }`. → Chapter 7

**Follow-up:** Why is seeing that expansion useful?</details>

**Q59.** *Mid* — Why prefer symbol tokens to string tokens?
<details><summary>Answer</summary>Strings collide. Two libraries both registering `'CONFIG'` silently overwrite each other, and which wins depends on module resolution order. A `Symbol` is unique by construction. → Chapter 7

**Follow-up:** How would that failure present?</details>

**Q60.** *Mid* — What is `@Optional()` for, and when is it misused?
<details><summary>Answer</summary>It makes a missing dependency `undefined` rather than a startup failure — appropriate for genuinely optional collaborators like telemetry. Misused, it silences a resolution error you have not diagnosed, converting a loud startup failure into a silent runtime one. → Chapter 7

**Follow-up:** What does the crash look like then?</details>

**Q61.** *Mid* — Explain `useExisting` versus `useClass` with a concrete failure.
<details><summary>Answer</summary>`useExisting` aliases one instance under a second token; `useClass` constructs a new one. Using `useClass` for a deprecation alias yields two gateway instances — two connection pools, two circuit-breaker states, and a rate limiter counting half the traffic. The bug only appears under load, in whichever half of the code uses the old token. → Chapter 7

**Follow-up:** How would you migrate off a deprecated token safely?</details>

**Q62.** *Mid* — Why does `useValue` fail for anything derived from configuration?
<details><summary>Answer</summary>It is evaluated when the module is *defined* — at import time, before `ConfigModule` has loaded a `.env` file. The value is silently `undefined`. → Chapter 8

**Follow-up:** What is the correct recipe?</details>

**Q63.** *Mid* — How does the `inject` array map to a factory's parameters?
<details><summary>Answer</summary>Positionally. There is no name matching, so transposing two entries silently passes the wrong object — and if both are objects, it may not throw until a specific field is touched. → Chapter 8

**Follow-up:** How do you make one dependency optional?</details>

**Q64.** *Senior* — What is the risk of an async provider, and how do you bound it?
<details><summary>Answer</summary>Bootstrap awaits it, so an unreachable dependency hangs startup indefinitely — no error, no log, no listening port, and health checks report nothing because the server never started. Bound it with `Promise.race` against a timeout. → Chapter 8

**Follow-up:** What does that failure look like in a deployment?</details>

**Q65.** *Mid* — What are the `APP_*` tokens and why do they matter?
<details><summary>Answer</summary>`APP_GUARD`, `APP_PIPE`, `APP_FILTER`, `APP_INTERCEPTOR` register global pipeline components as providers, so the container constructs them with dependencies injected — and they appear in testing modules. `main.ts` registration can do neither. → Chapter 8

**Follow-up:** What happens if you register `APP_GUARD` three times?  (All three run, in order.)</details>

**Q66.** *Junior* — How do you inject a non-class token?
<details><summary>Answer</summary>With `@Inject(TOKEN)`. String and symbol tokens have no type metadata, so the annotation alone cannot identify them. → Chapter 8

**Follow-up:** What error do you get if you forget?</details>

**Q67.** *Mid* — What is scope bubbling and why is it dangerous?
<details><summary>Answer</summary>Request scope propagates upward: anything injecting a request-scoped provider becomes request-scoped, and so does anything injecting that, up to the controller. Nothing warns you, and the classes involved show no sign of having changed. One provider low in the graph can convert most of the application to per-request instantiation. → Chapter 9

**Follow-up:** Does transient scope bubble?  (No.)</details>

**Q68.** *Mid* — What does `TRANSIENT` actually mean?
<details><summary>Answer</summary>One instance per injecting *consumer*, created when that consumer is created — not one per method call. A singleton injecting a transient provider gets one dedicated instance for the application's lifetime. → Chapter 9

**Follow-up:** What is it useful for?  (A provider that needs to know who injected it.)</details>

**Q69.** *Senior* — Do lifecycle hooks fire on request-scoped providers?
<details><summary>Answer</summary>No. The docs are explicit: request-scoped classes are not tied to the application lifecycle, are created per request, and are garbage-collected after the response. So an `onModuleDestroy` closing a handle never runs, and the handle leaks once per request. → Chapter 9

**Follow-up:** Where would you put that cleanup instead?</details>

**Q70.** *Mid* — Why does `moduleRef.get()` throw for a scoped provider?
<details><summary>Answer</summary>There is no single instance to return — one exists per context. Use `await moduleRef.resolve(token, contextId)`, which creates one tied to a context identifier. → Chapter 9

**Follow-up:** What does passing the same context id twice give you?</details>

**Q71.** *Senior* — What problem do durable providers solve?
<details><summary>Answer</summary>Multi-tenant request scope rebuilding the same dependency subtree on every request. A `ContextIdStrategy` keys the subtree on something coarser — usually the tenant — so twenty tenants mean twenty subtrees rather than one per request. → Chapter 9

**Follow-up:** What is the catch?  (They outlive the request, so the stateless rule applies again.)</details>

**Q72.** *Mid* — Name the initialization and termination hooks in order.
<details><summary>Answer</summary>Init: `onModuleInit`, then `onApplicationBootstrap`. Termination: `onModuleDestroy`, `beforeApplicationShutdown`, `onApplicationShutdown`. → Chapter 9

**Follow-up:** What can you rely on in `onModuleInit` that you cannot in the constructor?</details>

**Q73.** *Mid* — Why can't `onModuleInit` depend on another module?
<details><summary>Answer</summary>It runs once the *host* module's own dependencies are resolved, which says nothing about other modules. Cross-module work belongs in `onApplicationBootstrap`, which runs after every module has initialised. → Chapter 9

**Follow-up:** What does that failure look like?  (An injected service that is undefined.)</details>

**Q74.** *Senior* — What changed about termination hooks in Nest 11?
<details><summary>Answer</summary>The order modules are traversed within each termination phase was reversed. The hook sequence is unchanged, but code that relied on a dependency module still being alive during its own teardown can now find it already destroyed. → Chapter 9

**Follow-up:** How would you fix such a case?</details>

**Q75.** *Mid* — How do you resolve a circular dependency between two modules?
<details><summary>Answer</summary>`forwardRef(() => OtherModule)` in both modules' `imports`, and `@Inject(forwardRef(() => OtherService))` at both injection sites. Fixing only one side leaves the cycle. → Chapter 7

**Follow-up:** Should you be happy with that fix?</details>

**Q76.** *Senior* — What does a module cycle usually indicate?
<details><summary>Answer</summary>A third concept that belongs in its own module, or logic sitting on the wrong side of a boundary. The usual fixes are extracting the shared concern, inverting one direction with an event, or moving the method. `forwardRef` keeps the application booting long after the structure stopped making sense. → Chapter 7

**Follow-up:** When is `forwardRef` genuinely fine?</details>

**Q77.** *Mid* — What does `{ strict: false }` on `moduleRef.get()` do?
<details><summary>Answer</summary>Searches the whole application rather than the current module, deliberately bypassing encapsulation. It should be a marked exception, not a habit. → Chapter 7

**Follow-up:** What is the alternative?</details>

**Q78.** *Junior* — Why should a module class body be empty?
<details><summary>Answer</summary>A module is configuration, not behaviour. Logic there is hard to test and runs at construction time, which is not a lifecycle point you should depend on. → Chapter 4

**Follow-up:** What is the one thing a module class legitimately implements?  (`NestModule`, for `configure()`.)</details>

**Q79.** *Mid* — What is the difference between `register` and `forRoot` by convention?
<details><summary>Answer</summary>`forRoot` configures once at the application root, implying global state; `register` configures per-import with no such implication. Both have `Async` variants for options that come from injected providers. → Chapter 4

**Follow-up:** Which would you use for a JWT module?</details>

**Q80.** *Senior* — How would you detect a provider registered in more than one module, automatically?
<details><summary>Answer</summary>Inject `ModulesContainer` and walk the resolved graph on `onApplicationBootstrap`, collecting tokens per module and flagging any that appear more than once. It is a fact about the built application rather than a lint heuristic, so it catches custom and dynamic providers too. → Chapter 7

**Follow-up:** Would you fail the boot on it, or warn?</details>

**Q81.** *Mid* — What is `ContextIdFactory` for?
<details><summary>Answer</summary>Creating context identifiers used to resolve request-scoped providers programmatically, and the hook point for durable providers via `ContextIdFactory.apply()`. → Chapter 9

**Follow-up:** Where would a singleton need one?</details>

**Q82.** *Junior* — Can you inject a provider into a `useFactory`?
<details><summary>Answer</summary>Yes, through the `inject` array, which supplies the factory's parameters positionally. → Chapter 8

**Follow-up:** Can you inject into `useValue`?  (No — it is an already-built object.)</details>

**Q83.** *Mid* — Why is a factory a better place for an environment branch than `useClass`?
<details><summary>Answer</summary>A `useClass` ternary is evaluated at import time and is invisible from the consumer, so a misspelled variable ships a test double to production with nothing failing. A factory keeps the decision in one readable place with its dependencies declared, and can throw on the unsafe branch. → Chapter 8

**Follow-up:** How should that branch be written?  (An allowlist of safe environments, failing closed.)</details>

**Q84.** *Senior* — Your team keeps hitting resolution errors and has started adding providers everywhere. What do you tell them?
<details><summary>Answer</summary>The errors have stopped and the bugs have started — each duplicate registration is a separate instance. Walk them through the error anatomy instead: the token, the index, and the context module tell you whether the fix is an `exports` entry or an `imports` entry. A provider is registered exactly once, in the module that owns it, and shared by exporting. → Chapter 7

**Follow-up:** How would you make that mistake visible in CI?</details>

**Q85.** *Mid* — What does `@Global()` still require?
<details><summary>Answer</summary>`exports`, and one import somewhere — usually `AppModule`. It widens visibility of a module's exports; a global module with nothing exported has nothing to see, and one nobody imports is not in the graph at all. → Chapter 4

**Follow-up:** Why does that surprise people?</details>

**Q86.** *Senior* — What is the cost of `@Global()` in tests?
<details><summary>Answer</summary>Production gets the provider implicitly, while `Test.createTestingModule` builds a fresh graph that never imported it — so a provider available everywhere in production is missing in isolation, and the test fails for a reason unrelated to what it is testing. → Chapter 4

**Follow-up:** How do you fix that in the test?</details>

**Q87.** *Mid* — Should every module with options be dynamic?
<details><summary>Answer</summary>No. A dynamic module exists to let a consumer vary behaviour it does not own. Most modules have one consumer and nothing to vary, so `forRoot()` adds an options object, a token, and an async variant to maintain in exchange for indirection. → Chapter 4

**Follow-up:** What information do you lose if everything has one?</details>

**Q88.** *Mid* — What does `setExtras` do on `ConfigurableModuleBuilder`?
<details><summary>Answer</summary>Adds extra options — such as `isGlobal` — with a callback that transforms the generated module definition, so the consumer decides rather than the module author. → Chapter 4

**Follow-up:** Why let the consumer decide global-ness?</details>

**Q89.** *Senior* — How would you structure a library other teams install?
<details><summary>Answer</summary>Export the smallest surface that satisfies the use case — usually one service. Keep transports, renderers, and caches private, because every export is a commitment: once a consumer injects an internal, swapping its implementation becomes a breaking change for code you cannot see. Shrinking a surface later requires finding every consumer; growing it requires adding a line. → Chapter 4

**Follow-up:** What would make you export more?</details>

**Q90.** *Senior* — A base class every repository extends needs the tenant. Would you make it request-scoped?
<details><summary>Answer</summary>No — that base class sits at the bottom of the dependency graph, so request scope would bubble through every repository, service, and controller, converting the whole application to per-request instantiation to deliver one header value. Use `AsyncLocalStorage`. Durable providers are the right answer only if you need per-tenant *construction*, such as a separate connection. → Chapter 9

**Follow-up:** How would you measure the difference?</details>

---

## 4. The Request Pipeline

The most heavily interviewed section. Q91 is asked in some form in nearly every NestJS interview.

**Q91.** *Junior* — Walk me through what happens between the request arriving and your controller method running.
<details><summary>Answer</summary>Middleware first (global, then module-bound). Then guards, global → controller → route. Then interceptors' pre-half, same order. Then pipes, global → controller → route → parameter. Then the handler. On the way out, interceptors unwind route → global. If anything throws, exception filters take over, most specific first. → Chapter 10

**Follow-up:** Why is that the order — specifically, why guards before pipes?</details>

**Q92.** *Mid* — Why do guards run before pipes?
<details><summary>Answer</summary>Authorization is the cheapest rejection. There is no reason to parse and validate a body for a caller who is not permitted to reach the handler at all. Pipes are the most expensive per-request step and the most specific, so they run last. → Chapter 10

**Follow-up:** Why do interceptors run before pipes rather than after?</details>

**Q93.** *Mid* — Why do interceptors run before pipes?
<details><summary>Answer</summary>An interceptor wraps the whole operation, including validation — so a timeout must cover the time spent validating, and a logging interceptor should record requests that fail validation. → Chapter 10

**Follow-up:** What does that mean for a caching interceptor?</details>

**Q94.** *Mid* — Why does the post-interceptor order reverse?
<details><summary>Answer</summary>Interceptors nest around the handler, so they unwind in the opposite order to the one they entered. A global interceptor is the outermost layer: it sees the request first and the response last. → Chapter 10

**Follow-up:** Where should a metrics interceptor sit, and why?</details>

**Q95.** *Senior* — Where should a metrics interceptor be registered relative to a timeout interceptor?
<details><summary>Answer</summary>Outermost — registered first. As the outer layer it observes what the client experiences, including the `GatewayTimeoutException` the timeout raises. Placed innermost, a request that times out records nothing at all, so the very requests you most need to see are missing from the histogram. → Chapter 13

**Follow-up:** What is the intuitive-but-wrong instinct here?  ("Measure only my code.")</details>

**Q96.** *Junior* — What can middleware do that a guard cannot, and vice versa?
<details><summary>Answer</summary>Middleware runs on the raw platform request before route resolution, so it can work with the response stream and third-party Express packages — but it has no `ExecutionContext` and cannot know the target handler. A guard knows the class and handler, so it can read decorator metadata. → Chapter 11

**Follow-up:** Which of those makes authentication a guard's job?</details>

**Q97.** *Mid* — Why is authentication in middleware a mistake?
<details><summary>Answer</summary>Middleware cannot read route metadata, so `@Public()` and role decorators are invisible to it. You end up maintaining a path allowlist in a different file from the routes, and it drifts — the failure being an endpoint either unreachable or unintentionally open. → Chapter 11

**Follow-up:** Someone argues middleware is faster. Is it?</details>

**Q98.** *Junior* — What happens if middleware neither responds nor calls `next()`?
<details><summary>Answer</summary>The request hangs until the client times out. No error is raised and nothing is logged — it is the most common middleware bug. → Chapter 11

**Follow-up:** Is ending the response without `next()` legitimate?  (Yes — that is how middleware short-circuits.)</details>

**Q99.** *Mid* — When must middleware be a class rather than a function?
<details><summary>Answer</summary>When it needs an injected dependency or a lifecycle hook. Functional middleware is not constructed by the container, so it cannot reach `ConfigService` or a logger provider. → Chapter 11

**Follow-up:** How is middleware bound at all?  (Only through `configure()` — there is no decorator.)</details>

**Q100.** *Mid* — Why bind middleware to a controller class rather than a path string?
<details><summary>Answer</summary>The binding follows the controller if its route prefix changes; a string path silently stops matching. → Chapter 11

**Follow-up:** What determines middleware order across modules?</details>

**Q101.** *Senior* — Give a case where middleware is genuinely the right tool.
<details><summary>Answer</summary>Raw-body capture for webhook signature verification. The signature is computed over the exact bytes sent, and by the time a pipe runs the body has been parsed and re-serialised, changing key order and whitespace — so the signature can never verify. Also: correlation-id context, third-party Express packages, and anything wrapping the response stream such as compression. → Chapter 11

**Follow-up:** Where do you establish `AsyncLocalStorage` context, and what is the trap?</details>

**Q102.** *Senior* — What is the trap when establishing `AsyncLocalStorage` in middleware?
<details><summary>Answer</summary>`next()` must be called *inside* `als.run()`. Calling `run` and then `next()` outside it establishes a store that has already exited by the time any handler executes, so every downstream `getStore()` returns `undefined`. With a fallback value it presents as logs that all read "no-context" rather than as an error. → Chapter 11

**Follow-up:** Why is that hard to catch in review?</details>

**Q103.** *Junior* — What does `ExecutionContext` give you?
<details><summary>Answer</summary>`getClass()` and `getHandler()` — the metadata anchors passed to `Reflector` — plus `getType()` and the transport switches `switchToHttp()`, `switchToWs()`, `switchToRpc()`. → Chapter 12

**Follow-up:** What breaks if a global guard calls `switchToHttp()` unconditionally?</details>

**Q104.** *Mid* — A global guard using `switchToHttp()` breaks when a WebSocket gateway is added. Why, and what is the fix?
<details><summary>Answer</summary>A WS context has no HTTP request, so `switchToHttp().getRequest()` yields nothing. Branch on `context.getType()` and read the handshake for `ws`, the GraphQL context for `graphql`, and the request otherwise. → Chapter 12

**Follow-up:** Why does this only surface later?  (The guard is written when the app is HTTP-only.)</details>

**Q105.** *Mid* — Explain `getAllAndOverride` versus `getAllAndMerge`.
<details><summary>Answer</summary>`getAllAndOverride` returns the first non-undefined value from the targets array, so passing `[getHandler(), getClass()]` lets route metadata override the controller's. `getAllAndMerge` combines all values into one array. The choice decides whether a route can relax a requirement its controller declared, which makes it a security decision. → Chapter 12

**Follow-up:** With a `.some()` role check, does merging tighten the requirement?  (No — it still relaxes.)</details>

**Q106.** *Mid* — How do you implement `@Public()` on a globally-guarded API?
<details><summary>Answer</summary>`SetMetadata(IS_PUBLIC_KEY, true)` via a decorator, and the guard reads it with `getAllAndOverride` across handler and class, returning `true` early. → Chapter 12

**Follow-up:** Why authenticate globally and opt out, rather than opting in?</details>

**Q107.** *Senior* — Why is secure-by-default the right way round for authentication?
<details><summary>Answer</summary>The two mistakes have very different costs. With an opt-in guard, forgetting `@UseGuards()` leaves an endpoint open and nothing fails. With a global guard, forgetting `@Public()` makes an endpoint return 401 — visible, immediate, and harmless. → Chapter 12

**Follow-up:** How do you register a global guard that needs `Reflector`?</details>

**Q108.** *Mid* — Why must `RolesGuard` verify `request.user` exists rather than assuming it?
<details><summary>Answer</summary>The ordering invariant lives in one providers array and can be broken by a reorder, by `@UseGuards(RolesGuard)` applied directly to a route, or by a transport where the auth guard's path does not populate the same object. Reading `user.roles` on `undefined` throws a `TypeError` → 500, which is a crash rather than a denial — and 500s are retried and sometimes routed past authorization upstream. → Chapter 12

**Follow-up:** So what should it do instead?  (Throw `UnauthorizedException` — fail closed.)</details>

**Q109.** *Junior* — What is the difference between returning `false` and throwing from a guard?
<details><summary>Answer</summary>`false` always produces 403. Throwing lets you choose — `UnauthorizedException` for a missing credential, which is 401, with a usable message. → Chapter 12

**Follow-up:** Why does a client care?  (401 means authenticate and retry; 403 means do not bother.)</details>

**Q110.** *Mid* — What does an interceptor's `CallHandler` do?
<details><summary>Answer</summary>`next.handle()` invokes the route handler and returns an Observable emitting its return value. Everything before that call runs on the way in; everything piped onto it runs on the way out. → Chapter 13

**Follow-up:** What happens if you never call it?</details>

**Q111.** *Mid* — What happens if an interceptor returns without calling `next.handle()`?
<details><summary>Answer</summary>The handler is skipped entirely — and so are the pipes, since interceptors run first. That is how caching works, and it is why a cache key is built from unvalidated input. → Chapter 13

**Follow-up:** What is the risk that creates?</details>

**Q112.** *Senior* — Why does `tap(fn)` in a metrics interceptor produce dashboards that lie?
<details><summary>Answer</summary>`tap` with a single function subscribes only to the next notification, so failed requests are never recorded. Latency percentiles are computed from requests that did not fail, and the error count is zero during an outage. Use `tap({ next, error })` or `finalize`. → Chapter 13

**Follow-up:** Why is this failure mode particularly dangerous?  (The dashboard looks healthy while users are not served.)</details>

**Q113.** *Mid* — Which RxJS operators do you actually need for interceptors?
<details><summary>Answer</summary>`map` for transforming the value, `tap` for side effects, `catchError` for in-flight error handling, `timeout` for deadlines, `finalize` for cleanup on either outcome, and `of` for returning a value without calling the handler. → Chapter 13

**Follow-up:** What is the difference between `tap` and `finalize`?</details>

**Q114.** *Mid* — Why does a bare `timeout(5000)` produce a 500 rather than a 504?
<details><summary>Answer</summary>It emits `TimeoutError`, which is not an `HttpException`, so the filter treats it as an unknown error. Add `catchError` converting it to `GatewayTimeoutException`. → Chapter 13

**Follow-up:** Why must a timeout be an interceptor rather than middleware?</details>

**Q115.** *Senior* — Why can't a response-transforming interceptor act on a handler that injects `@Res()`?
<details><summary>Answer</summary>That handler writes the response itself, so no value flows back through the observable and `map` never runs. The symptom is one endpoint mysteriously exempt from a global envelope. `@Res({ passthrough: true })` avoids it. → Chapter 13

**Follow-up:** How would you find that endpoint?</details>

**Q116.** *Mid* — What two conditions must both hold for `ClassSerializerInterceptor` to strip a field?
<details><summary>Answer</summary>The returned value must be an instance of the decorated class — a plain ORM row has no decorators to apply — and the interceptor must actually be applied. Missing either fails silently and the excluded field ships. → Chapter 13

**Follow-up:** What does `excludeExtraneousValues: true` change?</details>

**Q117.** *Senior* — Why is `excludeExtraneousValues: true` the safer default?
<details><summary>Answer</summary>It inverts serialization from a denylist to an allowlist, so a newly added database column is not exposed until someone explicitly adds `@Expose()`. Without it, every future column is public by default. → Chapter 13

**Follow-up:** What breaks when you first enable it?  (Everything, until fields carry `@Expose()`.)</details>

**Q118.** *Mid* — Where should the conversion to a DTO happen — controller, service, or interceptor?
<details><summary>Answer</summary>At the service boundary, so every consumer — HTTP, queue, GraphQL — gets the safe shape rather than only routes with the interceptor applied. → Chapter 13

**Follow-up:** So is the interceptor redundant?  (No — it is defence in depth.)</details>

**Q119.** *Junior* — What is a pipe's job?
<details><summary>Answer</summary>Transform or reject a single handler argument. It receives the value and `ArgumentMetadata` describing what it is processing, and either returns the value the handler will get or throws. → Chapter 14

**Follow-up:** What is in `ArgumentMetadata`?</details>

**Q120.** *Mid* — Why is `metatype` the key field in `ArgumentMetadata`?
<details><summary>Answer</summary>It is the declared class, recorded by `emitDecoratorMetadata`, and it is what `ValidationPipe` validates against. For a parameter typed as an interface it is `Object`, so validation is skipped entirely. → Chapter 14

**Follow-up:** What does that mean for DTOs declared as interfaces?</details>

**Q121.** *Mid* — In what order do multiple pipes on one parameter run?
<details><summary>Answer</summary>Left to right, which is why `new DefaultValuePipe(1), ParseIntPipe` works and the reverse throws — the parser receives `undefined`. → Chapter 14

**Follow-up:** Give another case where order matters.</details>

**Q122.** *Junior* — Name the built-in pipes.
<details><summary>Answer</summary>`ValidationPipe`, `ParseIntPipe`, `ParseFloatPipe`, `ParseBoolPipe`, `ParseArrayPipe`, `ParseUUIDPipe`, `ParseEnumPipe`, `DefaultValuePipe`, `ParseFilePipe`. → Chapter 14

**Follow-up:** How do you change the status code one of them throws?</details>

**Q123.** *Senior* — Why might you make `ParseUUIDPipe` throw 404 instead of 400?
<details><summary>Answer</summary>To avoid telling a caller whether a given id format is even recognised by the system — the same reasoning that makes a scoped query return 404 rather than 403 for another user's record. → Chapter 14

**Follow-up:** When would 400 be the better choice?</details>

**Q124.** *Mid* — Where does the line fall between a pipe and a service?
<details><summary>Answer</summary>A pipe validates *shape*; a service validates *state*. Uniqueness is a fact about the system, so it belongs in the service where it applies to every entry point and can be enforced atomically by a constraint — a pipe's check-then-insert has a race window and only covers HTTP. → Chapter 14

**Follow-up:** What is the cost of getting that wrong?</details>

**Q125.** *Junior* — What does an exception filter do?
<details><summary>Answer</summary>Catches thrown exceptions and writes the response. It is not part of the inbound chain — nothing calls a filter; it runs because something threw. → Chapter 15

**Follow-up:** What does `@Catch()` with no arguments catch?</details>

**Q126.** *Mid* — When filters are bound at global, controller, and route level, which applies?
<details><summary>Answer</summary>The most specific binding wins — route over controller over global. → Chapter 15

**Follow-up:** What about ordering between several *global* filters?  (Unspecified — do not depend on it.)</details>

**Q127.** *Senior* — Why should a global catch-all filter check `host.getType()`?
<details><summary>Answer</summary>A filter registered with `APP_FILTER` also catches from WebSocket gateways and microservice handlers, which have no Express `Response`. Calling `.status()` on `undefined` throws *inside the error handler*, converting a recoverable error into an unhandled rejection. Rethrowing hands it to that transport's default handling, which is correct rather than a fallback. → Chapter 15

**Follow-up:** Why is an HTTP error contract meaningless over a WebSocket?</details>

**Q128.** *Mid* — What is the most common bug in a global `@Catch()` filter?
<details><summary>Answer</summary>Not checking for `HttpException` and preserving its status, so every deliberate status — 400 from validation, 401 from a guard — collapses into 500. → Chapter 15

**Follow-up:** How would you notice that in production?</details>

**Q129.** *Junior* — What are the three binding scopes for guards, interceptors, and pipes?
<details><summary>Answer</summary>Global, controller, and route. They apply cumulatively — a narrower binding does not replace a broader one. → Chapter 10

**Follow-up:** Which of the five components cannot be bound with a decorator?  (Middleware.)</details>

**Q130.** *Mid* — Does decorator stacking order determine pipeline order?
<details><summary>Answer</summary>No. Nest fixes the guard → interceptor → pipe sequence itself. Decorator order only sequences components of the *same* kind — two interceptors run in the order listed. → Chapter 2

**Follow-up:** So why do decorators evaluate top-down and apply bottom-up?</details>

---

## 5. Validation and DTOs

**Q131.** *Junior* — What does `whitelist: true` do, and why is it a security control?
<details><summary>Answer</summary>It strips properties the DTO does not declare. Without it, undeclared fields reach your service — and if a DTO is ever passed into an ORM `save()`, a client can set any column. That is mass assignment. → Chapter 14

**Follow-up:** What does `forbidNonWhitelisted` add?</details>

**Q132.** *Mid* — Why is `whitelist` without `forbidNonWhitelisted` a poor experience?
<details><summary>Answer</summary>Unknown fields vanish silently, so a client that misspells `emailAddress` gets a 201 and believes it worked. With both flags they get a 400 naming the offending property. → Chapter 14

**Follow-up:** Which combination hides an attack rather than reporting it?</details>

**Q133.** *Senior* — Name the two ways validation fails open.
<details><summary>Answer</summary>`@ValidateNested()` without `@Type()`, where `design:type` records only `Array` so the element type is erased and nested objects are never checked; and a DTO declared as an interface, which is erased so `metatype` is `Object` and the pipe skips it. Both produce a 201 for input that should have been rejected. → Chapter 14

**Follow-up:** How do you detect either one?  (Only a rejection test.)</details>

**Q134.** *Mid* — Why is failing open worse than failing closed here?
<details><summary>Answer</summary>There is no error, no log line, and no warning — the request succeeds. A happy-path test passes whether or not the rule exists, so nothing in a normal suite notices. → Chapter 14

**Follow-up:** What is the cheapest test that would catch it?</details>

**Q135.** *Junior* — What does `transform: true` give you?
<details><summary>Answer</summary>Real DTO class instances rather than plain objects, which is what makes `@Type()`, property defaults, and `ClassSerializerInterceptor` work. → Chapter 14

**Follow-up:** What does it not do?  (Coerce without `@Type()`, unless implicit conversion is on.)</details>

**Q136.** *Mid* — Why be suspicious of `enableImplicitConversion`?
<details><summary>Answer</summary>It coerces from the declared type without `@Type()`, which is convenient and surprising — a `string` field receiving `123` becomes `'123'`, and a boolean sent as `'false'` may convert in ways you did not intend. Explicit `@Type()` states the intent at the property and behaves the same everywhere. → Chapter 14

**Follow-up:** How would you coerce a boolean query flag correctly?</details>

**Q137.** *Mid* — `?active=false` arrives as a truthy string. How do you fix it?
<details><summary>Answer</summary>`@Transform(({ value }) => value === 'true' || value === true)` with `@IsBoolean()`. `@Type(() => Boolean)` does not work — it applies `Boolean('false')`, which is `true`. → Chapter 14

**Follow-up:** Why does that trap survive code review so often?</details>

**Q138.** *Junior* — Why derive update DTOs with `PartialType`?
<details><summary>Answer</summary>Duplicated DTOs drift, and the drift is invisible until something is accepted that should not be. `PartialType` preserves the original decorators while making every field optional. → Chapter 14

**Follow-up:** How would you make one field impossible to submit at all?</details>

**Q139.** *Mid* — How do you make an immutable field rejected rather than merely optional?
<details><summary>Answer</summary>Compose: `PartialType(OmitType(CreateDto, ['email'] as const))`. `OmitType` removes it entirely, so with `forbidNonWhitelisted` a client submitting it gets a 400. → Chapter 14

**Follow-up:** What is the difference from marking it `@IsOptional()`?</details>

**Q140.** *Mid* — Why import mapped types from `@nestjs/swagger` rather than `@nestjs/mapped-types`?
<details><summary>Answer</summary>The Swagger versions preserve API documentation metadata as well as validation rules. Mixing both packages produces DTOs that validate correctly and document inconsistently. → Chapter 14

**Follow-up:** What if you are not using Swagger?  (Either is fine; pick one.)</details>

**Q141.** *Senior* — When are validation groups the right tool, and when are they overuse?
<details><summary>Answer</summary>Right when one DTO genuinely serves two or three operations with slightly different requirements — a password required on create and not on update. Overuse beyond that: separate DTOs are clearer than one class whose rules depend on invisible context. → Chapter 14

**Follow-up:** How does a reader know which group applies?</details>

**Q142.** *Mid* — Why must `ValidationPipe` be registered with `APP_PIPE` rather than in `main.ts`?
<details><summary>Answer</summary>`main.ts` never runs in a testing module, so validation is absent from tests — meaning tests exercise a different pipeline than production. As a provider it is part of the module graph and applies everywhere. → Chapter 14

**Follow-up:** What else in `main.ts` has the same problem?</details>

**Q143.** *Junior* — Why does every list endpoint need a maximum page size?
<details><summary>Answer</summary>An unbounded `limit` lets a client request the whole table, which is a denial-of-service vector that looks like a missing feature. → Chapter 14

**Follow-up:** Where do you enforce it — DTO or service?  (Both; the DTO is the contract.)</details>

**Q144.** *Mid* — Why bound array length and string length in a DTO?
<details><summary>Answer</summary>Same reason as page size: unbounded input is a resource-exhaustion vector. An array with no `@ArrayMaxSize` lets one request create ten thousand nested objects that all get validated. → Chapter 14

**Follow-up:** What about nesting depth?</details>

**Q145.** *Senior* — Someone proposes `disableErrorMessages: true` in production. Evaluate it.
<details><summary>Answer</summary>The concern — not leaking the schema — is legitimate; the remedy is blunt. It replaces every failure with a bare 400, so clients cannot render form errors and your support team loses its fastest diagnostic. The schema exposure it prevents is modest, since field names are discoverable by probing. Use a custom `exceptionFactory` returning field paths and error codes without constraint internals or received values. → Chapter 14

**Follow-up:** Why must the response never echo the received value?</details>

**Q146.** *Mid* — Write the shape of a good validation error response.
<details><summary>Answer</summary>A stable machine-readable code plus a structured `errors` array of `{ field, code }` — never the raw constraint message, never the submitted value, and never a bare string that clients must parse. → Chapter 15

**Follow-up:** Why does Nest's default make this awkward?  (`message` is a string for most exceptions and an array for validation.)</details>

**Q147.** *Mid* — Where does validation sit in the pipeline, and why last?
<details><summary>Answer</summary>After guards and interceptors, immediately before the handler. It is the most expensive per-request step and the most specific, so it runs only for callers already known to be authenticated, authorized, and hitting a real handler. → Chapter 10

**Follow-up:** What does that mean for a caching interceptor's key?</details>

**Q148.** *Senior* — How would you validate a payload whose shape depends on a discriminator field?
<details><summary>Answer</summary>`@Type()` with a `discriminator`, mapping each `type` value to its own class. A single union-shaped DTO would have to permit the union of all fields, so a payload could mix keys from two variants and still pass `whitelist`. Bound the nesting depth too — recursive DTOs with unbounded depth are a denial-of-service vector validation itself will walk. → Chapter 14

**Follow-up:** When would you stop using DTOs entirely?  (Genuinely open-ended shapes — validate against a real schema and store as opaque.)</details>

**Q149.** *Mid* — Why does `@Exclude()` on a DTO sometimes break incoming validation?
<details><summary>Answer</summary>It is a `class-transformer` decorator affecting serialization in both directions, so with `transform: true` it can strip the property on the way *in* before validation sees it. Use serialization groups, or a separate response DTO. → Chapter 14

**Follow-up:** What does that suggest about reusing one class for request and response?</details>

**Q150.** *Senior* — A team generates DTOs from an OpenAPI spec as TypeScript interfaces, to guarantee the API and types never drift. Respond.
<details><summary>Answer</summary>The goal is right and the output format is wrong. Generated interfaces are erased, so you gain compile-time alignment and lose all runtime validation — the API accepts malformed bodies that type-check perfectly. Generate classes with `class-validator` decorators, or derive a validated class from the generated interface. The principle generalises: at a system boundary, a type that exists only at compile time is documentation, not enforcement. → Chapter 2

**Follow-up:** Does Standard Schema in v12 change that answer?</details>

---

## 6. Error Handling

**Q151.** *Junior* — Why does `throw new Error('not found')` produce a 500?
<details><summary>Answer</summary>Only `HttpException` subclasses carry a status. Anything else is treated as an unexpected failure, and its message is deliberately withheld because arbitrary errors contain connection strings, file paths, and query fragments. → Chapter 15

**Follow-up:** So how do you return a 404?</details>

**Q152.** *Junior* — Name the exceptions you would reach for most.
<details><summary>Answer</summary>`BadRequestException` (400), `UnauthorizedException` (401), `ForbiddenException` (403), `NotFoundException` (404), `ConflictException` (409), `UnprocessableEntityException` (422), `ServiceUnavailableException` (503), `GatewayTimeoutException` (504). → Chapter 15

**Follow-up:** When would you use 422 rather than 400?</details>

**Q153.** *Mid* — When is 422 the right status rather than 400?
<details><summary>Answer</summary>When the request is syntactically valid and semantically wrong — the shape parsed and validated, but a business rule rejects it. 400 is for malformed input. → Chapter 15

**Follow-up:** Do clients actually distinguish them?  (Only if you give them a `code` too.)</details>

**Q154.** *Mid* — Why should a service throw domain errors rather than HTTP exceptions?
<details><summary>Answer</summary>HTTP status codes are meaningless to a queue consumer, a scheduled job, or a GraphQL resolver calling the same method. Domain errors keep the service transport-agnostic, with one filter owning the mapping in a reviewable table. → Chapter 15

**Follow-up:** When is that indirection not worth it?  (A purely HTTP service with one entry point.)</details>

**Q155.** *Mid* — What must a client-usable error contract contain?
<details><summary>Answer</summary>A stable machine-readable `code` clients branch on — never a human-readable message, which gets reworded — and a correlation id, without which "it failed around 3pm" is the entire bug report. → Chapter 15

**Follow-up:** What standard format covers this?</details>

**Q156.** *Junior* — What is RFC 9457?
<details><summary>Answer</summary>The `application/problem+json` error format, obsoleting RFC 7807. Adopting it means gateways and client libraries already understand your errors. → Chapter 15

**Follow-up:** What fields does it define?</details>

**Q157.** *Mid* — What is the most common bug in a global exception filter?
<details><summary>Answer</summary>Not preserving `HttpException` statuses, so every deliberate 400 and 401 collapses into 500 — turning a well-behaved API into one that returns 500 for a mistyped email. → Chapter 15

**Follow-up:** How would you notice that in production?</details>

**Q158.** *Senior* — Why must a filter never throw?
<details><summary>Answer</summary>An exception inside a filter has nowhere to go — it becomes an unhandled rejection, taking down the response entirely and potentially the process. Filters are the last line of handling and must be the most defensive code in the system. → Chapter 15

**Follow-up:** What is the most common way a filter throws?  (`switchToHttp()` on a non-HTTP context.)</details>

**Q159.** *Mid* — What should a filter never put in a response?
<details><summary>Answer</summary>Stack traces, driver messages, and connection strings — driver errors in particular embed credentials. Log the detail; return a code and a correlation id. → Chapter 15

**Follow-up:** How would you test that?</details>

**Q160.** *Mid* — How do you map a database unique-constraint violation to 409?
<details><summary>Answer</summary>A filter catching the driver's error type and checking its code — `23505` in PostgreSQL — returning `ConflictException`'s status with a generic message. The driver's `meta`/`detail` names the table, column, and value, so it stays in the log. → Chapter 19

**Follow-up:** Why not catch it in the service?  (You would repeat it at every call site.)</details>

**Q161.** *Mid* — At what level should 4xx and 5xx be logged?
<details><summary>Answer</summary>4xx at warn or below — it is the caller's problem and needs no stack. 5xx at error with the stack — it is yours. Logging everything at error makes the error rate meaningless and buries real failures under validation noise. → Chapter 15

**Follow-up:** What does that do to alerting?</details>

**Q162.** *Senior* — Ordering between several global filters — what can you rely on?
<details><summary>Answer</summary>Nothing. The docs do not define it, so a design depending on it was never guaranteed. Use one global catch-all and bind narrower filters at controller or route scope, where "most specific wins" *is* defined. → Chapter 15

**Follow-up:** How would that failure present?  (The wrong filter handling an error after an unrelated refactor.)</details>

**Q163.** *Mid* — Why does `ValidationPipe` complicate a uniform error shape?
<details><summary>Answer</summary>Its `message` is an array while every other built-in exception's is a string, so any client parsing it must handle both types. Normalising that in one filter is among the highest-value things a global filter does. → Chapter 15

**Follow-up:** What structure would you normalise it into?</details>

**Q164.** *Senior* — A teammate wants stack traces in staging but not production, gated by an environment variable. Evaluate it.
<details><summary>Answer</summary>The value is real and the control is a single string comparison, whose failure mode is leaking file paths, queries, and credentials to the internet. Gate on an allowlist of non-production environments so an unset or misspelled variable fails closed. Better still: keep the stack out of the response entirely and let staging users look it up by correlation id — same debugging benefit, no exposure. Any path protected only by `!== 'production'` will eventually run in production. → Chapter 15

**Follow-up:** What else in a codebase is usually protected the same way?</details>

**Q165.** *Senior* — You must add a new error case for a mobile client that cannot be updated quickly. How?
<details><summary>Answer</summary>Add it within an existing status code and carry the specificity in `code`. An old client branching on status keeps working — it sees a 409 and shows its generic conflict message — while updated clients branch on the new code. That is precisely why a stable code alongside the status matters: the status is the coarse contract old clients depend on, and the code is the extensible one. What breaks old clients is changing the *status* for an existing condition, or changing `message` from a string to an object. → Chapter 15

**Follow-up:** What if no existing status fits?</details>

---

## 7. Databases and Transactions

**Q166.** *Junior* — What is `synchronize: true` and why is it dangerous?
<details><summary>Answer</summary>It inspects your entities and alters the schema to match — including dropping columns whose properties you removed, without prompting. Acceptable in a disposable local database, catastrophic anywhere with data you care about. → Chapter 17

**Follow-up:** What replaces it?</details>

**Q167.** *Mid* — What do migrations give you that `synchronize` cannot?
<details><summary>Answer</summary>An explicit, ordered, reviewable, reversible record of schema change — something a colleague can read in a pull request and a pipeline can test up, down, and up again. → Chapter 18

**Follow-up:** Should migrations run at application boot?</details>

**Q168.** *Mid* — Why should migrations not run at application boot?
<details><summary>Answer</summary>With several replicas they all start and race; the lock keeps the schema safe but instances block, and a slow migration can exhaust a deployment's startup timeout. Run them as a separate deploy step where they are observable. → Chapter 18

**Follow-up:** What else does that make possible?  (Knowing they succeeded before any instance serves traffic.)</details>

**Q169.** *Senior* — Why must a generated migration be read before committing?
<details><summary>Answer</summary>Generation diffs your entities against your *local* database, so any local drift is encoded into the migration — including destructive statements you did not intend. → Chapter 18

**Follow-up:** How would you catch a destructive statement in CI?</details>

**Q170.** *Mid* — Walk through adding a `NOT NULL` column to a large live table.
<details><summary>Answer</summary>Three steps across deploys: add it nullable (metadata-only, instant); deploy code that writes it; backfill in bounded batches; then set `NOT NULL`. One statement with a default rewrites the whole table while holding a lock. → Chapter 18

**Follow-up:** What about adding a foreign key?  (`NOT VALID`, then `VALIDATE CONSTRAINT` separately.)</details>

**Q171.** *Mid* — Why is renaming a column in one migration unsafe?
<details><summary>Answer</summary>During a rolling deploy the old code is still running and references the old name, so it breaks the instant the migration applies. Expand-migrate-contract: add the new column, write both, backfill, switch reads, then drop — four deploys and no outage. → Chapter 18

**Follow-up:** How many deploys is the unsafe version?  (One, plus an incident.)</details>

**Q172.** *Mid* — Why does `CREATE INDEX CONCURRENTLY` need `transaction = false`?
<details><summary>Answer</summary>It cannot run inside a transaction, and Nest wraps migrations in one by default. Without the opt-out the migration fails outright. → Chapter 18

**Follow-up:** What happens if you use a plain `CREATE INDEX` on a large table?</details>

**Q173.** *Junior* — What is the N+1 problem and how do you detect it?
<details><summary>Answer</summary>Fetching a collection then touching a lazily-loaded relation per row — 101 queries for 100 rows. Detect it by enabling query logging in development, where it presents as a wall of near-identical `SELECT`s, or by asserting query counts in a test. → Chapter 17

**Follow-up:** What is the wrong fix?</details>

**Q174.** *Senior* — Why is `eager: true` the wrong fix for N+1?
<details><summary>Answer</summary>It loads the relation on *every* query touching that entity, including a simple existence check that now joins four tables, and there is no per-call opt-out. It also replaces a visible problem — repeated statements in a log — with an invisible one spread evenly across every query. → Chapter 17

**Follow-up:** What is the right answer to "nobody remembers `relations`"?</details>

**Q175.** *Mid* — Why does `findOne({ where: { id: undefined } })` return a row?
<details><summary>Answer</summary>TypeORM drops a condition whose value is `undefined` rather than matching nothing, so the query matches everything and returns the first row. On a user table that is an authorization bypass. → Chapter 17

**Follow-up:** Where do you prevent it?  (A pipe at the boundary, not a guard in the service.)</details>

**Q176.** *Mid* — Why store money as integer minor units?
<details><summary>Answer</summary>`float` cannot represent `0.1`, and `decimal` is returned as a *string* by the PostgreSQL driver — so `+` concatenates rather than adds. Integers avoid both. → Chapter 17

**Follow-up:** What breaks first with `decimal`?  (Summing in JavaScript.)</details>

**Q177.** *Mid* — Why `timestamptz` rather than `timestamp`?
<details><summary>Answer</summary>Without the zone, the same instant is interpreted differently depending on the server's zone, so the bug appears only after a region change or a daylight-saving shift — long after the code was written. → Chapter 17

**Follow-up:** Where else does the same reasoning apply?  (Cron time zones.)</details>

**Q178.** *Mid* — Explain the difference between `save`, `insert`, and `upsert`.
<details><summary>Answer</summary>`save` runs a `SELECT` to decide insert versus update, then the write — two queries, plus hooks and cascades. `insert` is one query with neither. `upsert` is `INSERT … ON CONFLICT`, which is atomic and avoids a check-then-insert race. → Chapter 17

**Follow-up:** What is `save`'s partial-update trap?</details>

**Q179.** *Senior* — What is `save`'s most dangerous behaviour?
<details><summary>Answer</summary>Given an object *with* an id it performs a partial update, silently leaving other columns alone; given one *without* an id it inserts a new row. So a DTO missing its id where you expected an update quietly creates a duplicate. → Chapter 17

**Follow-up:** How would a test catch that?</details>

**Q180.** *Mid* — What is the single most common transaction bug in Nest?
<details><summary>Answer</summary>Using an injected repository inside `dataSource.transaction()`. It has its own connection, so its writes run outside the transaction and survive the rollback — silently, until a rollback actually happens. → Chapter 18

**Follow-up:** How do you avoid threading `manager` through every call?</details>

**Q181.** *Senior* — How would you propagate a transaction across service calls without passing `manager` everywhere?
<details><summary>Answer</summary>`AsyncLocalStorage`-based propagation, which is what `typeorm-transactional` packages — a decorator establishes a transaction that nested calls join automatically. It removes the parameter threading at the cost of making the transaction implicit, which is the same trade as ambient request context. → Chapter 18

**Follow-up:** When is explicit threading better?</details>

**Q182.** *Mid* — Why is read-then-write wrong for stock or counters?
<details><summary>Answer</summary>Two callers both read the old value and both write, so one update is lost. A conditional `UPDATE … WHERE stock >= :qty` evaluates the condition and the write atomically, and `affected === 0` tells you it failed. → Chapter 18

**Follow-up:** Why does this never show up in testing?</details>

**Q183.** *Mid* — Compare optimistic and pessimistic locking.
<details><summary>Answer</summary>Optimistic uses a version column and fails the second writer, costing nothing when there is no contention — right for user-facing edits, with a 409 and a client retry. Pessimistic takes a database lock so the second reader waits, correct under heavy contention, and it serialises access and can deadlock. → Chapter 18

**Follow-up:** How do you prevent the deadlock?</details>

**Q184.** *Senior* — How do you prevent lock-ordering deadlocks?
<details><summary>Answer</summary>Impose a global ordering on lock acquisition — sort the ids before locking. If every transaction in the system acquires locks in the same order, a cycle cannot form. Without it, A→B while B→A deadlocks under exactly the traffic you would expect, and it presents as intermittent failures nobody can reproduce deliberately. → Chapter 18

**Follow-up:** Do you still need retries?  (Yes — serialization failures can still occur.)</details>

**Q185.** *Mid* — When would you raise the isolation level?
<details><summary>Answer</summary>Only when correctness depends on it — a balance check followed by a withdrawal, reserving the last seat. `SERIALIZABLE` makes transactions abort under contention, so any code using it must retry. Usually a conditional `UPDATE … WHERE` gives the same guarantee at `READ COMMITTED` with no retry loop. → Chapter 18

**Follow-up:** Which SQLSTATE codes are safe to retry?  (`40001` and `40P01`.)</details>

**Q186.** *Senior* — Why should transactions be short and free of third-party calls?
<details><summary>Answer</summary>A transaction holds a pooled connection for its whole duration, so a slow external call inside one converts network latency into held database resources. Long transactions also increase lock contention and, in PostgreSQL, delay vacuum — degrading the whole database rather than just that endpoint. → Chapter 18

**Follow-up:** So where does the notification go?</details>

**Q187.** *Mid* — What does a composite index on `[a, b]` serve?
<details><summary>Answer</summary>Queries filtering on `a`, and on `a` and `b` together — usable left to right. It does nothing for a query filtering on `b` alone, which makes column order a decision rather than a formality. → Chapter 17

**Follow-up:** How would you verify the planner is using it?</details>

**Q188.** *Mid* — Why prefer keyset to offset pagination?
<details><summary>Answer</summary>`OFFSET` makes the database scan and discard every skipped row, so latency grows with depth, and concurrent inserts shift every subsequent page — duplicating or skipping rows as a user scrolls. A cursor is constant-cost and stable. → Chapter 17

**Follow-up:** What does keyset give up?  (Jumping to page 50.)</details>

**Q189.** *Senior* — What is the detail most keyset implementations omit?
<details><summary>Answer</summary>The tiebreaker. Ordering by a timestamp alone leaves rows sharing a millisecond with no defined order, so the cursor can land mid-group and silently drop records. Adding a unique column makes the ordering total, which is what makes the cursor exact. → Chapter 17

**Follow-up:** What SQL construct expresses that cleanly?  (Row-value comparison.)</details>

**Q190.** *Mid* — Why must client-supplied sort columns be allowlisted?
<details><summary>Answer</summary>Values in a query builder can be bound as parameters; identifiers cannot. So a column name interpolated into `ORDER BY` is SQL injection, and it is the vector that survives using an ORM. → Chapter 17

**Follow-up:** Where else does that apply?  (Table names, and anything in a `RETURNING` list.)</details>

**Q191.** *Mid* — What does `Brackets` protect against in a query builder?
<details><summary>Answer</summary>Operator precedence. An `orWhere` added without it produces `owner = x AND title LIKE y OR description LIKE z`, where the last clause matches regardless of owner — so a search endpoint starts returning other users' records. → Chapter 17

**Follow-up:** Would a test have caught it?  (Only one with two users' data.)</details>

**Q192.** *Junior* — What does `@DeleteDateColumn` enable, and what is easy to forget?
<details><summary>Answer</summary>Soft delete — `softDelete()` sets the timestamp and every subsequent `find` excludes those rows automatically. Easy to forget when writing raw SQL or a query builder, where the exclusion is not applied for you. → Chapter 17

**Follow-up:** How do you deliberately include them?  (`withDeleted: true`.)</details>

**Q193.** *Mid* — Why return DTOs rather than entities from a service?
<details><summary>Answer</summary>An entity serializes every column including ones added later, so a future migration can leak a hash or an internal flag without anyone touching the controller — and touching a lazy relation during serialization triggers extra queries. → Chapter 17

**Follow-up:** Where should that conversion happen?</details>

**Q194.** *Senior* — Prisma or TypeORM — how do you decide?
<details><summary>Answer</summary>On the dimension that will actually bite. Prisma if many partial selects and a large team make a field silently becoming `undefined` a realistic failure — its result types match the `select` exactly. TypeORM if module boundaries matter, because `forFeature` gives a real one and injecting a client that exposes every table is a genuine architectural loss. Note that complex analytical reads end up as raw SQL either way. Benchmarks should not decide it — your queries and indexes dominate. → Chapter 19

**Follow-up:** How would you regain encapsulation with Prisma?</details>

**Q195.** *Senior* — A 40-million-row table's list endpoint is slow despite an index. What do you investigate?
<details><summary>Answer</summary>Read the plan with `EXPLAIN ANALYZE` rather than reasoning about the index — at that size the cause is usually an index the planner declines to use. Check selectivity, whether a function or type mismatch makes it unusable, whether offset pagination is discarding millions of rows, and whether the index's column order matches the query. If the plan is optimal and it is still slow, the problem is volume: partitioning, a covering index, or a read replica, in that order of cost. → Chapter 34

**Follow-up:** What would you measure before any of that?</details>

---

## 8. Authentication and Security

**Q196.** *Junior* — What is the difference between authentication and authorization?
<details><summary>Answer</summary>Authentication establishes who the caller is; authorization decides whether they may do a thing. Keeping them in separate guards means each has one reason to change, and the registration order documents the dependency. → Chapter 21

**Follow-up:** Which one can be decided without loading the resource?</details>

**Q197.** *Mid* — What does a Passport strategy's `validate` return, and where does it go?
<details><summary>Answer</summary>Whatever it returns becomes `request.user`, which guards, decorators, and services then read. → Chapter 21

**Follow-up:** What should `validate` do besides decode the token?</details>

**Q198.** *Senior* — Why should `validate` do a database lookup?
<details><summary>Answer</summary>A valid signature proves the token was *issued*, not that the account still exists or is still permitted. Deletion, bans, and revoked permissions all happen after issuance. One indexed lookup closes the gap. → Chapter 21

**Follow-up:** What is the cost, and when would you skip it?</details>

**Q199.** *Mid* — Why pin `algorithms` in a JWT strategy?
<details><summary>Answer</summary>Without it, a library that accepts the token's own `alg` header can be tricked into verifying an `alg: none` token, or into treating an RS256 public key as an HMAC secret. Both are classic JWT vulnerabilities. → Chapter 21

**Follow-up:** How would you notice the omission?  (You would not — it works, for forged tokens too.)</details>

**Q200.** *Junior* — Why is Argon2id preferred for passwords?
<details><summary>Answer</summary>It is deliberately slow and memory-hard, resisting both GPU and side-channel attacks. A general-purpose hash like SHA-256 is brute-forceable at billions of guesses per second regardless of salting. → Chapter 21

**Follow-up:** What is bcrypt's specific trap?</details>

**Q201.** *Mid* — What is bcrypt's 72-byte behaviour?
<details><summary>Answer</summary>It silently truncates input at 72 bytes, so two different long passwords can verify against the same hash. If you use bcrypt, reject over-long passwords explicitly rather than letting them be cut. → Chapter 21

**Follow-up:** Where is the salt stored?  (Inside the hash string — you do not manage it.)</details>

**Q202.** *Senior* — How do you remove account enumeration from a login endpoint?
<details><summary>Answer</summary>Identical message *and* identical timing. Return one generic failure, and verify against a dummy hash when the user is absent so the response time does not distinguish "no such account" from "wrong password". Unifying only the message leaves a trivially measurable timing oracle. → Chapter 21

**Follow-up:** Where does the dummy hash come from?  (Computed once at boot.)</details>

**Q203.** *Mid* — Why does a slow hash not protect against credential stuffing?
<details><summary>Answer</summary>It protects the stored hashes if the database leaks. It does nothing to stop online guessing — that needs rate limiting on the endpoint, ordered before the hash so the throttle rejects first. → Chapter 21

**Follow-up:** What do you key the limit on?</details>

**Q204.** *Mid* — Why must refresh tokens be stored hashed?
<details><summary>Answer</summary>They are password-equivalent and long-lived, so a database leak containing raw refresh tokens is a full account compromise for every active session. SHA-256 is correct here — the input is random bytes with nothing to brute-force. → Chapter 21

**Follow-up:** Why is SHA-256 right there and wrong for passwords?</details>

**Q205.** *Senior* — What does refresh-token rotation detect that a static token cannot?
<details><summary>Answer</summary>Theft. A single-use token presented twice proves a copy exists somewhere it should not — the only signal you get that a token was stolen. Without rotation, an attacker holds a renewable session indefinitely and invisibly. → Chapter 21

**Follow-up:** What do you do when you detect reuse?</details>

**Q206.** *Senior* — Reuse detection logs out a legitimate user whose client retried. Defend that.
<details><summary>Answer</summary>The server genuinely cannot distinguish a harmless retry from an attacker replaying a stolen token, and choosing the lenient interpretation means theft goes undetected — exactly what rotation exists to prevent. The cost is bounded: the user logs in again. The alternative is unbounded. Mitigate by reducing *spurious* reuse — a short grace window where the immediately-previous token returns the same new pair — rather than by softening detection. → Chapter 21

**Follow-up:** Why revoke the whole family rather than one token?</details>

**Q207.** *Mid* — How do you revoke a JWT before it expires?
<details><summary>Answer</summary>You cannot, cleanly — and saying so is the correct answer. Keep access tokens short so the window is small, hold revocation state on the refresh token, and use a denylist only for emergencies, accepting that it reintroduces the per-request statefulness JWTs were chosen to avoid. → Chapter 21

**Follow-up:** So when are sessions the better choice?</details>

**Q208.** *Senior* — When would you choose server sessions over JWTs?
<details><summary>Answer</summary>When immediate revocation matters and everything sits behind one boundary — a session id can be invalidated instantly. JWTs win when independent services must verify identity without a shared store. The revocation row is what decides most real cases. → Chapter 21

**Follow-up:** What is the cost of a session?  (A store lookup per request.)</details>

**Q209.** *Mid* — Where should a client keep an access token?
<details><summary>Answer</summary>In memory, with the refresh token in an `httpOnly`, `Secure`, `SameSite=Strict`, path-scoped cookie. `localStorage` means any XSS anywhere on the origin — including in a third-party script — reads and exfiltrates it. → Chapter 21

**Follow-up:** What does `httpOnly` change about an XSS's impact?</details>

**Q210.** *Junior* — When does CSRF apply?
<details><summary>Answer</summary>When credentials travel in cookies, because the browser attaches them automatically to cross-site requests. A bearer-token API is not vulnerable — the browser does not add that header. → Chapter 23

**Follow-up:** What is the primary defence?</details>

**Q211.** *Mid* — Which CSRF package should you not use?
<details><summary>Answer</summary>`csurf` — deprecated and unmaintained, and still widely recommended in NestJS material. A security package that stopped being maintained is worse than none, because it creates the appearance of protection. Use a maintained alternative. → Chapter 23

**Follow-up:** How would you audit for others like it?</details>

**Q212.** *Mid* — Distinguish RBAC, ABAC, and ReBAC.
<details><summary>Answer</summary>RBAC decides from roles alone, so it needs no resource and fits a guard. ABAC uses attributes of subject, resource, action, and context — ownership, tenancy, state. ReBAC derives permission from a relationship graph and needs a dedicated system. → Chapter 22

**Follow-up:** What does inventing a role like `task-42-owner` tell you?</details>

**Q213.** *Mid* — Why check permissions rather than roles in business logic?
<details><summary>Answer</summary>A role-to-permission mapping in one place lets you add or split roles without touching every consumer. Scattered `user.role === 'admin'` comparisons must all be found and updated, and one will be missed. → Chapter 22

**Follow-up:** How many files does adding a role touch in each design?</details>

**Q214.** *Senior* — Why do ownership checks belong in the query rather than a guard?
<details><summary>Answer</summary>A guard must load the record to decide and cannot pass it to the handler, so it is fetched twice — and a missing record produces 403 rather than 404, leaking whether an id exists. A scoped query returns nothing for both cases, and the check cannot be forgotten at a call site. → Chapter 22

**Follow-up:** What is stronger still?  (Row-level security in the database.)</details>

**Q215.** *Mid* — Why return 404 rather than 403 for another user's resource?
<details><summary>Answer</summary>403 confirms the id is real, which is a useful oracle for anyone enumerating identifiers. Identical responses for "missing" and "not yours" remove it. → Chapter 22

**Follow-up:** When would 403 be acceptable?  (When existence is not sensitive.)</details>

**Q216.** *Senior* — What is CASL's silent bypass?
<details><summary>Answer</summary>Checking against a subject *type* rather than an instance. `ability.can('update', 'Task')` cannot evaluate conditions — it asks "may update *some* task" — so it returns `true`, type-checks, and lets a non-owner through. Conditions apply only to instances. → Chapter 22

**Follow-up:** How would you catch that in review?</details>

**Q217.** *Mid* — What makes CASL worth its dependency?
<details><summary>Answer</summary>The same rule definition drives both the instance check and the list-query filter, so "may I touch this" and "which may I see" cannot drift apart. A hand-written `WHERE` clause alongside ability rules is a second definition, and the query is the one that leaks. → Chapter 22

**Follow-up:** What is the symptom of that drift?</details>

**Q218.** *Senior* — Why is authorization in the HTTP pipeline insufficient?
<details><summary>Answer</summary>It protects HTTP. A queue consumer, cron job, CLI command, or GraphQL resolver calling the same service enforces nothing. The guard is the fast rejection; the service and the data layer are the guarantee. Centralise the *policy*, distribute the *enforcement*. → Chapter 22

**Follow-up:** Which layer catches a developer writing a raw query?</details>

**Q219.** *Mid* — Where does field-level authorization belong?
<details><summary>Answer</summary>In serialization, via `@Expose({ groups })` and `@SerializeOptions`, so a field is omitted declaratively rather than deleted by hand at each call site. → Chapter 22

**Follow-up:** What is the failure mode of the manual approach?</details>

**Q220.** *Mid* — What does `enableCors({ origin: true })` actually permit?
<details><summary>Answer</summary>Every origin — it reflects whatever `Origin` the caller sends. With `credentials: true` that exposes authenticated endpoints to any website. It is a development convenience that reaches production because nothing fails when it is wrong. → Chapter 23

**Follow-up:** What should an unset origin list mean?</details>

**Q221.** *Senior* — Why should a missing CORS configuration deny rather than allow?
<details><summary>Answer</summary>Allowing everything when configuration is absent is friendlier in development and is exactly the failure that reaches production — nothing breaks, and the only observable difference is that every site can now make credentialed requests. Denying breaks loudly in whatever environment forgot the variable, which is where you want to find out. → Chapter 23

**Follow-up:** Where else does the same principle apply?</details>

**Q222.** *Mid* — What does `trust proxy: true` allow an attacker to do?
<details><summary>Answer</summary>Forge `X-Forwarded-For`, getting a fresh rate-limit bucket per request and poisoning IP-based logging. Trust exactly the number of proxies you actually have. → Chapter 23

**Follow-up:** What is the other setting that makes rate limiting decorative?</details>

**Q223.** *Mid* — Why does in-memory rate limiting fail across replicas?
<details><summary>Answer</summary>Each instance keeps its own counter, so N instances permit N times the intended limit. Use shared storage. → Chapter 23

**Follow-up:** What else in a Nest app breaks the same way at N replicas?</details>

**Q224.** *Senior* — Explain mass assignment and prototype pollution, and the one option that closes both.
<details><summary>Answer</summary>Mass assignment is a client setting fields it should not by supplying extra properties, which becomes severe if a DTO is passed into `save()`. Prototype pollution is injecting `__proto__` through a merged object. `whitelist: true` strips every undeclared property, closing both systemically. → Chapter 23

**Follow-up:** So what is the DTO's real role?</details>

**Q225.** *Senior* — A client supplies a webhook URL you will call. What is the full threat model?
<details><summary>Answer</summary>SSRF first — resolve the hostname and reject private, loopback, link-local, and metadata ranges, re-checking on every call because DNS can be repointed after registration. Disable redirects or re-check each hop, since a public URL can 302 to an internal one. Then treat it as an outbound denial-of-service surface: bound the timeout, cap response size, cap concurrency per tenant. Sign the payload so the receiver can verify authenticity, forward no internal identifiers or auth headers, and deliver from a queue so a hostile endpoint cannot slow the request that triggered it. → Chapter 23

**Follow-up:** Why must the SSRF check happen after DNS resolution?</details>

---

## 9. Async: Queues, Events, Microservices

**Q226.** *Junior* — What is the test for whether work belongs in a queue?
<details><summary>Answer</summary>Whether the caller needs the result. A queue converts a synchronous failure into a retryable one — right for an email, wrong for a payment the user is waiting on. → Chapter 27

**Follow-up:** What goes wrong if you queue the charge?</details>

**Q227.** *Mid* — Why should job payloads contain ids rather than entities?
<details><summary>Answer</summary>Payloads are serialised into Redis, so an entity enqueued ten minutes ago is a stale snapshot, memory balloons, and any secret in the object now sits in Redis. Enqueue an id and re-read in the processor. → Chapter 27

**Follow-up:** What does re-reading also let you check?  (Whether the request was cancelled.)</details>

**Q228.** *Mid* — What does a deterministic `jobId` make idempotent, and what does it not?
<details><summary>Answer</summary>Enqueueing — adding a job with an existing id is a no-op, so a retried API call cannot create a second job. It does *not* make processing idempotent, because redelivery reuses the same job. → Chapter 27

**Follow-up:** So where does the real guarantee live?</details>

**Q229.** *Senior* — Where must an idempotency check live, and why?
<details><summary>Answer</summary>Inside the transaction that produces the effects, enforced by a unique constraint. A check before the transaction has a window where two workers both pass it, and a marker written after the commit has a window where a crash causes reprocessing. You cannot make delivery exactly-once, so you make *effects* idempotent. → Chapter 27

**Follow-up:** What does that pattern look like in a microservice consumer?</details>

**Q230.** *Mid* — Why classify failures rather than retrying everything?
<details><summary>Answer</summary>A malformed payload or a deleted referent fails identically every attempt, so retrying costs the full budget for the same outcome and starves healthy jobs. `UnrecoverableError` stops retries immediately. → Chapter 27

**Follow-up:** Which errors are worth retrying?</details>

**Q231.** *Mid* — What happens without `removeOnComplete` and `removeOnFail`?
<details><summary>Answer</summary>Finished jobs accumulate in Redis indefinitely until it exhausts its memory — a slow failure that presents as the queue mysteriously stopping. → Chapter 27

**Follow-up:** Why keep failed jobs longer than completed ones?</details>

**Q232.** *Senior* — A report is occasionally generated twice with no error. What do you check first?
<details><summary>Answer</summary>`lockDuration` against the real runtime. BullMQ renews a lock periodically; if CPU-bound work starves the renewal timer, the lock expires, the queue concludes the worker died, and hands the same job to another while the first is still running. Raise the duration, and move genuinely long CPU work off the event loop so renewal keeps happening. → Chapter 27

**Follow-up:** Would idempotency have prevented it?  (The duplicate effect, yes — after both did the work.)</details>

**Q233.** *Mid* — Why run processors in a separate process from the API?
<details><summary>Answer</summary>CPU-bound jobs block the event loop, so in-process workers degrade request latency for every user regardless of how the bytes arrived. → Chapter 27

**Follow-up:** How would you notice?  (Event-loop lag rising during job execution.)</details>

**Q234.** *Mid* — Why should queues use a different Redis from the cache?
<details><summary>Answer</summary>A cache flush would destroy pending work. Queues need a persistent instance; caches are ephemeral by design. → Chapter 27

**Follow-up:** What other shared-infrastructure mistake looks similar?</details>

**Q235.** *Mid* — What metrics actually tell you a queue is healthy?
<details><summary>Answer</summary>Depth *and* oldest-job age together. Depth alone is misleading — 10,000 jobs draining in a minute is healthy; 50 jobs whose oldest is an hour old is stuck. Plus failure rate, since retries can consume most of your throughput. → Chapter 27

**Follow-up:** Adding workers did not help. What does that tell you?</details>

**Q236.** *Junior* — Distinguish message patterns from event patterns.
<details><summary>Answer</summary>`@MessagePattern`/`send()` is request–response: the caller waits, one consumer replies, failure is visible. `@EventPattern`/`emit()` is fire-and-forget: no reply, any number of consumers, and the emitter does not know whether anyone handled it. → Chapter 26

**Follow-up:** Which should be the default, and why?</details>

**Q237.** *Mid* — Why prefer events to commands between services?
<details><summary>Answer</summary>A `send` couples the caller's latency and availability to the callee's; an `emit` does not, and new consumers can be added without touching the emitter. Reach for request–response only when the caller genuinely cannot proceed without the answer. → Chapter 26

**Follow-up:** What must a caller add when it does use `send`?</details>

**Q238.** *Senior* — `client.send()` appears to do nothing and logs no error. Why?
<details><summary>Answer</summary>It returns a *cold* Observable — nothing is transmitted until something subscribes. Return it from a handler so Nest subscribes, `await firstValueFrom(...)`, or call `.subscribe()`. → Chapter 26

**Follow-up:** What else must wrap that call?  (A timeout.)</details>

**Q239.** *Mid* — What happens without a timeout on a synchronous inter-service call?
<details><summary>Answer</summary>The caller waits indefinitely for a dead callee, waiting requests accumulate, and the caller exhausts its own resources — one service's outage becomes everyone's. → Chapter 26

**Follow-up:** What is the next step beyond a timeout?  (A circuit breaker.)</details>

**Q240.** *Mid* — Which transports persist messages, and which do not?
<details><summary>Answer</summary>RabbitMQ has durable queues and Kafka is a durable log. TCP, Redis pub/sub, and core NATS do not — Redis pub/sub drops anything published while no subscriber is connected, so a consumer restart loses it. → Chapter 26

**Follow-up:** Which one lets a *new* consumer read history?</details>

**Q241.** *Mid* — When is Kafka the requirement rather than a preference?
<details><summary>Answer</summary>When a future consumer will need events that already happened. Replay is the property only a durable log provides; no other common transport offers it. → Chapter 26

**Follow-up:** What does that cost operationally?</details>

**Q242.** *Senior* — What is the Kafka request–response trap in Nest?
<details><summary>Answer</summary>The client must call `subscribeToResponseOf(pattern)` in `onModuleInit` *before* sending, so the reply topic is consumed. Omit it and every `send()` hangs until timeout, with no error explaining why. → Chapter 26

**Follow-up:** How would you debug that from scratch?</details>

**Q243.** *Mid* — What does `noAck: false` change for RabbitMQ?
<details><summary>Answer</summary>It makes acknowledgement manual. With the default, the broker considers a message delivered the moment it is sent, so a crash mid-processing loses it silently. → Chapter 26

**Follow-up:** What must you then be careful about?  (Never `nack` with requeue on a poison message.)</details>

**Q244.** *Mid* — Why dead-letter rather than requeue a repeatedly failing message?
<details><summary>Answer</summary>A message failing deterministically fails again immediately, producing a hot loop that saturates the consumer and starves healthy messages. Dead-lettering is only useful if something watches it. → Chapter 26

**Follow-up:** What is the minimum monitoring for a DLQ?</details>

**Q245.** *Senior* — What ordering guarantees can you rely on?
<details><summary>Answer</summary>Per-partition at best. Kafka orders within a partition if you key by entity id; RabbitMQ orders only with a single consumer. Any handler assuming `created` arrives before `updated` will eventually be wrong, so the durable fix is order-independent handlers — a version or timestamp check. → Chapter 26

**Follow-up:** Is that fixable by configuration?</details>

**Q246.** *Mid* — What is the outbox pattern and what problem does it solve?
<details><summary>Answer</summary>Writing the state change and an event row in one transaction, with a relay publishing committed rows separately. It closes the gap where publishing after commit can be lost to a crash, and publishing inside a transaction can deliver an event for work that rolled back. → Chapter 28

**Follow-up:** What must consumers still handle?</details>

**Q247.** *Mid* — How do you run an outbox relay on several replicas?
<details><summary>Answer</summary>`SELECT … FOR UPDATE SKIP LOCKED`, so each instance claims a disjoint batch without blocking. A plain `FOR UPDATE` serialises them; no locking at all makes every instance publish every row. → Chapter 28

**Follow-up:** What happens if the relay crashes between publishing and marking?</details>

**Q248.** *Senior* — What is `@nestjs/cqrs`'s `EventBus` not?
<details><summary>Answer</summary>Durable. It is in-memory and in-process, so events are not persisted, not retried, and lost if the process dies between publishing and handling. It is a decoupling mechanism, not a delivery guarantee. → Chapter 28

**Follow-up:** So what do you use for anything that must not be lost?</details>

**Q249.** *Mid* — What is a saga, and why compensate rather than roll back?
<details><summary>Answer</summary>A process reacting to events by dispatching commands, coordinating a multi-step operation. Each step is independently committed and visible, so there is nothing to roll back — undoing means issuing an explicit reversing command. → Chapter 28

**Follow-up:** What happens to a saga with no failure path?</details>

**Q250.** *Senior* — Two services must both update state as part of one action, without a half-applied result. How?
<details><summary>Answer</summary>Outbox plus saga, not a two-phase commit. Each service writes its state and an outbox row in one local transaction; a relay publishes; the second service consumes and applies idempotently, then emits its own event. A later failure compensates with a reversing action. That means accepting eventual consistency and designing the user-facing experience around it — a pending state rather than a lie. If the operation cannot tolerate any window of inconsistency, that is strong evidence the two pieces of state belong in one service. → Chapter 28

**Follow-up:** How would you expose that in the API?  (202 with a status endpoint.)</details>

---

## 10. GraphQL and WebSockets

**Q251.** *Junior* — Why is code-first the better fit for NestJS?
<details><summary>Answer</summary>Decorated classes stay the single source of truth and the SDL is generated, so there is no second artefact to keep in sync — the same reasoning that makes decorators fit the rest of the framework. → Chapter 24

**Follow-up:** Should the generated schema be committed?</details>

**Q252.** *Mid* — Why commit the generated `schema.gql`?
<details><summary>Answer</summary>Schema diffs become visible in review, so a breaking change is something a reviewer sees rather than something a client discovers. Add a CI check that fails on uncommitted drift. → Chapter 24

**Follow-up:** What is the equivalent practice for REST?</details>

**Q253.** *Mid* — Why is the type thunk in `@Field(() => X)` mandatory?
<details><summary>Answer</summary>Metadata cannot distinguish `string` from `ID`, and cannot see an array's element type at all. Omitting it on an array is a startup error; omitting it on `ID` silently produces `String` — a subtly wrong schema with no error. → Chapter 24

**Follow-up:** What other Nest feature has the same erasure problem?</details>

**Q254.** *Senior* — Why is N+1 structural in GraphQL rather than incidental?
<details><summary>Answer</summary>The client chooses the query shape at runtime, so you cannot pre-plan joins — and a field resolver necessarily runs once per parent. Eager loading cannot help because you do not know what will be requested. → Chapter 24

**Follow-up:** What is the answer, and what are its limits?</details>

**Q255.** *Mid* — What does DataLoader do, and what can it not do?
<details><summary>Answer</summary>Batches loads made within the same tick into one query and deduplicates keys. It cannot batch across sibling branches that resolve in different ticks. → Chapter 24

**Follow-up:** What scope must it have?</details>

**Q256.** *Senior* — Why must a DataLoader be request-scoped?
<details><summary>Answer</summary>Its cache is keyed by id with no TTL, so a singleton would serve a user deleted ten minutes ago and — worse — hand one request's data to another. It is one of the few places request scope is genuinely justified. → Chapter 24

**Follow-up:** What is the other subtle DataLoader bug?</details>

**Q257.** *Senior* — Why must a batch function return results positionally?
<details><summary>Answer</summary>DataLoader matches by position, not by id. Returning a filtered array when some keys are missing shifts every subsequent result, so records are silently attributed to the wrong parent. Map and pad with a placeholder. → Chapter 24

**Follow-up:** What should a "no rows" result be for a list field?  (`[]`, not `undefined`.)</details>

**Q258.** *Mid* — Why does an HTTP guard fail on a GraphQL resolver?
<details><summary>Answer</summary>`switchToHttp()` yields nothing in a GraphQL context. Use `GqlExecutionContext.create(ctx).getContext().req`, or branch on `getType()` so one guard serves both. → Chapter 24

**Follow-up:** What else in the pipeline needs the same treatment?</details>

**Q259.** *Senior* — Why enforce authorization in services rather than resolvers?
<details><summary>Answer</summary>GraphQL makes one service reachable through many shapes — a top-level query, a field resolver on another type, a mutation's return value — and each is a separate entry point. A resolver-level check must be repeated on every one, and the field-resolver path is the one people forget because it does not look like an endpoint. → Chapter 24

**Follow-up:** What stays in the resolver, then?</details>

**Q260.** *Mid* — Why must introspection be disabled in production?
<details><summary>Answer</summary>It publishes the complete schema, including internal fields and admin mutations. It is not a secret in the cryptographic sense, and it is a map. → Chapter 24

**Follow-up:** What else leaks by default outside production?</details>

**Q261.** *Mid* — Why are depth limits insufficient on their own?
<details><summary>Answer</summary>They do not catch *wide* queries — a hundred sibling fields at depth two is cheap by depth and expensive to execute. Complexity limits assign a cost per field and cap the total. → Chapter 24

**Follow-up:** What is the stronger option for an internal API?</details>

**Q262.** *Senior* — What do persisted queries buy you?
<details><summary>Answer</summary>The client sends a hash of a pre-registered query, so arbitrary queries become impossible — which removes most of the cost-control problem rather than tuning complexity scores. Suitable when you control every client. → Chapter 24

**Follow-up:** When is that not an option?</details>

**Q263.** *Mid* — How do GraphQL errors reach the client, and why does that matter?
<details><summary>Answer</summary>As a 200 with an `errors` array, so HTTP status codes stop being the channel. Apollo also attaches `exception.stacktrace` outside production, so one misconfigured environment variable exposes it — use `formatError` to return only the fields you intend rather than relying on `NODE_ENV`. → Chapter 24

**Follow-up:** What does that imply for client error handling?</details>

**Q264.** *Junior* — What is a gateway, and how does its CORS work?
<details><summary>Answer</summary>A class handling WebSocket messages — a controller for sockets. Its CORS is configured in `@WebSocketGateway()` options and is entirely separate from `app.enableCors()`, so an API with correct CORS can have a gateway accepting connections from every origin. → Chapter 25

**Follow-up:** What would make you notice?  (Nothing, until it is exploited.)</details>

**Q265.** *Mid* — Where should a socket be authenticated?
<details><summary>Answer</summary>At the handshake, once, with the principal attached to `client.data` — not per message, which is wasteful and leaves a window where an unauthenticated socket is connected. Read the token from the handshake `auth` payload, never a query string, which proxies log. → Chapter 25

**Follow-up:** What is the long-term problem with authenticating once?</details>

**Q266.** *Senior* — A socket authenticated at connection outlives its token. What do you do?
<details><summary>Answer</summary>Store the expiry on the socket and disconnect stale ones on a schedule, or subscribe to a revocation channel and disconnect affected sockets. Otherwise a user who is banned or logs out keeps their connection indefinitely. → Chapter 25

**Follow-up:** How does that differ from HTTP?  (Every HTTP request re-verifies.)</details>

**Q267.** *Mid* — Why are room names an authorization boundary?
<details><summary>Answer</summary>If the client chooses the room name, it can subscribe to any resource whose id it can guess. Verify access on join, and namespace names by resource type so `task:42` cannot collide with `user:42`. → Chapter 25

**Follow-up:** What is the difference between `server.to()` and `client.to()`?</details>

**Q268.** *Senior* — What breaks the moment a WebSocket app runs on a second replica?
<details><summary>Answer</summary>Broadcasts reach only the emitting process, so roughly one third of users receive an event with three replicas — the Redis adapter fixes that. In-memory presence maps become silently wrong. And Socket.IO's polling fallback needs sticky sessions, which forcing `transports: ['websocket']` avoids entirely. → Chapter 25

**Follow-up:** Which of those fails visibly?  (None of them.)</details>

**Q269.** *Mid* — Why track presence as a set with a TTL?
<details><summary>Answer</summary>One user commonly has several connections, so a boolean marks them offline when any one tab closes. And `handleDisconnect` does not run if the process is killed, so without expiry a crash leaves users permanently "online" with nothing to correct it. → Chapter 25

**Follow-up:** What general rule does that illustrate?</details>

**Q270.** *Senior* — Users expect messages to survive a brief disconnect. How do you design that?
<details><summary>Answer</summary>Not in the socket layer — an in-flight message to a dropped connection is lost. Durability needs a per-user inbox the client reconciles on reconnect: a stream or list with a cursor the client sends on connect to fetch what it missed. Treat the socket as a low-latency notification channel over a durable store, rather than as the store itself. → Chapter 25

**Follow-up:** What does that mean for fan-out cost at scale?</details>

---

## 11. Testing

**Q271.** *Junior* — When do you need `Test.createTestingModule`, and when is `new` enough?
<details><summary>Answer</summary>Use the testing module when the container is part of what you are testing — factory providers, scoped providers, lifecycle hooks, anything reading `Reflector`. Use `new` when testing a class's logic, since you are substituting its dependencies anyway. That is the whole point of constructor injection. → Chapter 29

**Follow-up:** What does the container add in the second case?  (Setup time, and nothing else.)</details>

**Q272.** *Mid* — Why is importing `AppModule` in a unit test usually wrong?
<details><summary>Answer</summary>It builds the entire graph and connects to real infrastructure, turning a millisecond test into a second — or a failure, if the database is absent. Declare the two or three providers under test. → Chapter 29

**Follow-up:** When is importing it correct?  (End-to-end tests.)</details>

**Q273.** *Mid* — Distinguish a stub, a mock, and a fake.
<details><summary>Answer</summary>A stub returns canned values — use it when the test cares about the result. A mock asserts on interactions — use it when the call itself is the behaviour. A fake is a working implementation with a shortcut — use it when several operations must stay consistent. → Chapter 29

**Follow-up:** Which do you want for a repository?</details>

**Q274.** *Senior* — Why is a stubbed repository risky?
<details><summary>Answer</summary>Nothing connects `save` to the next `findById`, so a test passes while the code saves the wrong thing. A fake catches that, and the same fake serves the whole suite. → Chapter 29

**Follow-up:** What can a fake assert that a stub cannot?</details>

**Q275.** *Senior* — What is wrong with mocking every collaborator?
<details><summary>Answer</summary>The test becomes a restatement of the implementation: it fails on any refactor that changes nothing observable, and passes on any behaviour change that preserves the call sequence. Assert on outcomes wherever an outcome is observable. → Chapter 29

**Follow-up:** Give a behaviour a mock would hide entirely.</details>

**Q276.** *Senior* — Name a behaviour only a fake can pin down.
<details><summary>Answer</summary>A deliberately detached side effect. A plain mock resolves by default and nobody writes a rejection for a path they are not thinking about, so the suite stays green if someone converts `void this.notifications…` to `await`, silently turning a mail outage into a request outage. A fake with a failure switch makes it testable. → Chapter 29

**Follow-up:** What other design decisions are easy to reverse by accident?</details>

**Q277.** *Mid* — Name four ways a test passes while the code is broken.
<details><summary>Answer</summary>Asserting the mock rather than the resulting state; testing only the happy path where the failure mode is *accepting* bad input; faking so deep that you verify your own mock's fluent chain; and forgetting `await` on a rejection assertion, which passes unconditionally. → Chapter 29

**Follow-up:** Which of those does coverage report?  (None.)</details>

**Q278.** *Mid* — Why do validation and authorization need rejection tests specifically?
<details><summary>Answer</summary>Both fail open — the bug is that something is accepted — so a happy-path test passes whether or not the check exists. Only a negative test detects a missing decorator or guard. → Chapter 29

**Follow-up:** What is the cheapest way to write those?</details>

**Q279.** *Mid* — How do you test a guard without HTTP?
<details><summary>Answer</summary>Construct it directly and pass a plain object implementing only the `ExecutionContext` methods it uses — `switchToHttp().getRequest()`, `getHandler()`, `getClass()` — with a stubbed `Reflector`. Orders of magnitude faster than driving a full application. → Chapter 29

**Follow-up:** What about a pipe?</details>

**Q280.** *Senior* — Why test `ValidationPipe` directly rather than over HTTP?
<details><summary>Answer</summary>An end-to-end test posting a *valid* payload passes identically whether or not `@Type()` is present, because the failure mode is accepting bad input. Instantiating the pipe and calling `transform` is fast enough to run every rule as a unit test, keeps the assertion next to the DTO, and makes a removed decorator fail immediately. → Chapter 14

**Follow-up:** What does that suggest about where fail-open bugs get caught?</details>

**Q281.** *Mid* — How do you know your test suite is any good?
<details><summary>Answer</summary>Break the code deliberately and check that a test fails. Mutation testing automates it. Coverage measures lines executed, not assertions made — a 92% suite can execute the validation path without ever asserting rejection. → Chapter 29

**Follow-up:** What would you mutate first?</details>

**Q282.** *Mid* — Why does `module.get()` throw for a scoped provider?
<details><summary>Answer</summary>There is no single instance. Use `await module.resolve(token)`, which creates one per context id. → Chapter 29

**Follow-up:** What else in the testing module surprises people?</details>

**Q283.** *Mid* — Why does adding a global guard break every controller test?
<details><summary>Answer</summary>`APP_GUARD` is part of the module graph, so a testing module gets it too — which is exactly why it is testable, and why `overrideGuard` exists. → Chapter 29

**Follow-up:** Should you override it everywhere?</details>

**Q284.** *Senior* — Why is overriding the auth guard in every e2e test a mistake?
<details><summary>Answer</summary>It neuters the tests that matter most. Authorization fails open, so a route that lost its guard entirely would still look protected — and "we forgot `@Public()`" becomes indistinguishable from "we forgot the guard". Use real tokens and keep explicit 401 and 403 cases. → Chapter 30

**Follow-up:** What is the cost of real tokens?  (A `jwt.sign` per test.)</details>

**Q285.** *Junior* — What does `app.init()` do that `listen()` does not?
<details><summary>Answer</summary>Everything except bind a socket — the graph is built, providers instantiated, hooks run, routes registered. Supertest then drives `app.getHttpServer()` directly. → Chapter 30

**Follow-up:** What must you remember in teardown?</details>

**Q286.** *Mid* — What happens if you forget `app.close()`?
<details><summary>Answer</summary>The database pool and any timers stay open, so the runner reports "did not exit" and hangs — in CI, a job burning its whole timeout for a suite that actually passed. → Chapter 30

**Follow-up:** What is the correct teardown order with a container?</details>

**Q287.** *Mid* — Why does validation work in production and not in an e2e test?
<details><summary>Answer</summary>The pipe was registered in `main.ts`, which never runs in a testing module — so the test exercises a different pipeline than production. `APP_PIPE` puts it in the graph, where the test gets it automatically. → Chapter 30

**Follow-up:** What is the general form of that argument?</details>

**Q288.** *Mid* — Why is SQLite a poor stand-in for PostgreSQL in tests?
<details><summary>Answer</summary>Different types, different constraint behaviour, no `RETURNING`, no `SKIP LOCKED` — so tests pass and production breaks on exactly the semantics integration tests exist to verify. Use Testcontainers. → Chapter 30

**Follow-up:** What does that cost?</details>

**Q289.** *Mid* — Why apply migrations rather than `synchronize` in test setup?
<details><summary>Answer</summary>So the test exercises the same schema path production uses, and a broken migration fails in CI rather than during a deploy. → Chapter 30

**Follow-up:** What must you exclude when truncating between tests?</details>

**Q290.** *Senior* — An integration suite fails roughly one run in ten, on a different test each time. Diagnose.
<details><summary>Answer</summary>Flakiness that moves between tests is shared state, not a bug in any one test. Check parallelism against the database first — Jest's default workers truncating a common schema produces exactly this. Then look for state outliving a test: a global, an uncleared cache, sequence values reused because truncation did not restart identity, real timers. Running with one worker confirms it; the fix is a schema or container per worker rather than serialising forever. → Chapter 30

**Follow-up:** Why not just retry flaky tests?</details>

**Q291.** *Mid* — How do you test that an endpoint is idempotent under concurrency?
<details><summary>Answer</summary>Fire the two requests with `Promise.all` and assert both succeed, return the same id, and produced one row. A check-then-insert implementation passes when they run sequentially and fails here — which is the point. → Chapter 30

**Follow-up:** What does a sequential version of that test prove?  (Nothing about concurrency.)</details>

**Q292.** *Mid* — Why assert against the database as well as the response?
<details><summary>Answer</summary>A handler can return a correct-looking body without persisting anything, or persist something different from what it returned. → Chapter 30

**Follow-up:** Which endpoints most need that?</details>

**Q293.** *Senior* — What belongs in an e2e test and what does not?
<details><summary>Answer</summary>Worth it: wiring — routes exist, the pipeline applies in the right order, statuses are right, real SQL runs, migrations apply, authorization is enforced. Not worth it: business-rule permutations, which belong in unit tests against a fake. One e2e test per endpoint plus a few full journeys; depth comes from the unit suite. → Chapter 30

**Follow-up:** What does an all-e2e strategy cost you?</details>

**Q294.** *Senior* — A teammate wants to replace the unit suite with e2e tests, since "they test what users do". Respond.
<details><summary>Answer</summary>The premise is right and the conclusion does not follow. E2E tests are the only ones that catch wiring, and far too slow to enumerate branches — twenty pricing cases become twenty database round trips. They also localise poorly: a failure says the endpoint is wrong, not which branch. Keep both with different jobs. The inverse failure is equally real: a large unit suite with no e2e coverage passes happily while a route is unregistered. → Chapter 30

**Follow-up:** How would you split them in CI?</details>

**Q295.** *Senior* — Your suite takes eleven minutes and developers stopped running it. What do you do?
<details><summary>Answer</summary>Find where the time goes before optimising — it is usually a few tests importing `AppModule`, connecting to a real database, or awaiting real timers. Reclassify those as integration tests behind a separate command, leaving a fast unit suite people run on save. Replace real waits with fake timers and container setup with plain construction. The goal is two suites with different contracts: one under a few seconds because it runs constantly, one that may take minutes because it runs on push. → Chapter 29

**Follow-up:** What is the risk of splitting them?  (The slow one stops being run.)</details>

---

## 12. Performance and Production

**Q296.** *Junior* — An endpoint is slow. What do you do first?
<details><summary>Answer</summary>Reproduce it and measure where the time goes. The framework pipeline costs tens of microseconds, so if the endpoint takes 400 ms it is 0.01% of the problem. The usual causes are N+1 queries, a missing index, an unbounded payload, or event-loop blocking. → Chapter 34

**Follow-up:** Which tool tells you which of those it is?</details>

**Q297.** *Mid* — Which measurement answers which question?
<details><summary>Answer</summary>`clinic doctor` classifies the bottleneck — event loop, GC, I/O, or an external dependency — which tells you what to use next. autocannon establishes the throughput ceiling. `--cpu-prof` localises CPU time. → Chapter 34

**Follow-up:** What must you never benchmark?</details>

**Q298.** *Junior* — Why not benchmark `start:dev`?
<details><summary>Answer</summary>It runs the TypeScript watcher and development-only middleware, so you measure the tooling. Benchmark a production build with `NODE_ENV=production`. → Chapter 34

**Follow-up:** What else does watch mode hide?  (A failing build — it keeps serving the last good output.)</details>

**Q299.** *Mid* — Why does synchronous CPU work affect every concurrent request?
<details><summary>Answer</summary>Node runs all your JavaScript on one thread, so a 200 ms synchronous operation adds 200 ms to every request in flight, not just the one executing it. → Chapter 34

**Follow-up:** Does adding replicas help?</details>

**Q300.** *Senior* — Why won't more replicas fix a CPU-bound endpoint?
<details><summary>Answer</summary>Each replica has the same single thread and the same blocking work. The fixes are a worker thread pool, a queue with a separate worker process, or pushing the computation into the database. → Chapter 34

**Follow-up:** How do you detect the blocking in the first place?</details>

**Q301.** *Mid* — Why is event-loop lag the best early warning?
<details><summary>Answer</summary>It rises before latency and before memory, because requests queue behind blocking work before any of them time out. `monitorEventLoopDelay` makes it a gauge you can alert on. → Chapter 34

**Follow-up:** What p99 lag would concern you?</details>

**Q302.** *Mid* — Name the less obvious event-loop blockers.
<details><summary>Answer</summary>Synchronous crypto, `JSON.stringify` on very large objects, `zlib` and `fs` sync methods, sorting very large arrays, and a catastrophically backtracking regex — which is a denial-of-service vector rather than just a slow function. → Chapter 34

**Follow-up:** Which of those is user-controllable?</details>

**Q303.** *Mid* — When is the Fastify adapter worth it?
<details><summary>Answer</summary>When responses are large, throughput is high, and handlers are thin — it speeds up routing and serialization. If a request spends 5 ms in the framework and 200 ms in the database, halving the 5 ms changes nothing measurable. → Chapter 34

**Follow-up:** What does migrating late cost?</details>

**Q304.** *Senior* — What breaks when you scale from one replica to three?
<details><summary>Answer</summary>In-memory caches, rate limits, WebSocket broadcasts, and presence all become per-process and silently wrong. Scheduled jobs run three times. And connection pools multiply — a pool of 20 across ten replicas is 200 connections against a default `max_connections` of 100, so the app fails to connect at exactly the moment you scale to handle load. → Chapter 34

**Follow-up:** Which of those fails loudly?  (Only the connection limit.)</details>

**Q305.** *Mid* — What is the pool arithmetic you need to check?
<details><summary>Answer</summary>`max_connections ≥ replicas × pool_size + headroom`. Beyond a handful of replicas, put a pooler like PgBouncer in front rather than raising the database's limit — server-side connections are expensive. → Chapter 34

**Follow-up:** What does exceeding it look like?</details>

**Q306.** *Mid* — Why does keep-alive matter on outbound HTTP?
<details><summary>Answer</summary>Without it every call to a downstream service pays a fresh TCP and TLS handshake, often more time than the call itself. Configure agents with `keepAlive: true` — and always a timeout. → Chapter 34

**Follow-up:** What else should bound an outbound call?</details>

**Q307.** *Senior* — Why is caching a slow query dangerous?
<details><summary>Answer</summary>The underlying cost is unchanged, so every cold miss reproduces the original latency — and a deploy, a cache flush, or a cold key at peak reproduces it at the worst moment. By then the cache is load-bearing enough that nobody can remove it to investigate. Fix the query, then the blocking, then cache. → Chapter 34

**Follow-up:** So when is a cache introduced during an incident acceptable?</details>

**Q308.** *Mid* — What must a cache key contain?
<details><summary>Answer</summary>Every input the response varies by — the principal, the tenant, the API version, the serialization group, and any header that changes the body. Omitting the principal serves one user's data to another. → Chapter 20

**Follow-up:** Why is that easy to get wrong in an interceptor?</details>

**Q309.** *Mid* — Why is a cross-user cache collision worse than staleness?
<details><summary>Answer</summary>Staleness shows wrong-but-authorised data; a key collision shows another user's data. One is a bug, the other is a security incident. → Chapter 20

**Follow-up:** How would you make that mistake hard to express?</details>

**Q310.** *Mid* — Why must a cache failure not fail the request?
<details><summary>Answer</summary>Otherwise the cache reduces availability — an optional optimisation becomes a hard dependency, and a Redis outage becomes an application outage. Wrap reads and writes and degrade to the source of truth. → Chapter 20

**Follow-up:** What is the second-order failure if you do?</details>

**Q311.** *Mid* — Why jitter TTLs?
<details><summary>Answer</summary>Keys written together otherwise expire together across every replica, producing a synchronised stampede — the load the cache was absorbing, handed to the database in one spike. → Chapter 20

**Follow-up:** What else collapses a stampede?</details>

**Q312.** *Junior* — What does `enableShutdownHooks()` change, and what breaks without it?
<details><summary>Answer</summary>It attaches listeners for SIGTERM, SIGINT, SIGBREAK, and SIGHUP so termination hooks run. Without it the process dies immediately: no pool drain, no in-flight completion, and every termination hook you wrote is dead code. In Kubernetes that presents as a burst of 502s on every deploy. → Chapter 9

**Follow-up:** Why is it opt-in rather than default?</details>

**Q313.** *Senior* — Describe a shutdown that drops no requests.
<details><summary>Answer</summary>Fail readiness immediately on SIGTERM so the load balancer stops routing, wait longer than its detection interval — a fixed sleep, since the process cannot know when removal completed — then drain and close. Set `terminationGracePeriodSeconds` above that wait. The fixed sleep looks like a hack and is the only honest implementation. → Chapter 3

**Follow-up:** Why can't the app detect when it has been removed from rotation?</details>

**Q314.** *Mid* — What is the difference between liveness and readiness?
<details><summary>Answer</summary>Failing liveness gets the container killed; failing readiness removes it from rotation temporarily. So liveness must check almost nothing, and dependency checks belong in readiness. → Chapter 33

**Follow-up:** What happens if you put a database check in liveness?</details>

**Q315.** *Senior* — Why is a database check in liveness catastrophic?
<details><summary>Answer</summary>A thirty-second database blip fails liveness on every replica simultaneously, the orchestrator kills them all, and the restart storm hits a database already struggling — converting a recoverable incident into an outage that outlasts its cause. The test is: would restarting fix this? A wedged event loop, yes; anything outside the process, no. → Chapter 33

**Follow-up:** What should liveness check, then?</details>

**Q316.** *Mid* — What makes logs queryable?
<details><summary>Answer</summary>Structured fields rather than interpolated strings, so you can filter and aggregate on any of them. Pretty-printing in production costs throughput and emits text the platform cannot parse into fields. → Chapter 33

**Follow-up:** What must never appear in them?</details>

**Q317.** *Mid* — What makes a correlation id actually useful?
<details><summary>Answer</summary>Accepting it from the caller when present, echoing it in the response so a user can quote it, and propagating it into queue jobs and outbound calls — otherwise the trail ends at your process boundary. → Chapter 33

**Follow-up:** How does a worker re-establish it?</details>

**Q318.** *Senior* — Why is high metric label cardinality a production hazard?
<details><summary>Answer</summary>Every distinct label combination is a separate time series, so labelling by user id or raw URL creates millions and the metrics backend degrades or rejects writes. Use the route *pattern*. If you need per-user detail, that is a log, not a metric. → Chapter 33

**Follow-up:** Why prefer histograms to summaries?</details>

**Q319.** *Senior* — Why must OpenTelemetry be initialised first?
<details><summary>Answer</summary>Auto-instrumentation patches modules as they are loaded, so importing it after `AppModule` means HTTP, database, and Redis calls are never traced — and the failure is silent. You get traces containing only the spans you created by hand. → Chapter 33

**Follow-up:** How would you notice?</details>

**Q320.** *Senior* — p50 is fine and p99 is terrible. How do you find the cause?
<details><summary>Answer</summary>The divergence means a subset behaves differently, so aggregate metrics cannot localise it. Configure tracing to always sample requests over a latency threshold, giving traces for exactly that population rather than a random sample dominated by fast requests. Correlate with event-loop lag: if the tail tracks lag spikes, the cause is blocking work and no database tuning will help. Then check cold caches, a rare-relation N+1, pool exhaustion, and GC pauses. Make the instrumentation conditional on the symptom. → Chapter 34

**Follow-up:** Why does uniform sampling under-represent this?</details>

---

## 13. Architecture and Judgement

**Q321.** *Senior* — When would you *not* use NestJS?
<details><summary>Answer</summary>A single-purpose serverless function, where booting the module graph is real cold-start latency. A prototype you will throw away. A team unwilling to use TypeScript, since the container reads emitted type metadata. And a service that is one hot path, where the pipeline costs more than it returns. Nest earns its cost with multiple developers, a schema, and a multi-year horizon. → Chapter 1

**Follow-up:** What would you use for a high-throughput webhook receiver?</details>

**Q322.** *Senior* — Should folders be grouped by layer or by feature?
<details><summary>Answer</summary>By feature. Layer grouping means a feature change touches four directories with nothing showing which files belong together, and a layer folder has no meaningful public surface — which is what makes a module's `exports` real. → Chapter 35

**Follow-up:** What is the first sign the boundaries are wrong?</details>

**Q323.** *Senior* — How do you know a module boundary is wrong?
<details><summary>Answer</summary>You cannot state the module's job in one sentence without "and", or a typical feature change touches five modules. The common cause is drawing boundaries around entities rather than use cases, so everything imports everything. → Chapter 35

**Follow-up:** What does a cycle between two modules tell you?</details>

**Q324.** *Senior* — When does hexagonal architecture pay for itself?
<details><summary>Answer</summary>Where the domain has genuine invariants — a state machine, rules a non-programmer would recognise, several entry points that must all obey them. For CRUD it produces an anaemic domain object mapped to and from an ORM entity of identical shape, with a mapper maintained forever. → Chapter 35

**Follow-up:** What is the most common half-adoption?</details>

**Q325.** *Senior* — Is it acceptable for one module to be hexagonal and the rest layered?
<details><summary>Answer</summary>Yes, and it is usually right. Uniform structure across modules of very different complexity means either over-engineering the simple ones or under-protecting the hard one, and the second is more expensive. The inconsistency is self-explaining: a reader seeing `domain/` and `ports/` correctly infers there is complexity worth protecting. → Chapter 35

**Follow-up:** How would you document that choice?</details>

**Q326.** *Senior* — What are the signs an architecture costs more than it returns?
<details><summary>Answer</summary>A trivial change touching many files — one column crossing a DTO, a domain entity, an ORM entity, a mapper, a port, and an adapter. New developers routinely asking where logic lives. Abstractions with one implementation and no realistic second. Mappers that are identity functions. → Chapter 35

**Follow-up:** What is the honest default for most applications?</details>

**Q327.** *Senior* — When are microservices justified?
<details><summary>Answer</summary>A concrete driver: independent scaling where one component needs radically different capacity, team autonomy where separate release cadences are blocked, fault isolation, or a genuine technology difference. "More scalable" is not one — a stateless monolith scales horizontally too. → Chapter 26

**Follow-up:** What do you give up on day one?</details>

**Q328.** *Senior* — Two teams block each other in one codebase. Is splitting the answer?
<details><summary>Answer</summary>Diagnose the blocking first. Merge conflicts and shared files mean the boundaries are wrong — fix with module ownership, enforced import direction, and a codeowners file. Release blocking means separate deployables, which a monorepo provides without distributed systems. Only *runtime* interference — one team's load degrading the other's service — justifies separate services. Splitting for organisational reasons while leaving the coupling produces a distributed monolith: the same blocking, now over the network. → Chapter 35

**Follow-up:** How would you tell which one you have?</details>

**Q329.** *Senior* — When is CQRS worth it, and when is it cargo cult?
<details><summary>Answer</summary>Worth it when read and write models genuinely diverge and the read side needs independent optimisation or scaling. Cargo cult for CRUD, where a command class, a handler, a registration, and a bus dispatch replace one service method and nothing is gained. Separate the read and write *code* first; separate storage only when a measured asymmetry justifies it. → Chapter 28

**Follow-up:** Does CQRS imply event sourcing?</details>

**Q330.** *Senior* — Give the honest costs of event sourcing.
<details><summary>Answer</summary>Schema evolution is permanent — an event from three years ago must remain readable forever, so you maintain upcasters. Every query needs its own projection. Deletion conflicts directly with a right to erasure, and crypto-shredding must be designed in from the start. And debugging changes shape, since current state is a computation rather than a row. Use it where the audit trail *is* the product. → Chapter 28

**Follow-up:** What gives you an audit trail far more cheaply?</details>

**Q331.** *Senior* — A dashboard needs a six-table join and is slowing everything down. Is CQRS the answer?
<details><summary>Answer</summary>Maybe, and cheaper options come first: a materialised view on a schedule, or a denormalised summary table updated by a job. Both give a read shape independent of the write shape — the useful part of CQRS — for almost nothing. Full CQRS earns its cost when many read shapes diverge and the read side must scale or deploy separately. Decide the tolerable staleness first; that determines whether a five-minute refresh suffices. → Chapter 28

**Follow-up:** What would make you skip all of that?</details>

**Q332.** *Senior* — Should every module have a `forRoot()`, for consistency?
<details><summary>Answer</summary>No. A dynamic module exists to let a consumer vary behaviour it does not own. Most modules have one consumer and nothing to vary, so `forRoot()` adds an options object, a token, and an async variant to maintain in exchange for indirection — and you lose the signal that `forRoot()` in someone else's module means it was *built* to be configured. → Chapter 4

**Follow-up:** What is the right axis for that decision?</details>

**Q333.** *Senior* — Forty modules, and a feature touches six. A colleague proposes making the five most-used `@Global()`. Respond.
<details><summary>Answer</summary>It treats a symptom and destroys the diagnostic. Six modules per feature means boundaries drawn around layers or entities, and hiding the imports makes that harder to see. Globals also break tests specifically: the test module builds a fresh graph without them, so a provider available everywhere in production is missing in isolation. An `InfrastructureModule` re-exporting the common set gives the same one-line import while keeping the dependency visible. → Chapter 4

**Follow-up:** What is the legitimate use of `@Global()`?</details>

**Q334.** *Senior* — Would you migrate an existing TypeORM codebase to Prisma for performance?
<details><summary>Answer</summary>Almost certainly not. ORM performance problems are overwhelmingly N+1 queries, missing indexes, unbounded pages, and over-fetching — properties of how queries are written, all of which reproduce identically in Prisma. Measure first. A rewrite also carries risk unrelated to performance, since every query is retranslated by hand and subtle null-handling and transaction differences introduce bugs the tests do not cover. If the ORM genuinely is the bottleneck, use raw SQL for those queries. → Chapter 19

**Follow-up:** What would change your answer?</details>

**Q335.** *Senior* — How would you approach the v12 upgrade?
<details><summary>Answer</summary>Prepare on v11 and adopt late. Removing `require` and `__dirname`, moving to `NodeNext` resolution, and auditing dependencies for ESM support are all v11-compatible and constitute most of the work. Run the prereleases on a branch to find what breaks in *your* code, and adopt after a few patch versions when your ecosystem packages have compatible releases. Being early on a major buys nothing that being second does not, and the risk is one-sided. → Chapter 36

**Follow-up:** Which pillar costs the most?</details>

**Q336.** *Senior* — Should you migrate a working `class-validator` codebase to Zod when v12 lands?
<details><summary>Answer</summary>Not as a project in itself. Validation is a layer where regressions fail open — an accidentally weakened rule accepts bad data silently — so a wholesale rewrite carries a high-consequence risk for incremental benefit, and both approaches will be supported. Use Zod for new endpoints where inference genuinely helps: deeply nested payloads, schemas shared with a TypeScript frontend. If you do convert one, keep the old tests and add rejection tests for every rule. → Chapter 36

**Follow-up:** What is Zod's genuine advantage?</details>

**Q337.** *Senior* — Traffic will grow tenfold in three months. What do you check now?
<details><summary>Answer</summary>What breaks non-linearly, not what is currently slow. Connection arithmetic first — it hits a hard limit and fails abruptly. Then anything holding in-memory state: caches, rate limits, socket fan-out, schedulers, each correct today and silently wrong at more replicas. Then unbounded queries, fine at current data volumes and table scans later. Load-test at 10× against production-sized data, since most scaling failures are data-volume failures. And confirm the observability you would need during an incident already exists. → Chapter 34

**Follow-up:** Why production-sized data specifically?</details>

**Q338.** *Senior* — A manager asks you to "make the API 2× faster". How do you respond?
<details><summary>Answer</summary>Convert it into something measurable first — "the API" does not have a speed. Which endpoints, at which percentile, under what load, and is the complaint from users, dashboards, or cost? Measure the current state so improvement is demonstrable, and rank by traffic × latency, since halving a rarely-used endpoint achieves nothing. Usually two or three endpoints account for most of the pain, with the same causes every time. → Chapter 34

**Follow-up:** What happens if you skip that step?</details>

**Q339.** *Senior* — What is the most consequential mistake you have seen in a NestJS codebase?
<details><summary>Answer</summary>A good answer names something whose failure mode is silent. Strong candidates: request state on a singleton service, which corrupts across concurrent requests and only reproduces under load; a cache key without the principal, which leaks data rather than breaking a feature; and validation failing open through a missing `@Type()`, which returns 201 for input that should have been rejected. What they share is that nothing errors and no test notices.

**Follow-up:** How would you prevent it recurring, structurally?</details>

**Q340.** *Senior* — What would you change about NestJS?
<details><summary>Answer</summary>An opinionated answer is the point. Defensible ones: validation failing open is the wrong default for a security-relevant layer; `enableShutdownHooks()` being opt-in means most applications silently have dead lifecycle hooks; request scope bubbling silently, with no warning at startup that a subtree was promoted; and `@Injectable()` doing something entirely different from what its name implies. Each is a case where the framework's default is the less safe option. → Chapter 1

**Follow-up:** Which of those would you actually change, given backwards compatibility?</details>

---

**Next:** [Appendix B — 112 Coding Challenges →](B-coding-challenges.md)
