# Chapter 8 — API Design

## Learning Objectives

- Choose between REST, gRPC, GraphQL, and asynchronous APIs for a given consumer and workload
- Use HTTP method semantics — safe, idempotent — to make APIs that survive retries and caches
- Implement idempotency keys so non-idempotent operations can be retried without duplicating effects
- Design cursor-based pagination that stays correct while data changes underneath it
- Evolve an API without breaking consumers, and retire old versions deliberately

---

## 8.1 An API Is a Contract

An API is the boundary between a system and everything that depends on it. Internally you can
refactor freely; once an API is published, every client that calls it — mobile apps that users
never update, partner integrations, other teams' services — holds you to its exact shape. That
makes API design one of the few system design decisions that is genuinely expensive to reverse.

The design work has three layers:

1. **What the resources and operations are** — the domain model as consumers see it, which
   should *not* simply mirror your database tables.
2. **How they behave under real conditions** — retries, timeouts, partial failure, concurrent
   edits, large result sets. This is where most production bugs live, and where most of this
   chapter goes.
3. **How the contract changes over time** without breaking the clients you can't update.

> 💼 **Interview** — In a design interview, the API step is usually 3–5 minutes: a handful of
> endpoints with methods, key parameters, and response shapes. The signal interviewers look for is
> not REST purity; it's whether you mention pagination for list endpoints, idempotency for
> creation endpoints, and what happens when a client retries.

---

## 8.2 Choosing a Style

| | REST over HTTP/JSON | gRPC | GraphQL | Async (events, webhooks) |
|---|---|---|---|---|
| Model | Resources + HTTP methods | Remote procedures with typed messages | A typed graph queried by the client | Messages delivered later |
| Contract | OpenAPI (optional but advisable) | `.proto` schema (required) | Schema (required) | Event schema (advisable) |
| Encoding | JSON (text) | Protobuf (binary) over HTTP/2 | JSON | Varies |
| Caching | Native HTTP caching (Chapter 6) | None built in | Hard — usually one `POST` endpoint | N/A |
| Client controls response shape | No | No | Yes | No |
| Streaming | Limited (SSE, chunked) | First-class, bidirectional | Subscriptions | Inherent |
| Browser support | Native | Needs a proxy (gRPC-Web) | Native | Webhooks need a server |
| Best for | Public APIs, CRUD-shaped domains, cacheable reads | High-volume internal service calls, strict contracts, streaming | Many client types needing different views of the same data | Work that doesn't need an immediate answer; notifying other systems |

Reasonable defaults: **REST for public and partner APIs** (universally understood, cacheable,
debuggable with `curl`), **gRPC between internal services** where volume or contract strictness
justifies the tooling, **GraphQL as an aggregation layer** in front of many services when mobile
and web clients each need different shapes, and **events** whenever the caller doesn't need to wait
(Chapter 24).

GraphQL's flexibility has costs worth naming. Because clients compose queries, a single request can
fan out into thousands of backend calls (the N+1 problem in Chapter 5 and its DataLoader fix), and
a malicious query can request deeply nested data. Production GraphQL needs query depth and cost
limits, persisted queries for known clients, and per-field authorization — server-side work that
REST pushes onto endpoint design instead.

---

## 8.3 HTTP Semantics That Make APIs Robust

HTTP methods carry two properties that infrastructure relies on:

- **Safe**: the request doesn't change server state. Caches may serve it, crawlers may follow it,
  browsers may prefetch it.
- **Idempotent**: making the request N times has the same effect as making it once. Clients,
  proxies, and load balancers may **retry** it automatically after a timeout.

