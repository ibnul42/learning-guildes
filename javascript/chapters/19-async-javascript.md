# Chapter 19 — Callbacks, Promises, and async/await

## Learning Objectives

- Understand promise states, chaining, and the combinators
- Write clean `async`/`await` code with correct error handling
- Control concurrency (sequential, parallel, bounded)
- Convert callback APIs to promises

---

## 19.1 From Callbacks to Promises

The callback era's problem wasn't nesting — it was that errors and control flow had no structure.

```js
// Callback hell: error handling repeated at every level
getUser(id, (err, user) => {
  if (err) return handle(err);
  getPosts(user.id, (err, posts) => {
    if (err) return handle(err);
    getComments(posts[0].id, (err, comments) => {
      if (err) return handle(err);
      render(comments);
    });
  });
});
```

A **promise** is an object representing a future value, in one of three states:

- **pending** — not settled yet
- **fulfilled** — settled with a value
- **rejected** — settled with a reason

Once settled, a promise is **immutable** — it can never change state or value again.

```js
const p = new Promise((resolve, reject) => {
  // Executor runs SYNCHRONOUSLY, immediately
  setTimeout(() => resolve('done'), 100);
});

p.then(v => console.log(v))
 .catch(e => console.error(e))
 .finally(() => console.log('cleanup'));   // no arguments, passes value through
```

### Chaining

`.then` always returns a **new** promise. What you return inside determines what flows on:

```js
Promise.resolve(1)
  .then(v => v + 1)              // 2 — plain value wraps into a promise
  .then(v => Promise.resolve(v * 2))  // 4 — a returned promise is ADOPTED (flattened)
  .then(v => { throw new Error('x'); })
  .then(v => console.log('skipped'))  // skipped — we're on the rejection path
  .catch(e => 'recovered')       // returning a value RESUMES the fulfilled path
  .then(v => console.log(v));    // 'recovered'
```

Two rules worth memorising:

1. **Returning** in `.then` passes a value along; **throwing** switches to the rejection path.
2. A `.catch` that returns normally puts the chain back on the success path.

> ⚠️ Forgetting `return` inside `.then` is the most common promise bug — the chain continues with `undefined` without waiting.

---

## 19.2 async/await

Syntactic sugar over promises, and almost always the better choice.

```js
async function loadDashboard(userId) {
  try {
    const user = await getUser(userId);
    const [posts, settings] = await Promise.all([   // parallel where independent
      getPosts(user.id),
      getSettings(user.id),
    ]);
    return { user, posts, settings };
  } catch (err) {
    throw new Error('Dashboard load failed', { cause: err });
  }
}
```

**Facts:**

- `async` functions **always** return a promise, wrapping any returned value.
- `await` unwraps a promise, or passes a non-promise through (after one microtask tick).
- The function body runs **synchronously** until the first `await`.
- `await` in a non-async function is only legal at the top level of an ES module.

```js
async function f() { return 1; }
f() instanceof Promise;   // true
```

### Sequential vs parallel — the #1 async performance bug

```js
// ❌ 3 seconds — each waits for the previous
const a = await fetchA();   // 1s
const b = await fetchB();   // 1s
const c = await fetchC();   // 1s

// ✅ 1 second — all three start immediately
const [a, b, c] = await Promise.all([fetchA(), fetchB(), fetchC()]);

// ✅ Start now, await later — useful when you need `a` before `c`
const bPromise = fetchB();
const a = await fetchA();
const b = await bPromise;
```

Only sequence when there's a genuine dependency.

```js
// ❌ Sequential in disguise
for (const id of ids) results.push(await fetch(id));

// ✅ Parallel
const results = await Promise.all(ids.map(id => fetch(id)));
```

---

## 19.3 Combinators

```js
Promise.all(ps);          // all fulfilled → array of values; ANY rejection → rejects immediately
Promise.allSettled(ps);   // never rejects → [{status:'fulfilled',value}|{status:'rejected',reason}]
Promise.race(ps);         // first to SETTLE (fulfill or reject) wins
Promise.any(ps);          // first to FULFILL; all reject → AggregateError
```

```js
// Timeout via race
const withTimeout = (promise, ms) => Promise.race([
  promise,
  new Promise((_, rej) => setTimeout(() => rej(new Error('Timeout')), ms)),
]);

// Partial failure tolerance
const results = await Promise.allSettled(urls.map(fetchJson));
const ok = results.flatMap(r => r.status === 'fulfilled' ? [r.value] : []);

// First responder wins
const fastest = await Promise.any([mirror1(), mirror2(), mirror3()]);
```

