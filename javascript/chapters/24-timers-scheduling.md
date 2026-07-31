# Chapter 24 — Timers and Scheduling

## Learning Objectives

- Explain why `setTimeout(fn, 0)` never runs in 0ms
- Fix `setInterval` drift with a self-correcting recursive timeout
- Choose between `requestAnimationFrame`, `requestIdleCallback`, `queueMicrotask`, and `scheduler.postTask`
- Implement `debounce` and `throttle` with leading/trailing options, `cancel`, and `flush`

---

## 24.1 `setTimeout` and `setInterval` Semantics

A timer's delay is a **minimum**, not a promise. `setTimeout(fn, 100)` means "put `fn` in the
macrotask queue no earlier than 100ms from now." When it actually runs depends on whether the
call stack is empty and what else is queued.

```js
const id = setTimeout(fn, 100, 'arg1', 'arg2');  // extra args are passed to fn
clearTimeout(id);

const iid = setInterval(fn, 100);
clearInterval(iid);
```

Three separate reasons a timer fires late:

```js
setTimeout(() => console.log('timer'), 0);
const start = Date.now();
while (Date.now() - start < 500) {}          // block the thread for 500ms
// 'timer' logs after ~500ms, not ~0ms
```

**The 4ms clamp.** After five levels of nesting, browsers clamp `setTimeout` to a minimum of
4ms. This is in the HTML spec, not a bug.

```js
let depth = 0, last = performance.now();
function tick() {
  const now = performance.now();
  console.log(depth, (now - last).toFixed(1)); // 0 ~0.2  1 ~0.3 … 5 ~4.1  6 ~4.1
  last = now;
  if (++depth < 8) setTimeout(tick, 0);
}
setTimeout(tick, 0);
```

**Background tabs.** Timers in a hidden tab are throttled to roughly once per second, and after
a few minutes browsers may freeze them entirely. Never build a clock that assumes its timer
fired on schedule — read the wall clock instead (see §24.6).

| | `setTimeout` | `setInterval` |
|---|---|---|
| Runs | Once | Repeatedly |
| Delay measured from | Scheduling time | End of previous *scheduling*, not previous *completion* |
| Overlapping work | Impossible | Possible — callbacks can queue up behind slow work |
| Drift | N/A | Accumulates |

> ⚠️ **Gotcha** — `setInterval` does not wait for your callback to finish. If the callback
> takes 150ms and the interval is 100ms, the queue grows without bound and the browser fires
> them back-to-back the moment it gets a chance.

> 💼 **Interview** — "What's the minimum delay for `setTimeout`?" The answer they want is
> "there isn't one — it's a minimum bound, and nested timers are clamped to 4ms."

---

## 24.2 Interval Drift and the Self-Correcting Fix

Every `setInterval` cycle adds a little error: the queue delay, the callback's own runtime, the
clamp. Over an hour a 1-second interval can be seconds off.

```js
// ❌ Drifts. Also allows overlap if work() is async and slow.
setInterval(work, 1000);

// ✅ Self-correcting: each timeout is shortened by however late the last one was.
function accurateInterval(fn, period, { signal } = {}) {
  let expected = performance.now() + period;

  const schedule = () => {
    const drift = performance.now() - expected;      // positive = we're late
    const next = Math.max(0, period - drift);        // claw the drift back
    const id = setTimeout(run, next);
    signal?.addEventListener('abort', () => clearTimeout(id), { once: true });
  };

  const run = () => {
    if (signal?.aborted) return;
    expected += period;
    fn();          // if fn throws, the loop stops — wrap it if that's not what you want
    schedule();
  };

  schedule();
}
```

For async work, chain rather than schedule — this guarantees no overlap:

```js
async function poll(fn, period, { signal } = {}) {
  while (!signal?.aborted) {
    await fn();                                        // never overlaps
    await new Promise(r => setTimeout(r, period));
  }
}
```

---

## 24.3 `requestAnimationFrame` and the Frame Budget

`requestAnimationFrame` runs your callback **immediately before the next paint**, at the
display's refresh rate. On a 60Hz screen that's every ~16.7ms; on a 120Hz screen, ~8.3ms.
Never hardcode 16.

