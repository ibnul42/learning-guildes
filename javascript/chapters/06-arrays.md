# Chapter 6 — Arrays

## Learning Objectives

- Know which array methods mutate and which return new arrays
- Use `map`/`filter`/`reduce` idiomatically and know when not to
- Handle sparse arrays, `sort` pitfalls, and array-likes
- Apply ES2023+ non-mutating methods (`toSorted`, `with`, `at`)

---

## 6.1 Creating Arrays

```js
const a = [1, 2, 3];
Array.of(7);              // [7]      — unlike Array(7)
Array(7);                 // 7 empty slots (sparse!)
Array.from('abc');        // ['a','b','c']
Array.from({ length: 3 }, (_, i) => i * 2);   // [0, 2, 4]
Array.from(document.querySelectorAll('li'));  // NodeList → Array
[...new Set([1,1,2])];    // [1, 2] — dedupe
new Array(3).fill(0);     // [0,0,0]
```

> ⚠️ `Array(7)` creates **holes**, not `undefined`s. `map` skips holes; `fill` doesn't. `Array(3).map(x => 1)` returns three holes.

---

## 6.2 Mutating vs Non-Mutating — Memorise This Table

| Mutates | Non-mutating equivalent |
|---|---|
| `push`, `pop`, `shift`, `unshift` | `[...a, x]`, `a.slice(0,-1)`, `a.slice(1)`, `[x, ...a]` |
| `splice` | `toSpliced` (ES2023) |
| `sort` | `toSorted` (ES2023) |
| `reverse` | `toReversed` (ES2023) |
| `fill`, `copyWithin` | `map` |
| `a[i] = v` | `with(i, v)` (ES2023) |

Never-mutating: `slice`, `concat`, `map`, `filter`, `reduce`, `flat`, `flatMap`, `join`, `indexOf`, `includes`, `find`, `some`, `every`, `at`, `entries`, `keys`, `values`.

```js
const nums = [3, 1, 2];
const sorted = nums.toSorted();   // [1,2,3]; nums unchanged  ✅
const bad = nums.sort();          // nums IS NOW [1,2,3]      ⚠️
```

---

## 6.3 The Iteration Methods

```js
const users = [
  { name: 'Ada', age: 36, active: true },
  { name: 'Bob', age: 24, active: false },
  { name: 'Cy',  age: 41, active: true },
];

users.map(u => u.name);                  // ['Ada','Bob','Cy']
users.filter(u => u.active);             // 2 objects
users.find(u => u.age > 30);             // first match (or undefined)
users.findIndex(u => u.age > 30);        // 0
users.findLast(u => u.active);           // ES2023 — Cy
users.some(u => !u.active);              // true  — short-circuits
users.every(u => u.age > 20);            // true  — short-circuits
users.reduce((sum, u) => sum + u.age, 0);// 101
users.flatMap(u => u.active ? [u.name] : []);  // map + flatten, a filter-map in one
users.at(-1);                            // last element (ES2022)
```

### `reduce` properly

```js
// Sum
[1,2,3].reduce((acc, n) => acc + n, 0);

// Group (or just use Object.groupBy in ES2024)
users.reduce((acc, u) => {
  (acc[u.active] ??= []).push(u.name);
  return acc;
}, {});

// Build a Map index
users.reduce((m, u) => m.set(u.name, u), new Map());

// Pipe functions
[f, g, h].reduce((v, fn) => fn(v), input);
```

**Always pass an initial value.** Without it, `reduce` uses element 0 as the seed and throws on an empty array.

> 💼 **Interview** — "When would you *not* use `reduce`?" When a `for...of` is clearer. `reduce` shines for folding into a single value; it's often abused to do what `map`+`filter` say better.

---

## 6.4 Sorting Correctly