> ⚠️ `Promise.all` rejecting does **not** cancel the other promises. They keep running, and their rejections may become unhandled. Use `AbortController` for real cancellation.

### Bounded concurrency

Unlimited parallelism will exhaust connection pools and trip rate limits.

```js
async function pool(items, worker, limit = 5) {
  const results = new Array(items.length);
  let cursor = 0;

  async function runner() {
    while (cursor < items.length) {
      const i = cursor++;                       // claim an index atomically (single-threaded)
      results[i] = await worker(items[i], i);
    }
  }

  await Promise.all(Array.from({ length: Math.min(limit, items.length) }, runner));
  return results;
}

await pool(urls, fetchJson, 6);
```

---

## 19.4 Cancellation and Promisification

```js
// AbortController — the standard cancellation primitive
const controller = new AbortController();
const promise = fetch(url, { signal: controller.signal });
controller.abort();   // rejects with a DOMException named 'AbortError'

// Timeout shorthand
fetch(url, { signal: AbortSignal.timeout(5000) });
// Combine signals
fetch(url, { signal: AbortSignal.any([userSignal, AbortSignal.timeout(5000)]) });
```

```js
// Promisify a Node-style callback API
const promisify = (fn) => (...args) =>
  new Promise((resolve, reject) =>
    fn(...args, (err, result) => err ? reject(err) : resolve(result)));

// Promisify an event-based API, with cleanup
function waitForEvent(target, event, { signal } = {}) {
  return new Promise((resolve, reject) => {
    const opts = { once: true, signal };
    target.addEventListener(event, resolve, opts);
    signal?.addEventListener('abort', () => reject(signal.reason), { once: true });
  });
}
```

---

## 19.5 Common Mistakes and Best Practices

**Mistake: `forEach` with `async`.** It ignores the returned promises. Use `for...of` + `await` or `Promise.all(map(...))`.

**Mistake: sequential awaits for independent work.** See §19.2.

**Mistake: the explicit promise anti-pattern.**
```js
// ❌
return new Promise((res, rej) => getData().then(res).catch(rej));
// ✅
return getData();
```

**Mistake: floating promises.** An un-awaited async call whose rejection nobody handles crashes Node. Either `await` it or attach `.catch()`.

**Mistake: `try { return promise } catch`.** Without `await`, the rejection escapes the `try`.

**Mistake: `await` inside a `.map` expecting sequencing.** `map` doesn't wait.

**Best practice: `async`/`await` over `.then` chains** — better stack traces, easier debugging, natural `try`/`catch`.

**Best practice: always bound concurrency** on anything hitting a network or database.

**Best practice: make cancellation a first-class parameter.** Accept a `signal` in every async function that does I/O.

---

## 19.6 Real-World Example: A Robust Data Fetcher

```js
/**
 * Deduplicates in-flight requests, caches, retries with backoff+jitter,
 * supports cancellation, and bounds concurrency.
 */
export function createFetcher({ concurrency = 6, retries = 3, ttlMs = 30_000 } = {}) {
  const cache = new Map();     // key -> { at, value }
  const inFlight = new Map();  // key -> promise
  let active = 0;
  const waiters = [];

  const acquire = () => active < concurrency
    ? (active++, Promise.resolve())
    : new Promise(r => waiters.push(r));

  const release = () => {
    active--;
    const next = waiters.shift();
    if (next) { active++; next(); }
  };

  async function attempt(url, signal) {
    let lastError;
    for (let i = 1; i <= retries; i++) {
      signal?.throwIfAborted();
      try {
        const res = await fetch(url, { signal });
        if (res.status >= 500 || res.status === 429) throw new Error(`HTTP ${res.status}`);
        if (!res.ok) return Object.assign(new Error(`HTTP ${res.status}`), { fatal: true });
        return await res.json();
      } catch (err) {
        if (err.name === 'AbortError' || err.fatal) throw err;
        lastError = err;
        if (i === retries) break;
        const jitter = Math.random() * 100;
        await new Promise(r => setTimeout(r, 200 * 2 ** (i - 1) + jitter));
      }
    }
    throw new Error(`Failed after ${retries} attempts: ${url}`, { cause: lastError });
  }

  return async function get(url, { signal, force = false } = {}) {
    const hit = cache.get(url);
    if (!force && hit && Date.now() - hit.at < ttlMs) return hit.value;

    // Deduplicate: concurrent callers for the same URL share one request
    if (inFlight.has(url)) return inFlight.get(url);

    const promise = (async () => {
      await acquire();
      try {
        const value = await attempt(url, signal);
        cache.set(url, { at: Date.now(), value });
        return value;
      } finally {
        release();
        inFlight.delete(url);
      }
    })();

    inFlight.set(url, promise);
    return promise;
  };
}
```

