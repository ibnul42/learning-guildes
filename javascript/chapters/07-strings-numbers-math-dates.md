# Chapter 7 — Strings, Numbers, Math, and Dates

## Learning Objectives

- Manipulate strings with modern methods and handle Unicode correctly
- Explain floating-point imprecision and work around it
- Use `BigInt`, `Number` statics, and `Intl` formatting
- Work with `Date` safely — and know why `Temporal` replaces it

---

## 7.1 Strings

```js
const s = 'JavaScript';

s.length;                  // 10
s.at(-1);                  // 't'
s.slice(4);                // 'Script'   — negatives allowed
s.substring(0, 4);         // 'Java'     — negatives clamp to 0
s.toUpperCase();
s.trim(); s.trimStart(); s.trimEnd();
s.padStart(12, '*');       // '**JavaScript'
s.repeat(2);
s.includes('Script');      // true
s.startsWith('Java');
s.indexOf('a');            // 1
s.replace('a', 'X');       // first only
s.replaceAll('a', 'X');    // all (ES2021)
s.split('');               // ⚠️ breaks emoji
[...s];                    // ✅ code-point aware
s.localeCompare('java');   // locale-correct ordering
s.normalize('NFC');        // canonical Unicode form
```

**Strings are immutable.** Every method returns a new string.

### Template literals

```js
const name = 'Ada';
`Hello ${name}, ${1 + 1} items`;   // interpolation + expressions
`line1
line2`;                             // real newlines

// Tagged template
function html(strings, ...values) {
  return strings.reduce((out, str, i) =>
    out + str + (values[i] != null ? escapeHtml(String(values[i])) : ''), '');
}
const safe = html`<p>${userInput}</p>`;   // auto-escaping — see Chapter 31
```

### Unicode

```js
'😀'.length;          // 2 — a surrogate pair, not one "character"
[...'😀'].length;     // 1
'é'.normalize('NFC').length;  // 1, vs 2 for the decomposed form

// Correct grapheme counting (emoji with modifiers, flags)
const seg = new Intl.Segmenter('en', { granularity: 'grapheme' });
[...seg.segment('👨‍👩‍👧')].length;   // 1
```

> ⚠️ Never use `.length` to validate user-visible text length. Use `Intl.Segmenter`.

---

## 7.2 Numbers

All JavaScript numbers are IEEE-754 doubles.

```js
0.1 + 0.2;                      // 0.30000000000000004
0.1 + 0.2 === 0.3;              // false
Math.abs(0.1 + 0.2 - 0.3) < Number.EPSILON;   // true ✅

Number.MAX_SAFE_INTEGER;        // 9007199254740991 (2^53 - 1)
9007199254740992 === 9007199254740993;  // true (!) — beyond safe range

Number.isInteger(5.0);          // true
Number.isSafeInteger(2**53);    // false
Number.isFinite('5');           // false (no coercion) — vs global isFinite('5') → true

(1234.5678).toFixed(2);         // '1234.57'  (a STRING)
(0.000001234).toExponential(2); // '1.23e-6'
(255).toString(16);             // 'ff'
parseInt('ff', 16);             // 255
Number('  42  ');               // 42
Number('');                     // 0   ⚠️
Number('abc');                  // NaN

1_000_000;                      // numeric separators
```

### Money: never use floats

```js
// ❌
0.1 + 0.2;                 // 0.30000000000000004

// ✅ integer cents
const cents = 10 + 20;     // 30
const display = (cents / 100).toFixed(2);

// ✅ BigInt for very large exact integers
const big = 9007199254740993n + 1n;   // 9007199254740994n
typeof big;                            // 'bigint'
// BigInt and Number cannot be mixed in arithmetic:
// 1n + 1  →  TypeError
```

### `toFixed` rounding surprise

```js
(1.005).toFixed(2);   // '1.00' — 1.005 is actually 1.00499999...
// Safer:
const round = (n, d = 2) => Math.round((n + Number.EPSILON) * 10**d) / 10**d;
```

---

## 7.3 Math

```js
Math.round(2.5);    // 3
Math.round(-2.5);   // -2   ⚠️ rounds toward +Infinity
Math.trunc(-2.7);   // -2   (chops)
Math.floor(-2.7);   // -3
Math.ceil(-2.7);    // -2
Math.sign(-5);      // -1
Math.hypot(3, 4);   // 5
Math.max(...arr);   // ⚠️ stack overflow on very large arrays
Math.min();         // Infinity  (and Math.max() is -Infinity)
Math.random();      // [0, 1)

const randInt = (min, max) => Math.floor(Math.random() * (max - min + 1)) + min;

// Cryptographically secure — use this for tokens, never Math.random
crypto.getRandomValues(new Uint32Array(1))[0];
crypto.randomUUID();
```

