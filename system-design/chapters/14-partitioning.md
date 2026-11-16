# Chapter 14 — Partitioning

## Learning Objectives

- Choose between range and hash partitioning from a workload's query shapes and key distribution
- Explain why `hash(key) % N` breaks on resize and how consistent hashing and fixed slots fix it
- Move partitions between nodes online without losing writes or serving stale data
- Compare local and global secondary indexes, and predict the read or write cost of each
- Choose a partition key that spreads load, keeps common queries local, and survives growth

---

## 14.1 Why Partition

Replication (Chapter 13) copies the *same* data to several machines. That scales reads and
survives failures, but every copy still holds everything and the leader still takes every write.
When the data no longer fits on one machine, or the write rate exceeds what one leader can absorb,
the data itself must be split: each machine holds a **partition** (also called a shard, range,
region, tablet, or vnode, depending on the system), and each record belongs to exactly one
partition.

Partitioning and replication are combined in practice. Each partition is replicated, and each node
holds replicas of several partitions — leading some and following others — so load and failures
spread evenly:

```text
            partition P1        partition P2        partition P3
node A      leader              follower            follower
node B      follower            leader              follower
node C      follower            follower            leader
```

The goal is to spread data and load **evenly**. A partitioning scheme that puts 40% of the traffic
on one partition has wasted most of the cluster: the hot partition saturates while others idle, and
the cluster's capacity is set by its busiest node. That uneven load is called **skew**, and avoiding
it — while keeping common queries on a single partition — is the whole design problem.

