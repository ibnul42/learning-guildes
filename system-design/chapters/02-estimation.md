# Chapter 2 — Back-of-the-Envelope Estimation

## Learning Objectives

- Recall the latency and throughput numbers that decide architectures, to the right order of magnitude
- Convert daily users and actions into average and peak requests per second in your head
- Estimate storage, bandwidth, and cache size including replication, indexes, and retention
- Use Little's law to size concurrency — connection pools, worker counts, in-flight requests
- Sanity-check an estimate against known systems and present it so its assumptions can be challenged

---

## 2.1 Why Estimate at All

An estimate is the cheapest experiment you can run. Before anything is built, a few lines of
arithmetic can tell you whether the whole dataset fits in one machine's RAM, whether one database
can absorb the write rate, or whether your bandwidth bill will exceed your revenue. Those answers
change the architecture — and they're far cheaper to get from a napkin than from a load test.

The goal is the **order of magnitude**, not the digit. Whether a system needs 300 or 400 writes/s
doesn't change its design. Whether it needs 300 or 300,000 changes everything. So estimation is
aggressive rounding plus explicit assumptions:

- Round to one significant figure whenever it doesn't change the conclusion.
- Write every assumption down — "average post is 1 KB", "peak is 3× average" — so whoever
  disagrees can change that number, not argue with the conclusion.
- Compute the thing that decides the design first: usually peak write rate, total storage, or
  hot working-set size.

> 💼 **Interview** — Interviewers rarely care whether your QPS figure is 11,574 or 12,000. They
> care that you *can* get there in thirty seconds, that you separate reads from writes and average
> from peak, and that you then *use* the number — "12 k writes/s is more than I'd put on a single
> PostgreSQL primary, so I'll need to partition."

---

## 2.2 Numbers Worth Memorising

These are 2020s-hardware figures, rounded to be memorable. Exact values vary by a factor of two
or more across machines and clouds; the ratios between rows are what matter, and they've been
stable for a decade.

### Latency

| Operation | Approx. time | Mental model |
|---|---|---|
| L1 cache reference | 1 ns | |
| Branch mispredict | 3 ns | |
| L2 cache reference | 4 ns | |
| Uncontended mutex lock/unlock | 20 ns | |
| Main memory reference | 100 ns | 100× L1 |
| Compress 1 KB (LZ4/zstd-fast) | 1 µs | |
| Read 1 MB sequentially from RAM | 10–50 µs | |
| NVMe SSD random 4 KB read | 20–100 µs | ~1,000× RAM |
| Round trip, same data centre / zone | 0.1–0.5 ms | |
| Read 1 MB sequentially from NVMe | 0.2–0.5 ms | |
| Round trip, across zones in one region | 0.5–2 ms | |
| HDD seek | 5–10 ms | why random I/O on spinning disks was banned |
| Round trip, US East ↔ US West | 60–70 ms | |
| Round trip, US East ↔ Western Europe | 75–90 ms | |
| Round trip, US ↔ East Asia / Australia | 150–250 ms | the speed of light is the floor |

Four conclusions fall straight out of this table and recur in every chapter:

1. **Memory is ~1,000× faster than SSD, which is ~1,000× faster than a cross-continent round
   trip.** That's the whole argument for caching (Chapter 16) and for serving users from nearby
   regions (Chapters 6, 32).
2. **A network round trip within a data centre (~0.5 ms) costs as much as thousands of memory
   reads.** "Chatty" designs — N calls where one batched call would do — are slow because of
   round trips, not computation (Chapter 5).
3. **Cross-region round trips are tens to hundreds of milliseconds.** Any design that puts a
   synchronous cross-region call on the user's request path has spent most of its latency budget
   before doing any work. That's why synchronous cross-region replication is rare (Chapter 13).
4. **Sequential I/O is dramatically cheaper than random I/O,** even on SSDs. That's the reason
   log-structured storage engines exist (Chapter 10).

> ⚠️ **Gotcha** — Light in fibre travels at about 200,000 km/s — two-thirds of its vacuum speed —
> and real routes are far longer than great-circle distance. New York to London is ~5,600 km, so
> the theoretical minimum round trip is ~56 ms and the observed one is ~70–80 ms. No engineering
> removes that; only moving the computation closer does.

### Throughput (planning numbers per instance)

