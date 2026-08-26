# Chapter 33 — Logging and Observability

## Learning Objectives

- Replace Nest's default logger with a structured one, without losing bootstrap output
- Correlate every log line, span, and error belonging to one request
- Instrument an application with metrics that answer operational questions
- Add distributed tracing with OpenTelemetry
- Build health checks that tell an orchestrator the truth

---

## 33.1 Structured Logs

Nest's default logger prints coloured, human-shaped text. That is right for a terminal and wrong for production, where logs are queried rather than read.

```ts
// ❌ A string. Finding "all errors for user 42" means substring matching.
this.logger.error(`Failed to charge user ${userId}: ${err.message}`);

// ✅ Fields. Queryable by any of them.
this.logger.error({ event: 'charge.failed', userId, reason: err.message }, 'Charge failed');
```

`nestjs-pino` is the usual choice — Pino is fast, JSON-native, and integrates with the request lifecycle:

```bash
npm i nestjs-pino pino-http pino
```

```ts
LoggerModule.forRootAsync({
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    pinoHttp: {
      level: config.get('LOG_LEVEL') ?? 'info',
      // Pretty printing only outside production; in production JSON is the point.
      transport: config.get('NODE_ENV') !== 'production'
        ? { target: 'pino-pretty' }
        : undefined,
      // Redaction is not optional — see Chapter 16.
      redact: {
        paths: [
          'req.headers.authorization',
          'req.headers.cookie',
          'req.body.password',
          'res.headers["set-cookie"]',
        ],
        censor: '[redacted]',
      },
      customProps: (req) => ({ correlationId: (req as any).correlationId }),
      autoLogging: {
        // Probes would otherwise dominate the log volume and cost.
        ignore: (req) => ['/health', '/metrics'].includes(req.url ?? ''),
      },
    },
  }),
});
```

```ts
// src/main.ts
const app = await NestFactory.create(AppModule, { bufferLogs: true });
app.useLogger(app.get(Logger));
```

`bufferLogs: true` holds everything logged during bootstrap until the real logger is attached, so startup messages are not lost or emitted in the wrong format (→ [Ch 3](03-first-application.md)).

> ⚠️ **Gotcha** — `pino-pretty` as a transport in production costs real throughput and produces output your log platform cannot parse into fields. It is a development convenience that ends up shipped because nothing visibly breaks — the logs simply stop being queryable.

---

## 33.2 Correlation

A single request produces log lines from a guard, a service, a repository, and possibly a queue consumer minutes later. Without a shared id, they cannot be assembled.

The mechanism is the `AsyncLocalStorage` context from [Chapter 9](09-scopes-lifecycle.md): middleware generates or accepts a correlation id, and every log line includes it.

Three rules make it useful:

**Accept an inbound id.** If the caller sent `x-correlation-id`, use it — that is what links your logs to theirs.

**Echo it in the response.** A user reporting a problem can then quote it, and a support request becomes one query.

**Propagate it outward.** Every downstream HTTP call, queue message, and event must carry it, or the trail stops at your boundary:

```ts
await this.queue.add('send-email', {
  userId,
  correlationId: this.context.correlationId,     // survives into the worker
});
```

A queue consumer then re-establishes the context from the payload, so work happening minutes later is still attributable to the request that caused it.

---

## 33.3 Metrics

Logs answer "what happened to this request". Metrics answer "how is the system behaving". You need both, and they are not substitutes.

```bash
npm i @willsoto/nestjs-prometheus prom-client
```

Four metric types, with different uses:

| Type | Answers | Example |
|---|---|---|
| Counter | how many | requests, errors, jobs processed |
| Gauge | how many right now | queue depth, open connections |
| Histogram | distribution | request duration, payload size |
| Summary | client-side quantiles | rarely the right choice |

Prefer histograms to summaries: histogram buckets can be aggregated across instances, and summaries cannot, so a p99 computed per-replica cannot be combined into a meaningful cluster p99.

**Label cardinality is the operational trap.** Every distinct label combination is a separate time series:

```ts
// ❌ One series per user — millions, and the metrics backend falls over
httpDuration.observe({ route: req.url, userId: req.user.id }, ms);

// ✅ Bounded: the route pattern, not the URL
httpDuration.observe({ route: req.route?.path ?? 'unknown', method, status }, ms);
```

Never put an id, an email, or a raw URL in a label. If you need per-user detail, that is a log, not a metric.

