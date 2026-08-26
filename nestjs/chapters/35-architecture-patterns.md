# Chapter 35 — Architecture Patterns

## Learning Objectives

- Choose between layered, hexagonal, and modular-monolith structures from real constraints
- Draw module boundaries around use cases rather than around technical layers
- Apply hexagonal architecture where it pays, and recognise where it does not
- Set up a Nest monorepo with shared libraries and enforced boundaries
- Recognise architecture that is costing more than it returns

---

## 35.1 Layered Is the Default, and It Is Fine

The structure Nest steers you toward — controller, service, repository — is a layered architecture, and for most applications it is the correct one:

```
Controller   HTTP translation
    ↓
Service      business rules, orchestration
    ↓
Repository   data access
```

Its rules are simple: each layer depends only on the one below, and nothing skips a layer or reaches upward. Those two rules give you testability and a predictable place to look for anything.

**Where layering degrades** is in how the folders get organised. Grouping by technical layer scales badly:

```
❌ src/controllers/  src/services/  src/repositories/  src/entities/
```

A feature change touches four directories, and nothing shows which files belong together. Grouping by feature keeps related code adjacent and makes the module boundary visible:

```
✅ src/tasks/{tasks.controller.ts, tasks.service.ts, task.repository.ts, dto/}
   src/users/…
   src/billing/…
```

This is also what makes a module's `exports` meaningful (→ [Ch 4](04-modules.md)) — a feature folder has a public surface, a layer folder does not.

---

## 35.2 Drawing Boundaries

The most consequential architectural decision in a Nest codebase is where module boundaries go, and the common mistake is to draw them around entities.

**One module per entity** produces boundaries that do not match how the system is used. `UserModule`, `TaskModule`, `CommentModule`, `ProjectModule` all need each other, so every feature imports most of them and the boundaries stop meaning anything.

**One module per bounded concept** works better. A "collaboration" module owning tasks, comments, and assignments has a real surface: other modules ask it to do things and never touch its internals.

Two tests for whether a boundary is right:

**Can you describe the module's job in one sentence without "and"?** "Manages tasks and users and comments and notifications" is four modules or one badly-named one.

**Does a typical feature change touch one module?** If every feature spans five modules, the boundaries are cutting across the grain of the work (→ [Ch 4](04-modules.md)).

Dependencies should also flow in one direction. Cycles between modules are a design signal, not just a `forwardRef` problem (→ [Ch 7](07-dependency-injection.md)) — the usual fixes are extracting the shared concept into a third module, or inverting one direction with an event.

---

## 35.3 Hexagonal Architecture

Hexagonal — ports and adapters — inverts the dependency between business logic and infrastructure. The domain defines interfaces (ports); infrastructure implements them (adapters); the domain depends on nothing external.

```
        HTTP adapter          Postgres adapter
             ↓                       ↑
        [ application ] → [ ports ] ←┘
             ↓
        [ domain: entities, rules ]
```

```ts
// domain — no framework, no ORM, no imports from infrastructure
export class Task {
  private constructor(
    readonly id: string,
    private status: TaskStatus,
    readonly ownerId: string,
  ) {}

  static create(props: CreateTaskProps): Task { /* invariants enforced here */ }

  complete(actorId: string): void {
    if (this.ownerId !== actorId) throw new NotTaskOwner(this.id, actorId);
    if (this.status === 'archived') throw new TaskArchived(this.id);
    this.status = 'done';
  }
}

// port — defined by the domain, implemented outside it
export abstract class TaskRepository {
  abstract findById(id: string): Promise<Task | null>;
  abstract save(task: Task): Promise<void>;
}
```

```ts
// application — orchestrates, holds no rules of its own
@Injectable()
export class CompleteTaskUseCase {
  constructor(private readonly repo: TaskRepository) {}

  async execute(taskId: string, actorId: string): Promise<void> {
    const task = await this.repo.findById(taskId);
    if (!task) throw new TaskNotFound(taskId);
    task.complete(actorId);          // the rule lives on the entity
    await this.repo.save(task);
  }
}
```