| Component | Rough capacity | Depends heavily on |
|---|---|---|
| Stateless app server (one instance, a few cores) | 1 k–10 k simple req/s | Work per request; language runtime |
| PostgreSQL / MySQL, large instance, simple indexed reads | 10 k–50 k queries/s | Working set in RAM; query shape |
| Same, durable writes | 1 k–20 k transactions/s | fsync cost, batching, index count |
| Redis, single instance | 100 k–200 k ops/s | Command complexity, pipelining |
| Kafka partition (planning figure) | ~10 MB/s | Message size, replication, consumers |
| 10 Gbps network interface | ~1.25 GB/s | — |
| NVMe SSD sequential | 2–7 GB/s | — |

These are deliberately conservative *planning* numbers, not benchmarks. If a design needs one
component to run at the top of its range, treat that as a risk to call out, not a plan.

### Time and size

| Fact | Value | Shortcut |
|---|---|---|
| Seconds per day | 86,400 | **≈ 10⁵** |
| Seconds per month | ~2.6 M | ≈ 2.5 × 10⁶ |
| Seconds per year | ~31.5 M | **≈ π × 10⁷** |
| 1 M per day | ≈ 12 per second | |
| 1 B per day | ≈ 12 k per second | |
| KB, MB, GB, TB, PB | 10³, 10⁶, 10⁹, 10¹², 10¹⁵ bytes | each step is ×1,000 |
| UUID | 16 bytes binary, 36 as text | store binary |
| Timestamp, int64 | 8 bytes | |
| 1080p video at 5 Mbit/s | ~37 MB per minute | bitrate ÷ 8 × seconds |

The single most useful shortcut is **one day ≈ 100,000 seconds**. It's 16% high, which is
harmless — and it makes "how many per second" a matter of dividing by 10⁵.

---

## 2.3 Traffic: From Users to Requests per Second

The standard chain is **users → actions per user → requests per day → requests per second →
peak**. Do it separately for reads and writes, because they usually hit different components and
differ by orders of magnitude.

Take a photo-sharing app:

- 50 M daily active users (DAU)
- Each user views 100 photos/day and uploads 0.2 photos/day (one in five users uploads one)

```js
// Traffic estimate for a photo-sharing app. Every assumption is a named constant
// so a reviewer can change one and re-run, rather than arguing with the result.
const DAU = 50e6;
const VIEWS_PER_USER = 100;
const UPLOADS_PER_USER = 0.2;
const SECONDS_PER_DAY = 86_400;
const PEAK_FACTOR = 3; // daily curve for a consumer app; see below

const readsPerSec = (DAU * VIEWS_PER_USER) / SECONDS_PER_DAY;
const writesPerSec = (DAU * UPLOADS_PER_USER) / SECONDS_PER_DAY;

console.log(Math.round(readsPerSec), Math.round(readsPerSec * PEAK_FACTOR));   // 57870 173611
console.log(Math.round(writesPerSec), Math.round(writesPerSec * PEAK_FACTOR)); // 116 347
console.log(`read:write = ${Math.round(readsPerSec / writesPerSec)}:1`);        // read:write = 500:1
```

In your head: 50 M × 100 = 5 × 10⁹ views/day; ÷ 10⁵ = 50,000/s; × 3 = 150,000/s at peak.
Uploads: 10 M/day ÷ 10⁵ = 100/s. That's close enough to the precise figures to make the same
decisions, and it took a few seconds.

The result already shapes the design. **The system is 500:1 read-heavy** — so the read path must
be served almost entirely from caches and a CDN, and the write path, at a few hundred uploads per
second, is small enough that a single well-provisioned database could record the metadata.

### Peak factor

Average traffic is a fiction; capacity is set by the peak. Typical ratios:

| Traffic shape | Peak ÷ average |
|---|---|
| Global consumer app, smooth diurnal curve | 2–3× |
| Single-country consumer app (everyone awake at once) | 3–5× |
| B2B, business hours only | 4–6× (traffic concentrated in ~8 hours) |
| Event-driven: ticket drops, sports, flash sales | 10–100× for minutes |

If the business has a known spike — Black Friday, a product launch, the New Year's Eve "happy new
year" message burst — estimate that spike directly rather than multiplying an average. Chapter 47
is a whole case study in the 100× case.

> ⚠️ **Gotcha** — DAU is not the same as concurrent users. 50 M DAU with an average session of 10
> minutes spread over the day gives roughly 50 M × 10 / 1,440 ≈ 350 k *concurrently* online on
> average, ~1 M at peak. Confusing the two inflates connection estimates by 50× (for WebSocket
> designs, Chapter 9) — or deflates them, if someone assumes "concurrent = 1% of DAU" without
> checking session length.

