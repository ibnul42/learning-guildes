# Chapter 22 — Authorization

## Learning Objectives

- Distinguish RBAC from ABAC and choose between them from real requirements
- Implement role checks that a route can neither bypass nor silently relax
- Model ownership and conditional permissions with CASL
- Enforce authorization at the data layer so non-HTTP callers cannot bypass it
- Recognise when a permission check is in the wrong layer entirely

---

## 22.1 Three Models, Increasing Cost

Authorization answers "may this principal perform this action on this resource". Three models cover almost everything, and each is more expressive and more expensive than the last.

**RBAC — role-based.** The subject has roles; roles grant permissions. `admin` may delete tasks.

**ABAC — attribute-based.** The decision uses attributes of the subject, the resource, the action, and the environment. A user may edit a task *if they own it* and *it is not archived* and *it is within the edit window*.

**ReBAC — relationship-based.** Permission derives from a graph: you may view a document because you are a member of a team that owns the folder it lives in. This is what Google Docs and GitHub do, and it needs a dedicated system (Zanzibar-style) rather than an in-process check.

| | RBAC | ABAC | ReBAC |
|---|---|---|---|
| Decision inputs | roles | subject + resource + context | relationship graph |
| Needs the resource loaded | no | yes | yes, plus traversal |
| Guard-friendly | yes | partly | no |
| Right for | coarse admin boundaries | ownership, tenancy, state | sharing, nesting, delegation |

Most applications need RBAC for coarse boundaries and ABAC for everything real. The mistake is trying to express ownership as a role — `task-42-owner` is not a role, and a system that invents one has chosen the wrong model.

---

## 22.2 RBAC in a Guard

Roles are decidable from the request alone, which makes them a good fit for a guard (→ [Ch 12](12-guards.md)):

```ts
export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);

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
    if (!user) throw new UnauthorizedException();     // fail closed (→ Ch 12)

    if (!required.some((r) => user.roles.includes(r))) {
      throw new ForbiddenException(`Requires one of: ${required.join(', ')}`);
    }
    return true;
  }
}
```

**Roles should not be checked directly in business logic.** `if (user.role === 'admin')` scattered through services means adding a role requires finding every occurrence. Check *permissions* instead, and derive them from roles in one place:

```ts
// ❌ The role and the capability are conflated; adding `auditor` touches
//    every file that special-cased `admin`.
if (user.role === 'admin') return this.tasks.findAll();

// ✅ One mapping, many consumers
const ROLE_PERMISSIONS: Record<Role, Permission[]> = {
  admin:   ['task:read:any', 'task:write:any', 'user:manage'],
  auditor: ['task:read:any'],
  member:  ['task:read:own', 'task:write:own'],
};

if (user.can('task:read:any')) return this.tasks.findAll();
```

---

## 22.3 Ownership Is Not a Guard's Job

The moment a decision depends on the resource, a guard becomes the wrong place (→ [Ch 12](12-guards.md)). It must load the record to decide, cannot pass it to the handler, and turns a missing record into a 403 rather than a 404 — leaking whether an id exists.

Three placements, from weakest to strongest:

**Post-load check in the service.** Correct, and it reveals existence:

```ts
const task = await this.repo.findById(id);
if (!task) throw new NotFoundException();
if (task.ownerId !== actor.id) throw new ForbiddenException();
```

**Scoped query.** The row is simply not returned, so existence is not revealed and the check cannot be forgotten at a call site:

```ts
const task = await this.repo.findOne({ where: { id, ownerId: actor.id } });
if (!task) throw new NotFoundException();     // 404 for both cases, deliberately
```

**Row-level security in the database.** The strongest, because it holds even for a query written by a developer who forgot — including from a migration, a script, or a background job.

The middle option is the right default. Returning 404 rather than 403 for someone else's resource is a deliberate choice: a 403 confirms the id is real, which is a useful oracle for an attacker enumerating identifiers.

---

## 22.4 CASL

CASL expresses conditional permissions as data rather than as scattered `if` statements:

