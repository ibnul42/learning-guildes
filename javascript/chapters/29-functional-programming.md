# Chapter 29 — Functional Programming

## Learning Objectives

- Distinguish pure functions from side-effecting ones and explain why it matters
- Compose behaviour with `pipe`, `compose`, currying, and partial application
- Implement immutable updates without a library, including the shallow-freeze trap
- Model failure with `Result` instead of exceptions, and know when not to

---

## 29.1 Purity, Side Effects, and Referential Transparency

A **pure function** returns the same output for the same input and changes nothing outside
itself. That's the whole definition, and everything else in this chapter follows from it.

```js
// ✅ Pure: same input → same output, no observable effect
const add = (a, b) => a + b;
const withTax = (price, rate) => price * (1 + rate);

// ❌ Impure: reads external mutable state
let taxRate = 0.2;
const withTax2 = (price) => price * (1 + taxRate);

// ❌ Impure: mutates its argument
const addItem = (cart, item) => { cart.items.push(item); return cart; };

// ❌ Impure: non-deterministic
const id = () => Math.random().toString(36);
const now = () => Date.now();
```

**Referential transparency** is the payoff: a call to a pure function can be replaced by its
result without changing the program. That's what makes pure code trivially testable (no setup, no
mocks), cacheable (memoisation is always safe), parallelisable, and reorderable.

Side effects aren't wrong — a program with no effects does nothing. The goal is to **push them to
the edges**: a pure core that computes, and a thin impure shell that does I/O.

```js
// ❌ Effects tangled through the logic
async function processOrder(id) {
  const order = await db.get(id);
  order.total = order.items.reduce((s, i) => s + i.price, 0);
  if (order.total > 100) order.discount = order.total * 0.1;
  await db.save(order);
  await email.send(order.customer, `Total: ${order.total}`);
}

// ✅ Pure core, impure shell
const calculateTotal = (items) => items.reduce((s, i) => s + i.price, 0);
const applyDiscount = (total) => (total > 100 ? { total, discount: total * 0.1 } : { total, discount: 0 });
const priceOrder = (order) => ({ ...order, ...applyDiscount(calculateTotal(order.items)) });

async function processOrder(id) {                 // effects only, no logic
  const priced = priceOrder(await db.get(id));
  await db.save(priced);
  await email.send(priced.customer, `Total: ${priced.total}`);
}
```

`priceOrder` now needs no database and no mock to test. That, not elegance, is the argument.

> 💼 **Interview** — "What makes a function pure?" Both halves matter: deterministic output *and*
> no side effects. Candidates routinely give only the first.

---

## 29.2 Immutability in Practice

```js
// Objects
const updated = { ...user, name: 'Ada' };
const { password, ...safe } = user;                 // omit a key
const nested = { ...state, user: { ...state.user, name: 'Ada' } };   // manual, per level

// Arrays — non-mutating equivalents
const added   = [...items, item];
const removed = items.filter(i => i.id !== id);
const mapped  = items.map(i => (i.id === id ? { ...i, done: true } : i));
const inserted = items.toSpliced(2, 0, item);       // ES2023
const sorted  = items.toSorted((a, b) => a.n - b.n); // ES2023 — sort() MUTATES
const reversed = items.toReversed();                 // ES2023
const replaced = items.with(0, item);                // ES2023
```

The ES2023 methods exist precisely because `sort`, `reverse`, and `splice` mutating in place is a
persistent source of bugs. Memorise which mutate:

| Mutating | Non-mutating equivalent |
|---|---|
| `sort` | `toSorted` |
| `reverse` | `toReversed` |
| `splice` | `toSpliced` |
| `arr[i] = x` | `with(i, x)` |
| `push`/`pop`/`shift`/`unshift` | spread, `slice`, `concat` |
| `fill`, `copyWithin` | `map` |

**`Object.freeze` is shallow** — the single most common immutability mistake:

```js
const state = Object.freeze({ user: { name: 'Ada' }, tags: ['a'] });
state.user = {};             // silently ignored (throws in strict mode)
state.user.name = 'Grace';   // ✅ works — the nested object was never frozen
state.tags.push('b');        // ✅ works

function deepFreeze(obj) {
  for (const value of Object.values(obj)) {
    if (value && typeof value === 'object' && !Object.isFrozen(value)) deepFreeze(value);
  }
  return Object.freeze(obj);
}
```

