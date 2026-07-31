# Chapter 27 — Memory and Garbage Collection

## Learning Objectives

- Explain reachability and why it, not reference counting, defines garbage in JavaScript
- Describe generational mark-and-sweep collection and what it costs
- Reproduce and fix the four classic leak sources
- Use `WeakMap`, `WeakRef`, and `FinalizationRegistry` for what they're actually for

---

## 27.1 Stack, Heap, and Reachability

Primitives and references live on the **stack**, in the frame of the function that declared them.
Objects, arrays, functions, and closures live on the **heap**; the stack holds a pointer.

```js
function f() {
  let n = 42;                 // stack: the value itself
  let o = { a: 1 };           // stack: a pointer  →  heap: { a: 1 }
  let g = () => n;            // heap: a function object, plus a scope holding n
}                             // frame popped; heap objects now depend on reachability
```

JavaScript has no `free`. Memory is reclaimed when a value becomes **unreachable** — when no chain
of references leads to it from a **GC root**. The roots are:

- The global object (`globalThis`)
- Every active execution context on the call stack, including its scope chain
- Live registers and internal engine references

```js
let a = { name: 'a' };
let b = { name: 'b' };
a.next = b;
b.prev = a;                   // a cycle

a = null;
b = null;                     // both unreachable from any root → both collectable
```

This is why reference counting is the wrong model. Under reference counting, `a` and `b` above
each still have a count of 1, so neither would ever be freed. Reachability handles cycles for
free. (Pre-IE9 leaked on DOM/JS cycles because that boundary used reference counting, which is
where the folklore comes from.)

> ⚠️ **Gotcha** — "unreachable" is not "out of scope". A function returning does not free its
> objects if a closure, listener, array, or `Map` still points at them.

---

## 27.2 How the Collector Works

**Mark-and-sweep** is the base algorithm:

1. **Mark** — walk the object graph from every root, flagging everything reached.
2. **Sweep** — free every heap object that wasn't flagged.
3. Optionally **compact** — move survivors together so allocation stays fast.

V8 splits the heap by age, because most objects die young:

| Space | Contents | Collector | Cost |
|---|---|---|---|
| **New space** (young generation) | Fresh allocations, a few MB | Scavenger (Minor GC) | Sub-millisecond, very frequent |
| **Old space** (old generation) | Objects that survived two scavenges | Mark-Compact (Major GC) | Milliseconds, infrequent |

The scavenger uses semi-spaces: allocate into "from", copy survivors to "to", swap. Because it
only touches live objects, its cost is proportional to what *survived*, not to what was
allocated. Allocating a million short-lived objects is genuinely cheap; keeping a million alive
is not.

Major GC in modern V8 marks concurrently on background threads and splits main-thread work into
small slices, so pauses are far smaller than the classic "stop the world" description — but they
are still real.

```js
// Node: run with --expose-gc
performance.mark('before');
global.gc?.();
performance.mark('after');
console.log(performance.measure('gc', 'before', 'after').duration);
```

Two practical consequences:

**Allocation rate matters more than allocation count.** Churning objects inside a hot loop
triggers frequent scavenges. Reusing a buffer avoids them.

**Don't write code to please the GC.** Object pooling, `delete` avoidance, and manual nulling are
almost always noise. Fix leaks; ignore the rest until a profile says otherwise.

---

## 27.3 The Four Classic Leaks

### 1. Accidental globals

```js
function process() {
  results = [];             // ❌ no declaration → property of globalThis, never collected
  for (const x of input) results.push(expand(x));
}
```

Strict mode makes this a `ReferenceError`. Modules are strict by default, which is most of why
this leak has become rare.

### 2. Forgotten timers

```js
// ❌ The interval keeps `data` — and everything it references — alive forever.
function startPolling(data) {
  setInterval(() => sync(data), 1000);
}

// ✅
function startPolling(data, { signal }) {
  const id = setInterval(() => sync(data), 1000);
  signal.addEventListener('abort', () => clearInterval(id), { once: true });
}
```

A pending timer is a GC root for its callback, so the callback's entire closure is retained.

### 3. Detached DOM nodes

