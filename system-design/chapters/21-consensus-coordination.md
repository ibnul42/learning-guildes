# Chapter 21 — Consensus, Leases, and Coordination

## Learning Objectives

- Explain what consensus guarantees and why it needs a majority of nodes
- Walk through Raft's leader election and log replication, including the rules that keep it safe
- Use a coordination service (etcd, ZooKeeper) for leader election, configuration, and membership — and know its limits
- Implement leases correctly, and make them safe with fencing tokens
- Decide when a distributed lock is the right tool, and when a constraint, partitioning, or idempotency is better

---

## 21.1 The Problem Consensus Solves

Chapters 13, 19, and 20 kept arriving at the same requirement: several nodes must **agree** on
something — who the leader is, whether a write is committed, who holds a lock, what the cluster's
membership is — even though nodes crash, messages are lost, and nobody can tell slow from dead.

**Consensus** is the problem of getting a group of nodes to agree on a value (or, in practice, on a
sequence of values — a replicated log) such that:

- **Agreement**: no two nodes decide differently.
- **Validity**: the decided value was actually proposed by someone.
- **Integrity**: a node decides at most once — decisions are never revoked.
- **Termination**: every non-crashed node eventually decides — *if* the network behaves well enough for
  long enough (Chapter 19's partial synchrony; FLP rules out a guarantee in fully asynchronous systems).

The first three are **safety** properties and hold no matter what; the last is **liveness** and may pause
during bad periods. That split is exactly Chapter 19's design principle, built into an algorithm.

### Why a majority

Practical consensus algorithms make decisions by **majority quorum**: a value is decided once more than
half the nodes have accepted it. The reason is that **any two majorities overlap** in at least one node.
If one majority decided value X, any later majority must include a node that knows about X, so it can't
decide something contradictory. This gives the familiar sizing rule:

| Cluster size | Majority | Failures tolerated |
|---|---|---|
| 3 | 2 | 1 |
| 4 | 3 | 1 — no better than 3, and slower |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

Odd sizes are standard: a fourth node adds cost and a vote without adding tolerance. And during a
partition, only the side with a majority can make progress; the minority side **must** stop deciding —
which is CAP's consistency choice (Chapter 20), made deliberately.

**State machine replication** is how consensus becomes useful: if every replica applies the same commands
in the same order from a replicated log, every replica computes the same state. A linearizable key-value
store, a database's replication log, a lock service — all are a deterministic state machine fed by a
consensus-ordered log.

---

## 21.2 Raft

**Raft** was designed to be understandable, and it's now the most widely deployed consensus algorithm —
etcd, Consul, CockroachDB, TiKV, YugabyteDB, Kafka's KRaft, and many more. It splits consensus into leader
election, log replication, and safety rules.

**Terms.** Time is divided into numbered **terms**. Each term begins with an election and has at most one
leader. Terms act as a logical clock: any message carrying an older term is stale and ignored, and a node
that sees a newer term immediately becomes a follower. Terms are what make "the old leader came back" safe.

**Leader election.** Followers expect periodic heartbeats from the leader. A follower that hears nothing for
an **election timeout** — randomised, typically 150–300 ms in the original design and longer in practice —
becomes a **candidate**, increments the term, votes for itself, and asks the others for votes. Each node
grants at most one vote per term. A candidate with a majority becomes leader. Randomised timeouts make split
votes (several simultaneous candidates, none with a majority) rare and self-resolving: the next round's
random timeouts almost always produce a single candidate first.

**Log replication.** Clients send commands to the leader, which appends them to its log and sends them to
followers. An entry is **committed** once a majority has stored it; the leader then applies it to its state
machine, replies to the client, and tells followers the new commit index so they apply it too. A follower
whose log diverges (left over from a previous leader) has the conflicting suffix overwritten with the
leader's entries.

```text
          term 3 leader
client ──▶ [1|1|2|3|3]  ── append ──▶  follower A [1|1|2|3|3]   ✓ stored
                         ── append ──▶  follower B [1|1|2|3]     ✓ stored (up to 4)
                         ── append ──▶  follower C [1|1]          lagging
                         ── append ──▶  follower D [1]            lagging
entries 1–4 are on a majority (leader, A, B) → commit index 4
```

**Safety rules.** Two rules keep committed entries from ever being lost, and both fit in a few lines:

```js
// Two of Raft's core rules, as pure functions.

// 1. Commit rule: the leader may commit an index once a MAJORITY of the cluster (itself included)
//    has stored it — and, for safety, only entries from its own current term are committed this way.
function commitIndex({ matchIndex, log, currentTerm, previousCommit }) {
  const sorted = [...matchIndex].sort((a, b) => b - a);   // highest stored index per node, descending
  const majorityIndex = sorted[Math.floor(sorted.length / 2)]; // stored by at least a majority
  for (let n = majorityIndex; n > previousCommit; n--) {
    if (log[n - 1].term === currentTerm) return n;        // older-term entries commit indirectly
  }
  return previousCommit;
}

// 2. Voting rule: grant a vote only to a candidate whose log is at least as up-to-date as ours,
//    so no node missing a committed entry can ever become leader.
function grantVote({ me, candidate }) {
  if (candidate.term < me.currentTerm) return false;       // stale candidate
  if (candidate.term === me.currentTerm && me.votedFor && me.votedFor !== candidate.id) return false;
  const upToDate = candidate.lastLogTerm > me.lastLogTerm ||
    (candidate.lastLogTerm === me.lastLogTerm && candidate.lastLogIndex >= me.lastLogIndex);
  return upToDate;
}

const log = [{ term: 1 }, { term: 1 }, { term: 2 }, { term: 3 }, { term: 3 }];
// Five nodes; the leader (term 3) has all 5 entries; the others have stored up to 5, 4, 2, 1
console.log(commitIndex({ matchIndex: [5, 5, 4, 2, 1], log, currentTerm: 3, previousCommit: 2 })); // 4
// A node whose log ends at (term 3, index 5) is asked to vote for one ending at (term 2, index 9)
console.log(grantVote({ me: { currentTerm: 4, votedFor: null, lastLogTerm: 3, lastLogIndex: 5 },
  candidate: { id: 'n2', term: 4, lastLogTerm: 2, lastLogIndex: 9 } })); // false
```

The voting rule is the crux: a committed entry is on a majority; a winning candidate needs votes from a
majority; the two majorities overlap; and the overlapping node refuses to vote for a candidate missing that
entry. A *longer* log from an *older* term (the candidate above has 9 entries, but from term 2) loses to a
shorter log from a newer term. The commit rule's restriction to current-term entries closes a subtler gap
in which an old-term entry on a majority could still be overwritten.

> 💼 **Interview** — "How does Raft prevent a stale node from becoming leader?" The answer is the vote
> restriction: voters reject candidates whose last log entry is from an older term, or the same term but
> shorter. Combined with majority quorums for both commits and votes, every new leader is guaranteed to
> hold every committed entry.

**Cost.** Each write needs one round trip from the leader to a majority, plus an `fsync` on each — about a
millisecond or two within a region, tens to hundreds of milliseconds across regions. Reads that must be
linearizable need the leader to confirm it's still leader (a heartbeat round to a majority, or a time-based
leader lease, which then depends on bounded clock drift). Throughput is limited by the single leader, so
large systems run **many Raft groups**, one per data partition (Chapter 14), each with its own leader.

