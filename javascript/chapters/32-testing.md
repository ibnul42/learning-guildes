# Chapter 32 — Testing Fundamentals

## Learning Objectives

- Choose the right test level for a given behaviour, and know where the pyramid misleads
- Write async tests that fail loudly rather than passing vacuously
- Use mocks, stubs, spies, and fakes deliberately — and recognise over-mocking
- Query the DOM the way a user does, with Testing Library

---

## 32.1 Levels, and Where the Pyramid Lies to You

| Level | Tests | Speed | Confidence per test | Failure tells you |
|---|---|---|---|---|
| **Unit** | One function or module in isolation | ~1ms | Low | Exactly what broke |
| **Integration** | Several units together, real collaborators | ~10–100ms | Medium | Roughly where |
| **E2E** | The whole app in a real browser | ~1–30s | High | Something broke |

The pyramid says "mostly unit, some integration, few E2E". Where it misleads: a suite of unit tests
with every dependency mocked can be entirely green while the application is broken, because nothing
tested that the pieces fit. The mocks encode your *belief* about the collaborator, and beliefs drift.

The more useful shape is the "testing trophy": a thick integration layer, because that's where the
confidence-per-cost ratio peaks. Test a module against real collaborators wherever the real ones are
fast and deterministic — a real reducer, a real DOM, a real in-memory store — and mock only at the
boundaries you don't own: the network, the clock, the filesystem, randomness.

**What to test:** behaviour that would break a user's workflow, edge cases at boundaries (empty,
one, many, null, negative, huge), error paths, and every bug you've fixed (as a regression test).

**What not to test:** implementation details (private methods, internal call counts), third-party
library behaviour, trivial getters, and generated code. A test that must change when you rename a
private function is a test that costs more than it earns.

---

## 32.2 Anatomy of a Test

Arrange, act, assert. One logical assertion per test, and a name that states the behaviour.

```js
import { describe, it, expect, beforeEach, vi } from 'vitest';   // Jest's API is near-identical

describe('formatPrice', () => {
  it('formats a whole number with two decimals', () => {
    const result = formatPrice(30, 'GBP');       // arrange + act
    expect(result).toBe('£30.00');               // assert
  });

  it('throws on a negative amount', () => {
    expect(() => formatPrice(-1, 'GBP')).toThrow(RangeError);
  });
});
```

Name tests so a failure reads as a sentence: `formatPrice > throws on a negative amount`.
`test1`, `works`, and `should work correctly` tell you nothing at 3am.

**Matchers worth knowing:**

```js
expect(x).toBe(y);                 // Object.is — identity. Use for primitives.
expect(x).toEqual(y);              // deep structural equality; ignores undefined properties
expect(x).toStrictEqual(y);        // deep + class identity + undefined properties count
expect(x).toBeCloseTo(0.3, 5);     // floats: 0.1 + 0.2 !== 0.3
expect(arr).toContain(item);       // identity for primitives
expect(arr).toContainEqual(obj);   // deep for objects
expect(fn).toThrow(/pattern/);
expect(obj).toMatchObject({ a: 1 }); // partial match; ignores extra keys
expect(spy).toHaveBeenCalledWith(expect.objectContaining({ id: 1 }));
expect(x).toBeInstanceOf(Error);
await expect(promise).rejects.toThrow('boom');
await expect(promise).resolves.toEqual({ ok: true });
```

> ⚠️ **Gotcha** — `toBe` on objects compares references, so `expect({a:1}).toBe({a:1})` always fails.
> `toEqual` treats `{a: undefined}` and `{}` as equal, which hides a real class of bug; use
> `toStrictEqual` when the difference matters.

Hooks and their scope:

```js
beforeEach(() => { /* fresh state per test — the default choice */ });
afterEach(() => { vi.restoreAllMocks(); document.body.innerHTML = ''; });
beforeAll(() => { /* expensive setup shared by the file */ });
```

Shared mutable state between tests is the leading cause of "passes alone, fails in the suite". Set
up in `beforeEach`, and never rely on test execution order.

---

## 32.3 Async Testing and the Forgotten `await`