The metrics worth having from day one are request rate, error rate, and duration by route; queue depth and oldest-job age (→ [Ch 27](27-queues-jobs.md)); database pool utilisation; and event-loop lag, which is the single best early indicator that a Node process is in trouble.

---

## 33.4 Tracing

Tracing shows one request's path across services and layers, with timing at each step. It is what tells you the 800 ms is in a downstream call rather than in your code.

```bash
npm i @opentelemetry/sdk-node @opentelemetry/auto-instrumentations-node
```

```ts
// src/tracing.ts — MUST be imported before anything else
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';

const sdk = new NodeSDK({
  serviceName: process.env.SERVICE_NAME ?? 'api',
  instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();
```

```ts
// src/main.ts
import './tracing';                    // first import, before AppModule
import { NestFactory } from '@nestjs/core';
```

> ⚠️ **Gotcha** — Auto-instrumentation works by patching modules as they are loaded, so `tracing.ts` must be imported **before** anything it instruments. Importing it after `NestFactory` means HTTP, database, and Redis calls are never traced, and the failure is silent — you get traces containing only the spans you created by hand.

Custom spans mark the parts that matter:

```ts
async generateReport(id: string): Promise<Buffer> {
  return this.tracer.startActiveSpan('report.generate', async (span) => {
    span.setAttribute('report.id', id);
    try {
      return await this.render(id);
    } catch (err) {
      span.recordException(err as Error);
      span.setStatus({ code: SpanStatusCode.ERROR });
      throw err;
    } finally {
      span.end();                      // a span never ended is a trace never closed
    }
  });
}
```

**Sample in production.** Tracing every request is expensive at volume; 1–10% with a rule that always samples errors gives you the tail without the cost.

---

## 33.5 Health Checks

```bash
npm i @nestjs/terminus
```

Liveness and readiness answer different questions, and conflating them causes outages:

**Liveness** — is this process broken beyond recovery? Failing it gets the container killed. It must check almost nothing.

**Readiness** — should this instance receive traffic right now? Failing it removes the instance from rotation temporarily.

```ts
@Controller('health')
export class HealthController {
  @Get('live')
  @HealthCheck()
  live() {
    // Deliberately no dependency checks: if the database is down, restarting
    // this process does not help, and killing every replica makes it worse.
    return this.health.check([]);
  }

  @Get('ready')
  @HealthCheck()
  ready() {
    return this.health.check([
      () => this.db.pingCheck('database', { timeout: 1_500 }),
      () => this.memory.checkHeap('heap', 512 * 1024 * 1024),
    ]);
  }
}
```

**Putting a database check in liveness is the classic mistake.** A brief database outage then fails liveness on every replica simultaneously, the orchestrator kills them all, and the restart storm turns a recoverable dependency blip into a full outage.

Readiness should also fail during shutdown, before connections close, so the load balancer drains the instance first (→ [Ch 3](03-first-application.md)).

> 💼 **Interview** — "What is the difference between liveness and readiness?" The strong answer names the consequence of each failing — killed versus removed from rotation — and gives the concrete failure mode of checking dependencies in liveness.

---

## 33.6 Common Mistakes and Best Practices

**Mistake: string interpolation in logs.** The fields cannot be queried or aggregated.

**Mistake: `pino-pretty` in production.** Throughput cost, and the platform can no longer parse fields.

**Mistake: no redaction.** Authorization headers, cookies, and passwords land in the log store.

**Mistake: high-cardinality metric labels.** User ids or raw URLs create one series per value and overwhelm the backend.

**Mistake: importing tracing after the app.** Auto-instrumentation patches nothing and fails silently.

**Mistake: a span that is never ended.** The trace never closes and the data is unusable.

**Mistake: dependency checks in liveness.** A dependency blip triggers a cluster-wide restart storm.

**Best practice: buffer logs and swap in the real logger at bootstrap.** Startup output is often where a broken deploy explains itself.

**Best practice: propagate the correlation id into queues and outbound calls.** Otherwise the trail stops at your process boundary.

**Best practice: use histograms and bounded labels.** Aggregatable across replicas, and they do not explode.

---

## 33.7 Real-World Example: A Correlated Observability Layer

Logs, metrics, and traces sharing one correlation id, with cardinality bounded and secrets redacted.

