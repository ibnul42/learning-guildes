# Chapter 20 — Consistency Models

## Learning Objectives

- Define linearizability precisely and recognise histories that violate it
- Distinguish linearizable, sequential, causal, session, and eventual consistency by what each promises
- Identify the operations that genuinely require linearizability, and the ones that don't
- State the CAP theorem correctly, avoid its common misreadings, and use PACELC for the everyday trade-off
- Choose a consistency model per operation rather than per system

---

## 20.1 What a Consistency Model Is

Once data is replicated (Chapter 13) and the network is unreliable (Chapter 19), different readers
can see different versions of the same data at the same moment. A **consistency model** is the
contract that says which results a read is allowed to return, given the writes that have happened.
It's a promise from the storage system to the code that uses it — and every promise has a price in
latency and availability.

Models form a spectrum, from strongest (fewest allowed outcomes, easiest to program against, most
expensive) to weakest:

```text
strongest ──────────────────────────────────────────────────────────────▶ weakest
linearizable   sequential   causal   session guarantees   eventual
(one copy,     (one order,  (cause   (read-your-writes,   (converges
 real time)     no real      before   monotonic reads…)    eventually,
                time)        effect)                       no ordering promise)
```

Two cautions before going further. First, these are *single-object* or *ordering* guarantees — about
what reads see — and are distinct from *transaction isolation* (Chapter 15), which is about
multi-object operations interleaving. The strongest combined guarantee, **strict serializability**,
is serializable transactions that also respect real-time order. Second, the word "consistency" is
overloaded: the C in ACID (application invariants), the C in CAP (linearizability), and "eventual
consistency" (convergence) are three different ideas.

---

## 20.2 Linearizability

**Linearizability** (also called atomic or strong consistency) makes a replicated system behave as if
there were **one copy of the data**, and every operation took effect atomically at some instant
between when it started and when it finished. Two consequences make it precise:

