# Chapter 26 — Regular Expressions

## Learning Objectives

- Read and write patterns using groups, quantifiers, anchors, and lookaround
- Choose the right method among `test`, `exec`, `match`, `matchAll`, and `replace`
- Recognise and fix catastrophic backtracking
- Decide when a regex is the wrong tool

---

## 26.1 Creating Patterns

```js
const literal = /\d+/g;                        // compiled once, at parse time
const dynamic = new RegExp(`^${escape(input)}$`, 'i');   // when the pattern is data
```

Use the literal form unless the pattern is built at runtime. The constructor takes a string, so
every backslash needs doubling — `new RegExp('\\d+')` for `/\d+/`. ES2025 adds `RegExp.escape`
for the constructor case:

```js
RegExp.escape('1+1=2');                        // '1\\+1\\=2'
new RegExp(RegExp.escape(userInput), 'gi');    // safe: input can't inject syntax
```

**Flags:**

| Flag | Name | Effect |
|---|---|---|
| `g` | global | Find all matches, not just the first. Enables `lastIndex`. |
| `i` | ignoreCase | Case-insensitive |
| `m` | multiline | `^` and `$` match at line breaks, not just string ends |
| `s` | dotAll | `.` matches newlines too |
| `u` | unicode | Treat the pattern as Unicode code points; enables `\p{…}` |
| `v` | unicodeSets | Superset of `u`: set operations and string properties in classes |
| `y` | sticky | Match only at exactly `lastIndex`; no scanning forward |
| `d` | hasIndices | Populate `match.indices` with start/end offsets per group |

```js
'a\nb'.match(/a.b/);        // null
'a\nb'.match(/a.b/s);       // ['a\nb']

const sticky = /\d+/y;
sticky.lastIndex = 3;
sticky.test('abc123');      // true  — matches exactly at index 3
sticky.lastIndex = 2;
sticky.test('abc123');      // false — 'c' is not a digit; no scanning ahead
```

---

## 26.2 The Building Blocks

**Character classes**

```js
/[abc]/       // one of a, b, c            /[^abc]/  // anything BUT a, b, c
/[a-z0-9]/    // ranges
/\d/ /\D/     // digit / non-digit         [0-9]
/\w/ /\W/     // word char / non-word      [A-Za-z0-9_]  — ASCII only!
/\s/ /\S/     // whitespace / non-         space, tab, newline, \u00a0, …
/./           // any char except newline (unless /s)
```

> ⚠️ **Gotcha** — `\w` and `\b` are ASCII-only. `/\w+/.exec('café')` gives `'caf'`, and
> `/\bcafé\b/` behaves unexpectedly because `é` is a non-word character. For real text use
> `/[\p{L}\p{N}_]+/u`.

**Quantifiers**, and greedy versus lazy:

```js
/a*/    // 0 or more      /a+/   // 1 or more     /a?/    // 0 or 1
/a{3}/  // exactly 3      /a{2,}/ // 2 or more    /a{2,4}/ // 2 to 4

const html = '<em>a</em><em>b</em>';
html.match(/<em>.*<\/em>/)[0];    // '<em>a</em><em>b</em>'  greedy: longest possible
html.match(/<em>.*?<\/em>/)[0];   // '<em>a</em>'            lazy: shortest possible
```

**Anchors and boundaries**

```js
/^abc$/         // whole string (or whole line with /m)
/\bcat\b/       // 'cat' as a word: matches 'a cat.' not 'concatenate'
/\Bcat/         // 'cat' NOT at a word boundary: matches 'concat'
```

**Groups**

```js
/(\d{4})-(\d{2})/.exec('2026-08');       // ['2026-08', '2026', '08']
/(?<y>\d{4})-(?<m>\d{2})/.exec('2026-08').groups;   // { y: '2026', m: '08' }
/(?:ab)+/                                 // non-capturing: grouped, not captured
/(\w)\1/.test('aa');                      // true — \1 backreferences group 1
/(?<c>\w)\k<c>/.test('aa');                // true — named backreference
/cat|dog/                                 // alternation
```

Prefer named groups. `match[3]` breaks silently the moment someone inserts a group.

**Lookaround** asserts without consuming:

```js
/foo(?=bar)/    // foo followed by bar        — positive lookahead
/foo(?!bar)/    // foo NOT followed by bar    — negative lookahead
/(?<=\$)\d+/    // digits preceded by $       — positive lookbehind
/(?<!\$)\d+/    // digits NOT preceded by $   — negative lookbehind

'$100 and 200'.match(/(?<=\$)\d+/g);    // ['100']

// Thousands separators, right to left, without touching the decimals:
'1234567.89'.replace(/\B(?=(\d{3})+(?!\d))/g, ',');   // '1,234,567.89'

// Password policy as one pattern (readable? no. correct? yes.)
const strong = /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^\w\s]).{12,}$/;
```

---

## 26.3 The Methods

| Method | Returns | Notes |
|---|---|---|
| `re.test(str)` | boolean | Cheapest. Beware `lastIndex` with `/g`. |
| `re.exec(str)` | match array or `null` | With `/g`, call repeatedly to iterate |
| `str.match(re)` | without `/g`: match array; with `/g`: array of strings or `null` | Two different shapes |
| `str.matchAll(re)` | iterator of match arrays | **Requires `/g`** |
| `str.search(re)` | index or `-1` | |
| `str.replace(re, x)` | new string | With `/g` replaces all |
| `str.replaceAll(re, x)` | new string | Regex arg **must** have `/g` |
| `str.split(re)` | array | Capture groups are included in the output |

`match` returning two different shapes depending on a flag is a genuine API wart:

```js
'a1b2'.match(/([a-z])(\d)/);    // ['a1', 'a', '1', index: 0, input: 'a1b2', groups: undefined]
'a1b2'.match(/([a-z])(\d)/g);   // ['a1', 'b2']  — groups and index GONE
[...'a1b2'.matchAll(/([a-z])(\d)/g)];   // full match arrays for every match  ✅
```

**The `lastIndex` trap** — the most common regex bug in JavaScript:

```js
const re = /\d+/g;                 // ❌ stateful, and reused
re.test('a1');    // true,  lastIndex now 2
re.test('a1');    // false, resumed from index 2
re.test('a1');    // true,  lastIndex reset to 0 after the failure

// ✅ Fixes: drop /g when you only need a boolean, or construct per use.
const isNumeric = (s) => /^\d+$/.test(s);
```

Never share a `/g` regex across calls, and never use one as a module-level constant with `test`.

**Function replacers** are where `replace` gets powerful:

```js
'2026-08-01'.replace(/(?<y>\d{4})-(?<m>\d{2})-(?<d>\d{2})/,
  (...args) => {
    const { y, m, d } = args.at(-1);        // named groups object is the last argument
    return `${d}/${m}/${y}`;
  });                                        // '01/08/2026'

// Signature: (match, p1, p2, …, offset, string, groups)
'a-b-c'.replace(/-(\w)/g, (_, ch) => ch.toUpperCase());   // 'aBC'
```

Replacement-string patterns: `$&` whole match, `$1` group 1, `` $` `` before, `$'` after,
`$<name>` named group, `$$` a literal `$`.

`String.raw` avoids double-escaping when building patterns:

```js
new RegExp(String.raw`\d+\.\d+`);      // clearer than '\\d+\\.\\d+'
```

---

## 26.4 Catastrophic Backtracking (ReDoS)

A regex engine that can't match tries alternatives. When a pattern has nested quantifiers over
overlapping input, the number of alternatives is exponential in the input length. That's a
denial-of-service vector if the input is user-controlled.

```js
// ❌ Exponential. (a+)+ can split a run of a's in 2^n ways.
const bad = /^(a+)+$/;
bad.test('a'.repeat(30) + 'b');       // hangs for minutes

// ❌ The classic "email validator" from Stack Overflow
const badEmail = /^([a-zA-Z0-9_.-])+@(([a-zA-Z0-9-])+\.)+([a-zA-Z]{2,4})+$/;
```

The three warning signs:

1. A quantifier applied to a group that itself contains a quantifier — `(a+)+`, `(\w*)*`
2. Alternation where branches can match the same text — `(a|ab)+`
3. `.*` on both sides of something optional

Fixes, in order of preference:

```js
// 1. Make the inner and outer quantifiers non-overlapping
/^a+$/                                 // same language, linear time

// 2. Anchor and be specific instead of using .*
/^\d{4}-\d{2}-\d{2}$/                  // not /^.*\d+.*$/

// 3. Use a negated class so there is only one way to match each character
/"([^"\\]|\\.)*"/                      // quoted string, linear: classes are disjoint

// 4. Bound the input before matching
if (input.length > 256) return false;
```