| Method | Safe | Idempotent | Typical use |
|---|---|---|---|
| `GET`, `HEAD` | Yes | Yes | Read a resource or collection |
| `PUT` | No | Yes | Replace a resource at a known URL with the given representation |
| `DELETE` | No | Yes | Remove a resource (deleting twice leaves it deleted) |
| `POST` | No | **No** | Create a resource in a collection; trigger an action |
| `PATCH` | No | Not guaranteed | Partial update — idempotent only if the patch is (`set x = 5` is; `increment x` isn't) |

Idempotency describes the *effect* on the server, not the response. A second `DELETE /orders/42`
can return `404` instead of `204`; the state is the same either way.

> ⚠️ **Gotcha** — A `GET` endpoint that changes state — `GET /orders/42/cancel`, a "mark as read"
> on `GET` — will be executed by things you don't control: link-preview bots, browser prefetching,
> CDN revalidation, and automatic retries. Safe methods must actually be safe.

**Status codes** should tell the client what to do next, which mostly means distinguishing *retry*
from *don't retry*:

| Class | Meaning for the client | Examples |
|---|---|---|
| `2xx` | Done | `200` OK, `201` Created (with `Location`), `202` Accepted (async), `204` No Content |
| `4xx` | Your request is wrong; retrying unchanged won't help | `400` invalid, `401` unauthenticated, `403` forbidden, `404`, `409` conflict, `412` precondition failed, `422` unprocessable |
| `429` | Slow down; retry after `Retry-After` | Rate limited (Chapter 29) |
| `5xx` | Server-side failure; retry may succeed (if the operation is idempotent) | `500`, `502`, `503` (with `Retry-After`), `504` |

**Error bodies** should be machine-readable and consistent across the whole API. RFC 9457
("Problem Details for HTTP APIs") defines a standard shape —
`{ "type", "title", "status", "detail", "instance" }` plus extension fields — that saves inventing
one. Include a stable error code clients can branch on and a request ID for support.

### Optimistic concurrency with ETags

Two clients read the same resource, each modifies it, each writes it back; the second write
silently overwrites the first. This **lost update** is the default behaviour of naive `PUT`
endpoints. HTTP has the fix built in:

```text
GET  /documents/7            → 200, ETag: "v14"
PUT  /documents/7            If-Match: "v14"   → 200, ETag: "v15"     (first writer wins)
PUT  /documents/7            If-Match: "v14"   → 412 Precondition Failed (second writer must re-read)
```

The server compares `If-Match` against the current version — a version column in the database,
checked in the `UPDATE ... WHERE version = $expected` statement (Chapter 15) — and refuses stale
writes. It's cheap, needs no locks, and should be the default for any resource more than one actor
can edit.

---

## 8.4 Idempotency Keys

Networks fail *after* requests succeed. A client sends `POST /payments`; the server charges the
card; the response is lost to a timeout. The client can't tell "it failed" from "it worked and I
didn't hear back". If it retries, the customer may be charged twice. If it doesn't, a payment that
did fail is never completed. Neither is acceptable, and `POST` is not idempotent, so automatic
retries are unsafe.

**Idempotency keys** make a non-idempotent operation safe to retry. The client generates a unique
key per *logical operation* (a UUID, created once, before the first attempt) and sends it with every
attempt:

```text
POST /payments
Idempotency-Key: 6f1c7e1a-0b55-4d0e-9a1c-2c8f3e9b7a10
{ "amount": 5000, "currency": "EUR", "source": "card_…" }
```

The server records each key with the outcome of the first attempt and **replays that outcome** for
any later attempt carrying the same key. The protocol has more edge cases than it first appears:

| Situation | Correct behaviour |
|---|---|
| First request with this key | Claim the key atomically, run the operation, store the response |
| Retry after the first completed | Return the stored response — same status, same body (same payment ID) |
| Retry while the first is still running | Don't run it again; return `409` (with `Retry-After`) or wait for the first to finish |
| Same key, *different* request body | Client bug — return `422`; replaying would return a result for a different request |
| First attempt failed with a `5xx` or crashed | Release the key so a retry can run the operation again |
| First attempt failed with a `4xx` (e.g. validation) | Store and replay it — the request itself is wrong, and re-running won't change that |
| Key older than the retention window (e.g. 24 h) | Treat as new; document the window so clients know retries must happen within it |

Three implementation details decide whether it actually works:

1. **The claim must be atomic.** "Check whether the key exists, then insert it" is a race — two
   concurrent retries both see "absent" and both run. Use a single insert-if-absent primitive:
   `INSERT … ON CONFLICT DO NOTHING`, Redis `SET key value NX`, or a conditional put.
2. **Keys are scoped to the caller.** Two customers who happen to choose the same key must not see
   each other's responses. Store keys as `(account_id, key)`.
3. **Ideally, the key record and the side effect commit together.** If the operation writes to the
   same database as the key store, do both in one transaction, so a crash can't leave "charged but
   no key recorded" or "key recorded but not charged". When the side effect is an external call (a
   card network), pass the key *through* to the provider — mature payment APIs accept idempotency
   keys for exactly this reason — so the chain is idempotent end to end. Chapter 43 builds this for
   a payment system.

> 💼 **Interview** — "How do you prevent double charges when a client retries?" is one of the most
> common system design follow-ups. The full answer: client-generated idempotency key per logical
> operation; atomic claim; replay the stored response; reject key reuse with a different payload;
> pass the key to downstream providers; and keep keys for longer than any client will retry.

