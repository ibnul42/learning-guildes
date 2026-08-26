# Chapter 3 — Your First Application

## Learning Objectives

- Generate a project with the Nest CLI and explain the purpose of every file it creates
- Describe what `NestFactory.create()` does before the server accepts a connection
- Configure `main.ts` for production: validation, CORS, versioning, and graceful shutdown
- Trace a single HTTP request from the socket to your method and back
- Use the CLI's generators and schematics rather than writing boilerplate by hand

---

## 3.1 Generating a Project

Install the CLI globally, then scaffold:

```bash
npm i -g @nestjs/cli
nest new task-api          # prompts for a package manager, then installs
cd task-api
npm run start:dev          # watch mode on http://localhost:3000
```

`nest new` is not just `mkdir`. It generates a TypeScript configuration that already has `experimentalDecorators` and `emitDecoratorMetadata` enabled (→ [Ch 2](02-typescript-decorators.md)), an ESLint and Prettier setup, a Jest configuration for unit and e2e tests, and a working four-file application. Reproducing that by hand is an afternoon.

The scripts you will actually use:

| Script | What it does |
|---|---|
| `start:dev` | Watch mode, recompiles on change |
| `start:debug` | Watch mode with the Node inspector attached |
| `start:prod` | Runs the compiled `dist/main` — no TypeScript at runtime |
| `build` | Compiles to `dist/` |
| `test` / `test:watch` / `test:cov` | Jest unit tests |
| `test:e2e` | Jest against `test/*.e2e-spec.ts` with a separate config |
| `lint` / `format` | ESLint with `--fix`, and Prettier |

> ⚠️ **Gotcha** — `start:dev` runs the compiler in watch mode, which does not stop the server when a rebuild fails: it reports the error and keeps serving the **last output that compiled**. So a dev server that looks healthy can be running code from several edits ago while `npm run build` fails outright. Run `build` in CI, not just `test`.

---

## 3.2 What Every File Does

```
task-api/
├── src/
│   ├── main.ts                    entry point — creates and starts the app
│   ├── app.module.ts              the root module; everything hangs off this
│   ├── app.controller.ts          a sample controller
│   ├── app.controller.spec.ts     its unit test
│   └── app.service.ts             a sample provider
├── test/
│   ├── app.e2e-spec.ts            end-to-end test against a real HTTP server
│   └── jest-e2e.json              separate Jest config for e2e
├── nest-cli.json                  CLI/build configuration, monorepo settings
├── tsconfig.json                  decorators on, target, paths
├── tsconfig.build.json            excludes tests from the production build
├── eslint.config.mjs              flat-config ESLint (Nest 11)
└── package.json
```

Four of these deserve more than a line.

**`nest-cli.json`** controls the CLI's behaviour, and two settings matter early:

```json
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "deleteOutDir": true,        // clear dist/ before each build — prevents stale-file bugs
    "webpack": false             // set true for HMR; v12 replaces webpack with Rspack
  }
}
```

**`tsconfig.build.json`** exists so `dist/` does not contain your tests. If you have ever wondered why a production image is larger than expected, check whether the build actually used this file.

**`app.module.ts`** is the root of the graph Nest walks at startup:

```ts
// src/app.module.ts
@Module({
  imports: [],                    // other modules whose exports this module can use
  controllers: [AppController],   // instantiated; their routes get registered
  providers: [AppService],        // instantiated; injectable within this module
})
export class AppModule {}
```

**`main.ts`** is the entry point, and the default is deliberately minimal:

```ts
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

Note what is *not* there: no validation, no CORS policy, no security headers, no shutdown handling. §3.5 fixes that. A fresh Nest app is a starting point, not a secure default.

---

## 3.3 What `NestFactory.create()` Actually Does

Two lines of `main.ts` hide the entire startup sequence. In order:

1. **Scan the module graph.** Starting at `AppModule`, Nest follows every `imports` edge and builds a complete map of modules, controllers, and providers.
2. **Resolve dependencies.** For each provider it reads `design:paramtypes` and looks up each token in the module's own providers, then in the exports of its imported modules. An unresolvable token fails here — at startup, with a message naming the class and the parameter index.
3. **Instantiate providers.** Every default-scoped provider is constructed exactly once, dependencies first.
4. **Run initialization hooks.** `onModuleInit` on every provider, then `onApplicationBootstrap` (→ [Ch 9](09-scopes-lifecycle.md)).
5. **Register routes.** Nest reads the metadata that `@Controller()` and `@Get()` left behind and registers real Express or Fastify routes.
6. **Listen.** Only now, at `app.listen()`, does the process accept connections.

Steps 1–5 happen inside `create()`; only step 6 is `listen()`. That separation is what makes integration testing pleasant — `Test.createTestingModule()` performs steps 1–4 and hands you a working object graph with no socket bound (→ [Ch 29](29-unit-testing.md)).

> 💼 **Interview** — "What happens when a NestJS application starts?" Naming the ordering — graph scan, dependency resolution, instantiation, hooks, route registration, then listen — separates people who have debugged a startup failure from people who have only written controllers.

---

## 3.4 Generating a Resource

The CLI's generators are not a beginner convenience; they are how experienced Nest developers avoid boilerplate.

```bash
nest generate resource tasks        # or: nest g res tasks
```

It prompts for a transport layer (REST, GraphQL, microservice, WebSockets) and whether to generate CRUD entry points, then produces a module, controller, service, DTOs, and specs — all wired together and registered in `AppModule`.

Finer-grained generators:

| Command | Short | Produces |
|---|---|---|
| `nest g module tasks` | `nest g mo` | module, registered in `AppModule` |
| `nest g controller tasks` | `nest g co` | controller + spec, registered in its module |
| `nest g service tasks` | `nest g s` | provider + spec, registered in its module |
| `nest g guard auth` | `nest g gu` | guard + spec |
| `nest g interceptor logging` | `nest g in` | interceptor + spec |
| `nest g pipe trim` | `nest g pi` | pipe + spec |
| `nest g filter http-exception` | `nest g f` | filter + spec |

Two flags worth knowing:

```bash
nest g res tasks --dry-run      # -d: print what would change, write nothing
nest g co tasks --no-spec       # skip the test file (you will regret this)
nest g res tasks --flat         # no extra directory
```

The reason to prefer generators over copy-paste is registration. `nest g service` adds the provider to the nearest module's `providers` array for you — and "forgot to register it" is the single most common cause of the dependency-resolution error you will otherwise spend twenty minutes on.

---

## 3.5 A Production-Shaped `main.ts`

The generated `main.ts` is four lines. A real one is closer to this, and every addition below closes a specific hole:

```ts
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe, VersioningType, Logger } from '@nestjs/common';
import helmet from 'helmet';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    bufferLogs: true,               // hold logs until a custom logger is attached
  });

  app.use(helmet());                // security headers; must precede route handling

  app.enableCors({
    origin: process.env.CORS_ORIGINS?.split(',') ?? false,   // never `true` in production
    credentials: true,
  });

  app.setGlobalPrefix('api', { exclude: ['health'] });       // /api/*, but /health stays put

  app.enableVersioning({ type: VersioningType.URI, defaultVersion: '1' });   // /api/v1/*

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,              // strip properties with no decorator
      forbidNonWhitelisted: true,   // ...and 400 if any were sent
      transform: true,              // instantiate DTO classes, coerce primitives
      transformOptions: { enableImplicitConversion: false },   // explicit beats magic
    }),
  );

  app.enableShutdownHooks();        // required for onApplicationShutdown to ever fire

  const port = process.env.PORT ?? 3000;
  await app.listen(port);
  Logger.log(`Listening on ${await app.getUrl()}`, 'Bootstrap');
}

