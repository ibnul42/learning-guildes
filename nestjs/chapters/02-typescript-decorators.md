# Chapter 2 — TypeScript and Decorators for NestJS

## Learning Objectives

- Explain what a decorator compiles to, and why that makes NestJS possible
- Describe how `emitDecoratorMetadata` and `reflect-metadata` let the container read constructor types
- Predict the order in which decorators evaluate and apply
- Choose between a class and an interface for a DTO, and justify it at runtime
- Write a custom parameter decorator and a composed decorator

---

## 2.1 Why NestJS Needs TypeScript

Chapter 1 claimed the container reads a constructor's parameter types and supplies the instances. That should bother you, because TypeScript types are erased at compile time. This is valid TypeScript:

```ts
constructor(private readonly users: UsersService) {}
```

and it compiles to JavaScript with no trace of `UsersService` as a value in the signature:

```js
constructor(users) { this.users = users; }
```

So how does Nest know what to inject? Not from the type annotation — from **metadata the compiler emits alongside it**. That mechanism is the subject of this chapter, and it is the single piece of machinery that explains why Nest is the way it is: why DTOs must be classes, why interfaces can't be injected, and why a missing `@Injectable()` breaks resolution.

Get this chapter right and the rest of the framework stops being magic.

---

## 2.2 What a Decorator Actually Is

A decorator is a function that receives the thing it decorates and may modify it or record something about it. `@Controller('users')` is not syntax — it is a function call whose result is applied to the class.

```ts
// A decorator factory: called with arguments, returns the actual decorator
function Controller(prefix: string) {
  return function (target: Function) {         // `target` is the class constructor
    Reflect.defineMetadata('path', prefix, target);
  };
}

@Controller('users')
class UsersController {}

Reflect.getMetadata('path', UsersController); // 'users'
```

That is genuinely close to what `@Controller()` does. It attaches a string to the class and returns. It does not register a route, start a server, or wrap a method — **decorators in Nest almost never change behaviour directly**. They record intent. Later, at startup, Nest walks your classes and reads that recorded intent to decide what to build.

This is why decorators alone do nothing. A `@Get()` on a class that no module lists in `controllers` is inert metadata — no error, no route, no clue.

TypeScript has five decorator targets, and Nest uses all of them:

| Target | Signature receives | Nest example |
|---|---|---|
| Class | constructor | `@Controller()`, `@Injectable()`, `@Module()` |
| Method | target, key, descriptor | `@Get()`, `@UseGuards()` |
| Property | target, key | `@Column()` (TypeORM), `@IsEmail()` |
| Accessor | target, key, descriptor | rare |
| Parameter | target, key, index | `@Body()`, `@Param()`, `@Inject()` |

> ⚠️ **Gotcha** — These are TypeScript's **legacy** decorators, enabled by `experimentalDecorators: true`. TypeScript 5.0 also shipped the TC39 standard decorators, which are a different feature with different signatures. Nest 11 requires the legacy ones, because the standard proposal has no parameter decorators — and without those, `@Body()`, `@Param()`, and `@Inject()` cannot exist. Do not "modernise" your `tsconfig.json` by turning `experimentalDecorators` off.

---

## 2.3 The Trick: `emitDecoratorMetadata`

Here is the part that answers §2.1. With `emitDecoratorMetadata: true`, the TypeScript compiler emits type information as metadata — but **only for declarations that have at least one decorator**.

```ts
@Injectable()                                   // ← without this, no metadata is emitted
export class UsersController {
  constructor(private readonly users: UsersService) {}
}
```

compiles to roughly:

```js
let UsersController = class UsersController {
  constructor(users) { this.users = users; }
};
UsersController = __decorate([
  Injectable(),
  __metadata("design:paramtypes", [UsersService]),   // ← the constructor's types, as values
], UsersController);
```

`design:paramtypes` is an array of the actual constructor *functions*. Not names, not strings — the classes themselves. Nest reads it and now knows exactly what to construct:

```ts
Reflect.getMetadata('design:paramtypes', UsersController);
// [ class UsersService ]
```

Three metadata keys are emitted:

| Key | Applies to | Contains |
|---|---|---|
| `design:type` | properties, accessors | the declared type |
| `design:paramtypes` | constructors, methods | array of parameter types |
| `design:returntype` | methods | the declared return type |

Two consequences follow, and both are common interview questions.