---

## 2.4 Storage

Storage estimates have more multipliers than traffic estimates, and teams routinely forget half
of them. The full chain:

```text
records/day × bytes/record × retention × replication × (1 + index & overhead) × growth headroom
```

Continuing the photo app. Each upload produces a metadata row and a set of image files:

| Item | Assumption | Per upload |
|---|---|---|
| Metadata row | IDs, user, caption, timestamps, location | ~1 KB |
| Original image | compressed JPEG/HEIC from a phone | ~2 MB |
| Derivatives | 3 sizes in 2 formats (see Chapter 1's design doc) | ~400 KB total |

```js
const UPLOADS_PER_DAY = 10e6;
const DAYS_PER_YEAR = 365;

const metaPerDay = UPLOADS_PER_DAY * 1e3;           // 1 KB rows
const blobPerDay = UPLOADS_PER_DAY * 2.4e6;         // original + derivatives

const TB = 1e12, PB = 1e15;
const metaYear = (metaPerDay * DAYS_PER_YEAR) / TB;
const blobYear = (blobPerDay * DAYS_PER_YEAR) / PB;

// Database: 3 replicas, indexes and page overhead roughly double raw row size
console.log(`metadata: ${metaYear.toFixed(1)} TB/yr raw, ${(metaYear * 3 * 2).toFixed(0)} TB/yr provisioned`);
// metadata: 3.6 TB/yr raw, 22 TB/yr provisioned

// Object storage handles redundancy internally (erasure coding, ~1.3–1.5× overhead), billed on logical bytes
console.log(`images: ${blobYear.toFixed(1)} PB/yr`); // images: 8.8 PB/yr
```

The conclusion again changes the architecture: metadata is a few terabytes a year — a size a
partitioned relational database or a wide-column store handles without drama — while images are
~9 *petabytes* a year. **Images must live in object storage** (Chapter 17); the database stores
keys. And at ~9 PB/year, storage tiering (moving old, rarely viewed photos to cheaper classes)
becomes a cost line worth an engineer's attention.

Multipliers people forget:

| Multiplier | Typical value | Why |
|---|---|---|
| Replication factor | 3× for databases | Durability and read capacity (Chapter 13) |
| Indexes | +30–100% of table size | Each secondary index is a copy of some columns (Chapter 10) |
| Storage-engine overhead | +10–50% | Page fill factor, MVCC dead tuples, LSM space amplification |
| Backups and snapshots | +1–2× | Often in a different storage class, still billed |
| Headroom | keep disks < 70–80% full | Compaction, vacuum, and recovery need free space |
| Growth | e.g. +50%/year | Year 3 size is what you provision for, not today's |

> ⚠️ **Gotcha** — Storing UUIDs as 36-character text instead of 16-byte binary more than doubles
> the size of every primary key *and* every index and foreign key that contains it. On a
> billion-row table with three indexes, that's tens of gigabytes of avoidable RAM pressure — and a
> lower cache hit rate for every query.

---

## 2.5 Bandwidth, Memory, and Cache Size

**Bandwidth** is traffic × payload. At peak the photo app serves 150,000 image views/s. If the
average served derivative is ~100 KB:

150,000 × 100 KB = 15 GB/s ≈ **120 Gbit/s** of egress at peak.

No origin should serve that directly; it's a CDN's job (Chapter 6). Egress is also often the
largest line in a cloud bill — at a few cents per GB, 15 GB/s average-ish for a day is on the order
of a petabyte a day of transfer. Estimating egress early has killed more than one product design.

**Cache size** follows from the *hot set*, not the total. Access to most content is heavily skewed
— a small fraction of items gets most of the reads. A common starting assumption is that caching
the top ~20% of a day's distinct read items captures most of the read volume; measure the real
skew as soon as you have traffic.

For the photo metadata: 5 × 10⁹ views/day touch, say, 200 M distinct photos. 20% of that is 40 M
items × 1 KB = 40 GB — a few cache nodes, or one large one. Cache the metadata, let the CDN cache
the bytes.

**Connections** need their own estimate when clients hold them open. 1 M concurrent users on a
WebSocket service at ~50 k connections per gateway node (a conservative figure for a tuned node)
is 20 nodes before redundancy — a calculation Chapter 37 does properly.

---

## 2.6 Little's Law: Sizing Concurrency

One formula does more work in capacity planning than any other:

```text
L = λ × W

L = average number of items in the system (in flight, in a queue, holding a connection)
λ = average arrival rate (items per second)
W = average time each item spends in the system (seconds)
```

It holds for any stable system regardless of the arrival distribution, which is what makes it so
useful. It answers "how many of X do I need?" whenever X is held for the duration of the work:

```js
const inFlight = (ratePerSec, secondsEach) => ratePerSec * secondsEach;

// 2,000 req/s, each taking 150 ms end to end → how many requests are in progress at once?
console.log(inFlight(2_000, 0.150)); // 300

// 5,000 DB queries/s at 4 ms each → how many connections are busy on average?
console.log(inFlight(5_000, 0.004)); // 20

// A worker pool processes 50 jobs/s; each job takes 3 s → minimum workers just to keep up
console.log(inFlight(50, 3)); // 150
```

The second example is why database connection pools are smaller than people expect: 5,000
queries/s needs only ~20 connections busy on average. A pool of 500 connections "for safety"
doesn't add throughput — it lets 500 queries contend for the same cores and locks, which makes
each one slower. The right pool size is a little above L at peak, not orders of magnitude above.

Little's law also explains a failure mode you'll meet in Chapter 28: when a dependency slows
from 4 ms to 400 ms, **W** grows 100×, so at the same arrival rate **L** — the number of
connections, threads, or memory held — grows 100× too. Pools exhaust, and a slow dependency becomes
a dead service. Timeouts exist to cap W.

> 💼 **Interview** — "How many database connections do you need?" is a Little's-law question.
> Answering "throughput × query time, plus headroom" and noting that more connections than cores
> usually hurts is a strong answer.

---

## 2.7 Sanity Checks and Presentation

An estimate is only as good as its weakest assumption, so check it before acting on it.

**Cross-check against a known system.** If your estimate says a chat app with 10 M DAU needs
2 PB of storage per year, compare to reality: that's 200 MB per user per year of text — absurd,
unless you've counted media. Public figures from large services (posts per day, storage per user,
peak messages per second at New Year) make excellent reference points.

