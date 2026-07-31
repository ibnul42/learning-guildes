# Chapter 12 — Classes and OOP

## Learning Objectives

- Write classes with fields, private members, statics, getters, and setters
- Use `extends` and `super` correctly
- Know exactly what `class` is and isn't sugar for
- Choose between inheritance and composition

---

## 12.1 Class Syntax

```js
class User {
  // Public instance field — initialized before the constructor body runs
  role = 'member';

  // Private field — enforced by the engine, not a convention
  #password;

  // Static field
  static count = 0;

  // Private static
  static #instances = new Set();

  constructor(name, password) {
    this.name = name;
    this.#password = hash(password);
    User.count++;
    User.#instances.add(this);
  }

  // Prototype method — shared by all instances
  greet() { return `Hi, ${this.name}`; }

  // Getter / setter
  get displayName() { return this.name.toUpperCase(); }
  set displayName(v) {
    if (!v?.trim()) throw new TypeError('name required');
    this.name = v.trim();
  }

  // Private method
  #verify(pw) { return hash(pw) === this.#password; }
  login(pw) { return this.#verify(pw); }

  // Static method — `this` is the class
  static fromJSON(json) { return new this(json.name, json.password); }

  // Class-field arrow — bound to the instance, one per instance
  handleClick = () => console.log(this.name);

  // Static initialization block (ES2022)
  static { /* runs once when the class is defined */ }
}
```

**Key facts:**

- Class bodies are **always strict mode**.
- Class declarations are **hoisted into the TDZ** — not usable before their definition.
- Classes must be called with `new`; `User('x')` throws a `TypeError`.
- Field initializers run in order, **before** the constructor body, and after `super()` in a subclass.

### Private fields

```js
class Wallet {
  #balance = 0;
  deposit(n) { this.#balance += n; }
  static isWallet(o) { return #balance in o; }   // ergonomic brand check (ES2022)
}
const w = new Wallet();
w.#balance;              // ❌ SyntaxError at parse time — not even runtime
Object.keys(w);          // []  — invisible to reflection, JSON, spread
```

`#private` is genuinely inaccessible: not via bracket notation, not via `Object.getOwnPropertyNames`, not via a `Proxy`. Compare with `_convention`, which is just a hint.

---

## 12.2 Inheritance

```js
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound`; }
  static create(n) { return new this(n); }   // `this` = the subclass when called on one
}

class Dog extends Animal {
  #tricks = [];

  constructor(name, breed) {
    super(name);            // MUST be called before touching `this`
    this.breed = breed;
  }

  speak() {
    return `${super.speak()} — a bark`;   // call the parent's version
  }
}

const d = Dog.create('Rex');   // a Dog, not an Animal — `this` in statics is the subclass
d instanceof Dog && d instanceof Animal;   // true
```

**Rules for `super`:**

- In a derived constructor, `this` is in a TDZ until `super()` runs. Reading `this` first is a `ReferenceError`.
- If a derived class has no constructor, an implicit `constructor(...args) { super(...args); }` is used.
- `super.method()` works in any method, including statics (where it reaches the parent class).

Extending built-ins works:

```js
class ValidationError extends Error {
  constructor(field, message) {
    super(message);
    this.name = 'ValidationError';
    this.field = field;
  }
}
class Stack extends Array {
  peek() { return this.at(-1); }
}
// Note: Array methods like map return a Stack, via Symbol.species
```

---

## 12.3 What `class` Is (and Isn't) Sugar For

`class` is *mostly* syntax over prototypes. Methods land on `.prototype`, statics on the constructor.

```js
class C { m() {} }
Object.getOwnPropertyNames(C.prototype);   // ['constructor', 'm']
typeof C;                                   // 'function'
```

But these differ from function constructors:

| | `class` | `function` constructor |
|---|---|---|
| Callable without `new` | ❌ TypeError | ✅ (and pollutes `this`) |
| Hoisted usable | ❌ TDZ | ✅ |
| Strict mode | always | only if declared |
| Methods enumerable | ❌ no | ✅ yes |
| Private fields | ✅ `#x` | ❌ (closures only) |
| `super` | ✅ | ❌ |

