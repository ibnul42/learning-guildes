# Chapter 15 — Transactions and Isolation

## Learning Objectives

- State precisely what each ACID property guarantees, and what "consistency" in ACID does not mean
- Recognise dirty reads, read skew, lost updates, write skew, and phantoms from a description of a bug
- Choose an isolation level by the anomalies it prevents, accounting for how real databases differ from the standard
- Prevent lost updates and write skew with atomic updates, locks, version checks, constraints, or serializable isolation
- Keep transactions short, deadlock-free, and safe to retry

---

## 15.1 What a Transaction Promises

A **transaction** groups several reads and writes into one unit that either happens completely or
not at all, and that runs as if no other transaction were interfering. That promise is what lets
application code ignore most of the ways concurrent, failure-prone systems can go wrong — which is
why transactions are the single biggest reason to choose a relational database (Chapter 11).

The four ACID properties, stated precisely:

| Property | What it actually guarantees | What it doesn't |
|---|---|---|
| **Atomicity** | If the transaction fails partway — crash, constraint violation, explicit rollback — all its writes are undone. "All or nothing" is really *abortability*. | Anything about concurrency |
| **Consistency** | The application's invariants hold before and after — *if* the transactions themselves preserve them. The database helps only with what it can enforce: constraints, foreign keys, `CHECK`s. | That the data is "correct"; that's the application's job. (Unrelated to "consistency" in CAP — Chapter 20.) |
| **Isolation** | Concurrent transactions don't see each other's intermediate states; at the strongest level, the result equals *some* serial order. | That every database's default level is the strongest (they aren't — §15.3) |
| **Durability** | Once committed, the data survives crashes — via the WAL and `fsync` (Chapter 10), and replication (Chapter 13) | Survival of operator mistakes (Chapter 4's backups) |

Atomicity and durability are mostly solved problems: the WAL handles both. **Isolation** is where
the interesting trade-offs live, because full isolation is expensive, so databases offer weaker
levels by default — and the weaker levels permit specific, nameable anomalies that show up as
real bugs.

---

## 15.2 The Anomalies

Each anomaly below is a real bug pattern. Learning to recognise them from a symptom — "the stock
went negative", "two people booked the same room" — is the practical skill.

**Dirty read.** Transaction B reads a value that transaction A wrote but hasn't committed; A then
rolls back, so B acted on data that never existed. *Example:* a report includes an order that was
rolled back for a failed payment.

**Dirty write.** B overwrites a value A wrote but hasn't committed. If both then commit, the result
can mix their writes — the listing's price from one transaction and its invoice from another.
Essentially every database prevents this.

**Read skew (non-repeatable read).** A transaction reads the same data twice, or reads two related
items, and sees different points in time because another transaction committed in between.
*Example:* a transfer moves $100 from savings to checking; a balance check reads savings before the
transfer and checking after it, and reports $100 missing.

**Lost update.** Two transactions read the same value, each modifies it, and each writes it back;
the second write silently overwrites the first. *Example:* two concurrent "add a tag" requests each
read the tag list, append their tag, and save — one tag disappears. This is the read-modify-write
cycle, and it's everywhere in application code.

**Write skew.** Two transactions read an overlapping set of data, make a decision based on it, and
write *different* records — so neither overwrites the other, but together they violate an
invariant. *Example:* a rule says at least one doctor must be on call. Alice and Bob are both on
call; each checks "is someone else on call?", sees yes, and goes off call. Both commit. Nobody is on
call. Other examples: two users claim the same username (each checked it was free), two bookings
overlap the same room-hour, two withdrawals each check the shared balance.

**Phantom.** A transaction's query matches a set of rows by condition; another transaction inserts a
row that would have matched. Phantoms are what make write skew hard to prevent with locks: in the
username and booking examples, the conflicting row *doesn't exist yet*, so there's nothing to lock.

| Anomaly | Two transactions… | Typical symptom |
|---|---|---|
| Dirty read | One reads the other's uncommitted write | Acting on data that was rolled back |
| Read skew | One sees a mix of before and after another's commit | Totals that don't add up in a report or backup |
| Lost update | Both read-modify-write the same value | Increments, appends, or edits disappear |
| Write skew | Both read the same facts, then write different rows | An invariant spanning rows is violated |
| Phantom | One's query result changes because the other inserted | Duplicate bookings, usernames, or reservations |

---

## 15.3 Isolation Levels — Standard and Reality

The SQL standard defines four isolation levels by which anomalies they permit. Real databases
implement them differently, and the names alone are not reliable:

| Level | Dirty read | Read skew | Lost update | Write skew / phantoms | Default in |
|---|---|---|---|---|---|
| Read uncommitted | Possible (PostgreSQL treats it as read committed) | Possible | Possible | Possible | — |
| **Read committed** | Prevented | Possible | Possible | Possible | **PostgreSQL**, Oracle, SQL Server |
| **Repeatable read / snapshot isolation** | Prevented | Prevented | Prevented in PostgreSQL (aborts one); **possible in MySQL** for read-then-write | **Possible** | **MySQL (InnoDB)** |
| **Serializable** | Prevented | Prevented | Prevented | Prevented | CockroachDB, FoundationDB, Spanner (strict) |

Three facts about this table cause most real-world confusion:

1. **The common defaults are weak.** PostgreSQL's default is read committed: every statement sees a
   fresh snapshot, so read skew, lost updates, and write skew are all possible. Most application code
   is written as if transactions were serializable, and isn't.
2. **"Repeatable read" means different things.** In PostgreSQL it's **snapshot isolation**: the whole
   transaction reads from one snapshot, and if two transactions update the same row, the second one
   to commit is aborted with a serialization error. In MySQL/InnoDB, plain `SELECT`s read a snapshot
   but `UPDATE`s and locking reads see the latest committed data, so a read-then-write lost update can
   slip through.
3. **Snapshot isolation permits write skew.** It prevents conflicts on the *same* rows, not decisions
   based on rows another transaction is changing. Oracle's level named "serializable" is in fact
   snapshot isolation.

The difference between snapshot isolation and serializability is concrete enough to run. This
miniature multi-version store implements both and replays the lost-update and doctors-on-call
scenarios from §15.2:

```js
// mvcc.mjs — a tiny multi-version store: snapshot reads, first-committer-wins on
// write-write conflicts, and optional serializable validation of the read set.
class MvccStore {
  #versions = new Map(); // key → [{ value, committedAt }]
  #clock = 0;

  begin({ serializable = false } = {}) {
    const snapshot = this.#clock, reads = new Set(), writes = new Map();
    const visible = (key) => {
      const vs = this.#versions.get(key) ?? [];
      // The newest version committed at or before our snapshot — later commits are invisible
      for (let i = vs.length - 1; i >= 0; i--) if (vs[i].committedAt <= snapshot) return vs[i].value;
      return undefined;
    };
    const changedSince = (key) => (this.#versions.get(key) ?? []).some((v) => v.committedAt > snapshot);
    return {
      get: (key) => { reads.add(key); return writes.has(key) ? writes.get(key) : visible(key); },
      set: (key, value) => writes.set(key, value),
      commit: () => {
        // Snapshot isolation: abort if someone committed a newer version of a key we WRITE
        for (const key of writes.keys()) if (changedSince(key)) throw new Error(`write conflict on ${key}`);
        // Serializable (optimistic): also abort if anything we READ changed since our snapshot
        if (serializable) for (const key of reads) if (changedSince(key)) throw new Error(`read ${key} is stale`);
        const at = ++this.#clock;
        for (const [key, value] of writes) {
          if (!this.#versions.has(key)) this.#versions.set(key, []);
          this.#versions.get(key).push({ value, committedAt: at });
        }
      },
    };
  }
  seed(key, value) { this.#versions.set(key, [{ value, committedAt: 0 }]); }
}

const attempt = (fn) => { try { fn(); return 'committed'; } catch (e) { return `aborted (${e.message})`; } };

// 1. Lost update: two transactions both read a counter and write +1
const db = new MvccStore(); db.seed('likes', 10);
const t1 = db.begin(), t2 = db.begin();
t1.set('likes', t1.get('likes') + 1);
t2.set('likes', t2.get('likes') + 1);
console.log('lost update  t1:', attempt(t1.commit), '| t2:', attempt(t2.commit));
// lost update  t1: committed | t2: aborted (write conflict on likes)

// 2. Write skew: "at least one doctor must stay on call"; each doctor checks, then leaves
const onCallScenario = (serializable) => {
  const s = new MvccStore(); s.seed('alice', 'on'); s.seed('bob', 'on');
  const a = s.begin({ serializable }), b = s.begin({ serializable });
  const othersOnCall = (tx, me) => ['alice', 'bob'].filter((d) => d !== me && tx.get(d) === 'on').length;
  if (othersOnCall(a, 'alice') >= 1) a.set('alice', 'off');
  if (othersOnCall(b, 'bob') >= 1) b.set('bob', 'off');
  const ra = attempt(a.commit), rb = attempt(b.commit);
  const check = s.begin();
  return `alice: ${ra} | bob: ${rb} | on call now: ${['alice', 'bob'].filter((d) => check.get(d) === 'on').length}`;
};
console.log('write skew, snapshot isolation:', onCallScenario(false));
console.log('write skew, serializable:      ', onCallScenario(true));
// write skew, snapshot isolation: alice: committed | bob: committed | on call now: 0
// write skew, serializable:       alice: committed | bob: aborted (read alice is stale) | on call now: 1
```

Under snapshot isolation, the lost update is caught — both transactions wrote `likes` — but write
skew isn't: Alice and Bob wrote *different* keys, so there's no write conflict to detect, and the
invariant breaks. Serializable validation also checks the *read* set: Bob's decision depended on
reading Alice, whom a committed transaction changed, so Bob aborts. That's the essence of how
serializable isolation catches write skew — and why serializable transactions abort more and must
be retried.

> 💼 **Interview** — "What isolation level does your database use by default, and what can go wrong
> at that level?" is a strong senior question. For PostgreSQL: read committed; lost updates and write
> skew are possible unless the code uses atomic updates, `FOR UPDATE`, constraints, or a stronger
> level. Knowing that "repeatable read" differs between PostgreSQL and MySQL is a bonus signal.

---

## 15.4 How Databases Implement Isolation

Three mechanisms, often combined:

**Multi-version concurrency control (MVCC).** The database keeps several versions of each row,
tagged with the transaction that created them. Each transaction reads from a **snapshot** — the
versions committed before it started (or, under read committed, before each statement started).
Readers never block writers and writers never block readers, which is why MVCC is the basis of
PostgreSQL, MySQL/InnoDB, Oracle, and most modern databases. The cost is old versions: they
accumulate until no running transaction can see them, and must be cleaned up (PostgreSQL's
`VACUUM`, InnoDB's purge of undo logs). A transaction left open for hours prevents that cleanup
across the whole database.

**Two-phase locking (2PL).** Transactions take shared locks to read and exclusive locks to write, and
hold them until commit. Readers block writers and writers block readers, which provides
serializability (with predicate or range locks for phantoms) at a large cost in concurrency and in
**deadlocks** — two transactions each waiting for a lock the other holds, broken by the database
aborting one. Pure 2PL is the classic implementation of serializable isolation; SQL Server and
MySQL use it for their serializable levels.

**Serializable snapshot isolation (SSI).** Run on MVCC snapshots optimistically, track which
transactions read data that others wrote (read–write dependencies), and abort a transaction when
those dependencies form a pattern that could make the outcome non-serial — the check §15.3's
simulation does crudely. PostgreSQL's `SERIALIZABLE` level and CockroachDB use variants of it. It
keeps MVCC's concurrency for readers and adds aborts only where conflicts actually occur.

| | MVCC + snapshot isolation | 2PL (serializable) | SSI (serializable) |
|---|---|---|---|
| Readers block writers | No | Yes | No |
| Prevents write skew | No | Yes | Yes |
| Failure mode under contention | Write-conflict aborts | Lock waits, deadlocks | Serialization-failure aborts |
| Application must | Handle write conflicts | Handle deadlock aborts | Retry aborted transactions |

All three can abort transactions that did nothing wrong. **Retrying aborted transactions is not an
edge case to handle later; it's part of using isolation correctly**, and the real-world example in
§15.9 is the wrapper that does it.

---

## 15.5 Preventing Lost Updates and Write Skew in Practice

Serializable isolation prevents every anomaly but costs throughput and retries. Most systems run at
read committed or snapshot isolation and protect specific invariants explicitly. The tools, roughly
from cheapest to most general:

**Atomic updates.** Let the database do the read-modify-write in one statement, under its own row
lock:

```sql
-- ❌ Wrong — read, compute in the application, write back: a lost update under read committed
SELECT stock FROM products WHERE id = 42;          -- both transactions read 5
UPDATE products SET stock = 4 WHERE id = 42;       -- both write 4; one sale vanishes

-- ✅ Right — one atomic statement; the condition also enforces "never below zero"
UPDATE products SET stock = stock - 1 WHERE id = 42 AND stock >= 1;
-- rows affected = 0 → out of stock; tell the user
```

**Explicit locking.** When the logic can't be one statement, lock the rows you'll base the decision
on:

```sql
BEGIN;
SELECT balance FROM accounts WHERE id IN (7, 9) ORDER BY id FOR UPDATE; -- lock in a fixed order
-- application checks the balance, computes fees, etc.
UPDATE accounts SET balance = balance - 30 WHERE id = 7;
UPDATE accounts SET balance = balance + 30 WHERE id = 9;
COMMIT;
```

`FOR UPDATE` blocks other transactions from locking or modifying those rows until commit, so the
read and the write can't interleave with anyone else's. It fixes lost updates and write skew *on rows
that exist* — the doctors' rows, the account rows.

**Optimistic version checks.** Read a version number with the row; write only if it hasn't changed;
retry or report a conflict if it has (Chapter 8's `If-Match` is the HTTP face of this):

```sql
UPDATE documents SET body = $1, version = version + 1 WHERE id = $2 AND version = $3;
-- rows affected = 0 → someone else saved first: re-read and retry, or show a conflict
```

No locks are held between the read and the write, which suits long user "think time" — editing a
document for ten minutes — where holding a lock would be unacceptable.

**Constraints for phantoms.** When the conflicting row doesn't exist yet, there's nothing to lock — so
let the database refuse the second insert:

```sql
-- Two users can't claim the same username, no matter how they race
CREATE UNIQUE INDEX users_username ON users (lower(username));

-- Two bookings can't overlap the same room (PostgreSQL exclusion constraint)
CREATE EXTENSION IF NOT EXISTS btree_gist;
ALTER TABLE bookings ADD CONSTRAINT no_overlap
  EXCLUDE USING gist (room_id WITH =, tstzrange(starts_at, ends_at) WITH &&);
```

Constraints are the most robust tool on this list: they hold regardless of isolation level,
application bugs, or a new code path someone adds next year.

**Materialising the conflict.** When no constraint expresses the invariant, create rows that *do*
exist and lock them. For meeting-room bookings without exclusion constraints, pre-create one row per
room per time slot and `SELECT … FOR UPDATE` the slots being booked — turning a phantom into an
ordinary lock conflict. It's a workaround with a cost, to be used when nothing above fits.

**Serializable isolation, with retries.** Run the transactions that protect a complex invariant at
`SERIALIZABLE` and let the database find the conflicts. It's often simpler and safer than reasoning
about which rows to lock — provided every caller retries on serialization failure.

| Problem | First choice | When it doesn't fit |
|---|---|---|
| Counter / stock decrement | Atomic `UPDATE … WHERE stock >= 1` | Logic too complex for one statement → `FOR UPDATE` |
| Edit of a record with user think time | Optimistic version check | — |
| Uniqueness of a value | Unique constraint | Uniqueness across shards → lookup table (Chapter 14) |
| Non-overlapping ranges | Exclusion constraint | Store lacks them → materialise slots |
| Invariant over several existing rows | `FOR UPDATE` in a consistent order | Many rows or complex predicates → serializable |
| Complex multi-table invariant | Serializable + retry | Very high contention → redesign to reduce it |

---

## 15.6 Locks, Deadlocks, and Long Transactions

**Deadlocks** arise when transactions lock the same resources in different orders: T1 locks account 7
then wants 9; T2 locks 9 then wants 7. The database detects the cycle and aborts one. Prevention is
mostly discipline:

- **Lock in a consistent order** — by primary key, as the `ORDER BY id FOR UPDATE` above does.
- **Lock everything you need up front**, in one statement, rather than incrementally.
- **Keep transactions short**, so locks are held briefly and cycles are rare.
- **Treat deadlock aborts as retryable** (`40P01` in PostgreSQL); they're normal under contention.

**Long transactions** cause more production incidents than deadlocks:

- They **hold locks**, so everything that needs those rows queues behind them — and with Chapter 3's
  queueing curves, a few seconds of lock hold at high traffic becomes a pile-up.
- They **pin old MVCC versions**, so vacuum or purge can't reclaim space anywhere in the database;
  tables bloat and queries slow down.
- In PostgreSQL, a long transaction on a replica can conflict with replication and be cancelled, or
  delay the replica.

The worst version is a transaction **held open across a network call** — opening a transaction,
calling a payment provider or another service, then committing. Any slowness in the remote call
becomes lock-hold time in your database. Do remote calls before the transaction (gathering what you
need) or after it (with an outbox, Chapter 23), never inside. And set guardrails in the database:

```sql
-- PostgreSQL: cap the damage from forgotten or stuck transactions
ALTER ROLE app SET idle_in_transaction_session_timeout = '30s'; -- kill sessions idle inside a transaction
ALTER ROLE app SET lock_timeout = '5s';                         -- fail fast instead of queueing forever
ALTER ROLE app SET statement_timeout = '15s';                   -- no single statement runs away
```

> ⚠️ **Gotcha** — **Schema migrations take locks too.** An `ALTER TABLE` that needs an exclusive lock
> waits behind any long-running transaction on that table — and while it waits, *every new query on
> the table queues behind the migration*. A migration that would take milliseconds can freeze the
> table for as long as the oldest transaction lasts. Always run DDL with a short `lock_timeout` and
> retry it, rather than letting it wait (Chapter 31).

---

## 15.7 Transactions Beyond One Database

Everything in this chapter assumes one database. Once an operation spans two databases, a database
and a message broker, or two services, there's no shared transaction log to make it atomic, and the
techniques change: two-phase commit, sagas, the transactional outbox, and idempotent consumers
(Chapter 23). Two consequences are worth stating now, because they shape data modelling:

- **Keep invariants inside one transactional boundary when you can.** If "an order and its inventory
  reservation are created together" is an invariant, the order and inventory tables should live in
  the same database (or the same partition). Splitting them across services turns one `BEGIN … COMMIT`
  into a distributed protocol with compensations.
- **The most common distributed-transaction bug is a local one:** writing to the database and
  publishing an event (or calling an API) as two separate steps, as Chapter 11's dual-write example
  showed. The fix is the outbox — making the event part of the local transaction.

---

## 15.8 Common Mistakes and Best Practices

**Mistake: assuming the default isolation level is serializable.** At read committed — PostgreSQL's
default — lost updates and write skew are possible in ordinary read-then-write code.

**Mistake: read-modify-write in application code.** Reading a value, computing in the app, and writing
it back loses concurrent updates; use an atomic `UPDATE` or a lock.

**Mistake: guarding uniqueness or non-overlap with a `SELECT` check.** Two transactions both see "free"
and both insert; only constraints (or serializable isolation) stop phantoms.

**Mistake: serializable isolation without retries.** Serialization failures are normal, and code that
surfaces them as errors turns correct isolation into user-facing failures.

**Mistake: side effects inside retried transactions.** Emails, HTTP calls, and queue publishes inside
the transaction body run again on every retry — and run even when the transaction later aborts.

**Mistake: holding a transaction open across a network call.** Remote latency becomes lock-hold time,
and a slow dependency freezes rows for everyone.

**Mistake: inconsistent lock ordering.** Transactions that lock the same rows in different orders
deadlock under load.

**Best practice: push invariants into constraints.** Unique, foreign-key, check, and exclusion
constraints hold regardless of isolation level or application bugs.

**Best practice: one retry wrapper for all transactional work.** Retry serialization failures and
deadlocks with jittered backoff, and keep side effects outside it.

**Best practice: database-level timeouts.** `lock_timeout`, `statement_timeout`, and idle-in-transaction
timeouts turn stuck transactions into fast, visible failures.

---

## 15.9 Real-World Example: A Transaction Retry Wrapper

Every application using snapshot or serializable isolation needs this, and most write it after an
incident. It runs a unit of work in a transaction at a chosen isolation level, retries only the errors
that mean "conflict — try again", backs off with jitter, and documents the rule that makes retrying
safe. The demo uses a fake connection pool whose first two commits fail with serialization errors.

```js
// with-transaction.mjs — run a unit of work in a transaction, retrying the conflicts that
// serializable/repeatable-read isolation and deadlock detection are designed to produce.
// Works with any client exposing query(sql, params) — node-postgres, for example.

// PostgreSQL SQLSTATEs that mean "nothing was wrong with your transaction; run it again"
const RETRYABLE = new Set([
  '40001', // serialization_failure — SSI, or a write conflict under REPEATABLE READ
  '40P01', // deadlock_detected — the database aborted one of the participants
]);

export async function withTransaction(pool, work, {
  isolation = 'SERIALIZABLE', maxAttempts = 5, baseDelayMs = 10, onRetry = () => {},
  rand = Math.random, sleep = (ms) => new Promise((r) => setTimeout(r, ms)),
} = {}) {
  for (let attempt = 1; ; attempt++) {
    const client = await pool.connect();
    try {
      await client.query(`BEGIN ISOLATION LEVEL ${isolation}`);
      // `work` must be safe to run more than once: no emails, HTTP calls, or queue publishes
      // inside it. Collect side effects and perform them after commit (or use an outbox, Ch 23).
      const result = await work(client);
      await client.query('COMMIT');   // serialization failures can surface here, too
      return result;
    } catch (err) {
      await client.query('ROLLBACK').catch(() => {}); // the connection may already be broken
      if (!RETRYABLE.has(err.code) || attempt >= maxAttempts) throw err;
      // Full jitter: contending transactions that retry in lockstep just conflict again
      const delay = Math.floor(rand() * baseDelayMs * 2 ** attempt);
      onRetry({ attempt, code: err.code, delay });
      await sleep(delay);
    } finally {
      client.release();
    }
  }
}

// ── Demo with a fake pool whose first two COMMITs fail with serialization errors ─────
let commits = 0;
const fakePool = {
  async connect() {
    return {
      async query(sql) {
        if (sql === 'COMMIT' && ++commits <= 2) throw Object.assign(new Error('could not serialize'), { code: '40001' });
        if (sql.startsWith('SELECT')) return { rows: [{ balance: 100 }] };
        return { rows: [] };
      },
      release() {},
    };
  },
};

const result = await withTransaction(fakePool, async (tx) => {
  const { rows } = await tx.query('SELECT balance FROM accounts WHERE id = $1 FOR UPDATE', [7]);
  await tx.query('UPDATE accounts SET balance = $1 WHERE id = $2', [rows[0].balance - 30, 7]);
  return rows[0].balance - 30;
}, { onRetry: ({ attempt, code }) => console.log(`retry after attempt ${attempt} (${code})`), sleep: async () => {} });
console.log('new balance:', result, '| commits attempted:', commits);
// retry after attempt 1 (40001)
// retry after attempt 2 (40001)
// new balance: 70 | commits attempted: 3
```

The decision to defend is **retrying the entire unit of work — re-running every query from `BEGIN` —
rather than just re-issuing the failed statement or the `COMMIT`.** Re-sending only the commit looks
cheaper, but a serialization failure means the database has determined that the transaction's reads may
be inconsistent with what other transactions committed: the balance it read, the doctors it saw on call,
the stock level it checked. Every decision the transaction made was based on that possibly stale view,
so the only correct retry re-reads everything and re-decides. That's also why `work` must be a function
the wrapper can call repeatedly, and why side effects are banned inside it — a retried function that
sent an email has sent two.

---

## 15.10 Questions

**1.** MCQ — What is PostgreSQL's default isolation level? a) Read uncommitted b) Read committed c) Repeatable read d) Serializable
<details><summary>Answer</summary>**b)** Each statement sees a fresh snapshot of committed data; lost updates and write skew are possible.</details>

**2.** MCQ — Two transactions each check that a username is free, then insert it; both succeed. Which anomaly is this? a) Dirty read b) Lost update c) Write skew via a phantom d) Read skew
<details><summary>Answer</summary>**c)** Both read the same (empty) result and wrote rows that conflict with each other's decision; the conflicting row didn't exist when they checked.</details>

