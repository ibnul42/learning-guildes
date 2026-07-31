# Chapter 8 — Scope, Hoisting, and the TDZ

## Learning Objectives

- Describe the scope chain and lexical environments
- Explain hoisting precisely — including *what* is hoisted and *how*
- Define the Temporal Dead Zone and predict TDZ errors
- Avoid variable shadowing bugs

---

## 8.1 Scope Types

```js
const global = 'module or script scope';

function outer() {
  const fnScoped = 'function scope';

  if (true) {
    var varLeaks = 'function-scoped';   // escapes the block
    let blockOnly = 'block-scoped';
  }

  console.log(varLeaks);   // ✅ 'function-scoped'
  console.log(blockOnly);  // ❌ ReferenceError
}
```

Four kinds: **global**, **module**, **function**, **block**. Blocks are created by `{}`, `if`, `for`, `while`, `try`, and bare braces.

**Lexical scoping** means scope is determined by where code is *written*, not where it's *called*:

```js
const x = 'outer';
function reader() { return x; }        // captures the writing location
function caller() { const x = 'inner'; return reader(); }
caller();   // 'outer' — NOT 'inner'
```

The **scope chain** is the ordered list of environments the engine searches: current → enclosing → … → global. Search stops at the first match; a miss throws `ReferenceError`.

---

## 8.2 Hoisting — What Actually Happens

"Hoisting" isn't code moving. Before executing a scope, the engine creates all its bindings. What differs is *initialization*.

| Declaration | Binding created | Initialized to | Usable before the line? |
|---|---|---|---|
| `function foo(){}` | yes | the function | ✅ yes |
| `var x` | yes | `undefined` | ⚠️ yes, but `undefined` |
| `let x` / `const x` | yes | **nothing** (TDZ) | ❌ ReferenceError |
| `class C {}` | yes | **nothing** (TDZ) | ❌ ReferenceError |
| `import` | yes | the binding | ✅ yes (hoisted to the top) |

```js
console.log(a);   // undefined  — var is initialized to undefined
console.log(b);   // ReferenceError: Cannot access 'b' before initialization
console.log(c);   // ReferenceError: c is not defined  ← different message!
f();              // ✅ 'hi'
g();              // ❌ TypeError: g is not a function

var a = 1;
let b = 2;
function f() { console.log('hi'); }
var g = function () {};
```

Note the **two different errors**: TDZ says "cannot access before initialization" (the binding exists); an undeclared name says "is not defined". That distinction is a favourite interview probe.

### Function declarations beat `var`

```js
console.log(typeof thing);   // 'function'
var thing = 1;
function thing() {}
console.log(typeof thing);   // 'number'
```

Function declarations are hoisted *and initialized*; the `var` declaration of the same name is redundant, and the assignment runs later.

---

## 8.3 The Temporal Dead Zone

The TDZ is the region between entering a scope and the line where a `let`/`const`/`class` binding is initialized.

```js
{
  // TDZ for `x` starts here
  console.log(typeof x);   // ❌ ReferenceError — even `typeof` throws!
  let x = 1;               // TDZ ends
  console.log(x);          // ✅ 1
}
```

> ⚠️ `typeof` is *safe* for undeclared variables (`typeof undeclared` → `'undefined'`) but **throws** inside a TDZ. This asymmetry catches people out.

TDZ is temporal, not spatial — it's about *time of execution*, not position in the file:

```js
function f() { return x; }   // no error yet — not called
let x = 1;
f();                          // ✅ fine, TDZ already ended
```

But:

```js
function f() { return x; }
f();          // ❌ ReferenceError — called during x's TDZ
let x = 1;
```

