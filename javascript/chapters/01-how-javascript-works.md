# Chapter 1 — How JavaScript Works

## Learning Objectives

By the end of this chapter you will be able to:

- Explain the difference between the JavaScript *language*, an *engine*, and a *runtime*
- Describe what TC39 and ECMAScript are, and how new features ship
- Trace what happens between typing `node app.js` and seeing output
- Enable and justify strict mode
- Choose correctly between browser and Node.js execution contexts

---

## 1.1 JavaScript Is Three Things

People say "JavaScript" to mean three separate layers. Interviewers notice when you conflate them.

**The language** is defined by the ECMAScript specification. It gives you syntax, types, operators, `Array`, `Promise`, `Math`. Nothing more. The spec has no idea what a file, a network socket, or a button is.

**The engine** implements the language. V8 (Chrome, Node.js, Deno), SpiderMonkey (Firefox), JavaScriptCore (Safari, Bun). An engine parses your source and executes it.

**The runtime** wraps the engine and adds capabilities the spec doesn't define. The browser runtime supplies `document`, `fetch`, `localStorage`, `setTimeout`. The Node.js runtime supplies `fs`, `process`, `Buffer`, and *its own* `setTimeout`.

```js
// Pure ECMAScript — runs anywhere
const total = [1, 2, 3].reduce((a, b) => a + b, 0);

// Browser runtime only — no such thing in Node
document.title = 'Hello';

// Node runtime only — no such thing in a browser
const fs = require('node:fs');
```

> 💼 **Interview** — "Is `setTimeout` part of JavaScript?" The answer is no. It's a host-provided API. Both browsers and Node implement it, but with different internals and slightly different guarantees.

---

## 1.2 ECMAScript and TC39

TC39 is the committee that evolves the language. Proposals move through five stages:

| Stage | Name | Meaning |
|-------|------|---------|
| 0 | Strawperson | An idea someone had |
| 1 | Proposal | Worth exploring; a champion exists |
| 2 | Draft | Syntax and semantics sketched |
| 2.7 | — | Spec text complete and reviewed, awaiting tests |
| 3 | Candidate | Spec complete; engines start implementing |
| 4 | Finished | Two implementations shipped; enters the next yearly release |

A new ECMAScript edition drops every June, named for its year: ES2015 (also called ES6), ES2016, … ES2024, ES2025.

**Practical rule:** Stage 3 features are usually safe behind a transpiler. Stage 4 features are the standard. Anything below Stage 3 can and does change.

---

## 1.3 What Happens When Code Runs

Say you run this file:

```js
console.log('start');
const n = compute(21);
console.log(n);

function compute(x) {
  return x * 2;
}
```

Roughly, in order:

**1. Parsing.** The engine tokenizes your source into meaningful units (`const`, `n`, `=`, …) and builds an Abstract Syntax Tree. Syntax errors are thrown *here*, before any line executes. This is why a typo on line 500 prevents line 1 from running.

**2. Creation phase.** For each scope, the engine sets up a lexical environment. Function *declarations* are fully hoisted (available immediately). `var` bindings are created and initialized to `undefined`. `let` and `const` bindings are created but left uninitialized — the Temporal Dead Zone. That's why `compute(21)` works even though the function appears below it. Chapter 8 covers this in depth.

**3. Execution.** The interpreter (Ignition, in V8) generates bytecode and runs it. Hot code — functions called many times — is handed to an optimizing compiler (TurboFan) which produces machine code with speculative assumptions. If those assumptions break (you suddenly pass a string where you always passed a number), the code is *deoptimized* back to bytecode.

**4. Garbage collection** runs concurrently, reclaiming unreachable memory (Chapter 27).

This is called **JIT** — Just-In-Time compilation. JavaScript is neither purely interpreted nor purely compiled.

```js
// This function will be optimized after enough calls
function add(a, b) { return a + b; }

for (let i = 0; i < 100_000; i++) add(i, i);  // monomorphic: always numbers, fast

add('x', 'y'); // polymorphic now — V8 may deoptimize
```

