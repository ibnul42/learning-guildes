# Chapter 6 — Providers and Services

## Learning Objectives

- Distinguish a provider from a service, and say why the broader term matters
- Decide what belongs in a controller, a service, and a repository
- Recognise why a singleton service holding request state is a production bug
- Split a service that has grown too large, using a criterion rather than a feeling
- Design a repository seam that makes a service testable without a database

---

## 6.1 Providers, Services, and the Difference

**Provider** is the framework's word: anything the container can construct and inject. **Service** is a naming convention for the most common kind of provider — a class holding business logic.

Everything below is a provider:

```ts
@Injectable() export class TasksService {}          // a service
@Injectable() export class TaskRepository {}        // a repository
@Injectable() export class JwtAuthGuard {}          // a guard
{ provide: 'CONFIG', useValue: { retries: 3 } }     // a plain value
{ provide: Clock, useFactory: () => new Clock() }   // a factory result
```

The distinction matters because Chapter 8 shows that a provider need not be a class at all. Thinking "providers are services" makes `useValue` and `useFactory` look exotic when they are ordinary.

A provider becomes usable in two steps, and both are required:

```ts
@Injectable()                                   // 1. emits metadata (→ Ch 2)
export class TasksService {}

@Module({ providers: [TasksService] })          // 2. registers it with the container
export class TasksModule {}
```

`@Injectable()` alone does nothing — an unregistered decorated class is just a class. Registration alone fails too, once the class has dependencies of its own, because there is no `design:paramtypes` metadata to read.

---

## 6.2 Constructor Injection

Nest injects through the constructor. The `private readonly` shorthand declares the property and assigns it in one move:

```ts
@Injectable()
export class TasksService {
  constructor(
    private readonly repo: TaskRepository,
    private readonly notifications: NotificationsService,
    private readonly logger: Logger,
  ) {}
}
```

`readonly` is not decoration. It prevents a method from reassigning a dependency at runtime, which is a real class of bug in long-lived singletons — and it costs nothing.

Property injection also exists, and is worth knowing mainly so you can avoid it:

```ts
@Injectable()
export class TasksService {
  @Inject(NotificationsService)                 // works, but…
  private readonly notifications: NotificationsService;
}
```

It requires an explicit token, hides dependencies from anyone reading the constructor, and makes the class harder to instantiate directly in a test. Its one legitimate use is in a base class that subclasses inherit from, where threading dependencies through every subclass constructor would be worse.

**Default-scoped providers are singletons.** One instance per application, constructed at startup, shared by every consumer:

```ts
@Injectable()
export class CounterService {
  private count = 0;                            // shared across ALL requests
  increment() { return ++this.count; }
}
```

That is correct for a counter you intend to be global, and catastrophic for anything request-specific — which §6.3 covers.

> 💼 **Interview** — "Are NestJS services singletons?" Yes, by default. The follow-up is always: "So what happens if you store the current user on `this`?" The answer is that concurrent requests overwrite each other, and the bug appears only under load.

---

## 6.3 Services Must Be Stateless

This is the highest-consequence rule in the chapter.

```ts
// ❌ A singleton holding request state. Works perfectly in development,
//    corrupts data under concurrency.
@Injectable()
export class OrderService {
  private currentUser: User;                    // shared by every request

  async process(userId: string, items: Item[]) {
    this.currentUser = await this.users.find(userId);
    await this.chargePayment(items);            // await → another request runs
    return this.createOrder(this.currentUser);  // may now be someone else's user
  }
}
```

Node handles one request at a time on the main thread, but every `await` is a yield point. Request A sets `currentUser`, awaits the payment call, and while it waits request B sets `currentUser` to a different person. When A resumes, it creates an order for B's user. The bug is invisible with one user and reliable at scale — the worst possible combination.

```ts
// ✅ State travels through parameters and return values.
@Injectable()
export class OrderService {
  async process(userId: string, items: Item[]): Promise<Order> {
    const user = await this.users.find(userId);      // a local, per-call
    await this.chargePayment(items);
    return this.createOrder(user);
  }
}
```

The rule: **instance fields hold dependencies and configuration; everything request-specific is a parameter or a local.** If a service needs genuinely per-request state, that is what `Scope.REQUEST` is for — and it has costs you should understand before reaching for it (→ [Ch 9](09-scopes-lifecycle.md)).

Legitimate singleton state does exist: a connection pool, a cache with a bounded size, a circuit breaker's failure count, a metrics counter. The test is whether the state belongs to *the application* or to *one request*.