```ts
// src/observability/observability.interceptor.ts
@Injectable()
export class ObservabilityInterceptor implements NestInterceptor {
  private readonly logger = new Logger(ObservabilityInterceptor.name);

  constructor(
    private readonly context: RequestContext,
    @InjectMetric('http_request_duration_seconds')
    private readonly duration: Histogram<string>,
    @InjectMetric('http_requests_total')
    private readonly total: Counter<string>,
  ) {}

  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    if (ctx.getType() !== 'http') return next.handle();

    const req = ctx.switchToHttp().getRequest<Request>();
    const res = ctx.switchToHttp().getResponse<Response>();

    // The route PATTERN, never the URL. `/tasks/:id` is one series;
    // `/tasks/8f3c...` would be one series per task (→ §33.3).
    const route = req.route?.path ?? 'unknown';
    const start = process.hrtime.bigint();

    const record = (status: number, outcome: 'success' | 'error') => {
      const seconds = Number(process.hrtime.bigint() - start) / 1e9;
      const labels = { method: req.method, route, status: String(status) };
      this.duration.observe(labels, seconds);
      this.total.inc({ ...labels, outcome });
      return seconds;
    };

    return next.handle().pipe(
      tap({
        next: () => {
          const seconds = record(res.statusCode, 'success');
          this.logger.log({
            event: 'http.request',
            method: req.method,
            route,
            status: res.statusCode,
            durationMs: Math.round(seconds * 1000),
            correlationId: this.context.correlationId,
          });
        },
        // Without this branch, failures are absent from both the metric and
        // the log — the dashboard looks healthy during an outage (→ Ch 13).
        error: (err) => {
          const status = err instanceof HttpException ? err.getStatus() : 500;
          const seconds = record(status, 'error');
          this.logger[status >= 500 ? 'error' : 'warn']({
            event: 'http.request.failed',
            method: req.method,
            route,
            status,
            durationMs: Math.round(seconds * 1000),
            correlationId: this.context.correlationId,
            reason: err instanceof Error ? err.message : String(err),
          });
        },
      }),
    );
  }
}
```

```ts
// src/observability/event-loop.monitor.ts
@Injectable()
export class EventLoopMonitor implements OnModuleInit, OnModuleDestroy {
  private histogram?: IntervalHistogram;

  constructor(
    @InjectMetric('nodejs_event_loop_lag_seconds') private readonly gauge: Gauge<string>,
  ) {}

  onModuleInit(): void {
    // Event-loop lag is the earliest reliable signal that a Node process is
    // in trouble — it rises before latency does and before memory does.
    this.histogram = monitorEventLoopDelay({ resolution: 20 });
    this.histogram.enable();
  }

  @Interval('event-loop-lag', 10_000)
  sample(): void {
    if (!this.histogram) return;
    this.gauge.set({ quantile: 'p50' }, this.histogram.percentile(50) / 1e9);
    this.gauge.set({ quantile: 'p99' }, this.histogram.percentile(99) / 1e9);
    this.histogram.reset();
  }

  onModuleDestroy(): void {
    this.histogram?.disable();
  }
}
```

```ts
// src/observability/queue-context.ts
/**
 * Re-establishes the correlation id inside a worker, so a job processed
 * minutes later is still attributable to the request that enqueued it.
 * Without this the trail ends at the enqueue call.
 */
export function withJobContext<T>(
  context: RequestContext,
  job: Job<{ correlationId?: string }>,
  work: () => Promise<T>,
): Promise<T> {
  return context.run(
    { correlationId: job.data.correlationId ?? `job:${job.id}`, startedAt: Date.now() },
    work,
  );
}
```

```ts
// src/health/health.controller.ts
@Controller('health')
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly db: TypeOrmHealthIndicator,
    private readonly memory: MemoryHealthIndicator,
    private readonly shutdown: ShutdownService,
  ) {}

  @Public()
  @Get('live')
  @HealthCheck()
  live() {
    // Empty on purpose. Liveness must not depend on anything a restart
    // cannot fix, or a database blip becomes a cluster-wide restart storm.
    return this.health.check([]);
  }

  @Public()
  @Get('ready')
  @HealthCheck()
  ready() {
    if (this.shutdown.isShuttingDown()) {
      throw new ServiceUnavailableException('shutting down');
    }
    return this.health.check([
      () => this.db.pingCheck('database', { timeout: 1_500 }),
      () => this.memory.checkHeap('heap', 512 * 1024 * 1024),
    ]);
  }
}
```

The decision worth defending is an empty liveness check. It looks like an omission — a health endpoint that verifies nothing — and it is the only correct behaviour for a check whose failure kills the container. The tempting version pings the database, and its failure mode is catastrophic in exactly the situation you least want it: the database has a thirty-second blip, every replica fails liveness simultaneously, the orchestrator kills all of them, and the restarts hit a database that is already struggling, converting a recoverable incident into an outage that outlasts its cause. Readiness is where dependency checks belong, because its failure is reversible — the instance leaves rotation and rejoins when the dependency recovers, with no process death. The rule is to ask what restarting would fix: a wedged event loop or an exhausted heap, yes; anything outside the process, no.