`structuredClone` gives you a genuine deep copy, handling cycles, `Map`, `Set`, `Date`, and
`ArrayBuffer` — but not functions or class identity (Chapter 25).

**Structural sharing** is what makes immutability affordable: an update copies only the path from
the root to the change, and every untouched branch is shared by reference.

```js
const next = { ...state, ui: { ...state.ui, modal: 'open' } };
next.entities === state.entities;    // true — the big part was never copied
```

For deeply nested state, hand-writing that is painful, so write one helper rather than a spread
pyramid:

```js
/** setIn(obj, ['a','b','c'], 1) — copies only the nodes along the path. */
function setIn(obj, [key, ...rest], value) {
  const child = rest.length ? setIn(obj?.[key] ?? {}, rest, value) : value;
  return Array.isArray(obj) ? obj.with(Number(key), child) : { ...obj, [key]: child };
}
```

---

## 29.3 Higher-Order Functions and Composition

Functions are values: pass them, return them, store them.

```js
const map = (fn) => (arr) => arr.map(fn);              // returns a function
const withLogging = (fn) => (...args) => {              // takes and returns a function
  console.log(fn.name, args);
  return fn(...args);
};
```

`compose` applies right to left (mathematical order); `pipe` applies left to right (reading
order). Prefer `pipe` — it matches how people read.

```js
const pipe    = (...fns) => (x) => fns.reduce((acc, fn) => fn(acc), x);
const compose = (...fns) => (x) => fns.reduceRight((acc, fn) => fn(acc), x);

const slugify = pipe(
  (s) => s.trim(),
  (s) => s.toLowerCase(),
  (s) => s.normalize('NFKD').replace(/[̀-ͯ]/g, ''),   // strip accents
  (s) => s.replace(/[^a-z0-9]+/g, '-'),
  (s) => s.replace(/^-+|-+$/g, ''),
);
slugify('  Héllo, World!  ');    // 'hello-world'
```

For async steps, the same idea with `await`:

```js
const pipeAsync = (...fns) => (x) => fns.reduce((p, fn) => p.then(fn), Promise.resolve(x));
```

**Currying** turns `f(a, b, c)` into `f(a)(b)(c)`; **partial application** fixes some arguments
now and the rest later.

```js
function curry(fn) {
  return function curried(...args) {
    return args.length >= fn.length ? fn.apply(this, args)
                                    : (...rest) => curried.apply(this, [...args, ...rest]);
  };
}

const replace = curry((pattern, replacement, str) => str.replace(pattern, replacement));
const stripSpaces = replace(/\s+/g, '');       // data argument LAST makes this possible
stripSpaces('a b c');                          // 'abc'

// Partial application without currying
const log = (level, module, msg) => console.log(`[${level}] ${module}: ${msg}`);
const dbError = log.bind(null, 'ERROR', 'db');
dbError('connection lost');
```

Put the data argument last in functions you intend to curry. This is why `pipe(map(f), filter(g))`
reads well with `map = (fn) => (arr) => …` but not with the native method order.

**Point-free style** omits naming the argument. It's elegant until it isn't:

```js
const total = pipe(map(prop('price')), sum);         // ✅ clear
const f = pipe(flip(concat), map, apply(compose));   // ❌ nobody can read this
```

Use point-free where it removes noise; abandon it the moment you need `flip`, `apply`, or a
comment to explain the plumbing.

---

## 29.4 Memoisation, Recursion, and Their Limits

Memoisation is only correct for pure functions — and only worth it when the function is expensive
relative to hashing its arguments.

```js
function memoize(fn, keyFn = (...args) => JSON.stringify(args)) {
  const cache = new Map();
  return (...args) => {
    const key = keyFn(...args);
    if (cache.has(key)) return cache.get(key);
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}

// Object arguments: WeakMap keys so entries die with their key (Chapter 27)
function memoizeOne(fn) {
  const cache = new WeakMap();
  return (arg) => {
    if (cache.has(arg)) return cache.get(arg);
    const result = fn(arg);
    cache.set(arg, result);
    return result;
  };
}
```