The failure mode is silent: the test finishes before the assertion runs, and passes.

```js
// ❌ Passes even when the assertion would fail — nothing awaited the promise
it('loads the user', () => {
  fetchUser(1).then(user => expect(user.name).toBe('never checked'));
});

// ✅ Return or await
it('loads the user', async () => {
  const user = await fetchUser(1);
  expect(user.name).toBe('Ada');
});

// ✅ Asserting rejection — note the awaits
it('rejects on 404', async () => {
  await expect(fetchUser(999)).rejects.toThrow('Not found');
});

// ❌ This passes if fetchUser resolves! The catch never runs and nothing notices.
it('rejects on 404', async () => {
  try { await fetchUser(999); }
  catch (err) { expect(err.message).toBe('Not found'); }
});

// ✅ If you must use try/catch, assert it was reached
it('rejects on 404', async () => {
  expect.assertions(1);                     // fails if no assertion ran
  try { await fetchUser(999); expect.unreachable(); }
  catch (err) { expect(err.message).toBe('Not found'); }
});
```

**Fake timers** let you test time without spending it:

```js
beforeEach(() => vi.useFakeTimers());
afterEach(() => vi.useRealTimers());

it('debounces to one call', () => {
  const fn = vi.fn();
  const debounced = debounce(fn, 300);
  debounced(); debounced(); debounced();
  expect(fn).not.toHaveBeenCalled();
  vi.advanceTimersByTime(300);
  expect(fn).toHaveBeenCalledTimes(1);
});

it('retries with backoff', async () => {
  const promise = withRetry(failTwiceThenSucceed, 3)();
  await vi.runAllTimersAsync();             // advances timers AND drains microtasks
  await expect(promise).resolves.toBe('ok');
});
```

> ⚠️ **Gotcha** — with fake timers, `advanceTimersByTime` runs timer callbacks but doesn't flush the
> promise microtasks they schedule. Use `await vi.advanceTimersByTimeAsync(n)` or
> `runAllTimersAsync` for anything involving promises, or your test hangs or asserts too early.

---

## 32.4 Test Doubles, and Over-Mocking

| Double | What it does | Use when |
|---|---|---|
| **Spy** | Records calls, keeps real behaviour | You want to assert a call happened |
| **Stub** | Returns canned values, no real behaviour | You need a specific return |
| **Mock** | Stub with assertions about how it's called | The interaction *is* the behaviour |
| **Fake** | A working lightweight implementation | The collaborator is slow or external |

```js
const spy = vi.spyOn(console, 'warn');              // real behaviour preserved
const stub = vi.fn().mockResolvedValue({ id: 1 });
const mock = vi.fn();
expect(mock).toHaveBeenCalledWith('expected');

// A fake: real logic, no I/O. Usually the best of the four.
const createFakeDb = (seed = []) => {
  const rows = [...seed];
  return {
    async findById(id) { return rows.find(r => r.id === id) ?? null; },
    async insert(row) { rows.push(row); return row; },
  };
};
```

**Over-mocking** produces tests that pass while the app is broken:

```js
// ❌ This tests the mock, not the code. It would pass if getUser were deleted.
vi.mock('./api.js', () => ({ getUser: vi.fn().mockResolvedValue({ name: 'Ada' }) }));
it('shows the user name', async () => {
  render(UserCard, { id: 1 });
  expect(await screen.findByText('Ada')).toBeInTheDocument();
});

// ✅ Mock the network boundary, exercise everything you wrote
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

const server = setupServer(
  http.get('/api/users/:id', ({ params }) =>
    HttpResponse.json({ id: Number(params.id), name: 'Ada' }))
);
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));   // unmocked calls fail loudly
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

The rule: **mock at the boundary you don't own.** Network, clock, `crypto.randomUUID`, filesystem,
third-party SDKs. Everything inside your own module graph should be real, because that's what
integration confidence is made of. If a module is too hard to test without mocking its internals,
that's a design signal — inject the dependency (Chapter 30) rather than reaching for module mocking.

**Snapshot testing** captures output and compares it next run. Its failure mode is people running
`--update` on a real regression because the diff is 400 lines and nobody reads it.

```js
expect(formatInvoice(invoice)).toMatchInlineSnapshot();   // ✅ inline: visible in review
expect(container).toMatchSnapshot();                      // ❌ huge, opaque, rubber-stamped
```

Keep snapshots small and inline, and assert on specific values wherever you can name what matters.

**Coverage is a signal, not a goal.** 100% line coverage with no assertions is achievable and
worthless; 70% on the paths that matter beats 95% including trivial getters. Use it to *find*
untested branches, then decide whether each deserves a test.

---

## 32.5 Testing the DOM, and E2E

Testing Library's premise: query the way a user finds things, so tests break when the UX breaks and
not when the markup is refactored.

```js
import { screen, render, within } from '@testing-library/dom';
import userEvent from '@testing-library/user-event';