---

## 33.8 Questions

**1.** MCQ — Structured logging means: a) coloured output b) queryable fields c) less output d) JSON files
<details><summary>Answer</summary>**b)** Fields can be filtered and aggregated; interpolated strings cannot.</details>

**2.** MCQ — `bufferLogs: true` prevents: a) log loss during bootstrap b) slow logging c) duplicate logs d) redaction
<details><summary>Answer</summary>**a)** Output is held until the real logger is attached.</details>

**3.** MCQ — Which metric label is dangerous? a) route pattern b) HTTP method c) user id d) status code
<details><summary>Answer</summary>**c)** One time series per user destroys the metrics backend.</details>

**4.** MCQ — OpenTelemetry auto-instrumentation must be imported: a) anywhere b) before the modules it instruments c) after bootstrap d) in a provider
<details><summary>Answer</summary>**b)** It patches modules at load time, and failing to do so is silent.</details>

**5.** MCQ — A failing liveness probe causes the orchestrator to: a) remove from rotation b) restart the container c) alert d) scale up
<details><summary>Answer</summary>**b)** Which is why it must not depend on external services.</details>

**6.** MCQ — Prefer histograms to summaries because histograms: a) are cheaper b) aggregate across instances c) are more accurate d) need fewer labels
<details><summary>Answer</summary>**b)** Per-instance summary quantiles cannot be meaningfully combined.</details>

**7.** Short — Why must the correlation id be propagated into queue jobs?
<details><summary>Answer</summary>Otherwise the trail ends at the enqueue call, and work performed minutes later cannot be attributed to the request that caused it.</details>

**8.** Short — Why is `pino-pretty` wrong in production?
<details><summary>Answer</summary>It costs throughput and emits human-formatted text that the log platform cannot parse into queryable fields.</details>

**9.** Short — Why is event-loop lag worth measuring?
<details><summary>Answer</summary>It rises before latency and memory do, making it the earliest reliable indicator that a Node process is saturated or blocked.</details>

**10.** Short — Why should liveness check nothing external?
<details><summary>Answer</summary>Its failure kills the container, so a dependency blip would restart every replica at once and worsen the incident a restart cannot fix.</details>

**11.** Short — Why sample traces in production?
<details><summary>Answer</summary>Tracing every request is expensive at volume; a small sample plus always-sample-on-error captures the useful tail at a fraction of the cost.</details>

**12.** Output — A metric labels by `req.url` on an API with a million task ids. What happens to the backend?
<details><summary>Answer</summary>A million time series, and the metrics store degrades or starts rejecting writes.</details>

**13.** Output — `tracing.ts` is imported after `AppModule`. Which spans appear?
<details><summary>Answer</summary>Only manually created ones — HTTP, database, and Redis calls are unpatched and untraced, with no error.</details>

**14.** Output — A span is created and never `end()`ed. What does the trace show?
<details><summary>Answer</summary>An unterminated span, so the trace never closes and is discarded or shown as incomplete.</details>

**15.** Output — Liveness pings the database, which is down for 30 seconds across a 10-replica deployment. What happens?
<details><summary>Answer</summary>All ten fail liveness and are killed simultaneously, and the restart storm hits an already-struggling database.</details>

**16.** Debug — Logs are readable in the console and unqueryable in the platform. Cause?
<details><summary>Answer</summary>`pino-pretty` is enabled in production, so output is formatted text rather than JSON fields.</details>

**17.** Debug — An `Authorization` header appeared in the log store. What is missing?
<details><summary>Answer</summary>A `redact` configuration covering `req.headers.authorization`, cookies, and password fields.</details>

**18.** Debug — Traces show only your manual spans and none from the database driver. Cause?
<details><summary>Answer</summary>The tracing SDK was initialised after the driver was imported, so it could not patch it.</details>

**19.** Debug — Dashboards show a healthy error rate while users report failures. What is wrong with the instrumentation?
<details><summary>Answer</summary>The interceptor uses `tap(fn)` with only a next handler, so failed requests are never recorded (→ Ch 13).</details>

**20.** Debug — During a deploy, requests are dropped despite a readiness probe. What is missing?
<details><summary>Answer</summary>Readiness does not fail on shutdown, so the load balancer keeps routing until the process dies.</details>