```js
// ❌ The element is removed from the document but the array still points at it.
const cache = [];
function addRow(id) {
  const row = document.createElement('div');
  cache.push(row);                    // strong reference
  list.append(row);
}
list.innerHTML = '';                  // detached from the tree, NOT from memory
```

A detached node retains its whole subtree, including any listeners and their closures. Chrome
DevTools → Memory → heap snapshot → filter "Detached" lists them directly.

```js
// ✅ Don't keep node references; key data weakly instead.
const meta = new WeakMap();           // node → data; the entry dies with the node
meta.set(row, { id });
```

### 4. Unremoved listeners

```js
// ❌ Each instance adds a listener whose closure retains the whole component
class View {
  constructor() {
    this.data = new Array(1e6);
    window.addEventListener('resize', () => this.relayout());
  }
}

// ✅ One controller, one teardown
class View {
  #ac = new AbortController();
  constructor() {
    this.data = new Array(1e6);
    window.addEventListener('resize', () => this.relayout(), { signal: this.#ac.signal });
  }
  destroy() { this.#ac.abort(); }
}
```

`window` and `document` outlive every component, so a listener on them pins whatever it captures.
Listeners on a node being removed *are* collected with the node — but only if nothing else
references that node.

---

## 27.4 Closures Retain More Than You Think

An engine keeps a whole scope object if *any* closure over it survives. Two functions declared in
the same scope share that scope.

```js
function make() {
  const huge = new Array(1e7).fill('x');    // ~80MB
  const small = 42;

  return {
    getSmall: () => small,                  // this alone can retain `huge`
    getHuge:  () => huge.length,
  };
}
const api = make();
delete api.getHuge;                          // `huge` may STILL be retained
```

V8 does optimise this, dropping variables it can prove no surviving closure uses. "Prove" is the
catch: `eval`, `with`, and an open debugger defeat the analysis, and one reference anywhere in the
scope keeps the variable.

```js
// ✅ Explicit: extract what you need, let the rest go.
function make() {
  const huge = new Array(1e7).fill('x');
  const length = huge.length;                // copy out the primitive
  return { getLength: () => length };        // nothing closes over `huge`
}
```

The same applies to `catch` blocks. Storing caught error objects in a long-lived array retains
each stack, and each frame in a stack retains its scope.

---

## 27.5 Weak References

| | Keys | Values | Iterable | Size |
|---|---|---|---|---|
| `Map` | Strong, any type | Strong | ✅ | `.size` |
| `WeakMap` | **Weak**, objects/symbols only | Strong while the key lives | ❌ | ❌ |
| `Set` | Strong | — | ✅ | `.size` |
| `WeakSet` | **Weak**, objects only | — | ❌ | ❌ |

A `WeakMap` entry disappears when its key is collected, making it the right tool for attaching
data to objects you don't own:

```js
const handlers = new WeakMap();          // element → handler set
handlers.set(el, new Set([fn]));
// el removed from the DOM and dereferenced → entry gone, no cleanup code needed
```

`WeakMap` is deliberately not iterable and has no `size`. Exposing either would make GC timing
observable, which the spec forbids.

**`WeakRef`** holds a weak reference to a single object:

```js
const ref = new WeakRef(bigObject);
// …later…
const obj = ref.deref();                  // the object, or undefined if collected
if (obj) use(obj);
```

**`FinalizationRegistry`** runs a callback after an object is collected:

```js
const registry = new FinalizationRegistry((heldValue) => {
  console.log('collected:', heldValue);   // heldValue must NOT reference the target
});
registry.register(obj, 'obj-id-42', obj); // (target, heldValue, unregisterToken)
registry.unregister(obj);
```

> ⚠️ **Gotcha** — `FinalizationRegistry` is not a destructor. The spec guarantees nothing: the
> callback may never run, may run much later, and won't run if the program exits first. Never put
> required cleanup — closing a file, releasing a lock, flushing a write — in one. Use explicit
> `dispose()`/`close()` methods, or `using` declarations where available.

Both `WeakRef` and `FinalizationRegistry` exist for caches and diagnostics, and their own spec
proposal warns against reaching for them. If `WeakMap` solves the problem, use `WeakMap`.

---

## 27.6 Measuring

Guessing is worse than not looking. Three tools, in the order to reach for them:

**Heap snapshot comparison.** DevTools → Memory → take a snapshot, do the suspect action, take
another, switch the view to "Comparison", sort by Delta. Anything that grows and never shrinks is
a candidate. Select a row and read the **Retainers** pane bottom-up: it names the exact chain from
a GC root to the object, which is the real answer to "why is this still alive".

**The three-snapshot technique** for a repeating action: snapshot, perform it N times, snapshot,
perform it N more times, snapshot. Compare 1→3 and 3→5. A leak grows linearly across both
intervals; a warming cache grows once and then flattens.

**Performance panel with the Memory checkbox.** The JS heap should look like a sawtooth —
allocate, collect, allocate. A staircase that never returns to its previous floor is a leak.

**Allocation sampling** shows *where* allocations happen, by stack. Use it for allocation-rate
problems (GC pressure), not retention problems.

In Node:

```js
process.memoryUsage();
// { rss, heapTotal, heapUsed, external, arrayBuffers } — bytes
require('node:v8').writeHeapSnapshot('./heap.heapsnapshot');   // open in DevTools
```

In the browser, with cross-origin isolation:

```js
const { bytes, breakdown } = await performance.measureUserAgentSpecificMemory();
```

---

## 27.7 Common Mistakes and Best Practices

**Mistake: assuming out of scope means collected.** Reachability from a root is the only test.

**Mistake: caching in a `Map` keyed by DOM node or component instance.** Entries outlive their
keys forever. `WeakMap` exists for exactly this.

**Mistake: unbounded caches.** A `Map` that only ever grows is a leak with good intentions. Give
every cache a size or TTL bound.

**Mistake: leaving timers and listeners behind on teardown.** The two most common real SPA leaks,
both fixed by one `AbortController` per component lifetime.

**Mistake: putting required cleanup in a `FinalizationRegistry`.** It may never run.

**Mistake: storing error objects in a long-lived array.** Log `err.message` and `err.stack` as
strings; keeping the object retains every captured frame's scope.

**Mistake: `delete obj.prop` to "free memory".** It frees nothing the value wasn't already
holding, and it forces the object out of its hidden class into dictionary mode, making every
later access slower.

**Best practice: give every resource-owning object a `destroy()`.** Make teardown explicit and
symmetrical with setup — one `AbortController`, aborted in one place.

**Best practice: profile before optimising allocation.** Short-lived garbage is nearly free in a
generational collector. Retention is what costs you.

**Best practice: bound anything that accumulates.** Logs, caches, undo stacks, event histories.
"It's only a few KB per item" is how a 2GB heap starts.

---

## 27.8 Real-World Example: A Self-Evicting Cache

A cache holding large parsed results shouldn't force them to stay in memory when nothing else
wants them — but it also shouldn't drop hot entries. This one keeps a bounded number of strong
references so recent entries survive, demotes older ones to weak references so the collector can
reclaim them, and uses a `FinalizationRegistry` purely to stop its own bookkeeping from growing.

