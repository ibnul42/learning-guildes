# Chapter 2 — Variables, Types, and Operators

## Learning Objectives

- Choose correctly between `const`, `let`, and (never) `var`
- Name all primitive types and explain how they differ from objects
- Predict the result of any type coercion
- Explain `==` vs `===` precisely enough to satisfy an interviewer
- Use `typeof`, `instanceof`, and `Object.prototype.toString` appropriately

---

## 2.1 Declaring Variables

Three keywords, one recommendation.

```js
var a = 1;    // function-scoped, hoisted-and-initialized, redeclarable — legacy
let b = 2;    // block-scoped, TDZ, reassignable
const c = 3;  // block-scoped, TDZ, NOT reassignable
```

| | `var` | `let` | `const` |
|---|---|---|---|
| Scope | function | block | block |
| Hoisted | yes, as `undefined` | yes, into TDZ | yes, into TDZ |
| Redeclare in same scope | yes | no | no |
| Reassign | yes | yes | no |
| Creates property on `globalThis` | yes (at top level) | no | no |

**Rule: `const` by default, `let` when you must reassign, `var` never.** Not dogma — `const` signals intent, and a reader scanning your code knows the binding won't move under them.

> ⚠️ **Gotcha** — `const` freezes the *binding*, not the value.

```js
const user = { name: 'Ada' };
user.name = 'Grace';  // ✅ fine — the object is mutable
user = {};            // ❌ TypeError — the binding cannot be reassigned
```

To freeze the value, use `Object.freeze` (Chapter 5).

The classic `var`-in-a-loop bug:

```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i));
// 3, 3, 3  — one shared `i`, read after the loop finishes

for (let i = 0; i < 3; i++) setTimeout(() => console.log(i));
// 0, 1, 2  — `let` creates a fresh binding per iteration
```

That per-iteration binding is a specific, specified behaviour of `let` in `for` loops. It's asked about constantly.

---

## 2.2 The Type System

JavaScript has **seven primitives** and **one object type**.

```js
typeof undefined       // 'undefined'
typeof true            // 'boolean'
typeof 42              // 'number'
typeof 9007199254740993n // 'bigint'
typeof 'hi'            // 'string'
typeof Symbol()        // 'symbol'
typeof null            // 'object'   ← the famous bug
typeof {}              // 'object'
typeof []              // 'object'   ← arrays are objects
typeof function () {}  // 'function' ← special-cased, still an object
```

**Primitives are immutable and compared by value. Objects are mutable and compared by reference.**

```js
let s = 'abc';
s[0] = 'X';
console.log(s); // 'abc' — strings cannot be mutated

console.log({ a: 1 } === { a: 1 }); // false — two different objects
const x = { a: 1 };
console.log(x === x);               // true
```

**Autoboxing** explains why primitives seem to have methods:

```js
'hello'.toUpperCase();
// The engine temporarily wraps the primitive in a String object,
// calls the method, returns the result, discards the wrapper.
```

That's also why `s[0] = 'X'` fails silently — you mutated a throwaway wrapper.

### `null` vs `undefined`

- `undefined` — "no value has been assigned yet." The engine produces it.
- `null` — "intentionally empty." A programmer produces it.

```js
let x;                    // undefined
function f(a) { return a; }
f();                      // undefined — missing argument
({}).nope;                // undefined — missing property
const cleared = null;     // deliberate absence
```

---

## 2.3 Coercion

Every interviewer's favourite topic. There are two kinds.

**Explicit** (you asked for it):

```js
Number('42')    // 42
String(42)      // '42'
Boolean(0)      // false
parseInt('42px', 10) // 42
```

**Implicit** (the operator did it for you):

```js
'5' + 3    // '53'   → + prefers string concatenation if either side is a string
'5' - 3    // 2      → - has no string meaning, so both sides become numbers
'5' * '2'  // 10
true + 1   // 2      → true → 1
[] + {}    // '[object Object]'
```

### The falsy list — memorise it

Exactly **eight** values are falsy:

```js
false, 0, -0, 0n, '', null, undefined, NaN
```

Everything else is truthy — including `'0'`, `'false'`, `[]`, `{}`, and `function(){}`.

```js
if ([]) console.log('runs');        // empty array is truthy
console.log([] == false);           // true  ← but this is also true!
```

