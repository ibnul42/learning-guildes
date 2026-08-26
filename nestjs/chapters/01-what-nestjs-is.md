# Chapter 1 — What NestJS Is and Why It Exists

## Learning Objectives

- Explain the specific problem NestJS solves that Express does not
- Describe what Nest adds at runtime versus what is purely compile-time TypeScript
- Choose between the Express and Fastify platform adapters, and say what leaks through either
- Argue the case *against* using NestJS for a given project

---

## 1.1 The Problem Express Leaves You With

Express is a routing library with a middleware chain. That is the whole framework. It has no opinion about where your business logic lives, how a database handle reaches a route handler, or how you replace that handle in a test.

For a hundred lines this is a feature. At ten thousand lines, every Express codebase has independently invented the same four things, badly:

```js
// The shape every growing Express app converges on
const db = require('./db');              // 1. A module-level singleton, imported everywhere
const userService = require('./services/user');

app.get('/users/:id', async (req, res) => {
  try {                                   // 2. try/catch copy-pasted into every handler
    if (!req.user) return res.status(401).json({ error: 'Unauthorized' });  // 3. inline auth
    const id = Number(req.params.id);
    if (Number.isNaN(id)) return res.status(400).json({ error: 'Bad id' }); // 4. inline validation
    res.json(await userService.find(id));
  } catch (err) {
    res.status(500).json({ error: 'Internal' });
  }
});
```

Each of those four is a decision that gets made once per handler, by whoever wrote it, at whatever standard they held that afternoon. Nothing enforces consistency. And the `require('./db')` on line 2 is the expensive one: `userService` reaches into a module-level singleton, so to test it you must either hit a real database or intercept Node's module cache.

That last problem is not a style complaint. It is the reason large Express codebases are hard to test, and testability is what the rest of this chapter is really about.

> 💼 **Interview** — "Why would you choose Nest over Express?" The weak answer is "structure" or "it's more organised". The strong answer names inversion of control: Express gives you no way to swap a dependency without swapping a module, so your test strategy degrades into mocking the module loader.

---

## 1.2 What NestJS Actually Is

NestJS is three things stacked on top of an existing HTTP library:

| Layer | What it does | Runtime cost |
|---|---|---|
| **Platform adapter** | Wraps Express (default) or Fastify behind one interface | Thin — a pass-through |
| **IoC container** | Constructs your classes and injects their dependencies | Startup only, then cached |
| **Pipeline abstraction** | Guards, interceptors, pipes, filters in a fixed order | Per request |

It is not a new HTTP server. `@nestjs/platform-express` really is Express underneath — your Express middleware still works, and `req` is still an Express request if you reach for it.

The part that matters is the middle row. Instead of importing dependencies, a class **declares** them:

```ts
// src/users/users.controller.ts
@Controller('users')
export class UsersController {
  // Nest reads this constructor's parameter types and supplies the instances
  constructor(private readonly users: UsersService) {}

  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.users.findOne(id);   // no try/catch, no manual status codes
  }
}
```

There is no `require`, no `new UsersService()`, and no service locator. `UsersController` states what it needs and receives it. That inversion is the single idea the framework is built around — everything else follows from it.

Compare the two handlers. The Express version is 12 lines and does five jobs: auth, validation, coercion, the actual lookup, and error mapping. The Nest version is one line and does one job, because the other four moved into composable, testable, reusable units (→ [Ch 10](10-request-lifecycle.md)).

**What Nest does *not* do:** it does not make your code faster, it does not remove Node's single-threaded constraints, and it does not validate anything unless you tell it to. `ValidationPipe` is opt-in (→ [Ch 14](14-pipes-validation.md)). A fresh `nest new` project validates nothing.

---

## 1.3 The Three Building Blocks

Almost everything in Nest is one of three things.

**Controllers** handle incoming requests and return responses. They should contain no business logic — a controller that computes anything is a controller doing a service's job.

**Providers** are anything injectable: services, repositories, factories, helpers. The word "provider" is broader than "service", and the distinction matters when you get to custom providers (→ [Ch 8](08-custom-providers.md)).

**Modules** group related controllers and providers, and define what is visible outside the group. They are the encapsulation boundary — the thing that makes a Nest codebase navigable at 50,000 lines.

```ts
// src/users/users.module.ts
@Module({
  controllers: [UsersController],   // instantiated, routes registered
  providers: [UsersService],        // instantiated, injectable within this module
  exports: [UsersService],          // ...and injectable by modules that import this one
})
export class UsersModule {}
```

