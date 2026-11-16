# Chapter 29 — Rate Limiting and Load Shedding

## Learning Objectives

- Distinguish rate limiting (per client, for fairness) from load shedding (global, for survival)
- Compare fixed window, sliding log, sliding window counter, token bucket, and GCRA limiters and choose one
- Implement a distributed rate limiter that's atomic under concurrency, and decide when local approximation is enough
- Communicate limits to clients with status codes and headers that produce good retry behaviour
- Shed load by priority, cheaply and early, so overload degrades the least important work first

---

## 29.1 Saying No on Purpose

Every system has a capacity, and Chapter 3 showed what happens past it: queues grow, latency explodes, and
eventually everything times out — including the requests that would have succeeded. The alternative is to **refuse
some work deliberately**, early and cheaply, so the rest gets served. Two related mechanisms do this:

| | Rate limiting | Load shedding |
|---|---|---|
| Question it answers | "Is *this client* using more than its share?" | "Is *the system* at capacity right now?" |
| Scope | Per client, API key, tenant, IP, or user | Global, per instance, or per dependency |
| Trigger | A configured limit (100 requests/min) | Measured overload (concurrency, queue time, latency) |
| Protects | Fairness, cost, abuse prevention, downstream quotas | Availability for everyone during overload |
| Typical response | `429 Too Many Requests` with `Retry-After` | `503 Service Unavailable`, or a degraded response |

They complement each other. Rate limits stop a single runaway client — a buggy script in a loop, a scraper, a tenant
whose integration retries without backoff — from consuming everyone's capacity. But a limit set per client can't
protect against *many* clients at once: if 10,000 well-behaved clients each stay under their limit during a traffic
spike, the system can still be overloaded. Load shedding handles that.

Where limits live matters too:

- **At the edge** (CDN, WAF, API gateway): cheapest rejection, protects everything behind it; coarse keys (IP, API key).
- **In the service**: finer-grained keys (user, tenant, endpoint), business-aware limits ("5 password attempts per
  account per 15 minutes").
- **Before a dependency**: limit calls to a third-party API that enforces its own quota, so you spend your quota
  deliberately rather than hitting its limit and failing unpredictably.

---

## 29.2 Rate-Limiting Algorithms

All limiters answer "may this request proceed?" by tracking recent usage per key. They differ in how they count,
which shows up in burst behaviour and memory cost.

**Fixed window** counts requests per key per calendar window (per minute) and rejects once the count exceeds the
limit. It's a single counter per key — trivially cheap, usually `INCR` plus `EXPIRE` in Redis — but it allows up to
**twice the limit** in a short burst straddling a window boundary: a full window's worth at 0:59 and another at 1:00.

**Sliding log** stores the timestamp of every request in the last window and counts them. It's exact, but stores one
entry per request — expensive for high limits.

**Sliding window counter** approximates the sliding log with two counters: the current window's count plus the
previous window's count weighted by how much of it still overlaps the sliding window. Cheap and close to exact.

**Token bucket** gives each key a bucket of up to *b* tokens, refilled at *r* tokens per second; each request takes a
token, and requests without a token are rejected. It allows short bursts (up to *b*) while enforcing the long-run
rate *r* — usually exactly what's wanted. A **leaky bucket** is its mirror image (requests drain from a queue at a
fixed rate), smoothing output rather than allowing bursts.

**GCRA** (generic cell rate algorithm) is a token bucket expressed as a single timestamp per key — the "theoretical
arrival time" of the next request — which makes it compact and easy to update atomically. The real-world example in
§29.8 implements it.

The burst behaviour is easiest to see by giving each algorithm a limit of 10 requests per second and the same burst:
10 requests just before a window boundary and 10 just after.

