# Chapter 23 — Distributed Transactions and Sagas

## Learning Objectives

- Explain why atomicity across a database and a broker, or across services, can't come from a local transaction
- Describe two-phase commit, why it blocks, and where it's nonetheless the right tool
- Implement the transactional outbox and idempotent consumers so events are never lost or double-applied
- Design sagas with compensations, and handle the isolation anomalies sagas introduce
- Explain what "exactly-once" really means and how systems achieve its effect

---

## 23.1 Atomicity Across Boundaries

Chapter 15's transactions make several writes atomic — *within one database*. Real operations often
cross boundaries:

- Save an order **and** publish an `OrderPlaced` event to a message broker.
- Debit an account in the payments service **and** create a shipment in the fulfilment service.
- Write to two databases, or to a database and a search index, a cache, or an external API.

There's no shared log across those systems, so no single commit point makes the whole operation atomic.
The naive approach — do one, then the other — is the **dual write** that Chapters 11 and 15 warned about:

```js
// ❌ Wrong — a dual write. A crash, timeout, or broker outage between the two lines leaves an
// order that no downstream service will ever hear about (or, reversed, an event for an order
// that was rolled back).
async function placeOrder(db, broker, order) {
  await db.query('INSERT INTO orders (id, customer_id, total) VALUES ($1, $2, $3)', [order.id, order.customerId, order.total]);
  await broker.publish('orders', { type: 'OrderPlaced', orderId: order.id }); // may never happen
}
```

Publishing first doesn't help — it just inverts the failure. Retrying doesn't help either, because after a
crash nothing records which half happened. This chapter covers the three real answers: make both systems
participate in one atomic protocol (**two-phase commit**), make the second write part of the first
(**the outbox**), or accept a sequence of separate transactions with explicit undo (**sagas**).

---

## 23.2 Two-Phase Commit

**Two-phase commit (2PC)** coordinates several participants so they all commit or all abort:

1. **Prepare.** The coordinator asks every participant "can you commit?". Each participant makes its changes
   durable *without* committing — writes them to its log, holds its locks — and votes yes, or votes no.
   A yes vote is a **promise**: "I will commit if told to, even if I crash and restart."
2. **Commit or abort.** If every vote is yes, the coordinator records "commit" in its own durable log, then
   tells everyone to commit. If any vote is no (or times out), it tells everyone to abort.

The weakness is the window between the two phases. A participant that has voted yes has given up the right
to decide: it can't commit (others may have voted no) and can't abort (the coordinator may have decided to
commit). If the coordinator crashes then, participants are **in doubt** — stuck, holding locks, until the
coordinator recovers:

```js
// Two-phase commit, and the case that makes it "blocking": the coordinator crashes after
// participants have voted yes, so they can neither commit nor abort on their own.
function participant(name) {
  return {
    name, state: 'working', locksHeld: true,
    prepare() { this.state = 'prepared'; return 'yes'; }, // durable promise: "I can commit if told to"
    commit() { this.state = 'committed'; this.locksHeld = false; },
    abort() { this.state = 'aborted'; this.locksHeld = false; },
  };
}
function twoPhaseCommit(participants, { crashAfterPrepare = false } = {}) {
  const votes = participants.map((p) => p.prepare());     // phase 1: everyone votes
  if (crashAfterPrepare) return 'coordinator crashed';     // …before telling anyone the decision
  const decision = votes.every((v) => v === 'yes') ? 'commit' : 'abort';
  participants.forEach((p) => p[decision]());              // phase 2: everyone obeys
  return decision;
}
const ok = [participant('orders-db'), participant('payments-db')];
console.log(twoPhaseCommit(ok), ok.map((p) => `${p.name}:${p.state}`).join(' '));
// commit orders-db:committed payments-db:committed
const stuck = [participant('orders-db'), participant('payments-db')];
console.log(twoPhaseCommit(stuck, { crashAfterPrepare: true }),
  stuck.map((p) => `${p.name}:${p.state}${p.locksHeld ? ' (holding locks)' : ''}`).join(' '));
// coordinator crashed orders-db:prepared (holding locks) payments-db:prepared (holding locks)
```

Those held locks block every other transaction that touches the same rows, so a coordinator outage becomes
an outage of every participant. Classic 2PC across heterogeneous systems (XA transactions between a database
and a message broker) has further problems: every participant must support the protocol, the slowest
participant sets the latency, and the operational tooling for resolving in-doubt transactions is poor.

