# Chapter 16 — Caching

## Learning Objectives

- Place caches at the right layers and predict their effect on latency and load from the hit ratio
- Choose between cache-aside, read-through, write-through, write-behind, and refresh-ahead
- Invalidate correctly, including the race that leaves a stale value cached after a delete
- Prevent stampedes with single-flight loads, early refresh, and stale-while-revalidate
- Size a cache, choose an eviction policy, and design for the cache being slow, cold, or down

---

## 16.1 Why Cache, and Where

A **cache** keeps a copy of data somewhere faster or closer than its source, so most requests skip
the expensive path. Chapter 2's latency table is the whole argument: memory is about a thousand
times faster than an SSD, which is about a thousand times faster than a cross-continent round trip.
A cache that answers from memory turns a 20 ms database query into a 0.5 ms lookup — and, often more
importantly, removes that query's load from the database entirely.

Caches exist at every layer of a request path, and a mature system uses several:

| Layer | Holds | Latency | Scope | Chapter |
|---|---|---|---|---|
| Client (browser, app) | Responses, assets | 0 | One user | 6 |
| CDN / edge | Public responses, assets | ~10 ms | Everyone near a PoP | 6 |
| In-process (application memory) | Hot objects, config, computed values | ~µs | One instance | this |
| Distributed cache (Redis, Memcached) | Query results, sessions, objects | ~0.5–1 ms | All instances | this |
| Database buffer pool / page cache | Pages | ~µs (on the DB host) | The database | 10 |

The value of a cache comes down to its **hit ratio**, and the effect on latency is easy to compute:

```js
// Effective latency with a cache: hits are cheap, misses pay for the cache AND the database
const effective = ({ hit, cacheMs, dbMs }) => hit * cacheMs + (1 - hit) * (cacheMs + dbMs);
for (const hit of [0.5, 0.9, 0.99]) console.log(hit, effective({ hit, cacheMs: 0.5, dbMs: 20 }).toFixed(1), 'ms');
// 0.5 10.5 ms
// 0.9 2.5 ms
// 0.99 0.7 ms
```

