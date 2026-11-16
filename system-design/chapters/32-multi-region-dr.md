# Chapter 32 — Multi-Region and Disaster Recovery

## Learning Objectives

- Decide whether a system needs multiple regions, and state what it costs in consistency, money, and complexity
- Define RPO and RTO for each part of a system and choose a DR strategy that meets them
- Design active-passive failover and active-active topologies, including how data is replicated and owned
- Build backups that survive deletion, ransomware, and operator error — and prove they restore
- Decide when failover should be automatic, and test it before you need it

---

## 32.1 Why Multiple Regions, and What It Costs

A single region with several availability zones survives the failures Chapter 4 called common — a host, a rack, a
zone. It doesn't survive a regional outage: a provider's control-plane failure, a bad regional configuration push, a
network event that isolates the region, a natural disaster. Those are rare, but not rare enough to ignore for systems
where hours of downtime are unacceptable. Multiple regions also serve other goals:

| Goal | What it requires |
|---|---|
| **Survive regional failure** | A second region that can take over — with data — within the RTO |
| **Lower latency for distant users** | Serving users from a nearby region (Chapter 2's 150 ms trans-Pacific round trips) |
| **Data residency** | Keeping certain users' data in certain jurisdictions (§32.7) |
| **Scale beyond one region** | Rare — usually a provider capacity concern for very large systems |

Each goal leads to a different design, so name which one you're solving. A second region for disaster recovery can
sit mostly idle; a second region for latency must serve live traffic, which means writes, which means the replication
and conflict problems of Chapters 13 and 20.

The costs are significant and easy to underestimate:

- **Data**: cross-region replication is asynchronous in practice (synchronous would add 60–150 ms to every write), so
  failover loses recent writes, and active-active writes need conflict avoidance.
- **Money**: duplicate infrastructure, plus cross-region data transfer billed per GB.
- **Complexity**: global routing, per-region deployments and configuration, region-aware data models, and failover
  procedures that must be tested.
- **Correlated failure**: global dependencies — DNS, identity provider, a global configuration service, the deploy
  pipeline — can still take down every region at once. Multi-region doesn't help against what's shared.

> 💼 **Interview** — "How would you make this multi-region?" Start by asking why: disaster recovery, latency, or
> residency. Then state the RPO and RTO, choose active-passive or active-active accordingly, and address data
> explicitly — where writes go, how they replicate, what's lost on failover, and how conflicts are avoided.

---

## 32.2 RPO, RTO, and DR Strategies

Two numbers define recovery requirements:

- **RPO (recovery point objective)** — how much data you can afford to lose, measured in time: "at most 5 seconds of
  writes". Set by replication lag (or backup frequency).
- **RTO (recovery time objective)** — how long until service is restored: "within 15 minutes". Set by how much of the
  standby must be built, started, or warmed during recovery.

They differ by component and should be set per data class and per service, not once for the company: the payment
ledger may need RPO ≈ 0, while the recommendations cache can be rebuilt from scratch. Four classic strategies span the
range:

| Strategy | Standby region | Typical RPO | Typical RTO | Relative cost |
|---|---|---|---|---|
| **Backup and restore** | Nothing running; backups copied there | Hours (backup interval) | Hours to a day | Lowest |
| **Pilot light** | Data replicating; compute minimal or off | Seconds to minutes | Tens of minutes to an hour | Low |
| **Warm standby** | Data replicating; a scaled-down full stack running | Seconds | Minutes | Medium |
| **Multi-site active-active** | Full capacity, serving traffic | Near zero to seconds | Seconds to a minute | Highest (≈2× or more) |

Making the numbers concrete for a service taking 2,000 writes per second:

```js
// What a regional failure costs under each DR strategy, for a service taking 2,000 writes/s
const strategies = [
  { name: 'backup & restore', rpoSec: 24 * 3600, rtoSec: 8 * 3600 },  // nightly backup, rebuild region
  { name: 'pilot light',      rpoSec: 60,        rtoSec: 3600 },      // async replica, scale up app tier
  { name: 'warm standby',     rpoSec: 5,         rtoSec: 15 * 60 },   // replica + running, scaled-down app
  { name: 'active-active',    rpoSec: 1,         rtoSec: 60 },        // traffic already served in both
];
const writesPerSec = 2_000;
for (const s of strategies) {
  const lost = s.rpoSec * writesPerSec, down = s.rtoSec / 60;
  console.log(`${s.name.padEnd(17)} up to ${lost.toLocaleString('en-US').padStart(11)} writes lost, ~${String(down).padStart(3)} min down`);
}
// backup & restore  up to 172,800,000 writes lost, ~480 min down
// pilot light       up to     120,000 writes lost, ~ 60 min down
// warm standby      up to      10,000 writes lost, ~ 15 min down
// active-active     up to       2,000 writes lost, ~  1 min down
```

These numbers should be put in front of the business owner. "We'd lose up to 10,000 orders and be down 15 minutes in
a regional disaster, for this cost" is a decision someone can make; "we're multi-region" is not.

> ⚠️ **Gotcha** — **RTO includes the human parts.** Detecting the outage, deciding to fail over, getting approval,
> running the runbook, and verifying the result often take longer than the technical steps. A warm standby that can be
> promoted in two minutes still has a 45-minute RTO if the decision takes 40. Measure RTO in drills, end to end.

---

## 32.3 Active-Passive

In **active-passive**, one region serves all traffic and the other stands by with replicated data. It's the simpler
and more common design, and it fits disaster recovery well.

```text
            global routing (DNS / anycast LB)
             │ 100%                    │ 0% (standby)
     ┌───────▼────────┐        ┌───────▼────────┐
     │ region A       │ async  │ region B       │
     │ app (full)     │ repl.  │ app (scaled    │
     │ DB primary ────┼───────▶│ down) DB replica│
     └────────────────┘        └────────────────┘
```

A failover to region B requires, in order:

1. **Decide** that region A is really down — from independent vantage points, not from inside A (§32.6).
2. **Fence region A** so its database can't keep accepting writes if it's partially alive (Chapter 21) — otherwise the
   regions diverge.
3. **Promote** B's database replica to primary, accepting the loss of anything not yet replicated (the RPO).
4. **Scale up** B's application tier — ideally already provisioned, because a regional outage is exactly when other
   customers are also trying to launch capacity in other regions (Chapter 4's static stability).
5. **Shift traffic** at the global routing layer; a global anycast load balancer moves traffic in seconds, DNS takes
   minutes (Chapter 6).
6. **Verify** with synthetic transactions, then communicate.

And afterwards, **failback** — returning to region A — is a second migration: A's old data has diverged (it may hold
writes that never replicated), so it must be rebuilt as a replica of B before traffic moves back. Plan and practise
failback as carefully as failover; many teams simply make B the new primary and rebuild A as the standby.

Two design choices make active-passive work:

- **Pre-provision enough standby capacity** to take full traffic, or at least to take it while scaling, without relying
  on launching instances during the incident.
- **Keep the standby continuously exercised**: run a share of real traffic (read-only, or a small set of users) through
  it, run synthetic checks, and deploy to it with every release. A standby that receives no traffic drifts — wrong
  configuration, missing secrets, expired certificates — and fails at the moment it's needed.

---

## 32.4 Active-Active

In **active-active**, every region serves live traffic, including writes. It improves latency (users write to a nearby
region) and RTO (traffic simply stops going to a failed region), but it must answer the question active-passive avoids:
what happens when two regions write the same data?

The robust answer is to **avoid concurrent writes to the same data** rather than resolve them (Chapter 13's
conflict-avoidance strategy):

- **Home regions.** Each user, tenant, or account has a home region that owns its writes. Requests are routed to the home
  region (by a global routing layer that knows the mapping, or forwarded by the receiving region). Data is effectively
  single-leader per record, with each region leading for its own users and replicating them to the others for reads and
  for failover.
- **Global data in one place.** Data that must be globally unique or consistent — usernames, email addresses, account
  balances shared across regions, inventory for a global stock pool — goes either to a single home region (every write for
  it crosses regions) or into a consensus-replicated database spanning regions (Chapter 21), accepting tens to hundreds
  of milliseconds per write.
- **Truly multi-writer data only where it merges.** Data that is genuinely written everywhere at once must use CRDTs or
  commutative updates (Chapters 20, 46), or accept last-write-wins and its losses explicitly.

```text
user in Lisbon (home: eu-west) ──▶ eu-west ── writes for EU users ──▶ replicate ──▶ us-east (read copy)
user in Ohio   (home: us-east) ──▶ us-east ── writes for US users ──▶ replicate ──▶ eu-west (read copy)
username claim (global)        ──▶ consensus-replicated store across regions (slow, rare, linearizable)
```

When a region fails, its users' home is moved to a surviving region, which promotes its replica of their data —
active-passive *per user*, with the same RPO trade-off.

**Cells** take this further (Chapter 4): instead of "a region", the unit is a cell — a full copy of the stack serving a
fixed slice of users, with several cells per region. A thin routing layer maps users to cells. Failures, bad deploys, and
noisy tenants are contained to one cell, and capacity grows by adding cells. Large SaaS providers increasingly build this
way because the blast radius of any single fault becomes a small, known fraction of customers.

| | Active-passive | Active-active (home regions) |
|---|---|---|
| Write latency for distant users | Cross-region round trip | Local |
| Conflicts | None | None, if every record has one home |
| RTO | Minutes (failover) | Seconds to minutes (re-home users) |
| RPO | Replication lag | Replication lag, per region |
| Cost | Standby can be smaller | Every region sized for its traffic plus failover headroom |
| Complexity | Moderate | High: global routing, per-user homes, global data |

---

## 32.5 Backups Done Right

Replication protects against losing a machine or a region. It doesn't protect against **deleting the wrong thing**:
a buggy migration, a script run against production, a ransomware attacker with stolen credentials — replication copies
the damage to every replica within seconds. Backups protect against those, which makes them a different control with
different requirements:

- **Point-in-time recovery (PITR).** Base snapshots plus continuous archiving of the write-ahead log (Chapter 10) let you
  restore to any second — "just before the bad migration at 14:03:12". Managed databases offer this; know its retention.
- **Logically separate.** Backups in a different account (or subscription/project), with different credentials, so the
  credentials that can delete production can't delete backups.
- **Immutable.** Object lock / write-once retention (Chapter 17) so backups can't be altered or deleted, even by
  administrators, until their retention expires.
- **The 3-2-1 rule.** At least three copies, on two kinds of storage, with one off-site (another region or provider).
- **Everything, not just databases.** Object storage buckets, configuration, secrets (escrowed safely), infrastructure-as-
  code, DNS zones, and the knowledge of how to rebuild — a database backup with no way to recreate the services around it
  isn't a recovery plan.
- **Restores tested, regularly.** A backup that has never been restored is a hope. Automate periodic restores into an
  isolated environment, verify the data (row counts, checksums, an application smoke test), and record how long it took —
  that's your real RTO for a data-loss event.

> ⚠️ **Gotcha** — **Restore time scales with size, and grows silently.** A 4 TB database that restores in two hours today
> may take nine hours in two years. Test restores on a schedule, alert when restore duration approaches the RTO, and plan
> partitioning or faster restore paths (restoring the most critical tables first) before the backup outgrows the objective.

---

## 32.6 Failover Decisions, Drills, and Shared Dependencies

**Who decides?** Automatic failover is fast but can fire on a false alarm — a monitoring system's own network problem —
and a regional failover is disruptive and costly to reverse. Common practice:

- Automate failover where it's routine, low-risk, and tested: within-region database failover, cell evacuation, routing
  around an unhealthy zone.
- For regional failover, automate the **detection and the procedure**, but require a human decision — unless the RTO is
  shorter than humans can respond, in which case the automation must be conservative: a quorum of independent external
  probes, a sustained failure, a healthy standby, and replication lag within RPO. The real-world example in §32.9
  implements those rules.
- Either way, **never judge a region's health only from inside that region.** A region that's partitioned from the
  internet may look perfectly healthy to monitoring running within it.

**Drill it.** Run regional failover drills on a schedule — evacuate a region in staging monthly and in production
periodically (or one cell at a time). Each drill finds something: an expired certificate, a hard-coded endpoint, a missing
secret in the standby, a runbook step that no longer exists, a capacity shortfall. Measure the end-to-end RTO each time.

**Find the shared dependencies.** Multi-region architectures still fail together when they share something. Audit for:

| Shared dependency | Mitigation |
|---|---|
| DNS provider | Secondary DNS provider; long-enough TTLs that a provider blip doesn't remove records |
| Identity provider / SSO | Cached sessions and tokens survive its outage; break-glass accounts for operators |
| Global configuration or feature-flag service | Local caches with last-known-good values (static stability) |
| Container registry, artifact store, CI/CD | Replicated per region; ability to deploy with the pipeline down |
| Certificates | Monitored expiry; automated renewal; no single global certificate on the critical path |
| A "global" database | Explicitly designed: consensus-replicated, or one region with documented failover |

---

## 32.7 Data Residency and Sovereignty

Laws and contracts increasingly require that some data stays in a jurisdiction — EU personal data in the EU, health data
in-country, a government customer's data in a sovereign cloud. Residency changes the architecture:

- **Pin data to regions by owner.** The home-region model (§32.4) is the natural fit: a user's or tenant's data is stored
  in, and replicated only within, their permitted regions.
- **Restrict replication and backups.** DR copies must also stay in-jurisdiction — a second EU region, not a US one —
  which may mean a weaker DR posture for some data, chosen knowingly.
- **Watch the side channels.** Logs, traces, analytics pipelines, support tools, search indexes, and caches all copy data.
  Residency applies to them too, which often means regional telemetry pipelines and regional analytics.
- **Route by residency, not just latency.** The global routing layer must send a user's requests to a region permitted to
  process their data — even if another is closer.

Residency is a legal requirement interpreted by lawyers; the engineering job is to make data's location a known,
enforced, auditable property of the system rather than an accident of where services happened to be deployed.

---

## 32.8 Common Mistakes and Best Practices

**Mistake: multi-region without stating RPO and RTO.** Cost and complexity are spent without knowing what's being bought.

**Mistake: synchronous cross-region replication on the write path.** Every commit pays a cross-region round trip; most
systems can't afford it.

**Mistake: an untested standby.** Configuration drift, expired certificates, and missing secrets surface during the
disaster.

**Mistake: relying on launching capacity during a regional outage.** Everyone else is launching too; pre-provision.

**Mistake: active-active writes to the same records.** Conflicts and last-write-wins losses follow; give each record a home.

**Mistake: replication treated as backup.** Deletions and corruption replicate instantly; backups must be point-in-time,
separate, and immutable.

**Mistake: judging region health from inside the region.** A partitioned region can look healthy to itself.

**Best practice: RPO and RTO per data class.** The ledger and the cache deserve different strategies and budgets.

**Best practice: drill failover and restores on a schedule, and measure them.** The measured RTO is the real one.

**Best practice: hunt for shared global dependencies.** DNS, identity, config, and pipelines can take every region down
together.

---

## 32.9 Real-World Example: A Conservative Failover Controller

Automating regional failover is safe only if the automation is harder to fool than a human. This controller fails over
only when a quorum of independent, external probes agrees the primary is down, the failure has persisted, the standby is
healthy, and replication lag is within the RPO; otherwise it waits — or, when the data loss would exceed the RPO, asks a
human to approve it. It never fails back automatically, and an operator can freeze it.

```js
// failover-controller.mjs — decide when to fail a region over, conservatively: a quorum of
// independent probes must agree, the failure must persist, the standby must be healthy, and
// the data-loss window (replication lag) must be within RPO — or a human must approve.

export class FailoverController {
  state = 'primary-active';
  #suspectSince = null;

  constructor({ quorum = 3, persistMs = 30_000, maxLagSec = 5, now = Date.now }) {
    Object.assign(this, { quorum, persistMs, maxLagSec, now });
  }

  // probes: { primary: [bool…], standby: [bool…] } from independent vantage points (other regions,
  // an external monitoring provider) — never only from inside the region being judged
  evaluate({ probes, lagSec, override = null }) {
    if (override === 'freeze') return this.#set('frozen', 'operator froze automation');
    if (this.state === 'standby-active' || this.state === 'frozen') return this.#set(this.state, 'no automatic failback');
    if (override === 'approve' && this.state === 'awaiting-approval') return this.#failover(`approved with ${lagSec}s lag`);

    const down = (results) => results.filter((ok) => !ok).length >= this.quorum;
    if (!down(probes.primary)) { this.#suspectSince = null; return this.#set('primary-active', 'primary healthy'); }

    this.#suspectSince ??= this.now();
    const forMs = this.now() - this.#suspectSince;
    if (forMs < this.persistMs) return this.#set('suspect', `primary down for ${forMs / 1000}s; waiting`);
    if (down(probes.standby)) return this.#set('suspect', 'standby also unhealthy: failing over would not help');
    if (lagSec > this.maxLagSec) {
      return this.#set('awaiting-approval', `lag ${lagSec}s exceeds RPO ${this.maxLagSec}s: a human must accept the loss`);
    }
    return this.#failover(`primary down ${forMs / 1000}s, lag ${lagSec}s`);
  }

  #failover(reason) {
    // In production: fence the old primary, promote the standby database, shift traffic at the
    // global load balancer, and page humans — in that order (Chapters 13, 21)
    return this.#set('standby-active', `FAILOVER: ${reason}`);
  }
  #set(state, reason) { this.state = state; return { state, reason }; }
}

// ── Demo: a regional outage, observed by four external probes ───────────────────────
let t = 0;
const run = (ctl, steps) => steps.forEach(([at, primary, lagSec, override]) => {
  t = at * 1000;
  const { state, reason } = ctl.evaluate({ probes: { primary, standby: [true, true, true, true] }, lagSec, override });
  console.log(`t=${String(at).padStart(3)}s  ${state.padEnd(18)} ${reason}`);
});
const ok = [true, true, true, true], blip = [false, true, true, true], out = [false, false, false, true];

console.log('— lag within RPO —');
run(new FailoverController({ now: () => t }), [[0, ok, 1], [10, blip, 1], [20, out, 2], [40, out, 2], [55, out, 3]]);
console.log('— replication lag too high —');
run(new FailoverController({ now: () => t }), [[0, out, 40], [35, out, 40], [60, out, 45, 'approve']]);
// — lag within RPO —
// t=  0s  primary-active     primary healthy
// t= 10s  primary-active     primary healthy
// t= 20s  suspect            primary down for 0s; waiting
// t= 40s  suspect            primary down for 20s; waiting
// t= 55s  standby-active     FAILOVER: primary down 35s, lag 3s
// — replication lag too high —
// t=  0s  suspect            primary down for 0s; waiting
// t= 35s  awaiting-approval  lag 40s exceeds RPO 5s: a human must accept the loss
// t= 60s  standby-active     FAILOVER: approved with 45s lag
```

A single failing probe at 10 s doesn't move the controller; a quorum failure starts a suspicion timer; failover happens
only after 30 seconds of sustained failure. In the second scenario, failing over automatically would silently lose 40
seconds of writes — eight times the agreed RPO — so the controller escalates instead of acting.

The decision to defend is **refusing to fail over automatically when replication lag exceeds the RPO**, even though the
primary region is clearly down and every minute of waiting extends the outage. Failing over is the instinctive response
to a dead region, but promoting a standby that's 40 seconds behind means permanently discarding 40 seconds of committed
writes — orders accepted, payments recorded, messages sent — and that's a business decision with legal and financial
weight, not an operational reflex. The primary region may also come back in five minutes with all its data intact. Within
the agreed RPO, the trade has been pre-approved, so automation may act; beyond it, only a human who can weigh "more
downtime" against "more data loss" for this specific incident should choose.

---

## 32.10 Questions

**1.** MCQ — What does RPO measure? a) How long recovery takes b) How much data, measured in time, may be lost c) The number of regions d) The cost of standby capacity
<details><summary>Answer</summary>**b)** RPO is the acceptable data-loss window; RTO is the acceptable downtime.</details>