Before partitioning a relational database, exhaust the cheaper options (Chapter 1's design loop):
better indexes, a larger instance, read replicas for read load, caching, and archiving cold data.
Partitioning is a one-way door: it makes cross-partition queries, transactions, and unique
constraints harder forever.

---

## 14.2 Range Partitioning

**Range partitioning** assigns each partition a contiguous range of keys — A–F, G–M, N–Z, or
timestamps by month. Keys stay sorted within and across partitions, so a **range scan** ("all events
for tenant 42 between March and May") reads one or a few adjacent partitions in order. Bigtable,
HBase, CockroachDB, TiDB, and MongoDB's range sharding work this way, and most of them **split**
ranges automatically when a partition grows past a size threshold and move the halves to balance
nodes.

The weakness is **sequential keys**. If the key is a timestamp or an auto-increment ID, every new
write lands in the last range — the same moving hot spot Chapter 12 described for time buckets. The
cluster can have a hundred nodes and still write at the speed of one. Remedies:

- **Lead the key with something high-cardinality** — `(tenant_id, timestamp)` or
  `(device_id, timestamp)` — so concurrent writes spread across ranges while each entity's data stays
  contiguous.
- **Prefix with a small hash or salt** (`hash(id) % 16 || timestamp`), accepting that a time-range
  scan now fans out to 16 ranges.

Range boundaries also need choosing. Even-width boundaries (A–I, J–R, S–Z) are uneven in practice
(far more names start with S than Q); systems that split by size adapt to the real distribution.

---

## 14.3 Hash Partitioning and the Problem With Modulo

**Hash partitioning** runs the key through a hash function and assigns partitions by hash value. A
good hash spreads even sequential or skewed keys uniformly, which eliminates the moving-hot-spot
problem — at the cost of **range scans**: adjacent keys land on unrelated partitions, so "user IDs
1000–2000" touches every partition.

The obvious implementation is `partition = hash(key) % N`. It distributes evenly — until N changes:

```js
import { createHash } from 'node:crypto';
const hash32 = (s) => createHash('md5').update(s).digest().readUInt32BE(0);
const keys = Array.from({ length: 100_000 }, (_, i) => `user:${i}`);

// mod-N: the obvious scheme, and what happens when N changes
const modN = (key, n) => hash32(key) % n;
const movedModN = keys.filter((k) => modN(k, 4) !== modN(k, 5)).length / keys.length;
console.log(`mod-N, 4 → 5 nodes: ${(movedModN * 100).toFixed(0)}% of keys move`);
// mod-N, 4 → 5 nodes: 80% of keys move
```

Adding a fifth node moves 80% of all keys — in general, a fraction (N)/(N+1) of them. For a cache,
that means an 80% miss rate the moment the cluster grows, which can take down the database behind it.
For a database, it means copying most of the data to add one machine. Both of the schemes below exist
to fix this.

---

## 14.4 Consistent Hashing

**Consistent hashing** maps both keys and nodes onto the same circular hash space — a ring of 2³²
positions. Each key belongs to the first node found moving clockwise from the key's position. When a
node joins, it takes over only the keys between its position and its predecessor's; when a node
leaves, only its keys move, to its successor. On average, adding one node to N moves just 1/(N+1) of
the keys.

With one position per node, the arcs between nodes are random in length, so load is badly uneven, and
a departing node dumps all its keys onto a single neighbour. **Virtual nodes** fix both: each physical
node is placed at many positions (100–500 is typical), so it owns many small arcs, load averages out,
and a departing node's keys scatter across all the survivors.

```js
// hash-ring.mjs — consistent hashing with virtual nodes
import { createHash } from 'node:crypto';
const hash32 = (s) => createHash('md5').update(s).digest().readUInt32BE(0);

export class HashRing {
  #points = []; // sorted [{ h, node }]
  constructor(nodes = [], { vnodes = 100 } = {}) {
    this.vnodes = vnodes;
    nodes.forEach((n) => this.add(n));
  }
  add(node) {
    // Each physical node owns many small arcs, so load evens out and a departing
    // node's keys scatter across all survivors instead of landing on one neighbour
    for (let v = 0; v < this.vnodes; v++) this.#points.push({ h: hash32(`${node}#${v}`), node });
    this.#points.sort((a, b) => a.h - b.h);
  }
  remove(node) { this.#points = this.#points.filter((p) => p.node !== node); }
  lookup(key) {
    const h = hash32(key);
    let lo = 0, hi = this.#points.length;      // first point clockwise from h (binary search)
    while (lo < hi) { const mid = (lo + hi) >> 1; if (this.#points[mid].h < h) lo = mid + 1; else hi = mid; }
    return this.#points[lo % this.#points.length].node; // wrap around the ring
  }
}

const keys = Array.from({ length: 100_000 }, (_, i) => `user:${i}`);
const spread = (ring) => {
  const counts = {};
  for (const k of keys) { const n = ring.lookup(k); counts[n] = (counts[n] ?? 0) + 1; }
  const v = Object.values(counts);
  return `max/min load ${(Math.max(...v) / Math.min(...v)).toFixed(2)}`;
};

for (const vnodes of [1, 10, 100, 500]) {
  console.log(`${String(vnodes).padStart(3)} vnodes/node: ${spread(new HashRing(['a', 'b', 'c', 'd'], { vnodes }))}`);
}
//   1 vnodes/node: max/min load 19.68
//  10 vnodes/node: max/min load 1.68
// 100 vnodes/node: max/min load 1.34
// 500 vnodes/node: max/min load 1.07

const ring = new HashRing(['a', 'b', 'c', 'd'], { vnodes: 100 });
const before = keys.map((k) => ring.lookup(k));
ring.add('e');
const moved = keys.filter((k, i) => ring.lookup(k) !== before[i]);
console.log(`ring, 4 → 5 nodes: ${((moved.length / keys.length) * 100).toFixed(0)}% of keys move, all to e: ${moved.every((k) => ring.lookup(k) === 'e')}`);
// ring, 4 → 5 nodes: 21% of keys move, all to e: true
```

With one position per node, the busiest node holds nearly 20 times the keys of the quietest; with 500
virtual nodes, the spread is within 7%. Adding a fifth node moves 21% of keys — close to the ideal
1/5 — and every moved key moves *to the new node*, never between existing ones.

Virtual nodes also make heterogeneous hardware easy: give a machine twice the capacity twice the
virtual nodes. Consistent hashing is used by Dynamo-style stores (Cassandra, Riak, DynamoDB
internally), by client-side cache sharding (Memcached clients), and by load balancers for cache
affinity (Chapter 7, with bounded loads).

Two alternatives are worth knowing. **Rendezvous (highest-random-weight) hashing** scores every node
for each key (`hash(key, node)`) and picks the highest; it needs no ring, balances well, and moves
the minimum on change, at O(N) per lookup — fine for tens of nodes. **Jump consistent hash** maps a key
to one of N buckets with no memory and near-perfect balance, but only supports adding or removing
buckets at the end, which suits numbered shards better than named nodes.

> 💼 **Interview** — "How would you distribute keys across cache servers so adding a server doesn't
> invalidate everything?" is the consistent-hashing question. Give the ring, explain virtual nodes
> (balance, and spreading a failed node's load), and quantify: mod-N moves N/(N+1) of keys; the ring
> moves about 1/(N+1).

---

## 14.5 Rebalancing: Moving Partitions While Serving Traffic

Partitions must move when nodes are added, removed, or replaced, and when load is uneven. Three
broad approaches decide what moves:

| Approach | How | Used by | Trade-off |
|---|---|---|---|
| **Fixed number of partitions** | Create many more partitions than nodes (e.g. 1,024 or 16,384 slots) up front; rebalancing reassigns whole partitions | Redis Cluster, Elasticsearch shards, Couchbase vBuckets, many app-level sharding layers | Simple and explicit; partition count must be chosen well in advance |
| **Dynamic splitting** | Split a partition when it exceeds a size or load threshold; merge small ones | Bigtable, HBase, CockroachDB, TiDB, MongoDB | Adapts to data; more machinery |
| **Proportional to nodes** | Each node owns a fixed number of partitions (e.g. virtual nodes); joining nodes split existing ones | Cassandra, consistent-hashing designs | Partition size grows with data per node |

With fixed slots, the routing function is two steps — `slot = hash(key) % SLOTS` (which never
changes), then `node = slotMap[slot]` (which does) — so moving data means editing a small table rather
than changing a hash function. The real-world example in §14.10 implements it.

Moving a partition online is always the same recipe, the "snapshot then catch up" of Chapter 13:

1. **Mark the partition as migrating** and tell routers. New writes go to the destination (or to both).
2. **Copy existing data** from source to destination, throttled so the copy doesn't overload either.
3. **Catch up** any changes made on the source during the copy (from its log), unless writes were
   already redirected.
4. **Cut over**: the destination becomes the owner; routers update; reads stop consulting the source.
5. **Clean up** the source's copy after a safety period.

> ⚠️ **Gotcha** — **Automatic rebalancing can turn a blip into an outage.** A node is slow for a
> minute; the system declares it dead and starts moving its partitions; the moves add load to every
> other node and to the network, which makes more nodes slow, which triggers more moves. Many
> production systems therefore make rebalancing *proposals* automatic but require an operator (or a
> long, conservative timer) to *start* large data movements, and throttle them.

---

## 14.6 Secondary Indexes Across Partitions

Partitioning by primary key makes "get by primary key" a single-partition operation. But queries often
filter on something else — orders by status, users by email, products by colour. A secondary index
in a partitioned store comes in two forms:

**Local (document-partitioned) indexes.** Each partition indexes only its own records. Writes update
one partition's index, so they stay cheap and transactional. But a query by the indexed field doesn't
know which partitions hold matches, so it must **scatter** to every partition and **gather** the
results — and inherits the slowest partition's latency (Chapter 3's tail amplification).

**Global (term-partitioned) indexes.** The index itself is partitioned by the *indexed value*: all
entries for `status = 'open'` (or emails starting with `a`) live on one index partition. A query by
that value goes to one index partition. But a single record write may update index partitions on other
nodes, so global indexes are usually updated **asynchronously** — briefly stale — or require a
distributed transaction.

| | Local secondary index | Global secondary index |
|---|---|---|
| Write cost | One partition, synchronous | Possibly several partitions; usually async |
| Read by indexed field | Scatter-gather to all partitions | One (or few) index partitions |
| Freshness | Consistent with the data | Often eventually consistent |
| Tail latency | Max over all partitions | One partition's latency |
| Examples | Cassandra secondary indexes, MongoDB per-shard indexes, Elasticsearch | DynamoDB GSIs, application-maintained lookup tables |

A common middle path is the one Chapter 12 showed: an **application-maintained lookup table** —
`EMAIL#ana@example.com → user 42` — which is a global index you own, written in the same transaction as
the record where the store supports it, or via an event consumer where it doesn't.

---

## 14.7 Routing and Cross-Partition Work

Something must map each request to the partition — and node — that holds its data. Three placements:

```text
(a) any node forwards         (b) routing tier             (c) partition-aware client
client ─▶ node A ─▶ node C    client ─▶ router ─▶ node C   client (knows map) ─▶ node C
```

Whatever the placement, the **partition map** — which partitions exist and which node leads each — must
be kept current and agreed on. Systems store it in a coordination service (ZooKeeper or etcd — Chapter
21), spread it by gossip (Cassandra), or keep it in the cluster's own replicated metadata (Redis Cluster,
CockroachDB). Clients cache the map, and nodes reply "wrong partition; it's moved to C" (Redis's `MOVED`)
so stale caches correct themselves. Versioning the map, as §14.10 does, lets routers detect staleness
cheaply.

Operations that span partitions are where partitioning's costs concentrate:

- **Cross-partition queries** scatter-gather, paying the slowest partition's latency and N× the request
  load. Keep common queries single-partition by choosing the partition key around them.
- **Cross-partition transactions** need distributed commit protocols — slow, complex, and covered in
  Chapter 23 — or a redesign so the transaction's data shares a partition.
- **Joins** across partitions are scatter-gather joins. Co-locate data that's joined together by giving
  related tables the same partition key — all of a tenant's tables partitioned by `tenant_id`, so a
  tenant's joins are local (Citus calls these "co-located tables").
- **Global uniqueness** (unique email across all users) needs a global index or a single-partition
  lookup table, since each partition can only enforce uniqueness locally.

---

## 14.8 Choosing a Partition Key

The partition key is the most consequential decision in a partitioned design, and the hardest to change
later. A good key has four properties:

| Property | Why | Failure when absent |
|---|---|---|
| **High cardinality** | Many distinct values to spread across partitions | `country` as a key: a few huge partitions |
| **Even load** | Traffic per value is roughly balanced — or hot values are handled specially | Celebrity user IDs, a dominant tenant |
| **Query locality** | The most frequent queries and transactions touch one partition | Every request scatter-gathers |
| **Stable over time** | Values don't concentrate on "now" | Timestamps and sequential IDs: a moving hot spot |

In practice the answer is usually the **entity that owns the data and appears in nearly every query**:
`tenant_id` in B2B SaaS, `user_id` in consumer apps, `conversation_id` for messages, `device_id` for
telemetry. Compound keys (`tenant_id, project_id`) refine it when one value grows too large.

**Sharding an existing relational database** is a large migration, typically done in this order: choose
the key and add it to every table that lacks it (often `tenant_id`); make every query carry it;
introduce a routing layer — application-level, or a sharding layer such as Vitess (MySQL) or Citus
(PostgreSQL); move tenants or key ranges to new shards with the online-move recipe in §14.5; and handle
the queries that genuinely span shards separately (often moving them to a warehouse). The most common
reason these migrations take years is the long tail of queries that don't carry the partition key.

> ⚠️ **Gotcha** — **The biggest tenant will outgrow any one partition.** Customer sizes follow a power
> law; with `tenant_id` as the key, one customer eventually exceeds a single node. Plan for it before it
> happens: allow a large tenant's data to be sub-partitioned (`tenant_id, project_id`), or give it a
> dedicated shard (Chapter 12's hybrid tenancy), and make the routing layer able to express both.

---

## 14.9 Common Mistakes and Best Practices

**Mistake: `hash(key) % N` for anything that resizes.** Adding one node remaps N/(N+1) of keys — a
cache-miss storm or a near-total data copy.

**Mistake: consistent hashing without virtual nodes.** Random arc lengths make load wildly uneven, and a
failed node's whole load lands on one neighbour.

**Mistake: range partitioning on a sequential key.** Every current write goes to the last range, so the
cluster writes at the speed of one node.

**Mistake: a low-cardinality partition key.** A handful of distinct values can't spread across many nodes,
and the largest value dominates.

**Mistake: partition keys that common queries don't carry.** Every such query becomes a scatter-gather
across the whole cluster.

**Mistake: aggressive automatic rebalancing.** Data movement triggered by a transient slowdown adds load
everywhere and can cascade into a cluster-wide outage.

**Mistake: partitioning before exhausting vertical options.** Indexes, bigger instances, replicas, and
caching are reversible; partitioning makes cross-partition operations hard forever.

**Best practice: many more partitions than nodes.** Fixed slots or virtual nodes make rebalancing a
matter of moving small units, and make capacity changes incremental.

**Best practice: co-locate data that's queried or transacted together.** Give related tables the same
partition key so the common path never crosses partitions.

**Best practice: version the partition map.** Routers and clients detect stale maps cheaply, and nodes
can redirect requests that arrive at the wrong place.

---

## 14.10 Real-World Example: A Slot-Based Shard Map With Online Moves

This is the fixed-partition scheme from §14.5: keys hash to one of 1,024 slots (a function that never
changes), a versioned map assigns slots to shards, and a slot can be moved to another shard while reads
and writes continue. The demo moves a slot while a concurrent write lands in the middle of the
migration.

```js
// shard-map.mjs — a fixed number of hash slots mapped to shards, with online slot moves.
// The shape used by Redis Cluster (16,384 slots) and many application-level sharding layers.
import { createHash } from 'node:crypto';

const SLOTS = 1_024;
export const slotOf = (key) => createHash('md5').update(key).digest().readUInt16BE(0) % SLOTS;

export class ShardMap {
  version = 1;
  #owner = new Array(SLOTS);       // slot → shard name
  #migrating = new Map();          // slot → { from, to } while a move is in progress
  #shards;                         // name → store with get/set/delete/keysInSlot

  constructor(shards) {
    this.#shards = shards;
    const names = Object.keys(shards);
    for (let s = 0; s < SLOTS; s++) this.#owner[s] = names[s % names.length]; // initial even split
  }

  async get(key) {
    const slot = slotOf(key), move = this.#migrating.get(slot);
    if (move) {
      // During a move the key may be on either side: the destination wins if it has it,
      // because every write since the move began has gone there
      const v = await this.#shards[move.to].get(key);
      return v !== undefined ? v : this.#shards[move.from].get(key);
    }
    return this.#shards[this.#owner[slot]].get(key);
  }

  async set(key, value) {
    const slot = slotOf(key), move = this.#migrating.get(slot);
    // New writes for a migrating slot go straight to the destination, so the copy
    // below never overwrites a newer value with an older one
    return this.#shards[move ? move.to : this.#owner[slot]].set(key, value);
  }

  async moveSlot(slot, to, { batch = 100 } = {}) {
    const from = this.#owner[slot];
    if (from === to) return;
    this.#migrating.set(slot, { from, to });
    this.version++;                                        // routers refresh on version change
    for (const key of await this.#shards[from].keysInSlot(slot)) {
      // Copy only if the destination has no newer write. A real store makes this one
      // conditional put ("set if absent"), closing the gap between the check and the set.
      if ((await this.#shards[to].get(key)) === undefined) {
        await this.#shards[to].set(key, await this.#shards[from].get(key));
      }
      await this.#shards[from].delete(key);
      if (batch-- === 0) { batch = 100; await new Promise((r) => setImmediate(r)); } // throttle
    }
    this.#owner[slot] = to;                                // cut over
    this.#migrating.delete(slot);
    this.version++;
  }

  slotsOwnedBy(name) { return this.#owner.filter((o) => o === name).length; }
}

// In-memory shard store
const memShard = () => {
  const m = new Map();
  return {
    async get(k) { return m.get(k); }, async set(k, v) { m.set(k, v); }, async delete(k) { m.delete(k); },
    async keysInSlot(slot) { return [...m.keys()].filter((k) => slotOf(k) === slot); },
    size: () => m.size,
  };
};

const shards = { s1: memShard(), s2: memShard(), s3: memShard() };
const map = new ShardMap(shards);
for (let i = 0; i < 30_000; i++) await map.set(`user:${i}`, { n: i });

// Move one slot to s3 while a concurrent writer updates a key in that slot
const victim = 'user:42', slot = slotOf(victim);
const move = map.moveSlot(slot, 's3');
await map.set(victim, { n: 42, updated: true });            // lands mid-migration
await move;
console.log((await map.get(victim)).updated, map.slotsOwnedBy('s3'), map.version); // true 342 3
console.log(Object.values(shards).map((s) => s.size()).join(' + '), '=', 30_000);   // 9967 + 9821 + 10212 = 30000
```

The write that arrived mid-migration survived the copy, the slot now belongs to s3 (342 slots instead of
341), the map's version advanced twice, and every one of the 30,000 keys is on exactly one shard.

The decision to defend is **a fixed slot table instead of a consistent-hashing ring**, given that §14.4
just showed the ring moving the minimum data automatically. The ring's strength — placement emerges from
hashing node names — is also its operational weakness: nobody chose which keys a new node takes, so you
can't move one hot tenant, drain one rack, or move a single overloaded slot without changing node
positions and moving whatever else comes along. A slot table is a small, explicit, versioned document
(1,024 entries) that operators and automation can inspect, diff, and change one slot at a time, with
every move following the same online recipe. The hash-to-slot step still gives the even spread and
cheap routing; the slot-to-shard step gives control. Choose the ring when nodes come and go
automatically and nobody wants to manage placement; choose slots when placement is something you need
to reason about.

---

## 14.11 Questions

**1.** MCQ — Which partitioning scheme supports efficient range scans over the partition key? a) Hash b) Range c) Consistent hashing d) Random
<details><summary>Answer</summary>**b)** Range partitioning keeps keys sorted across partitions, so a key range maps to one or a few adjacent partitions.</details>

