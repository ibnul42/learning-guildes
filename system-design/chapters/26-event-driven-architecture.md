# Chapter 26 — Event-Driven Architecture

## Learning Objectives

- Distinguish events, commands, and queries, and design events that are facts with clear ownership
- Choose between event notification, event-carried state transfer, and event sourcing for an integration
- Design event envelopes and schemas that support deduplication, ordering, tracing, and evolution
- Decide when CQRS and event sourcing pay for themselves — and implement them correctly when they do
- Recognise the failure modes of event-driven systems: hidden coupling, lost intent, and untraceable flows

---

## 26.1 Events, Commands, and Queries

Chapters 23–25 built the machinery — outboxes, logs, consumers, stream processors. This chapter is about
the architectural style built on it: **event-driven architecture**, where services communicate by publishing
facts about what happened and reacting to facts published by others.

Three kinds of message are easy to confuse, and the confusion causes most design problems:

| | Command | Event | Query |
|---|---|---|---|
| Meaning | "Do this" | "This happened" | "Tell me this" |
| Naming | Imperative: `ChargeCard`, `ReserveStock` | Past tense: `CardCharged`, `StockReserved` | Question: `GetOrder` |
| Sender expects | It to be done (or refused) | Nothing — any number of listeners may react, or none | An answer |
| Recipients | Exactly one handler, which may reject it | Zero or more subscribers | One responder |
| Can be refused | Yes | No — it already happened | — |
| Owner of the schema | The handler | The publisher | The responder |

The distinction that matters most is **who decides**. A command transfers a decision to the recipient, who
may say no. An event reports a decision already made; subscribers can react, but they can't veto it. When a
service publishes `SendWelcomeEmail` as an "event", it has really issued a command to the email service while
pretending not to depend on it — the coupling is still there, just harder to see. Name events as facts about the
publisher's own domain (`UserRegistered`), and let the email service decide for itself that a new user deserves
a welcome email.

---

## 26.2 Three Ways to Use Events

The word "event-driven" covers three distinct patterns, with different coupling and different costs:

**Event notification.** The event says *something happened* and carries little more than an identifier:
`OrderPlaced { orderId }`. Subscribers that need details call back to the publisher's API. Payloads stay small
and schemas stable, but every subscriber adds query load on the publisher at exactly the moment events are
published, and the publisher must be available for subscribers to function.

**Event-carried state transfer.** The event carries the state subscribers need: `OrderPlaced { orderId,
customerId, lines, total, shippingAddress }`. Subscribers keep their own local copy of what they need and never
call back. This decouples availability — the shipping service works even when the order service is down — at the
cost of larger events, schemas that are harder to evolve, and data duplicated (deliberately) across services.

**Event sourcing.** The events *are* the publisher's source of truth: its state is computed by replaying them
(§26.5). This is an internal persistence choice, not an integration pattern, and it's often confused with the
other two.

| | Event notification | Event-carried state transfer | Event sourcing |
|---|---|---|---|
| Payload | IDs and type | The relevant state | Every state change, in full detail |
| Subscriber needs the publisher at runtime | Yes — to fetch details | No | — (internal) |
| Schema evolution | Easy | Harder — a public contract | Hardest — events are stored forever |
| Data duplication | Minimal | Deliberate, per consumer | — |
| Fits | Many consumers that need little; publisher can absorb callbacks | Consumers that must work independently | Domains needing full history and audit (§26.5) |

Most mature systems use **event-carried state transfer for integration** between services — with care about what
goes into the payload — and reserve event sourcing for the few domains where history is the product.

---

## 26.3 Designing Events

An event is a public contract that outlives the code that produces it, often by years. A few properties make
events reliable to consume:

- **A stable envelope** with metadata independent of the event's subject: a unique ID (for deduplication), a type
  and schema version, the aggregate it's about and a per-aggregate sequence number (for ordering), the time it
  occurred (event time — Chapter 25), and correlation and causation IDs (for tracing a business flow across
  services).
