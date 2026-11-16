# Chapter 11 — Choosing a Data Store

## Learning Objectives

- Derive data-store requirements from access patterns, consistency needs, and scale before naming a product
- Describe what relational, key-value, document, wide-column, graph, search, time-series, and vector stores each do well and badly
- Explain why a relational database is the right default for most new systems, and the evidence that should move you off it
- Recognise when a second data store is justified, and what keeping two in sync will cost
- Record a data-store decision with the conditions under which it should be revisited

---

## 11.1 Start From Access Patterns, Not Products

The most common data-store mistake is choosing the product first — "we'll use MongoDB", "let's put
it in Cassandra" — and discovering the access patterns afterwards. The order has to be reversed,
because every category of store is defined by the access patterns it makes cheap and the ones it
makes expensive or impossible.

Answer these before naming anything:

| Question | Why it decides the store |
|---|---|
| **How is data read?** By key? By range? By arbitrary filters? By relationships several hops deep? By text relevance? By similarity? | Each is a different index structure, and most stores support only some of them well |
| **How is it written?** Inserts only? Frequent updates to the same rows? Bulk loads? | Write-heavy appends favour LSM-based stores; update-heavy mixed workloads favour B-trees (Chapter 10) |
| **What must be transactional?** Does a single operation touch several entities that must change together? | Multi-entity atomicity is native in relational stores and limited or costly elsewhere |
| **How consistent must reads be?** Read-your-writes? Linearisable? Stale by seconds is fine? | Determines replication mode and which stores qualify (Chapters 13, 20) |
| **How big, how fast, how skewed?** Total size, peak QPS, hottest key's share | Decides whether one node suffices or partitioning is required (Chapter 14) |
| **How will the questions change?** Are queries known in advance, or will analysts invent new ones? | Query-first stores punish new queries; relational stores accommodate them |
| **Who operates it?** Managed service? Does the team know it? | A store nobody can debug at 3 a.m. is wrong regardless of fit |

Write the access patterns down as a list — "get order by ID", "list a customer's orders newest
first, 20 at a time", "find open orders older than 1 hour", "sum revenue by day" — with rough rates
for each. That list is the specification every candidate store is checked against.

> 💼 **Interview** — Saying "I'd use NoSQL because it scales" is a red flag. The strong version is:
> "The dominant access pattern is fetching a user's timeline by user ID and time, at 200 k reads
> per second, with no cross-user transactions — that's a partition-key-plus-sort-key pattern, so a
> wide-column store fits. Account balances need multi-row transactions, so those stay relational."

---

## 11.2 Relational Databases: The Default

A relational database (PostgreSQL, MySQL, SQL Server, Oracle) stores data in tables with a fixed
schema, enforces constraints, supports ACID transactions across any rows, and answers ad-hoc
queries — joins, filters, aggregations — through a query planner.

**What it's good at:**

- **Integrity.** Foreign keys, unique constraints, `CHECK` constraints, and transactions keep
  invariants true even when application code has bugs. "No two accounts share an email" is one
  `UNIQUE` constraint, not a race condition.
- **Flexibility of queries.** New questions don't need new data layouts. A query nobody planned for
  can be answered tomorrow with SQL and possibly a new index.
- **Transactions across entities.** Transferring money, reserving inventory and creating an order
  together, or moving a task between projects are single transactions (Chapter 15).
- **Maturity.** Decades of tooling, operational knowledge, managed offerings, and people who know
  how to run them.

**Where it strains:**

- **Write throughput beyond one primary.** A single primary handles thousands to tens of thousands
  of writes per second (Chapter 2). Beyond that you need partitioning (Chapter 14), which the
  database does not do for you in its classic form.
- **Very large datasets on one node.** Multi-terabyte tables work, but backups, schema migrations,
  and replica rebuilds get slow.
- **Schema changes on huge tables** need care (Chapter 31).

The practical guidance for most new systems is **start relational** — usually PostgreSQL — unless
an access pattern clearly rules it out. Its extensions cover a surprising share of "special" needs:
JSONB for semi-structured fields, full-text search for modest search needs, PostGIS for geospatial,
pgvector for embeddings, partitioning for time-series-shaped tables. One store you operate well
beats three you operate badly.

### Distributed SQL