**Where 2PC is the right tool** is *inside* a single distributed database. Spanner, CockroachDB, YugabyteDB,
and similar systems run 2PC across their partitions for multi-partition transactions — but each participant
and the coordinator's decision record are themselves **replicated with consensus** (Chapter 21), so a crashed
coordinator is replaced by another replica that reads the decision from the replicated log. That removes the
blocking problem, at the cost of consensus latency per transaction. Across independently operated services
and products, prefer the patterns below.

> 💼 **Interview** — "Why not just use a distributed transaction across the order and payment services?"
> Good answer: 2PC couples the services' availability (any participant or the coordinator down blocks all),
> holds locks across network round trips, and requires every participant to support it — so between services
> you use an outbox for reliable events and a saga for multi-step business operations, accepting eventual
> consistency with explicit compensation.

---

## 23.3 The Transactional Outbox

The most common cross-boundary need is "change my database *and* tell others about it". The **transactional
outbox** removes the dual write by making the event part of the local transaction: the service writes the
business change and an event row into an `outbox` table **in the same database transaction**. Either both
commit or neither does. A separate **relay** process reads committed outbox rows and publishes them to the
broker, marking them sent.

```text
request ──▶ BEGIN; INSERT INTO orders …; INSERT INTO outbox (event) …; COMMIT
                                                    │
                       relay: read unsent outbox rows in order ──▶ publish to broker ──▶ mark sent
```

The SQL side (PostgreSQL; requires a database to run):

```sql
CREATE TABLE outbox (
  id           bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- commit-ish order for the relay
  aggregate_id text        NOT NULL,     -- e.g. the order ID; becomes the message key (ordering per key)
  type         text        NOT NULL,
  payload      jsonb       NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now(),
  published_at timestamptz                -- NULL until the relay has published it
);
CREATE INDEX outbox_unpublished ON outbox (id) WHERE published_at IS NULL;

-- In the request handler: one transaction for the change AND the event
BEGIN;
INSERT INTO orders (id, customer_id, total) VALUES ($1, $2, $3);
INSERT INTO outbox (aggregate_id, type, payload) VALUES ($1, 'OrderPlaced', $4);
COMMIT;
```

And the relay (illustrative; requires a PostgreSQL client and a broker client):

```js
// outbox-relay.mjs — publish committed outbox rows, at least once, in id order.
export async function relayOnce(pool, broker, { batch = 100 } = {}) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    // SKIP LOCKED lets several relay instances share the work without double-claiming rows
    const { rows } = await client.query(
      `SELECT id, aggregate_id, type, payload FROM outbox
        WHERE published_at IS NULL ORDER BY id LIMIT $1 FOR UPDATE SKIP LOCKED`, [batch]);
    for (const r of rows) {
      // Key by aggregate so a partitioned broker keeps each order's events in order (Chapter 24)
      await broker.publish('orders', { key: r.aggregate_id, value: { id: r.id, type: r.type, ...r.payload } });
    }
    if (rows.length) {
      await client.query('UPDATE outbox SET published_at = now() WHERE id = ANY($1)', [rows.map((r) => r.id)]);
    }
    await client.query('COMMIT');
    return rows.length;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;                       // rows stay unpublished and are retried on the next pass
  } finally {
    client.release();
  }
}
```

Properties to understand:

- **At-least-once delivery.** If the relay publishes and then crashes before marking rows sent, it publishes
  them again. Consumers must be idempotent (§23.4). Including the outbox `id` in the message gives consumers a
  deduplication key.
- **Ordering.** `ORDER BY id` approximates commit order but isn't exact: a transaction that got a lower `id`
  can commit after one with a higher `id`, so the relay may see id 101 before 100 commits. If strict
  per-aggregate order matters, publish with the aggregate as the message key and include a per-aggregate
  version for consumers to check, or use the CDC variant below, which follows true commit order.
- **CDC instead of polling.** Rather than polling the table, a change-data-capture connector (Debezium is the
  common choice) can read outbox inserts from the database's replication log in exact commit order and publish
  them — lower latency and no polling load. The outbox table is then just a well-defined event schema in the
  log.