Read `providers` and `exports` as private and public. A provider listed in `providers` but not `exports` cannot be injected outside this module, and Nest will fail at startup rather than silently resolving it. That failure is the encapsulation working (→ [Ch 4](04-modules.md)).

Modules form a graph rooted at `AppModule`. Nest walks that graph at startup, instantiates every provider once, and wires the whole object tree before the server accepts a single connection. This is why a misconfigured dependency crashes on boot instead of on the first request — and it is a genuine advantage over lazily-required Express code.

---

## 1.4 Platform Adapters: Express and Fastify

Nest ships two HTTP platforms. Express is the default because it has the deeper middleware ecosystem; Fastify is meaningfully faster on JSON-heavy workloads.

```ts
// src/main.ts — Fastify instead of the default Express
import { NestFactory } from '@nestjs/core';
import { FastifyAdapter, NestFastifyApplication } from '@nestjs/platform-fastify';

const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
);
await app.listen(3000, '0.0.0.0');   // Fastify needs an explicit host in containers
```

The adapter is *mostly* transparent. It leaks in four places, and every one of them has bitten someone in production:

| Leak | Express | Fastify |
|---|---|---|
| Raw response object | `Response` (Express) | `FastifyReply` |
| Middleware signature | `(req, res, next)` | Fastify middleware/hooks |
| File uploads | `@nestjs/platform-express` + Multer | `@fastify/multipart` |
| Ecosystem middleware | Nearly everything works | Express middleware often does not |

> ⚠️ **Gotcha** — Type your app as `NestExpressApplication` or `NestFastifyApplication` when you call platform-specific methods like `useStaticAssets()`. Without the generic, `NestFactory.create()` returns `INestApplication`, which does not declare them, and you will reach for `as any` instead of the correct type.

**NestJS 11 ships Express 5 by default**, and Express 5 changed route matching in a way that breaks upgrades:

```ts
@Get('*')            // ❌ Express 5 — a bare asterisk is no longer a valid wildcard
@Get('*splat')       // ✅ wildcards must be named
@Get(':file{.:ext}') // ✅ optional segments now use braces, not a trailing `?`
```

Nest 11 also requires **Node.js 20 or later**. Deploying it on Node 18 does not fail cleanly — you get obscure runtime errors from `path-to-regexp` and native dependencies, not a helpful version check. Pin your CI, your Dockerfile, and your serverless runtime together.

---

## 1.5 When Not to Use NestJS

A book about a framework should be able to say when to skip it. Nest is the wrong choice more often than its popularity suggests.

**A single-purpose serverless function.** Nest boots the entire module graph before handling a request. On a cold start that is real latency for an endpoint that does one thing. Use a plain handler.

**A prototype you will throw away.** The structure Nest enforces pays off over months. Over a weekend it is pure overhead.

**A team that does not know TypeScript.** Nest without types is Express with extra ceremony — the DI container works by reading emitted type metadata (→ [Ch 2](02-typescript-decorators.md)). In plain JavaScript you must hand-write `@Inject()` everywhere, and you have chosen the worst of both worlds.

**A service that is mostly one hot path.** If the whole product is a proxy or a high-throughput ingest endpoint, the pipeline abstraction costs more than it returns. Fastify alone is a better fit.

Nest earns its cost when there are **multiple developers, a schema, and a maintenance horizon measured in years**. That is a large share of backend work, which is why it is popular — but it is not all of it.

> 💼 **Interview** — Being able to name a case where you would *not* use Nest is a strong senior signal. It demonstrates you chose the framework rather than defaulting to it.

---

## 1.6 Common Mistakes and Best Practices

**Mistake: treating Nest as "Express with decorators".** It is an IoC container that happens to serve HTTP; if you never inject anything, you have adopted the syntax and skipped the benefit.

**Mistake: putting business logic in controllers.** A controller should translate HTTP into a method call and nothing else — logic there cannot be reused by a queue worker or a CLI command.

**Mistake: importing a service directly instead of injecting it.** `import { UsersService }` for its *type* is correct; calling `new UsersService()` yourself bypasses the container and gives you a second, un-wired instance.

**Mistake: assuming validation is on.** A fresh project accepts any JSON body. `ValidationPipe` must be registered explicitly.

**Mistake: upgrading to Nest 11 without reading the Express 5 route changes.** Bare `*` wildcards and `?` optional params fail at startup, and the error message points at `path-to-regexp`, not at your route.