**2.** MCQ — With `hash(key) % N`, approximately what fraction of keys move when N goes from 9 to 10? a) 10% b) 50% c) 90% d) 100%
<details><summary>Answer</summary>**c)** About N/(N+1) = 9/10 of keys map to a different partition.</details>

**3.** MCQ — What do virtual nodes improve in consistent hashing? a) Hash function speed b) Load balance and how a failed node's load is spread c) Range scan support d) Replication factor
<details><summary>Answer</summary>**b)** Many small arcs per node average out load, and a departing node's keys scatter across all survivors.</details>

**4.** MCQ — A query filters by a field covered by a *local* secondary index in a 40-partition store. How many partitions does it contact? a) 1 b) The partitions holding matches only c) All 40 d) None
<details><summary>Answer</summary>**c)** Each partition indexes only its own data, so the query can't know where matches are and must scatter to every partition.</details>

**5.** MCQ — Which partition key is most likely to create a moving hot spot under range partitioning? a) `user_id` (random UUID) b) `tenant_id` c) `created_at` timestamp d) `hash(order_id)`
<details><summary>Answer</summary>**c)** All current writes have the newest timestamps and land in the last range.</details>

**6.** MCQ — Why do co-located tables (same partition key) help a sharded relational database? a) They compress better b) Joins and transactions for one key value stay on one shard c) They need fewer indexes d) They replicate faster
<details><summary>Answer</summary>**b)** A tenant's rows across tables live together, so the common joins and transactions never cross shards.</details>

