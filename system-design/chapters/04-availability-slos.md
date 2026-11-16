# Chapter 4 — Availability, SLOs, and Failure Math

## Learning Objectives

- Convert an availability target into a concrete downtime allowance and say what it rules out
- Compute the availability of a system from its components in series and in parallel
- Explain why reducing time-to-recover usually buys more availability than preventing failures
- Define SLIs and SLOs that reflect user experience, and use an error budget to make release decisions
- Identify failure domains and correlated failures that make redundancy math optimistic

---

## 4.1 What "Available" Means

**Availability** is the fraction of the time — or of requests — for which a system does what its
users need. The definition sounds obvious until you try to measure it: is a service that returns
errors for 2% of requests "up"? Is one that answers every request in 30 seconds?

Two ways to count, and they disagree:

| | Time-based | Request-based |
|---|---|---|
| Definition | Good minutes ÷ total minutes | Good requests ÷ total requests |
| A minute is "bad" when | Some threshold is crossed (e.g. > 1% errors) | — |
| 3 a.m. outage, little traffic | Counts the same as a peak-hour outage | Counts for little — few users affected |
| Partial failures (2% of requests fail) | Invisible if below the threshold | Counted exactly |
| Good for | Contractual uptime, low-traffic services | Most user-facing services |

Request-based counting is closer to what users experience and is the default in this book. Use
time-based when traffic is too low for request counts to be statistically meaningful, or when a
contract requires it.

The "nines" shorthand is everywhere, so know what each one allows:

| Availability | Downtime per year | per month | per week | What it implies operationally |
|---|---|---|---|---|
| 99% ("two nines") | 3.7 days | 7.3 h | 1.7 h | Manual recovery is fine |
| 99.9% | 8.8 h | 43.8 min | 10.1 min | On-call humans can fix most incidents in budget |
| 99.95% | 4.4 h | 21.9 min | 5.0 min | Automated failover for stateful parts |
| 99.99% | 52.6 min | 4.4 min | 1.0 min | Humans are too slow; recovery must be automatic |
| 99.999% | 5.3 min | 26 s | 6 s | Multi-region, active-active, near-perfect change management |

```js
// Downtime allowed by an availability target, per period
const MIN_PER_YEAR = 365.25 * 24 * 60;
const fmt = (min) => (min >= 1440 ? `${(min / 1440).toFixed(1)} days`
  : min >= 60 ? `${(min / 60).toFixed(1)} h`
  : min >= 1 ? `${min.toFixed(1)} min` : `${(min * 60).toFixed(0)} s`);

for (const [label, a] of [['99.9%', 0.999], ['99.99%', 0.9999], ['99.999%', 0.99999]]) {
  const down = (1 - a) * MIN_PER_YEAR;
  console.log(label, 'year', fmt(down), '| month', fmt(down / 12));
}
// 99.9% year 8.8 h | month 43.8 min
// 99.99% year 52.6 min | month 4.4 min
// 99.999% year 5.3 min | month 26 s
```

The last column of the table is the useful one. At 99.99%, a single incident that takes an
on-call engineer 15 minutes to notice, page in, and diagnose has spent more than three months of
budget. **Each extra nine is not a little more effort — it's a different operating model**, and
roughly an order of magnitude more cost. That's why the target must come from what the business
actually needs, not from what sounds good.

> 💼 **Interview** — When an interviewer says "it must be highly available", ask "how many nines,
> and for which operations?" Then translate: "99.99% for checkout means ~4 minutes a month, so
> failover has to be automatic; 99.9% for the recommendations panel means it can degrade and we
> hide it." That translation is the signal.

---

## 4.2 Composite Availability: Series and Parallel

A request usually needs several components to work at once. The availability of the whole
follows from two rules.

**In series** — every component must be up (the request passes through all of them) —
availabilities **multiply**, so the whole is *less* available than its weakest part:

```text
A_series = A₁ × A₂ × … × Aₙ
```

**In parallel** — redundant copies, any one of which suffices — the group fails only if *all*
copies fail at once, so unavailabilities multiply:

```text
A_parallel = 1 − (1 − A₁)(1 − A₂)…(1 − Aₙ)
```