**A class with no decorator gets no metadata.** This is the real reason `@Injectable()` is required on services that have dependencies — not because Nest needs the "injectable" flag, but because the decorator is what triggers metadata emission. A provider with an empty constructor works without it; add one dependency and resolution fails.

**Only types that exist at runtime can be recorded.** A class compiles to a function, so it survives. An interface compiles to nothing:

```ts
interface Logger { log(msg: string): void; }

@Injectable()
export class Service {
  constructor(private readonly logger: Logger) {}   // ❌ design:paramtypes records `Object`
}
```

Nest cannot resolve a token of `Object`, and you get the familiar error: *"Nest can't resolve dependencies of the Service (?). Please make sure that the argument dependency at index [0] is available."* The fix is an explicit token (→ [Ch 8](08-custom-providers.md)):

```ts
constructor(@Inject('LOGGER') private readonly logger: Logger) {}   // ✅
```

> 💼 **Interview** — "Why can't you inject an interface in NestJS?" Answer with the mechanism: interfaces are erased, `design:paramtypes` records `Object`, and the container has no token to look up. Then name the fix — a string or `Symbol` injection token, or an abstract class, which does survive compilation.

`reflect-metadata` is the polyfill providing `Reflect.defineMetadata`/`getMetadata`. Nest imports it internally, so you rarely import it yourself — but if you use decorators outside a Nest bootstrap (a standalone script, some test setups), you need `import 'reflect-metadata';` first, exactly once.

---

## 2.4 Decorator Evaluation Order

Two orders matter and they run in opposite directions.

**Decorator expressions evaluate top to bottom. Decorator functions apply bottom to top.**

```ts
function trace(label: string) {
  console.log(`eval ${label}`);                        // runs when the expression is evaluated
  return function (_t: any, _k?: any, _d?: any) {
    console.log(`apply ${label}`);                     // runs when the decorator is applied
  };
}

@trace('A')
@trace('B')
class Demo {}

// eval A
// eval B
// apply B
// apply A
```

Within a class, the application order is: instance members, then static members, then constructor parameters, then the class itself. So a class decorator always sees a class whose members have already been decorated — which is exactly why `@Controller()` can read the route metadata that `@Get()` left on the methods.

In practice you rarely depend on this, with one exception that does bite: when two decorators wrap the same method, the one *closest to the method* wraps first and therefore runs innermost.

```ts
@UseGuards(AuthGuard)        // outer
@UseInterceptors(LogInterceptor)
@Get()
findAll() {}
```

For guards and interceptors Nest controls the execution order itself (→ [Ch 10](10-request-lifecycle.md)), so decorator order does **not** determine pipeline order — a common and reasonable misconception. Ordering within the same *kind* is what decorator order affects: two interceptors run in the order listed.

---

## 2.5 Writing Custom Decorators

Three forms cover almost everything you will write.

**A parameter decorator** extracts something from the request:

```ts
// src/auth/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: keyof User | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;                  // put there by an auth guard
    return data ? user?.[data] : user;          // @CurrentUser('id') or @CurrentUser()
  },
);

// Usage — replaces `@Req() req` and `req.user` at every call site
@Get('me')
getProfile(@CurrentUser() user: User) { return user; }

@Get('my-id')
getId(@CurrentUser('id') id: string) { return id; }
```

**A metadata decorator** records something for a guard to read:

```ts
export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);

@Roles('admin')
@Delete(':id')
remove(@Param('id') id: string) {}
```

The guard then reads it with `Reflector` (→ [Ch 12](12-guards.md)). Note the constant: exporting `ROLES_KEY` rather than typing `'roles'` in two files is what keeps this from silently breaking when someone renames it.

**A composed decorator** bundles several into one:

```ts
import { applyDecorators, UseGuards, SetMetadata } from '@nestjs/common';

export function Auth(...roles: string[]) {
  return applyDecorators(
    SetMetadata(ROLES_KEY, roles),
    UseGuards(JwtAuthGuard, RolesGuard),
    ApiBearerAuth(),                    // Swagger
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}

@Auth('admin')                          // one decorator, four behaviours, applied consistently
@Delete(':id')
remove(@Param('id') id: string) {}
```

This is the highest-leverage pattern in the chapter. Four decorators repeated across thirty endpoints is thirty chances to forget one; `@Auth()` is one chance, in one file.

