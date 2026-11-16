# System Design: From Fundamentals to Expert

> **Status — work in progress.** Chapters marked *(pending)* below are not yet written.
> _(Delete this block once the topic is finished.)_

**How to design systems that survive real traffic, real failures, and a senior interview — by
reasoning from mechanisms and numbers instead of reciting architecture diagrams.**

---

## Who This Book Is For

You have built and shipped a web application. You know what a database index is, you have called
an HTTP API, and you have probably put Redis in front of something because it was slow. What you
may not have is a way to reason about the system as a whole: why adding a replica made reads
*inconsistent*, why the retry you added turned a blip into an outage, why the cache you trusted
served one user another user's data, or how to tell whether a design needs a queue at all.

This book does not hand you boxes to draw. It teaches the mechanism under each box — what a
B-tree does that an LSM tree doesn't, what "quorum" actually guarantees, why at-least-once
delivery forces idempotency on you, why p99 latency is a queueing problem — and then puts those
mechanisms together in fourteen end-to-end case studies. Every case study commits to numbers,
names the bottleneck, and says what it would change at 10× scale. That is the difference between
a design that sounds right and one that is.

## How to Use This Book

1. **Linear read** — Chapters 1–33 build the toolkit in order; Chapters 34–48 apply it. Each
   chapter cross-references the ones it depends on by number.
2. **Interview prep** — Chapters 1–4, then 13, 14, 16, 20, 24, and 28, then the method in
   Chapter 34 and as many case studies (35–48) as you have time for. Finish with
   Appendix A *(pending)*.
3. **Working reference** — Appendix F *(pending)* (cheat sheet, including the latency and
   capacity numbers) and Appendix G *(pending)* (glossary) are meant to stay open in a tab.

Every chapter follows the same shape:

- Learning objectives
- Explanations with diagrams, numbers, and runnable, commented code
- Common mistakes and best practices
- A real-world example
- 20–28 questions (MCQ, short answer, output prediction, debugging, coding, scenario)
- Chapter summary
- Practice exercises
- Mini project (where the topic supports one)

Do the arithmetic yourself. A design whose numbers you haven't checked is a guess with boxes
around it.

---

## Table of Contents

### Part I — Thinking in Systems

| # | Chapter | Core Idea |
|---|---------|-----------|
| 1 | [What System Design Is](chapters/01-what-system-design-is.md) | Trade-offs under constraints, made explicit |
| 2 | [Back-of-the-Envelope Estimation](chapters/02-estimation.md) | Numbers that decide the architecture |
| 3 | [Latency, Throughput, and Queueing](chapters/03-latency-throughput-queues.md) | Why p99 explodes before CPU hits 100% |
| 4 | [Availability, SLOs, and Failure Math](chapters/04-availability-slos.md) | Nines are a budget, not a boast |

### Part II — The Request Path

| # | Chapter | Core Idea |
|---|---------|-----------|
| 5 | [Networking for System Designers](chapters/05-networking.md) | Round trips are the unit of cost |
| 6 | [DNS, CDNs, and the Edge](chapters/06-dns-cdn-edge.md) | Serve it before it reaches you |
| 7 | [Load Balancing](chapters/07-load-balancing.md) | Spreading load without spreading failure |
| 8 | [API Design](chapters/08-api-design.md) | Contracts that survive retries and versions |
| 9 | [Real-Time Delivery](chapters/09-real-time-delivery.md) | Polling, SSE, WebSockets, and the fan-out bill |

### Part III — Data

| # | Chapter | Core Idea |
|---|---------|-----------|
| 10 | [Storage Engines and Indexes](chapters/10-storage-engines.md) | B-trees, LSM trees, and the write path |
| 11 | [Choosing a Data Store](chapters/11-choosing-a-data-store.md) | Pick by access pattern, not by fashion |
| 12 | [Data Modeling for Access Patterns](chapters/12-data-modeling.md) | Design the queries first |
| 13 | [Replication](chapters/13-replication.md) | Copies, lag, and the anomalies lag causes |
| 14 | [Partitioning](chapters/14-partitioning.md) | Splitting data without splitting queries |
| 15 | [Transactions and Isolation](chapters/15-transactions-isolation.md) | What each isolation level actually prevents |
| 16 | [Caching](chapters/16-caching.md) | The fastest query is the one you skip |
| 17 | [Object Storage and Large Files](chapters/17-object-storage.md) | Bytes don't belong in your database |
| 18 | [Search](chapters/18-search.md) | Inverted indexes and keeping them in sync |

### Part IV — Distributed Systems

| # | Chapter | Core Idea |
|---|---------|-----------|
| 19 | [Distributed Systems Fundamentals](chapters/19-distributed-fundamentals.md) | Partial failure is the normal case |
| 20 | [Consistency Models](chapters/20-consistency-models.md) | Linearizable, causal, eventual — and CAP honestly |
| 21 | [Consensus, Leases, and Coordination](chapters/21-consensus-coordination.md) | Agreeing when nodes lie by being slow |
| 22 | [Clocks, Ordering, and Unique IDs](chapters/22-clocks-ordering-ids.md) | Wall clocks lie; logical clocks don't |
| 23 | [Distributed Transactions and Sagas](chapters/23-distributed-transactions.md) | Atomicity across services, without 2PC |

