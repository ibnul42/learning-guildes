# Chapter 34 — Performance and Scaling

## Learning Objectives

- Measure before optimising, and know which measurement answers which question
- Identify and fix event-loop blocking, the dominant Node performance failure
- Decide when the Fastify adapter is worth the migration cost
- Scale horizontally with clustering and replicas, and know what breaks
- Tune connection pools, keep-alive, and payload size with evidence

---

## 34.1 Measure First

Almost every "NestJS is slow" report resolves to one of four causes, and none of them is the framework:

| Symptom | Usual cause | Chapter |
|---|---|---|
| Slow under any load | N+1 queries, missing index | [17](17-typeorm.md) |
| Fine at p50, terrible at p99 | event-loop blocking, GC pauses | §34.2 |
| Degrades as concurrency rises | pool exhaustion, unbounded queries | §34.5 |
| Slow only in production | no keep-alive, cold cache, network | §34.5 |

The framework overhead — the guard/interceptor/pipe pipeline — is measured in tens of microseconds. If your endpoint takes 400 ms, the pipeline is 0.01% of it, and optimising it is wasted effort.

Three tools answer three different questions:

```bash
# What is the throughput ceiling?
npx autocannon -c 100 -d 30 http://localhost:3000/tasks

# Where is CPU time spent?
node --cpu-prof --cpu-prof-dir=./profiles dist/main.js

# Is the event loop blocked, and by what?
npx clinic doctor -- node dist/main.js
```

`clinic doctor` is the one to reach for first: it diagnoses whether the bottleneck is the event loop, the garbage collector, I/O, or an external dependency, which tells you which of the other tools to use next.

> ⚠️ **Gotcha** — Benchmarking `npm run start:dev` measures the TypeScript watcher, not your application. Always benchmark a production build (`npm run build && node dist/main.js`) with `NODE_ENV=production`, which also disables development-only middleware and enables framework optimisations.

---

## 34.2 The Event Loop Is the Bottleneck

Node runs your JavaScript on one thread. Anything synchronous blocks every concurrent request, not just the one executing it — so a 200 ms synchronous operation adds up to 200 ms of latency to every request in flight.

The common blockers:

```ts
// ❌ Synchronous crypto — 100ms+ blocks everything
const hash = crypto.pbkdf2Sync(password, salt, 100_000, 64, 'sha512');

// ✅ Async version yields to the loop
const hash = await promisify(crypto.pbkdf2)(password, salt, 100_000, 64, 'sha512');
```

```ts
// ❌ JSON.stringify on a 50 MB object blocks for hundreds of milliseconds
return this.buildHugeReport();

// ✅ Stream it (→ Ch 31), or paginate it
```

```ts
// ❌ A tight loop over 500,000 rows
const totals = rows.reduce((acc, r) => expensiveTransform(acc, r), {});

// ✅ Push it to the database, or a worker thread, or a queue
```

Also blocking, and less obvious: `zlib` sync methods, `fs` sync methods, `Array.prototype.sort` on very large arrays, and complex regular expressions on long strings — a catastrophically backtracking regex is a denial-of-service vector, not just a slow function.

**Measure lag rather than guessing** (→ [Ch 33](33-logging-observability.md)):

```ts
import { monitorEventLoopDelay } from 'node:perf_hooks';

const histogram = monitorEventLoopDelay({ resolution: 20 });
histogram.enable();
setInterval(() => {
  // p99 above ~50ms means requests are queueing behind synchronous work.
  logger.log({ event: 'loop.lag', p99Ms: histogram.percentile(99) / 1e6 });
  histogram.reset();
}, 10_000);
```

For genuinely CPU-bound work, the options are a worker thread (`piscina` is the usual pool), a queue with a separate worker process (→ [Ch 27](27-queues-jobs.md)), or pushing the computation into the database. Adding replicas does not help: every replica has the same single thread and the same blocking work.

---

## 34.3 Fastify

The Fastify adapter is roughly twice as fast as Express on JSON-heavy workloads, mostly through faster routing and serialization.

```ts
const app = await NestFactory.create<NestFastifyApplication>(AppModule, new FastifyAdapter());
await app.listen(3000, '0.0.0.0');
```

**Whether it is worth it depends entirely on where your time goes.** If a request spends 5 ms in the framework and 200 ms in the database, halving the 5 ms changes nothing measurable. Fastify pays off when responses are large, throughput is high, and the handlers themselves are thin — a read-heavy API serving cached JSON is the ideal case.

