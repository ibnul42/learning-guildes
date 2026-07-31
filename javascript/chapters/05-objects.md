# Chapter 5 — Objects

## Learning Objectives

- Create, access, and mutate objects fluently, including computed and shorthand syntax
- Understand property descriptors, enumerability, and `Object.defineProperty`
- Copy objects correctly (shallow vs deep) and know the pitfalls of each approach
- Apply immutability with `Object.freeze` and structural sharing
- Use getters, setters, and `Proxy` deliberately

---

## 5.1 Creating and Accessing

```js
const key = 'dynamic';
const value = 42;

const obj = {
  literal: 1,
  'needs-quotes': 2,
  [key]: 3,              // computed key → { dynamic: 3 }
  value,                 // shorthand → { value: 42 }
  method() { return this.literal; },   // method shorthand
  get double() { return this.literal * 2; },
  set double(v) { this.literal = v / 2; },
};

obj.literal;          // dot: static keys only
obj['needs-quotes'];  // bracket: dynamic or invalid-identifier keys
obj?.missing?.deep;   // optional chaining
delete obj.literal;   // removes the property entirely
'literal' in obj;     // false now
```

**All object keys are strings or symbols.** Numeric keys are coerced to strings:

```js
const o = { 1: 'a' };
o[1] === o['1'];   // true
```

Integer-like keys iterate first, in ascending numeric order; other string keys follow in insertion order. This ordering is specified, not incidental.

```js
console.log(Object.keys({ b: 1, 2: 2, a: 3, 1: 4 }));
// ['1', '2', 'b', 'a']
```

---

## 5.2 Iterating and Inspecting

```js
Object.keys(o);       // own enumerable string keys
Object.values(o);
Object.entries(o);    // [[k, v], ...]
Object.fromEntries(pairs);          // the inverse
Object.getOwnPropertyNames(o);      // includes non-enumerable
Object.getOwnPropertySymbols(o);
Object.hasOwn(o, 'k');              // ES2022 — replaces hasOwnProperty
Object.groupBy(items, item => item.type);   // ES2024
```

`Object.hasOwn` exists because `o.hasOwnProperty('k')` breaks on objects created with `Object.create(null)` or objects that shadow the method.

Transform an object with `entries` + `fromEntries`:

```js
const prices = { apple: 1, pear: 2 };
const doubled = Object.fromEntries(
  Object.entries(prices).map(([k, v]) => [k, v * 2])
);   // { apple: 2, pear: 4 }
```

---

## 5.3 Property Descriptors

Every property has four attributes.

```js
const o = {};
Object.defineProperty(o, 'id', {
  value: 1,
  writable: false,      // cannot be reassigned
  enumerable: false,    // hidden from Object.keys, for...in, spread, JSON
  configurable: false,  // cannot be deleted or redefined
});

Object.getOwnPropertyDescriptor(o, 'id');
// { value: 1, writable: false, enumerable: false, configurable: false }
```

**Defaults matter:** properties created by assignment (`o.x = 1`) get all three flags `true`. Properties created by `defineProperty` default to all `false`.

Accessor descriptors use `get`/`set` instead of `value`/`writable`:

```js
Object.defineProperty(o, 'name', {
  get() { return this._name?.toUpperCase(); },
  set(v) { this._name = v.trim(); },
  enumerable: true,
});
```

---

## 5.4 Copying

```js
const source = { a: 1, nested: { b: 2 } };

// Shallow — nested is shared by reference
const s1 = { ...source };
const s2 = Object.assign({}, source);
s1.nested.b = 99;
console.log(source.nested.b);   // 99  ← mutation leaked

// Deep — structuredClone (ES2022 / broadly available)
const deep = structuredClone(source);
deep.nested.b = 7;
console.log(source.nested.b);   // 99 unchanged

// Deep via JSON — lossy, avoid
JSON.parse(JSON.stringify(source));
```

| Method | Depth | Handles Date/Map/Set | Handles functions | Handles cycles |
|---|---|---|---|---|
| spread / `Object.assign` | shallow | n/a | copies reference | n/a |
| `structuredClone` | deep | ✅ | ❌ throws | ✅ |
| `JSON.parse(JSON.stringify())` | deep | ❌ Date→string, Map→`{}` | ❌ dropped | ❌ throws |