```js
let raf;
function animate(timestamp) {          // timestamp is a DOMHighResTimeStamp, shared by all
  el.style.transform = `translateX(${timestamp / 10}px)`;  // callbacks in the same frame
  raf = requestAnimationFrame(animate);
}
raf = requestAnimationFrame(animate);
cancelAnimationFrame(raf);
```

Use the timestamp, not a frame counter, to make animation speed independent of refresh rate:

```js
let last = null;
function step(now) {
  const dt = last === null ? 0 : now - last;   // ms since previous frame
  last = now;
  x += velocity * dt;                          // same speed at 60Hz and 144Hz
  requestAnimationFrame(step);
}
```

The frame budget on a 60Hz display is 16.7ms total, and the browser needs part of that for
style, layout, paint, and compositing. Roughly 10ms of that is yours. rAF does not run in
background tabs at all — which makes it a good choice for animation and a bad one for anything
that must keep working while hidden.

---

## 24.4 The Rest of the Scheduling Toolkit

```js
queueMicrotask(fn);              // before the next macrotask, after the current one finishes
requestIdleCallback(fn, { timeout: 2000 });   // during idle time; timeout forces it eventually
```

`requestIdleCallback` hands you a deadline object. Respect it:

```js
const queue = [...tasks];
function flush(deadline) {
  while (queue.length && (deadline.timeRemaining() > 0 || deadline.didTimeout)) {
    process(queue.shift());
  }
  if (queue.length) requestIdleCallback(flush, { timeout: 1000 });
}
requestIdleCallback(flush, { timeout: 1000 });
```

**`scheduler.postTask`** (Chromium; behind a flag or absent elsewhere) is the first API to give
timers real priorities:

```js
await scheduler.postTask(work, { priority: 'user-blocking' });  // beats everything
await scheduler.postTask(work, { priority: 'user-visible' });   // default
await scheduler.postTask(work, { priority: 'background' });     // idle-ish
```

**`scheduler.yield()`** breaks a long task without losing your place in the queue — unlike
`await new Promise(r => setTimeout(r, 0))`, which sends you to the back:

```js
for (const item of tenThousandItems) {
  process(item);
  if (performance.now() - frameStart > 50) {
    await scheduler.yield?.() ?? new Promise(r => setTimeout(r));   // stay responsive
    frameStart = performance.now();
  }
}
```

| API | Runs | Use for |
|---|---|---|
| `queueMicrotask` | Before next macrotask | Deferring within the current tick |
| `requestAnimationFrame` | Before next paint | Visual updates only |
| `setTimeout` | After ≥ delay | Delays, debouncing, giving the UI a breath |
| `requestIdleCallback` | Idle time | Analytics, prefetch, non-urgent cleanup |
| `scheduler.postTask` | By priority | Prioritised work in a busy app |
| `scheduler.yield` | Resumes with priority | Chunking a long task |

**`AbortSignal.timeout`** is the declarative way to time-bound anything that takes a signal:

```js
const res = await fetch(url, { signal: AbortSignal.timeout(5000) });   // throws TimeoutError
const both = AbortSignal.any([AbortSignal.timeout(5000), userCancel.signal]);
```

---

## 24.5 Debounce and Throttle

**Debounce** waits for quiet: it fires once, after the calls stop. **Throttle** enforces a rate:
it fires at most once per interval, during a burst.

```
calls:     █ █ █ █ █ ░░░░░░░░░ █ █ ░░░░░
debounce:  ░░░░░░░░░░░░░░░░ ▲ ░░░░░░░░░▲     (trailing edge only)
throttle:  ▲ ░░░░ ▲ ░░░░ ▲ ░░░░░░░ ▲ ░░░░    (leading edge, then every wait ms)
```

Search-as-you-type is debounce. Scroll position readout is throttle.

```js
function debounce(fn, wait = 250, { leading = false, trailing = true } = {}) {
  let timer = null, lastArgs = null, lastThis = null, result;

  function invoke() {
    timer = null;
    if (trailing && lastArgs) {
      result = fn.apply(lastThis, lastArgs);
      lastArgs = lastThis = null;
    }
  }

  function debounced(...args) {
    lastArgs = args;
    lastThis = this;
    const callNow = leading && timer === null;
    clearTimeout(timer);
    timer = setTimeout(invoke, wait);
    if (callNow) {
      result = fn.apply(this, args);
      lastArgs = lastThis = null;      // don't also fire on the trailing edge
    }
    return result;
  }

  debounced.cancel = () => { clearTimeout(timer); timer = null; lastArgs = lastThis = null; };
  debounced.flush  = () => { if (timer) { clearTimeout(timer); invoke(); } return result; };
  debounced.pending = () => timer !== null;
  return debounced;
}
```

