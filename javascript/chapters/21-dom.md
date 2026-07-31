# Chapter 21 — The DOM

## Learning Objectives

- Select, create, and modify elements efficiently
- Understand reflow/repaint and batch DOM work
- Use `textContent` vs `innerHTML` safely
- Traverse and manipulate the tree

---

## 21.1 Selecting

```js
document.querySelector('.card');            // first match, or null
document.querySelectorAll('.card');         // STATIC NodeList
document.getElementById('main');            // fastest, but rarely the bottleneck
element.closest('.container');              // nearest ancestor (or self) matching
element.matches('.active');                 // boolean test
element.querySelectorAll(':scope > li');    // direct children only

// NodeList is not an Array
const items = document.querySelectorAll('li');
items.forEach(el => {});      // ✅ NodeList has forEach
items.map(el => {});          // ❌ no map
[...items].map(el => {});     // ✅
```

**Static vs live collections:**

```js
const staticList = document.querySelectorAll('.item');   // snapshot
const liveList = document.getElementsByClassName('item'); // updates automatically

document.body.append(newItem);
staticList.length;   // unchanged
liveList.length;     // +1
```

Live collections cause a classic infinite loop:

```js
const divs = document.getElementsByTagName('div');
for (let i = 0; i < divs.length; i++) document.body.append(document.createElement('div'));
// ❌ never terminates — the collection grows as you iterate
```

---

## 21.2 Creating and Inserting

```js
const el = document.createElement('div');
el.className = 'card';
el.dataset.id = '42';                   // → data-id="42"
el.textContent = 'Safe text';
el.setAttribute('aria-label', 'Card');

parent.append(el, 'text', el2);         // multiple nodes and strings, at the end
parent.prepend(el);
el.before(other); el.after(other);
el.replaceWith(other);
el.remove();

// insertAdjacentHTML — positions relative to the element
el.insertAdjacentHTML('beforeend', '<span>hi</span>');
// 'beforebegin' | 'afterbegin' | 'beforeend' | 'afterend'
```

**`textContent` vs `innerHTML` vs `innerText`:**

| | Parses HTML | XSS risk | Layout-aware | Speed |
|---|---|---|---|---|
| `textContent` | no | **safe** | no | fastest |
| `innerHTML` | yes | **dangerous** | no | slow |
| `innerText` | no | safe | yes (triggers reflow) | slowest |

> ⚠️ **Never** put user input into `innerHTML`. `el.innerHTML = userComment` is a textbook XSS hole (Chapter 31). Use `textContent`, or sanitize with `DOMPurify`, or use `setHTML()` where supported.

---

## 21.3 Attributes, Properties, Classes, Styles

```js
// Attributes (HTML) vs properties (JS object) — they can diverge
input.getAttribute('value');   // the ORIGINAL HTML value
input.value;                    // the CURRENT value the user typed

el.classList.add('a', 'b');
el.classList.remove('a');
el.classList.toggle('active', isActive);   // second arg forces on/off
el.classList.contains('a');
el.classList.replace('old', 'new');

el.style.color = 'red';                        // inline only
el.style.setProperty('--theme-hue', '210');    // CSS custom property
getComputedStyle(el).color;                     // resolved value — forces reflow
```

Prefer toggling classes over setting inline styles: CSS stays in CSS, and it's faster.

---

## 21.4 Performance: Reflow and Repaint

**Reflow (layout)** recalculates geometry — expensive. **Repaint** redraws pixels — cheaper. **Composite** is cheapest (only `transform` and `opacity`).

**Layout thrashing** is the classic killer: interleaving reads and writes forces a synchronous reflow on every iteration.

```js
// ❌ 100 forced reflows
for (const el of els) el.style.height = el.offsetHeight * 2 + 'px';

// ✅ Batch: read all, then write all
const heights = els.map(el => el.offsetHeight);   // all reads
els.forEach((el, i) => el.style.height = heights[i] * 2 + 'px');  // all writes
```

Properties that force synchronous layout when read: `offsetTop/Left/Width/Height`, `clientWidth/Height`, `scrollTop/Height`, `getBoundingClientRect()`, `getComputedStyle()`.

**Batch insertions with a fragment:**

```js
// ❌ 1000 reflows
for (const item of items) list.append(makeRow(item));

// ✅ 1 reflow
const frag = document.createDocumentFragment();
for (const item of items) frag.append(makeRow(item));
list.append(frag);
```

For very large lists, don't render them all — **virtualize** (render only the visible window).

---

## 21.5 Traversal and `<template>`

```js
el.parentElement; el.children;            // elements only
el.parentNode; el.childNodes;             // includes text nodes & comments
el.firstElementChild; el.nextElementSibling;
el.contains(other);

// Templates: parse once, clone many times
// <template id="row"><li class="row"><span class="name"></span></li></template>
const tpl = document.getElementById('row');
function makeRow(item) {
  const node = tpl.content.cloneNode(true);
  node.querySelector('.name').textContent = item.name;   // safe interpolation
  return node;
}
```

