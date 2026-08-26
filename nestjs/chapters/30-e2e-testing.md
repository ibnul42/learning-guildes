# Chapter 30 — Integration and E2E Testing

## Learning Objectives

- Drive a real application over HTTP with Supertest
- Isolate tests that share a database, and choose an isolation strategy deliberately
- Authenticate in tests without disabling the security you are trying to verify
- Use Testcontainers so tests run against the real database engine
- Decide what belongs in an end-to-end test and what does not

---

## 30.1 Booting a Real Application

An end-to-end test builds the actual module graph and drives it through HTTP:

```ts
// test/tasks.e2e-spec.ts
import * as request from 'supertest';

describe('Tasks (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleRef.createNestApplication();

    // The pipeline must match production, or the test verifies a different
    // application. Anything configured in main.ts must be repeated here —
    // which is the argument for APP_* providers instead (→ Ch 8).
    app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));

    await app.init();                 // no port is bound; Supertest uses the handler
  });

  afterAll(async () => {
    await app.close();                // release the pool, or Jest hangs
  });

  it('rejects an invalid body', () =>
    request(app.getHttpServer())
      .post('/tasks')
      .send({ title: '' })
      .expect(400));
});
```

`app.init()` runs everything `NestFactory.create()` does except binding a socket (→ [Ch 3](03-first-application.md)), and `app.getHttpServer()` gives Supertest the underlying handler.

> ⚠️ **Gotcha** — Forgetting `await app.close()` leaves the database pool and any timers open, so Jest reports "did not exit" and hangs the run. In CI that is a job that burns its whole timeout for a suite that actually passed.

**Configuration set in `main.ts` does not apply.** Global pipes, filters, and interceptors registered there must be repeated in the test, and any drift means the test exercises a different pipeline than production. Registering them with `APP_PIPE`/`APP_FILTER` puts them in the module graph, where the test gets them automatically — which is the strongest practical argument for that style.

---

## 30.2 Isolating Tests That Share a Database

Tests that write to a shared database interfere, and the interference is order-dependent and miserable to debug. Four strategies:

| Strategy | Isolation | Speed | Notes |
|---|---|---|---|
| Truncate between tests | good | moderate | simple; loses seed data |
| Transaction per test, rolled back | excellent | fast | breaks if the code commits |
| Schema per worker | excellent | fast after setup | works with parallel runners |
| Fresh container per suite | total | slow | strongest, use for the critical suite |

**Truncation** is the pragmatic default:

```ts
afterEach(async () => {
  const dataSource = app.get(DataSource);
  const tables = dataSource.entityMetadatas.map((e) => `"${e.tableName}"`).join(', ');
  // One statement, and RESTART IDENTITY so sequence values do not drift
  // between tests and make ids unpredictable.
  await dataSource.query(`TRUNCATE ${tables} RESTART IDENTITY CASCADE`);
});
```

**Transaction rollback** is faster and has a real limitation: any code path that opens its own transaction — which is most write paths (→ [Ch 18](18-migrations-transactions.md)) — will not participate in the test's outer transaction, so it commits for real and the rollback misses it.

**Parallel runners need separate schemas.** Jest runs test files in parallel workers by default, so with one shared database they truncate each other's data mid-test. Either set `maxWorkers: 1` for the integration suite or give each worker its own schema:

```ts
const schema = `test_${process.env.JEST_WORKER_ID ?? '1'}`;
```

---

## 30.3 Real Databases With Testcontainers

SQLite-in-memory as a stand-in for PostgreSQL is a false economy: it has different types, different constraint behaviour, no `RETURNING`, and no `SKIP LOCKED`. Tests pass and production breaks.

```bash
npm i -D @testcontainers/postgresql
```

```ts
let container: StartedPostgreSqlContainer;

beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16-alpine').start();
  process.env.DATABASE_URL = container.getConnectionUri();

  const moduleRef = await Test.createTestingModule({ imports: [AppModule] }).compile();
  app = moduleRef.createNestApplication();
  await app.init();

  // Migrations, not synchronize — so the test exercises the same schema path
  // production uses, and a broken migration fails here rather than on deploy.
  await app.get(DataSource).runMigrations();
}, 120_000);                          // image pull can be slow on a cold CI runner

afterAll(async () => {
  await app?.close();
  await container?.stop();
});
```

The cost is startup time; the benefit is that the test runs against the engine you deploy on, including its exact constraint, isolation, and type semantics.

---

## 30.4 Authentication in Tests

Three approaches, in increasing fidelity.

