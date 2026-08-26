# Chapter 17 — Databases with TypeORM

## Learning Objectives

- Wire TypeORM into a Nest application with async configuration
- Define entities with correct column types, indexes, and relations
- Inject and use repositories, and choose between `save`, `insert`, and `update`
- Diagnose and fix an N+1 query problem
- Explain why `synchronize: true` must never run against a production database

---

## 17.1 Wiring It Up

```bash
npm i @nestjs/typeorm typeorm pg
```

The root connection is configured once, asynchronously, so it can read validated configuration (→ [Ch 16](16-configuration.md)):

```ts
// src/database/database.module.ts
@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      inject: [databaseConfig.KEY],
      useFactory: (config: ConfigType<typeof databaseConfig>) => ({
        type: 'postgres',
        url: config.url,
        autoLoadEntities: true,        // entities registered via forFeature
        synchronize: false,            // NEVER true outside a scratch database
        migrationsRun: config.runMigrations,
        logging: config.logging ? ['query', 'error'] : ['error'],
        poolSize: config.poolSize,
        extra: {
          // Bound how long a connection can wait; the default is unlimited,
          // which turns a slow database into an unbounded queue of requests.
          connectionTimeoutMillis: 5_000,
          statement_timeout: 30_000,
        },
      }),
    }),
  ],
})
export class DatabaseModule {}
```

Feature modules then register the entities they own:

```ts
// src/tasks/tasks.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([Task, TaskComment])],
  providers: [TasksService],
  controllers: [TasksController],
})
export class TasksModule {}
```

`forFeature` is the `forFeature` convention from Chapter 4 doing exactly what it claims: registering a scoped slice of an already-root-configured module. It provides a `Repository<Task>` token inside that module only.

> ⚠️ **Gotcha** — `synchronize: true` inspects your entities and alters the schema to match. It will **drop columns** whose properties you removed, and it does so without prompting. It is acceptable in a disposable local database and catastrophic anywhere with data you care about. Set it `false` and use migrations (→ [Ch 18](18-migrations-transactions.md)).

---

## 17.2 Entities

An entity is a class mapped to a table:

```ts
// src/tasks/task.entity.ts
import {
  Column, CreateDateColumn, DeleteDateColumn, Entity, Index,
  ManyToOne, OneToMany, PrimaryGeneratedColumn, UpdateDateColumn,
} from 'typeorm';

export enum TaskStatus { Open = 'open', Done = 'done', Archived = 'archived' }

@Entity('tasks')
@Index(['ownerId', 'status'])            // composite: matches the common query
export class Task {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ length: 120 })               // becomes varchar(120), not unbounded text
  title: string;

  @Column({ type: 'text', nullable: true })
  description: string | null;

  @Column({ type: 'enum', enum: TaskStatus, default: TaskStatus.Open })
  status: TaskStatus;

  // Money as an integer count of minor units. `decimal` returns a STRING from
  // the pg driver, and `float` cannot represent 0.1 — both cause real bugs.
  @Column({ type: 'integer', default: 0 })
  estimateMinutes: number;

  @Column({ type: 'uuid' })
  ownerId: string;

  @ManyToOne(() => User, (user) => user.tasks, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'ownerId' })
  owner: User;

  @OneToMany(() => TaskComment, (comment) => comment.task)
  comments: TaskComment[];

  @CreateDateColumn({ type: 'timestamptz' })
  createdAt: Date;

  @UpdateDateColumn({ type: 'timestamptz' })
  updatedAt: Date;

  @DeleteDateColumn({ type: 'timestamptz' })   // enables soft delete
  deletedAt: Date | null;
}
```

Four column choices there are worth copying.

**`timestamptz`, not `timestamp`.** Without the time zone, the same instant reads differently depending on the server's zone, and the bug appears only when a deployment moves region or the clocks change.

**Integers for money.** `decimal` is returned as a *string* by the PostgreSQL driver — arithmetic on it silently concatenates — and `float` cannot represent `0.1`. Store minor units as integers.

