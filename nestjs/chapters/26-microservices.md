# Chapter 26 — Microservices and Transports

## Learning Objectives

- Distinguish message patterns from event patterns, and choose correctly
- Build a microservice and a hybrid application that serves HTTP and messages
- Configure TCP, Redis, NATS, RabbitMQ, Kafka, and gRPC transports and their trade-offs
- Handle the failure modes that distributed messaging introduces
- Recognise when a monolith with modules is the better answer

---

## 26.1 Two Patterns, Two Semantics

Nest's microservice layer is a transport abstraction over two interaction styles, and confusing them is the most consequential mistake in this chapter.

**Message patterns** are request–response. The caller waits for a reply.

```ts
@MessagePattern({ cmd: 'get_user' })
getUser(@Payload() id: string): Promise<User> {
  return this.users.findOne(id);      // the return value is sent back
}
```

**Event patterns** are fire-and-forget. There is no reply, and the emitter does not know whether anyone handled it.

```ts
@EventPattern('user.created')
async handleUserCreated(@Payload() event: UserCreatedEvent): Promise<void> {
  await this.email.sendWelcome(event.email);   // nothing is returned
}
```

On the client side these map to `send` and `emit`:

```ts
this.client.send({ cmd: 'get_user' }, id);   // Observable<User> — a reply is expected
this.client.emit('user.created', event);     // Observable<void> — dispatch only
```

| | `send` / `@MessagePattern` | `emit` / `@EventPattern` |
|---|---|---|
| Reply | yes | no |
| Caller blocked | yes | no |
| Consumers | one | any number |
| Failure visible to caller | yes | no |
| Use for | queries, commands needing a result | notifications, side effects |

**Use events by default.** A `send` couples the caller's availability and latency to the callee's; an `emit` does not. Reach for request–response only when the caller genuinely cannot proceed without the answer.

> ⚠️ **Gotcha** — `client.send()` returns a **cold Observable**. Nothing is sent until something subscribes. `this.client.send(pattern, data);` on its own is a no-op with no error — the message never leaves. Return it from a handler (Nest subscribes), `await lastValueFrom(...)`, or call `.subscribe()`.

---

## 26.2 Standalone and Hybrid Applications

A pure microservice has no HTTP server:

```ts
// src/main.ts
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.TCP,
  options: { host: '0.0.0.0', port: 3001 },
});
await app.listen();
```

A **hybrid application** serves both, which is the common shape — an HTTP API that also consumes events:

```ts
const app = await NestFactory.create(AppModule);

app.connectMicroservice<MicroserviceOptions>({
  transport: Transport.RMQ,
  options: {
    urls: [config.getOrThrow('RABBITMQ_URL')],
    queue: 'tasks_queue',
    queueOptions: { durable: true },        // survives a broker restart
    noAck: false,                           // manual acknowledgement — see §26.4
  },
});

await app.startAllMicroservices();          // must precede listen()
await app.listen(3000);
```

Calling another service needs a client:

```ts
ClientsModule.registerAsync([
  {
    name: 'USERS_SERVICE',
    imports: [ConfigModule],
    inject: [ConfigService],
    useFactory: (config: ConfigService) => ({
      transport: Transport.TCP,
      options: { host: config.getOrThrow('USERS_HOST'), port: 3001 },
    }),
  },
]);
```

```ts
@Injectable()
export class TasksService {
  constructor(@Inject('USERS_SERVICE') private readonly users: ClientProxy) {}

  async getOwner(id: string): Promise<User> {
    return firstValueFrom(
      this.users.send<User>({ cmd: 'get_user' }, id).pipe(
        timeout(3_000),                     // a hung callee must not hang us
        catchError(() => throwError(() => new ServiceUnavailableException())),
      ),
    );
  }
}
```

---

## 26.3 Choosing a Transport

