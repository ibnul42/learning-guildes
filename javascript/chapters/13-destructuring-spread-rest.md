# Chapter 13 — Destructuring, Spread, Rest, and Modern Operators

## Learning Objectives

- Destructure objects, arrays, and nested structures with defaults and renaming
- Distinguish spread from rest (same `...`, opposite jobs)
- Use `?.`, `??`, and logical assignment operators correctly

---

## 13.1 Destructuring

### Objects

```js
const user = { name: 'Ada', age: 36, address: { city: 'London' } };

const { name, age } = user;
const { name: fullName } = user;                  // rename
const { role = 'member' } = user;                 // default (only for undefined)
const { address: { city } } = user;               // nested
const { name: n = 'anon', ...rest } = user;       // rename + default + rest

// Dynamic key
const key = 'age';
const { [key]: value } = user;                    // value === 36

// Assign to existing variables — parens required
let a, b;
({ a, b } = { a: 1, b: 2 });
```

### Arrays

```js
const [first, second] = [1, 2, 3];
const [, , third] = [1, 2, 3];          // skip with holes
const [head, ...tail] = [1, 2, 3];      // tail === [2, 3]
const [x = 10] = [];                     // default
let p = 1, q = 2;
[p, q] = [q, p];                         // swap, no temp variable
const [[a1], [b1]] = [[1], [2]];         // nested
```

Array destructuring works on **any iterable**:

```js
const [c1, c2] = 'hi';                   // 'h', 'i'
const [k, v] = new Map([['a', 1]]).entries().next().value;
const [firstEl] = document.querySelectorAll('li');
```

### In parameters

```js
function draw({ x = 0, y = 0, color = 'black' } = {}) { /* ... */ }
draw();                       // all defaults — note the `= {}`
draw({ x: 10 });

// Destructure in callbacks
users.map(({ id, name }) => ({ id, label: name }));
Object.entries(obj).forEach(([key, value]) => {});
```

> ⚠️ Without `= {}`, calling `draw()` throws — you can't destructure `undefined`.

**Deep nesting is a trap:**

```js
const { a: { b: { c } } } = obj;   // throws if a or b is missing
const c = obj?.a?.b?.c;             // safer
```

---

## 13.2 Spread vs Rest

Same `...` token, opposite direction:

- **Spread** — *expands* an iterable/object into individual elements. Used in calls, array literals, object literals.
- **Rest** — *collects* remaining items into an array/object. Used in parameter lists and destructuring patterns.

```js
// SPREAD
const combined = [...a, ...b];
const copy = { ...obj, override: 1 };      // later keys win
Math.max(...nums);
const chars = [...'abc'];
new Set([...a, ...b]);

// REST
function f(first, ...others) {}            // others is a real Array
const [head, ...tail] = arr;
const { id, ...withoutId } = obj;
```

Rest must be **last** and there can be only one.

**Object spread nuances:**

```js
{ ...null }                    // {} — nullish sources are ignored, no error
{ ...'ab' }                    // { 0: 'a', 1: 'b' }
{ ...[1,2] }                   // { 0: 1, 1: 2 }
{ ...obj }                     // own ENUMERABLE props only; getters are invoked
                               // prototype and non-enumerables are NOT copied
```

Order matters — spread later to override, earlier to provide defaults:

```js
const config = { ...DEFAULTS, ...userConfig };     // user wins
const forced = { ...userConfig, apiKey: SECRET };  // we win
```

---

## 13.3 Optional Chaining `?.`

```js
user?.profile?.email          // undefined instead of throwing
user?.getName?.()             // call only if it's a function
arr?.[0]
config?.handlers?.[eventName]?.(payload)
```

Short-circuits: if the left side is `null` or `undefined`, the **entire chain** stops and yields `undefined` — subsequent calls and index lookups aren't evaluated.

```js
let i = 0;
null?.[i++];
console.log(i);   // 0 — the index expression never ran
```

> ⚠️ `?.` only guards `null`/`undefined`. `''?.length` is `0`, not `undefined`. And it does **not** make assignment safe: `a?.b = 1` is a SyntaxError.

**Don't over-apply it.** `a?.b?.c?.d?.e` often hides a bug — if `a.b` should always exist, a missing value should throw loudly rather than silently produce `undefined` three layers downstream.

---

## 13.4 Nullish Coalescing and Logical Assignment

```js
value ?? fallback              // fallback only when value is null/undefined
value || fallback              // fallback for ANY falsy value

0 ?? 5      // 0
0 || 5      // 5
'' ?? 'x'   // ''
'' || 'x'   // 'x'

// Logical assignment (ES2021)
obj.a ??= 1;    // assign only if null/undefined  → obj.a ?? (obj.a = 1)
obj.b ||= 1;    // assign if falsy
obj.c &&= 1;    // assign if truthy

// All three SHORT-CIRCUIT — the assignment doesn't happen at all if skipped.
// This matters with setters and Proxies:
const o = { set x(v) { console.log('setter ran'); } , get x() { return 1; } };
o.x ??= 2;      // getter returns 1, so no assignment — setter never runs
```

