# Chapter 22 — Events

## Learning Objectives

- Explain capture, target, and bubble phases
- Use event delegation for dynamic content
- Create and dispatch custom events
- Manage listener lifecycle without leaks

---

## 22.1 Propagation

Every event travels three phases:

1. **Capture** — `window` → `document` → … → the target's parent
2. **Target** — the element itself
3. **Bubble** — back up to `window`

```js
el.addEventListener('click', handler);                  // bubble (default)
el.addEventListener('click', handler, true);            // capture
el.addEventListener('click', handler, { capture: true, once: true, passive: true, signal });
```

Options:

- `once` — auto-removes after the first invocation
- `passive` — promises you won't call `preventDefault()`, letting the browser scroll without waiting. **Essential** for `touchstart`/`wheel`.
- `signal` — an `AbortSignal`; aborting removes the listener
- `capture` — listen during the descent

```js
e.stopPropagation();          // stop travelling to other elements
e.stopImmediatePropagation(); // also stop other listeners on THIS element
e.preventDefault();           // cancel the default action (navigation, submit)
e.target;                     // what was actually clicked
e.currentTarget;              // what the listener is attached to
```

> ⚠️ `e.target` vs `e.currentTarget` is the most-asked event question. Clicking a `<span>` inside a `<button>` gives `target = span`, `currentTarget = button`.

Not all events bubble: `focus`, `blur`, `load`, `mouseenter`, `mouseleave`. Their bubbling counterparts are `focusin`, `focusout`, `mouseover`, `mouseout`.

---

## 22.2 Event Delegation

Attach **one** listener to a stable ancestor instead of one per child.

```js
// ❌ 1000 listeners, and new rows get none
document.querySelectorAll('.row button').forEach(b => b.addEventListener('click', handle));

// ✅ One listener, works for rows added later
list.addEventListener('click', (e) => {
  const btn = e.target.closest('[data-action]');
  if (!btn || !list.contains(btn)) return;

  const { action, id } = btn.dataset;
  switch (action) {
    case 'delete': deleteItem(id); break;
    case 'edit':   editItem(id); break;
  }
});
```

Benefits: constant memory, works with dynamically added elements, no cleanup per row. This is how React's synthetic event system works under the hood.

---

## 22.3 Custom Events

```js
// Dispatch
el.dispatchEvent(new CustomEvent('cart:add', {
  detail: { sku: 'ABC', qty: 2 },
  bubbles: true,      // default is FALSE for CustomEvent
  cancelable: true,
  composed: true,     // cross shadow-DOM boundaries
}));

// Listen
el.addEventListener('cart:add', (e) => console.log(e.detail.sku));

// Cancellable pattern
const ev = new CustomEvent('beforeSave', { cancelable: true, detail: { data } });
if (el.dispatchEvent(ev)) save();   // dispatchEvent returns false if preventDefault was called
```

Namespaced names (`cart:add`) avoid collisions with built-ins.

---

## 22.4 Lifecycle and Cleanup

```js
// ❌ Cannot be removed — different function identity each time
el.addEventListener('click', () => handle());
el.removeEventListener('click', () => handle());   // no-op

// ✅ Keep the reference
const onClick = () => handle();
el.addEventListener('click', onClick);
el.removeEventListener('click', onClick);

// ✅✅ Best — one controller removes many listeners
const controller = new AbortController();
const { signal } = controller;
el.addEventListener('click', onClick, { signal });
window.addEventListener('resize', onResize, { signal });
document.addEventListener('keydown', onKey, { signal });
controller.abort();   // all three removed
```

The `signal` approach is the modern standard for component teardown.

---

## 22.5 Common Mistakes and Best Practices

**Mistake: confusing `target` and `currentTarget`.**

**Mistake: `preventDefault()` in a passive listener.** Silently ignored, with a console warning.

**Mistake: overusing `stopPropagation()`.** It breaks delegation and analytics listeners elsewhere in the app. Prefer checking conditions in your handler.

**Mistake: unremoved listeners on detached nodes.** They keep the node and everything the closure captured alive.

