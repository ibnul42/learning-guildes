# Chapter N — <Title>

<!--
  Copy this file to NN-slug.md and delete every HTML comment as you go.
  The section order below is the contract. See ../../CLAUDE.md.
-->

## Learning Objectives

<!-- 3–5 bullets. Start each with an action verb. State capabilities, not topics. -->

- <Explain why …>
- <Implement … without …>
- <Choose between … and … given …>
- <Diagnose … from its symptom>

---

## N.1 <Section Title>

<!--
  Plain-language explanation FIRST — two to four sentences, no preamble.
  Then a heavily commented code block. Comment the why, not the what.
-->

<Explanation.>

```js
// Comment that explains the non-obvious part
const example = 1 + 1;
console.log(example); // 2
```

<!-- Use a comparison table wherever two things are being contrasted. -->

| | `<thing A>` | `<thing B>` |
|---|---|---|
| <axis> | <value> | <value> |
| <axis> | <value> | <value> |

> ⚠️ **Gotcha** — <a specific trap, stated concretely. Name the exact input and the exact
> surprising result.>

> 💼 **Interview** — <how this is actually asked, in the words an interviewer would use.>

---

## N.2 <Section Title>

<Explanation.>

```js
// ❌ Wrong — <why>
const bad = 0;

// ✅ Right — <why>
const good = 1;
```

---

## N.3 <Section Title>

<!-- Typically 4–6 numbered sections total before the mistakes block. -->

---

## N.4 Common Mistakes and Best Practices

<!-- 5–7 mistakes, then 2–4 best practices. One-line diagnosis each. -->

**Mistake: <short name>.** <One-line diagnosis — what goes wrong and why.>

**Mistake: <short name>.** <One-line diagnosis.>

**Mistake: <short name>.** <One-line diagnosis.>

**Mistake: <short name>.** <One-line diagnosis.>

**Mistake: <short name>.** <One-line diagnosis.>

**Best practice: <short name>.** <One-line rationale.>

**Best practice: <short name>.** <One-line rationale.>

---

## N.5 Real-World Example: <Named Thing>

<!--
  40–90 lines. Production-shaped, not a toy: handle errors, clean up, name things properly.
  Fully commented. Then ONE short paragraph on the single non-obvious design decision —
  not a walkthrough of the code the reader just read.
-->

```js
export class Thing {
  #state = null;

  constructor(options = {}) {
    this.options = { retries: 3, ...options };
  }

  async run() {
    // …
  }
}
```

<One paragraph. Explain the one decision a reader would otherwise question — why the cache is
keyed by X rather than Y, why the retry is capped, why this is a class and not a closure.>

---

## N.6 Questions

<!--
  20–28 questions, continuously numbered, no gaps.
  Mix: MCQ (5–6) → Short (5–6) → Output (4–5) → Debug (4–6) → Code (3–4) → Scenario (2–3).
-->

**1.** MCQ — <question> a) <x> b) <y> c) <z> d) <w>
<details><summary>Answer</summary>**b)** <brief reason.></details>

**2.** MCQ — <question> a) <x> b) <y> c) <z> d) <w>
<details><summary>Answer</summary>**a)** <brief reason.></details>

**3.** Short — <question>
<details><summary>Answer</summary><One or two sentences.></details>

**4.** Short — <question>
<details><summary>Answer</summary><One or two sentences.></details>

**5.** Output — What prints?
```js
console.log(typeof null);
```
<details><summary>Answer</summary>`'object'` — <reason.></details>

**6.** Debug — <A symptom described in prose, with no code. "A modal closes the instant it
opens." Ask for cause and fix.>
<details><summary>Answer</summary><Cause, then fix.></details>

**7.** Code — <Task, e.g. "Implement `once(fn)` so `fn` runs at most one time.">
<details><summary>Answer</summary>

```js
function once(fn) {
  let called = false, result;
  return (...args) => {
    if (called) return result;
    called = true;
    return (result = fn(...args));
  };
}
```
</details>

**8.** Scenario — <A judgement call with no single right answer, framed as a real situation.>
<details><summary>Answer</summary><Three to five sentences. Commit to a recommendation, then
name the condition under which the other choice wins.></details>

---

## Chapter Summary

<!-- 6 bullets. Each is a takeaway a reader could act on, not a topic name.
     "X is evaluated lazily, so Y" ✅.  "Covered X and Y" ❌. -->

- <Takeaway.>
- <Takeaway.>
- <Takeaway.>
- <Takeaway.>
- <Takeaway.>
- <Takeaway.>

---

## Practice Exercises

<!-- 5 tasks, increasing difficulty. NO solutions — that is deliberate. -->

1. <Warm-up, single concept.>
2. <Combines two concepts from this chapter.>
3. <Requires reading a doc or spec.>
4. <Requires debugging or measurement.>
5. <Open-ended, build something small and complete.>

---

## Mini Project: <Name>

<!-- Where the topic supports one. 2–4 sentence spec, then requirements. -->

<Two to four sentences describing what to build and what makes it non-trivial.>

- <Requirement.>
- <Requirement.>
- <Requirement.>
- <Requirement.>

---

**Next:** [Chapter N+1 — <Title> →](NN-slug.md)
