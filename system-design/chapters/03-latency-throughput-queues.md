# Chapter 3 — Latency, Throughput, and Queueing

## Learning Objectives

- Distinguish latency from throughput and explain when improving one costs the other
- Report latency with percentiles, and explain why averaging percentiles across hosts is wrong
- Predict how fan-out amplifies tail latency, and choose a mitigation such as hedged requests
- Explain from queueing theory why latency explodes well before utilisation reaches 100%
- Allocate an end-to-end latency budget across the hops of a request path

---

## 3.1 Latency and Throughput Are Different Quantities

**Latency** is how long one operation takes, start to finish, as experienced by whoever is
waiting for it. **Throughput** is how many operations complete per unit of time. They are related
— Little's law (Chapter 2) ties them together through concurrency — but they are not two views of
one number, and optimising for one often costs the other.

A motorway makes the difference concrete. Adding lanes raises throughput (cars per hour) without
making any one car's trip faster. Raising the speed limit lowers latency per car. And a toll
booth that waits to batch ten cars before opening the barrier would raise the booth's efficiency
while making every car wait longer.

That last case is common in systems. **Batching** is the standard throughput lever:

| Technique | Throughput effect | Latency effect |
|---|---|---|
| Group-commit many transactions into one fsync | Far more commits per second | Each commit may wait for the batch |
| Kafka producer `linger.ms` | Fewer, larger network requests | Each message waits up to the linger time |
| Nagle's algorithm on TCP | Fewer small packets | Small writes delayed (Chapter 5) |
| Bulk inserts instead of row-by-row | Orders of magnitude more rows/s | The first row isn't durable until the batch is |
| Larger GPU/inference batches | Higher accelerator utilisation | Each request waits for the batch to fill |

None of these is wrong. The question is which one the requirement cares about: an interactive
API is latency-bound and should batch sparingly; an analytics ingest pipeline is throughput-bound
and should batch aggressively.

> 💼 **Interview** — "Is this system latency-sensitive or throughput-sensitive?" is a question you
> should *ask yourself* out loud early. It decides whether you reach for batching, async
> pipelines, and queues (throughput) or for caches, proximity, and fewer hops (latency).

---

## 3.2 Measure Percentiles, Not Averages

Latency distributions in real systems are skewed: most requests are fast, and a minority are
much slower because they hit a cache miss, a garbage-collection pause, a lock, a retransmitted
packet, or a cold disk. The **mean** of a skewed distribution describes almost no actual request.

```js
// 100 requests: 95 fast, 4 slow, 1 pathological. A typical real-world shape.
const latencies = [...Array(95).fill(20), ...Array(4).fill(200), 2_000];

const mean = latencies.reduce((a, b) => a + b, 0) / latencies.length;
// Nearest-rank percentile: the smallest value with at least p% of samples at or below it
const pct = (xs, p) => {
  const s = [...xs].sort((a, b) => a - b);
  return s[Math.min(s.length - 1, Math.ceil((p / 100) * s.length) - 1)];
};

console.log(mean, pct(latencies, 50), pct(latencies, 99), pct(latencies, 100)); // 47 20 200 2000
```

A mean of 47 ms corresponds to no request that happened: 95% of users saw 20 ms, and 5% saw ten
to a hundred times worse. Percentiles describe the distribution honestly:

| Statistic | Meaning | Use it for |
|---|---|---|
| p50 (median) | Half of requests are faster | The "typical" experience |
| p90 / p95 | The slow end of normal | User-facing targets for most products |
| p99 | 1 in 100 requests is slower | SLOs, capacity decisions |
| p99.9 | 1 in 1,000 | High-volume or fan-out services, where 1 in 1,000 is many users |
| max | The worst observed | Debugging only; one outlier dominates it |

Why care about 1 in 100? Because the users who see the tail are disproportionately your best
ones. A user who loads 50 pages per session, each making several requests, will hit your p99 on
nearly every visit. And the requests that are slow are often slow *for a reason* — the account
with the most data, the customer with the largest cart — which correlates with value.

> ⚠️ **Gotcha** — **You cannot average percentiles.** If host A's p99 is 30 ms and host B's is
> 890 ms, the fleet p99 is not 460 ms. The fleet p99 depends on how many requests each host
> served and on the full shape of both distributions. Averaging p99s across hosts, across minutes,
> or across endpoints produces a number with no meaning. Aggregate the underlying *distributions*
> (histograms) and compute the percentile from the merged result — §3.8 builds one.

### Where you measure changes the answer