The migration cost is real (→ [Ch 1](01-what-nestjs-is.md)): Express middleware mostly does not work, file uploads use `@fastify/multipart` instead of Multer, `@Res()` is a `FastifyReply`, and some ecosystem packages assume Express. Choose the platform on day one where possible.

---

## 34.4 Scaling Out

**Clustering** uses every core on one machine:

```ts
// src/cluster.ts
import cluster from 'node:cluster';
import { cpus } from 'node:os';

if (cluster.isPrimary) {
  for (let i = 0; i < cpus().length; i++) cluster.fork();
  cluster.on('exit', () => cluster.fork());     // replace a dead worker
} else {
  void bootstrap();
}
```

In a container platform, prefer **one process per container and more containers** — the orchestrator already handles restarts, rollout, and placement, and one process per container makes memory limits and metrics attributable.

Three things break when you go from one process to many, and all three have appeared in earlier chapters:

**In-memory state stops working.** Caches (→ [Ch 20](20-caching.md)), rate limits (→ [Ch 23](23-security.md)), WebSocket broadcasts (→ [Ch 25](25-websockets.md)), and presence all need a shared store.

**Scheduled jobs run everywhere** (→ [Ch 32](32-task-scheduling.md)).

**Connection pools multiply.** A pool of 20 across 10 replicas is 200 connections, and PostgreSQL's default limit is 100. The application starts failing to connect at exactly the moment you scale up to handle load:

```
max_connections ≥ replicas × pool_size + headroom
```

Beyond a handful of replicas, put PgBouncer in front rather than raising the database's limit — connections are expensive server-side, and a pooler multiplexes far more efficiently.

---

## 34.5 Cheap Wins

**HTTP keep-alive on outbound calls.** Without it, every request to a downstream service pays a fresh TCP and TLS handshake — often more time than the call itself:

```ts
HttpModule.register({
  httpAgent: new http.Agent({ keepAlive: true, maxSockets: 100 }),
  httpsAgent: new https.Agent({ keepAlive: true, maxSockets: 100 }),
  timeout: 5_000,                    // always bound it
});
```

**Compression**, for responses over a few kilobytes. It trades CPU for bandwidth, so it is a win on large JSON and a loss on small responses — set a threshold rather than compressing everything.

**Bounded payloads.** An endpoint with no page-size ceiling can be asked for the whole table, which is a denial-of-service vector shaped like a feature (→ [Ch 14](14-pipes-validation.md)).

**Select fewer columns.** `SELECT *` on a table with a large text column moves that column over the wire on every row for no reason.

**Cache deliberately** (→ [Ch 20](20-caching.md)), after fixing the query — a cache in front of an N+1 hides it until the cache goes cold, which it will do at the worst moment.

> 💼 **Interview** — "How would you speed up a slow NestJS endpoint?" Anyone who starts optimising before measuring has answered badly. The strong answer is: reproduce it, measure where the time goes, and only then act — and the expected first suspects are N+1 queries and event-loop blocking, not the framework.

---

## 34.6 Common Mistakes and Best Practices

**Mistake: optimising before measuring.** The framework is microseconds; your query is milliseconds.

**Mistake: benchmarking a dev build.** You measure the watcher and development middleware.

**Mistake: synchronous CPU work in a handler.** It blocks every concurrent request, and replicas do not help.

**Mistake: adding replicas for a CPU-bound endpoint.** Each replica has the same single thread.

**Mistake: ignoring pool × replica arithmetic.** Scaling up exhausts the database's connection limit.

**Mistake: no keep-alive on outbound HTTP.** Every call pays a handshake.

**Mistake: caching to hide a slow query.** The problem returns the moment the cache is cold.

**Best practice: measure event-loop lag continuously.** It is the earliest signal, and it rises before latency does.

**Best practice: bound every list endpoint.** An unbounded query is a denial-of-service vector.

**Best practice: choose the platform adapter on day one.** Migrating Express to Fastify late means rewriting middleware, uploads, and every raw response.

---

## 34.7 Real-World Example: Finding and Fixing a Real Bottleneck

A worked diagnosis: an endpoint at 2.4 s p99 taken to 90 ms, with each step justified by a measurement rather than a guess.

```ts
// ── The endpoint as found ────────────────────────────────────────────────
@Get('dashboard')
async dashboard(@CurrentUser('id') userId: string): Promise<DashboardDto> {
  const projects = await this.projects.findByOwner(userId);      // 1 query

  // Three compounding problems in six lines.
  const enriched = await Promise.all(
    projects.map(async (project) => ({
      ...project,
      tasks: await this.tasks.findByProject(project.id),         // N queries
      owner: await this.users.findOne(project.ownerId),          // N more
      // Synchronous work per project, on the event loop
      stats: this.computeStats(await this.tasks.findByProject(project.id)),
    })),
  );

  return { projects: enriched };
}
```

