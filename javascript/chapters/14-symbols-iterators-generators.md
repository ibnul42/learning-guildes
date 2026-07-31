# Chapter 14 — Symbols, Iterators, and Generators

## Learning Objectives

- Use symbols for collision-free keys and well-known protocol hooks
- Implement the iterable and iterator protocols by hand
- Write generators, including delegation and two-way communication
- Build lazy, memory-efficient data pipelines

---

## 14.1 Symbols

A symbol is a unique, immutable primitive. Two symbols are never equal, even with the same description.

```js
const a = Symbol('id');
const b = Symbol('id');
a === b;            // false
a.description;      // 'id'
typeof a;           // 'symbol'

// Global registry — shared across realms
Symbol.for('app.id') === Symbol.for('app.id');   // true
Symbol.keyFor(Symbol.for('x'));                   // 'x'
```

**As property keys**, symbols are collision-proof and hidden from ordinary enumeration:

```js
const SECRET = Symbol('secret');
const obj = { visible: 1, [SECRET]: 'hidden' };

Object.keys(obj);                   // ['visible']
JSON.stringify(obj);                // '{"visible":1}'
for (const k in obj) {}             // only 'visible'
Object.getOwnPropertySymbols(obj);  // [Symbol(secret)] — findable if you look
```

Not *private* (use `#` for that), but safe from accidental collision — ideal for library metadata attached to user objects.

### Well-known symbols

These let you hook into language behaviour:

```js
class Range {
  constructor(a, b) { this.a = a; this.b = b; }
  *[Symbol.iterator]() { for (let i = this.a; i <= this.b; i++) yield i; }
  get [Symbol.toStringTag]() { return 'Range'; }
  [Symbol.toPrimitive](hint) {
    return hint === 'number' ? this.b - this.a : `${this.a}..${this.b}`;
  }
}

const r = new Range(1, 4);
[...r];                              // [1,2,3,4]
Object.prototype.toString.call(r);   // '[object Range]'
+r;                                  // 3
`${r}`;                              // '1..4'
```

Others worth knowing: `Symbol.asyncIterator`, `Symbol.hasInstance` (customise `instanceof`), `Symbol.species` (what subclassed built-ins return from `map`).

---

## 14.2 The Iteration Protocols

**Iterator protocol:** an object with a `next()` method returning `{ value, done }`.

**Iterable protocol:** an object with a `[Symbol.iterator]()` method returning an iterator.

`for...of`, spread, destructuring, `Array.from`, `Promise.all`, `new Set()`, and `yield*` all consume iterables.

```js
// Manual iterator — no generator
const countdown = {
  from: 3,
  [Symbol.iterator]() {
    let n = this.from;
    return {
      next: () => n >= 0 ? { value: n--, done: false } : { value: undefined, done: true },
      // Optional: called on early exit (break, return, throw)
      return: () => { console.log('cleanup'); return { done: true }; },
      [Symbol.iterator]() { return this; },   // makes the iterator itself iterable
    };
  },
};

[...countdown];   // [3, 2, 1, 0]
for (const n of countdown) { if (n === 1) break; }   // logs 'cleanup'
```

That `return()` hook is how `for...of` releases resources when you `break` — generators get it for free.

---

## 14.3 Generators

A generator function (`function*`) returns a generator object that is both an iterator and an iterable. `yield` pauses execution and hands a value out; the function's entire state is preserved.

```js
function* idGen() {
  let id = 0;
  while (true) yield ++id;      // infinite — but lazy, so it's fine
}
const ids = idGen();
ids.next();   // { value: 1, done: false }
ids.next();   // { value: 2, done: false }
```

### Delegation with `yield*`

```js
function* inner() { yield 'a'; yield 'b'; return 'inner-done'; }
function* outer() {
  const result = yield* inner();   // yields a, b; captures the RETURN value
  yield result;
}
[...outer()];   // ['a', 'b', 'inner-done']
```

`yield*` delegates to any iterable, not just generators.

> ⚠️ **Gotcha** — the first `next(value)` argument is discarded. There is no paused `yield` yet for it to become the result of, so the value vanishes. Prime the generator with a bare `next()` before sending anything in.

> 💼 **Interview** — "What does `[...outer()]` produce?" The trap is `yield*`'s return value: it's captured by the assignment, not yielded, so it only appears in the output because the next line yields it explicitly.