```bash
npm i @casl/ability
```

```ts
// src/auth/ability.factory.ts
import { AbilityBuilder, createMongoAbility, MongoAbility, ExtractSubjectType } from '@casl/ability';

export type Action = 'manage' | 'create' | 'read' | 'update' | 'delete';
export type Subject = 'Task' | 'User' | 'all';
export type AppAbility = MongoAbility<[Action, Subject | Task | User]>;

@Injectable()
export class AbilityFactory {
  createForUser(user: AuthenticatedUser): AppAbility {
    const { can, cannot, build } = new AbilityBuilder<AppAbility>(createMongoAbility);

    if (user.roles.includes('admin')) {
      can('manage', 'all');                       // 'manage' means every action
    } else {
      can('read', 'Task', { ownerId: user.id });  // conditions are the ABAC part
      can('update', 'Task', { ownerId: user.id, status: { $ne: 'archived' } });
      can('create', 'Task');
      cannot('delete', 'Task', { status: 'locked' })
        .because('Locked tasks cannot be deleted');
    }

    return build({
      detectSubjectType: (item) => item.constructor as ExtractSubjectType<Subject>,
    });
  }
}
```

Checks then read as the requirement does:

```ts
const ability = this.abilities.createForUser(actor);

if (ability.cannot('update', task)) {
  throw new ForbiddenException();
}
```

CASL's real advantage is that the same rules can be **pushed into the query**, so a list endpoint returns only permitted rows without a hand-written filter:

```ts
import { accessibleBy } from '@casl/prisma';

const tasks = await this.prisma.task.findMany({
  where: { AND: [accessibleBy(ability).Task, { status: 'open' }] },
});
```

That is the property that makes it worth the dependency. A rule defined once governs both "may I touch this record" and "which records may I see", and those two cannot drift apart.

> ⚠️ **Gotcha** — `can('read', 'Task', { ownerId })` on a **subject type** (the string `'Task'`) cannot evaluate conditions, so `ability.can('read', 'Task')` returns `true` — it means "may read *some* task". Conditions are only evaluated against an *instance*. Checking the string where you meant the instance is a silent authorization bypass, and it type-checks.

---

## 22.5 Defence in Depth

A permission check in the HTTP pipeline protects HTTP. It does nothing for a queue consumer, a cron job, a CLI command, or a GraphQL resolver that calls the same service.

Enforcement belongs at more than one layer, with each layer covering the one above:

| Layer | Catches | Misses |
|---|---|---|
| Guard | unauthenticated and wrong-role callers, cheaply | anything not going through HTTP |
| Service | every caller, including background jobs | a query written directly against the ORM |
| Data layer | every query, including scripts and migrations | nothing |

The guard is a fast rejection; the data layer is the guarantee. A system with only the guard is one background worker away from a leak, and that worker will be written by someone who does not know the rule exists.

Field-level authorization is the case people forget. A user may read a task and may not read its `internalNotes`. Serialization groups handle it declaratively (→ [Ch 13](13-interceptors.md)):

```ts
export class TaskDto {
  @Expose() id: string;
  @Expose() title: string;

  @Expose({ groups: ['admin'] })      // omitted for everyone else
  internalNotes: string;
}

@SerializeOptions({ groups: ['admin'] })
@Get(':id')
findOne() { … }
```

---

## 22.6 Common Mistakes and Best Practices

**Mistake: modelling ownership as a role.** Roles are a fixed small set; per-resource relationships are not.

**Mistake: `if (user.role === 'admin')` in services.** Adding a role means finding every occurrence, and one will be missed.

**Mistake: ownership checks in guards.** They double-fetch the record and turn a missing one into a 403.

**Mistake: returning 403 for another user's resource.** It confirms the id exists, which is an enumeration oracle.

**Mistake: `ability.can('update', 'Task')` with the subject type.** Conditions are not evaluated, so it always passes.

**Mistake: authorization only in the HTTP pipeline.** Queue consumers and jobs call the same services and bypass it entirely.