**Mistake: heavy work in `scroll`/`resize`/`mousemove`.** These fire dozens of times per second — throttle or coalesce with `requestAnimationFrame`.

**Best practice: delegate for lists**, direct-attach for singletons.

**Best practice: `{ passive: true }` on scroll-blocking events** unless you genuinely need `preventDefault`.

---

## 22.6 Real-World Example: A Reusable Component Base

```js
export class Component {
  #controller = new AbortController();

  constructor(root) { this.root = root; }

  /** All listeners registered here are removed by destroy(). */
  on(target, type, handler, options = {}) {
    target.addEventListener(type, handler, { ...options, signal: this.#controller.signal });
    return this;
  }

  /** Delegated listener: fires only when the event originates inside `selector`. */
  delegate(type, selector, handler, options = {}) {
    return this.on(this.root, type, (e) => {
      const match = e.target.closest(selector);
      if (match && this.root.contains(match)) handler.call(match, e, match);
    }, options);
  }

  emit(type, detail) {
    return this.root.dispatchEvent(
      new CustomEvent(type, { detail, bubbles: true, cancelable: true })
    );
  }

  destroy() { this.#controller.abort(); }
}

class TodoList extends Component {
  constructor(root) {
    super(root);
    this.delegate('click', '[data-toggle]', (e, el) => this.toggle(el.dataset.id));
    this.delegate('click', '[data-delete]', (e, el) => {
      if (this.emit('todo:beforeDelete', { id: el.dataset.id })) this.remove(el.dataset.id);
    });
    this.on(document, 'keydown', (e) => { if (e.key === 'Escape') this.clearSelection(); });
  }
}
// One call cleans up everything:
todoList.destroy();
```

---

## 22.7 Questions

**1.** MCQ — Default phase for `addEventListener`: a) capture b) target c) bubble d) none
<details><summary>Answer</summary>**c)**</details>

**2.** MCQ — `e.currentTarget` is: a) what was clicked b) the listener's element c) the parent d) `window`
<details><summary>Answer</summary>**b)**</details>

**3.** MCQ — Which does NOT bubble? a) click b) focus c) input d) keydown
<details><summary>Answer</summary>**b)** (use `focusin`).</details>

**4.** MCQ — `passive: true` means: a) fires later b) won't call `preventDefault` c) capture phase d) once only
<details><summary>Answer</summary>**b)**</details>

**5.** MCQ — `CustomEvent` bubbles by default: a) true b) false
<details><summary>Answer</summary>**b)** You must set `bubbles: true`.</details>

**6.** Short — What is event delegation and why use it?
<details><summary>Answer</summary>One listener on a common ancestor handles events from many descendants via `e.target.closest()`. Constant memory, and it automatically covers dynamically added elements.</details>

**7.** Short — `stopPropagation` vs `stopImmediatePropagation`?
<details><summary>Answer</summary>The first prevents travel to other elements; the second also prevents remaining listeners on the same element.</details>

**8.** Short — Why can't you remove an anonymous listener?
<details><summary>Answer</summary>`removeEventListener` matches by function identity; a new arrow expression is a different object.</details>

**9.** Short — What does `dispatchEvent` return?
<details><summary>Answer</summary>`false` if the event was cancelable and `preventDefault()` was called; otherwise `true`.</details>

**10.** Short — Best modern way to clean up many listeners?
<details><summary>Answer</summary>Register all with a shared `AbortController` signal and call `abort()`.</details>

**11.** Output —
```html
<div id="o"><button id="i">x</button></div>
```
```js
o.addEventListener('click', e => console.log('o', e.target.id, e.currentTarget.id));
i.addEventListener('click', e => console.log('i'));
i.click();
```
<details><summary>Answer</summary>`i` then `o i o` — target stays the button, currentTarget differs.</details>

**12.** Output —
```js
el.addEventListener('click', () => console.log(1), true);
el.addEventListener('click', () => console.log(2));
el.click();
```
<details><summary>Answer</summary>`1 2` — on the target itself, listeners fire in registration order regardless of the capture flag.</details>

**13.** Output —
```js
el.addEventListener('click', () => console.log('a'), { once: true });
el.click(); el.click();
```
<details><summary>Answer</summary>`a` once.</details>

