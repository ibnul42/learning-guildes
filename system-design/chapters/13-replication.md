# Chapter 13 — Replication

## Learning Objectives

- Choose synchronous, asynchronous, or semi-synchronous replication from durability and latency requirements
- Name the anomalies replication lag causes and fix each with a specific session guarantee
- Explain what goes wrong during failover — lost writes, split brain — and how fencing prevents it
- Compare single-leader, multi-leader, and leaderless replication, including how each handles conflicts
- Reason precisely about quorums: what W + R > N guarantees, and what it doesn't

---

## 13.1 Why Replicate, and What Makes It Hard

**Replication** keeps copies of the same data on several machines. It serves four goals, and
which ones matter shapes the design:

| Goal | How replication serves it |
|---|---|
| **Durability** | Data survives the loss of a disk, machine, or zone |
| **Availability** | Another copy can serve when one fails (Chapter 4's parallel formula) |
| **Read scalability** | Reads spread across copies |
| **Latency** | Copies near users serve their reads locally (Chapter 32) |

If data never changed, replication would be trivial: copy it once. All the difficulty lives in
**propagating changes** — deciding who accepts writes, how quickly copies hear about them, what
readers see in between, and what happens when the machines disagree about who's in charge. There
are three broad answers, by how many nodes accept writes:

```text
single-leader          multi-leader                 leaderless
   writes                writes   writes              writes ─▶ any W of N
     │                     │        │                 reads  ─▶ any R of N
  [leader]            [leader A]◀─▶[leader B]       [n1] [n2] [n3]
   │    │               │             │
[fol.] [fol.]        [fol.]        [fol.]
```

---

## 13.2 Single-Leader Replication

One node — the **leader** (primary) — accepts all writes. It records each change in its
replication log (usually the WAL from Chapter 10) and streams it to **followers** (replicas), which
apply changes in the same order. Reads can go to the leader or to followers. PostgreSQL, MySQL,
SQL Server, MongoDB replica sets, and Redis all work this way by default.

The central choice is **when the leader acknowledges a write**:

| Mode | Leader acknowledges after | Write latency | Data lost if the leader dies | Availability of writes |
|---|---|---|---|---|
| **Asynchronous** | Its own local commit | Lowest | Writes not yet shipped (often a few hundred ms' worth) | Unaffected by followers |
| **Synchronous (all)** | Every follower confirms | Highest; slowest follower sets it | None | Any follower failure blocks writes |
| **Semi-synchronous / quorum** | At least one (or k) followers confirm | Moderate | None, if a confirmed follower is promoted | Survives some follower failures |

Fully synchronous replication to every follower is rare: one slow or dead follower stalls every
write. The common production configurations are **asynchronous** (accepting a small loss window —
the RPO of Chapter 32) or **one synchronous follower plus asynchronous others** (PostgreSQL's
`synchronous_standby_names` with a quorum of one, MySQL's semi-sync), which guarantees every
acknowledged write exists on two machines while tolerating the loss of any single follower.

> ⚠️ **Gotcha** — Synchronous replication across regions puts a cross-region round trip (60–150 ms,
> Chapter 2) on every commit. A system with a 50 ms write budget can't have it. Teams that need
> zero-loss *and* low latency usually replicate synchronously within a region (across zones, ~1 ms)
> and asynchronously across regions, and accept a small cross-region loss window.

### What the log contains

| Log format | What's shipped | Trade-off |
|---|---|---|
| Statement-based | The SQL statements | Non-deterministic functions (`now()`, `random()`) diverge on replicas; largely abandoned |
| Physical (WAL shipping) | Byte-level page changes | Exact and efficient; replicas must run the same engine version; can't filter tables |
| Logical (row-based) | "Row with key K changed to V" | Version-independent; can feed other systems (CDC, Chapter 11); a little more overhead |

Logical replication is what makes change data capture possible: the same stream that keeps a
replica current can keep a search index, cache, or warehouse current.

### Adding a follower

A new follower can't replay the log from the beginning of time. It starts from a **snapshot** of
the leader taken at a known log position, then streams every change after that position. The
pairing — "snapshot as of position P, then the log from P" — is the general recipe for creating any
derived copy without stopping writes, and it reappears in rebuilding search indexes (Chapter 18) and
backfilling stream processors (Chapter 25).

---

## 13.3 Replication Lag and Its Anomalies

With asynchronous followers, a follower is always somewhat behind the leader — usually milliseconds,
sometimes seconds or minutes under load, during long transactions, or after a network blip. Reads
from followers therefore see the past. That's acceptable in many places, but three specific anomalies
reliably produce bug reports:

| Anomaly | What the user sees | Guarantee that prevents it |
|---|---|---|
| **Not reading your own writes** | Saves their profile, page reloads, old profile shows | **Read-your-writes**: a user always sees their own committed writes |
| **Time going backwards** | Refreshes a comment thread; a comment appears, then disappears on the next refresh | **Monotonic reads**: a user never sees older data after newer |
| **Effects before causes** | Sees an answer to a question that hasn't appeared yet | **Consistent prefix reads**: reads see writes in an order consistent with causality |

The fixes, from crude to precise:

- **Read from the leader for data the user just changed** — e.g. their own profile always, or
  everything for a few seconds after any write. Simple, but the "few seconds" is a guess: lag spikes
  past it, and a heuristic window sends a lot of traffic to the leader.
- **Pin a user's session to one follower.** Gives monotonic reads (one follower never goes backwards)
  but not read-your-writes, and breaks when that follower fails.
- **Track log positions.** After a write, return the leader's commit position (an LSN in PostgreSQL)
  to the client as a token. Each read carries the token and goes only to a follower that has replayed
  at least that far — or to the leader if none has. This gives exact read-your-writes and monotonic
  reads with no guessed windows. The real-world example in §13.9 implements it.

Consistent prefix matters most in **partitioned** systems, where related writes live on different
partitions replicating independently (Chapter 14). The fixes there are to keep causally related data
in one partition, or to carry causal metadata (Chapter 22's vector clocks and hybrid timestamps).

> 💼 **Interview** — "A user updates their profile and sometimes sees the old one. Why, and what do
> you do?" The expected answer names replication lag on read replicas, then the read-your-writes
> guarantee, then a concrete mechanism — ideally the log-position token, with "read your own profile
> from the primary" as the simple version.

---

## 13.4 Failover: Where Replication Goes Wrong

When a leader fails, a follower must be **promoted**, clients redirected to it, and the old leader
prevented from accepting writes if it comes back. Each step has a classic failure.

**Detecting failure.** There's no reliable way to tell "dead" from "slow" or "unreachable from here"
(Chapter 19). A short timeout fails over on every GC pause or network blip — each failover is itself
a disruption — while a long one extends every real outage. Most systems use a timeout of 10–30
seconds plus confirmation from multiple observers.

**Choosing the new leader.** It should be the follower with the most recent data. With asynchronous
replication, even the most up-to-date follower may lack the old leader's last writes, and those
writes are **lost** — acknowledged to clients, then gone. If the old leader later rejoins with those
writes, they conflict with whatever the new leader accepted in the meantime; most systems discard
them. The loss window is the replication lag at the moment of failure, which is why semi-synchronous
replication (promote only a follower that confirmed every acknowledged write) exists.

> ⚠️ **Gotcha** — **Discarded writes can collide with reused identifiers.** If the old leader had
> handed out auto-increment IDs 1001–1010 that never reached the new leader, the new leader hands
> them out again for different rows. Any other system that recorded the old IDs — a cache, a search
> index, an external partner, an email already sent — now points at the wrong records. This exact
> pattern has caused publicly reported cross-customer data exposure. Prefer IDs that don't depend on
> a single leader's counter (Chapter 22), and treat failover as an event that requires reconciling
> derived stores.

**Split brain.** If the old leader isn't actually dead — just partitioned from the failure detector —
it may keep accepting writes while the new leader does too. Two leaders, two diverging histories, and
no clean way to merge them. The defences:

- **Fencing.** Before the new leader starts, the old one is cut off: its storage access revoked, its
  network isolated, its process killed ("STONITH" — shoot the other node in the head).
- **Fencing tokens / epochs.** Each leadership term gets a monotonically increasing number; storage
  and downstream systems reject writes carrying an older term. Chapter 21 builds this properly.
- **Consensus-based leadership.** Leaders hold a lease granted by a majority; a minority-side node
  can't renew its lease and steps down by itself (Chapter 21).

**Redirecting clients.** Clients find the new leader through DNS, a virtual IP, a proxy, or a
configuration service — each with Chapter 6's propagation delays. Connection pools must notice their
connections now point at a demoted node (which should reject writes, not silently accept them).

Because of all this, many teams keep **automated failover for the clear case** (leader unreachable
from a majority of observers for N seconds, a caught-up synchronous follower available) and **require
a human for the ambiguous cases**. Rehearse failover regularly; the first failover in production
should not be the first failover ever.

---

## 13.5 Multi-Leader Replication

In **multi-leader** (or active-active) replication, several nodes accept writes, each replicating its
changes to the others. Its reasons are specific:

- **Multi-region writes** — each region writes to a local leader at local latency, instead of crossing
  an ocean to one global leader (Chapter 32).
- **Offline-capable clients** — a mobile app or a collaborative editor is effectively a leader while
  offline, syncing later (Chapter 46).
- **Surviving a region's loss without failover** for writes.

The price is **write conflicts**: two leaders can accept changes to the same record concurrently, and
neither knows about the other until replication catches up. There's no conflict in a single-leader
system because one node orders all writes; multi-leader systems must *resolve* conflicts after the fact.

| Resolution strategy | How it works | Cost |
|---|---|---|
| **Last write wins (LWW)** | Keep the write with the highest timestamp; discard the other | Silently loses data; depends on clocks (Chapter 22) |
| **Merge** | Combine values (union of set items, concatenate edits) | Only works for mergeable data |
| **CRDTs** | Data types designed so concurrent updates always merge deterministically (counters, sets, text) | Constrains the data model; metadata overhead (Chapter 46) |
| **Record both, let the application or user decide** | Store siblings; resolve on read or ask the user | Complexity pushed to application and UI |
| **Avoid conflicts** | Route all writes for a given record to one "home" leader | Back to single-leader per record; fails over when the home region does |

The most effective strategy is the last one. Giving each user or tenant a **home region** that takes
all of their writes keeps most data effectively single-leader — conflicts can only arise during
failover — while still distributing write load and latency across regions. True concurrent multi-leader
writes to the *same* records are rare in well-designed systems and painful in the rest.

> ⚠️ **Gotcha** — LWW sounds like a resolution strategy but is really a data-loss policy. Two users
> add different items to a shared list at the same moment in two regions; LWW keeps one list and
> silently discards the other user's item. And with clock skew between regions, "last" may not even
> be the later write. Use LWW only where losing a concurrent update is genuinely acceptable — a "last
> seen" timestamp, a cache.

---

## 13.6 Leaderless Replication and Quorums

**Leaderless** replication — the Dynamo design, used by Cassandra, ScyllaDB, Riak, and DynamoDB's
internals — removes the leader entirely. A client (or a coordinator node acting for it) sends each
write to all **N** replicas of a key and considers it successful when **W** acknowledge; it sends each
read to replicas and waits for **R** responses, returning the newest version among them.

The core rule: if **W + R > N**, the set of replicas that acknowledged the latest write and the set
that answered the read **must overlap** in at least one node, so the read sees the write. The
arithmetic is exact enough to enumerate:

```js
// Enumerate every way a write can land on W of N replicas and a read can query R of them.
// The read is stale if the two sets don't overlap. Exact probabilities, no sampling.
function combinations(n, k, start = 0, acc = [], out = []) {
  if (acc.length === k) { out.push(new Set(acc)); return out; }
  for (let i = start; i < n; i++) combinations(n, k, i + 1, [...acc, i], out);
  return out;
}

function staleReadProbability(n, w, r) {
  const writes = combinations(n, w), reads = combinations(n, r);
  let stale = 0;
  for (const ws of writes) for (const rs of reads) if (![...rs].some((x) => ws.has(x))) stale++;
  return stale / (writes.length * reads.length);
}

for (const [n, w, r] of [[3, 1, 1], [3, 2, 1], [3, 2, 2], [3, 3, 1], [5, 2, 2], [5, 3, 3]]) {
  const p = staleReadProbability(n, w, r);
  console.log(`N=${n} W=${w} R=${r}  W+R>N: ${w + r > n ? 'yes' : 'no '}  stale reads: ${(p * 100).toFixed(1)}%`);
}
// N=3 W=1 R=1  W+R>N: no   stale reads: 66.7%
// N=3 W=2 R=1  W+R>N: no   stale reads: 33.3%
// N=3 W=2 R=2  W+R>N: yes  stale reads: 0.0%
// N=3 W=3 R=1  W+R>N: yes  stale reads: 0.0%
// N=5 W=2 R=2  W+R>N: no   stale reads: 30.0%
// N=5 W=3 R=3  W+R>N: yes  stale reads: 0.0%
```

These are worst-case numbers — a read issued the instant after the write, before background
propagation reaches the other replicas. In practice the non-quorum configurations are stale far less
often, because the remaining replicas usually receive the write within milliseconds. But "usually" is
the point: only W + R > N makes it a guarantee.

The common settings and what they trade:

| Setting (N = 3) | Write availability | Read availability | Reads see latest acknowledged write? |
|---|---|---|---|
| W = 1, R = 1 (`ONE`/`ONE`) | Survives 2 down | Survives 2 down | No — fast and eventually consistent |
| W = 2, R = 2 (`QUORUM`/`QUORUM`) | Survives 1 down | Survives 1 down | Yes, under normal operation |
| W = 3, R = 1 | No replica may be down | Survives 2 down | Yes — read-optimised, write-fragile |
| W = 1, R = 3 | Survives 2 down | No replica may be down | Yes — write-optimised, read-fragile |

### Keeping replicas converged

Replicas miss writes — they were down, slow, or partitioned. Three mechanisms repair them:

- **Read repair:** when a read sees an older version on some replica, the coordinator writes the newer
  version back to it.
- **Hinted handoff:** when a target replica is down, another node accepts the write with a "hint" and
  forwards it when the target returns. With a **sloppy quorum**, those stand-in nodes count toward W —
  which keeps writes available during failures but means W + R > N no longer guarantees overlap with
  the *real* replicas.
- **Anti-entropy:** a background process compares replicas — efficiently, using Merkle trees of hashes
  so only differing ranges are exchanged — and copies missing data. Chapter 39 builds these pieces into
  a full key-value store.

### What quorums don't give you

W + R > N is often described as "strong consistency". It isn't quite, and the gaps matter:

- **Concurrent writes** to the same key reach replicas in different orders; replicas need a rule to
  pick a winner (LWW timestamps or version vectors), and LWW loses one of them.
- **A write that fails** (reached fewer than W replicas) isn't rolled back on the replicas it did
  reach, so a later read may or may not return it.
- **A read concurrent with a write** may return the new value from one replica and then, on the next
  read, the old value from another — reads aren't monotonic or linearisable without extra mechanisms.
- **Sloppy quorums and hinted handoff** break the overlap guarantee during failures.

Quorums give you a high probability of fresh reads and a clear availability trade-off. When you need
linearisability — a lock, a uniqueness check, a compare-and-set — use consensus (Chapter 21) or a
single-leader system, and Chapter 20 explains precisely what that word means.

---

## 13.7 Choosing a Replication Strategy

| | Single-leader | Multi-leader | Leaderless |
|---|---|---|---|
| Write path | One leader orders everything | Any leader; conflicts resolved later | Any W of N replicas |
| Conflicts | None (one order) | Yes — must be resolved | Yes — concurrent writes need version resolution |
| Failover | Required, and risky (§13.4) | Not needed for writes | Not needed |
| Read consistency | Leader: latest; followers: lagging | Each leader: its own view | Tunable with R and W |
| Write latency across regions | One cross-region hop for remote writers | Local | Depends on where W replicas are |
| Fits | Most systems, especially transactional ones | Multi-region writes, offline clients | High write availability, partition tolerance, simple key access |

**Default to single-leader**, with one synchronous follower in another zone and asynchronous replicas
for read scaling. Move to multi-leader when users in several regions need local write latency, and
partition writes by home region to keep conflicts rare. Use leaderless stores when their access model
(key-based, no multi-key transactions) fits and you value write availability over strict ordering.

---

## 13.8 Common Mistakes and Best Practices

**Mistake: reading from replicas without a consistency plan.** Users see their own changes vanish and
reappear; decide per endpoint which guarantee it needs.

**Mistake: synchronous replication to every follower.** One slow follower stalls every write; use one
synchronous follower (or a quorum) and the rest asynchronous.

**Mistake: synchronous replication across regions on a latency-critical path.** Every commit pays a
cross-region round trip; replicate synchronously within a region instead.

**Mistake: automated failover without fencing.** A partitioned old leader keeps accepting writes and
the cluster splits into two diverging histories.

**Mistake: last-write-wins for data people edit.** Concurrent updates are silently discarded, and clock
skew can discard the later one.

**Mistake: treating W + R > N as linearisable.** Concurrent writes, partial failures, and sloppy quorums
all produce reads that quorum arithmetic alone doesn't prevent.

**Mistake: never rehearsing failover.** Runbooks, DNS TTLs, pool behaviour, and promotion scripts all
rot; the first real failover finds every gap at once.

**Best practice: log-position tokens for read-your-writes.** Exact, cheap, and free of guessed time
windows.

**Best practice: monitor replication lag as a first-class SLI.** Alert on lag in seconds and bytes, and
know the lag at which reads must stop using a replica.

**Best practice: avoid conflicts by giving records a home.** Route each user's or tenant's writes to one
leader, so multi-leader conflicts only arise during failover.

---

## 13.9 Real-World Example: A Read-Your-Writes Router

This router sends writes to the primary and returns the commit position as a session token; reads go
to any replica that has replayed at least that position, falling back to the primary when none has.
The simulated cluster underneath has one up-to-date replica and one that lags by one log entry.

```js
// consistency-router.mjs — read-your-writes and monotonic reads over async replicas,
// using log-position tokens instead of "read from the primary for a while after writing".

export class ConsistencyRouter {
  #primary; #replicas; #rand;

  constructor({ primary, replicas, rand = Math.random }) {
    this.#primary = primary;      // { name, lsn(), query(sql) }
    this.#replicas = replicas;    // [{ name, replayedLsn(), query(sql) }]
    this.#rand = rand;
  }

  // Writes go to the primary; the caller keeps the commit position as a session token
  // (in a cookie, a header, or the client's local state).
  async write(sql) {
    const result = await this.#primary.query(sql);
    return { result, token: this.#primary.lsn() };
  }

  // Reads go to any replica that has replayed at least the session's token. Monotonic reads
  // come for free: the caller raises its token to whatever position it has observed.
  async read(sql, { token = 0 } = {}) {
    const caughtUp = this.#replicas.filter((r) => r.replayedLsn() >= token);
    if (caughtUp.length === 0) {
      // Nobody is fresh enough: fall back to the primary rather than return stale data.
      // A storm of these means replicas are lagging badly — that's what to alert on.
      const result = await this.#primary.query(sql);
      return { result, servedBy: this.#primary.name, token: Math.max(token, this.#primary.lsn()) };
    }
    const replica = caughtUp[Math.floor(this.#rand() * caughtUp.length)];
    const result = await replica.query(sql);
    return { result, servedBy: replica.name, token: Math.max(token, replica.replayedLsn()) };
  }
}

// ── Simulation: one primary, two async replicas with different lag ──────────────────
function cluster() {
  const log = [];                                    // the primary's WAL: [{ lsn, key, value }]
  const state = (upTo) => Object.fromEntries(log.filter((e) => e.lsn <= upTo).map((e) => [e.key, e.value]));
  const primary = {
    name: 'primary', lsn: () => log.length,
    async query(sql) {
      const m = sql.match(/^SET (\w+)=(.+)$/);
      if (m) { log.push({ lsn: log.length + 1, key: m[1], value: m[2] }); return 'ok'; }
      return state(log.length)[sql.slice(4)];       // "GET key"
    },
  };
  const replica = (name, lag) => ({
    name, replayedLsn: () => Math.max(0, log.length - lag),
    async query(sql) { return state(this.replayedLsn())[sql.slice(4)]; },
  });
  return { primary, replicas: [replica('replica-a', 0), replica('replica-b', 1)] };
}

const c = cluster();
const router = new ConsistencyRouter({ ...c, rand: () => 0.99 });  // prefer replica-b, the laggy one
await router.write('SET name=Ana');
await router.write('SET plan=free');
const { token } = await router.write('SET plan=pro');               // the user upgrades

const naive = await router.read('GET plan');                        // no token
console.log('no token:  ', naive.result, 'from', naive.servedBy);   // no token:   free from replica-b
const ryw = await router.read('GET plan', { token });               // session token
console.log('with token:', ryw.result, 'from', ryw.servedBy);       // with token: pro from replica-a
c.replicas[0].replayedLsn = () => 1;                                // now every replica is behind
const fallback = await router.read('GET plan', { token });
console.log('all behind:', fallback.result, 'from', fallback.servedBy); // all behind: pro from primary
```

In PostgreSQL the same mechanism uses real positions: `pg_current_wal_lsn()` on the primary after a
commit, and `pg_last_wal_replay_lsn()` on each replica, polled every few hundred milliseconds by the
router.

The decision worth defending is **tokens instead of a time window** — the common shortcut of "after a
user writes, send their reads to the primary for 5 seconds". The window is a guess about lag, and it's
wrong in both directions: lag is usually milliseconds, so five seconds sends far more traffic to the
primary than needed, and under load or during a long-running transaction lag can exceed five seconds,
which is exactly when users notice stale data. A token states the actual requirement — "a copy that has
applied *this* write" — and checks it directly, so it's never stale and only falls back to the primary
when replicas really are behind. It also degrades informatively: a rising fallback rate is a precise
signal that replicas are lagging, available long before users complain.

---

## 13.10 Questions

**1.** MCQ — Which replication mode can lose acknowledged writes when the leader fails? a) Synchronous to all followers b) Semi-synchronous with promotion of a confirmed follower c) Asynchronous d) None — replication never loses writes
<details><summary>Answer</summary>**c)** The leader acknowledges before followers have the write; if it dies, writes not yet shipped are lost.</details>