**Compute it two ways.** Storage from the write rate (records/s × size × seconds) should agree
with storage from users (users × records per user × size). If they don't, one assumption is wrong.

**Check the unit at every step.** Most estimation errors are bits-vs-bytes (a factor of 8) or
per-day-vs-per-second (a factor of 86,400). Write the unit next to every number.

**Present assumptions before results.** A good estimate on a whiteboard looks like:

```text
Assumptions        Derived                        Decision
───────────        ───────                        ────────
50M DAU            reads  ~50k/s avg, 150k/s peak  serve reads from cache + CDN
100 views/user     writes ~100/s avg, 350/s peak   one metadata DB primary is fine
0.2 uploads/user   images ~9 PB/yr                 object storage, tiered by age
2.4 MB/upload      meta   ~4 TB/yr raw             relational, partition by year 2–3
peak = 3× avg      egress ~120 Gbit/s peak          CDN mandatory; budget egress
```

The third column is what makes it an *engineering* estimate rather than arithmetic. Each number
that doesn't lead to a decision is a number you didn't need to compute.

**Estimate cost when it could be decisive.** Rough late-2026 cloud list prices — object storage
around $0.02 per GB-month in the standard tier, internet egress a few cents per GB, a large
managed database instance a few thousand dollars per month — are enough to discover that a design
costs $3 M/year before anyone builds it. Treat specific prices as volatile and check the current
pricing page; treat the *magnitudes* as reliable.

---

## 2.8 Common Mistakes and Best Practices

**Mistake: designing for average load.** Capacity has to meet peak; average-based designs fail
every evening, or every launch day.

**Mistake: bits versus bytes.** Network speeds are quoted in bits per second, storage in bytes; a
10 Gbit/s link moves 1.25 GB/s, not 10.

**Mistake: forgetting the multipliers.** Raw data size ignores replication (3×), indexes, engine
overhead, backups, and disk headroom — together often 6–10× the raw figure.

**Mistake: precision theatre.** Quoting "11,574.07 QPS" signals arithmetic, not judgement; round,
and spend the time on what the number implies.

**Mistake: estimating everything except the bottleneck.** Ten derived numbers and no conclusion
wastes the exercise; compute the deciding quantity first and stop when the design is determined.

