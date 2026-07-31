# Chapter 30 — Design Patterns

## Learning Objectives

- Recognise the creational, structural, and behavioural patterns in idiomatic JavaScript
- Name the cheaper alternative to each pattern and when it wins
- Implement Observer, Strategy, Command, and State machine from scratch
- Resist applying a pattern because it has a name

---

## 30.1 Patterns Are a Vocabulary, Not a Checklist

The Gang of Four catalogued patterns in C++ and Smalltalk, where a language limitation forced each
workaround. JavaScript has first-class functions, closures, dynamic dispatch, and modules, so
several classic patterns collapse into two lines here.

| Classic pattern | In JavaScript |
|---|---|
| Singleton | A module — it's already a single instance |
| Strategy | Pass a function |
| Command | A closure, or `{ execute, undo }` |
| Iterator | `Symbol.iterator` / a generator |
| Template Method | A callback parameter |
| Abstract Factory | A function that returns an object |

The value of a pattern name is communication: "this is a Strategy" tells a reviewer the shape in
four words. The cost is that naming a thing makes people want to build it. Apply a pattern when you
have the problem it solves twice, not when you can imagine having it.

> 💼 **Interview** — Asked to "use a design pattern", say which problem you're solving first. An
> interviewer who wanted a Singleton and gets "a module already gives us one instance; a class-based
> Singleton adds a lazy-init hook if we need it" is hearing the better answer.

---

## 30.2 Creational

**Factory** — a function returning objects, choosing the concrete type at runtime.

```js
const parsers = { json: parseJson, csv: parseCsv, xml: parseXml };

export function createParser(type) {
  const parse = parsers[type];
  if (!parse) throw new Error(`Unsupported format: ${type}`);
  return { type, parse };
}
```

*Earns its keep* when construction involves a decision or validation callers shouldn't repeat.
*Cheaper alternative:* an object literal keyed by type, as above — that's already the pattern.

**Builder** — step-by-step construction of something with many optional parts.

```js
class QueryBuilder {
  #parts = { select: ['*'], from: null, where: [], order: null, limit: null };

  select(...cols) { this.#parts.select = cols; return this; }
  from(table)     { this.#parts.from = table; return this; }
  where(clause, ...params) { this.#parts.where.push({ clause, params }); return this; }
  orderBy(col, dir = 'ASC') { this.#parts.order = `${col} ${dir}`; return this; }
  limit(n)        { this.#parts.limit = n; return this; }

  build() {
    const { select, from, where, order, limit } = this.#parts;
    if (!from) throw new Error('from() is required');
    return {
      sql: [
        `SELECT ${select.join(', ')}`,
        `FROM ${from}`,
        where.length && `WHERE ${where.map(w => w.clause).join(' AND ')}`,
        order && `ORDER BY ${order}`,
        limit != null && `LIMIT ${limit}`,
      ].filter(Boolean).join(' '),
      params: where.flatMap(w => w.params),
    };
  }
}

new QueryBuilder().select('id', 'name').from('users').where('age > ?', 18).limit(10).build();
```

*Earns its keep* for more than about four optional parameters, or when order matters.
*Cheaper alternative:* an options object with defaults. Use that until validation between steps
becomes real.

**Singleton** — exactly one instance. In JavaScript, a module already is one:

```js
// config.js — evaluated once, cached by the module system
export const config = Object.freeze({ apiUrl: process.env.API_URL });
```

The class form only adds lazy initialisation:

```js
class Connection {
  static #instance = null;
  static get instance() { return Connection.#instance ??= new Connection(); }
  #constructor() {}     // (illustrative — JS can't truly prevent `new`)
}
```

*Earns its keep* rarely. *Cheaper alternative:* a module export, or explicit dependency injection —
which is testable, whereas a Singleton is global state and usually isn't.

**Prototype** — clone an existing object rather than constructing from scratch.

```js
const baseConfig = { retries: 3, timeout: 5000, headers: { accept: 'application/json' } };
const clone = structuredClone(baseConfig);            // deep, handles Map/Date/cycles
const derived = { ...baseConfig, timeout: 10_000 };   // shallow, usually enough
```

---

## 30.3 Structural

