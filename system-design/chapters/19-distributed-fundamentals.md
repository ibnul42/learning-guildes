# Chapter 19 — Distributed Systems Fundamentals

## Learning Objectives

- Explain why partial failure makes distributed systems fundamentally different from single machines
- Reason about lost, delayed, duplicated, and reordered messages, and why "slow" and "dead" are indistinguishable
- Choose timeouts from latency distributions and use adaptive failure detectors
- Avoid the bugs caused by unreliable clocks and process pauses, including expired leases acted on
- Name the fault model a design assumes, and separate its safety properties from its liveness properties

---

## 19.1 What Makes "Distributed" Different

On a single machine, failures are mostly total: if the power goes, everything stops together, and
when it comes back, the program starts from a known state. Inside one process, a function call
either returns or throws; memory is shared; there's one clock.

A **distributed system** is a set of machines cooperating over a network, and it loses every one of
those comforts:

- **Partial failure.** Some components fail while others keep running — one node of five crashes, a
  network link drops packets in one direction, a disk becomes slow. The system must keep working, or
  fail safely, while it's partly broken. This is the defining property, and it's why distributed
  programming is hard: every remote interaction has an outcome you may never learn.
- **No shared memory.** Nodes learn about each other only by sending messages, which can be lost or
  delayed.
- **No shared clock.** Each node has its own clock, and they disagree (§19.4).

The **fallacies of distributed computing** — a list compiled at Sun Microsystems in the 1990s — name the
assumptions engineers new to this make, each of which a real network eventually breaks:

| Fallacy | Reality | Where this book handles it |
|---|---|---|
| The network is reliable | Packets are lost, links fail, partitions happen | §19.2, Chapter 28 |
| Latency is zero | Round trips cost 0.1–200 ms (Chapter 2) | Chapter 5 |
| Bandwidth is infinite | Links saturate; egress costs money | Chapters 2, 6 |
| The network is secure | Traffic can be read and forged | Chapter 33 |
| Topology doesn't change | Nodes and routes come and go | Chapters 7, 14 |
| There is one administrator | Many teams and providers change things independently | Chapter 31 |
| Transport cost is zero | Serialisation, TLS, and per-GB charges add up | Chapter 5 |
| The network is homogeneous | Mixed versions, protocols, and hardware | Chapter 8 |

Everything in Part IV follows from taking partial failure seriously. This chapter describes the
failures; the next four describe what you can still guarantee in spite of them.

---

## 19.2 Unreliable Networks

Most data-centre networks are **asynchronous packet networks**: a node sends a message and the network
makes no promise about when, or whether, it arrives. When you send a request and wait for a reply,
there are many ways not to get one:

```text
1. the request is lost on the way                  ──x
2. the request is queued, and will arrive late     ──────────────────▶
3. the remote node crashed before processing       ──▶ ✗
4. the remote node is paused (GC, overloaded)      ──▶ … … … ▶ (processes it much later)
5. the remote node processed it; the reply is lost ──▶ ✓ ──x
6. the reply is merely delayed                     ──▶ ✓ ─────────────────▶
```

From the sender's side, **all six look the same**: no reply yet. In cases 5 and 6 the operation
*happened*. This is why Chapter 8 needed idempotency keys: a client that times out genuinely doesn't
know whether its payment went through, and nothing about the network can tell it.

Networks also **duplicate** messages (a retransmission when the original was only delayed) and
**reorder** them (two messages taking different paths). TCP hides duplication and reordering within one
connection, but not across connections, reconnects, or retries at a higher layer — so applications see
both.

**Network partitions** — some nodes can talk to each other but not to others — are not exotic. They
happen through switch failures, misconfigured firewalls, saturated links, and cloud networking
incidents. They can be **asymmetric**: A can reach B but B can't reach A, so B thinks A is dead while A
thinks everything is fine. And a node can be partitioned from *some* peers and not others, so different
parts of the cluster have different views of who's alive. Any design that needs every node to agree on
membership at every moment will eventually be wrong.

> ⚠️ **Gotcha** — **Two generals.** Two parties that can only communicate over an unreliable channel
> can't both become certain that they agree: the last message sent is never known to have arrived, and
> acknowledging it just creates a new last message. No protocol fixes this. Practical systems escape it
> by not needing *certainty* — they use timeouts, retries with idempotency, and agreement by majority
> (Chapter 21) rather than unanimous confirmation.

