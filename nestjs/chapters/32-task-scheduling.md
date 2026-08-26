# Chapter 32 — Task Scheduling

## Learning Objectives

- Schedule work with `@Cron`, `@Interval`, and `@Timeout`
- Explain why the same cron runs on every replica, and stop it
- Handle overlapping executions, missed runs, and time zones deliberately
- Choose between a scheduler, a queue, and a delayed job
- Make scheduled work observable, so a silent failure is not silent

---

## 32.1 The Three Decorators

```bash
npm i @nestjs/schedule
```

```ts
@Module({ imports: [ScheduleModule.forRoot()] })
export class AppModule {}
```

```ts
@Injectable()
export class ReportTasks {
  @Cron('0 3 * * *', { name: 'daily-report', timeZone: 'Europe/London' })
  async dailyReport(): Promise<void> {}

  @Interval('heartbeat', 30_000)          // every 30s after startup
  heartbeat(): void {}

  @Timeout('warmup', 5_000)               // once, 5s after startup
  warmCache(): void {}
}
```

`CronExpression` provides readable constants for common schedules:

```ts
@Cron(CronExpression.EVERY_DAY_AT_MIDNIGHT)
@Cron(CronExpression.EVERY_5_MINUTES)
```

**Always name a scheduled job.** The name is how `SchedulerRegistry` finds it to pause, resume, or remove it, and an unnamed job cannot be controlled at runtime:

```ts
constructor(private readonly registry: SchedulerRegistry) {}

pause(): void {
  this.registry.getCronJob('daily-report').stop();
}
```

**Specify a time zone explicitly.** Without one, the schedule follows the server's zone, so a job set for 03:00 runs at a different wall-clock time when the deployment region changes or when daylight saving shifts. `timeZone: 'UTC'` is the safest default unless the job genuinely tracks local business hours.

> ⚠️ **Gotcha** — During a daylight-saving transition a daily job scheduled for a local time inside the shifted hour either **runs twice or not at all**. If a job must run exactly once per day, schedule it in UTC, or make it idempotent and let the calendar do what it does.

---

## 32.2 Every Replica Runs Every Job

This is the defining problem of scheduling in a deployed application, and it is invisible in development.

`@Cron` is per-process. Three replicas means three executions at 03:00 — three reports emailed, three billing runs, three sets of charges. It works perfectly on a laptop and fails the moment the deployment scales.

Four ways to fix it, in increasing robustness:

**A dedicated scheduler process.** One replica runs the schedule module and the others do not. Simple, and it is a single point of failure with no automatic failover.

**A distributed lock.** Every instance tries to acquire a lock before running; one wins:

```ts
@Cron('0 3 * * *', { name: 'daily-report' })
async dailyReport(): Promise<void> {
  // NX = set only if absent, PX = auto-expire. The TTL must exceed the job's
  // runtime, or a second instance acquires the lock while the first is running.
  const acquired = await this.redis.set('lock:daily-report', this.instanceId, {
    NX: true,
    PX: 10 * 60 * 1000,
  });
  if (!acquired) return;

  try {
    await this.reports.generateDaily();
  } finally {
    // Release only if we still hold it — a job that overran its TTL must not
    // delete a lock another instance has since acquired.
    await this.releaseIfOwned('lock:daily-report', this.instanceId);
  }
}
```

**A database-backed claim.** A conditional update on a `scheduled_jobs` row gives the same guarantee with no extra infrastructure, and it is atomic:

```ts
const result = await this.repo.query(
  `UPDATE scheduled_jobs SET last_run_at = now(), locked_by = $1
   WHERE name = $2 AND (last_run_at IS NULL OR last_run_at < $3)`,
  [instanceId, name, windowStart],
);
if (result.rowCount === 0) return;      // another instance claimed it
```

**A queue with a repeatable job.** BullMQ's repeatable jobs are scheduled centrally in Redis and delivered to exactly one worker (→ [Ch 27](27-queues-jobs.md)):

