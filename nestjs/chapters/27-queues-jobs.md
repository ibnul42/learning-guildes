# Chapter 27 — Queues and Background Jobs

## Learning Objectives

- Register BullMQ queues and write processors that survive restarts
- Configure retries, backoff, and dead-letter handling deliberately
- Make a job idempotent, and explain why at-least-once execution demands it
- Choose between a queue, a scheduled task, and an inline call
- Diagnose a queue that is growing faster than it drains

---

## 27.1 What Belongs in a Queue

Move work off the request path when it is slow, unreliable, or unnecessary for the response:

| Work | Queue it? | Why |
|---|---|---|
| Send a welcome email | yes | slow, unreliable, not needed for the 201 |
| Generate a PDF report | yes | seconds of CPU |
| Resize an uploaded image | yes | CPU-bound, blocks the event loop |
| Charge a card | **no** | the user needs the outcome now |
| Validate a password | no | the response depends on it |
| Update a search index | yes | eventual consistency is fine |

The test is whether the caller needs the result. A queue converts a synchronous failure into a retryable one, which is exactly right for an email and exactly wrong for a payment the user is waiting on.

```bash
npm i @nestjs/bullmq bullmq
```

BullMQ requires Redis, and the Redis instance matters: it must be persistent, not the same ephemeral cache you use for §20 caching. A cache flush that also drops your job queue loses work.

---

## 27.2 Queues and Processors

```ts
// src/queues/queues.module.ts
@Module({
  imports: [
    BullModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        connection: { url: config.getOrThrow('REDIS_QUEUE_URL') },
        defaultJobOptions: {
          attempts: 3,
          backoff: { type: 'exponential', delay: 2_000 },
          // Without these two, completed and failed jobs accumulate in Redis
          // forever and eventually exhaust its memory.
          removeOnComplete: { age: 3_600, count: 1_000 },
          removeOnFail: { age: 24 * 3_600 },
        },
      }),
    }),
    BullModule.registerQueue({ name: 'emails' }),
  ],
})
export class QueuesModule {}
```

Producing a job:

```ts
@Injectable()
export class NotificationsService {
  constructor(@InjectQueue('emails') private readonly queue: Queue) {}

  async sendWelcome(user: User): Promise<void> {
    await this.queue.add(
      'welcome',
      { userId: user.id, email: user.email },
      {
        // A deterministic job id makes enqueueing idempotent: a retried HTTP
        // request cannot produce two welcome emails.
        jobId: `welcome:${user.id}`,
        delay: 5_000,
      },
    );
  }
}
```

Consuming:

```ts
@Processor('emails', { concurrency: 5 })
export class EmailProcessor extends WorkerHost {
  private readonly logger = new Logger(EmailProcessor.name);

  constructor(private readonly mail: MailService) { super(); }

  async process(job: Job<WelcomeJobData>): Promise<void> {
    switch (job.name) {
      case 'welcome':
        await this.mail.sendWelcome(job.data.email);
        return;
      default:
        // An unknown job name must fail loudly, not silently succeed —
        // otherwise a renamed job is dropped with no trace.
        throw new Error(`Unknown job name: ${job.name}`);
    }
  }

  @OnWorkerEvent('failed')
  onFailed(job: Job, err: Error): void {
    this.logger.error(`Job ${job.id} failed on attempt ${job.attemptsMade}`, err);
  }
}
```

> ⚠️ **Gotcha** — **Never put an entity or a large payload in job data.** It is serialised to JSON and stored in Redis, so a stale copy is processed minutes later, memory balloons, and any secret in the object is now sitting in Redis. Store an id and re-read inside the processor.

---

## 27.3 Retries and Failure

```ts
await this.queue.add('report', data, {
  attempts: 5,
  backoff: { type: 'exponential', delay: 1_000 },   // 1s, 2s, 4s, 8s, 16s
});
```

**Not every failure should be retried.** A malformed payload will fail identically five times; retrying costs five times the work for the same outcome. BullMQ's `UnrecoverableError` stops retries immediately:

```ts
async process(job: Job): Promise<void> {
  const user = await this.users.findOne(job.data.userId);

  // Permanent: the user is gone and will not come back. Retrying is pure waste.
  if (!user) throw new UnrecoverableError(`User ${job.data.userId} no longer exists`);

  try {
    await this.mail.send(user.email, job.data.template);
  } catch (err) {
    // Transient: the provider is down. Let it retry with backoff.
    if (isTransient(err)) throw err;
    throw new UnrecoverableError(`Permanent send failure: ${(err as Error).message}`);
  }
}
```

