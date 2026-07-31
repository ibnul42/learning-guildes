# Chapter 28 — Performance Optimization

## Learning Objectives

- Measure with `performance.mark`, `PerformanceObserver`, and Core Web Vitals before changing code
- Diagnose long tasks and the loading waterfall
- Fix layout thrashing, oversized bundles, and megamorphic call sites
- Recognise when an optimisation is folklore and disprove it with a benchmark

---

## 28.1 Measure First

Every performance conversation that starts with "I think X is slow" wastes a day. Start with
numbers.

```js
const t0 = performance.now();                 // sub-millisecond, monotonic
doWork();
console.log(performance.now() - t0);          // Date.now() is NOT suitable: coarse, non-monotonic

// Named measurements that appear in the DevTools Performance panel
performance.mark('parse:start');
parse(data);
performance.mark('parse:end');
const m = performance.measure('parse', 'parse:start', 'parse:end');
console.log(m.duration);
```

`PerformanceObserver` collects real-user data rather than your laptop's:

```js
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    report(entry.name, entry.duration, entry.startTime);
  }
}).observe({ entryTypes: ['longtask', 'largest-contentful-paint', 'layout-shift'], buffered: true });
```

`buffered: true` includes entries recorded before your observer existed — essential for LCP, which
usually happens before your analytics code runs.

**Core Web Vitals** are what users and search ranking actually see:

| Metric | Measures | Good | What moves it |
|---|---|---|---|
| **LCP** (Largest Contentful Paint) | Time until the biggest above-fold element renders | ≤ 2.5s | Server response, render-blocking CSS/JS, image size and priority |
| **INP** (Interaction to Next Paint) | Worst-case interaction latency, input to paint | ≤ 200ms | Long tasks, heavy event handlers, layout on every keystroke |
| **CLS** (Cumulative Layout Shift) | Unexpected movement of visible content | ≤ 0.1 | Images without dimensions, injected banners, web fonts |

INP replaced FID in 2024 because FID measured only input *delay* — the queue wait — and so scored
well on pages that felt terrible. INP measures the whole path to the next paint.

**Long tasks** are anything occupying the main thread for over 50ms. During one, nothing responds:
no clicks, no scroll, no paint. INP is largely a story about long tasks.

```js
// Break a long task up rather than optimising it
async function processAll(items) {
  let start = performance.now();
  for (const item of items) {
    process(item);
    if (performance.now() - start > 40) {          // stay under the 50ms threshold
      await (scheduler.yield?.() ?? new Promise(r => setTimeout(r)));
      start = performance.now();
    }
  }
}
```

> 💼 **Interview** — "How would you find out why a page feels slow?" Lead with measurement: RUM
> field data for what users see, then a Performance profile to find where the time goes. Anyone
> who starts naming optimisations hasn't answered the question.

---

## 28.2 Loading

The fastest code is code you don't send. Loading strategy usually beats runtime optimisation by an
order of magnitude.

```html
<!-- Blocks parsing AND execution. Almost never what you want. -->
<script src="app.js"></script>

<!-- Downloads in parallel, executes after parsing, in document order. The default choice. -->
<script src="app.js" defer></script>

<!-- Downloads in parallel, executes the moment it arrives, order NOT guaranteed. -->
<script src="analytics.js" async></script>

<!-- Modules are deferred automatically. -->
<script type="module" src="app.js"></script>
```

Resource hints, in decreasing order of aggressiveness:

```html
<link rel="preload" href="/hero.avif" as="image" fetchpriority="high">
<link rel="preload" href="/f.woff2" as="font" type="font/woff2" crossorigin>
<link rel="modulepreload" href="/chunk.js">
<link rel="preconnect" href="https://api.example.com">      <!-- DNS + TCP + TLS, early -->
<link rel="dns-prefetch" href="https://cdn.example.com">    <!-- DNS only, cheap -->
<link rel="prefetch" href="/next-page.js">                  <!-- idle-time, for a likely next nav -->
```

> ⚠️ **Gotcha** — preloading everything is the same as preloading nothing. Each `preload` competes
> for bandwidth against the resource you actually need first. Chrome warns about preloads unused
> within a few seconds; take the warning seriously.

**Code splitting** turns one big bundle into a small critical path:

```js
// Route-level: the natural split boundary
const Settings = () => import('./routes/Settings.js');

// Interaction-level: load the editor only when someone opens it
button.addEventListener('click', async () => {
  const { openEditor } = await import('./editor.js');       // one network round trip
  openEditor();
}, { once: true });
```