```js
// Every dependency on the request path must be up: availabilities multiply
const series = (...as) => as.reduce((acc, a) => acc * a, 1);
// Redundant copies: the group is down only if every copy is down (assumes independence — see §4.6)
const parallel = (...as) => 1 - as.reduce((acc, a) => acc * (1 - a), 1);
const pct = (a, d = 3) => `${(a * 100).toFixed(d)}%`;

// Five services in a call chain, each at 99.9%
console.log(pct(series(0.999, 0.999, 0.999, 0.999, 0.999))); // 99.501%

// Two and three redundant replicas at 99% each
console.log(pct(parallel(0.99, 0.99), 4), pct(parallel(0.99, 0.99, 0.99), 4)); // 99.9900% 99.9999%

// A full stack: one load balancer, three app instances, a primary/standby database pair
const lb = 0.9999, app = parallel(0.995, 0.995, 0.995), db = parallel(0.999, 0.999);
console.log(pct(series(lb, app, db))); // 99.990%
```

Three lessons hide in those numbers.

1. **Deep call chains erode availability.** Five services at 99.9% each give 99.5% end to end —
   five times the downtime of any one. A microservice architecture that puts ten synchronous hops
   on a request path needs each hop far more reliable than the target, or needs the hops to be
   optional (§4.6, Chapter 28).
2. **Redundancy is enormously effective — when failures are independent.** Two 99% replicas give
   99.99%; three give 99.9999%. That's the entire argument for running more than one of anything.
3. **The whole is bounded by the non-redundant parts.** In the last example, the app tier and
   database are redundant and contribute almost nothing; the single load balancer at 99.99% is the
   whole answer. Find the series components with no redundancy — they're the ceiling.

> ⚠️ **Gotcha** — A dependency doesn't have to be on the request path to be in series. If every
> service reads its configuration from one config store at startup, and deploys or autoscaling
> start new instances during that store's outage, the store is in series with every scale-up
> event. "Hidden" dependencies — DNS, the secrets manager, the identity provider, the container
> registry — show up in post-mortems far more often than in architecture diagrams.

---

## 4.3 MTBF, MTTR, and Why Recovery Wins

Availability can also be written in terms of how often things fail and how quickly they recover:

```text
A = MTBF / (MTBF + MTTR)

MTBF = mean time between failures
MTTR = mean time to recovery (detect + diagnose + mitigate)
```

```js
const availability = (mtbfHours, mttrHours) => mtbfHours / (mtbfHours + mttrHours);
const pct = (a) => (a * 100).toFixed(3);

// Fails monthly, 4 hours to recover  vs  fails monthly, 15 minutes  vs  fails weekly, 15 minutes
console.log(pct(availability(30 * 24, 4)), pct(availability(30 * 24, 0.25)), pct(availability(7 * 24, 0.25)));
// 99.448 99.965 99.851
```

Cutting recovery from 4 hours to 15 minutes improves availability far more than halving the
failure rate would — and the system that fails *weekly* but recovers in 15 minutes beats the one
that fails monthly and takes 4 hours. This is the most practical result in the chapter: **for
most teams, the cheapest nine comes from faster recovery, not from fewer failures.**

MTTR breaks into phases, each with its own lever:

| Phase | Typical lever |
|---|---|
| **Detect** — something is wrong | SLO-based alerting on user symptoms, not causes (Chapter 30) |
| **Escalate** — the right person is engaged | Paging that reaches a human in minutes, clear ownership |
| **Diagnose** — what is wrong | Dashboards per dependency, tracing, recent-change lists |
| **Mitigate** — users are no longer affected | Rollback, failover, feature-flag off, shed load — *before* root cause is known |
| **Repair** — the underlying fault is fixed | Can happen after the incident is over |

The key distinction is between **mitigate** and **repair**. Most of the downtime in an incident
is spent on diagnosis, and most incidents are triggered by changes (Chapter 31). So the fastest
mitigation is usually "roll back the last change" — done immediately, without proof it caused
the problem, because a rollback that wasn't needed costs minutes while a diagnosis that takes an
hour costs the budget.

> 💼 **Interview** — "How would you improve this system's availability?" invites a list of
> redundant components. The stronger answer starts with MTTR: "First, I'd make sure we detect
> user-facing errors within a minute and can roll back any deploy in under five. Then
> redundancy for the parts without it."

---

## 4.4 SLIs, SLOs, and SLAs

Three related terms that are often confused:

| Term | What it is | Example | Who cares |
|---|---|---|---|
| **SLI** — service level *indicator* | A measurement of one aspect of service | Fraction of `POST /checkout` requests returning non-5xx in < 800 ms, measured at the load balancer | Engineers |
| **SLO** — service level *objective* | A target for an SLI over a window | 99.9% of checkout requests good, over a rolling 28 days | Engineering + product |
| **SLA** — service level *agreement* | A contract with consequences if missed | Below 99.5% monthly availability, customers receive a 10% service credit | Customers, legal, finance |

The relationship is a hierarchy of strictness: **the SLO is tighter than the SLA**, so the team
gets warning — and room to act — long before contractual penalties apply. An SLA set at or above
the SLO means every missed objective is also a breach.

### Choosing good SLIs

A good SLI measures what users experience, as a ratio of good events to valid events:

```text
SLI = good events / valid events
```

| Service type | Typical SLIs |
|---|---|
| Request/response API | Availability (non-5xx), latency (fraction under threshold) |
| Data pipeline | Freshness (data less than N minutes old), correctness (fraction of records valid) |
| Storage | Durability (no acknowledged write lost), availability of reads and writes |
| Streaming/real-time | Fraction of messages delivered within N seconds |

Notice that latency SLIs are expressed as *the fraction of requests faster than a threshold*,
not as "p99 under X". The two are equivalent statements, but the fraction form composes with
availability into the same good/valid ratio and makes the error budget arithmetic in §4.5
uniform.

Decisions that make SLIs trustworthy:

- **Measure as close to the user as practical.** Server-side success rates miss failures in the
  load balancer, DNS, and TLS layers. Synthetic probes and client-side telemetry catch them.
- **Exclude invalid events deliberately.** Client errors (4xx) caused by bad input usually don't
  count against the service — but a 429 caused by your own overloaded rate limiter should.
- **Prefer a few SLOs per user journey over dozens per service.** "A user can check out" is what
  the business cares about; a hundred internal SLOs dilute attention.

> ⚠️ **Gotcha** — A 100% SLO is never correct. It forbids every change (all changes carry risk),
> it can't be measured faithfully (the client's own network fails more often than that), and
> users can't tell 100% from 99.99% because their ISP, Wi-Fi, and phone are far less reliable
> than your service. Pick the lowest target users won't notice.

---

## 4.5 Error Budgets

An SLO of 99.9% is also a statement that **0.1% of requests are allowed to fail.** That allowance
is the **error budget**, and treating it as a budget to spend — rather than a failure to avoid —
is what makes SLOs useful.

```js
const errorBudget = ({ objective, validRequests }) => Math.floor((1 - objective) * validRequests);

// 400 M requests in a 28-day window at 99.9%
console.log(errorBudget({ objective: 0.999, validRequests: 400e6 })); // 400000
```

400,000 failed requests a month is the budget. It is spent by incidents, by risky deploys, by
planned maintenance, by experiments. The budget turns reliability arguments into arithmetic:

- **Budget remaining → ship.** Release velocity is the point of having a budget. Unused budget at
  the end of the window is reliability the business paid for and users didn't need.
- **Budget exhausted → slow down.** Freeze risky launches; spend engineering time on the causes of
  the overspend. This policy, agreed in advance between product and engineering, is what gives the
  SLO teeth.

### Burn rate

The **burn rate** is how fast the budget is being consumed relative to plan. A burn rate of 1
spends exactly the whole budget over the window; a burn rate of 10 spends it ten times faster.

```text
burn rate = (observed error rate) / (1 − objective)
```

At 99.9% (budget = 0.1% errors), a sustained 1.44% error rate is a burn rate of 14.4, which
exhausts a 30-day budget in about 50 hours. A sustained 0.2% is a burn rate of 2 — bad, but a
ticket rather than a page. Alerting on burn rate over two windows (a short one to detect fast, a
long one to avoid paging on blips) is the standard approach, and Chapter 30 designs those alerts.

> 💼 **Interview** — "Your error budget is exhausted halfway through the month, and product wants
> to launch a major feature. What do you do?" The expected answer references the pre-agreed
> policy: launches pause or proceed only with extra safety (smaller canary, feature flag, a
> rollback rehearsal), and the reliability work that caused the overspend is prioritised. The
> point of the budget is that this isn't a negotiation at the moment of the launch.

---

## 4.6 Failure Domains, Correlation, and Degradation

The parallel formula in §4.2 assumes failures are **independent**. In practice they rarely are,
which is why "we have three replicas" so often fails to deliver the six nines the arithmetic
promises.

### Failure domains

A **failure domain** is the set of components a single fault can take out together:

| Domain | Typical correlated fault |
|---|---|
| Process / host | Crash, OOM, bad disk |
| Rack | Top-of-rack switch, power distribution |
| Availability zone | Power, cooling, network partition of a data centre |
| Region | Regional control-plane outage, natural disaster, a bad regional deploy |
| Provider / global | DNS or identity provider outage, a global configuration push, a certificate expiry |
| **Software version** | The same bug in every replica, triggered by the same input |

The last row is the one that bites hardest. Three replicas across three zones survive a zone
failure; they do not survive a deploy that ships a crash-on-startup bug to all three at once, or
a poison message that crashes every consumer that reads it. **Redundancy protects against
independent hardware faults; only staged change (Chapter 31) protects against correlated software
faults.**

Placement rules that follow:

- **Spread replicas across zones**, not just across hosts. Within-zone redundancy survives host
  failures and nothing larger.
- **N+1 is per failure domain.** If losing one of three zones must be survivable at peak, each
  zone must run at ≤ 67% of peak capacity, so the other two can absorb the load. Teams that size
  for N+1 *hosts* discover during a zone outage that they needed N+1 *zones*.
- **Avoid shared dependencies across "redundant" copies.** Two replicas behind the same
  single-zone database are one replica.

### Blast radius and cells

When correlated failure is inevitable, limit how much of the system any single fault can reach.
A **cell-based** architecture runs many independent copies of the full stack, each serving a slice
of customers, with a thin routing layer in front. A bad deploy, a poison request, or an overloaded
database affects one cell — say 5% of customers — rather than everyone. Chapter 32 returns to
cells in the multi-region context.

### Graceful degradation

Not every dependency deserves to be in series. If the recommendations service is down, the
product page should render without recommendations, not fail. Designing **optional dependencies**
— each with a timeout, a fallback (cached value, default, omission), and a circuit breaker
(Chapter 28) — removes them from the series product entirely:

```text
                ┌─▶ inventory      (required — page fails without it)
product page ──▶├─▶ pricing        (required)
                ├─▶ reviews        (optional — fallback: hide section)
                └─▶ recommendations (optional — fallback: bestsellers from cache)
```

The page's availability is now inventory × pricing × its own, not the product of all four. This
is usually the single largest availability improvement available in a service-oriented system.

### Static stability

A system is **statically stable** if it keeps working when its control plane fails — when it can
no longer make *changes*. Instances keep serving with their last-known configuration if the
config service is down; DNS keeps the last records if the update API is unavailable; a region
already provisioned for failover traffic doesn't need to launch instances during the incident.
The principle: **recovery shouldn't depend on the thing that's broken.** Failovers that need to
call a cloud API to provision capacity in the middle of a regional outage are exactly the ones
that fail, because everyone else is calling that API too.

---

## 4.7 Durability Is Not Availability

**Durability** is the probability that data, once acknowledged, is not lost. **Availability** is
the probability that you can access it right now. A system can be unavailable without losing
anything (the database is down, but the disks are intact) or available while having lost data (it
answers quickly, from a replica that never received the last minute of writes).

They're engineered differently:

| | Availability | Durability |
|---|---|---|
| Threatened by | Crashes, overload, network partitions, deploys | Disk loss, replica loss before replication, corruption, operator error |
| Protected by | Redundant serving capacity, failover, fast recovery | Replication *before acknowledgement*, checksums, backups, deletion protection |
| Typical targets | 99.9%–99.99% | Object stores advertise 99.999999999% ("eleven nines") per object per year |
| Measured | Continuously, via SLIs | Rarely directly — inferred from design and from restore tests |

Eleven nines of durability means that storing 10 million objects, you'd expect to lose one every
10,000 years *to hardware faults*. It says nothing about the most common cause of data loss: a
person or program deleting the data deliberately but wrongly. **Backups exist primarily to protect
against your own mistakes**, which is why they must be logically separate (different account,
deletion protection, retention locks) and why a backup you've never restored is a hypothesis, not
a backup. Chapter 32 covers backup and recovery objectives (RPO/RTO) in detail.

---

## 4.8 Common Mistakes and Best Practices

**Mistake: choosing nines by aspiration.** Every additional nine is roughly an order of magnitude
more cost and a different operating model; the target must come from business impact.

**Mistake: counting replicas but not failure domains.** Three replicas in one zone, or behind one
database, give the availability of one.

**Mistake: forgetting correlated software failure.** Identical code fails identically; redundancy
doesn't help against a bad deploy or a poison message — staged rollout does.

