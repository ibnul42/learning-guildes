# Chapter 18 — Migrations, Transactions, and Queries

## Learning Objectives

- Generate, run, and revert migrations, and explain why `synchronize` is not an alternative
- Write a migration that is safe to run against a live table
- Wrap multi-step work in a transaction and propagate it through service calls
- Choose between optimistic and pessimistic locking for a concurrent update
- Pick an isolation level deliberately rather than accepting the default

---

## 18.1 Migrations Are the Schema's History

`synchronize: true` computes a diff between your entities and the database and applies it. Migrations record an explicit, ordered, reviewable list of changes instead. The difference is not convenience — it is whether a schema change can be reviewed, tested, and reversed.

The CLI needs a `DataSource` defined outside Nest's DI, because it runs without an application:

```ts
// src/database/data-source.ts
import 'dotenv/config';
import { DataSource } from 'typeorm';

export default new DataSource({
  type: 'postgres',
  url: process.env.DATABASE_URL,
  entities: ['src/**/*.entity.ts'],
  migrations: ['src/database/migrations/*.ts'],
  migrationsTableName: 'migrations',
});
```

```jsonc
// package.json
"scripts": {
  "typeorm": "typeorm-ts-node-commonjs -d src/database/data-source.ts",
  "migration:generate": "npm run typeorm -- migration:generate",
  "migration:run": "npm run typeorm -- migration:run",
  "migration:revert": "npm run typeorm -- migration:revert"
}
```

```bash
npm run migration:generate -- src/database/migrations/AddTaskPriority
npm run migration:run
```

A generated migration has an `up` and a `down`:

```ts
export class AddTaskPriority1735689600000 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(
      `ALTER TABLE "tasks" ADD "priority" integer NOT NULL DEFAULT 0`,
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`ALTER TABLE "tasks" DROP COLUMN "priority"`);
  }
}
```

**Always read a generated migration before committing it.** Generation compares entities to the current database, so if your local database has drifted, the migration encodes that drift. Reviewing it catches the destructive statements before they reach a pull request.

> ⚠️ **Gotcha** — `migrationsRun: true` in the application config runs migrations at boot. With more than one replica, every instance starts and races to migrate. TypeORM takes a lock so the schema does not corrupt, but instances block until it clears, and a slow migration can exhaust a deployment's startup timeout. Run migrations as a separate deploy step.

---

## 18.2 Migrations That Are Safe on a Live Table

A migration runs while the application is serving traffic, and the old code is still running during a rolling deploy. Two rules follow.

**Adding a `NOT NULL` column with no default rewrites the table and fails if rows exist.** Split it into steps:

```ts
// Step 1 — nullable, deployable immediately
await queryRunner.query(`ALTER TABLE "tasks" ADD "priority" integer`);

// Step 2 — backfill in batches, so one statement does not lock the table
await queryRunner.query(`UPDATE "tasks" SET "priority" = 0 WHERE "priority" IS NULL`);

// Step 3 — only after all code writes the column
await queryRunner.query(`ALTER TABLE "tasks" ALTER COLUMN "priority" SET NOT NULL`);
```

**Renaming a column breaks the running old code instantly.** The safe sequence is expand, migrate, contract: add the new column, write to both, backfill, switch reads, then drop the old one — across several deploys. A rename is one migration and an outage; the expand-contract version is four and none.

**Index creation locks writes.** PostgreSQL's `CREATE INDEX CONCURRENTLY` does not, but it cannot run inside a transaction, so the migration must opt out:

```ts
export class AddTaskOwnerIndex1735689600000 implements MigrationInterface {
  transaction = false;                 // required for CONCURRENTLY

  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(
      `CREATE INDEX CONCURRENTLY "idx_tasks_owner_status" ON "tasks" ("ownerId", "status")`,
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`DROP INDEX CONCURRENTLY "idx_tasks_owner_status"`);
  }
}
```

---

## 18.3 Transactions

Any operation writing more than one row needs a transaction, or a partial failure leaves inconsistent data.