**Mistake: filtering permitted rows by hand alongside CASL rules.** The two definitions drift, and the query is the one that leaks.

**Best practice: check permissions, not roles.** One role-to-permission mapping keeps every consumer stable as roles change.

**Best practice: enforce ownership as a query predicate.** A row that is not returned cannot be leaked by a forgotten check.

**Best practice: derive the list filter from the same rules as the instance check.** It is the only way to guarantee they agree.

---

## 22.7 Real-World Example: CASL Rules Driving Both Checks and Queries

One rule set governing instance checks, list filtering, and field visibility — so the three cannot disagree.

```ts
// src/auth/ability.factory.ts
@Injectable()
export class AbilityFactory {
  createForUser(user: AuthenticatedUser): AppAbility {
    const { can, cannot, build } = new AbilityBuilder<AppAbility>(createMongoAbility);

    if (user.roles.includes(Role.Admin)) {
      can('manage', 'all');
    } else if (user.roles.includes(Role.Auditor)) {
      // Read everything in their tenant, change nothing.
      can('read', 'Task', { tenantId: user.tenantId });
    } else {
      can('read', 'Task', { tenantId: user.tenantId, ownerId: user.id });
      can('create', 'Task');
      can('update', 'Task', {
        ownerId: user.id,
        status: { $nin: ['archived', 'locked'] },
      });
      can('delete', 'Task', { ownerId: user.id, status: 'draft' });
    }

    // Applied after the role branches, so it overrides them — including for
    // admins. A suspended account keeps its roles and loses its capabilities.
    if (user.suspended) {
      cannot('create', 'all').because('Account suspended');
      cannot('update', 'all').because('Account suspended');
      cannot('delete', 'all').because('Account suspended');
    }

    return build({
      detectSubjectType: (item) => item.constructor as ExtractSubjectType<Subject>,
    });
  }
}
```

```ts
// src/auth/policies.guard.ts
export const CHECK_POLICIES_KEY = 'check_policies';
export type PolicyHandler = (ability: AppAbility) => boolean;
export const CheckPolicies = (...handlers: PolicyHandler[]) =>
  SetMetadata(CHECK_POLICIES_KEY, handlers);

/**
 * Handles only the checks decidable without the resource — "may this user
 * create tasks at all". Anything conditional on a specific record is left
 * to the service, which has the record.
 */
@Injectable()
export class PoliciesGuard implements CanActivate {
  constructor(
    private readonly reflector: Reflector,
    private readonly abilities: AbilityFactory,
  ) {}

  canActivate(context: ExecutionContext): boolean {
    const handlers =
      this.reflector.get<PolicyHandler[]>(CHECK_POLICIES_KEY, context.getHandler()) ?? [];
    if (!handlers.length) return true;

    const { user } = context.switchToHttp().getRequest();
    if (!user) throw new UnauthorizedException();

    const ability = this.abilities.createForUser(user);
    if (!handlers.every((handler) => handler(ability))) {
      throw new ForbiddenException();
    }
    return true;
  }
}
```

```ts
// src/tasks/tasks.service.ts
@Injectable()
export class TasksService {
  constructor(
    private readonly repo: TaskRepository,
    private readonly abilities: AbilityFactory,
  ) {}

  /**
   * The list filter comes from the SAME rules as the instance check, so a
   * change to the ability definition updates both. A hand-written WHERE
   * clause here would be a second definition, and the one that drifts.
   */
  async findAll(actor: AuthenticatedUser, filter: TaskFilter): Promise<Task[]> {
    const ability = this.abilities.createForUser(actor);
    return this.repo.findAccessible(ability, filter);
  }

  async update(id: string, dto: UpdateTaskDto, actor: AuthenticatedUser): Promise<Task> {
    const ability = this.abilities.createForUser(actor);

    // Scoped read: a task outside the actor's readable set is a 404, not a
    // 403, so the endpoint is not an existence oracle.
    const task = await this.repo.findAccessible(ability, { id }).then((r) => r[0]);
    if (!task) throw new NotFoundException(`Task ${id} not found`);

    // Readable does not imply writable — the auditor role proves the two
    // differ, so the write permission is checked separately against the
    // loaded instance where conditions can actually be evaluated.
    ForbiddenError.from(ability).throwUnlessCan('update', task);

    return this.repo.save(Object.assign(task, dto));
  }
}
```

