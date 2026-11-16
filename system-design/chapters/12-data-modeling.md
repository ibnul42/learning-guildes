# Chapter 12 — Data Modeling for Access Patterns

## Learning Objectives

- Decide when to normalise and when to denormalise, and what each costs on reads and writes
- Design partition and sort keys from a list of access patterns so each pattern is one query
- Maintain precomputed views, counters, and aggregates without making them a second source of truth
- Detect hot keys and remove them with sharding, caching, or write coalescing
- Choose a multi-tenancy data model and evolve schemas without downtime

---

## 12.1 Normalise First, Denormalise on Purpose

**Normalisation** stores each fact exactly once: a customer's email lives in the `customers` table,
and orders reference the customer by ID. Updates touch one place, so there's no way for two copies
to disagree. Reads that need data from several tables pay for it with **joins**.

**Denormalisation** copies data to where it's read: an order row stores the customer's name and
shipping address at the time of purchase; a post stores its author's display name; a product row
stores its review count. Reads get cheaper — no join, often one row. Writes get more expensive and
riskier: every copy must be updated, and the copies can drift.

| | Normalised | Denormalised |
|---|---|---|
| Each fact stored | Once | Once per place it's read |
| Reads | Joins across tables | Often a single row or partition |
| Writes | One place | Every copy — more work, more failure modes |
| Consistency risk | None between copies | Copies drift if any update path is missed |
| Fits | Relational stores, evolving queries, write-heavy data | Read-heavy paths, stores without joins, extreme scale |

Two distinctions keep denormalisation safe:

- **Snapshots are not copies.** An order's shipping address *should* be the address at purchase
  time, frozen; it's a different fact from "the customer's current address" and never needs
  updating. Many apparent denormalisations are really snapshots, and they're free.
- **Derived copies need an owner.** A review count stored on a product row is derived from the
  `reviews` table. It must be maintained by one mechanism — a trigger, a transactional update, or
  an event consumer (Chapter 11's CDC) — with a way to recompute it from the source if it drifts.

The default in a relational store is **normalise, then denormalise the specific paths that
measurements show are too slow**. In a store without joins (key-value, wide-column), denormalisation
is the starting point, and the discipline shifts to designing keys so each access pattern is one
query.

---

## 12.2 Keys From Access Patterns

In partitioned key-value and wide-column stores (Chapter 11), a query is efficient only if it
targets **one partition key** and, optionally, a **range of sort keys** within it. Modelling is
therefore the work of choosing keys so that every access pattern has that shape. The method:

1. **List the access patterns** with rates — exactly as Chapter 11 required.
2. **For each pattern, identify the lookup value** (what the caller knows: a customer ID, an order
   ID, a seller ID) — that becomes a partition key.
3. **Identify the order and range** the pattern needs (newest first, a date range, a status) —
   that becomes the sort key, often a composite string like `ORDER#2026-11-03#o2`.
4. **Group items that are read together into the same partition** — an "item collection" — so one
   query returns all of them.
5. **For patterns that need a different partition key** over the same data, add a secondary index
   (another copy keyed differently) or write a second item.

The real-world example in §12.9 runs this method end to end for an e-commerce domain. The patterns
that fall out of it are worth naming:

| Pattern | Key shape | Example |
|---|---|---|
| Entity by ID | `PK = TYPE#id`, `SK = META` | `ORDER#o2 / META` |
| One-to-many, sorted | `PK = parent`, `SK = CHILDTYPE#sortValue#id` | `CUST#c1 / ORDER#2026-11-03#o2` |
| Parent and children in one read | Same PK, different SK prefixes | `CUST#c1 / PROFILE` and `CUST#c1 / ORDER#…` |
| Lookup by an alternate attribute | Secondary index or a lookup item | `EMAIL#ana@example.com → CUST#c1` |
| Filtered subset ("open orders") | Sparse index: only items with the attribute appear | `SELLER#s9#OPEN / 2026-10-01#o1` |
| Time-bucketed series | `PK = entity#bucket`, `SK = timestamp` | `DEVICE#d7#2026-11-15 / 10:42:07.120` |

> ⚠️ **Gotcha** — **Sort keys compare as strings.** `ORDER#2026-11-03` sorts correctly because ISO
> dates sort lexicographically; `ORDER#3/11/2026` does not. Numbers need zero-padding (`ITEM#002`,
> not `ITEM#2`, or `ITEM#10` sorts before `ITEM#2`). Every component of a composite sort key must be
> formatted so that string order equals the intended order.

**Uniqueness constraints** need special care in stores without them. "No two customers share an
email" becomes a separate item keyed `EMAIL#<address>` written in the same transaction (or with a
conditional put) as the customer — the write fails if the email item already exists. That's the
pattern relational databases give you with one `UNIQUE` index, and it's a good reminder of what
you're trading away.

