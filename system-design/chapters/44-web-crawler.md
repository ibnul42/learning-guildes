# Chapter 44 — Design a Web Crawler

## Learning Objectives

- Scope a large-scale crawler and size fetch rate, bandwidth, storage, and the URL frontier
- Design a frontier that is polite by construction — per-host queues and delays — while prioritising important URLs
- Deduplicate URLs with normalisation and Bloom filters, and content with exact and near-duplicate fingerprints
- Respect robots.txt, avoid crawler traps, and schedule recrawls by how often pages change
- Distribute crawling across many workers without double-fetching or overloading any site

---

## 44.1 Requirements and Scope

A web crawler downloads pages from the web — starting from seed URLs, extracting links, and following them — to feed a search
index, an archive, or a dataset. The core loop is three lines long; the difficulty is doing it for billions of pages while being
a **good citizen** of other people's servers, avoiding infinite traps, and not wasting effort on duplicates.

**Functional requirements**

- Fetch pages starting from seeds; extract and follow links.
- Store fetched content (HTML) with metadata for downstream indexing.
- Recrawl pages to keep content fresh, more often for pages that change often.

**Out of scope**: indexing and ranking (Chapter 18), rendering JavaScript-heavy pages (mentioned in §44.8), non-HTML content
extraction beyond basics.

**Non-functional requirements**

| Requirement | Target |
|---|---|
| Politeness | Never more than one concurrent request per host by default; honour robots.txt and crawl delays; back off when a site is slow or erroring |
| Throughput | ~5 billion pages per month |
| Freshness | Frequently changing pages (news front pages) recrawled within hours; static pages within weeks |
| Efficiency | No URL fetched twice per crawl cycle; near-duplicate content detected |
| Robustness | Survive malformed HTML, infinite URL spaces, huge files, slow servers, and malicious content |

Politeness is listed first deliberately. A crawler that overloads sites gets blocked, harms the sites, and can be a legal problem; every
throughput goal is subject to it.

---

## 44.2 Estimation

```js
// Crawler: the numbers behind fetch capacity, bandwidth, and storage
const pagesPerMonth = 5e9, avgPageKB = 100, compressRatio = 0.25;
const pagesPerSec = pagesPerMonth / (30 * 86_400);
console.log(`~${Math.round(pagesPerSec).toLocaleString('en-US')} pages/s | ~${((pagesPerSec * avgPageKB * 8) / 1e6).toFixed(1)} Gbit/s download | ~${((pagesPerMonth * avgPageKB * compressRatio) / 1e9).toFixed(0)} TB/month stored compressed`);
// Politeness bounds per-host throughput: a 1 s delay means 1 page/s per host, however many workers there are
const concurrentFetches = pagesPerSec * 0.8;               // ~800 ms average fetch latency (Little's law)
console.log(`~${Math.round(concurrentFetches).toLocaleString('en-US')} fetches in flight; at 1 page/s per host, needs ≥ ${Math.round(pagesPerSec).toLocaleString('en-US')} hosts active at once`);
// ~1,929 pages/s | ~1.5 Gbit/s download | ~125 TB/month stored compressed
// ~1,543 fetches in flight; at 1 page/s per host, needs ≥ 1,929 hosts active at once
```

What the numbers decide:

- **~2,000 pages per second** with ~1,500 in flight is an I/O-bound workload: asynchronous fetchers handle thousands of concurrent connections
  each, so a modest fleet suffices. Parsing and link extraction are the CPU cost.
- **Politeness, not workers, limits throughput.** At one request per second per host, the crawler must always have thousands of different hosts with
  work ready; a frontier dominated by a few huge sites would stall. The frontier's job is to keep *many hosts* busy, not many workers.
- **The URL frontier is huge**: billions of discovered URLs waiting to be fetched, which must live on disk with an in-memory working set.
- **DNS** is a hidden bottleneck: ~2,000 lookups per second against public resolvers would be slow and rude, so the crawler runs its own caching resolver.

---

## 44.3 Data Model

