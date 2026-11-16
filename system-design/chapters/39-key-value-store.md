# Chapter 39 — Design a Distributed Key-Value Store

## Learning Objectives

- Scope a highly available, partition-tolerant key-value store and size a cluster for it
- Combine consistent hashing, replication, and tunable quorums into one request path
- Detect and resolve conflicting writes with version vectors and application-level merges
- Keep writes available through failures with sloppy quorums and hinted handoff, and repair replicas with read repair and Merkle-tree anti-entropy
- Manage membership, failure detection, and storage-engine behaviour, and know what changes at 10×

---

## 39.1 Requirements and Scope

This case study assembles Parts III and IV into one system: a **Dynamo-style** key-value store, of the kind Amazon described
for shopping carts and that Cassandra, Riak, ScyllaDB, and DynamoDB's internals draw on. The defining requirement is
**availability for writes** — a customer must always be able to add an item to their cart, even during failures and network
partitions — at the cost of strong consistency.

**Functional requirements**

- `put(key, value, context)` and `get(key) → (values, context)`; values up to ~1 MB; keys are opaque strings.
- Per-request tunable consistency (how many replicas must respond).
- Add and remove nodes without downtime.

**Out of scope**: range queries, secondary indexes, multi-key transactions, a query language.

**Non-functional requirements**

| Requirement | Target |
|---|---|
| Availability | Writes accepted as long as any W replicas (or stand-ins) are reachable — "always writable" |
| Latency | p99 < 10 ms for gets and puts within a region |
| Durability | An acknowledged write survives the loss of any N − 1 replicas |
| Consistency | Eventual by default; read-your-writes achievable with quorum settings; conflicts never silently lost |
| Scale | 1 M operations/s, 100 TB of data, hundreds of nodes, linear scaling by adding nodes |
| Operations | No single point of failure, no leader, no special nodes |

The last row is the philosophical core of the design: every node runs the same code and can coordinate any request.

---

## 39.2 Estimation

```js
// Sizing a key-value cluster: whichever of storage or replica operations needs more nodes wins
const readsPerSec = 700_000, writesPerSec = 300_000, n = 3, r = 2;
const dataTB = 100, nodeDiskTB = 4, maxFill = 0.5, opsPerNode = 20_000;
const replicaOps = readsPerSec * r + writesPerSec * n;                 // every op touches R or N replicas
const forOps = Math.ceil(replicaOps / opsPerNode);
const forStorage = Math.ceil((dataTB * n) / (nodeDiskTB * maxFill));   // LSM compaction needs free space
console.log(`replica ops ${(replicaOps / 1e6).toFixed(1)} M/s → ${forOps} nodes | storage ${dataTB * n} TB → ${forStorage} nodes | provision ${Math.max(forOps, forStorage)}+`);
// replica ops 2.3 M/s → 115 nodes | storage 300 TB → 150 nodes | provision 150+
```

Two things fall out:

- **Replication multiplies work.** One million client operations become 2.3 million replica operations, because each read
  touches R replicas and each write N.
- **Storage, not throughput, sizes this cluster.** With 3× replication and disks kept half full for compaction headroom
  (Chapter 10), 100 TB of data needs about 150 nodes — before adding N+1 capacity for a zone failure (Chapter 4). A workload
  with smaller data and more traffic would flip the conclusion; compute both.

---

## 39.3 API and Data Model

```text
get(key, { consistency: 'one' | 'quorum' | 'all' })
  → { values: [v1, v2?…], context }        ← more than one value means concurrent writes (siblings)
put(key, value, { context, consistency })
  → { context }                             ← the context returned by the get being updated
```

The **context** is an opaque version vector (Chapter 22) describing which versions the client had seen. Passing it back on `put`
tells the store that the new value supersedes those versions; without it, the store can't distinguish "an update" from "a
concurrent write", and conflicts would be lost. Clients treat the context as opaque.

Each node stores, per key, one or more **versions**, each a value plus its version vector, in a local LSM-based storage engine
(Chapter 10) — chosen because the workload is write-heavy and key-addressed.

---

## 39.4 High-Level Design