---

## 21.3 Paxos and the Family

**Paxos**, published by Leslie Lamport, is the original practical consensus protocol. Basic Paxos agrees on a
single value through two phases (prepare/promise, then accept/accepted) with proposal numbers playing the role
Raft's terms play; **Multi-Paxos** optimises a sequence of decisions by electing a stable leader that skips the
first phase. Paxos is famously hard to implement faithfully from the paper, which is why Raft was written.
**ZAB** (ZooKeeper) and **Viewstamped Replication** are close relatives. Under the differences, they share the
same structure: a leader per epoch, majority quorums, epoch numbers that fence off old leaders, and an
election rule that makes new leaders inherit everything committed.

The practical guidance is unambiguous: **don't implement consensus yourself.** The algorithms are subtle,
the published descriptions omit hard engineering details (membership changes, log compaction and snapshots,
disk-failure handling), and correctness bugs appear only under rare interleavings. Use a well-tested library
(etcd's Raft, HashiCorp's Raft) or, more often, a service that already runs consensus for you.

---

## 21.4 Coordination Services

A **coordination service** — etcd, Apache ZooKeeper, Consul — is a small, strongly consistent, highly available
key-value store built on consensus, with primitives designed for coordinating other systems:

| Primitive | etcd | ZooKeeper | Used for |
|---|---|---|---|
| Linearizable reads and writes | Yes (default) | Writes; `sync` before a read for linearizability | Config, metadata, the source of truth for "who's in charge" |
| Compare-and-set / transactions | `Txn` with conditions on revisions | Versioned `setData`, `multi` | Safe updates, leader claims |
| Leases / sessions | Leases with TTLs, attached to keys | Session-bound ephemeral znodes | Membership, locks, leaders — keys vanish when holders die |
| Watches | Watch a key or prefix from a revision | One-shot watches | Reacting to config and membership changes |
| Monotonic revisions | Global revision per write | `zxid`, per-node versions | Fencing tokens, ordering |