**Mistake: running Nest 11 on Node 18.** It is unsupported, and the failures are obscure rather than explicit.

**Best practice: keep `main.ts` thin.** Global pipes, filters, and CORS belong there, but anything conditional belongs in a module so it can be tested.

**Best practice: choose the platform adapter on day one.** Migrating Express → Fastify late means rewriting every piece of middleware, every upload handler, and every `@Res()` usage.

**Best practice: let the container own every instance.** The moment you construct a provider with `new`, its own dependencies stop being injected and its lifecycle hooks stop firing.

---

## 1.7 Real-World Example: The Same Endpoint, Both Ways

A single endpoint — fetch a user by id, requiring authentication — written in Express and then in Nest. The point is not line count; it is where each responsibility lives and what a test has to fake.

```ts
// ── Express ────────────────────────────────────────────────────────────────
// Every concern is inline. To unit-test the lookup you must fake `db`,
// which means intercepting the module cache.
import { db } from './db';

app.get('/users/:id', async (req, res) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'Unauthorized' });

  let claims;
  try {
    claims = verifyJwt(token);
  } catch {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  const id = Number(req.params.id);
  if (!Number.isInteger(id) || id < 1) {
    return res.status(400).json({ error: 'id must be a positive integer' });
  }

  try {
    const user = await db.users.findById(id);
    if (!user) return res.status(404).json({ error: 'Not found' });
    const { passwordHash, ...safe } = user;      // easy to forget, and nothing checks
    return res.json(safe);
  } catch (err) {
    req.log.error({ err }, 'user lookup failed');
    return res.status(500).json({ error: 'Internal' });
  }
});
```

```ts
// ── NestJS ─────────────────────────────────────────────────────────────────
// src/users/users.controller.ts
@Controller('users')
@UseGuards(JwtAuthGuard)                          // auth: reusable, tested once
export class UsersController {
  constructor(private readonly users: UsersService) {}

  @Get(':id')
  @UseInterceptors(ClassSerializerInterceptor)    // stripping is declarative, not manual
  async findOne(
    @Param('id', ParseIntPipe) id: number,        // 400 on a non-integer, automatically
  ): Promise<UserDto> {
    const user = await this.users.findOne(id);
    if (!user) throw new NotFoundException();     // filter maps this to 404 + a body
    return user;
  }
}

// src/users/users.service.ts
@Injectable()
export class UsersService {
  // The repository arrives by injection, so a test supplies a fake without
  // touching the module cache or the network.
  constructor(private readonly repo: UserRepository) {}

  findOne(id: number): Promise<User | null> {
    return this.repo.findById(id);
  }
}

// src/users/dto/user.dto.ts
export class UserDto {
  id: number;
  email: string;

  @Exclude()                                      // cannot be forgotten at one call site
  passwordHash: string;
}
```

The Express handler's five responsibilities did not disappear — they moved to a guard, a pipe, an interceptor, a filter, and a DTO, each declared once and applied everywhere. The design decision worth defending is `@Exclude()` on the DTO rather than a destructure in the handler: the destructure is correct exactly as often as every developer remembers it, whereas the decorator makes leaking the hash require actively deleting a line. Security controls belong at the point of definition, not the point of use.

---

## 1.8 Questions

**1.** MCQ — NestJS's default HTTP platform is: a) Fastify b) Express c) Koa d) its own server
<details><summary>Answer</summary>**b)** Express, via `@nestjs/platform-express`. Fastify is a supported alternative.</details>

**2.** MCQ — The minimum Node.js version for NestJS 11 is: a) 16 b) 18 c) 20 d) 22
<details><summary>Answer</summary>**c)** Node 20. Running on 18 produces obscure runtime failures rather than a clean version error.</details>

**3.** MCQ — Which is NOT one of Nest's three core building blocks? a) controller b) provider c) module d) adapter
<details><summary>Answer</summary>**d)** Platform adapters exist, but the three building blocks are controllers, providers, and modules.</details>

**4.** MCQ — A provider listed in `providers` but not `exports` is: a) unusable b) global c) usable only inside its own module d) request-scoped
<details><summary>Answer</summary>**c)** `exports` is what makes it available to importing modules.</details>

**5.** MCQ — In NestJS 11, `@Get('*')` will: a) match everything b) match nothing c) fail at startup d) match only `/`
<details><summary>Answer</summary>**c)** Express 5 requires named wildcards — `@Get('*splat')`. A bare asterisk throws from `path-to-regexp` during route registration.</details>