Classifying failures is the difference between a queue that recovers and one that burns capacity retrying impossible work.

**Jobs that exhaust their attempts land in the failed set.** That is a dead-letter queue only if something looks at it — an alert on failed-job count is the minimum, and a way to inspect and replay is what makes it useful.

---

## 27.4 Idempotency

BullMQ executes **at least once**. A worker that crashes after doing the work but before marking the job complete will have the job redelivered when its lock expires. Every processor must tolerate that.

Two mechanisms, and the second is the real one:

**A deterministic `jobId`** prevents duplicate *enqueueing*. Adding a job with an existing id is a no-op, so a retried API call does not create a second job.

**An idempotency check inside the transaction** prevents duplicate *effects*, which is the case `jobId` cannot cover, because redelivery reuses the same job:

```ts
async process(job: Job<ChargeJobData>): Promise<void> {
  await this.dataSource.transaction(async (manager) => {
    // The unique constraint is the guarantee. A redelivery hits it and the
    // transaction rolls back rather than charging twice.
    await manager.insert(ProcessedJob, { jobId: job.id! });
    await manager.save(Charge, Charge.from(job.data));
  });
}
```

This is the same pattern as [Chapter 26](26-microservices.md), for the same reason: you cannot make delivery exactly-once, so you make effects idempotent, inside the transaction that produces them.

> 💼 **Interview** — "How do you make a background job safe to retry?" The weak answer is "check if it already ran". The strong one names where the check lives — inside the transaction, enforced by a unique constraint — and explains that a check outside the transaction has a window in which two workers both pass it.

---

## 27.5 Operating a Queue

**Concurrency is per worker process.** `concurrency: 5` with three replicas means fifteen concurrent jobs. Size it against what the downstream can absorb, not against what the worker can run.

**A separate worker process is the usual production shape.** Running processors inside the API means CPU-heavy jobs block the event loop and degrade request latency (→ [Ch 34](34-performance-scaling.md)):

```ts
// worker.ts — same modules, no HTTP server
const app = await NestFactory.createApplicationContext(WorkerModule);
await app.init();
```

**Graceful shutdown matters more here than for HTTP.** A worker killed mid-job leaves it locked until the lock expires, delaying retry. `enableShutdownHooks()` plus BullMQ's own close lets in-flight jobs finish (→ [Ch 9](09-scopes-lifecycle.md)).

**The metrics that matter** are queue depth, oldest waiting job age, and failure rate. Depth alone is misleading — a queue of 10,000 draining in a minute is healthy, and a queue of 50 whose oldest job is an hour old is stuck.

**Rate limiting** protects a downstream that cannot take your throughput:

```ts
BullModule.registerQueue({
  name: 'emails',
  limiter: { max: 100, duration: 60_000 },   // 100 per minute, provider's limit
});
```

---

## 27.6 Common Mistakes and Best Practices

**Mistake: putting entities or secrets in job data.** They are serialised into Redis, go stale, and leak.

**Mistake: assuming exactly-once execution.** A worker crash after the work and before the ack redelivers the job.

**Mistake: retrying every failure.** A malformed payload fails identically every time and burns the whole attempt budget.

**Mistake: no `removeOnComplete`/`removeOnFail`.** Finished jobs accumulate in Redis until it runs out of memory.

**Mistake: an idempotency check outside the transaction.** Two workers can both pass it before either commits.

**Mistake: running processors in the API process.** CPU-bound jobs block the event loop and slow every request.

**Mistake: alerting on queue depth alone.** Oldest-job age is what distinguishes a busy queue from a stuck one.

**Best practice: enqueue ids, re-read in the processor.** The job sees current data and the payload stays small.

**Best practice: use a deterministic `jobId` for naturally-unique work.** It makes enqueueing idempotent for free.

**Best practice: use a dedicated Redis for queues.** Sharing with a cache means a cache flush destroys pending work.

---

## 27.7 Real-World Example: A Report Generator That Cannot Duplicate Work

A CPU-heavy job with classified failures, in-transaction idempotency, progress reporting, and cleanup that runs on shutdown.

