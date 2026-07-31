# Learning Topics

A collection of self-contained, book-length learning guides. Each topic lives in its own
folder, follows the same structure, and can be read start-to-finish or dipped into as a
reference.

These are written to be **used by other people**, not just their author. That means every
topic is expected to be navigable without explanation, honest about trade-offs, and full of
code you can actually run.

---

## Topics

| Topic | Status | Size | Contents |
|-------|--------|------|----------|
| [JavaScript](javascript/) | Complete | 33 chapters + 7 appendices | Fundamentals → engine internals → async → the browser → security, testing, performance, patterns. Plus 1,170 questions with answers, 121 coding challenges with worked solutions, 20 project specs, a 199-item best-practices checklist, a cheat sheet, and a 204-term glossary. |

**Suggested entry points for the JavaScript book:** start at
[Chapter 1](javascript/chapters/01-how-javascript-works.md) for a linear read, the
[roadmap](javascript/appendices/D-roadmap.md) for a staged plan with checkpoints, or the
[interview questions](javascript/appendices/A-interview-questions.md) to find out what you don't
know yet.

---

## How Each Topic Is Organised

```
<topic>/
├── README.md          Table of contents, who it's for, conventions
├── chapters/          NN-slug.md, read in order
└── appendices/        X-slug.md, reference material
```

Every chapter follows the same shape, so once you've read one you know where to find things
in all of them:

1. **Learning objectives** — what you'll be able to do afterwards
2. **Numbered sections** — explanation, then heavily commented runnable code
3. **Common mistakes and best practices** — named traps with one-line diagnoses
4. **Real-world example** — a production-shaped module, not a toy
5. **Questions** — 20–28 of them, with collapsible answers
6. **Chapter summary** — takeaways, not topic names
7. **Practice exercises** — no solutions, on purpose
8. **Mini project** — where the topic supports one

## Conventions Used Throughout

Output appears as a trailing comment:

```js
const doubled = [1, 2, 3].map(n => n * 2);
console.log(doubled); // [2, 4, 6]
```

Wrong and right examples are marked:

```js
if (x = 5) { }   // ❌ assignment, always truthy
if (x === 5) { } // ✅
```

Callouts flag the two things worth interrupting prose for:

> ⚠️ **Gotcha** — a specific trap, stated concretely.

> 💼 **Interview** — how this actually gets asked.

Answers are collapsed so you can self-test:

**1.** MCQ — Which is falsy? a) `'0'` b) `[]` c) `0` d) `{}`
<details><summary>Answer</summary>**c)** Empty string, <code>0</code>, <code>NaN</code>, <code>null</code>, <code>undefined</code>, <code>false</code>, and <code>0n</code> are the only falsy values.</details>

---

## How to Use This Repository

**Reading one topic front to back** is the default. Chapters build on each other and
cross-reference by number.

**Interview prep** — each topic's README lists a shorter chapter path, and the interview
appendix is the fastest route to knowing what you don't know.

**Reference** — the cheat sheet and glossary appendices are meant to be kept open in a tab.

Nothing here needs to be built or installed. Clone it, or read it on GitHub, or open the
folder in any Markdown editor.

## Licence

Content is free to read, share, and adapt with attribution. Code samples are public domain — copy them into your own work without asking.