---

## 2.6 Types That Survive to Runtime

The erasure rule has a direct architectural consequence: **DTOs must be classes**.

```ts
// ❌ Erased at compile time. ValidationPipe has nothing to inspect,
//    Swagger generates an empty schema, and the type enforces nothing at runtime.
export interface CreateUserDto {
  email: string;
  age: number;
}

// ✅ Survives as a value. Decorators attach validation rules to real properties.
export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsInt()
  @Min(18)
  age: number;
}
```

With `ValidationPipe` registered, the class version rejects `{"email": 12345}` with a 400 and a per-field message. The interface version accepts it, and `email` is a number for the rest of the request's life.

The same rule explains a subtler trap. `class-transformer` needs to know the type of nested properties, and `design:type` records only `Array` for `Item[]` — the element type is gone:

```ts
export class CreateOrderDto {
  @ValidateNested({ each: true })
  @Type(() => OrderItemDto)          // ❌ omit this and nested validation silently passes
  items: OrderItemDto[];
}
```

Without `@Type()`, the items arrive as plain objects, `class-validator` has no class to validate against, and invalid nested data sails through. It fails open, which is the worst way for a validator to fail.

| Construct | Exists at runtime? | Usable as DI token? | Validatable? |
|---|---|---|---|
| `class` | Yes | Yes | Yes |
| `abstract class` | Yes | Yes | Yes |
| `interface` | No | No — needs `@Inject()` | No |
| `type` alias | No | No | No |
| `enum` | Yes (object) | Yes | Yes |
| `const enum` | No (inlined) | No | No |

Abstract classes are the useful middle ground: they give you an interface-shaped contract that still compiles to a function, so it works as a token directly.

```ts
export abstract class PaymentGateway {
  abstract charge(amountCents: number): Promise<Receipt>;
}

// Providers can now swap implementations against a real token
{ provide: PaymentGateway, useClass: StripeGateway }
```

---

## 2.7 Common Mistakes and Best Practices

**Mistake: forgetting `@Injectable()` on a service with dependencies.** Without a decorator the compiler emits no `design:paramtypes`, so the container sees a constructor it cannot describe and resolution fails.

**Mistake: declaring DTOs as interfaces.** They vanish at compile time, so validation and Swagger both silently do nothing.

**Mistake: omitting `@Type()` on nested DTO arrays.** Nested validation fails open — invalid data is accepted rather than rejected.

**Mistake: injecting an interface.** `design:paramtypes` records `Object`; use an abstract class or an explicit `@Inject()` token.

**Mistake: assuming decorator order sets pipeline order.** Nest fixes the guard → interceptor → pipe sequence itself; decorator order only sequences items of the same kind.

**Mistake: turning off `experimentalDecorators` to adopt standard decorators.** The standard proposal has no parameter decorators, so the entire `@Body()`/`@Inject()` family stops compiling.

**Mistake: `import 'reflect-metadata'` in every file.** It patches a global once; repeating it is noise, and omitting it entirely only matters outside a Nest bootstrap.

**Best practice: export metadata keys as constants.** `SetMetadata(ROLES_KEY, …)` and `reflector.get(ROLES_KEY, …)` in different files must never drift apart.

**Best practice: compose repeated decorator stacks with `applyDecorators`.** One named decorator per policy beats four decorators copy-pasted per endpoint.

**Best practice: prefer abstract classes over interfaces for anything you inject.** You keep the contract and gain a usable runtime token.

---

## 2.8 Real-World Example: A Typed `@CurrentUser()` With a Guard Contract

A parameter decorator is only as trustworthy as the guard that populates it. This pair makes the relationship explicit and type-safe, so `@CurrentUser()` cannot be used on an unguarded route without it being obvious.

```ts
// src/auth/authenticated-request.ts
import { Request } from 'express';

export interface AuthenticatedUser {
  id: string;
  email: string;
  roles: readonly string[];
}

// Narrowing the Express request means `request.user` is typed everywhere,
// instead of `any` leaking out of every guard and decorator.
export interface AuthenticatedRequest extends Request {
  user: AuthenticatedUser;
}
```