So: sugar, but with real added semantics. Saying "classes are just functions" in an interview is half right — say "they're functions with additional restrictions and capabilities."

---

## 12.4 Composition Over Inheritance

Deep hierarchies become brittle. Two alternatives:

**Mixins:**

```js
const Serializable = (Base) => class extends Base {
  toJSON() { return { ...this, type: this.constructor.name }; }
};
const Timestamped = (Base) => class extends Base {
  createdAt = new Date();
};

class Document extends Serializable(Timestamped(Object)) {}
```

**Plain composition — usually better:**

```js
class Order {
  constructor({ pricing, notifier, repository }) {
    this.pricing = pricing;
    this.notifier = notifier;
    this.repository = repository;
  }
  async place(items) {
    const total = this.pricing.calculate(items);
    const saved = await this.repository.save({ items, total });
    await this.notifier.send(saved);
    return saved;
  }
}
```

Dependencies are injected, so each collaborator can be swapped or mocked. No hierarchy at all.

> 💼 **Interview** — "When would you use inheritance?" When there's a genuine *is-a* relationship with shared behaviour and a stable base — `HttpError extends Error`, `Circle extends Shape`. Reach for composition for *has-a* and for anything likely to change.

---

## 12.5 Common Mistakes and Best Practices

**Mistake: using `this` before `super()`.** ReferenceError.

**Mistake: forgetting `new`.** `const u = User('a')` throws — good, but the error message confuses beginners.

**Mistake: arrow class fields for every method.** They aren't on the prototype, so subclasses can't override them via `super`, and each instance allocates a new function.

**Mistake: deep inheritance chains.** Three levels is usually already too many.

**Mistake: exposing mutable internals.** `get items() { return this.#items; }` hands out the live array. Return a copy or a read-only view.

**Best practice: validate in the constructor.** An object should never exist in an invalid state.

**Best practice: `#private` by default**, and expose only what callers need.

**Best practice: static factory methods** (`User.fromJSON`, `Money.fromDecimal`) read better than overloaded constructors.

---

## 12.6 Real-World Example: A Domain Model

```js
class Entity {
  #id;
  constructor(id = crypto.randomUUID()) { this.#id = id; }
  get id() { return this.#id; }
  equals(other) { return other instanceof Entity && other.id === this.#id; }
}

class Cart extends Entity {
  #lines = new Map();          // sku -> { sku, qty, unitCents }
  #maxLines;

  constructor({ id, maxLines = 50 } = {}) {
    super(id);
    this.#maxLines = maxLines;
  }

  add(sku, qty, unitCents) {
    if (!Number.isInteger(qty) || qty < 1) throw new RangeError('qty must be >= 1');
    if (!this.#lines.has(sku) && this.#lines.size >= this.#maxLines) {
      throw new RangeError(`Cart limited to ${this.#maxLines} distinct items`);
    }
    const existing = this.#lines.get(sku);
    this.#lines.set(sku, { sku, unitCents, qty: (existing?.qty ?? 0) + qty });
    return this;                              // chainable
  }

  remove(sku) { this.#lines.delete(sku); return this; }

  // Read-only view — callers cannot mutate internal state
  get lines() { return Object.freeze([...this.#lines.values()].map(l => ({ ...l }))); }
  get totalCents() {
    return [...this.#lines.values()].reduce((t, l) => t + l.qty * l.unitCents, 0);
  }
  get isEmpty() { return this.#lines.size === 0; }

  toJSON() { return { id: this.id, lines: this.lines, totalCents: this.totalCents }; }

  static fromJSON({ id, lines }) {
    const cart = new Cart({ id });
    for (const l of lines) cart.add(l.sku, l.qty, l.unitCents);
    return cart;
  }
}
```

Every invariant (positive quantity, line limit) is enforced at the boundary. `#lines` cannot be reached, and `lines` returns frozen copies, so no caller can corrupt the cart.

---

## 12.7 Questions

**1.** MCQ — Class declarations are: a) hoisted and usable b) hoisted into the TDZ c) not hoisted d) function-scoped
<details><summary>Answer</summary>**b)**</details>