**Distributed SQL** (sometimes "NewSQL") databases — Google Spanner, CockroachDB, YugabyteDB, TiDB —
keep the relational model and transactions but partition data automatically across many nodes,
replicating each partition with consensus (Chapter 21). They remove the single-primary write
ceiling and survive node or zone loss automatically.

The costs are real: every write involves a consensus round trip among replicas, so write latency is
higher than a single-node database's (several milliseconds within a region, much more across
regions); transactions spanning partitions cost more coordination; and query performance depends on
data placement in ways a single-node database never exposes. They're the right answer when you need
relational semantics *and* horizontal write scale or multi-region survivability — and a premature
one when a single PostgreSQL primary would do.

---

## 11.3 Key-Value and Document Stores

### Key-value stores

A **key-value store** maps a key to an opaque value and supports get, put, and delete by key —
sometimes with TTLs, atomic counters, or conditional writes. Because it promises so little, it can
be extremely fast and scale almost linearly by partitioning on the key.

| | In-memory (Redis, Memcached, Valkey) | Durable, partitioned (DynamoDB, Riak-style, FoundationDB layers) |
|---|---|---|
| Latency | Sub-millisecond | Single-digit milliseconds |
| Durability | Optional (snapshots, append-only file) — treat as a cache unless configured otherwise | Durable, replicated |
| Data size | Bounded by RAM | Effectively unbounded |
| Extra structures | Lists, sets, sorted sets, streams, counters | Usually a sort key within a partition, secondary indexes |
| Typical use | Caches, sessions, rate limits, leaderboards, locks (with care — Chapter 21) | Primary store for key-addressed data at large scale |

**DynamoDB-style stores** deserve a note because they blur categories: each item has a **partition
key** (which decides the node) and an optional **sort key** (which orders items within a partition),
so "get all of user 42's orders, newest first" is a single efficient query. Anything *not* expressible
as "one partition key, a range of sort keys" needs a secondary index or a scan. Designing for such
stores means designing the keys around the access patterns up front — Chapter 12's subject.

### Document stores

A **document store** (MongoDB, Couchbase, Firestore) stores self-contained documents, usually JSON-like,
addressed by ID and queryable by fields inside them, with secondary indexes.

The document model's strength is **locality for aggregates**: an order with its line items, shipping
address, and payment summary is one document, read and written as a unit. If the dominant access
pattern is "load this whole thing, change it, save it", documents fit naturally and avoid joins.

Its weaknesses mirror that strength:

- **Relationships across documents** — many-to-many, or data referenced from many places — either
  duplicate data (which must be kept in sync) or require application-side joins.
- **Schema-on-read** means the database doesn't stop you writing inconsistent shapes; old documents
  with old shapes accumulate, and every reader must handle all of them. Schema validation features
  help but must be adopted deliberately.
- **Multi-document transactions** exist in modern document stores but are more expensive and more
  constrained than in relational databases; designs that need them constantly are fighting the model.

> ⚠️ **Gotcha** — Documents that grow without bound break the model. A "user" document that embeds
> every order the user has ever placed works at ten orders and fails at ten thousand: every read
> loads them all, every write rewrites them all, and eventually it hits the size limit (16 MB in
> MongoDB, 400 KB per item in DynamoDB). Embed what's bounded and read together; reference what grows.

---

## 11.4 Wide-Column Stores

**Wide-column stores** — Cassandra, ScyllaDB, Bigtable, HBase — organise data by a **partition key**
(which determines the node) and **clustering columns** (which sort rows within a partition). A
partition can hold many rows, stored together and sorted, so "all readings for device X on day D in
time order" is a single sequential read from one node.

They're built on LSM engines (Chapter 10), partition data automatically, and replicate it with
tunable consistency (Chapter 13). The result is very high write throughput and linear scaling for
access patterns that are **known in advance and keyed by partition**.

The modelling rule is **one table per query**:

```sql
-- Cassandra CQL: the same readings, stored twice, because there are two access patterns.

-- "Readings for a device over a time range" — partition by device and day
CREATE TABLE readings_by_device (
  device_id  uuid,
  day        date,          -- time bucket: bounds partition size
  ts         timestamp,
  value      double,
  PRIMARY KEY ((device_id, day), ts)
) WITH CLUSTERING ORDER BY (ts DESC);

-- "Latest reading per device in a site" — partition by site
CREATE TABLE latest_by_site (
  site_id    uuid,
  device_id  uuid,
  ts         timestamp,
  value      double,
  PRIMARY KEY ((site_id), device_id)
);
```

