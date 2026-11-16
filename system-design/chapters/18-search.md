# Chapter 18 — Search

## Learning Objectives

- Explain how an inverted index answers text queries, and what analysis does to documents and queries
- Rank results with BM25 and combine text relevance with business signals and vector similarity
- Describe how a search cluster shards, replicates, refreshes, and merges, and what that costs
- Keep a search index in sync with its source of truth using change streams and external versions
- Change mappings with a zero-downtime reindex behind an alias

---

## 18.1 What Search Needs That Databases Don't

A relational database answers `WHERE title = 'boots'` and `WHERE title LIKE 'boot%'` with indexes.
It doesn't answer "waterproof hiking boots" the way users expect: matching documents that contain
*most* of the words, in any order, in any form (`boot`, `Boots`, `booting`?), ranked so the best
match comes first, tolerant of typos, with counts by brand and price range alongside. That's a
different problem — **information retrieval** — with its own index structure and its own notion of a
correct answer: not "the rows that match" but "the documents most likely to be what the user meant".

| Need | Relational index | Search engine |
|---|---|---|
| Exact match, ranges, joins, transactions | Native | Possible, not its strength |
| Matching any words, any order, word variants | Poor (`LIKE '%x%'` scans) | Native — inverted index + analysis |
| Ranking by relevance | No | Native — BM25, boosts, custom scoring |
| Typo tolerance, synonyms | Limited (trigrams) | Native |
| Facet counts across a result set | Expensive `GROUP BY`s | Native aggregations |
| Source of truth | Yes | **No** — a derived copy (Chapter 11) |

Before adding a search cluster, check whether the database is enough. PostgreSQL's full-text search
(`tsvector`, `GIN` indexes, `ts_rank`) and trigram indexes (`pg_trgm`) handle moderate volumes and
simple relevance well, with no synchronisation problem at all. A dedicated engine — Elasticsearch,
OpenSearch, Solr, or managed and embedded alternatives — earns its operational cost when you need
serious relevance tuning, faceting over large result sets, multi-language analysis, very high query
rates, or search across data that lives in several systems.

---

## 18.2 The Inverted Index

A search engine stores, for every distinct term, the list of documents that contain it — a
**postings list**. Answering a query means looking up each query term's postings and combining them.
That reversal of the document → words relationship is why it's called an **inverted index**:

```text
documents                              inverted index (after analysis)
p1 "Waterproof hiking boots for men"   waterproof → p1, p4, p5
p4 "Waterproof jacket with hood"       hik        → p1, p3
p5 "Trail running shoes, waterproof"   boot       → p1
                                       runn       → p2, p5   …
```

Before anything is indexed, text goes through **analysis**: splitting into tokens, lower-casing,
removing stop words ("the", "for"), and **stemming** (reducing words to a common root so "boots"
matches "boot"). The same analysis must be applied to queries — if documents are stemmed and queries
aren't, "shoe" never matches "shoes". Real engines also store **positions** in each posting so phrase
queries ("hiking boots", adjacent and in order) can be answered.

Postings lists are sorted by document ID and compressed (delta-encoded IDs are small numbers), so
intersecting two lists for an `AND` query is a fast merge, and the whole index for millions of
documents often fits in memory. The index is built from immutable **segments**, much like an LSM tree
(Chapter 10): new documents go into small new segments, which are periodically merged into larger
ones, and deletes are marks that disappear on merge.

---

## 18.3 Relevance: BM25 and Beyond

Once candidate documents are found, they must be ordered. The standard text-relevance function is
**BM25**, which scores each document by summing, over the query's terms:

- **How rare the term is** across the collection (inverse document frequency) — "waterproof" in a
  product catalogue is common; "merino" is rare and therefore more informative.
- **How often the term appears in the document**, with diminishing returns — the fifth mention of
  "boot" adds less than the first (controlled by `k1`).
- **How long the document is**, relative to the average — a match in a short title means more than
  the same match buried in a long description (controlled by `b`).