---

## 7.4 Dates

```js
new Date();                          // now
new Date('2025-07-28T10:00:00Z');    // ISO — UTC, reliable
new Date(2025, 6, 28);               // ⚠️ month is 0-indexed! July
new Date(1753696800000);             // epoch ms
Date.now();                          // epoch ms, cheap

const d = new Date();
d.getFullYear(); d.getMonth(); d.getDate();  // local
d.getUTCHours();
d.toISOString();                     // '2025-07-28T10:00:00.000Z'
d.getTime();
```

**`Date`'s known problems:** mutable, 0-indexed months, no timezone support beyond UTC/local, and `new Date('2025-07-28')` is parsed as **UTC** while `new Date('2025/07/28')` is parsed as **local** — a classic off-by-one-day bug.

### Formatting: use `Intl`

```js
new Intl.DateTimeFormat('en-GB', {
  dateStyle: 'medium', timeStyle: 'short', timeZone: 'Asia/Dhaka',
}).format(d);   // '28 Jul 2025, 16:00'

new Intl.RelativeTimeFormat('en', { numeric: 'auto' }).format(-1, 'day');  // 'yesterday'

new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' })
  .format(1234.5);   // '$1,234.50'

new Intl.ListFormat('en').format(['a','b','c']);   // 'a, b, and c'
new Intl.PluralRules('en').select(1);              // 'one'
```

### `Temporal` (Stage 3, shipping now)

```js
Temporal.Now.plainDateISO();                    // 2025-07-28
Temporal.PlainDate.from('2025-07-28').add({ months: 1 });  // 2025-08-28
Temporal.ZonedDateTime.from('2025-07-28T10:00[Asia/Dhaka]');
Temporal.Duration.from({ hours: 3 });
```

Immutable, explicit about timezones, sane month numbering. When it's available in your targets, prefer it. Until then, `date-fns` or `Luxon`.

---

## 7.5 Common Mistakes and Best Practices

**Mistake: floats for currency.** Store integer minor units.

**Mistake: `new Date(y, m, d)` with a 1-indexed month.** Off by one, every time.

**Mistake: hand-rolling date formatting.** `Intl` handles locales, timezones, and edge cases you haven't thought of.

**Mistake: `Math.random()` for tokens, IDs, or shuffling anything that matters.** Use `crypto`.

**Mistake: `parseInt` on a float string.** `parseInt('3.9')` is `3`, silently.

**Best practice: store UTC, display local.** Persist ISO 8601 strings or epoch millis; convert at the presentation layer only.

**Best practice: validate numeric input explicitly.** `Number('')` is `0`, `Number(' ')` is `0`, `Number([])` is `0`. Test with `Number.isFinite(Number(x)) && x.trim() !== ''`.

---

## 7.6 Real-World Example: An Invoice Formatter

```js
/**
 * Money as integer cents throughout; formatting only at the boundary.
 */
class Money {
  #cents;
  constructor(cents) {
    if (!Number.isSafeInteger(cents)) throw new RangeError('cents must be a safe integer');
    this.#cents = cents;
  }
  static fromDecimal(str) {
    const n = Number(str);
    if (!Number.isFinite(n)) throw new TypeError(`Invalid amount: ${str}`);
    return new Money(Math.round(n * 100));
  }
  plus(other)  { return new Money(this.#cents + other.#cents); }
  times(qty)   { return new Money(Math.round(this.#cents * qty)); }
  get cents()  { return this.#cents; }
  format(locale = 'en-US', currency = 'USD') {
    return new Intl.NumberFormat(locale, { style: 'currency', currency })
      .format(this.#cents / 100);
  }
}

function renderInvoice(lines, { locale = 'en-US', currency = 'USD', tz = 'UTC' } = {}) {
  const subtotal = lines.reduce(
    (t, l) => t.plus(Money.fromDecimal(l.unitPrice).times(l.qty)),
    new Money(0)
  );
  const tax = subtotal.times(0.1);
  const total = subtotal.plus(tax);

  return {
    issuedAt: new Intl.DateTimeFormat(locale, { dateStyle: 'long', timeZone: tz })
      .format(new Date()),
    subtotal: subtotal.format(locale, currency),
    tax: tax.format(locale, currency),
    total: total.format(locale, currency),
  };
}
```

