# Chapter 29 — Unit Testing

## Learning Objectives

- Build a testing module and override providers, guards, and interceptors
- Decide when to use the Nest testing module and when plain construction is better
- Choose between fakes, stubs, and mocks from what the test is asserting
- Test pipeline components — guards, interceptors, pipes — in isolation
- Recognise tests that pass while the code is broken

---

## 29.1 Two Ways to Build a Unit

Nest's testing utilities create a real container with substituted providers:

```ts
import { Test, TestingModule } from '@nestjs/testing';

describe('TasksService', () => {
  let service: TasksService;
  let repo: jest.Mocked<TaskRepository>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        TasksService,
        {
          provide: TaskRepository,
          useValue: { findById: jest.fn(), save: jest.fn() },
        },
      ],
    }).compile();

    service = module.get(TasksService);
    repo = module.get(TaskRepository);
  });
});
```

But for a service whose dependencies you are substituting anyway, **plain construction is simpler and faster**:

```ts
const repo = new InMemoryTaskRepository();
const service = new TasksService(repo, new NoopNotifications());
```

No container, no async setup, no module resolution. That is the whole point of constructor injection (→ [Ch 6](06-providers-services.md)) — the class is testable without the framework.

Use `Test.createTestingModule` when the container is part of what you are testing: custom providers with factories, scoped providers, lifecycle hooks, or anything reading `Reflector`. Use `new` when you are testing a class's logic.

> ⚠️ **v12** — NestJS 12 makes Vitest the default runner for **new ESM projects**; CommonJS projects keep Jest. The `@nestjs/testing` API is unchanged either way, and the differences are in the runner and the mocking API (`vi.fn()` for `jest.fn()`). Nothing in this chapter's structure changes (→ [Ch 36](36-nestjs-11-to-12.md)).

---

## 29.2 Overriding

Anything registered in the graph can be replaced:

```ts
const module = await Test.createTestingModule({
  imports: [AppModule],                       // the real graph
})
  .overrideProvider(TaskRepository).useClass(InMemoryTaskRepository)
  .overrideProvider(Clock).useValue(new FixedClock(new Date('2026-01-01')))
  .overrideGuard(JwtAuthGuard).useValue({ canActivate: () => true })
  .overrideInterceptor(CacheInterceptor).useValue({ intercept: (_c, n) => n.handle() })
  .compile();
```

`overrideGuard` matters because a global guard registered with `APP_GUARD` is part of the module graph, so a testing module gets it too — which is the advantage of `APP_*` registration over `main.ts` (→ [Ch 8](08-custom-providers.md)) and also the reason unauthenticated unit tests suddenly return 401.

For scoped providers, `get` throws and `resolve` is required (→ [Ch 9](09-scopes-lifecycle.md)):

```ts
const scoped = await module.resolve(RequestScopedService);
```

**Importing `AppModule` in a unit test is usually a mistake.** It builds the entire graph, connects to real infrastructure, and turns a millisecond test into a second. Declare the two or three providers under test instead.

---

## 29.3 Fakes, Stubs, and Mocks

The three are not interchangeable, and choosing wrongly produces tests that assert the wrong thing.

**A stub** returns canned values. Use it when the test cares about the *result*:

```ts
repo.findById.mockResolvedValue(aTask({ id: '1', status: 'open' }));
expect(await service.complete('1')).toMatchObject({ status: 'done' });
```

**A mock** asserts on interactions. Use it when the *call itself* is the behaviour:

```ts
await service.create(dto, 'user-1');
expect(notifications.taskCreated).toHaveBeenCalledWith(
  expect.objectContaining({ ownerId: 'user-1' }),
);
```

**A fake** is a working implementation with a shortcut. Use it when several operations must be consistent:

```ts
export class InMemoryTaskRepository extends TaskRepository {
  private readonly items = new Map<string, Task>();
  async save(task: Task) { this.items.set(task.id, task); return task; }
  async findById(id: string) { return this.items.get(id) ?? null; }
}
```

**Prefer fakes for repositories.** A stubbed repository lets a test pass while the code saves the wrong thing, because nothing connects `save` to the next `findById`. A fake catches that, and the same fake serves every test.

**Over-mocking is the standard failure.** A test that mocks every collaborator and asserts every call is a restatement of the implementation: it fails on any refactor and passes on any behaviour change that keeps the calls the same. Assert on outcomes wherever an outcome is observable.

---

## 29.4 Testing Pipeline Components

Guards, interceptors, and pipes are classes and can be tested directly, which is far faster than driving them over HTTP.

**A guard** needs an `ExecutionContext`, which is a plain object for the parts you use:

```ts
function contextFor(user: unknown, handlerMeta: unknown = undefined): ExecutionContext {
  return {
    switchToHttp: () => ({ getRequest: () => ({ user }) }),
    getHandler: () => (() => undefined),
    getClass: () => class {},
    getType: () => 'http',
  } as unknown as ExecutionContext;
}

it('rejects a user without the required role', () => {
  const reflector = { getAllAndOverride: jest.fn().mockReturnValue(['admin']) };
  const guard = new RolesGuard(reflector as unknown as Reflector);

  expect(() => guard.canActivate(contextFor({ roles: ['member'] })))
    .toThrow(ForbiddenException);
});
```

**A pipe** is the easiest and the highest value, because validation fails open (→ [Ch 14](14-pipes-validation.md)):

```ts
const pipe = new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true });
const meta: ArgumentMetadata = { type: 'body', metatype: CreateTaskDto, data: '' };

it('rejects a nested item with a negative quantity', async () => {
  await expect(pipe.transform({ items: [{ quantity: -1 }] }, meta))
    .rejects.toThrow(BadRequestException);
});
```

**An interceptor** needs a `CallHandler`, which is one method:

```ts
const next: CallHandler = { handle: () => of({ id: 1 }) };
const result = await firstValueFrom(interceptor.intercept(ctx, next));
expect(result).toEqual({ data: { id: 1 } });
```

---

## 29.5 Tests That Pass While the Code Is Broken

Four patterns produce green suites over broken behaviour, and all four are common.

**Asserting the mock instead of the code.** `expect(repo.save).toHaveBeenCalled()` passes whether or not the saved value was correct.

**Testing only the happy path.** Validation, authorization, and idempotency all fail *open* — the bug is that something is accepted. Only a negative test detects it.

**Faking too deep.** Mocking the ORM's query builder means the test verifies your mock's fluent chain rather than the query, and it will keep passing when the query becomes wrong.

**Not awaiting.** An assertion after an un-awaited promise runs before the work does:

```ts
// ❌ Passes regardless — the rejection is unhandled and the test has finished
expect(service.create(bad)).rejects.toThrow();

// ✅
await expect(service.create(bad)).rejects.toThrow();
```

> 💼 **Interview** — "How do you know your tests are any good?" Coverage is a weak answer. A strong one is mutation testing, or the concrete habit of breaking the code deliberately and checking that a test fails — if none does, the test suite does not cover the behaviour regardless of what the coverage number says.

---

## 29.6 Common Mistakes and Best Practices

**Mistake: importing `AppModule` in a unit test.** It builds the whole graph and connects to real infrastructure.

**Mistake: `module.get()` on a scoped provider.** It throws; use `resolve()`.

**Mistake: stubbing a repository where a fake is needed.** The test cannot notice that the wrong data was saved.

**Mistake: mocking every collaborator.** The test becomes a restatement of the implementation and fails on every refactor.

**Mistake: only testing success paths.** Validation and authorization fail open, so only rejection tests catch them.

**Mistake: forgetting `await` on a rejection assertion.** The test passes no matter what happens.

**Mistake: shared mutable state between tests.** Order-dependent failures that are miserable to diagnose; rebuild fixtures in `beforeEach`.

**Best practice: construct with `new` when you are testing logic.** The container adds time and nothing else.

**Best practice: use one fake repository across the suite.** It keeps multi-step behaviour consistent and removes per-test stubbing.

**Best practice: break the code deliberately and confirm a test fails.** It is the only cheap check that a test asserts anything.

---

## 29.7 Real-World Example: A Service Suite Built on Fakes

A test suite exercising real behaviour — idempotency, limits, and a detached side effect — without a container, a database, or a mocking framework.

```ts
// test/fakes/in-memory-task.repository.ts
export class InMemoryTaskRepository extends TaskRepository {
  private readonly items = new Map<string, Task>();

  // Counters let a test assert that no write happened, which a stub cannot
  // express without asserting on the mock itself.
  saveCount = 0;

  constructor(seed: Task[] = []) {
    super();
    for (const task of seed) this.items.set(task.id, task);
  }

  async findById(id: string): Promise<Task | null> {
    return this.items.get(id) ?? null;
  }

  async findByIdempotencyKey(key: string): Promise<Task | null> {
    return [...this.items.values()].find((t) => t.idempotencyKey === key) ?? null;
  }

  async countActiveByUser(userId: string): Promise<number> {
    return [...this.items.values()].filter(
      (t) => t.ownerId === userId && t.status !== 'archived',
    ).length;
  }

  async save(task: Task): Promise<Task> {
    this.saveCount++;
    this.items.set(task.id, task);
    return task;
  }
}
```