```ts
// src/auth/current-user.decorator.ts
import { createParamDecorator, ExecutionContext, InternalServerErrorException } from '@nestjs/common';
import { AuthenticatedRequest, AuthenticatedUser } from './authenticated-request';

/**
 * Reads the user placed on the request by JwtAuthGuard.
 *
 * Overloaded so `@CurrentUser()` yields the whole user and
 * `@CurrentUser('id')` yields one field, both correctly typed.
 */
export const CurrentUser = createParamDecorator(
  <K extends keyof AuthenticatedUser>(
    field: K | undefined,
    ctx: ExecutionContext,
  ): AuthenticatedUser | AuthenticatedUser[K] => {
    const request = ctx.switchToHttp().getRequest<AuthenticatedRequest>();

    // Fail loudly. A missing user means the route is unguarded — a routing
    // bug, not a client error, so it must not surface as a 401.
    if (!request.user) {
      throw new InternalServerErrorException(
        'CurrentUser used on a route with no authentication guard',
      );
    }

    return field ? request.user[field] : request.user;
  },
);
```

```ts
// src/auth/auth.decorator.ts
import { applyDecorators, SetMetadata, UseGuards } from '@nestjs/common';

export const ROLES_KEY = 'roles';

/** One policy decorator: authentication, role metadata, and role enforcement. */
export function Auth(...roles: string[]) {
  return applyDecorators(
    SetMetadata(ROLES_KEY, roles),
    UseGuards(JwtAuthGuard, RolesGuard),
  );
}
```

```ts
// src/users/users.controller.ts
@Controller('users')
export class UsersController {
  @Auth()                                   // authenticated, any role
  @Get('me')
  me(@CurrentUser() user: AuthenticatedUser) {
    return user;
  }

  @Auth('admin')                            // authenticated, admin only
  @Delete(':id')
  remove(@CurrentUser('id') actorId: string, @Param('id') id: string) {
    return this.users.remove(id, actorId);
  }
}
```

The decision worth defending is throwing a 500 rather than a 401 when `request.user` is absent. A 401 would be a lie: nobody failed to authenticate, the developer forgot a guard. Returning 401 hides that bug behind a plausible-looking response and it reaches production; a 500 with an explicit message fails on the first request in development and names the actual mistake. Decorators that depend on upstream state should assert that state loudly, because they are used far from where it is established.

---

## 2.9 Questions

**1.** MCQ — `emitDecoratorMetadata` emits type metadata for: a) every declaration b) only decorated declarations c) only classes d) only constructors
<details><summary>Answer</summary>**b)** At least one decorator must be present, which is why `@Injectable()` is required on services with dependencies.</details>

**2.** MCQ — Which metadata key holds a constructor's parameter types? a) `design:type` b) `design:params` c) `design:paramtypes` d) `design:returntype`
<details><summary>Answer</summary>**c)** `design:paramtypes`, an array of the actual constructor functions.</details>

**3.** MCQ — An injected `interface Logger` is recorded in `design:paramtypes` as: a) `Logger` b) `Object` c) `undefined` d) `Function`
<details><summary>Answer</summary>**b)** `Object`. Interfaces are erased, so there is no token to resolve.</details>

**4.** MCQ — Decorator functions are applied: a) top to bottom b) bottom to top c) in parallel d) alphabetically
<details><summary>Answer</summary>**b)** Bottom to top, though the expressions are *evaluated* top to bottom.</details>

**5.** MCQ — Which does NOT survive compilation? a) `class` b) `abstract class` c) `enum` d) `type` alias
<details><summary>Answer</summary>**d)** Type aliases are erased, as are interfaces and `const enum`s.</details>

**6.** MCQ — Nest 11 requires which decorator mode? a) TC39 standard b) legacy `experimentalDecorators` c) either d) neither
<details><summary>Answer</summary>**b)** Standard decorators have no parameter decorators, so `@Body()`, `@Param()`, and `@Inject()` could not exist.</details>

**7.** Short — In one sentence, how does Nest know what to inject into a constructor?
<details><summary>Answer</summary>The compiler emits the parameter types as `design:paramtypes` metadata on the decorated class, and the container reads that array to look up each token.</details>

**8.** Short — Why must DTOs be classes rather than interfaces?
<details><summary>Answer</summary>Validation and serialization run at runtime; an interface leaves nothing behind to attach rules to or inspect, so `ValidationPipe` and Swagger both do nothing.</details>

