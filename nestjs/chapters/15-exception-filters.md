# Chapter 15 — Exception Filters

## Learning Objectives

- Throw the right built-in exception and know what status each produces
- Write a filter that catches a specific exception type, and one that catches everything
- Design an error contract clients can program against
- Keep HTTP status codes out of your service layer
- Avoid the two filters that turn a working API into an opaque one

---

## 15.1 Throwing Is a Normal Way to Respond

In Nest, `throw` is not an escape hatch — it is the idiomatic way to end a request with a non-2xx status.

```ts
@Get(':id')
async findOne(@Param('id') id: string): Promise<Task> {
  const task = await this.tasks.findOne(id);
  if (!task) throw new NotFoundException(`Task ${id} not found`);
  return task;
}
```

Nest ships a built-in exception for every status you are likely to need:

| Exception | Status | Use for |
|---|---|---|
| `BadRequestException` | 400 | malformed input |
| `UnauthorizedException` | 401 | missing or invalid credentials |
| `ForbiddenException` | 403 | authenticated but not permitted |
| `NotFoundException` | 404 | no such resource |
| `MethodNotAllowedException` | 405 | wrong verb |
| `ConflictException` | 409 | duplicate, version conflict |
| `GoneException` | 410 | permanently removed |
| `PayloadTooLargeException` | 413 | body too big |
| `UnprocessableEntityException` | 422 | valid shape, invalid semantics |
| `InternalServerErrorException` | 500 | unexpected failure |
| `BadGatewayException` | 502 | upstream returned garbage |
| `ServiceUnavailableException` | 503 | overloaded, shutting down |
| `GatewayTimeoutException` | 504 | upstream too slow |

All extend `HttpException`, which you can use directly for anything unusual:

```ts
throw new HttpException(
  { code: 'QUOTA_EXCEEDED', retryAfter: 3600 },   // object body, not just a string
  HttpStatus.TOO_MANY_REQUESTS,
);
```

Anything thrown that is *not* an `HttpException` becomes a 500 with a generic body — Nest deliberately does not leak the message, because arbitrary errors contain connection strings, file paths, and query fragments.

> ⚠️ **Gotcha** — `throw new Error('Task not found')` produces a **500**, not a 404. The message is swallowed and the client gets "Internal server error". This is the most common cause of an endpoint that logs the right thing and returns the wrong status.

---

## 15.2 Writing a Filter

A filter catches exceptions and writes the response:

```ts
// src/common/filters/http-exception.filter.ts
import { ArgumentsHost, Catch, ExceptionFilter, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)                      // only HttpExceptions reach this filter
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    response.status(exception.getStatus()).json({
      statusCode: exception.getStatus(),
      path: request.url,
      timestamp: new Date().toISOString(),
      message: exception.getResponse(),
    });
  }
}
```

`@Catch()` decides what the filter handles:

```ts
@Catch(HttpException)                      // one type
@Catch(QueryFailedError, EntityNotFoundError)   // several
@Catch()                                   // EVERYTHING, including non-Error values
```

`ArgumentsHost` is the transport-agnostic parent of `ExecutionContext`. In a filter used across transports, branch on `host.getType()` before calling `switchToHttp()` — a WebSocket context has no `Response` object, and calling `.status()` on `undefined` inside an error handler produces an unhandleable failure.

Bind filters at any scope:

```ts
@UseFilters(HttpExceptionFilter)           // controller or route
@Controller('tasks')
export class TasksController {}

// global, as a provider so it can inject
{ provide: APP_FILTER, useClass: AllExceptionsFilter }
```

When filters exist at several scopes, the **most specific binding wins** — a route-bound filter takes precedence over a controller-bound one, which takes precedence over a global one.

The docs do not define the evaluation order when several *global* filters are registered, so do not build a design that depends on it. The reliable pattern is **one global catch-all**, with narrower filters bound at controller or route scope where genuinely needed.

> ⚠️ **Gotcha** — A method- or controller-scoped filter extending `BaseExceptionFilter` must **not** be instantiated with `new`. Use `@UseFilters(MyFilter)` and let Nest construct it, or the base class will not have the HTTP adapter it needs.

---

## 15.3 Keeping HTTP Out of the Service Layer

