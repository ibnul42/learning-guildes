# Chapter 34 — The Design Interview Method

## Learning Objectives

- Explain what a system design interview evaluates and how expectations differ by level
- Run a 45-minute design in six phases with a deliberate time budget
- Turn an open prompt into scoped requirements and a few decision-driving numbers in under ten minutes
- Choose and drive deep dives, stating mechanisms, numbers, failure modes, and trade-offs
- Avoid the common failure patterns that sink otherwise strong candidates

---

## 34.1 What Is Being Evaluated

A system design interview is a compressed version of the design loop from Chapter 1: an ambiguous problem, a
collaborator (the interviewer), a whiteboard, and 45 to 60 minutes. There's no single right answer, and interviewers
don't grade the final diagram. They watch **how you get there**, looking for signals that predict how you'll design
real systems with real colleagues:

| Signal | What strong looks like | What weak looks like |
|---|---|---|
| **Problem framing** | Clarifies users, scale, and must-haves before drawing; cuts scope explicitly | Starts drawing a generic architecture immediately |
| **Quantitative reasoning** | A few quick numbers that drive decisions ("~30 k reads/s → cache in front") | No numbers, or numbers that change nothing |
| **Design fundamentals** | Simple working design first; each component justified | Every box from every blog post, unjustified |
| **Depth** | Explains mechanisms inside components: how the cache invalidates, how the queue redelivers | Names products as answers ("use Kafka") |
| **Trade-offs** | Volunteers what each choice costs and when the alternative wins | Presents choices as obviously correct |
| **Failure thinking** | Asks what breaks, at what load, and what happens then | Only the happy path |
| **Communication** | Structured, thinks aloud, checks in, adapts to hints | Silent drawing, or monologue that ignores the interviewer |

Expectations rise with level. A mid-level candidate is expected to produce a sound, working design with some depth. A
senior candidate is expected to drive the conversation, find the genuinely hard parts unprompted, and reason about
failure and operations. A staff-level candidate is expected to frame the problem in business terms, make and defend
strategic trade-offs (build vs buy, consistency vs cost), anticipate organisational and evolutionary concerns, and
know where *not* to spend complexity.

> 💼 **Interview** — Interviewers often say "there's no right answer" — and mean it. What they won't say is that there
> are many wrong *processes*: skipping requirements, ignoring scale, refusing hints, or never committing to a decision.
> The method in this chapter exists to make the process visibly sound, so the conversation can spend its time on depth.

---

## 34.2 The 45-Minute Structure

Every case study in Chapters 35–48 follows the same six phases, in the same order, so they can be compared section by
section. In an interview, the phases also form a time budget:

```js
// A 45-minute interview, budgeted. The point isn't the exact minutes; it's that deep dives get
// the largest share, and that requirements and wrap-up are never skipped.
const plan = [
  ['Requirements and scope', 5], ['Estimation', 4], ['API and data model', 6],
  ['High-level design', 8], ['Deep dives', 17], ['Bottlenecks, failures, 10×', 5],
];
let clock = 0;
for (const [phase, minutes] of plan) {
  console.log(`${String(clock).padStart(2)}–${String(clock + minutes).padStart(2)} min  ${phase}`);
  clock += minutes;
}
console.log(`total ${clock} min; deep dives ${Math.round((17 / clock) * 100)}%`);
//  0– 5 min  Requirements and scope
//  5– 9 min  Estimation
//  9–15 min  API and data model
// 15–23 min  High-level design
// 23–40 min  Deep dives
// 40–45 min  Bottlenecks, failures, 10×
// total 45 min; deep dives 38%
```

The budget's shape matters more than its minutes. The most common time failure is spending 25 minutes on a high-level
design and having no time for depth — which is where seniority is demonstrated. The second most common is skipping
requirements to save time and then designing the wrong system. Glance at the clock at each phase boundary, and say so
when moving on: "I have a reasonable high-level design; let me go deep on the feed fan-out, since that's the hard part."

In a 60-minute interview, extend the deep dives; in a 35-minute one, compress the API and high-level phases, not the
requirements or the depth.

---

## 34.3 Phase 1: Requirements and Scope (≈5 minutes)