**9.** Short — What does `@Injectable()` actually do?
<details><summary>Answer</summary>Very little directly — it marks the class and, crucially, triggers `emitDecoratorMetadata` so `design:paramtypes` is written. A dependency-free provider works without it.</details>

**10.** Short — Why is an abstract class a better injection contract than an interface?
<details><summary>Answer</summary>It compiles to a real function, so it can serve as a DI token directly, while still expressing an implementation-free contract.</details>

**11.** Short — What is `applyDecorators` for?
<details><summary>Answer</summary>Composing several decorators into one reusable decorator, so a repeated stack (guards, metadata, Swagger annotations) is declared once and applied by name.</details>

**12.** Output — What prints?
```ts
function d(label: string) {
  console.log('eval', label);
  return () => console.log('apply', label);
}
@d('X')
@d('Y')
class C {}
```
<details><summary>Answer</summary>`eval X`, `eval Y`, `apply Y`, `apply X`. Expressions top-down, application bottom-up.</details>

**13.** Output — With `ValidationPipe` registered globally and `CreateUserDto` declared as an **interface** with `email: string`, what does `POST /users` with `{"email": 42}` return?
<details><summary>Answer</summary>`201` — it succeeds. The interface is erased, so the pipe has no class metadata to validate against and passes the body through untouched.</details>

**14.** Output — What does this log?
```ts
@Injectable() class Dep {}
@Injectable() class Svc { constructor(private d: Dep) {} }
console.log(Reflect.getMetadata('design:paramtypes', Svc));
```
<details><summary>Answer</summary>`[ [class Dep] ]` — an array containing the actual `Dep` constructor function.</details>

**15.** Output — Same as above but `@Injectable()` is removed from `Svc`. Now what?
<details><summary>Answer</summary>`undefined`. With no decorator on the class, the compiler emits no metadata at all, and Nest reports that it cannot resolve the dependency at index [0].</details>

**16.** Debug — A service injects `ConfigService` and Nest reports it cannot resolve the dependency at index [0], though `ConfigModule` is imported. The class has no decorators. Cause?
<details><summary>Answer</summary>The missing `@Injectable()`. Without it there is no `design:paramtypes`, so the container cannot see what index [0] even is. Add the decorator — the module wiring was never the problem.</details>

**17.** Debug — Nested objects inside an array property pass validation no matter what you send, while top-level fields validate correctly. What is missing?
<details><summary>Answer</summary>`@Type(() => ItemDto)` alongside `@ValidateNested({ each: true })`. `design:type` records only `Array`, so without an explicit element type `class-transformer` leaves plain objects and validation fails open.</details>

**18.** Debug — A standalone script that imports Nest decorators throws `Reflect.getMetadata is not a function`. Fix?
<details><summary>Answer</summary>Add `import 'reflect-metadata';` once, before anything that uses decorators. Inside a normal Nest bootstrap this is already done internally.</details>

**19.** Debug — After switching `tsconfig.json` to standard decorators, every `@Body()` and `@Param()` fails to compile. Why?
<details><summary>Answer</summary>The TC39 standard decorators proposal does not include parameter decorators. Restore `experimentalDecorators: true` and `emitDecoratorMetadata: true`.</details>

**20.** Debug — Two interceptors are listed on a handler but run in the opposite order to what you expected relative to a guard. What did you misunderstand?
<details><summary>Answer</summary>Decorator order sequences items of the same kind only. Guards always run before interceptors regardless of how the decorators are stacked, because Nest fixes the pipeline order itself.</details>

**21.** Code — Write a `@ClientIp()` parameter decorator returning the caller's IP, preferring `X-Forwarded-For`'s first entry when present.
<details><summary>Answer</summary>

```ts
export const ClientIp = createParamDecorator((_data: unknown, ctx: ExecutionContext): string => {
  const req = ctx.switchToHttp().getRequest<Request>();
  const forwarded = req.headers['x-forwarded-for'];
  if (typeof forwarded === 'string' && forwarded.length > 0) {
    return forwarded.split(',')[0].trim();   // left-most entry is the original client
  }
  return req.ip ?? '';
});
```
Trust `X-Forwarded-For` only behind a proxy you control, and set `app.set('trust proxy', …)` accordingly — otherwise a client can forge it.
</details>

**22.** Code — Create a composed `@PublicEndpoint()` decorator that marks a route public and documents it for Swagger.
<details><summary>Answer</summary>