A service that throws `NotFoundException` has an opinion about HTTP. That is fine until the same service is called by a queue consumer, a scheduled job, or a GraphQL resolver, where a 404 means nothing.

The cleaner separation is domain errors in services, translated by a filter:

```ts
// src/common/errors/domain-errors.ts
export abstract class DomainError extends Error {
  abstract readonly code: string;
}

export class ResourceNotFound extends DomainError {
  readonly code = 'RESOURCE_NOT_FOUND';
  constructor(readonly resource: string, readonly id: string) {
    super(`${resource} ${id} not found`);
  }
}

export class BusinessRuleViolated extends DomainError {
  readonly code = 'RULE_VIOLATED';
  constructor(readonly rule: string, message: string) {
    super(message);
  }
}
```

```ts
// The service knows nothing about HTTP
async complete(id: string): Promise<Task> {
  const task = await this.repo.findById(id);
  if (!task) throw new ResourceNotFound('Task', id);
  if (task.status === 'archived') {
    throw new BusinessRuleViolated('no_archived_edits', 'Archived tasks cannot be completed');
  }
  return this.repo.save(task.complete());
}
```

```ts
// One filter owns the mapping
@Catch(DomainError)
export class DomainErrorFilter implements ExceptionFilter {
  private static readonly STATUS: Record<string, number> = {
    RESOURCE_NOT_FOUND: 404,
    RULE_VIOLATED: 422,
    PERMISSION_DENIED: 403,
  };

  catch(exception: DomainError, host: ArgumentsHost): void {
    const status = DomainErrorFilter.STATUS[exception.code] ?? 500;
    host.switchToHttp().getResponse<Response>().status(status).json({
      code: exception.code,
      message: exception.message,
    });
  }
}
```

The service is now reusable from any transport, and the mapping lives in one reviewable table.

This is worth the indirection in an application with several entry points. In a purely HTTP service, throwing `NotFoundException` directly is fine and simpler — apply the pattern when a second caller actually exists.

---

## 15.4 Designing an Error Contract

Clients need to program against errors, which means a stable shape. Nest's default is inconsistent: `message` is a string for most exceptions and an **array** for validation failures.

```json
{ "statusCode": 404, "message": "Task 42 not found", "error": "Not Found" }
{ "statusCode": 400, "message": ["email must be an email"], "error": "Bad Request" }
```

Any client parsing `message` must handle both types. Normalising this is one of the highest-value things a global filter does.

Two things make an error contract usable:

**A machine-readable code.** Clients should branch on `code === 'INSUFFICIENT_STOCK'`, never on a human-readable message that will be reworded.

**A correlation id.** Without one, "it failed at about 3pm" is the entire bug report.

RFC 9457 (which obsoleted RFC 7807) defines `application/problem+json` for exactly this, and adopting it means client libraries and API gateways already understand your errors:

```json
{
  "type": "https://api.example.com/errors/insufficient-stock",
  "title": "Insufficient stock",
  "status": 409,
  "detail": "Product abc has 2 units available, 5 requested",
  "instance": "/orders",
  "code": "INSUFFICIENT_STOCK",
  "correlationId": "018f2c...",
  "errors": [{ "field": "items[0].quantity", "code": "max", "message": "…" }]
}
```

---

## 15.5 What Not to Do in a Filter

Two filters routinely make things worse.

**The filter that swallows the status:**

```ts
// ❌ Every error becomes 500 — validation 400s, auth 401s, all of them
@Catch()
export class BadFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost): void {
    host.switchToHttp().getResponse().status(500).json({ message: 'Error' });
  }
}
```

A global `@Catch()` filter must check for `HttpException` and preserve its status. This one converts a well-behaved API into one that returns 500 for a mistyped email.

**The filter that leaks internals:**

```ts
// ❌ Ships stack traces, and often connection strings inside them
.json({ message: exception.message, stack: (exception as Error).stack })
```

Log the detail; return a code and a correlation id.

**The filter that throws.** An exception inside a filter has nowhere to go and produces an unhandled rejection, taking down the response entirely. Filters must be defensive — especially about `host.getType()` and about serialising unknown values.

---

## 15.6 Common Mistakes and Best Practices