```ts
// test/fakes/recording-notifications.ts
/**
 * A fake rather than a mock: it records calls AND can be made to fail, which
 * is what lets the suite assert that a notification outage does not fail the
 * operation it accompanies.
 */
export class RecordingNotifications implements NotificationsPort {
  readonly sent: Array<{ type: string; taskId: string }> = [];
  shouldFail = false;

  async taskCreated(task: Task): Promise<void> {
    if (this.shouldFail) throw new Error('provider down');
    this.sent.push({ type: 'created', taskId: task.id });
  }
}
```

```ts
// src/tasks/tasks.service.spec.ts
describe('TasksService', () => {
  let repo: InMemoryTaskRepository;
  let notifications: RecordingNotifications;
  let service: TasksService;

  // Rebuilt every test: shared state between tests produces order-dependent
  // failures that are far harder to diagnose than the setup cost.
  beforeEach(() => {
    repo = new InMemoryTaskRepository();
    notifications = new RecordingNotifications();
    service = new TasksService(repo, notifications, { maxActiveTasksPerUser: 3 });
  });

  describe('create', () => {
    it('persists the task and notifies', async () => {
      const task = await service.create({ title: 'Write tests' }, 'user-1');

      // Assert the OUTCOME — the stored state — not that save() was called.
      await expect(repo.findById(task.id)).resolves.toMatchObject({
        title: 'Write tests',
        ownerId: 'user-1',
      });
      expect(notifications.sent).toEqual([{ type: 'created', taskId: task.id }]);
    });

    it('is idempotent for a repeated idempotency key', async () => {
      const first = await service.create({ title: 'x', idempotencyKey: 'k1' }, 'user-1');
      const countAfterFirst = repo.saveCount;

      const second = await service.create({ title: 'x', idempotencyKey: 'k1' }, 'user-1');

      expect(second.id).toBe(first.id);
      // The strongest assertion available: no second write occurred at all.
      expect(repo.saveCount).toBe(countAfterFirst);
    });

    it('rejects creation past the active limit', async () => {
      for (let i = 0; i < 3; i++) await service.create({ title: `t${i}` }, 'user-1');

      await expect(service.create({ title: 'over' }, 'user-1'))
        .rejects.toThrow(ForbiddenException);          // note the await
    });

    it('still creates the task when notification fails', async () => {
      notifications.shouldFail = true;

      // The detached side effect must not propagate. This is exactly the
      // behaviour a mocked notifier would hide, because a mock does not throw
      // unless told to and nobody remembers to tell it to.
      const task = await service.create({ title: 'x' }, 'user-1');

      await expect(repo.findById(task.id)).resolves.not.toBeNull();
      expect(notifications.sent).toHaveLength(0);
    });
  });
});
```

```ts
// src/auth/roles.guard.spec.ts — a pipeline component, no HTTP involved
describe('RolesGuard', () => {
  const contextFor = (user: unknown): ExecutionContext =>
    ({
      switchToHttp: () => ({ getRequest: () => ({ user }) }),
      getHandler: () => () => undefined,
      getClass: () => class {},
    }) as unknown as ExecutionContext;

  const guardRequiring = (roles: string[]) =>
    new RolesGuard({ getAllAndOverride: () => roles } as unknown as Reflector);

  it('allows a user holding one of the required roles', () => {
    expect(guardRequiring(['admin', 'auditor']).canActivate(contextFor({ roles: ['auditor'] })))
      .toBe(true);
  });

  it('throws Forbidden for a user with none of them', () => {
    expect(() => guardRequiring(['admin']).canActivate(contextFor({ roles: ['member'] })))
      .toThrow(ForbiddenException);
  });

  it('throws Unauthorized when no user is present', () => {
    // Fails closed rather than crashing — the invariant from Chapter 12.
    expect(() => guardRequiring(['admin']).canActivate(contextFor(undefined)))
      .toThrow(UnauthorizedException);
  });
});
```

The decision worth defending is the fourth test — asserting that a failing notification does not fail task creation. It looks like a test of an edge case and it is actually the only test that pins down the design decision from [Chapter 6](06-providers-services.md): the notification is deliberately detached, so the task must survive its failure. With a mocked notifier this behaviour is untestable in practice, because a `jest.fn()` resolves by default and nobody writes `mockRejectedValue` for a path they are not thinking about — so the test suite would be equally green if someone later changed `void this.notifications...` to `await this.notifications...`, silently converting a mail outage into a checkout outage. A fake with a `shouldFail` switch makes the failure mode a first-class thing the suite can express. Tests should encode the decisions that are easy to reverse by accident, and detached side effects are near the top of that list.