```js
function throttle(fn, wait = 250, { leading = true, trailing = true } = {}) {
  let last = 0, timer = null, lastArgs = null, lastThis = null;

  function throttled(...args) {
    const now = performance.now();
    if (!last && !leading) last = now;         // skip the leading call
    const remaining = wait - (now - last);
    lastArgs = args; lastThis = this;

    if (remaining <= 0) {
      clearTimeout(timer); timer = null;
      last = now;
      fn.apply(this, args);
      lastArgs = lastThis = null;
    } else if (trailing && timer === null) {
      timer = setTimeout(() => {
        last = leading ? performance.now() : 0;
        timer = null;
        if (lastArgs) fn.apply(lastThis, lastArgs);
        lastArgs = lastThis = null;
      }, remaining);
    }
  }

  throttled.cancel = () => { clearTimeout(timer); timer = null; last = 0; lastArgs = null; };
  return throttled;
}
```

> ⚠️ **Gotcha** — `input.addEventListener('input', debounce(handler, 300))` is correct;
> `input.addEventListener('input', () => debounce(handler, 300)())` creates a **new** debounced
> function on every keystroke, so nothing is ever debounced. Debounce once, outside the handler.

> 💼 **Interview** — Implementing `debounce` from scratch is the single most common JavaScript
> whiteboard task. Know it cold, including why `apply` is used to preserve `this`.

---

## 24.6 Common Mistakes and Best Practices

**Mistake: expecting `setTimeout(fn, 0)` to run synchronously or immediately.** It runs after
the current task and everything already queued, with a 4ms floor once nested.

**Mistake: counting timer ticks to measure elapsed time.** Background throttling and drift make
tick-counting wrong. Read `Date.now()` or `performance.now()` and compute the difference.

**Mistake: `setInterval` around async work.** Callbacks overlap. Chain with `await` instead.

**Mistake: creating the debounced function inside the event handler.** A fresh closure each call
means no shared timer, so no debouncing.

**Mistake: forgetting `clearTimeout`/`clearInterval` on teardown.** The callback keeps its whole
closure — and any DOM nodes in it — alive. This is one of the four classic leaks (Chapter 27).

**Mistake: `requestAnimationFrame` for non-visual work.** It stops entirely in a hidden tab, so
your polling or saving silently stops too.

**Mistake: animating with `setInterval(…, 16)`.** It fights the compositor and looks wrong on
anything that isn't 60Hz. Use rAF.

**Best practice: tie every timer to an `AbortSignal`.** One `controller.abort()` then cleans up
timers, listeners, and in-flight fetches together.

**Best practice: throttle to a frame, not to a millisecond count.** For scroll and pointer
handlers, coalescing into one rAF callback is both simpler and more correct than a 16ms throttle.

**Best practice: prefer CSS transitions and the Web Animations API over JS animation loops.**
They run off the main thread and keep going when the main thread is busy.

---

## 24.7 Real-World Example: A Batched Animation Scheduler

A common performance bug is many components each calling `requestAnimationFrame` independently:
N callbacks, N chances to interleave reads and writes, N layout thrashes. This scheduler
collapses all of them into one frame, splits reads from writes, and gives up when the frame
budget is blown.

