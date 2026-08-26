# Chapter 24 — GraphQL

## Learning Objectives

- Set up a code-first GraphQL API and explain why code-first suits NestJS
- Write resolvers, field resolvers, and input types with correct nullability
- Diagnose and fix the N+1 problem that GraphQL makes structural
- Make guards, pipes, and filters work across both HTTP and GraphQL contexts
- Bound query complexity so a single request cannot exhaust your database

---

## 24.1 Code-First

```bash
npm i @nestjs/graphql @nestjs/apollo @apollo/server graphql
```

GraphQL in Nest comes in two flavours. **Schema-first** writes SDL and generates types; **code-first** writes decorated classes and generates the SDL. Code-first is the better fit, for the same reason decorators fit the rest of the framework: one definition, no generated file to keep in sync, and the types are the source of truth.

```ts
// src/app.module.ts
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: join(process.cwd(), 'src/schema.gql'),   // generated, committed
  sortSchema: true,                    // stable diffs when the schema changes
  playground: false,
  introspection: process.env.NODE_ENV !== 'production',
  context: ({ req, res }) => ({ req, res }),
});
```

> ⚠️ **Gotcha** — Leave `introspection` on in production and anyone can download your entire schema, including internal fields and admin mutations. It is not a secret in the cryptographic sense, and it is a map. Disable it, and disable the playground with it.

Types are classes with decorators:

```ts
// src/tasks/task.model.ts
@ObjectType()
export class Task {
  @Field(() => ID)
  id: string;

  @Field()
  title: string;

  @Field({ nullable: true })          // `String` in SDL, not `String!`
  description?: string;

  @Field(() => TaskStatus)
  status: TaskStatus;

  @Field(() => User)
  owner: User;                        // resolved separately — see §24.3

  @Field(() => [Comment])
  comments: Comment[];
}

registerEnumType(TaskStatus, { name: 'TaskStatus' });
```

**The type thunk is not optional decoration.** `@Field(() => ID)` needs the arrow because the metadata cannot distinguish `string` from `ID`, and cannot see the element type of an array at all (→ [Ch 2](02-typescript-decorators.md)). Omitting it on an array produces a schema error at startup; omitting it on `ID` silently produces `String`.

Inputs are separate types, and reuse the same validation as REST DTOs (→ [Ch 14](14-pipes-validation.md)):

```ts
@InputType()
export class CreateTaskInput {
  @Field()
  @IsString()
  @Length(1, 120)
  title: string;

  @Field({ nullable: true })
  @IsOptional()
  @MaxLength(2_000)
  description?: string;
}
```

---

## 24.2 Resolvers

```ts
// src/tasks/tasks.resolver.ts
@Resolver(() => Task)
export class TasksResolver {
  constructor(private readonly tasks: TasksService) {}

  @Query(() => [Task], { name: 'tasks' })
  findAll(@Args() args: TaskFilterArgs, @CurrentUser() user: AuthenticatedUser) {
    return this.tasks.findAll(args, user);
  }

  @Query(() => Task, { nullable: true })
  task(@Args('id', { type: () => ID }) id: string) {
    return this.tasks.findOne(id);
  }

  @Mutation(() => Task)
  createTask(@Args('input') input: CreateTaskInput, @CurrentUser() user: AuthenticatedUser) {
    return this.tasks.create(input, user);
  }
}
```

A resolver is a controller for GraphQL — it translates and delegates, and business logic stays in services (→ [Ch 6](06-providers-services.md)). That matters more here than in REST, because the same service must serve both transports.

**Field resolvers** compute a field only when it is requested:

```ts
@ResolveField(() => [Comment])
comments(@Parent() task: Task): Promise<Comment[]> {
  return this.comments.findByTask(task.id);      // runs only if `comments` is selected
}
```

That is GraphQL's central advantage — and it is exactly how the N+1 problem becomes structural.

---

## 24.3 N+1 Is Not Optional Here

In REST you choose what to join. In GraphQL the client chooses, and a field resolver runs **once per parent**:

```graphql
query {
  tasks(limit: 100) {      # 1 query
    title
    owner { name }         # 100 more — one per task
  }
}
```

You cannot fix this by eager-loading, because you do not know what the client will ask for. The answer is **DataLoader**, which batches the calls made within one tick and deduplicates keys:

```bash
npm i dataloader
```