**Override the guard** — fast, and it removes the thing you may want to test:

```ts
.overrideGuard(JwtAuthGuard)
.useValue({ canActivate: (ctx) => {
  ctx.switchToHttp().getRequest().user = { id: 'u1', roles: ['member'] };
  return true;
} })
```

**Issue a real token** — exercises the actual guard, strategy, and claim handling:

```ts
const jwt = app.get(JwtService);
const token = jwt.sign({ sub: user.id, roles: user.roles });

await request(app.getHttpServer())
  .get('/tasks')
  .set('Authorization', `Bearer ${token}`)
  .expect(200);
```

**Log in over HTTP** — the highest fidelity and the slowest, worth it for the auth suite itself.

Use real tokens for most tests. Overriding the guard everywhere means no test ever proves that a route is protected, and "we forgot `@Public()`" and "we forgot the guard" become indistinguishable.

**Always keep tests that assert rejection:**

```ts
it('rejects an unauthenticated request', () =>
  request(app.getHttpServer()).get('/tasks').expect(401));

it('rejects a member from an admin route', () =>
  request(app.getHttpServer())
    .delete('/tasks/1')
    .set('Authorization', `Bearer ${memberToken}`)
    .expect(403));
```

Authorization fails open (→ [Ch 22](22-authorization.md)), so only these detect a missing guard.

---

## 30.5 What Belongs in an E2E Test

End-to-end tests are slow, and a suite of thousands is a suite nobody runs. Spend them on what cheaper tests cannot reach:

**Worth it:** the wiring itself — that routes exist, the pipeline applies in the right order, status codes are right, real SQL runs, migrations apply, and authorization is enforced.

**Not worth it:** business rule permutations. Twenty combinations of a pricing rule belong in unit tests against a fake; one end-to-end test proving pricing is reached through the API is enough.

The useful shape is one end-to-end test per endpoint covering the happy path and its authorization failure, plus a handful of full-journey tests through critical flows. Depth comes from the unit suite.

> 💼 **Interview** — "How do you decide between a unit and an e2e test?" A good answer is about what each can catch: unit tests cover logic branches cheaply and cannot catch wiring, e2e tests catch wiring and are too slow to enumerate branches. Naming the specific things only an e2e test finds — route registration, pipeline order, real SQL — shows the distinction is understood rather than recited.

---

## 30.6 Common Mistakes and Best Practices

**Mistake: forgetting `app.close()`.** The pool stays open, Jest hangs, and CI burns its timeout.

**Mistake: not replicating `main.ts` configuration.** The test exercises a pipeline that is not production's.

**Mistake: SQLite standing in for PostgreSQL.** Different types and constraint semantics mean tests pass where production fails.

**Mistake: parallel workers on one database.** They truncate each other's data mid-test, producing random failures.

**Mistake: overriding the auth guard everywhere.** No test then proves any route is protected.

**Mistake: transaction rollback when the code commits its own transactions.** The rollback silently misses those writes.

**Mistake: exhaustive business-rule permutations end to end.** Slow, and a fake would cover them in milliseconds.

**Best practice: register global pipeline components with `APP_*` tokens.** The test gets production's pipeline for free.

**Best practice: use Testcontainers for the real engine.** Constraint and isolation behaviour is exactly what integration tests exist to check.

**Best practice: keep explicit 401 and 403 tests for every protected route.** They are the only detection for a missing guard.

---

## 30.7 Real-World Example: An E2E Suite With Real Infrastructure

A suite running against a real PostgreSQL container, isolated per test, authenticating with real tokens, and asserting the failures rather than only the successes.

```ts
// test/setup/test-app.ts
export interface TestContext {
  app: INestApplication;
  dataSource: DataSource;
  container: StartedPostgreSqlContainer;
  tokenFor(user: Partial<User>): string;
}

export async function createTestApp(): Promise<TestContext> {
  const container = await new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('test')
    .start();

  process.env.DATABASE_URL = container.getConnectionUri();
  process.env.JWT_SECRET = 'test-secret-at-least-32-characters-long';

  const moduleRef = await Test.createTestingModule({
    imports: [AppModule],
  })
    // Only external services are faked. The database, the guards, the pipes,
    // and the filters are all real — those are what this suite exists to test.
    .overrideProvider(MailService).useClass(FakeMailService)
    .overrideProvider(PaymentGateway).useClass(FakePaymentGateway)
    .compile();

  const app = moduleRef.createNestApplication();
  app.useGlobalPipes(
    new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true }),
  );
  app.enableShutdownHooks();
  await app.init();

  const dataSource = app.get(DataSource);
  // Migrations rather than synchronize: a broken migration must fail here,
  // not during a deploy.
  await dataSource.runMigrations();

  const jwt = app.get(JwtService);

  return {
    app,
    dataSource,
    container,
    tokenFor: (user) => jwt.sign({ sub: user.id, roles: user.roles ?? ['member'] }),
  };
}

export async function truncateAll(dataSource: DataSource): Promise<void> {
  const tables = dataSource.entityMetadatas
    .filter((e) => e.tableName !== 'migrations')     // never wipe migration history
    .map((e) => `"${e.tableName}"`)
    .join(', ');
  await dataSource.query(`TRUNCATE ${tables} RESTART IDENTITY CASCADE`);
}
```