**2.** MCQ — Which DR strategy keeps data replicating and a scaled-down full stack running in the standby region? a) Backup and restore b) Pilot light c) Warm standby d) Active-active
<details><summary>Answer</summary>**c)** Pilot light keeps data replicating with minimal or no compute; warm standby runs a reduced full stack.</details>

**3.** MCQ — In active-active with home regions, how are write conflicts avoided? a) Last write wins b) Each record's writes go to one home region c) Synchronous replication everywhere d) Locks across regions
<details><summary>Answer</summary>**b)** Each record is effectively single-leader; conflicts can only occur during re-homing.</details>

**4.** MCQ — What protects against a buggy migration that deletes production data? a) Synchronous replication b) Multi-region active-active c) Point-in-time recovery from separate, immutable backups d) A larger database
<details><summary>Answer</summary>**c)** Replication copies the deletion everywhere; only point-in-time backups restore the state before it.</details>

**5.** MCQ — Why should a region's health be judged from outside it? a) It's cheaper b) A region partitioned from the internet can look healthy to monitoring inside it c) External probes are faster d) Regulations require it
<details><summary>Answer</summary>**b)** Internal monitoring shares the region's view and its failures.</details>

**6.** MCQ — Which is a shared dependency that can take down all regions at once? a) A per-region database replica b) The DNS provider c) Regional load balancers d) Per-region object storage
<details><summary>Answer</summary>**b)** A global dependency is outside any one region's failure domain — and inside everyone's.</details>