Both are correct and they don't contradict: `if ([])` uses `ToBoolean` (objects are always `true`), while `[] == false` triggers `ToPrimitive` on the array (giving `''`), then numeric comparison (`0 == 0`).

### `==` vs `===`

`===` compares type and value, with no coercion. `==` coerces first.

The `==` algorithm, simplified:

1. Same type → behave like `===`
2. `null == undefined` → `true` (and neither equals anything else)
3. Number vs string → convert string to number
4. Boolean on either side → convert boolean to number
5. Object vs primitive → `ToPrimitive` the object, then retry

```js
0 == '';          // true   ('' → 0)
0 == '0';         // true   ('0' → 0)
'' == '0';        // false  (both strings, different)
null == undefined // true
null == 0;        // false  (rule 2 is exhaustive)
NaN == NaN;       // false  (NaN equals nothing)
[] == ![];        // true   (![] → false → 0; [] → '' → 0)
```

> 💼 **Interview** — Use `===` always. The one accepted exception: `x == null` as a concise check for "null or undefined." Say that out loud and you sound experienced rather than dogmatic.

### `Object.is`

```js
Object.is(NaN, NaN);  // true  (=== gives false)
Object.is(0, -0);     // false (=== gives true)
Object.is(1, 1);      // true
```

Use it when `NaN` or signed zero actually matter — rarely, but `Array.prototype.includes` uses this semantic, which is why `[NaN].includes(NaN)` is `true` while `[NaN].indexOf(NaN)` is `-1`.

---

## 2.4 Operators Worth Knowing

```js
// Arithmetic
7 / 2        // 3.5  — no integer division
7 % 2        // 1
-7 % 2       // -1   — sign follows the dividend
2 ** 10      // 1024

// Increment
let i = 5;
console.log(i++); // 5, then i is 6  (post: return old)
console.log(++i); // 7               (pre: return new)

// Logical — these RETURN OPERANDS, not booleans
'a' || 'b'   // 'a'
'' || 'b'    // 'b'
'a' && 'b'   // 'b'
0 && 'b'     // 0

// Nullish coalescing — only null/undefined trigger the fallback
0 ?? 10      // 0    ← differs from || which gives 10
null ?? 10   // 10

// Logical assignment (ES2021)
let cfg = { retries: 0 };
cfg.retries ||= 3;   // 3  — wrong here! 0 is falsy
cfg.timeout ??= 5;   // 5  — correct: only fills null/undefined

// Comma — evaluates left to right, returns the last
const z = (1, 2, 3);  // 3

// Optional chaining
user?.profile?.email  // undefined instead of throwing
fn?.()                // call only if fn exists
arr?.[0]
```

**Precedence trap:**

```js
console.log(1 + 2 + '3');   // '33'  — left to right: 3, then '33'
console.log('1' + 2 + 3);   // '123'
```

`??` cannot be mixed with `||` or `&&` without parentheses — that's a `SyntaxError`, deliberately, to prevent ambiguity:

```js
// a || b ?? c        ❌ SyntaxError
(a || b) ?? c      // ✅
```

---

## 2.5 Type Checking Correctly

```js
// Primitives: typeof
typeof value === 'string'

// null: strict equality
value === null

// Arrays: never typeof
Array.isArray(value)

// NaN
Number.isNaN(value)        // ✅ no coercion
isNaN('abc')               // ⚠️ true — coerces first, avoid

// Integers
Number.isInteger(value)

// Plain object (not array, not null, not Date)
const isPlain = (v) =>
  Object.prototype.toString.call(v) === '[object Object]';

// Class instances
value instanceof Date      // ⚠️ fails across iframes/realms
```

The `Object.prototype.toString` trick is the most reliable general tag:

```js
const typeOf = (v) =>
  Object.prototype.toString.call(v).slice(8, -1).toLowerCase();

typeOf([]);        // 'array'
typeOf(null);      // 'null'
typeOf(new Date);  // 'date'
typeOf(/x/);       // 'regexp'
typeOf(new Map);   // 'map'
```

---

## 2.6 Common Mistakes and Best Practices

**Mistake: `||` for defaults with valid falsy values.**
```js
function connect(port) {
  port = port || 8080;   // ❌ port=0 becomes 8080
  port = port ?? 8080;   // ✅
}
```