> 💼 **Interview** — When you choose a partition key in a design interview, say what it optimises
> *and* what it costs: "Partitioning messages by conversation ID makes 'load a conversation' a
> single-partition range read, but 'all messages from user X across conversations' becomes a
> scatter-gather — so I'd maintain a per-user index if we need it."

---

## 12.3 Precomputation: Views, Counters, and Aggregates

The fastest way to answer an expensive query is to have answered it already. **Precomputation**
moves work from read time to write time (or to the background):

| Technique | What's precomputed | Maintained by | Staleness |
|---|---|---|---|
| Denormalised column | A value from another table (author name on a post) | Same transaction, or events | None, or seconds |
| Counter column | A count (followers, likes, stock) | Increment in the write path | None, or seconds |
| Materialised view | A query's full result | Database refresh (full or incremental) | Until refresh |
| Rollup / summary table | Aggregates by time bucket (revenue per hour) | Periodic job or stream processor (Chapter 25) | Bucket duration |
| Read model / projection | A query-shaped copy of data for one screen | Event consumer (Chapter 26) | Seconds |
| Fan-out on write | Each reader's result list (a timeline) | Writer pushes to every reader's list | Seconds (Chapter 36) |

All of them are **derived data** in Chapter 11's sense: they must have one maintenance path, they can
lag, and they need a way to be rebuilt from the source of truth when they drift — and they will
drift, because maintenance code has bugs. A periodic **reconciliation job** that recomputes a sample
(or all) of the derived values and corrects differences is cheap insurance for any counter or rollup
that matters.

Precomputation changes the cost model in a predictable way: a read-heavy system pays a little more
on every write to save a lot on every read. If reads outnumber writes 100:1 (Chapter 2's ratios),
that's a good trade. If writes dominate, or each write would fan out to millions of derived rows,
it's a bad one — the trade-off at the heart of Chapter 36's news feed.

---

## 12.4 Hot Keys and Hot Partitions

Partitioned stores spread load by key. That works only if load is spread across keys — and real
access is skewed. A **hot key** receives a disproportionate share of traffic: the celebrity's
profile, the viral post's like counter, the "today" partition of a time-bucketed table, the global
"total users" counter. Because one key lives on one partition, and one partition lives on one node,
a hot key's throughput is capped by a single node no matter how large the cluster is. Managed stores
make the cap explicit — DynamoDB, for instance, limits each partition to on the order of 1,000 write
units and 3,000 read units per second.

Hot keys come from a few recurring shapes:

| Cause | Example | Remedy |
|---|---|---|
| Celebrity / viral entity | A post with 2 M likes in an hour | Shard the counter; cache reads; coalesce writes |
| Global aggregate in one row | `site_stats.total_signups` | Shard it, or compute asynchronously |
| Time-based partition key | All of today's events under `DAY#2026-11-15` | Add a shard suffix to the key; partition by entity + time |
| Sequential partition key on a range-partitioned store | Auto-increment IDs, timestamps | Hash or salt the key; use hash partitioning (Chapter 14) |
| Read-hot reference data | Feature-flag config read on every request | Cache in process memory with a short TTL |

### Sharding a hot counter

The standard remedy for a write-hot counter is to split it into N physical keys, increment a random
one, and sum them on read:

```js
// Sharded counter: spread increments for one hot logical key over N physical keys,
// so no single row/partition takes every write. Reads sum the shards.
function createShardedCounter(store, name, { shards = 16, rand = Math.random } = {}) {
  return {
    async increment(by = 1) {
      const shard = Math.floor(rand() * shards);   // random: no coordination needed
      await store.incr(`${name}#${shard}`, by);
    },
    async read() {
      let total = 0;
      for (let s = 0; s < shards; s++) total += (await store.get(`${name}#${s}`)) ?? 0;
      return total;                                // N reads instead of 1: the trade
    },
  };
}

