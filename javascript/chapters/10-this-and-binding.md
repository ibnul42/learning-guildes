# Chapter 10 — `this` and Binding

## Learning Objectives

- Apply the five rules that determine `this` at every call site
- Use `call`, `apply`, and `bind` correctly
- Explain why arrow functions have no own `this`
- Fix lost-`this` bugs systematically

---

## 10.1 The One Rule That Matters

> **`this` is determined by *how a function is called*, not where it is defined.**

The single exception is arrow functions, which have no `this` of their own and inherit it lexically.

To resolve `this`, find the **call site** and apply these rules **in priority order**:

### 1. `new` binding — highest priority

```js
function Person(name) { this.name = name; }
const p = new Person('Ada');   // `this` is the newly created object
```

### 2. Explicit binding — `call` / `apply` / `bind`

```js
function greet() { return `Hi ${this.name}`; }
greet.call({ name: 'Ada' });        // 'Hi Ada'
greet.apply({ name: 'Bob' }, []);   // 'Hi Bob'  — args as an array
const bound = greet.bind({ name: 'Cy' });
bound();                             // 'Hi Cy' — permanently bound
```

`bind` wins over later `call`: `bound.call({name:'X'})` still gives `'Hi Cy'`.

### 3. Implicit binding — a method call

```js
const obj = { name: 'Ada', greet() { return this.name; } };
obj.greet();   // 'Ada' — `this` is whatever is left of the dot
```

**Only the *last* object before the dot matters:**

```js
const a = { name: 'A', b: { name: 'B', get() { return this.name; } } };
a.b.get();   // 'B'
```

### 4. Default binding

```js
function f() { return this; }
f();   // undefined in strict mode / modules; globalThis in sloppy mode
```

### 5. Arrow functions — lexical, and they override everything above

```js
const obj = {
  name: 'Ada',
  regular() { return this.name; },
  arrow: () => this?.name,          // `this` from the enclosing scope
};
obj.regular();   // 'Ada'
obj.arrow();     // undefined
```

You cannot rebind an arrow. `arrowFn.call({name:'X'})` is ignored.

---

## 10.2 The Lost `this` Problem

This is the #1 source of `this` bugs.

```js
const counter = {
  count: 0,
  increment() { this.count++; },
};

counter.increment();                       // ✅ 3 rules apply: implicit binding

const fn = counter.increment;
fn();                                       // ❌ TypeError — `this` is undefined

setTimeout(counter.increment, 100);         // ❌ same problem
[1,2].forEach(counter.increment);           // ❌ same problem
```

Extracting a method **discards the object**. The function value knows nothing about where it came from.

**Four fixes:**

```js
// 1. bind
setTimeout(counter.increment.bind(counter), 100);

// 2. arrow wrapper — usually the clearest
setTimeout(() => counter.increment(), 100);

// 3. thisArg, where the API supports it
[1,2].forEach(counter.increment, counter);

// 4. class field arrow — bound at construction
class Counter {
  count = 0;
  increment = () => { this.count++; };   // always bound to the instance
}
```

Option 4 costs one function object per instance (rather than one shared prototype method), which matters only if you create thousands of instances.

---

## 10.3 `call` vs `apply` vs `bind`

```js
fn.call(thisArg, a, b, c);     // invoke now, args listed
fn.apply(thisArg, [a, b, c]);  // invoke now, args as an array
const g = fn.bind(thisArg, a); // returns a new function, partially applied
```

Mnemonic: **A**pply takes an **A**rray. **C**all takes a **C**omma list. **B**ind returns a **B**ound function.

Spread has replaced most `apply` uses:

```js
Math.max.apply(null, nums);   // old
Math.max(...nums);            // modern
```

`bind` also does partial application, and the bound function's `this` cannot be changed again — even by `new`... except that `new` on a bound function *does* override `this` (though preset arguments are kept). An edge case worth knowing exists, not worth memorising.

Borrowing methods:

```js
const arrayLike = { 0: 'a', 1: 'b', length: 2 };
Array.prototype.slice.call(arrayLike);   // ['a','b']
Array.from(arrayLike);                    // ✅ modern equivalent
```

---

## 10.4 `this` in Different Contexts

```js
// Module top level
this;                     // undefined (in ESM); module.exports in CJS

// Script top level (non-module)
this;                     // globalThis

// Class method
class C { m() { return this; } }
new C().m();              // the instance

// Class static method
class D { static s() { return this; } }
D.s();                    // the class D itself

// Event handler (regular function)
btn.addEventListener('click', function () { return this; });  // the element
btn.addEventListener('click', () => this);                     // enclosing scope

// Constructor called without `new`
function P() { this.x = 1; }
P();   // TypeError in strict mode; pollutes globalThis in sloppy mode
```

> ⚠️ In a DOM event handler, a regular function's `this` is the element (same as `event.currentTarget`). Switching to an arrow silently changes this — a common cause of "it worked before I 'modernized' it."

---

## 10.5 Common Mistakes and Best Practices