**Tree shaking** removes unused exports, but only under conditions people routinely break:

```js
import { debounce } from 'lodash';        // ❌ pulls the whole library on older setups
import debounce from 'lodash-es/debounce'; // ✅ ESM, one function

// ❌ Side effects at module top level prevent elimination
console.log('module loaded');
window.myGlobal = init();

// ✅ Declare purity in package.json
{ "sideEffects": false }
```

Tree shaking needs static ESM `import`/`export` — CommonJS `require` is dynamic and can't be
analysed. Set a **bundle budget** in CI and fail the build when it's exceeded, because bundle size
only ever regresses by accident.

Images and fonts are usually the bigger win anyway:

```html
<img src="hero.avif" width="1200" height="600" fetchpriority="high" alt="…">
<img src="below.avif" width="400" height="300" loading="lazy" decoding="async" alt="…">
```

Always set `width` and `height` — it reserves space and is the single biggest CLS fix. Use
`font-display: swap` so text is readable before the webfont arrives, and `size-adjust` to reduce
the shift when it does.

---

## 28.3 Runtime: The Costs That Are Real

### Layout thrashing

Reading a layout property after a write forces the browser to recalculate layout synchronously.
Alternating them in a loop makes it happen N times.

```js
// ❌ Forced synchronous layout on every iteration
for (const el of items) {
  el.style.height = `${el.offsetHeight * 2}px`;    // write, then read, then write…
}

// ✅ Batch: all reads, then all writes
const heights = items.map(el => el.offsetHeight);   // one layout
items.forEach((el, i) => { el.style.height = `${heights[i] * 2}px`; });
```

The read-triggering properties are worth memorising: `offsetTop/Left/Width/Height`,
`scrollTop/Left/Width/Height`, `clientTop/Left/Width/Height`, `getBoundingClientRect()`,
`getComputedStyle()`, `focus()`, and `scrollIntoView()`.

Animate only `transform` and `opacity`. They're composited, skipping layout and paint entirely.
Animating `width`, `top`, or `margin` re-runs layout every frame.

### Rendering long lists

```js
// ❌ 10,000 DOM nodes: slow to create, slow to lay out, huge memory
items.forEach(item => list.append(renderRow(item)));

// ✅ Render only what's visible
// Or, for a fixed-height list, the cheapest possible win:
```

```css
.row { content-visibility: auto; contain-intrinsic-size: auto 48px; }
```

`content-visibility: auto` skips rendering work for off-screen elements with no JavaScript at all.
`contain-intrinsic-size` supplies a placeholder size so scrollbars behave. For very large or
variable-height lists, a real virtualiser (rendering a window of rows plus a spacer) is still the
answer.

### Object shape stability

V8 assigns each object a hidden class and optimises property access against it. Objects created
with the same shape share it; adding properties in different orders creates different shapes.

```js
// ✅ One shape — monomorphic access sites, inline-cached
function makePoint(x, y) { return { x, y }; }

// ❌ Two shapes for logically identical objects
const a = { x: 1, y: 2 };
const b = { y: 2, x: 1 };        // different hidden class

// ❌ Shape transition after creation
const c = { x: 1 };
c.y = 2;                          // new hidden class
```

A call site seeing 1 shape is **monomorphic** (fast), 2–4 is **polymorphic** (slower), 5+ is
**megamorphic** — the inline cache is abandoned and every access becomes a hash lookup.

```js
// ❌ Megamorphic: called with many differently-shaped objects
function getId(o) { return o.id; }

// ✅ Normalise at the boundary so the hot path sees one shape
const normalise = ({ id, name }) => ({ id, name });
```

This matters in hot loops running thousands of times per frame, and nowhere else. Do not
restructure application code for hidden classes without a profile.

### Algorithms beat micro-optimisation

```js
// ❌ O(n²) — 10,000 items = 100,000,000 comparisons
const dupes = a.filter(x => b.includes(x));

// ✅ O(n + m)
const set = new Set(b);
const dupes = a.filter(x => set.has(x));
```

No amount of loop tuning recovers a quadratic algorithm. Look at complexity before anything else.

### Web Workers for CPU work

Anything taking more than ~50ms and not touching the DOM belongs off the main thread: parsing,
diffing, image processing, compression, crypto, search indexing. See Chapter 25.

---

## 28.4 Premature Optimization, With Evidence

A lot of received wisdom about JavaScript performance was true in 2010 and hasn't been since.
Here's how to check, rather than argue:

```js
function bench(label, fn, iterations = 1000) {
  fn();                                  // warm up: let the JIT optimise before measuring
  const samples = [];
  for (let i = 0; i < 10; i++) {
    const t0 = performance.now();
    for (let j = 0; j < iterations; j++) fn();
    samples.push(performance.now() - t0);
  }
  samples.sort((a, b) => a - b);
  console.log(`${label}: median ${samples[5].toFixed(2)}ms`);   // median, not mean: outlier-robust
}

const arr = Array.from({ length: 10_000 }, (_, i) => i);

bench('for loop',   () => { let s = 0; for (let i = 0; i < arr.length; i++) s += arr[i]; return s; });
bench('for-of',     () => { let s = 0; for (const n of arr) s += n; return s; });
bench('reduce',     () => arr.reduce((s, n) => s + n, 0));
bench('cached len', () => { let s = 0; for (let i = 0, n = arr.length; i < n; i++) s += arr[i]; return s; });
```

Run it. On any current engine, "cache the array length" — advice repeated for fifteen years — is
indistinguishable from not caching it, because the JIT hoists the bounds check itself. `reduce`
carries a real but small callback overhead that disappears entirely against any actual work in the
loop body. The differences here are single-digit milliseconds over ten thousand iterations; a
single unnecessary layout costs more.

Claims worth disbelieving until measured in *your* code:

| Folklore | Reality |
|---|---|
| `for` is much faster than `forEach`/`map` | Measurable only in microbenchmarks with empty bodies |
| Cache `array.length` in loops | The JIT does it |
| `++i` beats `i++` | Identical output |
| `delete` is fine for cleanup | It's slow *and* deoptimises the object (Chapter 27) |
| `try/catch` prevents optimisation | Untrue since TurboFan (2016) |
| String concatenation needs an array join | Engines use rope representations; `+=` is fine |
| Bitwise tricks beat `Math.floor` | Same speed, and wrong above 2³¹ |

What *is* consistently worth doing: send less JavaScript, avoid unnecessary layout, don't block
the main thread, and pick the right data structure. Everything else needs a profile.

> ⚠️ **Gotcha** — microbenchmarks lie in both directions. An empty loop body lets the JIT
> eliminate the loop entirely; a benchmark run once measures compilation, not execution. Warm up,
> take multiple samples, use the median, and return a value so your work isn't optimised away.

---

## 28.5 Common Mistakes and Best Practices

**Mistake: optimising without a profile.** You will pick the wrong thing. Everyone does.

**Mistake: benchmarking a cold function once.** You measured parse and compile time.

**Mistake: reading and writing layout alternately.** N forced reflows instead of one.

**Mistake: animating `width`, `height`, `top`, or `left`.** Layout every frame. Use `transform`.

**Mistake: images without `width` and `height`.** The largest single contributor to CLS.

**Mistake: `preload`ing a dozen resources.** They compete with each other and with LCP.

**Mistake: shipping the whole library for one function.** Import the submodule or use a smaller
dependency; check the cost on bundlephobia before adding one.

**Mistake: judging performance on your own machine.** Throttle to 4× slower CPU and Fast 3G in
DevTools, and check field data, because the median user's device is far slower than yours.

**Best practice: set a performance budget in CI.** Bundle size, LCP, and INP thresholds that fail
the build. Regressions arrive one 12KB dependency at a time.

**Best practice: fix the loading waterfall before the JavaScript.** A 200KB render-blocking
stylesheet costs more than any function you'll rewrite.

**Best practice: measure in the field, not just the lab.** Lighthouse tells you what's possible;
RUM tells you what's happening.

---

## 28.6 Real-World Example: Instrumenting and Fixing an Interaction

A filter input on a 5,000-row table has an INP of 850ms. Here's the full loop: measure, find the
cause, fix, verify.