The prompt is deliberately vague — "design a URL shortener", "design WhatsApp" — and the first job is to make it
specific enough to design. Ask questions that change the architecture, write the answers down, and propose defaults
when the interviewer turns the question back to you.

**Functional requirements.** List the core use cases and then **cut** to the three or four that shape the system:
"Users send one-to-one and group messages, see delivery and read receipts, and get history on a new device. I'll leave
out voice calls, payments, and stories unless you want them." Saying what's out of scope is as valuable as what's in.

**Non-functional requirements.** The questions from Chapter 1, quantified:

| Question | Why it matters |
|---|---|
| How many users, and how active (DAU)? | Scale — and whether one machine is enough |
| Read/write ratio? | Caching, replicas, fan-out strategy |
| Latency targets for the key operations? | Synchronous paths, edge, caching |
| Consistency: what must never be stale or wrong? | Where to pay for coordination (Chapter 20) |
| Availability target, and for which operations? | Redundancy, degradation (Chapter 4) |
| Data size and retention? | Storage choice, tiering |
| Geography? | Multi-region, latency, residency (Chapter 32) |

Don't ask twenty questions. Ask the four or five that matter for *this* prompt, and offer assumptions for the rest:
"I'll assume 50 million daily users, mostly mobile, global, read-heavy — tell me if you'd like different numbers."
Interviewers appreciate a candidate who makes reasonable assumptions explicitly and keeps moving.

---

## 34.4 Phase 2: Estimation (≈4 minutes)

Estimation in an interview has one purpose: to find the numbers that **decide the design** (Chapter 2). Usually that's
three or four: peak write rate, peak read rate (and the ratio), storage over a few years, and sometimes bandwidth or
concurrent connections.

```js
// The estimation an interviewer actually wants: four numbers, each tied to a decision.
function quickEstimate({ dau, writesPerUser, readsPerUser, bytesPerWrite, peak = 3, years = 5 }) {
  const day = 1e5;                                   // seconds per day, rounded (Chapter 2)
  const w = (dau * writesPerUser) / day, r = (dau * readsPerUser) / day;
  const tb = (dau * writesPerUser * bytesPerWrite * 365 * years) / 1e12;
  return [
    `writes ~${Math.round(w)}/s avg, ~${Math.round(w * peak)}/s peak`,
    `reads ~${Math.round(r)}/s avg, ~${Math.round(r * peak)}/s peak (read:write ${Math.round(r / w)}:1)`,
    `storage ~${tb.toFixed(0)} TB raw over ${years} years`,
  ];
}
console.log(quickEstimate({ dau: 10e6, writesPerUser: 2, readsPerUser: 100, bytesPerWrite: 1_000 }).join('\n'));
// writes ~200/s avg, ~600/s peak
// reads ~10000/s avg, ~30000/s peak (read:write 50:1)
// storage ~37 TB raw over 5 years
```

Then **say what the numbers mean**: "600 writes per second is fine for a single primary; 30,000 reads per second with a
50:1 ratio means reads must come from a cache and replicas; 37 TB raw, nearer 200 TB with replication and indexes, so
we'll partition eventually but don't need to on day one." A number that doesn't lead to a sentence like that wasn't
worth computing.

Do the arithmetic aloud and round aggressively. Interviewers don't check digits; they check whether you can reason
quantitatively and whether the numbers influence your choices.

---

## 34.5 Phases 3 and 4: API, Data Model, and High-Level Design (≈14 minutes)

**API (a few minutes).** Sketch the handful of endpoints or messages that implement the core use cases — method, path,
key parameters, key response fields. This forces precision about what the system actually does, and it's where
mentioning **pagination** for list endpoints and **idempotency keys** for creation endpoints (Chapter 8) earns credit
quickly.

```text
POST /v1/messages          { conversationId, clientMessageId, body }  → { messageId, seq }
GET  /v1/conversations/:id/messages?before=<cursor>&limit=50         → { messages[], nextCursor }
```

**Data model (a few minutes).** The core entities, their keys, and — most importantly — the **access patterns** and
which store serves them (Chapters 11–12). The partition key is the decision that matters most: "messages partitioned by
conversation ID, sorted by sequence number, so loading a conversation is one range read".

