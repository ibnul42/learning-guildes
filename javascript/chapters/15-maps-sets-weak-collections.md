# Chapter 15 — Map, Set, WeakMap, WeakSet

## Learning Objectives

- Choose between `Map` and a plain object, and between `Set` and an array
- Understand SameValueZero key equality
- Use weak collections to avoid memory leaks

---

## 15.1 Map

```js
const m = new Map([['a', 1], ['b', 2]]);
m.set('c', 3).set(objKey, 'any value');   // set() is chainable
m.get('a');        // 1
m.has('a');        // true
m.delete('a');     // true
m.size;            // number, not a method
m.clear();

for (const [k, v] of m) {}
[...m.keys()]; [...m.values()]; [...m.entries()];
Object.fromEntries(m);         // → plain object (string keys only)
new Map(Object.entries(obj));  // ← from plain object
```

### Map vs Object

| | `Map` | Object |
|---|---|---|
| Key types | **any** (objects, functions, NaN) | string / symbol only |
| Order | insertion order, guaranteed | integer keys first, then insertion |
| Size | `.size`, O(1) | `Object.keys(o).length`, O(n) |
| Iteration | directly iterable | needs `Object.entries` |
| Prototype keys | none | inherits from `Object.prototype` |
| JSON | needs conversion | native |
| Frequent add/delete | optimized | slower |

**Use a `Map` when:** keys aren't strings, you add/remove often, you need `.size`, or keys come from untrusted input (no `__proto__` hazard).

**Use an object when:** you need JSON serialization, the shape is fixed and known, or you want destructuring.

```js
// Object keys stringify — every object collides
const o = {};
o[{a:1}] = 'x'; o[{b:2}] = 'y';
Object.keys(o);   // ['[object Object]'] — one key!

// Map keys are identity-based
const map = new Map();
map.set({a:1}, 'x').set({b:2}, 'y');
map.size;   // 2
```

Key equality is **SameValueZero**: `NaN` matches `NaN`, and `0` matches `-0`.

---

## 15.2 Set

```js
const s = new Set([1, 2, 2, 3]);   // {1,2,3} — duplicates dropped
s.add(4).add(4);                    // chainable, still one 4
s.has(2); s.delete(2); s.size;
[...s];                             // back to an array

// Dedupe
[...new Set(arr)];

// Set operations (ES2025)
a.union(b); a.intersection(b); a.difference(b);
a.symmetricDifference(b); a.isSubsetOf(b); a.isSupersetOf(b); a.isDisjointFrom(b);
```

`Set.has()` is O(1); `Array.includes()` is O(n). For membership checks in a loop, that's the difference between O(n) and O(n²):

```js
// ❌ O(n*m)
const dupes = a.filter(x => b.includes(x));
// ✅ O(n+m)
const bSet = new Set(b);
const dupes = a.filter(x => bSet.has(x));
```

> ⚠️ Sets dedupe by identity for objects. `new Set([{a:1},{a:1}]).size` is `2`. Dedupe by a key instead (Chapter 6, `uniqueBy`).

---

## 15.3 WeakMap and WeakSet

Keys must be objects (or non-registered symbols), and references are **weak**: if nothing else holds the key, it's garbage collected along with its entry.

```js
const wm = new WeakMap();
let key = {};
wm.set(key, 'metadata');
key = null;      // entry becomes eligible for GC automatically
```

**Not available:** `size`, iteration, `clear()`, `keys()`. You cannot enumerate a weak collection — if you could, GC would be observable.

### The two real uses

**1. Metadata about objects you don't own:**

```js
const listeners = new WeakMap();   // DOM node -> handlers

function track(node, handler) {
  if (!listeners.has(node)) listeners.set(node, new Set());
  listeners.get(node).add(handler);
}
// When the node is removed from the DOM and dereferenced,
// the entry vanishes. With a Map, it would leak forever.
```

**2. Private state (pre-`#fields`):**

```js
const privates = new WeakMap();
class Counter {
  constructor() { privates.set(this, { count: 0 }); }
  inc() { return ++privates.get(this).count; }
}
```

`#private` fields have superseded this pattern, but it appears throughout older libraries.

```js
// WeakSet — tag objects without mutating them
const processed = new WeakSet();
function process(obj) {
  if (processed.has(obj)) return;
  processed.add(obj);
  // ... expensive work
}
```