> ⚠️ **Gotcha** — This bug does not reproduce in unit tests, which call one method at a time, and rarely in manual testing. It reproduces under concurrent load, which is why it reaches production. Reviewing for mutable non-dependency fields is far cheaper than debugging it later.

---

## 6.4 Where the Logic Goes

Three layers, with a clear division:

| Layer | Owns | Never does |
|---|---|---|
| **Controller** | HTTP: routes, status codes, DTO shapes | business rules, database access |
| **Service** | business rules, orchestration, transactions | HTTP concerns, SQL |
| **Repository** | data access and mapping | business rules |

The test for a controller is whether the logic would still be needed if the same operation were triggered by a queue message or a CLI command. If yes, it belongs in a service.

```ts
// ❌ Business rules stranded in the controller — unreachable from a worker
@Post()
async create(@Body() dto: CreateTaskDto) {
  if (dto.dueDate < new Date()) throw new BadRequestException('Past due date');
  const count = await this.tasks.countForUser(dto.userId);
  if (count >= 100) throw new ForbiddenException('Task limit reached');
  return this.tasks.create(dto);
}

// ✅ The controller translates HTTP; the service owns the rules
@Post()
create(@Body() dto: CreateTaskDto) {
  return this.tasks.create(dto);
}
```

Services calling services is normal and expected. What is not normal is a service reaching into another module's *repository* — that bypasses the owning service's rules:

```ts
// ❌ ReportsService reaching past UsersService into its data layer
constructor(private readonly userRepo: UserRepository) {}

// ✅ Go through the module's public surface
constructor(private readonly users: UsersService) {}
```

**When a service gets too big**, the useful criterion is not line count but *reasons to change*. A `UsersService` doing authentication, profile updates, avatar uploads, and billing has four unrelated reasons to change and should be four services. A 400-line service that does one thing thoroughly is fine.

---

## 6.5 The Repository Seam

A service that constructs its own database access cannot be tested without a database. Injecting the data layer creates a seam:

```ts
// The contract. An abstract class, so it doubles as a DI token (→ Ch 2).
export abstract class TaskRepository {
  abstract findById(id: string): Promise<Task | null>;
  abstract save(task: Task): Promise<Task>;
  abstract countByUser(userId: string): Promise<number>;
}
```

```ts
@Injectable()
export class TasksService {
  constructor(private readonly repo: TaskRepository) {}   // depends on the contract

  async create(dto: CreateTaskDto): Promise<Task> {
    if (await this.repo.countByUser(dto.userId) >= 100) {
      throw new ForbiddenException('Task limit reached');
    }
    return this.repo.save(Task.create(dto));
  }
}
```

Production wires the real implementation; a test wires a fake, with no database and no mocking framework:

```ts
// src/tasks/tasks.module.ts
@Module({
  providers: [TasksService, { provide: TaskRepository, useClass: TypeOrmTaskRepository }],
})
export class TasksModule {}

// in a test
const module = await Test.createTestingModule({
  providers: [TasksService, { provide: TaskRepository, useClass: InMemoryTaskRepository }],
}).compile();
```

This is worth the indirection when a service has rules worth testing in isolation. It is not worth it for a service that only forwards calls to the ORM — there, injecting the repository directly (→ [Ch 17](17-typeorm.md)) is simpler and equally testable. Add the abstraction when the logic justifies it, not on principle.

---

## 6.6 Common Mistakes and Best Practices

**Mistake: storing request data on `this`.** Concurrent requests overwrite each other at every `await`, and it only reproduces under load.

**Mistake: business rules in the controller.** They become unreachable from a queue worker, a scheduled job, or a second transport.

**Mistake: injecting another module's repository.** It bypasses the owning service's validation and invariants entirely.

**Mistake: `new SomeService()` inside a service.** The instance is unwired — its own dependencies are undefined and its lifecycle hooks never fire.

**Mistake: one service per entity, always.** It produces god services for large entities and pointless indirection for small ones; split by reason-to-change instead.

**Mistake: property injection by default.** It hides dependencies and makes direct instantiation in tests harder, for no benefit over the constructor.

**Mistake: returning ORM entities from a service.** Every consumer becomes coupled to the database schema, and lazy relations can trigger queries during serialization.

**Best practice: declare dependencies `private readonly`.** It documents them in one place and prevents accidental reassignment in a long-lived singleton.

**Best practice: make services depend on contracts when the logic warrants it.** An abstract class gives you a swap point and a real DI token in one move.

**Best practice: keep the constructor short.** More than five dependencies is a reliable signal that the class has more than one responsibility.

---

