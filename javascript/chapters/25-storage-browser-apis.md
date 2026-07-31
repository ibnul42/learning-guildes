# Chapter 25 — Storage and Browser APIs

## Learning Objectives

- Choose between Web Storage, cookies, IndexedDB, and the Cache API for a given requirement
- Handle quota errors and cross-tab synchronisation correctly
- Use the four observer APIs instead of polling
- Move CPU work off the main thread with Web Workers

---

## 25.1 Web Storage

`localStorage` and `sessionStorage` share one API. They differ only in lifetime:
`sessionStorage` is per-tab and dies with it; `localStorage` persists until cleared and is
shared across all tabs on the same origin.

```js
localStorage.setItem('theme', 'dark');
localStorage.getItem('theme');        // 'dark'
localStorage.getItem('missing');      // null — not undefined
localStorage.removeItem('theme');
localStorage.clear();
localStorage.length;                  // 1
localStorage.key(0);                  // 'theme'
```

Three properties define what they're good for:

**String-only.** Everything is coerced with `String()`.

```js
localStorage.setItem('n', 42);
typeof localStorage.getItem('n');     // 'string'
localStorage.setItem('o', { a: 1 });
localStorage.getItem('o');            // '[object Object]'  ❌
localStorage.setItem('o', JSON.stringify({ a: 1 }));   // ✅
```

**Synchronous.** Every read and write blocks the main thread. Reading 2MB of JSON in a
`localStorage` call on page load is a measurable LCP regression.

**Roughly 5MB per origin,** and exceeding it throws:

```js
try {
  localStorage.setItem(key, value);
} catch (err) {
  // Safari private mode throws on ANY write; other browsers throw only when full.
  if (err.name === 'QuotaExceededError') evictOldEntries();
  else throw err;
}
```

> ⚠️ **Gotcha** — `localStorage` can be entirely unavailable: Safari private browsing, disabled
> third-party storage in an iframe, or a hardened browser config. Accessing it can throw a
> `SecurityError` on the *property access itself*, so feature-detect inside a `try`.

**Cross-tab sync** comes free via the `storage` event, which fires in **other** tabs only:

```js
window.addEventListener('storage', (e) => {
  if (e.key === 'theme') applyTheme(e.newValue);   // e.oldValue, e.url also available
  if (e.key === null) handleClear();               // clear() fires with key === null
});
```

---

## 25.2 Cookies

Cookies exist because the **server** needs to see them. If only your JavaScript needs the data,
use Web Storage or IndexedDB instead — cookies are sent on every single request to the origin.

```js
document.cookie = 'session=abc; Max-Age=3600; Path=/; Secure; SameSite=Lax';
document.cookie;    // 'a=1; b=2'  — a string of ALL readable cookies, no metadata
```

| Attribute | Effect |
|---|---|
| `HttpOnly` | Invisible to JavaScript. **The point** — it survives XSS. |
| `Secure` | HTTPS only |
| `SameSite=Strict` | Never sent cross-site. Breaks "return from payment provider" flows. |
| `SameSite=Lax` | Sent on top-level GET navigations. The modern default. |
| `SameSite=None` | Sent cross-site; requires `Secure`. |
| `Max-Age` / `Expires` | Lifetime. Omit both for a session cookie. |
| `Path` / `Domain` | Scope |

`document.cookie` cannot read or set `HttpOnly` cookies, which is exactly why auth tokens belong
in them (see Chapter 31). Modern code should prefer the async `cookieStore` API where available:

```js
await cookieStore.set({ name: 'theme', value: 'dark', sameSite: 'lax' });
const c = await cookieStore.get('theme');    // { name, value, domain, … } — real object
```

---

## 25.3 IndexedDB

IndexedDB is the real database: asynchronous, transactional, indexed, stores structured clones
(including `Blob`, `Map`, `Date`, `ArrayBuffer`), and offers hundreds of megabytes. Its native
API is event-based and unpleasant, so wrap it once.

```js
function openDB(name, version, upgrade) {
  return new Promise((resolve, reject) => {
    const req = indexedDB.open(name, version);
    req.onupgradeneeded = (e) => upgrade(req.result, e.oldVersion, e.newVersion);
    req.onsuccess = () => resolve(req.result);
    req.onerror = () => reject(req.error);
    req.onblocked = () => reject(new Error('Another tab holds an older version open'));
  });
}

const db = await openDB('notes', 2, (db, oldVersion) => {
  // Migrations are cumulative and must be idempotent: no `else`, fall through each version.
  if (oldVersion < 1) {
    const store = db.createObjectStore('notes', { keyPath: 'id', autoIncrement: true });
    store.createIndex('by_updated', 'updatedAt');
  }
  if (oldVersion < 2) {
    db.transaction('notes', 'versionchange');   // add indexes to an existing store here
  }
});
```