```text
client ─▶ any node (coordinator for this request)
            │ 1. hash key → preference list on the ring (Ch 14)
            │ 2. send to first N healthy nodes; wait for W (put) or R (get) responses (Ch 13)
            ▼
 ┌────── ring of nodes, each with many virtual nodes ───────┐
 │  n1 … n150: storage engine (LSM) + hinted-handoff queue  │
 │  gossip: membership + failure suspicion (phi accrual)    │
 │  anti-entropy: Merkle-tree comparison between replicas   │
 └──────────────────────────────────────────────────────────┘
```

Clients either talk to a load balancer that forwards to any node, or use a **partition-aware client** that knows the ring and
sends each request directly to a node in the key's preference list, saving a hop (Chapter 14). Nodes learn membership and the
ring by **gossip**: each node periodically exchanges its view with a few random peers, so changes spread through the cluster in
O(log N) rounds without any central coordinator.

---

## 39.5 Deep Dive: Partitioning and Replication

Keys are placed with **consistent hashing and virtual nodes** (Chapter 14): each node owns many small arcs of the ring, so load is
even and adding a node moves only ~1/N of the data, from many existing nodes rather than one. A key's **preference list** is the
list of distinct physical nodes found walking clockwise from its hash; its first N entries are its **home replicas**.

Placement must respect failure domains: the N replicas of a key should be in different racks and availability zones (Chapter 4),
so the walk skips nodes in a zone already represented. Otherwise three "replicas" may all disappear with one zone.

Replication is **leaderless** (Chapter 13): the coordinator writes to all N replicas in parallel and waits for W acknowledgements;
reads ask replicas and wait for R. The familiar trade-offs apply:

| Setting (N = 3) | Write availability | Read latency | Freshness |
|---|---|---|---|
| W = 1, R = 1 | Highest | Lowest | Eventually consistent |
| W = 2, R = 2 (quorum) | Survives one replica down | Waits for 2 | Overlapping quorums — reads usually see the latest acknowledged write |
| W = 3, R = 1 | Fails if any replica is down | Lowest | Fresh reads, fragile writes |

For a shopping cart, the default is W = 2, R = 2: survive a replica failure for both reads and writes, and see your own recent
writes in the common case — while remembering Chapter 13's caveat that quorums alone aren't linearizable.

---

## 39.6 Deep Dive: Versions and Conflicts

In a leaderless store, two clients can update the same key concurrently through different coordinators — or one client's two
devices can write while partitioned from each other. The store must notice, and must not silently discard either update.

**Version vectors** make this possible (Chapter 22). Each write increments the coordinator's entry in the vector it was given (the
client's context). Comparing two versions' vectors tells whether one descends from the other (keep only the newer) or they're
concurrent (keep both). Concurrent versions are returned to the reader as **siblings**, with a context that merges both vectors;
the application **reconciles** them and writes the result back, which descends from both and replaces them.

| Strategy | What happens to concurrent writes | Fits |
|---|---|---|
| **Last write wins** (timestamp) | One is silently discarded | Caches, "last seen" values — loss is acceptable |
| **Siblings + application merge** | Both kept; the app merges (e.g. union of cart items) | Carts, sets, documents with domain-specific merges |
| **CRDTs** (Chapter 46) | Merged automatically and deterministically | Counters, sets, maps, text — when the data type fits |

The shopping-cart example is the classic illustration and has a known quirk: merging carts by union can **resurrect a deleted
item** (one sibling removed the mug, the other still has it). The fix is to represent deletions explicitly — an observed-remove set
CRDT that tracks additions and removals with tags — rather than storing only the current contents. Choosing the merge is a product
decision as much as an engineering one.

Two mechanisms keep replicas converging after the fact:

- **Read repair**: when a quorum read sees replicas with older versions, the coordinator writes the reconciled result back to them.
  Frequently read keys stay consistent for free.
- **Anti-entropy** (§39.7): a background process that finds and repairs differences for keys that aren't read often.

---

## 39.7 Deep Dive: Failures, Hinted Handoff, and Anti-Entropy

**Sloppy quorums and hinted handoff.** If one of a key's home replicas is down, a strict quorum would still need W of the remaining
home replicas — and with two down, writes would fail. To stay "always writable", a **sloppy quorum** sends the write to the next
healthy node in the preference list instead, which stores it with a **hint** naming the intended home. When the home node recovers,
the stand-in **hands off** the write and deletes its hint. The cost (Chapter 13): during the failure, a read quorum of home replicas
may not include the stand-in, so W + R > N no longer guarantees overlap. Availability is bought with a weaker consistency guarantee.