## 6.7 Real-World Example: A Service With Rules Worth Testing

An account service enforcing genuine invariants — a limit, an idempotency guarantee, and a notification that must not roll back the transaction. Every rule here is one a test can assert without a database.

```ts
// src/tasks/task.repository.ts
export abstract class TaskRepository {
  abstract findById(id: string): Promise<Task | null>;
  abstract findByIdempotencyKey(key: string): Promise<Task | null>;
  abstract countActiveByUser(userId: string): Promise<number>;
  abstract save(task: Task): Promise<Task>;
}
```

```ts
// src/tasks/tasks.service.ts
import { Injectable, Logger, ForbiddenException, NotFoundException } from '@nestjs/common';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  // Configuration and collaborators only — no request state lives on `this`.
  constructor(
    private readonly repo: TaskRepository,
    private readonly notifications: NotificationsService,
    private readonly config: TasksConfig,
  ) {}

  async create(dto: CreateTaskDto, actorId: string): Promise<Task> {
    // Idempotency first. A retried request must not create a second task,
    // and clients retry far more often than they admit.
    if (dto.idempotencyKey) {
      const existing = await this.repo.findByIdempotencyKey(dto.idempotencyKey);
      if (existing) {
        this.logger.debug(`Idempotent replay for key ${dto.idempotencyKey}`);
        return existing;
      }
    }

    const active = await this.repo.countActiveByUser(actorId);
    if (active >= this.config.maxActiveTasksPerUser) {
      throw new ForbiddenException(
        `Limit of ${this.config.maxActiveTasksPerUser} active tasks reached`,
      );
    }

    const task = await this.repo.save(
      Task.create({ ...dto, ownerId: actorId }),
    );

    // Deliberately not awaited into the caller's failure path: a notification
    // outage must not fail a task that is already durably saved.
    void this.notifications
      .taskCreated(task)
      .catch((err) => this.logger.error(`Notification failed for ${task.id}`, err));

    return task;
  }

  async complete(id: string, actorId: string): Promise<Task> {
    const task = await this.repo.findById(id);
    if (!task) throw new NotFoundException(`Task ${id} not found`);

    // The rule lives on the domain object, not scattered across services.
    task.completeBy(actorId);
    return this.repo.save(task);
  }
}
```

```ts
// src/tasks/tasks.service.spec.ts — no database, no mocking framework
describe('TasksService.create', () => {
  it('returns the existing task for a repeated idempotency key', async () => {
    const existing = Task.create({ title: 'x', ownerId: 'u1' });
    const repo = new InMemoryTaskRepository([existing]);
    const service = new TasksService(repo, new NoopNotifications(), { maxActiveTasksPerUser: 10 });

    const result = await service.create(
      { title: 'x', idempotencyKey: existing.idempotencyKey },
      'u1',
    );

    expect(result.id).toBe(existing.id);
    expect(repo.saveCount).toBe(0);        // proves nothing new was written
  });
});
```

The decision worth defending is the un-awaited notification. Awaiting it would be the obvious code and the wrong behaviour: the task is already committed, so a failure in the notification provider would surface to the client as a failed request for work that actually succeeded, and a retry would then hit the idempotency check and return the same task anyway — noise for no benefit. Detaching it makes the failure mode explicit and logged rather than propagated. The honest caveat is that this is fire-and-forget, so a dropped notification is silently lost; when delivery genuinely matters, the correct answer is not `await` but a queue with retries and a dead-letter path (→ [Ch 27](27-queues-jobs.md)). The pattern to avoid is the middle ground where a best-effort side effect is awaited into a critical path, which converts a minor outage into a major one.

---

## 6.8 Questions

**1.** MCQ — By default, a provider is instantiated: a) per request b) per module c) once per application d) per injection
<details><summary>Answer</summary>**c)** Once per application — default scope is singleton.</details>

**2.** MCQ — `@Injectable()` on an unregistered class means: a) it is globally available b) it is an ordinary class c) a startup error d) it is request-scoped
<details><summary>Answer</summary>**b)** Registration in a module's `providers` is what makes it a provider.</details>

**3.** MCQ — Which is NOT a provider? a) a service b) a guard c) a DTO class d) a `useValue` object
<details><summary>Answer</summary>**c)** A DTO is a data shape used for validation and typing; it is not registered with the container.</details>

**4.** MCQ — Storing the current user on a service instance field causes: a) a memory leak b) cross-request data corruption c) a compile error d) slower requests
<details><summary>Answer</summary>**b)** Concurrent requests overwrite the shared field at each `await`.</details>

