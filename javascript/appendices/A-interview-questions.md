# Appendix A — 300+ Interview Questions

Grouped by topic, and within each topic by level. Every answer includes a **Follow-up:** line
stating what a good interviewer asks next — because the follow-up is usually where interviews are
won or lost.

**Levels:** *Junior* — you should know this. *Mid* — you should be able to explain why.
*Senior* — you should have an opinion and know the trade-off.

---

## Contents

| Section | Questions |
|---|---|
| [0. How to Answer](#0-how-to-answer) | — |
| [1. Rapid Fire Warm-Up](#1-rapid-fire-warm-up) | 20 |
| [2. Fundamentals and Types](#2-fundamentals-and-types) | 40 |
| [3. Scope, Closures, and `this`](#3-scope-closures-and-this) | 35 |
| [4. Objects, Prototypes, and Classes](#4-objects-prototypes-and-classes) | 35 |
| [5. Arrays and Functional](#5-arrays-and-functional) | 25 |
| [6. Async and the Event Loop](#6-async-and-the-event-loop) | 45 |
| [7. DOM, Events, and the Browser](#7-dom-events-and-the-browser) | 30 |
| [8. Modules and Tooling](#8-modules-and-tooling) | 15 |
| [9. Performance and Memory](#9-performance-and-memory) | 25 |
| [10. Security](#10-security) | 20 |
| [11. Testing](#11-testing) | 15 |
| [12. Patterns and Architecture](#12-patterns-and-architecture) | 20 |
| [13. Modern Syntax, ES2015–ES2025](#13-modern-syntax-es2015es2025) | 20 |

---

## 0. How to Answer

**Technical questions.** Answer in three beats: the direct answer, the reason, then one concrete
consequence. "`typeof null` is `'object'`. It's a bug from the first implementation that can't be
fixed without breaking the web. So null-checking with `typeof` silently passes and you need
`x === null`." That's ten seconds and it demonstrates you've hit the problem, not just read about it.

**When you don't know.** Say so, then reason out loud toward an answer. "I haven't used
`FinalizationRegistry`. From the name I'd expect a post-collection callback — if so, I'd be careful,
because GC timing isn't guaranteed." An interviewer learns more from that than from a lucky guess,
and far more than from a bluff they can detect.

**Behavioural-technical questions** ("tell me about a bug you couldn't fix", "a time you disagreed
with a technical decision") want evidence you operate well under uncertainty. Use situation → what
you did → outcome → what you'd do differently, and keep it under two minutes. Pick a story with a
real technical detail in it; "we had a performance problem and I fixed it" is not a story. Name what
you measured. Admit what you got wrong first — every senior interviewer is listening for whether you
can do that.

**Whiteboard code.** State your approach before writing. Ask about input constraints and edge cases
— empty, one element, duplicates, null. Write the naive version, get it correct, then optimise if
asked. Talking through a wrong first idea and correcting it reads as competence; silence reads as
being stuck.

**Questions to ask them.** "How do you decide what to test?" "What does your on-call rotation look
like?" "What's the oldest code in production and who understands it?" These get you real information
and signal seniority.

---

## 1. Rapid Fire Warm-Up

Twenty questions in five minutes. If any of these are shaky, start at the chapter it points to.

**Q1. How many primitive types are there, and what are they?**
<details><summary>Answer</summary>Seven: `string`, `number`, `boolean`, `null`, `undefined`, `symbol`, `bigint`. Everything else is an object. → Chapter 2

**Follow-up:** Which one does `typeof` lie about?</details>

**Q2. What are all the falsy values?**
<details><summary>Answer</summary>`false`, `0`, `-0`, `0n`, `''`, `null`, `undefined`, `NaN`. Eight values; everything else is truthy, including `'0'`, `'false'`, `[]`, and `{}`.

**Follow-up:** Is `document.all` truthy?  (No — it's the one legacy exception.)</details>

**Q3. `==` vs `===`?**
<details><summary>Answer</summary>`===` compares type and value with no coercion. `==` coerces first, following a table nobody remembers correctly. Use `===` always; the one defensible `==` is `x == null` to test for both `null` and `undefined`.

**Follow-up:** What does `[] == false` give, and why?</details>

**Q4. `let`, `const`, `var` — the differences?**
<details><summary>Answer</summary>`var` is function-scoped, hoisted and initialised to `undefined`, redeclarable. `let` and `const` are block-scoped and hoisted-but-uninitialised (the TDZ), so accessing them early throws. `const` prevents rebinding, not mutation.

**Follow-up:** Is `const arr = []; arr.push(1)` legal?  (Yes.)</details>

**Q5. What is a closure?**
<details><summary>Answer</summary>A function plus a reference to the lexical environment where it was created, so it keeps access to those variables after the outer function has returned.

**Follow-up:** Does it capture the value or the variable?</details>

**Q6. What is hoisting?**
<details><summary>Answer</summary>Declarations are processed before any code runs. `var` and `function` declarations are initialised (to `undefined` and the function respectively); `let`, `const`, and `class` are not, leaving them in the TDZ until execution reaches them.

**Follow-up:** What's the difference between hoisting a function declaration and a function expression?</details>

**Q7. Arrow function vs regular function — name three differences.**
<details><summary>Answer</summary>Lexical `this` (no own binding), no `arguments`, no `prototype` so not `new`-able. Also can't be generators, and can't be hoisted as declarations.

**Follow-up:** When is an arrow the wrong choice?  (Object methods needing `this`, prototype methods, event handlers wanting the element.)</details>

**Q8. `null` vs `undefined`?**
<details><summary>Answer</summary>`undefined` means "no value assigned" and is the engine's default. `null` is an explicit "intentionally empty" that you assign. `typeof null` is `'object'`; `typeof undefined` is `'undefined'`.

**Follow-up:** Does `null == undefined`?  (True. `null === undefined` is false.)</details>

**Q9. What does `this` refer to?**
<details><summary>Answer</summary>Whatever the call site says. Five rules, highest priority first: `new`, explicit (`call`/`apply`/`bind`), implicit (method call), default (`undefined` in strict mode, `globalThis` otherwise). Arrows ignore all four and inherit lexically.

**Follow-up:** What is `this` in a callback passed to `setTimeout`?</details>

**Q10. Difference between a microtask and a macrotask?**
<details><summary>Answer</summary>The entire microtask queue drains after each macrotask and before the next render. Promise callbacks and `queueMicrotask` are microtasks; timers, I/O, and events are macrotasks.

**Follow-up:** Can a microtask starve the event loop?  (Yes — one that queues another, forever.)</details>

**Q11. What does `async` do to a function's return value?**
<details><summary>Answer</summary>Wraps it in a resolved promise. A `throw` becomes a rejected promise. Returning a promise doesn't double-wrap.

**Follow-up:** What does `await` do with a non-promise?  (Wraps and resolves it, costing one microtask tick.)</details>

**Q12. `Promise.all` vs `allSettled` vs `race` vs `any`?**
<details><summary>Answer</summary>`all` — all fulfil, or reject on the first rejection. `allSettled` — never rejects, reports every outcome. `race` — first to settle either way. `any` — first to fulfil, rejects with `AggregateError` only if all reject.

**Follow-up:** Which do you want for "load six widgets, show whichever succeed"?  (`allSettled`.)</details>

**Q13. What is the prototype chain?**
<details><summary>Answer</summary>Failed property lookups delegate to `Object.getPrototypeOf(obj)`, recursively, until `null`. It's how inheritance works, and it's lookup-only — writes always create an own property.

**Follow-up:** Why does `for...in` see inherited keys but `Object.keys` doesn't?</details>

**Q14. `map` vs `forEach`?**
<details><summary>Answer</summary>`map` returns a new array of the callback's return values; `forEach` returns `undefined` and exists for side effects. Neither can be broken out of early.

**Follow-up:** Which array methods mutate?</details>

**Q15. What is event delegation?**
<details><summary>Answer</summary>One listener on a stable ancestor handling events from many descendants, matched with `e.target.closest(selector)`. Constant memory, and it covers elements added later.

**Follow-up:** What's the difference between `e.target` and `e.currentTarget`?</details>

**Q16. `textContent` vs `innerHTML`?**
<details><summary>Answer</summary>`textContent` sets text and cannot create elements, so it's XSS-safe. `innerHTML` parses markup and is a primary XSS sink.

**Follow-up:** Does `innerHTML` execute a `<script>` tag?  (No — but `<img onerror>` does.)</details>

**Q17. What does `0.1 + 0.2 === 0.3` return, and why?**
<details><summary>Answer</summary>`false`. IEEE-754 doubles can't represent 0.1 or 0.2 exactly, so the sum is `0.30000000000000004`. Compare with a tolerance, or use integers (pence, not pounds) for money.

**Follow-up:** How would you store a currency amount?</details>

**Q18. `Map` vs plain object — when do you reach for `Map`?**
<details><summary>Answer</summary>Any-type keys, guaranteed insertion order, an honest `size`, no prototype keys to collide with, and better performance for frequent additions and deletions.

**Follow-up:** When is an object still better?  (JSON serialisation, fixed known shape, string keys.)</details>

**Q19. What is `NaN`, and how do you test for it?**
<details><summary>Answer</summary>The result of an invalid numeric operation, and the only value not equal to itself. Test with `Number.isNaN(x)`; the global `isNaN` coerces first, so `isNaN('abc')` is `true`.

**Follow-up:** What is `typeof NaN`?  (`'number'`.)</details>

**Q20. What is optional chaining, and what problem did it solve?**
<details><summary>Answer</summary>`a?.b` short-circuits to `undefined` when `a` is `null` or `undefined`, replacing chains of `a && a.b && a.b.c`. It works on calls (`fn?.()`) and indexes (`arr?.[0]`) too.

**Follow-up:** Difference between `??` and `||`?</details>

---

## 2. Fundamentals and Types

### Junior

**Q21. What does `typeof` return for each primitive plus arrays, functions, and `null`?**
<details><summary>Answer</summary>`'string'`, `'number'`, `'boolean'`, `'undefined'`, `'symbol'`, `'bigint'`, `'function'` for functions, and `'object'` for objects, arrays, and `null`.

**Follow-up:** How do you distinguish an array from an object?  (`Array.isArray`.)</details>

**Q22. Why is `typeof null === 'object'`?**
<details><summary>Answer</summary>An artefact of the original implementation, where type tags lived in the low bits of a value and the null pointer's tag happened to be the object tag. Fixing it was proposed and rejected because it would break existing sites.

**Follow-up:** How do you reliably check for `null`?</details>

**Q23. What is type coercion? Give three surprising examples.**
<details><summary>Answer</summary>Implicit type conversion during an operation. `'5' - 2` is `3` but `'5' + 2` is `'52'`; `[] + {}` is `'[object Object]'`; `[] == false` is `true`.

**Follow-up:** Which operator triggers `ToPrimitive` with a `'number'` hint?</details>

**Q24. `let`, `const`, and `var` in a loop with `setTimeout` — what happens?**
<details><summary>Answer</summary>`let` creates a new binding per iteration, so the callbacks log `0 1 2`. `var` has one function-scoped binding, so they all log the final value.

**Follow-up:** How would you fix the `var` version without changing the keyword?</details>

**Q25. What is the TDZ?**
<details><summary>Answer</summary>The span between a `let`/`const`/`class` binding being hoisted and being initialised. Access inside it throws `ReferenceError` rather than yielding `undefined`, which turns a class of silent bugs into loud ones.

**Follow-up:** Is `typeof x` safe on a TDZ variable?  (No — it throws, unlike on an undeclared variable.)</details>

**Q26. Difference between `slice` and `splice`?**
<details><summary>Answer</summary>`slice(start, end)` returns a shallow copy and doesn't mutate. `splice(start, count, ...items)` removes and inserts in place, returning what it removed.

**Follow-up:** What's the ES2023 non-mutating equivalent of `splice`?</details>

**Q27. What does `+` do with mixed operand types?**
<details><summary>Answer</summary>If either operand becomes a string after `ToPrimitive`, it concatenates; otherwise it adds numerically. It's the only arithmetic operator that does this — `-`, `*`, `/` always coerce to number.

**Follow-up:** What is `1 + '2' - 1`?  (`11` — concatenate then numerically subtract.)</details>

**Q28. Template literals — what can they do that concatenation can't?**
<details><summary>Answer</summary>Interpolate expressions, span lines, nest, and act as tagged templates where a function receives the string parts and values separately — which is how `String.raw` and CSS-in-JS libraries work.

**Follow-up:** What would you use a tagged template for?  (Escaping, i18n, SQL parameterisation.)</details>

**Q29. What is a symbol and why would you use one?**
<details><summary>Answer</summary>A unique, non-string primitive usable as a property key. Two symbols with the same description are never equal, so they're collision-proof for metadata on objects you don't own, and they're skipped by `Object.keys` and `JSON.stringify`.

**Follow-up:** What does `Symbol.iterator` do?</details>

**Q30. Shallow vs deep copy?**
<details><summary>Answer</summary>A shallow copy (spread, `Object.assign`, `slice`) duplicates the top level and shares nested references. A deep copy duplicates the whole graph — `structuredClone` handles cycles, `Map`, `Set`, and `Date`, unlike `JSON.parse(JSON.stringify(x))`.

**Follow-up:** What does the JSON round-trip lose?</details>

### Mid

**Q31. Walk through `[] == ![]`.**
<details><summary>Answer</summary>`![]` is `false`. `[] == false` → `false` becomes `0`; `[]` becomes `''` via `ToPrimitive`, then `0`. `0 == 0` is `true`. The whole expression is `true`, and this is the canonical argument for never using `==`.

**Follow-up:** What is `[] == []`?  (`false` — two distinct object references.)</details>

**Q32. `Object.is` vs `===`?**
<details><summary>Answer</summary>Two differences: `Object.is(NaN, NaN)` is `true` where `===` is `false`, and `Object.is(0, -0)` is `false` where `===` is `true`. It's SameValue; `===` is StrictEquality.

**Follow-up:** Which comparison does `[].includes` use?  (SameValueZero — finds `NaN`, treats `0` and `-0` as equal.)</details>

**Q33. How would you represent money in JavaScript?**
<details><summary>Answer</summary>As an integer count of the smallest unit — pence, cents — so all arithmetic is exact, formatting with `Intl.NumberFormat` at the edges. For very large values or a currency with unusual subdivision, `BigInt` or a decimal library. Never a float.

**Follow-up:** What's wrong with `toFixed(2)` for rounding?  (It's binary rounding on an inexact value: `(1.005).toFixed(2)` is `'1.00'`.)</details>

**Q34. What is `BigInt` and what are its limitations?**
<details><summary>Answer</summary>Arbitrary-precision integers, written `1n`. You can't mix them with `number` in arithmetic (a `TypeError`), `Math` methods don't accept them, and `JSON.stringify` throws on them. Comparison with `==` works across types; `===` doesn't.

**Follow-up:** When is `BigInt` the right choice over a float?  (Integer IDs above 2^53, cryptography, exact large arithmetic.)</details>

**Q35. Explain `Number.MAX_SAFE_INTEGER`.**
<details><summary>Answer</summary>2^53 − 1. Above it, doubles can't represent every integer, so `9007199254740993 === 9007199254740992` is `true`. This is why 64-bit IDs from a backend must be transported as strings.

**Follow-up:** What breaks if a server sends a snowflake ID as a JSON number?</details>

**Q36. What does `Object.freeze` actually prevent?**
<details><summary>Answer</summary>Adding, removing, and reconfiguring own properties, and writing to writable ones — one level deep. Nested objects stay mutable. In strict mode violations throw; otherwise they fail silently.

**Follow-up:** What's the difference between `freeze`, `seal`, and `preventExtensions`?</details>

**Q37. What are property descriptors?**
<details><summary>Answer</summary>The metadata behind each property: `value`, `writable`, `enumerable`, `configurable`, or `get`/`set` for accessors. Properties created by assignment default to all-true; those created by `defineProperty` default to all-false.

**Follow-up:** Why does that default difference matter for polyfills?</details>

**Q38. `for...in` vs `for...of` vs `Object.keys`?**
<details><summary>Answer</summary>`for...in` iterates enumerable string keys including inherited ones. `for...of` iterates any iterable's values and doesn't work on plain objects. `Object.keys` gives own enumerable string keys as an array.

**Follow-up:** Why is `for...in` over an array a bad idea?  (Index keys are strings, order isn't guaranteed for exotic objects, and it picks up added prototype properties.)</details>

**Q39. What is `Symbol.toPrimitive`?**
<details><summary>Answer</summary>A method the engine calls to convert an object to a primitive, receiving a hint of `'number'`, `'string'`, or `'default'`. It takes priority over `valueOf` and `toString`, which is how `Temporal` and `Date` control their own coercion.

**Follow-up:** What hint does `+obj` pass?  (`'number'`.)</details>

**Q40. Difference between `parseInt`, `Number`, and the unary `+`?**
<details><summary>Answer</summary>`parseInt` parses a leading integer and stops at the first invalid character, so `parseInt('12px')` is `12`. `Number('12px')` is `NaN`. Unary `+` is `Number`. `parseInt` also needs an explicit radix to be safe.

**Follow-up:** What is `['1','2','3'].map(parseInt)`?  (`[1, NaN, NaN]` — the index becomes the radix.)</details>

**Q41. What does `structuredClone` handle that `JSON` doesn't?**
<details><summary>Answer</summary>Cycles, `Map`, `Set`, `Date`, `RegExp`, `Blob`, `ArrayBuffer`, and typed arrays. It still can't clone functions, DOM nodes, or class identity — you get a plain object with the same own properties.

**Follow-up:** Which values does JSON silently drop?  (`undefined`, functions, and symbol-keyed properties.)</details>

**Q42. Why is `NaN !== NaN`?**
<details><summary>Answer</summary>IEEE-754 defines NaN as unordered, so every comparison with it is false. It follows that `NaN` is the only value for which `x !== x` is a valid identity test — which is exactly how `Number.isNaN` can be polyfilled.

**Follow-up:** Does `[NaN].indexOf(NaN)` find it?  (No — `includes` does.)</details>

**Q43. What are labelled statements for?**
<details><summary>Answer</summary>Breaking or continuing an outer loop from an inner one, which is otherwise impossible without a flag variable. Rare, and usually a sign the loop body should be a function you can `return` from.

**Follow-up:** How would you rewrite a labelled break as a function?</details>

**Q44. What's the difference between `Array(3)` and `Array.of(3)`?**
<details><summary>Answer</summary>`Array(3)` creates a length-3 array of holes; `Array.of(3)` creates `[3]`. Holes are skipped by `map`, `forEach`, and `filter` but not by `fill` or spread — which is why `Array(3).fill(0)` works and `Array(3).map(() => 0)` doesn't.

**Follow-up:** What does `[...Array(3)]` give?  (`[undefined, undefined, undefined]` — spread converts holes.)</details>

**Q45. Explain `Object.entries`, `values`, `fromEntries`, and how they compose.**
<details><summary>Answer</summary>They turn an object into arrays and back, making array methods available for object transformation: `Object.fromEntries(Object.entries(o).filter(([k, v]) => v != null))`. All three see own enumerable string keys only.

**Follow-up:** How would you invert an object's keys and values?</details>

### Senior

**Q46. When would you deliberately use `==`?**
<details><summary>Answer</summary>`x == null` as a single check for both `null` and `undefined`, which is idiomatic and unambiguous. Everything else is a liability, and even this one is often clearer as `x === null || x === undefined` in a codebase where a linter bans `==` outright. Consistency has more value here than brevity.

**Follow-up:** Does your linter config allow it?</details>

**Q47. How do you decide between `null` and `undefined` in an API you're designing?**
<details><summary>Answer</summary>Pick one and be consistent. My default: `undefined` means "absent" (a missing key, an omitted argument) and `null` means "explicitly no value" (a cleared field, a deliberate empty). Where the distinction carries meaning — a PATCH request where `null` clears a field and absent leaves it — encode it in the type and document it, because callers will not guess. If it doesn't carry meaning, use `undefined` only, since it's what the language produces by default and `JSON.stringify` drops it.

**Follow-up:** What does that mean for your JSON payloads?</details>

**Q48. A colleague proposes banning `let` in favour of `const` everywhere. Do you agree?**
<details><summary>Answer</summary>Largely yes, as a default with `prefer-const` enforcing it — it documents intent and catches accidental reassignment. But it's a default, not a rule: accumulator variables and loop counters genuinely reassign, and contorting code into `reduce` to avoid a `let` costs readability for no correctness gain. The real value is that a `let` becomes a signal worth a second look in review.

**Follow-up:** Does `const` give you immutability?  (No, only non-rebinding.)</details>

**Q49. How would you validate untrusted external data at a boundary?**
<details><summary>Answer</summary>Parse, don't validate: convert unknown input into a known shape at one boundary and let everything downstream trust the type. A schema validator (Zod, Valibot) or a hand-written `Result`-returning parser both work; what matters is that it happens once, at the edge, and produces either a typed value or a structured error. Collect every error rather than failing on the first, and never trust client-side validation to have already run.

**Follow-up:** Where does that leave TypeScript types?  (Erased at runtime — they document the output of validation, they don't perform it.)</details>

**Q50. Is JavaScript's dynamic typing a liability? Argue both sides.**
<details><summary>Answer</summary>For prototyping and glue code it's genuinely faster, and structural flexibility (duck typing, dynamic property access) makes certain patterns trivial that a nominal type system fights. At scale it shifts whole classes of error from compile time to production, and coercion makes some of those errors silent rather than loud. My practical position: use TypeScript for anything with more than one maintainer or a lifetime over six months, keep `strict` on, and validate at runtime anyway — types don't survive the network boundary.

**Follow-up:** What does TypeScript not protect you against?</details>

**Q51. Explain the abstract equality algorithm well enough that I could implement it.**
<details><summary>Answer</summary>Same types: strict equality. `null` and `undefined` equal each other and nothing else. Number and string: string is converted to number. Boolean: converted to number. Object and primitive: the object goes through `ToPrimitive` and the comparison retries. `NaN` never equals anything, including itself. That's the whole table, and the fact it takes six clauses to state is the argument against using it.

**Follow-up:** Which step makes `[] == false` true?</details>

**Q52. Why can't `typeof` distinguish an array from a `Date` from a plain object, and what do you use instead?**
<details><summary>Answer</summary>`typeof` reports the language's type tag, and all three are objects. For the common cases: `Array.isArray`, `x instanceof Date`, or `Object.prototype.toString.call(x)` which reads the internal `Symbol.toStringTag` and distinguishes all built-ins. `instanceof` fails across realms — an array from an iframe is not `instanceof` your `Array` — which is precisely why `Array.isArray` exists.

**Follow-up:** How would you write a `getType(x)` that handles realms correctly?</details>

**Q53. `Object.freeze` in production code — worth it?**
<details><summary>Answer</summary>In development, yes: deep-freezing state catches accidental mutation at the write rather than three renders later, and it's the cheapest possible immutability enforcement. In production I'd compile it out, because deep-freezing a large state tree on every update has a real cost and frozen objects can also miss engine fast paths. The exception is a small exported constant, where freezing is free and documents intent.

**Follow-up:** Why does freezing fail silently in sloppy mode?</details>

**Q54. How do you handle dates and timezones correctly?**
<details><summary>Answer</summary>Store UTC instants, do arithmetic in an explicit timezone, and format at the very last moment in the user's locale. `Date` makes all three hard — it's mutable, zero-indexed by month, and silently local — so use `Temporal` behind its polyfill, keeping the conversion at the input and output boundaries. The bugs that bite are always the same two: DST transitions where a day isn't 24 hours, and "today" meaning different dates for two users at the same instant.

**Follow-up:** What's wrong with adding 86,400,000 milliseconds to get tomorrow?</details>

**Q55. What's your position on coercion-dependent code like `!!x` and `+x`?**
<details><summary>Answer</summary>`!!x` is idiomatic enough to be fine, though `Boolean(x)` says the same thing to a reader who hasn't seen it. `+x` I'd avoid in favour of `Number(x)`, because it's easy to miss visually next to a `+` concatenation. The deeper answer is that truthiness checks themselves are the risk: `if (count)` and `if (count != null)` differ exactly when `count` is `0`, and that's a real bug I've shipped.

**Follow-up:** Where has truthiness bitten you?</details>

**Q56. Why did `??` need to exist when `||` was already there?**
<details><summary>Answer</summary>`||` falls back on any falsy value, so `port || 3000` overrides a deliberate `0` and `name || 'Anon'` overrides a deliberate empty string. `??` triggers only on `null` and `undefined`, which is what "supply a default for a missing value" actually means. The same reasoning gives `??=` over `||=`, and mixing `??` with `||`/`&&` without parentheses is a syntax error precisely because the precedence would be a footgun.

**Follow-up:** Why is `a ?? b || c` a syntax error?</details>

**Q57. How would you explain the difference between value and reference semantics to a junior?**
<details><summary>Answer</summary>Variables hold values; for objects the value *is* a reference. Assigning copies whatever the variable holds — the number itself, or the reference. So `b = a` on an object gives two names for one thing, and `b.x = 1` is visible through `a`, but `b = {}` isn't. The practical test I'd give them: predict what happens when you pass an object to a function that reassigns its parameter versus one that mutates a property.

**Follow-up:** Is JavaScript pass-by-value or pass-by-reference?  (Pass-by-value, where the value can be a reference — "call by sharing".)</details>

**Q58. What's the cost of `delete obj.prop` and what should you do instead?**
<details><summary>Answer</summary>It forces the object out of its hidden class into dictionary mode, so every subsequent property access on it is a hash lookup instead of an inline-cached offset — and it doesn't release memory the value wasn't already holding. Set it to `undefined` or `null` if you just need the value gone, rebuild the object with destructuring (`const { drop, ...rest } = obj`) if the shape must change, or use a `Map` if keys are genuinely dynamic.

**Follow-up:** When is `delete` actually correct?  (When you need `'prop' in obj` to become false, or when iterating keys must not see it.)</details>

**Q59. A codebase has 200 uses of `parseInt` without a radix. How do you approach it?**
<details><summary>Answer</summary>It's a real bug only where input can begin `0x` or where a leading zero was once octal, so the blast radius is narrower than the count suggests — but it's also a trivially safe fix. I'd enable the `radix` lint rule as an error, autofix where the tooling can, and review the rest for the actual intent, because a fair number of them want `Number()` and not `parseInt` at all. What I wouldn't do is a mass find-and-replace, since `parseInt('12px')` and `Number('12px')` differ and some callers depend on the truncating behaviour.

**Follow-up:** Where does `parseInt` differ from `Number` in a way that matters?</details>

**Q60. How would you introduce stricter type discipline into a large untyped codebase?**
<details><summary>Answer</summary>Incrementally, from the edges in. Turn on `checkJs` with JSDoc types first so nothing needs renaming, and type the boundaries — API responses, config, shared utilities — because that's where wrong assumptions cost most. Convert file by file with `strict` on for new files only, and never let a migration branch run for months. The failure mode I've seen is a big-bang rewrite that stalls at 60% and leaves two conventions in the codebase forever.

**Follow-up:** What do you do about a dependency with no types?</details>

---

## 3. Scope, Closures, and `this`

### Junior

**Q61. What is lexical scope?**
<details><summary>Answer</summary>Scope determined by where code is *written*, not where it's called from. An inner function can read its enclosing scopes' variables because the chain is fixed at authoring time.

**Follow-up:** What's the opposite?  (Dynamic scope, which JavaScript uses only for `this`.)</details>

**Q62. What is the scope chain?**
<details><summary>Answer</summary>The ordered list of environments a lookup walks: the current scope, then each enclosing one, ending at the global scope. First match wins; no match is a `ReferenceError`.

**Follow-up:** What is shadowing?</details>

**Q63. Give the classic closure loop bug and two fixes.**
<details><summary>Answer</summary>`for (var i = 0; i < 3; i++) fns.push(() => i)` — all three log `3`, because they share one binding. Fix with `let` (a fresh binding per iteration) or an IIFE per iteration that captures a copy.

**Follow-up:** What exactly does `let` do differently in a `for` head?</details>

**Q64. Name three real uses of closures.**
<details><summary>Answer</summary>Data privacy (a factory returning methods over a private variable), stateful functions (counters, `once`, memoisation), and partial application. Also every event handler and callback that references an outer variable.

**Follow-up:** How does a closure leak memory?</details>

**Q65. What are the five `this` binding rules, in priority order?**
<details><summary>Answer</summary>`new` binding, then explicit (`call`/`apply`/`bind`), then implicit (the object before the dot), then default (`undefined` in strict mode, `globalThis` otherwise). Arrow functions have no `this` of their own and inherit from the enclosing scope, overriding all four.

**Follow-up:** What does `bind` return, and can it be re-bound?</details>

**Q66. Why does `this` break when a method is passed as a callback?**
<details><summary>Answer</summary>The method is extracted from its object, so the call site has no receiver and implicit binding doesn't apply. Fix with `bind(this)`, an arrow wrapper, or a class field holding an arrow.

**Follow-up:** Which of those has a per-instance memory cost?  (The class field — one function object per instance.)</details>

**Q67. `call` vs `apply` vs `bind`?**
<details><summary>Answer</summary>`call(thisArg, a, b)` and `apply(thisArg, [a, b])` invoke immediately, differing only in how arguments are passed. `bind(thisArg, ...partial)` returns a new permanently-bound function without calling it.

**Follow-up:** What does `apply` give you that spread doesn't?  (Almost nothing now — spread replaced most uses.)</details>

**Q68. What is an IIFE and why was it needed?**
<details><summary>Answer</summary>`(function () { … })()` — a function invoked immediately to create a scope. Before block scoping and modules it was the only way to avoid polluting the global namespace.

**Follow-up:** What replaced it?</details>

**Q69. What is `this` at the top level of a module vs a script?**
<details><summary>Answer</summary>`undefined` in a module (modules are always strict and have their own scope), and `globalThis` in a classic script.

**Follow-up:** How do you reliably get the global object?  (`globalThis`.)</details>

**Q70. Does a closure capture the variable or its value?**
<details><summary>Answer</summary>The variable — a live binding. Later changes to it are visible inside the closure, which is why the `var` loop logs the final value.

**Follow-up:** How do you capture a snapshot instead?</details>

### Mid

**Q71. Implement `bind` from scratch.**
<details><summary>Answer</summary>

```js
Function.prototype.myBind = function (thisArg, ...bound) {
  const target = this;
  function wrapper(...args) {
    // `new wrapper()` must ignore thisArg and construct from target
    return target.apply(this instanceof wrapper ? this : thisArg, [...bound, ...args]);
  }
  wrapper.prototype = Object.create(target.prototype ?? null);
  return wrapper;
};
```

**Follow-up:** Why the `this instanceof wrapper` check?  (So a bound function is still usable with `new`, which the spec requires.)</details>

**Q72. Why can't you rebind a function returned by `bind`?**
<details><summary>Answer</summary>The bound function stores its `this` internally and ignores the receiver on subsequent calls, so a second `bind` or a `call` has no effect. `new` is the one exception — construction overrides it.

**Follow-up:** What does that mean for `fn.bind(a).bind(b)`?  (`a` wins.)</details>

**Q73. What is a memory leak caused by a closure? Give a concrete case.**
<details><summary>Answer</summary>A listener on `window` whose handler references a component instance: the listener is a GC root, so the whole component and any DOM it captured stay alive after unmount. The fix is an `AbortController` whose signal is passed to every listener and aborted on teardown.

**Follow-up:** Does removing the element remove the listener?  (Yes, if nothing else references the element.)</details>

**Q74. Do two closures from the same factory call share state?**
<details><summary>Answer</summary>Two closures from the *same* invocation share one environment; closures from two separate invocations have independent environments. That's the whole basis of the factory pattern.

**Follow-up:** How would you prove it in three lines?</details>

**Q75. Explain function-declaration hoisting versus function-expression hoisting.**
<details><summary>Answer</summary>A declaration is hoisted with its body, so it's callable before its position. An expression assigned to `var` hoists only the variable (initialised `undefined`), so calling it early is a `TypeError`; assigned to `let`/`const` it's in the TDZ and throws `ReferenceError`.

**Follow-up:** What about a named function expression's name?  (Visible only inside its own body.)</details>

**Q76. What is `this` inside a class field arrow versus a class method?**
<details><summary>Answer</summary>A field initialised to an arrow captures the instance at construction, so it's permanently bound and safe to pass as a callback — at the cost of one function object per instance. A method lives on the prototype and loses `this` when detached.

**Follow-up:** Which do you use for a React event handler, and why?</details>

**Q77. How do closures make memoisation possible?**
<details><summary>Answer</summary>The cache lives in the enclosing scope, reachable only through the returned function, so it survives between calls without being global or a property anyone can tamper with.

**Follow-up:** What's the danger?  (An unbounded cache is a leak; use an LRU bound or a `WeakMap` for object keys.)</details>

**Q78. Explain the module pattern using closures.**
<details><summary>Answer</summary>An IIFE creates a private scope; the returned object exposes only chosen functions, which close over the private variables. It gave encapsulation before `#private` fields and ESM existed.

**Follow-up:** What does a real module give you that the pattern doesn't?  (Static analysis, tree shaking, and live bindings.)</details>

**Q79. What does `bind` cost, and when does it matter?**
<details><summary>Answer</summary>Each call allocates a new function object with its own bound-this and partial arguments. In a render path that binds per row per frame, that's real allocation churn; done once in a constructor it's free.

**Follow-up:** How would you avoid rebinding in a list render?</details>

**Q80. How does `this` behave in `setTimeout` callbacks, and why?**
<details><summary>Answer</summary>A plain function callback gets default binding — `undefined` in strict mode, or `globalThis`/the `Timeout` object depending on host in sloppy mode — because the timer invokes it with no receiver. An arrow inherits `this` from where it was written, which is almost always what you want.

**Follow-up:** What is `this` in a `setTimeout` callback in Node?</details>

**Q81. Implement `once(fn)` such that `fn` runs at most once.**
<details><summary>Answer</summary>

```js
function once(fn) {
  let called = false, result;
  return function (...args) {
    if (called) return result;
    called = true;
    return (result = fn.apply(this, args));   // apply preserves the receiver
  };
}
```

**Follow-up:** What if `fn` throws — should the next call retry?  (A design decision; state it. Retrying means not setting `called` until success.)</details>

**Q82. What is currying, and how is it different from partial application?**
<details><summary>Answer</summary>Currying transforms an n-ary function into n unary functions (`f(a)(b)(c)`). Partial application fixes some arguments and returns a function of the rest, without insisting on one at a time. Both rely on closures.

**Follow-up:** Why does argument order matter for currying?</details>

**Q83. Why does `fn.length` sometimes disagree with the number of parameters?**
<details><summary>Answer</summary>It counts parameters before the first default or rest parameter, so `(a, b = 1, ...c) => {}` has length 1. Any `curry` implementation relying on `fn.length` therefore breaks on such functions and needs an explicit arity.

**Follow-up:** How would you fix `curry` for that case?</details>

**Q84. What's a stale closure, and where does it commonly bite?**
<details><summary>Answer</summary>A closure capturing a binding that has since been replaced rather than mutated — most visibly in React, where a `useEffect` with a missing dependency keeps the first render's `state` value forever. The general shape is any long-lived callback created once over a variable that gets reassigned.

**Follow-up:** How does a ref solve it?  (By capturing a stable container whose `.current` is read at call time.)</details>

**Q85. Explain why an arrow function can't be used as a constructor.**
<details><summary>Answer</summary>It has no `[[Construct]]` internal method and no `prototype` property, so there's nothing for `new` to link an instance to. It also has no own `this` to initialise, which is the deeper reason — `new`'s whole job is binding a fresh `this`.

**Follow-up:** Does an arrow have `arguments`?  (No — it sees the enclosing function's.)</details>

### Senior

**Q86. How would you explain closures to someone who's convinced they're just about counters?**
<details><summary>Answer</summary>Reframe from the mechanism to the consequence: a closure is how a function keeps private state that nobody else can reach, which is the same tool behind module encapsulation, memoisation, event handlers that remember their context, and every dependency-injection factory. I'd show the same code twice — as a counter and as a rate limiter — so they see the shape is identical and only the intent changed. Then the memory hazard, because that's the part that turns understanding into caution.

**Follow-up:** Where does that understanding change how they'd write code tomorrow?</details>

**Q87. `this` is often called JavaScript's worst design decision. Do you agree?**
<details><summary>Answer</summary>The dynamic binding is genuinely a mistake — a function's meaning depending on its call site defeats local reasoning, and the fact that it took arrow functions to fix it is telling. But it exists because JavaScript needed to look like Java in 1995 while being prototypal, and `this`-based methods do enable useful patterns like `call`-borrowing and mixins. In practice I sidestep the debate: arrows and class fields for anything passed as a callback, and I don't write code where the answer to "what is `this`?" is interesting.

**Follow-up:** What's your rule for when a method may rely on `this`?</details>

**Q88. When is a closure the wrong tool and a class the right one?**
<details><summary>Answer</summary>When you need many instances with shared behaviour, because a closure allocates every method per instance while a prototype shares them — at a thousand instances that's measurable. Classes also give you `instanceof`, inheritance where it's genuinely warranted, and better debugger and profiler output, since named prototype methods appear in stack traces where anonymous closures don't. I reach for closures for one-off factories, single-instance modules, and anything where the privacy is the point.

**Follow-up:** Do `#private` fields change that calculus?  (Yes — they give classes real privacy, removing the closure's main advantage.)</details>

**Q89. A junior's code has `const self = this` at the top of every method. What do you say?**
<details><summary>Answer</summary>It's a pre-2015 idiom that arrow functions made unnecessary, so the fix is replacing the inner callbacks with arrows rather than keeping the alias. I'd explain *why* it existed — that a nested `function` gets its own `this` — because deleting the workaround without understanding the problem means they'll be baffled the first time an arrow *is* wrong. Then I'd point at the one place the alias still has a use: needing both the lexical and dynamic `this` in the same scope, which is rare enough to justify a comment when it happens.

**Follow-up:** Where would an arrow be the wrong replacement?</details>

**Q90. How do you debug a `this`-related bug in unfamiliar code?**
<details><summary>Answer</summary>Look at the call site, not the definition — the value of `this` is decided there, so reading the function body tells you nothing. In practice that means a breakpoint on the first line and reading the call stack one frame up, checking whether the function was invoked as a method, detached, bound, or passed to a host API that calls it with its own receiver. `console.log(this)` at the top is faster and usually enough; the answer is almost always "it was detached from its object".

**Follow-up:** What do the devtools show you that a log doesn't?</details>

**Q91. What are the memory implications of heavy closure use in a hot render path?**
<details><summary>Answer</summary>Each closure created per item per frame is an allocation, and a thousand rows at 60fps is 60,000 short-lived objects a second — enough to make scavenges visible in a profile. The fix is hoisting the function out and passing data through a parameter or a `data-` attribute with delegation, so one function serves all rows. That said, I'd measure first: this is one of the few micro-optimisations that sometimes matters, and one of the many that usually doesn't.

**Follow-up:** How would you confirm it in DevTools?  (Allocation sampling, and GC time in the Performance panel.)</details>

**Q92. How does V8 decide which closure variables to retain?**
<details><summary>Answer</summary>At parse time it works out which variables any inner function references and builds a context containing only those, dropping the rest — so a returned function that ignores a large array usually doesn't retain it. The analysis is defeated by `eval`, `with`, and an attached debugger, and it's per-scope rather than per-closure, so one sibling function referencing a variable retains it for all of them. It's an optimisation, not a guarantee, so I don't write code that depends on it.

**Follow-up:** How would you make the retention explicit instead?</details>

**Q93. Would you ban `bind` in favour of arrows in a style guide?**
<details><summary>Answer</summary>No — they solve overlapping but distinct problems, and a blanket ban invites worse workarounds. Arrows are right for preserving lexical `this`; `bind` is right for partial application and for binding to something *other* than the lexical `this`, which an arrow can't express. What I would ban is `bind` inside a render or a JSX prop, because that allocates per render and breaks memoisation — and that's a lint rule, not a language rule.

**Follow-up:** What lint rule expresses that?</details>

**Q94. Explain the TDZ's design rationale — why not just hoist `let` as `undefined`?**
<details><summary>Answer</summary>Because `undefined` is a legitimate value, so hoisting-as-undefined makes "used before declared" indistinguishable from "declared and holds undefined", and the bug surfaces far from its cause. The TDZ makes it a `ReferenceError` at the exact line, which is strictly more useful. It's also what makes `const` coherent — a `const` that briefly held `undefined` before its initialiser would violate its own contract.

**Follow-up:** Why does `typeof` throw in the TDZ when it doesn't for undeclared variables?</details>

**Q95. How would you audit a codebase for closure-related leaks?**
<details><summary>Answer</summary>Start from the symptom rather than the code: heap snapshots around a repeated action, then read the Retainers chain, which names the exact reference keeping things alive. Statically, I'd grep for the four known shapes — `addEventListener` on `window`/`document` without a `signal` or matching `removeEventListener`, `setInterval` without a stored handle, arrays or `Map`s holding DOM nodes, and subscriptions whose unsubscribe is discarded. Then make the fix structural: one `AbortController` per component lifetime, so the correct pattern is the easy one.

**Follow-up:** What lint rules help?  (Exhaustive-deps for stale closures; there's no reliable static check for listener cleanup, which is why the structural fix matters.)</details>

---

## 4. Objects, Prototypes, and Classes

### Junior

**Q96. What is the prototype chain and how does lookup work?**
<details><summary>Answer</summary>Every object has an internal link to another object (or `null`). A failed own-property lookup follows that link, repeatedly, until found or the chain ends. Only *reads* delegate — assignment always creates an own property on the target.

**Follow-up:** What happens if you assign to a property that exists only on the prototype?</details>

**Q97. `__proto__` vs `prototype`?**
<details><summary>Answer</summary>`__proto__` is a legacy accessor for an object's own prototype link (use `Object.getPrototypeOf`). `prototype` is a property of *constructor functions*, holding the object that instances will link to.

**Follow-up:** Does an arrow function have a `prototype`?  (No.)</details>

**Q98. What does `new` actually do?**
<details><summary>Answer</summary>Creates an object linked to the constructor's `prototype`, calls the constructor with `this` bound to it, and returns that object unless the constructor explicitly returns a different object.

**Follow-up:** What if the constructor returns a primitive?  (Ignored — the new object is returned.)</details>

**Q99. Are JavaScript classes "real" classes?**
<details><summary>Answer</summary>They're syntax over prototypes — methods land on `Constructor.prototype` and `extends` sets up the chain. But they're not purely sugar: class bodies are always strict, methods aren't enumerable, `new` is required, and `#private` fields have no prototypal equivalent.

**Follow-up:** Name something a class does that you can't replicate with functions.</details>

**Q100. What is `super` for?**
<details><summary>Answer</summary>`super(...)` in a constructor calls the parent constructor and must run before `this` is used. `super.method()` calls a parent method, which is how you extend rather than replace behaviour.

**Follow-up:** Why must `super()` come first?  (`this` doesn't exist until the parent constructor creates it.)</details>

**Q101. How do you make a property private?**
<details><summary>Answer</summary>`#field` syntax — genuinely inaccessible outside the class body, enforced by the language rather than convention. Before it, closures (in a factory) or an underscore prefix (convention only, fully accessible).

**Follow-up:** How do you check whether an object has a private field?  (`#x in obj`.)</details>

**Q102. What is a static member?**
<details><summary>Answer</summary>A property or method on the constructor rather than instances — `Array.isArray`, `Math.max`. Useful for factories, constants, and utilities that don't need an instance.

**Follow-up:** Can a static method access `#private` instance fields?  (Yes, given an instance — privacy is per-class, not per-instance.)</details>

**Q103. Difference between `Object.keys`, `getOwnPropertyNames`, and `Reflect.ownKeys`?**
<details><summary>Answer</summary>`keys` gives own enumerable string keys. `getOwnPropertyNames` adds non-enumerable ones. `Reflect.ownKeys` adds symbol keys too — the complete list.

**Follow-up:** Which does `JSON.stringify` use?  (Own enumerable string keys, like `Object.keys`.)</details>

**Q104. What is `instanceof` and when does it fail?**
<details><summary>Answer</summary>It walks the prototype chain looking for the constructor's `prototype`. It fails across realms (an iframe's array isn't `instanceof` your `Array`), on primitives, and if the prototype was reassigned.

**Follow-up:** What's the realm-safe alternative for arrays?</details>

**Q105. How do you copy an object?**
<details><summary>Answer</summary>Shallow: `{ ...obj }` or `Object.assign({}, obj)` — both copy own enumerable properties and invoke getters. Deep: `structuredClone(obj)`. Neither preserves the prototype; `Object.create(Object.getPrototypeOf(o), Object.getOwnPropertyDescriptors(o))` does.

**Follow-up:** Why does spread lose accessors?  (It reads the value, so a getter becomes a plain property.)</details>

### Mid

**Q106. Implement inheritance without `class`.**
<details><summary>Answer</summary>

```js
function Animal(name) { this.name = name; }
Animal.prototype.speak = function () { return `${this.name} makes a noise`; };

function Dog(name) { Animal.call(this, name); }          // borrow the parent constructor
Dog.prototype = Object.create(Animal.prototype);          // link the chain
Dog.prototype.constructor = Dog;                          // repair the constructor pointer
Dog.prototype.speak = function () {
  return `${Animal.prototype.speak.call(this)}, specifically a bark`;   // manual "super"
};
```

**Follow-up:** Why `Object.create` rather than `new Animal()`?  (Calling the parent constructor to build a prototype runs its side effects and requires its arguments.)</details>

**Q107. What is prototype pollution?**
<details><summary>Answer</summary>Writing to `Object.prototype` so a property appears on every object. The usual vector is a recursive merge over untrusted JSON containing `__proto__`, after which something like `obj.isAdmin` reads `true` everywhere.

**Follow-up:** Name three fixes.</details>

**Q108. How do mixins work, and why use them over inheritance?**
<details><summary>Answer</summary>A function taking a base class and returning a subclass with extra behaviour, composed as `class A extends M1(M2(Base))`. It gives you multiple sources of behaviour without a single-inheritance hierarchy, and `super` still works because each mixin is a real class in the chain.

**Follow-up:** What's the downside?  (Order matters, the chain gets deep, and name collisions are silent.)</details>

**Q109. What is `Object.create(null)` for?**
<details><summary>Answer</summary>An object with no prototype — no `toString`, no `hasOwnProperty`, and nothing for prototype pollution to attach to. It's the right choice for a dictionary with untrusted keys, at the cost of losing the methods you'd otherwise inherit.

**Follow-up:** What breaks when you use one?  (`obj.hasOwnProperty(k)` — use `Object.hasOwn(obj, k)`.)</details>

**Q110. Explain getters and setters, and when they're a bad idea.**
<details><summary>Answer</summary>Accessors that look like properties, useful for computed values and for validating on write. They're a bad idea when they hide expensive work — a getter that fetches or recomputes a large derivation surprises every caller, because property access reads as free.

**Follow-up:** How do you define one with `defineProperty`?</details>

**Q111. What is `Proxy` and give a real use.**
<details><summary>Answer</summary>An object wrapping another whose fundamental operations (get, set, has, delete) you can intercept. Real uses: reactivity (Vue 3), validation on assignment, negative-index arrays, and logging access in development.

**Follow-up:** Why always delegate through `Reflect`?  (It preserves correct receiver semantics for inherited accessors.)</details>

**Q112. How does `Object.assign` differ from spread?**
<details><summary>Answer</summary>Spread creates a fresh object; `Object.assign` mutates its first argument. `Object.assign` also *invokes setters* on the target, where spread just defines properties — so they differ when the target has accessors.

**Follow-up:** Which one triggers a setter on the target?</details>

**Q113. What is the `constructor` property and why does it break?**
<details><summary>Answer</summary>A non-enumerable property on `Constructor.prototype` pointing back at the constructor. Replacing the whole prototype object (`Dog.prototype = Object.create(...)`) discards it, so instances report the wrong constructor until you reassign it.

**Follow-up:** Is `constructor` reliable for type checks?  (No — it's writable and easily wrong.)</details>

**Q114. Composition over inheritance — what does that mean concretely here?**
<details><summary>Answer</summary>Instead of `class Duck extends Bird extends Animal`, give `Duck` objects that *do* things: a `flyer`, a `swimmer`. It avoids the fragile-base-class problem and the "penguin is a bird that can't fly" hierarchy break, and it's easier to test each capability in isolation.

**Follow-up:** When is inheritance still the right call?  (A genuine is-a with a stable, shallow hierarchy — framework component base classes, error hierarchies.)</details>

**Q115. How would you implement a class hierarchy for custom errors?**
<details><summary>Answer</summary>

```js
class AppError extends Error {
  constructor(message, options) {
    super(message, options);              // options carries `cause`
    this.name = this.constructor.name;     // so stack traces say ValidationError
  }
}
class ValidationError extends AppError {
  constructor(message, { field, ...rest } = {}) { super(message, rest); this.field = field; }
}
```

**Follow-up:** Why set `name` from `constructor.name`?  (Otherwise every subclass reports `Error`, and it survives subclassing without repetition.)</details>

**Q116. What are `WeakMap` and `WeakSet` for, concretely?**
<details><summary>Answer</summary>Associating data with objects you don't own, where the association should die with the object — private data keyed by instance, metadata on DOM nodes, memoisation keyed by argument object. They hold keys weakly, so no cleanup code is needed.

**Follow-up:** Why aren't they iterable?  (It would make GC timing observable.)</details>

**Q117. Explain property enumeration order.**
<details><summary>Answer</summary>Integer-like keys first, in ascending numeric order; then string keys in insertion order; then symbol keys in insertion order. It's specified as of ES2020 — which is why `{ 2: 'b', 1: 'a' }` enumerates `1` first.

**Follow-up:** How do you preserve true insertion order for numeric-looking keys?  (Use a `Map`.)</details>

**Q118. What does `Object.defineProperty` give you that assignment doesn't?**
<details><summary>Answer</summary>Control over `writable`, `enumerable`, and `configurable`, plus the ability to define accessors. Critically, its defaults are all `false`, where assignment defaults to all `true` — which is why polyfills must use it to avoid appearing in `for...in`.

**Follow-up:** What happens if you `defineProperty` twice on a non-configurable property?</details>

**Q119. How do you make a class field truly read-only from outside?**
<details><summary>Answer</summary>A `#private` field with a public getter and no setter. `Object.freeze` on the instance also works but is coarse and blocks all writes; a `get`-only accessor over a private field is precise and expresses intent.

**Follow-up:** Can a subclass write to the parent's `#field`?  (No — private names are lexically scoped to the declaring class body.)</details>

**Q120. What's the difference between `Object.getPrototypeOf` and `instanceof` for type checks?**
<details><summary>Answer</summary>`getPrototypeOf` checks one level, exactly; `instanceof` walks the whole chain. For "is this exactly a plain object" the former is right; for "is this any kind of Error" the latter is. Both break across realms, where `Object.prototype.toString.call` still works.

**Follow-up:** How would you test for a plain object specifically?</details>

### Senior

**Q121. Would you use classes or factory functions for a new codebase?**
<details><summary>Answer</summary>Factory functions by default, classes where I need many instances or a real hierarchy. Factories compose better, have no `this` hazard, make dependency injection natural, and `#private`-equivalent privacy comes free from closures. Classes win on per-instance memory at scale (prototype methods are shared), on `instanceof` checks, and on debugger output — named prototype methods appear in stack traces. The deciding question is usually instance count: a hundred, factories; a hundred thousand, classes.

**Follow-up:** What changes your answer?  (Framework convention — fighting React or Angular's idiom costs more than either choice gains.)</details>

**Q122. When is prototypal inheritance the right tool over composition?**
<details><summary>Answer</summary>When the relationship is genuinely is-a, the hierarchy is shallow and stable, and you need polymorphic dispatch — error hierarchies and framework base classes are the honest examples. Everywhere else composition wins, because inheritance couples subclasses to the base's implementation details and every added level makes a change riskier. My rule: two levels is fine, three needs justifying, four is a design smell.

**Follow-up:** What's the fragile base class problem?</details>

**Q123. How would you design an API that must remain backwards-compatible for years?**
<details><summary>Answer</summary>Accept an options object rather than positional parameters, so adding a parameter is never breaking. Return objects rather than tuples for the same reason. Never widen what you accept and narrow what you return without a major version, be conservative about what you expose in the first place — everything public is a promise — and mark experimental surface clearly. And deprecate with a runtime warning and an overlapping support window, not a removal.

**Follow-up:** How do you deprecate something people depend on?</details>

**Q124. A codebase deep-merges user config into defaults. What's your review comment?**
<details><summary>Answer</summary>Prototype pollution, if any of that config is untrusted — `{"__proto__":{"isAdmin":true}}` through a naive recursive merge sets a property visible on every object in the process. The fix is three-part and I'd want all of it: reject `__proto__`/`constructor`/`prototype` keys, merge into `Object.create(null)`, and freeze `Object.prototype` at boot so a miss throws instead of succeeding. I'd also question whether deep-merging config is right at all, since it makes the effective configuration hard to reason about.

**Follow-up:** Is `JSON.parse` itself vulnerable?  (No — it creates an own property. The merge promotes it.)</details>

**Q125. What are the performance implications of the prototype chain?**
<details><summary>Answer</summary>Lookup cost is proportional to chain depth on a cache miss, but inline caches make repeated access from the same call site effectively free regardless of depth — so depth alone is rarely the problem. What is: mutating an object's shape after creation, which invalidates the cache, and `Object.setPrototypeOf` on a live object, which deoptimises everything touching it. The practical rule is to fix an object's shape in its constructor and never change its prototype.

**Follow-up:** Why is `setPrototypeOf` singled out as slow in V8's docs?</details>

**Q126. How do you decide what belongs on the prototype versus an instance field?**
<details><summary>Answer</summary>Behaviour on the prototype, state on the instance — shared functions cost one object total, per-instance arrows cost one each. The exception is a method that must survive detachment (an event handler, a callback passed to a framework), where an instance-field arrow buys correctness for that memory. At small instance counts it doesn't matter; at large ones the difference is measurable, and I'd rather establish the habit than audit it later.

**Follow-up:** How much memory is that, concretely?</details>

**Q127. `#private` fields versus closures versus underscore convention — which and why?**
<details><summary>Answer</summary>`#private` for classes: enforced by the language, visible in the syntax, and it survives `Object.keys`, `JSON.stringify`, and a determined caller. Closures for factories, where they're the natural mechanism and equally airtight. Underscore only in code that must run untranspiled somewhere ancient, and even then it's documentation rather than privacy — I've watched people reach into `_internal` in production. The one real cost of `#` is that it's genuinely inaccessible, which makes some testing and debugging harder.

**Follow-up:** How do you test something behind a `#private` field?  (Through the public interface — if that's impossible, the class is doing too much.)</details>

**Q128. Is `Proxy` worth the cost in application code?**
<details><summary>Answer</summary>Rarely in application code, often in libraries. Every trapped operation goes through a function call and can't be inline-cached, so hot paths slow measurably, and debugging becomes indirect because the object in the inspector isn't the object being read. Where it pays off is framework-level reactivity — Vue 3's whole model — and development-only tooling like a strict-mode object that throws on unknown property access. I wouldn't reach for one to validate a config object.

**Follow-up:** What's the alternative for validation?</details>

**Q129. How would you migrate a large prototype-based codebase to classes?**
<details><summary>Answer</summary>Mostly I wouldn't, unless there's a concrete reason — the two interoperate fine, `class` instances and prototype-built instances are the same thing, so a rewrite is churn without behaviour change. Where I would: files being substantially modified anyway, and constructor functions with hand-rolled inheritance chains, because `Object.create` plus `constructor` repair plus `Parent.prototype.method.call` is where the real bugs live. Convert leaf-first, one type per commit, with tests already in place.

**Follow-up:** What's the trap in converting a constructor that returns an object?  (A class constructor returning an object is legal but changes `new.target` semantics — and `super()` ordering constraints have no equivalent in the function form.)</details>

**Q130. Explain the diamond problem and how JavaScript avoids it.**
<details><summary>Answer</summary>In multiple inheritance, if B and C both extend A and D extends both, it's ambiguous which A-derived member D gets. JavaScript avoids it by only having single inheritance — the prototype chain is a chain, not a graph. Mixins reintroduce the composition benefit while keeping linearity: `M1(M2(Base))` produces one unambiguous order, and `super` resolves along it. The cost is that mixin order becomes semantically significant and collisions are silent.

**Follow-up:** How would you detect a mixin name collision?</details>

---

## 5. Arrays and Functional

### Junior

**Q131. Which array methods mutate, and what are the non-mutating equivalents?**
<details><summary>Answer</summary>Mutating: `push`, `pop`, `shift`, `unshift`, `splice`, `sort`, `reverse`, `fill`, `copyWithin`. Non-mutating equivalents: spread/`concat`, `slice`, `toSpliced`, `toSorted`, `toReversed`, `map`, and `with` for single-index replacement.

**Follow-up:** Which of the mutating ones surprises people most?  (`sort` — it reorders in place *and* returns the same array.)</details>

**Q132. `map` vs `filter` vs `reduce`?**
<details><summary>Answer</summary>`map` transforms each element one-to-one. `filter` selects a subset. `reduce` collapses to a single value of any type — and can express both of the others.

**Follow-up:** When is `reduce` the wrong choice?  (When a `for` loop or `flatMap` is clearer — a `reduce` that pushes into an array accumulator is a loop in disguise.)</details>

**Q133. What does `[10, 9, 1].sort()` return, and why?**
<details><summary>Answer</summary>`[1, 10, 9]` — the default comparator converts elements to strings and compares lexicographically. Numbers need `sort((a, b) => a - b)`.

**Follow-up:** Is `sort` stable?  (Yes, guaranteed since ES2019.)</details>

**Q134. How do you remove duplicates from an array?**
<details><summary>Answer</summary>`[...new Set(arr)]` for primitives. For objects you need a key: `[...new Map(arr.map(x => [x.id, x])).values()]`, which also keeps the last occurrence per key.

**Follow-up:** Does `Set` deduplicate `NaN`?  (Yes — it uses SameValueZero.)</details>

**Q135. What's the difference between `find` and `filter`?**
<details><summary>Answer</summary>`find` returns the first matching *element* (or `undefined`) and stops early. `filter` returns an *array* of all matches and always traverses everything.

**Follow-up:** What returns the index instead?  (`findIndex`, and `findLastIndex` from the end.)</details>

**Q136. `flat` vs `flatMap`?**
<details><summary>Answer</summary>`flat(depth)` flattens nested arrays by a given depth (default 1, `Infinity` for all). `flatMap(fn)` maps then flattens one level, which is the idiomatic way to map-and-drop by returning `[]` for exclusions.

**Follow-up:** How do you filter and map in one pass?</details>

**Q137. What does `Array.from` do that spread doesn't?**
<details><summary>Answer</summary>It accepts array-likes (objects with `length`) as well as iterables, and takes a mapping function as a second argument — so `Array.from({ length: 3 }, (_, i) => i)` builds `[0, 1, 2]` without an intermediate array.

**Follow-up:** Why does `[...{length: 3}]` throw?  (A plain object isn't iterable.)</details>

**Q138. How do you check whether every / any element passes a test?**
<details><summary>Answer</summary>`every(fn)` and `some(fn)`, both short-circuiting. Note `[].every(fn)` is `true` and `[].some(fn)` is `false` — vacuous truth, which trips people up on empty input.

**Follow-up:** Why does `every` return true for an empty array?</details>

**Q139. What is a higher-order function?**
<details><summary>Answer</summary>One that takes a function as an argument, returns a function, or both. `map`, `setTimeout`, `debounce`, and every decorator are examples.

**Follow-up:** Name a higher-order function you've written.</details>

**Q140. What does `arr.at(-1)` do, and what did it replace?**
<details><summary>Answer</summary>Returns the last element, replacing `arr[arr.length - 1]`. It works on strings and typed arrays too, and it returns `undefined` rather than throwing for out-of-range indices.

**Follow-up:** Why can't you use `arr[-1]`?  (That's a string property key, not an index.)</details>

### Mid

**Q141. Implement `map`, `filter`, and `reduce` from scratch.**
<details><summary>Answer</summary>

```js
Array.prototype.myMap = function (fn, thisArg) {
  const out = new Array(this.length);
  for (let i = 0; i < this.length; i++) {
    if (i in this) out[i] = fn.call(thisArg, this[i], i, this);   // `i in this` preserves holes
  }
  return out;
};

Array.prototype.myFilter = function (fn, thisArg) {
  const out = [];
  for (let i = 0; i < this.length; i++) {
    if (i in this && fn.call(thisArg, this[i], i, this)) out.push(this[i]);
  }
  return out;
};

Array.prototype.myReduce = function (fn, ...init) {
  let i = 0, acc;
  if (init.length) acc = init[0];
  else {
    while (i < this.length && !(i in this)) i++;
    if (i >= this.length) throw new TypeError('Reduce of empty array with no initial value');
    acc = this[i++];
  }
  for (; i < this.length; i++) if (i in this) acc = fn(acc, this[i], i, this);
  return acc;
};
```

**Follow-up:** Why the `i in this` check?  (Sparse arrays — the real methods skip holes.)</details>

**Q142. What's wrong with `arr.sort((a, b) => a.name > b.name)`?**
<details><summary>Answer</summary>The comparator must return a number; a boolean coerces to `0` or `1` and never `-1`, so the sort is undefined for the "less than" case and results vary by engine and input size. Use `a.name.localeCompare(b.name)`.

**Follow-up:** Why `localeCompare` over `<`?  (Correct handling of accents, case, and non-Latin scripts — and it's locale-aware.)</details>

**Q143. How would you group an array by a key?**
<details><summary>Answer</summary>`Object.groupBy(items, x => x.type)` (ES2024), or `Map.groupBy` when keys should be objects. Before those, a `reduce` with `(acc[key] ??= []).push(x)`.

**Follow-up:** Why does `Object.groupBy` return a null-prototype object?  (So a key called `toString` or `__proto__` can't collide or pollute.)</details>

**Q144. Explain sparse arrays and which methods respect holes.**
<details><summary>Answer</summary>An array with missing indices, created by `Array(3)`, `delete arr[1]`, or `arr[10] = 1` on a short array. `map`, `filter`, `forEach`, `some`, `every`, and `reduce` skip holes; `fill`, `join`, spread, `for...of`, `Array.from`, and the `to*` methods treat them as `undefined`.

**Follow-up:** Why avoid sparse arrays entirely?  (Inconsistent method behaviour and a slower element-kind in V8.)</details>

**Q145. Implement `chunk(arr, size)`.**
<details><summary>Answer</summary>

```js
const chunk = (arr, size) => {
  if (size < 1) throw new RangeError('size must be >= 1');
  return Array.from({ length: Math.ceil(arr.length / size) },
                    (_, i) => arr.slice(i * size, i * size + size));
};
chunk([1,2,3,4,5], 2);   // [[1,2],[3,4],[5]]
```

**Follow-up:** How would you make it lazy for a huge array?  (A generator yielding slices.)</details>

**Q146. What is a pure function, and why care?**
<details><summary>Answer</summary>Deterministic output for the same input, with no side effects. It makes tests need no setup, memoisation always safe, and reasoning local — you can understand the function without understanding the program.

**Follow-up:** Is a function that mutates its argument and returns it pure?  (No.)</details>

**Q147. Implement `pipe` and `compose`.**
<details><summary>Answer</summary>

```js
const pipe    = (...fns) => (x) => fns.reduce((acc, fn) => fn(acc), x);          // left → right
const compose = (...fns) => (x) => fns.reduceRight((acc, fn) => fn(acc), x);     // right → left
```

**Follow-up:** Which reads better and why?  (`pipe` — it matches execution and reading order.)</details>

**Q148. What is memoisation, and what are its two hazards?**
<details><summary>Answer</summary>Caching a pure function's results by argument. The hazards are an unbounded cache (a leak) and a wrong key function — `JSON.stringify(args)` gives different keys for `{a:1,b:2}` and `{b:2,a:1}`, and drops `undefined` values.

**Follow-up:** When would you use a `WeakMap` for the cache?</details>

**Q149. Why does `structuredClone` beat `JSON.parse(JSON.stringify(x))`?**
<details><summary>Answer</summary>It handles cycles, and round-trips `Map`, `Set`, `Date`, `RegExp`, `Blob`, and `ArrayBuffer`, all of which JSON either loses or throws on. It's also implemented natively, so it's faster for large structures.

**Follow-up:** What can't `structuredClone` handle?  (Functions, DOM nodes, class identity.)</details>

**Q150. What's the difference between `Array.prototype.includes` and `indexOf`?**
<details><summary>Answer</summary>`includes` uses SameValueZero so it finds `NaN`; `indexOf` uses strict equality so it can't. `includes` returns a boolean and reads better; `indexOf` gives you the position when you need it.

**Follow-up:** Which does `Set.has` use?  (SameValueZero — same as `includes`.)</details>

### Senior

**Q151. When does `reduce` hurt readability, and what would you write instead?**
<details><summary>Answer</summary>When the accumulator is mutated and returned — that's a `for` loop with extra ceremony and worse stack traces. Also when the callback needs more than about three lines, or when the operation has a purpose-built method: `Object.groupBy` over a grouping reduce, `flatMap` over a concat-reduce, `Object.fromEntries` over an object-building reduce. I keep `reduce` for genuine folds where the result type differs from the element type and the step is a one-liner.

**Follow-up:** Show me a reduce you'd rewrite.</details>

**Q152. How do you handle immutability in a large state tree without a library?**
<details><summary>Answer</summary>Structural sharing by hand: copy only the path from the root to the change and reuse every other branch by reference, which keeps cost proportional to depth rather than to data size. Beyond about three levels, hand-written spread pyramids become unreadable, so I'd write one `setIn(obj, path, value)` helper and use it everywhere. In development I'd deep-freeze state so accidental mutation throws at the write; in production I'd compile that out.

**Follow-up:** How do you prove structural sharing is working?  (`next.untouched === prev.untouched`.)</details>

**Q153. Is a functional style always better? Where does it lose?**
<details><summary>Answer</summary>It loses in tight numeric loops, where per-element allocation makes an immutable rewrite dramatically slower — image processing over a million pixels is the clear case. It also loses where the problem *is* sequential mutation: a parser's cursor, a canvas, a game loop. And point-free style loses the moment it needs `flip` or `apply` to make the plumbing work. My position is that purity belongs at the interface — a function that takes data and returns new data — while the implementation is free to be imperative and local.

**Follow-up:** How do you keep an imperative implementation testable?</details>

**Q154. How would you process a 2GB dataset in the browser?**
<details><summary>Answer</summary>Stream it — `File.stream()` or a `fetch` response's `body` through a `TextDecoderStream` and a line splitter — so peak memory stays bounded rather than loading it all. Do the parsing in a Web Worker so the main thread stays responsive, and transfer typed arrays rather than cloning them across the boundary. Aggregate incrementally, keeping only what the UI needs; if the user can only see 50 rows, never materialise 20 million. And report progress, because at that size the operation is measured in minutes and a spinner is not enough.

**Follow-up:** What's your backpressure strategy?</details>

**Q155. What's your view on adopting a functional utility library like Ramda?**
<details><summary>Answer</summary>Adopt the patterns before the dependency. `pipe`, `Result`, and a curry helper are twenty lines and cover most of the value, while the library brings a learning curve for every new hire, stack traces that name `_curry2` instead of your function, and a bundle cost for utilities the language now has — `?.`, `??`, `Object.groupBy`, `toSorted`. I'd take the dependency only if a concrete pain persists after using the patterns for a while, and I'd treat "the team already knows it" as a legitimate argument in its favour.

**Follow-up:** What if half the team knows it and half doesn't?</details>

---

## 6. Async and the Event Loop

### Junior

**Q156. What is the call stack?**
<details><summary>Answer</summary>The structure tracking which function is executing and where to return. Each call pushes a frame, each return pops one. Exceeding its depth throws `RangeError: Maximum call stack size exceeded`.

**Follow-up:** Roughly how deep can it go?  (~10,000 frames, engine-dependent.)</details>

**Q157. What is the event loop, in one paragraph?**
<details><summary>Answer</summary>While the stack is empty, take the next macrotask from the queue and run it to completion; then drain the entire microtask queue; then, if it's time, render. Repeat. Nothing else runs while a task is on the stack, which is why a long synchronous loop freezes the page.

**Follow-up:** Where does `requestAnimationFrame` fit?</details>

**Q158. Microtasks vs macrotasks — which are which?**
<details><summary>Answer</summary>Microtasks: promise reactions, `queueMicrotask`, `MutationObserver` callbacks. Macrotasks: `setTimeout`, `setInterval`, I/O, UI events, `setImmediate` in Node. The microtask queue drains completely between every macrotask.

**Follow-up:** Which starves the loop more easily?</details>

**Q159. What are the three promise states?**
<details><summary>Answer</summary>Pending, fulfilled, rejected. Transition happens once and is irreversible — "settled" means fulfilled or rejected. A promise's value can't change after settling.

**Follow-up:** What does `.then` return?  (A new promise, which is what makes chaining work.)</details>

**Q160. What is callback hell and what solved it?**
<details><summary>Answer</summary>Deeply nested callbacks with error handling repeated at every level, making control flow unreadable. Promises flattened it into a chain; `async`/`await` made it read like synchronous code.

**Follow-up:** Are callbacks always bad?  (No — they're right for events and multiple invocations, which promises can't express.)</details>

**Q161. `await` — what does it actually do?**
<details><summary>Answer</summary>Suspends the async function, returns control to the caller, and schedules the rest of the function as a microtask to run when the awaited value settles. Non-promises get wrapped, still costing a tick.

**Follow-up:** Does `await` block the thread?  (No — only the function.)</details>

**Q162. How do you handle errors in async code?**
<details><summary>Answer</summary>`try`/`catch` around `await`, or `.catch()` on a chain. Every async function returns a promise, so an unhandled rejection needs a handler somewhere — plus a global `unhandledrejection` listener as a backstop.

**Follow-up:** What happens to an unhandled rejection?  (A warning in browsers; in Node it crashes the process by default.)</details>

**Q163. `Promise.all` — what happens if one rejects?**
<details><summary>Answer</summary>The combined promise rejects immediately with that reason, and the other results are discarded — though the other operations keep running, since a promise can't be cancelled. Use `allSettled` when you need every outcome.

**Follow-up:** How do you actually cancel the others?  (An `AbortController` shared by all of them.)</details>

**Q164. What's wrong with `forEach` and `async`?**
<details><summary>Answer</summary>`forEach` ignores the returned promises, so it doesn't wait — the code after it runs before any callback finishes. Use `for...of` with `await` for sequential, or `Promise.all(arr.map(fn))` for parallel.

**Follow-up:** Which does `map` with an async callback give you?  (An array of promises.)</details>

**Q165. How do you run async operations in parallel versus sequentially?**
<details><summary>Answer</summary>Parallel: start them all, then await — `await Promise.all([a(), b()])`. Sequential: `await a(); await b();`. The common bug is awaiting inside a loop when the iterations are independent, turning a 100ms operation into 10 seconds.

**Follow-up:** How do you limit parallelism to N at a time?</details>

### Mid

**Q166. Predict the output.**
```js
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
queueMicrotask(() => console.log('4'));
console.log('5');
```
<details><summary>Answer</summary>`1 5 3 4 2` — synchronous first, then microtasks in queue order, then the macrotask.

**Follow-up:** Add `await null` before the `console.log('5')` inside an async function — what changes?</details>

**Q167. Predict the output.**
```js
async function f() {
  console.log('a');
  await null;
  console.log('b');
}
f();
console.log('c');
```
<details><summary>Answer</summary>`a c b` — the function body runs synchronously up to the first `await`, then the rest is a microtask, so `c` gets there first.

**Follow-up:** Why does `await null` still yield?  (The spec wraps and resolves any value, which always costs one tick.)</details>

**Q168. What's the difference between `return promise` and `return await promise` in a try block?**
<details><summary>Answer</summary>Inside `try`, `return promise` returns before the promise settles, so a rejection escapes the `catch`. `return await promise` waits, so the `catch` sees it. Outside a `try`, they're equivalent apart from one extra tick.

**Follow-up:** Which does a linter flag, and is it right?  (`no-return-await` flags the second — it's wrong inside a `try`.)</details>

**Q169. Implement a promise-based `sleep` that's cancellable.**
<details><summary>Answer</summary>

```js
const sleep = (ms, { signal } = {}) => new Promise((resolve, reject) => {
  if (signal?.aborted) return reject(signal.reason);
  const id = setTimeout(resolve, ms);
  signal?.addEventListener('abort', () => { clearTimeout(id); reject(signal.reason); },
                           { once: true });
});
```

**Follow-up:** Why clear the timeout on abort?  (Otherwise the pending timer keeps the closure alive.)</details>

**Q170. What is an unhandled rejection and how do you catch them globally?**
<details><summary>Answer</summary>A promise that rejects with no rejection handler attached by the time the microtask queue drains. Catch them with `window.addEventListener('unhandledrejection', …)` or `process.on('unhandledRejection', …)`, and treat them as bugs to fix rather than to suppress.

**Follow-up:** Why does attaching a handler later not help?  (The check happens when the queue drains; late handlers fire `rejectionhandled` but the warning already went out.)</details>

**Q171. Implement a concurrency-limited task pool.**
<details><summary>Answer</summary>

```js
async function pool(tasks, limit = 5) {
  const results = new Array(tasks.length);
  let next = 0;
  const worker = async () => {
    while (next < tasks.length) {
      const i = next++;                          // claim an index before awaiting
      results[i] = await tasks[i]();
    }
  };
  await Promise.all(Array.from({ length: Math.min(limit, tasks.length) }, worker));
  return results;
}
```

**Follow-up:** How do you make it not abort everything on one failure?  (Wrap each task in a try/catch and store a `{status, value}` result.)</details>

**Q172. How does `AbortController` work, and what does it actually cancel?**
<details><summary>Answer</summary>It exposes a `signal` you pass to cooperating APIs; `abort()` sets `aborted`, fires an `abort` event, and cooperating operations reject with `AbortError`. It doesn't cancel the underlying work unless that work checks the signal — `fetch` genuinely aborts the request, a CPU loop won't stop unless you poll `signal.aborted`.

**Follow-up:** How do you combine two signals?  (`AbortSignal.any([a, b])`.)</details>

**Q173. Explain the Node event loop phases.**
<details><summary>Answer</summary>Timers → pending callbacks → idle/prepare → poll (I/O) → check (`setImmediate`) → close callbacks, with `process.nextTick` and microtasks draining between each. `nextTick` outranks promise microtasks, which is why it can starve I/O if used recursively.

**Follow-up:** `setImmediate` vs `setTimeout(fn, 0)` — which runs first?  (Depends on context; inside an I/O callback, `setImmediate`.)</details>

**Q174. What's a promise anti-pattern you've seen?**
<details><summary>Answer</summary>The explicit-construction anti-pattern: `new Promise((res, rej) => somePromise().then(res).catch(rej))`, which is just `somePromise()`. Also `.then(x => x)` no-ops, forgetting to return inside `.then` (breaking the chain), and mixing `await` with `.then` in the same function.

**Follow-up:** Why is the wrapper actively harmful?  (It swallows non-Error rejections' stacks and is easy to get wrong for the error path.)</details>

**Q175. Implement `promisify`.**
<details><summary>Answer</summary>

```js
const promisify = (fn) => (...args) =>
  new Promise((resolve, reject) =>
    fn(...args, (err, ...values) =>
      err ? reject(err) : resolve(values.length > 1 ? values : values[0])));
```

**Follow-up:** What does Node's version do that this doesn't?  (Honours `util.promisify.custom` and preserves `this` when called as a method.)</details>

**Q176. How do you implement retry with exponential backoff and jitter?**
<details><summary>Answer</summary>

```js
async function retry(fn, { attempts = 3, base = 100, signal } = {}) {
  for (let i = 0; ; i++) {
    try { return await fn(); }
    catch (err) {
      if (i >= attempts - 1 || err.name === 'AbortError' || !isRetryable(err)) throw err;
      const delay = base * 2 ** i * (0.5 + Math.random() / 2);   // full jitter
      await sleep(delay, { signal });
    }
  }
}
```

**Follow-up:** Why jitter?  (Without it, all clients that failed together retry together — a thundering herd that re-breaks the service.)</details>

**Q177. What's the difference between a promise and a thenable?**
<details><summary>Answer</summary>A thenable is any object with a callable `then`. `await` and `Promise.resolve` accept thenables, which is how interop with older libraries works — and why a polluted `then` property on a plain object can hijack a chain.

**Follow-up:** How is that a security concern?  (Prototype pollution adding `then` makes every object a thenable.)</details>

**Q178. Why does an `async` function's synchronous part matter?**
<details><summary>Answer</summary>Everything before the first `await` runs immediately, on the caller's stack. So an `async` function that validates arguments and throws before any `await`... still returns a rejected promise rather than throwing synchronously, but the validation work itself blocks the caller. It also means calling an async function starts it — you don't need to await it for the work to begin.

**Follow-up:** How do you start two async operations but await them later?</details>

**Q179. What are async generators for?**
<details><summary>Answer</summary>Producing a sequence of values that each require awaiting — paginated APIs, streamed lines, event batches. `for await...of` consumes them with backpressure built in, since the producer doesn't advance until the consumer asks.

**Follow-up:** Write one that paginates an API.</details>

**Q180. How would you time out a promise without leaking a timer?**
<details><summary>Answer</summary>

```js
function withTimeout(promise, ms) {
  let id;
  const timeout = new Promise((_, reject) => {
    id = setTimeout(() => reject(new DOMException(`Timed out after ${ms}ms`, 'TimeoutError')), ms);
  });
  return Promise.race([promise, timeout]).finally(() => clearTimeout(id));
}
```

**Follow-up:** What does this *not* do?  (Cancel the underlying operation — for that you need `AbortSignal.timeout(ms)` passed into it.)</details>

**Q181. Predict the output.**
```js
setTimeout(() => console.log('t1'), 0);
(async () => {
  console.log('a1');
  await Promise.resolve();
  console.log('a2');
})();
Promise.resolve().then(() => { console.log('p1'); return Promise.resolve(); })
  .then(() => console.log('p2'));
console.log('sync');
```
<details><summary>Answer</summary>`a1 sync a2 p1 p2 t1` — with `p2` notably late: returning a promise from `.then` costs two extra microtask ticks (assimilation), so it lands after `a2`.

**Follow-up:** Why does returning a promise cost two ticks rather than one?</details>

**Q182. How do you dedupe concurrent identical requests?**
<details><summary>Answer</summary>Cache the *promise*, not the result, keyed by request identity, and delete the entry when it settles:

```js
const inFlight = new Map();
const dedupe = (key, fn) => {
  if (!inFlight.has(key)) inFlight.set(key, fn().finally(() => inFlight.delete(key)));
  return inFlight.get(key);
};
```

**Follow-up:** Why delete in `finally` rather than keeping it?  (Keeping it makes this a result cache, which needs its own invalidation story.)</details>

**Q183. What's the problem with `await` inside a `for` loop?**
<details><summary>Answer</summary>It serialises independent work — N × latency instead of one round trip. It's correct when each iteration depends on the last, or when you're deliberately rate-limiting. Otherwise use `Promise.all` over a `map`, or a bounded pool if N is large enough to overwhelm the server.

**Follow-up:** How large is "large enough"?</details>

**Q184. `Promise.race` versus `Promise.any` — when do you want each?**
<details><summary>Answer</summary>`race` settles on the first *settlement*, so it's right for timeouts and cancellation, where a rejection is a meaningful outcome. `any` settles on the first *fulfilment*, ignoring rejections until all fail, so it's right for redundant sources — three mirrors, take whichever answers.

**Follow-up:** What does `any` reject with?  (`AggregateError` with an `errors` array.)</details>

**Q185. How does `queueMicrotask` differ from `Promise.resolve().then`?**
<details><summary>Answer</summary>Functionally almost identical, but `queueMicrotask` allocates no promise and, crucially, a throw inside it becomes an uncaught error rather than an unhandled rejection — which is often what you want for reporting. It also can't be chained, which makes intent clearer when you only need to defer.

**Follow-up:** Where have you used it?  (Rethrowing a caught error asynchronously so it reaches `window.onerror` without unwinding the current operation.)</details>

### Senior

**Q186. Explain how `await` is implemented under the hood.**
<details><summary>Answer</summary>An async function is a coroutine: the engine transforms it into a state machine that can suspend and resume. At an `await`, the remainder of the function is captured as a continuation and registered as a reaction on the awaited value's promise; the function returns control immediately with its own promise pending. When the awaited promise settles, the continuation is queued as a microtask and resumes with the value or throws inside the function. Generators use the same suspension machinery, which is why async functions were originally polyfilled with generators plus a driver.

**Follow-up:** Why can't you `await` at the top level of a CommonJS module?</details>

**Q187. How would you design a request layer for an app with flaky connectivity?**
<details><summary>Answer</summary>Layered, with each concern separate and composable: an `AbortController` per request with a timeout signal, retry with exponential backoff and full jitter on idempotent methods only, in-flight deduplication keyed by method plus URL plus body, and a circuit breaker that fails fast after a threshold rather than hammering a dead service. Writes go into a persistent outbox flushed on `online`, with idempotency keys so a retry can't double-charge. And every layer reports — you cannot debug flaky connectivity without knowing which retry attempt succeeded.

**Follow-up:** How do you make retries safe for POST?</details>

**Q188. `async`/`await` versus promise chains — is there a case for chains?**
<details><summary>Answer</summary>Yes, two. Fixed pipelines of transformations read well as `.then(parse).then(validate).then(save)` — naming each step is clearer than a sequence of awaits with intermediate variables. And `.finally()` for cleanup is more concise than a `try/finally` wrapping the whole body. Otherwise `await` wins: real stack traces, ordinary `try`/`catch`, loops and conditionals that work normally, and no accidental unreturned promise silently breaking the chain.

**Follow-up:** What's the stack-trace difference?</details>

**Q189. How do you debug an intermittent race condition in async code?**
<details><summary>Answer</summary>Make it deterministic before trying to fix it. That means finding the interleaving: add sequence-numbered logging with timestamps at every await boundary, then artificially delay one path to widen the window and reproduce reliably. Most of these turn out to be the same three shapes — a stale response overwriting a newer one, shared mutable state read across an await, or two code paths both initialising something. The structural fixes are request cancellation, capturing values before awaiting rather than re-reading after, and a single in-flight promise for initialisation.

**Follow-up:** What's the "stale response" fix?  (Abort the previous request, and tag responses so a superseded one is discarded.)</details>

**Q190. When is a callback still the right API over a promise?**
<details><summary>Answer</summary>Whenever the thing happens more than once — events, streams, subscriptions, progress reporting. A promise settles exactly once, so it structurally cannot model a sequence, and shoehorning one in produces a promise-per-event with no backpressure. Callbacks also win where synchronous invocation matters, like an array comparator or a `MutationObserver` callback. For a single async result, promises win on composition and error propagation.

**Follow-up:** What sits between them?  (Async iterators for pull-based sequences, observables for push-based.)</details>

**Q191. How would you prevent a long task from blocking interaction?**
<details><summary>Answer</summary>First ask whether it needs the main thread — parsing, diffing, and compression belong in a Web Worker, and moving them is a bigger win than any chunking. If it must be on the main thread (it touches the DOM), chunk it with a time-budget check and yield via `scheduler.yield()` where available, falling back to `setTimeout`. Prefer `scheduler.yield` over a bare `setTimeout` because it resumes with priority rather than going to the back of the queue behind whatever arrived while you ran.

**Follow-up:** How do you measure whether it worked?  (INP from field data, and long-task entries from a `PerformanceObserver`.)</details>

**Q192. What are the pitfalls of `Promise.all` at scale?**
<details><summary>Answer</summary>It starts everything at once, so 10,000 promises means 10,000 simultaneous connections — which the browser queues at six per origin anyway, while your server sees a spike. It also holds every result in memory until all settle, and one rejection discards the rest while the work continues uncancelled. At scale I'd use a bounded pool with a shared `AbortController`, streaming results out as they arrive rather than collecting them.

**Follow-up:** What's a sensible concurrency limit?  (Measured, not guessed — but 5–10 for HTTP is a reasonable starting point.)</details>

**Q193. Explain backpressure and how you'd handle it in JavaScript.**
<details><summary>Answer</summary>Backpressure is a consumer signalling that it can't keep up, so the producer must slow down rather than buffer without bound. Async iterators give it for free — the producer doesn't advance until `next()` is called — which is why `for await...of` over a paginated API is naturally safe. Streams implement it explicitly via the `desiredSize` of a writable's queue. The failure mode without it is an unbounded in-memory buffer, which is a memory leak that looks like a performance problem.

**Follow-up:** How does `pipeTo` handle it?</details>

**Q194. How do you test async code that involves timers and retries?**
<details><summary>Answer</summary>Fake timers, with the async variants — `advanceTimersByTimeAsync` and `runAllTimersAsync` — because the synchronous versions run timer callbacks without draining the microtasks those callbacks schedule, which either hangs the test or asserts too early. Assert on the *schedule*, not just the outcome: spy on `setTimeout` and check the delays are `[100, 200, 400]`, since a test that only checks eventual success passes whether you retried three times or thirty. And inject the clock rather than mocking `Date` globally, so the dependency is visible.

**Follow-up:** Why assert the call count on a non-retryable error?  (Because the outcome is identical whether you retried or not — only the count catches hammering a 400.)</details>

**Q195. Would you introduce RxJS to a codebase? Argue it.**
<details><summary>Answer</summary>Only where the domain is genuinely stream-shaped: real-time data, complex event coordination, or a search-as-you-type with switchMap-style cancellation that's fiddly to hand-roll. Its operators are excellent for exactly those problems. Against: a large API surface that every new hire must learn, debugging through operator chains is genuinely hard, and most of what teams use it for — one async result, or a simple debounce — is now cleaner with `async`/`await` plus `AbortController`. If a team already knows it and the app is event-heavy, that's a real argument for; "we might need it" is not.

**Follow-up:** What would you use instead for search-as-you-type?  (Debounce plus an `AbortController` per request — about fifteen lines.)</details>

**Q196. How does the event loop differ between browsers and Node, and when does it matter?**
<details><summary>Answer</summary>Node has explicit phases with `setImmediate` and `process.nextTick`, where `nextTick` drains before promise microtasks and can starve I/O if used recursively. Browsers have rendering integrated into the loop — `requestAnimationFrame` before paint, and the microtask queue draining before rendering, which is why a microtask loop blocks paint just as effectively as a `while` loop. It matters when porting code that relied on ordering, and when a library uses `nextTick` for something that should be a microtask.

**Follow-up:** Why does `setTimeout(fn, 0)` behave differently in Node?</details>

**Q197. What's your approach to error handling across an async boundary?**
<details><summary>Answer</summary>Catch where you can add information or make a decision, and nowhere else — a `catch` that logs and rethrows at every level produces five log lines for one failure. Wrap with `cause` when crossing a layer boundary so the original stack survives: `new AppError('Failed to save draft', { cause: err })`. Distinguish expected failures (a 404, a validation error) which belong in the return type, from exceptional ones which should propagate to a boundary handler. And always have that boundary: an `unhandledrejection` listener, an error boundary, a process handler.

**Follow-up:** How do you avoid losing the original stack?</details>

**Q198. How would you implement cancellation for a multi-step async workflow?**
<details><summary>Answer</summary>One `AbortController` for the workflow, its signal threaded into every step — passed to `fetch`, checked at each loop boundary, and combined with per-step timeouts via `AbortSignal.any`. Steps that have already committed side effects need compensating actions, so I'd model the workflow as a list of steps each with a `run` and an `undo`, and unwind on abort. The part people miss is that aborting doesn't undo — a cancelled multi-step write leaves partial state unless you designed for it.

**Follow-up:** How does that differ from a database transaction?</details>

**Q199. Why can promises not be cancelled, and what would it have taken?**
<details><summary>Answer</summary>Because a promise is a value representing a result, and multiple consumers can await the same one — letting any of them cancel it would let one consumer break another. The cancellation proposal was withdrawn for exactly this reason, plus disagreement over whether cancellation is an error or a third state. `AbortController` resolves it by separating the *operation* (cancellable, owned by whoever started it) from the *result* (a plain promise), which is the right factoring.

**Follow-up:** So what does `abort()` do to the promise?  (Rejects it with `AbortError` — the promise itself was never cancelled.)</details>

**Q200. A page makes 200 requests on load. How do you approach fixing it?**
<details><summary>Answer</summary>Measure the waterfall first, because the count matters less than the shape — 200 parallel requests to one origin serialise at six at a time, and 200 sequential ones are catastrophically worse. Then look for the usual causes: an N+1 pattern where a list fetches each item's detail, no batching endpoint, missing caching, and third-party scripts. The fixes in order of payoff are a batch endpoint or GraphQL-style single query, HTTP caching with proper `Cache-Control`, deferring anything below the fold to an `IntersectionObserver`, and deduplicating identical in-flight requests. I'd also check whether all 200 are needed for first render, because usually about five are.

**Follow-up:** Does HTTP/2 make the count irrelevant?  (It removes the connection limit, not the per-request server cost or the bytes.)</details>

---

## 7. DOM, Events, and the Browser

### Junior

**Q201. `querySelector` vs `getElementById` vs `getElementsByClassName`?**
<details><summary>Answer</summary>`querySelector` takes any CSS selector and returns the first match or `null`. `getElementById` is fastest but id-only. `getElementsByClassName` returns a *live* `HTMLCollection` that updates as the DOM changes, unlike `querySelectorAll`'s static `NodeList`.

**Follow-up:** Why does the live/static distinction cause bugs?  (Removing elements while iterating a live collection skips items.)</details>

**Q202. What are the three event propagation phases?**
<details><summary>Answer</summary>Capture (window down to the target's parent), target, then bubble (back up to window). Listeners default to the bubble phase; pass `true` or `{ capture: true }` for capture.

**Follow-up:** Which events don't bubble?  (`focus`, `blur`, `load`, `mouseenter`, `mouseleave`.)</details>

**Q203. `e.target` vs `e.currentTarget`?**
<details><summary>Answer</summary>`target` is what the event originated on; `currentTarget` is the element whose listener is running. Clicking a `<span>` inside a `<button>` gives `target = span`, `currentTarget = button`.

**Follow-up:** Which do you use for delegation?  (`target`, then `closest()`.)</details>

**Q204. How do you stop a form submitting and handle it yourself?**
<details><summary>Answer</summary>`form.addEventListener('submit', e => { e.preventDefault(); … })`. Read values with `new FormData(form)`, and let the browser's constraint validation run first via `form.checkValidity()`.

**Follow-up:** Why listen on the form rather than the button?  (Enter-key submission and programmatic submits don't go through the button.)</details>

**Q205. What does `preventDefault` do that `stopPropagation` doesn't?**
<details><summary>Answer</summary>`preventDefault` cancels the browser's default action (navigation, submission, text entry) but the event keeps travelling. `stopPropagation` halts travel but the default action still happens. They're independent.

**Follow-up:** What does `stopImmediatePropagation` add?  (It also stops other listeners on the same element.)</details>

**Q206. What are `defer` and `async` on a script tag?**
<details><summary>Answer</summary>Both download in parallel with parsing. `defer` executes after parsing, in document order. `async` executes as soon as it arrives, in no guaranteed order. Modules are deferred by default.

**Follow-up:** Which for an analytics snippet with no dependencies?  (`async`.)</details>

**Q207. How do you add and remove classes?**
<details><summary>Answer</summary>`el.classList.add/remove/toggle/contains/replace`. `toggle(name, force)` takes a boolean second argument, which is cleaner than an `if`. Avoid `className = ` — it clobbers existing classes.

**Follow-up:** How would you toggle based on a condition?  (`el.classList.toggle('active', isActive)`.)</details>

**Q208. What is `localStorage` and what are its limits?**
<details><summary>Answer</summary>Synchronous, string-only, origin-scoped key-value storage of roughly 5MB that persists until cleared. It blocks the main thread, throws `QuotaExceededError` when full, and can be entirely unavailable in private mode or a sandboxed iframe.

**Follow-up:** Why shouldn't you store an auth token in it?</details>

**Q209. What's the difference between `innerHTML`, `textContent`, and `innerText`?**
<details><summary>Answer</summary>`innerHTML` parses markup (an XSS sink). `textContent` gets or sets raw text, including hidden elements, with no parsing. `innerText` is layout-aware — it respects CSS visibility and triggers a reflow to compute.

**Follow-up:** Which is fastest and why?  (`textContent` — no parsing, no layout.)</details>

**Q210. How do you create and insert an element?**
<details><summary>Answer</summary>`document.createElement`, set properties, then `append`, `prepend`, `before`, `after`, or `replaceChildren`. For many nodes, build a `DocumentFragment` and insert once, so layout runs once rather than per node.

**Follow-up:** What's the difference between `append` and `appendChild`?  (`append` accepts multiple nodes and strings, and returns `undefined`.)</details>

### Mid

**Q211. Explain reflow and repaint, and what triggers each.**
<details><summary>Answer</summary>Reflow (layout) recalculates geometry; repaint redraws pixels without changing geometry. Changing `width`, `font-size`, or DOM structure reflows; changing `color` or `background` repaints; `transform` and `opacity` do neither — they're composited.

**Follow-up:** Which is more expensive and why?  (Reflow — it can cascade to the whole subtree or document.)</details>

**Q212. What is layout thrashing and how do you fix it?**
<details><summary>Answer</summary>Interleaving layout reads with writes, so each read forces a synchronous recalculation. Fix by batching all reads, then all writes. The read-forcing properties are `offsetTop`/`Width`/`Height`, `scrollTop`, `clientWidth`, `getBoundingClientRect`, and `getComputedStyle`.

**Follow-up:** How would you enforce that structurally?  (A frame scheduler with separate read and write phases.)</details>

**Q213. How does event delegation work, and when is it wrong?**
<details><summary>Answer</summary>One listener on a stable ancestor, matching with `e.target.closest(selector)` and checking containment. It's wrong for non-bubbling events (`focus`, `mouseenter`), when you need `stopPropagation` semantics per item, or for a single stable element where a direct listener is simpler.

**Follow-up:** How do you delegate `focus`?  (Use `focusin`, which bubbles.)</details>

**Q214. What is a `MutationObserver` for?**
<details><summary>Answer</summary>Reacting to DOM changes you don't control — a third-party widget injecting nodes, or a CMS rewriting markup. It batches records and delivers them as microtasks, which is why it's cheaper than polling.

**Follow-up:** How do you avoid an infinite loop when the callback mutates the DOM?  (Disconnect, mutate, reconnect — or filter records by target.)</details>

**Q215. `IntersectionObserver` — how would you build infinite scroll with it?**
<details><summary>Answer</summary>Observe a sentinel element after the last item, with `rootMargin` so loading starts before it's visible. On intersection, `unobserve` immediately (or guard with a loading flag), fetch, append, then re-observe the new sentinel — otherwise the sentinel stays intersecting and fires repeatedly.

**Follow-up:** Why `unobserve` rather than just a flag?  (Both work; unobserving also stops the callback churn while loading.)</details>

**Q216. How do you clean up event listeners in a component?**
<details><summary>Answer</summary>One `AbortController` per component lifetime, its signal passed to every `addEventListener`, and `abort()` on teardown. It handles listeners, `fetch`, and timers uniformly, and it can't drift out of sync the way paired add/remove calls do.

**Follow-up:** Why can't you remove an arrow function passed inline?  (Removal matches by identity, and each evaluation creates a new function.)</details>

**Q217. What does `{ passive: true }` do?**
<details><summary>Answer</summary>Promises you won't call `preventDefault()`, so the browser can start scrolling without waiting for your handler to return. Essential on `touchstart`, `touchmove`, and `wheel`; browsers now default those to passive on document-level targets.

**Follow-up:** What happens if you call `preventDefault` anyway?  (It's ignored, with a console warning.)</details>

**Q218. How do you create and dispatch a custom event?**
<details><summary>Answer</summary>`el.dispatchEvent(new CustomEvent('cart:add', { detail, bubbles: true, cancelable: true }))`. Note `bubbles` defaults to `false` for `CustomEvent`, and `dispatchEvent` returns `false` if a listener called `preventDefault`.

**Follow-up:** What does `composed: true` do?  (Lets the event cross shadow-DOM boundaries.)</details>

**Q219. `pushState` versus `replaceState`, and why doesn't routing "just work"?**
<details><summary>Answer</summary>`pushState` adds a history entry; `replaceState` modifies the current one. Neither fires `popstate` — only user navigation does — so a router must call its own render after pushing. That asymmetry is why hand-rolled routers work on back/forward but not on link clicks, or vice versa.

**Follow-up:** How do you intercept link clicks for a router?</details>

**Q220. How would you virtualise a list of 100,000 rows?**
<details><summary>Answer</summary>Render only the visible window plus a small buffer, with a spacer element sized to the total scroll height so the scrollbar is correct. Compute the window from `scrollTop` divided by row height, reuse a pool of row elements rather than recreating them, and coalesce scroll updates into one `requestAnimationFrame`. For variable heights you need measured offsets and a binary search, which is where it gets genuinely hard — try `content-visibility: auto` first.

**Follow-up:** What breaks with variable row heights?</details>

**Q221. What is the shadow DOM and what does it give you?**
<details><summary>Answer</summary>An encapsulated subtree with its own scope: styles inside don't leak out and outside styles don't leak in (except inherited properties and CSS custom properties). It's the basis of web components, and the encapsulation is real enough that `querySelector` from outside can't see into a closed root.

**Follow-up:** How do you style a shadow root from outside?  (CSS custom properties and `::part()`.)</details>

**Q222. Explain `requestAnimationFrame` versus `setTimeout` for animation.**
<details><summary>Answer</summary>rAF runs immediately before the next paint at the display's refresh rate, so frames align with rendering and it pauses entirely in a hidden tab. `setTimeout(fn, 16)` drifts, doesn't align with paint, and assumes 60Hz. Use the rAF timestamp for delta-time so speed is refresh-rate independent.

**Follow-up:** What's better than both?  (CSS transitions or the Web Animations API — they run off the main thread.)</details>

**Q223. What are the Core Web Vitals and what moves each?**
<details><summary>Answer</summary>LCP (largest contentful paint) — server response, render-blocking resources, image priority and size. INP (interaction to next paint) — long tasks and heavy handlers. CLS (cumulative layout shift) — images without dimensions, injected content, font swaps.

**Follow-up:** Why did INP replace FID?  (FID measured only queue delay, so pages that felt terrible scored well.)</details>

**Q224. How do you handle a form accessibly?**
<details><summary>Answer</summary>Every input has a real `<label for>`, errors are associated with `aria-describedby` and announced via a live region, invalid fields get `aria-invalid`, and focus moves to the first error on failed submit. Use the constraint validation API for the checks but render your own messages, since native bubbles aren't stylable or reliably announced.

**Follow-up:** Why is `placeholder` not a label?</details>

**Q225. What is `structuredClone` used for in a browser context?**
<details><summary>Answer</summary>It's the algorithm behind `postMessage`, IndexedDB storage, and the History API's state — exposed directly as a function since 2022. It deep-copies handling cycles, `Map`, `Set`, `Date`, and `ArrayBuffer`, but throws on functions and DOM nodes.

**Follow-up:** How do you avoid the copy cost for a large buffer?  (Transfer it: `postMessage(buf, [buf])`.)</details>

### Senior

**Q226. How would you diagnose "the page feels janky when scrolling"?**
<details><summary>Answer</summary>Record a Performance profile while scrolling and look at the frames track first — dropped frames tell you whether it's the main thread or the compositor. Main-thread jank means a scroll listener doing layout reads or heavy work, so I'd check for non-passive listeners and forced synchronous layout (DevTools flags those). Compositor jank usually means too many paint areas or an animated property that isn't `transform`/`opacity`. Node count matters too — 10,000 elements with complex selectors makes style recalculation itself the bottleneck, which `content-visibility` or virtualisation fixes.

**Follow-up:** What does the "Layout Shift" region in the profile tell you?</details>

**Q227. When would you use a web component over a framework component?**
<details><summary>Answer</summary>When the consumer's framework is unknown or plural — a design system used by three teams on three stacks, or a widget embedded in customers' sites. The encapsulation is genuine and the thing outlives framework churn. Against: worse ergonomics for state and data flow, server-side rendering is awkward, form participation needs explicit work, and accessibility inside a shadow root has real sharp edges. Inside a single React or Vue app, a framework component is simply better tooling.

**Follow-up:** What's the SSR story?</details>

**Q228. How do you make a large DOM update feel instant?**
<details><summary>Answer</summary>Do less: render only what's visible, and let the early-exit do the work rather than optimising a full render. Then batch — build a `DocumentFragment` or use `replaceChildren` so layout runs once, and split reads from writes so nothing forces synchronous layout. Coalesce into one `requestAnimationFrame` so many state changes produce one paint. If the computation itself is heavy, move it to a worker and post only the diff back. The perceptual trick worth knowing is that showing a correct skeleton immediately beats showing the real thing 200ms later.

**Follow-up:** Where does `startViewTransition` fit?</details>

**Q229. What's your approach to third-party scripts?**
<details><summary>Answer</summary>Treat them as untrusted code running with your privileges, because that's what they are. Load them `async` and never in the critical path, sandbox in an iframe where the functionality allows, and constrain them with CSP — `connect-src` in particular, so a compromised tag can't exfiltrate freely. Budget them explicitly: measure the LCP and INP cost of each, and make the owning team justify it against that number. Subresource Integrity where the file is versioned, and a documented removal owner for every tag, because the usual failure is a script nobody remembers adding.

**Follow-up:** How do you measure one script's INP contribution?</details>

**Q230. How would you build an accessible autocomplete from scratch?**
<details><summary>Answer</summary>Start from the ARIA combobox pattern: `role="combobox"` with `aria-expanded`, `aria-controls`, and `aria-activedescendant` pointing at the highlighted option, and the list as `role="listbox"` with `role="option"` children. Keyboard support is the whole feature — arrows to move, Enter to select, Escape to close and restore, Home/End, and typing must never lose focus from the input. Announce result counts through a polite live region, debounce the query, and abort superseded requests so a slow response can't overwrite a newer one. I'd reach for an existing tested implementation before writing this, because the number of ways to get it subtly wrong is large.

**Follow-up:** Why `aria-activedescendant` rather than moving focus?</details>

---

## 8. Modules and Tooling

**Q231. *(Junior)* ESM vs CommonJS — name four differences.**
<details><summary>Answer</summary>ESM is static and parse-time, asynchronous, gives live bindings, and is tree-shakeable. CommonJS is dynamic and runtime, synchronous, copies values, and mostly isn't. ESM also supports top-level `await`; CJS has `__dirname` and `require` in exchange.

**Follow-up:** Why can't `require` load an ESM module?</details>

**Q232. *(Junior)* What is a default export and when would you avoid it?**
<details><summary>Answer</summary>One unnamed export per module, imported with any name. Avoid it when consistency of naming matters — named exports are refactorable, autocomplete better, and can't be silently renamed at each import site. A single-class or single-function module is the reasonable case for default.

**Follow-up:** Can you have both?  (Yes.)</details>

**Q233. *(Junior)* What is tree shaking?**
<details><summary>Answer</summary>Eliminating exports nothing imports, made possible by ESM's static structure. It's defeated by dynamic `require`, top-level side effects the bundler can't prove safe to drop, and a missing `sideEffects: false` in `package.json`.

**Follow-up:** Why can't CommonJS be tree-shaken?</details>

**Q234. *(Junior)* What does dynamic `import()` give you?**
<details><summary>Answer</summary>Runtime loading returning a promise, which is what enables code splitting — a route or a heavy feature loaded only when reached. Unlike static `import`, it can take a computed specifier and appear anywhere.

**Follow-up:** What's the cost?  (A network round trip at the moment of use, so prefetch when you can predict it.)</details>

**Q235. *(Mid)* What are live bindings, and why do they matter?**
<details><summary>Answer</summary>An ESM import is a reference to the exporting module's binding, not a copy of its value — so if the exporter reassigns it, importers see the new value. CommonJS copies at require time, so they don't. It matters for circular dependencies and for modules exporting mutable state.

**Follow-up:** Can you assign to an imported binding?  (No — imports are read-only.)</details>

**Q236. *(Mid)* How do circular dependencies behave in ESM?**
<details><summary>Answer</summary>They're allowed. Modules are instantiated (bindings created) before evaluation, so a cycle resolves as long as nothing *uses* a binding before the other module has evaluated it. Function declarations are hoisted so they work; `const` values are in the TDZ and throw.

**Follow-up:** How would you break a cycle?  (Extract the shared piece into a third module.)</details>

**Q237. *(Mid)* What are import maps and import attributes?**
<details><summary>Answer</summary>Import maps let the browser resolve bare specifiers (`import x from 'lodash'`) without a bundler. Import attributes declare a module's type: `import data from './d.json' with { type: 'json' }` — required, because a server could otherwise serve JavaScript where JSON was expected.

**Follow-up:** Why is the type attribute a security feature?</details>

**Q238. *(Mid)* Explain the `exports` field in `package.json`.**
<details><summary>Answer</summary>It declares a package's public entry points and encapsulates everything else — deep imports into unlisted paths fail. It supports conditional resolution (`import`, `require`, `browser`, `node`, `types`), which is how a package ships both ESM and CJS builds.

**Follow-up:** What's the dual-package hazard?  (Both builds loaded at once, so `instanceof` fails across them and module-level state is duplicated.)</details>

**Q239. *(Mid)* Transpiling versus polyfilling — which fixes what?**
<details><summary>Answer</summary>Transpiling rewrites *syntax* the engine can't parse — `?.`, `#private`, class fields. Polyfilling adds *missing APIs* at runtime — `Array.at`, `Object.groupBy`. Syntax must be transpiled because a parse error kills the file before any shim runs; some things can be neither, like `Proxy` and true `WeakMap` semantics.

**Follow-up:** What drives both?  (`browserslist`.)</details>

**Q240. *(Mid)* What is a source map and how does it break?**
<details><summary>Answer</summary>A mapping from built output back to original sources, so stack traces and debugger breakpoints refer to code you wrote. It breaks when maps aren't uploaded to your error tracker, when a build step in the middle of the chain doesn't forward them, or when they're served publicly and you didn't intend to publish your source.

**Follow-up:** Should source maps be public?</details>

**Q241. *(Senior)* How would you decide between Vite, webpack, and no bundler?**
<details><summary>Answer</summary>Vite for almost any new app — native ESM in development means startup doesn't scale with codebase size, and Rollup for production is well-understood. Webpack where an existing configuration encodes years of hard-won requirements, or where a loader ecosystem feature has no equivalent; migrating a working webpack build is often negative value. No bundler is genuinely viable now for small projects using import maps and HTTP/2, and it's the right call for a demo or an internal tool — but you give up tree shaking, and dependency count is what usually forces the issue.

**Follow-up:** What would make you migrate off webpack?</details>

**Q242. *(Senior)* How do you keep a monorepo's build times sane?**
<details><summary>Answer</summary>Content-addressed caching and task graphs — Turborepo or Nx — so unchanged packages never rebuild and CI restores from cache rather than recomputing. Enforce package boundaries so the dependency graph stays shallow, because a single shared "utils" package that everything imports means every change invalidates everything. Only build what changed: derive the affected set from the graph and the diff. And measure, because the usual culprit turns out to be one package with a slow type-check that everything depends on.

**Follow-up:** How do you stop a shared package becoming a bottleneck?</details>

**Q243. *(Senior)* What's your position on dependencies?**
<details><summary>Answer</summary>Every dependency is code you're responsible for and a maintainer whose account could be compromised, so the bar is "does this save more than it costs over a year". Small utilities I'd write — a debounce is fifteen lines and I own it forever. Hard, well-specified problems I'd take the dependency: date handling, a sanitiser, a test runner, cryptography. What I check before adding: bundle cost, transitive count, last release date, and whether the maintainer is one person. And I commit lockfiles, review lockfile diffs, and disable install scripts.

**Follow-up:** How do you handle an unmaintained dependency you depend on?</details>

**Q244. *(Senior)* How would you migrate a large CommonJS codebase to ESM?**
<details><summary>Answer</summary>Incrementally, leaf-first, with both formats coexisting — Node supports mixed graphs, so a big-bang cutover is unnecessary risk. Start with modules that import nothing internal, convert upward, and use `.mjs` or a package-level `"type": "module"` per directory so the change is explicit rather than global. The real work isn't the syntax: it's `__dirname` and `require.resolve` replacements, the loss of synchronous `require` in conditional code paths, and dependencies that only ship CJS. And the dual-package hazard, which is why I'd avoid publishing both formats from one package if I can ship ESM-only.

**Follow-up:** What's the `__dirname` equivalent?  (`import.meta.dirname`, or `path.dirname(fileURLToPath(import.meta.url))` on older Node.)</details>

**Q245. *(Senior)* What belongs in CI for a JavaScript project?**
<details><summary>Answer</summary>Fast and blocking: lint, type-check, unit and integration tests, and a build. Then the things teams skip and regret: a bundle-size budget that fails the build, a dependency audit, and a small E2E suite over the critical journeys. Everything should run on the changed set where possible, and the whole blocking pipeline should finish in under ten minutes, because a slow pipeline gets bypassed. Anything flaky gets fixed or quarantined the same week — a suite people don't trust is worse than no suite.

**Follow-up:** What do you do with a flaky test you can't fix today?</details>

---

## 9. Performance and Memory

**Q246. *(Junior)* What's the first thing you do about a performance problem?**
<details><summary>Answer</summary>Measure. A Performance profile or field data tells you where time actually goes; intuition reliably picks the wrong thing. Only then change code, and measure again to confirm.

**Follow-up:** What tool, specifically?</details>

**Q247. *(Junior)* Why `performance.now()` over `Date.now()`?**
<details><summary>Answer</summary>It's monotonic — unaffected by clock adjustments — and sub-millisecond. `Date.now()` has ~1ms granularity and can jump backwards when the system clock syncs.

**Follow-up:** What's `performance.mark` for?</details>

**Q248. *(Junior)* How does JavaScript free memory?**
<details><summary>Answer</summary>A garbage collector reclaims anything unreachable from a GC root — the global object, the call stack, and internal engine references. There's no `free`, and reachability handles reference cycles that reference counting couldn't.

**Follow-up:** Name a GC root.</details>

**Q249. *(Junior)* Name the four classic memory leaks.**
<details><summary>Answer</summary>Accidental globals, forgotten timers, detached DOM nodes still referenced from JavaScript, and unremoved event listeners. The last two are the common ones in SPAs, and both are fixed by an `AbortController` per component lifetime.

**Follow-up:** Which one does strict mode prevent?</details>

**Q250. *(Junior)* What's a debounce, and when do you use it over a throttle?**
<details><summary>Answer</summary>Debounce fires once after the calls stop; throttle fires at most once per interval during a burst. Search-as-you-type is debounce (one request when typing stops); a scroll position readout is throttle (regular updates during the scroll).

**Follow-up:** Where do you create the debounced function?  (Once, outside the handler.)</details>

**Q251. *(Mid)* Explain generational garbage collection.**
<details><summary>Answer</summary>The heap is split by age. New space is small and collected by a scavenger that copies survivors to a second semi-space — its cost is proportional to what *survived*, so short-lived garbage is nearly free. Objects surviving two scavenges are promoted to old space, collected by mark-compact, which is more expensive but infrequent and now largely concurrent.

**Follow-up:** What does that imply about allocating a million temporary objects?</details>

**Q252. *(Mid)* What is a hidden class and why does it matter?**
<details><summary>Answer</summary>V8's internal shape descriptor for an object, letting property access compile to a fixed offset rather than a hash lookup. Objects created with the same properties in the same order share one, so a call site sees one shape and stays monomorphic. Adding properties later or in a different order creates new shapes; at five or more the inline cache is abandoned and every access becomes a lookup.

**Follow-up:** How would you keep a hot path monomorphic?</details>

**Q253. *(Mid)* How do you use a heap snapshot to find a leak?**
<details><summary>Answer</summary>Snapshot, perform the suspect action N times, snapshot again, switch to Comparison view, and sort by delta. Then select a growing object and read the **Retainers** pane bottom-up: it names the exact chain from a GC root, which *is* the answer. Three snapshots with the action repeated between each distinguishes a leak (linear growth) from a warming cache (grows once, flattens).

**Follow-up:** What does the "Detached" filter show?</details>

**Q254. *(Mid)* What is `content-visibility` and when does it help?**
<details><summary>Answer</summary>`content-visibility: auto` skips rendering work — style, layout, paint — for off-screen elements, with no JavaScript. Pair it with `contain-intrinsic-size` so scrollbars behave. It's the cheapest first attempt at a long-list problem; a real virtualiser is still better for very large or variable-height lists.

**Follow-up:** What's the accessibility caveat?  (Content is still findable and in the a11y tree, unlike `display: none` — that's the point.)</details>

**Q255. *(Mid)* When is a Web Worker worth it?**
<details><summary>Answer</summary>Any CPU-bound work over ~50ms that doesn't need the DOM: parsing large files, diffing, image processing, compression, search indexing, cryptography. The cost is the message boundary — structured cloning a large result can eat the gain, so transfer buffers instead of copying them.

**Follow-up:** What can't a worker do?</details>

**Q256. *(Mid)* What does an unbounded cache cost you?**
<details><summary>Answer</summary>It's a memory leak with good intentions — every entry is retained forever, and "just a few KB each" is how a 2GB heap starts. Bound every cache by size (LRU) or time (TTL), and use `WeakMap` when keys are objects so entries die with their keys.

**Follow-up:** How would you implement an LRU in twenty lines?  (A `Map` — insertion order is LRU order if you delete and re-set on access.)</details>

**Q257. *(Mid)* Which JavaScript performance advice is out of date?**
<details><summary>Answer</summary>Caching `array.length` in loops (the JIT hoists it), `++i` over `i++` (identical), `try`/`catch` preventing optimisation (untrue since 2016), string concatenation needing an array join (engines use ropes), and bitwise tricks beating `Math.floor` (same speed, and wrong above 2³¹). What still holds: send less JavaScript, avoid unnecessary layout, don't block the main thread, pick the right data structure.

**Follow-up:** How would you disprove one of those in five minutes?</details>

**Q258. *(Mid)* Explain the difference between allocation-rate and retention problems.**
<details><summary>Answer</summary>High allocation rate causes frequent scavenges — visible as GC time in a profile, fixed by reusing buffers or reducing per-frame garbage. Retention means objects stay reachable — visible as a rising heap floor, fixed by finding the retainer. They need different tools: allocation sampling for the first, heap snapshots for the second, and confusing them wastes days.

**Follow-up:** Which one does object pooling address?</details>

**Q259. *(Mid)* Why is `delete` bad for performance?**
<details><summary>Answer</summary>It transitions the object out of its hidden class into dictionary mode, so every later property access on it is a hash lookup rather than a fixed offset — and it frees nothing that setting the property to `undefined` wouldn't. Use `undefined`, destructuring to rebuild, or a `Map` for genuinely dynamic keys.

**Follow-up:** When is `delete` actually required?</details>

**Q260. *(Senior)* How do you set and defend a performance budget?**
<details><summary>Answer</summary>Derive it from the user, not from a round number: pick a target device and connection from your analytics, decide what LCP and INP the experience needs, and work back to a JavaScript byte budget. Then enforce it in CI so a regression fails the build rather than arriving in a retrospective, because budgets erode one 12KB dependency at a time. Defending it means having the number in the PR that adds the dependency — "this costs 40KB and 90ms of INP on a mid-tier Android" is a conversation, "we should be careful about bundle size" is not.

**Follow-up:** What do you do when the business wants a feature that blows the budget?</details>

**Q261. *(Senior)* Walk me through diagnosing a 600ms INP.**
<details><summary>Answer</summary>Break it into its three phases from the `event` performance entries — input delay, processing, presentation — because the fix is completely different for each and guessing wastes the investigation. High input delay means the click queued behind a long task, so I'd look at what else runs on interaction (hydration, third-party, an unrelated timer). High processing means the handler itself, which a profile localises immediately. High presentation means the render, usually a full re-render where a narrow update would do, or forced synchronous layout. I've most often found it's presentation, and most often the fix is doing less work rather than doing it faster.

**Follow-up:** What's the field-versus-lab difference here?</details>

**Q262. *(Senior)* A colleague wants object pooling for a particle system. Do you agree?**
<details><summary>Answer</summary>Possibly — particles are one of the few genuine cases, since thousands of short-lived objects per frame does show up as scavenge time. But I'd want the profile first, and I'd bound the pool, because a pool is by construction a set of objects you've made immortal. The real risk is correctness: incomplete reset logic produces state-bleed bugs that are far harder to find than a 0.3ms GC pause. Outside a per-frame loop, the answer is no.

**Follow-up:** How would you verify the reset is complete?</details>

**Q263. *(Senior)* How do you approach a memory leak in production you can't reproduce locally?**
<details><summary>Answer</summary>Instrument rather than guess: report `performance.memory` or `measureUserAgentSpecificMemory` samples with session duration and route history, so you can find which user journeys correlate with growth. Dev tooling — HMR, source-map registries, an open DevTools — changes both retention and V8's ability to elide closure variables, which is exactly why local reproduction fails. Once a journey is implicated, reproduce it with a production build and no DevTools open during the action, taking snapshots only at the boundaries. In Node, `writeHeapSnapshot` on a signal gets you a real snapshot from the running process.

**Follow-up:** What do you do if the growth is in a third-party script?</details>

**Q264. *(Senior)* When is it right to *not* optimise?**
<details><summary>Answer</summary>When you haven't measured, when the code isn't on a hot path, and when the optimisation costs readability that future maintainers will pay for repeatedly. Also when the win is real but small against a larger problem — shaving 3ms off a function inside a page with a 4-second LCP is misallocated effort. My filter is whether a user could notice, and whether I can state the number before and after; if I can't, I'm not optimising, I'm guessing.

**Follow-up:** How do you push back on a "make it faster" ticket with no numbers?</details>

**Q265. *(Senior)* How would you make a data-heavy dashboard performant?**
<details><summary>Answer</summary>Move data work off the main thread — parsing, aggregation, and downsampling in a worker, posting back only what the current view needs. Render only visible rows and points; a chart showing 100,000 samples in 800 pixels should downsample to about 800, which is both faster and visually identical. Consolidate polling into one scheduler so the CPU wakes once rather than six times, pause it when the tab is hidden, and refresh once on becoming visible. Then bound everything that accumulates — chart buffers, WebSocket history, notification lists — because a dashboard left open for eight hours is the classic leak.

**Follow-up:** Why does downsampling not lose information here?</details>

---

## 10. Security

**Q266. *(Junior)* What is XSS?**
<details><summary>Answer</summary>Attacker-controlled JavaScript executing in your origin, which gives it the user's session, storage, and the ability to make authenticated requests. Three types: stored (in your database), reflected (echoed from the request), and DOM-based (never reaches the server).

**Follow-up:** Name three sinks.</details>

**Q267. *(Junior)* Why is `innerHTML` dangerous?**
<details><summary>Answer</summary>It parses markup, so untrusted input can create elements with event handlers — `<img src=x onerror=alert(1)>` executes immediately. `<script>` tags inserted this way *don't* run, which misleads people into thinking it's safe.

**Follow-up:** What's the safe alternative?</details>

**Q268. *(Junior)* What's the difference between escaping and sanitising?**
<details><summary>Answer</summary>Escaping converts characters so the value renders as data, and is context-specific — HTML body, attribute, URL, and JS contexts each differ. Sanitising parses markup and removes what isn't allowlisted. Escape when no markup is wanted; sanitise only when some is required, and always with a real parser.

**Follow-up:** Why is a regex-based sanitiser wrong?</details>

**Q269. *(Junior)* Where should you store an auth token?**
<details><summary>Answer</summary>An `HttpOnly`, `Secure`, `SameSite=Lax` cookie, because script can't read it — so an XSS doesn't become an account takeover. `localStorage` is readable by any XSS in one line, and there's no mitigation for that.

**Follow-up:** But doesn't a cookie bring CSRF?</details>

**Q270. *(Junior)* What is CSRF and what stops it?**
<details><summary>Answer</summary>A victim's browser sending an authenticated request they didn't intend, because cookies attach automatically. `SameSite=Lax` blocks the classic cross-site form POST; add a CSRF token and an `Origin` check for defence in depth, and never let a GET change state.

**Follow-up:** Why isn't `SameSite` alone enough?</details>

**Q271. *(Mid)* What is CSP and why does `unsafe-inline` defeat it?**
<details><summary>Answer</summary>An allowlist of what may load and execute, delivered as a header. `script-src` is the directive that stops XSS from running — but an injected payload *is* inline script, so allowing inline script allows the payload. Use a per-response nonce with `'strict-dynamic'` instead.

**Follow-up:** Why must the nonce be per-response?</details>

**Q272. *(Mid)* Which CSP directives do people forget?**
<details><summary>Answer</summary>`object-src 'none'` (kills plugin-based bypasses), `base-uri 'none'` (stops `<base href>` injection redirecting every relative URL), `form-action 'self'` (stops an injected form POSTing credentials elsewhere), and `frame-ancestors` for clickjacking. `connect-src` limits where exfiltrated data can go.

**Follow-up:** How do you roll CSP out without an outage?</details>

**Q273. *(Mid)* What are Trusted Types?**
<details><summary>Answer</summary>A CSP-enforced mechanism making the dangerous sinks refuse plain strings, so `el.innerHTML = untrusted` throws. You define one policy whose `createHTML` sanitises, and that becomes the single audited chokepoint — turning "did we escape everywhere?" into a question the browser answers.

**Follow-up:** How do you adopt it in an existing app?</details>

**Q274. *(Mid)* What is prototype pollution and how do you fix it?**
<details><summary>Answer</summary>Writing to `Object.prototype` so a property appears on every object — usually via a recursive merge over untrusted JSON containing `__proto__`. Three fixes, all worth applying: reject `__proto__`/`constructor`/`prototype` keys, merge into `Object.create(null)`, and freeze `Object.prototype` at boot so a miss throws.

**Follow-up:** Is `JSON.parse` itself vulnerable?</details>

**Q275. *(Mid)* Why is `Math.random()` unsuitable for tokens?**
<details><summary>Answer</summary>It's a PRNG optimised for speed, not unpredictability — its internal state can be recovered from a handful of outputs, so future values become predictable. Use `crypto.getRandomValues` or `crypto.randomUUID`.

**Follow-up:** What else is it wrong for?  (Nonces, password generation, shuffle in anything adversarial.)</details>

**Q276. *(Mid)* Explain why CORS is not authorisation.**
<details><summary>Answer</summary>CORS relaxes the same-origin policy so a browser will let script *read* a cross-origin response. It doesn't authenticate anyone, doesn't prevent the request being sent, and doesn't apply outside a browser at all — `curl` ignores it entirely. Authorisation is the server checking who's asking.

**Follow-up:** What's wrong with reflecting the `Origin` header?</details>

**Q277. *(Mid)* What's the `javascript:` URL sink and how do you close it?**
<details><summary>Answer</summary>Assigning an unvalidated URL to `href`, `src`, `action`, or `window.open` — `javascript:alert(1)` executes on activation. Close it with a protocol allowlist via `new URL(value, base).protocol`, never a blocklist, since `\tjavascript:` and mixed case defeat string matching while `URL` normalises them.

**Follow-up:** Why allowlist rather than blocklist?</details>

**Q278. *(Mid)* What is ReDoS?**
<details><summary>Answer</summary>A regex with nested quantifiers over overlapping input backtracks exponentially, so a crafted 40-character string can hang the thread for minutes. Fix with disjoint character classes so each character has one way to match, anchor and be specific rather than using `.*`, and bound input length before matching.

**Follow-up:** How do you safely run user-supplied patterns?</details>

**Q279. *(Mid)* What supply-chain protections do you use?**
<details><summary>Answer</summary>Committed lockfiles with `npm ci`, reviewed lockfile diffs in PRs, `npm audit` and Dependabot, Subresource Integrity for CDN scripts, and `ignore-scripts` where the toolchain allows — postinstall hooks are the standard delivery mechanism for a malicious package. Beyond tooling: fewer dependencies, and scepticism about any package maintained by one account.

**Follow-up:** What does SRI actually protect against?</details>

**Q280. *(Senior)* How would you threat-model a new feature?**
<details><summary>Answer</summary>Enumerate what the feature adds — new inputs, new sinks, new privileges, new data at rest — and for each ask what an attacker gains and what stops them. That's a conversation that fits in a code review, unlike a full audit. For a comment feature: untrusted text reaching a sink (sanitise plus CSP), user-supplied links (protocol allowlist), and stored content served to others (so a single miss is stored XSS affecting every reader). I'd also ask what the blast radius is if it goes wrong, because that decides how much defence in depth is warranted.

**Follow-up:** What if the feature is internal-only?</details>

**Q281. *(Senior)* Marketing wants a tag manager on every page. How do you handle it?**
<details><summary>Answer</summary>Name the actual decision: a tag manager exists to inject arbitrary script, so adding one grants whoever holds that account the same privileges as your own code. That should be an explicit, documented choice rather than a CSP workaround. If it proceeds, use a nonce with `'strict-dynamic'` so the loader can bring in children without opening `unsafe-inline` to injected payloads, restrict `connect-src`, and keep `object-src` and `base-uri` locked. Push for server-side tagging if data collection is the real requirement, and never load it on pages handling authentication or payment.

**Follow-up:** How do you audit what a tag manager is actually loading?</details>

**Q282. *(Senior)* A review finds `innerHTML = markdownToHtml(userInput)` and the author says the converter escapes HTML. Response?**
<details><summary>Answer</summary>The claim covers one vector and there are two. Converters typically escape HTML in text but many allow raw HTML passthrough by default, and nearly all support link syntax — so `[click](javascript:alert(1))` is the likelier attack than a raw tag. I'd verify the options (`html: false`, plus a link protocol allowlist), then sanitise the output anyway, because the sanitiser is a boundary we control and the converter's defaults can change on a minor version bump. And add a CSP so a miss in either layer isn't a breach.

**Follow-up:** Which Markdown converters are safe by default?</details>

**Q283. *(Senior)* How do you handle secrets in a frontend build?**
<details><summary>Answer</summary>You don't — anything shipped to a browser is public, whether it's minified, base64'd, or injected at build time from `.env`. The pattern is a server-side proxy holding the credential, with your own authentication in front of it. For genuinely public keys (a Stripe publishable key, a Maps key) the protection is server-side restriction: referrer allowlists, rate limits, scoped permissions. I'd also add a CI check scanning the build output for high-entropy strings, because the mistake usually arrives by accident.

**Follow-up:** What do you do when a secret has already shipped?</details>

**Q284. *(Senior)* How would you harden an app that renders user-generated content?**
<details><summary>Answer</summary>Layer it, assuming each layer will eventually fail. Sanitise with a real parser at one chokepoint that returns a node rather than a string, so callers can't re-concatenate it into a template. Vet every URL against a protocol allowlist, and add `rel="noopener noreferrer"` and `referrerPolicy` on outbound links. Delegate all events so no inline handlers exist and CSP needs no `unsafe-inline`. Then the CSP itself with a nonce, `object-src 'none'`, `base-uri 'none'`, and a restricted `connect-src`. Serve user uploads from a separate origin so a stored HTML file can't run in yours.

**Follow-up:** Why a separate origin for uploads?</details>

**Q285. *(Senior)* Sessions are being hijacked; the app stores a JWT in `localStorage` and has a comment feature. What now?**
<details><summary>Answer</summary>Treat it as an active incident: invalidate every session immediately, since any token that leaked is still valid until you do. The near-certain cause is stored XSS in the comment rendering exfiltrating tokens, so I'd find and fix the sink first, then move the token to an `HttpOnly` cookie with `SameSite=Lax` and a CSRF token so the next XSS isn't also an account takeover. Then a CSP with `connect-src` restricted, which would have limited exfiltration even with the XSS present. Afterwards, a review of every other sink, because a codebase with one usually has several.

**Follow-up:** How do you tell how long it was exploited?</details>

---

## 11. Testing

**Q286. *(Junior)* What's the difference between unit, integration, and E2E tests?**
<details><summary>Answer</summary>Unit tests one module in isolation (fast, precise failures, low confidence). Integration tests several together with real collaborators (medium everything). E2E tests the whole app in a real browser (slow, high confidence, vague failures).

**Follow-up:** Which gives the best confidence per unit of cost?</details>

**Q287. *(Junior)* `toBe` vs `toEqual`?**
<details><summary>Answer</summary>`toBe` is `Object.is` — identity, correct for primitives. `toEqual` is deep structural equality. `toStrictEqual` additionally distinguishes `{a: undefined}` from `{}` and checks class identity.

**Follow-up:** Why does `expect({a:1}).toBe({a:1})` fail?</details>

**Q288. *(Junior)* What makes a good test name?**
<details><summary>Answer</summary>It states the behaviour so the failure reads as a sentence: `formatPrice > throws on a negative amount`. `test1` and `should work` tell you nothing at 3am, which is precisely when you'll read them.

**Follow-up:** How many assertions per test?</details>

**Q289. *(Junior)* Why is `beforeEach` usually better than `beforeAll`?**
<details><summary>Answer</summary>It gives each test fresh state, so tests can't affect each other. Shared mutable state is the leading cause of "passes alone, fails in the suite" — and of tests that depend on execution order.

**Follow-up:** When is `beforeAll` right?</details>

**Q290. *(Junior)* What's a spy, and how does it differ from a stub?**
<details><summary>Answer</summary>A spy records calls while keeping the real behaviour. A stub replaces the behaviour with canned return values. A mock is a stub you also assert against, and a fake is a working lightweight implementation.

**Follow-up:** Which is usually the best of the four?</details>

**Q291. *(Mid)* Why does this test pass without verifying anything?**
```js
it('doubles', () => {
  Promise.resolve(2).then(n => expect(n * 2).toBe(999));
});
```
<details><summary>Answer</summary>Nothing awaits the promise, so the test function returns before the callback runs. The assertion's rejection is unhandled and the runner has already moved on. Return or `await` it.

**Follow-up:** How would you catch this class of bug across a whole suite?</details>

**Q292. *(Mid)* What's wrong with using `try`/`catch` to assert a rejection?**
<details><summary>Answer</summary>If the promise resolves, the `catch` never runs and nothing is asserted — the test passes while the code is broken. Use `await expect(p).rejects.toThrow()`, or add `expect.assertions(1)` so a missing assertion fails.

**Follow-up:** What does `expect.unreachable()` add?</details>

**Q293. *(Mid)* Why do fake timers hang tests involving promises?**
<details><summary>Answer</summary>`advanceTimersByTime` runs timer callbacks but doesn't drain the microtasks those callbacks schedule, so an `await` inside a timer never resolves. Use the async variants — `advanceTimersByTimeAsync` or `runAllTimersAsync`.

**Follow-up:** How would you test exponential backoff delays specifically?</details>

**Q294. *(Mid)* What is over-mocking, and how do you recognise it?**
<details><summary>Answer</summary>Mocking your own modules, so the test verifies the mock's fidelity to a belief rather than the code. You recognise it when a behaviour-preserving refactor breaks dozens of tests, or when the suite is green and the app is broken. Mock only boundaries you don't own: network, clock, randomness, filesystem.

**Follow-up:** What do you do when a module is hard to test without mocking its internals?</details>

**Q295. *(Mid)* Why query by role rather than test id?**
<details><summary>Answer</summary>Roles are how users and assistive technology find controls, so the test breaks when the UX breaks rather than when the markup is refactored — and a missing accessible name fails the test, catching accessibility bugs for free. Test ids are invisible to users and verify nothing about whether the thing is usable.

**Follow-up:** When is `getByTestId` legitimate?</details>

**Q296. *(Mid)* Why is `waitForTimeout` in an E2E test a bug?**
<details><summary>Answer</summary>A fixed sleep is either too short on a slow run (flaky) or too long on a fast one (slow), and it never states what you're waiting for. Assert on the resulting state and let the framework retry until it holds.

**Follow-up:** What's the equivalent mistake in a unit test?</details>

**Q297. *(Senior)* A team has 4,000 unit tests, 92% coverage, and ships a broken checkout every other release. Diagnose it.**
<details><summary>Answer</summary>Everything is unit-tested in isolation with mocked collaborators, so nothing verifies the pieces compose — and coverage measures execution, not verification. I'd add a handful of E2E tests over the checkout journey first, since one test there buys more confidence than another hundred unit tests. Then convert the highest-value unit tests to integration tests by replacing internal mocks with real modules and mocking only the network. I'd expect the test count to fall and confidence to rise, which is usually an uncomfortable conversation.

**Follow-up:** How do you get buy-in for deleting tests?</details>

**Q298. *(Senior)* What's your honest view of TDD?**
<details><summary>Answer</summary>Writing the test first genuinely produces better interfaces, because you design from the call site and can't comfortably test a badly-shaped API — and it forces you to define "done" before starting. It's a poor fit for exploratory work where you don't yet know the shape, and strict red-green-refactor on every trivial function is dogma. I use it where requirements are clear and write the test immediately after where they aren't. The one case with no counterargument is a bug fix: the failing test comes first, or you haven't proved you fixed anything.

**Follow-up:** What do you do when you can't write the test first?</details>

**Q299. *(Senior)* How would you handle a flaky test suite?**
<details><summary>Answer</summary>Treat flakiness as a bug with a deadline, not a fact of life — a suite people don't trust gets bypassed, and then it protects nothing. Quantify it first: track per-test failure rates in CI so you're fixing the actual top offenders rather than the memorable ones. The causes are nearly always the same four: order dependence from shared state, fixed sleeps, real network or clock, and unawaited promises. Randomise test order to expose the first, and quarantine anything you can't fix this week into a separate non-blocking job so the main pipeline stays trustworthy.

**Follow-up:** What's the danger of a quarantine?</details>

**Q300. *(Senior)* Someone proposes enforcing 100% coverage in CI. What do you counter with?**
<details><summary>Answer</summary>100% is worth it in a few places — a library, a payments core, anywhere one untested branch is a real liability. Enforced globally it drives tests written to touch lines rather than verify behaviour, and makes every unreachable `catch` a chore. I'd propose a floor around 80% on *changed* files so new code is covered without a six-month backfill, plus a hard rule that every bug fix ships with a regression test. Then use the coverage report as a review aid: an untested branch is a question, not a build failure.

**Follow-up:** What metric would you rather track?</details>

---

## 12. Patterns and Architecture

**Q301. *(Junior)* What's the simplest Singleton in JavaScript?**
<details><summary>Answer</summary>A module export — modules are evaluated once and cached, so it already is one. The class-with-static-instance form only adds lazy initialisation, at the cost of global mutable state that's harder to test.

**Follow-up:** What's the testable alternative?</details>

**Q302. *(Junior)* What is the Observer pattern?**
<details><summary>Answer</summary>A subject maintaining a list of observers it notifies on change. It's the most-used pattern in front-end code — `addEventListener` is one, as is every store subscription. PubSub is the variant where a broker sits between, so publisher and subscriber don't know each other.

**Follow-up:** What should `on()` return?</details>

**Q303. *(Junior)* What is Strategy, and how does JavaScript express it?**
<details><summary>Answer</summary>Swapping an algorithm at runtime. In JavaScript it's a function parameter or an object keyed by name — `sortStrategies[name](a, b)` — rather than a class hierarchy. It earns its keep over an `if` chain when strategies are added often or come from configuration.

**Follow-up:** When is a `switch` still better?</details>

**Q304. *(Junior)* What is dependency injection and why bother?**
<details><summary>Answer</summary>Passing collaborators in rather than importing them, usually as `createService({ db, clock })`. Tests then pass a plain object instead of needing a module-mocking framework, and dependencies are visible in the signature rather than hidden in imports.

**Follow-up:** What's the cost?</details>

**Q305. *(Mid)* Why does decorator order matter?**
<details><summary>Answer</summary>`withCache(withRetry(fn))` caches the retried result, which is what you want. `withRetry(withCache(fn))` retries the cache lookup, which is nearly always a bug. Each layer only sees what the layer inside it exposes, so composition order is semantics, not style.

**Follow-up:** Where would you put a rate limiter in that stack?</details>

**Q306. *(Mid)* When does a state machine beat boolean flags?**
<details><summary>Answer</summary>As soon as three or more flags describe one thing. N booleans allow 2^N combinations and most are impossible — `isLoading && hasError && isEmpty` is a state your code can reach and your UI has no design for. A machine enumerates only legal states and transitions, making the impossible unrepresentable.

**Follow-up:** How would you model a request lifecycle?</details>

**Q307. *(Mid)* Why copy the listener set before dispatching?**
<details><summary>Answer</summary>A handler may unsubscribe during dispatch — including a `once` wrapper removing itself — which mutates the collection you're iterating, skipping listeners or throwing. Copy first. Also isolate each handler's errors, so one buggy listener doesn't cancel the rest.

**Follow-up:** How do you isolate them without swallowing the error?</details>

**Q308. *(Mid)* What's the real benefit of the Repository pattern?**
<details><summary>Answer</summary>Query code lives in one file, so a schema change has one blast radius and you can find every access to a table. Database portability is the advertised benefit and almost never the realised one — nobody swaps Postgres for Mongo behind an interface.

**Follow-up:** When is a repository over-engineering?</details>

**Q309. *(Mid)* Implement a middleware pipeline.**
<details><summary>Answer</summary>

```js
const run = async (middleware, context) => {
  let index = -1;
  const dispatch = async (i) => {
    if (i <= index) throw new Error('next() called twice');   // the classic bug, made loud
    index = i;
    const fn = middleware[i];
    if (fn) await fn(context, () => dispatch(i + 1));
  };
  await dispatch(0);
  return context;
};
```

**Follow-up:** How would you add error-handling middleware?  (A middleware that wraps `await next()` in a try/catch — it must be registered first.)</details>

**Q310. *(Mid)* What is a Facade and when is it worth it?**
<details><summary>Answer</summary>One simple interface over a messy subsystem — `media.play(url)` hiding codec detection, audio context setup, and analytics. It's worth it when multiple callers would otherwise repeat the same six-step dance, and it's over-engineering when there's one caller.

**Follow-up:** How is it different from an Adapter?</details>

**Q311. *(Mid)* How does `Proxy` implement the Proxy pattern differently from the classic form?**
<details><summary>Answer</summary>The classic pattern is a hand-written wrapper class with the same interface. `Proxy` intercepts *fundamental operations* — property get, set, has, delete — so it works for properties you never declared, which is what makes reactivity possible. The cost is that every trapped access is a function call and can't be inline-cached.

**Follow-up:** Why always delegate through `Reflect`?</details>

**Q312. *(Mid)* What's the Command pattern for, beyond undo?**
<details><summary>Answer</summary>Anything needing operations as data: queuing, retrying, logging an audit trail, replaying a session, or sending an action across a worker boundary. Undo is the famous case because it needs the inverse captured alongside the operation, but serialisability is often the more useful property.

**Follow-up:** What must a command capture for undo to work?</details>

**Q313. *(Senior)* How do you decide whether to introduce a pattern?**
<details><summary>Answer</summary>When I have the problem it solves, twice. Once is a coincidence and abstracting on it usually produces the wrong abstraction — I'd rather duplicate and see the second case's shape. The name has real value for communication ("this is a Strategy" is four words) and real cost, because naming a thing makes people want to build it. And in JavaScript I check first whether the pattern collapses to a function, since several of the classic ones do.

**Follow-up:** Which patterns collapse to a function here?</details>

**Q314. *(Senior)* Your app has 40 event names on a global emitter and nobody knows who listens. What do you do?**
<details><summary>Answer</summary>That's the predictable failure of PubSub: total decoupling means no discoverable call graph. I'd move to direct subscriptions where there's a real owner — a component subscribing to a store it holds a reference to — so "find all references" works again. Where a global bus is genuinely right (auth expiry, cross-cutting concerns), define the event names in one module so the vocabulary is at least enumerable, and add a development-mode listener registry you can dump. The underlying fix is usually fewer, coarser events carrying more data.

**Follow-up:** How do you migrate without a big-bang rewrite?</details>

**Q315. *(Senior)* How would you structure a large frontend codebase?**
<details><summary>Answer</summary>By feature, not by type — `features/checkout/` containing its components, hooks, and API calls, rather than `components/`, `hooks/`, `api/` directories that force every change to touch four folders. A `shared/` layer for genuinely cross-feature code, with a lint rule enforcing that features don't import each other directly, because that's the boundary that erodes first. Keep the dependency graph acyclic and shallow. The test I apply: can a new hire delete a feature by deleting one directory? If not, the boundaries aren't real.

**Follow-up:** Where do you put cross-feature state?</details>

**Q316. *(Senior)* When is a state management library warranted?**
<details><summary>Answer</summary>When state is genuinely shared across distant parts of the tree and changes frequently — a collaborative editor, a complex form wizard, cross-cutting notifications. Before that, local state plus context or props handles more than people expect, and a store adds indirection to every read. I'd also separate two problems people conflate: server cache (which a query library solves properly, with staleness and revalidation) and client UI state (which is usually small). Reaching for Redux to cache API responses is the common mistake.

**Follow-up:** What's your default for a new app?</details>

**Q317. *(Senior)* How do you handle a legacy module everyone is afraid to touch?**
<details><summary>Answer</summary>Get it under test before changing anything — characterisation tests that assert current behaviour, including behaviour that looks wrong, so you find out what depends on it. Then introduce a seam: wrap it behind an interface so new code depends on the interface rather than the module, and new functionality goes in a new implementation. Strangle it incrementally, moving callers over one at a time, with the old and new paths coexisting. What I wouldn't do is rewrite it in one branch, because the fear is usually justified — the module encodes requirements nobody remembers.

**Follow-up:** What if it has no tests and no clear behaviour?</details>

**Q318. *(Senior)* Micro-frontends — when do they earn their keep?**
<details><summary>Answer</summary>When independent deployment by separate teams is a hard organisational requirement and the alternative is a release train everyone waits for. That's an org problem with a technical solution, and it's the only case I've seen justify it. The costs are severe: duplicated framework bundles, shared-state coordination across boundaries, inconsistent UX, harder E2E testing, and debugging across ownership lines. For a single team, it's a distributed monolith with extra latency — a monorepo with enforced package boundaries gets most of the benefit.

**Follow-up:** How do you share design system components across them?</details>

**Q319. *(Senior)* How do you make architectural decisions durable in a team?**
<details><summary>Answer</summary>Write them down with the reasoning and the rejected alternatives — an ADR, short, in the repo. The value isn't the decision, it's that in eighteen months someone can see *why*, and either respect the constraint or knowingly overturn it; without that, decisions get re-litigated quarterly or cargo-culted forever. Then encode what's mechanically checkable as lint rules or CI checks, because a convention that relies on reviewers remembering it will erode. And revisit explicitly — an ADR marked superseded is healthy; one that's silently ignored is worse than none.

**Follow-up:** What's in an ADR?</details>

**Q320. *(Senior)* A reviewer says your module needs a Factory, an Abstract Factory, and a Builder. Response?**
<details><summary>Answer</summary>Ask which problem each one solves here. A Factory is justified if construction involves a decision callers shouldn't repeat; a Builder if there are many interdependent optional parts; an Abstract Factory only with genuine families of related products to swap, which is rare outside frameworks. If the answer is "flexibility", the flexibility already exists — a function that returns an object can become any of these later in one commit without changing call sites. Ship the function; add the pattern when a second caller needs it.

**Follow-up:** How do you have that conversation without being dismissive?</details>

---

## 13. Modern Syntax, ES2015–ES2025

**Q321. *(Junior)* Which edition introduced optional chaining and nullish coalescing?**
<details><summary>Answer</summary>ES2020, together — they were designed as a pair and are almost always used together, alongside `BigInt`, `Promise.allSettled`, dynamic `import()`, and `globalThis`.

**Follow-up:** Why is `a ?? b || c` a syntax error?</details>

**Q322. *(Junior)* What's the difference between `??` and `||`?**
<details><summary>Answer</summary>`||` falls back on any falsy value, so it overrides a deliberate `0`, `''`, or `false`. `??` triggers only on `null` and `undefined`. Same distinction for `??=` versus `||=`.

**Follow-up:** Give a bug caused by using `||`.</details>

**Q323. *(Junior)* What did `#private` fields change?**
<details><summary>Answer</summary>They gave classes real, language-enforced privacy (ES2022) — invisible to `Object.keys`, `JSON.stringify`, and any caller. Before them, privacy meant a closure or an underscore convention that anyone could ignore.

**Follow-up:** How do you brand-check for one?  (`#x in obj`.)</details>

**Q324. *(Junior)* Which array methods did ES2023 add and why?**
<details><summary>Answer</summary>`toSorted`, `toReversed`, `toSpliced`, and `with` — non-mutating equivalents of `sort`, `reverse`, `splice`, and index assignment. They exist because in-place mutation of shared arrays is one of the most common bug sources in JavaScript. Also `findLast` and `findLastIndex`.

**Follow-up:** Do they work on typed arrays?  (Yes.)</details>

**Q325. *(Junior)* What is `Object.hasOwn` for?**
<details><summary>Answer</summary>A safe replacement for `obj.hasOwnProperty(k)` (ES2022) — it works on null-prototype objects and can't be shadowed by a property called `hasOwnProperty`.

**Follow-up:** When would `hasOwnProperty` actually be shadowed?</details>

**Q326. *(Mid)* What problem does `Error.cause` solve?**
<details><summary>Answer</summary>Rethrowing with context used to discard the original error and its stack, so the log showed "Failed to save" and nothing about why. `new Error('Failed to save', { cause: err })` preserves the chain, and tooling now walks it.

**Follow-up:** How do you serialise a cause chain for a log?</details>

**Q327. *(Mid)* What does `Promise.withResolvers` replace?**
<details><summary>Answer</summary>The "deferred" pattern — assigning `resolve` and `reject` out of the executor into outer variables, which everyone hand-rolled and which was untypeable without a cast. `const { promise, resolve, reject } = Promise.withResolvers()` (ES2024).

**Follow-up:** Where do you need it?  (Bridging an event-based API into a promise.)</details>

**Q328. *(Mid)* What are iterator helpers and why are they different from array methods?**
<details><summary>Answer</summary>`map`, `filter`, `take`, `drop`, `flatMap`, `reduce`, and `toArray` on any iterator (ES2025). They're lazy, so they never materialise an intermediate array and can operate on infinite sequences — `naturals().map(f).take(5).toArray()` terminates where the array version can't exist.

**Follow-up:** When does laziness actually matter?</details>

**Q329. *(Mid)* What did ES2025 add to `Set`?**
<details><summary>Answer</summary>`union`, `intersection`, `difference`, `symmetricDifference`, `isSubsetOf`, `isSupersetOf`, and `isDisjointFrom` — all returning new sets, never mutating. It closed a gap that had every codebase carrying its own `intersect` helper.

**Follow-up:** What's the complexity of `intersection`?</details>

**Q330. *(Mid)* Explain `Object.groupBy` versus `Map.groupBy`.**
<details><summary>Answer</summary>Both group an iterable by a key function (ES2024). `Object.groupBy` returns a null-prototype object, so string keys only but no prototype-collision surface. `Map.groupBy` returns a `Map`, so keys can be objects and insertion order is guaranteed.

**Follow-up:** Why is the null prototype significant?</details>

**Q331. *(Mid)* What is top-level `await` and what does it cost?**
<details><summary>Answer</summary>`await` at module scope (ES2022), which makes a module's evaluation asynchronous. The cost is that every importer waits for it, so a top-level `await` in a widely-imported module serialises startup. Fine for config loading at an entry point; a problem in a leaf utility.

**Follow-up:** Why can't CommonJS support it?</details>

**Q332. *(Mid)* What are import attributes for?**
<details><summary>Answer</summary>Declaring a module's type: `import data from './d.json' with { type: 'json' }` (ES2025). It's a security requirement rather than a convenience — without it, a server could serve JavaScript where JSON was expected and the browser would execute it.

**Follow-up:** What replaced the earlier `assert` keyword?</details>

**Q333. *(Mid)* What does `RegExp.escape` fix?**
<details><summary>Answer</summary>Building a pattern from user input safely (ES2025). Without it, a typed `(` or `*` either throws `SyntaxError` or changes the grammar into something that backtracks exponentially. Previously everyone carried a `replace(/[.*+?^${}()|[\]\\]/g, '\\$&')` helper.

**Follow-up:** Why is the manual version risky?</details>

**Q334. *(Mid)* What does `Temporal` fix about `Date`?**
<details><summary>Answer</summary>Immutability (every operation returns a new value), one-indexed months, first-class timezone and calendar support instead of implicit local time, and a parser that's consistent across engines. `Date` has been broken since 1995 and can't be fixed compatibly.

**Follow-up:** Can you use it today?</details>

**Q335. *(Mid)* Which features can't be polyfilled?**
<details><summary>Answer</summary>Anything requiring engine-level semantics: `Proxy`, true `WeakMap`/`WeakRef` weakness, `BigInt` arithmetic, and all syntax — `?.`, `#private`, class fields — because a parse error kills the file before any shim can run. Syntax must be transpiled; APIs can be shimmed.

**Follow-up:** What's a partial `Proxy` polyfill limited to?</details>

**Q336. *(Senior)* How do you decide when to adopt a new language feature?**
<details><summary>Answer</summary>Support in the browsers my analytics actually show, not a generic threshold — and whether the fallback is transpilation (cheap, automatic) or a polyfill (bundle cost, sometimes impossible). Then whether it removes more complexity than it adds: `?.` and `??` were immediate adoptions because they deleted whole idioms, where decorators I'd wait on because the ecosystem hasn't settled. And I'd rather adopt uniformly than have three ways to do the same thing in one codebase, so a feature arrives with a lint rule and a codemod, not a suggestion.

**Follow-up:** What's still not worth adopting?</details>

**Q337. *(Senior)* Your `browserslist` hasn't been reviewed in four years. What do you do?**
<details><summary>Answer</summary>Look at real analytics rather than a generic `> 0.5%` query, which bears no relation to any specific product's users. Set the target from that, run `npx browserslist` to see what it resolves to, and diff the bundle before and after — dropping ES5 output typically removes a large share of the `core-js` payload and makes stack traces readable again. Then add a bundle-size check to CI so the win doesn't erode, and a calendar reminder to revisit annually, because this decays silently.

**Follow-up:** What if analytics show 2% on a browser that needs ES5?</details>

**Q338. *(Senior)* Would you adopt `Temporal` now?**
<details><summary>Answer</summary>The API, yes — behind the official polyfill, because correct date arithmetic today plus a free upgrade later is a good trade, and `Date`'s bugs are real bugs. But the polyfill is tens of kilobytes, so I'd measure the impact and load it only on routes doing date maths if that matters. Not native `Temporal`: it shipped in Firefox first and remains flagged or absent elsewhere. Critically, I'd keep the conversion at the boundary — parse on input, format on output — so the eventual switch to native touches no business logic.

**Follow-up:** What's the migration risk?</details>

**Q339. *(Senior)* How do you handle the gap between "in the spec" and "in my users' browsers"?**
<details><summary>Answer</summary>Assume they're unrelated. A feature reaching stage 4 says nothing about the browser on someone's three-year-old Android, and stage 3 says even less — it means implementations are invited. So: `browserslist` from real analytics as the single source of truth, feature-detect rather than version-detect, and treat progressive enhancement as the default posture where the feature is an improvement rather than a requirement. And check MDN's compatibility data rather than a blog post, since it's generated from the same data caniuse uses and it's usually more current than anyone's memory.

**Follow-up:** How do you feature-detect syntax rather than an API?</details>

**Q340. *(Senior)* Which recent additions have actually changed how you write JavaScript?**
<details><summary>Answer</summary>`?.` and `??` most of all — they deleted entire defensive idioms and made a class of null-check bug disappear. `#private` fields, because privacy stopped being a convention people ignored. The `to*` array methods, because accidental mutation of shared arrays was a bug I shipped more than once. `AbortController` and `AbortSignal.any`, which turned cleanup from four different mechanisms into one. And `Error.cause`, which is small but changed how I write error handling at layer boundaries — I now wrap rather than swallow, because the original stack survives.

**Follow-up:** Which ones turned out not to matter?</details>

---

**Next:** [Appendix B — 100+ Coding Challenges →](B-coding-challenges.md)
