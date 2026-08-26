# Appendix E — Best Practices Checklist

Reviewable, item by item. Every line is specific enough to check against real code — "Register global pipes with `APP_PIPE` so they apply in tests" rather than "configure validation properly".

Use it three ways: as a self-review before opening a pull request, as a reviewer's prompt when something feels wrong but you can't name it, and as an audit of an unfamiliar codebase.

**Not every item applies to every project.** A prototype doesn't need distributed locking. Skip deliberately, not by accident.

🔒 marks items where getting it wrong is a security problem rather than a quality one.

---

## Contents

| Section | Items |
|---|---|
| [Modules and Structure](#modules-and-structure) | 14 |
| [Dependency Injection](#dependency-injection) | 14 |
| [Controllers and Routing](#controllers-and-routing) | 13 |
| [Services and Business Logic](#services-and-business-logic) | 13 |
| [The Request Pipeline](#the-request-pipeline) | 13 |
| [Validation](#validation) | 12 |
| [Error Handling](#error-handling) | 12 |
| [Configuration](#configuration) | 12 |
| [Databases](#databases) | 16 |
| [Caching](#caching) | 11 |
| [Authentication and Authorization](#authentication-and-authorization) | 16 |
| [Security](#security) | 16 |
| [Async Work](#async-work-queues-events-scheduling) | 15 |
| [Testing](#testing) | 15 |
| [Observability](#observability) | 13 |
| [Performance](#performance) | 13 |
| [Production Readiness](#production-readiness) | 14 |

---

## Modules and Structure

- [ ] Folders are grouped by feature, not by technical layer.
- [ ] Every module's job can be stated in one sentence without "and".
- [ ] A typical feature change touches one module, not five.
- [ ] `exports` contains the smallest surface that satisfies consumers.
- [ ] No provider is registered in more than one module.
- [ ] Shared providers are exported and imported, never duplicated.
- [ ] `@Global()` is used for configuration and logging only — two or three modules at most.
- [ ] Module classes have empty bodies; no logic lives in them.
- [ ] Dynamic modules follow the `forRoot`/`forFeature`/`register` naming convention.
- [ ] Modules with options use `ConfigurableModuleBuilder` rather than hand-written async variants.
- [ ] There are no `forwardRef` cycles beyond a deliberate pair.
- [ ] Module dependencies flow in one direction, verified by a generated graph.
- [ ] A module is dynamic only because a consumer needs to vary its behaviour.
- [ ] Import direction between layers or libraries is enforced by a lint rule, not a convention.

---

## Dependency Injection

- [ ] Every provider with dependencies carries `@Injectable()`.
- [ ] Dependencies are declared `private readonly` in the constructor.
- [ ] Constructor injection is used; property injection only in base classes.
- [ ] No class is constructed with `new` where the container should own it.
- [ ] Contracts are abstract classes, not interfaces, so they work as tokens.
- [ ] String tokens are replaced with exported `Symbol` constants.
- [ ] Injection tokens are exported from one file and imported, never retyped.
- [ ] `useExisting` is used for aliases; `useClass` is never used to alias.
- [ ] Factory providers' `inject` array order matches the factory signature exactly.
- [ ] Async factories have a timeout so an unreachable dependency cannot hang bootstrap.
- [ ] Factories validate their inputs and throw rather than degrading silently. 🔒
- [ ] `@Optional()` appears only where every use site handles absence.
- [ ] `ModuleRef` is used only for genuinely dynamic tokens, not as a default.
- [ ] `{ strict: false }` appears rarely and is commented where it does.

---

## Controllers and Routing

- [ ] Controllers contain no business logic — signature, call, return.
- [ ] Static routes are declared before dynamic ones.
- [ ] Path and query parameters are converted with a pipe, never assumed to be typed.
- [ ] Query parameters are bound as a DTO, not extracted one at a time.
- [ ] `POST` endpoints that are actions rather than creations set `@HttpCode(200)`.
- [ ] `DELETE` returning nothing uses 204 and returns no body.
- [ ] Bare `@Res()` is not used; `@Res({ passthrough: true })` where a raw response is needed.
- [ ] Controllers return DTOs, never ORM entities.
- [ ] Every list endpoint has a maximum page size. 🔒
- [ ] Wildcards use Express 5 named syntax (`*splat`).
- [ ] API versioning is enabled from the first release.
- [ ] Every controller has an explicit version once more than one exists.
- [ ] Response envelopes are applied globally with a documented opt-out.

---

## Services and Business Logic

- [ ] No service instance field holds request-specific state. 🔒
- [ ] Instance fields hold only dependencies and configuration.
- [ ] Business rules live in services, reachable from queues and CLI entry points.
- [ ] Services never inject another module's repository directly.
- [ ] Constructors have five or fewer dependencies.
- [ ] Services are split by reason to change, not by entity.
- [ ] Public methods have explicit return types.
- [ ] Domain errors are thrown; HTTP exceptions stay out of services with multiple callers.
- [ ] Operations that clients retry are idempotent, keyed on a client-supplied value.
- [ ] Idempotency is enforced by a constraint, not only by a prior read. 🔒
- [ ] Best-effort side effects are detached explicitly, not awaited into a critical path.
- [ ] Anything that must not be lost goes through a durable queue, not fire-and-forget.
- [ ] Entities carry their own invariants where the domain has real rules.

---

## The Request Pipeline

- [ ] Global guards, pipes, filters, and interceptors are registered with `APP_*` tokens.
- [ ] Nothing registered in `main.ts` is required for correct behaviour in tests.
- [ ] Authentication is a guard, never middleware. 🔒
- [ ] Middleware is limited to platform-level work: context, raw bodies, third-party Express packages.
- [ ] Every middleware either responds or calls `next()` — never neither.
- [ ] Middleware is bound to controller classes rather than string paths where possible.
- [ ] Health and metrics endpoints are excluded from logging, throttling, and auth.
- [ ] Interceptors doing measurement handle the error path (`tap({ next, error })` or `finalize`).
- [ ] Timeout interceptors convert `TimeoutError` into an `HttpException`.
- [ ] Guards are fast and free of side effects.
- [ ] Guards fail closed if an expected upstream guard did not run. 🔒
- [ ] Globally registered guards and filters branch on `getType()` for non-HTTP transports.
- [ ] Caching interceptors are opt-in per route and declare what the response varies by. 🔒

---

## Validation

- [ ] `ValidationPipe` is registered globally via `APP_PIPE`.
- [ ] `whitelist: true` is enabled. 🔒
- [ ] `forbidNonWhitelisted: true` is enabled so unknown fields are rejected, not dropped.
- [ ] `transform: true` is enabled so DTOs arrive as class instances.
- [ ] Every DTO is a class; none is an interface. 🔒
- [ ] Every nested object or array has `@Type()` alongside `@ValidateNested()`. 🔒
- [ ] Every array property has a maximum size. 🔒
- [ ] Every string property has a maximum length.
- [ ] Numeric properties have explicit bounds.
- [ ] Update DTOs are derived with `PartialType`, not duplicated.
- [ ] Mapped types are imported from `@nestjs/swagger` if Swagger is in use.
- [ ] Negative tests exist asserting rejection for each validation rule. 🔒

---

## Error Handling

- [ ] One global catch-all filter normalises every error into one shape.
- [ ] The filter preserves `HttpException` statuses rather than collapsing to 500.
- [ ] Responses carry a stable machine-readable `code` clients can branch on.
- [ ] Responses carry a correlation id.
- [ ] No response contains a stack trace, driver message, or connection string. 🔒
- [ ] `ValidationPipe`'s array `message` is normalised into a structured `errors` list.
- [ ] Filters check `host.getType()` before calling `switchToHttp()`.
- [ ] Filters never throw.
- [ ] 4xx is logged at warn or below; 5xx at error.
- [ ] Expected conditions throw typed exceptions, not plain `Error`.
- [ ] Domain errors map to statuses through one reviewable table.
- [ ] The design does not depend on ordering between multiple global filters.

---

## Configuration

- [ ] Every environment variable is described by a validation schema.
- [ ] Validation runs at startup with `abortEarly: false`.
- [ ] `allowUnknown: true` is set so OS variables do not fail the boot.
- [ ] Required values use `getOrThrow`, not `get` with a non-null assertion.
- [ ] Optional values have explicit defaults.
- [ ] Configuration is namespaced with `registerAs` and consumed as `ConfigType`.
- [ ] No service reads `process.env` directly.
- [ ] Anything derived from configuration is provided by `useFactory`, never `useValue`.
- [ ] Production secrets come from a secret manager, not `.env`. 🔒
- [ ] No fallback secret exists anywhere in the codebase. 🔒
- [ ] The config object is redacted by key pattern before any logging. 🔒
- [ ] `.env` is gitignored and `.env.example` is generated from the schema.

---

## Databases

- [ ] `synchronize` is `false`. 🔒
- [ ] Schema changes go through reviewed migrations with tested `down` methods.
- [ ] Migrations run as a deploy step, not at application boot.
- [ ] Generated migrations are read before being committed.
- [ ] Adding a `NOT NULL` column is split across deploys.
- [ ] Renames use expand-migrate-contract, never a single migration.
- [ ] Large-table indexes are created `CONCURRENTLY`.
- [ ] Timestamps are `timestamptz`.
- [ ] Money is stored as integer minor units.
- [ ] Indexes match the queries actually run, with column order considered.
- [ ] No relation uses `eager: true`.
- [ ] Relations are loaded explicitly per query.
- [ ] Query logging is enabled in development.
- [ ] Every query-builder value is a bound parameter; identifiers use an allowlist. 🔒
- [ ] Multi-row writes run in a transaction, using the transaction's manager throughout.
- [ ] Counters and stock use conditional `UPDATE … WHERE`, never read-then-write. 🔒

---

## Caching

- [ ] Caching is applied only after the underlying query is correct.
- [ ] TTL units match the `cache-manager` major version in use.
- [ ] Cache keys include the principal and the tenant. 🔒
- [ ] Cache keys include the API version and serialization group.
- [ ] Keys are built in one shared function, not with inline template strings. 🔒
- [ ] `CacheInterceptor` is not applied globally to authenticated routes. 🔒
- [ ] Cache misses are detected with `!== undefined`, not truthiness.
- [ ] Cache read and write failures degrade to the source of truth.
- [ ] Concurrent misses are collapsed with single-flight.
- [ ] TTLs are jittered so keys do not expire together.
- [ ] Invalidation uses versioned keys rather than enumerated deletions.

---

## Authentication and Authorization

- [ ] Authentication is global with an explicit `@Public()` opt-out. 🔒
- [ ] JWT strategies pin the `algorithms` list. 🔒
- [ ] `validate` confirms the subject still exists and is permitted. 🔒
- [ ] Passwords use Argon2id or bcrypt with an explicit length limit. 🔒
- [ ] Login is indistinguishable between unknown user and wrong password, in message and timing. 🔒
- [ ] Login is rate-limited. 🔒
- [ ] Access tokens are short-lived.
- [ ] Refresh tokens are opaque, hashed at rest, and single-use. 🔒
- [ ] Refresh reuse revokes the whole token family. 🔒
- [ ] Tokens are never stored in `localStorage`. 🔒
- [ ] Refresh cookies are `httpOnly`, `Secure`, `SameSite`, and path-scoped. 🔒
- [ ] Authentication and authorization are separate guards.
- [ ] Business logic checks permissions, not roles.
- [ ] Ownership is enforced as a query predicate, returning 404 rather than 403. 🔒
- [ ] Authorization is enforced in services, so non-HTTP callers are covered. 🔒
- [ ] Field-level visibility uses serialization groups.

---

## Security

- [ ] Helmet is enabled with a configured CSP, not disabled. 🔒
- [ ] CORS origins come from configuration and deny by default. 🔒
- [ ] `trust proxy` is set to the exact number of proxies. 🔒
- [ ] Rate limiting uses shared storage across replicas. 🔒
- [ ] Rate limits key on the principal with an IP fallback.
- [ ] Body size is bounded globally.
- [ ] DTOs never pass straight into `save()`. 🔒
- [ ] Redirect targets are allowlisted and reject protocol-relative values. 🔒
- [ ] Outbound URLs supplied by clients are checked after DNS resolution. 🔒
- [ ] Upload types are validated by magic bytes, not the declared MIME type. 🔒
- [ ] Uploaded filenames are generated, never taken from the client. 🔒
- [ ] User uploads are served from a separate origin with `attachment` disposition. 🔒
- [ ] No deprecated security package is in use (`csurf`). 🔒
- [ ] CSRF protection exists if and only if credentials travel in cookies.
- [ ] `npm audit` runs in CI and a lockfile is committed.
- [ ] A test asserts that no error response contains a secret or a stack trace. 🔒

---

## Async Work: Queues, Events, Scheduling

- [ ] Work the caller does not need the result of is queued.
- [ ] Work the caller is waiting on is not queued.
- [ ] Job payloads contain ids, never entities or secrets. 🔒
- [ ] Processors re-read state rather than trusting the payload.
- [ ] Job enqueueing is idempotent via a deterministic `jobId`.
- [ ] Job effects are idempotent via an in-transaction unique constraint. 🔒
- [ ] Permanent failures throw `UnrecoverableError` rather than consuming retries.
- [ ] `removeOnComplete` and `removeOnFail` are configured.
- [ ] `lockDuration` exceeds the longest plausible job runtime.
- [ ] Processors run in a separate process from the API.
- [ ] Queues use a dedicated persistent Redis, not the cache instance.
- [ ] Scheduled jobs are named and carry an explicit time zone.
- [ ] Cluster-wide single execution is guaranteed by a lock or a repeatable job. 🔒
- [ ] Overlap guards reset in a `finally`.
- [ ] Events that must not be lost are published through an outbox, not an in-memory bus.

---

## Testing

- [ ] Unit tests construct classes directly where the container is not under test.
- [ ] No unit test imports `AppModule`.
- [ ] Repositories are faked, not stubbed.
- [ ] Assertions are about outcomes, not about calls, wherever an outcome is observable.
- [ ] Every validation rule has a test asserting rejection. 🔒
- [ ] Every protected route has explicit 401 and 403 tests. 🔒
- [ ] E2E tests use real tokens rather than overriding the auth guard. 🔒
- [ ] E2E tests assert against the database, not only the response.
- [ ] Integration tests run against the real database engine.
- [ ] Test isolation works under parallel workers.
- [ ] `app.close()` runs in teardown.
- [ ] Migrations are applied in test setup, not `synchronize`.
- [ ] Idempotent endpoints have a concurrent test using `Promise.all`.
- [ ] Every `rejects` assertion is awaited.
- [ ] The suite has been validated by breaking code deliberately and confirming failures.

---

## Observability

- [ ] Logs are structured fields, not interpolated strings.
- [ ] `bufferLogs` is enabled and a real logger attached at bootstrap.
- [ ] Sensitive headers, cookies, and body fields are redacted. 🔒
- [ ] Pretty-printing is disabled in production.
- [ ] A correlation id is accepted, generated when absent, and echoed in the response.
- [ ] The correlation id propagates into queue jobs and outbound calls.
- [ ] Request rate, error rate, and duration are recorded by route pattern.
- [ ] Metrics are recorded on the failure path as well as success.
- [ ] No metric label contains an id, email, or raw URL.
- [ ] Event-loop lag is measured continuously.
- [ ] Queue depth *and* oldest-job age are monitored.
- [ ] Scheduled jobs alert on absence, not only on failure.
- [ ] Tracing is initialised before any instrumented module is imported.

---

## Performance

- [ ] No optimisation has been made without a measurement.
- [ ] Benchmarks run against a production build.
- [ ] No endpoint issues a query per row.
- [ ] Query counts per endpoint are asserted in tests for hot paths.
- [ ] `select` lists the columns actually used.
- [ ] Deep pagination uses a cursor, not an offset.
- [ ] No synchronous CPU work runs in a request handler.
- [ ] CPU-bound work uses worker threads or a queue.
- [ ] Outbound HTTP uses keep-alive agents and timeouts.
- [ ] Pool size × replica count is within the database's connection limit.
- [ ] Compression has a size threshold.
- [ ] Large responses stream rather than buffer.
- [ ] The platform adapter was chosen deliberately at project start.

---

## Production Readiness

- [ ] `enableShutdownHooks()` is called.
- [ ] Readiness fails on shutdown before connections close.
- [ ] Liveness checks nothing external. 🔒
- [ ] Graceful shutdown drains in-flight requests and jobs within the platform's grace period.
- [ ] The app runs on the Node version the framework requires.
- [ ] `deleteOutDir` is enabled so stale build output cannot ship.
- [ ] `npm run build` runs in CI, not only tests.
- [ ] Production images contain no dev dependencies.
- [ ] Swagger is disabled or protected in production. 🔒
- [ ] GraphQL introspection and playground are disabled in production. 🔒
- [ ] Every external call has a timeout.
- [ ] In-memory state has been audited against multi-replica deployment.
- [ ] Scheduled jobs, caches, rate limits, and socket broadcasts all work at more than one replica.
- [ ] An upgrade audit exists for the next major version.

---

**Next:** [Appendix F — Cheat Sheet →](F-cheatsheet.md)