- **Facts about the publisher's own domain**, named in the past tense.
- **Enough data, deliberately chosen.** Include what consumers need to act without calling back, but not the
  publisher's internal representation — a public event mirroring an internal table couples every consumer to that
  table.
- **Backward-compatible evolution**: add optional fields, never repurpose or remove them; version breaking changes
  as new event types or new schema versions, and validate with a schema registry (Chapter 24).

```js
// An event envelope: the metadata every event needs so consumers can deduplicate, order,
// trace, and evolve — independent of what the event is about.
import { randomUUID } from 'node:crypto';

function envelope({ type, version = 1, aggregateId, sequence, data, causedBy = null, now = new Date() }) {
  // A lint, not a grammar check: events are named as past-tense facts, commands as imperatives
  if (!/^[A-Z][A-Za-z]*(ed|Paid|Sent|Held)$/.test(type)) {
    throw new Error(`event type "${type}" should be a past-tense fact, e.g. OrderPlaced`);
  }
  return {
    id: randomUUID(),                        // dedupe key for idempotent consumers (Chapter 23)
    type, version,                           // schema version: consumers branch or upcast on it
    aggregateId, sequence,                   // per-aggregate order: partition key + monotonic number
    occurredAt: now.toISOString(),           // event time, not processing time (Chapter 25)
    correlationId: causedBy?.correlationId ?? randomUUID(), // ties a whole business flow together
    causationId: causedBy?.id ?? null,       // the event or command that directly caused this one
    data,
  };
}

const placed = envelope({ type: 'OrderPlaced', aggregateId: 'order-77', sequence: 1, data: { total: 1200 } });
const paid = envelope({ type: 'OrderPaid', aggregateId: 'order-77', sequence: 2, data: { chargeId: 'ch-9' }, causedBy: placed });
console.log(paid.correlationId === placed.correlationId, paid.causationId === placed.id, paid.sequence); // true true 2
try { envelope({ type: 'SendEmail', aggregateId: 'x', sequence: 1, data: {} }); } catch (e) { console.log(e.message); }
// event type "SendEmail" should be a past-tense fact, e.g. OrderPlaced
```

**Domain events and integration events.** Inside a service, fine-grained domain events (`LineItemQuantityChanged`)
are useful for its own logic. Publishing all of them externally exposes internals and churns consumers. A common
approach is to keep domain events internal and publish a smaller set of coarser **integration events**
(`OrderUpdated` with the current order summary) designed as a public API — the same distinction as between internal
functions and a versioned HTTP API (Chapter 8).

> 💼 **Interview** — "Should the order service publish events or call the shipping service?" A strong answer
> separates the questions: if order placement needs shipping to *accept* something before it can succeed, that's a
> command or synchronous call (or a saga step, Chapter 23); if shipping just needs to *know*, publish
> `OrderPlaced` with event-carried state, and shipping reacts on its own schedule.

---

## 26.4 CQRS

**Command Query Responsibility Segregation (CQRS)** separates the model that handles writes from the models that
serve reads. Writes go to a **write model** designed to enforce invariants — normalised, transactional, small.
Changes flow (usually as events, via an outbox or CDC) into one or more **read models** (projections), each shaped
for a particular set of queries: a denormalised document for the order-detail page, a search index for support, a
pre-aggregated table for the dashboard.

```text
commands ──▶ write model (validates, enforces invariants) ──▶ events ──▶ projector ──▶ read model A (order page)
                                                                         └─────────▶ read model B (search index)
queries  ──────────────────────────────────────────────────────────────────────────▶ read models
```

Chapters 11, 12, and 18 already used the idea without the name: a search index fed by CDC is a read model. CQRS
makes it the organising principle, and its costs are the familiar ones of derived data:

- **Read models are eventually consistent.** A user who submits a command and immediately queries a read model may
  not see the effect. Mitigations are Chapter 13's: return the new state (or its version) in the command's response,
  have the UI apply the change optimistically, or let the query wait until the projection has reached the command's
  version.
- **Projections must be rebuildable** from the source of truth, idempotent, and monitored for lag.
- **More moving parts** — projectors, their infrastructure, and the code paths that keep them in sync.