```js
// inverted-index.mjs — tokenise, build postings lists, and rank with BM25.
const STOP = new Set(['the', 'a', 'an', 'and', 'or', 'of', 'for', 'to', 'in', 'with']);
// Crude stemmer: real engines use language-specific analyzers (Snowball, etc.)
const stem = (t) => t.replace(/([^s])s$/, '$1').replace(/ing$/, '');
const analyze = (text) => text.toLowerCase().split(/[^a-z0-9]+/).filter((t) => t && !STOP.has(t)).map(stem);

class InvertedIndex {
  #postings = new Map(); // term → Map(docId → term frequency)
  #lengths = new Map();  // docId → number of terms
  add(id, text) {
    const terms = analyze(text);
    this.#lengths.set(id, terms.length);
    for (const t of terms) {
      if (!this.#postings.has(t)) this.#postings.set(t, new Map());
      const p = this.#postings.get(t);
      p.set(id, (p.get(id) ?? 0) + 1);
    }
  }
  // BM25: rare terms count more (IDF); repeated terms help with diminishing returns (k1);
  // long documents are penalised so they don't win just by containing more words (b)
  search(query, { k1 = 1.2, b = 0.75, limit = 3 } = {}) {
    const N = this.#lengths.size;
    const avgLen = [...this.#lengths.values()].reduce((a, x) => a + x, 0) / N;
    const scores = new Map();
    for (const term of new Set(analyze(query))) {
      const postings = this.#postings.get(term);
      if (!postings) continue;
      const idf = Math.log(1 + (N - postings.size + 0.5) / (postings.size + 0.5));
      for (const [id, tf] of postings) {
        const norm = tf * (k1 + 1) / (tf + k1 * (1 - b + b * this.#lengths.get(id) / avgLen));
        scores.set(id, (scores.get(id) ?? 0) + idf * norm);
      }
    }
    return [...scores].sort((x, y) => y[1] - x[1]).slice(0, limit).map(([id, s]) => `${id}:${s.toFixed(2)}`);
  }
}

const idx = new InvertedIndex();
idx.add('p1', 'Waterproof hiking boots for men');
idx.add('p2', 'Lightweight running shoes');
idx.add('p3', 'Hiking socks, merino wool, pack of 3');
idx.add('p4', 'Waterproof jacket with hood');
idx.add('p5', 'Trail running shoes, waterproof');
console.log(analyze('Running Shoes for Hiking'));  // [ 'runn', 'shoe', 'hik' ]
console.log(idx.search('waterproof hiking boots')); // [ 'p1:2.80', 'p3:0.73', 'p4:0.60' ]
console.log(idx.search('running shoe'));            // [ 'p2:1.95', 'p5:1.75' ]
```

p1 matches all three terms and wins by a wide margin; for "running shoe", p2 and p5 match equally,
and p2 ranks higher because it's shorter.

Text relevance is rarely the whole ranking. Production search combines it with:

- **Field boosts** — a match in the title counts more than one in the description.
- **Business signals** — popularity, conversion rate, stock, recency, margin, applied as score
  modifiers ("function scores").
- **Learning to rank** — a model trained on click and purchase data re-ranks the top few hundred BM25
  candidates using many features. Expensive, so it's applied only to a candidate set.
- **Semantic (vector) search** — embeddings find documents that are related in meaning without sharing
  words ("rain boots" for "wellies"). Chapter 11 covered the indexes. **Hybrid search** runs both
  keyword and vector retrieval and fuses the two ranked lists, commonly with *reciprocal rank fusion*:
  each document scores Σ 1 / (k + rank) across the lists, which needs no score calibration between the
  two systems.

Relevance is never "done": measure it with offline judgement sets (queries with known good results,
scored with metrics like NDCG) and online with click-through and zero-result rates, and treat ranking
changes like any other change — tested and rolled out gradually.

> 💼 **Interview** — "How would you design product search?" Interviewers want the pipeline: analysis
> and an inverted index for recall, BM25 plus boosts and business signals for ranking, filters and
> facets for navigation, a CDC-fed index kept separate from the source of truth, and metrics for
> relevance. Mentioning zero-result rate as a key metric is a strong touch.

---

## 18.4 Search Cluster Architecture

A search engine distributes its index the way Chapter 14 described, with some specifics worth knowing.