**Why the TDZ exists:** it turns "used before defined" from a silent `undefined` into a loud error, and it makes `const` meaningful (a `const` can't be `undefined` first and assigned later).

Also note the self-reference trap:

```js
let x = x + 1;   // ❌ ReferenceError — the right side reads x while in TDZ
var y = y + 1;   // ✅ NaN — y is undefined, undefined + 1
```

---

## 8.4 Shadowing

```js
const value = 'outer';
function f() {
  const value = 'inner';   // shadows — legal
  console.log(value);      // 'inner'
}

// Illegal shadowing
function g() {
  let a = 1;
  { var a = 2; }   // ❌ SyntaxError — var would escape into the same scope
}
```

Shadowing is legal and sometimes useful (narrowing a type, rebinding a parameter). It becomes a bug when unintentional — a linter rule (`no-shadow`) is worth enabling on teams.

**Parameter scope is subtle:**

```js
function f(a, b = () => a) {
  var a = 99;         // creates a SEPARATE binding in the body scope
  return b();         // 1 — the default closure captured the parameter scope
}
f(1);   // 1
```

When any parameter has a default, parameters get their own scope separate from the body.

---

## 8.5 Common Mistakes and Best Practices

**Mistake: relying on `var` hoisting.** Code that reads a variable above its declaration is confusing even when it "works."

**Mistake: `var` in loops with closures.** See Chapter 2 — one shared binding.

**Mistake: accidental globals.** In sloppy mode, `x = 1` without a declaration creates a global. Strict mode makes it a `ReferenceError`.

**Mistake: assuming `typeof` is always safe.** It isn't in a TDZ.

**Best practice: declare at first use, not at the top.** The old "declare everything at the top" style existed to work around `var`. With `let`/`const`, narrow scope is better.

**Best practice: keep functions small.** The scope chain gets long and hard to reason about in 200-line functions.

---

## 8.6 Real-World Example: A Module-Scoped Singleton

```js
// analytics.js — module scope is the natural place for private state

let queue = [];               // module-private; not on globalThis
let flushTimer = null;        // TDZ-protected, never accidentally read early
const MAX_BATCH = 20;

// Function declaration: hoisted, so `track` below can reference it safely
function scheduleFlush() {
  if (flushTimer !== null) return;
  flushTimer = setTimeout(flush, 5000);
}

export function track(event, props = {}) {
  queue.push({ event, props, at: Date.now() });
  if (queue.length >= MAX_BATCH) flush();
  else scheduleFlush();       // ✅ works: declaration hoisted
}

async function flush() {
  clearTimeout(flushTimer);
  flushTimer = null;
  if (queue.length === 0) return;

  const batch = queue;        // block-scoped snapshot
  queue = [];                 // reset before awaiting — avoids losing events

  try {
    await fetch('/collect', { method: 'POST', body: JSON.stringify(batch) });
  } catch {
    queue = [...batch, ...queue];   // re-queue on failure
  }
}
```

Module scope gives encapsulation with no class and no IIFE. `queue` and `flushTimer` are unreachable from outside the file — real privacy enforced by the module system.

---

## 8.7 Questions

**1.** MCQ — `console.log(x); let x = 1;` throws: a) TypeError b) ReferenceError c) SyntaxError d) nothing
<details><summary>Answer</summary>**b)**</details>

**2.** MCQ — `var` is scoped to: a) block b) function c) module d) global only
<details><summary>Answer</summary>**b)**</details>

**3.** MCQ — `typeof` inside a TDZ: a) returns `'undefined'` b) throws c) returns `'tdz'` d) returns `null`
<details><summary>Answer</summary>**b)**</details>

**4.** MCQ — Which is hoisted AND initialized? a) `let` b) `const` c) `class` d) function declaration
<details><summary>Answer</summary>**d)**</details>

**5.** MCQ — JavaScript uses: a) dynamic scoping b) lexical scoping c) both d) neither
<details><summary>Answer</summary>**b)**</details>

**6.** MCQ — `let x = x + 1;` produces: a) `NaN` b) `1` c) ReferenceError d) `undefined`
<details><summary>Answer</summary>**c)**</details>

**7.** Short — Define hoisting in one sentence.
<details><summary>Answer</summary>Bindings for all declarations in a scope are created before any code in that scope runs; only their initialization timing differs.</details>

**8.** Short — What is the TDZ and why does it exist?
<details><summary>Answer</summary>The window between entering a scope and initializing a `let`/`const`/`class` binding, during which access throws. It converts use-before-definition from a silent `undefined` into an explicit error and makes `const` semantically coherent.</details>

**9.** Short — Difference between "Cannot access 'x' before initialization" and "x is not defined"?
<details><summary>Answer</summary>The first means the binding exists but is in the TDZ. The second means no binding exists in the scope chain at all.</details>

**10.** Short — Why is `let` in a `for` header special?
<details><summary>Answer</summary>The spec creates a fresh binding per iteration and copies the previous value in, so closures capture distinct variables.</details>

**11.** Short — Name two ways to create a private variable without a class.
<details><summary>Answer</summary>Module scope (unexported bindings) and a closure (IIFE or factory function).</details>

**12.** Output — What prints?
```js
var x = 1;
function f() { console.log(x); var x = 2; }
f();
```
<details><summary>Answer</summary>`undefined` — the inner `var x` is hoisted, shadowing the outer one.</details>