**What it buys:** business rules testable with no framework and no database, infrastructure swappable without touching the domain, and rules that live on entities rather than scattered across services.

**What it costs:** roughly twice the files, a mapping layer between domain objects and ORM entities that must be written and maintained, and indirection that makes navigation slower for anyone new.

Use it where the domain has genuine complexity — real invariants, state machines, rules a non-programmer would recognise. For CRUD over a schema, it is ceremony: an anaemic domain object mapped to and from an ORM entity that already had the same shape.

> ⚠️ **Gotcha** — The most common half-adoption is a domain that imports the ORM. A `@Entity()` decorator on a domain class, or a domain method returning a `Repository`, means the dependency is not inverted and none of the benefit is realised — while all of the file-count cost is. Either the domain imports nothing external, or you are doing layered architecture with extra folders.

---

## 35.4 Monorepos

The Nest CLI supports a monorepo with several applications and shared libraries:

```bash
nest generate app admin-api
nest generate library shared
```

```
apps/api/          apps/admin-api/       apps/worker/
libs/shared/       libs/domain/          libs/database/
```

```jsonc
// nest-cli.json
{
  "monorepo": true,
  "root": "apps/api",
  "projects": {
    "api":     { "type": "application", "root": "apps/api" },
    "worker":  { "type": "application", "root": "apps/worker" },
    "shared":  { "type": "library", "root": "libs/shared", "prefix": "@app" }
  }
}
```

A monorepo is worth it when several deployables genuinely share code — an API and a worker sharing modules and entities is the canonical case (→ [Ch 27](27-queues-jobs.md)). It gives atomic cross-cutting changes and one dependency set.

The failure mode is a shared library that becomes a dumping ground. `libs/common` importing from three feature libraries and being imported by all of them is a cycle wearing a folder structure, and it removes any independence the split was meant to provide. Keep libraries narrow, direction-enforced, and named for what they contain rather than for where things go when nobody knows.

**Enforce boundaries mechanically.** A convention nobody can violate accidentally is worth more than a document:

```jsonc
// .eslintrc — with eslint-plugin-boundaries or an import/no-restricted-paths rule
"import/no-restricted-paths": ["error", {
  "zones": [
    { "target": "./libs/domain", "from": "./libs/database" },   // domain must not import infra
    { "target": "./libs/domain", "from": "./apps" }
  ]
}]
```

---

## 35.5 Knowing When It Is Too Much

Architecture has a cost, paid on every change, and it is easy to buy more than the problem needs. Concrete signals:

**A trivial change touches many files.** Adding one field crossing a DTO, a domain entity, an ORM entity, a mapper, a port, and an adapter is six edits for one column.

**Nobody can find anything.** If new developers routinely ask where logic lives, the structure is not communicating.

**Abstractions with one implementation, forever.** A port with exactly one adapter and no realistic second one is indirection with no payoff.

**Mappers that are identity functions.** A mapper copying fifteen fields unchanged is a maintenance tax for zero translation.

The honest position is that most applications are well served by a modular monolith: feature-grouped folders, clear module boundaries, services holding business logic, and repositories at the edge. That structure supports a large codebase, is understood by everyone, and can be evolved toward hexagonal in the one or two modules where the domain genuinely warrants it — which is far cheaper than adopting it everywhere and discovering the cost.

> 💼 **Interview** — "How do you structure a large NestJS application?" A strong answer starts with feature modules and clear boundaries, mentions hexagonal as something applied selectively where the domain is complex, and names a cost of each choice. Advocating hexagonal or microservices universally reads as pattern enthusiasm rather than experience.

---

## 35.6 Common Mistakes and Best Practices

**Mistake: folders by technical layer.** A feature change touches four directories and nothing shows what belongs together.

**Mistake: one module per entity.** Everything imports everything and the boundaries stop meaning anything.

**Mistake: a domain that imports the ORM.** The dependency is not inverted, so you pay hexagonal's cost and get none of its benefit.

