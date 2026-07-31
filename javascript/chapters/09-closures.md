# Chapter 9 — Closures

## Learning Objectives

- Define a closure precisely and identify one in any code sample
- Use closures for privacy, factories, memoization, and partial application
- Recognise closure-caused memory leaks
- Answer the standard closure interview questions confidently

---

## 9.1 What a Closure Is

> A **closure** is a function bundled with a reference to its lexical environment — the scope where it was *created*.

Every JavaScript function is a closure. The term matters when the function *outlives* the scope it captured.

```js
function makeGreeter(greeting) {
  // `greeting` would normally die when makeGreeter returns
  return function (name) {
    return `${greeting}, ${name}!`;   // ...but the inner fn holds a reference
  };
}

const hello = makeGreeter('Hello');
const hi = makeGreeter('Hi');

hello('Ada');   // 'Hello, Ada!'
hi('Bob');      // 'Hi, Bob!'   ← separate environment, separate `greeting`
```

Each call to `makeGreeter` creates a **new environment**. The two closures do not share `greeting`.

**Closures capture variables, not values:**

```js
function make() {
  let n = 0;
  const read = () => n;
  n = 42;              // changed AFTER read was created
  return read;
}
make()();   // 42 — the live variable, not a snapshot
```

This is the single most important sentence in the chapter, and it explains the `var`-loop bug:

```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i));   // 3,3,3
// One `i`. All three closures reference the same variable, read after the loop.

for (let i = 0; i < 3; i++) setTimeout(() => console.log(i));   // 0,1,2
// Three bindings. Three environments.
```

---

## 9.2 The Five Practical Uses

### 1. Data privacy

```js
function createAccount(initial = 0) {
  let balance = initial;                          // truly private
  return {
    deposit(n) { if (n > 0) balance += n; return balance; },
    withdraw(n) {
      if (n > balance) throw new RangeError('Insufficient funds');
      balance -= n; return balance;
    },
    get balance() { return balance; },
  };
}
const acct = createAccount(100);
acct.balance;      // 100
acct.balance = 1e9; // no-op — getter only
// No way to reach `balance` from outside. Not a convention: a guarantee.
```

### 2. Function factories

```js
const multiplyBy = (f) => (n) => n * f;
const double = multiplyBy(2);
const triple = multiplyBy(3);
```

### 3. Memoization

```js
function memoize(fn) {
  const cache = new Map();          // captured, lives as long as the closure
  return (...args) => {
    const key = args.join('|');
    if (!cache.has(key)) cache.set(key, fn(...args));
    return cache.get(key);
  };
}
```

### 4. Once / rate limiting

```js
function once(fn) {
  let done = false, result;
  return (...a) => done ? result : (done = true, result = fn(...a));
}

function throttle(fn, ms) {
  let last = 0;                     // captured across calls
  return (...a) => {
    const now = Date.now();
    if (now - last >= ms) { last = now; fn(...a); }
  };
}
```

### 5. Module pattern

```js
const Store = (() => {
  const items = new Map();                    // private
  return {
    set: (k, v) => (items.set(k, v), undefined),
    get: (k) => items.get(k),
    get size() { return items.size; },
  };
})();
```

ES modules made this pattern mostly redundant — but you'll read it in legacy code constantly.

---

## 9.3 Closures and Memory

A closure keeps its captured environment alive. That's the point — and the hazard.

```js
function leaky() {
  const hugeArray = new Array(1e6).fill('data');   // ~8 MB
  return () => 'hi';                                // doesn't use hugeArray...
}
```

Modern V8 is smart: it only retains variables the inner function actually references, so `hugeArray` is usually collected here. But **don't rely on it** — this optimization has holes, notably when `eval` or a debugger is present, and when *any* inner function in the same scope references the variable:

```js
function reallyLeaky() {
  const huge = new Array(1e6).fill('x');
  const unused = () => huge.length;   // this forces `huge` to be retained
  return () => 'hi';                   // ...even though the RETURNED fn doesn't use it
}
```

The classic real leak is an event listener:

```js
function attach(node) {
  const bigData = loadEverything();
  node.addEventListener('click', () => console.log(bigData.length));
  // If the listener is never removed, `bigData` AND `node` stay alive forever.
}
```

**Fix:** remove listeners, use `{ signal }` with an `AbortController`, or null out large captures explicitly. Chapter 27 goes deeper.

> ⚠️ **Gotcha** — closures over a `var` loop counter all share one binding, so they all see the final value. `let` creates a fresh binding per iteration and they each see their own. This is the single most-asked closure question, and the fix is one keyword.

> 💼 **Interview** — "Does a closure capture the value or the variable?" The variable. Expect the follow-up immediately: rewrite a `var` loop to log `0 1 2` without changing `var` to `let` — the answer is an IIFE per iteration, which is what `let` does for you.

---

## 9.4 Common Mistakes and Best Practices