**Mistake: `parseInt` without a radix.** Always pass `10`. Legacy engines interpreted leading `0` as octal.

**Mistake: comparing objects with `===` expecting deep equality.** Write or import a `deepEqual`.

**Mistake: `NaN` propagation.** One bad input poisons a whole calculation, silently, because `NaN` is falsy-ish but not falsy. Validate at boundaries.

**Best practice: convert at the edges.** Parse strings into numbers the moment data enters your system (form input, API response), not at the point of use. Then the interior of your program only handles real types.

**Best practice: one declaration per line.** `let a, b, c;` obscures which are initialized.

---

## 2.7 Real-World Example: Normalizing API Input

```js
/**
 * API payloads arrive with inconsistent types from various clients.
 * Normalize once, at the boundary, so downstream code can trust its inputs.
 */
function normalizeOrder(raw) {
  return {
    // Coerce to number, but reject garbage rather than passing NaN along
    quantity: toPositiveInt(raw.quantity, 1),

    // ?? not || — a 0 discount is meaningful
    discount: typeof raw.discount === 'number' ? raw.discount : 0,

    // Trim and lowercase; treat empty string as absent
    email: (raw.email ?? '').trim().toLowerCase() || null,

    // Explicit boolean — 'false' the string is truthy!
    giftWrap: raw.giftWrap === true || raw.giftWrap === 'true',

    notes: raw.notes?.trim() ?? '',
  };
}

function toPositiveInt(value, fallback) {
  const n = Number(value);
  return Number.isInteger(n) && n > 0 ? n : fallback;
}

console.log(normalizeOrder({
  quantity: '3', discount: 0, email: '  ADA@EX.COM ',
  giftWrap: 'false', notes: undefined,
}));
// { quantity: 3, discount: 0, email: 'ada@ex.com', giftWrap: false, notes: '' }
```

Note `giftWrap: 'false'` → `false`. Naive code writes `Boolean(raw.giftWrap)`, and every string including `'false'` becomes `true`. This is a real bug that reaches production regularly.

---

## 2.8 Questions

**1.** MCQ — How many falsy values are there? a) 5 b) 6 c) 7 d) 8
<details><summary>Answer</summary>**d)** `false, 0, -0, 0n, '', null, undefined, NaN`</details>

**2.** MCQ — `typeof null` returns: a) `'null'` b) `'object'` c) `'undefined'` d) throws
<details><summary>Answer</summary>**b)**</details>

**3.** MCQ — `const arr = [1,2]; arr.push(3);` — what happens? a) TypeError b) Works fine c) SyntaxError d) Silently ignored
<details><summary>Answer</summary>**b)** `const` protects the binding, not the contents.</details>

**4.** MCQ — `0 ?? 'default'` evaluates to: a) `0` b) `'default'` c) `undefined` d) `false`
<details><summary>Answer</summary>**a)** Only `null`/`undefined` trigger `??`.</details>

**5.** MCQ — Which correctly detects an array? a) `typeof x === 'array'` b) `x instanceof Object` c) `Array.isArray(x)` d) `x.length !== undefined`
<details><summary>Answer</summary>**c)**</details>

**6.** MCQ — `'5' - 3` is: a) `'53'` b) `2` c) `NaN` d) `'2'`
<details><summary>Answer</summary>**b)** `-` forces numeric conversion.</details>

**7.** MCQ — `Object.is(0, -0)` returns: a) `true` b) `false`
<details><summary>Answer</summary>**b)** `Object.is` distinguishes signed zeros; `===` does not.</details>

**8.** MCQ — Which is a `SyntaxError`? a) `a ?? b` b) `a || b ?? c` c) `(a || b) ?? c` d) `a && b`
<details><summary>Answer</summary>**b)** Mixing `??` with `||`/`&&` requires parentheses.</details>

**9.** Short — Explain why `const` doesn't make objects immutable.
<details><summary>Answer</summary>`const` prevents rebinding the variable to a different value. The value itself — if it's an object — retains its own mutability. Use `Object.freeze()` for a shallow immutable object.</details>

**10.** Short — When is `==` acceptable?
<details><summary>Answer</summary>`x == null`, as a compact test for "null or undefined". Everywhere else, `===`.</details>