**Adapter** — make an incompatible interface fit the one your code expects.

```js
// Your code wants: { get(key), set(key, value) }
const localStorageAdapter = {
  get: (k) => JSON.parse(localStorage.getItem(k) ?? 'null'),
  set: (k, v) => localStorage.setItem(k, JSON.stringify(v)),
};
const redisAdapter = (client) => ({
  get: async (k) => JSON.parse(await client.get(k) ?? 'null'),
  set: (k, v) => client.set(k, JSON.stringify(v)),
});
```

*Earns its keep* at every third-party boundary — it's the seam that lets you swap or mock the
dependency. Note the async/sync mismatch above: make your interface async so both fit.

**Decorator** — wrap to add behaviour, keeping the same interface.

```js
const withRetry = (fn, attempts = 3) => async (...args) => {
  let lastError;
  for (let i = 0; i < attempts; i++) {
    try { return await fn(...args); }
    catch (err) { lastError = err; await sleep(2 ** i * 100); }
  }
  throw lastError;
};
const withCache = (fn, cache = new Map()) => async (...args) => {
  const key = JSON.stringify(args);
  if (!cache.has(key)) cache.set(key, fn(...args));   // cache the PROMISE: dedupes in-flight calls
  return cache.get(key);
};

const fetchUser = withCache(withRetry(rawFetchUser));
```

*Earns its keep* for cross-cutting concerns: retry, cache, log, rate-limit, instrument.
Order matters — `withCache(withRetry(f))` caches the retried result; `withRetry(withCache(f))`
retries the cache lookup, which is nearly always wrong.

**Facade** — one simple interface over a messy subsystem.

```js
export const media = {
  async play(url) { await ensureCodec(url); await ensureAudioContext(); return player.start(url); },
  pause() { player.pause(); analytics.track('pause'); },
};
```

**Proxy** — intercept operations on an object. JavaScript has it natively.

```js
const observable = (target, onChange) => new Proxy(target, {
  set(obj, prop, value, receiver) {
    const previous = obj[prop];
    const ok = Reflect.set(obj, prop, value, receiver);   // always delegate via Reflect
    if (ok && previous !== value) onChange(prop, value, previous);
    return ok;
  },
  deleteProperty(obj, prop) {
    const ok = Reflect.deleteProperty(obj, prop);
    if (ok) onChange(prop, undefined, undefined);
    return ok;
  },
});

const state = observable({ count: 0 }, (k, v) => render(k, v));
state.count++;    // triggers render
```

*Earns its keep* for reactivity, validation, and negative caching. *Cost:* every property access
goes through a trap, so it's slower than a plain object, and it makes debugging indirect.

**Composite** — treat a single item and a group of items uniformly.

```js
class Node {
  constructor(name) { this.name = name; this.children = []; }
  add(child) { this.children.push(child); return this; }
  size() { return 1 + this.children.reduce((s, c) => s + c.size(), 0); }
}
```

---

## 30.4 Behavioural

**Observer / PubSub** — the most-used pattern in front-end code.

```js
export class Emitter {
  #listeners = new Map();     // event → Set<handler>

  on(event, handler) {
    if (!this.#listeners.has(event)) this.#listeners.set(event, new Set());
    this.#listeners.get(event).add(handler);
    return () => this.off(event, handler);     // return the unsubscribe, don't make callers store both
  }
  off(event, handler) { this.#listeners.get(event)?.delete(handler); }
  once(event, handler) {
    const wrapped = (...args) => { this.off(event, wrapped); handler(...args); };
    return this.on(event, wrapped);
  }
  emit(event, ...args) {
    // Copy first: a handler that unsubscribes during emit would otherwise mutate the live set.
    for (const handler of [...(this.#listeners.get(event) ?? [])]) {
      try { handler(...args); }
      catch (err) { queueMicrotask(() => { throw err; }); }   // one bad listener ≠ all listeners
    }
  }
}
```

Observer means the subject knows its observers; PubSub routes through a broker so neither knows the
other. The broker decouples more and debugs worse — you lose the ability to see who's listening.

**Strategy** — swap an algorithm at runtime. In JavaScript this is just a function parameter.

