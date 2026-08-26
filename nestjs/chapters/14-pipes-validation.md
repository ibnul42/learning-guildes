# Chapter 14 — Pipes and Validation

## Learning Objectives

- Implement `PipeTransform` and read `ArgumentMetadata` to write reusable pipes
- Configure `ValidationPipe` so unknown and invalid input is rejected rather than ignored
- Compose DTOs with `PartialType`, `PickType`, and `OmitType` instead of duplicating them
- Diagnose the two ways validation silently fails open
- Decide what belongs in a pipe versus a service

---

## 14.1 A Pipe Transforms One Argument

A pipe receives a single handler argument and returns the value the handler will actually get. It either transforms it or throws.

```ts
// src/common/trim.pipe.ts
import { ArgumentMetadata, Injectable, PipeTransform } from '@nestjs/common';

@Injectable()
export class TrimPipe implements PipeTransform<string, string> {
  transform(value: string, metadata: ArgumentMetadata): string {
    if (typeof value !== 'string') return value;
    return value.trim();
  }
}
```

`ArgumentMetadata` describes what is being processed:

```ts
interface ArgumentMetadata {
  type: 'body' | 'query' | 'param' | 'custom';
  metatype?: Type<unknown>;    // the declared class — undefined for interfaces/primitives
  data?: string;               // the decorator's argument, e.g. 'id' in @Param('id')
}
```

`metatype` is where validation comes from: it is the DTO class recorded by `emitDecoratorMetadata` (→ [Ch 2](02-typescript-decorators.md)). When the parameter is typed as an interface, `metatype` is `Object` and there is nothing to validate against — which is the root cause of §14.5's first failure mode.

Pipes run **after** guards and interceptors and immediately before the handler (→ [Ch 10](10-request-lifecycle.md)). That is the last possible moment, which is correct: validation is the most expensive per-request step and should not run for requests that will be rejected anyway.

Pipes apply per-argument, per-route, per-controller, or globally:

```ts
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {}     // one argument

@UsePipes(new ValidationPipe())                        // whole handler
@Post()
create(@Body() dto: CreateTaskDto) {}
```

---

## 14.2 Built-in Pipes

Nine ship with Nest, and they cover most parameter handling:

| Pipe | Converts / validates |
|---|---|
| `ParseIntPipe` | string → number, else 400 |
| `ParseFloatPipe` | string → float |
| `ParseBoolPipe` | `'true'`/`'false'` → boolean |
| `ParseArrayPipe` | comma-separated → array, with item validation |
| `ParseUUIDPipe` | validates UUID format, with `version` option |
| `ParseEnumPipe` | validates against an enum |
| `DefaultValuePipe` | substitutes a default for `undefined` |
| `ParseFilePipe` | validates uploads (→ [Ch 31](31-file-uploads-streaming.md)) |
| `ValidationPipe` | full DTO validation |

They compose left to right, which matters for defaults:

```ts
// ✅ Default applied first, so `undefined` becomes 1 before parsing
@Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number

// ❌ Reversed: ParseIntPipe receives undefined and throws 400
@Query('page', ParseIntPipe, new DefaultValuePipe(1)) page: number
```

Customising the status code is done through the pipe's options:

```ts
@Param('id', new ParseUUIDPipe({ version: '4', errorHttpStatusCode: HttpStatus.NOT_FOUND }))
id: string
```

Returning 404 rather than 400 for a malformed id is a deliberate choice — it avoids telling a caller whether a given id format is even recognised.

---

## 14.3 `ValidationPipe`

Register it globally, as a provider so it is present in tests too (→ [Ch 8](08-custom-providers.md)):

```ts
{
  provide: APP_PIPE,
  useValue: new ValidationPipe({
    whitelist: true,               // strip properties with no decorator
    forbidNonWhitelisted: true,    // ...and 400 if any were sent
    transform: true,               // return DTO instances, coerce primitives
    transformOptions: { enableImplicitConversion: false },
    stopAtFirstError: false,       // report every invalid field at once
  }),
}
```

Each option earns its place:

**`whitelist`** removes properties the DTO does not declare. Without it, a client can send `{ email, isAdmin: true }` and the extra field reaches your service — and if you ever pass a DTO straight into an ORM `save()`, it reaches the database. This is mass assignment, and `whitelist` is the fix.