> ⚠️ `structuredClone` throws on functions, DOM nodes, and symbols. For objects containing those, write a custom clone.

**Spread copies own *enumerable* properties only** — getters are invoked and flattened to values, and the prototype is not copied. Use `Object.create(Object.getPrototypeOf(o), Object.getOwnPropertyDescriptors(o))` for a faithful shallow copy.

---

## 5.5 Immutability

```js
const config = Object.freeze({ host: 'x', nested: { port: 80 } });
config.host = 'y';          // silently ignored (throws in strict mode)
config.nested.port = 443;   // ✅ works — freeze is SHALLOW

Object.isFrozen(config);    // true
Object.seal(o);             // no add/delete, but existing props writable
Object.preventExtensions(o); // no add, but delete/write allowed
```

Deep freeze:

```js
function deepFreeze(o) {
  for (const v of Object.values(o)) {
    if (v && typeof v === 'object' && !Object.isFrozen(v)) deepFreeze(v);
  }
  return Object.freeze(o);
}
```

In practice, most teams get immutability from *discipline plus structural sharing* rather than freezing everything:

```js
// Update without mutating
const updated = { ...state, user: { ...state.user, name: 'Ada' } };
```

---

## 5.6 Proxy and Reflect

```js
const target = { a: 1 };

const safe = new Proxy(target, {
  get(obj, prop, receiver) {
    if (!(prop in obj)) throw new ReferenceError(`No property "${String(prop)}"`);
    return Reflect.get(obj, prop, receiver);
  },
  set(obj, prop, value, receiver) {
    if (typeof value !== 'number') throw new TypeError('numbers only');
    return Reflect.set(obj, prop, value, receiver);
  },
});

safe.a;        // 1
safe.b;        // ReferenceError
safe.a = 'x';  // TypeError
```

Traps include `get`, `set`, `has`, `deleteProperty`, `ownKeys`, `apply`, `construct`. `Reflect` provides the default behaviour for each trap, so you forward rather than reimplement.

Real uses: validation, observable state (Vue 3's reactivity), negative array indices, logging, access control. Proxies add real overhead — don't wrap hot-path objects.

---

## 5.7 Common Mistakes and Best Practices

**Mistake: comparing objects with `===`.** It compares identity. Write a `deepEqual` or compare a canonical key.

**Mistake: assuming spread is a deep copy.** It isn't, and the resulting shared-reference bugs are hard to trace.

**Mistake: `for...in` without `Object.hasOwn`.** Picks up inherited properties.

**Mistake: using an object as a dictionary with untrusted keys.** `{}` inherits `toString`, `constructor`, `__proto__`. Use `Object.create(null)` or, better, a `Map` (Chapter 15).

```js
const dict = Object.create(null);   // no prototype, no surprises
dict.toString;                       // undefined
```

**Best practice: prefer flat shapes.** Deeply nested objects make copying, diffing, and reasoning expensive.

**Best practice: use `Object.freeze` for module-level constants** so accidental mutation throws in strict mode.

---

## 5.8 Real-World Example: A Config Loader with Layered Defaults

```js
/**
 * Merges defaults, a config file, and environment overrides.
 * Deep-merges plain objects; arrays and primitives are replaced wholesale.
 */
function isPlainObject(v) {
  return Object.prototype.toString.call(v) === '[object Object]';
}

function deepMerge(base, override) {
  const out = { ...base };
  for (const [key, value] of Object.entries(override)) {
    // Reject prototype-pollution vectors (Chapter 31)
    if (key === '__proto__' || key === 'constructor') continue;
    out[key] = isPlainObject(value) && isPlainObject(base[key])
      ? deepMerge(base[key], value)
      : value;
  }
  return out;
}

const DEFAULTS = Object.freeze({
  server: { host: 'localhost', port: 3000, tls: false },
  logging: { level: 'info' },
});

export function loadConfig(fileConfig = {}, env = process.env) {
  let config = deepMerge(DEFAULTS, fileConfig);

  const envOverrides = {};
  if (env.PORT) envOverrides.server = { port: Number(env.PORT) };
  if (env.LOG_LEVEL) envOverrides.logging = { level: env.LOG_LEVEL };

  config = deepMerge(config, envOverrides);
  return deepFreeze(config);   // callers cannot corrupt shared config
}
```

The `__proto__` guard is not paranoia — it's the fix for a widely exploited class of vulnerability.

---

## 5.9 Questions

**1.** MCQ — Object keys can be: a) strings only b) strings and numbers c) strings and symbols d) anything
<details><summary>Answer</summary>**c)** Numbers are coerced to strings.</details>