const m = new Map();
const store = { async incr(k, by) { m.set(k, (m.get(k) ?? 0) + by); }, async get(k) { return m.get(k); } };
const likes = createShardedCounter(store, 'likes:post42', { shards: 8 });
for (let i = 0; i < 10_000; i++) await likes.increment();
console.log(await likes.read(), m.size); // 10000 8
```

With 8 shards, each physical key takes about an eighth of the writes. Reads cost 8 lookups, so cache
the summed value for a few seconds if it's also read-hot. Two refinements are common: **only shard
keys that are actually hot** (detect them by sampling, and keep cold counters as single keys), and
**write coalescing** — accumulate increments in memory for a second and apply one `+N` instead of N
`+1`s, which cuts write volume by orders of magnitude at the cost of losing up to a second of counts
on a crash.

> ⚠️ **Gotcha** — **Time-bucketed keys create a moving hot spot.** Partitioning events by
> `DAY#<date>` spreads *historical* data evenly but sends *every current write* to today's partition.
> The cluster looks balanced on disk and is bottlenecked on one node. Partition by
> `(entity, time bucket)` — `DEVICE#d7#2026-11-15` — so current writes spread across entities, or add
> a random shard suffix (`DAY#2026-11-15#07`) when writes have no natural entity.

---

## 12.5 Time-Series and Append-Only Data

A large share of the data in modern systems is append-only: events, logs, metrics, audit trails,
location pings, transactions. Modelling it well rests on three habits.

**Bucket by time, partition by entity.** The key is `(entity, bucket)` with a timestamp as the sort
key, and the bucket size comes from the busiest entity's write rate (Chapter 11's partition-size
calculation). Queries for "entity X between t1 and t2" touch one or a few partitions in order.

**Expire by dropping, not deleting.** Retention ("keep 90 days") should remove whole partitions or
files — `DROP` a monthly table partition in PostgreSQL, TTLs that expire whole SSTables in an LSM
store, lifecycle rules on object storage — rather than deleting rows one by one, which is slow in
B-trees and creates tombstone debt in LSM stores (Chapter 10).

**Keep the raw events immutable and derive the rest.** Store what happened, never update it, and
build the current-state views (balances, statuses, dashboards) as projections or rollups from the
event history. Corrections become new events ("reversal of transaction 81") rather than edits, so
the history stays auditable and any projection can be rebuilt by replaying it. Chapters 26 and 43
build on this.

```sql
-- PostgreSQL: monthly range partitions make retention a metadata operation
CREATE TABLE events (
  tenant_id  bigint      NOT NULL,
  event_id   uuid        NOT NULL,
  occurred_at timestamptz NOT NULL,
  kind       text        NOT NULL,
  payload    jsonb       NOT NULL,
  PRIMARY KEY (tenant_id, occurred_at, event_id)   -- partition key must be part of the PK
) PARTITION BY RANGE (occurred_at);

CREATE TABLE events_2026_11 PARTITION OF events
  FOR VALUES FROM ('2026-11-01') TO ('2026-12-01');

-- Retention: instant, no row-by-row deletes, no vacuum debt
DROP TABLE events_2026_08;
```

---

## 12.6 Multi-Tenancy Models

A **multi-tenant** system serves many customers (tenants) from shared infrastructure. Where each
tenant's data lives is one of the most consequential modelling decisions in B2B software, because
changing it later means migrating every customer.

| Model | Isolation | Cost per tenant | Operational load | Noisy-neighbour risk | Fits |
|---|---|---|---|---|---|
| **Shared tables**, `tenant_id` column on every row | Logical only — enforced by queries (or row-level security) | Lowest | One schema to migrate | High | Many small tenants (SaaS self-serve) |
| **Schema per tenant** in a shared database | Stronger; separate tables | Moderate | Migrations run N times; catalogue bloat at thousands of schemas | Medium | Hundreds of mid-sized tenants |
| **Database (or cluster) per tenant** | Strong; separate resources | Highest | N databases to operate, back up, upgrade | Low | Few large or regulated tenants |
| **Hybrid**: shared by default, dedicated for large tenants | Per tier | Varies | Two code paths to keep equivalent | Controlled | Most mature B2B products |

