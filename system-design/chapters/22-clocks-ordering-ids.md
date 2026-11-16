# Chapter 22 — Clocks, Ordering, and Unique IDs

## Learning Objectives

- Explain what physical clocks can and can't tell you about event order, and how bounded-uncertainty clocks work
- Use Lamport clocks to order events consistently with causality
- Use version vectors to detect concurrent updates that need reconciling
- Explain hybrid logical clocks and why databases use them for timestamps
- Choose a unique-ID scheme — auto-increment, UUIDv4, UUIDv7, Snowflake, range allocation — for a workload

---

## 22.1 Why Order Matters

Distributed systems constantly need to answer "which came first?":

- **Conflict resolution** — two replicas accepted writes to the same key; which survives (Chapter 13)?
- **Causality** — a reply must not be shown before the message it answers (Chapter 20).
- **Snapshots and MVCC** — a transaction reads "as of" a timestamp; which writes are visible (Chapter 15)?
- **Sort keys and IDs** — feeds sorted newest first, time-bucketed partitions, primary keys that keep
  B-tree inserts sequential (Chapters 10, 12).

On one machine, a single clock and a single memory make the answer obvious. Across machines, Chapter 19
established that wall clocks disagree and jump. This chapter is about the tools that give dependable order
anyway — each answering a slightly different question:

| Tool | Answers | Doesn't answer |
|---|---|---|
| Physical clocks (NTP-synced) | Roughly when, in human time | Order of close events on different machines |
| Physical clocks with bounded uncertainty (TrueTime) | Definite order, if you wait out the uncertainty | Anything, cheaply, without special hardware |
| Lamport clocks | A total order consistent with causality | Whether two events were concurrent |
| Version vectors | Whether two versions are ordered or concurrent | A total order |
| Hybrid logical clocks | A causality-respecting order close to wall time | Concurrency detection |

---

## 22.2 Physical Time, Honestly

Machines synchronised by NTP typically agree within a few milliseconds on a good network, but the bound
isn't guaranteed and a node can't tell when it's violated. Two practical consequences follow from
Chapter 19: **use the monotonic clock for durations**, and **never use wall-clock timestamps alone to order
events that happen within the skew window on different machines**.

Some details matter in production:

- **Leap seconds** have historically been inserted into UTC, producing a repeated or skipped second that has
  crashed systems. Large providers **smear** leap seconds — slow the clock slightly over many hours instead of
  stepping — and the international timekeeping community has decided to stop adding them by 2035. If you
  compare timestamps across providers, know whether each smears.
- **Time zones are for display.** Store and compute in UTC (or as Unix time); convert at the edge. Daylight
  saving transitions create local times that occur twice or never.
- **Virtual machines and containers** inherit the host's clock behaviour; live migration and suspension can
  produce sudden jumps.

### Clocks with bounded uncertainty

Google's **Spanner** takes a different approach: its TrueTime API returns not a time but an *interval*,
`[earliest, latest]`, guaranteed (with GPS receivers and atomic clocks in each data centre) to contain the true
time — typically a few milliseconds wide. To make a transaction's timestamp unambiguous, Spanner assigns it a
timestamp and then **waits until `earliest` has passed that timestamp** ("commit wait") before making the commit
visible. After the wait, every node's clock interval is past the timestamp, so any later transaction anywhere
gets a larger one. The result is externally consistent (linearizable) transactions across continents, paid for
with the uncertainty interval as extra commit latency — which is why the hardware that keeps the interval small
matters. Cloud providers now expose similar precision time services; systems such as CockroachDB use a
configurable maximum clock offset and handle uncertainty with read restarts instead of waiting.

---

## 22.3 Lamport Clocks

A **Lamport clock** is a single counter per node, maintained by two rules:

1. Before each local event or message send, increment the counter; stamp outgoing messages with it.
2. On receiving a message, set the counter to `max(local, received) + 1`.

```js
// Lamport clock: a counter that only moves forward, bumped past anything you hear about.
class LamportClock {
  #t = 0;
  tick() { return ++this.#t; }                              // local event or send
  receive(remote) { this.#t = Math.max(this.#t, remote) + 1; return this.#t; }
}
const a = new LamportClock(), b = new LamportClock();
const sent = a.tick();            // A: "question" at 1
b.tick(); b.tick();               // B: two unrelated local events (1, 2)
const got = b.receive(sent);      // B receives the question → max(2, 1) + 1 = 3
const reply = b.tick();           // B: "reply" at 4 — guaranteed greater than the question's 1
console.log(sent, got, reply);    // 1 3 4
```