---

## 29.8 Questions

**1.** MCQ — `Test.createTestingModule().compile()` returns: a) an app b) a testing module c) a provider d) a mock
<details><summary>Answer</summary>**b)** A `TestingModule` you can `get` providers from.</details>

**2.** MCQ — For a request-scoped provider in a test, use: a) `get()` b) `resolve()` c) `select()` d) `create()`
<details><summary>Answer</summary>**b)** `get()` throws for scoped providers.</details>

**3.** MCQ — A global guard registered with `APP_GUARD` in a testing module: a) is absent b) is present c) throws d) is disabled
<details><summary>Answer</summary>**b)** It is part of the module graph, which is why `overrideGuard` exists.</details>

**4.** MCQ — Which double is best for a repository used across several operations? a) stub b) mock c) fake d) spy
<details><summary>Answer</summary>**c)** A fake keeps `save` and `findById` consistent, which a stub cannot.</details>

**5.** MCQ — `expect(promise).rejects.toThrow()` without `await`: a) fails b) passes regardless c) throws d) times out
<details><summary>Answer</summary>**b)** The assertion is never awaited, so the test finishes first.</details>

**6.** MCQ — In NestJS 12 the default test runner becomes: a) Mocha b) Vitest c) node:test d) unchanged
<details><summary>Answer</summary>**b)** The `@nestjs/testing` API is unchanged; the mocking helpers differ.</details>

**7.** Short — When is plain `new` better than a testing module?
<details><summary>Answer</summary>When you are testing a class's logic and substituting its dependencies anyway — the container adds setup time and resolution without adding coverage.</details>

**8.** Short — Why is a stubbed repository risky?
<details><summary>Answer</summary>Nothing connects `save` to the next `findById`, so the test cannot notice that the wrong data was written.</details>

**9.** Short — Why does over-mocking make tests worse?
<details><summary>Answer</summary>The test restates the implementation, so it fails on any refactor and passes on any behaviour change that preserves the call sequence.</details>

**10.** Short — Why do validation and authorization need negative tests specifically?
<details><summary>Answer</summary>Both fail open — the bug is that something is accepted — so a happy-path test passes whether or not the check exists.</details>

**11.** Short — What does breaking the code deliberately tell you?
<details><summary>Answer</summary>Whether any test actually asserts the behaviour. If nothing fails, the suite does not cover it regardless of the coverage number.</details>

**12.** Output — A test mocks `repo.save` and asserts `toHaveBeenCalled()`. The service saves the wrong object. Does the test pass?
<details><summary>Answer</summary>Yes — the assertion is about the call, not its argument or the resulting state.</details>

**13.** Output — `module.get(RequestScopedService)` on a `Scope.REQUEST` provider. What happens?
<details><summary>Answer</summary>It throws — there is no single instance. Use `await module.resolve(...)`.</details>

**14.** Output — A unit test imports `AppModule` with no database available. What happens?
<details><summary>Answer</summary>The testing module fails to compile, or hangs, because the real graph tries to connect to real infrastructure.</details>

**15.** Output — A service is changed from `void notify()` to `await notify()`, and the notifier is a `jest.fn()`. Does any test fail?
<details><summary>Answer</summary>No — the mock resolves by default, so the changed failure semantics are invisible until production.</details>

**16.** Debug — Tests pass individually and fail when run together. Cause?
<details><summary>Answer</summary>Shared mutable state between tests. Rebuild fixtures in `beforeEach` rather than declaring them once.</details>

**17.** Debug — Every controller test returns 401 after adding a global guard. Fix?
<details><summary>Answer</summary>`overrideGuard(JwtAuthGuard).useValue({ canActivate: () => true })`. The `APP_GUARD` provider is in the graph the testing module builds.</details>

**18.** Debug — Coverage is 92% and a production bug slipped through validation. What does that tell you?
<details><summary>Answer</summary>Coverage measures lines executed, not assertions made. The validation path was executed by happy-path tests without any test asserting rejection.</details>

**19.** Debug — A test mocking a query builder keeps passing after the query became wrong. Why?
<details><summary>Answer</summary>It asserts the mock's fluent chain, not the resulting SQL. Use a fake repository, or an integration test against a real database (→ Ch 30).</details>

**20.** Debug — A guard test throws `Cannot read properties of undefined (reading 'getRequest')`. Cause?
<details><summary>Answer</summary>The fake `ExecutionContext` does not implement the method the guard calls. Provide the subset of the interface actually used.</details>