- **Cleanup.** Delete or partition-drop published rows after a retention period; an ever-growing outbox slows
  the relay's query.

---

## 23.4 Idempotent Consumers and the Inbox

At-least-once delivery means every consumer will eventually see some messages twice — after a relay crash, a
consumer crash before acknowledging, a broker failover, or a rebalance (Chapter 24). A consumer that charges a
card or increments a counter on each delivery will double-apply. The consumer side therefore needs
**idempotent processing**, and there are three ways to get it:

1. **Naturally idempotent operations.** "Set order status to shipped" applied twice is the same as once.
   Prefer state-setting operations (`SET status = 'shipped'`) to state-changing ones (`SET count = count + 1`)
   where the domain allows.
2. **Version checks.** Apply an event only if it's newer than what's stored — the external-versioning approach
   from Chapters 11 and 18: `UPDATE … WHERE id = $id AND version < $eventVersion`.
3. **An inbox (processed-messages) table** written in the *same transaction* as the consumer's effects:

```sql
CREATE TABLE inbox (message_id text PRIMARY KEY, processed_at timestamptz NOT NULL DEFAULT now());

BEGIN;
INSERT INTO inbox (message_id) VALUES ($1) ON CONFLICT DO NOTHING;
-- if that inserted 0 rows, this message was already processed: COMMIT and acknowledge
UPDATE accounts SET balance = balance - $2 WHERE id = $3;   -- the effect, only on first delivery
COMMIT;
-- acknowledge to the broker only after COMMIT
```

Because the dedupe record and the effect commit together, a crash anywhere leaves them consistent: either both
happened (the redelivery is skipped) or neither did (the redelivery applies it). The same reasoning as the
outbox, mirrored. When the consumer's effect is an external call rather than a local write, pass the message ID
through as an idempotency key (Chapter 8).

---

## 23.5 Sagas

Some business operations genuinely span services and take time: place an order (orders), reserve stock
(inventory), charge the card (payments), book a courier (shipping). No single transaction can cover them, and
holding locks across all four for seconds would be unacceptable even if one could. A **saga** models such an
operation as a **sequence of local transactions**, each in one service, where every step that can be undone has
a **compensating action** that semantically reverses it:

| Step | Action | Compensation |
|---|---|---|
| 1 | Reserve stock | Release the reservation |
| 2 | Charge card | Refund the charge |
| 3 | Book courier | Cancel the booking |
| 4 | Confirm order | — (the last step) |

If step 3 fails for a business reason (no courier serves the address), the saga runs the compensations for the
completed steps **in reverse**: refund, then release. The system ends in a consistent state — just not the one
originally intended — and every intermediate state was a committed, visible state in some service.

Sagas have their own vocabulary for designing the order of steps:

- **Compensatable steps** can be undone (a reservation, a charge that can be refunded).
- **The pivot step** is the point of no return — after it the saga must go forward (shipping the parcel, sending
  an irrevocable payout). Place steps likely to fail *before* the pivot, so failures are cheap to compensate.
- **Retriable steps** come after the pivot and must eventually succeed; they're retried until they do (send the
  confirmation email, update the loyalty points).

### Compensations are not rollbacks

A compensation is a new forward action, visible to the world, and it can't erase what others observed. A
refund appears on the customer's statement next to the charge; an email that said "your order is confirmed"
can't be unsent. Designing compensations is product work as much as engineering: what does the customer see, and
what does support need to know? Some actions have no compensation at all — which is why they belong at or after
the pivot.

### Orchestration and choreography

| | Orchestration | Choreography |
|---|---|---|
| Who drives the saga | A central orchestrator calls each service in turn | Each service reacts to the previous service's events |
| Where the flow is defined | In one place (the orchestrator's code or a workflow definition) | Implicit, spread across services' subscriptions |
| Visibility and debugging | Clear: the saga's state is one record | Hard: reconstruct the flow from events across services |
| Coupling | Orchestrator knows every participant | Services know only events, not each other |
| Fits | Multi-step business processes with compensations | Simple reactions with few steps ("on OrderPlaced, send email") |

For anything with more than two or three steps and real compensations, **orchestration is easier to get right**:
the state machine is explicit, timeouts and retries live in one place, and "where is order 77 stuck?" has an
answer. Durable workflow engines (Temporal, AWS Step Functions, and similar) are orchestrators with persistence,
retries, and timers built in. Chapter 26 returns to the trade-off in event-driven architecture.

### The isolation problem

A saga's steps commit individually, so other operations can see — and act on — its intermediate states. Stock is
reserved and the card charged, but the order may yet be compensated; another process reads "stock reserved" and
makes a decision based on it. These are the anomalies of Chapter 15 without the database to prevent them. The
countermeasures are design patterns rather than mechanisms:

- **Semantic locks:** mark records with the saga's state (`order.status = 'pending'`) so other operations know the
  data is provisional and can wait, reject, or treat it specially.
- **Commutative updates:** design steps so their order doesn't matter (increment and decrement balances rather
  than overwrite them).