---

## 21.6 Common Mistakes and Best Practices

**Mistake: `innerHTML` with user data.** XSS.

**Mistake: `innerHTML +=`.** It re-parses the entire subtree, destroys and rebuilds every child node, and drops all attached event listeners.

**Mistake: querying inside a loop.** Cache selections outside.

**Mistake: mutating a live `HTMLCollection` while iterating.**

**Mistake: assuming an element exists.** `document.querySelector('.x').textContent` throws if absent. Use `?.` or guard.

**Best practice: batch reads then writes.**

**Best practice: prefer `data-*` attributes** over storing state in classes or the DOM tree shape.

**Best practice: let CSS do the work.** Toggle a class; don't animate with JS unless you need physics.

---

## 21.7 Real-World Example: A Virtualized List

```js
/**
 * Renders only visible rows. 100,000 items, ~20 DOM nodes.
 */
export function virtualList(container, { items, rowHeight, renderRow, overscan = 5 }) {
  const spacer = document.createElement('div');
  spacer.style.height = `${items.length * rowHeight}px`;
  spacer.style.position = 'relative';
  container.replaceChildren(spacer);

  let rendered = new Map();   // index -> element
  let frame = null;

  function draw() {
    frame = null;
    // READ phase — all measurements together
    const scrollTop = container.scrollTop;
    const viewport = container.clientHeight;

    const start = Math.max(0, Math.floor(scrollTop / rowHeight) - overscan);
    const end = Math.min(items.length, Math.ceil((scrollTop + viewport) / rowHeight) + overscan);

    // WRITE phase
    const next = new Map();
    const frag = document.createDocumentFragment();

    for (let i = start; i < end; i++) {
      let el = rendered.get(i);
      if (el) rendered.delete(i);                 // reuse
      else {
        el = renderRow(items[i], i);
        el.style.cssText = `position:absolute;top:${i * rowHeight}px;height:${rowHeight}px;left:0;right:0`;
        frag.append(el);
      }
      next.set(i, el);
    }

    for (const el of rendered.values()) el.remove();   // recycle offscreen rows
    spacer.append(frag);
    rendered = next;
  }

  const onScroll = () => { frame ??= requestAnimationFrame(draw); };
  container.addEventListener('scroll', onScroll, { passive: true });
  draw();

  return () => container.removeEventListener('scroll', onScroll);
}
```

The read/write split and the `requestAnimationFrame` coalescing are what keep this at 60fps.

---

## 21.8 Questions

**1.** MCQ — `querySelectorAll` returns: a) Array b) live HTMLCollection c) static NodeList d) Set
<details><summary>Answer</summary>**c)**</details>

**2.** MCQ — Safe for user text: a) `innerHTML` b) `textContent` c) `outerHTML` d) `insertAdjacentHTML`
<details><summary>Answer</summary>**b)**</details>

**3.** MCQ — Which forces synchronous layout? a) `textContent` b) `classList.add` c) `offsetHeight` d) `dataset.x`
<details><summary>Answer</summary>**c)**</details>

**4.** MCQ — `getElementsByClassName` returns: a) static b) live c) an array d) null
<details><summary>Answer</summary>**b)**</details>

**5.** MCQ — Cheapest to animate: a) `height` b) `top` c) `transform` d) `margin`
<details><summary>Answer</summary>**c)** — composite-only, no layout or paint.</details>

**6.** Short — Why is `innerHTML +=` bad?
<details><summary>Answer</summary>It serializes and re-parses the whole subtree, recreating every child node and discarding their event listeners and state.</details>

**7.** Short — What is layout thrashing?
<details><summary>Answer</summary>Alternating DOM reads and writes so each read forces a synchronous reflow of pending writes, turning one layout into N.</details>

**8.** Short — Attribute vs property?
<details><summary>Answer</summary>The attribute is the HTML source value; the property is the live JS state. For inputs they diverge as soon as the user types.</details>

**9.** Short — Why use a DocumentFragment?
<details><summary>Answer</summary>Nodes are assembled off-document and inserted in one operation, causing a single reflow instead of one per node.</details>

**10.** Short — When is `closest()` the right tool?
<details><summary>Answer</summary>Event delegation — finding which logical component a clicked descendant belongs to.</details>

**11.** Output —
```js
document.body.innerHTML = '<p>a</p>';
const p = document.querySelector('p');
document.body.innerHTML += '<p>b</p>';
console.log(document.contains(p));
```
<details><summary>Answer</summary>`false` — `+=` rebuilt the subtree, so the original `p` is detached.</details>