**High-level design (about eight minutes).** Draw the **simplest design that satisfies the functional requirements**,
then walk one request through it end to end. Every box should have a reason you can state in a sentence. Resist adding
components the requirements don't yet demand — the deep dives and the 10× discussion are where complexity is justified.

```text
clients ─▶ LB ─▶ API service ─▶ messages DB (partitioned by conversation)
              │       └─▶ queue ─▶ delivery workers ─▶ connection gateways ─▶ recipients
              └─▶ connection gateways (WebSocket)
```

A good habit at the end of this phase: name the two or three hardest problems the design must solve, and propose them as
the deep dives. It shows you know where the difficulty is — and lets the interviewer redirect you if they'd rather probe
something else.

---

## 34.6 Phase 5: Deep Dives (≈17 minutes)

Deep dives are where the interview is decided. Pick (or be steered to) the genuinely hard sub-problems — usually where
scale, consistency, or failure bites — and go into **mechanism**, not vocabulary. A useful pattern for each:

1. **State the problem precisely.** "A celebrity with 30 million followers posts; fan-out on write means 30 million
   timeline inserts."
2. **Give two or three options with numbers.** "Fan-out on write costs 30 M writes per post; fan-out on read costs a
   merge of followed users' posts per timeline load."
3. **Choose, with the reason and the reversal condition.** "Hybrid: fan-out on write for normal accounts, on read for
   accounts above ~100 k followers — the threshold is tunable from metrics."
4. **Explain the failure behaviour.** "If the fan-out workers fall behind, timelines are stale but correct when merged
   with the celebrity pull; lag is the alert."

Topics that make good deep dives, and the chapters that prepare them:

| Recurring hard part | Mechanisms to reach for |
|---|---|
| Hot keys and skew | Sharding counters, local caches, splitting keys (Ch 12, 14) |
| Read scaling | Caching with invalidation and stampede protection, replicas with session guarantees (Ch 13, 16) |
| Write scaling | Partitioning, batching, LSM stores, async pipelines (Ch 10, 14, 24) |
| Exactly-once effects | Idempotency keys, outbox, inbox (Ch 8, 23) |
| Ordering | Partition keys, sequence numbers, logical clocks (Ch 22, 24) |
| Real-time delivery | Gateways, pub/sub backbone, resume from sequence (Ch 9) |
| Contention | Atomic conditional updates, queues, reservations (Ch 15, 47) |
| Failure handling | Timeouts, retries with budgets, breakers, failover (Ch 21, 28, 32) |

Interviewers often interrupt deep dives with "what if…" questions — "what if the cache goes down?", "what if a region
fails?". These are invitations, not attacks; answer them with mechanisms from Part IV and VI, and return to your thread.

---

## 34.7 Phase 6: Bottlenecks, Failures, and 10× (≈5 minutes)

Close by stepping back. Three short answers make a strong ending:

- **What breaks first as load grows?** "At 10× writes, the messages database's busiest partitions; I'd split hot
  conversations and add capacity by adding shards, which the slot map supports without downtime."
- **What happens when each stateful component fails?** "Gateway node dies: clients reconnect with jittered backoff and
  resume from their last sequence number. Queue lags: delivery is slower but nothing is lost. Primary database fails:
  automatic failover within the region, seconds of unavailability for writes to affected partitions."
- **What would you do with more time?** Monitoring and SLOs, abuse prevention, multi-region, cost — named briefly,
  showing awareness without starting a new design.

### Communication habits that help throughout

- **Think aloud**, but in structured chunks: "Three options here…"; "I'm choosing X because…".
- **Check in** at phase boundaries: "Does this level of detail work, or would you like me to go deeper on storage?"
- **Take hints.** An interviewer asking "how would that handle a celebrity account?" is pointing at a gap; follow it.
- **Commit.** Hedging every choice ("we could do A or B or C") reads as indecision. Choose, explain, and name when the
  other option would win.
- **Keep the diagram current** — update it as decisions change, so it always reflects the design you're defending.

---

## 34.8 Common Mistakes and Best Practices

