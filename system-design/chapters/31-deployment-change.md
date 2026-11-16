# Chapter 31 — Deployment and Change Management

## Learning Objectives

- Treat every change — code, configuration, flags, schema, infrastructure — as the most likely cause of the next outage
- Choose between rolling, blue/green, canary, and shadow deployments for a change
- Judge a canary statistically against a concurrent baseline and roll back automatically
- Use feature flags to separate deploying from releasing, without accumulating flag debt
- Change database schemas and data online, with every intermediate state compatible with both code versions

---

## 31.1 Most Outages Are Changes

Systems that are left alone tend to keep working. The common trigger for an outage is a **change**: a deploy, a
configuration edit, a feature flag flipped for everyone, a schema migration, a certificate rotation, an
infrastructure update, a dependency upgrade. Post-incident reviews at large organisations consistently find that a
large majority of incidents follow a change — which makes change management the single largest lever on availability
after recovery speed (Chapter 4).

That doesn't argue for changing less. Infrequent, large releases are riskier: they bundle many changes, so a failure
is harder to attribute, harder to roll back, and touches more. The safer pattern is the opposite — **small changes,
shipped often, exposed gradually, observed closely, and reversible quickly**. Each of those properties is a technique
in this chapter:

| Property | Technique |
|---|---|
| Small | Trunk-based development, small pull requests, one concern per deploy |
| Exposed gradually | Canary and progressive delivery, percentage rollouts, cells and regions in waves |
| Observed | SLO-based canary analysis, deploy markers on dashboards (Chapter 30) |
| Reversible | Fast rollback, feature-flag kill switches, expand-contract schema changes |
| Decoupled | Deploying code separately from releasing behaviour; migrations separately from deploys |

And every kind of change gets the same treatment. Configuration and feature flags cause outages as often as code —
a global flag flip reaches every user in seconds, with none of a deploy's gradual exposure — so they deserve the same
review, staging, and gradual rollout.

---

## 31.2 Release Strategies

| Strategy | How | Blast radius while bad | Rollback | Cost |
|---|---|---|---|---|
| **Recreate** | Stop all old, start all new | Everyone, plus downtime | Redeploy old | Simple; downtime |
| **Rolling** | Replace instances in batches | Grows batch by batch | Roll forward/back in batches; old and new mix | Default in most orchestrators |
| **Blue/green** | Stand up the full new version beside the old; switch traffic at once | Everyone, from the switch | Switch back — instant | Double capacity during the switch |
| **Canary** | New version takes a small share of traffic first; expand as it proves healthy | The canary's share (1–5%) | Route traffic away from the canary | Needs traffic splitting and analysis |
| **Shadow (dark launch)** | New version receives a *copy* of traffic; responses discarded | None (if side effects are suppressed) | Nothing to roll back | Duplicate load; side effects must be disabled |

**Canary releases are the default worth building toward**, because they bound the damage of an undetected bug to a
small share of traffic *and* provide a real-traffic comparison against the old version. Progressive delivery automates
the sequence: 1% for 10 minutes, 5%, 25%, 50%, 100%, with an automated judgement at each step and automatic rollback on
failure. At larger scale, the same idea applies geographically: one cell (Chapter 4), then one zone, then one region,
then the rest — never everything at once.

**Blue/green** suits changes where mixing versions is a problem (some stateful upgrades) and makes rollback a switch flip,
but it exposes everyone at the moment of cut-over. **Shadow** traffic is valuable for risky rewrites — a new search ranker,
a re-platformed pricing engine — where you can compare outputs on real requests before anyone sees them, provided writes,
emails, and charges are suppressed in the shadow.

All strategies except recreate run **old and new versions simultaneously**, which makes compatibility a requirement
(§31.6).

> 💼 **Interview** — "How do you deploy safely?" The answer interviewers want: small, frequent deploys; canary to a
> small slice with automated comparison against the old version on SLO metrics; automatic rollback; feature flags to
> decouple release from deploy; schema changes expand-then-contract; and rollout in waves across cells or regions.