void bootstrap();
```

The three that people most often get wrong:

**`whitelist` without `forbidNonWhitelisted`** silently drops unknown fields. That is safe but confusing — a client sends `{ "isAdmin": true }`, gets a 201, and believes it worked. With both flags, they get a 400 naming the offending property.

**`enableShutdownHooks()` is opt-in.** Without it, `onApplicationShutdown` and `onModuleDestroy` never run on `SIGTERM`, so your database pool is not drained and in-flight requests are cut off mid-response. In Kubernetes this shows up as intermittent 502s during every deploy.

**`origin: true` reflects any origin.** It is a development convenience that reaches production regularly. Read the allowed list from configuration.

> ⚠️ **Gotcha** — NestJS 11 **reversed the order of termination lifecycle hooks**. Shutdown now unwinds in the opposite direction to previous versions, so a module that assumed a dependency was still alive during its own teardown can now find it already destroyed. If you are upgrading, audit anything that does work in `onModuleDestroy`.

---

## 3.6 A Request, End to End

Follow `GET /api/v1/tasks/42` through a running application:

```ts
// src/tasks/tasks.controller.ts
@Controller('tasks')
export class TasksController {
  constructor(private readonly tasks: TasksService) {}

  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.tasks.findOne(id);
  }
}
```

1. **Express receives the request** and matches the route Nest registered at startup for `/api/v1/tasks/:id`.
2. **Middleware runs** — anything bound in a module's `configure()`, plus `app.use()` middleware like Helmet (→ [Ch 11](11-middleware.md)).
3. **Guards run.** None here. A guard returning `false` ends the request with a 403 and nothing further executes (→ [Ch 12](12-guards.md)).
4. **Interceptors run their "before" half** (→ [Ch 13](13-interceptors.md)).
5. **Pipes run.** `ParseIntPipe` converts `'42'` to `42`, or throws a 400 for `'abc'` (→ [Ch 14](14-pipes-validation.md)).
6. **The handler runs.** `findOne(42)` is called; the returned promise is awaited.
7. **Interceptors run their "after" half**, transforming the response value.
8. **The response is serialized** and written.

If anything in 2–7 throws, control jumps to the exception filters (→ [Ch 15](15-exception-filters.md)), which map the error to a status code and body. Returning a value and throwing an exception are both normal, supported ways to end a request — `throw new NotFoundException()` is idiomatic, not exceptional.

Chapter 10 owns the exact ordering, including where global versus controller-scoped components sit. Everything else in Part III refers back to it.

---

## 3.7 Common Mistakes and Best Practices

**Mistake: assuming the generated app is production-ready.** It has no validation, no security headers, no CORS policy, and no shutdown handling.

**Mistake: forgetting `enableShutdownHooks()`.** Termination hooks never fire, connections are not drained, and deploys drop requests.

**Mistake: `enableCors({ origin: true })` in production.** It reflects whatever origin asks, which defeats the point of CORS.

**Mistake: registering a provider in the wrong module.** A provider must be in the `providers` array of a module that the consumer can see, or resolution fails at startup.

**Mistake: putting environment logic in `main.ts`.** `if (process.env.NODE_ENV === 'production')` there is untestable; a module can be configured and asserted on.

**Mistake: running `start:dev` in production.** It runs the TypeScript watcher, recompiles, and swallows errors a real build would reject. Use `build` then `start:prod`.

**Mistake: deleting the `.spec.ts` files with `--no-spec` to "clean up".** The generated tests are the only ones that exist, and writing them later never happens.

**Best practice: keep `main.ts` declarative.** Register global concerns and start the server; anything conditional belongs in a module.

**Best practice: use the generators.** They register the class in the right module, which removes the most common startup error before it happens.

**Best practice: leave `deleteOutDir: true` alone.** It is on in generated projects; people switch it off to speed up builds, and stale files in `dist/` then produce failures that survive a rebuild and waste an afternoon.

---

## 3.8 Real-World Example: A Bootstrap That Survives a Deploy

This is `main.ts` for a service running behind a load balancer, where a rolling deploy must not drop requests. The graceful-shutdown handling is the part most tutorials omit.

```ts
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe, VersioningType } from '@nestjs/common';
import { NestExpressApplication } from '@nestjs/platform-express';
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';
import { Logger } from 'nestjs-pino';
import helmet from 'helmet';
import { AppModule } from './app.module';