**Mistake: drawing before asking.** A design for unstated requirements is a design for the wrong system.

**Mistake: estimation that changes nothing.** Numbers without conclusions waste minutes and signal box-ticking.

**Mistake: the kitchen-sink architecture.** Kafka, Kubernetes, three databases, and a service mesh for a problem that
needs a database and a cache signals pattern-matching, not judgement.

**Mistake: product names as answers.** "Use Cassandra" isn't a design; partition key, consistency level, and why are.

**Mistake: never reaching depth.** Spending the whole interview at the high level leaves no evidence of seniority.

**Mistake: ignoring the interviewer.** Missing hints, monologuing, or arguing with redirections turns a collaboration into
a performance.

**Mistake: no failure discussion.** A design that only works when everything works is half a design.

**Best practice: a fixed six-phase structure with a time budget.** Process becomes visible and reliable, and depth gets
the time it needs.

**Best practice: "X over Y, because Z, unless W" for every major choice.** It's the trade-off statement interviewers are
listening for (Chapter 1).

**Best practice: practise out loud, timed, with a whiteboard.** Reading designs and producing them under time pressure are
different skills.

---

## 34.9 Real-World Example: A Complete 45-Minute Whiteboard

This is what the whiteboard should look like at the end of a well-run 45-minute interview for "design a paste-sharing
service" — compressed, but with every phase present. The case studies that follow expand each phase into a full
section; this shows the shape in one view.

```text
1. REQUIREMENTS                                   2. ESTIMATES (round, then decide)
 F: create paste (text ≤ 1 MB), get by short URL, ·  10 M DAU; 1 paste/10 users/day → 1 M writes/day
    optional expiry, view count                   ·  ≈ 12 writes/s avg, ~40 peak  → one DB primary is fine
 F out: accounts, editing, syntax-aware search    ·  100 reads per write → ~1.2 k/s avg, ~4 k peak
 NF: read p99 < 100 ms; 99.95% for reads;         ·    → cache + CDN; 95% hit → ~200 DB reads/s
     pastes never lost once acknowledged          ·  avg 10 KB → 10 GB/day → ~18 TB/5 yr raw
     view counts may lag minutes                  ·    → bodies in object storage, metadata in DB

3. API                                            4. DATA MODEL
 POST /pastes {content, expiresIn?}               ·  pastes(id PK=base62 key, object_key, created,
   Idempotency-Key → {key, url}                   ·         expires_at, size, owner_ip_hash)
 GET /p/:key → 200 content | 404 | 410 expired    ·  bodies: object store, key = sha256(content)
                                                  ·  views: counter table, updated async (batched)
5. HIGH-LEVEL
 client ─▶ CDN ─▶ LB ─▶ paste API ─┬─▶ cache (key → metadata)
                                   ├─▶ Postgres (metadata) ─▶ replicas
                                   ├─▶ object store (bodies, content-addressed, immutable)
                                   └─▶ view events ─▶ queue ─▶ counter aggregator

6. DEEP DIVES
 a) Key generation: random 7-char base62 (3.5 T keys) + unique constraint + retry on collision
    — over a counter: no coordination, unguessable; collisions ~0 at this scale (birthday math)
 b) Caching: immutable bodies → CDN cache forever; metadata TTL 5 min; expiry checked at read
    — expired = 410 even if cached (store expires_at in cache entry)
 c) View counts: batch increments per key per 10 s → avoids hot-row contention on viral pastes

7. FAILURES, BOTTLENECKS, 10×
 · DB primary down → reads continue from cache/replicas; writes fail fast for ~30 s failover
 · Viral paste → CDN absorbs; counter batching absorbs the write side
 · 10× → still one primary for writes (~400/s); add replicas; shard by key prefix past ~5 k writes/s
 · Abuse: rate-limit creates per IP (token bucket); scan content; takedown endpoint
```

The decision this whiteboard defends is **spending the first ten minutes on requirements and estimates before drawing
anything**, which feels risky with an interviewer waiting and a clock running. But nearly every decision on the board traces
back to those ten minutes: the 100:1 read ratio is why there's a CDN and cache; "never lost once acknowledged" is why bodies
go to durable object storage; "counts may lag minutes" is what permits batched counters; and 40 writes per second at peak is
why the design has one database primary instead of a sharded cluster — the single most important simplification on the
board. A candidate who starts drawing at minute one produces something that looks similar but can't explain why any of it is
there, and explanation is what the interview is measuring.