> ⚠️ **Gotcha** — an unbounded memo cache is a leak (Chapter 27), and `JSON.stringify` as a key
> function is both slow and wrong: `{a:1,b:2}` and `{b:2,a:1}` produce different keys, and
> `undefined` values vanish. Bound the cache, and write a key function that matches your arguments.

**Recursion has no tail-call optimisation in practice.** It's in the ES2015 spec; only
JavaScriptCore ever shipped it. Deep recursion overflows the stack at roughly 10,000 frames.

```js
const sum = (n) => (n === 0 ? 0 : n + sum(n - 1));
sum(100_000);          // ❌ RangeError: Maximum call stack size exceeded
```

**Trampolining** converts recursion into iteration by returning a thunk instead of recursing:

```js
const trampoline = (fn) => (...args) => {
  let result = fn(...args);
  while (typeof result === 'function') result = result();
  return result;
};

const sumT = trampoline(function loop(n, acc = 0) {
  return n === 0 ? acc : () => loop(n - 1, acc + n);   // return a thunk, don't recurse
});
sumT(1_000_000);       // ✅ 500000500000
```

For a tree walk, an explicit stack is simpler and faster than either:

```js
function* walk(root) {
  const stack = [root];
  while (stack.length) {
    const node = stack.pop();
    yield node;
    if (node.children) stack.push(...node.children);
  }
}
```

---

## 29.5 `Maybe` and `Result` Without the Jargon

A **functor** is anything with a `map` that follows the obvious rules. `Array` is one. So is a box
that might be empty:

```js
class Maybe {
  #value;
  constructor(value) { this.#value = value; }
  static of(value) { return new Maybe(value); }

  get isNothing() { return this.#value === null || this.#value === undefined; }

  map(fn)     { return this.isNothing ? this : Maybe.of(fn(this.#value)); }
  flatMap(fn) { return this.isNothing ? this : fn(this.#value); }   // fn returns a Maybe
  getOrElse(fallback) { return this.isNothing ? fallback : this.#value; }
}

Maybe.of(user).map(u => u.address).map(a => a.city).getOrElse('unknown');
```