1. **Real-time order is respected.** If operation A *finished* before operation B *started* (by any
   observer's clock), B must see A's effect.
2. **Once any read returns a new value, all later reads return it (or something newer).** Time never
   goes backwards for anyone, across clients.

```text
                 the write takes effect at some instant inside its interval
client W   ├────── write x = 1 ──────┤
client A        ├─ read x → 1 ─┤                          (concurrent: 0 or 1 both allowed)
client B                               ├─ read x → 0 ─┤   ✗ not linearizable: the write finished
                                                             before this read started
```

Operations that **overlap in time** are concurrent, and either order is allowed for them; it's only
non-overlapping operations whose order is fixed. That's why a read concurrent with a write may return
either value — but once *some* reader has seen the new value, no reader that starts later may see the
old one. The real-world example in §20.9 is a checker that decides, for a recorded history, whether any
valid "single copy" ordering exists.

### What actually needs linearizability

Linearizability is expensive, so it's worth knowing exactly which operations need it:

| Operation | Why it needs linearizability |
|---|---|
| **Leader election and locks** | Every node must agree on the single current holder; a stale read means two leaders (Chapter 21) |
| **Uniqueness constraints** | "Claim username `ana` if free" is a compare-and-set; two replicas each saying "free" means two owners |
| **Compare-and-set / conditional writes** | Account balance ≥ amount, then debit; seat free, then hold |
| **Cross-channel timing dependencies** | A user uploads an image then sends its ID via a queue; the consumer must be able to read it |
| **Configuration that gates safety** | Fencing tokens, feature kill switches that must take effect immediately |

What *doesn't* need it is most reads: a product page, a like count, a feed, a search result. Those can
tolerate staleness measured in seconds, and paying linearizability's cost for them is waste.

### What it costs

To be linearizable, a read must be sure no newer write exists anywhere — so it must either go to the
single leader (and the leader must be sure it's still the leader, which takes a quorum check or a lease),
or read from a quorum *and* coordinate with concurrent writers. Either way, linearizable operations need
**coordination among replicas on the request path**, which means:

- **Latency**: at least one round trip among replicas — about 1 ms within a region, 50–150 ms across
  regions (Chapter 2).
- **Availability**: a replica cut off from the majority can't serve linearizable reads or writes at all
  — which is the CAP theorem (§20.6).

> ⚠️ **Gotcha** — **Quorum reads and writes alone aren't linearizable** (Chapter 13). With W + R > N,
> a read concurrent with a write can return the new value from one replica, and a *later* read can
> return the old value from a different quorum — exactly the "time goes backwards" violation above.
> Leaderless stores need read repair performed synchronously before returning, and even then
> concurrent writes need more. Systems that genuinely need linearizability use consensus (Chapter 21).

---

## 20.3 Sequential and Causal Consistency

**Sequential consistency** requires that all operations appear in *some* single total order consistent
with each client's own program order — but not necessarily with real time. Client A's write may appear
"before" client B's earlier-finished read, as long as everyone agrees on the order. It's weaker than
linearizability and rarely offered as a product guarantee; its main role is in describing CPU memory
models and as a step on the spectrum.

**Causal consistency** is weaker again — and much more useful. It requires only that operations that are
**causally related** are seen by everyone in the same order; operations that are **concurrent** (neither
could have influenced the other) may be seen in different orders by different observers.

"Causally related" means one *happened before* the other in a specific sense: the same client did them in
order, or one read the result of the other, or a chain of those. A reply happens-after the message it
replies to; a comment happens-after the post it comments on; an "unfriend Ana, then post a complaint
about Ana" sequence has the post depending on the unfriending.

Causal consistency forbids the anomalies users find most confusing — seeing a reply before the question,
seeing a post appear for someone you just blocked — while allowing everything else to flow without
coordination. A message can be held until what it depends on has been delivered:

```js
// Causal delivery: hold a message until everything it depends on has been delivered.
// A reply is never shown before the question it answers, whatever order the network chose.
function causalBuffer(deliver) {
  const delivered = new Set(), waiting = [];
  const ready = (m) => m.deps.every((d) => delivered.has(d));
  return function receive(msg) {
    waiting.push(msg);
    let progress = true;
    while (progress) {                       // delivering one message may unblock others
      progress = false;
      for (let i = 0; i < waiting.length; i++) {
        if (ready(waiting[i])) {
          const [m] = waiting.splice(i, 1);
          delivered.add(m.id); deliver(m); progress = true; break;
        }
      }
    }
  };
}
const shown = [];
const receive = causalBuffer((m) => shown.push(m.id));
// The network delivers the answer and a reaction before the question
receive({ id: 'reply', deps: ['question'] });
receive({ id: 'like', deps: ['reply'] });
receive({ id: 'unrelated', deps: [] });
receive({ id: 'question', deps: [] });
console.log(shown); // [ 'unrelated', 'question', 'reply', 'like' ]
```

The key theoretical fact: **causal consistency is (essentially) the strongest model that can remain
available during a network partition.** Each replica can accept writes locally and still respect causal
order, because causality only constrains an operation relative to things its replica has already seen.
That makes it the natural target for geo-distributed and offline-capable systems. Its practical cost is
tracking dependencies — explicitly, as above, or with version vectors and hybrid logical clocks
(Chapter 22).

---

## 20.4 Session Guarantees

Many systems offer neither linearizability nor full causal consistency, but users still need their
*own* experience to make sense. **Session guarantees** are per-client promises — the ones Chapter 13
implemented with log-position tokens:

| Guarantee | Promise to one client | Violation users see |
|---|---|---|
| **Read your writes** | You see your own writes | "I saved it and it's gone" |
| **Monotonic reads** | You never see older data after newer | A comment appears, then vanishes on refresh |
| **Monotonic writes** | Your writes take effect in the order you made them | An address change, then a phone change — the phone change applied first and the address change "lost" |
| **Writes follow reads** | Your writes are ordered after what you'd read | You reply to a comment; some replicas have your reply but not the comment |

Together, the four session guarantees give each client a causally consistent view of its own
interactions. They're cheap — a token per session recording what that client has seen or written — and
they fix the anomalies that generate most "the data is wrong" bug reports. Implement them before
considering anything stronger.

---

## 20.5 Eventual Consistency

**Eventual consistency** promises only that if writes stop, all replicas will *eventually* converge to the
same value. It says nothing about how long that takes, what reads return in the meantime, or the order in
which writes are seen. It's the guarantee of asynchronous replication, DNS, caches, CDNs, and search
indexes — most of the derived data in this book.

Two refinements make it more useful:

- **Bounded staleness** — measure and cap how far behind a replica may be ("no more than 5 seconds or
  1,000 writes"), and stop serving reads from replicas that exceed it. This turns "eventually" into a
  number you can put in an SLO.
- **Strong eventual consistency** — replicas that have received the *same set* of updates are guaranteed
  to be in the *same state*, regardless of order. That requires updates that commute: CRDTs (counters,
  sets, registers, sequences designed to merge deterministically — Chapter 46). It removes the need for
  conflict resolution policies like last-write-wins.

> 💼 **Interview** — "Is eventual consistency acceptable here?" The strong answer quantifies the
> staleness and names the user-visible consequence: "The follower count can be up to a few seconds
> stale; nobody can tell. The account balance shown before a transfer must be read-your-writes for the
> account owner, and the transfer itself must be linearizable on the balance."

---

## 20.6 CAP, Honestly

The **CAP theorem** is the most quoted and most misquoted result in distributed systems. Stated
precisely: in a system that replicates data, **during a network partition** you must choose between:

- **Consistency** — specifically *linearizability*: every read sees the latest write; and
- **Availability** — specifically, *every* request to a non-failed node receives a (non-error) response.

A node cut off from the others either answers — possibly with stale data, or accepting writes the other
side can't see (available, not linearizable) — or refuses until the partition heals (linearizable, not
available). That's all it says.

Misreadings to avoid:

| Misreading | Correction |
|---|---|
| "Pick two of three" | Partitions aren't optional; the choice is only *what to do during one* |
| "CA systems" exist | A system that never tolerates partitions is a single node — or one that silently breaks during them |
| "Our system is CP" or "AP" as a whole | The choice is often per operation, and per configuration (Cassandra with `QUORUM` behaves differently from `ONE`) |
| "Consistency" means ACID consistency or "not corrupt" | CAP's C is linearizability only |
| "Availability" means high uptime | CAP's A is every non-failed node answering — a much stronger, stranger property; a CP system with a majority available is still highly available in the everyday sense |

CAP is narrow because partitions are relatively rare. Most of the time there's no partition, and the
trade-off you face daily is different. **PACELC** extends CAP to capture it: **if** there's a
**P**artition, choose **A**vailability or **C**onsistency; **E**lse, choose **L**atency or
**C**onsistency. Coordination for consistency costs round trips even when the network is fine.

| System (default configuration) | During a partition (PA / PC) | Normally (EL / EC) |
|---|---|---|
| Single-leader SQL, reads from leader | PC — minority side can't write | EC |
| Single-leader SQL, reads from async replicas | PC for writes, PA for replica reads | EL for replica reads |
| Cassandra / Dynamo-style, low consistency levels | PA | EL |
| Cassandra with `QUORUM` | PC-ish (fails without a quorum) | EC-ish (not linearizable without more — §20.2) |
| Spanner, CockroachDB, etcd, ZooKeeper | PC | EC |
| DNS, CDNs, caches | PA | EL |

The useful output of thinking about CAP and PACELC is a set of *operation-level* decisions: which
operations must refuse rather than risk inconsistency during a partition, and which operations are worth
paying coordination latency for all the time.

---

## 20.7 Choosing a Model Per Operation

Real systems mix models, and the design work is assigning each operation the weakest model that's
correct for it:

| Operation | Model | Mechanism |
|---|---|---|
| Acquire a lock, elect a leader | Linearizable | Consensus store (etcd, ZooKeeper) — Chapter 21 |
| Create a unique username | Linearizable on that key | Unique constraint on the leader; conditional put |
| Debit an account balance | Linearizable on that account (plus isolation) | Leader + atomic conditional update — Chapter 15 |
| Show my profile after I edit it | Read your writes | Session token routing — Chapter 13 |
| Show a comment thread | Causal | Dependencies or ordering per thread |
| Show follower counts, like counts | Eventual (bounded) | Async replicas, caches, counters — Chapters 12, 16 |
| Search results, recommendations | Eventual | Derived stores via CDC — Chapters 11, 18 |
| Analytics dashboards | Eventual (minutes) | Warehouse — Chapter 25 |

A useful habit is to write the model into the API or data-access layer explicitly — a `consistency:
'strong' | 'session' | 'eventual'` option on reads, defaulting to session — so that choosing a weak model
is deliberate and visible in code review, rather than an accident of which replica a load balancer picked.

---

## 20.8 Common Mistakes and Best Practices

**Mistake: "strong consistency" everywhere.** Paying coordination latency and partition unavailability for
reads that tolerate seconds of staleness is pure cost.

**Mistake: eventual consistency for invariants.** Uniqueness, balances, and locks checked against stale
replicas allow duplicates, overdrafts, and two leaders.

**Mistake: assuming quorums are linearizable.** Concurrent reads can see new then old values from
different quorums; use consensus for linearizable operations.

**Mistake: "we're CP" as a system-wide label.** The choice is per operation and per configuration; labels
hide the operations that are actually unsafe.

**Mistake: confusing CAP's C with ACID's C.** Linearizability and application invariants are different
properties with different mechanisms.

**Mistake: ignoring session guarantees.** Most user-visible "data is wrong" reports are read-your-writes and
monotonic-read violations, cheaply fixed.

**Mistake: unbounded "eventually".** Staleness that's never measured can quietly grow to hours.

**Best practice: weakest correct model per operation.** Linearizable for coordination and invariants,
session for a user's own data, eventual for everything derived.

**Best practice: make the read model explicit in code.** A visible consistency option per read makes weak
choices deliberate and reviewable.

**Best practice: test consistency claims.** Record histories under faults and check them, as §20.9 does,
instead of trusting documentation.

---

## 20.9 Real-World Example: A Linearizability Checker

The only way to know whether a system is linearizable is to record what clients observed — each
operation's start time, end time, and result — and check whether *some* single-copy ordering explains it
all. Jepsen's checkers do this for real databases under injected faults; this is the core search in
miniature, for a single register. It tries to place operations one at a time, only ever choosing an
operation that no other pending operation strictly precedes in real time, and backtracks when a read's
value can't be explained.

```js
// linearizability.mjs — check whether a history of reads and writes on one register could have
// come from a single copy of the data, with each operation taking effect at some instant between
// its start and its end. This is the core idea behind Jepsen's checkers, in miniature.

export function checkLinearizable(history, initial = null) {
  // history: [{ id, op: 'write' | 'read', value, start, end }] — times as observed by clients
  const ops = history.map((o, i) => ({ ...o, i }));

  function search(remaining, value, order) {
    if (remaining.length === 0) return order;                 // every op placed: a valid linearization
    // An op may go next only if no other pending op finished before it started —
    // otherwise real-time order would be violated
    const earliestEnd = Math.min(...remaining.map((o) => o.end));
    for (const op of remaining.filter((o) => o.start <= earliestEnd)) {
      if (op.op === 'read' && op.value !== value) continue;   // this read can't happen at this point
      const next = op.op === 'write' ? op.value : value;
      const found = search(remaining.filter((o) => o !== op), next, [...order, op.id]);
      if (found) return found;
    }
    return null;                                              // dead end: backtrack
  }

  const order = search(ops, initial, []);
  return order ? { linearizable: true, order } : { linearizable: false };
}

// ── Histories ──────────────────────────────────────────────────────────────────────
// 1. Concurrent write and read: the read may see either value — both are linearizable
const concurrent = [
  { id: 'w1', op: 'write', value: 1, start: 0, end: 10 },
  { id: 'r1', op: 'read', value: 1, start: 5, end: 15 },
];
// 2. Stale read: the write FINISHED before the read STARTED, yet the read returned the old value
const stale = [
  { id: 'w1', op: 'write', value: 1, start: 0, end: 10 },
  { id: 'r1', op: 'read', value: null, start: 12, end: 20 },
];
// 3. Time going backwards: one client sees the new value, a later client sees the old one
const flipFlop = [
  { id: 'w1', op: 'write', value: 1, start: 0, end: 30 },   // a slow write, still in flight
  { id: 'rA', op: 'read', value: 1, start: 5, end: 10 },    // A sees the new value…
  { id: 'rB', op: 'read', value: null, start: 15, end: 20 },// …then B, strictly later, sees the old one
];
// 4. The same slow write, but the reads see old-then-new: fine — the write took effect between them
const oldThenNew = [
  { id: 'w1', op: 'write', value: 1, start: 0, end: 30 },
  { id: 'rA', op: 'read', value: null, start: 5, end: 10 },
  { id: 'rB', op: 'read', value: 1, start: 15, end: 20 },
];
for (const [name, h] of Object.entries({ concurrent, stale, flipFlop, oldThenNew })) {
  console.log(name.padEnd(10), JSON.stringify(checkLinearizable(h)));
}
// concurrent {"linearizable":true,"order":["w1","r1"]}
// stale      {"linearizable":false}
// flipFlop   {"linearizable":false}
// oldThenNew {"linearizable":true,"order":["rA","w1","rB"]}
```

Histories 3 and 4 have exactly the same operations and timing; only the order of the two reads' results
differs. That's the essence of linearizability: it isn't about any single read being fresh — a read
concurrent with a write may legitimately return the old value — but about whether *all* the observations
together are consistent with one copy.

The decision to defend is **an exhaustive backtracking search** rather than a fast heuristic such as "check
that every read returns the most recent write that finished before it started". The heuristic is linear
time, but it's wrong in both directions: it rejects valid histories (a read concurrent with a write may
return either value, and which one is allowed depends on what *other* reads saw), and it misses invalid
ones like `flipFlop`, where each read individually looks plausible and only the combination is impossible.
Checking linearizability is NP-complete in general, so real checkers bound the search with pruning and
by partitioning histories per key; but the search is the definition, and a checker that approximates it
gives false confidence — the most dangerous output a test can produce.

---

## 20.10 Questions

**1.** MCQ — Which guarantee does linearizability add over sequential consistency? a) A single total order b) Respect for real-time order between non-overlapping operations c) Per-client program order d) Convergence
<details><summary>Answer</summary>**b)** Both have a single total order respecting each client's order; linearizability also requires that an operation that finished before another started comes first.</details>