CQRS pays off when reads and writes have genuinely different shapes or scales — a write model enforcing complex
invariants alongside several read patterns no single schema can serve well, or read traffic orders of magnitude
above write traffic. For CRUD screens where the read shape matches the write shape, it's overhead.

---

## 26.5 Event Sourcing

In **event sourcing**, a service stores the sequence of events that happened to each entity — `AccountOpened`,
`MoneyDeposited`, `MoneyWithdrawn` — as its source of truth, rather than the current state. The current state is
computed by **folding** over the events in order. A balance isn't stored; it's derived.

What it gives you:

- **A complete, immutable history**, which is the audit log by construction — valuable in finance, compliance,
  logistics, and anything where "how did it get this way?" matters.
- **Temporal queries**: the state as of any past moment is a replay up to that point.
- **New read models from old data**: a projection invented next year can be built by replaying every event since the
  beginning.
- **Natural integration**: the event stream already exists for other services to consume (through integration
  events, not raw internal ones).

What it costs:

- **Every read of current state needs a fold** — fast with snapshots, but more machinery.
- **Events are stored forever**, so their schemas can never change in place. Old events must be read by every future
  version of the code, typically by **upcasting**: translating old schema versions into the current shape as they're
  loaded, without rewriting stored data.
- **Concurrency control moves to the append.** Two commands that both read version 7 and then append must not both
  succeed, or their decisions (each based on the same balance) can violate an invariant. Appends carry an **expected
  version** and fail if the stream has moved on — optimistic concurrency, as in Chapter 15.
- **Deletion is hard.** "Erase this customer's personal data" conflicts with an immutable log. The usual technique is
  **crypto-shredding**: encrypt personal fields with a per-subject key stored separately, and delete the key.
- **Queries across entities** ("all accounts with a balance over €10,000") need projections; the event store itself
  only answers "events for stream X".

The real-world example in §26.9 implements an event-sourced account with all four mechanisms: folding, upcasting,
snapshots, and expected-version appends.

> ⚠️ **Gotcha** — **Event sourcing is not event-driven integration.** Teams sometimes adopt event sourcing to
> "publish events to other services", then expose their internal, fine-grained, forever-stored domain events as the
> public contract — and discover they can never refactor their domain model again without breaking consumers. Keep
> the event store internal; publish separately designed integration events.

---

## 26.6 How Event-Driven Systems Go Wrong

Events reduce some coupling and hide the rest. The characteristic failures:

**The distributed monolith.** Services that react to each other's events in long chains — A's event triggers B,
whose event triggers C, then D — are as coupled as one program, but without a call stack, a debugger, or a single
deploy. A change to A's event schema can break D, which A's team has never heard of. Keep chains short, and use an
orchestrated saga (Chapter 23) when a flow has many steps.

**Lost intent.** Events say what happened, not why the system is now waiting. When a choreographed flow stalls,
nothing in any one service says "order 77 is waiting for a shipment that will never come". Make long-running
processes explicit — a saga record, a process manager, a timeout per step.

**Untraceable flows.** Without correlation and causation IDs propagated through every event and into logs and traces
(Chapter 30), answering "what happened to this order?" means grepping six services' logs by timestamp. Put the IDs in
the envelope and require them.

**Temporal coupling in disguise.** A consumer that needs another event to have been processed first — a
`PaymentCaptured` before the `OrderPlaced` it refers to has reached the same service — depends on ordering across
topics, which brokers don't provide. Either key related events onto the same partition, or make consumers tolerate
out-of-order arrival (buffer, retry, or look up the missing entity).

