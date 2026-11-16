# Chapter 35 — Design a URL Shortener

## Learning Objectives

- Scope a URL shortener and derive the numbers that shape it: write rate, redirect rate, and key-space size
- Compare key-generation strategies — hashing, random keys, counters, range leasing — by collisions, coordination, and guessability
- Serve redirects at high rates with a layered cache, and choose between 301 and 302 deliberately
- Collect click analytics without putting them on the redirect's critical path
- Identify the failure modes and abuse vectors of a public link service, and what changes at 10×

---

## 35.1 Requirements and Scope

A URL shortener turns a long URL into a short one (`sho.rt/LDYvwns`) and redirects visitors from the short URL to
the original. It looks trivial, which is why it's a classic: the interesting parts are generating keys at scale, serving
a very read-heavy workload fast, and the operational realities of a public service that anyone can feed URLs into.

**Functional requirements**

- Create a short link for a long URL; optionally with a custom alias and an expiry.
- Redirect a short link to its target.
- Basic click analytics for the link's owner: counts over time, referrers, countries.

**Out of scope** (stated explicitly): user accounts beyond an API key, editing a link's target after creation, link
previews, QR codes, A/B routing.

**Non-functional requirements**

| Requirement | Target |
|---|---|
| Redirect latency | p99 < 50 ms at the edge for cached links, < 100 ms uncached |
| Availability | 99.99% for redirects (a dead link is a broken page on someone else's site); 99.9% for creation |
| Durability | A created link must never be lost or start pointing elsewhere |
| Key length | 7 characters |
| Analytics freshness | Minutes; counts may be approximate in real time and exact daily |
| Abuse | Malicious and spam links must be detectable and removable |

The asymmetry in availability matters: creation can tolerate brief outages; redirects can't, because they're embedded in
emails, documents, and printed material that will be clicked for years.

---

## 35.2 Estimation

Assume 100 million new links per month and 100 redirects per link on average over its lifetime, with links kept for at
least five years.

```js
// URL shortener: the numbers that decide the design
const linksPerMonth = 100e6, readsPerWrite = 100, years = 5, bytesPerLink = 500;
const writesPerSec = linksPerMonth / (30 * 86_400);
const readsPerSec = writesPerSec * readsPerWrite;
const totalLinks = linksPerMonth * 12 * years;
console.log(`writes ~${writesPerSec.toFixed(0)}/s avg | redirects ~${Math.round(readsPerSec).toLocaleString('en-US')}/s avg, ~${Math.round(readsPerSec * 3).toLocaleString('en-US')}/s peak`);
console.log(`links after ${years} years: ${(totalLinks / 1e9).toFixed(0)} B → ${(totalLinks * bytesPerLink / 1e12).toFixed(0)} TB raw`);
console.log(`7-char base62 space: ${(62 ** 7 / 1e12).toFixed(1)} T keys → ${((totalLinks / 62 ** 7) * 100).toFixed(2)}% used after ${years} years`);
// writes ~39/s avg | redirects ~3,858/s avg, ~11,574/s peak
// links after 5 years: 6 B → 3 TB raw
// 7-char base62 space: 3.5 T keys → 0.17% used after 5 years
```

What the numbers decide:

- **~40 writes/s** is trivial for any database — creation doesn't need sharding for write throughput.
- **~12,000 redirects/s at peak** (and far more for viral links) means redirects must come from caches, not the database.
- **6 billion links, ~3 TB raw** (perhaps 15–20 TB with replication and indexes) is too large to want on one node
  forever, but its access pattern is pure key lookup — ideal for a partitioned key-value store, or a relational database
  partitioned by key.
- **A 7-character base62 key space of 3.5 trillion** is 0.17% used after five years — plenty of room, which makes random
  keys viable (§35.5).

Redirects follow a heavy-tailed distribution: a small fraction of links receives most clicks, often in a burst right after
being shared. That skew is what makes caching so effective here.

---

## 35.3 API and Data Model

```text
POST /v1/links                    Idempotency-Key: <uuid>
     { "url": "https://…", "alias": "spring-sale"?, "expiresAt": "2027-01-01T00:00:00Z"? }
  → 201 { "key": "LDYvwns", "shortUrl": "https://sho.rt/LDYvwns", "expiresAt": … }

GET  /:key                        → 302 Location: <target>    | 404 unknown | 410 expired or removed
GET  /v1/links/:key/stats?from=&to=&granularity=hour → { "clicks": [...], "topReferrers": [...] }
```

Creation takes an **idempotency key** (Chapter 8) so a client retrying after a timeout doesn't create two links.

**Data model.** One record per link, accessed almost exclusively by key:

| Field | Notes |
|---|---|
| `key` (PK) | 7-char base62; also the partition key |
| `target_url` | Normalised and validated at creation; up to a few KB |
| `owner_id` | API key or account |
| `created_at`, `expires_at` | Expiry checked at redirect time |
| `status` | `active`, `disabled` (abuse takedown) |

Two secondary access patterns need their own structures: "links owned by X" (an index on `owner_id`, or a separate
owner → keys table) and "does this alias exist?" (the primary key itself, via a conditional insert). Clicks are not stored
on the link record — they're an append-only event stream (§35.7).

A key-value store (DynamoDB, Cassandra) or a relational database partitioned by key both fit. With ~40 writes/s and
lookups by primary key, the choice is mostly operational; this design uses PostgreSQL partitioned by key hash at first,
because uniqueness of custom aliases and the owner index are simple there (Chapter 11's default).

---

## 35.4 High-Level Design

```text
                     ┌──────────────────────── redirect path (hot) ─────────────────────────┐
browser ─▶ CDN/edge ─┤ cache hit → 302                                                       │
                     └─▶ LB ─▶ redirect service ─▶ in-process cache ─▶ Redis ─▶ links DB (replicas)
                                     │
                                     └─▶ click event (async, fire-and-forget) ─▶ log/queue ─▶ analytics
API clients ─▶ LB ─▶ link API ─▶ key allocator ─▶ links DB (primary) ─▶ outbox ─▶ cache warm / abuse scan
```

The **redirect service** is a separate, minimal service: look up the key through a cache hierarchy, return a redirect, and
emit a click event without waiting for it. Keeping it separate from the link API means it can be scaled, deployed, and
protected independently — its availability target is higher, and it shouldn't be redeployed every time the dashboard
changes.

The **link API** validates and normalises URLs, checks them against abuse lists, allocates a key (§35.5), writes the link
with an outbox event (Chapter 23), and returns. Downstream consumers of that event pre-warm caches and run slower abuse
checks.

---

## 35.5 Deep Dive: Generating Short Keys

Four approaches, with very different properties:

| Approach | How | Collisions | Coordination | Guessable | Notes |
|---|---|---|---|---|---|
| **Hash the URL** (truncate SHA-256 to 7 base62 chars) | Same URL → same key | Yes — truncated hashes collide; must detect and resolve | None | No | Deduplicates identical URLs, which is often *undesirable* (different owners, analytics, expiry) |
| **Random key** + uniqueness check | Generate 7 random chars, insert with a conditional put, retry on conflict | Rare at this occupancy | Per insert (the conditional write) | No | Simple; collision probability grows with occupancy |
| **Global counter** + base62 | Auto-increment ID encoded | None | Every key goes through one counter | **Yes** — sequential keys reveal volume and allow enumeration (Chapter 22) | A single sequence is a bottleneck and a single point of failure |
| **Range leasing** + permutation | Servers lease blocks of IDs; each ID is scrambled by a keyed bijection, then encoded | None | One write per block (e.g. 1,000 keys) | No | Collision-free, low coordination, non-sequential — the real-world example |

**Random keys** are the simplest good answer, and the collision arithmetic shows why:

```js
// Random keys instead of a counter: how often does a new key collide with an existing one?
const space = 62 ** 7;
for (const existing of [1e8, 1e9, 6e9]) {
  const pNew = existing / space;                    // chance one fresh random key is already taken
  console.log(`${(existing / 1e9).toFixed(1)} B keys stored → ${(pNew * 100).toFixed(3)}% of new keys collide; expected tries ${(1 / (1 - pNew)).toFixed(4)}`);
}
// 0.1 B keys stored → 0.003% of new keys collide; expected tries 1.0000
// 1.0 B keys stored → 0.028% of new keys collide; expected tries 1.0003
// 6.0 B keys stored → 0.170% of new keys collide; expected tries 1.0017
```

Even after five years, fewer than two creations in a thousand need a second try — *provided* every insert is a conditional
write (`INSERT … ON CONFLICT DO NOTHING`, or a put-if-absent) so a collision is detected rather than silently overwriting
someone else's link. Note the difference from Chapter 22's birthday bound: there, the question was "will *any* two keys
ever collide"; here it's "does this new key hit an existing one", which is linear in occupancy and easily handled by a retry.

**Range leasing with a permutation** removes even those retries and the conditional-write dependency for generated keys,
at the cost of a secret key for the permutation that can never change. Either is a good interview answer; what matters is
explaining the collision handling and why sequential keys are unacceptable for a public service.

**Custom aliases** ("sho.rt/spring-sale") share the same key space. Reserve them with the same conditional insert; reject
aliases that collide with generated keys' format (e.g. require a length other than 7, or a hyphen) so the two never compete;
and block reserved words and offensive or brand-impersonating aliases.

---

## 35.6 Deep Dive: Serving Redirects Fast

At ~12,000 redirects per second — and tens of thousands for one viral link — the redirect path is a caching problem
(Chapter 16), with a few specifics.

**The cache hierarchy.** Each layer absorbs most of what reaches it:

1. **CDN/edge**: for cacheable redirects, the edge answers without reaching you. A viral link costs nothing beyond the CDN.
2. **In-process cache** in each redirect instance (an LRU of the hottest few hundred thousand keys, TTL of seconds to
   minutes): no network hop, and immune to a Redis hot key.
3. **Redis** (or similar): the shared hot set, sized from the skew (Chapter 2's 20% rule is a starting point).
4. **Database replicas**: the long tail.

**Negative caching.** Requests for keys that don't exist — typos, scanners, enumeration attempts — would all reach the
database. Cache "not found" for a short time (Chapter 16), and rate-limit sources that generate many 404s.

**Expiry and takedowns** must be visible through every cache layer. Store `expires_at` and `status` in the cached entry
and check them on every hit, so an expired link returns `410` even from cache; for takedowns, purge the key from the CDN
(Chapter 6) and publish an invalidation to every instance's in-process cache.

**301 or 302?**

| | `301 Moved Permanently` | `302 Found` (or `307`) |
|---|---|---|
| Browser caching | Browsers may cache it indefinitely | Not cached by default |
| Repeat clicks reach your servers | No — invisible to analytics | Yes |
| Server and CDN load | Lowest | Higher (mitigated by CDN caching with short TTLs) |
| Can change or disable the link later | Not for users whose browsers cached it | Yes |

Because analytics, expiry, and abuse takedowns all need clicks to reach the service, **302 with short CDN caching** is the
usual choice; a service that promises immutable, permanent links and doesn't care about repeat-click analytics could use 301.

> 💼 **Interview** — The 301 vs 302 question is a favourite follow-up. The answer interviewers want connects the status code
> to requirements: 301 lets browsers cache the redirect forever, which saves load but blinds analytics and makes takedowns
> impossible for anyone who already visited; 302 keeps every click visible.

---

## 35.7 Deep Dive: Click Analytics Without Slowing Redirects

Every redirect generates a click event — key, timestamp, referrer, user agent, coarse location from the IP. Writing it to the
database synchronously would put a write on the hottest path and turn a viral link into a hot row (Chapter 12).

Instead, the redirect service **emits the event asynchronously** — to a local buffer flushed in batches to a log such as
Kafka (Chapter 24) — and returns the redirect immediately. If the analytics pipeline is down, redirects keep working; events
are buffered, or in the worst case dropped and counted as dropped. Analytics is less important than the redirect, and the
design should say so.

Downstream:

- A **stream processor** (Chapter 25) aggregates clicks per key per minute (and per referrer and country), writing
  pre-aggregated rows to a time-series or columnar store that the stats API reads.
- **Raw events** land in object storage as partitioned Parquet for exact daily recomputation and ad-hoc analysis.
- **Bot filtering** happens in the pipeline, not on the redirect path: crawlers, link-preview fetchers, and security scanners
  inflate counts and should be classified and excluded from user-facing numbers.

Real-time counts can be approximate (at-least-once delivery may double-count slightly); daily numbers recomputed from raw
events are exact. Stating that split is both honest and a common interview point.

---

## 35.8 Failure Modes, Bottlenecks, and 10×

| Failure | Effect | Mitigation |
|---|---|---|
| Database primary down | Creation fails (~30 s during failover) | Redirects unaffected: served from caches and replicas |
| Redis down | More load on replicas | In-process caches absorb hot keys; replicas sized for cache-cold traffic; load shedding |
| Redirect region down | Links unreachable for users routed there | Multi-region redirect tier with replicated link data (read-only), anycast edge routing |
| Analytics pipeline down | Stats delayed | Redirects unaffected by design; local buffering |
| Viral link | Tens of thousands of requests/s for one key | CDN caching, in-process cache, no synchronous per-click writes |

**Abuse** is the real operational load of a public shortener: links to phishing pages and malware, spam campaigns, and
attempts to use your domain's reputation to evade filters. Defences: validate and normalise target URLs (no `javascript:` or
internal addresses — Chapter 33's SSRF list applies to any preview fetching); check targets against threat-intelligence and
safe-browsing lists at creation and periodically afterwards; rate-limit creation per API key and IP (Chapter 29); and support
instant takedown that propagates through every cache. Many shorteners show an interstitial warning page for links flagged as
suspicious.

**At 10×** — a billion links a month, 120,000 redirects per second at peak:

- Writes reach ~400/s — still modest; the key allocator's block leasing is unaffected.
- Storage grows to ~30 TB raw over five years: partition the links table by key (it's already keyed for it) across more
  nodes, or move to a managed key-value store.
- Redirect capacity scales horizontally; the CDN carries most of it. The redirect tier should become **multi-region** with
  replicated, read-only link data near users, since read traffic dominates and replication lag only delays new links by
  seconds.
- Analytics volume (billions of events per day) becomes the largest system — size the log and stream processor for it, and
  aggregate early.

---

## 35.9 Common Mistakes and Best Practices

**Mistake: sequential keys from a global counter.** They reveal volume, allow enumeration of every link, and funnel every
creation through one sequence.

**Mistake: truncated hashes without collision handling.** Two different URLs mapping to one key means one silently redirects
to the other's target.

**Mistake: unconditional writes for random keys.** A rare collision overwrites an existing link instead of retrying.

**Mistake: synchronous click logging on the redirect path.** It adds latency to every redirect and makes viral links hot rows.

**Mistake: 301 redirects with analytics or takedown requirements.** Browsers cache them; later clicks and removals never reach
you.

**Mistake: caches that ignore expiry and status.** Expired or taken-down links keep redirecting from cache.

**Mistake: no abuse controls.** A public shortener becomes a phishing and malware distribution tool, and its domain gets
blocklisted.

**Best practice: separate the redirect path from everything else.** A minimal, independently scaled and deployed service with
the highest availability target.

**Best practice: collision-free or collision-checked key generation.** Range leasing with a permutation, or random keys with
conditional inserts.

**Best practice: analytics as an asynchronous pipeline.** Redirects never wait for, or fail because of, analytics.

---

## 35.10 Real-World Example: A Range-Leased, Permuted Key Allocator

This is the hardest component of the design: key generation that is collision-free, needs only occasional coordination, and
produces keys that reveal nothing about volume or order. Each server leases blocks of 1,000 sequence numbers from a central
counter; each number is scrambled by a keyed **Feistel permutation** of the 7-character key space (with cycle-walking to stay
inside it); and the result is base62-encoded to exactly 7 characters.

```js
// key-allocator.mjs — short, unique, non-guessable keys without per-key coordination:
// 1) each server leases a block of sequence numbers from a central counter (one DB write per block);
// 2) each number is scrambled by a keyed permutation of [0, 62^7), so neighbours look unrelated;
// 3) the result is base62-encoded to exactly 7 characters. Collisions are impossible by construction.
import { createHmac } from 'node:crypto';

const ALPHABET = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz';
const LEN = 7, SPACE = 62n ** BigInt(LEN);          // ≈ 3.5 trillion keys, just under 2^42
const HALF = 21n, MASK = (1n << HALF) - 1n;

// A 4-round Feistel network over 42-bit values is a bijection on [0, 2^42) for ANY round
// function, so it's invertible without the round function itself being invertible.
const round = (secret, r, x) =>
  BigInt('0x' + createHmac('sha256', secret).update(`${r}:${x}`).digest('hex').slice(0, 8)) & MASK;
function feistel(secret, x, inverse = false) {
  let l = x >> HALF, rgt = x & MASK;
  const rounds = inverse ? [3, 2, 1, 0] : [0, 1, 2, 3];
  for (const r of rounds) {
    if (!inverse) [l, rgt] = [rgt, l ^ round(secret, r, rgt)];
    else [l, rgt] = [rgt ^ round(secret, r, l), l];
  }
  return (l << HALF) | rgt;
}
// Cycle-walking: 2^42 is a bit larger than 62^7, so re-apply until we land inside the space.
// Still a bijection on [0, 62^7), and each walk succeeds ~80% of the time.
export function scramble(secret, n) { let x = feistel(secret, BigInt(n)); while (x >= SPACE) x = feistel(secret, x); return x; }
export function unscramble(secret, x) { let n = feistel(secret, x, true); while (n >= SPACE) n = feistel(secret, n, true); return n; }

export const encode = (x) => { let s = ''; for (let i = 0; i < LEN; i++) { s = ALPHABET[Number(x % 62n)] + s; x /= 62n; } return s; };
export const decode = (s) => [...s].reduce((x, ch) => {
  const v = ALPHABET.indexOf(ch);
  if (v < 0) throw new Error('invalid key');
  return x * 62n + BigInt(v);
}, 0n);

// Range leasing: the only coordinated step is reserving a block (one UPDATE … RETURNING)
export function createAllocator({ reserveBlock, secret, blockSize = 1_000 }) {
  let next = 0n, end = 0n, pending = null;
  return async function nextKey() {
    while (next >= end) {
      pending ??= reserveBlock(blockSize).then((start) => { next = BigInt(start); end = next + BigInt(blockSize); pending = null; });
      await pending;
    }
    return encode(scramble(secret, next++));
  };
}

// ── Demo: two servers sharing one counter ──────────────────────────────────────────
const secret = 'rotate-me-never';                        // changing it would remap every key: treat as permanent
let counter = 0;
const reserveBlock = async (size) => { const start = counter; counter += size; return start; }; // atomic in a real DB
const serverA = createAllocator({ reserveBlock, secret }), serverB = createAllocator({ reserveBlock, secret });
const keys = [];
for (let i = 0; i < 3; i++) keys.push(await serverA(), await serverB());
console.log(keys.join(' '));                                     // LDYvwns 3bNLbKo BRGk5ZU 9JPyEjl NUuiBcT vD4Rgsx
console.log('decodes back to sequence numbers:', keys.map((k) => unscramble(secret, decode(k))).join(' '));
// decodes back to sequence numbers: 0 1000 1 1001 2 1002
const many = new Set(), alloc = createAllocator({ reserveBlock, secret });
for (let i = 0; i < 50_000; i++) many.add(await alloc());
console.log('50,000 keys, unique:', many.size === 50_000, '| blocks reserved so far:', counter / 1_000);
// 50,000 keys, unique: true | blocks reserved so far: 52
```

Consecutive sequence numbers — even from the same server — produce unrelated-looking keys, and decoding a key recovers its
sequence number, which is useful internally (for example, to find which block and server issued it). In SQL, `reserveBlock`
is `UPDATE key_counter SET next = next + $1 RETURNING next - $1`, one row-locked statement per thousand keys.

The decision to defend is **scrambling with a keyed permutation instead of a simpler encoding of the counter**. Encoding the
sequence number directly (or multiplying it by a constant) produces keys that differ predictably — consecutive links share
prefixes or step by a fixed amount — so anyone can estimate volume from two links and enumerate others by guessing nearby
keys, which turns "unlisted" links (shared documents, private invites) into public ones. Random keys avoid that but need a
conditional write and occasional retries for every creation. A Feistel permutation keeps the counter's guarantees —
uniqueness by construction, coordination only per block — while giving keys that are as unpredictable as random ones to
anyone without the secret. The price is that the secret is permanent: changing it would remap every key ever issued, so it's
stored like a root encryption key, never rotated, and never logged.

---

## 35.11 Questions

**1.** MCQ — Why are sequential short keys a problem for a public URL shortener? a) They're longer b) They reveal volume and let anyone enumerate other links c) They collide d) They can't be cached
<details><summary>Answer</summary>**b)** Consecutive keys expose growth and make neighbouring (possibly private) links guessable.</details>

**2.** MCQ — What happens if random keys are inserted with an unconditional write? a) Nothing b) A rare collision silently overwrites an existing link c) Inserts are slower d) Keys get longer
<details><summary>Answer</summary>**b)** The insert must be conditional (put-if-absent) so a collision is detected and retried.</details>

