# Chapter 5 — Controllers and Routing

## Learning Objectives

- Map an HTTP request onto a controller method using the full set of route decorators
- Extract path, query, body, and header data with the correct decorator for each
- Control status codes, headers, and redirects without reaching for the raw response
- Predict which route wins when two patterns could match, and order routes accordingly
- Version an API without breaking existing clients

---

## 5.1 Routing Is Two Prefixes and a Method

A route is assembled from the controller prefix and the method decorator. Nothing else participates unless you add a global prefix or versioning.

```ts
// src/tasks/tasks.controller.ts
@Controller('tasks')          // prefix
export class TasksController {
  @Get()                      // GET /tasks
  findAll() {}

  @Get(':id')                 // GET /tasks/:id
  findOne() {}

  @Post()                     // POST /tasks
  create() {}

  @Patch(':id')               // PATCH /tasks/:id
  update() {}

  @Delete(':id')              // DELETE /tasks/:id
  remove() {}
}
```

Every HTTP verb has a decorator — `@Get`, `@Post`, `@Put`, `@Patch`, `@Delete`, `@Head`, `@Options`, and `@All`, which matches any verb. A controller with no argument (`@Controller()`) mounts at the root.

The return value becomes the response body. Nest serializes objects and arrays to JSON automatically, and awaits promises and observables:

```ts
@Get()
findAll(): Promise<Task[]> {
  return this.tasks.findAll();     // no await needed — Nest awaits it
}
```

**Default status codes are 201 for `POST` and 200 for everything else.** That surprises people writing a `POST /auth/login` that returns a session rather than creating a resource:

```ts
@Post('login')
@HttpCode(200)                     // otherwise 201, which is wrong for a login
login(@Body() dto: LoginDto) {}
```

> 💼 **Interview** — "What status code does a NestJS `POST` return by default, and when would you change it?" The answer is 201, and you change it whenever the POST is an action rather than a creation — login, search-with-a-body, or any RPC-shaped endpoint.

---

## 5.2 Extracting Request Data

Each part of the request has a decorator. Prefer them over `@Req()`, because they are the seam that pipes and validation plug into.

| Decorator | Reads | Typical use |
|---|---|---|
| `@Param(key?)` | path parameters | `@Param('id')` |
| `@Query(key?)` | query string | `@Query('page')` |
| `@Body(key?)` | parsed body | `@Body()` with a DTO |
| `@Headers(name?)` | headers | `@Headers('x-request-id')` |
| `@Ip()` | client IP | rate limiting, audit |
| `@HostParam(key?)` | sub-domain params | multi-tenant routing |
| `@Session()` | session object | requires session middleware |
| `@Req()` / `@Res()` | raw platform objects | escape hatch |

```ts
@Get(':id')
findOne(
  @Param('id', ParseIntPipe) id: number,       // '42' → 42, or 400
  @Query() filter: TaskFilterDto,              // whole query string, validated
  @Headers('x-request-id') requestId?: string,
) {
  return this.tasks.findOne(id, filter);
}
```

Calling a decorator with no argument gives you the whole object; with a key, one value. **The whole-object form is the one that supports validation**, because a DTO class can be attached to it:

```ts
// ❌ Field-by-field: nothing validates, everything is a string, and adding a
//    filter means changing the signature.
findAll(@Query('page') page: string, @Query('limit') limit: string) {}

// ✅ One DTO: validated, coerced, and self-documenting.
findAll(@Query() query: PaginationDto) {}
```

> ⚠️ **Gotcha** — Query parameters and path parameters are **always strings**. `@Query('page') page: number` does not make `page` a number; the type annotation is erased (→ [Ch 2](02-typescript-decorators.md)) and you get `'2'`. You need `ParseIntPipe`, or `transform: true` on `ValidationPipe` together with `@Type(() => Number)` on the DTO property.

---

## 5.3 Controlling the Response

Three decorators cover almost everything, and none of them require the raw response object.

```ts
@Post()
@HttpCode(HttpStatus.ACCEPTED)                 // 202
@Header('Cache-Control', 'no-store')
create(@Body() dto: CreateTaskDto) {}

@Get('docs')
@Redirect('https://docs.example.com', 301)
toDocs() {}
```

`@Redirect()` can also be decided at runtime by returning a `{ url, statusCode }` object from the handler, which overrides the decorator's arguments.

Then there is `@Res()`, which is an escape hatch with a cost:

```ts
// ❌ Nest now assumes YOU will send the response. Interceptors that map the
//    return value stop working, and forgetting res.send() hangs the request.
@Get('report')
getReport(@Res() res: Response) {
  res.status(200).json({ ok: true });
}

// ✅ passthrough keeps Nest's response handling; use res only for the extra bit
@Get('report')
getReport(@Res({ passthrough: true }) res: Response) {
  res.cookie('seen', '1');
  return { ok: true };                        // still serialized by Nest
}
```

Injecting `@Res()` without `passthrough: true` switches that handler into manual mode. Interceptors that transform the response value silently do nothing, because there is no return value to transform. This is a genuinely common source of "my serialization interceptor works everywhere except one endpoint".

Use `@Res({ passthrough: true })` when you need to set a cookie or a header that no decorator covers. Use bare `@Res()` only for streaming or when you are writing the response body yourself (→ [Ch 31](31-file-uploads-streaming.md)).

---

## 5.4 Route Matching Order

Routes are matched **in registration order**, and registration order is the order the methods appear in the class. This produces the single most common routing bug in Nest:

```ts
@Controller('users')
export class UsersController {
  @Get(':id')                    // ❌ registered first — matches /users/me
  findOne(@Param('id') id: string) {}

  @Get('me')                     // never reached
  me() {}
}
```

`GET /users/me` hits `findOne` with `id = 'me'`. There is no error, no warning — just a 404 or a database lookup for a user called "me". **Static segments must be declared before dynamic ones:**

```ts
@Controller('users')
export class UsersController {
  @Get('me')                     // ✅ specific first
  me() {}

  @Get(':id')
  findOne(@Param('id') id: string) {}
}
```

The same applies across controllers: whichever module is imported first registers first.

NestJS 11 runs Express 5, which changed wildcard syntax (→ [Ch 1](01-what-nestjs-is.md)):

```ts
@Get('*')             // ❌ throws at startup in Express 5
@Get('*splat')        // ✅ named wildcard; @Param('splat') holds the rest
@Get('files/*path')   // ✅ matches files/a/b/c, path = 'a/b/c'
@Get(':name{.:ext}')  // ✅ optional segment — braces, not a trailing ?
```

---

## 5.5 Versioning

Versioning is enabled globally, then applied per controller or per route. It exists so you can change a response shape without breaking clients that have not migrated.

```ts
// src/main.ts
app.enableVersioning({ type: VersioningType.URI, defaultVersion: '1' });
```

| Type | Where the version lives | Trade-off |
|---|---|---|
| `URI` | `/v1/tasks` | Visible, cacheable, ugly in the path |
| `HEADER` | a custom header | Clean URLs, invisible in logs and browsers |
| `MEDIA_TYPE` | `Accept: application/json;v=1` | Correct by REST purism, awkward in practice |
| `CUSTOM` | your own extractor | Anything else |

URI versioning is the default choice for a reason: it survives being pasted into a browser, a curl command, and a bug report.

```ts
@Controller({ path: 'tasks', version: '2' })   // whole controller is v2
export class TasksV2Controller {}

@Controller('tasks')
export class TasksController {
  @Version('1')                                 // just this route
  @Get()
  findAllV1() {}

  @Version(VERSION_NEUTRAL)                     // answers on every version
  @Get('health')
  health() {}
}
```

The usual shape is one controller per version, both delegating to the same service. Duplicating the controller is cheap; duplicating the business logic is not.

> ⚠️ **Gotcha** — `defaultVersion` applies to controllers with no explicit version. If you add a v2 controller and forget that the v1 controller has no `version` set, both may answer on the same path and the first-registered one wins. Set the version explicitly on every controller once you have more than one.

---

## 5.6 Common Mistakes and Best Practices

**Mistake: declaring `@Get(':id')` before `@Get('me')`.** The dynamic route swallows the static one, silently.

**Mistake: expecting `@Param`/`@Query` to give you a number.** They are always strings; the type annotation is erased and enforces nothing.

**Mistake: using bare `@Res()`.** It disables Nest's response handling, so interceptors that map the return value stop working on that route only.

**Mistake: putting business logic in the controller.** It cannot then be reused by a queue worker, a CLI command, or a GraphQL resolver.

**Mistake: returning entities directly.** Every column ships to the client, including the ones added later — use a DTO and `ClassSerializerInterceptor`.

**Mistake: leaving `POST` at its default 201 for non-creating actions.** A login returning 201 tells the client a resource was created.

**Mistake: extracting query parameters one by one.** You lose validation, coercion, and Swagger documentation in one move.

**Best practice: one DTO per request shape.** It is the only place validation, transformation, and API documentation can all attach.

**Best practice: keep controllers thin enough to read in one screen.** A controller method should be a signature, a call, and a return.

**Best practice: version from the first release.** Adding versioning after clients exist means every existing URL is already the unversioned legacy case.