**7.** Short — Why does partitioning usually come after vertical scaling, replicas, and caching in the design loop?
<details><summary>Answer</summary>Those options are reversible and keep all operations local. Partitioning permanently makes cross-partition queries, transactions, joins, and uniqueness harder, so it's justified only when data size or write rate genuinely exceeds one node.</details>

**8.** Short — Describe the general recipe for moving a partition online.
<details><summary>Answer</summary>Mark it migrating and route new writes to the destination (or both); copy existing data with throttling; catch up changes made during the copy; cut over ownership and update routers; clean up the source after a safety period.</details>

**9.** Short — Why are global secondary indexes usually eventually consistent?
<details><summary>Answer</summary>The index is partitioned by indexed value, so one record write may need to update index partitions on other nodes; doing that synchronously would need a distributed transaction, so stores update them asynchronously.</details>

**10.** Short — What is the trade-off between rendezvous hashing and a consistent-hashing ring?
<details><summary>Answer</summary>Rendezvous hashing needs no ring or virtual nodes and balances well with minimal movement, but costs O(N) per lookup (score every node); a ring costs O(log V) per lookup with a sorted list of virtual nodes. Rendezvous suits tens of nodes; rings suit larger clusters.</details>

**11.** Short — Why can automatic rebalancing cause cascading failure?
<details><summary>Answer</summary>Moving data adds disk, CPU, and network load to every node involved. If a transient slowdown triggers moves, the added load makes more nodes slow, triggering more moves — a feedback loop. Throttling and operator-gated large moves break it.</details>