```js
// ── Step 1: instrument. Attribute the cost to a phase, don't guess. ─────────────────
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration < 200) continue;
    console.log({
      interaction: entry.name,
      total: entry.duration,
      inputDelay: entry.processingStart - entry.startTime,       // queued behind a long task
      processing: entry.processingEnd - entry.processingStart,   // our handler
      presentation: entry.startTime + entry.duration - entry.processingEnd,  // render
    });
  }
}).observe({ type: 'event', durationThreshold: 16, buffered: true });

// Reported: inputDelay 40ms, processing 120ms, presentation 690ms.
// The handler isn't the problem. Rendering is.

// ── Step 2: the original implementation ─────────────────────────────────────────────
input.addEventListener('input', (e) => {
  const q = e.target.value.toLowerCase();
  const matches = rows.filter(r =>
    r.name.toLowerCase().includes(q) || r.email.toLowerCase().includes(q)   // re-lowercased
  );                                                                        // on every keystroke
  tbody.innerHTML = matches.map(renderRow).join('');   // 5,000 nodes destroyed and rebuilt
});

// ── Step 3: the fix ─────────────────────────────────────────────────────────────────
const VISIBLE = 50;

// Precompute the search key ONCE, not per keystroke. This is the algorithmic win.
const indexed = rows.map(r => ({ ...r, key: `${r.name} ${r.email}`.toLowerCase() }));

// Reuse row elements instead of recreating them: no allocation, no style recalculation
// for nodes whose content didn't change.
const pool = Array.from({ length: VISIBLE }, () => {
  const tr = document.createElement('tr');
  tr.innerHTML = '<td class="name"></td><td class="email"></td>';
  tbody.append(tr);
  return { tr, name: tr.querySelector('.name'), email: tr.querySelector('.email') };
});

let pending = null;

function scheduleRender(query) {
  pending = query;
  // Coalesce: many keystrokes, at most one render per frame.
  requestAnimationFrame(() => {
    if (pending === null) return;
    const q = pending;
    pending = null;

    const matches = [];
    for (const row of indexed) {                  // early exit: stop at VISIBLE, don't filter all
      if (row.key.includes(q)) matches.push(row);
      if (matches.length === VISIBLE) break;
    }

    for (let i = 0; i < pool.length; i++) {
      const slot = pool[i];
      const row = matches[i];
      if (!row) { slot.tr.hidden = true; continue; }
      slot.tr.hidden = false;
      // Assign only when changed: writing identical textContent still dirties style.
      if (slot.name.textContent !== row.name) slot.name.textContent = row.name;
      if (slot.email.textContent !== row.email) slot.email.textContent = row.email;
    }

    count.textContent = `${matches.length}${matches.length === VISIBLE ? '+' : ''} results`;
  });
}

input.addEventListener('input', (e) => scheduleRender(e.target.value.toLowerCase()));

// Result: inputDelay 8ms, processing 3ms, presentation 22ms. INP 33ms.
```

The largest single win here is not the element pool or the rAF coalescing — it's the early `break`
at 50 matches. Filtering all 5,000 rows to render 50 of them was doing 100× the necessary work,
and no amount of DOM tuning would have recovered that. Precomputing the lowercase search key is
the second: it moves 10,000 `toLowerCase()` calls per keystroke to 10,000 calls once. The pool and
the frame coalescing are what take it from good to imperceptible, and they were worth doing only
after the algorithm was right.

---

## 28.7 Questions

**1.** MCQ — A long task is one exceeding: a) 16ms b) 50ms c) 100ms d) 200ms
<details><summary>Answer</summary>**b)** 50ms, per the Long Tasks API.</details>

**2.** MCQ — Which metric replaced FID? a) LCP b) CLS c) INP d) TTFB
<details><summary>Answer</summary>**c)** INP measures the whole interaction, not just input delay.</details>

**3.** MCQ — `defer` scripts execute: a) immediately on arrival b) after HTML parsing, in order c) before parsing d) never
<details><summary>Answer</summary>**b)**</details>

**4.** MCQ — Which properties are composited without layout or paint? a) `width`/`height` b) `top`/`left` c) `transform`/`opacity` d) `margin`/`padding`
<details><summary>Answer</summary>**c)**</details>

**5.** MCQ — A call site seeing 5+ object shapes is: a) monomorphic b) polymorphic c) megamorphic d) optimised
<details><summary>Answer</summary>**c)** The inline cache is discarded.</details>

**6.** MCQ — The biggest single CLS fix is: a) `font-display: swap` b) `width`/`height` on images c) `content-visibility` d) preconnect
<details><summary>Answer</summary>**b)**</details>

**7.** Short — Why `performance.now()` rather than `Date.now()` for measurement?
<details><summary>Answer</summary>It's monotonic (unaffected by clock adjustments) and sub-millisecond. `Date.now()` has ~1ms granularity and can jump backwards when the system clock changes.</details>

**8.** Short — What is layout thrashing?
<details><summary>Answer</summary>Interleaving layout reads and writes so the browser must recalculate layout synchronously on each read. Batching all reads before all writes reduces N reflows to one.</details>