That's the entire monad concept: `of` to put a value in, `map` to transform inside, `flatMap` to
transform with a function that itself returns a box (so you don't end up with `Maybe<Maybe<T>>`).
`Promise` is the one every JavaScript developer already uses — `.then` is both `map` and `flatMap`.

Optional chaining covers most of what `Maybe` is for, so the honest version is: in JavaScript,
`?.` won. `Result` is the one that still earns its keep, because it carries *why* something failed:

```js
const Ok  = (value) => ({ ok: true,  value, map: (f) => Ok(f(value)),
                          flatMap: (f) => f(value), unwrapOr: () => value });
const Err = (error) => ({ ok: false, error, map: () => Err(error),
                          flatMap: () => Err(error), unwrapOr: (d) => d });

const parse = (s) => { try { return Ok(JSON.parse(s)); } catch (e) { return Err(e.message); } };

parse('{"n":1}').map(o => o.n * 2).unwrapOr(0);     // 2
parse('nope').map(o => o.n * 2).unwrapOr(0);        // 0 — the map never ran
```

`Result` versus exceptions: exceptions are right for genuinely exceptional conditions and for
errors nobody up the stack can handle. `Result` is right when failure is an expected outcome —
validation, parsing, a lookup that may miss — because it puts the failure in the type signature
where the caller cannot forget it. A mixed codebase is fine; a codebase where validation throws is
one where someone forgets a `try`.

**Transducers**, briefly: chaining `filter().map()` creates an intermediate array per step. A
transducer composes the *transformations* instead, so one pass does all of them.

```js
const mapping   = (f) => (step) => (acc, x) => step(acc, f(x));
const filtering = (p) => (step) => (acc, x) => (p(x) ? step(acc, x) : acc);

const xf = pipe(filtering(n => n % 2 === 0), mapping(n => n * 10));
[1,2,3,4].reduce(xf((acc, x) => (acc.push(x), acc)), []);   // [20, 40] — one pass
```

Worth it for large collections or streams. For a 50-item array it's a needless abstraction.

**When FP is the wrong tool:** tight numeric loops where allocation matters, code whose essence is
sequential mutation (a parser's cursor, a canvas), and anywhere point-free style would obscure
intent. A `for` loop that reads clearly beats a clever `reduce`.

---

## 29.6 Common Mistakes and Best Practices

**Mistake: calling a function pure when it mutates its argument.** `(cart) => { cart.items.push(x);
return cart; }` returns a value and is still impure.

**Mistake: trusting `Object.freeze` to be deep.** One level only. Nested objects stay mutable.

**Mistake: forgetting `sort` and `reverse` mutate.** `items.sort()` reorders the original — a
classic bug when `items` is props or shared state. Use `toSorted`.

**Mistake: `JSON.parse(JSON.stringify(x))` as a deep clone.** Loses `Date`, `Map`, `Set`,
`undefined`, functions, and `NaN`; throws on cycles. Use `structuredClone`.

**Mistake: unbounded memoisation.** A cache with no eviction is a leak.

**Mistake: putting the data argument first.** It makes the function impossible to curry usefully.

**Mistake: point-free style at any cost.** If you need `flip` or a comment, name the argument.

**Mistake: `reduce` where a loop is clearer.** A `reduce` that mutates its accumulator and returns
it is a `for` loop wearing a costume.

**Best practice: pure core, impure shell.** Keep I/O at the boundaries; the logic in between needs
no mocks to test.

**Best practice: prefer the non-mutating array methods by default.** Reach for the mutating one
deliberately, when you own the array and it's hot.

**Best practice: compose small named functions.** Five one-line functions with names beat one
fifteen-line function, and each is independently testable.

---

## 29.7 Real-World Example: A Composable Validation Library

Validation is the case where functional composition genuinely wins: rules are pure predicates,
they compose, and collecting *all* errors rather than throwing on the first is what users need.

```js
// ── Result ─────────────────────────────────────────────────────────────────────────
const Ok  = (value) => ({ ok: true, value });
const Err = (errors) => ({ ok: false, errors });

/** A Validator is (value, path) => Result. Composition happens at this type. */

// ── Primitive validators ───────────────────────────────────────────────────────────
const required = (msg = 'is required') => (v, path) =>
  v === undefined || v === null || v === '' ? Err([{ path, msg }]) : Ok(v);

const string = (msg = 'must be a string') => (v, path) =>
  typeof v === 'string' ? Ok(v) : Err([{ path, msg }]);

const number = (msg = 'must be a number') => (v, path) =>
  typeof v === 'number' && Number.isFinite(v) ? Ok(v) : Err([{ path, msg }]);

const minLength = (n, msg = `must be at least ${n} characters`) => (v, path) =>
  v.length >= n ? Ok(v) : Err([{ path, msg }]);

const pattern = (re, msg = 'has an invalid format') => (v, path) =>
  re.test(v) ? Ok(v) : Err([{ path, msg }]);

const range = (min, max, msg = `must be between ${min} and ${max}`) => (v, path) =>
  v >= min && v <= max ? Ok(v) : Err([{ path, msg }]);

// ── Combinators ────────────────────────────────────────────────────────────────────

/** Sequential: stop at the first failure. `string()` must pass before `minLength()` runs. */
const all = (...validators) => (v, path) => {
  let current = v;
  for (const validate of validators) {
    const result = validate(current, path);
    if (!result.ok) return result;
    current = result.value;                 // validators may normalise as well as check
  }
  return Ok(current);
};

/** Parallel: run everything, collect every error. This is what a form needs. */
const every = (...validators) => (v, path) => {
  const errors = validators.flatMap(fn => { const r = fn(v, path); return r.ok ? [] : r.errors; });
  return errors.length ? Err(errors) : Ok(v);
};

/** First success wins; if all fail, report every reason. */
const either = (...validators) => (v, path) => {
  const errors = [];
  for (const validate of validators) {
    const result = validate(v, path);
    if (result.ok) return result;
    errors.push(...result.errors);
  }
  return Err(errors);
};

const optional = (validator) => (v, path) =>
  v === undefined || v === null || v === '' ? Ok(v) : validator(v, path);

/** Transform on success — where normalisation belongs. */
const then = (validator, fn) => (v, path) => {
  const result = validator(v, path);
  return result.ok ? Ok(fn(result.value)) : result;
};

// ── Structural ─────────────────────────────────────────────────────────────────────
const shape = (schema) => (obj, path = '') => {
  if (obj === null || typeof obj !== 'object') return Err([{ path, msg: 'must be an object' }]);
  const errors = [];
  const value = {};
  for (const [key, validate] of Object.entries(schema)) {
    const childPath = path ? `${path}.${key}` : key;
    const result = validate(obj[key], childPath);
    if (result.ok) value[key] = result.value;
    else errors.push(...result.errors);      // collect, don't bail: users want every error at once
  }
  return errors.length ? Err(errors) : Ok(value);
};

const arrayOf = (validator) => (arr, path = '') => {
  if (!Array.isArray(arr)) return Err([{ path, msg: 'must be an array' }]);
  const errors = [];
  const value = [];
  arr.forEach((item, i) => {
    const result = validator(item, `${path}[${i}]`);
    if (result.ok) value.push(result.value);
    else errors.push(...result.errors);
  });
  return errors.length ? Err(errors) : Ok(value);
};

// ── Usage ──────────────────────────────────────────────────────────────────────────
const validateUser = shape({
  name:  all(required(), string(), then(string(), s => s.trim()), minLength(2)),
  email: all(required(), string(), pattern(/^[^@\s]+@[^@\s]+\.[^@\s]+$/, 'must be an email')),
  age:   optional(all(number(), range(13, 120))),
  tags:  optional(arrayOf(all(string(), minLength(1)))),
});

validateUser({ name: 'A', email: 'nope', age: 200, tags: ['ok', ''] });
// {
//   ok: false,
//   errors: [
//     { path: 'name',    msg: 'must be at least 2 characters' },
//     { path: 'email',   msg: 'must be an email' },
//     { path: 'age',     msg: 'must be between 13 and 120' },
//     { path: 'tags[1]', msg: 'must be at least 1 characters' },
//   ]
// }
```

The design decision worth defending is having both `all` (fail fast) and `every` (collect all)
rather than one combinator. They're needed for different reasons: `all` is sequential because
`minLength` would crash on a non-string, so type checks must gate the checks that follow; `every`
is parallel because a user filling in a form wants all four problems at once, not one per
submission. Collapsing them into a single "validate everything" combinator produces either
`TypeError`s from validators receiving the wrong type, or a form that reveals its complaints one
at a time.

---

## 29.8 Questions

**1.** MCQ — A pure function must: a) return a value b) be deterministic and effect-free c) take one argument d) be an arrow function
<details><summary>Answer</summary>**b)** Both halves are required.</details>