| Transport | Delivery | Persistence | Consumers | Best for |
|---|---|---|---|---|
| **TCP** | at-most-once | none | one | simple internal request–response |
| **Redis** | at-most-once | none | pub/sub fan-out | ephemeral notifications |
| **NATS** | at-most-once (core) | with JetStream | queue groups | low-latency internal messaging |
| **RabbitMQ** | at-least-once | durable queues | competing consumers | work queues needing acknowledgement |
| **Kafka** | at-least-once | a durable log | consumer groups, replay | event streaming, audit, replay |
| **gRPC** | at-most-once | none | one | typed contracts, polyglot services |

Two rows decide most choices.

**Persistence.** Redis pub/sub drops messages when no subscriber is connected — a consumer restart loses everything published during it. Anything that must not be lost needs RabbitMQ or Kafka.

**Replay.** Only Kafka lets a new consumer read history. If a future service will need events that already happened, that is a Kafka requirement and nothing else provides it.

> ⚠️ **Gotcha** — Kafka request–response requires the client to subscribe to the reply topic **before** sending, in `onModuleInit`. Omitting it makes every `send()` hang until it times out, with no error explaining why:

```ts
async onModuleInit() {
  this.client.subscribeToResponseOf('get_user');   // creates `get_user.reply`
  await this.client.connect();
}
```

**gRPC** is different in kind — it is a typed RPC protocol with `.proto` contracts:

```ts
{
  transport: Transport.GRPC,
  options: {
    package: 'users',
    protoPath: join(__dirname, 'users.proto'),
    url: '0.0.0.0:5000',
  },
}
```

```ts
@GrpcMethod('UsersService', 'FindOne')
findOne(data: { id: string }): Promise<User> {
  return this.users.findOne(data.id);
}
```

Choose gRPC when services are written in different languages, or when a schema-checked contract is worth a build step.

---

## 26.4 The Failure Modes

Distributed messaging introduces problems that in-process calls do not have, and none of them are optional to think about.

**Messages are delivered at least once, not exactly once.** A broker that does not receive an acknowledgement redelivers, so handlers must be idempotent (→ [Ch 27](27-queues-jobs.md)):

```ts
@EventPattern('order.placed')
async handle(@Payload() event: OrderPlacedEvent, @Ctx() context: RmqContext): Promise<void> {
  const channel = context.getChannelRef();
  const message = context.getMessage();

  try {
    // Idempotency key from the event, not generated here — a redelivery must
    // produce the same key, so the second attempt is recognised as a repeat.
    await this.orders.processOnce(event.eventId, event);
    channel.ack(message);
  } catch (err) {
    // requeue=false → dead-letter rather than an infinite redelivery loop
    channel.nack(message, false, false);
  }
}
```

**`noAck: false` is opt-in.** With the default, RabbitMQ considers a message delivered the moment it is sent, so a crash mid-processing loses it silently.

**Ordering is per-partition at best.** Kafka guarantees order within a partition; RabbitMQ guarantees it only with a single consumer. Any handler assuming `created` arrives before `updated` will eventually be wrong.

**The caller must time out.** A `send` with no timeout waits forever if the callee is down, and the waiting requests accumulate until the caller falls over too — one service's outage becomes everyone's.

---

## 26.5 When Not to Do This

Microservices trade in-process function calls for network calls, and the exchange rate is poor unless you are buying something specific.

A monolith with well-drawn modules gives you: type-checked calls, one transaction spanning a whole operation, one deployment, one log stream, and a stack trace that crosses the whole request. Splitting gives up all of that.

Split when there is a concrete driver:

- **Independent scaling** — one component needs ten times the capacity of the rest.
- **Team autonomy** — separate teams need separate release cadences.
- **Fault isolation** — one component must not take the rest down.
- **Technology** — a component genuinely needs a different runtime.

"Microservices are more scalable" is not a driver; a well-built monolith scales horizontally too. The cost you take on is distributed transactions, eventual consistency, network partitions, versioned message contracts, and debugging across services — and that cost arrives immediately while the benefits arrive later, if at all.