| Data | Store | Notes |
|---|---|---|
| **Frontier** | Disk-backed queues per host, with an in-memory index of ready hosts | Billions of URLs; hosts sharded across crawler nodes (§44.5) |
| **Seen URLs** | Bloom filter in memory + persistent set | "Have we already queued this URL?" for billions of URLs (§44.6) |
| **Page store** | Object storage, compressed, batched into large files (WARC-style) | Content plus headers, fetch time, status |
| **Page metadata** | Wide-column store keyed by URL | Last fetch, ETag/Last-Modified, content hash, change history, next fetch time |
| **Content fingerprints** | Hash index (exact) + SimHash index (near-duplicate) | Skip storing and re-indexing duplicates |
| **Host state** | Key-value store | robots.txt rules and expiry, crawl delay, error rates, response times, IP |

Storing pages as **many small objects** would be expensive (Chapter 17's request-cost trap); crawlers batch fetched pages into large, compressed
archive files and index them by URL and offset.

---

## 44.4 High-Level Design

```text
seeds ─▶ frontier (per-host queues, politeness heap, priorities) ◀───────────────────────┐
              │ next ready URL                                                           │
              ▼                                                                          │
         fetchers (async, thousands of connections) ─▶ DNS cache ─▶ robots cache          │
              │ response                                                                 │
              ▼                                                                          │
         content processing: size/type checks → exact & near-dup check → store (archive) │
              │                                                                          │
              ▼                                                                          │
         link extraction → normalise → filter (traps, scope) → seen-URL check ───────────┘
              │
              └─▶ metadata store (change history, next recrawl time) ─▶ indexer pipeline
```

Each crawler node owns a **set of hosts** (assigned by hashing the hostname), including their frontier queues, politeness state, and robots rules.
A discovered URL for a host owned by another node is sent to that node. This keeps every per-host decision — "may I fetch this host now?" — local to one
node, so no coordination is needed to enforce politeness.

---

## 44.5 Deep Dive: The Frontier

The frontier decides what to fetch next. A single global priority queue is wrong for two reasons: it can't enforce per-host politeness (the top 1,000
URLs may all be on one site), and it can't express that some URLs matter more than others within politeness constraints.

The classic solution (from the Mercator crawler) has two parts:

- **Front queues for priority.** URLs are scored — by estimated importance (link-based rank, site quality), freshness need, and depth — and higher-priority
  URLs are selected first.
- **Back queues for politeness.** Each host has its own FIFO queue, and a **heap of hosts keyed by the earliest time each may be fetched again**. A fetcher
  takes the host at the top of the heap if its time has come, fetches one URL, and puts the host back with its next allowed time (now + its delay).

The real-world example in §44.10 implements the back-queue machinery with priorities within each host. Properties that follow:

- **Politeness is structural**: a host can't be fetched faster than its delay, no matter how many workers there are.
- **Many hosts stay busy**: while one host waits out its delay, others are ready, which is what keeps throughput high.
- **Delays adapt per host**: robots.txt `Crawl-delay`, slower responses, or rising error rates increase a host's delay; healthy, fast hosts that allow it can
  be crawled a little faster (or with a few concurrent connections for very large sites that permit it).

---

## 44.6 Deep Dive: Deduplication

The web is full of duplicates at two levels.

