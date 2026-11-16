# Maintainer Notes

This book is **complete**: 36 chapters and 7 appendices. This file used to be a continuation brief
for finishing it; it now records the conventions it was written to, so edits and additions stay
consistent.

For repository-wide conventions, see [../CLAUDE.md](../CLAUDE.md) and [../_template/](../_template/).

---

## 1. Current Status

**Complete — 36 chapters + 7 appendices, ~175,000 words.**

### Chapters (`chapters/`)

| # | File | Topic |
|---|---|---|
| 1 | `01-what-nestjs-is.md` | Express's gap, IoC, when not to use Nest, platform adapters |
| 2 | `02-typescript-decorators.md` | Decorators, `reflect-metadata`, `emitDecoratorMetadata`, DTO types |
| 3 | `03-first-application.md` | CLI, project anatomy, `bootstrap()`, request flow end to end |
| 4 | `04-modules.md` | `@Module`, encapsulation, exports, shared/global, dynamic modules |
| 5 | `05-controllers-routing.md` | Route decorators, params, versioning, response control |
| 6 | `06-providers-services.md` | `@Injectable`, stateless singletons, repository seam |
| 7 | `07-dependency-injection.md` | Container internals, token resolution, `@Inject`, circular deps |
| 8 | `08-custom-providers.md` | `useClass`/`useValue`/`useFactory`/`useExisting`, async providers |
| 9 | `09-scopes-lifecycle.md` | DEFAULT/REQUEST/TRANSIENT, bubbling, lifecycle hooks, shutdown |
| 10 | `10-request-lifecycle.md` | The full ordered pipeline — the chapter everything else references |
| 11 | `11-middleware.md` | `NestMiddleware`, `configure()`, functional middleware, ordering |
| 12 | `12-guards.md` | `CanActivate`, `ExecutionContext`, `Reflector`, metadata-driven auth |
| 13 | `13-interceptors.md` | `NestInterceptor`, RxJS operators, timeouts, caching, serialization |
| 14 | `14-pipes-validation.md` | Built-in pipes, `ValidationPipe`, the two fail-open traps |
| 15 | `15-exception-filters.md` | `HttpException`, filter scope, error contracts, RFC 9457 |
| 16 | `16-configuration.md` | `@nestjs/config`, namespaces, schema validation, secrets |
| 17 | `17-typeorm.md` | Entities, repositories, relations, N+1, keyset pagination |
| 18 | `18-migrations-transactions.md` | Safe migrations, transactions, locking, isolation |
| 19 | `19-prisma-choosing-orm.md` | Prisma integration, the honest comparison |
| 20 | `20-caching.md` | `CacheModule`, keys as a security boundary, stampedes |
| 21 | `21-authentication.md` | Passport, Argon2id, JWT, refresh rotation with reuse detection |
| 22 | `22-authorization.md` | RBAC/ABAC/ReBAC, CASL, defence in depth |
| 23 | `23-security.md` | Helmet, CORS, rate limits, injection, SSRF, uploads |
| 24 | `24-graphql.md` | Code-first, DataLoader, guards across transports, cost limits |
| 25 | `25-websockets.md` | Gateways, handshake auth, rooms, Redis adapter, presence |
| 26 | `26-microservices.md` | Message vs event patterns, transports, at-least-once, when not to |
| 27 | `27-queues-jobs.md` | BullMQ, retries, idempotency, lock duration, operations |
| 28 | `28-cqrs-events.md` | Commands, queries, outbox, sagas, event sourcing honestly |
| 29 | `29-unit-testing.md` | Testing module vs `new`, fakes vs mocks, tests that pass while broken |
| 30 | `30-e2e-testing.md` | Supertest, Testcontainers, isolation, real tokens |
| 31 | `31-file-uploads-streaming.md` | Multer, magic bytes, `StreamableFile`, pre-signed URLs |
| 32 | `32-task-scheduling.md` | `@Cron`, the multi-replica trap, distributed locks, absence alerting |
| 33 | `33-logging-observability.md` | Structured logs, correlation, metric cardinality, tracing, probes |
| 34 | `34-performance-scaling.md` | Measure first, event-loop blocking, scaling out, a worked fix |
| 35 | `35-architecture-patterns.md` | Layered, hexagonal applied selectively, monorepos, boundaries |
| 36 | `36-nestjs-11-to-12.md` | Version reference, ESM migration, upgrade audit |

### Appendices (`appendices/`)

| # | File | Contents |
|---|---|---|
| A | `A-interview-questions.md` | 340 questions, 13 sections, each with a **Follow-up:** line |
| B | `B-coding-challenges.md` | 112 challenges, 11 sections, worked solutions + common wrong answer |
| C | `C-projects.md` | 20 project specs, easy → expert, no solutions |
| D | `D-roadmap.md` | 7 phases, checkpoints, 4-week interview sprint, honest timelines |
| E | `E-best-practices.md` | 232 checklist items across 17 categories |
| F | `F-cheatsheet.md` | Decorator tables, pipeline order, provider recipes, a 20-row trap table |
| G | `G-glossary.md` | 186 terms, alphabetical, chapter cross-references |

---

## 2. Chapter Contract

Every chapter contains these sections, in this order. The canonical source is
[`../_template/chapters/00-chapter-template.md`](../_template/chapters/00-chapter-template.md).