```ts
// src/users/user.loader.ts
@Injectable({ scope: Scope.REQUEST })      // per-request: the cache must not outlive it
export class UserLoader {
  constructor(private readonly users: UsersService) {}

  readonly byId = new DataLoader<string, User>(async (ids) => {
    const users = await this.users.findByIds([...ids]);       // ONE query
    const map = new Map(users.map((u) => [u.id, u]));
    // Must return results in the same order as the requested keys, with a
    // placeholder for misses — DataLoader matches by position, not by id.
    return ids.map((id) => map.get(id) ?? new Error(`User ${id} not found`));
  });
}
```

```ts
@ResolveField(() => User)
owner(@Parent() task: Task): Promise<User> {
  return this.loaders.byId.load(task.ownerId);   // 100 calls → 1 query
}
```

Two properties make this correct and are easy to get wrong.

**The loader must be request-scoped.** Its cache is keyed by id with no TTL, so a singleton loader would serve stale users forever and leak data across requests. Request scope is genuinely justified here — one of the few places it is (→ [Ch 9](09-scopes-lifecycle.md)).

**The batch function must return results positionally.** Returning a filtered array when some ids are missing shifts every subsequent result, so tasks silently get the wrong owner. Map and pad.

> 💼 **Interview** — "Why does GraphQL make N+1 worse?" Because the query shape is chosen by the client at runtime, so you cannot pre-plan joins. The follow-up is how DataLoader helps — batching within a tick — and its limit: it cannot batch across sibling branches that resolve in different ticks.

---

## 24.4 Guards, Pipes, and Filters

Nest's pipeline works with GraphQL, but `ExecutionContext.switchToHttp()` does not — which is the transport leak flagged in [Chapter 12](12-guards.md):

```ts
@Injectable()
export class GqlAuthGuard extends AuthGuard('jwt') {
  // Passport reads the request from here; without this override it gets
  // undefined, because a GraphQL context is not an HTTP context.
  getRequest(context: ExecutionContext) {
    return GqlExecutionContext.create(context).getContext().req;
  }
}
```

A guard used by both transports must branch:

```ts
private getRequest(context: ExecutionContext) {
  if (context.getType<GqlContextType>() === 'graphql') {
    return GqlExecutionContext.create(context).getContext().req;
  }
  return context.switchToHttp().getRequest();
}
```

Validation works unchanged — `ValidationPipe` applies to `@Args()` inputs exactly as to `@Body()`.

**Errors need deliberate handling.** GraphQL returns 200 with an `errors` array, so HTTP status codes stop being the channel. Apollo's default also includes stack traces in development and can leak them if `NODE_ENV` is wrong:

```ts
formatError: (error) => ({
  message: error.message,
  code: error.extensions?.code ?? 'INTERNAL_SERVER_ERROR',
  // Never forward `exception.stacktrace` — Apollo includes it by default
  // outside production, and one misconfigured environment variable exposes it.
}),
```

---

## 24.5 Bounding the Blast Radius

REST endpoints have a fixed cost. A GraphQL query's cost is chosen by the caller, and an unbounded schema is a denial-of-service vector by design:

```graphql
query {
  tasks { comments { task { comments { task { comments { title } } } } } }
}
```

Three limits are not optional in a public API:

**Depth limiting** rejects deeply nested queries:

```ts
import depthLimit from 'graphql-depth-limit';

GraphQLModule.forRoot<ApolloDriverConfig>({
  validationRules: [depthLimit(7)],
});
```

**Complexity limiting** assigns a cost per field and caps the total, which catches wide queries that depth alone misses — a hundred sibling fields at depth two.

**Pagination everywhere.** Any list field without a bounded `limit` lets one query select the whole table (→ [Ch 17](17-typeorm.md)).

For an internal API behind authentication, persisted queries are the stronger option: the client sends a hash of a pre-registered query, so arbitrary queries are impossible.

---

## 24.6 Common Mistakes and Best Practices

**Mistake: leaving introspection enabled in production.** It hands out a complete map of your API.

**Mistake: omitting the type thunk.** Arrays fail at startup; `ID` silently becomes `String`.

**Mistake: no DataLoader.** Field resolvers issue one query per parent, and the client controls how many parents there are.

**Mistake: a singleton DataLoader.** Its unbounded cache serves stale data and leaks across requests.

**Mistake: a batch function returning a filtered array.** Positional mismatch assigns the wrong parent's data.

**Mistake: `switchToHttp()` in a guard used by GraphQL.** There is no HTTP context; use `GqlExecutionContext`.

**Mistake: no depth or complexity limit.** A single query can exhaust the database.

**Best practice: code-first with a committed generated schema.** Diffs on `schema.gql` make breaking changes visible in review.