**7.** Short — Why should RPO and RTO be set per data class rather than once for the system?
<details><summary>Answer</summary>Different data has very different costs of loss and downtime — a payment ledger versus a rebuildable cache — so a single target either overspends on low-value data or underprotects critical data.</details>

**8.** Short — Why does failback need as much planning as failover?
<details><summary>Answer</summary>The old primary's data has diverged — it may hold writes that never replicated — so it must be rebuilt as a replica of the new primary before traffic moves back, with its own risks and runbook.</details>

**9.** Short — Why must a standby region be continuously exercised?
<details><summary>Answer</summary>Unused infrastructure drifts: configuration, secrets, certificates, capacity, and deploys fall out of date, so the standby fails exactly when it's needed. Serving some traffic, deploying to it, and running synthetic checks keep it honest.</details>

**10.** Short — Name three properties backups need that replicas don't have.
<details><summary>Answer</summary>Any three of: point-in-time history (restore to before a mistake); logical separation (different account and credentials); immutability (retention locks); off-site copies (3-2-1); regular, tested restores.</details>

**11.** Short — How does data residency change a multi-region design?
<details><summary>Answer</summary>Users' data must be stored, replicated, backed up, logged, and processed only in permitted regions, so routing follows residency rather than latency alone, DR copies stay in-jurisdiction, and telemetry and analytics pipelines become regional.</details>