**Mistake: oversized connection pools.** Pools far above Little's-law L add contention, not
throughput, and can overload the database during a spike.

**Mistake: confusing DAU with concurrency.** Concurrent users depend on session length; a fixed
percentage of DAU is a guess presented as a fact.

**Best practice: keep assumptions as named, changeable inputs.** Whoever disagrees changes one
number and re-runs, instead of debating the conclusion.

**Best practice: round to one significant figure and use 10⁵ seconds per day.** Mental arithmetic
becomes fast enough to do while talking.

**Best practice: end each estimate with a decision.** "≈ 9 PB/year → object storage with
tiering" is the reason the estimate exists.

---

## 2.9 Real-World Example: A Reusable Capacity Model

A capacity model is a small program that turns a list of assumptions into the handful of numbers
that decide a design. Teams that keep one per service can answer "what happens if traffic
doubles?" in a minute, and design reviews argue about assumptions instead of arithmetic.

```js
// capacity-model.mjs — derive design-deciding numbers from explicit assumptions.
// Run: node capacity-model.mjs

const SEC_PER_DAY = 86_400;
const units = (n, steps, labels) => {
  let i = 0;
  while (n >= steps && i < labels.length - 1) { n /= steps; i++; }
  return `${n.toFixed(n < 10 ? 1 : 0)} ${labels[i]}`;
};
const bytes = (n) => units(n, 1000, ['B', 'KB', 'MB', 'GB', 'TB', 'PB', 'EB']);
const rate = (n) => units(n, 1000, ['/s', 'k/s', 'M/s']);

export function capacityModel(a) {
  // Validate early: a model that silently accepts a typo'd assumption produces confident nonsense
  for (const k of ['dau', 'readsPerUser', 'writesPerUser', 'bytesPerWrite', 'peakFactor']) {
    if (!(a[k] > 0)) throw new RangeError(`assumption ${k} must be > 0`);
  }
  const readAvg = (a.dau * a.readsPerUser) / SEC_PER_DAY;
  const writeAvg = (a.dau * a.writesPerUser) / SEC_PER_DAY;
  const rawPerYear = writeAvg * SEC_PER_DAY * 365 * a.bytesPerWrite;

  // Provisioned storage for the planning horizon: each year's new data is larger than the last,
  // and all of it accumulates; then replication, index overhead, and disk headroom apply
  let accumulated = 0;
  for (let y = 0; y < a.horizonYears; y++) accumulated += rawPerYear * (1 + a.yearlyGrowth) ** y;
  const provisioned = accumulated * a.replication * (1 + a.indexOverhead) / a.maxDiskFill;

  const peakRead = readAvg * a.peakFactor;
  const appServers = Math.ceil((peakRead + writeAvg * a.peakFactor) /
    (a.perServerRps * a.targetUtilisation)) + a.spareServers;

  return {
    assumptions: a,
    derived: {
      reads: `${rate(readAvg)} avg, ${rate(peakRead)} peak`,
      writes: `${rate(writeAvg)} avg, ${rate(writeAvg * a.peakFactor)} peak`,
      readWriteRatio: `${Math.round(readAvg / writeAvg)}:1`,
      rawStoragePerYear: bytes(rawPerYear),
      provisionedStorage: `${bytes(provisioned)} over ${a.horizonYears} yr`,
      cacheHotSet: bytes(a.dau * a.readsPerUser * a.distinctReadFraction * a.hotFraction * a.bytesPerRead),
      peakEgress: `${((peakRead * a.bytesPerRead * 8) / 1e9).toFixed(1)} Gbit/s`,
      appServers,
    },
  };
}

const model = capacityModel({
  dau: 20e6, readsPerUser: 40, writesPerUser: 2,
  bytesPerWrite: 1_500, bytesPerRead: 4_000, peakFactor: 3,
  replication: 3, indexOverhead: 0.6, maxDiskFill: 0.75,
  yearlyGrowth: 0.5, horizonYears: 3,
  distinctReadFraction: 0.1, hotFraction: 0.2,
  perServerRps: 2_000, targetUtilisation: 0.6, spareServers: 2,
});
console.table(model.derived);
// reads: '9.3 k/s avg, 28 k/s peak'      writes: '463 /s avg, 1.4 k/s peak'
// readWriteRatio: '20:1'                  rawStoragePerYear: '22 TB'
// provisionedStorage: '666 TB over 3 yr'  cacheHotSet: '64 GB'
// peakEgress: '0.9 Gbit/s'                appServers: 27
```