**Mistake: adopting hexagonal for CRUD.** Twice the files and a mapper for a domain object with no rules on it.

**Mistake: a `libs/common` that imports from feature libraries.** A cycle in folder form.

**Mistake: ports with one permanent implementation.** Indirection with no payoff.

**Mistake: enforcing boundaries only by documentation.** They will be crossed, and nothing will notice.

**Best practice: group by feature, not by layer.** It makes the module's public surface real.

**Best practice: apply hexagonal selectively.** One or two modules with genuine domain complexity, not the whole codebase.

**Best practice: enforce import direction with lint rules.** A rule that fails the build beats a convention in a wiki.

---

## 35.7 Real-World Example: A Modular Monolith With One Hexagonal Module

Most modules stay layered; the one with real invariants gets ports and adapters. The asymmetry is the design.

```
src/
├── tasks/                     layered — CRUD with light rules
│   ├── tasks.controller.ts
│   ├── tasks.service.ts
│   ├── task.repository.ts
│   └── dto/
├── users/                     layered
└── billing/                   hexagonal — real invariants, real money
    ├── domain/                no framework, no ORM imports
    │   ├── subscription.ts
    │   ├── money.ts
    │   └── errors.ts
    ├── ports/
    │   ├── subscription.repository.ts
    │   └── payment-gateway.ts
    ├── application/
    │   └── renew-subscription.use-case.ts
    └── infrastructure/
        ├── typeorm-subscription.repository.ts
        ├── stripe-payment.gateway.ts
        └── subscription.orm-entity.ts
```

```ts
// billing/domain/subscription.ts — imports nothing external, on purpose
export class Subscription {
  private constructor(
    readonly id: string,
    readonly customerId: string,
    private status: SubscriptionStatus,
    private currentPeriodEnd: Date,
    private failedAttempts: number,
  ) {}

  /**
   * Every rule about renewal lives here rather than in a service, so the
   * invariants cannot be bypassed by a second caller that forgets one.
   */
  renew(now: Date, period: BillingPeriod): RenewalDecision {
    if (this.status === 'cancelled') throw new SubscriptionCancelled(this.id);

    // Idempotence as a domain rule, not an infrastructure concern: renewing
    // twice in one period is a business error, and stating it here means
    // every entry point inherits it.
    if (this.currentPeriodEnd > now) {
      return RenewalDecision.alreadyCurrent(this.currentPeriodEnd);
    }

    // Three failures ends the subscription. Encoded once, on the entity.
    if (this.failedAttempts >= 3) {
      this.status = 'past_due';
      return RenewalDecision.abandoned('too many failed attempts');
    }
    return RenewalDecision.charge(period.priceFor(this.status));
  }

  recordPayment(paidAt: Date, period: BillingPeriod): void {
    this.status = 'active';
    this.failedAttempts = 0;
    this.currentPeriodEnd = period.endFrom(paidAt);
  }

  recordFailure(): void {
    this.failedAttempts += 1;
  }
}
```

```ts
// billing/ports/payment-gateway.ts — the domain states what it needs
export abstract class PaymentGateway {
  abstract charge(customerId: string, amount: Money): Promise<ChargeResult>;
}

// billing/ports/subscription.repository.ts
export abstract class SubscriptionRepository {
  abstract findDue(now: Date, limit: number): Promise<Subscription[]>;
  abstract save(subscription: Subscription): Promise<void>;
}
```