### Two-way communication

`next(value)` sends a value *into* the generator as the result of the paused `yield`.

```js
function* dialogue() {
  const name = yield 'What is your name?';
  const age = yield `Hello ${name}. Your age?`;
  return `${name} is ${age}`;
}
const d = dialogue();
d.next();          // { value: 'What is your name?', done: false }
d.next('Ada');     // { value: 'Hello Ada. Your age?', done: false }
d.next(36);        // { value: 'Ada is 36', done: true }
```

Note the first `next()` argument is discarded — there's no paused `yield` to receive it yet.

Also available: `gen.return(v)` (finish early, running `finally` blocks) and `gen.throw(err)` (inject an exception at the `yield`).

```js
function* withCleanup() {
  try { yield 1; yield 2; }
  finally { console.log('released'); }   // runs on .return() or break
}
```

### Async generators

```js
async function* paginate(url) {
  let next = url;
  while (next) {
    const res = await fetch(next);
    const { items, nextUrl } = await res.json();
    yield* items;                     // yield each item individually
    next = nextUrl;
  }
}

for await (const item of paginate('/api/items')) {
  console.log(item);                  // streams pages transparently
}
```

---

## 14.4 Why Laziness Matters

```js
// Eager: allocates 10 million numbers, ~80 MB, before doing anything
const evens = Array.from({ length: 1e7 }, (_, i) => i).filter(n => n % 2 === 0).slice(0, 5);

// Lazy: allocates nothing beyond the 5 results
function* naturals() { for (let i = 0; ; i++) yield i; }
function* filter(it, fn) { for (const x of it) if (fn(x)) yield x; }
function* take(it, n) { let i = 0; for (const x of it) { if (i++ >= n) return; yield x; } }

[...take(filter(naturals(), n => n % 2 === 0), 5)];   // [0,2,4,6,8]
```

Nothing computes until something pulls. Infinite sequences become usable.

---

## 14.5 Common Mistakes and Best Practices

**Mistake: treating a generator as reusable.** Once exhausted, it's done forever. Return a *function* that creates a generator, or implement `[Symbol.iterator]` on a wrapper object so each `for...of` gets a fresh one.

**Mistake: spreading an infinite generator.** `[...naturals()]` hangs. Always bound it with `take`.

**Mistake: using symbols expecting privacy.** `Object.getOwnPropertySymbols` finds them.

**Mistake: forgetting `*` on `[Symbol.iterator]`.** Without it you must return an iterator manually.

**Best practice: generators for streaming and state machines**, not for simple array transforms — `map`/`filter` are clearer and faster on small data.

**Best practice: put cleanup in `finally`.** It runs on `break`, `return()`, and `throw()`.

---

## 14.6 Real-World Example: Streaming a Large CSV

```js
/**
 * Processes a multi-gigabyte CSV with constant memory.
 * Each stage is lazy; one row is in flight at a time.
 */
async function* readLines(stream) {
  const decoder = new TextDecoder();
  let buffer = '';
  for await (const chunk of stream) {
    buffer += decoder.decode(chunk, { stream: true });
    let nl;
    while ((nl = buffer.indexOf('\n')) !== -1) {
      yield buffer.slice(0, nl).replace(/\r$/, '');
      buffer = buffer.slice(nl + 1);
    }
  }
  if (buffer) yield buffer;               // trailing line without newline
}

async function* parseCsv(lines) {
  let headers = null;
  for await (const line of lines) {
    const cells = line.split(',');
    if (!headers) { headers = cells; continue; }
    yield Object.fromEntries(headers.map((h, i) => [h, cells[i] ?? '']));
  }
}

async function* filterAsync(source, predicate) {
  for await (const row of source) if (predicate(row)) yield row;
}

// Usage — memory stays flat regardless of file size
const file = (await fetch('/data/orders.csv')).body;
const bigOrders = filterAsync(parseCsv(readLines(file)), r => Number(r.total) > 1000);

let count = 0;
for await (const row of bigOrders) {
  await save(row);
  if (++count >= 10_000) break;    // `finally` blocks in every stage run
}
```

The `break` propagates back through all three generators, letting each clean up. Loading this file into an array first would exhaust memory.

---

## 14.7 Questions