Two things follow. Latency improves dramatically only at high hit ratios — at 50%, the cache barely
helps the average and doesn't help the tail at all, because half the requests still pay the full
miss. And **load reduction is linear in the miss ratio**: going from 90% to 99% hits cuts database
load by 10×, which is often what makes a design feasible (Chapter 1's catalogue example).

What's worth caching is data that is **read far more often than it changes**, **expensive to
produce**, and **tolerant of being slightly stale** — product details, user profiles, rendered
fragments, permission sets, computed aggregates. Data that changes on nearly every read, or whose
staleness causes real harm (an account balance used to authorise a payment), is a poor candidate.

---

## 16.2 Caching Patterns

The patterns differ in who loads the cache and when writes reach it:

| Pattern | Read path | Write path | Strengths | Weaknesses |
|---|---|---|---|---|
| **Cache-aside** (lazy loading) | App checks cache; on miss reads DB and fills cache | App writes DB, then deletes (or updates) the cache entry | Simple; cache holds only what's read; cache failure isn't fatal | First read of each key is a miss; invalidation races (§16.3) |
| **Read-through** | Cache itself loads from the DB on miss | — | Loading logic in one place | Cache library/service must know the data source |
| **Write-through** | Reads served from cache | Writes go to the cache, which writes the DB synchronously | Cache always fresh for written keys | Every write pays both; caches data that may never be read |
| **Write-behind** (write-back) | Reads served from cache | Writes go to the cache; DB updated asynchronously | Very fast writes; can batch | Data loss if the cache fails before flushing; ordering is hard |
| **Refresh-ahead** | Reads served from cache | — (background refresh before expiry) | Hot keys never miss | Refreshes keys that may not be read again |

**Cache-aside is the default** for application caching, and the rest of this chapter assumes it. It
keeps the database as the single source of truth, it degrades gracefully (a dead cache means slower
reads, not wrong ones), and it's easy to reason about. Write-behind is the dangerous one: it makes the
cache the source of truth for a window, so it belongs only where losing recent writes is acceptable
(view counters, analytics) or where the cache is itself durable and replicated.

On writes in cache-aside, **delete rather than update** the cached entry. Updating looks more
efficient, but two concurrent writers can update the database in one order and the cache in the
other, leaving the cache permanently holding the older value. Deleting forces the next reader to load
the current value from the database.

> 💼 **Interview** — "How do you keep the cache consistent with the database?" The honest answer is
> that cache-aside gives eventual consistency bounded by the TTL: write the database, delete the key,
> keep TTLs short enough that any missed invalidation self-heals, and use versioned values or
> CDC-driven invalidation where staleness matters more. Claiming perfect consistency is the wrong
> answer.

---

## 16.3 Invalidation, and the Race Everyone Hits

Phil Karlton's line — "there are only two hard things in computer science: cache invalidation and
naming things" — is quoted so often because invalidation really is where caches go wrong. The
strategies:

| Strategy | How | Staleness bound |
|---|---|---|
| **TTL only** | Entries expire after N seconds | N seconds, always |
| **Delete on write** | The writing code deletes affected keys | Near zero — *if* every write path remembers, and no race occurs |
| **Versioned keys** | Key includes a version: `product:42:v17`; writes bump the version | Zero for readers who know the new version |
| **Event-driven (CDC)** | A consumer of the database's change stream deletes or updates keys | Replication lag of the stream (Chapter 11) |

Use **TTL as the safety net under every other strategy**. Delete-on-write is fast but fragile — one
forgotten write path (a bulk update script, an admin tool) and the cache serves stale data until the
TTL saves you. CDC-driven invalidation fixes the forgotten-path problem, because every committed
change appears on the stream no matter which code made it.

### The delete-then-stale-set race

Even with perfect delete-on-write, cache-aside has a race. A reader misses and reads the database;
before it writes the value into the cache, a writer updates the database and deletes the key; then
the reader writes its now-stale value. The delete happened *before* the stale set, so it had nothing
to delete:

```js
// The classic cache-aside race, replayed step by step.
const db = new Map([['price:42', 100]]);
const cache = new Map();

const loaded = cache.get('price:42') ?? db.get('price:42');   // reader: miss → reads 100
db.set('price:42', 80);                                       // writer: update the database…
cache.delete('price:42');                                     // …and invalidate (nothing there yet)
cache.set('price:42', loaded);                                // reader: populates the cache with 100

console.log('db:', db.get('price:42'), '| cache:', cache.get('price:42')); // db: 80 | cache: 100

// Fix: version the value, and only let a set win if it isn't older than what's cached
const vdb = new Map([['price:42', { value: 100, version: 1 }]]);
const vcache = new Map();
const setIfNewer = (k, entry) => {
  const cur = vcache.get(k);
  if (!cur || entry.version > cur.version) vcache.set(k, entry);
};
const vloaded = vcache.get('price:42') ?? vdb.get('price:42');   // version 1
vdb.set('price:42', { value: 80, version: 2 });
setIfNewer('price:42', vdb.get('price:42'));                     // the writer (or CDC) sets version 2
setIfNewer('price:42', vloaded);                                 // the slow reader's stale set is rejected
console.log('versioned cache:', vcache.get('price:42'));         // versioned cache: { value: 80, version: 2 }
```

The window is small — the reader must be slow between its database read and its cache write — but at
high traffic it happens daily, and the stale value survives until the TTL. Fixes, in increasing rigour:

- **Short TTLs**, so the damage is bounded. The pragmatic default.
- **Delayed double delete**: delete on write, then delete again a second later, after any in-flight
  reader would have finished. Crude, but it closes most of the window.
- **Leases**: on a miss, the cache hands the reader a token; a delete invalidates outstanding tokens;
  a set with an invalidated token is rejected. Memcached's lease design (described in Facebook's
  memcache paper) exists for exactly this race, and also prevents stampedes.
- **Versioned compare-and-set**, as above: store the row's version with the value and reject sets that
  are older than what's cached. In Redis this is a small Lua script; a CDC consumer setting values
  with the source's commit position does the same thing.

---

## 16.4 Stampedes and Thundering Herds

When a hot key expires, every request for it misses at once, and every one of them goes to the
database to recompute the same value. A key read 5,000 times per second whose recomputation takes
200 ms produces about 1,000 concurrent identical database queries — enough to overload the database,
which makes the recomputation slower, which makes the stampede larger. A **cache stampede** can take
down a healthy system in seconds, and it often starts after a deploy or restart when many keys are
cold at once.