Two rules apply to every model:

- **The tenant ID is part of every key and every query.** In shared tables, make `tenant_id` the
  leading column of the primary key and of every index, so tenants' data is clustered and every
  query is naturally scoped. Enforce it below the application where you can — PostgreSQL's
  **row-level security** can reject any query that doesn't carry the current tenant's ID — because a
  single missing `WHERE tenant_id = …` is a cross-tenant data leak.
- **Plan for the tenant that outgrows the model.** Tenant sizes follow a power law: one customer
  ends up larger than the next hundred combined. The design needs a way to move a tenant from shared
  to dedicated infrastructure without downtime — which is far easier if `tenant_id` already leads
  every key (Chapter 14 covers moving partitions).

---

## 12.7 Modelling for Change

Data outlives code. The model you choose today will be read by code written in five years, so it must
be possible to change without stopping the system.

**Expand, then contract.** Never rename or retype a column in one step. Add the new column (expand),
write both old and new, backfill historical rows in batches, switch reads to the new column, stop
writing the old one, and only then drop it (contract). Each step is independently deployable and
reversible. Chapter 31 covers the operational side of migrations; the modelling side is choosing
changes that can be made this way.

**Make additions cheap.** Nullable columns with no default (or a constant default, which modern
PostgreSQL adds without rewriting the table) can be added to large tables instantly. A `jsonb`
column for genuinely variable attributes (per-category product attributes, integration-specific
settings) avoids a migration for every new field — but core, queried, constrained fields belong in
real columns, where the database can type-check and index them.

**Version anything that's stored and replayed.** Events, messages, and cached serialised objects are
read long after they're written. Give them a schema version, keep readers able to handle every
version still in storage, and upgrade-on-read or re-write in the background.

---

## 12.8 Common Mistakes and Best Practices

**Mistake: denormalising without an owner.** A copied value with several update paths, or none,
drifts silently; every derived value needs exactly one maintenance mechanism and a recompute path.

**Mistake: designing keys before listing access patterns.** In stores without joins, a key chosen for
the first query makes the third query impossible.

**Mistake: sort keys that don't sort.** Unpadded numbers and non-ISO dates in composite sort keys
return results in the wrong order.

**Mistake: a single row for a global counter.** Every increment contends on one lock or one partition;
shard it, coalesce writes, or compute it asynchronously.

**Mistake: time-only partition keys.** All current writes land on today's partition, creating a hot
spot that moves daily.

**Mistake: row-by-row retention deletes.** Deleting old data row by row bloats B-trees and builds
tombstone debt in LSM stores; drop whole partitions instead.

**Mistake: tenant scoping enforced only in application code.** One missing filter leaks one customer's
data to another; lead every key with `tenant_id` and enforce it with row-level security.

**Best practice: normalise first, denormalise measured hot paths.** Keep the source of truth
normalised and derive read-optimised shapes from it.

**Best practice: reconcile derived data on a schedule.** A job that recomputes counters and rollups
from the source catches the bugs that maintenance code will eventually have.

**Best practice: expand–migrate–contract for every schema change.** Each step is safe to deploy and
roll back on its own.

---

## 12.9 Real-World Example: An Access-Pattern-First Key Design

This applies §12.2's method to an e-commerce order domain in a partition/sort-key store. The store
is simulated in memory with exactly the operations such databases offer — put by key, and query one
partition by sort-key prefix — so every access pattern is shown to be a single partition query.