There are no joins and only limited secondary indexing; a query the table wasn't designed for is a
full-cluster scan, which in practice means "not supported". Data is denormalised into as many tables
as there are query shapes, and the application writes to all of them.

**Partition size** is the key capacity number. Partitions should stay bounded — tens to low hundreds
of megabytes at most — because a partition lives on one node and is read and compacted as a unit:

```js
// Partition size for a table keyed by (device_id, time bucket)
const partitionMB = ({ readingsPerSec, bytesPerReading, bucketSeconds }) =>
  (readingsPerSec * bytesPerReading * bucketSeconds) / 1e6;

console.log(partitionMB({ readingsPerSec: 1, bytesPerReading: 100, bucketSeconds: 86_400 }).toFixed(1));  // 8.6
console.log(partitionMB({ readingsPerSec: 50, bytesPerReading: 100, bucketSeconds: 86_400 }).toFixed(1)); // 432.0
console.log(partitionMB({ readingsPerSec: 50, bytesPerReading: 100, bucketSeconds: 3_600 }).toFixed(1));  // 18.0
```

A device reporting once a second fits comfortably in a daily bucket; a device reporting 50 times a
second produces 432 MB partitions and needs hourly buckets. The bucket size is a design parameter
derived from the write rate of the busiest key, not the average one.

| Choose wide-column when | Avoid it when |
|---|---|
| Write-heavy, append-mostly, at large scale | Queries are ad hoc or still being discovered |
| Every query is "partition key + range of clustering key" | You need multi-row transactions or strong cross-entity invariants |
| You need multi-datacentre replication with tunable consistency | The dataset fits comfortably in one relational database |
| Data has a natural time or entity partition | The team can't absorb operating it (or buy it managed) |

---

## 11.5 Specialised Stores

Some access patterns are different enough that a general-purpose store can't serve them efficiently
at scale, and a specialised store earns its operational cost.

| Store type | Access pattern it optimises | Examples | Chapter |
|---|---|---|---|
| **Search engine** | Full-text relevance, faceting, fuzzy matching, filtering across many fields | Elasticsearch, OpenSearch, Solr | 18 |
| **Time-series** | Append-heavy timestamped metrics, range scans by series, downsampling, retention | Prometheus, InfluxDB, TimescaleDB, VictoriaMetrics | 48 |
| **Graph** | Traversals several hops deep: friends-of-friends, fraud rings, dependency paths | Neo4j, Neptune, graph layers on relational stores | — |
| **Vector** | Nearest-neighbour search over embeddings for semantic search and recommendation | pgvector, dedicated vector databases, search engines' vector fields | — |
| **Column store / warehouse** | Scans and aggregations over billions of rows, few columns | ClickHouse, BigQuery, Snowflake, Redshift, DuckDB | 10, 25 |
| **Object store** | Large immutable blobs, cheap and durable | S3 and compatible stores | 17 |
| **Ledger / append-only** | Immutable, verifiable history | Ledger tables, event stores | 26, 43 |

Two of these need more than a pointer.

**Graph databases** pay off when queries traverse relationships to a depth you don't know in advance.
"Friends of friends who live in Lisbon" is a two-hop join that SQL handles fine; "is there *any* path
of length ≤ 6 between these accounts through shared devices or cards" is a variable-depth traversal
where each hop in SQL is another self-join and a graph database follows pointers directly. If your
traversals are fixed and shallow, a relational store with good indexes is usually enough.

**Vector stores** support similarity search over embeddings — numeric vectors where semantically
similar items are close together. The exact answer is a brute-force scan:

```js
// Brute-force nearest neighbours: exact, simple, and O(N·d) per query.
// Fine for 100k vectors; at 100M it's why approximate indexes (HNSW, IVF) exist.
const dot = (a, b) => a.reduce((s, x, i) => s + x * b[i], 0);
const norm = (a) => Math.sqrt(dot(a, a));
const cosine = (a, b) => dot(a, b) / (norm(a) * norm(b));

function topK(query, items, k = 3) {
  return items
    .map(({ id, vector }) => ({ id, score: cosine(query, vector) }))
    .sort((x, y) => y.score - x.score)
    .slice(0, k);
}

const docs = [
  { id: 'refund-policy', vector: [0.9, 0.1, 0.0] },
  { id: 'shipping-times', vector: [0.1, 0.9, 0.2] },
  { id: 'return-label', vector: [0.8, 0.3, 0.1] },
  { id: 'careers', vector: [0.0, 0.1, 0.9] },
];
console.log(topK([0.85, 0.2, 0.05], docs, 2).map((r) => `${r.id} ${r.score.toFixed(3)}`));
// [ 'refund-policy 0.991', 'return-label 0.990' ]
```

Real embeddings have hundreds to thousands of dimensions, so a brute-force query over 10 million
1,536-dimension vectors is ~15 billion multiply-adds. **Approximate nearest-neighbour (ANN)** indexes
— HNSW graphs, IVF clustering, product quantisation — answer in milliseconds by examining a small
fraction of vectors, trading a little recall (occasionally missing a true nearest neighbour) for
orders of magnitude of speed. The design questions are the ones in this chapter: how many vectors,
how often they change, whether you need metadata filters alongside similarity ("similar products *in
stock in the EU*"), and whether your existing database's vector support is sufficient — which, below
tens of millions of vectors, it often is.

---

## 11.6 Polyglot Persistence and Its Bill

Large systems use several stores — a relational database for orders, Redis for sessions and caching,
a search engine for product search, a warehouse for analytics, object storage for images. This
**polyglot persistence** is normal and often right. But every additional store has a cost that
architecture diagrams don't show:

- **Operations:** another thing to provision, upgrade, monitor, back up, secure, and page someone
  about.
- **Consistency between stores:** the same fact now lives in two places that can disagree. The
  search index says the product is in stock; the database says it sold out a minute ago.
- **Synchronisation machinery:** something must copy changes from the source of truth to the
  derived store — reliably, in order, without losing updates.

That last point is where most polyglot designs go wrong. The tempting approach is the **dual write**:

```js
// ❌ Wrong — dual write. If the process crashes between the two calls, or the second one fails,
// the stores disagree permanently. Retrying doesn't help: there's no record of what was missed.
async function updateProduct(product) {
  await db.query('UPDATE products SET … WHERE id = $1', [product.id]);
  await search.index('products', product.id, product); // may fail, time out, or run out of order
}
```

Two concurrent updates can even reach the two stores in opposite orders, leaving each with a
different "latest" value. The reliable patterns make **one store the source of truth** and derive the
others from its committed changes:

- **Change data capture (CDC):** read the source database's replication log (PostgreSQL logical
  replication, MySQL binlog) and apply each committed change to the derived store, in commit order.
  Nothing is missed, because the log is the record of what committed.
- **Transactional outbox:** write the change *and* an event describing it in the same database
  transaction; a separate process publishes outbox events to the derived stores. Chapter 23 builds it.

Both produce **eventually consistent** derived stores — the search index lags the database by
milliseconds to seconds — so the design must decide which reads may use the derived store and which
must go to the source of truth (checkout reads stock from the database, not from search).

> ⚠️ **Gotcha** — A derived store is only as good as your ability to **rebuild it**. Search indexes,
> caches, and read models get corrupted, need new mappings, or fall behind. If the only way to
> repopulate one is "replay every change since 2019", you have a second source of truth by accident.
> Keep a bulk-rebuild path from the source of truth, and practise it (Chapter 18).

---

## 11.7 Common Mistakes and Best Practices

**Mistake: choosing a product before listing access patterns.** The store gets chosen for its
reputation and then fought for years because its cheap operations aren't the ones you need.

**Mistake: NoSQL "for scale" at small scale.** Giving up transactions, constraints, and ad-hoc queries
to solve a write-throughput problem you don't have is a large, lasting cost for no return.

**Mistake: query-first stores for evolving queries.** Wide-column and key-value stores make unplanned
queries expensive or impossible; products still discovering their questions pay for every new one.

**Mistake: unbounded documents or partitions.** Embedding every child record, or partitioning by a key
whose data grows forever, eventually hits size limits and makes every access slow.

**Mistake: dual writes to keep stores in sync.** Crashes and reordering leave stores permanently
inconsistent; derive secondary stores from the source of truth's committed changes.