```ts
export const IS_PUBLIC_KEY = 'isPublic';

export const PublicEndpoint = () =>
  applyDecorators(
    SetMetadata(IS_PUBLIC_KEY, true),
    ApiOperation({ summary: 'Public endpoint — no authentication required' }),
  );
```
A global auth guard then reads `IS_PUBLIC_KEY` via `Reflector` and skips those routes (→ Ch 12).
</details>

**23.** Code — Make `PaymentGateway` injectable as a contract with a swappable implementation, without using a string token.
<details><summary>Answer</summary>

```ts
export abstract class PaymentGateway {
  abstract charge(amountCents: number): Promise<Receipt>;
}

@Injectable()
export class StripeGateway extends PaymentGateway {
  async charge(amountCents: number): Promise<Receipt> { /* … */ }
}

@Module({
  providers: [{ provide: PaymentGateway, useClass: StripeGateway }],
  exports: [PaymentGateway],
})
export class PaymentsModule {}
```
Consumers write `constructor(private readonly gateway: PaymentGateway) {}` and receive the Stripe implementation; a test overrides the same token with a fake.
</details>

**24.** Scenario — A teammate proposes generating DTOs from your OpenAPI spec as TypeScript interfaces, to guarantee the API and types never drift. How do you respond?
<details><summary>Answer</summary>The goal is right and the output format is wrong. Generated interfaces are erased, so you would gain compile-time alignment and lose all runtime validation — the API would accept malformed bodies that type-check perfectly in the editor. Generate classes with `class-validator` decorators instead; several OpenAPI generators support this, and if yours does not, generate the interface and derive a validated class from it. The principle generalises: at a system boundary, a type that exists only at compile time is documentation, not enforcement.</details>

**25.** Scenario — Builds are slow and someone proposes switching the compiler to esbuild, which is dramatically faster. What is your response?
<details><summary>Answer</summary>Reject esbuild specifically: it does not implement `emitDecoratorMetadata`, so `design:paramtypes` is never written and every constructor injection in the application stops resolving. The failure is at startup and total, not subtle. SWC is the right answer to the same problem — it supports legacy decorators and metadata emission behind explicit flags, and the Nest CLI has a first-class SWC builder. The general lesson: any change to the toolchain has to preserve decorator metadata emission, because that is the substrate the container runs on, and "faster TypeScript compiler" is not automatically a drop-in replacement.</details>

---

## Chapter Summary

- Decorators are functions that record metadata; they almost never change behaviour directly, which is why an undeclared controller's `@Get()` silently does nothing.
- `emitDecoratorMetadata` writes `design:paramtypes` only for declarations carrying at least one decorator — the real reason `@Injectable()` is mandatory on services with dependencies.
- Interfaces and type aliases are erased, so they cannot be DI tokens or validation targets; classes and abstract classes survive and can be both.
- Decorator expressions evaluate top-down and apply bottom-up, but pipeline order is fixed by Nest, not by how you stack decorators.
- Nested DTO validation fails **open** without `@Type()`, which is the most dangerous default in the validation stack.
- `applyDecorators` turns a repeated decorator stack into one named policy, removing a whole class of "forgot one" bugs.

---

## Practice Exercises

1. Compile a decorated class with `emitDecoratorMetadata` on and off, and diff the emitted JavaScript.
2. Write a decorator that logs every method call on a class, including arguments and duration, without modifying the methods' source.
3. Build a `@Cached(ttlMs)` method decorator backed by an in-memory `Map`, then explain why this is usually the wrong layer for caching in Nest.
4. Take an endpoint with four stacked decorators and collapse it into one composed decorator.
5. Construct a case where nested validation fails open, prove it with a failing test, then fix it with `@Type()`.

---

## Mini Project: A Decorator Toolkit

Build `src/common/decorators/` as a small, documented toolkit for a real API.

**Requirements:**

- `@CurrentUser()` with a field overload, throwing loudly if no guard populated the request.
- `@Auth(...roles)` composing authentication, role metadata, and Swagger annotations.
- `@ApiPaginatedResponse(Model)` producing a correct generic Swagger schema for paginated results.
- All metadata keys exported as constants from one file, with no string literal duplicated.
- A unit test per decorator that asserts on the emitted metadata rather than on HTTP behaviour.

---

**Next:** [Chapter 3 — Your First Application →](03-first-application.md)