```js
export class SoftCache {
  #strong = new Map();                  // key → value; insertion order is LRU order
  #weak = new Map();                    // key → WeakRef(value), for demoted entries
  #maxStrong;
  #registry;

  constructor({ maxStrong = 50 } = {}) {
    this.#maxStrong = maxStrong;

    // Fires (maybe) after a weakly-held value is collected. Its only job is deleting the dead
    // WeakRef so #weak doesn't accumulate empties. The held value is the KEY, a string — holding
    // the value here would keep it alive and defeat the entire point.
    this.#registry = new FinalizationRegistry((key) => {
      const ref = this.#weak.get(key);
      if (ref && ref.deref() === undefined) this.#weak.delete(key);
    });
  }

  get(key) {
    if (this.#strong.has(key)) {
      const value = this.#strong.get(key);
      this.#strong.delete(key);
      this.#strong.set(key, value);     // re-insert: now the most recently used
      return value;
    }

    const ref = this.#weak.get(key);
    const revived = ref?.deref();
    if (revived === undefined) {
      if (ref) this.#weak.delete(key);  // collected — clean up now, don't wait for the registry
      return undefined;
    }

    this.#weak.delete(key);
    this.#promote(key, revived);        // still wanted, so make it strong again
    return revived;
  }

  set(key, value) {
    if (value === null || typeof value !== 'object') {
      throw new TypeError('SoftCache holds objects only — primitives cannot be weakly held');
    }
    this.#weak.delete(key);
    this.#strong.delete(key);
    this.#promote(key, value);
    return this;
  }

  #promote(key, value) {
    this.#strong.set(key, value);
    while (this.#strong.size > this.#maxStrong) {
      const [oldestKey, oldestValue] = this.#strong.entries().next().value;
      this.#strong.delete(oldestKey);
      // Demote rather than discard: if something else still holds it, a later get() is a hit.
      this.#weak.set(oldestKey, new WeakRef(oldestValue));
      this.#registry.register(oldestValue, oldestKey);
    }
  }

  delete(key) {
    return this.#strong.delete(key) || this.#weak.delete(key);
  }

  /** Drop dead weak entries now. Cheap, and doesn't depend on the registry ever firing. */
  sweep() {
    for (const [key, ref] of this.#weak) {
      if (ref.deref() === undefined) this.#weak.delete(key);
    }
  }

  get stats() {
    return { strong: this.#strong.size, weak: this.#weak.size, max: this.#maxStrong };
  }
}

const parsed = new SoftCache({ maxStrong: 20 });
parsed.set('/api/report/1', bigParsedReport);
parsed.get('/api/report/1');
requestIdleCallback(() => parsed.sweep());
```

The decision worth defending is that `sweep()` exists at all, given the registry is supposed to
handle exactly that. Because finalisation callbacks carry no guarantee of ever running, code that
depends on them passes tests and then grows unboundedly in production. The registry is an
optimisation that usually helps; `sweep()`, called from an idle callback, is the mechanism you can
actually reason about.

---

## 27.9 Questions

**1.** MCQ — JavaScript determines garbage by: a) reference counting b) reachability from roots c) scope exit d) explicit `free`
<details><summary>Answer</summary>**b)**</details>

**2.** MCQ — Two objects referencing only each other are: a) leaked forever b) collectable c) promoted to old space d) frozen
<details><summary>Answer</summary>**b)** The cycle is unreachable from any root, so mark-and-sweep collects it.</details>

**3.** MCQ — A `WeakMap` key must be: a) a string b) any value c) an object or symbol d) a number
<details><summary>Answer</summary>**c)** Non-registered symbols were permitted from ES2023.</details>

**4.** MCQ — `WeakMap` has no `size` because: a) it would be slow b) GC timing would become observable c) an oversight d) it's unordered
<details><summary>Answer</summary>**b)** Observable GC timing is forbidden by the spec.</details>

**5.** MCQ — Which is guaranteed to run? a) a `FinalizationRegistry` callback b) `deref()` returning `undefined` c) a `finally` block d) a scavenge per allocation
<details><summary>Answer</summary>**c)** Finalisation callbacks carry no guarantee at all.</details>

**6.** MCQ — The scavenger's cost is proportional to: a) objects allocated b) objects that survived c) total heap size d) time elapsed
<details><summary>Answer</summary>**b)** It copies survivors, so dying young is cheap.</details>

**7.** Short — What is a detached DOM node and why does it leak?
<details><summary>Answer</summary>A node removed from the document but still referenced from JavaScript. The reference keeps it and its whole subtree reachable, so nothing inside it can be collected.</details>

**8.** Short — Why does a pending `setInterval` prevent collection?
<details><summary>Answer</summary>The scheduled callback is a GC root until cleared, so its closure and everything that closure captured stay reachable.</details>

**9.** Short — How can one closure retain a variable a different closure uses?
<details><summary>Answer</summary>Closures declared in the same scope share one environment record. Unless the engine can prove a variable is unused by every surviving closure, the whole record is retained.</details>

**10.** Short — `WeakRef` versus `WeakMap` — when is each right?
<details><summary>Answer</summary>`WeakMap` when you're associating data with a key object you don't own and want the association to die with the key. `WeakRef` when the *value* is the expensive thing and you want to keep using it while it happens to be alive. Prefer `WeakMap`; it needs no `deref` check and can't be misused.</details>

**11.** Short — Why is `delete obj.prop` a bad optimisation?
<details><summary>Answer</summary>It doesn't release anything a reassignment wouldn't, and it transitions the object out of its hidden class into slower dictionary mode.</details>

