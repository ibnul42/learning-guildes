# Chapter 28 — Resilience Patterns

## Learning Objectives

- Trace how a slow or failing dependency cascades into an outage of its callers
- Set timeouts from latency budgets and propagate deadlines across service hops
- Retry only what's safe, with exponential backoff, jitter, and a retry budget — in exactly one layer
- Use circuit breakers, bulkheads, and adaptive concurrency limits to contain failures
- Design fallbacks and graceful degradation, and test that they actually work

---

## 28.1 How Failures Cascade

Most large outages don't start large. A database slows down for thirty seconds; a dependency starts returning
errors at 2%; one availability zone's network gets lossy. What turns these into outages is the response of
everything around them. Three mechanisms do most of the damage:

**Resource exhaustion from waiting.** Little's law (Chapter 2): when a dependency's latency goes from 20 ms to
2 seconds, every caller holds 100 times as many requests in flight — threads, connections, memory, sockets. Pools
fill, new requests queue, the caller becomes slow for *everything*, including requests that never touch the slow
dependency, and its own callers start piling up in turn.

**Retry amplification.** A request that fails is retried. If each of three layers retries three times, one user
request can become 3³ = 27 calls to the bottom layer — arriving exactly when it's already failing. Retries convert a
dependency's partial failure into a load spike that guarantees total failure.

