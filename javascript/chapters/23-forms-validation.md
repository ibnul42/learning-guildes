# Chapter 23 — Forms and Validation

## Learning Objectives

- Read and write form values reliably
- Use the Constraint Validation API instead of hand-rolling
- Build accessible, debounced validation
- Handle submission, `FormData`, and file inputs

---

## 23.1 Reading Values

```js
const form = document.querySelector('#signup');

form.elements.email.value;        // by name attribute — most robust
form.elements['user[name]'].value;

// FormData — the standard way to collect everything
const data = new FormData(form);
data.get('email');
data.getAll('tags');              // multiple values (checkboxes, multi-select)
Object.fromEntries(data);         // ⚠️ keeps only the LAST value of repeated names

// Input types
checkbox.checked;                 // boolean, not .value
radioGroup.value;                 // via form.elements[name].value
select.value;                     // single
[...multiSelect.selectedOptions].map(o => o.value);
fileInput.files;                  // FileList
numberInput.valueAsNumber;        // number or NaN — better than Number(el.value)
dateInput.valueAsDate;            // Date or null
```

> ⚠️ `input.value` is **always a string**, even for `type="number"`. Use `valueAsNumber`.

---

## 23.2 The Constraint Validation API

HTML gives you validation for free; JavaScript lets you customise it.

```html
<input name="email" type="email" required maxlength="80">
<input name="age" type="number" min="18" max="120" required>
<input name="username" pattern="[a-z0-9_]{3,20}" required>
```

```js
input.checkValidity();      // boolean; fires an `invalid` event if false
input.reportValidity();     // same, plus shows the browser's UI bubble
input.validity;             // ValidityState: valueMissing, typeMismatch,
                            // patternMismatch, tooShort, tooLong, rangeUnderflow,
                            // rangeOverflow, stepMismatch, badInput, customError
input.validationMessage;    // localized message string

// Custom rules
input.setCustomValidity('Username is taken');   // marks invalid
input.setCustomValidity('');                     // clears — MUST do this to revalidate
```

Disable native bubbles and render your own messages:

```html
<form novalidate> ... </form>
```

```js
form.addEventListener('submit', (e) => {
  e.preventDefault();
  let firstInvalid = null;

  for (const el of form.elements) {
    if (!el.name || el.disabled) continue;
    const ok = el.checkValidity();
    showError(el, ok ? '' : messageFor(el));
    if (!ok && !firstInvalid) firstInvalid = el;
  }

  if (firstInvalid) { firstInvalid.focus(); return; }
  submit(new FormData(form));
});

function messageFor(el) {
  const v = el.validity;
  if (v.valueMissing)   return `${label(el)} is required`;
  if (v.typeMismatch)   return `Enter a valid ${el.type}`;
  if (v.tooShort)       return `At least ${el.minLength} characters`;
  if (v.rangeUnderflow) return `Must be at least ${el.min}`;
  if (v.patternMismatch) return el.title || 'Invalid format';
  return el.validationMessage;
}
```

### Accessible error display

```js
function showError(input, message) {
  const errorId = `${input.id}-error`;
  let node = document.getElementById(errorId);
  if (!node) {
    node = document.createElement('p');
    node.id = errorId;
    node.className = 'error';
    node.setAttribute('role', 'alert');       // announced by screen readers
    input.after(node);
  }
  node.textContent = message;                  // textContent, not innerHTML
  input.setAttribute('aria-invalid', message ? 'true' : 'false');
  input.setAttribute('aria-describedby', message ? errorId : '');
}
```

`aria-invalid`, `aria-describedby`, and `role="alert"` are the three attributes that make validation usable for screen-reader users. Colour alone is not sufficient.

---

## 23.3 Timing and Async Validation

Validating on every keystroke is hostile. The conventional pattern:

- **On `blur`** — first validation of a field
- **On `input`** — only *after* the field has already been marked invalid, so the error clears as the user fixes it
- **On `submit`** — validate everything

```js
const touched = new Set();

form.addEventListener('blur', (e) => {
  if (!e.target.name) return;
  touched.add(e.target.name);
  validateField(e.target);
}, true);   // capture — blur doesn't bubble

form.addEventListener('input', (e) => {
  if (touched.has(e.target.name)) validateField(e.target);
});
```

Async checks (username availability) need debouncing plus cancellation:

```js
let controller;
const checkUsername = debounce(async (input) => {
  controller?.abort();
  controller = new AbortController();
  try {
    const { available } = await fetchJson(
      `/api/username?u=${encodeURIComponent(input.value)}`,
      { signal: controller.signal }
    );
    input.setCustomValidity(available ? '' : 'Username is taken');
    showError(input, input.validationMessage);
  } catch (e) {
    if (e.name !== 'AbortError') showError(input, 'Could not verify — try again');
  }
}, 400);
```

---

## 23.4 Submission