**Shards.** An index is split into a fixed number of primary shards, each a self-contained index over
a subset of documents (document-partitioned — Chapter 14's local indexes). A query is scattered to one
copy of every shard, each shard returns its top results, and a coordinating node merges them. So query
latency is bound by the slowest shard (Chapter 3), and more shards mean more fan-out per query.

**Replicas.** Each shard has replicas on other nodes, serving reads and taking over on failure. Adding
replicas scales query throughput; it doesn't help indexing throughput, since every replica indexes
every document.

**Refresh and near-real-time search.** New documents are buffered in memory and become searchable
only when a **refresh** writes them into a new segment — typically every second. Search is therefore
*near*-real-time: a document indexed now is findable about a second later. Refresh more often and
indexing throughput drops (many tiny segments); refresh less often during bulk loads to speed them up.

**Merges.** Background merging combines small segments into large ones and physically removes deleted
documents. Merges are I/O-heavy and are the usual cause of indexing-related latency spikes.

**Sizing.** Shard count is fixed when an index is created in many engines, so choose it from expected
data size (shards in the tens of gigabytes are a common guideline) and query fan-out. Thousands of tiny
shards waste memory and coordination; a few huge ones make recovery and rebalancing slow.

> ⚠️ **Gotcha** — **Deep pagination is expensive.** Asking for page 500 with `from: 10000, size: 20`
> makes *every shard* return its top 10,020 results to the coordinator, which sorts them all to
> keep 20. Engines cap this (Elasticsearch's default limit is 10,000) for good reason. Use cursor-style
> pagination (`search_after` with the last result's sort values — Chapter 8's keyset idea) for deep
> scrolling, and a point-in-time snapshot when results must stay stable across pages.

---

## 18.5 Keeping the Index in Sync

The index is a derived copy (Chapter 11), and keeping it correct is most of the operational work of
search. The reliable architecture:

```text
source DB ──(CDC or outbox)──▶ change stream ──▶ indexer ──▶ search cluster
                                                  │
                                                  └── fetch related data, build the search document
```

**Never dual-write** from the request path to the database and the index — Chapter 11's failure
modes all apply. Feed the indexer from committed changes instead.

**Build search documents, not table rows.** A product search document typically joins the product,
its brand name, its category path, its current price and stock, and its review average —
denormalised at index time, because the search engine can't join at query time. That means a change
to a *brand name* must re-index every product of that brand. Map out which source changes affect
which search documents, and have the indexer handle each.

**Make indexing idempotent and order-safe.** Change streams deliver at least once and, across
partitions, not necessarily in order (Chapter 24). Use **external versioning**: write each document
with a version taken from the source — the change's log position or the row's version column — and
have the engine reject writes whose version is not newer than what it has. Replays and out-of-order
deliveries then become harmless, exactly as Chapter 11's `applyChange` did.

**Handle deletes as versioned events too.** A delete that arrives before a stale update would
otherwise be "undone" when the update arrives; keep a tombstone (or rely on the engine's versioned
delete) so the later-arriving older update loses.

**Monitor lag.** Indexing lag — time from commit to searchable — is an SLI. Alert on it, and on the
rate of indexing errors (mapping rejections, oversized documents), which otherwise silently leave
documents missing from search.

---

## 18.6 Reindexing Without Downtime

Many index changes can't be made in place: changing a field's type or analyzer, adding a new
tokenisation, or changing the shard count all require building a **new index** from the source. Doing
that while the product keeps serving searches and accepting writes has a standard recipe:

1. Readers and the live indexer address the index through an **alias** (`products` → `products_v1`),
   never by its physical name.
2. Record the source's change-log position, create `products_v2` with the new mapping, and bulk-copy a
   snapshot as of that position — throttled, with refresh disabled for speed.
3. **Replay all changes since the snapshot position** into the new index, until it has caught up.
4. Verify: document counts, and sampled spot checks of documents and queries against the old index.
5. **Atomically swap the alias** to `products_v2`. Readers move instantly.
6. Replay once more to cover changes made around the swap; keep `products_v1` for a while for rollback.

The real-world example in §18.9 implements this, with concurrent writes landing during the copy.

The alias is what makes rollback trivial: if relevance regresses after the swap, point the alias back.
It also makes reindexing a routine operation rather than a migration — which matters, because a search
index that can't be rebuilt from scratch on demand has quietly become a second source of truth.

---

## 18.7 Queries, Filters, and Performance

**Filters versus scored queries.** A clause that only restricts the result set ("in stock", "brand =
Acme", "tenant = 42") should be a **filter**: it isn't scored, and its results can be cached as bitsets
and reused across queries. Putting restrictions in the scored part wastes CPU and changes ranking.

**Facets (aggregations)** — counts by brand, price buckets, ratings — are computed over the matching
set on every shard and merged. High-cardinality facets (counts per seller across millions of sellers)
are expensive; approximate algorithms and limits on bucket counts keep them bounded.

**Typo tolerance** uses fuzzy matching by edit distance: "boots" matches "bots" and "boost" at distance
1. It multiplies the terms examined, so engines limit it (distance ≤ 2, a required unchanged prefix),
and it should apply only after exact matches fail or with lower weight.

**Autocomplete** is a different index shape — prefix-oriented structures such as edge n-grams or
completion tries, tuned for single-digit-millisecond latency. Chapter 42 designs it.

**Multi-tenant search** puts many tenants in one index with a mandatory tenant filter (applied by the
search layer, never trusted from the client — the same leak risk as Chapter 12), and uses custom
routing so each tenant's documents live on one shard: tenant queries then hit one shard instead of
all. Very large tenants get their own indexes.

**Protect the cluster.** Search clusters are easily overloaded by expensive queries: leading wildcards
(`*boots`), huge facet requests, deep pagination, and regex queries. Enforce query timeouts, cap result
windows and bucket counts, reject or rewrite pathological query types, and rate-limit per client.

---

## 18.8 Common Mistakes and Best Practices

**Mistake: treating the search index as the source of truth.** Indexes get corrupted, need remapping,
and lag; if one can't be rebuilt from the source, data has been lost waiting to happen.

**Mistake: dual-writing to the database and the index.** Crashes and reordering leave them permanently
inconsistent; feed the indexer from CDC or an outbox.

**Mistake: indexing without external versions.** Redelivered or reordered changes overwrite newer
documents with older ones, and deletes get resurrected.

**Mistake: different analysis at index and query time.** Stemmed documents and unstemmed queries (or
vice versa) silently fail to match.

**Mistake: deep pagination with `from`/`size`.** Every shard returns `from + size` results to be merged;
use `search_after` with a point-in-time snapshot.

**Mistake: restrictions as scored queries.** Uncached, slower, and they distort relevance; use filters.

**Mistake: addressing physical index names.** Every mapping change becomes an outage; read and write
through aliases.

**Best practice: start with the database's full-text search.** Add a search engine when relevance,
scale, or faceting needs justify a second system.

**Best practice: measure relevance.** Judgement sets offline, click-through and zero-result rate
online, and gradual rollout for ranking changes.

**Best practice: rebuildable by design.** A tested, throttled, alias-swapped reindex from the source is
a routine operation, not an emergency.

---

## 18.9 Real-World Example: A Zero-Downtime Reindexer

This implements §18.6: snapshot the source at a recorded change-log position, bulk-copy into a new
index, replay changes since that position, verify, swap the alias, and replay once more. Every write
carries the source's log position as an external version, so replays and overlaps can't regress a
document. The in-memory stand-ins keep writing — renames and deletes — while the copy is in progress.

```js
// reindex.mjs — rebuild a search index with a new mapping while writes continue, then swap
// an alias so readers move atomically. Changes are replayed from the source's change log,
// and every write carries the log position as an external version, so replays are harmless.

export async function reindex({ source, search, alias, newIndex, mapping, toDoc, batch = 500, onProgress = () => {} }) {
  const snapshotPos = await source.position();          // 1. remember where the snapshot starts
  await search.createIndex(newIndex, mapping);           // 2. new index, new mapping

  let copied = 0;                                        // 3. bulk-copy the snapshot
  for await (const rows of source.snapshot({ asOf: snapshotPos, batch })) {
    await search.bulk(newIndex, rows.map((r) => ({ id: r.id, doc: toDoc(r), version: r.version })));
    onProgress({ copied: (copied += rows.length) });
  }

  // 4. Replay everything that changed since the snapshot (including during the copy)
  const catchUp = async (from) => {
    let pos = from;
    for (const c of await source.changesSince(pos)) {
      await search.bulk(newIndex, [c.op === 'delete'
        ? { id: c.id, delete: true, version: c.pos }
        : { id: c.id, doc: toDoc(c.row), version: c.pos }]);
      pos = c.pos;
    }
    return pos;
  };
  let pos = await catchUp(snapshotPos);

  // 5. Verify before exposing it to readers
  const [expected, actual] = await Promise.all([source.count(), search.count(newIndex)]);
  if (expected !== actual) throw new Error(`count mismatch: source ${expected}, index ${actual}`);

  const previous = await search.swapAlias(alias, newIndex); // 6. atomic switch for readers
  pos = await catchUp(pos);                                // 7. close the window around the swap
  return { previous, position: pos };                      // keep `previous` for fast rollback
}

// ── In-memory stand-ins: a source with a change log, and a versioned search engine ──
function createSource() {
  const log = []; // { pos, id, op, row }
  const state = (upTo) => {
    const m = new Map();
    for (const c of log) if (c.pos <= upTo) c.op === 'delete' ? m.delete(c.id) : m.set(c.id, { ...c.row, version: c.pos });
    return m;
  };
  return {
    write: (id, row) => log.push({ pos: log.length + 1, id, op: 'upsert', row: { id, ...row } }),
    remove: (id) => log.push({ pos: log.length + 1, id, op: 'delete' }),
    async position() { return log.length; },
    async *snapshot({ asOf, batch }) {
      const rows = [...state(asOf).values()];
      for (let i = 0; i < rows.length; i += batch) { yield rows.slice(i, i + batch); this.duringCopy?.(); }
    },
    async changesSince(pos) { return log.filter((c) => c.pos > pos); },
    async count() { return state(log.length).size; },
    current: () => state(log.length),
  };
}
function createSearch() {
  const indexes = new Map(), aliases = new Map();
  return {
    async createIndex(name) { indexes.set(name, new Map()); },
    async bulk(name, ops) {
      const idx = indexes.get(aliases.get(name) ?? name);
      for (const op of ops) {
        const cur = idx.get(op.id);
        if (cur && cur.version >= op.version) continue;      // external versioning: stale writes lose
        idx.set(op.id, op.delete ? { deleted: true, version: op.version } : { doc: op.doc, version: op.version });
      }
    },
    async count(name) { return [...indexes.get(aliases.get(name) ?? name).values()].filter((d) => !d.deleted).length; },
    async swapAlias(alias, target) { const prev = aliases.get(alias); aliases.set(alias, target); return prev; },
    get: (name, id) => indexes.get(aliases.get(name) ?? name).get(id),
  };
}

const source = createSource(), search = createSearch();
for (let i = 1; i <= 2_000; i++) source.write(`p${i}`, { title: `Product ${i}`, price: i });
let writesDuringCopy = 0;
source.duringCopy = () => {                               // the application keeps writing mid-reindex
  source.write(`p${7 + writesDuringCopy}`, { title: 'Renamed', price: 1 });
  source.remove(`p${1_000 + writesDuringCopy}`);
  writesDuringCopy++;
};

await reindex({ source, search, alias: 'products', newIndex: 'products_v2', mapping: {},
  toDoc: (r) => ({ title: r.title, price_cents: r.price * 100 }) });
const expected = source.current();
const mismatches = [...expected.keys()].filter((id) => search.get('products', id)?.doc?.title !== expected.get(id).title);
console.log('writes during copy:', writesDuringCopy, '| docs:', await search.count('products'), '| expected:', expected.size, '| mismatches:', mismatches.length);
console.log('p7 →', search.get('products', 'p7').doc.title, '| p1000 deleted:', search.get('products', 'p1000').deleted === true);
// writes during copy: 4 | docs: 1996 | expected: 1996 | mismatches: 0
// p7 → Renamed | p1000 deleted: true
```

The decision to defend is **replaying from a recorded log position instead of dual-writing every change
to both indexes during the rebuild.** Dual-writing looks simpler — point the live indexer at both
indexes and run the backfill alongside — but it has a race at its core: the backfill reads a row, a
live write updates it in both indexes, and then the backfill writes its now-stale copy into the new
index. Avoiding that requires exactly the external versioning used here anyway, plus a second write path
in the live indexer that must be added, tested, and removed for every reindex. Recording a position and
replaying from it keeps one write path, makes the new index's state a pure function of "snapshot at P
plus the log after P", and gives a precise answer to "is it caught up?" — the replay position equals the
log head. The final replay after the swap closes the only remaining window, and versions make it safe to
apply anything twice.

---

## 18.10 Questions

**1.** MCQ — What does an inverted index map? a) Documents to their words b) Terms to the documents containing them c) Documents to their scores d) Queries to results
<details><summary>Answer</summary>**b)** Each term points to a postings list of documents (and often positions) — the reverse of a document's word list.</details>