**Feedback loops.** Overload makes things slower; slower makes timeouts fire; timeouts trigger retries; retries add
load. Health checks fail under load and remove instances, concentrating load on the rest (Chapter 7). Autoscaling adds
cold instances that are slow to start (Chapter 7's slow start). Each loop is reasonable in isolation and catastrophic
in combination.

The patterns in this chapter break those loops. Each one bounds something: **timeouts** bound waiting, **retry
budgets** bound amplification, **circuit breakers** bound how long you keep calling something that's failing,
**bulkheads** bound how much of your capacity one dependency can consume, and **load shedding** (Chapter 29) bounds how
much work you accept.

> 💼 **Interview** — "The payment provider is slow; what happens to our checkout?" is a cascade question. The strong
> answer walks the chain: threads and connections fill (Little's law), unrelated endpoints slow down, retries multiply
> load; then the fixes: a timeout from the latency budget, a bulkhead so payments can't consume all workers, a breaker
> to fail fast, a retry budget, and a product decision about the fallback (queue the order, show "payment pending").

---

## 28.2 Timeouts and Deadlines

Every network call needs a timeout. A call without one can wait forever, and "forever" is how a slow dependency turns
into exhausted pools. There are usually two:

- **Connect timeout** — how long to wait for a connection to be established. Short (100 ms–1 s within a region):
  if a host doesn't answer a TCP handshake quickly, it won't.
- **Request (read) timeout** — how long to wait for the response. Derived from the latency budget (Chapter 3), and set
  above the dependency's normal p99.9 so healthy requests aren't cut off (Chapter 19's timeout arithmetic).

Per-call timeouts aren't enough on a multi-hop path. If the user-facing request has a 1-second budget and service A
spends 900 ms before calling B, B's own 800 ms timeout lets it keep working long after the user gave up. **Deadline
propagation** fixes this: the edge sets an absolute deadline, every hop passes the *remaining* budget downstream (gRPC
does this natively; over HTTP, a header such as a timeout in milliseconds), and each hop's timeout is the smaller of
its own limit and the remaining budget. When the deadline passes, everyone below stops — no work is done for a request
nobody is waiting for.

```text
edge: deadline = now + 1000 ms
  └─▶ A (uses 300 ms) ── remaining 700 ms ──▶ B: timeout = min(B's 800 ms, 700 ms) = 700 ms
                                                └─▶ C: remaining after B's own 200 ms = 500 ms
```

Two related habits: **check the deadline before starting expensive work** (if 5 ms remain, don't start a 50 ms
query), and **cancel downstream work** when the caller disconnects or times out — `AbortSignal` in Node, context
cancellation in Go — so abandoned requests release their resources.

---

## 28.3 Retries Done Right

Retries are the most useful and most dangerous resilience tool. Transient failures — a dropped connection, a node
mid-failover, a brief overload — are common, and one retry fixes most of them. Done carelessly, retries are how a blip
becomes an outage. The rules:

**Retry only what's safe and useful.** Only idempotent operations, or operations with an idempotency key (Chapter 8).
Only transient errors: timeouts, connection resets, `503`, `429` (respecting `Retry-After`), not `400` or `404`, which
will fail identically every time.

**Back off exponentially, with jitter.** Retrying immediately hits a dependency at the moment it's least able to cope.
Exponential backoff spreads retries over time; **jitter** — randomising each delay — prevents clients that failed
together from retrying together. Without jitter, a thousand clients that failed at the same instant retry at the same
instant, every time:

```js
// 1,000 clients fail at the same instant (a dependency blipped) and retry with backoff.
// Without jitter their retries stay synchronised; with full jitter they spread out.
function mulberry32(seed) {
  return () => { seed |= 0; seed = (seed + 0x6d2b79f5) | 0; let t = Math.imul(seed ^ (seed >>> 15), 1 | seed);
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t; return ((t ^ (t >>> 14)) >>> 0) / 4294967296; };
}
const strategies = {
  'no jitter':   (attempt) => 100 * 2 ** attempt,
  'full jitter': (attempt, rand) => rand() * 100 * 2 ** attempt,
};
for (const [name, delay] of Object.entries(strategies)) {
  const rand = mulberry32(3);
  const buckets = new Map();                     // 10 ms slot → retries arriving in it
  for (let client = 0; client < 1_000; client++) {
    let t = 0;
    for (let attempt = 0; attempt < 4; attempt++) {
      t += delay(attempt, rand);
      const slot = Math.floor(t / 10);
      buckets.set(slot, (buckets.get(slot) ?? 0) + 1);
    }
  }
  console.log(`${name.padEnd(11)} peak retries in one 10 ms slot: ${Math.max(...buckets.values())}`);
}
// no jitter   peak retries in one 10 ms slot: 1000
// full jitter peak retries in one 10 ms slot: 162
```

With synchronised backoff, every retry wave hits the dependency as a single 1,000-request spike; full jitter
(`delay = random(0, base × 2^attempt)`) cuts the peak by more than 6×. Full jitter is a good default; "decorrelated
jitter" variants exist and behave similarly.

**Cap attempts and total time.** Two or three attempts, bounded by the caller's deadline — a retry that can't finish
before the deadline is wasted work.

**Retry in one layer only.** If the mobile app, the gateway, the BFF, the service, and the mesh each retry three times,
the bottom of the stack sees 3⁵ = 243 calls per user action. Pick one layer — usually the one closest to the failing
call, which knows whether the operation is idempotent — and make every other layer pass failures through.

**Use a retry budget.** Fixed retry counts still let retries double or triple load during an outage, precisely when the
dependency can least afford it. A **retry budget** caps retries as a fraction of normal traffic — say, retries may add
at most 10% to the request rate. During a blip, the budget covers the few failures; during an outage, it stops
retrying almost immediately, so the dependency sees roughly its normal load rather than a multiple. gRPC's retry
throttling and Envoy's retry budgets implement this.

> ⚠️ **Gotcha** — **A timeout plus a retry is a hedge you didn't intend.** If a request times out at 1 second and is
> retried, the original may still be executing on the server — so the dependency now does the work twice, and both
> copies compete for the capacity that made the first one slow. Under overload, timeouts and retries together *increase*
> work per user request. Cancel the original where the protocol allows, keep server-side deadlines consistent with
> client timeouts, and let retry budgets stop the multiplication.

---

## 28.4 Circuit Breakers

A **circuit breaker** stops calling a dependency that's failing, so callers fail fast instead of waiting for timeouts,
and the dependency gets breathing room to recover. It's a small state machine per dependency:

```text
          failure rate ≥ threshold                 cool-down elapsed
 CLOSED ───────────────────────────────▶ OPEN ───────────────────────▶ HALF-OPEN
 (calls pass; outcomes recorded)        (calls rejected immediately)   (a few probe calls allowed)
    ▲                                                                       │
    └──────────────────────── probes succeed ───────────────────────────────┤
                                       probes fail ──▶ back to OPEN ◀──────┘
```

Design choices that matter:

- **Trip on failure *rate* over a window**, with a minimum request count, not on a raw count of failures — five failures
  out of ten requests is a problem; five out of ten thousand isn't. Count timeouts as failures, and consider slow calls
  (above a latency threshold) as failures too, since slowness is what exhausts callers.
- **One breaker per dependency (and often per endpoint or per host)**, so one broken thing doesn't block calls to healthy
  ones.
- **Fail fast with a fallback**, not just an error: a cached value, a default, a degraded response (§28.6).
- **Half-open admits a trickle**, not a flood: one or a few probe requests decide whether to close.

A circuit breaker is the client-side complement of Chapter 7's outlier ejection: the balancer ejects one bad *instance*;
the breaker sheds calls to a whole *dependency* that's failing.

---

## 28.5 Bulkheads and Concurrency Limits

A ship's bulkheads stop one flooded compartment from sinking the ship. A software **bulkhead** stops one dependency from
consuming all of a service's capacity. Without one, a single slow dependency fills the shared thread pool or connection
pool, and every endpoint — even ones that never call it — stalls behind it.

The mechanism is simply a **separate, bounded concurrency limit per dependency** (or per class of work): at most 20
in-flight calls to the payment provider, 50 to the database, 10 to the recommendations service. When a limit is
reached, new calls are **rejected immediately** rather than queued — queueing just moves the waiting somewhere less
visible and lets it grow without bound (Chapter 3).

Choosing the limit is Little's law again: expected throughput × normal latency, plus headroom. A dependency called
200 times per second at 50 ms needs about 10 in flight; a limit of 20 absorbs normal variance and caps the damage when
latency explodes.

**Adaptive concurrency limits** go further: instead of a fixed number, the client measures latency and adjusts the limit
the way TCP adjusts its congestion window — additive increase while latency is normal, multiplicative decrease when it
rises (AIMD), or limits derived from the ratio of current to minimum observed latency. They find the dependency's real
capacity automatically and shrink when it degrades, which is exactly when fixed limits are wrong. Netflix's
concurrency-limits library and Envoy's adaptive concurrency filter are implementations.

Bulkheads also apply at larger scales: separate instance pools for critical and non-critical traffic (checkout servers
that never serve the reporting API), separate queues per tenant tier, and Chapter 4's cells.

---

## 28.6 Fallbacks and Graceful Degradation

When a dependency is unavailable — breaker open, bulkhead full, timeout — the caller needs an answer to give. Deciding it
in advance is product design, and it's what turns "the recommendations service is down" from an outage into a minor
degradation:

| Dependency | Fallback | User experience |
|---|---|---|
| Recommendations | Best-sellers from a cache, or hide the section | Slightly less relevant page |
| Reviews | Hide reviews, show the product | Page works |
| Personalised pricing | List price | Correct but not discounted (product decides if acceptable) |
| Search | Category browsing, or a cached result for popular queries | Reduced functionality |
| Payment provider | Queue the order as "payment pending" and retry asynchronously | Order accepted, confirmed later |
| Inventory check | Allow the order and reconcile, or refuse | A deliberate risk decision — oversell vs lost sale |

Principles for fallbacks:

- **Prefer stale over nothing** for read data — the soft/hard TTL cache from Chapter 16 is a fallback mechanism.
- **Make fallbacks cheap and local.** A fallback that calls another remote service can fail too, and adds load during an
  incident. A fallback that's itself expensive (a heavy database query) can overload the database.
- **Fail closed where safety demands it.** For authorization, fraud checks, and payments, an unavailable dependency
  usually means *refuse*, not *allow* — the fallback for "can't check permission" is never "grant permission".
- **Exercise fallbacks regularly.** A fallback path that's never used is broken by the time it's needed: wrong data,
  missing permissions, a cache nobody populates. Inject failures in tests and in production game days.

**Hedged requests** (Chapter 3) are a complementary tool for tail latency: send a duplicate after a delay and take the
first response. They belong to read-heavy, idempotent calls with independent replicas, and must be capped by the same
retry budget, or they become another amplifier.

---

## 28.7 Testing Resilience

Resilience mechanisms that aren't exercised don't work when needed: timeouts are set wrong, breakers never trip because
thresholds are too high, fallbacks reference data that's no longer there. Testing them is part of building them:

- **Fault injection in integration tests**: dependencies that time out, return 503s, return malformed data, or respond
  slowly. Assert the caller's latency stays bounded and the fallback is served.
- **Load tests with failures**: run at realistic load while a dependency degrades, and watch for the cascade signatures —
  pool exhaustion, retry storms, unrelated endpoints slowing.
- **Chaos experiments in production** (carefully, with a blast-radius limit): terminate instances, add latency to one
  dependency, fail a zone. Start with a hypothesis ("checkout p99 stays under 800 ms if recommendations fail"), observe,
  and stop automatically if SLOs burn too fast (Chapter 4).
- **Game days**: rehearse incident response with real failures injected, including the human parts — paging, runbooks,
  rollbacks.

---

## 28.8 Common Mistakes and Best Practices

**Mistake: calls without timeouts.** A slow dependency holds resources indefinitely, and pools exhaust.

**Mistake: retrying non-idempotent operations or permanent errors.** Duplicates on one hand, wasted load on the other.

**Mistake: retrying without backoff and jitter.** Synchronised retries hit a struggling dependency as repeated spikes.

**Mistake: retries at every layer.** Attempts multiply across the stack; 3 attempts at 5 layers is 243 calls.

**Mistake: fixed retry counts with no budget.** During an outage, retries double or triple load on the failing dependency.

**Mistake: one shared pool for all dependencies.** A single slow dependency stalls every endpoint.

**Mistake: fallbacks that are remote, expensive, or untested.** They fail during the incident they were meant to absorb.

**Best practice: deadlines from the edge, propagated down.** No hop works on a request its caller has abandoned.

**Best practice: breaker + bulkhead + budgeted retries per dependency.** Each bounds a different way failures spread.

**Best practice: decide fallbacks as product behaviour, and fail closed for security.** Degrade gracefully where it's
safe, refuse where it isn't.

---

## 28.9 Real-World Example: A Resilient Dependency Client

This wrapper combines the protections for one dependency: a bulkhead that rejects rather than queues when too many
calls are in flight, a circuit breaker on failure rate over a rolling window with a half-open probe, retries with full
jitter limited by a retry budget, and a deadline that caps every attempt by the caller's remaining time. The demo drives
it through a healthy period, a total outage, and recovery, with a simulated clock.

```js
// resilient-call.mjs — the four protections every call to a dependency needs, in one wrapper:
// a bulkhead (concurrency cap), a circuit breaker, retries with jittered backoff limited by a
// retry budget, and a deadline that bounds every attempt by the caller's remaining time.

export class Rejected extends Error {}                  // we chose not to try: fail fast

export function resilient(call, {
  maxConcurrent = 20, minRequests = 10, failureRate = 0.5, windowMs = 10_000, openMs = 5_000,
  maxAttempts = 3, baseDelayMs = 50, retryRatio = 0.1, perTryMs = 1_000,
  now = Date.now, rand = Math.random, sleep = (ms) => new Promise((r) => setTimeout(r, ms)),
} = {}) {
  let inFlight = 0;
  let state = 'closed', openedAt = 0, window = [];     // window: [{ at, ok }]
  let requests = 0, retries = 0;                       // for the retry budget

  const record = (ok) => {
    window.push({ at: now(), ok });
    window = window.filter((e) => now() - e.at < windowMs);
    const fails = window.filter((e) => !e.ok).length;
    if (state === 'half-open') { state = ok ? 'closed' : 'open'; if (!ok) openedAt = now(); if (ok) window = []; }
    else if (window.length >= minRequests && fails / window.length >= failureRate) { state = 'open'; openedAt = now(); }
  };

  return async function invoke(args, { deadline = now() + 2_000 } = {}) {
    if (state === 'open' && now() - openedAt >= openMs) state = 'half-open';   // let one probe through
    if (state === 'open') throw new Rejected('circuit open');
    if (inFlight >= maxConcurrent) throw new Rejected('bulkhead full');        // don't queue: shed
    inFlight++; requests++;
    try {
      for (let attempt = 1; ; attempt++) {
        const remaining = deadline - now();
        if (remaining <= 0) throw new Rejected('deadline exceeded');
        try {
          // Each attempt gets the smaller of its own timeout and what's left of the caller's budget
          const result = await call(args, AbortSignal.timeout(Math.min(perTryMs, remaining)));
          record(true);
          return result;
        } catch (err) {
          record(false);
          const retryable = err.retryable !== false && attempt < maxAttempts && state !== 'open';
          // Retry budget: retries may add at most `retryRatio` extra load on top of first attempts
          if (!retryable || retries >= requests * retryRatio) throw err;
          retries++;
          await sleep(Math.min(rand() * baseDelayMs * 2 ** attempt, deadline - now())); // full jitter
        }
      }
    } finally {
      inFlight--;
    }
  };
}

// ── Demo: a dependency that fails completely for a while, then recovers ──────────────
let t = 0, healthy = true, calls = 0;
const dep = async () => { calls++; if (!healthy) throw new Error('503'); return 'ok'; };
const call = resilient(dep, { windowMs: 500, now: () => t, rand: () => 0.5, sleep: async (ms) => { t += ms; } });
const outcomes = async (n) => {
  const tally = {};
  for (let i = 0; i < n; i++) {
    t += 10;
    const r = await call({}).then(() => 'ok', (e) => (e instanceof Rejected ? e.message : 'failed'));
    tally[r] = (tally[r] ?? 0) + 1;
  }
  return tally;
};
console.log('healthy:   ', await outcomes(100), '| dependency calls:', calls);
healthy = false; calls = 0;
console.log('outage:    ', await outcomes(100), '| dependency calls:', calls);
healthy = true; t += 6_000; calls = 0;
console.log('recovered: ', await outcomes(100), '| dependency calls:', calls);
// healthy:    { ok: 100 } | dependency calls: 100
// outage:     { failed: 3, 'circuit open': 97 } | dependency calls: 9
// recovered:  { ok: 100 } | dependency calls: 100
```

During the outage, the first three requests each try three times (the budget allows those six retries, since 100
healthy requests came before), the failure rate in the window crosses 50%, and the breaker opens: the remaining 97
requests fail in microseconds without touching the dependency, which sees 9 calls instead of the 300 that fixed
three-attempt retries would have sent. After the cool-down, one probe succeeds and the breaker closes.

The decision to defend is **a retry budget proportional to traffic instead of simply "retry up to 3 times"**. A fixed
count is easier to explain and handles the common case — an occasional transient failure — perfectly. But its behaviour
in the uncommon case is exactly backwards: when the dependency is failing for everyone, every request uses all three
attempts, and the dependency receives three times its normal load at the moment it's least able to handle it, which
prolongs the outage and can take down its other callers too. A budget keeps retries at a bounded fraction of real
traffic: generous when failures are rare (when retries actually help) and nearly zero when failures are widespread
(when retries only hurt). It's the difference between retries as a recovery mechanism and retries as an amplifier.

---

## 28.10 Questions

**1.** MCQ — A dependency's latency rises from 20 ms to 2 s while request volume stays the same. What happens to the caller's in-flight requests to it? a) Unchanged b) Doubles c) Rises about 100× d) Falls
<details><summary>Answer</summary>**c)** Little's law: in-flight = rate × latency, so 100× the latency is 100× the held resources.</details>

