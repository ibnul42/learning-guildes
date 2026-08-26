# Chapter 19 — Prisma, and Choosing an ORM

## Learning Objectives

- Integrate Prisma into a Nest application with correct connection lifecycle
- Contrast Prisma's generated client with TypeORM's decorator entities
- Handle Prisma errors and map them to HTTP responses
- Choose between Prisma, TypeORM, Drizzle, and raw SQL from actual requirements
- Explain what each tool gives up in exchange for what it provides

---

## 19.1 Prisma Inverts the Model

TypeORM derives the schema from decorated classes. Prisma derives the client from a schema file, and the types are generated rather than declared.

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Task {
  id        String     @id @default(uuid()) @db.Uuid
  title     String     @db.VarChar(120)
  status    TaskStatus @default(OPEN)
  ownerId   String     @db.Uuid
  owner     User       @relation(fields: [ownerId], references: [id], onDelete: Cascade)
  comments  Comment[]
  createdAt DateTime   @default(now()) @db.Timestamptz
  updatedAt DateTime   @updatedAt @db.Timestamptz

  @@index([ownerId, status])
  @@map("tasks")
}

enum TaskStatus { OPEN DONE ARCHIVED }
```

```bash
npx prisma migrate dev --name add_tasks    # creates and applies a migration
npx prisma generate                        # regenerates the typed client
```

The consequence is the thing people actually notice: **query results are typed by what you selected**, not by the entity class.

```ts
const task = await prisma.task.findUnique({
  where: { id },
  select: { id: true, title: true },
});
// task is { id: string; title: string } | null — `status` is not on the type at all
```

With TypeORM, `select` narrows the SQL but the static type is still `Task`, so reading an unselected property compiles and is `undefined` at runtime. Prisma makes that a compile error. This is the strongest single argument for it.

---

## 19.2 Wiring Prisma Into Nest

Prisma has no Nest module of its own; you write a thin service that owns the connection lifecycle:

```ts
// src/prisma/prisma.service.ts
import { Injectable, OnModuleInit, OnApplicationShutdown, Logger } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnApplicationShutdown {
  private readonly logger = new Logger(PrismaService.name);

  constructor() {
    super({
      log: [{ emit: 'event', level: 'query' }, { emit: 'stdout', level: 'error' }],
    });
  }

  async onModuleInit(): Promise<void> {
    // Connect at boot so an unreachable database fails the deploy rather than
    // the first request. Prisma otherwise connects lazily.
    await this.$connect();
  }

  async onApplicationShutdown(): Promise<void> {
    await this.$disconnect();
  }
}
```

```ts
// src/prisma/prisma.module.ts
@Global()                       // the connection is genuine shared infrastructure
@Module({ providers: [PrismaService], exports: [PrismaService] })
export class PrismaModule {}
```

Remember that `onApplicationShutdown` only fires if `enableShutdownHooks()` was called (→ [Ch 9](09-scopes-lifecycle.md)).

Services then inject it:

```ts
@Injectable()
export class TasksService {
  constructor(private readonly prisma: PrismaService) {}

  findByOwner(ownerId: string) {
    return this.prisma.task.findMany({
      where: { ownerId, deletedAt: null },
      include: { owner: { select: { id: true, name: true } } },   // one JOIN, no N+1
      orderBy: { createdAt: 'desc' },
      take: 20,
    });
  }
}
```

> ⚠️ **Gotcha** — `PrismaService` extends `PrismaClient`, so injecting it exposes the entire database to any service that has it. There is no module boundary the way `forFeature` gives one for TypeORM repositories (→ [Ch 4](04-modules.md)). If you want that encapsulation, wrap Prisma in per-entity repository classes and export only those.

---

## 19.3 Transactions and Errors

Prisma offers two transaction forms:

```ts
// Sequential — all or nothing, no interdependence
const [order, audit] = await this.prisma.$transaction([
  this.prisma.order.create({ data: orderData }),
  this.prisma.auditEntry.create({ data: auditData }),
]);

