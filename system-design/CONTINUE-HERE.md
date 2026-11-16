# Maintainer Notes

This book is **in progress**. This file is the continuation brief: the planned inventory, the
conventions every chapter is written to, and the checks to run before committing.

For repository-wide conventions, see [../CLAUDE.md](../CLAUDE.md) and [../_template/](../_template/).

---

## 1. Current Status

Planned: **48 chapters + 7 appendices** in seven parts. The count was chosen to fit the subject —
33 concept chapters (Parts I–VI) and 15 application chapters (Part VII: one method chapter and
fourteen case studies). It is deliberately not the JavaScript book's 33. Case studies are where a
system design book earns its keep, so they are full chapters, not an appendix.

A chapter is "done" when it is written, passes §5, and its README row is linked (no
`*(pending)*`). Update the README row and this table together.

### Chapters (`chapters/`)

| # | File | Topic | Status |
|---|---|---|---|
| 1 | `01-what-system-design-is.md` | Requirements, NFRs, trade-offs, the design loop, design docs | done |
| 2 | `02-estimation.md` | Latency numbers, QPS/storage/bandwidth math, Little's law, sanity checks | done |
| 3 | `03-latency-throughput-queues.md` | Percentiles, tail amplification, utilization curves, USL | done |
| 4 | `04-availability-slos.md` | Nines, series/parallel math, SLI/SLO/SLA, error budgets | done |
| 5 | `05-networking.md` | TCP/TLS handshakes, HTTP/1.1/2/3, keep-alive, pooling, NAT/ports | done |
| 6 | `06-dns-cdn-edge.md` | DNS resolution and TTLs, CDN caching, cache keys, edge compute | done |
| 7 | `07-load-balancing.md` | L4 vs L7, algorithms, health checks, draining, sticky sessions | done |
| 8 | `08-api-design.md` | REST/gRPC/GraphQL, pagination, idempotency keys, versioning | done |
| 9 | `09-real-time-delivery.md` | Polling, long polling, SSE, WebSockets, connection fan-out | done |
| 10 | `10-storage-engines.md` | WAL, B-tree vs LSM, compaction, amplification, secondary indexes | done |
| 11 | `11-choosing-a-data-store.md` | Relational, document, wide-column, KV, graph, time-series | done |
| 12 | `12-data-modeling.md` | Access-pattern-first design, denormalization, hot keys | done |
| 13 | `13-replication.md` | Leader/follower, multi-leader, leaderless, lag anomalies, quorums | done |
| 14 | `14-partitioning.md` | Range vs hash, consistent hashing, rebalancing, hot partitions | done |
| 15 | `15-transactions-isolation.md` | Anomalies, isolation levels, MVCC, 2PL, SSI, locking | done |
| 16 | `16-caching.md` | Patterns, invalidation, stampedes, eviction, multi-tier | done |
| 17 | `17-object-storage.md` | Object stores, pre-signed URLs, multipart, chunking, dedup | done |
| 18 | `18-search.md` | Inverted indexes, relevance, CDC into search, reindexing | done |
| 19 | `19-distributed-fundamentals.md` | Partial failure, timeouts, network/clock/process faults | done |
| 20 | `20-consistency-models.md` | Linearizability, causal, session guarantees, CAP, PACELC | done |
| 21 | `21-consensus-coordination.md` | Raft, leader election, leases, fencing tokens, etcd/ZooKeeper | done |
| 22 | `22-clocks-ordering-ids.md` | Lamport/vector/HLC clocks, Snowflake, UUIDv7 | done |
| 23 | `23-distributed-transactions.md` | 2PC, sagas, outbox, idempotency, "exactly once" | done |
| 24 | `24-queues-logs.md` | Queues vs logs, delivery semantics, ordering, DLQs, backpressure | done |
| 25 | `25-stream-batch-processing.md` | Batch vs stream, windows, watermarks, state, lambda/kappa | done |
| 26 | `26-event-driven-architecture.md` | Events vs commands, choreography/orchestration, CQRS, ES | done |
| 27 | `27-service-architecture.md` | Monolith, modular monolith, microservices, gateways, mesh | done |
| 28 | `28-resilience-patterns.md` | Timeouts, retries + jitter, breakers, bulkheads, hedging | done |
| 29 | `29-rate-limiting-load-shedding.md` | Token/leaky bucket, windows, distributed limits, shedding | done |
| 30 | `30-observability.md` | Metrics/logs/traces, RED/USE, SLO alerting, cardinality | done |
| 31 | `31-deployment-change.md` | Canary, blue/green, flags, schema migrations, rollback | done |
| 32 | `32-multi-region-dr.md` | Active-passive/active-active, RPO/RTO, failover, data residency | done |
| 33 | `33-security-architecture.md` | Trust boundaries, authn/z architecture, secrets, tenancy | done |
| 34 | `34-design-interview-method.md` | The 45-minute structure, signalling, common failure modes | done |
| 35 | `35-url-shortener.md` | ID generation, redirects, read-heavy caching, analytics | done |
| 36 | `36-news-feed.md` | Fan-out on write/read, hybrid, ranking, celebrities | done |
| 37 | `37-chat-messaging.md` | Connection gateways, per-conversation ordering, receipts, presence | done |
| 38 | `38-notification-system.md` | Channels, priorities, preferences, dedup, provider failover | done |
| 39 | `39-key-value-store.md` | Dynamo-style: ring, quorums, hinted handoff, anti-entropy | done |
| 40 | `40-video-streaming.md` | Upload, transcoding DAG, ABR/HLS, CDN economics | done |
| 41 | `41-ride-hailing-proximity.md` | Geohash/S2/quadtree, location ingest, matching | done |
| 42 | `42-search-autocomplete.md` | Tries, precomputed top-k, freshness, latency budget | done |
| 43 | `43-payment-system.md` | Double-entry ledger, idempotency, PSPs, reconciliation | done |
| 44 | `44-web-crawler.md` | Frontier, politeness, dedup, DNS, traps | done |
| 45 | `45-job-scheduler.md` | Durable timers, leasing, at-least-once execution, fairness | done |
| 46 | `46-collaborative-editing.md` | OT vs CRDT, presence, persistence, offline | done |
| 47 | `47-ticketing-flash-sales.md` | Seat holds, contention, queues/waiting rooms, oversell | done |
| 48 | `48-metrics-monitoring.md` | TSDB ingest, compression, downsampling, cardinality, alerting | pending |