```js
export class FrameScheduler {
  #reads = new Set();
  #writes = new Set();
  #raf = null;
  #budgetMs;

  /** @param budgetMs - drop out and continue next frame past this much time in the callback. */
  constructor({ budgetMs = 8 } = {}) {
    this.#budgetMs = budgetMs;
  }

  /** Measure phase — anything that reads layout (offsetTop, getBoundingClientRect). */
  read(fn)  { this.#reads.add(fn);  this.#schedule(); return () => this.#reads.delete(fn); }

  /** Mutate phase — anything that writes style or the DOM. */
  write(fn) { this.#writes.add(fn); this.#schedule(); return () => this.#writes.delete(fn); }

  #schedule() {
    this.#raf ??= requestAnimationFrame(this.#flush);
  }

  #flush = (timestamp) => {
    this.#raf = null;
    const start = performance.now();

    // ALL reads before ANY write: one layout recalculation for the whole frame.
    const reads = [...this.#reads];
    this.#reads.clear();
    for (const fn of reads) this.#run(fn, timestamp);

    const writes = [...this.#writes];
    this.#writes.clear();
    for (const fn of writes) {
      // Over budget: requeue the rest rather than blow the frame and drop a paint.
      if (performance.now() - start > this.#budgetMs) {
        for (const remaining of writes.slice(writes.indexOf(fn))) this.#writes.add(remaining);
        break;
      }
      this.#run(fn, timestamp);
    }

    if (this.#reads.size || this.#writes.size) this.#schedule();
  };

  /** One task throwing must not abort the frame for every other subscriber. */
  #run(fn, timestamp) {
    try { fn(timestamp); }
    catch (err) { queueMicrotask(() => { throw err; }); }   // report without unwinding us
  }

  cancel() {
    cancelAnimationFrame(this.#raf);
    this.#raf = null;
    this.#reads.clear();
    this.#writes.clear();
  }
}

// Usage — a parallax effect and a progress bar sharing one frame.
const frames = new FrameScheduler();

window.addEventListener('scroll', () => {
  let y, height;
  frames.read(()  => { y = window.scrollY; height = document.body.scrollHeight; });
  frames.write(() => {
    bg.style.transform = `translate3d(0, ${y * 0.5}px, 0)`;
    bar.style.width = `${(y / (height - innerHeight)) * 100}%`;
  });
}, { passive: true });
```

The interesting decision is rethrowing task errors inside `queueMicrotask` rather than
swallowing them or letting them propagate. Letting them propagate would abandon every remaining
subscriber's frame because of one buggy component; swallowing them hides real bugs. Rethrowing
asynchronously reports the error to `window.onerror` and your monitoring exactly as an unhandled
exception would, while the frame finishes normally.

---

## 24.8 Questions

**1.** MCQ — The minimum delay a nested `setTimeout` is clamped to: a) 0ms b) 1ms c) 4ms d) 16ms
<details><summary>Answer</summary>**c)** After five levels of nesting, per the HTML spec.</details>

**2.** MCQ — `requestAnimationFrame` in a background tab: a) runs at 1Hz b) runs normally c) doesn't run d) throws
<details><summary>Answer</summary>**c)** It stops entirely until the tab is visible again.</details>

**3.** MCQ — Which fires soonest after the current synchronous code? a) `setTimeout(fn,0)` b) `queueMicrotask(fn)` c) `requestIdleCallback(fn)` d) `setInterval(fn,0)`
<details><summary>Answer</summary>**b)** Microtasks drain before the next macrotask.</details>

**4.** MCQ — Search-as-you-type should use: a) throttle b) debounce c) rAF d) `setInterval`
<details><summary>Answer</summary>**b)** You want one request after typing stops, not one per burst.</details>

**5.** MCQ — `AbortSignal.timeout(1000)` rejects the awaiting operation with: a) `AbortError` b) `TimeoutError` c) `RangeError` d) `null`
<details><summary>Answer</summary>**b)** A `DOMException` named `TimeoutError` — distinguishable from a user cancel.</details>

**6.** MCQ — The rAF callback argument is: a) elapsed ms since the last frame b) a `DOMHighResTimeStamp` for this frame c) the frame index d) `undefined`
<details><summary>Answer</summary>**b)** And every callback in the same frame receives the identical value.</details>

**7.** Short — Why does `setTimeout(fn, 100)` sometimes run after 400ms?
<details><summary>Answer</summary>The delay is a minimum. The timer only queues the callback; it runs when the stack is empty and earlier queued tasks have finished. A long synchronous block or a busy queue delays it arbitrarily.</details>

**8.** Short — What is interval drift and why does it accumulate?
<details><summary>Answer</summary>Each cycle is late by the queue delay plus the callback's runtime, and `setInterval` measures the next delay from the scheduled time rather than correcting for lateness. The errors add up.</details>

**9.** Short — Why is `queueMicrotask` not a way to yield to the browser?
<details><summary>Answer</summary>Microtasks run before the browser gets to render. The task queue is drained completely, so scheduling microtasks in a loop blocks paint just as effectively as a `while` loop.</details>