- **Reread before acting:** a step re-checks the current state rather than trusting what an earlier step saw.
- **Order steps to minimise exposure:** do the step most likely to fail first, and the most externally visible
  one last.

---

## 23.6 What "Exactly Once" Really Means

Marketing material and conference talks promise **exactly-once** processing. In distributed systems, a message
**delivery** can't be exactly once — Chapter 19's two generals ensure a sender can never be sure a message wasn't
received — but a message's **effect** can be. The real guarantee is:

```text
exactly-once effect  =  at-least-once delivery  +  idempotent (or deduplicated) processing
```

Every reliable pipeline in this book achieves its exactly-once *effect* this way: idempotency keys on API calls
(Chapter 8), versioned writes to search indexes (Chapter 18), the outbox plus inbox in this chapter.

Some systems provide exactly-once *within their own boundary*. Kafka's transactions let a stream processor read
from input topics, write to output topics, and commit its consumer offsets atomically, so processing *inside
Kafka* has exactly-once effect. The guarantee ends at the boundary: when the processor calls an external API or
writes to a database, that side effect needs its own idempotency. Read "exactly-once" in any product as "exactly-
once within this system, under these conditions", and check what happens at the edges.

> ⚠️ **Gotcha** — **Acknowledging before committing loses messages; committing before acknowledging duplicates
> them.** A consumer that acknowledges a message and then crashes before applying it has lost the message; one
> that applies it and crashes before acknowledging will see it again. Since loss is usually worse than
> duplication, the right order is apply-and-commit first, acknowledge second — and make the apply idempotent so
> the duplicate is harmless.

---

## 23.7 Choosing an Approach

| Situation | Approach |
|---|---|
| Change your database and notify other systems | **Transactional outbox** (polling relay or CDC) + idempotent consumers |
| Consume events and update your database | **Inbox** or version-checked writes in the same transaction as the effect |
| Multi-step business process across services, with undo | **Saga**, orchestrated, with compensations and a pivot |
| Atomic multi-partition writes inside one database | The database's own distributed transactions (2PC over consensus) |
| Two services that "must" update atomically, constantly | Reconsider the boundary — they may belong in one service and one database (Chapter 27) |
| Call an external API exactly once | Idempotency key passed to the provider; record the outcome locally |

The last-but-one row deserves emphasis: a design that needs distributed transactions on its hot path is often
telling you the service boundary is in the wrong place. Data that must change atomically together usually belongs
together.

---

## 23.8 Common Mistakes and Best Practices

**Mistake: dual writes to a database and a broker.** Any failure between them silently loses or invents events;
use an outbox.

**Mistake: 2PC between independently operated services.** A coordinator or participant outage blocks everyone and
holds locks across the network.

**Mistake: non-idempotent consumers.** At-least-once delivery guarantees duplicates eventually; without dedupe or
idempotent operations, effects double.

**Mistake: deduplication outside the effect's transaction.** Recording "processed" separately from the effect lets
a crash lose the message or apply it twice.

**Mistake: compensations designed as afterthoughts.** A refund, a cancellation, and a customer email are product
decisions; undesigned compensations produce confusing or impossible states.

**Mistake: putting failure-prone steps after the pivot.** Once past the point of no return, failures can't be
compensated, only retried — or escalated to humans.

**Mistake: taking "exactly-once" literally.** Delivery is at-least-once; the exactly-once effect comes from
idempotency, and it ends at the system's boundary.

**Best practice: one source of truth, events derived from it.** Write the change and its event together, and let
everything downstream consume the event.

**Best practice: orchestrate multi-step sagas, persist their state.** An explicit, durable state machine makes
failures visible and recoverable.