### Appendices (`appendices/`)

| # | File | Contents | Status |
|---|---|---|---|
| A | `A-interview-questions.md` | Questions by topic, each with a **Follow-up:** line | pending |
| B | `B-coding-challenges.md` | Building blocks implemented (LRU, rate limiter, ring, bloom…) with solutions | pending |
| C | `C-projects.md` | Project specs, easy → expert, no solutions | pending |
| D | `D-roadmap.md` | Phases, checkpoints, interview sprint | pending |
| E | `E-best-practices.md` | Design-review checklist by category | pending |
| F | `F-cheatsheet.md` | Latency/capacity numbers, trade-off tables, trap table | pending |
| G | `G-glossary.md` | Terms, alphabetical, chapter cross-references | pending |

---

## 2. Chapter Contract

Every chapter contains these sections in this order. The canonical source is
[`../_template/chapters/00-chapter-template.md`](../_template/chapters/00-chapter-template.md).

```markdown
# Chapter N — Title
## Learning Objectives          3–5 bullets, action verbs
## N.1 … N.x                    explanation, diagrams, numbers, commented code
## N.x Common Mistakes and Best Practices    5–7 mistakes, 2–4 best practices
## N.y Real-World Example: <Named Thing>     40–90 lines, then ONE paragraph on the
                                            single non-obvious decision
## N.z Questions                20–28, inline format, continuously numbered
## Chapter Summary              5–7 bullets, takeaways not topic names
## Practice Exercises           5 tasks, increasing difficulty, NO solutions
## Mini Project: <Name>
**Next:** [Chapter N+1 — Title →](NN-slug.md)
```

### Case-study chapters (35–48)

Same contract; the numbered sections follow the design-interview order from Chapter 34, so a
reader can compare any two case studies section by section:

1. Requirements and scope (functional, non-functional, explicitly out of scope)
2. Estimation (with the arithmetic shown)
3. API and data model
4. High-level design (one diagram)
5. Deep dives — two to four sections, one per genuinely hard sub-problem
6. Failure modes, bottlenecks, and what changes at 10×

The Real-World Example in a case study is code for the hardest component (the ID allocator, the
fan-out worker, the seat-hold reservation) — not a sketch of the whole system.

### Question labels

The six labels are kept so the repo-wide greps work. In this book:

- **Output** covers both "what does this code print" and "what is the result" questions with a
  single determinate answer — a computed estimate, a quorum outcome, the order a log is replayed.
  Every Output question has exactly one right answer; judgement calls are Scenarios.
- **Debug** is a production symptom described in prose ("p99 doubles every day at 02:00"), asking
  for cause and fix. It rarely has code.
- **Code** asks for a building block in JavaScript.

Mix used throughout: MCQ (6) → Short (5) → Output (4) → Debug (4–5) → Code (3) → Scenario (2–3).

---

## 3. Book-Specific Conventions