Idiomatic uses:

```js
(map[key] ??= []).push(item);          // get-or-create in one expression
options.retries ??= 3;                  // fill missing config
cache.value ??= expensiveCompute();     // lazy init, computed only once
```

---

## 13.5 Common Mistakes and Best Practices

**Mistake: `||` where `??` is meant.** Zero, empty string, and `false` are valid values.

**Mistake: forgetting `= {}` on a destructured parameter.**

**Mistake: assuming spread is deep.** It's one level. Nested objects stay shared.

**Mistake: spreading a huge array into a call.** `Math.max(...millionItems)` overflows the argument stack. Use `reduce`.

**Mistake: destructuring in hot loops.** Minor, but it does allocate; irrelevant except in measured hot paths.

**Best practice: destructure in the signature** to document what a function actually uses.

**Best practice: limit destructuring depth to two levels.** Beyond that, optional chaining or an explicit guard reads better.

---

## 13.6 Real-World Example: A Resilient API Response Handler

```js
/**
 * Real API responses are inconsistent. Destructuring + ?? + ?. handles
 * missing fields declaratively instead of with a wall of if-statements.
 */
function parseSearchResponse(response, { pageSize = 20 } = {}) {
  const {
    data: {
      items = [],
      pagination: { page = 1, total = 0, cursor = null } = {},
    } = {},
    meta: { requestId, tookMs } = {},
    errors = [],
  } = response ?? {};

  if (errors.length > 0) {
    const [{ code = 'UNKNOWN', message = 'Request failed' } = {}] = errors;
    throw new Error(`[${code}] ${message}`, { cause: { requestId, errors } });
  }

  return {
    items: items.map(({ id, attributes: { title, author } = {}, ...extra }) => ({
      id,
      title: title?.trim() || 'Untitled',        // || is right: empty title → fallback
      author: author?.name ?? 'Unknown',          // ?? is right: preserve '' if meaningful
      extra,
    })),
    page,
    totalPages: Math.max(1, Math.ceil(total / pageSize)),
    hasMore: cursor != null,
    diagnostics: { requestId, tookMs },
  };
}
```

Note the deliberate mix: `||` for `title` (an empty title should be replaced), `??` for `author` (an empty author name is still a value we'd keep). That choice is the whole point of having both operators.

---

## 13.7 Questions

**1.** MCQ — `const {a = 1} = {a: null}` gives `a` =: a) `1` b) `null` c) `undefined` d) error
<details><summary>Answer</summary>**b)** Defaults only fire for `undefined`.</details>

**2.** MCQ — Rest parameters must be: a) first b) last c) anywhere d) typed
<details><summary>Answer</summary>**b)**</details>

**3.** MCQ — `{...null}` produces: a) TypeError b) `null` c) `{}` d) `undefined`
<details><summary>Answer</summary>**c)**</details>

**4.** MCQ — `0 ?? 5` is: a) `0` b) `5` c) `null` d) `NaN`
<details><summary>Answer</summary>**a)**</details>

**5.** MCQ — `a?.b = 1` is: a) valid b) SyntaxError c) undefined d) TypeError
<details><summary>Answer</summary>**b)**</details>

**6.** MCQ — Object spread copies: a) everything b) own enumerable props c) the prototype too d) descriptors
<details><summary>Answer</summary>**b)**</details>

**7.** Short — Spread vs rest?
<details><summary>Answer</summary>Same syntax, opposite jobs: spread expands a collection into elements; rest collects remaining elements into a collection.</details>

**8.** Short — Why does `function f({a}) {}` throw on `f()`?
<details><summary>Answer</summary>You can't destructure `undefined`. Add `= {}`.</details>

**9.** Short — When is `||` correct and `??` wrong?
<details><summary>Answer</summary>When *any* falsy value should trigger the fallback — e.g. replacing an empty string title with 'Untitled'.</details>

**10.** Short — Does `obj.x ??= compute()` always call `compute()`?
<details><summary>Answer</summary>No. It short-circuits — `compute()` runs only when `obj.x` is nullish.</details>

**11.** Short — How do you swap two variables without a temp?
<details><summary>Answer</summary>`[a, b] = [b, a]`</details>

**12.** Output — `const [a=1,b=2] = [undefined, null]; console.log(a, b);`
<details><summary>Answer</summary>`1 null`</details>

