# Appendix C — 20 Real-World Projects

Ordered easy to expert. Each specifies the goal, the chapters it exercises, hard requirements, stretch goals, and a realistic time estimate.

**No solutions.** That's deliberate — a project you've been shown is a tutorial, and tutorials produce the illusion of competence. Build them badly, then fix them.

**Rules that apply to every project here:**

- `ValidationPipe` with `whitelist` and `forbidNonWhitelisted`, registered via `APP_PIPE`.
- Every list endpoint has a maximum page size.
- Global exception filter producing one error shape with a machine-readable `code`.
- `enableShutdownHooks()`, and a graceful shutdown that drops no in-flight request.
- No request-specific state on any singleton provider.
- Tests that assert *rejection*, not only success.

Estimates assume you already program and are learning NestJS specifically. Halve them if you are experienced with another DI framework; double them if TypeScript is also new.

---

## Contents

| # | Project | Difficulty | Chapters | Time |
|---|---------|-----------|----------|------|
| 1 | [URL Shortener](#1-url-shortener) | Easy | 3–6, 14 | 4–6 h |
| 2 | [Task API with Pagination](#2-task-api-with-pagination) | Easy | 5, 14, 17 | 6–8 h |
| 3 | [Configuration Service](#3-configuration-service) | Easy | 8, 16 | 4–6 h |
| 4 | [Request Context and Logging](#4-request-context-and-logging) | Easy | 9, 11, 33 | 5–7 h |
| 5 | [Rate-Limited Public API](#5-rate-limited-public-api) | Easy | 12, 23 | 6–8 h |
| 6 | [Pipeline Visualiser](#6-pipeline-visualiser) | Medium | 10–15 | 8–12 h |
| 7 | [Auth Service](#7-auth-service) | Medium | 21 | 12–16 h |
| 8 | [Multi-Tenant Notes API](#8-multi-tenant-notes-api) | Medium | 9, 22 | 12–16 h |
| 9 | [File Upload Service](#9-file-upload-service) | Medium | 27, 31 | 10–14 h |
| 10 | [Cached Product Catalogue](#10-cached-product-catalogue) | Medium | 17, 20 | 10–14 h |
| 11 | [Inventory with Reservations](#11-inventory-with-reservations) | Medium | 18 | 12–16 h |
| 12 | [Webhook Receiver](#12-webhook-receiver) | Medium | 11, 27 | 10–14 h |
| 13 | [Notification Fan-Out](#13-notification-fan-out) | Medium | 26, 27 | 14–18 h |
| 14 | [Real-Time Board](#14-real-time-collaboration-board) | Hard | 25 | 16–24 h |
| 15 | [GraphQL Layer](#15-graphql-layer-over-an-existing-api) | Hard | 24 | 16–20 h |
| 16 | [Scheduled Billing](#16-scheduled-billing-runner) | Hard | 18, 32 | 16–20 h |
| 17 | [Order Saga](#17-order-saga-across-two-services) | Hard | 26, 28 | 20–30 h |
| 18 | [Observability Platform](#18-observability-for-an-existing-service) | Hard | 33, 34 | 16–20 h |
| 19 | [Test Harness](#19-a-test-suite-that-catches-real-bugs) | Expert | 29, 30 | 20–25 h |
| 20 | [Production SaaS Backend](#20-production-saas-backend) | Expert | all | 60–100 h |

---

## 1. URL Shortener

**Goal:** the smallest useful API — create a short code, redirect, count hits.

**Requirements:**
- `POST /links` accepts a URL, returns a short code. `GET /:code` redirects.
- URLs validated as `http`/`https` only, with a length limit.
- Redirect targets rejected if they resolve to a private address.
- Hit counting that does not slow the redirect.
- 404 for unknown codes; 410 for expired ones.

**Stretch:** custom aliases with collision handling; per-link expiry.

**Why it's first:** every concept from Chapters 3–6 in one small surface, plus an open-redirect problem that is real rather than academic.

---

## 2. Task API with Pagination

**Goal:** CRUD done properly, which is harder than CRUD.

**Requirements:**
- Full CRUD with DTOs derived using `PartialType` and `OmitType`.
- Filtering, sorting with a column allowlist, and keyset pagination.
- A stable response envelope with `data` and `meta`.
- Soft delete, with deleted rows excluded from every read path.
- Correct status codes throughout, including 204 with no body.

**Stretch:** two API versions sharing one service, where v2 renames a field.

---

## 3. Configuration Service

**Goal:** make a misconfigured deploy fail at boot.

**Requirements:**
- One schema validating every variable, `abortEarly: false`.
- Namespaced config consumed as `ConfigType` — no `ConfigService` in domain services.
- A provider that is `undefined` with `useValue` and correct with `useFactory`, both demonstrated.
- Startup log of effective configuration with pattern-based redaction.
- A test asserting a known secret never appears in captured log output.

**Stretch:** generate `.env.example` from the schema and fail CI on drift.

---

## 4. Request Context and Logging

**Goal:** correlate everything belonging to one request without touching provider scope.

**Requirements:**
- Middleware establishing an `AsyncLocalStorage` context, with `next()` called *inside* `run`.
- Correlation id accepted from the caller, generated when absent, echoed in the response.
- Structured logs carrying the id from guards, services, and repositories.
- The id propagated into a queue job and re-established in the worker.
- Every provider remains a singleton — prove it by logging constructor calls.

**Stretch:** the same thing with `Scope.REQUEST`, then measure the instantiation difference.

---

## 5. Rate-Limited Public API

**Goal:** limits that actually hold.

**Requirements:**
- Tiered limits: burst and sustained, with per-route overrides.
- Keyed on the principal where present, IP otherwise.
- Shared storage so limits hold across replicas — prove it with two instances.
- `trust proxy` set correctly, with a test showing a forged `X-Forwarded-For` does not reset the bucket.
- 429 with a `Retry-After` header.

**Stretch:** per-tenant quotas with a monthly window and usage reporting.

---

## 6. Pipeline Visualiser

**Goal:** make the request lifecycle observable rather than memorised.

**Requirements:**
- Tracing guard, interceptor, pipe, and filter, bound at global, controller, and route scope.
- A log showing the full order for: success, guard rejection, validation failure, and handler throw.
- `GET /_pipeline?route=…` listing what would run, in order, read from registered metadata.
- Refuses to register when `NODE_ENV=production`.
- A test asserting the reported order matches the documented pipeline.

**Why it's worth building:** you will remember Chapter 10 permanently afterwards.

---

## 7. Auth Service

**Goal:** authentication you would be willing to ship.

**Requirements:**
- Registration and login with Argon2id and rate limiting.
- No account enumeration — identical message *and* timing for unknown email and wrong password.
- Access tokens under 15 minutes; refresh tokens opaque, hashed at rest, single-use, family-grouped.
- Reuse detection revoking the family, with a grace window for honest retries.
- Refresh token in an `httpOnly`, `SameSite=Strict`, path-scoped cookie.

**Stretch:** OAuth via a second Passport strategy, normalised onto the same principal.

**Tests required:** reuse revokes the family; a deleted user's valid token is rejected; timing does not distinguish unknown emails.

---

## 8. Multi-Tenant Notes API

**Goal:** isolation that survives a developer forgetting.

**Requirements:**
- Tenant resolved from a header; requests without one rejected before any handler.
- Tenant context available everywhere without promoting any provider to request scope.
- Isolation enforced at the data layer, not only in a guard.
- CASL rules driving both instance checks and list filters from one definition.
- A test proving two concurrent requests for different tenants never observe each other.

**Stretch:** durable providers giving each tenant its own connection, with LRU-bounded subtrees.

---

## 9. File Upload Service

**Goal:** accept files from people you do not trust.

**Requirements:**
- Pre-signed uploads with server-enforced type and size; a `pending` record written *before* the URL is issued.
- Asynchronous verification by magic bytes, quarantining anything contradicting its declaration.
- Downloads via short-lived pre-signed URLs with forced `attachment` disposition.
- A sweeper reconciling `pending` rows older than the URL expiry.
- Tests for: a spoofed MIME type, a traversal filename, an abandoned upload, an oversized file.

**Stretch:** multipart upload for multi-gigabyte files, with abort-incomplete lifecycle rules.

---

## 10. Cached Product Catalogue

**Goal:** caching that cannot leak.

**Requirements:**
- Fix the queries first — no N+1, bounded pages, columns selected explicitly.
- Opt-in `@Cacheable({ ttl, varyBy })`; the interceptor refuses to cache an authenticated route with no `varyBy`.
- Keys built in one function where tenant is a required field.
- Single-flight plus jittered TTLs; 200 concurrent misses cause one computation.
- Safe degradation — a test makes the store throw and asserts a 200.

**Stretch:** versioned-key invalidation, then add a new write path and confirm it cannot leave stale data.

---

## 11. Inventory with Reservations

**Goal:** correctness under concurrency, which testing normally hides.

**Requirements:**
- Reserving stock can never drive the count negative, under any interleaving.
- Reservations expire after a TTL and return stock automatically.
- Conditional `UPDATE … WHERE` rather than read-then-write.
- Deterministic lock ordering wherever two rows are locked.
- A load test firing 100 concurrent purchases of the last item, asserting exactly one succeeds.

**Stretch:** optimistic locking with a 409 and a client retry, compared against the pessimistic version.

---

## 12. Webhook Receiver

**Goal:** the endpoint everyone gets subtly wrong.

**Requirements:**
- Raw body captured in middleware, with signatures verified over the exact bytes.
- 200 returned quickly; processing happens on a queue.
- Idempotent by provider event id, enforced by a constraint inside the transaction.
- Replays and out-of-order deliveries handled correctly.
- Failures dead-lettered, never requeued blindly.

**Stretch:** a replay endpoint reprocessing dead-lettered events safely.

---

## 13. Notification Fan-Out

**Goal:** one event, several channels, no lost messages.

**Requirements:**
- An event published through an outbox so it cannot exist without the state change.
- A relay using `SKIP LOCKED`, safe to run on several replicas.
- Email, push, and in-app consumers, each independently retryable.
- Per-channel rate limiting matching each provider's real limit.
- User preferences honoured, evaluated at send time rather than enqueue time.

**Stretch:** digest batching — collapse twenty events in an hour into one message.

---

## 14. Real-Time Collaboration Board

**Goal:** real-time that works at more than one replica.

**Requirements:**
- Handshake authentication with disconnect on failure, plus expiry-driven disconnection.
- Rooms namespaced by tenant and board, with membership authorized on join.
- Redis adapter, proven by broadcasting from one instance and receiving on another.
- Presence as a Redis set with a TTL, correct for multiple tabs and after an unclean shutdown.
- A reconnect path where the client sends a cursor and receives missed events.

**Stretch:** operational transforms or CRDT-based conflict resolution for concurrent edits.

---

## 15. GraphQL Layer Over an Existing API

**Goal:** add GraphQL without duplicating rules or opening holes.

**Requirements:**
- Code-first schema with a committed `schema.gql` and CI failing on drift.
- Every relation field batched; a test asserts a 50-item query issues a fixed query count.
- One guard implementation serving both REST and GraphQL.
- Depth and complexity limits, plus a bounded `limit` on every list field.
- Authorization enforced in services — proven by reaching a record through a field resolver and being refused.

**Stretch:** subscriptions over WebSockets, sharing the auth path from project 14.

---

## 16. Scheduled Billing Runner

**Goal:** a job that must run exactly once, and must not be silently missed.

**Requirements:**
- Window-based processing: a row per period, claimed atomically, so missed runs catch up.
- Cluster-wide single execution via a renewable lock with compare-and-delete release.
- Every run recorded with start, end, outcome, and counts.
- Absence alerting per job, with a configurable expected interval and grace.
- A chaos test killing the running instance mid-window; another completes it exactly once.

**Stretch:** dry-run mode producing a diff of what *would* be charged.

---

## 17. Order Saga Across Two Services

**Goal:** a multi-step process that is correct when a step fails.

**Requirements:**
- Two deployables — an order service and a fulfilment service — with a message broker between them.
- State changes and events written atomically via an outbox.
- Consumers with manual acks, in-transaction idempotency, bounded retries, and dead-lettering.
- A saga coordinating payment and fulfilment, with compensating commands for each failure path.
- `202 Accepted` with a status endpoint — not a `201` the client cannot yet read.

**Stretch:** a chaos harness killing each service at each step, asserting a consistent terminal state every time.

---

## 18. Observability for an Existing Service

**Goal:** instrument something you did not write, then find a real bottleneck.

**Requirements:**
- Structured logs with redaction and correlation propagated into queues.
- Request rate, error rate, and duration by route *pattern*, recorded on both paths.
- Event-loop lag and pool utilisation as gauges.
- Tracing initialised before anything it instruments, with tail-based sampling.
- Liveness with no external checks; readiness failing on shutdown.

**Then:** take one slow endpoint from symptom to fix — query, then blocking work, then cache — measuring after each step and writing up what did *not* help.

---

## 19. A Test Suite That Catches Real Bugs

**Goal:** prove your tests are worth their runtime.

**Requirements:**
- Fakes for every port, shared across the suite.
- Negative tests for every validation, authorization, and idempotency rule.
- Pipeline components tested directly — a guard, a pipe, an interceptor — with no HTTP.
- Integration tests against a real database in a container, isolated under parallel workers.
- Real JWTs in e2e tests, with explicit 401 and 403 cases on every protected route.

**The deliverable that matters:** a documented list of deliberate mutations — remove a decorator, invert a condition, drop an `await` — each paired with the test that catches it. Any mutation nothing catches is a gap.

---

## 20. Production SaaS Backend

**Goal:** everything, integrated, at a size where the decisions actually bind.

**Requirements:**
- Multi-tenant with isolation enforced at the data layer.
- Authentication with rotation, authorization via one rule set driving checks and queries.
- PostgreSQL with reviewed migrations, transactions, and no N+1 on any hot path.
- Queues for async work; scheduled jobs running once cluster-wide.
- Real-time updates working across replicas.
- Full observability, health checks, and graceful shutdown.
- Runs correctly at three replicas — caches, limits, sockets, and schedulers all verified.
- Feature-grouped modules with enforced import direction, and exactly one module using ports and adapters, with a written justification.

**Stretch:** an upgrade-readiness audit for v12, with an ESM migration branch and an effort estimate.

**This is a portfolio project.** Take the time. A well-built version of this answers most of [Appendix A](A-interview-questions.md) by itself, and gives you the three stories the [roadmap](D-roadmap.md) says to prepare.

---

**Next:** [Appendix D — The NestJS Roadmap →](D-roadmap.md)