**3.** MCQ — Under snapshot isolation, which anomaly is still possible? a) Dirty reads b) Read skew c) Lost update on the same row (in PostgreSQL) d) Write skew
<details><summary>Answer</summary>**d)** Snapshot isolation detects conflicting writes to the same rows, not decisions based on rows another transaction changed.</details>

**4.** MCQ — Which technique prevents double-booking of overlapping time ranges regardless of isolation level? a) `SELECT` to check for overlaps first b) An exclusion constraint on room and time range c) Read committed d) A larger connection pool
<details><summary>Answer</summary>**b)** The constraint makes the second conflicting insert fail, whatever the isolation level or application code does.</details>

**5.** MCQ — What does `SELECT … FOR UPDATE` protect against? a) Phantoms from rows that don't exist yet b) Concurrent modification of the selected rows until commit c) Dirty reads only d) Deadlocks
<details><summary>Answer</summary>**b)** It locks existing rows; it can't lock rows that haven't been inserted, and it can contribute to deadlocks if ordering is inconsistent.</details>

**6.** MCQ — Why do long-running transactions degrade an MVCC database even if they hold no locks? a) They use more CPU b) They pin old row versions so vacuum/purge can't reclaim them, causing bloat c) They disable indexes d) They block replication entirely
<details><summary>Answer</summary>**b)** Old versions must be kept while any running transaction might need them.</details>