Transactions auto-commit when the microtask queue empties, which is the API's biggest trap:

```js
const tx = db.transaction('notes', 'readwrite');
const store = tx.objectStore('notes');
store.put({ id: 1, text: 'hello' });
await someUnrelatedFetch();          // ❌ tx has already committed — the next line throws
store.put({ id: 2, text: 'world' }); // TransactionInactiveError

// ✅ Do all store work first, then await completion.
await new Promise((res, rej) => {
  tx.oncomplete = res;
  tx.onerror = () => rej(tx.error);
});
```

Queries go through indexes and cursors:

```js
const tx = db.transaction('notes', 'readonly');
const index = tx.objectStore('notes').index('by_updated');
const recent = index.getAll(IDBKeyRange.lowerBound(Date.now() - 86_400_000));
```

| | Web Storage | IndexedDB | Cache API | Cookies |
|---|---|---|---|---|
| Async | ❌ | ✅ | ✅ | ❌ (`cookieStore` ✅) |
| Capacity | ~5MB | Large, quota-based | Large | ~4KB each |
| Value type | String | Structured clone | `Request`/`Response` | String |
| Sent to server | ❌ | ❌ | ❌ | ✅ every request |
| Indexed queries | ❌ | ✅ | ❌ | ❌ |
| Good for | Small prefs | App data, offline | HTTP responses | Auth, server state |

Check your budget before writing big things:

```js
const { usage, quota } = await navigator.storage.estimate();
console.log(`${(usage / quota * 100).toFixed(1)}% used`);
await navigator.storage.persist();     // ask not to be evicted under storage pressure
```

---

## 25.4 The Observer APIs

Four observers replace four things people used to do with polling or scroll handlers. All share
the same shape: construct with a callback, `observe(target)`, `disconnect()` when done.

```js
// Visibility — lazy loading, infinite scroll, impression tracking
const io = new IntersectionObserver((entries) => {
  for (const e of entries) {
    if (e.isIntersecting) { load(e.target); io.unobserve(e.target); }
  }
}, { rootMargin: '200px', threshold: [0, 0.5, 1] });   // rootMargin: start early

// Size — container queries in JS, canvas resizing. Fires without any resize event.
new ResizeObserver(([entry]) => {
  const { inlineSize, blockSize } = entry.contentBoxSize[0];
  chart.resize(inlineSize, blockSize);
}).observe(container);

// DOM changes — reacting to markup you don't control
new MutationObserver((records) => {
  for (const r of records) if (r.type === 'childList') scan(r.addedNodes);
}).observe(root, { childList: true, subtree: true, attributes: true });

// Performance entries — real user monitoring
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) report(entry.name, entry.duration);
}).observe({ type: 'longtask', buffered: true });   // buffered: include ones before we started
```

> ⚠️ **Gotcha** — writing to an element's size inside a `ResizeObserver` callback for that same
> element causes an infinite loop, and the browser will log "ResizeObserver loop completed with
> undelivered notifications". Write to a different element, or guard with a size comparison.

`structuredClone` is the deep-copy algorithm these APIs use, exposed directly:

```js
const copy = structuredClone({ d: new Date(), m: new Map([[1, 'a']]), buf: new ArrayBuffer(8) });
structuredClone({ fn: () => {} });   // ❌ DataCloneError — functions aren't cloneable
```

It handles cycles and `Map`/`Set`/`Date`/`RegExp`/`Blob`, which `JSON.parse(JSON.stringify(x))`
does not. It cannot handle functions, DOM nodes, or class identity (you get a plain object).

---

## 25.5 Workers and Cross-Context Messaging

A Web Worker is a separate thread with no DOM access. Use one for anything CPU-bound that would
otherwise block rendering.

```js
// main.js
const worker = new Worker(new URL('./parse.worker.js', import.meta.url), { type: 'module' });
worker.postMessage({ csv });                       // structured-cloned, not shared
worker.addEventListener('message', (e) => render(e.data));
worker.addEventListener('error', (e) => report(e.message));
worker.terminate();                                // no graceful shutdown — it stops mid-task

// parse.worker.js
self.addEventListener('message', ({ data }) => {
  self.postMessage(parseCsv(data.csv));            // heavy work, off the main thread
});
```