**2.** MCQ — In BM25, why does a rare term contribute more to a score than a common one? a) It's longer b) Its inverse document frequency is higher, so it's more informative c) It appears in titles d) It's stemmed
<details><summary>Answer</summary>**b)** A term in few documents distinguishes them better; IDF rewards that.</details>

**3.** MCQ — A document indexed now isn't returned by a search one millisecond later. Why? a) Replication lag b) It's not searchable until the next refresh writes a segment c) Merges are running d) The query was cached
<details><summary>Answer</summary>**b)** Search engines are near-real-time; refresh (typically every second) makes new documents visible.</details>

**4.** MCQ — What's the best way to page deeply through search results? a) `from`/`size` with large `from` b) `search_after` with the last result's sort values, plus a point-in-time snapshot c) Requesting all results at once d) Increasing the shard count
<details><summary>Answer</summary>**b)** Cursor-style paging avoids every shard returning `from + size` results.</details>

**5.** MCQ — Why should "in stock" be a filter rather than part of the scored query? a) Filters are scored higher b) Filters aren't scored and can be cached and reused across queries c) Queries can't express booleans d) Filters run on replicas only
<details><summary>Answer</summary>**b)** Pure restrictions shouldn't affect ranking, and cached filter bitsets make them nearly free.</details>

