# Chapter 45 — Design a Distributed Job Scheduler

## Learning Objectives

- Scope a multi-tenant job scheduler for one-off, delayed, and recurring jobs, and size its fire rate and top-of-the-hour bursts
- Store timers durably and fire them on time, without scanning everything and without a single point of failure
- Execute jobs at least once with leases, heartbeats, and fencing tokens, and make their effects exactly once with idempotency
- Retry with backoff, dead-letter poison jobs, and handle misfires, time zones, and daylight-saving transitions correctly
- Keep tenants fair and the system stable when millions of jobs are due in the same second

---

## 45.1 Requirements and Scope

A job scheduler runs work at a time: "send this reminder in 24 hours", "generate every customer's invoice on the 1st", "retry this
webhook in 5 minutes", "run the cleanup every night at 02:00". It sits underneath notification systems (Chapter 38), payments
retries (Chapter 43), and almost every product's background work. Its promise sounds simple — *this will run, once, at about this
time* — and every word of it is hard in a distributed system.

**Functional requirements**

- Schedule one-off jobs (run at time *T* or after delay *D*) and recurring jobs (cron expressions with a time zone).
- Run each job's handler with its payload; record the outcome, attempts, and logs.
- Retry failed jobs with backoff; move jobs that keep failing to a dead-letter state for inspection and replay.
- Cancel, pause, and reschedule jobs; query a job's status.

**Out of scope**: workflow orchestration of multi-step jobs with dependencies (mentioned in §45.8), the job handlers themselves.

**Non-functional requirements**

| Requirement | Target |
|---|---|
| Durability | An acknowledged schedule is never lost |
| Execution | Every due job runs **at least once**; duplicate runs are rare and handlers are idempotent |
| Timeliness | p99 of jobs start within 5 seconds of their due time under normal load |
| Scale | ~200 million job executions per day; hundreds of millions of scheduled jobs pending |
| Fairness | One tenant's backlog can't delay other tenants' jobs beyond their share |
| Availability | Scheduling and firing survive the loss of any node or a zone |

"Exactly once" is deliberately absent from the execution row. A scheduler can't know whether a worker that went silent ran the job;
it can only run it again. Exactly-once **effects** are the handler's job, with the scheduler's help (§45.6).

---

## 45.2 Estimation

```js
// Job scheduler: average vs top-of-the-hour load, and what the lease table holds
const jobsPerDay = 200e6, avgRunSec = 4, topOfHourShare = 0.3;   // 30% of hourly jobs are scheduled for :00
const avgPerSec = jobsPerDay / 86_400;
const peakPerSec = (jobsPerDay / 24) * topOfHourShare;           // all due in the same second without jitter
const running = avgPerSec * avgRunSec;                           // Little's law
console.log(`avg ~${Math.round(avgPerSec).toLocaleString('en-US')} jobs/s | :00 burst ~${Math.round(peakPerSec).toLocaleString('en-US')} jobs in one second | ~${Math.round(running).toLocaleString('en-US')} running at once`);
const jitterSec = 300;                                           // spread :00 jobs over 5 minutes
console.log(`with ${jitterSec / 60} min of jitter: ~${Math.round(peakPerSec / jitterSec).toLocaleString('en-US')} extra jobs/s instead of the burst`);
// avg ~2,315 jobs/s | :00 burst ~2,500,000 jobs in one second | ~9,259 running at once
// with 5 min of jitter: ~8,333 extra jobs/s instead of the burst
```

What the numbers decide:

- **The average is easy; the shape is not.** ~2,300 jobs per second is a modest database workload. But humans schedule on round numbers —
  midnight, the top of the hour, the 1st of the month — so millions of jobs can be due in the same second. Absorbing that burst, or
  spreading it (§45.7), is the central capacity problem.
- **~9,000 jobs running at once** means ~9,000 live leases, each renewed by heartbeats: a write load of its own.
- **Hundreds of millions of pending jobs** can't be scanned to find what's due; the store needs an index on due time, partitioned so
  firing scales out (§45.5).
- **Executions per day, kept for 30 days, are billions of rows** of history — a separate, cheaper store from the hot scheduling table.

---

## 45.3 API and Data Model

```text
POST   /v1/jobs                     { handler, payload, runAt | delay | cron+timezone, idempotencyKey, maxAttempts, priority }
GET    /v1/jobs/{id}                → state, nextRunAt, attempts, lastError
DELETE /v1/jobs/{id}                cancel (pending) or request stop (running)
POST   /v1/jobs/{id}/runs/{runId}/heartbeat | complete | fail    (worker protocol, with lease token)
```