**7.** Short — Why is atomicity better described as "abortability"?
<details><summary>Answer</summary>Its guarantee is that a transaction that fails partway can be rolled back with all its writes undone, so the application can safely retry or give up. It says nothing about concurrent transactions — that's isolation.</details>

**8.** Short — How does "repeatable read" differ between PostgreSQL and MySQL/InnoDB?
<details><summary>Answer</summary>PostgreSQL's is snapshot isolation: the whole transaction reads one snapshot, and concurrent updates to the same row abort the later committer. In InnoDB, plain reads use a snapshot but updates and locking reads see the latest committed data, so a read-then-write lost update can go undetected.</details>

**9.** Short — When is an optimistic version check better than `SELECT … FOR UPDATE`?
<details><summary>Answer</summary>When there's long "think time" between reading and writing — a user editing a form — because no lock is held in between. Conflicts are detected at write time and resolved by re-reading.</details>

**10.** Short — What does "materialising the conflict" mean?
<details><summary>Answer</summary>Creating rows that represent the contested resource (e.g. one row per room per time slot) so that transactions that would conflict lock the same existing rows, turning a phantom-based write skew into an ordinary lock conflict.</details>

**11.** Short — Why must side effects be kept out of a retried transaction body?
<details><summary>Answer</summary>The body may run several times (once per retry), and each run's side effects happen even if that attempt aborts — so emails, API calls, and publishes would be duplicated or occur for transactions that never committed.</details>

