# Chapter 42 — Design Search Autocomplete

## Learning Objectives

- Scope typeahead suggestions and estimate request rates from keystrokes rather than searches
- Serve top-k completions within a tight latency budget using a trie with precomputed results per node
- Build suggestion data offline from query logs with recency decay, filtering, and thresholds
- Blend fresh trending queries into slowly rebuilt suggestions
- Reduce request volume and latency with debouncing, caching per prefix, and sharding

---

## 42.1 Requirements and Scope

Autocomplete — typeahead — suggests completions as a user types into a search box: "wea" → "weather", "weather tomorrow", "weather
radar". It looks like a small feature and is a demanding system: a request on almost every keystroke, a latency budget measured in
tens of milliseconds, and suggestions that must reflect what people are searching for *now*.

**Functional requirements**

- Return up to 10 suggestions for a prefix, ranked by popularity (with recency weighting).
- Suggestions reflect real query volume and incorporate trending queries within minutes.
- Unsafe, offensive, or private suggestions are filtered out.

**Out of scope**: the search results themselves (Chapter 18), spelling correction beyond simple prefix tolerance (mentioned in §42.7),
deep personalisation (mentioned in §42.8).

**Non-functional requirements**

| Requirement | Target |
|---|---|
| Latency | p99 < 50 ms server-side; suggestions must feel instantaneous while typing |
| Availability | High, but degradation is acceptable — search still works without suggestions |
| Freshness | Popular suggestions rebuilt daily; trending queries within ~10 minutes |
| Scale | 500 M DAU, 5 searches each per day |

The availability row matters: autocomplete is an **optional dependency** (Chapter 4). If it's slow, the client should simply stop showing
suggestions, never block typing or searching.

---

## 42.2 Estimation

```js
// Autocomplete: keystrokes, not searches, set the request rate — and debouncing cuts it sharply
const dau = 500e6, searchesPerUser = 5, charsPerQuery = 12;
const keystrokeReqs = (dau * searchesPerUser * charsPerQuery) / 86_400;
const debounced = keystrokeReqs * (5 / charsPerQuery);       // ~5 requests per query after a 100 ms debounce
console.log(`per keystroke: ~${Math.round(keystrokeReqs / 1000)}k req/s avg | debounced: ~${Math.round(debounced / 1000)}k/s avg, ~${Math.round((debounced * 3) / 1000)}k/s peak`);
const topQueries = 100e6, avgLen = 20, k = 10;
// Rough trie size: one node per distinct prefix (bounded by total characters), each holding k suggestion refs
const nodes = topQueries * avgLen * 0.5;                     // prefixes shared heavily near the root
console.log(`~${(nodes / 1e9).toFixed(1)} B nodes × ${k} refs × 4 B ≈ ${((nodes * k * 4) / 1e9).toFixed(0)} GB for top-k lists`);
// per keystroke: ~347k req/s avg | debounced: ~145k/s avg, ~434k/s peak
// ~1.0 B nodes × 10 refs × 4 B ≈ 40 GB for top-k lists
```

What the numbers decide:

- **Hundreds of thousands of requests per second**, each needing an answer in milliseconds, rules out querying a database or a search
  engine per keystroke. Suggestions must come from **memory**.
- **The data fits in memory**: the most popular hundred million queries, with precomputed top-k lists, are tens of gigabytes — one large
  machine's worth, replicated many times for throughput (or sharded by prefix).
- **Client behaviour is a lever**: debouncing (waiting ~100 ms after the last keystroke) cuts requests by more than half without users
  noticing, and caching short prefixes removes even more.

---

## 42.3 API and Data Model

```text
GET /v1/suggest?q=wea&lang=en&region=PT&limit=10
→ 200 { "prefix": "wea", "suggestions": ["weather", "weather tomorrow", "weather radar", …] }
   Cache-Control: public, max-age=300        ← popular prefixes are identical for everyone in a region
```