**12.** Output — A service takes 800 writes/s; cross-region replication lag averages 3 s and peaks at 20 s during busy periods. If the primary region fails at peak, roughly how many acknowledged writes are lost?
<details><summary>Answer</summary>**~16,000.** 800 × 20 — RPO must be judged at peak lag, not the average.</details>

**13.** Output — Using §32.9's controller with defaults, probes report the primary down by quorum at t = 0 s and t = 25 s with lag 2 s. What is the state at t = 25 s?
<details><summary>Answer</summary>**`suspect`.** The failure has persisted 25 s, less than the 30 s required.</details>

**14.** Output — Same controller: the primary is down by quorum continuously from t = 0, lag is 2 s, but at t = 40 s three of four probes also report the standby down. What happens?
<details><summary>Answer</summary>**It stays `suspect` ("standby also unhealthy").** Failing over to an unhealthy standby wouldn't help — and both regions failing together suggests the probes or a shared dependency may be the problem.</details>

**15.** Output — A database restore test takes 2 h for 3 TB. Data grows 4% per month and restore time scales linearly with size. After how many months does the restore exceed a 4-hour RTO?
<details><summary>Answer</summary>**18 months.** Size must double: 1.04ⁿ ≥ 2 gives n ≥ ln 2 / ln 1.04 ≈ 17.7, so the 18th month.</details>