**6.** MCQ — What does an index alias enable? a) Faster queries b) Swapping readers to a rebuilt index atomically, and rolling back c) Cross-cluster replication d) Automatic stemming
<details><summary>Answer</summary>**b)** Clients address the alias; repointing it moves all readers at once without client changes.</details>

**7.** Short — Why must the same analysis be applied at index time and query time?
<details><summary>Answer</summary>Matching compares analysed terms. If documents are lower-cased and stemmed but queries aren't (or vice versa), "Shoes" in a query never matches "shoe" in the index.</details>

**8.** Short — How does external versioning make a CDC-fed indexer safe against redelivery and reordering?
<details><summary>Answer</summary>Each write carries the source's version (log position or row version), and the engine ignores writes not newer than what it holds — so replays are no-ops and an older update arriving late can't overwrite a newer one.</details>

**9.** Short — Why does denormalising into search documents complicate indexing?
<details><summary>Answer</summary>A search document combines data from several sources (brand, category, price). A change to any of them must re-index every affected document, so the indexer must know which documents each kind of source change touches.</details>

**10.** Short — What is reciprocal rank fusion, and why is it used for hybrid search?
<details><summary>Answer</summary>Each document's fused score is Σ 1/(k + rank) over the ranked lists it appears in. It combines keyword and vector results using only ranks, so it needs no calibration between BM25 scores and similarity scores.</details>