---

## 31.3 Judging a Canary

A canary only helps if someone (or something) notices when it's worse. Two approaches:

- **Fixed thresholds** — roll back if the canary's error rate exceeds 1%. Simple, but blind to context: during an
  upstream incident *every* version exceeds 1%, so the canary is rolled back for something it didn't cause; and a canary
  whose error rate rose from 0.05% to 0.5% passes, though it's ten times worse.
- **Comparison with a baseline** — run a few instances of the *old* version, receiving the same share and kind of traffic
  at the same time, and compare canary against baseline statistically. Ambient conditions (an upstream blip, a traffic
  spike, time of day) affect both equally and cancel out; only the difference the new version makes remains.

Comparison needs enough traffic to distinguish signal from noise — a canary that has served 300 requests with one error
can't be told apart from a baseline at 0.2% — so a judge must also be able to say **wait**. The real-world example in §31.9
implements such a judge: an absolute guard for obviously broken builds, a minimum sample size, a two-proportion test on
error rates, and a latency comparison.

Good canary metrics are the SLIs (Chapter 4): error rate and latency at the service's edge, plus business signals where
they're fast enough (checkout conversion, payment success). Resource metrics (CPU, memory) are useful context — a version
that uses 40% more memory will fall over at peak — but shouldn't be the only gate.

---

## 31.4 Feature Flags: Deploying Is Not Releasing

A **feature flag** wraps new behaviour in a runtime condition, so code can be **deployed** (present in production, turned
off) separately from being **released** (turned on for users). That decoupling enables:

- **Trunk-based development** — unfinished work merges continuously behind a flag instead of living in long branches.
- **Gradual release** — 1% of users, then 10%, then everyone, independently of deploys.
- **Targeted release** — internal users first, then a beta cohort, then one country.
- **Kill switches** — turning a misbehaving feature off in seconds, without a deploy or rollback.
- **Experiments** — A/B tests with stable assignment.

Percentage rollouts need **stable assignment**: a user who sees the new checkout at 10% must keep seeing it at 50%, or the
experience flickers and metrics are polluted. Hash the user (and the flag's key, so different flags bucket users
independently) into a fixed range and compare against the percentage:

```js
// Feature flag evaluation: explicit rules first, then a percentage rollout by stable hash,
// so each user consistently sees the same variant as the percentage grows.
import { createHash } from 'node:crypto';

const bucket = (flag, userId) => createHash('sha256').update(`${flag}:${userId}`).digest().readUInt32BE(0) % 10_000;

function evaluate(flag, ctx) {
  if (flag.killed) return false;                                   // kill switch beats everything
  for (const rule of flag.rules ?? []) {
    if (rule.attribute in ctx && rule.values.includes(ctx[rule.attribute])) return rule.enabled;
  }
  // Hash flag + user: users are spread independently per flag, and 10% today ⊂ 50% tomorrow
  return bucket(flag.key, ctx.userId) < flag.percent * 100;
}

const flag = { key: 'new-checkout', percent: 10, rules: [
  { attribute: 'tenant', values: ['internal'], enabled: true },     // dogfood first
  { attribute: 'country', values: ['DE'], enabled: false },         // not yet legally reviewed
] };
const users = Array.from({ length: 10_000 }, (_, i) => `u${i}`);
const on10 = users.filter((u) => evaluate(flag, { userId: u, country: 'PT' }));
const on50 = users.filter((u) => evaluate({ ...flag, percent: 50 }, { userId: u, country: 'PT' }));
console.log(on10.length, on50.length, on10.every((u) => on50.includes(u)));   // 952 4913 true
console.log(evaluate(flag, { userId: 'u1', tenant: 'internal' }), evaluate(flag, { userId: 'u1', country: 'DE' }),
  evaluate({ ...flag, killed: true }, { userId: 'u1', tenant: 'internal' })); // true false false
```

About 10% and 50% of users are enabled at each step, and every user enabled at 10% is still enabled at 50%.

Flags have costs that must be managed:

- **Flag debt.** Every flag doubles the paths through a piece of code. Release flags should be removed within weeks of
  reaching 100%; track each flag's owner and expiry date, and fail CI or alert on stale flags.
- **Flags are configuration changes.** Flipping a flag to 100% instantly is a big-bang release with no canary. Roll flags
  out by percentage with the same observation as code, and audit who changed what.
- **The flag service is a dependency.** Evaluate flags locally from a cached rule set (streamed or polled), with safe
  defaults if the service is unreachable — never a network call per evaluation on the request path (Chapter 4's static
  stability).
- **Test both paths.** A flag that's been on everywhere for months may have an "off" path that no longer works — which is
  the path a kill switch relies on.

---

## 31.5 Changing Schemas and Data Online

Code can be rolled back in minutes; data changes often can't. Schema migrations therefore need the most care, and two
rules cover most of it:

**Every migration must be compatible with both the old and the new code**, because during a rolling or canary deploy both
run at once, and after a rollback the old code runs against the new schema. That rules out renaming or dropping a column
in the same release as the code change. Chapter 12's **expand–migrate–contract** sequence is the general pattern:

```text
release 1 (expand):   add new column `email_normalized` (nullable); code writes BOTH old and new
backfill:             fill `email_normalized` for existing rows in small batches
release 2:            code reads the new column (falls back to old if NULL); still writes both
release 3 (contract): code stops writing the old column
release 4:            drop the old column — only after release 3 is fully rolled out and stable
```

Each step is independently deployable and reversible, and no single release depends on another having finished.

**Migrations must not lock tables in ways that block production traffic.** Many DDL operations take exclusive locks; on a
large, busy table, even a short wait for that lock queues every query behind it (Chapter 15's migration gotcha). Practices:

```sql
-- PostgreSQL: fail fast instead of queueing every query behind a waiting DDL statement
SET lock_timeout = '3s';
ALTER TABLE users ADD COLUMN email_normalized text;              -- fast: no table rewrite

-- Build indexes without blocking writes (takes longer, can't run in a transaction)
CREATE INDEX CONCURRENTLY users_email_normalized ON users (email_normalized);

-- Add a constraint without a long validating lock, then validate separately
ALTER TABLE users ADD CONSTRAINT email_normalized_present CHECK (email_normalized IS NOT NULL) NOT VALID;
ALTER TABLE users VALIDATE CONSTRAINT email_normalized_present;  -- scans without blocking writes

-- Backfill in small keyset batches, committing each, throttled between batches
UPDATE users SET email_normalized = lower(email)
 WHERE id IN (SELECT id FROM users WHERE email_normalized IS NULL AND id > $1 ORDER BY id LIMIT 1000);
```

Further rules: **run migrations separately from deploys** (a migration that fails halfway shouldn't leave a half-deployed
application, and vice versa); **throttle backfills** and watch replication lag (Chapter 13), since a backfill can flood
replicas; and for very large tables, use online schema-change tools (gh-ost and pt-online-schema-change for MySQL, or
built-in online DDL where the database provides it) that copy into a shadow table and swap.

> ⚠️ **Gotcha** — **Some "simple" DDL rewrites the whole table.** Changing a column's type, adding a column with a volatile
> default, or adding certain constraints can rewrite every row while holding an exclusive lock — minutes to hours on a
> large table, during which writes (and sometimes reads) block. Check what each statement does on your database version
> before running it in production, and test migrations against a production-sized copy.

---

## 31.6 Compatibility During a Rollout

Any deployment strategy except recreate mixes versions, so every interface the change touches must work across the mix:

- **APIs between services** — the new server must accept requests from old clients, and the new client must work with old
  servers until they're upgraded (Chapter 8's additive-change rules). Deploy servers before the clients that depend on
  their new capabilities.
- **Messages and events** — consumers may be older or newer than producers, and messages produced by the old version sit
  in queues after the deploy. Keep schemas backward and forward compatible (Chapter 24).
- **Caches and serialised state** — a new version that writes a new format into a shared cache can crash old instances
  that read it. Version cached formats, or use new keys for new formats.