**2.** MCQ — `Object.freeze` is: a) deep b) shallow c) recursive d) a no-op on arrays
<details><summary>Answer</summary>**b)** One level only.</details>

**3.** MCQ — Which mutates? a) `toSorted` b) `slice` c) `sort` d) `concat`
<details><summary>Answer</summary>**c)**</details>

**4.** MCQ — `pipe(f, g)(x)` equals: a) `f(g(x))` b) `g(f(x))` c) `f(x) + g(x)` d) `f(g)`
<details><summary>Answer</summary>**b)** `pipe` is left to right; `compose` would give `f(g(x))`.</details>

**5.** MCQ — Currying `f(a, b, c)` produces: a) `f(a, b)(c)` b) `f(a)(b)(c)` c) `f([a,b,c])` d) `f()`
<details><summary>Answer</summary>**b)**</details>

**6.** MCQ — Tail-call optimisation in JavaScript is: a) universal b) in the spec but unimplemented in most engines c) automatic in strict mode d) enabled by a flag
<details><summary>Answer</summary>**b)** Specified in ES2015; only JavaScriptCore shipped it.</details>

**7.** Short — What is referential transparency and what does it buy you?
<details><summary>Answer</summary>An expression can be replaced by its value without changing the program. It makes memoisation always safe, tests need no setup, and code reorderable or parallelisable.</details>

**8.** Short — Why put the data argument last?
<details><summary>Answer</summary>So partially applying the configuration arguments yields a one-argument function that slots directly into `pipe` — `replace(/\s/g, '')` becomes a usable transformer.</details>

**9.** Short — What is structural sharing?
<details><summary>Answer</summary>An immutable update copies only the nodes on the path from the root to the change; every untouched branch is reused by reference, so the cost is proportional to depth rather than to data size.</details>

**10.** Short — Difference between `map` and `flatMap` on a `Maybe`?
<details><summary>Answer</summary>`map` applies a plain function to the contained value and rewraps it. `flatMap` applies a function that itself returns a `Maybe`, avoiding a nested `Maybe<Maybe<T>>`.</details>