```ts
// test/tasks.e2e-spec.ts
describe('Tasks (e2e)', () => {
  let ctx: TestContext;
  let owner: User;
  let stranger: User;

  beforeAll(async () => {
    ctx = await createTestApp();
  }, 120_000);

  afterAll(async () => {
    // Order matters: close the app first so the pool drains before the
    // container disappears, otherwise shutdown logs connection errors.
    await ctx.app.close();
    await ctx.container.stop();
  });

  beforeEach(async () => {
    await truncateAll(ctx.dataSource);
    owner = await seedUser(ctx.dataSource, { roles: ['member'] });
    stranger = await seedUser(ctx.dataSource, { roles: ['member'] });
  });

  const http = () => request(ctx.app.getHttpServer());

  describe('POST /tasks', () => {
    it('creates a task and returns 201', async () => {
      const response = await http()
        .post('/tasks')
        .set('Authorization', `Bearer ${ctx.tokenFor(owner)}`)
        .send({ title: 'Write the e2e suite' })
        .expect(201);

      // Assert against the database, not just the response — a handler can
      // return a correct-looking body without persisting anything.
      const stored = await ctx.dataSource.getRepository(Task).findOneBy({ id: response.body.id });
      expect(stored).toMatchObject({ title: 'Write the e2e suite', ownerId: owner.id });
    });

    it('rejects an unauthenticated request', () =>
      http().post('/tasks').send({ title: 'x' }).expect(401));

    it('rejects unknown properties', () =>
      http()
        .post('/tasks')
        .set('Authorization', `Bearer ${ctx.tokenFor(owner)}`)
        .send({ title: 'x', ownerId: stranger.id })      // mass-assignment attempt
        .expect(400));
  });

  describe('GET /tasks/:id', () => {
    it("returns 404 for another user's task rather than 403", async () => {
      const task = await seedTask(ctx.dataSource, { ownerId: owner.id });

      // 404, not 403 — the endpoint must not confirm the id exists (→ Ch 22).
      await http()
        .get(`/tasks/${task.id}`)
        .set('Authorization', `Bearer ${ctx.tokenFor(stranger)}`)
        .expect(404);
    });
  });

  describe('concurrency', () => {
    it('does not create duplicates for a repeated idempotency key', async () => {
      const body = { title: 'Only once', idempotencyKey: randomUUID() };
      const auth = `Bearer ${ctx.tokenFor(owner)}`;

      // Fired together deliberately: a check-then-insert implementation passes
      // when these run sequentially and fails here, which is the point.
      const [first, second] = await Promise.all([
        http().post('/tasks').set('Authorization', auth).send(body),
        http().post('/tasks').set('Authorization', auth).send(body),
      ]);

      expect([first.status, second.status].every((s) => s < 400)).toBe(true);
      expect(first.body.id).toBe(second.body.id);

      const count = await ctx.dataSource.getRepository(Task).countBy({ ownerId: owner.id });
      expect(count).toBe(1);
    });
  });
});
```

The decision worth defending is issuing real tokens rather than overriding the auth guard. Overriding is faster and simpler, and it makes every authorization assertion in this file meaningless: the 401 test would pass because the override was not applied to it, the 404-not-403 test would exercise a request whose user was injected by the test rather than derived from a token, and a route that lost its guard entirely would still look protected. Since authorization fails open, the tests that matter most are exactly the ones an override neuters. Real tokens cost a `jwt.sign` per test and exercise the strategy, the claim extraction, and the guard chain as production runs them. The external services — mail and payments — are still faked, because those are genuinely outside the boundary and faking them removes network flakiness without removing anything the suite is meant to verify. The line to hold is that you fake what you do not own and run what you do.

---

## 30.8 Questions

