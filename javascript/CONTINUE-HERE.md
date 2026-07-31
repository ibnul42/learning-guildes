# Maintainer Notes

This book is **complete**: 33 chapters and 7 appendices. This file used to be a continuation brief
for finishing it; it now records the conventions it was written to, so edits and additions stay
consistent.

For repository-wide conventions, see [../CONTRIBUTING.md](../CONTRIBUTING.md).

---

## 1. Current Status

**Complete — 33 chapters + 7 appendices, ~150,000 words.**

### Chapters (`chapters/`)

| # | File | Topic |
|---|---|---|
| 1 | `01-how-javascript-works.md` | Engine vs runtime, TC39, parse→execute, JIT, strict mode |
| 2 | `02-variables-types-operators.md` | var/let/const, 7 primitives, falsy values, coercion, `==` |
| 3 | `03-control-flow-loops.md` | Conditionals, switch, all loop forms, labels |
| 4 | `04-functions.md` | Declarations/expressions/arrows/IIFE, params, currying, recursion |
| 5 | `05-objects.md` | Key order, descriptors, copying, freeze, Proxy/Reflect |
| 6 | `06-arrays.md` | Mutating vs non-mutating (incl. ES2023), reduce, sort traps |
| 7 | `07-strings-numbers-math-dates.md` | Unicode, IEEE-754, BigInt, Intl, Temporal |
| 8 | `08-scope-hoisting-tdz.md` | Scope chain, hoisting table, TDZ |
| 9 | `09-closures.md` | Capture semantics, 5 uses, closure leaks |
| 10 | `10-this-and-binding.md` | 5 binding rules, lost `this`, call/apply/bind |
| 11 | `11-prototypes-inheritance.md` | Chain, `new`, pre-class inheritance, pollution |
| 12 | `12-classes-oop.md` | Fields, `#private`, statics, `super`, mixins |
| 13 | `13-destructuring-spread-rest.md` | All forms, `?.`, `??`, logical assignment |
| 14 | `14-symbols-iterators-generators.md` | Protocols, generators, async generators |
| 15 | `15-maps-sets-weak-collections.md` | Map vs Object, ES2025 set ops, LRU cache |
| 16 | `16-error-handling.md` | try/catch/finally, `cause`, custom hierarchies |
| 17 | `17-modules.md` | ESM, live bindings, CJS diff, dynamic import, cycles |
| 18 | `18-event-loop.md` | Call stack, micro/macrotasks, Node phases |
| 19 | `19-async-javascript.md` | Promises, async/await, combinators, concurrency pool |
| 20 | `20-fetch-json-http.md` | Fetch semantics, JSON replacer/reviver, CORS, uploads |
| 21 | `21-dom.md` | Selection, reflow/repaint, thrashing, virtualization |
| 22 | `22-events.md` | Propagation, delegation, CustomEvent, AbortController |
| 23 | `23-forms-validation.md` | Constraint Validation API, FormData, a11y, async checks |
| 24 | `24-timers-scheduling.md` | 4ms clamp, drift, rAF, debounce/throttle, `scheduler` |
| 25 | `25-storage-browser-apis.md` | Web Storage, cookies, IndexedDB, observers, workers |
| 26 | `26-regular-expressions.md` | Flags, groups, lookaround, `lastIndex`, ReDoS, `\p{}` |
| 27 | `27-memory-garbage-collection.md` | Reachability, generational GC, 4 leaks, WeakRef |
| 28 | `28-performance.md` | Core Web Vitals, loading, thrashing, hidden classes, folklore |
| 29 | `29-functional-programming.md` | Purity, immutability, pipe/curry, memo, Result |
| 30 | `30-design-patterns.md` | Creational/structural/behavioural, JS-specific, DI |
| 31 | `31-security.md` | XSS sinks, CSP, Trusted Types, CSRF, pollution, tokens |
| 32 | `32-testing.md` | Levels, async traps, doubles, Testing Library, Playwright |
| 33 | `33-es6-to-es2025.md` | Year-by-year tables, transpile vs polyfill, browserslist |

### Appendices (`appendices/`)

| # | File | Contents |
|---|---|---|
| A | `A-interview-questions.md` | 340 questions, 13 sections, each with a **Follow-up:** line |
| B | `B-coding-challenges.md` | 121 challenges, 10 sections, worked solutions + complexity |
| C | `C-projects.md` | 20 project specs, easy → expert, no solutions |
| D | `D-roadmap.md` | 7 phases, checkpoints, 4-week interview sprint, honest timelines |
| E | `E-best-practices.md` | 199 checklist items across 15 categories |
| F | `F-cheatsheet.md` | Tables: methods, operators, `this`, regex, DOM, one-liners |
| G | `G-glossary.md` | 204 terms, alphabetical, chapter cross-references |

---

## 2. Chapter Template

Every chapter contains these sections, in this order.