The callback form is the one to reach for — it commits on success and rolls back on any throw:

```ts
@Injectable()
export class OrdersService {
  constructor(private readonly dataSource: DataSource) {}

  async placeOrder(dto: CreateOrderDto): Promise<Order> {
    return this.dataSource.transaction(async (manager) => {
      const order = await manager.save(Order, Order.from(dto));

      for (const item of dto.items) {
        // Conditional update: the WHERE clause is the concurrency control.
        // Read-then-write would let two callers both see stock and oversell.
        const result = await manager
          .createQueryBuilder()
          .update(Product)
          .set({ stock: () => 'stock - :qty' })
          .where('id = :id AND stock >= :qty', { id: item.productId, qty: item.quantity })
          .setParameter('qty', item.quantity)
          .execute();

        if (result.affected === 0) {
          throw new ConflictException(`Insufficient stock for ${item.productId}`);
        }
      }
      return order;
    });
  }
}
```

**Every query inside must use the provided `manager`.** A repository injected into the service has its own connection and runs *outside* the transaction — so it is not rolled back:

```ts
await this.dataSource.transaction(async (manager) => {
  await manager.save(Order, order);        // ✅ in the transaction
  await this.auditRepo.save(auditEntry);   // ❌ separate connection, survives a rollback
});
```

This is the most common transaction bug in Nest, and it is silent: everything works until a rollback happens, and then the data that should have vanished is still there.

Passing `manager` down through service calls is explicit and tedious. The alternative is `AsyncLocalStorage`-based propagation (→ [Ch 9](09-scopes-lifecycle.md)) — the approach `typeorm-transactional` packages — where a decorator establishes a transaction that nested calls join automatically. It removes the parameter threading at the cost of making the transaction implicit, which is the same trade discussed in Chapter 9.

---

## 18.4 Locking

Two rows updated concurrently produce a lost update: both read, both compute, the second write overwrites the first.

**Optimistic locking** adds a version column and fails the second writer:

```ts
@Entity()
export class Task {
  @VersionColumn()                       // incremented on every save
  version: number;
}

// The UPDATE includes `WHERE version = :expected`; a mismatch means someone
// else wrote first, and TypeORM throws OptimisticLockVersionMismatchError.
await manager.save(Task, task, { reload: true });
```

It costs nothing when there is no contention, which makes it right for the common case — a user editing a form. The caller must handle the conflict, usually by returning 409 and letting the client retry with fresh data.

**Pessimistic locking** takes a database lock and makes the second reader wait:

```ts
await this.dataSource.transaction(async (manager) => {
  const account = await manager.findOne(Account, {
    where: { id },
    lock: { mode: 'pessimistic_write' },   // SELECT … FOR UPDATE
  });
  account!.balance -= amount;
  await manager.save(account!);
});
```

It is correct under heavy contention and it serialises access, so it costs throughput and can deadlock if two transactions lock the same rows in different orders. Always acquire locks in a consistent order.

| | Optimistic | Pessimistic |
|---|---|---|
| Cost with no contention | none | a lock round-trip |
| Behaviour on conflict | fails, caller retries | waits |
| Deadlock risk | none | yes |
| Right for | user-facing edits | balances, counters, inventory |

---

## 18.5 Isolation Levels

The default in PostgreSQL and MySQL is `READ COMMITTED`, which prevents dirty reads and permits non-repeatable reads: the same query twice in one transaction can return different rows.

| Level | Prevents | Still allows |
|---|---|---|
| `READ UNCOMMITTED` | nothing | dirty reads |
| `READ COMMITTED` | dirty reads | non-repeatable, phantom reads |
| `REPEATABLE READ` | non-repeatable reads | phantoms (in some engines) |
| `SERIALIZABLE` | everything | nothing — but transactions may abort |

```ts
await this.dataSource.transaction('SERIALIZABLE', async (manager) => {
  // …
});
```