**16.** Debug — During a regional outage, failover to the standby region took 3 hours instead of the planned 20 minutes: TLS certificates in the standby had expired, two services had hard-coded the primary region's endpoints, and the standby couldn't scale because instance launches were throttled. What process changes would have caught these?
<details><summary>Answer</summary>Regular failover drills that exercise the standby end to end; deploying every release to the standby; serving a share of live or synthetic traffic through it; certificate expiry monitoring; configuration audits for region-specific endpoints; and pre-provisioned capacity rather than launching during an incident.</details>

**17.** Debug — After an attacker obtained an administrator's cloud credentials, they deleted the production database and its automated backups, which lived in the same account. What should the backup design have been?
<details><summary>Answer</summary>Backups copied to a separate account with separate credentials, protected by object-lock retention so even administrators can't delete them early, with point-in-time recovery and at least one off-site copy — and regular restore tests from that isolated copy.</details>

**18.** Debug — A team's active-active deployment occasionally shows users "losing" profile changes made while travelling. Users are routed to the nearest region, and both regions accept writes to the same profile, replicated with last-write-wins. What's the fix?
<details><summary>Answer</summary>Concurrent writes to the same record in different regions conflict, and LWW discards some. Give each user a home region that owns their writes (route or forward writes there regardless of location), and keep other regions as read replicas for that user.</details>

