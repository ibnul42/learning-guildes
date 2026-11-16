# Chapter 25 — Stream and Batch Processing

## Learning Objectives

- Distinguish bounded from unbounded data, and choose batch, micro-batch, or stream processing for a job
- Design batch jobs that are deterministic, rerunnable, and atomic in their output
- Reason about event time versus processing time, and use windows and watermarks to aggregate unbounded data
- Explain how stream processors keep state correct across failures with checkpoints and replay
- Join streams with streams and with tables, and choose between lambda and kappa architectures

---

## 25.1 Bounded and Unbounded Data

Most of this book has been about **online** systems: a request arrives, a response leaves, and latency is
measured in milliseconds. A large share of real systems' work is **derived data** computed in the background
— daily revenue reports, search indexes (Chapter 18), recommendation features, fraud scores, billing totals,
cleaned copies of raw events for analysts. Two families of systems compute it, distinguished by the data they
consume:

| | Batch processing | Stream processing |
|---|---|---|
| Input | **Bounded**: a fixed dataset (yesterday's logs, a table snapshot) | **Unbounded**: an endless sequence of events (a Kafka topic) |
| Runs | Periodically, start to finish | Continuously |
| Latency of results | Minutes to hours | Milliseconds to seconds |
| Correctness model | Simple: all input is present before computing | Hard: input arrives late, out of order, forever |
| Reprocessing | Rerun the job | Replay the log, or rebuild state |
| Typical tools | Spark, SQL warehouses (BigQuery, Snowflake), dbt, Hadoop MapReduce | Flink, Kafka Streams, Spark Structured Streaming, Beam/Dataflow |

**Micro-batch** sits between: process the stream in small batches every few seconds (Spark Structured Streaming's
default). It inherits batch simplicity at second-level latency, at the cost of a floor on latency.

The choice is a latency-versus-complexity trade-off. Batch is simpler, cheaper, and easier to get right, and it's
the right default whenever results can be hours old: monthly invoices, model training data, compliance reports.
Streaming earns its complexity when the value of the result decays quickly: fraud must be flagged before the order
ships; a surge in errors should page someone now; a dashboard of live orders is useless an hour late.

---

## 25.2 Batch Processing Done Right

The **MapReduce** model — still the conceptual core of Spark, warehouses, and most batch engines — has three steps:
**map** each input record to key-value pairs, **shuffle** so every pair with the same key reaches the same
worker, then **reduce** each key's values to a result.

```js
// The MapReduce shape: map each record to (key, value) pairs, group by key (the shuffle —
// in a cluster, every pair crosses the network to the reducer that owns its key), then reduce.
function mapReduce(records, map, reduce, { reducers = 3 } = {}) {
  const partitions = Array.from({ length: reducers }, () => new Map());
  const hash = (k) => [...String(k)].reduce((h, c) => (h * 31 + c.charCodeAt(0)) >>> 0, 7);
  for (const r of records) for (const [k, v] of map(r)) {           // map phase
    const p = partitions[hash(k) % reducers];                       // shuffle: route by key
    if (!p.has(k)) p.set(k, []);
    p.get(k).push(v);
  }
  const out = [];
  for (const p of partitions) for (const [k, vs] of p) out.push([k, reduce(k, vs)]); // reduce phase
  return out.sort(([a], [b]) => (a < b ? -1 : 1));
}

// Revenue per country per day from raw order events
const orders = [
  { country: 'PT', day: '2026-11-14', cents: 1200 }, { country: 'DE', day: '2026-11-14', cents: 800 },
  { country: 'PT', day: '2026-11-14', cents: 300 },  { country: 'PT', day: '2026-11-15', cents: 999 },
  { country: 'DE', day: '2026-11-15', cents: 4500 },
];
console.log(mapReduce(orders, (o) => [[`${o.day}/${o.country}`, o.cents]], (_, vs) => vs.reduce((a, b) => a + b, 0)));
// [
//   [ '2026-11-14/DE', 800 ],
//   [ '2026-11-14/PT', 1500 ],
//   [ '2026-11-15/DE', 4500 ],
//   [ '2026-11-15/PT', 999 ]
// ]
```

The **shuffle** is the expensive part: in a cluster, every intermediate pair crosses the network to the worker that
owns its key, and is usually written to disk on the way. Batch performance work is mostly shuffle work — reducing
how much data is shuffled (filter and project early, pre-aggregate on the map side with "combiners"), and avoiding
**skew**, where one hot key (one huge customer, a `null` key) sends most of the data to one reducer and makes the
whole job wait for it.

The properties that make batch jobs operable matter more than any engine choice:

- **Immutable, partitioned inputs.** Raw data lands once, partitioned by date (`events/date=2026-11-15/hour=10/`),
  as columnar files (Parquet — Chapter 10). Jobs read partitions; nothing edits raw data in place.
- **Deterministic and rerunnable.** Running a job twice on the same input produces the same output. No "now()" inside
  the logic (pass the run date in), no dependence on processing order, no side effects like sending emails.
- **Atomic outputs.** Write results to a staging location and publish them with a single atomic swap — replace a
  partition, rename a directory, switch a table pointer — so readers never see half a job's output and a failed job
  leaves the previous output intact.
- **Overwrite, don't append.** A job that rewrites "the output for 2026-11-15" can be rerun freely; a job that appends
  doubles its output on every retry.
- **Backfills are just reruns.** When logic changes, rerun the job for past dates. If that's hard, the job wasn't
  deterministic.

These are the same idempotency and atomicity ideas from Chapters 8 and 23, applied to whole datasets.

---

## 25.3 Stream Processing Fundamentals

A stream processor consumes events from a log, transforms them, and emits results to other logs, databases, or
caches. Its operations fall into two classes:

- **Stateless** operators process each event independently: parse, filter, enrich from a static lookup, route.
  They're easy — scale them by adding consumers, and replays are harmless if outputs are idempotent.
- **Stateful** operators remember things across events: counts per key, the last value of each user's profile, the
  contents of a window, pending join partners. They're where the difficulty lives, because state must survive
  crashes and must be consistent with the input position the processor has reached.

Stateful processors keep state **partitioned by key**, co-located with the input partition that feeds it (all of
user 42's events go to one partition, so user 42's state lives on that partition's worker — Chapter 24's keying).
State is kept in memory or an embedded store (often RocksDB) for speed, and made durable by **checkpointing**:

```text
input partition offsets  +  operator state  ──snapshot──▶  durable storage (object store)
        at offset 9,412          counts, windows               checkpoint #57

crash → restore checkpoint #57 (state as of offset 9,412) → resume reading from offset 9,412
```

Because the checkpoint captures state *and* input positions together, recovery rewinds both to a consistent point and
replays the input from there. Every event after the checkpoint is processed again — but it's applied to state that
didn't yet include it, so the state ends up exactly as if each event had been processed once. Flink's checkpoint
barriers and Kafka Streams' changelog topics are two implementations of this idea. The guarantee covers the
processor's **internal state**; outputs written to external systems need idempotent writes or transactional sinks
(Chapter 23's boundary rule again).

---

## 25.4 Time, Windows, and Watermarks

Aggregating an unbounded stream needs a boundary — "orders per minute", "errors in the last 5 minutes" — and the
first question is *which* time.

- **Event time** is when the event actually happened, recorded by the producer.
- **Processing time** is when the stream processor sees it.

They differ, often by a lot: a phone that was offline for an hour uploads its events late; a consumer that was down
for ten minutes processes a backlog; network delays reorder events. Aggregating by processing time is simple but
answers the wrong question — "how many orders did we *receive* this minute" — and gives different answers every time
the data is replayed. Aggregating by **event time** gives the right, reproducible answer, and forces you to deal with
lateness.

**Windows** group events by event time:

| Window | Shape | Example |
|---|---|---|
| **Tumbling** | Fixed size, non-overlapping | Orders per calendar minute |
| **Hopping / sliding** | Fixed size, overlapping, advancing by a step | Error rate over the last 5 minutes, updated every 30 seconds |
| **Session** | Per key, closed after a gap of inactivity | A user's browsing session, ending after 30 minutes idle |

The hard question is **when a window is complete**. With events arriving out of order, a processor can never be sure
it has seen every event for the 10:00–10:01 window. A **watermark** is the processor's declaration that it doesn't
expect any more events older than time *W*. When the watermark passes a window's end, the window is emitted. The
simplest watermark is "the latest event time seen, minus the maximum expected disorder" — if events are at most 10
seconds out of order, the watermark trails the newest event by 10 seconds.

Watermarks are a trade-off between **latency** and **completeness**: a watermark that trails by 10 seconds emits
results quickly but treats anything later as late; one that trails by 10 minutes is more complete and 10 minutes
slower. For the events that arrive after the watermark anyway, choose explicitly:

- **Allowed lateness** — keep a window's state for a while after emitting it, and emit **corrected results** if late
  events arrive. Downstream consumers must handle updates (upserts keyed by window), not just inserts.
- **Side output** — route events too late even for allowed lateness to a separate stream for reconciliation, rather
  than dropping them silently.
- **Batch correction** — let a later batch job recompute the authoritative numbers from the complete data (the lambda
  idea, §25.6).

The real-world example in §25.9 implements tumbling windows with all three behaviours.

> ⚠️ **Gotcha** — **One idle partition stops the watermark.** A processor reading many partitions takes the *minimum*
> of their watermarks, since any partition could still deliver old events. If one partition receives no events (a
> quiet tenant, a misconfigured producer), its watermark never advances, and every window across the whole job stays
> open forever — results stop, and state grows until memory runs out. Stream processors offer idleness detection that
> excludes silent partitions from the watermark after a timeout; turn it on.

---

## 25.5 Joins in Streams

Joining is as common in streaming as in SQL, and comes in three shapes:

**Stream–stream joins** match events from two streams within a time window: join `AdShown` with `AdClicked` on ad
impression ID within 30 minutes to compute click-through. The processor buffers each side's events for the window
duration, so state grows with event rate × window size — size windows deliberately.

**Stream–table joins (enrichment)** look up the current value of something for each event: attach the customer's
country and tier to each order. The tempting implementation — a database query per event — puts a synchronous network
call on every event and hammers the database. The streaming approach is to consume the table's **changelog** (via CDC,
Chapter 11) into a local, key-partitioned copy inside the processor, so each lookup is local. Co-partition the order
stream and the customer changelog by customer ID so each worker holds only the customers it needs.

**Table–table joins** maintain a join of two changing tables continuously, emitting updated results whenever either
side changes — effectively an incrementally maintained materialised view.

A subtle consequence of enrichment from a changelog: the order is joined with the customer's state **as of when the
processor saw the order**, not necessarily as of when the order happened. If the customer changed country a second
before ordering and the changelog event was delayed, the order is enriched with the old country. For most enrichment
that's fine; for things like tax jurisdiction, join on event time with versioned (temporal) tables or resolve it at
the source.

---

## 25.6 Lambda, Kappa, and Reprocessing

Logic changes — a bug is fixed, a definition of "active user" changes — and derived data must be recomputed for the
past. Two architectures answer how:

**Lambda architecture** runs two pipelines over the same data: a **batch layer** that periodically recomputes
authoritative results from all raw data, and a **speed layer** (stream processing) that provides approximate
low-latency results for the period the batch hasn't covered yet; queries merge them. It handles late data and
reprocessing well, but you maintain the same logic twice, in two frameworks, and the two disagree in subtle ways.

**Kappa architecture** uses one stream pipeline for everything. To reprocess, start a **new version** of the job
reading the retained log from the beginning (or from object-storage archives of it), let it catch up while the old
version keeps serving, then switch readers to the new output and retire the old job — the same alias-swap pattern
as Chapter 18's reindexing. It requires retaining the raw log long enough (often by tiering it to object storage) and
a stream processor that can replay fast.

| | Lambda | Kappa |
|---|---|---|
| Pipelines | Batch + streaming | Streaming only |
| Logic written | Twice | Once |
| Reprocessing | Rerun batch | Replay the log into a new job version |
| Late data | Batch layer corrects it | Allowed lateness, or replay |
| Operational cost | Two systems | One system, plus long log retention |
| Fits | Heavy analytical backfills, warehouses as source of truth | Event-driven products with a durable log |

In practice many organisations converge on a pragmatic blend: streams for low-latency operational outputs (alerts,
live counters, search and cache updates), and a **lakehouse or warehouse** — raw events landed continuously as
Parquet or table formats like Iceberg and Delta, queried and transformed with SQL — as the authoritative store for
analytics and backfills. The rule underneath is the one from Chapter 11: **keep the raw, immutable events, and treat
everything computed from them as rebuildable.**

---

## 25.7 Choosing an Approach

| Requirement | Approach |
|---|---|
| Daily/hourly reports, invoices, training data | Batch: SQL in the warehouse, or Spark; partitioned, rerunnable jobs |
| Dashboards that may be a few minutes stale | Micro-batch, or a warehouse with frequent incremental loads |
| Alerts, fraud scoring, live counters, cache/search updates | Stream processing with event time and checkpoints |
| Exactly-correct financial totals | Batch (or stream with careful idempotent sinks) over immutable ledger events — reconcile against the source |
| Ad-hoc analysis by humans | Warehouse/lakehouse fed by CDC and event pipelines |
| Simple per-event reactions (send an email) | A plain queue consumer (Chapter 24) — no stream processor needed |

The last row is a common over-engineering trap: a stream processing framework is for **stateful computation over
streams**. Consuming events and calling an API is a consumer, not a streaming job.

---

## 25.8 Common Mistakes and Best Practices

**Mistake: processing-time aggregation for business metrics.** Results depend on when data happened to arrive and
change every time the data is replayed.

**Mistake: silently dropping late events.** Totals become quietly wrong; route late events to a side output and
reconcile them.

**Mistake: idle partitions holding back the watermark.** Windows never close and state grows without bound; enable
idleness detection.

**Mistake: append-mode batch outputs.** Every retry duplicates data; overwrite whole partitions atomically.

**Mistake: non-deterministic batch jobs.** Jobs that read the current time or depend on ordering can't be rerun or
backfilled reliably.

**Mistake: a database call per event for enrichment.** It adds latency to every event and overloads the database;
consume the table's changelog into local state.

**Mistake: a streaming framework for stateless consumers.** A queue consumer is simpler, cheaper, and easier to
operate.

**Best practice: keep raw events immutable and partitioned.** Every derived dataset can then be rebuilt by rerunning
or replaying.

**Best practice: checkpoint state and input positions together.** That's what makes stateful recovery exact.

**Best practice: choose the watermark lag as a product decision.** It's the trade-off between how fast results appear
and how complete they are.

---

## 25.9 Real-World Example: An Event-Time Window Aggregator

This aggregator computes per-key totals over one-minute tumbling windows in event time. Its watermark trails the
newest event by the source's known disorder; windows emit when the watermark passes their end; late events within an
allowed-lateness period produce corrected results; events later than that go to a side output instead of disappearing;
and the whole state can be snapshotted for checkpointing.

```js
// windowed-aggregator.mjs — event-time tumbling windows with a watermark, allowed lateness,
// a side output for events that arrive too late, and snapshot/restore for checkpointing.

export class TumblingWindows {
  #windows = new Map();     // windowStart → { sums: Map(key → total), emitted: boolean }
  #maxEventTime = -Infinity;

  constructor({ sizeMs, maxOutOfOrderMs, allowedLatenessMs = 0, emit, late }) {
    Object.assign(this, { sizeMs, maxOutOfOrderMs, allowedLatenessMs, emit, late });
  }

  // "We believe no event older than this will still arrive": the latest event time seen,
  // minus how out of order the source is known to be
  get watermark() { return this.#maxEventTime - this.maxOutOfOrderMs; }

  add({ key, ts, value }) {
    const start = Math.floor(ts / this.sizeMs) * this.sizeMs;
    if (start + this.sizeMs + this.allowedLatenessMs <= this.watermark) {
      this.late({ key, ts, value });            // past its window's lateness: don't silently drop it
      return;
    }
    if (!this.#windows.has(start)) this.#windows.set(start, { sums: new Map(), emitted: false });
    const w = this.#windows.get(start);
    w.sums.set(key, (w.sums.get(key) ?? 0) + value);
    if (w.emitted) this.emit({ start, key, total: w.sums.get(key), update: true }); // late but allowed: correct it
    this.#maxEventTime = Math.max(this.#maxEventTime, ts);
    this.#advance();
  }

  #advance() {
    for (const [start, w] of [...this.#windows].sort(([a], [b]) => a - b)) {
      const end = start + this.sizeMs;
      if (!w.emitted && end <= this.watermark) {           // complete as far as we can tell: emit
        for (const [key, total] of w.sums) this.emit({ start, key, total, update: false });
        w.emitted = true;
      }
      if (end + this.allowedLatenessMs <= this.watermark) this.#windows.delete(start); // free state
    }
  }

  // Checkpointing: state plus the watermark's input, so a restarted job resumes exactly here
  snapshot() {
    return JSON.stringify({ maxEventTime: this.#maxEventTime,
      windows: [...this.#windows].map(([s, w]) => [s, w.emitted, [...w.sums]]) });
  }
  restore(json) {
    const { maxEventTime, windows } = JSON.parse(json);
    this.#maxEventTime = maxEventTime;
    this.#windows = new Map(windows.map(([s, emitted, sums]) => [s, { emitted, sums: new Map(sums) }]));
  }
}

// ── Demo: 1-minute windows; events up to 10 s out of order; 30 s allowed lateness ──────
const out = [];
const agg = new TumblingWindows({
  sizeMs: 60_000, maxOutOfOrderMs: 10_000, allowedLatenessMs: 30_000,
  emit: (r) => out.push(`${r.update ? 'UPDATE' : 'EMIT  '} [${r.start / 1000}s] ${r.key}=${r.total}`),
  late: (e) => out.push(`LATE   ${e.key} @${e.ts / 1000}s dropped to side output`),
});
const s = (sec) => sec * 1000;
[
  { key: 'PT', ts: s(5), value: 3 }, { key: 'PT', ts: s(50), value: 2 },
  { key: 'PT', ts: s(65), value: 1 },          // next window; watermark now 55 s
  { key: 'PT', ts: s(58), value: 4 },          // out of order, but window [0,60) still open
  { key: 'PT', ts: s(75), value: 1 },          // watermark 65 s → window [0,60) emitted
  { key: 'PT', ts: s(40), value: 10 },         // late, within allowed lateness → UPDATE
  { key: 'PT', ts: s(130), value: 1 },         // watermark 120 s → [60,120) emitted; [0,60) state freed
  { key: 'PT', ts: s(20), value: 99 },         // far too late → side output
].forEach((e) => agg.add(e));
console.log(out.join('\n'));
// EMIT   [0s] PT=9
// UPDATE [0s] PT=19
// EMIT   [60s] PT=2
// LATE   PT @20s dropped to side output

const restored = new TumblingWindows({ sizeMs: 60_000, maxOutOfOrderMs: 10_000, emit: () => {}, late: () => {} });
restored.restore(agg.snapshot());
console.log('restored watermark:', restored.watermark / 1000, 's'); // restored watermark: 120 s
```

The out-of-order event at 58 s still lands in its window, because the watermark (55 s) hadn't passed the window's end;
the event at 40 s arrives after the window was emitted but within the lateness period, so it produces an `UPDATE`; and
the event at 20 s, two windows too late, goes to the side output.

The decision to defend is **emitting a window's result when the watermark passes its end, and then emitting corrections,
rather than waiting long enough to be sure the result is final**. Waiting is tempting: downstream consumers would only
ever see one, final number per window. But "long enough to be sure" doesn't exist for real sources — a phone can upload
events days late — so any fixed wait is still a guess, and a long one makes every result late for the sake of a few
stragglers. Emitting on the watermark gives most consumers a timely, nearly complete answer; allowed lateness corrects
it for the stragglers that arrive soon after; and the side output catches the rest for reconciliation. The cost is that
consumers must treat window results as upserts keyed by window, not as immutable facts — a small, explicit price for
results that are both prompt and eventually right.

---

## 25.10 Questions

**1.** MCQ — Which is unbounded data? a) Yesterday's log files b) A table snapshot c) The `orders` topic in Kafka, still receiving events d) A CSV export
<details><summary>Answer</summary>**c)** It never ends; the others are fixed datasets.</details>

