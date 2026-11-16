# Chapter 48 — Design a Metrics and Monitoring System

## Learning Objectives

- Scope a multi-tenant metrics platform and size it by its true unit of cost: active time series
- Design the ingest path — collection, distribution by series, replication, and a write-ahead in-memory head
- Store samples compactly with delta-of-delta timestamps and XOR-compressed values, in immutable blocks on object storage
- Index series with an inverted index, and control cardinality and churn before they take the platform down
- Serve queries and evaluate alerts reliably, including monitoring the monitoring system itself

---

## 48.1 Requirements and Scope

Every system in this book has assumed a place where its metrics go: request rates, error ratios, latency histograms, queue depths, burn-rate
alerts (Chapter 30). This chapter designs that place — a metrics platform that ingests numeric time series from every service in an
organisation, stores them for months, answers dashboard and ad-hoc queries in under a second, and evaluates the alerts that wake people up.
It has one requirement no other system has: **it must work best when everything else is broken**, because that's when people look at it.

**Functional requirements**

- Ingest samples — `(series labels, timestamp, value)` — from many thousands of targets, by scraping (pull) or receiving writes (push).
- Query by label selectors and functions over time: rates of counters, percentiles from histograms, aggregations across series.
- Evaluate recording and alerting rules continuously; route, group, and deduplicate notifications.
- Retain raw data for weeks and downsampled data for over a year; enforce per-tenant limits.

**Out of scope**: logs and traces (Chapter 30 — related pipelines, different storage), dashboards' UI, anomaly detection beyond rules.

**Non-functional requirements**

| Requirement | Target |
|---|---|
| Scale | ~20 million active series; ~1.3 million samples per second |
| Ingest | Accept writes during partial failures; a sample acknowledged is durable |
| Query latency | Dashboard queries over the last few hours in p99 < 1 s; long-range queries in seconds |
| Alerting | Rules evaluated every 15–60 s; alerts fire within a minute or two of the condition |
| Availability | Higher than any service it monitors, and independent of them |
| Retention | Raw 15-second data for 15 days; 5-minute aggregates for 13 months |

Metrics tolerate things other data doesn't: a few lost samples barely change a rate, and a value a few seconds stale is fine. They don't
tolerate an unavailable platform during an incident, or one that's been taken down by a single team's careless label.

---

## 48.2 Estimation

```js
// Metrics platform: active series drive everything — ingest rate, memory, and storage
const targets = 20_000, seriesPerTarget = 1_000, scrapeSec = 15, bytesPerSample = 1.5;
const series = targets * seriesPerTarget;
const samplesPerSec = series / scrapeSec;
const perDayGB = (samplesPerSec * 86_400 * bytesPerSample) / 1e9;
console.log(`${(series / 1e6).toFixed(0)} M active series | ~${(samplesPerSec / 1e6).toFixed(2)} M samples/s | ~${perDayGB.toFixed(0)} GB/day compressed (vs ~${(perDayGB * 16 / bytesPerSample / 1e3).toFixed(1)} TB raw)`);
const headBytesPerSeries = 4_000;                                   // labels, index entries, open chunk, buffers
console.log(`in-memory head: ~${(series * headBytesPerSeries / 1e9).toFixed(0)} GB across the ingest tier; 13 months at 15 s ≈ ${(perDayGB * 395 / 1e3).toFixed(0)} TB, at 5 min ≈ ${(perDayGB * 395 / 20 / 1e3).toFixed(1)} TB`);
// 20 M active series | ~1.33 M samples/s | ~173 GB/day compressed (vs ~1.8 TB raw)
// in-memory head: ~80 GB across the ingest tier; 13 months at 15 s ≈ 68 TB, at 5 min ≈ 3.4 TB
```

What the numbers decide:

- **Active series, not samples, are the cost driver.** Every series carries fixed memory (labels, index entries, an open chunk) regardless of how
  few samples it has; ~4 KB per series puts ~80 GB in the ingest tier's memory, before replication. A team that doubles its series doubles its share
  of that — which is why cardinality is a platform-level concern (§48.6).
- **Compression changes the storage problem by an order of magnitude.** 16 raw bytes per sample (8-byte timestamp, 8-byte float) becomes ~1.5 with
  Gorilla-style encoding (§48.10): 173 GB a day instead of 1.8 TB.
- **Long retention needs downsampling.** Thirteen months at full resolution is ~68 TB; 5-minute aggregates are 20× fewer points. Keeping several
  aggregates per point (min, max, sum, count) makes it ~14 TB — still far less, and long-range queries read 20× fewer samples.
- **1.3 million samples per second** is a write-heavy workload — an LSM-shaped problem (Chapter 10) where data arrives roughly in time order and is
  almost never updated.

---

## 48.3 API and Data Model

A **series** is a metric name plus a set of label pairs; a **sample** is a timestamp and a float value in that series.

```text
http_requests_total{service="api", route="/orders/:id", status="500", pod="api-7f9c"}   (1793000015, 182734)
```