---

## 5.7 Real-World Example: A Paginated, Filtered, Versioned Resource

A list endpoint that is genuinely production-shaped: validated pagination, sortable, filterable, correctly typed, and returning a stable envelope that can grow without breaking clients.

```ts
// src/tasks/dto/list-tasks.dto.ts
import { Type } from 'class-transformer';
import { IsEnum, IsInt, IsOptional, IsString, Max, Min, MaxLength } from 'class-validator';

export enum TaskStatus {
  Open = 'open',
  Done = 'done',
}

export class ListTasksDto {
  // Query values arrive as strings; @Type drives the conversion that
  // ValidationPipe's `transform` then applies. Without it, @IsInt always fails.
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page: number = 1;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)              // a hard ceiling: without it a client can request 10 million rows
  limit: number = 20;

  @IsOptional()
  @IsEnum(TaskStatus)
  status?: TaskStatus;

  @IsOptional()
  @IsString()
  @MaxLength(120)
  search?: string;
}
```

```ts
// src/tasks/dto/paginated.dto.ts
export class PaginatedDto<T> {
  data: T[];
  meta: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
  };
}
```

```ts
// src/tasks/tasks.controller.ts
import { Controller, Get, Post, Patch, Delete, Param, Query, Body,
         HttpCode, HttpStatus, ParseUUIDPipe, NotFoundException } from '@nestjs/common';

@Controller({ path: 'tasks', version: '1' })
export class TasksController {
  constructor(private readonly tasks: TasksService) {}

  /**
   * Static routes are declared before dynamic ones so `/tasks/stats`
   * is never captured by `/tasks/:id`.
   */
  @Get('stats')
  getStats(): Promise<TaskStatsDto> {
    return this.tasks.getStats();
  }

  @Get()
  findAll(@Query() query: ListTasksDto): Promise<PaginatedDto<TaskDto>> {
    return this.tasks.findAll(query);
  }

  @Get(':id')
  async findOne(@Param('id', ParseUUIDPipe) id: string): Promise<TaskDto> {
    const task = await this.tasks.findOne(id);
    if (!task) throw new NotFoundException(`Task ${id} not found`);
    return task;
  }

  @Post()
  create(@Body() dto: CreateTaskDto): Promise<TaskDto> {
    return this.tasks.create(dto);            // 201 by default — correct here
  }

  @Patch(':id')
  update(
    @Param('id', ParseUUIDPipe) id: string,
    @Body() dto: UpdateTaskDto,
  ): Promise<TaskDto> {
    return this.tasks.update(id, dto);
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)            // 204: nothing to return
  remove(@Param('id', ParseUUIDPipe) id: string): Promise<void> {
    return this.tasks.remove(id);
  }
}
```

The decision worth defending is the `meta` envelope rather than returning a bare array with pagination in headers. Headers are the more RESTful answer and they are worse in practice: they are invisible in a browser, dropped by several HTTP clients unless explicitly requested, and awkward to consume from a typed frontend. The envelope costs one level of nesting and buys a response that documents itself and can gain fields — a `cursor`, a `hasMore` — without breaking a single existing consumer. The `@Max(100)` on `limit` is the other decision that matters: an unbounded page size is a denial-of-service vector that looks like a feature, and it is far easier to raise a ceiling later than to discover you never had one.

---

## 5.8 Questions

**1.** MCQ — The default status for `@Post()` is: a) 200 b) 201 c) 202 d) 204
<details><summary>Answer</summary>**b)** 201. Everything else defaults to 200.</details>

**2.** MCQ — `@Param('id') id: number` gives you: a) a number b) a string c) `NaN` d) `undefined`
<details><summary>Answer</summary>**b)** A string. The type annotation is erased; use `ParseIntPipe` to convert.</details>

**3.** MCQ — Which decorator matches any HTTP verb? a) `@Any()` b) `@All()` c) `@Match()` d) `@Every()`
<details><summary>Answer</summary>**b)** `@All()`.</details>

**4.** MCQ — Routes are matched in: a) alphabetical order b) specificity order c) registration order d) random order
<details><summary>Answer</summary>**c)** Registration order, which follows method declaration order in the class.</details>

**5.** MCQ — `@Res()` without `passthrough` means: a) nothing changes b) you must send the response yourself c) it throws d) Nest sends it twice
<details><summary>Answer</summary>**b)** Nest hands over response handling entirely; forgetting to send hangs the request.</details>

**6.** MCQ — Which versioning type keeps URLs clean but is invisible in logs? a) `URI` b) `HEADER` c) `MEDIA_TYPE` d) `CUSTOM`
<details><summary>Answer</summary>**b)** `HEADER`.</details>