**12.** Output — What does this print?
```js
const moved = (n) => n / (n + 1);
console.log(moved(4).toFixed(2), moved(19).toFixed(2), (1 / (19 + 1)).toFixed(2));
```
<details><summary>Answer</summary>`0.80 0.95 0.05` — with mod-N, going from 19 to 20 nodes moves 95% of keys; consistent hashing moves about 5%.</details>

**13.** Output — Using §14.4's `HashRing` with nodes `a`–`d` and 100 virtual nodes, after `ring.remove('b')`, which nodes can keys that previously mapped to `b` now map to?
<details><summary>Answer</summary>**Any of `a`, `c`, or `d`.** Each of b's 100 arcs is absorbed by whichever node's point follows it on the ring, so b's keys scatter across all survivors; keys that didn't map to b don't move.</details>

**14.** Output — A cluster uses 16,384 fixed slots across 8 nodes and adds 2 more. If rebalancing moves the minimum number of slots to even out ownership, about how many slots move?
<details><summary>Answer</summary>**~3,277.** Each node should own 16,384 / 10 ≈ 1,638; the two new nodes need ~3,277 slots between them, taken from the existing eight (~410 each). That's 20% of the data, the minimum possible.</details>

**15.** Output — A scatter-gather query hits 50 partitions in parallel. Each partition independently exceeds 100 ms with probability 2%. What's the probability the query exceeds 100 ms?
<details><summary>Answer</summary>**~64%.** 1 − 0.98⁵⁰ ≈ 0.636 — Chapter 3's tail amplification applied to partitions.</details>