**Mistake: reading invariant-critical data from a derived store.** A search index or cache that lags
by seconds will sell stock you don't have.

**Mistake: a new store per feature.** Each one adds operational load and a consistency boundary;
extensions to the store you already run often suffice.

**Best practice: start relational, move with evidence.** Begin with PostgreSQL (or your organisation's
standard relational store) and move specific access patterns elsewhere when measurements show a need.

**Best practice: one source of truth per fact.** Every other copy is derived, rebuildable, and labelled
as possibly stale.

**Best practice: record the decision with its revisit triggers.** Write down the measurements that
would make you change stores, so the decision gets revisited by evidence rather than by fashion.

---

## 11.8 Real-World Example: A Data-Store Decision Record

The artefact of this chapter is a decision record — an ADR, as introduced in Chapter 1 — that a
reviewer can check against the access patterns. This one is for the order service of a mid-sized
marketplace.

```markdown
# ADR-014: Primary data store for the order service

Status: accepted · Supersedes: — · Date: 2026-11-02

## Context
Orders are created at checkout and then updated by payment, fulfilment, and support.
Current: 40k orders/day; forecast 400k/day within 2 years (peak ~60 orders/s, ~600 updates/s).
Order + line items + status history averages 6 KB. ~1.5 TB over 5 years including indexes.

## Access patterns (rate at forecast peak)
1. Create order with line items and reserve inventory atomically      60/s   — transactional
2. Get order by ID (customer, support, fulfilment)                     1.5k/s
3. List a customer's orders, newest first, paginated                   400/s
4. List a seller's open orders by status and age                       150/s
5. Find orders stuck in a status > N minutes (ops job)                 1/min
6. Ad-hoc support and finance queries                                  unknown, rising
7. Revenue by day/seller/category                                      analytics only

## Options considered
- PostgreSQL (managed, one primary + 2 replicas): patterns 1–6 native; 1 needs multi-row
  transactions across orders and inventory, which it provides. Write load is ~1% of a
  single primary's capacity. Team operates 4 PostgreSQL clusters today.
- DynamoDB: 2–4 map well to partition/sort keys; 1 needs a transaction across items and
  inventory (supported, at 2× write cost, limited item count); 5 and 6 need extra GSIs or
  scans; every new support query needs a new index or an export. No team experience.
- Cassandra: excellent write scale we don't need; no multi-row transactions for 1;
  6 effectively unsupported. Rejected.

## Decision
PostgreSQL, partitioned by month on created_at once the table exceeds ~500 GB.
Pattern 7 is served from the warehouse via CDC, never from the primary.
Search over orders (support "find by email/phone") via the existing OpenSearch cluster, fed by CDC.

## Consequences
+ Atomic order creation with inventory; constraints enforce status transitions.
+ New support queries need at most an index, not a data-model change.
– Single-primary write ceiling; vertical scaling and partitioning are our levers.
– CDC pipeline becomes a dependency for analytics and support search freshness (target < 30 s).

## Revisit when
- Sustained writes exceed 30% of primary capacity at peak, or
- Any single customer/seller exceeds 5% of total write traffic (hot-partition risk), or
- Multi-region active-active becomes a requirement (see ADR-009).
```

The part to defend is the **"Revisit when" section**, which most decision records omit. Without it, a
data-store decision is revisited for the wrong reasons — a new engineer's preference, a conference talk,
an incident that had nothing to do with the store — or never revisited at all until it fails. Stating
the triggers as measurements does two things: it makes the decision falsifiable ("are we above 30% of
primary capacity? no — then this isn't the conversation"), and it tells whoever owns the service which
metrics to put on a dashboard *now*, so the moment to change is noticed months before it becomes an
emergency migration. The triggers are also an honest admission that the choice has a ceiling, which is
what makes the choice credible.

---

## 11.9 Questions

**1.** MCQ — Which access pattern most clearly calls for a relational database over a key-value store? a) Get a session by ID b) Atomically create an order, its line items, and an inventory reservation c) Increment a page-view counter d) Store a cached HTML fragment
<details><summary>Answer</summary>**b)** Multi-entity atomic changes with constraints are what relational transactions provide natively.</details>