> 💼 **Interview** — "How would you validate an email address with a regex?" The senior answer is
> "I wouldn't." Use `<input type="email">` plus a server-side confirmation email. The full RFC
> 5322 grammar is not a regex worth maintaining, and every short version rejects valid addresses.

**Unicode property escapes** (with `u` or `v`) replace a lot of hand-rolled classes:

```js
/\p{L}/u          // any letter, any script          /\p{Lu}/u  // uppercase letter
/\p{N}/u          // any number                      /\p{Script=Greek}/u
/\p{Emoji_Presentation}/u
/[\p{L}\p{M}\p{N}_]+/u    // a real-world "word"

// /v adds set difference and intersection:
/[\p{L}--[a-z]]/v          // letters except ASCII lowercase
/[\p{L}&&\p{Script=Latin}]/v
```

---

## 26.5 When Not to Use a Regex

| Task | Use instead |
|---|---|
| Parsing HTML or XML | `DOMParser`, `document.createRange().createContextualFragment` |
| Parsing URLs | `new URL(str)` and `URLSearchParams` |
| Validating email | `<input type="email">` + a verification email |
| Parsing JSON | `JSON.parse` |
| Parsing dates | `Temporal.PlainDate.from` or a date library |
| Splitting on a fixed string | `str.split(',')` — no regex needed |
| Checking a prefix/suffix | `startsWith` / `endsWith` / `includes` |
| Sanitising HTML for output | A real sanitiser (Chapter 31); a regex will not do it |

HTML is not a regular language — arbitrary nesting can't be expressed in a regex, so any pattern
you write is wrong for some valid input. When the wrongness is a security boundary, it's a
vulnerability rather than a bug.

```js
// ❌ Breaks on <img alt="a > b">, comments, CDATA, attributes containing '<'
html.replace(/<[^>]*>/g, '');

// ✅
new DOMParser().parseFromString(html, 'text/html').body.textContent;
```

---

## 26.6 Common Mistakes and Best Practices

**Mistake: reusing a `/g` regex with `test` or `exec`.** `lastIndex` persists between calls, so
results alternate.

**Mistake: `str.match(re)` with `/g` when you need capture groups.** The global form returns bare
strings. Use `matchAll`.

**Mistake: interpolating user input into a pattern.** Any `(`, `*`, or `+` changes the grammar
and can hang the thread. Use `RegExp.escape`.

**Mistake: assuming `\w` and `\b` handle non-English text.** They're ASCII-only. Use `\p{L}` with
the `u` flag.

**Mistake: nested quantifiers.** `(a+)*`, `(\w*)+` — exponential backtracking waiting for a
crafted input.

**Mistake: forgetting `^` and `$` in a validator.** `/\d{4}/.test('abc1234xyz')` is `true`.

**Mistake: unescaped `.` in a domain or filename pattern.** `/example.com/` matches
`exampleXcom`, which is exactly the bypass an attacker wants.

**Best practice: name your groups.** `groups.year` survives refactoring; `match[2]` doesn't.

**Best practice: build long patterns from commented pieces.** JavaScript has no `/x` flag, so
compose with `String.raw` and a template, or accept that a 200-character literal is unreadable
and add a comment explaining what it matches with an example.

**Best practice: bound input length before matching user-supplied text.** It converts a possible
hang into a rejection.

---

## 26.7 Real-World Example: A Safe Log Parser

Log parsing is where regex genuinely earns its keep: the format is fixed, line-oriented, and
regular. This parser uses named groups, disjoint character classes so it runs in linear time, and
a sticky-flag tokeniser for the structured tail.

