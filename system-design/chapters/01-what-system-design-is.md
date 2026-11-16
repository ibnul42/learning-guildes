# Chapter 1 — What System Design Is

## Learning Objectives

- Turn a vague product request into functional requirements and *quantified* non-functional ones
- Name the trade-off behind a design choice and the condition under which the other option wins
- Distinguish vertical from horizontal scaling and explain why state is what blocks the second
- Run the design loop — simplest working design, find the bottleneck, change one thing, repeat
- Write a one-page design document that a reviewer can disagree with productively

---

## 1.1 Design Is Choosing Under Constraints

System design is the work of deciding how the parts of a software system are arranged — what
runs where, what stores what, what talks to what, and what happens when any of it breaks — so
that the system meets its requirements at an acceptable cost. It is not drawing boxes. The boxes
are the *output*; the work is the reasoning that chose those boxes over others.

The fastest way to see this is to hold the feature constant and change the constraints. Take
"users can upload a profile photo and others can see it":

| Constraint | Reasonable design |
|---|---|
| Internal tool, 200 users | Store the file in a `bytea` column next to the user row. One server, one database, done in an afternoon. |
| Consumer app, 2 M users, photos viewed 100× more than uploaded | Upload straight to object storage with a pre-signed URL; store only the key in the database; serve through a CDN; resize asynchronously. |
| 500 M users across continents, regulatory data residency | All of the above, plus per-region buckets, a residency-aware upload router, content moderation in the pipeline, and a cache hierarchy tuned so 99% of reads never leave the user's region. |

Every one of those is *correct*. The first design is not a worse version of the third — it is
the right design for its constraints, and the third would be a disaster there: weeks of work,
five moving parts to operate, and a monthly bill for infrastructure nobody needs. The skill this
book teaches is matching the design to the constraints, and being able to say *which constraint
would have to change* before you'd move to the next row.

> 💼 **Interview** — "Design Instagram" is not asking for the most impressive architecture you
> can draw. Interviewers are watching whether you ask *how many users, what read/write ratio,
> what latency, what consistency* before drawing anything — and whether your design changes when
> the answers do.

A useful definition, then: **a design is a set of decisions, each justified by a requirement and
each paid for with a trade-off.** If you can't name the requirement a component serves, it
shouldn't be there. If you can't name what it costs, you haven't understood it yet.

---

## 1.2 Requirements: Functional and Non-Functional

**Functional requirements** say what the system does: "a user can post a message to a channel",
"a message appears for everyone in the channel", "users can search message history". They are
the features. They are usually the easy part, and they are usually *too many* — the first job is
to cut them down to the three to five that shape the architecture.

**Non-functional requirements (NFRs)** say how well it must do it. They are what actually decide
the architecture, and they are worthless until they have numbers:

| NFR | Vague (useless) | Quantified (designable) |
|---|---|---|
| Latency | "fast" | p99 < 200 ms for message send, measured at the API edge |
| Availability | "always up" | 99.95% of minutes with successful sends, per month |
| Durability | "never lose data" | an acknowledged message survives the loss of any one data centre |
| Consistency | "accurate" | a sender sees their own message immediately; others within 2 s |
| Throughput | "scalable" | 50 k sends/s peak, 5 M concurrent connections |
| Scale of data | "lots" | 10 TB/year of messages, retained 7 years |
| Cost | "cheap" | infrastructure < $0.02 per monthly active user |

Notice that each quantified version contains a *measurement point* (at the API edge), a
*statistic* (p99, not average), and a *window* (per month). Without those three, two engineers
can both claim the system meets "p99 < 200 ms" while measuring entirely different things. Chapter
3 explains why the statistic matters; Chapter 4 explains how to turn availability targets into
error budgets.

> ⚠️ **Gotcha** — "Never lose data" and "always available" are not both achievable during a
> network partition (Chapter 20). If both appear in a requirements list unqualified, the
> requirements are not yet finished — someone has to decide which one gives way, and that
> decision belongs to the product owner, not the database.

### Eliciting requirements

The questions that do the most work, in roughly the order to ask them:

1. **Who are the users and how many?** Daily active users (DAU), and how traffic is shaped over a
   day — a B2B tool peaks at 10:00 on weekdays; a streaming service peaks at 21:00 on Sundays.