**12.** Output — Using §15.3's `MvccStore`, a store seeded with `x = 1` starts transaction A, then transaction B; B sets `x = 2` and commits; A reads `x`. What does A see?
<details><summary>Answer</summary>**1.** A's snapshot was taken before B committed, so B's version (committed later) is invisible to A.</details>

**13.** Output — Same store: A and B both start; A sets `x = 5` and commits; B sets `y = 7` (never touching `x`) and commits, without serializable mode. What are the outcomes?
<details><summary>Answer</summary>**Both commit.** B wrote a different key, so there's no write conflict; without serializable validation its reads aren't checked.</details>

**14.** Output — Stock is 1. Two transactions run `UPDATE products SET stock = stock - 1 WHERE id = 42 AND stock >= 1` concurrently under read committed. How many rows does each affect, and what is the final stock?
<details><summary>Answer</summary>**One affects 1 row, the other 0; final stock 0.** The second waits for the first's row lock, then re-evaluates the condition against the committed value (0) and matches nothing.</details>

**15.** Output — What does this print?
```js
const delay = (attempt, base, r) => Math.floor(r * base * 2 ** attempt);
console.log(delay(1, 10, 0.5), delay(3, 10, 0.5), delay(5, 10, 0.99));
```
<details><summary>Answer</summary>`10 40 316` — full-jitter backoff: a random fraction of an exponentially growing ceiling.</details>