**11.** Short — Why does adding replicas increase query throughput but not indexing throughput?
<details><summary>Answer</summary>Queries can be served by any copy, so more copies serve more queries; but every replica must index every document, so indexing work grows with replicas rather than being spread by them.</details>

**12.** Output — Using §18.3's `analyze`, what does `analyze('The Hiking Boots and Socks')` return?
<details><summary>Answer</summary>`[ 'hik', 'boot', 'sock' ]` — "the" and "and" are stop words; "hiking" loses "ing"; "boots" and "socks" lose the trailing "s".</details>

**13.** Output — Using §18.3's index, what does `idx.search('merino')` return?
<details><summary>Answer</summary>`[ 'p3:1.15' ]` — only p3 contains "merino". The rare term has a high IDF (≈ 1.25), but p3 has six terms against an average of four, so length normalisation pulls its score below the IDF.</details>

**14.** Output — A query with `from: 2000, size: 50` runs on an index with 8 shards. How many results do the shards send to the coordinator in total?
<details><summary>Answer</summary>**16,400.** Each shard returns its top 2,050, and 8 × 2,050 = 16,400 are merged to keep 50.</details>

**15.** Output — Reciprocal rank fusion with k = 60: document D is rank 1 in the keyword list and rank 3 in the vector list. What is its fused score, to four decimal places?
<details><summary>Answer</summary>**0.0323.** 1/61 + 1/63 ≈ 0.01639 + 0.01587 = 0.03227.</details>

