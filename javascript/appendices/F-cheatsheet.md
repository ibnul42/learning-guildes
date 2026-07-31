# Appendix F — Cheat Sheet

Dense and scannable. Tables over prose. Meant to be kept open in a tab.

**Legend:** ⚠️ mutates · ✂️ short-circuits / early exit · 🕳️ skips holes in sparse arrays

---

## Contents

[Syntax Skeletons](#syntax-skeletons) · [Truthy and Falsy](#truthy-and-falsy) ·
[Operators](#operators) · [Array Methods](#array-methods) · [String Methods](#string-methods) ·
[Object Statics](#object-statics) · [Map, Set, Collections](#map-set-and-collections) ·
[Promise Statics](#promise-statics) · [Destructuring](#destructuring-forms) ·
[Regex](#regex) · [`this` Rules](#this-rules) · [Classes](#classes) ·
[One-Liners](#useful-one-liners) · [DOM Reference](#dom-quick-reference) ·
[Console](#console-beyond-log) · [Browser APIs](#useful-browser-apis) ·
[Intl and Temporal](#intl-and-temporal)

---

## Syntax Skeletons

```js
// Functions
function name(a, b = 1, ...rest) {}          // hoisted, has `this`, `arguments`, `prototype`
const fn = function () {};                    // not hoisted
const arrow = (a) => a * 2;                   // lexical `this`, no arguments/prototype
const obj = { method() {}, get x() {}, set x(v) {}, async *gen() {} };
(function () {})();                            // IIFE
async function f() { await p; }
function* g() { yield 1; yield* other(); }
async function* ag() { yield await p; }

// Classes
class A extends B {
  static VERSION = 1;
  static { /* static init block */ }
  #priv = 0;                                   // private field
  pub = 1;                                     // public field
  constructor() { super(); }
  #method() {}                                 // private method
  static isA(o) { return #priv in o; }          // brand check
  get value() { return this.#priv; }
  [Symbol.iterator]() {}
}

// Control flow
if (a) {} else if (b) {} else {}
switch (x) { case 1: break; default: }
for (let i = 0; i < n; i++) {}
for (const v of iterable) {}                   // values
for (const k in object) {}                     // enumerable keys, incl. inherited
for await (const v of asyncIterable) {}
while (c) {} / do {} while (c);
outer: for (…) { break outer; continue outer; }
try {} catch (e) {} finally {}                 // catch binding is optional: catch {}

// Modules
import def, { named, other as alias } from './m.js';
import * as ns from './m.js';
import data from './d.json' with { type: 'json' };
const m = await import('./m.js');              // dynamic
export default x; export { a, b }; export * from './m.js';
export const c = 1;
```

## Truthy and Falsy

**Exactly eight falsy values.** Everything else is truthy.

```js
false, 0, -0, 0n, '', null, undefined, NaN
```

```js
// Truthy surprises
Boolean('0');        // true      Boolean('false');   // true
Boolean([]);         // true      Boolean({});        // true
Boolean(' ');        // true      Boolean(-1);        // true
Boolean(new Boolean(false));      // true — it's an object
```

| | `Object.is` | `===` | `==` | SameValueZero |
|---|---|---|---|---|
| `NaN, NaN` | `true` | `false` | `false` | `true` |
| `0, -0` | `false` | `true` | `true` | `true` |
| `null, undefined` | `false` | `false` | `true` | `false` |
| Coerces types | ❌ | ❌ | ✅ | ❌ |
| Used by | — | — | — | `includes`, `Set`, `Map` |

## Operators

```js
// Precedence, high → low (essentials only)
() . [] ?.          // access, call
new                 // construction
++ -- ! ~ typeof void delete +x -x await
**                  // right-associative: 2 ** 3 ** 2 === 512
* / %
+ -
<< >> >>>
< <= > >= in instanceof
== != === !==
& ^ |
&&
||  ??               // cannot be mixed without parentheses
?:                   // right-associative
= += -= &&= ||= ??=
,                    // lowest
```

```js
a?.b        a?.[k]      a?.()          // optional chaining — short-circuits on null/undefined
a ?? b                                  // only null/undefined trigger the fallback
a ||= b     a &&= b     a ??= b         // logical assignment
a ** b                                  // exponentiation
1_000_000                               // numeric separator
x = cond ? a : b
[a, b] = [b, a]                         // swap
label in obj                            // property existence, incl. inherited
#priv in obj                            // private-field brand check
void 0                                  // undefined
~~x                                     // truncate (⚠️ 32-bit only)
+x  `${x}`  !!x                         // to number / string / boolean
```

## Array Methods

| Method | Mutates | Returns | Notes |
|---|---|---|---|
| `push(...v)` | ⚠️ | new length | |
| `pop()` | ⚠️ | removed element | |
| `shift()` | ⚠️ | removed element | O(n) — reindexes |
| `unshift(...v)` | ⚠️ | new length | O(n) |
| `splice(i, n, ...v)` | ⚠️ | removed array | |
| `sort(cmp?)` | ⚠️ | same array | Default is **lexicographic**. Stable. |
| `reverse()` | ⚠️ | same array | |
| `fill(v, s?, e?)` | ⚠️ | same array | Fills holes |
| `copyWithin(t, s?, e?)` | ⚠️ | same array | |
| `toSorted(cmp?)` | — | new array | ES2023 |
| `toReversed()` | — | new array | ES2023 |
| `toSpliced(i, n, ...v)` | — | new array | ES2023 |
| `with(i, v)` | — | new array | ES2023, supports negative index |
| `slice(s?, e?)` | — | new array | Shallow copy |
| `concat(...v)` | — | new array | |
| `flat(d = 1)` | — | new array | Removes holes |
| `flatMap(fn)` | — | new array | Map then flatten 1 level. `[]` drops. |
| `map(fn)` 🕳️ | — | new array | Preserves holes |
| `filter(fn)` 🕳️ | — | new array | Compacts holes |
| `forEach(fn)` 🕳️ | — | `undefined` | Can't break |
| `reduce(fn, init?)` 🕳️ | — | any | Throws on empty with no init |
| `reduceRight(fn, init?)` 🕳️ | — | any | |
| `find(fn)` ✂️ | — | element or `undefined` | |
| `findIndex(fn)` ✂️ | — | index or `-1` | |
| `findLast(fn)` ✂️ | — | element or `undefined` | ES2023 |
| `findLastIndex(fn)` ✂️ | — | index or `-1` | ES2023 |
| `some(fn)` ✂️ | — | boolean | `[]` → `false` |
| `every(fn)` ✂️ | — | boolean | `[]` → `true` |
| `includes(v, from?)` | — | boolean | SameValueZero — finds `NaN` |
| `indexOf(v, from?)` | — | index or `-1` | Strict equality — misses `NaN` |
| `lastIndexOf(v)` | — | index or `-1` | |
| `at(i)` | — | element or `undefined` | Negative index allowed |
| `join(sep = ',')` | — | string | `null`/`undefined` → `''` |
| `keys()` `values()` `entries()` | — | iterator | |
| **Statics** | | | |
| `Array.isArray(v)` | — | boolean | Realm-safe |
| `Array.from(x, fn?)` | — | new array | Accepts array-likes and iterables |
| `Array.of(...v)` | — | new array | `Array.of(3)` → `[3]` |

```js
Array(3);                          // [<3 holes>] — length 3, no elements
Array(3).fill(0);                  // [0,0,0]     ✅ fill writes holes
Array(3).map(() => 0);             // [<3 holes>] ❌ map skips them
[...Array(3)];                     // [undefined × 3]
Array.from({ length: 3 }, (_, i) => i);   // [0,1,2]  ✅ the idiom
```

## String Methods

| Method | Returns | Notes |
|---|---|---|
| `at(i)` | char or `undefined` | Negative index |
| `charAt(i)` / `[i]` | char / `undefined` | |
| `codePointAt(i)` | number | Handles surrogate pairs |
| `charCodeAt(i)` | number | UTF-16 code unit |
| `includes(s, from?)` | boolean | |
| `startsWith` / `endsWith(s, pos?)` | boolean | |
| `indexOf` / `lastIndexOf(s)` | index or `-1` | |
| `slice(s?, e?)` | string | Negative indices allowed |
| `substring(s?, e?)` | string | Swaps args if reversed; no negatives |
| `split(sep, limit?)` | array | Regex separator includes capture groups |
| `concat(...s)` | string | Use `+` or a template |
| `repeat(n)` | string | Throws on negative |
| `padStart` / `padEnd(len, pad?)` | string | |
| `trim` / `trimStart` / `trimEnd()` | string | |
| `toUpperCase` / `toLowerCase()` | string | Not locale-aware |
| `toLocaleUpperCase(loc?)` | string | Use for Turkish `i`, etc. |
| `localeCompare(s, loc?, opt?)` | `-1 \| 0 \| 1` | **Use for sorting** |
| `normalize(form?)` | string | `'NFC'`, `'NFD'`, `'NFKC'`, `'NFKD'` |
| `match(re)` | match / array / `null` | Two shapes: with `/g` loses groups |
| `matchAll(re)` | iterator | **Requires `/g`** |
| `replace(x, y)` | string | Replaces first, unless `/g` |
| `replaceAll(x, y)` | string | Regex arg **must** have `/g` |
| `search(re)` | index or `-1` | |
| `isWellFormed` / `toWellFormed()` | boolean / string | ES2024, lone surrogates |
| `String.raw\`…\`` | string | No escape processing |

```js
// Replacement patterns: $& whole, $1 group, $<name> named, $` before, $' after, $$ literal $
'a-b'.replace(/-(\w)/g, (_, c) => c.toUpperCase());     // 'aB'
[...'👍'].length;   // 1        '👍'.length;   // 2  (UTF-16 code units)
```

## Object Statics

```js
Object.keys(o) / values(o) / entries(o)      // own enumerable STRING keys
Object.fromEntries(pairs)                     // inverse of entries
Object.assign(target, ...src)                 // ⚠️ mutates target, invokes its setters
Object.freeze(o) / isFrozen(o)                // shallow!
Object.seal(o) / isSealed(o)                  // no add/remove, existing stay writable
Object.preventExtensions(o) / isExtensible(o)
Object.create(proto, descriptors?)
Object.getPrototypeOf(o) / setPrototypeOf(o, p)   // setPrototypeOf deoptimises — avoid
Object.defineProperty(o, k, desc)             // defaults: all FALSE
Object.defineProperties(o, descs)
Object.getOwnPropertyNames(o)                 // incl. non-enumerable
Object.getOwnPropertySymbols(o)
Object.getOwnPropertyDescriptor(s)(o, k?)
Object.hasOwn(o, k)                           // ES2022 — prefer over hasOwnProperty
Object.is(a, b)                               // SameValue
Object.groupBy(items, fn)                     // ES2024 → null-prototype object
Reflect.ownKeys(o)                            // strings + symbols, everything own
structuredClone(o)                            // deep, handles cycles/Map/Set/Date
```

| Descriptor | Assignment default | `defineProperty` default |
|---|---|---|
| `writable` | `true` | `false` |
| `enumerable` | `true` | `false` |
| `configurable` | `true` | `false` |

**Key enumeration order:** integer-like keys ascending → string keys in insertion order →
symbol keys in insertion order.

## Map, Set, and Collections

| | `Map` | `WeakMap` | `Set` | `WeakSet` |
|---|---|---|---|---|
| Key types | any | object/symbol | any | object |
| Weakly held | ❌ | ✅ | ❌ | ✅ |
| Iterable | ✅ | ❌ | ✅ | ❌ |
| `.size` | ✅ | ❌ | ✅ | ❌ |

```js
const m = new Map([[k, v]]);
m.set(k, v); m.get(k); m.has(k); m.delete(k); m.clear(); m.size;
m.keys(); m.values(); m.entries(); [...m];    // [[k, v], …]
Map.groupBy(items, fn);                        // ES2024

const s = new Set([1, 2]);
s.add(v); s.has(v); s.delete(v); s.clear(); s.size; [...s];

// ES2025 set operations — all return NEW sets
a.union(b) / intersection(b) / difference(b) / symmetricDifference(b)
a.isSubsetOf(b) / isSupersetOf(b) / isDisjointFrom(b)

new WeakRef(obj).deref();                      // object or undefined
new FinalizationRegistry(cb).register(obj, held, token);   // ⚠️ may never run
```

```js
// Iterator helpers (ES2025) — lazy, work on any iterator
iter.map(fn).filter(fn).take(n).drop(n).flatMap(fn).toArray()
iter.reduce(fn, init) / some(fn) / every(fn) / find(fn) / forEach(fn)
```

## Promise Statics

| Method | Resolves when | Rejects when | Result |
|---|---|---|---|
| `all(it)` | **all** fulfil | **first** rejection | array, input order |
| `allSettled(it)` | all settle | never | `[{status, value\|reason}]` |
| `race(it)` | first **settles** | first settles as rejected | that value/reason |
| `any(it)` | first **fulfils** | **all** reject | value / `AggregateError` |

```js
Promise.resolve(v) / reject(r)
Promise.withResolvers()          // ES2024 → { promise, resolve, reject }
Promise.try(fn)                  // ES2025 — sync-or-async into a chain
p.then(onOk, onErr) / catch(onErr) / finally(fn)    // finally passes the value through
```

```js
// AbortController
const ac = new AbortController();
fetch(url, { signal: ac.signal });
ac.abort(reason);                          // → DOMException 'AbortError'
AbortSignal.timeout(5000);                 // → 'TimeoutError'
AbortSignal.any([a, b]);                   // first to abort wins
AbortSignal.abort(reason);                 // already-aborted signal
```

**Event loop order:** sync → **all** microtasks (promises, `queueMicrotask`, `MutationObserver`) →
render (rAF, then style/layout/paint) → **one** macrotask (timers, I/O, events) → repeat.
Node inserts `process.nextTick` *before* promise microtasks.

## Destructuring Forms

```js
const [a, b = 2, , ...rest] = arr;               // skip with a bare comma
const { x, y: renamed, z = 3, ...others } = obj;
const { a: { b: deep } } = obj;                   // nested
const [{ id }] = arrayOfObjects;
const { length } = 'hello';                        // 5
({ a, b } = obj);                                  // parens required for assignment
function f({ a = 1, b } = {}) {}                   // = {} makes the argument optional
for (const [k, v] of Object.entries(o)) {}
const [, second] = arr;
const { 0: first, length } = arr;                  // arrays are objects

// Spread
[...arr, x]        { ...obj, k: v }        f(...args)
new Set([...a, ...b])
{ ...obj, ...(cond && { extra: 1 }) }              // conditional property
```

## Regex

**Flags**

| | Name | Effect |
|---|---|---|
| `g` | global | All matches; enables `lastIndex` |
| `i` | ignoreCase | |
| `m` | multiline | `^`/`$` match at line breaks |
| `s` | dotAll | `.` matches `\n` |
| `u` | unicode | Code points; enables `\p{…}` |
| `v` | unicodeSets | `u` + set operations in classes |
| `y` | sticky | Match only at `lastIndex` |
| `d` | hasIndices | Populates `match.indices` |

```js
// Classes                          Quantifiers                Anchors
[abc] [^abc] [a-z]                  * + ? {n} {n,} {n,m}       ^ $ \b \B
\d \D \w \W \s \S .                 *? +? ??  (lazy)
\p{L} \p{N} \p{Lu} \p{Script=Greek}                            // needs /u

// Groups                           Lookaround
(x)          capture                (?=x)   positive lookahead
(?:x)        non-capturing          (?!x)   negative lookahead
(?<n>x)      named                  (?<=x)  positive lookbehind
\1  \k<n>    backreference          (?<!x)  negative lookbehind
x|y          alternation
(?i:x)       scoped flag (ES2025)

RegExp.escape(str)                  // ES2025 — safe interpolation
```

```js
re.test(s)                       // ⚠️ /g makes it stateful via lastIndex
re.exec(s)                       // match array or null; loop with /g
s.match(re)                      // /g → strings only, groups LOST
[...s.matchAll(re)]              // ✅ full match objects; requires /g
s.replace(re, (m, p1, …, offset, str, groups) => …)

// Thousands separators
'1234567.89'.replace(/\B(?=(\d{3})+(?!\d))/g, ',');   // '1,234,567.89'
```

⚠️ **ReDoS:** nested quantifiers (`(a+)+`, `(\w*)*`) backtrack exponentially. Use disjoint classes:
`"([^"\\]|\\.)*"` is linear.

## `this` Rules

Highest priority first:

| # | Rule | `this` is | Example |
|---|---|---|---|
| 1 | `new` | the new instance | `new F()` |
| 2 | Explicit | the first argument | `f.call(o)`, `f.apply(o)`, `f.bind(o)` |
| 3 | Implicit | the object before the dot | `o.f()` |
| 4 | Default | `undefined` (strict) / `globalThis` | `f()` |
| — | **Arrow** | inherited lexically — **overrides all above** | `() => this` |

```js
const { method } = obj;  method();        // ❌ detached: rule 4
obj.method();                              // ✅ rule 3
setTimeout(obj.method, 0);                 // ❌ detached
setTimeout(() => obj.method(), 0);         // ✅
class A { handler = () => this.x; }        // ✅ bound per instance
f.bind(a).bind(b)();                        // `a` wins — can't rebind
```

Module top level: `this` is `undefined`. Classic script top level: `globalThis`.

## Classes

```js
class Base {
  static registry = new Map();
  static { Base.registry.set('base', Base); }     // static init block
  #secret;                                          // private, enforced
  count = 0;                                        // public field
  constructor(x) { this.#secret = x; }
  #hash() { return this.#secret; }                  // private method
  get value() { return this.#secret; }
  set value(v) { this.#secret = v; }
  static create(x) { return new this(x); }          // `this` is the subclass
  static has(o) { return #secret in o; }             // brand check
}
class Sub extends Base {
  constructor(x) { super(x); }                       // MUST precede `this`
  method() { return super.value; }
}
const Mixin = (B) => class extends B { extra() {} };
class C extends Mixin(Base) {}
```

Class bodies are **always strict**. Methods are non-enumerable. `new` is required.

## Useful One-Liners

```js
// Dedupe / group / chunk
const unique = [...new Set(arr)];
const uniqueBy = (a, f) => [...new Map(a.map(x => [f(x), x])).values()];
const grouped = Object.groupBy(arr, x => x.type);
const chunk = (a, n) => Array.from({ length: Math.ceil(a.length / n) },
                                   (_, i) => a.slice(i * n, i * n + n));

// Sets
const inter = (a, b) => { const s = new Set(b); return a.filter(x => s.has(x)); };
const diff  = (a, b) => { const s = new Set(b); return a.filter(x => !s.has(x)); };

// Numbers
const clamp = (n, lo, hi) => Math.min(hi, Math.max(lo, n));
const sum   = (a) => a.reduce((s, x) => s + x, 0);
const round = (n, d = 2) => Math.round(n * 10 ** d) / 10 ** d;
const randomInt = (lo, hi) => lo + Math.floor(Math.random() * (hi - lo + 1));

// Objects
const deepGet = (o, p) => p.split('.').reduce((x, k) => x?.[k], o);
const pick = (o, ks) => Object.fromEntries(ks.filter(k => k in o).map(k => [k, o[k]]));
const omit = (o, ks) => Object.fromEntries(Object.entries(o).filter(([k]) => !ks.includes(k)));
const invert = (o) => Object.fromEntries(Object.entries(o).map(([k, v]) => [v, k]));
const isEmpty = (o) => Object.keys(o).length === 0;

// Async
const sleep = (ms) => new Promise(r => setTimeout(r, ms));
const timeout = (p, ms) => Promise.race([p, sleep(ms).then(() => { throw new Error('timeout'); })]);

// Strings
const capitalise = (s) => s.charAt(0).toUpperCase() + s.slice(1);
const slug = (s) => s.toLowerCase().normalize('NFKD')
  .replace(/[̀-ͯ]/g, '').replace(/[^a-z0-9]+/g, '-').replace(/^-|-$/g, '');
const escapeHtml = (s) => s.replace(/[&<>"']/g, c =>
  ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]));

// Misc
const pipe = (...f) => (x) => f.reduce((a, fn) => fn(a), x);
const range = (a, b, s = 1) => Array.from({ length: Math.ceil((b - a) / s) }, (_, i) => a + i * s);
const shuffle = (a) => { const o = a.slice();
  for (let i = o.length - 1; i > 0; i--) { const j = Math.floor(Math.random() * (i + 1));
    [o[i], o[j]] = [o[j], o[i]]; } return o; };
```

## DOM Quick Reference

```js
// Selection
document.querySelector(sel)              // first or null
document.querySelectorAll(sel)           // STATIC NodeList
document.getElementById(id)              // fastest
el.closest(sel)                          // nearest ancestor-or-self
el.matches(sel)                          // boolean
el.querySelectorAll(':scope > li')       // direct children
document.getElementsByClassName(c)       // LIVE HTMLCollection

// Traversal
el.parentElement / children / firstElementChild / lastElementChild
el.nextElementSibling / previousElementSibling

// Creating and inserting
document.createElement(tag)
document.createDocumentFragment()
el.append(...nodes)        el.prepend(...nodes)      // strings → text nodes
el.before(...) / after(...) / replaceWith(...)
el.replaceChildren(...nodes)                          // clears then appends
el.insertAdjacentElement('beforebegin'|'afterbegin'|'beforeend'|'afterend', node)
el.remove()   el.cloneNode(deep)

// Content
el.textContent = s        // ✅ safe, fast, includes hidden
el.innerText              // layout-aware, forces reflow
el.innerHTML = s          // ⚠️ XSS sink
el.setHTML(s, { sanitizer: new Sanitizer() })          // ✅ where supported

// Attributes, classes, styles
el.getAttribute / setAttribute / removeAttribute / hasAttribute / toggleAttribute
el.dataset.foo                            // data-foo
el.classList.add / remove / toggle(n, force?) / contains / replace
el.style.setProperty('--x', v)  getComputedStyle(el).getPropertyValue('--x')

// Geometry — ⚠️ all of these force layout
el.getBoundingClientRect()   el.offsetWidth/Height/Top/Left
el.clientWidth/Height        el.scrollWidth/Height/Top/Left
el.scrollIntoView({ behavior, block, inline })

// Events
el.addEventListener(type, fn, { capture, once, passive, signal })
el.removeEventListener(type, fn, { capture })
el.dispatchEvent(new CustomEvent('x', { detail, bubbles: true, cancelable: true }))
e.target / currentTarget / composedPath()
e.preventDefault() / stopPropagation() / stopImmediatePropagation()
```

**Don't bubble:** `focus`, `blur`, `load`, `mouseenter`, `mouseleave`.
Bubbling equivalents: `focusin`, `focusout`, `mouseover`, `mouseout`.

**Force layout on read:** `offset*`, `scroll*`, `client*`, `getBoundingClientRect`,
`getComputedStyle`, `focus`, `scrollIntoView`. Batch all reads before all writes.

## Console Beyond `log`

```js
console.table(arrayOfObjects, ['col1', 'col2'])    // genuinely useful
console.group(label) / groupCollapsed / groupEnd
console.time(label) / timeLog / timeEnd
console.count(label) / countReset
console.assert(cond, msg)                           // logs only when falsy
console.trace()                                     // stack trace
console.dir(obj, { depth: null })                   // object, not DOM rendering
console.warn / error / info / debug
console.profile(label) / profileEnd                 // DevTools CPU profile
copy(obj)                                            // DevTools only — to clipboard
$0 $1 …                                              // DevTools — recent selections
$(sel) $$(sel)                                       // DevTools — querySelector(All)
monitorEvents(el, 'click')                           // DevTools only
```

## Useful Browser APIs

```js
// Observers
new IntersectionObserver(cb, { root, rootMargin, threshold }).observe(el)
new ResizeObserver(cb).observe(el, { box: 'content-box' })
new MutationObserver(cb).observe(el, { childList, subtree, attributes, characterData })
new PerformanceObserver(cb).observe({ type: 'longtask', buffered: true })

// Scheduling
setTimeout / setInterval / clearTimeout / clearInterval
requestAnimationFrame(cb) / cancelAnimationFrame(id)
requestIdleCallback(cb, { timeout }) / cancelIdleCallback
queueMicrotask(cb)
scheduler.postTask(fn, { priority: 'user-blocking'|'user-visible'|'background' })
await scheduler.yield()

// Storage
localStorage / sessionStorage          // sync, string-only, ~5MB, can throw
indexedDB.open(name, version)          // async, large, structured clone
caches.open(name)                      // Request/Response pairs
await navigator.storage.estimate()     // { usage, quota }
await navigator.storage.persist()
await cookieStore.set({ name, value, sameSite })

// Cross-context
new Worker(new URL('./w.js', import.meta.url), { type: 'module' })
worker.postMessage(data, [transferList])
new BroadcastChannel('name')
navigator.sendBeacon(url, data)        // survives page unload
structuredClone(value)

// Navigation and history
history.pushState(state, '', url)      // ⚠️ does NOT fire popstate
history.replaceState(state, '', url)
addEventListener('popstate', cb)       // user navigation only
new URL(href, base)  url.searchParams
new URLSearchParams(qs)

// Performance and metrics
performance.now() / mark(n) / measure(n, s, e) / getEntriesByType(t)
await performance.measureUserAgentSpecificMemory()   // needs cross-origin isolation

// Crypto
crypto.randomUUID()
crypto.getRandomValues(new Uint8Array(32))
await crypto.subtle.digest('SHA-256', bytes)

// Misc
navigator.onLine   addEventListener('online'|'offline', cb)
document.visibilityState   addEventListener('visibilitychange', cb)
matchMedia('(prefers-reduced-motion: reduce)').matches
navigator.clipboard.writeText(s)
new Notification(title, { body })      // needs permission from a user gesture
document.startViewTransition(cb)
```

## Intl and Temporal

```js
new Intl.NumberFormat('en-GB', { style: 'currency', currency: 'GBP' }).format(1234.5);
// '£1,234.50'
new Intl.NumberFormat('en', { notation: 'compact' }).format(1_200_000);     // '1.2M'
new Intl.NumberFormat('en', { style: 'unit', unit: 'megabyte' }).format(5); // '5 MB'

new Intl.DateTimeFormat('en-GB', { dateStyle: 'medium', timeStyle: 'short' }).format(d);
new Intl.RelativeTimeFormat('en', { numeric: 'auto' }).format(-1, 'day');   // 'yesterday'
new Intl.ListFormat('en', { type: 'conjunction' }).format(['a', 'b', 'c']); // 'a, b, and c'
new Intl.PluralRules('en').select(1);                                        // 'one'
new Intl.Collator('en', { sensitivity: 'base' }).compare(a, b);              // sorting
new Intl.Segmenter('en', { granularity: 'grapheme' });                       // real characters
Intl.DateTimeFormat().resolvedOptions().timeZone;                            // user's zone
```

```js
// Temporal (ES2025 — polyfill required in most browsers)
Temporal.Now.instant()                        // exact time
Temporal.Now.plainDateISO()                    // today, no time or zone
Temporal.PlainDate.from('2026-08-01')          // months are 1-INDEXED
Temporal.ZonedDateTime.from('2026-08-01T09:00[Europe/London]')
date.add({ months: 1, days: 3 })               // returns a NEW value
date.since(other)  date.until(other)           // → Duration
Temporal.Duration.from({ hours: 2 })
date.toLocaleString('en-GB')
```

⚠️ **`Date` traps:** mutable, months are 0-indexed, `new Date('2026-8-1')` parses
inconsistently across engines, and everything is silently local time. Use `Temporal` or a library.

---

**Next:** [Appendix G — Glossary →](G-glossary.md)
