# Chapter 18 — The Event Loop

## Learning Objectives

- Describe the call stack, task queue, and microtask queue
- Predict the exact execution order of any mixed sync/async snippet
- Explain starvation, blocking, and rendering interaction
- Distinguish browser and Node.js event loop details

---

## 18.1 The Model

JavaScript has **one thread** and **one call stack**. It cannot do two things at once. Concurrency comes from the runtime, not the language.

```
┌─────────────┐
│  Call Stack │  ← synchronous code runs here, one frame at a time
└──────┬──────┘
       │ when empty ↓
┌──────────────────────────────────────┐
│  MICROTASK QUEUE (drained COMPLETELY)│  ← promises, queueMicrotask,
│                                       │     MutationObserver, await resumption
└──────┬───────────────────────────────┘
       │ then ↓
┌──────────────────────────────────────┐
│  (browser: render if needed)          │
└──────┬───────────────────────────────┘
       │ then ONE ↓
┌──────────────────────────────────────┐
│  MACROTASK / TASK QUEUE               │  ← setTimeout, setInterval, I/O,
│  (exactly one per iteration)          │     events, setImmediate (Node)
└──────────────────────────────────────┘
       │ loop back to top
```

**The algorithm, memorised:**

1. Run all synchronous code until the call stack is empty.
2. Drain the **entire** microtask queue — including microtasks added *while* draining.
3. (Browser) Render if it's time.
4. Take **one** macrotask, run it to completion.
5. Go to step 2.

The asymmetry is the key: **all** microtasks, but only **one** macrotask per turn.

---

## 18.2 Working an Example

```js
console.log('1');

setTimeout(() => console.log('2'), 0);

Promise.resolve().then(() => console.log('3'));

queueMicrotask(() => console.log('4'));

(async () => {
  console.log('5');
  await null;
  console.log('6');
})();

console.log('7');
```

**Output: `1 5 7 3 4 6 2`**

Tracing it:

- `1` — sync.
- `setTimeout` → macrotask queued.
- `.then` → microtask **3** queued.
- `queueMicrotask` → microtask **4** queued.
- The async IIFE runs synchronously up to the `await`, so `5` prints. `await null` suspends and queues the continuation as microtask **6**.
- `7` — sync. Stack is now empty.
- Drain microtasks in FIFO order: `3`, `4`, `6`.
- Take one macrotask: `2`.

> 💼 **Interview** — This exact style of question appears constantly. Practise until you can do it without running the code.

### Microtasks added during the drain

```js
Promise.resolve().then(() => {
  console.log('A');
  Promise.resolve().then(() => console.log('B'));   // added mid-drain
});
setTimeout(() => console.log('C'));
// A B C  — B runs before C, because the drain continues until the queue is empty
```

This is why a self-scheduling microtask **starves** the event loop entirely:

```js
function starve() { Promise.resolve().then(starve); }
starve();   // ❌ setTimeout callbacks never run; the page freezes forever
```

The equivalent with `setTimeout` is fine — each is a separate macrotask, so rendering and other tasks interleave.

---

## 18.3 What Goes Where

| Microtask | Macrotask |
|---|---|
| `.then` / `.catch` / `.finally` | `setTimeout` / `setInterval` |
| `await` continuations | `setImmediate` (Node) |
| `queueMicrotask()` | I/O callbacks |
| `MutationObserver` | DOM events (click, input) |
| `process.nextTick` (Node — even higher priority) | `postMessage`, `MessageChannel` |

`requestAnimationFrame` is neither — it runs in the rendering phase, just before paint.

---

## 18.4 Blocking and Rendering

```js
// ❌ Blocks the thread for 3 seconds: no rendering, no clicks, no anything
const end = Date.now() + 3000;
while (Date.now() < end) {}

// ❌ Also blocks — sync work is sync no matter how you wrap it
async function slow() { return heavyComputation(); }   // still blocks
```

`async` does **not** create a thread. An `async` function runs synchronously until its first `await`.

**Fixes for heavy work:**

```js
// 1. Chunk it, yielding to the loop between chunks
async function processInChunks(items, fn, chunkSize = 500) {
  for (let i = 0; i < items.length; i += chunkSize) {
    items.slice(i, i + chunkSize).forEach(fn);
    await new Promise(r => setTimeout(r, 0));   // let the browser render
  }
}

// 2. Yield cooperatively (modern, better-scheduled)
if (navigator.scheduling?.isInputPending?.()) await scheduler.yield();

// 3. Move it off-thread entirely — the real answer for CPU-bound work
const worker = new Worker('./heavy.js', { type: 'module' });
worker.postMessage(data);
worker.onmessage = (e) => render(e.data);
```

`setTimeout(fn, 0)` is not really 0 — the spec clamps nested timeouts to **4 ms** after 5 levels of nesting. For a genuine "next macrotask," `MessageChannel` is faster.

---

## 18.5 Node.js Specifics