**2.** MCQ — Each of four layers retries failed calls up to 3 times (3 attempts). How many calls can one user request cause at the bottom layer? a) 4 b) 12 c) 81 d) 3
<details><summary>Answer</summary>**c)** 3⁴ = 81 — attempts multiply across layers.</details>

**3.** MCQ — What's the purpose of jitter in backoff? a) Faster recovery b) Preventing clients that failed together from retrying in synchronised waves c) Reducing timeouts d) Encrypting retries
<details><summary>Answer</summary>**b)** Randomised delays spread retries over time.</details>

**4.** MCQ — What does a circuit breaker in the half-open state do? a) Rejects every call b) Allows every call c) Lets a small number of probe calls through to test recovery d) Restarts the dependency
<details><summary>Answer</summary>**c)** Probe outcomes decide whether to close or reopen.</details>

**5.** MCQ — What's the main purpose of a bulkhead? a) Encrypting traffic b) Preventing one dependency from consuming all of a service's concurrency c) Caching responses d) Load balancing
<details><summary>Answer</summary>**b)** Separate, bounded limits per dependency keep one slow dependency from stalling everything.</details>

**6.** MCQ — An authorization service is unreachable. What's the right fallback for a permission check? a) Allow the request b) Deny the request (fail closed) c) Use a random decision d) Retry forever
<details><summary>Answer</summary>**b)** Security-critical checks must fail closed; a cached recent decision with a short TTL is a possible middle ground only if the product accepts it.</details>