**21.** Code — Test a guard without HTTP.
<details><summary>Answer</summary>

```ts
const ctx = {
  switchToHttp: () => ({ getRequest: () => ({ user: { roles: ['member'] } }) }),
  getHandler: () => () => undefined,
  getClass: () => class {},
} as unknown as ExecutionContext;

const guard = new RolesGuard({ getAllAndOverride: () => ['admin'] } as unknown as Reflector);
expect(() => guard.canActivate(ctx)).toThrow(ForbiddenException);
```
</details>

**22.** Code — Test that a `ValidationPipe` rejects invalid nested data.
<details><summary>Answer</summary>

```ts
const pipe = new ValidationPipe({ whitelist: true, transform: true });
const meta: ArgumentMetadata = { type: 'body', metatype: CreateOrderDto, data: '' };

await expect(pipe.transform({ items: [{ quantity: -1 }] }, meta))
  .rejects.toThrow(BadRequestException);
```
This fails if `@Type()` is missing — which an end-to-end happy-path test would not catch.
</details>

**23.** Code — Override a provider and a global guard in one testing module.
<details><summary>Answer</summary>

```ts
const module = await Test.createTestingModule({ imports: [TasksModule] })
  .overrideProvider(TaskRepository).useClass(InMemoryTaskRepository)
  .overrideGuard(JwtAuthGuard).useValue({ canActivate: () => true })
  .compile();
```
</details>

**24.** Scenario — A teammate proposes a rule that every service method must have a unit test with all dependencies mocked. Evaluate it.
<details><summary>Answer</summary>The coverage goal is reasonable and the mocking mandate is counterproductive. Mocking every dependency guarantees tests that mirror the implementation: they assert call sequences, break on refactors that change nothing observable, and stay green when behaviour changes but the calls do not. Fakes for repositories and ports give the same isolation while letting assertions be about outcomes, which is what you actually want to protect. A per-method rule also drives tests for trivial pass-throughs while leaving the interesting cases — concurrency, idempotency, failure paths — untested, because those do not map to a single method. Set the expectation on behaviour coverage instead, and verify it by breaking code deliberately rather than by counting.</details>

**25.** Scenario — Your suite runs in eleven minutes and developers have stopped running it locally. How do you fix that?
<details><summary>Answer</summary>Find out what the time is actually spent on before optimising, because the usual cause is a small number of tests doing far more than a unit test should — typically importing `AppModule`, connecting to a real database, or awaiting real timers. Those should be reclassified as integration tests and moved to a separate command that runs in CI, leaving a fast unit suite developers will run on save. Replace real waits with fake timers, and replace container-based setup with plain construction wherever the container is not under test. If the slow tests are genuinely valuable integration coverage, keep them and parallelise by test file with an isolated schema per worker (→ Ch 30). The goal is not one fast suite but two suites with different contracts: one that must stay under a few seconds because it runs constantly, and one that may take minutes because it runs on push.</details>

---

## Chapter Summary

- Constructor injection means most services can be tested with `new` and no container; reserve `Test.createTestingModule` for when the container itself is part of the behaviour.
- `APP_*`-registered globals appear in testing modules, which is why they are testable at all and why `overrideGuard` exists.
- Fakes beat stubs for repositories because they keep writes and reads consistent, so a test can notice that the wrong thing was saved.
- Over-mocking produces a test that restates the implementation: it breaks on refactors and survives behaviour changes.
- Validation, authorization, and idempotency fail open, so only negative tests detect their absence.
- A missing `await` on a rejection assertion makes the test pass unconditionally, and coverage will never tell you.

---

## Practice Exercises

1. Write the same service test twice — with a testing module and with `new` — and compare setup time.
2. Replace a stubbed repository with a fake and find an assertion the stub could not express.
3. Delete a validation decorator and confirm which of your tests fail. If none do, write the one that should.
4. Remove `await` from a `rejects` assertion and confirm the test still passes.
5. Run mutation testing over one service and compare the score with its line coverage.

---

## Mini Project: A Test Suite That Catches Real Bugs

Build a suite verified by deliberate breakage rather than by coverage.

**Requirements:**

- Fakes for every port — repository, notifications, clock — shared across the suite.
- Negative tests for every validation rule, authorization rule, and idempotency guard.
- Pipeline components tested directly: a guard, a pipe, and an interceptor, with no HTTP.
- A documented list of deliberate mutations (remove a decorator, invert a condition, drop an await) with the test that catches each.
- The whole suite under five seconds, with anything slower moved to an integration command.

---

**Next:** [Chapter 30 — Integration and E2E Testing →](30-e2e-testing.md)