```text
POST /api/v1/write            (push) batches of { labels, samples[] }, compressed; tenant from auth
GET  /metrics                 (pull) each target exposes its current values; the platform scrapes every 15 s
GET  /api/v1/query            ?query=sum by (status) (rate(http_requests_total{service="api"}[5m]))&time=…
GET  /api/v1/query_range      ?query=…&start=…&end=…&step=60s
GET  /api/v1/series           ?match[]={service="api"}            (label exploration, cardinality debugging)
```

| Data | Store | Notes |
|---|---|---|
| **Head** (last ~2 h) | Ingester memory + write-ahead log on local disk | Open chunks per series, appended in time order |
| **Blocks** (older) | Object storage (Chapter 17), immutable, ~2 h → compacted to 24 h+ | Compressed chunks + the block's series index |
| **Downsampled blocks** | Object storage | 5-minute and 1-hour aggregates per series |
| **Rules and alert state** | Configuration store + rule evaluators' memory | Which rules, which alerts are pending or firing |
| **Tenant limits** | Configuration store | Max active series, samples per second, labels per series, query limits |

**Pull or push?** Pulling (the platform scrapes targets) gives the platform control of the rate, makes "target is down" a first-class signal (the scrape
fails), and needs service discovery to find targets. Pushing suits short-lived jobs, clients behind NAT, and agents that batch and forward. Most large
platforms do both: local agents scrape nearby targets and push batches to the central platform — combining pull's health signal with push's simple
network path.

---

## 48.4 High-Level Design

```text
targets ◀─scrape─ agents (per cluster) ──remote write──▶ distributors (validate, apply limits)
                                                               │ hash(tenant, series labels) → 3 ingesters
                                                               ▼
                                                  ingesters: WAL + in-memory head (2 h)
                                                               │ every 2 h: cut a block
                                                               ▼
                                           object storage: immutable blocks ◀── compactor (merge, downsample, retention)
                                                               ▲
query frontend (split by time, cache results) ─▶ queriers ─────┤ recent data: ask the ingesters
                                                               │ older data: read blocks (via store gateways + caches)
rule evaluators (recording + alerting rules) ─▶ queriers       │
        └─▶ alert manager (group, deduplicate, silence, route) ─▶ pager / chat / tickets
```

- **Distributors** are stateless: they authenticate the tenant, validate labels, enforce limits, and route each series to **three ingesters** chosen by
  consistent hashing on its labels (Chapter 14). A write succeeds when two of three acknowledge — a write quorum (Chapter 13).
- **Ingesters** append samples to per-series chunks in memory and a **write-ahead log** for crash recovery; every two hours they cut an immutable block and
  upload it. Because all samples of a series go to the same three ingesters, each ingester's head is a complete picture of its series.