The real-world example in §8.9 implements this as reusable middleware.

---

## 8.5 Pagination

Any endpoint that returns a collection must be paginated from day one — adding it later is a
breaking change, and an unpaginated endpoint becomes an outage the day one customer has 2 million
records. There are two families of pagination, and the difference matters as data grows.

**Offset pagination** — `?limit=50&offset=10000` — is simple and allows jumping to page N. It has
two problems. The database must still walk past the first 10,000 rows to return the next 50, so
deep pages get progressively slower. And it's unstable under writes: if a row is inserted at the
top while a client pages through, every row shifts down by one, and the client sees a duplicate at
the next page boundary (or misses a row on deletion).

**Cursor (keyset) pagination** remembers *where you stopped* rather than *how far you've gone*.
The response includes an opaque cursor encoding the sort key of the last row; the next request asks
for rows strictly after it:

```js
// Opaque cursors for keyset pagination. The cursor encodes the sort key of the last row
// returned; the next page starts strictly after it. Base64url hides the structure from
// clients so it can change later — sign it (HMAC) if tampering would matter.
const encodeCursor = (row) => Buffer.from(JSON.stringify([row.createdAt, row.id])).toString('base64url');
const decodeCursor = (c) => { const [createdAt, id] = JSON.parse(Buffer.from(c, 'base64url')); return { createdAt, id }; };

function nextPageQuery({ cursor, limit = 50 }) {
  const pageSize = Math.min(Math.max(1, limit), 100); // never let clients ask for everything
  if (!cursor) {
    return { sql: 'SELECT * FROM orders WHERE customer_id = $1 ORDER BY created_at DESC, id DESC LIMIT $2', pageSize };
  }
  const { createdAt, id } = decodeCursor(cursor);
  // Row-value comparison: "strictly before (createdAt, id)" in (created_at DESC, id DESC) order.
  // The id tiebreaker makes the order total, so rows sharing a timestamp are neither skipped nor repeated.
  return {
    sql: 'SELECT * FROM orders WHERE customer_id = $1 AND (created_at, id) < ($3, $4) ORDER BY created_at DESC, id DESC LIMIT $2',
    params: [createdAt, id], pageSize,
  };
}

const c = encodeCursor({ createdAt: '2026-11-01T10:00:00Z', id: 98123 });
console.log(c);                                         // WyIyMDI2LTExLTAxVDEwOjAwOjAwWiIsOTgxMjNd
console.log(nextPageQuery({ cursor: c, limit: 1000 }).pageSize); // 100
```

With an index on `(customer_id, created_at DESC, id DESC)`, every page — the first or the
millionth — is an index seek plus 50 rows. Inserts at the top don't shift anything, because the
next page is defined by a value, not a position.

| | Offset | Cursor / keyset |
|---|---|---|
| Deep-page cost | Grows with offset (O(offset)) | Constant (index seek) |
| Stable while rows are inserted/deleted | No — duplicates and gaps | Yes |
| Jump to arbitrary page | Yes | No — only next/previous |
| Total count | Often included (and expensive) | Usually omitted or approximate |
| Good for | Small admin tables, "page 3 of 7" UIs | Feeds, APIs, exports, anything large or live |

> ⚠️ **Gotcha** — The sort order must be **total** — every row must have a unique position — or
> keyset pagination skips rows. Sorting by `created_at` alone loses rows when several share a
> timestamp at a page boundary: the cursor says "after 10:00:00.000", and the other rows at exactly
> that time are never returned. Always add a unique tiebreaker (the primary key) to both the
> `ORDER BY` and the cursor.

Two more rules: **cap the page size** server-side regardless of what the client asks, and make
cursors **opaque** (encoded, documented as meaningless) so you can change what's inside them
without breaking clients.

---

## 8.6 Evolving an API Without Breaking Clients

Clients upgrade on their own schedule — some never. The default must therefore be changes that
old clients don't notice.

| Usually safe (additive) | Breaking |
|---|---|
| Adding an endpoint | Removing or renaming an endpoint or field |
| Adding an optional request field | Making an optional field required |
| Adding a response field | Changing a field's type or format (`"42"` → `42`, date formats) |
| Adding an enum value — *if clients were told to tolerate unknown values* | Changing the meaning of an existing field or status code |
| Relaxing validation | Tightening validation (rejecting previously accepted input) |
| Adding a new optional query parameter | Changing default behaviour (sort order, page size, filter) |