2. **What is the read/write ratio?** A 100:1 read-heavy system (a news site) and a 1:1 system (a
   chat app) want fundamentally different data paths.
3. **What must never happen?** Double-charging a card, showing a user someone else's data,
   overselling a seat. These become invariants the design must protect even under failure.
4. **What can be stale, and for how long?** Like counts can lag by minutes. Account balances
   cannot. The answer determines where you're allowed to cache and replicate asynchronously.
5. **What is explicitly out of scope?** Writing it down prevents scope creep and makes the
   design reviewable — "we don't support message editing in v1" removes an entire consistency
   problem.

### Constraints are requirements too

Teams forget the requirements nobody wrote down: the company runs on one cloud provider; the
team is four engineers and none have operated Kafka; there is an existing PostgreSQL cluster with
a DBA; the launch is in six weeks. These constraints rule out designs as decisively as a latency
target, and a design that ignores them is wrong no matter how elegant it is.

---

## 1.3 The Trade-offs That Recur

Almost every design decision in this book is an instance of one of a small number of trade-offs.
Learning to recognise them is most of the skill: once you see that "should the like count be
exact?" is the consistency/latency trade-off, you already know the menu of answers.

| Trade-off | You gain | You pay | The other side wins when… |
|---|---|---|---|
| **Consistency vs availability** (Ch 20) | Every read sees the latest write | Some requests fail during partitions | Stale data is harmless and downtime is expensive |
| **Consistency vs latency** (Ch 20) | No anomalies | Cross-replica coordination on the hot path | Users never compare notes (a like count) |
| **Latency vs throughput** (Ch 3) | Each request returns sooner | Less batching, less efficient hardware use | The work is offline or bulk |
| **Read vs write optimisation** (Ch 10, 12) | Fast reads via precomputation, indexes, denormalisation | Every write does more work | Writes dominate or data changes constantly |
| **Durability vs latency** (Ch 13) | An ack means the data is on several machines | Waiting for remote acknowledgements | Losing the last second of data is acceptable |
| **Simplicity vs flexibility** (Ch 27) | Fewer moving parts, easier debugging | Harder to scale parts independently | Teams or load profiles genuinely diverge |
| **Cost vs redundancy** (Ch 4, 32) | Survives more failures | Paying for idle capacity | The cost of downtime is lower than the cost of standby |
| **Build vs buy** | Exactly what you need | Engineering time, operational load | The managed service fits and the bill is acceptable |

Two habits make these trade-offs explicit rather than accidental.

**State the decision as "X over Y, because Z."** "We chose asynchronous replication over
synchronous, because a 30 ms write-latency budget can't absorb a cross-region round trip, and
losing up to one second of writes on regional failure is acceptable per the product owner." That
sentence can be reviewed. "We use async replication" cannot.

**Name the reversal condition.** Every trade-off has a point at which you'd choose differently.
"If regulatory requirements make the one-second loss window unacceptable, we move to synchronous
replication within a region and accept the latency." Writing the reversal condition down is what
turns a design into something that can evolve rather than something that must be rewritten.

> 💼 **Interview** — The single most reliable senior signal in a design interview is volunteering
> the trade-off before being asked: "I'm going to fan out on write here, which makes reads cheap
> but makes posting expensive for accounts with millions of followers — I'll come back to that."
> It shows you know the design has a weak point and where it is.

---

## 1.4 Scaling: Up, Out, and the Problem of State

When one machine can't keep up, there are two ways to get more capacity.

**Vertical scaling (scaling up)** means a bigger machine — more cores, more RAM, faster disks.
It requires no code changes, which makes it chronically underrated. A single modern server with
64+ cores, 512 GB of RAM, and NVMe storage can run a well-indexed PostgreSQL database serving tens
of thousands of simple queries per second. Many companies that think they need sharding actually
need a bigger instance and better indexes.

**Horizontal scaling (scaling out)** means more machines sharing the work. It has no hard
ceiling and it gives you redundancy for free — two machines can survive one failing. But it only
works if the work can actually be split, and the thing that prevents splitting is **state**.

| | Vertical | Horizontal |
|---|---|---|
| Code changes | None | Must be stateless, or state must be partitioned |
| Ceiling | The largest machine you can buy | Effectively none for stateless tiers |
| Failure | One machine = single point of failure | Survives individual node loss |
| Cost curve | Superlinear at the top end | Roughly linear |
| Operational complexity | Low | Load balancing, deployment, coordination |
| Good first move for | Databases, anything stateful | Stateless application servers |

