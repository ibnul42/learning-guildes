# Appendix E — Best Practices Checklist

Reviewable, item by item. Every line is specific enough to check against real code — "Always check
`res.ok` after `fetch`" rather than "handle errors properly".

Use it three ways: as a self-review before opening a pull request, as a reviewer's prompt when
something feels wrong but you can't name it, and as an audit of an unfamiliar codebase.

**Not every item applies to every project.** A prototype doesn't need a performance budget. Skip
deliberately, not by accident.

---

## Contents

| Section | Items |
|---|---|
| [Code Style and Naming](#code-style-and-naming) | 12 |
| [Variables and Types](#variables-and-types) | 14 |
| [Functions](#functions) | 13 |
| [Objects and Arrays](#objects-and-arrays) | 14 |
| [Async](#async) | 16 |
| [Error Handling](#error-handling) | 13 |
| [Modules](#modules) | 10 |
| [DOM](#dom) | 14 |
| [Performance](#performance) | 15 |
| [Memory](#memory) | 10 |
| [Security](#security) | 16 |
| [Testing](#testing) | 14 |
| [Accessibility](#accessibility) | 13 |
| [Code Review](#code-review) | 11 |
| [Production Readiness](#production-readiness) | 14 |

**199 items.**

---

## Code Style and Naming

- [ ] Name things for what they mean, not what they are — `retryCount`, not `num`
- [ ] Booleans read as assertions: `isLoading`, `hasPermission`, `canEdit`
- [ ] Functions are verb phrases; values are noun phrases
- [ ] Avoid abbreviations except the universal ones (`id`, `url`, `http`)
- [ ] Plural names hold collections; singular names hold one thing
- [ ] No negated booleans in names — `isEnabled`, never `isNotDisabled`
- [ ] Constants that are genuinely constant are `SCREAMING_SNAKE_CASE`; config objects are not
- [ ] One concept per name across the codebase — pick `user` or `account`, not both
- [ ] Comments explain *why*, never *what*; if the what isn't obvious, rename something
- [ ] Delete commented-out code — version control already remembers it
- [ ] A formatter (Prettier, Biome) runs in CI, so style is never a review topic
- [ ] Lint rules are errors, not warnings — a warning nobody fixes is a rule nobody has

## Variables and Types

- [ ] `const` by default; `let` only where reassignment is real; never `var`
- [ ] `===` everywhere, with `== null` the one permitted exception for a nullish check
- [ ] Declare variables at first use, not at the top of the function
- [ ] One declaration per statement — no `let a, b, c`
- [ ] `??` for defaults, not `||`, unless you specifically want falsy to fall through
- [ ] `??=` over `||=` for the same reason
- [ ] `Number.isNaN`, never the global `isNaN`, which coerces
- [ ] `Number.isInteger` rather than `x % 1 === 0`
- [ ] `parseInt` always with an explicit radix — or use `Number()` if that's what you meant
- [ ] Money as integer minor units; never a float
- [ ] IDs above 2⁵³ transported as strings, not JSON numbers
- [ ] No implicit coercion in conditions where `0` or `''` is a valid value
- [ ] `Object.hasOwn(obj, key)` rather than `obj.hasOwnProperty(key)`
- [ ] Validate external data at the boundary, once, into a known shape

## Functions

- [ ] One job per function; if the name needs "and", split it
- [ ] Three parameters maximum — beyond that, take an options object
- [ ] Options objects destructured with defaults in the signature, so the contract is visible
- [ ] Return early rather than nesting — guard clauses first
- [ ] No boolean parameters that switch behaviour; two named functions are clearer
- [ ] Pure where possible; effects pushed to the edges
- [ ] Don't mutate parameters — return a new value
- [ ] Consistent return type: don't return an object sometimes and `null` other times without saying so
- [ ] Arrow functions for callbacks; named `function` declarations for anything that appears in a stack trace
- [ ] Data argument last in anything intended for currying or composition
- [ ] No side effects in a getter — property access reads as free
- [ ] Default parameters over `arguments` inspection
- [ ] Functions used as callbacks bound once, not per render

## Objects and Arrays

- [ ] Prefer the non-mutating array methods: `toSorted`, `toReversed`, `toSpliced`, `with`
- [ ] Never `sort()` an array you don't own — it mutates in place
- [ ] Always supply a comparator to `sort` for numbers; the default is lexicographic
- [ ] Comparators return a number, never a boolean
- [ ] `localeCompare` for user-visible string sorting
- [ ] `Map` when keys aren't strings, order matters, or entries churn
- [ ] `Set` for membership tests inside a loop — never `array.includes` in a `filter`
- [ ] `Object.create(null)` or `Map` for dictionaries with untrusted keys
- [ ] `Object.groupBy` / `Map.groupBy` rather than a grouping `reduce`
- [ ] `structuredClone` for deep copies, never the JSON round-trip
- [ ] No sparse arrays — method behaviour is inconsistent across them
- [ ] `at(-1)` rather than `arr[arr.length - 1]`
- [ ] Avoid `delete` on objects; rebuild with destructuring or use a `Map`
- [ ] Immutable state updates with structural sharing, so identity comparison stays meaningful

## Async

- [ ] Every promise is awaited, returned, or explicitly `.catch()`ed — never floating
- [ ] `res.ok` checked after every `fetch` — it does not reject on 4xx or 5xx
- [ ] Independent operations run in parallel; `await` in a loop only when iterations depend on each other
- [ ] `Promise.allSettled` when you need every outcome, not `all`
- [ ] `AbortController` on every cancellable operation, and its signal threaded all the way down
- [ ] Superseded requests aborted, so a slow response can't overwrite a newer one
- [ ] Timeouts on every network call — `AbortSignal.timeout(ms)`
- [ ] Retries only on transient failures (5xx, network), never on 4xx
- [ ] Exponential backoff with jitter, and a cap
- [ ] Concurrency bounded — never fire 10,000 requests at once
- [ ] In-flight requests deduplicated by request identity
- [ ] No `forEach` with an async callback; it doesn't wait
- [ ] `return await` inside a `try` block, so the `catch` actually sees the rejection
- [ ] Async generators for paginated or streamed sources, so backpressure is automatic
- [ ] A global `unhandledrejection` handler as a backstop, treated as a bug report
- [ ] Loading, empty, error, and offline states all designed — not just the happy path

## Error Handling

- [ ] Catch only where you can add information or make a decision
- [ ] Never an empty `catch` — at minimum, log with context
- [ ] Wrap with `cause` when crossing a layer boundary: `new Error(msg, { cause: err })`
- [ ] Custom error classes with `this.name = this.constructor.name`
- [ ] Throw `Error` instances, never strings — a string has no stack
- [ ] Distinguish expected failures (return them) from exceptional ones (throw them)
- [ ] Validation errors collected and reported together, not one per submission
- [ ] User-facing messages say what to do next; technical detail goes to the log
- [ ] `finally` for cleanup that must run on both paths
- [ ] Log the serialised message and stack, not the error object, in long-lived buffers
- [ ] Errors reported to a tracker with source maps uploaded
- [ ] Error boundaries at route or component level, so one failure doesn't blank the page
- [ ] Don't leak internal detail — stack traces and SQL never reach the client

## Modules

- [ ] ESM everywhere; CommonJS only where a runtime forces it
- [ ] Named exports by default; default exports only for a single-thing module
- [ ] No side effects at module top level beyond definitions
- [ ] `sideEffects: false` in `package.json` when true, so tree shaking works
- [ ] Import the specific submodule, not the whole library
- [ ] No circular dependencies — extract the shared piece into a third module
- [ ] Public API declared with the `exports` field, so deep imports can't couple to internals
- [ ] Dynamic `import()` for route- and interaction-level code splitting
- [ ] Top-level `await` only at entry points, never in a widely-imported leaf
- [ ] Feature-based folders, with a lint rule preventing cross-feature imports

## DOM

- [ ] `textContent` by default; `innerHTML` requires a justification in review
- [ ] One delegated listener per list, not one per row
- [ ] Every listener registered with an `AbortController` signal, aborted on teardown
- [ ] `{ passive: true }` on `scroll`, `wheel`, `touchstart`, and `touchmove`
- [ ] All layout reads batched before all writes
- [ ] Animate only `transform` and `opacity`
- [ ] Build with `createElement` and `append`; strings appended become text nodes
- [ ] Many nodes inserted via a `DocumentFragment` or `replaceChildren`, so layout runs once
- [ ] `width` and `height` on every image, to prevent layout shift
- [ ] Observers (`Intersection`, `Resize`, `Mutation`) rather than polling or scroll handlers
- [ ] `requestAnimationFrame` for visual updates; never `setInterval(fn, 16)`
- [ ] Avoid `stopPropagation` — it breaks other people's listeners
- [ ] `closest()` for delegation targets, not `matches()` on `e.target`
- [ ] `pushState` always paired with a render call — it doesn't fire `popstate`

## Performance

- [ ] Measured before optimised, and measured again after
- [ ] A performance budget in CI that fails the build: bundle size, LCP, INP
- [ ] `defer` or `type="module"` on scripts; never a blocking `<script>` in `<head>`
- [ ] Critical CSS inlined, the rest deferred
- [ ] Route- and interaction-level code splitting
- [ ] `preload` used sparingly — everything preloaded is nothing preloaded
- [ ] `fetchpriority="high"` on the LCP image; `loading="lazy"` below the fold
- [ ] Modern image formats (AVIF, WebP) with correct dimensions
- [ ] No long tasks over 50ms; chunk with `scheduler.yield()` or move to a worker
- [ ] CPU-bound work over 50ms in a Web Worker, with buffers transferred not cloned
- [ ] Long lists virtualised, or at minimum `content-visibility: auto`
- [ ] Complexity checked before micro-optimising — no `includes` inside `filter`
- [ ] Object shapes fixed at construction in profiled hot paths
- [ ] Tested with 4× CPU throttling and Fast 3G, not just on your machine
- [ ] Field data (RUM) monitored, not only Lighthouse

## Memory

- [ ] Every listener, timer, observer, and subscription removed on teardown
- [ ] One `AbortController` per component lifetime, aborted in one place
- [ ] `clearInterval`/`clearTimeout` for every timer created
- [ ] `WeakMap` for data keyed by objects you don't own the lifetime of
- [ ] Every cache bounded by size or TTL
- [ ] No DOM nodes stored in long-lived arrays or `Map`s
- [ ] Accumulating structures bounded: logs, undo stacks, message history, chart buffers
- [ ] Error objects not retained; store the message and stack strings
- [ ] No required cleanup in a `FinalizationRegistry` — it may never run
- [ ] Heap snapshots taken around repeated actions before shipping a long-lived view

## Security

- [ ] No `innerHTML`, `outerHTML`, or `insertAdjacentHTML` with interpolated values
- [ ] No `eval`, `new Function`, or string-form `setTimeout`/`setInterval`
- [ ] User-supplied markup sanitised with a real parser (`Sanitizer`, DOMPurify) — never a regex
- [ ] Escaping matched to context, and attributes always quoted
- [ ] Every URL assigned to `href`/`src`/`action` checked against a protocol allowlist
- [ ] `rel="noopener noreferrer"` on every `target="_blank"`
- [ ] A CSP deployed, with no `unsafe-inline` or `unsafe-eval`
- [ ] `object-src 'none'`, `base-uri 'none'`, `form-action 'self'`, `frame-ancestors` all set
- [ ] Per-response CSP nonce, with `'strict-dynamic'`
- [ ] Auth tokens in `HttpOnly; Secure; SameSite=Lax` cookies, never `localStorage`
- [ ] CSRF token plus `Origin` check on every state-changing request
- [ ] Recursive merges reject `__proto__`, `constructor`, and `prototype`
- [ ] `crypto.getRandomValues` / `crypto.randomUUID` for anything security-relevant
- [ ] No secrets in client code, including build-time environment variables
- [ ] Lockfile committed, `npm ci` in CI, and lockfile diffs reviewed
- [ ] Server-side validation for everything — the client's checks are a UX feature

## Testing

- [ ] Test names state the behaviour, so a failure reads as a sentence
- [ ] Every bug fix ships with a regression test that failed first
- [ ] Mocks only at boundaries you don't own: network, clock, randomness, filesystem
- [ ] Network mocked at the HTTP layer (MSW), with unhandled requests failing the test
- [ ] Async assertions awaited — `await expect(p).rejects.toThrow()`
- [ ] `expect.assertions(n)` when using `try`/`catch` to assert a rejection
- [ ] Fake timers used with their async variants, so microtasks drain
- [ ] Fresh state per test via `beforeEach`; no order dependence
- [ ] Queries by role and label, not test id
- [ ] `userEvent` rather than `fireEvent`
- [ ] No fixed sleeps — wait for a condition
- [ ] Snapshots small and inline, so a diff is reviewable
- [ ] Edge cases covered: empty, one, many, null, negative, huge
- [ ] Coverage read as a signal, with a floor on changed files rather than a global target

## Accessibility

- [ ] Semantic elements first — `<button>`, not a `<div>` with a click handler
- [ ] Every input has an associated `<label for>`; a placeholder is not a label
- [ ] Keyboard-operable end to end, with a visible focus indicator
- [ ] Focus order follows visual order; no positive `tabindex`
- [ ] Modals trap focus, close on Escape, and restore focus to the trigger
- [ ] Errors linked with `aria-describedby` and announced via a live region
- [ ] `aria-invalid` set on failing fields
- [ ] Images have meaningful `alt`, or `alt=""` when decorative
- [ ] Colour is never the only signal of state
- [ ] Contrast meets WCAG AA — 4.5:1 for body text
- [ ] `prefers-reduced-motion` respected
- [ ] Established ARIA patterns followed for custom widgets, not invented ones
- [ ] Tested with an actual screen reader, not just an automated checker

## Code Review

- [ ] Read the whole diff before commenting on any of it
- [ ] Distinguish blocking from optional — say which each comment is
- [ ] Ask rather than assert when you might be missing context
- [ ] Comment on the code, never the author
- [ ] Explain the reasoning, so the lesson transfers beyond this line
- [ ] Approve when it's better than what's there, not when it's perfect
- [ ] Flag missing tests for the specific behaviour changed
- [ ] Check the error and empty paths, which are where bugs actually hide
- [ ] Watch for the four leak shapes: listeners, timers, unbounded caches, retained nodes
- [ ] Question every new dependency's cost
- [ ] Pull the branch and run it for anything non-trivial — reading isn't reviewing

## Production Readiness

- [ ] Error tracking wired up, with source maps uploaded and release tagged
- [ ] Real-user performance monitoring for LCP, INP, and CLS
- [ ] A visible, actionable error state for every failure path
- [ ] Offline handled — detected, communicated, and recovered from
- [ ] Loading states for anything over 200ms; skeletons over spinners where the shape is known
- [ ] No `console.log` left in shipped code; a real logger with levels instead
- [ ] Feature flags for anything risky, with a documented kill switch
- [ ] Rollback tested, not assumed
- [ ] Environment configuration validated at startup, failing loudly if wrong
- [ ] Health check and version endpoint, so you know what's deployed
- [ ] Security headers set: CSP, HSTS, `X-Content-Type-Options`, `Referrer-Policy`
- [ ] Rate limiting on anything a user can trigger repeatedly
- [ ] A runbook naming the owner, the dashboards, and the three most likely failures
- [ ] Dependencies audited and updated on a schedule, not on an incident

---

## Using This as a Reviewer

Three questions catch more real problems than reading line by line:

**"What happens when this fails?"** Most code is written for the happy path. Ask about the network
being down, the array being empty, the value being `null`, and two of these happening at once.

**"Who cleans this up?"** For every listener, timer, subscription, and cache entry added, find the
line that removes it. If there isn't one, that's the finding.

**"How would I know this broke in production?"** If the answer is "a user tells us", the change needs
instrumentation more than it needs a code comment.

---

**Next:** [Appendix F — Cheat Sheet →](F-cheatsheet.md)