```js
// single-table.mjs — access-pattern-first key design for a partition/sort-key store
// (the DynamoDB model), with a tiny in-memory store so the queries actually run.

// The store offers what such databases offer: put by (PK, SK), and a query over one
// partition with an optional sort-key prefix. Nothing else — no joins, no scans.
class PartitionedStore {
  #parts = new Map(); // PK → array of items sorted by SK
  put(item) {
    const items = this.#parts.get(item.PK) ?? [];
    const i = items.findIndex((x) => x.SK === item.SK);
    if (i >= 0) items[i] = item; else items.push(item);
    items.sort((a, b) => (a.SK < b.SK ? -1 : a.SK > b.SK ? 1 : 0));
    this.#parts.set(item.PK, items);
  }
  query(PK, { prefix = '', descending = false, limit = Infinity } = {}) {
    const items = (this.#parts.get(PK) ?? []).filter((x) => x.SK.startsWith(prefix));
    return (descending ? items.reverse() : items).slice(0, limit);
  }
}

// Key design, derived from the access patterns:
//   AP1 get customer profile                 PK=CUST#id         SK=PROFILE
//   AP2 customer's orders, newest first      PK=CUST#id         SK=ORDER#<iso date>#<orderId>
//   AP3 order with its line items            PK=ORDER#id        SK=META | ITEM#<nnn>
//   AP4 seller's open orders, oldest first   PK=SELLER#id#OPEN  SK=<iso date>#<orderId>  (sparse index)
const keys = {
  profile: (c) => ({ PK: `CUST#${c}`, SK: 'PROFILE' }),
  orderInCustomer: (c, at, o) => ({ PK: `CUST#${c}`, SK: `ORDER#${at}#${o}` }),
  orderMeta: (o) => ({ PK: `ORDER#${o}`, SK: 'META' }),
  orderItem: (o, n) => ({ PK: `ORDER#${o}`, SK: `ITEM#${String(n).padStart(3, '0')}` }),
  sellerOpen: (s, at, o) => ({ PK: `SELLER#${s}#OPEN`, SK: `${at}#${o}` }),
};

const db = new PartitionedStore();
function placeOrder({ customerId, sellerId, orderId, at, items }) {
  // One logical write fans out into several items — the price of cheap reads.
  // A real store would do this in one transaction (or derive the seller index from a stream).
  const summary = { orderId, at, status: 'open', total: items.reduce((s, i) => s + i.price, 0) };
  db.put({ ...keys.orderInCustomer(customerId, at, orderId), ...summary });
  db.put({ ...keys.orderMeta(orderId), customerId, sellerId, ...summary });
  items.forEach((it, n) => db.put({ ...keys.orderItem(orderId, n + 1), ...it }));
  db.put({ ...keys.sellerOpen(sellerId, at, orderId), customerId });
}

db.put({ ...keys.profile('c1'), name: 'Ana', tier: 'gold' });
placeOrder({ customerId: 'c1', sellerId: 's9', orderId: 'o1', at: '2026-10-01', items: [{ sku: 'mug', price: 12 }] });
placeOrder({ customerId: 'c1', sellerId: 's9', orderId: 'o2', at: '2026-11-03',
  items: [{ sku: 'tea', price: 8 }, { sku: 'pot', price: 30 }] });

// Every access pattern is one partition query — no joins, no scans
console.log('AP1', db.query('CUST#c1', { prefix: 'PROFILE' })[0].name);                  // AP1 Ana
console.log('AP2', db.query('CUST#c1', { prefix: 'ORDER#', descending: true }).map((o) => o.orderId)); // AP2 [ 'o2', 'o1' ]
console.log('AP3', db.query('ORDER#o2').map((x) => x.SK));          // AP3 [ 'ITEM#001', 'ITEM#002', 'META' ]
console.log('AP4', db.query('SELLER#s9#OPEN', { limit: 10 }).map((x) => x.SK)); // AP4 [ '2026-10-01#o1', '2026-11-03#o2' ]
// Profile and recent orders in ONE request, because both live in partition CUST#c1
console.log('AP1+2', db.query('CUST#c1', { descending: true }).map((x) => x.SK));
// AP1+2 [ 'PROFILE', 'ORDER#2026-11-03#o2', 'ORDER#2026-10-01#o1' ]
```

Shipping an order removes its `SELLER#s9#OPEN` item, which is what makes that index **sparse** — it
contains only open orders, so "a seller's open orders" never reads closed ones no matter how many
there are.

The decision most readers question is **storing different entity types — the profile and the orders
— in the same partition under generic `PK`/`SK` attribute names**, instead of a clean table per
entity. It looks like a hack. It's the point: in a store without joins, the only way to fetch a
customer and their recent orders in one round trip is for them to be adjacent in one partition, and
that only works if the key attributes are shared and the sort key's prefix (`PROFILE`, `ORDER#…`)
distinguishes the types. The cost is real — the schema is no longer self-describing, ad-hoc queries
are hopeless, and every access pattern has to be designed in advance — which is exactly why this
modelling style belongs to workloads whose access patterns are known and whose scale demands it, and
why Chapter 11's default is still a relational database.

