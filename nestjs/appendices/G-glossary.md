# Appendix G — Glossary

Every term used in this book, alphabetical, with a chapter cross-reference where one applies.

[A](#a) · [B](#b) · [C](#c) · [D](#d) · [E](#e) · [F](#f) · [G](#g) · [H](#h) · [I](#i) ·
[J](#j) · [K](#k) · [L](#l) · [M](#m) · [N](#n) · [O](#o) · [P](#p) · [Q](#q) · [R](#r) ·
[S](#s) · [T](#t) · [U](#u) · [V](#v) · [W](#w) · [Z](#z)

---

## A

**ABAC (Attribute-Based Access Control)** — Authorization decided from attributes of the subject, the resource, the action, and the environment, rather than from roles alone. "May edit if they own it and it is not archived." → [Ch 22](../chapters/22-authorization.md)

**Abstract class (as DI token)** — A class with abstract members, used as both a contract and an injection token. Unlike an interface it survives compilation, so the container can resolve it. → [Ch 2](../chapters/02-typescript-decorators.md), [Ch 7](../chapters/07-dependency-injection.md)

**Adapter (hexagonal)** — An implementation of a port, living in the infrastructure layer. Adapters depend on the domain; the domain never depends on them. → [Ch 35](../chapters/35-architecture-patterns.md)

**Adapter (platform)** — The layer wrapping Express or Fastify behind one interface, so Nest code is mostly platform-agnostic. It leaks for raw request/response objects, middleware signatures, and uploads. → [Ch 1](../chapters/01-what-nestjs-is.md)

**Argon2id** — The current recommended password-hashing algorithm: deliberately slow and memory-hard, resisting both GPU and side-channel attacks. → [Ch 21](../chapters/21-authentication.md)

**ArgumentMetadata** — The object a pipe receives describing what it is transforming: `type` (`body`/`query`/`param`/`custom`), `metatype` (the declared class), and `data` (the decorator's argument). → [Ch 14](../chapters/14-pipes-validation.md)

**ArgumentsHost** — The transport-agnostic parent of `ExecutionContext`, given to exception filters. Exposes `switchToHttp()`, `switchToWs()`, `switchToRpc()`, and `getType()`. → [Ch 15](../chapters/15-exception-filters.md)

**AsyncLocalStorage** — A Node API keeping a store alive for an async execution chain, used to carry per-request context without changing any provider's scope. → [Ch 9](../chapters/09-scopes-lifecycle.md)

**At-least-once delivery** — The guarantee brokers and queues actually provide: a message may be delivered more than once, so handlers must be idempotent. → [Ch 26](../chapters/26-microservices.md), [Ch 27](../chapters/27-queues-jobs.md)

**Authentication** — Establishing who the caller is. Distinct from authorization, and best kept in a separate guard. → [Ch 21](../chapters/21-authentication.md)

**Authorization** — Deciding whether an authenticated caller may perform an action on a resource. → [Ch 22](../chapters/22-authorization.md)

**autoLoadEntities** — A TypeORM option making entities registered via `forFeature` available to the root connection automatically. → [Ch 17](../chapters/17-typeorm.md)

---

## B

**Backpressure** — A stream's mechanism for signalling that a destination cannot keep up. `pipeline` respects it; a manual `data`/`write` loop does not, and memory grows unbounded. → [Ch 31](../chapters/31-file-uploads-streaming.md)

**Bounded context** — A boundary within which a model has one consistent meaning. The right unit for a Nest module, in preference to one module per entity. → [Ch 35](../chapters/35-architecture-patterns.md)

**bufferLogs** — A `NestFactory.create` option holding log output produced during bootstrap until a custom logger is attached, so early messages are not lost. → [Ch 3](../chapters/03-first-application.md), [Ch 33](../chapters/33-logging-observability.md)

**BullMQ** — A Redis-backed job queue with retries, delayed jobs, repeatable jobs, and rate limiting. Integrated via `@nestjs/bullmq`. → [Ch 27](../chapters/27-queues-jobs.md)

---

## C

**Cache stampede** — Many concurrent requests missing the same expired key at once, all running the expensive computation the cache was absorbing. Mitigated by single-flight and jittered TTLs. → [Ch 20](../chapters/20-caching.md)

**CallHandler** — The object passed to an interceptor whose `handle()` invokes the route handler and returns an Observable of its result. Not calling it skips the handler entirely. → [Ch 13](../chapters/13-interceptors.md)

**CanActivate** — The interface a guard implements. Returns a boolean, promise, or Observable deciding whether the request proceeds. → [Ch 12](../chapters/12-guards.md)

**CASL** — A library expressing permissions as data, including conditions, and translating the same rules into query filters so instance checks and list filters cannot drift. → [Ch 22](../chapters/22-authorization.md)

**class-transformer** — The library behind `@Type()`, `@Exclude()`, and `@Expose()`. Converts plain objects to class instances and controls serialization. → [Ch 13](../chapters/13-interceptors.md), [Ch 14](../chapters/14-pipes-validation.md)

**class-validator** — The decorator-based validation library `ValidationPipe` uses. Requires DTOs to be classes, since decorators need a runtime target. → [Ch 14](../chapters/14-pipes-validation.md)

**ClassSerializerInterceptor** — The interceptor applying `class-transformer` decorators to a handler's return value. Requires a real class instance; a plain object passes through untouched. → [Ch 13](../chapters/13-interceptors.md)

**Code-first (GraphQL)** — Generating the SDL schema from decorated TypeScript classes, rather than writing SDL and generating types. The better fit for NestJS. → [Ch 24](../chapters/24-graphql.md)

**Command (CQRS)** — An object representing an intent to change state, dispatched through a `CommandBus` to exactly one handler. → [Ch 28](../chapters/28-cqrs-events.md)

**CommonJS (CJS)** — Node's original module system, using `require` and `module.exports`. NestJS 11 targets it; v12 migrates to ESM. → [Ch 36](../chapters/36-nestjs-11-to-12.md)

**Compensating action** — A command that reverses an already-committed step, used when a distributed process fails and rollback is impossible. → [Ch 28](../chapters/28-cqrs-events.md)

**ConfigurableModuleBuilder** — A helper generating `forRoot`/`forRootAsync` (or `register`/`registerAsync`) and an options token for a dynamic module. → [Ch 4](../chapters/04-modules.md)

**ConfigService** — The service exposing validated configuration. `get()` returns `T | undefined`; `getOrThrow()` removes `undefined` from the type. → [Ch 16](../chapters/16-configuration.md)

**ContextIdFactory** — The factory creating context identifiers used to resolve request-scoped providers programmatically, and the hook point for durable providers. → [Ch 9](../chapters/09-scopes-lifecycle.md)

**Controller** — A class translating HTTP requests into method calls. Should contain no business logic. → [Ch 5](../chapters/05-controllers-routing.md)

**Correlation ID** — An identifier shared by every log line, span, and downstream call belonging to one request. Accepted from the caller when present, echoed to the client, and propagated into queues. → [Ch 33](../chapters/33-logging-observability.md)

**CORS** — The browser mechanism controlling which origins may make credentialed cross-origin requests. `origin: true` permits every origin and is a common production mistake. → [Ch 23](../chapters/23-security.md)

**CQRS** — Separating the model that changes state from the model that reads it. Separate databases and event sourcing are optional extensions, not the pattern. → [Ch 28](../chapters/28-cqrs-events.md)

**Cron expression** — A five- or six-field schedule string. In Nest, supplied to `@Cron()` along with an explicit time zone. → [Ch 32](../chapters/32-task-scheduling.md)

**CSRF** — An attack exploiting credentials the browser attaches automatically. Relevant to cookie authentication, not to bearer tokens. → [Ch 23](../chapters/23-security.md)

**Custom provider** — A provider defined with `useClass`, `useValue`, `useFactory`, or `useExisting` rather than the class shorthand. → [Ch 8](../chapters/08-custom-providers.md)

---

## D

**DataLoader** — A batching and deduplicating loader that collapses many per-parent lookups made in one tick into a single query. Essential in GraphQL, and must be request-scoped. → [Ch 24](../chapters/24-graphql.md)

**DataSource** — TypeORM's connection object, exposing repositories, migrations, and `transaction()`. → [Ch 17](../chapters/17-typeorm.md), [Ch 18](../chapters/18-migrations-transactions.md)

**Dead-letter queue** — Where messages that exhausted their retries are sent, so a permanently failing message does not loop forever. → [Ch 26](../chapters/26-microservices.md), [Ch 27](../chapters/27-queues-jobs.md)

**Decorator** — A function that receives what it decorates and usually records metadata about it. In Nest, decorators almost never change behaviour directly. → [Ch 2](../chapters/02-typescript-decorators.md)

**Dependency injection (DI)** — Supplying a class's collaborators from outside rather than letting it construct or import them. The idea NestJS is built around. → [Ch 7](../chapters/07-dependency-injection.md)

**design:paramtypes** — The metadata key holding a constructor's parameter types as actual class references. Emitted only for declarations carrying at least one decorator. → [Ch 2](../chapters/02-typescript-decorators.md)

**Distributed lock** — A cluster-wide mutual exclusion, typically `SET NX PX` in Redis, ensuring one instance runs a scheduled job. Requires compare-and-delete release and renewal. → [Ch 32](../chapters/32-task-scheduling.md)

**DTO (Data Transfer Object)** — A class describing a request or response shape. Must be a class, not an interface, because validation and documentation need a runtime target. → [Ch 14](../chapters/14-pipes-validation.md)

**Durable provider** — A request-scoped provider whose dependency subtree is reused across requests sharing an attribute, usually a tenant. Long-lived, so it must stay stateless. → [Ch 9](../chapters/09-scopes-lifecycle.md)

**Dynamic module** — A module configured by its consumer through a static method returning a `DynamicModule`. The mechanism behind `forRoot()` and `register()`. → [Ch 4](../chapters/04-modules.md)

---

## E

**Eager loading** — Loading a relation on every query touching an entity. Avoid `eager: true`: it cannot be opted out of per query. → [Ch 17](../chapters/17-typeorm.md)

**emitDecoratorMetadata** — The TypeScript flag emitting type information as metadata for decorated declarations. Without it, constructor injection cannot work. → [Ch 2](../chapters/02-typescript-decorators.md)

**Entity** — A class mapped to a database table. Distinct from a domain entity in hexagonal architecture, where the two are deliberately separate. → [Ch 17](../chapters/17-typeorm.md), [Ch 35](../chapters/35-architecture-patterns.md)

**ESM (ECMAScript Modules)** — The standard module system using `import`/`export`. NestJS 12 migrates all official packages to it. → [Ch 36](../chapters/36-nestjs-11-to-12.md)

**Event pattern** — A fire-and-forget message with no reply, declared with `@EventPattern` and emitted with `client.emit()`. The default choice over request–response. → [Ch 26](../chapters/26-microservices.md)

**Event sourcing** — Storing the sequence of events as the source of truth and deriving state by replay. Independent of CQRS, and far more expensive than it appears. → [Ch 28](../chapters/28-cqrs-events.md)

**EventBus** — The `@nestjs/cqrs` in-memory event dispatcher. Decouples code; provides no delivery guarantee. → [Ch 28](../chapters/28-cqrs-events.md)

**Exception filter** — A class catching thrown exceptions and writing the response. Runs only on the throw path. → [Ch 15](../chapters/15-exception-filters.md)

**ExecutionContext** — The object exposing the target class and handler plus the transport, given to guards and interceptors. What separates a guard from middleware. → [Ch 10](../chapters/10-request-lifecycle.md), [Ch 12](../chapters/12-guards.md)

**experimentalDecorators** — The TypeScript flag enabling legacy decorators. Required by Nest, because the TC39 standard proposal has no parameter decorators. → [Ch 2](../chapters/02-typescript-decorators.md)

**Expand-migrate-contract** — The safe sequence for a schema change under rolling deploys: add the new shape, write both, backfill, switch reads, then remove the old. → [Ch 18](../chapters/18-migrations-transactions.md)

---

## F

**Fake** — A working test double with a shortcut, such as an in-memory repository. Preferred over stubs for repositories because reads and writes stay consistent. → [Ch 29](../chapters/29-unit-testing.md)

**Fastify** — An alternative HTTP platform, roughly twice as fast as Express on thin JSON handlers. Migrating late means rewriting middleware, uploads, and raw responses. → [Ch 1](../chapters/01-what-nestjs-is.md), [Ch 34](../chapters/34-performance-scaling.md)

**Field resolver** — A GraphQL resolver computing one field, running once per parent. The structural source of N+1 in GraphQL. → [Ch 24](../chapters/24-graphql.md)

**FileTypeValidator** — A `ParseFilePipe` validator checking the client-supplied MIME type. Not real validation — content must be checked by magic bytes. → [Ch 31](../chapters/31-file-uploads-streaming.md)

**forFeature** — The convention for registering a scoped slice of an already-root-configured module: one entity, one queue, one namespace. → [Ch 4](../chapters/04-modules.md), [Ch 17](../chapters/17-typeorm.md)

**forRoot / forRootAsync** — The convention for configuring a dynamic module once at the application root, synchronously or from injected providers. → [Ch 4](../chapters/04-modules.md)

**forwardRef** — A wrapper deferring a reference so two mutually dependent modules or providers can resolve. Fixes the mechanism, not the design. → [Ch 7](../chapters/07-dependency-injection.md)

**Functional middleware** — Middleware written as a plain function. Cannot inject dependencies; needing one is what forces the class form. → [Ch 11](../chapters/11-middleware.md)

---

## G

**Gateway** — A class handling WebSocket messages, declared with `@WebSocketGateway()`. A controller for sockets, with its own CORS configuration. → [Ch 25](../chapters/25-websockets.md)

**Global module** — A module marked `@Global()` whose exports are visible everywhere without being imported. Justified for configuration and logging, rarely otherwise. → [Ch 4](../chapters/04-modules.md)

**GqlExecutionContext** — The wrapper giving access to the GraphQL context from a guard or interceptor, since `switchToHttp()` yields nothing there. → [Ch 24](../chapters/24-graphql.md)

**gRPC** — A typed RPC protocol using `.proto` contracts. Chosen for polyglot services or where a schema-checked contract justifies a build step. → [Ch 26](../chapters/26-microservices.md)

**Guard** — A pipeline component deciding whether a request proceeds. Runs after middleware and before interceptors and pipes. → [Ch 12](../chapters/12-guards.md)

---

## H

**Health check** — An endpoint reporting whether an instance is alive or ready. Liveness must check nothing external; readiness may check dependencies. → [Ch 33](../chapters/33-logging-observability.md)

**Helmet** — Middleware setting security headers, including a Content Security Policy. Commonly disabled when CSP breaks something, which is the wrong fix. → [Ch 23](../chapters/23-security.md)

**Hexagonal architecture** — Ports and adapters: the domain defines interfaces, infrastructure implements them, and the domain depends on nothing external. Worth applying selectively. → [Ch 35](../chapters/35-architecture-patterns.md)

**HttpException** — The base class carrying an HTTP status. Anything thrown that is not one becomes a 500 with its message withheld. → [Ch 15](../chapters/15-exception-filters.md)

**Hybrid application** — An application serving both HTTP and one or more microservice transports, connected with `connectMicroservice()` and started with `startAllMicroservices()`. → [Ch 26](../chapters/26-microservices.md)

---

## I

**Idempotency** — The property that repeating an operation produces the same result as performing it once. Required wherever delivery is at-least-once. → [Ch 26](../chapters/26-microservices.md), [Ch 27](../chapters/27-queues-jobs.md)

**Idempotency key** — A client-supplied or event-carried identifier used to recognise a repeat. Must travel with the request or event, never be generated in the handler. → [Ch 6](../chapters/06-providers-services.md)

**@Injectable()** — The decorator marking a class as a provider and, crucially, triggering metadata emission so its constructor can be resolved. → [Ch 6](../chapters/06-providers-services.md)

**Injection token** — The key a provider is registered and looked up under: a class, a string, or a symbol. Symbols cannot collide; strings can. → [Ch 7](../chapters/07-dependency-injection.md)

**Interceptor** — A pipeline component wrapping the handler, able to see and transform the response and to skip the handler entirely. → [Ch 13](../chapters/13-interceptors.md)

**Introspection (GraphQL)** — The ability to query a schema for its own shape. Must be disabled in production. → [Ch 24](../chapters/24-graphql.md)

**IoC (Inversion of Control)** — Handing responsibility for constructing and wiring dependencies to a container. The gap Express leaves that NestJS fills. → [Ch 1](../chapters/01-what-nestjs-is.md)

**Isolation level** — How much concurrent transactions may observe of each other. `READ COMMITTED` is the usual default; `SERIALIZABLE` requires retry handling. → [Ch 18](../chapters/18-migrations-transactions.md)

---

## J

**Jest** — The default test runner in NestJS 11. Replaced by Vitest for new ESM projects in v12. → [Ch 29](../chapters/29-unit-testing.md)

**Joi** — A schema validation library commonly used for environment validation in `ConfigModule`. → [Ch 16](../chapters/16-configuration.md)

**JWT (JSON Web Token)** — A signed token carrying claims. Cannot be revoked before expiry without a denylist, which is its central trade-off. → [Ch 21](../chapters/21-authentication.md)

---

## K

**Kafka** — A durable log transport supporting consumer groups and replay. The only common transport letting a new consumer read history. → [Ch 26](../chapters/26-microservices.md)

**Keyset pagination** — Paginating by a cursor and a unique tiebreaker rather than an offset. Constant cost at any depth and stable under concurrent inserts. → [Ch 17](../chapters/17-typeorm.md)

**Keyv** — The storage abstraction underlying `cache-manager` v6, which NestJS 11 uses. Its arrival changed TTL units from seconds to milliseconds. → [Ch 20](../chapters/20-caching.md)

---

## L

**Layered architecture** — Controller → service → repository, each depending only on the layer below. The right default for most applications. → [Ch 35](../chapters/35-architecture-patterns.md)

**Lifecycle hook** — A method Nest calls at a defined point: `onModuleInit`, `onApplicationBootstrap`, `onModuleDestroy`, `beforeApplicationShutdown`, `onApplicationShutdown`. Never fires for request-scoped providers. → [Ch 9](../chapters/09-scopes-lifecycle.md)

**Liveness probe** — A health check whose failure causes the container to be restarted. Must not depend on anything a restart cannot fix. → [Ch 33](../chapters/33-logging-observability.md)

**Lock duration** — How long a queue holds a job's lock. Shorter than the job's runtime means a second worker starts while the first is still running. → [Ch 27](../chapters/27-queues-jobs.md)

---

## M

**Magic bytes** — The leading bytes identifying a file's actual format, and the only trustworthy basis for upload validation. → [Ch 31](../chapters/31-file-uploads-streaming.md)

**Mapped types** — `PartialType`, `PickType`, `OmitType`, and `IntersectionType`, deriving DTOs from others while preserving decorators. → [Ch 14](../chapters/14-pipes-validation.md)

**Mass assignment** — A vulnerability where a client sets fields it should not by supplying extra properties. Closed by `whitelist: true`. → [Ch 23](../chapters/23-security.md)

**Message pattern** — A request–response message declared with `@MessagePattern` and sent with `client.send()`. Couples caller availability to callee. → [Ch 26](../chapters/26-microservices.md)

**Metadata** — Data attached to a class, method, or parameter by a decorator, read later with `Reflector` or `Reflect.getMetadata`. → [Ch 2](../chapters/02-typescript-decorators.md)

**Middleware** — The first pipeline stage, running on the raw platform request before route resolution. Has no `ExecutionContext`. → [Ch 11](../chapters/11-middleware.md)

**Migration** — An explicit, ordered, reversible schema change. The alternative to `synchronize`, and the only reviewable one. → [Ch 18](../chapters/18-migrations-transactions.md)

**Mock** — A test double asserting on interactions. Appropriate when the call itself is the behaviour; over-used, it turns tests into a restatement of the implementation. → [Ch 29](../chapters/29-unit-testing.md)

**Module** — The unit of encapsulation: `imports`, `controllers`, `providers`, `exports`. `exports` is the public surface. → [Ch 4](../chapters/04-modules.md)

**ModuleRef** — A service locator giving programmatic access to the container. `get()` for singletons, `resolve()` for scoped providers. → [Ch 7](../chapters/07-dependency-injection.md)

**Monorepo** — A repository containing several applications and shared libraries, supported by the Nest CLI. Justified when deployables genuinely share code. → [Ch 35](../chapters/35-architecture-patterns.md)

**Multer** — The Express middleware handling multipart uploads, wrapped by `FileInterceptor`. Buffers to memory by default. → [Ch 31](../chapters/31-file-uploads-streaming.md)

---

## N

**N+1 problem** — Fetching a collection and then querying once per row for a relation. Structural in GraphQL, incidental but common in REST. → [Ch 17](../chapters/17-typeorm.md), [Ch 24](../chapters/24-graphql.md)

**Nest CLI** — The scaffolding and build tool. Its generators matter because they register the class in the right module. → [Ch 3](../chapters/03-first-application.md)

**NestFactory** — The entry point creating an application. Scans the module graph, resolves and instantiates providers, runs init hooks, and registers routes — all before `listen()`. → [Ch 3](../chapters/03-first-application.md)

**NestMiddleware** — The interface class middleware implements, with a `use(req, res, next)` method. → [Ch 11](../chapters/11-middleware.md)

**noAck** — A RabbitMQ option. Left at its default, the broker considers a message delivered on dispatch, so a crash mid-processing loses it. → [Ch 26](../chapters/26-microservices.md)

---

## O

**Observable** — The RxJS type interceptors work with and `ClientProxy.send()` returns. `send()`'s is cold: nothing is transmitted until subscription. → [Ch 13](../chapters/13-interceptors.md), [Ch 26](../chapters/26-microservices.md)

**Offset pagination** — Paginating with `skip`/`take`. Simple, supports jumping to a page, and degrades with depth while shifting under concurrent writes. → [Ch 17](../chapters/17-typeorm.md)

**onApplicationBootstrap** — The lifecycle hook running after every module has initialised and before the server listens. Where cross-module work belongs. → [Ch 9](../chapters/09-scopes-lifecycle.md)

**onModuleInit** — The lifecycle hook running once the host module's own dependencies are resolved. Cannot rely on other modules being ready. → [Ch 9](../chapters/09-scopes-lifecycle.md)

**OpenTelemetry** — The vendor-neutral tracing standard. Its auto-instrumentation patches modules at load time, so it must be imported first. → [Ch 33](../chapters/33-logging-observability.md)

**Optimistic locking** — Detecting concurrent modification with a version column and failing the second writer. Free when there is no contention. → [Ch 18](../chapters/18-migrations-transactions.md)

**Outbox pattern** — Writing a state change and its event in one transaction, then relaying the event separately. The only way to make the two atomic. → [Ch 28](../chapters/28-cqrs-events.md)

**oxlint** — The linter replacing ESLint in the NestJS 12 default toolchain. → [Ch 36](../chapters/36-nestjs-11-to-12.md)

---

## P

**ParseFilePipe** — The pipe composing upload validators such as `MaxFileSizeValidator` and `FileTypeValidator`. Runs after the file is fully received. → [Ch 31](../chapters/31-file-uploads-streaming.md)

**ParseIntPipe** — A built-in pipe converting a string parameter to a number or throwing 400. Necessary because path and query values are always strings. → [Ch 5](../chapters/05-controllers-routing.md), [Ch 14](../chapters/14-pipes-validation.md)

**Passport** — The authentication middleware Nest wraps with `@nestjs/passport`. Strategies become providers whose `validate` return value lands on `request.user`. → [Ch 21](../chapters/21-authentication.md)

**Pessimistic locking** — Taking a database lock so a concurrent reader waits. Correct under contention, and deadlocks unless locks are acquired in a consistent order. → [Ch 18](../chapters/18-migrations-transactions.md)

**Pino** — A fast JSON logger, integrated via `nestjs-pino`. Structured output is what makes logs queryable. → [Ch 33](../chapters/33-logging-observability.md)

**Pipe** — A pipeline component transforming or rejecting one handler argument. Runs last, immediately before the handler. → [Ch 14](../chapters/14-pipes-validation.md)

**Port (hexagonal)** — An interface defined by the domain and implemented by infrastructure. In Nest, best expressed as an abstract class so it doubles as a DI token. → [Ch 35](../chapters/35-architecture-patterns.md)

**Prefetch** — How many messages a consumer may hold unacknowledged at once. Bounds in-flight work per consumer. → [Ch 26](../chapters/26-microservices.md)

**Pre-signed URL** — A short-lived, scoped URL letting a client upload to or download from object storage directly, so the application never handles the bytes. → [Ch 31](../chapters/31-file-uploads-streaming.md)

**Prisma** — An ORM generating a typed client from a schema file. Result types match the `select` exactly; it provides no module-level encapsulation in Nest. → [Ch 19](../chapters/19-prisma-choosing-orm.md)

**Problem Details (RFC 9457)** — The standard `application/problem+json` error format, obsoleting RFC 7807. → [Ch 15](../chapters/15-exception-filters.md)

**Projection** — A read model built from events, maintained separately from the write model. → [Ch 28](../chapters/28-cqrs-events.md)

**Prototype pollution** — An attack injecting `__proto__` through a merged object. Closed systemically by `whitelist: true`. → [Ch 23](../chapters/23-security.md)

**Provider** — Anything the container can construct and inject: services, repositories, guards, factory results, and plain values. → [Ch 6](../chapters/06-providers-services.md)

---

## Q

**Query (CQRS)** — An object representing a read, dispatched through a `QueryBus`. Mutates nothing. → [Ch 28](../chapters/28-cqrs-events.md)

**QueryBuilder** — TypeORM's fluent SQL builder. Values must be bound parameters; column names cannot be, so they need an allowlist. → [Ch 17](../chapters/17-typeorm.md)

**QueryRunner** — TypeORM's handle to a single connection, used inside migrations and manual transactions. → [Ch 18](../chapters/18-migrations-transactions.md)

---

## R

**RabbitMQ** — A broker with durable queues, competing consumers, and acknowledgements. The usual choice for work queues that must not lose messages. → [Ch 26](../chapters/26-microservices.md)

**Rate limiting** — Bounding request volume per principal or IP. In-memory storage does not work across replicas. → [Ch 23](../chapters/23-security.md)

**RBAC (Role-Based Access Control)** — Authorization from roles alone, decidable without loading the resource, which makes it a good fit for a guard. → [Ch 22](../chapters/22-authorization.md)

**ReBAC (Relationship-Based Access Control)** — Authorization derived from a relationship graph. Needs a dedicated system rather than an in-process check. → [Ch 22](../chapters/22-authorization.md)

**Readiness probe** — A health check whose failure removes an instance from rotation without killing it. Where dependency checks belong. → [Ch 33](../chapters/33-logging-observability.md)

**Redis adapter** — The Socket.IO adapter propagating broadcasts across replicas. Without it, a broadcast reaches only one process's sockets. → [Ch 25](../chapters/25-websockets.md)

**reflect-metadata** — The polyfill providing `Reflect.defineMetadata` and `Reflect.getMetadata`. Imported internally by Nest. → [Ch 2](../chapters/02-typescript-decorators.md)

**Reflector** — The service reading decorator metadata in guards and interceptors. `getAllAndOverride` lets route metadata win; `getAllAndMerge` combines. → [Ch 12](../chapters/12-guards.md)

**Refresh token rotation** — Issuing a new refresh token on each use and invalidating the old one, so a replayed token reveals theft. → [Ch 21](../chapters/21-authentication.md)

**Repeatable job** — A BullMQ job scheduled centrally in Redis and delivered to exactly one worker. Usually better than `@Cron` in a cluster. → [Ch 32](../chapters/32-task-scheduling.md)

**Repository** — The data-access layer. As a contract it creates a testing seam; over the ORM directly it is often simpler. → [Ch 6](../chapters/06-providers-services.md)

**Request scope** — A provider instantiated per request. Propagates upward through every consumer, which is its defining hazard. → [Ch 9](../chapters/09-scopes-lifecycle.md)

**Resolver** — A GraphQL class mapping queries, mutations, and fields to methods. A controller for GraphQL. → [Ch 24](../chapters/24-graphql.md)

**Rspack** — The bundler replacing Webpack in the NestJS 12 default toolchain. → [Ch 36](../chapters/36-nestjs-11-to-12.md)

---

## S

**Saga** — A process reacting to events by dispatching commands, coordinating a multi-step operation and compensating on failure. → [Ch 28](../chapters/28-cqrs-events.md)

**Scope** — How many instances of a provider exist and how long each lives: `DEFAULT`, `REQUEST`, or `TRANSIENT`. → [Ch 9](../chapters/09-scopes-lifecycle.md)

**Scope bubbling** — Request scope propagating upward: a request-scoped provider makes every consumer above it request-scoped too. → [Ch 9](../chapters/09-scopes-lifecycle.md)

**Serialization groups** — `@Expose({ groups })` plus `@SerializeOptions`, controlling which fields appear per caller. The right place for field-level authorization. → [Ch 13](../chapters/13-interceptors.md)

**Single-flight** — Collapsing concurrent misses for one cache key into a single computation. → [Ch 20](../chapters/20-caching.md)

**SKIP LOCKED** — A SQL clause letting concurrent workers claim disjoint rows without blocking. Used by outbox relays and job claims. → [Ch 28](../chapters/28-cqrs-events.md)

**Soft delete** — Marking a row deleted with a timestamp rather than removing it. Enabled by `@DeleteDateColumn`, and excluded from reads automatically. → [Ch 17](../chapters/17-typeorm.md)

**SSRF (Server-Side Request Forgery)** — Causing a server to fetch an attacker-chosen URL, reaching internal services and cloud metadata endpoints. Must be checked after DNS resolution. → [Ch 23](../chapters/23-security.md)

**Standard Schema** — A shared interface implemented by Zod, Valibot, and ArkType, gaining native support in NestJS 12 route decorators. → [Ch 36](../chapters/36-nestjs-11-to-12.md)

**StreamableFile** — The wrapper piping a stream to the response, keeping memory flat regardless of file size. → [Ch 31](../chapters/31-file-uploads-streaming.md)

**Strategy (Passport)** — An authentication mechanism — local, JWT, OAuth — implemented as a provider extending `PassportStrategy`. → [Ch 21](../chapters/21-authentication.md)

**Stub** — A test double returning canned values. Appropriate when the test cares about the result; risky for repositories, where a fake is better. → [Ch 29](../chapters/29-unit-testing.md)

**Supertest** — The library driving an application over HTTP in tests, using the handler rather than a bound port. → [Ch 30](../chapters/30-e2e-testing.md)

**synchronize** — A TypeORM option altering the schema to match entities. Will drop columns; must be `false` outside a disposable database. → [Ch 17](../chapters/17-typeorm.md)

---

## T

**Terminus** — The `@nestjs/terminus` package providing health-check indicators for databases, memory, disk, and HTTP dependencies. → [Ch 33](../chapters/33-logging-observability.md)

**Testcontainers** — A library starting real infrastructure in containers for tests, so integration tests run against the engine you deploy on. → [Ch 30](../chapters/30-e2e-testing.md)

**Testing module** — The container built by `Test.createTestingModule()`, supporting `overrideProvider`, `overrideGuard`, and related methods. → [Ch 29](../chapters/29-unit-testing.md)

**Throttler** — `@nestjs/throttler`, providing rate limiting as a guard. Needs shared storage across replicas. → [Ch 23](../chapters/23-security.md)

**Transaction** — A group of statements committing or rolling back together. Every query inside must use the transaction's manager. → [Ch 18](../chapters/18-migrations-transactions.md)

**Transient scope** — A provider instantiated once per injecting consumer. Does not bubble, unlike request scope. → [Ch 9](../chapters/09-scopes-lifecycle.md)

**Transport** — The mechanism carrying microservice messages: TCP, Redis, NATS, RabbitMQ, Kafka, or gRPC. → [Ch 26](../chapters/26-microservices.md)

**TypeORM** — The ORM with the official Nest integration, using decorated entity classes and providing real module boundaries through `forFeature`. → [Ch 17](../chapters/17-typeorm.md)

---

## U

**UnrecoverableError** — A BullMQ error stopping retries immediately, for failures that will never succeed. → [Ch 27](../chapters/27-queues-jobs.md)

**Upcaster** — A function converting an old event shape to the current one, required indefinitely in event-sourced systems. → [Ch 28](../chapters/28-cqrs-events.md)

**useClass** — A provider recipe naming a class for the container to construct, allowing the token and the implementation to differ. → [Ch 8](../chapters/08-custom-providers.md)

**useExisting** — A provider recipe aliasing one token to another, resolving to the same instance. The correct way to deprecate a token. → [Ch 8](../chapters/08-custom-providers.md)

**useFactory** — A provider recipe running a function, with dependencies supplied positionally through `inject`. → [Ch 8](../chapters/08-custom-providers.md)

**useValue** — A provider recipe supplying an existing object. Evaluated at import time, so unsuitable for anything derived from configuration. → [Ch 8](../chapters/08-custom-providers.md)

---

## V

**ValidationPipe** — The pipe validating DTOs with `class-validator`. `whitelist` and `forbidNonWhitelisted` are what make it a security control. → [Ch 14](../chapters/14-pipes-validation.md)

**@VersionColumn** — The TypeORM column enabling optimistic locking. → [Ch 18](../chapters/18-migrations-transactions.md)

**Versioning** — Serving several API versions concurrently, by URI, header, media type, or a custom extractor. → [Ch 5](../chapters/05-controllers-routing.md)

**Vitest** — The test runner replacing Jest for new ESM projects in NestJS 12. The `@nestjs/testing` API is unchanged. → [Ch 29](../chapters/29-unit-testing.md), [Ch 36](../chapters/36-nestjs-11-to-12.md)

---

## W

**whitelist** — The `ValidationPipe` option stripping properties the DTO does not declare. A security control against mass assignment, not a convenience. → [Ch 14](../chapters/14-pipes-validation.md), [Ch 23](../chapters/23-security.md)

**Worker thread** — A separate thread for CPU-bound work, typically pooled with `piscina`, keeping the event loop free. → [Ch 34](../chapters/34-performance-scaling.md)

**WsException** — The exception type for WebSocket handlers, delivered through a `BaseWsExceptionFilter`. There is no status code channel. → [Ch 25](../chapters/25-websockets.md)

---

## Z

**Zod** — A schema validation library with type inference, implementing Standard Schema and usable in NestJS 12 route decorators. → [Ch 16](../chapters/16-configuration.md), [Ch 36](../chapters/36-nestjs-11-to-12.md)
