# Chapter 20 — Caching

## Learning Objectives

- Register `CacheModule` with an in-memory or Redis store and choose between them
- Design cache keys that cannot leak data between users or tenants
- Pick an invalidation strategy and accept its failure mode explicitly
- Prevent a cache stampede from turning a cache miss into an outage
- Decide what is worth caching by measuring rather than assuming

---

## 20.1 Registering the Cache

```bash
npm i @nestjs/cache-manager cache-manager
npm i cache-manager-redis-yet   # Redis store
```

NestJS 11 uses `cache-manager` v6, which is built on **Keyv** — a change from earlier versions, so store packages and configuration from a Nest 10 tutorial will not work unmodified.

```ts
// src/cache/cache.module.ts
import { CacheModule } from '@nestjs/cache-manager';
import { createKeyv } from '@keyv/redis';

@Module({
  imports: [
    CacheModule.registerAsync({
      isGlobal: true,
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => {
        const url = config.get<string>('REDIS_URL');
        return {
          ttl: 60_000,                      // milliseconds in v6 — seconds in v5
          stores: url ? [createKeyv(url)] : [],   // no URL → in-memory only
        };
      },
    }),
  ],
})
export class AppCacheModule {}
```

> ⚠️ **Gotcha** — TTL is **milliseconds** in `cache-manager` v6 and was **seconds** in v5. Copying `ttl: 60` from an older example gives you a 60-millisecond cache, which behaves like no cache at all and is easy to miss because nothing breaks — it is just slow.

**In-memory versus Redis** is not a matter of scale alone:

| | In-memory | Redis |
|---|---|---|
| Shared across instances | No | Yes |
| Survives restart | No | Yes |
| Latency | ~0 | a network round trip |
| Invalidation across instances | impossible | straightforward |
| Memory pressure | your process | separate |

The decisive property is the second row. With several replicas and an in-memory cache, invalidating on instance A leaves stale data on B and C — so any data that changes needs a shared store. In-memory is right for values that are immutable or where staleness is genuinely harmless.

---

## 20.2 Using the Cache Directly

```ts
@Injectable()
export class TasksService {
  constructor(
    @Inject(CACHE_MANAGER) private readonly cache: Cache,
    private readonly repo: TaskRepository,
  ) {}

  async getStats(ownerId: string): Promise<Stats> {
    const key = `stats:${ownerId}`;

    const cached = await this.cache.get<Stats>(key);
    if (cached) return cached;

    const stats = await this.repo.computeStats(ownerId);   // expensive
    await this.cache.set(key, stats, 300_000);             // 5 minutes
    return stats;
  }
}
```

That is the read-through pattern, and it has three problems worth knowing before you copy it.

**`if (cached)` is wrong for falsy values.** A cached `0`, `false`, or empty string is a hit that this code treats as a miss, so the expensive path runs every time. Use `undefined` as the sentinel:

```ts
const cached = await this.cache.get<Stats>(key);
if (cached !== undefined) return cached;
```

**A cache failure becomes a request failure.** If Redis is unreachable, `get` rejects and the endpoint returns 500 — the cache has made the system *less* available than no cache at all. Wrap it:

```ts
private async safeGet<T>(key: string): Promise<T | undefined> {
  try {
    return await this.cache.get<T>(key);
  } catch (err) {
    this.logger.warn(`Cache read failed for ${key}`, err);
    return undefined;                      // degrade to the source of truth
  }
}
```

**It stampedes.** §20.4.

The built-in interceptor handles the simple case declaratively:

```ts
@UseInterceptors(CacheInterceptor)
@CacheKey('all-tasks')
@CacheTTL(30_000)
@Get()
findAll() { … }
```

It caches only `GET` requests and keys by URL — which is exactly why it must not be applied globally to an authenticated API (→ §20.3).

---

## 20.3 Keys Are a Security Boundary

The most damaging cache bug is not staleness. It is serving one user's data to another, and it comes from a key that omits identity.

```ts
// ❌ Two users requesting /tasks share one entry
const key = request.originalUrl;

// ✅ Every dimension the response varies by, in the key
const key = [
  'v1',                          // bump to invalidate everything on a shape change
  request.user.tenantId,
  request.user.id,
  request.originalUrl,
  request.headers['accept-language'] ?? 'en',
].join(':');
```

A key must include **everything the response depends on**. In practice that means the principal, the tenant, the API version, the serialization group if roles change the shape, and any header that alters the body.

This is why `CacheInterceptor` registered globally is dangerous: it keys by URL alone, so any authenticated endpoint it touches is a cross-user leak. Use it for genuinely public responses and write an explicit interceptor for anything else (→ [Ch 13](13-interceptors.md)).