- **The compactor** merges small blocks into larger ones (and deduplicates the three replicas' copies), builds downsampled blocks, and deletes data past
  retention.
- **Queriers** fan a query out to ingesters (for the last few hours) and to block storage (for older data), merging and deduplicating replicas. A **query
  frontend** splits long ranges into per-day pieces, runs them in parallel, and caches results for days that can't change any more.
- **Rule evaluators** run recording rules (precomputed aggregates) and alerting rules on a schedule; the **alert manager** turns alert states into
  notifications.

---

## 48.5 Deep Dive: The Storage Engine

**The head.** Samples for a series arrive roughly in time order, one every scrape interval. The ingester keeps an **open chunk** per series in memory —
typically ~120 samples (30 minutes at 15 s) — compressed as it's appended (§48.10). Full chunks are sealed and kept memory-mapped until the block is cut.
Every sample is also written to the **WAL** before it's acknowledged, so a crashed ingester replays the log and rebuilds its head — and since every series
has three ingesters, the other two cover the gap while it does.

**Blocks.** Every two hours, the head's oldest two hours become an **immutable block**: chunks for every series, plus that block's series index, written
as files and uploaded to object storage. Immutability makes everything downstream simple: blocks are cached anywhere, replicated by copying, and
compacted by writing new ones and deleting old ones — Chapter 10's SSTables, with time as the primary sort key.

**Compaction and deduplication.** Three ingesters uploaded three copies of each series' samples. The compactor merges blocks for the same time range,
deduplicates replica samples, and combines two-hour blocks into day-sized ones, so long-range queries open far fewer files and indexes.

**Downsampling and retention.** For older data, the compactor writes blocks with one point per 5 minutes (and later per hour), keeping **min, max, sum,
and count** per window rather than a single average, so queries can still compute correct averages, rates, and extremes at the coarser resolution.
Retention deletes whole blocks, which is cheap — no tombstones, no row deletes — because time is how the data is partitioned.

**Out-of-order and late samples.** A strictly append-only head rejects samples older than the series' last one. Agents that buffer during network
problems send late data; modern TSDBs accept a bounded out-of-order window (e.g., an hour) into a separate structure merged at compaction, rather than
dropping it.

> ⚠️ **Gotcha** — Averages of averages are wrong, and so are downsampled averages without counts. If one 5-minute window has 1 request averaging 2 s and
> the next has 1,000 requests averaging 100 ms, the hourly "average of the averages" is ~1 s; the true mean is ~102 ms. Store sum and count per window
> and divide at query time — and for percentiles, keep histogram buckets (Chapter 30), never precomputed p99s.

---

## 48.6 Deep Dive: Cardinality and the Series Index

Chapter 30 introduced cardinality from the instrumenting side: label combinations multiply. Here's what that does to the platform.

**The index.** A query selects series by labels — `{service="api", status=~"5.."}` — so the platform keeps an **inverted index** (Chapter 18's structure,
with label pairs as terms): each `label=value` maps to a sorted **postings list** of series IDs, and a selector is an intersection of postings lists
(unions for regular expressions):

```js
// The series index: every distinct label set is a series with an ID, and an inverted index maps each
// label=value pair to the sorted IDs of series that carry it. A selector is an intersection of postings.
class SeriesIndex {
  #ids = new Map();               // canonical label string → series ID
  #postings = new Map();          // 'name=value' → sorted array of series IDs
  getOrCreate(labels) {
    const keyStr = Object.keys(labels).sort().map(k => `${k}=${labels[k]}`).join(',');
    let id = this.#ids.get(keyStr);
    if (id !== undefined) return id;
    id = this.#ids.size;
    this.#ids.set(keyStr, id);
    for (const [k, v] of Object.entries(labels)) {
      const p = `${k}=${v}`;
      if (!this.#postings.has(p)) this.#postings.set(p, []);
      this.#postings.get(p).push(id);                          // IDs are assigned in order, so lists stay sorted
    }
    return id;
  }
  select(matchers) {              // { label: exactValue | RegExp }
    const lists = Object.entries(matchers).map(([k, m]) => {
      if (typeof m === 'string') return this.#postings.get(`${k}=${m}`) ?? [];
      const ids = [];             // a regex matcher unions the postings of every matching value
      for (const [p, list] of this.#postings) {
        const [pk, pv] = [p.slice(0, p.indexOf('=')), p.slice(p.indexOf('=') + 1)];
        if (pk === k && m.test(pv)) ids.push(...list);
      }
      return [...new Set(ids)].sort((a, b) => a - b);
    });
    lists.sort((a, b) => a.length - b.length);                 // intersect starting from the smallest list
    return lists.reduce((acc, list) => { const s = new Set(list); return acc.filter(id => s.has(id)); });
  }
  get size() { return this.#ids.size; }
}

const idx = new SeriesIndex();
const services = ['api', 'auth', 'billing', 'search'], statuses = ['200', '201', '400', '404', '500', '503'];
const routes = Array.from({ length: 20 }, (_, i) => `/route/${i}`), pods = Array.from({ length: 30 }, (_, i) => `pod-${i}`);
for (const service of services) for (const status of statuses) for (const route of routes) for (const pod of pods) {
  idx.getOrCreate({ __name__: 'http_requests_total', service, status, route, pod });
}
console.log('series for one metric:', idx.size.toLocaleString('en-US'), '=', [services, statuses, routes, pods].map(a => a.length).join(' × '));
console.log('api 5xx series:', idx.select({ __name__: 'http_requests_total', service: 'api', status: /^5..$/ }).length);
// One careless label — a user ID with 10,000 values — multiplies everything
console.log(`adding user_id with 10,000 values: ${(idx.size * 10_000).toLocaleString('en-US')} series`);
// series for one metric: 14,400 = 4 × 6 × 20 × 30
// api 5xx series: 1200
// adding user_id with 10,000 values: 144,000,000 series
```

(Real series counts are lower than the full cross product — not every route returns every status — but the cross product is the ceiling a label
change can reach.) The query `{service="api", status=~"5.."}` touches 1,200 series, and each one is a chunk to decompress per time range. Cardinality
therefore costs three times: **memory** in the ingesters' heads and indexes, **index size** in every block, and **query time** proportional to series
touched.

**Churn** is cardinality over time. A `pod` label is bounded at any moment, but every deploy replaces every pod with new names — new series, while the
old ones linger in the head until the block is cut. A service with 30 pods that deploys 20 times a day creates 600 pod values a day; with a frequent
autoscaler, thousands. Churn inflates the head and every day's index even though "active" series look stable.

**Controlling it** has to happen at the platform, because one team's mistake affects every tenant sharing an ingester:

- **Per-tenant limits** at the distributor: maximum active series, series per metric, labels per series, and label value length — rejecting new series
  (not existing ones) when exceeded, with a clear error and a metric the tenant can alert on.
- **Cardinality visibility**: per-tenant, per-metric series counts and top label values, so teams can find the culprit themselves.
- **Relabelling at the agent**: drop or rewrite labels (strip `pod` from metrics aggregated by service, map raw paths to route templates) before data
  leaves the cluster.
- **Recording rules**: precompute the aggregates dashboards actually use (`sum by (service, status)`), so queries read a few series instead of thousands.

> 💼 **Interview** — The metrics-system question rewards knowing what the cost unit is. Say early that active series — not requests or samples — drive
> memory, index size, and query cost; describe hashing series to replicated ingesters with a WAL and two-hour immutable blocks on object storage; and name
> per-tenant series limits as the defence against one team's label taking down everyone's monitoring.

---

## 48.7 Deep Dive: Querying and Alerting

**Query execution.** A query like `sum by (status) (rate(http_requests_total{service="api"}[5m]))` resolves its selector through the index, reads each
series' chunks for the range (from ingesters for recent data, from blocks for older), computes the per-series rate — handling **counter resets**, where a
restarted process starts from zero (Question 20) — and aggregates. The expensive part is the number of series and the range, so the platform protects
itself with **query limits** (series touched, samples read, time range), and the query frontend splits ranges by day, runs them in parallel, and caches
the immutable past.

**Recording rules** precompute expensive expressions every interval and store the result as a new, small series. A dashboard that sums 10,000 series
on every refresh, for 50 viewers, becomes one that reads a few precomputed series.

**Alert evaluation.** Rule evaluators run each alerting rule every interval and track a state per alert instance: **inactive → pending → firing**. A rule with
`for: 5m` must be true for five consecutive minutes before it fires, which filters blips (Chapter 30's burn-rate windows serve the same purpose). Rules are
sharded across evaluators by hash, and each runs with **two replicas** so a crashed evaluator doesn't silence alerts; duplicates are expected.

**The alert manager** receives alert states from all evaluator replicas and does the human-facing work:

- **Deduplicate** the same alert from replicas and repeated evaluations.
- **Group** related alerts (all `HighErrorRate` alerts for one service, across 30 pods) into one notification.
- **Inhibit** dependent alerts (don't page for every service's errors when the alert for their shared database is already firing).
- **Silence** during planned maintenance, and **route** by labels to the owning team's channel and on-call rotation.

**Watching the watcher.** If the platform is down, its alerts don't fire — and nobody notices, because silence looks like health. Two defences:

- A **dead man's switch**: an alert that *always* fires, sent to an external service that pages if it ever *stops* arriving.
- **Independent meta-monitoring**: a small, separate monitoring stack (different infrastructure, ideally a different region or provider) that watches the
  main platform's ingest rate, query latency, and rule evaluation lag.

---

## 48.8 Failure Modes, Bottlenecks, and 10×

| Failure | Effect | Mitigation |
|---|---|---|
| Ingester crashes | Its in-memory head is lost | WAL replay on restart; two other replicas keep serving its series; write quorum 2 of 3 |
| Cardinality explosion from one tenant | Ingester memory exhausted, everyone affected | Per-tenant series limits at the distributor; reject new series, keep existing ones |
| Expensive query | Queriers overloaded, dashboards slow for all | Query limits on series and samples, per-tenant query concurrency, frontend splitting and caching |
| Agents can't reach the platform | Gaps in data | Agent-side buffering on disk with bounded retention; out-of-order window for late samples |
| Object storage slow | Long-range queries slow | Block, index, and chunk caches in front of storage; recent queries served from ingesters |
| Platform down during an incident | Blind exactly when it matters | Dead man's switch, independent meta-monitoring, platform deployed on separate infrastructure from what it watches |
| Rule evaluator lag | Alerts fire late | Shard rules, alert on evaluation lag, keep alert rules cheap (use recording rules) |

**Deployment independence.** The metrics platform must not depend on the systems it monitors: not on the same Kubernetes cluster, the same service mesh, the
same DNS, or the same identity provider — or a failure in any of them blinds you to that failure.

**At 10×** — 200 million active series: ingesters scale horizontally because series hash evenly; the index becomes the hardest part (sharded blocks,
index caches, and limits that matter more); per-tenant isolation moves from shared ingesters to dedicated shards for the largest tenants; and recording
rules and downsampling stop being optimisations and become the only way long-range dashboards stay interactive. The design — series-hashed ingest,
immutable blocks, an inverted index, limits at the front door — doesn't change.

---

## 48.9 Common Mistakes and Best Practices

**Mistake: treating samples as the cost.** Active series drive memory and index size; a few series sampled often are cheap, millions sampled rarely
are not.

**Mistake: no per-tenant limits.** One team's `user_id` label takes down monitoring for the whole company.

**Mistake: averaging averages, or storing precomputed percentiles.** Downsample to sum and count; keep histograms for latency.

**Mistake: running the monitoring platform on the infrastructure it monitors.** Its outage coincides with the outage you need to see.

**Mistake: no alert on missing data.** A target that stopped reporting looks like a quiet one. Alert on absent series and failed scrapes.

**Mistake: dashboards that aggregate thousands of raw series on every refresh.** Use recording rules.

**Mistake: rejecting all late samples.** Agents buffering through a network blip lose exactly the data from the incident.

**Best practice: hash series to replicated ingesters with a WAL.** Durable, horizontally scalable ingest with no shared write bottleneck.

**Best practice: immutable blocks on object storage, compacted and downsampled.** Cheap retention, trivially cacheable, deletion by dropping blocks.

**Best practice: a dead man's switch and independent meta-monitoring.** The monitoring system's silence must itself be detectable.

---

## 48.10 Real-World Example: Gorilla-Style Chunk Compression

This is the encoding at the heart of the storage engine, in the style of Facebook's Gorilla paper and the many TSDBs that adopted it. Timestamps are
stored as the **difference between successive deltas**: regular scrapes have constant deltas, so the delta-of-delta is zero — a single bit. Values are
XORed with the previous value: an unchanged value is a single bit, and a slowly changing one stores only the bits between its leading and trailing zeros,
often reusing the previous sample's bit window.

```js
// gorilla.mjs — time-series chunk compression in the style of Facebook's Gorilla (used, with variations,
// by most modern TSDBs): timestamps as delta-of-delta, values as XOR against the previous value.
// Regular scrapes and slowly changing values compress to a bit or two per sample instead of 16 bytes.

class BitWriter {
  #bytes = []; #cur = 0; #n = 0;                                 // #n = bits filled in #cur
  write(value, bits) {                                           // value: BigInt, most significant bit first
    for (let i = bits - 1; i >= 0; i--) {
      this.#cur = (this.#cur << 1) | Number((value >> BigInt(i)) & 1n);
      if (++this.#n === 8) { this.#bytes.push(this.#cur); this.#cur = 0; this.#n = 0; }
    }
  }
  bytes() { return Uint8Array.from(this.#n ? [...this.#bytes, this.#cur << (8 - this.#n)] : this.#bytes); }
}
class BitReader {
  #b; #pos = 0;
  constructor(bytes) { this.#b = bytes; }
  read(bits) {
    let v = 0n;
    for (let i = 0; i < bits; i++, this.#pos++) v = (v << 1n) | BigInt((this.#b[this.#pos >> 3] >> (7 - (this.#pos & 7))) & 1);
    return v;
  }
}

const f64 = new DataView(new ArrayBuffer(8));
const toBits = (x) => (f64.setFloat64(0, x), f64.getBigUint64(0));
const fromBits = (b) => (f64.setBigUint64(0, b), f64.getFloat64(0));
const clz64 = (x) => { let n = 0; for (let i = 63n; i >= 0n && !((x >> i) & 1n); i--) n++; return n; };
const ctz64 = (x) => { let n = 0; while (n < 64 && !((x >> BigInt(n)) & 1n)) n++; return n; };
const signed = (v, bits) => BigInt.asIntN(bits, v);              // reinterpret a bit field as two's complement

export function encode(samples) {                                // samples: [[tsSeconds, value], ...]
  const w = new BitWriter();
  const [t0, v0] = samples[0];
  w.write(BigInt(t0), 64); w.write(toBits(v0), 64);              // first sample stored raw
  let prevT = t0, prevDelta = 0, prevV = toBits(v0), prevLead = -1, prevTrail = 0;
  for (const [t, v] of samples.slice(1)) {
    // Timestamps: most scrapes are exactly one interval apart, so delta-of-delta is usually 0 → one bit
    const delta = t - prevT, dod = BigInt(delta - prevDelta);
    if (dod === 0n) w.write(0n, 1);
    else if (dod >= -63n && dod <= 64n) { w.write(0b10n, 2); w.write(BigInt.asUintN(7, dod), 7); }
    else if (dod >= -255n && dod <= 256n) { w.write(0b110n, 3); w.write(BigInt.asUintN(9, dod), 9); }
    else if (dod >= -2047n && dod <= 2048n) { w.write(0b1110n, 4); w.write(BigInt.asUintN(12, dod), 12); }
    else { w.write(0b1111n, 4); w.write(BigInt.asUintN(32, dod), 32); }
    prevDelta = delta; prevT = t;
    // Values: XOR with the previous value; unchanged → one bit, small change → only the meaningful bits
    const bits = toBits(v), x = bits ^ prevV;
    if (x === 0n) w.write(0n, 1);
    else {
      const lead = Math.min(clz64(x), 31), trail = ctz64(x);
      if (prevLead >= 0 && lead >= prevLead && trail >= prevTrail) {      // fits the previous window: reuse it
        w.write(0b10n, 2); w.write(x >> BigInt(prevTrail), 64 - prevLead - prevTrail);
      } else {
        const len = 64 - lead - trail;
        w.write(0b11n, 2); w.write(BigInt(lead), 5); w.write(BigInt(len - 1), 6); w.write(x >> BigInt(trail), len);
        prevLead = lead; prevTrail = trail;
      }
    }
    prevV = bits;
  }
  return { bytes: w.bytes(), count: samples.length };
}

export function decode({ bytes, count }) {
  const r = new BitReader(bytes);
  let t = Number(r.read(64)), vBits = r.read(64);
  const out = [[t, fromBits(vBits)]];
  let delta = 0, lead = 0, trail = 0;
  for (let i = 1; i < count; i++) {
    let dod = 0n;
    if (r.read(1) === 1n) {
      if (r.read(1) === 0n) dod = signed(r.read(7), 7);
      else if (r.read(1) === 0n) dod = signed(r.read(9), 9);
      else if (r.read(1) === 0n) dod = signed(r.read(12), 12);
      else dod = signed(r.read(32), 32);
    }
    delta += Number(dod); t += delta;
    if (r.read(1) === 1n) {
      if (r.read(1) === 1n) { lead = Number(r.read(5)); const len = Number(r.read(6)) + 1; trail = 64 - lead - len; }
      vBits ^= r.read(64 - lead - trail) << BigInt(trail);
    }
    out.push([t, fromBits(vBits)]);
  }
  return out;
}

// ── Demo: two hours of 15-second scrapes ──────────────────────────────────────────────
const start = 1_793_000_000, n = 480;
let seed = 7;
const rand = () => ((seed = (seed * 1103515245 + 12345) % 2 ** 31) / 2 ** 31);
const series = {
  'requests_total (counter)': Array.from({ length: n }, (_, i) => [start + 15 * i, 1_000_000 + 37 * i]),
  'queue_depth (gauge, often flat)': Array.from({ length: n }, (_, i) => [start + 15 * i, i % 40 < 30 ? 12 : 12 + (i % 40)]),
  'cpu_ratio (noisy gauge)': Array.from({ length: n }, (_, i) => [start + 15 * i + (i % 50 === 0 ? 1 : 0), Math.round((0.4 + 0.2 * rand()) * 1000) / 1000]),
};
for (const [name, samples] of Object.entries(series)) {
  const chunk = encode(samples);
  const ok = JSON.stringify(decode(chunk)) === JSON.stringify(samples);
  console.log(`${name.padEnd(32)} ${(chunk.bytes.length / n).toFixed(2)} bytes/sample (raw 16) round-trip ${ok}`);
}
// requests_total (counter)         2.11 bytes/sample (raw 16) round-trip true
// queue_depth (gauge, often flat)  0.60 bytes/sample (raw 16) round-trip true
// cpu_ratio (noisy gauge)          7.15 bytes/sample (raw 16) round-trip true
```

Every series round-trips exactly. The flat-most-of-the-time gauge compresses to 0.6 bytes per sample — mostly two bits per sample, one for the
timestamp and one for "unchanged". The counter costs ~2 bytes: its timestamps are free, but each increment flips a run of mantissa bits. The noisy
decimal gauge costs ~7 bytes — still better than 16, but noise and decimal fractions (which aren't exact in binary) are what XOR encoding can't remove;
rounding values to the precision anyone actually uses is the cheapest storage optimisation available. Real workloads average 1–2 bytes per sample.

The decision to defend is **compressing per series, in time order, with a format that can only be read sequentially**, rather than using a general-purpose
columnar format with random access (each sample independently addressable, compressed with a general codec). Random access sounds more flexible —
read sample 87 without decoding 86 others. But metric queries never want sample 87: they want *every* sample of a series over a range, to compute a
rate or an aggregate, and they read chunks of ~120 samples at a time. Sequential delta and XOR encoding exploits exactly what's true of metrics — regular
timestamps, values close to their predecessor — to get 10× compression that a general codec can't match on such short blocks, and decoding a whole chunk
is a tight loop of bit reads. Chunks bound the cost of the one thing sequential encoding gives up: the worst case is decoding 119 extra samples to reach
the start of a range, which is trivial next to the I/O it saved.

---

## 48.11 Questions

**1.** MCQ — What is the primary cost unit of a metrics platform? a) Queries b) Active time series c) Dashboards d) Users
<details><summary>Answer</summary>**b)** Each series carries fixed memory and index cost regardless of samples.</details>