> 💼 **Interview** — "When would you use microservices?" The strong answer names a specific organisational or scaling driver and acknowledges the costs. Answering "for scalability" invites the follow-up "what stops you scaling a monolith horizontally?" — for which there is usually no good answer.

---

## 26.6 Common Mistakes and Best Practices

**Mistake: not subscribing to `send()`.** The Observable is cold; nothing is transmitted and nothing errors.

**Mistake: `send` where `emit` belongs.** It couples the caller's latency and availability to the callee's.

**Mistake: no timeout on `send`.** A dead callee turns into unbounded waiting and then a cascading failure.

**Mistake: assuming exactly-once delivery.** Every handler must be idempotent, because redelivery is normal.

**Mistake: leaving `noAck` at its default.** A crash mid-processing loses the message with no trace.

**Mistake: `nack` with requeue on a poison message.** It loops forever; dead-letter it instead.

**Mistake: forgetting `subscribeToResponseOf` with Kafka.** Every request hangs until timeout with no explanatory error.

**Best practice: prefer events to commands.** They decouple availability and let consumers be added without touching the emitter.

**Best practice: put an idempotency key in the event.** Generating one in the handler defeats the purpose, because a redelivery must produce the same key.

**Best practice: version message payloads from the start.** Consumers and producers deploy independently, so both old and new shapes will be in flight.

---

## 26.7 Real-World Example: An Event Consumer That Survives Redelivery

A RabbitMQ consumer with manual acknowledgement, idempotency, bounded retries, and dead-lettering — the four properties that separate a working consumer from one that loses or duplicates data.

```ts
// src/orders/order-events.controller.ts
@Controller()
export class OrderEventsController {
  private readonly logger = new Logger(OrderEventsController.name);
  private static readonly MAX_ATTEMPTS = 3;

  constructor(
    private readonly orders: OrdersService,
    private readonly processed: ProcessedEventRepository,
  ) {}

  @EventPattern('order.placed')
  async handleOrderPlaced(
    @Payload() event: OrderPlacedEvent,
    @Ctx() context: RmqContext,
  ): Promise<void> {
    const channel = context.getChannelRef();
    const message = context.getMessage();
    const attempt = (message.properties.headers?.['x-attempt'] ?? 0) + 1;

    try {
      // Idempotency first. The key travels WITH the event, so a redelivery
      // carries the same one and is recognised rather than reprocessed.
      const already = await this.processed.exists(event.eventId);
      if (already) {
        this.logger.debug(`Duplicate ${event.eventId}, acking without work`);
        channel.ack(message);
        return;
      }

      await this.orders.fulfil(event);

      // Recording completion and acking are not atomic: a crash between them
      // causes redelivery, which the exists() check above absorbs. Recording
      // first is the safe order — a duplicate ack is harmless, duplicate
      // fulfilment is not.
      await this.processed.record(event.eventId);
      channel.ack(message);
    } catch (err) {
      this.logger.error(`Attempt ${attempt} failed for ${event.eventId}`, err as Error);

      if (attempt >= OrderEventsController.MAX_ATTEMPTS) {
        // Dead-letter. Requeueing a message that fails deterministically
        // produces an infinite loop that saturates the consumer.
        channel.nack(message, false, false);
        return;
      }

      // Republish with an incremented attempt counter and a backoff delay,
      // rather than nack(requeue=true), which redelivers immediately.
      channel.publish(
        '',
        message.fields.routingKey,
        message.content,
        {
          headers: { ...message.properties.headers, 'x-attempt': attempt },
          expiration: String(2 ** attempt * 1000),
        },
      );
      channel.ack(message);
    }
  }
}
```