```js
/**
 * Parses combined-format access logs:
 * 127.0.0.1 - alice [01/Aug/2026:13:55:36 +0000] "GET /a?b=1 HTTP/1.1" 200 2326 key=val …
 */
const LINE = new RegExp(String.raw`
  ^(?<ip>\S+)\s+                       # client address, no whitespace
  (?<identity>\S+)\s+                  # RFC 1413 identity, usually '-'
  (?<user>\S+)\s+                      # authenticated user, or '-'
  \[(?<ts>[^\]]+)\]\s+                 # [^\]] not .*? — one way to match, so linear
  "(?<method>[A-Z]+)\s
   (?<path>[^"\s]*)\s
   (?<proto>HTTP/[\d.]+)"\s+
  (?<status>\d{3})\s+
  (?<bytes>\d+|-)
  (?<extra>.*)$                        # trailing key=value pairs, tokenised separately
`.replace(/\s+#.*$/gm, '').replace(/\s+/g, ''), '');   // strip comments AND whitespace

// A sticky tokeniser for `key=value` and `key="quoted value"` pairs.
const PAIR = /\s*(?<key>[\w.-]+)=(?:"(?<quoted>(?:[^"\\]|\\.)*)"|(?<bare>\S*))/y;

function parsePairs(input) {
  const out = {};
  PAIR.lastIndex = 0;                  // sticky state is ours to manage
  let match;
  while ((match = PAIR.exec(input))) {
    const { key, quoted, bare } = match.groups;
    out[key] = quoted !== undefined ? quoted.replace(/\\(.)/g, '$1') : bare;
    if (PAIR.lastIndex === match.index) break;   // zero-length match guard: never loop forever
  }
  return out;
}

export function parseLogLine(line) {
  if (line.length > 8192) return { ok: false, reason: 'line too long' };   // bound the input
  const m = LINE.exec(line);
  if (!m) return { ok: false, reason: 'unparseable', line };

  const g = m.groups;
  return {
    ok: true,
    ip: g.ip,
    user: g.user === '-' ? null : g.user,
    timestamp: g.ts,
    method: g.method,
    path: g.path,
    // The platform parses URLs correctly; a regex would not. Base is required for relative paths.
    query: Object.fromEntries(new URL(g.path, 'http://x').searchParams),
    status: Number(g.status),
    bytes: g.bytes === '-' ? 0 : Number(g.bytes),
    fields: parsePairs(g.extra),
  };
}

export function* parseLog(text) {
  for (const line of text.split('\n')) {
    if (line.trim()) yield parseLogLine(line);
  }
}
```

The pattern is written multi-line with `#` comments and then stripped, because JavaScript has no
verbose-regex flag and a 300-character single-line literal is unmaintainable. Note the cost: the
`.replace(/\s+/g, '')` means the pattern can contain no literal spaces, so `\s` is mandatory
everywhere whitespace is intended. That trade is worth it here and would not be for a shorter
pattern. The zero-length-match guard in `parsePairs` matters because a sticky regex whose
alternatives can all match empty will otherwise spin on the same index forever.

---

## 26.8 Questions

**1.** MCQ — Which flag makes `.` match newlines? a) `m` b) `s` c) `u` d) `g`
<details><summary>Answer</summary>**b)** `s`, for dotAll. `m` changes `^`/`$`, not `.`.</details>

**2.** MCQ — `'aaa'.match(/a+?/)[0]` is: a) `'aaa'` b) `'aa'` c) `'a'` d) `null`
<details><summary>Answer</summary>**c)** `+?` is lazy — the fewest characters that satisfy the pattern.</details>

**3.** MCQ — `matchAll` throws unless the regex has: a) `u` b) `y` c) `g` d) `d`
<details><summary>Answer</summary>**c)** A `TypeError` without `/g`.</details>

**4.** MCQ — `(?:x)` is: a) a named group b) non-capturing c) a lookahead d) optional
<details><summary>Answer</summary>**b)**</details>

**5.** MCQ — Which is at risk of catastrophic backtracking? a) `/^\d+$/` b) `/^(\d+)+$/` c) `/^[^,]+$/` d) `/^a{1,10}$/`
<details><summary>Answer</summary>**b)** A quantifier over a quantified group.</details>

**6.** MCQ — `'1.2'.match(/1.2/)` matches because: a) `.` is a literal dot b) `.` matches any char c) it doesn't match d) the string is coerced
<details><summary>Answer</summary>**b)** It also matches `'1x2'` — escape it as `/1\.2/` if you mean a dot.</details>

**7.** Short — What is the `lastIndex` trap?
<details><summary>Answer</summary>A regex with `/g` or `/y` stores where the last match ended, and `test`/`exec` resume from there. A reused global regex therefore returns alternating results on identical input.</details>