**13.** Output — `const {length} = 'hello'; console.log(length);`
<details><summary>Answer</summary>`5` — primitives are boxed for destructuring.</details>

**14.** Output — `console.log({...{a:1}, ...{a:2}});`
<details><summary>Answer</summary>`{a: 2}` — later wins.</details>

**15.** Output — What prints?
```js
const o = {a: {b: 1}};
const c = {...o};
c.a.b = 2;
console.log(o.a.b);
```
<details><summary>Answer</summary>`2` — shallow.</details>

**16.** Output — `let i=0; const o=null; o?.[i++]; console.log(i);`
<details><summary>Answer</summary>`0` — full short-circuit.</details>

**17.** Output — `const {a, ...r} = {a:1,b:2,c:3}; console.log(r);`
<details><summary>Answer</summary>`{b:2, c:3}`</details>

**18.** Output — `console.log([...'héllo'].length, 'héllo'.length);`
<details><summary>Answer</summary>`5 5` — but with `'😀'` they'd differ (1 vs 2).</details>

**19.** Debug — A feature flag defaulting to `false` is always `true`:
```js
const enabled = flags.newUI || true;
```
<details><summary>Answer</summary>`|| true` makes it unconditionally true. The author meant `?? true` (or, given the intent, `?? false`). Also, `||` would override a deliberate `false`.</details>

**20.** Debug — Config merging drops nested defaults:
```js
const config = {...DEFAULTS, ...user};
```
<details><summary>Answer</summary>Spread is shallow — `user.server = {port: 1}` replaces the entire `DEFAULTS.server` object, losing `host`. Use a deep merge (Chapter 5).</details>

**21.** Debug — `Math.max(...bigArray)` throws "Maximum call stack size exceeded."
<details><summary>Answer</summary>Spread becomes individual arguments; engines cap argument count around 65k–125k. Use `bigArray.reduce((m, n) => n > m ? n : m, -Infinity)`.</details>

**22.** Code — Write `pick(obj, keys)` using only destructuring-adjacent tools.
<details><summary>Answer</summary>

```js
const pick = (obj, keys) =>
  keys.reduce((out, k) => (k in obj ? { ...out, [k]: obj[k] } : out), {});
```
(For large key lists, mutate the accumulator rather than spreading each iteration.)
</details>

**23.** Code — Write `defaults(target, ...sources)` filling only nullish values.
<details><summary>Answer</summary>

```js
function defaults(target, ...sources) {
  const out = { ...target };
  for (const src of sources)
    for (const [k, v] of Object.entries(src)) out[k] ??= v;
  return out;
}
```
</details>

**24.** Code — Destructure this in one statement, extracting `city`, the first tag, and everything else:
```js
const data = { id: 1, address: { city: 'Dhaka' }, tags: ['a','b'], extra: true };
```
<details><summary>Answer</summary>

```js
const { address: { city } = {}, tags: [firstTag] = [], ...rest } = data;
// city='Dhaka', firstTag='a', rest={id:1, extra:true}
```
</details>

**25.** Scenario — A reviewer flags `a?.b?.c?.d` as a code smell. Are they right?
<details><summary>Answer</summary>Often yes. Long optional chains usually mean the data contract is unclear. Validate the shape once at the boundary, then access confidently. Keep `?.` for genuinely optional fields.</details>

**26.** Scenario — Your team debates `??=` versus an explicit `if`. Which and why?
<details><summary>Answer</summary>`??=` for simple nullish defaults — it's shorter and short-circuits correctly. Use an explicit `if` when the assignment has side effects worth making visible, or when you need to log/branch.</details>

---

## Chapter Summary

- Destructuring works on objects, arrays, strings, and any iterable, with renaming, defaults, nesting, and rest.
- Defaults fire only for `undefined`, never `null`.
- Spread expands, rest collects — same token, opposite direction. Both are **shallow**.
- Object spread copies own enumerable properties only, invoking getters and dropping the prototype.
- `?.` short-circuits the whole chain; it guards nullish only and can't be assigned to.
- `??` preserves `0`, `''`, and `false`; `||` doesn't. Choose deliberately.
- `??=`, `||=`, `&&=` short-circuit, so the right side may never evaluate.

---

## Practice Exercises

1. Write one destructuring statement extracting six values (including nested, renamed, defaulted, and rest) from a complex object.
2. Implement `omitDeep(obj, keys)` removing keys at any depth, immutably.
3. Rewrite five `||` defaults in your codebase, deciding `??` vs `||` for each and justifying it.
4. Implement `zipObject(keys, values)` using spread and destructuring.
5. Write a function accepting either `(x, y)` or `({x, y})` and normalizing internally.

---

**Next:** [Chapter 14 — Symbols, Iterators, and Generators →](14-symbols-iterators-generators.md)