Latency measured inside a request handler excludes time spent in the accept queue, the load
balancer, TLS, and the network. Latency measured by the client includes all of it. For SLOs
(Chapter 4) measure as close to the user as is practical — the edge load balancer at minimum —
because that's the latency the user experiences. Server-side handler timing is still useful, but
for attributing time, not for judging it.

---

## 3.3 Tail Latency Amplification

Modern requests fan out. Rendering a page might call 30 services; a search query might be
scattered to 100 index shards and gathered back. The response can't complete until the
**slowest** of those calls returns — so the overall latency is governed by the tail of each
component, not its median.

If each backend independently has a 1% chance of being slow (its p99), the chance that a request
touching *N* backends sees at least one slow response is 1 − 0.99ᴺ:

```js
const pAnySlow = (n, pSlow = 0.01) => 1 - (1 - pSlow) ** n;
for (const n of [1, 10, 100, 500]) console.log(n, `${(pAnySlow(n) * 100).toFixed(1)}%`);
// 1 1.0%
// 10 9.6%
// 100 63.4%
// 500 99.3%
```

At a fan-out of 100, **a backend's p99 becomes the request's median**. A backend team that
proudly reports "p99 = 50 ms" is, from the perspective of a fan-out caller, reporting its typical
contribution. That's why large search and feed systems track p99.9 and p99.99 for leaf services.

Mitigations, roughly in order of how often you should reach for them:

| Technique | How it works | Cost |
|---|---|---|
| **Reduce fan-out** | Fewer, coarser calls; batch lookups; precompute | Design effort |
| **Tighter per-call deadline + partial results** | Return what arrived by the deadline (search can omit one shard) | Answers may be incomplete — only valid where that's acceptable |
| **Hedged requests** | If no reply after ~p95, send a duplicate to another replica; use the first reply | ~5% extra load; requests must be safe to duplicate |
| **Tied requests** | Send to two replicas, each told about the other; whichever starts first cancels the other | Needs server cooperation |
| **Fix the cause of the tail** | GC tuning, avoiding head-of-line blocking, isolating noisy neighbours | Often the most effective, always the hardest |

Hedging is surprisingly cheap because it only fires for the slowest ~5% of calls. The published
experience from large search systems is that hedging after the p95 delay reduces p99.9 latency
dramatically for a few percent of extra load. The constraint is that the request must be
**idempotent** (Chapter 8) — hedging a "charge card" call would charge twice.

> ⚠️ **Gotcha** — Hedging everything immediately (sending two requests up front, always) doubles
> load, and doubled load pushes every backend up the queueing curve in §3.4 — which makes the tail
> *worse*. Hedge only after a delay near the p95, and cap the fraction of hedged requests.

---

## 3.4 Queueing: Why Latency Explodes Before 100%

This section is the most important idea in the chapter. Whenever requests arrive at a resource —
a CPU, a disk, a database connection, a worker — faster than it can momentarily serve them, they
wait in a queue. Requests don't arrive evenly; they arrive in random clumps. So even a resource
that's busy only 70% of the time on average has moments where several requests arrive together
and some must wait.

The simplest queueing model (one server, random arrivals, random service times — "M/M/1") gives
a result worth memorising. If the server takes **S** to serve a request on average and is busy a
fraction **ρ** (rho, the utilisation) of the time, the average time a request spends in the
system is:

```text
W = S / (1 − ρ)
```

| Utilisation ρ | Average time in system | Relative to the bare service time |
|---|---|---|
| 50% | 2 S | 2× |
| 70% | 3.3 S | 3.3× |
| 80% | 5 S | 5× |
| 90% | 10 S | 10× |
| 95% | 20 S | 20× |
| 99% | 100 S | 100× |

The curve is a hockey stick. From 50% to 80% busy, latency rises gently; from 90% to 99%, each
extra point of utilisation multiplies it. The simulation below reproduces the theory with
seeded randomness, and shows the tail too:

```js
// Single-server FIFO queue with random (exponential) arrivals and service times.
// A seeded PRNG makes every run print the same numbers.
function mulberry32(seed) {
  return () => {
    seed |= 0; seed = (seed + 0x6d2b79f5) | 0;
    let t = Math.imul(seed ^ (seed >>> 15), 1 | seed);
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t;
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}
const exp = (rand, mean) => -Math.log(1 - rand()) * mean; // exponential sample with this mean

function simulate({ utilisation, serviceMs = 10, n = 200_000, seed = 42 }) {
  const rand = mulberry32(seed);
  const interArrival = serviceMs / utilisation;   // arrival rate λ = ρ × μ
  let clock = 0, serverFreeAt = 0;
  const times = [];
  for (let i = 0; i < n; i++) {
    clock += exp(rand, interArrival);             // the next request arrives
    const start = Math.max(clock, serverFreeAt);  // it waits if the server is still busy
    serverFreeAt = start + exp(rand, serviceMs);
    times.push(serverFreeAt - clock);             // queueing + service
  }
  times.sort((a, b) => a - b);
  const p = (q) => times[Math.floor(q * (times.length - 1))];
  return { mean: times.reduce((a, b) => a + b, 0) / n, p50: p(0.5), p99: p(0.99) };
}

for (const u of [0.5, 0.7, 0.8, 0.9, 0.95]) {
  const r = simulate({ utilisation: u });
  console.log(`${u * 100}% busy  mean ${r.mean.toFixed(0)} ms  p50 ${r.p50.toFixed(0)} ms  p99 ${r.p99.toFixed(0)} ms`);
}
// 50% busy  mean 20 ms  p50 14 ms  p99 92 ms
// 70% busy  mean 33 ms  p50 23 ms  p99 152 ms
// 80% busy  mean 50 ms  p50 35 ms  p99 234 ms
// 90% busy  mean 100 ms  p50 69 ms  p99 438 ms
// 95% busy  mean 199 ms  p50 137 ms  p99 910 ms
```

The service time never changed — it's 10 ms on average throughout. Every millisecond above 10 is
time spent *waiting*. At 95% utilisation, a request that needs 10 ms of work has a 1-in-100 chance
of taking nearly a second.

Real systems are not exactly M/M/1, but the shape holds generally, with two adjustments:

- **Multiple servers sharing one queue** (a pool of workers, a multi-core CPU) tolerate higher
  utilisation before the knee, because a single slow request no longer blocks everything behind
  it. That's one reason one shared queue feeding N workers beats N separate queues.
- **More variable service times make it worse.** Queue wait grows with the variance of service
  time, which is why one occasional 2-second query on a database otherwise doing 2 ms queries
  wrecks the p99 for everyone.

The practical consequences shape capacity planning everywhere in this book:

1. **Plan for 50–70% peak utilisation** on latency-sensitive resources. The "wasted" 30–50% is
   what buys you a flat latency curve. Chapter 1's `serversNeeded` took a target utilisation for
   exactly this reason.
2. **CPU at 80% is not "20% headroom".** On the hockey stick, going from 80% to 90% doubles
   latency. Alerts should fire on latency and saturation, not just when CPU nears 100%.
3. **Small load increases near saturation cause large latency increases** — so a 10% traffic
   spike on a hot system looks like an outage, and a retry storm (Chapter 28) is self-reinforcing.
4. **Queues must be bounded.** An unbounded queue at a saturated server only converts overload
   into ever-increasing latency until something times out. Better to reject early (Chapter 29).

> 💼 **Interview** — "Why not run servers at 90% CPU to save money?" is a queueing question. The
> answer is the 1/(1−ρ) curve: latency at 90% is roughly double that at 80% and five times that
> at 50%, and the tail is worse still, so the savings are paid for in user-visible latency and in
> fragility to spikes.

---

## 3.5 Why Adding Machines Stops Helping

If one server handles 1,000 requests/s, do 64 servers handle 64,000? Only if the work is
perfectly parallel. Two effects get in the way:

- **Contention (σ)**: part of the work is serialised — a single lock, a single primary database,
  one hot row. **Amdahl's law** says that if a fraction σ of the work is serial, speed-up is capped
  at 1/σ no matter how many machines you add. 5% serial work caps you at 20×.
- **Coherence (κ)**: nodes must coordinate — cache invalidation broadcasts, distributed locks,
  cluster membership gossip, cross-shard queries. This cost grows with the *number of pairs* of
  nodes, roughly N², and it's what makes throughput eventually go *down* as you add machines.

The **Universal Scalability Law** combines both:

```text
throughput(N) = N / (1 + σ(N − 1) + κ N(N − 1))
```

```js
// Relative throughput with 5% serialised work and a small pairwise coordination cost
const usl = (n, sigma, kappa) => n / (1 + sigma * (n - 1) + kappa * n * (n - 1));
for (const n of [1, 8, 16, 32, 64, 128]) console.log(n, usl(n, 0.05, 0.001).toFixed(1));
// 1 1.0
// 8 5.7
// 16 8.0
// 32 9.0
// 64 7.8
// 128 5.4
```

With these parameters, the cluster peaks at about 32 nodes delivering 9× the throughput of one,
and 128 nodes deliver *less* than 16. Coordination costs are why "just add more nodes" eventually
fails, and why the designs in Parts III and IV work so hard to keep operations local to one
partition: a design whose hot path touches a single partition has κ near zero.

