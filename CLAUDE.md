# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A collection of book-length technical learning guides written in Markdown. **There is no code to
build, no dependencies to install, and no test runner.** The deliverable is prose and runnable
example snippets. Every change is an edit to a `.md` file.

Two topics and a scaffold:

- `javascript/` — **complete**: 33 chapters + 7 appendices, ~147k words.
- `nestjs/` — **complete**: 36 chapters + 7 appendices, ~175k words. Targets NestJS 11 on Node 20+.
  See [nestjs/CONTINUE-HERE.md](nestjs/CONTINUE-HERE.md) §4 Known Deviations before touching it.
- `_template/` — the starting point for a new topic.

## Where the Conventions Actually Live

Read these before editing content — they are the real spec, and they are more specific than the
READMEs:

- [_template/chapters/00-chapter-template.md](_template/chapters/00-chapter-template.md) — the
  section-order contract for a chapter, with inline HTML comments explaining each section's intent
  and target length. Copy this to start a chapter.
- [javascript/CONTINUE-HERE.md](javascript/CONTINUE-HERE.md) — maintainer notes for the JavaScript
  book: the full chapter/appendix inventory, the template restated, **§3 Known Deviations**, the
  §4 verification greps, and the §5 procedure for adding a chapter.
- [_template/README.md](_template/README.md) — the shape of a topic README.

> These two files used to point at a `CONTRIBUTING.md` at the repo root that has never existed.
> Both now point here instead. There is no `CONTRIBUTING.md` — this file plus `_template/` are the
> conventions source of truth.

## Verification (this repo's substitute for tests)

Run these against any chapter or appendix you touch. From `javascript/`:

```bash
F=chapters/09-closures.md

# <details> tags balance (a stray one silently swallows the rest of the file on GitHub)
echo "open=$(grep -c '<details>' $F) close=$(grep -c '</details>' $F)"

# Every question is numbered continuously AND carries a type label — these two must be equal
grep -c '^\*\*[0-9]\+\.\*\*' $F
grep -c '^\*\*[0-9]\+\.\*\* \(MCQ\|Short\|Output\|Debug\|Code\|Scenario\) —' $F

# Section numbers match the file's chapter number, with no gaps
grep -o '^## [0-9]\+\.[0-9]\+' $F
```

Repo-wide, before committing:

- Every internal link resolves, including each file's `**Next:**` footer. The footers form one
  chain: ch01 → … → ch33 → Appendix A → … → Appendix G (G has no footer; it ends the book).
- Cross-references in prose (`→ Ch 27`, `Chapter 31`) name a chapter that exists and actually
  covers that topic.
- No `*(pending)*` rows left in a README table, and every row links to a real file.

> ⚠️ `appendices/B-coding-challenges.md` contains two **literal NUL bytes** — they are intentional,
> a `\0` join separator inside a `memoize` example around line 1017. The side effect is that `grep`
> and ripgrep classify the file as binary and print `Binary file … matches` instead of the matching
> lines, so it drops out of repo-wide sweeps. Pass `grep -a` when searching it. Do not "clean up"
> those bytes — the surrounding prose explains why the separator is what it is.

## Editing Rules That Are Easy to Get Wrong

**Never renumber chapters.** Hundreds of cross-references and every `**Next:**` footer are keyed to
chapter numbers. A new chapter goes at the end, never in the middle — and per `CONTINUE-HERE.md` §5,
a new topic usually belongs as an appendix rather than in the 1–33 progression at all.

**The deviations in `CONTINUE-HERE.md` §3 are deliberate.** Chapter 33 has no Real-World Example,
12 chapters have no Mini Project, summary bullet counts vary from 5 to 7, and later chapters run far
longer than early ones. These were recorded specifically so nobody would normalize them. Verified as
still accurate — leave them alone.

**Don't pad to hit a number.** The template's counts (6 summary bullets, 20–28 questions, 5–7
mistakes) are targets, not quotas. A genuine takeaway trimmed to hit six is worse than seven bullets.

**Practice Exercises have no solutions, on purpose.** Don't add them.

## Content Formatting Contract

```js
const doubled = [1, 2, 3].map(n => n * 2);
console.log(doubled); // [2, 4, 6]   ← output is always a trailing comment
```

- Wrong/right examples are marked `// ❌ Wrong — why` and `// ✅ Right — why`.
- Two callout types only: `> ⚠️ **Gotcha** — <concrete trap, naming the exact input and result>`
  and `> 💼 **Interview** — <how it's actually asked>`.
- Questions use a compact inline format with a collapsed answer, all on adjacent lines:

  ```markdown
  **1.** MCQ — Which is falsy? a) `'0'` b) `[]` c) `0` d) `{}`
  <details><summary>Answer</summary>**c)** brief reason.</details>
  ```

- Comparison tables wherever two things are contrasted.
- Code comments explain **why**, not what.
- Tone: direct, opinionated where justified, honest about trade-offs, zero filler.
- JavaScript examples target Node 22+ / modern browsers and prefer modern syntax (ESM, `#private`,
  `??=`, `Object.groupBy`, `Array.prototype.at`). Browser-only and Node-only APIs are labeled.

## Repository Layout

```
<topic>/
├── README.md          TOC, who it's for, conventions, environment note
├── CONTINUE-HERE.md   maintainer notes: inventory, deviations, checks
├── chapters/          NN-slug.md, read in order
└── appendices/        X-slug.md, reference material
```

`_template/` mirrors this and is the starting point for a new topic. Adding a topic also means
adding a row to the table in the root [README.md](README.md).

Note: `.gitignore` excludes `.claude/`, so local permission settings are not tracked.