---

## 34.10 Questions

**1.** MCQ — What do interviewers primarily evaluate in a system design interview? a) The final diagram's correctness b) The process: framing, quantitative reasoning, trade-offs, depth, failure thinking, communication c) Knowledge of specific products d) Coding speed
<details><summary>Answer</summary>**b)** There's no single right design; the reasoning is the signal.</details>

**2.** MCQ — In a 45-minute design interview, which phase should get the largest share of time? a) Requirements b) Estimation c) High-level design d) Deep dives
<details><summary>Answer</summary>**d)** Depth is where seniority shows; budget roughly a third or more of the time for it.</details>

**3.** MCQ — The interviewer asks "what if the cache goes down?" mid deep dive. What's the best response? a) Say it won't b) Explain the failure behaviour and mitigations (fall back to DB with load protection, warm-up, stale serving), then return to the thread c) Change the whole design d) Ask to move on
<details><summary>Answer</summary>**b)** "What if" questions are invitations to show failure thinking.</details>

**4.** MCQ — Which estimate is most useful? a) "11,574.07 requests per second" b) "~12 k reads/s at peak, so reads must come from a cache" c) "A lot of traffic" d) "It depends"
<details><summary>Answer</summary>**b)** A rounded number tied to a design decision.</details>

**5.** MCQ — Which answer best demonstrates depth? a) "We'll use Kafka" b) "Events are keyed by order ID so per-order order is preserved; consumers commit offsets after the database transaction, with an inbox for duplicates" c) "We'll make it scalable" d) "We'll use microservices"
<details><summary>Answer</summary>**b)** Mechanism, ordering, and delivery semantics — not a product name.</details>

**6.** MCQ — What distinguishes staff-level expectations from senior? a) More boxes b) Framing in business terms, strategic trade-offs (build vs buy, cost), evolution, and knowing where not to add complexity c) Faster drawing d) Knowing more products
<details><summary>Answer</summary>**b)** Breadth of judgement beyond the technical design itself.</details>

**7.** Short — Why state what's out of scope during requirements?
<details><summary>Answer</summary>It bounds the problem so the design can go deep on what matters, shows prioritisation, and invites the interviewer to correct you early if they wanted something you cut.</details>

**8.** Short — What's the pattern for a deep dive?
<details><summary>Answer</summary>State the problem precisely, give two or three options with numbers, choose with a reason and a reversal condition, and explain the failure behaviour of the choice.</details>

**9.** Short — Why propose the deep-dive topics yourself at the end of the high-level design?
<details><summary>Answer</summary>It shows you know where the real difficulty lies, lets you steer toward strengths, and gives the interviewer a natural point to redirect you to what they want to probe.</details>

**10.** Short — What's wrong with hedging every decision?
<details><summary>Answer</summary>It reads as indecision and avoids the trade-off reasoning being evaluated. Commit to one option, explain why, and say when the other would win.</details>

**11.** Short — What three things make a strong wrap-up?
<details><summary>Answer</summary>What breaks first as load grows (and the fix), what happens when each stateful component fails, and what you'd address with more time (observability, abuse, multi-region, cost).</details>

**12.** Output — Using §34.4's `quickEstimate`, what is the peak write rate for `{ dau: 50e6, writesPerUser: 0.2, readsPerUser: 30, bytesPerWrite: 500 }`?
<details><summary>Answer</summary>**~300/s.** 50 M × 0.2 = 10 M writes/day; ÷ 10⁵ = 100/s average; × 3 = 300/s peak.</details>

**13.** Output — Same inputs: what are the average read rate, the read:write ratio, and the 5-year raw storage?
<details><summary>Answer</summary>**~15,000 reads/s, 150:1, ~9 TB.** 50 M × 30 / 10⁵ = 15,000; 15,000 / 100 = 150; 10 M × 500 B × 365 × 5 ≈ 9.1 TB.</details>