**`forbidNonWhitelisted`** turns stripping into an error. Silently dropping a field means a client that misspells `emailAddress` gets a 201 and believes it worked.

**`transform`** returns real DTO instances rather than plain objects, which is what makes `@Type()` and defaults work — and what `ClassSerializerInterceptor` needs (→ [Ch 13](13-interceptors.md)).

**`enableImplicitConversion`** deserves suspicion. It coerces based on the declared type without `@Type()`, which is convenient and surprising: a `string` field receiving `123` becomes `'123'`, and `@IsBoolean()` on a field sent as `'false'` may convert in ways you did not intend. Prefer explicit `@Type()`.

Errors come back as a 400 with per-field messages:

```json
{
  "statusCode": 400,
  "message": ["email must be an email", "age must not be less than 18"],
  "error": "Bad Request"
}
```

In production, `disableErrorMessages: true` hides these. That is a real trade-off — it stops leaking your schema, and it makes client-side debugging much harder. A better middle ground is a custom `exceptionFactory` that returns field names and codes without internal detail.

---

## 14.4 Writing DTOs

`class-validator` decorators attach the rules:

```ts
// src/tasks/dto/create-task.dto.ts
export class CreateTaskDto {
  @IsString()
  @Length(1, 120)
  title: string;

  @IsOptional()
  @IsString()
  @MaxLength(2_000)
  description?: string;

  @IsEnum(Priority)
  priority: Priority = Priority.Normal;

  @IsOptional()
  @Type(() => Date)                 // string → Date
  @IsDate()
  @MinDate(() => new Date())        // a lazy default: evaluated per request
  dueDate?: Date;

  @IsArray()
  @ArrayMaxSize(20)
  @IsString({ each: true })         // validates every element
  tags: string[] = [];
}
```

**Do not duplicate DTOs for updates.** The mapped-type helpers derive them:

```ts
import { PartialType, PickType, OmitType, IntersectionType } from '@nestjs/swagger';

// Every field optional, decorators preserved
export class UpdateTaskDto extends PartialType(CreateTaskDto) {}

// Only these fields
export class TaskTitleDto extends PickType(CreateTaskDto, ['title'] as const) {}

// Everything except these
export class PublicTaskDto extends OmitType(CreateTaskDto, ['ownerId'] as const) {}
```

Import them from `@nestjs/swagger` rather than `@nestjs/mapped-types` when you use Swagger — the swagger versions preserve API documentation metadata as well as validation rules. Using both packages in one project produces DTOs that validate correctly but document inconsistently.

**Validation groups** let one DTO serve several operations:

```ts
export class UserDto {
  @IsEmail({}, { groups: ['create', 'update'] })
  email: string;

  @IsString({ groups: ['create'] })     // required on create only
  @MinLength(12, { groups: ['create'] })
  password: string;
}

@Post()
@UsePipes(new ValidationPipe({ groups: ['create'] }))
create(@Body() dto: UserDto) {}
```

Groups are useful and easy to overuse — beyond two or three, separate DTOs are clearer than a single class whose rules depend on invisible context.

---

## 14.5 The Two Silent Failures

Both fail **open**, accepting invalid data rather than rejecting it. That is the worst direction for a validator to fail.

**Nested objects are not validated without `@Type()`.**

```ts
// ❌ items is validated as an Array and nothing more — every element passes
export class CreateOrderDto {
  @ValidateNested({ each: true })
  items: OrderItemDto[];
}

// ✅
export class CreateOrderDto {
  @ValidateNested({ each: true })
  @Type(() => OrderItemDto)
  items: OrderItemDto[];
}
```

`design:type` records only `Array` — the element type is erased — so without `@Type()` the items stay plain objects, `class-validator` has no class to check them against, and arbitrary nested data is accepted.

**Interfaces validate nothing at all.**

```ts
// ❌ Erased at compile time; metatype is Object, so ValidationPipe skips it
export interface CreateTaskDto { title: string; }

// ✅ A class survives to runtime and carries the decorators
export class CreateTaskDto {
  @IsString() title: string;
}
```

> ⚠️ **Gotcha** — Both failures produce a **201 Created** for input that should have been rejected. There is no warning, no log line, and no error. The only reliable detection is a test that sends deliberately invalid nested data and asserts on the 400.

---

## 14.6 Custom Pipes

Write one when a transformation is genuinely reusable across routes:

```ts
// src/common/parse-date-range.pipe.ts
@Injectable()
export class ParseDateRangePipe implements PipeTransform<string, DateRange> {
  transform(value: string, metadata: ArgumentMetadata): DateRange {
    const [fromRaw, toRaw] = (value ?? '').split('..');
    const from = new Date(fromRaw);
    const to = new Date(toRaw);

    if (Number.isNaN(from.getTime()) || Number.isNaN(to.getTime())) {
      throw new BadRequestException(
        `${metadata.data ?? 'range'} must be "YYYY-MM-DD..YYYY-MM-DD"`,
      );
    }
    if (from > to) throw new BadRequestException('range start must precede end');

    return { from, to };
  }
}
```

The line pipes should not cross is **business rules**. A pipe validates *shape*; a service validates *state*.

```ts
// ❌ In a pipe: needs a database, is invisible from a queue consumer,
//    and duplicates a check the service must do anyway
@Injectable()
export class UniqueEmailPipe implements PipeTransform {
  async transform(dto: CreateUserDto) {
    if (await this.users.existsByEmail(dto.email)) {
      throw new ConflictException('Email in use');
    }
    return dto;
  }
}
```

Uniqueness is a rule about the system's state, not the request's shape. In the service it applies to every caller — HTTP, queue, CLI — and it can be enforced atomically with a database constraint, which the pipe's check-then-insert cannot be.

---

## 14.7 Common Mistakes and Best Practices

**Mistake: DTOs as interfaces.** They are erased, so `metatype` is `Object` and `ValidationPipe` skips them entirely.

**Mistake: `@ValidateNested()` without `@Type()`.** Nested validation fails open and accepts arbitrary objects.

**Mistake: `whitelist` without `forbidNonWhitelisted`.** Unknown fields vanish silently and clients believe malformed requests succeeded.

**Mistake: no `whitelist` at all.** Extra fields reach your service, and a DTO passed to `save()` becomes mass assignment.

**Mistake: `DefaultValuePipe` after `ParseIntPipe`.** Pipes compose left to right, so the parser sees `undefined` and throws.

**Mistake: `enableImplicitConversion` as a convenience.** It coerces from declared types in surprising ways; prefer explicit `@Type()`.

**Mistake: business rules in pipes.** Uniqueness and state checks belong in services, where every entry point benefits and a constraint can enforce them atomically.

**Best practice: register `ValidationPipe` with `APP_PIPE`.** It applies in tests too, so tests exercise the same validation as production.

**Best practice: derive update DTOs with `PartialType`.** Duplicated DTOs drift, and the drift is invisible until something is accepted that should not be.

**Best practice: test the invalid cases.** Validation that fails open produces successful responses, so only a negative test detects it.

---

## 14.8 Real-World Example: A Validated, Nested, Composable DTO Set

An order-creation payload with nested items, a custom cross-field rule, and derived DTOs — closing both fail-open holes deliberately.

```ts
// src/orders/dto/order-item.dto.ts
export class OrderItemDto {
  @IsUUID('4')
  productId: string;

  @Type(() => Number)               // query/form values arrive as strings
  @IsInt()
  @Min(1)
  @Max(999)
  quantity: number;

  @IsOptional()
  @IsString()
  @MaxLength(200)
  note?: string;
}
```

```ts
// src/orders/dto/create-order.dto.ts
import { Type } from 'class-transformer';
import {
  ArrayMaxSize, ArrayMinSize, IsArray, IsIn, IsOptional,
  IsString, MaxLength, ValidateNested, Validate,
} from 'class-validator';

@ValidatorConstraint({ name: 'uniqueProducts', async: false })
export class UniqueProductsConstraint implements ValidatorConstraintInterface {
  validate(items: OrderItemDto[]): boolean {
    if (!Array.isArray(items)) return true;          // let @IsArray report shape errors
    const ids = items.map((i) => i?.productId);
    return new Set(ids).size === ids.length;
  }
  defaultMessage(): string {
    return 'items must not contain duplicate productId values';
  }
}

export class CreateOrderDto {
  @IsArray()
  @ArrayMinSize(1)
  @ArrayMaxSize(50)                 // an unbounded array is a DoS vector
  @ValidateNested({ each: true })
  @Type(() => OrderItemDto)         // WITHOUT this, every item validates trivially
  @Validate(UniqueProductsConstraint)
  items: OrderItemDto[];

  @IsString()
  @MaxLength(280)
  shippingAddress: string;

  @IsOptional()
  @IsIn(['standard', 'express'])
  shippingMethod: 'standard' | 'express' = 'standard';
}
```

