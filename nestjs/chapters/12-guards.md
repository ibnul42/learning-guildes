# Chapter 12 — Guards

## Learning Objectives

- Implement `CanActivate` and choose between returning `false` and throwing
- Read route and controller metadata with `Reflector`, and pick the right merge strategy
- Build a global authentication guard that respects a `@Public()` decorator
- Use `ExecutionContext` to write guards that work across HTTP, WebSocket, and RPC transports
- Separate authentication from authorization, and know where ownership checks belong

---

## 12.1 A Guard Answers One Question

A guard decides whether a request proceeds. That is its entire contract:

```ts
// src/auth/api-key.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';

@Injectable()
export class ApiKeyGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest<Request>();
    return request.headers['x-api-key'] === process.env.API_KEY;
  }
}
```

`canActivate` may return a boolean, a promise, or an observable. Returning `false` rejects the request; anything truthy lets it through.

Guards run after middleware and before interceptors and pipes (→ [Ch 10](10-request-lifecycle.md)). That position is deliberate: a guard is the cheapest place to reject, because nothing has been validated or handled yet.

**Returning `false` always produces 403 Forbidden.** That is wrong for a missing credential, which is 401:

```ts
// ❌ A missing token reports 403 — "you may not", when the truth is "who are you?"
if (!token) return false;

// ✅ Throwing gives the right status and a usable message
if (!token) throw new UnauthorizedException('Missing bearer token');
```

The distinction matters to clients: a 401 tells them to authenticate and retry, a 403 tells them not to bother. Getting it wrong sends well-behaved clients into pointless retry loops or stops them retrying when they should.

> 💼 **Interview** — "What is the difference between a guard and middleware?" The answer is `ExecutionContext`: a guard knows which handler is about to run and can read its metadata, so authorization decisions can live next to the routes they protect.

---

## 12.2 `ExecutionContext`

`ExecutionContext` is what makes guards more capable than middleware. It exposes both the transport and the target:

```ts
canActivate(context: ExecutionContext) {
  context.getClass();      // the controller class
  context.getHandler();    // the method about to run
  context.getType();       // 'http' | 'ws' | 'rpc' | 'graphql'

  context.switchToHttp().getRequest();
  context.switchToWs().getClient();
  context.switchToRpc().getData();
}
```

`getClass()` and `getHandler()` are the metadata anchors — they are what you pass to `Reflector` to read decorators.

Because the transport is discoverable, one guard can serve several:

```ts
private getRequest(context: ExecutionContext): { headers: Record<string, string> } {
  switch (context.getType()) {
    case 'ws':
      return context.switchToWs().getClient().handshake;
    case 'rpc':
      return context.switchToRpc().getContext();
    default:
      return context.switchToHttp().getRequest();
  }
}
```

> ⚠️ **Gotcha** — A guard written with `context.switchToHttp()` throws or returns `undefined` when applied to a WebSocket gateway or a microservice handler. A global guard added for HTTP silently breaks other transports the moment they are introduced, so branch on `getType()` in any guard you register globally.

---

## 12.3 `Reflector` and Metadata

The pattern that makes guards powerful: a decorator records intent on the route, and the guard reads it.

```ts
// src/auth/roles.decorator.ts
export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);
```

```ts
// src/auth/roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const required = this.reflector.getAllAndOverride<string[]>(ROLES_KEY, [
      context.getHandler(),      // route metadata — checked first
      context.getClass(),        // controller metadata — fallback
    ]);

    if (!required?.length) return true;          // no roles declared → unrestricted

    const { user } = context.switchToHttp().getRequest();
    if (!user) throw new UnauthorizedException();

    return required.some((role) => user.roles.includes(role));
  }
}
```

`Reflector` offers four methods, and choosing wrongly is a real source of bugs:

| Method | Behaviour |
|---|---|
| `get(key, target)` | reads one target |
| `getAll(key, targets)` | returns an array of each target's value |
| `getAllAndOverride(key, targets)` | **first non-undefined wins** — most specific overrides |
| `getAllAndMerge(key, targets)` | **combines** all values into one array |