**2.** MCQ — What is usually the most expensive phase of a distributed batch job? a) Reading input b) The shuffle c) Writing output d) Scheduling
<details><summary>Answer</summary>**b)** Every intermediate pair crosses the network (and often disk) to the worker owning its key.</details>

**3.** MCQ — Why aggregate by event time rather than processing time? a) It's faster b) Results reflect when things happened and are reproducible on replay c) It needs no state d) It avoids watermarks
<details><summary>Answer</summary>**b)** Processing-time results depend on arrival timing and change on replay.</details>

**4.** MCQ — What does a watermark at time W assert? a) All events have been processed b) No events with event time earlier than W are expected anymore c) The processor's clock reads W d) W events have arrived
<details><summary>Answer</summary>**b)** It's the processor's estimate of input completeness, used to decide when windows can be emitted.</details>

**5.** MCQ — Which window type groups a user's events until 30 minutes of inactivity? a) Tumbling b) Hopping c) Session d) Global
<details><summary>Answer</summary>**c)** Session windows are per key and close after a gap.</details>

**6.** MCQ — What's the main cost of the lambda architecture? a) No low-latency results b) The same logic maintained in two pipelines that can disagree c) No reprocessing d) It requires no storage
<details><summary>Answer</summary>**b)** Batch and speed layers duplicate logic in different frameworks.</details>