The guarantee: **if event X happened before event Y** (same node in order, or X is a send and Y its receipt, or a
chain of these), **then L(X) < L(Y)**. Ties between nodes are broken by node ID, giving a **total order** that
every node computes identically and that never contradicts causality — useful for ordering a replicated log, for
deterministic conflict resolution, and for mutual-exclusion algorithms.

The limitation is the converse: **L(X) < L(Y) does not mean X happened before Y.** Two concurrent events get
different numbers, and the order between them is arbitrary. A Lamport clock can't tell you "these two updates
were concurrent and one will be lost if you pick a winner" — and that is exactly what conflict resolution needs to
know.

---

## 22.4 Version Vectors: Detecting Concurrency

A **version vector** (or vector clock, when tracking events rather than data versions) keeps one counter **per
replica**. When replica *r* modifies a value, it increments its own entry; when versions are merged, each entry
takes the maximum. Comparing two vectors tells you their relationship precisely:

- If every entry of A is ≥ the matching entry of B (and at least one is greater), **A descends from B** — A
  supersedes B, and B can be discarded.
- If each vector has some entry greater than the other's, they're **concurrent** — neither saw the other, and
  both must be kept or merged.

```js
// Version vectors: one counter per replica. Comparing two tells you whether one version
// descends from the other — or whether they're concurrent and must be reconciled.
function compare(a, b) {
  let aBigger = false, bBigger = false;
  for (const node of new Set([...Object.keys(a), ...Object.keys(b)])) {
    const x = a[node] ?? 0, y = b[node] ?? 0;
    if (x > y) aBigger = true;
    if (y > x) bBigger = true;
  }
  if (aBigger && bBigger) return 'concurrent';
  if (aBigger) return 'a after b';
  if (bBigger) return 'b after a';
  return 'equal';
}
const merge = (a, b) => Object.fromEntries([...new Set([...Object.keys(a), ...Object.keys(b)])]
  .map((n) => [n, Math.max(a[n] ?? 0, b[n] ?? 0)]));

const base = { r1: 2, r2: 1 };                  // both replicas have seen this version
const onR1 = { ...base, r1: 3 };                // r1 accepts an edit
const onR2 = { ...base, r2: 2 };                // r2 accepts a different edit, unaware of r1's
console.log(compare(onR1, base), '|', compare(onR1, onR2), '|', JSON.stringify(merge(onR1, onR2)));
// a after b | concurrent | {"r1":3,"r2":2}
```

This is how Dynamo-style stores avoid silently losing concurrent writes: instead of last-write-wins, they keep
both concurrent versions as **siblings** and return them to the application (or a merge function) to reconcile —
union two shopping carts, or ask the user. The merged result gets the merged vector, plus an increment, so it
descends from both.

Version vectors have costs. Their size grows with the number of replicas that have written a key, so systems
track replicas (a handful per key) rather than clients (millions). **Dotted version vectors** refine the scheme to
avoid false concurrency when many clients write through the same replica. And concurrency detection only moves
the problem: something still has to merge siblings, which is why CRDTs (Chapter 46) — data types whose merges are
automatic — are often the better answer.

> 💼 **Interview** — "How do you detect conflicting writes in a leaderless store?" Answer: version vectors per
> key, compared on write and read; if neither descends from the other they're concurrent, so keep siblings and
> resolve them by application merge or CRDT. Then contrast with last-write-wins, which is simpler and loses data.

---

## 22.5 Hybrid Logical Clocks

Lamport clocks respect causality but drift arbitrarily far from wall time, so their values are meaningless to
humans and useless for "show me data as of 10:30". Physical timestamps are meaningful but can contradict
causality. A **hybrid logical clock (HLC)** combines them: each timestamp is a pair (`l`, `c`) where `l` tracks
the largest physical time seen and `c` is a logical counter that breaks ties when `l` doesn't advance.