Defences, usually combined:

**Single-flight (request coalescing).** Within a process, the first miss for a key starts the load;
concurrent misses for the same key wait for that same promise instead of starting their own. This
reduces N concurrent loads per process to one per process. Across many instances, a short-lived
distributed lock or a lease (§16.3) extends it to one per cluster.

**Stale-while-revalidate.** Keep two expiry times: a **soft TTL** after which the value is considered
stale, and a **hard TTL** after which it's gone. Between them, serve the stale value immediately and
refresh in the background. Users never wait on a refresh, and only one refresh runs per key. This is
Chapter 6's CDN directive applied to the application cache, and it's the most effective single defence.

**Probabilistic early expiration.** Each reader, as expiry approaches, decides at random whether to
refresh early — with probability rising as the deadline nears and with the cost of recomputation. One
lucky reader refreshes slightly before expiry, so the key never actually expires under load. The
"XFetch" formulation fits in one line:

```js
// XFetch: refresh early with a probability that rises as expiry approaches, scaled by how
// long recomputation takes. One reader usually refreshes just before the key would expire.
const shouldRefreshEarly = ({ now, expiry, recomputeMs, beta = 1, rand = Math.random }) =>
  now - recomputeMs * beta * Math.log(rand()) >= expiry;

console.log(
  shouldRefreshEarly({ now: 0, expiry: 10_000, recomputeMs: 500, rand: () => 0.5 }),     // far from expiry
  shouldRefreshEarly({ now: 9_800, expiry: 10_000, recomputeMs: 500, rand: () => 0.5 })); // 200 ms before
// false true
```

**Jittered TTLs.** Keys written together (by a batch warm-up, or after a deploy) expire together.
Adding ±10% randomness to each TTL spreads their expiries out.

**Warm before traffic.** After a cache cluster restart or a new region launch, preload the hot set
before sending traffic, rather than letting real users discover every miss.

> ⚠️ **Gotcha** — **A cache outage is a database load test you didn't schedule.** If the database is
> sized for the 5% of reads that miss, losing the cache sends it 20× its normal load instantly. Either
> size the database to survive a cold cache (expensive), or protect it: shed or queue requests when
> the cache is unavailable, cap concurrent loads per key and in total, and serve stale data from a
> local in-process layer while the distributed cache recovers.

---

## 16.5 Eviction and Sizing

A cache has finite memory, so when it's full, something must go. The **eviction policy** decides what:

| Policy | Evicts | Good for | Weakness |
|---|---|---|---|
| **LRU** (least recently used) | The entry untouched for longest | General workloads with temporal locality | A one-off scan of many keys flushes the hot set |
| **LFU** (least frequently used) | The entry used least often | Stable popularity (reference data) | Slow to adapt when popularity shifts; old counts linger |
| **TTL-based** | The entry expiring soonest | Data with natural expiry | Ignores popularity |
| **Random** | A random entry | Cheap approximation | Occasionally evicts hot keys |
| **W-TinyLFU** (Caffeine, others) | Admits new entries only if likely more popular than the victim | Mixed and scan-heavy workloads | More complex |

Redis approximates LRU or LFU by sampling a few keys and evicting the best candidate among them, which
is cheap and close to exact. In-process caches can implement LRU exactly with one ordered map:

```js
// lru.mjs — a Map remembers insertion order, so delete-and-reinsert moves a key to the end;
// the first key is always the least recently used.
class LruCache {
  #map = new Map();
  constructor(capacity) { this.capacity = capacity; }
  get(key) {
    if (!this.#map.has(key)) return undefined;
    const value = this.#map.get(key);
    this.#map.delete(key); this.#map.set(key, value);   // mark as most recently used
    return value;
  }
  set(key, value) {
    this.#map.delete(key);
    this.#map.set(key, value);
    if (this.#map.size > this.capacity) this.#map.delete(this.#map.keys().next().value); // evict LRU
  }
  keys() { return [...this.#map.keys()]; }
}
const lru = new LruCache(3);
lru.set('a', 1); lru.set('b', 2); lru.set('c', 3);
lru.get('a');            // a is now most recent
lru.set('d', 4);         // evicts b, the least recently used
console.log(lru.keys()); // [ 'c', 'a', 'd' ]
```