`POST /v1/jobs` takes an **idempotency key** (Chapter 8) so a client that retries a timed-out create doesn't schedule the same reminder
twice.

```sql
-- The hot table: pending and running jobs only. Finished runs move to an append-only history store.
CREATE TABLE jobs (
  id            bigint PRIMARY KEY,
  tenant_id     bigint      NOT NULL,
  handler       text        NOT NULL,
  payload       jsonb       NOT NULL,
  run_at        timestamptz NOT NULL,                 -- next due time, in UTC
  state         text        NOT NULL CHECK (state IN ('scheduled', 'running', 'dead', 'paused')),
  attempts      int         NOT NULL DEFAULT 0,
  max_attempts  int         NOT NULL DEFAULT 5,
  lease_until   timestamptz,                         -- set while running; expired = worker presumed dead
  lease_token   bigint,                              -- fencing token, from a sequence, new on every claim
  cron          text,                                -- recurring jobs: expression + IANA time zone
  timezone      text,
  shard         int         NOT NULL                 -- hash(id) % N: which firing partition owns it
);
CREATE INDEX jobs_due ON jobs (shard, run_at) WHERE state = 'scheduled';
CREATE INDEX jobs_lease ON jobs (shard, lease_until) WHERE state = 'running';
```

Two partial indexes do most of the work: one finds due jobs, the other finds expired leases. Neither indexes the billions of finished
runs, which is why those live elsewhere.

---

## 45.4 High-Level Design

```text
clients ─▶ scheduling API ─▶ jobs table (partitioned by shard) ◀─────────────────────────────┐
                                    │                                                        │
                      pollers (one owner per shard, by lease)                                │
                     claim due jobs: SKIP LOCKED, set lease + token                          │
                                    │                                                        │
                                    ▼                                                        │
                  dispatch queues per handler/priority (Chapter 24) ─▶ workers ──heartbeat───┤
                                                                       │                      │
                                                                       └─complete / fail─────┘
                                                                              │
                                                     history store, metrics, dead-letter view
```