**12.** Short — What does the Retainers pane in a heap snapshot tell you?
<details><summary>Answer</summary>The chain of references from a GC root down to the selected object — that is, the specific reason it hasn't been collected, which is the only thing you need to fix a leak.</details>

**13.** Output — Which of these can be collected after the last line?
```js
let a = { big: new Array(1e6) };
const weak = new WeakMap();
weak.set(a, 'meta');
const strong = new Map();
strong.set('k', a);
a = null;
```
<details><summary>Answer</summary>Nothing. `strong` still holds the object under key `'k'`, so both it and the `WeakMap` entry stay alive. Weak keys only help when every *strong* reference is gone.</details>

**14.** Output —
```js
const registry = new FinalizationRegistry(k => console.log('gone', k));
(function () {
  const obj = {};
  registry.register(obj, 'A');
})();
console.log('done');
```
<details><summary>Answer</summary>`done` — and possibly `gone A` at some unpredictable later point, or never. Nothing about the timing is guaranteed.</details>

**15.** Output —
```js
function outer() {
  const big = new Array(1e6).fill(0);
  return function () { return 'nothing to do with big'; };
}
const fn = outer();
```
<details><summary>Answer</summary>No output — the question is whether `big` is retained. In modern V8 it usually isn't, because no surviving closure references it. But this is an optimisation, not a guarantee, so relying on it is unwise.</details>

**16.** Debug — An SPA's heap grows by 8MB per route change and never comes back down.
<details><summary>Answer</summary>Unmounted views left listeners on `window`/`document` and pending timers, both of which are roots retaining each view's closure. Give every view an `AbortController`, pass its signal to every listener and timer cleanup, and abort on unmount.</details>

**17.** Debug — A table that re-renders 1,000 rows every second shows "Detached HTMLTableRowElement × 400,000" in a snapshot.
<details><summary>Answer</summary>Row elements are being pushed into a module-level array (or a `Map`) that's never cleared, so every removed row stays reachable. Store row *data* keyed by id, look elements up from the DOM when needed, or key auxiliary data with a `WeakMap`.</details>

**18.** Debug — A Node service's RSS climbs steadily; the heap snapshot shows a growing array of `Error` objects.
<details><summary>Answer</summary>An error-logging buffer stores the error objects themselves, and each retained stack pins the scopes of every frame it captured. Store the serialised message and stack string, and bound the buffer.</details>

**19.** Debug — A memoised function keyed by argument object never releases memory, even after the arguments are gone.
<details><summary>Answer</summary>The cache is a `Map`, so its keys are strong references. Switch to a `WeakMap` when keys are objects, or add an LRU bound if primitive keys are needed.</details>

**20.** Debug — Memory looks flat in development but a production build leaks. What's the most likely difference?
<details><summary>Answer</summary>Dev tooling — hot module replacement, source-map registries, DevTools being open — changes both retention and the engine's ability to elide unused closure variables. Reproduce with a production build, no DevTools open during the action, and snapshots taken only at the start and end.</details>

**21.** Code — Write a `Disposable` base class that tracks cleanup functions and runs them once.
<details><summary>Answer</summary>

```js
export class Disposable {
  #cleanups = new Set();
  #disposed = false;

  track(cleanup) {
    if (this.#disposed) { cleanup(); return cleanup; }   // late registration: run immediately
    this.#cleanups.add(cleanup);
    return cleanup;
  }

  dispose() {
    if (this.#disposed) return;
    this.#disposed = true;
    for (const fn of this.#cleanups) {
      try { fn(); } catch (err) { queueMicrotask(() => { throw err; }); }
    }
    this.#cleanups.clear();
  }

  [Symbol.dispose]() { this.dispose(); }   // enables `using d = new Disposable()`
}
```
</details>

**22.** Code — Write `weakMemoize(fn)` for single-object-argument functions.
<details><summary>Answer</summary>

```js
function weakMemoize(fn) {
  const cache = new WeakMap();
  return (arg) => {
    if (cache.has(arg)) return cache.get(arg);
    const result = fn(arg);
    cache.set(arg, result);
    return result;               // entry disappears when `arg` is collected
  };
}
```
</details>