**3.** MCQ — Why choose 302 over 301 for redirects? a) 302 is faster b) 301 lets browsers cache the redirect, hiding later clicks from analytics and making takedowns ineffective for past visitors c) 301 is deprecated d) 302 is permanent
<details><summary>Answer</summary>**b)** 302 keeps every click visible to the service.</details>

**4.** MCQ — Where should click events be recorded? a) Synchronously in the links table b) Asynchronously to a log, aggregated downstream c) In the browser only d) Nowhere
<details><summary>Answer</summary>**b)** Off the redirect's critical path, so redirects stay fast and available even if analytics isn't.</details>

**5.** MCQ — Hashing the URL to produce the key has which drawback? a) It requires a counter b) Identical URLs from different owners share one link, and truncated hashes still collide c) It's too slow d) Keys become sequential
<details><summary>Answer</summary>**b)** Deduplication merges owners, expiry, and analytics, and truncation makes collisions possible.</details>

**6.** MCQ — What's the main purpose of negative caching in the redirect service? a) Caching redirects to the error page b) Absorbing lookups of non-existent keys (typos, scanners) so they don't all reach the database c) Faster creation d) Analytics
<details><summary>Answer</summary>**b)** Many requests are for keys that don't exist; caching "not found" briefly protects the database.</details>