```js
// Hybrid logical clock: physical time when it's moving, a logical counter when it isn't —
// so timestamps stay close to wall time but never go backwards and always respect causality.
class HybridClock {
  #l = 0; #c = 0;
  constructor(physical = Date.now) { this.physical = physical; }
  now() {                                         // local event or send
    const pt = this.physical();
    if (pt > this.#l) { this.#l = pt; this.#c = 0; } else this.#c++;
    return { l: this.#l, c: this.#c };
  }
  receive({ l: ml, c: mc }) {
    const pt = this.physical(), prev = this.#l;
    this.#l = Math.max(prev, ml, pt);
    if (this.#l === prev && this.#l === ml) this.#c = Math.max(this.#c, mc) + 1;
    else if (this.#l === prev) this.#c++;
    else if (this.#l === ml) this.#c = mc + 1;
    else this.#c = 0;
    return { l: this.#l, c: this.#c };
  }
}
const fmt = ({ l, c }) => `${l}.${c}`;
let tA = 1_000, tB = 900;                        // B's clock is 100 ms behind A's
const A = new HybridClock(() => tA), B = new HybridClock(() => tB);
const m = A.now();                               // A sends at 1000.0
tB += 5;                                         // a moment later on B (its clock reads 905)
const r = B.receive(m);                          // B: 905 by its own clock — the HLC says 1000.1
const e = B.now();                               // B's next event still sorts after the message
console.log(fmt(m), fmt(r), fmt(e));             // 1000.0 1000.1 1000.2
```

B's own clock reads 905 ms, but the message it received carries 1000; a plain physical timestamp would order B's
receipt *before* A's send. The HLC instead stamps the receipt 1000.1 and B's next event 1000.2 — after the
message, as causality requires — while staying within the clock skew of real time. Once B's physical clock passes
1000, `l` follows it again and `c` resets.

HLC timestamps fit in 64 bits (milliseconds plus a small counter), compare as plain numbers, and need no extra
coordination, which is why CockroachDB, YugabyteDB, MongoDB (for cluster time), and others use them for
transaction and version timestamps. Their guarantee is the Lamport guarantee plus "close to physical time" — not
concurrency detection, and not linearizability by themselves.

---

## 22.6 Unique IDs

Almost every entity needs an identifier, and the choice has more consequences than it seems: index performance
(Chapter 10), shard routing (Chapter 14), information leakage, and whether ID generation becomes a dependency
that can fail.

| Scheme | Size | Coordination | Time-ordered | Leaks | Notes |
|---|---|---|---|---|---|
| **Auto-increment** (database sequence) | 8 bytes | The database | Yes | Volume and growth rate (`/orders/1052`) | Simple; a single-primary bottleneck; reused after lossy failover (Chapter 13) |
| **UUIDv4** (random) | 16 bytes | None | No | Nothing | Random inserts scatter across B-trees (Chapter 10) |
| **UUIDv7** (RFC 9562) | 16 bytes | None | Yes (ms) | Creation time | Best general default for new systems |
| **ULID** | 16 bytes | None | Yes (ms) | Creation time | Like UUIDv7, with a sortable Base32 text form |
| **Snowflake-style** | 8 bytes | Unique worker IDs | Yes (ms) | Creation time, worker | Fits in a `bigint`; high throughput per worker |
| **Range allocation** ("ticket server", segments) | 8 bytes | A central allocator, rarely | Roughly | Volume | Each node leases a block (e.g. 10,000 IDs) and hands them out locally |
| **Content hash** | 32 bytes | None | No | Content identity | For immutable content (Chapter 17), not entities |

**UUIDv7** is the default worth reaching for: 48 bits of Unix milliseconds followed by random bits, so IDs are
unique without coordination *and* sorted by creation time, keeping B-tree inserts on the rightmost pages:

```js
// UUIDv7 (RFC 9562): 48-bit Unix milliseconds, then version and random bits.
// Time-ordered (good for B-tree inserts, Chapter 10) and needs no coordination.
import { randomBytes } from 'node:crypto';
function uuidv7(ms = Date.now(), rnd = randomBytes(10)) {
  const b = Buffer.alloc(16);
  b.writeUIntBE(ms, 0, 6);                       // 48-bit timestamp, big-endian so bytes sort by time
  rnd.copy(b, 6);                                // 80 random bits…
  b[6] = (b[6] & 0x0f) | 0x70;                   // …minus 4 for the version (7)
  b[8] = (b[8] & 0x3f) | 0x80;                   // …and 2 for the variant (10xx)
  const h = b.toString('hex');
  return `${h.slice(0, 8)}-${h.slice(8, 12)}-${h.slice(12, 16)}-${h.slice(16, 20)}-${h.slice(20)}`;
}
const ids = [1_763_200_000_000, 1_763_200_000_001, 1_763_200_003_000].map((ms) => uuidv7(ms, Buffer.alloc(10, 0xab)));
console.log(ids[0]);                                                    // 019a86e8-e000-7bab-abab-abababababab
console.log('sorted by time:', ids.every((id, i) => i === 0 || ids[i - 1] < id)); // sorted by time: true
```