**7.** Short — Why should batch jobs overwrite output partitions rather than append?
<details><summary>Answer</summary>Retries and backfills rerun jobs; overwriting a partition makes reruns idempotent, whereas appending duplicates the output each time.</details>

**8.** Short — How does checkpointing give a stateful stream processor correct state after a crash?
<details><summary>Answer</summary>A checkpoint captures operator state and input offsets at the same consistent point. Recovery restores both and replays input from those offsets, so every event after the checkpoint is applied exactly once to state that didn't yet include it.</details>

**9.** Short — Why does one idle partition stall a whole stream job?
<details><summary>Answer</summary>The job's watermark is the minimum across input partitions; a partition with no events never advances its watermark, so windows never close. Idleness detection excludes silent partitions after a timeout.</details>

**10.** Short — Why consume a table's changelog into local state for enrichment rather than querying the database per event?
<details><summary>Answer</summary>Per-event queries add a network round trip to every event and load the database proportionally to event rate; a co-partitioned local copy makes lookups local and scales with the stream.</details>

**11.** Short — What does kappa architecture do to reprocess data after a logic change?
<details><summary>Answer</summary>It runs a new version of the streaming job from the start of the retained log, lets it catch up alongside the old one, then switches readers to the new output and retires the old job.</details>

**12.** Output — Using §25.2's `mapReduce`, what does this return?
```js
mapReduce(['a b', 'b c b'], (line) => line.split(' ').map((w) => [w, 1]), (_, vs) => vs.length);
```
<details><summary>Answer</summary>`[ [ 'a', 1 ], [ 'b', 3 ], [ 'c', 1 ] ]` — a word count; outputs are sorted by key.</details>