Raise the level only for an operation whose correctness depends on it — a balance check followed by a withdrawal, a "reserve the last seat" flow. `SERIALIZABLE` makes transactions abort with a serialization failure under contention, so any code using it must be prepared to retry.

Most of the time, a conditional `UPDATE … WHERE` (as in §18.3) achieves the same guarantee at `READ COMMITTED` with no retries, because the database evaluates the condition and the write atomically. Reach for the concurrency-safe *statement* before reaching for a stronger isolation level.

---

## 18.6 Common Mistakes and Best Practices

**Mistake: using an injected repository inside a transaction callback.** It runs on a different connection and is not rolled back.

**Mistake: `migrationsRun: true` with multiple replicas.** Instances race and block on the migration lock during startup.

**Mistake: committing a generated migration unread.** Generation encodes local schema drift, including destructive statements.

**Mistake: adding a `NOT NULL` column in one step.** It rewrites the table and fails when rows exist.

**Mistake: renaming a column in one migration.** The old code is still running during a rolling deploy and breaks immediately.

**Mistake: `CREATE INDEX` without `CONCURRENTLY` on a large table.** It blocks writes for the duration.

**Mistake: read-then-write for counters and stock.** Two callers both read the old value; use a conditional update.

**Best practice: run migrations as a deploy step, not at application boot.** It keeps startup fast and the migration observable.

**Best practice: write and test the `down` migration.** A migration you cannot revert is a one-way door discovered during an incident.

**Best practice: prefer a conditional `UPDATE` to a higher isolation level.** It is atomic, needs no retry loop, and costs nothing.

---

## 18.7 Real-World Example: A Transactional Transfer With Retry

Money moving between two accounts — the canonical case where a partial write is unacceptable, deadlocks are possible, and optimistic conflicts must be retried.

```ts
// src/accounts/transfer.service.ts
import { Injectable, Logger, BadRequestException, ConflictException } from '@nestjs/common';
import { DataSource, EntityManager } from 'typeorm';

@Injectable()
export class TransferService {
  private readonly logger = new Logger(TransferService.name);
  private static readonly MAX_ATTEMPTS = 3;

  constructor(private readonly dataSource: DataSource) {}

  async transfer(fromId: string, toId: string, amountMinor: number): Promise<Transfer> {
    if (amountMinor <= 0) throw new BadRequestException('Amount must be positive');
    if (fromId === toId) throw new BadRequestException('Cannot transfer to the same account');

    for (let attempt = 1; attempt <= TransferService.MAX_ATTEMPTS; attempt++) {
      try {
        return await this.attempt(fromId, toId, amountMinor);
      } catch (err) {
        // Serialization failures and deadlocks are expected under contention
        // and are safe to retry — the transaction rolled back completely.
        if (this.isRetryable(err) && attempt < TransferService.MAX_ATTEMPTS) {
          const backoff = 2 ** attempt * 25;
          this.logger.warn(`Transfer retry ${attempt} after ${backoff}ms`);
          await new Promise((r) => setTimeout(r, backoff));
          continue;
        }
        throw err;
      }
    }
    throw new ConflictException('Transfer failed after retries');
  }

  private attempt(fromId: string, toId: string, amountMinor: number): Promise<Transfer> {
    return this.dataSource.transaction('REPEATABLE READ', async (manager) => {
      // Lock in a deterministic order. Two concurrent opposite transfers that
      // locked in argument order would deadlock; sorting makes that impossible.
      const [firstId, secondId] = [fromId, toId].sort();
      await this.lock(manager, firstId);
      await this.lock(manager, secondId);

      // The balance check and the debit are one statement, so no window exists
      // between them — this is what makes overdraft impossible rather than rare.
      const debit = await manager
        .createQueryBuilder()
        .update(Account)
        .set({ balanceMinor: () => 'balance_minor - :amount' })
        .where('id = :id AND balance_minor >= :amount', { id: fromId, amount: amountMinor })
        .setParameter('amount', amountMinor)
        .execute();

      if (debit.affected === 0) {
        // Distinguish "no such account" from "not enough money" for the caller.
        const exists = await manager.countBy(Account, { id: fromId });
        throw exists
          ? new ConflictException('Insufficient funds')
          : new NotFoundException(`Account ${fromId} not found`);
      }

      const credit = await manager
        .createQueryBuilder()
        .update(Account)
        .set({ balanceMinor: () => 'balance_minor + :amount' })
        .where('id = :id', { id: toId })
        .setParameter('amount', amountMinor)
        .execute();

      if (credit.affected === 0) {
        // Throwing rolls back the debit too — that is the whole point.
        throw new NotFoundException(`Account ${toId} not found`);
      }

      // Same manager, so the ledger entry shares the transaction's fate.
      return manager.save(Transfer, {
        fromId, toId, amountMinor, occurredAt: new Date(),
      });
    });
  }

  private lock(manager: EntityManager, id: string): Promise<unknown> {
    return manager.findOne(Account, { where: { id }, lock: { mode: 'pessimistic_write' } });
  }

  private isRetryable(err: unknown): boolean {
    const code = (err as { code?: string })?.code;
    return code === '40001' || code === '40P01';   // serialization_failure, deadlock_detected
  }
}
```