**9.** Short — Why does `buffered: true` matter for a `PerformanceObserver`?
<details><summary>Answer</summary>It delivers entries recorded before the observer was created. Without it you'd miss LCP and early long tasks, since analytics code usually loads after they occur.</details>

**10.** Short — Why is `async` wrong for a script other scripts depend on?
<details><summary>Answer</summary>`async` scripts execute in arrival order, not document order, so a dependency may run after its dependent. Use `defer`, which preserves order.</details>

**11.** Short — What breaks tree shaking?
<details><summary>Answer</summary>Dynamic `require`, top-level side effects the bundler can't prove are safe to drop, and a missing or wrong `sideEffects` field in `package.json`.</details>

**12.** Short — Why throttle CPU to 4× in DevTools?
<details><summary>Answer</summary>Your development machine is far faster than the median user's phone. Untested on slow hardware, an interaction that feels instant to you can exceed 500ms in the field.</details>

**13.** Output — How many forced reflows?
```js
for (let i = 0; i < 3; i++) {
  boxes[i].style.width = boxes[i].offsetWidth + 10 + 'px';
}
```
<details><summary>Answer</summary>Three — each read follows a write from the previous iteration, so each `offsetWidth` forces a synchronous layout. Batching the reads makes it one.</details>

**14.** Output — Which is faster, and by how much?
```js
const a = [...Array(1e4).keys()];
const b = [...Array(1e4).keys()];
a.filter(x => b.includes(x)).length;
const s = new Set(b); a.filter(x => s.has(x)).length;
```
<details><summary>Answer</summary>The `Set` version, by roughly three orders of magnitude. The first is O(n·m) — 100 million comparisons; the second is O(n + m).</details>

**15.** Output — What does this benchmark actually measure?
```js
const t0 = performance.now();
expensiveFn();
console.log(performance.now() - t0);
```
<details><summary>Answer</summary>Mostly parse and compile time plus interpretation of the unoptimised bytecode, since the JIT hasn't had a chance to optimise. Warm up, then sample repeatedly.</details>

**16.** Debug — LCP is 4.2s, yet the server responds in 80ms and the hero image is 40KB.
<details><summary>Answer</summary>Something render-blocking sits ahead of it — most likely a synchronous stylesheet or a `<script>` without `defer` in `<head>`, or the image is discovered late because it's set via CSS `background-image` or JavaScript. Inline critical CSS, defer the rest, and make the hero a real `<img>` with `fetchpriority="high"`.</details>

**17.** Debug — INP is 600ms on a button that only sets a boolean.
<details><summary>Answer</summary>The cost isn't in the handler. Either input delay (the click queued behind a long task, likely third-party or hydration work) or presentation delay (the state change triggers a full re-render). Read the `event` entry's three phases to tell which, then break up the long task or narrow the render.</details>

**18.** Debug — A page scores 98 in Lighthouse but users report it as slow.
<details><summary>Answer</summary>Lighthouse is a single lab run on one simulated device and typically on a warm cache. Field data will show a different distribution — check the p75 of LCP and INP from RUM, segmented by device class and connection. Slow real users are usually on hardware Lighthouse never simulated.</details>

**19.** Debug — Scrolling a 3,000-row table janks at 12fps.
<details><summary>Answer</summary>Too many nodes with too much per-node style work. First try `content-visibility: auto` with `contain-intrinsic-size`, which is one CSS line. If rows are variable-height or the count grows further, virtualise so only a window of rows exists in the DOM. Also check for a non-passive scroll listener doing layout reads.</details>

**20.** Debug — A bundle grew from 180KB to 340KB after one small feature merged.
<details><summary>Answer</summary>A dependency was added that pulls a large transitive tree, or an import defeated tree shaking (`import { x } from 'lib'` on a CommonJS build, or a wildcard re-export barrel file). Run the bundler's analyser to see what's in the new bytes, then import the specific submodule or replace the dependency. Add a size budget to CI so the next one fails the build instead of shipping.</details>

**21.** Debug — An animation is smooth in isolation and janky once a data fetch resolves.
<details><summary>Answer</summary>Parsing and rendering the response runs a long task on the main thread, so frames are dropped even though the animation itself is cheap. Move parsing into a worker, and if the animation is `transform`/`opacity` only, it will keep running on the compositor while the main thread is busy.</details>

**22.** Code — Write a `bench(label, fn)` that warms up and reports a median.
<details><summary>Answer</summary>

