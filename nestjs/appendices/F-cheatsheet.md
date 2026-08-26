# Appendix F — Cheat Sheet

Dense and scannable. Tables over prose. Meant to be kept open in a tab.

**Legend:** ⚠️ common trap · 🔒 security-relevant · 🐢 performance-relevant · 🆕 changes in v12

---

## Contents

[CLI](#cli) · [Project Anatomy](#project-anatomy) · [Decorators](#decorator-reference) ·
[Request Pipeline](#the-request-pipeline) · [Modules](#modules) · [Providers](#provider-recipes) ·
[Scopes and Lifecycle](#scopes-and-lifecycle) · [Routing](#routing) ·
[Parameter Decorators](#parameter-decorators) · [Validation](#validation) ·
[Exceptions](#exception-classes) · [TypeORM](#typeorm) · [Testing](#testing) ·
[Config](#configuration) · [Bootstrap](#a-production-maints) ·
[Traps](#the-traps-worth-memorising) · [v11 → v12](#v11--v12)

---

## CLI

| Command | Short | Produces |
|---|---|---|
| `nest new <name>` | — | a project with tsconfig, lint, and test setup |
| `nest g resource <name>` | `g res` | module + controller + service + DTOs + specs |
| `nest g module <name>` | `g mo` | module, registered in the parent |
| `nest g controller <name>` | `g co` | controller + spec, registered |
| `nest g service <name>` | `g s` | provider + spec, registered |
| `nest g guard <name>` | `g gu` | guard + spec |
| `nest g interceptor <name>` | `g in` | interceptor + spec |
| `nest g pipe <name>` | `g pi` | pipe + spec |
| `nest g filter <name>` | `g f` | filter + spec |
| `nest g app <name>` | — | an application in a monorepo |
| `nest g library <name>` | `g lib` | a shared library |

```bash
nest g res tasks --dry-run     # -d: print, write nothing
nest g co tasks --no-spec      # skip the test file
nest g res tasks --flat        # no extra directory
```

**Scripts:** `start:dev` (watch) · `start:debug` (watch + inspector) · `start:prod` (runs `dist/main`) · `build` · `test` · `test:watch` · `test:cov` · `test:e2e` · `lint` · `format`

⚠️ Generators matter because they register the class in the nearest module — the step most commonly forgotten.

---

## Project Anatomy

| File | Purpose |
|---|---|
| `src/main.ts` | entry point; global pipeline, CORS, shutdown hooks |
| `src/app.module.ts` | root of the module graph |
| `nest-cli.json` | CLI/build config, `deleteOutDir`, monorepo projects |
| `tsconfig.json` | `experimentalDecorators`, `emitDecoratorMetadata` |
| `tsconfig.build.json` | excludes tests from `dist/` |
| `eslint.config.mjs` | flat-config ESLint (Nest 11) |
| `test/jest-e2e.json` | separate config for e2e tests |

---

## Decorator Reference

**Class**

| Decorator | Purpose |
|---|---|
| `@Module({})` | declares a module |
| `@Controller('prefix')` | declares a controller and its route prefix |
| `@Injectable()` | marks a provider; **triggers metadata emission** |
| `@Global()` | makes a module's exports visible everywhere |
| `@Catch(...)` | declares which exceptions a filter handles |
| `@WebSocketGateway({})` | declares a WebSocket gateway |
| `@Resolver(() => T)` | declares a GraphQL resolver |
| `@Processor('queue')` | declares a BullMQ processor |

**Method**

| Decorator | Purpose |
|---|---|
| `@Get`/`@Post`/`@Put`/`@Patch`/`@Delete`/`@Head`/`@Options`/`@All` | route |
| `@HttpCode(n)` | override the status code |
| `@Header(k, v)` | set a response header |
| `@Redirect(url, code)` | redirect; overridable by returning `{ url, statusCode }` |
| `@UseGuards(...)` | bind guards |
| `@UseInterceptors(...)` | bind interceptors |
| `@UsePipes(...)` | bind pipes |
| `@UseFilters(...)` | bind exception filters |
| `@SetMetadata(k, v)` | attach metadata for a guard to read |
| `@Version('1')` | route version |
| `@SerializeOptions({ groups })` | serialization groups |
| `@Cron` / `@Interval` / `@Timeout` | scheduling |
| `@MessagePattern` / `@EventPattern` | microservice handlers |
| `@SubscribeMessage('name')` | WebSocket message handler |

---

## The Request Pipeline

```
1.  Incoming request
2.  Middleware            global (app.use) → module-bound (configure)
3.  Guards                global → controller → route
4.  Interceptors (pre)    global → controller → route
5.  Pipes                 global → controller → route → parameter
6.  Route handler
7.  Service layer
8.  Interceptors (post)   route → controller → global      ← reversed
9.  Exception filters     route → controller → global      ← most specific first
10. Response
```

| Component | Knows the handler? | Can stop the request? | Sees the response? |
|---|---|---|---|
| Middleware | ❌ | yes (`res.end()`) | only via raw `res` |
| Guard | ✅ | yes (return `false`) | ❌ |
| Interceptor | ✅ | yes (return early) | ✅ |
| Pipe | ✅ | yes (throw) | ❌ |
| Filter | ✅ | — | ✅ (it writes it) |

⚠️ Decorator order does **not** set pipeline order. It only sequences components of the same kind.

---

## Modules

```ts
@Module({
  imports: [OtherModule],      // what this module may use
  controllers: [FooController],// instantiated; routes registered
  providers: [FooService],     // instantiated; injectable HERE only
  exports: [FooService],       // ...and by modules importing this one
})
export class FooModule {}
```

| Naming convention | Meaning |
|---|---|
| `forRoot(opts)` | configure once at the application root |
| `forRootAsync(opts)` | same, with options from injected providers |
| `forFeature(x)` | register a scoped slice — one entity, one queue |
| `register(opts)` | configure per-import, no global state implied |
| `registerAsync(opts)` | async form of `register` |

⚠️ Registering one provider in two modules creates **two instances**. Register once, export, import.

---

## Provider Recipes

```ts
providers: [
  FooService,                                            // = useClass shorthand
  { provide: TOKEN,    useValue: { a: 1 } },             // existing object
  { provide: Contract, useClass: Impl },                 // container constructs it
  { provide: TOKEN,    useFactory: (c) => …, inject: [C] }, // positional inject
  { provide: 'ALIAS',  useExisting: FooService },        // SAME instance
]
```

| Recipe | Can inject? | Instances | Use for |
|---|---|---|---|
| `useValue` | ❌ | the object you gave | constants, test doubles, third-party clients |
| `useClass` | ✅ | new per token | swapping implementations |
| `useFactory` | ✅ (via `inject`) | the return value | construction needing logic or config |
| `useExisting` | — | **shared** | aliasing, deprecating a token |

**Global pipeline components** (multi-provider, run in registration order):

```ts
import { APP_GUARD, APP_PIPE, APP_FILTER, APP_INTERCEPTOR } from '@nestjs/core';
{ provide: APP_GUARD, useClass: JwtAuthGuard }
```

⚠️ `useGlobalGuards(new Guard())` in `main.ts` **cannot inject** and is absent from tests. Prefer `APP_*`.

---

## Scopes and Lifecycle

| Scope | Instances | Hooks fire? |
|---|---|---|
| `DEFAULT` | one per application | ✅ |
| `REQUEST` | one per request | ❌ **never** |
| `TRANSIENT` | one per injecting consumer | init only |

⚠️ `REQUEST` **bubbles upward** — every consumer above becomes request-scoped. `TRANSIENT` does not.

**Order:**

```
Init:       onModuleInit → onApplicationBootstrap → (listen)
Terminate:  onModuleDestroy → beforeApplicationShutdown → onApplicationShutdown
```

⚠️ Termination hooks run only after `app.enableShutdownHooks()` (SIGTERM, SIGINT, SIGBREAK, SIGHUP).
⚠️ Nest 11 **reversed** module traversal order within the termination phases.

```ts
moduleRef.get(Token)              // singleton, sync — throws for scoped
await moduleRef.resolve(Token)    // scoped, async — new per context id
moduleRef.get(Token, { strict: false })   // search the whole app
```

---

## Routing

```ts
@Controller('tasks')                          // /tasks
@Controller({ path: 'tasks', version: '2' })  // /v2/tasks
```

**Express 5 syntax (Nest 11):**

| Intent | Express 4 | Express 5 ✅ |
|---|---|---|
| Wildcard | `'*'` | `'*splat'` |
| Path wildcard | `'files/*'` | `'files/*path'` |
| Optional segment | `':name.:ext?'` | `':name{.:ext}'` |

**Default status:** `POST` → 201, everything else → 200. Override with `@HttpCode()`.

⚠️ Routes match in **registration order** — declare `@Get('me')` before `@Get(':id')`.

**Versioning:** `URI` (visible, cacheable) · `HEADER` (clean URLs, invisible in logs) · `MEDIA_TYPE` · `CUSTOM`

---

## Parameter Decorators

| Decorator | Reads | Note |
|---|---|---|
| `@Param(key?)` | path params | **always strings** |
| `@Query(key?)` | query string | always strings; whole-object form supports DTOs |
| `@Body(key?)` | parsed body | whole-object form for validation |
| `@Headers(name?)` | headers | |
| `@Ip()` | client IP | needs correct `trust proxy` 🔒 |
| `@HostParam(key?)` | sub-domain params | |
| `@Session()` | session | needs session middleware |
| `@Req()` / `@Res()` | raw platform objects | ⚠️ see below |

⚠️ Bare `@Res()` disables Nest's response handling — response-transforming interceptors stop working on that route. Use `@Res({ passthrough: true })`.

**Built-in pipes:** `ValidationPipe` · `ParseIntPipe` · `ParseFloatPipe` · `ParseBoolPipe` · `ParseArrayPipe` · `ParseUUIDPipe` · `ParseEnumPipe` · `DefaultValuePipe` · `ParseFilePipe`

⚠️ Pipes compose left to right: `new DefaultValuePipe(1), ParseIntPipe` — reversed, the parser sees `undefined`.

---

## Validation

```ts
new ValidationPipe({
  whitelist: true,              // 🔒 strip undeclared properties (mass assignment)
  forbidNonWhitelisted: true,   // ...and 400 instead of silently dropping
  transform: true,              // return DTO instances, coerce primitives
  transformOptions: { enableImplicitConversion: false },
  stopAtFirstError: false,
})
```

| Decorator | Checks |
|---|---|
| `@IsString` `@IsInt` `@IsBoolean` `@IsDate` `@IsUUID` `@IsEmail` `@IsEnum` | type |
| `@Min` `@Max` `@Length` `@MinLength` `@MaxLength` | bounds |
| `@IsOptional` | skip when absent |
| `@IsArray` `@ArrayMinSize` `@ArrayMaxSize` | arrays |
| `@ValidateNested({ each: true })` | nested objects — **needs `@Type()`** |
| `@Type(() => T)` | `class-transformer`; conversion and element type |
| `@Transform(({ value }) => …)` | custom coercion |

**Mapped types** (import from `@nestjs/swagger` when using Swagger):

```ts
PartialType(Dto)        // every field optional
PickType(Dto, ['a'])    // only these
OmitType(Dto, ['a'])    // everything except
IntersectionType(A, B)  // combined
```

⚠️ **Validation fails open two ways** — both return 201 for invalid input:
1. `@ValidateNested()` without `@Type()` — nested objects never checked.
2. A DTO declared as an `interface` — erased, so `metatype` is `Object` and the pipe skips it.

---

## Exception Classes

| Class | Status | | Class | Status |
|---|---|---|---|---|
| `BadRequestException` | 400 | | `PayloadTooLargeException` | 413 |
| `UnauthorizedException` | 401 | | `UnsupportedMediaTypeException` | 415 |
| `ForbiddenException` | 403 | | `UnprocessableEntityException` | 422 |
| `NotFoundException` | 404 | | `InternalServerErrorException` | 500 |
| `MethodNotAllowedException` | 405 | | `NotImplementedException` | 501 |
| `RequestTimeoutException` | 408 | | `BadGatewayException` | 502 |
| `ConflictException` | 409 | | `ServiceUnavailableException` | 503 |
| `GoneException` | 410 | | `GatewayTimeoutException` | 504 |

⚠️ `throw new Error('not found')` → **500**, message withheld. Only `HttpException` subclasses carry a status.

```ts
throw new HttpException({ code: 'QUOTA', retryAfter: 60 }, HttpStatus.TOO_MANY_REQUESTS);
```

---

## TypeORM

```ts
TypeOrmModule.forRootAsync({ useFactory: … })   // root connection
TypeOrmModule.forFeature([Task])                // per-module repositories
@InjectRepository(Task) private repo: Repository<Task>
```

| Method | SQL | Notes |
|---|---|---|
| `save(entity)` | `SELECT` + `INSERT`/`UPDATE` | hooks, cascades; **two queries** 🐢 |
| `insert(dto)` | `INSERT` | one query, no hooks |
| `update(where, dto)` | `UPDATE` | one query, returns no entity |
| `upsert(dto, keys)` | `INSERT … ON CONFLICT` | atomic |
| `delete(where)` | `DELETE` | hard |
| `softDelete(where)` | `UPDATE … deletedAt` | needs `@DeleteDateColumn` |

**Column choices that matter:**

| Use | Not |
|---|---|
| `timestamptz` | `timestamp` (shifts with server zone) |
| integer minor units | `decimal` (returns a **string**) or `float` |
| explicit `length` | default `varchar(255)` |

```ts
// Operators
where: { status: In([...]), dueDate: LessThan(d), title: ILike(`%${q}%`), deletedAt: IsNull() }
```

⚠️ `synchronize: true` **drops columns**. Use migrations.
⚠️ `findOne({ where: { id: undefined } })` returns the **first row**, not `null`.
⚠️ Never `eager: true` — it joins on every query with no per-call opt-out. 🐢

```ts
await dataSource.transaction(async (manager) => { … });   // use `manager` for EVERY query
```

---

## Testing

```ts
const module = await Test.createTestingModule({ providers: [Svc, { provide: Dep, useValue: fake }] })
  .overrideProvider(Token).useClass(Fake)
  .overrideGuard(JwtAuthGuard).useValue({ canActivate: () => true })
  .overrideInterceptor(CacheInterceptor).useValue({ intercept: (_c, n) => n.handle() })
  .compile();

module.get(Token);              // singleton
await module.resolve(Token);    // scoped
```

**E2E:**

```ts
app = module.createNestApplication();
app.useGlobalPipes(new ValidationPipe({ whitelist: true }));   // main.ts does NOT run
await app.init();                        // no port bound
await request(app.getHttpServer()).get('/tasks').expect(200);
await app.close();                       // ⚠️ or Jest hangs
```

| Double | Use when |
|---|---|
| Stub | the test cares about the result |
| Mock | the call itself is the behaviour |
| Fake | several operations must stay consistent — **repositories** |

⚠️ `expect(p).rejects.toThrow()` without `await` **passes unconditionally**.

---

## Configuration

```ts
ConfigModule.forRoot({
  isGlobal: true,
  cache: true,
  envFilePath: ['.env.local', '.env'],
  load: [databaseConfig, jwtConfig],
  validationSchema: envSchema,
  validationOptions: { abortEarly: false, allowUnknown: true },
});
```

```ts
config.get<T>('KEY')            // T | undefined
config.get<T>('KEY', fallback)  // with default
config.getOrThrow<T>('KEY')     // T — use for anything required

export default registerAs('db', () => ({ url: process.env.DATABASE_URL! }));
constructor(@Inject(dbConfig.KEY) private cfg: ConfigType<typeof dbConfig>) {}
```

⚠️ `useValue` evaluates at **import time**, before `.env` loads. Use `useFactory`. 🔒 Never log the config object unredacted.

---

## A Production `main.ts`

```ts
const app = await NestFactory.create<NestExpressApplication>(AppModule, { bufferLogs: true });

app.useLogger(app.get(Logger));
app.use(helmet());                                          // 🔒
app.set('trust proxy', 1);                                  // 🔒 exactly your proxy count
app.enableCors({ origin: originsFromConfig ?? false, credentials: true });   // 🔒 never `true`
app.setGlobalPrefix('api', { exclude: ['health'] });
app.enableVersioning({ type: VersioningType.URI, defaultVersion: '1' });
app.useGlobalPipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true }));
app.enableShutdownHooks();                                  // ⚠️ opt-in
await app.listen(process.env.PORT ?? 3000);
```

---

## The Traps Worth Memorising

| Trap | Consequence | Ch |
|---|---|---|
| `@Get(':id')` before `@Get('me')` | dynamic route swallows the static one, silently | [5](../chapters/05-controllers-routing.md) |
| Missing `@Injectable()` | no metadata → "can't resolve dependencies (?, ?, ?)" | [2](../chapters/02-typescript-decorators.md) |
| Provider in two modules | two instances, divergent state | [4](../chapters/04-modules.md) |
| `@ValidateNested()` without `@Type()` | nested validation **fails open** 🔒 | [14](../chapters/14-pipes-validation.md) |
| DTO as an interface | validation silently skipped 🔒 | [14](../chapters/14-pipes-validation.md) |
| `whitelist` without `forbidNonWhitelisted` | fields dropped silently | [14](../chapters/14-pipes-validation.md) |
| Request state on a singleton | cross-request corruption under load 🔒 | [6](../chapters/06-providers-services.md) |
| Request-scoped provider deep in the graph | whole subtree rebuilt per request 🐢 | [9](../chapters/09-scopes-lifecycle.md) |
| `tap(fn)` for metrics | failures never recorded; dashboards lie | [13](../chapters/13-interceptors.md) |
| Cache key without the principal | one user's data served to another 🔒 | [20](../chapters/20-caching.md) |
| `ttl: 60` on cache-manager v6 | 60 **milliseconds** | [20](../chapters/20-caching.md) |
| No `enableShutdownHooks()` | termination hooks are dead code | [9](../chapters/09-scopes-lifecycle.md) |
| `@Cron` with replicas | job runs N times | [32](../chapters/32-task-scheduling.md) |
| `client.send()` unsubscribed | nothing transmitted, no error | [26](../chapters/26-microservices.md) |
| Missing Redis adapter | broadcasts reach one process | [25](../chapters/25-websockets.md) |
| DB check in liveness | dependency blip → cluster restart storm | [33](../chapters/33-logging-observability.md) |
| `trust proxy: true` | client forges its IP, bypasses rate limits 🔒 | [23](../chapters/23-security.md) |
| `origin: true` + credentials | every origin can call your API 🔒 | [23](../chapters/23-security.md) |
| `synchronize: true` | columns dropped | [17](../chapters/17-typeorm.md) |
| Injected repo inside a transaction | writes survive the rollback | [18](../chapters/18-migrations-transactions.md) |

---

## v11 → v12

**NestJS 11 (this book):** Node 20+ · Express 5 · reversed termination hook order · `cache-manager` v6 (milliseconds).

🆕 **NestJS 12** (~Q3 2026, prereleases on the `next` tag):

| Pillar | Impact |
|---|---|
| Full ESM | no `require`, no `__dirname`, `.js` extensions on relative imports |
| Standard Schema | Zod/Valibot/ArkType usable directly in `@Body()`, `@Query()`, `@Param()` |
| Vitest | default for new **ESM** projects; CJS projects keep Jest |
| Rspack + oxlint | replace Webpack and ESLint |

```ts
// ESM equivalents
import { dirname } from 'node:path';
import { fileURLToPath } from 'node:url';
const __dirname = dirname(fileURLToPath(import.meta.url));
```

Decorators and DI are **unaffected** by ESM. → [Ch 36](../chapters/36-nestjs-11-to-12.md)

---

**Next:** [Appendix G — Glossary →](G-glossary.md)