**Explicit lengths.** `@Column()` on a string defaults to `varchar(255)`; declaring the real limit documents it and matches the validation on the DTO.

**A composite index matching the query.** `@Index(['ownerId', 'status'])` serves `WHERE ownerId = ? AND status = ?` and also `WHERE ownerId = ?` alone, because a composite index can be used left-to-right. The reverse order would not serve the second query.

**Soft deletes** come from `@DeleteDateColumn`. `softRemove()` and `softDelete()` set it, and every subsequent `find` excludes those rows automatically — which is convenient and easy to forget when writing raw SQL or a query builder with `withDeleted()`.

---

## 17.3 Repositories

Inject a repository with `@InjectRepository`:

```ts
@Injectable()
export class TasksService {
  constructor(
    @InjectRepository(Task)
    private readonly tasks: Repository<Task>,
  ) {}

  findOne(id: string): Promise<Task | null> {
    return this.tasks.findOne({ where: { id } });
  }
}
```

The write methods are not interchangeable, and choosing wrongly is a common performance bug:

| Method | SQL | Notes |
|---|---|---|
| `save(entity)` | `SELECT` then `INSERT`/`UPDATE` | runs hooks and cascades; **two queries** |
| `insert(dto)` | `INSERT` | one query, no hooks, no cascades |
| `update(criteria, dto)` | `UPDATE` | one query, no hooks, returns no entity |
| `upsert(dto, keys)` | `INSERT … ON CONFLICT` | atomic, avoids check-then-insert races |
| `delete(criteria)` | `DELETE` | hard delete |
| `softDelete(criteria)` | `UPDATE … SET deletedAt` | needs `@DeleteDateColumn` |

`save` is the default reach and does the most work: it queries first to decide between insert and update, runs subscribers, and cascades to relations. For a straightforward creation, `insert` is one query instead of two.

```ts
// ❌ Two queries, and the returned entity is often discarded anyway
await this.tasks.save(this.tasks.create(dto));

// ✅ One query when you do not need hooks or the hydrated entity
await this.tasks.insert(dto);
```

`save` also has a trap worth knowing: **passing a partial object with an id performs a partial update**, silently leaving other columns alone, while passing one *without* an id inserts a new row. A DTO missing its id where you expected an update creates a duplicate.

**Finding:**

```ts
this.tasks.find({
  where: { ownerId, status: TaskStatus.Open },
  relations: { owner: true },            // JOIN, avoiding N+1
  select: { id: true, title: true },     // fewer columns over the wire
  order: { createdAt: 'DESC' },
  skip: (page - 1) * limit,
  take: limit,
});
```

Operators cover most filtering without a query builder:

```ts
import { In, LessThan, ILike, IsNull, Not, Between } from 'typeorm';

where: {
  status: In([TaskStatus.Open, TaskStatus.Done]),
  dueDate: LessThan(new Date()),
  title: ILike(`%${search}%`),           // parameterised — not string interpolation
  deletedAt: IsNull(),
}
```

> ⚠️ **Gotcha** — `findOne({ where: {} })` with an empty object returns the **first row in the table**, not `null`. If a filter value is `undefined`, TypeORM drops that condition rather than matching nothing. An unchecked `findOne({ where: { id: req.params.id } })` where the param is absent returns an arbitrary record — which, on a user table, is an authorization bypass.

---

## 17.4 Relations and the N+1 Problem

The single most common database performance bug in a Nest application:

```ts
// ❌ 1 query for the tasks, then 1 per task for its owner — 101 queries for 100 tasks
const tasks = await this.tasks.find();
for (const task of tasks) {
  console.log(task.owner.name);          // lazy load fires here
}

// ✅ 1 query with a JOIN
const tasks = await this.tasks.find({ relations: { owner: true } });
```

It is easy to introduce accidentally and invisible until the table grows. Two habits prevent it:

**Never use `eager: true`.** It loads the relation on *every* query touching that entity, including ones that do not need it — you cannot opt out per query, only opt in. Explicit `relations` at each call site is more typing and always correct.