**16.** Debug — An inventory service occasionally sells more units than it has. The code reads `stock`, checks `stock >= qty`, and writes `stock - qty` in a transaction at read committed. Cause and fix?
<details><summary>Answer</summary>A lost update: concurrent transactions read the same stock, both pass the check, and both write. Use a single atomic `UPDATE … SET stock = stock - $qty WHERE id = $id AND stock >= $qty` and check the affected row count — or lock with `FOR UPDATE`.</details>

**17.** Debug — After switching a booking service to `SERIALIZABLE` to fix double-bookings, the error rate jumps to 3% during peak, all "could not serialize access". What's missing?
<details><summary>Answer</summary>Retries. Serialization failures are the mechanism working — the database aborted transactions that would have produced anomalies. Wrap the transactional work in a retry loop with jittered backoff (and consider an exclusion constraint, which avoids most aborts for this specific invariant).</details>

**18.** Debug — Every few days, all writes to the `orders` table hang for several minutes. The trigger is a migration adding a column; the slow-query log shows a report query that runs for 20 minutes. Explain the chain and the fix.
<details><summary>Answer</summary>The `ALTER TABLE` needs an exclusive lock and waits behind the long report transaction; while it waits, every new query on `orders` queues behind the migration's pending lock. Run DDL with a short `lock_timeout` and retry, move long reports to a replica or warehouse, and set `statement_timeout` for reporting roles.</details>

