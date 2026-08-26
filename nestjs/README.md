# NestJS: From Fundamentals to Expert

**A practical, interview-ready guide to NestJS 11 — the framework, the architecture behind it, and the decisions you'll defend in a senior interview.**

> **Status — complete.** All 36 chapters and 7 appendices: ~175,000 words, 1,239 questions
> with answers (899 in the chapters, 340 in Appendix A), and 112 coding challenges with worked
> solutions.

---

## Who This Book Is For

You know JavaScript and you've built an HTTP API before — probably with Express, probably by wiring routes to handlers by hand. You've heard NestJS described as "Angular for the backend" and you're not sure whether that's a recommendation or a warning.

This book does not teach you what an HTTP verb is. It teaches you why NestJS resolves a provider the way it does, what actually runs between the request arriving and your controller method being called, why your `REQUEST`-scoped provider quietly made the whole call chain request-scoped, and how to test a module without booting the world. NestJS is a small amount of framework wrapped around a large amount of *convention* — and the convention is the part interviews probe.

## How to Use This Book

Four passes work well:

1. **Linear read** — Chapters 1–36 in order. Each builds on the last.
2. **Guided path** — [Appendix D](appendices/D-roadmap.md) stages the book with "you're ready to move on when you can…" checkpoints.
3. **Interview prep** — Chapters 7–15 are the core of any NestJS interview, then
   [Appendix A](appendices/A-interview-questions.md) and [B](appendices/B-coding-challenges.md).
4. **Reference** — [Appendix F](appendices/F-cheatsheet.md) (cheat sheet) and
   [G](appendices/G-glossary.md) (glossary) once you're working.

Every chapter follows the same shape:

- Learning objectives
- Explanations with runnable, commented code
- Common mistakes and best practices
- A real-world example
- 20–28 questions (MCQ, short answer, output prediction, debugging, coding, scenario)
- Chapter summary
- Practice exercises
- Mini project (where the topic supports one)

Type the code out. Reading a decorator and knowing what metadata it emits are different skills, and only one of them survives an interview.

---

## Table of Contents

### Part I — Foundations

| # | Chapter | Core Idea |
|---|---------|-----------|
| 1 | [What NestJS Is and Why It Exists](chapters/01-what-nestjs-is.md) | The problem Express leaves you with |
| 2 | [TypeScript and Decorators for NestJS](chapters/02-typescript-decorators.md) | Metadata is the whole trick |
| 3 | [Your First Application](chapters/03-first-application.md) | CLI, bootstrap, project anatomy |
| 4 | [Modules](chapters/04-modules.md) | The unit of encapsulation |
| 5 | [Controllers and Routing](chapters/05-controllers-routing.md) | Turning HTTP into method calls |
| 6 | [Providers and Services](chapters/06-providers-services.md) | Where the business logic goes |

### Part II — Dependency Injection

| # | Chapter | Core Idea |
|---|---------|-----------|
| 7 | [Dependency Injection Deep Dive](chapters/07-dependency-injection.md) | The IoC container, resolved step by step |
| 8 | [Custom Providers](chapters/08-custom-providers.md) | `useClass`, `useValue`, `useFactory`, tokens |
| 9 | [Provider Scopes and Lifecycle](chapters/09-scopes-lifecycle.md) | Singleton, request, transient, and hooks |

### Part III — The Request Pipeline

| # | Chapter | Core Idea |
|---|---------|-----------|
| 10 | [The Request Lifecycle](chapters/10-request-lifecycle.md) | The exact order everything runs in |
| 11 | [Middleware](chapters/11-middleware.md) | The Express layer underneath |
| 12 | [Guards](chapters/12-guards.md) | Can this request proceed? |
| 13 | [Interceptors](chapters/13-interceptors.md) | Wrapping the handler, both directions |
| 14 | [Pipes and Validation](chapters/14-pipes-validation.md) | DTOs, `class-validator`, transformation |
| 15 | [Exception Filters](chapters/15-exception-filters.md) | Turning throws into responses |

### Part IV — Data

| # | Chapter | Core Idea |
|---|---------|-----------|
| 16 | [Configuration and Environments](chapters/16-configuration.md) | `@nestjs/config`, schema validation |
| 17 | [Databases with TypeORM](chapters/17-typeorm.md) | Entities, repositories, relations |
| 18 | [Migrations, Transactions, and Queries](chapters/18-migrations-transactions.md) | Production data work |
| 19 | [Prisma, and Choosing an ORM](chapters/19-prisma-choosing-orm.md) | The honest comparison |
| 20 | [Caching](chapters/20-caching.md) | Cache manager, Redis, invalidation |

### Part V — Authentication and Security