async function bootstrap(): Promise<void> {
  const app = await NestFactory.create<NestExpressApplication>(AppModule, {
    bufferLogs: true,               // nothing is lost between create() and the logger swap
  });

  // Replace Nest's default logger with the injected structured one.
  app.useLogger(app.get(Logger));

  // Behind a proxy, trust exactly one hop so req.ip is the real client and
  // not something a caller can forge by sending their own X-Forwarded-For.
  app.set('trust proxy', 1);

  app.use(helmet());
  app.enableCors({
    origin: process.env.CORS_ORIGINS?.split(',') ?? false,
    credentials: true,
    maxAge: 86_400,                 // cache preflights for a day
  });

  app.setGlobalPrefix('api', { exclude: ['health', 'metrics'] });
  app.enableVersioning({ type: VersioningType.URI, defaultVersion: '1' });

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
      // Hide validation internals from clients in production, keep them in dev.
      disableErrorMessages: process.env.NODE_ENV === 'production',
    }),
  );

  if (process.env.NODE_ENV !== 'production') {
    const config = new DocumentBuilder()
      .setTitle('Task API')
      .setVersion('1.0')
      .addBearerAuth()
      .build();
    SwaggerModule.setup('docs', app, SwaggerModule.createDocument(app, config));
  }

  // Must precede listen(). Without it, SIGTERM kills the process outright and
  // no module gets a chance to drain its pool or finish in-flight work.
  app.enableShutdownHooks();

  await app.listen(process.env.PORT ?? 3000);
}

void bootstrap();
```

```ts
// src/health/shutdown.service.ts
import { Injectable, OnApplicationShutdown, Logger } from '@nestjs/common';

/**
 * Kubernetes stops routing traffic only after the readiness probe fails, and
 * that check may lag the SIGTERM by seconds. Failing readiness immediately and
 * then pausing gives the load balancer time to drain before anything closes.
 */
@Injectable()
export class ShutdownService implements OnApplicationShutdown {
  private readonly logger = new Logger(ShutdownService.name);
  private shuttingDown = false;

  isShuttingDown(): boolean {
    return this.shuttingDown;
  }