**Mistake: expecting a snapshot.** Closures see the current value. If you need a snapshot, copy: `const snapshot = value;` inside the loop body.

**Mistake: creating closures in hot loops.** Each iteration allocates a new function object. Usually irrelevant; occasionally a real cost in tight render loops.

**Mistake: unbounded memoization caches.** They grow forever. Bound the size or use a `WeakMap` keyed by object identity.

**Best practice: return an interface, not the internals.** A factory returning `{ get value() }` gives read-only access without exposing the variable.

**Best practice: name your returned functions.** `return function tick() {}` produces far better stack traces than an anonymous arrow.

---

## 9.5 Real-World Example: A Rate-Limited API Client

```js
/**
 * Every piece of state here lives in closures — no class, no `this`,
 * nothing reachable or corruptible from outside.
 */
function createApiClient({ baseUrl, maxPerSecond = 5, cacheTtlMs = 30_000 }) {
  const cache = new Map();                 // url -> { at, promise }
  let tokens = maxPerSecond;
  const queue = [];

  setInterval(() => {                      // refill bucket
    tokens = maxPerSecond;
    while (tokens > 0 && queue.length) { tokens--; queue.shift()(); }
  }, 1000).unref?.();

  function scheduled(fn) {
    return new Promise((resolve, reject) => {
      const run = () => fn().then(resolve, reject);
      if (tokens > 0) { tokens--; run(); } else queue.push(run);
    });
  }

  function get(path) {
    const url = baseUrl + path;
    const hit = cache.get(url);
    if (hit && Date.now() - hit.at < cacheTtlMs) return hit.promise;

    const promise = scheduled(() => fetch(url).then(r => {
      if (!r.ok) throw new Error(`HTTP ${r.status}`);
      return r.json();
    })).catch(err => { cache.delete(url); throw err; });   // don't cache failures

    cache.set(url, { at: Date.now(), promise });
    return promise;
  }

  return { get, clearCache: () => cache.clear(), get pending() { return queue.length; } };
}
```

`cache`, `tokens`, and `queue` are unreachable from the returned object except through the methods that are allowed to touch them.

---

## 9.6 Questions

**1.** MCQ — A closure captures: a) values b) variables c) copies d) types
<details><summary>Answer</summary>**b)** Live variable bindings.</details>

**2.** MCQ — How many closures does `for (let i=0;i<3;i++) fns.push(()=>i)` create with distinct `i`? a) 1 b) 2 c) 3 d) 0
<details><summary>Answer</summary>**c)**</details>

**3.** MCQ — Closures are primarily used for: a) speed b) encapsulation c) inheritance d) typing
<details><summary>Answer</summary>**b)**</details>

**4.** MCQ — Which does NOT create a closure? a) nested function b) arrow in a callback c) a top-level function referencing only globals d) all create closures
<details><summary>Answer</summary>**d)** Technically every function closes over its environment; the term is meaningful when the environment outlives its creator.</details>

**5.** Short — Define a closure in one sentence.
<details><summary>Answer</summary>A function together with the lexical environment it was created in, which it keeps alive and can read/write even after that scope has returned.</details>

**6.** Short — Why does `let` fix the loop bug and `var` doesn't?
<details><summary>Answer</summary>`let` creates a fresh binding per iteration, so each closure captures a distinct variable; `var` has one function-scoped binding shared by all.</details>

**7.** Short — How can a closure cause a memory leak?
<details><summary>Answer</summary>By keeping a reference to a large object (or a DOM node) alive for as long as the closure itself is reachable — commonly via an un-removed event listener or an unbounded cache.</details>

**8.** Short — Do two calls to the same factory share state?
<details><summary>Answer</summary>No. Each invocation creates a new environment.</details>

**9.** Short — How do closures enable the module pattern?
<details><summary>Answer</summary>An IIFE creates a scope; the returned object's methods close over the private variables, exposing only a controlled interface.</details>

**10.** Output — What prints?
```js
function f() { let c = 0; return { inc: () => ++c, get: () => c }; }
const a = f(), b = f();
a.inc(); a.inc(); b.inc();
console.log(a.get(), b.get());
```
<details><summary>Answer</summary>`2 1`</details>

**11.** Output — What prints?
```js
const fns = [];
for (var i = 0; i < 3; i++) fns.push(() => i);
console.log(fns.map(f => f()));
```
<details><summary>Answer</summary>`[3, 3, 3]`</details>

**12.** Output — What prints?
```js
function f() { let x = 1; setTimeout(() => console.log(x), 0); x = 2; }
f();
```
<details><summary>Answer</summary>`2` — the closure reads the variable when the callback runs.</details>

**13.** Output — What prints?
```js
const add = (a) => (b) => a + b;
const add5 = add(5);
console.log(add5(3), add(1)(2));
```
<details><summary>Answer</summary>`8 3`</details>

**14.** Output — What prints?
```js
let x = 1;
const read = () => x;
x = 99;
console.log(read());
```
<details><summary>Answer</summary>`99`</details>