**Log queries in development.** `logging: ['query']` makes an N+1 obvious immediately — a wall of near-identical `SELECT`s — instead of at the first load test.

For deeper control, the query builder handles what the find options cannot:

```ts
const rows = await this.tasks
  .createQueryBuilder('task')
  .leftJoinAndSelect('task.owner', 'owner')
  .where('task.ownerId = :ownerId', { ownerId })              // ALWAYS parameterised
  .andWhere('task.createdAt > :since', { since })
  .orderBy('task.createdAt', 'DESC')
  .take(20)
  .getMany();
```

**Every value must be a bound parameter.** Interpolating into the string is SQL injection:

```ts
// ❌ SQL injection
.where(`task.title = '${search}'`)

// ✅
.where('task.title = :search', { search })
```

Column and table *names* cannot be parameterised, so a client-supplied sort column must be checked against an allowlist before it reaches `orderBy` (→ [Ch 14](14-pipes-validation.md)).

> 💼 **Interview** — "What is the N+1 problem and how do you detect it?" Name the cause — a loop touching a lazily-loaded relation — the fix (`relations` or a join), and the detection method (query logging, or asserting the query count in a test). Adding that `eager: true` trades one problem for a worse one signals real experience.

---

## 17.5 Common Mistakes and Best Practices

**Mistake: `synchronize: true` against a real database.** It silently drops columns to match your entities.

**Mistake: `eager: true` on relations.** Every query pays for the join, and there is no way to opt out per call.

**Mistake: `findOne` with a possibly-`undefined` filter.** The condition is dropped and an arbitrary row is returned.

**Mistake: `decimal` or `float` for money.** `decimal` comes back as a string; `float` cannot represent `0.1`.

**Mistake: `timestamp` instead of `timestamptz`.** The same instant reads differently after a region change.

**Mistake: `save()` where `insert()` would do.** It costs an extra `SELECT` on every write.

**Mistake: string interpolation in a query builder.** It is SQL injection; bind parameters instead.

**Best practice: return DTOs, not entities.** Entities expose the schema and can trigger lazy loads during serialization (→ [Ch 13](13-interceptors.md)).

**Best practice: enable query logging in development.** N+1 problems announce themselves immediately.

**Best practice: index for the queries you actually run.** A composite index works left-to-right, so column order in the index is a decision, not a formality.

---

## 17.6 Real-World Example: A Repository With Correct Pagination

A tasks repository handling filtering, search, and pagination without N+1 queries, and using keyset pagination where offset would break down.