```ts
await this.queue.add('daily-report', {}, {
  repeat: { pattern: '0 3 * * *', tz: 'UTC' },
  jobId: 'daily-report',
});
```

The queue is usually the right answer. It gives single execution, retries, history, and visibility, none of which `@Cron` provides.

---

## 32.3 Overlap, Drift, and Missed Runs

**Overlap.** A job scheduled every five minutes that takes six will start again before the previous run finishes, and the overlap compounds until the process falls over. `@Cron` does not prevent this:

```ts
private running = false;

@Cron(CronExpression.EVERY_5_MINUTES, { name: 'sync' })
async sync(): Promise<void> {
  if (this.running) {
    this.logger.warn('Previous sync still running, skipping');
    return;
  }
  this.running = true;
  try {
    await this.doSync();
  } finally {
    this.running = false;           // must be in finally, or one throw jams it forever
  }
}
```

The in-process flag is enough for one instance; across replicas the distributed lock covers both concerns at once.

**Missed runs.** If the process is down at 03:00, the run does not happen — the scheduler has no memory. Nothing catches up on restart, and nothing reports the miss. If a run matters, record expected executions and reconcile:

```ts
@Cron(CronExpression.EVERY_HOUR)
async catchUp(): Promise<void> {
  const missed = await this.repo.findWindowsWithoutRun(subDays(new Date(), 2));
  for (const window of missed) await this.process(window);
}
```

**Startup timing.** `@Interval` counts from process start, so with rolling restarts every replica has a different phase and jobs fire at unpredictable moments. `@Timeout` fires once per process — meaning once per *restart*, not once per deployment.

---

## 32.4 Scheduler, Queue, or Delayed Job

| Need | Use |
|---|---|
| Recurring, must run once cluster-wide | repeatable queue job |
| Recurring, harmless if it runs per-instance | `@Interval` (cache refresh, metrics) |
| One-off in the future ("remind me in 3 days") | queue job with `delay` |
| Immediately, off the request path | queue job |
| On startup, per process | `@Timeout` |

**A one-off future action should never be `setTimeout`.** It is lost on restart, it does not survive a deploy, and nothing records that it was supposed to happen:

```ts
// ❌ Gone on the next deploy, with no trace
setTimeout(() => this.sendReminder(id), 3 * 24 * 60 * 60 * 1000);

// ✅ Durable, visible, retryable
await this.queue.add('reminder', { id }, { delay: 3 * 24 * 60 * 60 * 1000, jobId: `reminder:${id}` });
```

---

## 32.5 Observability

A failing scheduled job is silent by default. No user reports it, no request fails, and the only symptom is that something did not happen — which nobody notices until a monthly report is missing.

Three things make it visible:

**Record every run.** Start time, end time, outcome, and rows affected, in a table you can query.

**Alert on absence, not failure.** "The daily report has not succeeded in 26 hours" catches the process being down, the lock being stuck, and the schedule being wrong. "The job threw" catches only one of those.

**Wrap the handler.** A job that throws inside a cron callback produces an unhandled rejection with no context:

```ts
@Cron('0 3 * * *', { name: 'daily-report' })
async dailyReport(): Promise<void> {
  const run = await this.runs.start('daily-report');
  try {
    const result = await this.reports.generateDaily();
    await this.runs.succeed(run.id, result.count);
  } catch (err) {
    await this.runs.fail(run.id, err as Error);
    this.logger.error('daily-report failed', err as Error);
    // Deliberately swallowed: rethrowing from a cron callback produces an
    // unhandled rejection that can terminate the process under Node's default.
  }
}
```

> 💼 **Interview** — "How do you make sure a cron job runs exactly once across a cluster?" The expected answer names a distributed lock or a queue with repeatable jobs. The follow-up that separates people who have operated one: what happens when the job outruns its lock TTL, and how you know a run was missed entirely.

---

## 32.6 Common Mistakes and Best Practices