**Mistake: `throw new Error(...)` for an expected condition.** It becomes a 500 with a generic message, not the status you meant.

**Mistake: a global `@Catch()` that ignores `HttpException`.** Every deliberate status collapses into 500.

**Mistake: returning stack traces or raw messages.** They contain paths, queries, and sometimes credentials.

**Mistake: `new` on a controller-scoped filter extending `BaseExceptionFilter`.** It will not have the HTTP adapter; pass the class instead.

**Mistake: relying on ordering between multiple global filters.** It is not specified; use one catch-all plus narrower bindings.

**Mistake: calling `switchToHttp()` unconditionally in a shared filter.** Non-HTTP contexts have no response object, and the failure happens inside your error handler.

**Mistake: clients branching on `message`.** Reword the message and every consumer breaks; provide a stable `code`.

**Best practice: one global catch-all normalising every error into one shape.** It is the only way a client can parse errors reliably.

**Best practice: include a correlation id in both the response and the log.** It turns an unreproducible report into a single log lookup.

**Best practice: log at the right level.** 4xx is the client's problem and belongs at `warn` or lower; 5xx is yours and belongs at `error`.

---

## 15.7 Real-World Example: A Production Error Contract

One global filter producing an RFC 9457-shaped response for every failure, preserving deliberate statuses, normalising validation errors, and never leaking internals.