**7.** Short — Why is deadline propagation better than independent per-hop timeouts?
<details><summary>Answer</summary>Independent timeouts let downstream hops keep working after the user's request has already timed out at the edge. Passing the remaining budget makes every hop stop when the overall deadline passes, so no work is wasted on abandoned requests.</details>

**8.** Short — Why should a bulkhead reject excess calls rather than queue them?
<details><summary>Answer</summary>Queueing just relocates the waiting: the queue grows without bound when the dependency is slow, latency climbs for everyone, and memory fills. Rejecting immediately fails fast, lets callers use a fallback, and keeps latency bounded.</details>

**9.** Short — How does a retry budget behave differently from a fixed retry count during an outage?
<details><summary>Answer</summary>A fixed count makes every failing request retry, multiplying load on the failing dependency. A budget caps retries as a fraction of traffic, so during widespread failure retries quickly stop and the dependency sees roughly normal load.</details>

**10.** Short — Why should a circuit breaker trip on failure rate with a minimum request count rather than on a fixed number of failures?
<details><summary>Answer</summary>A fixed count trips on noise at high volume (5 failures in 10,000 requests) and too slowly at low volume. A rate over a window with a minimum sample reflects how unhealthy the dependency actually is.</details>

**11.** Short — What are adaptive concurrency limits, and why are they better than fixed limits?
<details><summary>Answer</summary>Limits adjusted from observed latency — growing while latency is normal and shrinking when it rises (AIMD or gradient-based). They track the dependency's real capacity, shrinking automatically when it degrades, where a fixed limit is either too low normally or too high during trouble.</details>

