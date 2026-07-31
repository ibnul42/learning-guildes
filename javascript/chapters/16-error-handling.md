# Chapter 16 — Error Handling

## Learning Objectives

- Use `try`/`catch`/`finally` correctly, including with async code
- Build custom error classes and use `cause`
- Distinguish operational errors from programmer bugs
- Handle unhandled rejections and global errors

---

## 16.1 `try` / `catch` / `finally`

```js
try {
  risky();
} catch (err) {          // binding is optional: `catch {` if unused (ES2019)
  console.error(err.message);
} finally {
  cleanup();             // ALWAYS runs — even on return or throw
}
```

`finally` runs before the function actually returns, and can override the return value:

```js
function f() {
  try { return 'try'; }
  finally { return 'finally'; }   // ⚠️ wins — and swallows exceptions too
}
f();   // 'finally'
```

Never `return` from `finally`. Use it for cleanup only.

`try`/`catch` does **not** catch:
- Errors thrown inside a `setTimeout` callback (different tick, different stack)
- Rejected promises you don't `await`
- Errors in other threads (workers)

```js
try { setTimeout(() => { throw new Error('boom'); }, 0); } catch { /* never runs */ }
```

---

## 16.2 The Error Object

```js
const err = new Error('Something failed');
err.name;       // 'Error'
err.message;    // 'Something failed'
err.stack;      // string — not standardized, but universally present

// ES2022: cause preserves the original error
try { await db.query(); }
catch (dbErr) {
  throw new Error('Could not load user', { cause: dbErr });
}
```

Built-in types: `Error`, `TypeError`, `RangeError`, `ReferenceError`, `SyntaxError`, `URIError`, `EvalError`, and `AggregateError` (from `Promise.any`).

### Custom errors

```js
class AppError extends Error {
  constructor(message, { code, status = 500, cause, retryable = false } = {}) {
    super(message, { cause });
    this.name = this.constructor.name;   // 'ValidationError', not 'Error'
    this.code = code;
    this.status = status;
    this.retryable = retryable;
    Error.captureStackTrace?.(this, this.constructor);   // V8: hide this frame
  }
}

class ValidationError extends AppError {
  constructor(field, message) {
    super(message, { code: 'VALIDATION', status: 400 });
    this.field = field;
  }
}
class NotFoundError extends AppError {
  constructor(resource) {
    super(`${resource} not found`, { code: 'NOT_FOUND', status: 404 });
  }
}
```

Now callers can branch on type:

```js
catch (err) {
  if (err instanceof ValidationError) return res.status(400).json({ field: err.field });
  if (err instanceof NotFoundError) return res.status(404).end();
  throw err;                                          // re-throw what you don't handle
}
```

> ⚠️ **Anything can be thrown** — `throw 'a string'` is legal. Always guard:
> ```js
> catch (e) { const err = e instanceof Error ? e : new Error(String(e)); }
> ```

---

## 16.3 Async Error Handling

```js
// async/await — use try/catch normally
async function load(id) {
  try {
    const res = await fetch(`/api/${id}`);
    if (!res.ok) throw new AppError(`HTTP ${res.status}`, { status: res.status });
    return await res.json();          // note: `await` inside try, or errors escape
  } catch (err) {
    throw new AppError('Load failed', { cause: err });
  }
}

// Promises — .catch()
fetch(url).then(r => r.json()).catch(handleError);

// ⚠️ Classic bug: forgetting `return`/`await` means the try block exits first
async function bad() {
  try { return fetch(url); }        // ❌ rejection escapes this try
  catch { /* never runs */ }
}
async function good() {
  try { return await fetch(url); }  // ✅
  catch { }
}
```

### Concurrency

```js
await Promise.all(tasks);        // rejects on the FIRST failure; others keep running
await Promise.allSettled(tasks); // never rejects; inspect each result
await Promise.any(tasks);        // first success; rejects with AggregateError if all fail
```

For partial-failure tolerance, `allSettled` is almost always what you want:

```js
const results = await Promise.allSettled(urls.map(fetchJson));
const ok = results.filter(r => r.status === 'fulfilled').map(r => r.value);
const failed = results.filter(r => r.status === 'rejected').map(r => r.reason);
```

---

## 16.4 Global Handlers

```js
// Browser
window.addEventListener('error', (e) => report(e.error));
window.addEventListener('unhandledrejection', (e) => {
  report(e.reason);
  e.preventDefault();      // suppress the console warning if you've handled it
});

// Node
process.on('uncaughtException', (err) => { log(err); process.exit(1); });
process.on('unhandledRejection', (reason) => { log(reason); process.exit(1); });
```

In Node, an unhandled rejection **terminates the process** by default (since v15). Don't fight it — after an uncaught exception the process state is unknown, so log and exit. Let a supervisor restart you.

---

## 16.5 Operational Errors vs Bugs

| | Operational | Programmer bug |
|---|---|---|
| Examples | network timeout, 404, invalid user input, disk full | `undefined is not a function`, off-by-one, bad type |
| Expected? | yes | no |
| Response | handle, retry, show a message | let it crash; fix the code |
| Catch it? | yes, specifically | only at the top level, to log |

The distinction drives design: **catch what you can act on, let everything else propagate.** A `catch` block that swallows a `TypeError` hides a real bug behind a generic "something went wrong."

---

## 16.6 Common Mistakes and Best Practices

**Mistake: empty catch blocks.** `catch {}` deletes information. At minimum, log with context.

**Mistake: catching too broadly.** Wrap the smallest scope that can fail.

**Mistake: losing the original error.** Always pass `{ cause }`.

**Mistake: throwing non-Errors.** No stack trace, no `.message`.

**Mistake: `console.log(err)` and continuing** as if nothing happened.

**Best practice: fail fast at boundaries.** Validate inputs at entry and throw immediately with a clear message.

**Best practice: error messages should say what failed, with what input, and what to do.** `"Invalid config: 'port' must be 1-65535, got 0"` beats `"Invalid input"`.

**Best practice: never leak internals to users.** Log the stack server-side, return a correlation id to the client.

---

## 16.7 Real-World Example: A Resilient Service Layer

```js
class HttpError extends Error {
  constructor(status, body, url) {
    super(`HTTP ${status} for ${url}`);
    this.name = 'HttpError';
    this.status = status;
    this.body = body;
    this.retryable = status >= 500 || status === 429;
  }
}

async function request(url, { retries = 3, timeoutMs = 5000, ...init } = {}) {
  let lastError;

  for (let attempt = 1; attempt <= retries; attempt++) {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), timeoutMs);

    try {
      const res = await fetch(url, { ...init, signal: controller.signal });
      if (!res.ok) {
        const body = await res.text().catch(() => '');
        throw new HttpError(res.status, body, url);
      }
      return await res.json();

    } catch (err) {
      lastError = err;

      // AbortError from our timeout is retryable; a caller's abort is not.
      const isTimeout = err.name === 'AbortError' && !init.signal?.aborted;
      const retryable = isTimeout || err.retryable || err instanceof TypeError; // network

      if (!retryable || attempt === retries) break;
      await new Promise(r => setTimeout(r, 200 * 2 ** (attempt - 1)));

    } finally {
      clearTimeout(timer);      // always — even on success or early break
    }
  }

  throw new Error(`Request to ${url} failed after ${retries} attempts`, { cause: lastError });
}
```

`finally` guarantees the timer is cleared on every path. The `cause` chain means a log at the top level still shows the underlying `HttpError` and its status.

---

## 16.8 Questions

**1.** MCQ — `finally` runs: a) only on error b) only on success c) always d) never with return
<details><summary>Answer</summary>**c)**</details>