**7.** Short — Why is the redirect path a separate service from link creation?
<details><summary>Answer</summary>It has a higher availability target, a much higher request rate, and a minimal job; separating it lets it scale, deploy, and be protected independently, so changes to creation or dashboards can't take redirects down.</details>

**8.** Short — How do expiry and takedowns stay correct when redirects are cached in several layers?
<details><summary>Answer</summary>Each cached entry includes `expires_at` and `status`, checked on every hit, and takedowns purge the CDN and broadcast invalidations to in-process caches; short TTLs bound any remaining staleness.</details>

**9.** Short — Why are real-time click counts approximate while daily counts are exact?
<details><summary>Answer</summary>The real-time path uses at-least-once delivery and streaming aggregation (possible duplicates, late events); the daily path recomputes from immutable raw events with deduplication and complete data.</details>

**10.** Short — How do custom aliases share the key space safely with generated keys?
<details><summary>Answer</summary>They're reserved with the same conditional insert, and constrained to a format generated keys can never take (different length or characters), so the two never compete; reserved and abusive words are blocked.</details>

**11.** Short — Why does range leasing need only occasional coordination?
<details><summary>Answer</summary>A server reserves a block (say 1,000 numbers) with one atomic update, then issues keys from it locally; coordination happens once per block instead of once per key, and blocks never overlap.</details>