You can estimate σ and κ for a real system by measuring throughput at a few cluster sizes and
fitting the curve — far cheaper than discovering the peak in production.

---

## 3.6 Measuring Latency Without Lying to Yourself

Load tests are where most latency numbers come from, and most load tests under-report latency.

**Coordinated omission** is the main reason. A *closed-loop* load generator sends a request,
waits for the response, then sends the next. When the server stalls for 2 seconds, the generator
stalls with it — it simply doesn't send the requests that would have arrived during those 2
seconds. Real users don't coordinate with your server like that; they keep arriving. So the test
records *one* slow request where reality would have had hundreds, and the reported p99 can be
off by orders of magnitude.

| | Closed-loop generator | Open-loop generator |
|---|---|---|
| Next request sent when | The previous one completes | On a schedule, regardless of responses |
| Models | A fixed pool of users who wait | Independent arrivals, like real traffic |
| During a server stall | Stops sending — hides the stall | Keeps sending — queue builds, as in production |
| Good for | Measuring max throughput | Measuring latency at a given rate |

Use an open-loop (constant arrival-rate) generator when the question is "what's the p99 at
5,000 requests/s?", and measure each request's latency from its *intended* send time, not its
actual one.

Other measurement traps:

- **Testing with a warm cache and a tiny dataset.** Production has a cold tail and a working set
  bigger than RAM; the test has neither.
- **Single-client tests.** One client from one machine hits one load-balancer target and never
  exercises connection limits or uneven routing.
- **Ignoring the client's own limits.** A load generator at 100% CPU reports its own queueing as
  server latency.

---

## 3.7 Latency Budgets

A latency target for a user-facing operation is a budget, and every hop spends some of it.
Writing the budget down early prevents the situation where every team meets its own target and
the product still misses its one.

For a p99 target of 300 ms on "load the home feed", measured at the edge:

```text
client ─▶ edge/LB ─▶ API gateway ─▶ feed service ─┬─▶ timeline store
                                                  ├─▶ ranking service
                                                  └─▶ user/profile service (batched)
```

| Hop | Budget (p99) | Notes |
|---|---|---|
| TLS termination, LB, gateway | 15 ms | Connection reuse makes this mostly constant |
| Feed service own work | 30 ms | Serialisation, merging |
| Timeline store read | 40 ms | Called in parallel with profile lookups |
| Ranking service | 120 ms | Most expensive; candidate for a deadline + fallback ordering |
| Profile lookups (batched) | 40 ms | Parallel with timeline; batched to avoid fan-out |
| Headroom | 95 ms | For retries, GC, the unexpected |

Three rules make the budget useful:

1. **Serial hops add; parallel hops take the max** — but the max of several tails is worse than
   any one tail (§3.3). Budget parallel branches with that in mind.
2. **Percentiles don't add exactly** — the p99 of a sum is not the sum of p99s (it's usually
   somewhat less, because not every hop is slow at once). Summing p99s is a conservative upper
   bound, which is the right direction to err for a budget.
3. **Every hop gets a timeout derived from the budget**, and the caller passes its remaining
   deadline downstream so a callee doesn't keep working on a request its caller has already
   abandoned (deadline propagation, Chapter 28).

---

## 3.8 Common Mistakes and Best Practices

**Mistake: reporting averages.** The mean of a skewed latency distribution describes no real
request and hides the tail users actually complain about.

**Mistake: averaging percentiles.** p99s from different hosts, minutes, or endpoints can't be
averaged; merge histograms and compute the percentile once.

**Mistake: targeting high utilisation.** Running latency-sensitive resources at 85–95% puts them
on the steep part of the queueing curve, where a small spike becomes an outage.

**Mistake: unbounded queues.** A queue that can grow without limit converts overload into
unbounded latency instead of fast, honest rejections.

**Mistake: hedging without a delay or a cap.** Duplicating every request doubles load and makes
the tail worse; hedge after the p95 delay, for idempotent calls only.

**Mistake: closed-loop latency tests.** They silently omit the requests that would have arrived
during a stall, under-reporting p99 by orders of magnitude.

**Mistake: optimising leaf p50 in a fan-out system.** With a fan-out of 100, the leaf's p99 is the
caller's median; the tail is what matters.

**Best practice: set capacity by a latency target at a utilisation, not by maximum throughput.**
"5,000 req/s at p99 < 100 ms" is a capacity; "8,000 req/s before falling over" is not.

**Best practice: keep a latency budget per user-facing operation.** Each hop's timeout derives
from it, and regressions are attributable to a hop.

**Best practice: record latency as mergeable histograms.** They support any percentile, any
aggregation, and any time window after the fact.