- **Mobile and desktop clients** — they update on users' schedules, if at all. Servers must support several client
  versions for months, and the oldest supported version needs an enforced minimum.
- **Rollback is a compatibility case too** — the old version must be able to run against whatever the new version wrote
  (data, cache entries, messages). If it can't, rollback isn't actually available, and that should be known before
  deploying.

---

## 31.7 Rollback, Roll Forward, and Irreversible Changes

When a deploy goes wrong, the fastest mitigation is almost always **rollback**: restore the previous version, then
diagnose. Rolling forward with a fix is sometimes faster, but it's a new, untested change made under pressure. A good
default policy: roll back first unless rolling back is unsafe or the fix is trivial and tested.

Make rollback cheap and certain:

- **Keep the previous version deployable and its artefacts available**, with the same pipeline that deploys forward.
- **Automate it**: progressive delivery tools should roll back automatically on failed analysis, and manual rollback should
  be one command.
- **Rehearse it**: a rollback path that's never exercised may not work when needed — just like fallbacks (Chapter 28).

Some changes **can't be rolled back**: data deleted or transformed destructively, emails sent, payments made, a schema
contracted, a message format consumed by systems you don't control. Identify them before deploying, separate them from
reversible changes (ship the reversible part first and observe), take backups or snapshots immediately before (Chapter 32),
and require explicit review.

**Change freezes** — no deploys during peak events like Black Friday — reduce risk temporarily, but the changes queue up
and ship together afterwards, which is riskier. Prefer making changes safer to making them rarer; when freezes are necessary,
keep them short and allow small, low-risk changes through a reviewed exception path.

---

## 31.8 Common Mistakes and Best Practices

**Mistake: big, infrequent releases.** Many changes ship together, failures are hard to attribute, and rollback undoes
everything.

**Mistake: flags and config changes without rollout.** A global flip has no canary; it's a big-bang release in disguise.

**Mistake: canary judged against fixed thresholds only.** Ambient incidents roll back healthy canaries, and large relative
regressions pass.

**Mistake: schema changes that break the old code.** Mixed versions during rollout, and rollback afterwards, both run old
code against the new schema.

**Mistake: migrations coupled to deploys.** A failed migration leaves a half-deployed application, and vice versa.

**Mistake: DDL without a lock timeout on busy tables.** The waiting statement queues every query behind it.

**Mistake: never removing flags.** Flag debt multiplies code paths and leaves untested "off" paths behind kill switches.

**Best practice: small changes, progressive exposure, automated comparison, fast rollback.** The combination is what makes
frequent deploys safe.

**Best practice: expand–migrate–contract for every schema change.** Every intermediate state works with both code versions.

**Best practice: roll back first, diagnose second.** Restore service with a known-good version, then find the cause.

---

## 31.9 Real-World Example: A Canary Judge

This judge decides whether a canary is safe to promote by comparing it with a **baseline** — the old version, serving an
equal slice of the same traffic at the same time. It rolls back immediately on a catastrophic error rate, waits until both
groups have enough traffic to compare, uses a two-proportion z-test so that a higher canary error rate must exceed what
chance explains, and compares tail latency.