**13.** Output — Using §25.9's `TumblingWindows` (60 s windows, 10 s max disorder, no allowed lateness), events arrive with timestamps 10 s, 70 s, then 55 s. What happens to the 55 s event?
<details><summary>Answer</summary>**It goes to the side output as late.** After the 70 s event the watermark is 60 s, so window [0, 60) is emitted and — with zero allowed lateness — its state is freed at the same moment. The 55 s event then satisfies 0 + 60 + 0 ≤ 60 and is routed to `late`. With any allowed lateness it would have produced an `UPDATE` instead.</details>

**14.** Output — A stream–stream join buffers both sides for a 30-minute window. Each side receives 2,000 events/s of 500 bytes. Roughly how much join state is held?
<details><summary>Answer</summary>**~3.6 GB.** Each side holds 2,000 × 1,800 s = 3.6 M events × 500 B = 1.8 GB; two sides ≈ 3.6 GB, plus overhead — partitioned across workers.</details>

**15.** Output — With a watermark trailing the newest event by 2 minutes, at what processing moment (relative to the newest event time) is the window [10:00, 10:05) emitted?
<details><summary>Answer</summary>**When an event with time ≥ 10:07 arrives.** The watermark must reach 10:05, i.e. newest event time − 2 min ≥ 10:05.</details>