**2.** MCQ — A user edits a comment, refreshes, and sees the old version. Which guarantee is missing? a) Monotonic reads b) Read-your-writes c) Consistent prefix d) Linearisability of other users' writes
<details><summary>Answer</summary>**b)** The user's own committed write isn't visible to their subsequent read, which went to a lagging replica.</details>

**3.** MCQ — With N = 5 replicas, which setting guarantees that every read quorum overlaps every write quorum? a) W = 2, R = 2 b) W = 3, R = 2 c) W = 1, R = 4 d) W = 3, R = 3
<details><summary>Answer</summary>**d)** Overlap requires W + R > N, i.e. at least 6 here. The other options sum to 4 or 5.</details>

**4.** MCQ — What is split brain? a) A replica with corrupt data b) Two nodes both believing they're leader and accepting writes c) A network partition between followers d) A read quorum larger than the write quorum
<details><summary>Answer</summary>**b)** Typically an old leader that was partitioned, not dead, continuing to accept writes after a new leader is promoted.</details>

**5.** MCQ — Which conflict-handling strategy keeps multi-leader conflicts rare without changing the data model? a) Last write wins b) CRDTs c) Routing each record's writes to a home leader d) Asking the user
<details><summary>Answer</summary>**c)** Each record is effectively single-leader; conflicts arise only during failover of its home.</details>