- **Scheduling API** validates, computes the first `run_at` (for cron, in the job's time zone), and inserts the job.
- **Pollers** own shards of the jobs table; each repeatedly claims jobs due in its shard and hands them to dispatch. Shard ownership is
  itself a lease (Chapter 21), so a dead poller's shards move to another within seconds.
- **Workers** run handlers, heartbeat to extend the lease on long jobs, and report completion or failure with their lease token.
- **A lease reaper** (part of each poller) finds running jobs whose lease expired and makes them claimable again.

Many real systems collapse the poller and the worker: workers claim directly from the table (the "database as a queue" pattern). That's
a fine design to ~thousands of jobs per second; the separate dispatch tier helps when workers are heterogeneous, numerous, or should
not hold database connections.

---

## 45.5 Deep Dive: Storing and Firing Timers

**Finding what's due.** The core query claims a batch of due jobs for one shard, skipping rows another poller has already locked:

```sql
-- Claim up to 500 due jobs in shard 7. SKIP LOCKED lets several claimers share a shard without blocking each other.
WITH due AS (
  SELECT id FROM jobs
  WHERE shard = 7 AND state = 'scheduled' AND run_at <= now()
  ORDER BY run_at
  LIMIT 500
  FOR UPDATE SKIP LOCKED
)
UPDATE jobs j
SET state = 'running', lease_until = now() + interval '30 seconds', lease_token = nextval('lease_tokens')
FROM due WHERE j.id = due.id
RETURNING j.id, j.handler, j.payload, j.lease_token;
```

The claim and the lease grant are **one transaction**: there's no moment where a job is selected but not yet owned, so two pollers can't
both take it.

**Partitioning the timer space.** One table and one poller cap out; sharding by `hash(id)` spreads both the rows and the polling. A
popular alternative is **time-bucketed storage**: jobs are written to a bucket per minute (`bucket = floor(run_at / 60s)`), and a firer
for the current minute reads its bucket in full. This suits wide-column stores (Chapter 11) with no secondary indexes, and makes
pre-loading the next minute's jobs into memory easy. In memory, the same idea at finer grain is a **timing wheel**: an array of slots
indexed by `time mod wheel size`, giving O(1) insert and fire for timers within the wheel's span.

**Polling vs pushing.** Polling every second adds up to a second of latency and a constant query load; it's simple and robust. Systems
that need sub-second precision load the next few seconds of jobs into an in-memory timing wheel and fire from there, falling back to
the table as the source of truth after a restart.

**Recurring jobs.** Store only the *next* occurrence. When a recurring job runs, the scheduler computes the following `run_at` from the
cron expression and the job's **IANA time zone** — in the same transaction that records the run — and reschedules it. Computing in UTC
is wrong for any schedule a human wrote in local time:

```js
// "Run daily at 02:30 New York time" — what the wall clock does on the spring-forward night
const fmt = new Intl.DateTimeFormat('en-US', { timeZone: 'America/New_York', hour: '2-digit', minute: '2-digit', hourCycle: 'h23', timeZoneName: 'short' });
const start = Date.parse('2026-03-08T06:00:00Z');
const seen = [];
for (let t = start; t <= start + 2 * 3_600_000; t += 30 * 60_000) seen.push(fmt.format(t));
console.log(seen.join(' → '));
// 01:00 EST → 01:30 EST → 03:00 EDT → 03:30 EDT → 04:00 EDT
```

> ⚠️ **Gotcha** — On 8 March 2026 in New York, 02:30 never happens: the clock jumps from 01:59 EST to 03:00 EDT. On 1 November, 01:30
> happens twice. A cron job "at 02:30 daily" must have a written policy for both — commonly, run at the next valid time when the slot is
> skipped, and run once (on the first occurrence) when it repeats. Without one, the job silently doesn't run, or runs twice.

**Misfires.** If the scheduler is down for an hour, what happens to the 60 runs of an every-minute job? The **misfire policy** is per
job: *fire once now and resume* (cleanup, sync), *fire every missed run* (billing per period — usually with the period in the payload),
or *skip to the next* (a "send the morning digest" job that's pointless at noon). The default should be fire-once; "fire all" after a
long outage is a self-inflicted burst.

---

## 45.6 Deep Dive: Leases, Fencing, and At-Least-Once Execution

A claimed job is **leased**, not given away. The lease says "this worker owns the job until `lease_until`". If the worker finishes, it
reports completion; if it crashes, the lease expires and the job becomes claimable again. That's how the scheduler guarantees **at least
once**: a job leaves the running state only through a recorded completion, a recorded failure, or lease expiry back to claimable.

The cost is that a lease can expire while the worker is **alive but slow** — a long GC pause, a network partition, an overloaded host.
Now two workers believe they own the job. This is Chapter 21's lease problem exactly, and the answer is the same:

- **Fencing tokens.** Every claim gets a new, larger token (from a sequence). Completion, failure, and heartbeat calls carry it, and the
  scheduler rejects any whose token isn't the current one. The stale worker learns it lost the job and stops.
- **Heartbeats.** Long jobs extend their lease periodically (e.g., every third of the lease). A heartbeat rejected for a stale token
  tells the worker to abort.
- **Idempotent handlers.** Fencing protects the scheduler's own state, but the stale worker may already have *sent the email* or
  *charged the card*. Side effects need their own deduplication: pass a stable key — the job ID plus the scheduled occurrence, not the
  run ID — to downstream services as an idempotency key (Chapter 8), or record the effect and the job's completion in one transaction.

Choosing the lease length is a trade-off: short leases recover crashed jobs quickly but expire more often on healthy-but-slow workers,
causing duplicate runs; long leases waste time on crashes. Heartbeats break the trade-off for long jobs — a short lease, renewed while
alive.

**Retries and dead letters.** A failed run is rescheduled with exponential backoff and jitter (Chapter 28): `run_at = now + base × 2^attempt
± jitter`. Errors are classified: transient (timeouts, 503s) retry; permanent (validation failure, "customer deleted") go straight to
dead-letter. After `max_attempts`, the job is **dead**: kept with its error history, alerted on by count, and replayable by an operator
after a fix (Chapter 24's dead-letter queue, applied to jobs). An expired lease counts as a failed attempt; otherwise a job that crashes
its worker every time would retry forever.

> 💼 **Interview** — The scheduler question is really a delivery-semantics question in disguise. Say "at-least-once with leases, fencing
> tokens to reject stale workers, and idempotent handlers for exactly-once effects" early, then spend your time on the top-of-the-hour
> burst and fairness, which is where designs differ.

---

## 45.7 Deep Dive: Fairness, Priorities, and Thundering Herds

**Fairness across tenants.** If claiming is "oldest due first", a tenant that schedules 10 million jobs for midnight occupies every worker
until they're done, and every other tenant's midnight jobs wait behind it. Options, from simplest:

- **Round-robin claiming across tenants** (§45.10): take one due job per tenant in turn, so each tenant with work gets a share.
- **Per-tenant concurrency limits**: a tenant may have at most *N* running jobs; excess due jobs wait. This also protects the *tenant's own*
  downstream systems from a burst they created.
- **Weighted fair queuing**: shares proportional to plan or priority, so paying tiers or urgent jobs get more throughput without starving
  the rest.

**Priorities.** Separate dispatch queues per priority, with workers draining higher priorities first but reserving some capacity for lower
ones, so a flood of high-priority jobs can't starve everything else indefinitely.

**The top-of-the-hour herd.** Millions of jobs due at 00:00:00 hit the scheduler, the workers, and every downstream service at once. The fix
is to stop pretending they need to run at that exact second:

- **Jitter at scheduling time.** Recurring jobs whose owners don't need precision get a stable random offset within a window (e.g., 0–5
  minutes), derived from the job ID so it's the same every run — Question 20 implements it. Many scheduling APIs expose this as a
  "run within" window.
- **Rate-limit dispatch** per handler and per downstream (Chapter 29), so a burst becomes a short queue rather than an outage.
- **Pre-warm** worker capacity before known peaks (midnight UTC, month start), since autoscaling reacts after the burst has arrived.

---

## 45.8 Failure Modes, Bottlenecks, and 10×

| Failure | Effect | Mitigation |
|---|---|---|
| Worker crashes mid-job | Job stuck in running | Lease expiry returns it to claimable; counts as an attempt |
| Worker stalls past its lease | Two workers run the job | Fencing tokens reject the stale worker; idempotent handlers absorb the duplicate effect |
| Poller dies | Its shards stop firing | Shard ownership is a lease; another poller takes over within seconds |
| Database primary fails | Scheduling and firing pause | Synchronous replica and fast failover (Chapter 13); jobs due during the gap fire late, per misfire policy |
| Poison job crashes every worker | Repeated crashes, wasted capacity | Expired leases count as attempts; dead-letter after `max_attempts` |
| Clock skew between nodes | Jobs fire early or late; leases misjudged | Use the database's clock (`now()`) for due times and leases, not each node's |
| Midnight burst | Downstream overload, queue spikes | Jitter windows, dispatch rate limits, pre-warmed capacity |

**Workflows.** Jobs that are really sequences — "charge, then email, then update the CRM, compensating on failure" — outgrow a scheduler.
They need a **durable workflow engine** that persists each step's result and resumes from the last completed step (Chapter 23's saga, made
into a runtime). A scheduler is usually one component of such an engine, not a substitute for it.

**At 10×** — 2 billion executions per day, with bursts of tens of millions: the single relational table becomes many shards or a
time-bucketed wide-column store; firing moves to in-memory timing wheels loaded a few seconds ahead from the store; leases and heartbeats
move to a store built for high write rates; and history becomes a log-structured, time-partitioned store (Chapter 48's shape). The
contracts — at-least-once, fencing, idempotency keys, misfire policies — don't change.

---

## 45.9 Common Mistakes and Best Practices

**Mistake: promising exactly-once execution.** The scheduler can't know if a silent worker ran the job. Promise at-least-once and make effects
idempotent.

**Mistake: claiming jobs without a lease.** A job marked "running" by a worker that then died stays running forever.

**Mistake: leases without fencing tokens.** A stalled worker wakes up and completes or fails a job that another worker now owns.

**Mistake: using the run ID as the downstream idempotency key.** Each retry gets a new run ID, so the duplicate effect isn't detected. Key on
the job and its scheduled occurrence.

**Mistake: computing cron schedules in UTC.** "09:00 every weekday" drifts by an hour twice a year for every user not on UTC.

**Mistake: no misfire policy.** After an outage, an every-minute job fires hundreds of times at once, or a daily report silently never runs.

**Mistake: oldest-first claiming in a multi-tenant system.** One tenant's burst starves everyone else.

**Best practice: claim and lease in one transaction, with `SKIP LOCKED`.** No window where a job is selected but unowned.

**Best practice: count expired leases as attempts.** Poison jobs reach the dead-letter state instead of crashing workers forever.

**Best practice: jitter recurring jobs that don't need precision.** Turn the midnight spike into a five-minute plateau.

---

## 45.10 Real-World Example: A Leased Job Store

This is the core of the scheduler's state machine in memory: due-time claiming with leases, fencing tokens on every lease grant, heartbeats,
retries with exponential backoff, dead-lettering, and round-robin claiming across tenants. A production version is the SQL in §45.3 and §45.5
behind the same methods.

```js
// jobstore.mjs — the core of a durable job scheduler: due-time claiming with leases, fencing tokens so a
// stalled worker can't complete a job someone else now owns, retries with backoff, a dead-letter state,
// and round-robin claiming across tenants so one tenant's backlog can't starve the rest.

export class JobStore {
  #jobs = new Map();            // in production: a table indexed on (state, run_at), claimed with SKIP LOCKED
  #nextId = 1;
  #nextToken = 1;               // monotonically increasing fencing token, one per lease grant
  #rrCursor = 0;

  constructor({ leaseMs = 30_000, maxAttempts = 3, baseBackoffMs = 1_000 } = {}) {
    Object.assign(this, { leaseMs, maxAttempts, baseBackoffMs });
  }

  schedule({ tenant, payload, runAt }) {
    const id = this.#nextId++;
    this.#jobs.set(id, { id, tenant, payload, runAt, state: 'scheduled', attempts: 0, leaseUntil: 0, token: 0 });
    return id;
  }

  // A job is claimable if it's due, or if its lease expired (its worker died or stalled)
  #claimable(j, now) {
    return (j.state === 'scheduled' && j.runAt <= now) || (j.state === 'running' && j.leaseUntil <= now);
  }

  claim(worker, now, limit = 1) {
    const byTenant = new Map();
    for (const j of this.#jobs.values()) {
      if (!this.#claimable(j, now)) continue;
      if (!byTenant.has(j.tenant)) byTenant.set(j.tenant, []);
      byTenant.get(j.tenant).push(j);
    }
    const tenants = [...byTenant.keys()].sort();
    for (const list of byTenant.values()) list.sort((a, b) => a.runAt - b.runAt || a.id - b.id);
    const claimed = [];
    for (let i = 0; claimed.length < limit && tenants.some(t => byTenant.get(t).length); i++) {
      const t = tenants[(this.#rrCursor + i) % tenants.length];   // take one job per tenant in turn
      const j = byTenant.get(t).shift();
      if (!j) continue;
      if (j.state === 'running') j.attempts++;                      // the expired lease counts as a failed attempt
      if (j.attempts >= this.maxAttempts) { j.state = 'dead'; continue; }
      Object.assign(j, { state: 'running', worker, leaseUntil: now + this.leaseMs, token: this.#nextToken++ });
      claimed.push({ id: j.id, tenant: j.tenant, token: j.token });
    }
    this.#rrCursor++;
    return claimed;
  }

  heartbeat(id, token, now) {                                       // long jobs extend their lease while alive
    const j = this.#jobs.get(id);
    if (j?.state !== 'running' || j.token !== token) return false;
    j.leaseUntil = now + this.leaseMs;
    return true;
  }

  complete(id, token) {
    const j = this.#jobs.get(id);
    if (j?.state !== 'running' || j.token !== token) return false; // fenced: the lease moved on
    j.state = 'done';
    return true;
  }

  fail(id, token, now) {
    const j = this.#jobs.get(id);
    if (j?.state !== 'running' || j.token !== token) return false;
    j.attempts++;
    if (j.attempts >= this.maxAttempts) { j.state = 'dead'; return true; }
    const backoff = this.baseBackoffMs * 2 ** (j.attempts - 1);
    Object.assign(j, { state: 'scheduled', runAt: now + backoff, leaseUntil: 0 });
    return true;
  }

  get(id) { const { state, attempts, runAt, worker } = this.#jobs.get(id); return { state, attempts, runAt, worker }; }
}

// ── Demo ─────────────────────────────────────────────────────────────────────────────
const store = new JobStore({ leaseMs: 30_000 });
const report = store.schedule({ tenant: 'acme', payload: 'monthly-report', runAt: 0 });

const [a] = store.claim('worker-A', 0);                  // A takes the job, then stalls (GC pause, network split)
const [b] = store.claim('worker-B', 31_000);             // lease expired at 30 s, so B takes it with a newer token
console.log('B token > A token:', b.token > a.token);
console.log('A completes late:', store.complete(report, a.token));   // rejected — A no longer holds the lease
console.log('B completes:', store.complete(report, b.token), store.get(report));

// Fairness: tenant "big" has 6 due jobs, "small" has 2; claims alternate between them
const fair = new JobStore();
for (let i = 0; i < 6; i++) fair.schedule({ tenant: 'big', payload: i, runAt: 0 });
for (let i = 0; i < 2; i++) fair.schedule({ tenant: 'small', payload: i, runAt: 0 });
console.log(fair.claim('w', 0, 4).map(c => c.tenant).join(' '));

// Retries with backoff, then dead-letter
const r = new JobStore({ maxAttempts: 3, baseBackoffMs: 1_000 });
const flaky = r.schedule({ tenant: 'acme', payload: 'sync', runAt: 0 });
let t = 0;
for (let i = 0; i < 3; i++) {
  const [c] = r.claim('w', t);
  r.fail(flaky, c.token, t);
  const s = r.get(flaky);
  console.log(`attempt ${s.attempts}: ${s.state}${s.state === 'scheduled' ? ` again at t=${s.runAt / 1000}s` : ''}`);
  t = s.runAt;
}
// B token > A token: true
// A completes late: false
// B completes: true { state: 'done', attempts: 1, runAt: 0, worker: 'worker-B' }
// big small big small
// attempt 1: scheduled again at t=1s
// attempt 2: scheduled again at t=3s
// attempt 3: dead
```

Worker A's late completion is rejected because its token is stale — exactly the case where, without fencing, A would mark B's run as
done (or failed) underneath it. B's completion shows `attempts: 1`: the expired lease counted. The fairness demo gives the small tenant half
the claims despite having a quarter of the jobs, and the retry demo backs off 1 s, then 2 s, then dead-letters.

The decision to defend is **counting an expired lease as a failed attempt**, rather than treating it as "the worker's fault, not the job's"
and retrying for free. The free-retry view is kinder to jobs on flaky infrastructure: a node that dies of unrelated causes shouldn't push a
job toward the dead-letter state. But the scheduler can't tell an unrelated crash from a job that *causes* the crash — a payload that runs
the worker out of memory, an infinite loop that misses every heartbeat. If expiry is free, that job is retried forever, taking down a worker
each time, and the dead-letter mechanism that exists precisely to catch poison work never triggers. Counting expiry costs at most an
occasional premature dead-letter for a job that was unlucky three times, which is visible, alertable, and fixed by a one-click replay. The
alternative fails silently and expensively.

---

## 45.11 Questions

**1.** MCQ — What execution guarantee can a distributed scheduler actually provide? a) Exactly once b) At least once, with exactly-once effects achieved by idempotent handlers c) At most once only d) None
<details><summary>Answer</summary>**b)** A silent worker may or may not have run the job; the scheduler must run it again.</details>

