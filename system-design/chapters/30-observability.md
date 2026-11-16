# Chapter 30 — Observability

## Learning Objectives

- Explain what metrics, logs, traces, and profiles each answer, and use them together in an investigation
- Instrument services with RED and resources with USE, and control metric cardinality
- Write structured, correlated logs that are useful and affordable
- Propagate trace context across services and choose a sampling strategy
- Design alerts on user-facing symptoms using multi-window SLO burn rates, with low noise and clear ownership

---

## 30.1 Monitoring and Observability

**Monitoring** answers questions you knew to ask in advance: is the error rate above 1%? Is the disk 90% full?
**Observability** is the ability to answer questions you didn't anticipate — "why are checkout requests from one
mobile app version in one region slow since 14:05?" — from the data the system already emits, without shipping new
code to find out. Distributed systems need the second, because their failures are novel combinations: a slow
dependency *and* a specific tenant *and* a recent deploy.

Every investigation moves through the same three questions, and each signal type serves one best:

| Question | Best signal | Why |
|---|---|---|
| **Is something wrong?** | Metrics (aggregates over time) | Cheap to store and query at full fidelity; drive alerts and dashboards |
| **Where is it wrong?** | Traces (one request across services) | Show which hop, which dependency, which span took the time or failed |
| **Why is it wrong?** | Logs (detailed events) and profiles (where CPU and memory go) | Carry the specific context: the error message, the query, the hot function |

The signals are only powerful together, which requires them to be **correlated**: a metric spike links to example
traces (exemplars), a trace links to the logs written during it (shared trace IDs), and all of them share consistent
labels for service, version, region, and tenant. **OpenTelemetry** has become the standard way to produce all three
with consistent context, vendor-neutrally, and this chapter uses its concepts.

---

## 30.2 Metrics

A **metric** is a numeric measurement aggregated over time, identified by a name and a set of labels:
`http_requests_total{service="orders", route="/orders/:id", status="500"}`. Three types cover almost everything:

| Type | Behaviour | Examples |
|---|---|---|
| **Counter** | Only goes up (resets on restart); rate is computed at query time | Requests, errors, bytes sent, jobs processed |
| **Gauge** | Goes up and down; a current value | Queue depth, in-flight requests, memory used, replication lag |
| **Histogram** | Counts observations into buckets; percentiles computed later | Request latency, payload size, batch size |

Histograms are the right type for latency because, as Chapter 3 showed, percentiles can't be averaged but bucketed
distributions can be merged across instances and time. Never record "p99 latency" as a gauge from each instance.

Two methods give every component a standard minimum set of metrics:

- **RED, for request-driven services**: **R**ate (requests per second), **E**rrors (failed requests per second, or
  the error ratio), **D**uration (latency histogram). These map directly onto availability and latency SLIs
  (Chapter 4).