**Mistake: deep synchronous call chains.** Availabilities multiply along the request path, so ten
required hops at 99.9% each give 99%.

**Mistake: SLA equal to SLO.** With no gap, every missed objective is a contractual breach and the
team never gets warning.

**Mistake: measuring success only server-side.** Failures in DNS, TLS, and load balancers never
reach the server's logs, so the SLI reads 100% during real outages.

**Mistake: optimising only MTBF.** Preventing failures has diminishing returns; detection and
mitigation time usually dominate downtime.

**Best practice: make dependencies optional wherever the product allows.** A timeout, a fallback,
and a breaker remove a dependency from the series product.

**Best practice: agree the error-budget policy before you need it.** Deciding what happens when
the budget runs out, in advance, turns a political argument into arithmetic.

**Best practice: mitigate first, diagnose later.** Roll back or fail over as soon as an incident
correlates with a change; root cause can wait until users are unaffected.

**Best practice: test restores, not backups.** Durability against operator error is only as good
as the last successful restore drill.

---

## 4.9 Real-World Example: A Rolling-Window SLO Tracker

An SLO is only useful if the team can see, at any moment, how much budget remains and how fast
it's burning. This tracker keeps a rolling window of good/total counts in fixed memory and reports
the SLI, remaining budget, and short- and long-window burn rates — the inputs to the alerting in
Chapter 30.

```js
// slo-tracker.mjs — request-based SLO over a rolling window, with burn rates.
// Fixed memory: the window is a ring of time buckets, each holding good/total counts.

export class SloTracker {
  #buckets; #bucketMs; #now;

  constructor({ objective, windowMs = 28 * 86_400_000, bucketMs = 60_000, now = Date.now }) {
    if (!(objective > 0 && objective < 1)) throw new RangeError('objective must be in (0, 1)');
    this.objective = objective;
    this.#bucketMs = bucketMs;
    this.#now = now;
    // One slot per bucket; each slot remembers which bucket id it holds, so slots left
    // over from a previous lap around the ring are detected and treated as empty
    this.#buckets = Array.from({ length: Math.ceil(windowMs / bucketMs) },
      () => ({ id: -1, good: 0, total: 0 }));
  }

  #at(id) { return this.#buckets[((id % this.#buckets.length) + this.#buckets.length) % this.#buckets.length]; }

  record(good, count = 1) {
    const id = Math.floor(this.#now() / this.#bucketMs);
    const s = this.#at(id);
    if (s.id !== id) { s.id = id; s.good = 0; s.total = 0; } // reclaim a stale slot
    s.total += count;
    if (good) s.good += count;
  }

  // Sum the buckets covering the last `spanMs` (the whole window by default)
  #sum(spanMs = this.#buckets.length * this.#bucketMs) {
    const current = Math.floor(this.#now() / this.#bucketMs);
    const n = Math.min(this.#buckets.length, Math.ceil(spanMs / this.#bucketMs));
    let good = 0, total = 0;
    for (let id = current - n + 1; id <= current; id++) {
      const s = this.#at(id);
      if (s.id === id) { good += s.good; total += s.total; }
    }
    return { good, total };
  }

  // 1.0 means "on pace to spend exactly the whole budget over the window"
  burnRate(spanMs) {
    const { good, total } = this.#sum(spanMs);
    return total === 0 ? 0 : (1 - good / total) / (1 - this.objective);
  }

  status() {
    const { good, total } = this.#sum();
    const allowedBad = (1 - this.objective) * total;
    return {
      sli: total ? good / total : 1,
      // Goes negative once overspent; keep the sign, it says by how much
      budgetRemaining: allowedBad ? 1 - (total - good) / allowedBad : 1,
      burn1h: this.burnRate(3_600_000),
      burn6h: this.burnRate(6 * 3_600_000),
    };
  }
}

// Simulated clock: 10 days at 2,000 req/min and 99.95% good, then a 2-hour incident at 5% errors
let t = 0;
const slo = new SloTracker({ objective: 0.999, now: () => t });
for (let minute = 0; minute < 10 * 1440; minute++, t += 60_000) {
  slo.record(true, 1_999); slo.record(false, 1);
}
for (let minute = 0; minute < 120; minute++, t += 60_000) {
  slo.record(true, 1_900); slo.record(false, 100);
}
t -= 60_000; // read status at the last recorded minute
const s = slo.status();
console.log(s.sli.toFixed(5), s.budgetRemaining.toFixed(2), s.burn1h.toFixed(1), s.burn6h.toFixed(1));
// 0.99909 0.09 50.0 17.0
```