**12.** Output — Using §28.9's `resilient` with default settings, a caller passes `{ deadline: now() + 300 }`, and the dependency hangs. What timeout does the first attempt get?
<details><summary>Answer</summary>**300 ms** — `Math.min(perTryMs = 1000, remaining = 300)`; the per-attempt timeout never exceeds the caller's remaining budget.</details>

**13.** Output — What does this print?
```js
const fullJitter = (attempt, base, r) => Math.floor(r * base * 2 ** attempt);
console.log([0, 1, 2, 3].map((a) => fullJitter(a, 100, 0.25)));
```
<details><summary>Answer</summary>`[ 25, 50, 100, 200 ]` — a quarter of the exponentially growing ceiling each time.</details>

**14.** Output — A service receives 2,000 requests/s; its retry budget allows retries up to 10% of requests. During a full outage of its dependency, at most how many calls per second reach the dependency?
<details><summary>Answer</summary>**~2,200** (first attempts plus at most 200 retries) — versus 6,000 with three fixed attempts. With a circuit breaker open, it falls to near zero plus probes.</details>

**15.** Output — A dependency is called 400 times/s with a normal latency of 25 ms. What's the in-flight average, and a sensible bulkhead limit with 2× headroom?
<details><summary>Answer</summary>**10 in flight; a limit of about 20.** 400 × 0.025 = 10 by Little's law.</details>