**Event storms on change.** A bulk update — re-pricing a million products — publishes a million events, and every
consumer must absorb them at once. Rate-limit bulk producers, or publish one coarse event ("price list v42
published") that consumers handle in bulk.

**No schema governance.** Without compatibility checks, one producer's careless change crashes every consumer
(Chapter 24). Treat event schemas like public APIs, with owners, reviews, and deprecation periods.

---

## 26.7 When to Choose What

| Situation | Choice |
|---|---|
| Caller needs an answer to proceed | Synchronous call (query or command) |
| Caller needs another service to *do* something, possibly refusing | Command — synchronous, or async via a queue with a reply |
| Other services should react to something that happened | Integration events, event-carried state, via an outbox |
| Many services need the same reference data | Event-carried state transfer into local copies (or a shared read API) |
| Reads and writes have very different shapes or scales | CQRS with projections |
| History, audit, and "as of" queries are core to the domain | Event sourcing, for that domain only |
| A multi-step business process across services | An orchestrated saga, emitting events as it goes |
| Plain CRUD with simple reads | None of the above — a normal service with a database |

The most common mistake isn't choosing the wrong pattern; it's applying event-driven patterns everywhere because they
appear in one place. A system can use event sourcing for its ledger, CQRS for its catalogue search, plain CRUD for
user settings, and synchronous calls for checkout — and should.

---

## 26.8 Common Mistakes and Best Practices

**Mistake: commands disguised as events.** `SendWelcomeEmail` published as an "event" keeps the coupling and hides it;
publish `UserRegistered` and let subscribers decide.

**Mistake: public events that mirror internal tables.** Every consumer becomes coupled to the publisher's schema, and
the publisher can no longer refactor.

**Mistake: event notification with heavy callbacks.** Every subscriber querying the publisher on every event recreates
synchronous coupling and load.

**Mistake: long choreographed chains.** They behave like a distributed monolith with no visibility; orchestrate
multi-step flows.

**Mistake: event sourcing everywhere.** Its costs — folds, upcasting, deletion, projections — only pay off in domains
where history is the point.

**Mistake: appends without an expected version.** Concurrent commands both succeed on stale state and violate
invariants.

**Mistake: no correlation IDs.** Business flows across services can't be traced or debugged.

**Best practice: design integration events as public APIs.** Owned, versioned, reviewed, and backward compatible.

**Best practice: a standard envelope on every event.** ID, type, version, aggregate and sequence, event time,
correlation and causation IDs.

**Best practice: choose patterns per domain.** Event sourcing, CQRS, events, and CRUD can coexist; use each where it
pays.

---

## 26.9 Real-World Example: An Event-Sourced Account

This implements event sourcing for a bank account with the mechanisms that make it work in production: state as a
fold over events, commands that validate and then emit events, appends guarded by an expected version, upcasting of
an old event schema at read time, snapshots to bound replay, and a CQRS projection fed from the appended events.

```js
// event-store.mjs — an event-sourced account: state is a fold over its events; appends are
// guarded by an expected version; snapshots bound replay time; old event versions are upcast.

export class ConcurrencyError extends Error {}

export class EventStore {
  #streams = new Map();   // streamId → [events]; a real store is a table with UNIQUE (stream_id, version)
  #snapshots = new Map();
  #subscribers = [];
  append(streamId, events, { expectedVersion }) {
    const stream = this.#streams.get(streamId) ?? [];
    // Optimistic concurrency: the caller decided based on version N; if someone else appended
    // since, its decision may be invalid (Chapter 15's lost update, at the event level)
    if (stream.length !== expectedVersion) {
      throw new ConcurrencyError(`${streamId}: expected version ${expectedVersion}, found ${stream.length}`);
    }
    const stamped = events.map((e, i) => ({ ...e, streamId, version: expectedVersion + i + 1 }));
    this.#streams.set(streamId, [...stream, ...stamped]);
    for (const fn of this.#subscribers) stamped.forEach(fn);   // in production: an outbox or log (Chapter 23)
    return expectedVersion + stamped.length;
  }
  read(streamId, fromVersion = 0) { return (this.#streams.get(streamId) ?? []).filter((e) => e.version > fromVersion); }
  saveSnapshot(streamId, snapshot) { this.#snapshots.set(streamId, snapshot); }
  loadSnapshot(streamId) { return this.#snapshots.get(streamId) ?? null; }
  subscribe(fn) { this.#subscribers.push(fn); }
  all() { return [...this.#streams.values()].flat(); }
}

// v1 events stored amounts as decimal strings; v2 uses integer cents. Old events are never
// rewritten — they're translated when read ("upcast"), so the fold only knows the latest shape.
const upcast = (e) => (e.schema === 1 && 'amount' in e.data
  ? { ...e, schema: 2, data: { cents: Math.round(Number(e.data.amount) * 100) } } : e);

const evolve = (state, e) => {
  switch (e.type) {
    case 'AccountOpened': return { ...state, open: true, balance: 0 };
    case 'MoneyDeposited': return { ...state, balance: state.balance + e.data.cents };
    case 'MoneyWithdrawn': return { ...state, balance: state.balance - e.data.cents };
    default: return state;                                    // unknown types are ignored, not fatal
  }
};

export function loadAccount(store, id) {
  const snap = store.loadSnapshot(id);
  let state = snap?.state ?? { open: false, balance: 0 }, version = snap?.version ?? 0;
  for (const e of store.read(id, version)) { state = evolve(state, upcast(e)); version = e.version; }
  return { state, version };
}

// Commands validate against current state, then emit events — they never mutate state directly
export function withdraw(store, id, cents) {
  const { state, version } = loadAccount(store, id);
  if (!state.open) throw new Error('account not open');
  if (state.balance < cents) throw new Error('insufficient funds');
  return store.append(id, [{ type: 'MoneyWithdrawn', schema: 2, data: { cents } }], { expectedVersion: version });
}

// ── Demo ─────────────────────────────────────────────────────────────────────────────
const store = new EventStore();
const balances = new Map();                                    // a read model (CQRS projection)
store.subscribe((e) => {
  const u = upcast(e);
  if (u.type === 'MoneyDeposited') balances.set(e.streamId, (balances.get(e.streamId) ?? 0) + u.data.cents);
  if (u.type === 'MoneyWithdrawn') balances.set(e.streamId, (balances.get(e.streamId) ?? 0) - u.data.cents);
});

store.append('acc-1', [{ type: 'AccountOpened', schema: 1, data: {} },
  { type: 'MoneyDeposited', schema: 1, data: { amount: '25.50' } }], { expectedVersion: 0 }); // an old-format event
store.append('acc-1', [{ type: 'MoneyDeposited', schema: 2, data: { cents: 1_000 } }], { expectedVersion: 2 });
withdraw(store, 'acc-1', 500);
console.log(loadAccount(store, 'acc-1'), '| projection:', balances.get('acc-1'));
// { state: { open: true, balance: 3050 }, version: 4 } | projection: 3050

// Two concurrent withdrawals decided on the same version: the second append is rejected
const v = loadAccount(store, 'acc-1').version;
store.append('acc-1', [{ type: 'MoneyWithdrawn', schema: 2, data: { cents: 3_000 } }], { expectedVersion: v });
try { store.append('acc-1', [{ type: 'MoneyWithdrawn', schema: 2, data: { cents: 2_000 } }], { expectedVersion: v }); }
catch (e) { console.log(e.constructor.name, e.message); }
// ConcurrencyError acc-1: expected version 4, found 5

store.saveSnapshot('acc-1', loadAccount(store, 'acc-1'));       // replay starts here next time
console.log('events replayed after snapshot:', store.read('acc-1', store.loadSnapshot('acc-1').version).length);
// events replayed after snapshot: 0
```

The v1 deposit of `'25.50'` is upcast to 2,550 cents on every read, so the fold, the projection, and every future
version of the code see one schema while the stored event stays exactly as written.

The decision to defend is **checking the expected version on every append**, which forces a command that loses the
race to reload and retry instead of simply appending its event after the other one. Appending unconditionally looks
harmless — events are facts, and facts don't conflict — but the event a command emits is the result of a *decision*
made against a particular state. The second withdrawal above was approved against a balance of €30.50 that no longer
existed once the first withdrawal of €30 landed; appended blindly, it would overdraw the account, and the immutable
log would then contain a fact that should never have happened, which no later event can truly erase. The expected
version turns "these two decisions were made on the same state" into a detected conflict at write time, exactly as
Chapter 15's version column does for a row — and with a unique `(stream_id, version)` constraint in a real store, the
database enforces it even under concurrency.

---

## 26.10 Questions

**1.** MCQ — Which is an event rather than a command? a) `ReserveStock` b) `ChargeCard` c) `PaymentCaptured` d) `SendInvoice`
<details><summary>Answer</summary>**c)** A past-tense fact about something that has happened; the others are requests that a handler may refuse.</details>