**6.** MCQ — What does hinted handoff with a sloppy quorum trade away? a) Write availability b) The guarantee that W + R > N overlaps with the key's real replicas c) Durability of hints d) Read availability
<details><summary>Answer</summary>**b)** Stand-in nodes count toward W, keeping writes available, but a read from the real replicas may not overlap them.</details>

**7.** Short — Why do most teams avoid fully synchronous replication to every follower?
<details><summary>Answer</summary>Every write waits for the slowest follower, and any follower failure blocks all writes. One synchronous follower (or a quorum) gives zero-loss for acknowledged writes while tolerating follower failures.</details>

**8.** Short — How does "snapshot at position P, then stream the log from P" let you add a replica without stopping writes?
<details><summary>Answer</summary>The snapshot captures a consistent state as of P while writes continue; the new replica then applies every change after P from the log, so it converges without the leader ever pausing.</details>

**9.** Short — Why is failure detection the hardest part of automated failover?
<details><summary>Answer</summary>A dead leader and a slow or partitioned one look the same from outside. Short timeouts cause needless (disruptive) failovers on pauses and blips; long ones extend real outages; and failing over while the old leader is alive risks split brain.</details>

**10.** Short — Why is last-write-wins described as a data-loss policy?
<details><summary>Answer</summary>Of two concurrent writes, it keeps one and silently discards the other; with clock skew it may even discard the truly later one. It's only acceptable where losing concurrent updates doesn't matter.</details>