Node's loop has ordered **phases** per iteration:

```
timers → pending callbacks → idle/prepare → poll → check → close callbacks
  ▲          (setTimeout)                  (I/O)   (setImmediate)
```

Between **every** phase transition, Node drains `process.nextTick` (first) then the microtask queue.

```js
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('promise'));

// nextTick, promise, then timeout/immediate in NON-DETERMINISTIC order
// at the top level (depends on how long process startup took).
// Inside an I/O callback, setImmediate ALWAYS wins — it's the very next phase.
```

`process.nextTick` outranks promises. Overusing it starves I/O, which is why Node's docs recommend `queueMicrotask` for most cases.

---

## 18.6 Common Mistakes and Best Practices

**Mistake: assuming `setTimeout(fn, 0)` runs immediately.** It runs after all sync code and all microtasks.

**Mistake: thinking `async` makes code non-blocking.** Only `await` yields, and only at that point.

**Mistake: recursive microtasks.** They lock the page.

**Mistake: heavy work in an animation frame.** Anything over ~8 ms drops frames at 120 Hz.

**Best practice: measure with `performance.now()` and long-task observers**, not intuition.

**Best practice: CPU-bound → Web Worker. I/O-bound → async/await.** Async solves waiting, not computing.

---

## 18.7 Real-World Example: A Non-Blocking Batch Processor

```js
/**
 * Processes a large dataset while keeping the UI at 60fps by yielding
 * whenever the current frame's budget is exhausted.
 */
export async function processResponsively(items, work, { onProgress, signal } = {}) {
  const FRAME_BUDGET_MS = 5;      // leave headroom in a 16.7ms frame
  let index = 0;

  while (index < items.length) {
    const frameStart = performance.now();

    // Work until the budget runs out, then yield.
    while (index < items.length && performance.now() - frameStart < FRAME_BUDGET_MS) {
      work(items[index], index);
      index++;
    }

    onProgress?.(index / items.length);

    if (signal?.aborted) throw new DOMException('Aborted', 'AbortError');

    // Yield a MACROTASK so the browser can render. A microtask would not help.
    await new Promise(resolve => {
      if (typeof scheduler?.yield === 'function') scheduler.yield().then(resolve);
      else {
        const ch = new MessageChannel();      // faster than setTimeout(0)
        ch.port1.onmessage = () => resolve();
        ch.port2.postMessage(null);
      }
    });
  }
}
```

The critical line is the macrotask yield. Awaiting a resolved promise would only queue a microtask, which runs *before* rendering — the page would still freeze.

---

## 18.8 Questions

**1.** MCQ — How many macrotasks run per loop iteration? a) all b) one c) two d) depends
<details><summary>Answer</summary>**b)**</details>

**2.** MCQ — `queueMicrotask` schedules a: a) macrotask b) microtask c) frame d) tick
<details><summary>Answer</summary>**b)**</details>

**3.** MCQ — Which has highest priority in Node? a) `setImmediate` b) `Promise.then` c) `process.nextTick` d) `setTimeout`
<details><summary>Answer</summary>**c)**</details>

**4.** MCQ — `await` continuations are: a) macrotasks b) microtasks c) frames d) synchronous
<details><summary>Answer</summary>**b)**</details>

**5.** MCQ — Nested `setTimeout(fn, 0)` is clamped to: a) 0ms b) 1ms c) 4ms d) 16ms
<details><summary>Answer</summary>**c)** after 5 levels of nesting.</details>

**6.** MCQ — `requestAnimationFrame` is a: a) microtask b) macrotask c) render-phase callback d) tick
<details><summary>Answer</summary>**c)**</details>

**7.** Short — Why do all microtasks run before the next macrotask?
<details><summary>Answer</summary>The spec drains the microtask queue to empty after each task, so promise chains complete before yielding to timers, I/O, or rendering.</details>

**8.** Short — Can a microtask starve the event loop? Can a macrotask?
<details><summary>Answer</summary>A recursive microtask can — the drain never completes. A self-scheduling macrotask cannot; each is a separate turn, so rendering and other tasks interleave.</details>

**9.** Short — Does `async` create a thread?
<details><summary>Answer</summary>No. It's a single-threaded scheduling mechanism; the function body runs synchronously until the first `await`.</details>

**10.** Short — How do you truly parallelize CPU work?
<details><summary>Answer</summary>Web Workers (browser) or `worker_threads` / child processes (Node).</details>

**11.** Output —
```js
console.log('a');
setTimeout(() => console.log('b'));
Promise.resolve().then(() => console.log('c'));
console.log('d');
```
<details><summary>Answer</summary>`a d c b`</details>

**12.** Output —
```js
async function f() { console.log(1); await null; console.log(2); }
f();
console.log(3);
```
<details><summary>Answer</summary>`1 3 2`</details>