**Mistake: assuming `@Cron` runs once cluster-wide.** It runs on every replica.

**Mistake: no time zone.** The schedule follows the server's zone and shifts with region or DST.

**Mistake: no overlap guard.** A job slower than its interval compounds until the process dies.

**Mistake: a lock TTL shorter than the job's runtime.** A second instance starts while the first is still running.

**Mistake: releasing a lock you no longer own.** An overrunning job deletes another instance's lock.

**Mistake: `setTimeout` for future actions.** Lost on every deploy, with no record.

**Mistake: rethrowing from a cron callback.** It becomes an unhandled rejection with no context.

**Best practice: prefer a repeatable queue job.** Single execution, retries, and history come free.

**Best practice: name every job.** Without a name it cannot be paused, resumed, or inspected at runtime.

**Best practice: alert on a job not having succeeded recently.** Absence covers every failure mode; an error alert covers one.

---

## 32.7 Real-World Example: A Cluster-Safe Scheduled Job

A nightly job that runs exactly once across replicas, cannot overlap itself, records every run, and reports when a run is missed entirely.

```ts
// src/scheduling/job-lock.service.ts
@Injectable()
export class JobLockService {
  private readonly instanceId = randomUUID();

  constructor(@Inject(REDIS_CLIENT) private readonly redis: RedisClientType) {}

  /** NX+PX is atomic: exactly one instance can succeed for a given key. */
  async acquire(name: string, ttlMs: number): Promise<string | null> {
    const token = `${this.instanceId}:${Date.now()}`;
    const ok = await this.redis.set(`joblock:${name}`, token, { NX: true, PX: ttlMs });
    return ok ? token : null;
  }

  /**
   * Compare-and-delete in a Lua script so the check and the delete are atomic.
   * A plain GET-then-DEL could delete a lock another instance acquired in the
   * window between the two calls — which is the classic distributed-lock bug.
   */
  async release(name: string, token: string): Promise<void> {
    await this.redis.eval(
      `if redis.call("get", KEYS[1]) == ARGV[1] then return redis.call("del", KEYS[1]) else return 0 end`,
      { keys: [`joblock:${name}`], arguments: [token] },
    );
  }

  /** Extends the lock for a job that is legitimately taking longer. */
  async renew(name: string, token: string, ttlMs: number): Promise<boolean> {
    const result = await this.redis.eval(
      `if redis.call("get", KEYS[1]) == ARGV[1] then return redis.call("pexpire", KEYS[1], ARGV[2]) else return 0 end`,
      { keys: [`joblock:${name}`], arguments: [token, String(ttlMs)] },
    );
    return result === 1;
  }
}
```

```ts
// src/scheduling/nightly-billing.task.ts
@Injectable()
export class NightlyBillingTask {
  private readonly logger = new Logger(NightlyBillingTask.name);
  private static readonly NAME = 'nightly-billing';
  private static readonly LOCK_TTL = 15 * 60 * 1000;

  constructor(
    private readonly locks: JobLockService,
    private readonly runs: JobRunRepository,
    private readonly billing: BillingService,
  ) {}

  // UTC explicitly: a local-time schedule runs twice or not at all across a
  // daylight-saving transition, and billing must do neither.
  @Cron('0 2 * * *', { name: NightlyBillingTask.NAME, timeZone: 'UTC' })
  async run(): Promise<void> {
    const token = await this.locks.acquire(NightlyBillingTask.NAME, NightlyBillingTask.LOCK_TTL);
    if (!token) {
      this.logger.debug('Another instance holds the lock; skipping');
      return;                        // not an error — exactly one instance should proceed
    }

    const run = await this.runs.start(NightlyBillingTask.NAME);

    // Renew while working, so a legitimately slow run does not lose its lock
    // and let a second instance start billing the same accounts.
    const renewal = setInterval(
      () => void this.locks.renew(NightlyBillingTask.NAME, token, NightlyBillingTask.LOCK_TTL),
      NightlyBillingTask.LOCK_TTL / 3,
    );

    try {
      const result = await this.billing.chargeDueSubscriptions();
      await this.runs.succeed(run.id, { charged: result.count });
      this.logger.log(`Billed ${result.count} subscriptions`);
    } catch (err) {
      await this.runs.fail(run.id, err as Error);
      this.logger.error('Nightly billing failed', err as Error);
      // Not rethrown: an unhandled rejection from a cron callback can take
      // the process down, and the failure is already recorded and alertable.
    } finally {
      clearInterval(renewal);
      await this.locks.release(NightlyBillingTask.NAME, token);
    }
  }
}
```

