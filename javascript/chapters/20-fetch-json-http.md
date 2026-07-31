# Chapter 20 — Fetch, JSON, and HTTP

## Learning Objectives

- Use `fetch` correctly, including its non-obvious error semantics
- Serialize and parse JSON safely with `replacer`/`reviver`
- Handle headers, CORS, streaming, and file uploads

---

## 20.1 Fetch Basics

```js
const res = await fetch('/api/users', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', Accept: 'application/json' },
  body: JSON.stringify({ name: 'Ada' }),
  credentials: 'same-origin',   // 'include' to send cookies cross-origin
  signal: AbortSignal.timeout(5000),
  cache: 'no-store',
});
```

> ⚠️ **The single biggest fetch gotcha:** `fetch` does **not** reject on HTTP error status. A 404 or 500 resolves normally. It rejects only on network failure, CORS blocks, or abort.

```js
// ❌ Wrong — silently proceeds on a 500
const data = await fetch(url).then(r => r.json());

// ✅ Always check res.ok
async function fetchJson(url, init) {
  const res = await fetch(url, init);
  if (!res.ok) {
    const body = await res.text().catch(() => '');
    throw Object.assign(new Error(`HTTP ${res.status} ${res.statusText}`),
                        { status: res.status, body, url });
  }
  return res.status === 204 ? null : res.json();
}
```

### The Response object

```js
res.ok;          // true for 200–299
res.status;      // 404
res.headers.get('content-type');
res.redirected;

// Body can be read ONCE — pick one:
await res.json(); await res.text(); await res.blob();
await res.arrayBuffer(); await res.formData();

const copy = res.clone();   // read the body twice (e.g. log + parse)
```

Reading a body twice throws `TypeError: Body has already been consumed`.

---

## 20.2 JSON

```js
JSON.stringify(value, replacer, space);
JSON.parse(text, reviver);
```

What `stringify` does to non-JSON values:

```js
JSON.stringify({
  u: undefined,        // dropped
  f: () => {},         // dropped
  s: Symbol(),         // dropped
  n: NaN,              // → null
  i: Infinity,         // → null
  d: new Date(),       // → ISO string (via toJSON)
  b: 1n,               // ❌ TypeError — BigInt is not serializable
  m: new Map([[1,2]]), // → {}
  set: new Set([1]),   // → {}
});
```

Array holes become `null`. Circular references throw.

### `replacer` and `reviver`

```js
// Redact secrets on the way out
JSON.stringify(user, (key, value) =>
  ['password', 'token', 'ssn'].includes(key) ? '[REDACTED]' : value
);

// Whitelist fields
JSON.stringify(obj, ['id', 'name']);

// Revive dates on the way in
const ISO = /^\d{4}-\d{2}-\d{2}T[\d:.]+Z$/;
JSON.parse(text, (key, value) =>
  typeof value === 'string' && ISO.test(value) ? new Date(value) : value
);

// Custom serialization per class
class Money { constructor(c) { this.cents = c; } toJSON() { return { $money: this.cents }; } }
```

**Safe parsing:**

```js
function safeJsonParse(text, fallback = null) {
  try { return JSON.parse(text); } catch { return fallback; }
}
```

> ⚠️ Never `eval()` JSON, and never trust `JSON.parse` output's shape — validate it (Zod, Valibot, or a hand-written normalizer) before use. Also guard `__proto__` keys when merging parsed data (Chapter 31).

---

## 20.3 Headers, CORS, and Uploads

```js
const headers = new Headers({ Authorization: `Bearer ${token}` });
headers.append('X-Request-Id', crypto.randomUUID());
for (const [k, v] of headers) console.log(k, v);
```

**CORS in one paragraph:** the browser blocks cross-origin responses unless the *server* sends `Access-Control-Allow-Origin`. Requests that aren't "simple" (custom headers, methods beyond GET/POST/HEAD, JSON content-type) trigger a **preflight** `OPTIONS` request first. Cookies require `credentials: 'include'` on the client *and* `Access-Control-Allow-Credentials: true` plus an explicit (non-`*`) origin on the server. You cannot fix CORS from the client — it's a server configuration.

```js
// File upload — do NOT set Content-Type; the browser adds the multipart boundary
const form = new FormData();
form.append('file', fileInput.files[0]);
form.append('meta', JSON.stringify({ tags: ['a'] }));
await fetch('/upload', { method: 'POST', body: form });

// Streaming a response with progress
const res = await fetch(url);
const total = Number(res.headers.get('content-length')) || 0;
let received = 0;
const chunks = [];
for await (const chunk of res.body) {
  chunks.push(chunk);
  received += chunk.length;
  onProgress(total ? received / total : null);
}
const blob = new Blob(chunks);
```