**Mechanism before box.** Every component introduced is explained by what it does internally and
what that costs, before it appears in a diagram. "Add a cache" is never a sufficient sentence.

**Show the arithmetic.** Any capacity claim shows its calculation, either inline or in a runnable
JS block. Round numbers are fine (and preferred); unexplained numbers are not.

**Diagrams are ```text blocks** using box-drawing characters and arrows (`──▶`, `│`, `└──`). No
Mermaid — it does not render in every Markdown viewer the README promises.

**Code is JavaScript, Node 22+, ESM, dependency-free** unless the block states otherwise, so each
block runs with `node file.mjs`. SQL is PostgreSQL 16+. Simulations are seeded or deterministic
when an Output question depends on them.

**Vendor-neutral by default.** Name a product when it is the clearest illustration, describe its
behaviour as of late 2026, and say which property generalises.

**Every trade-off names the condition under which the other choice wins.** This applies to prose,
tables, and Scenario answers alike.

**Chapter ownership.** One chapter owns each concept; others link to it instead of re-explaining:
latency numbers → Ch 2; queueing/percentiles → Ch 3; quorums → Ch 13; consistent hashing → Ch 14;
isolation levels → Ch 15; idempotency keys → Ch 8; outbox → Ch 23; delivery semantics → Ch 24;
retries/backoff → Ch 28; rate-limiting algorithms → Ch 29. Keep it that way — a second copy will
drift.

**The real-world example's closing paragraph defends one decision.** Not a walkthrough — the single
choice a reader would otherwise question, and why the obvious alternative is worse.

---

## 4. Known Deviations

Deliberate, and recorded so nobody "fixes" them by accident.

**48 chapters, not 33.** See §1. Do not merge case studies to shrink the count, or split concept
chapters to grow it.

**"Output" questions include computed results** rather than only printed program output (§2).

**Diagrams are text, not images or Mermaid** (§3).

_(Add entries here as chapters are written and deviate for good reason.)_

---

## 5. Verification

Before committing a change to any chapter or appendix, from `system-design/`:

```bash
F=chapters/01-what-system-design-is.md

# <details> tags balance
echo "open=$(grep -c '<details>' $F) close=$(grep -c '</details>' $F)"

# Numbered and labelled questions must be equal, with no numbering gaps
grep -c '^\*\*[0-9]\+\.\*\*' $F
grep -c '^\*\*[0-9]\+\.\*\* \(MCQ\|Short\|Output\|Debug\|Code\|Scenario\) —' $F
grep -o '^\*\*[0-9]\+\.\*\*' $F | grep -o '[0-9]\+' | tr '\n' ' '

# Section numbers match the filename's chapter number
grep -o '^## [0-9]\+\.[0-9]\+' $F

# Code fences balance (even count)
grep -c '^```' $F
```

Run every `js` block that claims output, and check the trailing comments match. Blocks are written
to run standalone, so extracting them is mechanical:

```js
// extract.mjs — write each ```js block of a chapter to its own file: node extract.mjs <chapter.md> <outdir>
import { readFileSync, writeFileSync, mkdirSync } from 'node:fs';
const [, , md, out] = process.argv;
mkdirSync(out, { recursive: true });
const src = readFileSync(md, 'utf8');
let m, i = 0;
for (const re = /^```js\n([\s\S]*?)^```/gm; (m = re.exec(src)); ) {
  const line = src.slice(0, m.index).split('\n').length;
  writeFileSync(`${out}/b${String(++i).padStart(2, '0')}_L${line}.mjs`, m[1]);
}
console.log(`${i} blocks`);
```

Then `for f in out/*.mjs; do node "$f"; done`. Blocks marked as requiring a database or broker
(the outbox relay in Ch 23, for instance) define functions without running them. Also check that
every `§N.M` cross-reference points at the section it names — section numbers shift when a
chapter has one more or fewer concept sections, and the real-world example is always the section
after Common Mistakes.

Whole-book checks:

- Every internal link resolves, including each file's `**Next:**` footer. The chain runs
  ch01 → … → ch48 → Appendix A → … → Appendix G (G has no footer; it ends the book).
- Cross-references (`→ Ch 14`, `Chapter 20`) name a chapter that exists and covers that topic.
- When finished: no `*(pending)*` rows remain in the README, and the status block is removed.

---

## 6. Adding a Chapter

1. Copy [`../_template/chapters/00-chapter-template.md`](../_template/chapters/00-chapter-template.md)
   to `chapters/NN-slug.md`, using the number and slug from §1.
2. Write to the contract in §2 and the conventions in §3. For a case study, use the section order
   in §2 "Case-study chapters".
3. Run §5. Run the code.
4. Link the README row, mark it done in §1, and make sure the previous chapter's `**Next:**`
   footer points at the new file.