**6.** MCQ — Nest instantiates providers: a) on first request b) at application startup c) per request d) lazily per injection
<details><summary>Answer</summary>**b)** At startup, by default, once each. Request and transient scopes change this (→ Ch 9).</details>

**7.** Short — State the core problem Nest solves that Express does not.
<details><summary>Answer</summary>Inversion of control. Express classes import their dependencies, so swapping one for a test means intercepting the module system; Nest classes declare dependencies and the container supplies them, so any dependency can be substituted.</details>

**8.** Short — Why is "provider" a broader term than "service"?
<details><summary>Answer</summary>A service is one kind of provider. Providers also include repositories, factories, configuration values, and anything else registered with the container — including plain values via `useValue`.</details>

**9.** Short — Name two things that leak through the platform adapter abstraction.
<details><summary>Answer</summary>Any two of: the raw request/response object types, middleware signatures, file-upload handling, and third-party middleware compatibility.</details>

**10.** Short — Why does a dependency mistake in Nest usually crash at boot rather than at request time?
<details><summary>Answer</summary>Nest walks the whole module graph and instantiates every singleton provider before the server listens, so unresolvable dependencies surface during that pass.</details>

**11.** Short — Does adding NestJS make an application faster?
<details><summary>Answer</summary>No. It adds a thin per-request pipeline on top of Express or Fastify. Its benefits are structural — testability, consistency, and encapsulation — not throughput.</details>

**12.** Output — A fresh `nest new` project receives `POST /users` with body `{"email": 12345}` and a DTO declaring `email: string`. What happens?
<details><summary>Answer</summary>The request succeeds and `email` is the number `12345`. Validation is opt-in — without a registered `ValidationPipe`, DTO types are erased at compile time and enforce nothing at runtime.</details>

**13.** Output — What does this log?
```ts
@Injectable()
export class A { constructor() { console.log('A built'); } }
// A is listed in providers; no controller injects it. App starts, no requests sent.
```
<details><summary>Answer</summary>`A built` — once, at startup. Default-scoped providers are eagerly instantiated whether or not anything has injected them yet.</details>

**14.** Output — A service is registered in `providers` *and* also constructed manually somewhere with `new UsersService(repo)`. How many instances exist?
<details><summary>Answer</summary>Two. The container's singleton and your manual one, which is not wired into the container and whose lifecycle hooks never fire.</details>

**15.** Output — With `NestFactory.create(AppModule)` (no generic), what does `app.useStaticAssets('public')` do at compile time?
<details><summary>Answer</summary>It fails to type-check. The return type is `INestApplication`, which does not declare `useStaticAssets`; you need `NestFactory.create<NestExpressApplication>(...)`.</details>

**16.** Debug — After upgrading to Nest 11 the app throws during startup with an error from `path-to-regexp`, before any request is handled. What is the likely cause?
<details><summary>Answer</summary>Express 5 route syntax. A bare `*` wildcard or a `?` optional parameter somewhere in your route decorators is now invalid. Rename wildcards (`*splat`) and convert optional segments to brace syntax.</details>

**17.** Debug — A teammate reports their new provider's `onModuleInit` never runs, though the class is clearly `@Injectable()`. What do you check first?
<details><summary>Answer</summary>Whether it is registered in a module's `providers` array and obtained by injection. An `@Injectable()` class that is imported and constructed with `new` is invisible to the container, so no lifecycle hooks fire.</details>

**18.** Debug — Tests pass locally, but in CI every request returns 500 with a stack trace mentioning native bindings. Deploys are on Node 18. Diagnose.
<details><summary>Answer</summary>Nest 11 requires Node 20+. The failure is not a clean version check — dependencies compiled against newer Node APIs fail at runtime. Align the CI image, Dockerfile, and runtime on Node 20 or 22.</details>

**19.** Debug — An endpoint returns a user object including `passwordHash`, even though the DTO marks it `@Exclude()`. What is missing?
<details><summary>Answer</summary>`ClassSerializerInterceptor` is not applied, or the service returns a plain object rather than a DTO instance. `@Exclude()` is metadata that only takes effect when `class-transformer` runs over a real instance of that class.</details>

**20.** Code — Bootstrap a Nest application on Fastify, listening on port 3000 and bound to all interfaces.
<details><summary>Answer</summary>