**16.** Debug — When the recommendations service slows down, the whole product site becomes slow, including checkout, which never calls recommendations. All requests share one HTTP client pool of 200 connections. Cause and fix?
<details><summary>Answer</summary>Slow recommendation calls fill the shared pool and every other outbound call waits for a connection. Give each dependency its own bounded pool (a bulkhead), add a tight timeout and a breaker on recommendations, and make it optional with a cached fallback.</details>

**17.** Debug — A 30-second database failover caused a 20-minute outage. Metrics show request volume to the database tripling right after the failover completed, and staying high. What happened?
<details><summary>Answer</summary>Synchronised retries without jitter or budget: clients (often at several layers) retried in waves, tripling load on a freshly promoted, cold primary, which slowed, causing more timeouts and retries. Add full jitter, a single retry layer, a retry budget, circuit breakers, and slow-start/warm-up for the new primary.</details>

**18.** Debug — A circuit breaker on a payment client never opens during payment-provider incidents, though users see errors. Its configuration trips after 50 consecutive failures. Why doesn't it trip?
<details><summary>Answer</summary>Partial failures interleave successes with failures, so 50 *consecutive* failures rarely occur. Trip on failure rate over a rolling window with a minimum request count, and count timeouts and slow calls as failures.</details>

**19.** Debug — During an incident, the "fallback to cached prices" path threw errors because the cache had been decommissioned months earlier; nobody noticed since the fallback had never run. What process change prevents this?
<details><summary>Answer</summary>Exercise fallbacks continuously: inject dependency failures in integration tests and periodic production game days, alert when a fallback errors, and treat fallback paths as code that's tested and owned like any other.</details>