**1.** MCQ — `app.init()` differs from `NestFactory.create()` in that it: a) skips DI b) binds no socket c) skips guards d) skips modules
<details><summary>Answer</summary>**b)** The full graph is built; only `listen()` is skipped.</details>

**2.** MCQ — Forgetting `app.close()` causes: a) a failed test b) Jest to hang c) a memory leak only d) nothing
<details><summary>Answer</summary>**b)** Open pools and timers keep the process alive.</details>

**3.** MCQ — Global pipes registered in `main.ts` in an e2e test are: a) applied b) not applied c) partially applied d) applied to GET only
<details><summary>Answer</summary>**b)** `main.ts` never runs; register with `APP_PIPE` or repeat the configuration.</details>

**4.** MCQ — Jest's default parallelism against one shared database causes: a) faster tests b) cross-test truncation c) deadlocks only d) no issue
<details><summary>Answer</summary>**b)** Workers wipe each other's data mid-test.</details>

**5.** MCQ — Transaction-rollback isolation fails when the code under test: a) reads b) opens its own transaction c) uses the ORM d) throws
<details><summary>Answer</summary>**b)** Its commits are outside the test's transaction.</details>

**6.** MCQ — SQLite substituting for PostgreSQL risks: a) slower tests b) different constraint and type semantics c) nothing d) missing migrations only
<details><summary>Answer</summary>**b)** Which is exactly what integration tests exist to verify.</details>

**7.** Short — Why register global components with `APP_*` rather than in `main.ts`?
<details><summary>Answer</summary>They become part of the module graph, so testing modules get production's pipeline automatically and cannot drift from it.</details>

**8.** Short — Why prefer real tokens to overriding the auth guard?
<details><summary>Answer</summary>Overriding neuters exactly the tests that matter — authorization fails open, so a route that lost its guard would still appear protected.</details>

**9.** Short — What should an e2e test cover that a unit test cannot?
<details><summary>Answer</summary>Wiring: route registration, pipeline order, real SQL, migrations applying, and enforcement actually being reached.</details>

**10.** Short — Why assert against the database rather than only the response?
<details><summary>Answer</summary>A handler can return a correct-looking body without persisting anything, or persist something different from what it returned.</details>

**11.** Short — Why exclude the migrations table from truncation?
<details><summary>Answer</summary>Wiping it makes the framework believe no migrations have run, so the next run re-applies them against an existing schema and fails.</details>

**12.** Output — Two Jest workers run integration files against one database, each truncating in `afterEach`. What happens?
<details><summary>Answer</summary>Intermittent failures — one worker truncates while another is mid-test, so rows vanish unpredictably.</details>

**13.** Output — An e2e test posts a body with an extra `ownerId` and the app has `forbidNonWhitelisted`. What status?
<details><summary>Answer</summary>400, naming the unexpected property — which is the mass-assignment defence working.</details>

**14.** Output — A request for another user's task hits a scoped query. What status is correct, and what does the alternative leak?
<details><summary>Answer</summary>404. A 403 would confirm the id exists, giving an enumeration oracle.</details>

**15.** Output — Two identical requests with the same idempotency key are fired with `Promise.all` against a check-then-insert implementation. What does the count assertion show?
<details><summary>Answer</summary>2 — both passed the existence check before either inserted, which is precisely the race a sequential test would miss.</details>

**16.** Debug — CI passes locally and hangs after the last test. Cause?
<details><summary>Answer</summary>`app.close()` is missing, so the database pool keeps the process alive until the job times out.</details>

**17.** Debug — Validation works in production and not in the e2e suite. Why?
<details><summary>Answer</summary>The global `ValidationPipe` is registered in `main.ts`, which the test never runs. Register it with `APP_PIPE` or add it in the test setup.</details>

**18.** Debug — Tests pass with SQLite and a unique-constraint bug reaches production. Explain.
<details><summary>Answer</summary>SQLite's constraint and type behaviour differs from PostgreSQL's, so the violation was never triggered. Use the real engine via Testcontainers.</details>

**19.** Debug — An e2e suite passes with every guard overridden, then an unprotected endpoint ships. What was wrong?
<details><summary>Answer</summary>No test could detect a missing guard, because the override made every request authenticated. Keep explicit 401 and 403 tests using real tokens.</details>

**20.** Debug — Shutdown logs connection errors after each suite. What is the likely ordering mistake?
<details><summary>Answer</summary>The container is stopped before `app.close()`, so the pool tries to drain against a database that is gone. Close the app first.</details>

**21.** Code — Write setup and teardown that do not leak.
<details><summary>Answer</summary>