**2.** MCQ — Which operation most clearly needs linearizability? a) Showing a product description b) Acquiring a distributed lock c) Rendering a like count d) Returning search results
<details><summary>Answer</summary>**b)** Every node must agree on the current holder; a stale read means two holders.</details>

**3.** MCQ — During a network partition, a linearizable system's minority side must: a) Keep accepting writes b) Refuse operations that need the majority c) Switch to eventual consistency silently d) Shut down every node
<details><summary>Answer</summary>**b)** It can't know whether the other side has newer writes, so it can't serve linearizable reads or writes until it can reach a majority.</details>

**4.** MCQ — What is causal consistency's key advantage? a) It's linearizable b) It can remain available during partitions while ordering cause before effect c) It needs no metadata d) It provides serializable transactions
<details><summary>Answer</summary>**b)** It's essentially the strongest model compatible with availability under partition.</details>

**5.** MCQ — "I changed my address and then my phone number; my phone change was applied but the address change was lost." Which session guarantee was violated? a) Read your writes b) Monotonic reads c) Monotonic writes d) Writes follow reads
<details><summary>Answer</summary>**c)** One client's writes took effect out of order (and the earlier one was overwritten).</details>

**6.** MCQ — What does PACELC add to CAP? a) Byzantine faults b) The latency-versus-consistency trade-off when there's no partition c) A fourth property, durability d) Serializability
<details><summary>Answer</summary>**b)** Else (no partition), systems still trade Latency against Consistency, which is the everyday trade-off.</details>