**19.** Debug — Automated regional failover triggered at 03:00 and moved all traffic to the standby region — but the primary was healthy; the monitoring system, running inside the primary region, had lost its own network path. What changes prevent a repeat?
<details><summary>Answer</summary>Judge region health from a quorum of independent external vantage points (other regions, a third-party monitor), require the failure to persist, check the standby's health, and gate failover on replication lag — as §32.9's controller does — or keep a human decision for regional failover.</details>

**20.** Code — Write `meetsRpo({ lagSamplesSec, rpoSec, percentile })` returning whether the given percentile of observed replication lag is within the RPO.
<details><summary>Answer</summary>

```js
function meetsRpo({ lagSamplesSec, rpoSec, percentile = 99 }) {
  const s = [...lagSamplesSec].sort((a, b) => a - b);
  const p = s[Math.min(s.length - 1, Math.ceil((percentile / 100) * s.length) - 1)];
  return { lagAtPercentile: p, ok: p <= rpoSec };
}
console.log(meetsRpo({ lagSamplesSec: [1, 1, 2, 2, 3, 3, 4, 9, 15, 30], rpoSec: 5, percentile: 90 })); // { lagAtPercentile: 15, ok: false }
```
Judging RPO by average lag hides exactly the busy periods when a failure is most costly.</details>