---

## 12.10 Questions

**1.** MCQ — An order row stores the shipping address as it was at purchase time. Is this denormalisation that needs to be kept in sync with the customer's address? a) Yes, on every address change b) No — it's a snapshot of a different fact c) Only for open orders d) Only if the address is indexed
<details><summary>Answer</summary>**b)** The order's address is what it was shipped to, frozen at purchase; changing the customer's current address must *not* change it.</details>

**2.** MCQ — Which sort key component sorts correctly as a string? a) `ITEM#7` b) `DATE#15/11/2026` c) `DATE#2026-11-15` d) `PRICE#9.5`
<details><summary>Answer</summary>**c)** ISO-8601 dates sort lexicographically in chronological order; unpadded numbers and day-first dates don't.</details>

**3.** MCQ — A table partitioned only by `DAY#<date>` receives 50,000 writes/s. What's the problem? a) Too many partitions b) All current writes land on one partition, creating a moving hot spot c) Dates can't be partition keys d) Old partitions can't be read
<details><summary>Answer</summary>**b)** Today's partition takes every write, so one node bottlenecks the whole cluster.</details>

**4.** MCQ — What does a sharded counter trade for higher write throughput? a) Durability b) More expensive reads that must sum N keys c) Exactness d) Storage of the full history
<details><summary>Answer</summary>**b)** Writes spread across N keys; reads sum N keys (and are usually cached).</details>

**5.** MCQ — Which multi-tenancy model offers the lowest per-tenant cost but the weakest isolation? a) Database per tenant b) Schema per tenant c) Shared tables with a `tenant_id` column d) Cluster per tenant
<details><summary>Answer</summary>**c)** Everything is shared; isolation depends on every query carrying the tenant ID (ideally enforced with row-level security).</details>

**6.** MCQ — What's the safest way to rename a column on a large production table? a) `ALTER TABLE … RENAME` during low traffic b) Expand–migrate–contract: add the new column, dual-write, backfill, switch reads, drop the old c) Create a new table and swap it d) Rename in code only
<details><summary>Answer</summary>**b)** Each step is independently deployable and reversible, so running code never breaks — old and new code versions both work during the change.</details>

**7.** Short — Why does every derived value need exactly one maintenance mechanism?
<details><summary>Answer</summary>With several update paths, one will eventually be missed or race with another, and the copy drifts; with none, it's stale from the start. One owner makes correctness checkable, and a recompute path fixes drift.</details>

**8.** Short — What is a sparse index, and why is it useful for "open orders"?
<details><summary>Answer</summary>An index containing only items that have the indexed attribute. Writing an index entry only while an order is open (and removing it on close) means querying open orders never touches the much larger set of closed ones.</details>

**9.** Short — Why should retention remove whole partitions rather than delete rows?
<details><summary>Answer</summary>Dropping a partition or file is a metadata operation. Row-by-row deletes are slow, bloat B-tree tables until vacuumed, and create tombstones in LSM stores that slow reads until compacted.</details>

**10.** Short — How do you enforce "email is unique" in a key-value store without unique constraints?
<details><summary>Answer</summary>Write a separate item keyed by the email (`EMAIL#address`) with a conditional put (fail if exists), in the same transaction as the customer item — so a second customer with the same email fails atomically.</details>

**11.** Short — Why should `tenant_id` lead every primary key and index in a shared-table design?
<details><summary>Answer</summary>It clusters each tenant's data together, makes every query naturally tenant-scoped and efficient, enables row-level security, and makes moving one tenant to dedicated infrastructure a range copy rather than a table-wide scan.</details>

**12.** Output — Using the §12.9 store and keys, what does `db.query('ORDER#o1').map((x) => x.SK)` return?
<details><summary>Answer</summary>`[ 'ITEM#001', 'META' ]` — order o1 has one line item, and `ITEM#…` sorts before `META`.</details>

**13.** Output — What does this print?
```js
const keys = ['ITEM#10', 'ITEM#2', 'ITEM#1'].sort();
const padded = ['ITEM#010', 'ITEM#002', 'ITEM#001'].sort();
console.log(keys, padded);
```
<details><summary>Answer</summary>`[ 'ITEM#1', 'ITEM#10', 'ITEM#2' ] [ 'ITEM#001', 'ITEM#002', 'ITEM#010' ]` — string order puts 10 before 2 unless the numbers are zero-padded.</details>