**7.** Short — Why isn't "pick two of C, A, and P" a correct reading of CAP?
<details><summary>Answer</summary>Partitions happen whether you choose them or not. CAP only says that *during* a partition you must give up linearizability or availability; when there's no partition, you can have both.</details>

**8.** Short — Why are quorum reads and writes (W + R > N) not enough for linearizability?
<details><summary>Answer</summary>A read concurrent with a write may see the new value from one quorum, and a later read may see the old value from another quorum where the write hasn't landed — time going backwards. Linearizability needs coordination (consensus, or synchronous read repair plus more).</details>

**9.** Short — What is strong eventual consistency, and what does it remove the need for?
<details><summary>Answer</summary>Replicas that have received the same set of updates are in the same state regardless of delivery order, typically via CRDTs. It removes the need for conflict-resolution policies like last-write-wins.</details>

**10.** Short — Give a "cross-channel" scenario that needs linearizability.
<details><summary>Answer</summary>A service writes an image to storage and then sends its ID over a queue; the consumer reads the image by ID. If the storage read can be stale, the consumer may not find an image whose write "already happened" — the queue message is a side channel revealing the order.</details>

**11.** Short — Why should a read API expose its consistency level explicitly?
<details><summary>Answer</summary>So choosing a weak model is deliberate and visible in review, defaults can be safe (e.g. session), and invariant-protecting code paths can't silently read from a stale replica.</details>