```ts
// src/tasks/task.repository.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, Brackets } from 'typeorm';
import { Task, TaskStatus } from './task.entity';

export interface TaskQuery {
  ownerId: string;
  status?: TaskStatus;
  search?: string;
  cursor?: string;
  limit: number;
}

// Allowlist: column names cannot be bound as parameters, so a client-supplied
// sort field must be validated against a fixed set before reaching the SQL.
const SORTABLE = { createdAt: 'task.createdAt', title: 'task.title' } as const;

@Injectable()
export class TaskRepository {
  constructor(
    @InjectRepository(Task)
    private readonly repo: Repository<Task>,
  ) {}

  async findPage(query: TaskQuery, sort: keyof typeof SORTABLE = 'createdAt') {
    const qb = this.repo
      .createQueryBuilder('task')
      // A single join rather than a lazy load per row.
      .leftJoinAndSelect('task.owner', 'owner')
      .where('task.ownerId = :ownerId', { ownerId: query.ownerId });

    if (query.status) {
      qb.andWhere('task.status = :status', { status: query.status });
    }

    if (query.search) {
      // Brackets keep the OR grouped; without them it would combine with the
      // ownerId condition and leak other users' tasks.
      qb.andWhere(
        new Brackets((w) => {
          w.where('task.title ILIKE :q', { q: `%${query.search}%` })
            .orWhere('task.description ILIKE :q', { q: `%${query.search}%` });
        }),
      );
    }

    // Keyset pagination: OFFSET must scan and discard every skipped row, so
    // page 5,000 gets progressively slower and rows shift under concurrent
    // inserts. A cursor is O(1) regardless of depth and stable under writes.
    if (query.cursor) {
      qb.andWhere('task.createdAt < :cursor', { cursor: new Date(query.cursor) });
    }

    const rows = await qb
      .orderBy(SORTABLE[sort], 'DESC')
      .addOrderBy('task.id', 'DESC')     // tiebreaker: without it, equal
      .take(query.limit + 1)             // timestamps make the cursor unstable
      .getMany();

    const hasMore = rows.length > query.limit;
    const page = hasMore ? rows.slice(0, query.limit) : rows;

    return {
      data: page,
      nextCursor: hasMore ? page.at(-1)!.createdAt.toISOString() : null,
    };
  }

  /** Atomic: no check-then-insert race between concurrent callers. */
  async upsertByExternalId(dto: UpsertTaskDto): Promise<void> {
    await this.repo.upsert(dto, {
      conflictPaths: ['externalId'],
      skipUpdateIfNoValuesChanged: true,
    });
  }

  /**
   * Ownership is a query predicate rather than a post-load check, so a task
   * belonging to someone else is not "found and rejected" — it does not exist
   * for this caller at all (→ Ch 12).
   */
  async findOwned(id: string, ownerId: string): Promise<Task | null> {
    return this.repo.findOne({ where: { id, ownerId } });
  }
}
```

The decision worth defending is keyset pagination with a tiebreaker rather than `skip`/`take`. Offset pagination is simpler, supports jumping to an arbitrary page, and is correct for small result sets — but the database must scan and discard every skipped row, so latency grows linearly with page depth, and any insert between two requests shifts every subsequent page, causing rows to be duplicated or skipped as a user scrolls. The tiebreaker on `task.id` is the part most implementations omit: ordering by a timestamp alone means rows sharing a millisecond have no defined order, so the cursor can land mid-group and silently drop records. Adding a unique column to the sort makes the ordering total, which is what makes the cursor exact. The honest trade-off is that keyset pagination cannot jump to page 50 — if the interface needs numbered pages, offset is the right choice and the depth should be capped.

---

## 17.7 Questions

**1.** MCQ — `synchronize: true` in production: a) is recommended b) can drop columns c) only adds columns d) is a no-op
<details><summary>Answer</summary>**b)** It alters the schema to match entities, including dropping columns for removed properties.</details>

**2.** MCQ — `save()` on a new entity issues: a) one query b) two c) three d) none
<details><summary>Answer</summary>**b)** A `SELECT` to decide insert versus update, then the write.</details>

**3.** MCQ — `eager: true` on a relation means: a) load on demand b) load on every query c) never load d) load in a transaction
<details><summary>Answer</summary>**b)** Which is why explicit `relations` per query is preferred.</details>

**4.** MCQ — `findOne({ where: { id: undefined } })` returns: a) null b) an error c) the first row d) an empty array
<details><summary>Answer</summary>**c)** The undefined condition is dropped, so it matches everything and returns the first row.</details>

**5.** MCQ — Money should be stored as: a) `float` b) `decimal` c) integer minor units d) `varchar`
<details><summary>Answer</summary>**c)** `float` cannot represent `0.1`, and `decimal` is returned as a string by the pg driver.</details>

**6.** MCQ — A composite index on `[a, b]` can serve a query filtering only on: a) `b` b) `a` c) neither d) both individually
<details><summary>Answer</summary>**b)** Composite indexes are usable left-to-right.</details>

**7.** Short — What causes an N+1 query problem?
<details><summary>Answer</summary>Fetching a collection and then touching a lazily-loaded relation inside a loop, so each row triggers its own query.</details>

**8.** Short — Why is `eager: true` a bad fix for N+1?
<details><summary>Answer</summary>It joins the relation on every query touching that entity, including ones that never use it, and there is no way to opt out per call.</details>

