# Appendix D — The JavaScript Roadmap

A staged plan, with checkpoints you can honestly test yourself against.

**Read this first, because it changes how you'll use everything below.** Real proficiency in
JavaScript takes about two years of steady work, and roughly 3–6 months to be genuinely employable
at junior level. Anyone selling you three weeks is selling you the feeling of progress. The gap
isn't syntax — you'll have that in a month. The gap is the hundred small judgements that only come
from having shipped something, broken it, and fixed it.

The plan below assumes 10–15 hours a week and that you already program in some language. Scale the
durations to your actual time; the *order* matters more than the timeline.

---

## The Phases at a Glance

| Phase | Focus | Chapters | Duration |
|---|---|---|---|
| 1 | Foundations | 1–7 | 3–4 weeks |
| 2 | Core mechanics | 8–17 | 4–6 weeks |
| 3 | Asynchronous JavaScript | 18–20 | 2–3 weeks |
| 4 | The browser | 21–25 | 3–4 weeks |
| 5 | Advanced and professional | 26–33 | 4–6 weeks |
| 6 | The ecosystem | — | 4–6 weeks |
| 7 | Specialisation | — | ongoing |

Total to "competent and employable": roughly 5–7 months at this pace. Phases 1–5 are this book;
phase 6 onward is not.

---

## Phase 1 — Foundations