```ts
// src/scheduling/missed-run.monitor.ts
@Injectable()
export class MissedRunMonitor {
  private static readonly EXPECTED: Array<{ name: string; maxAgeMs: number }> = [
    { name: 'nightly-billing', maxAgeMs: 26 * 60 * 60 * 1000 },   // daily + 2h grace
    { name: 'hourly-sync', maxAgeMs: 90 * 60 * 1000 },
  ];

  /**
   * Alerts on ABSENCE rather than on failure. A job that threw is already
   * logged; a job that never ran — because the process was down, the lock was
   * stuck, or the schedule was wrong — produces no signal at all otherwise.
   */
  @Cron(CronExpression.EVERY_30_MINUTES, { name: 'missed-run-monitor', timeZone: 'UTC' })
  async check(): Promise<void> {
    for (const { name, maxAgeMs } of MissedRunMonitor.EXPECTED) {
      const last = await this.runs.lastSuccess(name);
      const age = last ? Date.now() - last.finishedAt.getTime() : Infinity;

      if (age > maxAgeMs) {
        this.alerts.raise({
          severity: 'high',
          title: `Scheduled job "${name}" has not succeeded recently`,
          detail: last
            ? `Last success ${Math.round(age / 3_600_000)}h ago`
            : 'No successful run on record',
        });
      }
    }
  }
}
```

The decision worth defending is renewing the lock on a timer rather than simply setting a generously long TTL. A long TTL looks simpler and it fails badly in the one case that matters: if the instance holding the lock dies mid-run, nothing releases it, and the job is blocked for the whole remaining TTL — so a fifteen-minute TTL chosen "with plenty of margin" becomes a fifteen-minute outage for a job that should have retried immediately on another instance. A short TTL with active renewal inverts that: a crashed instance stops renewing, the lock expires within a third of the TTL, and another replica picks it up on the next tick. It also means the TTL no longer has to be guessed against the worst-case runtime, which is the estimate people get wrong. The cost is the renewal timer and the atomic compare-and-set scripts, and the Lua is not optional — a `GET` followed by a `DEL` has a window in which you delete a lock somebody else now owns, which is exactly the failure the lock exists to prevent.

---

## 32.8 Questions

**1.** MCQ — With three replicas, `@Cron('0 3 * * *')` executes: a) once b) three times c) randomly d) on the leader
<details><summary>Answer</summary>**b)** The decorator is per-process.</details>

**2.** MCQ — Without `timeZone`, a cron follows: a) UTC b) the server's zone c) the user's zone d) GMT
<details><summary>Answer</summary>**b)** Which shifts with deployment region and daylight saving.</details>

**3.** MCQ — A job slower than its interval will: a) be skipped b) overlap itself c) throw d) reschedule
<details><summary>Answer</summary>**b)** `@Cron` does not guard against overlap.</details>

**4.** MCQ — `@Timeout` fires: a) once per deployment b) once per process start c) daily d) per request
<details><summary>Answer</summary>**b)** So a restart re-runs it.</details>

**5.** MCQ — A lock TTL shorter than the job's runtime causes: a) a deadlock b) a second instance to start c) an error d) a skipped run
<details><summary>Answer</summary>**b)** The lock expires while the first instance is still working.</details>