---

## 20.4 Common Mistakes and Best Practices

**Mistake: not checking `res.ok`.** The default behaviour surprises everyone once.

**Mistake: setting `Content-Type` on `FormData`.** Breaks the multipart boundary.

**Mistake: reading the body twice.** Use `res.clone()`.

**Mistake: `JSON.parse` on an unchecked response.** An HTML error page throws a confusing `Unexpected token '<'`.

**Mistake: putting tokens in `localStorage`.** XSS-readable. Prefer `HttpOnly` cookies (Chapter 31).

**Best practice: one wrapper module** handling base URL, auth, timeouts, retries, and error normalization. Never call bare `fetch` from feature code.

**Best practice: send an `Idempotency-Key`** on POSTs you might retry.

---

## 20.5 Real-World Example: A Typed API Client

```js
export function createClient({ baseUrl, getToken, onUnauthorized }) {
  async function request(path, { method = 'GET', body, signal, timeoutMs = 10_000, ...rest } = {}) {
    const token = await getToken?.();
    const isForm = body instanceof FormData;

    const res = await fetch(new URL(path, baseUrl), {
      method,
      signal: signal
        ? AbortSignal.any([signal, AbortSignal.timeout(timeoutMs)])
        : AbortSignal.timeout(timeoutMs),
      headers: {
        Accept: 'application/json',
        ...(!isForm && body && { 'Content-Type': 'application/json' }),
        ...(token && { Authorization: `Bearer ${token}` }),
        'X-Request-Id': crypto.randomUUID(),
        ...rest.headers,
      },
      body: isForm ? body : body ? JSON.stringify(body) : undefined,
      ...rest,
    });

    if (res.status === 401) { onUnauthorized?.(); throw new Error('Unauthorized'); }

    if (!res.ok) {
      // Server may return JSON errors or an HTML error page — handle both
      const text = await res.text().catch(() => '');
      let detail = text;
      try { detail = JSON.parse(text).message ?? text; } catch {}
      throw Object.assign(new Error(`HTTP ${res.status}: ${detail}`), { status: res.status });
    }

    if (res.status === 204) return null;
    const type = res.headers.get('content-type') ?? '';
    return type.includes('application/json') ? res.json() : res.text();
  }

  return {
    get:  (p, o) => request(p, o),
    post: (p, body, o) => request(p, { ...o, method: 'POST', body }),
    put:  (p, body, o) => request(p, { ...o, method: 'PUT', body }),
    del:  (p, o) => request(p, { ...o, method: 'DELETE' }),
  };
}
```

Every concern — auth, timeout, content negotiation, error shape — is handled once.

---

## 20.6 Questions

**1.** MCQ — Does `fetch` reject on a 404? a) yes b) no
<details><summary>Answer</summary>**b)**</details>

**2.** MCQ — `res.ok` is true for: a) 200 only b) 200–299 c) <400 d) any response
<details><summary>Answer</summary>**b)**</details>

**3.** MCQ — `JSON.stringify(undefined)` returns: a) `'null'` b) `'undefined'` c) `undefined` d) throws
<details><summary>Answer</summary>**c)** The value `undefined`, not a string.</details>

**4.** MCQ — `JSON.stringify(1n)`: a) `'1'` b) `'1n'` c) TypeError d) `null`
<details><summary>Answer</summary>**c)**</details>

**5.** MCQ — Reading `res.json()` twice: a) works b) returns null c) throws TypeError d) reparses
<details><summary>Answer</summary>**c)**</details>

**6.** MCQ — CORS is enforced by: a) the server b) the browser c) the network d) DNS
<details><summary>Answer</summary>**b)** — using headers the server supplies.</details>

**7.** Short — When does `fetch` actually reject?
<details><summary>Answer</summary>Network failure, DNS failure, CORS block, or abort — never on an HTTP error status.</details>

**8.** Short — What triggers a CORS preflight?
<details><summary>Answer</summary>Non-simple requests: methods other than GET/POST/HEAD, custom headers, or a `Content-Type` outside the three simple values.</details>

**9.** Short — Why not set `Content-Type` for `FormData`?
<details><summary>Answer</summary>The browser must generate the multipart boundary; overriding it makes the body unparseable.</details>

**10.** Short — What does a `reviver` do?
<details><summary>Answer</summary>Transforms each key/value during `JSON.parse` — typically to rehydrate dates or custom classes.</details>

**11.** Short — How do you read a response body twice?
<details><summary>Answer</summary>`res.clone()` before consuming either copy.</details>

