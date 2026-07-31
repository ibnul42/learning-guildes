# Chapter 31 — Security

## Learning Objectives

- Identify the three XSS types and the sinks that enable each
- Distinguish escaping from sanitising, and apply the right one per context
- Write a CSP that actually mitigates, and explain why `unsafe-inline` defeats it
- Exploit and fix prototype pollution, and choose token storage correctly

---

## 31.1 Cross-Site Scripting

XSS is attacker-controlled JavaScript running in your origin. Once it runs, it has everything the
user has: session, `localStorage`, DOM, and the ability to make authenticated requests.

**Three types:**

| Type | Payload lives in | Example |
|---|---|---|
| **Stored** | Your database | A comment containing `<img onerror>` served to every reader |
| **Reflected** | The request | `?q=<script>` echoed into the results page |
| **DOM-based** | Never reaches the server | `location.hash` written to `innerHTML` |

**The sinks.** Every XSS needs one of these:

```js
el.innerHTML = untrusted;             // ❌ the classic
el.outerHTML = untrusted;             // ❌
document.write(untrusted);            // ❌
el.insertAdjacentHTML('afterbegin', untrusted);   // ❌
eval(untrusted);                      // ❌
new Function(untrusted);              // ❌
setTimeout(untrusted, 0);             // ❌ string form compiles code
setInterval(untrusted, 0);            // ❌
el.setAttribute('onclick', untrusted);// ❌
a.href = untrusted;                   // ❌ 'javascript:alert(1)' executes on click
el.style = untrusted;                 // ❌ historically exploitable
new Worker(untrusted);                // ❌
import(untrusted);                    // ❌
```

`<script>` tags inserted via `innerHTML` don't execute — which misleads people into thinking
`innerHTML` is safe. Event handler attributes do:

```js
el.innerHTML = '<script>alert(1)</script>';        // does NOT run
el.innerHTML = '<img src=x onerror=alert(1)>';     // runs immediately
el.innerHTML = '<svg onload=alert(1)>';            // runs
el.innerHTML = '<iframe src="javascript:alert(1)">';// runs
```

**The safe sinks** need no escaping because they can't create markup:

```js
el.textContent = untrusted;          // ✅ text, always
el.setAttribute('title', untrusted); // ✅ for non-URL, non-event attributes
input.value = untrusted;             // ✅
el.append(untrusted);                // ✅ a string appends as a text node
```

> ⚠️ **Gotcha** — `javascript:` URLs are the sink people forget. `<a href>`, `<form action>`,
> `<iframe src>`, and `window.open` all execute them. Validate the protocol:
> `new URL(href, location.origin).protocol` must be in an allowlist.

**Escaping versus sanitising:**

**Escaping** converts characters so they're rendered as data. It's context-dependent — HTML body,
attribute, URL, and JavaScript contexts each need different escaping — and there is no single
"escape" function that's correct everywhere.

```js
const escapeHtml = (s) => String(s).replace(/[&<>"'`]/g, (c) => ({
  '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;', '`': '&#96;',
}[c]));

// Attribute context needs quoting too — this is exploitable despite escaping:
el.innerHTML = `<div class=${escapeHtml(cls)}>`;   // ❌ unquoted: `a onmouseover=alert(1)`
el.innerHTML = `<div class="${escapeHtml(cls)}">`; // ✅
```

**Sanitising** parses HTML and removes what isn't allowed. You need it only when users must be able
to submit *some* markup.

```js
// ✅ Native, where supported (Chrome 105+, Firefox 152+):
el.setHTML(untrusted, { sanitizer: new Sanitizer() });