**16.** Debug — Products deleted from the catalogue occasionally reappear in search a few minutes later and stay there. The indexer consumes CDC events from several partitions. Cause and fix?
<details><summary>Answer</summary>Out-of-order delivery: an older update for a product arrives after its delete and re-creates the document. Use external versions from the source on every write and delete (with versioned delete tombstones), so the older update is rejected.</details>

**17.** Debug — After renaming a brand, searches for the new brand name return nothing for its products, though the product rows didn't change. Why?
<details><summary>Answer</summary>Brand names are denormalised into product search documents, but the indexer only re-indexes on product changes. Brand changes must trigger re-indexing of every product of that brand (or the query must use a brand ID filter rather than the denormalised name).</details>

**18.** Debug — Search p99 jumps from 80 ms to 2 s for a few minutes several times a day, coinciding with bulk imports from a supplier feed. What's happening, and what do you change?
<details><summary>Answer</summary>Bulk indexing creates many segments, frequent refreshes, and heavy merges that compete with queries for I/O and CPU. Throttle bulk imports, lengthen the refresh interval during them, schedule them off-peak, and consider separating indexing-heavy and query-heavy load (e.g. dedicated ingest nodes or replicas).</details>

**19.** Debug — A multi-tenant SaaS's search occasionally shows one tenant's documents to another. The client sends `tenant_id` as a query filter. What's wrong?
<details><summary>Answer</summary>The tenant filter is trusted from the client and can be omitted or altered. Apply the tenant filter server-side from the authenticated context (or via filtered aliases per tenant), never from request parameters, and add tests that run queries under a different tenant.</details>

**20.** Code — Write `rrf(lists, k = 60)` that fuses several ranked lists of IDs into one list ordered by reciprocal-rank score.
<details><summary>Answer</summary>

```js
function rrf(lists, k = 60) {
  const scores = new Map();
  for (const list of lists) list.forEach((id, i) => scores.set(id, (scores.get(id) ?? 0) + 1 / (k + i + 1)));
  return [...scores].sort((a, b) => b[1] - a[1]).map(([id]) => id);
}
console.log(rrf([['a', 'b', 'c'], ['c', 'a', 'd']])); // [ 'a', 'c', 'b', 'd' ]
```
</details>

**21.** Code — Write `intersect(a, b)` that intersects two sorted postings lists of document IDs in linear time.
<details><summary>Answer</summary>

```js
function intersect(a, b) {
  const out = [];
  let i = 0, j = 0;
  while (i < a.length && j < b.length) {
    if (a[i] === b[j]) { out.push(a[i]); i++; j++; }
    else if (a[i] < b[j]) i++;   // advance whichever list is behind
    else j++;
  }
  return out;
}
console.log(intersect([1, 4, 7, 9, 12], [2, 4, 9, 10, 12])); // [ 4, 9, 12 ]
```
</details>

**22.** Code — Write `editDistance(a, b)` (Levenshtein) and use it to check whether "boost" is within distance 1 of "boots".
<details><summary>Answer</summary>