**10.** Short — Difference between `scheduler.yield()` and `await new Promise(r => setTimeout(r))`?
<details><summary>Answer</summary>Both yield to the event loop, but `setTimeout` puts your continuation at the back of the task queue behind any work queued while you were running. `scheduler.yield()` resumes with priority, so unrelated tasks can't starve you.</details>

**11.** Short — Why pass `{ passive: true }` to a scroll listener that schedules a rAF?
<details><summary>Answer</summary>It tells the browser you won't call `preventDefault()`, so scrolling can proceed on the compositor without waiting for your handler to return.</details>

**12.** Output — What prints, in order?
```js
console.log('A');
setTimeout(() => console.log('B'), 0);
queueMicrotask(() => console.log('C'));
requestAnimationFrame(() => console.log('D'));
Promise.resolve().then(() => console.log('E'));
console.log('F');
```
<details><summary>Answer</summary>`A F C E B` then `D` — or `A F C E D B` depending on when the next frame lands. Sync first, then microtasks in order, then the timer and the frame callback race; the frame usually wins only if a paint was already pending.</details>

**13.** Output —
```js
for (let i = 0; i < 3; i++) setTimeout(() => console.log(i), 0);
for (var j = 0; j < 3; j++) setTimeout(() => console.log(j), 0);
```
<details><summary>Answer</summary>`0 1 2` then `3 3 3` — `let` creates a fresh binding per iteration, `var` does not.</details>

**14.** Output —
```js
let n = 0;
const id = setInterval(() => { if (++n === 3) clearInterval(id); console.log(n); }, 10);
```
<details><summary>Answer</summary>`1 2 3` — `clearInterval` inside the callback still lets the current invocation finish.</details>

**15.** Output —
```js
const d = debounce(() => console.log('run'), 100);
d(); d(); d();
d.flush();
d();
```
<details><summary>Answer</summary>`run` immediately from `flush()`, then `run` again ~100ms later from the final call.</details>

**16.** Debug — A countdown timer built on `setInterval(tick, 1000)` shows the wrong remaining time after the user switches tabs and comes back.
<details><summary>Answer</summary>Background throttling meant far fewer ticks fired than seconds elapsed, and the code decremented a counter per tick. Store the target timestamp and compute `target - Date.now()` on every tick instead of counting.</details>

**17.** Debug — A debounced save fires on every keystroke.
<details><summary>Answer</summary>`debounce()` is being called inside the handler, producing a new function (and a new timer) each time. Create the debounced function once and pass the reference.</details>

**18.** Debug — An SPA gets slower with every route change, and the CPU profile shows dozens of identical rAF callbacks.
<details><summary>Answer</summary>Each mounted view starts an animation loop that recursively re-schedules itself, and unmount never calls `cancelAnimationFrame`. Store the handle and cancel on teardown, or tie the loop to an `AbortSignal`.</details>

**19.** Debug — A polling function that `await`s a slow endpoint every 5s sends bursts of overlapping requests under load.
<details><summary>Answer</summary>It uses `setInterval`, which doesn't wait for the async callback. Requests pile up when the endpoint is slower than the interval. Replace with a chained `await fn(); await sleep(5000);` loop.</details>

**20.** Debug — A throttled `mousemove` handler feels laggy even at a 16ms interval.
<details><summary>Answer</summary>The throttle's timer is not aligned to frames, so writes land mid-frame and get composited a frame late. Coalesce into a single `requestAnimationFrame` callback instead of throttling by time.</details>

**21.** Code — Write `sleep(ms)` that is cancellable via an `AbortSignal`.
<details><summary>Answer</summary>

```js
function sleep(ms, { signal } = {}) {
  return new Promise((resolve, reject) => {
    if (signal?.aborted) return reject(signal.reason);
    const id = setTimeout(resolve, ms);
    signal?.addEventListener('abort', () => {
      clearTimeout(id);
      reject(signal.reason);
    }, { once: true });
  });
}
```
</details>

**22.** Code — Write `rafThrottle(fn)`, which runs `fn` at most once per frame with the latest arguments.
<details><summary>Answer</summary>

```js
function rafThrottle(fn) {
  let raf = null, lastArgs;
  const throttled = (...args) => {
    lastArgs = args;
    raf ??= requestAnimationFrame(() => { raf = null; fn(...lastArgs); });
  };
  throttled.cancel = () => { cancelAnimationFrame(raf); raf = null; };
  return throttled;
}
```
</details>