```ts
// src/reports/report.processor.ts
import { Processor, WorkerHost, OnWorkerEvent } from '@nestjs/bullmq';
import { Job, UnrecoverableError } from 'bullmq';

interface ReportJobData {
  reportId: string;                 // an id, never the report itself
  requestedBy: string;
}

@Processor('reports', {
  concurrency: 2,                   // CPU-bound: more than cores helps nothing
  lockDuration: 300_000,            // must exceed the longest plausible run,
})                                  // or the lock expires mid-job and it is redelivered
export class ReportProcessor extends WorkerHost {
  private readonly logger = new Logger(ReportProcessor.name);

  constructor(
    private readonly reports: ReportsService,
    private readonly storage: StorageService,
    private readonly dataSource: DataSource,
    private readonly notifications: NotificationsService,
  ) { super(); }

  async process(job: Job<ReportJobData>): Promise<{ url: string }> {
    const { reportId, requestedBy } = job.data;

    // Re-read rather than trusting the payload: the job may have waited in
    // the queue for minutes and the request could have been cancelled.
    const request = await this.reports.findRequest(reportId);
    if (!request) {
      throw new UnrecoverableError(`Report request ${reportId} no longer exists`);
    }
    if (request.status === 'cancelled') {
      throw new UnrecoverableError(`Report ${reportId} was cancelled`);
    }

    // Cheap short-circuit for a redelivery that already succeeded. This is an
    // optimisation; the constraint below is the actual guarantee.
    if (request.status === 'completed' && request.fileUrl) {
      this.logger.debug(`Report ${reportId} already complete, skipping`);
      return { url: request.fileUrl };
    }

    await job.updateProgress(10);

    let rows: ReportRow[];
    try {
      rows = await this.reports.gather(request);
      await job.updateProgress(60);
    } catch (err) {
      // Transient — the database was busy. Let BullMQ retry with backoff.
      if (this.isTransient(err)) throw err;
      throw new UnrecoverableError(`Query failed permanently: ${(err as Error).message}`);
    }

    const buffer = await this.reports.render(rows);
    await job.updateProgress(85);

    const url = await this.storage.upload(`reports/${reportId}.pdf`, buffer);

    await this.dataSource.transaction(async (manager) => {
      // The unique insert and the state change commit together. A redelivery
      // that reaches here hits the constraint and rolls back, so the report
      // is never marked complete twice or billed twice.
      await manager.insert(ProcessedJob, { jobId: job.id!, kind: 'report' });
      await manager.update(ReportRequest, { id: reportId }, {
        status: 'completed',
        fileUrl: url,
        completedAt: new Date(),
      });
    });

    await job.updateProgress(100);

    // Deliberately after the transaction and not awaited into it: a
    // notification failure must not roll back a completed report.
    void this.notifications
      .reportReady(requestedBy, url)
      .catch((err) => this.logger.error(`Notify failed for ${reportId}`, err));

    return { url };
  }

  @OnWorkerEvent('failed')
  async onFailed(job: Job<ReportJobData>, err: Error): Promise<void> {
    const exhausted = job.attemptsMade >= (job.opts.attempts ?? 1);
    if (!exhausted) return;

    // Terminal failure: record it so the user sees an error rather than a
    // request that stays "processing" forever.
    this.logger.error(`Report ${job.data.reportId} failed permanently`, err);
    await this.reports.markFailed(job.data.reportId, err.message);
  }

  private isTransient(err: unknown): boolean {
    const code = (err as { code?: string })?.code;
    return ['ETIMEDOUT', 'ECONNRESET', '40001', '40P01'].includes(code ?? '');
  }
}
```

```ts
// src/reports/reports.service.ts
@Injectable()
export class ReportsService {
  constructor(@InjectQueue('reports') private readonly queue: Queue) {}

  async request(userId: string, params: ReportParams): Promise<ReportRequest> {
    const request = await this.repo.save(ReportRequest.create(userId, params));

    await this.queue.add(
      'generate',
      { reportId: request.id, requestedBy: userId },   // ids only
      {
        jobId: `report:${request.id}`,    // enqueueing is idempotent
        attempts: 3,
        backoff: { type: 'exponential', delay: 5_000 },
        removeOnComplete: { age: 3_600 },
        removeOnFail: { age: 7 * 24 * 3_600 },   // keep failures for investigation
      },
    );
    return request;
  }
}
```

The decision worth defending is setting `lockDuration` to five minutes rather than leaving the default. BullMQ holds a lock while a job runs and renews it periodically; if a job's work blocks the event loop long enough that renewal cannot happen, the lock expires and the queue concludes the worker died — so it hands the same job to another worker while the first is still running it. For a CPU-bound render this is not hypothetical: synchronous PDF generation is precisely the kind of work that starves the timer that renews the lock. The result is two workers producing the same report simultaneously, which the in-transaction constraint catches but only after both have done the expensive work. Raising `lockDuration` above the longest plausible runtime avoids the duplicate entirely, and the real fix for genuinely long CPU work is to move it off the event loop — a worker thread or a child process — so renewal keeps happening. Any queue whose jobs occasionally get processed twice for no apparent reason should have its lock duration checked before anything else.