The response for a given (prefix, language, region) is the same for all users, which makes it **cacheable** at the CDN and in browsers —
the single biggest lever on load (§42.7). Personalised suggestions, if added, are merged client-side or in a separate request so the shared
part stays cacheable.

**Data model.** The serving structure is a **trie** (prefix tree) per language/region, where each node holds the top-k completions for its
prefix (§42.5). It's built offline from an aggregated table of `(query, region, language) → score`, where the score is a recency-decayed count
(§42.6).

---

## 42.4 High-Level Design

```text
OFFLINE (daily)                                         ONLINE
search logs ─▶ aggregate counts per query (decayed) ─▶ filter ─▶ build tries ─▶ snapshot files
                                                                                  │ distribute
trending (streaming, minutes) ─▶ hot-query table ──────────────────────┐          ▼
                                                                       └─▶ suggest servers (tries in memory)
user types ─▶ debounce ─▶ browser cache ─▶ CDN cache ─▶ suggest servers ─▶ merge trending ─▶ response
```

Two paths feed suggestions: a **batch pipeline** that rebuilds the full trie from weeks of logs (stable, thorough, slow) and a **streaming
pipeline** that detects queries surging in the last minutes (fast, small) — the lambda pattern from Chapter 25, applied to a read-optimised
structure. Serving nodes load trie snapshots at startup, swap in new ones atomically when published, and merge in the small trending set at
query time.

---

## 42.5 Deep Dive: A Trie With Precomputed Top-K

A trie stores strings by their characters: each node represents a prefix, and its children extend it by one character. Finding the node for a
prefix takes O(length of the prefix). The question is how to get the **top-k completions** of that prefix:

| Approach | Lookup cost | Memory | Build cost |
|---|---|---|---|
| Traverse the prefix's whole subtree, collect completions, sort | Proportional to the subtree — huge for short prefixes ("a" has millions of completions) | Smallest | Cheap |
| **Store top-k at every node** | O(prefix length) | k references per node (§42.2: tens of GB) | Computed once per build |
| Database `LIKE 'pre%' ORDER BY score LIMIT k` | Index range scan per keystroke | Disk | None |

Storing the top-k list at each node is the standard answer: the expensive work happens once, offline, and every lookup is a short walk plus
returning a precomputed list. The real-world example in §42.10 builds such a trie. Memory is controlled by:

- **Limiting the corpus** to queries above a minimum count (which is also a privacy measure, §42.6).
- **Storing references, not strings**, in top-k lists (an integer ID into a table of query strings).
- **Compressing the trie**: collapsing chains of single-child nodes into one node with a multi-character edge (a radix tree), and storing nodes
  in compact arrays rather than objects.

> 💼 **Interview** — "How do you return the top 10 suggestions for a prefix fast?" The expected answer is a trie with precomputed top-k per node,
> built offline from aggregated counts, held in memory, and replicated; then the latency budget (debounce, CDN caching of popular prefixes) and
> freshness (a streaming layer for trending queries).

---

## 42.6 Deep Dive: The Data Pipeline

Suggestions are only as good as the counts behind them. The batch pipeline (Chapter 25) runs over the search logs:

1. **Normalise** queries: lower-case, trim and collapse whitespace, Unicode normalisation — so "Weather " and "weather" count together.
2. **Aggregate** counts per (query, language, region) per day.
3. **Score with recency decay**: an exponentially decayed sum of daily counts (a half-life of a week, say), so last month's spike fades and
   this week's interest rises (§42.10 shows the formula).
4. **Filter**: remove queries below a minimum count — a strong privacy protection, since rare queries often contain personal information (names,
   addresses, phone numbers) — and apply blocklists and classifiers for offensive, dangerous, or legally problematic suggestions.
5. **Build** a trie per language and region, serialise it to a compact file, validate it (size, spot checks of known prefixes), and publish it
   with a version number.