// Query priority, best to worst:
screen.getByRole('button', { name: /save/i });     // ✅ accessible name — also tests a11y
screen.getByLabelText('Email');                    // ✅ form fields
screen.getByText('Welcome back');
screen.getByPlaceholderText('Search');             // ⚠️ placeholders aren't labels
screen.getByTestId('save-btn');                    // ❌ last resort: invisible to users
```

| Prefix | Missing element | Async | Use for |
|---|---|---|---|
| `getBy` | Throws | No | It should be there now |
| `queryBy` | Returns `null` | No | Asserting absence |
| `findBy` | Rejects | Yes | It will appear |

```js
it('submits the form and shows a confirmation', async () => {
  const user = userEvent.setup();                       // real event sequences, not synthetic clicks
  render(contactForm());

  await user.type(screen.getByLabelText('Email'), 'ada@example.com');
  await user.click(screen.getByRole('button', { name: /send/i }));

  expect(await screen.findByRole('status')).toHaveTextContent('Message sent');
  expect(screen.queryByRole('alert')).not.toBeInTheDocument();
});
```

Prefer `userEvent` over `fireEvent`: `user.type` fires the full `keydown`/`keypress`/`input`/`keyup`
sequence with focus handling, which is what your code actually receives in a browser.

**E2E with Playwright** for critical user journeys — signup, checkout, the one flow that must never
break:

```js
import { test, expect } from '@playwright/test';

test('a user can complete checkout', async ({ page }) => {
  await page.goto('/products');
  await page.getByRole('link', { name: 'Blue Widget' }).click();
  await page.getByRole('button', { name: 'Add to basket' }).click();
  await page.getByRole('link', { name: /basket \(1\)/i }).click();
  await page.getByLabel('Card number').fill('4242424242424242');
  await page.getByRole('button', { name: 'Pay' }).click();

  // Web-first assertions auto-retry until timeout — no sleeps, no flake
  await expect(page.getByRole('heading', { name: 'Order confirmed' })).toBeVisible();
});
```

Never `waitForTimeout`. A fixed sleep is either too short (flaky) or too long (slow), and it's the
number one source of unreliable E2E suites. Assert on state, and let the framework retry.

**TDD, honestly:** writing the test first genuinely produces better interfaces, because you design
from the call site and can't test a badly-shaped API comfortably. It also forces you to define
"done" before you start. It's a poor fit for exploratory work where you don't yet know the shape, and
strict red-green-refactor on every trivial function is dogma that wastes time. Use it where the
requirements are clear; write the test immediately after where they aren't. Either way, a bug fix
starts with a failing test — that's the case with no counterargument.

---

## 32.6 Common Mistakes and Best Practices

**Mistake: not awaiting an async assertion.** The test passes without asserting anything.

**Mistake: `try/catch` to assert rejection without `expect.assertions`.** Passes when the promise
resolves.

**Mistake: mocking your own modules.** You end up testing the mock's fidelity to a belief.

**Mistake: asserting on call counts of internal functions.** The test now fails on any refactor that
doesn't change behaviour.

**Mistake: shared state between tests.** "Passes alone, fails in the suite" — set up in
`beforeEach`.

**Mistake: `waitForTimeout` / `sleep` in tests.** Flaky and slow at once. Wait for a condition.

**Mistake: huge snapshots.** Nobody reads a 400-line diff; `--update` becomes reflex.

**Mistake: `getByTestId` as the default query.** Test ids are invisible to users, so they don't
verify the thing that matters — and using roles catches accessibility bugs for free.

**Mistake: treating coverage percentage as the goal.** It measures execution, not verification.

**Best practice: name the behaviour, not the function.** `rejects a transfer that would overdraw`,
not `test transfer 3`.

**Best practice: mock only the boundaries you don't own.** Network, time, randomness, filesystem.

**Best practice: every bug fix gets a regression test first.** Confirm it fails, then fix it.

**Best practice: make failure messages self-explanatory.** Assert on the specific value, not on a
boolean — `expect(total).toBe(30)` tells you what went wrong; `expect(isValid).toBe(true)` doesn't.

---

## 32.7 Real-World Example: Testing `createFetcher`

Here's a full suite for a fetcher with retries, in-flight deduplication, and abort support — the
kind of module where the interesting behaviour is all timing and failure, and therefore the kind
people skip testing. Note what's mocked (the network, the clock) and what isn't (everything we
wrote).

```js
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
import { createFetcher } from './fetcher.js';