**Mistake: arrow as an object method or prototype method.** No dynamic `this`.

**Mistake: passing a method reference as a callback.** Always wrap or bind.

**Mistake: `const self = this`.** It works, but arrows made it obsolete. If you see it, it's legacy.

**Mistake: binding in `render`.** `onClick={this.handle.bind(this)}` creates a new function every render, breaking memoization. Bind once in the constructor or use a class field.

**Best practice: prefer arrow class fields for handlers** and regular methods for everything else.

**Best practice: design so `this` doesn't matter.** Pure functions taking explicit arguments never have this problem. If a "method" doesn't use `this`, make it a standalone function or `static`.

---

## 10.6 Real-World Example: An Event-Bound Component

```js
class Carousel {
  #index = 0;
  #controller = new AbortController();

  constructor(root) {
    this.root = root;
    this.slides = [...root.querySelectorAll('.slide')];

    const { signal } = this.#controller;
    // Arrow callbacks: `this` stays the Carousel instance.
    root.querySelector('.next')
        .addEventListener('click', () => this.next(), { signal });
    root.querySelector('.prev')
        .addEventListener('click', () => this.prev(), { signal });

    // A regular function here would make `this` the <div>, not the Carousel:
    document.addEventListener('keydown', (e) => {
      if (e.key === 'ArrowRight') this.next();
      if (e.key === 'ArrowLeft') this.prev();
    }, { signal });

    this.#render();
  }

  next() { this.#index = (this.#index + 1) % this.slides.length; this.#render(); }
  prev() { this.#index = (this.#index - 1 + this.slides.length) % this.slides.length; this.#render(); }

  #render() {
    this.slides.forEach((el, i) => el.classList.toggle('active', i === this.#index));
  }

  destroy() { this.#controller.abort(); }   // removes every listener at once
}
```

Two things to notice: arrow callbacks preserve `this`, and a single `AbortController` handles all cleanup — no bookkeeping of bound references to pass back to `removeEventListener`.

---

## 10.7 Questions

**1.** MCQ — `this` is determined by: a) where the function is defined b) how it's called c) its name d) the file
<details><summary>Answer</summary>**b)** (Arrows excepted.)</details>

**2.** MCQ — Which has the highest binding priority? a) implicit b) explicit c) `new` d) default
<details><summary>Answer</summary>**c)**</details>

**3.** MCQ — `apply` differs from `call` by: a) return value b) `this` handling c) argument format d) speed
<details><summary>Answer</summary>**c)**</details>

**4.** MCQ — `this` in a static method is: a) the instance b) the class c) undefined d) globalThis
<details><summary>Answer</summary>**b)**</details>

**5.** MCQ — Can you rebind an arrow function's `this` with `call`? a) yes b) no
<details><summary>Answer</summary>**b)**</details>

**6.** MCQ — In a module's top-level scope, `this` is: a) `globalThis` b) `undefined` c) `module` d) `{}`
<details><summary>Answer</summary>**b)**</details>

**7.** Short — Why does `const fn = obj.method; fn();` lose `this`?
<details><summary>Answer</summary>Implicit binding comes from the call site's receiver. Assigning to a variable discards the receiver, so the call falls back to default binding.</details>

**8.** Short — `call` vs `apply` vs `bind` in one line each.
<details><summary>Answer</summary>`call` invokes with comma-separated args; `apply` invokes with an array of args; `bind` returns a permanently bound new function.</details>

**9.** Short — Why is an arrow wrong as an object method?
<details><summary>Answer</summary>It has no own `this`, so `this` resolves to the enclosing (usually module) scope rather than the object.</details>

**10.** Short — Trade-off of class-field arrow handlers?
<details><summary>Answer</summary>Correct `this` automatically, but one function object per instance instead of one shared on the prototype, and they aren't on the prototype so they can't be overridden by subclasses in the usual way.</details>

**11.** Short — What is `this` in a `forEach` callback by default?
<details><summary>Answer</summary>`undefined` in strict mode, unless you pass a `thisArg` as the second parameter to `forEach`.</details>

**12.** Output — What prints?
```js
const o = { n: 1, f() { return this.n; } };
const g = o.f;
console.log(o.f(), typeof g());
```
<details><summary>Answer</summary>`1` then a TypeError (reading `n` of undefined) in strict mode. In sloppy mode: `1 'undefined'`.</details>

**13.** Output — What prints?
```js
const o = { n: 1, f() { const inner = () => this.n; return inner(); } };
console.log(o.f());
```
<details><summary>Answer</summary>`1` — the arrow inherits `f`'s `this`.</details>

**14.** Output — What prints?
```js
function f() { return this?.n; }
console.log(f.call({n:1}), f.bind({n:2})(), f.bind({n:2}).call({n:3}));
```
<details><summary>Answer</summary>`1 2 2` — `bind` cannot be overridden by a later `call`.</details>