```js
[10, 9, 1].sort();               // [1, 10, 9]  ← string comparison!
[10, 9, 1].sort((a, b) => a - b); // [1, 9, 10] ✅

// Strings with locale/case
names.sort((a, b) => a.localeCompare(b));

// Multi-key
items.toSorted((a, b) =>
  a.priority - b.priority || a.name.localeCompare(b.name)
);

// Booleans
items.toSorted((a, b) => Number(b.pinned) - Number(a.pinned));
```

`sort` is **stable** as of ES2019 — equal elements keep their relative order. Rely on it.

The comparator must return a number and be *consistent*: `f(a,b)` and `f(b,a)` must have opposite signs. Returning `true`/`false` is a bug (`true` coerces to 1, `false` to 0, so "less than" is never expressed).

---

## 6.5 Flattening, Searching, Copying

```js
[1,[2,[3,[4]]]].flat();          // [1,2,[3,[4]]] — depth 1 by default
[1,[2,[3,[4]]]].flat(Infinity);  // [1,2,3,4]

[1,2,3].includes(2);             // true — uses SameValueZero, finds NaN
[NaN].indexOf(NaN);              // -1
[NaN].includes(NaN);             // true

const shallow = arr.slice();     // or [...arr]
const deep = structuredClone(arr);
```

---

## 6.6 Common Mistakes and Best Practices

**Mistake: mutating props/state.** `props.items.sort()` mutates the caller's array. Use `toSorted` or copy first.

**Mistake: `forEach` with async.**
```js
// ❌ doesn't wait
items.forEach(async (i) => { await save(i); });
console.log('done');   // logs before any save finishes

// ✅ sequential
for (const i of items) await save(i);
// ✅ parallel
await Promise.all(items.map(i => save(i)));
```

**Mistake: `delete arr[i]`.** Leaves a hole and doesn't change `length`. Use `splice` or `filter`.

**Mistake: `arr.length = 0` to clear a shared array.** It works, but mutates every reference. Usually you want a new `[]`.

**Mistake: chaining five methods over 100k items.** Each allocates. Fuse into one pass when it matters.

**Best practice: `Array.isArray`, never `typeof`.**

**Best practice: name your predicates.** `filter(isActive)` beats `filter(u => u.active === true)` in a long chain.

---

## 6.7 Real-World Example: A Data Table Pipeline

```js
/**
 * Filter → sort → paginate, without mutating the source.
 */
function queryTable(rows, { search = '', sortBy, sortDir = 'asc', page = 1, pageSize = 20 } = {}) {
  const needle = search.trim().toLowerCase();

  const filtered = needle
    ? rows.filter(r =>
        Object.values(r).some(v => String(v).toLowerCase().includes(needle)))
    : rows;

  const sorted = sortBy
    ? filtered.toSorted((a, b) => {
        const [x, y] = sortDir === 'asc' ? [a[sortBy], b[sortBy]] : [b[sortBy], a[sortBy]];
        if (typeof x === 'number' && typeof y === 'number') return x - y;
        return String(x).localeCompare(String(y), undefined, { numeric: true });
      })
    : filtered;

  const total = sorted.length;
  const pages = Math.max(1, Math.ceil(total / pageSize));
  const safePage = Math.min(Math.max(1, page), pages);
  const start = (safePage - 1) * pageSize;

  return { rows: sorted.slice(start, start + pageSize), total, pages, page: safePage };
}
```

`toSorted` matters here: the source `rows` array is often shared React state, and `sort` would mutate it in place, breaking change detection.

---

## 6.8 Questions

**1.** MCQ — Which mutates? a) `slice` b) `splice` c) `concat` d) `map`
<details><summary>Answer</summary>**b)**</details>

**2.** MCQ — `[1,2,3].map(x => x*2)` returns: a) `[1,2,3]` b) `[2,4,6]` c) `undefined` d) `6`
<details><summary>Answer</summary>**b)**</details>

**3.** MCQ — `[10,9,1].sort()` gives: a) `[1,9,10]` b) `[1,10,9]` c) `[10,9,1]` d) error
<details><summary>Answer</summary>**b)** Default sort is lexicographic.</details>