---

## 19.3 Timeouts and Failure Detection

Since a remote node's state can't be observed directly, systems **infer** failure from silence: if
there's no reply or heartbeat within a timeout, assume it's down. Every such inference can be wrong, and
the timeout chooses which way it's wrong:

| Timeout | Effect when the node is actually fine | Effect when the node is actually dead |
|---|---|---|
| Too short | False alarms: needless failovers, retries that add load, nodes flapping in and out | Detected quickly |
| Too long | Few false alarms | Users wait; the outage lasts longer before recovery starts |

A reasonable timeout comes from the **observed latency distribution**, not from a round number. If a
dependency's p99.9 latency is 120 ms, a 50 ms timeout fails one request in ten; a 150 ms timeout fails
about one in a thousand plus the real failures. The tail is what matters (Chapter 3), and it's what
changes under load — which is why a timeout tuned on a quiet day misfires during a busy one.

```js
// Choose a timeout from observed latencies: a high percentile plus a margin, then check
// what fraction of healthy requests it would wrongly cut off.
function timeoutFor(samplesMs, { percentile = 99.9, margin = 1.5 } = {}) {
  const sorted = [...samplesMs].sort((a, b) => a - b);
  const pAt = sorted[Math.min(sorted.length - 1, Math.ceil((percentile / 100) * sorted.length) - 1)];
  const timeout = Math.ceil(pAt * margin);
  const falseTimeouts = sorted.filter((s) => s > timeout).length / sorted.length;
  return { timeout, falseTimeouts };
}
// 10,000 healthy samples: mostly ~20 ms, a 0.5% tail around 90 ms, a 0.05% tail of 300 ms
const samples = [...Array(9_945).fill(20), ...Array(50).fill(90), ...Array(5).fill(300)];
console.log(timeoutFor(samples));                       // { timeout: 135, falseTimeouts: 0.0005 }
console.log(timeoutFor(samples, { percentile: 99 }));   // { timeout: 30, falseTimeouts: 0.0055 }
```

Basing the timeout on p99 instead of p99.9 cuts off more than ten times as many healthy requests — 0.55% instead of 0.05%, enough to feed a retry
storm under load. Two further refinements matter:

- **Failure detection for cluster membership** (is node B alive?) uses **heartbeats** — periodic "I'm
  alive" messages — and declares a node dead after missing several. The real-world example in §19.8 is
  an **adaptive** detector that learns each peer's normal heartbeat timing and reports a suspicion level
  rather than a binary verdict.
- **Detection is not action.** Declaring a node dead and *acting on it* (failing over, rebalancing data)
  should have different thresholds. Chapter 14's gotcha is the cost of conflating them: a node slow for a
  minute triggers data movement that overloads everyone else.

> 💼 **Interview** — "How do you detect that a node has failed?" The strong answer: you can't detect it
> with certainty, only suspect it from missing heartbeats; the timeout trades detection speed against
> false positives; adaptive detectors (phi accrual) adjust to observed network jitter; and the system must
> be safe even when the suspicion is wrong — which is what fencing (Chapter 21) is for.

---

## 19.4 Unreliable Clocks