The order of the targets array is what defines "most specific". Passing `[getHandler(), getClass()]` means route metadata overrides controller metadata, which is almost always what you want.

**Override versus merge is a security decision.** For roles, `getAllAndOverride` lets a route relax its controller's requirement — `@Roles('admin')` on the controller and `@Roles('user')` on one route means that route accepts users. With `getAllAndMerge` the route would require both. Neither is universally right, but pick deliberately:

```ts
// Controller: @Roles('admin')   Route: @Roles('user')
getAllAndOverride → ['user']            // route wins: relaxed
getAllAndMerge    → ['admin', 'user']   // combined: with .some(), still relaxed
```

Note that merging with a `.some()` check still relaxes rather than tightens. If you want the controller's requirement to be a floor that routes cannot lower, enforce it explicitly rather than relying on the reflector.

---

## 12.4 A Global Auth Guard With `@Public()`

The standard production shape: authenticate everything by default, opt out explicitly.

```ts
// src/auth/public.decorator.ts
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

```ts
// src/auth/jwt-auth.guard.ts
@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(
    private readonly reflector: Reflector,
    private readonly jwt: JwtService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;

    const request = context.switchToHttp().getRequest();
    const token = request.headers.authorization?.replace(/^Bearer /, '');
    if (!token) throw new UnauthorizedException('Missing bearer token');

    try {
      request.user = await this.jwt.verifyAsync(token);
    } catch {
      // Never surface the underlying reason — "expired" vs "malformed"
      // tells an attacker whether a token was ever valid.
      throw new UnauthorizedException('Invalid token');
    }
    return true;
  }
}
```

```ts
// src/app.module.ts
@Module({
  providers: [{ provide: APP_GUARD, useClass: JwtAuthGuard }],
})
export class AppModule {}
```

**Secure by default is the point.** With an opt-in guard, forgetting `@UseGuards()` leaves an endpoint open and nothing fails. With a global guard, forgetting `@Public()` makes an endpoint return 401 — a visible, immediate, harmless failure. The two mistakes have very different costs.

Register it with `APP_GUARD`, not `useGlobalGuards(new JwtAuthGuard())`, because the latter cannot inject `Reflector` or `JwtService` (→ [Ch 8](08-custom-providers.md)).

Multiple `APP_GUARD` registrations accumulate and run in order, so authentication and authorization can be separate guards:

```ts
providers: [
  { provide: APP_GUARD, useClass: JwtAuthGuard },   // establishes request.user
  { provide: APP_GUARD, useClass: RolesGuard },     // reads it
]
```

Order matters here — `RolesGuard` depends on `JwtAuthGuard` having populated `request.user`.

---

## 12.5 Where Ownership Checks Belong

"Can this user edit *this* record" is different from "is this user an admin", and guards handle it badly.

```ts
// ❌ The guard loads the task, then the service loads it again
@Injectable()
export class TaskOwnerGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const task = await this.tasks.findOne(request.params.id);   // query #1
    return task?.ownerId === request.user.id;
  }
}
```

The guard cannot pass the loaded record to the handler, so it is fetched twice. Worse, the guard's 403 and the service's own not-found handling can disagree — a task that does not exist returns 403 rather than 404, which leaks whether an id is in use.

Two better options:

**Do the check in the service**, where the record is already loaded:

```ts
async update(id: string, dto: UpdateTaskDto, actorId: string): Promise<Task> {
  const task = await this.repo.findById(id);
  if (!task) throw new NotFoundException();
  if (task.ownerId !== actorId) throw new ForbiddenException();
  return this.repo.save(Object.assign(task, dto));
}
```

**Or scope the query**, so ownership is a filter rather than a check — the strongest option, because a row the caller cannot see simply is not returned.

Use guards for decisions derivable from the request alone: authentication, roles, feature flags, rate limits, IP allowlists. Use the service layer for decisions requiring the record.

---

## 12.6 Common Mistakes and Best Practices

**Mistake: returning `false` for a missing credential.** It reports 403 when the correct answer is 401, which changes how clients retry.

**Mistake: `useGlobalGuards(new Guard())` for a guard with dependencies.** It cannot inject; use `APP_GUARD`.

**Mistake: assuming `request.user` exists.** It only does if an earlier guard set it, and guard order is registration order.

**Mistake: `switchToHttp()` in a globally-registered guard.** It breaks the moment a WebSocket gateway or microservice handler is added.

**Mistake: ownership checks in guards.** The record gets loaded twice, and a missing record returns 403 instead of 404.

**Mistake: leaking why a token failed.** "Expired" versus "malformed" tells an attacker whether a token was ever valid.

**Mistake: expensive work in a guard.** It runs on every request to every route it covers, including ones that will fail validation moments later.

**Best practice: authenticate globally, opt out with `@Public()`.** Forgetting the decorator produces a visible 401 rather than an open endpoint.

**Best practice: separate authentication and authorization into different guards.** Each has one reason to change, and the order documents the dependency.

**Best practice: choose `getAllAndOverride` versus `getAllAndMerge` deliberately.** It determines whether a route can relax its controller's requirement.

---

## 12.7 Real-World Example: Layered Authentication and Authorization

Three guards, each with one job, composed into a single decorator so endpoints declare policy rather than mechanism.

```ts
// src/auth/auth.decorators.ts
export const IS_PUBLIC_KEY = 'isPublic';
export const ROLES_KEY = 'roles';
export const SCOPES_KEY = 'scopes';