Large buffers should be **transferred**, not copied — ownership moves and the copy is free:

```js
const buf = new ArrayBuffer(50_000_000);
worker.postMessage(buf, [buf]);      // second arg is the transfer list
buf.byteLength;                      // 0 — neutered in this thread
```

Three related messaging APIs:

```js
// Between tabs/workers on the same origin — no shared reference needed
const channel = new BroadcastChannel('auth');
channel.postMessage({ type: 'logout' });
channel.onmessage = (e) => { if (e.data.type === 'logout') redirectToLogin(); };

// Fire-and-forget telemetry that survives page unload — fetch() would be cancelled
addEventListener('pagehide', () => {
  navigator.sendBeacon('/analytics', JSON.stringify(metrics));
});

// Client-side routing without a page load
history.pushState({ page: 2 }, '', '/products?page=2');
history.replaceState({ page: 2 }, '', '/products?page=2');   // no new entry
addEventListener('popstate', (e) => render(e.state));        // back/forward only
```

> ⚠️ **Gotcha** — `pushState` does **not** fire `popstate`. Only user navigation does. Call your
> render function yourself after pushing, or you'll ship a router that works on back but not on
> link clicks.

---

## 25.6 Common Mistakes and Best Practices

**Mistake: storing objects in `localStorage` without `JSON.stringify`.** You get the string
`'[object Object]'` and no error.

**Mistake: `JSON.parse(localStorage.getItem(k))` unguarded.** A corrupt or hand-edited value
throws `SyntaxError` on load and takes the whole app down. Always `try`/`catch` and fall back.

**Mistake: assuming `localStorage` exists.** Feature-detect with a real write inside a `try` —
the property access itself can throw in a sandboxed iframe.

**Mistake: awaiting anything in the middle of an IndexedDB transaction.** It auto-commits when
the microtask queue drains, and subsequent operations throw `TransactionInactiveError`.

**Mistake: non-cumulative IndexedDB migrations.** Using `else if` on `oldVersion` means a user
upgrading from v1 to v3 skips the v2 migration. Use sequential `if (oldVersion < n)` blocks.

**Mistake: storing auth tokens in `localStorage`.** Any XSS reads them instantly. `HttpOnly`
cookies are unreadable by script (Chapter 31).

**Mistake: polling for element size or visibility.** `ResizeObserver` and
`IntersectionObserver` are cheaper, more accurate, and don't run when nothing changed.

**Best practice: namespace and version your storage keys.** `myapp:v2:prefs` lets you ship a
schema change without colliding with, or being poisoned by, the old shape.

**Best practice: treat all client storage as untrusted and disposable.** The user can edit it,
the browser can evict it, and a different device won't have it. Validate on read; never rely on
it for authorisation.

**Best practice: one `BroadcastChannel` for cross-tab state, not the `storage` event.** It
carries structured data, fires for workers too, and doesn't require writing to disk.

---

## 25.7 Real-World Example: A Typed, Versioned, TTL-Aware Storage Layer

Every non-trivial app ends up needing this: namespaced keys, JSON handling, expiry, schema
versioning, and — critically — not crashing when storage is full or unavailable.