**12.** Output — Using §35.2's model with 300 million new links per month, what's the average write rate, to the nearest whole number?
<details><summary>Answer</summary>**~116/s.** 300,000,000 / 2,592,000 s ≈ 115.7.</details>

**13.** Output — Using §35.5's reasoning, with 2 trillion keys already stored in the 62⁷ space, what fraction of new random keys collide?
<details><summary>Answer</summary>**~57%.** 2 × 10¹² / 3.52 × 10¹² ≈ 0.57 — at that occupancy random keys need many retries, and it's time for 8-character keys.</details>

**14.** Output — What does `encode(0n)` return with §35.10's `encode`, and why is that a problem if used directly as a key?
<details><summary>Answer</summary>**`'0000000'`.** Unscrambled small numbers produce obviously sequential keys — which is why each number is permuted before encoding.</details>

**15.** Output — With blocks of 1,000 keys and 40 creations per second spread across 8 servers, how many block reservations per second hit the counter?
<details><summary>Answer</summary>**0.04/s** — one every 25 seconds. 40 / 1,000; the counter row is nowhere near a bottleneck.</details>

**16.** Debug — A viral link pushed the links database to 100% CPU, although the database holds only one row for it. Redirects for that link went through Redis, which also showed one hot shard. Cause and fix?
<details><summary>Answer</summary>A hot key: every redirect for one link hit the same Redis shard and, on misses or expiries, the database. Add an in-process LRU in redirect instances, cache redirects at the CDN with short TTLs, and use stampede protection (single-flight) so expiries don't fan out to the database.</details>