Kubernetes stores all cluster state in etcd; Kafka (before KRaft) used ZooKeeper for controller election and
metadata; many databases and schedulers use one of them for leader election and partition maps (Chapter 14).

Coordination services are deliberately **small and slow**: datasets in megabytes to low gigabytes, write rates
in the low thousands per second, every write going through consensus. Treat them as the place where a *small
amount of critical metadata* lives — who leads each partition, which nodes are members, which config version is
current — and never as a general database or a high-volume queue. A busy application writing per-request data
into etcd will degrade every system that depends on it, often including the cluster scheduler.

---

## 21.5 Leases and Leader Election in Applications

Most applications don't need consensus directly; they need **one instance to do something** — run the scheduler,
consume a partition, own a shard, send the nightly invoices. The standard tool is a **lease**: a lock with a
time limit, granted by a coordination service (or a database row with an expiry), that the holder must renew
before it expires.

```text
instance A: acquire lease "scheduler" (TTL 15 s) ✓ → token 41
instance A: renew every 5 s ✓ ✓ ✓
instance A: crashes                                      … lease expires at t + 15 s
instance B: acquire lease "scheduler" ✓ → token 42  → B is now the scheduler
```

Leases solve the crash case — a dead holder's lease expires and someone else takes over — without anyone having
to decide the holder is dead. Their safety depends on details:

- **The holder must stop before its lease expires**, by its own clock, with a margin for clock drift. If the
  lease is 15 seconds, stop acting at 12 unless renewed. Use the monotonic clock (Chapter 19) to measure.
- **Renewal must be frequent enough** to survive a missed renewal or two (renew at a third of the TTL).
- **A renewal failure means "you are no longer the leader"** — stop work immediately; don't wait to be told.
- **And none of that is enough**, because of process pauses: a holder can pass every check, pause for 20
  seconds, and resume believing it still holds the lease. Chapter 19 replayed that bug. The fix is not on the
  holder's side at all.

---

## 21.6 Fencing Tokens

A **fencing token** is a number, strictly increasing with each lease grant, that the holder attaches to every
operation it performs on the protected resource. The **resource** remembers the highest token it has seen and
rejects any operation with a lower one. A paused former holder resumes and writes with its old token — and is
refused, because the new holder has already written with a higher one.

```text
lock service: grant to A, token 33 ───────────────── (A pauses) ─────────── A resumes: write(token 33) ✗ rejected
lock service: (A's lease expires) grant to B, token 34 ── B: write(token 34) ✓ — storage now requires ≥ 34
```

Three things make fencing work:

1. **Tokens come from the consensus system**, so they're strictly monotonic: etcd's revision of the lease key,
   ZooKeeper's `zxid` or the znode version, a Raft term.