**20.** Code — Write `remainingBudget(headers, { now, defaultMs, maxMs })` that reads an incoming `x-deadline-ms` header (an absolute epoch-ms deadline), returns milliseconds remaining, falls back to `defaultMs`, and caps at `maxMs`.
<details><summary>Answer</summary>

```js
function remainingBudget(headers, { now = Date.now(), defaultMs = 1_000, maxMs = 5_000 } = {}) {
  const deadline = Number(headers['x-deadline-ms']);
  const remaining = Number.isFinite(deadline) ? deadline - now : defaultMs;
  return Math.max(0, Math.min(remaining, maxMs)); // never trust a caller's budget beyond our own cap
}
console.log(remainingBudget({ 'x-deadline-ms': '10750' }, { now: 10_000 }), remainingBudget({}, { now: 0 })); // 750 1000
```
Absolute deadlines assume roughly synchronised clocks; propagating a relative "timeout remaining" avoids that at the cost of not accounting for transit time.</details>

**21.** Code — Write `aimdLimit({ limit, latencyMs, targetMs, min, max })` returning the next concurrency limit: +1 when latency is at or under target, ×0.8 (rounded down) when it's over.
<details><summary>Answer</summary>

```js
function aimdLimit({ limit, latencyMs, targetMs, min = 1, max = 1_000 }) {
  const next = latencyMs <= targetMs ? limit + 1 : Math.floor(limit * 0.8); // additive increase, multiplicative decrease
  return Math.max(min, Math.min(max, next));
}
let limit = 20;
for (const lat of [30, 30, 30, 90, 90, 30]) limit = aimdLimit({ limit, latencyMs: lat, targetMs: 50 });
console.log(limit); // 15
```
</details>

**22.** Code — Write `withFallback(primary, fallback, { timeoutMs })` that returns the primary's result, or the fallback's if the primary throws or exceeds the timeout, marking which was used.
<details><summary>Answer</summary>

```js
async function withFallback(primary, fallback, { timeoutMs = 200 } = {}) {
  let timer;
  const timeout = new Promise((_, rej) => { timer = setTimeout(() => rej(new Error('timeout')), timeoutMs); });
  try {
    return { value: await Promise.race([primary(), timeout]), degraded: false };
  } catch {
    return { value: await fallback(), degraded: true };    // a cheap, local fallback — not another remote call
  } finally {
    clearTimeout(timer);
  }
}
const slow = () => new Promise((r) => setTimeout(() => r('personalised'), 500));
console.log(await withFallback(slow, async () => 'bestsellers', { timeoutMs: 50 })); // { value: 'bestsellers', degraded: true }
```
Report `degraded` in metrics so fallbacks in use are visible.</details>