A useful discipline is to make caching opt-in with a decorator that forces the author to declare the variance:

```ts
export const Cacheable = (opts: { ttl: number; varyBy: Array<'user' | 'tenant' | 'locale'> }) =>
  SetMetadata(CACHEABLE_KEY, opts);

@Cacheable({ ttl: 60_000, varyBy: ['user'] })
@Get('me/summary')
summary() { … }
```

The interceptor then refuses to cache anything without a `varyBy` on an authenticated route — turning a silent leak into a startup or request-time error.

> 💼 **Interview** — "How do you decide what goes in a cache key?" The answer is every input the response varies by, and the follow-up is always about authenticated endpoints. Mentioning that a version prefix lets you invalidate everything at once during a deploy shows you have operated a cache, not just configured one.

---

## 20.4 Invalidation and Stampedes

There are three strategies, and each fails differently.

**TTL only.** Simple, and data is stale for up to the TTL. Correct when staleness is acceptable and quantifiable — a dashboard that can be a minute old.

**Write-through / explicit invalidation.** Delete the key when the underlying data changes. Fresher, and it fails when someone adds a write path and forgets:

```ts
async update(id: string, dto: UpdateTaskDto): Promise<Task> {
  const task = await this.repo.update(id, dto);
  await this.cache.del(`task:${id}`);           // easy to forget in the next writer
  await this.cache.del(`stats:${task.ownerId}`);
  return task;
}
```

Concentrating invalidation in the repository rather than the service reduces the number of places that must remember.

**Versioned keys.** Instead of deleting, bump a version that participates in every key:

```ts
const version = await this.cache.get<number>(`ver:tasks:${ownerId}`) ?? 1;
const key = `tasks:${ownerId}:v${version}`;
// on any write:
await this.cache.set(`ver:tasks:${ownerId}`, version + 1);
```

Nothing needs deleting — old entries simply become unreachable and expire on their own. This is the most robust option when many keys derive from one entity, because you cannot forget one.

**A stampede** happens when a popular key expires and every concurrent request misses simultaneously, all running the expensive query at once. The cache was absorbing that load, and its expiry hands it to the database in a single spike:

```ts
private readonly inFlight = new Map<string, Promise<unknown>>();

async getOrCompute<T>(key: string, ttlMs: number, compute: () => Promise<T>): Promise<T> {
  const cached = await this.safeGet<T>(key);
  if (cached !== undefined) return cached;

  // Per-process single-flight: concurrent misses share one computation
  // instead of each running the query.
  const existing = this.inFlight.get(key) as Promise<T> | undefined;
  if (existing) return existing;

  const promise = compute()
    .then(async (value) => {
      await this.safeSet(key, value, ttlMs);
      return value;
    })
    .finally(() => this.inFlight.delete(key));   // always clear, even on failure

  this.inFlight.set(key, promise);
  return promise;
}
```

This collapses concurrent misses **within one process**. Across replicas you need a distributed lock or, more simply, jittered TTLs so keys do not all expire at the same instant.

---

## 20.5 Common Mistakes and Best Practices

**Mistake: `if (cached)` as a hit check.** Cached `0`, `false`, and `''` are treated as misses.

**Mistake: cache keys without the principal.** Two users share an entry, which leaks data rather than breaking a feature.

**Mistake: `CacheInterceptor` globally on an authenticated API.** It keys by URL alone.

**Mistake: letting a cache failure fail the request.** The cache then reduces availability instead of improving it.

**Mistake: `ttl: 60` on cache-manager v6.** That is 60 milliseconds; v6 uses milliseconds, v5 used seconds.

**Mistake: in-memory caching of mutable data across replicas.** Invalidation on one instance leaves the others stale.

**Mistake: caching before measuring.** Most endpoints are slow because of an N+1 or a missing index, and a cache hides that until the cache is cold.

**Best practice: make caching opt-in with a decorator declaring the variance.** It forces the author to state what the response depends on, and lets a reviewer check it.

**Best practice: prefer versioned keys to explicit deletion.** You cannot forget to bump a version that every key already reads.

**Best practice: jitter TTLs.** Identical expiry times across keys and replicas produce synchronised stampedes.

---

## 20.6 Real-World Example: A Cache Service That Degrades Safely

A wrapper providing single-flight, safe failure, versioned invalidation, and keys that cannot omit identity.