**11.** Short — Give two reasons W + R > N doesn't make a leaderless store linearisable.
<details><summary>Answer</summary>Any two of: concurrent writes reach replicas in different orders and need a resolution rule; failed writes remain on some replicas and may be read later; a read concurrent with a write can return new then old values; sloppy quorums break overlap during failures.</details>

**12.** Output — Using §13.6's `staleReadProbability`, what does `staleReadProbability(4, 2, 2)` return?
<details><summary>Answer</summary>**1/6 ≈ 0.167.** There are 6 write sets and 6 read sets; for each write set exactly one read set (its complement) is disjoint: 6 / 36.</details>

**13.** Output — What does this print?
```js
const overlaps = (n, w, r) => w + r > n;
console.log(overlaps(3, 2, 2), overlaps(5, 2, 3), overlaps(7, 4, 4));
```
<details><summary>Answer</summary>`true false true` — 4 > 3; 5 is not > 5; 8 > 7.</details>

**14.** Output — Using §13.9's router and cluster, after the three writes a session holds token 2 (from an earlier read). With `rand` returning 0.99, which replica serves `GET plan`, and what value is returned?
<details><summary>Answer</summary>**replica-b, `free`.** Both replicas have replayed at least 2 (b is at 2, a at 3), and `rand = 0.99` picks the last eligible one. The session sees the state as of its own token — `free` — which is correct for read-your-writes because the session never observed position 3.</details>