```js
// Four classic limiters, each allowing "10 requests per second", fed the same burst:
// 10 requests at t = 0.9 s and 10 more at t = 1.1 s — 20 requests in 200 ms.
const fixedWindow = (limit, windowMs) => {
  const counts = new Map();
  return (t) => { const w = Math.floor(t / windowMs); const c = (counts.get(w) ?? 0) + 1; counts.set(w, c); return c <= limit; };
};
const slidingLog = (limit, windowMs) => {
  const log = [];
  return (t) => { while (log.length && log[0] <= t - windowMs) log.shift(); if (log.length >= limit) return false; log.push(t); return true; };
};
const slidingWindowCounter = (limit, windowMs) => {
  const counts = new Map();
  return (t) => {
    const w = Math.floor(t / windowMs), elapsed = (t % windowMs) / windowMs;
    // Weight the previous window by how much of it still overlaps the sliding window
    const estimate = (counts.get(w - 1) ?? 0) * (1 - elapsed) + (counts.get(w) ?? 0);
    if (estimate >= limit) return false;
    counts.set(w, (counts.get(w) ?? 0) + 1);
    return true;
  };
};
const tokenBucket = (ratePerSec, burst) => {
  let tokens = burst, last = 0;
  return (t) => {
    tokens = Math.min(burst, tokens + ((t - last) / 1000) * ratePerSec); last = t;  // refill continuously
    if (tokens < 1) return false;
    tokens -= 1; return true;
  };
};

const burst = [...Array(10).fill(900), ...Array(10).fill(1100)];
for (const [name, make] of Object.entries({
  'fixed window': () => fixedWindow(10, 1000),
  'sliding log': () => slidingLog(10, 1000),
  'sliding counter': () => slidingWindowCounter(10, 1000),
  'token bucket': () => tokenBucket(10, 10),
})) {
  const allow = make();
  console.log(`${name.padEnd(16)} allowed ${burst.filter((t) => allow(t)).length} of 20 within 200 ms`);
}
// fixed window     allowed 20 of 20 within 200 ms
// sliding log      allowed 10 of 20 within 200 ms
// sliding counter  allowed 11 of 20 within 200 ms
// token bucket     allowed 12 of 20 within 200 ms
```

The fixed window lets the whole double burst through; the sliding log enforces exactly 10 per rolling second; the
sliding counter is within one request of exact; the token bucket allows its burst of 10 plus the 2 tokens refilled in
the following 200 ms.

| Algorithm | State per key | Burst at boundary | Accuracy | Good for |
|---|---|---|---|---|
| Fixed window | 1 counter | Up to 2× limit | Coarse | Coarse quotas, cheap edge limits |
| Sliding log | 1 entry per request | None | Exact | Low limits needing exactness (login attempts) |
| Sliding window counter | 2 counters | Small | Approximate | General API limits at scale |
| Token bucket / GCRA | 2 numbers / 1 timestamp | Up to bucket size, by design | Exact for rate + burst | Most APIs: allows bursts, enforces rate |
| Leaky bucket | Queue | Smoothed | Exact output rate | Shaping traffic to a fixed-rate downstream |

> 💼 **Interview** — "Design a rate limiter" is a staple. A complete answer chooses an algorithm (token bucket, with
> its burst parameter), stores state in a shared store with atomic updates (Redis Lua script or GCRA), keys by API key
> or user, returns `429` with `Retry-After` and rate-limit headers, fails open or closed deliberately when the store
> is down, and discusses the local-versus-global accuracy trade-off.

---

## 29.3 Distributed Rate Limiting

With one server, a limiter is an in-memory map. With fifty, a client's requests are spread across instances, and each
instance sees only a fiftieth of them. Three approaches:

**Central store.** Every instance checks and updates a shared counter (Redis is the usual choice). Accurate, but adds a
network round trip to every request, makes the store a dependency, and creates a hot key for heavy clients. The check
**must be atomic**: "read the counter, decide, write it back" as separate commands lets concurrent requests all read
the same value and all pass. Use a single atomic operation — `INCR` for fixed windows, or a Lua script (or GCRA's single
compare-and-set) for token buckets.

**Local limits.** Each instance enforces `limit ÷ N` locally. No network call and no dependency, but it's only accurate
if load balancing spreads each client evenly, and N changes as instances scale. Fine for coarse protection, poor for
precise quotas or small limits (a limit of 5 per minute across 50 instances is 0.1 each).

**Hybrid.** Instances enforce locally and synchronise with the central store periodically (every few hundred
milliseconds), or reserve batches of tokens from it ("give me 50 tokens for user 42"). Most of the accuracy, a fraction
of the round trips, at the cost of some overshoot between syncs.

Two design decisions that are easy to forget:

- **What happens when the limiter's store is unavailable?** Failing *open* (allow everything) protects availability but
  removes protection exactly when something is wrong; failing *closed* (reject everything) turns a Redis outage into a
  full outage. Most systems fail open for general API limits, with a local fallback limiter, and fail closed for
  security limits such as login attempts.
- **Which key?** IP addresses are shared (corporate NAT, mobile carriers put thousands of users behind one address) and
  cheap to rotate for attackers; API keys and user IDs are better for authenticated traffic. Abuse protection often
  layers several keys: per IP, per account, and per target (per email address being tried).

---

## 29.4 Telling Clients About Limits

A limit is only useful if well-behaved clients can respect it. Responses should make the right behaviour obvious:

- **`429 Too Many Requests`** for rate limits, with **`Retry-After`** (seconds, or an HTTP date) saying when to try
  again. Clients — and your own SDKs — should honour it instead of retrying immediately (Chapter 28).
- **Rate-limit headers** on *every* response, not just rejections, so clients can slow down before they hit the limit.
  The IETF has standardised `RateLimit` and `RateLimit-Policy` fields (successors to the widely used
  `X-RateLimit-Limit/Remaining/Reset` headers): remaining quota, the window, and when it resets.
- **`503 Service Unavailable`** (with `Retry-After` if known) for load shedding — it's not the client's fault.
- **Consistent semantics across the API**: limits documented per endpoint and per plan, the same headers everywhere.

On the client side, a good SDK reads the headers, paces itself, honours `Retry-After`, and backs off with jitter — so
the limit shapes traffic smoothly instead of producing waves of rejections and retries.

---

## 29.5 Load Shedding

Rate limits protect against individual clients; **load shedding** protects the system when total demand exceeds total
capacity. Its principles follow from Chapter 3's queueing behaviour:

**Detect overload directly.** CPU percentage is a poor signal. Better signals are the ones that predict latency:
**in-flight concurrency** above what the service handles well, **time requests spend queued** before work starts, or
latency of recent requests above target. A service that knows its comfortable concurrency (Little's law: throughput ×
latency) can reject request number N+1 instantly.

**Reject early and cheaply.** A rejection should cost almost nothing — no authentication call, no database query, no
parsing a large body. Rejecting at the load balancer or at the very start of the request pipeline keeps capacity for
the work you accept. An expensive rejection path is itself a way to fall over.

**Shed by priority.** Not all requests matter equally. A checkout request matters more than a "recently viewed" widget;
a user-facing read matters more than a background sync; a paying tier may matter more than a free one. Tag requests
with a **criticality** (at the edge, from the route, client, or a header), and when shedding, drop the least critical
first:

```js
// A priority load shedder: bounded concurrency, a bounded queue served by priority, and
// rejection of work that has waited longer than it could still be useful.
const PRIORITY = { critical: 0, normal: 1, sheddable: 2 };

class Shedder {
  #inFlight = 0; #queue = [];
  constructor({ maxConcurrent, maxQueue, maxQueueMs, now = () => performance.now() }) {
    Object.assign(this, { maxConcurrent, maxQueue, maxQueueMs, now });
  }
  run(priority, work) {
    if (this.#inFlight < this.maxConcurrent) return this.#start(work);
    if (this.#queue.length >= this.maxQueue) {
      // Full: evict the least important queued item if the newcomer outranks it, else reject
      const worst = this.#queue.reduce((a, b) => (PRIORITY[b.priority] > PRIORITY[a.priority] ? b : a));
      if (PRIORITY[worst.priority] <= PRIORITY[priority]) return Promise.reject(new Error(`shed:${priority}`));
      this.#queue.splice(this.#queue.indexOf(worst), 1);
      worst.reject(new Error(`shed:${worst.priority}`));
    }
    return new Promise((resolve, reject) => this.#queue.push({ priority, work, resolve, reject, at: this.now() }));
  }
  #start(work) {
    this.#inFlight++;
    return Promise.resolve().then(work).finally(() => { this.#inFlight--; this.#next(); });
  }
  #next() {
    this.#queue.sort((a, b) => PRIORITY[a.priority] - PRIORITY[b.priority] || a.at - b.at);
    while (this.#queue.length && this.#inFlight < this.maxConcurrent) {
      const item = this.#queue.shift();
      // Work that waited past its usefulness is dropped: its caller has likely timed out already
      if (this.now() - item.at > this.maxQueueMs) { item.reject(new Error(`expired:${item.priority}`)); continue; }
      this.#start(item.work).then(item.resolve, item.reject);
    }
  }
}

const shedder = new Shedder({ maxConcurrent: 4, maxQueue: 6, maxQueueMs: 1_000 });
const kinds = ['sheddable', 'normal', 'critical'];
const results = await Promise.allSettled(Array.from({ length: 30 }, (_, i) =>
  shedder.run(kinds[i % 3], () => new Promise((r) => setTimeout(() => r(kinds[i % 3]), 20)))));
const tally = {};
for (const r of results) {
  const k = r.status === 'fulfilled' ? `served ${r.value}` : r.reason.message.replace(':', ' ');
  tally[k] = (tally[k] ?? 0) + 1;
}
for (const k of Object.keys(tally).sort()) console.log(k.padEnd(17), tally[k]);
// served critical   7
// served normal     1
// served sheddable  2
// shed critical     3
// shed normal       9
// shed sheddable    8
```