```ts
// src/tasks/task.repository.ts
@Injectable()
export class TaskRepository {
  constructor(@InjectRepository(Task) private readonly repo: Repository<Task>) {}

  /** Rules are translated into SQL, so unreadable rows never leave the database. */
  async findAccessible(ability: AppAbility, filter: TaskFilter): Promise<Task[]> {
    return this.repo.find({
      where: { ...accessibleBy(ability, 'read').ofType(Task), ...filter },
      take: filter.limit ?? 20,
    });
  }
}
```

```ts
// src/tasks/tasks.controller.ts
@Controller('tasks')
@UseGuards(JwtAuthGuard, PoliciesGuard)
export class TasksController {
  @CheckPolicies((ability) => ability.can('create', 'Task'))   // no record needed
  @Post()
  create(@Body() dto: CreateTaskDto, @CurrentUser() actor: AuthenticatedUser) {
    return this.tasks.create(dto, actor);
  }

  // No policy decorator: the rule depends on the record, so the service owns it.
  @Patch(':id')
  update(
    @Param('id', ParseUUIDPipe) id: string,
    @Body() dto: UpdateTaskDto,
    @CurrentUser() actor: AuthenticatedUser,
  ) {
    return this.tasks.update(id, dto, actor);
  }
}
```

The decision worth defending is splitting the check across two layers rather than putting all of it in `PoliciesGuard`. A guard that handled `update` would have to load the task, which reintroduces the double-fetch and the 403-versus-404 leak from §22.3 — and the loaded record still could not reach the handler. Keeping resource-independent checks in the guard preserves the cheap early rejection for the cases where it works, while resource-dependent checks live where the resource already is. The split looks inconsistent on the controller, since some routes carry `@CheckPolicies` and some do not, and that inconsistency is meaningful: it marks exactly which decisions need the record. The alternative — pushing everything into the service for uniformity — loses the ability to reject an unauthorized `create` before any work happens, and the alternative of pushing everything into guards trades a security property for visual symmetry.

---

## 22.8 Questions

**1.** MCQ — Which model uses attributes of subject, resource, and context? a) RBAC b) ABAC c) ReBAC d) ACL
<details><summary>Answer</summary>**b)** ABAC. RBAC uses roles only; ReBAC uses a relationship graph.</details>

**2.** MCQ — `ability.can('update', 'Task')` with the string subject: a) evaluates conditions b) ignores conditions c) throws d) returns false
<details><summary>Answer</summary>**b)** Conditions apply only to instances, so this asks "may update *some* task" and usually returns `true`.</details>

**3.** MCQ — Returning 403 rather than 404 for another user's record leaks: a) nothing b) the record c) that the id exists d) the owner
<details><summary>Answer</summary>**c)** An enumeration oracle.</details>

**4.** MCQ — Which layer catches a queue consumer bypassing authorization? a) guard b) interceptor c) service or data layer d) middleware
<details><summary>Answer</summary>**c)** Guards only cover HTTP.</details>

**5.** MCQ — In CASL, `can('manage', 'all')` means: a) read everything b) every action on every subject c) admin role d) manage users
<details><summary>Answer</summary>**b)** `manage` is the wildcard action and `all` the wildcard subject.</details>

**6.** MCQ — Ownership is best expressed as: a) a role b) a query predicate c) a middleware check d) a header
<details><summary>Answer</summary>**b)** A row that is not returned cannot be leaked by a forgotten check.</details>

**7.** Short — Why is `task-42-owner` a sign of the wrong model?
<details><summary>Answer</summary>Roles are a small fixed set describing what kind of user someone is; per-resource relationships are unbounded and belong in conditions or a relationship graph.</details>