```ts
// src/common/filters/all-exceptions.filter.ts
import {
  ArgumentsHost, Catch, ExceptionFilter, HttpException,
  HttpStatus, Logger,
} from '@nestjs/common';
import { Request, Response } from 'express';
import { RequestContext } from '../../context/request-context';
import { DomainError } from '../errors/domain-errors';

interface ProblemDetails {
  type: string;
  title: string;
  status: number;
  detail?: string;
  instance: string;
  code: string;
  correlationId: string;
  errors?: Array<{ field: string; code: string; message: string }>;
}

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name);

  private static readonly DOMAIN_STATUS: Record<string, number> = {
    RESOURCE_NOT_FOUND: HttpStatus.NOT_FOUND,
    RULE_VIOLATED: HttpStatus.UNPROCESSABLE_ENTITY,
    PERMISSION_DENIED: HttpStatus.FORBIDDEN,
    CONFLICT: HttpStatus.CONFLICT,
  };

  constructor(private readonly context: RequestContext) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    // Non-HTTP contexts have no response object; rethrow rather than crash
    // inside the error handler itself.
    if (host.getType() !== 'http') throw exception;

    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const correlationId = this.context.correlationId;

    const problem = this.toProblem(exception, request, correlationId);

    // 5xx is our fault and needs the stack; 4xx is the caller's and does not.
    if (problem.status >= 500) {
      this.logger.error(
        `${request.method} ${request.url} → ${problem.status} [${correlationId}]`,
        exception instanceof Error ? exception.stack : String(exception),
      );
    } else {
      this.logger.warn(
        `${request.method} ${request.url} → ${problem.status} ${problem.code} [${correlationId}]`,
      );
    }

    response
      .status(problem.status)
      .type('application/problem+json')
      .json(problem);
  }

  private toProblem(exception: unknown, request: Request, correlationId: string): ProblemDetails {
    const base = { instance: request.url, correlationId };

    if (exception instanceof HttpException) {
      const status = exception.getStatus();          // preserve the intended status
      const body = exception.getResponse();
      const validation = this.extractValidationErrors(body);

      return {
        ...base,
        type: `https://api.example.com/errors/${status}`,
        title: exception.name.replace(/Exception$/, ''),
        status,
        detail: typeof body === 'string' ? body : (body as any)?.message,
        code: (body as any)?.code ?? this.codeForStatus(status),
        ...(validation ? { errors: validation } : {}),
      };
    }

    if (exception instanceof DomainError) {
      const status = AllExceptionsFilter.DOMAIN_STATUS[exception.code] ?? 500;
      return {
        ...base,
        type: `https://api.example.com/errors/${exception.code.toLowerCase()}`,
        title: exception.code,
        status,
        detail: exception.message,
        code: exception.code,
      };
    }

    // Unknown: reveal nothing. The correlation id is the bridge to the log.
    return {
      ...base,
      type: 'https://api.example.com/errors/internal',
      title: 'Internal Server Error',
      status: HttpStatus.INTERNAL_SERVER_ERROR,
      code: 'INTERNAL_ERROR',
    };
  }

  /** ValidationPipe puts an array in `message`; every other exception puts a string. */
  private extractValidationErrors(body: unknown) {
    const message = (body as { message?: unknown })?.message;
    if (!Array.isArray(message)) return undefined;

    return message.map((m) => ({
      field: typeof m === 'string' ? (m.split(' ')[0] ?? 'unknown') : 'unknown',
      code: 'validation',
      message: String(m),
    }));
  }

  private codeForStatus(status: number): string {
    return (
      { 400: 'BAD_REQUEST', 401: 'UNAUTHENTICATED', 403: 'FORBIDDEN',
        404: 'NOT_FOUND', 409: 'CONFLICT', 429: 'RATE_LIMITED' }[status] ?? 'ERROR'
    );
  }
}
```

```ts
// src/app.module.ts
@Module({
  providers: [
    // One global catch-all. Narrower filters are bound per-controller where needed,
    // rather than registering several globals whose order is unspecified.
    { provide: APP_FILTER, useClass: AllExceptionsFilter },
  ],
})
export class AppModule {}
```

The decision worth defending is rethrowing when `host.getType() !== 'http'` rather than attempting to handle every transport. A filter registered with `APP_FILTER` catches exceptions from WebSocket gateways and microservice handlers too, and those contexts have no Express `Response` — so the natural-looking `host.switchToHttp().getResponse().status(...)` calls `.status()` on `undefined` *inside the error handler*, converting a recoverable error into an unhandled rejection that can take down the process. Rethrowing hands the exception back to Nest's default handling for that transport, which is correct behaviour rather than a fallback: an HTTP error contract is meaningless over a WebSocket, and pretending otherwise produces a response nobody can consume. The general rule is that error handlers must be the most defensive code in the system, because a failure there has nowhere left to go.

---

## 15.8 Questions

**1.** MCQ — `throw new Error('not found')` produces: a) 404 b) 400 c) 500 d) 200
<details><summary>Answer</summary>**c)** Only `HttpException` subclasses carry a status; anything else becomes a 500 with a generic body.</details>

**2.** MCQ — `@Catch()` with no arguments catches: a) nothing b) only `HttpException` c) everything d) only `Error`
<details><summary>Answer</summary>**c)** Every unhandled exception, including non-`Error` values.</details>

**3.** MCQ — When filters are bound at global, controller, and route scope, which applies? a) global b) controller c) route d) all three
<details><summary>Answer</summary>**c)** The most specific binding wins.</details>

**4.** MCQ — `ValidationPipe` puts what in `message`? a) a string b) an array c) an object d) null
<details><summary>Answer</summary>**b)** An array, unlike every other built-in exception — which is why a normalising filter is valuable.</details>

**5.** MCQ — A controller-scoped filter extending `BaseExceptionFilter` should be bound with: a) `new MyFilter()` b) the class c) `APP_FILTER` d) `useValue`
<details><summary>Answer</summary>**b)** Pass the class so Nest constructs it with the HTTP adapter.</details>

**6.** MCQ — RFC 9457 defines: a) JWT b) problem+json error format c) HTTP caching d) CORS
<details><summary>Answer</summary>**b)** `application/problem+json`, obsoleting RFC 7807.</details>

**7.** Short — Why should services throw domain errors rather than `NotFoundException`?
<details><summary>Answer</summary>HTTP status codes are meaningless to a queue consumer, a scheduled job, or a GraphQL resolver — keeping the service transport-agnostic lets one filter own the mapping.</details>

**8.** Short — Why must clients branch on a `code` rather than a message?
<details><summary>Answer</summary>Messages are human-facing and get reworded, translated, or made more specific; any client parsing them breaks on a copy edit.</details>

**9.** Short — What does a correlation id in an error response buy you?
<details><summary>Answer</summary>It turns an unreproducible user report into a single log lookup, without exposing any internal detail in the response.</details>

**10.** Short — Why must a global filter check for `HttpException`?
<details><summary>Answer</summary>Otherwise every deliberate status — 400 from validation, 401 from a guard — is rewritten as 500, and the API becomes uninformative.</details>

**11.** Short — Why should a shared filter check `host.getType()`?
<details><summary>Answer</summary>Non-HTTP contexts have no response object, so `switchToHttp().getResponse()` yields `undefined` and the filter itself throws — inside the error handler, where nothing can recover.</details>

**12.** Output — A service throws `new Error('Task 42 not found')`. What does the client receive?
<details><summary>Answer</summary>`500` with `{"statusCode":500,"message":"Internal server error"}`. The message is deliberately withheld.</details>

**13.** Output — `throw new HttpException({ code: 'QUOTA', retryAfter: 60 }, 429)`. What is the body?
<details><summary>Answer</summary>`{"code":"QUOTA","retryAfter":60}` with status 429 — an object passed to `HttpException` becomes the response body verbatim.</details>

**14.** Output — A global `@Catch()` filter always sets status 500. What does a failed `ValidationPipe` return?
<details><summary>Answer</summary>500, not 400. The filter caught the `BadRequestException` and discarded its status.</details>

**15.** Output — A filter throws while handling an exception. What happens?
<details><summary>Answer</summary>An unhandled rejection; the client gets no useful response and the process may terminate. Filters must never throw.</details>

**16.** Debug — An endpoint logs "Task not found" but returns 500. Cause?
<details><summary>Answer</summary>It throws a plain `Error` rather than `NotFoundException`. Only `HttpException` subclasses map to a status.</details>

**17.** Debug — After adding a global filter, validation errors lost their per-field messages. Why?
<details><summary>Answer</summary>The filter normalises `message` without handling the array form `ValidationPipe` produces. Detect the array and map it into a structured `errors` list.</details>

**18.** Debug — A WebSocket gateway crashes the process on any error after a global filter was added. Cause?
<details><summary>Answer</summary>The filter calls `switchToHttp().getResponse()`, which is undefined for a WS context, so it throws inside the handler. Branch on `host.getType()` and rethrow for non-HTTP.</details>

**19.** Debug — Error responses occasionally include a database connection string. Where from?
<details><summary>Answer</summary>The filter returns `exception.message` or a stack trace for unknown errors. Driver errors embed connection details — return a generic body and a correlation id, and log the rest.</details>

**20.** Debug — Two global filters are registered and the wrong one handles an error after an unrelated refactor. Diagnosis?
<details><summary>Answer</summary>The evaluation order for multiple global filters is unspecified, so the behaviour was never guaranteed. Use one global catch-all and bind narrower filters at controller or route scope.</details>

**21.** Code — Write a filter mapping a TypeORM unique-constraint violation to 409.
<details><summary>Answer</summary>

```ts
@Catch(QueryFailedError)
export class QueryFailedFilter implements ExceptionFilter {
  catch(exception: QueryFailedError & { code?: string }, host: ArgumentsHost): void {
    const res = host.switchToHttp().getResponse<Response>();
    const isUnique = exception.code === '23505';        // PostgreSQL unique_violation
    const status = isUnique ? HttpStatus.CONFLICT : HttpStatus.INTERNAL_SERVER_ERROR;

    res.status(status).json({
      code: isUnique ? 'DUPLICATE' : 'DB_ERROR',
      message: isUnique ? 'Resource already exists' : 'Database error',
    });
  }
}
```
The driver message is never returned — it contains the table, column, and offending value.
</details>

**22.** Code — Define a domain error hierarchy and the filter that maps it.
<details><summary>Answer</summary>

```ts
export abstract class DomainError extends Error {
  abstract readonly code: string;
}
export class NotFound extends DomainError { readonly code = 'NOT_FOUND'; }
export class Conflict extends DomainError { readonly code = 'CONFLICT'; }

