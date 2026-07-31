# Appendix B — 121 Coding Challenges

Every challenge has a worked solution with complexity analysis, the reasoning behind the approach,
and the wrong answer people usually give first. Attempt each one before opening the solution — the
value is in the attempt, and reading a solution feels like learning in a way that doesn't survive
contact with an interview.

**Difficulty:** `easy` — you should get this in five minutes. `medium` — twenty, with an edge case
you'll miss first time. `hard` — an hour, or a hint.

---

## Contents

| Section | Count |
|---|---|
| [1. Strings](#1-strings) | 12 |
| [2. Arrays](#2-arrays) | 15 |
| [3. Objects](#3-objects) | 10 |
| [4. Functions and Closures](#4-functions-and-closures) | 12 |
| [5. Async](#5-async) | 13 |
| [6. Recursion](#6-recursion) | 8 |
| [7. Algorithms and Data Structures](#7-algorithms-and-data-structures) | 15 |
| [8. Polyfills](#8-polyfills) | 20 |
| [9. DOM](#9-dom) | 6 |
| [10. Output Prediction](#10-output-prediction) | 10 |

---

## 1. Strings

### 1. Reverse a string  `easy`
Return the input reversed.
**Example:** `'hello'` → `'olleh'`
<details><summary>Solution</summary>

```js
const reverse = (s) => [...s].reverse().join('');
```
**Complexity:** O(n) time, O(n) space.
**Why this approach:** Spread iterates by code point, so surrogate pairs (emoji) survive.
**Common wrong answer:** `s.split('')` splits by UTF-16 code unit and corrupts any character
outside the BMP — `[...'👍a'].reverse()` works, `'👍a'.split('').reverse()` produces mojibake.
Grapheme clusters (`'é'` as `e` + combining accent) still break both; `Intl.Segmenter` fixes that.
</details>

### 2. Check for a palindrome  `easy`
Ignore case, spaces, and punctuation.
**Example:** `'A man, a plan, a canal: Panama'` → `true`
<details><summary>Solution</summary>

```js
function isPalindrome(s) {
  const clean = s.toLowerCase().replace(/[^a-z0-9]/g, '');
  for (let i = 0, j = clean.length - 1; i < j; i++, j--) {
    if (clean[i] !== clean[j]) return false;
  }
  return true;
}
```
**Complexity:** O(n) time, O(n) space for the cleaned copy.
**Why this approach:** Two pointers exit at the first mismatch, so a non-palindrome costs O(1) in
the common case rather than building a reversed string.
**Common wrong answer:** `clean === [...clean].reverse().join('')` — correct but allocates two extra
strings and always scans the whole input.
</details>

### 3. Count character frequency  `easy`
Return a map of character to count.
**Example:** `'hello'` → `{ h: 1, e: 1, l: 2, o: 1 }`
<details><summary>Solution</summary>

```js
const frequency = (s) =>
  [...s].reduce((acc, c) => (acc[c] = (acc[c] ?? 0) + 1, acc), Object.create(null));
```
**Complexity:** O(n) time and space.
**Why this approach:** `Object.create(null)` means a character sequence like `constructor` can't
collide with an inherited property.
**Common wrong answer:** `{}` as the accumulator — then `frequency('constructor')` returns a
function for `acc['constructor']` on the first read, and `??` doesn't save you because it isn't nullish.
</details>

### 4. Find the first non-repeating character  `easy`
Return its index, or `-1`.
**Example:** `'leetcode'` → `0`; `'loveleetcode'` → `2`
<details><summary>Solution</summary>

```js
function firstUnique(s) {
  const counts = new Map();
  for (const c of s) counts.set(c, (counts.get(c) ?? 0) + 1);
  for (let i = 0; i < s.length; i++) if (counts.get(s[i]) === 1) return i;
  return -1;
}
```
**Complexity:** O(n) time, O(k) space for k distinct characters.
**Why this approach:** Two passes, both linear. One pass can't work — you don't know a character is
unique until you've seen the whole string.
**Common wrong answer:** `s.indexOf(c) === s.lastIndexOf(c)` inside a loop, which is O(n²).
</details>

### 5. Check whether two strings are anagrams  `easy`
**Example:** `('listen', 'silent')` → `true`
<details><summary>Solution</summary>

```js
function isAnagram(a, b) {
  if (a.length !== b.length) return false;              // cheap rejection first
  const counts = new Map();
  for (const c of a) counts.set(c, (counts.get(c) ?? 0) + 1);
  for (const c of b) {
    const n = counts.get(c);
    if (!n) return false;
    counts.set(c, n - 1);
  }
  return true;
}
```
**Complexity:** O(n) time, O(k) space.
**Why this approach:** Counting beats sorting — O(n) versus O(n log n) — and the length check
rejects most non-anagrams immediately.
**Common wrong answer:** Comparing sorted strings. Correct, but slower, and `.split('')` breaks on
non-BMP characters.
</details>

### 6. Truncate a string with an ellipsis  `easy`
Never exceed the maximum length including the ellipsis, and don't cut mid-word.
**Example:** `('The quick brown fox', 12)` → `'The quick…'`
<details><summary>Solution</summary>

```js
function truncate(s, max, ellipsis = '…') {
  if (s.length <= max) return s;
  const limit = max - ellipsis.length;
  if (limit <= 0) return ellipsis.slice(0, max);
  const cut = s.slice(0, limit + 1);
  const lastSpace = cut.lastIndexOf(' ');
  return (lastSpace > 0 ? cut.slice(0, lastSpace) : cut.slice(0, limit)).trimEnd() + ellipsis;
}
```
**Complexity:** O(max) time.
**Why this approach:** Slicing one character past the limit lets you tell whether the boundary
itself was a space, so a string ending exactly on a word boundary isn't truncated early.
**Common wrong answer:** `s.slice(0, max) + '…'` — the result is longer than `max`, which breaks
every layout that relied on the limit.
</details>

### 7. Convert between case conventions  `medium`
Implement `toCamel`, `toKebab`, and `toSnake`, round-tripping correctly.
<details><summary>Solution</summary>

```js
const toCamel = (s) => s
  .replace(/[-_\s]+(.)?/g, (_, c) => (c ? c.toUpperCase() : ''))
  .replace(/^(.)/, (c) => c.toLowerCase());

const toKebab = (s) => s
  .replace(/([a-z0-9])([A-Z])/g, '$1-$2')          // fooBar → foo-Bar
  .replace(/([A-Z])([A-Z][a-z])/g, '$1-$2')        // HTTPServer → HTTP-Server
  .replace(/[\s_]+/g, '-')
  .toLowerCase();

const toSnake = (s) => toKebab(s).replace(/-/g, '_');
```
**Complexity:** O(n) time.
**Why this approach:** The second `replace` in `toKebab` is what handles consecutive capitals —
without it `HTTPServer` becomes `h-t-t-p-server`.
**Common wrong answer:** A single `replace(/[A-Z]/g, c => '-' + c.toLowerCase())`, which mangles
acronyms and prefixes a leading dash on already-capitalised input.
</details>

### 8. Compress a string with run-length encoding  `medium`
Only compress where it's shorter.
**Example:** `'aaabbc'` → `'a3b2c'`; `'abc'` → `'abc'`
<details><summary>Solution</summary>

```js
function compress(s) {
  let out = '';
  for (let i = 0; i < s.length; ) {
    let j = i;
    while (j < s.length && s[j] === s[i]) j++;
    const run = j - i;
    out += run > 1 ? s[i] + run : s[i];
    i = j;
  }
  return out.length < s.length ? out : s;
}
```
**Complexity:** O(n) time, O(n) space.
**Why this approach:** The final length check makes it lossless-or-original, which is what a real
encoder does — compression that expands the input is worse than none.
**Common wrong answer:** Always emitting the count, so `'abc'` becomes `'a1b1c1'`.
</details>

### 9. Find the longest substring without repeating characters  `medium`
**Example:** `'abcabcbb'` → `3` (`'abc'`)
<details><summary>Solution</summary>

```js
function longestUnique(s) {
  const lastSeen = new Map();
  let best = 0, start = 0;
  for (let i = 0; i < s.length; i++) {
    const prev = lastSeen.get(s[i]);
    // Only move `start` forward — a repeat from before the window must be ignored.
    if (prev !== undefined && prev >= start) start = prev + 1;
    lastSeen.set(s[i], i);
    best = Math.max(best, i - start + 1);
  }
  return best;
}
```
**Complexity:** O(n) time, O(k) space.
**Why this approach:** A sliding window with last-seen indices jumps `start` directly rather than
shrinking one character at a time.
**Common wrong answer:** Omitting the `prev >= start` check, so `'abba'` gives 3 — the second `a`'s
stale index drags `start` backwards.
</details>

### 10. Validate balanced brackets  `medium`
Handle `()`, `[]`, `{}`.
**Example:** `'{[()]}'` → `true`; `'{[(])}'` → `false`
<details><summary>Solution</summary>

```js
function isBalanced(s) {
  const pairs = { ')': '(', ']': '[', '}': '{' };
  const stack = [];
  for (const c of s) {
    if (c === '(' || c === '[' || c === '{') stack.push(c);
    else if (c in pairs && stack.pop() !== pairs[c]) return false;
  }
  return stack.length === 0;
}
```
**Complexity:** O(n) time, O(n) space.
**Why this approach:** A stack is the only structure that captures nesting order; counting can't.
**Common wrong answer:** Counting opens and closes — `')('` has equal counts and is unbalanced.
Also forgetting the final `stack.length === 0`, which passes `'((('`.
</details>

### 11. Word-wrap text to a column width  `medium`
Break on spaces, never mid-word unless a single word exceeds the width.
<details><summary>Solution</summary>

```js
function wordWrap(text, width) {
  const lines = [];
  for (const paragraph of text.split('\n')) {
    let line = '';
    for (const word of paragraph.split(/\s+/).filter(Boolean)) {
      if (!line) {
        line = word;
      } else if (line.length + 1 + word.length <= width) {
        line += ' ' + word;
      } else {
        lines.push(line);
        line = word;
      }
      while (line.length > width) {           // a single word longer than the width
        lines.push(line.slice(0, width));
        line = line.slice(width);
      }
    }
    lines.push(line);
  }
  return lines.join('\n');
}
```
**Complexity:** O(n) time.
**Why this approach:** Handling the over-long word inside the loop means it's split whether it
arrived at the start of a line or after a break.
**Common wrong answer:** Ignoring words longer than the width, so a long URL blows the layout.
</details>

### 12. Implement a template renderer  `hard`
Support `{{key}}` and `{{a.b.c}}`, HTML-escaped by default with `{{{raw}}}` for unescaped.
<details><summary>Solution</summary>

```js
const escapeHtml = (s) => String(s).replace(/[&<>"']/g, (c) =>
  ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]));

const get = (obj, path) => path.split('.').reduce((o, k) => o?.[k], obj);

function render(template, data) {
  // Triple-brace FIRST in the alternation, or {{ matches the inner braces of {{{ }}}.
  return template.replace(/\{\{\{(\s*[\w.]+\s*)\}\}\}|\{\{(\s*[\w.]+\s*)\}\}/g,
    (_, raw, escaped) => {
      const value = get(data, (raw ?? escaped).trim());
      if (value === undefined || value === null) return '';
      return raw !== undefined ? String(value) : escapeHtml(value);
    });
}

render('<p>{{user.name}} — {{{bio}}}</p>', { user: { name: '<b>Ada</b>' }, bio: '<i>hi</i>' });
// '<p>&lt;b&gt;Ada&lt;/b&gt; — <i>hi</i></p>'
```
**Complexity:** O(n) time in the template length.
**Why this approach:** Escaping by default and requiring explicit opt-out for raw HTML is the only
safe direction — the reverse ships XSS the first time someone forgets.
**Common wrong answer:** Not escaping at all, or ordering the alternation so `{{` wins and
`{{{raw}}}` renders a stray brace.
</details>

---

## 2. Arrays

### 13. Flatten a nested array to a given depth  `easy`
**Example:** `([1,[2,[3,[4]]]], 2)` → `[1,2,3,[4]]`
<details><summary>Solution</summary>

```js
function flatten(arr, depth = 1) {
  return depth < 1 ? arr.slice()
    : arr.reduce((acc, x) =>
        acc.concat(Array.isArray(x) ? flatten(x, depth - 1) : x), []);
}
```
**Complexity:** O(n) time in total elements, O(depth) stack.
**Why this approach:** `Array.isArray` is realm-safe where `instanceof Array` isn't.
**Common wrong answer:** Ignoring `depth` and always fully flattening, or using `concat(...arr)`
which blows the stack on a large array.
</details>

### 14. Chunk an array  `easy`
**Example:** `([1,2,3,4,5], 2)` → `[[1,2],[3,4],[5]]`
<details><summary>Solution</summary>

```js
function chunk(arr, size) {
  if (!Number.isInteger(size) || size < 1) throw new RangeError('size must be a positive integer');
  return Array.from({ length: Math.ceil(arr.length / size) },
                    (_, i) => arr.slice(i * size, i * size + size));
}
```
**Complexity:** O(n) time and space.
**Why this approach:** Computing the chunk count up front avoids a push-based loop and makes the
result length obvious.
**Common wrong answer:** No guard on `size`, so `chunk(arr, 0)` produces an infinite loop or an
`Infinity`-length array.
</details>

### 15. Deduplicate an array  `easy`
Handle primitives and objects by key.
<details><summary>Solution</summary>

```js
const unique = (arr) => [...new Set(arr)];

const uniqueBy = (arr, keyFn) => [...new Map(arr.map(x => [keyFn(x), x])).values()];
uniqueBy([{ id: 1 }, { id: 1, n: 2 }], x => x.id);   // [{ id: 1, n: 2 }] — last wins
```
**Complexity:** O(n) time and space.
**Why this approach:** `Set` uses SameValueZero, so it deduplicates `NaN` where an `indexOf`-based
filter can't.
**Common wrong answer:** `arr.filter((x, i) => arr.indexOf(x) === i)` — O(n²) and drops all `NaN`s.
</details>

### 16. Find the intersection and difference of two arrays  `easy`
<details><summary>Solution</summary>

```js
const intersection = (a, b) => { const s = new Set(b); return a.filter(x => s.has(x)); };
const difference   = (a, b) => { const s = new Set(b); return a.filter(x => !s.has(x)); };

// ES2025, when both are already Sets:
const inter = new Set(a).intersection(new Set(b));
```
**Complexity:** O(n + m) time, O(m) space.
**Why this approach:** Building the `Set` once outside the filter is the whole trick.
**Common wrong answer:** `a.filter(x => b.includes(x))` — O(n·m), which at 10,000 elements each is
100 million comparisons instead of 20,000.
</details>

### 17. Rotate an array by k positions  `easy`
**Example:** `([1,2,3,4,5], 2)` → `[4,5,1,2,3]`
<details><summary>Solution</summary>

```js
function rotate(arr, k) {
  if (arr.length === 0) return [];
  const n = ((k % arr.length) + arr.length) % arr.length;   // normalise negatives
  return [...arr.slice(-n || arr.length), ...arr.slice(0, -n || undefined)];
}
```
**Complexity:** O(n) time and space.
**Why this approach:** The double modulo handles `k` larger than the length and negative `k` in one
expression.
**Common wrong answer:** `arr.slice(-k)` without normalising, which returns the whole array for
`k = 0` (because `slice(0)` is everything) and misbehaves for `k > length`.
</details>

### 18. Zip and unzip arrays  `easy`
**Example:** `zip([1,2],['a','b'])` → `[[1,'a'],[2,'b']]`
<details><summary>Solution</summary>

```js
const zip = (...arrays) =>
  Array.from({ length: Math.min(...arrays.map(a => a.length)) },
             (_, i) => arrays.map(a => a[i]));

const unzip = (pairs) => zip(...pairs);
```
**Complexity:** O(n·k) time.
**Why this approach:** `Math.min` truncates to the shortest input, which is what most callers
expect; using `max` would pad with `undefined`.
**Common wrong answer:** Assuming equal lengths, then reading past the end of the shorter array.
</details>

### 19. Group an array by a key  `medium`
<details><summary>Solution</summary>

```js
const groupBy = (arr, keyFn) => Object.groupBy(arr, keyFn);      // ES2024

const groupByFallback = (arr, keyFn) =>
  arr.reduce((acc, x) => { (acc[keyFn(x)] ??= []).push(x); return acc; }, Object.create(null));
```
**Complexity:** O(n) time and space.
**Why this approach:** `Object.groupBy` returns a null-prototype object, so a key of `__proto__` or
`toString` can't collide or pollute.
**Common wrong answer:** `{}` as the accumulator with `acc[key] = acc[key] || []` — a key called
`constructor` finds the inherited function, which is truthy, and `.push` throws.
</details>

### 20. Sort an array of objects by multiple keys  `medium`
Support ascending and descending per key.
<details><summary>Solution</summary>

```js
function sortBy(arr, ...specs) {
  const compare = (a, b) => {
    for (const spec of specs) {
      const { key, dir = 'asc' } = typeof spec === 'string' ? { key: spec } : spec;
      const [x, y] = [a[key], b[key]];
      let c = 0;
      if (x == null && y == null) c = 0;
      else if (x == null) c = 1;                      // nullish sorts last, either direction
      else if (y == null) c = -1;
      else c = typeof x === 'string' ? x.localeCompare(y) : x < y ? -1 : x > y ? 1 : 0;
      if (c !== 0) return dir === 'desc' ? -c : c;
    }
    return 0;
  };
  return arr.toSorted(compare);
}

sortBy(users, { key: 'lastName' }, { key: 'age', dir: 'desc' });
```
**Complexity:** O(n log n · k) time.
**Why this approach:** `toSorted` doesn't mutate the caller's array, and negating after the nullish
check keeps nulls at the end in both directions.
**Common wrong answer:** Returning a boolean from the comparator — it coerces to `0` or `1`, never
`-1`, so the sort is undefined for the less-than case and results vary by engine.
</details>

### 21. Find the k largest elements  `medium`
<details><summary>Solution</summary>

```js
// For small k relative to n, a bounded insertion beats a full sort.
function kLargest(arr, k) {
  const top = [];
  for (const x of arr) {
    if (top.length < k) {
      top.push(x);
      top.sort((a, b) => a - b);              // k is small, so this is cheap
    } else if (x > top[0]) {
      top[0] = x;
      top.sort((a, b) => a - b);
    }
  }
  return top.reverse();
}
```
**Complexity:** O(n · k log k) time, O(k) space — versus O(n log n) and O(n) for a full sort. A real
min-heap gives O(n log k).
**Why this approach:** It never holds more than k elements, which matters when n is a stream you
can't materialise.
**Common wrong answer:** `arr.toSorted((a,b) => b-a).slice(0, k)` — perfectly fine for a small
array in memory, and the honest answer unless the interviewer specifies streaming or huge n.
</details>

### 22. Move all zeros to the end, in place  `medium`
Preserve the relative order of non-zeros.
**Example:** `[0,1,0,3,12]` → `[1,3,12,0,0]`
<details><summary>Solution</summary>

```js
function moveZeros(arr) {
  let write = 0;
  for (let read = 0; read < arr.length; read++) {
    if (arr[read] !== 0) arr[write++] = arr[read];
  }
  while (write < arr.length) arr[write++] = 0;
  return arr;
}
```
**Complexity:** O(n) time, O(1) space.
**Why this approach:** Two indices in one pass — the write index only advances on a non-zero, so
order is preserved without swapping.
**Common wrong answer:** `filter` plus concat, which is correct but allocates and isn't in place;
or `splice` inside a loop, which is O(n²) and skips elements as indices shift.
</details>

### 23. Implement a range function  `easy`
Support a step, including negative.
**Example:** `range(0, 10, 3)` → `[0,3,6,9]`
<details><summary>Solution</summary>

```js
function range(start, end, step = 1) {
  if (step === 0) throw new RangeError('step cannot be zero');
  const length = Math.max(0, Math.ceil((end - start) / step));
  return Array.from({ length }, (_, i) => start + i * step);
}
range(5, 0, -1);   // [5, 4, 3, 2, 1]
```
**Complexity:** O(n) time and space.
**Why this approach:** Computing `start + i * step` rather than accumulating avoids floating-point
drift — an accumulating loop with `step = 0.1` produces `0.30000000000000004`.
**Common wrong answer:** A `while (current < end)` loop, which never terminates for a negative step.
</details>

### 24. Find all pairs summing to a target  `medium`
Return unique pairs.
**Example:** `([1,2,3,4,3], 6)` → `[[2,4],[3,3]]`
<details><summary>Solution</summary>

```js
function pairsWithSum(arr, target) {
  const seen = new Map();                            // value → count
  const out = new Map();                             // canonical key → pair
  for (const x of arr) {
    const need = target - x;
    if (seen.get(need)) {
      const key = `${Math.min(x, need)},${Math.max(x, need)}`;
      out.set(key, [Math.min(x, need), Math.max(x, need)]);
    }
    seen.set(x, (seen.get(x) ?? 0) + 1);
  }
  return [...out.values()];
}
```
**Complexity:** O(n) time and space.
**Why this approach:** Checking before inserting means an element can't pair with itself unless it
appears twice, which is what makes `[3,3]` valid and a lone `3` with target 6 invalid.
**Common wrong answer:** A nested loop (O(n²)), or inserting before checking so `[3]` with target 6
returns `[3,3]`.
</details>

### 25. Implement `flatMap` behaviour by hand  `easy`
<details><summary>Solution</summary>

```js
function flatMap(arr, fn) {
  const out = [];
  for (let i = 0; i < arr.length; i++) {
    const result = fn(arr[i], i, arr);
    if (Array.isArray(result)) out.push(...result);   // one level only, per spec
    else out.push(result);
  }
  return out;
}
// The idiomatic use: map and filter in one pass
[1, 2, 3, 4].flatMap(n => (n % 2 ? [n * 10] : []));   // [10, 30]
```
**Complexity:** O(n + m) time in inputs and outputs.
**Why this approach:** Returning `[]` to drop an element is the pattern worth internalising — it
replaces a chained `filter().map()` with one traversal.
**Common wrong answer:** Flattening recursively, which `flatMap` never does.
</details>

### 26. Compute a moving average  `medium`
Window size k, returning one value per position from k onwards.
<details><summary>Solution</summary>

```js
function movingAverage(arr, k) {
  if (k < 1 || k > arr.length) return [];
  const out = [];
  let sum = 0;
  for (let i = 0; i < arr.length; i++) {
    sum += arr[i];
    if (i >= k) sum -= arr[i - k];                   // slide: add one, drop one
    if (i >= k - 1) out.push(sum / k);
  }
  return out;
}
```
**Complexity:** O(n) time, O(n) space for the output.
**Why this approach:** Maintaining a running sum makes each step O(1) rather than O(k).
**Common wrong answer:** `arr.slice(i, i+k).reduce(...)` per position — O(n·k), and for a 1M-element
array with k=100 that's 100 million additions instead of 1 million.
</details>

### 27. Shuffle an array uniformly  `medium`
<details><summary>Solution</summary>

```js
function shuffle(arr) {
  const out = arr.slice();
  for (let i = out.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));   // 0 <= j <= i, inclusive
    [out[i], out[j]] = [out[j], out[i]];
  }
  return out;
}
// For anything adversarial (a lottery, a game), use crypto:
const secureIndex = (n) => crypto.getRandomValues(new Uint32Array(1))[0] % n;   // slight bias
```
**Complexity:** O(n) time, O(n) space.
**Why this approach:** Fisher-Yates is provably uniform: each element has exactly 1/n probability
for each position.
**Common wrong answer:** `arr.sort(() => Math.random() - 0.5)` — badly non-uniform (some
permutations are orders of magnitude more likely) and the comparator is inconsistent, which the
spec says gives an implementation-defined result.
</details>

---

## 3. Objects

### 28. Deep clone an object  `medium`
Handle nested objects, arrays, `Date`, `Map`, `Set`, and cycles.
<details><summary>Solution</summary>

```js
const deepClone = (x) => structuredClone(x);          // ✅ prefer this

// By hand, when you need control over class instances:
function clone(value, seen = new WeakMap()) {
  if (value === null || typeof value !== 'object') return value;
  if (seen.has(value)) return seen.get(value);        // cycle: return the copy we started

  let copy;
  if (value instanceof Date) copy = new Date(value);
  else if (value instanceof RegExp) copy = new RegExp(value.source, value.flags);
  else if (value instanceof Map) copy = new Map();
  else if (value instanceof Set) copy = new Set();
  else if (Array.isArray(value)) copy = [];
  else copy = Object.create(Object.getPrototypeOf(value));

  seen.set(value, copy);                               // register BEFORE recursing

  if (value instanceof Map) for (const [k, v] of value) copy.set(clone(k, seen), clone(v, seen));
  else if (value instanceof Set) for (const v of value) copy.add(clone(v, seen));
  else for (const key of Reflect.ownKeys(value)) copy[key] = clone(value[key], seen);

  return copy;
}
```
**Complexity:** O(n) time and space in the number of nodes.
**Why this approach:** Registering the copy in `seen` before recursing is what makes cycles
terminate — a self-referential object finds its own partial copy.
**Common wrong answer:** `JSON.parse(JSON.stringify(x))` — throws on cycles, and loses `Date`
(becomes a string), `Map`/`Set` (become `{}`), `undefined`, functions, and `NaN` (becomes `null`).
</details>

### 29. Deep equality check  `medium`
<details><summary>Solution</summary>

```js
function deepEqual(a, b, seen = new WeakMap()) {
  if (Object.is(a, b)) return true;                          // handles NaN, and identity
  if (typeof a !== 'object' || typeof b !== 'object' || !a || !b) return false;
  if (Object.getPrototypeOf(a) !== Object.getPrototypeOf(b)) return false;

  if (seen.get(a) === b) return true;                        // already comparing this pair
  seen.set(a, b);

  if (a instanceof Date) return a.getTime() === b.getTime();
  if (a instanceof RegExp) return a.source === b.source && a.flags === b.flags;
  if (a instanceof Map) {
    if (a.size !== b.size) return false;
    for (const [k, v] of a) if (!b.has(k) || !deepEqual(v, b.get(k), seen)) return false;
    return true;
  }
  if (a instanceof Set) {
    if (a.size !== b.size) return false;
    for (const v of a) if (!b.has(v)) return false;          // reference equality for Set members
    return true;
  }

  const ka = Reflect.ownKeys(a), kb = Reflect.ownKeys(b);
  return ka.length === kb.length &&
         ka.every(k => Object.hasOwn(b, k) && deepEqual(a[k], b[k], seen));
}
```
**Complexity:** O(n) time in total properties.
**Why this approach:** `Object.is` at the top means `NaN` equals `NaN`, matching what test
frameworks do and what callers almost always want.
**Common wrong answer:** `JSON.stringify(a) === JSON.stringify(b)` — key order changes the result,
so `{a:1,b:2}` and `{b:2,a:1}` compare unequal.
</details>

### 30. Get a nested value by path safely  `easy`
**Example:** `get(obj, 'a.b[0].c', 'default')`
<details><summary>Solution</summary>

```js
function get(obj, path, fallback) {
  const keys = Array.isArray(path)
    ? path
    : path.replace(/\[(\d+)\]/g, '.$1').split('.').filter(Boolean);
  const value = keys.reduce((o, k) => (o == null ? undefined : o[k]), obj);
  return value === undefined ? fallback : value;
}
```
**Complexity:** O(depth) time.
**Why this approach:** Normalising `[0]` to `.0` means one split handles both notations, and the
`== null` guard short-circuits without throwing.
**Common wrong answer:** Returning the fallback for any falsy value, so `get({ n: 0 }, 'n', 5)`
returns `5`. Only `undefined` should trigger the fallback.
</details>

### 31. Set a nested value immutably  `medium`
Copy only the path to the change.
<details><summary>Solution</summary>

```js
function setIn(obj, path, value) {
  const keys = Array.isArray(path) ? path : path.split('.');
  const [key, ...rest] = keys;
  const child = rest.length ? setIn(obj?.[key] ?? (isIndex(rest[0]) ? [] : {}), rest, value) : value;
  if (Array.isArray(obj)) { const copy = obj.slice(); copy[key] = child; return copy; }
  return { ...obj, [key]: child };
}
const isIndex = (k) => /^\d+$/.test(String(k));
```
**Complexity:** O(depth + width of touched levels) time.
**Why this approach:** Untouched branches are reused by reference — structural sharing — so cost is
proportional to depth rather than data size. Verify with
`next.otherBranch === prev.otherBranch`.
**Common wrong answer:** `structuredClone` then mutate — correct but copies the entire tree, which
defeats any downstream identity-based memoisation.
</details>

### 32. Flatten and unflatten an object  `medium`
**Example:** `{ a: { b: 1 } }` ↔ `{ 'a.b': 1 }`
<details><summary>Solution</summary>

```js
function flattenObject(obj, prefix = '', out = {}) {
  for (const [k, v] of Object.entries(obj)) {
    const key = prefix ? `${prefix}.${k}` : k;
    // Only recurse into plain objects — Date, Map, and arrays are leaves.
    if (v !== null && typeof v === 'object' && !Array.isArray(v) &&
        Object.getPrototypeOf(v) === Object.prototype) {
      flattenObject(v, key, out);
    } else {
      out[key] = v;
    }
  }
  return out;
}

const unflatten = (flat) =>
  Object.entries(flat).reduce((acc, [path, value]) => setIn(acc, path, value), {});
```
**Complexity:** O(n) time in leaf count.
**Why this approach:** The prototype check stops `Date` and class instances being destructured into
their internals, which is the bug that makes flattened config unusable.
**Common wrong answer:** Recursing into anything `typeof === 'object'`, so a `Date` becomes `{}`
and an array becomes `{ '0': …, '1': … }`.
</details>

### 33. Pick and omit keys  `easy`
<details><summary>Solution</summary>

```js
const pick = (obj, keys) =>
  Object.fromEntries(keys.filter(k => Object.hasOwn(obj, k)).map(k => [k, obj[k]]));

const omit = (obj, keys) => {
  const drop = new Set(keys);
  return Object.fromEntries(Object.entries(obj).filter(([k]) => !drop.has(k)));
};
```
**Complexity:** `pick` O(k), `omit` O(n).
**Why this approach:** `Object.hasOwn` in `pick` means requesting a missing key omits it rather
than adding `undefined`.
**Common wrong answer:** `pick` without the `hasOwn` filter, producing `{ missing: undefined }`,
which then serialises differently and breaks deep-equality checks.
</details>

### 34. Safe deep merge  `medium`
Must resist prototype pollution.
<details><summary>Solution</summary>

```js
const FORBIDDEN = new Set(['__proto__', 'constructor', 'prototype']);

function safeMerge(target, ...sources) {
  for (const source of sources) {
    for (const key of Object.keys(source)) {              // own keys only
      if (FORBIDDEN.has(key)) continue;                   // the whole point
      const value = source[key];
      if (value !== null && typeof value === 'object' && !Array.isArray(value)) {
        if (!Object.hasOwn(target, key) || typeof target[key] !== 'object' || target[key] === null) {
          target[key] = Object.create(null);
        }
        safeMerge(target[key], value);
      } else {
        target[key] = value;
      }
    }
  }
  return target;
}
```
**Complexity:** O(n) time in total properties.
**Why this approach:** Three defences at once — key rejection, `Object.keys` skipping inherited
properties, and null-prototype intermediates with nothing to pollute.
**Common wrong answer:** The textbook recursive merge with `for...in`, which lets
`{"__proto__":{"isAdmin":true}}` set a property visible on every object in the process.
</details>

### 35. Convert an object to and from query-string form  `medium`
Handle nested objects and arrays.
<details><summary>Solution</summary>

```js
function toQuery(obj, prefix = '') {
  const params = new URLSearchParams();
  const walk = (value, key) => {
    if (value === undefined || value === null) return;
    if (Array.isArray(value)) value.forEach((v, i) => walk(v, `${key}[${i}]`));
    else if (typeof value === 'object') {
      for (const [k, v] of Object.entries(value)) walk(v, key ? `${key}[${k}]` : k);
    } else {
      params.append(key, String(value));                 // URLSearchParams handles encoding
    }
  };
  walk(obj, prefix);
  return params.toString();
}

const fromQuery = (qs) => Object.fromEntries(new URLSearchParams(qs));
```
**Complexity:** O(n) time in leaves.
**Why this approach:** `URLSearchParams` does the percent-encoding correctly, including spaces and
non-ASCII, which hand-rolled `encodeURIComponent` concatenation usually gets subtly wrong.
**Common wrong answer:** Building the string by hand with `+` and forgetting to encode, so a value
containing `&` splits into two parameters.
</details>

### 36. Count leaf nodes and maximum depth of an object  `easy`
<details><summary>Solution</summary>

```js
function stats(obj, depth = 1) {
  let leaves = 0, maxDepth = depth;
  for (const value of Object.values(obj)) {
    if (value !== null && typeof value === 'object') {
      const child = stats(value, depth + 1);
      leaves += child.leaves;
      maxDepth = Math.max(maxDepth, child.maxDepth);
    } else {
      leaves++;
    }
  }
  return { leaves, maxDepth };
}
```
**Complexity:** O(n) time, O(depth) stack.
**Why this approach:** Returning both values from one traversal avoids walking the tree twice.
**Common wrong answer:** No cycle guard — add a `WeakSet` if the input can be self-referential, or
this recurses forever.
</details>

### 37. Implement an object diff  `hard`
Report added, removed, and changed keys with old and new values.
<details><summary>Solution</summary>

```js
function diff(before, after, path = '') {
  const changes = [];
  const keys = new Set([...Object.keys(before ?? {}), ...Object.keys(after ?? {})]);

  for (const key of keys) {
    const at = path ? `${path}.${key}` : key;
    const inBefore = Object.hasOwn(before ?? {}, key);
    const inAfter = Object.hasOwn(after ?? {}, key);

    if (!inAfter) { changes.push({ type: 'removed', path: at, from: before[key] }); continue; }
    if (!inBefore) { changes.push({ type: 'added', path: at, to: after[key] }); continue; }

    const [a, b] = [before[key], after[key]];
    const bothPlain = a && b && typeof a === 'object' && typeof b === 'object' &&
                      !Array.isArray(a) && !Array.isArray(b);
    if (bothPlain) changes.push(...diff(a, b, at));
    else if (!deepEqual(a, b)) changes.push({ type: 'changed', path: at, from: a, to: b });
  }
  return changes;
}
```
**Complexity:** O(n) time in total properties.
**Why this approach:** Treating arrays as atomic values keeps the output readable — a per-index
array diff produces noise for any insertion, since every subsequent index "changed".
**Common wrong answer:** Iterating only `after`'s keys, so removals are never reported.
</details>

---

## 4. Functions and Closures

### 38. Implement `debounce` with options  `medium`
Support `leading`, `trailing`, `cancel`, and `flush`.
<details><summary>Solution</summary>

```js
function debounce(fn, wait = 250, { leading = false, trailing = true } = {}) {
  let timer = null, lastArgs = null, lastThis = null, result;

  const invoke = () => {
    timer = null;
    if (trailing && lastArgs) {
      result = fn.apply(lastThis, lastArgs);
      lastArgs = lastThis = null;
    }
  };

  function debounced(...args) {
    lastArgs = args; lastThis = this;
    const callNow = leading && timer === null;
    clearTimeout(timer);
    timer = setTimeout(invoke, wait);
    if (callNow) { result = fn.apply(this, args); lastArgs = lastThis = null; }
    return result;
  }

  debounced.cancel = () => { clearTimeout(timer); timer = null; lastArgs = lastThis = null; };
  debounced.flush = () => { if (timer) { clearTimeout(timer); invoke(); } return result; };
  debounced.pending = () => timer !== null;
  return debounced;
}
```
**Complexity:** O(1) per call.
**Why this approach:** Clearing `lastArgs` after a leading call is what stops it also firing on the
trailing edge — the single most-missed detail.
**Common wrong answer:** Creating the debounced function inside the event handler, so every
keystroke gets a fresh timer and nothing is ever debounced.
</details>

### 39. Implement `throttle`  `medium`
<details><summary>Solution</summary>

```js
function throttle(fn, wait = 250, { leading = true, trailing = true } = {}) {
  let last = 0, timer = null, lastArgs = null, lastThis = null;

  function throttled(...args) {
    const now = performance.now();
    if (!last && !leading) last = now;
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
**Complexity:** O(1) per call.
**Why this approach:** `performance.now()` is monotonic, so a system clock change can't make the
throttle fire early or stall.
**Common wrong answer:** A boolean flag with `setTimeout(() => flag = false, wait)`, which drops the
final call in a burst — the trailing edge is usually the one that matters.
</details>

### 40. Implement `curry`  `medium`
Support explicit arity.
<details><summary>Solution</summary>

```js
function curry(fn, arity = fn.length) {
  return function curried(...args) {
    return args.length >= arity
      ? fn.apply(this, args)
      : (...more) => curried.apply(this, [...args, ...more]);
  };
}

const add3 = curry((a, b, c) => a + b + c);
add3(1)(2)(3);   // 6
add3(1, 2)(3);   // 6
```
**Complexity:** O(1) per partial application.
**Why this approach:** The explicit `arity` parameter is necessary because `fn.length` stops
counting at the first default or rest parameter — `(a, b = 1) => {}` reports 1.
**Common wrong answer:** Relying on `fn.length` alone, so a function with a default parameter
never reaches full application and silently returns a function.
</details>

### 41. Implement `memoize` with an LRU bound  `medium`
<details><summary>Solution</summary>

```js
function memoize(fn, { max = 100, keyFn = (...a) => a.join(' ') } = {}) {
  const cache = new Map();                             // insertion order IS LRU order
  return function (...args) {
    const key = keyFn(...args);
    if (cache.has(key)) {
      const value = cache.get(key);
      cache.delete(key); cache.set(key, value);        // refresh recency
      return value;
    }
    const value = fn.apply(this, args);
    cache.set(key, value);
    if (cache.size > max) cache.delete(cache.keys().next().value);   // evict oldest
    return value;
  };
}
```
**Complexity:** O(1) amortised per call.
**Why this approach:** A `Map`'s guaranteed insertion order makes an LRU trivial — the first key is
always the least recently used, given the delete-and-reinsert on hit.
**Common wrong answer:** `JSON.stringify(args)` as the default key function: `{a:1,b:2}` and
`{b:2,a:1}` produce different keys, `undefined` values vanish, and it's slow. The ` ` join
above is faster but only correct for primitives — which is why it's the documented default, not a
universal answer.
</details>

### 42. Implement `once`  `easy`
<details><summary>Solution</summary>

```js
function once(fn) {
  let called = false, result;
  return function (...args) {
    if (called) return result;
    called = true;
    return (result = fn.apply(this, args));
  };
}
```
**Complexity:** O(1).
**Why this approach:** `apply` preserves the receiver, so `obj.method = once(obj.method)` still
works.
**Common wrong answer:** `fn = null` after calling to "free memory" — then a second call throws
instead of returning the cached result.
</details>

### 43. Implement `pipe` and `compose`, sync and async  `easy`
<details><summary>Solution</summary>

```js
const pipe      = (...fns) => (x) => fns.reduce((acc, fn) => fn(acc), x);
const compose   = (...fns) => (x) => fns.reduceRight((acc, fn) => fn(acc), x);
const pipeAsync = (...fns) => (x) => fns.reduce((p, fn) => p.then(fn), Promise.resolve(x));
```
**Complexity:** O(k) in the number of functions.
**Why this approach:** `pipe` reads in execution order, which matters more for maintenance than
matching mathematical convention.
**Common wrong answer:** `pipeAsync` with `reduce((acc, fn) => await fn(acc))` — you can't `await`
inside a non-async reducer, and making the reducer async returns an array of promises.
</details>

### 44. Implement a rate limiter  `medium`
Allow N calls per window, queueing the rest.
<details><summary>Solution</summary>

```js
function rateLimit(fn, { limit = 5, windowMs = 1000 } = {}) {
  const timestamps = [];
  const queue = [];

  const drain = () => {
    const now = performance.now();
    while (timestamps.length && timestamps[0] <= now - windowMs) timestamps.shift();

    while (queue.length && timestamps.length < limit) {
      const { args, thisArg, resolve, reject } = queue.shift();
      timestamps.push(performance.now());
      Promise.resolve().then(() => fn.apply(thisArg, args)).then(resolve, reject);
    }
    if (queue.length) setTimeout(drain, windowMs - (now - timestamps[0]) + 1);
  };

  return function (...args) {
    return new Promise((resolve, reject) => {
      queue.push({ args, thisArg: this, resolve, reject });
      drain();
    });
  };
}
```
**Complexity:** O(1) amortised per call.
**Why this approach:** A sliding window of timestamps is accurate; a fixed-window counter allows
2N calls across a boundary.
**Common wrong answer:** Dropping calls over the limit rather than queueing, which silently loses
work the caller believes was scheduled.
</details>

### 45. Implement function composition with a spy  `easy`
Wrap a function to record its calls.
<details><summary>Solution</summary>

```js
function spy(fn = () => {}) {
  const calls = [];
  const wrapper = function (...args) {
    const record = { args, thisArg: this };
    calls.push(record);
    try { return (record.returned = fn.apply(this, args)); }
    catch (err) { record.threw = err; throw err; }
  };
  wrapper.calls = calls;
  wrapper.callCount = () => calls.length;
  wrapper.calledWith = (...args) =>
    calls.some(c => c.args.length === args.length && c.args.every((a, i) => a === args[i]));
  wrapper.reset = () => { calls.length = 0; };
  return wrapper;
}
```
**Complexity:** O(1) per call, O(n) space in calls.
**Why this approach:** Recording throws as well as returns is what lets a test assert on the error
path without a second mechanism.
**Common wrong answer:** Recording only the arguments, so you can't assert on return values or
verify the function actually threw.
</details>

### 46. Implement `bind` from scratch  `medium`
Must work with `new`.
<details><summary>Solution</summary>

```js
Function.prototype.myBind = function (thisArg, ...bound) {
  if (typeof this !== 'function') throw new TypeError('Bind must be called on a function');
  const target = this;
  function wrapper(...args) {
    // With `new`, ignore thisArg and use the freshly constructed object.
    return target.apply(this instanceof wrapper ? this : thisArg, [...bound, ...args]);
  }
  wrapper.prototype = Object.create(target.prototype ?? null);
  return wrapper;
};
```
**Complexity:** O(1) per bind.
**Why this approach:** The `this instanceof wrapper` test distinguishes a construct call from an
ordinary one, which the spec requires — a bound function must remain `new`-able.
**Common wrong answer:** `return (...args) => target.apply(thisArg, [...bound, ...args])` — an arrow
can't be constructed, so `new BoundFn()` throws.
</details>

### 47. Build a private counter three ways  `easy`
Closure, `#private` field, and `WeakMap`.
<details><summary>Solution</summary>

```js
// 1. Closure
const makeCounter = (start = 0) => {
  let count = start;
  return { increment: () => ++count, get value() { return count; } };
};

// 2. Private field
class Counter {
  #count;
  constructor(start = 0) { this.#count = start; }
  increment() { return ++this.#count; }
  get value() { return this.#count; }
}

// 3. WeakMap — how privacy was done for class instances pre-ES2022
const counts = new WeakMap();
class LegacyCounter {
  constructor(start = 0) { counts.set(this, start); }
  increment() { const n = counts.get(this) + 1; counts.set(this, n); return n; }
  get value() { return counts.get(this); }
}
```
**Complexity:** O(1) per operation for all three.
**Why this approach:** All three are genuinely private. `#private` is the modern default; the
`WeakMap` version is worth knowing because it's what transpilers emit.
**Common wrong answer:** `this._count`, which is a naming convention and no barrier at all.
</details>

### 48. Implement `partial` with placeholders  `medium`
<details><summary>Solution</summary>

```js
const _ = Symbol('placeholder');

function partial(fn, ...preset) {
  return function (...later) {
    const args = preset.map(a => (a === _ ? later.shift() : a));
    return fn.apply(this, [...args, ...later]);      // remaining `later` args append
  };
}

const greet = (greeting, name, punct) => `${greeting}, ${name}${punct}`;
const hello = partial(greet, 'Hello', _, '!');
hello('Ada');   // 'Hello, Ada!'
```
**Complexity:** O(k) per call.
**Why this approach:** A `Symbol` placeholder can't collide with a legitimate argument value the way
`undefined` or a string sentinel could.
**Common wrong answer:** Using `undefined` as the placeholder, so `partial(fn, undefined, 2)` can't
distinguish "fill this in" from "pass undefined".
</details>

### 49. Implement a trampoline for deep recursion  `medium`
<details><summary>Solution</summary>

```js
const trampoline = (fn) => (...args) => {
  let result = fn(...args);
  while (typeof result === 'function') result = result();
  return result;
};

const sum = trampoline(function loop(n, acc = 0) {
  return n === 0 ? acc : () => loop(n - 1, acc + n);   // return a THUNK, don't recurse
});
sum(1_000_000);   // 500000500000 — no stack overflow
```
**Complexity:** O(n) time, O(1) stack.
**Why this approach:** JavaScript has no tail-call optimisation in practice, so converting recursion
to a loop over thunks is the only way to recurse a million deep.
**Common wrong answer:** Assuming TCO exists because it's in the ES2015 spec — only JavaScriptCore
ever shipped it.
</details>

---

## 5. Async

### 50. Implement `sleep` with cancellation  `easy`
<details><summary>Solution</summary>

```js
const sleep = (ms, { signal } = {}) => new Promise((resolve, reject) => {
  if (signal?.aborted) return reject(signal.reason);
  const id = setTimeout(resolve, ms);
  signal?.addEventListener('abort', () => { clearTimeout(id); reject(signal.reason); },
                           { once: true });
});
```
**Complexity:** O(1).
**Why this approach:** Clearing the timeout on abort matters — a pending timer is a GC root that
keeps the whole closure alive.
**Common wrong answer:** Checking `signal.aborted` only at the start, so an abort during the sleep
is ignored and the promise resolves anyway.
</details>

### 51. Implement `promisify`  `easy`
<details><summary>Solution</summary>

```js
const promisify = (fn) => function (...args) {
  return new Promise((resolve, reject) => {
    fn.call(this, ...args, (err, ...values) =>
      err ? reject(err) : resolve(values.length > 1 ? values : values[0]));
  });
};
```
**Complexity:** O(1).
**Why this approach:** `function` rather than an arrow, plus `fn.call(this, …)`, keeps it working
when the promisified function is used as a method.
**Common wrong answer:** An arrow wrapper, which loses `this` and breaks `promisify(obj.method)`.
</details>

### 52. Implement `Promise.all` from scratch  `medium`
<details><summary>Solution</summary>

```js
function all(iterable) {
  return new Promise((resolve, reject) => {
    const items = [...iterable];
    const results = new Array(items.length);
    let remaining = items.length;
    if (remaining === 0) return resolve(results);          // empty resolves immediately

    items.forEach((item, i) => {
      Promise.resolve(item).then(
        (value) => { results[i] = value; if (--remaining === 0) resolve(results); },
        reject                                            // first rejection wins
      );
    });
  });
}
```
**Complexity:** O(n) time and space.
**Why this approach:** Writing to `results[i]` by index preserves input order regardless of
settlement order, and the counter avoids scanning for completion.
**Common wrong answer:** `results.push(value)`, which orders by completion, not input.
</details>

### 53. Implement `Promise.allSettled`  `easy`
<details><summary>Solution</summary>

```js
const allSettled = (iterable) => Promise.all(
  [...iterable].map(item => Promise.resolve(item).then(
    (value) => ({ status: 'fulfilled', value }),
    (reason) => ({ status: 'rejected', reason })
  ))
);
```
**Complexity:** O(n).
**Why this approach:** Converting every rejection into a fulfilment means the underlying
`Promise.all` can never reject, which is exactly the semantics required.
**Common wrong answer:** Attempting it with a rejection handler on the outer `all`, which loses
the individual outcomes.
</details>

### 54. Implement `Promise.race` and `Promise.any`  `medium`
<details><summary>Solution</summary>

```js
const race = (iterable) => new Promise((resolve, reject) => {
  for (const item of iterable) Promise.resolve(item).then(resolve, reject);
});

function any(iterable) {
  return new Promise((resolve, reject) => {
    const items = [...iterable];
    const errors = new Array(items.length);
    let remaining = items.length;
    if (remaining === 0) return reject(new AggregateError([], 'All promises were rejected'));

    items.forEach((item, i) => {
      Promise.resolve(item).then(resolve, (err) => {
        errors[i] = err;
        if (--remaining === 0) reject(new AggregateError(errors, 'All promises were rejected'));
      });
    });
  });
}
```
**Complexity:** O(n).
**Why this approach:** `race` is trivially short because settling a promise twice is a no-op — the
first call wins and the rest are ignored.
**Common wrong answer:** `race` with an empty iterable is expected to stay pending forever, which
this handles correctly; adding a guard that rejects would be wrong.
</details>

### 55. Run promises with limited concurrency  `medium`
<details><summary>Solution</summary>

```js
async function pool(tasks, limit = 5) {
  const results = new Array(tasks.length);
  let next = 0;

  const worker = async () => {
    while (next < tasks.length) {
      const i = next++;                                // claim the index BEFORE awaiting
      try { results[i] = { status: 'fulfilled', value: await tasks[i]() }; }
      catch (reason) { results[i] = { status: 'rejected', reason }; }
    }
  };

  await Promise.all(Array.from({ length: Math.min(limit, tasks.length) }, worker));
  return results;
}
```
**Complexity:** O(n) time, O(limit) concurrent operations.
**Why this approach:** Incrementing `next` before the `await` is essential — reading it after would
let two workers claim the same task.
**Common wrong answer:** Chunking into batches of `limit`, which idles: the whole batch waits for
its slowest member before the next starts.
</details>

### 56. Implement retry with exponential backoff and jitter  `medium`
<details><summary>Solution</summary>

```js
async function retry(fn, { attempts = 3, base = 100, cap = 10_000, signal,
                           retryable = () => true } = {}) {
  for (let i = 0; ; i++) {
    try { return await fn(i); }
    catch (err) {
      const last = i >= attempts - 1;
      if (last || err.name === 'AbortError' || !retryable(err)) throw err;
      const backoff = Math.min(cap, base * 2 ** i);
      await sleep(backoff * (0.5 + Math.random() / 2), { signal });   // full jitter
    }
  }
}
```
**Complexity:** O(attempts) calls, O(1) space.
**Why this approach:** Jitter prevents a thundering herd — without it, every client that failed
together retries together and re-breaks the recovering service.
**Common wrong answer:** Retrying every error, including 400s and aborts. A 400 will never succeed,
so retrying it just hammers the endpoint four times for one guaranteed failure.
</details>

### 57. Add a timeout to a promise without leaking the timer  `easy`
<details><summary>Solution</summary>

```js
function withTimeout(promise, ms) {
  let id;
  const timeout = new Promise((_, reject) => {
    id = setTimeout(() => reject(new DOMException(`Timed out after ${ms}ms`, 'TimeoutError')), ms);
  });
  return Promise.race([promise, timeout]).finally(() => clearTimeout(id));
}
```
**Complexity:** O(1).
**Why this approach:** `finally` clears the timer on both paths, so a fast success doesn't leave a
pending timeout holding its closure alive.
**Common wrong answer:** Forgetting this doesn't *cancel* the operation — the underlying request
continues. For real cancellation, pass `AbortSignal.timeout(ms)` into `fetch`.
</details>

### 58. Deduplicate concurrent identical requests  `medium`
<details><summary>Solution</summary>

```js
function dedupe(fn, keyFn = (...args) => JSON.stringify(args)) {
  const inFlight = new Map();
  return function (...args) {
    const key = keyFn(...args);
    if (!inFlight.has(key)) {
      // Cache the PROMISE, and clear on settle so this stays in-flight-only.
      inFlight.set(key, Promise.resolve(fn.apply(this, args)).finally(() => inFlight.delete(key)));
    }
    return inFlight.get(key);
  };
}
```
**Complexity:** O(1) per call.
**Why this approach:** Deleting in `finally` keeps it a request deduplicator rather than a result
cache, which would need its own invalidation story.
**Common wrong answer:** Caching the resolved value, turning it into a cache with no TTL — so a
stale response is served forever.
</details>

### 59. Write an async generator that paginates an API  `medium`
<details><summary>Solution</summary>

```js
async function* paginate(url, { signal, pageSize = 100 } = {}) {
  let cursor = null;
  do {
    const params = new URLSearchParams({ limit: String(pageSize) });
    if (cursor) params.set('cursor', cursor);
    const res = await fetch(`${url}?${params}`, { signal });
    if (!res.ok) throw new Error(`${res.status} ${res.statusText}`);
    const { items, nextCursor } = await res.json();
    yield* items;                                    // yield individual items, not the page
    cursor = nextCursor;
  } while (cursor);
}

for await (const item of paginate('/api/users', { signal })) {
  process(item);
  if (done) break;                                   // breaking stops fetching further pages
}
```
**Complexity:** O(pages) requests, O(pageSize) memory.
**Why this approach:** Backpressure is free — the next page isn't fetched until the consumer asks
for an item beyond the current one, and `break` genuinely stops the work.
**Common wrong answer:** Fetching all pages into an array first, which defeats the point and holds
the whole dataset in memory.
</details>

### 60. Implement an event emitter that returns promises  `medium`
`once(event)` should resolve on the next occurrence.
<details><summary>Solution</summary>

```js
class AsyncEmitter {
  #listeners = new Map();

  on(event, handler) {
    if (!this.#listeners.has(event)) this.#listeners.set(event, new Set());
    this.#listeners.get(event).add(handler);
    return () => this.#listeners.get(event)?.delete(handler);
  }

  once(event, { signal } = {}) {
    return new Promise((resolve, reject) => {
      const off = this.on(event, (payload) => { off(); resolve(payload); });
      signal?.addEventListener('abort', () => { off(); reject(signal.reason); }, { once: true });
    });
  }

  emit(event, payload) {
    for (const handler of [...(this.#listeners.get(event) ?? [])]) {   // copy: handlers may unsubscribe
      try { handler(payload); }
      catch (err) { queueMicrotask(() => { throw err; }); }            // isolate failures
    }
  }
}
```
**Complexity:** O(k) per emit in listener count.
**Why this approach:** Copying before iterating is required because `once` unsubscribes during
dispatch, mutating the set you're walking.
**Common wrong answer:** No `signal` support on `once`, so a promise awaiting an event that never
fires is a permanent leak.
</details>

### 61. Implement a task queue that processes serially  `medium`
<details><summary>Solution</summary>

```js
class SerialQueue {
  #tail = Promise.resolve();
  #size = 0;

  add(task) {
    this.#size++;
    // Chain onto the tail so tasks run one at a time, in order.
    const result = this.#tail.then(task, task);       // run regardless of the previous outcome
    this.#tail = result.catch(() => {});              // swallow here so one failure can't stall the chain
    result.finally(() => this.#size--);
    return result;                                    // caller still sees the real rejection
  }

  get size() { return this.#size; }
  get idle() { return this.#tail; }
}
```
**Complexity:** O(1) per add.
**Why this approach:** Two references — the returned promise rejects for the caller, while the
internal tail swallows so the queue keeps draining. Conflating them means one failure blocks
everything after it.
**Common wrong answer:** `this.#tail = this.#tail.then(task)` alone, which stops the queue forever
on the first rejection.
</details>

### 62. Cancel an in-flight fetch on a new request  `easy`
<details><summary>Solution</summary>

```js
function createSearcher(url) {
  let controller = null;
  return async function search(query) {
    controller?.abort();                             // supersede the previous request
    controller = new AbortController();
    try {
      const res = await fetch(`${url}?q=${encodeURIComponent(query)}`,
                              { signal: controller.signal });
      if (!res.ok) throw new Error(`${res.status}`);
      return await res.json();
    } catch (err) {
      if (err.name === 'AbortError') return null;     // superseded, not a failure
      throw err;
    }
  };
}
```
**Complexity:** O(1).
**Why this approach:** Aborting the previous request means a slow response for an old query can
never overwrite a newer one — the out-of-order-response bug that debouncing alone doesn't fix.
**Common wrong answer:** Treating `AbortError` as a real error and showing it to the user.
</details>

---

## 6. Recursion

### 63. Compute Fibonacci three ways  `easy`
Naive, memoised, and iterative.
<details><summary>Solution</summary>

```js
const fibNaive = (n) => (n < 2 ? n : fibNaive(n - 1) + fibNaive(n - 2));   // O(2^n)

const fibMemo = memoize((n) => (n < 2 ? n : fibMemo(n - 1) + fibMemo(n - 2)));   // O(n)

function fib(n) {                                                          // O(n) time, O(1) space
  let [a, b] = [0, 1];
  for (let i = 0; i < n; i++) [a, b] = [b, a + b];
  return a;
}
```
**Complexity:** O(2ⁿ) / O(n) time and space / O(n) time and O(1) space.
**Why this approach:** The iterative version is strictly best — no stack, no cache. The memoised
one exists to demonstrate the technique.
**Common wrong answer:** Presenting the naive version as acceptable. `fibNaive(50)` makes about 2.5
billion calls; the iterative version does 50 additions. Also note `fib(79)` exceeds
`Number.MAX_SAFE_INTEGER` — use `BigInt` beyond that.
</details>

### 64. Flatten a deeply nested array recursively  `easy`
<details><summary>Solution</summary>

```js
const deepFlatten = (arr) =>
  arr.reduce((acc, x) => acc.concat(Array.isArray(x) ? deepFlatten(x) : x), []);

// Iterative, for arrays nested deeper than the stack allows:
function flattenIterative(arr) {
  const stack = [...arr], out = [];
  while (stack.length) {
    const x = stack.pop();
    if (Array.isArray(x)) stack.push(...x);
    else out.push(x);
  }
  return out.reverse();
}
```
**Complexity:** O(n) time; O(depth) stack recursive, O(n) heap iterative.
**Why this approach:** The iterative version can't overflow, which matters for
machine-generated data of unknown depth.
**Common wrong answer:** Forgetting the `reverse()` in the iterative version — `pop` and `push`
produce reversed output.
</details>

### 65. Traverse a tree depth-first and breadth-first  `easy`
<details><summary>Solution</summary>

```js
function* dfs(node) {
  yield node;
  for (const child of node.children ?? []) yield* dfs(child);
}

function* bfs(root) {
  const queue = [root];
  for (let i = 0; i < queue.length; i++) {          // index cursor: no O(n) shift()
    const node = queue[i];
    yield node;
    queue.push(...(node.children ?? []));
  }
}
```
**Complexity:** O(n) time; DFS O(depth) stack, BFS O(width) space.
**Why this approach:** Iterating with an index rather than `queue.shift()` avoids O(n) per removal,
which makes a naive BFS O(n²) on a wide tree.
**Common wrong answer:** `while (queue.length) queue.shift()` — correct but quadratic for large
trees.
</details>

### 66. Generate all permutations  `medium`
<details><summary>Solution</summary>

```js
function permutations(arr) {
  if (arr.length <= 1) return [arr.slice()];
  const out = [];
  for (let i = 0; i < arr.length; i++) {
    const rest = [...arr.slice(0, i), ...arr.slice(i + 1)];
    for (const perm of permutations(rest)) out.push([arr[i], ...perm]);
  }
  return out;
}
```
**Complexity:** O(n! · n) time and space.
**Why this approach:** Fixing each element as the head and permuting the rest is the clearest
formulation; a generator version avoids materialising all n! results at once.
**Common wrong answer:** Not handling duplicates when the input has them — `[1,1,2]` yields 6
permutations with repeats. Deduplicate by sorting and skipping equal siblings.
</details>

### 67. Generate the power set  `medium`
<details><summary>Solution</summary>

```js
const powerSet = (arr) =>
  arr.reduce((sets, x) => [...sets, ...sets.map(s => [...s, x])], [[]]);

// Bitmask alternative — clearer about why there are 2^n subsets:
const powerSetBits = (arr) =>
  Array.from({ length: 2 ** arr.length }, (_, mask) =>
    arr.filter((_, i) => mask & (1 << i)));
```
**Complexity:** O(2ⁿ · n) time and space.
**Why this approach:** The reduce doubles the set count per element, which is the direct expression
of 2ⁿ. The bitmask version fails above 31 elements due to `1 << i` overflow — irrelevant, since 2³¹
subsets is already impossible.
**Common wrong answer:** Forgetting the empty set, which is a member of every power set.
</details>

### 68. Solve the N-Queens problem  `hard`
Return one valid board for size n.
<details><summary>Solution</summary>

```js
function nQueens(n) {
  const cols = new Set(), diag1 = new Set(), diag2 = new Set();
  const placement = [];

  const place = (row) => {
    if (row === n) return true;
    for (let col = 0; col < n; col++) {
      // O(1) conflict check: a queen attacks along its column and both diagonals.
      if (cols.has(col) || diag1.has(row - col) || diag2.has(row + col)) continue;
      cols.add(col); diag1.add(row - col); diag2.add(row + col);
      placement.push(col);
      if (place(row + 1)) return true;
      placement.pop();                                     // backtrack
      cols.delete(col); diag1.delete(row - col); diag2.delete(row + col);
    }
    return false;
  };

  return place(0) ? placement : null;
}
```
**Complexity:** O(n!) worst case, far better in practice with pruning.
**Why this approach:** The three sets make conflict detection O(1) instead of scanning the board,
which is what makes n = 20 feasible.
**Common wrong answer:** Checking conflicts by scanning all previously placed queens — O(n) per
check, so O(n²) per placement, and the search becomes intractable around n = 12.
</details>

### 69. Compute the depth of a nested structure  `easy`
<details><summary>Solution</summary>

```js
function depth(value, seen = new WeakSet()) {
  if (value === null || typeof value !== 'object' || seen.has(value)) return 0;
  seen.add(value);
  const children = Array.isArray(value) ? value : Object.values(value);
  return 1 + Math.max(0, ...children.map(c => depth(c, seen)));
}
```
**Complexity:** O(n) time, O(depth) stack.
**Why this approach:** The `WeakSet` guard makes it terminate on cyclic input rather than
overflowing the stack.
**Common wrong answer:** `Math.max(...[])` returns `-Infinity`, so an empty object gives
`-Infinity` without the `0` seed.
</details>

### 70. Implement a recursive directory-style tree renderer  `medium`
<details><summary>Solution</summary>

```js
function renderTree(node, prefix = '', isLast = true) {
  const lines = [prefix + (prefix ? (isLast ? '└── ' : '├── ') : '') + node.name];
  const children = node.children ?? [];
  children.forEach((child, i) => {
    const childPrefix = prefix + (prefix ? (isLast ? '    ' : '│   ') : '');
    lines.push(...renderTree(child, childPrefix, i === children.length - 1));
  });
  return lines;
}
console.log(renderTree(tree).join('\n'));
```
**Complexity:** O(n) time in nodes.
**Why this approach:** Threading both the accumulated prefix and an `isLast` flag is what produces
correct continuation lines — the `│` must appear for every ancestor that has later siblings.
**Common wrong answer:** Using depth × spaces for indentation, which loses the vertical connectors
and makes deep trees unreadable.
</details>

---

## 7. Algorithms and Data Structures

### 71. Binary search  `easy`
Return the index or `-1`.
<details><summary>Solution</summary>

```js
function binarySearch(sorted, target) {
  let lo = 0, hi = sorted.length - 1;
  while (lo <= hi) {
    const mid = lo + Math.floor((hi - lo) / 2);      // avoids overflow in other languages; clearer here
    if (sorted[mid] === target) return mid;
    if (sorted[mid] < target) lo = mid + 1;
    else hi = mid - 1;
  }
  return -1;
}
```
**Complexity:** O(log n) time, O(1) space.
**Why this approach:** `lo <= hi` with `mid ± 1` updates is the formulation that terminates for
every input, including a single-element and empty array.
**Common wrong answer:** `while (lo < hi)` — misses the target when the range narrows to one
element. Also `hi = mid` instead of `mid - 1`, which loops forever.
</details>

### 72. Implement a Stack and a Queue  `easy`
<details><summary>Solution</summary>

```js
class Stack {
  #items = [];
  push(x) { this.#items.push(x); return this; }
  pop() { return this.#items.pop(); }
  peek() { return this.#items.at(-1); }
  get size() { return this.#items.length; }
}

class Queue {
  #items = [];
  #head = 0;                                        // cursor instead of shift(): O(1) dequeue
  enqueue(x) { this.#items.push(x); return this; }
  dequeue() {
    if (this.#head >= this.#items.length) return undefined;
    const x = this.#items[this.#head];
    this.#items[this.#head++] = undefined;          // release the reference
    if (this.#head > 32 && this.#head * 2 >= this.#items.length) {
      this.#items = this.#items.slice(this.#head);  // amortised compaction
      this.#head = 0;
    }
    return x;
  }
  get size() { return this.#items.length - this.#head; }
}
```
**Complexity:** O(1) amortised for all operations.
**Why this approach:** `Array.shift()` is O(n) because it reindexes; a head cursor with periodic
compaction gives O(1) dequeue without unbounded memory growth.
**Common wrong answer:** `shift()` for dequeue — fine for small queues, quadratic for a million
items.
</details>

### 73. Implement a LinkedList with reversal  `medium`
<details><summary>Solution</summary>

```js
class LinkedList {
  #head = null;
  #size = 0;

  push(value) {
    const node = { value, next: null };
    if (!this.#head) this.#head = node;
    else { let cur = this.#head; while (cur.next) cur = cur.next; cur.next = node; }
    this.#size++;
    return this;
  }

  reverse() {
    let prev = null, cur = this.#head;
    while (cur) {
      const next = cur.next;    // save before overwriting
      cur.next = prev;
      prev = cur;
      cur = next;
    }
    this.#head = prev;
    return this;
  }

  *[Symbol.iterator]() { let cur = this.#head; while (cur) { yield cur.value; cur = cur.next; } }
  get size() { return this.#size; }
}
```
**Complexity:** `push` O(n) (keep a tail pointer for O(1)), `reverse` O(n) time and O(1) space.
**Why this approach:** Saving `next` before reassigning is the entire trick to iterative reversal —
without it you lose the rest of the list.
**Common wrong answer:** Recursive reversal, which is elegant and overflows on a long list.
</details>

### 74. Implement an LRU cache  `medium`
O(1) get and set.
<details><summary>Solution</summary>

```js
class LRUCache {
  #map = new Map();                                  // insertion order IS recency order
  #max;
  constructor(max = 100) { this.#max = max; }

  get(key) {
    if (!this.#map.has(key)) return undefined;
    const value = this.#map.get(key);
    this.#map.delete(key); this.#map.set(key, value);          // move to most-recent
    return value;
  }

  set(key, value) {
    this.#map.delete(key);                                     // reinsert to refresh position
    this.#map.set(key, value);
    if (this.#map.size > this.#max) this.#map.delete(this.#map.keys().next().value);
    return this;
  }

  get size() { return this.#map.size; }
}
```
**Complexity:** O(1) for both, amortised.
**Why this approach:** `Map` gives insertion-ordered iteration plus O(1) delete, which is exactly a
doubly-linked list plus hash map — the classic answer, for free.
**Common wrong answer:** Tracking recency in a separate array and using `indexOf` to find entries,
which makes `get` O(n).
</details>

### 75. Implement a min-heap / priority queue  `medium`
<details><summary>Solution</summary>

```js
class MinHeap {
  #heap = [];
  #compare;
  constructor(compare = (a, b) => a - b) { this.#compare = compare; }

  push(value) {
    this.#heap.push(value);
    let i = this.#heap.length - 1;
    while (i > 0) {
      const parent = (i - 1) >> 1;
      if (this.#compare(this.#heap[i], this.#heap[parent]) >= 0) break;
      [this.#heap[i], this.#heap[parent]] = [this.#heap[parent], this.#heap[i]];
      i = parent;
    }
    return this;
  }

  pop() {
    if (this.#heap.length <= 1) return this.#heap.pop();
    const top = this.#heap[0];
    this.#heap[0] = this.#heap.pop();                 // move last to root, then sift down
    let i = 0;
    for (;;) {
      const [l, r] = [2 * i + 1, 2 * i + 2];
      let smallest = i;
      if (l < this.#heap.length && this.#compare(this.#heap[l], this.#heap[smallest]) < 0) smallest = l;
      if (r < this.#heap.length && this.#compare(this.#heap[r], this.#heap[smallest]) < 0) smallest = r;
      if (smallest === i) break;
      [this.#heap[i], this.#heap[smallest]] = [this.#heap[smallest], this.#heap[i]];
      i = smallest;
    }
    return top;
  }

  peek() { return this.#heap[0]; }
  get size() { return this.#heap.length; }
}
```
**Complexity:** O(log n) push and pop, O(1) peek.
**Why this approach:** An array-backed heap needs no node allocation — parent and children are
index arithmetic.
**Common wrong answer:** A sorted array, giving O(n) insert. Fine for tiny queues, wrong for a
scheduler.
</details>

### 76. Implement a Trie with prefix search  `medium`
<details><summary>Solution</summary>

```js
class Trie {
  #root = { children: new Map(), isWord: false };

  insert(word) {
    let node = this.#root;
    for (const c of word) {
      if (!node.children.has(c)) node.children.set(c, { children: new Map(), isWord: false });
      node = node.children.get(c);
    }
    node.isWord = true;
    return this;
  }

  #find(prefix) {
    let node = this.#root;
    for (const c of prefix) {
      node = node.children.get(c);
      if (!node) return null;
    }
    return node;
  }

  has(word) { return this.#find(word)?.isWord ?? false; }

  startsWith(prefix) {
    const node = this.#find(prefix);
    if (!node) return [];
    const out = [];
    const walk = (n, acc) => {
      if (n.isWord) out.push(acc);
      for (const [c, child] of n.children) walk(child, acc + c);
    };
    walk(node, prefix);
    return out;
  }
}
```
**Complexity:** O(m) insert and lookup for word length m; prefix search O(m + k) in results.
**Why this approach:** Prefix lookup is independent of dictionary size, which is why autocomplete
uses a trie rather than filtering an array.
**Common wrong answer:** `words.filter(w => w.startsWith(p))` — O(n·m) per keystroke, which is
fine for 100 words and unusable for 100,000.
</details>

### 77. Implement quicksort and mergesort  `medium`
<details><summary>Solution</summary>

```js
function quicksort(arr) {
  if (arr.length <= 1) return arr;
  const pivot = arr[arr.length >> 1];               // middle pivot: avoids worst case on sorted input
  const less = [], equal = [], greater = [];
  for (const x of arr) (x < pivot ? less : x > pivot ? greater : equal).push(x);
  return [...quicksort(less), ...equal, ...quicksort(greater)];
}

function mergesort(arr) {
  if (arr.length <= 1) return arr;
  const mid = arr.length >> 1;
  const [a, b] = [mergesort(arr.slice(0, mid)), mergesort(arr.slice(mid))];
  const out = [];
  let i = 0, j = 0;
  while (i < a.length && j < b.length) out.push(a[i] <= b[j] ? a[i++] : b[j++]);
  return [...out, ...a.slice(i), ...b.slice(j)];
}
```
**Complexity:** quicksort O(n log n) average / O(n²) worst; mergesort O(n log n) always, O(n) space.
**Why this approach:** The three-way partition handles duplicates in O(n) rather than degrading.
Mergesort is stable, which quicksort isn't — that's why `Array.sort` is required to be stable and
engines use a merge-based algorithm.
**Common wrong answer:** `arr[0]` as the pivot, which is O(n²) on already-sorted input — exactly
the common real-world case.
</details>

### 78. Detect a cycle in a linked list  `medium`
<details><summary>Solution</summary>

```js
function hasCycle(head) {
  let slow = head, fast = head;
  while (fast?.next) {
    slow = slow.next;
    fast = fast.next.next;                          // moves twice as fast
    if (slow === fast) return true;                 // must meet inside a cycle
  }
  return false;
}
```
**Complexity:** O(n) time, O(1) space.
**Why this approach:** Floyd's algorithm needs no extra memory — if there's a cycle, the fast
pointer laps the slow one and they collide.
**Common wrong answer:** A `Set` of visited nodes. Correct and O(n) time, but O(n) space, which is
the point of the question.
</details>

### 79. Two-sum, three-sum  `medium`
<details><summary>Solution</summary>

```js
function twoSum(nums, target) {
  const seen = new Map();                            // value → index
  for (let i = 0; i < nums.length; i++) {
    const need = target - nums[i];
    if (seen.has(need)) return [seen.get(need), i];
    seen.set(nums[i], i);
  }
  return null;
}

function threeSum(nums, target = 0) {
  const sorted = nums.toSorted((a, b) => a - b);
  const out = [];
  for (let i = 0; i < sorted.length - 2; i++) {
    if (i > 0 && sorted[i] === sorted[i - 1]) continue;         // skip duplicate anchors
    let lo = i + 1, hi = sorted.length - 1;
    while (lo < hi) {
      const sum = sorted[i] + sorted[lo] + sorted[hi];
      if (sum === target) {
        out.push([sorted[i], sorted[lo], sorted[hi]]);
        while (sorted[lo] === sorted[++lo] && lo < hi);          // skip duplicates
        while (sorted[hi] === sorted[--hi] && lo < hi);
      } else if (sum < target) lo++;
      else hi--;
    }
  }
  return out;
}
```
**Complexity:** twoSum O(n); threeSum O(n²) after an O(n log n) sort.
**Why this approach:** Sorting enables the two-pointer inner loop, turning what would be O(n³) into
O(n²).
**Common wrong answer:** Checking `seen` after inserting in twoSum, so an element pairs with itself.
</details>

### 80. Find the longest common prefix  `easy`
<details><summary>Solution</summary>

```js
function longestCommonPrefix(strings) {
  if (strings.length === 0) return '';
  let prefix = strings[0];
  for (const s of strings.slice(1)) {
    while (!s.startsWith(prefix)) {
      prefix = prefix.slice(0, -1);
      if (!prefix) return '';
    }
  }
  return prefix;
}
```
**Complexity:** O(n·m) time in count and prefix length.
**Why this approach:** Shrinking a candidate prefix exits early — one string sharing nothing ends
the whole search immediately.
**Common wrong answer:** Comparing character by character across all strings simultaneously, which
is correct but needs careful bounds handling for the shortest string.
</details>

### 81. Implement a debounced binary-search-based autocomplete index  `hard`
Given sorted words, return all with a prefix in O(log n + k).
<details><summary>Solution</summary>

```js
function prefixRange(sorted, prefix) {
  // Lower bound: first index where sorted[i] >= prefix
  const lower = (() => {
    let lo = 0, hi = sorted.length;
    while (lo < hi) { const m = (lo + hi) >> 1; sorted[m] < prefix ? lo = m + 1 : hi = m; }
    return lo;
  })();

  // Upper bound: first index NOT starting with prefix.
  // Incrementing the last character gives the exclusive end of the range.
  const end = prefix.slice(0, -1) + String.fromCharCode(prefix.charCodeAt(prefix.length - 1) + 1);
  const upper = (() => {
    let lo = lower, hi = sorted.length;
    while (lo < hi) { const m = (lo + hi) >> 1; sorted[m] < end ? lo = m + 1 : hi = m; }
    return lo;
  })();

  return sorted.slice(lower, upper);
}
```
**Complexity:** O(log n + k) time, O(k) space.
**Why this approach:** Two binary searches bracket the range without scanning. The
increment-last-character trick converts a prefix test into an ordinary comparison.
**Common wrong answer:** One binary search to find any match then linear scanning outward, which is
O(n) when the prefix matches most of the list.
</details>

### 82. Compute Levenshtein distance  `hard`
<details><summary>Solution</summary>

```js
function levenshtein(a, b) {
  if (a === b) return 0;
  if (!a.length) return b.length;
  if (!b.length) return a.length;

  // Only two rows are ever needed, so space is O(min(n, m)) not O(n·m).
  let prev = Array.from({ length: b.length + 1 }, (_, i) => i);
  for (let i = 1; i <= a.length; i++) {
    const curr = [i];
    for (let j = 1; j <= b.length; j++) {
      curr[j] = Math.min(
        prev[j] + 1,                                       // deletion
        curr[j - 1] + 1,                                   // insertion
        prev[j - 1] + (a[i - 1] === b[j - 1] ? 0 : 1)       // substitution
      );
    }
    prev = curr;
  }
  return prev[b.length];
}
```
**Complexity:** O(n·m) time, O(min(n, m)) space.
**Why this approach:** The recurrence only ever reads the previous row, so the full matrix is
unnecessary — for two 10,000-character strings that's 20KB instead of 400MB.
**Common wrong answer:** Naive recursion without memoisation, which is exponential.
</details>

### 83. Implement a simple observable store with selectors  `medium`
<details><summary>Solution</summary>

```js
function createStore(initial) {
  let state = initial;
  const subscribers = new Set();

  return {
    getState: () => state,
    setState(updater) {
      const next = typeof updater === 'function' ? updater(state) : updater;
      if (next === state) return;                         // identity check: skip no-op updates
      const prev = state;
      state = next;
      for (const sub of [...subscribers]) sub(state, prev);
    },
    subscribe(selector, listener) {
      let last = selector(state);
      const wrapped = (next) => {
        const selected = selector(next);
        if (selected === last) return;                    // only fire when the SLICE changes
        last = selected;
        listener(selected);
      };
      subscribers.add(wrapped);
      return () => subscribers.delete(wrapped);
    },
  };
}
```
**Complexity:** O(k) per update in subscriber count.
**Why this approach:** Per-subscriber selector memoisation means a component watching `state.user`
doesn't re-render when `state.cart` changes — which requires immutable updates upstream so
identity comparison is meaningful.
**Common wrong answer:** Deep-comparing selected values, which is slower than the render it saves
and unnecessary if updates are immutable.
</details>

### 84. Implement a virtual-list window calculator  `medium`
<details><summary>Solution</summary>

```js
function windowFor({ scrollTop, viewportHeight, rowHeight, total, overscan = 3 }) {
  const first = Math.max(0, Math.floor(scrollTop / rowHeight) - overscan);
  const visible = Math.ceil(viewportHeight / rowHeight);
  const last = Math.min(total - 1, first + visible + overscan * 2);
  return {
    first,
    last,
    paddingTop: first * rowHeight,
    paddingBottom: Math.max(0, (total - last - 1) * rowHeight),   // keeps the scrollbar honest
  };
}
```
**Complexity:** O(1).
**Why this approach:** Overscan on both sides prevents blank rows appearing during fast scrolls,
and the padding values preserve total scroll height so the scrollbar doesn't jump.
**Common wrong answer:** Omitting `paddingBottom`, so the scroll height shrinks as you scroll and
the scrollbar thumb behaves erratically.
</details>

### 85. Detect whether two rectangles overlap, and compute the union  `easy`
<details><summary>Solution</summary>

```js
const overlaps = (a, b) =>
  a.x < b.x + b.width && b.x < a.x + a.width &&
  a.y < b.y + b.height && b.y < a.y + a.height;

const union = (a, b) => {
  const x = Math.min(a.x, b.x), y = Math.min(a.y, b.y);
  return {
    x, y,
    width: Math.max(a.x + a.width, b.x + b.width) - x,
    height: Math.max(a.y + a.height, b.y + b.height) - y,
  };
};
```
**Complexity:** O(1).
**Why this approach:** Testing for *non*-overlap on each axis and negating is fewer comparisons and
harder to get wrong than enumerating overlap cases.
**Common wrong answer:** `<=` instead of `<`, which reports touching edges as overlapping — usually
wrong for hit-testing and collision.
</details>

---

## 8. Polyfills

### 86. `Array.prototype.map`  `easy`
<details><summary>Solution</summary>

```js
Array.prototype.myMap = function (fn, thisArg) {
  if (typeof fn !== 'function') throw new TypeError(`${fn} is not a function`);
  const out = new Array(this.length);
  for (let i = 0; i < this.length; i++) {
    if (i in this) out[i] = fn.call(thisArg, this[i], i, this);   // preserve holes
  }
  return out;
};
```
**Complexity:** O(n).
**Why this approach:** `i in this` skips holes in sparse arrays, matching the spec — the result
keeps the hole rather than filling it with `undefined`.
**Common wrong answer:** `forEach`-and-push, which compacts sparse arrays and changes the length.
</details>

### 87. `Array.prototype.filter`  `easy`
<details><summary>Solution</summary>

```js
Array.prototype.myFilter = function (fn, thisArg) {
  const out = [];
  for (let i = 0; i < this.length; i++) {
    if (i in this && fn.call(thisArg, this[i], i, this)) out.push(this[i]);
  }
  return out;
};
```
**Complexity:** O(n).
**Why this approach:** `filter` *is* specified to compact, unlike `map` — the output has no holes.
**Common wrong answer:** Caching `this.length` is actually correct here (the spec fixes length up
front), but mutating the array inside the callback then behaves surprisingly either way.
</details>

### 88. `Array.prototype.reduce`  `medium`
<details><summary>Solution</summary>

```js
Array.prototype.myReduce = function (fn, ...initial) {
  let i = 0, acc;
  if (initial.length) {
    acc = initial[0];
  } else {
    while (i < this.length && !(i in this)) i++;                  // skip leading holes
    if (i >= this.length) throw new TypeError('Reduce of empty array with no initial value');
    acc = this[i++];
  }
  for (; i < this.length; i++) if (i in this) acc = fn(acc, this[i], i, this);
  return acc;
};
```
**Complexity:** O(n).
**Why this approach:** Using a rest parameter distinguishes "no initial value" from
"initial value of `undefined`", which a default parameter can't.
**Common wrong answer:** `function (fn, acc)` then checking `acc === undefined` — so
`[1,2].reduce(fn, undefined)` wrongly uses `1` as the seed.
</details>

### 89. `Array.prototype.forEach`  `easy`
<details><summary>Solution</summary>

```js
Array.prototype.myForEach = function (fn, thisArg) {
  for (let i = 0; i < this.length; i++) {
    if (i in this) fn.call(thisArg, this[i], i, this);
  }
  // returns undefined, deliberately — it's a side-effect method
};
```
**Complexity:** O(n).
**Why this approach:** Returning `undefined` is part of the contract; returning the array would
invite chaining that the real method doesn't support.
**Common wrong answer:** Trying to support `break` by returning a value — you can't; use `some`,
`every`, or a `for...of`.
</details>

### 90. `Function.prototype.call` and `apply`  `medium`
<details><summary>Solution</summary>

```js
Function.prototype.myCall = function (thisArg, ...args) {
  thisArg = thisArg ?? globalThis;
  const key = Symbol('fn');                          // Symbol: can't collide with a real property
  const target = Object(thisArg);                    // boxes primitives, as the spec does
  target[key] = this;
  try { return target[key](...args); }                // implicit binding does the work
  finally { delete target[key]; }
};

Function.prototype.myApply = function (thisArg, args = []) {
  return this.myCall(thisArg, ...args);
};
```
**Complexity:** O(1) plus the call.
**Why this approach:** Temporarily installing the function as a property makes the engine's implicit
`this` binding do the work, and a `Symbol` key guarantees no collision.
**Common wrong answer:** A string key like `'__fn__'`, which overwrites a real property if one
exists — and `delete` then removes the caller's data.
</details>

### 91. `Function.prototype.bind`  `medium`
<details><summary>Solution</summary>

```js
Function.prototype.myBind = function (thisArg, ...bound) {
  if (typeof this !== 'function') throw new TypeError('Bind must be called on a function');
  const target = this;
  function wrapper(...args) {
    return target.apply(this instanceof wrapper ? this : thisArg, [...bound, ...args]);
  }
  wrapper.prototype = Object.create(target.prototype ?? null);
  return wrapper;
};
```
**Complexity:** O(1).
**Why this approach:** See challenge 46 — the `instanceof` check preserves construct behaviour.
**Common wrong answer:** An arrow function wrapper, which can't be `new`-ed.
</details>

### 92. `new` operator  `medium`
<details><summary>Solution</summary>

```js
function myNew(Constructor, ...args) {
  if (typeof Constructor !== 'function') throw new TypeError('not a constructor');
  const obj = Object.create(Constructor.prototype);   // link the prototype
  const result = Constructor.apply(obj, args);         // run with `this` = obj
  // An explicitly returned OBJECT wins; a returned primitive is ignored.
  return (result !== null && (typeof result === 'object' || typeof result === 'function'))
    ? result : obj;
}
```
**Complexity:** O(1).
**Why this approach:** The return-value rule is the part people miss, and it's why the factory
pattern can hide inside a constructor.
**Common wrong answer:** Always returning `obj`, which breaks any constructor that deliberately
returns something else (a Proxy wrapper, a cached instance).
</details>

### 93. `instanceof`  `easy`
<details><summary>Solution</summary>

```js
function myInstanceof(obj, Constructor) {
  if (obj === null || (typeof obj !== 'object' && typeof obj !== 'function')) return false;
  const target = Constructor.prototype;
  let proto = Object.getPrototypeOf(obj);
  while (proto !== null) {
    if (proto === target) return true;
    proto = Object.getPrototypeOf(proto);            // walk the whole chain
  }
  return false;
}
```
**Complexity:** O(depth) in chain length.
**Why this approach:** Walking the chain is what distinguishes `instanceof` from a single-level
prototype comparison.
**Common wrong answer:** `Object.getPrototypeOf(obj) === Constructor.prototype` — only checks the
direct prototype, so a subclass instance fails against its parent.
</details>

### 94. `Promise` from scratch  `hard`
<details><summary>Solution</summary>

```js
class MyPromise {
  #state = 'pending';
  #value;
  #callbacks = [];

  constructor(executor) {
    const settle = (state) => (value) => {
      if (this.#state !== 'pending') return;                // settle exactly once
      // A thenable resolution value must be assimilated, not stored.
      if (state === 'fulfilled' && value && typeof value.then === 'function') {
        return value.then(settle('fulfilled'), settle('rejected'));
      }
      this.#state = state;
      this.#value = value;
      // Always asynchronous: callbacks run as microtasks, never synchronously.
      this.#callbacks.forEach(cb => queueMicrotask(cb));
      this.#callbacks = [];
    };
    try { executor(settle('fulfilled'), settle('rejected')); }
    catch (err) { settle('rejected')(err); }
  }

  then(onFulfilled, onRejected) {
    return new MyPromise((resolve, reject) => {
      const run = () => {
        const handler = this.#state === 'fulfilled' ? onFulfilled : onRejected;
        if (typeof handler !== 'function') {
          // Pass through when no handler is supplied — this is what makes chains work.
          return this.#state === 'fulfilled' ? resolve(this.#value) : reject(this.#value);
        }
        try { resolve(handler(this.#value)); }
        catch (err) { reject(err); }
      };
      this.#state === 'pending' ? this.#callbacks.push(run) : queueMicrotask(run);
    });
  }

  catch(onRejected) { return this.then(undefined, onRejected); }

  finally(onFinally) {
    return this.then(
      (value) => MyPromise.resolve(onFinally?.()).then(() => value),
      (err) => MyPromise.resolve(onFinally?.()).then(() => { throw err; })
    );
  }

  static resolve(value) {
    return value instanceof MyPromise ? value : new MyPromise((res) => res(value));
  }
  static reject(reason) { return new MyPromise((_, rej) => rej(reason)); }
}
```
**Complexity:** O(1) per operation.
**Why this approach:** Three details make it spec-compliant: settling exactly once, always
asynchronous callbacks, and thenable assimilation. `finally` passing the value through is the
fourth thing people forget.
**Common wrong answer:** Calling handlers synchronously when the promise is already settled, which
breaks the guarantee that `.then` never runs before the current task finishes.
</details>

### 95. `Promise.all`  `medium`
See challenge 52.
<details><summary>Solution</summary>

```js
Promise.myAll = (iterable) => new Promise((resolve, reject) => {
  const items = [...iterable];
  const results = new Array(items.length);
  let remaining = items.length;
  if (!remaining) return resolve(results);
  items.forEach((item, i) => Promise.resolve(item).then(
    (value) => { results[i] = value; if (--remaining === 0) resolve(results); },
    reject));
});
```
**Complexity:** O(n).
**Why this approach:** Index assignment preserves input order.
**Common wrong answer:** Not handling an empty iterable, leaving the promise pending forever.
</details>

### 96. `Promise.race`  `easy`
<details><summary>Solution</summary>

```js
Promise.myRace = (iterable) => new Promise((resolve, reject) => {
  for (const item of iterable) Promise.resolve(item).then(resolve, reject);
});
```
**Complexity:** O(n) to attach handlers.
**Why this approach:** No bookkeeping needed — settling an already-settled promise is a no-op, so
the first winner sticks.
**Common wrong answer:** Adding a "first result" flag, which is harmless but reveals a
misunderstanding of promise semantics.
</details>

### 97. `debounce`  `medium`
See challenge 38.
<details><summary>Solution</summary>

```js
const debounce = (fn, wait) => {
  let timer;
  const debounced = (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), wait);
  };
  debounced.cancel = () => clearTimeout(timer);
  return debounced;
};
```
**Complexity:** O(1) per call.
**Why this approach:** This minimal version is what an interviewer usually wants; add
leading/trailing/flush only if asked.
**Common wrong answer:** Not returning a `cancel`, so a pending call fires after teardown.
</details>

### 98. `throttle`  `medium`
<details><summary>Solution</summary>

```js
const throttle = (fn, wait) => {
  let last = 0, timer = null, lastArgs = null;
  return (...args) => {
    const now = Date.now();
    lastArgs = args;
    if (now - last >= wait) { last = now; fn(...args); lastArgs = null; }
    else if (!timer) {
      timer = setTimeout(() => {
        last = Date.now(); timer = null;
        if (lastArgs) fn(...lastArgs);
      }, wait - (now - last));
    }
  };
};
```
**Complexity:** O(1) per call.
**Why this approach:** The trailing call ensures the final event in a burst isn't dropped, which is
usually the one that matters (the last scroll position).
**Common wrong answer:** A pure leading-edge throttle that drops the trailing call, so a scroll
handler never sees the final position.
</details>

### 99. `deepClone`  `medium`
See challenge 28.
<details><summary>Solution</summary>

```js
function deepClone(value, seen = new WeakMap()) {
  if (value === null || typeof value !== 'object') return value;
  if (seen.has(value)) return seen.get(value);
  const copy = Array.isArray(value) ? [] : Object.create(Object.getPrototypeOf(value));
  seen.set(value, copy);
  for (const key of Reflect.ownKeys(value)) copy[key] = deepClone(value[key], seen);
  return copy;
}
```
**Complexity:** O(n).
**Why this approach:** `Reflect.ownKeys` includes symbol keys, which `Object.keys` drops.
**Common wrong answer:** Registering in `seen` after recursing, so cycles never terminate.
</details>

### 100. `flatten`  `easy`
<details><summary>Solution</summary>

```js
Array.prototype.myFlat = function (depth = 1) {
  const out = [];
  const walk = (arr, d) => {
    for (let i = 0; i < arr.length; i++) {
      if (!(i in arr)) continue;                     // flat removes holes
      const x = arr[i];
      if (Array.isArray(x) && d > 0) walk(x, d - 1);
      else out.push(x);
    }
  };
  walk(this, depth);
  return out;
};
```
**Complexity:** O(n).
**Why this approach:** `flat` is specified to remove holes, unlike `map` which preserves them.
**Common wrong answer:** `Infinity` not working because `depth` was coerced with `|0`, which turns
`Infinity` into `0`.
</details>

### 101. `curry`  `medium`
See challenge 40.
<details><summary>Solution</summary>

```js
const curry = (fn, arity = fn.length) => {
  const collect = (args) => args.length >= arity
    ? fn(...args)
    : (...more) => collect([...args, ...more]);
  return collect([]);
};
```
**Complexity:** O(1) per application.
**Why this approach:** The inner `collect` avoids naming the returned function twice and keeps the
accumulated arguments in one place.
**Common wrong answer:** Relying on `fn.length` with defaults or rest parameters present.
</details>

### 102. `memoize`  `easy`
<details><summary>Solution</summary>

```js
const memoize = (fn, keyFn = (...a) => a[0]) => {
  const cache = new Map();
  return (...args) => {
    const key = keyFn(...args);
    if (!cache.has(key)) cache.set(key, fn(...args));
    return cache.get(key);
  };
};

// Object arguments: WeakMap so entries die with their key
const memoizeObject = (fn) => {
  const cache = new WeakMap();
  return (arg) => {
    if (!cache.has(arg)) cache.set(arg, fn(arg));
    return cache.get(arg);
  };
};
```
**Complexity:** O(1) per call.
**Why this approach:** `has`/`get` rather than checking for `undefined`, so a function legitimately
returning `undefined` is still cached.
**Common wrong answer:** `if (cache[key]) return cache[key]` — misses cached falsy results and uses
a polluted-prototype-vulnerable object.
</details>

### 103. `EventEmitter`  `medium`
<details><summary>Solution</summary>

```js
class EventEmitter {
  #listeners = new Map();

  on(event, handler) {
    if (!this.#listeners.has(event)) this.#listeners.set(event, new Set());
    this.#listeners.get(event).add(handler);
    return () => this.off(event, handler);
  }

  off(event, handler) {
    const set = this.#listeners.get(event);
    set?.delete(handler);
    if (set?.size === 0) this.#listeners.delete(event);      // don't accumulate empty sets
    return this;
  }

  once(event, handler) {
    const wrapped = (...args) => { this.off(event, wrapped); handler(...args); };
    return this.on(event, wrapped);
  }

  emit(event, ...args) {
    const set = this.#listeners.get(event);
    if (!set?.size) return false;
    for (const handler of [...set]) {                          // copy: `once` mutates during dispatch
      try { handler(...args); }
      catch (err) { queueMicrotask(() => { throw err; }); }    // one bad listener ≠ all listeners
    }
    return true;
  }

  listenerCount(event) { return this.#listeners.get(event)?.size ?? 0; }
}
```
**Complexity:** O(1) add/remove, O(k) emit.
**Why this approach:** Three details separate a real emitter from a toy: copy before dispatch,
isolate handler errors, and return the unsubscribe from `on`.
**Common wrong answer:** Iterating the live `Set`, so a `once` handler removing itself mid-dispatch
causes later handlers to be skipped.
</details>

### 104. `Object.assign`  `easy`
<details><summary>Solution</summary>

```js
Object.myAssign = function (target, ...sources) {
  if (target === null || target === undefined) throw new TypeError('Cannot convert to object');
  const to = Object(target);
  for (const source of sources) {
    if (source === null || source === undefined) continue;      // skipped, not an error
    for (const key of Reflect.ownKeys(source)) {
      const desc = Object.getOwnPropertyDescriptor(source, key);
      if (desc.enumerable) to[key] = source[key];               // assignment: triggers target setters
    }
  }
  return to;
};
```
**Complexity:** O(n) in total properties.
**Why this approach:** Plain assignment rather than `defineProperty` is what the spec requires, and
it's the observable difference from spread — `Object.assign` invokes setters on the target.
**Common wrong answer:** Using `Object.keys`, which drops symbol keys.
</details>

### 105. `JSON.stringify` (a useful subset)  `hard`
<details><summary>Solution</summary>

```js
function stringify(value, seen = new WeakSet()) {
  if (value === null) return 'null';
  if (typeof value === 'boolean' || typeof value === 'number') {
    return Number.isFinite(value) || typeof value === 'boolean' ? String(value) : 'null';
  }
  if (typeof value === 'string') return JSON.stringify(value);   // escaping is genuinely fiddly
  if (typeof value === 'bigint') throw new TypeError('BigInt cannot be serialized');
  if (typeof value === 'function' || typeof value === 'undefined' || typeof value === 'symbol') {
    return undefined;                                            // omitted in objects, null in arrays
  }

  if (typeof value.toJSON === 'function') return stringify(value.toJSON(), seen);

  if (seen.has(value)) throw new TypeError('Converting circular structure to JSON');
  seen.add(value);
  try {
    if (Array.isArray(value)) {
      return `[${value.map(v => stringify(v, seen) ?? 'null').join(',')}]`;
    }
    const parts = [];
    for (const [k, v] of Object.entries(value)) {
      const s = stringify(v, seen);
      if (s !== undefined) parts.push(`${JSON.stringify(k)}:${s}`);   // omit undefined values
    }
    return `{${parts.join(',')}}`;
  } finally {
    seen.delete(value);                        // sibling references aren't cycles
  }
}
```
**Complexity:** O(n) in nodes.
**Why this approach:** `seen.delete` in the `finally` is the subtle part — a `WeakSet` that never
releases would report two references to the same object as a cycle, which they aren't.
**Common wrong answer:** Treating `NaN` and `Infinity` as their string forms rather than `null`, and
forgetting that `undefined` is omitted in objects but becomes `null` in arrays.
</details>

---

## 9. DOM

### 106. Implement event delegation  `easy`
<details><summary>Solution</summary>

```js
function delegate(root, type, selector, handler, options) {
  const listener = (e) => {
    const match = e.target.closest(selector);
    if (match && root.contains(match)) handler.call(match, e, match);
  };
  root.addEventListener(type, listener, options);
  return () => root.removeEventListener(type, listener, options);
}
```
**Complexity:** O(depth) per event for the `closest` walk.
**Why this approach:** The `root.contains(match)` check matters — `closest` can escape the root and
match an ancestor outside it.
**Common wrong answer:** `e.target.matches(selector)`, which fails when the click lands on a child
of the matching element (an icon inside a button).
</details>

### 107. Serialise a form to an object  `easy`
Handle multi-value fields.
<details><summary>Solution</summary>

```js
function serializeForm(form) {
  const data = Object.create(null);
  for (const [key, value] of new FormData(form)) {
    if (key in data) data[key] = [].concat(data[key], value);   // checkbox groups, multi-selects
    else data[key] = value;
  }
  return data;
}
```
**Complexity:** O(n) in fields.
**Why this approach:** `FormData` respects `name`, `disabled`, and unchecked-checkbox semantics, so
you don't reimplement form rules.
**Common wrong answer:** `Object.fromEntries(new FormData(form))`, which silently keeps only the
last value for any repeated name — so a checkbox group returns one selection.
</details>

### 108. Build a DOM element from a spec object  `medium`
<details><summary>Solution</summary>

```js
const SAFE_PROTOCOLS = new Set(['http:', 'https:', 'mailto:']);

function el(tag, props = {}, ...children) {
  const node = document.createElement(tag);
  for (const [key, value] of Object.entries(props)) {
    if (key === 'class') node.className = value;
    else if (key === 'style' && typeof value === 'object') Object.assign(node.style, value);
    else if (key === 'dataset') Object.assign(node.dataset, value);
    else if (key.startsWith('on') && typeof value === 'function') {
      node.addEventListener(key.slice(2).toLowerCase(), value);
    } else if (key === 'href' || key === 'src') {
      // Vet URLs: an unchecked href accepts `javascript:`
      try {
        const url = new URL(value, location.origin);
        if (SAFE_PROTOCOLS.has(url.protocol)) node.setAttribute(key, url.href);
      } catch { /* unparseable: omit */ }
    } else node.setAttribute(key, value);
  }
  node.append(...children.flat(Infinity).filter(c => c != null));   // strings append as text nodes
  return node;
}

el('a', { href: '/x', class: 'link' }, 'Click me');
```
**Complexity:** O(props + children).
**Why this approach:** `append` with strings creates text nodes, so user content can never be
parsed as markup — the safety comes from the API choice, not from escaping.
**Common wrong answer:** `innerHTML` for children, which reintroduces XSS for every caller.
</details>

### 109. Detect when an element enters the viewport  `easy`
<details><summary>Solution</summary>

```js
const onVisible = (el, { rootMargin = '0px', threshold = 0 } = {}) =>
  new Promise((resolve) => {
    const io = new IntersectionObserver(([entry]) => {
      if (!entry.isIntersecting) return;
      io.disconnect();                              // one-shot: stop observing immediately
      resolve(entry);
    }, { rootMargin, threshold });
    io.observe(el);
  });
```
**Complexity:** O(1).
**Why this approach:** `IntersectionObserver` runs off the main thread's scroll path, so it costs
nothing when nothing changes — unlike a scroll handler calling `getBoundingClientRect`.
**Common wrong answer:** A throttled scroll listener with `getBoundingClientRect`, which forces
layout on every call.
</details>

### 110. Batch DOM reads and writes to avoid thrashing  `medium`
<details><summary>Solution</summary>

```js
const frame = (() => {
  const reads = [], writes = [];
  let scheduled = false;

  const flush = () => {
    scheduled = false;
    const r = reads.splice(0), w = writes.splice(0);
    const values = r.map(fn => fn());       // ALL reads first: one layout for the whole frame
    w.forEach((fn, i) => fn(values[i]));    // then all writes
  };

  const schedule = () => { if (!scheduled) { scheduled = true; requestAnimationFrame(flush); } };
  return {
    read: (fn) => { reads.push(fn); schedule(); },
    write: (fn) => { writes.push(fn); schedule(); },
  };
})();
```
**Complexity:** O(1) per registration.
**Why this approach:** Separating the phases means N components produce one layout recalculation
instead of N.
**Common wrong answer:** Interleaving in registration order, which is exactly the thrashing the
batcher was meant to prevent.
</details>

### 111. Implement a click-outside handler  `medium`
<details><summary>Solution</summary>

```js
function onClickOutside(element, callback, { signal } = {}) {
  const handler = (e) => {
    // composedPath crosses shadow boundaries; contains() alone misses those.
    if (!e.composedPath().includes(element)) callback(e);
  };
  // Register on the NEXT task: the click that opened this is still propagating.
  setTimeout(() => {
    document.addEventListener('pointerdown', handler, { signal });
  }, 0);
  return () => document.removeEventListener('pointerdown', handler);
}
```
**Complexity:** O(path depth) per click.
**Why this approach:** Two subtleties. Deferring registration stops the opening click from
immediately closing the popover, and `composedPath` handles shadow DOM where `contains` doesn't.
**Common wrong answer:** Registering synchronously on `click`, so the menu closes the instant it
opens — the single most common version of this bug.
</details>

---

## 10. Output Prediction

### 112. Coercion  `medium`
```js
console.log([] + {});
console.log({} + []);
console.log([] == false);
console.log([1] == 1);
console.log('5' - 3, '5' + 3);
```
<details><summary>Solution</summary>

`'[object Object]'`, `'[object Object]'`, `true`, `true`, `2 '53'`.

`[]` becomes `''` and `{}` becomes `'[object Object]'` via `ToPrimitive`. The famous
`{} + []` → `0` result only happens in a REPL where `{}` is parsed as an empty *block*, not an
object — inside `console.log` it's an expression, so it concatenates.
**Why this matters:** `+` is the only arithmetic operator that concatenates, and `==` runs the
whole abstract equality table. This is the case for `===` everywhere.
</details>

### 113. Loop closures  `easy`
```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log('var', i), 0);
for (let j = 0; j < 3; j++) setTimeout(() => console.log('let', j), 0);
```
<details><summary>Solution</summary>

`var 3` three times, then `let 0`, `let 1`, `let 2`.

`var` has one function-scoped binding all three closures share, and by the time the timers fire
the loop has finished. `let` in a `for` head creates a fresh binding per iteration.
**Why this matters:** It's the single most-asked closure question, and the fix is one keyword.
</details>

### 114. Event loop ordering  `medium`
```js
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
queueMicrotask(() => console.log('4'));
requestAnimationFrame(() => console.log('5'));
console.log('6');
```
<details><summary>Solution</summary>

`1 6 3 4` then `5` and `2` — with the last two racing. Synchronous code first, then the microtask
queue drains in order, then rendering and the timer. `5` before `2` if a paint was already pending;
otherwise `2` first.
**Why this matters:** The microtask queue drains completely between macrotasks, which is why `3` and
`4` beat `2` despite the timer being registered first.
</details>

### 115. Promise chain ordering  `hard`
```js
Promise.resolve(1)
  .then(v => { console.log('a', v); return 2; })
  .then(v => { console.log('b', v); return Promise.resolve(3); })
  .then(v => console.log('c', v));

Promise.resolve()
  .then(() => console.log('x'))
  .then(() => console.log('y'))
  .then(() => console.log('z'))
  .then(() => console.log('w'));
```
<details><summary>Solution</summary>

`a 1`, `x`, `b 2`, `y`, `z`, `c 3`, `w`.

Both chains interleave one tick at a time. The surprise is `c 3` arriving after `z`: returning a
promise from `.then` costs **two extra microtask ticks** for assimilation, not one — the spec
resolves it by calling `.then` on it, which itself queues.
**Why this matters:** It's the reason `return await p` inside a chain sometimes reorders logs, and
it's a favourite senior interview question.
</details>

### 116. Hoisting and the TDZ  `medium`
```js
console.log(typeof a, typeof b, typeof c);
var a = 1;
function b() {}
try { console.log(d); } catch (e) { console.log(e.constructor.name); }
let d = 2;
let c = 3;
```
<details><summary>Solution</summary>

`undefined function` — then it throws. `typeof c` hits the TDZ and raises `ReferenceError` before
anything prints, so the first `console.log` never completes.

If you remove `typeof c`, you get `undefined function` then `ReferenceError` from the `try`.
**Why this matters:** `typeof` is safe on *undeclared* identifiers but not on TDZ ones — the one
place `typeof` can throw.
</details>

### 117. `this` in four contexts  `medium`
```js
const obj = {
  name: 'obj',
  regular() { return this?.name; },
  arrow: () => globalThis.name,
};
const { regular } = obj;
console.log(obj.regular(), regular?.(), obj.arrow(), obj.regular.call({ name: 'x' }));
```
<details><summary>Solution</summary>

`'obj' undefined '' 'x'`.

Implicit binding gives `'obj'`. Detaching loses the receiver, so `this` is `undefined` in a module
and `this?.name` is `undefined`. The arrow never had its own `this`, and `globalThis.name` is `''`
in a browser. Explicit binding wins with `'x'`.
**Why this matters:** All four binding rules in one line, and it shows why passing a method as a
callback breaks it.
</details>

### 118. Array holes  `medium`
```js
const a = [1, , 3];
console.log(a.length, a[1]);
console.log(a.map(x => 2));
console.log([...a]);
console.log(a.filter(() => true).length);
console.log(Array(3).fill(0), Array(3).map(() => 0));
```
<details><summary>Solution</summary>

`3 undefined`, `[2, <hole>, 2]`, `[1, undefined, 3]`, `2`, `[0,0,0]` and `[<3 holes>]`.

`map` preserves holes and skips the callback for them; spread converts them to `undefined`;
`filter` compacts them away. `fill` writes holes, `map` doesn't.
**Why this matters:** This inconsistency is the reason to avoid sparse arrays entirely, and it's why
`Array(3).map(...)` silently does nothing.
</details>

### 119. Object key coercion and ordering  `medium`
```js
const o = {};
o[1] = 'a';
o['1'] = 'b';
o[true] = 'c';
o['02'] = 'd';
o.z = 'e';
o[2] = 'f';
console.log(Object.keys(o), o[1], Object.keys(o).length);
```
<details><summary>Solution</summary>

`['1', '2', 'true', '02', 'z'] 'b' 5`.

Every non-symbol key is coerced to a string, so `o[1]` and `o['1']` are the same property and `'b'`
wins. Enumeration puts integer-like keys first in ascending numeric order, then string keys in
insertion order — `'02'` is *not* integer-like (it has a leading zero), so it stays with the strings.
**Why this matters:** It's why a `Map` is required when you need genuine insertion order or non-string
keys.
</details>

### 120. `async` throw timing  `hard`
```js
async function f() { throw new Error('boom'); }
function g() { throw new Error('bang'); }

try { f(); console.log('after f'); } catch { console.log('caught f'); }
try { g(); console.log('after g'); } catch { console.log('caught g'); }
```
<details><summary>Solution</summary>

`after f`, then `caught g` — followed by an unhandled rejection for `'boom'`.

An `async` function never throws synchronously; it returns a rejected promise. The `try`/`catch`
sees nothing, `'after f'` prints, and the rejection goes unhandled because nobody awaited it. `g` is
an ordinary function, so its throw is caught.
**Why this matters:** It's the reason `f()` without `await` silently swallows errors, and why a bare
call to an async function should be flagged in review.
</details>

### 121. Getter side effects and spread  `hard`
```js
let reads = 0;
const source = {
  a: 1,
  get b() { reads++; return 2; },
};
const spread = { ...source };
const assigned = Object.assign({}, source);
const described = Object.getOwnPropertyDescriptor(spread, 'b');

console.log(reads, described.get, described.value, spread.b, reads);
```
<details><summary>Solution</summary>

`2 undefined 2 2 2`.

Both spread and `Object.assign` *invoke* the getter and copy the resulting value, so `reads` is 2
before the log. The copy has a plain data property — `get` is `undefined`, `value` is `2` — so
reading `spread.b` doesn't increment again.
**Why this matters:** Copying an object silently converts accessors into snapshots, so a computed
property stops recomputing. `Object.create(proto, Object.getOwnPropertyDescriptors(source))`
preserves them.
</details>

---

**Next:** [Appendix C — 20 Real-World Projects →](C-projects.md)
