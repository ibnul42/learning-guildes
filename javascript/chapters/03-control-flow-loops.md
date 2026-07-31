# Chapter 3 — Control Flow and Loops

## Learning Objectives

- Choose the right branching construct for a given problem
- Use every loop form correctly and know which iterates keys vs values
- Understand `switch` fall-through and its strict-equality matching
- Avoid the classic `for...in` on arrays mistake
- Replace imperative loops with array methods when it improves clarity

---

## 3.1 Branching

```js
// if / else if / else
if (score >= 90) grade = 'A';
else if (score >= 80) grade = 'B';
else grade = 'F';

// Ternary — for expressions, not statements
const grade = score >= 90 ? 'A' : score >= 80 ? 'B' : 'F';

// Short-circuit as a guard
user && render(user);          // works, but an `if` reads better
user?.save();                  // clearer intent
```

**Nested ternaries** are readable up to two levels if formatted as above. Beyond that, use a lookup table or early returns.

### `switch`

```js
switch (action.type) {
  case 'ADD':
  case 'INSERT':            // fall-through: both share a body
    return add(action.payload);
  case 'REMOVE':
    return remove(action.payload);
  default:
    return state;
}
```

Two things to know:

1. **`switch` uses `===`.** `switch ('1') { case 1: }` never matches.
2. **Cases fall through without `break`.** Deliberate fall-through is fine; accidental fall-through is a top-ten bug source. Linters enforce a `// falls through` comment.

Since each `case` shares one block scope, declaring with `let` in two cases collides:

```js
switch (x) {
  case 1: let y = 1; break;
  case 2: let y = 2; break;  // ❌ SyntaxError: y already declared
}
// Fix: wrap each case body in braces { }
```

**Object lookup often beats `switch`:**

```js
const handlers = {
  ADD: add,
  REMOVE: remove,
};
return (handlers[action.type] ?? (() => state))(action.payload);
```

---

## 3.2 Loops

### `for`

```js
for (let i = 0; i < arr.length; i++) { }
```
Full control over index, step, and direction. Still the fastest and the right choice when you need the index or non-unit steps.

### `while` and `do...while`

```js
while (queue.length) process(queue.shift());

do { attempt++; } while (attempt < 3 && !success);  // body runs at least once
```

### `for...of` — values of any iterable

```js
for (const item of ['a', 'b']) console.log(item);      // 'a', 'b'
for (const ch of 'héllo') console.log(ch);              // handles surrogate pairs
for (const [k, v] of Object.entries(obj)) console.log(k, v);
for (const [k, v] of map) console.log(k, v);
```

Works on arrays, strings, `Map`, `Set`, `NodeList`, `arguments`, generators — anything implementing `Symbol.iterator` (Chapter 14). **Does not work on plain objects.**

With index, use `entries()`:

```js
for (const [i, item] of arr.entries()) console.log(i, item);
```

### `for...in` — enumerable string keys, including inherited

```js
const arr = ['a', 'b'];
for (const k in arr) console.log(k, typeof k);  // '0' string, '1' string
```

> ⚠️ **Gotcha** — `for...in` gives you *string* keys, walks the *prototype chain*, and has **no guaranteed order for non-integer keys**. Never use it on arrays.

```js
Array.prototype.custom = () => {};
for (const k in ['a']) console.log(k);  // '0', 'custom'  ← polluted
for (const v of ['a'])  console.log(v);  // 'a'            ← safe
```

If you must use `for...in`, guard it:

```js
for (const key in obj) {
  if (!Object.hasOwn(obj, key)) continue;   // ES2022; replaces hasOwnProperty
  // ...
}
```

Better: `Object.keys(obj)`, `Object.values(obj)`, or `Object.entries(obj)`.

### Quick reference

| Loop | Iterates | Works on | Inherited props? |
|------|----------|----------|------------------|
| `for` | index | anything indexable | n/a |
| `for...of` | values | iterables | no |
| `for...in` | string keys | objects | **yes** |
| `.forEach()` | values | arrays | no |

---

