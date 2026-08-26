# JavaScript: From Fundamentals to Expert


**A practical, interview-ready guide to modern JavaScript (ES2025+)**

---

## Who This Book Is For

You already know how to program. You understand variables, loops, and functions in *some* language. What you want is JavaScript — the real thing, including the parts that trip people up in interviews and the parts that bite you in production.

This book does not teach you what a `for` loop is. It teaches you why JavaScript's `for...in` iterates prototype-chain keys, why `this` breaks in a callback, why `0.1 + 0.2 !== 0.3`, and what actually happens between `await` and the next line.

## How to Use This Book

Four passes work well:

1. **Linear read** — Chapters 1–33 in order. Each builds on the last.
2. **Guided path** — [Appendix D](appendices/D-roadmap.md) stages the whole book with
   "you're ready to move on when you can…" checkpoints.
3. **Interview prep** — Chapters 8–12 and 18–19, then [Appendix A](appendices/A-interview-questions.md)
   and [B](appendices/B-coding-challenges.md). Appendix D includes a 4-week sprint schedule.
4. **Reference** — [Appendix F](appendices/F-cheatsheet.md) (cheat sheet) and
   [G](appendices/G-glossary.md) (glossary) once you're working.

Every chapter follows the same shape:

- Learning objectives
- Explanations with runnable, commented code
- Common mistakes and best practices
- A real-world example
- 20–28 questions (MCQ, short answer, output prediction, debugging, coding, scenario)
- Chapter summary
- Practice exercises
- Mini project (where the topic supports one)

Run the code. Reading JavaScript and writing JavaScript are different skills, and only one of them gets you hired.

---

## Table of Contents

### Part I — Foundations

| # | Chapter | Core Idea |
|---|---------|-----------|
| 1 | [How JavaScript Works](chapters/01-how-javascript-works.md) | Engines, runtimes, the spec, strict mode |
| 2 | [Variables, Types, and Operators](chapters/02-variables-types-operators.md) | `let`/`const`, primitives, coercion, equality |
| 3 | [Control Flow and Loops](chapters/03-control-flow-loops.md) | Branching, all loop forms, labels |
| 4 | [Functions](chapters/04-functions.md) | Declarations, expressions, arrows, params, IIFE |
| 5 | [Objects](chapters/05-objects.md) | Property descriptors, copying, immutability |
| 6 | [Arrays](chapters/06-arrays.md) | Mutating vs non-mutating, iteration methods |
| 7 | [Strings, Numbers, Math, Dates](chapters/07-strings-numbers-math-dates.md) | Text, floats, `Intl`, `Temporal` |

### Part II — Core Mechanics

| # | Chapter | Core Idea |
|---|---------|-----------|
| 8 | [Scope, Hoisting, and the TDZ](chapters/08-scope-hoisting-tdz.md) | Lexical environments |
| 9 | [Closures](chapters/09-closures.md) | Functions that remember |
| 10 | [`this` and Binding](chapters/10-this-and-binding.md) | Five call rules |
| 11 | [Prototypes and Inheritance](chapters/11-prototypes-inheritance.md) | The delegation chain |
| 12 | [Classes and OOP](chapters/12-classes-oop.md) | Syntax sugar, private fields, composition |
| 13 | [Destructuring, Spread, Rest, and Modern Operators](chapters/13-destructuring-spread-rest.md) | `?.`, `??`, `&&=` |
| 14 | [Symbols, Iterators, and Generators](chapters/14-symbols-iterators-generators.md) | Custom iteration protocols |
| 15 | [Map, Set, WeakMap, WeakSet](chapters/15-maps-sets-weak-collections.md) | Keyed collections |
| 16 | [Error Handling](chapters/16-error-handling.md) | `try`/`catch`, custom errors, `cause` |
| 17 | [Modules](chapters/17-modules.md) | ESM, CJS, dynamic import, bundling |

### Part III — Asynchronous JavaScript