---

## 27.8 Questions

**1.** MCQ — BullMQ executes jobs: a) exactly once b) at most once c) at least once d) in order
<details><summary>Answer</summary>**c)** A crash after the work and before the ack causes redelivery.</details>

**2.** MCQ — Job data should contain: a) full entities b) ids c) secrets d) buffers
<details><summary>Answer</summary>**b)** Data is serialised into Redis, so entities go stale and secrets leak.</details>

**3.** MCQ — `UnrecoverableError` causes BullMQ to: a) retry immediately b) stop retrying c) requeue d) pause the queue
<details><summary>Answer</summary>**b)** It marks the job failed without consuming further attempts.</details>

**4.** MCQ — `concurrency: 5` with three replicas gives: a) 5 b) 8 c) 15 d) 3
<details><summary>Answer</summary>**c)** Concurrency is per worker process.</details>

**5.** MCQ — Without `removeOnComplete`, finished jobs: a) are deleted b) accumulate in Redis c) move to failed d) expire in an hour
<details><summary>Answer</summary>**b)** Until Redis runs out of memory.</details>

**6.** MCQ — A deterministic `jobId` makes which operation idempotent? a) processing b) enqueueing c) both d) neither
<details><summary>Answer</summary>**b)** Redelivery reuses the same job, so processing still needs its own guard.</details>

**7.** Short — What is the test for whether work belongs in a queue?
<details><summary>Answer</summary>Whether the caller needs the result. A queue converts a synchronous failure into a retryable one, which is right for an email and wrong for a payment the user is waiting on.</details>

**8.** Short — Why must an idempotency check be inside the transaction?
<details><summary>Answer</summary>A check outside it has a window where two workers both pass before either commits; inside, a unique constraint makes one of them roll back.</details>

**9.** Short — Why is queue depth a poor alert on its own?
<details><summary>Answer</summary>Depth without drain rate is meaningless — 10,000 jobs draining in a minute is healthy, and 50 jobs whose oldest is an hour old is stuck.</details>

**10.** Short — Why run processors in a separate process from the API?
<details><summary>Answer</summary>CPU-bound jobs block the event loop, so in-process workers degrade request latency for every user.</details>

**11.** Short — Why does a too-short `lockDuration` cause duplicate processing?
<details><summary>Answer</summary>If the lock expires because renewal was starved by blocking work, the queue assumes the worker died and hands the job to another while the first is still running.</details>

**12.** Output — A worker crashes after sending an email but before acknowledging. What happens?
<details><summary>Answer</summary>The job is redelivered when the lock expires and the email is sent again, unless the processor is idempotent.</details>

**13.** Output — `queue.add('welcome', data, { jobId: 'welcome:42' })` is called twice. How many jobs exist?
<details><summary>Answer</summary>One — adding with an existing id is a no-op.</details>

**14.** Output — A processor throws `UnrecoverableError` on attempt 1 of 5. How many attempts run?
<details><summary>Answer</summary>One. The remaining attempts are skipped and the job moves straight to failed.</details>

**15.** Output — A full `User` entity is enqueued and processed ten minutes later, after the user changed their email. Which address is used?
<details><summary>Answer</summary>The stale one from enqueue time — which is why processors should re-read by id.</details>

**16.** Debug — Redis memory grows steadily though the queue drains. Cause?
<details><summary>Answer</summary>No `removeOnComplete`/`removeOnFail`, so every finished job is retained indefinitely.</details>

**17.** Debug — Reports are occasionally generated twice with no error. What do you check first?
<details><summary>Answer</summary>`lockDuration` against actual job runtime. If the lock expires mid-job, a second worker picks it up while the first is still running.</details>

**18.** Debug — A queue burns through attempts on jobs that can never succeed. Fix?
<details><summary>Answer</summary>Classify failures: throw `UnrecoverableError` for permanent conditions such as a deleted referent or a malformed payload, and let only transient errors retry.</details>

**19.** Debug — A cache flush lost all pending jobs. What was wrong?
<details><summary>Answer</summary>The queue shares a Redis instance with the cache. Queues need their own persistent instance.</details>

**20.** Debug — API latency degraded after adding image processing. Cause?
<details><summary>Answer</summary>Processors run in the API process and CPU-bound work blocks the event loop. Move workers to a separate process.</details>

**21.** Code — Enqueue idempotently with retries and cleanup.
<details><summary>Answer</summary>