> ⚠️ **Gotcha** — Don't write "clever" micro-optimizations based on JIT folklore. Engines change every six weeks. Measure (Chapter 28) instead of guessing.

---

## 1.4 Strict Mode

ES5 introduced strict mode to remove the worst legacy behaviours. Enable it with a directive at the top of a file or function:

```js
'use strict';
```

What changes:

```js
'use strict';

undeclaredVar = 5;    // ❌ ReferenceError (sloppy mode: creates a global)

function f() {
  console.log(this);  // undefined (sloppy mode: the global object)
}
f();

const frozen = Object.freeze({ a: 1 });
frozen.a = 2;         // ❌ TypeError (sloppy mode: silently fails)

// Duplicate parameter names are a SyntaxError
// `delete someVariable` is a SyntaxError
// Octal literals like 010 are a SyntaxError
```

**You are already in strict mode if:**

- You're inside an ES module (`.mjs`, or `<script type="module">`)
- You're inside a `class` body

Both are strict *automatically and permanently* — you cannot opt out. Since modern code is module code, strict mode is effectively the default for new work.

> 💼 **Interview** — "Name three things strict mode changes." Good answers: assignments to undeclared variables throw; `this` is `undefined` in plain function calls; writes to non-writable properties throw instead of failing silently.

---

## 1.5 Common Mistakes and Best Practices

**Mistake: assuming browser globals exist in Node.**

```js
// ❌ Breaks in Node
if (window.innerWidth > 768) { /* ... */ }

// ✅ Feature-detect
if (typeof window !== 'undefined' && window.innerWidth > 768) { /* ... */ }
```

**Mistake: relying on sloppy-mode leniency.** Code that "works" without strict mode often has a latent bug — a typo creating an accidental global, for instance.

**Mistake: confusing the engine with the runtime.** V8 provides the language; `fetch`, `fs`, and the DOM come from Chrome or Node. A feature "not working" is usually a missing host API, not a missing language feature.

**Mistake: treating a TC39 stage as a support guarantee.** Stage 3 means the spec text is done and implementations are invited, not that any browser you target has one.

**Mistake: reading `'use strict'` as file-wide when it isn't first.** Any statement before it — even a stray comment-plus-expression — silently makes it an ordinary string literal and leaves the file sloppy.

**Best practice: use modules.** `<script type="module">` gives you strict mode, deferred loading, and real imports for free.

**Best practice: pin your target.** Decide which browsers/Node versions you support, then let tooling (Babel, esbuild, TypeScript) enforce it. "It works on my machine" is not a support matrix.

---

## 1.6 Real-World Example: One Codebase, Two Runtimes

A shared validation module used by both a browser form and a Node API:

```js
// validate.js — pure ECMAScript, zero host dependencies
export function validateEmail(value) {
  if (typeof value !== 'string') return { ok: false, error: 'must be a string' };
  const trimmed = value.trim();
  if (trimmed.length === 0) return { ok: false, error: 'required' };
  if (!trimmed.includes('@')) return { ok: false, error: 'must contain @' };
  return { ok: true, value: trimmed.toLowerCase() };
}
```

```js
// browser.js — browser runtime supplies the DOM
import { validateEmail } from './validate.js';

document.querySelector('#email').addEventListener('blur', (e) => {
  const result = validateEmail(e.target.value);
  e.target.setCustomValidity(result.ok ? '' : result.error);
});
```

```js
// server.js — Node runtime supplies HTTP
import { validateEmail } from './validate.js';

export function handler(req, res) {
  const result = validateEmail(req.body.email);
  if (!result.ok) return res.status(400).json({ error: result.error });
  // ... persist result.value
}
```

The lesson: **push logic into runtime-agnostic modules**. Only the thin edges touch `document` or `req`. This is also what makes the core testable without a browser.

---

## 1.7 Questions

**1.** MCQ — Which is *not* defined by the ECMAScript specification? a) `Promise` b) `Array.prototype.map` c) `setTimeout` d) `Math.random`
<details><summary>Answer</summary>**c)** `setTimeout` is a host API. Browsers and Node each supply their own.</details>