Also worth knowing: `WeakRef` and `FinalizationRegistry` exist for advanced cache scenarios, but their timing is non-deterministic and they're rarely the right answer.

---

## 15.4 Common Mistakes and Best Practices

**Mistake: `JSON.stringify(map)`** → `'{}'`. Convert first: `Object.fromEntries(map)` or `[...map]`.

**Mistake: `map.length`.** It's `.size`.

**Mistake: using a `Map` where an object is clearer.** Fixed-shape config data belongs in an object.

**Mistake: a `Map` cache keyed by DOM nodes or component instances.** That's a leak — use a `WeakMap`.

**Mistake: expecting a `WeakMap` entry to disappear immediately.** GC timing is unspecified.

**Best practice: `Set` for membership**, `Map` for lookup, arrays for ordered data you'll iterate and transform.

---

## 15.5 Real-World Example: An LRU Cache

```js
/**
 * Map preserves insertion order, so the first key is the least recently used.
 * Re-inserting on access moves an entry to the end. That's the whole algorithm.
 */
class LRUCache {
  #map = new Map();
  #max;

  constructor(max = 100) { this.#max = max; }

  get(key) {
    if (!this.#map.has(key)) return undefined;
    const value = this.#map.get(key);
    this.#map.delete(key);
    this.#map.set(key, value);      // move to most-recent position
    return value;
  }

  set(key, value) {
    if (this.#map.has(key)) this.#map.delete(key);
    else if (this.#map.size >= this.#max) {
      this.#map.delete(this.#map.keys().next().value);   // evict oldest
    }
    this.#map.set(key, value);
    return this;
  }

  get size() { return this.#map.size; }
  has(key) { return this.#map.has(key); }
}

// Paired with a WeakMap for object-keyed data that must not leak:
const domMeasurements = new WeakMap();
function measure(el) {
  if (!domMeasurements.has(el)) domMeasurements.set(el, el.getBoundingClientRect());
  return domMeasurements.get(el);
}
```

Both operations are O(1). Doing this with an array would make eviction O(n).

---

## 15.6 Questions

**1.** MCQ — `Map` key types: a) strings b) strings+symbols c) any value d) objects only
<details><summary>Answer</summary>**c)**</details>

**2.** MCQ — `new Set([NaN, NaN]).size` is: a) 0 b) 1 c) 2 d) NaN
<details><summary>Answer</summary>**b)** SameValueZero.</details>

**3.** MCQ — `WeakMap` supports: a) `size` b) iteration c) `clear()` d) none of these
<details><summary>Answer</summary>**d)**</details>

**4.** MCQ — `JSON.stringify(new Map([['a',1]]))` gives: a) `'{"a":1}'` b) `'{}'` c) `'[["a",1]]'` d) throws
<details><summary>Answer</summary>**b)**</details>

**5.** MCQ — `Set.has` complexity: a) O(1) b) O(n) c) O(log n) d) O(n²)
<details><summary>Answer</summary>**a)**</details>

**6.** Short — When would you pick `Map` over an object?
<details><summary>Answer</summary>Non-string keys, frequent insert/delete, needing `.size`, guaranteed insertion order, or untrusted keys.</details>

**7.** Short — Why can't you iterate a `WeakMap`?
<details><summary>Answer</summary>Enumeration would make garbage-collection timing observable, breaking the weakness guarantee.</details>

**8.** Short — What is SameValueZero?
<details><summary>Answer</summary>Like `===` except `NaN` equals `NaN`. Used by `Map`/`Set` keys and `Array.includes`.</details>

**9.** Short — Give a concrete `WeakSet` use case.
<details><summary>Answer</summary>Marking objects as already-processed (cycle detection in a serializer) without preventing them from being collected.</details>

**10.** Short — Why is `Map` faster for frequent deletes?
<details><summary>Answer</summary>`delete` on a plain object can force the engine into slower dictionary mode; `Map` is designed for it.</details>

**11.** Output —
```js
const m = new Map();
m.set(NaN, 'a').set(0, 'b').set(-0, 'c');
console.log(m.size, m.get(NaN), m.get(0));
```
<details><summary>Answer</summary>`2 'a' 'c'` — `NaN` is one key; `0` and `-0` are the same key, so `'c'` overwrote `'b'`.</details>

**12.** Output — `console.log(new Set([{a:1},{a:1}]).size);`
<details><summary>Answer</summary>`2` — different object identities.</details>