Every production data layer needs these five behaviours. Writing them once beats scattering `fetch` calls across a codebase.

---

## 19.7 Questions

**1.** MCQ — An `async` function returns: a) the value b) a promise c) undefined d) a generator
<details><summary>Answer</summary>**b)**</details>

**2.** MCQ — `Promise.race` settles on the first: a) fulfillment b) rejection c) settlement d) timeout
<details><summary>Answer</summary>**c)**</details>

**3.** MCQ — `Promise.any` with all rejections throws: a) Error b) AggregateError c) TypeError d) the first reason
<details><summary>Answer</summary>**b)**</details>

**4.** MCQ — The promise executor runs: a) asynchronously b) synchronously c) as a microtask d) on next tick
<details><summary>Answer</summary>**b)**</details>

**5.** MCQ — Can a settled promise change state? a) yes b) no
<details><summary>Answer</summary>**b)**</details>

**6.** MCQ — Does `Promise.all` rejecting cancel the other promises? a) yes b) no
<details><summary>Answer</summary>**b)** They continue running.</details>

**7.** Short — `all` vs `allSettled` vs `race` vs `any`?
<details><summary>Answer</summary>`all`: all succeed or reject on first failure. `allSettled`: wait for everything, never reject. `race`: first to settle either way. `any`: first to fulfill.</details>

**8.** Short — Why is `forEach` with `async` wrong?
<details><summary>Answer</summary>`forEach` discards return values, so it neither waits nor propagates rejections.</details>

**9.** Short — What is a floating promise and why care?
<details><summary>Answer</summary>An async call whose result is never awaited or caught. Its rejection becomes an unhandled rejection, which terminates a Node process and hides failures in the browser.</details>

**10.** Short — How do you cancel a `fetch`?
<details><summary>Answer</summary>Pass an `AbortController`'s `signal` and call `controller.abort()`; the promise rejects with an `AbortError`.</details>

**11.** Short — Why bound concurrency?
<details><summary>Answer</summary>Unbounded parallel requests exhaust sockets and connection pools, trigger rate limits, and can degrade the server you're calling.</details>

**12.** Output —
```js
console.log(1);
new Promise(r => { console.log(2); r(); }).then(() => console.log(3));
console.log(4);
```
<details><summary>Answer</summary>`1 2 4 3` — the executor is synchronous.</details>

**13.** Output —
```js
Promise.resolve(1).then(v => { v + 1; }).then(v => console.log(v));
```
<details><summary>Answer</summary>`undefined` — no `return` in the first `.then`.</details>

**14.** Output —
```js
Promise.reject(new Error('a')).catch(() => 'b').then(v => console.log(v));
```
<details><summary>Answer</summary>`b` — `catch` returning a value resumes the success path.</details>

**15.** Output —
```js
async function f() { throw new Error('x'); }
f().catch(e => console.log('caught', e.message));
```
<details><summary>Answer</summary>`caught x` — a throw in an async function becomes a rejection.</details>

**16.** Output —
```js
const p = Promise.resolve(1);
p.then(v => console.log('a', v));
p.then(v => console.log('b', v));
```
<details><summary>Answer</summary>`a 1` then `b 1` — multiple handlers on one promise all run, in registration order.</details>

**17.** Output —
```js
async function f() { return Promise.resolve(1); }
f().then(v => console.log(v));
```
<details><summary>Answer</summary>`1` — returned promises are adopted, not double-wrapped.</details>

**18.** Debug — A loop of 100 API calls takes 100 seconds.
<details><summary>Answer</summary>Sequential `await` in the loop. Use `Promise.all` with a bounded pool (~6 concurrent).</details>

**19.** Debug — `try/catch` around an async call never catches.
<details><summary>Answer</summary>Missing `await`. `try { return await fn(); }`, not `try { return fn(); }`.</details>

**20.** Debug — A Node worker exits silently under load.
<details><summary>Answer</summary>Unhandled rejection from a floating promise. Add `.catch()` to fire-and-forget calls and a `process.on('unhandledRejection')` logger.</details>