The decision worth defending is **returning the assumptions alongside the derived numbers instead
of only the results.** It looks redundant — the caller passed them in — but a capacity estimate
gets screenshotted, pasted into documents, and quoted in meetings months later, long after anyone
remembers that it assumed 40 reads per user. An estimate separated from its assumptions becomes an
unfalsifiable fact; one that carries them can be challenged, corrected, and re-run. That's the
same reason §2.7 puts the assumptions column first on the whiteboard.

---

## 2.10 Questions

**1.** MCQ — Roughly how many times slower is a random 4 KB read from an NVMe SSD than a main-memory reference? a) 2× b) 10× c) ~1,000× d) ~1,000,000×
<details><summary>Answer</summary>**c)** ~100 ns vs ~20–100 µs is about three orders of magnitude. That gap is what makes an in-memory cache worthwhile even in front of fast SSDs.</details>

**2.** MCQ — Which round trip is closest to 70–80 ms? a) Between two servers in one rack b) Between two availability zones in one region c) New York ↔ London d) Sydney ↔ London
<details><summary>Answer</summary>**c)** Transatlantic round trips are ~70–80 ms. Same-rack is microseconds, cross-zone is ~1 ms, and Sydney–London is well over 200 ms.</details>

**3.** MCQ — 1 billion events per day is approximately: a) 1,200/s b) 12,000/s c) 120,000/s d) 1.2 M/s
<details><summary>Answer</summary>**b)** 10⁹ ÷ 86,400 ≈ 11,600/s. With the 10⁵ shortcut: 10⁹ / 10⁵ = 10⁴.</details>

**4.** MCQ — A link is rated at 10 Gbit/s. About how long does transferring 100 GB take at full line rate? a) 10 s b) 80 s c) 800 s d) 10,000 s
<details><summary>Answer</summary>**b)** 10 Gbit/s = 1.25 GB/s; 100 / 1.25 = 80 s. Forgetting the bits-to-bytes factor gives the wrong answer of 10 s.</details>

**5.** MCQ — What does Little's law relate? a) Latency and bandwidth b) Items in the system, arrival rate, and time in system c) Availability and redundancy d) Cache size and hit rate
<details><summary>Answer</summary>**b)** L = λW: average items in the system equals arrival rate times average time spent in it.</details>

**6.** MCQ — Which multiplier is most often forgotten in database storage estimates? a) Peak factor b) Read/write ratio c) Secondary indexes and replication d) Seconds per day
<details><summary>Answer</summary>**c)** Indexes add 30–100% and replication typically 3×; together they can make provisioned size 6× raw. Peak factor and read/write ratio affect traffic, not storage.</details>

**7.** Short — Why is "one day ≈ 10⁵ seconds" an acceptable approximation?
<details><summary>Answer</summary>It's 16% high (86,400 vs 100,000), which never changes an architectural decision but makes per-second rates a single division by 10⁵ — fast enough to do while talking.</details>

**8.** Short — Why must reads and writes be estimated separately?
<details><summary>Answer</summary>They usually hit different components (caches/replicas vs the primary) and differ by orders of magnitude; a combined figure hides both the read-heavy shape that justifies caching and the write rate that limits a single primary.</details>

**9.** Short — Explain why a 500-connection database pool can be slower than a 30-connection pool for the same workload.
<details><summary>Answer</summary>If Little's law says ~25 connections are busy on average, the extra 470 only allow more queries to run concurrently on the same cores and locks, adding context switching and lock contention so each query slows down. Throughput doesn't rise; latency does, and a spike can overload the database.</details>

**10.** Short — How does Little's law explain why a slow dependency can take down a healthy service?
<details><summary>Answer</summary>At a fixed arrival rate, the number of in-flight requests (and the threads, connections, and memory they hold) scales with time-in-system. If the dependency gets 100× slower, held resources grow 100× until pools or memory are exhausted. Timeouts bound W and therefore L.</details>

**11.** Short — What is the "hot set", and why does cache sizing start from it rather than total data size?
<details><summary>Answer</summary>The hot set is the subset of items receiving most reads in a period. Because access is skewed, caching that subset captures most read volume; sizing to the total data would cost far more for little extra hit rate.</details>