```ts
import { NestFactory } from '@nestjs/core';
import { FastifyAdapter, NestFastifyApplication } from '@nestjs/platform-fastify';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter(),
  );
  await app.listen(3000, '0.0.0.0');   // '0.0.0.0' is required inside containers
}
bootstrap();
```
</details>

**21.** Code — Write a module exposing `UsersService` to other modules while keeping `PasswordHasher` private to it.
<details><summary>Answer</summary>

```ts
@Module({
  controllers: [UsersController],
  providers: [UsersService, PasswordHasher],  // both instantiated here
  exports: [UsersService],                    // only this one escapes the module
})
export class UsersModule {}
```
Importing `UsersModule` elsewhere makes `UsersService` injectable; injecting `PasswordHasher` from another module fails at startup.
</details>

**22.** Code — Convert these Express 4 routes to Nest 11 (Express 5) decorators: `GET /files/*` and `GET /report/:name.:ext?`.
<details><summary>Answer</summary>

```ts
@Get('files/*path')          // wildcards must be named in Express 5
findFile(@Param('path') path: string) {}

@Get('report/:name{.:ext}')  // optional segment uses braces, not a trailing `?`
getReport(@Param('name') name: string, @Param('ext') ext?: string) {}
```
</details>

**23.** Scenario — Your team is building a webhook receiver: one endpoint, verify a signature, push to a queue, return 200. Traffic is high and latency matters. Would you use Nest?
<details><summary>Answer</summary>Probably not. One endpoint with no domain model gets almost nothing from modules and DI, while paying boot time and a per-request pipeline on a latency-sensitive hot path — Fastify alone is the better fit. The condition that flips this: if the receiver is one endpoint *today* but sits in a repo that will grow more endpoints, sharing auth and config with them, take the structure early. Retrofitting Nest onto an established Express service is far more expensive than starting with it.</details>

**24.** Scenario — You inherit a 40,000-line Express codebase. Business wants new features; you want maintainability. Do you migrate to Nest?
<details><summary>Answer</summary>Not as a big-bang rewrite — that trades a working system for months of no features. Nest can mount an existing Express app, so the viable path is strangler-fig: stand up a Nest application, route new endpoints through it, and move old ones only when you are already changing them for other reasons. Commit to migrating only if the codebase's real problem is testability and coupling; if the actual pain is a bad data model, Nest will not fix it and the rewrite will fail for reasons no framework addresses.</details>

---

## Chapter Summary

- Express gives you routing and middleware and no opinion about anything else; the gap that costs the most is inversion of control, because it determines how testable the code can be.
- Nest is a platform adapter, an IoC container, and a fixed request pipeline layered over Express or Fastify — not a new HTTP server.
- Controllers translate HTTP to method calls, providers hold logic, and modules define what is visible outside a group; `exports` is the public/private line.
- The whole module graph is instantiated at startup, which is why wiring mistakes crash on boot rather than on the first request.
- Nest 11 requires Node 20+ and ships Express 5, whose route syntax rejects bare `*` wildcards and `?` optional parameters.
- Nest earns its overhead with multiple developers and a multi-year horizon; for one-endpoint services, throwaway prototypes, and teams avoiding TypeScript, it is the wrong tool.

---

## Practice Exercises

1. Install the Nest CLI and generate a project. Read every file in `src/` and write one sentence per file explaining why it exists.
2. Take an existing Express route of your own and list its responsibilities. Map each one to the Nest construct that would own it.
3. Boot the same project on both `platform-express` and `platform-fastify`. Record every change required to make both serve an identical response.
4. Run a Nest 11 project on Node 18 in a container and capture the exact failure. Then fix it by changing only the base image.
5. Write a one-page argument, aimed at a skeptical tech lead, for *not* adopting Nest on a specific project you know. Make it concrete enough to be persuasive.

---

## Mini Project: A Framework Decision Record

Write an architecture decision record (ADR) choosing a backend framework for a real or invented service.

**Requirements:**

- State the service's constraints: team size, expected lifetime, latency budget, TypeScript familiarity.
- Compare plain Express, Fastify, and NestJS against those constraints — not in the abstract.
- Commit to one choice and name the single condition that would reverse it.
- Include the migration cost of being wrong, in each direction.
- Keep it under one page. An ADR nobody reads has failed regardless of its conclusions.

---

**Next:** [Chapter 2 — TypeScript and Decorators for NestJS →](02-typescript-decorators.md)