**15.** Output — An asynchronous replica lags 400 ms behind a primary handling 2,500 writes/s. If the primary dies and this replica is promoted, roughly how many acknowledged writes are lost?
<details><summary>Answer</summary>**~1,000.** 2,500 × 0.4 = 1,000 writes acknowledged by the old primary but not yet replicated.</details>

**16.** Debug — After an automated failover, the new primary began issuing order IDs that duplicated IDs in the analytics warehouse and in emails already sent to customers. What happened, and how do you prevent it?
<details><summary>Answer</summary>The old primary had committed (and exported) rows with IDs that never replicated; the promoted replica's sequence restarted below them and reissued those IDs for new orders. Prevent it with IDs not tied to one leader's counter (UUIDv7, Snowflake-style), semi-synchronous replication for zero loss, and a post-failover reconciliation of derived stores.</details>

**17.** Debug — Users report that a newly posted comment sometimes appears, disappears on refresh, and reappears later. Reads are load-balanced randomly across three async replicas. Cause and fix?
<details><summary>Answer</summary>A monotonic-reads violation: successive reads hit replicas at different lag, so the user sees newer then older state. Carry a log-position token per session and only read from replicas at or past it, or pin the session to one replica.</details>

**18.** Debug — During a brief network partition, a database cluster ended up with two primaries for about 40 seconds. After the partition healed, some transactions from that period were missing. What allowed this, and what prevents it?
<details><summary>Answer</summary>The failover promoted a new primary without fencing the old one, which stayed reachable to some clients and kept accepting writes; on rejoin its divergent writes were discarded. Use fencing (revoke the old node's access, epoch/fencing tokens checked by storage) or consensus-based leases so a minority-side leader can't keep acting as primary.</details>

**19.** Debug — A Cassandra-backed feature using `ONE`/`ONE` consistency occasionally shows a user a setting they changed a minute ago reverting, then fixing itself. Writes all succeed. What's happening, and what are the options?
<details><summary>Answer</summary>With W = R = 1 of N = 3, a read can hit a replica that missed the write (it was briefly down or slow) until read repair or anti-entropy catches it up. Use `QUORUM`/`QUORUM` (or `LOCAL_QUORUM`) for this data, accepting slightly higher latency, or route a user's reads for their own settings with read-your-writes logic.</details>

**20.** Code — Write `pickReplica(replicas, token)` that returns the least-lagged replica among those with `replayedLsn >= token`, or `null` if none qualifies.
<details><summary>Answer</summary>

```js
function pickReplica(replicas, token) {
  const eligible = replicas.filter((r) => r.replayedLsn >= token);
  if (eligible.length === 0) return null;            // caller falls back to the primary
  return eligible.reduce((best, r) => (r.replayedLsn > best.replayedLsn ? r : best));
}
console.log(pickReplica([{ name: 'a', replayedLsn: 90 }, { name: 'b', replayedLsn: 120 }], 100)?.name); // b
```
Picking the freshest replica is one policy; spreading load randomly among eligible ones (as §13.9 does) is another.</details>

**21.** Code — Write `quorumRead(responses, r)` that, given an array of `{ replica, value, version }` responses, waits for at least `r` of them and returns the value with the highest version plus the list of replicas that need read repair.
<details><summary>Answer</summary>

```js
function quorumRead(responses, r) {
  if (responses.length < r) throw new Error(`only ${responses.length} of ${r} replicas responded`);
  const newest = responses.reduce((a, b) => (b.version > a.version ? b : a));
  const stale = responses.filter((x) => x.version < newest.version).map((x) => x.replica);
  return { value: newest.value, version: newest.version, repair: stale };
}
console.log(quorumRead([{ replica: 'n1', value: 'pro', version: 7 }, { replica: 'n2', value: 'free', version: 6 }], 2));
// { value: 'pro', version: 7, repair: [ 'n2' ] }
```
</details>

**22.** Code — Write `lagAlert(samples, { maxLagSec, forSamples })` that returns `true` when the last `forSamples` replication-lag samples (in seconds) all exceed `maxLagSec`.
<details><summary>Answer</summary>

```js
function lagAlert(samples, { maxLagSec = 10, forSamples = 3 } = {}) {
  if (samples.length < forSamples) return false;
  // Several consecutive breaches: one slow sample during a long transaction shouldn't page anyone
  return samples.slice(-forSamples).every((s) => s > maxLagSec);
}
console.log(lagAlert([1, 12, 15, 30], { maxLagSec: 10, forSamples: 3 }), lagAlert([1, 12, 3, 30])); // true false
```
</details>

**23.** Scenario — A financial application requires that no acknowledged transaction is ever lost, with a p99 commit latency under 20 ms. It runs in one region across three zones and must also survive a full region loss. Design the replication.
<details><summary>Answer</summary>Within the region, single-leader with at least one synchronous follower in another zone (semi-sync/quorum commit): ~1–2 ms extra per commit, zero loss for zone failures, and failover promotes only a confirmed follower. Across regions, synchronous replication would break the 20 ms budget, so replicate asynchronously and state the cross-region RPO (seconds) explicitly to the business. If region loss must also be zero-loss, the latency requirement must change — or use a consensus-based distributed database with replicas in nearby regions, accepting higher commit latency.</details>

**24.** Scenario — A product team wants to add read replicas to cut primary load, which is 70% reads. Leadership worries about "stale data bugs". How do you roll this out safely?
<details><summary>Answer</summary>Classify reads by required guarantee: reads of a user's own recently changed data need read-your-writes; reads that feed writes (check-then-act, balances) stay on the primary; everything else (browsing, search, reports) can tolerate lag. Implement token-based routing for the first class, keep the second on the primary, and move the third to replicas gradually behind a flag, monitoring lag and the primary-fallback rate. Set a lag threshold above which a replica is removed from rotation.</details>

---

## Chapter Summary

- Replication serves durability, availability, read scaling, and latency, and all its difficulty is in
  propagating changes: who accepts writes, when they're acknowledged, and what readers see in between.
- Asynchronous replication is fast but loses the lag's worth of writes on failover; one synchronous
  follower in another zone gives zero-loss for acknowledged writes without stalling on every follower.
- Replication lag causes read-your-writes, monotonic-read, and consistent-prefix anomalies; log-position
  tokens fix the first two exactly, without guessed time windows.
- Failover risks lost writes, reused identifiers, and split brain; fencing, epochs, and consensus-based
  leases are what make it safe, and it must be rehearsed.
- Multi-leader replication buys local writes everywhere at the cost of conflicts; avoid them by giving
  each record a home leader, and treat last-write-wins as data loss.
- In leaderless systems W + R > N guarantees overlap, not linearisability; concurrent writes, failed
  writes, and sloppy quorums all produce reads quorum arithmetic doesn't prevent.

---

## Practice Exercises

1. Set up PostgreSQL with one synchronous and one asynchronous replica (containers are fine), and
   measure commit latency with `synchronous_commit` set to `on`, `remote_write`, and `off`.
2. On the same setup, write a script that updates a row on the primary and immediately reads it from
   the async replica in a loop, recording how often the read is stale and by how much.
3. Read the documentation on consistency levels for Cassandra (or DynamoDB's consistent reads) and map
   each level to N, W, and R.
4. Kill the primary in your test cluster mid-load and measure how many acknowledged writes are missing
   after promoting the async replica versus the sync one.
5. Implement the §13.9 router against real PostgreSQL using `pg_current_wal_lsn()` and
   `pg_last_wal_replay_lsn()`, and measure the fraction of reads that fall back to the primary under
   load.

---

## Mini Project: A Replication Simulator

Build a simulator for a replicated key-value store that supports single-leader (sync/async) and
leaderless (configurable N, W, R) modes, with injected faults, so you can watch each anomaly and
guarantee in this chapter happen.

- Nodes with independent, configurable replication delays; a network that can drop, delay, or
  partition messages.
- Single-leader mode with failover driven by a timeout, with and without fencing, reporting lost
  writes and split-brain periods.
- Leaderless mode with read repair, hinted handoff (strict and sloppy quorums), and a Merkle-tree
  anti-entropy pass.
- A client model that checks read-your-writes and monotonic reads per session and counts violations.
- A report table comparing configurations under the same fault schedule.

---

**Next:** [Chapter 14 — Partitioning →](14-partitioning.md)