**8.** Short — When would you use the `y` flag?
<details><summary>Answer</summary>Tokenising. Sticky matching anchors each attempt at exactly `lastIndex` instead of scanning forward, which is what a lexer needs — a failure means "not this token here", not "look further along".</details>

**9.** Short — Why is `\w` a poor choice for matching words in international text?
<details><summary>Answer</summary>It's defined as `[A-Za-z0-9_]`, so accented letters, Cyrillic, and CJK all fall outside it. `[\p{L}\p{N}_]` with the `u` flag is the correct equivalent.</details>

**10.** Short — Difference between lookahead and a capture group?
<details><summary>Answer</summary>A lookahead asserts that something follows without consuming it, so it doesn't appear in the match and the position doesn't advance. A capture group consumes its text and stores it.</details>

**11.** Short — Why does `String.raw` help when writing patterns?
<details><summary>Answer</summary>It disables escape processing in the template, so `String.raw`\d`` is the two characters `\d` rather than requiring `'\\d'` — one fewer layer of escaping to reason about.</details>

**12.** Output —
```js
const re = /\d/g;
console.log(re.test('a1'), re.test('a1'), re.test('a1'));
```
<details><summary>Answer</summary>`true false true` — `lastIndex` advances to 2, the second call fails and resets it to 0, the third succeeds.</details>

**13.** Output —
```js
console.log('a1b2'.match(/([a-z])(\d)/g));
console.log([...'a1b2'.matchAll(/([a-z])(\d)/g)].map(m => m[1]));
```
<details><summary>Answer</summary>`['a1', 'b2']` then `['a', 'b']` — `match` with `/g` discards groups; `matchAll` keeps them.</details>

**14.** Output —
```js
console.log('a-b_c'.split(/([-_])/));
```
<details><summary>Answer</summary>`['a', '-', 'b', '_', 'c']` — capture groups are interleaved into `split`'s output.</details>

**15.** Output —
```js
console.log('price: $30'.replace(/(\d+)/, '$$$1.00'));
```
<details><summary>Answer</summary>`price: $$30.00` — `$$` is a literal `$` and `$1` is the group, so the original `$` plus the inserted one both appear.</details>

**16.** Output —
```js
console.log('café'.match(/\w+/)[0]);
console.log('café'.match(/[\p{L}]+/u)[0]);
```
<details><summary>Answer</summary>`caf` then `café`.</details>

**17.** Debug — A "valid username" check passes for `hello world!`.
<details><summary>Answer</summary>The pattern lacks anchors, so it matched the `hello` substring. Add `^` and `$` — `/^\w{3,16}$/`.</details>

**18.** Debug — A search filter works the first time a user types and then intermittently returns no results.
<details><summary>Answer</summary>A module-level `/g` regex is being reused with `test`, so `lastIndex` carries over between keystrokes. Build the regex inside the function, or drop `/g` since only a boolean is needed.</details>

**19.** Debug — A request with a 40-character query string pegs a server CPU at 100% for two minutes.
<details><summary>Answer</summary>Catastrophic backtracking in a validation regex with nested quantifiers. Rewrite the pattern with disjoint character classes so each character has exactly one way to match, and reject over-long inputs before matching.</details>

**20.** Debug — A tag stripper turns `<p title="a > b">hi</p>` into `b">hi`.
<details><summary>Answer</summary>`/<[^>]*>/g` stopped at the `>` inside the attribute value. Regex can't parse HTML; use `DOMParser` and read `textContent`.</details>

**21.** Debug — `new RegExp(searchTerm)` throws `SyntaxError: Invalid regular expression` for some searches.
<details><summary>Answer</summary>The user typed a regex metacharacter such as `(` or `*`. Escape the input with `RegExp.escape(searchTerm)` (or a `replace(/[.*+?^${}()|[\]\\]/g, '\\$&')` fallback) before constructing.</details>

**22.** Debug — A `while ((m = re.exec(s)))` loop never terminates.
<details><summary>Answer</summary>The pattern can match an empty string, so `lastIndex` never advances. Ensure at least one character is consumed, or manually bump `lastIndex` when `m[0] === ''`.</details>

**23.** Code — Write `escapeRegExp(str)` without `RegExp.escape`.
<details><summary>Answer</summary>

```js
const escapeRegExp = (str) => str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
escapeRegExp('1+1 (a)');   // '1\\+1 \\(a\\)'
```
</details>