**2.** MCQ — Calling a class without `new`: a) works b) returns undefined c) throws TypeError d) returns the class
<details><summary>Answer</summary>**c)**</details>

**3.** MCQ — `#private` fields are accessible via: a) bracket notation b) `Object.keys` c) a Proxy d) none of these
<details><summary>Answer</summary>**d)**</details>

**4.** MCQ — In a derived constructor, `this` before `super()`: a) is `undefined` b) throws ReferenceError c) is the parent d) works fine
<details><summary>Answer</summary>**b)**</details>

**5.** MCQ — Class methods are: a) enumerable b) non-enumerable c) private d) static
<details><summary>Answer</summary>**b)** — unlike methods assigned to a function's prototype.</details>

**6.** MCQ — `this` in a static method is: a) the instance b) the class c) undefined d) globalThis
<details><summary>Answer</summary>**b)**</details>

**7.** Short — Is `class` purely syntactic sugar?
<details><summary>Answer</summary>No. It adds enforced `new`, TDZ hoisting, implicit strict mode, non-enumerable methods, `super`, and true private fields.</details>

**8.** Short — Why must `super()` come first?
<details><summary>Answer</summary>The parent constructor creates and initializes the instance; `this` doesn't exist in the derived constructor until it returns.</details>

**9.** Short — `#private` vs `_private`?
<details><summary>Answer</summary>`#` is enforced at parse/runtime and invisible to all reflection. `_` is a naming convention with no enforcement.</details>

**10.** Short — When is inheritance the right tool?
<details><summary>Answer</summary>A true *is-a* relationship over a stable base with genuinely shared behaviour — e.g. custom error subclasses.</details>

**11.** Short — What does a static initialization block do?
<details><summary>Answer</summary>Runs once at class-definition time with `this` bound to the class, for setup that can't be expressed as a single field initializer (including access to private statics).</details>

**12.** Output — What prints?
```js
class A { constructor() { this.x = 1; } }
class B extends A { constructor() { super(); this.x = 2; } }
console.log(new B().x);
```
<details><summary>Answer</summary>`2`</details>

**13.** Output — What prints?
```js
class C { static n = 0; constructor() { C.n++; } }
new C(); new C();
console.log(C.n);
```
<details><summary>Answer</summary>`2`</details>

**14.** Output — What prints?
```js
class P { static who() { return this.name; } }
class Q extends P {}
console.log(Q.who());
```
<details><summary>Answer</summary>`'Q'` — `this` in a static is the class it was called on.</details>

**15.** Output — What prints?
```js
class A { m() { return 'A'; } }
class B extends A { m() { return super.m() + 'B'; } }
console.log(new B().m());
```
<details><summary>Answer</summary>`'AB'`</details>

**16.** Output — What prints?
```js
class X { #v = 1; static has(o) { return #v in o; } }
console.log(X.has(new X()), X.has({}));
```
<details><summary>Answer</summary>`true false`</details>

**17.** Output — What prints?
```js
class Y { a = this.b; b = 2; }
console.log(new Y().a);
```
<details><summary>Answer</summary>`undefined` — field initializers run top to bottom, so `b` isn't set yet.</details>

**18.** Debug — "Cannot read properties of undefined" in a subclass constructor.
<details><summary>Answer</summary>`this` used before `super()`. Move `super(...)` to the first statement.</details>

**19.** Debug — A cart's total goes negative after a caller does `cart.lines.push(...)`.
<details><summary>Answer</summary>The getter returned the live internal array. Return a frozen copy (as in §12.6).</details>

**20.** Debug — A subclass's overridden method is never called from the parent.
<details><summary>Answer</summary>The parent's version was defined as a class-field arrow, so it lives on the *instance* and shadows the prototype method the subclass defined. Make it a regular prototype method.</details>

**21.** Code — Write an `Observable` base class with `subscribe`, `unsubscribe`, and `notify`, private subscriber storage, and safe iteration during notification.
<details><summary>Answer</summary>