(Within one millisecond, random bits order IDs arbitrarily; RFC 9562 describes optional counters for strict
monotonicity within a generator.) Store UUIDs in a native 16-byte type, not as 36-character text (Chapter 2).

**Collision risk** for random IDs follows the birthday bound — the probability of any collision among *n* IDs of
*b* random bits is about n² / 2^(b+1):

```js
// Birthday bound: probability of at least one collision among n random IDs of `bits` bits.
// expm1 keeps precision for tiny probabilities that 1 − exp(−x) would round to zero.
const collisionP = (n, bits) => -Math.expm1(-(n * n) / (2 * 2 ** bits));
console.log(collisionP(1e9, 64).toExponential(1), collisionP(1e9, 122).toExponential(1), collisionP(1e12, 122).toExponential(1));
// 2.7e-2 9.4e-20 9.4e-14
```

A billion random 64-bit IDs have a 2.7% chance of a collision — too high for primary keys. A UUIDv4's 122 random
bits make a billion IDs collide with probability around 10⁻¹⁹, and even a trillion around 10⁻¹³: effectively never,
*provided the random source is good*. UUIDv7 trades some random bits for the timestamp, but its collision risk is
per millisecond, so it stays negligible. The real-world collision risks are bad randomness (a seeded or forked
PRNG generating identical sequences) and duplicated worker IDs in Snowflake-style schemes.

> ⚠️ **Gotcha** — **Sequential IDs in URLs leak business data and invite enumeration.** `/invoices/40211` tells a
> competitor your invoice volume (compare two IDs a month apart), and tells an attacker to try 40210. Even with
> correct authorization checks, prefer non-sequential public identifiers — UUIDs, or an opaque public ID mapped to
> an internal key — for anything exposed outside your systems.

---

## 22.7 Common Mistakes and Best Practices

**Mistake: ordering cross-machine events by wall clock.** Skew reorders events within milliseconds of each other
and makes last-write-wins discard newer data.

**Mistake: treating Lamport order as causality.** A smaller Lamport timestamp doesn't mean "happened before";
concurrent events get arbitrary order.

**Mistake: last-write-wins where concurrent edits matter.** Without version vectors or CRDTs, one of two
concurrent updates is silently lost.

**Mistake: version vectors per client.** Entries per writer grow without bound; track per replica, or use dotted
version vectors.

**Mistake: random UUIDv4 primary keys on large B-tree tables.** Inserts scatter across the index; use time-ordered
UUIDv7 or Snowflake-style IDs.

**Mistake: Snowflake worker IDs from hostnames or random picks.** Two live generators with the same worker ID issue
duplicate IDs; assign worker IDs from a lease.

**Mistake: sequential public IDs.** They leak volume and invite enumeration attacks.

**Best practice: UUIDv7 as the default entity ID.** Unique without coordination, time-ordered for indexes, standard
and supported by databases.

**Best practice: hybrid logical clocks for version timestamps.** Close to wall time, causality-respecting, and
comparable as plain numbers.

**Best practice: handle the clock going backwards explicitly.** Any time-based generator must wait out small steps
and refuse large ones rather than risk duplicates.

---

## 22.8 Real-World Example: A Snowflake-Style ID Generator

When IDs must fit in a 64-bit integer — for compact indexes, or systems that store IDs as `bigint` — a
Snowflake-style generator packs a millisecond timestamp, a worker ID, and a per-millisecond sequence into 64 bits.
Each worker generates independently, so there's no coordination per ID; the only coordination is assigning each
live worker a unique worker ID. The hard parts are the edge cases: sequence overflow within one millisecond, and a
clock that moves backwards.