**2.** MCQ — What's the main benefit of event-carried state transfer over event notification? a) Smaller events b) Subscribers don't need to call the publisher at runtime c) Easier schema evolution d) Strong consistency
<details><summary>Answer</summary>**b)** Subscribers keep local copies from the events, so they work even when the publisher is unavailable.</details>

**3.** MCQ — In CQRS, what is a projection? a) The write model b) A read model derived from changes to the write model c) A snapshot of an event stream d) A schema registry
<details><summary>Answer</summary>**b)** A query-shaped view kept up to date from events or CDC.</details>

**4.** MCQ — How does an event-sourced system handle a schema change to stored events? a) Rewrites all events b) Upcasts old events to the new shape when reading c) Deletes old events d) Stops reading old events
<details><summary>Answer</summary>**b)** Stored events are immutable; translation happens at load time.</details>

**5.** MCQ — What does an expected-version check on append prevent? a) Duplicate event IDs b) Two commands appending decisions made against the same, now stale, state c) Slow reads d) Schema drift
<details><summary>Answer</summary>**b)** It's optimistic concurrency at the event level.</details>

**6.** MCQ — Which technique lets an event-sourced system "delete" personal data? a) Rewriting events b) Crypto-shredding: encrypt personal fields with a per-subject key and delete the key c) Snapshots d) Compaction
<details><summary>Answer</summary>**b)** The events remain, but the personal data in them becomes unreadable.</details>