**Duplicate URLs.** The same page appears under many URLs: `http` and `https`, `www.` and not, trailing slashes, reordered query parameters, tracking
parameters, session IDs, fragments. **Normalisation** maps them to one canonical form (Chapter 6's cache-key normalisation is the same idea), and the page's
own `<link rel="canonical">` can collapse further. Then the **seen-URL check** — "have we already queued this?" — runs for every extracted link: billions of
lookups. A **Bloom filter** (Chapter 10) answers "definitely not seen" from memory in a few bits per URL, so only "probably seen" answers need the persistent
store; a false positive merely skips a URL that wasn't actually crawled, which is acceptable at a low rate.

**Duplicate content.** Different URLs often serve the same or nearly the same content: mirrors, syndicated articles, printer-friendly versions, pages that
differ only in a date, an ad, or a session token. An exact hash catches identical content; **near-duplicates** need a similarity-preserving fingerprint.
**SimHash** produces a 64-bit fingerprint where similar documents differ in few bits:

```js
// SimHash: a 64-bit fingerprint where similar documents get fingerprints differing in few bits,
// so near-duplicates (the same article with a different ad, date, or session ID) are detectable.
import { createHash } from 'node:crypto';

function simhash(text) {
  const weights = new Array(64).fill(0);
  const words = text.toLowerCase().match(/[a-z0-9]+/g) ?? [];
  for (let i = 0; i + 2 < words.length; i++) {             // 3-word shingles capture local word order
    const h = createHash('md5').update(words.slice(i, i + 3).join(' ')).digest();
    for (let bit = 0; bit < 64; bit++) {
      weights[bit] += (h[bit >> 3] >> (bit & 7)) & 1 ? 1 : -1;  // each shingle votes on every bit
    }
  }
  return weights.reduce((fp, w, bit) => (w > 0 ? fp | (1n << BigInt(bit)) : fp), 0n);
}
const hamming = (a, b) => { let x = a ^ b, n = 0; while (x) { n += Number(x & 1n); x >>= 1n; } return n; };

const article = 'The city council approved the new transit plan on Tuesday after a long debate about funding for buses trams and cycle lanes across the region';
const sameWithNoise = article + ' Advertisement subscribe now for updates';
const different = 'Local football club wins the regional championship after a dramatic penalty shootout in front of a record crowd on Sunday evening';
console.log('near-duplicate distance:', hamming(simhash(article), simhash(sameWithNoise)));
console.log('different article distance:', hamming(simhash(article), simhash(different)));
// near-duplicate distance: 10
// different article distance: 37
```

The near-duplicate differs in 10 bits; the unrelated article in 37 — close to the 32 expected for random fingerprints. Real pages are much longer than these
snippets, so near-duplicates differ in very few bits, and the threshold is tuned on labelled data. Finding all fingerprints within a few bits of a new one
among billions uses tables keyed by blocks of the fingerprint's bits (if two fingerprints differ in at most *k* bits, some block of bits must match exactly).

---

## 44.7 Deep Dive: Politeness, Robots, Traps, and Freshness

**robots.txt.** Before crawling a host, fetch and cache its `robots.txt` (with an expiry — typically hours to a day) and obey its rules for your user agent:
disallowed paths, crawl delays, and sitemaps. Identify the crawler honestly in the `User-Agent` with a URL explaining it and a contact address, so site
operators can reach you instead of blocking you.

**Adaptive rate.** Treat the server's response as feedback: rising latency or `429`/`503` responses mean slow down (with `Retry-After` honoured, Chapter 29);
consistent fast responses permit the default rate. Many crawlers also cap total load per IP address, since many hosts may share one server.

**Crawler traps** are URL spaces that never end:

- Calendars with "next month" links forever; faceted search with endless combinations of filters.
- Session IDs or timestamps in URLs that make every link unique.
- Deliberately malicious link farms, and pages that generate infinite unique URLs.

Defences: limits on URL length, path depth, and query-parameter count; per-host budgets of pages per crawl cycle; detecting repeating path segments
(`/a/b/a/b/a/b`); parameter normalisation that drops session-like parameters; and near-duplicate detection, which notices that the "new" pages are all the
same.

**Fetch safety**: cap response sizes and download time, check content types before downloading bodies, decompress with limits (decompression bombs), and
parse untrusted HTML in sandboxed, resource-limited workers.

**Freshness.** Pages change at very different rates. Track each page's change history (did the content hash change since the last fetch?) and estimate its
change rate; schedule the next fetch accordingly — sooner for pages that change often and matter, later for static ones. Use **conditional requests**
(`If-None-Match` / `If-Modified-Since`, Chapter 6) so unchanged pages cost a `304` instead of a full download, and use sitemaps' `lastmod` hints where sites
provide them. Recrawl budget is finite, so freshness is a prioritisation problem: spend fetches where they buy the most freshness for the most important
pages.

> 💼 **Interview** — Crawler interviews reward three ideas above all: per-host queues with politeness delays (not a single global queue), URL normalisation
> plus a Bloom filter for seen URLs and SimHash for near-duplicate content, and explicit trap defences. Mentioning a local DNS cache and conditional
> requests for recrawls shows operational depth.

---

## 44.8 Failure Modes, Bottlenecks, and 10×

| Failure | Effect | Mitigation |
|---|---|---|
| Crawler node dies | Its hosts stop being crawled; in-flight fetches lost | Frontier queues on durable storage; host ownership reassigned via consistent hashing (Chapter 14); lost fetches simply re-queued |
| A huge site dominates the frontier | Throughput collapses to that site's politeness limit | Per-host budgets; frontier keeps thousands of hosts ready |
| DNS resolver slow | Fetchers idle | Local caching resolvers, prefetching DNS for queued hosts, respecting DNS TTLs |
| Trap explosion | Frontier fills with junk URLs | Depth, length, and per-host limits; near-duplicate feedback lowers the host's priority |
| Being blocked by sites | Coverage loss | Honest user agent, robots compliance, adaptive rates, a contact channel |

**JavaScript-rendered pages** need a headless browser to see their content, which is orders of magnitude more expensive per page. Render selectively: only
for hosts known to need it, with a separate, smaller rendering fleet and its own queue.

**At 10×** — 50 billion pages per month: fetching scales out with more nodes and hosts per node; the frontier and seen-URL structures grow past memory and
become the main engineering effort (tiered queues on disk, sharded Bloom filters); storage grows to petabytes per month, so retention policy and
deduplication matter more; and freshness becomes the dominant optimisation, since the budget can't recrawl everything often.

---

## 44.9 Common Mistakes and Best Practices

**Mistake: a single global priority queue.** It concentrates fetches on a few hosts, violating politeness and stalling throughput.

**Mistake: ignoring robots.txt or crawl delays.** Sites get overloaded, and the crawler gets blocked — or worse.

**Mistake: no URL normalisation.** The same page is fetched many times under trivially different URLs.

**Mistake: exact-hash-only content deduplication.** Pages that differ only in ads, dates, or session tokens are stored and indexed repeatedly.

**Mistake: no trap limits.** Calendars and session URLs fill the frontier with infinite junk.

**Mistake: using public DNS resolvers at crawl rates.** Lookups become a bottleneck and an abuse of a shared service.

**Mistake: refetching unchanged pages in full.** Conditional requests turn most recrawls into cheap `304`s.

**Best practice: per-host queues and a ready-time heap.** Politeness enforced by structure, with many hosts busy at once.

**Best practice: host ownership by consistent hashing.** All decisions about a host are made on one node, with no coordination.

**Best practice: schedule recrawls by observed change rate and importance.** Spend the fetch budget where freshness matters.

---

## 44.10 Real-World Example: A Polite Crawl Frontier

This is the back-queue half of a Mercator-style frontier, with priorities within each host: one queue per host, a min-heap of hosts ordered by the earliest
time each may be fetched again, per-host delays that can be adapted (from robots.txt or server behaviour), and deduplication of URLs on insertion. A real
crawler persists the queues and replaces the `Set` with a Bloom filter backed by a store.

```js
// frontier.mjs — a crawl frontier that is polite by construction: one FIFO queue per host, a heap of
// hosts ordered by the earliest time each may be fetched again, and priority by URL score within a host.

class MinHeap {                                         // small binary heap of { at, host }
  #a = [];
  push(x) { this.#a.push(x); this.#up(this.#a.length - 1); }
  pop() { const top = this.#a[0], last = this.#a.pop(); if (this.#a.length) { this.#a[0] = last; this.#down(0); } return top; }
  peek() { return this.#a[0]; }
  get size() { return this.#a.length; }
  #up(i) { while (i > 0) { const p = (i - 1) >> 1; if (this.#a[p].at <= this.#a[i].at) break; [this.#a[p], this.#a[i]] = [this.#a[i], this.#a[p]]; i = p; } }
  #down(i) { for (;;) { const l = 2 * i + 1, r = l + 1; let m = i;
    if (l < this.#a.length && this.#a[l].at < this.#a[m].at) m = l;
    if (r < this.#a.length && this.#a[r].at < this.#a[m].at) m = r;
    if (m === i) return; [this.#a[m], this.#a[i]] = [this.#a[i], this.#a[m]]; i = m; } }
}

export class Frontier {
  #hosts = new Map();          // host → { urls: [{ url, score }], delayMs, scheduled }
  #ready = new MinHeap();      // hosts with work, keyed by when they may next be fetched
  #seen = new Set();           // in production: a Bloom filter in front of a persistent store (§44.6)

  add(url, { score = 0, defaultDelayMs = 1_000, now = 0 } = {}) {
    if (this.#seen.has(url)) return false;
    this.#seen.add(url);
    const host = new URL(url).host;
    let h = this.#hosts.get(host);
    if (!h) this.#hosts.set(host, (h = { urls: [], delayMs: defaultDelayMs, nextAt: now, scheduled: false }));
    h.urls.push({ url, score });
    h.urls.sort((a, b) => b.score - a.score);           // best URLs of a host first
    if (!h.scheduled) { h.scheduled = true; this.#ready.push({ at: h.nextAt, host }); }
    return true;
  }

  // Returns the next URL that may be fetched at `now`, or the time to wait until one is allowed
  next(now) {
    const top = this.#ready.peek();
    if (!top) return { done: true };
    if (top.at > now) return { waitUntil: top.at };      // every host is still in its politeness delay
    this.#ready.pop();
    const h = this.#hosts.get(top.host);
    const { url } = h.urls.shift();
    h.nextAt = now + h.delayMs;                           // this host can't be hit again until then
    if (h.urls.length) this.#ready.push({ at: h.nextAt, host: top.host }); else h.scheduled = false;
    return { url };
  }

  // Adapt per host: honour robots.txt Crawl-delay, back off when the server is slow or erroring
  setDelay(host, ms) { const h = this.#hosts.get(host); if (h) h.delayMs = ms; }
}

// ── Demo: a big site and two small ones, 1 s politeness delay ─────────────────────────
const f = new Frontier();
for (let i = 1; i <= 4; i++) f.add(`https://big.example/p${i}`, { score: 10 - i });
f.add('https://small.example/a'); f.add('https://other.example/x'); f.add('https://big.example/p1'); // duplicate
f.setDelay('other.example', 3_000);                       // its robots.txt asks for 3 s
const log = [];
for (let t = 0, fetched = 0; fetched < 6; ) {
  const r = f.next(t);
  if (r.done) break;
  if (r.waitUntil !== undefined) { t = r.waitUntil; continue; }   // a real crawler would serve other work meanwhile
  log.push(`t=${t / 1000}s ${r.url.replace('https://', '')}`); fetched++;
}
console.log(log.join('\n'));
// t=0s big.example/p1
// t=0s other.example/x
// t=0s small.example/a
// t=1s big.example/p2
// t=2s big.example/p3
// t=3s big.example/p4
```

All three hosts are fetched immediately, then `big.example` is fetched at exactly one page per second in score order, however much of its work is queued;
the duplicate URL is rejected on insertion.

The decision to defend is **organising the frontier around hosts rather than around URLs** — a heap of hosts keyed by their next allowed fetch time, each with
its own queue — instead of one global priority queue of URLs with a politeness check at fetch time. The global queue is simpler and puts the most important URL
first. But when a large, important site dominates the top of that queue, the politeness check rejects almost every URL it offers: workers spin on URLs they're
not allowed to fetch, throughput collapses to that one site's allowed rate, and URLs from thousands of other hosts wait behind it. Keying the scheduler by host
makes "is this host allowed now?" the first question rather than the last, so every pop yields a fetch that's both permitted and useful, and throughput scales
with the number of hosts that have work — which is the real parallelism of the web.

---

## 44.11 Questions

**1.** MCQ — What limits a polite crawler's throughput more than the number of workers? a) CPU b) Per-host politeness delays, so throughput depends on how many hosts have work ready c) Disk d) DNS only
<details><summary>Answer</summary>**b)** At one request per second per host, thousands of hosts must be ready at once.</details>

**2.** MCQ — What structure enforces politeness in a Mercator-style frontier? a) A global FIFO b) Per-host back queues and a heap of hosts ordered by next allowed fetch time c) A Bloom filter d) A hash ring
<details><summary>Answer</summary>**b)** A host can't be selected before its delay expires.</details>

**3.** MCQ — What does a Bloom filter do in a crawler? a) Stores pages b) Answers "definitely not seen" for URLs from memory, avoiding most lookups in the persistent seen-set c) Detects near-duplicate content d) Parses HTML
<details><summary>Answer</summary>**b)** A false positive just skips a URL occasionally.</details>

**4.** MCQ — Which technique detects pages that differ only in an ad or a date? a) MD5 of the content b) SimHash with a small Hamming-distance threshold c) URL normalisation d) robots.txt
<details><summary>Answer</summary>**b)** Exact hashes change completely on any difference; SimHash changes a few bits.</details>

**5.** MCQ — Why assign hosts to crawler nodes by hashing the hostname? a) Security b) All politeness and robots decisions for a host are made on one node, without coordination c) Faster DNS d) Load balancing only
<details><summary>Answer</summary>**b)** Consistent hashing also limits reassignment when nodes join or leave.</details>

**6.** MCQ — Which is a crawler trap defence? a) More workers b) Limits on URL length, path depth, and pages per host per cycle c) Ignoring robots.txt d) Longer timeouts
<details><summary>Answer</summary>**b)** Infinite URL spaces must be bounded explicitly.</details>

**7.** Short — Why is a single global priority queue a poor frontier?
<details><summary>Answer</summary>Its top is often dominated by a few large sites; politeness then rejects most of what it offers, throughput collapses to those sites' rates, and other hosts starve.</details>

**8.** Short — Name four URL normalisations.
<details><summary>Answer</summary>Any four of: lower-casing scheme and host; removing default ports and fragments; sorting query parameters; dropping tracking and session parameters; resolving `.` and `..` path segments; consistent trailing slashes; following `rel="canonical"`.</details>

**9.** Short — How do conditional requests reduce recrawl cost?
<details><summary>Answer</summary>Sending `If-None-Match` or `If-Modified-Since` with the stored validators lets an unchanged page return `304 Not Modified` with no body, so recrawling static pages costs a round trip instead of a download.</details>

**10.** Short — How should a crawler adapt its rate to a server?
<details><summary>Answer</summary>Start from robots.txt's crawl delay (or a default), increase the delay when responses slow or `429`/`503` appear (honouring `Retry-After`), and cap total load per IP address since many hosts share servers.</details>

**11.** Short — Why does a crawler run its own DNS cache?
<details><summary>Answer</summary>Thousands of lookups per second against public resolvers would be slow and abusive; a local caching resolver (respecting TTLs, prefetching for queued hosts) keeps fetchers busy and avoids becoming a bottleneck.</details>

**12.** Output — Using §44.10's frontier, if `small.example` had three URLs (`a`, `b`, `c`, equal scores) and a 2-second delay, at what times would they be fetched in the demo?
<details><summary>Answer</summary>**0 s, 2 s, and 4 s.** Each fetch pushes the host's next allowed time 2 seconds later.</details>

**13.** Output — What is the Hamming distance between the 4-bit fingerprints `0b1011` and `0b0010`?
<details><summary>Answer</summary>**2.** XOR gives `0b1001`, which has two set bits.</details>

**14.** Output — Using §44.2's model with 20 billion pages per month and 60 KB per page, what's the download bandwidth in Gbit/s?
<details><summary>Answer</summary>**~3.7 Gbit/s.** 20 × 10⁹ / 2,592,000 ≈ 7,716 pages/s × 60 KB × 8 ≈ 3.7 Gbit/s.</details>

**15.** Output — A Bloom filter for 10 billion seen URLs at a 1% false-positive rate needs about 9.6 bits per URL. How much memory is that?
<details><summary>Answer</summary>**~12 GB.** 10 × 10⁹ × 9.6 bits ≈ 9.6 × 10¹⁰ bits = 1.2 × 10¹⁰ bytes — sharded across crawler nodes.</details>

**16.** Debug — Throughput dropped from 2,000 to 300 pages/s after a large news site was added to the seeds. Workers report mostly "host not ready". Cause and fix?
<details><summary>Answer</summary>The frontier is URL-centric, so the new site's many high-priority URLs dominate it and politeness blocks them. Switch to per-host queues with a ready-time heap, add per-host budgets, and ensure the frontier keeps thousands of hosts with work ready.</details>

**17.** Debug — A site operator complains that the crawler fetched 2 million URLs from their site in a day, almost all `/calendar?month=…` pages. What went wrong?
<details><summary>Answer</summary>A crawler trap: an infinite URL space. Add per-host page budgets, limits on query-parameter variety and path depth, parameter normalisation, and near-duplicate feedback that demotes hosts producing near-identical pages; respond to the operator and honour their robots rules.</details>

**18.** Debug — The page store grows much faster than expected; analysis shows many URLs with different session IDs storing identical content. What do you change?
<details><summary>Answer</summary>Normalise URLs (strip session and tracking parameters), honour `rel="canonical"`, and check content fingerprints before storing — exact hashes for identical pages, SimHash for near-identical ones — storing a reference instead of a copy.</details>

**19.** Debug — Fetchers spend 40% of their time waiting on DNS during peaks. What's the fix?
<details><summary>Answer</summary>Run local caching resolvers on (or near) crawler nodes, prefetch DNS for hosts about to become ready, respect TTLs, and batch or parallelise lookups — instead of resolving each fetch against a remote public resolver.</details>

**20.** Code — Write `normalizeUrl(href, base)` that resolves relative links, lower-cases the host, removes fragments and default ports, drops `utm_*` and `sessionid` parameters, and sorts the rest.
<details><summary>Answer</summary>

```js
function normalizeUrl(href, base) {
  const u = new URL(href, base);                           // resolves ./, ../, and relative paths
  u.hash = '';
  u.hostname = u.hostname.toLowerCase();
  if ((u.protocol === 'https:' && u.port === '443') || (u.protocol === 'http:' && u.port === '80')) u.port = '';
  const kept = [...u.searchParams].filter(([k]) => !/^utm_/i.test(k) && k.toLowerCase() !== 'sessionid')
    .sort(([a], [b]) => (a < b ? -1 : a > b ? 1 : 0));
  u.search = new URLSearchParams(kept).toString();
  return u.toString();
}
console.log(normalizeUrl('../news/?b=2&utm_source=x&a=1#top', 'https://Example.com:443/blog/post'));
// https://example.com/news/?a=1&b=2
```
</details>

**21.** Code — Write `isAllowed(robotsRules, path)` given parsed rules `[{ type: 'allow' | 'disallow', prefix }]`, using the longest matching prefix (allow wins ties).
<details><summary>Answer</summary>

```js
function isAllowed(rules, path) {
  let best = null;
  for (const r of rules) {
    if (!path.startsWith(r.prefix)) continue;
    if (!best || r.prefix.length > best.prefix.length || (r.prefix.length === best.prefix.length && r.type === 'allow')) best = r;
  }
  return !best || best.type === 'allow';                   // no matching rule: allowed
}
const rules = [{ type: 'disallow', prefix: '/private' }, { type: 'allow', prefix: '/private/press' }];
console.log(isAllowed(rules, '/private/hr'), isAllowed(rules, '/private/press/2026'), isAllowed(rules, '/blog')); // false true true
```
</details>

**22.** Code — Write `nextFetchDelay({ history, minHours, maxHours, startHours })` that, for each past fetch in `history` (`true` = content changed), halves the recrawl interval on a change and doubles it otherwise, within bounds.
<details><summary>Answer</summary>

```js
function nextFetchDelay({ history, minHours = 1, maxHours = 24 * 30, startHours = 24 }) {
  let hours = startHours;
  for (const changed of history) hours = changed ? hours / 2 : hours * 2;   // adapt to observed change rate
  return Math.min(maxHours, Math.max(minHours, hours));
}
console.log(nextFetchDelay({ history: [true, true, true] }), nextFetchDelay({ history: [false, false, false, false] })); // 3 384
```
A news front page converges to frequent recrawls; a static page backs off toward the maximum.</details>

**23.** Scenario — You're building a crawler for a price-comparison site that must refresh 50 million product pages daily from 20,000 retailers. How does the design differ from a general web crawler?
<details><summary>Answer</summary>The URL set is known and bounded, so freshness — not discovery — dominates: schedule each product page by price-change history and importance, use sitemaps and product feeds or APIs where retailers offer them, and use conditional requests. Politeness per retailer is the binding constraint: 50 M pages over 20,000 hosts is ~2,500 per host per day, which some retailers will only allow with agreements. Extraction is structured (prices, stock), so parsers per site template and change detection on extracted fields matter more than SimHash.</details>

**24.** Scenario — Legal asks that the crawler be able to stop crawling any domain within minutes of a request, and delete everything collected from it. How do you support this?
<details><summary>Answer</summary>Maintain a blocklist checked at URL admission and before every fetch (distributed to all nodes within minutes, with the owning node purging that host's queues). For deletion, the page store must be indexed by host so archive files can be rewritten without the host's records (or records marked deleted and compacted), with metadata, fingerprints, and downstream indexes purged too. Record the request and completion for audit.</details>

---

## Chapter Summary

- A crawler's loop is simple; its difficulty is scale, politeness, traps, and duplicates — and politeness, not worker count, bounds its throughput.
- Organise the frontier around hosts: per-host queues, a heap of hosts by next allowed fetch time, adaptive delays, and priorities within and across hosts.
- Assign hosts to nodes by consistent hashing so per-host decisions need no coordination, and run local DNS caches.
- Deduplicate URLs with normalisation, canonical links, and a Bloom filter; deduplicate content with exact hashes and SimHash for near-duplicates.
- Obey robots.txt, identify the crawler honestly, back off on slow or erroring servers, and bound URL spaces against traps; fetch defensively.
- Recrawl by observed change rate and importance with conditional requests; at 10× the frontier, seen-set, storage, and freshness budget dominate the
  engineering.

---

## Practice Exercises

1. Build a small crawler on top of §44.10's frontier that fetches pages from a local test site (with a calendar trap), extracts links, normalises them, and
   respects robots.txt.
2. Replace the frontier's `Set` with Chapter 10's Bloom filter backed by a persistent store, and measure memory and false-positive skips.
3. Read the robots.txt standard (RFC 9309) and implement its matching rules, including wildcards and end-of-path anchors.
4. Compute SimHash fingerprints for a few hundred real pages (or Wikipedia article revisions) and find the Hamming-distance threshold that separates near-duplicates
   from distinct pages.
5. Simulate recrawl scheduling for pages with different true change rates, and compare uniform recrawling with adaptive scheduling at the same fetch budget.

---

## Mini Project: A Polite Distributed Crawler

Build a crawler that runs as several worker processes, crawls a set of local test sites (including traps and duplicates), and demonstrates every property in this
chapter.

- Hosts assigned to workers by consistent hashing; URLs for other workers' hosts forwarded to them.
- A per-host frontier with ready-time heap, robots.txt caching and compliance, adaptive delays, and per-host page budgets.
- URL normalisation, a Bloom-filter seen-set, exact and SimHash content deduplication, and storage in batched archive files.
- Trap limits (depth, length, parameters) and fetch safety limits (size, time, content type).
- A report: pages per second, maximum observed request rate per host (must respect delays), duplicates skipped, traps contained, and coverage of the test sites.

---

**Next:** [Chapter 45 — Design a Distributed Job Scheduler →](45-job-scheduler.md)