Every arithmetic operation is integer arithmetic. Formatting happens once, at the edge.

---

## 7.7 Questions

**1.** MCQ — `0.1 + 0.2 === 0.3` is: a) true b) false
<details><summary>Answer</summary>**b)** IEEE-754.</details>

**2.** MCQ — `'😀'.length` is: a) 1 b) 2 c) 4 d) error
<details><summary>Answer</summary>**b)**</details>

**3.** MCQ — `new Date(2025, 1, 1)` is: a) 1 Jan b) 1 Feb c) invalid d) 2 Jan
<details><summary>Answer</summary>**b)** Months are 0-indexed.</details>

**4.** MCQ — `(1234.5).toFixed(0)` returns type: a) number b) string
<details><summary>Answer</summary>**b)**</details>

**5.** MCQ — `Math.max()` returns: a) `0` b) `undefined` c) `-Infinity` d) `NaN`
<details><summary>Answer</summary>**c)**</details>

**6.** MCQ — `1n + 1` : a) `2n` b) `2` c) TypeError d) `'11'`
<details><summary>Answer</summary>**c)** BigInt and Number can't mix.</details>

**7.** MCQ — `Number('')` is: a) `NaN` b) `0` c) `undefined` d) `''`
<details><summary>Answer</summary>**b)**</details>

**8.** Short — Why is `0.1 + 0.2` imprecise?
<details><summary>Answer</summary>0.1 and 0.2 have no exact binary representation in IEEE-754 doubles, so tiny rounding errors accumulate.</details>

**9.** Short — How should you store money?
<details><summary>Answer</summary>As integers in the smallest currency unit (cents), or as a decimal string / BigInt. Never floats.</details>

**10.** Short — `Number.isNaN` vs global `isNaN`?
<details><summary>Answer</summary>Global `isNaN` coerces first, so `isNaN('abc')` is `true`. `Number.isNaN` only returns true for the actual `NaN` value.</details>

**11.** Short — Why prefer `Intl.Segmenter` over `.split('')`?
<details><summary>Answer</summary>`split('')` splits UTF-16 code units, mangling emoji, flags, and combined graphemes. `Segmenter` yields user-perceived characters.</details>

**12.** Short — What problem does `Temporal` solve?
<details><summary>Answer</summary>`Date` is mutable, has 0-indexed months, ambiguous string parsing, and no first-class timezone or duration support. `Temporal` is immutable and explicit about all of these.</details>

**13.** Output — `console.log((1.005).toFixed(2));`
<details><summary>Answer</summary>`'1.00'` — 1.005 stored as slightly less than 1.005.</details>

**14.** Output — `console.log(Math.round(-2.5), Math.round(2.5));`
<details><summary>Answer</summary>`-2 3` — rounds half toward +Infinity.</details>

**15.** Output — `console.log(new Date('2025-07-28').getDate());`
<details><summary>Answer</summary>Depends on timezone. The string is parsed as UTC midnight; west of UTC that's the 27th locally. This is the classic off-by-one-day bug.</details>

**16.** Output — `console.log(typeof NaN, 9007199254740992 === 9007199254740993);`
<details><summary>Answer</summary>`'number' true` — beyond `MAX_SAFE_INTEGER`, adjacent integers collide.</details>

**17.** Output — `console.log('abc'.padStart(2, '*'), 'a'.padStart(4, 'xy'));`
<details><summary>Answer</summary>`'abc' 'xyxa'` — no truncation if already long enough; pad string is repeated and truncated.</details>

**18.** Debug — A totals column shows `19.999999999999998`. Fix?
<details><summary>Answer</summary>Accumulating floats. Convert to integer cents at input, sum as integers, divide by 100 only for display.</details>

**19.** Debug — Users in Sydney see the wrong date on a "created" field.
<details><summary>Answer</summary>You're formatting a UTC timestamp with local methods or without a `timeZone`. Store UTC, format with `Intl.DateTimeFormat(locale, { timeZone })` using the user's zone from `Intl.DateTimeFormat().resolvedOptions().timeZone`.</details>

**20.** Debug — A "random" invite code collides frequently:
```js
const code = Math.random().toString(36).slice(2, 8);
```
<details><summary>Answer</summary>Only ~36^6 space, and `Math.random` isn't cryptographically secure or collision-resistant. Use `crypto.randomUUID()` or `crypto.getRandomValues`.</details>