// Interactive — when later steps depend on earlier results
await this.prisma.$transaction(async (tx) => {
  const order = await tx.order.create({ data: orderData });
  const updated = await tx.product.updateMany({
    where: { id: productId, stock: { gte: qty } },      // conditional, atomic
    data: { stock: { decrement: qty } },
  });
  if (updated.count === 0) throw new ConflictException('Insufficient stock');
  return order;
});
```

The same rule as TypeORM applies: **every query inside must use `tx`**, not the injected service, or it runs outside the transaction and survives a rollback (→ [Ch 18](18-migrations-transactions.md)).

Interactive transactions have a default 5-second timeout. Long-running work inside one aborts with `P2028`, so the timeout must be raised deliberately or the work moved out.

Prisma errors carry codes worth mapping in a filter (→ [Ch 15](15-exception-filters.md)):

| Code | Meaning | Status |
|---|---|---|
| `P2002` | unique constraint violation | 409 |
| `P2003` | foreign key constraint failure | 409 or 400 |
| `P2025` | record not found | 404 |
| `P2028` | transaction timed out | 503 |

```ts
@Catch(Prisma.PrismaClientKnownRequestError)
export class PrismaExceptionFilter implements ExceptionFilter {
  private static readonly MAP: Record<string, number> = {
    P2002: 409, P2003: 409, P2025: 404, P2028: 503,
  };