- **USE, for resources** (CPU, memory, disks, network, connection pools, queues): **U**tilisation (how busy),
  **S**aturation (how much work is waiting — queue length, pool waiters), **E**rrors. Saturation is the one most often
  missing and the one that predicts trouble (Chapter 3's queueing curve).

### Cardinality

Every unique combination of label values is a separate **time series** that the metrics system must store, index, and
query. Series multiply:

```js
// Every unique combination of label values is a separate time series the metrics system
// must store and index. Cardinality multiplies — one careless label dwarfs everything else.
const seriesCount = (labels) => Object.values(labels).reduce((n, values) => n * values, 1);
const base = { service: 1, method: 5, route: 40, status: 6, instance: 30 };
const histogramBuckets = 12;                     // each latency histogram series is ~12 series
console.log(seriesCount(base) * histogramBuckets);                             // 432000
console.log(seriesCount({ ...base, customer_id: 50_000 }) * histogramBuckets); // 21600000000
```

A reasonable latency histogram for one service is a few hundred thousand series. Adding a `customer_id` label turns it
into tens of billions — which no metrics system will accept, and which will take down shared monitoring for everyone
before it's rejected. The rules:

- **Labels must have bounded, small value sets**: route *templates* (`/orders/:id`), not raw paths; status *codes*, not
  error messages; region, not IP address.
- **High-cardinality identifiers belong in traces and logs**, where they're attributes of individual events rather than
  dimensions of an aggregate.
- **If you need per-customer aggregates**, compute them deliberately for the top N customers, or in an analytics system
  built for that cardinality (Chapter 25), not in the operational metrics pipeline. Chapter 48 designs a metrics store
  and returns to cardinality in depth.

---

## 30.3 Logs

**Logs** record discrete events with detail: a request failed with this error, a job processed these records, a
configuration value changed. They're the richest signal and, at scale, the most expensive one.

Practices that make logs useful:

- **Structured, not free text.** Emit JSON (or another structured format) with consistent field names — `level`,
  `msg`, `service`, `trace_id`, `span_id`, `tenant_id`, `duration_ms`, `error.type` — so logs can be filtered and
  aggregated like data. `"user 42 failed to pay 30 EUR"` can't be queried; `{msg: "payment failed", user_id: 42,
  amount_cents: 3000, currency: "EUR", error_type: "card_declined"}` can.
- **Correlated.** Every log line written while handling a request carries that request's trace ID, so an investigation
  can pull every log from every service for one failing request. §30.8 does this automatically.
- **Levels used consistently.** `error` for failures someone may need to act on; `warn` for degraded but handled
  situations; `info` for significant business events; `debug` off in production by default (and switchable per service
  at runtime when investigating).
- **Never log secrets or unnecessary personal data.** Passwords, tokens, full card numbers, and session cookies end up in
  log storage with broad access and long retention. Redact at the logger, by field name and pattern, and treat logs as
  sensitive data (Chapter 33).

Cost control matters: at 50,000 requests per second, one 1 KB log line per request is 4 TB per day. Log **events**, not
every function call; **sample** high-volume success logs (keep 1%, keep 100% of errors); keep detailed logs hot for
days and archive cheaply for longer; and prefer metrics for anything you'd only ever count.

---

## 30.4 Traces

A **distributed trace** records one request's path through the system as a tree of **spans**. Each span is a timed
operation — an incoming request, an outgoing call, a database query — with a trace ID shared by the whole request, its
own span ID, its parent's span ID, timing, status, and attributes (route, tenant, order ID, SQL statement shape).

Traces answer questions the other signals can't: which of the eleven services in this checkout made it slow; whether
the time went to the database or to waiting for a connection; which calls ran sequentially that could run in parallel;
how deep the call graph really is (Chapter 27's distributed-monolith symptoms).

**Context propagation** is what holds a trace together: every outgoing call carries the current trace ID, span ID, and
sampling decision, and the receiving service continues the trace instead of starting a new one. The **W3C Trace
Context** standard defines the `traceparent` header for HTTP (`00-<trace-id>-<parent-span-id>-<flags>`), and the same
context should flow through message headers on queues and logs (Chapter 24) so asynchronous hops stay in the trace.
Within a process, the current span lives in an async-local context so code doesn't pass it explicitly.

**Sampling** is necessary because tracing every request at high volume is expensive. Two strategies:

| | Head-based sampling | Tail-based sampling |
|---|---|---|
| Decision made | At the start of the trace (the root), propagated to all services | After the trace completes, by a collector that sees all its spans |
| Cost | Cheap — unsampled traces are never recorded | Every span must be sent to the collector and buffered |
| Can keep "all errors and slow requests" | No — the decision precedes the outcome | **Yes** — decide on the outcome |
| Typical use | Baseline sampling (1–10%) | Keeping 100% of errors and tail latency, plus a small sample of the rest |

Many systems combine them: head-sample a modest percentage for the baseline, and use tail sampling in collectors to
also keep every trace with an error or high latency. Whatever the strategy, the decision must be **consistent across a
trace**: if service A samples and service B doesn't, the trace has holes exactly where you need to look.

---

## 30.5 Alerting

Alerts wake people up, so every alert must be worth it. The principles:

- **Alert on symptoms, not causes.** Page on "users are getting errors" (an SLO burning), not on "CPU is at 85%" or
  "one instance restarted". Causes belong on dashboards to diagnose with; symptoms are what users feel. A cause-based
  alert pages for things that don't hurt users and misses failures nobody predicted.
- **Every page must be actionable, urgent, and owned.** If the response is "wait and see" or "it'll recover by itself",
  it's not a page. Each alert names an owning team and links a runbook.
- **Tickets for slow problems.** Things that need attention this week (disk filling in 10 days, error budget slowly
  burning) open tickets, not pages.

**SLO burn-rate alerting** puts these principles into practice. Chapter 4 defined burn rate — how fast the error budget
is being spent relative to plan. A good alert fires when the burn is fast enough to matter **and** is still happening,
which needs two windows: a long one to prove the burn is real (not a 30-second blip) and a short one to prove it hasn't
already stopped (so the page clears soon after recovery). The configuration popularised by Google's SRE Workbook:

```js
// Multi-window, multi-burn-rate SLO alerting (the approach in Google's SRE Workbook):
// page only when the budget is burning fast over BOTH a long window (it's real) and a short
// window (it's still happening); open a ticket for slow, sustained burns.
const RULES = [
  { severity: 'page', burn: 14.4, long: '1h', short: '5m' },    // ~2% of a 30-day budget in an hour
  { severity: 'page', burn: 6, long: '6h', short: '30m' },      // ~5% in six hours
  { severity: 'ticket', burn: 1, long: '3d', short: '6h' },     // on pace to exhaust the budget
];