**17.** Debug — A link reported as phishing was disabled in the database within minutes, but users kept being redirected to the phishing site for hours. Why?
<details><summary>Answer</summary>Redirects were cached — at the CDN, in in-process caches, and possibly in browsers if 301 was used — and takedowns didn't propagate. Purge the key at the CDN, broadcast invalidations to instances, check status on every cache hit, keep TTLs short, and use 302.</details>

**18.** Debug — The analytics team notices click counts for some links are roughly double the referring site's own numbers. What are likely causes?
<details><summary>Answer</summary>Bots and link-preview fetchers (chat apps and social sites fetch links to build previews), security scanners, and at-least-once duplicates in the event pipeline. Classify and exclude non-human user agents and known scanner IPs, and deduplicate events by ID in the daily recomputation.</details>

**19.** Debug — Creation latency spiked to seconds during a traffic burst. Profiling shows every creation waiting on `SELECT nextval(...)` from one sequence on the primary, and the primary is far away from most API servers. What would you change?
<details><summary>Answer</summary>Each key costs a round trip to one central sequence. Lease blocks of numbers per server (one round trip per thousand keys), or switch to random keys with conditional inserts so creation needs no central counter.</details>

**20.** Code — Write `normaliseUrl(input)` that rejects non-HTTP(S) schemes and credentials in URLs, lower-cases the host, and removes the default port.
<details><summary>Answer</summary>