**Best practice: let the need for distributed transactions question the boundary.** Data that must change
atomically together usually belongs in one service.

---

## 23.9 Real-World Example: A Durable Saga Orchestrator

This orchestrator runs a saga's steps in order, gives each step a stable idempotency key, persists the saga's
state after every step, distinguishes business failures (compensate) from transient ones (retry), and runs
compensations in reverse. Calling `run` again with the same saga ID resumes from the persisted state — which is
how it recovers from its own crash. The demo kills the orchestrator right after the card is charged, resumes in a
"new process", and then hits a permanent shipping failure.

```js
// saga.mjs — a durable saga orchestrator: run steps in order, persist progress after each,
// and on a business failure run the compensations of completed steps in reverse order.
// Re-running a saga ID resumes from its persisted state, so a crashed orchestrator recovers.

export class BusinessError extends Error {}   // "no, and retrying won't change that" (declined card)

export function createSagaRunner({ store, steps, maxRetries = 3, sleep = async () => {} }) {
  const retrying = async (fn) => {
    for (let attempt = 1; ; attempt++) {
      try { return await fn(); }
      catch (err) {
        if (err instanceof BusinessError || attempt > maxRetries) throw err;
        await sleep(2 ** attempt * 100);        // transient failure: back off and retry (Chapter 28)
      }
    }
  };

  return async function run(sagaId, input) {
    const state = (await store.get(sagaId)) ?? { id: sagaId, status: 'running', next: 0, ctx: input, done: [] };
    if (state.status === 'completed' || state.status === 'compensated') return state;   // idempotent

    while (state.status === 'running' && state.next < steps.length) {
      const step = steps[state.next];
      try {
        // Each step gets a stable idempotency key, so a resumed saga never repeats an effect
        const output = await retrying(() => step.action(state.ctx, `${sagaId}:${step.name}`));
        Object.assign(state.ctx, output);
        state.done.push(step.name);
        state.next++;
      } catch (err) {
        state.status = 'compensating';
        state.error = `${step.name}: ${err.message}`;
      }
      await store.put(sagaId, state);           // persist BEFORE moving on (see text)
    }

    while (state.status === 'compensating' && state.done.length) {
      const step = steps.find((s) => s.name === state.done.at(-1));
      // Compensations must eventually succeed: retry far more patiently, then page a human
      await retrying(() => step.compensate(state.ctx, `${sagaId}:${step.name}:undo`));
      state.done.pop();
      await store.put(sagaId, state);
    }
    state.status = state.status === 'running' ? 'completed' : 'compensated';
    await store.put(sagaId, state);
    return state;
  };
}

// ── Demo: the orchestrator crashes mid-saga, resumes, then a step fails for good ─────
const log = [], applied = new Set();
const once = (key, effect) => { if (!applied.has(key)) { applied.add(key); log.push(effect); } }; // idempotent services
const mem = new Map();
let puts = 0;
const store = {
  async get(k) { return mem.get(k) && structuredClone(mem.get(k)); },
  async put(k, v) {
    mem.set(k, structuredClone(v));
    if (++puts === 2) throw new Error('orchestrator process killed');   // right after chargeCard is saved
  },
};
const steps = [
  { name: 'reserveStock', action: async (c, key) => { once(key, `reserve ${c.sku}`); return { reservation: 'r-1' }; },
    compensate: async (c, key) => once(key, `release ${c.reservation}`) },
  { name: 'chargeCard', action: async (c, key) => { once(key, `charge ${c.amount}`); return { chargeId: 'ch-9' }; },
    compensate: async (c, key) => once(key, `refund ${c.chargeId}`) },
  { name: 'createShipment', action: async () => { throw new BusinessError('no courier for this address'); },
    compensate: async () => {} },
];
const run = createSagaRunner({ store, steps });
await run('order-77', { sku: 'mug', amount: 1200 }).catch((e) => console.log('first attempt:', e.message));
const result = await run('order-77');                  // a new process picks the saga up again
console.log(result.status, '|', result.error);
console.log(log.join(' → '));
// first attempt: orchestrator process killed
// compensated | createShipment: no courier for this address
// reserve mug → charge 1200 → refund ch-9 → release r-1
```