**21.** Code — Write `homeRegion(userId, assignments, fallbackOrder, health)` returning the user's assigned home region, or the first healthy region in `fallbackOrder` when the home is marked down.
<details><summary>Answer</summary>

```js
function homeRegion(userId, assignments, fallbackOrder, health) {
  const home = assignments.get(userId);
  if (home && health[home] !== 'down') return home;
  // Re-home only to regions permitted for this user (residency) — here, the ordered fallback list
  return fallbackOrder.find((r) => health[r] !== 'down') ?? null;
}
const assignments = new Map([['u1', 'eu-west']]);
console.log(homeRegion('u1', assignments, ['eu-central', 'us-east'], { 'eu-west': 'down', 'eu-central': 'up' })); // eu-central
```
</details>

**22.** Code — Write `restoreDrill(restore, verify, { rtoMs, now })` that runs a restore and a verification, measures total duration, and reports whether it met the RTO.
<details><summary>Answer</summary>

```js
async function restoreDrill(restore, verify, { rtoMs, now = () => performance.now() }) {
  const start = now();
  const target = await restore();                 // e.g. restore last night's snapshot into an isolated env
  const checks = await verify(target);            // row counts, checksums, an application smoke test
  const durationMs = now() - start;
  return { durationMs, verified: checks.every((c) => c.ok), metRto: durationMs <= rtoMs, failed: checks.filter((c) => !c.ok) };
}
```
Schedule it, record results over time, and alert when duration trends toward the RTO.</details>