A server is **stateless** when any instance can handle any request, because nothing a request
needs lives only in that instance's memory or disk. That's the property that lets a load balancer
send each request anywhere. The classic violation is session state kept in process memory:

```js
// ❌ Wrong — session lives in one process's memory. With two instances behind a load balancer,
// a user who logs in on instance A is "logged out" whenever a request lands on instance B.
const sessions = new Map();

export function login(userId) {
  const sid = crypto.randomUUID();
  sessions.set(sid, { userId, createdAt: Date.now() });
  return sid;
}

export function whoAmI(sid) {
  return sessions.get(sid)?.userId ?? null; // null on every instance except the one that logged in
}
```

```js
// ✅ Right — the instance is stateless; state lives in a store every instance shares.
// The store is injected, so tests use an in-memory fake and production uses Redis or a database.
export function createSessionService(store) {
  return {
    async login(userId) {
      const sid = crypto.randomUUID();
      // The TTL lives with the data, so expiry works no matter which instance wrote it
      await store.set(`session:${sid}`, JSON.stringify({ userId }), { ttlSeconds: 86_400 });
      return sid;
    },
    async whoAmI(sid) {
      const raw = await store.get(`session:${sid}`);
      return raw ? JSON.parse(raw).userId : null;
    },
  };
}

// A minimal in-memory store with the same interface, for tests and for running this file
const memoryStore = () => {
  const m = new Map();
  return {
    async set(k, v) { m.set(k, v); },
    async get(k) { return m.get(k) ?? null; },
  };
};

const shared = memoryStore();               // one shared store…
const instanceA = createSessionService(shared);
const instanceB = createSessionService(shared); // …two stateless "instances"
const sid = await instanceA.login('u42');
console.log(await instanceB.whoAmI(sid)); // u42
```

Moving state out of the application tier doesn't make it disappear — it concentrates it in the
data tier, which now has to scale. That's the recurring shape of this book: **stateless tiers
scale out easily; all the hard problems live where the state went.** Part III is about scaling
that state.

> ⚠️ **Gotcha** — "Sticky sessions" (the load balancer pins a user to one instance) look like a
> fix for in-memory state. They aren't: when that instance is deployed, restarted, or scaled in,
> every user pinned to it loses their session at once, and load stays uneven because a few heavy
> users can't be rebalanced. Chapter 7 covers when stickiness is legitimate (WebSocket
> connections, local caches) and when it's hiding a bug.

### Three axes of scaling out

A helpful map of *how* to split work, often called the scale cube:

| Axis | What you split | Example | Limits |
|---|---|---|---|
| **X — clone** | Nothing; run N identical copies | 20 stateless API servers behind a load balancer | Shared state underneath becomes the bottleneck |
| **Y — functional** | By function or service | Separate search, checkout, and catalogue services | Cross-service calls and transactions (Ch 23, 27) |
| **Z — data** | By a key in the data | Users A–M on shard 1, N–Z on shard 2 | Queries spanning shards; hot keys (Ch 14) |

Real systems use all three, usually in that order: clone first because it's nearly free, split
by data when one database can't hold the write load, split by function when teams or load
profiles diverge.

---

## 1.5 The Design Loop

Good designs are rarely produced in one pass. They come out of a loop:

```text
   ┌──────────────────────────────────────────────────────────┐
   │                                                          │
   ▼                                                          │
 simplest design that ──▶ estimate load ──▶ find the first ──▶ change the one
 meets functional reqs     against it       thing that breaks   thing that breaks
```

The discipline is in the last step: change *one* thing, the thing that breaks first, and then
re-estimate. Adding a cache, a queue, a read replica, and sharding all at once produces a design
nobody can reason about, with components that solve problems the system doesn't have.

Here is the loop run on a simple service — a product catalogue with 10 M products, 1 M daily
users, each viewing about 30 product pages per day.

**Step 0 — one box.** App and database on one server.

```text
client ──▶ [ app + PostgreSQL ]
```