The second run starts from the persisted state — stock reserved, card charged — and goes straight to the shipment
step; the card is charged once, refunded once, and the reservation released once.

The decision to defend is **persisting the saga's state after every step, before invoking the next one**, which
costs a database write per step and looks redundant when the orchestrator usually runs to completion in
milliseconds. Holding saga state in memory and saving it at the end would be faster — until the orchestrator
process restarts mid-saga, which in a fleet that deploys several times a day isn't rare. At that moment an in-
memory orchestrator has no record of what it did: the card may have been charged, the stock reserved, and nothing
anywhere knows to refund or release them. Persisting after each step means every crash leaves a precise record —
"steps 1 and 2 done, step 3 next" — that any instance can resume from, while the per-step idempotency keys make
the one ambiguous case (a step that ran but whose completion wasn't recorded) safe to repeat. The per-step write
is the price of the saga being recoverable at all, which is the only reason to use one.

---

## 23.10 Questions

**1.** MCQ — What problem does the transactional outbox solve? a) Slow queries b) Losing or inventing events when a database write and a broker publish don't happen together c) Broker ordering d) Schema evolution
<details><summary>Answer</summary>**b)** The event is committed in the same transaction as the change, so it exists if and only if the change does.</details>

**2.** MCQ — Why is 2PC called a blocking protocol? a) It uses locks b) Participants that voted yes can't decide on their own if the coordinator fails, so they wait holding locks c) It's synchronous d) It requires a majority
<details><summary>Answer</summary>**b)** A prepared participant has promised to commit if told to and can't abort unilaterally, so a coordinator crash leaves it in doubt.</details>

**3.** MCQ — What delivery guarantee does an outbox relay provide? a) At most once b) At least once c) Exactly once d) None
<details><summary>Answer</summary>**b)** A crash after publishing but before marking rows sent republishes them; consumers must deduplicate.</details>

**4.** MCQ — In a saga, what is the pivot step? a) The first step b) The step after which the saga can no longer be compensated and must complete c) The compensation for the last step d) The orchestrator
<details><summary>Answer</summary>**b)** Steps after it must be retried until they succeed; failure-prone steps belong before it.</details>

**5.** MCQ — Where must a consumer's deduplication record be written to be reliable? a) In memory b) In a separate cache c) In the same transaction as the consumer's effect d) In the broker
<details><summary>Answer</summary>**c)** Then a crash leaves both or neither, so redelivery is either skipped or applied exactly once.</details>

**6.** MCQ — "Exactly-once processing" in practice means: a) Each message is delivered once b) At-least-once delivery with idempotent or deduplicated processing c) Two-phase commit for every message d) Messages are never retried
<details><summary>Answer</summary>**b)** Delivery can't be guaranteed exactly once; the effect can.</details>

**7.** Short — Why doesn't publishing the event before writing the database fix the dual write?
<details><summary>Answer</summary>It inverts the failure: a crash or rollback after publishing leaves an event describing a change that never committed, and downstream services act on it.</details>

**8.** Short — Why is 2PC acceptable inside distributed databases like Spanner or CockroachDB?
<details><summary>Answer</summary>The coordinator's decision and each participant are replicated with consensus, so a crashed coordinator is replaced by a replica that reads the decision from the replicated log — removing the blocking problem, at the cost of consensus latency.</details>

**9.** Short — Why are compensations not the same as rollbacks?
<details><summary>Answer</summary>The original step committed and was visible; the compensation is a new forward action (a refund, a cancellation) that others can also see. It can't erase observations, emails, or side effects already made.</details>

**10.** Short — Why does orchestration usually beat choreography for multi-step sagas with compensations?
<details><summary>Answer</summary>The flow and its state are explicit in one place, so timeouts, retries, and compensations are centralised and "where is this saga stuck?" has a direct answer; choreography spreads the flow across services' subscriptions, making it hard to see and change.</details>

**11.** Short — Name two countermeasures for the lack of isolation in sagas.
<details><summary>Answer</summary>Any two of: semantic locks (mark records as pending), commutative updates, rereading state before acting, and ordering steps so the riskiest come first and the most visible last.</details>

**12.** Output — Using §23.9's runner with the same three steps and no crash, run a new saga `order-88`. What does the log contain for it, and what is the final status?
<details><summary>Answer</summary>**reserve, charge, refund, release; status `compensated`.** The shipment step fails with a business error, so the two completed steps are compensated in reverse order.</details>