**Sizing** follows Chapter 2: estimate the hot set (distinct keys read in a period × the fraction that
gets most reads × value size), add overhead per entry (keys, metadata, allocator fragmentation —
easily 50–100 bytes per entry in Redis, and fragmentation can add 20–50% more), and leave headroom so
eviction isn't constant. Then **measure the hit ratio against cache size**: hit ratio rises steeply
until the hot set fits and flattens afterwards, and the knee of that curve is the right size. Beyond
it, more memory buys almost nothing.

A sharp edge: **eviction is not a substitute for expiry.** If entries have no TTL and the cache is
large enough never to evict, stale entries live forever. Every entry should have a TTL, even a long one.

---

## 16.6 Distributed Caches in Practice

A shared cache like Redis or Memcached is itself a distributed system, with the problems of Chapters 13
and 14:

- **Sharding.** Keys are spread across nodes with consistent hashing or slots (Chapter 14), so adding a
  node remaps only a fraction of keys. `hash % N` sharding turns every resize into a near-total miss
  storm.
- **Replication.** Redis replicas take over on primary failure, but replication is asynchronous, so a
  failover can lose recent writes — fine for a cache, fatal if you've quietly started using it as a
  database (Chapter 11's cart example).
- **Hot keys.** One key on one node can take more traffic than that node can serve — a celebrity's
  profile, a global config blob. Add a small **in-process L1 cache** (hundreds of milliseconds to a few
  seconds of TTL) in front of the distributed cache for the hottest keys, or replicate those keys under
  several names and read a random copy.
- **Large values.** A 5 MB cached value takes milliseconds to transfer and serialise, blocks Redis's
  single command thread for other clients, and wastes memory if only a field is needed. Cache smaller,
  more targeted objects.
- **Timeouts.** Cache calls need tight timeouts (tens of milliseconds). A cache that is slow, rather
  than down, is worse than no cache: every request waits for it and then goes to the database anyway.

**Multi-level caching** — in-process L1, distributed L2, database — compounds hit ratios and protects
against hot keys, but each level adds a place for staleness. Keep L1 TTLs very short, and if L1 holds
data that must be invalidated promptly, broadcast invalidations to all instances (via pub/sub) rather
than relying on TTL alone.

---

## 16.7 What Not to Cache — and Caching Safely

- **Don't cache without a key that includes everything the value depends on.** A value computed for
  one user, tenant, locale, or permission set, stored under a key that omits it, is served to the
  wrong user. Cache keys are a security boundary — Chapter 6 showed this at the edge, and it's equally
  true in Redis.
- **Don't cache errors as successes.** A transient database error cached for 60 seconds becomes a
  60-second outage. Cache only successful results — and use short-lived **negative caching** for
  genuine "not found" answers, which protects the database from repeated lookups of keys that don't
  exist (including deliberate floods of random IDs).
- **Don't put the cache on the write path of invariants.** Reading a cached balance or stock level to
  decide whether to allow a purchase reintroduces every anomaly from Chapter 15, with seconds of
  staleness added. Decisions that protect invariants read from the source of truth.
- **Don't cache what's cheap to compute.** Serialisation, network, and memory overhead can exceed the
  cost of a fast indexed query; measure first.

---

## 16.8 Common Mistakes and Best Practices

**Mistake: updating the cache on write instead of deleting.** Concurrent writers apply to the database
and cache in different orders, leaving the older value cached.

**Mistake: entries without TTLs.** A missed invalidation, or the delete-then-stale-set race, leaves
stale data forever instead of for a bounded time.

**Mistake: no stampede protection on hot keys.** One expiry triggers hundreds of identical recomputations
and can overload the database.

**Mistake: a database sized only for cache misses.** Losing or flushing the cache sends it many times its
normal load with no warning.

**Mistake: cache keys missing a dimension the value depends on.** Users, tenants, or locales get each
other's data — a security incident, not a bug.

**Mistake: caching transient errors.** A brief failure becomes a TTL-long outage for everyone reading
that key.

**Mistake: slow cache calls without timeouts.** A degraded cache adds its latency to every request and
still sends them to the database.

**Best practice: cache-aside, delete on write, TTL as a safety net.** Simple, source-of-truth-preserving,
and self-healing.

**Best practice: soft and hard TTLs with background refresh.** Users never wait on a recomputation, and
only one refresh runs per key.

**Best practice: treat the cache as optional.** Cache failures become misses behind tight timeouts, with
load protection for the database behind it.

---

## 16.9 Real-World Example: A Defensive Read Cache

This wraps any loader (a database query, a service call) in cache-aside with the protections from this
chapter: single-flight loads per key, soft and hard TTLs for stale-while-revalidate, jittered expiry,
negative caching, tight timeouts on the cache itself, and graceful behaviour when the cache fails.

```js
// read-cache.mjs — cache-aside with the protections production needs: single-flight loads,
// soft/hard TTLs (serve stale while refreshing), jittered expiry, negative caching, and
// treating the cache as optional when it fails.

export function createReadCache({ store, load, softTtlMs = 30_000, hardTtlMs = 300_000,
  negativeTtlMs = 5_000, cacheTimeoutMs = 50, now = Date.now, rand = Math.random,
  onEvent = () => {} }) {
  const inFlight = new Map(); // key → Promise: one load per key at a time, per process

  const withTimeout = (p) => Promise.race([p,
    new Promise((_, rej) => setTimeout(() => rej(new Error('cache timeout')), cacheTimeoutMs))]);

  // Jitter spreads expiries so keys cached together don't all expire together
  const jitter = (ms) => Math.round(ms * (0.9 + rand() * 0.2));

  async function refresh(key) {
    if (inFlight.has(key)) return inFlight.get(key);
    const p = (async () => {
      const value = await load(key);              // the expensive call: database, service
      const ttl = value === null ? negativeTtlMs : softTtlMs;
      const entry = { value, softExpiry: now() + jitter(ttl), hardExpiry: now() + jitter(hardTtlMs) };
      // A failed cache write must not fail the request — the value is still correct
      await withTimeout(store.set(key, entry, hardTtlMs)).catch(() => onEvent('cache_write_failed'));
      return value;
    })().finally(() => inFlight.delete(key));
    inFlight.set(key, p);
    return p;
  }

  return async function get(key) {
    let entry = null;
    try { entry = await withTimeout(store.get(key)); }
    catch { onEvent('cache_read_failed'); }       // cache down or slow: behave as a miss

    if (entry && now() < entry.softExpiry) { onEvent('hit'); return entry.value; }
    if (entry && now() < entry.hardExpiry) {
      onEvent('stale');                           // serve stale now, refresh in the background
      refresh(key).catch(() => onEvent('refresh_failed')); // keep serving stale if the source is down
      return entry.value;
    }
    onEvent('miss');
    return refresh(key);
  };
}

// ── Demo ─────────────────────────────────────────────────────────────────────────────
const mem = new Map();
const store = { async get(k) { return mem.get(k) ?? null; }, async set(k, v) { mem.set(k, v); } };
let t = 0, loads = 0;
const events = {};
const get = createReadCache({
  store, now: () => t, rand: () => 0.5,
  load: async (key) => { loads++; await new Promise((r) => setTimeout(r, 10)); return key === 'missing' ? null : `v${loads}`; },
  onEvent: (e) => { events[e] = (events[e] ?? 0) + 1; },
});

await Promise.all(Array.from({ length: 100 }, () => get('product:1'))); // 100 concurrent misses
console.log('loads after 100 concurrent misses:', loads);  // loads after 100 concurrent misses: 1
t += 40_000;                                               // past the soft TTL, before the hard TTL
console.log('stale read returns:', await get('product:1')); // stale read returns: v1
await new Promise((r) => setTimeout(r, 20));               // the background refresh completes
console.log('after refresh:', await get('product:1'));     // after refresh: v2
await get('missing'); await get('missing');                // negative caching: one load for two misses
console.log('loads:', loads, events);                      // loads: 3 { miss: 101, stale: 1, hit: 2 }
```

The single-flight map is per process, so a fleet of 50 instances still makes up to 50 loads when a hot
key expires; for keys where that matters, add a short distributed lock or lease around `refresh`, or
rely on the soft TTL so expiry never coincides with a burst.

The decision to defend is **serving data past its soft TTL — knowingly stale — rather than making the
caller wait for fresh data.** It can feel like a correctness compromise. But the soft TTL was already a
staleness budget: a 30-second TTL said "data up to 30 seconds old is acceptable". Serving it at 31
seconds while a refresh runs exceeds that budget by the refresh time — tens of milliseconds — whereas
blocking the caller adds that same refresh time to user-facing latency, and blocking *every* caller at
once is what turns an expiry into a stampede. The hard TTL keeps the trade honest: if the source stays
down long enough, stale data eventually stops being served instead of being served forever. For data
where even the soft TTL is too much staleness, the answer is a shorter TTL or no cache, not a blocking
refresh.

---

## 16.10 Questions

**1.** MCQ — In cache-aside, what should a write do to the cache? a) Update it with the new value b) Delete the key c) Nothing d) Flush the whole cache
<details><summary>Answer</summary>**b)** Deleting avoids concurrent writers leaving the older value cached; the next read loads the current value.</details>