**13.** Output — What prints?
```js
function f() { console.log(typeof g); { function g() {} } }
f();
```
<details><summary>Answer</summary>`'undefined'` in strict mode/modules (block-scoped function declaration, not yet reached). In sloppy mode Annex B semantics hoist a `var g` binding initialized to `undefined`, giving the same result here.</details>

**14.** Output — What prints?
```js
let a = 1;
{ console.log(a); let a = 2; }
```
<details><summary>Answer</summary>ReferenceError — the inner `a` shadows from the moment the block is entered, so the log hits its TDZ.</details>

**15.** Output — What prints?
```js
console.log(f(), typeof g);
function f() { return 'F'; }
var g = function () {};
```
<details><summary>Answer</summary>`'F' 'undefined'`</details>

**16.** Output — What prints?
```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i), 0);
for (let j = 0; j < 3; j++) setTimeout(() => console.log(j), 0);
```
<details><summary>Answer</summary>`3 3 3` then `0 1 2`.</details>

**17.** Debug — A config value reads as `undefined` intermittently:
```js
function init() { setup(); var config = load(); }
function setup() { console.log(config); }
```
<details><summary>Answer</summary>`config` in `setup` isn't the local one — it's an outer/global lookup, likely `undefined` or a `ReferenceError`. Pass it explicitly: `setup(load())`.</details>

**18.** Debug — A `class` used before its definition throws, but a function in the same position works. Why?
<details><summary>Answer</summary>Class declarations are hoisted into the TDZ, not initialized. Function declarations are initialized at hoist time. Move the class above its first use.</details>

**19.** Debug — In a bundled app, a helper suddenly refers to the wrong variable after a merge.
<details><summary>Answer</summary>Almost certainly accidental shadowing — an inner declaration with the same name captured the reference. Enable `no-shadow` and rename.</details>

**20.** Code — Fix this without `let`, so it logs 0,1,2:
```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i));
```
<details><summary>Answer</summary>

```js
for (var i = 0; i < 3; i++) {
  ((n) => setTimeout(() => console.log(n)))(i);   // IIFE captures a copy
}
```
</details>

**21.** Code — Write `createCounter()` with truly private state and `inc`/`dec`/`value`.
<details><summary>Answer</summary>

```js
function createCounter(start = 0) {
  let count = start;                     // unreachable from outside
  return {
    inc() { return ++count; },
    dec() { return --count; },
    get value() { return count; },
  };
}
```
</details>

**22.** Code — Demonstrate the parameter-scope quirk with a runnable example and explain it.
<details><summary>Answer</summary>

```js
function f(a, getA = () => a) {
  var a = 99;
  return [a, getA()];
}
console.log(f(1));   // [99, 1]
```
Because a default exists, parameters live in their own scope. `var a` in the body creates a separate binding initialized from the parameter, then reassigned. The default's closure still sees the parameter scope.
</details>

**23.** Scenario — A teammate proposes banning shadowing entirely. Reasonable?
<details><summary>Answer</summary>Mostly. Ban *accidental* shadowing with `no-shadow`, but allow deliberate narrowing (e.g. `const user = maybeUser` after a guard) via targeted disables. A blanket ban leads to names like `user2`, which is worse.</details>

**24.** Scenario — Why do bundlers wrap each module in a function scope?
<details><summary>Answer</summary>To emulate module scope in environments without native ESM, keeping each module's top-level bindings private and preventing collisions across the concatenated bundle.</details>

---

## Chapter Summary

- Scope is **lexical** — determined by where code is written.
- Hoisting creates bindings early; only *initialization* differs. Function declarations are ready; `var` is `undefined`; `let`/`const`/`class` sit in the **TDZ**.
- TDZ access throws — including via `typeof`.
- "Not defined" ≠ "cannot access before initialization."
- `let` in `for` headers gets a fresh binding per iteration.
- Module scope is the cleanest way to get private state.

---

## Practice Exercises

1. Write a file demonstrating all five hoisting behaviours from the table, with comments predicting each output. Verify.
2. Explain why `typeof undeclaredVar` is safe but `typeof tdzVar` throws.
3. Take a function using `var` throughout and convert it to `let`/`const`, documenting every behaviour change.
4. Build a nested scope-chain example five levels deep and trace resolution for a name declared at level 2.
5. Reproduce the parameter-scope quirk and write a test that pins the behaviour.

---

**Next:** [Chapter 9 — Closures →](09-closures.md)