function evaluate({ objective, errorRate }) {
  const budget = 1 - objective;
  const burn = (w) => errorRate[w] / budget;
  for (const r of RULES) {
    if (burn(r.long) >= r.burn && burn(r.short) >= r.burn) {
      return `${r.severity}: burn ${burn(r.long).toFixed(1)}× over ${r.long} (and ${burn(r.short).toFixed(1)}× over ${r.short})`;
    }
  }
  return 'ok';
}

const objective = 0.999;
console.log(evaluate({ objective, errorRate: { '5m': 0.02, '1h': 0.016, '30m': 0.018, '6h': 0.004, '3d': 0.0008 } }));
console.log(evaluate({ objective, errorRate: { '5m': 0.0001, '1h': 0.016, '30m': 0.002, '6h': 0.004, '3d': 0.0008 } }));
console.log(evaluate({ objective, errorRate: { '5m': 0.0012, '1h': 0.0011, '30m': 0.0012, '6h': 0.0011, '3d': 0.0011 } }));
console.log(evaluate({ objective, errorRate: { '5m': 0.05, '1h': 0.0009, '30m': 0.001, '6h': 0.0004, '3d': 0.0003 } }));
// page: burn 16.0× over 1h (and 20.0× over 5m)
// ok
// ticket: burn 1.1× over 3d (and 1.1× over 6h)
// ok
```

The four cases show the design: an ongoing major incident pages; the same hour-long burn that *has stopped* (5-minute
error rate back to normal) doesn't page any more; a slow sustained burn opens a ticket; and a two-minute spike — 50×
burn over 5 minutes, but not over the hour — doesn't page at all. Compared with "error rate > 1% for 5 minutes",
burn-rate alerts page on what threatens the SLO and stay quiet on what doesn't.

> 💼 **Interview** — "How would you set up alerting for this service?" A strong answer: SLIs on user-facing symptoms
> (availability and latency at the edge), SLOs with error budgets, multi-window burn-rate pages, cause metrics on
> dashboards rather than pages, runbooks for every alert, and a review of alert noise in every on-call handover.

---

## 30.6 Investigating With the Signals Together

A typical investigation, and the tool that serves each step:

```text
1. PAGE: checkout availability SLO burning 14×        ← burn-rate alert on edge metrics
2. Service dashboard: errors are 503s from /checkout, began 14:05, all regions      ← RED metrics, deploy markers
3. Exemplar trace from the error spike: payments-client span times out at 2 s       ← metrics → traces link
4. Payments service dashboard: DB connection pool saturated, waiters climbing       ← USE metrics
5. Logs for that trace ID in payments: "pool exhausted" since deploy v812 at 14:03  ← traces → logs link
6. Profile of payments v812: new N+1 query in fraud-score lookup holding connections ← continuous profiling
7. Mitigate: roll back v812 (Chapter 31). Fix: batch the lookup.
```

Three capabilities make this fast, and all are cheap to add early and expensive to retrofit:

- **Deploy and change markers** on every dashboard. Most incidents are changes (Chapter 31); seeing "v812 deployed at
  14:03" next to a spike at 14:05 often ends the investigation.
- **Exemplars**: histogram buckets that store a few sample trace IDs, so a latency spike on a graph links directly to
  traces of the slow requests.
- **Continuous profiling**: low-overhead, always-on CPU and memory profiles, labelled by service and version, so "why is
  this version slower?" is answered by a diff of two flame graphs rather than a reproduction.

Dashboards should follow the investigation's shape: a top-level view per user journey (SLIs, burn), then one per service
(RED, dependencies' RED from the caller's side, saturation), then per resource (USE). A dashboard with 60 graphs and no
hierarchy is a wall of noise during an incident.

---

## 30.7 Common Mistakes and Best Practices

**Mistake: averaging percentiles or recording p99 as a gauge.** The numbers are meaningless; record histograms and
compute percentiles after aggregation (Chapter 3).

**Mistake: unbounded label values.** User IDs, raw URLs, or error strings as labels explode cardinality and can take down
shared monitoring.

**Mistake: unstructured logs.** Free text can't be filtered, joined, or aggregated during an incident.

**Mistake: logging secrets and personal data.** Tokens and card numbers in logs are a breach waiting for broad log
access.

**Mistake: inconsistent sampling across services.** Traces with missing spans hide exactly the hops under investigation.

**Mistake: paging on causes.** CPU and restart alerts wake people for things that don't hurt users and miss novel
failures.

**Mistake: no change markers on dashboards.** The most likely cause of an incident — a recent change — stays invisible.

**Best practice: RED for every service, USE for every resource.** A standard minimum that makes every service
debuggable the same way.

**Best practice: one trace ID everywhere.** In traces, every log line, message headers, and error reports, so all signals
for one request can be pulled together.

**Best practice: burn-rate alerts on SLOs, with runbooks.** Pages that mean users are hurting, with a known first step.

---

## 30.8 Real-World Example: Trace Propagation With Correlated Logs

This module implements the core of distributed tracing: parsing and emitting W3C `traceparent` headers, spans held in
async-local context so nested code needn't pass them around, a sampling decision made once at the root and inherited
everywhere, and a logger that stamps every line with the current trace and span IDs. The demo traces a request from a
gateway through an orders service and its database query.

```js
// tracing.mjs — W3C trace-context propagation, spans, and logs that carry trace IDs
// automatically. The shape of what OpenTelemetry does, small enough to read in one sitting.
import { AsyncLocalStorage } from 'node:async_hooks';
import { randomBytes } from 'node:crypto';