**2.** MCQ — `catch` without a binding is: a) invalid b) ES2019+ valid c) deprecated d) strict-only
<details><summary>Answer</summary>**b)**</details>

**3.** MCQ — `Promise.all` rejects: a) after all settle b) on first rejection c) never d) with AggregateError
<details><summary>Answer</summary>**b)**</details>

**4.** MCQ — `Promise.any` rejects with: a) Error b) TypeError c) AggregateError d) the first reason
<details><summary>Answer</summary>**c)**</details>

**5.** MCQ — `error.cause` was added in: a) ES2015 b) ES2020 c) ES2022 d) ES2025
<details><summary>Answer</summary>**c)**</details>

**6.** Short — Why doesn't `try/catch` catch errors from `setTimeout`?
<details><summary>Answer</summary>The callback runs later on a fresh call stack; the `try` block has already exited.</details>

**7.** Short — Operational error vs programmer bug?
<details><summary>Answer</summary>Operational errors are expected runtime conditions you can handle (network, input). Bugs are defects you should fix, not catch.</details>

**8.** Short — Why is `return` inside `finally` dangerous?
<details><summary>Answer</summary>It overrides the `try`/`catch` return value and swallows in-flight exceptions.</details>

**9.** Short — Why set `this.name = this.constructor.name` in a custom error?
<details><summary>Answer</summary>Otherwise `name` inherits as `'Error'`, so logs and `err.name` checks are useless.</details>

**10.** Short — When would you use `allSettled` over `all`?
<details><summary>Answer</summary>When partial success is acceptable and you want every result, not just the first failure.</details>

**11.** Output —
```js
function f() { try { return 1; } finally { console.log('F'); } }
console.log(f());
```
<details><summary>Answer</summary>`F` then `1`.</details>