```js
// canary-judge.mjs — decide whether a canary is safe to promote by comparing it with a
// baseline running the OLD version at the same time, rather than against fixed thresholds.

export function judge({ baseline, canary, minRequests = 2_000, maxZ = 3, maxLatencyRatio = 1.2, hardErrorRate = 0.05 }) {
  const rate = (g) => g.errors / g.requests;
  // Absolute guard: a canary failing badly is rolled back immediately, whatever the baseline does
  if (canary.requests >= 100 && rate(canary) > hardErrorRate) {
    return { decision: 'rollback', reason: `canary error rate ${(rate(canary) * 100).toFixed(1)}% above hard limit` };
  }
  if (canary.requests < minRequests || baseline.requests < minRequests) {
    return { decision: 'wait', reason: `need ${minRequests} requests per group` };
  }
  // Two-proportion z-test: is the canary's error rate higher than chance would explain?
  const pooled = (canary.errors + baseline.errors) / (canary.requests + baseline.requests);
  const se = Math.sqrt(pooled * (1 - pooled) * (1 / canary.requests + 1 / baseline.requests));
  const z = se === 0 ? 0 : (rate(canary) - rate(baseline)) / se;
  if (z > maxZ) {
    return { decision: 'rollback', reason: `errors ${(rate(canary) * 100).toFixed(2)}% vs ${(rate(baseline) * 100).toFixed(2)}% (z=${z.toFixed(1)})` };
  }
  const latencyRatio = canary.p99Ms / baseline.p99Ms;
  if (latencyRatio > maxLatencyRatio) {
    return { decision: 'rollback', reason: `p99 ${canary.p99Ms} ms vs ${baseline.p99Ms} ms (${latencyRatio.toFixed(2)}×)` };
  }
  return { decision: 'promote', reason: `errors z=${z.toFixed(1)}, p99 ratio ${latencyRatio.toFixed(2)}` };
}

const scenarios = {
  // An upstream incident raises errors for BOTH groups: not the canary's fault
  'ambient incident': { baseline: { requests: 50_000, errors: 1_500, p99Ms: 420 }, canary: { requests: 5_000, errors: 160, p99Ms: 430 } },
  'real regression':  { baseline: { requests: 50_000, errors: 100, p99Ms: 180 },   canary: { requests: 5_000, errors: 40, p99Ms: 185 } },
  'slower version':   { baseline: { requests: 50_000, errors: 100, p99Ms: 180 },   canary: { requests: 5_000, errors: 11, p99Ms: 260 } },
  'too early':        { baseline: { requests: 50_000, errors: 100, p99Ms: 180 },   canary: { requests: 300, errors: 1, p99Ms: 175 } },
  'broken build':     { baseline: { requests: 50_000, errors: 100, p99Ms: 180 },   canary: { requests: 300, errors: 90, p99Ms: 90 } },
};
for (const [name, s] of Object.entries(scenarios)) {
  const { decision, reason } = judge(s);
  console.log(`${name.padEnd(17)} ${decision.padEnd(9)} ${reason}`);
}
// ambient incident  promote   errors z=0.8, p99 ratio 1.02
// real regression   rollback  errors 0.80% vs 0.20% (z=8.0)
// slower version    rollback  p99 260 ms vs 180 ms (1.44×)
// too early         wait      need 2000 requests per group
// broken build      rollback  canary error rate 30.0% above hard limit
```

During the ambient incident both versions run at about 3% errors; a 1%-threshold rule would have rolled the canary back for
an outage it didn't cause, while the comparison correctly finds no difference. The real regression is "only" 0.8% errors —
under most fixed thresholds — but four times the baseline, with overwhelming statistical significance. (The broken build's
latency is *lower* because it fails fast — another reason never to judge on latency alone.)

The decision to defend is **comparing against a concurrently running baseline rather than against the production fleet's
historical numbers**. Comparing the canary with "how the service usually behaves" seems simpler — no extra baseline
instances — but usual behaviour is a moving target: traffic mix changes by hour and weekday, upstream dependencies have bad
days, and the large old fleet differs from a freshly started canary in cache warmth and connection state. Any of these makes
the canary look worse or better for reasons unrelated to the code. A small baseline of the old version, started at the same
time and given the same traffic slice, differs from the canary in exactly one thing — the version — so whatever difference
remains is attributable to the change. That attribution is the whole point of a canary.

---

## 31.10 Questions

**1.** MCQ — Which deployment strategy limits a bad release's impact to a small share of traffic and compares it with the old version? a) Recreate b) Blue/green c) Canary d) Rolling without analysis
<details><summary>Answer</summary>**c)** A canary takes a small slice first, judged against a baseline.</details>

**2.** MCQ — What does a feature flag decouple? a) Code from tests b) Deploying code from releasing behaviour c) Databases from services d) Frontend from backend
<details><summary>Answer</summary>**b)** Code ships turned off and is enabled independently, gradually, and reversibly.</details>