The decision worth defending is sorting the two account ids before locking. Without it the code reads more naturally — lock the source, then the destination — and it deadlocks under exactly the traffic you would expect: A transferring to B while B transfers to A, each holding the lock the other needs. The database detects it and aborts one transaction, so it is not corruption, but it surfaces as intermittent failures under load that are almost impossible to reproduce deliberately. Sorting imposes a global ordering on lock acquisition, which is the standard and complete solution to lock-ordering deadlocks: if every transaction in the system acquires locks in the same order, a cycle cannot form. The retry loop remains because `REPEATABLE READ` can still abort with a serialization failure, and those are safe to retry precisely because the transaction rolled back entirely — a partial retry would be far more dangerous than no retry.

---

## 18.8 Questions

**1.** MCQ — `synchronize: true` versus migrations: migrations are preferred because they are: a) faster b) reviewable and reversible c) simpler d) automatic
<details><summary>Answer</summary>**b)** They are an explicit, ordered, testable record of schema change.</details>

**2.** MCQ — A repository injected into a service, used inside `dataSource.transaction()`: a) joins the transaction b) runs outside it c) throws d) blocks
<details><summary>Answer</summary>**b)** It uses its own connection, so its writes survive a rollback.</details>

**3.** MCQ — `CREATE INDEX CONCURRENTLY` requires: a) a transaction b) no transaction c) a lock d) superuser
<details><summary>Answer</summary>**b)** It cannot run inside one, so the migration must set `transaction = false`.</details>

**4.** MCQ — Optimistic locking detects a conflict using: a) a row lock b) a version column c) an advisory lock d) `SERIALIZABLE`
<details><summary>Answer</summary>**b)** `@VersionColumn` plus a `WHERE version = :expected` clause.</details>

**5.** MCQ — The default isolation level in PostgreSQL is: a) `READ UNCOMMITTED` b) `READ COMMITTED` c) `REPEATABLE READ` d) `SERIALIZABLE`
<details><summary>Answer</summary>**b)**</details>

**6.** MCQ — Deadlocks between two transfers are prevented by: a) retries b) a consistent lock order c) higher isolation d) shorter transactions
<details><summary>Answer</summary>**b)** A global ordering on lock acquisition makes a cycle impossible.</details>

**7.** Short — Why should migrations not run at application boot in a multi-replica deployment?
<details><summary>Answer</summary>Every instance races to migrate; they block on the migration lock, and a slow migration can exhaust the deployment's startup timeout.</details>

**8.** Short — Why is renaming a column in one migration unsafe?
<details><summary>Answer</summary>During a rolling deploy the old code is still running and references the old name, so it breaks the moment the migration applies. Use expand-migrate-contract.</details>