**16.** Debug — A Memcached tier sharded with `hash(key) % N` was scaled from 6 to 7 nodes during a traffic spike; the database behind it immediately overloaded. Why, and how should sharding work?
<details><summary>Answer</summary>Changing N remapped ~86% of keys, so almost every lookup missed the cache at once and fell through to the database. Use consistent hashing with virtual nodes (only ~1/7 of keys move, all to the new node), and add capacity before peaks rather than during them.</details>

**17.** Debug — An event store range-partitioned by `event_id` (a time-ordered ID) has 24 nodes, but sustained writes cap at roughly one node's throughput, and one node is always much busier than the rest — a different one each few hours. Cause and fix?
<details><summary>Answer</summary>Time-ordered IDs make the newest range the only one receiving writes; when it splits, the newer half becomes the hot one. Lead the key with a high-cardinality entity (`(stream_id, event_id)`), or salt it with a small hash prefix and accept that time-range scans fan out.</details>

**18.** Debug — After moving to a sharded database, a support search ("find users by email") went from 5 ms to 900 ms p99. The users table is sharded by `user_id`. Why, and what's the fix?
<details><summary>Answer</summary>Email isn't the partition key, so every lookup scatters to all shards and waits for the slowest. Maintain a global lookup table keyed by email (`email → user_id`) — written transactionally where possible, or via an event consumer — so the lookup is two single-shard reads.</details>