**11.** Short — Why is `[] == false` true but `if ([])` truthy?
<details><summary>Answer</summary>Different algorithms. `if` applies `ToBoolean`, and all objects are truthy. `==` with a boolean converts the boolean to a number (`0`), then converts the array via `ToPrimitive` to `''`, then to `0`. `0 == 0` is true.</details>

**12.** Short — Difference between `null` and `undefined`?
<details><summary>Answer</summary>`undefined` means a value was never assigned (produced by the engine). `null` means deliberate emptiness (produced by a developer).</details>

**13.** Short — Why does `let` fix the classic `setTimeout` loop bug?
<details><summary>Answer</summary>`let` in a `for` loop creates a new binding for each iteration, and each closure captures its own. `var` has a single function-scoped binding shared by every closure.</details>

**14.** Short — What is autoboxing?
<details><summary>Answer</summary>The engine temporarily wraps a primitive in its object equivalent so a method can be called, then discards the wrapper. It's why `'a'.toUpperCase()` works and why `'a'.foo = 1` silently does nothing.</details>

**15.** Output — `console.log(1 + 2 + '3' + 4 + 5);`
<details><summary>Answer</summary>`'3345'` — left to right: `3`, then `'33'`, then `'334'`, then `'3345'`.</details>

**16.** Output — What prints?
```js
console.log(typeof NaN, NaN === NaN, [NaN].includes(NaN), [NaN].indexOf(NaN));
```
<details><summary>Answer</summary>`'number' false true -1` — `includes` uses SameValueZero (NaN matches itself); `indexOf` uses strict equality.</details>

**17.** Output — What prints?
```js
let a = 1;
{ let a = 2; console.log(a); }
console.log(a);
```
<details><summary>Answer</summary>`2` then `1`.</details>

**18.** Output — What prints?
```js
console.log([] + []);
console.log([] + {});
console.log(typeof ([] + {}));
```
<details><summary>Answer</summary>`''`, `'[object Object]'`, `'string'`.</details>

**19.** Output — What prints?
```js
console.log(0.1 + 0.2 === 0.3);
console.log(0.1 + 0.2);
```
<details><summary>Answer</summary>`false`, then `0.30000000000000004`. IEEE-754 binary floats cannot represent 0.1 exactly. (Chapter 7.)</details>

**20.** Output — What prints?
```js
const o = { valueOf: () => 42 };
console.log(o + 1, `${o}`);
```
<details><summary>Answer</summary>`43` and `'42'`. `+` hints `'default'` and tries `valueOf` first; template literals hint `'string'` and try `toString` first, but the inherited `Object.prototype.toString` is skipped in favour of... actually no: for a string hint `toString` is tried first, and the inherited one returns `'[object Object]'`. **Careful:** the real output is `43` and `'[object Object]'`. Verify this one yourself — it's a good lesson in not trusting intuition.</details>

**21.** Debug — Discount codes with 0% off aren't applying. Find the bug.
```js
const discount = userDiscount || defaultDiscount;
```
<details><summary>Answer</summary>`0` is falsy, so a legitimate zero discount falls through to the default. Use `userDiscount ?? defaultDiscount`.</details>

**22.** Debug — This validator accepts empty submissions:
```js
function isValid(input) {
  if (input.length > 0) return true;
  return false;
}
isValid('   ');  // true
```
<details><summary>Answer</summary>Whitespace has length. Use `input.trim().length > 0`. Also guard the type: `typeof input === 'string' && input.trim().length > 0`, otherwise `isValid(null)` throws.</details>

**23.** Debug — Why does this always log `'admin'`?
```js
if (user.role = 'admin') console.log('admin');
```
<details><summary>Answer</summary>`=` is assignment, not comparison. The expression evaluates to `'admin'`, which is truthy — and it also overwrites the user's role. Use `===`. Linters catch this (`no-cond-assign`).</details>

**24.** Code — Write `isEmpty(value)` returning `true` for `null`, `undefined`, `''`, `'  '`, `[]`, and `{}`, but `false` for `0` and `false`.
<details><summary>Answer</summary>

```js
function isEmpty(value) {
  if (value == null) return true;
  if (typeof value === 'string') return value.trim().length === 0;
  if (Array.isArray(value)) return value.length === 0;
  if (value instanceof Map || value instanceof Set) return value.size === 0;
  if (typeof value === 'object') return Object.keys(value).length === 0;
  return false; // numbers, booleans, functions are never "empty"
}
```
</details>