**2.** MCQ — A cache's hit ratio rises from 95% to 99%. How does database read load change? a) Falls by ~4% b) Falls by 5× c) Falls by 99% d) Unchanged
<details><summary>Answer</summary>**b)** The miss ratio falls from 5% to 1%.</details>

**3.** MCQ — Which pattern risks losing writes if the cache fails? a) Cache-aside b) Read-through c) Write-through d) Write-behind
<details><summary>Answer</summary>**d)** Writes are acknowledged from the cache and flushed to the database later; a failure before the flush loses them.</details>

**4.** MCQ — What does single-flight loading prevent? a) Stale reads b) Many concurrent identical loads for the same key within a process c) Evictions d) Cache poisoning
<details><summary>Answer</summary>**b)** Concurrent misses share one in-flight load instead of each querying the source.</details>

**5.** MCQ — Which eviction weakness does a one-time scan of millions of keys expose? a) LFU forgets popular items b) LRU flushes the hot set c) TTL eviction stops working d) Random eviction slows down
<details><summary>Answer</summary>**b)** Every scanned key becomes "most recently used", pushing the genuinely hot keys out. Scan-resistant policies like W-TinyLFU address this.</details>

**6.** MCQ — Why add an in-process L1 cache in front of Redis? a) To make writes durable b) To absorb hot keys that would overload one Redis node c) To replace TTLs d) To avoid serialisation entirely
<details><summary>Answer</summary>**b)** A very short local cache serves the hottest keys without a network hop, so no single Redis shard takes all their traffic.</details>