**2.** MCQ — Why do distributors send each series to the same set of ingesters? a) Security b) So each ingester holds complete, in-order data for its series, compressible as it arrives c) DNS d) Billing
<details><summary>Answer</summary>**b)** Hashing on series labels gives every series a stable, replicated home.</details>

**3.** MCQ — What does delta-of-delta timestamp encoding exploit? a) Large values b) Regular scrape intervals, which make the second difference usually zero c) Sorted labels d) Random access
<details><summary>Answer</summary>**b)** A zero delta-of-delta costs one bit.</details>

**4.** MCQ — What should a downsampled point store? a) The average only b) Min, max, sum, and count over the window c) The last value only d) The p99
<details><summary>Answer</summary>**b)** Sum and count give correct averages and rates; min and max preserve extremes.</details>

**5.** MCQ — How is a label selector resolved? a) Scanning all samples b) Intersecting (and, for regexes, unioning) postings lists from an inverted index c) A B-tree on timestamps d) Full-text search
<details><summary>Answer</summary>**b)** Smallest list first.</details>

**6.** MCQ — What is a dead man's switch in monitoring? a) An emergency shutdown b) An always-firing alert sent to an external service that pages if it stops arriving c) A kill switch for queries d) A silence
<details><summary>Answer</summary>**b)** It detects the monitoring system's own silence.</details>

