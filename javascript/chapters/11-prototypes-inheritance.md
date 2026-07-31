# Chapter 11 — Prototypes and Inheritance

## Learning Objectives

- Explain the prototype chain and how property lookup works
- Distinguish `__proto__`, `[[Prototype]]`, and `.prototype`
- Implement inheritance with `Object.create` and constructor functions
- Recognise prototype pollution and shadowing bugs

---

## 11.1 The Prototype Chain

Every object has a hidden internal slot `[[Prototype]]` pointing to another object or `null`. When you read a property, the engine walks this chain until it finds the key or hits `null`.

```js
const animal = { eats: true, speak() { return '...'; } };
const dog = Object.create(animal);   // dog.[[Prototype]] === animal
dog.barks = true;

dog.barks;   // true  — own property
dog.eats;    // true  — found on animal
dog.missing; // undefined — chain exhausted

Object.getPrototypeOf(dog) === animal;   // true
Object.hasOwn(dog, 'eats');              // false — inherited, not own
'eats' in dog;                            // true  — `in` checks the whole chain
```

The full chain for a plain object:

```
{}  →  Object.prototype  →  null
[]  →  Array.prototype  →  Object.prototype  →  null
fn  →  Function.prototype  →  Object.prototype  →  null
```

That's why every object has `toString` — it's inherited from `Object.prototype`.

### Writing never walks the chain

```js
const parent = { x: 1 };
const child = Object.create(parent);
child.x = 99;             // creates an OWN property; parent is untouched
parent.x;                 // 1
delete child.x;
child.x;                  // 1 — the inherited one is visible again
```

This is **shadowing**. Reads walk up; writes always land on the object itself.

> ⚠️ One exception: if the prototype has a **setter** for that key, the setter runs instead of creating an own property.

---

## 11.2 `.prototype` vs `[[Prototype]]`

These are different things with confusingly similar names.

- **`[[Prototype]]`** — the internal link every object has. Read it with `Object.getPrototypeOf(obj)`. Exposed legacily as `obj.__proto__`.
- **`.prototype`** — a normal property that exists only on **functions**. It's the object that will become the `[[Prototype]]` of instances created with `new`.

```js
function Dog(name) { this.name = name; }
Dog.prototype.speak = function () { return `${this.name} barks`; };

const rex = new Dog('Rex');

Object.getPrototypeOf(rex) === Dog.prototype;   // true
rex.__proto__ === Dog.prototype;                 // true (same thing, legacy syntax)
rex.prototype;                                    // undefined — instances have no .prototype
Dog.prototype.constructor === Dog;               // true
```

What `new Dog('Rex')` does, step by step:

1. Create a new empty object.
2. Set its `[[Prototype]]` to `Dog.prototype`.
3. Call `Dog` with `this` bound to that object.
4. Return that object — unless the constructor explicitly returns a different *object*.

```js
function myNew(Ctor, ...args) {
  const obj = Object.create(Ctor.prototype);
  const result = Ctor.apply(obj, args);
  return (result !== null && typeof result === 'object') ? result : obj;
}
```

---

## 11.3 Inheritance Before `class`

```js
function Animal(name) { this.name = name; }
Animal.prototype.speak = function () { return `${this.name} makes a sound`; };

function Dog(name, breed) {
  Animal.call(this, name);          // 1. inherit instance properties
  this.breed = breed;
}
Dog.prototype = Object.create(Animal.prototype);   // 2. inherit methods
Dog.prototype.constructor = Dog;                    // 3. restore constructor
Dog.prototype.speak = function () {                 // 4. override
  return `${Animal.prototype.speak.call(this)} — a bark`;
};

const d = new Dog('Rex', 'Lab');
d.speak();                 // 'Rex makes a sound — a bark'
d instanceof Dog;          // true
d instanceof Animal;       // true
```

All four steps are required. Chapter 12 shows how `class` collapses this into `extends` and `super`.

> ⚠️ Never write `Dog.prototype = Animal.prototype` — they'd be the *same object*, so adding a method to `Dog` would add it to `Animal` too.

---

## 11.4 Checking Types

```js
d instanceof Dog;                              // walks the chain against Dog.prototype
Dog.prototype.isPrototypeOf(d);                // same idea, explicit
Object.getPrototypeOf(d) === Dog.prototype;    // direct-parent check only
d.constructor === Dog;                          // fragile — constructor is writable
```

`instanceof` fails across **realms** (iframes, worker threads, `vm` contexts), because each realm has its own `Array`, `Error`, etc. `Array.isArray` works across realms; `x instanceof Array` may not.

You can customise `instanceof`:

```js
class Even {
  static [Symbol.hasInstance](n) { return typeof n === 'number' && n % 2 === 0; }
}
4 instanceof Even;   // true
```

---

## 11.5 Common Mistakes and Best Practices

**Mistake: extending built-in prototypes.**

```js
Array.prototype.last = function () { return this.at(-1); };   // ❌
```
It breaks `for...in`, collides with future standard methods (this is exactly how `Array.prototype.flatten` got renamed to `flat`), and affects every library in the page. Write a standalone function instead.