**Failure detection and membership.** Nodes are suspected via gossip-propagated heartbeats with an adaptive phi-accrual detector
(Chapter 19). Two separate notions keep the system stable:

- **Temporary unavailability** (a node is unreachable): route around it with sloppy quorums and hints; *don't* move its data.
- **Permanent membership change** (a node is added or decommissioned): an explicit operator or automation action that rebalances
  ring ownership and streams data (Chapter 14's online move), throttled.

Conflating the two — rebalancing whenever a node is briefly unreachable — causes the data-movement cascades from Chapter 14.

**Anti-entropy with Merkle trees.** Hints can be lost (the stand-in fails too) and read repair only fixes keys that are read. A
background process compares replicas pairwise and repairs differences. Comparing every key would mean exchanging terabytes, so each
replica maintains a **Merkle tree** over its key range: leaves are hashes of buckets of keys, parents are hashes of their children.
Two replicas compare roots; if they match, everything matches; if not, they descend only into subtrees whose hashes differ:

```js
// Merkle-tree anti-entropy: two replicas find which key ranges differ by comparing hashes top-down,
// exchanging a handful of hashes instead of every key.
import { createHash } from 'node:crypto';
const h = (s) => createHash('sha256').update(s).digest('hex').slice(0, 16);
const bucketOf = (key, buckets) => parseInt(h(key).slice(0, 8), 16) % buckets;

function buildTree(data, buckets = 16) {
  const leaves = Array.from({ length: buckets }, () => []);
  for (const [k, v] of [...data].sort()) leaves[bucketOf(k, buckets)].push(`${k}=${v}`);
  let level = leaves.map((items) => h(items.join('|')));
  const levels = [level];
  while (level.length > 1) {
    level = Array.from({ length: level.length / 2 }, (_, i) => h(level[2 * i] + level[2 * i + 1]));
    levels.unshift(level);                      // levels[0] is the root
  }
  return levels;
}

function diffBuckets(a, b) {
  let compared = 0;
  const walk = (depth, i) => {
    compared++;
    if (a[depth][i] === b[depth][i]) return [];  // identical subtree: skip everything below
    if (depth === a.length - 1) return [i];      // a differing leaf bucket
    return [...walk(depth + 1, 2 * i), ...walk(depth + 1, 2 * i + 1)];
  };
  return { buckets: walk(0, 0), hashesCompared: compared };
}

const replicaA = new Map(Array.from({ length: 10_000 }, (_, i) => [`user:${i}`, `v1`]));
const replicaB = new Map(replicaA);
replicaB.set('user:42', 'v2');                   // one missed update…
replicaB.delete('user:9001');                    // …and one missed delete
const { buckets, hashesCompared } = diffBuckets(buildTree(replicaA), buildTree(replicaB));
console.log(`differing buckets: ${buckets} | hashes compared: ${hashesCompared} of 31 | keys to exchange: ~${Math.round((10_000 / 16) * buckets.length)}`);
// differing buckets: 4,11 | hashes compared: 15 of 31 | keys to exchange: ~1250
```

With 16 leaf buckets for illustration, two differences are located by comparing 15 hashes, and only two buckets' worth of keys need
exchanging. Real stores use many thousands of buckets per range, so the keys exchanged are a tiny fraction of the data. Trees must be
rebuilt or updated as data changes, which costs I/O; stores typically build them per token range and schedule repairs to run within a
period shorter than the tombstone grace period (Chapter 10) — otherwise a replica that missed a delete can "resurrect" the data after
the tombstone is compacted away elsewhere.

> 💼 **Interview** — "Design a key-value store" usually expects the full Dynamo toolkit, each with its trade-off: consistent hashing
> with virtual nodes; N/W/R quorums; version vectors and siblings (or LWW, knowing it loses data); sloppy quorums with hinted handoff;
> read repair and Merkle-tree anti-entropy; gossip and phi-accrual failure detection. The strongest answers explain *why each piece
> exists* — which failure it handles.

---

## 39.8 Failure Modes, Bottlenecks, and 10×

| Failure | Effect | Mitigation |
|---|---|---|
| One node down | Its key ranges served by remaining replicas | Sloppy quorums + hints; replace if permanent |
| A zone down | A third of replicas gone | Zone-aware placement keeps ≥ 2 replicas of every key; N+1 capacity per zone |
| Network partition | Both sides keep accepting writes | Siblings on heal; application merge or CRDTs |
| Hot key | One replica set overloaded | Client-side caching; key splitting at the application level (Chapter 12) |
| Compaction falling behind | Read amplification and disk usage climb | Throttle writes, add nodes, tune compaction strategy (Chapter 10) |
| Large partitions / values | Latency spikes, GC pressure | Enforce value size limits; chunk large values |
| Tombstone resurrection | Deleted data reappears | Repair every replica within the tombstone grace period |

**At 10×** — 10 M operations/s and a petabyte of data: the cluster grows to well over a thousand nodes, and its problems become
operational. Gossip traffic and membership convergence time grow with cluster size, so very large deployments split into several
clusters (by data domain or tenant) rather than one enormous ring. Repairs and bootstrapping new nodes must be throttled and
scheduled, because streaming terabytes competes with live traffic. Multi-region replication adds per-datacentre consistency levels
(`LOCAL_QUORUM`) so requests don't wait on remote regions (Chapter 32).

---

## 39.9 Common Mistakes and Best Practices

**Mistake: `hash % N` placement.** Adding a node remaps nearly every key; use consistent hashing with virtual nodes.

**Mistake: replicas in the same failure domain.** Three copies in one zone are one copy during a zone outage.

**Mistake: last-write-wins for data people edit concurrently.** Concurrent updates are silently discarded.

**Mistake: writes without the read context.** The store can't tell an update from a concurrent write, so conflicts are lost or siblings
pile up.

**Mistake: rebalancing on transient failure.** Moving data whenever a node is briefly unreachable cascades into cluster-wide overload.

**Mistake: never running anti-entropy.** Rarely read keys diverge silently, and missed deletes resurrect after tombstones expire.

**Mistake: treating quorum reads as linearizable.** Sloppy quorums and concurrent writes break that assumption; use consensus where it's
needed (Chapter 21).

**Best practice: separate "unavailable" from "removed".** Route around the first with hints; handle the second with explicit, throttled
rebalancing.

**Best practice: design the merge with the product.** Siblings force a decision about what concurrent edits mean — make it deliberately,
or use a CRDT that encodes it.

**Best practice: size for both storage and replica operations.** Whichever needs more nodes sets the cluster size.

---

## 39.10 Real-World Example: A Dynamo-Style Coordinator

This is the request path that ties the design together: preference lists from a hash ring with virtual nodes, sloppy quorums that skip
unreachable nodes and leave hints, version vectors that keep concurrent writes as siblings, read repair on every quorum read, and
hinted handoff when nodes recover. The demo uses five nodes with N = 3, W = 2, R = 2.

```js
// dynamo-coordinator.mjs — the request path of a leaderless key-value store: a preference list from
// a hash ring, sloppy quorums with hinted handoff, version vectors with siblings, and read repair.
import { createHash } from 'node:crypto';
const hash32 = (s) => createHash('md5').update(s).digest().readUInt32BE(0);

// Version vectors (Chapter 22)
const descends = (a, b) => Object.keys(b).every((k) => (a[k] ?? 0) >= b[k]);
const merge = (...vs) => vs.reduce((m, v) => { for (const [k, n] of Object.entries(v)) m[k] = Math.max(m[k] ?? 0, n); return m; }, {});

export class Cluster {
  constructor(names, { n = 3, w = 2, r = 2, vnodes = 16 } = {}) {
    Object.assign(this, { n, w, r });
    this.nodes = new Map(names.map((name) => [name, { up: true, data: new Map(), hints: [] }]));
    this.ring = names.flatMap((name) => Array.from({ length: vnodes }, (_, v) => ({ h: hash32(`${name}#${v}`), name })))
      .sort((a, b) => a.h - b.h);
  }
  // Distinct nodes clockwise from the key; the first N are its "home" replicas
  preferenceList(key) {
    const start = this.ring.findIndex((p) => p.h >= hash32(key));
    const names = [];
    for (let i = 0; names.length < this.nodes.size; i++) {
      const { name } = this.ring[((start < 0 ? 0 : start) + i) % this.ring.length];
      if (!names.includes(name)) names.push(name);
    }
    return names;
  }
  // Keep only versions not dominated by another: concurrent versions survive as siblings
  #reconcile(versions) {
    return versions.filter((v, i) => !versions.some((o, j) => j !== i && descends(o.clock, v.clock) && !descends(v.clock, o.clock)))
      .filter((v, i, arr) => arr.findIndex((o) => JSON.stringify(o.clock) === JSON.stringify(v.clock)) === i);
  }
  #store(nodeName, key, version) {
    const node = this.nodes.get(nodeName);
    node.data.set(key, this.#reconcile([...(node.data.get(key) ?? []), version]));
  }

  put(key, value, { context = {}, coordinator }) {
    const clock = merge(context, { [coordinator]: (context[coordinator] ?? 0) + 1 });
    const prefs = this.preferenceList(key), home = prefs.slice(0, this.n);
    const targets = prefs.filter((name) => this.nodes.get(name).up).slice(0, this.n); // sloppy: skip down nodes
    if (targets.length < this.w) throw new Error(`only ${targets.length} replicas reachable, need W=${this.w}`);
    const missing = home.filter((name) => !targets.includes(name));
    for (const name of targets) {
      this.#store(name, key, { value, clock });
      // A stand-in replica keeps a hint so it can hand the write to the home node later
      if (!home.includes(name)) this.nodes.get(name).hints.push({ for: missing.shift(), key, version: { value, clock } });
    }
    return clock;
  }

  get(key) {
    const replicas = this.preferenceList(key).slice(0, this.n).filter((name) => this.nodes.get(name).up);
    if (replicas.length < this.r) throw new Error(`only ${replicas.length} replicas reachable, need R=${this.r}`);
    const responses = replicas.slice(0, this.r).map((name) => ({ name, versions: this.nodes.get(name).data.get(key) ?? [] }));
    const result = this.#reconcile(responses.flatMap((x) => x.versions));
    for (const { name } of responses) for (const v of result) this.#store(name, key, v); // read repair
    return { values: result.map((v) => v.value), context: merge({}, ...result.map((v) => v.clock)) };
  }

  handoff() {                                       // run periodically: deliver hints to recovered nodes
    for (const node of this.nodes.values()) {
      node.hints = node.hints.filter((h) => {
        if (!this.nodes.get(h.for).up) return true;
        this.#store(h.for, h.key, h.version);
        return false;
      });
    }
  }
}