**13.** Output —
```js
const m = new Map([['b',1],['a',2]]);
console.log([...m.keys()]);
```
<details><summary>Answer</summary>`['b','a']` — insertion order, not sorted (unlike an object with integer-like keys).</details>

**14.** Output —
```js
const s = new Set([1,2,3]);
for (const x of s) if (x === 2) s.add(4);
console.log([...s]);
```
<details><summary>Answer</summary>`[1,2,3,4]` — items added during iteration are visited.</details>

**15.** Output — `console.log(new Map([[1,'a']]).get('1'));`
<details><summary>Answer</summary>`undefined` — no coercion; `1` and `'1'` are distinct keys.</details>

**16.** Debug — A cache keyed by React component instances grows forever.
<details><summary>Answer</summary>A `Map` holds strong references, keeping unmounted components alive. Switch to a `WeakMap`.</details>

**17.** Debug — `map.forEach((k, v) => ...)` gives swapped values.
<details><summary>Answer</summary>The callback signature is `(value, key, map)` — value first, matching `Array.forEach`. Rename the params.</details>

**18.** Debug — Deduping objects by `new Set(users)` doesn't remove duplicates.
<details><summary>Answer</summary>Identity comparison. Dedupe by a key: `[...new Map(users.map(u => [u.id, u])).values()]`.</details>

**19.** Code — Implement `groupBy` returning a `Map`.
<details><summary>Answer</summary>

```js
const groupBy = (arr, fn) => arr.reduce((m, x) => {
  const k = fn(x);
  m.set(k, [...(m.get(k) ?? []), x]);
  return m;
}, new Map());
```
For large inputs, push into an existing array instead of spreading.
</details>

**20.** Code — Implement `intersection(a, b)` for arrays in O(n+m).
<details><summary>Answer</summary>

```js
const intersection = (a, b) => { const s = new Set(b); return a.filter(x => s.has(x)); };
```
</details>

**21.** Code — Implement a `MultiMap` (one key → many values).
<details><summary>Answer</summary>

```js
class MultiMap {
  #m = new Map();
  add(k, v) { (this.#m.get(k) ?? this.#m.set(k, new Set()).get(k)).add(v); return this; }
  get(k) { return [...(this.#m.get(k) ?? [])]; }
  delete(k, v) { return v === undefined ? this.#m.delete(k) : !!this.#m.get(k)?.delete(v); }
  get size() { return this.#m.size; }
}
```
</details>

**22.** Code — Write `memoizeByObject(fn)` caching on a single object argument without leaking.
<details><summary>Answer</summary>

```js
function memoizeByObject(fn) {
  const cache = new WeakMap();
  return (obj) => {
    if (!cache.has(obj)) cache.set(obj, fn(obj));
    return cache.get(obj);
  };
}
```
</details>

**23.** Scenario — You need a serializable cache *and* object keys. Reconcile.
<details><summary>Answer</summary>You can't serialize object identity. Derive a stable string id for each object (a `uuid` field, or a content hash) and key the cache by that. Keep a separate `WeakMap` from object → id if you need lookup by reference.</details>

**24.** Scenario — When is a plain array still the right choice over a `Set`?
<details><summary>Answer</summary>When order and indexing matter, when you need `map`/`reduce`/`sort`, when duplicates are meaningful, or when the collection is tiny (under ~10 items, `includes` is competitive and simpler).</details>

---

## Chapter Summary

- `Map`: any key type, insertion order, O(1) `.size`, no prototype hazards. Objects for fixed-shape, JSON-friendly data.
- `Set`: O(1) membership, dedupes by SameValueZero identity. ES2025 adds full set algebra.
- Weak collections hold keys weakly, take only object keys, and cannot be iterated or sized.
- Use `WeakMap`/`WeakSet` for metadata about objects whose lifetime you don't control.
- Neither `Map` nor `Set` serializes to JSON directly.

---

## Practice Exercises

1. Implement a bidirectional map (`BiMap`) keeping both directions in sync.
2. Build a graph with `Map<node, Set<node>>` and implement BFS.
3. Write `deepEqual` using a `WeakMap` to handle circular references.
4. Benchmark `Set.has` vs `Array.includes` at sizes 10, 1k, and 100k. Report where the crossover is.
5. Implement a TTL cache combining a `Map` with expiry timestamps and lazy eviction.

---

**Next:** [Chapter 16 — Error Handling →](16-error-handling.md)