**2.** MCQ — `Object.freeze` is: a) deep b) shallow c) recursive d) reversible
<details><summary>Answer</summary>**b)**</details>

**3.** MCQ — Which preserves `Date` objects when cloning? a) JSON round-trip b) spread c) `structuredClone` d) `Object.keys`
<details><summary>Answer</summary>**c)**</details>

**4.** MCQ — Properties added via `Object.defineProperty` default to: a) all true b) all false c) writable only d) enumerable only
<details><summary>Answer</summary>**b)**</details>

**5.** MCQ — `Object.hasOwn(o, k)` is preferred over `o.hasOwnProperty(k)` because: a) faster b) works on null-prototype objects c) newer d) deeper
<details><summary>Answer</summary>**b)**</details>

**6.** MCQ — Spread copies: a) all properties b) own enumerable properties c) inherited too d) descriptors
<details><summary>Answer</summary>**b)** And it flattens getters into values.</details>

**7.** Short — Why does `{a:1} === {a:1}` return false?
<details><summary>Answer</summary>Objects compare by reference; these are two distinct objects.</details>

**8.** Short — What order does `Object.keys` return?
<details><summary>Answer</summary>Integer-like keys in ascending numeric order first, then other string keys in insertion order.</details>

**9.** Short — When is `Object.create(null)` the right choice?
<details><summary>Answer</summary>When building a dictionary from untrusted keys — it has no prototype, so no inherited properties or `__proto__` hazard. A `Map` is usually better still.</details>

**10.** Short — Name two things `structuredClone` cannot clone.
<details><summary>Answer</summary>Functions and DOM nodes (also symbols and property descriptors).</details>

**11.** Short — Difference between `seal` and `freeze`?
<details><summary>Answer</summary>`seal` prevents adding/deleting properties but allows writing existing ones. `freeze` also prevents writing.</details>

**12.** Output — What prints?
```js
const a = { x: { y: 1 } };
const b = { ...a };
b.x.y = 2;
console.log(a.x.y);
```
<details><summary>Answer</summary>`2` — shallow copy shares the nested object.</details>

**13.** Output — What prints?
```js
const o = { b: 1, 2: 2, a: 3, 1: 4 };
console.log(Object.keys(o).join(','));
```
<details><summary>Answer</summary>`1,2,b,a`</details>

**14.** Output — What prints?
```js
const o = {};
Object.defineProperty(o, 'x', { value: 1 });
console.log(Object.keys(o), o.x, JSON.stringify(o));
```
<details><summary>Answer</summary>`[] 1 '{}'` — non-enumerable by default.</details>

**15.** Output — What prints?
```js
const o = Object.freeze({ a: 1 });
console.log(delete o.a, o.a);
```
<details><summary>Answer</summary>`false 1` in sloppy mode; throws `TypeError` in strict mode.</details>

**16.** Output — What prints?
```js
console.log(JSON.stringify({ a: undefined, b: () => {}, c: NaN, d: new Date(0) }));
```
<details><summary>Answer</summary>`'{"c":null,"d":"1970-01-01T00:00:00.000Z"}'` — `undefined` and functions are dropped, `NaN` becomes `null`, `Date` becomes an ISO string.</details>

**17.** Debug — A merge utility lets attackers set properties on every object in the app. What's missing?
<details><summary>Answer</summary>A guard rejecting `__proto__`, `constructor`, and `prototype` keys. This is prototype pollution (Chapter 31).</details>

**18.** Debug — `config.retries` is `undefined` after `Object.assign(config, userOptions)` where `userOptions` has a getter that throws. Why?
<details><summary>Answer</summary>`Object.assign` invokes getters during copying, so a throwing getter aborts the whole assignment partway through, leaving `config` half-merged.</details>

**19.** Debug — Two "identical" cache entries never hit. Key is an object.
<details><summary>Answer</summary>Object keys on a plain object stringify to `'[object Object]'` — so all objects collide into one key (or with a `Map`, distinct references never match). Serialize to a stable string key or key by a stable id.</details>

