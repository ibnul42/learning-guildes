# Chapter 28 — CQRS and Event-Driven Design

## Learning Objectives

- Separate commands from queries, and say what that buys and costs
- Implement handlers, events, and sagas with `@nestjs/cqrs`
- Publish events reliably using the outbox pattern
- Explain event sourcing honestly, including why most systems should not use it
- Recognise when CQRS is overkill, which is most of the time

---

## 28.1 The Idea, and Its Price

CQRS separates the model that changes state from the model that reads it. A command mutates and returns nothing meaningful; a query reads and mutates nothing.

That is the whole idea. Everything else — separate databases, event sourcing, eventual consistency — is optional and frequently mistaken for the pattern itself.

```bash
npm i @nestjs/cqrs
```

```ts
// src/tasks/commands/create-task.command.ts
export class CreateTaskCommand {
  constructor(
    public readonly title: string,
    public readonly ownerId: string,
  ) {}
}

@CommandHandler(CreateTaskCommand)
export class CreateTaskHandler implements ICommandHandler<CreateTaskCommand> {
  constructor(
    private readonly repo: TaskRepository,
    private readonly events: EventBus,
  ) {}

  async execute(command: CreateTaskCommand): Promise<string> {
    const task = await this.repo.save(Task.create(command));
    this.events.publish(new TaskCreatedEvent(task.id, task.ownerId));
    return task.id;                    // an id is acceptable; a full read model is not
  }
}
```

```ts
@Controller('tasks')
export class TasksController {
  constructor(
    private readonly commands: CommandBus,
    private readonly queries: QueryBus,
  ) {}

  @Post()
  create(@Body() dto: CreateTaskDto, @CurrentUser('id') userId: string) {
    return this.commands.execute(new CreateTaskCommand(dto.title, userId));
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.queries.execute(new GetTaskQuery(id));
  }
}
```

**The cost is visible immediately.** A create-task operation that was one service method is now a command class, a handler, a registration, and a bus dispatch. In exchange you get: handlers with exactly one reason to change, a natural seam for cross-cutting behaviour, and read paths that can be optimised independently of writes.

Those are real benefits at a certain size and pure overhead below it. §28.5 is about telling the difference.

---

## 28.2 Events and Sagas

`EventBus` decouples what happened from what should follow:

```ts
export class TaskCreatedEvent {
  constructor(public readonly taskId: string, public readonly ownerId: string) {}
}

@EventsHandler(TaskCreatedEvent)
export class SendTaskNotification implements IEventHandler<TaskCreatedEvent> {
  constructor(private readonly notifications: NotificationsService) {}

  handle(event: TaskCreatedEvent): void {
    void this.notifications.taskCreated(event.taskId);
  }
}
```

Several handlers can subscribe to one event, and the publisher does not know about any of them — which is the point. Adding an audit log means adding a handler, not editing the command handler.

**A saga** reacts to events by dispatching commands, coordinating a multi-step process:

```ts
@Injectable()
export class TaskSaga {
  @Saga()
  taskCreated = (events$: Observable<IEvent>): Observable<ICommand> =>
    events$.pipe(
      ofType(TaskCreatedEvent),
      delay(1_000),
      map((event) => new IndexTaskCommand(event.taskId)),
    );
}
```

> ⚠️ **Gotcha** — `EventBus` is **in-process and in-memory**. Events are not persisted, not retried, and lost if the process dies between publishing and handling. It is a decoupling mechanism, not a delivery guarantee. Anything that must not be lost needs a durable queue (→ [Ch 27](27-queues-jobs.md)) fed by the outbox pattern in §28.4.

---

## 28.3 Separate Read Models

The version of CQRS worth the cost is separating read *storage*, not just read code. Writes go to a normalised model; reads come from a shape built for the query:

```
Command → write model (normalised, constrained)
              ↓ event
          read model (denormalised, indexed for one query)
Query   → read model
```

A dashboard needing six joins becomes one indexed row. The list endpoint stops competing with writes for the same tables. Reads can scale independently.

The cost is **eventual consistency**, and it is not a detail:

```ts
const id = await this.commands.execute(new CreateTaskCommand(dto));
const task = await this.queries.execute(new GetTaskQuery(id));   // may be null
```

The read model has not caught up. Every consumer must handle it, which means the API contract changes:

**Return what the client needs from the command**, so it does not immediately query. Returning the id and letting the client render optimistically is the usual answer.

**Or accept the staleness explicitly** — a `202 Accepted` with a status endpoint, rather than a `201` that lies.