**7.** Short — Why is publishing `SendWelcomeEmail` as an event a design smell?
<details><summary>Answer</summary>It's a command — it tells a specific recipient what to do — disguised as an event, so the publisher still depends on the email service's behaviour while appearing decoupled. Publishing `UserRegistered` lets the email service own the decision.</details>

**8.** Short — What's the difference between domain events and integration events?
<details><summary>Answer</summary>Domain events are fine-grained, internal facts used within a service's own logic; integration events are a smaller, coarser set designed and versioned as a public contract for other services, so internals can change without breaking consumers.</details>

**9.** Short — How can a user see their own change immediately when reads come from an eventually consistent projection?
<details><summary>Answer</summary>Return the new state or version in the command response and apply it optimistically in the UI, or have the query wait until the projection has processed that version — read-your-writes via version tokens.</details>

**10.** Short — What are correlation and causation IDs for?
<details><summary>Answer</summary>The correlation ID ties every message in one business flow together across services; the causation ID points to the message that directly caused this one. Together they let logs and traces reconstruct what happened and why.</details>

**11.** Short — Why is a long choreographed event chain called a "distributed monolith"?
<details><summary>Answer</summary>The services are as tightly coupled as one program — a change to an early event can break a late consumer — but without a single codebase, call stack, or deploy, so the coupling is invisible and debugging is hard.</details>

**12.** Output — Using §26.9's `upcast`, what does `upcast({ schema: 1, type: 'MoneyDeposited', data: { amount: '0.07' } }).data` return?
<details><summary>Answer</summary>`{ cents: 7 }` — 0.07 × 100 = 7.000000000000001, rounded to 7.</details>

**13.** Output — Using §26.9's store, a new stream `acc-2` receives `AccountOpened` and a deposit of 1,000 cents (expected version 0). Then `withdraw(store, 'acc-2', 1500)` is called. What happens?
<details><summary>Answer</summary>**It throws `insufficient funds`** and appends nothing — the command validates against the folded state (balance 1,000) before emitting any event.</details>

**14.** Output — An event-sourced account has 120,000 events. Snapshots are saved every 1,000 events. In the worst case, how many events does loading the account replay?
<details><summary>Answer</summary>**999.** The latest snapshot is at most 999 events old; loading replays only events after it.</details>

