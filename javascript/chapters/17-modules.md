# Chapter 17 — Modules

## Learning Objectives

- Use ESM `import`/`export` in all their forms
- Contrast ESM with CommonJS and know when each applies
- Use dynamic import for code splitting
- Understand circular dependencies and module resolution

---

## 17.1 ESM Syntax

```js
// --- Named exports ---
export const PI = 3.14159;
export function area(r) { return PI * r ** 2; }
export class Circle {}

// or collected at the bottom
const a = 1, b = 2;
export { a, b as renamedB };

// --- Default export: one per module ---
export default function main() {}

// --- Re-export (barrel files) ---
export { helper } from './helper.js';
export * from './utils.js';
export * as math from './math.js';
export { default as Button } from './Button.js';
```

```js
// --- Imports ---
import main from './main.js';                    // default
import { a, renamedB as b } from './vals.js';    // named
import main, { a } from './mod.js';              // both
import * as everything from './mod.js';          // namespace object
import './styles.css';                            // side effect only

// Dynamic — returns a promise, works anywhere
const { heavy } = await import('./heavy.js');
```

**Key semantics:**

- Modules are **always strict mode**.
- Top-level `this` is `undefined`.
- Each module is **evaluated once**, then cached — imports are singletons.
- Imports are **hoisted** and resolved before any code runs.
- Bindings are **live**, not copies:

```js
// counter.js
export let count = 0;
export function inc() { count++; }

// main.js
import { count, inc } from './counter.js';
console.log(count);   // 0
inc();
console.log(count);   // 1  ← the binding updated
count = 5;            // ❌ TypeError — imports are read-only
```

That live-binding behaviour is a real difference from CommonJS, where you'd get a snapshot.

### Top-level await

```js
// Legal in ESM only. Blocks the module's consumers until it resolves.
const config = await fetch('/config.json').then(r => r.json());
export default config;
```

Useful, but it delays every importer — don't put a slow network call at a module's top level in a hot path.

---

## 17.2 CommonJS

Node's original system, still everywhere.

```js
// Exporting
module.exports = function main() {};
module.exports.helper = () => {};
exports.other = 1;              // shorthand — but reassigning `exports` breaks it

// Importing — synchronous, runtime, dynamic
const main = require('./main');
const { helper } = require('./main');
if (condition) { const x = require('./maybe'); }   // legal — ESM can't do this statically
```

| | ESM | CommonJS |
|---|---|---|
| Syntax | `import`/`export` | `require`/`module.exports` |
| Timing | static, parse-time | dynamic, runtime |
| Loading | asynchronous | synchronous |
| Bindings | live references | value copies |
| Tree-shaking | ✅ (statically analysable) | ❌ mostly |
| Top-level `await` | ✅ | ❌ |
| Browser native | ✅ | ❌ (needs bundling) |
| `__dirname` | ❌ (use `import.meta.url`) | ✅ |

**Interop:** ESM can `import` CommonJS (the `module.exports` object becomes the default export). CommonJS **cannot** `require()` an ESM module — it must use `await import()`.

> ⚠️ **Gotcha** — named imports from a CommonJS module work by static analysis of the file, not by inspecting the exports object. `module.exports = { a, b }` assigned conditionally, or built in a loop, defeats it — the import then fails at load with "does not provide an export named", despite the property existing at runtime. Import the default and destructure from it.

> 💼 **Interview** — "Why does `require()` of an ESM module throw?" Because ESM resolution and evaluation are asynchronous, and `require` must return synchronously. That single difference is what forced the whole dual-package ecosystem.

In Node, the mode is chosen by:
- `"type": "module"` in `package.json` → `.js` files are ESM
- `.mjs` → always ESM; `.cjs` → always CommonJS

```js
// ESM equivalents of CJS globals
import { fileURLToPath } from 'node:url';
import path from 'node:path';
const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
```

---

## 17.3 Dynamic Import and Code Splitting

```js
// Load only when needed — the bundler creates a separate chunk
button.addEventListener('click', async () => {
  const { Chart } = await import('./chart.js');   // ~200 KB, deferred
  new Chart(canvas).render(data);
});

// Conditional
const locale = navigator.language.startsWith('bn')
  ? await import('./i18n/bn.js')
  : await import('./i18n/en.js');

// Parallel
const [a, b] = await Promise.all([import('./a.js'), import('./b.js')]);

// With error handling and a fallback
let mod;
try { mod = await import('./optional.js'); }
catch { mod = { feature: () => null }; }
```

This is the single highest-leverage performance technique for large front ends: ship a small initial bundle, load the rest on demand.

---

## 17.4 Circular Dependencies