Thirty requests arrive at once for ten slots (four running, six queued). The first four run whatever their priority;
after that, critical requests displace lower-priority ones in the queue, so 7 of 10 critical requests are served
against 1 of 10 normal — while the queue's bound keeps latency for accepted work predictable.

**Don't do work for requests nobody is waiting for.** A request that has sat in a queue past its caller's deadline will
be thrown away by the caller anyway; serving it wastes capacity during the overload. Drop requests that have waited too
long (as above), honour propagated deadlines (Chapter 28), and under severe overload consider serving the queue **LIFO**
rather than FIFO — newest requests are the ones whose callers are still waiting.

**Shed at every tier, not just the front.** Each service protects itself, because a service in the middle of the call
graph can be overloaded by a retry storm or a fan-out from one caller while the edge looks fine.

> ⚠️ **Gotcha** — **Autoscaling is not load shedding.** New instances take minutes to launch and warm up (Chapter 7's
> slow start), while overload collapses a service in seconds. Without shedding, the fleet falls over before the new
> capacity arrives — and then the new instances, cold, receive the full backlog. Shed to survive the next minute;
> autoscale to stop needing to.

---

## 29.6 Quotas, Fairness, and Multi-Tenancy

In multi-tenant systems, limits are also a product feature and a fairness mechanism:

- **Tiered quotas** — free, pro, enterprise — expressed as rate limits and as longer-term quotas (requests per month,
  rows exported per day), with the long-term ones tracked durably and the short-term ones in fast storage.
- **Per-tenant concurrency limits** stop one tenant's slow, expensive queries from occupying all workers — a bulkhead
  per tenant (Chapter 28).
- **Fair queueing**: when shared capacity is contended, serve tenants in proportion to their weight rather than first
  come, first served. Weighted fair queueing — round-robin across per-tenant queues, taking more from heavier-weighted
  ones — prevents a tenant that sends 10,000 jobs from starving one that sends ten.
- **Cost-based limits**: requests aren't equal. A GraphQL query or a search with heavy aggregations can cost 1,000 times
  a simple read. Charge each request its estimated cost against the bucket (token buckets support `cost` naturally), as
  public GraphQL APIs do with query cost analysis.

---

## 29.7 Common Mistakes and Best Practices

**Mistake: fixed windows where bursts matter.** Clients can send twice the limit around every window boundary.

**Mistake: non-atomic distributed counters.** Read-then-write across round trips lets concurrent requests all pass;
use atomic increments, Lua scripts, or compare-and-set.

**Mistake: limiting only by IP address.** Shared NATs penalise innocent users, and attackers rotate addresses; prefer
API keys and user IDs, layered with IP limits.

**Mistake: rejections without `Retry-After` or rate-limit headers.** Clients guess, retry immediately, and turn limits
into retry storms.

**Mistake: per-client limits as the only overload protection.** Many clients under their limits can still overload the
system; shed load globally too.

**Mistake: expensive rejection paths.** Authenticating, parsing, and querying before deciding to reject spends the
capacity you're trying to protect.

**Mistake: relying on autoscaling for spikes.** New capacity arrives minutes after overload collapses the service.