**16.** Debug — A real-time dashboard's per-minute order counts change every time the team replays the topic after an incident, and differ from the warehouse's daily totals. What's wrong?
<details><summary>Answer</summary>The job aggregates by processing time, so counts depend on when events were read, and replays shift them between minutes. Aggregate by event time with watermarks, and handle late data with allowed lateness or reconciliation.</details>

**17.** Debug — A Flink job's output stopped entirely overnight, and memory grew until the job was killed. One of its 48 input partitions belongs to a tenant that went quiet. What happened and what's the fix?
<details><summary>Answer</summary>The idle partition's watermark never advanced, holding the job's watermark back, so no windows closed and state accumulated. Configure idleness detection on the source so silent partitions are excluded from watermark calculation after a timeout.</details>

**18.** Debug — A nightly revenue job was retried after a cluster hiccup, and the finance dashboard showed double revenue for that day. Why, and how should the job be written?
<details><summary>Answer</summary>The job appended its results, so the retry added them again. Write to a staging location and atomically overwrite the day's partition, so reruns replace rather than add.</details>

**19.** Debug — An order-enrichment stream job adds 40 ms of latency per event and its database's CPU rises with order volume. It queries the customers table per event. What do you change?
<details><summary>Answer</summary>Feed the customers table's CDC changelog into the job as a co-partitioned local table (keyed by customer ID, matching the order stream's key) and join locally, removing the per-event query.</details>