```js
// a.js
import { b } from './b.js';
export const a = 'A';
console.log('a.js sees b as:', b);

// b.js
import { a } from './a.js';
export const b = 'B';
console.log('b.js sees a as:', a);   // ❌ ReferenceError — a is in its TDZ
```

ESM handles the *graph* fine (no infinite loop), but if `b.js` evaluates before `a.js` finishes, `a` is still in the TDZ. CommonJS is more forgiving but gives you a half-populated object instead — arguably worse, because it fails silently.

**Fixes, in order of preference:**

1. Extract the shared piece into a third module both depend on.
2. Move the dependency inside a function so it's read at call time, not evaluation time.
3. Use a dynamic `import()` at the point of use.

Circular dependencies are almost always a design smell. Treat them as a signal to reorganise.

---

## 17.5 Common Mistakes and Best Practices

**Mistake: barrel files everywhere.** `export * from` in a big `index.js` defeats tree-shaking in many bundlers and creates import cycles. Import from the specific file.

**Mistake: side effects at module top level.** A module that starts a timer or mutates the DOM on import is impossible to test and unpredictable in ordering. Export an `init()` instead.

**Mistake: omitting the file extension in native ESM.** Browsers and Node's ESM loader require `./util.js`, not `./util`. Bundlers hide this, then production breaks.

**Mistake: mixing `require` and `import` in one file.** Pick one per file.

**Mistake: default-exporting everything.** Named exports are better for refactoring (rename detection), auto-import, and tree-shaking. Reserve `default` for a module's single obvious subject.

**Best practice: one clear responsibility per module**, and a public surface you'd be comfortable documenting.

---

## 17.6 Real-World Example: A Lazily-Loaded Plugin Architecture

```js
// registry.js — no plugin code is loaded until it's actually used
const REGISTRY = {
  markdown: () => import('./plugins/markdown.js'),
  charts:   () => import('./plugins/charts.js'),
  latex:    () => import('./plugins/latex.js'),
};

const loaded = new Map();

export async function getPlugin(name) {
  if (!REGISTRY[name]) throw new Error(`Unknown plugin: ${name}`);

  // Cache the PROMISE, not the result — concurrent callers share one load
  if (!loaded.has(name)) {
    loaded.set(
      name,
      REGISTRY[name]()
        .then(mod => mod.default ?? mod)
        .catch(err => { loaded.delete(name); throw new Error(`Failed to load ${name}`, { cause: err }); })
    );
  }
  return loaded.get(name);
}

export async function renderWith(names, content) {
  const plugins = await Promise.all(names.map(getPlugin));
  return plugins.reduce((out, p) => p.transform(out), content);
}
```

Caching the promise rather than the resolved module is the important detail: two simultaneous calls to `getPlugin('charts')` trigger one network request, not two.

---

## 17.7 Questions

**1.** MCQ — ESM modules are: a) sloppy mode b) always strict c) configurable d) strict only in Node
<details><summary>Answer</summary>**b)**</details>

**2.** MCQ — How many default exports per module? a) 0 b) 1 c) unlimited d) 2
<details><summary>Answer</summary>**b)** At most one.</details>

**3.** MCQ — `import()` returns: a) the module b) a promise c) undefined d) a namespace
<details><summary>Answer</summary>**b)**</details>

**4.** MCQ — ESM imports are: a) copies b) live bindings c) mutable d) deep clones
<details><summary>Answer</summary>**b)** — and read-only.</details>

**5.** MCQ — CommonJS can `require()` an ESM module: a) yes b) no
<details><summary>Answer</summary>**b)** It must use `await import()`.</details>

**6.** MCQ — Top-level `this` in an ESM module is: a) `globalThis` b) `undefined` c) `module` d) `exports`
<details><summary>Answer</summary>**b)**</details>

**7.** Short — Why can ESM be tree-shaken but CommonJS mostly can't?
<details><summary>Answer</summary>ESM imports/exports are static and analysable at build time; `require()` is a runtime function call with a dynamic argument.</details>

**8.** Short — How many times is a module evaluated?
<details><summary>Answer</summary>Once per resolved specifier, then cached — imports are effectively singletons.</details>

**9.** Short — Name two ESM replacements for CJS globals.
<details><summary>Answer</summary>`import.meta.url` for `__filename`/`__dirname`; `await import()` for conditional `require`.</details>

**10.** Short — When is a default export appropriate?
<details><summary>Answer</summary>When a module has one obvious subject (a React component, a class). Otherwise prefer named exports.</details>

**11.** Short — What breaks with circular imports in ESM?
<details><summary>Answer</summary>Not the graph resolution, but TDZ access — a module may read a `const`/`let` binding from a partner that hasn't finished evaluating.</details>