export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);
export const Scopes = (...scopes: string[]) => SetMetadata(SCOPES_KEY, scopes);

/** One decorator per policy, so an endpoint cannot half-apply the stack. */
export function Auth(options: { roles?: Role[]; scopes?: string[] } = {}) {
  return applyDecorators(
    ...(options.roles ? [Roles(...options.roles)] : []),
    ...(options.scopes ? [Scopes(...options.scopes)] : []),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Missing or invalid token' }),
    ApiForbiddenResponse({ description: 'Insufficient role or scope' }),
  );
}
```

```ts
// src/auth/jwt-auth.guard.ts
@Injectable()
export class JwtAuthGuard implements CanActivate {
  private readonly logger = new Logger(JwtAuthGuard.name);

  constructor(
    private readonly reflector: Reflector,
    private readonly jwt: JwtService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    if (this.isPublic(context)) return true;

    const request = this.getRequest(context);
    const token = this.extractToken(request);
    if (!token) throw new UnauthorizedException('Missing bearer token');

    try {
      request.user = await this.jwt.verifyAsync<JwtPayload>(token);
      return true;
    } catch (err) {
      // Log the real reason; return a generic one.
      this.logger.debug(`Token rejected: ${(err as Error).message}`);
      throw new UnauthorizedException('Invalid token');
    }
  }

  private isPublic(context: ExecutionContext): boolean {
    return (
      this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
        context.getHandler(),
        context.getClass(),
      ]) ?? false
    );
  }

  /** Transport-aware, so registering this globally does not break gateways. */
  private getRequest(context: ExecutionContext): any {
    return context.getType() === 'ws'
      ? context.switchToWs().getClient().handshake
      : context.switchToHttp().getRequest();
  }

  private extractToken(request: { headers?: Record<string, string> }): string | null {
    const header = request.headers?.authorization;
    if (!header?.startsWith('Bearer ')) return null;
    return header.slice(7) || null;
  }
}
```

```ts
// src/auth/roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const required = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!required?.length) return true;

    const { user } = context.switchToHttp().getRequest();
    // Defensive: if JwtAuthGuard did not run, fail closed rather than crash.
    if (!user) throw new UnauthorizedException();

    if (!required.some((role) => user.roles?.includes(role))) {
      throw new ForbiddenException(`Requires one of: ${required.join(', ')}`);
    }
    return true;
  }
}
```

```ts
// src/app.module.ts — order is the dependency: authenticate, then authorize
@Module({
  providers: [
    { provide: APP_GUARD, useClass: JwtAuthGuard },
    { provide: APP_GUARD, useClass: RolesGuard },
    { provide: APP_GUARD, useClass: ScopesGuard },
  ],
})
export class AppModule {}
```

```ts
// src/tasks/tasks.controller.ts
@Controller('tasks')
export class TasksController {
  @Public()                                    // opt out of the global guard
  @Get('public-stats')
  publicStats() {}