The output tells the story in four numbers: the SLI still reads a healthy-looking 99.909%, but
two hours of incident spent most of the budget — only 9% remains — and the one-hour burn rate of
50 says the remainder will be gone within the hour if nothing changes.

The decision to defend is **a rolling window instead of a calendar month.** Calendar windows are
easier to explain and match billing cycles, but they reset: an incident on the 30th is "free"
because the budget refills on the 1st, and a team that blew its budget early in the month gets a
clean slate regardless of whether anything was fixed. A rolling window always reflects the last 28
days of user experience, so the budget recovers only as the bad days age out — which is the same
rate at which users forget them. Use calendar windows for SLAs (contracts need them); use rolling
windows to steer engineering.

---

## 4.10 Questions

**1.** MCQ — Approximately how much downtime per month does a 99.95% availability target allow? a) 4 minutes b) 22 minutes c) 44 minutes d) 7 hours
<details><summary>Answer</summary>**b)** 0.05% of ~43,800 minutes per month ≈ 22 minutes.</details>

**2.** MCQ — A request passes through four required services, each 99.9% available, with independent failures. What is the end-to-end availability? a) 99.9% b) 99.6% c) 99.0% d) 96%
<details><summary>Answer</summary>**b)** 0.999⁴ ≈ 0.996. Availabilities in series multiply.</details>

**3.** MCQ — Which improvement raises availability the most for a service that fails about once a month and takes 3 hours to recover? a) Halving the failure rate b) Cutting recovery time to 10 minutes c) Adding a fourth replica in the same zone d) Moving to faster disks
<details><summary>Answer</summary>**b)** Halving failures cuts downtime from ~3 h to ~1.5 h a month; cutting MTTR to 10 minutes cuts it to ~10 minutes. Same-zone replicas don't help zone failures, and disks are irrelevant.</details>

**4.** MCQ — Which pairing of SLO and SLA is sensible? a) SLO 99.9%, SLA 99.95% b) SLO 99.9%, SLA 99.9% c) SLO 99.95%, SLA 99.9% d) SLO 100%, SLA 99.99%
<details><summary>Answer</summary>**c)** The SLO must be stricter than the SLA so the team gets warning and room to act before contractual penalties apply.</details>

**5.** MCQ — Three replicas run in separate zones. Which fault does this redundancy *not* protect against? a) One host's disk failing b) One zone losing power c) A deploy shipping a crash-on-startup bug to all replicas d) A network switch failing in one zone
<details><summary>Answer</summary>**c)** Identical software fails identically. Only staged rollout and fast rollback protect against correlated software faults.</details>

**6.** MCQ — What does a burn rate of 4 mean for a 28-day SLO window? a) Four incidents have occurred b) The budget will be exhausted in about 7 days at this rate c) Availability is 4% d) The SLO is four nines
<details><summary>Answer</summary>**b)** Spending budget 4× faster than plan exhausts a 28-day budget in 28 / 4 = 7 days.</details>

**7.** Short — Why is request-based availability usually a better measure than time-based uptime for a user-facing service?
<details><summary>Answer</summary>It weights failures by how many users they affect: a peak-hour outage counts far more than a 3 a.m. one, and partial failures (2% of requests failing) are counted exactly rather than hidden below a "bad minute" threshold.</details>

**8.** Short — Why does adding a fourth nine usually require automated recovery?
<details><summary>Answer</summary>99.99% allows about 4.4 minutes of downtime a month. Detecting, paging, and diagnosing an incident with humans in the loop routinely takes longer than that, so failover and rollback must happen without waiting for a person.</details>

**9.** Short — What makes a dependency "optional", and why does that matter for availability?
<details><summary>Answer</summary>The caller has a timeout, a fallback (cache, default, or omission), and usually a circuit breaker, so the request succeeds when the dependency fails. Optional dependencies drop out of the series product, so their availability no longer caps the caller's.</details>

**10.** Short — What is static stability?
<details><summary>Answer</summary>A system's ability to keep serving with its last-known state when its control plane — config service, provisioning APIs, DNS updates — is unavailable. Recovery shouldn't require calling the thing that's broken.</details>