```ts
// src/orders/processed-event.entity.ts
@Entity('processed_events')
export class ProcessedEvent {
  // The unique constraint is the real guarantee. Two consumers processing a
  // redelivery concurrently both pass the exists() check; only one insert wins.
  @PrimaryColumn({ type: 'uuid' })
  eventId: string;

  @CreateDateColumn({ type: 'timestamptz' })
  processedAt: Date;
}
```

```ts
// src/orders/orders.service.ts
@Injectable()
export class OrdersService {
  async fulfil(event: OrderPlacedEvent): Promise<void> {
    await this.dataSource.transaction(async (manager) => {
      // The processed-event insert is INSIDE the transaction with the work,
      // so either both commit or neither does. Recording outside would allow
      // an event marked processed whose effects rolled back.
      await manager.insert(ProcessedEvent, { eventId: event.eventId });
      await manager.save(Fulfilment, Fulfilment.from(event));
      await manager.decrement(Inventory, { sku: event.sku }, 'available', event.quantity);
    });
  }
}
```

```ts
// src/main.ts
app.connectMicroservice<MicroserviceOptions>({
  transport: Transport.RMQ,
  options: {
    urls: [config.getOrThrow('RABBITMQ_URL')],
    queue: 'orders_queue',
    queueOptions: {
      durable: true,                                    // survives broker restart
      deadLetterExchange: 'orders_dlx',                 // where nacks land
      deadLetterRoutingKey: 'orders.dead',
    },
    noAck: false,                                       // manual ack — the default loses messages
    prefetchCount: 10,                                  // bound in-flight work per consumer
  },
});
```

The decision worth defending is putting the `ProcessedEvent` insert inside the same transaction as the work, rather than recording completion after the transaction commits. Recording afterwards is the obvious ordering and it has a window: if the process dies between commit and record, the event is redelivered, the `exists` check finds nothing, and the order is fulfilled twice — which for an inventory decrement means real stock loss. Putting the insert inside means the primary-key constraint and the business writes share one atomic outcome, so a redelivery either finds the marker and skips, or finds nothing because the whole thing rolled back and correctly reprocesses. The `exists` check before the transaction is then purely an optimisation to avoid doing work that will fail on the constraint; the constraint is the actual guarantee. This is the general shape of exactly-once processing on top of at-least-once delivery: you cannot make delivery exactly-once, so you make *effects* idempotent, and the only reliable place to do that is inside the transaction that produces them.

---

## 26.8 Questions

**1.** MCQ — `client.send()` returns: a) a Promise b) a hot Observable c) a cold Observable d) the value
<details><summary>Answer</summary>**c)** Nothing is transmitted until something subscribes.</details>

**2.** MCQ — `@EventPattern` handlers return: a) a reply b) nothing to the emitter c) an ack d) a status
<details><summary>Answer</summary>**b)** Events are fire-and-forget.</details>

**3.** MCQ — Which transport supports replaying past events? a) Redis b) TCP c) Kafka d) RabbitMQ
<details><summary>Answer</summary>**c)** Kafka is a durable log; the others discard delivered messages.</details>

**4.** MCQ — With RabbitMQ's default `noAck`, a crash mid-processing: a) redelivers b) loses the message c) dead-letters d) retries
<details><summary>Answer</summary>**b)** The broker considered it delivered on dispatch.</details>

**5.** MCQ — Kafka request–response requires: a) nothing extra b) `subscribeToResponseOf` in `onModuleInit` c) a separate client d) manual acks
<details><summary>Answer</summary>**b)** Without it, every `send` hangs until timeout with no explanatory error.</details>

**6.** MCQ — In a hybrid app, `startAllMicroservices()` must be called: a) after `listen()` b) before `listen()` c) in a module d) anywhere
<details><summary>Answer</summary>**b)**</details>

**7.** Short — Why prefer `emit` over `send` by default?
<details><summary>Answer</summary>`send` couples the caller's latency and availability to the callee's, while `emit` does not — and events allow new consumers without touching the emitter.</details>