**14.** Output — In §34.9, keys are 7 random base62 characters. How many possible keys are there, to two significant figures?
<details><summary>Answer</summary>**~3.5 trillion.** 62⁷ ≈ 3.52 × 10¹².</details>

**15.** Output — You've spent 22 minutes of a 45-minute interview and are just finishing the high-level design. If you reserve 5 minutes for wrap-up, how much time remains for deep dives, and what fraction of the interview is that?
<details><summary>Answer</summary>**18 minutes, 40%.** 45 − 22 − 5 = 18, which is still a healthy deep-dive budget — but only if you move on now.</details>

**16.** Debug — A candidate's feedback said "strong knowledge, but couldn't demonstrate depth"; they spent 30 minutes drawing a detailed high-level diagram with 20 components. What went wrong, and what should they change?
<details><summary>Answer</summary>Time went to breadth: many unjustified components and no deep dives. Draw the simplest design that works in under ten minutes, name the two or three hard parts, and spend the largest block of time going deep on mechanisms, numbers, and failures.</details>

**17.** Debug — A candidate was told their design "solved the wrong problem": they designed a global, multi-region chat system for what turned out to be an internal tool for 2,000 employees. Where did the process fail?
<details><summary>Answer</summary>Requirements: they didn't ask about users and scale, or didn't propose assumptions for confirmation. Five minutes of scoping would have revealed that a single-region, modest design was appropriate.</details>

**18.** Debug — An interviewer asked twice "how would this work for an account with 50 million followers?" and the candidate repeated their fan-out-on-write explanation both times. What should they have done?
<details><summary>Answer</summary>Recognised a hint pointing at a gap — fan-out on write for 50 M followers is 50 M writes per post — and adapted: a hybrid with pull-based fan-out for very large accounts, with numbers and the threshold rationale.</details>

**19.** Debug — A candidate's estimates were meticulous (four decimal places) but the design never referred to them. Feedback: "estimation didn't inform the design." How should they present estimates?
<details><summary>Answer</summary>Round aggressively, compute only the numbers that decide something (peak writes, read ratio, storage, connections), and immediately state the consequence of each ("600 writes/s → one primary is enough").</details>

**20.** Code — Write `budget(totalMinutes)` that returns phase durations scaled from the 45-minute plan, keeping requirements and wrap-up at their minimums of 4 and 4 minutes.
<details><summary>Answer</summary>

```js
function budget(total) {
  const base = { requirements: 5, estimation: 4, apiAndData: 6, highLevel: 8, deepDives: 17, wrapUp: 5 };
  const fixed = { requirements: Math.max(4, Math.round(5 * total / 45)), wrapUp: Math.max(4, Math.round(5 * total / 45)) };
  const flexible = ['estimation', 'apiAndData', 'highLevel', 'deepDives'];
  const flexBase = flexible.reduce((s, k) => s + base[k], 0);
  const left = total - fixed.requirements - fixed.wrapUp;
  const out = { ...fixed };
  for (const k of flexible) out[k] = Math.round((base[k] / flexBase) * left);
  return out;
}
console.log(JSON.stringify(budget(35)));
// {"requirements":4,"wrapUp":4,"estimation":3,"apiAndData":5,"highLevel":6,"deepDives":13}
```
</details>

**21.** Code — Write `tradeoff({ choice, over, because, unless })` that formats a decision as "X over Y, because Z, unless W", throwing if any part is missing.
<details><summary>Answer</summary>

```js
function tradeoff({ choice, over, because, unless }) {
  for (const [k, v] of Object.entries({ choice, over, because, unless })) if (!v) throw new Error(`missing ${k}`);
  return `${choice} over ${over}, because ${because}, unless ${unless}.`;
}
console.log(tradeoff({ choice: 'Fan-out on write', over: 'fan-out on read', because: 'reads outnumber posts 100:1',
  unless: 'the author has more than ~100k followers' }));
// Fan-out on write over fan-out on read, because reads outnumber posts 100:1, unless the author has more than ~100k followers.
```
</details>