Average load is 1 M × 30 / 86,400 ≈ 350 reads/s; with a 3× peak factor (Chapter 2), ~1,000
reads/s. A single PostgreSQL instance serving indexed primary-key lookups handles that
comfortably. **Nothing breaks yet** — and that's an answer worth giving. Many systems should stop
here.

**Step 1 — the first failure mode is not load, it's availability.** One server means a kernel
patch is an outage. Separate the tiers and run two app instances behind a load balancer.

```text
client ──▶ load balancer ──▶ app ×2 ──▶ PostgreSQL (primary + standby)
```

**Step 2 — growth to 30 M daily users.** Peak reads are now ~30,000/s. The app tier scales out
trivially (it's stateless). The database is now the bottleneck. Two candidate fixes:

| Fix | Effect | Cost |
|---|---|---|
| Read replicas | Divides read load across N copies | Replication lag — a just-edited product may show stale data (Ch 13) |
| Cache in front of the DB | Absorbs the hot set; product reads are highly skewed | Invalidation on edit; cold-start behaviour (Ch 16) |

Product views follow a heavy-tailed distribution — a small fraction of products gets most views —
so a cache with a 95% hit rate turns 30,000 reads/s into 1,500 at the database. Choose the cache;
keep replicas in reserve for when the cache isn't enough.

```text
client ──▶ CDN ──▶ load balancer ──▶ app ×N ──▶ cache ──▶ PostgreSQL (primary + standby)
```

**Step 3 — the next bottleneck.** Suppose the business adds real-time inventory: 5,000 writes/s
of stock updates. Now writes, not reads, threaten the primary — and that's the point at which
partitioning (Chapter 14) or moving inventory to its own store (Chapter 12) enters the
conversation. Not before.

Notice what the loop *didn't* add: a message queue, microservices, a search cluster, Kubernetes.
Each might eventually be right. None was justified by a requirement at any step, so none is in
the design.

> 💼 **Interview** — Interviewers often say "now traffic grows 10×" to see whether you can run
> this loop out loud. The strong answer names the *first* component to break and the cheapest
> fix, rather than redrawing everything.

---

## 1.6 Writing It Down: The Design Document

A design that exists only in someone's head can't be reviewed, and a design that can't be
reviewed is probably wrong in a way nobody has noticed yet. The design document is the artefact
of the reasoning in §1.2–1.5. Its purpose is not ceremony; it's to make disagreement cheap —
cheaper on paper than in production.

The sections that earn their space:

| Section | Purpose | Common failure |
|---|---|---|
| Context and problem | Why this work exists now | Describes the solution instead of the problem |
| Goals and **non-goals** | Bounds the scope | Non-goals missing, so reviewers argue about things nobody intended to build |
| Requirements with numbers | What "done" means | Unquantified NFRs (§1.2) |
| Proposed design | The diagram and the data flow | All components, no *why* |
| **Alternatives considered** | Shows the decision was a decision | "We considered X but chose Y" with no reason |
| Failure modes | What breaks and what happens then | Only the happy path |
| Rollout and rollback | How it ships without an outage | Assumed to be someone else's problem (Ch 31) |
| Open questions | What's not yet decided | Missing, so open questions get decided by default |

Keep it short. A design document for a single service should fit in two to five pages; if it's
longer, it's probably two designs. For decisions that outlive the document — "we standardise on
PostgreSQL", "services communicate via events, not shared databases" — record an **Architecture
Decision Record (ADR)**: a half-page note with context, decision, and consequences, numbered and
never edited after acceptance (superseded by a new ADR instead). Six months later, the ADR answers
"why on earth did we do it this way?" in a minute rather than an archaeology session.

---

## 1.7 Common Mistakes and Best Practices

**Mistake: designing before asking.** Drawing the architecture before the read/write ratio,
scale, and consistency needs are known produces a design for imagined requirements.

**Mistake: unquantified non-functional requirements.** "Highly available" and "low latency" can't
be designed against or tested; every NFR needs a number, a statistic, and a measurement point.

**Mistake: résumé-driven architecture.** Adding Kafka, Kubernetes, or microservices because
they're impressive rather than because a requirement demands them multiplies operational cost
with no return.

**Mistake: solving the 100× problem on day one.** Designing for scale you won't see for years
costs you the speed you need to reach it; design for ~10× current load and know what changes
after that.

**Mistake: hidden state in a "stateless" tier.** In-memory sessions, local file uploads, and
per-instance caches that must be consistent all silently break horizontal scaling.

**Mistake: trade-offs left implicit.** A design that doesn't say what it gave up will be
"fixed" by the next engineer in a way that breaks the thing it was protecting.

**Mistake: happy-path-only designs.** If the document never says what happens when the database
is down, the answer in production will be "everything stops".

**Best practice: start with the simplest design that works, then let the bottleneck lead.** Each
added component should trace back to a specific load estimate or requirement it fixes.

**Best practice: write decisions as "X over Y, because Z, unless W".** It makes the trade-off
reviewable and records the condition for revisiting it.

**Best practice: prefer boring technology where it's sufficient.** A well-understood database
your team can operate at 3 a.m. beats a better-fit one nobody has run in production.

---

## 1.8 Real-World Example: A One-Page Design Document

This is a compressed but complete design document for a real-sized feature — the kind a senior
engineer writes before a week of implementation. Read it as a template: each section is doing a
job described in §1.6.

```markdown
# Design: Product Image Thumbnails (v1)

Author: catalogue team · Status: in review · Reviewers: storage, SRE

## Context
Product pages load full-size seller images (avg 2.4 MB). Mobile p75 page load is 4.1 s;
images are 78% of bytes. Sellers upload ~40k images/day; pages serve ~25M image views/day.

## Goals
- Serve 3 sizes (160, 480, 1080 px wide, WebP + JPEG fallback) for every product image.
- p95 image bytes per product page < 300 KB on mobile.
- A newly uploaded image has thumbnails within 60 s (p99).

## Non-goals
- Arbitrary on-the-fly sizes (fixed set only; revisit if design asks for more than 5 sizes).
- Re-processing the existing 30M images in this project (separate backfill, see Rollout).
- Image moderation (owned by trust & safety; we emit an event they already consume).

## Requirements (numbers)
- Ingest: 40k uploads/day ≈ 0.5/s avg, 5/s peak (seller bulk imports).
- Read: 25M views/day ≈ 290/s avg, ~1k/s peak — served by CDN, origin sees < 5%.
- Storage: 40k × 3 sizes × 2 formats × ~60 KB ≈ 14 GB/day ≈ 5 TB/year.

## Proposed design
upload ─▶ object store (originals) ─▶ "image.uploaded" event ─▶ resize workers ×N
                                                                    │
           CDN ◀── object store (derivatives, immutable keys) ◀──────┘
Derivative keys embed a content hash: /img/{hash}/{size}.{fmt}, cached forever at the CDN.
Product records store the hash; the page renders URLs for all sizes via <picture>/srcset.
Workers are idempotent: output key is a pure function of input hash + size + format.

## Alternatives considered
- Resize on request at the CDN edge: no pipeline to run, but cold-miss latency ~800 ms
  and per-request compute cost at 25M views/day exceeds the worker cost ~6×. Rejected.
- Resize synchronously in the upload request: simplest, but bulk imports of 5k images would
  hold connections for minutes and fail partially. Rejected.

## Failure modes
- Worker backlog: page falls back to the original URL if a derivative 404s; alert if queue
  age > 5 min. Uploads never fail because of resize problems.
- Poison image (crashes decoder): 3 attempts, then dead-letter + metric; seller sees original.
- CDN outage: origin can't absorb 1k/s of full traffic; accepted, CDN SLA 99.9%.

## Rollout
Dark-launch workers on 1% of uploads → compare output → 100% → backfill top 1M images by
traffic → flip page rendering behind a flag, per country. Rollback = flag off.

## Open questions
- Do we need AVIF in v1? (Data from design: +12% savings, 2× encode CPU.)
```

The decision a reader is most likely to question is **content-hash keys with infinite CDN
caching** rather than keys based on the product ID with a TTL. Product-ID keys look simpler, but
they make every image update a cache-invalidation problem: you must purge the CDN, and until the
purge propagates some users see the old image and some see the new one. With hash-based keys an
updated image *is a different URL*, so nothing ever needs to be invalidated, caches can be told
"this never changes", and the worker's idempotency falls out for free — the same input always
writes the same key. Chapter 6 generalises this as "immutable URLs for mutable content", and it's
one of the highest-leverage patterns in the book.

---

## 1.9 Questions

**1.** MCQ — Which of these is a usable non-functional requirement? a) "The API should be fast." b) "The API should be highly available." c) "p99 latency for `GET /orders/:id` under 150 ms, measured at the load balancer, over each 5-minute window." d) "The API should scale to millions of users."
<details><summary>Answer</summary>**c)** It has a statistic (p99), a threshold, a measurement point, and a window. The others can't be tested or designed against.</details>