**Best practice: token bucket (or GCRA) as the default limiter.** It enforces a rate while allowing a deliberate burst,
with tiny state.

**Best practice: decide failure behaviour of the limiter itself.** Fail open with a local fallback for general limits,
fail closed for security limits.

**Best practice: tag criticality at the edge and shed by it everywhere.** Overload then degrades the least important
work first, at every tier.

---

## 29.8 Real-World Example: A GCRA Rate Limiter

GCRA is a token bucket that stores a single number per key — the **theoretical arrival time** (TAT) at which the bucket
would next be empty — which makes it ideal for a shared store: one read-modify-write of one value, performed atomically.
This implementation returns the decision plus standard headers, so clients can pace themselves.

```js
// gcra.mjs — a distributed rate limiter using the Generic Cell Rate Algorithm: a token bucket
// that stores ONE number per key (the "theoretical arrival time"), updated atomically in a
// shared store. Returns standard rate-limit headers so clients know when to come back.

export function createLimiter(store, { rate, periodMs, burst, now = Date.now }) {
  const interval = periodMs / rate;            // time "cost" of one request
  const tolerance = interval * (burst - 1);    // how far ahead of schedule a client may run

  return async function check(key, cost = 1) {
    // store.update runs the function atomically per key (in Redis: a Lua script; in a DB: one
    // conditional UPDATE). Read-modify-write across two round trips would let racing requests
    // both see the old value and both pass.
    return store.update(`rl:${key}`, (tat) => {
      const t = now();
      const base = Math.max(tat ?? t, t);      // a key idle for a while starts from "now"
      const newTat = base + interval * cost;
      const allowAt = newTat - tolerance - interval;   // earliest time this request fits
      if (t < allowAt) {
        return { value: tat, result: { allowed: false, retryAfterMs: Math.ceil(allowAt - t),
          headers: { 'RateLimit': `limit=${burst}, remaining=0, reset=${Math.ceil((tat - t) / 1000)}`,
                     'Retry-After': String(Math.ceil((allowAt - t) / 1000)) } } };
      }
      const remaining = Math.floor((tolerance - (newTat - t - interval)) / interval);
      // ttlMs lets the store expire idle keys (Redis: PX), so memory tracks only active clients
      return { value: newTat, ttlMs: newTat - t, result: { allowed: true,
        headers: { 'RateLimit': `limit=${burst}, remaining=${Math.max(0, remaining)}, reset=${Math.ceil((newTat - t) / 1000)}` } } };
    });
  };
}

// An in-memory store with the one primitive the limiter needs: atomic per-key update
function memoryStore() {
  const m = new Map();
  return { async update(k, fn) { const { value, result } = fn(m.get(k)); m.set(k, value); return result; } };
}

// ── Demo: 5 requests per second, bursts of up to 5 ─────────────────────────────────────
let t = 0;
const check = createLimiter(memoryStore(), { rate: 5, periodMs: 1_000, burst: 5, now: () => t });
const line = [];
for (let i = 0; i < 7; i++) { const r = await check('user:42'); line.push(r.allowed ? 'ok' : `429(${r.retryAfterMs}ms)`); }
console.log('burst of 7:   ', line.join(' '));
t += 400;                                       // 400 ms later: 2 requests' worth has refilled
const after = [];
for (let i = 0; i < 3; i++) after.push((await check('user:42')).allowed ? 'ok' : '429');
console.log('after 400 ms: ', after.join(' '));
t += 5_000;
console.log('after idle:   ', (await check('user:42')).headers.RateLimit);
// burst of 7:    ok ok ok ok ok 429(200ms) 429(200ms)
// after 400 ms:  ok ok 429
// after idle:    limit=5, remaining=4, reset=1
```

The client gets its burst of five, is told to come back in 200 ms (one interval), gets exactly the two requests that
refilled in 400 ms, and after a long idle period starts again with a full burst.

The decision to defend is **storing a single timestamp per key instead of a token count and a last-refill time**, the
textbook token bucket. The two-field version works, but updating it means reading both fields, computing the refill,
and writing both back — and in a shared store that must be done atomically, typically with a script that holds a lock
on the key. GCRA folds the state into one number whose update is a pure function of that number and the current time,
so any store with an atomic compare-and-set (or a short script) can apply it, the value is naturally expirable (it's in
the past once the bucket is full again), and memory per active client is eight bytes. It's the same rate and burst
behaviour with half the state and a much simpler atomicity story — which is why API gateways and Redis rate-limiting
modules commonly use it.