**5.** MCQ — The best signal that a service should be split is: a) line count b) method count c) number of reasons to change d) file size
<details><summary>Answer</summary>**c)** Unrelated reasons to change indicate unrelated responsibilities.</details>

**6.** MCQ — `private readonly` on an injected dependency mainly prevents: a) injection failures b) reassignment at runtime c) circular dependencies d) memory leaks
<details><summary>Answer</summary>**b)** Reassignment, which matters in a long-lived singleton.</details>

**7.** Short — Why is "provider" the more useful term than "service"?
<details><summary>Answer</summary>Because providers include values, factories, guards, and repositories — not just classes with business logic — so the mental model extends naturally to `useValue` and `useFactory`.</details>

**8.** Short — Why does request state on a singleton only fail under load?
<details><summary>Answer</summary>The corruption requires two requests interleaving at an `await`. One request at a time never overlaps, so tests and manual checks pass.</details>

**9.** Short — What belongs in a controller?
<details><summary>Answer</summary>HTTP translation only: routing, extracting and validating the request shape, status codes, and returning a DTO. Anything still needed if the trigger were a queue message belongs in a service.</details>

**10.** Short — Why should a service not inject another module's repository?
<details><summary>Answer</summary>It bypasses the owning service's rules and invariants, so validation and limits enforced there are silently skipped.</details>

**11.** Short — When is the repository abstraction not worth it?
<details><summary>Answer</summary>When the service only forwards calls to the ORM with no rules of its own — the indirection buys nothing, and injecting the repository directly is simpler and equally testable.</details>

**12.** Output — Two concurrent requests call this. What can `getName` return for request A?
```ts
@Injectable()
class S {
  private user: User;
  async run(id: string) {
    this.user = await this.repo.find(id);
    await delay(10);
    return this.user.name;
  }
}
```
<details><summary>Answer</summary>Request B's user name. B overwrites `this.user` while A is suspended at `delay`, and A reads the field after resuming.</details>

**13.** Output — What is `count` after two requests hit this endpoint?
```ts
@Injectable() class C { count = 0; hit() { return ++this.count; } }
```
<details><summary>Answer</summary>`2`. It is one singleton shared by both requests — correct for a global counter, wrong for anything per-request.</details>

**14.** Output — A service does `const other = new OtherService();` then calls `other.doWork()`, which uses an injected repository. What happens?
<details><summary>Answer</summary>A runtime `TypeError` — the manually constructed instance received no constructor arguments, so its repository is `undefined`.</details>

**15.** Output — A test constructs `new TasksService(fakeRepo, fakeNotifications, config)` directly. Does dependency injection still work?
<details><summary>Answer</summary>There is nothing to work — you supplied the dependencies yourself, which is exactly why constructor injection makes classes testable without the container.</details>

**16.** Debug — A report occasionally shows another user's data, only in production, only at peak. Where do you look first?
<details><summary>Answer</summary>Mutable instance fields on singleton services holding per-request data. Search for assignments to `this.<field>` in service methods where the field is not a dependency.</details>

**17.** Debug — Moving a validation rule from a controller into a service caused a queue worker to start rejecting messages it previously accepted. Is this a bug?
<details><summary>Answer</summary>No — it is the fix working. The rule was always meant to apply; living in the controller meant the worker silently bypassed it.</details>

**18.** Debug — A service's `onModuleInit` never runs, though it is `@Injectable()` and listed in `providers`. What else could be wrong?
<details><summary>Answer</summary>Something is constructing it with `new` and using that instance instead of the injected one, so the container's instance — the one with hooks — is never used.</details>

**19.** Debug — Serializing a service's return value triggers unexpected database queries. Cause?
<details><summary>Answer</summary>The service returned ORM entities with lazy relations; touching those properties during serialization loads them. Map to a DTO at the service boundary.</details>

**20.** Debug — A `UsersService` with nine constructor dependencies is hard to test and slow to change. What is the underlying problem?
<details><summary>Answer</summary>It has multiple responsibilities. Split by reason to change — authentication, profile, billing — rather than trying to mock nine collaborators.</details>

**21.** Code — Refactor this to be safe under concurrency.
```ts
@Injectable()
export class ReportService {
  private rows: Row[] = [];
  async build(userId: string) {
    this.rows = await this.repo.rowsFor(userId);
    return this.format(this.rows);
  }
}
```
<details><summary>Answer</summary>

```ts
@Injectable()
export class ReportService {
  async build(userId: string): Promise<string> {
    const rows = await this.repo.rowsFor(userId);   // local, not shared
    return this.format(rows);
  }
  private format(rows: Row[]): string { /* … */ }
}
```
Passing `rows` explicitly makes the method reentrant; nothing request-specific survives on the instance.
</details>