**19.** Debug — A cluster lost one node for 90 seconds during a kernel upgrade; automatic rebalancing started moving its partitions, other nodes' latency spiked, two more were marked dead, and the cluster was degraded for an hour. What went wrong, and what settings change?
<details><summary>Answer</summary>Rebalancing triggered on a short outage, and the data movement overloaded the remaining nodes, causing a cascade. Use a much longer dead-node timeout before moving data (replicas already serve reads and writes during a brief absence), throttle movement, gate large moves on an operator, and use maintenance mode for planned restarts.</details>

**20.** Code — Write `rendezvous(key, nodes)` returning the node with the highest `hash(key + node)` score.
<details><summary>Answer</summary>

```js
import { createHash } from 'node:crypto';
const score = (s) => createHash('md5').update(s).digest().readUInt32BE(0);
function rendezvous(key, nodes) {
  let best = null, bestScore = -1;
  for (const node of nodes) {
    const sc = score(`${key}|${node}`);
    if (sc > bestScore) { best = node; bestScore = sc; }
  }
  return best;
}
const k = 'user:7', before = rendezvous(k, ['a', 'b', 'c']);
console.log(before === rendezvous(k, ['a', 'b', 'c', 'd']) || rendezvous(k, ['a', 'b', 'c', 'd']) === 'd'); // true
```
Adding a node only moves the keys for which the new node scores highest.</details>

**21.** Code — Write `planRebalance(owners, nodes)` that, given an array of slot owners and the full list of nodes (including new empty ones), returns the minimum list of `{ slot, from, to }` moves that evens ownership to within one slot.
<details><summary>Answer</summary>

```js
function planRebalance(owners, nodes) {
  const target = Math.floor(owners.length / nodes.length), extra = owners.length % nodes.length;
  const quota = new Map(nodes.map((n, i) => [n, target + (i < extra ? 1 : 0)]));
  const count = new Map(nodes.map((n) => [n, 0]));
  owners.forEach((o) => count.set(o, (count.get(o) ?? 0) + 1));
  const moves = [];
  owners.forEach((from, slot) => {
    if (count.get(from) <= quota.get(from)) return;       // this owner isn't over quota
    const to = nodes.find((n) => count.get(n) < quota.get(n));
    if (!to) return;
    count.set(from, count.get(from) - 1); count.set(to, count.get(to) + 1);
    moves.push({ slot, from, to });
  });
  return moves;
}
const owners = Array.from({ length: 12 }, (_, i) => ['a', 'b', 'c'][i % 3]);
console.log(planRebalance(owners, ['a', 'b', 'c', 'd']).length); // 3
```
</details>

**22.** Code — Write `scatterGather(partitions, query, { timeoutMs })` that queries every partition in parallel and returns `{ results, missing }`, where partitions that don't answer within the timeout are listed in `missing` instead of failing the whole query.
<details><summary>Answer</summary>