**12.** Output —
```js
const list = document.getElementsByTagName('li');
console.log(list.length);          // say 3
document.querySelector('ul').append(document.createElement('li'));
console.log(list.length);
```
<details><summary>Answer</summary>`3` then `4` — live collection.</details>

**13.** Output — `el.dataset.userId = 5; console.log(el.getAttribute('data-user-id'), typeof el.dataset.userId);`
<details><summary>Answer</summary>`'5' 'string'` — dataset values are always strings.</details>

**14.** Output —
```js
const d = document.createElement('div');
d.innerHTML = '<b>hi</b>';
console.log(d.textContent, d.children.length);
```
<details><summary>Answer</summary>`'hi' 1`</details>

**15.** Debug — A click handler stops working after the parent's content is refreshed.
<details><summary>Answer</summary>`innerHTML` replacement destroyed the node the listener was attached to. Use event delegation on a stable ancestor, or re-attach after re-render.</details>

**16.** Debug — A list of 5,000 rows takes 3 seconds to render.
<details><summary>Answer</summary>One insertion (and reflow) per row. Batch with a `DocumentFragment`, and virtualize if the list can grow.</details>

**17.** Debug — Scroll animation janks at 20fps.
<details><summary>Answer</summary>Layout thrashing in the scroll handler, or animating layout properties. Split reads/writes, coalesce with `requestAnimationFrame`, use `{passive: true}`, and animate `transform`/`opacity` only.</details>

**18.** Debug — `document.querySelector('#app')` returns `null` in a `<head>` script.
<details><summary>Answer</summary>The script runs before the element is parsed. Use `defer`, `type="module"`, place the script at the end of `<body>`, or wait for `DOMContentLoaded`.</details>

**19.** Code — Write `createElement(tag, props, ...children)` — a mini JSX-like helper that is XSS-safe.
<details><summary>Answer</summary>

```js
function h(tag, props = {}, ...children) {
  const el = document.createElement(tag);
  for (const [k, v] of Object.entries(props)) {
    if (k === 'class') el.className = v;
    else if (k.startsWith('on') && typeof v === 'function') el.addEventListener(k.slice(2).toLowerCase(), v);
    else if (k === 'dataset') Object.assign(el.dataset, v);
    else el.setAttribute(k, v);
  }
  el.append(...children.flat());   // strings become TEXT nodes — never parsed as HTML
  return el;
}
```
</details>

**20.** Code — Write `onVisible(el, cb)` using `IntersectionObserver`.
<details><summary>Answer</summary>

```js
function onVisible(el, cb, options = { threshold: 0.1 }) {
  const io = new IntersectionObserver(([entry]) => {
    if (entry.isIntersecting) { cb(entry); io.disconnect(); }
  }, options);
  io.observe(el);
  return () => io.disconnect();
}
```
</details>

**21.** Code — Write `batchDom(reads, writes)` scheduling reads then writes in one frame.
<details><summary>Answer</summary>

```js
function batchDom(read, write) {
  return new Promise(resolve => requestAnimationFrame(() => {
    const data = read();
    requestAnimationFrame(() => { write(data); resolve(); });
  }));
}
```
</details>

**22.** Scenario — A dashboard with 12 live-updating widgets drops frames.
<details><summary>Answer</summary>Coalesce all updates into a single `requestAnimationFrame` callback, batch reads before writes, use `transform` for movement, and consider `content-visibility: auto` plus an `IntersectionObserver` to skip offscreen widgets entirely.</details>

**23.** Scenario — Marketing wants to inject user-authored HTML into comments.
<details><summary>Answer</summary>Sanitize server-side *and* client-side with an allowlist library (DOMPurify), render into a container with a restrictive CSP, and never allow `<script>`, `on*` attributes, or `javascript:` URLs. If rich text is the real requirement, store structured markup (Markdown or a JSON document model) and render it yourself rather than accepting raw HTML.</details>

---

## Chapter Summary

- `querySelectorAll` is static; `getElementsBy*` is live (and loop-hazardous).
- `textContent` is safe and fast; `innerHTML` parses HTML and is an XSS vector; `innerText` triggers reflow.
- Batch reads before writes to avoid layout thrashing; use `DocumentFragment` for bulk inserts.
- Animate `transform`/`opacity` only.
- Virtualize long lists; use `<template>` for repeated structures.
- Cache selections; guard against missing elements.

---

## Practice Exercises

1. Build a sortable, filterable table over 10,000 rows that stays at 60fps.
2. Write a `diff`-based renderer updating only changed text nodes.
3. Measure a thrashing loop vs a batched one with the Performance panel; report the numbers.
4. Implement infinite scroll with `IntersectionObserver`.
5. Build a safe HTML renderer with an explicit tag/attribute allowlist and test it against 10 XSS payloads.

---

**Next:** [Chapter 22 — Events →](22-events.md)