**25.** Code — Implement `looseEqual(a, b)` that mimics `==` without using `==`.
<details><summary>Answer</summary>

```js
function looseEqual(a, b) {
  if (typeof a === typeof b) return a === b;
  if (a === null && b === undefined) return true;
  if (a === undefined && b === null) return true;
  if (typeof a === 'number' && typeof b === 'string') return a === Number(b);
  if (typeof a === 'string' && typeof b === 'number') return Number(a) === b;
  if (typeof a === 'boolean') return looseEqual(Number(a), b);
  if (typeof b === 'boolean') return looseEqual(a, Number(b));
  if (typeof a === 'object' && a !== null) return looseEqual(a.valueOf?.() ?? String(a), b);
  if (typeof b === 'object' && b !== null) return looseEqual(a, b.valueOf?.() ?? String(b));
  return false;
}
```
Simplified (ignores BigInt and Symbol edge cases) — writing it out is the point.
</details>

**26.** Scenario — A form sends `{ "active": "false" }`. Your code does `if (data.active)` and always treats users as active. How do you fix this systematically, not just here?
<details><summary>Answer</summary>Add a normalization layer at the API boundary that parses the payload into typed values before any business logic sees it — a schema validator (Zod, Valibot) or a hand-written `normalize()` per endpoint. Fixing this one `if` leaves the same bug waiting in every other field.</details>

**27.** Scenario — A teammate insists on `var` because "it's what they learned." Make the case in two sentences.
<details><summary>Answer</summary>`var` leaks out of blocks and is silently redeclarable, which turns typos and loop-closure mistakes into runtime bugs rather than errors. `let`/`const` scope to the block you actually wrote, so the code means what it looks like it means.</details>

---

## Chapter Summary

- **`const` > `let` > `var`.** `const` locks the binding, not the value.
- **Seven primitives** (`undefined`, `boolean`, `number`, `bigint`, `string`, `symbol`, plus `null` awkwardly) and objects. Primitives compare by value, objects by reference.
- **Eight falsy values.** Everything else, including `[]` and `{}`, is truthy.
- **`===` always**, except `x == null`.
- **`??` over `||`** whenever `0`, `''`, or `false` are legitimate values.
- Check types with `typeof`, `Array.isArray`, `Number.isNaN`, and `Object.prototype.toString.call`.
- **Normalize at boundaries** so interior code can trust its inputs.

---

## Practice Exercises

1. Without running it, predict the output of all 25 comparisons in a table of `0, '', '0', [], {}, null, undefined, NaN, false` compared pairwise with `==`. Then verify.
2. Write a `typeOf(value)` utility returning precise lowercase tags for 12 different value kinds.
3. Refactor a function that uses `||` for three defaults so it uses `??` where appropriate — and justify each choice.
4. Implement `deepEqual(a, b)` handling nested objects, arrays, `Date`, and `NaN`.
5. Write a `coerceBoolean(v)` that maps `'true'/'1'/'yes'/true` → `true` and `'false'/'0'/'no'/false/''` → `false`, throwing otherwise.

---

## Mini Project: A Type Inspector CLI

Build `inspect.js` exporting `inspect(value)` that returns a report:

```js
{
  value: '<safe string preview>',
  typeof: 'object',
  tag: 'array',            // via Object.prototype.toString
  isPrimitive: false,
  isFalsy: false,
  coercions: {
    toNumber: NaN,
    toString: '1,2,3',
    toBoolean: true,
  },
  notes: ['Arrays are objects; use Array.isArray to detect them.'],
}
```

**Requirements:**

- Must not throw for *any* input, including `Symbol()`, `BigInt`, circular objects, and `Object.create(null)`.
- `notes` should flag at least five known gotchas when relevant (e.g. `typeof null`, `NaN !== NaN`, empty array is truthy but `== false`).
- Add a `compare(a, b)` function reporting `===`, `==`, and `Object.is` results side by side with an explanation of any disagreement.

**Stretch:** print results with `console.table` and add a REPL mode.

---

**Next:** [Chapter 3 — Control Flow and Loops →](03-control-flow-loops.md)