```ts
// billing/application/renew-subscription.use-case.ts
@Injectable()
export class RenewSubscriptionUseCase {
  constructor(
    private readonly repo: SubscriptionRepository,   // ports, not implementations
    private readonly gateway: PaymentGateway,
    private readonly events: EventPublisher,
  ) {}

  async execute(subscription: Subscription, now: Date): Promise<void> {
    const decision = subscription.renew(now, this.period);

    // The use case orchestrates and decides nothing — every branch here
    // corresponds to a decision the domain already made.
    if (decision.kind === 'already-current') return;

    if (decision.kind === 'abandoned') {
      await this.repo.save(subscription);
      await this.events.publish(new SubscriptionAbandoned(subscription.id));
      return;
    }

    const result = await this.gateway.charge(subscription.customerId, decision.amount);

    if (result.succeeded) {
      subscription.recordPayment(now, this.period);
      await this.repo.save(subscription);
      await this.events.publish(new SubscriptionRenewed(subscription.id));
    } else {
      subscription.recordFailure();
      await this.repo.save(subscription);
    }
  }
}
```

```ts
// billing/infrastructure/typeorm-subscription.repository.ts
@Injectable()
export class TypeOrmSubscriptionRepository extends SubscriptionRepository {
  constructor(
    @InjectRepository(SubscriptionOrmEntity)
    private readonly repo: Repository<SubscriptionOrmEntity>,
  ) { super(); }

  async findDue(now: Date, limit: number): Promise<Subscription[]> {
    const rows = await this.repo.find({
      where: { currentPeriodEnd: LessThan(now), status: In(['active', 'past_due']) },
      take: limit,
    });
    // The mapping is the cost of the pattern, and it is what keeps the ORM
    // out of the domain. It earns its place only where the domain has rules
    // worth protecting — which is why `tasks/` above does not do this.
    return rows.map(toDomain);
  }

  async save(subscription: Subscription): Promise<void> {
    await this.repo.save(toOrm(subscription));
  }
}
```

```ts
// billing/billing.module.ts — the only place implementations are named
@Module({
  imports: [TypeOrmModule.forFeature([SubscriptionOrmEntity])],
  providers: [
    RenewSubscriptionUseCase,
    { provide: SubscriptionRepository, useClass: TypeOrmSubscriptionRepository },
    { provide: PaymentGateway, useClass: StripePaymentGateway },
  ],
  exports: [RenewSubscriptionUseCase],     // one export: the use case
})
export class BillingModule {}
```

The decision worth defending is applying hexagonal architecture to `billing/` alone while `tasks/` and `users/` stay plainly layered. Consistency is a real virtue and applying one structure everywhere is the instinct, but the two modules have genuinely different shapes: `Subscription` has invariants that must hold across several entry points — a scheduled renewal, an admin retry, a webhook from the payment provider — and encoding those on the entity is what stops the third caller from forgetting the failed-attempt limit. `Task` has no such rules; a domain object for it would be a data holder mapped to and from an ORM entity of identical shape, and the mapper would be an identity function maintained forever. The inconsistency is visible and self-explaining: a reader opening `billing/` sees `domain/` and `ports/` and correctly infers there is complexity worth protecting, while `tasks/` announces that there is not. Uniform structure across modules of very different complexity means either over-engineering the simple ones or under-protecting the hard one, and the second is the more expensive mistake.

---

## 35.8 Questions

**1.** MCQ — Grouping folders by technical layer causes: a) better reuse b) feature changes spanning directories c) faster builds d) clearer boundaries
<details><summary>Answer</summary>**b)** And nothing indicates which files belong together.</details>

**2.** MCQ — In hexagonal architecture, the domain depends on: a) the ORM b) the framework c) nothing external d) adapters
<details><summary>Answer</summary>**c)** Adapters depend on the domain, not the reverse.</details>

**3.** MCQ — A domain class carrying an `@Entity()` decorator means: a) correct hexagonal b) the dependency is not inverted c) better performance d) nothing
<details><summary>Answer</summary>**b)** You pay the cost of the pattern without the benefit.</details>

**4.** MCQ — One module per entity typically produces: a) clean boundaries b) mutual imports everywhere c) faster tests d) fewer files
<details><summary>Answer</summary>**b)** Which makes the boundaries meaningless.</details>

**5.** MCQ — A `libs/common` importing from feature libraries is: a) normal b) a cycle in folder form c) required d) efficient
<details><summary>Answer</summary>**b)** It removes the independence the split was meant to provide.</details>