**20.** Code — Write `hoppingWindows(ts, sizeMs, stepMs)` returning the start times of every hopping window containing timestamp `ts`.
<details><summary>Answer</summary>

```js
function hoppingWindows(ts, sizeMs, stepMs) {
  const starts = [];
  const latest = Math.floor(ts / stepMs) * stepMs;          // the most recent window to start at or before ts
  for (let s = latest; s > ts - sizeMs && s >= 0; s -= stepMs) starts.push(s); // time starts at 0
  return starts.reverse();
}
console.log(hoppingWindows(130_000, 300_000, 60_000)); // [ 0, 60000, 120000 ]
```
A 5-minute window advancing every minute contains each event in up to five windows; this one is near the start of time, so only three exist.</details>

**21.** Code — Write `sessionize(events, gapMs)` that groups a single user's events (sorted by `ts`) into sessions separated by gaps longer than `gapMs`.
<details><summary>Answer</summary>

```js
function sessionize(events, gapMs) {
  const sessions = [];
  for (const e of events) {
    const current = sessions.at(-1);
    if (current && e.ts - current.end <= gapMs) { current.end = e.ts; current.count++; }
    else sessions.push({ start: e.ts, end: e.ts, count: 1 });
  }
  return sessions;
}
console.log(sessionize([{ ts: 0 }, { ts: 60_000 }, { ts: 3_000_000 }], 30 * 60_000));
// [ { start: 0, end: 60000, count: 2 }, { start: 3000000, end: 3000000, count: 1 } ]
```
In a stream, out-of-order events can merge two sessions into one, which is why session windows need watermarks too.</details>