**19.** Debug — Database bloat grows steadily and vacuum seems ineffective; `pg_stat_activity` shows a connection "idle in transaction" for two days from a batch tool. What's happening and how do you prevent it?
<details><summary>Answer</summary>The open transaction pins old row versions, so vacuum can't remove anything newer than its snapshot anywhere. Terminate it, fix the tool to commit, and set `idle_in_transaction_session_timeout` so abandoned transactions are killed automatically.</details>

**20.** Code — Write `transfer(client, from, to, amount)` SQL logic (as JS calling `client.query`) that locks both accounts in a consistent order, checks the balance, and moves the money.
<details><summary>Answer</summary>

```js
async function transfer(client, from, to, amount) {
  const [first, second] = from < to ? [from, to] : [to, from];       // consistent order: no deadlocks
  const { rows } = await client.query(
    'SELECT id, balance FROM accounts WHERE id IN ($1, $2) ORDER BY id FOR UPDATE', [first, second]);
  const source = rows.find((r) => r.id === from);
  if (!source || source.balance < amount) throw Object.assign(new Error('insufficient funds'), { code: 'NSF' });
  await client.query('UPDATE accounts SET balance = balance - $1 WHERE id = $2', [amount, from]);
  await client.query('UPDATE accounts SET balance = balance + $1 WHERE id = $2', [amount, to]);
}
// Called inside withTransaction(pool, (tx) => transfer(tx, 9, 7, 30))
```
</details>