**2.** MCQ — In a wide-column store, what determines which node holds a row? a) The clustering columns b) The partition key c) The table name d) The write timestamp
<details><summary>Answer</summary>**b)** The partition key is hashed to a node; clustering columns order rows within the partition.</details>

**3.** MCQ — A "user" document embeds every order the user has placed. What eventually goes wrong? a) Nothing; documents are unbounded b) Reads and writes get slower as the document grows, and it eventually hits the size limit c) The database converts it to a table d) Indexes stop working
<details><summary>Answer</summary>**b)** Every access loads and rewrites the whole document, and stores impose maximum document sizes.</details>

**4.** MCQ — What is the main cost of distributed SQL compared with a single-node relational database? a) No SQL support b) No transactions c) Higher write latency from consensus, and costlier cross-partition transactions d) No replication
<details><summary>Answer</summary>**c)** Every write waits for a consensus quorum among replicas, and transactions spanning partitions need extra coordination.</details>

**5.** MCQ — Which query is a good reason to adopt a graph database? a) Count orders per day b) Find whether any path of up to 6 hops links two accounts through shared devices, cards, or addresses c) Get a user by email d) List a customer's last 20 orders
<details><summary>Answer</summary>**b)** Variable-depth traversal is where graph stores outperform repeated self-joins.</details>

**6.** MCQ — What do approximate nearest-neighbour indexes trade for speed? a) Durability b) A small loss of recall c) Write throughput only d) Consistency between replicas
<details><summary>Answer</summary>**b)** They examine a fraction of vectors and occasionally miss a true nearest neighbour.</details>

**7.** Short — Why is "one table per query" the modelling rule for wide-column stores?
<details><summary>Answer</summary>They efficiently serve only "partition key + clustering range" lookups, with no joins and limited secondary indexing. Each distinct query shape therefore needs its own table keyed for it, with the data written to all of them.</details>

**8.** Short — Why are dual writes to a database and a search index unreliable?
<details><summary>Answer</summary>A crash or failure between the writes leaves the stores permanently inconsistent with no record of what was missed, and concurrent updates can reach the two stores in different orders. Deriving the index from the database's committed change log (CDC or outbox) avoids both.</details>

**9.** Short — Name two PostgreSQL capabilities that often make a second, specialised store unnecessary at moderate scale.
<details><summary>Answer</summary>Any two of: JSONB for semi-structured data, full-text search, PostGIS for geospatial, pgvector for embeddings, declarative partitioning for time-series-shaped tables.</details>

**10.** Short — What makes a derived store "rebuildable", and why does it matter?
<details><summary>Answer</summary>A bulk path to repopulate it entirely from the source of truth. Derived stores get corrupted, need re-mapping, or fall behind; without a rebuild path they silently become a second source of truth.</details>

**11.** Short — In a DynamoDB-style store, which query shapes are efficient, and what happens to the rest?
<details><summary>Answer</summary>Get by partition key, or a range of sort keys within one partition. Anything else needs a secondary index (another copy, more write cost) or a scan.</details>

**12.** Output — What does this print?
```js
const partitionMB = (rps, bytes, sec) => (rps * bytes * sec) / 1e6;
console.log(partitionMB(10, 200, 86_400).toFixed(0), partitionMB(10, 200, 3_600).toFixed(1));
```
<details><summary>Answer</summary>`173 7.2` — 10 readings/s of 200 bytes is ~173 MB per daily partition, ~7.2 MB per hourly one.</details>

**13.** Output — Brute-force similarity search over 2 million vectors of 768 dimensions: roughly how many multiply-adds per query, to one significant figure?
<details><summary>Answer</summary>**2 × 10⁹.** 2,000,000 × 768 ≈ 1.5 × 10⁹, which rounds to 2 × 10⁹. At a few billion operations per second per core, that's around a second of CPU per query — why ANN indexes exist.</details>

**14.** Output — Using §11.5's `topK`, what is the top result for the query `[0.0, 0.2, 0.95]` over the same four documents?
<details><summary>Answer</summary>**`careers`.** Its vector `[0.0, 0.1, 0.9]` points almost the same way as the query; cosine similarity ≈ 0.995.</details>

**15.** Output — A wide-column table is keyed `((user_id), event_ts)` and stores 1 KB events. The most active user generates 2,000 events/hour, forever. How large is that user's partition after one year, in GB?
<details><summary>Answer</summary>**~17.5 GB.** 2,000 × 24 × 365 × 1 KB ≈ 17.5 GB in one partition on one node — far too large; add a time bucket to the partition key.</details>