```
autocannon -c 50 -d 20 → p99 2,412 ms, 41 req/s
clinic doctor          → "event loop blocked"; queries dominate wall time
Query log              → 121 statements for one request
```

```ts
// ── Step 1: kill the N+1 (2,412 ms → 340 ms) ─────────────────────────────
@Get('dashboard')
async dashboard(@CurrentUser('id') userId: string): Promise<DashboardDto> {
  // One query with joins, selecting only the columns the DTO uses. The
  // `select` matters as much as the join: the description column is large
  // and was previously moved for every row.
  const projects = await this.projects.findByOwnerWithRelations(userId);
  return { projects: projects.map((p) => this.toDto(p)) };
}
```

```ts
// src/projects/project.repository.ts
async findByOwnerWithRelations(ownerId: string): Promise<Project[]> {
  return this.repo
    .createQueryBuilder('project')
    .leftJoinAndSelect('project.owner', 'owner')
    .leftJoinAndSelect('project.tasks', 'task')
    .select([
      'project.id', 'project.name', 'project.createdAt',
      'owner.id', 'owner.name',
      'task.id', 'task.status', 'task.estimateMinutes',
    ])
    .where('project.ownerId = :ownerId', { ownerId })
    .take(50)                       // bounded: an unbounded list is a DoS vector
    .getMany();
}
```

```
121 queries → 1.  p99 2,412 ms → 340 ms.
Remaining time: clinic now shows computeStats dominating CPU.
```

```ts
// ── Step 2: move the CPU work off the event loop (340 ms → 95 ms) ────────
@Injectable()
export class StatsService implements OnModuleDestroy {
  // A worker pool, not a queue: the result is needed for THIS response, so
  // it must be awaited — but it must not run on the main thread, where it
  // would add its full duration to every concurrent request.
  private readonly pool = new Piscina({
    filename: join(__dirname, 'stats.worker.js'),
    maxThreads: Math.max(1, cpus().length - 1),   // leave one for the event loop
  });

  async computeMany(batches: TaskBatch[]): Promise<Stats[]> {
    return Promise.all(batches.map((batch) => this.pool.run(batch)));
  }

  async onModuleDestroy(): Promise<void> {
    await this.pool.destroy();      // otherwise the process will not exit
  }
}
```

```ts
// ── Step 3: cache, only now that the query is correct (95 ms → 12 ms hit) ─
@Get('dashboard')
async dashboard(@CurrentUser('id') userId: string): Promise<DashboardDto> {
  // Caching is applied LAST, deliberately. Applied first it would have
  // masked the N+1 and the blocking work, and both would have returned at
  // full force the first time the cache went cold (→ Ch 20).
  return this.cache.getOrCompute(
    { namespace: 'dashboard', userId, tenantId },
    60_000,
    () => this.buildDashboard(userId),
  );
}
```

```ts
// ── Step 4: bound the outbound calls the dashboard makes ─────────────────
HttpModule.registerAsync({
  useFactory: () => ({
    // Without keep-alive every downstream call paid a fresh TLS handshake,
    // which was ~40ms of the remaining time.
    httpAgent: new http.Agent({ keepAlive: true, maxSockets: 100 }),
    httpsAgent: new https.Agent({ keepAlive: true, maxSockets: 100 }),
    timeout: 3_000,
    maxRedirects: 3,
  }),
});
```

```
Final: p99 90 ms (12 ms on a cache hit), 1,180 req/s.
Event-loop p99 lag: 118 ms → 4 ms.
```

The decision worth defending is applying the cache last rather than first. Caching the original endpoint would have taken p99 from 2,400 ms to about 15 ms on a hit — a far more impressive number, delivered in ten minutes instead of a day — and it would have left 121 queries and a blocking computation intact behind it. Every cache miss would still cost 2.4 s, so a deploy, a cache flush, or a cold key at peak traffic would reproduce the original problem at the worst possible moment, and by then the cache would be load-bearing enough that nobody could remove it to investigate. Caching is a multiplier on whatever it fronts: applied to a correct implementation it reduces load, and applied to a broken one it hides the breakage and raises the cost of the eventual failure. Fix the query, then fix the blocking, then cache what remains — and treat any cache introduced during an incident as debt to be revisited once the incident is over.

---