**Mistake: forgetting `Dog.prototype.constructor = Dog`.** Anything relying on `instance.constructor` silently gets the parent.

**Mistake: shared mutable state on the prototype.**

```js
function User() {}
User.prototype.tags = [];        // ❌ ONE array shared by every instance
new User().tags.push('a');
new User().tags;                  // ['a']  — surprise
```
Put mutable state on the instance (`this.tags = []` in the constructor). Only methods and immutable constants belong on the prototype.

**Mistake: `__proto__` in modern code.** It's normative-optional legacy. Use `Object.getPrototypeOf` / `Object.setPrototypeOf`.

**Mistake: calling `Object.setPrototypeOf` on an existing object.** It deoptimizes the object severely in every engine. Set the prototype at creation time with `Object.create`.

**Prototype pollution** — the security consequence:

```js
JSON.parse('{"__proto__": {"isAdmin": true}}');   // safe by itself
merge({}, JSON.parse(payload));                    // ❌ a naive merge pollutes Object.prototype
({}).isAdmin;                                       // true — every object in the app
```
Guard merge functions against `__proto__`, `constructor`, and `prototype` keys. See Chapter 31.

---

## 11.6 Real-World Example: A Plugin Chain with `Object.create`

```js
/**
 * Layered defaults using the prototype chain instead of copying.
 * Lookups fall through automatically; overrides shadow.
 */
const baseTheme = {
  fontFamily: 'system-ui',
  fontSize: 16,
  color: '#222',
  spacing(n) { return `${n * 8}px`; },
};

const darkTheme = Object.create(baseTheme);
darkTheme.color = '#eee';
darkTheme.background = '#111';

const compactDark = Object.create(darkTheme);
compactDark.spacing = function (n) { return `${n * 4}px`; };

compactDark.fontFamily;   // 'system-ui'  ← from baseTheme, 2 hops
compactDark.color;         // '#eee'       ← from darkTheme, 1 hop
compactDark.spacing(2);    // '8px'        ← own override

// Change the base at runtime; every descendant sees it immediately.
baseTheme.fontSize = 18;
compactDark.fontSize;      // 18
```

That last line is the real advantage over copying: **live delegation**. Change once, and every object inheriting from it updates. Copying with spread would have frozen the values at merge time.

The trade-off: lookups cost a chain walk, and `Object.keys(compactDark)` returns only own keys, which surprises people serializing these objects.

---

## 11.7 Questions

**1.** MCQ — `[[Prototype]]` is read with: a) `.prototype` b) `Object.getPrototypeOf` c) `constructor` d) `instanceof`
<details><summary>Answer</summary>**b)**</details>

**2.** MCQ — `.prototype` exists on: a) all objects b) functions c) instances d) arrays only
<details><summary>Answer</summary>**b)**</details>

**3.** MCQ — Assigning to an inherited property: a) modifies the prototype b) creates an own property c) throws d) is ignored
<details><summary>Answer</summary>**b)** (Unless the prototype defines a setter.)</details>

**4.** MCQ — `'toString' in {}` is: a) true b) false
<details><summary>Answer</summary>**a)** `in` searches the chain.</details>

**5.** MCQ — `Object.hasOwn({}, 'toString')` is: a) true b) false
<details><summary>Answer</summary>**b)**</details>

**6.** MCQ — Which works across iframes? a) `instanceof Array` b) `Array.isArray` c) `constructor === Array` d) none
<details><summary>Answer</summary>**b)**</details>

**7.** Short — Describe property lookup in one sentence.
<details><summary>Answer</summary>The engine checks the object's own properties, then walks `[[Prototype]]` links upward until it finds the key or reaches `null`.</details>

**8.** Short — `.prototype` vs `[[Prototype]]`?
<details><summary>Answer</summary>`.prototype` is a property on constructor functions holding the object that instances will delegate to. `[[Prototype]]` is the actual internal delegation link on every object.</details>

**9.** Short — Why restore `constructor` after reassigning a prototype?
<details><summary>Answer</summary>`Object.create(Parent.prototype)` produces an object whose inherited `constructor` is `Parent`. Code (and some libraries) reading `instance.constructor` would get the wrong function.</details>

**10.** Short — Why is extending built-in prototypes discouraged?
<details><summary>Answer</summary>Global effect, `for...in` pollution, and collision risk with future standard methods.</details>

**11.** Short — What is prototype pollution?
<details><summary>Answer</summary>An attacker sets a key like `__proto__.isAdmin` through a naive deep-merge or path-set, adding a property to `Object.prototype` that every object then inherits.</details>

**12.** Output — What prints?
```js
const a = { x: 1 };
const b = Object.create(a);
b.x = 2;
delete b.x;
console.log(b.x, Object.hasOwn(b, 'x'));
```
<details><summary>Answer</summary>`1 false`</details>

**13.** Output — What prints?
```js
function F() {}
F.prototype.tags = [];
const p = new F(), q = new F();
p.tags.push('a');
console.log(q.tags);
```
<details><summary>Answer</summary>`['a']` — the array is shared on the prototype.</details>