**9.** Short — Why is a conditional `UPDATE … WHERE stock >= :qty` better than read-then-write?
<details><summary>Answer</summary>The condition and the write are evaluated atomically by the database, so there is no window in which two callers both see sufficient stock.</details>

**10.** Short — When is optimistic locking the right choice?
<details><summary>Answer</summary>When contention is rare — user-facing edits — because it costs nothing when there is no conflict and only requires the caller to handle a 409.</details>

**11.** Short — Why must a generated migration be read before committing?
<details><summary>Answer</summary>Generation diffs entities against your local database, so any local drift is encoded into the migration, potentially including destructive statements.</details>

**12.** Output — Inside a transaction, `manager.save(Order)` succeeds and `this.repo.save(Audit)` succeeds, then the callback throws. What persists?
<details><summary>Answer</summary>The audit row. It was written on a separate connection outside the transaction, so the rollback does not remove it.</details>

**13.** Output — Two concurrent requests read `stock = 1` and both write `stock = 0` after decrementing. What is the final stock and what was sold?
<details><summary>Answer</summary>Stock is 0 and two units were sold from one — a lost update. A conditional `UPDATE` prevents it.</details>

**14.** Output — `ALTER TABLE tasks ADD priority integer NOT NULL` on a table with a million rows. What happens?
<details><summary>Answer</summary>It fails — existing rows have no value — or, with a default, rewrites the whole table while holding a lock. Add it nullable, backfill, then set `NOT NULL`.</details>

**15.** Output — Two transactions lock accounts A then B, and B then A, simultaneously. What does the database do?
<details><summary>Answer</summary>Detects the deadlock and aborts one with SQLSTATE `40P01`. The aborted transaction rolled back fully and can be retried.</details>

**16.** Debug — A rollback left an audit record behind. Cause?
<details><summary>Answer</summary>The audit write used an injected repository rather than the transaction's `manager`, so it ran on a separate connection.</details>

**17.** Debug — Deploys intermittently time out during startup after migrations were enabled. Diagnosis?
<details><summary>Answer</summary>`migrationsRun: true` with several replicas: they contend for the migration lock and block. Move migrations to a separate deploy step.</details>

**18.** Debug — An inventory system oversells during flash sales but never in testing. Cause?
<details><summary>Answer</summary>Read-then-write on the stock column. It requires concurrent requests to interleave, which testing does not produce. Use a conditional update.</details>

**19.** Debug — Adding an index took the site down for four minutes. What was missing?
<details><summary>Answer</summary>`CONCURRENTLY`. A plain `CREATE INDEX` blocks writes for the duration of the build.</details>

**20.** Debug — `SERIALIZABLE` transactions fail intermittently with SQLSTATE `40001`. Is this a bug?
<details><summary>Answer</summary>No — it is the level working as designed. Serialization failures are expected under contention and must be retried by the caller.</details>

**21.** Code — Write a transactional method where every write participates in the transaction.
<details><summary>Answer</summary>

```ts
async createWithAudit(dto: CreateDto, actorId: string): Promise<Order> {
  return this.dataSource.transaction(async (manager) => {
    const order = await manager.save(Order, Order.from(dto));
    await manager.save(AuditEntry, { actorId, action: 'order.created', targetId: order.id });
    return order;
  });
}
```
Both use `manager`, so both roll back together.
</details>

**22.** Code — Write a safe three-step migration adding a `NOT NULL` column.
<details><summary>Answer</summary>

```ts
// Migration 1
await q.query(`ALTER TABLE "tasks" ADD "priority" integer`);
// Migration 2 (after the deploy that writes the column)
await q.query(`UPDATE "tasks" SET "priority" = 0 WHERE "priority" IS NULL`);
// Migration 3
await q.query(`ALTER TABLE "tasks" ALTER COLUMN "priority" SET NOT NULL`);
```
</details>

**23.** Code — Implement optimistic locking with a 409 on conflict.
<details><summary>Answer</summary>