Each machine has a quartz clock that drifts — typically by tens of parts per million, or a few seconds a
day — and is periodically corrected by **NTP** (or a cloud provider's time service). After correction,
clocks on well-run machines usually agree within a few milliseconds, sometimes far worse: a misconfigured
NTP server, a VM resumed after migration, or a network delay to the time server can leave a node off by
seconds or more, and it may not know.

Computers expose two kinds of clock, and confusing them is a classic bug:

| | Wall clock (time of day) | Monotonic clock |
|---|---|---|
| JavaScript | `Date.now()` | `performance.now()`, `process.hrtime.bigint()` |
| Meaning | Milliseconds since 1970, intended to match other machines | Time since an arbitrary point on *this* machine |
| Can jump | **Yes** — backwards and forwards when NTP steps it, or an operator changes it | No — only moves forward |
| Comparable across machines | Roughly (to within skew) | No |
| Use for | Timestamps shown to humans; coarse cross-machine reasoning with known skew bounds | Measuring durations, timeouts, rate limits on one machine |

```js
// Why durations must use a monotonic clock: wall clocks can step backwards
const wallReadings = [1_763_200_000_000, 1_763_199_999_700];  // NTP stepped the clock back 300 ms
const monoReadings = [5_000.0, 5_012.4];                       // performance.now(): only moves forward
console.log('wall-clock "duration":', wallReadings[1] - wallReadings[0], 'ms');      // wall-clock "duration": -300 ms
console.log('monotonic duration:', (monoReadings[1] - monoReadings[0]).toFixed(1), 'ms'); // monotonic duration: 12.4 ms
```

A negative duration breaks rate limiters, retry backoffs, and latency metrics; a forward jump expires
every timer at once. **Measure durations with the monotonic clock, always.**

**Ordering events by wall-clock timestamps across machines is unreliable.** If node A's clock is 20 ms
ahead of node B's, a write on B that happened 10 ms *after* a write on A gets an *earlier* timestamp. Any
system that resolves conflicts by "latest timestamp wins" (Chapter 13's LWW) will then keep the older
write and discard the newer one — silently. Chapter 22 covers the alternatives: logical clocks, hybrid
logical clocks, and clocks with explicit uncertainty bounds (Spanner's TrueTime).

### Process pauses

A subtler problem: a running process can **stop** for an unbounded time without knowing it. Garbage
collection pauses (hundreds of milliseconds to seconds), a VM being suspended or live-migrated, the OS
swapping memory to disk, a laptop lid closing, `SIGSTOP`, a synchronous disk write on a saturated device.
When the process resumes, it continues exactly where it was — with its view of the world now stale.

This turns check-then-act code into a bug, even on one node:

```js
// ❌ Wrong — checks the lease, then acts. A pause between the two makes the check worthless.
// Simulated: lease held until t = 10 s; the check happens at 9 s; a 3 s GC pause follows.
const log = [];
let t = 9_000;
const lease = { holder: 'node-a', expiresAt: 10_000 };

if (t < lease.expiresAt) {          // node-a: "my lease is valid"
  t += 3_000;                       // …a 3-second pause it never notices…
  // Meanwhile, at t = 10 s, the lock service gave the lease to node-b, which wrote at 10.5 s
  log.push({ at: 10_500, writer: 'node-b' });
  log.push({ at: t, writer: 'node-a' }); // node-a resumes and writes, believing it still holds the lease
}
console.log(log.map((e) => `${e.writer}@${e.at / 1000}s`).join(', ')); // node-b@10.5s, node-a@12s
```

Both nodes wrote while each believed it was the only lease holder. No amount of re-checking the clock
fixes this, because the pause can happen between *any* check and the action that follows. The fix is to
make the **resource** reject stale holders: each lease grant carries a monotonically increasing **fencing
token**, and storage refuses writes with a token lower than one it has already seen. Chapter 21 builds it.
The general lesson: **a node can't rely on its own belief that it holds a lock, lease, or leadership** —
that belief may be seconds out of date.

---

## 19.5 Fault Models

Every distributed algorithm assumes something about how components fail. Naming the assumption is how
you know whether an algorithm fits your environment:

| Fault model | What can happen | Typical setting |
|---|---|---|
| **Crash-stop** | A node works correctly until it stops, forever | Theoretical baseline |
| **Crash-recovery** | A node stops, then may restart — losing memory but keeping durable state | **Most real systems** — what Raft, Paxos, and databases assume |
| **Omission** | A node or link drops some messages | Real networks; covered by retries and timeouts |
| **Byzantine** | A node can behave arbitrarily: send wrong or contradictory messages, lie | Blockchains, aerospace, multi-party systems with mutual distrust |

Systems inside one organisation almost always assume **crash-recovery with omission failures**, and
non-Byzantine nodes — nodes may be slow, crash, or lose messages, but they don't lie. That assumption is
reasonable because Byzantine fault tolerance is expensive (it typically needs 3f + 1 nodes to tolerate f
faulty ones, versus 2f + 1 for crash faults) and because the threats it addresses — malicious nodes — are
handled by authentication and access control (Chapter 33) instead.

"Non-Byzantine" still requires care. A node with a bit-flipping memory error, a corrupted disk returning
garbage, or a software bug sending malformed data is behaving Byzantine-ly by accident. Cheap protections
cover most of it: **checksums** on stored and transmitted data (Chapter 10's log records), **input
validation** on everything received from other nodes, and **sanity checks** on values (a timestamp ten
years in the future, a negative quantity) that reject rather than propagate.

Timing assumptions matter as much as fault types. Algorithms assume one of:

- **Synchronous** — message delays and pauses have a known upper bound. Real networks don't provide it.
- **Asynchronous** — no bounds at all. Under this model, consensus is provably impossible to guarantee
  if even one node can crash (the FLP result) — no algorithm can always terminate.
- **Partially synchronous** — the system *usually* behaves within bounds, but sometimes doesn't, for
  unknown stretches. This is the realistic model, and practical consensus algorithms are designed for it:
  they stay **safe** no matter how bad the timing gets, and make **progress** whenever timing is
  well-behaved for long enough.

---

## 19.6 Safety, Liveness, and Testing for Partial Failure

The partially synchronous model suggests a precise way to state what a system guarantees.

- A **safety** property says something bad *never* happens: two nodes never both believe they're
  leader in the same term; an acknowledged write is never lost; a seat is never sold twice. If violated,
  there's a specific moment it went wrong, and it can't be undone.
- A **liveness** property says something good *eventually* happens: a request eventually gets a response;
  a failed leader is eventually replaced. It can be temporarily unmet without anything being permanently
  broken.

Well-designed distributed systems **never compromise safety, and give up liveness when they must**: during
a partition, a consensus-based database may refuse writes (not live) rather than accept writes on both
sides (unsafe). When you design, write the safety properties down — they're your invariants — and accept
that liveness depends on the network eventually behaving.

**Testing for partial failure** needs tools that make failures happen on purpose, because they rarely
happen in development:

| Technique | What it finds |
|---|---|
| **Fault injection in tests** | Behaviour when a dependency times out, errors, returns garbage, or is slow |
| **Network partition testing** (Jepsen-style) | Lost writes, split brain, and broken consistency guarantees under partitions and clock skew |
| **Deterministic simulation** | Runs the whole system on one thread with simulated network, disk, and clocks under a seeded scheduler, so rare interleavings are found and *replayed exactly* |
| **Chaos engineering in production** | Whether real deployments survive instance, zone, and dependency failures as designed |

Deterministic simulation deserves emphasis: systems such as FoundationDB were built around it, running
the equivalent of years of failure scenarios before shipping. Every simulation in this book — seeded
random numbers, injected clocks (`now: () => t`), fake stores — is a small instance of the same idea:
when time, randomness, and I/O are inputs rather than globals, failures become reproducible.

> ⚠️ **Gotcha** — **A system that's never been partitioned in testing will lose data the first time it's
> partitioned in production.** Published partition tests of databases have repeatedly found lost
> acknowledged writes, stale reads, and split-brain behaviour in systems whose documentation claimed
> otherwise. Don't trust a consistency claim — your own or a vendor's — that hasn't been tested under
> partitions, pauses, and clock skew.

---

## 19.7 Common Mistakes and Best Practices

**Mistake: treating a timeout as a failure.** A timed-out request may have succeeded; retrying it without
idempotency duplicates the effect.

**Mistake: timeouts chosen as round numbers.** A timeout below the dependency's real tail cuts off healthy
requests and causes retry storms under load.

**Mistake: measuring durations with the wall clock.** NTP steps make durations negative or huge; timers,
rate limits, and metrics break.

**Mistake: ordering events across machines by wall-clock timestamps.** Clock skew makes later events look
earlier, and last-write-wins discards the newer write.

**Mistake: trusting your own lease or leadership.** A process pause can make that belief stale by seconds;
resources must reject stale holders with fencing tokens.

**Mistake: acting on failure detection as quickly as you detect.** Failover and data movement on a
transient slowdown turn blips into outages.

**Mistake: never testing under partitions and pauses.** Consistency bugs stay hidden until production
creates the conditions.

**Best practice: state safety properties explicitly.** Invariants written down can be tested and protected;
liveness can degrade, safety mustn't.

**Best practice: inject time, randomness, and I/O.** Code that takes `now` and `rand` as parameters can be
simulated deterministically and its failures replayed.

**Best practice: checksums and sanity checks at every boundary.** They turn accidental Byzantine faults —
corruption, bugs — into detected errors.

---

## 19.8 Real-World Example: An Adaptive Failure Detector

A fixed heartbeat timeout is wrong for some peer: too short for a node across a jittery WAN link, too long
for a node on a quiet LAN. The **phi accrual** detector instead tracks the distribution of each peer's
heartbeat intervals and reports **phi** — a suspicion level equal to −log₁₀ of the probability that a
heartbeat is merely late. Callers choose the threshold at which to act. This is the design used by Akka,
and in a simplified form by Cassandra.

```js
// phi-accrual.mjs — a failure detector that reports suspicion on a continuous scale instead of
// a yes/no timeout, and adapts to each peer's observed heartbeat timing. (The design used by
// Akka and, in a simplified form, Cassandra.)

export class PhiAccrualDetector {
  #intervals = []; #last = null;

  constructor({ window = 100, minStdMs = 100, acceptablePauseMs = 0, now = () => performance.now() } = {}) {
    // minStd stops a very regular peer making the detector hypersensitive to tiny delays;
    // acceptablePause adds slack for known pauses (GC, a busy event loop)
    Object.assign(this, { window, minStdMs, acceptablePauseMs, now });
  }

  heartbeat() {
    const t = this.now();
    if (this.#last !== null) {
      this.#intervals.push(t - this.#last);
      if (this.#intervals.length > this.window) this.#intervals.shift();
    }
    this.#last = t;
  }

  // phi = −log10(probability that a heartbeat would still be on its way after this long).
  // phi 1 ≈ 10% chance we're wrong to suspect, phi 3 ≈ 0.1%, phi 8 ≈ one in a hundred million.
  phi() {
    if (this.#intervals.length < 2) return 0;
    const n = this.#intervals.length;
    const raw = this.#intervals.reduce((a, b) => a + b, 0) / n;
    const variance = this.#intervals.reduce((a, b) => a + (b - raw) ** 2, 0) / n;
    const mean = raw + this.acceptablePauseMs;
    const std = Math.max(Math.sqrt(variance), this.minStdMs);
    const elapsed = this.now() - this.#last;
    // Logistic approximation of the normal CDF (as in Akka's implementation)
    const y = (elapsed - mean) / std;
    const e = Math.exp(-y * (1.5976 + 0.070566 * y * y));
    return elapsed > mean ? -Math.log10(e / (1 + e)) : -Math.log10(1 - 1 / (1 + e));
  }

  isAvailable(threshold = 8) { return this.phi() < threshold; }
}

// ── Demo: two peers, both heartbeating about once a second, then going silent ──────
let seed = 7;
const rand = () => (seed = (seed * 16807) % 2147483647) / 2147483647;
function peer(spreadMs) {
  let t = 0;
  const d = new PhiAccrualDetector({ now: () => t });
  for (let i = 0; i < 100; i++) { t += 1_000 + (rand() - 0.5) * spreadMs; d.heartbeat(); }
  const last = t;
  return (silence) => { t = last + silence; return d.phi(); };
}
const steady = peer(100), noisy = peer(1_200);   // a quiet LAN peer, and one across a busy WAN
for (const silence of [1_000, 1_300, 1_600, 2_000, 3_000]) {
  console.log(`silent ${String(silence).padStart(5)} ms  steady phi ${steady(silence).toFixed(1).padStart(5)}  noisy phi ${noisy(silence).toFixed(1).padStart(5)}`);
}
// silent  1000 ms  steady phi   0.3  noisy phi   0.3
// silent  1300 ms  steady phi   2.9  noisy phi   0.7
// silent  1600 ms  steady phi  10.6  noisy phi   1.3
// silent  2000 ms  steady phi  37.2  noisy phi   2.6
// silent  3000 ms  steady phi 257.8  noisy phi   9.3
```

With a threshold of 8, the steady peer is suspected after about 1.5 seconds of silence and the noisy one
after about 3 — the same detector, adapting to each link without per-peer configuration.

The decision to defend is **returning a continuous suspicion level instead of a boolean "dead or alive"**.
A boolean forces the detector to pick one threshold for every consumer, but consumers have different costs
of being wrong. A load balancer can stop routing to a suspected node at phi 3 — a false alarm costs almost
nothing, since traffic goes elsewhere and returns when heartbeats resume. Promoting a new database leader
should wait for phi 8 or more, because a false alarm there causes a failover, possibly lost writes, and a
risk of split brain. One detector serving both, each with its own threshold, gives every decision the
trade-off between speed and false positives it actually needs — which a single timeout cannot.

---

## 19.9 Questions

**1.** MCQ — A client sends a request and gets no reply within its timeout. What can it conclude? a) The request was lost b) The server crashed c) Nothing certain — the request may or may not have been processed d) The network is partitioned
<details><summary>Answer</summary>**c)** Lost request, crashed server, paused server, lost reply, and slow reply all look identical from the sender.</details>

