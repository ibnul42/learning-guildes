# Chapter 4 — Functions

## Learning Objectives

- Distinguish declarations, expressions, arrows, and methods — and their hoisting/`this` behaviour
- Use default, rest, and destructured parameters
- Understand first-class functions, HOFs, and callbacks
- Know when an arrow function is wrong
- Apply currying, partial application, and recursion

---

## 4.1 The Four Ways to Make a Function

```js
// 1. Declaration — hoisted completely, has its own `this`, has `arguments`
function add(a, b) { return a + b; }

// 2. Expression — not hoisted (the variable is, in the TDZ)
const sub = function (a, b) { return a - b; };

// 3. Named function expression — the name is visible only inside itself (useful for recursion/stacks)
const fact = function f(n) { return n <= 1 ? 1 : n * f(n - 1); };

// 4. Arrow — no own `this`, `arguments`, `super`, or `new.target`; cannot be `new`ed
const mul = (a, b) => a * b;
```

Arrow forms:

```js
x => x * 2                 // single param, implicit return
(x, y) => x + y
() => 42
x => ({ value: x })        // parens required to return an object literal!
x => { return x * 2; }     // block body needs explicit return
```

### Declaration vs expression hoisting

```js
greet();      // ✅ 'hi' — declarations are fully hoisted
sayBye();     // ❌ ReferenceError — TDZ

function greet() { console.log('hi'); }
const sayBye = () => console.log('bye');
```

---

## 4.2 Parameters

```js
// Defaults — evaluated at call time, left to right, only for `undefined`
function fetchData(url, { retries = 3, timeout = retries * 1000 } = {}) { }
fetchData('/api');                    // retries 3, timeout 3000
fetchData('/api', { retries: 1 });    // timeout 1000

// null does NOT trigger a default
function f(x = 5) { return x; }
f(undefined); // 5
f(null);      // null

// Rest — must be last, is a real Array
function sum(first, ...rest) { return rest.reduce((a, b) => a + b, first); }
sum(1, 2, 3); // 6

// arguments — array-LIKE, not in arrow functions, avoid in new code
function old() { return Array.from(arguments); }
```

`length` counts parameters *before* the first default or rest:

```js
((a, b) => {}).length;        // 2
((a, b = 1) => {}).length;    // 1
((...args) => {}).length;     // 0
```

> ⚠️ **Gotcha** — A default value that's an object is created fresh on every call, unlike Python. `function f(list = []) {}` is safe here.

---

## 4.3 Functions Are Values

```js
const ops = { add, sub };                  // stored in objects
[add, sub].forEach(fn => console.log(fn(4, 2)));  // stored in arrays
function apply(fn, x) { return fn(x); }    // passed as arguments
function counter() { let n = 0; return () => ++n; }  // returned
```

A **higher-order function** takes or returns a function. `map`, `filter`, `setTimeout`, and `addEventListener` are all HOFs.

### IIFE

```js
(function () { /* private scope */ })();
(() => { /* same */ })();
(async () => { await main(); })();   // still useful pre-top-level-await
```

Modules made IIFEs largely obsolete for scoping, but the async IIFE remains common.

---

## 4.4 Arrow Functions: When Not To

Arrows have **no own `this`** — they close over the enclosing scope's `this` lexically. That's a feature and a trap.

```js
// ✅ Arrow fixes the callback `this` problem
class Timer {
  constructor() { this.n = 0; }
  start() { setInterval(() => this.n++, 1000); }   // `this` is the Timer
}

// ❌ Arrow as an object method — `this` is NOT the object
const counter = {
  n: 0,
  inc: () => { this.n++; },      // `this` is module scope / undefined
  dec() { this.n--; },           // ✅ shorthand method
};

// ❌ Arrow as a constructor
const P = () => {};
new P();   // TypeError: P is not a constructor

// ❌ Arrow as a prototype method needing `this`
Array.prototype.last = () => this[this.length - 1];  // broken
```

**Rule:** arrow for callbacks and short pure functions; `function`/method shorthand when `this` must be dynamic.

---

## 4.5 Currying, Partial Application, Recursion

```js
// Currying — one argument at a time
const add = a => b => c => a + b + c;
add(1)(2)(3);   // 6

// Partial application — fix some args now
const log = (level, msg) => console.log(`[${level}] ${msg}`);
const error = log.bind(null, 'ERROR');
error('disk full');   // [ERROR] disk full

// Recursion — always define the base case first
function flatten(arr) {
  return arr.reduce(
    (out, v) => out.concat(Array.isArray(v) ? flatten(v) : v),
    []
  );
}
```

JavaScript engines do **not** implement tail-call optimization (Safari aside). Deep recursion overflows the stack around 10,000 frames. Convert to iteration with an explicit stack for unbounded depth.

---

## 4.6 Common Mistakes and Best Practices