```js
const shipping = {
  standard: (w) => 5 + w * 0.5,
  express:  (w) => 15 + w * 1.2,
  free:     () => 0,
};
const cost = (weight, method) => (shipping[method] ?? shipping.standard)(weight);
```

*Earns its keep* over an `if`/`switch` chain when strategies are added often or come from
configuration. *Cheaper alternative:* a `switch`, for two or three fixed cases.

**Command** — an operation as an object, which is what makes undo possible.

```js
class CommandStack {
  #done = [];
  #undone = [];

  execute(command) {
    command.do();
    this.#done.push(command);
    this.#undone.length = 0;      // a new action invalidates the redo branch
  }
  undo() { const c = this.#done.pop(); if (c) { c.undo(); this.#undone.push(c); } }
  redo() { const c = this.#undone.pop(); if (c) { c.do(); this.#done.push(c); } }
}

const addShape = (canvas, shape) => ({
  label: `add ${shape.type}`,
  do:   () => canvas.shapes.push(shape),
  undo: () => canvas.shapes.splice(canvas.shapes.indexOf(shape), 1),
});
```

**State machine** — legal states and legal transitions, declared.

```js
const machine = {
  idle:    { FETCH: 'loading' },
  loading: { RESOLVE: 'success', REJECT: 'failure', ABORT: 'idle' },
  success: { FETCH: 'loading' },
  failure: { FETCH: 'loading', RESET: 'idle' },
};

function transition(state, event) {
  const next = machine[state]?.[event];
  if (!next) { console.warn(`Illegal transition: ${event} from ${state}`); return state; }
  return next;
}
```

*Earns its keep* the moment you have three or more booleans describing one thing — `isLoading`,
`hasError`, `isEmpty` allow eight combinations, half of them impossible. A machine makes the
impossible ones unrepresentable.

**Iterator** — native, via `Symbol.iterator`.

```js
class Range {
  constructor(from, to, step = 1) { Object.assign(this, { from, to, step }); }
  *[Symbol.iterator]() { for (let i = this.from; i < this.to; i += this.step) yield i; }
}
[...new Range(0, 10, 3)];    // [0, 3, 6, 9]
```

**Chain of Responsibility** — middleware. Each link handles the request or passes it on.

```js
function createPipeline() {
  const middleware = [];
  return {
    use(fn) { middleware.push(fn); return this; },
    async run(context) {
      let index = -1;
      const dispatch = async (i) => {
        if (i <= index) throw new Error('next() called twice');   // the classic middleware bug
        index = i;
        const fn = middleware[i];
        if (fn) await fn(context, () => dispatch(i + 1));
      };
      await dispatch(0);
      return context;
    },
  };
}

createPipeline()
  .use(async (ctx, next) => { const t = performance.now(); await next(); ctx.ms = performance.now() - t; })
  .use(async (ctx, next) => { if (!ctx.user) throw new Error('401'); await next(); })
  .use(async (ctx) => { ctx.body = await loadData(ctx); });
```

**Template Method** — a fixed algorithm with pluggable steps. In JavaScript, pass the steps in:

```js
const createImporter = ({ parse, validate, transform = (x) => x }) => async (file) => {
  const rows = parse(await file.text());
  const { valid, errors } = validate(rows);
  return { records: valid.map(transform), errors };
};
```

---

## 30.5 JavaScript-Specific Patterns

**Module / Revealing Module** — private by closure, public by return.

```js
export const counter = (() => {
  let count = 0;                                // genuinely private
  const increment = () => ++count;
  return { increment, get value() { return count; } };
})();
```

The IIFE form predates ESM. In new code, a module file *is* the module pattern; use `#private`
fields for per-instance privacy.

**Mixin** — share behaviour without a class hierarchy.

```js
const Serializable = (Base) => class extends Base {
  toJSON() { return { ...this, type: this.constructor.name }; }
};
const Comparable = (Base) => class extends Base {
  equals(other) { return JSON.stringify(this) === JSON.stringify(other); }
};

class Point extends Serializable(Comparable(Object)) {
  constructor(x, y) { super(); Object.assign(this, { x, y }); }
}
```

**Dependency Injection** — pass collaborators in rather than importing them.