**12.** Output — 30 M DAU each send 40 messages per day. Using a 3× peak factor and the 10⁵ seconds-per-day shortcut, what is the peak message rate?
<details><summary>Answer</summary>**~36,000 messages/s.** 30 M × 40 = 1.2 × 10⁹/day; ÷ 10⁵ = 12,000/s average; × 3 = 36,000/s.</details>

**13.** Output — What does this print?
```js
const L = (lambda, w) => lambda * w;
console.log(L(400, 0.25), L(400, 2.5));
```
<details><summary>Answer</summary>`100 1000` — the same 400 req/s holds 100 requests in flight at 250 ms, but 1,000 if latency rises to 2.5 s. Ten times the latency is ten times the held resources.</details>

**14.** Output — A service stores 5 M new 2 KB records per day, keeps them for 2 years, replicates 3×, and indexes add 50%. Ignoring growth and headroom, how much provisioned storage is needed, to one significant figure?
<details><summary>Answer</summary>**~33 TB, i.e. ~30 TB.** 5 M × 2 KB = 10 GB/day; × 730 days = 7.3 TB raw; × 3 = 21.9 TB; × 1.5 = ~33 TB.</details>

**15.** Output — A CDN serves 40,000 requests/s at peak with an average response of 250 KB. What is peak egress in Gbit/s?
<details><summary>Answer</summary>**80 Gbit/s.** 40,000 × 250 KB = 10 GB/s; × 8 = 80 Gbit/s.</details>

**16.** Debug — A team's estimate said their logging pipeline needed 2 Gbit/s; in production the network links saturate at 16 Gbit/s. The event count matched the estimate exactly. What went wrong?
<details><summary>Answer</summary>A bits/bytes confusion: they computed 2 GB/s (bytes) and labelled it Gbit/s, under-counting by 8×. Writing units beside every intermediate number catches this.</details>

**17.** Debug — A database was provisioned with 3× the estimated data size. Eight months in, disks hit 95% and writes stall, though row counts match the estimate. What was likely missed?
<details><summary>Answer</summary>Index size, storage-engine overhead (MVCC bloat, LSM space amplification, fill factor), WAL/snapshots, and the free space compaction or vacuum needs. The 3× covered replication only. Re-estimate with the full multiplier chain and keep disks under ~75% full.</details>

**18.** Debug — A WebSocket service was sized for "1% of DAU concurrent" — 100 k connections for 10 M DAU. On launch evening, 600 k connections arrived and the gateways fell over. What was wrong with the estimate?
<details><summary>Answer</summary>Concurrency depends on session length and the daily curve, not a fixed fraction. Long sessions (e.g. 60 min/day) concentrated in the evening give 10 M × 60 / 1,440 ≈ 420 k average, and more at peak. Estimate concurrency from session time and peak shape.</details>

**19.** Debug — After a downstream API's latency rose from 50 ms to 2 s during an incident, a service's memory climbed until it was OOM-killed, even though request volume was flat. Explain and fix.
<details><summary>Answer</summary>By Little's law, in-flight requests rose ~40× at constant arrival rate, each holding buffers, sockets, and promises. Fix: a timeout on the downstream call, a cap on concurrent outbound requests (bulkhead), and load shedding when the cap is reached (Chapters 28–29).</details>

**20.** Code — Write `perSecond(perDay, peakFactor = 1)` that returns a rounded per-second rate, and use it to compute the peak for 250 M events/day at 4× peak.
<details><summary>Answer</summary>

```js
const perSecond = (perDay, peakFactor = 1) => Math.round((perDay / 86_400) * peakFactor);
console.log(perSecond(250e6, 4)); // 11574
```
</details>

**21.** Code — Write `formatBytes(n)` that renders a byte count using decimal units (KB = 1,000) with one decimal place below 10 and none above, e.g. `formatBytes(3_700_000_000_000)` → `'3.7 TB'`.
<details><summary>Answer</summary>

```js
function formatBytes(n) {
  const labels = ['B', 'KB', 'MB', 'GB', 'TB', 'PB', 'EB'];
  let i = 0;
  while (n >= 1000 && i < labels.length - 1) { n /= 1000; i++; }
  return `${n.toFixed(n < 10 ? 1 : 0)} ${labels[i]}`;
}
console.log(formatBytes(3_700_000_000_000), formatBytes(512)); // 3.7 TB 512 B
```
Decimal units match how storage is sold and billed; binary units (KiB, MiB) are 2.4%–12.6% larger depending on the prefix, a gap that grows with each step.</details>