**2.** MCQ — What most directly prevents an application tier from scaling horizontally? a) Using a relational database b) State held only in a single instance's memory c) Running on virtual machines rather than containers d) Using HTTP instead of gRPC
<details><summary>Answer</summary>**b)** If a request depends on memory in one specific instance, the load balancer can't send it anywhere else. The other options don't affect the tier's ability to clone.</details>

**3.** MCQ — A team's single PostgreSQL server is at 85% CPU serving reads. Their query plans show sequential scans on the two most frequent queries. What is the best first move? a) Shard the database b) Add read replicas c) Add the missing indexes and re-measure d) Move to a NoSQL store
<details><summary>Answer</summary>**c)** The bottleneck is wasted work, not capacity. Indexes may cut CPU by an order of magnitude at zero operational cost; scaling the waste out is strictly worse.</details>

**4.** MCQ — In the scale cube, splitting users across databases by user ID is which axis? a) X b) Y c) Z d) None — it's vertical scaling
<details><summary>Answer</summary>**c)** Z-axis is splitting by a key in the data (partitioning/sharding). X is cloning identical copies; Y is splitting by function.</details>

**5.** MCQ — Which section of a design document most directly prevents reviewers from arguing about features nobody planned to build? a) Context b) Non-goals c) Failure modes d) Rollout
<details><summary>Answer</summary>**b)** Non-goals bound the scope explicitly, so review effort goes to what is actually proposed.</details>