**14.** Output — What prints?
```js
function F() {}
const f = new F();
console.log(F.prototype.constructor === F, Object.getPrototypeOf(f) === F.prototype, f.prototype);
```
<details><summary>Answer</summary>`true true undefined`</details>

**15.** Output — What prints?
```js
const o = Object.create(null);
console.log(typeof o.toString, Object.getPrototypeOf(o));
```
<details><summary>Answer</summary>`'undefined' null`</details>

**16.** Output — What prints?
```js
class A {}
class B extends A {}
const b = new B();
console.log(b instanceof A, Object.getPrototypeOf(B) === A);
```
<details><summary>Answer</summary>`true true` — classes also inherit statics via the constructor's own prototype link.</details>

**17.** Debug — After adding a helper, `for...in` over arrays picks up an extra key.
<details><summary>Answer</summary>Someone assigned to `Array.prototype`, creating an enumerable inherited property. Use a standalone function, or if you must patch, use `Object.defineProperty` with `enumerable: false`.</details>

**18.** Debug — Every user object suddenly has `isAdmin: true`.
<details><summary>Answer</summary>Prototype pollution through a merge/path-set with untrusted input. Add key guards and consider `Object.freeze(Object.prototype)` in defence-in-depth.</details>

**19.** Debug — `d.constructor.name` returns `'Animal'` for a `Dog`.
<details><summary>Answer</summary>`Dog.prototype.constructor` was never reset after `Dog.prototype = Object.create(Animal.prototype)`.</details>

**20.** Code — Implement `myInstanceof(obj, Ctor)`.
<details><summary>Answer</summary>

```js
function myInstanceof(obj, Ctor) {
  if (obj === null || (typeof obj !== 'object' && typeof obj !== 'function')) return false;
  let proto = Object.getPrototypeOf(obj);
  while (proto !== null) {
    if (proto === Ctor.prototype) return true;
    proto = Object.getPrototypeOf(proto);
  }
  return false;
}
```
</details>

**21.** Code — Write `getChain(obj)` returning an array of constructor names up the chain.
<details><summary>Answer</summary>

```js
function getChain(obj) {
  const out = [];
  for (let p = Object.getPrototypeOf(obj); p; p = Object.getPrototypeOf(p)) {
    out.push(p.constructor?.name ?? 'null-prototype');
  }
  return out;
}
getChain([]);   // ['Array', 'Object']
```
</details>

**22.** Code — Implement `myNew(Ctor, ...args)` matching `new` semantics including object returns.
<details><summary>Answer</summary>See §11.2 — the four-step version. Add a guard: `if (typeof Ctor !== 'function') throw new TypeError(...)`.</details>

**23.** Code — Write a safe `deepMerge` immune to prototype pollution.
<details><summary>Answer</summary>

```js
const BLOCKED = new Set(['__proto__', 'constructor', 'prototype']);
function safeMerge(target, source) {
  for (const [k, v] of Object.entries(source)) {
    if (BLOCKED.has(k)) continue;
    if (v && typeof v === 'object' && !Array.isArray(v)) {
      target[k] = safeMerge(target[k] ?? Object.create(null), v);
    } else target[k] = v;
  }
  return target;
}
```
`Object.entries` already skips `__proto__` as a getter in most cases, but the explicit guard covers `JSON.parse` results where it's an own property.
</details>

**24.** Scenario — A perf profile shows a hot function slowed 10× after a refactor that added `Object.setPrototypeOf`. Explain.
<details><summary>Answer</summary>Changing an object's prototype after creation invalidates every inline cache and hidden class the engine built for it, forcing megamorphic dictionary-mode lookups. Set the prototype at creation with `Object.create` or a constructor instead.</details>

**25.** Scenario — Would you use prototypal delegation for a theming system today, or plain object merging?
<details><summary>Answer</summary>Merging, usually — it's easier to serialize, diff, and reason about, and `Object.keys` behaves as people expect. Delegation wins when you specifically want live inheritance from a mutable base and have many descendants.</details>

---

## Chapter Summary

- Every object has `[[Prototype]]`; reads walk the chain, writes create own properties (shadowing).
- `.prototype` lives on functions and becomes instances' `[[Prototype]]` under `new`.
- Manual inheritance = `Parent.call(this)` + `Object.create(Parent.prototype)` + reset `constructor` + override.
- `instanceof` walks the chain but breaks across realms; `Array.isArray` doesn't.
- Never put mutable state on a prototype, never extend built-ins, never `setPrototypeOf` a live object.
- Guard merges against `__proto__` to prevent prototype pollution.

---

## Practice Exercises

1. Draw the full prototype chain for `[]`, `function(){}`, `new Date()`, and `Object.create(null)`.
2. Build a three-level hierarchy with constructor functions, then rewrite it with `class` and compare.
3. Implement `Object.create` yourself using a constructor function.
4. Write a mixin function that copies methods (with descriptors) onto a target prototype.
5. Demonstrate prototype pollution against a naive merge, then patch it and prove the fix with a test.

---

**Next:** [Chapter 12 — Classes and OOP →](12-classes-oop.md)