**6.** MCQ — The strongest way to enforce import direction is: a) documentation b) code review c) lint rules that fail the build d) naming conventions
<details><summary>Answer</summary>**c)** A rule nobody can bypass accidentally.</details>

**7.** Short — What are the two tests for a good module boundary?
<details><summary>Answer</summary>Can you state its job in one sentence without "and", and does a typical feature change touch one module rather than five.</details>

**8.** Short — What does hexagonal architecture actually buy?
<details><summary>Answer</summary>Business rules testable without a framework or database, swappable infrastructure, and rules that live on entities instead of being scattered across services.</details>

**9.** Short — When is hexagonal not worth it?
<details><summary>Answer</summary>For CRUD over a schema, where the domain object has no invariants and the mapper is an identity function over an ORM entity of the same shape.</details>

**10.** Short — Why is an identity-function mapper a warning sign?
<details><summary>Answer</summary>It is a permanent maintenance cost providing no translation, which means the abstraction it supports is not separating anything.</details>

**11.** Short — When is a monorepo justified?
<details><summary>Answer</summary>When several deployables genuinely share code — an API and a worker sharing modules and entities — so changes can be atomic across them.</details>

**12.** Output — Adding one database column requires editing a DTO, a domain entity, an ORM entity, a mapper, a port, and an adapter. What does that indicate?
<details><summary>Answer</summary>The architecture is costing more than it returns for this module — six edits for one column with no invariant being protected.</details>

**13.** Output — A domain method returns `Repository<Task>`. Is the dependency inverted?
<details><summary>Answer</summary>No — the domain now depends on the ORM, which is exactly what ports exist to prevent.</details>

**14.** Output — Two modules import each other and `forwardRef` makes it boot. What has been fixed?
<details><summary>Answer</summary>Only the startup error. The design cycle remains and will keep producing coupling.</details>

**15.** Output — Every feature request touches five modules. What does that say about the boundaries?
<details><summary>Answer</summary>They cut across the grain of the work, most likely drawn around entities or layers rather than use cases.</details>

**16.** Debug — New developers cannot find where a rule lives. What has the architecture failed at?
<details><summary>Answer</summary>Communication. Structure exists to make the location of things predictable; if it does not, the indirection is pure cost.</details>

**17.** Debug — A "shared" library is imported by every app and imports from two of them. Diagnosis?
<details><summary>Answer</summary>A dependency cycle expressed as folders. Split it into narrow, direction-enforced libraries.</details>

**18.** Debug — Domain tests require a database. What went wrong?
<details><summary>Answer</summary>The domain depends on infrastructure — usually an ORM import or an entity decorator — so it cannot be instantiated in isolation.</details>

**19.** Debug — Hexagonal was adopted everywhere and velocity dropped. Likely cause?
<details><summary>Answer</summary>It was applied to modules with no domain complexity, so most of the new files are mappers and anaemic entities that add edits without protecting anything.</details>

**20.** Debug — A rule about payment retries is implemented differently in three places. Which pattern would have prevented it?
<details><summary>Answer</summary>Putting the rule on the domain entity, so every caller goes through one implementation rather than reimplementing it.</details>

**21.** Code — Define a port and wire its implementation in one place.
<details><summary>Answer</summary>

```ts
export abstract class PaymentGateway {
  abstract charge(customerId: string, amount: Money): Promise<ChargeResult>;
}

@Module({
  providers: [{ provide: PaymentGateway, useClass: StripePaymentGateway }],
  exports: [PaymentGateway],
})
export class PaymentsModule {}
```
An abstract class is both the contract and a usable DI token (→ Ch 2).
</details>

**22.** Code — Put an invariant on the entity rather than in a service.
<details><summary>Answer</summary>

```ts
export class Subscription {
  complete(actorId: string): void {
    if (this.ownerId !== actorId) throw new NotOwner(this.id);
    if (this.status === 'cancelled') throw new AlreadyCancelled(this.id);
    this.status = 'active';
  }
}
```
Every caller inherits the rule; a service-level check must be repeated at each entry point.
</details>