**13.** Output — In §23.9, after `order-77` finishes, `run('order-77')` is called a third time. Does anything new appear in the log?
<details><summary>Answer</summary>**No.** The persisted status is `compensated`, so `run` returns the stored state immediately without executing any step.</details>

**14.** Output — An outbox relay publishes a batch of rows 101–200, then crashes before the `UPDATE … SET published_at`. On restart, which rows does it publish?
<details><summary>Answer</summary>**101–200 again** (plus any newer unpublished rows). The batch's transaction rolled back, so those rows are still unpublished — consumers see them twice and must deduplicate.</details>

**15.** Output — A consumer processes 1,000,000 messages a day; about 0.1% are redelivered. Without deduplication, how many duplicate effects occur daily? With an inbox table?
<details><summary>Answer</summary>**About 1,000; zero.** The inbox records each message ID in the same transaction as its effect, so redeliveries are skipped.</details>

**16.** Debug — The analytics team finds orders in the database with no corresponding `OrderPlaced` event, about 30 a week. The order service saves the order, then publishes to Kafka. Cause and fix?
<details><summary>Answer</summary>A dual write: when the publish fails or the process dies between the two steps, the event is lost. Use a transactional outbox (polled relay or CDC), so the event commits with the order and is published at least once.</details>

**17.** Debug — During a database failover, an XA transaction coordinator restarted and several participant databases reported "prepared transactions" holding row locks; checkout stalled for 40 minutes until a DBA resolved them manually. What's the design lesson?
<details><summary>Answer</summary>Classic 2PC across independently operated systems blocks on coordinator failure, and in-doubt transactions hold locks until resolved. Replace it with an outbox plus a saga for the cross-system flow, or co-locate the data that must change atomically in one database.</details>

**18.** Debug — Customers are occasionally credited loyalty points twice. The points consumer reads `OrderPaid` events and runs `UPDATE points SET balance = balance + $n`, acknowledging the message afterwards. Why, and what's the fix?
<details><summary>Answer</summary>Redelivery after a consumer crash (or rebalance) between the update and the acknowledgement applies the increment twice. Record the message ID in an inbox table in the same transaction as the update and skip duplicates, or make the update conditional on an event version.</details>

**19.** Debug — A choreographed order flow across six services occasionally leaves orders "stuck" with payment taken and no shipment, and nobody can tell which service dropped them. What would you change?
<details><summary>Answer</summary>Move the flow to an orchestrated saga with persisted state per order, explicit timeouts per step, and compensations — so each order's state is one queryable record and stuck sagas trigger alerts or automated compensation. Add correlation IDs to every event for tracing in the meantime.</details>

**20.** Code — Write `consumeOnce(db, message, apply)` that uses an inbox table so `apply(tx)` runs at most once per `message.id`, inside one transaction, returning `'applied'` or `'duplicate'`.
<details><summary>Answer</summary>

```js
async function consumeOnce(db, message, apply) {
  return db.transaction(async (tx) => {
    const { rowCount } = await tx.query(
      'INSERT INTO inbox (message_id) VALUES ($1) ON CONFLICT DO NOTHING', [message.id]);
    if (rowCount === 0) return 'duplicate';   // already processed: commit nothing new, then ack
    await apply(tx);                          // the effect commits with the inbox row
    return 'applied';
  });
}
```
Acknowledge to the broker only after this resolves.</details>

**21.** Code — Write `planCompensation(steps, completedNames)` that returns the names of compensations to run, in order, skipping steps marked `compensate: null` (not compensatable).
<details><summary>Answer</summary>

```js
function planCompensation(steps, completed) {
  const byName = new Map(steps.map((s) => [s.name, s]));
  return [...completed].reverse()
    .filter((n) => byName.get(n)?.compensate)
    .map((n) => `undo:${n}`);
}
console.log(planCompensation(
  [{ name: 'reserve', compensate: () => {} }, { name: 'notify', compensate: null }, { name: 'charge', compensate: () => {} }],
  ['reserve', 'notify', 'charge'])); // [ 'undo:charge', 'undo:reserve' ]
```
A non-compensatable step before others is a design smell: it should be moved after the pivot.</details>