**7.** Short — What is series churn and why does it matter?
<details><summary>Answer</summary>The creation of new series over time as label values change — typically pod names on every deploy or autoscale. Old series linger until blocks are cut, so the head and each block's index grow even though the active count at any instant looks stable.</details>

**8.** Short — Why are blocks immutable, and what does that buy?
<details><summary>Answer</summary>Once a time range is complete, its data doesn't change. Immutability makes blocks safe to cache anywhere, cheap to replicate, compactable by writing new blocks, and deletable by dropping files — no updates, tombstones, or coordination.</details>

**9.** Short — Why use pull (scraping) at all, given push is simpler for networks?
<details><summary>Answer</summary>Pull makes target health a first-class signal (a failed scrape means down), lets the platform control the rate, and keeps targets simple. Agents near the targets can scrape and then push upstream, combining both.</details>

**10.** Short — What does the alert manager do that rule evaluators don't?
<details><summary>Answer</summary>Deduplicates the same alert from replicas, groups related alerts into one notification, inhibits dependent alerts, applies silences, and routes to the owning team's channels and rotation.</details>

**11.** Short — Why must the metrics platform not run on the infrastructure it monitors?
<details><summary>Answer</summary>A failure of that shared infrastructure (cluster, DNS, mesh, identity) would take down monitoring at the same moment, blinding responders to the very failure they need to diagnose.</details>