2. **The resource enforces them.** A database can enforce a token with a conditional write
   (`UPDATE … WHERE fence_token <= $token`, then store `$token`); object stores with conditional writes can
   compare a token stored in metadata; a service can track the highest token per resource. If the resource
   can't check tokens, fencing isn't possible and the lock only provides best-effort exclusion.
3. **Every write path carries the token.** One write path that skips it reopens the hole.

The real-world example in §21.9 implements a lease lock that issues tokens and a store that enforces them,
replaying Chapter 19's pause scenario to show the stale write rejected. Raft terms are the same idea internally:
a deposed leader's messages carry an old term, and every node rejects them.

---

## 21.7 Distributed Locks, Honestly

Distributed locks are requested far more often than they're needed, and implemented unsafely more often than
safely. Before adding one, separate two purposes:

| Purpose | Consequence of two holders at once | Requirements |
|---|---|---|
| **Efficiency** — avoid duplicate work (two instances both sending the same report) | Some wasted work, maybe a duplicate email | Any reasonable lease; Redis `SET key value NX PX ttl` is fine |
| **Correctness** — protect an invariant (two writers corrupt a file, double-spend) | Data corruption | Consensus-backed lease **plus fencing tokens enforced by the resource** |

Locks built on a single Redis node, or on algorithms that rely on timing assumptions across several Redis nodes
(the much-debated Redlock), are adequate for efficiency and inadequate for correctness: they can't survive
process pauses or clock jumps without fencing, and they don't provide tokens the resource can check. If you need
correctness, use a consensus-backed lease *and* fence the resource.

Often, the better design avoids the lock:

- **Let the database enforce the invariant.** A unique constraint, a conditional update, or a serializable
  transaction (Chapter 15) protects data without any external lock.
- **Make the operation idempotent.** If running twice is harmless (an idempotency key, a deterministic output
  key), duplicate execution stops mattering, and an efficiency lock suffices.
- **Partition the work so it has one owner by construction.** Kafka assigns each partition to one consumer in a
  group; a scheduler assigns each job to one worker by hash. Single-writer designs don't need locks — they need
  ownership changes to be fenced, which the partitioning system's epochs provide.
- **Use a queue with visibility timeouts** for "exactly one worker processes each item", plus idempotent
  processing for the rare redelivery (Chapter 24).

> ⚠️ **Gotcha** — **A lock whose TTL is shorter than the work it protects isn't a lock.** If a job takes 90
> seconds and the lease is 30 seconds with no renewal, a second worker starts at 30 seconds while the first is
> still running. Either renew the lease throughout the work (and stop if renewal fails), size the TTL well
> above the worst-case duration, or — best — make the work idempotent and fenced so overlap is harmless.

---

## 21.8 Common Mistakes and Best Practices

**Mistake: implementing consensus from a paper.** The published algorithms omit membership changes, snapshots,
and disk failures; correctness bugs hide in rare interleavings. Use a proven implementation.

**Mistake: even-sized consensus clusters.** Four nodes tolerate no more failures than three and need a larger
majority.

**Mistake: application data in the coordination service.** etcd and ZooKeeper are for small, critical metadata;
high write volumes degrade everything that depends on them.

**Mistake: leases without fencing for correctness-critical work.** A paused holder resumes and acts on a lease it
no longer holds.

**Mistake: fencing tokens the resource doesn't check.** Generating tokens is pointless unless every write path is
rejected for a stale one.

**Mistake: ignoring renewal failures.** A holder that can't renew must assume it has lost the lease and stop
immediately.

**Mistake: a lock when a constraint would do.** External locks add a dependency and failure modes that a unique
index or conditional update avoids.

**Best practice: separate efficiency locks from correctness locks.** Simple leases for the first; consensus-backed
leases with enforced fencing for the second.

**Best practice: prefer single-owner partitioning and idempotency.** Work that has one owner by construction, or is
harmless to repeat, rarely needs a lock at all.

**Best practice: odd-sized clusters across failure domains.** Three or five members, one per zone, so a zone loss
leaves a majority.

---

## 21.9 Real-World Example: A Fenced Lease Lock