**3.** MCQ — Why must percentage rollouts use stable hashing of the user? a) For security b) So users keep the same variant as the percentage grows, avoiding flicker and polluted metrics c) To reduce latency d) To save memory
<details><summary>Answer</summary>**b)** Each user's bucket is fixed, so 10% is a subset of 50%.</details>

**4.** MCQ — During a rolling deploy, which schema change is unsafe? a) Adding a nullable column b) Creating an index concurrently c) Renaming a column the old code still reads d) Adding a new table
<details><summary>Answer</summary>**c)** Old instances still running will fail; use expand–migrate–contract.</details>

**5.** MCQ — What's the main advantage of blue/green over rolling deployment? a) Lower cost b) Instant rollback by switching traffic back c) No need for compatibility d) Gradual exposure
<details><summary>Answer</summary>**b)** The old environment stays intact; switching back is immediate. It exposes everyone at the switch, though.</details>

**6.** MCQ — In shadow (dark launch) testing, what must be suppressed? a) Logging b) Side effects such as writes, emails, and charges c) Metrics d) Traffic copies
<details><summary>Answer</summary>**b)** The shadow version sees real traffic, so its side effects must not reach the real world.</details>

**7.** Short — Why are small, frequent releases safer than large, infrequent ones?
<details><summary>Answer</summary>Each release carries fewer changes, so failures are easier to attribute, smaller in impact, and quicker to roll back without undoing unrelated work; frequent practice also keeps the pipeline and rollback paths working.</details>

**8.** Short — Why judge a canary against a concurrent baseline rather than fixed thresholds?
<details><summary>Answer</summary>Ambient conditions — upstream incidents, traffic mix, time of day — affect both groups equally and cancel out, leaving only the version's effect. Fixed thresholds roll back healthy canaries during incidents and pass significant relative regressions.</details>

**9.** Short — What is flag debt, and how do you control it?
<details><summary>Answer</summary>Accumulated flags that are fully rolled out (or abandoned) but still in code, doubling code paths and leaving untested branches. Give each flag an owner and expiry, remove release flags soon after 100%, and fail CI or alert on stale flags.</details>

**10.** Short — Why should migrations run separately from deploys?
<details><summary>Answer</summary>So a failed or slow migration doesn't leave the application half-deployed (or vice versa), each can be retried or rolled back independently, and the schema is always compatible with whichever code version is running.</details>

**11.** Short — Why is "roll back first, diagnose later" the default?
<details><summary>Answer</summary>Rollback restores a known-good state quickly, while diagnosis and a forward fix take unknown time and introduce a new untested change under pressure. Rolling forward is the exception — when rollback is unsafe or a trivial, tested fix is ready.</details>

**12.** Output — Using §31.9's `judge`, what's the decision for baseline `{ requests: 40_000, errors: 400, p99Ms: 200 }` and canary `{ requests: 4_000, errors: 44, p99Ms: 210 }`?
<details><summary>Answer</summary>**promote.** Error rates are 1.0% vs 1.1%, z ≈ 0.6 (below 3), and the p99 ratio is 1.05 (below 1.2).</details>

**13.** Output — Using §31.4's `evaluate`, a flag has `killed: false`, `percent: 100`, and a rule `{ attribute: 'country', values: ['FR'], enabled: false }`. What does it return for `{ userId: 'u9', country: 'FR' }`?
<details><summary>Answer</summary>**`false`.** Rules are checked before the percentage, so the country rule disables it even at 100%.</details>

**14.** Output — A canary takes 2% of 10,000 requests/s, and the judge needs 2,000 canary requests. How long until the first judgement?
<details><summary>Answer</summary>**10 seconds.** 2% is 200 requests/s; 2,000 / 200 = 10 s. (Real analysis usually waits longer, to cover warm-up and periodic behaviour.)</details>