```js
// ❌ Untestable: the dependency is baked in
import { db } from './db.js';
export const getUser = (id) => db.query('…', id);

// ✅ Injected: pass a fake in tests, no module mocking needed
export const createUserService = ({ db, cache, clock = Date }) => ({
  async get(id) {
    return cache.get(id) ?? cache.set(id, await db.query('…', id));
  },
  stamp() { return clock.now(); },
});
```

Injecting `clock` looks fussy until you need to test expiry without waiting an hour.

**Repository** — one place that knows how data is stored.

```js
export const createUserRepo = ({ db }) => ({
  findById:  (id) => db.query('SELECT * FROM users WHERE id = ?', [id]).then(rowToUser),
  findByEmail: (email) => db.query('SELECT * FROM users WHERE email = ?', [email]).then(rowToUser),
  save: (user) => db.query('INSERT INTO users … ON CONFLICT UPDATE', userToRow(user)),
});
```

The point isn't swapping databases — you won't. It's that SQL exists in one file, so a schema
change touches one file.

---

## 30.6 Common Mistakes and Best Practices

**Mistake: a class-based Singleton where a module export would do.** Extra ceremony, same result,
harder to test.

**Mistake: an abstraction layer for a dependency you'll never swap.** A Repository over one
database is fine because it centralises queries; a Repository over `localStorage` "in case we
change" is speculative cost.

**Mistake: wrong decorator order.** `withRetry(withCache(f))` retries the cache read. Compose
inside-out and think about what each layer sees.

**Mistake: `emit` iterating the live listener set.** A handler that unsubscribes during dispatch
mutates the collection you're iterating. Copy first.

**Mistake: one listener's throw killing the rest.** Isolate each handler.

**Mistake: booleans instead of a state machine.** Three booleans, eight states, four of them
impossible, and a bug in every one.

**Mistake: Observer without unsubscribe.** Every subscription is a leak until proven otherwise
(Chapter 27). Return the unsubscribe function from `on`.

**Best practice: start with a function.** Reach for a class when you need multiple instances with
shared behaviour, and for a pattern when you have the problem twice.

**Best practice: inject dependencies at the module boundary.** The `createXService({ deps })` shape
costs one line and removes all module mocking from your tests.

**Best practice: name the pattern in a comment or the type name, not in every identifier.**
`UserRepositoryFactoryStrategyImpl` is not communication.

---

## 30.7 Real-World Example: A Small App Wiring Repository, Strategy, and Observer

Patterns are worth learning because they compose. Here's a task list where each pattern does one
job: the repository owns persistence, strategies own sorting, and the store notifies observers.

```js
// ── Repository: the only code that knows how tasks are stored ──────────────────────
export const createTaskRepo = ({ storage, idFactory = () => crypto.randomUUID() }) => {
  const KEY = 'tasks:v1';
  const load = () => storage.get(KEY) ?? [];

  return {
    all: () => load(),
    add(fields) {
      const task = { id: idFactory(), done: false, createdAt: Date.now(), ...fields };
      storage.set(KEY, [...load(), task]);
      return task;
    },
    update(id, changes) {
      const next = load().map(t => (t.id === id ? { ...t, ...changes } : t));
      storage.set(KEY, next);
      return next.find(t => t.id === id);
    },
    remove(id) { storage.set(KEY, load().filter(t => t.id !== id)); },
  };
};

// ── Strategy: sorting, as data. Adding one is a new entry, not a new branch. ───────
export const sortStrategies = {
  created:  (a, b) => b.createdAt - a.createdAt,
  alpha:    (a, b) => a.title.localeCompare(b.title),
  priority: (a, b) => (b.priority ?? 0) - (a.priority ?? 0),
  // Composite strategy: incomplete first, then by whatever the user picked.
  pending:  (inner) => (a, b) => Number(a.done) - Number(b.done) || inner(a, b),
};

// ── Store: Observer over repository state, plus the current view strategy ──────────
export const createStore = ({ repo }) => {
  const emitter = new Emitter();
  let view = { sort: 'created', pendingFirst: true, filter: '' };

  const visible = () => {
    const base = sortStrategies[view.sort] ?? sortStrategies.created;
    const compare = view.pendingFirst ? sortStrategies.pending(base) : base;
    const needle = view.filter.toLowerCase();
    return repo.all()
      .filter(t => !needle || t.title.toLowerCase().includes(needle))
      .toSorted(compare);                 // non-mutating: repo.all() may return a live array
  };

  // Every mutation goes through here, so there is exactly one place that notifies.
  const commit = (label) => { emitter.emit('change', { tasks: visible(), view, label }); };

  return {
    subscribe: (fn) => {
      const off = emitter.on('change', fn);
      fn({ tasks: visible(), view, label: 'init' });   // emit current state immediately
      return off;
    },
    add(title, priority = 0) { repo.add({ title, priority }); commit('add'); },
    toggle(id) { const t = repo.all().find(t => t.id === id); repo.update(id, { done: !t.done }); commit('toggle'); },
    remove(id) { repo.remove(id); commit('remove'); },
    setView(changes) { view = { ...view, ...changes }; commit('view'); },
  };
};

// ── Wiring: dependencies injected once, at the edge ────────────────────────────────
const store = createStore({ repo: createTaskRepo({ storage: new Store('tasks') }) });

const unsubscribe = store.subscribe(({ tasks, view }) => {
  list.replaceChildren(...tasks.map(renderTask));
  count.textContent = `${tasks.filter(t => !t.done).length} pending`;
  sortSelect.value = view.sort;
});
```