```js
// snowflake.mjs — 64-bit, roughly time-ordered IDs generated without coordination:
//   1 unused sign bit | 41 bits ms since a custom epoch | 10 bits worker id | 12 bits sequence
// 41 bits of milliseconds last ~69 years; 12 bits allow 4,096 IDs per worker per millisecond.

const EPOCH = 1_704_067_200_000n;           // 2024-01-01T00:00:00Z — pick once, never change
const WORKER_BITS = 10n, SEQ_BITS = 12n;
const MAX_WORKER = (1n << WORKER_BITS) - 1n, MAX_SEQ = (1n << SEQ_BITS) - 1n;

export class SnowflakeGenerator {
  #lastMs = -1n; #seq = 0n;

  constructor({ workerId, now = () => Date.now(), maxBackwardsMs = 10, sleepUntil }) {
    const w = BigInt(workerId);
    if (w < 0n || w > MAX_WORKER) throw new RangeError(`workerId must be 0–${MAX_WORKER}`);
    // Worker ids must be unique among live generators: assign them from a lease (Chapter 21),
    // never from a hash of the hostname, which can collide
    this.workerId = w;
    this.now = () => BigInt(now());
    this.maxBackwardsMs = BigInt(maxBackwardsMs);
    this.sleepUntil = sleepUntil ?? ((ms) => { while (this.now() < ms) { /* spin: sub-ms waits */ } });
  }

  next() {
    let ms = this.now();
    if (ms < this.#lastMs) {
      // The clock went backwards (NTP step, VM migration). Reusing earlier timestamps could
      // duplicate IDs already issued, so wait out small steps and refuse large ones loudly.
      if (this.#lastMs - ms > this.maxBackwardsMs) {
        throw new Error(`clock moved back ${this.#lastMs - ms} ms; refusing to issue IDs`);
      }
      this.sleepUntil(this.#lastMs);
      ms = this.now();
    }
    if (ms === this.#lastMs) {
      this.#seq = (this.#seq + 1n) & MAX_SEQ;
      if (this.#seq === 0n) {                // 4,096 IDs this millisecond: wait for the next one
        this.sleepUntil(this.#lastMs + 1n);
        ms = this.now();
      }
    } else {
      this.#seq = 0n;
    }
    this.#lastMs = ms;
    return ((ms - EPOCH) << (WORKER_BITS + SEQ_BITS)) | (this.workerId << SEQ_BITS) | this.#seq;
  }

  static decode(id) {
    return {
      time: new Date(Number((id >> (WORKER_BITS + SEQ_BITS)) + EPOCH)).toISOString(),
      workerId: Number((id >> SEQ_BITS) & MAX_WORKER),
      seq: Number(id & MAX_SEQ),
    };
  }
}

// ── Demo with a controllable clock ───────────────────────────────────────────────
let t = Date.UTC(2026, 10, 15, 12, 0, 0);
const gen = new SnowflakeGenerator({ workerId: 37, now: () => t, sleepUntil: (ms) => { t = Number(ms); } });
const first = gen.next(), second = gen.next();
console.log(String(first), SnowflakeGenerator.decode(first), SnowflakeGenerator.decode(second).seq);
// 380326064947351552 { time: '2026-11-15T12:00:00.000Z', workerId: 37, seq: 0 } 1

const burst = Array.from({ length: 5_000 }, () => gen.next());       // more than 4,096 in one ms
console.log('unique:', new Set(burst).size === burst.length, '| sorted:', burst.every((x, i) => i === 0 || burst[i - 1] < x),
  '| clock advanced to', SnowflakeGenerator.decode(burst.at(-1)).time);
// unique: true | sorted: true | clock advanced to 2026-11-15T12:00:00.001Z