**2.** MCQ — Which clock should measure a request's duration? a) `Date.now()` b) A monotonic clock such as `performance.now()` c) An NTP server d) The database's `now()`
<details><summary>Answer</summary>**b)** Monotonic clocks never jump backwards; wall clocks can be stepped by NTP.</details>

**3.** MCQ — What fault model do Raft and most databases assume? a) Byzantine b) Crash-stop only c) Crash-recovery with possible message loss, non-Byzantine d) Synchronous with bounded delays
<details><summary>Answer</summary>**c)** Nodes may crash and restart with durable state, and messages may be lost or delayed, but nodes don't lie.</details>

**4.** MCQ — Which is a safety property? a) Every request eventually gets a response b) A failed leader is eventually replaced c) Two nodes are never leader in the same term d) The cluster eventually rebalances
<details><summary>Answer</summary>**c)** It says something bad never happens; the others say something good eventually happens (liveness).</details>

**5.** MCQ — Why can't a node rely on having checked its lease just before writing? a) Leases are always revoked b) A process pause between the check and the write can outlast the lease c) Clocks are monotonic d) Writes are asynchronous
<details><summary>Answer</summary>**b)** GC, VM suspension, or swapping can stop the process for seconds after the check, during which the lease expires and is granted elsewhere.</details>