```js
function normaliseUrl(input) {
  const u = new URL(input);                                   // throws on malformed input
  if (!['http:', 'https:'].includes(u.protocol)) throw new Error('only http(s) URLs');
  if (u.username || u.password) throw new Error('credentials in URL not allowed');
  u.hostname = u.hostname.toLowerCase();
  if ((u.protocol === 'https:' && u.port === '443') || (u.protocol === 'http:' && u.port === '80')) u.port = '';
  return u.toString();
}
console.log(normaliseUrl('HTTPS://Example.COM:443/a?b=1')); // https://example.com/a?b=1
```
</details>

**21.** Code — Write `createLink(db, { url, owner, alias })` that inserts with a conditional write, generating a random 7-character key and retrying up to 5 times on conflict, but failing immediately if a requested alias is taken.
<details><summary>Answer</summary>

```js
import { randomInt } from 'node:crypto';
const ALPHABET = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz';
const randomKey = () => Array.from({ length: 7 }, () => ALPHABET[randomInt(62)]).join('');

async function createLink(db, { url, owner, alias }) {
  for (let attempt = 1; attempt <= (alias ? 1 : 5); attempt++) {
    const key = alias ?? randomKey();
    const { rowCount } = await db.query(
      'INSERT INTO links (key, target_url, owner_id) VALUES ($1, $2, $3) ON CONFLICT (key) DO NOTHING', [key, url, owner]);
    if (rowCount === 1) return key;
    if (alias) throw Object.assign(new Error('alias taken'), { status: 409 });
  }
  throw new Error('could not allocate a key'); // effectively impossible at sane occupancy — alert if it happens
}
```
</details>