This is the minimum safe shape for a correctness-critical lock: a lease service that hands out strictly
increasing fencing tokens, and a resource that enforces them. The demo replays Chapter 19's scenario — node A
checks its lease, pauses past expiry, node B acquires the lease and writes, and node A resumes and tries to
write — and shows A's stale write rejected.

```js
// fenced-lock.mjs — a lease-based lock that hands out fencing tokens, and a storage service that
// enforces them. A holder that pauses past its lease can't corrupt anything: its token is stale.

export class LeaseLock {
  #holder = null; #expiresAt = 0; #token = 0;
  constructor({ ttlMs = 10_000, now = Date.now } = {}) { this.ttlMs = ttlMs; this.now = now; }

  acquire(client) {
    if (this.#holder && this.now() < this.#expiresAt && this.#holder !== client) return null;
    this.#holder = client;
    this.#expiresAt = this.now() + this.ttlMs;
    return { token: ++this.#token, expiresAt: this.#expiresAt }; // strictly increasing per grant
  }

  renew(client, token) {
    if (this.#holder !== client || token !== this.#token || this.now() >= this.#expiresAt) return false;
    this.#expiresAt = this.now() + this.ttlMs;
    return true;
  }
}

// The resource checks tokens. This is the part that makes the lock safe; the lease alone isn't.
export class FencedStore {
  #highestToken = 0; #data = new Map();
  write(key, value, token) {
    if (token < this.#highestToken) {
      throw Object.assign(new Error(`stale token ${token} < ${this.#highestToken}`), { code: 'FENCED' });
    }
    this.#highestToken = token;
    this.#data.set(key, value);
  }
  read(key) { return this.#data.get(key); }
}

// ── Chapter 19's pause scenario, replayed with fencing ─────────────────────────────
let t = 0;
const lock = new LeaseLock({ ttlMs: 10_000, now: () => t });
const store = new FencedStore();
const results = [];
const tryWrite = (who, token, value) => {
  try { store.write('report.csv', value, token); results.push(`${who}(token ${token}) wrote`); }
  catch (e) { results.push(`${who}(token ${token}) rejected: ${e.message}`); }
};

