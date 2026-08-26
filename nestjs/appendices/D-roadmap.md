# Appendix D — The NestJS Roadmap

A staged plan, with checkpoints you can honestly test yourself against.

**Read this first, because it changes how you'll use everything below.** NestJS itself is small — you can write controllers and services after a weekend. What takes months is everything the framework *assumes*: dependency injection as a design tool, transactions, concurrency, authorization that holds outside HTTP, and the operational judgement to run the thing. Anyone promising NestJS mastery in two weeks is describing the syntax, which was never the hard part.

The plan below assumes **10–15 hours a week**, that you already write JavaScript comfortably, and that you have built at least one HTTP API before. If TypeScript is new to you, add three weeks to Phase 1 and expect Chapter 2 to be slow — it is the chapter everything else rests on.

Scale the durations to your actual time. The *order* matters more than the timeline.

---

## The Phases at a Glance

| Phase | Focus | Chapters | Duration |
|---|---|---|---|
| 1 | Foundations and the container | 1–9 | 3–4 weeks |
| 2 | The request pipeline | 10–15 | 2–3 weeks |
| 3 | Data | 16–20 | 3–4 weeks |
| 4 | Auth and security | 21–23 | 2–3 weeks |
| 5 | Beyond REST | 24–28 | 3–5 weeks |
| 6 | Quality and production | 29–36 | 4–5 weeks |
| 7 | Depth and specialisation | — | ongoing |

Roughly **4–6 months** to competent-and-employable on a NestJS team, at this pace. Phases 1–6 are this book; phase 7 is not.