```ts
await this.queue.add('welcome', { userId: user.id }, {
  jobId: `welcome:${user.id}`,                       // no duplicate enqueue
  attempts: 3,
  backoff: { type: 'exponential', delay: 2_000 },
  removeOnComplete: { age: 3_600, count: 1_000 },
  removeOnFail: { age: 24 * 3_600 },
});
```
</details>

**22.** Code — Make a processor's effects idempotent.
<details><summary>Answer</summary>

```ts
async process(job: Job<ChargeData>): Promise<void> {
  await this.dataSource.transaction(async (manager) => {
    await manager.insert(ProcessedJob, { jobId: job.id! });   // unique constraint
    await manager.save(Charge, Charge.from(job.data));
  });
}
```
A redelivery violates the constraint and the whole transaction rolls back.
</details>

**23.** Code — Classify transient and permanent failures.
<details><summary>Answer</summary>

```ts
try {
  await this.provider.send(payload);
} catch (err) {
  const code = (err as { code?: string }).code;
  if (['ETIMEDOUT', 'ECONNRESET', '429'].includes(code ?? '')) throw err;   // retry
  throw new UnrecoverableError(`Permanent failure: ${(err as Error).message}`);
}
```
</details>

**24.** Scenario — A teammate proposes queueing the payment charge so checkout responds instantly. Evaluate it.
<details><summary>Answer</summary>It makes checkout fast and the product wrong. The user needs the outcome to know whether they bought anything, so queueing it means returning a success the system cannot yet honour, then reconciling by email when the card declines — which is a worse experience than a two-second wait and a clear error. It also multiplies the failure surface: a queued charge can be redelivered, so the idempotency work becomes mandatory rather than merely wise, and a stuck queue means money moving hours after the user acted. Charge synchronously with a tight timeout, and queue everything *after* the charge — receipt, fulfilment, analytics — which is where the latency actually is. The general rule stands: queue what the caller does not need the result of.</details>

**25.** Scenario — Your queue is growing faster than it drains, and adding workers has not helped. How do you diagnose it?
<details><summary>Answer</summary>Adding workers not helping is the diagnostic: it means the bottleneck is downstream rather than in worker capacity. Check whether a rate limiter is capping throughput deliberately, whether the jobs contend on the same database rows and are now serialising on locks, and whether the downstream provider is throttling you — in which case more workers just produce more 429s and more retries, making it worse. Look at oldest-job age and failure rate together: a high failure rate with retries means much of your throughput is reprocessing the same jobs, and classifying permanent failures will free most of it. If the arrival rate genuinely exceeds any achievable service rate, no amount of tuning fixes it and the answer is upstream — batching several jobs into one, or shedding lower-priority work into a separate queue that is allowed to lag.</details>

---

## Chapter Summary

- Queue work the caller does not need the result of; queueing something they are waiting on turns a clear failure into a silent one.
- Enqueue ids, never entities — job data is serialised into Redis, so payloads go stale, grow, and leak secrets.
- Execution is at least once, so a deterministic `jobId` makes enqueueing idempotent while only an in-transaction unique constraint makes the *effects* idempotent.
- Classify failures: retrying a permanently broken job burns the whole attempt budget for the same outcome, which is what `UnrecoverableError` exists to prevent.
- `removeOnComplete` and `removeOnFail` are not housekeeping — without them Redis fills with finished jobs until it stops accepting new ones.
- A `lockDuration` shorter than the real job runtime causes duplicate processing, and blocking CPU work is what starves the renewal that would have prevented it.

---

## Practice Exercises

1. Enqueue a full entity, change the underlying record, and observe the processor acting on stale data.
2. Kill a worker mid-job and watch the redelivery, then add in-transaction idempotency and repeat.
3. Set `lockDuration` below a job's runtime and observe two workers processing it simultaneously.
4. Fill a queue with permanently failing jobs and measure the wasted attempts, then add `UnrecoverableError`.
5. Run processors inside the API and measure request latency during CPU-heavy jobs, then move them out.

---

## Mini Project: A Job Pipeline With Operational Visibility

Build background processing you could run on call.

**Requirements:**

- A separate worker process sharing modules with the API but serving no HTTP.
- Deterministic job ids for enqueue idempotency and an in-transaction guard for effect idempotency.
- Failure classification with `UnrecoverableError`, plus retained failed jobs and a replay command.
- Metrics for depth, oldest-job age, throughput, and failure rate, with alerts on age rather than depth.
- Graceful shutdown that lets in-flight jobs finish, proven by a test that stops a worker mid-job without duplication.

---

**Next:** [Chapter 28 — CQRS and Event-Driven Design →](28-cqrs-events.md)