**Mistake: forgetting parens around a returned object literal.** `x => { a: x }` returns `undefined` — `{ }` is parsed as a block and `a:` as a label.

**Mistake: mutating arguments.** Callers don't expect it. Copy first.

**Mistake: too many parameters.** Past three, use an options object — it self-documents at the call site and order stops mattering.

**Best practice: pure by default.** Same input → same output, no side effects. Trivially testable (Chapter 29).

**Best practice: one job per function.** If the name needs "and," split it.

**Best practice: guard clauses over nesting.** Validate and return early.

---

## 4.7 Real-World Example: A Composable Middleware Pipeline

```js
/**
 * Express-style middleware, built from nothing but closures and HOFs.
 */
function createPipeline() {
  const middlewares = [];

  return {
    use(fn) { middlewares.push(fn); return this; },   // chainable

    async run(ctx) {
      let index = -1;
      const dispatch = async (i) => {
        if (i <= index) throw new Error('next() called twice');
        index = i;
        const fn = middlewares[i];
        if (!fn) return;
        await fn(ctx, () => dispatch(i + 1));
      };
      await dispatch(0);
      return ctx;
    },
  };
}

const pipeline = createPipeline()
  .use(async (ctx, next) => {
    const start = performance.now();
    await next();                                  // run everything downstream
    ctx.duration = performance.now() - start;      // then resume here
  })
  .use(async (ctx, next) => {
    if (!ctx.user) throw new Error('Unauthorized');
    await next();
  })
  .use(async (ctx) => { ctx.body = `Hello ${ctx.user}`; });

await pipeline.run({ user: 'Ada' });
```

Every piece is a function: stored in an array, passed `next` as a callback, closing over `index`. This is the whole idea of first-class functions doing real work.

---

## 4.8 Questions

**1.** MCQ — Which is hoisted and callable before its definition? a) arrow b) function expression c) function declaration d) class method
<details><summary>Answer</summary>**c)**</details>

**2.** MCQ — Arrow functions lack: a) `this` b) `arguments` c) `prototype` d) all of these
<details><summary>Answer</summary>**d)**</details>

**3.** MCQ — `((a, b = 2, ...c) => {}).length` is: a) 3 b) 2 c) 1 d) 0
<details><summary>Answer</summary>**c)** Counts params before the first default.</details>

**4.** MCQ — `f(null)` where `function f(x = 5)` returns: a) 5 b) null c) undefined d) error
<details><summary>Answer</summary>**b)** Only `undefined` triggers defaults.</details>

**5.** MCQ — `arguments` is: a) an Array b) array-like c) a Set d) a string
<details><summary>Answer</summary>**b)**</details>

**6.** MCQ — Which cannot be used with `new`? a) declaration b) expression c) arrow d) class
<details><summary>Answer</summary>**c)**</details>

**7.** Short — Why must `x => ({ a: 1 })` have parentheses?
<details><summary>Answer</summary>Without them, `{` starts a block body, not an object literal.</details>

**8.** Short — What is a higher-order function?
<details><summary>Answer</summary>One that takes a function as an argument, returns a function, or both.</details>

**9.** Short — Currying vs partial application?
<details><summary>Answer</summary>Currying transforms an n-ary function into n unary functions. Partial application fixes *some* arguments and returns a function taking the rest.</details>

**10.** Short — Why is a named function expression's name useful?
<details><summary>Answer</summary>It's bound inside the function body (allowing self-reference even if the outer variable is reassigned) and it appears in stack traces.</details>

**11.** Short — When should you use an options object instead of positional params?
<details><summary>Answer</summary>Beyond ~3 parameters, or whenever several are optional/boolean — call sites become self-documenting and order-independent.</details>

**12.** Output — What prints?
```js
const o = { n: 1, get: () => this.n, get2() { return this.n; } };
console.log(o.get(), o.get2());
```
<details><summary>Answer</summary>`undefined 1` — the arrow has no own `this`.</details>

**13.** Output — What prints?
```js
function f() { return arguments.length; }
const g = (...a) => a.length;
console.log(f(1,2,3), g(1,2,3));
```
<details><summary>Answer</summary>`3 3`</details>

**14.** Output — What prints?
```js
console.log(typeof foo, typeof bar);
function foo() {}
var bar = function () {};
```
<details><summary>Answer</summary>`'function' 'undefined'` — `var bar` is hoisted as `undefined`.</details>

**15.** Output — What prints?
```js
const add = a => b => a + b;
console.log(add(2)(3), typeof add(2));
```
<details><summary>Answer</summary>`5 'function'`</details>

**16.** Output — What prints?
```js
function f(a, b = a * 2) { return [a, b]; }
console.log(f(3), f(3, undefined), f(3, null));
```
<details><summary>Answer</summary>`[3,6] [3,6] [3,null]`</details>

