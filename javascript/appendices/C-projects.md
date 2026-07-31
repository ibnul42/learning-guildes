# Appendix C — 20 Real-World Projects

Ordered easy to advanced. Each specifies the goal, the chapters it exercises, hard requirements,
stretch goals, and a realistic time estimate.

**No solutions.** That's deliberate — a project you've been shown is a tutorial, and tutorials
produce the illusion of competence. Build them badly, then fix them.

**Rules that apply to every project here:**

- Vanilla JavaScript unless the project says otherwise. Frameworks hide the mechanics you're
  practising.
- Keyboard-accessible, and usable with a screen reader. If it can't be operated without a mouse,
  it isn't finished.
- No unhandled promise rejections, and no listener left behind on teardown.
- Every network call handles the failure path — offline, 500, and slow.

Estimates assume you already program and are learning JavaScript specifically. Halve them if you're
experienced; double them if you're also learning CSS.

---

## Contents

| # | Project | Difficulty | Time |
|---|---------|-----------|------|
| 1 | [Tip Calculator](#1-tip-calculator) | Easy | 2–3 h |
| 2 | [To-Do List](#2-to-do-list) | Easy | 4–6 h |
| 3 | [Digital Clock and Countdown](#3-digital-clock-and-countdown) | Easy | 3–5 h |
| 4 | [Quiz App](#4-quiz-app) | Easy | 6–8 h |
| 5 | [Weather Dashboard](#5-weather-dashboard) | Easy–Medium | 6–10 h |
| 6 | [Markdown Previewer](#6-markdown-previewer) | Medium | 8–12 h |
| 7 | [Image Carousel](#7-image-carousel) | Medium | 6–10 h |
| 8 | [Expense Tracker](#8-expense-tracker) | Medium | 10–14 h |
| 9 | [Pomodoro Timer](#9-pomodoro-timer) | Medium | 8–12 h |
| 10 | [Infinite-Scroll Gallery](#10-infinite-scroll-gallery) | Medium | 10–14 h |
| 11 | [Drag-and-Drop Kanban](#11-drag-and-drop-kanban) | Medium–Hard | 16–24 h |
| 12 | [Autocomplete](#12-autocomplete) | Medium–Hard | 10–16 h |
| 13 | [Virtualized Data Table](#13-virtualized-data-table) | Hard | 20–30 h |
| 14 | [Client-Side Router and SPA](#14-client-side-router-and-spa) | Hard | 20–30 h |
| 15 | [Offline-First Notes](#15-offline-first-notes) | Hard | 30–40 h |
| 16 | [Real-Time Chat](#16-real-time-chat) | Hard | 25–35 h |
| 17 | [Canvas Drawing App](#17-canvas-drawing-app) | Hard | 25–35 h |
| 18 | [State Management Library](#18-state-management-library) | Hard | 20–30 h |
| 19 | [Test Runner](#19-test-runner) | Hard | 20–30 h |
| 20 | [Mini Reactive Framework](#20-mini-reactive-framework) | Expert | 40–60 h |

---

## 1. Tip Calculator

**Goal.** Take a bill total, a tip percentage, and a party size, and show the tip, total, and
per-person amount, updating live as the user types.

**Concepts practised.** [Ch 2 — Variables and Types](../chapters/02-variables-types-operators.md),
[Ch 7 — Numbers](../chapters/07-strings-numbers-math-dates.md),
[Ch 21 — The DOM](../chapters/21-dom.md), [Ch 22 — Events](../chapters/22-events.md)

**Requirements**

- [ ] Live update on `input`, not on a submit button
- [ ] Money handled as integer pence — never a float (`0.1 + 0.2 !== 0.3`)
- [ ] Format output with `Intl.NumberFormat`, honouring the user's locale
- [ ] Reject negative and non-numeric input without letting the UI show `NaN`
- [ ] Party size of zero doesn't produce `Infinity`
- [ ] Preset tip buttons (10/15/20%) plus a custom field, kept in sync
- [ ] Keyboard-operable, with labels associated to every input
- [ ] Works with the currency symbol before or after the amount, depending on locale

**Stretch goals**

1. Split unevenly — assign line items to people.
2. Round the total up and show how the rounding is distributed.
3. Remember the last-used tip percentage across reloads.

**Estimated time.** 2–3 hours.

---

## 2. To-Do List

**Goal.** Add, complete, edit, delete, and filter tasks, persisting between sessions.

**Concepts practised.** [Ch 6 — Arrays](../chapters/06-arrays.md),
[Ch 21 — The DOM](../chapters/21-dom.md), [Ch 22 — Events](../chapters/22-events.md),
[Ch 25 — Storage](../chapters/25-storage-browser-apis.md)

**Requirements**

- [ ] One delegated listener for the whole list, not one per row
- [ ] Filter by all / active / completed, reflected in the URL hash so it survives a reload
- [ ] Edit in place on double-click, with Escape to cancel and Enter to commit
- [ ] Persist to `localStorage`, guarded so a corrupt value can't crash startup
- [ ] Namespaced, versioned storage key (`todos:v1`)
- [ ] Render task text with `textContent` — prove it by adding a task called `<img src=x onerror=alert(1)>`
- [ ] An empty state, and a live-region announcement of the remaining count
- [ ] No task lost if the tab is closed mid-edit

**Stretch goals**

1. Undo the last destructive action, with a 5-second toast.
2. Reorder by drag-and-drop using Pointer Events.
3. Sync across tabs with `BroadcastChannel`.

**Estimated time.** 4–6 hours.

---

## 3. Digital Clock and Countdown

**Goal.** A clock showing the current time, plus a countdown to a target datetime that stays
accurate no matter how the browser throttles it.

**Concepts practised.** [Ch 7 — Dates](../chapters/07-strings-numbers-math-dates.md),
[Ch 24 — Timers and Scheduling](../chapters/24-timers-scheduling.md)

**Requirements**

- [ ] Remaining time computed from `target - Date.now()`, never by decrementing a counter
- [ ] Self-correcting timeout so ticks land near second boundaries rather than drifting
- [ ] Correct after the tab is backgrounded for ten minutes — test this, don't assume it
- [ ] Pause on `visibilitychange` when hidden; one immediate update on becoming visible
- [ ] Format with `Intl.DateTimeFormat`, respecting 12- and 24-hour locales
- [ ] Complete event fires exactly once, even if stop and expiry race
- [ ] Handles a target in the past without going negative or looping
- [ ] Clean teardown — no timer left running after the component is destroyed

**Stretch goals**

1. Multiple timezones side by side, with correct DST handling.
2. Rewrite the date handling with `Temporal` (via the polyfill) and note what got simpler.
3. An analogue face drawn with SVG, driven by the same clock source.

**Estimated time.** 3–5 hours.

---

## 4. Quiz App

**Goal.** Present multiple-choice questions one at a time, score the result, and show a review of
wrong answers.

**Concepts practised.** [Ch 5 — Objects](../chapters/05-objects.md),
[Ch 6 — Arrays](../chapters/06-arrays.md), [Ch 22 — Events](../chapters/22-events.md),
[Ch 30 — Design Patterns](../chapters/30-design-patterns.md)

**Requirements**

- [ ] Questions loaded from JSON, not hardcoded in the render logic
- [ ] Model the flow as an explicit state machine (`idle → question → feedback → results`), not booleans
- [ ] Options shuffled with Fisher-Yates — not `sort(() => Math.random() - 0.5)`
- [ ] Per-question timer that auto-advances, with the remaining time visible
- [ ] Progress indicator, and the ability to go back without losing answers
- [ ] Results screen listing each wrong answer with the correct one
- [ ] Fully keyboard-operable: number keys select, Enter confirms, arrows navigate
- [ ] Screen-reader announcement when the question changes

**Stretch goals**

1. Question categories and difficulty, with a weighted selection.
2. Persist an in-progress attempt so a refresh resumes it.
3. Spaced repetition — resurface questions you got wrong.

**Estimated time.** 6–8 hours.

---

## 5. Weather Dashboard

**Goal.** Search for a city and display current conditions plus a forecast, from a real API.

**Concepts practised.** [Ch 19 — Async](../chapters/19-async-javascript.md),
[Ch 20 — Fetch and HTTP](../chapters/20-fetch-json-http.md),
[Ch 16 — Error Handling](../chapters/16-error-handling.md)

**Requirements**

- [ ] Check `res.ok` — `fetch` does not reject on a 404 or 500
- [ ] Distinct UI for loading, empty, error, and offline states
- [ ] Abort the previous request when a new search starts
- [ ] Debounce the search input, and dedupe identical in-flight requests
- [ ] Retry transient failures (5xx, network) with exponential backoff and jitter; never retry a 4xx
- [ ] Cache responses with a TTL so repeat searches are instant
- [ ] API key never in client code — proxy through a tiny server, or use a keyless API
- [ ] Units toggle (°C/°F) that doesn't refetch

**Stretch goals**

1. Geolocation with a graceful fallback when permission is denied.
2. A temperature chart from the forecast data.
3. Compare two cities side by side, fetched in parallel.

**Estimated time.** 6–10 hours.

---

## 6. Markdown Previewer

**Goal.** A split editor and live preview, converting a Markdown subset to safe HTML.

**Concepts practised.** [Ch 26 — Regular Expressions](../chapters/26-regular-expressions.md),
[Ch 31 — Security](../chapters/31-security.md), [Ch 21 — The DOM](../chapters/21-dom.md)

**Requirements**

- [ ] Headings, bold, italic, inline code, links, lists, blockquotes, and fenced code blocks
- [ ] Fenced-code contents not processed for other syntax
- [ ] All text HTML-escaped before insertion; test with `<script>alert(1)</script>` and `[x](javascript:alert(1))`
- [ ] Link URLs validated against a protocol allowlist
- [ ] Every regex linear-time — document why each one can't backtrack catastrophically
- [ ] Preview updates coalesced into one `requestAnimationFrame`, not per keystroke
- [ ] Synchronised scrolling between editor and preview
- [ ] Content persisted so a refresh doesn't lose work

**Stretch goals**

1. Tables and task lists.
2. A written note on which features pushed you toward a real parser, and why regex ran out.
3. Export to a self-contained HTML file.

**Estimated time.** 8–12 hours.

---

## 7. Image Carousel

**Goal.** A slideshow with autoplay, manual navigation, and touch support.

**Concepts practised.** [Ch 22 — Events](../chapters/22-events.md),
[Ch 24 — Timers](../chapters/24-timers-scheduling.md),
[Ch 28 — Performance](../chapters/28-performance.md)

**Requirements**

- [ ] Animate with `transform` only — never `left` or `margin`
- [ ] Autoplay that pauses on hover, on focus, and when the tab is hidden
- [ ] Respects `prefers-reduced-motion` by disabling animation
- [ ] Touch and mouse via Pointer Events, with `setPointerCapture` and `touch-action: none`
- [ ] Lazy-load off-screen images; preload only the next and previous
- [ ] Correct ARIA: a labelled region, and slide changes announced politely
- [ ] Keyboard: arrows navigate, Home/End jump to first/last, Tab reaches only the visible slide's links
- [ ] Infinite looping with no visible jump at the seam

**Stretch goals**

1. Velocity-based swipe that snaps to the nearest slide.
2. Thumbnail strip synchronised to the main view.
3. Measure and report the frame rate during a swipe.

**Estimated time.** 6–10 hours.

---

## 8. Expense Tracker

**Goal.** Record income and expenses by category, show a running balance, and visualise the split.

**Concepts practised.** [Ch 6 — Arrays](../chapters/06-arrays.md),
[Ch 25 — Storage](../chapters/25-storage-browser-apis.md),
[Ch 29 — Functional Programming](../chapters/29-functional-programming.md)

**Requirements**

- [ ] All money as integer minor units; formatting only at the edges
- [ ] Pure reducer functions for every total — no computation inside render
- [ ] Immutable updates, so no operation mutates the stored transaction list
- [ ] Group by category with `Object.groupBy`, and by month
- [ ] Filter by date range, category, and free-text search, composable
- [ ] Storage layer with a versioned key and a quota-exceeded fallback
- [ ] CSV import that validates every row and reports all errors at once, not the first
- [ ] A chart of category totals (SVG or Canvas, drawn by hand)

**Stretch goals**

1. Recurring transactions, projected forward.
2. Budget limits per category with a warning at 80%.
3. Multi-currency, with rates fetched and cached.

**Estimated time.** 10–14 hours.

---

## 9. Pomodoro Timer

**Goal.** Cycle through work and break intervals with notifications and session history.

**Concepts practised.** [Ch 24 — Timers](../chapters/24-timers-scheduling.md),
[Ch 25 — Browser APIs](../chapters/25-storage-browser-apis.md),
[Ch 30 — Design Patterns](../chapters/30-design-patterns.md)

**Requirements**

- [ ] Accurate across tab-backgrounding — compute from timestamps, never tick counts
- [ ] Explicit state machine: `idle → work → shortBreak → work → … → longBreak`
- [ ] Configurable durations and long-break interval
- [ ] Notifications API with permission requested on a user gesture, not on load
- [ ] An audible cue that respects a mute setting and doesn't autoplay-block
- [ ] Session history persisted, with a daily count
- [ ] Document title shows the remaining time so it's visible in a background tab
- [ ] Pause, resume, skip, and reset all behave correctly mid-interval

**Stretch goals**

1. Cross-tab sync so two open tabs show one shared timer.
2. A weekly heatmap of completed sessions.
3. A Web Worker driving the countdown, to bypass timer throttling.

**Estimated time.** 8–12 hours.

---

## 10. Infinite-Scroll Gallery

**Goal.** Load images page by page as the user scrolls, with a responsive masonry layout.

**Concepts practised.** [Ch 25 — Observers](../chapters/25-storage-browser-apis.md),
[Ch 20 — Fetch](../chapters/20-fetch-json-http.md),
[Ch 28 — Performance](../chapters/28-performance.md)

**Requirements**

- [ ] `IntersectionObserver` on a sentinel with `rootMargin`, never a scroll listener
- [ ] Sentinel unobserved (or guarded) while loading, so one scroll fires one request
- [ ] `width`/`height` on every image so scrolling causes zero layout shift
- [ ] Native `loading="lazy"` plus `decoding="async"`
- [ ] Graceful end-of-results, and a retry affordance on failure
- [ ] `ResizeObserver` for the layout, not a `resize` listener
- [ ] Restore scroll position and loaded pages when returning via the back button
- [ ] Measured: CLS below 0.1 and no long tasks during scroll

**Stretch goals**

1. Recycle DOM nodes so memory stays flat after 1,000 images.
2. A lightbox with keyboard navigation and a focus trap.
3. Blur-up placeholders from a low-resolution preview.

**Estimated time.** 10–14 hours.

---

## 11. Drag-and-Drop Kanban

**Goal.** Columns of cards that can be reordered and moved between columns, persisted.

**Concepts practised.** [Ch 22 — Events](../chapters/22-events.md),
[Ch 21 — The DOM](../chapters/21-dom.md),
[Ch 24 — Scheduling](../chapters/24-timers-scheduling.md)

**Requirements**

- [ ] Pointer Events with `setPointerCapture` — one code path for mouse, touch, and pen
- [ ] Drag position driven by `transform` inside a `requestAnimationFrame`
- [ ] Drop targets computed from cached geometry, measured once per drag, not per move
- [ ] A keyboard-accessible alternative: pick up with Space, move with arrows, drop with Space
- [ ] Live-region announcements of position ("Card 2 of 5, In Progress")
- [ ] Optimistic reorder with rollback if persistence fails
- [ ] Auto-scroll when dragging near a container edge
- [ ] 60fps with 200 cards — profile it and show the frame chart

**Stretch goals**

1. Multi-select and drag several cards at once.
2. Cross-tab sync so a move in one tab appears in another.
3. Undo/redo for every board mutation.

**Estimated time.** 16–24 hours.

---

## 12. Autocomplete

**Goal.** A search input that suggests results from a remote API as the user types.

**Concepts practised.** [Ch 19 — Async](../chapters/19-async-javascript.md),
[Ch 24 — Debounce](../chapters/24-timers-scheduling.md),
[Ch 23 — Forms](../chapters/23-forms-validation.md)

**Requirements**

- [ ] Debounced input, with the debounced function created once
- [ ] `AbortController` per request, aborting the previous — a slow response must never overwrite a newer one
- [ ] Responses tagged with their query and discarded if stale, as a second defence
- [ ] Full ARIA combobox pattern: `aria-expanded`, `aria-controls`, `aria-activedescendant`
- [ ] Keyboard: arrows, Enter, Escape (close and restore the typed value), Home/End
- [ ] Focus stays in the input at all times — never moved into the list
- [ ] Result count announced through a polite live region
- [ ] Matched substrings highlighted, escaped so a user typing `<b>` sees literal text
- [ ] Empty, loading, error, and no-results states all distinct

**Stretch goals**

1. Client-side cache keyed by query prefix, so backspacing is instant.
2. Multi-select with removable chips.
3. Fuzzy matching, with results ranked by score.

**Estimated time.** 10–16 hours.

---

## 13. Virtualized Data Table

**Goal.** Display 100,000 rows with sorting, filtering, and resizable columns, at 60fps.

**Concepts practised.** [Ch 28 — Performance](../chapters/28-performance.md),
[Ch 21 — The DOM](../chapters/21-dom.md),
[Ch 27 — Memory](../chapters/27-memory-garbage-collection.md)

**Requirements**

- [ ] Only the visible window plus overscan in the DOM, with spacers preserving scroll height
- [ ] Row elements pooled and reused, not recreated per scroll
- [ ] Scroll updates coalesced into one `requestAnimationFrame`
- [ ] Sort and filter off the main thread in a Web Worker; the UI never blocks
- [ ] Multi-column sort with a stable comparator, nulls consistently last
- [ ] Column resize and reorder by drag, with widths persisted
- [ ] Keyboard navigation with a focus model that survives virtualisation
- [ ] Correct table semantics for screen readers despite the virtual DOM window
- [ ] Prove it: flat memory after scrolling the full range twice, and no long tasks

**Stretch goals**

1. Variable row heights with measured offsets and binary-search lookup.
2. Frozen columns and grouped headers.
3. Cell editing with validation and an undo stack.

**Estimated time.** 20–30 hours.

---

## 14. Client-Side Router and SPA

**Goal.** A single-page app with real URLs, nested routes, and code splitting.

**Concepts practised.** [Ch 17 — Modules](../chapters/17-modules.md),
[Ch 25 — History API](../chapters/25-storage-browser-apis.md),
[Ch 30 — Design Patterns](../chapters/30-design-patterns.md)

**Requirements**

- [ ] Path matching with parameters (`/users/:id`) and a wildcard 404
- [ ] `pushState` navigation *and* a render call — remember `pushState` doesn't fire `popstate`
- [ ] Intercepted link clicks, excluding modified clicks, external hosts, and `target=_blank`
- [ ] Route-level code splitting with dynamic `import()`, plus a loading state
- [ ] Nested routes with layouts that don't remount when only the child changes
- [ ] Route guards that can redirect, and cancel a navigation cleanly
- [ ] Scroll restoration on back/forward, and scroll-to-top on a new navigation
- [ ] Focus moved to the main heading on navigation, and the route announced
- [ ] Every in-flight request for an abandoned route aborted

**Stretch goals**

1. Prefetch a route's chunk on link hover.
2. Data loaders that fetch in parallel with the chunk, not after it.
3. View transitions with `document.startViewTransition`.

**Estimated time.** 20–30 hours.

---

## 15. Offline-First Notes

**Goal.** A note app that works fully offline and reconciles when the network returns.

**Concepts practised.** [Ch 25 — IndexedDB and Workers](../chapters/25-storage-browser-apis.md),
[Ch 19 — Async](../chapters/19-async-javascript.md),
[Ch 16 — Error Handling](../chapters/16-error-handling.md)

**Requirements**

- [ ] Notes in IndexedDB with cumulative, idempotent migrations
- [ ] All store operations complete before the transaction auto-commits — no `await` mid-transaction
- [ ] A service worker caching the app shell, so a cold offline load boots
- [ ] Outbound queue flushed on `online` and on app start, with idempotency keys
- [ ] Last-write-wins conflict resolution by timestamp, with the loser kept recoverable
- [ ] Cross-tab sync via `BroadcastChannel` so two tabs never diverge
- [ ] Quota indicator from `navigator.storage.estimate()`, and `persist()` requested
- [ ] Degrades to memory-only when IndexedDB is unavailable, without a blank screen
- [ ] Full-text search over notes, running in a worker

**Stretch goals**

1. CRDT-based merge so concurrent edits to one note both survive.
2. Attachments stored as `Blob`s with a size budget.
3. End-to-end encryption with `SubtleCrypto`, key derived from a passphrase.

**Estimated time.** 30–40 hours.

---

## 16. Real-Time Chat

**Goal.** Multi-room chat over WebSockets, with presence and reliable delivery.

**Concepts practised.** [Ch 19 — Async](../chapters/19-async-javascript.md),
[Ch 22 — Events](../chapters/22-events.md),
[Ch 31 — Security](../chapters/31-security.md)

**Requirements**

- [ ] Reconnect with exponential backoff and jitter, capped, and resumed from the last message id
- [ ] Heartbeat ping/pong to detect a half-open connection
- [ ] Outbound queue so messages typed while disconnected send on reconnect
- [ ] Optimistic rendering with a pending state, and reconciliation on server acknowledgement
- [ ] Message content rendered with `textContent` — verify with an XSS payload
- [ ] Typing indicators, throttled, and presence with a timeout
- [ ] Virtualised message list that stays pinned to the bottom unless the user has scrolled up
- [ ] Unread count and a title badge when the tab is hidden
- [ ] Bounded message history in memory — an all-day session must not grow without limit

**Stretch goals**

1. File and image upload with a progress bar and a cancel button.
2. Read receipts and message editing with an edit history.
3. Voice notes recorded with `MediaRecorder`.

**Estimated time.** 25–35 hours.

---

## 17. Canvas Drawing App

**Goal.** A drawing tool with multiple brushes, layers, and complete undo/redo.

**Concepts practised.** [Ch 30 — Command Pattern](../chapters/30-design-patterns.md),
[Ch 28 — Performance](../chapters/28-performance.md),
[Ch 22 — Pointer Events](../chapters/22-events.md)

**Requirements**

- [ ] Every action a command object with `do`, `undo`, and a human-readable label
- [ ] A continuous drag produces one undoable command, not one per `pointermove`
- [ ] Bounded history (50 entries) that discards oldest and clears redo on a new action
- [ ] Tools as a Strategy map, so adding one is a single entry
- [ ] Pressure and tilt from Pointer Events where the device supports it
- [ ] Correct handling of `devicePixelRatio` so lines aren't blurry on a retina screen
- [ ] Drawing driven by `requestAnimationFrame` with coalesced pointer events
- [ ] Layers with reorder, opacity, and visibility toggles
- [ ] Export to PNG and to a replayable JSON command log

**Stretch goals**

1. Replay the drawing as an animation from the command log.
2. Flood fill with a scanline algorithm, run in a worker.
3. Collaborative drawing over WebSockets with operation ordering.

**Estimated time.** 25–35 hours.

---

## 18. State Management Library

**Goal.** A store with immutable updates, selectors, middleware, and time-travel debugging.

**Concepts practised.** [Ch 29 — Functional Programming](../chapters/29-functional-programming.md),
[Ch 30 — Design Patterns](../chapters/30-design-patterns.md),
[Ch 27 — Memory](../chapters/27-memory-garbage-collection.md)

**Requirements**

- [ ] `dispatch(action)` running a pure reducer; no state object ever mutated
- [ ] Structural sharing — prove `next.untouched === prev.untouched`
- [ ] `subscribe(selector, listener)` firing only when the selected slice changes identity
- [ ] Middleware chain built with `pipe`, including a logger and a validator
- [ ] Bounded time-travel history with `undo`, `redo`, and jump-to-state
- [ ] Action log serialisable and replayable into an identical final state
- [ ] Development-only `deepFreeze` on every state, compiled out in production
- [ ] Memoised derived selectors with a cache that can't leak
- [ ] No listener leak: every `subscribe` returns a working unsubscribe, verified with a heap snapshot

**Stretch goals**

1. A devtools panel showing the action log and state diffs.
2. Async action support with a thunk-style middleware and cancellation.
3. Persist and rehydrate selected slices, with a migration hook.

**Estimated time.** 20–30 hours.

---

## 19. Test Runner

**Goal.** A working test runner and assertion library, no dependencies.

**Concepts practised.** [Ch 32 — Testing](../chapters/32-testing.md),
[Ch 16 — Error Handling](../chapters/16-error-handling.md),
[Ch 19 — Async](../chapters/19-async-javascript.md)

**Requirements**

- [ ] `describe`/`it` with arbitrary nesting, full names in the output
- [ ] `beforeEach`/`afterEach`/`beforeAll`/`afterAll` with correct nesting order
- [ ] Matchers: `toBe`, `toEqual`, `toThrow`, `toHaveBeenCalledWith`, plus `.not` for each
- [ ] A `fn()` spy recording calls, arguments, return values, and thrown errors
- [ ] Async tests with a per-test timeout that reports which test hung
- [ ] `it.only` and `it.skip`, and a non-zero exit code on any failure
- [ ] Readable structural diffs for `toEqual` failures — the part that makes it usable
- [ ] Fake timers with `advance`, including microtask draining
- [ ] Its own test suite, run by itself

**Stretch goals**

1. Parallel execution across worker threads, with isolated module registries.
2. Coverage instrumentation via source rewriting.
3. Snapshot testing with an update flag.

**Estimated time.** 20–30 hours.

---

## 20. Mini Reactive Framework

**Goal.** A framework with fine-grained reactivity, a component model, and DOM reconciliation.

**Concepts practised.** every chapter — particularly
[Ch 5 — Proxy](../chapters/05-objects.md),
[Ch 18 — Event Loop](../chapters/18-event-loop.md),
[Ch 27 — Memory](../chapters/27-memory-garbage-collection.md),
[Ch 28 — Performance](../chapters/28-performance.md)

**Requirements**

- [ ] `signal(value)`, `computed(fn)`, and `effect(fn)` with automatic dependency tracking
- [ ] Dependencies collected at run time by a currently-executing-effect stack, not declared
- [ ] Diamond dependencies evaluated once, not twice — topological ordering, no glitches
- [ ] Updates batched into one microtask, so N writes produce one DOM update
- [ ] Components with props, local state, and lifecycle hooks
- [ ] Keyed list reconciliation that moves nodes rather than recreating them
- [ ] Every effect and listener disposed on unmount; verify with a heap snapshot over 1,000 mounts
- [ ] `computed` values cached and invalidated lazily — never recomputed if unread
- [ ] A template syntax (tagged template literals or hyperscript) with no build step
- [ ] Benchmarked against a real framework on a 1,000-row table, with the numbers written up

**Stretch goals**

1. Server-side rendering with hydration that doesn't re-run effects.
2. An async component boundary with suspense-style fallbacks.
3. A devtools panel visualising the dependency graph.

**Estimated time.** 40–60 hours.

---

## Choosing What to Build

**If you've finished chapters 1–17**, projects 1–8 will stretch you without breaking you.

**If you've finished the async chapters (18–20)**, add 5, 9, 10, and 12.

**Interview portfolio.** Two or three finished projects beat ten abandoned ones. Pick one from
11–14 and one from 15–20, and finish them properly — including the accessibility and error-handling
requirements, which are what a senior reviewer actually looks at.

**The hardest part of every project here is the state you didn't think about**: loading, empty,
error, offline, stale, partial, and concurrent. A project that handles the happy path is about 40%
done. Budget accordingly.

---

**Next:** [Appendix D — The JavaScript Roadmap →](D-roadmap.md)