const als = new AsyncLocalStorage();
let hex = (bytes) => randomBytes(bytes).toString('hex');
export const setIdGenerator = (fn) => { hex = fn; };   // deterministic IDs for tests and this demo
export const exported = [];                         // stand-in for an exporter to a tracing backend

// traceparent: version-traceId(16 bytes)-parentSpanId(8 bytes)-flags (01 = sampled)
export function parseTraceparent(header) {
  const m = /^00-([0-9a-f]{32})-([0-9a-f]{16})-([0-9a-f]{2})$/.exec(header ?? '');
  return m ? { traceId: m[1], parentId: m[2], sampled: (parseInt(m[3], 16) & 1) === 1 } : null;
}

export function startSpan(name, { incoming, sampleRate = 0.1, rand = Math.random } = {}) {
  const parent = als.getStore() ?? incoming;          // in-process parent first, then the remote one
  const span = {
    traceId: parent?.traceId ?? hex(16),
    spanId: hex(8),
    parentId: parent?.spanId ?? parent?.parentId ?? null,
    // The sampling decision is made ONCE, at the root, and inherited by every descendant
    sampled: parent ? parent.sampled : rand() < sampleRate,
    name, start: performance.now(), attributes: {},
  };
  return span;
}

export async function withSpan(name, fn, opts) {
  const span = startSpan(name, opts);
  try {
    return await als.run(span, () => fn(span));
  } catch (err) {
    span.attributes.error = err.message;
    throw err;
  } finally {
    span.durationMs = +(performance.now() - span.start).toFixed(1);
    if (span.sampled) exported.push(span);
  }
}