```ts
// src/cache/cache.service.ts
import { Inject, Injectable, Logger } from '@nestjs/common';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

export interface CacheKeyParts {
  namespace: string;
  tenantId?: string;
  userId?: string;
  discriminator?: string;
}

@Injectable()
export class SafeCacheService {
  private readonly logger = new Logger(SafeCacheService.name);
  private readonly inFlight = new Map<string, Promise<unknown>>();

  constructor(@Inject(CACHE_MANAGER) private readonly cache: Cache) {}

  /**
   * Keys are built here and nowhere else. A structured object means a caller
   * cannot forget the tenant the way a hand-written template string allows.
   */
  private async buildKey(parts: CacheKeyParts): Promise<string> {
    const version = await this.getVersion(parts.namespace, parts.tenantId);
    return [
      'v1',                                   // global prefix: bump to flush everything
      parts.namespace,
      `ver${version}`,
      parts.tenantId ?? 'global',
      parts.userId ?? 'anon',
      parts.discriminator ?? '-',
    ].join(':');
  }

  private versionKey(namespace: string, tenantId?: string): string {
    return `ver:${namespace}:${tenantId ?? 'global'}`;
  }

  private async getVersion(namespace: string, tenantId?: string): Promise<number> {
    return (await this.safeGet<number>(this.versionKey(namespace, tenantId))) ?? 1;
  }

  /**
   * Invalidation by version bump rather than key deletion: every derived key
   * becomes unreachable at once, so there is no list of keys to remember.
   */
  async invalidate(namespace: string, tenantId?: string): Promise<void> {
    const key = this.versionKey(namespace, tenantId);
    const current = (await this.safeGet<number>(key)) ?? 1;
    await this.safeSet(key, current + 1, 0);   // 0 = no expiry
  }

  async getOrCompute<T>(
    parts: CacheKeyParts,
    ttlMs: number,
    compute: () => Promise<T>,
  ): Promise<T> {
    const key = await this.buildKey(parts);

    const cached = await this.safeGet<T>(key);
    if (cached !== undefined) return cached;   // `undefined`, not falsy

    const existing = this.inFlight.get(key) as Promise<T> | undefined;
    if (existing) return existing;

    const promise = compute()
      .then(async (value) => {
        // Jitter ±20% so keys written together do not expire together and
        // hand the database a synchronised spike.
        const jittered = Math.round(ttlMs * (0.8 + Math.random() * 0.4));
        await this.safeSet(key, value, jittered);
        return value;
      })
      .finally(() => this.inFlight.delete(key));

    this.inFlight.set(key, promise);
    return promise;
  }

  // A cache outage must degrade to the source of truth, never fail the request.
  private async safeGet<T>(key: string): Promise<T | undefined> {
    try {
      return await this.cache.get<T>(key);
    } catch (err) {
      this.logger.warn(`Cache read failed: ${key}`, err as Error);
      return undefined;
    }
  }

  private async safeSet(key: string, value: unknown, ttlMs: number): Promise<void> {
    try {
      await this.cache.set(key, value, ttlMs);
    } catch (err) {
      this.logger.warn(`Cache write failed: ${key}`, err as Error);
    }
  }
}
```

```ts
// src/tasks/tasks.service.ts
@Injectable()
export class TasksService {
  constructor(
    private readonly repo: TaskRepository,
    private readonly cache: SafeCacheService,
  ) {}

  getStats(userId: string, tenantId: string): Promise<Stats> {
    return this.cache.getOrCompute(
      { namespace: 'task-stats', tenantId, userId },
      300_000,
      () => this.repo.computeStats(userId),
    );
  }

  async update(id: string, dto: UpdateTaskDto, tenantId: string): Promise<Task> {
    const task = await this.repo.update(id, dto);
    // One call invalidates every derived key in the namespace.
    await this.cache.invalidate('task-stats', tenantId);
    return task;
  }
}
```

The decision worth defending is building keys from a structured object inside the service rather than letting callers pass a string. Template-string keys are more flexible and they make the dangerous mistake trivially easy: `` `stats:${userId}` `` looks complete, omits the tenant, and produces a cross-tenant leak that no test written against a single tenant will catch. Forcing callers to supply `CacheKeyParts` means the tenant and principal are parameters the type system asks for, the version prefix is applied automatically, and the one place keys are assembled is a short function a reviewer can actually check. The cost is that unusual keys need a new field rather than an ad-hoc string, which is a real constraint and the point — cache-key construction is exactly the kind of code where flexibility buys very little and a single omission leaks data.

---

## 20.7 Questions

**1.** MCQ — In `cache-manager` v6, TTL is expressed in: a) seconds b) milliseconds c) minutes d) ticks
<details><summary>Answer</summary>**b)** It was seconds in v5, which makes copied examples silently produce a 60ms cache.</details>