**22.** Code — Write `publishPartition(storage, { table, partition, rows })` that writes to a staging path and then atomically swaps it in, so readers never see partial output.
<details><summary>Answer</summary>

```js
async function publishPartition(storage, { table, partition, rows }) {
  const staging = `${table}/_staging/${partition}-${crypto.randomUUID()}`;
  await storage.write(staging, rows);                        // a failure here leaves the live data untouched
  await storage.atomicReplace(`${table}/${partition}`, staging); // e.g. a table-format commit or metastore pointer swap
}
```
On object storage, "atomic replace" is provided by a table format (Iceberg, Delta) or a metastore pointer, since object renames aren't atomic across many files.</details>

**23.** Scenario — A fraud team wants transactions scored within 200 ms using features like "number of transactions on this card in the last 10 minutes" and "average amount over 30 days". Design the computation.
<details><summary>Answer</summary>Compute short-window features (10-minute counts per card) in a stream processor keyed by card ID, with event-time sliding windows and state in an embedded store, writing current feature values to a low-latency feature store. Compute long-window features (30-day averages) in a daily or hourly batch job and load them into the same store. The scoring service reads both by card ID at request time. Reconcile stream features against batch recomputations to catch drift, and use idleness detection so quiet partitions don't stall windows.</details>

**24.** Scenario — A team built a lambda architecture for usage-based billing and the batch and streaming numbers disagree by 0.3%, generating support tickets. What do you recommend?
<details><summary>Answer</summary>Decide which number is authoritative: for billing, the batch computation over immutable usage events after a cut-off (allowing for late data) should be the invoice of record, with the stream providing an explicitly labelled "estimate so far". Then reduce disagreement by sharing the logic — one definition (e.g. SQL in a framework that runs on both batch and streams, or a kappa design with one job), and investigate the 0.3% (usually late events, processing-time windows, or non-idempotent sinks).</details>