**11.** Short — Why do backups protect against something replication cannot?
<details><summary>Answer</summary>Replication faithfully copies every change, including a mistaken `DELETE` or a corrupting bug, to every replica within seconds. Backups are point-in-time and logically separate, so they can restore the state before the mistake.</details>

**12.** Output — What does this print?
```js
const series = (...as) => as.reduce((acc, a) => acc * a, 1);
const parallel = (...as) => 1 - as.reduce((acc, a) => acc * (1 - a), 1);
console.log((series(0.9999, parallel(0.99, 0.99)) * 100).toFixed(2));
```
<details><summary>Answer</summary>`99.98` — two 99% replicas in parallel give 99.99%, then in series with a 99.99% component: 0.9999 × 0.9999 ≈ 99.98%.</details>

**13.** Output — A service has an MTBF of 500 hours and an MTTR of 30 minutes. What is its availability, to two decimal places of a percent?
<details><summary>Answer</summary>**99.90%.** 500 / (500 + 0.5) = 0.999001.</details>

**14.** Output — A service handles 90 M valid requests in a 30-day window with a 99.95% SLO. How many failed requests does the error budget allow?
<details><summary>Answer</summary>**45,000.** 0.0005 × 90,000,000 = 45,000.</details>

**15.** Output — With a 99.9% SLO, the service is currently failing 0.5% of requests. What is the burn rate, and how long would a full 30-day budget last at that rate?
<details><summary>Answer</summary>**Burn rate 5; about 6 days.** 0.005 / 0.001 = 5; 30 / 5 = 6 days.</details>

**16.** Debug — A team's dashboard showed 100% availability during an incident in which customers couldn't reach the site at all. The SLI is computed from application server logs. Why, and how should it change?
<details><summary>Answer</summary>Requests that failed in DNS, TLS, or the load balancer never reached the application, so they're absent from both numerator and denominator. Measure at the load balancer at minimum, and add synthetic probes or client-side telemetry to catch failures before it.</details>

**17.** Debug — A service runs six instances across three zones and is sized so peak traffic uses 85% of total capacity. During a zone outage it failed completely rather than degrading. Explain.
<details><summary>Answer</summary>Losing a zone removed a third of capacity, so the remaining four instances faced ~128% of their capacity, overloaded, and fell over (made worse by retries). To survive a zone loss at peak, capacity must be N+1 per zone: each zone ≤ ~67% of peak with three zones, plus load shedding for the excess.</details>

**18.** Debug — During a regional cloud incident, a team's automated failover to a second region failed: the failover script tried to launch 200 instances in the standby region and the requests timed out. What principle was violated, and what's the fix?
<details><summary>Answer</summary>Static stability: recovery depended on a control-plane action (provisioning) at the moment that control plane was overloaded by everyone else failing over. Pre-provision the standby region's capacity (or enough to absorb failover), so failover is a routing change, not a provisioning job.</details>

**19.** Debug — A product's SLO is 99.9% and the team meets it every month, yet customer complaints about errors keep rising. Investigation shows the SLI counts all `/api/*` requests, dominated by a high-volume, very reliable health-polling endpoint. What's wrong?
<details><summary>Answer</summary>The SLI dilutes the user journeys that matter with cheap, reliable traffic, so a checkout endpoint can fail 5% of the time while the aggregate stays above 99.9%. Define SLIs per user journey (or weight them), exclude synthetic and polling traffic, and set SLOs for the critical paths separately.</details>

**20.** Code — Write `downtimeAllowed(objective, days)` that returns the allowed downtime in minutes, rounded to one decimal, for a time-based availability objective over a window of `days`.
<details><summary>Answer</summary>

```js
const downtimeAllowed = (objective, days) => Math.round((1 - objective) * days * 1440 * 10) / 10;
console.log(downtimeAllowed(0.999, 30), downtimeAllowed(0.9999, 28)); // 43.2 4
```
</details>

**21.** Code — Write `compose(tree)` that computes availability for a nested description such as `{ series: [0.9999, { parallel: [0.99, 0.99] }, 0.999] }`, where leaves are numbers.
<details><summary>Answer</summary>

```js
function compose(node) {
  if (typeof node === 'number') return node;
  if (node.series) return node.series.reduce((acc, n) => acc * compose(n), 1);
  if (node.parallel) return 1 - node.parallel.reduce((acc, n) => acc * (1 - compose(n)), 1);
  throw new TypeError('node must be a number, {series}, or {parallel}');
}
console.log(compose({ series: [0.9999, { parallel: [0.99, 0.99] }, 0.999] }).toFixed(5)); // 0.99880
```
</details>