**23.** Scenario — A fintech in one cloud region asks for "five nines and zero data loss" for its payment ledger, and a reasonable budget. Design the DR posture.
<details><summary>Answer</summary>Zero data loss with cross-region survival needs synchronous replication across regions — a consensus-replicated database with replicas in three nearby regions, paying tens of milliseconds per commit — or accepting a small RPO with async replication. For the ledger, use a consensus-replicated store across three regions (RPO ≈ 0, RTO ≈ seconds) and keep it small and focused; for everything else (dashboards, notifications, analytics), use warm standby with seconds of RPO. Add PITR backups in a separate account with object lock, quarterly regional failover drills, and an honest statement to the business of the latency cost.</details>

**24.** Scenario — A global SaaS wants to expand into the EU with a requirement that EU customers' data never leaves the EU, while US customers stay in us-east. What changes?
<details><summary>Answer</summary>Move to a home-region model: each tenant is pinned to a jurisdiction and region, with routing that sends their requests there. EU tenants replicate and back up only to a second EU region (DR within the EU). Logs, traces, analytics, support tools, and search indexes for EU tenants become EU-regional. Global data (tenant directory, billing identity) is minimised and reviewed for what may cross borders. Operational access is audited by region. The tenant directory becomes a critical, highly available global dependency to design carefully.</details>

---

## Chapter Summary

- Multiple regions protect against regional failure, improve latency, and satisfy residency — at real costs in data
  consistency, money, complexity, and the shared global dependencies they don't remove.
- RPO (data loss) and RTO (downtime) per data class drive the choice between backup and restore, pilot light, warm
  standby, and active-active; put the concrete numbers in front of the business.
- Active-passive is simpler: fence, promote, shift traffic, verify — with pre-provisioned, continuously exercised standby
  capacity and a planned failback.
- Active-active avoids conflicts by giving each record a home region, puts global data in one place or in a consensus
  store, and contains failures further with cells.
- Backups protect against deletion and corruption that replication copies instantly: point-in-time, separate account,
  immutable, off-site, everything included — and restores tested on a schedule.
- Judge regions from outside, automate failover conservatively (or keep a human decision), drill it regularly, and enforce
  data residency as an auditable property of routing, storage, backups, and telemetry.

---

## Practice Exercises

1. Write RPO and RTO targets for five data classes in a system you know, and check which current mechanisms actually meet
   them at peak.
2. Run a restore of a production-sized database backup into an isolated environment, verify it, and record the end-to-end
   time against your RTO.
3. Read your cloud provider's documentation on cross-region replication for your database and object storage, and note the
   typical lag, how failover works, and what happens to un-replicated writes.
4. Audit a system for global shared dependencies (DNS, identity, config, registries, certificates) and write the mitigation
   for each.
5. Extend §32.9's controller with a check that the old primary has been fenced before promoting, and with a minimum interval
   between failovers to prevent flapping.

---

## Mini Project: A Two-Region Failover Lab

Build a small service in two simulated regions (containers or separate processes with injected latency) and practise the
full disaster-recovery lifecycle on it.

- A primary database with asynchronous replication to a standby, a service tier in each region, and a global routing layer.
- Measured replication lag under load, and a dashboard showing current RPO exposure.
- A failover runbook automated as scripts: decide (§32.9's controller with external probes), fence, promote, shift traffic,
  verify with synthetic transactions.
- A failback procedure that rebuilds the old primary as a replica of the new one before moving traffic back.
- A drill report: injected outage time, decision time, technical failover time, writes lost, and the gaps found.

---

**Next:** [Chapter 33 — Security Architecture →](33-security-architecture.md)