**21.** Code — Write `truncate(str, max)` adding an ellipsis, emoji-safe.
<details><summary>Answer</summary>

```js
function truncate(str, max) {
  const chars = [...new Intl.Segmenter().segment(str)].map(s => s.segment);
  return chars.length <= max ? str : chars.slice(0, max - 1).join('') + '…';
}
```
</details>

**22.** Code — Write `formatBytes(n)` → `'1.5 MB'`.
<details><summary>Answer</summary>

```js
function formatBytes(n, decimals = 1) {
  if (n === 0) return '0 B';
  const units = ['B','KB','MB','GB','TB'];
  const i = Math.min(Math.floor(Math.log(n) / Math.log(1024)), units.length - 1);
  return `${(n / 1024 ** i).toFixed(decimals)} ${units[i]}`;
}
```
</details>

**23.** Code — Write `daysBetween(a, b)` ignoring time-of-day and DST.
<details><summary>Answer</summary>

```js
const daysBetween = (a, b) => {
  const utc = (d) => Date.UTC(d.getFullYear(), d.getMonth(), d.getDate());
  return Math.round((utc(b) - utc(a)) / 86_400_000);
};
```
Normalizing to UTC midnight avoids DST hour shifts.
</details>

**24.** Code — Write `slugify(title)`.
<details><summary>Answer</summary>

```js
const slugify = (s) => s
  .normalize('NFD').replace(/[\u0300-\u036f]/g, '')   // strip accents
  .toLowerCase().trim()
  .replace(/[^a-z0-9]+/g, '-')
  .replace(/^-+|-+$/g, '');
```
</details>

**25.** Scenario — Finance asks why the ledger is off by $0.03 across 40,000 transactions. Diagnose and prescribe.
<details><summary>Answer</summary>Float accumulation error. Migrate to integer minor units end to end — database column, API contract, and client. Add a reconciliation test that sums 100k random amounts both ways and asserts exact equality.</details>

**26.** Scenario — A scheduling feature works everywhere except during DST transitions. What's the structural fix?
<details><summary>Answer</summary>Stop doing arithmetic on epoch milliseconds for calendar concepts. Store the wall-clock time plus an IANA timezone id, and compute with `Temporal.ZonedDateTime` (or Luxon), which handles the gap/overlap hours explicitly.</details>

---

## Chapter Summary

- Strings are immutable and UTF-16; use spread or `Intl.Segmenter` for character-accurate work.
- All numbers are doubles. `Number.EPSILON` for comparisons, integer cents for money, `BigInt` for exact large integers.
- `Number.isNaN`/`isFinite` don't coerce; the globals do.
- `Math.round` rounds half up toward +Infinity; `Math.random` is not secure.
- `Date` is mutable with 0-indexed months and ambiguous parsing. Format with `Intl`; migrate to `Temporal`.
- Store UTC, display local.

---

## Practice Exercises

1. Implement `roundHalfEven(n, decimals)` (banker's rounding).
2. Write a `parseMoney(str)` accepting `'$1,234.56'`, `'1234.56'`, and `'(1,234.56)'` (negative) → cents.
3. Write `wordWrap(text, width)` that never splits a grapheme.
4. Build `humanizeDuration(ms)` → `'2 days, 3 hours'` using `Intl.ListFormat`.
5. Compare `new Date('2025-07-28')` and `new Date('2025/07/28')` in three timezones and explain.

---

## Mini Project: A Locale-Aware Formatting Library

Build `format.js` exporting `money`, `date`, `relative`, `list`, `plural`, `bytes`, and `duration`.

**Requirements:**

- Every function takes an optional `{ locale, timeZone }`, defaulting to the runtime's resolved values.
- `money` accepts integer minor units and a currency code; it must never receive a float.
- Cache `Intl.*Format` instances in a `Map` — constructing them is expensive and this is a measurable win.
- `relative` picks the largest sensible unit automatically (`'3 months ago'`, not `'92 days ago'`).
- Include a test suite covering `en-US`, `de-DE`, `ar-EG`, and `bn-BD`.

**Stretch:** add a `Temporal`-based implementation behind a feature detect, falling back to `Date`.

---

**Next:** [Chapter 8 — Scope, Hoisting, and the TDZ →](08-scope-hoisting-tdz.md)