**12.** Output — Using §48.2's model, 50,000 targets × 2,000 series each at a 30-second scrape interval: how many samples per second?
<details><summary>Answer</summary>**~3.33 million.** 100 million series / 30 s.</details>

**13.** Output — Using §48.10's encoding, how many bits does a sample cost when its timestamp is exactly one interval after the previous and its value is unchanged?
<details><summary>Answer</summary>**2 bits.** One `0` bit for a zero delta-of-delta, one `0` bit for a zero XOR.</details>

**14.** Output — Using §48.6's index, how many series match `{ __name__: 'http_requests_total', service: 'auth', route: '/route/3' }`?
<details><summary>Answer</summary>**180.** 6 statuses × 30 pods.</details>

**15.** Output — A window has 1 request at 2 s; the next has 1,000 requests at 100 ms average. What's the true mean, and what does averaging the two averages give?
<details><summary>Answer</summary>**~102 ms true mean; 1,050 ms averaged.** (2 + 100) s / 1,001 requests ≈ 0.102 s, versus (2 s + 0.1 s) / 2.</details>

**16.** Debug — Ingester memory doubled overnight; active series per tenant look flat, but a new team's metric shows 3 million series. A label holds request IDs. Fix, and prevent?
<details><summary>Answer</summary>Drop or relabel the request-ID label at the agent and have the team move it to traces or logs. Prevent with per-tenant and per-metric series limits at the distributor, rejecting new series with a clear error, and expose per-metric cardinality so teams see it themselves.</details>