## 3.3 `break`, `continue`, and Labels

```js
for (const n of nums) {
  if (n < 0) continue;   // skip to next iteration
  if (n > 100) break;    // exit the loop
}
```

Labels escape nested loops:

```js
outer:
for (const row of grid) {
  for (const cell of row) {
    if (cell === target) break outer;   // exits BOTH loops
  }
}
```

Labels are rare but legitimate. The common alternative is extracting the nested loop into a function and using `return`, which is usually clearer.

> ⚠️ **You cannot `break` out of `forEach`.** `return` inside `forEach` only skips that one callback invocation. If you need early exit, use `for...of`, `.some()`, or `.find()`.

```js
[1,2,3].forEach(n => { if (n === 2) return; console.log(n); }); // 1, 3
[1,2,3].some(n => { console.log(n); return n === 2; });         // 1, 2 — stops
```

---

## 3.4 Loops vs Array Methods

```js
// Imperative
const result = [];
for (const u of users) {
  if (u.active) result.push(u.name.toUpperCase());
}

// Declarative
const result = users
  .filter(u => u.active)
  .map(u => u.name.toUpperCase());
```

The second says *what*, not *how*. Prefer it — until you have a measured performance problem, or the logic needs `break`, or you'd chain more than three or four operations over a large array (which allocates an intermediate array per step).

---

## 3.5 Common Mistakes and Best Practices

**Mistake: modifying an array while iterating it.**
```js
// ❌ skips elements — indices shift under you
for (let i = 0; i < arr.length; i++) if (arr[i].bad) arr.splice(i, 1);

// ✅
const clean = arr.filter(x => !x.bad);
// or iterate backwards if you must mutate in place
for (let i = arr.length - 1; i >= 0; i--) if (arr[i].bad) arr.splice(i, 1);
```

**Mistake: off-by-one.** `<=` with `arr.length` reads one past the end, yielding `undefined`.

**Mistake: infinite loop from a forgotten increment.** In a `while`, put the update on the last line of the body and check it exists before running.

**Best practice: prefer early returns to deep nesting.**
```js
// ❌
function f(u) { if (u) { if (u.active) { if (u.email) { return send(u); } } } }
// ✅
function f(u) {
  if (!u?.active || !u.email) return;
  return send(u);
}
```

**Best practice: hoist invariants out of loop conditions** when the condition is expensive — though `arr.length` specifically is cheap and engines handle it fine, so don't cache it out of superstition.

---

## 3.6 Real-World Example: A Retry Loop with Backoff

```js
/**
 * Retries an async operation with exponential backoff.
 * Demonstrates: do...while, break, labelled early exit, guard clauses.
 */
async function withRetry(fn, { attempts = 3, baseDelayMs = 200 } = {}) {
  let lastError;

  for (let attempt = 1; attempt <= attempts; attempt++) {
    try {
      return await fn();                    // success → exit immediately
    } catch (err) {
      lastError = err;

      // Don't retry client errors — they'll fail identically every time
      if (err.status >= 400 && err.status < 500) break;

      // Don't sleep after the final attempt
      if (attempt === attempts) break;

      const delay = baseDelayMs * 2 ** (attempt - 1);   // 200, 400, 800...
      await new Promise(r => setTimeout(r, delay));
    }
  }

  throw new Error(`Failed after ${attempts} attempts`, { cause: lastError });
}
```

Notice the two distinct `break` conditions. Retrying a `404` forever is a real production incident, not a hypothetical.

---

## 3.7 Questions

**1.** MCQ — `for...in` over an array yields: a) values b) numeric indices c) string indices d) entries
<details><summary>Answer</summary>**c)** String keys.</details>

**2.** MCQ — `switch` compares using: a) `==` b) `===` c) `Object.is` d) `ToBoolean`
<details><summary>Answer</summary>**b)**</details>

**3.** MCQ — Which allows early exit? a) `forEach` b) `map` c) `for...of` d) `filter`
<details><summary>Answer</summary>**c)**</details>