  @Auth()                                      // authenticated, any role
  @Get()
  findAll() {}

  @Auth({ roles: [Role.Admin], scopes: ['tasks:delete'] })
  @Delete(':id')
  remove(@Param('id') id: string, @CurrentUser('id') actorId: string) {
    // Ownership is checked in the service, where the record is already loaded
    return this.tasks.remove(id, actorId);
  }
}
```

The decision worth defending is `RolesGuard` throwing `UnauthorizedException` when `request.user` is absent, rather than trusting that `JwtAuthGuard` ran. It looks like defensive clutter — the guards are registered in order, so `user` should always exist. But that ordering is an invariant held in one providers array, and it can be broken by anything: a reordered registration during a refactor, a route where someone applies `RolesGuard` directly with `@UseGuards()`, or a future transport where the authentication guard's `switchToHttp` path does not populate the same object. If `user` is undefined and the guard reads `user.roles`, the result is a `TypeError` producing a 500 — which is a crash, not a denial, and 500s are frequently retried and sometimes routed past authorization by an upstream proxy. Failing closed with an explicit 401 costs one line and converts a possible security hole into a correct rejection. Security components should never depend on another component having run; they should verify it.

---

## 12.8 Questions

**1.** MCQ — Returning `false` from a guard produces: a) 400 b) 401 c) 403 d) 500
<details><summary>Answer</summary>**c)** 403 Forbidden. Throw `UnauthorizedException` for 401.</details>

**2.** MCQ — Which does a guard have that middleware does not? a) the request b) `ExecutionContext` c) the response d) headers
<details><summary>Answer</summary>**b)** It exposes the target class and handler, enabling metadata-driven decisions.</details>

**3.** MCQ — `getAllAndOverride([handler, class])` returns: a) all values merged b) the first non-undefined c) the class value d) an array
<details><summary>Answer</summary>**b)** First non-undefined, so route metadata overrides controller metadata.</details>

**4.** MCQ — Multiple `APP_GUARD` registrations: a) conflict b) all run in order c) only the last runs d) throw
<details><summary>Answer</summary>**b)** The token is multi-provider; they run in registration order.</details>

**5.** MCQ — `canActivate` may return: a) boolean only b) boolean or promise c) boolean, promise, or observable d) void
<details><summary>Answer</summary>**c)**</details>

**6.** MCQ — A global guard using `switchToHttp()` breaks when: a) never b) a WebSocket gateway is added c) validation fails d) under load
<details><summary>Answer</summary>**b)** Non-HTTP transports need `getType()` branching.</details>

**7.** Short — Why authenticate globally and opt out with `@Public()`?
<details><summary>Answer</summary>Forgetting the decorator yields a visible 401 on a public route, whereas with opt-in guards forgetting `@UseGuards()` silently leaves an endpoint open — the failure modes have very different costs.</details>

**8.** Short — Why is `getAllAndOverride` versus `getAllAndMerge` a security decision?
<details><summary>Answer</summary>It determines whether route metadata replaces or combines with controller metadata, which decides whether a route can relax a requirement the controller declared.</details>

**9.** Short — Why do ownership checks belong in the service rather than a guard?
<details><summary>Answer</summary>A guard cannot hand the loaded record to the handler, so the record is fetched twice, and a missing record produces 403 instead of 404, leaking whether an id exists.</details>

**10.** Short — Why should a token failure not report why it failed?
<details><summary>Answer</summary>Distinguishing "expired" from "malformed" tells an attacker whether a token was ever valid, which is useful information for a forgery attempt.</details>

**11.** Short — Why must `RolesGuard` be registered after `JwtAuthGuard`?
<details><summary>Answer</summary>It reads `request.user`, which the authentication guard populates. Guards run in registration order.</details>

**12.** Output — Controller has `@Roles('admin')`, route has `@Roles('user')`. What does `getAllAndOverride(ROLES_KEY, [getHandler(), getClass()])` return?
<details><summary>Answer</summary>`['user']` — the handler is checked first and its value wins.</details>

**13.** Output — Same setup with `getAllAndMerge`?
<details><summary>Answer</summary>`['user', 'admin']` — both values combined. With a `.some()` check this still admits users, so merging does not tighten the requirement.</details>

**14.** Output — A guard throws `UnauthorizedException`. Do interceptors' post-halves run?
<details><summary>Answer</summary>No. The exception path skips to the filters; nothing downstream of the guard executes.</details>

**15.** Output — `RolesGuard` runs before `JwtAuthGuard` and reads `user.roles`. What is the HTTP status?
<details><summary>Answer</summary>500 — `user` is undefined, so reading `.roles` throws a `TypeError`, which is not an `HttpException`. This is why guards should verify rather than assume.</details>

**16.** Debug — Every endpoint returns 401 after adding a global guard, including the login route. Fix?
<details><summary>Answer</summary>Add `@Public()` to the login and registration routes, and confirm the guard reads `IS_PUBLIC_KEY` with both `getHandler()` and `getClass()`.</details>

**17.** Debug — A guard's injected `Reflector` is undefined. How was it registered?
<details><summary>Answer</summary>With an instance — `@UseGuards(new RolesGuard())` or `useGlobalGuards(new RolesGuard())`. Pass the class so the container constructs it, or use `APP_GUARD`.</details>

**18.** Debug — A WebSocket gateway rejects every connection after a global HTTP guard was added. Cause?
<details><summary>Answer</summary>The guard calls `switchToHttp()`, which does not yield a usable request for a WS context. Branch on `context.getType()`.</details>

**19.** Debug — Requesting a non-existent task returns 403 rather than 404. Where is the check?
<details><summary>Answer</summary>In an ownership guard that loads the record and returns false when it is missing. Move the check into the service so a missing record produces 404.</details>

**20.** Debug — `@Roles('admin')` on a controller is ignored on one route that has its own `@Roles()` with no arguments. Why?
<details><summary>Answer</summary>`@Roles()` with no arguments sets an empty array, which is defined, so `getAllAndOverride` returns `[]` and the guard treats "no roles required" as unrestricted.</details>

**21.** Code — Write a `@Public()` decorator and a guard that honours it.
<details><summary>Answer</summary>

```ts
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}
  canActivate(ctx: ExecutionContext): boolean {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      ctx.getHandler(),
      ctx.getClass(),
    ]);
    if (isPublic) return true;
    const req = ctx.switchToHttp().getRequest();
    if (!req.headers.authorization) throw new UnauthorizedException();
    return true;
  }
}
```
</details>

**22.** Code — Write a guard enforcing a per-user rate limit, returning 429.
<details><summary>Answer</summary>

```ts
@Injectable()
export class RateLimitGuard implements CanActivate {
  constructor(private readonly cache: CacheService) {}