---

## Chapter Summary

- Batch processes bounded data simply and cheaply; stream processing handles unbounded data with low latency and
  more complexity — choose by how quickly the result loses value.
- Batch jobs should read immutable, partitioned inputs, be deterministic, and publish outputs atomically by
  overwriting partitions, so reruns and backfills are safe; the shuffle and skew dominate their cost.
- Stateful stream processing keeps key-partitioned state and checkpoints it together with input offsets, so recovery
  replays to exactly consistent state; external sinks still need idempotency.
- Aggregate by event time; windows emit when the watermark passes them, allowed lateness produces corrections, and
  side outputs keep too-late events from vanishing — and idle partitions must not hold the watermark back.
- Enrich streams from changelogs held as local, co-partitioned tables, not per-event database queries.
- Lambda duplicates logic for easy reprocessing; kappa replays one pipeline; in every case keep raw events immutable
  and treat everything derived as rebuildable.

---

## Practice Exercises

1. Write a Spark (or DuckDB) job that computes daily revenue per country from partitioned Parquet files, and make it
   idempotent by overwriting the output partition; rerun it three times and verify identical output.
2. Extend §25.9's aggregator with sliding windows and a per-partition watermark that takes the minimum across
   partitions, then reproduce the idle-partition stall and fix it with an idleness timeout.
3. Read the Flink documentation on checkpoints and exactly-once sinks, and explain which guarantees end at a JDBC
   sink and which continue through a Kafka transactional sink.
4. Generate an event stream with a realistic lateness distribution, and chart result completeness versus emission
   delay for watermark lags of 5 s, 1 min, and 10 min.
5. Build a stream–table join that enriches orders with customer data from a CDC changelog, and measure the
   enrichment error rate when customer updates are delayed relative to orders.

---

## Mini Project: A Live Metrics Pipeline With Batch Reconciliation

Build a pipeline that ingests synthetic e-commerce events into a log, computes live per-minute metrics with a stream
processor you write, lands raw events as partitioned files, and reconciles the live numbers against a nightly batch.

- Producers that emit events with realistic delays, reordering, and occasional days-late uploads.
- A stream job with event-time tumbling and sliding windows, watermarks with idleness handling, allowed lateness,
  side outputs, and checkpointing to disk with exact recovery after a kill.
- A sink that upserts window results keyed by window and key, idempotent under replay.
- A raw-event lander writing hourly Parquet (or JSON Lines) partitions, and a deterministic batch job recomputing the
  same metrics.
- A reconciliation report comparing stream and batch results per window, attributing differences to late events.

---

**Next:** [Chapter 26 — Event-Driven Architecture →](26-event-driven-architecture.md)