**11.** Short — When is `Result` better than throwing?
<details><summary>Answer</summary>When failure is an expected outcome — validation, parsing, a lookup that may miss. It puts the failure in the return value where the caller can't silently forget it, whereas a `throw` is invisible in the signature.</details>

**12.** Short — Why is a transducer faster than chained `filter().map()`?
<details><summary>Answer</summary>The chained version allocates an intermediate array per step and traverses the data once per step. A transducer composes the transformations, so a single pass applies all of them with no intermediates.</details>

**13.** Output —
```js
const o = Object.freeze({ a: { b: 1 } });
o.a.b = 2;
o.a = {};
console.log(o.a.b, Object.keys(o));
```
<details><summary>Answer</summary>`2 ['a']` — the nested write succeeded because freeze is shallow; the top-level write was silently ignored (non-strict).</details>

**14.** Output —
```js
const nums = [10, 9, 1];
console.log(nums.sort());
console.log(nums.toSorted((a, b) => a - b));
console.log(nums);
```
<details><summary>Answer</summary>`[1, 10, 9]` (default sort is lexicographic and it mutated `nums`), then `[1, 9, 10]`, then `[1, 10, 9]` — `nums` retains the mutation from the first call.</details>

**15.** Output —
```js
const pipe = (...fns) => (x) => fns.reduce((a, f) => f(a), x);
console.log(pipe(x => x + 1, x => x * 2, String)(3));
```
<details><summary>Answer</summary>`'8'` — `(3+1)*2 = 8`, then stringified.</details>

**16.** Output —
```js
const memo = new Map();
const key = (...a) => JSON.stringify(a);
console.log(key({ a: 1, b: 2 }) === key({ b: 2, a: 1 }));
```
<details><summary>Answer</summary>`false` — key order changes the serialisation, so logically equal arguments miss the cache.</details>

**17.** Debug — A React component's list re-renders unpredictably after a "sort by name" button is added.
<details><summary>Answer</summary>`items.sort()` mutates the array in place, so props or state changed identity-in-place without a re-render being triggered correctly. Replace with `items.toSorted(...)`, or `[...items].sort(...)`.</details>

**18.** Debug — A deep clone loses every `Date` and turns `Map`s into `{}`.
<details><summary>Answer</summary>It's `JSON.parse(JSON.stringify(x))`, which serialises `Date` to a string and has no representation for `Map`. Use `structuredClone`.</details>

**19.** Debug — A memoised selector returns stale data after the store updates.
<details><summary>Answer</summary>The cache key doesn't include everything the function reads — most likely it keys on an id while the function also closes over mutable state. Make the function pure by passing all its inputs as arguments, and include them all in the key.</details>

**20.** Debug — `deepFreeze` on a large state object throws `RangeError: Maximum call stack size exceeded`.
<details><summary>Answer</summary>The object graph contains a cycle, so the recursion never terminates. Track visited objects in a `WeakSet` and skip anything already seen — the `Object.isFrozen` guard in the version above handles the common case but not a cycle through an unfrozen node.</details>

**21.** Debug — A curried function silently returns another function instead of a result.
<details><summary>Answer</summary>`fn.length` doesn't count parameters after a default value or a rest parameter, so `curry` never believes it has enough arguments. Pass the arity explicitly: `curry(fn, 3)`.</details>

**22.** Code — Implement `pipe` and `pipeAsync`.
<details><summary>Answer</summary>

```js
const pipe = (...fns) => (x) => fns.reduce((acc, fn) => fn(acc), x);
const pipeAsync = (...fns) => (x) => fns.reduce((p, fn) => p.then(fn), Promise.resolve(x));
```
</details>

**23.** Code — Write `deepFreeze` that handles cycles.
<details><summary>Answer</summary>

```js
function deepFreeze(obj, seen = new WeakSet()) {
  if (obj === null || typeof obj !== 'object' || seen.has(obj)) return obj;
  seen.add(obj);
  for (const value of Object.values(obj)) deepFreeze(value, seen);
  return Object.freeze(obj);
}
```
</details>

**24.** Code — Write `memoize` with an LRU bound.
<details><summary>Answer</summary>