**22.** Code — Write `applyIfNewer(row, event)` for a version-checked consumer that returns the new row only if `event.version > row.version`, else returns `null`.
<details><summary>Answer</summary>

```js
const applyIfNewer = (row, event) =>
  (row && row.version >= event.version) ? null : { ...row, ...event.changes, version: event.version };
console.log(applyIfNewer({ status: 'paid', version: 3 }, { version: 2, changes: { status: 'pending' } }),
  applyIfNewer({ status: 'paid', version: 3 }, { version: 4, changes: { status: 'shipped' } }));
// null { status: 'shipped', version: 4 }
```
In SQL this is one statement: `UPDATE … SET … , version = $v WHERE id = $id AND version < $v`.</details>

**23.** Scenario — A travel booking flow reserves a flight, a hotel, and a car through three external providers, then charges the customer. The car provider's cancellation costs a fee after 10 minutes. Design the saga.
<details><summary>Answer</summary>Orchestrate it with persisted state. Order steps by failure likelihood and compensation cost: authorise (not capture) payment first so a declined card fails cheaply; reserve flight and hotel (cancellable); reserve the car last since its compensation becomes costly; then capture payment as the pivot within the free-cancellation window, with a saga timeout well under 10 minutes that compensates if not reached. After the pivot, send confirmations as retriable steps. Every provider call carries an idempotency key; compensations are retried patiently and escalate to support if they keep failing.</details>

**24.** Scenario — Two teams own "accounts" and "ledger" services and keep needing distributed transactions between them for every transfer. They ask for a 2PC coordinator. What do you advise?
<details><summary>Answer</summary>The constant need for atomicity across them says the boundary is wrong: balances and ledger entries are one consistency unit and belong in one service and database, where a local transaction covers each transfer (Chapter 43). If organisational constraints keep them separate, use a saga with an outbox — but expect it to be harder to get right than merging the data, and don't add a cross-service 2PC coordinator to the critical path.</details>

---

## Chapter Summary

- Local transactions stop at one database; a database write plus a broker publish (or two services' writes) can't
  be made atomic by doing one after the other.
- Two-phase commit makes participants agree but blocks them, holding locks, if the coordinator fails; it belongs
  inside distributed databases that replicate the coordinator with consensus, not between services.
- The transactional outbox commits events with the change they describe and publishes them at least once via a
  polling relay or CDC.
- Consumers must be idempotent — state-setting operations, version checks, or an inbox table written in the same
  transaction as the effect.
- Sagas split long operations into local transactions with compensations; order steps around a pivot, orchestrate
  and persist them, and design for the isolation they lack.
- Exactly-once is an effect, not a delivery guarantee: at-least-once plus idempotency, ending at each system's
  boundary — and a hot-path need for distributed transactions usually means the boundary is wrong.

---

## Practice Exercises

1. Implement the outbox in PostgreSQL with the polling relay from §23.3, kill the relay mid-batch repeatedly under
   load, and verify that consumers receive every event at least once.
2. Replace the polling relay with Debezium (or PostgreSQL logical decoding) and compare publish latency and
   ordering behaviour.
3. Read the documentation for a durable workflow engine (Temporal, Step Functions) and map its concepts —
   activities, retries, timers, compensation — onto §23.9's orchestrator.
4. Add an inbox to a consumer you own, then run a test that redelivers every message twice and asserts the
   end state is unchanged.
5. Pick a multi-service operation in a system you know and design its saga: steps, compensations, pivot, and the
   semantic locks other operations must respect.

---

## Mini Project: An Order Saga Across Three Services

Build three small services — orders, inventory, payments — each with its own database, and implement order
placement as an orchestrated saga with reliable messaging throughout.

- Each service publishes events through a transactional outbox and consumes commands idempotently via an inbox.
- An orchestrator with persisted saga state, per-step timeouts, retries for transient failures, and compensations
  for business failures.
- Semantic locking: orders are `pending` until the saga completes, and inventory treats reserved stock as
  unavailable to other orders.
- A chaos test that kills each service and the orchestrator at random points during 1,000 concurrent sagas and
  verifies every saga ends `completed` or `compensated`, with no double charges and no leaked reservations.
- A dashboard query listing sagas by state and age, with an alert for any saga stuck longer than its timeout.

---

**Next:** [Chapter 24 — Message Queues and Event Logs →](24-queues-logs.md)