**6.** MCQ — A durable one-off future action should use: a) `setTimeout` b) `@Timeout` c) a delayed queue job d) `@Interval`
<details><summary>Answer</summary>**c)** It survives restarts and is visible and retryable.</details>

**7.** Short — Why is a repeatable queue job usually better than `@Cron`?
<details><summary>Answer</summary>It is scheduled centrally and delivered to exactly one worker, and it brings retries, history, and visibility that the scheduler does not provide.</details>

**8.** Short — Why must a lock release be a compare-and-delete?
<details><summary>Answer</summary>A plain delete can remove a lock another instance acquired after yours expired, which defeats the mutual exclusion entirely.</details>

**9.** Short — Why renew a lock instead of setting a long TTL?
<details><summary>Answer</summary>A long TTL blocks the job for its full duration if the holder crashes; renewal lets the lock expire quickly when the holder dies while still covering a slow run.</details>

**10.** Short — Why alert on absence rather than on failure?
<details><summary>Answer</summary>Absence catches every mode — process down, lock stuck, schedule wrong, job never registered — while a failure alert only catches a run that actually happened and threw.</details>

**11.** Short — What happens to a scheduled run if the process is down at that moment?
<details><summary>Answer</summary>Nothing — the scheduler has no memory, so the run is skipped silently and nothing catches up.</details>

**12.** Output — The overlap flag is set to `true` and the job throws before the `finally`. What happens to subsequent runs?
<details><summary>Answer</summary>Every one is skipped forever — the flag is never cleared. It must be reset in a `finally`.</details>

**13.** Output — A daily 02:30 local job during a spring-forward transition. How many times does it run?
<details><summary>Answer</summary>Zero — that wall-clock time does not exist that day. In autumn the same schedule runs twice.</details>

**14.** Output — Three replicas each acquire with `SET NX PX`. How many succeed?
<details><summary>Answer</summary>One. `NX` makes the set atomic and conditional.</details>

**15.** Output — A job overruns its TTL, then calls a plain `DEL` on release. What has it deleted?
<details><summary>Answer</summary>Possibly another instance's lock, letting a third instance start concurrently.</details>

**16.** Debug — Customers were charged three times on the first of the month. Cause?
<details><summary>Answer</summary>A `@Cron` billing job running on three replicas with no distributed lock.</details>

**17.** Debug — A nightly job stopped running after a region migration, with no errors. Why?
<details><summary>Answer</summary>No explicit `timeZone`, so the schedule follows the server's zone and now fires at a different wall-clock time than it did before the move.</details>

**18.** Debug — A five-minute sync silently stopped after one failure. What is the likely bug?
<details><summary>Answer</summary>An overlap flag set before the work and cleared after it, with no `finally` — the throw left it permanently `true`, so every later run short-circuits.</details>

**19.** Debug — Reminders scheduled with `setTimeout` disappear after every deploy. Fix?
<details><summary>Answer</summary>Use a delayed queue job with a deterministic job id, so the schedule lives in Redis rather than in process memory.</details>

**20.** Debug — A monthly report was missing and nobody noticed for three weeks. What monitoring was absent?
<details><summary>Answer</summary>An alert on the job not having succeeded within its expected window. Error alerting cannot fire for a run that never started.</details>

**21.** Code — Acquire and release a distributed lock safely.
<details><summary>Answer</summary>

```ts
const token = randomUUID();
const acquired = await redis.set(`joblock:${name}`, token, { NX: true, PX: ttlMs });
if (!acquired) return;

try {
  await work();
} finally {
  // Compare-and-delete: never remove a lock we no longer own.
  await redis.eval(
    `if redis.call("get", KEYS[1]) == ARGV[1] then return redis.call("del", KEYS[1]) else return 0 end`,
    { keys: [`joblock:${name}`], arguments: [token] },
  );
}
```
</details>

**22.** Code — Prevent a job from overlapping itself within one process.
<details><summary>Answer</summary>