**23.** Scenario — Checkout depends on payments, fraud scoring, inventory, tax calculation, and recommendations (for an upsell widget). Design the resilience policy for each.
<details><summary>Answer</summary>Payments: tight timeout, idempotency keys, budgeted retries for transient errors, a breaker, and a fallback of accepting the order as "payment pending" with async capture if the business allows; otherwise a clear retry message. Fraud: timeout and breaker; fail closed above a risk or amount threshold, and queue low-risk orders for post-hoc review. Inventory: bounded call with breaker; on failure, the product decides oversell risk vs refusing. Tax: cache rates by region; fall back to the cached rate and reconcile. Recommendations: optional, short timeout, breaker, hide the widget. Separate bulkheads for each, one retry layer, and a deadline propagated from the edge.</details>

**24.** Scenario — A team proposes adding retries to every service-to-service call in the mesh "for reliability", configured at 3 attempts with 100 ms fixed delay. What's your response?
<details><summary>Answer</summary>Retries at the mesh, on top of existing application retries, multiply attempts; fixed delays synchronise retry waves; and blanket retries hit non-idempotent calls. Instead: retry in one layer (often the mesh, if applications stop retrying, or vice versa), only for idempotent methods and transient errors, with exponential backoff and full jitter, a retry budget (e.g. 10–20% of traffic), and a cap within the caller's deadline. Pair with circuit breakers and per-route timeouts.</details>

---

## Chapter Summary

- Cascades come from resources exhausted while waiting, retries multiplying load, and feedback loops between timeouts,
  retries, health checks, and autoscaling; every pattern here bounds one of them.
- Every call needs connect and request timeouts set from the latency budget, and deadlines should be propagated so no
  hop works on an abandoned request.
- Retry only idempotent operations and transient errors, with exponential backoff and full jitter, in exactly one layer,
  within the deadline, and under a retry budget.
- Circuit breakers fail fast on high failure rates over a window and probe for recovery; bulkheads give each dependency a
  bounded, rejecting concurrency limit, ideally adaptive.
- Fallbacks are product decisions made in advance — stale over nothing for reads, fail closed for security — and must be
  cheap, local, and exercised.
- Test resilience with fault injection, failure-laden load tests, chaos experiments, and game days, because untested
  mechanisms fail when needed.

---

## Practice Exercises

1. Audit every outbound call in a service you own for connect timeout, request timeout, retry policy, and pool; record any
   that are missing or unbounded.
2. Run §28.3's simulation with decorrelated jitter (`delay = min(cap, random(base, prev × 3))`) and compare peaks and
   total retry time with full jitter.
3. Read Envoy's documentation on retry budgets and circuit breaking (or your mesh's equivalent) and configure them for one
   route in a test environment.
4. Inject 2-second latency into one dependency of a service under load and observe which endpoints slow down; add a
   bulkhead and repeat.
5. Implement deadline propagation across three toy services over HTTP and show that work stops downstream once the edge
   deadline passes.

---

## Mini Project: A Resilience Test Bench

Build a test bench with a caller service and three dependencies whose behaviour (latency distribution, error rate,
outage windows) can be scripted, and use it to show the effect of each pattern in this chapter.

- Scenario scripts: a slow dependency, an error-rate spike, a full outage, and a flapping dependency.
- The caller with switchable protections: timeouts, deadline propagation, retries (fixed vs jittered vs budgeted),
  circuit breakers, bulkheads (fixed and adaptive), and fallbacks.
- Metrics per run: caller p50/p99, error rate, dependency request volume (to show amplification), and pool usage.
- A results table comparing "no protections" with each protection added cumulatively for every scenario.
- A chaos mode that randomly combines scenarios for a soak test, with SLO-based pass/fail criteria.

---

**Next:** [Chapter 29 — Rate Limiting and Load Shedding →](29-rate-limiting-load-shedding.md)