**21.** Code — Write `saveWithVersion(client, { id, body, expectedVersion })` that performs an optimistic update and throws a `ConflictError` when the version has changed.
<details><summary>Answer</summary>

```js
class ConflictError extends Error {}
async function saveWithVersion(client, { id, body, expectedVersion }) {
  const { rowCount } = await client.query(
    'UPDATE documents SET body = $1, version = version + 1 WHERE id = $2 AND version = $3',
    [body, id, expectedVersion]);
  if (rowCount === 0) throw new ConflictError(`document ${id} changed since version ${expectedVersion}`);
  return expectedVersion + 1;
}
```
</details>

**22.** Code — Write `afterCommit(work)` helper semantics: a function that runs `work(tx, defer)` where `defer(fn)` queues side effects, and the queued functions run only once the transaction has committed successfully (using §15.9's `withTransaction`).
<details><summary>Answer</summary>

```js
async function runWithDeferred(pool, work, opts) {
  let deferred = [];
  const result = await withTransaction(pool, async (tx) => {
    deferred = [];                                  // reset on every attempt: aborted attempts' effects are dropped
    return work(tx, (fn) => deferred.push(fn));
  }, opts);
  for (const fn of deferred) await fn();            // only the committed attempt's side effects run
  return result;
}
```
Effects run after commit but aren't atomic with it — a crash in between loses them, which is why critical ones belong in an outbox (Chapter 23).</details>

**23.** Scenario — A ticketing system must never sell more seats than a venue section holds. Sales run at 2,000/s during on-sale. The team proposes `SERIALIZABLE` for the purchase transaction. What do you recommend?
<details><summary>Answer</summary>At that contention, serializable on a hot section would abort constantly. Better: an atomic conditional decrement on a per-section counter (`UPDATE sections SET remaining = remaining - $n WHERE id = $id AND remaining >= $n`), or seat-level rows claimed with `UPDATE … WHERE status = 'free'` — both are single-row atomic operations that never oversell. Shard hot counters if one row's lock throughput is the limit, and front the on-sale with a queue (Chapter 47). Keep serializable for low-contention admin workflows.</details>

**24.** Scenario — A team's monthly-statement job reads account balances across a million accounts in one long read-committed transaction while transfers continue. Totals occasionally don't match the ledger. What's going on, and what do you change?
<details><summary>Answer</summary>Read skew: under read committed each statement sees a different snapshot, so balances read early and late straddle concurrent transfers. Run the job at repeatable read (one snapshot for the whole read-only transaction), ideally on a replica so the long snapshot doesn't bloat the primary — or read from a consistent backup or point-in-time snapshot. Better still, compute statements from the immutable ledger entries up to a cut-off time.</details>

---

## Chapter Summary

- ACID's atomicity means abortability, consistency is the application's invariants, isolation is the
  hard part, and durability comes from the WAL and replication.
- The anomalies — dirty reads, read skew, lost updates, write skew, phantoms — are concrete bug
  patterns; learn to recognise them from symptoms like negative stock and double bookings.
- Default isolation levels are weak (PostgreSQL uses read committed), "repeatable read" differs by
  database, and snapshot isolation still permits write skew.
- MVCC lets readers and writers proceed without blocking; 2PL and SSI add serializability at the cost
  of lock waits or aborts — and every isolation mechanism requires retrying aborted transactions.
- Protect invariants with the cheapest sufficient tool: atomic updates, `FOR UPDATE` in a fixed order,
  version checks, unique and exclusion constraints, or serializable isolation with retries.
- Keep transactions short, never span network calls, keep side effects out of retried bodies, and use
  database timeouts so stuck transactions fail fast.

---

## Practice Exercises

1. In PostgreSQL, open two `psql` sessions and reproduce a lost update at read committed, then show
   that repeatable read aborts one of them.
2. Reproduce the doctors-on-call write skew at repeatable read in PostgreSQL, then show that
   `SERIALIZABLE` aborts one transaction.
3. Read your database's documentation on its isolation levels and write a one-page table of which
   anomalies each permits *in that database*.
4. Find a read-modify-write pattern in a codebase you know and rewrite it as an atomic update or a
   version-checked update; write a concurrent test that fails before and passes after.
5. Add an exclusion constraint to a booking-style table, then write a load test with 100 concurrent
   overlapping bookings and confirm exactly one succeeds per conflicting set.

---

## Mini Project: An Isolation Anomaly Lab

Build a test harness that runs pairs of transactions against a real PostgreSQL instance with
controlled interleavings, so each anomaly in this chapter can be reproduced on demand and each fix
verified.

- A scheduler that runs two transactions step by step in a specified interleaving (using separate
  connections and barriers).
- Scenarios for dirty read (shown to be impossible), read skew, lost update, write skew, and phantom
  insert, each run at read committed, repeatable read, and serializable.
- A result matrix showing which anomalies occur at which level.
- For each anomaly, a fixed version (atomic update, `FOR UPDATE`, version check, constraint, or
  serializable with retry) and a test proving the fix.
- A short report comparing throughput and abort rates of the fixes under 50 concurrent clients.

---

**Next:** [Chapter 16 — Caching →](16-caching.md)