  catch(exception: Prisma.PrismaClientKnownRequestError, host: ArgumentsHost): void {
    const status = PrismaExceptionFilter.MAP[exception.code] ?? 500;
    // `meta.target` names the offending columns — useful in logs, and it
    // reveals schema details, so it must not go into the response.
    host.switchToHttp().getResponse<Response>().status(status).json({
      code: exception.code === 'P2002' ? 'DUPLICATE' : 'DB_ERROR',
      message: status === 409 ? 'Resource already exists' : 'Database error',
    });
  }
}
```

---

## 19.4 The Honest Comparison

| | Prisma | TypeORM | Drizzle | Raw SQL |
|---|---|---|---|---|
| Schema source | `.prisma` file | decorated classes | TypeScript objects | migrations |
| Result typing | exact, from `select` | entity class | exact | manual |
| Nest integration | manual service | official module | manual | manual |
| Migrations | strong, generated | generated, manual review | generated | hand-written |
| Complex SQL | escape to `$queryRaw` | query builder | close to SQL | native |
| Runtime | Node engine + client | pure TypeScript | thin, pure TS | driver only |
| Learning curve | low | moderate | low if you know SQL | SQL |

**Prisma's real strengths** are the generated types matching the query, migration tooling that is genuinely better than the alternatives, and a schema file that is readable by people who do not know your codebase.

**Prisma's real costs** are a separate schema language, a generation step that must run before the code type-checks (and in CI, and after every `git pull`), weaker support for genuinely complex SQL, and no module-level encapsulation in Nest.

**TypeORM's strengths** are the official Nest integration, `forFeature` giving real module boundaries, decorators keeping the schema next to the code, and a mature query builder for hard queries.

**TypeORM's costs** are looser typing — `select` does not narrow the type — the `synchronize` foot-gun, and an API with several ways to do the same thing that behave differently (→ [Ch 17](17-typeorm.md)).

**When to use raw SQL:** more often than people do. A reporting query with three CTEs and a window function is clearer as SQL than as any query builder, and both tools support parameterised raw queries. Using an ORM for writes and SQL for complex reads is a legitimate, common architecture.

> 💼 **Interview** — "Prisma or TypeORM?" There is no correct answer, and "whichever the team knows" is a weak one on its own. A strong answer picks a dimension that actually decides it: Prisma if type safety on query results matters most, TypeORM if module encapsulation and the official integration matter most, and either way note that complex reads will end up as raw SQL regardless.

---

## 19.5 Migrating Between Them

Rewriting a data layer wholesale is rarely justified. If you must, the tractable path is the same strangler-fig approach as any migration: both tools can point at the same database, so introduce the new one for new modules, and convert existing ones only when you are already changing them.

Two things make this survivable:

**A repository interface per aggregate** (→ [Ch 6](06-providers-services.md)) means the ORM appears in one file per entity rather than throughout the services. Without that seam, the ORM is spread across every service and the migration touches everything.

**One tool owns migrations.** Two migration systems against one database will conflict over the schema-history table and over the schema itself. Pick one, and have the other read a schema it does not manage — Prisma's `db pull` and TypeORM's entity generation both support this.

---

## 19.6 Common Mistakes and Best Practices

**Mistake: not connecting in `onModuleInit`.** Prisma connects lazily, so an unreachable database surfaces on the first request rather than at deploy.

**Mistake: forgetting `$disconnect`.** Connections leak across restarts, and in serverless environments they exhaust the database's connection limit.

**Mistake: using the injected service inside `$transaction`.** It runs outside the transaction and survives a rollback.

**Mistake: long work inside an interactive transaction.** The default 5-second timeout aborts it with `P2028`.

**Mistake: injecting `PrismaService` everywhere.** Every service gains access to every table, with no module boundary.

**Mistake: forgetting `prisma generate` in CI.** The build fails or, worse, type-checks against a stale client.

**Mistake: returning `meta.target` from a `P2002` in the response.** It names your columns to the client.

**Best practice: wrap Prisma in per-aggregate repositories.** You regain encapsulation and make a future ORM change a per-file job.

**Best practice: map Prisma error codes in one filter.** `P2002` should be a 409 everywhere, not a try/catch in each service.

**Best practice: use raw SQL for genuinely complex reads.** Both tools support parameterised raw queries, and a CTE is clearer than any builder chain.

---

## 19.7 Real-World Example: Prisma Behind a Repository Contract

Prisma used without leaking it into the domain — the service depends on a contract, so the ORM is confined to one file and the module boundary is restored.

```ts
// src/tasks/task.repository.ts — the contract the domain depends on
export abstract class TaskRepository {
  abstract findOwned(id: string, ownerId: string): Promise<Task | null>;
  abstract findPage(query: TaskQuery): Promise<{ data: Task[]; nextCursor: string | null }>;
  abstract create(input: CreateTaskInput): Promise<Task>;
  abstract completeIfOpen(id: string, ownerId: string): Promise<Task | null>;
}
```

```ts
// src/tasks/prisma-task.repository.ts — the ONLY file that imports Prisma
@Injectable()
export class PrismaTaskRepository extends TaskRepository {
  constructor(private readonly prisma: PrismaService) { super(); }

  // Ownership is part of the predicate, so another user's task is not
  // "found then rejected" — it does not exist for this caller (→ Ch 12).
  async findOwned(id: string, ownerId: string): Promise<Task | null> {
    const row = await this.prisma.task.findFirst({
      where: { id, ownerId, deletedAt: null },
      include: { owner: { select: { id: true, name: true } } },
    });
    return row ? toDomain(row) : null;
  }

  async findPage(query: TaskQuery) {
    const rows = await this.prisma.task.findMany({
      where: {
        ownerId: query.ownerId,
        deletedAt: null,
        ...(query.status && { status: query.status }),
        ...(query.search && {
          OR: [
            { title: { contains: query.search, mode: 'insensitive' } },
            { description: { contains: query.search, mode: 'insensitive' } },
          ],
        }),
      },
      // Keyset pagination: cursor plus a unique tiebreaker, so equal
      // timestamps cannot cause skipped rows (→ Ch 17).
      ...(query.cursor && { cursor: { id: query.cursor }, skip: 1 }),
      orderBy: [{ createdAt: 'desc' }, { id: 'desc' }],
      take: query.limit + 1,
    });

    const hasMore = rows.length > query.limit;
    const page = hasMore ? rows.slice(0, query.limit) : rows;
    return {
      data: page.map(toDomain),
      nextCursor: hasMore ? page.at(-1)!.id : null,
    };
  }