**4.** MCQ — `arr.at(-1)` returns: a) `undefined` b) first element c) last element d) error
<details><summary>Answer</summary>**c)**</details>

**5.** MCQ — `reduce` without an initial value on `[]`: a) returns `undefined` b) returns `0` c) throws TypeError d) returns `[]`
<details><summary>Answer</summary>**c)**</details>

**6.** MCQ — Which finds `NaN` in an array? a) `indexOf` b) `includes` c) both d) neither
<details><summary>Answer</summary>**b)**</details>

**7.** MCQ — Since ES2019, `sort` is: a) unstable b) stable c) random d) always numeric
<details><summary>Answer</summary>**b)**</details>

**8.** Short — `map` vs `forEach`?
<details><summary>Answer</summary>`map` returns a new array of the callback's return values; `forEach` returns `undefined` and exists for side effects.</details>

**9.** Short — Why does `Array(3).map(x => 1)` return holes?
<details><summary>Answer</summary>`Array(3)` creates a sparse array; iteration methods skip holes.</details>

**10.** Short — How do you flatten arbitrarily deep arrays?
<details><summary>Answer</summary>`arr.flat(Infinity)`.</details>

**11.** Short — Why prefer `toSorted` over `sort` in a React component?
<details><summary>Answer</summary>`sort` mutates in place; mutating state breaks referential-equality change detection and can corrupt shared data.</details>

**12.** Short — What does `flatMap` do that `map` can't?
<details><summary>Answer</summary>Returns zero, one, or many items per input — so it can filter and map in one pass.</details>

**13.** Output — `console.log([1,2,3].map(String), ['1','2','3'].map(Number));`
<details><summary>Answer</summary>`['1','2','3'] [1,2,3]`</details>

**14.** Output — `console.log(['1','7','11'].map(parseInt));`
<details><summary>Answer</summary>`[1, NaN, 3]` — `map` passes the index as `parseInt`'s radix: `parseInt('7',1)` is `NaN`, `parseInt('11',2)` is `3`. Fix with `.map(s => parseInt(s, 10))`.</details>

**15.** Output — What prints?
```js
const a = [1,2,3];
const b = a;
b.push(4);
console.log(a.length);
```
<details><summary>Answer</summary>`4` — same array by reference.</details>

**16.** Output — What prints?
```js
const arr = [1,2,3];
delete arr[1];
console.log(arr, arr.length);
```
<details><summary>Answer</summary>`[1, <1 empty item>, 3] 3`</details>

**17.** Output — `console.log([].reduce((a,b)=>a+b, 0), [].every(x=>false), [].some(x=>true));`
<details><summary>Answer</summary>`0 true false` — `every` on empty is vacuously true.</details>

**18.** Output — `console.log([1,[2,[3]]].flat().length);`
<details><summary>Answer</summary>`3` — `[1, 2, [3]]`.</details>

**19.** Debug — Sorting by score gives wrong results:
```js
users.sort((a, b) => a.score > b.score);
```
<details><summary>Answer</summary>The comparator returns a boolean. Return a number: `a.score - b.score`.</details>

**20.** Debug — All saves report success but nothing is saved before the response is sent:
```js
items.forEach(async (i) => await save(i));
res.send('ok');
```
<details><summary>Answer</summary>`forEach` ignores the returned promises. Use `await Promise.all(items.map(save))`.</details>

**21.** Debug — A "remove item" function occasionally removes the wrong one:
```js
const i = list.indexOf(item);
list.splice(i, 1);
```
<details><summary>Answer</summary>If `item` isn't found, `indexOf` returns `-1` and `splice(-1, 1)` deletes the **last** element. Guard with `if (i !== -1)`.</details>

**22.** Code — Implement `groupBy(arr, keyFn)` without `Object.groupBy`.
<details><summary>Answer</summary>

```js
const groupBy = (arr, keyFn) => arr.reduce((acc, item) => {
  (acc[keyFn(item)] ??= []).push(item);
  return acc;
}, {});
```
</details>