**15.** Output — What does this print?
```js
const steps = [1, 5, 25, 50, 100], minutesEach = 10;
const exposure = steps.reduce((sum, pct) => sum + pct * minutesEach, 0); // percent-minutes of traffic on the new version
console.log(steps.length * minutesEach, exposure, (1 + 5) * minutesEach);
```
<details><summary>Answer</summary>`50 1810 60` — the full rollout takes 50 minutes; a bug caught at the 5% step has exposed only 60 percent-minutes of traffic, against 1,000 for the same 10 minutes at 100%.</details>

**16.** Debug — A deploy renamed `user_name` to `username` in one migration plus a code change. During the rolling deploy, half the requests failed for eight minutes. Why, and how should it have been done?
<details><summary>Answer</summary>Old instances still queried `user_name`, which no longer existed. Use expand–migrate–contract: add `username`, write both, backfill, switch reads, stop writing the old column, and drop it in a later release.</details>

**17.** Debug — A kill switch was flipped during an incident, but the "off" path threw errors because a dependency it used had been removed months earlier. What process failure is this?
<details><summary>Answer</summary>Flag debt and an untested off path: the flag sat at 100% long after release, and nobody tested the disabled branch. Remove release flags promptly, keep kill switches as explicitly tested paths (run tests with flags off), and exercise them in game days.</details>

**18.** Debug — Every few weeks, a canary is automatically rolled back for a high error rate, yet investigations find nothing wrong with the new version; each time, an upstream provider was having an incident. What do you change?
<details><summary>Answer</summary>The canary is judged against fixed thresholds. Add a concurrently running baseline of the old version with the same traffic slice and judge on the difference (as in §31.9), keeping only a high absolute threshold for catastrophic failures.</details>

**19.** Debug — `CREATE INDEX` on a 400 GB table during business hours froze writes to the table for 25 minutes. What should have been done?
<details><summary>Answer</summary>`CREATE INDEX CONCURRENTLY` (PostgreSQL) builds without blocking writes; run it outside a transaction, with a lock timeout on any DDL, tested against a production-sized copy, and monitored for replication lag.</details>

**20.** Code — Write `nextStep(current, steps, judgement)` for a progressive rollout: on `promote` advance to the next percentage, on `wait` stay, on `rollback` return 0.
<details><summary>Answer</summary>

```js
function nextStep(current, steps, judgement) {
  if (judgement === 'rollback') return 0;                   // all traffic back to the old version
  if (judgement === 'wait') return current;
  const i = steps.indexOf(current);
  return i === -1 ? steps[0] : steps[Math.min(i + 1, steps.length - 1)];
}
console.log(nextStep(5, [1, 5, 25, 50, 100], 'promote'), nextStep(25, [1, 5, 25, 50, 100], 'rollback')); // 25 0
```
</details>

**21.** Code — Write `backfillBatches(client, { batchSize, pauseMs })` that repeatedly updates rows where `email_normalized IS NULL` in keyset order, committing each batch and pausing between them, until none remain.
<details><summary>Answer</summary>

```js
async function backfillBatches(client, { batchSize = 1_000, pauseMs = 200 } = {}) {
  let lastId = 0, total = 0;
  for (;;) {
    const { rows } = await client.query(
      `UPDATE users SET email_normalized = lower(email)
        WHERE id IN (SELECT id FROM users WHERE email_normalized IS NULL AND id > $1 ORDER BY id LIMIT $2)
        RETURNING id`, [lastId, batchSize]);                  // each statement commits on its own
    if (rows.length === 0) return total;
    total += rows.length;
    lastId = Math.max(...rows.map((r) => r.id));
    await new Promise((r) => setTimeout(r, pauseMs));       // throttle: let replicas and traffic breathe
  }
}
```
A production version also checks replication lag before each batch and stops if it exceeds a threshold.</details>

**22.** Code — Write `isStaleFlag(flag, { now, maxDaysAtFull })` returning `true` for flags at 100% (or 0%) with no change for more than `maxDaysAtFull` days, unless marked `permanent` (kill switches, ops toggles).
<details><summary>Answer</summary>