**23.** Code — Write `withTimeout(promise, ms)` that rejects with a `TimeoutError` if the promise doesn't settle in time, without leaving a dangling timer.
<details><summary>Answer</summary>

```js
function withTimeout(promise, ms) {
  let id;
  const timeout = new Promise((_, reject) => {
    id = setTimeout(
      () => reject(new DOMException(`Timed out after ${ms}ms`, 'TimeoutError')),
      ms
    );
  });
  return Promise.race([promise, timeout]).finally(() => clearTimeout(id));
}
```
</details>

**24.** Code — Implement `requestIdleCallback` on top of `setTimeout` for environments that lack it.
<details><summary>Answer</summary>

```js
globalThis.requestIdleCallback ??= (cb, { timeout = 50 } = {}) => {
  const start = performance.now();
  return setTimeout(() => cb({
    didTimeout: false,
    timeRemaining: () => Math.max(0, timeout - (performance.now() - start)),
  }), 1);
};
globalThis.cancelIdleCallback ??= clearTimeout;
```
Note this is a shim, not a polyfill: it can't detect real idle time, so it only approximates the API surface.</details>

**25.** Scenario — A dashboard polls six endpoints on independent intervals. Users complain it drains battery and that data is stale after they return to the tab.
<details><summary>Answer</summary>Six independent timers means six wakeups per cycle and no coordination. Consolidate into one scheduler that fires all due fetches together, so the CPU wakes once. Pause polling on `visibilitychange` when hidden and do one immediate refresh on becoming visible — that fixes both the battery drain and the staleness, since a throttled background timer was never going to keep data fresh anyway. If freshness while hidden genuinely matters, move to a server push (SSE or WebSocket) rather than fighting the throttle.</details>

**26.** Scenario — An autocomplete uses a 300ms debounce, but users on slow connections still see results for a query they've since edited.
<details><summary>Answer</summary>Debouncing limits how many requests you send; it does nothing about responses arriving out of order. Attach an `AbortController` per request and abort the previous one before issuing the next, so a superseded response never resolves. As a belt-and-braces measure, tag each request with the query string and discard responses whose tag doesn't match the current input value. Lowering the debounce then becomes safe, which makes the feature feel faster rather than slower.</details>

---

## Chapter Summary

- Timer delays are minimum bounds; nested timeouts are clamped to 4ms and hidden tabs are throttled to ~1Hz.
- `setInterval` drifts and allows overlap. Chain timeouts, correcting each delay by the measured drift.
- Never count ticks to measure time — compute against a stored timestamp.
- `requestAnimationFrame` is for visual work only; use its timestamp so speed is refresh-rate independent.
- Debounce waits for silence, throttle enforces a rate. Create the wrapped function once, outside the handler.
- Tie every timer to an `AbortSignal` so one `abort()` tears down timers, listeners, and fetches together.

---

## Practice Exercises

1. Prove the 4ms clamp: log the real delay for eight levels of nested `setTimeout(fn, 0)`.
2. Write a `Stopwatch` with start, stop, lap, and reset that stays accurate across tab-backgrounding.
3. Extend the `throttle` above with a `flush()` method and write tests for the four leading/trailing combinations.
4. Build a task queue that processes 10,000 items without ever blocking the main thread for more than 50ms, using `scheduler.yield` with a `setTimeout` fallback.
5. Measure whether a 16ms throttle or a rAF-coalesced handler produces smoother scrolling, using the Performance panel's frame chart as evidence.

---

## Mini Project: An Accurate `Ticker`

Build a `Ticker` class that drives a countdown and stays correct no matter how the browser
throttles it. Construct it with a target `Date`, subscribe to per-second updates, and have it
survive the tab being backgrounded for ten minutes.

- Compute remaining time from `target - Date.now()`, never from a tick count
- Self-correct the next timeout so ticks land near second boundaries rather than drifting
- Pause on `visibilitychange` when hidden and emit one immediate update on becoming visible
- Expose `start()`, `pause()`, `resume()`, `stop()`, and `on('tick' | 'complete')`
- Accept an `AbortSignal` that stops everything and removes all listeners
- Emit `complete` exactly once, even if `stop()` and expiry race

---

**Next:** [Chapter 25 — Storage and Browser APIs →](25-storage-browser-apis.md)