**6.** MCQ — What does a phi value of 3 from a phi-accrual detector mean? a) The node missed 3 heartbeats b) About a 0.1% chance the heartbeat is merely late c) 3 seconds of silence d) Three nodes suspect it
<details><summary>Answer</summary>**b)** phi = −log₁₀(probability of a heartbeat being this late if the node is alive); 10⁻³ = 0.1%.</details>

**7.** Short — Why is partial failure the defining difficulty of distributed systems?
<details><summary>Answer</summary>Some components fail while others continue, and the healthy ones can't reliably tell which have failed, so every remote interaction has a possibly unknowable outcome and the system must behave correctly while partly broken.</details>

**8.** Short — What is an asymmetric partition, and why does it confuse failure detection?
<details><summary>Answer</summary>A can reach B but B can't reach A. B may declare A dead while A believes everything is fine, so different nodes hold contradictory views of membership and leadership.</details>

**9.** Short — Why does last-write-wins with wall-clock timestamps lose data?
<details><summary>Answer</summary>Clock skew can give a later write an earlier timestamp, so conflict resolution keeps the older write and silently discards the newer one.</details>

**10.** Short — What does the FLP result say, and why don't practical systems give up on consensus?
<details><summary>Answer</summary>In a fully asynchronous system where even one node may crash, no deterministic algorithm can guarantee consensus always terminates. Practical algorithms assume partial synchrony: they're always safe and make progress whenever timing behaves well for long enough.</details>