  async create(input: CreateTaskInput): Promise<Task> {
    return toDomain(await this.prisma.task.create({ data: input }));
  }

  /**
   * The status check is in the WHERE clause rather than a prior read, so two
   * concurrent completions cannot both succeed. updateMany returns a count
   * instead of throwing P2025, which makes "already completed" a normal
   * outcome rather than an exception to catch.
   */
  async completeIfOpen(id: string, ownerId: string): Promise<Task | null> {
    const result = await this.prisma.task.updateMany({
      where: { id, ownerId, status: 'OPEN' },
      data: { status: 'DONE', completedAt: new Date() },
    });
    return result.count === 0 ? null : this.findOwned(id, ownerId);
  }
}
```

```ts
// src/tasks/tasks.module.ts
@Module({
  imports: [PrismaModule],
  providers: [
    TasksService,
    { provide: TaskRepository, useClass: PrismaTaskRepository },   // swap point
  ],
  exports: [TasksService],
})
export class TasksModule {}
```

The decision worth defending is `updateMany` for a single-row update. `update` is the obvious call and it throws `P2025` when the `where` clause matches nothing — which conflates three different situations: the task does not exist, it belongs to someone else, or it was already completed. Catching that exception and guessing which occurred requires a second query, and the guess is racy. `updateMany` returns a count instead, so zero means "the precondition did not hold" and the service decides what that means with one follow-up read it was going to do anyway. The wider principle is that putting the precondition in the `WHERE` clause makes the check and the write atomic, which is the same reason `UPDATE … WHERE stock >= :qty` beats read-then-write — exceptions are a poor channel for expected outcomes, and a condition evaluated by the database cannot race.

---

## 19.8 Questions

**1.** MCQ — Prisma's schema lives in: a) decorated classes b) a `.prisma` file c) migrations d) JSON
<details><summary>Answer</summary>**b)** And the typed client is generated from it.</details>

**2.** MCQ — With Prisma's `select`, the result type is: a) the full model b) exactly the selected fields c) `any` d) a partial
<details><summary>Answer</summary>**b)** Which is its strongest advantage over TypeORM.</details>

**3.** MCQ — Prisma's interactive transaction default timeout is: a) 1s b) 5s c) 30s d) none
<details><summary>Answer</summary>**b)** Exceeding it aborts with `P2028`.</details>

**4.** MCQ — `P2002` means: a) not found b) unique constraint violation c) timeout d) foreign key failure
<details><summary>Answer</summary>**b)** Usually mapped to 409.</details>

**5.** MCQ — Without `$connect()` in `onModuleInit`, Prisma connects: a) never b) at boot c) lazily on first query d) per request
<details><summary>Answer</summary>**c)** So a broken database surfaces on the first request rather than at deploy.</details>

**6.** MCQ — What does TypeORM offer that Prisma does not, in Nest? a) migrations b) typing c) `forFeature` module boundaries d) relations
<details><summary>Answer</summary>**c)** Prisma's client exposes every model to any service that injects it.</details>

**7.** Short — Why does Prisma's typing beat TypeORM's for partial selects?
<details><summary>Answer</summary>Prisma's result type is derived from the `select`, so reading an unselected field is a compile error; TypeORM still types the result as the entity, so it compiles and is `undefined` at runtime.</details>

**8.** Short — Why wrap Prisma in repository classes?
<details><summary>Answer</summary>To restore the module boundary Prisma lacks, and to confine the ORM to one file per aggregate so a future change is not spread across every service.</details>

**9.** Short — What breaks if `prisma generate` is missing from CI?
<details><summary>Answer</summary>The build fails because the client does not exist, or — worse, if a stale client is cached — it type-checks against an out-of-date schema and passes while being wrong.</details>

**10.** Short — Why must `meta.target` from a `P2002` stay out of the response?
<details><summary>Answer</summary>It names the offending columns, exposing your schema to the client. Log it and return a generic conflict message.</details>

**11.** Short — When is raw SQL the right answer with either ORM?
<details><summary>Answer</summary>For genuinely complex reads — CTEs, window functions, recursive queries — where SQL is clearer than any builder chain. Both tools support parameterised raw queries.</details>

**12.** Output — `findUnique({ where: { id }, select: { id: true } })` then reading `.status`. What happens?
<details><summary>Answer</summary>A TypeScript compile error — `status` is not on the result type, because the type is derived from the `select`.</details>

**13.** Output — The equivalent TypeORM `findOne` with `select: { id: true }`, then reading `.status`. What happens?
<details><summary>Answer</summary>It compiles — the type is still `Task` — and the value is `undefined` at runtime.</details>

**14.** Output — Inside `$transaction(async (tx) => ...)`, a query uses `this.prisma` instead of `tx`, then the callback throws. What persists?
<details><summary>Answer</summary>That query's write. It ran outside the transaction on a separate connection, so the rollback does not remove it.</details>

**15.** Output — `updateMany` matching no rows returns what, and `update` matching no rows does what?
<details><summary>Answer</summary>`updateMany` returns `{ count: 0 }`; `update` throws `P2025`. Which is why `updateMany` is better for a conditional write.</details>

**16.** Debug — A serverless deployment exhausts the database connection limit after a few hours. Cause?
<details><summary>Answer</summary>`$disconnect` is never called, so each instance leaks its pool. Implement `onApplicationShutdown` and ensure `enableShutdownHooks()` is called.</details>

**17.** Debug — A long import inside `$transaction` fails with `P2028`. Fix?
<details><summary>Answer</summary>It exceeded the 5-second interactive transaction timeout. Raise `timeout` deliberately, or move the long work outside the transaction and keep only the atomic writes inside.</details>

**18.** Debug — A duplicate signup returns 500 instead of 409. What is missing?
<details><summary>Answer</summary>A filter mapping `Prisma.PrismaClientKnownRequestError` with code `P2002` to 409. Without it the error is unrecognised and becomes a generic 500.</details>

**19.** Debug — After `git pull`, the project no longer type-checks, with errors about missing Prisma types. Fix?
<details><summary>Answer</summary>The schema changed and the client is stale. Run `prisma generate`, and add it to a `postinstall` script so it cannot be forgotten.</details>

**20.** Debug — Two developers ran migrations from both Prisma and TypeORM against one database. What is the likely damage?
<details><summary>Answer</summary>Conflicting schema-history tables and a schema neither tool believes it owns. One tool must own migrations; the other should introspect a schema it does not manage.</details>

**21.** Code — Write a `PrismaService` with correct connection lifecycle.
<details><summary>Answer</summary>

```ts
@Injectable()
export class PrismaService extends PrismaClient
  implements OnModuleInit, OnApplicationShutdown {
  async onModuleInit() { await this.$connect(); }
  async onApplicationShutdown() { await this.$disconnect(); }
}
```
`enableShutdownHooks()` must be called in `main.ts` or the disconnect never runs.
</details>

**22.** Code — Map Prisma error codes to statuses in one filter.
<details><summary>Answer</summary>

```ts
@Catch(Prisma.PrismaClientKnownRequestError)
export class PrismaFilter implements ExceptionFilter {
  private static readonly MAP: Record<string, number> = {
    P2002: 409, P2003: 409, P2025: 404, P2028: 503,
  };
  catch(e: Prisma.PrismaClientKnownRequestError, host: ArgumentsHost) {
    const status = PrismaFilter.MAP[e.code] ?? 500;
    host.switchToHttp().getResponse<Response>()
      .status(status).json({ code: e.code, message: 'Request could not be completed' });
  }
}
```
</details>

**23.** Code — Write an atomic conditional completion that cannot double-apply.
<details><summary>Answer</summary>

```ts
async completeIfOpen(id: string, ownerId: string): Promise<boolean> {
  const { count } = await this.prisma.task.updateMany({
    where: { id, ownerId, status: 'OPEN' },      // precondition in the WHERE
    data: { status: 'DONE', completedAt: new Date() },
  });
  return count > 0;
}
```
Two concurrent callers cannot both see `count > 0`, because the condition and the write are one statement.
</details>

**24.** Scenario — Your team is starting a new Nest service and is split between Prisma and TypeORM. How do you decide?
<details><summary>Answer</summary>Decide on the dimension that will actually bite this project rather than on general preference. If the service has many partial selects and a large team where a typo becoming `undefined` at runtime is a realistic failure, Prisma's exact result typing is worth its separate schema language and generation step. If the codebase is large enough that module boundaries matter — several teams owning different aggregates — TypeORM's `forFeature` gives a real boundary that Prisma does not, and injecting a client exposing every table is a genuine architectural loss. If the workload is read-heavy with complex analytical queries, note that both will delegate those to raw SQL, so the choice matters less than it appears. What should not decide it is benchmark differences, which are dominated by your queries and indexes rather than the client.</details>

**25.** Scenario — An existing TypeORM codebase has performance problems and someone proposes migrating to Prisma to fix them. Evaluate it.
<details><summary>Answer</summary>Almost certainly the wrong remedy. ORM performance problems are overwhelmingly N+1 queries, missing indexes, unbounded page sizes, and over-fetching — all properties of how queries are written, and all of which reproduce identically in Prisma. Measure first: enable query logging, find the endpoints issuing hundreds of statements, and read the plans on the slow ones. A rewrite also carries risk unrelated to performance, since every query is retranslated by hand and subtle differences in null handling and transaction semantics introduce bugs the tests do not cover. If after measuring the problem genuinely is the ORM — which does happen on very heavy write paths — the proportionate fix is raw SQL for those specific queries, not replacing the data layer wholesale.</details>

---

## Chapter Summary

- Prisma generates a typed client from a schema file, so result types match the `select` exactly — the single strongest reason to choose it over TypeORM.
- It has no Nest module of its own: you own the connection lifecycle, and `$connect()` in `onModuleInit` is what makes a broken database fail the deploy rather than the first request.
- Injecting `PrismaService` exposes every model to every service, so per-aggregate repositories are how you regain the boundary `forFeature` gives TypeORM.
- Every query inside `$transaction` must use `tx`, and interactive transactions abort after five seconds by default.
- Map Prisma's error codes in one filter — `P2002` is a 409 everywhere — and never return `meta.target`, which names your columns.
- Putting a precondition in the `WHERE` clause with `updateMany` beats catching `P2025` from `update`, because a count distinguishes expected outcomes while an exception conflates them.

---

## Practice Exercises

1. Select two fields with Prisma and with TypeORM, then read a third, and compare what the compiler says.
2. Remove `$connect()` from `onModuleInit`, point at an unreachable database, and observe when the failure appears.
3. Run a query with `this.prisma` inside a `$transaction` callback, force a rollback, and confirm the write survives.
4. Write a filter mapping four Prisma error codes and prove a duplicate insert returns 409.
5. Wrap Prisma behind a repository contract and swap it for an in-memory implementation in a test.

---

## Mini Project: A Dual-ORM Benchmark

Build the same small API twice and compare them on evidence rather than opinion.

**Requirements:**

- Identical endpoints — list with filters, detail with relations, create, conditional update — on Prisma and TypeORM.
- Both behind the same repository contract, so the service layer and tests are shared.
- A query-count assertion per endpoint, proving neither implementation issues an N+1.
- Identical error-mapping behaviour verified by the same HTTP tests against both.
- A written comparison covering type safety, encapsulation, and lines of code.

---

**Next:** [Chapter 20 — Caching →](20-caching.md)