```ts
// src/orders/dto/update-order.dto.ts
import { PartialType, OmitType } from '@nestjs/swagger';

// Items are replaced through a dedicated endpoint, not a partial update,
// so they are omitted here rather than made optional.
export class UpdateOrderDto extends PartialType(
  OmitType(CreateOrderDto, ['items'] as const),
) {}
```

```ts
// src/orders/orders.controller.ts
@Controller('orders')
export class OrdersController {
  constructor(private readonly orders: OrdersService) {}

  @Post()
  create(@Body() dto: CreateOrderDto, @CurrentUser('id') userId: string) {
    // Shape is guaranteed here. Stock levels, pricing, and payment are NOT —
    // those are state, and the service owns them.
    return this.orders.create(dto, userId);
  }

  @Get()
  findAll(
    @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
    @Query('limit', new DefaultValuePipe(20), ParseIntPipe) limit: number,
    @Query('range', new DefaultValuePipe('1970-01-01..2100-01-01'), ParseDateRangePipe)
    range: DateRange,
  ) {
    return this.orders.findAll({ page, limit: Math.min(limit, 100), range });
  }
}
```

```ts
// src/orders/orders.controller.spec.ts — the only reliable detection of fail-open
describe('CreateOrderDto validation', () => {
  const pipe = new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true });
  const meta: ArgumentMetadata = { type: 'body', metatype: CreateOrderDto, data: '' };

  it('rejects a nested item with a negative quantity', async () => {
    await expect(
      pipe.transform(
        { items: [{ productId: randomUUID(), quantity: -5 }], shippingAddress: 'x' },
        meta,
      ),
    ).rejects.toThrow(BadRequestException);       // fails without @Type()
  });

  it('rejects unknown properties', async () => {
    await expect(
      pipe.transform(
        { items: [{ productId: randomUUID(), quantity: 1 }], shippingAddress: 'x', isAdmin: true },
        meta,
      ),
    ).rejects.toThrow(BadRequestException);
  });
});
```

The decision worth defending is testing the pipe directly rather than only through end-to-end requests. An e2e test that posts a valid order and asserts 201 passes identically whether or not `@Type()` is present — the failure mode is *accepting* bad input, so only a negative case detects it, and negative cases are exactly what gets skipped when a suite is written against the happy path. Instantiating `ValidationPipe` and calling `transform` directly is fast enough to run every rule as a unit test, keeps the assertion adjacent to the DTO it protects, and makes a removed decorator fail immediately rather than at the next security review. The general principle: when a component's failure mode is silent permissiveness, its tests must assert on rejection, because nothing else will notice.

---

## 14.9 Questions

**1.** MCQ — Pipes run: a) before guards b) after guards, before the handler c) after the handler d) only on errors
<details><summary>Answer</summary>**b)** After guards and interceptors, immediately before the handler.</details>

**2.** MCQ — Which option returns 400 for unknown properties? a) `whitelist` b) `transform` c) `forbidNonWhitelisted` d) `stopAtFirstError`
<details><summary>Answer</summary>**c)** `whitelist` alone strips them silently.</details>

**3.** MCQ — `@ValidateNested()` without `@Type()`: a) throws b) validates correctly c) silently passes everything d) returns 500
<details><summary>Answer</summary>**c)** The element type is erased, so nested objects are never checked.</details>

**4.** MCQ — `ArgumentMetadata.metatype` for a parameter typed as an interface is: a) the interface b) `Object` c) `undefined` d) `Function`
<details><summary>Answer</summary>**b)** Interfaces are erased, so `ValidationPipe` has nothing to validate against.</details>

**5.** MCQ — `@Query('p', ParseIntPipe, new DefaultValuePipe(1))` for a missing `p`: a) returns 1 b) throws 400 c) returns undefined d) returns 0
<details><summary>Answer</summary>**b)** Pipes compose left to right, so `ParseIntPipe` receives `undefined` first. Reverse the order.</details>

**6.** MCQ — `PartialType(CreateDto)` produces a DTO where every field is: a) required b) optional c) readonly d) excluded
<details><summary>Answer</summary>**b)** Optional, with the original validation decorators preserved.</details>