**15.** Output — What prints?
```js
function outer() {
  const arr = [];
  for (let i = 0; i < 3; i++) { arr.push(() => i * 2); }
  return arr;
}
console.log(outer().map(f => f()));
```
<details><summary>Answer</summary>`[0, 2, 4]`</details>

**16.** Debug — All buttons alert the last index. Fix without `let`.
```js
for (var i = 0; i < btns.length; i++)
  btns[i].onclick = () => alert(i);
```
<details><summary>Answer</summary>Wrap in an IIFE: `(function(n){ btns[n].onclick = () => alert(n); })(i);` — or just use `let`, or `forEach`.</details>

**17.** Debug — A memoized function grows to gigabytes.
<details><summary>Answer</summary>Unbounded cache. Add LRU eviction with a `maxSize`, add a TTL, or key by object identity in a `WeakMap` so entries die with their keys.</details>

**18.** Debug — A SPA's memory climbs on every route change even after components unmount.
<details><summary>Answer</summary>Listeners registered in closures over component data are never removed, so both the closure and the captured data stay reachable. Register with `{ signal: controller.signal }` and abort on teardown.</details>

**19.** Code — Write `createCounter()` supporting `inc(step)`, `reset()`, and a read-only `value`.
<details><summary>Answer</summary>

```js
function createCounter(start = 0) {
  let n = start;
  return {
    inc(step = 1) { n += step; return n; },
    reset() { n = start; return n; },
    get value() { return n; },
  };
}
```
</details>

**20.** Code — Implement `once(fn)` that also releases the reference to `fn` after the first call.
<details><summary>Answer</summary>

```js
function once(fn) {
  let result, called = false;
  return function (...a) {
    if (!called) { called = true; result = fn.apply(this, a); fn = null; }
    return result;
  };
}
```
Setting `fn = null` matters: otherwise the closure pins `fn` and everything it captured, forever.
</details>

**21.** Code — Write a `secretHolder(secret)` where the secret can be read only with the correct passphrase, and only three attempts are allowed.
<details><summary>Answer</summary>

```js
function secretHolder(secret, passphrase, maxAttempts = 3) {
  let attempts = 0;
  return function reveal(guess) {
    if (attempts >= maxAttempts) throw new Error('Locked out');
    attempts++;
    if (guess !== passphrase) throw new Error('Wrong passphrase');
    attempts = 0;
    return secret;
  };
}
```
</details>

**22.** Code — Implement `limitCalls(fn, n)` — after `n` invocations it throws.
<details><summary>Answer</summary>

```js
const limitCalls = (fn, n) => {
  let used = 0;
  return (...a) => {
    if (used++ >= n) throw new Error(`Limit of ${n} calls exceeded`);
    return fn(...a);
  };
};
```
</details>

**23.** Scenario — An interviewer asks: "Isn't a closure just an object with one method?" How do you respond?
<details><summary>Answer</summary>They're duals — both bundle state with behaviour. Closures give genuinely inaccessible state and are lighter for single-behaviour cases; objects/classes are better when you need several coordinated methods, inheritance, or introspection. Historically, closures were the only real privacy in JS before `#private` fields.</details>

**24.** Scenario — Your team's React app re-renders excessively because inline arrow props create new closures each render. Real problem?
<details><summary>Answer</summary>Only if the child is memoized and the prop identity defeats the memo. Allocation cost itself is negligible. Measure first; then use `useCallback` on the specific props feeding memoized children, rather than wrapping everything.</details>

---

## Chapter Summary

- A closure is a function plus its captured lexical environment. It captures **variables**, not values.
- Each factory invocation creates a fresh environment — no shared state between instances.
- Uses: privacy, factories, memoization, `once`/`throttle`/`debounce`, the module pattern.
- Closures keep captured data alive; un-removed listeners and unbounded caches are the two common leaks.
- Set large captured references to `null` when a closure's job is done.

---

## Practice Exercises

1. Implement `debounce` with `cancel()` and `flush()`, using only closures.
2. Build an LRU cache in a closure with `get`/`set`/`has` and a `maxSize`.
3. Write a closure-based event emitter (`on`, `off`, `emit`, `once`) with no `this`.
4. Demonstrate — with a heap snapshot — a closure leak, then fix it.
5. Reimplement `Function.prototype.bind` using a closure.

---

## Mini Project: A Plugin System

Build `plugins.js` exporting `createRegistry()`.

**Requirements:**

- Plugins are registered by name; the registry's internal map is unreachable from outside.
- `register(name, factory)` calls `factory(api)` where `api` exposes only a whitelisted surface.
- Each plugin gets its own isolated storage closure, so plugins can't read each other's state.
- `dispose(name)` must fully release the plugin's captured memory (verify with a heap snapshot).
- Support `hook(event, fn)` with priorities and an `AbortSignal` for automatic cleanup.

---

**Next:** [Chapter 10 — `this` and Binding →](10-this-and-binding.md)