**2.** MCQ — What does `FOR UPDATE SKIP LOCKED` give a job claim query? a) Faster indexes b) Several claimers can take different due rows concurrently without blocking on each other's locks c) Serializable isolation d) Automatic retries
<details><summary>Answer</summary>**b)** Rows another transaction locked are skipped rather than waited on.</details>

**3.** MCQ — Why does every lease grant get a new, larger token? a) Logging b) So the scheduler can reject heartbeats and completions from a worker whose lease has moved on c) Ordering jobs d) Encryption
<details><summary>Answer</summary>**b)** That's fencing (Chapter 21).</details>

**4.** MCQ — What should a downstream idempotency key for a recurring job's email be based on? a) The run ID b) The job ID plus the scheduled occurrence c) The worker ID d) A random UUID per attempt
<details><summary>Answer</summary>**b)** Retries of the same occurrence then share a key; the next occurrence gets a new one.</details>

**5.** MCQ — What is the main capacity problem for a scheduler with a modest average rate? a) Disk size b) Bursts of jobs due at the same instant (midnight, top of the hour) c) DNS d) TLS
<details><summary>Answer</summary>**b)** Humans schedule on round numbers.</details>

**6.** MCQ — Which store is a natural fit for time-bucketed timers without secondary indexes? a) A wide-column store with one partition per minute bucket b) A CDN c) A Bloom filter d) Object storage only
<details><summary>Answer</summary>**a)** The firer reads the current minute's partition in full.</details>