**Best practice: keep resolvers as thin as controllers.** The same services must serve REST, queues, and GraphQL.

**Best practice: paginate every list field.** An unbounded list is the most common way one query becomes a full table scan.

---

## 24.7 Real-World Example: Batched Resolvers With Authorization

A resolver where every relation is batched, authorization is enforced in the service rather than the resolver, and the loader cannot leak between requests.

```ts
// src/loaders/loaders.module.ts
@Injectable({ scope: Scope.REQUEST })
export class DataLoaders {
  constructor(
    private readonly users: UsersService,
    private readonly comments: CommentsService,
  ) {}

  /**
   * One loader instance per request. The cache has no TTL and is keyed only
   * by id, so a singleton would return a user deleted ten minutes ago — and,
   * worse, hand one request's data to another.
   */
  readonly userById = new DataLoader<string, User>(async (ids) => {
    const rows = await this.users.findByIds([...ids]);
    const byId = new Map(rows.map((u) => [u.id, u]));
    // Positional mapping with an Error placeholder for misses. Returning
    // `rows` directly would misalign every result after the first gap.
    return ids.map((id) => byId.get(id) ?? new Error(`User ${id} not found`));
  });

  readonly commentsByTaskId = new DataLoader<string, Comment[]>(async (taskIds) => {
    const rows = await this.comments.findByTaskIds([...taskIds]);
    const grouped = new Map<string, Comment[]>();
    for (const c of rows) {
      const list = grouped.get(c.taskId) ?? [];
      list.push(c);
      grouped.set(c.taskId, list);
    }
    // A task with no comments must yield [], not undefined — the field is
    // non-nullable in the schema and undefined would fail the whole query.
    return taskIds.map((id) => grouped.get(id) ?? []);
  });
}
```

```ts
// src/tasks/tasks.resolver.ts
@Resolver(() => Task)
@UseGuards(GqlAuthGuard)
export class TasksResolver {
  constructor(
    private readonly tasks: TasksService,
    private readonly loaders: DataLoaders,
  ) {}

  @Query(() => TaskConnection)
  tasks(
    @Args() args: TaskPageArgs,                 // limit is bounded by @Max(100)
    @CurrentUser() actor: AuthenticatedUser,
  ): Promise<TaskConnection> {
    // Authorization lives in the service, so the queue consumer and the REST
    // controller calling the same method get the same rules (→ Ch 22).
    return this.tasks.findPage(args, actor);
  }

  @Mutation(() => Task)
  async updateTask(
    @Args('id', { type: () => ID }) id: string,
    @Args('input') input: UpdateTaskInput,
    @CurrentUser() actor: AuthenticatedUser,
  ): Promise<Task> {
    return this.tasks.update(id, input, actor);
  }

  @ResolveField(() => User)
  owner(@Parent() task: Task): Promise<User> {
    return this.loaders.userById.load(task.ownerId);        // batched
  }

  @ResolveField(() => [Comment])
  comments(@Parent() task: Task): Promise<Comment[]> {
    return this.loaders.commentsByTaskId.load(task.id);     // batched
  }
}
```

```ts
// src/common/gql-auth.guard.ts
@Injectable()
export class GqlAuthGuard extends AuthGuard('jwt') {
  getRequest(context: ExecutionContext) {
    // The transport abstraction leaks here and must be handled explicitly.
    return GqlExecutionContext.create(context).getContext().req;
  }
}
```

```ts
// src/app.module.ts
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
  sortSchema: true,
  playground: false,
  introspection: false,                         // production default
  validationRules: [depthLimit(7)],
  context: ({ req, res }) => ({ req, res }),
  formatError: (error) => ({
    message: error.message,
    code: (error.extensions?.code as string) ?? 'INTERNAL_SERVER_ERROR',
    path: error.path,
    // Deliberately omitting `exception.stacktrace`, which Apollo attaches
    // outside production and which leaks file paths and query fragments.
  }),
});
```

The decision worth defending is keeping authorization in `TasksService` rather than in the resolver, where a `@CheckPolicies` decorator would be more visible. GraphQL makes a single service reachable through many shapes — a top-level query, a field resolver on another type, a mutation's return value — and each of those is a separate entry point that a resolver-level check would have to be repeated on. Missing one is not a broken feature but an unauthorized read, and the field-resolver path is the one people forget because it does not look like an endpoint. Enforcing in the service means every path through the schema is covered by construction, and it keeps the rules identical for the REST controller and the queue consumer calling the same method. The resolver keeps only the authentication guard, which is genuinely transport-level and has to live there because of the `GqlExecutionContext` leak.