  async canActivate(ctx: ExecutionContext): Promise<boolean> {
    const req = ctx.switchToHttp().getRequest();
    const key = `rl:${req.user?.id ?? req.ip}`;
    const count = await this.cache.increment(key, 60);   // 60s window
    if (count > 100) {
      throw new HttpException('Rate limit exceeded', HttpStatus.TOO_MANY_REQUESTS);
    }
    return true;
  }
}
```
Keying on the user when present and the IP otherwise prevents one user behind a shared NAT from exhausting everyone's quota.
</details>

**23.** Code — Write a transport-agnostic guard that reads a header on HTTP and a handshake header on WebSockets.
<details><summary>Answer</summary>

```ts
@Injectable()
export class TokenGuard implements CanActivate {
  canActivate(ctx: ExecutionContext): boolean {
    const headers =
      ctx.getType() === 'ws'
        ? ctx.switchToWs().getClient().handshake?.headers
        : ctx.switchToHttp().getRequest().headers;

    if (!headers?.authorization) throw new UnauthorizedException();
    return true;
  }
}
```
</details>

**24.** Scenario — A teammate wants one `AuthGuard` doing authentication, role checks, scope checks, and rate limiting, arguing that four guards means four passes over the request. Evaluate it.
<details><summary>Answer</summary>The performance argument is negligible — the four guards share the same request object and the work is identical whether it sits in one method or four. The cost of combining them is that the class has four reasons to change and no way to apply them selectively: a route needing rate limiting but not roles gets both, and disabling one for a health endpoint means adding a flag to a class that is now security-critical and conditionally branched. Separate guards also fail independently and produce accurate status codes — 401 from authentication, 403 from roles, 429 from the limiter — whereas a combined guard tends to collapse these into whichever the author wrote first. Keep them separate and compose them with a decorator so endpoints still declare one thing.</details>

**25.** Scenario — Your API needs to support both JWTs and long-lived API keys, with API keys restricted to a subset of endpoints. How do you structure the guards?
<details><summary>Answer</summary>Use one authentication guard that recognises both credential types and normalises them onto `request.user` with a `principalType` field, rather than two guards racing to authenticate — two guards means the second must know whether the first succeeded, which is exactly the implicit ordering dependency worth avoiding. Endpoint restriction is then a separate authorization concern: a decorator such as `@AllowPrincipals('jwt')` plus a guard reading it, so the default is that API keys are refused and each endpoint opts in explicitly. That ordering matters because the safe default for a long-lived credential is no access, and an allowlist fails closed when someone adds an endpoint and forgets. Give API keys their own scopes rather than reusing user roles, since a key is not a person and will otherwise inherit permissions nobody intended to grant.</details>

---

## Chapter Summary

- A guard answers whether a request proceeds; returning `false` always yields 403, so throw `UnauthorizedException` when the honest answer is 401.
- `ExecutionContext` is what separates guards from middleware — it exposes the target handler, which is what makes decorator-driven authorization possible.
- `Reflector.getAllAndOverride` lets route metadata override controller metadata; `getAllAndMerge` combines them, and the choice determines whether a route can relax a requirement.
- Authenticate globally with `APP_GUARD` and opt out with `@Public()`, because forgetting the decorator fails visibly instead of leaving an endpoint open.
- A globally-registered guard must branch on `getType()`, or it will break the first time a non-HTTP transport is added.
- Ownership checks belong in the service where the record is already loaded — in a guard they double-fetch and turn a missing record into a 403.

---

## Practice Exercises

1. Write a guard that returns `false` and one that throws `UnauthorizedException`, then compare the responses a client sees.
2. Apply `@Roles()` at both controller and route level and observe the difference between `getAllAndOverride` and `getAllAndMerge`.
3. Register two `APP_GUARD`s in the wrong order and capture the resulting 500.
4. Add a WebSocket gateway to an app with a global HTTP guard and fix the breakage with `getType()`.
5. Move an ownership check from a guard into the service and count the database queries before and after.

---

## Mini Project: A Policy-Based Authorization Layer

Build authorization that supports roles, scopes, and ownership without duplicating logic.

**Requirements:**

- A global authentication guard establishing a normalised principal for both JWTs and API keys.
- `@Auth({ roles, scopes })` composing metadata, guards, and Swagger annotations in one decorator.
- Ownership enforced in the data layer as a query filter, never as a post-load check.
- Every guard fails closed if an expected upstream guard did not run.
- Tests covering: public route, missing token, expired token, wrong role, insufficient scope, and non-owner.

---

**Next:** [Chapter 13 — Interceptors →](13-interceptors.md)