**11.** Short — Why should detecting a failure and acting on it use different thresholds?
<details><summary>Answer</summary>Actions have different costs when the suspicion is wrong. Rerouting traffic is cheap to undo; failover or data movement is expensive and can cascade. Expensive actions should wait for stronger evidence.</details>

**12.** Output — Using §19.3's `timeoutFor`, what does `timeoutFor([...Array(990).fill(10), ...Array(10).fill(100)], { percentile: 99.9, margin: 1.2 })` return?
<details><summary>Answer</summary>`{ timeout: 120, falseTimeouts: 0 }` — the 99.9th percentile is 100 ms, × 1.2 = 120 ms, and no healthy sample exceeds it.</details>

**13.** Output — What does this print?
```js
const t0 = 1_000_000, t1 = 999_850;  // wall-clock readings; NTP stepped the clock back between them
const elapsed = t1 - t0;
console.log(elapsed, Math.max(0, elapsed));
```
<details><summary>Answer</summary>`-150 0` — the "duration" is negative; clamping hides the symptom but the measurement is still wrong. Use a monotonic clock.</details>

**14.** Output — How many nodes does a crash-fault-tolerant consensus group need to tolerate 2 failures, and how many does a Byzantine-fault-tolerant one need?
<details><summary>Answer</summary>**5 and 7.** Crash tolerance needs 2f + 1 = 5; Byzantine tolerance needs 3f + 1 = 7.</details>