**7.** Short — Why prefer `@Query() dto: SomeDto` over `@Query('page')`?
<details><summary>Answer</summary>The whole-object form can carry a DTO class, so it gets validation, type coercion, defaults, and Swagger documentation; the per-key form gets none of those and returns raw strings.</details>

**8.** Short — What breaks when you inject bare `@Res()`?
<details><summary>Answer</summary>Nest stops handling the response, so interceptors that transform the return value have nothing to transform, and the route hangs if you never call `res.send()`.</details>

**9.** Short — Why must static routes be declared before dynamic ones?
<details><summary>Answer</summary>Matching is in registration order, so `@Get(':id')` declared first will capture `/me`, `/stats`, and every other static sibling.</details>

**10.** Short — When should a `POST` not return 201?
<details><summary>Answer</summary>When it performs an action rather than creating a resource — login, logout, search-with-a-body, or any RPC-shaped call. Use `@HttpCode(200)`.</details>

**11.** Short — Why return a DTO rather than an entity?
<details><summary>Answer</summary>An entity serializes every column, including ones added later, so a future migration can leak a password hash or an internal flag without anyone touching the controller.</details>

**12.** Output — What does `GET /users/me` hit?
```ts
@Controller('users')
export class UsersController {
  @Get(':id') findOne(@Param('id') id: string) { return { id }; }
  @Get('me')  me() { return { me: true }; }
}
```
<details><summary>Answer</summary>`findOne` with `id = 'me'`, returning `{ id: 'me' }`. The dynamic route was registered first. No error is raised.</details>

**13.** Output — With `ValidationPipe({ transform: true })` and `@Query() q: Dto` where `Dto.page` is `@IsInt()` with no `@Type(() => Number)`, what does `?page=2` produce?
<details><summary>Answer</summary>A 400. The value is the string `'2'`, `@IsInt()` rejects it, and without `@Type(() => Number)` nothing converted it first.</details>

**14.** Output — What is the response body and status?
```ts
@Delete(':id')
@HttpCode(HttpStatus.NO_CONTENT)
remove() { return { deleted: true }; }
```
<details><summary>Answer</summary>Status 204 with an empty body — 204 must not carry one, so the returned object is discarded. Returning a value alongside 204 signals a design mistake.</details>

**15.** Output — A global interceptor maps every response to `{ data: <value> }`. One handler injects bare `@Res()` and calls `res.json({ ok: true })`. What does that route return?
<details><summary>Answer</summary>`{ ok: true }`, unwrapped. The interceptor never sees a return value because the handler wrote the response directly.</details>

**16.** Debug — `GET /tasks/stats` returns "invalid UUID" instead of statistics. Cause and fix?
<details><summary>Answer</summary>`@Get(':id')` with `ParseUUIDPipe` is declared before `@Get('stats')`, so `stats` is being parsed as an id. Move the static route above the dynamic one.</details>

**17.** Debug — After upgrading to Nest 11, startup fails on a controller containing `@Get('*')`. Fix?
<details><summary>Answer</summary>Express 5 requires named wildcards. Use `@Get('*splat')` and read it with `@Param('splat')`.</details>

**18.** Debug — A `Set-Cookie` header set with `@Res({ passthrough: true })` appears, but the response body is empty. What went wrong?
<details><summary>Answer</summary>The handler returned nothing. With `passthrough: true` Nest still sends the response, but only if the handler returns a value — setting a header is not returning a body.</details>

**19.** Debug — Adding a v2 controller made some v1 requests hit v2 intermittently. Cause?
<details><summary>Answer</summary>The v1 controller has no explicit `version`, so `defaultVersion` applies and both controllers answer on overlapping paths; registration order then decides. Set the version explicitly on every controller.</details>

**20.** Debug — A DTO property `tags: string[]` is empty for `?tags=a&tags=b` on some requests and a string on others. Explain.
<details><summary>Answer</summary>Query parsing yields a string for one occurrence and an array for several. Normalise with `@Transform(({ value }) => [].concat(value ?? []))` so the shape is stable regardless of how many values arrive.</details>

**21.** Code — Write a controller route that streams a redirect target decided at runtime.
<details><summary>Answer</summary>

```ts
@Get('go/:code')
@Redirect()                                   // no default; the return value supplies it
async go(@Param('code') code: string) {
  const target = await this.links.resolve(code);
  if (!target) throw new NotFoundException();
  return { url: target, statusCode: 302 };    // overrides the decorator
}
```
</details>