**15.** Output — What prints?
```js
const a = { n: 'A', b: { n: 'B', get() { return this.n; } } };
console.log(a.b.get());
```
<details><summary>Answer</summary>`'B'` — only the last object before the dot counts.</details>

**16.** Output — What prints?
```js
class C { static who() { return this.name; } }
console.log(C.who());
```
<details><summary>Answer</summary>`'C'` — `this` is the class, and classes have a `name`.</details>

**17.** Output — What prints?
```js
const obj = { n: 1, arrow: () => this?.n, fn() { return this.n; } };
console.log(obj.arrow(), obj.fn());
```
<details><summary>Answer</summary>`undefined 1`</details>

**18.** Debug — `this.items` is undefined inside the callback:
```js
class List {
  constructor() { this.items = []; }
  loadAll(urls) { urls.forEach(function (u) { this.items.push(u); }); }
}
```
<details><summary>Answer</summary>The plain callback has default binding. Use an arrow: `urls.forEach(u => this.items.push(u))`, or pass `this` as `forEach`'s second argument.</details>

**19.** Debug — A React class component's handler throws "cannot read setState of undefined."
<details><summary>Answer</summary>The handler was passed as a bare method reference to `onClick`, losing `this`. Bind in the constructor or convert to a class-field arrow.</details>

**20.** Debug — A previously working jQuery-style handler broke after refactoring to arrows:
```js
$('.btn').on('click', () => { $(this).hide(); });
```
<details><summary>Answer</summary>jQuery sets `this` to the element via `apply`, but an arrow ignores it. Use a regular function, or use `event.currentTarget`.</details>

**21.** Code — Implement `myBind`.
<details><summary>Answer</summary>

```js
Function.prototype.myBind = function (thisArg, ...preset) {
  const fn = this;
  function bound(...later) {
    // Support `new bound()`: then `this` should be the new object.
    return fn.apply(this instanceof bound ? this : thisArg, [...preset, ...later]);
  }
  bound.prototype = Object.create(fn.prototype || null);
  return bound;
};
```
</details>

**22.** Code — Implement `myCall` and `myApply` without using `call`/`apply`/`bind`.
<details><summary>Answer</summary>

```js
Function.prototype.myCall = function (thisArg, ...args) {
  const key = Symbol('fn');
  const ctx = thisArg ?? globalThis;
  ctx[key] = this;
  try { return ctx[key](...args); } finally { delete ctx[key]; }
};
Function.prototype.myApply = function (thisArg, args = []) {
  return this.myCall(thisArg, ...args);
};
```
A `Symbol` key avoids clobbering an existing property.
</details>

**23.** Code — Write `autoBind(instance)` binding every prototype method to the instance.
<details><summary>Answer</summary>

```js
function autoBind(instance) {
  const proto = Object.getPrototypeOf(instance);
  for (const name of Object.getOwnPropertyNames(proto)) {
    const desc = Object.getOwnPropertyDescriptor(proto, name);
    if (name !== 'constructor' && typeof desc.value === 'function') {
      instance[name] = desc.value.bind(instance);
    }
  }
  return instance;
}
```
Checking the descriptor's `value` avoids triggering getters.
</details>

**24.** Scenario — A library exposes `emitter.on('x', obj.handle)` in its docs, and users report `this` errors. What's the library's best fix?
<details><summary>Answer</summary>Accept an optional context argument (`on(event, fn, thisArg)`) and call `fn.call(thisArg ?? undefined, ...)`. Also update the docs to show `obj.handle.bind(obj)` or an arrow wrapper. The underlying issue is user-side, but a good API makes the correct usage the easy one.</details>

**25.** Scenario — Should you use arrow class fields for every method?
<details><summary>Answer</summary>No. Use them for handlers passed as callbacks. Regular prototype methods are shared across instances (cheaper), appear on the prototype (subclassable, spyable in tests), and are what most tooling expects.</details>

---

## Chapter Summary

- Resolve `this` by finding the **call site** and applying: `new` → explicit (`call`/`apply`/`bind`) → implicit (method call) → default → lexical (arrows override all).
- Extracting a method loses its receiver. Wrap in an arrow, `bind`, pass a `thisArg`, or use a class-field arrow.
- **A**pply→**A**rray, **C**all→**C**ommas, **B**ind→returns a **B**ound function.
- Arrows have no own `this` and cannot be rebound — ideal for callbacks, wrong for object/prototype methods and DOM handlers that need the element.
- The best fix for `this` bugs is often to not need `this`.

---

## Practice Exercises

1. Write a table predicting `this` for ten different call forms, then verify each.
2. Implement `myCall`, `myApply`, and `myBind` and pass a test suite including `new`.
3. Take a legacy file using `const self = this` and modernize it with arrows.
4. Build a `Timer` class in three variants (bind in constructor, arrow field, arrow wrapper) and compare memory with 10,000 instances.
5. Write a function that behaves correctly whether called as `f()`, `obj.f()`, or `new f()`.

---

**Next:** [Chapter 11 — Prototypes and Inheritance →](11-prototypes-inheritance.md)