**2.** MCQ — `CacheInterceptor` keys entries by: a) user b) URL c) method and body d) headers
<details><summary>Answer</summary>**b)** Which makes it unsafe to apply globally on an authenticated API.</details>

**3.** MCQ — `if (cached)` incorrectly treats which cached value as a miss? a) `null` b) `0` c) `undefined` d) both a and b
<details><summary>Answer</summary>**b)** `0` is a legitimate cached value; use `!== undefined`.</details>

**4.** MCQ — With three replicas and an in-memory cache, invalidating on one instance: a) clears all b) clears one c) throws d) clears none
<details><summary>Answer</summary>**b)** The other two keep serving stale data.</details>

**5.** MCQ — A cache stampede happens when: a) the cache is full b) a popular key expires under load c) Redis restarts d) TTL is too long
<details><summary>Answer</summary>**b)** Every concurrent request misses at once and runs the expensive path.</details>

**6.** MCQ — Versioned keys invalidate by: a) deleting each key b) bumping a version in the key c) flushing the store d) shortening TTL
<details><summary>Answer</summary>**b)** Old keys become unreachable and expire naturally.</details>

**7.** Short — Why must a cache read failure not fail the request?
<details><summary>Answer</summary>Otherwise the cache reduces availability — an optional optimisation becomes a hard dependency, and a Redis outage becomes an application outage.</details>

**8.** Short — What must a cache key include?
<details><summary>Answer</summary>Every input the response varies by: the principal, the tenant, the API version, the serialization group, and any header that changes the body.</details>

**9.** Short — Why is a cross-user cache leak worse than staleness?
<details><summary>Answer</summary>Staleness shows wrong-but-authorised data; a key collision shows another user's data, which is a security incident rather than a bug.</details>

**10.** Short — Why jitter TTLs?
<details><summary>Answer</summary>Keys written at the same moment otherwise expire at the same moment across every replica, producing a synchronised stampede.</details>

**11.** Short — Why measure before caching?
<details><summary>Answer</summary>Most slow endpoints are slow because of an N+1 or a missing index; caching hides that until the cache is cold, at which point the underlying problem returns at the worst time.</details>

**12.** Output — A value of `0` is cached and the code uses `if (cached) return cached;`. What happens on the next request?
<details><summary>Answer</summary>A miss — the expensive computation runs again and re-caches `0`, permanently.</details>

**13.** Output — `ttl: 60` on cache-manager v6. How long do entries live?
<details><summary>Answer</summary>60 milliseconds, so effectively nothing is cached.</details>

**14.** Output — Global `CacheInterceptor`, users A and B both `GET /me`. What does B see?
<details><summary>Answer</summary>A's response, if A's request was cached first — the key is the URL alone.</details>

**15.** Output — 500 concurrent requests hit an expired key with no single-flight. How many expensive computations run?
<details><summary>Answer</summary>Up to 500 — the stampede the cache was preventing.</details>

**16.** Debug — After adding Redis caching, the API returns 500s whenever Redis is slow. Fix?
<details><summary>Answer</summary>Wrap reads and writes in try/catch and degrade to the source of truth. A cache must be an optimisation, not a dependency.</details>

**17.** Debug — A dashboard shows another tenant's numbers intermittently. Where do you look?
<details><summary>Answer</summary>The cache key. It almost certainly omits the tenant, so two tenants requesting the same path share an entry.</details>

**18.** Debug — Invalidation works locally but not in production. Cause?
<details><summary>Answer</summary>Production runs several replicas with an in-memory store, so the delete only affects the instance that handled the write. Move to a shared store.</details>

**19.** Debug — Every deploy causes a database load spike. Why?
<details><summary>Answer</summary>Restarting empties in-memory caches on every instance simultaneously, so all traffic hits the database cold. Use a shared store that survives restarts, or warm critical keys at boot.</details>

**20.** Debug — A new write path left stale data behind. Which invalidation strategy would have prevented it?
<details><summary>Answer</summary>Versioned keys — nothing needs to be deleted, so a new writer only has to bump the namespace version rather than know every derived key.</details>

**21.** Code — Write a safe read that treats `0` correctly and degrades on failure.
<details><summary>Answer</summary>

```ts
async get<T>(key: string): Promise<T | undefined> {
  try {
    const value = await this.cache.get<T>(key);
    return value;                      // `undefined` is the only miss sentinel
  } catch (err) {
    this.logger.warn(`Cache read failed: ${key}`, err);
    return undefined;
  }
}
```
</details>