const a = lock.acquire('node-a');                 // t = 0: node-a holds token 1 until t = 10 s
t = 9_000;                                        // node-a checks its lease: still valid…
t = 12_000;                                       // …then pauses for 3 s (GC) and the lease expires
const b = lock.acquire('node-b');                 // t = 12 s: node-b gets token 2
tryWrite('node-b', b.token, 'fresh data');
tryWrite('node-a', a.token, 'stale data');        // node-a resumes and writes with its old token
console.log(results.join('\n'));
console.log('final:', store.read('report.csv'), '| node-a can renew:', lock.renew('node-a', a.token));
// node-b(token 2) wrote
// node-a(token 1) rejected: stale token 1 < 2
// final: fresh data | node-a can renew: false
```

In production the `LeaseLock` role is played by etcd or ZooKeeper (the token is the lease key's revision or the
znode's version), and the `FencedStore` role by whatever the lock protects — a database row with a `fence_token`
column checked in the `UPDATE`, or a service that tracks the highest token per resource.

The decision to defend is **putting the safety check in the resource rather than in the lock holder**. It's
tempting to make the holder more careful instead: re-check the lease right before every write, shorten the TTL,
add a safety margin. None of it works, because the dangerous pause can occur *after* the last check and *before*
the write reaches the resource — inside a library call, a GC cycle, or the network stack — and no code running
on the holder can detect a pause it was frozen during. Only the resource sees the writes in the order they
actually arrive, so only the resource can tell that a write comes from a holder that has already been
superseded. A lock without an enforcing resource reduces the probability of overlap; a fenced resource makes
overlap harmless.

---

## 21.10 Questions

**1.** MCQ — How many node failures can a five-node Raft cluster tolerate while still making progress? a) 1 b) 2 c) 3 d) 4
<details><summary>Answer</summary>**b)** A majority of 3 must remain.</details>

**2.** MCQ — Why are consensus clusters usually odd-sized? a) Votes can't tie b) An extra node to make it even adds no failure tolerance but raises the majority c) Raft requires it d) Leases need it
<details><summary>Answer</summary>**b)** Four nodes need 3 for a majority and tolerate 1 failure — the same as three nodes, with more coordination.</details>

**3.** MCQ — What makes Raft's split votes rare? a) A fixed leader b) Randomised election timeouts c) Majority quorums d) Fencing tokens
<details><summary>Answer</summary>**b)** Nodes time out at different random moments, so one candidate usually starts and wins before others begin.</details>

**4.** MCQ — When is a Raft log entry committed? a) When the leader appends it b) When every follower stores it c) When a majority has stored it (and it's from the leader's current term, or precedes one that is) d) When the client acknowledges it
<details><summary>Answer</summary>**c)** Majority replication commits it; the current-term restriction closes an edge case with entries from older terms.</details>

**5.** MCQ — What is a coordination service like etcd good for? a) Storing user sessions at high volume b) A work queue c) Small, critical metadata: leader election, membership, configuration d) Analytics
<details><summary>Answer</summary>**c)** It runs every write through consensus and is sized for megabytes of metadata at modest write rates.</details>

**6.** MCQ — What must enforce a fencing token for it to provide safety? a) The lock holder b) The lock service c) The protected resource d) The load balancer
<details><summary>Answer</summary>**c)** Only the resource sees writes in arrival order and can reject a superseded holder's write.</details>

**7.** Short — Why do any two majorities of a cluster overlap, and why does consensus depend on it?
<details><summary>Answer</summary>Two sets each larger than half can't be disjoint — together they'd exceed the cluster. So any majority that elects a leader or commits an entry includes at least one node that knows about every earlier committed decision, preventing contradictory decisions.</details>

**8.** Short — Explain Raft's vote restriction and what it prevents.
<details><summary>Answer</summary>A node grants its vote only if the candidate's last log entry has a higher term, or the same term and an index at least as high. Since committed entries are on a majority and winning needs a majority, the overlapping voter refuses any candidate missing a committed entry — so no leader can lose committed data.</details>

**9.** Short — What's the difference between an efficiency lock and a correctness lock?
<details><summary>Answer</summary>An efficiency lock avoids duplicate work; occasional double execution only wastes effort. A correctness lock protects an invariant; double execution corrupts data. The latter needs a consensus-backed lease and fencing enforced by the resource.</details>

**10.** Short — Why should a lease holder stop working when a renewal fails, rather than when it's told it lost the lease?
<details><summary>Answer</summary>A failed renewal may mean the lease has expired and been granted elsewhere; the "you lost it" message may never arrive (partition). Stopping on renewal failure, before expiry by its own monotonic clock with a margin, minimises overlap — fencing covers the rest.</details>

**11.** Short — Name two designs that remove the need for a distributed lock.
<details><summary>Answer</summary>Any two of: database constraints or conditional updates enforcing the invariant; idempotent operations so duplicates are harmless; partitioning work so each item has one owner (with fenced ownership changes); a queue with visibility timeouts plus idempotent consumers.</details>

**12.** Output — Using §21.2's `commitIndex`, what does `commitIndex({ matchIndex: [6, 6, 6, 3, 3], log: [{term:1},{term:1},{term:2},{term:2},{term:2},{term:4}], currentTerm: 4, previousCommit: 3 })` return?
<details><summary>Answer</summary>**6.** Sorted descending: [6, 6, 6, 3, 3]; the majority index is 6, and entry 6 is from the current term 4.</details>

**13.** Output — Same function, but `currentTerm: 5` (the leader was elected in term 5 and hasn't appended a term-5 entry yet). What does it return?
<details><summary>Answer</summary>**3 (unchanged).** Entries 4–6 are on a majority but none is from term 5, so the leader can't commit them directly; they'll commit once a term-5 entry reaches a majority.</details>

**14.** Output — Using §21.2's `grantVote`, does a node with `{ currentTerm: 7, votedFor: null, lastLogTerm: 6, lastLogIndex: 12 }` vote for candidate `{ id: 'c', term: 7, lastLogTerm: 6, lastLogIndex: 11 }`?
<details><summary>Answer</summary>**No.** Same last term, but the candidate's log is shorter, so it isn't at least as up to date.</details>

**15.** Output — Using §21.9's classes, client X acquires the lock (token 1) and writes; its lease expires; Y acquires (token 2) but hasn't written yet; X writes again with token 1. Is X's second write accepted?
<details><summary>Answer</summary>**Yes.** The store has only seen token 1 so far; it can only reject tokens lower than the highest *seen*. Once Y writes with token 2, any later token-1 write is rejected. Fencing guarantees no stale write lands *after* a newer holder's write — which is the property that prevents corruption — not that superseded holders can't act at all before the new holder does.</details>

**16.** Debug — A three-node etcd cluster is spread across two zones: two nodes in zone A, one in zone B. Zone A loses power and the whole control plane stops. Why, and how should it be laid out?
<details><summary>Answer</summary>Losing two of three members loses the majority, so etcd can't elect a leader or accept writes. Place one member per zone across three zones, so any single zone's loss leaves a majority of two.</details>

**17.** Debug — Two instances of a "singleton" billing job both ran last night, charging some customers twice. The job takes a Redis lock with a 60-second TTL and runs for about 4 minutes. What happened, and how do you fix it?
<details><summary>Answer</summary>The lock expired after 60 seconds while the first run continued, so the second instance acquired it. Renew the lease during the work and stop on renewal failure — but primarily make charging idempotent (idempotency keys per customer per billing period) so overlap can't double-charge, and use a fenced, consensus-backed lease if the lock is meant to protect correctness.</details>

**18.** Debug — After the team started writing per-request audit records into etcd, Kubernetes API calls became slow and occasionally timed out. Why?
<details><summary>Answer</summary>Every etcd write goes through consensus with an `fsync` on a majority; high-volume application writes compete with Kubernetes' own state updates and grow the keyspace and compaction load. Move audit records to a database or log pipeline, and keep etcd for small control-plane metadata.</details>

**19.** Debug — A leader-election library issues fencing tokens, but a stale leader still overwrote data during a partition. Every write goes through the service's `save()` method, which checks the token — except a nightly batch path that writes directly to the database. What's the lesson?
<details><summary>Answer</summary>Fencing is only as strong as its least-protected write path. Enforce the token in the database itself (a conditional update on a `fence_token` column) or route every write — including batch paths — through the checking layer.</details>

**20.** Code — Write `majority(n)` and `faultsTolerated(n)` for a consensus cluster of size `n`.
<details><summary>Answer</summary>

```js
const majority = (n) => Math.floor(n / 2) + 1;
const faultsTolerated = (n) => n - majority(n);
console.log([3, 4, 5, 7].map((n) => `${n}:${majority(n)}/${faultsTolerated(n)}`).join(' ')); // 3:2/1 4:3/1 5:3/2 7:4/3
```
</details>

**21.** Code — Write a SQL-backed fenced update helper `fencedUpdate(client, { id, value, token })` that writes only if the row's stored `fence_token` is not greater than `token`, storing the new token, and throws `FENCED` otherwise.
<details><summary>Answer</summary>

```js
async function fencedUpdate(client, { id, value, token }) {
  const { rowCount } = await client.query(
    'UPDATE resources SET value = $1, fence_token = $2 WHERE id = $3 AND fence_token <= $2',
    [value, token, id]);
  if (rowCount === 0) throw Object.assign(new Error(`write with stale token ${token}`), { code: 'FENCED' });
}
```
The check and the write are one atomic statement, so no pause can slip between them.</details>

**22.** Code — Write `runWhileLeader(lease, work, { renewEveryMs })` that runs `work(signal)` while renewing the lease periodically, and aborts the signal (stopping the work) as soon as a renewal fails.
<details><summary>Answer</summary>

```js
async function runWhileLeader(lease, work, { renewEveryMs = 3_000 } = {}) {
  const controller = new AbortController();
  const timer = setInterval(async () => {
    const ok = await lease.renew().catch(() => false);
    if (!ok) controller.abort(new Error('lease lost'));   // stop immediately; don't wait to be told
  }, renewEveryMs);
  try { return await work(controller.signal); }
  finally { clearInterval(timer); }
}
```
`work` must check `signal.aborted` (or pass the signal to its I/O) — and still use fencing tokens, because abort is cooperative and a paused process won't notice it in time.</details>

**23.** Scenario — A team wants to run their own three-node Raft cluster, implemented from the paper, to replicate configuration for their service. What do you advise?
<details><summary>Answer</summary>Don't implement it: membership changes, snapshots, disk-failure handling, and rare-interleaving bugs make home-grown consensus a long-term liability. Store the configuration in an existing coordination service (etcd, Consul) or a managed equivalent, use watches for change notification, and spend the effort on making clients handle the service being briefly unavailable (static stability, Chapter 4).</details>

**24.** Scenario — A job scheduler must ensure each scheduled job runs exactly once across a fleet of 20 workers. The proposal is a distributed lock per job. What design do you propose instead?
<details><summary>Answer</summary>Store due jobs in a database and have workers claim them with an atomic conditional update (`UPDATE jobs SET owner = $me, lease_until = now() + interval '60 s' WHERE id = $id AND (owner IS NULL OR lease_until < now())`), heartbeat the lease while running, and make job execution idempotent with a per-run key so a reclaimed job that was actually still running causes no duplicate effect. Ownership comes from the database's atomicity, and fencing from a run version column. Chapter 45 builds this scheduler.</details>

---

## Chapter Summary

- Consensus makes a group agree on values — in practice a replicated log — with safety that always holds and
  progress whenever the network behaves; majority quorums work because any two majorities overlap.
- Raft uses terms, randomised elections, majority-committed log entries, and a vote restriction that guarantees
  every new leader holds every committed entry; large systems run one Raft group per partition.
- Don't implement consensus: use a proven library, or more often a coordination service such as etcd or
  ZooKeeper for small, critical metadata — never for application data.
- Leases let one instance act on behalf of a group, but process pauses make a holder's belief in its lease
  unreliable; stop on renewal failure and measure with monotonic clocks.
- Fencing tokens from the consensus system, enforced by the protected resource on every write path, are what
  make leases safe for correctness.
- Distinguish efficiency locks from correctness locks, and prefer constraints, idempotency, and single-owner
  partitioning, which often remove the need for a lock entirely.

---

## Practice Exercises

1. Run a three-node etcd cluster in containers, kill the leader, and measure how long writes are unavailable;
   repeat with different election timeouts.
2. Use etcd's lease API (or ZooKeeper's ephemeral nodes) to implement leader election for a small service, and
   verify that killing the leader hands leadership to another instance.
3. Read the Raft paper's sections on membership changes and log compaction, and summarise why each is harder
   than it first appears.
4. Find a distributed lock in a codebase you know, classify it as efficiency or correctness, and check whether
   the protected resource enforces fencing.
5. Extend §21.9's demo with a database-backed `FencedStore` using the conditional update from Q21, and test it
   with two processes and an injected pause.

---

## Mini Project: A Raft Simulator

Implement Raft's leader election and log replication in a deterministic simulator (reuse the network simulator
from Chapter 19's mini project) and verify its safety properties under faults.

- Nodes with terms, randomised election timeouts, `RequestVote` and `AppendEntries`, commit index tracking, and a
  key-value state machine.
- Fault injection: message loss and delay, partitions, node crashes and restarts with persisted term, vote, and log.
- Invariant checks after every step: at most one leader per term; committed entries never lost or changed; all
  state machines apply the same commands in the same order.
- A client that issues writes and linearizable reads, with histories checked by Chapter 20's checker.
- A seeded fuzzer that runs thousands of fault schedules and reports any invariant violation with its seed.

---

**Next:** [Chapter 22 — Clocks, Ordering, and Unique IDs →](22-clocks-ordering-ids.md)