```ts
beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16-alpine').start();
  process.env.DATABASE_URL = container.getConnectionUri();
  const ref = await Test.createTestingModule({ imports: [AppModule] }).compile();
  app = ref.createNestApplication();
  await app.init();
  await app.get(DataSource).runMigrations();
}, 120_000);

afterAll(async () => {
  await app?.close();          // drain the pool first
  await container?.stop();
});
```
</details>

**22.** Code — Truncate every table between tests without breaking migrations.
<details><summary>Answer</summary>

```ts
const tables = dataSource.entityMetadatas
  .filter((e) => e.tableName !== 'migrations')
  .map((e) => `"${e.tableName}"`)
  .join(', ');
await dataSource.query(`TRUNCATE ${tables} RESTART IDENTITY CASCADE`);
```
</details>

**23.** Code — Assert that a protected route rejects both anonymous and under-privileged callers.
<details><summary>Answer</summary>

```ts
it('rejects anonymous', () =>
  request(app.getHttpServer()).delete('/tasks/1').expect(401));

it('rejects a member', () =>
  request(app.getHttpServer())
    .delete('/tasks/1')
    .set('Authorization', `Bearer ${memberToken}`)
    .expect(403));
```
</details>

**24.** Scenario — A teammate proposes replacing the unit suite with end-to-end tests, since "they test what users actually do". Evaluate it.
<details><summary>Answer</summary>The premise is right and the conclusion does not follow. End-to-end tests are the only ones that catch wiring, and they are far too slow to enumerate logic branches — a pricing rule with twenty cases becomes twenty database round trips, and the suite reaches a runtime nobody tolerates. They also localise poorly: a failing e2e test tells you the endpoint is wrong, not which branch, so debugging is slower exactly when it matters. Keep both with different jobs — e2e for wiring, authorization, and critical journeys; units for branch coverage against fakes. The failure mode to guard against is the inverse too: a large unit suite with no e2e coverage passes happily while a route is unregistered.</details>

**25.** Scenario — Your integration suite is flaky: roughly one run in ten fails on a different test each time. How do you diagnose it?
<details><summary>Answer</summary>Flakiness that moves between tests is almost always shared state rather than a bug in any one test. Check parallelism against the database first, since Jest's default workers truncating a common schema produces exactly this signature. Then look for state that outlives a test: a global registered in one file, a cache not cleared, sequence values reused because truncation did not restart identity, or real timers making assertions time-dependent. Run the suite with `maxWorkers: 1` — if the flakiness disappears, it is isolation, and the fix is a schema or container per worker rather than serialising forever. Fix flaky tests rather than retrying them, because a suite people have learned to re-run is one that no longer blocks bad changes.</details>

---

## Chapter Summary

- `app.init()` builds the full graph without binding a socket, and `app.close()` is what stops the pool from hanging the runner.
- Anything configured in `main.ts` is absent from tests, which is the strongest practical reason to register global pipeline components as `APP_*` providers.
- Shared-database tests need a deliberate isolation strategy; parallel workers on one schema truncate each other and produce flakiness that moves between tests.
- Testcontainers costs startup time and buys the real engine's constraint, type, and isolation semantics — the things integration tests exist to verify.
- Overriding the auth guard neuters exactly the tests that matter, because authorization fails open; use real tokens and keep explicit 401 and 403 cases.
- Spend end-to-end tests on wiring and critical journeys, and leave rule permutations to fast unit tests against fakes.

---

## Practice Exercises

1. Remove `app.close()` and observe how the failure presents locally and in CI.
2. Run an integration suite with default parallelism against one database and count the flaky runs.
3. Swap PostgreSQL for SQLite and find a test that passes where the real engine would fail.
4. Override the auth guard globally, delete a guard from a controller, and confirm no test notices.
5. Write a concurrent idempotency test with `Promise.all` and watch a check-then-insert implementation fail it.

---

## Mini Project: An Integration Suite You Can Trust

Build a suite that catches wiring bugs and never flakes.

**Requirements:**

- A real PostgreSQL container per suite, with migrations applied rather than `synchronize`.
- Per-test isolation that works under parallel workers, plus a documented reason for the strategy chosen.
- Real JWTs for authenticated requests, with explicit 401 and 403 tests on every protected route.
- Assertions against the database as well as the response, for at least every write endpoint.
- A concurrency test per idempotent endpoint using `Promise.all`, and a CI run that fails if the suite leaves the process open.

---

**Next:** [Chapter 31 — File Uploads and Streaming →](31-file-uploads-streaming.md)