// ── Demo: N=3, W=2, R=2 on five nodes ────────────────────────────────────────────────
const c = new Cluster(['n1', 'n2', 'n3', 'n4', 'n5']);
const key = 'cart:ana', [home1] = c.preferenceList(key);
c.put(key, ['mug'], { coordinator: 'n1' });
c.nodes.get(home1).up = false;                    // a home replica fails…
const { context } = c.get(key);
c.put(key, ['mug', 'tea'], { context, coordinator: 'n2' });   // …the write still succeeds (sloppy quorum + hint)
// Two clients edit concurrently from the same context, via different coordinators
const ctx2 = c.get(key).context;
c.put(key, ['mug', 'tea', 'pot'], { context: ctx2, coordinator: 'n3' });
c.put(key, ['mug', 'tea', 'cup'], { context: ctx2, coordinator: 'n4' });
console.log('siblings:', JSON.stringify(c.get(key).values));
// siblings: [["mug","tea","pot"],["mug","tea","cup"]]
// The application merges (here: union of cart items) and writes back with the merged context
const { values, context: ctx3 } = c.get(key);
c.put(key, [...new Set(values.flat())], { context: ctx3, coordinator: 'n3' });
c.nodes.get(home1).up = true; c.handoff();
console.log('after merge + handoff:', JSON.stringify(c.get(key).values), '| hints left:', [...c.nodes.values()].reduce((s, n) => s + n.hints.length, 0));
// after merge + handoff: [["mug","tea","pot","cup"]] | hints left: 0
```

With a home replica down, writes still succeed on a stand-in that keeps a hint; two concurrent edits from the same context come back as
siblings instead of one overwriting the other; the application's merge descends from both and replaces them; and when the failed node
returns, handoff brings it up to date. A production version adds zone-aware placement, timeouts and parallel requests, persistent
hints, and the storage engine underneath.

The decision to defend is **returning siblings to the application instead of resolving conflicts inside the store**, which pushes
complexity onto every client — they must handle multiple values and write back a merge. Resolving internally with last-write-wins
would give clients a simple single-value API, and that's exactly what many stores offer by default. But the store can't know what
"merging" two shopping carts, two profile edits, or two counters means; LWW picks one by timestamp and silently drops the other, and in a
system designed to accept writes on both sides of a partition, concurrent writes aren't rare edge cases — they're the expected outcome of
every partition. Siblings keep the data and hand the decision to the only party that understands it. Where a data type has a known merge,
CRDTs move that merge back into the store safely — the next best thing to an API with no conflicts at all.

---

## 39.11 Questions

**1.** MCQ — Why does a Dynamo-style store use sloppy quorums? a) To improve consistency b) To keep writes available when home replicas are unreachable c) To reduce storage d) To elect a leader
<details><summary>Answer</summary>**b)** Writes go to stand-in nodes with hints, trading quorum overlap for availability.</details>

**2.** MCQ — What does the `context` returned by `get` contain? a) The value's timestamp b) The version vector(s) the client has seen, so a following `put` can supersede them c) A lock token d) The coordinator's address
<details><summary>Answer</summary>**b)** Without it the store can't distinguish an update from a concurrent write.</details>

**3.** MCQ — What is hinted handoff? a) Transferring leadership b) A stand-in replica forwarding writes it accepted on behalf of an unavailable home node once that node returns c) Moving data during rebalancing d) Client-side caching
<details><summary>Answer</summary>**b)** The hint records which node the write was meant for.</details>

**4.** MCQ — What do Merkle trees reduce in anti-entropy? a) Write latency b) The data exchanged to find differences between replicas c) The number of replicas d) Clock skew
<details><summary>Answer</summary>**b)** Matching subtree hashes let replicas skip identical ranges and exchange only differing buckets.</details>

**5.** MCQ — Why must anti-entropy repair complete within the tombstone grace period? a) To save disk b) Otherwise a replica that missed a delete can reintroduce the data after other replicas compact the tombstone away c) For performance d) Gossip requires it
<details><summary>Answer</summary>**b)** Once the tombstone is gone, the stale replica's old value looks like the newest version.</details>

**6.** MCQ — Which conflict strategy never silently discards a concurrent update? a) Last write wins b) Siblings with application merge (or CRDTs) c) Highest node ID wins d) First write wins
<details><summary>Answer</summary>**b)** Both versions are kept until merged.</details>

**7.** Short — Why should temporary unavailability and permanent membership changes be handled differently?
<details><summary>Answer</summary>Transient failures are common and short; moving data for them adds load and can cascade. Route around unavailable nodes with sloppy quorums and hints, and move data only on explicit, throttled membership changes.</details>

**8.** Short — Why can merging carts by union resurrect deleted items, and what fixes it?
<details><summary>Answer</summary>If one sibling removed an item and the other still has it, the union includes it again. Track removals explicitly — an observed-remove set CRDT with tags for additions and removals — so concurrent remove and keep can be resolved correctly.</details>

**9.** Short — What does gossip provide, and how fast does it spread information?
<details><summary>Answer</summary>Decentralised membership and failure-suspicion propagation: each node periodically exchanges state with a few random peers, so information reaches all N nodes in O(log N) rounds without a central coordinator.</details>

**10.** Short — Why does replication multiply the work the cluster must do?
<details><summary>Answer</summary>Each client read touches R replicas and each write N replicas, so total replica operations are reads × R + writes × N — often more than double the client rate.</details>

**11.** Short — How does read repair keep frequently read keys consistent?
<details><summary>Answer</summary>When a quorum read returns differing versions, the coordinator writes the reconciled result back to the stale replicas, so popular keys converge as a side effect of being read.</details>

**12.** Output — Using §39.2's model with 400,000 reads/s and 600,000 writes/s (N = 3, R = 2) at 20,000 replica ops per node, how many nodes do operations require?
<details><summary>Answer</summary>**130.** 400,000 × 2 + 600,000 × 3 = 2.6 M replica ops/s ÷ 20,000.</details>

**13.** Output — Using §39.10's `Cluster` (N = 3, W = 2), two of a key's three home replicas are down and the other two nodes are up. Does a `put` succeed?
<details><summary>Answer</summary>**Yes.** With five nodes, the sloppy quorum writes to the one remaining home replica plus two stand-ins (three up nodes ≥ W = 2), leaving hints for the two missing homes.</details>

**14.** Output — Same cluster: four of five nodes are down. Does a `put` succeed?
<details><summary>Answer</summary>**No — it throws.** Only one node is reachable, fewer than W = 2.</details>

**15.** Output — A Merkle tree has 1,024 leaf buckets over 10 million keys. Two replicas differ in keys that fall into 3 buckets. Roughly how many keys must be exchanged after the hash comparison?
<details><summary>Answer</summary>**~29,000.** About 9,766 keys per bucket × 3 — under 0.3% of the data, instead of all 10 million.</details>

**16.** Debug — After a partition healed, customers reported items they'd removed from their carts reappearing. The store keeps siblings, and the cart service merges siblings by taking the union of items. What's wrong?
<details><summary>Answer</summary>Union can't represent removals: a sibling that still contains the item reintroduces it. Model carts as an observed-remove set (tracking add and remove operations with unique tags) or store removal markers, so a concurrent remove beats a stale keep.</details>

**17.** Debug — Operators see rows deleted weeks ago reappear on some replicas. Repairs run monthly; the tombstone grace period is ten days. Cause and fix?
<details><summary>Answer</summary>A replica missed the delete; other replicas compacted the tombstone after ten days; the next repair saw the stale value as the only version and propagated it back. Run repairs more often than the grace period (or lengthen the grace period beyond the repair interval).</details>

**18.** Debug — A node went unreachable for 90 seconds during a network blip. The cluster began streaming its data to other nodes, overloaded several, and then more nodes were marked down. What should change?
<details><summary>Answer</summary>Transient unavailability was treated as a membership change. Use sloppy quorums and hinted handoff during short outages, require an explicit or long-timeout decision to remove a node, and throttle any data streaming.</details>

**19.** Debug — After a zone outage, some keys became completely unavailable for reads at `QUORUM`, though two-thirds of nodes were up. What's the likely placement problem?
<details><summary>Answer</summary>Replica placement wasn't zone-aware, so all (or two) of some keys' replicas were in the failed zone. Place each key's N replicas in different zones, skipping nodes in already-used zones when walking the ring.</details>

**20.** Code — Write `zoneAwarePreferenceList(ringOrder, zoneOf, n)` that walks nodes in ring order and picks the first `n` nodes in distinct zones.
<details><summary>Answer</summary>

```js
function zoneAwarePreferenceList(ringOrder, zoneOf, n) {
  const picked = [], zones = new Set();
  for (const node of ringOrder) {
    if (picked.includes(node) || zones.has(zoneOf[node])) continue; // one replica per zone
    picked.push(node); zones.add(zoneOf[node]);
    if (picked.length === n) break;
  }
  return picked;
}
console.log(zoneAwarePreferenceList(['a', 'b', 'c', 'd', 'e'], { a: 'z1', b: 'z1', c: 'z2', d: 'z2', e: 'z3' }, 3)); // [ 'a', 'c', 'e' ]
```
</details>

**21.** Code — Write `orsetMerge(a, b)` for an observed-remove set represented as `{ adds: Map(tag → item), removes: Set(tag) }`, returning the merged set and its visible items.
<details><summary>Answer</summary>

```js
function orsetMerge(a, b) {
  const adds = new Map([...a.adds, ...b.adds]);
  const removes = new Set([...a.removes, ...b.removes]);
  const items = [...new Set([...adds].filter(([tag]) => !removes.has(tag)).map(([, item]) => item))];
  return { adds, removes, items };     // an item is visible if any of its add-tags hasn't been removed
}
const base = { adds: new Map([['t1', 'mug']]), removes: new Set() };
const left = { adds: new Map(base.adds), removes: new Set(['t1']) };                       // removed the mug
const right = { adds: new Map([...base.adds, ['t2', 'tea']]), removes: new Set() };        // added tea, still had mug
console.log(orsetMerge(left, right).items); // [ 'tea' ]
```
The removal of tag `t1` wins against the stale copy; a *new* add of the mug would get a new tag and survive.</details>

**22.** Code — Write `gossipRounds(n, fanout)` estimating how many rounds of push gossip are needed for a rumour to reach all `n` nodes, simulating with a seeded PRNG.
<details><summary>Answer</summary>

```js
function gossipRounds(n, fanout = 3, seed = 1) {
  let s = seed; const rand = () => ((s = (s * 16807) % 2147483647) - 1) / 2147483646;
  const informed = new Set([0]);
  let rounds = 0;
  while (informed.size < n) {
    rounds++;
    for (const node of [...informed]) {
      for (let k = 0; k < fanout; k++) informed.add(Math.floor(rand() * n)); // tell random peers
    }
  }
  return rounds;
}
console.log(gossipRounds(150), gossipRounds(1_500)); // 7 9
```
Rounds grow roughly with log n, which is why gossip scales to large clusters.</details>

**23.** Scenario — A team wants to use this design for a bank's account balances because it's "always available". What do you advise?
<details><summary>Answer</summary>Don't: balances need linearizable conditional updates (no overdrafts, no lost debits), and a store that accepts writes on both sides of a partition and resolves conflicts afterwards can't guarantee them. Use a consensus-based or single-leader store with transactions (Chapters 15, 21, 43). The Dynamo design suits data where availability matters more than immediate consistency and conflicts can be merged — carts, preferences, session state.</details>

**24.** Scenario — You must add 30 nodes to a 150-node cluster during business hours without hurting p99 latency. Plan it.
<details><summary>Answer</summary>Add nodes gradually (a few at a time), each taking ownership of a share of virtual nodes from many existing nodes so the transfer is spread out; stream data with tight throttling and monitor p99, compaction backlog, and disk; let new nodes serve reads only after their ranges are fully streamed (or treat them as joining with hints); and pause if latency degrades. Run a repair on transferred ranges afterwards to fix anything missed during streaming.</details>

---

## Chapter Summary

- A Dynamo-style store trades strong consistency for write availability: every node can coordinate any request, and there's no leader
  to fail.
- Consistent hashing with virtual nodes places data evenly and moves only ~1/N of it on membership changes; replicas must span failure
  domains.
- Leaderless N/W/R quorums tune availability, latency, and freshness per request; sloppy quorums with hinted handoff keep writes available
  through failures at the cost of quorum overlap.
- Version vectors detect concurrent writes, which become siblings for the application (or a CRDT) to merge — last-write-wins silently
  loses them.
- Read repair fixes hot keys, Merkle-tree anti-entropy fixes the rest within the tombstone grace period, and gossip with adaptive failure
  detection maintains membership.
- Size the cluster for both replica operations and storage with compaction headroom, separate transient failure from membership change,
  and at 10× split into multiple clusters and schedule repairs carefully.

---

## Practice Exercises

1. Extend §39.10's coordinator with zone-aware preference lists and test that a zone failure never leaves a key with fewer than two
   replicas.
2. Persist hints to disk and show that a stand-in's crash before handoff is eventually healed by anti-entropy.
3. Read the Amazon Dynamo paper and list each technique it describes alongside the problem it solves and the chapter of this book that
   covers it.
4. Implement a Merkle tree over 1,024 buckets with incremental updates on write, and measure repair traffic between replicas after injecting
   random differences.
5. Run a three-node Cassandra or ScyllaDB cluster, write with `ONE` while partitioning a node, and observe hints, read repair, and the
   results of a repair after healing.

---

## Mini Project: A Mini Dynamo

Build a small leaderless key-value store as several processes (or one process with simulated nodes over a fault-injecting network from
Chapter 19's project) that survives failures and partitions without losing writes.

- Ring with virtual nodes and zone-aware preference lists; configurable N, W, R per request.
- Version vectors, siblings, and an OR-set CRDT example application (a shopping cart).
- Sloppy quorums with persistent hints and handoff; read repair; background Merkle-tree anti-entropy.
- Gossip-based membership with a phi-accrual detector, and explicit, throttled node addition and removal.
- A test suite under partitions and crashes asserting no acknowledged write is lost and all replicas converge after healing.

---

**Next:** [Chapter 40 — Design a Video Streaming Platform →](40-video-streaming.md)