**6.** MCQ — Which statement about vertical scaling is accurate? a) It always requires rewriting the application b) It has no upper limit c) It's often the right first step for a stateful component like a database d) It improves availability by removing single points of failure
<details><summary>Answer</summary>**c)** A bigger machine needs no code changes and is the cheapest fix for a stateful tier until the largest instance is reached. It doesn't remove the single point of failure, and it has a hard ceiling.</details>

**7.** Short — Why is "the system must never lose data and must always be available" an unfinished requirement?
<details><summary>Answer</summary>During a network partition a system must either refuse some operations (losing availability) or accept writes it can't safely replicate (risking loss or conflict). Someone has to decide which property yields, and under what circumstances.</details>

**8.** Short — What three elements turn a latency target into something testable?
<details><summary>Answer</summary>A statistic (e.g. p99, not average), a measurement point (e.g. at the API gateway), and a time window (e.g. per 5 minutes or per month), alongside the threshold itself.</details>

**9.** Short — Why do sticky sessions not really solve the problem of in-memory session state?
<details><summary>Answer</summary>All users pinned to an instance lose their sessions when it restarts, deploys, or scales in, and load can't be rebalanced away from instances holding heavy users. The state is still trapped in one place; stickiness only hides it until that place goes away.</details>

**10.** Short — What is an ADR, and why is it never edited after acceptance?
<details><summary>Answer</summary>An Architecture Decision Record captures one decision's context, choice, and consequences. It's immutable so the history of reasoning stays intact; a changed decision gets a new ADR that supersedes the old one, preserving why each choice was made at the time.</details>

**11.** Short — In the design loop, why change only one thing per iteration?
<details><summary>Answer</summary>Each change shifts where the next bottleneck is. Changing one thing lets you re-estimate and confirm it fixed the problem; changing several at once adds components whose necessity was never established and makes the design impossible to reason about.</details>

**12.** Output — A site has 2 M daily active users, each making 50 requests per day. Ignoring peaks, what is the average request rate, to the nearest hundred per second?
<details><summary>Answer</summary>**~1,200 requests/s.** 2,000,000 × 50 = 100 M requests/day; 100,000,000 / 86,400 ≈ 1,157.</details>

**13.** Output — What does this print?
```js
const instances = [new Map(), new Map()];
const route = (n) => instances[n % 2];
route(0).set('sid1', 'u1');      // login handled by request #0
console.log(route(1).get('sid1') ?? 'logged out', route(2).get('sid1') ?? 'logged out');
```
<details><summary>Answer</summary>`logged out u1` — request #1 goes to the other instance's map; request #2 goes back to the first. Round-robin with per-instance state produces exactly this flapping, which users experience as random logouts.</details>