**7.** Short — Why is `whitelist` a security control rather than a convenience?
<details><summary>Answer</summary>Without it, undeclared properties reach your service, and a DTO passed into an ORM `save()` lets a client set any column — mass assignment.</details>

**8.** Short — Why do both nested-validation failures fail open?
<details><summary>Answer</summary>Because the missing metadata means there is nothing to validate against, and `class-validator` treats "no rules" as "valid" — so bad input produces a 201.</details>

**9.** Short — Why prefer explicit `@Type()` to `enableImplicitConversion`?
<details><summary>Answer</summary>Implicit conversion coerces from declared types in ways that are easy to misread, whereas `@Type()` states the intent at the property and behaves the same everywhere.</details>

**10.** Short — Why do uniqueness checks belong in a service rather than a pipe?
<details><summary>Answer</summary>Uniqueness is a fact about system state, so it must apply to every entry point and be enforced atomically with a database constraint — a pipe's check-then-insert has a race window and only covers HTTP.</details>

**11.** Short — Why import mapped types from `@nestjs/swagger` when using Swagger?
<details><summary>Answer</summary>Those versions preserve API documentation metadata alongside validation rules; mixing both packages yields DTOs that validate correctly but document inconsistently.</details>

**12.** Output — With `whitelist: true, forbidNonWhitelisted: false`, `{ "title": "x", "isAdmin": true }` posts to a DTO declaring only `title`. Status and handler value?
<details><summary>Answer</summary>201, and the handler receives `{ title: 'x' }`. The unknown field is stripped with no indication to the client.</details>

**13.** Output — Same request with `forbidNonWhitelisted: true`?
<details><summary>Answer</summary>400, with a message naming `isAdmin` as a property that should not exist.</details>

**14.** Output — `CreateOrderDto` uses `@ValidateNested({ each: true })` but no `@Type()`. What does posting `{ items: [{ quantity: -99 }] }` return?
<details><summary>Answer</summary>201. The items are never validated, so the negative quantity reaches the service.</details>

**15.** Output — A DTO is declared as an interface and `ValidationPipe` is global. What does `{ "email": 12345 }` return?
<details><summary>Answer</summary>201, with `email` as the number `12345`. The interface is erased, so validation is skipped entirely.</details>

**16.** Debug — Validation works locally but is skipped in tests. Cause?
<details><summary>Answer</summary>The pipe is registered with `app.useGlobalPipes()` in `main.ts`, which testing modules never execute. Register it with `APP_PIPE` so it is part of the module graph.</details>

**17.** Debug — `@IsInt()` on a query parameter always fails, even for `?page=2`. Fix?
<details><summary>Answer</summary>Query values are strings. Add `@Type(() => Number)` and ensure `transform: true` so the conversion runs before validation.</details>

**18.** Debug — After adding `@Exclude()` to a DTO, requests started failing validation. Why?
<details><summary>Answer</summary>`@Exclude()` is a `class-transformer` decorator affecting serialization. With `transform: true` it can also strip the property on the way in, so the incoming field disappears before validation. Use serialization groups or a separate response DTO.</details>

**19.** Debug — A client reports that misspelled fields are ignored rather than rejected. Which option is missing?
<details><summary>Answer</summary>`forbidNonWhitelisted: true`. With only `whitelist`, unknown properties are stripped silently.</details>

**20.** Debug — A custom pipe injecting a repository has an `undefined` dependency. How was it applied?
<details><summary>Answer</summary>As an instance — `@UsePipes(new MyPipe())`. Pass the class so the container constructs it with its dependencies.</details>

**21.** Code — Write a DTO for pagination that coerces, bounds, and defaults its values.
<details><summary>Answer</summary>

```ts
export class PageQueryDto {
  @IsOptional() @Type(() => Number) @IsInt() @Min(1)
  page: number = 1;

  @IsOptional() @Type(() => Number) @IsInt() @Min(1) @Max(100)
  limit: number = 20;
}
```
</details>

**22.** Code — Write a pipe that parses `sort=field:asc` into `{ field, direction }`, rejecting unknown fields.
<details><summary>Answer</summary>