**12.** Output — Using §20.9's checker, is this history linearizable: write `x = 1` over [0, 10], write `x = 2` over [5, 15], read over [20, 25] returning 1?
<details><summary>Answer</summary>**Yes.** The two writes overlap, so either may take effect last; ordering w2 then w1 leaves `x = 1` for the later read.</details>

**13.** Output — Same checker: write `x = 1` over [0, 10], then write `x = 2` over [12, 20], then a read over [25, 30] returning 1. Linearizable?
<details><summary>Answer</summary>**No.** The writes don't overlap, so w1 precedes w2 in real time; the read starts after w2 finished and must return 2.</details>

**14.** Output — What does this print?
```js
const shown = [];
const delivered = new Set();
const msgs = [{ id: 'c', deps: ['b'] }, { id: 'b', deps: ['a'] }, { id: 'a', deps: [] }];
let pending = [...msgs];
while (pending.length) {
  const m = pending.find((x) => x.deps.every((d) => delivered.has(d)));
  delivered.add(m.id); shown.push(m.id); pending = pending.filter((x) => x !== m);
}
console.log(shown.join(''));
```
<details><summary>Answer</summary>`abc` — each message waits for its dependency, so causal order is restored regardless of arrival order.</details>

**15.** Output — A linearizable write in a three-region consensus group must be acknowledged by a majority. Inter-region round trips from the leader are 70 ms and 90 ms. What is the minimum commit latency for a write arriving at the leader?
<details><summary>Answer</summary>**~70 ms.** The leader plus the nearest other region form a majority of three, so the commit waits for the faster of the two remote acknowledgements.</details>