```markdown
# Chapter N — Title
## Learning Objectives          3–5 bullets, action verbs
## N.1 … N.x                    4–6 numbered sections, explanation then commented code
## N.x Common Mistakes and Best Practices    5–7 mistakes, 2–4 best practices
## N.y Real-World Example: <Named Thing>     40–90 lines, then ONE paragraph on the
                                            single non-obvious decision
## N.z Questions                20–28, inline format, continuously numbered
## Chapter Summary              6 bullets, takeaways not topic names
## Practice Exercises           5 tasks, increasing difficulty, NO solutions
## Mini Project: <Name>
**Next:** [Chapter N+1 — Title →](NN-slug.md)
```

Question format, and the mix used throughout — MCQ (6) → Short (5) → Output (4) → Debug (4–5) →
Code (3) → Scenario (2):

```markdown
**1.** MCQ — question a) x b) y c) z d) w
<details><summary>Answer</summary>**b)** brief reason.</details>
```

**Tone:** direct, opinionated where justified, zero filler. Honest about trade-offs — the book says
when Nest is the wrong tool, when CQRS is overkill, and when an abstraction costs more than it
returns.

**Code style:** TypeScript throughout. `// output` as trailing comments. Wrong examples `// ❌`,
correct ones `// ✅`. Label a block with its file path whenever the target file is not obvious.
Prefer constructor injection, `readonly` dependencies, explicit return types on public methods, and
DTO classes over interfaces.

---

## 3. NestJS-Specific Conventions

**Version target is NestJS 11 (11.2.x) on Node 20/22 LTS.** Do not rewrite examples against v12
previews. Where v12 changes something, add a short flagged note instead:

> ⚠️ **v12** — This becomes ESM-only in NestJS 12. See [Chapter 36](chapters/36-nestjs-11-to-12.md).

**Chapter 10 owns the request pipeline order.** Every chapter touching the pipeline links to it
rather than restating the order. Keep it that way — a second copy will drift.

**Express is the default platform.** Anything Fastify-specific carries an explicit label. The
platform abstraction leaks, and the book says so rather than pretending otherwise.

**Examples build one coherent domain** — a task/project API with users, auth, and background jobs.
Later chapters extend earlier code rather than introducing a new toy. Keep entity and DTO names
stable.

**The real-world example's closing paragraph defends one decision.** Not a walkthrough of the code
the reader just read — the single choice they would otherwise question, and why the obvious
alternative is worse. This is the most distinctive convention in the book; preserve it.

---

## 4. Known Deviations

Deliberate, and recorded so nobody "fixes" them by accident.

**Chapters run longer than the JavaScript book's.** Chapters here are 3,175–3,971 words, against
1,800–4,500 for the JavaScript book and 1,800–3,000 for its early chapters. NestJS carries more
code per idea: the smallest honest illustration is a module plus a provider plus a consumer, and
most of the framework's real traps are configuration-shaped, needing the broken version *and* the
fixed one. Don't trim these, and don't pad the JavaScript book to match.

**Every chapter has a Real-World Example and a Mini Project.** The template marks Mini Projects
optional and the JavaScript book skips twelve; this book skips none, because every topic here
supports a buildable exercise. Chapter 36 is the closest call — its mini project is an upgrade
readiness report rather than code.

**Question counts are 24–25 rather than a fixed number.** The template allows 20–28. Chapters that
needed one more Scenario or Code question got one.

**Appendix B has 112 challenges and Appendix G has 186 terms**, against round-number targets of
~110 and ~200. Both are what the material actually supported. Padding either to hit a number would
have made it worse.

---

## 5. Verification

Before committing a change to any chapter or appendix, from `nestjs/`:

```bash
F=chapters/10-request-lifecycle.md

# <details> tags balance
echo "open=$(grep -c '<details>' $F) close=$(grep -c '</details>' $F)"

# Numbered and labelled questions must be equal, with no numbering gaps
grep -c '^\*\*[0-9]\+\.\*\*' $F
grep -c '^\*\*[0-9]\+\.\*\* \(MCQ\|Short\|Output\|Debug\|Code\|Scenario\) —' $F
grep -o '^\*\*[0-9]\+\.\*\*' $F | grep -o '[0-9]\+'

# Section numbers match the filename's chapter number
grep -o '^## [0-9]\+\.[0-9]\+' $F

# Code fences balance, and TypeScript blocks are tagged ts
grep -c '```' $F
grep -n '```javascript' $F
```

For the appendices, the counts declared in each contents table must match reality — Appendix A's
13 section counts, B's 11, and E's 17 are all checkable programmatically, and all three have been
verified against the files.

Whole-book checks:

- Every internal link resolves, including each file's `**Next:**` footer. The chain runs
  ch01 → … → ch36 → Appendix A → … → Appendix G (G has no footer; it ends the book).
- Cross-references (`→ Ch 10`, `Chapter 14`) name a chapter that exists and covers that topic.
- No `*(pending)*` rows remain in the README, and every row links to a real file.

> ⚠️ A naive link checker reports two false positives here: the `NN-slug.md` placeholder inside the
> fenced code block in §2 above, and a `logger[...]({` call in Chapter 33 that matches a `](...)`
> link pattern. Neither is a real link.

---

## 6. Adding Material

Chapters 1–36 form a deliberate progression, so new material usually belongs in an appendix.
If a chapter genuinely belongs in the sequence:

1. Copy [`../_template/chapters/00-chapter-template.md`](../_template/chapters/00-chapter-template.md).
2. Insert it at the end rather than renumbering — renumbering breaks every cross-reference and
   every `**Next:**` footer, and there are hundreds.
3. Update the previous chapter's footer and add a row to the README table.
4. Run the §5 checks.

**Do not renumber chapters.** This is the one rule with no exceptions.