**22.** Code — Write a `PaginationDto` whose `page` and `limit` are coerced, bounded, and defaulted.
<details><summary>Answer</summary>

```ts
export class PaginationDto {
  @IsOptional() @Type(() => Number) @IsInt() @Min(1)
  page: number = 1;

  @IsOptional() @Type(() => Number) @IsInt() @Min(1) @Max(100)
  limit: number = 20;
}
```
`@Type` performs the string→number conversion; `@Max(100)` prevents a client from requesting an unbounded page.
</details>

**23.** Code — Expose the same `TasksService` under `/v1/tasks` and `/v2/tasks` where v2 renames `title` to `name`.
<details><summary>Answer</summary>

```ts
@Controller({ path: 'tasks', version: '1' })
export class TasksV1Controller {
  constructor(private readonly tasks: TasksService) {}
  @Get() findAll() { return this.tasks.findAll(); }
}

@Controller({ path: 'tasks', version: '2' })
export class TasksV2Controller {
  constructor(private readonly tasks: TasksService) {}
  @Get()
  async findAll(): Promise<TaskV2Dto[]> {
    const tasks = await this.tasks.findAll();
    return tasks.map(({ title, ...rest }) => ({ ...rest, name: title }));
  }
}
```
The shape change lives in the controller; the service is untouched, so there is one implementation of the business logic.
</details>

**24.** Scenario — A frontend developer asks you to add `?fields=id,title` so they can request only the columns they need, to cut payload size. Do you build it?
<details><summary>Answer</summary>Not as a general mechanism. Arbitrary field selection turns every endpoint's response into an unbounded contract: you can no longer change a column without checking which clients name it, caching keys multiply by every field combination, and validating that the caller is allowed to see each requested field is a per-field authorization problem you now own. Measure first — if the payload is genuinely the bottleneck, the usual causes are unbounded page sizes and serialized relations, both fixable without a new API surface. If the need is real and specific, add named views (`?view=summary`) with a fixed set of shapes you control, or expose GraphQL where field selection is the point and the tooling handles it (→ Ch 24).</details>

**25.** Scenario — Your team must change a response field's type from string to object. Some mobile clients cannot be forced to update. How do you proceed?
<details><summary>Answer</summary>Ship a new version rather than mutating the existing field, because old clients will crash on a type change in a way they will not on an added field. Add a v2 controller with the new shape, keep v1 delegating to the same service with a mapping layer, and instrument v1 so you know how much traffic is still on it. Set a deprecation window with a `Sunset` header and communicate it, then remove v1 when usage is near zero. The alternative sometimes floated — adding a parallel field and leaving both forever — is cheaper today and leaves you maintaining two representations of the same data indefinitely, which is how response payloads become unexplainable.</details>

---

## Chapter Summary

- A route is the controller prefix plus the method decorator; the return value becomes the body, and promises and observables are awaited for you.
- `POST` defaults to 201 and everything else to 200, so action-shaped POSTs need an explicit `@HttpCode(200)`.
- Path and query values are always strings — the type annotation is erased, so conversion needs a pipe or `@Type()` plus `transform`.
- Routes match in registration order, which makes `@Get(':id')` above `@Get('me')` a silent bug rather than an error.
- Bare `@Res()` opts the handler out of Nest's response pipeline, breaking response-transforming interceptors on that route only; `passthrough: true` avoids that.
- Version from the first release: retrofitting versioning means every existing URL is already the legacy case.

---

## Practice Exercises

1. Build a controller with five routes and deliberately order them so a static route is unreachable. Prove it with `curl`, then fix it.
2. Write one endpoint twice — once extracting six query parameters individually, once with a DTO — and compare the Swagger output.
3. Add an endpoint that sets a cookie and returns a body, using `@Res({ passthrough: true })`. Then remove `passthrough` and observe what breaks.
4. Implement all four versioning types on the same controller and note what each does to a browser request.
5. Write an endpoint whose `limit` is unbounded, measure the response time at 100,000 rows, then add `@Max()` and measure again.

---

## Mini Project: A Read-Only Search API

Build a search endpoint over an in-memory dataset of at least 1,000 records.

**Requirements:**

- `GET /v1/search` accepting `q`, `page`, `limit`, `sort`, and `order`, all validated and bounded.
- A stable response envelope with `data` and `meta`, where `meta` includes `total` and `totalPages`.
- Static routes (`/search/suggestions`, `/search/stats`) that are never captured by a dynamic route.
- Correct status codes throughout, including 204 for an endpoint that returns nothing.
- A second version of the endpoint with a renamed field, sharing one service with v1.

---

**Next:** [Chapter 6 — Providers and Services →](06-providers-services.md)