## 34.8 Questions

**1.** MCQ — The NestJS pipeline typically adds: a) tens of microseconds b) milliseconds c) tens of ms d) it varies wildly
<details><summary>Answer</summary>**a)** Which is why it is almost never the bottleneck.</details>

**2.** MCQ — Synchronous CPU work in a handler affects: a) that request b) every concurrent request c) the next request d) nothing
<details><summary>Answer</summary>**b)** One thread runs all JavaScript.</details>

**3.** MCQ — Adding replicas fixes: a) event-loop blocking b) N+1 queries c) neither d) both
<details><summary>Answer</summary>**c)** Each replica has the same single thread and issues the same queries.</details>

**4.** MCQ — 10 replicas with a pool of 20 opens: a) 20 b) 30 c) 200 d) 100
<details><summary>Answer</summary>**c)** Which exceeds PostgreSQL's default `max_connections` of 100.</details>

**5.** MCQ — Fastify's advantage over Express is largest when: a) queries are slow b) handlers are thin and responses large c) uploads dominate d) always
<details><summary>Answer</summary>**b)** It speeds up routing and serialization, not your database.</details>

**6.** MCQ — Which tool first tells you *what kind* of bottleneck you have? a) autocannon b) clinic doctor c) --cpu-prof d) htop
<details><summary>Answer</summary>**b)** It classifies the problem, which tells you which tool to use next.</details>

**7.** Short — Why benchmark a production build?
<details><summary>Answer</summary>`start:dev` runs the TypeScript watcher and development-only middleware, so you measure the tooling rather than the application.</details>

**8.** Short — Why is event-loop lag the best early warning?
<details><summary>Answer</summary>It rises before latency and memory do, because requests queue behind blocking work before any of them time out.</details>

**9.** Short — Why is caching a slow query dangerous?
<details><summary>Answer</summary>The underlying cost is unchanged, so every cold miss reproduces the original latency — and by then the cache is load-bearing and hard to remove.</details>

**10.** Short — When does keep-alive matter most?
<details><summary>Answer</summary>When outbound calls are frequent and short, since the TLS handshake can exceed the call itself.</details>

**11.** Short — Why prefer one process per container over clustering in an orchestrator?
<details><summary>Answer</summary>The orchestrator already handles restarts and placement, and one process per container makes memory limits and metrics attributable to a single unit.</details>

**12.** Output — A handler runs a 200 ms synchronous loop. 50 concurrent requests arrive. What is the latency of the last one?
<details><summary>Answer</summary>Roughly 10 seconds — they serialise behind each other on the single thread.</details>

**13.** Output — An endpoint issues 121 queries per request. What does adding a second replica do to total database load?
<details><summary>Answer</summary>Doubles it. Replicas multiply queries; they do not reduce them.</details>

**14.** Output — A Piscina pool is created and never destroyed. What happens when the app shuts down?
<details><summary>Answer</summary>The process does not exit — the worker threads keep it alive. Destroy the pool in `onModuleDestroy`.</details>

**15.** Output — `maxThreads: cpus().length` for a worker pool. What is the cost?
<details><summary>Answer</summary>The main thread competes with workers for cores, so event-loop lag rises under load. Leave one core for the event loop.</details>

**16.** Debug — p50 is 20 ms and p99 is 3 s. Where do you look first?
<details><summary>Answer</summary>Event-loop blocking or GC pauses — a minority of requests doing synchronous work delays everything queued behind them. Measure loop lag and correlate with the tail.</details>

**17.** Debug — Scaling from 5 to 12 replicas caused connection errors. Cause?
<details><summary>Answer</summary>Pool size times replicas exceeded the database's `max_connections`. Reduce the pool, raise the limit, or add PgBouncer.</details>

**18.** Debug — An endpoint is fast locally and slow in production with the same data volume. What differs?
<details><summary>Answer</summary>Network round trips — no keep-alive on outbound calls, higher latency to the database, and a cold cache. Local runs hide all three.</details>

**19.** Debug — Switching to Fastify produced no measurable improvement. Why?
<details><summary>Answer</summary>The time was in the database or downstream calls, not in routing and serialization, so halving a small fraction changed nothing.</details>

**20.** Debug — CPU sits at 100% on one core while others idle. Diagnosis?
<details><summary>Answer</summary>Single-threaded CPU-bound work on the event loop. Move it to worker threads or a separate worker process; more replicas will not use the idle cores of this one.</details>

**21.** Code — Move CPU-bound work off the event loop while still awaiting the result.
<details><summary>Answer</summary>