**22.** Code — Write `redirectResponse(entry, now)` that returns `{ status, headers }` for a cached link entry `{ target, status, expiresAt }` or `null` (unknown key).
<details><summary>Answer</summary>

```js
function redirectResponse(entry, now = Date.now()) {
  if (!entry) return { status: 404, headers: { 'Cache-Control': 'public, max-age=30' } };      // negative cache briefly
  if (entry.status !== 'active') return { status: 410, headers: { 'Cache-Control': 'no-store' } };
  if (entry.expiresAt && Date.parse(entry.expiresAt) <= now) return { status: 410, headers: { 'Cache-Control': 'no-store' } };
  // Short edge caching keeps viral links cheap while takedowns still take effect quickly
  return { status: 302, headers: { Location: entry.target, 'Cache-Control': 'public, max-age=60' } };
}
console.log(redirectResponse({ target: 'https://example.com', status: 'active' }).status, redirectResponse(null).status); // 302 404
```
</details>

**23.** Scenario — Product wants links to be editable: the owner can change a link's target after creation. What changes in the design?
<details><summary>Answer</summary>Caching gets harder: edits must invalidate the CDN, in-process, and Redis caches promptly (CDN purge plus broadcast invalidation), and 301s become unacceptable. Store target versions with timestamps for auditing, rate-limit edits, and re-run abuse checks on each new target — editability is a classic abuse vector (create a benign link, get it whitelisted, then swap the target). Consider showing an interstitial for recently changed links.</details>