**15.** Output — Node A's clock is 40 ms ahead of node B's. A writes at true time 1,000 ms; B writes the same key at true time 1,030 ms. With last-write-wins on local timestamps, which write survives?
<details><summary>Answer</summary>**A's (the older one).** A stamps 1,040; B stamps 1,030. LWW keeps 1,040 and discards B's genuinely later write.</details>

**16.** Debug — A rate limiter occasionally lets through a burst of thousands of requests at once, always within a minute of the hosts' daily NTP sync. It computes elapsed time with `Date.now()`. Cause and fix?
<details><summary>Answer</summary>NTP steps the wall clock; a forward step makes the limiter think a long time has passed and refills its budget (a backward step makes elapsed negative). Use a monotonic clock for elapsed time, and configure NTP to slew rather than step where possible.</details>

**17.** Debug — A worker holding a distributed lock occasionally corrupts a shared file: logs show another worker writing at the same time, each believing it held the lock. The lock has a 10-second TTL; the workers are JVMs with occasional multi-second GC pauses. Explain and fix.
<details><summary>Answer</summary>A GC pause after acquiring (or after checking) the lock outlasted the TTL; the lock was granted to another worker, and the first resumed and wrote. Re-checking doesn't help. Use fencing tokens: each grant carries an increasing number, and the storage rejects writes with an older token (Chapter 21).</details>

**18.** Debug — A cluster's failover triggers several times a week, each time promoting a new leader and dropping in-flight work, but post-incident reviews find the old leader was healthy. The heartbeat timeout is 1 second over a cross-zone link. What's happening, and what do you change?
<details><summary>Answer</summary>The timeout is inside the link's normal jitter and GC pauses, so healthy leaders are declared dead. Base the timeout on the observed heartbeat distribution (or use an adaptive detector), require confirmation from several observers, and use a higher bar for failover than for routing decisions.</details>

**19.** Debug — During a network incident, some clients reported their payments failed (timeouts) and retried; customers were charged twice. The payment service had no outage. What's the root cause?
<details><summary>Answer</summary>The original requests succeeded but responses were lost or delayed; clients treated timeouts as failures and retried non-idempotent operations. Add idempotency keys (Chapter 8) and pass them to the payment provider.</details>

**20.** Code — Write `elapsedSince(startNs)` using `process.hrtime.bigint()` that returns milliseconds as a number, and use it to time an awaited 20 ms sleep.
<details><summary>Answer</summary>

```js
const elapsedSince = (startNs) => Number(process.hrtime.bigint() - startNs) / 1e6; // monotonic, ns resolution
const start = process.hrtime.bigint();
await new Promise((r) => setTimeout(r, 20));
console.log(elapsedSince(start) >= 19); // true
```
</details>

**21.** Code — Write `withDeadline(promise, ms)` that rejects with a `TimeoutError` if the promise hasn't settled in time, and clears its timer either way.
<details><summary>Answer</summary>

```js
class TimeoutError extends Error {}
function withDeadline(promise, ms) {
  let timer;
  const deadline = new Promise((_, reject) => { timer = setTimeout(() => reject(new TimeoutError(`timed out after ${ms} ms`)), ms); });
  return Promise.race([promise, deadline]).finally(() => clearTimeout(timer));
}
await withDeadline(new Promise((r) => setTimeout(r, 50)), 10).catch((e) => console.log(e instanceof TimeoutError)); // true
```
A timeout ends the *wait*, not the remote operation — it may still complete, which is why the caller must treat the outcome as unknown.</details>

**22.** Code — Write `sanityCheck(event, { now, maxSkewMs })` that rejects events whose timestamp is more than `maxSkewMs` in the future or that have a negative quantity, returning a list of problems.
<details><summary>Answer</summary>