Choosing eventual consistency and then papering over it with a synchronous read is the worst of both: the complexity of separate models plus a race that appears under load.

---

## 28.4 Publishing Events Reliably

An event published after a transaction commits can be lost if the process dies in between. An event published inside one can be delivered for a transaction that then rolls back. Neither is acceptable, and the **outbox pattern** resolves it:

```ts
async execute(command: CreateTaskCommand): Promise<string> {
  return this.dataSource.transaction(async (manager) => {
    const task = await manager.save(Task, Task.create(command));

    // The event row and the state change commit together. Either both
    // happen or neither does — no window where one exists without the other.
    await manager.insert(OutboxEvent, {
      id: randomUUID(),
      type: 'task.created',
      payload: { taskId: task.id, ownerId: task.ownerId },
      occurredAt: new Date(),
    });

    return task.id;
  });
}
```

A relay then publishes committed rows and marks them sent:

```ts
@Injectable()
export class OutboxRelay {
  @Interval(1_000)
  async publish(): Promise<void> {
    const pending = await this.outbox.findUnpublished(100);
    for (const event of pending) {
      await this.queue.add(event.type, event.payload, { jobId: event.id });
      await this.outbox.markPublished(event.id);
    }
  }
}
```

If the relay crashes between publishing and marking, the event is published twice — so consumers must be idempotent, which they had to be anyway (→ [Ch 26](26-microservices.md)). The `jobId` makes the duplicate a no-op.

---

## 28.5 Event Sourcing, Honestly

Event sourcing stores the sequence of events as the source of truth, and derives current state by replaying them.

**What it genuinely gives you:** a complete audit trail that cannot be falsified, the ability to reconstruct state at any past moment, and new read models built from history rather than from now.

**What it costs, and this is the part usually understated:**

- **Schema evolution is permanent.** An event written three years ago must still be readable, forever, in whatever shape it had then. You will maintain upcasters.
- **Queries need projections.** "All open tasks" cannot be answered from an event log; you build and maintain a read model for every question.
- **Deletion is hard.** An append-only log conflicts directly with a legal right to erasure, and the usual remedy — crypto-shredding — must be designed in from the start.
- **Debugging changes shape.** Current state is a computation, so "why is this wrong" means replaying a history rather than reading a row.

Use it where the audit trail *is* the product — ledgers, compliance systems, anything where "how did we get here" is a business question. Do not adopt it because CQRS material presents it as the natural next step; the two are independent, and CQRS without event sourcing is the far more common and more sensible combination.

> 💼 **Interview** — "When would you use CQRS?" A strong answer separates it from event sourcing, names a concrete asymmetry between read and write load, and says that most applications should use neither. Enthusiasm for CQRS without naming its costs reads as inexperience.

---

## 28.6 Common Mistakes and Best Practices

**Mistake: treating `EventBus` as durable.** It is in-memory; events vanish on a crash.

**Mistake: adopting CQRS for a CRUD application.** Three files replace one method and nothing is gained.

**Mistake: publishing events outside the transaction.** They are lost on a crash, or delivered for work that rolled back.

**Mistake: querying immediately after a command with separate read models.** The read model has not caught up.

**Mistake: assuming CQRS implies event sourcing.** They are independent; combining them doubles the cost.

**Mistake: commands returning full read models.** It reintroduces the coupling the separation was meant to remove.

**Mistake: a saga with no failure path.** A multi-step process that cannot compensate leaves half-applied state.

**Best practice: separate read and write *code* first.** Separate storage only when a measured asymmetry justifies it.

**Best practice: use the outbox for anything that must not be lost.** It is the only way to make state change and publication atomic.

**Best practice: make eventual consistency visible in the API.** `202` with a status endpoint is honest; a `201` the client cannot immediately read is not.

---

## 28.7 Real-World Example: Commands, Outbox, and a Compensating Saga

An order flow using CQRS with a durable outbox and a saga that can undo itself — the shape worth using when the complexity is justified.