```js
export class Store {
  #prefix;
  #backing;
  #memory = new Map();     // fallback when the real backing store fails

  /**
   * @param namespace - key prefix, e.g. 'myapp'
   * @param version   - bump to invalidate every existing entry
   */
  constructor(namespace, { version = 1, backing = 'local' } = {}) {
    this.#prefix = `${namespace}:v${version}:`;
    this.#backing = Store.#probe(backing);
  }

  /** Storage can throw on property access alone (sandboxed iframes, Safari private). */
  static #probe(kind) {
    try {
      const s = kind === 'session' ? sessionStorage : localStorage;
      const probe = '__probe__';
      s.setItem(probe, '1');
      s.removeItem(probe);
      return s;
    } catch {
      return null;
    }
  }

  #key(key) { return this.#prefix + key; }

  /** @param ttlMs - omit for no expiry. */
  set(key, value, { ttlMs } = {}) {
    const record = { v: value, e: ttlMs ? Date.now() + ttlMs : null };
    const serialised = JSON.stringify(record);

    if (!this.#backing) { this.#memory.set(key, serialised); return false; }

    try {
      this.#backing.setItem(this.#key(key), serialised);
      return true;
    } catch (err) {
      if (err.name !== 'QuotaExceededError') throw err;
      // Full: drop expired entries, retry once, then degrade to memory rather than throw.
      this.prune();
      try {
        this.#backing.setItem(this.#key(key), serialised);
        return true;
      } catch {
        this.#memory.set(key, serialised);
        return false;
      }
    }
  }

  get(key, fallback = null) {
    const raw = this.#backing?.getItem(this.#key(key)) ?? this.#memory.get(key);
    if (raw == null) return fallback;

    let record;
    try { record = JSON.parse(raw); }
    catch { this.delete(key); return fallback; }     // corrupt value: discard, don't throw

    if (record.e !== null && record.e < Date.now()) { this.delete(key); return fallback; }
    return record.v;
  }

  delete(key) {
    this.#backing?.removeItem(this.#key(key));
    this.#memory.delete(key);
  }

  /** Remove expired entries in this namespace. Also drops entries from older versions. */
  prune() {
    if (!this.#backing) return;
    const namespace = this.#prefix.split(':v')[0] + ':';
    for (const key of Object.keys(this.#backing)) {
      if (!key.startsWith(namespace)) continue;
      if (!key.startsWith(this.#prefix)) { this.#backing.removeItem(key); continue; }
      try {
        const { e } = JSON.parse(this.#backing.getItem(key));
        if (e !== null && e < Date.now()) this.#backing.removeItem(key);
      } catch {
        this.#backing.removeItem(key);
      }
    }
  }

  /** Cross-tab updates for this namespace. Returns an unsubscribe function. */
  subscribe(key, onChange) {
    const handler = (e) => {
      if (e.key === null) return onChange(null);            // clear()
      if (e.key !== this.#key(key)) return;
      onChange(this.get(key));
    };
    addEventListener('storage', handler);
    return () => removeEventListener('storage', handler);
  }
}

const prefs = new Store('myapp', { version: 2 });
prefs.set('theme', 'dark');
prefs.set('draft', { title: 'Hi' }, { ttlMs: 60 * 60 * 1000 });
prefs.get('theme', 'light');            // 'dark'
```

The design decision worth defending is that `set` returns a boolean rather than throwing when it
falls back to memory. A preferences write is not important enough to break a render, and a
caller who cares can check the return value — but a caller who doesn't care gets working
software in Safari private mode instead of a white screen. Encoding TTL in the stored record
rather than a separate key means expiry survives a page reload and can't desynchronise from its
value.

---

## 25.8 Questions

**1.** MCQ — `localStorage.getItem('nope')` returns: a) `undefined` b) `null` c) `''` d) throws
<details><summary>Answer</summary>**b)** `null`.</details>

**2.** MCQ — The `storage` event fires in: a) the tab that wrote b) all tabs c) other tabs on the same origin d) all tabs on any origin
<details><summary>Answer</summary>**c)** Never in the tab that made the change.</details>

**3.** MCQ — Which cookie attribute makes a cookie invisible to JavaScript? a) `Secure` b) `SameSite` c) `HttpOnly` d) `Path`
<details><summary>Answer</summary>**c)**</details>

**4.** MCQ — IndexedDB stores values using: a) JSON b) the structured clone algorithm c) strings only d) `toString()`
<details><summary>Answer</summary>**b)** So `Date`, `Map`, `Blob`, and `ArrayBuffer` round-trip intact.</details>

**5.** MCQ — Which observer tells you an element entered the viewport? a) `MutationObserver` b) `ResizeObserver` c) `IntersectionObserver` d) `PerformanceObserver`
<details><summary>Answer</summary>**c)**</details>

**6.** MCQ — `structuredClone(fn)` where `fn` is a function: a) clones it b) returns `undefined` c) returns the same reference d) throws `DataCloneError`
<details><summary>Answer</summary>**d)**</details>

**7.** Short — Why is `localStorage` a poor choice for large data?
<details><summary>Answer</summary>It's synchronous, so every read and write blocks the main thread, and it caps out around 5MB with a string-only value type. IndexedDB is async and orders of magnitude larger.</details>

**8.** Short — `sessionStorage` vs `localStorage`?
<details><summary>Answer</summary>Same API. `sessionStorage` is scoped to one tab and cleared when it closes; `localStorage` persists and is shared across all tabs of the origin.</details>