Two habits make additive change work. **Servers are strict about what they promise and conservative
about what they change**: never repurpose a field. **Clients are tolerant readers**: ignore unknown
fields and handle unknown enum values with a fallback, so the server can add both. Put the second
rule in your client SDKs and your API documentation; a client that crashes on an unexpected field
turns every additive change into a breaking one.

When a breaking change is truly needed, version it:

| Versioning approach | Example | Trade-off |
|---|---|---|
| URL path | `/v2/orders` | Obvious, cache-friendly, easy routing; versions the whole API at once |
| Header | `Api-Version: 2026-11-01` | Clean URLs; allows date-based versions per account (pinned at signup) |
| Media type | `Accept: application/vnd.example.v2+json` | Fine-grained; least discoverable |
| None — evolve one version forever, with additive changes only | — | Simplest for consumers; requires discipline and occasionally ugly field names |

**Date-based versions pinned per account** — the model used by several large payment APIs — are
worth knowing: each account is pinned to the API version current when it integrated, changes ship
as new dated versions, and the server translates responses down to each account's pinned version.
Old integrations never break, and upgrading is an explicit choice. The cost is maintaining those
translation layers.

**Retiring a version** is a project, not an announcement: measure who still calls it (per client,
per endpoint), announce a date, send `Deprecation` and `Sunset` response headers, contact the
remaining heavy users directly, and consider brownouts — planned short windows where the old
version returns errors — so stragglers discover the dependency before the final shutdown.

---

## 8.7 Long-Running Work, Batches, and Webhooks

Not every operation finishes within a request's latency budget. Generating a report, transcoding a
video, or provisioning infrastructure can take minutes. Holding an HTTP request open that long
fails on every timeout between client and server (Chapter 5).

**The asynchronous operation pattern:**

```text
POST /exports                         → 202 Accepted
                                        Location: /operations/op_81f
GET  /operations/op_81f               → 200 { "status": "running", "progress": 0.4 }
GET  /operations/op_81f               → 200 { "status": "succeeded", "result": { "url": "…" } }
```

The `POST` creates a durable operation record (and enqueues the work — Chapter 24), returns
immediately, and the client polls the operation resource or receives a webhook when it completes.
The operation's status should be a small, documented state machine (`pending → running →
succeeded | failed | cancelled`), and the creating `POST` needs an idempotency key like any other.

**Batch endpoints** — `POST /orders/batch` with 100 items — cut round trips (Chapter 5), but must
define partial failure explicitly. Either the whole batch is atomic (simple, but one bad item fails
everything), or each item gets its own result in the response (`207`-style per-item statuses), and
clients must handle mixed outcomes. Document which.

**Webhooks** push events to a URL the consumer controls. Because the receiver is someone else's
server, the sender must assume it's slow, down, or wrong:

- **Sign every delivery** (an HMAC of the timestamp and body with a per-consumer secret) so
  receivers can verify it came from you; include the timestamp to stop replays.
- **Retry with exponential backoff** for hours or days on failure, then disable the endpoint and
  notify the owner.
- **Deliver at least once** and say so; include a unique event ID so receivers can deduplicate
  (Chapter 24 explains why exactly-once delivery isn't on offer).
- **Don't promise ordering**; include a sequence number or timestamp, or have receivers re-fetch
  current state from the API rather than trusting the event body.
- **Receivers should acknowledge fast** — persist the event and return `2xx`, then process
  asynchronously — so slow processing doesn't cause timeouts and redundant retries.

---

## 8.8 Common Mistakes and Best Practices

**Mistake: state changes on `GET`.** Prefetchers, bots, caches, and retries will trigger them.

**Mistake: retrying `POST` without an idempotency key.** A lost response becomes a duplicate
order, payment, or message.

**Mistake: check-then-insert idempotency.** Two concurrent retries both pass the check and both
execute; the claim must be a single atomic insert-if-absent.

**Mistake: offset pagination on large or live collections.** Deep pages get slower linearly, and
concurrent inserts produce duplicates and gaps.

**Mistake: a non-unique sort key in keyset pagination.** Rows sharing the sort value at a page
boundary are silently skipped; add the primary key as a tiebreaker.

**Mistake: last-write-wins updates on shared resources.** Concurrent editors overwrite each other;
use `ETag`/`If-Match` optimistic concurrency.

**Mistake: clients that reject unknown fields or enum values.** Every additive server change
becomes a breaking change for them.

**Best practice: paginate every collection endpoint from the first version.** Adding pagination
later breaks every client that assumed it got everything.

**Best practice: one error format across the API.** Problem Details with stable error codes and a
request ID makes errors programmable and supportable.

**Best practice: measure before deprecating.** Know exactly which clients call an old version
before announcing its sunset, and use brownouts to flush out the ones you missed.

---

## 8.9 Real-World Example: Idempotency Middleware

This wraps any handler so that requests carrying an `Idempotency-Key` header execute at most once
per key, with retries receiving the original response. It implements every row of the table in
§8.4 except end-to-end propagation, which depends on what the handler calls.

```js
// idempotency.mjs — make any non-idempotent handler safe to retry.
// Framework-agnostic: wraps (request) => Promise<{ status, body, headers? }>.
import { createHash } from 'node:crypto';