```js
function editDistance(a, b) {
  let prev = Array.from({ length: b.length + 1 }, (_, j) => j);
  for (let i = 1; i <= a.length; i++) {
    const cur = [i];
    for (let j = 1; j <= b.length; j++) {
      cur[j] = Math.min(prev[j] + 1, cur[j - 1] + 1, prev[j - 1] + (a[i - 1] === b[j - 1] ? 0 : 1));
    }
    prev = cur;
  }
  return prev[b.length];
}
console.log(editDistance('boost', 'boots'), editDistance('bots', 'boots')); // 2 1
```
"boost" → "boots" needs two substitutions under plain Levenshtein; engines that count adjacent transpositions as one edit (Damerau–Levenshtein) put it at distance 1.</details>

**23.** Scenario — A marketplace with 2 million products in PostgreSQL wants better search: typo tolerance, facets by brand and price, and relevance tuned by sales. Search traffic is 300 queries/s. Would you add a search engine, and how would you build it?
<details><summary>Answer</summary>Yes — facets over large result sets, typo tolerance, and sales-weighted relevance at 300 QPS are where a search engine pays off. Feed it from PostgreSQL via CDC into an indexer that builds denormalised product documents (brand, category path, price, stock, sales rank) with external versions; read through an alias; put filters (stock, category) in filter context; tune ranking with boosts and a sales signal, measured against judgement sets and zero-result rate. Keep checkout and stock checks on PostgreSQL.</details>

**24.** Scenario — You need to change the analyzer for product titles (adding synonyms and a new stemmer) on a 50 GB index serving production traffic. Plan the change.
<details><summary>Answer</summary>Create `products_v2` with the new mapping, snapshot the source at a recorded position, bulk-copy with refresh disabled and throttled, replay changes since the position, then verify counts and run a judgement set and sampled real queries against both indexes to compare relevance. Swap the alias (possibly for a percentage of traffic first via a second alias and routing), replay once more, watch click-through and zero-result rates, and keep `products_v1` for rollback until confident.</details>

---

## Chapter Summary

- Search answers "most relevant", not "matching" — a different problem with its own index — but start
  with the database's full-text search and add an engine when relevance, facets, or scale demand it.
- Inverted indexes map analysed terms to postings lists; identical analysis at index and query time is
  what makes them match.
- BM25 ranks by term rarity, saturating frequency, and document length; production ranking adds field
  boosts, business signals, learning to rank, and hybrid fusion with vector search.
- Clusters shard documents (scatter-gather queries), replicate for read throughput, refresh for
  near-real-time visibility, and merge segments in the background; deep pagination needs cursors.
- Feed the index from committed changes with external versions so redelivery, reordering, and deletes
  are safe, and treat indexing lag as an SLI.
- Read and write through aliases and make rebuilding routine: snapshot at a position, copy, replay,
  verify, swap, replay again.

---

## Practice Exercises

1. Implement PostgreSQL full-text search (`tsvector`, `GIN`, `ts_rank`) for a product table and compare
   its results on ten queries with §18.3's BM25 implementation.
2. Extend §18.3's index with positions and implement phrase queries ("hiking boots" as adjacent terms).
3. Read your search engine's documentation on refresh intervals, segment merging, and `search_after`,
   and summarise the settings you'd change for a bulk load.
4. Build a judgement set of 30 queries with known good results for a dataset you have, compute NDCG@10
   for BM25 alone, then with a title boost, and compare.
5. Run §18.9's reindexer against a real search engine and a PostgreSQL source with logical decoding,
   with a load generator writing throughout, and verify zero mismatches after the swap.

---

## Mini Project: A Product Search Service

Build a product search service over a PostgreSQL catalogue with a search engine (or your own inverted
index) kept in sync by CDC, supporting relevance tuning and zero-downtime schema changes.

- An indexer consuming committed changes, building denormalised documents, writing with external
  versions, and re-indexing dependent documents when brands or categories change.
- Queries with BM25 relevance, a title boost, a popularity signal, filters, and facets by brand and
  price range.
- Cursor pagination with `search_after`, and query guardrails: timeouts, result window caps, and a
  rejection of leading-wildcard queries.
- A reindex command implementing §18.9 against the real engine, with rollback by alias.
- Metrics for indexing lag, indexing errors, query latency, and zero-result rate.

---

**Next:** [Chapter 19 — Distributed Systems Fundamentals →](19-distributed-fundamentals.md)