describe('createFetcher', () => {
  let fetchMock;

  beforeEach(() => {
    fetchMock = vi.fn();
    vi.stubGlobal('fetch', fetchMock);          // the boundary we don't own
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.useRealTimers();
    vi.unstubAllGlobals();
  });

  const ok = (body) => new Response(JSON.stringify(body), {
    status: 200, headers: { 'content-type': 'application/json' },
  });
  const fail = (status) => new Response('', { status });

  // ── Happy path ───────────────────────────────────────────────────────────────────
  it('resolves with the parsed body', async () => {
    fetchMock.mockResolvedValue(ok({ id: 1 }));
    const fetcher = createFetcher();
    await expect(fetcher.get('/api/users/1')).resolves.toEqual({ id: 1 });
    expect(fetchMock).toHaveBeenCalledTimes(1);
  });

  it('throws on a non-2xx response with the status attached', async () => {
    fetchMock.mockResolvedValue(fail(404));
    const fetcher = createFetcher({ retries: 0 });
    // Assert the SHAPE callers depend on, not just that it threw.
    await expect(fetcher.get('/api/users/999')).rejects.toMatchObject({ status: 404 });
  });

  // ── Retries: this is why we need fake timers ─────────────────────────────────────
  it('retries a 500 and succeeds on the third attempt', async () => {
    fetchMock
      .mockResolvedValueOnce(fail(500))
      .mockResolvedValueOnce(fail(500))
      .mockResolvedValueOnce(ok({ id: 1 }));

    const promise = createFetcher({ retries: 3 }).get('/api/users/1');
    // Async variant: advances timers AND drains the microtasks each retry schedules.
    await vi.runAllTimersAsync();

    await expect(promise).resolves.toEqual({ id: 1 });
    expect(fetchMock).toHaveBeenCalledTimes(3);
  });

  it('does not retry a 400 — client errors are not transient', async () => {
    fetchMock.mockResolvedValue(fail(400));
    const promise = createFetcher({ retries: 3 }).get('/api/x');
    await expect(promise).rejects.toMatchObject({ status: 400 });
    expect(fetchMock).toHaveBeenCalledTimes(1);       // the assertion that catches the bug
  });

  it('backs off exponentially', async () => {
    fetchMock.mockResolvedValue(fail(503));
    const delays = [];
    vi.spyOn(globalThis, 'setTimeout').mockImplementation((fn, ms) => {
      delays.push(ms);                                // record the schedule, then run immediately
      return queueMicrotask(fn);
    });

    await createFetcher({ retries: 4, baseDelay: 100 }).get('/api/x').catch(() => {});
    expect(delays).toEqual([100, 200, 400]);          // 3 waits for 4 attempts
  });

  // ── Deduplication ────────────────────────────────────────────────────────────────
  it('dedupes concurrent identical requests', async () => {
    let resolveFetch;
    fetchMock.mockReturnValue(new Promise(r => { resolveFetch = r; }));

    const fetcher = createFetcher();
    const [a, b] = [fetcher.get('/api/users/1'), fetcher.get('/api/users/1')];
    resolveFetch(ok({ id: 1 }));

    expect(await a).toEqual(await b);
    expect(fetchMock).toHaveBeenCalledTimes(1);
  });

  it('does not dedupe across different URLs', async () => {
    fetchMock.mockResolvedValue(ok({}));
    const fetcher = createFetcher();
    await Promise.all([fetcher.get('/api/a'), fetcher.get('/api/b')]);
    expect(fetchMock).toHaveBeenCalledTimes(2);
  });

  it('allows a new request after the in-flight one settles', async () => {
    fetchMock.mockResolvedValue(ok({}));
    const fetcher = createFetcher();
    await fetcher.get('/api/a');
    await fetcher.get('/api/a');
    expect(fetchMock).toHaveBeenCalledTimes(2);       // the cache is in-flight only, not a result cache
  });

  // ── Abort ────────────────────────────────────────────────────────────────────────
  it('rejects with an AbortError when the signal fires', async () => {
    const controller = new AbortController();
    fetchMock.mockImplementation((url, { signal }) => new Promise((_, reject) => {
      signal.addEventListener('abort', () => reject(new DOMException('', 'AbortError')));
    }));

    const promise = createFetcher().get('/api/slow', { signal: controller.signal });
    controller.abort();
    await expect(promise).rejects.toMatchObject({ name: 'AbortError' });
  });

  it('does not retry an aborted request', async () => {
    const controller = new AbortController();
    fetchMock.mockRejectedValue(new DOMException('', 'AbortError'));
    const promise = createFetcher({ retries: 3 }).get('/api/x', { signal: controller.signal });
    await vi.runAllTimersAsync();
    await expect(promise).rejects.toMatchObject({ name: 'AbortError' });
    expect(fetchMock).toHaveBeenCalledTimes(1);
  });

  // ── Edge cases: the tests that find real bugs ────────────────────────────────────
  it('handles a 204 with no body', async () => {
    fetchMock.mockResolvedValue(new Response(null, { status: 204 }));
    await expect(createFetcher().get('/api/x')).resolves.toBeNull();
  });

  it('surfaces a parse error rather than hanging', async () => {
    fetchMock.mockResolvedValue(new Response('not json', {
      status: 200, headers: { 'content-type': 'application/json' },
    }));
    await expect(createFetcher({ retries: 0 }).get('/api/x')).rejects.toThrow(SyntaxError);
  });
});
```

The decision worth explaining is `expect(fetchMock).toHaveBeenCalledTimes(1)` in the "does not retry
a 400" test. Asserting the rejection alone would pass whether the fetcher retried three times or
none — the outcome is identical either way, and a bug that hammers a failing endpoint four times
would ship unnoticed. Behaviour you care about that isn't visible in the return value has to be
asserted separately, and this is the main legitimate use of call-count assertions: not "did my
internal helper get called", but "did we make three network requests when we promised one".

---

## 32.8 Questions

**1.** MCQ — `expect({a:1}).toBe({a:1})`: a) passes b) fails c) throws d) is deprecated
<details><summary>Answer</summary>**b)** `toBe` is `Object.is`; use `toEqual`.</details>

**2.** MCQ — Which passes when the promise unexpectedly resolves? a) `await expect(p).rejects.toThrow()` b) a bare `try/catch` with the assertion in `catch` c) `expect.assertions(1)` d) `await expect(p).resolves`
<details><summary>Answer</summary>**b)** The `catch` never runs, so nothing is asserted and the test is green.</details>

**3.** MCQ — Testing Library's highest-priority query is: a) `getByTestId` b) `getByText` c) `getByRole` d) `querySelector`
<details><summary>Answer</summary>**c)** It matches how assistive technology and users find controls.</details>

**4.** MCQ — `queryBy*` differs from `getBy*` in that it: a) is async b) returns `null` when absent c) retries d) matches partially
<details><summary>Answer</summary>**b)** Which is what makes it the right tool for asserting absence.</details>

**5.** MCQ — You should mock: a) every dependency b) nothing c) boundaries you don't own d) only classes
<details><summary>Answer</summary>**c)** Network, clock, randomness, filesystem, third-party SDKs.</details>

**6.** MCQ — 100% line coverage guarantees: a) no bugs b) every line executed c) every branch tested d) correct behaviour
<details><summary>Answer</summary>**b)** Execution is not verification — a test with no assertions still covers lines.</details>

**7.** Short — Why can a fully-mocked unit suite be green while the app is broken?
<details><summary>Answer</summary>Each mock encodes an assumption about a collaborator's contract. When the real collaborator changes, the mocks don't, so every unit passes against a contract nobody implements any more. Only integration tests catch that.</details>

**8.** Short — What's the difference between a stub and a mock?
<details><summary>Answer</summary>A stub supplies canned return values so the code under test can proceed. A mock additionally asserts *how* it was called, making the interaction itself the thing under test.</details>

**9.** Short — When is a fake better than a mock?
<details><summary>Answer</summary>When the collaborator has real logic you'd otherwise re-encode in every test. An in-memory database fake behaves correctly for all callers, so tests exercise real integration without the I/O and without repeating stub setup.</details>

**10.** Short — Why prefer `userEvent` over `fireEvent`?
<details><summary>Answer</summary>`fireEvent` dispatches one synthetic event; `userEvent` reproduces the full sequence a browser produces — focus, `keydown`, `keypress`, `input`, `keyup` — which is what the code under test actually handles in production.</details>

**11.** Short — Why is `waitForTimeout` in an E2E test a bug?
<details><summary>Answer</summary>A fixed sleep is either shorter than the operation on a slow run (flaky) or longer than needed on a fast one (slow), and it never encodes what you're actually waiting for. Assert on the resulting state and let the framework retry until it holds.</details>

**12.** Short — Why should a bug fix start with a failing test?
<details><summary>Answer</summary>It proves the test actually exercises the bug — a test written after the fix may pass for the wrong reason and never guard the regression it was written for.</details>

**13.** Output — Does this test pass, and does it verify anything?
```js
it('doubles', () => {
  Promise.resolve(2).then(n => expect(n * 2).toBe(999));
});
```
<details><summary>Answer</summary>It passes, and verifies nothing. The test function returns before the `then` callback runs; the assertion's rejection is unhandled and the runner has already moved on.</details>

**14.** Output —
```js
expect({ a: 1, b: undefined }).toEqual({ a: 1 });
expect({ a: 1, b: undefined }).toStrictEqual({ a: 1 });
```
<details><summary>Answer</summary>The first passes — `toEqual` ignores `undefined` properties. The second fails, which is why `toStrictEqual` is the right choice when a missing key and an explicit `undefined` mean different things.</details>

**15.** Output —
```js
vi.useFakeTimers();
const fn = vi.fn();
setTimeout(() => Promise.resolve().then(fn), 100);
vi.advanceTimersByTime(100);
expect(fn).toHaveBeenCalled();
```
<details><summary>Answer</summary>Fails. `advanceTimersByTime` runs the timer callback but doesn't drain the microtask it queued. Use `await vi.advanceTimersByTimeAsync(100)`.</details>

**16.** Output —
```js
let count = 0;
it('a', () => { count++; expect(count).toBe(1); });
it('b', () => { count++; expect(count).toBe(1); });
```
<details><summary>Answer</summary>`a` passes, `b` fails — shared module state leaks between tests. Reset it in `beforeEach`.</details>

**17.** Debug — A test passes locally and fails in CI, roughly one run in five.
<details><summary>Answer</summary>Order dependence or a timing assumption. Run the suite with a randomised order and with `--no-threads` to see whether isolation or concurrency is the trigger, and look for shared state, a real timer, or an un-awaited promise. Fixed sleeps and dates constructed from `Date.now()` are the usual culprits.</details>

**18.** Debug — After a refactor that changed no behaviour, 40 tests fail.
<details><summary>Answer</summary>The tests assert on implementation — internal call counts, private methods, or a large snapshot of markup. Rewrite them to assert on observable behaviour through the public interface; the fact that a behaviour-preserving change broke them is the diagnosis.</details>

**19.** Debug — A component test passes but the feature is broken in the browser.
<details><summary>Answer</summary>The API module was mocked, so the test never exercised the request-building or response-parsing code where the bug lives. Mock at the network layer with MSW instead, and set `onUnhandledRequest: 'error'` so any request you didn't anticipate fails the test.</details>

**20.** Debug — `expect(spy).toHaveBeenCalledTimes(1)` fails with 2, and both calls look identical.
<details><summary>Answer</summary>Either an effect runs twice (a double-invoked lifecycle, or a listener registered on each render), or the spy wasn't reset between tests. Add `vi.restoreAllMocks()` in `afterEach` to rule out the second, then log the arguments and stack of each call to find the second registration.</details>

**21.** Debug — A Playwright test intermittently clicks the wrong row of a table.
<details><summary>Answer</summary>The locator matches multiple elements and resolves before the list finishes updating, so it acts on stale content. Scope the locator — `page.getByRole('row', { name: /Blue Widget/ }).getByRole('button')` — so it identifies the row by content rather than position, and assert the expected row count first so the action waits for the settled list.</details>

**22.** Code — Write a test for `debounce` using fake timers, covering leading and trailing edges.
<details><summary>Answer</summary>

```js
describe('debounce', () => {
  beforeEach(() => vi.useFakeTimers());
  afterEach(() => vi.useRealTimers());

  it('fires once on the trailing edge', () => {
    const fn = vi.fn();
    const d = debounce(fn, 100);
    d(1); d(2); d(3);
    expect(fn).not.toHaveBeenCalled();
    vi.advanceTimersByTime(100);
    expect(fn).toHaveBeenCalledExactlyOnceWith(3);   // last arguments win
  });

  it('fires immediately with leading: true and not again', () => {
    const fn = vi.fn();
    const d = debounce(fn, 100, { leading: true, trailing: false });
    d('a'); d('b');
    expect(fn).toHaveBeenCalledExactlyOnceWith('a');
    vi.advanceTimersByTime(100);
    expect(fn).toHaveBeenCalledTimes(1);
  });

  it('cancel() prevents a pending call', () => {
    const fn = vi.fn();
    const d = debounce(fn, 100);
    d(); d.cancel();
    vi.advanceTimersByTime(500);
    expect(fn).not.toHaveBeenCalled();
  });
});
```
</details>

**23.** Code — Write a 60-line assertion library and test runner.
<details><summary>Answer</summary>

```js
const tests = [];
let currentSuite = [];