**22.** Code — Implement single-flight so concurrent misses share one computation.
<details><summary>Answer</summary>

```ts
private readonly inFlight = new Map<string, Promise<unknown>>();

async getOrCompute<T>(key: string, ttl: number, compute: () => Promise<T>): Promise<T> {
  const hit = await this.get<T>(key);
  if (hit !== undefined) return hit;

  const existing = this.inFlight.get(key) as Promise<T> | undefined;
  if (existing) return existing;

  const p = compute()
    .then(async (v) => { await this.set(key, v, ttl); return v; })
    .finally(() => this.inFlight.delete(key));   // clear on failure too

  this.inFlight.set(key, p);
  return p;
}
```
</details>

**23.** Code — Build a cache key that cannot omit the tenant.
<details><summary>Answer</summary>

```ts
interface KeyParts { namespace: string; tenantId: string; userId?: string; extra?: string }

private key(p: KeyParts): string {
  return ['v1', p.namespace, p.tenantId, p.userId ?? 'anon', p.extra ?? '-'].join(':');
}
```
Making `tenantId` a required field means the compiler enforces what a template string leaves to memory.
</details>

**24.** Scenario — A teammate proposes caching every `GET` endpoint globally for 60 seconds to "cut database load in half". Evaluate it.
<details><summary>Answer</summary>The mechanism is right and the blanket application is wrong on two counts. Any authenticated endpoint cached by URL leaks data across users, so the change is a security incident rather than a performance improvement unless every key carries the principal. Beyond that, a uniform 60-second TTL applies the same staleness budget to a product catalogue and to an account balance, and one of those is unacceptable. The load reduction is also probably attainable more cheaply — a blanket cache commonly masks one or two endpoints with N+1 queries, and fixing those helps even on a cold cache. Cache deliberately: opt in per endpoint, declare what each response varies by, and set the TTL from how stale that specific data may acceptably be.</details>

**25.** Scenario — Your cache hit rate is 95% but p99 latency is unchanged. What is happening and what do you do?
<details><summary>Answer</summary>A high hit rate with unchanged tail latency almost always means the 5% of misses are the slow requests, and they are the ones users notice. Check whether misses correlate with a specific key set — a long tail of rarely-requested items that never stay warm — or with expiry, in which case the misses are stampedes where many requests queue behind one slow computation. The remedies differ: for a long tail, caching is not the answer and the underlying query needs optimising, since you cannot warm what nobody requests twice; for expiry-driven misses, single-flight and jitter collapse the spike, and refresh-ahead — recomputing shortly before expiry — removes it entirely. Measure the latency of misses separately from hits, because an aggregate p99 across both hides which of these you have.</details>

---

## Chapter Summary

- `cache-manager` v6 in NestJS 11 takes TTL in **milliseconds**, so a copied `ttl: 60` yields a 60ms cache that silently does nothing.
- In-memory caching cannot be invalidated across replicas, which makes a shared store the requirement for any data that changes.
- `if (cached)` treats `0` and `false` as misses; `undefined` is the only correct miss sentinel.
- A cache key must contain every dimension the response varies by, and omitting the principal turns a performance feature into a cross-user data leak.
- A cache read must never fail a request — an optimisation that reduces availability is worse than no cache.
- Versioned keys beat explicit deletion because a new write path cannot forget a key it never had to know about, and jittered TTLs plus single-flight are what keep expiry from becoming a stampede.

---

## Practice Exercises

1. Cache the value `0` and read it back with `if (cached)`, then observe that the cache never hits.
2. Apply `CacheInterceptor` globally to an authenticated endpoint and demonstrate the leak with two accounts.
3. Stop Redis mid-request and observe the 500, then add safe degradation and re-run.
4. Fire 200 concurrent requests at an expired key with and without single-flight, and count the database queries.
5. Implement versioned-key invalidation, then add a new write path and confirm it cannot leave stale data.

---

## Mini Project: A Caching Layer With Guardrails

Build caching that makes the dangerous mistakes hard to express.

**Requirements:**

- An opt-in `@Cacheable({ ttl, varyBy })` decorator; the interceptor refuses to cache an authenticated route with no `varyBy`.
- Keys built in one place from a structured object where tenant is a required field.
- Single-flight per process plus jittered TTLs, with a test proving 200 concurrent misses cause one computation.
- Safe degradation verified by a test that makes the store throw and asserts the endpoint still returns 200.
- A metrics counter for hits, misses, and errors, with miss latency recorded separately from hit latency.

---

**Next:** [Chapter 21 — Authentication →](21-authentication.md)