**21.** Code — Record duration and outcome for both success and failure.
<details><summary>Answer</summary>

```ts
return next.handle().pipe(
  tap({
    next: () => histogram.observe({ route, status: String(res.statusCode) }, elapsed()),
    error: (e) => histogram.observe(
      { route, status: String(e instanceof HttpException ? e.getStatus() : 500) },
      elapsed(),
    ),
  }),
);
```
</details>

**22.** Code — Write liveness and readiness that fail correctly.
<details><summary>Answer</summary>

```ts
@Get('live') @HealthCheck()
live() { return this.health.check([]); }        // nothing external

@Get('ready') @HealthCheck()
ready() {
  if (this.shutdown.isShuttingDown()) throw new ServiceUnavailableException();
  return this.health.check([() => this.db.pingCheck('database', { timeout: 1_500 })]);
}
```
</details>

**23.** Code — Re-establish correlation context inside a queue worker.
<details><summary>Answer</summary>

```ts
async process(job: Job<{ correlationId?: string }>): Promise<void> {
  await this.context.run(
    { correlationId: job.data.correlationId ?? `job:${job.id}` },
    () => this.doWork(job),
  );
}
```
</details>

**24.** Scenario — A teammate proposes logging every request body to help debugging. Evaluate it.
<details><summary>Answer</summary>The debugging benefit is real and the cost is usually underestimated on three axes. Volume: bodies dominate log storage and the bill, often by an order of magnitude. Privacy: bodies contain passwords, tokens, personal data, and payment details, so the log store inherits the compliance obligations of your database while having weaker access controls. And usefulness decays — a body is only helpful when you already know which request to look at, which is what the correlation id is for. Log a bounded set of fields chosen deliberately, redact aggressively, and if full bodies are genuinely needed for a class of endpoint, sample them, cap their size, and set a short retention. Blanket body logging tends to be enabled during an incident and never turned off.</details>

**25.** Scenario — Latency is fine at p50 and terrible at p99, and nobody can say why. How do you find it?
<details><summary>Answer</summary>A p50/p99 divergence means a subset of requests behaves differently, so aggregate metrics will not localise it — you need per-request detail for the slow ones specifically. Configure tracing to always sample requests exceeding a latency threshold, which gives you traces for exactly the population in question rather than a random sample dominated by fast requests. Then check event-loop lag against the same timeline: if the tail correlates with lag spikes, the cause is blocking work rather than any downstream call, and no amount of database tuning will help. Also look for the usual suspects that only affect a minority — cold caches, an N+1 that triggers on a rare relation, connection-pool exhaustion, and garbage-collection pauses. The general method is to make the instrumentation conditional on the symptom, because uniform sampling systematically under-represents the thing you are trying to see.</details>

---

## Chapter Summary

- Structured, field-based logs are queryable; interpolated strings are not, and `pino-pretty` in production quietly removes that property.
- `bufferLogs: true` preserves bootstrap output, which is often where a broken deploy explains itself.
- A correlation id is only useful if it is accepted from the caller, echoed to the client, and propagated into queues and outbound calls.
- Metric labels must be bounded — a route pattern, never a URL or a user id — and histograms aggregate across replicas where summaries cannot.
- OpenTelemetry auto-instrumentation patches at import time, so initialising it late produces traces missing everything you cared about, with no error.
- Liveness must check nothing external, because its failure kills the container and turns a dependency blip into a cluster-wide restart storm; dependency checks belong in readiness.

---

## Practice Exercises

1. Convert a string log line to structured fields and query both in your log platform.
2. Label a histogram by raw URL, generate a thousand distinct ids, and observe the series count.
3. Import the tracing SDK after `AppModule` and compare the resulting traces.
4. Put a database check in liveness, stop the database, and watch the restart behaviour.
5. Enqueue a job without the correlation id, then with it, and try to reconstruct the request in each case.

---

## Mini Project: An Observability Baseline

Build the instrumentation you would want during an incident.

**Requirements:**

- Structured logs with redaction, correlation ids accepted, echoed, and propagated into queues.
- Request rate, error rate, and duration by route pattern, recorded on both success and failure paths.
- Event-loop lag and database pool utilisation as gauges.
- Tracing initialised before anything else, with tail-based sampling that always keeps slow and failed requests.
- Liveness with no external checks and readiness that fails on shutdown, verified by a deploy that drops no requests.

---

**Next:** [Chapter 34 — Performance and Scaling →](34-performance-scaling.md)