// ✅ Otherwise, a maintained library. Never a regex, never your own.
el.innerHTML = DOMPurify.sanitize(untrusted, { ALLOWED_TAGS: ['b', 'i', 'em', 'a'] });
```

> 💼 **Interview** — "How do you prevent XSS?" The complete answer has three layers: escape by
> context at every output point, sanitise (with a library) only where markup is required, and add a
> CSP as defence in depth for the sink you missed. Anyone answering with only one has one bug away
> from a breach.

---

## 31.2 Trusted Types and CSP

**Trusted Types** makes the dangerous sinks refuse strings outright, converting "did we escape
everywhere?" into a question the browser answers:

```http
Content-Security-Policy: require-trusted-types-for 'script'; trusted-types app-policy
```

```js
const policy = trustedTypes.createPolicy('app-policy', {
  createHTML: (input) => DOMPurify.sanitize(input),      // ONE audited chokepoint
  createScriptURL: (url) =>
    new URL(url, location.origin).origin === location.origin ? url : (() => { throw new Error() })(),
});

el.innerHTML = policy.createHTML(untrusted);   // ✅
el.innerHTML = untrusted;                      // ❌ TypeError — the browser blocks it
```

**Content Security Policy** is an allowlist of what may load and execute.

```http
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-r4nd0m' 'strict-dynamic';
  style-src 'self';
  img-src 'self' data: https:;
  connect-src 'self' https://api.example.com;
  object-src 'none';
  base-uri 'none';
  frame-ancestors 'none';
  form-action 'self';
  require-trusted-types-for 'script';
  report-uri /csp-report
```

| Directive | Why it matters |
|---|---|
| `script-src` | The one that stops XSS from executing |
| `object-src 'none'` | Kills Flash/plugin-based bypasses. Always set it. |
| `base-uri 'none'` | Stops `<base href>` injection redirecting every relative URL |
| `frame-ancestors` | Clickjacking; supersedes `X-Frame-Options` |
| `form-action` | Stops injected forms POSTing credentials elsewhere |
| `connect-src` | Limits where exfiltrated data can be sent |

**`unsafe-inline` defeats the entire point.** The attacker's payload *is* inline script; allowing
inline script allows the payload. Same for `unsafe-eval`. Use a per-response nonce instead:

```html
<script nonce="r4nd0m">/* allowed: nonce matches the header */</script>
<script>/* blocked: no nonce */</script>
```

The nonce must be unguessable and regenerated per response — a static nonce is `unsafe-inline` with
extra steps. `'strict-dynamic'` lets a nonce'd script load further scripts, which is what makes CSP
workable with bundlers and third-party tags.

Roll out in report-only mode first:

```http
Content-Security-Policy-Report-Only: script-src 'self'; report-uri /csp-report
```

---

## 31.3 CSRF, Clickjacking, and CORS

**CSRF** makes a victim's browser send an authenticated request they didn't intend. It works because
cookies are attached automatically.

```html
<!-- On attacker.com, while the victim is logged in to bank.com -->
<form action="https://bank.com/transfer" method="POST">
  <input name="to" value="attacker"><input name="amount" value="10000">