**22.** Code — Write `storageWithGrowth({ firstYearBytes, growth, years })` returning the total bytes stored after `years`, where each year's new data is `growth` larger than the previous year's (e.g. `growth = 0.5` for +50%).
<details><summary>Answer</summary>

```js
function storageWithGrowth({ firstYearBytes, growth, years }) {
  let total = 0, thisYear = firstYearBytes;
  for (let y = 0; y < years; y++) {
    total += thisYear;          // data accumulates; nothing expires in this model
    thisYear *= 1 + growth;
  }
  return total;
}
console.log(storageWithGrowth({ firstYearBytes: 10e12, growth: 0.5, years: 3 }) / 1e12); // 47.5
```
Three years at +50% stores 4.75× the first year — not 3× — which is why provisioning for "today × years" falls short.</details>

**23.** Scenario — A product manager asks whether a new analytics feature, logging every page view with 2 KB of context for 300 M daily page views, can go into the existing PostgreSQL cluster. How do you answer?
<details><summary>Answer</summary>Do the estimate out loud: 300 M × 2 KB = 600 GB/day, ~220 TB/year raw, and ~3,500 writes/s average, ~10 k at peak. That's an append-only, scan-heavy workload that would dwarf the transactional data and compete with it for I/O and vacuum. Recommend an event pipeline into a columnar store or warehouse (Chapters 11, 25). If the real need is a handful of aggregates, pre-aggregate and store only those — then PostgreSQL might be fine.</details>

**24.** Scenario — Your estimate shows a design needs 3 PB of hot SSD-backed storage in year one, and the business case assumed a small infrastructure budget. What do you do before changing the design?
<details><summary>Answer</summary>Challenge the assumptions that drive the number: retention period, whether originals need keeping, compression, replication factor, and what fraction is genuinely hot. Often most data is cold and can tier to object storage at a fraction of the cost, keeping only recent data on SSD. Bring the cost estimate to the product owner with the trade-off — "90-day hot retention cuts this by 80%" — because the right retention is a product decision, not an engineering one.</details>

---

## Chapter Summary

- Estimation exists to find the order of magnitude that decides the design; round aggressively,
  write assumptions down, and end every estimate with a decision.
- Memory, SSD, same-region network, and cross-continent network latencies are each roughly three
  orders of magnitude apart — that ratio is the argument for caching and for proximity.
- Traffic is users × actions ÷ 10⁵ seconds, done separately for reads and writes, then multiplied
  by a peak factor that matches the traffic's real shape.
- Provisioned storage is raw data times replication, indexes, engine overhead, backups, disk
  headroom, and growth — often an order of magnitude above the raw figure.
- Little's law (L = λW) sizes everything held for the duration of the work, and explains why a slow
  dependency exhausts pools even at constant traffic.
- Cross-check estimates two ways and against known systems; the classic errors are bits vs bytes,
  per-day vs per-second, and DAU vs concurrency.

---

## Practice Exercises

1. Without a calculator, estimate the average and peak requests per second for a service with
   80 M DAU making 25 requests each, then check your figure with code.
2. Estimate the three-year storage for a messaging app with 10 M DAU sending 30 messages of
   200 bytes each, with 3× replication, 50% index overhead, and 40% yearly growth.
3. Find the published round-trip latencies between two cloud regions you might use, and work out
   what fraction of a 200 ms p99 budget a synchronous cross-region call would consume.
4. Measure your own database: record queries per second and average query time for an hour, apply
   Little's law, and compare the result to your configured connection pool size.
5. Build a capacity model like §2.9 for a system you work on, then ask a colleague to change the
   two assumptions they trust least and see which conclusions flip.

---

## Mini Project: The Estimation Notebook

Build a small command-line tool that reads a JSON file of assumptions and prints a design-ready
estimate: average and peak reads and writes, storage per year and over a horizon with all
multipliers, hot-set cache size, peak egress, and server counts — each line tagged with the
assumptions it depends on.

- Every derived number lists the assumption keys it was computed from.
- A `--what-if key=value` flag re-runs with one assumption changed and shows the delta.
- Inputs are validated with clear errors (negative rates, fill factors above 1, missing keys).
- Output uses decimal units consistently and labels bits versus bytes explicitly.
- Include three example assumption files: a read-heavy content site, a chat app, and an IoT
  telemetry ingest pipeline.

---

**Next:** [Chapter 3 — Latency, Throughput, and Queueing →](03-latency-throughput-queues.md)