**17.** Debug — `this.total` is `undefined`. Why?
```js
const cart = { items: [1,2,3], total: 0,
  sum() { this.items.forEach(function (i) { this.total += i; }); } };
```
<details><summary>Answer</summary>The plain callback has its own `this` (undefined in strict mode). Use an arrow callback, or pass `this` as `forEach`'s second argument.</details>

**18.** Debug — This returns `undefined`:
```js
const makeUser = (name) => { name, id: Date.now() };
```
<details><summary>Answer</summary>Block body parsed instead of object literal. Wrap in parens: `({ name, id: Date.now() })`.</details>

**19.** Debug — Stack overflow on a 50,000-element nested array with a recursive `flatten`. Fix?
<details><summary>Answer</summary>No TCO in JS. Rewrite iteratively with an explicit stack, or use `arr.flat(Infinity)`.</details>

**20.** Code — Implement `once(fn)` so `fn` runs at most once, returning the cached result.
<details><summary>Answer</summary>

```js
function once(fn) {
  let called = false, result;
  return function (...args) {
    if (!called) { called = true; result = fn.apply(this, args); fn = null; }
    return result;
  };
}
```
</details>

**21.** Code — Implement `curry(fn)` for any arity.
<details><summary>Answer</summary>

```js
function curry(fn) {
  return function curried(...args) {
    return args.length >= fn.length
      ? fn.apply(this, args)
      : (...more) => curried.apply(this, [...args, ...more]);
  };
}
```
</details>

**22.** Code — Implement `memoize(fn)` with a custom key resolver.
<details><summary>Answer</summary>

```js
function memoize(fn, keyFn = (...a) => JSON.stringify(a)) {
  const cache = new Map();
  return function (...args) {
    const key = keyFn(...args);
    if (!cache.has(key)) cache.set(key, fn.apply(this, args));
    return cache.get(key);
  };
}
```
Note: unbounded cache is a leak risk — see Chapter 27.
</details>

**23.** Code — Implement `pipe(...fns)` and `compose(...fns)`.
<details><summary>Answer</summary>

```js
const pipe    = (...fns) => x => fns.reduce((v, f) => f(v), x);
const compose = (...fns) => x => fns.reduceRight((v, f) => f(v), x);
```
</details>

**24.** Scenario — A React-style component's event handler loses `this` when passed to `addEventListener`. Two fixes?
<details><summary>Answer</summary>Define the handler as a class field arrow (`handle = () => {}`) so it binds lexically, or bind it in the constructor (`this.handle = this.handle.bind(this)`). The arrow field is preferred in modern code.</details>

**25.** Scenario — A colleague memoizes a function taking large objects, keyed by `JSON.stringify`. What are the two problems?
<details><summary>Answer</summary>Stringifying large objects on every call can cost more than the function saves, and key order/undefined values make it unreliable. Prefer a `WeakMap` keyed by object identity (Chapter 15), or a cheap explicit key like an id.</details>

---

## Chapter Summary

- Declarations hoist fully; expressions and arrows sit in the TDZ.
- Arrows have no own `this`, `arguments`, `prototype`, or `new` — great for callbacks, wrong for methods and constructors.
- Defaults trigger only on `undefined`; rest params are real arrays; `fn.length` ignores defaults and rest.
- Functions are values: store, pass, return them. HOFs are the backbone of idiomatic JS.
- Prefer pure, single-purpose functions with guard clauses and options objects.
- No TCO — convert deep recursion to iteration.

---

## Practice Exercises

1. Write `debounce(fn, ms)` and `throttle(fn, ms)` from scratch (see also Chapter 24).
2. Implement `partial(fn, ...preset)` supporting a placeholder symbol for skipped positions.
3. Convert a 5-parameter function in your code to an options object with defaults.
4. Write a recursive `deepClone`, then an iterative version. Benchmark on a deeply nested object.
5. Implement `unary(fn)` and explain why `['1','2','3'].map(parseInt)` fails but `.map(unary(parseInt))` works.

---

## Mini Project: A Function Utility Library

Build `fp.js` exporting: `pipe`, `compose`, `curry`, `partial`, `memoize`, `once`, `debounce`, `throttle`, `tap`, `identity`, `constant`, and `negate`.

**Requirements:**

- Every function preserves `this` where meaningful (use `function`, not arrows, for the returned wrappers).
- `memoize` accepts `{ maxSize }` and evicts least-recently-used entries.
- `debounce` supports `{ leading, trailing }` options and exposes `.cancel()` and `.flush()`.
- Write at least three assertions per utility using `node:assert`.
- Document each with a JSDoc block including `@example`.

**Stretch:** add `curry` support for placeholders, and a `pipeAsync` that awaits each step.

---

**Next:** [Chapter 5 — Objects →](05-objects.md)