The decision worth explaining is that `subscribe` immediately invokes the new subscriber with
current state rather than waiting for the next change. Without it, every consumer needs an initial
render call alongside its subscription, which is two places that must agree on how to render and
one of them will drift. It also means a subscriber added after a mutation isn't stale. The cost is
that `subscribe` is no longer a pure registration — it has an effect — so it must never be called
during an `emit`, which the copy-before-iterate in `Emitter.emit` makes safe rather than merely
unlikely.

---

## 30.8 Questions

**1.** MCQ — In JavaScript, the simplest Singleton is: a) a class with a static instance b) a module export c) a frozen global d) an IIFE
<details><summary>Answer</summary>**b)** Modules are evaluated once and cached.</details>

**2.** MCQ — Strategy in JavaScript is usually: a) a class hierarchy b) a passed function c) a Proxy d) a mixin
<details><summary>Answer</summary>**b)**</details>

**3.** MCQ — Which pattern makes undo possible? a) Observer b) Command c) Facade d) Adapter
<details><summary>Answer</summary>**b)** It reifies the operation, so it can carry its own inverse.</details>

**4.** MCQ — `withRetry(withCache(fn))` will: a) cache the retried result b) retry the cache lookup c) do both correctly d) throw
<details><summary>Answer</summary>**b)** Which is almost always wrong — cache is the outer layer you want.</details>

**5.** MCQ — Middleware chains implement: a) Composite b) Chain of Responsibility c) Visitor d) Bridge
<details><summary>Answer</summary>**b)**</details>

**6.** MCQ — Observer differs from PubSub in that: a) it's synchronous b) the subject knows its observers c) it can't unsubscribe d) it uses classes
<details><summary>Answer</summary>**b)** PubSub routes through a broker, so publisher and subscriber don't know each other.</details>

**7.** Short — Why does a `Proxy` cost performance?
<details><summary>Answer</summary>Every intercepted operation goes through a trap function rather than the engine's optimised fast path, so property access can't be inline-cached the way it is on a plain object.</details>

**8.** Short — When does a state machine beat boolean flags?
<details><summary>Answer</summary>As soon as there are three or more flags describing one thing. N booleans allow 2^N combinations, most of them impossible; a machine enumerates only the legal states and transitions.</details>

**9.** Short — Why return the unsubscribe function from `on`?
<details><summary>Answer</summary>It removes the need for the caller to keep both the event name and the exact handler reference, which is where listener leaks come from. One value to hold, one call to clean up.</details>

**10.** Short — What does Dependency Injection buy in a language with module mocking?
<details><summary>Answer</summary>Tests that need no mocking framework and no module registry hacks: pass a plain object. It also makes dependencies visible in the signature rather than hidden in imports.</details>

**11.** Short — Why copy the listener set before iterating in `emit`?
<details><summary>Answer</summary>A handler may call `off` — including its own `once` wrapper — mutating the collection mid-iteration and skipping listeners or throwing.</details>