**14.** Debug — Buttons added after page load don't respond.
<details><summary>Answer</summary>Listeners were attached only to elements present at load. Delegate from a stable ancestor.</details>

**15.** Debug — A modal's outside-click handler fires immediately on open.
<details><summary>Answer</summary>The opening click is still bubbling to `document` when the listener is attached. Attach it in a `setTimeout(0)` / next microtask, or check `e.target` against the trigger element.</details>

**16.** Debug — Memory grows on every SPA route change.
<details><summary>Answer</summary>Listeners on `window`/`document` from unmounted views were never removed, pinning their closures. Use an `AbortController` per view.</details>

**17.** Debug — `preventDefault()` on `touchstart` does nothing.
<details><summary>Answer</summary>The listener is passive — either explicitly, or by the browser's default for `touchstart`/`wheel` on document-level targets. Pass `{ passive: false }`.</details>

**18.** Code — Write `delegate(root, selector, type, handler)`.
<details><summary>Answer</summary>

```js
function delegate(root, selector, type, handler, options) {
  const listener = (e) => {
    const match = e.target.closest(selector);
    if (match && root.contains(match)) handler.call(match, e, match);
  };
  root.addEventListener(type, listener, options);
  return () => root.removeEventListener(type, listener, options);
}
```
</details>

**19.** Code — Write `once(target, type)` returning a promise for the next event.
<details><summary>Answer</summary>

```js
const nextEvent = (target, type, { signal } = {}) =>
  new Promise((resolve, reject) => {
    target.addEventListener(type, resolve, { once: true, signal });
    signal?.addEventListener('abort', () => reject(signal.reason), { once: true });
  });
```
</details>

**20.** Code — Write an `EventEmitter` class with `on`, `off`, `once`, `emit`.
<details><summary>Answer</summary>

```js
class EventEmitter {
  #map = new Map();
  on(t, fn) { (this.#map.get(t) ?? this.#map.set(t, new Set()).get(t)).add(fn); return () => this.off(t, fn); }
  off(t, fn) { this.#map.get(t)?.delete(fn); }
  once(t, fn) { const w = (...a) => { this.off(t, w); fn(...a); }; return this.on(t, w); }
  emit(t, ...a) { for (const fn of [...(this.#map.get(t) ?? [])]) fn(...a); }
}
```
</details>

**21.** Scenario — A drag-and-drop feature is janky on mobile.
<details><summary>Answer</summary>Use Pointer Events (`pointerdown`/`move`/`up`) with `setPointerCapture` instead of separate mouse/touch paths, mark the move listener `{ passive: true }` and drive position with `transform` inside a `requestAnimationFrame`, and set `touch-action: none` in CSS rather than calling `preventDefault` on every move.</details>

**22.** Scenario — An analytics library stopped receiving clicks after your team added `stopPropagation()` to a card component.
<details><summary>Answer</summary>`stopPropagation` prevents the event from reaching document-level listeners. Remove it and instead guard inside your handler (check `e.target.closest()`), or if suppression is truly needed, re-dispatch a dedicated analytics event.</details>

---

## Chapter Summary

- Three phases: capture → target → bubble. `target` is what was hit; `currentTarget` is where the listener lives.
- Delegate with `closest()` for lists and dynamic content.
- `{ once, passive, signal, capture }` cover most listener needs; `signal` is the cleanest teardown.
- `CustomEvent` needs `bubbles: true` explicitly; `dispatchEvent` returns `false` when cancelled.
- Avoid `stopPropagation` — it breaks other people's listeners.
- Throttle high-frequency events and coalesce with `requestAnimationFrame`.

---

## Practice Exercises

1. Build a keyboard-navigable dropdown (arrows, Home/End, Escape, type-ahead) with one delegated listener.
2. Implement drag-and-drop reordering with Pointer Events.
3. Write a global hotkey manager supporting combos and scoped contexts.
4. Build a form wizard communicating between steps via custom events.
5. Prove a listener leak with a heap snapshot, then fix it with `AbortController`.

---

**Next:** [Chapter 23 — Forms and Validation →](23-forms-validation.md)