**4.** MCQ — `do...while` differs from `while` because: a) it's faster b) the body always runs at least once c) it can't use `break` d) it needs a label
<details><summary>Answer</summary>**b)**</details>

**5.** MCQ — Which is NOT iterable with `for...of`? a) `'abc'` b) `new Set()` c) `{a:1}` d) `[1,2]`
<details><summary>Answer</summary>**c)** Plain objects have no `Symbol.iterator`.</details>

**6.** MCQ — `break outer;` requires: a) a function b) a labelled statement c) strict mode d) a `switch`
<details><summary>Answer</summary>**b)**</details>

**7.** Short — Why should you never use `for...in` on an array?
<details><summary>Answer</summary>It returns string keys rather than numbers, includes enumerable inherited properties, and doesn't guarantee order for non-integer keys.</details>

**8.** Short — How do you get both index and value in a `for...of`?
<details><summary>Answer</summary>`for (const [i, v] of arr.entries())`.</details>

**9.** Short — What does `return` do inside a `forEach` callback?
<details><summary>Answer</summary>Ends that single callback invocation only. The loop continues. It behaves like `continue`, not `break`.</details>

**10.** Short — Give a case where deliberate `switch` fall-through is good design.
<details><summary>Answer</summary>Grouping several cases that share one body, e.g. `case 'ADD': case 'INSERT': return add(...)`.</details>

**11.** Short — Why does `let` in two `switch` cases cause a SyntaxError?
<details><summary>Answer</summary>All cases share a single block scope, so the second declaration is a redeclaration. Wrap each case body in `{ }`.</details>

**12.** Output — What prints?
```js
for (var i = 0; i < 3; i++) {}
console.log(i);
```
<details><summary>Answer</summary>`3` — `var` leaks out of the block. With `let` it's a `ReferenceError`.</details>

**13.** Output — What prints?
```js
let s = 0;
for (let i = 0; i < 5; i++) { if (i === 2) continue; if (i === 4) break; s += i; }
console.log(s);
```
<details><summary>Answer</summary>`4` — adds 0, 1, skips 2, adds 3, breaks at 4.</details>

**14.** Output — What prints?
```js
switch (1) {
  case 1: console.log('one');
  case 2: console.log('two'); break;
  case 3: console.log('three');
}
```
<details><summary>Answer</summary>`one` then `two` — no `break` after case 1.</details>

**15.** Output — What prints?
```js
const arr = [1,2,3];
for (const i in arr) console.log(i + 1);
```
<details><summary>Answer</summary>`'01'`, `'11'`, `'21'` — keys are strings, so `+` concatenates.</details>

**16.** Output — What prints?
```js
let i = 0;
do { console.log(i); i++; } while (i < 0);
```
<details><summary>Answer</summary>`0` — the body runs once before the condition is tested.</details>

**17.** Output — What prints?
```js
const a = [1,2,3,4];
for (let i = 0; i < a.length; i++) if (a[i] % 2 === 0) a.splice(i, 1);
console.log(a);
```
<details><summary>Answer</summary>`[1, 3]` here by luck, but the pattern is buggy: after removing index 1, everything shifts left and index 2 is skipped. With `[1,2,4,3]` you'd get `[1,4,3]`. Use `filter`.</details>

**18.** Debug — Why does this never terminate?
```js
let i = 10;
while (i > 0) { console.log(i); }
```
<details><summary>Answer</summary>`i` is never decremented. Add `i--`.</details>

**19.** Debug — This "search" always returns `undefined`:
```js
function find(arr, target) {
  arr.forEach(item => { if (item.id === target) return item; });
}
```
<details><summary>Answer</summary>`return` inside `forEach` returns from the *callback*, and `forEach` returns `undefined`. Use `arr.find(item => item.id === target)`.</details>

**20.** Debug — A reducer silently returns the wrong branch:
```js
switch (action.type) {
  case 'increment': return n + 1;
  case 'decrement': return n - 1;
}
```
<details><summary>Answer</summary>No `default`. An unknown action returns `undefined`, wiping state. Always add `default: return state;`.</details>