**2.** MCQ — What is V8? a) A runtime b) A JavaScript engine c) A framework d) A spec version
<details><summary>Answer</summary>**b)** An engine. Node.js and Chrome are runtimes that *embed* V8.</details>

**3.** MCQ — At which TC39 stage do engines typically begin shipping implementations? a) 1 b) 2 c) 3 d) 4
<details><summary>Answer</summary>**c)** Stage 3 (Candidate).</details>

**4.** MCQ — Which construct is *always* in strict mode? a) A `function` declaration b) A `class` body c) An arrow function d) A `for` loop
<details><summary>Answer</summary>**b)** Class bodies are strict, unconditionally.</details>

**5.** MCQ — ES6 is another name for: a) ES2015 b) ES2016 c) ES2020 d) ES2025
<details><summary>Answer</summary>**a)** ES2015.</details>

**6.** MCQ — JIT compilation means: a) Code is compiled ahead of shipping b) Code is interpreted, then hot paths are compiled at runtime c) Code is never compiled d) Code is compiled by the developer
<details><summary>Answer</summary>**b)**</details>

**7.** MCQ — In strict mode, `this` inside a plain function call is: a) The global object b) `undefined` c) The function d) `null`
<details><summary>Answer</summary>**b)** `undefined`.</details>

**8.** Short — Why does a syntax error on the last line of a file stop the first line from running?
<details><summary>Answer</summary>Parsing completes before execution begins. The engine builds the entire AST first; if that fails, no code runs at all.</details>

**9.** Short — What is deoptimization?
<details><summary>Answer</summary>When the optimizing compiler's speculative assumptions (e.g. "this parameter is always a small integer") are violated at runtime, the engine discards the optimized machine code and falls back to bytecode.</details>

**10.** Short — Give two ways to enter strict mode without writing `'use strict'`.
<details><summary>Answer</summary>Use an ES module, or write code inside a `class` body.</details>

**11.** Short — Why is `'use strict'` at the top of an ES module redundant?
<details><summary>Answer</summary>Modules are strict by specification. The directive has no additional effect.</details>

**12.** Short — What is the difference between an engine and a runtime, in one sentence each?
<details><summary>Answer</summary>An engine parses and executes ECMAScript. A runtime is an engine plus host-provided APIs (DOM, filesystem, timers) and an event loop.</details>

**13.** Output — What prints?
```js
console.log(typeof greet);
function greet() {}
```
<details><summary>Answer</summary>`'function'` — function declarations are fully hoisted during the creation phase.</details>

**14.** Output — What prints?
```js
'use strict';
function f() { return this; }
console.log(f());
```
<details><summary>Answer</summary>`undefined`</details>

**15.** Output — What prints?
```js
function f() { return this; }
console.log(f() === globalThis);
```
<details><summary>Answer</summary>`true` in sloppy mode (a non-module script). In a module it would be `false`, because modules are strict and `this` would be `undefined`.</details>

**16.** Output — What prints?
```js
'use strict';
const o = Object.freeze({ x: 1 });
o.x = 99;
console.log(o.x);
```
<details><summary>Answer</summary>It throws `TypeError: Cannot assign to read only property 'x'`. Nothing is logged. Without strict mode it would log `1` after silently ignoring the write.</details>

**17.** Debug — This works locally in the browser but crashes on the Node server. Why?
```js
export function getTheme() {
  return localStorage.getItem('theme') ?? 'light';
}
```
<details><summary>Answer</summary>`localStorage` is a browser API and is `undefined` in Node, producing a `ReferenceError`. Guard it: `typeof localStorage === 'undefined' ? 'light' : localStorage.getItem('theme') ?? 'light'`. Better still, inject storage as a dependency so the function stays runtime-agnostic.</details>