---

## 3.9 Real-World Example: A Mergeable Latency Histogram

Metrics systems need to compute percentiles across hosts and time windows without storing every
sample. The standard answer is a histogram with geometrically growing buckets: memory is fixed,
relative error is bounded, and two histograms merge by adding counts. This is the idea behind
HDR Histogram, DDSketch, and the native histograms in modern metrics systems (Chapter 48).

```js
// latency-histogram.mjs — fixed-memory, mergeable latency histogram.
// Buckets grow geometrically, so relative error is bounded (about ±2.5% at 5% growth)
// from microseconds to minutes, and histograms from different hosts merge by addition.

export class LatencyHistogram {
  static GROWTH = 1.05;   // each bucket is 5% wider than the one before
  static MIN_MS = 0.01;   // anything faster lands in bucket 0
  static BUCKETS = 400;   // 0.01 ms × 1.05^400 ≈ 3 million ms (~50 min): beyond any timeout

  #counts = new Float64Array(LatencyHistogram.BUCKETS);
  #total = 0;
  #max = 0;

  #index(ms) {
    if (!(ms > LatencyHistogram.MIN_MS)) return 0; // also catches NaN and negatives
    const i = Math.floor(Math.log(ms / LatencyHistogram.MIN_MS) / Math.log(LatencyHistogram.GROWTH));
    return Math.min(i, LatencyHistogram.BUCKETS - 1); // overflow clamps into the last bucket
  }

  // Midpoint of bucket i: the value reported for every sample that landed in it
  static #value(i) {
    const lo = LatencyHistogram.MIN_MS * LatencyHistogram.GROWTH ** i;
    return (lo * (1 + LatencyHistogram.GROWTH)) / 2;
  }

  record(ms, count = 1) {
    this.#counts[this.#index(ms)] += count;
    this.#total += count;
    if (ms > this.#max) this.#max = ms;
  }

  // Per-host histograms add up to an exact fleet histogram — per-host percentiles never can
  merge(other) {
    for (let i = 0; i < LatencyHistogram.BUCKETS; i++) this.#counts[i] += other.#counts[i];
    this.#total += other.#total;
    this.#max = Math.max(this.#max, other.#max);
    return this;
  }

  percentile(p) {
    if (this.#total === 0) return NaN;
    if (p >= 100) return this.#max;              // the true max, not a bucket midpoint
    const rank = (p / 100) * this.#total;
    let seen = 0;
    for (let i = 0; i < LatencyHistogram.BUCKETS; i++) {
      seen += this.#counts[i];
      if (seen >= rank) return Math.min(LatencyHistogram.#value(i), this.#max);
    }
    return this.#max;
  }

  get count() { return this.#total; }

  // Compact snapshot for shipping to a metrics backend: only non-empty buckets
  toJSON() {
    const buckets = {};
    this.#counts.forEach((c, i) => { if (c) buckets[i] = c; });
    return { growth: LatencyHistogram.GROWTH, minMs: LatencyHistogram.MIN_MS, buckets, max: this.#max };
  }
}

// Two hosts with equal traffic: one healthy, one with a slow tail
const a = new LatencyHistogram(), b = new LatencyHistogram();
for (let i = 0; i < 9_900; i++) a.record(20 + (i % 10));   // 20–29 ms
for (let i = 0; i < 100; i++) a.record(400);
for (let i = 0; i < 9_000; i++) b.record(25 + (i % 10));
for (let i = 0; i < 1_000; i++) b.record(900);

const p99 = (h) => h.percentile(99).toFixed(0);
console.log('host A p99', p99(a), '| host B p99', p99(b));   // host A p99 29 | host B p99 887
console.log('average of p99s', ((a.percentile(99) + b.percentile(99)) / 2).toFixed(0)); // average of p99s 458
const fleet = new LatencyHistogram().merge(a).merge(b);
console.log('fleet p99', p99(fleet), 'of', fleet.count);      // fleet p99 887 of 20000
```

The decision a reader might question is **reporting a bucket midpoint, with up to ±2.5% error,
instead of storing raw samples and computing exact percentiles.** Exact percentiles need every
sample — at 50,000 requests/s that's 4 billion numbers a day per service — and, worse, they can't
be combined: the fleet p99 requires the raw samples from every host in one place. A bucketed
histogram is a few kilobytes regardless of traffic, ships cheaply every few seconds, and merges
exactly, so the fleet number above is correct while the "average of p99s" is off by a factor of
two. A few percent of relative error on a latency value is far below the noise between two runs
of the same load test; the inability to aggregate is a much bigger error.

---