**22.** Code — Write `shouldPage({ burnShort, burnLong, threshold })` that returns `true` only when both a short-window and a long-window burn rate exceed the threshold, and explain in a comment why both are required.
<details><summary>Answer</summary>

```js
function shouldPage({ burnShort, burnLong, threshold = 14.4 }) {
  // The long window proves the burn is sustained (not a 30-second blip);
  // the short window proves it is still happening (so the page stops soon after recovery)
  return burnShort > threshold && burnLong > threshold;
}
console.log(shouldPage({ burnShort: 40, burnLong: 16 }), shouldPage({ burnShort: 40, burnLong: 3 })); // true false
```
</details>

**23.** Scenario — The CEO wants the product's SLO raised from 99.9% to 99.99% "because competitors advertise it". The service has one primary database with manual failover that takes 20–40 minutes. What do you tell them?
<details><summary>Answer</summary>Translate it: 99.99% is 4.4 minutes a month, and a single manual database failover consumes five to ten months of that budget. Reaching it means automated failover, multi-zone everything, staged deploys with automatic rollback, and likely cells — a significant, ongoing cost. Ask what user-visible problem the extra nine solves; often the better investment is 99.95% for the critical journeys, with graceful degradation for the rest. If the business case is real, present the plan and its cost explicitly.</details>

**24.** Scenario — Your team has spent 120% of its error budget this window, mostly from two bad deploys. Product has a launch scheduled next week that a large customer is waiting for. What do you recommend?
<details><summary>Answer</summary>Follow the agreed budget policy rather than negotiating it: pause risky changes, but treat the launch as a special case with extra safety — behind a feature flag, enabled first for the waiting customer only, with a rehearsed rollback. In parallel, prioritise the deploy-safety work that caused the overspend (canaries, automatic rollback on SLO burn). If no policy exists, this is the moment to write one with product, so the next case isn't a negotiation.</details>

---

## Chapter Summary

- Each extra nine is roughly an order of magnitude more cost and a different operating model;
  translate targets into minutes and pick the lowest one users won't notice.
- Availabilities multiply in series and unavailabilities multiply in parallel — so deep required
  call chains erode availability and redundancy restores it, if failures are independent.
- Recovery speed usually buys more availability than failure prevention; detect on user symptoms
  and mitigate (roll back, fail over) before diagnosing.
- SLIs are good-over-valid ratios measured near the user; SLOs set targets stricter than SLAs; the
  error budget turns reliability trade-offs into an agreed spending policy.
- Correlated failures — shared zones, shared dependencies, and above all identical software — break
  the independence that redundancy math assumes; staged change and cells limit the blast radius.
- Durability and availability are different properties; backups protect against your own mistakes
  and are only as real as your last restore test.

---

## Practice Exercises

1. For a service you know, list every dependency on the request path — including DNS, identity,
   and config — and compute the series availability using each one's published SLA.
2. Pick one of those dependencies and design it as optional: timeout, fallback, and what the user
   sees. Recompute the composite availability.
3. Read a public post-mortem from a major cloud provider and classify the root cause by failure
   domain (host, zone, region, global, software version). Was the failure correlated?
4. Instrument a service with the §4.9 tracker (or your metrics stack's equivalent) and chart the
   1-hour and 6-hour burn rates through a week, marking deploys on the chart.
5. Write an error-budget policy for a team: what happens at 50%, 100%, and 150% of budget spent,
   who decides exceptions, and how the policy itself is reviewed.

---

## Mini Project: An Availability Model and SLO Report

Build a tool that takes a YAML or JSON description of a service's dependency graph — each node
with an availability, a failure domain, and whether it's required or optional — and reports the
composite availability, the components that cap it, and the effect of each candidate improvement.

- Supports series, parallel, and optional (fallback) dependencies, nested arbitrarily.
- Flags "redundant" groups whose members share a failure domain, and recomputes treating them as
  one component.
- Ranks improvements (make X redundant, make Y optional, cut MTTR of Z) by availability gained.
- Ingests a CSV of per-minute good/total request counts and produces an SLO report: SLI, budget
  remaining, worst burn-rate periods, and which days spent the most budget.
- Includes two example graphs: a monolith with a database, and a 12-service microservice request
  path.

---

**Next:** [Chapter 5 — Networking for System Designers →](05-networking.md)