**8.** Short — Why check permissions rather than roles in business logic?
<details><summary>Answer</summary>A role-to-permission mapping in one place lets you add or split roles without touching every consumer; scattered role comparisons must all be found and updated, and one will be missed.</details>

**9.** Short — What makes CASL worth its dependency?
<details><summary>Answer</summary>The same rule definition drives instance checks *and* the list query filter, so "may I touch this" and "which may I see" cannot drift apart.</details>

**10.** Short — Why is readable not the same as writable?
<details><summary>Answer</summary>An auditor role can read every record in a tenant and modify none, so a single "can access" check would grant writes it should not.</details>

**11.** Short — Where does field-level authorization belong?
<details><summary>Answer</summary>In serialization, via groups on the DTO, so a field is omitted declaratively rather than deleted by hand at each call site.</details>

**12.** Output — A non-owner requests `GET /tasks/:id` for an existing task, and the query is scoped by owner. What status?
<details><summary>Answer</summary>404 — the scoped query returns nothing, so the endpoint cannot confirm the id exists.</details>

**13.** Output — A suspended admin has `can('manage', 'all')` followed by `cannot('update', 'all')`. Can they update?
<details><summary>Answer</summary>No. Later rules override earlier ones in CASL, so the suspension takes effect despite the admin grant.</details>

**14.** Output — A guard checks `ability.can('delete', 'Task')` and the rule is `can('delete', 'Task', { ownerId: user.id })`. What happens for a non-owner?
<details><summary>Answer</summary>It passes — the subject-type check ignores conditions, so a non-owner reaches the handler and the real check must happen against the instance.</details>

**15.** Output — A background job calls `tasksService.deleteAll()` with authorization only in a guard. What is enforced?
<details><summary>Answer</summary>Nothing. The job never passes through the HTTP pipeline.</details>

**16.** Debug — Adding an `auditor` role required changes in fourteen files. What is the underlying problem?
<details><summary>Answer</summary>Business logic compares roles directly instead of checking permissions. Introduce a role-to-permission mapping and check capabilities.</details>

**17.** Debug — A list endpoint returns records the detail endpoint refuses. Cause?
<details><summary>Answer</summary>The list filter is hand-written and the detail check uses the ability rules; the two definitions have drifted. Derive the filter from the same rules.</details>

**18.** Debug — Users report 403s on tasks they own, but only recently-archived ones. What rule explains it?
<details><summary>Answer</summary>`can('update', 'Task', { ownerId, status: { $nin: ['archived'] } })` — ownership holds but the status condition fails, which is the rule working as written.</details>

**19.** Debug — A GraphQL resolver bypasses authorization that works over REST. Why?
<details><summary>Answer</summary>The check lives in an HTTP guard using `switchToHttp()`, which does not apply to the GraphQL context. Move enforcement into the service, and make the guard transport-aware.</details>

**20.** Debug — An admin script leaked cross-tenant data. Which layer was missing?
<details><summary>Answer</summary>The data layer. Guards and services do not constrain a query written directly against the ORM; row-level security or a mandatory tenant predicate in the repository would have.</details>

**21.** Code — Write a role-to-permission mapping and a capability check.
<details><summary>Answer</summary>

```ts
const ROLE_PERMISSIONS: Record<Role, readonly Permission[]> = {
  admin:   ['task:read:any', 'task:write:any'],
  auditor: ['task:read:any'],
  member:  ['task:read:own', 'task:write:own'],
};

export function can(user: AuthenticatedUser, permission: Permission): boolean {
  return user.roles.some((role) => ROLE_PERMISSIONS[role]?.includes(permission));
}
```
</details>

**22.** Code — Write a CASL ability where members edit only their own unarchived tasks.
<details><summary>Answer</summary>