  async onApplicationShutdown(signal?: string): Promise<void> {
    this.shuttingDown = true;                       // readiness probe now returns 503
    this.logger.log(`Received ${signal}, draining for 10s`);
    await new Promise((resolve) => setTimeout(resolve, 10_000));
  }
}
```

The decision worth defending is the fixed ten-second sleep, which looks like a hack and is in fact the correct shape. The process cannot know when the load balancer has finished removing it from rotation — that is the balancer's own polling interval, invisible from inside the pod. Waiting for a signal that never arrives is not an option, so the only honest implementation is to fail readiness immediately, then wait longer than the balancer's detection window before releasing anything. Set the duration from that window, and set `terminationGracePeriodSeconds` above it.

---

## 3.9 Questions

**1.** MCQ — Which command scaffolds a module, controller, service, and DTOs at once? a) `nest g module` b) `nest g resource` c) `nest new` d) `nest build`
<details><summary>Answer</summary>**b)** `nest g resource` (`nest g res`).</details>

**2.** MCQ — `app.enableShutdownHooks()` is: a) on by default b) opt-in c) Fastify-only d) deprecated
<details><summary>Answer</summary>**b)** Opt-in. Without it, `onApplicationShutdown` and `onModuleDestroy` never fire.</details>

**3.** MCQ — Routes are registered: a) on first request b) at `NestFactory.create()` c) at `app.listen()` d) lazily per route
<details><summary>Answer</summary>**b)** During `create()`, from the metadata decorators recorded. `listen()` only binds the socket.</details>

**4.** MCQ — Which `ValidationPipe` option returns 400 for unknown properties? a) `whitelist` b) `transform` c) `forbidNonWhitelisted` d) `disableErrorMessages`
<details><summary>Answer</summary>**c)** `whitelist` alone strips them silently; `forbidNonWhitelisted` makes it an error.</details>

**5.** MCQ — `tsconfig.build.json` exists to: a) speed up builds b) exclude tests from `dist/` c) enable decorators d) configure Jest
<details><summary>Answer</summary>**b)** It keeps spec files out of the production build.</details>

**6.** MCQ — In Nest 11, the order of termination lifecycle hooks was: a) unchanged b) reversed c) removed d) made synchronous
<details><summary>Answer</summary>**b)** Reversed — a breaking change worth auditing on upgrade.</details>

**7.** Short — Name the six things `NestFactory.create()` does before `listen()`.
<details><summary>Answer</summary>Scans the module graph, resolves dependencies, instantiates providers, runs `onModuleInit`/`onApplicationBootstrap`, and registers routes. `listen()` then binds the socket.</details>

**8.** Short — Why does a dependency error appear at startup rather than at request time?
<details><summary>Answer</summary>Resolution and instantiation happen during `create()`, before any request exists, so an unresolvable token fails the boot.</details>

**9.** Short — Why prefer `nest g service` over creating the file manually?
<details><summary>Answer</summary>The generator registers the provider in the nearest module's `providers` array — the step most commonly forgotten, and the usual cause of "Nest can't resolve dependencies".</details>

**10.** Short — What does `bufferLogs: true` accomplish?
<details><summary>Answer</summary>It holds log output produced during bootstrap until a custom logger is attached with `useLogger`, so early messages are not lost or emitted in the wrong format.</details>

**11.** Short — Why is `origin: true` in `enableCors` unsafe in production?
<details><summary>Answer</summary>It reflects whatever `Origin` the caller sends, so every origin is allowed — with `credentials: true` this exposes authenticated endpoints to any site.</details>

**12.** Output — With `whitelist: true` and `forbidNonWhitelisted: false`, a client POSTs `{ "email": "a@b.c", "isAdmin": true }` to a DTO declaring only `email`. What is the response and what does the handler receive?
<details><summary>Answer</summary>`201`, and the handler receives `{ email: 'a@b.c' }`. The unknown property is stripped silently — the client has no indication it was ignored.</details>

**13.** Output — Same request with `forbidNonWhitelisted: true`?
<details><summary>Answer</summary>`400`, with a message naming `isAdmin` as a property that should not exist.</details>

**14.** Output — `app.setGlobalPrefix('api', { exclude: ['health'] })` plus `enableVersioning({ type: VersioningType.URI, defaultVersion: '1' })`. What paths serve `@Controller('tasks') @Get()` and a health controller at `@Controller('health')`?
<details><summary>Answer</summary>`/api/v1/tasks` and `/health`. The exclusion removes both the prefix and versioning for that route, which is what probes and load balancers expect.</details>

**15.** Output — A provider's `onApplicationShutdown` logs a message. The container receives SIGTERM and `enableShutdownHooks()` was never called. What is logged?
<details><summary>Answer</summary>Nothing. The process exits without running termination hooks.</details>

**16.** Debug — `npm run start:dev` works; `npm run build` fails with type errors. How is that possible?
<details><summary>Answer</summary>Watch mode does not halt on a failed rebuild — it logs the error and keeps the previously compiled output running, so the server you are testing predates the broken change. The compiler is equally strict in both modes; only the failure handling differs. Always run `build` in CI so this surfaces before deploy.</details>

**17.** Debug — After renaming a service file, the app still uses the old implementation even after restarting. What do you check?
<details><summary>Answer</summary>Stale output in `dist/`. Generated projects set `deleteOutDir: true` in `nest-cli.json`, so check whether it was turned off — then delete `dist/` and rebuild.</details>

**18.** Debug — Every rolling deploy produces a burst of 502s. The app has no shutdown handling. Explain and fix.
<details><summary>Answer</summary>On SIGTERM the process dies immediately while the load balancer is still routing to it, so in-flight requests are cut. Call `enableShutdownHooks()`, fail the readiness probe on shutdown, wait longer than the balancer's detection interval, then close connections — and set the grace period above that wait.</details>

**19.** Debug — A newly generated controller's routes return 404 despite the file being correct. What is missing?
<details><summary>Answer</summary>It is not listed in any module's `controllers` array, so Nest never read its metadata and never registered the routes. There is no error — the decorators are simply inert.</details>

**20.** Debug — `req.ip` shows the load balancer's address rather than the client's. Fix, and the risk of fixing it wrongly?
<details><summary>Answer</summary>Set `app.set('trust proxy', 1)` to trust exactly one hop. Trusting all hops (`true`) lets a client forge `X-Forwarded-For` and defeat any IP-based rate limiting or logging.</details>

**21.** Code — Write a `main.ts` that enables validation with whitelisting, URI versioning defaulting to `1`, and graceful shutdown.
<details><summary>Answer</summary>

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableVersioning({ type: VersioningType.URI, defaultVersion: '1' });
  app.useGlobalPipes(
    new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true }),
  );
  app.enableShutdownHooks();
  await app.listen(process.env.PORT ?? 3000);
}
void bootstrap();
```
</details>

**22.** Code — Add a readiness endpoint that returns 503 once shutdown has begun.
<details><summary>Answer</summary>