export const describe = (name, fn) => {
  currentSuite.push(name);
  fn();
  currentSuite.pop();
};

export const it = (name, fn) => tests.push({ name: [...currentSuite, name].join(' > '), fn });

class AssertionError extends Error { name = 'AssertionError'; }

const deepEqual = (a, b) => {
  if (Object.is(a, b)) return true;
  if (typeof a !== 'object' || typeof b !== 'object' || !a || !b) return false;
  if (Array.isArray(a) !== Array.isArray(b)) return false;
  const ka = Object.keys(a), kb = Object.keys(b);
  return ka.length === kb.length && ka.every(k => deepEqual(a[k], b[k]));
};

const show = (v) => typeof v === 'string' ? JSON.stringify(v) : String(v);

export const expect = (actual) => ({
  toBe(expected) {
    if (!Object.is(actual, expected)) {
      throw new AssertionError(`expected ${show(actual)} to be ${show(expected)}`);
    }
  },
  toEqual(expected) {
    if (!deepEqual(actual, expected)) {
      throw new AssertionError(`expected ${JSON.stringify(actual)} to equal ${JSON.stringify(expected)}`);
    }
  },
  toThrow(match) {
    let threw = false, error;
    try { actual(); } catch (err) { threw = true; error = err; }
    if (!threw) throw new AssertionError('expected function to throw');
    if (match && !String(error.message).match(match)) {
      throw new AssertionError(`expected "${error.message}" to match ${match}`);
    }
  },
  get not() {
    return {
      toBe: (expected) => {
        if (Object.is(actual, expected)) throw new AssertionError(`expected not to be ${show(expected)}`);
      },
    };
  },
});