```js
function isStaleFlag(flag, { now = Date.now(), maxDaysAtFull = 30 } = {}) {
  if (flag.permanent) return false;                          // kill switches and operational toggles stay
  const settled = flag.percent === 100 || flag.percent === 0;
  const days = (now - Date.parse(flag.lastChangedAt)) / 86_400_000;
  return settled && days > maxDaysAtFull;
}
console.log(isStaleFlag({ percent: 100, lastChangedAt: '2026-08-01' }, { now: Date.parse('2026-11-15') })); // true
```
</details>

**23.** Scenario — Your team deploys weekly on Thursday evenings, with all changes from the week in one release, and has frequent long incidents afterwards. Leadership suggests deploying monthly to reduce incidents. What do you propose?
<details><summary>Answer</summary>The opposite: deploy smaller changes more often. Move to trunk-based development behind feature flags, deploy daily or on every merge, and add canary analysis against a baseline with automatic rollback, plus deploy markers on dashboards. Batching more changes makes each release riskier and harder to roll back; making each change small and observable is what reduces incidents.</details>

**24.** Scenario — You need to split a `customers.address` text column into structured fields across a 200-million-row table, used by four services, with zero downtime. Plan it.
<details><summary>Answer</summary>Expand: add nullable structured columns; deploy code that writes both forms (parsing on write); backfill existing rows in throttled keyset batches, watching replication lag; deploy readers that prefer structured fields with a fallback; migrate the four services one by one (each tolerating both shapes, with contract tests); stop writing the old column once all writers are updated; finally drop it after a safety period and a backup. Each step is a separate, reversible deploy, and none requires the others to be complete.</details>

---

## Chapter Summary

- Most outages follow changes; make every change — code, config, flags, schema, infrastructure — small, gradually exposed,
  observed, and reversible, rather than rare.
- Canary and progressive delivery bound the damage of undetected bugs and should be the default, rolled out in waves across
  cells and regions; blue/green and shadow traffic fit specific cases.
- Judge canaries against a concurrently running baseline with statistical tests, a minimum sample, and a hard limit for
  catastrophic failures — and roll back automatically.
- Feature flags separate deploying from releasing; use stable hashing for percentage rollouts, evaluate flags locally, roll
  flags out gradually, and remove them before they become debt.
- Change schemas with expand–migrate–contract so every state works with both code versions, run migrations separately from
  deploys, use non-blocking DDL with lock timeouts, and throttle backfills.
- Keep every interface compatible across mixed versions and rollback, roll back first and diagnose later, and treat
  irreversible changes as separate, reviewed events.

---

## Practice Exercises

1. Measure your team's last 20 deploys: size (lines, files, services), time to production, and how many caused incidents;
   look for a relationship between size and failures.
2. Set up a progressive-delivery tool (Argo Rollouts, Flagger, or your platform's equivalent) for a test service with an
   automated analysis step and automatic rollback.
3. Read your database's documentation on which DDL statements take exclusive locks or rewrite tables, and build a checklist
   for reviewing migrations.
4. Audit your feature flags: owner, age, percentage, last change; list the flags to remove and any kill switch whose off path
   is untested.
5. Rehearse a rollback of your last deploy in staging, timing each step, and remove one manual step from the process.

---

## Mini Project: A Progressive Delivery Controller

Build a small controller that rolls out a new version of a service behind a traffic-splitting proxy in steps, judges each step
against a baseline, and rolls back automatically.

- A proxy that splits traffic among stable, baseline (old version), and canary pools by configurable weights.
- Metrics collection per pool: request counts, errors, and latency histograms.
- The §31.9 judge at each step, with configurable step sizes, durations, and thresholds.
- Automatic rollback that sets canary weight to zero and records the reason; promotion that replaces stable with canary.
- Test scenarios: a healthy release, an error regression, a latency regression, an upstream incident affecting all pools, and
  a crash-on-start build — each producing the correct decision.

---

**Next:** [Chapter 32 — Multi-Region and Disaster Recovery →](32-multi-region-dr.md)