**12.** Output — `console.log(JSON.stringify({a: undefined, b: [undefined]}));`
<details><summary>Answer</summary>`'{"b":[null]}'` — dropped in objects, nulled in arrays.</details>

**13.** Output — `console.log(JSON.stringify(new Date(0)));`
<details><summary>Answer</summary>`'"1970-01-01T00:00:00.000Z"'`</details>

**14.** Output —
```js
const o = {a:1}; o.self = o;
try { JSON.stringify(o); } catch (e) { console.log(e.constructor.name); }
```
<details><summary>Answer</summary>`TypeError` — circular structure.</details>

**15.** Output — `console.log(JSON.parse('{"a":1}', (k,v) => typeof v === 'number' ? v*2 : v));`
<details><summary>Answer</summary>`{a: 2}`</details>

**16.** Debug — "Unexpected token '<' in JSON at position 0".
<details><summary>Answer</summary>The server returned HTML (an error page or a redirect to login) and you called `.json()` without checking `res.ok` or the content-type.</details>

**17.** Debug — Cookies aren't sent to the API.
<details><summary>Answer</summary>Cross-origin requests need `credentials: 'include'` client-side plus `Access-Control-Allow-Credentials: true` and a specific origin (not `*`) server-side. Also check `SameSite` on the cookie.</details>

**18.** Debug — Uploads fail with "Unexpected end of form".
<details><summary>Answer</summary>A manually set `Content-Type: multipart/form-data` without a boundary. Delete the header and let the browser set it.</details>

**19.** Debug — Aborting a fetch logs an unhandled `AbortError`.
<details><summary>Answer</summary>Expected behaviour — the promise rejects. Catch and ignore it: `catch (e) { if (e.name !== 'AbortError') throw e; }`.</details>

**20.** Code — Write `fetchWithTimeout(url, ms)` that also honours a caller's signal.
<details><summary>Answer</summary>

```js
const fetchWithTimeout = (url, ms, { signal, ...init } = {}) =>
  fetch(url, {
    ...init,
    signal: signal ? AbortSignal.any([signal, AbortSignal.timeout(ms)]) : AbortSignal.timeout(ms),
  });
```
</details>

**21.** Code — Write `safeStringify(obj)` handling circular references.
<details><summary>Answer</summary>

```js
function safeStringify(obj, space) {
  const seen = new WeakSet();
  return JSON.stringify(obj, (k, v) => {
    if (typeof v === 'object' && v !== null) {
      if (seen.has(v)) return '[Circular]';
      seen.add(v);
    }
    return typeof v === 'bigint' ? v.toString() : v;
  }, space);
}
```
</details>

**22.** Code — Write a `downloadWithProgress(url, onProgress)` returning a Blob.
<details><summary>Answer</summary>See §20.3 — read `res.body` with `for await`, accumulate chunks, compare against `content-length`.</details>

**23.** Scenario — A retried POST creates duplicate orders.
<details><summary>Answer</summary>Retrying a non-idempotent request. Generate an `Idempotency-Key` per logical operation (not per attempt) and have the server deduplicate on it. Never blindly retry POSTs without one.</details>

**24.** Scenario — Your team wants to store the auth JWT in `localStorage` for convenience.
<details><summary>Answer</summary>Push back: any XSS reads it instantly. Prefer an `HttpOnly`, `Secure`, `SameSite=Lax` cookie with a short-lived access token and a refresh endpoint. If a header is required for the API, keep the token in memory only and re-obtain it from the cookie-backed refresh endpoint on reload.</details>

---

## Chapter Summary

- `fetch` rejects only on network/CORS/abort — always check `res.ok`.
- Response bodies read once; `clone()` to read twice.
- `JSON.stringify` drops `undefined`/functions/symbols, nulls `NaN`/`Infinity`, throws on BigInt and cycles.
- Use `replacer` to redact and `reviver` to rehydrate.
- CORS is server-configured and browser-enforced; preflights come from non-simple requests.
- Never set `Content-Type` for `FormData`; centralize all HTTP in one client module.

---

## Practice Exercises

1. Build a `fetchJson` with retries, timeout, and typed error objects; test it against a mock server.
2. Write a JSON serializer preserving `Map`, `Set`, `Date`, and `BigInt` via `replacer`/`reviver`.
3. Implement a resumable chunked file uploader using `Range` headers.
4. Write a request-deduplication layer keyed by method + URL + body hash.
5. Reproduce a CORS preflight in the network tab and explain each header.

---

**Next:** [Chapter 21 — The DOM →](21-dom.md)