**Freshness for trending queries.** A daily build can't suggest "earthquake lisbon" ten minutes after an earthquake. A streaming job counts queries
over short sliding windows (Chapter 25), detects ones whose recent rate far exceeds their baseline, filters them, and publishes a small **trending
table** every minute or so. Serving nodes merge trending entries into the precomputed top-k for matching prefixes at query time — cheap, because the
trending set is small.

> ⚠️ **Gotcha** — **Suggestions are a manipulation target.** If suggestions follow raw query volume, anyone who can generate queries — bots,
> coordinated groups — can push a suggestion onto everyone's search box, whether to promote something or to defame someone. Count unique users
> rather than raw queries, discount traffic from suspected automation, cap the influence of any single source, and review trending suggestions
> before they reach broad audiences.

---

## 42.7 Deep Dive: Latency and Load

The budget is tight — suggestions that arrive after the user typed two more characters are useless — and the request rate is high. The layers that
make it work:

- **Client debouncing and cancellation.** Send a request only after ~100 ms without a keystroke, and cancel in-flight requests when a newer one
  starts, so the server answers the prefix the user actually paused on.
- **Client-side caching and prefix reuse.** Once "wea" returned results, "weat" can often be filtered locally from them while the request is in flight.
- **CDN and browser caching.** Responses for (prefix, language, region) are shared across users; the short, popular prefixes — which receive most
  requests — are almost always cache hits at the edge (Chapter 6). A few minutes of TTL is fine, since suggestions change slowly.
- **In-memory serving, replicated.** Each serving node holds the whole trie for its language/region (tens of GB) and answers in microseconds; add
  replicas for throughput. When a single trie no longer fits, **shard by prefix** (a–f, g–m…), with care for skew: far more queries start with "s"
  than "x", so shard boundaries follow measured load, and the hottest short prefixes are cached and replicated everywhere.