**16.** Debug — A product search page sometimes shows items as in stock that checkout rejects as sold out. Stock is updated in PostgreSQL and the search index is updated by the same request handler right after. What's wrong, and what should change?
<details><summary>Answer</summary>The index is a lagging (and dual-written, possibly inconsistent) copy, and the page treats it as authoritative. Feed the index from CDC so it's reliably eventual, accept that search may be slightly stale, and have checkout and the product detail page read stock from the source of truth.</details>

**17.** Debug — A team on Cassandra needs a new report "orders by coupon code". Their query with `ALLOW FILTERING` times out on production. Why, and what are the options?
<details><summary>Answer</summary>The table isn't partitioned by coupon, so the query scans every partition on every node. Options: a new table keyed by coupon code populated on write (and backfilled), a materialised view or index if the store's implementation is suitable, or — usually best for reports — answering it in the warehouse fed by CDC.</details>

**18.** Debug — After two years, a MongoDB collection's reads are slow and memory pressure is high. Documents for older customers are 4–8 MB; new ones are 20 KB. What happened, and how do you fix it?
<details><summary>Answer</summary>Documents embed unbounded child data (e.g. full order or activity history), so long-lived customers' documents grew huge, and every read loads them whole. Move unbounded children to their own collection referenced by customer ID, keep only bounded summaries embedded, and migrate existing documents incrementally.</details>

**19.** Debug — A team added Redis as the "source of truth" for user carts for speed. After a Redis failover, some carts reverted to older contents and others vanished. Why?
<details><summary>Answer</summary>Redis replication is asynchronous and persistence was likely periodic, so writes acknowledged by the old primary were lost on failover. Either persist carts in a durable store (with Redis as a cache in front) or configure and accept the durability trade-offs explicitly — and treat Redis as a cache unless it's been configured and tested as a database.</details>

**20.** Code — Write `needsBucketing({ eventsPerSecForHottestKey, bytesPerEvent, maxPartitionMB })` returning the largest bucket size from `['day', 'hour', 'minute']` that keeps the hottest partition under the limit, or `null` if none does.
<details><summary>Answer</summary>

```js
function needsBucketing({ eventsPerSecForHottestKey, bytesPerEvent, maxPartitionMB = 100 }) {
  const buckets = [['day', 86_400], ['hour', 3_600], ['minute', 60]];
  for (const [name, seconds] of buckets) {
    const mb = (eventsPerSecForHottestKey * bytesPerEvent * seconds) / 1e6;
    if (mb <= maxPartitionMB) return name;
  }
  return null; // even minute buckets are too big: this key needs to be split another way
}
console.log(needsBucketing({ eventsPerSecForHottestKey: 1, bytesPerEvent: 200 }),
  needsBucketing({ eventsPerSecForHottestKey: 100, bytesPerEvent: 200 })); // day hour
```
</details>

**21.** Code — Write `checkAccessPatterns(store, patterns)` where `store` declares `{ supports: ['key', 'range-in-partition', …] }` and each pattern is `{ name, kind }`, returning the names of patterns the store can't serve efficiently.
<details><summary>Answer</summary>

```js
function checkAccessPatterns(store, patterns) {
  const supported = new Set(store.supports);
  return patterns.filter((p) => !supported.has(p.kind)).map((p) => p.name);
}
const kv = { supports: ['key', 'range-in-partition'] };
console.log(checkAccessPatterns(kv, [
  { name: 'get order', kind: 'key' },
  { name: 'customer orders', kind: 'range-in-partition' },
  { name: 'support search', kind: 'ad-hoc-filter' },
  { name: 'create order + reserve stock', kind: 'multi-entity-txn' },
])); // [ 'support search', 'create order + reserve stock' ]
```
Trivial code, but the habit — checking every access pattern against a store's capabilities before choosing it — is the point.</details>

**22.** Code — Write a minimal `applyChange(index, change)` for a CDC consumer that keeps a derived store in sync, where each change is `{ op: 'upsert' | 'delete', id, row, lsn }`, ignoring any change whose `lsn` is not newer than the one already applied for that id.
<details><summary>Answer</summary>