**13.** Output —
```js
setTimeout(() => console.log('t1'));
Promise.resolve().then(() => { console.log('p1'); setTimeout(() => console.log('t2')); });
Promise.resolve().then(() => console.log('p2'));
```
<details><summary>Answer</summary>`p1 p2 t1 t2`</details>

**14.** Output —
```js
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log('T', i));
  Promise.resolve().then(() => console.log('P', i));
}
```
<details><summary>Answer</summary>`P 0, P 1, P 2, T 0, T 1, T 2`</details>

**15.** Output —
```js
console.log(1);
(async () => { console.log(2); await new Promise(r => setTimeout(r)); console.log(3); })();
Promise.resolve().then(() => console.log(4));
console.log(5);
```
<details><summary>Answer</summary>`1 2 5 4 3` — the `await`ed promise resolves in a macrotask, so `3` comes last.</details>

**16.** Output — In Node, what prints?
```js
process.nextTick(() => console.log('n'));
Promise.resolve().then(() => console.log('p'));
setImmediate(() => console.log('i'));
```
<details><summary>Answer</summary>`n p i` — nextTick outranks the microtask queue, both before any phase callback.</details>

**17.** Debug — A progress bar never updates during a long loop that `await`s a resolved promise each iteration.
<details><summary>Answer</summary>Awaiting an already-resolved promise queues a microtask, which runs before rendering. Yield a macrotask (`setTimeout(0)` or `MessageChannel`) so the browser can paint.</details>

**18.** Debug — A page freezes with no error and 100% CPU on one core.
<details><summary>Answer</summary>Either an infinite synchronous loop or a recursive microtask. Check for `Promise.resolve().then(self)` patterns and unbounded `while`s. Profile with the Performance panel.</details>

**19.** Debug — `setInterval(fn, 100)` drifts and sometimes fires back-to-back.
<details><summary>Answer</summary>If `fn` takes longer than the interval, callbacks queue up. Use a self-scheduling `setTimeout` that re-arms *after* the work finishes.</details>

**20.** Code — Write `nextMacrotask()` and `nextMicrotask()` helpers.
<details><summary>Answer</summary>

```js
const nextMicrotask = () => Promise.resolve();
const nextMacrotask = () => new Promise(resolve => {
  const ch = new MessageChannel();
  ch.port1.onmessage = () => resolve();
  ch.port2.postMessage(null);
});
```
</details>

**21.** Code — Write `runWithTimeSlicing(tasks, budgetMs)` that runs tasks yielding every `budgetMs`.
<details><summary>Answer</summary>See §18.7 — the same frame-budget loop, parameterised.</details>

**22.** Code — Implement a non-drifting `interval(fn, ms)` returning a `stop()` function.
<details><summary>Answer</summary>

```js
function interval(fn, ms) {
  let stopped = false, expected = performance.now() + ms;
  const tick = async () => {
    if (stopped) return;
    await fn();
    expected += ms;
    setTimeout(tick, Math.max(0, expected - performance.now()));   // correct drift
  };
  setTimeout(tick, ms);
  return () => { stopped = true; };
}
```
</details>

**23.** Scenario — Sorting 500,000 rows in the browser freezes the tab for 4 seconds. Options, ranked.
<details><summary>Answer</summary>1) Move the sort to a Web Worker — CPU-bound work doesn't belong on the main thread. 2) Sort server-side and paginate. 3) If it must stay on-thread, chunk with a merge sort that yields between passes. Time-slicing alone still costs 4 seconds of wall time; only the worker or the server actually fixes it.</details>

**24.** Scenario — Why does `await` inside a loop over 1000 network calls take so long, and what's the fix?
<details><summary>Answer</summary>Sequential awaits serialize the requests — total time is the sum of latencies. Use `Promise.all` (or a bounded concurrency pool of ~6–10) so requests overlap. The event loop was never the bottleneck; the code just never issued more than one request at a time.</details>

---

## Chapter Summary

- One thread, one stack. Concurrency comes from the runtime's queues.
- Order: sync → **all** microtasks → render → **one** macrotask → repeat.
- Microtasks: promises, `await` continuations, `queueMicrotask`, `MutationObserver`. Macrotasks: timers, I/O, events.
- Recursive microtasks starve the loop; recursive macrotasks don't.
- `async` is scheduling, not threading. CPU-bound work needs a Worker.
- Node adds phases and `process.nextTick`, which outranks even promises.

---

## Practice Exercises

1. Write ten mixed sync/async snippets, predict each output, then verify. Keep score.
2. Build a visual event-loop simulator logging each queue's state per turn.
3. Demonstrate microtask starvation, then fix it with a macrotask yield.
4. Compare `setTimeout(0)`, `MessageChannel`, and `scheduler.yield()` latency over 1000 iterations.
5. Move a 2-second computation into a Web Worker and confirm the main thread stays at 60fps.

---

**Next:** [Chapter 19 — Callbacks, Promises, async/await →](19-async-javascript.md)