**14.** Output — A post receives 6,000 likes per second. Each partition accepts at most 1,000 writes per second. What's the minimum number of counter shards, and how many keys must a read sum?
<details><summary>Answer</summary>**6 shards (in practice ~8–10 for headroom), and a read sums all of them.** 6,000 / 1,000 = 6 at exactly full capacity; random shard choice is uneven, so add headroom.</details>

**15.** Output — Write coalescing accumulates increments for 1 second before applying them. A counter receives 20,000 increments/s spread over 200 hot keys. How many store writes per second result, roughly?
<details><summary>Answer</summary>**~200 writes/s** — one `+N` per hot key per second instead of 20,000 individual increments (100× fewer), at the cost of losing up to a second of counts on a crash.</details>

**16.** Debug — A product page shows "142 reviews" while the reviews tab lists 151. The count column is incremented by the review-creation endpoint. What could cause drift, and how do you fix it durably?
<details><summary>Answer</summary>Another path creates or deletes reviews without updating the count (a moderation tool, a bulk import, a delete endpoint), or increments are lost on failures between the two writes. Make one mechanism own the count — same transaction, or a consumer of review change events — and add a reconciliation job that recomputes counts from the reviews table and corrects drift.</details>

**17.** Debug — A wide-column cluster's dashboards show one node at 95% CPU while others idle at 20%. Data is evenly distributed on disk. The busiest table is keyed by `event_date`. Cause and fix?
<details><summary>Answer</summary>A moving hot spot: all current writes go to today's partition, which lives on one node (and its replicas). Re-key as `(entity_id, event_date)` so writes spread across entities, or add a shard suffix to the date key; reads by date then fan out over the shards.</details>

**18.** Debug — A customer reports seeing another company's invoice in their dashboard. The SaaS app uses shared tables with `tenant_id`; the bug was a new report query missing its tenant filter. What structural changes prevent a recurrence?
<details><summary>Answer</summary>Enforce isolation below the application: PostgreSQL row-level security policies keyed on a session-level tenant setting, so a query without the right tenant context returns nothing. Lead every index with `tenant_id`, add tests that run every query under a different tenant, and audit query paths that bypass the standard data-access layer.</details>