- **Tolerance for typos.** For prefixes with no or few completions, a fallback can try small edits (Chapter 18's edit distance) against a fuzzy
  index — bounded to short prefixes and a distance of 1 to keep it cheap.

---

## 42.8 Failure Modes, Bottlenecks, and 10×

| Failure | Effect | Mitigation |
|---|---|---|
| Suggest servers slow or down | No suggestions | Client gives up after a short timeout; search still works (optional dependency) |
| Bad trie published (empty, corrupted, or containing unsafe content) | Wrong suggestions everywhere | Validation before publish, staged rollout of new versions (Chapter 31), instant rollback to the previous version |
| Batch pipeline fails | Suggestions go stale | Keep serving the last good version; alert on age of the published trie |
| Trending pipeline manipulated | A harmful suggestion spreads | Unique-user counting, automation filtering, caps per source, review before broad exposure |
| Hot prefix after an event | Spike on a few keys | CDN caching; trending merge is cheap; replicas absorb the rest |

**Personalisation** (recent searches, location, language) adds value but breaks shared caching. The usual compromise: the shared, cacheable
suggestions come from the global trie, and a small personal list (the user's own recent queries matching the prefix) is merged on the client or
fetched separately with a short private cache.

**At 10×** — billions of daily users across many languages: the trie per language/region multiplies, sharding by prefix becomes necessary for large
languages, and the trending pipeline becomes a real stream-processing system with per-region windows. The serving cost stays modest relative to the
CDN savings, because the edge absorbs most traffic.

---

## 42.9 Common Mistakes and Best Practices

**Mistake: a database query per keystroke.** Hundreds of thousands of `LIKE` queries per second can't meet a 50 ms budget.

**Mistake: computing top-k by traversing subtrees at query time.** Short prefixes have millions of completions; precompute top-k per node.

**Mistake: no client debouncing.** Every keystroke becomes a request, most of them answered after the user has typed more.

**Mistake: uncacheable responses.** Mixing personal data into the shared response prevents CDN caching of popular prefixes.

**Mistake: suggesting rare queries.** They often contain personal information; enforce minimum-count thresholds.

**Mistake: raw query volume as the score.** Bots and coordinated campaigns can push suggestions onto everyone's screen.

**Mistake: blocking the search box on suggestions.** Autocomplete must degrade silently when it's slow.

**Best practice: offline build, online serve.** Heavy aggregation and ranking happen in a batch pipeline; serving is a read-only, in-memory lookup.

**Best practice: a small streaming layer for trends.** Merge minute-fresh trending queries into the daily trie at query time.

**Best practice: versioned trie snapshots with validation and rollback.** A bad build is a deploy like any other (Chapter 31).

---

## 42.10 Real-World Example: A Top-K Autocomplete Trie

This builds the serving structure from aggregated, recency-decayed scores: a trie in which every node stores its own top-k completions, filled by
inserting queries in descending score order so no node ever needs re-sorting. A lookup walks the prefix's characters and returns the node's
precomputed list. The demo includes a breaking story (searched heavily today) and an event from two weeks ago, to show decay at work.

```js
// autocomplete-trie.mjs — a prefix trie where every node stores its own top-k completions, so a
// lookup is a walk of the prefix's characters: O(prefix length), independent of how many queries
// share the prefix. Built offline from aggregated query counts; served read-only.

export function buildTrie(counts, { k = 5 } = {}) {
  const root = { children: new Map(), top: [] };
  // Insert in descending score order: each node's top-k fills with the best queries first
  const ranked = [...counts].sort((a, b) => b[1] - a[1] || (a[0] < b[0] ? -1 : 1));   // ties: alphabetical
  for (const [query, score] of ranked) {
    let node = root;
    for (const ch of query) {
      if (!node.children.has(ch)) node.children.set(ch, { children: new Map(), top: [] });
      node = node.children.get(ch);
      if (node.top.length < k) node.top.push([query, score]);   // already sorted: no re-sorting needed
    }
  }
  return root;
}

export function suggest(trie, prefix, { limit = 5 } = {}) {
  let node = trie;
  for (const ch of prefix.toLowerCase()) {
    node = node.children.get(ch);
    if (!node) return [];                        // nobody has searched anything starting with this
  }
  return node.top.slice(0, limit).map(([q]) => q);
}

// Recency-weighted scores: older counts decay so last week's news fades from suggestions
export const decayedScore = (dailyCounts, halfLifeDays = 7) =>
  dailyCounts.reduce((s, c, ageDays) => s + c * 0.5 ** (ageDays / halfLifeDays), 0);

// ── Demo ────────────────────────────────────────────────────────────────────────────
const counts = new Map([
  ['weather', 9_000], ['weather tomorrow', 4_000], ['web mail', 2_500], ['wedding dresses', 1_800],
  ['weekend deals', 1_200], ['webcam test', 900], ['weather radar', 3_100], ['wells fargo', 2_000],
]);
// A breaking story: few total searches, but all of them today
counts.set('weather warning lisbon', decayedScore([5_000]));                  // 5,000 searches today
counts.set('wembley tickets', decayedScore([0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 12_000])); // 12k, two weeks ago
const trie = buildTrie(counts);
console.log(suggest(trie, 'we').join(' | '));
// weather | weather warning lisbon | weather tomorrow | weather radar | wembley tickets
console.log(suggest(trie, 'weath').join(' | '));
// weather | weather warning lisbon | weather tomorrow | weather radar
console.log(suggest(trie, 'wem'), suggest(trie, 'xyz'));
// [ 'wembley tickets' ] []
```

The 12,000 searches for Wembley tickets two weeks ago are worth 3,000 today after two half-lives, enough to still appear for "we" but behind the
5,000 searches for today's weather warning. Unknown prefixes return immediately with nothing.

The decision to defend is **precomputing the top-k list at every node** — storing tens of gigabytes of redundant lists — instead of computing each
answer on demand by walking the prefix's subtree. On-demand computation uses a fraction of the memory and always reflects the latest scores. But the
cost of on-demand lookup is proportional to the size of the subtree, and the prefixes users type most — one or two characters — have the largest
subtrees of all: millions of completions to scan and rank for a request that must answer in milliseconds, hundreds of thousands of times per second.
Precomputation turns every lookup into a walk of a few nodes regardless of prefix, shifts all the ranking cost into an offline build that runs once a
day, and makes latency independent of how popular a prefix is. Memory is the cheaper resource here, and it's bounded by the corpus filters that also
protect privacy; the freshness it gives up is recovered by the small trending layer merged at query time.

---

## 42.11 Questions

**1.** MCQ — What sets the request rate for autocomplete? a) Searches per user b) Keystrokes (reduced by debouncing) c) Page views d) Result clicks
<details><summary>Answer</summary>**b)** A request on (almost) every keystroke, which debouncing cuts substantially.</details>