// Outgoing calls carry the current context so the next service continues the same trace
export const traceparent = () => {
  const s = als.getStore();
  return s ? `00-${s.traceId}-${s.spanId}-${s.sampled ? '01' : '00'}` : undefined;
};

// Every log line includes the trace and span IDs, without callers passing them around
export function log(level, msg, fields = {}) {
  const s = als.getStore();
  console.log(JSON.stringify({ level, msg, trace_id: s?.traceId, span_id: s?.spanId, ...fields }));
}

// ── Demo: a gateway calls an orders service (over a simulated HTTP hop) ─────────────
let n = 0;
setIdGenerator((bytes) => String(++n).padStart(bytes * 2, '0'));
async function ordersService(headers) {
  return withSpan('orders.get', async (span) => {
    span.attributes['order.id'] = 'o-77';
    await withSpan('db.query', () => new Promise((r) => setTimeout(r, 15)));
    log('info', 'order loaded', { order_id: 'o-77' });
    return { status: 200 };
  }, { incoming: parseTraceparent(headers.traceparent) });
}

await withSpan('gateway GET /orders/o-77', async () => {
  log('info', 'request received');
  await ordersService({ traceparent: traceparent() }); // the header crosses the "network"
}, { rand: () => 0.05 });                               // below 0.1 → this trace is sampled