**23.** Code — Implement `uniqueBy(arr, keyFn)` preserving first occurrence.
<details><summary>Answer</summary>

```js
function uniqueBy(arr, keyFn) {
  const seen = new Set();
  return arr.filter(x => {
    const k = keyFn(x);
    return seen.has(k) ? false : (seen.add(k), true);
  });
}
```
</details>

**24.** Code — Implement `zip(...arrays)`.
<details><summary>Answer</summary>

```js
const zip = (...arrays) =>
  Array.from({ length: Math.min(...arrays.map(a => a.length)) },
             (_, i) => arrays.map(a => a[i]));
```
</details>

**25.** Code — Rotate an array right by `k` positions, non-mutating.
<details><summary>Answer</summary>

```js
function rotate(arr, k) {
  if (arr.length === 0) return [];
  const n = ((k % arr.length) + arr.length) % arr.length;   // handles negatives
  return [...arr.slice(-n || arr.length), ...arr.slice(0, -n || arr.length)];
}
```
Simpler: `[...arr.slice(arr.length - n), ...arr.slice(0, arr.length - n)]`.
</details>

**26.** Code — Implement `chunkBy(arr, predicate)` splitting into runs whenever the predicate changes.
<details><summary>Answer</summary>

```js
function chunkBy(arr, fn) {
  return arr.reduce((out, item, i) => {
    if (i === 0 || fn(item) !== fn(arr[i - 1])) out.push([]);
    out.at(-1).push(item);
    return out;
  }, []);
}
```
</details>

**27.** Scenario — A list of 200,000 rows re-renders slowly because you run `filter().map().sort()` on every keystroke. Fixes?
<details><summary>Answer</summary>Debounce the input (Chapter 24); memoize the filtered result keyed by the query; fuse the chain into one pass; sort once and keep a sorted copy; and virtualize rendering so only visible rows exist in the DOM.</details>

**28.** Scenario — Why might `Array.from({length: 1e7})` be a bad idea?
<details><summary>Answer</summary>It allocates 10 million slots at once — hundreds of megabytes. Use a generator to stream values lazily instead (Chapter 14).</details>

---

## Chapter Summary

- Learn the mutating/non-mutating split cold. ES2023 gave us `toSorted`, `toReversed`, `toSpliced`, `with`.
- `sort` needs a numeric comparator; the default is lexicographic. Sort is stable.
- `reduce` always takes an initial value.
- `forEach` ignores promises — use `for...of` + `await` or `Promise.all(map(...))`.
- `includes` finds `NaN`; `indexOf` doesn't.
- `Array(n)` makes holes; `Array.from({length:n}, fn)` makes values.

---

## Practice Exercises

1. Implement `flat(depth)` yourself, recursively and iteratively.
2. Write `intersection`, `union`, and `difference` for arrays of objects, keyed by a field.
3. Implement a stable `sortBy(arr, ...keyFns)` supporting per-key direction.
4. Build `partition(arr, predicate)` returning `[matches, rest]` in a single pass.
5. Benchmark `filter().map()` vs a single `reduce` on 1e6 numbers. Report the difference honestly.

---

## Mini Project: A Query Engine

Build `query.js` exporting a chainable, lazy query builder:

```js
query(users).where(u => u.active).orderBy('age', 'desc').select('name').take(5).toArray();
```

**Requirements:**

- Nothing executes until a terminal method (`toArray`, `first`, `count`, `sum`) is called.
- Never mutate the source array.
- Support `groupBy(keyFn)` returning a `Map`.
- Support `orderBy` chained with `thenBy`.
- Include `distinctBy`, `skip`, `take`, `flatMap`.

**Stretch:** back it with generators so a 10-million-item source streams without materializing intermediates.

---

**Next:** [Chapter 7 — Strings, Numbers, Math, Dates →](07-strings-numbers-math-dates.md)