---

## 24.8 Questions

**1.** MCQ — Code-first means the schema is: a) hand-written SDL b) generated from decorated classes c) generated from the database d) inferred at runtime
<details><summary>Answer</summary>**b)** Classes are the source of truth and the SDL is generated.</details>

**2.** MCQ — `@Field()` on `comments: Comment[]` without a thunk: a) works b) fails at startup c) returns null d) becomes `String`
<details><summary>Answer</summary>**b)** The element type is erased, so the thunk is required for arrays.</details>

**3.** MCQ — A DataLoader must be: a) global b) request-scoped c) transient d) static
<details><summary>Answer</summary>**b)** Its cache has no TTL and would otherwise leak between requests.</details>

**4.** MCQ — A batch function must return results: a) sorted b) filtered c) in the same order as the keys d) as a map
<details><summary>Answer</summary>**c)** DataLoader matches positionally, so gaps must be padded.</details>

**5.** MCQ — In a GraphQL guard, the request comes from: a) `switchToHttp()` b) `GqlExecutionContext.create(ctx).getContext().req` c) `@Req()` d) `switchToRpc()`
<details><summary>Answer</summary>**b)**</details>

**6.** MCQ — GraphQL errors are returned with HTTP status: a) 400 b) 500 c) 200 d) the mapped status
<details><summary>Answer</summary>**c)** Errors travel in the `errors` array, so status codes are not the channel.</details>

**7.** Short — Why is N+1 structural in GraphQL rather than incidental?
<details><summary>Answer</summary>The client chooses the query shape at runtime, so you cannot pre-plan joins — a field resolver necessarily runs once per parent unless something batches it.</details>

**8.** Short — Why must introspection be disabled in production?
<details><summary>Answer</summary>It publishes the complete schema, including internal fields and admin mutations, which is a map of your API for anyone probing it.</details>

**9.** Short — Why is depth limiting insufficient on its own?
<details><summary>Answer</summary>It does not catch wide queries — a hundred sibling fields at depth two is cheap by depth and expensive to execute. Complexity limits cost per field.</details>

**10.** Short — Why enforce authorization in services rather than resolvers?
<details><summary>Answer</summary>The same service is reachable through top-level queries, field resolvers, and mutation return types; a resolver-level check must be repeated on each, and the field-resolver path is the one that gets missed.</details>

**11.** Short — What does a DataLoader batch, and what can it not batch?
<details><summary>Answer</summary>It batches loads made within the same tick; it cannot batch across sibling branches that resolve in different ticks.</details>

**12.** Output — A query requests 100 tasks and their owners with no DataLoader. How many database queries?
<details><summary>Answer</summary>101 — one for the tasks and one per task in the field resolver.</details>

**13.** Output — A batch function returns `rows` directly when two of ten ids are missing. What do clients see?
<details><summary>Answer</summary>Misaligned data — every result after the first gap is attributed to the wrong parent, silently.</details>

**14.** Output — `@Field(() => ID) id: string` versus `@Field() id: string`. What differs in the SDL?
<details><summary>Answer</summary>The first is `ID!`, the second `String!`. There is no error — just a subtly wrong schema.</details>

**15.** Output — A `commentsByTaskId` loader returns `undefined` for a task with no comments, and the field is `[Comment!]!`. What happens?
<details><summary>Answer</summary>The non-nullable field receives null and the error propagates, failing that part of the query. Return `[]` instead.</details>

**16.** Debug — A guard works over REST and rejects every GraphQL request. Cause?
<details><summary>Answer</summary>It calls `switchToHttp()`, which yields no request in a GraphQL context. Use `GqlExecutionContext`, or branch on `getType()`.</details>

**17.** Debug — One GraphQL query took down the database. What was missing?
<details><summary>Answer</summary>Depth and complexity limits, and probably pagination on a list field — the caller chose a query shape you never anticipated.</details>

**18.** Debug — Users occasionally see another account's profile in a nested field. What do you check first?
<details><summary>Answer</summary>Whether the DataLoader is request-scoped. A singleton loader's cache persists across requests and serves whatever it first loaded for a given id.</details>

**19.** Debug — Stack traces appear in GraphQL error responses in staging. Cause?
<details><summary>Answer</summary>Apollo attaches `exception.stacktrace` outside production. Use `formatError` to return only the fields you intend, rather than relying on `NODE_ENV`.</details>

**20.** Debug — A field returns data for admins that ordinary users should not see, and the resolver has no checks. Where is the gap?
<details><summary>Answer</summary>The field resolver is a separate entry point with no authorization. Enforce in the service, and use field-level visibility for the response shape.</details>