**24.** Code — Convert `kebab-case` to `camelCase`, and back.
<details><summary>Answer</summary>

```js
const toCamel  = (s) => s.replace(/-([a-z])/g, (_, c) => c.toUpperCase());
const toKebab  = (s) => s.replace(/[A-Z]/g, (c) => `-${c.toLowerCase()}`);
toCamel('background-color');   // 'backgroundColor'
toKebab('backgroundColor');    // 'background-color'
```
</details>

**25.** Code — Highlight every occurrence of a user's search term in a string, safely.
<details><summary>Answer</summary>

```js
function highlight(text, term) {
  if (!term) return escapeHtml(text);
  const re = new RegExp(RegExp.escape(term), 'gi');
  // Escape FIRST, then wrap — otherwise the <mark> tags get escaped too.
  return escapeHtml(text).replace(re, (m) => `<mark>${m}</mark>`);
}
const escapeHtml = (s) => s.replace(/[&<>"']/g, (c) =>
  ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]));
```
</details>

**26.** Code — Extract all `key=value` pairs from a string into an object, allowing quoted values.
<details><summary>Answer</summary>

```js
function parsePairs(str) {
  const re = /(?<key>[\w.-]+)=(?:"(?<q>[^"\\]*(?:\\.[^"\\]*)*)"|(?<b>[^\s]*))/g;
  return Object.fromEntries(
    [...str.matchAll(re)].map(({ groups: g }) => [g.key, g.q ?? g.b])
  );
}
parsePairs('a=1 b="two words" c=');   // { a: '1', b: 'two words', c: '' }
```
</details>

**27.** Scenario — A colleague proposes a 400-character regex to validate email addresses on signup. What do you argue for?
<details><summary>Answer</summary>Validate only that there's an `@` with something either side, then send a confirmation email — the email is the actual proof of validity, and nothing else is. The long pattern will reject valid addresses (new TLDs, `+` tags, quoted local parts, IDN) and generate support tickets, while still accepting addresses that don't exist. If the goal is catching typos, offer a domain suggestion instead. Keep `<input type="email">` for the free browser-level check and the correct mobile keyboard.</details>

**28.** Scenario — User-supplied regex filters are a product feature. How do you ship that without a DoS?
<details><summary>Answer</summary>Don't run untrusted patterns on the main thread. Execute them in a Web Worker (or a separate process server-side) with a hard timeout that terminates the worker, since you can't interrupt a running regex any other way. Cap both pattern and input length, and reject patterns that fail a static check for nested quantifiers. If the feature can tolerate it, a linear-time engine such as RE2 removes the class of problem entirely rather than mitigating it.</details>

---

## Chapter Summary

- Prefer literals; use `new RegExp` with `RegExp.escape` only when the pattern is data.
- `/g` makes a regex stateful via `lastIndex` — never share one across `test`/`exec` calls.
- `match` with `/g` throws away capture groups; `matchAll` is what you want.
- Named groups and function replacers make non-trivial patterns maintainable.
- Nested quantifiers cause exponential backtracking; disjoint character classes keep it linear.
- HTML, URLs, and email are not regular languages — use `DOMParser`, `URL`, and a confirmation email.

---

## Practice Exercises

1. Write patterns for a hex colour, an ISO 8601 date, and a semver version, anchored and tested against ten valid and ten invalid inputs each.
2. Time `/^(a+)+$/` against `'a'.repeat(n) + 'b'` for n = 20…30 and plot the growth.
3. Rewrite `/"(.*?)"/` to be linear-time while still handling escaped quotes.
4. Build a syntax highlighter for a small language using one sticky regex per token type.
5. Take a regex-based HTML tag stripper and construct three inputs that defeat it.

---

## Mini Project: A Markdown Subset Converter

Convert a Markdown subset to HTML with regex, and discover where regex stops being enough.

- Headings, bold, italic, inline code, links, unordered lists, blockquotes
- Fenced code blocks whose contents are **not** processed for other syntax
- HTML-escape all text content before inserting it, so the output is XSS-safe
- Handle nested emphasis (`**bold with _italic_**`) correctly, or document precisely why you can't
- Every pattern must be linear-time; state the reasoning for each
- Write a short note on which features forced you toward a real parser and why

---

**Next:** [Chapter 27 — Memory and Garbage Collection →](27-memory-garbage-collection.md)