**If you only have four weeks and an interview**, skip to the [sprint](#the-four-week-interview-sprint) — but come back, because it teaches you to answer questions rather than to build systems.

---

## Phase 1 — Foundations and the Container

**Chapters 1–9 · 3–4 weeks**

This is the phase people rush and then pay for. Chapters 2, 7, and 9 are the ones that make every later error message legible: if you understand `design:paramtypes`, token resolution, and scope bubbling, you will diagnose problems instead of guessing at them.

Build alongside: a small task API with three or four modules, real DI, and no database yet.

**You're ready to move on when you can:**

- Explain why a DTO must be a class, without saying "because Nest requires it".
- Read a "Nest can't resolve dependencies of X (?, Y)" error and name the fix before opening any file.
- Say what `exports` does and what happens if you register one provider in two modules.
- Choose between `useValue`, `useClass`, `useFactory`, and `useExisting` for a given requirement.
- Describe what happens between `NestFactory.create()` and `listen()`.
- Explain what a request-scoped provider does to everything above it.

> ⚠️ **Do not skip Chapter 2.** Almost every confusing NestJS error is a metadata problem wearing a different hat.

---

## Phase 2 — The Request Pipeline

**Chapters 10–15 · 2–3 weeks**

The most heavily interviewed part of the framework and the most immediately useful. Chapter 10 is the reference; the rest are its details.

Build alongside: add authentication guards (fake them for now), a validation layer, a response envelope, and one global exception filter to the Phase 1 API.

**You're ready to move on when you can:**

- Recite the pipeline order and explain *why* guards precede pipes.
- Say which components can see the response, and which know the target handler.
- Write a guard that reads a decorator with `Reflector` and choose override versus merge deliberately.
- Explain why an interceptor using `tap(fn)` produces dashboards that lie.
- Name both ways validation fails open, and write the test that catches each.
- Design an error contract a client could program against.

---

## Phase 3 — Data

**Chapters 16–20 · 3–4 weeks**

Where most production bugs actually live. Take Chapter 18 slowly — transactions and locking are the topics where "it works on my machine" is most misleading, because concurrency bugs need concurrency to appear.

Build alongside: put the task API on PostgreSQL, with migrations, a transactional multi-step operation, and caching added *after* the queries are correct.

**You're ready to move on when you can:**

- Diagnose an N+1 from a query log and fix it two different ways.
- Explain why `synchronize: true` is dangerous in concrete terms.
- Write a migration that is safe to run against a live table during a rolling deploy.
- Explain why an injected repository inside a transaction callback is a bug.
- Choose between optimistic and pessimistic locking for a given operation.
- Name what must be in a cache key, and what happens when the principal is missing.

---

## Phase 4 — Auth and Security

**Chapters 21–23 · 2–3 weeks**

Short, and disproportionately consequential. This is the phase where being *nearly* right is worse than being obviously wrong, because a broken authorization check produces no error.

Build alongside: replace the fake guards with real JWT authentication, refresh-token rotation, and CASL-driven authorization enforced in the service layer.

**You're ready to move on when you can:**

- Explain why refresh-token reuse must revoke a whole family.
- Say why login must be indistinguishable in *timing*, not just in message.
- Explain why ownership checks belong in the query, and why 404 beats 403.
- Name what a guard cannot protect, and what covers that gap.
- Explain what `origin: true` and `trust proxy: true` actually permit.
- Describe how mass assignment happens and the single option that closes it.

---

## Phase 5 — Beyond REST

**Chapters 24–28 · 3–5 weeks**

The widest phase and the most skippable. Read all five chapters, then go deep on whatever your work or target role actually uses. Queues (27) are the most broadly useful; CQRS (28) is the most commonly over-applied.

Build alongside: add a queue with an idempotent processor, and one of GraphQL, WebSockets, or a second service — whichever matches your goals.

**You're ready to move on when you can:**

- Explain why N+1 is structural in GraphQL and what DataLoader does about it.
- Say what breaks in a WebSocket app the moment you add a second replica.
- Distinguish message patterns from event patterns and justify preferring events.
- Explain at-least-once delivery and where idempotency must be enforced.
- Describe the outbox pattern and the problem it solves that neither alternative does.
- Argue *against* microservices for a specific system.

---

## Phase 6 — Quality and Production

**Chapters 29–36 · 4–5 weeks**

The phase that separates people who can build a NestJS app from people you can put on call. Chapters 33 and 34 will change how you write the earlier chapters' code.

Build alongside: bring the task API to production shape — tests that catch real bugs, structured logs with correlation, health checks, graceful shutdown, and a load test.

**You're ready to move on when you can:**

- Explain why a test can pass while the code is broken, and name four ways.
- Say why overriding the auth guard in every e2e test defeats the suite's purpose.
- Explain why liveness must not check the database.
- Diagnose a p50/p99 divergence and name the first thing you would measure.
- Explain what scaling from one replica to three silently breaks.
- Describe the v12 ESM migration and what can be done now, on v11.

---

## Phase 7 — Depth and Specialisation

**Ongoing**

The book ends; the work does not. Reasonable next directions, in no particular order:

**Operate something.** Nothing in Phases 1–6 substitutes for being paged at 3am. Deploy a service, put real traffic on it, and break it.

**Go deeper on the database.** Query plans, index design, isolation semantics, and replication will pay off more than any framework knowledge.

**Learn the platform you deploy to.** Kubernetes, ECS, or a PaaS — the failure modes in Chapters 32–34 are shaped by it.

**Pick a hard domain.** Payments, real-time collaboration, or multi-tenancy each force the patterns in this book past their comfortable range.

**Contribute.** Reading `@nestjs/core`'s injector is the fastest way to make Chapter 7 permanent.

---

## The Four-Week Interview Sprint

For an interview in a month, with the fundamentals already in place. This is triage, not education — it optimises for answering well, which is not the same as building well.

### Week 1 — The container

Chapters 1–9. Do not skim Chapter 7.

- Be able to draw the module graph of an app you have built.
- Practise the DI error walk-through out loud, in under thirty seconds.
- Write, from memory: a dynamic module with `forRootAsync`, and a factory provider with `inject`.

### Week 2 — The pipeline

Chapters 10–15. This is the highest-yield week.

- Recite the order until it is automatic.
- Be able to justify the ordering, not just recall it.
- Write a guard, an interceptor, a pipe, and a filter from scratch, without looking.
- Know both fail-open validation traps cold — they come up constantly.

### Week 3 — Data and auth

Chapters 16–23, focusing on 17, 18, 21, 22.

- Explain N+1 with a concrete before/after and a query count.
- Explain transactions, the `manager` trap, and both locking strategies.
- Explain JWT revocation honestly — "you can't, and here's what you do instead".
- Explain why ownership belongs in the query.

### Week 4 — Production judgement and rehearsal

Chapters 29–34, then [Appendix A](A-interview-questions.md) end to end.

- Work through the interview questions, saying answers aloud rather than reading them.
- Do [Appendix B](B-coding-challenges.md)'s DI, pipeline, and testing sections.
- Prepare three stories: a bug you diagnosed, a design decision you defended, and a thing you got wrong.
- Prepare one honest opinion — where NestJS is the wrong tool, and why.

**The single highest-value preparation:** be able to walk through what happens between a request arriving and your controller method running, in order, with reasons. It is asked in some form in nearly every NestJS interview, and answering it well signals more than any list of features you can name.

---

## Honest Notes on Timelines

**"I know NestJS" means different things.** Writing controllers and services: two weeks. Debugging a resolution error without guessing: two months. Designing module boundaries that still make sense a year later: considerably longer, and mostly learned from getting it wrong.

**The framework is not the hard part.** Concurrency, transactions, and authorization are hard in every framework. NestJS gives you structure for them; it does not give you judgement about them.

**Reading is not building.** Every phase above has a "build alongside" line for a reason. A chapter you have read and not applied will not be there when you need it under pressure.

**The gaps that show up in interviews** are consistently: scope bubbling, transaction boundaries, at-least-once delivery, and what breaks at more than one replica. If you are short on time, those four are where to spend it.

---

**Next:** [Appendix E — Best Practices Checklist →](E-best-practices.md)