```js
class Observable {
  #subs = new Set();
  subscribe(fn) { this.#subs.add(fn); return () => this.#subs.delete(fn); }
  unsubscribe(fn) { this.#subs.delete(fn); }
  notify(data) {
    for (const fn of [...this.#subs]) {   // copy: a handler may unsubscribe
      try { fn(data); } catch (e) { queueMicrotask(() => { throw e; }); }
    }
  }
}
```
</details>

**22.** Code — Create a `Temperature` class storing kelvin privately, with `celsius`/`fahrenheit` accessors and validation.
<details><summary>Answer</summary>

```js
class Temperature {
  #kelvin;
  constructor(kelvin) { this.kelvin = kelvin; }
  set kelvin(k) {
    if (!Number.isFinite(k) || k < 0) throw new RangeError('below absolute zero');
    this.#kelvin = k;
  }
  get kelvin() { return this.#kelvin; }
  get celsius() { return this.#kelvin - 273.15; }
  set celsius(c) { this.kelvin = c + 273.15; }
  get fahrenheit() { return this.celsius * 9/5 + 32; }
  set fahrenheit(f) { this.celsius = (f - 32) * 5/9; }
  static fromCelsius(c) { const t = new this(0); t.celsius = c; return t; }
}
```
</details>

**23.** Code — Write a `Singleton` base class such that any subclass gets one instance.
<details><summary>Answer</summary>

```js
class Singleton {
  static #instances = new Map();
  constructor() {
    const existing = Singleton.#instances.get(new.target);
    if (existing) return existing;         // constructors may return an object
    Singleton.#instances.set(new.target, this);
  }
}
```
`new.target` gives the actual subclass being constructed.
</details>

**24.** Scenario — Your `Vehicle → Car → SportsCar → ElectricSportsCar` hierarchy needs an electric truck. What now?
<details><summary>Answer</summary>The hierarchy encodes two orthogonal axes (body type and powertrain) as one chain, so it can't express the combination. Refactor to composition: a `Vehicle` holding a `Powertrain` and a `Body`. This is the diamond problem inheritance can't solve.</details>

**25.** Scenario — A reviewer says private fields hurt testability. Response?
<details><summary>Answer</summary>Test through the public interface — if a private is untestable that way, it's usually a sign the class does too much and the private logic wants to be its own module. Reaching into internals in tests couples them to implementation and makes refactoring painful.</details>

---

## Chapter Summary

- Classes add enforced `new`, TDZ hoisting, strict mode, non-enumerable methods, `super`, and real `#private` fields on top of prototypes.
- `super()` must precede any use of `this` in a derived constructor.
- Field initializers run in order, before the constructor body.
- `this` in statics is the class the method was called on — subclass-aware factories work for free.
- Prefer composition and dependency injection; use inheritance only for stable *is-a* relationships.
- Never leak mutable internals from a getter.

---

## Practice Exercises

1. Convert the Chapter 11 constructor-function hierarchy to classes; list every behavioural difference.
2. Build a `LinkedList` class with private nodes and an iterator (see Chapter 14).
3. Write a `Result`/`Either` class with `map`, `flatMap`, and `unwrapOr`.
4. Implement a mixin that adds `EventEmitter` behaviour to any class.
5. Write a class that enforces at most three instances, throwing on the fourth.

---

## Mini Project: A Shape Hierarchy and Renderer

Build `shapes.js` with an abstract `Shape` base and `Circle`, `Rectangle`, `Triangle`, and `Composite` subclasses.

**Requirements:**

- `Shape` throws if instantiated directly (`if (new.target === Shape) throw ...`).
- Each subclass implements `area()`, `perimeter()`, and `path()` (returning SVG path data).
- All dimensions private and validated; shapes immutable after construction.
- `Composite` holds child shapes and sums their areas; it must handle nesting.
- A `Renderer` class (composition, not inheritance) takes any `Shape` and produces SVG.
- `toJSON`/`fromJSON` round-trip including composites.

**Stretch:** add a `Transform` decorator class wrapping any shape with translate/rotate/scale.

---

**Next:** [Chapter 13 — Destructuring, Spread, Rest, and Modern Operators →](13-destructuring-spread-rest.md)