**24.** Scenario — A large enterprise customer wants their own branded domain (`go.acme.com/xyz`) and guarantees that their links are never served from a shared cache with other customers' links. How do you support it?
<details><summary>Answer</summary>Map custom domains to tenants at the edge (TLS certificates issued per domain, e.g. via ACME automation), include the host in every cache key so keys are scoped per domain, and store links under `(domain, key)` so the same key can exist on different domains. For strict isolation, give the tenant dedicated cache namespaces and, if contractually required, a dedicated redirect deployment — with the same code and configuration pipeline as the shared one.</details>

---

## Chapter Summary

- A URL shortener is a very read-heavy key-lookup service: tens of writes per second, tens of thousands of redirects per
  second, and billions of links that must never be lost.
- Generate keys collision-free (range leasing with a keyed permutation) or collision-checked (random keys with conditional
  inserts) — never sequential, never unconditional, never truncated hashes without resolution.
- Serve redirects through a cache hierarchy — CDN, in-process LRU, Redis, replicas — with negative caching, and expiry and
  status checked on every hit.
- Use 302 with short edge caching when analytics and takedowns matter; 301 trades both away for load.
- Record clicks asynchronously into a pipeline that aggregates for near-real-time stats and recomputes exact daily numbers,
  filtering bots.
- Treat abuse as a core requirement: validate targets, scan and rate-limit creation, and make takedowns propagate through
  every cache; at 10×, partition storage and take the redirect tier multi-region.

---

## Practice Exercises

1. Implement the redirect service with an in-process LRU, Redis, and PostgreSQL, and load-test it with a Zipf-distributed
   key workload; report hit ratios per layer.
2. Measure how quickly a takedown propagates through your caches, and reduce it below 60 seconds.
3. Read the documentation of a safe-browsing or threat-intelligence API and design how creation-time and periodic re-checks
   would use it within its rate limits.
4. Compare random-key creation with conditional inserts against the §35.10 allocator under 1,000 concurrent creations:
   latency, database load, and failure behaviour when the database is briefly unavailable.
5. Build the click pipeline: redirect service → local batch buffer → Kafka (or a file log) → per-minute aggregator, with a bot
   filter and a daily exact recomputation.

---

## Mini Project: A Production-Shaped Link Service

Build a URL shortener with the separation and protections described in this chapter, deployable on a laptop with containers.

- Link API with idempotency keys, URL normalisation and validation, custom aliases, expiry, and the §35.10 key allocator.
- A separate redirect service with layered caching, negative caching, status/expiry checks on every hit, and 302 responses
  with edge-friendly cache headers.
- Asynchronous click events to a log, a per-minute aggregator, and a stats endpoint.
- Abuse controls: a blocklist check at creation, per-key creation rate limits, and a takedown endpoint that invalidates all
  cache layers.
- A load test with a viral link and a chaos test that stops Redis and the analytics pipeline, showing redirects stay within
  their SLO.

---

**Next:** [Chapter 36 — Design a News Feed →](36-news-feed.md)