**7.** Short — Why does a lease alone not prevent duplicate execution?
<details><summary>Answer</summary>A worker can be alive but paused past its lease (GC, partition); the job is re-leased to another worker, and both run it. Fencing stops the stale one from changing scheduler state, and idempotent effects absorb what it already did.</details>

**8.** Short — Describe three misfire policies and a job suited to each.
<details><summary>Answer</summary>Fire once and resume (cache cleanup); fire every missed occurrence (per-period billing, with the period in the payload); skip to the next (a morning digest that's useless by noon).</details>

**9.** Short — How do heartbeats resolve the lease-length trade-off?
<details><summary>Answer</summary>Short leases recover crashed jobs fast but expire on slow, healthy workers; long leases do the opposite. A short lease renewed by heartbeats while the worker is alive gets fast recovery without spurious expiry.</details>

**10.** Short — Why use the database's clock for due times and leases?
<details><summary>Answer</summary>Node clocks are skewed (Chapter 22); if each node judged "due" and "expired" by its own clock, jobs would fire early or late and leases would be reclaimed while still valid. One clock — the store's `now()` — makes those comparisons consistent.</details>

**11.** Short — Name three ways to keep one tenant's burst from delaying others.
<details><summary>Answer</summary>Round-robin claiming across tenants; per-tenant concurrency limits; weighted fair queuing by plan or priority.</details>

**12.** Output — In §45.10, a job with `baseBackoffMs: 1_000` and `maxAttempts: 5` fails at t = 0 and each retry fails immediately. At what times does it run?
<details><summary>Answer</summary>**0 s, 1 s, 3 s, 7 s, 15 s**, then it's dead. Backoffs are 1, 2, 4, and 8 seconds.</details>

**13.** Output — In §45.10's fairness demo, what would `fair.claim('w', 0, 6)` return for tenants?
<details><summary>Answer</summary>**`big small big small big big`.** After small's two jobs are taken, only big has work.</details>

**14.** Output — Using §45.2's model, if the 2,500,000-job burst at :00 is spread over a 10-minute jitter window instead of 5, what's the extra rate?
<details><summary>Answer</summary>**~4,167 jobs/s.** 2,500,000 / 600 s.</details>

**15.** Output — A daily job is set for 01:30 `America/New_York`. On 1 November 2026 (fall back), how many times does 01:30 occur on the wall clock?
<details><summary>Answer</summary>**Twice** — once in EDT and again an hour later in EST. The policy must say which one runs (commonly the first).</details>

**16.** Debug — Invoice emails were sent twice to some customers after a deploy that restarted workers. The handler uses `run.id` as the email service's idempotency key. Why?
<details><summary>Answer</summary>Restarted workers' jobs expired and were re-leased with new run IDs, so the email service saw different keys. Key on the job ID and its scheduled occurrence, which stay the same across retries.</details>

**17.** Debug — A handful of jobs keep "running" forever; their workers were terminated weeks ago. What's missing?
<details><summary>Answer</summary>Leases (or a reaper for expired ones): the jobs were marked running with no expiry. Add `lease_until`, a reaper that returns expired jobs to claimable, and count expiry as an attempt.</details>

**18.** Debug — After a two-hour database outage, a "sync every minute" job ran 120 times in a burst and overloaded a partner API. Fix?
<details><summary>Answer</summary>Its misfire policy was effectively "fire all". Use fire-once for sync-style jobs, and rate-limit dispatch per downstream so recovery doesn't become a burst.</details>

**19.** Debug — Every night at 00:00 UTC, p99 start latency jumps to 12 minutes for all tenants. One tenant schedules 8 million jobs at midnight. What do you change?
<details><summary>Answer</summary>Claiming is oldest-first, so that tenant's backlog occupies every worker. Add round-robin or weighted fair claiming and per-tenant concurrency limits; offer (or default to) jitter windows for jobs that don't need the exact second; pre-warm capacity before midnight.</details>

**20.** Code — Write `jitteredRunAt(scheduledMs, jobId, windowMs)` that returns a time within `[scheduledMs, scheduledMs + windowMs)`, stable for a given job across runs.
<details><summary>Answer</summary>

```js
function jitteredRunAt(scheduledMs, jobId, windowMs) {
  let h = 2166136261;                                   // FNV-1a over the id: stable per job, spread across jobs
  for (const ch of String(jobId)) h = Math.imul(h ^ ch.charCodeAt(0), 16777619) >>> 0;
  return scheduledMs + (h % windowMs);
}
const top = Date.parse('2026-11-15T10:00:00Z');
const offs = ['job-1', 'job-2', 'job-3'].map(id => (jitteredRunAt(top, id, 300_000) - top) / 1000);
console.log(offs.join(' '), jitteredRunAt(top, 'job-1', 300_000) === jitteredRunAt(top, 'job-1', 300_000));
// 270.038 292.419 14.8 true
```
A random offset per run would also spread the load, but a stable one keeps each job's cadence regular — users see "every hour at about :04", not a different time each hour.</details>

**21.** Code — Write `backoffMs(attempt, { baseMs, capMs })` with exponential growth, a cap, and "full jitter", taking an injectable `random` for testing.
<details><summary>Answer</summary>

```js
function backoffMs(attempt, { baseMs = 1_000, capMs = 300_000, random = Math.random } = {}) {
  const ceiling = Math.min(capMs, baseMs * 2 ** attempt);   // grows 1 s, 2 s, 4 s … up to the cap
  return Math.floor(random() * ceiling);                    // full jitter spreads retries of a shared failure
}
console.log([0, 3, 10].map(a => backoffMs(a, { random: () => 0.5 })).join(' ')); // 500 4000 150000
```
</details>

**22.** Code — Write the SQL that returns expired running jobs in shard 3 to `scheduled`, incrementing `attempts`, and dead-letters those that reached `max_attempts`.
<details><summary>Answer</summary>

```sql
-- One statement, so a job is either reclaimed or dead-lettered, never both and never skipped
UPDATE jobs
SET attempts    = attempts + 1,
    state       = CASE WHEN attempts + 1 >= max_attempts THEN 'dead' ELSE 'scheduled' END,
    run_at      = now(),
    lease_until = NULL
WHERE shard = 3 AND state = 'running' AND lease_until <= now();
```
The old `lease_token` stays on the row until the next claim replaces it with a larger one, so a late completion from the expired worker still fails the token check.</details>

**23.** Scenario — Your company wants "send this push notification at 09:00 in each user's local time" for 50 million users. How do you schedule it?
<details><summary>Answer</summary>Don't create 50 million jobs at once, and don't compute in UTC. Create one campaign job per time zone (about 40 distinct offsets at any moment, from IANA zones), each firing at 09:00 local and fanning out through the notification pipeline (Chapter 38) in batches with rate limits. Users' zones are resolved at fire time, so moves are respected; jitter within a few minutes protects the push providers; and the fan-out uses idempotency keys of campaign + user, so retries don't double-send.</details>

**24.** Scenario — A team proposes replacing the scheduler with "a cron entry on one server that calls our API". When is that fine, and when isn't it?
<details><summary>Answer</summary>Fine for a few internal, idempotent jobs where a missed run is noticed and harmless. Not fine when jobs must survive that server's loss (it's a single point of failure with no failover), when missed runs must be detected and caught up (cron has no misfire record), when there are many tenants or dynamic schedules, or when runs need retries, history, and alerting. A middle ground is cron-style triggers in a managed scheduler that enqueue jobs into a durable queue with at-least-once workers.</details>