**22.** Code — Write `firstToBreak(components, load)` where each component is `{ name, capacity }` in requests/s and `load` maps names to expected requests/s; return components sorted by utilisation, highest first.
<details><summary>Answer</summary>

```js
function firstToBreak(components, load) {
  return components
    .map((c) => ({ name: c.name, utilisation: (load[c.name] ?? 0) / c.capacity }))
    .sort((a, b) => b.utilisation - a.utilisation)
    .map((c) => `${c.name} ${(c.utilisation * 100).toFixed(0)}%`);
}
console.log(firstToBreak([{ name: 'db-primary', capacity: 5_000 }, { name: 'cache', capacity: 200_000 }, { name: 'api', capacity: 40_000 }],
  { 'db-primary': 4_000, cache: 60_000, api: 30_000 })); // [ 'db-primary 80%', 'api 75%', 'cache 30%' ]
```
This is the "what breaks at 10×" question as arithmetic: multiply the load by 10 and the order tells you where to look first.</details>

**23.** Scenario — Ten minutes into an interview, the interviewer says "let's skip the high-level design; tell me how you'd store and query the location updates." How do you adapt?
<details><summary>Answer</summary>Follow the redirect: briefly restate the relevant requirements and estimate (update rate, query shape, latency), then go straight into the storage deep dive — options (geohash/S2 cells in a key-value store, in-memory spatial index, PostGIS), numbers, the choice with its trade-off, and failure behaviour. Sketch only the minimal surrounding design needed to make the storage choice make sense. Interviewers redirect to probe depth; arguing for your planned order wastes the opportunity.</details>

**24.** Scenario — You realise 30 minutes in that an early decision (strong consistency for likes) is wrong and is making the rest of the design awkward. What do you do?
<details><summary>Answer</summary>Say so and fix it: "I want to revisit the consistency choice for likes — given the hot-row contention we've just discussed, eventual consistency with sharded counters fits better." Update the diagram and continue. Recognising and correcting a mistake is a strong signal; defending a wrong decision to avoid admitting it is a weak one.</details>

---

## Chapter Summary

- Design interviews evaluate process — framing, quantitative reasoning, justified components, depth, trade-offs, failure
  thinking, and communication — with expectations that rise from sound designs (mid) to strategic judgement (staff).
- Run six phases with a time budget — requirements, estimation, API and data model, high-level design, deep dives, and
  wrap-up — and give deep dives the largest share.
- Scope ruthlessly: a few architecture-shaping use cases, quantified non-functional requirements, explicit exclusions, and
  stated assumptions.
- Estimate only what decides the design, round aggressively, and say what each number means.
- Draw the simplest working design first, propose the hard parts as deep dives, and go into mechanisms, numbers, choices
  with reversal conditions, and failure behaviour.
- Close with what breaks first, what happens when each stateful component fails, and what you'd do next — and throughout,
  think aloud, check in, take hints, and commit.

---

## Practice Exercises

1. Pick three prompts (a URL shortener, a chat app, a ride-hailing service) and write only the requirements and estimation
   phases for each in under ten minutes, timed.
2. Record yourself doing a full 45-minute design out loud on a whiteboard or drawing tool, then review the recording for
   time spent per phase and unjustified components.
3. Read two published interview write-ups or mock-interview transcripts, and identify where each candidate demonstrated or
   missed depth.
4. For each case study in Chapters 35–48, write the "X over Y, because Z, unless W" statement for its most important
   decision.
5. Do a mock interview with a colleague playing interviewer who is instructed to redirect you twice and ask three "what if"
   questions; practise adapting without losing structure.

---

## Mini Project: A Design-Interview Practice Kit

Build a practice kit you (and others) can use to rehearse the method in this chapter.

- A prompt bank of at least 15 design problems, each with suggested scoping questions and the numbers that should drive
  the design.
- A timer that announces phase boundaries for 35-, 45-, and 60-minute formats.
- A one-page whiteboard template with the six phases (like §34.9's layout).
- A scoring rubric for a partner, covering each signal in §34.1 with concrete "strong" and "weak" descriptors.
- Three completed whiteboards of your own, each followed by a self-review against the rubric.

---

**Next:** [Chapter 35 — Design a URL Shortener →](35-url-shortener.md)