</form>
<script>document.forms[0].submit()</script>
```

Defences, in order:

```http
Set-Cookie: session=…; HttpOnly; Secure; SameSite=Lax; Path=/
```

`SameSite=Lax` (the modern browser default) blocks the POST above outright. Add a CSRF token for
defence in depth — a per-session random value sent in a header or hidden field and verified
server-side — because `SameSite` alone won't protect a same-site subdomain takeover. Also verify
`Origin`/`Sec-Fetch-Site` on state-changing requests, and never make a GET change state.

**Clickjacking** frames your page invisibly over the attacker's UI so the victim's clicks land on
yours.

```http
Content-Security-Policy: frame-ancestors 'none'
X-Frame-Options: DENY
```

**CORS is not authorisation** — a point people get backwards. CORS relaxes the same-origin policy
for *reading* cross-origin responses in a browser. It doesn't authenticate anything, doesn't stop a
request from being *sent*, and doesn't apply to `curl` or a server.

```js
// Server: this is a read permission, not a permission to act
res.setHeader('Access-Control-Allow-Origin', 'https://app.example.com');   // never '*' with credentials
res.setHeader('Access-Control-Allow-Credentials', 'true');
```

`Access-Control-Allow-Origin: *` combined with credentials is rejected by browsers. Reflecting the
request's `Origin` header unconditionally is equivalent to `*` and is a common misconfiguration.

---

## 31.4 Prototype Pollution

If an attacker can write to `Object.prototype`, they can add properties that appear on every object
in the program. A recursive merge over untrusted JSON is the usual entry point.

```js
// ❌ The vulnerable merge — every deep-merge tutorial has this bug
function merge(target, source) {
  for (const key in source) {
    if (typeof source[key] === 'object' && source[key] !== null) {
      target[key] ??= {};
      merge(target[key], source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}

merge({}, JSON.parse('{"__proto__": {"isAdmin": true}}'));
({}).isAdmin;                     // true  — every object in the process is now admin
```

The impact depends on what your code reads. `isAdmin` on a permissions check, `shell` on a config
object passed to `child_process`, or a polluted `then` turning any object into a thenable that
hijacks your promise chain.

**Three fixes, all of which you should apply:**

```js
// 1. Reject the dangerous keys explicitly
const FORBIDDEN = new Set(['__proto__', 'constructor', 'prototype']);
function safeMerge(target, source) {
  for (const key of Object.keys(source)) {           // Object.keys skips inherited keys
    if (FORBIDDEN.has(key)) continue;
    const value = source[key];
    if (value && typeof value === 'object' && !Array.isArray(value)) {
      if (!Object.hasOwn(target, key) || typeof target[key] !== 'object') target[key] = {};
      safeMerge(target[key], value);
    } else {
      target[key] = value;
    }
  }
  return target;
}

// 2. Use a prototype-less object as the target — there's nothing to pollute
const bag = Object.create(null);
const bag2 = { __proto__: null };
// Or use a Map, which has no property semantics at all.

// 3. Freeze the prototypes at startup — turns silent pollution into a strict-mode throw
Object.freeze(Object.prototype);
Object.freeze(Array.prototype);
Object.freeze(Function.prototype);
```

Node also offers `--disable-proto=throw`. Note that `JSON.parse` itself is safe — the `__proto__`
key in JSON becomes an ordinary own property. It's the *merge* that promotes it.

**ReDoS** is the same shape of problem in a different subsystem: user input reaching a regex with
nested quantifiers hangs the thread (Chapter 26). Bound input length, use disjoint character
classes, and never compile a user-supplied pattern on the main thread.

---

## 31.5 Tokens, Secrets, and Supply Chain

**Token storage.** `localStorage` loses to `HttpOnly` cookies, and the reason is simple: any XSS
reads `localStorage` in one line.

| | `localStorage` | `HttpOnly` cookie |
|---|---|---|
| Readable by XSS | ✅ (bad) | ❌ |
| Sent automatically | ❌ | ✅ |
| CSRF exposure | None | Needs `SameSite` + token |
| Works cross-origin API | Easily | Needs `SameSite=None; Secure` |

The common counterargument — "cookies bring CSRF" — trades a vulnerability you fully mitigate with
`SameSite=Lax` plus a CSRF token for one you cannot mitigate at all. If you must hold a token in
JavaScript, keep it in a module-scoped variable (gone on reload, not readable via `localStorage`
enumeration) with a short expiry and a refresh token in an `HttpOnly` cookie.

**Secrets never live in client code.** Anything shipped to a browser is public — minified,
obfuscated, in a `.env` consumed at build time, or fetched at runtime. All public.

```js
const API_KEY = 'sk_live_…';                          // ❌ published
const key = atob('c2tfbGl2ZV8…');                     // ❌ published, with extra steps
const res = await fetch('/api/proxy/thirdparty');      // ✅ the server holds the key
```

**Randomness.** `Math.random()` is not cryptographically secure and is predictable from a few
outputs. Never use it for tokens, IDs, passwords, or nonces.

```js
crypto.randomUUID();                                    // ✅ v4 UUID
crypto.getRandomValues(new Uint8Array(32));              // ✅ raw bytes
const token = [...crypto.getRandomValues(new Uint8Array(32))]
  .map(b => b.toString(16).padStart(2, '0')).join('');   // ✅ 256-bit hex
```

**SubtleCrypto** for hashing and signing. It's async and requires a secure context:

```js
const digest = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(data));
const hex = [...new Uint8Array(digest)].map(b => b.toString(16).padStart(2, '0')).join('');

// Comparing secrets: use a constant-time comparison, not ===
function timingSafeEqual(a, b) {
  if (a.length !== b.length) return false;
  let diff = 0;
  for (let i = 0; i < a.length; i++) diff |= a.charCodeAt(i) ^ b.charCodeAt(i);
  return diff === 0;
}
```

Never hash passwords client-side and treat the result as a password — it just becomes the password.
Password hashing is Argon2 or bcrypt, server-side, with a per-user salt.

**Supply chain.** Your dependencies run with your privileges.

- Commit the lockfile and use `npm ci` in CI so builds are reproducible.
- `npm audit` and Dependabot for known CVEs; review the diff of any lockfile change in a PR.
- Prefer fewer dependencies. Every one is a maintainer whose account could be compromised.
- Pin third-party scripts with Subresource Integrity, so a compromised CDN can't swap the file:

```html
<script src="https://cdn.example.com/lib.js"
        integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC"
        crossorigin="anonymous"></script>
```

- Disable install scripts where you can (`npm config set ignore-scripts true`), since postinstall
  hooks are the standard delivery mechanism for a malicious package.

---

## 31.6 Common Mistakes and Best Practices

**Mistake: `innerHTML` with any interpolated value.** Even a "safe-looking" number today becomes a
user-controlled string after a refactor.

**Mistake: believing `<script>` not executing makes `innerHTML` safe.** `onerror` and `onload` do
execute.

**Mistake: escaping into an unquoted attribute.** `<div class=${escaped}>` is still exploitable.
Quote your attributes.

**Mistake: a hand-written sanitiser or a regex tag-stripper.** Browsers parse HTML in ways your
regex doesn't model. Use `Sanitizer` or DOMPurify.

**Mistake: `unsafe-inline` in `script-src`.** The policy now permits exactly the thing it exists to
block.

**Mistake: assigning an unvalidated URL to `href` or `src`.** `javascript:` executes on click.

**Mistake: a recursive merge over untrusted JSON.** Prototype pollution in one line.

**Mistake: `Math.random()` for anything security-relevant.** Predictable.

**Mistake: `Access-Control-Allow-Origin` reflecting any `Origin`.** That's `*` with a disguise.

**Mistake: treating CORS as an authorisation mechanism.** It's a browser read permission; it stops
nothing else.

**Best practice: default to `textContent`.** Make `innerHTML` require a justification in review.

**Best practice: deploy CSP in report-only first, then enforce.** You'll find inline handlers you
didn't know about, and you'll find them without an outage.

**Best practice: validate on the server, always.** Client-side validation is a UX feature. The
attacker isn't using your client.

**Best practice: threat-model the feature, not the codebase.** Ask what an attacker gains from each
new input, sink, and privilege — that fits in a code review, whereas a full audit doesn't.

---

## 31.7 Real-World Example: Hardening a Comment Component

A comment renderer is the canonical XSS target: untrusted text, some permitted markup, user-supplied
links, and author-controlled display names. Here's the whole thing, defended layer by layer.

```js
// ── Layer 1: a single policy for all HTML entering the DOM ─────────────────────────
const ALLOWED_TAGS = new Set(['B', 'I', 'EM', 'STRONG', 'CODE', 'P', 'BR', 'A', 'UL', 'OL', 'LI']);
const ALLOWED_ATTRS = { A: new Set(['href']) };
const SAFE_PROTOCOLS = new Set(['http:', 'https:', 'mailto:']);

/** Resolve and vet a URL. Returns null for anything not on the protocol allowlist. */
function safeUrl(value) {
  try {
    const url = new URL(value, location.origin);
    return SAFE_PROTOCOLS.has(url.protocol) ? url.href : null;   // blocks javascript:, data:, blob:
  } catch {
    return null;                                                  // unparseable is not safe
  }
}

/**
 * Parse untrusted HTML and rebuild it from an allowlist.
 * Parsing with DOMParser (an inert document — no scripts run, no resources load) and then
 * walking the result is the only reliable approach: it uses the browser's own parser, so we
 * can't be fooled by markup our own tokeniser would misread.
 */
function sanitizeHtml(dirty) {
  const doc = new DOMParser().parseFromString(dirty, 'text/html');
  const fragment = document.createDocumentFragment();

  const clean = (source, destination) => {
    for (const node of source.childNodes) {
      if (node.nodeType === Node.TEXT_NODE) {
        destination.append(node.textContent);       // text is always safe
        continue;
      }
      if (node.nodeType !== Node.ELEMENT_NODE) continue;   // drop comments, PIs, doctypes

      if (!ALLOWED_TAGS.has(node.tagName)) {
        clean(node, destination);                   // unwrap: keep the text, discard the element
        continue;
      }

      const el = document.createElement(node.tagName);
      for (const { name, value } of node.attributes) {
        if (!ALLOWED_ATTRS[node.tagName]?.has(name)) continue;   // drops every on* handler
        if (name === 'href') {
          const href = safeUrl(value);
          if (!href) continue;
          el.setAttribute('href', href);
          el.rel = 'nofollow noopener noreferrer';   // noopener: no window.opener access
          el.target = '_blank';
        } else {
          el.setAttribute(name, value);
        }
      }
      clean(node, el);
      destination.append(el);
    }
  };

  clean(doc.body, fragment);
  return fragment;                                   // a node, not a string: no re-parse, no re-entry
}

// ── Layer 2: build the comment with safe sinks for everything not user markup ──────
export function renderComment({ author, avatarUrl, bodyHtml, createdAt, profileUrl }) {
  const article = document.createElement('article');
  article.className = 'comment';

  const img = document.createElement('img');
  const avatar = safeUrl(avatarUrl);
  img.src = avatar ?? '/img/default-avatar.svg';     // an attacker-supplied avatar can't be javascript:
  img.alt = '';                                       // decorative: empty alt, not the author's name
  img.width = 40; img.height = 40;                    // set dimensions: no layout shift
  img.loading = 'lazy';
  img.referrerPolicy = 'no-referrer';                 // don't leak our URLs to their image host

  const name = document.createElement(profileUrl && safeUrl(profileUrl) ? 'a' : 'span');
  name.className = 'author';
  name.textContent = author;                          // ✅ textContent, never innerHTML
  if (name.tagName === 'A') {
    name.href = safeUrl(profileUrl);
    name.rel = 'nofollow noopener noreferrer';
  }

  const time = document.createElement('time');
  time.dateTime = new Date(createdAt).toISOString();
  time.textContent = new Intl.DateTimeFormat(undefined, { dateStyle: 'medium' }).format(createdAt);

  const body = document.createElement('div');
  body.className = 'body';
  body.append(sanitizeHtml(bodyHtml));                // the ONLY place untrusted markup enters

  article.append(img, name, time, body);
  return article;
}

// ── Layer 3: delegate events; never inline handlers, so CSP needs no unsafe-inline ─
list.addEventListener('click', (e) => {
  const button = e.target.closest('[data-action]');
  if (!button || !list.contains(button)) return;
  actions[button.dataset.action]?.(button.dataset.id);
}, { signal: controller.signal });
```

Two decisions are worth defending. `sanitizeHtml` returns a `DocumentFragment` rather than a string,
because returning a string invites a caller to concatenate it into a template and re-parse it —
which reintroduces every problem the sanitiser just solved. Returning a node makes the safe path the
only path the API offers.

Second, unknown elements are *unwrapped* rather than dropped. Dropping `<div>Hello</div>` silently
deletes the user's text, which reads as data loss and generates support tickets; unwrapping keeps
the content and discards only the disallowed element. It's the more forgiving choice and costs
nothing in safety, because the text was never dangerous — the element was.

None of this replaces the CSP. This component is the layer that should be correct; the CSP is the
layer that saves you when it isn't.

---

## 31.8 Questions

**1.** MCQ — Which does NOT execute when assigned via `innerHTML`? a) `<img onerror>` b) `<svg onload>` c) `<script>` d) `<iframe src="javascript:">`
<details><summary>Answer</summary>**c)** Which is exactly why people wrongly conclude `innerHTML` is safe.</details>

**2.** MCQ — DOM-based XSS differs in that the payload: a) is stored b) never reaches the server c) needs a POST d) requires cookies
<details><summary>Answer</summary>**b)** It lives in the fragment or client-side state, so server logs show nothing.</details>

**3.** MCQ — `script-src 'self' 'unsafe-inline'` provides: a) strong XSS mitigation b) essentially none c) protection from stored XSS only d) CSRF protection
<details><summary>Answer</summary>**b)** The injected payload is inline script, which is now allowed.</details>

**4.** MCQ — Which cookie attribute blocks the classic CSRF form POST? a) `Secure` b) `HttpOnly` c) `SameSite=Lax` d) `Path`
<details><summary>Answer</summary>**c)**</details>

**5.** MCQ — CORS exists to: a) authorise requests b) allow reading cross-origin responses c) prevent CSRF d) encrypt traffic
<details><summary>Answer</summary>**b)**</details>

**6.** MCQ — `merge({}, JSON.parse('{"__proto__":{"x":1}}'))` with a naive recursive merge: a) throws b) does nothing c) pollutes `Object.prototype` d) creates an own `__proto__` key
<details><summary>Answer</summary>**c)** `JSON.parse` alone is safe; the merge is what promotes the key.</details>

**7.** MCQ — For a session token, use: a) `Math.random()` b) `Date.now()` c) `crypto.getRandomValues` d) a UUID from a library
<details><summary>Answer</summary>**c)** Or `crypto.randomUUID()`.</details>

**8.** Short — Escaping versus sanitising?
<details><summary>Answer</summary>Escaping converts characters so the value renders as data, and is context-specific. Sanitising parses markup and removes what isn't allowlisted. Escape when no markup is wanted; sanitise only when some is required.</details>

**9.** Short — Why is `localStorage` a poor place for auth tokens?
<details><summary>Answer</summary>Any XSS reads it in one line, and there's no mitigation for that. An `HttpOnly` cookie is unreadable by script, trading an unmitigable risk for CSRF, which `SameSite` plus a token fully handles.</details>

**10.** Short — What does `rel="noopener"` prevent?
<details><summary>Answer</summary>It stops the opened page from receiving a `window.opener` reference, which it could otherwise use to navigate your page to a phishing clone (reverse tabnabbing).</details>

**11.** Short — Why must a CSP nonce be per-response?
<details><summary>Answer</summary>An attacker who can inject script can also read a static nonce from the page and attach it, making the policy equivalent to `unsafe-inline`. A fresh unguessable nonce per response can't be known at injection time.</details>

**12.** Short — Why is client-side validation not a security control?
<details><summary>Answer</summary>The attacker doesn't use your client. It's a UX affordance; the server must revalidate everything, because requests can be crafted directly.</details>

**13.** Short — Why hash passwords server-side rather than in the browser?
<details><summary>Answer</summary>A client-side hash becomes the credential — an attacker who captures it can replay it without knowing the password. Hashing must happen where the salt and cost parameters are controlled and the output is never accepted as input.</details>

**14.** Output — Which of these run code?
```js
el.innerHTML = '<script>alert(1)</script>';
el.innerHTML = '<img src=x onerror=alert(2)>';
el.textContent = '<img src=x onerror=alert(3)>';
a.href = 'javascript:alert(4)';
```
<details><summary>Answer</summary>The second runs immediately. The fourth runs when clicked. The first is inserted but never executed; the third is displayed as literal text.</details>

**15.** Output —
```js
const o = {};
JSON.parse('{"__proto__":{"polluted":true}}');
console.log(o.polluted);
const merged = Object.assign({}, JSON.parse('{"__proto__":{"p":1}}'));
console.log(({}).p);
```
<details><summary>Answer</summary>`undefined` then `undefined` — neither `JSON.parse` nor `Object.assign` walks into `__proto__` as a prototype. Only a recursive merge that assigns through it pollutes.</details>

**16.** Output —
```js
console.log(new URL('javascript:alert(1)', location.origin).protocol);
console.log(new URL('//evil.com/x', 'https://app.com').origin);
```
<details><summary>Answer</summary>`'javascript:'` then `'https://evil.com'` — showing why a protocol allowlist is needed, and that a protocol-relative URL escapes your origin.</details>

**17.** Debug — A search page reflects the query and a pen tester reports XSS, though the developer swears they escaped it.
<details><summary>Answer</summary>The value is escaped but interpolated into an unquoted attribute, so a space plus `onmouseover=` breaks out without needing `<` or `>`. Quote the attribute, or set it with `setAttribute`.</details>

**18.** Debug — After enabling CSP, the app breaks and the console is full of "Refused to execute inline script".
<details><summary>Answer</summary>The page uses inline `<script>` blocks or `onclick=` attributes. Move them to external files and delegate events, then add a per-response nonce for the blocks that must stay inline. Deploy report-only first to enumerate them all without breaking users.</details>

**19.** Debug — A settings API lets any authenticated user become an admin by posting `{"__proto__":{"role":"admin"}}`.
<details><summary>Answer</summary>Prototype pollution via a deep merge of the request body into a defaults object, after which every object's `role` reads as `'admin'`. Reject `__proto__`/`constructor`/`prototype` keys, merge into `Object.create(null)`, freeze `Object.prototype` at boot, and check roles from an own property rather than a plain lookup.</details>

**20.** Debug — An internal API is readable from any website despite requiring a session cookie.
<details><summary>Answer</summary>The server reflects the request's `Origin` in `Access-Control-Allow-Origin` and sets `Allow-Credentials: true`, so any origin passes. Replace the reflection with a strict allowlist comparison.</details>

**21.** Debug — Users report their sessions being hijacked; the app stores its JWT in `localStorage` and there's a comment feature.
<details><summary>Answer</summary>Almost certainly stored XSS in the comment rendering, exfiltrating the token. Fix the sink (sanitise or use `textContent`), then move the token to an `HttpOnly` cookie so the next XSS isn't also an account takeover, add a CSP, and invalidate every existing session.</details>

**22.** Code — Write `safeUrl(value)` for use with `href` and `src`.
<details><summary>Answer</summary>

```js
const SAFE = new Set(['http:', 'https:', 'mailto:']);
export function safeUrl(value, base = location.origin) {
  try {
    const url = new URL(String(value), base);
    return SAFE.has(url.protocol) ? url.href : null;
  } catch { return null; }
}
```
Allowlist protocols — never blocklist, since `\tjavascript:` and `JaVaScRiPt:` both defeat a
blocklist while `new URL` normalises them for the allowlist.</details>

**23.** Code — Write a prototype-pollution-safe deep merge.
<details><summary>Answer</summary>

```js
const FORBIDDEN = new Set(['__proto__', 'constructor', 'prototype']);

export function safeMerge(target, source) {
  for (const key of Object.keys(source)) {
    if (FORBIDDEN.has(key)) continue;
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
  return target;
}
```
</details>

**24.** Code — Generate a cryptographically secure 32-byte token as URL-safe base64.
<details><summary>Answer</summary>

```js
export function secureToken(bytes = 32) {
  const buf = crypto.getRandomValues(new Uint8Array(bytes));
  return btoa(String.fromCharCode(...buf))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}
```
</details>

**25.** Scenario — Marketing wants a third-party tag manager on every page, and your CSP forbids inline script.
<details><summary>Answer</summary>Tag managers exist to inject arbitrary script, so adding one means granting whoever holds that account the same privileges as your own code — that's the actual decision, and it should be made explicitly rather than as a CSP workaround. If it goes ahead, use a nonce plus `'strict-dynamic'` so the loader can bring in its children without opening `unsafe-inline` to injected payloads, restrict `connect-src` so a compromised tag can't exfiltrate freely, and keep `object-src 'none'` and `base-uri 'none'`. Push for a server-side tagging setup if the data collection is the real requirement, and never let the tag load on pages handling payment or authentication.</details>

**26.** Scenario — A code review finds `element.innerHTML = markdownToHtml(userInput)`. The author says the Markdown converter escapes HTML.
<details><summary>Answer</summary>Two things need checking, and the claim only covers one. Markdown converters typically escape HTML in *text*, but most permit raw HTML passthrough by default, and nearly all support link syntax — so `[click](javascript:alert(1))` is the likelier vector than a raw tag. Verify the converter's options (`html: false`, and a link protocol allowlist), then sanitise the output anyway, since the sanitiser is a boundary you control and the converter's defaults can change on a minor version bump. Add a CSP so a miss in either layer isn't a breach.</details>

---

## Chapter Summary

- XSS needs a sink: `innerHTML`, `eval`, string `setTimeout`, `javascript:` URLs. Default to `textContent`.
- Escaping is context-dependent and quoting matters; sanitising needs a real parser, never a regex.
- CSP is defence in depth, and `unsafe-inline` removes the defence — use a per-response nonce with `'strict-dynamic'`.
- `SameSite=Lax` plus a CSRF token beats `localStorage` tokens, because XSS-readable storage has no mitigation.
- A recursive merge over untrusted JSON is prototype pollution; reject the three keys, use null prototypes, freeze at boot.
- CORS grants read access in browsers; it authorises nothing and stops nothing outside one.

---

## Practice Exercises

1. Build a deliberately vulnerable comment page, exploit it three different ways, then fix each vector.
2. Write a CSP for an existing app in report-only mode and enumerate every violation before enforcing.
3. Exploit the naive `merge` in §31.4 to bypass a permissions check, then apply each of the three fixes and confirm.
4. Audit a page for `javascript:` URL sinks and add a `safeUrl` allowlist at each one.
5. Set up SRI for a CDN script, then change one byte of the file and confirm the browser refuses it.

---

## Mini Project: A Security Review Checklist Tool

Build a script that scans a codebase and reports likely security issues with file and line.

- Flag every dangerous sink: `innerHTML`, `outerHTML`, `insertAdjacentHTML`, `eval`, `new Function`, string-form timers, `document.write`
- Flag assignments to `href`/`src` from a non-literal, and `target="_blank"` without `rel="noopener"`
- Detect recursive merge functions and warn if they don't guard `__proto__`
- Detect `Math.random()` in files whose names or contents suggest tokens, ids, or passwords
- Parse any CSP found in headers or meta tags and warn on `unsafe-inline`, `unsafe-eval`, missing `object-src`, and missing `base-uri`
- Support inline `// security-reviewed: <reason>` suppressions, and fail if the reason is empty
- Exit non-zero on findings, with a summary grouped by severity

---

**Next:** [Chapter 32 — Testing Fundamentals →](32-testing.md)