// The store needs one atomic primitive: "insert if absent". Redis SET NX, SQL INSERT …
// ON CONFLICT DO NOTHING, or a conditional put all provide it. A real store also expires
// entries — including in-progress ones, so a crashed worker's claim is eventually released.
export function memoryStore() {
  const m = new Map();
  return {
    async insertIfAbsent(key, value) { if (m.has(key)) return false; m.set(key, value); return true; },
    async get(key) { return m.get(key) ?? null; },
    async set(key, value) { m.set(key, value); },
    async delete(key) { m.delete(key); },
  };
}

const fingerprint = (req) =>
  createHash('sha256').update(`${req.method} ${req.path}\n${JSON.stringify(req.body ?? null)}`).digest('hex');

export function withIdempotency(handler, { store, ttlMs = 24 * 3_600_000, now = Date.now } = {}) {
  return async function idempotent(req) {
    const key = req.headers['idempotency-key'];
    if (!key) return handler(req); // or reject with 400 on endpoints that require a key
    // Scope keys per caller: two customers who pick the same key must not collide
    const storeKey = `idem:${req.userId}:${key}`;
    const fp = fingerprint(req);

    const claimed = await store.insertIfAbsent(storeKey, { state: 'in_progress', fp, at: now() });
    if (!claimed) {
      const prior = await store.get(storeKey);
      if (prior.fp !== fp) {
        // Same key, different request: a client bug. Replaying would return the wrong result.
        return { status: 422, body: { error: 'idempotency_key_reused' } };
      }
      if (prior.state === 'in_progress') {
        // A concurrent duplicate (an impatient retry, a hedge): don't run twice
        return { status: 409, body: { error: 'request_in_progress' }, headers: { 'Retry-After': '1' } };
      }
      if (now() - prior.at < ttlMs) return { ...prior.response, headers: { 'Idempotent-Replayed': 'true' } };
    }

    try {
      const response = await handler(req);
      if (response.status >= 500) {
        await store.delete(storeKey);  // transient failure: let a retry run the handler again
      } else {
        // 2xx and 4xx are final outcomes — replay them, validation errors included
        await store.set(storeKey, { state: 'done', fp, at: now(), response });
      }
      return response;
    } catch (err) {
      await store.delete(storeKey);    // crashed before finishing: release the key for a retry
      throw err;
    }
  };
}

// ── Demo ──────────────────────────────────────────────────────────────────────────────
let charges = 0;
const createCharge = withIdempotency(async (req) => {
  await new Promise((r) => setTimeout(r, 20));       // the slow, side-effecting part
  charges++;
  return { status: 201, body: { chargeId: `ch_${charges}`, amount: req.body.amount } };
}, { store: memoryStore() });

const req = (amount, key = 'k-123') =>
  ({ method: 'POST', path: '/charges', userId: 'u1', headers: { 'idempotency-key': key }, body: { amount } });