## 3.10 Questions

**1.** MCQ — Enabling write batching in a database driver doubles throughput. What most likely happens to latency per write? a) It halves b) It's unchanged c) It may increase, because writes wait for a batch d) It becomes zero
<details><summary>Answer</summary>**c)** Batching amortises fixed costs across many operations, raising throughput, but individual writes can wait for the batch to fill or the linger timer to fire.</details>

**2.** MCQ — Which statistic best represents the worst experience of a regular user who makes hundreds of requests per session? a) Mean b) p50 c) p99 or p99.9 d) Minimum
<details><summary>Answer</summary>**c)** A user making hundreds of requests will almost certainly hit the 1-in-100 or 1-in-1,000 slow request; the tail is their normal.</details>

**3.** MCQ — A request fans out to 50 backends in parallel, each independently slow 1% of the time. Roughly what fraction of requests see at least one slow backend? a) 1% b) 5% c) 40% d) 99%
<details><summary>Answer</summary>**c)** 1 − 0.99⁵⁰ ≈ 39.5%.</details>

**4.** MCQ — In the M/M/1 model, what happens to average time in system when utilisation goes from 80% to 90%? a) Rises by 10% b) Roughly doubles c) Is unchanged d) Falls
<details><summary>Answer</summary>**b)** W = S/(1−ρ): 1/0.2 = 5 S versus 1/0.1 = 10 S.</details>

**5.** MCQ — What does the coherence term (κ) in the Universal Scalability Law represent? a) Disk speed b) The serial fraction of work c) The cost of coordination between pairs of nodes d) Network bandwidth
<details><summary>Answer</summary>**c)** Coherence grows with roughly N² node pairs — cache invalidation, distributed locking, cross-node queries — and is what makes throughput decline beyond a peak.</details>

**6.** MCQ — Which load-testing approach exposes coordinated omission? a) Closed-loop with more threads b) Open-loop at a fixed arrival rate, measuring from intended send time c) Measuring only server-side handler time d) Using a warmer cache
<details><summary>Answer</summary>**b)** Open-loop generators keep sending during stalls, so the queue builds as it would in production, and measuring from the intended send time counts the wait.</details>

**7.** Short — Why does the mean misrepresent a skewed latency distribution?
<details><summary>Answer</summary>A few very slow requests pull the mean far above the typical value while still hiding how bad the tail is, so it describes neither the common case nor the worst cases. Percentiles report both.</details>

**8.** Short — Why must hedged requests be idempotent?
<details><summary>Answer</summary>A hedged request may be executed twice — both the original and the duplicate can reach servers and complete. Side effects like charging a card or sending a message would happen twice unless the operation is safe to repeat.</details>

**9.** Short — Why does a single shared queue feeding N workers have better latency than N separate per-worker queues at the same utilisation?
<details><summary>Answer</summary>With separate queues, a request can wait behind a slow job while another worker sits idle. A shared queue sends each request to whichever worker frees up first, so one slow job delays far fewer requests.</details>

**10.** Short — What does Amdahl's law say about a system where 10% of the work is serialised?
<details><summary>Answer</summary>Its speed-up is capped at 1/0.1 = 10× no matter how many machines are added, because the serial portion can't be parallelised.</details>

**11.** Short — Why should a caller pass its remaining deadline to the services it calls?
<details><summary>Answer</summary>So downstream services stop working on a request the caller has already abandoned. Without it, a timed-out request keeps consuming capacity across the call tree, adding load at exactly the moment the system is struggling.</details>

**12.** Output — What does this print?
```js
const W = (serviceMs, rho) => serviceMs / (1 - rho);
console.log([0.5, 0.75, 0.95].map((rho) => Math.round(W(4, rho))));
```
<details><summary>Answer</summary>`[ 8, 16, 80 ]` — the same 4 ms of work takes 2×, 4×, and 20× as long as utilisation rises.</details>

**13.** Output — What does this print?
```js
const pAny = (n, p) => 1 - (1 - p) ** n;
console.log(pAny(20, 0.001).toFixed(3), pAny(200, 0.001).toFixed(3));
```
<details><summary>Answer</summary>`0.020 0.181` — at a leaf p99.9, 20-way fan-out puts ~2% of requests in the tail; 200-way puts ~18% there.</details>