---

## 29.9 Questions

**1.** MCQ — What's the main weakness of a fixed-window rate limiter? a) It needs a lot of memory b) It can allow up to twice the limit in a burst across a window boundary c) It can't be distributed d) It never resets
<details><summary>Answer</summary>**b)** A full window's quota just before the boundary and another just after.</details>

**2.** MCQ — Which algorithm allows a configured burst while enforcing a long-run rate? a) Fixed window b) Sliding log c) Token bucket d) Leaky bucket
<details><summary>Answer</summary>**c)** The bucket size is the burst; the refill rate is the long-run rate. (A leaky bucket smooths output instead.)</details>

**3.** MCQ — Why must a distributed limiter's check-and-update be atomic? a) For speed b) Otherwise concurrent requests can all read the same count and all pass c) Redis requires it d) To save memory
<details><summary>Answer</summary>**b)** A non-atomic read-modify-write races, letting bursts exceed the limit.</details>

**4.** MCQ — What status code fits load shedding because the server is overloaded? a) `400` b) `403` c) `429` d) `503`
<details><summary>Answer</summary>**d)** Overload isn't the client's fault; `429` is for a client exceeding its own limit.</details>

**5.** MCQ — Which is the best signal for load shedding? a) CPU percentage b) In-flight concurrency or time spent queued before work starts c) Number of instances d) Request size
<details><summary>Answer</summary>**b)** They predict latency directly, where CPU is a lagging and indirect indicator.</details>

**6.** MCQ — Why can limiting by IP address alone be unfair? a) IPs change slowly b) Many users can share one IP behind NAT or carrier gateways c) IPv6 is unsupported d) IPs are encrypted
<details><summary>Answer</summary>**b)** One office or carrier gateway can represent thousands of users.</details>

**7.** Short — Why are rate limiting and load shedding both needed?
<details><summary>Answer</summary>Rate limits stop individual clients exceeding their share, but many clients each within their limits can still overload the system together; load shedding protects global capacity regardless of who's sending.</details>

**8.** Short — What should a rate limiter do when its shared store is unavailable?
<details><summary>Answer</summary>Decide explicitly per limit: fail open (allow, perhaps with a local fallback limiter) for general API limits to protect availability, and fail closed for security-critical limits like login attempts, where allowing everything defeats the purpose.</details>

**9.** Short — Why should rejections be cheap?
<details><summary>Answer</summary>Shedding happens during overload; if rejecting requires authentication, parsing, or database work, the rejections themselves consume the capacity being protected, and the service still falls over.</details>

**10.** Short — Why might LIFO beat FIFO for a queue under severe overload?
<details><summary>Answer</summary>The oldest queued requests are the likeliest to have exceeded their callers' deadlines and will be discarded anyway; serving the newest first gives work to callers who are still waiting.</details>

**11.** Short — What's weighted fair queueing, and what problem does it solve in multi-tenant systems?
<details><summary>Answer</summary>Serving per-tenant queues in rotation, in proportion to each tenant's weight, instead of first come, first served. It stops one tenant with a huge backlog from starving others.</details>

**12.** Output — Using §29.2's `tokenBucket(10, 10)` starting full at t = 0, how many of 15 requests all at t = 0 are allowed, and how many more of 5 requests at t = 300 ms?
<details><summary>Answer</summary>**10, then 3.** The burst of 10 drains the bucket; 300 ms refills 3 tokens.</details>

**13.** Output — Using §29.8's limiter with `rate: 2, periodMs: 1000, burst: 1`, three requests arrive at t = 0, 100, and 600 ms. Which are allowed?
<details><summary>Answer</summary>**t = 0 and t = 600.** The interval is 500 ms and there's no burst tolerance: after the first request, the next fits at 500 ms, so 100 ms is rejected and 600 ms is allowed.</details>

**14.** Output — A limit of 600 requests per minute is enforced locally as `limit ÷ N` on 12 instances. How many requests per minute does each instance allow per client, and what's the worst-case total if one instance receives all of a client's traffic?
<details><summary>Answer</summary>**50 per instance; the client is then limited to 50/min in total** — 12× stricter than intended — because local limits assume even distribution. (Conversely, with uneven N changes, clients can exceed the limit.)</details>