**9.** Short — Why `timestamptz` rather than `timestamp`?
<details><summary>Answer</summary>Without a time zone, the same instant is interpreted differently depending on the server's zone, so the bug appears only after a region change or a clock shift.</details>

**10.** Short — Why is `upsert` better than checking existence then inserting?
<details><summary>Answer</summary>It is a single atomic statement, so two concurrent callers cannot both pass the existence check and then both insert.</details>

**11.** Short — Why return DTOs rather than entities?
<details><summary>Answer</summary>Entities expose every column including future ones, and touching a lazy relation during serialization triggers extra queries.</details>

**12.** Output — 100 tasks each with an owner, fetched with `find()` then accessed in a loop. How many queries?
<details><summary>Answer</summary>101 — one for the tasks and one per task for its owner.</details>

**13.** Output — `repo.save({ id: 'abc', title: 'new' })` on an entity with ten columns. What happens to the other nine?
<details><summary>Answer</summary>They are left unchanged — `save` with an id performs a partial update of the supplied properties only.</details>

**14.** Output — `repo.save({ title: 'new' })` with no id, where a row with that title exists. What happens?
<details><summary>Answer</summary>A new row is inserted. Without an id, `save` cannot match an existing record, which is how accidental duplicates appear.</details>

**15.** Output — An entity has `@DeleteDateColumn` and a row was soft-deleted. What does `repo.find()` return for it?
<details><summary>Answer</summary>Nothing — soft-deleted rows are excluded automatically unless you pass `withDeleted: true`.</details>

**16.** Debug — An endpoint takes 8 seconds with 500 records and 40ms with 5. Where do you look?
<details><summary>Answer</summary>An N+1 query. Enable query logging, confirm the repeated `SELECT`s, and load the relation with `relations` or a join.</details>

**17.** Debug — A column vanished from production after a deploy. Cause?
<details><summary>Answer</summary>`synchronize: true` removed it to match an entity whose property was deleted. Set it false and use migrations.</details>

**18.** Debug — Summing a `decimal` price column produces string concatenation. Why?
<details><summary>Answer</summary>The PostgreSQL driver returns `decimal` as a string to preserve precision, so `+` concatenates. Store minor units as integers, or convert explicitly.</details>

**19.** Debug — A search endpoint with an OR condition started returning other users' records. What is the likely mistake?
<details><summary>Answer</summary>The OR was added without `Brackets`, so it combined with the ownership condition as `owner = x AND title LIKE y OR description LIKE z`, and the last clause matches regardless of owner.</details>

**20.** Debug — Users report items appearing twice while scrolling an infinite list. Cause?
<details><summary>Answer</summary>Offset pagination with concurrent inserts shifts rows between pages. Switch to keyset pagination with a unique tiebreaker in the sort.</details>

**21.** Code — Fetch tasks with their owner and comment count, without N+1.
<details><summary>Answer</summary>

```ts
this.repo
  .createQueryBuilder('task')
  .leftJoinAndSelect('task.owner', 'owner')
  .loadRelationCountAndMap('task.commentCount', 'task.comments')
  .where('task.ownerId = :ownerId', { ownerId })
  .getMany();
```
`loadRelationCountAndMap` adds one aggregate query rather than one per row.
</details>

**22.** Code — Write a safe sortable query where the client supplies the column.
<details><summary>Answer</summary>

```ts
const SORTABLE = { createdAt: 'task.createdAt', title: 'task.title' } as const;

findSorted(sort: string, dir: 'ASC' | 'DESC') {
  const column = SORTABLE[sort as keyof typeof SORTABLE];
  if (!column) throw new BadRequestException(`sort must be one of ${Object.keys(SORTABLE)}`);
  return this.repo.createQueryBuilder('task').orderBy(column, dir).getMany();
}
```
Column names cannot be bound as parameters, so an allowlist is the only safe approach.
</details>

**23.** Code — Implement keyset pagination with a stable tiebreaker.
<details><summary>Answer</summary>