```js
function sanityCheck(event, { now = Date.now(), maxSkewMs = 5 * 60_000 } = {}) {
  const problems = [];
  if (!Number.isFinite(event.ts)) problems.push('missing or invalid timestamp');
  else if (event.ts > now + maxSkewMs) problems.push(`timestamp ${event.ts - now} ms in the future`);
  if (!(event.quantity >= 0)) problems.push('quantity must be non-negative');
  return problems; // reject and report instead of propagating corrupt data downstream
}
console.log(sanityCheck({ ts: Date.now() + 3_600_000, quantity: -2 }));
// [ 'timestamp 3600000 ms in the future', 'quantity must be non-negative' ]
```
</details>

**23.** Scenario — A team proposes ordering events from 40 microservices by wall-clock timestamp to build a global audit log, arguing "our clocks are NTP-synced to within a few milliseconds". What do you advise?
<details><summary>Answer</summary>NTP sync is usually good and occasionally far off, and "within a few ms" still misorders events that are causally related but close together. If the requirement is causal order (A caused B), carry causal metadata — trace IDs and parent links, logical or hybrid logical clocks (Chapter 22). If a total order is required, route events through a single ordered log (a partition of Kafka per entity). Keep wall-clock timestamps for human reading, flagged as approximate.</details>

**24.** Scenario — You're reviewing a design that says "if the primary doesn't respond for 500 ms, the secondary takes over and starts accepting writes". What questions do you ask?
<details><summary>Answer</summary>What prevents both from accepting writes if the primary is paused or partitioned rather than dead (fencing, epochs, majority-based leases)? Is 500 ms above the primary's GC pauses and network jitter, or will it fail over on blips? Who decides — the secondary alone, or a majority of observers? What happens to writes the primary acknowledged but hadn't replicated? How do clients learn the new leader, and what stops them writing to the old one? These are Chapters 13 and 21's questions; the design as stated is unsafe.</details>

---

## Chapter Summary

- Partial failure — some parts broken, others working, and no certain way to tell which — is what makes
  distributed systems hard; every remote call can end with an unknowable outcome.
- Networks lose, delay, duplicate, reorder, and partition messages, sometimes asymmetrically; a timeout
  can't distinguish slow from dead, so retries need idempotency.
- Choose timeouts from the latency tail, prefer adaptive detectors such as phi accrual, and require
  stronger evidence for costly actions like failover than for cheap ones like rerouting.
- Wall clocks drift and jump; measure durations with monotonic clocks and never order cross-machine events
  by wall time alone.
- Process pauses make a node's belief that it holds a lease or leadership unreliable; resources must reject
  stale holders with fencing tokens.
- Name the fault and timing model, never compromise safety for liveness, and test under partitions,
  pauses, and skew — ideally with deterministic simulation.

---

## Practice Exercises

1. Log a week of latency samples for a dependency you call, compute timeouts at p99 and p99.9 with
   §19.3's function, and estimate how many healthy requests each would have cut off at peak.
2. Write a test that injects a 3-second pause between a lease check and a write in your own code (or a
   library's), and observe whether anything prevents the stale write.
3. Read a published Jepsen analysis of a database you use and list each safety violation found, the fault
   that caused it, and whether it was fixed.
4. Find every use of wall-clock time to measure a duration in a codebase you know, and replace it with a
   monotonic clock.
5. Refactor a small service so time, randomness, and network I/O are injected, then write a seeded
   simulation test that reproduces a failure deterministically.

---

## Mini Project: A Fault-Injecting Network Simulator

Build a deterministic simulator in which several in-process "nodes" exchange messages through a simulated
network, so you can watch the failure modes of this chapter happen and replay them exactly.

- A seeded scheduler driving all time, randomness, and message delivery from one PRNG.
- Network faults: loss, delay distributions, duplication, reordering, and symmetric and asymmetric
  partitions, configurable per link and over time.
- Node faults: crashes, restarts with durable state, process pauses, and clock skew per node.
- A phi-accrual membership layer on top, reporting each node's view of who is alive over time.
- A scenario file format and a "replay seed N" command that reproduces any failing run exactly.

---

**Next:** [Chapter 20 — Consistency Models →](20-consistency-models.md)