| # | Chapter | Core Idea |
|---|---------|-----------|
| 18 | [The Event Loop](chapters/18-event-loop.md) | Call stack, microtasks, macrotasks |
| 19 | [Callbacks, Promises, async/await](chapters/19-async-javascript.md) | Concurrency in practice |
| 20 | [Fetch, JSON, and HTTP](chapters/20-fetch-json-http.md) | Network, aborts, retries, streams |

### Part IV — The Browser

| # | Chapter | Core Idea |
|---|---------|-----------|
| 21 | [The DOM](chapters/21-dom.md) | Selecting, creating, mutating |
| 22 | [Events](chapters/22-events.md) | Propagation, delegation, custom events |
| 23 | [Forms and Validation](chapters/23-forms-validation.md) | Constraint API, `FormData` |
| 24 | [Timers and Scheduling](chapters/24-timers-scheduling.md) | `setTimeout`, rAF, debounce/throttle |
| 25 | [Storage and Browser APIs](chapters/25-storage-browser-apis.md) | Web Storage, IndexedDB, observers, workers |

### Part V — Advanced and Professional

| # | Chapter | Core Idea |
|---|---------|-----------|
| 26 | [Regular Expressions](chapters/26-regular-expressions.md) | Patterns, groups, lookaround, ReDoS |
| 27 | [Memory and Garbage Collection](chapters/27-memory-garbage-collection.md) | Reachability, leaks, profiling |
| 28 | [Performance Optimization](chapters/28-performance.md) | Measuring, rendering, algorithmic wins |
| 29 | [Functional Programming](chapters/29-functional-programming.md) | Purity, composition, immutability |
| 30 | [Design Patterns](chapters/30-design-patterns.md) | Module, observer, factory, strategy |
| 31 | [Security](chapters/31-security.md) | XSS, CSRF, CSP, prototype pollution |
| 32 | [Testing Fundamentals](chapters/32-testing.md) | Unit, integration, mocking, TDD |
| 33 | [ES6 → ES2025 Reference](chapters/33-es6-to-es2025.md) | Year-by-year, with examples |

### Appendices

| # | Appendix | Contents |
|---|----------|----------|
| A | [340 Interview Questions](appendices/A-interview-questions.md) | By topic and level, each with a follow-up |
| B | [121 Coding Challenges](appendices/B-coding-challenges.md) | Worked solutions, complexity, common wrong answer |
| C | [20 Real-World Projects](appendices/C-projects.md) | Easy → advanced, specs only |
| D | [The JavaScript Roadmap](appendices/D-roadmap.md) | Staged plan, plus a 4-week interview sprint |
| E | [Best Practices Checklist](appendices/E-best-practices.md) | 199 reviewable items |
| F | [Cheat Sheet](appendices/F-cheatsheet.md) | Syntax and method reference, tables throughout |
| G | [Glossary](appendices/G-glossary.md) | 204 terms, cross-referenced to chapters |

---

## Conventions

Code blocks are runnable unless marked otherwise. Output is shown as a trailing comment:

```js
const doubled = [1, 2, 3].map(n => n * 2);
console.log(doubled); // [2, 4, 6]
```

Warnings look like this:

> ⚠️ **Gotcha** — `typeof null` returns `'object'`. This is a bug from 1995 that can never be fixed without breaking the web.

Interview-relevant points are flagged:

> 💼 **Interview** — Expect to be asked to explain the difference between `==` and `===` in your first five minutes.

Questions carry an inline type label and a collapsible answer, so you can self-test:

**1.** MCQ — Which is falsy? a) `'0'` b) `[]` c) `0` d) `{}`
<details><summary>Answer</summary>**c)** Only <code>false</code>, <code>0</code>, <code>-0</code>, <code>0n</code>, <code>''</code>, <code>null</code>, <code>undefined</code>, and <code>NaN</code> are falsy.</details>

Wrong and right examples are marked `// ❌` and `// ✅`.

---

## A Note on Environment

Everything here runs in a modern browser console or in Node.js 22+. Where a feature is browser-only (DOM, `localStorage`) or Node-only (`require`, `fs`), it's labeled. For the newest features (Chapter 33), check [caniuse.com](https://caniuse.com) before shipping to production.

Start with [Chapter 1](chapters/01-how-javascript-works.md).