**8.** Short — Why must every event handler be idempotent?
<details><summary>Answer</summary>Brokers deliver at least once, so redelivery after a missing acknowledgement is normal operation rather than an error case.</details>

**9.** Short — Why must the idempotency key travel with the event?
<details><summary>Answer</summary>A key generated inside the handler differs on each redelivery, so the duplicate is never recognised.</details>

**10.** Short — Why dead-letter rather than requeue a repeatedly failing message?
<details><summary>Answer</summary>A message failing deterministically will fail again immediately, producing an infinite loop that saturates the consumer and blocks healthy messages.</details>

**11.** Short — What does a monolith give you that microservices take away?
<details><summary>Answer</summary>Type-checked calls, one transaction across an operation, one deployment, one log stream, and stack traces that span the whole request.</details>

**12.** Output — `this.client.send({ cmd: 'x' }, data);` with no subscribe. What is transmitted?
<details><summary>Answer</summary>Nothing, and no error is raised. The Observable is cold.</details>

**13.** Output — An event is redelivered and the handler records completion *after* the transaction commits. The process dies between them. What happens?
<details><summary>Answer</summary>The event is reprocessed, so the effects apply twice — for an inventory decrement, real stock is lost.</details>

**14.** Output — `channel.nack(message, false, true)` on a message that always fails. What follows?
<details><summary>Answer</summary>Immediate redelivery, forever — a hot loop that starves other messages.</details>

**15.** Output — Two consumers process the same redelivered event concurrently, both pass `exists()`. What prevents double processing?
<details><summary>Answer</summary>The primary-key constraint on `eventId` inside the transaction — one insert wins and the other transaction rolls back.</details>

**16.** Debug — A `send()` call appears to do nothing and no error is logged. Cause?
<details><summary>Answer</summary>Nothing subscribed to the returned Observable. Return it from the handler, or use `firstValueFrom`.</details>

**17.** Debug — A consumer restart loses every message published during the downtime. Which transport is in use?
<details><summary>Answer</summary>Redis pub/sub, which has no persistence — messages published with no subscriber connected are dropped. Use RabbitMQ or Kafka.</details>

**18.** Debug — One service going down takes the whole system with it. What is missing on the callers?
<details><summary>Answer</summary>Timeouts on `send`, and ideally a circuit breaker. Without them, waiting requests accumulate until callers exhaust their own resources.</details>

**19.** Debug — Every Kafka request times out after 30 seconds with no error detail. Cause?
<details><summary>Answer</summary>`subscribeToResponseOf` was never called, so the reply topic is not consumed and the response never arrives.</details>

**20.** Debug — Events sometimes arrive out of order and a handler assumes `created` precedes `updated`. Is that fixable by configuration?
<details><summary>Answer</summary>Only partially — Kafka orders within a partition if you key by entity id, and RabbitMQ orders only with a single consumer. The durable fix is to make handlers order-independent, usually with a version or timestamp check.</details>

**21.** Code — Call another service with a timeout and a clean failure.
<details><summary>Answer</summary>

```ts
async getUser(id: string): Promise<User> {
  return firstValueFrom(
    this.client.send<User>({ cmd: 'get_user' }, id).pipe(
      timeout(3_000),
      catchError(() => throwError(() => new ServiceUnavailableException('users'))),
    ),
  );
}
```
</details>

**22.** Code — Write an idempotent consumer with manual acknowledgement.
<details><summary>Answer</summary>

```ts
@EventPattern('order.placed')
async handle(@Payload() event: OrderPlacedEvent, @Ctx() ctx: RmqContext) {
  const channel = ctx.getChannelRef();
  const message = ctx.getMessage();
  try {
    await this.orders.fulfil(event);      // inserts ProcessedEvent in the same tx
    channel.ack(message);
  } catch (err) {
    channel.nack(message, false, false);  // dead-letter, never requeue blindly
  }
}
```
</details>