**21.** Code — FizzBuzz, 1–100, but write it with no `if` statements.
<details><summary>Answer</summary>

```js
for (let i = 1; i <= 100; i++) {
  const s = (i % 3 ? '' : 'Fizz') + (i % 5 ? '' : 'Buzz');
  console.log(s || i);
}
```
</details>

**22.** Code — Write `chunk(arr, size)` splitting an array into groups.
<details><summary>Answer</summary>

```js
function chunk(arr, size) {
  if (size < 1) throw new RangeError('size must be >= 1');
  const out = [];
  for (let i = 0; i < arr.length; i += size) out.push(arr.slice(i, i + size));
  return out;
}
chunk([1,2,3,4,5], 2); // [[1,2],[3,4],[5]]
```
</details>

**23.** Code — Given a 2D grid, find the coordinates of the first cell equal to `target`, or `null`.
<details><summary>Answer</summary>

```js
function findCell(grid, target) {
  for (let r = 0; r < grid.length; r++)
    for (let c = 0; c < grid[r].length; c++)
      if (grid[r][c] === target) return [r, c];
  return null;
}
```
`return` beats a labelled `break` here — no flag variable needed.
</details>

**24.** Code — Rewrite this with array methods:
```js
let total = 0;
for (let i = 0; i < orders.length; i++)
  if (orders[i].status === 'paid') total += orders[i].amount;
```
<details><summary>Answer</summary>

```js
const total = orders
  .filter(o => o.status === 'paid')
  .reduce((sum, o) => sum + o.amount, 0);

// Single pass:
const total = orders.reduce((sum, o) => o.status === 'paid' ? sum + o.amount : sum, 0);
```
</details>

**25.** Scenario — You must process 5 million records and the `filter().map().reduce()` chain is causing memory pressure. What do you change?
<details><summary>Answer</summary>Each chained method allocates a full intermediate array. Collapse to a single `for...of` pass (or a single `reduce`) so only one traversal and no intermediates occur. For truly huge data, use a generator pipeline (Chapter 14) so items stream one at a time.</details>

**26.** Scenario — A code reviewer says your labelled `break` is "spaghetti." Do you agree?
<details><summary>Answer</summary>Partly. Labels are legal and clear for exiting nested loops, but extracting the inner loop into a named function and using `return` is usually more readable and testable. If the extraction is awkward, keep the label and add a comment.</details>

---

## Chapter Summary

- `switch` uses `===` and falls through without `break`; brace each case body if you declare with `let`.
- `for...of` iterates **values** of iterables; `for...in` iterates **inherited string keys** — never use it on arrays.
- `forEach` cannot `break`; use `for...of`, `some`, or `find`.
- Labels exit nested loops, but extracting a function and `return`ing is usually cleaner.
- Never mutate an array while forward-iterating it.
- Prefer array methods for clarity; drop to a single loop for very large data.

---

## Practice Exercises

1. Write a function printing a right-aligned pyramid of `*` of height `n` using nested loops.
2. Implement `range(start, stop, step)` returning an array, handling negative steps.
3. Convert three imperative loops in your own code into array-method chains, then convert one back and explain when each is better.
4. Write `flattenDeep(arr)` using an explicit stack and a `while` loop (no recursion).
5. Build a `switch`-based state machine for a traffic light, then rewrite it as an object lookup. Compare.

---

## Mini Project: A Text-Adventure Game Loop

Build a small console game in `adventure.js`.

**Requirements:**

- A `rooms` object where each room has a `description` and `exits: { north: 'hall', ... }`.
- A main game loop (`while`) reading commands until the player types `quit`.
- A `switch` handling verbs: `go`, `look`, `take`, `inventory`, `help`, `quit` — with a `default` that reports unknown commands.
- Use `continue` to skip invalid input without ending the turn.
- An inventory `Set`, and at least one puzzle requiring an item to pass through an exit.
- A win condition that `break`s the loop with a message.

**Stretch:** add a move counter, a `history` array, and an `undo` command.

---

**Next:** [Chapter 4 — Functions →](04-functions.md)