**21.** Code — Write a DataLoader that batches users by id correctly.
<details><summary>Answer</summary>

```ts
readonly userById = new DataLoader<string, User>(async (ids) => {
  const rows = await this.users.findByIds([...ids]);
  const byId = new Map(rows.map((u) => [u.id, u]));
  return ids.map((id) => byId.get(id) ?? new Error(`User ${id} not found`));
});
```
Positional mapping with a placeholder is what keeps results aligned.
</details>

**22.** Code — Write a guard usable by both HTTP and GraphQL.
<details><summary>Answer</summary>

```ts
@Injectable()
export class UniversalAuthGuard extends AuthGuard('jwt') {
  getRequest(context: ExecutionContext) {
    return context.getType<GqlContextType>() === 'graphql'
      ? GqlExecutionContext.create(context).getContext().req
      : context.switchToHttp().getRequest();
  }
}
```
</details>

**23.** Code — Configure GraphQL with production-safe defaults.
<details><summary>Answer</summary>

```ts
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: 'src/schema.gql',
  sortSchema: true,
  playground: false,
  introspection: false,
  validationRules: [depthLimit(7)],
  formatError: (e) => ({ message: e.message, code: e.extensions?.code, path: e.path }),
});
```
</details>

**24.** Scenario — Your team wants to expose GraphQL publicly alongside an existing REST API. What do you require before shipping it?
<details><summary>Answer</summary>Three things REST gave you for free. Cost control, because query shape is now the caller's choice — depth limits, complexity limits, and a bounded `limit` on every list field, or one request can scan the whole database. Batching, since every relation field is an N+1 by default and the client decides how many parents there are. And an authorization audit of every field resolver, because each is an entry point that does not look like one, and the checks must sit in the services rather than the resolvers. If the API is only for your own clients, persisted queries remove most of the first category by making arbitrary queries impossible, and that is usually the better trade than tuning complexity scores.</details>

**25.** Scenario — A frontend team asks for GraphQL because REST requires too many round trips. Is that the right fix?
<details><summary>Answer</summary>Sometimes, and it is worth checking the cheaper options first. If the round trips come from a handful of screens each needing a fixed set of resources, purpose-built REST endpoints returning exactly those shapes solve it with none of GraphQL's operational cost, and they stay cacheable by HTTP infrastructure — which GraphQL over POST is not. GraphQL earns its cost when the *variety* of shapes is genuinely large and changing, typically several client types evolving independently, because then no fixed set of endpoints will keep up. Be explicit that you are trading a fixed-cost API for a variable-cost one: caching, rate limiting, and capacity planning all become harder, and those costs land on whoever operates the service rather than on the team asking for it.</details>

---

## Chapter Summary

- Code-first fits NestJS because decorated classes stay the single source of truth and the SDL is generated — commit it so schema diffs are reviewable.
- Type thunks are mandatory: arrays fail at startup without one and `ID` silently degrades to `String`.
- N+1 is structural in GraphQL because the client picks the query shape, so DataLoader is a requirement rather than an optimisation.
- A DataLoader must be request-scoped and its batch function must return results positionally — a filtered array misattributes data silently.
- Guards must use `GqlExecutionContext` because there is no HTTP context, which is the platform leak Chapter 12 warned about.
- The caller chooses query cost, so depth limits, complexity limits, and bounded list fields are baseline requirements rather than hardening.

---

## Practice Exercises

1. Build a query returning 100 items with a relation, count the database queries, then add a DataLoader and count again.
2. Make a DataLoader a singleton and demonstrate stale data leaking between two requests.
3. Return a filtered array from a batch function with missing keys and observe the misalignment.
4. Apply an HTTP-only guard to a resolver, watch it fail, then fix it with `GqlExecutionContext`.
5. Write a deeply nested recursive query, then add `depthLimit` and confirm the rejection.

---

## Mini Project: A Bounded GraphQL API

Build a GraphQL layer over an existing service layer without duplicating rules.

**Requirements:**

- Code-first schema with a committed `schema.gql` and a CI check that fails on uncommitted drift.
- Every relation field batched, with a test asserting a 50-item query issues a fixed number of queries.
- Guards working over both REST and GraphQL from one implementation.
- Depth and complexity limits, plus a bounded `limit` on every list field, verified by tests that expect rejection.
- Authorization enforced in services, proven by a test that reaches a record through a field resolver and is refused.

---

**Next:** [Chapter 25 — WebSockets and Gateways →](25-websockets.md)