@Catch(DomainError)
export class DomainErrorFilter implements ExceptionFilter {
  private static readonly MAP: Record<string, number> = { NOT_FOUND: 404, CONFLICT: 409 };

  catch(e: DomainError, host: ArgumentsHost): void {
    const status = DomainErrorFilter.MAP[e.code] ?? 500;
    host.switchToHttp().getResponse<Response>()
      .status(status).json({ code: e.code, message: e.message });
  }
}
```
</details>

**23.** Code — Write a filter that returns `application/problem+json` and preserves deliberate statuses.
<details><summary>Answer</summary>

```ts
@Catch()
export class ProblemFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost): void {
    if (host.getType() !== 'http') throw exception;

    const res = host.switchToHttp().getResponse<Response>();
    const req = host.switchToHttp().getRequest<Request>();
    const status = exception instanceof HttpException ? exception.getStatus() : 500;

    res.status(status).type('application/problem+json').json({
      type: `https://api.example.com/errors/${status}`,
      title: HttpStatus[status] ?? 'Error',
      status,
      instance: req.url,
    });
  }
}
```
</details>

**24.** Scenario — Your team wants every error to include the stack trace in staging but not production, controlled by an environment variable. Evaluate it.
<details><summary>Answer</summary>Reasonable in principle and risky in the detail. The value is real — staging errors are much faster to diagnose with a stack — but the control is a single environment variable, and the failure mode of getting it wrong is leaking file paths, queries, and sometimes credentials to the internet. Make the unsafe branch impossible rather than conditional: gate on an explicit allowlist of non-production environments rather than `!== 'production'`, so an unset or misspelled variable fails closed instead of open. Better still, do not put the stack in the response at all — log it with the correlation id and give staging users a way to look it up, which gets the same debugging benefit with none of the exposure. The principle is that any code path whose only protection is a string comparison against an environment name will eventually run in production.</details>

**25.** Scenario — Your API is consumed by a mobile app that cannot be updated quickly. You need to add a new error case. How do you introduce it without breaking old clients?
<details><summary>Answer</summary>Add the new case within an existing status code rather than inventing a new one, and carry the specificity in the `code` field. An old client branching on status alone continues to work — it sees a 409 and shows its generic conflict message — while updated clients branch on `code === 'SUBSCRIPTION_LAPSED'` and render something better. This is exactly why a stable machine-readable code alongside the status matters: the status is the coarse contract old clients depend on, and the code is the fine-grained one you can extend. What breaks old clients is changing the *status* for an existing condition, or moving from a string `message` to an object, so treat both as versioned surface. If the new case genuinely has no reasonable existing status, ship it behind a version or an opt-in header until adoption is high enough to make it the default.</details>

---

## Chapter Summary

- Throwing is the idiomatic way to return a non-2xx status, but only `HttpException` subclasses carry one — a plain `Error` becomes a 500 with its message withheld.
- `@Catch()` with no arguments catches everything, which makes it powerful and makes forgetting to preserve `HttpException` statuses a way to turn every deliberate 400 into a 500.
- The most specific filter binding wins, but ordering between multiple *global* filters is unspecified — use one catch-all plus narrower bindings.
- Services throwing domain errors stay usable from queues, jobs, and other transports, with one filter owning the status mapping.
- A usable error contract needs a stable machine-readable code and a correlation id; clients must never branch on a message, and responses must never carry stack traces.
- Filters are the last line of handling, so they must be the most defensive code in the application — check `host.getType()` and never throw.

---

## Practice Exercises

1. Throw a plain `Error`, an `HttpException`, and a custom domain error from the same endpoint and compare the three responses.
2. Write a global `@Catch()` filter that forgets to preserve statuses, then observe every validation error become a 500.
3. Normalise `ValidationPipe`'s array `message` into a structured `errors` list and verify the shape is identical across error types.
4. Add a correlation id to both the error response and the log, then trace one failed request end to end.
5. Register a global filter, add a WebSocket gateway, and fix the resulting crash with a `getType()` check.

---

## Mini Project: An RFC 9457 Error Layer

Build a complete, documented error contract for an API.

**Requirements:**

- Every failure returns `application/problem+json` with `type`, `title`, `status`, `code`, and `correlationId`.
- Validation failures include a structured `errors` array with field paths.
- Domain errors map to statuses through one reviewable table; services never import HTTP exceptions.
- Nothing internal is ever returned — verified by a test that throws a driver error containing a connection string and asserts it does not appear in the body.
- The error catalogue is generated into the OpenAPI document so clients can see every `code` they may receive.

---

**Next:** [Chapter 16 — Configuration and Environments →](16-configuration.md)