**17.** Debug — A dashboard panel takes 40 seconds and times out; its query sums `rate()` over 80,000 series across 30 days. What do you change?
<details><summary>Answer</summary>Create a recording rule that precomputes the aggregate the panel shows; for long ranges, read downsampled data; let the query frontend split and cache by day. Enforce query limits so one panel can't overload queriers.</details>

**18.** Debug — During a regional network outage, alerts didn't fire, and afterwards there were gaps in the data exactly for the outage window. What two changes help?
<details><summary>Answer</summary>Agent-side buffering on disk with an out-of-order acceptance window, so late samples from the outage are ingested when connectivity returns; and a dead man's switch plus independent meta-monitoring, so the platform's own blindness pages someone.</details>

**19.** Debug — A service restarted and its error-rate graph showed a huge negative spike. Why, and how is it computed correctly?
<details><summary>Answer</summary>Counters reset to zero on restart, and a naive difference treats the drop as a negative rate. `rate()` and `increase()` must detect decreases and treat the new value as the increase since a reset (Question 20).</details>

**20.** Code — Write `increase(samples)` for a counter, handling resets.
<details><summary>Answer</summary>

```js
function increase(samples) {                       // samples: [[t, counterValue], ...] in time order
  let total = 0;
  for (let i = 1; i < samples.length; i++) {
    const d = samples[i][1] - samples[i - 1][1];
    total += d >= 0 ? d : samples[i][1];           // a drop means the process restarted from 0
  }
  return total;
}
console.log(increase([[0, 100], [15, 130], [30, 5], [45, 25]])); // 55
```
</details>

