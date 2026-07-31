# Appendix G — Glossary

Every term used in this book, alphabetical, with a chapter cross-reference where one applies.

[A](#a) · [B](#b) · [C](#c) · [D](#d) · [E](#e) · [F](#f) · [G](#g) · [H](#h) · [I](#i) ·
[J](#j) · [K](#k) · [L](#l) · [M](#m) · [N](#n) · [O](#o) · [P](#p) · [Q](#q) · [R](#r) ·
[S](#s) · [T](#t) · [U](#u) · [V](#v) · [W](#w) · [X](#x) · [Y](#y)

---

## A

**AbortController** — An object exposing a `signal` you pass to cancellable operations. Calling
`abort()` rejects cooperating operations with an `AbortError`. The modern mechanism for cleaning up
listeners, timers, and requests together. → [Ch 22](../chapters/22-events.md),
[Ch 24](../chapters/24-timers-scheduling.md)

**Abstract equality** — The coercing comparison performed by `==`, following a six-clause algorithm
that converts operands before comparing. Source of `[] == false` being `true`.
→ [Ch 2](../chapters/02-variables-types-operators.md)

**Accessible name** — The label assistive technology announces for an element, computed from its
content, `aria-label`, or associated `<label>`. Querying by it in tests catches accessibility bugs
for free. → [Ch 23](../chapters/23-forms-validation.md), [Ch 32](../chapters/32-testing.md)

**ADR (Architecture Decision Record)** — A short document recording a decision, its reasoning, and
the alternatives rejected, so future maintainers can respect or knowingly overturn it.
→ [App A §12](A-interview-questions.md#12-patterns-and-architecture)

**Adapter** — A wrapper making an incompatible interface fit the one your code expects. The seam
that lets a dependency be swapped or mocked. → [Ch 30](../chapters/30-design-patterns.md)

**AggregateError** — An error holding an array of errors in its `errors` property. Thrown by
`Promise.any` when every input rejects. → [Ch 19](../chapters/19-async-javascript.md)

**Allocation rate** — How fast a program creates objects. High rates cause frequent minor GCs; it's
a distinct problem from retention and needs a different tool to diagnose.
→ [Ch 27](../chapters/27-memory-garbage-collection.md)

**ArrayBuffer** — A fixed-length block of raw binary data, read and written through typed array
views. Transferable between threads at zero cost.
→ [Ch 25](../chapters/25-storage-browser-apis.md)

**Arity** — The number of parameters a function declares, readable as `fn.length`. Stops counting at
the first default or rest parameter, which breaks naive `curry` implementations.
→ [Ch 4](../chapters/04-functions.md), [Ch 29](../chapters/29-functional-programming.md)

**Arrow function** — A function expression with lexically-inherited `this`, no `arguments`, and no
`prototype` — so it cannot be used with `new`. → [Ch 4](../chapters/04-functions.md),
[Ch 10](../chapters/10-this-and-binding.md)

**AST (Abstract Syntax Tree)** — The tree representation a parser produces from source code. What
bundlers, linters, and transpilers operate on. → [Ch 1](../chapters/01-how-javascript-works.md)

**Async generator** — A function declared `async function*` whose `yield`s can await. Consumed with
`for await...of`, giving backpressure for free.
→ [Ch 14](../chapters/14-symbols-iterators-generators.md)

**Autoboxing** — The engine temporarily wrapping a primitive in its object equivalent so a method
can be called on it, as in `'abc'.toUpperCase()`.
→ [Ch 7](../chapters/07-strings-numbers-math-dates.md)

## B

**Backpressure** — A consumer signalling it can't keep up, so the producer slows rather than
buffering without bound. Async iterators provide it inherently; streams expose it explicitly.
→ [Ch 20](../chapters/20-fetch-json-http.md)

**Backtracking** — A regex engine retrying alternative match paths after a failure. With nested
quantifiers the number of paths is exponential, which is the ReDoS vulnerability.
→ [Ch 26](../chapters/26-regular-expressions.md)

**BigInt** — A primitive for arbitrary-precision integers, written with an `n` suffix. Cannot be
mixed with `number` in arithmetic. → [Ch 7](../chapters/07-strings-numbers-math-dates.md)

**Binding** — The association between an identifier and a storage location in a scope. Closures
capture bindings, not values, which is why a `var` loop counter surprises people.
→ [Ch 8](../chapters/08-scope-hoisting-tdz.md), [Ch 9](../chapters/09-closures.md)

**Block scope** — The scope created by a `{ }` block, in which `let`, `const`, and `class`
declarations live. `var` ignores it. → [Ch 8](../chapters/08-scope-hoisting-tdz.md)

**Brand check** — Testing whether an object was constructed by a particular class, using
`#field in obj`. Unforgeable, unlike a duck-typed check.
→ [Ch 12](../chapters/12-classes-oop.md)

**BroadcastChannel** — An API for messaging between tabs, windows, and workers of the same origin.
Preferable to the `storage` event for cross-tab coordination.
→ [Ch 25](../chapters/25-storage-browser-apis.md)

**browserslist** — A shared configuration declaring which browsers a project supports. Drives
transpilation targets, polyfill injection, and CSS prefixing.
→ [Ch 33](../chapters/33-es6-to-es2025.md)

**Bubbling** — The third event propagation phase, travelling from the target back up to `window`.
The default phase for listeners. → [Ch 22](../chapters/22-events.md)

**Bundler** — A tool combining modules into fewer files, resolving imports and eliminating unused
exports. Vite, webpack, Rollup, esbuild. → [Ch 17](../chapters/17-modules.md)

## C

**Cache API** — Browser storage for `Request`/`Response` pairs, designed for service workers to
serve assets offline. → [Ch 25](../chapters/25-storage-browser-apis.md)

**Call stack** — The structure tracking which functions are executing and where each returns.
Exceeding its depth throws `RangeError`. → [Ch 18](../chapters/18-event-loop.md)

**Capture phase** — The first event propagation phase, travelling from `window` down to the target's
parent. Opt in with `{ capture: true }`. → [Ch 22](../chapters/22-events.md)

**Chain of Responsibility** — A pattern where each handler either processes a request or passes it
along. Middleware is its most common form. → [Ch 30](../chapters/30-design-patterns.md)

**Circular dependency** — Two or more modules importing each other. Legal in ESM, since bindings are
created before evaluation, but throws if a `const` is read before its module has evaluated.
→ [Ch 17](../chapters/17-modules.md)

**CLS (Cumulative Layout Shift)** — A Core Web Vital measuring unexpected movement of visible
content. Missing image dimensions is the largest single cause.
→ [Ch 28](../chapters/28-performance.md)

**Closure** — A function together with a reference to the lexical environment where it was created,
retaining access to those variables after the enclosing function returns.
→ [Ch 9](../chapters/09-closures.md)

**Coercion** — Implicit type conversion during an operation, such as `'5' - 2` producing `3`.
→ [Ch 2](../chapters/02-variables-types-operators.md)

**Command** — A pattern representing an operation as an object, typically with `do` and `undo`.
What makes undo stacks and replayable action logs possible.
→ [Ch 30](../chapters/30-design-patterns.md)

**CommonJS** — Node's original module system, using `require` and `module.exports`. Synchronous,
runtime-resolved, and copies values rather than creating live bindings.
→ [Ch 17](../chapters/17-modules.md)

**Composition** — Building behaviour by combining independent pieces rather than inheriting from a
hierarchy. Generally preferable to inheritance in JavaScript.
→ [Ch 12](../chapters/12-classes-oop.md), [Ch 29](../chapters/29-functional-programming.md)

**Compositing** — The final rendering step, where painted layers are combined. `transform` and
`opacity` changes are composited without layout or paint, which is why they animate cheaply.
→ [Ch 21](../chapters/21-dom.md), [Ch 28](../chapters/28-performance.md)

**Computed property** — An object key derived from an expression, written `{ [expr]: value }`.
→ [Ch 5](../chapters/05-objects.md)

**Constraint Validation API** — The browser's built-in form validation: `required`, `pattern`,
`checkValidity()`, `setCustomValidity()`. → [Ch 23](../chapters/23-forms-validation.md)

**Content-visibility** — A CSS property that skips rendering work for off-screen elements. The
cheapest first attempt at a long-list performance problem.
→ [Ch 28](../chapters/28-performance.md)

**Core Web Vitals** — Google's user-experience metrics: LCP, INP, and CLS.
→ [Ch 28](../chapters/28-performance.md)

**CORS (Cross-Origin Resource Sharing)** — Headers letting a server permit browsers to *read*
cross-origin responses. Not an authorisation mechanism, and irrelevant outside a browser.
→ [Ch 20](../chapters/20-fetch-json-http.md), [Ch 31](../chapters/31-security.md)

**CSP (Content Security Policy)** — A header allowlisting what a page may load and execute. Defence
in depth against XSS, defeated entirely by `unsafe-inline`.
→ [Ch 31](../chapters/31-security.md)

**CSRF (Cross-Site Request Forgery)** — Tricking a victim's browser into sending an authenticated
request they didn't intend, exploiting automatic cookie attachment. Mitigated by `SameSite` and a
token. → [Ch 31](../chapters/31-security.md)

**Currying** — Transforming an n-ary function into a chain of unary functions, so `f(a, b)` becomes
`f(a)(b)`. → [Ch 29](../chapters/29-functional-programming.md)

## D

**Debounce** — Delaying a function until a quiet period has elapsed since the last call. Fires once,
after the calls stop. Right for search-as-you-type.
→ [Ch 24](../chapters/24-timers-scheduling.md)

**Deep clone** — A copy duplicating the entire object graph rather than sharing nested references.
`structuredClone` handles cycles, `Map`, `Set`, and `Date`; the JSON round-trip doesn't.
→ [Ch 5](../chapters/05-objects.md), [Ch 25](../chapters/25-storage-browser-apis.md)

**Decorator** — A wrapper adding behaviour while preserving the interface — retry, cache, log.
Composition order is semantically significant. → [Ch 30](../chapters/30-design-patterns.md)

**Delegation** — (1) Attaching one event listener to an ancestor to handle many descendants.
(2) The prototype chain's mechanism for property lookup.
→ [Ch 11](../chapters/11-prototypes-inheritance.md), [Ch 22](../chapters/22-events.md)

**Dependency injection** — Passing collaborators into a module rather than importing them, making
dependencies visible and tests mock-free. → [Ch 30](../chapters/30-design-patterns.md)

**Descriptor** — The metadata behind a property: `value`, `writable`, `enumerable`,
`configurable`, or `get`/`set`. Assignment defaults to all-true; `defineProperty` to all-false.
→ [Ch 5](../chapters/05-objects.md)

**Detached node** — A DOM node removed from the document but still referenced from JavaScript, so it
and its whole subtree remain uncollectable. One of the four classic leaks.
→ [Ch 27](../chapters/27-memory-garbage-collection.md)

**Dictionary mode** — A slower object representation V8 falls back to after operations like `delete`
that make the hidden class unusable. → [Ch 27](../chapters/27-memory-garbage-collection.md),
[Ch 28](../chapters/28-performance.md)

**DOM (Document Object Model)** — The tree of objects representing a document, and the API for
reading and modifying it. Provided by the host, not the language.
→ [Ch 21](../chapters/21-dom.md)

**DocumentFragment** — A lightweight container for nodes, not part of the document. Building in one
and inserting once means layout runs once instead of per node.
→ [Ch 21](../chapters/21-dom.md)

## E

**ECMAScript** — The standardised specification JavaScript implements, maintained by TC39. Editions
are named by ratification year: ES2015 (ES6) onward are annual.
→ [Ch 1](../chapters/01-how-javascript-works.md), [Ch 33](../chapters/33-es6-to-es2025.md)

**Engine** — The program that parses and executes JavaScript: V8, SpiderMonkey, JavaScriptCore. It
provides the language; the runtime provides the APIs.
→ [Ch 1](../chapters/01-how-javascript-works.md)

**ESM (ECMAScript Modules)** — The standard module system, using `import` and `export`. Static,
asynchronous, tree-shakeable, with live bindings. → [Ch 17](../chapters/17-modules.md)

**Event delegation** — See **Delegation**.

**Event loop** — The mechanism that runs one macrotask, drains all microtasks, optionally renders,
and repeats — as long as the call stack is empty. → [Ch 18](../chapters/18-event-loop.md)

**Execution context** — The internal record of a running function: its variable environment, scope
chain, and `this`. Pushed onto the call stack.
→ [Ch 8](../chapters/08-scope-hoisting-tdz.md)

**Exports field** — The `package.json` key declaring a package's public entry points, encapsulating
everything else and enabling conditional resolution.
→ [Ch 17](../chapters/17-modules.md)

## F

**Facade** — A simplified interface over a complex subsystem.
→ [Ch 30](../chapters/30-design-patterns.md)

**Factory** — A function that returns objects, deciding the concrete type or performing validation
callers shouldn't repeat. → [Ch 30](../chapters/30-design-patterns.md)

**Fake** — A working lightweight implementation of a collaborator, such as an in-memory database.
Usually preferable to a mock. → [Ch 32](../chapters/32-testing.md)

**Fake timers** — A test facility replacing timer functions so time can be advanced instantly. Their
async variants are needed for anything involving promises.
→ [Ch 32](../chapters/32-testing.md)

**FinalizationRegistry** — An API registering a callback to run after an object is collected. Carries
no guarantee of ever running, so it must never hold required cleanup.
→ [Ch 27](../chapters/27-memory-garbage-collection.md)

**Fisher-Yates** — The provably uniform shuffle algorithm. `sort(() => Math.random() - 0.5)` is not
uniform and is not a substitute. → [App B](B-coding-challenges.md)

**Forced synchronous layout** — When reading a layout property after a write compels the browser to
recalculate layout immediately. Interleaving reads and writes in a loop is layout thrashing.
→ [Ch 21](../chapters/21-dom.md), [Ch 28](../chapters/28-performance.md)

**Frame budget** — The time available per frame before a paint is dropped: roughly 16.7ms at 60Hz,
of which about 10ms is yours. → [Ch 24](../chapters/24-timers-scheduling.md)

**Functor** — Any type with a `map` obeying the identity and composition laws. `Array` and
`Promise` both qualify. → [Ch 29](../chapters/29-functional-programming.md)

## G

**Garbage collection** — Automatic reclamation of memory unreachable from a GC root. V8 uses a
generational mark-and-sweep collector.
→ [Ch 27](../chapters/27-memory-garbage-collection.md)

**GC root** — A starting point for reachability analysis: the global object, the call stack and its
scope chain, and internal engine references.
→ [Ch 27](../chapters/27-memory-garbage-collection.md)

**Generational GC** — Splitting the heap by object age, collecting the young generation frequently
and cheaply and the old generation rarely. Makes short-lived garbage nearly free.
→ [Ch 27](../chapters/27-memory-garbage-collection.md)

**Generator** — A function declared `function*` that can pause at `yield` and resume, preserving its
entire state. Both an iterator and an iterable.
→ [Ch 14](../chapters/14-symbols-iterators-generators.md)

**globalThis** — The standard name for the global object across all environments, replacing
`window`, `global`, and `self`. → [Ch 33](../chapters/33-es6-to-es2025.md)

**Grapheme cluster** — What a user perceives as one character, which may span several code points —
`é` as `e` plus a combining accent. Segmented with `Intl.Segmenter`.
→ [Ch 7](../chapters/07-strings-numbers-math-dates.md)

## H

**Heap** — The memory region where objects, arrays, functions, and closures live. The stack holds
references into it. → [Ch 27](../chapters/27-memory-garbage-collection.md)

**Heap snapshot** — A capture of the object graph, used to find leaks by comparing two snapshots and
reading the retainer chain of anything that grew.
→ [Ch 27](../chapters/27-memory-garbage-collection.md)

**Hidden class** — V8's internal shape descriptor for an object, enabling property access at a fixed
offset instead of a hash lookup. Shared by objects with identical shape.
→ [Ch 28](../chapters/28-performance.md)

**Hoisting** — Declarations being processed before execution. `var` and `function` are initialised;
`let`, `const`, and `class` are not, leaving them in the TDZ.
→ [Ch 8](../chapters/08-scope-hoisting-tdz.md)

**Host** — The environment embedding an engine and supplying APIs beyond the language: a browser,
Node, Deno. → [Ch 1](../chapters/01-how-javascript-works.md)

**HttpOnly** — A cookie attribute making the cookie invisible to JavaScript, so an XSS cannot read
it. The reason auth tokens belong in cookies. → [Ch 31](../chapters/31-security.md)

**Hydration** — Attaching event listeners and client state to server-rendered markup. A common
source of long tasks and therefore poor INP. → [Ch 28](../chapters/28-performance.md)

## I

**Idempotent** — An operation producing the same result whether applied once or many times. A
prerequisite for safe retries. → [Ch 20](../chapters/20-fetch-json-http.md)

**IIFE (Immediately Invoked Function Expression)** — `(function () {})()`. Created a private scope
before block scoping and modules existed. → [Ch 4](../chapters/04-functions.md)

**Immutability** — Never modifying a value after creation, producing new values instead. Enables
identity-based change detection and safe sharing.
→ [Ch 29](../chapters/29-functional-programming.md)

**Import attributes** — Syntax declaring a module's type: `with { type: 'json' }`. A security
requirement, not a convenience. → [Ch 33](../chapters/33-es6-to-es2025.md)

**IndexedDB** — The browser's asynchronous, transactional, indexed database. Stores structured
clones, and its transactions auto-commit when the microtask queue drains.
→ [Ch 25](../chapters/25-storage-browser-apis.md)

**Inline cache** — An engine optimisation remembering the shape seen at a property-access site.
Monomorphic sites are fastest; megamorphic sites abandon the cache.
→ [Ch 28](../chapters/28-performance.md)

**INP (Interaction to Next Paint)** — A Core Web Vital measuring worst-case interaction latency from
input to the resulting paint. Replaced FID in 2024.
→ [Ch 28](../chapters/28-performance.md)

**IntersectionObserver** — An API reporting when an element enters or leaves a viewport or ancestor.
Replaces scroll handlers for lazy loading and infinite scroll.
→ [Ch 25](../chapters/25-storage-browser-apis.md)

**Iterable** — An object with a `[Symbol.iterator]` method, usable with `for...of` and spread.
→ [Ch 14](../chapters/14-symbols-iterators-generators.md)

**Iterator** — An object with a `next()` method returning `{ value, done }`.
→ [Ch 14](../chapters/14-symbols-iterators-generators.md)

**Iterator helpers** — Lazy `map`, `filter`, `take`, and friends on any iterator, added in ES2025.
Work on infinite sequences and allocate no intermediates.
→ [Ch 33](../chapters/33-es6-to-es2025.md)

## J

**JIT (Just-In-Time compilation)** — Compiling code at runtime, typically after interpreting it and
identifying hot paths. Why microbenchmarks need a warm-up.
→ [Ch 1](../chapters/01-how-javascript-works.md), [Ch 28](../chapters/28-performance.md)

**JSON** — A text data format, and the `JSON.parse`/`stringify` API. Loses `undefined`, functions,
symbols, `Map`, `Set`, and `Date` fidelity; throws on cycles.
→ [Ch 20](../chapters/20-fetch-json-http.md)

## K

**Key enumeration order** — Integer-like keys ascending, then string keys in insertion order, then
symbol keys in insertion order. Specified since ES2020.
→ [Ch 5](../chapters/05-objects.md)

## L

**Layout** — The rendering step computing element geometry. Also called reflow. Triggered by
structural or size changes, and forced by reading geometry properties.
→ [Ch 21](../chapters/21-dom.md)

**Layout thrashing** — Repeatedly forcing synchronous layout by interleaving reads and writes. Fixed
by batching all reads before all writes. → [Ch 28](../chapters/28-performance.md)

**LCP (Largest Contentful Paint)** — A Core Web Vital measuring when the largest above-fold element
renders. Driven by server response time and render-blocking resources.
→ [Ch 28](../chapters/28-performance.md)

**Lexical environment** — The record holding a scope's bindings plus a reference to its parent
environment. What a closure captures. → [Ch 8](../chapters/08-scope-hoisting-tdz.md)

**Lexical scope** — Scope determined by where code is written, not where it is called.
→ [Ch 8](../chapters/08-scope-hoisting-tdz.md)

**Live binding** — An ESM import referencing the exporter's binding rather than copying its value,
so reassignment by the exporter is visible to importers.
→ [Ch 17](../chapters/17-modules.md)

**Live collection** — A DOM collection that updates automatically as the document changes, such as
`getElementsByClassName`'s result. `querySelectorAll` returns a static list instead.
→ [Ch 21](../chapters/21-dom.md)

**Long task** — Main-thread work exceeding 50ms, during which nothing responds. The dominant cause
of poor INP. → [Ch 28](../chapters/28-performance.md)

**LRU (Least Recently Used)** — A cache eviction policy discarding the least recently accessed
entry. Implementable in a few lines with a `Map`, whose insertion order is recency order.
→ [Ch 15](../chapters/15-maps-sets-weak-collections.md)

## M

**Macrotask** — A unit of work from the task queue: a timer callback, an I/O completion, an event
handler. One runs per event loop turn. → [Ch 18](../chapters/18-event-loop.md)

**Mark-and-sweep** — The garbage collection algorithm that marks everything reachable from the roots
and frees the rest. Handles reference cycles, which reference counting cannot.
→ [Ch 27](../chapters/27-memory-garbage-collection.md)

**Megamorphic** — A property-access site that has seen five or more object shapes, causing the
engine to abandon its inline cache. → [Ch 28](../chapters/28-performance.md)

**Memoisation** — Caching a pure function's results by argument. Requires purity to be correct and a
bound to avoid leaking. → [Ch 29](../chapters/29-functional-programming.md)

**Microtask** — A unit of work from the microtask queue — promise reactions, `queueMicrotask`,
`MutationObserver` callbacks. The queue drains completely between macrotasks and before rendering.
→ [Ch 18](../chapters/18-event-loop.md)

**Middleware** — A composable function receiving a context and a `next` callback, forming a Chain of
Responsibility. → [Ch 30](../chapters/30-design-patterns.md)

**Minification** — Reducing code size by shortening names and removing whitespace, without changing
behaviour. Distinct from transpilation. → [Ch 28](../chapters/28-performance.md)

**Mixin** — A function taking a base class and returning a subclass with added behaviour, allowing
behaviour composition within single inheritance. → [Ch 12](../chapters/12-classes-oop.md)

**Mock** — A test double that both stubs behaviour and asserts on how it was called. Over-mocking
produces suites that pass while the application is broken.
→ [Ch 32](../chapters/32-testing.md)

**Monad** — A type with `of` and `flatMap` obeying certain laws, letting you chain operations that
each wrap their result. `Promise` is the one every JavaScript developer already uses.
→ [Ch 29](../chapters/29-functional-programming.md)

**Monomorphic** — A property-access site that has only ever seen one object shape, and is therefore
optimally cached. → [Ch 28](../chapters/28-performance.md)

**MutationObserver** — An API reporting DOM changes as batched records. Used to react to markup you
don't control. → [Ch 25](../chapters/25-storage-browser-apis.md)

## N

**NaN** — The result of an invalid numeric operation, and the only value not equal to itself. Test
with `Number.isNaN`, never the coercing global `isNaN`.
→ [Ch 7](../chapters/07-strings-numbers-math-dates.md)

**Nonce** — A single-use random value. In CSP, a per-response token permitting specific inline
scripts. A static nonce is `unsafe-inline` with extra steps.
→ [Ch 31](../chapters/31-security.md)

**Nullish coalescing** — The `??` operator, falling back only for `null` and `undefined` — unlike
`||`, which falls back on any falsy value.
→ [Ch 13](../chapters/13-destructuring-spread-rest.md)

## O

**Observer** — A pattern where a subject notifies registered dependents of changes. `addEventListener`
is one; PubSub is the broker-mediated variant.
→ [Ch 30](../chapters/30-design-patterns.md)

**Optional chaining** — The `?.` operator, short-circuiting to `undefined` when the left side is
`null` or `undefined`. Works on property access, indexing, and calls.
→ [Ch 13](../chapters/13-destructuring-spread-rest.md)

**Origin** — The tuple of scheme, host, and port that defines a security boundary. Same-origin
policy and CORS are both defined in terms of it.
→ [Ch 20](../chapters/20-fetch-json-http.md)

**Overscan** — Rendering a few extra rows beyond the visible window in a virtualised list, so fast
scrolling doesn't reveal blanks. → [Ch 28](../chapters/28-performance.md)

## P

**Paint** — The rendering step that fills in pixels. Cheaper than layout, and skipped entirely for
composited properties. → [Ch 21](../chapters/21-dom.md)

**Partial application** — Fixing some of a function's arguments now and supplying the rest later.
Distinct from currying, which insists on one argument at a time.
→ [Ch 29](../chapters/29-functional-programming.md)

**Passive listener** — A listener declared `{ passive: true }`, promising not to call
`preventDefault()` so the browser needn't wait for it before scrolling.
→ [Ch 22](../chapters/22-events.md)

**PerformanceObserver** — An API delivering performance entries — long tasks, LCP, layout shifts.
Use `buffered: true` to receive entries recorded before it was created.
→ [Ch 28](../chapters/28-performance.md)

**Point-free style** — Composing functions without naming their arguments. Clear until it requires
`flip` or `apply`, at which point it isn't.
→ [Ch 29](../chapters/29-functional-programming.md)

**Polyfill** — Code implementing a missing API at runtime. Cannot fix missing *syntax*, which must be
transpiled. → [Ch 33](../chapters/33-es6-to-es2025.md)

**Polymorphic** — A property-access site that has seen two to four object shapes: slower than
monomorphic, faster than megamorphic. → [Ch 28](../chapters/28-performance.md)

**Primitive** — One of the seven non-object types: `string`, `number`, `boolean`, `null`,
`undefined`, `symbol`, `bigint`. Immutable, and compared by value.
→ [Ch 2](../chapters/02-variables-types-operators.md)

**Promise** — An object representing a future value, in one of three states: pending, fulfilled, or
rejected. Settles exactly once, irreversibly.
→ [Ch 19](../chapters/19-async-javascript.md)

**Prototype** — The object another object delegates failed property lookups to. Also the `prototype`
property of a constructor function, which instances link to.
→ [Ch 11](../chapters/11-prototypes-inheritance.md)

**Prototype chain** — The sequence of prototypes traversed during property lookup, ending at `null`.
→ [Ch 11](../chapters/11-prototypes-inheritance.md)

**Prototype pollution** — Writing to `Object.prototype` so a property appears on every object,
usually via a recursive merge over untrusted JSON containing `__proto__`.
→ [Ch 31](../chapters/31-security.md)

**Proxy** — An object wrapping another whose fundamental operations can be intercepted with traps.
The basis of framework reactivity, at a real performance cost.
→ [Ch 5](../chapters/05-objects.md)

**Pure function** — One that is deterministic for the same inputs and has no side effects. Both
halves are required. → [Ch 29](../chapters/29-functional-programming.md)

## Q

**queueMicrotask** — Schedules a callback as a microtask without allocating a promise. A throw inside
becomes an uncaught error rather than an unhandled rejection.
→ [Ch 18](../chapters/18-event-loop.md), [Ch 24](../chapters/24-timers-scheduling.md)

**Quota** — The storage limit a browser imposes per origin. Exceeding it throws
`QuotaExceededError`. → [Ch 25](../chapters/25-storage-browser-apis.md)

## R

**Reachability** — Whether a value can be reached by following references from a GC root. The only
criterion for whether it is garbage. → [Ch 27](../chapters/27-memory-garbage-collection.md)

**Realm** — An isolated global environment with its own set of intrinsics. An array from an iframe is
a different realm's `Array`, which is why `instanceof` fails and `Array.isArray` exists.
→ [Ch 5](../chapters/05-objects.md)

**ReDoS (Regular expression Denial of Service)** — Hanging a thread by feeding input that triggers
exponential backtracking in a vulnerable pattern.
→ [Ch 26](../chapters/26-regular-expressions.md), [Ch 31](../chapters/31-security.md)

**Reference counting** — A memory management strategy tracking how many references point at each
object. Cannot reclaim cycles, which is why JavaScript uses reachability instead.
→ [Ch 27](../chapters/27-memory-garbage-collection.md)

**Referential transparency** — The property that an expression can be replaced by its value without
changing the program. What makes memoisation always safe.
→ [Ch 29](../chapters/29-functional-programming.md)

**Reflect** — A namespace of methods mirroring the internal object operations. Proxy traps should
delegate through it to preserve correct receiver semantics.
→ [Ch 5](../chapters/05-objects.md)

**Reflow** — See **Layout**.

**Repository** — A pattern centralising data access, so a schema change has one blast radius.
→ [Ch 30](../chapters/30-design-patterns.md)

**requestAnimationFrame** — Schedules a callback immediately before the next paint, at the display's
refresh rate. Does not run in a hidden tab.
→ [Ch 24](../chapters/24-timers-scheduling.md)

**requestIdleCallback** — Schedules low-priority work for idle periods, with a deadline object the
callback should respect. → [Ch 24](../chapters/24-timers-scheduling.md)

**ResizeObserver** — An API reporting element size changes without a `resize` event.
→ [Ch 25](../chapters/25-storage-browser-apis.md)

**Rest parameter** — `...args` in a parameter list, collecting remaining arguments into a real array.
→ [Ch 13](../chapters/13-destructuring-spread-rest.md)

**Result** — A value representing either success or failure, making error handling visible in the
return type. Preferable to exceptions for expected failures.
→ [Ch 29](../chapters/29-functional-programming.md)

**Retainer** — A reference keeping an object reachable. The retainer chain in a heap snapshot is the
answer to why something hasn't been collected.
→ [Ch 27](../chapters/27-memory-garbage-collection.md)

**Runtime** — The host environment: the engine plus its APIs, event loop, and module resolution.
Chrome and Node are runtimes; V8 is the engine both embed.
→ [Ch 1](../chapters/01-how-javascript-works.md)

## S

**SameSite** — A cookie attribute controlling whether it is sent on cross-site requests. `Lax` is
the modern default and blocks the classic CSRF form POST.
→ [Ch 31](../chapters/31-security.md)

**SameValueZero** — The equality algorithm used by `includes`, `Set`, and `Map`: like `===` but
treating `NaN` as equal to itself. → [Ch 15](../chapters/15-maps-sets-weak-collections.md)

**Sanitising** — Parsing untrusted markup and removing anything not allowlisted. Requires a real
parser; a regex cannot do it correctly. → [Ch 31](../chapters/31-security.md)

**Scavenger** — V8's minor garbage collector, copying survivors between semi-spaces. Its cost is
proportional to what survived, making short-lived garbage cheap.
→ [Ch 27](../chapters/27-memory-garbage-collection.md)

**Scope chain** — The ordered sequence of environments a variable lookup traverses.
→ [Ch 8](../chapters/08-scope-hoisting-tdz.md)

**Service worker** — A script intercepting network requests for its scope, enabling offline
behaviour and custom caching. → [Ch 25](../chapters/25-storage-browser-apis.md)

**Shadowing** — An inner scope declaring a variable with the same name as an outer one, hiding it.
→ [Ch 8](../chapters/08-scope-hoisting-tdz.md)

**Shadow DOM** — An encapsulated subtree with its own style scope, underlying web components.
→ [Ch 21](../chapters/21-dom.md)

**Shallow copy** — A copy duplicating only the top level, sharing nested references. What spread and
`Object.assign` produce. → [Ch 5](../chapters/05-objects.md)

**Shim** — Code approximating an API's surface without truly implementing its semantics, as distinct
from a faithful polyfill. → [Ch 33](../chapters/33-es6-to-es2025.md)

**Side effect** — Any observable change outside a function's return value: mutation, I/O, logging.
→ [Ch 29](../chapters/29-functional-programming.md)

**Signal** — (1) An `AbortSignal`, carrying cancellation. (2) In reactive systems, a value whose
readers are tracked so dependents update automatically.
→ [Ch 22](../chapters/22-events.md), [App C](C-projects.md)

**Sparse array** — An array with missing indices. Method behaviour differs across them — `map`
preserves holes, `filter` compacts them — which is why they're best avoided.
→ [Ch 6](../chapters/06-arrays.md)

**Spread** — The `...` syntax expanding an iterable or object's own enumerable properties into a new
one. → [Ch 13](../chapters/13-destructuring-spread-rest.md)

**SRI (Subresource Integrity)** — An `integrity` attribute containing a hash, so a browser refuses a
resource whose content has changed. Supply-chain defence.
→ [Ch 31](../chapters/31-security.md)

**Stack** — (1) The call stack. (2) The memory region holding primitives and references, as opposed
to the heap. → [Ch 27](../chapters/27-memory-garbage-collection.md)

**State machine** — An explicit enumeration of legal states and transitions, making impossible
states unrepresentable. The answer when three booleans describe one thing.
→ [Ch 30](../chapters/30-design-patterns.md)

**Strategy** — A pattern swapping an algorithm at runtime. In JavaScript, usually a function
parameter or a map of functions. → [Ch 30](../chapters/30-design-patterns.md)

**Strict mode** — A restricted variant in which assignments to undeclared variables throw, `this` is
`undefined` in plain calls, and silent failures become errors. Modules and class bodies are always
strict. → [Ch 1](../chapters/01-how-javascript-works.md)

**Structural sharing** — An immutable update copying only the path from the root to the change,
reusing every untouched branch by reference.
→ [Ch 29](../chapters/29-functional-programming.md)

**structuredClone** — The algorithm behind `postMessage` and IndexedDB, exposed as a function. Deep
copies handling cycles, `Map`, `Set`, `Date`, and `ArrayBuffer`.
→ [Ch 25](../chapters/25-storage-browser-apis.md)

**Stub** — A test double returning canned values so the code under test can proceed.
→ [Ch 32](../chapters/32-testing.md)

**SubtleCrypto** — The `crypto.subtle` API for hashing, signing, and encryption. Asynchronous, and
requires a secure context. → [Ch 31](../chapters/31-security.md)

**Symbol** — A unique primitive usable as a property key, never equal to any other symbol. Skipped by
`Object.keys` and `JSON.stringify`.
→ [Ch 14](../chapters/14-symbols-iterators-generators.md)

## T

**Tagged template** — A function invoked with a template literal's string parts and interpolated
values separately, letting it control how they combine. How `String.raw` works.
→ [Ch 7](../chapters/07-strings-numbers-math-dates.md)

**Tail call optimisation** — Reusing a stack frame for a call in tail position. Specified in ES2015
but implemented only in JavaScriptCore, so trampolining is required in practice.
→ [Ch 29](../chapters/29-functional-programming.md)

**TC39** — The committee that standardises ECMAScript, advancing proposals through stages 0 to 4.
Stage 3 means implementations are invited, not that any exist.
→ [Ch 1](../chapters/01-how-javascript-works.md)

**TDZ (Temporal Dead Zone)** — The span between a `let`, `const`, or `class` binding being hoisted
and initialised, during which access throws `ReferenceError`.
→ [Ch 8](../chapters/08-scope-hoisting-tdz.md)

**Temporal** — The ES2025 date and time API: immutable, timezone-aware, one-indexed months.
Replaces `Date`, which cannot be fixed compatibly.
→ [Ch 7](../chapters/07-strings-numbers-math-dates.md), [Ch 33](../chapters/33-es6-to-es2025.md)

**Thenable** — Any object with a callable `then` method. `await` and `Promise.resolve` accept them,
which is why a polluted `then` property can hijack a chain.
→ [Ch 19](../chapters/19-async-javascript.md)

**Throttle** — Limiting a function to at most one call per interval during a burst. Distinct from
debounce, which waits for silence. → [Ch 24](../chapters/24-timers-scheduling.md)

**ToPrimitive** — The internal operation converting an object to a primitive, consulting
`Symbol.toPrimitive`, `valueOf`, and `toString` with a hint.
→ [Ch 2](../chapters/02-variables-types-operators.md)

**Trampolining** — Converting recursion into iteration by returning a thunk instead of recursing, so
the stack never grows. → [Ch 29](../chapters/29-functional-programming.md)

**Transducer** — A composable transformation applied in a single pass, avoiding the intermediate
arrays that chained `filter().map()` allocates.
→ [Ch 29](../chapters/29-functional-programming.md)

**Transpiler** — A tool rewriting source into equivalent code using older syntax. Necessary for
syntax; polyfills handle missing APIs. → [Ch 33](../chapters/33-es6-to-es2025.md)

**Tree shaking** — Eliminating unused exports, made possible by ESM's static structure. Defeated by
dynamic `require` and unprovable top-level side effects.
→ [Ch 17](../chapters/17-modules.md), [Ch 28](../chapters/28-performance.md)

**Trusted Types** — A CSP-enforced mechanism making dangerous DOM sinks refuse plain strings,
funnelling all HTML through one audited policy. → [Ch 31](../chapters/31-security.md)

**Truthy** — Any value that is not one of the eight falsy values.
→ [Ch 2](../chapters/02-variables-types-operators.md)

**Typed array** — A fixed-width view over an `ArrayBuffer`: `Uint8Array`, `Float32Array`, and
others. → [Ch 25](../chapters/25-storage-browser-apis.md)

## U

**Unhandled rejection** — A promise that rejects with no handler attached by the time the microtask
queue drains. A warning in browsers; fatal in Node by default.
→ [Ch 19](../chapters/19-async-javascript.md)

**Unicode property escape** — `\p{…}` in a regex with the `u` flag, matching by Unicode property —
`\p{L}` for any letter. → [Ch 26](../chapters/26-regular-expressions.md)

## V

**V8** — Google's JavaScript engine, embedded in Chrome, Edge, Node, and Deno.
→ [Ch 1](../chapters/01-how-javascript-works.md)

**Virtualisation** — Rendering only the visible portion of a long list, with spacers preserving the
total scroll height. → [Ch 28](../chapters/28-performance.md)

## W

**WeakMap** — A collection holding its keys weakly, so an entry disappears when its key is
collected. Not iterable and has no `size`, because either would make GC timing observable.
→ [Ch 15](../chapters/15-maps-sets-weak-collections.md)

**WeakRef** — A weak reference to a single object, read with `deref()`. For caches and diagnostics;
prefer `WeakMap` where it fits. → [Ch 27](../chapters/27-memory-garbage-collection.md)

**Web Worker** — A separate thread with no DOM access, communicating by `postMessage`. The right
place for CPU-bound work. → [Ch 25](../chapters/25-storage-browser-apis.md)

## X

**XSS (Cross-Site Scripting)** — Attacker-controlled JavaScript executing in your origin, giving it
the user's session, storage, and authenticated request capability. Three types: stored, reflected,
and DOM-based. → [Ch 31](../chapters/31-security.md)

## Y

**yield** — The generator keyword that pauses execution and produces a value. `yield*` delegates to
another iterable and captures its return value.
→ [Ch 14](../chapters/14-symbols-iterators-generators.md)

---

**Back to:** [Table of Contents](../README.md)