**14.** Output — A serial request path has three hops with p99s of 20 ms, 50 ms, and 30 ms, plus a parallel branch whose two calls have p99s of 40 ms and 60 ms. What conservative p99 budget does the path need (summing p99s, taking the parallel branch's max)?
<details><summary>Answer</summary>**160 ms.** 20 + 50 + 30 + max(40, 60) = 160 ms. Summing p99s is an upper bound; the real p99 is usually somewhat lower — but note the parallel branch's tail is worse than either call's alone.</details>

**15.** Output — Using the USL with σ = 0.1 and κ = 0, what is the relative throughput of 10 nodes, to one decimal place?
<details><summary>Answer</summary>**5.3.** 10 / (1 + 0.1 × 9) = 10 / 1.9 ≈ 5.26. With no coherence cost this is Amdahl's law, approaching 10× asymptotically.</details>

**16.** Debug — A service's dashboard shows p99 averaging 120 ms across its 40 hosts, but the client team measures p99 above 600 ms. Both are measured correctly at the same point. What's wrong with the dashboard?
<details><summary>Answer</summary>It averages per-host p99s. A few hosts with a bad tail have a large effect on the true fleet p99 but are diluted by the averaging. Ship histograms per host, merge them, and compute the fleet percentile from the merged distribution.</details>

**17.** Debug — A database runs at 70% CPU and p99 query latency is 15 ms. A marketing campaign raises traffic 25%; CPU goes to ~88% and p99 jumps to 120 ms. The team suspects a bad query plan. What else explains it?
<details><summary>Answer</summary>The queueing curve: moving from 70% to ~88% utilisation multiplies waiting time several-fold (1/0.3 ≈ 3.3 vs 1/0.12 ≈ 8.3), and the tail grows faster than the mean. No plan change is needed to explain it; capacity planning at 50–70% peak utilisation is the fix.</details>

**18.** Debug — A load test reports p99 = 25 ms at 3,000 requests/s. In production at the same rate, p99 is 1.4 s during brief database stalls. The test used 50 threads, each sending a request after the previous completed. Why the discrepancy?
<details><summary>Answer</summary>Coordinated omission: during a stall, the closed-loop threads stopped sending, so the test recorded only the 50 in-flight slow requests rather than the hundreds that real traffic would have queued. Re-run with an open-loop generator at a fixed arrival rate, measuring from intended send times.</details>

**19.** Debug — After enabling hedged requests on all calls to a storage tier with zero delay, p99 got worse and storage CPU rose 90%. Explain and fix.
<details><summary>Answer</summary>Immediate hedging duplicated nearly every request, almost doubling storage load and pushing it up the queueing curve, which lengthened the tail it was meant to cut. Hedge only after a delay near the p95 latency, cap hedges to a few percent of requests, and cancel the loser when possible.</details>

**20.** Code — Write `percentile(samples, p)` using the nearest-rank method, without mutating the input.
<details><summary>Answer</summary>

```js
function percentile(samples, p) {
  if (samples.length === 0) return NaN;
  const sorted = [...samples].sort((a, b) => a - b); // copy: callers' arrays stay untouched
  const rank = Math.ceil((p / 100) * sorted.length);
  return sorted[Math.min(sorted.length - 1, Math.max(0, rank - 1))];
}
console.log(percentile([5, 1, 9, 3, 7], 50), percentile([5, 1, 9, 3, 7], 100)); // 5 9
```
</details>

**21.** Code — Write `hedged(fn, { delayMs })` that calls `fn()`, and if it hasn't settled after `delayMs`, calls it a second time, resolving with whichever result arrives first. Each call receives an `AbortSignal` that is aborted once the other wins.
<details><summary>Answer</summary>

```js
function hedged(fn, { delayMs }) {
  return new Promise((resolve, reject) => {
    const controllers = [];
    let settled = false, failures = 0, timer = null;

    const launch = () => {
      const c = new AbortController();
      controllers.push(c);
      fn(c.signal).then(
        (value) => {
          if (settled) return;
          settled = true;
          clearTimeout(timer);
          controllers.forEach((other) => { if (other !== c) other.abort(); }); // cancel the loser
          resolve(value);
        },
        (err) => {
          // Fail only once every launched attempt has failed and no hedge is still scheduled
          if (++failures === controllers.length && timer === null && !settled) {
            settled = true;
            reject(err);
          }
        },
      );
    };

    launch();
    timer = setTimeout(() => { timer = null; if (!settled) launch(); }, delayMs);
  });
}

// The first attempt takes 300 ms; the hedge fires at 50 ms and answers in 20 ms
let calls = 0;
const slowThenFast = (signal) => new Promise((resolve) => {
  const id = ++calls, t = setTimeout(() => resolve(`attempt ${id}`), id === 1 ? 300 : 20);
  signal.addEventListener('abort', () => clearTimeout(t));
});
console.log(await hedged(slowThenFast, { delayMs: 50 })); // attempt 2
```
A production version also caps the fraction of requests that get hedged; the key ideas are the delay before the duplicate and cancelling the loser.</details>

**22.** Code — Write `utilisationFor({ serviceMs, targetMeanMs })` that returns the highest utilisation at which an M/M/1 server keeps its mean time in system at or below the target.
<details><summary>Answer</summary>

```js
function utilisationFor({ serviceMs, targetMeanMs }) {
  // W = S / (1 − ρ)  ⇒  ρ = 1 − S / W
  if (targetMeanMs <= serviceMs) return 0; // can't beat the bare service time
  return 1 - serviceMs / targetMeanMs;
}
console.log(utilisationFor({ serviceMs: 10, targetMeanMs: 40 })); // 0.75
```
</details>

**23.** Scenario — A search service scatters each query to 120 shards. The p99 target is 200 ms, and each shard's p99 is 80 ms but its p99.9 is 900 ms. Leadership proposes buying faster hardware for every shard. What do you propose instead?
<details><summary>Answer</summary>The tail is the problem, not the median: with 120 shards, ~11% of queries hit at least one shard's p99.9. First, set a per-shard deadline (~150 ms) and return partial results with a marker, if product accepts slightly incomplete results. Second, hedge shard requests after their p95 to a replica, capped at a few percent. Third, investigate what causes the 900 ms tail (GC, compaction, noisy neighbours). Faster hardware lowers the median and rarely fixes the tail; buy it only if profiling shows shards are CPU-bound.</details>

**24.** Scenario — Finance wants to raise target CPU utilisation on the API fleet from 50% to 85% to cut costs by ~40%. The API has a p99 SLO of 250 ms; current p99 is 140 ms. How do you respond?
<details><summary>Answer</summary>Quantify instead of refusing: queueing theory predicts mean wait at 85% to be several times that at 50%, with the tail worse, and a 10% spike at 85% leaves no headroom at all. Propose a staged test — raise the target to 60%, then 70%, in production, watching p99 and SLO burn — and stop at the knee. Combine with autoscaling on latency or concurrency rather than CPU, and with load shedding for spikes. The likely outcome is ~65–70%, which captures much of the saving without breaking the SLO.</details>

---

## Chapter Summary

- Latency and throughput are different quantities; batching trades one for the other, so decide
  first which one the requirement cares about.
- Report latency as percentiles measured near the user, and aggregate by merging histograms —
  averaging percentiles produces meaningless numbers.
- Fan-out turns a component's tail into the caller's typical case; reduce fan-out, set deadlines,
  and hedge idempotent calls after a delay near the p95.
- Queueing delay grows as 1/(1 − utilisation), so latency-sensitive resources should run at
  50–70% peak utilisation and queues must be bounded.
- Serialised work caps speed-up and pairwise coordination eventually reverses it; keeping the hot
  path within one partition is what keeps systems scaling.
- Load tests must be open-loop to avoid coordinated omission, and every user-facing operation
  needs a latency budget that sets each hop's timeout.

---

## Practice Exercises

1. Run the §3.4 simulation with service times that are constant instead of exponential, then with
   a 1% chance of a 20× slower request. Compare the p99 curves and explain the differences.
2. Extend the simulation to *k* servers sharing one queue, and find the utilisation at which p99
   reaches 5× the service time for k = 1, 4, and 16.
3. Read the documentation for your load-testing tool and determine whether it is closed-loop or
   open-loop by default, and how it handles coordinated omission.
4. Instrument an endpoint you own with the §3.9 histogram (or your metrics library's native
   histogram), and compare the merged fleet p99 to the average of per-host p99s for a day.
5. Write a latency budget for a user-facing operation you own, derive a timeout for each hop from
   it, and check whether the current timeouts are consistent with the budget.

---

## Mini Project: A Queueing Laboratory

Build a small discrete-event simulator for a service: requests arrive at a configurable rate,
pass through one or more stages (each a pool of servers with its own service-time distribution
and a bounded queue), and are dropped when a queue is full or a deadline expires. The point is to
make the curves in this chapter something you've *seen* rather than memorised.

- Stages are configurable: server count, service-time distribution (constant, exponential,
  bimodal), queue capacity, and timeout.
- Output a table of throughput, p50/p99/p99.9, and drop rate for a sweep of arrival rates.
- Implement hedging for one stage and show its effect on the tail and on load.
- Show that unbounded queues lead to unbounded latency past saturation, and that bounded queues
  keep the latency of *accepted* requests flat while rejecting the excess.
- Seed all randomness so results are reproducible.

---

**Next:** [Chapter 4 — Availability, SLOs, and Failure Math →](04-availability-slos.md)