```js
form.addEventListener('submit', async (e) => {
  e.preventDefault();
  const button = form.querySelector('[type=submit]');
  button.disabled = true;                       // prevent double submission

  try {
    const res = await fetch(form.action, {
      method: form.method || 'POST',
      body: new FormData(form),                 // multipart — no Content-Type header!
    });
    if (!res.ok) throw new Error(await res.text());
    form.reset();
  } catch (err) {
    showFormError(err.message);
  } finally {
    button.disabled = false;
  }
});
```

For JSON APIs:

```js
const body = JSON.stringify(Object.fromEntries(new FormData(form)));
```

---

## 23.5 Common Mistakes and Best Practices

**Mistake: trusting client validation.** It's UX, not security. Always validate on the server.

**Mistake: forgetting `setCustomValidity('')`.** The field stays permanently invalid.

**Mistake: `Object.fromEntries(formData)` with checkbox groups.** Only the last value survives; use `getAll`.

**Mistake: validating on every keystroke from the first character.** "Invalid email" while typing `a` is infuriating.

**Mistake: no `name` attributes.** `FormData` ignores unnamed fields entirely.

**Mistake: not disabling the submit button.** Double-clicks create duplicate records.

**Best practice: use `autocomplete` attributes** (`email`, `new-password`, `one-time-code`). Free UX win.

**Best practice: `inputmode` and correct `type`** so mobile keyboards match the field.

---

## 23.6 Real-World Example: A Validated Form Controller

```js
export function createForm(formEl, { schema, onSubmit }) {
  const touched = new Set();

  function validateField(el) {
    el.setCustomValidity('');                    // reset before re-checking
    const rule = schema[el.name];
    if (rule) {
      const custom = rule(el.value, formEl.elements);
      if (custom) el.setCustomValidity(custom);
    }
    const valid = el.checkValidity();
    showError(el, valid ? '' : el.validationMessage);
    return valid;
  }

  function validateAll() {
    return [...formEl.elements]
      .filter(el => el.name && !el.disabled)
      .map(validateField)
      .every(Boolean);                            // map first — validate every field
  }

  formEl.addEventListener('blur', (e) => {
    if (e.target.name) { touched.add(e.target.name); validateField(e.target); }
  }, true);

  formEl.addEventListener('input', (e) => {
    if (touched.has(e.target.name)) validateField(e.target);
  });

  formEl.addEventListener('submit', async (e) => {
    e.preventDefault();
    [...formEl.elements].forEach(el => el.name && touched.add(el.name));

    if (!validateAll()) {
      formEl.querySelector('[aria-invalid="true"]')?.focus();
      return;
    }

    const submitBtn = formEl.querySelector('[type=submit]');
    submitBtn.disabled = true;
    try {
      await onSubmit(new FormData(formEl));
    } catch (err) {
      // Map server field errors back onto inputs
      for (const [field, message] of Object.entries(err.fields ?? {})) {
        const el = formEl.elements[field];
        if (el) { el.setCustomValidity(message); showError(el, message); }
      }
      if (!err.fields) showFormError(err.message);
    } finally {
      submitBtn.disabled = false;
    }
  });

  return { validateAll, reset: () => { formEl.reset(); touched.clear(); } };
}

// Usage
createForm(document.querySelector('#signup'), {
  schema: {
    confirmPassword: (v, els) => v === els.password.value ? '' : 'Passwords do not match',
    age: (v) => Number(v) >= 18 ? '' : 'Must be 18 or older',
  },
  onSubmit: (data) => api.post('/signup', data),
});
```

Note `.map(validateField).every(Boolean)` rather than `.every(validateField)` — `every` short-circuits, so only the first invalid field would get an error message.

---

## 23.7 Questions

**1.** MCQ — `input.value` for `type="number"` is a: a) number b) string c) NaN d) null
<details><summary>Answer</summary>**b)** Use `valueAsNumber`.</details>

**2.** MCQ — Which shows the browser's validation UI? a) `checkValidity` b) `reportValidity` c) `validity` d) `setCustomValidity`
<details><summary>Answer</summary>**b)**</details>

**3.** MCQ — To clear a custom error: a) `setCustomValidity(null)` b) `setCustomValidity('')` c) `removeAttribute` d) `reset()`
<details><summary>Answer</summary>**b)**</details>

**4.** MCQ — `FormData` includes fields without a `name`: a) yes b) no
<details><summary>Answer</summary>**b)**</details>

**5.** MCQ — Which announces errors to screen readers? a) `class="error"` b) `role="alert"` c) `title` d) red text
<details><summary>Answer</summary>**b)** — with `aria-invalid` and `aria-describedby`.</details>

**6.** Short — Why is client-side validation not security?
<details><summary>Answer</summary>Anyone can bypass it with devtools or by posting directly to the endpoint. It's a UX affordance; the server is the authority.</details>

**7.** Short — Why use capture for `blur` delegation?
<details><summary>Answer</summary>`blur` doesn't bubble, so a listener on the form only sees it during the capture phase (or use `focusout`, which does bubble).</details>