**23.** Code — Enforce that the domain cannot import infrastructure.
<details><summary>Answer</summary>

```jsonc
"import/no-restricted-paths": ["error", {
  "zones": [
    { "target": "./src/**/domain", "from": "./src/**/infrastructure" },
    { "target": "./src/**/domain", "from": "./node_modules/typeorm" }
  ]
}]
```
A build failure is the only enforcement that survives a deadline.
</details>

**24.** Scenario — A teammate proposes migrating the whole codebase to hexagonal architecture over a quarter. Evaluate it.
<details><summary>Answer</summary>A quarter spent on structure with no feature output needs a stronger justification than consistency, and the benefit is concentrated in modules that most codebases have few of. Ask which specific problems it solves: if the answer is that business rules are scattered and untestable in one or two areas, convert those and leave the rest layered. A wholesale migration also touches every module simultaneously, which maximises merge conflicts and risk while delivering the benefit last. The cheaper path is incremental — introduce ports in the module with the most invariant-related bugs, measure whether defects and change cost actually drop, and let that evidence decide whether to continue. Architecture migrations justified by uniformity rather than by a named problem tend to stall halfway and leave two patterns in the codebase, which is worse than either.</details>

**25.** Scenario — Two teams work in one Nest codebase and constantly block each other. Is a monorepo split or microservices the answer?
<details><summary>Answer</summary>Diagnose the blocking before choosing a topology, because the three common causes have different fixes and only one of them is structural. If teams block on merge conflicts and shared files, the boundaries are wrong and the fix is module ownership within the same codebase — clear `exports`, enforced import direction, and a codeowners file. If they block on releases, because one team cannot deploy without the other's work being ready, separate deployables help and a monorepo with two applications gives that without distributed systems. Only if they block on *runtime* concerns — one team's load or failures degrading the other's service — does splitting into separate services address the actual problem, and then it is worth the distributed transactions and versioned contracts it brings (→ Ch 26). The failure mode to avoid is splitting for organisational reasons while leaving the coupling intact, which produces a distributed monolith: the same blocking, now over the network.</details>

---

## Chapter Summary

- Layered architecture with feature-grouped folders is the right default; grouping by technical layer makes every change span directories and hides what belongs together.
- Draw module boundaries around use cases, not entities — if a typical feature touches five modules, the boundaries cut across the grain of the work.
- Hexagonal inverts the dependency so the domain imports nothing external; a domain class with an ORM decorator pays the pattern's full cost and gets none of its benefit.
- Apply hexagonal selectively, in modules with genuine invariants; for CRUD it produces anaemic entities and identity-function mappers maintained forever.
- A monorepo is justified when several deployables share real code, and its failure mode is a `common` library that imports from the features that import it.
- Boundaries survive only if a lint rule fails the build; a convention in a document will be crossed and nothing will notice.

---

## Practice Exercises

1. Reorganise a layer-grouped project into feature folders and count the directories a single feature change now touches.
2. Take one module with real invariants and move the rules from its service onto its entity.
3. Introduce ports and adapters in one module, then measure the file count and edit cost of adding a field.
4. Add a lint rule forbidding domain code from importing the ORM, and fix whatever it flags.
5. Find a `common` library in a codebase you know and map its imports — look for the cycle.

---

## Mini Project: A Structured Modular Monolith

Build a codebase whose structure explains itself.

**Requirements:**

- Feature-grouped modules with boundaries drawn around use cases, each describable in one sentence.
- Exactly one module using ports and adapters, chosen because it has invariants worth protecting, with a written justification.
- Lint rules enforcing import direction, failing the build on a violation.
- A dependency graph generated from the code, with no cycles.
- A short architecture note recording what was deliberately *not* abstracted, and why.

---

**Next:** [Chapter 36 — NestJS 11 → 12 and the Ecosystem →](36-nestjs-11-to-12.md)