**15.** Output — What does this print?
```js
const est = (prev, cur, elapsed) => prev * (1 - elapsed) + cur;
console.log(est(80, 30, 0.25), est(80, 30, 0.75));
```
<details><summary>Answer</summary>`90 50` — the sliding window counter weights the previous window by how much of it still overlaps: 80 × 0.75 + 30, and 80 × 0.25 + 30.</details>

**16.** Debug — Customers report being rate-limited at about half their documented limit, but only when using the API from offices. Limits are keyed by IP address. What's happening?
<details><summary>Answer</summary>Many employees (and other companies' tools) share each office's NAT address, so they share one IP's quota. Key authenticated traffic by API key or user, and keep IP-based limits as a coarse abuse layer with higher thresholds.</details>

**17.** Debug — A Redis-backed limiter occasionally lets a client exceed its limit by 3–4× during bursts. The code does `GET count`, checks it in the app, then `SET count+1`. Cause and fix?
<details><summary>Answer</summary>A race: concurrent requests read the same count before any writes, and all pass. Use an atomic operation — `INCR` with `EXPIRE` for fixed windows, or a Lua script / GCRA compare-and-set for token buckets.</details>

**18.** Debug — During a traffic spike, a service at 100% concurrency kept accepting requests, its queue grew to minutes long, and it eventually crashed out of memory; autoscaling launched new instances 4 minutes later, which were overwhelmed on start. What should have happened?
<details><summary>Answer</summary>The service should have shed load as soon as concurrency or queue time exceeded its comfortable level — rejecting cheaply with `503` and `Retry-After`, prioritising critical traffic, and dropping queued work past its deadline — so it stayed up until autoscaled capacity arrived and was slow-started.</details>

**19.** Debug — After introducing rate limits, the backend saw *more* load than before, in sharp spikes. Clients are the company's own mobile apps. Why?
<details><summary>Answer</summary>The apps retry `429`s immediately (or with fixed delays, in sync), turning each rejection into more requests. Send `Retry-After` and rate-limit headers, and update the SDK to honour them with jittered backoff (Chapter 28).</details>

**20.** Code — Write `fixedWindowRedis(redis, key, limit, windowSec)` using `INCR` and `EXPIRE` that returns whether the request is allowed.
<details><summary>Answer</summary>

```js
async function fixedWindowRedis(redis, key, limit, windowSec) {
  const window = Math.floor(Date.now() / 1000 / windowSec);
  const k = `rl:${key}:${window}`;
  const count = await redis.incr(k);                // atomic: no read-then-write race
  if (count === 1) await redis.expire(k, windowSec * 2); // first hit in this window sets its TTL
  return count <= limit;
}
```
A tiny race remains between `INCR` and `EXPIRE` (a crash could leave a key without TTL); a `MULTI`/Lua version closes it.</details>

**21.** Code — Write `weightedRoundRobin(queues, weights)` that returns a function yielding the next tenant to serve, taking up to `weights[tenant]` items from each non-empty queue per round.
<details><summary>Answer</summary>

```js
function weightedRoundRobin(queues, weights) {
  const order = Object.keys(queues);
  let i = 0, servedThisTurn = 0;
  return function next() {
    for (let scanned = 0; scanned <= order.length; ) {
      const tenant = order[i];
      if (queues[tenant].length && servedThisTurn < (weights[tenant] ?? 1)) { servedThisTurn++; return tenant; }
      i = (i + 1) % order.length; servedThisTurn = 0; scanned++;   // move to the next tenant
    }
    return null;                                                   // every queue is empty
  };
}
const q = { a: [1, 2, 3, 4], b: [1, 2] }, next = weightedRoundRobin(q, { a: 2, b: 1 });
const served = []; for (let n = 0; n < 6; n++) { const t = next(); if (!t) break; q[t].shift(); served.push(t); }
console.log(served.join('')); // aabaab
```
</details>

**22.** Code — Write `shouldShed({ inFlight, limit, priority })` that sheds `sheddable` work above 80% of the limit, `normal` above 95%, and `critical` only at 100%.
<details><summary>Answer</summary>

```js
function shouldShed({ inFlight, limit, priority }) {
  const threshold = { sheddable: 0.8, normal: 0.95, critical: 1.0 }[priority] ?? 0.8;
  return inFlight >= Math.floor(limit * threshold); // reserve headroom for more important work
}
console.log(['sheddable', 'normal', 'critical'].map((p) => shouldShed({ inFlight: 90, limit: 100, priority: p }))); // [ true, false, false ]
```
</details>

**23.** Scenario — You're designing limits for a public API with free and paid tiers. Free users have 60 requests/minute; paid have 6,000/minute with bursts. Some endpoints run expensive searches. What do you build?
<details><summary>Answer</summary>Token buckets (or GCRA) per API key in a shared store, with tier-specific rate and burst; cost-weighted requests so an expensive search consumes, say, 10 tokens; monthly quotas tracked durably and separately; `429` with `Retry-After` and `RateLimit` headers on every response; a coarse per-IP limit at the edge for unauthenticated abuse; a global load shedder with criticality (paid > free when overloaded); fail-open with a local fallback limiter if the store is down; and SDKs that honour the headers.</details>

**24.** Scenario — A login endpoint is being hit by credential-stuffing attacks from thousands of IPs, each attempt using a different username. Per-IP limits don't help. What do you do?
<details><summary>Answer</summary>Layer limits across several keys: per account (failed attempts per username, with lockout or step-up challenges), per IP and per subnet/ASN with higher thresholds, and a global failed-login rate that triggers stronger defences (CAPTCHA, proof of work, device fingerprinting) for everyone. Use sliding logs or exact counters for these low limits and fail closed if the limiter's store is down. Add breached-password checks and MFA so a correct guess isn't enough. Limits slow attacks; authentication design defeats them (Chapter 33).</details>

---

## Chapter Summary

- Rate limiting enforces per-client fairness and quotas; load shedding protects global capacity during overload; both
  refuse work deliberately so the rest is served.
- Fixed windows allow double bursts at boundaries; sliding logs are exact but costly; sliding window counters
  approximate cheaply; token buckets and GCRA enforce a rate with a deliberate burst and are the usual default.
- Distributed limiters need atomic updates in a shared store, or local approximations with known inaccuracy; decide
  whether each limiter fails open or closed, and key by identity rather than IP alone.
- Return `429` with `Retry-After` and rate-limit headers (and `503` for shedding), and make clients honour them with
  jittered backoff.
- Shed load on direct overload signals, cheaply and early, by criticality, at every tier — and drop work whose caller
  has stopped waiting; autoscaling is too slow to substitute.
- In multi-tenant systems, combine tiered quotas, per-tenant concurrency limits, fair queueing, and cost-weighted
  limits so no tenant can starve the others.

---

## Practice Exercises

1. Implement the token bucket as an atomic Redis Lua script, then load-test it with 200 concurrent clients and verify
   the observed rate never exceeds the configured limit.
2. Extend §29.2's comparison with a leaky bucket and a GCRA limiter, and chart allowed requests over time for a bursty
   arrival pattern.
3. Read the IETF `RateLimit` header specification and implement the headers for your API's existing limits.
4. Add a concurrency-based load shedder to a service and run a load test past capacity, comparing p99 latency and
   goodput with and without shedding.
5. Find a third-party API your systems call, read its rate-limit documentation, and implement a client-side limiter
   that spends your quota deliberately and honours its headers.

---

## Mini Project: An API Protection Layer

Build a middleware layer for an HTTP API that combines rate limiting, quotas, and load shedding, and demonstrate it
under abusive and overload traffic.

- Per-API-key GCRA limits from a shared store with atomic updates, per-tier configuration, and cost-weighted endpoints.
- Monthly quotas tracked durably, with usage exposed at `/usage`.
- A per-instance load shedder using in-flight concurrency, with three criticality tiers derived from route and tier.
- Standard `429`/`503` responses with `Retry-After` and `RateLimit` headers, and a client SDK that honours them.
- A test harness with a runaway client, a credential-stuffing pattern, and a global spike, reporting goodput, latency for
  well-behaved clients, and rejections by reason.

---

**Next:** [Chapter 30 — Observability →](30-observability.md)