### Part V — Asynchronous Systems

| # | Chapter | Core Idea |
|---|---------|-----------|
| 24 | [Message Queues and Event Logs](chapters/24-queues-logs.md) | Decoupling in time, and its delivery semantics |
| 25 | [Stream and Batch Processing](chapters/25-stream-batch-processing.md) | Bounded and unbounded data, windows, watermarks |
| 26 | [Event-Driven Architecture](chapters/26-event-driven-architecture.md) | Events, CQRS, event sourcing — and when not to |

### Part VI — Operating at Scale

| # | Chapter | Core Idea |
|---|---------|-----------|
| 27 | [Service Boundaries and Architecture Styles](chapters/27-service-architecture.md) | Monolith first, split on evidence |
| 28 | [Resilience Patterns](chapters/28-resilience-patterns.md) | Timeouts, retries, breakers, bulkheads |
| 29 | [Rate Limiting and Load Shedding](chapters/29-rate-limiting-load-shedding.md) | Saying no early and cheaply |
| 30 | [Observability](chapters/30-observability.md) | Metrics, logs, traces, and alerts that matter |
| 31 | [Deployment and Change Management](chapters/31-deployment-change.md) | Most outages are changes |
| 32 | [Multi-Region and Disaster Recovery](chapters/32-multi-region-dr.md) | RPO, RTO, and the cost of each nine |
| 33 | [Security Architecture](chapters/33-security-architecture.md) | Trust boundaries drawn on the diagram |

### Part VII — Case Studies

| # | Chapter | Core Idea |
|---|---------|-----------|
| 34 | [The Design Interview Method](chapters/34-design-interview-method.md) | A repeatable 45-minute structure |
| 35 | [Design a URL Shortener](chapters/35-url-shortener.md) | ID generation and a read-heavy cache |
| 36 | [Design a News Feed](chapters/36-news-feed.md) | Fan-out on write vs read, and celebrities |
| 37 | [Design a Chat System](chapters/37-chat-messaging.md) | Connections, ordering, delivery receipts |
| 38 | [Design a Notification System](chapters/38-notification-system.md) | Many channels, priorities, no duplicates |
| 39 | [Design a Distributed Key-Value Store](chapters/39-key-value-store.md) | Dynamo from first principles |
| 40 | [Design a Video Streaming Platform](chapters/40-video-streaming.md) | Transcoding pipelines and adaptive bitrate |
| 41 | [Design Ride-Hailing and Proximity Search](chapters/41-ride-hailing-proximity.md) | Geospatial indexes under constant writes |
| 42 | [Design Search Autocomplete](chapters/42-search-autocomplete.md) | Tries, top-k, and a 50 ms budget |
| 43 | [Design a Payment System](chapters/43-payment-system.md) | Ledgers, idempotency, reconciliation |
| 44 | [Design a Web Crawler](chapters/44-web-crawler.md) | Politeness, frontiers, and dedup at scale |
| 45 | [Design a Distributed Job Scheduler](chapters/45-job-scheduler.md) | Exactly-once intent from at-least-once parts |
| 46 | [Design a Collaborative Editor](chapters/46-collaborative-editing.md) | OT, CRDTs, and convergence |
| 47 | [Design Ticketing and Flash Sales](chapters/47-ticketing-flash-sales.md) | Inventory under extreme contention |
| 48 | Design a Metrics and Monitoring System *(pending)* | Time-series ingestion and cardinality |

### Appendices

| # | Appendix | Contents |
|---|----------|----------|
| A | Interview Questions *(pending)* | Questions by topic, each with a follow-up |
| B | Design Drills and Coding Challenges *(pending)* | Building blocks implemented, with solutions |
| C | Projects *(pending)* | Buildable project specs, easy → expert |
| D | Roadmap *(pending)* | Staged study plan with checkpoints |
| E | Best Practices Checklist *(pending)* | Design-review checklist by category |
| F | Cheat Sheet *(pending)* | Latency numbers, capacity math, trade-off tables |
| G | Glossary *(pending)* | Terms with chapter cross-references |

---

## Conventions

Code blocks are runnable unless marked otherwise. Output is shown as a trailing comment:

```js
const qps = (100_000_000 * 10) / 86_400;
console.log(Math.round(qps)); // 11574
```

Architecture diagrams are plain-text blocks, so they render identically everywhere:

```text
client ──▶ load balancer ──▶ app servers ──▶ primary DB
                                  │
                                  └──▶ cache
```

Warnings look like this:

> ⚠️ **Gotcha** — a concrete trap, not a general caution.

Interview-relevant points are flagged:

> 💼 **Interview** — how this actually gets asked.

---

## A Note on Environment

The ideas in this book are vendor-neutral; where a specific product is the clearest illustration
(PostgreSQL, Redis, Kafka, S3, DynamoDB, Cassandra, Elasticsearch), it is named, and its
behaviour is described as of late 2026. Products change their defaults — check the current
documentation before relying on one.

Code examples are JavaScript targeting **Node 22+** as ES modules, with no dependencies unless a
block says otherwise, so each one runs with `node file.mjs`. SQL examples target PostgreSQL 16+.
Numbers in estimates are deliberately round; the point is the order of magnitude, and the book
says so whenever a precise figure matters.

Start with [Chapter 1](chapters/01-what-system-design-is.md).