```ts
@Entity() export class Doc { @VersionColumn() version: number; }

async update(id: string, dto: UpdateDocDto, expectedVersion: number): Promise<Doc> {
  const doc = await this.repo.findOneByOrFail({ id });
  if (doc.version !== expectedVersion) {
    throw new ConflictException('Document was modified by another user');
  }
  Object.assign(doc, dto);
  try {
    return await this.repo.save(doc);
  } catch (e) {
    if (e instanceof OptimisticLockVersionMismatchError) {
      throw new ConflictException('Document was modified by another user');
    }
    throw e;
  }
}
```
The explicit check gives a good message; the catch closes the race between the check and the save.
</details>

**24.** Scenario — A teammate proposes wrapping every service method in a transaction "to be safe". Evaluate it.
<details><summary>Answer</summary>It costs more than it protects. A transaction holds a connection for its whole duration, so wrapping read-only methods drains the pool under load and turns a slow external call inside a method into held database resources. Long transactions also increase lock contention and, in PostgreSQL, delay vacuum, which degrades the whole database rather than just that endpoint. Transactions belong around operations that write more than one row and must be atomic — and the boundary should be as small as the invariant requires. The correct default is no transaction for reads, one explicit transaction per multi-write operation, and no network calls to third parties inside one.</details>

**25.** Scenario — You must add a `NOT NULL` foreign key to a 50-million-row table with continuous traffic. Walk through the plan.
<details><summary>Answer</summary>Four deploys, none of which blocks. First, add the column nullable with no default and no constraint, which is metadata-only in modern PostgreSQL and instant. Second, deploy code that writes the column for every new and updated row, so the set of null rows stops growing. Third, backfill existing rows in bounded batches with a pause between them, keeping each transaction short so it never holds a long lock or bloats the write-ahead log. Fourth, add the foreign key as `NOT VALID` — which skips the full-table check and only enforces new rows — then `VALIDATE CONSTRAINT` separately, which takes a weaker lock, and finally set `NOT NULL`. The reason for the sequence is that each step is individually reversible and none holds an exclusive lock long enough to queue requests; doing it in one migration is a single statement and an outage.</details>

---

## Chapter Summary

- Migrations make schema change reviewable, testable, and reversible; `synchronize` makes it automatic and silent, which is the opposite of what a production schema needs.
- A migration runs while old code is still serving traffic, so `NOT NULL` columns, renames, and index builds must be split into steps that are each individually safe.
- Every query inside a transaction must use the callback's `manager` — an injected repository runs on another connection and survives the rollback.
- Optimistic locking costs nothing without contention and fails the second writer; pessimistic locking serialises access and can deadlock unless locks are always acquired in the same order.
- Raise the isolation level only when correctness depends on it, and be ready to retry — a conditional `UPDATE … WHERE` usually gives the same guarantee for free.
- Keep transactions small and free of third-party network calls, because they hold a pooled connection for their whole duration.

---

## Practice Exercises

1. Generate a migration after deliberately drifting your local schema, and read what it produces.
2. Write a transaction mixing `manager` and an injected repository, force a rollback, and observe what persists.
3. Reproduce a lost update with two concurrent decrements, then fix it with a conditional update.
4. Create a deliberate deadlock with two transfers in opposite directions, then fix it by sorting lock acquisition.
5. Add a `NOT NULL` column to a table with a million rows in one step, measure the lock, then do it in three steps.

---

## Mini Project: A Safe Schema Change Pipeline

Build the tooling that makes schema changes routine.

**Requirements:**

- Migrations run as a discrete deploy step with a lock, never at application boot.
- A CI check rejecting any migration containing `DROP COLUMN` or a bare `CREATE INDEX` without review approval.
- Every migration has a tested `down`, verified by a job that migrates up, down, and up again.
- A transactional service with a retry loop for serialization failures and deterministic lock ordering.
- A load test proving no oversell under 100 concurrent purchases of the last item.

---

**Next:** [Chapter 19 — Prisma, and Choosing an ORM →](19-prisma-choosing-orm.md)