```js
function memoize(fn, { max = 100, keyFn = (...a) => a.join('|') } = {}) {
  const cache = new Map();                 // insertion order == LRU order
  return (...args) => {
    const key = keyFn(...args);
    if (cache.has(key)) {
      const value = cache.get(key);
      cache.delete(key); cache.set(key, value);      // refresh recency
      return value;
    }
    const value = fn(...args);
    cache.set(key, value);
    if (cache.size > max) cache.delete(cache.keys().next().value);
    return value;
  };
}
```
</details>

**25.** Code — Write `groupBy` and `partition` without mutating the input.
<details><summary>Answer</summary>

```js
const groupBy = (arr, keyFn) => Object.groupBy(arr, keyFn);   // ES2024, or:
const groupByFallback = (arr, keyFn) =>
  arr.reduce((acc, x) => { const k = keyFn(x); (acc[k] ??= []).push(x); return acc; }, {});

const partition = (arr, predicate) =>
  arr.reduce(([yes, no], x) => predicate(x) ? [[...yes, x], no] : [yes, [...no, x]], [[], []]);
```
The `partition` above is O(n²) from re-spreading; for large arrays push into two preallocated
arrays instead — a case where purity inside the function isn't worth the cost.</details>

**26.** Scenario — A colleague wants to adopt a full FP library (Ramda, fp-ts) across the codebase. What do you weigh?
<details><summary>Answer</summary>The wins are real but narrow: composable data transformation and validation genuinely read better, and `fp-ts`-style `Result` types catch a class of forgotten-error bug. The costs are a learning curve for everyone who joins, stack traces that name `_curry2` instead of your function, and a bundle-size hit for utilities that are now mostly in the language (`?.`, `??`, `Object.groupBy`, `toSorted`). I'd adopt the *patterns* first — pure core, `pipe`, `Result` for validation, all writable in twenty lines — and only take the dependency if a concrete pain persists.</details>

**27.** Scenario — An image-processing loop over a million pixels is 40× slower after being rewritten in immutable style.
<details><summary>Answer</summary>This is where FP is the wrong tool: every pixel update allocates, and a generational collector is being asked to scavenge continuously. Mutate a typed array in place, keeping the function pure at its *boundary* — it takes input data and returns a new buffer, with all mutation local and invisible to callers. That gives testability and referential transparency where they matter without paying per-pixel allocation. Purity is a property of the interface; the implementation is free to be imperative.</details>

---

## Chapter Summary

- Pure means deterministic *and* effect-free; push effects to the edges and the core needs no mocks.
- `Object.freeze` is shallow, and `sort`/`reverse`/`splice` mutate — the ES2023 `to*` methods exist for this.
- `pipe` reads left to right and beats `compose` for legibility; put the data argument last so currying works.
- Memoise only pure functions, always with a bound, and use `WeakMap` keys for object arguments.
- There's no TCO in practice: trampoline or use an explicit stack for deep recursion.
- `Result` beats exceptions for expected failures because the caller can't forget it; `?.` already covers most of `Maybe`.

---

## Practice Exercises

1. Refactor a function that fetches, transforms, and saves into a pure core plus a three-line impure shell.
2. Implement `setIn`, `updateIn`, and `deleteIn` for nested immutable updates, with array-index support.
3. Write `curry` that respects an explicit arity and handles placeholder arguments.
4. Build a `Result`-returning parser combinator that parses a comma-separated list of integers.
5. Benchmark chained `filter().map().reduce()` against a transducer at 100, 10,000, and 1,000,000 elements, and identify where the crossover is.

---

## Mini Project: An Immutable Store with Time Travel

Build a state container in which every update produces a new state and history is replayable.

- `dispatch(action)` runs a pure reducer; the store never mutates a previous state
- Structural sharing, so a change to one slice doesn't copy the others — prove it with `===`
- `undo()` and `redo()` over a bounded history (say 50 entries), with the bound enforced
- `subscribe(selector, callback)` that only fires when the selected slice changes identity
- A middleware chain implemented with `pipe`, including a logger and a validator
- Serialise the action log and replay it into an identical final state
- Development-mode `deepFreeze` on every state so accidental mutation throws immediately, compiled out in production

---

**Next:** [Chapter 30 — Design Patterns →](30-design-patterns.md)