**9.** Short — Why does an IndexedDB transaction fail after an `await`?
<details><summary>Answer</summary>Transactions auto-commit once the microtask queue drains with no pending requests. Awaiting unrelated work lets that happen, so later operations throw `TransactionInactiveError`.</details>

**10.** Short — What does `rootMargin` do on an `IntersectionObserver`?
<details><summary>Answer</summary>It grows or shrinks the root's bounding box for intersection purposes, so `'200px'` reports an element as visible 200px before it actually is — the standard way to preload just off-screen images.</details>

**11.** Short — Why transfer an `ArrayBuffer` to a worker instead of posting it?
<details><summary>Answer</summary>`postMessage` structured-clones by default, which copies the whole buffer. Transferring moves ownership at zero cost, at the price of neutering the buffer in the sending thread.</details>

**12.** Short — Why can't `fetch` reliably send analytics on page unload?
<details><summary>Answer</summary>The browser cancels in-flight requests as the document tears down. `navigator.sendBeacon` hands the request to the browser to send independently of the page's lifetime.</details>

**13.** Output —
```js
localStorage.setItem('a', 1);
localStorage.setItem('b', [1, 2]);
localStorage.setItem('c', { x: 1 });
console.log(localStorage.getItem('a'), localStorage.getItem('b'), localStorage.getItem('c'));
```
<details><summary>Answer</summary>`1 1,2 [object Object]` — all strings, via `String()` coercion.</details>

**14.** Output —
```js
localStorage.setItem('n', '0');
if (localStorage.getItem('n')) console.log('truthy'); else console.log('falsy');
```
<details><summary>Answer</summary>`truthy` — the string `'0'` is truthy; only the empty string is falsy.</details>

**15.** Output —
```js
const o = { d: new Date(0), s: new Set([1]) };
console.log(JSON.parse(JSON.stringify(o)));
console.log(structuredClone(o));
```
<details><summary>Answer</summary>First: `{ d: '1970-01-01T00:00:00.000Z', s: {} }` — the `Date` became a string and the `Set` an empty object. Second: a real `Date` and a real `Set`.</details>

**16.** Output —
```js
history.pushState({ p: 1 }, '', '/a');
addEventListener('popstate', () => console.log('pop'));
history.pushState({ p: 2 }, '', '/b');
```
<details><summary>Answer</summary>Nothing. `pushState` never fires `popstate`.</details>

**17.** Debug — An app works in normal browsing but shows a blank page in Safari private mode.
<details><summary>Answer</summary>Safari private mode throws `QuotaExceededError` on any `localStorage.setItem`, and the unhandled error kills initialisation. Wrap storage access in `try`/`catch` with an in-memory fallback.</details>

**18.** Debug — After deploying a new settings shape, some users see crashes reading a property of `undefined` from stored preferences.
<details><summary>Answer</summary>Old-shape values are still in `localStorage`. Version the key prefix (`app:v2:prefs`) so a schema change starts clean, and validate the parsed object before use.</details>

**19.** Debug — A `ResizeObserver` that adjusts an element's height logs "loop completed with undelivered notifications" and pegs the CPU.
<details><summary>Answer</summary>The callback resizes the same element it observes, so each write triggers another notification. Write to a different element, or compare against the last applied size and bail when unchanged.</details>

**20.** Debug — Logging out in one tab leaves the other tabs logged in.
<details><summary>Answer</summary>Nothing notified them. Broadcast the logout — a `BroadcastChannel` message, or a `localStorage` write whose `storage` event the other tabs act on — and have each tab clear its in-memory state and redirect.</details>

**21.** Debug — An IndexedDB upgrade works for new users but leaves long-time users missing a store.
<details><summary>Answer</summary>The `onupgradeneeded` handler used `if/else if` on `oldVersion`, so a jump from v1 to v3 executed only the v3 branch. Rewrite as independent sequential `if (oldVersion < n)` blocks so every intermediate migration runs.</details>

**22.** Debug — Infinite scroll fires the loader dozens of times for one scroll to the bottom.
<details><summary>Answer</summary>The sentinel stays intersecting while the fetch is in flight, and each notification triggers another load. `unobserve` the sentinel on trigger and re-observe once the new content is appended, or guard with an `isLoading` flag.</details>

**23.** Code — Write `safeLocalStorage` that never throws on `get` or `set`.
<details><summary>Answer</summary>