```ts
// src/orders/commands/place-order.handler.ts
@CommandHandler(PlaceOrderCommand)
export class PlaceOrderHandler implements ICommandHandler<PlaceOrderCommand> {
  constructor(private readonly dataSource: DataSource) {}

  async execute(command: PlaceOrderCommand): Promise<string> {
    return this.dataSource.transaction(async (manager) => {
      const order = await manager.save(Order, Order.create(command));

      // Reserve stock conditionally in the same transaction. Checking then
      // updating would let two orders both pass the check (→ Ch 18).
      for (const item of command.items) {
        const result = await manager
          .createQueryBuilder()
          .update(Inventory)
          .set({ available: () => 'available - :qty' })
          .where('sku = :sku AND available >= :qty', { sku: item.sku, qty: item.quantity })
          .setParameter('qty', item.quantity)
          .execute();

        if (result.affected === 0) {
          throw new ConflictException(`Insufficient stock for ${item.sku}`);
        }
      }

      // Outbox row, not EventBus: this event must survive a crash, and it
      // must not exist if the transaction rolls back.
      await manager.insert(OutboxEvent, {
        id: randomUUID(),
        aggregateId: order.id,
        type: 'order.placed',
        payload: { orderId: order.id, total: order.totalMinor, items: command.items },
      });

      return order.id;                 // an id, not a read model
    });
  }
}
```

```ts
// src/outbox/outbox.relay.ts
@Injectable()
export class OutboxRelay {
  private readonly logger = new Logger(OutboxRelay.name);

  constructor(
    private readonly dataSource: DataSource,
    @InjectQueue('domain-events') private readonly queue: Queue,
  ) {}

  @Interval(1_000)
  async relay(): Promise<void> {
    // SKIP LOCKED lets several replicas relay concurrently without any
    // instance blocking on rows another is already publishing.
    const events = await this.dataSource.query<OutboxEvent[]>(
      `SELECT * FROM outbox_events WHERE published_at IS NULL
       ORDER BY created_at LIMIT 100 FOR UPDATE SKIP LOCKED`,
    );

    for (const event of events) {
      try {
        // The event id is the job id, so a crash between publishing and
        // marking produces a duplicate that BullMQ discards.
        await this.queue.add(event.type, event.payload, { jobId: event.id });
        await this.dataSource.query(
          `UPDATE outbox_events SET published_at = now() WHERE id = $1`,
          [event.id],
        );
      } catch (err) {
        this.logger.error(`Relay failed for ${event.id}`, err as Error);
        break;                         // stop the batch; ordering is preserved
      }
    }
  }
}
```

```ts
// src/orders/sagas/order.saga.ts
@Injectable()
export class OrderSaga {
  /**
   * Payment follows placement. If it fails, the saga must undo the stock
   * reservation — the order is already committed and visible, so there is
   * nothing to roll back and the only correct move is to compensate.
   */
  @Saga()
  orderPlaced = (events$: Observable<IEvent>): Observable<ICommand> =>
    events$.pipe(
      ofType(OrderPlacedEvent),
      map((event) => new ChargePaymentCommand(event.orderId, event.total)),
    );

  @Saga()
  paymentFailed = (events$: Observable<IEvent>): Observable<ICommand> =>
    events$.pipe(
      ofType(PaymentFailedEvent),
      // Compensation, not rollback. Each step is independently committed,
      // so undoing means issuing a reversing command.
      mergeMap((event) => [
        new ReleaseInventoryCommand(event.orderId),
        new CancelOrderCommand(event.orderId, event.reason),
      ]),
    );

  @Saga()
  paymentSucceeded = (events$: Observable<IEvent>): Observable<ICommand> =>
    events$.pipe(
      ofType(PaymentSucceededEvent),
      map((event) => new FulfilOrderCommand(event.orderId)),
    );
}
```

```ts
// src/orders/orders.controller.ts
@Controller('orders')
export class OrdersController {
  constructor(private readonly commands: CommandBus, private readonly queries: QueryBus) {}

  @Post()
  @HttpCode(HttpStatus.ACCEPTED)     // 202, not 201: fulfilment is not done yet
  async place(@Body() dto: PlaceOrderDto, @CurrentUser('id') userId: string) {
    const orderId = await this.commands.execute(new PlaceOrderCommand(dto, userId));

    // Return a location the client can poll rather than a resource it cannot
    // yet read consistently. A 201 with a body would be a lie.
    return { orderId, status: 'processing', statusUrl: `/orders/${orderId}/status` };
  }
}
```

The decision worth defending is returning `202 Accepted` with a status URL rather than `201 Created` with the order body. The 201 is what a REST-shaped API would normally return and it would be dishonest here: payment and fulfilment happen asynchronously, so the resource the client receives describes a state that may be reversed seconds later by a compensating command. Clients that trust a 201 will render a confirmed order and then have to un-render it, and the ones that immediately `GET` the resource may read a projection that has not caught up. The 202 makes the asynchrony part of the contract instead of a race the client discovers in production — it says the request was accepted, not that the work is complete, and the status URL gives a defined way to find out. Whenever a system chooses eventual consistency internally, the API has to expose that choice rather than hide it, because hiding it does not remove the window; it only removes the client's ability to handle it.