**15.** Output — What does this print?
```js
const events = [{ type: 'Opened' }, { type: 'Deposited', n: 50 }, { type: 'Withdrawn', n: 20 }, { type: 'Deposited', n: 5 }];
const balance = events.reduce((b, e) => e.type === 'Deposited' ? b + e.n : e.type === 'Withdrawn' ? b - e.n : b, 0);
console.log(balance);
```
<details><summary>Answer</summary>`35` — the current state is a fold over the events: 50 − 20 + 5.</details>

**16.** Debug — The analytics team's dashboards broke after the order team renamed a column in their orders table. The analytics service consumes `OrderUpdated` events. What's the design flaw?
<details><summary>Answer</summary>The public event mirrors the internal table, so an internal refactor changed the contract. Publish integration events with a deliberately designed schema, decoupled from storage, and validate changes against a schema registry.</details>

**17.** Debug — Two support agents issued refunds on the same order simultaneously, and both refunds were recorded in the event-sourced order, refunding more than was paid. Each command checked "refundable amount ≥ requested" first. What's missing?
<details><summary>Answer</summary>The appends don't check the expected version, so both commands validated against the same state and both appended. Require an expected version on append (enforced by a unique `(stream_id, version)` constraint) and retry the losing command after reloading.</details>

**18.** Debug — When the order service is slow, the shipping, loyalty, and email services all slow down too, though they "only consume events". Each event carries only an order ID. Why?
<details><summary>Answer</summary>Event notification with callbacks: every consumer calls the order service's API to fetch details for each event, so they depend on it at runtime and add load at the worst moment. Carry the needed state in the events, so consumers don't call back.</details>

**19.** Debug — A customer support lead asks why order 77 has been "processing" for three days. Six services handle orders via events, and no one can say which step is stuck. What would you change?
<details><summary>Answer</summary>Make the process explicit: an orchestrated saga or process manager with persisted state and per-step timeouts, so each order's stage is queryable and stalls alert. Meanwhile, ensure every event carries a correlation ID that's logged and traced, so the flow can be reconstructed.</details>

**20.** Code — Write `foldAt(events, asOf, evolve, initial)` that computes an entity's state as of a given time from events with `occurredAt` timestamps.
<details><summary>Answer</summary>

```js
function foldAt(events, asOf, evolve, initial) {
  return events
    .filter((e) => Date.parse(e.occurredAt) <= Date.parse(asOf))   // temporal query: ignore later events
    .reduce(evolve, initial);
}
const evs = [{ occurredAt: '2026-01-01', d: 10 }, { occurredAt: '2026-02-01', d: 5 }, { occurredAt: '2026-03-01', d: -7 }];
console.log(foldAt(evs, '2026-02-15', (s, e) => s + e.d, 0)); // 15
```
This uses event time; if events can be corrected retroactively, you also need the time each event was *recorded* (bitemporal data).</details>

**21.** Code — Write `rebuildProjection(store, project)` that resets a read model and replays every stored event through `project`, returning the number processed.
<details><summary>Answer</summary>