```js
async function scatterGather(partitions, query, { timeoutMs = 200 } = {}) {
  const timeout = (ms) => new Promise((_, rej) => setTimeout(() => rej(new Error('timeout')), ms));
  const settled = await Promise.allSettled(
    partitions.map((p) => Promise.race([p.query(query), timeout(timeoutMs)])));
  const results = [], missing = [];
  settled.forEach((s, i) => (s.status === 'fulfilled' ? results.push(...s.value) : missing.push(partitions[i].name)));
  return { results, missing }; // caller decides whether partial results are acceptable
}
const p = (name, ms, rows) => ({ name, query: () => new Promise((r) => setTimeout(() => r(rows), ms)) });
console.log(await scatterGather([p('p1', 10, [1]), p('p2', 500, [2]), p('p3', 20, [3])], 'q', { timeoutMs: 100 }));
// { results: [ 1, 3 ], missing: [ 'p2' ] }
```
</details>

**23.** Scenario — A multi-tenant SaaS on one PostgreSQL instance is approaching its write ceiling. 80% of load comes from 5% of tenants, and one tenant alone is 15%. Propose a partitioning plan.
<details><summary>Answer</summary>Partition by `tenant_id` (it's in every query and keeps joins local), using a sharding layer like Citus or an application-level tenant directory. Put many small tenants per shard; give the largest tenants dedicated shards; and make the directory able to sub-partition the 15% tenant by `(tenant_id, project_id)` if it outgrows one shard. Move tenants online with snapshot-plus-CDC, starting with the largest. Cross-tenant reporting moves to the warehouse.</details>

**24.** Scenario — You're designing a distributed cache for session data across 30 nodes that are added and removed by autoscaling several times a day. Fixed slots or a consistent-hashing ring?
<details><summary>Answer</summary>A consistent-hashing ring with virtual nodes (or rendezvous hashing at this size): nodes change automatically and often, nobody needs to place specific keys, and each change should remap only ~1/N of keys without an operator updating a slot table. Sessions should also tolerate a miss (re-authenticate, or fall back to a durable session store) since even minimal remapping invalidates some entries.</details>

---

## Chapter Summary

- Partitioning splits data so each record lives on one partition; it's needed when size or write rate
  exceeds one node, and it should come after reversible options are exhausted.
- Range partitioning keeps keys sorted for range scans but turns sequential keys into a moving hot spot;
  hash partitioning spreads load evenly but gives up range scans.
- `hash % N` remaps N/(N+1) of keys on resize; consistent hashing moves about 1/(N+1), and virtual nodes
  make its load even and spread a failed node's keys across all survivors.
- Many more partitions than nodes — fixed slots or virtual nodes — make rebalancing incremental; moves
  follow snapshot-then-catch-up, throttled, and large moves shouldn't be triggered by blips.
- Local secondary indexes keep writes cheap but scatter reads; global ones make reads targeted but writes
  costly and usually stale; lookup tables you maintain are the common middle path.
- Choose a high-cardinality, evenly loaded, query-local, time-stable partition key — usually the owning
  entity — and plan for the tenant that outgrows any one partition.

---

## Practice Exercises

1. Extend §14.4's ring to report the standard deviation of load for 3–50 nodes at 10, 100, and 500
   virtual nodes, and find the point of diminishing returns.
2. Implement jump consistent hash and compare its balance and movement with the ring when adding a node
   at the end and when removing one from the middle.
3. Read the Redis Cluster specification's sections on hash slots, `MOVED`, and `ASK` redirections, and map
   them onto §14.10's migration states.
4. Take a database you know and list every query that doesn't include the column you'd choose as a
   partition key; estimate how many would become scatter-gather.
5. Simulate a Zipf-distributed tenant workload over 16 shards partitioned by `tenant_id`, and measure the
   load of the hottest shard before and after giving the top three tenants dedicated shards.

---

## Mini Project: An Online Resharding Tool

Build a small sharding layer over several PostgreSQL databases (or in-memory stores) that routes by
tenant, and a tool that moves a tenant from one shard to another with no downtime and no lost writes.

- A versioned tenant directory; routers cache it and refresh on version change or a "moved" reply.
- Online move: snapshot the tenant's rows, stream changes made during the copy (via a change table or
  logical replication), briefly block the tenant's writes for cut-over, then switch the directory.
- A verification step that compares row counts and checksums between source and destination before
  cut-over.
- A load generator writing continuously to the moving tenant; the test asserts zero lost or duplicated
  writes and reports the cut-over pause.
- Rollback: the move can be abandoned at any stage before cut-over with no effect on the tenant.

---

**Next:** [Chapter 15 — Transactions and Isolation →](15-transactions-isolation.md)