**7.** Short — Describe the delete-then-stale-set race in cache-aside.
<details><summary>Answer</summary>A reader misses and reads the old value from the database; a writer updates the database and deletes the (absent) key; the reader then writes the old value into the cache, where it stays until the TTL expires.</details>

**8.** Short — What are soft and hard TTLs, and what does each protect against?
<details><summary>Answer</summary>After the soft TTL the value is stale and is served while a background refresh runs, removing refresh latency and stampedes. After the hard TTL the value is no longer served at all, bounding how stale data can get if the source stays down.</details>

**9.** Short — Why is a slow cache worse than a dead one?
<details><summary>Answer</summary>A dead cache fails fast and requests go to the source; a slow one makes every request wait for it and then often go to the source anyway, adding its latency to everything. Tight timeouts turn slowness into fast misses.</details>

**10.** Short — What is negative caching, and what attack does it blunt?
<details><summary>Answer</summary>Briefly caching "not found" results so repeated lookups of missing keys don't hit the database. It blunts floods of requests for random non-existent IDs designed to bypass the cache.</details>

**11.** Short — Why should invariant-protecting decisions not read from a cache?
<details><summary>Answer</summary>Cached values can be stale by up to the TTL, so checks like "is there enough stock/balance?" can pass on outdated data, reintroducing lost updates and oversells. Such decisions must read the source of truth, ideally atomically.</details>

**12.** Output — Using §16.1's `effective` function, what's the effective latency at a 95% hit ratio with a 1 ms cache and a 30 ms database?
<details><summary>Answer</summary>**2.5 ms.** 0.95 × 1 + 0.05 × (1 + 30) = 0.95 + 1.55 = 2.5 ms.</details>

**13.** Output — What does this print?
```js
const lru = new Map();
const touch = (k) => { const v = lru.get(k); lru.delete(k); lru.set(k, v); };
['a', 'b', 'c'].forEach((k) => lru.set(k, 1));
touch('b'); touch('a');
console.log([...lru.keys()][0]);
```
<details><summary>Answer</summary>`c` — after touching b then a, the order is c, b, a, so c is the least recently used and would be evicted next.</details>