**2.** MCQ — Why store top-k completions at every trie node? a) To save memory b) So a lookup is O(prefix length), independent of how many completions the prefix has c) To support deletions d) To compress strings
<details><summary>Answer</summary>**b)** Short prefixes would otherwise require scanning millions of completions.</details>

**3.** MCQ — Why can autocomplete responses be cached at a CDN? a) They're small b) For a given prefix, language, and region they're the same for everyone c) They never change d) CDNs require it
<details><summary>Answer</summary>**b)** Shared responses for popular short prefixes are almost always edge hits.</details>

**4.** MCQ — What's the main purpose of a minimum-count threshold for suggestions? a) Speed b) Privacy and quality — rare queries often contain personal data c) Storage only d) Ranking ties
<details><summary>Answer</summary>**b)** It also keeps the trie smaller.</details>

**5.** MCQ — How do trending queries appear within minutes when the trie is rebuilt daily? a) Rebuild the trie every minute b) A streaming pipeline publishes a small trending table merged at query time c) Clients send trends d) They don't
<details><summary>Answer</summary>**b)** Batch for completeness, stream for freshness.</details>

**6.** MCQ — If the suggest service is slow, what should the search box do? a) Wait b) Show an error c) Stop showing suggestions and keep working normally d) Retry repeatedly
<details><summary>Answer</summary>**c)** Autocomplete is an optional dependency.</details>

**7.** Short — Why does debouncing reduce load without hurting the experience?
<details><summary>Answer</summary>Users type faster than they read suggestions; waiting ~100 ms after the last keystroke skips requests for intermediate prefixes nobody would look at, and cancelling stale requests avoids wasted work.</details>

**8.** Short — How does recency decay work, and why use it?
<details><summary>Answer</summary>Each day's count is weighted by 0.5^(age / half-life), so old spikes fade and recent interest counts more. Without it, last year's events would dominate suggestions forever.</details>

**9.** Short — How can suggestions be manipulated, and how do you defend against it?
<details><summary>Answer</summary>By generating many queries to push a suggestion. Count unique users rather than raw queries, discount automated traffic, cap any single source's influence, and review trending suggestions before broad exposure.</details>

**10.** Short — How do you shard a trie that no longer fits on one machine?
<details><summary>Answer</summary>By prefix ranges, with boundaries chosen from measured load rather than the alphabet (prefixes are very unevenly popular), and with the hottest short prefixes cached and replicated everywhere.</details>