t -= 5;                                                                 // small backwards step: waited out
console.log('after 5 ms step back:', gen.next() > burst.at(-1));        // after 5 ms step back: true
t -= 5_000;                                                             // large step: refuse
try { gen.next(); } catch (e) { console.log(e.message); }               // clock moved back 5000 ms; refusing to issue IDs
```

IDs from one worker are strictly increasing; IDs across workers are ordered by millisecond only, which is "roughly
time-ordered" — good for index locality and approximate sorting, not for causality.

The decision to defend is **refusing to generate IDs when the clock moves back more than a few milliseconds**,
which makes ID generation — and every insert that needs an ID — fail on that node. The alternative, used by some
implementations, is to keep issuing IDs from the last-seen timestamp until the clock catches up. That's safe only
while the process lives: after a restart, the generator has forgotten its last timestamp, and a clock that is now
behind will re-issue timestamps — and therefore IDs — that the previous process already handed out. Duplicate
primary keys corrupt data silently; a failing node is loud, is removed by health checks (Chapter 7), and recovers
when its clock is fixed. Small backward steps are waited out because they're common and brief; large ones mean the
node's notion of time is broken, and the safest thing a generator of time-based IDs can do with a broken clock is
stop.

---

## 22.9 Questions

**1.** MCQ — If L(X) < L(Y) for Lamport timestamps, what can you conclude? a) X happened before Y b) Y happened before X c) Y did not happen before X d) X and Y are concurrent
<details><summary>Answer</summary>**c)** If Y happened before X, then L(Y) < L(X), contradicting the premise; but X may have happened before Y or been concurrent with it.</details>

**2.** MCQ — What can version vectors tell you that Lamport clocks can't? a) A total order b) Whether two versions are concurrent c) The wall-clock time d) Which replica is the leader
<details><summary>Answer</summary>**b)** Comparing vectors distinguishes "descends from" from "concurrent".</details>

**3.** MCQ — Why does Spanner wait before making a commit visible? a) To batch writes b) To let the clock uncertainty interval pass the commit timestamp, so every later transaction anywhere gets a larger timestamp c) To replicate to all regions d) To avoid deadlocks
<details><summary>Answer</summary>**b)** Commit wait converts bounded clock uncertainty into externally consistent ordering.</details>

**4.** MCQ — Which ID scheme keeps B-tree inserts sequential without coordination? a) UUIDv4 b) UUIDv7 c) Content hash d) Random 64-bit integer
<details><summary>Answer</summary>**b)** Its leading timestamp bits make new IDs sort after older ones.</details>

**5.** MCQ — In a Snowflake-style layout with 12 sequence bits, how many IDs can one worker issue per millisecond? a) 1,024 b) 4,096 c) 65,536 d) Unlimited
<details><summary>Answer</summary>**b)** 2¹² = 4,096; beyond that, the generator waits for the next millisecond.</details>

**6.** MCQ — What's the main risk of exposing auto-increment IDs in public URLs? a) Collisions b) They leak business volume and invite enumeration c) They're too long d) They can't be indexed
<details><summary>Answer</summary>**b)** Competitors can infer growth, and attackers can probe adjacent IDs.</details>

**7.** Short — What do the two Lamport clock rules guarantee?
<details><summary>Answer</summary>That if X happened before Y (by local order or message causality, transitively), X's timestamp is smaller than Y's. With node-ID tie-breaking, this yields a total order consistent with causality.</details>

**8.** Short — What does a Dynamo-style store do when two writes have concurrent version vectors?
<details><summary>Answer</summary>It keeps both as siblings rather than discarding one, and returns them for reconciliation — by an application merge or a CRDT — after which the merged value gets a vector descending from both.</details>

**9.** Short — What problem do hybrid logical clocks solve that neither physical nor Lamport clocks solve alone?
<details><summary>Answer</summary>They give timestamps that respect causality (like Lamport clocks) while staying within clock skew of real time (unlike Lamport clocks), so they're meaningful for "as of" reads and comparable as plain numbers.</details>

**10.** Short — Why must Snowflake worker IDs be assigned from a lease rather than derived from hostnames?
<details><summary>Answer</summary>Two live generators with the same worker ID issue identical IDs whenever they generate in the same millisecond with the same sequence. Hostname hashes can collide and containers can be cloned; a lease guarantees uniqueness among live workers.</details>

**11.** Short — Why is store-and-replay of the last timestamp unsafe as a response to clock regression in an ID generator?
<details><summary>Answer</summary>The last timestamp lives only in memory; after a restart, a clock that is behind re-issues timestamps — and IDs — that the previous process already used. Refusing to generate is loud and recoverable; duplicate IDs are silent corruption.</details>

**12.** Output — Using §22.4's `compare`, what is `compare({ r1: 4, r2: 2 }, { r1: 3, r2: 2, r3: 1 })`?
<details><summary>Answer</summary>`'concurrent'` — the first is ahead on r1, the second on r3.</details>

**13.** Output — What does this print?
```js
let t = 0;
const tick = () => ++t;
const recv = (r) => (t = Math.max(t, r) + 1);
tick(); tick(); recv(10); tick();
console.log(t);
```
<details><summary>Answer</summary>`12` — two ticks reach 2, receiving 10 sets it to 11, and the last tick makes 12.</details>

**14.** Output — Using §22.8's layout and epoch, what is the timestamp field (milliseconds since the custom epoch) of an ID generated at exactly 2024-01-01T00:00:01.500Z?
<details><summary>Answer</summary>**1,500.** The custom epoch is 2024-01-01T00:00:00Z, so the field stores 1,500 ms; the ID is 1,500 × 2²² plus worker and sequence bits.</details>

**15.** Output — Using §22.6's `collisionP`, roughly what's the probability of a collision among 10 million random 48-bit IDs?
<details><summary>Answer</summary>**About 16%.** n² / 2^(b+1) = 10¹⁴ / 2⁴⁹ ≈ 10¹⁴ / 5.6 × 10¹⁴ ≈ 0.18, so 1 − e^(−0.18) ≈ 0.16 — far too high, which is why 48 random bits isn't an ID.</details>

**16.** Debug — In a multi-region store using last-write-wins on server timestamps, users occasionally report that their latest edit "reverted" to an earlier one made seconds before on another device. Cause and fix?
<details><summary>Answer</summary>The earlier edit was handled by a region whose clock is ahead, so it carries a later timestamp and wins. Use version vectors to detect concurrency and merge (or a CRDT), or route each user's writes to one home region so edits are ordered by one leader; at minimum use hybrid logical clocks so causally later edits win.</details>

**17.** Debug — Two services generated the same 64-bit order ID twice in one week, causing a primary-key violation on a downstream ledger. Both use the same Snowflake library with worker IDs set to `hash(hostname) % 1024`. What happened?
<details><summary>Answer</summary>Two hosts hashed to the same worker ID (with ~1,000 hosts collisions are very likely — the birthday bound again), and issued the same ID in the same millisecond with the same sequence. Assign worker IDs from a lease in a coordination service or database so each live generator's ID is unique.</details>

**18.** Debug — After a VM live migration, a service crash-looped with "clock moved back 4,200 ms" errors from its ID generator for about four seconds, then recovered. Is this a bug?
<details><summary>Answer</summary>No — it's the generator correctly refusing to issue possibly duplicate IDs while the clock was behind. Make it operationally graceful: fail readiness rather than crash so the balancer routes elsewhere, alert on clock regressions, and configure time synchronisation to slew where possible.</details>

**19.** Debug — A database table with a UUIDv4 primary key has insert latency that rises steadily as the table grows past memory. Switching to auto-increment isn't possible because IDs are generated by clients. What do you change?
<details><summary>Answer</summary>Generate UUIDv7 on the clients instead: still uncoordinated and unique, but time-ordered, so inserts land on the rightmost B-tree pages. Store them in a native 16-byte UUID column. Existing rows can stay; new inserts get the benefit immediately.</details>

**20.** Code — Write `vvIncrement(vector, replica)` and `vvDominates(a, b)` (true when `a` descends from or equals `b`).
<details><summary>Answer</summary>

```js
const vvIncrement = (v, r) => ({ ...v, [r]: (v[r] ?? 0) + 1 });
const vvDominates = (a, b) => Object.keys(b).every((k) => (a[k] ?? 0) >= b[k]);
const v1 = vvIncrement({}, 'r1'), v2 = vvIncrement(v1, 'r2');
console.log(vvDominates(v2, v1), vvDominates(v1, v2)); // true false
```
</details>

**21.** Code — Write `parseUuidV7Time(uuid)` returning the embedded creation time as a `Date`.
<details><summary>Answer</summary>

```js
function parseUuidV7Time(uuid) {
  const hex = uuid.replace(/-/g, '');
  if (hex[12] !== '7') throw new Error('not a version 7 UUID');
  return new Date(parseInt(hex.slice(0, 12), 16));   // first 48 bits are Unix milliseconds
}
console.log(parseUuidV7Time('019a86e8-e000-7bab-abab-abababababab').toISOString()); // 2025-11-15T09:46:40.000Z
```
</details>

**22.** Code — Write a range allocator: `createRangeAllocator(reserve, blockSize)` where `reserve(n)` atomically reserves `n` IDs from a central counter and returns the first; the allocator hands out IDs locally, reserving a new block only when the current one runs out.
<details><summary>Answer</summary>

```js
function createRangeAllocator(reserve, blockSize = 10_000) {
  let next = 0, end = 0, pending = null;
  return async function nextId() {
    while (next >= end) {
      // One reservation at a time; concurrent callers wait for the same block
      pending ??= reserve(blockSize).then((start) => { next = start; end = start + blockSize; pending = null; });
      await pending;
    }
    return next++;
  };
}
let counter = 1;
const reserve = async (n) => { const start = counter; counter += n; return start; };   // e.g. UPDATE … RETURNING
const nextId = createRangeAllocator(reserve, 3);
console.log(await Promise.all([nextId(), nextId(), nextId(), nextId()])); // [ 1, 2, 3, 4 ]
```
A crash wastes the rest of a block (gaps are harmless); IDs are unique but only roughly time-ordered across nodes.</details>

**23.** Scenario — A team is choosing primary keys for a new multi-region system: entities are created in every region, IDs appear in public URLs, and tables will reach billions of rows. What do you recommend?
<details><summary>Answer</summary>UUIDv7: generated anywhere without coordination (no cross-region dependency on creation), time-ordered for B-tree locality, and non-sequential enough not to leak volume or invite enumeration (it does reveal creation time — acceptable for most entities; use an opaque public ID if not). Store as native 16-byte UUIDs. Snowflake-style 64-bit IDs are an option if index size is critical, at the cost of managing worker-ID leases per region.</details>

**24.** Scenario — A collaborative note-taking app syncs notes between a user's phone and laptop, both of which can edit offline. The current design uses last-write-wins on device timestamps. Users complain about lost edits. What do you propose?
<details><summary>Answer</summary>Device clocks are unreliable and LWW discards concurrent edits. Track a version vector per note (one entry per device) to detect concurrent edits; on concurrency, merge — ideally by representing note content as a text CRDT so merges are automatic (Chapter 46), or at least by keeping both versions as a visible conflict. Use hybrid logical clocks for human-facing "last edited" times.</details>

---

## Chapter Summary

- Ordering decides conflict resolution, causality, snapshots, and sort keys; physical clocks give approximate
  human time, not dependable order across machines.
- Spanner-style clocks with bounded uncertainty buy external consistency by waiting out the uncertainty; most
  systems use logical or hybrid clocks instead.
- Lamport clocks give a total order consistent with causality, but a smaller timestamp doesn't mean "happened
  before" — they can't detect concurrency.
- Version vectors detect concurrent updates, so conflicting writes become siblings to merge rather than silent
  losses; CRDTs make the merge automatic.
- Hybrid logical clocks combine causality with closeness to wall time and fit in 64 bits, which is why databases
  use them for version timestamps.
- Default to UUIDv7 for entity IDs; use Snowflake-style or range allocation when 64-bit IDs matter, assign worker
  IDs from leases, and refuse to generate when the clock jumps back.

---

## Practice Exercises

1. Simulate three nodes exchanging messages with random delays, stamp events with Lamport clocks and vector
   clocks, and count how often Lamport order disagrees with real (simulated) time for concurrent events.
2. Implement sibling handling for a key-value store with version vectors: concurrent writes produce siblings,
   reads return all of them, and a write with the merged vector resolves them.
3. Read RFC 9562's sections on UUIDv7 and its monotonicity options, and extend §22.6's generator to guarantee
   strictly increasing IDs within one process.
4. Benchmark inserts of 50 million rows into PostgreSQL with UUIDv4, UUIDv7, and `bigint` Snowflake keys, and
   compare insert throughput and index size.
5. Measure the clock offset between several machines you have access to (`chronyc tracking` or `ntpq -p`) over
   a day, and estimate the largest skew your system should tolerate.

---

## Mini Project: An ID and Versioning Toolkit

Build a small library and test suite covering the ordering tools in this chapter, suitable for use by other
services in an organisation.

- UUIDv7 generation with optional within-millisecond monotonicity, plus parsing of the embedded timestamp.
- A Snowflake-style generator whose worker ID is leased from a database table with heartbeats, and which fails
  readiness on clock regression.
- A hybrid logical clock with serialisation to a sortable 64-bit integer.
- Version-vector utilities: increment, compare, merge, and sibling resolution helpers.
- Property-based tests: generated IDs are unique and ordered under simulated clock steps; HLC timestamps respect
  causality for random message schedules; version-vector comparisons are consistent with simulated histories.

---

**Next:** [Chapter 23 — Distributed Transactions and Sagas →](23-distributed-transactions.md)