```ts
@Controller('health')
export class HealthController {
  constructor(private readonly shutdown: ShutdownService) {}

  @Get('ready')
  ready() {
    if (this.shutdown.isShuttingDown()) {
      throw new ServiceUnavailableException('shutting down');
    }
    return { status: 'ok' };
  }
}
```
Exclude `health` from the global prefix so probe URLs stay stable across versioning changes.
</details>

**23.** Code — Move the global `ValidationPipe` out of `main.ts` into `AppModule` so it is covered by tests.
<details><summary>Answer</summary>

```ts
// src/app.module.ts
import { APP_PIPE } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_PIPE,
      useValue: new ValidationPipe({
        whitelist: true,
        forbidNonWhitelisted: true,
        transform: true,
      }),
    },
  ],
})
export class AppModule {}
```
Registered this way the pipe is part of the module graph, so `Test.createTestingModule` picks it up automatically. Use `useClass` instead of `useValue` if the pipe needs its own injected dependencies — `useGlobalPipes` in `main.ts` cannot inject anything, which is the main reason to prefer `APP_PIPE`.
</details>

**24.** Scenario — A teammate wants to move all global configuration out of `main.ts` and into `AppModule` so it can be unit-tested. Good idea?
<details><summary>Answer</summary>Mostly yes, and it is what `APP_PIPE`, `APP_GUARD`, `APP_FILTER`, and `APP_INTERCEPTOR` providers exist for — registered in a module, they are testable and can inject dependencies, which `useGlobalPipes` cannot. The exception is anything that must run before the Nest pipeline: `helmet`, `trust proxy`, and CORS operate at the platform level and belong in `main.ts`. So split it — pipeline components move into modules, platform configuration stays.</details>

**25.** Scenario — Your team debates committing a generated Swagger JSON file so the frontend can generate a client in CI. Where do you land?
<details><summary>Answer</summary>Generate it in CI rather than committing it by hand — a committed spec drifts from the code the moment someone forgets to regenerate, and a stale contract is worse than none. Add a CI step that boots the app, writes the document, and fails the build if it differs from the committed copy; that keeps the artifact available for downstream jobs while making drift a build error rather than a runtime surprise. Serving `/docs` publicly in production is a separate decision, and usually no.</details>

---

## Chapter Summary

- `nest new` generates a working app plus the TypeScript, lint, and test configuration that would otherwise take an afternoon — but it is a starting point, not a secure default.
- `NestFactory.create()` scans the graph, resolves and instantiates providers, runs init hooks, and registers routes; `listen()` only binds the socket, which is why tests can boot an app with no port.
- Generators matter because they register the class in the right module, eliminating the most common startup failure.
- A production `main.ts` adds validation, security headers, an explicit CORS origin list, versioning, and shutdown hooks — each closing a specific hole.
- `whitelist` without `forbidNonWhitelisted` strips unknown fields silently, which confuses clients more than a 400 would.
- Nest 11 reversed termination hook order, so upgrades should audit anything doing real work in `onModuleDestroy`.

---

## Practice Exercises

1. Generate a project and delete `app.controller.ts`, `app.service.ts`, and their specs. Get it running again with an empty `AppModule`.
2. Generate the same resource with and without `--flat`, and with `--dry-run` first. Diff the results.
3. Add a global `ValidationPipe` and prove with `curl` that `whitelist` and `forbidNonWhitelisted` behave differently.
4. Instrument every lifecycle hook with a log line, then start and stop the app to record the exact order — with and without `enableShutdownHooks()`.
5. Write a Dockerfile that builds with `npm run build` and runs `start:prod`, with no dev dependencies in the final image.

---

## Mini Project: A Bootstrap Checklist Service

Build a small service whose only job is to demonstrate a correct production bootstrap.

**Requirements:**

- `GET /health/live` and `GET /health/ready`, both excluded from the global prefix.
- Readiness flips to 503 on SIGTERM, then the process waits a configurable drain period.
- Global validation with whitelisting, and URI versioning with `/api/v1`.
- CORS origins read from an environment variable, defaulting to deny.
- A shutdown log line proving hook order, and a `docker stop` that exits cleanly within the grace period.

---

**Next:** [Chapter 4 — Modules →](04-modules.md)