**1.** MCQ — `Symbol('a') === Symbol('a')` is: a) true b) false
<details><summary>Answer</summary>**b)**</details>

**2.** MCQ — Symbol keys appear in: a) `Object.keys` b) `JSON.stringify` c) `for...in` d) `Object.getOwnPropertySymbols`
<details><summary>Answer</summary>**d)**</details>

**3.** MCQ — An iterator's `next()` returns: a) a value b) `{value, done}` c) a promise d) an array
<details><summary>Answer</summary>**b)**</details>

**4.** MCQ — `function*` returns: a) a promise b) an array c) a generator object d) undefined
<details><summary>Answer</summary>**c)**</details>

**5.** MCQ — `yield*` does: a) yields an array b) delegates to another iterable c) returns d) throws
<details><summary>Answer</summary>**b)**</details>

**6.** MCQ — Which makes an object work with `for...of`? a) `next()` b) `[Symbol.iterator]()` c) `length` d) `forEach`
<details><summary>Answer</summary>**b)**</details>

**7.** Short — Symbol vs `#private`?
<details><summary>Answer</summary>Symbols avoid *collisions* but are discoverable via `getOwnPropertySymbols`. `#` fields are genuinely inaccessible from outside the class.</details>

**8.** Short — Iterable vs iterator?
<details><summary>Answer</summary>An iterable has `[Symbol.iterator]()`. An iterator has `next()`. Generators are both.</details>

**9.** Short — Why can't you reuse an exhausted generator?
<details><summary>Answer</summary>Its execution state is complete; every subsequent `next()` returns `{value: undefined, done: true}`. Create a new one.</details>

**10.** Short — What does `gen.return()` do?
<details><summary>Answer</summary>Terminates the generator early, running any `finally` blocks, and returns `{value, done: true}`.</details>

**11.** Short — Why is the first `next()` argument ignored?
<details><summary>Answer</summary>The generator hasn't started, so there's no suspended `yield` expression to receive the value.</details>

**12.** Short — Name three well-known symbols and what they do.
<details><summary>Answer</summary>`Symbol.iterator` (sync iteration), `Symbol.asyncIterator` (`for await`), `Symbol.toPrimitive` (coercion), `Symbol.hasInstance` (`instanceof`), `Symbol.toStringTag` (the `[object X]` tag).</details>

**13.** Output — What prints?
```js
function* g() { yield 1; yield 2; }
const it = g();
console.log([...it], [...it]);
```
<details><summary>Answer</summary>`[1,2] []` — exhausted after the first spread.</details>

**14.** Output — What prints?
```js
function* g() { const x = yield 1; console.log('got', x); }
const it = g();
it.next(); it.next(42);
```
<details><summary>Answer</summary>logs `got 42`</details>

**15.** Output — What prints?
```js
function* a() { yield 1; return 2; }
function* b() { const r = yield* a(); yield r; }
console.log([...b()]);
```
<details><summary>Answer</summary>`[1, 2]` — the `return` value becomes `r`, not a yielded item.</details>

**16.** Output — What prints?
```js
function* g() { try { yield 1; } finally { console.log('fin'); } }
for (const x of g()) break;
```
<details><summary>Answer</summary>logs `fin` — `break` triggers the iterator's `return()`.</details>