const [first, concurrent] = await Promise.all([createCharge(req(500)), createCharge(req(500))]);
const retry = await createCharge(req(500));
const misuse = await createCharge(req(900));
console.log(first.status, first.body.chargeId);                                   // 201 ch_1
console.log(concurrent.status, concurrent.body.error);                            // 409 request_in_progress
console.log(retry.status, retry.body.chargeId, retry.headers['Idempotent-Replayed']); // 201 ch_1 true
console.log(misuse.status, misuse.body.error);                                    // 422 idempotency_key_reused
console.log('charges executed:', charges);                                        // charges executed: 1
```

The decision worth defending is **storing and replaying the full original response rather than
simply rejecting duplicates.** A simpler design would answer every repeat with "`409` — already
processed", and it would prevent double charges just as well. But the client retried precisely
*because* it never received the first response — it doesn't know the charge ID, the order number,
or even whether the first attempt succeeded or was declined. Rejecting the duplicate leaves it
exactly as blind as before, and forces every client to build a separate "look up what happened"
path, which is usually impossible without the ID it never got. Replaying the stored response makes
a retry indistinguishable from the original request succeeding late, which is the only property
that lets clients retry blindly and correctly.

---

## 8.10 Questions

**1.** MCQ — Which method is idempotent but not safe? a) `GET` b) `POST` c) `PUT` d) `HEAD`
<details><summary>Answer</summary>**c)** `PUT` changes state, but repeating it leaves the same state. `GET` and `HEAD` are both safe and idempotent; `POST` is neither.</details>

**2.** MCQ — Which pagination approach stays correct when new rows are inserted at the top while a client pages through? a) Offset b) Page number c) Keyset cursor with a unique tiebreaker d) Random sampling
<details><summary>Answer</summary>**c)** The next page is defined by the last-seen sort value, not a position, so inserts above it don't shift results.</details>

**3.** MCQ — A request with an idempotency key arrives while an earlier request with the same key is still executing. What should the server do? a) Execute it again b) Return `409` or wait for the first to complete, without executing c) Return `500` d) Delete the key
<details><summary>Answer</summary>**b)** Executing again defeats the purpose; returning an in-progress conflict (with `Retry-After`) or waiting both prevent duplicate execution.</details>

**4.** MCQ — Which change is breaking for existing clients? a) Adding a response field b) Adding an optional query parameter c) Making an optional request field required d) Adding a new endpoint
<details><summary>Answer</summary>**c)** Old clients don't send the field, so their requests start failing validation.</details>

**5.** MCQ — What does `If-Match` with an `ETag` prevent? a) Cache misses b) Lost updates from concurrent edits c) Replay attacks d) Slow pagination
<details><summary>Answer</summary>**b)** The server rejects writes based on a stale version with `412`, so the second editor must re-read instead of overwriting.</details>

**6.** MCQ — Why is GraphQL hard to cache at a CDN compared with REST? a) It uses XML b) Queries are usually `POST`s to one endpoint, so the URL doesn't identify the response c) It doesn't support HTTP d) Responses are always private
<details><summary>Answer</summary>**b)** HTTP caching keys on method and URL; one `POST /graphql` endpoint with arbitrary bodies defeats it unless you use persisted queries over `GET`.</details>

**7.** Short — Why must an idempotency key be generated by the client rather than the server?
<details><summary>Answer</summary>The key must identify the logical operation across attempts, including attempts made after the client never heard back. Only the client knows that two requests are the same operation; a server-generated key would be in the response the client never received.</details>

**8.** Short — Why should a validation error (`4xx`) be stored and replayed for an idempotency key, while a `5xx` releases the key?
<details><summary>Answer</summary>A `4xx` is a final outcome for that exact request — re-running it will fail the same way, and replaying keeps the response consistent. A `5xx` is a transient server failure, so a retry should be allowed to execute the operation again.</details>

**9.** Short — What is a "tolerant reader", and why does it matter for API evolution?
<details><summary>Answer</summary>A client that ignores unknown fields and handles unknown enum values gracefully. It lets the server make additive changes without breaking clients; strict clients turn every addition into a breaking change.</details>

**10.** Short — Name three properties a webhook delivery system should have.
<details><summary>Answer</summary>Signed payloads with a timestamp (authenticity and replay protection); retries with exponential backoff and eventual disabling; unique event IDs so receivers can deduplicate at-least-once deliveries. Also: no ordering guarantee unless stated, and fast acknowledgement by receivers.</details>

**11.** Short — When should an API return `202 Accepted` instead of `201 Created`?
<details><summary>Answer</summary>When the work hasn't finished yet — it's been durably accepted and will complete asynchronously. The response should point to an operation resource the client can poll, or promise a webhook.</details>

**12.** Output — Using the §8.9 middleware with the same store, a client sends `POST /charges` with key `k1` and amount 500 (succeeds, `ch_1`), then the same key with amount 500 again, then key `k2` with amount 500. How many charges are executed, and which charge IDs are returned?
<details><summary>Answer</summary>**2 charges; `ch_1`, `ch_1`, `ch_2`.** The second request replays `k1`'s response; `k2` is a new logical operation.</details>

**13.** Output — What does this print?
```js
const encode = (row) => Buffer.from(JSON.stringify([row.t, row.id])).toString('base64url');
const decode = (c) => JSON.parse(Buffer.from(c, 'base64url'));
console.log(decode(encode({ t: 1700, id: 9 })));
```
<details><summary>Answer</summary>`[ 1700, 9 ]` — the cursor round-trips the sort key of the last row.</details>

**14.** Output — Rows sorted by `created_at` only (no tiebreaker). A page ends with a row at `10:00:00`; three more rows share that exact timestamp. The next page uses `WHERE created_at < '10:00:00'` (descending order). How many of those three rows are ever returned?
<details><summary>Answer</summary>**0.** The strict comparison skips every row sharing the boundary timestamp; adding `id` as a tiebreaker in both `ORDER BY` and the cursor fixes it.</details>

**15.** Output — An offset-paginated endpoint returns 20 items per page, sorted newest first. A client fetches page 1 (offset 0), then 5 new items are created, then it fetches page 2 (offset 20). How many items on page 2 did the client already see on page 1?
<details><summary>Answer</summary>**5.** The new items push everything down by five, so the last five items of the original page 1 reappear at the top of page 2.</details>

**16.** Debug — Customers occasionally see two identical orders created a second apart. Logs show the mobile app retries on timeout; the API has idempotency keys implemented as "`SELECT` the key; if not found, process and `INSERT`". Cause and fix?
<details><summary>Answer</summary>A race: two attempts run the `SELECT` before either inserts, so both process. Claim the key atomically first (`INSERT … ON CONFLICT DO NOTHING` and check the affected row count, or `SET NX`), and ideally commit the key record in the same transaction as the order.</details>

**17.** Debug — An export endpoint with `?offset=` pagination is fast for the first pages but takes 40 seconds near page 5,000, and the database CPU spikes during exports. What's happening and what do you change?
<details><summary>Answer</summary>Each deep page makes the database scan and discard all preceding rows, so cost grows linearly with offset. Switch to keyset pagination with an index on the sort key plus tiebreaker; each page becomes an index seek.</details>

**18.** Debug — Two support agents edit the same customer record; one agent's changes keep disappearing. The API uses `PUT` with the full record. Cause and fix?
<details><summary>Answer</summary>Lost updates: the second `PUT`, based on an older read, overwrites the first. Return an `ETag` (version) on reads and require `If-Match` on writes; reject stale writes with `412`, and have the UI re-fetch and show the conflict.</details>

**19.** Debug — After adding a new value `"partially_refunded"` to an order status enum, an older version of the Android app crashes on the order list screen. The API team says the change was additive. Who's right, and what's the fix?
<details><summary>Answer</summary>Both, in a way: adding an enum value is only additive if clients tolerate unknown values, and this client didn't. Short-term, map the new value to an existing one for old app versions (by version header or user agent) until they age out. Long-term, make SDKs and apps handle unknown enum values with a fallback, and document that new values may appear.</details>

**20.** Code — Write `isRetryable({ method, status, hasIdempotencyKey })` returning whether a client may automatically retry a failed request.
<details><summary>Answer</summary>

```js
function isRetryable({ method, status, hasIdempotencyKey = false }) {
  const idempotent = ['GET', 'HEAD', 'PUT', 'DELETE', 'OPTIONS'].includes(method) || hasIdempotencyKey;
  if (!idempotent) return false;                 // retrying could duplicate the side effect
  if (status === undefined) return true;         // network error or timeout: outcome unknown
  return status === 429 || status === 502 || status === 503 || status === 504;
}
console.log(isRetryable({ method: 'POST', status: 503 }), isRetryable({ method: 'POST', status: 503, hasIdempotencyKey: true })); // false true
```
Plain `500` is excluded here because it often signals a bug that a retry won't fix; some APIs retry it once.</details>

**21.** Code — Write `checkPrecondition({ ifMatch, currentVersion })` returning the HTTP status for a conditional write: `428` if no precondition was sent, `412` if it doesn't match, or `null` to proceed.
<details><summary>Answer</summary>

```js
function checkPrecondition({ ifMatch, currentVersion }) {
  if (ifMatch == null) return 428;                          // require clients to send If-Match
  const tags = ifMatch.split(',').map((t) => t.trim());
  if (tags.includes('*') || tags.includes(`"${currentVersion}"`)) return null;
  return 412;
}
console.log(checkPrecondition({ ifMatch: '"v14"', currentVersion: 'v14' }), checkPrecondition({ ifMatch: '"v13"', currentVersion: 'v14' })); // null 412
```
</details>

**22.** Code — Write `signWebhook(secret, body, timestamp)` and `verifyWebhook(secret, body, timestamp, signature, { toleranceSec, now })` using HMAC-SHA256 over `${timestamp}.${body}`, rejecting stale timestamps and comparing in constant time.
<details><summary>Answer</summary>

```js
import { createHmac, timingSafeEqual } from 'node:crypto';
const signWebhook = (secret, body, ts) => createHmac('sha256', secret).update(`${ts}.${body}`).digest('hex');