**Duration.** 3–4 weeks.
**Read.** Chapters [1](../chapters/01-how-javascript-works.md)–[7](../chapters/07-strings-numbers-math-dates.md).
**Build.** Projects [1](C-projects.md#1-tip-calculator) (tip calculator),
[2](C-projects.md#2-to-do-list) (to-do list), [3](C-projects.md#3-digital-clock-and-countdown) (clock).

The temptation here is to skim, because you've seen variables and loops before. Don't — this is
where JavaScript's specific weirdness lives, and every one of these will cost you a debugging
session later if you skip it.

**Focus on:** the difference between engine and runtime; why `==` coerces and `===` doesn't; the
eight falsy values; which array methods mutate; and why `0.1 + 0.2 !== 0.3`.

**You're ready to move on when you can:**

- [ ] List all eight falsy values without looking
- [ ] Predict the output of `[] + {}`, `'5' - 3`, `'5' + 3`, and explain each
- [ ] Say which of `sort`, `slice`, `splice`, `map`, `reverse` mutate, and name the non-mutating equivalent
- [ ] Explain why money should never be a float, and what to use instead
- [ ] Write a function that reads and updates the DOM without a framework
- [ ] Explain the difference between `let`, `const`, and `var` in three sentences

**Common trap.** Learning a framework now. You'll spend a year unable to tell framework behaviour
from language behaviour, which makes every bug twice as hard.

---

## Phase 2 — Core Mechanics

**Duration.** 4–6 weeks. This is the longest phase and the one that separates people.
**Read.** Chapters [8](../chapters/08-scope-hoisting-tdz.md)–[17](../chapters/17-modules.md).
**Build.** Projects [4](C-projects.md#4-quiz-app) (quiz), [6](C-projects.md#6-markdown-previewer)
(markdown previewer), [8](C-projects.md#8-expense-tracker) (expense tracker).

Scope, closures, `this`, and prototypes are the four things interviewers use to sort juniors from
mids, and they're the four things that make framework source code readable. Spend the time.

**Focus on:** closures capturing variables rather than values; the five `this` binding rules; the
prototype chain as delegation; and why ESM's static structure enables tree shaking.

**You're ready to move on when you can:**

- [ ] Explain the `var`-in-a-loop closure bug and fix it two different ways
- [ ] Predict `this` in five different call sites and say which rule applies
- [ ] Implement inheritance with `Object.create` and no `class` keyword
- [ ] Write `once`, `memoize`, and a private counter using only closures
- [ ] Explain the TDZ and why it throws instead of giving `undefined`
- [ ] Say what a live binding is and why CommonJS doesn't have them
- [ ] Build a custom error hierarchy with `cause` and use it

**Common trap.** Memorising the `this` rules without practising them. Write ten snippets, predict
the output, then run them. Getting three wrong teaches you more than reading the rules five times.

---

## Phase 3 — Asynchronous JavaScript

**Duration.** 2–3 weeks. Short, and non-negotiable.
**Read.** Chapters [18](../chapters/18-event-loop.md)–[20](../chapters/20-fetch-json-http.md).
**Build.** Projects [5](C-projects.md#5-weather-dashboard) (weather dashboard),
[9](C-projects.md#9-pomodoro-timer) (pomodoro), [12](C-projects.md#12-autocomplete) (autocomplete).

Nearly every production bug that isn't a typo is an async bug. This phase is where you stop
treating `await` as magic.

**Focus on:** the microtask/macrotask split; that `fetch` doesn't reject on 404; the four promise
combinators and when each is right; and `AbortController` for cancellation.

**You're ready to move on when you can:**

- [ ] Predict the output of a mixed `setTimeout`/`Promise`/`queueMicrotask`/sync snippet
- [ ] Explain why `forEach` with an async callback doesn't wait
- [ ] Choose correctly between `all`, `allSettled`, `race`, and `any` for a given requirement
- [ ] Write retry with exponential backoff and jitter, and say why jitter matters
- [ ] Cancel an in-flight request when a new one supersedes it
- [ ] Explain the difference between `return promise` and `return await promise` inside a `try`
- [ ] Spot an unawaited promise in code review

**Common trap.** Believing you understand the event loop because you've read about it. Do the
output-prediction challenges ([Appendix B](B-coding-challenges.md#10-output-prediction)) and check
your answers. Most people get the promise-assimilation one wrong.

---

## Phase 4 — The Browser

**Duration.** 3–4 weeks.
**Read.** Chapters [21](../chapters/21-dom.md)–[25](../chapters/25-storage-browser-apis.md).
**Build.** Projects [7](C-projects.md#7-image-carousel) (carousel),
[10](C-projects.md#10-infinite-scroll-gallery) (infinite scroll),
[11](C-projects.md#11-drag-and-drop-kanban) (kanban).

Frameworks abstract this away, which is exactly why you should learn it first — you can't reason
about a framework's performance without knowing what it's doing to the DOM.

**Focus on:** event delegation and propagation; reflow versus repaint; the four observer APIs
replacing polling; and the real differences between storage mechanisms.

**You're ready to move on when you can:**

- [ ] Implement event delegation with `closest()` and explain when it doesn't work
- [ ] Name the layout-forcing properties and explain layout thrashing
- [ ] Choose between `localStorage`, IndexedDB, cookies, and the Cache API for a requirement
- [ ] Replace a scroll handler with an `IntersectionObserver`
- [ ] Build a component with `AbortController` teardown and prove there's no leak
- [ ] Make a custom widget keyboard-accessible and screen-reader usable
- [ ] Explain why `pushState` doesn't fire `popstate`

**Common trap.** Skipping accessibility as "polish". It's a requirement, it's frequently a legal
one, and it's the fastest signal of seniority in a code review.

---

## Phase 5 — Advanced and Professional

**Duration.** 4–6 weeks.
**Read.** Chapters [26](../chapters/26-regular-expressions.md)–[33](../chapters/33-es6-to-es2025.md).
**Build.** Projects [13](C-projects.md#13-virtualized-data-table)–[19](C-projects.md#19-test-runner) —
pick two.

This is the phase most self-taught developers skip, and it's the one that shows. Security, testing,
and performance are what make the difference between code that works on your machine and code that
survives production.

**Focus on:** the XSS sinks and CSP; measuring before optimising; what to mock and what not to; and
reading heap snapshots.

**You're ready to move on when you can:**

- [ ] Name five XSS sinks and the safe alternative for each
- [ ] Write a CSP and explain why `unsafe-inline` defeats it
- [ ] Exploit prototype pollution through a recursive merge, then fix it three ways
- [ ] Find a memory leak from a heap snapshot's retainer chain
- [ ] Diagnose a slow interaction from a Performance profile rather than guessing
- [ ] Write a test that would catch a regression you actually shipped
- [ ] Explain why a fully-mocked unit suite can be green while the app is broken
- [ ] Say which edition introduced `?.`, `#private`, `toSorted`, and `Object.groupBy`

**Common trap.** Treating security as someone else's job. One `innerHTML` is one breach.

---

## Phase 6 — The Ecosystem

**Duration.** 4–6 weeks, and genuinely never finished.

Now, and only now, learn the tools. You'll learn them far faster with the language underneath, and
you'll be able to tell a framework bug from your own.

**In order of value:**

**TypeScript** — the highest-value single addition. Start with `checkJs` and JSDoc on existing
files, then convert with `strict` on. Learn what it doesn't do: types are erased at runtime, so you
still validate at boundaries. *2–3 weeks to productive.*

**One framework, properly** — React, Vue, or Svelte. Learn one deeply rather than three shallowly;
the concepts transfer and the APIs don't matter. Build something with routing, data fetching, and
forms, because that's where the real questions are. *4–6 weeks to productive.*

**Node.js** — enough to build the API your frontend needs, and to understand why `require` can't
load ESM. Also where you'll finally need streams and backpressure for real. *2–3 weeks.*

**Build tooling** — Vite for new work; enough webpack to maintain what exists. Understand what
`browserslist` controls, and set a bundle budget. *1 week.*

**Testing in practice** — Vitest, Testing Library, Playwright. Write tests for something you've
already built, which teaches you far more than tests written alongside. *1–2 weeks.*

**Accessibility** — WCAG at a working level, the ARIA patterns you'll actually use, and a screen
reader you can navigate. This is a genuine differentiator. *1–2 weeks, then ongoing.*

**Security** — revisit [Chapter 31](../chapters/31-security.md) after you've shipped something.
It reads differently when the code is yours. *Ongoing.*

**Git, properly** — rebase, bisect, reflog, and how to recover from your own mistakes. `git bisect`
alone will pay for the time. *A few days.*

---

## Phase 7 — Specialisation

Ongoing, and where you stop following anyone's roadmap. Some directions worth knowing exist:

| Direction | What it involves |
|---|---|
| **Frontend architecture** | Design systems, monorepos, module boundaries, migration strategy |
| **Performance engineering** | Core Web Vitals at scale, RUM, rendering internals, profiling |
| **Full-stack / backend** | Node or Deno, databases, API design, queues, observability |
| **Developer tooling** | Bundlers, compilers, linters, ASTs, language servers |
| **Data visualisation** | D3, Canvas, WebGL, and the perception side of chart design |
| **Graphics and games** | WebGL/WebGPU, game loops, physics, asset pipelines |
| **Accessibility** | Auditing, remediation, assistive tech, standards work |
| **Security** | Application security, threat modelling, audits |

Pick one because a problem you hit made you curious, not because it looks employable. The
motivation matters more than the market.

---

## The Four-Week Interview Sprint

For when you have a specific interview. This assumes you've done phases 1–4 already; it's revision,
not learning.

### Week 1 — The Fundamentals They Actually Ask

- **Days 1–2.** Chapters [8](../chapters/08-scope-hoisting-tdz.md)–[10](../chapters/10-this-and-binding.md)
  (scope, closures, `this`). Then [Appendix A §3](A-interview-questions.md#3-scope-closures-and-this) —
  all 35 questions, out loud.
- **Days 3–4.** Chapters [11](../chapters/11-prototypes-inheritance.md)–[12](../chapters/12-classes-oop.md)
  (prototypes, classes) and [Appendix A §4](A-interview-questions.md#4-objects-prototypes-and-classes).
- **Day 5.** Chapters [2](../chapters/02-variables-types-operators.md),
  [5](../chapters/05-objects.md), [6](../chapters/06-arrays.md) skimmed, plus
  [Appendix A §2](A-interview-questions.md#2-fundamentals-and-types).
- **Weekend.** [Appendix B](B-coding-challenges.md) sections 1–4 — strings, arrays, objects,
  closures. Write them out; don't read the solutions first.

### Week 2 — Async, Which Is Where People Fail

- **Days 1–2.** Chapters [18](../chapters/18-event-loop.md)–[19](../chapters/19-async-javascript.md).
  Every output-prediction question in [Appendix B §10](B-coding-challenges.md#10-output-prediction).
- **Day 3.** [Appendix A §6](A-interview-questions.md#6-async-and-the-event-loop) — all 45.
- **Days 4–5.** Implement, from scratch and without looking:
  `Promise`, `Promise.all`, `Promise.race`, `Promise.any`, `promisify`, a concurrency pool, retry
  with backoff.
- **Weekend.** [Appendix B §5](B-coding-challenges.md#5-async) and
  [§8](B-coding-challenges.md#8-polyfills) polyfills. Do `debounce`, `throttle`, `bind`, and
  `EventEmitter` until they're automatic — these four come up more than anything else.

### Week 3 — Browser, Performance, Security

- **Days 1–2.** Chapters [21](../chapters/21-dom.md)–[22](../chapters/22-events.md) and
  [Appendix A §7](A-interview-questions.md#7-dom-events-and-the-browser).
- **Day 3.** Chapter [28](../chapters/28-performance.md) and
  [Appendix A §9](A-interview-questions.md#9-performance-and-memory). Know the Core Web Vitals and
  what moves each.
- **Day 4.** Chapter [31](../chapters/31-security.md) and
  [Appendix A §10](A-interview-questions.md#10-security). Know five XSS sinks cold.
- **Day 5.** Chapters [27](../chapters/27-memory-garbage-collection.md) and
  [32](../chapters/32-testing.md), skimmed, plus their question sections.
- **Weekend.** Build one small thing end to end under time pressure — an autocomplete with debounce
  and abort is the single most representative take-home task there is.

### Week 4 — System Design, Rehearsal, and Rest

- **Days 1–2.** Chapter [30](../chapters/30-design-patterns.md) and
  [Appendix A §12](A-interview-questions.md#12-patterns-and-architecture). Practise saying "I'd use
  X because Y, and the cheaper alternative is Z".
- **Day 3.** Prepare three project stories in situation → action → outcome → hindsight form. Each
  needs a real technical detail and a real mistake. Rehearse them out loud, under two minutes each.
- **Day 4.** [Appendix A §1](A-interview-questions.md#1-rapid-fire-warm-up) rapid fire, twice
  through. Then [Appendix E](E-best-practices.md) as a checklist against your own portfolio code —
  you'll be asked about it.
- **Day 5.** A full mock interview with someone else. Being watched changes your performance more
  than any amount of solo practice.
- **Weekend.** Rest. Genuinely. Cramming the night before costs you more in recall than it adds in
  knowledge.

**Throughout.** Twenty minutes a day of writing code by hand, without autocomplete. And prepare
questions for them — the ones in [Appendix A §0](A-interview-questions.md#0-how-to-answer) are a
starting point.

---

## What to Learn After JavaScript

Ordered by return on time invested, for someone employable in JavaScript who wants to be more:

1. **TypeScript** — not optional at this point in the industry.
2. **Testing, seriously** — the skill that most distinguishes mid from senior.
3. **A framework, deeply** — one, not three.
4. **HTTP and the network** — caching headers, CORS, TLS, HTTP/2. Vast leverage, rarely learned.
5. **Accessibility** — a differentiator, and the right thing.
6. **SQL and data modelling** — even for frontend work.
7. **Node or another backend** — being able to build the whole thing changes what you can ship.
8. **CS fundamentals, selectively** — complexity analysis and the common data structures. You don't
   need a degree; you do need to know why `includes` inside `filter` is quadratic.
9. **Observability** — logs, metrics, traces, and error tracking. Debugging production is a distinct
   skill.
10. **Another language** — Go, Rust, or Python. It's the fastest way to see which of your habits are
    JavaScript's and which are yours.

---

## How Long This Actually Takes

Honest numbers, from watching people do it:

| Milestone | Realistic time (10–15 h/week) |
|---|---|
| Can build a working interactive page | 4–6 weeks |
| Can explain closures, `this`, and the event loop correctly | 3 months |
| Employable at junior level | 4–7 months |
| Comfortable in an unfamiliar codebase | 1 year |
| Trusted with architectural decisions | 2–3 years |
| Genuinely expert — you know what you don't know | 4–5 years |

Two things move these numbers more than anything else. **Building things that are slightly too hard**
beats any amount of reading or tutorials, because only real projects produce the states you didn't
think about. And **reading other people's code** — the source of a library you use, or a colleague's
pull request — teaches judgement, which is the thing you can't get from documentation.

Two things reliably slow people down. **Tutorial hell**, where you follow along and feel productive
without ever making a decision. And **framework-first learning**, which leaves you unable to
distinguish the framework's behaviour from the language's, so every bug takes twice as long.

Plateaus are normal, and the second one — around the year mark, when you're competent but not fast —
is the one people quit on. It passes.

---

**Next:** [Appendix E — Best Practices Checklist →](E-best-practices.md)