```js
function rebuildProjection(store, project, readModel = new Map()) {
  readModel.clear();                                   // a rebuild starts from nothing
  const events = store.all().sort((a, b) => (a.streamId === b.streamId ? a.version - b.version : 0));
  for (const e of events) project(readModel, e);       // projection logic must be idempotent and deterministic
  return { processed: events.length, readModel };
}
```
In production, rebuild into a new table or index and swap it in (Chapter 18's alias pattern) rather than clearing the live one.</details>

**22.** Code — Write `appendWithRetry(store, id, decide, { attempts })` where `decide(state)` returns events to append; on `ConcurrencyError`, reload and decide again.
<details><summary>Answer</summary>

```js
function appendWithRetry(store, id, decide, { attempts = 3 } = {}) {
  for (let i = 1; ; i++) {
    const { state, version } = loadAccount(store, id);   // fresh state every attempt
    const events = decide(state);                        // may throw a business error: don't retry those
    try { return store.append(id, events, { expectedVersion: version }); }
    catch (e) { if (!(e instanceof ConcurrencyError) || i >= attempts) throw e; }
  }
}
```
Re-deciding — not just re-appending — is the point: the conflicting append may have changed whether the command is still valid.</details>

**23.** Scenario — A team proposes event sourcing for their entire e-commerce platform "so we have events for everything". The platform has users, catalogue, carts, orders, payments, and settings. What do you recommend?
<details><summary>Answer</summary>Use it selectively. Payments and the ledger benefit — audit and history are the product, and temporal queries matter. Orders may benefit from an event log of status changes, but plain state plus an outbox may suffice. Catalogue, carts, user settings, and profiles are CRUD; event sourcing adds folds, upcasting, deletion pain, and projections for no gain. Get "events for everything" from transactional outboxes publishing integration events — which doesn't require event sourcing at all.</details>

**24.** Scenario — Six services need customer names, tiers, and countries. Today each calls the customer service's API on most requests, which is now a bottleneck and a single point of failure. What do you propose?
<details><summary>Answer</summary>Event-carried state transfer: the customer service publishes `CustomerUpdated` integration events (with the fields consumers need) via an outbox to a compacted topic keyed by customer ID; each consumer maintains a local copy and stops calling the API at request time. New consumers bootstrap from the compacted topic. Accept seconds of staleness for these fields, and keep any check that needs current data (e.g. account suspended before a payment) as a synchronous call or a check at the source.</details>

---

## Chapter Summary

- Commands ask a recipient to decide; events report decisions already made; queries ask for answers — and disguising
  commands as events keeps the coupling while hiding it.
- Event notification keeps payloads small but makes subscribers call back; event-carried state transfer decouples
  availability at the cost of larger, public schemas; event sourcing is an internal persistence choice.
- Design events as public contracts: a standard envelope with ID, version, aggregate sequence, event time, and
  correlation and causation IDs, with backward-compatible evolution and integration events separated from internal
  ones.
- CQRS separates an invariant-enforcing write model from query-shaped read models; it pays when read and write shapes
  differ, and its projections are eventually consistent and must be rebuildable.
- Event sourcing gives full history and temporal queries in exchange for folds, snapshots, upcasting, expected-version
  appends, and crypto-shredding for deletion — worth it only where history is the product.
- Event-driven systems fail through long choreographed chains, lost intent, and untraceable flows; keep chains short,
  orchestrate processes, propagate correlation IDs, and choose patterns per domain.

---

## Practice Exercises

1. Audit the events published by a system you know: classify each as an event or a disguised command, and as
   notification or event-carried state.
2. Define a standard event envelope for your organisation and a JSON Schema for one integration event, and add a
   compatibility check that rejects removing or retyping a field.
3. Implement §26.9's event store on PostgreSQL with a unique `(stream_id, version)` constraint, and test concurrent
   appends with 50 parallel commands.
4. Add a second, new projection to an event-sourced stream (e.g. "largest withdrawal per account") and build it by
   replaying history into a new table, then swap it in.
5. Implement crypto-shredding for one personal field in an event stream: per-subject keys in a separate store, and a
   deletion that makes the field unreadable in every stored event.

---

## Mini Project: An Event-Sourced Ledger With CQRS Read Models

Build a small accounts service whose source of truth is an event store on PostgreSQL, with commands, projections, and
integration events for other services.

- Commands (open, deposit, withdraw, transfer between accounts in one stream or via a saga) that validate against
  folded state and append with expected versions, retrying on conflicts.
- Snapshots every N events, and upcasters for at least one schema change.
- Two projections (current balances; monthly statements) built incrementally and rebuildable from scratch.
- Integration events (`AccountBalanceChanged`) published through an outbox, separate from the internal events.
- Tests proving that concurrent commands never overdraw, that projections match a full replay, and that a rebuilt
  projection equals the incrementally maintained one.

---

**Next:** [Chapter 27 — Service Boundaries and Architecture Styles →](27-service-architecture.md)