**23.** Code — Configure a durable queue with dead-lettering and bounded prefetch.
<details><summary>Answer</summary>

```ts
{
  transport: Transport.RMQ,
  options: {
    urls: [url],
    queue: 'orders_queue',
    queueOptions: {
      durable: true,
      deadLetterExchange: 'orders_dlx',
      deadLetterRoutingKey: 'orders.dead',
    },
    noAck: false,
    prefetchCount: 10,
  },
}
```
</details>

**24.** Scenario — A team proposes splitting a working monolith into eight microservices to "improve scalability". Evaluate it.
<details><summary>Answer</summary>Scalability alone is not a reason, because a stateless monolith scales horizontally by running more copies — which is cheaper than eight deployments, eight pipelines, and a message contract between every pair. The costs arrive on day one: operations that were one transaction become distributed sagas with compensating actions, a stack trace no longer spans a request, and every inter-service call is a new failure mode needing timeouts and retries. Ask what specifically cannot be solved in-process. If one component needs radically different resources, extract that one. If two teams block each other on releases, split along that seam. Eight services chosen by entity rather than by an actual constraint usually produces a distributed monolith — all the coupling, none of the independence.</details>

**25.** Scenario — Two services must both update state as part of one user action, and it must not end up half-applied. How do you handle it without distributed transactions?
<details><summary>Answer</summary>Use the outbox pattern with a saga rather than trying to coordinate a two-phase commit. Each service writes its state change and an outbox row in one local transaction, and a relay publishes from the outbox — which makes "the state changed" and "the event was published" atomic, closing the gap where one happens without the other. The second service consumes, applies its change idempotently, and emits its own event. If a later step fails, you compensate with an explicit reversing action rather than rolling back, since the earlier commit is already visible. This means accepting eventual consistency and designing the user-facing experience around it — a pending state rather than a lie. If the operation genuinely cannot tolerate a window of inconsistency, that is strong evidence the two pieces of state belong in one service.</details>

---

## Chapter Summary

- `send`/`@MessagePattern` is request–response and couples caller to callee; `emit`/`@EventPattern` is fire-and-forget and should be the default.
- `client.send()` returns a cold Observable, so an unsubscribed call transmits nothing and reports nothing.
- Transport choice comes down to persistence and replay: Redis pub/sub drops messages with no subscriber, and only Kafka lets a new consumer read history.
- Delivery is at least once, so handlers must be idempotent, the idempotency key must travel with the event, and the guarantee belongs inside the transaction that produces the effects.
- `noAck: false` is opt-in, and repeated failures must dead-letter rather than requeue, or one poison message saturates the consumer.
- Splitting a monolith buys independent scaling, team autonomy, and fault isolation, and immediately costs distributed transactions, versioned contracts, and cross-service debugging.

---

## Practice Exercises

1. Call `send()` without subscribing and confirm nothing is transmitted and nothing errors.
2. Publish to Redis pub/sub with the consumer stopped, restart it, and observe the loss.
3. Build a consumer without idempotency, force a redelivery, and measure the duplicate effects.
4. Requeue a poison message and watch the loop, then convert it to dead-lettering.
5. Set up Kafka request–response without `subscribeToResponseOf` and observe the silent hang.

---

## Mini Project: An Order Pipeline

Build an event-driven flow across two services that survives failure.

**Requirements:**

- An HTTP service accepting orders, writing state and an outbox row in one transaction.
- A relay publishing outbox rows to RabbitMQ with an event id that survives redelivery.
- A fulfilment consumer with manual acks, in-transaction idempotency, bounded retries, and dead-lettering.
- Timeouts on every synchronous call, with a test proving one service being down does not hang the other.
- A chaos test killing the consumer mid-transaction and asserting effects apply exactly once.

---

**Next:** [Chapter 27 — Queues and Background Jobs →](27-queues-jobs.md)