**12.** Short — What is the real benefit of the Repository pattern?
<details><summary>Answer</summary>Query code lives in one file, so a schema change has one blast radius. Database portability is the advertised benefit and almost never the realised one.</details>

**13.** Output —
```js
const e = new Emitter();
const off = e.on('x', () => console.log('a'));
e.on('x', () => { off(); console.log('b'); });
e.emit('x'); e.emit('x');
```
<details><summary>Answer</summary>`a b` then `b` — the first emit runs both from the copied set, and `a` is gone by the second.</details>

**14.** Output —
```js
const withA = (f) => (...a) => { console.log('A'); return f(...a); };
const withB = (f) => (...a) => { console.log('B'); return f(...a); };
withA(withB(() => console.log('core')))();
```
<details><summary>Answer</summary>`A B core` — the outermost decorator runs first.</details>

**15.** Output —
```js
const p = new Proxy({}, { get: (t, k) => (k in t ? t[k] : `no ${String(k)}`) });
p.a = 1;
console.log(p.a, p.b);
```
<details><summary>Answer</summary>`1 'no b'` — the `set` is untrapped so it lands normally; the missing key hits the `get` trap.</details>

**16.** Output —
```js
class A { greet() { return 'A'; } }
const M = (B) => class extends B { greet() { return `M(${super.greet()})`; } };
console.log(new (M(A))().greet());
```
<details><summary>Answer</summary>`M(A)` — the mixin's `super` resolves to the class it wrapped.</details>

**17.** Debug — A dropdown's outside-click handler accumulates: after ten opens, one click logs ten times.
<details><summary>Answer</summary>Each open subscribes without unsubscribing on close. Return and call the unsubscribe function, or register with an `AbortController` signal aborted on close.</details>

**18.** Debug — Middleware sometimes runs the final handler twice.
<details><summary>Answer</summary>A middleware calls `next()` more than once — commonly `await next()` inside a loop or after an early return that wasn't returned. Guard with the index check shown in §30.4, which converts a silent double-execution into a thrown error.</details>

**19.** Debug — A Singleton connection works in the app and breaks tests that run in parallel.
<details><summary>Answer</summary>The instance is global mutable state shared across tests, so one test's connection state leaks into another. Convert to a factory with injected configuration and create one per test, keeping a module-level default for production if convenient.</details>

**20.** Debug — Adding an "urgency" sort option required edits in five places.
<details><summary>Answer</summary>Sorting is an `if`/`switch` chain duplicated across components rather than a Strategy. Move the comparators into one exported map keyed by name; adding a sort then becomes one entry.</details>

**21.** Debug — An undo stack restores the shape but not its position.
<details><summary>Answer</summary>The command captures a reference to the mutable object rather than the values it changed, so `undo` restores whatever the object holds now. Capture the previous values at `do()` time and restore those.</details>

**22.** Code — Implement `once(fn)` and `memoizeOne(fn)` as decorators.
<details><summary>Answer</summary>

```js
const once = (fn) => { let done = false, value;
  return (...a) => done ? value : (done = true, value = fn(...a)); };

const memoizeOne = (fn) => { let lastArgs, lastValue, has = false;
  return (...a) => {
    if (has && a.length === lastArgs.length && a.every((x, i) => x === lastArgs[i])) return lastValue;
    lastArgs = a; has = true;
    return (lastValue = fn(...a));
  };
};
```
</details>

**23.** Code — Write a state machine runner with entry actions.
<details><summary>Answer</summary>

```js
export function createMachine({ initial, states }) {
  let current = initial;
  const emitter = new Emitter();

  return {
    get state() { return current; },
    can: (event) => Boolean(states[current]?.on?.[event]),
    send(event, payload) {
      const next = states[current]?.on?.[event];
      if (!next) return false;
      states[current]?.onExit?.(payload);
      current = next;
      states[current]?.onEntry?.(payload);
      emitter.emit('transition', { state: current, event, payload });
      return true;
    },
    subscribe: (fn) => emitter.on('transition', fn),
  };
}

const fetcher = createMachine({
  initial: 'idle',
  states: {
    idle:    { on: { FETCH: 'loading' } },
    loading: { on: { RESOLVE: 'success', REJECT: 'failure' }, onEntry: () => showSpinner() },
    success: { on: { FETCH: 'loading' }, onExit: () => clearData() },
    failure: { on: { FETCH: 'loading', RESET: 'idle' } },
  },
});
```
</details>