**22.** Code — Define a repository contract and wire a fake for tests.
<details><summary>Answer</summary>

```ts
export abstract class TaskRepository {
  abstract findById(id: string): Promise<Task | null>;
  abstract save(task: Task): Promise<Task>;
}

export class InMemoryTaskRepository extends TaskRepository {
  constructor(private readonly items = new Map<string, Task>()) { super(); }
  async findById(id: string) { return this.items.get(id) ?? null; }
  async save(task: Task) { this.items.set(task.id, task); return task; }
}

// test
Test.createTestingModule({
  providers: [TasksService, { provide: TaskRepository, useClass: InMemoryTaskRepository }],
});
```
</details>

**23.** Code — Write a service method that is idempotent on a client-supplied key.
<details><summary>Answer</summary>

```ts
async create(dto: CreateDto): Promise<Order> {
  if (dto.idempotencyKey) {
    const existing = await this.repo.findByIdempotencyKey(dto.idempotencyKey);
    if (existing) return existing;              // replay: same result, no new write
  }
  return this.repo.save(Order.create(dto));
}
```
For strict correctness under concurrent replays, back this with a unique constraint on the key and catch the conflict — the read-then-write above still has a race window.
</details>

**24.** Scenario — A teammate proposes making every service depend on an abstract interface "so we can swap implementations later". Do you agree?
<details><summary>Answer</summary>Not universally. The seam earns its cost where there is logic worth testing in isolation, a genuine second implementation, or a boundary you expect to change — a payment gateway, a storage backend. Applied everywhere it doubles the file count and adds a lookup step to every navigation, in exchange for flexibility that mostly never gets used. The honest version of "swap it later" is that you can introduce the abstraction at the moment you need it, and the refactor is mechanical because the constructor already names the dependency. Add the indirection when a second implementation exists or the logic justifies the isolation, not in anticipation.</details>

**25.** Scenario — Your `OrdersService` must charge a card, save an order, and email a receipt. The email provider is unreliable. How do you structure it?
<details><summary>Answer</summary>Separate the operations by what must be atomic. The charge and the order save belong in one transaction, because an order without a payment or a payment without an order are both corruption. The email is a best-effort side effect and must sit outside that boundary — awaiting it into the request path means a mail outage fails requests whose money has already moved. For a receipt that genuinely must arrive, fire-and-forget is not enough either: enqueue it after the transaction commits, with retries and a dead-letter queue, so delivery is durable without being synchronous (→ Ch 27). The general rule is that a side effect's reliability requirement determines where it lives, and the two wrong answers are awaiting an unreliable call in a critical path and silently dropping one that mattered.</details>

---

## Chapter Summary

- Every injectable is a provider; "service" is a convention for the subset holding business logic, and keeping the broader term in mind makes `useValue` and `useFactory` unremarkable.
- Providers are singletons by default, so instance fields must hold dependencies and configuration only — request state on `this` corrupts concurrent requests at every `await` and reproduces only under load.
- Controllers translate HTTP; if the logic would still be needed when triggered by a queue or a CLI, it belongs in a service.
- Reaching into another module's repository bypasses the rules its service enforces — go through the public surface.
- Split a service by reasons to change, not by line count; a constructor past five dependencies is the reliable warning sign.
- A repository contract buys testability where real rules exist, and is pure overhead where the service only forwards to the ORM.

---

## Practice Exercises

1. Write a service that stores request state on `this`, then drive it with twenty concurrent requests and observe the corruption. Fix it and re-run.
2. Take a controller with inline business rules and move them into a service without changing the HTTP contract. Verify with the same test suite.
3. Implement a service against an abstract repository and write two implementations — in-memory and ORM-backed — sharing one test suite.
4. Find a service in your own code with more than five dependencies and split it by reason to change.
5. Implement idempotent creation with a unique constraint, and write a test that fires two concurrent identical requests.

---

## Mini Project: An Inventory Service

Build an inventory service with rules that are worth testing.

**Requirements:**

- Reserving stock must never allow the count to go negative, even under concurrent reservations.
- Reservations expire after a configurable TTL and return stock automatically.
- All persistence sits behind a repository contract, with an in-memory implementation for tests.
- No request-specific state on any service instance — enforce it with a review checklist.
- A test suite that runs entirely without a database and covers the concurrent-reservation case.

---

**Next:** [Chapter 7 — Dependency Injection Deep Dive →](07-dependency-injection.md)