**11.** Short — How do you add personalisation without losing CDN caching?
<details><summary>Answer</summary>Keep the shared suggestions in a cacheable response, and merge a small personal list (the user's own matching recent queries) on the client or via a separate, privately cached request.</details>

**12.** Output — Using §42.10's `decayedScore` with a 7-day half-life, what is the score of `[1000, 0, 0, 0, 0, 0, 0, 1000]` (1,000 today and 1,000 a week ago)?
<details><summary>Answer</summary>**1,500.** 1,000 × 1 + 1,000 × 0.5.</details>

**13.** Output — Using §42.10's trie, what does `suggest(trie, 'web')` return?
<details><summary>Answer</summary>**`[ 'web mail', 'webcam test' ]`.** Those are the only completions starting with "web", in score order.</details>

**14.** Output — Using §42.2's model with 200 M DAU, 4 searches per user, 10 characters per query, and debouncing reducing requests to 4 per query, what's the average request rate?
<details><summary>Answer</summary>**~37,000/s.** 200 M × 4 × 4 / 86,400 ≈ 37,037.</details>

**15.** Output — A CDN serves 92% of suggestion requests from cache. With 434,000 requests/s at peak, how many reach the suggest servers?
<details><summary>Answer</summary>**~34,700/s.** 8% of 434,000.</details>

**16.** Debug — After a new trie build, users saw no suggestions for any prefix for 20 minutes. The build had produced an almost empty file due to a bad input partition. What safeguards were missing?
<details><summary>Answer</summary>Validation before publishing (size bounds, spot checks of known prefixes, comparison with the previous version), a staged rollout of new trie versions, and instant rollback — plus alerting on the published trie's size and age.</details>

**17.** Debug — Suggestion latency p99 is 400 ms for one- and two-character prefixes but 5 ms for longer ones. The service collects completions by traversing the prefix's subtree. Cause and fix?
<details><summary>Answer</summary>Short prefixes have enormous subtrees, so on-demand traversal is slow exactly for the most frequent requests. Precompute top-k at every node during the offline build (and cache short-prefix responses at the edge).</details>

**18.** Debug — A suggestion containing a private individual's name and phone number appeared for a common prefix. How did it get there and how do you prevent it?
<details><summary>Answer</summary>A rare query — possibly repeated by one user or a small group — made it into the trie. Enforce minimum unique-user thresholds, add classifiers for personal data patterns (phone numbers, addresses, emails), and provide a fast removal path that propagates to serving nodes and caches.</details>

**19.** Debug — The suggest service's load is 3× what was planned. Client logs show a request per keystroke, and responses include the user's recent searches, with `Cache-Control: private`. What do you change?
<details><summary>Answer</summary>Debounce and cancel requests on the client, and split the response: shared suggestions in a public, CDN-cacheable response; the personal recent-searches list fetched separately or kept on the device.</details>

**20.** Code — Write `normalizeQuery(q)` that lower-cases, applies Unicode NFKC normalisation, trims, and collapses internal whitespace.
<details><summary>Answer</summary>

```js
const normalizeQuery = (q) => q.normalize('NFKC').toLowerCase().trim().replace(/\s+/g, ' ');
console.log(JSON.stringify(normalizeQuery('  Weather   TOMORROW ')), normalizeQuery('ｗｅａｔｈｅｒ')); // "weather tomorrow" weather
```
NFKC folds full-width and compatibility characters, so visually identical queries count together.</details>

**21.** Code — Write `mergeTrending(base, trending, prefix, limit)` that merges a node's precomputed `[query, score]` list with trending entries matching the prefix, deduplicating and re-ranking.
<details><summary>Answer</summary>

```js
function mergeTrending(base, trending, prefix, limit = 10) {
  const scores = new Map(base);
  for (const [q, s] of trending) if (q.startsWith(prefix)) scores.set(q, Math.max(scores.get(q) ?? 0, s));
  return [...scores].sort((a, b) => b[1] - a[1]).slice(0, limit).map(([q]) => q);
}
console.log(mergeTrending([['weather', 9000], ['weather radar', 3100]], [['weather warning', 7000], ['wembley', 9999]], 'weath', 3));
// [ 'weather', 'weather warning', 'weather radar' ]
```
</details>

**22.** Code — Write a `debounce(fn, ms)` for the client that also aborts the previous in-flight request when a new one starts.
<details><summary>Answer</summary>

```js
function debounceLatest(fetchSuggestions, ms = 100) {
  let timer = null, controller = null;
  return (prefix) => new Promise((resolve) => {
    clearTimeout(timer);
    timer = setTimeout(async () => {
      controller?.abort();                         // the user kept typing: the old answer is useless
      controller = new AbortController();
      try { resolve(await fetchSuggestions(prefix, controller.signal)); } catch { resolve(null); }
    }, ms);
  });
}
```
Earlier calls' promises never resolve once superseded; UI code renders only the latest result.</details>

**23.** Scenario — An e-commerce site wants autocomplete that suggests both queries and products ("iph" → "iphone 16 case" and the product "iPhone 16 Pro"). How does the design change?
<details><summary>Answer</summary>Keep a query trie as above, and add a product-name index (a trie or an edge-n-gram search index, Chapter 18) scored by sales and availability. Serve both in parallel within the budget and blend them in fixed sections (queries, then products) or by a learned ranking. Product suggestions must respect stock and region (filter at query time or rebuild frequently), and because the catalogue changes, the product index is rebuilt more often or updated via CDC.</details>

**24.** Scenario — Legal requires that certain names never appear as suggestions in one country, effective within an hour of a court order. How do you support it?
<details><summary>Answer</summary>Maintain a per-region suppression list applied at serving time (filtering top-k results and trending merges), distributed to all serving nodes within minutes, with CDN cache purges for affected prefixes. The next offline build also excludes them so the list doesn't grow forever. Audit-log each suppression, and test that suggestions for prefixes of suppressed names never include them.</details>

---

## Chapter Summary

- Autocomplete's load comes from keystrokes, and its budget is tens of milliseconds, so suggestions must come from memory — never from a database per
  keystroke.
- A trie with precomputed top-k completions at every node makes each lookup a short walk, independent of how popular the prefix is; memory is bounded by
  corpus filtering and compact encoding.
- An offline pipeline normalises, aggregates, recency-decays, and filters query logs into versioned trie snapshots; a small streaming layer adds trending
  queries within minutes.
- Debouncing, client-side prefix reuse, and CDN caching of shared prefix responses remove most requests before they reach the servers.
- Minimum thresholds protect privacy; unique-user counting and review defend against manipulation; validation, staged rollout, and rollback protect against
  bad builds.
- Treat autocomplete as an optional dependency that degrades silently, and keep personalisation separate from the shared, cacheable response.

---

## Practice Exercises

1. Build the §42.10 trie from a public query log or a word-frequency list, and measure its memory usage before and after converting to a radix tree.
2. Add an HTTP endpoint with `Cache-Control` headers and put a caching proxy in front of it; measure hit ratio for a Zipf-distributed prefix workload.
3. Read a paper or engineering post on query autocompletion (for example, on ranking with recency and personalisation) and summarise how it scores
   candidates.
4. Implement a streaming trending detector over a simulated query stream: sliding-window counts, a baseline per query, and a ratio threshold.
5. Implement debounced, cancellable fetching in a small browser page and measure requests per typed query with and without it.

---

## Mini Project: An Autocomplete Service

Build an end-to-end autocomplete service with an offline build, online serving, and a freshness layer.

- An offline job that normalises a query log, computes decayed scores per region, filters by minimum unique users and a blocklist, and writes a versioned
  trie snapshot with validation.
- Serving nodes that load snapshots, swap versions atomically, and answer `/suggest` in under a millisecond server-side.
- A streaming trending layer merged at query time, with a review flag for new trending entries.
- Edge caching (a caching proxy) with appropriate headers, and a client with debouncing and cancellation.
- Load test results: p99 latency, server request rate with and without caching and debouncing, and the effect of a bad snapshot with automated rollback.

---

**Next:** [Chapter 43 — Design a Payment System →](43-payment-system.md)