**21.** Code — Implement `promiseAll(promises)` from scratch.
<details><summary>Answer</summary>

```js
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    const items = [...promises];
    const results = new Array(items.length);
    let remaining = items.length;
    if (remaining === 0) return resolve([]);
    items.forEach((p, i) => {
      Promise.resolve(p).then(v => {
        results[i] = v;
        if (--remaining === 0) resolve(results);
      }, reject);
    });
  });
}
```
</details>

**22.** Code — Implement `promiseAllSettled`.
<details><summary>Answer</summary>

```js
const promiseAllSettled = (ps) => Promise.all(
  [...ps].map(p => Promise.resolve(p).then(
    value => ({ status: 'fulfilled', value }),
    reason => ({ status: 'rejected', reason })
  ))
);
```
</details>

**23.** Code — Implement `retryWithBackoff(fn, {attempts, baseMs, isRetryable, signal})`.
<details><summary>Answer</summary>

```js
async function retryWithBackoff(fn, { attempts = 3, baseMs = 200, isRetryable = () => true, signal } = {}) {
  let last;
  for (let i = 1; i <= attempts; i++) {
    signal?.throwIfAborted();
    try { return await fn(); }
    catch (e) {
      last = e;
      if (!isRetryable(e) || i === attempts) break;
      await new Promise(r => setTimeout(r, baseMs * 2 ** (i - 1) + Math.random() * 100));
    }
  }
  throw last;
}
```
</details>

**24.** Code — Write `sequential(tasks)` and `parallel(tasks)` and explain when each is right.
<details><summary>Answer</summary>

```js
const sequential = async (tasks) => {
  const out = [];
  for (const t of tasks) out.push(await t());
  return out;
};
const parallel = (tasks) => Promise.all(tasks.map(t => t()));
```
Sequential when tasks depend on each other or must not overload a resource; parallel when independent.
</details>

**25.** Scenario — An autocomplete shows stale results when the user types quickly.
<details><summary>Answer</summary>Race condition: an earlier request resolves after a later one. Abort the previous request with an `AbortController` on each keystroke, and/or tag each request with a sequence number and discard responses that aren't the latest. Debounce the input too (Chapter 24).</details>

**26.** Scenario — You need to upload 10,000 files without overwhelming the server or losing progress on failure.
<details><summary>Answer</summary>A bounded worker pool (5–10 concurrent), per-file retry with exponential backoff and jitter, `allSettled` semantics so one failure doesn't abort the batch, persisted progress so the job is resumable, and an `AbortController` for user cancellation.</details>

---

## Chapter Summary

- Promises are immutable once settled; `.then` returns a new promise; returning adopts, throwing rejects.
- `async` functions always return promises and run synchronously until the first `await`.
- Sequential `await` for independent work is the most common async performance bug.
- `all` (fail fast), `allSettled` (report all), `race` (first settled), `any` (first fulfilled).
- `Promise.all` rejecting doesn't cancel anything — use `AbortController`.
- Bound concurrency on all I/O; make `signal` a standard parameter.

---

## Practice Exercises

1. Implement `Promise` from scratch: `then`, `catch`, `finally`, `resolve`, `reject`, `all`, `race`, and the thenable-adoption rules.
2. Build a task queue with configurable concurrency, priorities, and pause/resume.
3. Write an async pipeline where each stage can time out independently.
4. Convert a callback-based library to promises using `promisify`, preserving `this`.
5. Reproduce the autocomplete race condition, then fix it two different ways and test both.

---

## Mini Project: A Job Queue with Persistence

Build `queue.js` exporting `createQueue({ concurrency, storage })`.

**Requirements:**

- `add(job, { priority, maxRetries, timeoutMs })` returns a promise resolving with the job's result.
- Bounded concurrency; higher priority jobs jump the queue.
- Per-job timeout via `AbortSignal.timeout`, and retry with exponential backoff + jitter.
- `pause()`, `resume()`, `clear()`, and events for `progress`, `success`, `failure`, `drain`.
- Persist pending jobs to the injected `storage` so a restart resumes them.
- Graceful shutdown: `close()` waits for in-flight jobs, rejecting queued ones.

**Stretch:** add a circuit breaker that stops dispatching after N consecutive failures and half-opens after a cooldown.

---

**Next:** [Chapter 20 — Fetch, JSON, and HTTP →](20-fetch-json-http.md)