```ts
@Injectable()
export class ParseSortPipe implements PipeTransform<string, Sort> {
  constructor(private readonly allowed: readonly string[]) {}

  transform(value: string): Sort {
    const [field, direction = 'asc'] = (value ?? '').split(':');
    if (!this.allowed.includes(field)) {
      throw new BadRequestException(`sort field must be one of: ${this.allowed.join(', ')}`);
    }
    if (direction !== 'asc' && direction !== 'desc') {
      throw new BadRequestException('sort direction must be asc or desc');
    }
    return { field, direction };
  }
}
```
The allowlist is essential — interpolating a client-supplied column into an ORDER BY is an injection vector.
</details>

**23.** Code — Derive an update DTO that makes everything optional except an immutable field, which is removed.
<details><summary>Answer</summary>

```ts
import { PartialType, OmitType } from '@nestjs/swagger';

export class UpdateUserDto extends PartialType(
  OmitType(CreateUserDto, ['email'] as const),
) {}
```
Composing `OmitType` inside `PartialType` removes the field entirely rather than making it optional, so a client cannot submit it at all.
</details>

**24.** Scenario — A teammate proposes `disableErrorMessages: true` in production to avoid leaking the schema. Evaluate it.
<details><summary>Answer</summary>The concern is legitimate but the remedy is blunt: it replaces every validation failure with a bare 400, so a client integrating against your API cannot tell which field was wrong, and your own support team loses the fastest diagnostic they have. The schema exposure it prevents is also modest — anyone can discover field names by probing, and if a field name is itself sensitive that is a naming problem. The better answer is a custom `exceptionFactory` returning a stable machine-readable shape: field path plus an error code, without constraint internals or received values. That keeps clients able to render form errors, keeps logs useful, and avoids echoing back user input, which is the part that actually risks leaking data.</details>

**25.** Scenario — Your API accepts a deeply nested configuration object whose shape varies by a `type` discriminator. How do you validate it?
<details><summary>Answer</summary>Use discriminated validation rather than one permissive DTO. `class-transformer`'s `@Type()` supports a discriminator so each `type` value maps to its own class, which keeps every variant's rules explicit and keeps `whitelist` meaningful — a single union-shaped DTO would have to permit the union of all fields, so a payload could mix keys from two variants and still pass. Bound the nesting depth explicitly, because recursive DTOs with unbounded depth are a denial-of-service vector that validation itself will happily walk. If the shapes are genuinely open-ended — user-supplied JSON schemas, plugin configuration — stop trying to express it in DTOs and validate against a real schema validator at the service boundary, then store it as an opaque value; forcing an open shape into `class-validator` produces rules nobody can read and gaps nobody can see.</details>

---

## Chapter Summary

- A pipe transforms or rejects one argument, and `ArgumentMetadata.metatype` is the DTO class that makes validation possible — which is why interfaces validate nothing.
- Pipes run last in the inbound pipeline, so the most expensive check happens only for requests that passed authorization.
- `whitelist` is a security control against mass assignment; without `forbidNonWhitelisted` it strips silently and clients never learn their request was malformed.
- Validation fails **open** in two ways — missing `@Type()` on nested arrays and DTOs declared as interfaces — and both produce a 201 for input that should have been rejected.
- Built-in pipes compose left to right, which is why `DefaultValuePipe` must precede `ParseIntPipe`.
- Pipes validate shape; services validate state, so uniqueness and business rules belong where every entry point and a database constraint can enforce them.

---

## Practice Exercises

1. Build a DTO with a nested array, omit `@Type()`, and prove invalid nested data is accepted. Add it and prove the 400.
2. Compare the responses for the same malformed request under all four combinations of `whitelist` and `forbidNonWhitelisted`.
3. Write a custom `exceptionFactory` producing `{ field, code }` pairs and use it instead of `disableErrorMessages`.
4. Derive three DTOs from one base with `PartialType`, `PickType`, and `OmitType`, and inspect the generated Swagger schema for each.
5. Instantiate `ValidationPipe` directly in a unit test and assert on rejection for six invalid payloads.

---

## Mini Project: A Validation Test Harness

Build a harness that proves an API rejects what it should.

**Requirements:**

- Discovers every DTO in the project and generates invalid variants for each decorated property.
- Asserts each variant produces a 400 rather than a 2xx.
- Detects the fail-open cases specifically: nested arrays lacking `@Type()`, and DTOs declared as interfaces.
- Reports any DTO with no validation decorators at all as a failure.
- Runs in CI in under ten seconds by testing pipes directly rather than over HTTP.

---

**Next:** [Chapter 15 — Exception Filters →](15-exception-filters.md)