| # | Chapter | Core Idea |
|---|---------|-----------|
| 21 | [Authentication](chapters/21-authentication.md) | Passport, JWT, refresh rotation |
| 22 | [Authorization](chapters/22-authorization.md) | RBAC, ABAC, CASL |
| 23 | [Securing a NestJS API](chapters/23-security.md) | Helmet, CORS, rate limits, secrets |

### Part VI — Beyond REST

| # | Chapter | Core Idea |
|---|---------|-----------|
| 24 | [GraphQL](chapters/24-graphql.md) | Code-first, resolvers, N+1 |
| 25 | [WebSockets and Gateways](chapters/25-websockets.md) | Real-time, rooms, auth |
| 26 | [Microservices and Transports](chapters/26-microservices.md) | Patterns, TCP, Redis, gRPC, Kafka |
| 27 | [Queues and Background Jobs](chapters/27-queues-jobs.md) | BullMQ, retries, idempotency |
| 28 | [CQRS and Event-Driven Design](chapters/28-cqrs-events.md) | Commands, queries, sagas |

### Part VII — Quality and Production

| # | Chapter | Core Idea |
|---|---------|-----------|
| 29 | [Unit Testing](chapters/29-unit-testing.md) | `Test.createTestingModule`, doubles |
| 30 | [Integration and E2E Testing](chapters/30-e2e-testing.md) | Supertest, real databases |
| 31 | [File Uploads and Streaming](chapters/31-file-uploads-streaming.md) | Multipart, streams, `StreamableFile` |
| 32 | [Task Scheduling](chapters/32-task-scheduling.md) | Cron, intervals, and the multi-instance trap |
| 33 | [Logging and Observability](chapters/33-logging-observability.md) | Structured logs, tracing, health checks |
| 34 | [Performance and Scaling](chapters/34-performance-scaling.md) | Fastify, clustering, measuring |
| 35 | [Architecture Patterns](chapters/35-architecture-patterns.md) | Layered, hexagonal, monorepo, DDD |
| 36 | [NestJS 11 → 12 and the Ecosystem](chapters/36-nestjs-11-to-12.md) | Version reference and migration |

### Appendices

| # | Appendix | Contents |
|---|----------|----------|
| A | [Interview Questions](appendices/A-interview-questions.md) | By topic and level, each with a follow-up |
| B | [Coding Challenges](appendices/B-coding-challenges.md) | Worked solutions and common wrong answers |
| C | [Real-World Projects](appendices/C-projects.md) | Easy → expert, specs only |
| D | [The NestJS Roadmap](appendices/D-roadmap.md) | Staged plan, plus an interview sprint |
| E | [Best Practices Checklist](appendices/E-best-practices.md) | Reviewable items |
| F | [Cheat Sheet](appendices/F-cheatsheet.md) | Decorators, CLI, lifecycle, tables throughout |
| G | [Glossary](appendices/G-glossary.md) | Cross-referenced to chapters |

---

## Conventions

Code blocks are runnable unless marked otherwise. Output is shown as a trailing comment:

```ts
const doubled = [1, 2, 3].map((n) => n * 2);
console.log(doubled); // [2, 4, 6]
```

A file path labels the block whenever the file it belongs in is not obvious:

```ts
// src/users/users.controller.ts
@Controller('users')
export class UsersController {}
```

Warnings look like this:

> ⚠️ **Gotcha** — Injecting a `REQUEST`-scoped provider into a singleton silently promotes that
> singleton to request scope, and everything that depends on it too.

Interview-relevant points are flagged:

> 💼 **Interview** — "Walk me through what happens between the request arriving and your controller
> method running" is the single most common NestJS interview question.

Questions carry an inline type label and a collapsible answer, so you can self-test:

**1.** MCQ — Which runs first? a) pipe b) guard c) interceptor d) filter
<details><summary>Answer</summary>**b)** Guards run before interceptors, which run before pipes. Filters run last, and only on a throw.</details>

Wrong and right examples are marked `// ❌` and `// ✅`.

Shell commands are shown without a prompt character, so they can be copied directly:

```bash
nest generate resource users
```

---

## A Note on Environment

Everything here targets **NestJS 11** (11.2.x) on **Node.js 20 LTS or 22 LTS**, with TypeScript 5.x and `experimentalDecorators` enabled — the default in a `nest new` project.

NestJS 12 is in development for approximately Q3 2026, and it is a significant break: full ESM across all official packages, Standard Schema support in route decorators, and a default toolchain that swaps Jest for Vitest, ESLint for oxlint, and Webpack for Rspack. Nothing in Chapters 1–35 assumes it. [Chapter 36](chapters/36-nestjs-11-to-12.md) covers what changes and how to prepare, and anything v12 will alter is flagged where it appears.

Where an example is Express-specific or Fastify-specific, it's labeled — NestJS runs on both, and the platform leaks through more often than the docs suggest.

Start with [Chapter 1](chapters/01-what-nestjs-is.md).