**8.** Short — Problem with `Object.fromEntries(new FormData(form))`?
<details><summary>Answer</summary>Repeated names (checkbox groups, multi-selects) collapse to the last value. Use `getAll` for those fields.</details>

**9.** Short — When should a field first show an error?
<details><summary>Answer</summary>On blur (after the user has finished with it), then live on input once it's already been marked invalid.</details>

**10.** Short — Why `.map().every()` instead of `.every()` when validating?
<details><summary>Answer</summary>`every` short-circuits at the first failure, so later fields never get validated or get error messages shown.</details>

**11.** Output —
```js
const fd = new FormData();
fd.append('t','a'); fd.append('t','b');
console.log(fd.getAll('t'), Object.fromEntries(fd));
```
<details><summary>Answer</summary>`['a','b'] {t:'b'}`</details>

**12.** Output — `input.value = 'abc'; console.log(input.valueAsNumber);` (type=number)
<details><summary>Answer</summary>`NaN` — and `input.value` reads back as `''`, because invalid input isn't stored.</details>

**13.** Debug — A field stays invalid after the user fixes it.
<details><summary>Answer</summary>`setCustomValidity('')` was never called to clear the previous custom error.</details>

**14.** Debug — File upload sends an empty body.
<details><summary>Answer</summary>Either a manually set `Content-Type` header broke the multipart boundary, or the form lacks `enctype="multipart/form-data"` for a native submit.</details>

**15.** Debug — Async username check shows a stale result.
<details><summary>Answer</summary>Out-of-order responses. Abort the previous request on each new keystroke with an `AbortController`, and debounce.</details>

**16.** Debug — Screen-reader users don't hear validation errors.
<details><summary>Answer</summary>Missing `role="alert"` / `aria-live` on the error node and `aria-describedby` linking it to the input. Also ensure focus moves to the first invalid field on submit.</details>

**17.** Code — Write `serializeForm(form)` handling repeated names and checkboxes.
<details><summary>Answer</summary>

```js
function serializeForm(form) {
  const out = {};
  for (const [k, v] of new FormData(form)) {
    if (k in out) out[k] = [].concat(out[k], v);
    else out[k] = v;
  }
  // Unchecked checkboxes are absent from FormData — add them explicitly
  for (const el of form.querySelectorAll('input[type=checkbox]')) {
    if (el.name && !el.checked && !(el.name in out)) out[el.name] = false;
  }
  return out;
}
```
</details>

**18.** Code — Write a password-strength validator returning a score and message.
<details><summary>Answer</summary>

```js
function passwordStrength(pw) {
  const checks = [
    [pw.length >= 12, 'at least 12 characters'],
    [/[a-z]/.test(pw), 'a lowercase letter'],
    [/[A-Z]/.test(pw), 'an uppercase letter'],
    [/\d/.test(pw), 'a number'],
    [/[^\w\s]/.test(pw), 'a symbol'],
  ];
  const missing = checks.filter(([ok]) => !ok).map(([, m]) => m);
  return { score: checks.length - missing.length,
           message: missing.length ? `Add ${missing.join(', ')}` : '' };
}
```
Length matters far more than character-class rules; weight accordingly in real systems.
</details>

**19.** Code — Write a multi-step wizard validating only the current step before advancing.
<details><summary>Answer</summary>Group fields in `<fieldset data-step="n">`; on "Next", run `checkValidity()` over `[...currentStep.elements]`, show errors, focus the first invalid, and only then toggle visibility. Keep all steps in one `<form>` so a final `FormData(form)` captures everything.</details>

**20.** Scenario — Conversion drops 30% after adding validation.
<details><summary>Answer</summary>Likely validating too eagerly (errors while typing the first character), rejecting legitimate inputs (over-strict email/phone regexes, no international formats), or errors appearing far from the field. Validate on blur, loosen patterns, keep messages adjacent and actionable, and instrument which field causes abandonment.</details>

---

## Chapter Summary

- `input.value` is always a string; use `valueAsNumber`/`valueAsDate`.
- The Constraint Validation API (`checkValidity`, `validity`, `setCustomValidity`) does most of the work — always clear custom errors with `''`.
- `FormData` needs `name` attributes; use `getAll` for repeated fields.
- Validate on blur, then live once invalid, then everything on submit.
- Accessibility requires `aria-invalid`, `aria-describedby`, and `role="alert"`.
- Client validation is UX; the server is the authority.

---

## Practice Exercises

1. Build a fully accessible signup form with eight fields and cross-field validation.
2. Write an async email-availability check with debounce, abort, and error recovery.
3. Implement a file uploader with type/size validation, previews, and progress.
4. Build a wizard that persists partial state to `sessionStorage` and restores on reload.
5. Audit an existing form with a screen reader and fix every issue you find.

---

**Next:** [Chapter 24 — Timers and Scheduling →](24-timers-scheduling.md)