**12.** Output —
```js
// m.js:  export let n = 0; export const inc = () => n++;
import { n, inc } from './m.js';
inc(); inc();
console.log(n);
```
<details><summary>Answer</summary>`2` — live binding.</details>

**13.** Output — `import { n } from './m.js'; n = 5;`
<details><summary>Answer</summary>`TypeError: Assignment to constant variable` — imported bindings are read-only.</details>

**14.** Output —
```js
console.log('start');
import './side-effect.js';   // logs 'side'
```
<details><summary>Answer</summary>`side` then `start` — imports are hoisted and evaluated first.</details>

**15.** Output — `import * as m from './m.js'; console.log(typeof m, Object.isFrozen(m));`
<details><summary>Answer</summary>`'object' true` — namespace objects are sealed/frozen.</details>

**16.** Debug — "Cannot use import statement outside a module" in Node.
<details><summary>Answer</summary>The file is being treated as CommonJS. Add `"type": "module"` to `package.json` or rename to `.mjs`.</details>

**17.** Debug — Bundle size didn't shrink after switching to named imports from a barrel file.
<details><summary>Answer</summary>`export *` barrels often defeat tree-shaking because the bundler can't prove the re-exported modules are side-effect free. Import from the concrete file, and set `"sideEffects": false` in `package.json`.</details>

**18.** Debug — A module's exported value is `undefined` only in one importer.
<details><summary>Answer</summary>A circular dependency: that importer evaluates first and reads the binding before it's initialized. Extract the shared code into a third module.</details>

**19.** Debug — Native ESM in the browser 404s on `import { x } from './util'`.
<details><summary>Answer</summary>Extensions are mandatory in native ESM. Use `'./util.js'`.</details>

**20.** Code — Convert this CJS module to ESM:
```js
const fs = require('fs');
module.exports = { read: (p) => fs.readFileSync(p, 'utf8') };
```
<details><summary>Answer</summary>

```js
import fs from 'node:fs';
export const read = (p) => fs.readFileSync(p, 'utf8');
```
</details>

**21.** Code — Write a `loadLocale(lang)` that dynamically imports a translation module with an English fallback and caches results.
<details><summary>Answer</summary>

```js
const cache = new Map();
export function loadLocale(lang) {
  if (!cache.has(lang)) {
    cache.set(lang, import(`./locales/${lang}.js`)
      .catch(() => import('./locales/en.js')));
  }
  return cache.get(lang);
}
```
Note: template-literal specifiers need bundler support (`import.meta.glob` in Vite, magic comments in webpack).
</details>

**22.** Code — Write a module exposing a singleton connection that's created lazily on first use, not on import.
<details><summary>Answer</summary>

```js
let conn = null;
export async function getConnection() {
  conn ??= await createConnection(process.env.DB_URL);
  return conn;
}
export async function close() { await conn?.close(); conn = null; }
```
Lazy init keeps the module import side-effect free and testable.
</details>

**23.** Scenario — Your initial JS bundle is 2 MB and Time-to-Interactive is 8 seconds. First move?
<details><summary>Answer</summary>Measure with a bundle analyzer, then route-level code splitting with dynamic `import()` — usually the biggest single win. Follow with deferring third-party libraries (charting, editors, date pickers) behind interaction, and replacing heavyweight deps.</details>

**24.** Scenario — A teammate wants a single `utils/index.js` barrel exporting 80 helpers. Push back or accept?
<details><summary>Answer</summary>Push back. It creates a cycle magnet, hurts tree-shaking, and makes every helper's blast radius the whole app. Import from concrete files; if ergonomics matter, group into a few small domain-specific barrels instead of one giant one.</details>

---

## Chapter Summary

- ESM: static, strict, asynchronous, live read-only bindings, tree-shakeable, top-level `await`.
- CJS: dynamic, synchronous, value copies. ESM can import CJS; CJS needs `await import()` for ESM.
- Modules evaluate once and are cached — imports are singletons.
- Dynamic `import()` enables code splitting; cache the promise, not just the result.
- Circular dependencies signal a design problem; extract shared code.
- Prefer named exports, concrete import paths, and side-effect-free modules.

---

## Practice Exercises

1. Convert a small CJS project to ESM, including `__dirname` usage and a conditional `require`.
2. Create a deliberate circular dependency, observe the TDZ error, then fix it three different ways.
3. Add route-based code splitting to a small SPA and measure the initial bundle before and after.
4. Write a module with zero import-time side effects that still initializes lazily and exactly once.
5. Build a plugin loader like §17.6 and write a test proving concurrent calls trigger one load.

---

**Next:** [Chapter 18 — The Event Loop →](18-event-loop.md)