```js
function applyChange(index, { op, id, row, lsn }) {
  const current = index.get(id);
  if (current && current.lsn >= lsn) return false;          // stale or duplicate: already applied
  if (op === 'delete') index.set(id, { deleted: true, lsn }); // keep a tombstone so late upserts lose
  else index.set(id, { ...row, lsn });
  return true;
}
const idx = new Map();
applyChange(idx, { op: 'upsert', id: 7, row: { stock: 3 }, lsn: 10 });
applyChange(idx, { op: 'upsert', id: 7, row: { stock: 5 }, lsn: 9 });   // arrives late, ignored
console.log(idx.get(7)); // { stock: 3, lsn: 10 }
```
Tracking the source's log position per record makes the consumer idempotent and order-safe even when changes are redelivered.</details>

**23.** Scenario — A startup is building a social app and the founding engineer wants Cassandra "because Instagram uses it". Expected first-year load is 2,000 writes/s at peak, and the feature set is still changing weekly. What do you advise?
<details><summary>Answer</summary>Start with PostgreSQL: 2,000 writes/s is comfortably within a single primary, and weekly-changing features need ad-hoc queries, joins, and cheap schema evolution that query-first stores punish. Record the decision with revisit triggers — sustained writes above a capacity threshold, a specific hot access pattern like timelines at very high read rates — and plan to move *that pattern* (e.g. feeds to a wide-column store or cache, Chapter 36) when the evidence arrives.</details>

**24.** Scenario — A support tool needs "find a customer by any of email, phone, name fragment, or order number" with typo tolerance. The data lives in PostgreSQL. Do you add a search engine?
<details><summary>Answer</summary>Check whether PostgreSQL suffices first: exact lookups by email, phone, and order number are ordinary indexes; name fragments with typos can use trigram indexes (`pg_trgm`). For an internal support tool at moderate scale, that's often enough and avoids a new store. Add a search engine, fed by CDC with a rebuild path, if relevance ranking, multi-field fuzzy search across large volumes, or latency requirements exceed what trigram search delivers.</details>

---

## Chapter Summary

- Choose data stores from a written list of access patterns, rates, consistency needs, and
  operational ability — never from a product's reputation.
- Relational databases are the right default: transactions, constraints, and ad-hoc queries are hard
  to give up, and extensions cover many specialised needs at moderate scale.
- Key-value and wide-column stores scale writes and key-addressed reads near-linearly, but only for
  access patterns designed into their keys; unplanned queries are expensive or impossible.
- Document stores fit bounded aggregates read and written as a unit; unbounded embedding and
  cross-document relationships fight the model.
- Specialised stores — search, time-series, graph, vector, columnar — earn their cost when an access
  pattern genuinely needs their index structure.
- Every extra store is a consistency boundary: keep one source of truth, derive the rest via CDC or an
  outbox, never dual-write, and keep a rebuild path.

---

## Practice Exercises

1. For a system you work on, write the full access-pattern list with peak rates and consistency
   needs, then check each against its current data store.
2. Model the same small domain (customers, orders, line items) in PostgreSQL, a document store, and a
   wide-column store, and list which of five queries each model serves efficiently.
3. Read the documentation on item or partition limits for DynamoDB, Cassandra, and MongoDB, and record
   the numbers that would constrain your model.
4. Load 1 million 768-dimension random vectors into PostgreSQL with pgvector and compare exact search
   latency with an HNSW index, recording recall at k = 10.
5. Write an ADR for a real data-store decision on your team, including a "Revisit when" section with
   measurable triggers, and add those metrics to a dashboard.

---

## Mini Project: One Domain, Three Stores

Implement the same small product — a task tracker with users, projects, and tasks — on PostgreSQL, a
document store, and a key-value or wide-column store, behind one repository interface, and compare
them honestly.

- Eight defined access patterns, including one transactional (move a task between projects
  atomically) and one ad-hoc (tasks matching several optional filters).
- The same API and test suite run against all three implementations.
- Measure latency and throughput for each access pattern at a fixed data size.
- Record which patterns each store couldn't serve without a new index, table, or scan.
- Write a one-page conclusion naming which store you'd choose for which scale, with revisit triggers.

---

**Next:** [Chapter 12 — Data Modeling for Access Patterns →](12-data-modeling.md)