**21.** Code — Write `step(state, { firing, now }, forMs)` for an alert state machine (inactive → pending → firing) honouring a `for` duration.
<details><summary>Answer</summary>

```js
function step(state, { firing, now }, forMs) {
  if (!firing) return { status: 'inactive' };                        // any false evaluation resets
  if (state.status === 'inactive') return { status: 'pending', since: now };
  if (state.status === 'pending' && now - state.since >= forMs) return { status: 'firing', since: state.since };
  return state;
}
let s = { status: 'inactive' }; const out = [];
for (const [now, firing] of [[0, true], [60_000, true], [120_000, true], [180_000, true], [240_000, false]]) { s = step(s, { firing, now }, 120_000); out.push(s.status); }
console.log(out.join(' ')); // pending pending firing firing inactive
```
</details>

**22.** Code — Write `downsample(samples, windowSec)` that returns `{ start, min, max, sum, count }` per window.
<details><summary>Answer</summary>

```js
function downsample(samples, windowSec) {
  const out = new Map();
  for (const [t, v] of samples) {
    const start = t - (t % windowSec);                        // align windows so every series agrees on boundaries
    const w = out.get(start) ?? { start, min: Infinity, max: -Infinity, sum: 0, count: 0 };
    Object.assign(w, { min: Math.min(w.min, v), max: Math.max(w.max, v), sum: w.sum + v, count: w.count + 1 });
    out.set(start, w);
  }
  return [...out.values()];
}
console.log(JSON.stringify(downsample([[0, 2], [15, 4], [300, 10], [315, 6]], 300)));
// [{"start":0,"min":2,"max":4,"sum":6,"count":2},{"start":300,"min":6,"max":10,"sum":16,"count":2}]
```
</details>

**23.** Scenario — Product wants per-customer API latency dashboards for 40,000 customers, in the metrics platform. How do you respond?
<details><summary>Answer</summary>Per-customer labels on a latency histogram would add tens of millions of series. Instead: keep operational metrics per service and route; send per-request events with customer ID to an analytics store built for high cardinality (Chapter 25) for per-customer dashboards; and, if a few customers need real-time metrics, emit them for an explicit top-N list or with a bounded `tier` label.</details>

**24.** Scenario — Your company's monitoring runs on the same Kubernetes cluster as production, and an outage of that cluster's DNS took monitoring down with it. Redesign the deployment.
<details><summary>Answer</summary>Run the platform on separate infrastructure — its own cluster, DNS, and identity dependencies, ideally in another region — with agents in each production cluster that buffer locally and push. Add a small independent meta-monitoring stack and a dead man's switch to an external paging provider, and test failure scenarios where production's shared services are down.</details>

---

## Chapter Summary

- Active series are the cost unit: they drive ingester memory, index size, and query cost, so cardinality and churn are platform-level concerns.
- Ingest hashes each series to three ingesters with a write quorum, a WAL, and an in-memory head; two-hour immutable blocks go to object storage.
- Delta-of-delta timestamps and XOR-encoded values compress samples from 16 bytes to about 1–2; noise and needless precision are what remain.
- Compaction merges and deduplicates blocks; downsampling keeps min, max, sum, and count; retention drops whole blocks.
- An inverted index resolves selectors; per-tenant limits, relabelling, and recording rules keep cardinality and query cost bounded.
- Alerting needs replicated evaluators, an alert manager that dedupes, groups, inhibits, and routes — and a dead man's switch, because a silent monitor
  looks healthy.

---

## Practice Exercises

1. Extend §48.10's encoder into a chunked store: 120 samples per chunk, a series index from §48.6, and a range query that decodes only the chunks it needs.
2. Measure §48.10's bytes per sample on real metrics exported from a running service, and compare with general-purpose compression of the same data.
3. Simulate churn: a service with 30 pods deploying 20 times a day, and chart active series versus series created per day over a week.
4. Implement per-tenant series limits in a distributor that rejects new series beyond a limit while still accepting samples for existing ones.
5. Build an alert manager that groups alerts by `alertname` and `service`, inhibits service alerts while a `DatabaseDown` alert fires, and deduplicates
   replica alerts.

---

## Mini Project: A Small Metrics Platform

Build a working metrics pipeline end to end and test it under load and failure.

- An agent that scrapes targets in a simple text exposition format and pushes compressed batches, buffering on disk when the platform is unreachable.
- A distributor with tenant limits, hashing series to three ingesters with a 2-of-3 write quorum.
- Ingesters with a WAL, Gorilla-compressed in-memory chunks, and two-hour blocks written to a local directory standing in for object storage.
- A querier supporting selectors, `rate()` with counter resets, `sum by`, and range queries across ingesters and blocks with replica deduplication.
- A rule evaluator with `for` durations, a minimal alert manager with grouping and routing, and a dead man's switch — tested by killing an ingester,
  injecting a cardinality explosion, and partitioning the agent.

---

**Next:** [Appendix A — Interview Questions →](../appendices/A-interview-questions.md)