**23.** Code — Write `trackLeaks(label, obj)` that logs whether `obj` was ever collected.
<details><summary>Answer</summary>

```js
const seen = new FinalizationRegistry((label) => console.log(`collected: ${label}`));
const alive = new Set();

export function trackLeaks(label, obj) {
  const ref = new WeakRef(obj);
  alive.add({ label, ref });
  seen.register(obj, label);
  return obj;
}

export function report() {
  for (const entry of alive) {
    if (entry.ref.deref() === undefined) alive.delete(entry);
    else console.warn(`still alive: ${entry.label}`);
  }
}
```
Diagnostics only — `report()` may show survivors purely because no GC has run yet.</details>

**24.** Scenario — A long-lived dashboard is fine for an hour and dies after eight. Where do you start?
<details><summary>Answer</summary>Start with the Performance panel's memory line over a long session to distinguish a genuine leak (staircase) from fragmentation or a large steady-state working set (sawtooth with a rising floor is the ambiguous case). Then use the three-snapshot technique around one refresh cycle, since anything that repeats hourly will show a linear delta. Look first at the accumulators — chart data buffers, WebSocket message history, notification lists — because a dashboard that appends and never trims is the common cause. Fix by bounding those structures, not by tuning GC flags.</details>

**25.** Scenario — A colleague proposes object pooling for a particle system to "reduce GC pressure". Do you agree?
<details><summary>Answer</summary>Only with a profile showing scavenges as a real cost, which for particles is plausible — thousands of short-lived objects per frame is one of the few genuine cases. Measure the allocation rate and GC time in the Performance panel first. If it does help, pool only the hot particle objects and keep the pool bounded, because a pool is by construction a set of objects you've made immortal, and getting the reset logic wrong produces state-bleed bugs that are far worse than a 0.3ms scavenge. For anything not in a per-frame loop, the answer is no.</details>

**26.** Scenario — You need to associate metadata with third-party DOM nodes you don't control the lifetime of.
<details><summary>Answer</summary>Use a `WeakMap` keyed by node, so entries vanish when the nodes are removed and dereferenced, with no cleanup hook needed. Avoid `dataset`, which stringifies and mutates markup you don't own, and avoid a `Map`, which would retain every node the third party ever created. If you need to enumerate your entries — which `WeakMap` won't let you do — keep a separate bounded list of `WeakRef`s and sweep it, accepting that the list is best-effort.</details>

---

## Chapter Summary

- Values die when unreachable from a GC root, not when they go out of scope — cycles are fine.
- V8's generational split makes short-lived garbage nearly free; retention is what costs you.
- The four classic leaks are accidental globals, forgotten timers, detached nodes, and unremoved listeners.
- Closures share one scope record, so one survivor can retain everything declared alongside it.
- `WeakMap` is the tool for object-keyed association; `WeakRef` and `FinalizationRegistry` are last resorts.
- Fix leaks by reading the Retainers chain in a heap snapshot, not by guessing or by nulling variables.

---

## Practice Exercises

1. Build a page that leaks 1MB per button click, confirm it with a heap snapshot, then fix it and prove the fix.
2. Reproduce the detached-node leak, find the entries with the "Detached" filter, and identify the retainer.
3. Write a `Map`-based cache and a `WeakMap`-based cache for the same workload and compare heap growth.
4. Measure GC time for `new Array(1e6).fill(0)` in a loop versus reusing one array, using `--expose-gc`.
5. Test whether your engine elides an unreferenced closure variable, using two closures over one large array.

---

## Mini Project: A Leak Detector for Component Trees

Build a development-only utility that warns when a component is unmounted but never collected.

- `track(instance, label)` registers a `WeakRef` and a `FinalizationRegistry` entry
- `markUnmounted(instance)` records the time the component was supposed to die
- A periodic check reports anything still alive N seconds after unmount, with its label
- Force a GC first where possible (`--expose-gc` in Node, the DevTools collect button in Chrome) and explain in the README why the report is advisory without it
- Zero cost in production: the whole module compiles out behind a build flag
- Include a deliberately leaky example component and a fixed one, so users can see both outputs

---

**Next:** [Chapter 28 — Performance Optimization →](28-performance.md)