**14.** Output — A cache in front of a database receives 20,000 reads/s with a 95% hit rate. How many reads per second reach the database?
<details><summary>Answer</summary>**1,000/s.** Misses are 5% of 20,000. Note that dropping to a 90% hit rate doubles database load to 2,000/s — hit rate matters far more near the top than it looks.</details>

**15.** Output — An API server handles 800 requests/s at the target utilisation. Peak traffic is 12,000 requests/s and you want to survive losing 2 servers at peak. What is the minimum number of servers?
<details><summary>Answer</summary>**17.** 12,000 / 800 = 15 servers needed at peak; add 2 for the failure tolerance (N+2).</details>

**16.** Debug — After a team scaled their web tier from 1 to 4 instances, users report being logged out "randomly, a few times an hour". Nothing changed in the auth code. Cause and fix?
<details><summary>Answer</summary>Sessions are stored in process memory, so any request that lands on an instance other than the one that created the session sees no session. Fix: move sessions to a shared store (Redis, the database) or use self-contained signed tokens; don't paper over it with sticky sessions.</details>

**17.** Debug — Users uploaded avatars successfully, but about half the time their avatar shows as a broken image. The team recently added a second app server. Cause and fix?
<details><summary>Answer</summary>Uploads are written to the local disk of whichever instance handled the upload; reads routed to the other instance find no file. Store uploads in shared object storage (Chapter 17) and serve them from there, or via a CDN.</details>

**18.** Debug — A design was approved with "p99 < 300 ms". Six weeks after launch, the SRE team says p99 is 900 ms and the developers say it's 250 ms. Both have dashboards. What happened?
<details><summary>Answer</summary>The requirement didn't specify a measurement point or window. Developers are likely measuring server-side handler time; SRE is measuring at the edge, including queueing, network, and TLS — or one is using a daily window that smooths spikes. Fix the requirement to name both, then align the dashboards.</details>

**19.** Debug — A small team's service has a message queue, a cache, three microservices, and a search cluster. Incidents are frequent and each takes hours to diagnose; total traffic is 40 requests/s. What's the underlying problem, and what would you do?
<details><summary>Answer</summary>Components were added without a requirement demanding them, multiplying failure modes and diagnosis paths for load a single service and database could serve. Re-derive the requirements, then remove components that don't trace back to one — starting with the ones that cause the most incidents.</details>

**20.** Code — Write `serversNeeded({ peakRps, perServerRps, targetUtilisation, spares })` returning the number of servers to provision, where each server is only allowed to run at `targetUtilisation` of its capacity.
<details><summary>Answer</summary>

```js
function serversNeeded({ peakRps, perServerRps, targetUtilisation = 0.7, spares = 1 }) {
  if (perServerRps <= 0 || targetUtilisation <= 0 || targetUtilisation > 1) {
    throw new RangeError('invalid capacity inputs');
  }
  // Usable capacity per server is less than its maximum, leaving headroom for spikes (Ch 3)
  const usable = perServerRps * targetUtilisation;
  return Math.ceil(peakRps / usable) + spares;
}
console.log(serversNeeded({ peakRps: 12_000, perServerRps: 1_000, targetUtilisation: 0.7, spares: 2 })); // 20
```
</details>

**21.** Code — Write `checkRequirements(nfrs)` that takes an array of `{ name, target }` objects and returns the names whose target contains no digit — i.e. the requirements that aren't quantified yet.
<details><summary>Answer</summary>

```js
function checkRequirements(nfrs) {
  // A digit is a crude but effective proxy: every quantified NFR states a number somewhere
  return nfrs.filter(({ target }) => !/\d/.test(target ?? '')).map(({ name }) => name);
}
console.log(checkRequirements([
  { name: 'latency', target: 'p99 < 200 ms at edge' },
  { name: 'availability', target: 'highly available' },
  { name: 'durability', target: '' },
])); // [ 'availability', 'durability' ]
```
</details>

**22.** Code — Implement a `createSessionService(store)` whose sessions expire after a TTL even if the store has no native TTL support. The store has only `get(key)` and `set(key, value)`.
<details><summary>Answer</summary>