```ts
const { can, build } = new AbilityBuilder<AppAbility>(createMongoAbility);
can('read', 'Task', { ownerId: user.id });
can('update', 'Task', { ownerId: user.id, status: { $ne: 'archived' } });
return build({ detectSubjectType: (i) => i.constructor as ExtractSubjectType<Subject> });
```
Check it against a loaded task — `ability.can('update', task)` — never against the string `'Task'`.
</details>

**23.** Code — Enforce ownership as a query predicate returning 404 for non-owners.
<details><summary>Answer</summary>

```ts
async findOwned(id: string, ownerId: string): Promise<Task> {
  const task = await this.repo.findOne({ where: { id, ownerId } });
  if (!task) throw new NotFoundException(`Task ${id} not found`);
  return task;                       // same response whether missing or not yours
}
```
</details>

**24.** Scenario — A teammate proposes putting every authorization check in guards "so it is all in one place". Evaluate it.
<details><summary>Answer</summary>The instinct toward one place is right and guards are the wrong place for it. Any rule depending on the resource forces the guard to load it, which double-fetches, prevents handing the record to the handler, and converts a missing record into a 403 that confirms the id exists. It also protects only HTTP, so a queue consumer or scheduled job calling the same service enforces nothing. The single place worth consolidating in is the rule *definition* — one ability factory — while enforcement happens wherever the necessary information exists: guards for resource-independent checks, services for instance checks, and the data layer for the guarantee. Centralise the policy, distribute the enforcement.</details>

**25.** Scenario — Your product adds team-based sharing: users can share tasks with teams, and team members inherit access with a role that varies per team. Does CASL still work?
<details><summary>Answer</summary>It works up to a point and this is the boundary where it stops. CASL evaluates conditions against attributes it can express as a query, so a single hop — "tasks whose `teamId` is in the user's team list" — is fine if you load the memberships into the ability. It degrades once access derives from traversal: a task in a folder owned by a team whose parent team you belong to cannot be expressed as a `WHERE` clause without recursive joins, and the ability object would have to embed the whole graph. That is the signal to move to a ReBAC system built on the Zanzibar model, which stores relationship tuples and answers reachability questions directly. Do not attempt it by denormalising every relationship into user attributes — that works until the graph changes and then leaves stale permissions no one can trace.</details>

---

## Chapter Summary

- RBAC decides from roles alone and fits a guard; ABAC needs the resource; ReBAC needs a graph and a dedicated system — modelling ownership as a role is the sign you picked the wrong one.
- Check permissions rather than roles in business logic, so adding or splitting a role touches one mapping instead of every consumer.
- Ownership belongs in the query, not a guard: a scoped read returns 404 for both missing and forbidden, which avoids confirming that an id exists.
- CASL earns its dependency by driving instance checks and list filters from one rule set, which is the only way to keep them from drifting.
- Checking a CASL ability against a subject *type* skips conditions entirely and type-checks — a silent bypass that must be checked against instances.
- Authorization in the HTTP pipeline protects HTTP only; the guard is the fast rejection and the data layer is the guarantee.

---

## Practice Exercises

1. Write `ability.can('update', 'Task')` and `ability.can('update', task)` for a non-owner and compare the results.
2. Implement ownership as a post-load check and as a query predicate, then compare the status codes a non-owner receives.
3. Add a new role to a codebase that compares roles directly, and count the files you had to change.
4. Derive a list filter from CASL rules, then deliberately change one rule and confirm both the list and the detail check move together.
5. Write a background job that calls a service directly and demonstrate that guard-based authorization does not apply.

---

## Mini Project: A Policy Engine

Build authorization that cannot be bypassed by a caller that skips HTTP.

**Requirements:**

- One ability factory producing rules from a principal, covering roles, ownership, tenancy, and suspension.
- A policies guard for resource-independent checks only, with instance checks in services.
- List endpoints filtered by rules translated into the query, never by a hand-written predicate.
- Field-level visibility through serialization groups, verified by a test that an ordinary user cannot see an admin-only field.
- Tests proving a queue consumer calling the same service is subject to the same rules.

---

**Next:** [Chapter 23 — Securing a NestJS API →](23-security.md)