```ts
private readonly pool = new Piscina({
  filename: join(__dirname, 'stats.worker.js'),
  maxThreads: Math.max(1, cpus().length - 1),   // keep a core for the loop
});

compute(input: Input): Promise<Stats> { return this.pool.run(input); }

async onModuleDestroy() { await this.pool.destroy(); }
```
</details>

**22.** Code — Configure outbound HTTP with keep-alive and a timeout.
<details><summary>Answer</summary>

```ts
HttpModule.register({
  httpAgent: new http.Agent({ keepAlive: true, maxSockets: 100 }),
  httpsAgent: new https.Agent({ keepAlive: true, maxSockets: 100 }),
  timeout: 5_000,
  maxRedirects: 3,
});
```
</details>

**23.** Code — Replace an N+1 with one bounded, column-limited query.
<details><summary>Answer</summary>

```ts
return this.repo
  .createQueryBuilder('project')
  .leftJoinAndSelect('project.tasks', 'task')
  .select(['project.id', 'project.name', 'task.id', 'task.status'])
  .where('project.ownerId = :ownerId', { ownerId })
  .take(50)
  .getMany();
```
</details>

**24.** Scenario — A manager asks you to "make the API 2x faster" with no further detail. How do you proceed?
<details><summary>Answer</summary>Convert it into a measurable objective before touching anything, because "the API" is not a thing that has a speed. Ask which endpoints, at which percentile, under what load, and whether the complaint originates from users, dashboards, or a cost concern — those lead to different work. Then measure the current state so improvement is demonstrable, and rank endpoints by traffic times latency, since halving a rarely-used endpoint achieves nothing. Usually two or three endpoints account for most of the pain, and the causes are the same suspects: N+1 queries, missing indexes, unbounded payloads, and blocking work. Report back with a specific target and the measurements behind it — a vague goal produces vague work, and there is no way to tell when you are finished.</details>

**25.** Scenario — Traffic will grow 10x in three months. What do you check now?
<details><summary>Answer</summary>Find what breaks non-linearly rather than tuning what is currently slow. Connection arithmetic first, because pool size times replicas hits a hard database limit and fails abruptly rather than degrading. Then anything holding in-memory state — caches, rate limits, WebSocket fan-out, schedulers — since each is correct today and silently wrong at more replicas. Then unbounded queries, which are fine at current data volumes and become table scans as rows accumulate. Load-test at 10x against production-sized data, not an empty database, because most scaling failures are data-volume failures rather than request-rate failures. Finally check that the things you would need during an incident already exist — event-loop lag, queue depth and age, pool utilisation — because adding observability under load is the worst time to do it.</details>

---

## Chapter Summary

- The framework costs microseconds; slow endpoints are almost always N+1 queries, missing indexes, unbounded payloads, or event-loop blocking.
- Synchronous work blocks every concurrent request, so CPU-bound handlers cannot be fixed by adding replicas — they need worker threads, a queue, or the database.
- Measure before acting: `clinic doctor` classifies the bottleneck, autocannon establishes the ceiling, and `--cpu-prof` localises CPU time.
- Fastify roughly doubles throughput on thin, JSON-heavy handlers and changes nothing when the time is downstream — and migrating late means rewriting middleware, uploads, and raw responses.
- Scaling out breaks in-memory caches, rate limits, socket broadcasts, and schedulers, and multiplies connection pools into the database's hard limit.
- Cache last: applied to a broken query it hides the breakage and guarantees the original latency returns on the first cold miss.

---

## Practice Exercises

1. Benchmark the same endpoint under `start:dev` and a production build, and compare.
2. Add a 200 ms synchronous loop to a handler and measure latency at 50 concurrent connections.
3. Instrument event-loop lag, introduce blocking work, and watch lag rise before latency does.
4. Compute pool size times replicas against your database's `max_connections`, then scale past it deliberately.
5. Take one slow endpoint through the full sequence: fix the query, move the CPU work, then cache — measuring after each step.

---

## Mini Project: A Performance Investigation

Take one slow endpoint from symptom to fix, with evidence at every step.

**Requirements:**

- A reproducible load test against production-sized data, with a recorded baseline.
- A diagnosis naming the bottleneck class before any change is made.
- Fixes applied in order — query, blocking work, then caching — with a measurement after each.
- Event-loop lag, query count per request, and pool utilisation recorded throughout.
- A written summary stating what was tried and did not help, which is usually the more valuable half.

---

**Next:** [Chapter 35 — Architecture Patterns →](35-architecture-patterns.md)