const ids = new Set(exported.map((s) => s.traceId));
console.log(`spans exported: ${exported.length}, distinct traces: ${ids.size}`);
for (const s of exported) console.log(`${s.name.padEnd(26)} parent=${s.parentId ? 'yes' : 'root'}`);
// {"level":"info","msg":"request received","trace_id":"00000000000000000000000000000001","span_id":"0000000000000002"}
// {"level":"info","msg":"order loaded","trace_id":"00000000000000000000000000000001","span_id":"0000000000000003","order_id":"o-77"}
// spans exported: 3, distinct traces: 1
// db.query                   parent=yes
// orders.get                 parent=yes
// gateway GET /orders/o-77   parent=root
```

Both log lines — written in different "services" — share one trace ID, so a log search for that ID returns the whole
request; the three spans form one trace with the gateway as root. In production, OpenTelemetry's SDK provides all of this
plus automatic instrumentation of HTTP, database, and messaging libraries.

The decision to defend is **making the sampling decision once, at the root, and propagating it in the `traceparent`
flags**, rather than letting each service sample independently at its own rate. Independent sampling looks flexible — a
busy service could sample less — but it's corrosive: with three services each sampling 10% independently, only 0.1% of
traces contain all three, and the rest show a gateway span calling into nothing, or a database query with no parent.
The traces you'd need during an incident are almost always incomplete. Propagating one decision means a sampled trace
is complete everywhere and an unsampled one costs nothing anywhere. When some services need to keep more (all errors,
all slow requests), that's what tail-based sampling in the collector is for — a second decision made on complete traces,
not a per-service guess made on fragments.

---

## 30.9 Questions

**1.** MCQ — Which signal is best for answering "which service in this request made it slow"? a) Metrics b) Logs c) Traces d) Uptime checks
<details><summary>Answer</summary>**c)** A trace shows each hop's span and timing for one request.</details>

**2.** MCQ — In the USE method, what does "saturation" measure? a) Error count b) How busy a resource is c) How much work is waiting for the resource d) Total capacity
<details><summary>Answer</summary>**c)** Queue length, pool waiters — the signal that predicts latency trouble.</details>

**3.** MCQ — Which label is safe on an HTTP request metric? a) `user_id` b) Raw request path `/orders/918273` c) Route template `/orders/:id` d) Full error message
<details><summary>Answer</summary>**c)** It has a small, bounded set of values; the others are unbounded.</details>

**4.** MCQ — What can tail-based sampling do that head-based sampling can't? a) Reduce cost b) Keep every trace that ended in an error or was slow c) Propagate context d) Sample consistently
<details><summary>Answer</summary>**b)** It decides after seeing the whole trace, so it can select on outcome.</details>

**5.** MCQ — Why does a burn-rate alert use both a long and a short window? a) To double the alert count b) The long window shows the burn is significant; the short one shows it's still happening c) Prometheus requires it d) To measure latency
<details><summary>Answer</summary>**b)** Together they avoid paging on brief blips and stop paging soon after recovery.</details>

**6.** MCQ — What's the main problem with alerting on "CPU > 85%"? a) CPU can't be measured b) It's a cause, not a user-facing symptom: it can fire without user impact and miss failures with normal CPU c) It's too slow d) It never fires
<details><summary>Answer</summary>**b)** Page on symptoms such as SLO burn; keep cause metrics on dashboards.</details>

**7.** Short — Why should latency be recorded as a histogram rather than a p99 gauge per instance?
<details><summary>Answer</summary>Percentiles from different instances or time windows can't be combined meaningfully, while histogram buckets can be summed and the percentile computed from the merged distribution (Chapter 3).</details>

**8.** Short — What does W3C Trace Context's `traceparent` header carry?
<details><summary>Answer</summary>A version, the 16-byte trace ID, the 8-byte ID of the calling span (the parent), and flags including whether the trace is sampled.</details>

**9.** Short — Name three practices that keep logging affordable at high volume.
<details><summary>Answer</summary>Any three of: log events rather than every step; sample high-volume success logs while keeping all errors; tier retention (hot for days, cheap archive for longer); use metrics for things you'd only count; keep debug logs off by default with runtime toggles.</details>

**10.** Short — What are exemplars, and why are they useful?
<details><summary>Answer</summary>Sample trace IDs attached to metric data points (often histogram buckets). They let you jump from a spike on a latency graph straight to traces of the requests that caused it.</details>

**11.** Short — Why must every page be actionable and owned?
<details><summary>Answer</summary>Pages that require no action, or that nobody knows how to handle, train responders to ignore alerts and erode on-call health, so real incidents are missed or slow to handle.</details>

**12.** Output — Using §30.2's `seriesCount`, how many series does a latency histogram with 12 buckets have for labels `{ route: 25, method: 4, status: 5, region: 3, instance: 20 }`?
<details><summary>Answer</summary>**360,000.** 25 × 4 × 5 × 3 × 20 = 30,000 label combinations × 12 buckets.</details>

**13.** Output — Using §30.5's rules with a 99.9% objective, error rates `{ '5m': 0.008, '1h': 0.007, '30m': 0.008, '6h': 0.0065, '3d': 0.002 }` — what does `evaluate` return?
<details><summary>Answer</summary>**`page: burn 6.5× over 6h (and 8.0× over 30m)`.** The first rule needs 14.4× (the hour is only 7×), so it doesn't fire; the second needs 6× over both 6 h (6.5×) and 30 min (8×), so it pages.</details>

**14.** Output — Three services each sample traces independently at 20%. What fraction of requests that touch all three produce a complete trace?
<details><summary>Answer</summary>**0.8%.** 0.2³ = 0.008 — which is why the sampling decision must be propagated from the root.</details>

**15.** Output — A service logs one 800-byte line per request at 20,000 requests/s. How much log data per day is that, in TB (decimal)?
<details><summary>Answer</summary>**~1.4 TB/day.** 20,000 × 800 B × 86,400 s ≈ 1.38 × 10¹² bytes.</details>

**16.** Debug — The shared metrics cluster fell over shortly after a team deployed a change; ingestion series count jumped 40×. What likely happened, and how is it prevented?
<details><summary>Answer</summary>The change added a high-cardinality label (user ID, raw path, request ID) to a metric. Remove it, move the identifier to traces or logs, and add guardrails: per-service series limits in the metrics pipeline, label allow-lists, and cardinality checks in CI or at the collector.</details>

**17.** Debug — Traces for checkout show the gateway span calling the orders service, but the orders service's spans appear as separate traces. Both use the same tracing SDK. Cause and fix?
<details><summary>Answer</summary>Context isn't propagated across that hop: the HTTP client used by the gateway (or a proxy in between) drops or doesn't add the `traceparent` header, or the orders service doesn't extract it. Instrument the client (or forward the header in the proxy) and extract incoming context in the server.</details>

**18.** Debug — On-call engineers are paged 30 times a week; 26 of the pages resolve on their own within minutes. Most are "error rate > 1% for 2 minutes". What do you change?
<details><summary>Answer</summary>Replace threshold alerts with multi-window burn-rate alerts against SLOs, so brief blips that don't threaten the budget don't page; move cause-based alerts to dashboards or tickets; require every remaining page to have a runbook and an owner; and review alert noise each on-call rotation.</details>

**19.** Debug — During an incident, the team spent 40 minutes before noticing the spike began two minutes after a configuration change. What would have surfaced it immediately?
<details><summary>Answer</summary>Change markers — deploys, config changes, feature-flag flips, migrations — annotated on every dashboard and searchable in the incident tool, plus a standard first step in runbooks: "what changed in the last hour?" (Chapter 31).</details>

**20.** Code — Write `redact(record)` that replaces values of fields named like `password`, `token`, `secret`, or `authorization` (case-insensitive, at any depth) with `"[REDACTED]"`.
<details><summary>Answer</summary>

```js
const SENSITIVE = /pass(word)?|token|secret|authorization|api[-_]?key|cookie/i;
function redact(value) {
  if (Array.isArray(value)) return value.map(redact);
  if (value && typeof value === 'object') {
    return Object.fromEntries(Object.entries(value).map(([k, v]) => [k, SENSITIVE.test(k) ? '[REDACTED]' : redact(v)]));
  }
  return value;
}
console.log(JSON.stringify(redact({ user: 'ana', password: 'x', headers: { Authorization: 'Bearer abc', accept: 'json' } })));
// {"user":"ana","password":"[REDACTED]","headers":{"Authorization":"[REDACTED]","accept":"json"}}
```
Field-name redaction misses secrets embedded in free text, so also avoid logging raw request bodies and URLs with query strings.</details>

**21.** Code — Write `routeTemplate(path, patterns)` that maps a raw path to its template for use as a metric label, returning `'other'` when nothing matches.
<details><summary>Answer</summary>

```js
function routeTemplate(path, patterns) {
  for (const p of patterns) {
    const re = new RegExp(`^${p.replace(/:\w+/g, '[^/]+')}$`);
    if (re.test(path)) return p;
  }
  return 'other'; // unknown paths share one bucket instead of creating a series each
}
console.log(routeTemplate('/orders/918273/items', ['/orders/:id', '/orders/:id/items']), routeTemplate('/wp-admin.php', ['/orders/:id'])); // /orders/:id/items other
```
</details>

**22.** Code — Write `burnRate(goodTotalPairs, objective)` that takes `[{ good, total }]` samples covering a window and returns the window's burn rate.
<details><summary>Answer</summary>

```js
function burnRate(samples, objective) {
  const good = samples.reduce((s, x) => s + x.good, 0), total = samples.reduce((s, x) => s + x.total, 0);
  if (total === 0) return 0;
  return (1 - good / total) / (1 - objective); // sum first, then divide — never average per-sample ratios
}
console.log(burnRate([{ good: 990, total: 1000 }, { good: 9_990, total: 10_000 }], 0.999).toFixed(2)); // 1.82
```
Summing before dividing weights each sample by its traffic; averaging the two ratios would give 5.5 — wrong in exactly the way averaging percentiles is wrong.</details>

**23.** Scenario — You're joining a team with 300 dashboards, 150 alerts, no traces, and long incident resolution times. What do you do first?
<details><summary>Answer</summary>Define SLIs and SLOs for the top user journeys and replace the noisiest alerts with burn-rate pages (each with a runbook); delete alerts nobody acts on. Add OpenTelemetry tracing with auto-instrumentation and propagated context, and stamp trace IDs into logs. Add deploy and change markers to dashboards. Build a small hierarchy of dashboards — journey, service (RED), resource (USE) — and retire the rest gradually. Measure progress by time-to-detect and time-to-mitigate.</details>

**24.** Scenario — Product managers want per-customer latency and error dashboards for 40,000 enterprise customers. The platform team says it will break the metrics system. Who's right, and what do you build?
<details><summary>Answer</summary>Both: per-customer labels on operational metrics would explode cardinality. Build it from traces and logs instead — attach `customer_id` as a span attribute and log field, and compute per-customer aggregates in an analytics store (columnar, Chapter 25) fed from the telemetry pipeline, at minute granularity. Optionally keep real-time metrics for the top N customers by revenue as a bounded label set. Customer-facing SLA reports come from the same analytics pipeline.</details>

---

## Chapter Summary

- Observability is answering unanticipated questions from emitted data; metrics say something's wrong, traces say where,
  and logs and profiles say why — and only correlated signals make investigations fast.
- Instrument services with RED and resources with USE, record latency as mergeable histograms, and keep label
  cardinality bounded; high-cardinality identifiers belong in traces and logs.
- Logs should be structured, correlated by trace ID, levelled consistently, redacted of secrets, and sampled and tiered to
  control cost.
- Traces need propagated context (W3C `traceparent`, including through queues) and a sampling decision made once per
  trace, with tail sampling to keep errors and slow requests.
- Alert on user-facing symptoms with multi-window SLO burn rates; every page must be urgent, actionable, owned, and backed
  by a runbook, with slow burns as tickets.
- Deploy markers, exemplars, and continuous profiling turn an investigation from guesswork into a short sequence of links.

---

## Practice Exercises

1. Add OpenTelemetry to two small services that call each other over HTTP and a queue, and confirm one trace spans both,
   including the asynchronous hop.
2. Audit a service's metrics for cardinality: list every label, its number of distinct values, and the resulting series
   count; remove or bucket any unbounded label.
3. Read the burn-rate alerting chapter of Google's SRE Workbook and implement its alert rules for one of your SLOs in your
   monitoring system.
4. Take a past incident and reconstruct the investigation path: which signal would have answered each question fastest,
   and which links (exemplars, trace IDs in logs, change markers) were missing.
5. Enable continuous profiling for a service, deploy a deliberately slower version, and find the regression from a
   flame-graph diff alone.

---

## Mini Project: An Observability Starter Kit

Build a reusable package that gives any Node service production-grade telemetry with one import, and prove it on a
three-service demo application with an injected fault.

- HTTP server and client instrumentation: RED metrics as histograms with route templates, spans with propagated W3C
  context, and correlated structured logs with redaction.
- Queue producer and consumer helpers that propagate trace context through message headers.
- Head sampling at the edge with consistent propagation, plus a collector configuration that tail-samples errors and slow
  traces.
- SLO definitions as code generating multi-window burn-rate alert rules, each with a linked runbook template.
- A fault-injection scenario (a slow dependency after a "deploy") and a written investigation that uses the alert,
  dashboard, exemplar, trace, logs, and change marker in order.

---

**Next:** [Chapter 31 — Deployment and Change Management →](31-deployment-change.md)