```markdown
# Chapter N — Title

## Learning Objectives
- 3–5 bullets, action verbs

---

## N.1 Section
Plain-language explanation, then a heavily commented code block.
Use comparison tables wherever two things are being contrasted.

> ⚠️ **Gotcha** — a specific trap, stated concretely.
> 💼 **Interview** — how this is actually asked.

## N.2 ... (typically 4–6 numbered sections)

## N.x Common Mistakes and Best Practices
**Mistake: <name>.** One-line diagnosis.  (5–7 of these)
**Best practice: <name>.** One-line rationale.  (2–4 of these)

## N.y Real-World Example: <Named Thing>
A substantial, production-shaped module (40–90 lines), fully commented,
followed by ONE short paragraph explaining the single non-obvious design decision.

## N.z Questions
20–28 questions, continuously numbered, in this compact inline format:
**1.** MCQ — question a) x b) y c) z d) w
<details><summary>Answer</summary>**b)** brief reason.</details>

Format labels used, roughly in this proportion:
MCQ (5–6) → Short (5–6) → Output (4–5) → Debug (4–6) → Code (3–4) → Scenario (2–3)
"Output" questions show a snippet and ask what prints.
"Debug" questions describe a symptom in prose and ask for the cause+fix.
"Code" answers contain a full working implementation.
"Scenario" answers are 3–5 sentence engineering-judgement paragraphs.

---

## Chapter Summary
6 tight bullets, each a takeaway not a topic name.

---

## Practice Exercises
5 numbered tasks, increasing difficulty, no solutions.

## Mini Project: <Name>   ← where the topic supports one
2–4 sentence spec plus a bulleted requirements list.

---

**Next:** [Chapter N+1 — Title →](NN-slug.md)
```

**Tone:** direct, opinionated where justified, zero filler. Honest about trade-offs
("measure first", "don't rely on JIT folklore"). Never pad.

**Code style:** `// output` as trailing comments. Mark wrong examples `// ❌` and correct ones
`// ✅`. Prefer modern syntax throughout (ESM, `#private`, `??=`, `AbortSignal.any`,
`Object.groupBy`, `Array.prototype.at`).

---

## 3. Known Deviations

Deliberate, and recorded so nobody "fixes" them by accident.

**Chapter length varies more than originally planned.** The brief specified 1,500–1,900 words with
chapters 1–7 allowed to run to 2,900. In practice chapters 1–23 run 1,550–3,000 and chapters 24–33
run 3,300–4,500, because the later topics carry more code per idea — a security chapter needs the
vulnerable version *and* the fix, and a performance chapter needs the measurement alongside the
change. Don't pad the early chapters to match, and don't cut the later ones; the structure is
identical either way, which is what a reader actually navigates by.

**Chapter 33 has no Real-World Example section.** The tables *are* that chapter, and a
production-shaped module would be filler. It's the one chapter that legitimately skips a
contract section.

**Chapter Summary bullet counts vary from 5 to 7** in chapters 1, 2, 9, 10, 13, 15, and 16. The
template says 6. Trimming a genuine takeaway to hit a number would be worse than the
inconsistency — leave them.

**Mini Projects are absent from 12 chapters** (8, 10, 11, 13, 15–18, 20–23). The template marks
them optional: "where the topic supports one". Don't invent one to fill a gap.

**Questions in chapters 1–14 were converted, not authored, in the inline format.** They were
originally grouped under `### Multiple Choice`-style sub-headings with unlabelled numbers, and were
migrated to the `**N.** MCQ — …` format used by chapters 15–33. If a question reads slightly
oddly with its label, that's why.

---

## 4. Verification

Before committing a change to any chapter or appendix:

```bash
# Every <details> has a matching close
grep -c '<details>' FILE; grep -c '</details>' FILE

# Question numbering is continuous with no gaps or repeats
grep -o '^\*\*[0-9]\+\.\*\*' FILE | grep -o '[0-9]\+'

# Every question carries a type label
grep -c '^\*\*[0-9]\+\.\*\* \(MCQ\|Short\|Output\|Debug\|Code\|Scenario\) —' FILE

# Section numbers match the filename's chapter number
grep -o '^## [0-9]\+\.[0-9]\+' FILE
```

And the whole-repo checks:

- Every internal link resolves, including each chapter's `**Next:**` footer. Grep for the old
  filename before renaming anything.
- Cross-references (`→ Ch 27`, `Chapter 31`) name a chapter that exists and covers that topic.
- The README's table has no `*(pending)*` rows left, and every row links to a real file.

---

## 5. Adding a Chapter

Chapters 1–33 form a deliberate progression, so a new chapter usually belongs as an appendix
instead. If one genuinely belongs in the sequence:

1. Copy [`../_template/chapters/00-chapter-template.md`](../_template/chapters/00-chapter-template.md).
2. Insert it at the end rather than renumbering — renumbering breaks every cross-reference in the
   book, and there are hundreds.
3. Update the previous chapter's `**Next:**` footer and add a row to the README table.
4. Run the checks in §4.