```ts
const rows = await this.repo
  .createQueryBuilder('t')
  .where(cursor ? '(t.createdAt, t.id) < (:date, :id)' : '1=1', {
    date: cursor?.date,
    id: cursor?.id,
  })
  .orderBy('t.createdAt', 'DESC')
  .addOrderBy('t.id', 'DESC')
  .take(limit + 1)
  .getMany();
```
The row-value comparison `(a, b) < (x, y)` is the exact form; ordering by timestamp alone leaves ties undefined and can skip records.
</details>

**24.** Scenario — A teammate proposes enabling `eager: true` on all relations so nobody has to remember `relations`. Evaluate it.
<details><summary>Answer</summary>It replaces a visible problem with an invisible one. N+1 queries at least announce themselves in a query log as repeated statements; eager loading spreads the cost evenly across every query touching the entity, including a simple existence check that now joins four tables. Because it is opt-out-impossible, the only remedy at a call site that does not need the relation is a query builder that bypasses the find API entirely — so the convenience evaporates exactly where performance matters. The real answer to "nobody remembers" is tooling rather than defaults: log queries in development, assert query counts in tests for hot endpoints, and review the relation list at the repository layer where these decisions are concentrated rather than scattered across services.</details>

**25.** Scenario — Your `tasks` table has grown to 40 million rows and the list endpoint has become slow despite an index. What do you investigate?

<details><summary>Answer</summary>Start by reading the actual plan with `EXPLAIN ANALYZE` rather than reasoning about the index, because at that size the common causes are not "no index" but an index the planner declines to use. Check whether the filter is selective enough to beat a sequential scan, whether a function or a type mismatch on the column makes the index unusable, and whether offset pagination is forcing the database to scan and discard millions of rows. Also check the index order against the query: a composite index only helps left-to-right, so an index on `[status, ownerId]` does nothing for a query filtering on `ownerId` alone. If the plan is genuinely optimal and it is still slow, the problem is the data volume rather than the query, and the remedies are partitioning by a time range, a covering index that avoids the heap lookup, or moving the list to a read replica — in that order of cost.</details>

---

## Chapter Summary

- Configure the connection asynchronously so it reads validated config, and keep `synchronize: false` — it alters the schema to match entities and will drop columns.
- Column types matter more than they look: `timestamptz` over `timestamp`, integer minor units over `decimal` or `float`, and explicit lengths on strings.
- `save` costs an extra `SELECT` and does a partial update when given an id — `insert`, `update`, and `upsert` are one query each and say what they mean.
- `findOne` drops conditions whose value is `undefined`, so an unchecked parameter returns an arbitrary row rather than nothing.
- N+1 comes from touching a lazy relation in a loop; fix it with explicit `relations` or a join, and never with `eager: true`, which makes every query pay.
- Values in a query builder must be bound parameters, and column names — which cannot be bound — need an allowlist.

---

## Practice Exercises

1. Enable query logging, write a loop that triggers an N+1, and count the statements before and after adding `relations`.
2. Store a price as `decimal`, sum it in JavaScript, and observe the string concatenation. Convert to integer minor units.
3. Call `findOne` with an `undefined` filter value and confirm it returns an arbitrary row.
4. Compare `save`, `insert`, and `upsert` for the same creation and count the queries each issues.
5. Implement offset and keyset pagination over 100,000 rows and measure page 1 against page 2,000 for both.

---

## Mini Project: A Repository Layer With Query Budgets

Build a data layer that makes query count a tested property.

**Requirements:**

- Entities for users, tasks, and comments with correct types and indexes chosen from real queries.
- A repository exposing filtered, searched, keyset-paginated results with a sortable-column allowlist.
- Ownership enforced as a query predicate, never as a post-load check.
- A test helper that counts queries per request and fails if an endpoint exceeds its declared budget.
- Soft delete for tasks, with a test proving deleted rows are excluded from every read path.

---

**Next:** [Chapter 18 — Migrations, Transactions, and Queries →](18-migrations-transactions.md)