---

## Chapter Summary

- A scheduler promises at least once, not exactly once: leases recover crashed work, and idempotent handlers keyed on job and occurrence make effects exactly once.
- Claim due jobs and grant their lease in one transaction (`SKIP LOCKED`), with a fresh fencing token on every claim so stalled workers can't change state.
- Index only the hot set — pending and running jobs — by shard and due time; move history out; use time buckets or timing wheels at higher scale.
- Recurring jobs store only their next occurrence, computed in the job's IANA time zone with explicit DST and misfire policies.
- Retry with capped exponential backoff and jitter, count expired leases as attempts, and dead-letter poison jobs with history for replay.
- Fairness and bursts dominate real load: round-robin or weighted claiming, per-tenant limits, jitter windows, dispatch rate limits, and pre-warmed capacity.

---

## Practice Exercises

1. Back §45.10's `JobStore` with PostgreSQL using §45.3's schema and §45.5's claim query, and run four concurrent claimers to verify no job is claimed twice.
2. Add recurring jobs: parse a five-field cron expression, compute the next occurrence in a given IANA time zone, and test both 2026 DST transitions in New York.
3. Measure duplicate runs as a function of lease length when workers have random pauses, with and without heartbeats.
4. Implement weighted fair claiming (tenants with weights 3:1:1) and verify the shares under sustained backlog.
5. Simulate a midnight burst of 1 million jobs with and without a 5-minute jitter window, and plot downstream request rate over time.

---

## Mini Project: A Durable Job Scheduler

Build a scheduler service with a database-backed store, a poller, a worker pool, and an API, and demonstrate its guarantees under failure.

- One-off and recurring jobs (cron + IANA time zone) with idempotent creation, cancel, and status.
- Claims with `SKIP LOCKED`, leases with heartbeats, fencing tokens on every worker call, and a lease reaper.
- Retries with capped, jittered backoff; error classification; a dead-letter view with replay.
- Round-robin tenant fairness, per-tenant concurrency limits, and per-handler dispatch rate limits.
- A chaos test that kills and pauses workers and the poller at random, verifying that every job ran at least once, that fenced workers never changed
  state, and that a downstream service keyed on job + occurrence saw each effect exactly once.

---

**Next:** [Chapter 46 — Design a Collaborative Editor →](46-collaborative-editing.md)