function verifyWebhook(secret, body, ts, signature, { toleranceSec = 300, now = Date.now() / 1000 } = {}) {
  if (Math.abs(now - ts) > toleranceSec) return false;      // stops replay of old captured deliveries
  const expected = Buffer.from(signWebhook(secret, body, ts), 'hex');
  const given = Buffer.from(signature, 'hex');
  return given.length === expected.length && timingSafeEqual(given, expected);
}
const ts = 1_800_000_000, sig = signWebhook('s3cret', '{"id":"evt_1"}', ts);
console.log(verifyWebhook('s3cret', '{"id":"evt_1"}', ts, sig, { now: ts + 10 }),
  verifyWebhook('s3cret', '{"id":"evt_1"}', ts, sig, { now: ts + 900 })); // true false
```
</details>

**23.** Scenario — A mobile team wants GraphQL so screens can fetch exactly what they need; the backend is 15 REST microservices. The platform team worries about performance and security. What do you recommend?
<details><summary>Answer</summary>A GraphQL gateway is a reasonable fit for many client types aggregating many services, but only with guardrails: persisted queries for first-party apps (which also enables `GET` caching), depth and cost limits, DataLoader-style batching to avoid N+1 fan-out, per-field authorization enforced by the underlying services, and per-operation latency metrics. Keep public partner APIs as REST. If there's only one client and a few screens, a backend-for-frontend REST layer is simpler.</details>

**24.** Scenario — You need to change the `amount` field in a payments API from a decimal string (`"10.50"`) to an integer of minor units (`1050`). Thousands of integrations use the current format. How do you roll this out?
<details><summary>Answer</summary>Don't change the field — that's a breaking change. Add a new field (`amount_minor`) alongside the old one and document it as preferred; new integrations use it. If removing the old field matters, ship it in a new dated API version that accounts opt into, translating responses for accounts still pinned to the old version. Measure usage of the old field per account, and only consider removal after usage is negligible, with direct outreach and brownouts first.</details>

---

## Chapter Summary

- An API is a contract that's expensive to change; design resources for consumers, and make
  behaviour under retries, concurrency, and large results explicit from the first version.
- Safe and idempotent are promises infrastructure acts on — retries, prefetches, caches — so `GET`
  must never change state and non-idempotent operations need another protection.
- Idempotency keys make `POST` retryable: a client-generated key, an atomic claim, replay of the
  original response, rejection of key reuse, and propagation to downstream providers.
- Cursor pagination with a total sort order stays fast and correct on large, changing collections;
  offset pagination degrades linearly and duplicates rows under inserts.
- `ETag` with `If-Match` prevents lost updates cheaply; it should be the default for shared,
  editable resources.
- Evolve additively with tolerant readers; when breaking changes are unavoidable, version them, and
  retire old versions by measurement, deprecation headers, outreach, and brownouts.

---

## Practice Exercises

1. Take an API you use and classify each endpoint as safe, idempotent, or neither; find one that
   should be idempotent and isn't.
2. Write the OpenAPI definition for an orders API with cursor pagination, idempotency keys on
   creation, `ETag`-based updates, and a Problem Details error schema.
3. Read a major payment provider's documentation on idempotency keys and compare its rules
   (retention window, key reuse behaviour, which errors are replayed) with §8.4.
4. Load a table with 5 million rows and measure page latency at offsets 0, 100k, and 4.9M with
   offset pagination, then with keyset pagination using an index.
5. Implement the §8.9 middleware against PostgreSQL using `INSERT … ON CONFLICT DO NOTHING`, with
   the key row written in the same transaction as the business operation, and test it with 50
   concurrent identical requests.

---

## Mini Project: A Retry-Safe Orders API

Build a small orders API that a deliberately unreliable client can hammer without producing
duplicates, lost updates, or pagination errors.

- `POST /orders` requires an `Idempotency-Key`; concurrent and sequential retries never create
  duplicates, and key reuse with a different body returns `422`.
- `GET /orders` uses keyset pagination with opaque cursors and a capped page size.
- `PATCH /orders/:id` requires `If-Match`; stale writes return `412`.
- Errors use RFC 9457 Problem Details with stable error codes.
- A test client randomly drops responses, retries with backoff, and pages through the collection
  while another process inserts orders; the test asserts no duplicates, no gaps, and no lost
  updates.

---

**Next:** [Chapter 9 — Real-Time Delivery →](09-real-time-delivery.md)