export async function run() {
  let passed = 0;
  const failures = [];
  for (const { name, fn } of tests) {
    try { await fn(); passed++; console.log(`  ✓ ${name}`); }
    catch (err) { failures.push({ name, err }); console.log(`  ✗ ${name}`); }
  }
  for (const { name, err } of failures) console.log(`\n${name}\n  ${err.message}`);
  console.log(`\n${passed} passed, ${failures.length} failed`);
  return failures.length === 0;
}
```
</details>

**24.** Code — Write a fake clock that lets you test a TTL cache without waiting.
<details><summary>Answer</summary>

```js
export const createFakeClock = (start = 0) => {
  let now = start;
  return { now: () => now, advance: (ms) => { now += ms; } };
};

// The cache takes the clock as a dependency, which is the whole point (Chapter 30).
const clock = createFakeClock();
const cache = createCache({ ttlMs: 1000, clock });

cache.set('k', 1);
expect(cache.get('k')).toBe(1);
clock.advance(1001);
expect(cache.get('k')).toBeUndefined();
```
</details>

**25.** Scenario — A team has 4,000 unit tests, 92% coverage, and ships a broken checkout every other release.
<details><summary>Answer</summary>The suite is testing units in isolation with mocked collaborators, so nothing verifies that the pieces compose — the coverage number measures execution, not integration. Add a handful of E2E tests for the checkout journey first, since that's the flow that keeps breaking and one test there buys more confidence than a hundred more unit tests. Then convert the highest-value unit tests to integration tests by replacing internal mocks with real modules and mocking only the network. Expect the count to *fall* and confidence to rise.</details>

**26.** Scenario — A colleague wants 100% coverage enforced in CI. What do you propose instead?
<details><summary>Answer</summary>A coverage floor is worth having, but 100% pays for itself only in a few contexts — a library, a payments core, anything where a single untested branch is a real liability. Elsewhere it drives tests written to touch lines rather than verify behaviour, and it makes every `catch` branch for an impossible error a chore. I'd set a floor around 80% on changed files rather than the whole repo, so new code is tested without a six-month backfill project, plus a hard rule that every bug fix ships with a regression test. Then use the coverage report as a review aid — an untested branch is a question, not a build failure.</details>

---

## Chapter Summary

- The pyramid undersells integration tests; mocked-everything unit suites go green while the app breaks.
- An un-awaited async assertion passes silently — return or await, and use `rejects`/`resolves`.
- Mock only boundaries you don't own: network, clock, randomness, filesystem.
- Query by role and label so tests survive refactors and catch accessibility bugs for free.
- Never sleep in a test; wait for a condition and let the framework retry.
- Coverage finds untested branches; it never tells you a behaviour is verified.

---

## Practice Exercises

1. Take an existing test that mocks an internal module and rewrite it to mock only the network with MSW.
2. Write a test that passes without asserting anything, then fix it with `expect.assertions`.
3. Test a retry-with-backoff function's exact delay schedule using fake timers.
4. Convert five `getByTestId` queries to role-based queries and fix any accessibility problems that surfaces.
5. Find a module in your codebase you can't test without mocking internals, and refactor it to accept injected dependencies.

---

## Mini Project: A Test Runner From Scratch

Build a working runner and assertion library in about 200 lines, with no dependencies.

- `describe`/`it` with nesting, and full names in the output
- `beforeEach`/`afterEach`/`beforeAll`/`afterAll` with correct nesting order
- Matchers: `toBe`, `toEqual`, `toThrow`, `toHaveBeenCalledWith`, plus `.not` for each
- A `fn()` spy that records calls, arguments, and return values
- Async test support, with a per-test timeout that reports which test hung
- `it.only` and `it.skip`, and a non-zero exit code on failure
- A readable diff for `toEqual` failures — the part that makes a runner actually usable

---

**Next:** [Chapter 33 — ES6 → ES2025 Reference →](33-es6-to-es2025.md)