**24.** Code — Write an `Adapter` that gives `localStorage` and IndexedDB the same async interface.
<details><summary>Answer</summary>

```js
const localAdapter = () => ({
  async get(k) { return JSON.parse(localStorage.getItem(k) ?? 'null'); },
  async set(k, v) { localStorage.setItem(k, JSON.stringify(v)); },
  async delete(k) { localStorage.removeItem(k); },
});

const idbAdapter = (db, store = 'kv') => ({
  async get(k) { return promisify(db.transaction(store).objectStore(store).get(k)); },
  async set(k, v) {
    const tx = db.transaction(store, 'readwrite');
    tx.objectStore(store).put(v, k);
    return promisify(tx);
  },
  async delete(k) {
    const tx = db.transaction(store, 'readwrite');
    tx.objectStore(store).delete(k);
    return promisify(tx);
  },
});
```
Both interfaces are async even where the underlying call isn't, so callers never need to know which
one they have.</details>

**25.** Scenario — A reviewer says your module needs a Factory, an Abstract Factory, and a Builder. How do you respond?
<details><summary>Answer</summary>Ask which problem each one solves in this module. A Factory is justified if construction involves a decision callers shouldn't repeat; a Builder if there are many interdependent optional parts; an Abstract Factory only if you genuinely have families of related products to swap, which is rare outside frameworks. If the answer is "for flexibility", the flexibility already exists — JavaScript functions return objects, so any of these can be introduced later in one commit without changing call sites. Ship the function; add the pattern when a second caller needs it.</details>

**26.** Scenario — Your team's app has 40 event names on a global emitter and nobody knows who listens to what.
<details><summary>Answer</summary>This is the predictable failure mode of PubSub: total decoupling means no discoverable call graph. Move to direct subscriptions where there's a real owner — a component subscribing to a store it holds a reference to — so "find all references" works again. Where a global bus is genuinely right (cross-cutting concerns like auth expiry), define the event names in one typed module so at least the vocabulary is enumerable, and add a development-mode listener registry you can dump. The underlying fix is usually fewer, coarser events carrying more data, not better tooling for 40 fine-grained ones.</details>

---

## Chapter Summary

- Several classic patterns collapse to a function or a module in JavaScript; know which, and don't rebuild the C++ version.
- Apply a pattern when you have its problem twice — the name is for communication, not justification.
- Decorator order matters: cache outside retry, not the reverse.
- Copy the listener set before dispatch, isolate each handler's errors, and always return an unsubscribe.
- Three booleans describing one thing means you wanted a state machine.
- Inject dependencies at the boundary; it costs one line and deletes all your module mocking.

---

## Practice Exercises

1. Refactor an `if`/`else` chain of five branches into a Strategy map and note what each new case now costs.
2. Implement `withRateLimit`, `withTimeout`, and `withLogging` decorators, then compose all three and reason about the order.
3. Convert a component using four boolean flags into an explicit state machine and list the states that were previously reachable but invalid.
4. Build a middleware pipeline with error-handling middleware that catches downstream throws.
5. Take a module with three hard-coded imports and convert it to dependency injection; delete the mocking setup from its tests.

---

## Mini Project: Undo/Redo for a Drawing App

Build a canvas drawing tool where every action is a command and history is fully reversible.

- Commands for draw, move, delete, restyle, and group, each with `do`, `undo`, and a `label`
- A bounded history (50 entries) that discards the oldest, and clears redo on any new action
- Coalescing: a continuous drag produces one undoable command, not one per `pointermove`
- A visible history panel listing labels, with click-to-jump to any point
- Serialise the command log to JSON and replay it into an identical canvas
- A Strategy map for tools (pen, rectangle, ellipse, eraser) so adding one is a single entry
- An Observer so the panel, canvas, and undo buttons all update from one notification

---

**Next:** [Chapter 31 — Security →](31-security.md)