**16.** Debug — Two users successfully registered the same username during a brief network split between data centres. Each region's service checked its local replica for the name before inserting. What's wrong, and what's the fix?
<details><summary>Answer</summary>Uniqueness needs linearizability, but each region checked a replica that couldn't see the other's write (an available, not consistent choice). Route username claims to a single leader or consensus-backed store with a unique constraint, accepting that claims fail (rather than duplicate) during a partition.</details>

**17.** Debug — Users report that a reply to their post sometimes appears in notifications before the post itself loads. Posts and replies are stored in different partitions replicated independently. What model is being violated, and how do you fix it?
<details><summary>Answer</summary>Causal consistency: the reply happens-after the post but is visible first. Carry the dependency (the reply references the post's version or position) and hold the reply until the post is readable at that version, or store a thread's items in the same partition so they replicate in order.</details>

**18.** Debug — A team's design doc claims "our Cassandra cluster is CP because we use QUORUM", but a lock implemented on it occasionally lets two workers in. Explain.
<details><summary>Answer</summary>Quorum reads/writes aren't linearizable (concurrent reads can disagree), lightweight conflicts are resolved by timestamps, and a quorum-based "lock" has no fencing. Locks need consensus with leases and fencing tokens (Chapter 21) — Cassandra's lightweight transactions (Paxos-based) are the closer fit if it must stay in Cassandra.</details>

**19.** Debug — A service reads feature-flag values from an eventually consistent cache. During an incident, the team flipped a kill switch, but some instances kept running the broken code path for 12 minutes. What should change?
<details><summary>Answer</summary>A safety-critical switch needs bounded, short staleness (or linearizable reads). Have instances subscribe to changes and also poll the source with a short TTL, bound and monitor propagation time, and treat kill switches as a different class of configuration from ordinary flags.</details>

**20.** Code — Write `readWith(consistency, { leader, replicas, sessionToken })` that routes `'strong'` reads to the leader, `'session'` reads to a replica at or past the token (else the leader), and `'eventual'` reads to any replica.
<details><summary>Answer</summary>

```js
function readWith(consistency, { leader, replicas, sessionToken = 0, rand = Math.random }) {
  if (consistency === 'strong') return leader;   // and the leader must confirm it still leads
  if (consistency === 'session') {
    const ok = replicas.filter((r) => r.position >= sessionToken);
    return ok.length ? ok[Math.floor(rand() * ok.length)] : leader;
  }
  if (consistency === 'eventual') return replicas[Math.floor(rand() * replicas.length)] ?? leader;
  throw new RangeError(`unknown consistency ${consistency}`);
}
const replicas = [{ name: 'r1', position: 90 }, { name: 'r2', position: 120 }];
console.log(readWith('session', { leader: { name: 'L' }, replicas, sessionToken: 100, rand: () => 0 }).name); // r2
```
</details>

**21.** Code — Write `isConcurrent(a, b)` for operations with `start` and `end`, returning `true` when neither finished before the other started.
<details><summary>Answer</summary>

```js
const isConcurrent = (a, b) => !(a.end < b.start || b.end < a.start);
console.log(isConcurrent({ start: 0, end: 10 }, { start: 5, end: 15 }), isConcurrent({ start: 0, end: 10 }, { start: 12, end: 20 })); // true false
```
Only non-concurrent operations have a fixed order under linearizability.</details>

**22.** Code — Write `staleReplicas(replicas, { leaderPosition, maxLagEntries, maxLagMs, now })` returning the names of replicas that exceed either staleness bound and should stop serving reads.
<details><summary>Answer</summary>

```js
function staleReplicas(replicas, { leaderPosition, maxLagEntries = 1_000, maxLagMs = 5_000, now = Date.now() }) {
  return replicas
    .filter((r) => leaderPosition - r.position > maxLagEntries || now - r.lastAppliedAt > maxLagMs)
    .map((r) => r.name);
}
console.log(staleReplicas([
  { name: 'r1', position: 9_990, lastAppliedAt: 1_000_000 },
  { name: 'r2', position: 8_000, lastAppliedAt: 1_000_000 },
  { name: 'r3', position: 9_999, lastAppliedAt: 990_000 },
], { leaderPosition: 10_000, now: 1_001_000 })); // [ 'r2', 'r3' ]
```
</details>

**23.** Scenario — A global social app is moving to active-active across three regions. Product wants "strong consistency" for everything. How do you respond?
<details><summary>Answer</summary>Strong (linearizable) everything means every write and many reads pay a cross-region round trip (~100 ms) and fail in a region cut off from the majority — unacceptable for posting and browsing. Classify operations: usernames, payments, and account security changes go through a consensus-backed home region (linearizable); a user's own content is read-your-writes via session tokens; threads and replies are causal; feeds, counts, and search are eventual with bounded staleness. Write the model into the API so it's explicit.</details>

**24.** Scenario — An inventory service is choosing between a leaderless store at `ONE`/`ONE` and a consensus-based SQL database. It serves 20,000 stock reads/s for product pages and 500 reservations/s at checkout. Which do you choose?
<details><summary>Answer</summary>Split by operation, not by system. Reservations need linearizable conditional decrements — consensus-based SQL (or a single-leader database) with atomic updates (Chapter 15). Product-page stock reads tolerate seconds of staleness and can come from replicas or a cache fed by CDC, showing "low stock" approximately. If one store must serve both, choose the one that can do the linearizable decrement and offload the read volume to caches, not the other way round.</details>

---

## Chapter Summary

- A consistency model is a contract about what reads may return; stronger models are easier to program
  against and cost more latency and availability.
- Linearizability makes replicated data behave like one copy respecting real time; it's needed for locks,
  leadership, uniqueness, and conditional writes — and not for most reads.
- Causal consistency orders cause before effect, allows concurrent operations to vary, and is essentially
  the strongest model that stays available during partitions.
- Session guarantees — read your writes, monotonic reads and writes, writes follow reads — are cheap and fix
  most user-visible anomalies; implement them first.
- CAP says only that during a partition you give up linearizability or availability; PACELC adds the
  everyday latency-versus-consistency trade-off, and both apply per operation.
- Assign each operation the weakest correct model, make the choice explicit in code, and test consistency
  claims with recorded histories.

---

## Practice Exercises

1. Record histories from §13.9's simulated cluster under random replica lag and check them with §20.9's
   checker, with and without session tokens.
2. Extend §20.9's checker to handle a compare-and-set operation (`cas(expected, new)` returning success or
   failure) and test it on histories with concurrent CAS attempts.
3. Read Kleppmann's "A Critique of the CAP Theorem" or Abadi's PACELC paper and write a one-page summary of
   where the common CAP framing misleads.
4. For a system you know, list ten operations and assign each the weakest correct consistency model, with
   the mechanism you'd use.
5. Implement version-vector-based causal delivery for a three-replica simulated chat, and show replies are
   never displayed before the messages they answer.

---

## Mini Project: A Consistency Test Harness

Build a harness that runs concurrent clients against a replicated store (your Chapter 13 simulator, or a
real database in containers), injects faults, records every operation's interval and result, and checks
the recorded history against the model the store claims.

- Concurrent client workers performing reads, writes, and compare-and-set on a small set of keys.
- Fault injection: partitions, leader kills, replica lag, and clock skew, on a seeded schedule.
- History recording with client-side start and end timestamps from a monotonic clock.
- Checkers for linearizability (per key, with pruning), read-your-writes, and monotonic reads.
- A report listing violations with the minimal sub-history that demonstrates each one.

---

**Next:** [Chapter 21 — Consensus, Leases, and Coordination →](21-consensus-coordination.md)