**20.** Code — Write `pick(obj, keys)` and `omit(obj, keys)`.
<details><summary>Answer</summary>

```js
const pick = (o, keys) =>
  Object.fromEntries(keys.filter(k => k in o).map(k => [k, o[k]]));

const omit = (o, keys) => {
  const drop = new Set(keys);
  return Object.fromEntries(Object.entries(o).filter(([k]) => !drop.has(k)));
};
```
</details>

**21.** Code — Write `deepEqual(a, b)` handling nested objects, arrays, and `NaN`.
<details><summary>Answer</summary>

```js
function deepEqual(a, b) {
  if (Object.is(a, b)) return true;
  if (typeof a !== 'object' || typeof b !== 'object' || !a || !b) return false;
  if (Array.isArray(a) !== Array.isArray(b)) return false;
  const ak = Object.keys(a), bk = Object.keys(b);
  if (ak.length !== bk.length) return false;
  return ak.every(k => Object.hasOwn(b, k) && deepEqual(a[k], b[k]));
}
```
</details>

**22.** Code — Write `flattenObject({a:{b:{c:1}}})` → `{'a.b.c': 1}`.
<details><summary>Answer</summary>

```js
function flattenObject(o, prefix = '', out = {}) {
  for (const [k, v] of Object.entries(o)) {
    const key = prefix ? `${prefix}.${k}` : k;
    if (v && typeof v === 'object' && !Array.isArray(v)) flattenObject(v, key, out);
    else out[key] = v;
  }
  return out;
}
```
</details>

**23.** Code — Create an object where `id` is readable but never writable or deletable.
<details><summary>Answer</summary>

```js
const rec = {};
Object.defineProperty(rec, 'id', {
  value: crypto.randomUUID(), writable: false,
  enumerable: true, configurable: false,
});
```
</details>

**24.** Scenario — Your Redux-style state updates are slow because you deep-clone the entire tree on every action. Better approach?
<details><summary>Answer</summary>Use structural sharing: copy only the path that changed and reuse every untouched branch by reference (`{...state, a: {...state.a, b: newB}}`). Or adopt Immer, which gives mutable-looking syntax with structural sharing underneath.</details>

**25.** Scenario — When would you reach for a `Proxy` in production code?
<details><summary>Answer</summary>Reactivity systems, API clients that generate methods on demand, validation layers, and dev-only strict-access guards. Avoid them on hot paths — every operation goes through a trap.</details>

---

## Chapter Summary

- Keys are strings or symbols; integer-like keys sort first.
- Descriptors control `writable`, `enumerable`, `configurable`; assignment gives `true`, `defineProperty` gives `false`.
- Spread and `Object.assign` are **shallow**. `structuredClone` is deep but rejects functions. JSON round-tripping is lossy.
- `Object.freeze` is shallow; deep-freeze or use structural sharing.
- Use `Object.hasOwn`, `Object.create(null)`, and prototype-pollution guards when handling untrusted keys.
- `Proxy` + `Reflect` intercept operations — powerful, but not free.

---

## Practice Exercises

1. Implement `deepFreeze` handling cycles.
2. Write `getIn(obj, 'a.b.0.c')` and `setIn(obj, path, value)` (immutably).
3. Build `diff(a, b)` returning added/removed/changed keys.
4. Write a `Proxy` giving arrays negative-index support (`arr[-1]`).
5. Implement `invert(obj)` swapping keys and values, handling duplicate values sensibly.

---

## Mini Project: An Observable Store

Build `store.js` exporting `createStore(initialState)`.

**Requirements:**

- `getState()` returns a deeply frozen snapshot.
- `setState(updaterFn)` receives a draft, applies changes immutably with structural sharing, and notifies subscribers.
- `subscribe(path, cb)` fires only when the value at that dot-path actually changes (compare with `Object.is`).
- Reject prototype-pollution keys in any path.
- Track a bounded `history` array supporting `undo()` and `redo()`.

**Stretch:** implement a `Proxy`-based draft so callers can write `draft.user.name = 'x'` and still get an immutable result.

---

**Next:** [Chapter 6 — Arrays →](06-arrays.md)