**14.** Output — Using §16.4's `shouldRefreshEarly`, with `expiry: 10_000`, `recomputeMs: 1_000`, and `rand: () => 0.5`, does a reader at `now: 9_000` refresh early?
<details><summary>Answer</summary>**No.** −1,000 × ln(0.5) ≈ 693, and 9,000 + 693 = 9,693 is still below 10,000. With r = 0.5 the reader refreshes early only from about `now` = 9,307 onward; at 9,000 it would need a draw below e⁻¹ ≈ 0.37.</details>

**15.** Output — A key is read 4,000 times per second. Recomputing it takes 250 ms. Without stampede protection, roughly how many concurrent recomputations start when it expires?
<details><summary>Answer</summary>**~1,000.** Every request in the 250 ms recomputation window misses: 4,000 × 0.25 = 1,000 (Little's law again).</details>

**16.** Debug — After a Redis cluster restart, the primary database's CPU hits 100% and the site is down for 10 minutes, though Redis came back in 30 seconds. What happened, and what prevents it?
<details><summary>Answer</summary>The cache came back empty, so nearly every read missed and stampeded the database, which was sized only for normal miss rates; slow queries made refreshes slower and the stampede longer. Prevent it with single-flight and stale-while-revalidate, a cap on concurrent loads, load shedding while cold, warming the hot set before admitting traffic, and an in-process L1 that survives the distributed cache's restart.</details>

**17.** Debug — Product prices occasionally show the old value for exactly 10 minutes after an update, though the update code deletes the cache key. TTL is 10 minutes. Cause and fix?
<details><summary>Answer</summary>The delete-then-stale-set race: a reader that loaded the old price before the update wrote it to the cache after the delete. Use versioned compare-and-set (reject older versions), leases, or delayed double delete; and shorten the TTL to bound the damage.</details>

**18.** Debug — A user saw another user's dashboard widgets after a caching change. The widget data is cached under `widgets:dashboard:<dashboardId>`. What's wrong?
<details><summary>Answer</summary>The cached value depends on the viewer (permissions, personalisation), but the key omits the user or permission set, so the first viewer's result is served to everyone. Include every dimension the value depends on in the key (`user` or a permission-set hash), or cache only the viewer-independent part and filter per request.</details>

**19.** Debug — A brief database failover caused a 5-minute outage of the product page, long after the database recovered in 20 seconds. Investigation shows `null` was cached for many products. Why?
<details><summary>Answer</summary>The loader returned `null` on error (or errors were treated as "not found"), and those results were cached with the normal TTL. Distinguish errors from absence: never cache errors, use a short negative-cache TTL only for genuine not-found results, and serve stale entries when the source fails.</details>

**20.** Code — Write `singleFlight(fn)` returning a function that, for concurrent calls with the same key, invokes `fn(key)` once and shares the result.
<details><summary>Answer</summary>

```js
function singleFlight(fn) {
  const inFlight = new Map();
  return (key) => {
    if (inFlight.has(key)) return inFlight.get(key);
    const p = Promise.resolve().then(() => fn(key)).finally(() => inFlight.delete(key));
    inFlight.set(key, p);
    return p;
  };
}
let calls = 0;
const load = singleFlight(async (k) => { calls++; await new Promise((r) => setTimeout(r, 10)); return k.toUpperCase(); });
console.log(await Promise.all([load('a'), load('a'), load('b')]), calls); // [ 'A', 'A', 'B' ] 2
```
</details>

**21.** Code — Write `ttlWithJitter(baseMs, spread = 0.1, rand = Math.random)` returning a TTL uniformly distributed in `[base × (1 − spread), base × (1 + spread)]`.
<details><summary>Answer</summary>

```js
const ttlWithJitter = (baseMs, spread = 0.1, rand = Math.random) =>
  Math.round(baseMs * (1 - spread + rand() * 2 * spread));
console.log(ttlWithJitter(60_000, 0.1, () => 0), ttlWithJitter(60_000, 0.1, () => 0.999)); // 54000 65988
```
</details>

**22.** Code — Write a `setIfNewer(store, key, entry)` for a store with an atomic `compareAndSet(key, expectedRaw, newRaw)` that only replaces the cached entry when `entry.version` is greater, retrying if the CAS loses a race.
<details><summary>Answer</summary>

```js
async function setIfNewer(store, key, entry, maxTries = 5) {
  for (let i = 0; i < maxTries; i++) {
    const raw = await store.get(key);
    const current = raw ? JSON.parse(raw) : null;
    if (current && current.version >= entry.version) return false;   // cached value is as new or newer
    if (await store.compareAndSet(key, raw, JSON.stringify(entry))) return true;
  }
  return false; // heavy contention: let the TTL and the next write sort it out
}
```
In Redis the same logic is usually a short Lua script, which runs atomically without a retry loop.</details>

**23.** Scenario — A product page makes 14 database queries and has a p99 of 900 ms. A teammate proposes caching each query result in Redis with a 1-hour TTL. What do you think, and what would you do?
<details><summary>Answer</summary>Caching fourteen fragments for an hour risks long staleness and fourteen invalidation paths, and doesn't fix the p99 driven by misses. First measure which queries dominate and whether indexes or batching fix them. Then cache the assembled, viewer-independent page model (or a few coarse fragments) with a short soft TTL plus stale-while-revalidate, invalidated by CDC on product changes, and serve personalised parts separately. Keep stock and price checks at checkout on the source of truth.</details>

**24.** Scenario — Your company runs one large Redis instance holding sessions, rate-limit counters, cached API responses, and a job queue. It's at 85% memory and evictions are starting. What do you recommend?
<details><summary>Answer</summary>Separate workloads by durability needs: caches can evict, but sessions, rate-limit state, and especially the queue must not be evicted — under memory pressure the eviction policy may delete exactly those. Split into at least a cache instance (with an LRU/LFU policy and TTLs on everything) and a durable instance (no eviction, persistence on) for sessions and counters, and move the queue to a proper queue (Chapter 24). Then size the cache from the hit-ratio curve.</details>

---

## Chapter Summary

- A cache's value is set by its hit ratio: latency improves meaningfully only at high hit ratios, while
  load on the source falls linearly with the miss ratio.
- Cache-aside with delete-on-write and a TTL on every entry is the right default; write-behind makes the
  cache a temporary source of truth and belongs only where losing writes is acceptable.
- Invalidation fails through forgotten write paths and the delete-then-stale-set race; TTLs bound the
  damage, and CDC-driven invalidation, leases, or versioned compare-and-set close the gaps.
- Stampedes turn one expiry into hundreds of recomputations; single-flight, soft/hard TTLs with
  background refresh, early probabilistic refresh, and jitter prevent them.
- Size caches from the hot set and the hit-ratio curve, pick eviction to match access patterns, and add
  a short in-process layer for hot keys.
- Treat the cache as optional behind tight timeouts, protect the source from a cold cache, include every
  dependent dimension in keys, and never cache errors or use cached values to protect invariants.

---

## Practice Exercises

1. Measure the hit ratio of a cache you operate against its memory size by replaying a day of key access
   logs through LRU caches of several sizes, and find the knee of the curve.
2. Reproduce the delete-then-stale-set race against a real Redis and database with a deliberate delay in
   the reader, then fix it with a Lua-based versioned set.
3. Read the "Scaling Memcache at Facebook" paper's sections on leases and regional invalidation, and map
   each problem it solves to a section of this chapter.
4. Load-test an endpoint with a hot key and a 10-second TTL, with and without single-flight and
   stale-while-revalidate, and chart database queries per second around each expiry.
5. Implement a W-TinyLFU-style admission filter (a frequency sketch that decides whether a new key may
   evict the LRU victim) and compare its hit ratio with plain LRU on a trace that includes a large scan.

---

## Mini Project: A Two-Level Cache Library

Build a caching library with an in-process L1 and a Redis (or simulated) L2 that an application can wrap
around any loader, with behaviour you can observe and test.

- L1 with LRU eviction and very short TTLs; L2 with soft/hard TTLs, jitter, and negative caching.
- Single-flight within a process and an optional short distributed lock across processes.
- Invalidation by key and by tag, broadcast to every instance's L1 via pub/sub.
- Versioned compare-and-set on L2 writes to defeat the stale-set race.
- Metrics for hits, misses, stale serves, refreshes, and load latency per level, plus a chaos test that
  kills L2 under load and shows the source's load staying bounded.

---

**Next:** [Chapter 17 — Object Storage and Large Files →](17-object-storage.md)