```js
export function bench(label, fn, { iterations = 1000, samples = 11 } = {}) {
  for (let i = 0; i < 100; i++) fn();                 // warm up: let the JIT settle
  const times = [];
  for (let s = 0; s < samples; s++) {
    const t0 = performance.now();
    let sink;
    for (let i = 0; i < iterations; i++) sink = fn();  // keep the result so it isn't elided
    times.push(performance.now() - t0);
    if (sink === Symbol.for('never')) throw new Error('unreachable');
  }
  times.sort((a, b) => a - b);
  const median = times[Math.floor(times.length / 2)];
  console.log(`${label}: ${median.toFixed(3)}ms / ${iterations} iters`);
  return median;
}
```
</details>

**23.** Code — Write `batchLayout(reads, writes)` that runs all reads then all writes in one frame.
<details><summary>Answer</summary>

```js
function batchLayout(reads, writes) {
  return new Promise((resolve) => {
    requestAnimationFrame(() => {
      const values = reads.map(fn => fn());   // one layout pass for all reads
      requestAnimationFrame(() => {           // writes next frame: nothing can read between
        writes.forEach((fn, i) => fn(values[i]));
        resolve(values);
      });
    });
  });
}
```
</details>

**24.** Code — Report INP-style phase breakdown for slow interactions.
<details><summary>Answer</summary>

```js
new PerformanceObserver((list) => {
  for (const e of list.getEntries()) {
    if (e.duration < 200) continue;
    report({
      name: e.name,
      total: e.duration,
      inputDelay: e.processingStart - e.startTime,
      processing: e.processingEnd - e.processingStart,
      presentation: e.startTime + e.duration - e.processingEnd,
    });
  }
}).observe({ type: 'event', durationThreshold: 16, buffered: true });
```
</details>

**25.** Scenario — Product wants a 2MB charting library on the landing page. How do you handle it?
<details><summary>Answer</summary>Don't put it in the critical path: load it on interaction or when the chart scrolls into view, so LCP and INP are unaffected for the majority who never reach it. Render a lightweight static placeholder — a server-generated SVG or an image — so there's no layout shift and the content is visible immediately. Check whether a smaller library covers the two chart types actually needed, since 2MB usually buys 40 chart types for a page that needs one. If the chart genuinely is the landing page's primary content, then it belongs in the critical path and the conversation is about which library, not whether to defer.</details>

**26.** Scenario — A team wants to rewrite hot loops to use `for` instead of `map`/`filter` across the codebase for speed.
<details><summary>Answer</summary>Ask for the profile first: outside microbenchmarks the difference is a rounding error against any real loop body, and the change costs readability across every file it touches. Where a profile does implicate array iteration, the win is nearly always removing a pass (three chained `filter`/`map` calls becoming one `reduce`) or fixing complexity, not the loop construct. Spend the same effort on the loading waterfall and long tasks, where the numbers are in hundreds of milliseconds rather than single-digit ones.</details>

---

## Chapter Summary

- Measure before you change anything: `performance.mark` in the lab, `PerformanceObserver` with `buffered: true` in the field.
- LCP is a loading problem, INP is a long-task problem, CLS is mostly missing image dimensions.
- Loading strategy — `defer`, code splitting, right-sized images — beats runtime tuning by an order of magnitude.
- Batch layout reads before writes, and animate only `transform` and `opacity`.
- Algorithmic complexity and node count dominate; hidden classes matter only in profiled hot loops.
- Most JavaScript performance folklore is a decade out of date — write the benchmark instead of arguing.

---

## Practice Exercises

1. Add `performance.mark`/`measure` around three phases of your app's startup and read them in the Performance panel.
2. Reproduce layout thrashing over 500 elements, measure it, then batch the reads and measure again.
3. Run the §28.4 benchmark yourself and report which folklore claims survive on your engine.
4. Take a page to LCP under 2.5s on Fast 3G with 4× CPU throttling, documenting each change and its effect.
5. Add a bundle-size budget to a build and deliberately break it with an import.

---

## Mini Project: A Performance Regression Harness

Build a script that fails CI when a page's performance regresses.

- Launch the page headless, three runs, and take the median of LCP, CLS, and total JS bytes
- Simulate a mid-tier device: 4× CPU throttle and Fast 3G
- Store a baseline in the repo and compare against it with an explicit tolerance per metric
- Exit non-zero with a readable diff naming the metric, baseline, current value, and delta
- Include a `--update-baseline` flag, and require the baseline change to appear in the diff of a PR
- Emit a markdown summary suitable for a PR comment

---

**Next:** [Chapter 29 — Functional Programming →](29-functional-programming.md)