```ts
private running = false;

@Cron(CronExpression.EVERY_5_MINUTES, { name: 'sync' })
async sync(): Promise<void> {
  if (this.running) return;
  this.running = true;
  try {
    await this.doSync();
  } finally {
    this.running = false;      // must be finally, or one throw jams it forever
  }
}
```
</details>

**23.** Code — Replace a cron with a repeatable queue job that runs once cluster-wide.
<details><summary>Answer</summary>

```ts
await this.queue.add(
  'daily-report',
  {},
  { repeat: { pattern: '0 3 * * *', tz: 'UTC' }, jobId: 'daily-report' },
);
```
The schedule lives in Redis, so exactly one worker receives each occurrence, with retries and history included.
</details>

**24.** Scenario — A teammate proposes running the scheduler on only one replica by setting an environment flag, to avoid the complexity of locking. Evaluate it.
<details><summary>Answer</summary>It is a reasonable first step and it trades one problem for another. Single execution is guaranteed while that instance is healthy, and there is no failover: if the scheduler pod is evicted, rescheduled, or crash-looping, every job silently stops and nothing runs until someone notices. It also creates an instance that is not interchangeable with the others, which complicates rolling deploys and autoscaling. If the jobs are non-critical it is a fine simplification, provided you alert on absence so a dead scheduler is visible within one interval. For anything that must happen — billing, expiry, reconciliation — use a repeatable queue job instead: it gives single execution *and* failover, and it is less work than the environment flag once you account for the monitoring the flag requires.</details>

**25.** Scenario — A daily job must process yesterday's data, and you need certainty it ran for every day even across outages. How do you design it?
<details><summary>Answer</summary>Stop thinking in terms of a schedule and start thinking in terms of windows. Create a row per processing window — one per day — with a status, and have the job claim and process unfinished windows rather than "process yesterday". Then a run that is missed entirely is not lost: the next execution finds two unprocessed windows and handles both, and a crash mid-window leaves it claimable again after its lease expires. Make the work idempotent per window so a redelivery is harmless, and drive the claim with a conditional update so concurrent replicas cannot take the same window. The scheduler then becomes a trigger rather than the source of truth, which is what makes catch-up automatic and lets you answer "did every day get processed" with a query instead of a log search.</details>

---

## Chapter Summary

- `@Cron`, `@Interval`, and `@Timeout` are per-process, so every replica runs every job — the defining scheduling bug, and invisible until you scale.
- Always name jobs and set an explicit time zone; an unnamed job cannot be controlled at runtime, and a local-time schedule runs twice or not at all across a DST transition.
- Single cluster-wide execution needs a distributed lock, a database claim, or a repeatable queue job — and the queue is usually the right answer because retries and history come with it.
- A lock must be released by compare-and-delete and renewed while working; a long TTL turns a crashed holder into an outage lasting the full TTL.
- Overlap guards must reset in a `finally`, or a single throw disables the job permanently.
- Scheduled failures are silent, so alert on a job not having succeeded recently rather than on it having thrown.

---

## Practice Exercises

1. Run three instances of an app with a `@Cron` job and count the executions.
2. Implement a distributed lock, kill the holder mid-run, and measure how long the job is blocked with and without renewal.
3. Set an overlap flag without a `finally`, force a throw, and confirm every later run is skipped.
4. Schedule a daily job in a local time zone and simulate a DST transition.
5. Replace a `@Cron` with a BullMQ repeatable job and compare what each gives you when the job fails.

---

## Mini Project: A Reliable Scheduler

Build scheduled work you could rely on for billing.

**Requirements:**

- Window-based processing: a row per period, claimed atomically, so missed runs are caught up automatically.
- Cluster-wide single execution via a renewable distributed lock with compare-and-delete release.
- Every run recorded with start, end, outcome, and counts, queryable by period.
- Absence alerting per job, with a configurable expected interval and grace period.
- A chaos test that kills the running instance mid-window and asserts another completes it exactly once.

---

**Next:** [Chapter 33 — Logging and Observability →](33-logging-observability.md)