---

## 28.8 Questions

**1.** MCQ — CQRS separates: a) reads from writes b) services from controllers c) modules d) databases necessarily
<details><summary>Answer</summary>**a)** Separate databases are an optional extension, not the pattern.</details>

**2.** MCQ — `EventBus` in `@nestjs/cqrs` is: a) durable b) in-memory and in-process c) Redis-backed d) transactional
<details><summary>Answer</summary>**b)** Events are lost if the process dies.</details>

**3.** MCQ — A saga reacts to events by: a) writing to the database b) dispatching commands c) returning responses d) publishing events
<details><summary>Answer</summary>**b)**</details>

**4.** MCQ — The outbox pattern makes which two operations atomic? a) two writes b) state change and event publication c) command and query d) two services
<details><summary>Answer</summary>**b)** Both commit in one transaction.</details>

**5.** MCQ — CQRS and event sourcing are: a) the same b) independent c) mutually exclusive d) always combined
<details><summary>Answer</summary>**b)** CQRS without event sourcing is the more common and usually more sensible combination.</details>

**6.** MCQ — With separate read models, reading immediately after a command may return: a) the new data b) stale or missing data c) an error d) a lock
<details><summary>Answer</summary>**b)** The projection has not caught up.</details>

**7.** Short — What does CQRS cost in a plain CRUD application?
<details><summary>Answer</summary>A command class, a handler, a registration, and a bus dispatch replace one service method, with no compensating benefit when reads and writes have similar shape and load.</details>

**8.** Short — Why can't you publish an event after committing a transaction?
<details><summary>Answer</summary>The process can die between commit and publish, leaving state changed with no event — an inconsistency nothing will detect or repair.</details>

**9.** Short — Why must outbox consumers be idempotent?
<details><summary>Answer</summary>A relay crash between publishing and marking published republishes the event, so at-least-once delivery is guaranteed rather than exceptional.</details>

**10.** Short — Name two costs of event sourcing that are usually understated.
<details><summary>Answer</summary>Permanent schema evolution — old events must remain readable forever, requiring upcasters — and deletion, since an append-only log conflicts with a right to erasure.</details>

**11.** Short — Why should a command not return a full read model?
<details><summary>Answer</summary>It couples the write path to the read shape, which is exactly the coupling the separation exists to remove; return an id instead.</details>

**12.** Output — A command handler publishes via `EventBus` and the process crashes immediately after. What did the handler receive?
<details><summary>Answer</summary>Nothing — the event was in memory and is gone, with no record that it was ever published.</details>

**13.** Output — With separate read models, `execute(CreateTaskCommand)` then `execute(GetTaskQuery)` runs immediately. What comes back?
<details><summary>Answer</summary>Possibly `null`. The projection is updated asynchronously.</details>

**14.** Output — The outbox relay publishes then crashes before marking. What happens on restart?
<details><summary>Answer</summary>The event is published again. The deterministic `jobId` makes the duplicate a no-op.</details>

**15.** Output — A saga dispatches `ChargePaymentCommand` and payment fails, with no compensating handler. What is the system state?
<details><summary>Answer</summary>An order that exists with stock reserved and no payment — half-applied, and nothing will correct it.</details>

**16.** Debug — Events fire in development and vanish under load in production. Cause?
<details><summary>Answer</summary>`EventBus` is in-memory, so restarts, crashes, and scaling events lose anything in flight. Use the outbox with a durable queue.</details>

**17.** Debug — A client creates a resource, immediately fetches it, and gets a 404. Diagnosis?
<details><summary>Answer</summary>Eventual consistency from a separate read model. Return what the client needs from the command, or make the asynchrony explicit with a 202 and a status endpoint.</details>

**18.** Debug — After adopting CQRS, feature work slowed and nobody can say what improved. What went wrong?
<details><summary>Answer</summary>It was adopted without a driver. For CRUD-shaped work the pattern adds files and indirection with no asymmetry to exploit.</details>

**19.** Debug — Two relay replicas publish the same outbox rows. Fix?
<details><summary>Answer</summary>Select with `FOR UPDATE SKIP LOCKED` so each instance claims a disjoint batch without blocking.</details>

**20.** Debug — Replaying an event log fails on events written two years ago. Cause?
<details><summary>Answer</summary>The event shape changed and no upcaster exists. In event sourcing every historical version must remain readable forever.</details>

**21.** Code — Write a command handler that persists state and an outbox event atomically.
<details><summary>Answer</summary>