```js
const memory = new Map();
const available = (() => {
  try { localStorage.setItem('__t__', '1'); localStorage.removeItem('__t__'); return true; }
  catch { return false; }
})();

export const safeLocalStorage = {
  get(key, fallback = null) {
    try {
      const raw = available ? localStorage.getItem(key) : memory.get(key) ?? null;
      return raw === null ? fallback : JSON.parse(raw);
    } catch { return fallback; }
  },
  set(key, value) {
    const raw = JSON.stringify(value);
    try { available ? localStorage.setItem(key, raw) : memory.set(key, raw); return true; }
    catch { memory.set(key, raw); return false; }
  },
};
```
</details>

**24.** Code — Promisify a single IndexedDB request.
<details><summary>Answer</summary>

```js
const promisify = (request) => new Promise((resolve, reject) => {
  request.onsuccess = () => resolve(request.result);
  request.onerror = () => reject(request.error);
});

const note = await promisify(db.transaction('notes').objectStore('notes').get(1));
```
</details>

**25.** Code — Write `onVisible(el)` returning a promise that resolves the first time `el` enters the viewport.
<details><summary>Answer</summary>

```js
const onVisible = (el, { rootMargin = '0px' } = {}) =>
  new Promise((resolve) => {
    const io = new IntersectionObserver(([entry]) => {
      if (!entry.isIntersecting) return;
      io.disconnect();
      resolve(entry);
    }, { rootMargin });
    io.observe(el);
  });
```
</details>

**26.** Scenario — You need to cache 200MB of map tiles for offline use. Which storage, and why not the others?
<details><summary>Answer</summary>The Cache API, because tiles are HTTP responses and it stores `Request`/`Response` pairs directly, letting a service worker serve them with no serialisation step. `localStorage` is out on both size and its string-only, synchronous nature. IndexedDB would work and is the right answer if you need to query tiles by metadata, but you'd be reimplementing HTTP caching by hand. Call `navigator.storage.persist()` either way, since 200MB is well into the range where the browser will evict you under pressure.</details>

**27.** Scenario — Parsing a 40MB CSV freezes the UI for eight seconds. What do you change?
<details><summary>Answer</summary>Move the parse into a Web Worker and stream the file in with `File.stream()` rather than reading it all into memory, so peak memory stays bounded and the main thread stays free. Post progress messages back so you can show a real progress bar instead of a spinner. Transfer the resulting typed arrays rather than cloning them, since a 40MB copy across the boundary would reintroduce a chunk of the stall you just removed. If the result is only ever rendered a page at a time, keep it in the worker and post slices on demand.</details>

---

## Chapter Summary

- Web Storage is synchronous, string-only, ~5MB, and can throw on access alone — always guard it.
- Cookies are for data the server reads; `HttpOnly` is their one irreplaceable feature.
- IndexedDB is the real client database, but its transactions auto-commit the moment you `await` something unrelated.
- Make IndexedDB migrations cumulative with sequential `if (oldVersion < n)` blocks.
- The four observers replace polling for visibility, size, DOM changes, and performance entries.
- Treat all client storage as untrusted, evictable, and per-device: validate on read, never authorise on it.

---

## Practice Exercises

1. Write a feature-detect for `localStorage` that returns `false` in a sandboxed iframe, and prove it in one.
2. Fill `localStorage` until it throws, and log how much you actually got in each browser you have.
3. Build a cross-tab "you have unsaved changes" warning using `BroadcastChannel`.
4. Wrap IndexedDB in a promise-based `get`/`set`/`delete`/`keys` API with no third-party library.
5. Lazy-load images with `IntersectionObserver` and measure the LCP difference against eager loading.

---

## Mini Project: Offline-First Notes

Build a note-taking app that works with the network unplugged and reconciles when it returns.

- Notes stored in IndexedDB with `id`, `body`, `updatedAt`, and a `dirty` flag
- A service worker caching the app shell so a cold offline load still boots
- An outbound queue that flushes to the server on `online` and on app start
- Last-write-wins conflict resolution by `updatedAt`, with the loser kept as a recoverable copy
- Cross-tab sync via `BroadcastChannel` so two open tabs never diverge
- A storage-quota indicator using `navigator.storage.estimate()`
- Graceful degradation to memory-only when IndexedDB is unavailable

---

**Next:** [Chapter 26 — Regular Expressions →](26-regular-expressions.md)