**18.** Debug — A junior reports that this "silently does nothing":
```js
function setup() {
  confg = { retries: 3 };  // note the typo
}
setup();
console.log(config);
```
<details><summary>Answer</summary>In sloppy mode the typo `confg` creates an accidental global instead of erroring, and `config` is then undefined — a `ReferenceError` at the log, with a confusing cause. Strict mode turns the typo itself into an immediate `ReferenceError` at the real location.</details>

**19.** Code — Write a function `getRuntime()` that returns `'node'`, `'browser'`, or `'unknown'`.
<details><summary>Answer</summary>

```js
function getRuntime() {
  if (typeof process !== 'undefined' && process.versions?.node) return 'node';
  if (typeof window !== 'undefined' && typeof document !== 'undefined') return 'browser';
  return 'unknown';
}
```
Checking `process.versions.node` rather than just `process` avoids false positives from bundler shims.
</details>

**20.** Code — Write `isStrictMode()` — a function that reports whether it is running in strict mode.
<details><summary>Answer</summary>

```js
function isStrictMode() {
  return (function () { return this; })() === undefined;
}
```
The inner function inherits the strictness of its enclosing code. In sloppy mode `this` becomes the global object; in strict mode it stays `undefined`.
</details>

**21.** Scenario — Your team wants to use a Stage 2 proposal in production code because it "reads nicer." What do you say?
<details><summary>Answer</summary>Push back. Stage 2 semantics can still change materially, and there's no commitment from engines. If the ergonomics matter enough, write a small helper function that achieves the same thing with stable syntax — you can swap the implementation later without touching call sites.</details>

**22.** Scenario — A build outputs one bundle for browsers and one for Node, and you're seeing "document is not defined" in the Node bundle. Where do you look first?
<details><summary>Answer</summary>Find the module that touches `document` and trace who imports it. Almost always a shared "utils" file has accreted a DOM dependency. Split it: pure logic in one module, DOM adapters in another, and import the adapter only from browser entry points.</details>

---

## Chapter Summary

- **Language ≠ engine ≠ runtime.** ECMAScript defines syntax and built-ins; V8 executes it; browsers and Node add host APIs.
- TC39 advances features through **Stages 0–4**, releasing a new edition each June. Stage 3 is implementable; Stage 4 is standard.
- Execution is **parse → create scopes → execute with JIT**. Syntax errors fire before anything runs.
- **Strict mode** eliminates accidental globals, makes silent failures throw, and sets `this` to `undefined` in plain calls. Modules and classes are strict automatically.
- Keep business logic **runtime-agnostic**; isolate host APIs at the edges.

---

## Practice Exercises

1. Open your browser console and run `typeof window`, `typeof global`, `typeof globalThis`. Do the same in Node. Explain each result.
2. Write a file that logs `'A'`, then contains a deliberate syntax error, then logs `'B'`. Run it. Explain why neither logs.
3. Create two files with identical content, one with `'use strict'` and one without, each containing `x = 1; console.log(x);`. Run both and explain.
4. Look up one Stage 3 proposal on the [TC39 proposals repo](https://github.com/tc39/proposals) and write a paragraph explaining what problem it solves.
5. Take an existing function in your own code that touches `document` or `fs`, and refactor it so the I/O is passed in as an argument.

---

## Mini Project: A Runtime Report Card

Build a single module, `environment.js`, exporting one function `report()` that returns an object describing the current environment:

```js
{
  runtime: 'browser' | 'node' | 'unknown',
  strictMode: boolean,
  globalName: 'window' | 'global' | 'globalThis',
  features: {
    optionalChaining: boolean,
    topLevelAwait: boolean,
    structuredClone: boolean,
    // add three more of your choosing
  }
}
```

**Requirements:**

- The module must not throw in *any* environment — feature-detect everything.
- Detect features by capability (`typeof structuredClone === 'function'`), never by user-agent sniffing.
- Include a `console.table(report().features)` demo script.
- Run it in a browser and in Node; the same file must work in both.

**Stretch:** add a `warnings` array that flags anything the environment lacks that your code depends on.

---

**Next:** [Chapter 2 — Variables, Types, and Operators →](02-variables-types-operators.md)