```ts
async execute(cmd: CreateTaskCommand): Promise<string> {
  return this.dataSource.transaction(async (manager) => {
    const task = await manager.save(Task, Task.create(cmd));
    await manager.insert(OutboxEvent, {
      id: randomUUID(),
      type: 'task.created',
      payload: { taskId: task.id },
    });
    return task.id;
  });
}
```
</details>

**22.** Code — Write a relay safe to run on several replicas.
<details><summary>Answer</summary>

```ts
const rows = await this.dataSource.query(
  `SELECT * FROM outbox_events WHERE published_at IS NULL
   ORDER BY created_at LIMIT 100 FOR UPDATE SKIP LOCKED`,
);
for (const row of rows) {
  await this.queue.add(row.type, row.payload, { jobId: row.id });
  await this.dataSource.query(`UPDATE outbox_events SET published_at = now() WHERE id = $1`, [row.id]);
}
```
</details>

**23.** Code — Write a saga that compensates on failure.
<details><summary>Answer</summary>

```ts
@Saga()
paymentFailed = (events$: Observable<IEvent>): Observable<ICommand> =>
  events$.pipe(
    ofType(PaymentFailedEvent),
    mergeMap((e) => [
      new ReleaseInventoryCommand(e.orderId),
      new CancelOrderCommand(e.orderId, e.reason),
    ]),
  );
```
Compensation rather than rollback: the earlier steps are already committed and visible.
</details>

**24.** Scenario — A teammate proposes CQRS with event sourcing for a new internal admin tool with about thirty entities and light traffic. Evaluate it.
<details><summary>Answer</summary>Wrong tool at every axis. Light traffic means there is no read/write asymmetry to exploit, so CQRS adds three files per operation and buys nothing. Event sourcing adds permanent schema evolution, a projection to maintain for every query the tool needs, and a debugging model where current state is a computation — on a tool whose main requirement is being quick to change. Thirty entities makes it worse, not better, since the per-entity overhead multiplies. Use a plain layered service with an ORM. If a full audit trail is the actual requirement behind the proposal, an append-only audit table gives that directly, without making the audit log the source of truth for state.</details>

**25.** Scenario — Your write load is fine but a dashboard requires a six-table join that is slowing everything down. Is CQRS the answer?
<details><summary>Answer</summary>It might be, and cheaper options come first. Try a materialised view refreshed on a schedule, or a denormalised summary table updated by a trigger or a job — both give you a read model without command buses, handler classes, or a new architectural pattern across the codebase. If that is enough, stop there; you have taken the useful part of CQRS, which is a read shape independent of the write shape, and paid almost nothing for it. Full CQRS earns its cost when many read shapes diverge from the write model and the read side needs to scale or deploy separately. Either way, decide what staleness the dashboard tolerates first, because that number determines whether a five-minute refresh suffices or you need event-driven projection.</details>

---

## Chapter Summary

- CQRS separates the write model from the read model; separate databases, event sourcing, and eventual consistency are optional extensions frequently mistaken for the pattern.
- `@nestjs/cqrs`'s `EventBus` is in-memory and in-process, so it decouples code without guaranteeing delivery.
- The outbox pattern is what makes a state change and its event atomic; the relay's crash window means consumers must be idempotent regardless.
- Separate read storage buys independent optimisation and costs eventual consistency, which has to appear in the API contract rather than be hidden behind a synchronous read.
- Event sourcing gives an unfalsifiable audit trail and costs permanent schema evolution, a projection per query, and a conflict with deletion rights.
- Most applications need neither; adopt the pattern when a measured asymmetry or a genuine audit requirement justifies it, not because it is the next step in a tutorial.

---

## Practice Exercises

1. Convert one CRUD service to commands and queries and count the files before and after.
2. Publish an event with `EventBus`, kill the process immediately, and confirm it is gone.
3. Implement the outbox pattern and prove no event survives a rolled-back transaction.
4. Run two relay instances without `SKIP LOCKED` and observe duplicate publication.
5. Build a saga with a compensating path, force the failure, and verify state is fully reversed.

---

## Mini Project: An Order Flow With Compensation

Build a multi-step process that is correct when a step fails.

**Requirements:**

- Commands and queries on separate buses, with commands returning ids only.
- State changes and events written atomically via an outbox, relayed with `SKIP LOCKED`.
- A saga coordinating payment and fulfilment, with compensating commands for each failure path.
- A read model updated from events, and an API that returns 202 with a status endpoint rather than pretending the work is done.
- A chaos test killing the process at each step and asserting the system reaches a consistent terminal state.

---

**Next:** [Chapter 29 — Unit Testing →](29-unit-testing.md)