**17.** Output — What prints?
```js
const o = { [Symbol.toPrimitive](h) { return h === 'number' ? 7 : 'seven'; } };
console.log(+o, `${o}`, o + '');
```
<details><summary>Answer</summary>`7 'seven' 'seven'` — hints: number, string, default (default isn't 'number', so the string branch runs).</details>

**18.** Debug — A custom collection works with `for...of` once, then never again.
<details><summary>Answer</summary>`[Symbol.iterator]` returns a stored generator instead of creating a new one each call. Make the method itself a generator (`*[Symbol.iterator]()`) so every consumer gets a fresh iterator.</details>

**19.** Debug — A pagination generator holds an open DB cursor after the consumer breaks.
<details><summary>Answer</summary>No `finally` block. Wrap the loop in `try { ... } finally { await cursor.close(); }` — `break` triggers `return()`, which runs `finally`.</details>

**20.** Debug — `JSON.stringify` loses a field added with a symbol key.
<details><summary>Answer</summary>Expected — symbol keys are skipped by JSON. Use a string key, or add a `toJSON()` method that includes it.</details>

**21.** Code — Write `zip(...iterables)` as a generator.
<details><summary>Answer</summary>

```js
function* zip(...its) {
  const iters = its.map(i => i[Symbol.iterator]());
  while (true) {
    const results = iters.map(i => i.next());
    if (results.some(r => r.done)) return;
    yield results.map(r => r.value);
  }
}
```
</details>

**22.** Code — Write `chunk(iterable, size)` as a generator.
<details><summary>Answer</summary>

```js
function* chunk(it, size) {
  let buf = [];
  for (const x of it) { buf.push(x); if (buf.length === size) { yield buf; buf = []; } }
  if (buf.length) yield buf;
}
```
</details>

**23.** Code — Implement a `Fibonacci` class that is iterable and takes an optional limit.
<details><summary>Answer</summary>

```js
class Fibonacci {
  constructor(limit = Infinity) { this.limit = limit; }
  *[Symbol.iterator]() {
    let [a, b] = [0, 1];
    while (a <= this.limit) { yield a; [a, b] = [b, a + b]; }
  }
}
[...new Fibonacci(50)];   // [0,1,1,2,3,5,8,13,21,34]
```
</details>

**24.** Code — Write `range(start, stop, step)` as a generator supporting negative steps.
<details><summary>Answer</summary>

```js
function* range(start, stop, step = 1) {
  if (step === 0) throw new RangeError('step cannot be 0');
  if (step > 0) for (let i = start; i < stop; i += step) yield i;
  else for (let i = start; i > stop; i += step) yield i;
}
```
</details>

**25.** Scenario — You need to process a 4 GB log file in a Node script with a 512 MB memory limit. Design.
<details><summary>Answer</summary>Async generator pipeline: a read stream → line splitter → parser → filter → sink, each stage `yield`ing one record. Memory stays proportional to a single record plus buffer, not file size. Add `finally` blocks to close handles on early exit.</details>

**26.** Scenario — A library wants to attach metadata to objects it doesn't own, without risking collisions or appearing in `JSON.stringify`. Recommendation?
<details><summary>Answer</summary>A module-level `Symbol` as the key (invisible to `Object.keys`/JSON), or a `WeakMap` keyed by the object if you want zero mutation of the user's object at all — the `WeakMap` is cleaner and also avoids retaining objects (Chapter 15).</details>

---

## Chapter Summary

- Symbols are unique primitives — collision-free keys, invisible to `Object.keys`/JSON, but discoverable.
- Well-known symbols hook into `for...of`, coercion, `instanceof`, and string tags.
- Iterable = has `[Symbol.iterator]()`; iterator = has `next()` returning `{value, done}`.
- Generators pause and resume, preserving state; `yield*` delegates; `next(v)` sends values in.
- `finally` in a generator runs on `break`, `return()`, and `throw()` — put cleanup there.
- Laziness makes infinite sequences and constant-memory streaming possible.

---

## Practice Exercises

1. Implement `map`, `filter`, `take`, `takeWhile`, and `reduce` as generator-based lazy operators.
2. Build a `LinkedList` that is iterable in both directions (`[Symbol.iterator]` and a `reversed()` generator).
3. Write a generator-based tree traversal supporting depth-first and breadth-first modes.
4. Use `Symbol.hasInstance` to make `x instanceof Positive` work for positive numbers.
5. Implement a simple coroutine runner: given a generator that yields promises, drive it to completion (this is how `async/await` was polyfilled).

---

## Mini Project: A Lazy Sequence Library

Build `seq.js` exporting `seq(iterable)` returning a chainable lazy wrapper.

**Requirements:**

- `.map`, `.filter`, `.take`, `.drop`, `.flatMap`, `.zip`, `.chunk`, `.distinct` — all lazy.
- Terminal ops: `.toArray()`, `.reduce()`, `.forEach()`, `.first()`, `.count()`.
- Works on infinite sources: `seq(naturals()).filter(isPrime).take(100).toArray()`.
- The wrapper is itself iterable, so `for...of` works directly.
- An `.async()` mode returning an async-iterable pipeline.
- Prove constant memory with a 10-million-element source and a heap measurement.

---

**Next:** [Chapter 15 — Map, Set, WeakMap, WeakSet →](15-maps-sets-weak-collections.md)