```js
function createSessionService(store, { ttlMs = 86_400_000, now = () => Date.now() } = {}) {
  return {
    async login(userId) {
      const sid = crypto.randomUUID();
      // Store the expiry with the value so any instance can enforce it
      await store.set(`session:${sid}`, JSON.stringify({ userId, exp: now() + ttlMs }));
      return sid;
    },
    async whoAmI(sid) {
      const raw = await store.get(`session:${sid}`);
      if (!raw) return null;
      const { userId, exp } = JSON.parse(raw);
      return exp > now() ? userId : null; // expired sessions are treated as absent
    },
  };
}
```
Expired entries still occupy space; a real store needs a sweeper or native TTL.</details>

**23.** Scenario — A startup with five engineers is building a B2B scheduling product. The CTO wants microservices "so we don't have to rewrite later". Expected load in year one is under 100 requests/s. What do you recommend?
<details><summary>Answer</summary>A modular monolith on one database: enforce module boundaries in code so a later split is possible, but deploy one unit. Microservices at this size buy independent deployment and scaling the team doesn't need, at the cost of network failure modes, distributed transactions, and operational overhead five people can't absorb. Revisit when separate teams own separate domains or one module's load profile genuinely diverges (Chapter 27).</details>

**24.** Scenario — Product wants "real-time" like counts on posts. Engineering proposes a strongly consistent counter updated in the same transaction as each like. Traffic is expected to include viral posts with 50 k likes per minute. How do you push back, and what do you propose?
<details><summary>Answer</summary>Ask what "real-time" means to users: they can't tell a count that lags by two seconds from an exact one, but a single hot row taking 800 writes/s will serialise on its lock and slow every like. Propose recording likes durably (idempotent per user) and maintaining the count asynchronously or via sharded counters (Chapter 12), with "a few seconds stale" as the stated requirement. If the count drives money — say, payouts per like — exactness matters, but it can still be computed in batch rather than on the hot path.</details>

---

## Chapter Summary

- A design is a set of decisions, each justified by a requirement and paid for with a trade-off;
  the same feature has different correct designs at different scales.
- Non-functional requirements decide the architecture, and only quantified ones — statistic,
  threshold, measurement point, window — can be designed against or tested.
- Most decisions are instances of a few recurring trade-offs; write each as "X over Y, because Z,
  unless W" so it can be reviewed and revisited.
- Stateless tiers scale out almost for free; the hard problems move to wherever the state went,
  which is why most of this book is about data.
- Run the design loop: simplest working design, estimate, fix the first bottleneck, repeat — and
  treat "nothing breaks yet" as a legitimate answer.
- A short design document with non-goals, alternatives considered, and failure modes makes
  disagreement cheap before it becomes expensive in production.

---

## Practice Exercises

1. Pick an app you use daily. Write five functional requirements and five quantified NFRs for it,
   each with a statistic, threshold, measurement point, and window.
2. For the same app, write three trade-off statements in the form "X over Y, because Z, unless W"
   that you believe its designers made.
3. Read the public engineering blog post of any large company describing a migration (from a
   monolith, from one database, between clouds). Identify the bottleneck that triggered it and
   whether a simpler fix was considered.
4. Take a stateful toy server (sessions, uploads, or an in-memory counter), run two instances
   behind a simple round-robin proxy, and measure how often requests fail. Then fix it and measure
   again.
5. Run the design loop of §1.5 for a URL shortener from 1,000 to 100 M daily users, stopping at
   each step to name what breaks first and the cheapest fix.

---

## Mini Project: A Design Document Under Review

Write a two-to-four-page design document for a feature you know well — a file-sharing link, a
comment system, a password-reset flow — using the structure in §1.6. Then hand it to someone else
(or set it aside for two days and review it yourself) and record every question the reviewer
asks that the document should have answered.

- Every NFR is quantified with statistic, threshold, measurement point, and window.
- At least two alternatives are considered, each with a specific rejection reason.
- The failure-modes section covers the loss of every stateful dependency.
- Non-goals list at least three things a reasonable reader might assume are in scope.
- A second revision addresses every reviewer question, either in the design or in Open Questions.

---

**Next:** [Chapter 2 — Back-of-the-Envelope Estimation →](02-estimation.md)