**12.** Output —
```js
async function f() { try { return Promise.reject(new Error('x')); } catch { return 'caught'; } }
f().then(console.log, e => console.log('rejected', e.message));
```
<details><summary>Answer</summary>`rejected x` — no `await`, so the rejection escapes the try.</details>

**13.** Output —
```js
try { throw 'plain string'; } catch (e) { console.log(typeof e, e.message); }
```
<details><summary>Answer</summary>`'string' undefined`</details>

**14.** Output —
```js
try { null.x; } catch (e) { console.log(e.constructor.name); }
```
<details><summary>Answer</summary>`TypeError`</details>

**15.** Output —
```js
const e = new Error('outer', { cause: new Error('inner') });
console.log(e.cause.message);
```
<details><summary>Answer</summary>`inner`</details>

**16.** Debug — A retry loop retries 404s forever.
<details><summary>Answer</summary>No distinction between retryable and permanent failures. Only retry 5xx, 429, and network/timeout errors.</details>

**17.** Debug — Errors show as `{}` in logs.
<details><summary>Answer</summary>`JSON.stringify(error)` returns `{}` because `message` and `stack` are non-enumerable. Log `err.stack`, or serialize explicitly with `{ name, message, stack, cause }`.</details>

**18.** Debug — A Node service silently dies with no log.
<details><summary>Answer</summary>An unhandled promise rejection terminates the process. Add `process.on('unhandledRejection')` to log before exiting, and audit for missing `await`/`.catch`.</details>

**19.** Debug — Users see "Cannot read properties of undefined (reading 'id')" in a toast.
<details><summary>Answer</summary>An internal bug is being surfaced verbatim. Catch at the boundary, log the real error with a correlation id, and show a generic message plus that id.</details>

**20.** Code — Write `tryCatch(fn)` returning `[error, result]` Go-style.
<details><summary>Answer</summary>

```js
async function tryCatch(fn) {
  try { return [null, await fn()]; }
  catch (e) { return [e instanceof Error ? e : new Error(String(e)), null]; }
}
```
</details>

**21.** Code — Write `serializeError(err)` capturing name, message, stack, and the full cause chain.
<details><summary>Answer</summary>

```js
function serializeError(err, depth = 5) {
  if (!(err instanceof Error)) return { message: String(err) };
  return {
    name: err.name, message: err.message, stack: err.stack,
    ...(err.code && { code: err.code }),
    ...(err.cause && depth > 0 && { cause: serializeError(err.cause, depth - 1) }),
  };
}
```
The depth cap prevents infinite recursion on a cyclic cause chain.
</details>

**22.** Code — Write an error hierarchy for a payments API with at least four error types and a `toResponse()` method.
<details><summary>Answer</summary>Extend a shared `PaymentError` base carrying `code`/`status`/`userMessage`; subclass into `CardDeclinedError`, `InsufficientFundsError`, `GatewayTimeoutError` (retryable), and `InvalidAmountError`. `toResponse()` returns `{ status, body: { code, message: this.userMessage, correlationId } }` — never `this.message` or the stack.</details>

**23.** Scenario — Should you wrap every function in try/catch?
<details><summary>Answer</summary>No. Catch where you can *do* something: retry, fall back, translate to a user message, or add context before re-throwing. Everywhere else, let it propagate to a single boundary handler.</details>

**24.** Scenario — Your team's logs are full of "Error: Error". Fix the culture, not just the code.
<details><summary>Answer</summary>Enforce a rule that every thrown error states what failed and with what input, add a lint rule banning empty catch blocks, require `{ cause }` on re-throws, and add a structured logger that serializes the cause chain. Review error messages in code review the way you'd review copy.</details>

---

## Chapter Summary

- `finally` always runs; never `return` from it.
- `try/catch` can't catch async callbacks or un-awaited rejections — `await` inside the `try`.
- Subclass `Error`, set `name`, and always pass `{ cause }` when re-throwing.
- Anything can be thrown; normalize non-Errors in your catch.
- `all` fails fast, `allSettled` reports everything, `any` takes the first success.
- Handle operational errors; let bugs crash and get fixed.
- Log internals, show users a generic message plus a correlation id.

---

## Practice Exercises

1. Build an error hierarchy for a file-upload service and map each type to an HTTP status.
2. Write `retry(fn, { attempts, isRetryable, backoff })` with jitter.
3. Implement a `Result<T, E>` type with `map`, `mapErr`, `unwrapOr`, and `match`.
4. Add a global error boundary to a small app that reports to a mock endpoint and de-duplicates identical errors.
5. Write a test proving your `finally` cleanup runs on success, on throw, and on early return.

---

**Next:** [Chapter 17 — Modules →](17-modules.md)