**19.** Debug — Nightly retention deletes of events older than 90 days take longer every week, the table keeps growing on disk, and daytime query latency has risen. The job runs `DELETE … WHERE occurred_at < now() - interval '90 days'`. What's happening and what should change?
<details><summary>Answer</summary>Row-by-row deletes are slow, leave dead tuples that bloat the table and indexes until vacuum catches up (which it isn't), and compete with daytime load. Range-partition the table by month and drop expired partitions — an instant metadata operation — then rebuild or vacuum the bloated table once.</details>

**20.** Code — Write `sortKey(...parts)` that builds a composite sort key, zero-padding non-negative integers to 10 digits and passing strings through, joined by `#`.
<details><summary>Answer</summary>

```js
const sortKey = (...parts) =>
  parts.map((p) => (Number.isInteger(p) && p >= 0 ? String(p).padStart(10, '0') : String(p))).join('#');
console.log(sortKey('ORDER', '2026-11-03', 42)); // ORDER#2026-11-03#0000000042
```
</details>

**21.** Code — Write `coalescingCounter(flush, { intervalMs })` returning `{ increment(key, by), stop() }` that accumulates increments in memory and calls `flush(Map<key, total>)` once per interval with the accumulated deltas.
<details><summary>Answer</summary>

```js
function coalescingCounter(flush, { intervalMs = 1_000 } = {}) {
  let pending = new Map();
  const tick = async () => {
    if (pending.size === 0) return;
    const batch = pending; pending = new Map();   // swap first: increments during flush go to the next batch
    await flush(batch);
  };
  const timer = setInterval(tick, intervalMs);
  return {
    increment(key, by = 1) { pending.set(key, (pending.get(key) ?? 0) + by); },
    async stop() { clearInterval(timer); await tick(); }, // flush what's left on shutdown
  };
}
const c = coalescingCounter(async (b) => console.log([...b]), { intervalMs: 50 });
for (let i = 0; i < 1000; i++) c.increment(i % 2 ? 'post:1' : 'post:2');
await c.stop(); // [ [ 'post:2', 500 ], [ 'post:1', 500 ] ]
```
</details>

**22.** Code — Write `bucketKey(entityId, timestamp, bucket)` returning a partition key like `DEVICE#d7#2026-11-15` for `'day'` or `DEVICE#d7#2026-11-15T10` for `'hour'`, using UTC.
<details><summary>Answer</summary>

```js
function bucketKey(entityId, timestamp, bucket = 'day') {
  const iso = new Date(timestamp).toISOString();           // always UTC: buckets mustn't shift with DST
  const cut = { day: 10, hour: 13, minute: 16 }[bucket];
  if (!cut) throw new RangeError(`unknown bucket ${bucket}`);
  return `DEVICE#${entityId}#${iso.slice(0, cut)}`;
}
console.log(bucketKey('d7', Date.UTC(2026, 10, 15, 10, 42), 'hour')); // DEVICE#d7#2026-11-15T10
```
</details>

**23.** Scenario — A B2B SaaS has 3,000 small tenants on shared tables and has just signed a bank that requires its data to be physically separated and encrypted with its own keys. What do you do?
<details><summary>Answer</summary>Move to a hybrid model: keep small tenants shared and give the bank a dedicated database (or cluster) with its own encryption keys, routed by a tenant directory consulted on each request. Because `tenant_id` leads every key, the bank's data can be copied out by range, then cut over with a brief write pause or CDC. Keep one codebase and schema for both deployments, and run the same migrations against both so they never diverge.</details>

**24.** Scenario — A team wants to store each user's notification feed as a single document containing an array of all notifications, "so we can load it in one read". Heavy users receive 300 notifications a day. What do you propose instead?
<details><summary>Answer</summary>The document grows without bound and every new notification rewrites it. Store each notification as its own item keyed `(user_id, created_at#id)` in a partition per user (time-bucketed if a user can be very hot), so "latest 50" is one sorted range read, inserts are appends, and retention can drop old buckets. Keep an unread counter as a separate, small derived value.</details>

---

## Chapter Summary

- Normalise the source of truth and denormalise measured hot paths; snapshots are free, but every
  derived copy needs one owner and a recompute path.
- In partitioned stores, list access patterns first and design partition and sort keys so each
  pattern is a single-partition query, adding secondary indexes for patterns that need another key.
- Precomputed counters, rollups, and views move work from reads to writes — a good trade when reads
  dominate — and need reconciliation because they will drift.
- Hot keys cap throughput at one node; shard write-hot counters, coalesce writes, cache read-hot keys,
  and never partition by time alone.
- Append-only data is bucketed by entity and time, expired by dropping partitions, and kept immutable
  with current state derived from it.
- Lead every key with `tenant_id` and enforce isolation below the application; evolve schemas with
  expand–migrate–contract so no step breaks running code.

---

## Practice Exercises

1. Take a relational schema you know and identify every denormalised value in it; classify each as a
   snapshot or a derived copy and name the mechanism that maintains it.
2. Design partition and sort keys for a ticket-tracking system (projects, tickets, comments,
   assignees) supporting eight access patterns, and check each is one partition query.
3. Read your managed database's documentation on per-partition throughput limits and adaptive capacity,
   and compute the largest single counter it can sustain unsharded.
4. Find the hottest keys in a cache or database you operate (sampling, key-space analysis, or slow-log
   tools), and estimate what share of traffic the top 10 receive.
5. Convert an append-only table you know to monthly range partitions with partition-drop retention, and
   measure the before/after cost of expiring a month of data.

---

## Mini Project: A Social Counter Service

Build a service that tracks likes, views, and follower counts for posts and users, correct under
heavy skew — a few entities receive most of the traffic — and that can always be reconciled with the
source of truth.

- Likes are stored as individual `(post, user)` records (idempotent: a user can like once); counts
  are derived.
- Hot-key detection by sampling; only hot counters are sharded, and sharding is transparent to readers.
- Write coalescing with a configurable flush interval, and a documented bound on counts lost on crash.
- A reconciliation job that recomputes counts from the like records and reports and fixes drift.
- A load test with a Zipf-distributed workload showing per-partition write rates before and after
  sharding.

---

**Next:** [Chapter 13 — Replication →](13-replication.md)
