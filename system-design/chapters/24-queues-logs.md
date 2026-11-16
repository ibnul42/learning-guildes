# Chapter 24 — Message Queues and Event Logs

## Learning Objectives

- Explain what asynchronous messaging buys — decoupling in time, load levelling, fan-out — and what it costs
- Choose between a message queue and a partitioned event log for a workload
- State at-most-once, at-least-once, and effectively-once delivery precisely, and get acknowledgement order right
- Preserve ordering where it matters with partition keys, and size partitions and consumers for throughput
- Handle retries, poison messages, dead-letter queues, backpressure, and consumer lag

---

## 24.1 Why Asynchronous Messaging

In a synchronous call, the caller waits while the callee does the work. If the callee is slow, the caller is
slow; if it's down, the caller fails; if it can handle 500 requests per second and 5,000 arrive, most of them
fail. A **message broker** sits between producers and consumers and changes all three:

| Property | What it means | Example |
|---|---|---|
| **Decoupling in time** | The producer doesn't wait for, or even need, the consumer to be running | Orders are accepted while the email service is being deployed |
| **Load levelling** | Bursts are absorbed in the queue and worked off at the consumer's pace | A flash sale's 10× spike becomes a few minutes of backlog instead of errors |
| **Fan-out** | One event feeds many independent consumers | `OrderPlaced` drives email, analytics, fraud, and loyalty without the order service knowing them |
| **Retry and isolation** | Failed work is retried later without blocking the producer | A flaky shipping API doesn't fail checkout |

The costs are just as concrete. The producer can't return the consumer's result, so the work becomes
**eventually** done; failures surface later and elsewhere; ordering, duplicates, and backlogs become things to
design for; and the broker is a stateful system to operate. The design test from Chapter 1 applies: **put work
behind a queue when the caller doesn't need its result to respond**, and keep it synchronous when it does.

> 💼 **Interview** — Adding a queue is the most common reflex in design interviews, and interviewers probe it:
> "What happens if the consumer is down for an hour? If a message can't be processed? If the same message
> arrives twice? If ordering matters?" Have the answers ready: backlog and lag alerts, retries with a
> dead-letter queue, idempotent consumers, and partition keys.

---

## 24.2 Queues and Logs

"Message broker" covers two different designs, and choosing the wrong one is a common, expensive mistake.

**Message queues** (RabbitMQ, Amazon SQS, ActiveMQ, Azure Service Bus queues) hand each message to **one**
consumer, which acknowledges it; acknowledged messages are deleted. Many workers can consume one queue in
parallel, each taking whatever message is next. Delivery state is tracked **per message**.

**Partitioned event logs** (Apache Kafka, Amazon Kinesis, Apache Pulsar, Redpanda) append messages to
durable, ordered **partitions** and **retain** them for a configured period regardless of consumption. Consumers
track their own position — an **offset** — per partition, so many independent consumer groups can read the same
data, each at its own pace, and any of them can **rewind** and replay. Delivery state is a **single number per
partition per consumer group**.

```text
queue:  producer ──▶ [ m5 m4 m3 m2 m1 ] ──▶ worker A takes m1, worker B takes m2 … (deleted when acked)

log:    producer ──▶ partition 0: [ 0 1 2 3 4 5 6 … ]   group "email"     at offset 6
                                                        group "analytics" at offset 2  (replaying)
                     partition 1: [ 0 1 2 3 4 … ]       each group owns its offsets
```

| | Message queue | Partitioned log |
|---|---|---|
| A message is consumed by | One worker | One consumer *per group*; many groups |
| After consumption | Deleted | Retained (hours to forever) |
| Replay | No (once acked, gone) | Yes — reset the offset |
| Ordering | Usually none, or FIFO at reduced throughput | Strict within a partition |
| Parallelism | Add workers freely | Bounded by partition count per group |
| Per-message ack, delay, and retry | Native (visibility timeout, delayed redelivery) | Not native — offsets are positional; retries need extra topics |
| A slow message | Blocks only its worker | Blocks its partition (head-of-line) |
| Throughput | High | Very high (sequential appends, batching) |
| Fits | Task distribution: jobs, emails, image processing | Event streams: many consumers, replay, ordering, stream processing, CDC |

The rule of thumb: **a queue is for work to be done; a log is for facts that happened.** "Resize this image" is
a task — one worker should do it once, retrying on failure — so it belongs in a queue. "Order 77 was placed" is
a fact that several systems need to know, some of them not yet built, and that analytics may want to replay next
year — so it belongs in a log.

---

## 24.3 Delivery Semantics

A message's journey has three steps the consumer controls: **receive** it, **apply** its effect, and
**acknowledge** it (delete it from the queue, or commit the offset). A crash can happen between any two, and the
acknowledgement's position decides what a crash costs:

| Order | Crash before applying | Crash after applying, before ack | Guarantee |
|---|---|---|---|
| Ack, then apply | Message lost | — | **At most once** |
| Apply, then ack | Redelivered (fine) | Redelivered — effect applied **twice** | **At least once** |
| Apply idempotently, then ack | Redelivered (fine) | Redelivered — duplicate skipped | **Effectively once** |

The difference is easy to see in a simulation where the consumer crashes on 1% of messages:

```js
// Where the consumer acknowledges decides what a crash costs. 10,000 messages, and the consumer
// crashes 1% of the time — either just before or just after applying the message's effect.
function mulberry32(seed) {
  return () => { seed |= 0; seed = (seed + 0x6d2b79f5) | 0; let t = Math.imul(seed ^ (seed >>> 15), 1 | seed);
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t; return ((t ^ (t >>> 14)) >>> 0) / 4294967296; };
}
function run(mode, { n = 10_000, crashRate = 0.01, seed = 1 } = {}) {
  const rand = mulberry32(seed);
  const queue = Array.from({ length: n }, (_, i) => i);
  const applied = new Map();                     // message id → times its effect happened
  while (queue.length) {
    const id = queue[0];
    const crash = rand() < crashRate ? (rand() < 0.5 ? 'beforeApply' : 'afterApply') : null;
    const apply = () => {
      if (mode === 'idempotent' && applied.has(id)) return; // dedupe by message id
      applied.set(id, (applied.get(id) ?? 0) + 1);
    };
    if (mode === 'ackFirst') {                   // at-most-once: acknowledge, then work
      queue.shift();
      if (crash === 'beforeApply') continue;     // acked but never applied: lost
      apply();
    } else {                                     // at-least-once: work, then acknowledge
      if (crash === 'beforeApply') continue;     // nothing happened; the message is redelivered
      apply();
      if (crash === 'afterApply') continue;      // applied but not acked: redelivered → duplicate
      queue.shift();
    }
  }
  const lost = n - applied.size;
  const duplicated = [...applied.values()].filter((c) => c > 1).length;
  return `${mode.padEnd(10)} lost ${String(lost).padStart(3)}   duplicated ${String(duplicated).padStart(3)}`;
}
for (const mode of ['ackFirst', 'ackAfter', 'idempotent']) console.log(run(mode));
// ackFirst   lost  47   duplicated   0
// ackAfter   lost   0   duplicated  52
// idempotent lost   0   duplicated   0
```

**At-least-once with idempotent consumers is the default for almost everything**, because losing a message is
usually worse than processing it twice, and duplicates can be made harmless (Chapter 23's inbox, version checks,
or naturally idempotent operations). At-most-once is acceptable only for data where loss is fine — metrics
samples, presence pings, best-effort notifications.

Two broker mechanisms produce redelivery even without crashes:

- **Visibility timeouts** (SQS and similar): a received message becomes invisible for N seconds; if not deleted in
  time, it reappears for another worker. A handler that runs longer than the timeout — without extending it —
  gets its message processed twice, concurrently.
- **Consumer group rebalances** (Kafka and similar): when consumers join, leave, or stall past a session timeout,
  partitions move to other consumers, which resume from the last *committed* offset — re-reading anything processed
  but not yet committed.

> ⚠️ **Gotcha** — **Auto-commit can silently turn at-least-once into at-most-once.** Many log consumer libraries
> commit offsets on a timer by default. If the timer commits an offset for messages that were *received* but
> whose processing then fails or crashes, those messages are skipped forever. For anything that matters, disable
> auto-commit and commit offsets only after the messages' effects are durable.

---

## 24.4 Ordering and Partitioning

Ordering across a whole topic would require a single sequence — one partition, one consumer, no parallelism. Logs
instead guarantee **order within a partition**, and producers choose the partition by a **key**: all messages with
the same key go to the same partition, in order. Choose the key as the entity whose events must be ordered:

- Key by `orderId`: every event for an order — placed, paid, shipped, cancelled — is processed in sequence, while
  different orders proceed in parallel.
- Key by `userId` for a user's activity, by `accountId` for ledger entries, by `deviceId` for telemetry.

Within a consumer group, each partition is assigned to **one consumer** at a time, so a group's parallelism is
capped by the partition count: a topic with 12 partitions can keep at most 12 consumers busy in one group. That
makes partition count a capacity decision made early, since increasing it later changes which partition each key
maps to (Chapter 14's `hash % N` problem) and breaks per-key ordering during the transition.

```js
// Partitions needed for a topic: enough for producer throughput AND for consumer parallelism
function partitionsNeeded({ peakMBps, perPartitionMBps = 10, peakMsgsPerSec, perConsumerMsgsPerSec, headroom = 2 }) {
  const forThroughput = Math.ceil(peakMBps / perPartitionMBps);
  const forConsumers = Math.ceil(peakMsgsPerSec / perConsumerMsgsPerSec);
  return { forThroughput, forConsumers, choose: Math.max(forThroughput, forConsumers) * headroom };
}
console.log(partitionsNeeded({ peakMBps: 60, peakMsgsPerSec: 40_000, perConsumerMsgsPerSec: 2_500 }));
// { forThroughput: 6, forConsumers: 16, choose: 32 }
```

Here consumer processing speed, not bytes, sets the partition count: 16 consumers are needed at peak, and doubling
for growth gives 32.

Ordering has costs that show up in operations:

- **Head-of-line blocking.** A message that's slow or failing blocks everything behind it in its partition — and so
  every key that shares the partition. Retrying it in place preserves order but stalls the partition; moving it to a
  retry topic unblocks the partition but breaks order for that key (§24.5).
- **Hot partitions.** A key with disproportionate traffic (one huge tenant, a viral entity) overloads one partition
  and one consumer — Chapter 12's hot-key problem in a new place. Remedies are the same: split the key
  (`tenant#shard`), or accept that this key's events aren't totally ordered.
- **Ordering only within one producer path.** If two producers emit events for the same key without coordination,
  the partition records them in arrival order, which may not be causal order. The outbox (Chapter 23), with one
  writer per aggregate, gives a meaningful order.

Many queues offer **FIFO** variants (SQS FIFO with message group IDs, RabbitMQ single active consumer) that provide
per-group ordering with lower throughput. They're the queue-world equivalent of partition keys.

---

## 24.5 Retries, Poison Messages, and Dead-Letter Queues

Consumers fail for two different reasons, and they need different treatment:

- **Transient failures** — a dependency timed out, a database failed over, a rate limit was hit. Retrying later,
  with backoff and jitter (Chapter 28), usually succeeds.
- **Permanent failures** — the message is malformed, refers to something that doesn't exist, or triggers a bug.
  Retrying will never succeed. Such a message is a **poison message**: retried forever, it consumes capacity and,
  in an ordered partition, blocks everything behind it.

The standard arrangement:

1. **Retry with backoff** a bounded number of times. Queues support this natively (release the message with a delay,
   or let the visibility timeout expire). Logs need **retry topics**: the consumer commits the original offset and
   republishes the message to `orders.retry.1m`, `orders.retry.10m`, consumed by delayed consumers — which unblocks
   the partition at the cost of ordering for that key.
2. **Dead-letter** after the last attempt: move the message, with its error and attempt history, to a
   **dead-letter queue (DLQ)**, and stop retrying.
3. **Alert on DLQ depth**, inspect, fix the cause, and **redrive** the messages back to the main queue. A DLQ that
   nobody watches is just a slower way to lose messages.

Classify errors in the consumer so permanent failures go to the DLQ immediately instead of burning retries, and so
transient ones aren't dead-lettered by accident. And when ordering per key matters, a dead-lettered message raises a
question the design must answer: should later messages for the same key wait, or proceed? (A payment refund
processed before the payment it refunds is a problem; a profile update processed out of order may not be.)

---

## 24.6 Backpressure and Consumer Lag

A queue absorbs bursts only if consumers eventually catch up. The health of an asynchronous system is measured by
**lag** — how far behind consumers are — in two units:

- **Lag in messages** (log end offset − committed offset, or queue depth): how much work is waiting.
- **Lag in time** (age of the oldest unprocessed message): how stale the consumers' view is. This is the one users
  feel and the one to alert on, because 100,000 messages is fine for a fast consumer and terrible for a slow one.

Little's law (Chapter 2) connects them: at a steady consumption rate, time to drain = backlog ÷ (consumption rate −
arrival rate). A backlog of 600,000 messages, consumed at 5,000/s while 3,000/s keep arriving, drains in
600,000 / 2,000 = 300 seconds. If consumption is *slower* than arrival, the backlog grows without bound, and no amount
of buffering helps — Chapter 3's unstable queue.

```js
// How long until consumers catch up — or whether they ever will
function drainTime({ backlog, consumeRate, arrivalRate }) {
  const net = consumeRate - arrivalRate;
  return net <= 0 ? Infinity : backlog / net;  // seconds
}
console.log(drainTime({ backlog: 600_000, consumeRate: 5_000, arrivalRate: 3_000 }),
  drainTime({ backlog: 600_000, consumeRate: 3_000, arrivalRate: 3_500 })); // 300 Infinity
```

Responses to growing lag, from automatic to drastic:

- **Autoscale consumers on lag** (time-based), up to the partition count for logs.
- **Make consumers faster**: batch database writes, increase per-consumer concurrency for independent keys, remove
  synchronous calls from the hot path.
- **Backpressure the producers**: when the queue exceeds a bound, producers slow down or reject new work (Chapter 29),
  rather than let the backlog — and its staleness — grow forever.
- **Shed low-value work**: drop or sample analytics events, coalesce updates for the same key.

Queues aren't infinite either: brokers have storage limits and retention windows. A log with 7-day retention and a
consumer that's 8 days behind has *lost* a day of messages; alert long before lag approaches retention.

---

## 24.7 Operating Logs and Queues

**Retention and compaction.** Logs retain messages by time or size. **Log compaction** (Kafka) instead keeps the
*latest* message per key forever, discarding older ones — turning a topic into a durable, replayable table of
current state ("the latest profile for every user"), which new consumers can bootstrap from. Deletes are tombstones
(a key with a null value), compacted away after a delay.

**Schemas.** Messages outlive their producers' code (Chapter 12). Define message schemas (Avro, Protobuf, JSON
Schema), keep changes backward compatible (add optional fields; never repurpose or remove), and use a **schema
registry** to reject incompatible changes at publish time. A producer that ships a breaking change can crash every
consumer of the topic at once.

**Message size.** Brokers have limits (around 1 MB by default in Kafka, 256 KB in SQS), and large messages hurt
throughput and latency for everything sharing a partition. Put large payloads in object storage and send a reference
— the **claim-check** pattern (Chapter 17).

**Durability settings.** Kafka's durability depends on producer acknowledgements (`acks=all`), replication factor
(3), and minimum in-sync replicas (2) — the same quorum reasoning as Chapter 13. Defaults have varied across
versions; set them explicitly for anything you can't afford to lose.

**Don't use the database as a queue — mostly.** A table polled with `SELECT … FOR UPDATE SKIP LOCKED` is a perfectly
good queue at modest volumes (it's how the outbox relay works, and how several job libraries work), with the
advantage of transactional enqueue. It stops being good at high throughput, where polling, row churn, and vacuum
(Chapter 10's tombstone and bloat problems) take over.

---

## 24.8 Common Mistakes and Best Practices

**Mistake: a log where a queue fits, or vice versa.** Task distribution with per-message retry fights a log's
positional offsets; multi-consumer replayable events fight a queue's delete-on-ack.

**Mistake: acknowledging before the effect is durable.** Crashes lose messages; ack (or commit offsets) after
applying.

**Mistake: auto-committed offsets.** Messages received but not processed are skipped after a crash or failure.

**Mistake: non-idempotent consumers.** At-least-once delivery, rebalances, and visibility timeouts guarantee
duplicates eventually.

**Mistake: retrying poison messages forever.** They consume capacity and block ordered partitions; dead-letter after
a bounded number of attempts.

**Mistake: a DLQ without alerts or redrive.** Messages park there unseen until someone notices missing work weeks
later.

**Mistake: alerting on queue depth instead of message age.** Depth means nothing without consumption rate; age is
what users experience.

**Best practice: key messages by the entity whose events must stay ordered.** Order where it matters, parallelism
everywhere else.

**Best practice: size partitions for consumer parallelism with headroom.** Repartitioning later reshuffles keys and
breaks ordering.

**Best practice: schemas with compatibility checks.** A registry stops one producer's breaking change from taking
down every consumer.

---

## 24.9 Real-World Example: An At-Least-Once Queue Worker

This worker consumes an SQS-style queue with the protections from this chapter: concurrency bounded so it never
holds more messages than it can work on, visibility extended while a handler runs, acknowledgement only after
success, exponential backoff with jitter on failure, and a dead-letter queue after a bounded number of attempts. The
in-memory queue below it implements the same semantics so the demo runs.

```js
// queue-worker.mjs — an at-least-once consumer for an SQS-style queue: bounded concurrency,
// visibility timeouts extended while working, exponential backoff with jitter on failure,
// a dead-letter queue after N attempts, and acknowledgement only after the handler succeeds.

export function startWorker({ queue, handler, concurrency = 8, maxAttempts = 5, baseDelayMs = 1_000,
  visibilityMs = 30_000, rand = Math.random, onEvent = () => {} }) {
  let running = 0, stopped = false;

  async function processOne(msg) {
    // Keep the message invisible while we work, so a slow handler isn't mistaken for a dead one
    const keepAlive = setInterval(() => queue.extendVisibility(msg.receipt, visibilityMs), visibilityMs / 3);
    try {
      await handler(msg.body, { messageId: msg.id, attempt: msg.receiveCount });
      await queue.delete(msg.receipt);                       // ack only after success
      onEvent('done', msg);
    } catch (err) {
      if (msg.receiveCount >= maxAttempts) {
        await queue.deadLetter(msg, err.message);            // park it: a poison message mustn't loop forever
        onEvent('dead-lettered', msg);
      } else {
        const delay = Math.floor(rand() * baseDelayMs * 2 ** msg.receiveCount); // full jitter
        await queue.release(msg.receipt, delay);             // visible again after the backoff
        onEvent('retry', msg);
      }
    } finally {
      clearInterval(keepAlive);
    }
  }

  async function loop() {
    while (!stopped) {
      const free = concurrency - running;                   // never pull more than we can work on
      const batch = free > 0 ? await queue.receive(Math.min(free, 10), visibilityMs) : [];
      if (batch.length === 0) { await new Promise((r) => setTimeout(r, 20)); continue; }
      for (const msg of batch) {
        running++;
        processOne(msg).finally(() => { running--; });
      }
    }
  }
  loop();
  return { stop: () => { stopped = true; } };
}

// ── An in-memory queue with SQS-like semantics, for the demo ──────────────────────────
function memoryQueue() {
  const msgs = [], dlq = [];
  let seq = 0;
  const now = () => Date.now();
  return {
    dlq,
    send(body) { msgs.push({ id: `m${++seq}`, body, visibleAt: 0, receiveCount: 0 }); },
    async receive(max, visibilityMs) {
      const out = msgs.filter((m) => m.visibleAt <= now()).slice(0, max);
      for (const m of out) { m.visibleAt = now() + visibilityMs; m.receiveCount++; m.receipt = m; }
      return out.map((m) => ({ id: m.id, body: m.body, receiveCount: m.receiveCount, receipt: m }));
    },
    async extendVisibility(m, ms) { m.visibleAt = now() + ms; },
    async release(m, delayMs) { m.visibleAt = now() + delayMs; },
    async delete(m) { msgs.splice(msgs.indexOf(m), 1); },
    async deadLetter(msg, reason) { dlq.push({ id: msg.id, reason }); await this.delete(msg.receipt); },
    get depth() { return msgs.length; },
  };
}

const q = memoryQueue();
for (let i = 1; i <= 20; i++) q.send({ orderId: i });
const attempts = new Map(), done = new Set();
const worker = startWorker({
  queue: q, concurrency: 4, maxAttempts: 3, baseDelayMs: 5, rand: () => 0.5,
  handler: async ({ orderId }, { messageId }) => {
    attempts.set(orderId, (attempts.get(orderId) ?? 0) + 1);
    if (orderId === 13) throw new Error('malformed address');                 // poison: always fails
    if (orderId % 5 === 0 && attempts.get(orderId) === 1) throw new Error('timeout'); // transient
    done.add(messageId);                                                        // idempotent effect
  },
});
while (q.depth > 0) await new Promise((r) => setTimeout(r, 20));
worker.stop();
console.log('processed:', done.size, '| dead-lettered:', q.dlq.map((d) => `${d.id} (${d.reason})`).join(', '));
console.log('attempts for 5, 10, 13:', attempts.get(5), attempts.get(10), attempts.get(13));
// processed: 19 | dead-lettered: m13 (malformed address)
// attempts for 5, 10, 13: 2 2 3
```

Messages 5, 10, 15, and 20 fail once with a transient error and succeed on retry; message 13 fails every time and is
dead-lettered after its third attempt instead of looping forever.

The decision to defend is **sending messages to a dead-letter queue after a fixed number of attempts**, rather than
either retrying indefinitely or dropping them. Infinite retry feels safest — nothing is ever given up on — but a
poison message retried forever consumes worker capacity on every cycle, and in an ordered queue or partition it blocks
every message behind it, so one malformed order can stop all order processing. Dropping after N attempts solves that
and loses data. The DLQ is the honest middle: the system stops spending capacity on a message it has evidence it can't
process, the message and its error history are preserved, an alert tells a human, and once the bug or data issue is
fixed the message is redriven. "Bounded retries, then park and alert" turns an infinite automated loop into a finite
manual decision — which is the right shape for failures that automation can't fix.

---

## 24.10 Questions

**1.** MCQ — Which workload fits a partitioned log better than a queue? a) Distributing image-resize jobs to workers b) An `OrderPlaced` stream consumed by email, analytics, and fraud, with replay for a new consumer c) Sending one email per signup d) A per-message delayed retry
<details><summary>Answer</summary>**b)** Multiple independent consumer groups and replay are what retained, offset-tracked logs provide.</details>

**2.** MCQ — A consumer acknowledges a message, then applies its effect, then crashes midway. What guarantee does this ordering give? a) Exactly once b) At least once c) At most once d) None
<details><summary>Answer</summary>**c)** Acknowledged-but-unapplied messages are lost on a crash.</details>

**3.** MCQ — A topic has 8 partitions and a consumer group has 12 consumers. How many consumers are actively processing? a) 12 b) 8 c) 4 d) 1
<details><summary>Answer</summary>**b)** Each partition is assigned to one consumer in the group; the extra 4 sit idle.</details>

**4.** MCQ — Which key preserves the order of all events for an order while allowing parallelism? a) A random key b) The event type c) The order ID d) The timestamp
<details><summary>Answer</summary>**c)** All of an order's events land in one partition in order; different orders spread across partitions.</details>

**5.** MCQ — What's the best metric to alert on for an asynchronous pipeline's health? a) Queue depth in messages b) Age of the oldest unprocessed message c) Broker CPU d) Number of consumers
<details><summary>Answer</summary>**b)** Time lag is what users experience and is meaningful regardless of consumption rate.</details>

**6.** MCQ — What does Kafka log compaction keep? a) The oldest message per key b) The latest message per key c) Messages from the last 7 days d) Only tombstones
<details><summary>Answer</summary>**b)** Older values for a key are discarded, making the topic a replayable table of current state.</details>

**7.** Short — Why is "a queue is for work, a log is for facts" a useful rule?
<details><summary>Answer</summary>Work should be done once by one worker with per-message retry and then discarded — a queue's model. Facts are needed by many consumers, possibly later, and may be replayed — a log's model of retention and per-group offsets.</details>

**8.** Short — How do visibility timeouts cause duplicate processing, and how do you prevent it?
<details><summary>Answer</summary>If a handler runs longer than the timeout without extending it, the message becomes visible and another worker processes it concurrently. Extend visibility periodically while working, size the timeout above normal processing time, and keep handlers idempotent.</details>

**9.** Short — Why does increasing a topic's partition count break per-key ordering?
<details><summary>Answer</summary>Keys map to partitions by hash modulo the partition count; changing the count remaps many keys, so a key's new messages land in a different partition from its old, unconsumed ones, and the two can be processed out of order.</details>

**10.** Short — What's the trade-off between retrying a failed message in place and moving it to a retry topic?
<details><summary>Answer</summary>Retrying in place preserves ordering but blocks the partition (head-of-line). A retry topic unblocks the partition but lets later messages for the same key overtake the failed one.</details>

**11.** Short — What is the claim-check pattern?
<details><summary>Answer</summary>Store a large payload in object storage and send only a reference (and metadata) through the broker, keeping messages small for throughput and within size limits.</details>

**12.** Output — Using §24.4's `partitionsNeeded`, what does `partitionsNeeded({ peakMBps: 200, peakMsgsPerSec: 30_000, perConsumerMsgsPerSec: 3_000 })` return?
<details><summary>Answer</summary>`{ forThroughput: 20, forConsumers: 10, choose: 40 }` — here bytes dominate: 200 / 10 = 20 partitions, doubled for headroom.</details>

**13.** Output — Using §24.6's `drainTime`, how long does a backlog of 1,800,000 messages take to drain at 12,000/s consumed while 9,000/s arrive?
<details><summary>Answer</summary>**600 seconds (10 minutes).** 1,800,000 / (12,000 − 9,000).</details>

**14.** Output — In §24.9's demo, how many times is message 15's handler called, and how many total handler calls happen across all 20 messages?
<details><summary>Answer</summary>**2, and 26 in total.** Fifteen messages succeed first time (15 calls); 5, 10, 15, and 20 fail once and succeed on retry (8 calls); 13 is tried 3 times and dead-lettered (3 calls): 15 + 8 + 3 = 26.</details>

**15.** Output — A log retains 3 days of messages. A consumer group stalls for 80 hours and then resumes. What happens?
<details><summary>Answer</summary>**It has lost about 8 hours of messages.** Messages older than 72 hours were deleted by retention before the consumer reached them; depending on its reset policy it jumps to the earliest available offset (skipping the gap) or to the latest.</details>

**16.** Debug — After a deploy, a consumer group's lag climbs steadily while consumers show 100% CPU on one instance and near zero on the rest. What's the likely cause, and the fix?
<details><summary>Answer</summary>A hot partition: one key (a large tenant, a hot entity) dominates traffic, and its partition's single consumer can't keep up. Split the key (e.g. `tenant#shard`) if strict per-key order isn't required, speed up that consumer's processing, or isolate the hot key on its own topic.</details>

**17.** Debug — Some order events are silently never processed by the analytics consumer, though they're in the topic. The consumer uses its library's default settings and occasionally throws on bad records. What's happening?
<details><summary>Answer</summary>Offsets are auto-committed on a timer, so offsets for records that were received but threw (or were in flight during a crash) get committed and the records are skipped. Disable auto-commit, commit after successful processing, and route bad records to a DLQ instead of throwing past them.</details>

**18.** Debug — An email worker sometimes sends the same email twice, a minute apart, during peak hours. Each send takes a few seconds normally but up to 70 seconds under load; the queue's visibility timeout is 60 seconds. Cause and fix?
<details><summary>Answer</summary>Under load, sends exceed the visibility timeout, the message reappears, and a second worker sends it again. Extend visibility while working (heartbeat), raise the timeout above worst-case duration, and make sending idempotent with a per-message key checked before sending.</details>

**19.** Debug — One malformed message stopped all processing of the `payments` topic for two hours; consumers kept restarting. What design changes prevent this?
<details><summary>Answer</summary>The consumer retried the poison message in place forever, blocking its partition and crash-looping. Classify errors, retry transient ones a bounded number of times, send permanent failures to a DLQ (committing past them), alert on DLQ depth, and validate messages at the producer with a schema registry.</details>

**20.** Code — Write `classifyError(err)` returning `'retry'` for timeouts, connection resets, and HTTP 429/503, and `'dead-letter'` for validation errors and 4xx otherwise.
<details><summary>Answer</summary>

```js
function classifyError(err) {
  const transientCodes = new Set(['ETIMEDOUT', 'ECONNRESET', 'ECONNREFUSED', 'EAI_AGAIN']);
  if (transientCodes.has(err.code)) return 'retry';
  if (err.status === 429 || err.status === 503 || err.status === 502 || err.status === 504) return 'retry';
  if (err.name === 'ValidationError' || (err.status >= 400 && err.status < 500)) return 'dead-letter';
  return 'retry'; // unknown: retry (bounded), and let maxAttempts dead-letter it eventually
}
console.log(classifyError({ code: 'ETIMEDOUT' }), classifyError({ status: 422 }), classifyError({ status: 503 })); // retry dead-letter retry
```
</details>

**21.** Code — Write `partitionFor(key, partitions)` using a stable hash so the same key always maps to the same partition.
<details><summary>Answer</summary>

```js
import { createHash } from 'node:crypto';
const partitionFor = (key, partitions) =>
  createHash('md5').update(String(key)).digest().readUInt32BE(0) % partitions;
console.log(partitionFor('order-77', 12) === partitionFor('order-77', 12)); // true
```
Any stable hash works; it must match across every producer, so use the broker client's default partitioner unless you control them all.</details>

**22.** Code — Write `lagSeconds(partitions, now)` where each partition is `{ committedOffset, endOffset, oldestUnconsumedTs }`, returning the maximum age in seconds of the oldest unconsumed message (0 if caught up).
<details><summary>Answer</summary>

```js
function lagSeconds(partitions, now = Date.now()) {
  return Math.max(0, ...partitions
    .filter((p) => p.endOffset > p.committedOffset)          // only partitions with a backlog
    .map((p) => (now - p.oldestUnconsumedTs) / 1000));
}
console.log(lagSeconds([
  { committedOffset: 100, endOffset: 100, oldestUnconsumedTs: 0 },
  { committedOffset: 50, endOffset: 80, oldestUnconsumedTs: 940_000 },
], 1_000_000)); // 60
```
</details>

**23.** Scenario — A team wants to use Kafka for a job system that resizes uploaded images, needs per-job retries with delays, and occasionally has jobs that take minutes. What do you advise?
<details><summary>Answer</summary>This is task distribution: one worker per job, per-message delayed retries, variable durations. A queue (SQS, RabbitMQ) fits natively — visibility timeouts, delayed redelivery, a DLQ, and workers scaling independently of any partition count. On Kafka, a slow job blocks its partition, retries need retry topics, and parallelism is capped by partitions. If `ImageUploaded` events are already in Kafka, have a small bridge enqueue resize jobs from them.</details>

**24.** Scenario — A ledger service consumes `PaymentCaptured` and `PaymentRefunded` events. Refunds must never be applied before the capture they reverse. Design the topic and consumer.
<details><summary>Answer</summary>Publish both event types to one topic keyed by payment ID (via the outbox, so the producer's order is the commit order), so each payment's events are ordered in one partition. The consumer applies events idempotently with an inbox or a per-payment version check, commits offsets after the database transaction, and treats a refund whose capture isn't present as a sequencing error — retried briefly, then dead-lettered with an alert rather than skipped, because skipping would let later events for that payment proceed out of order.</details>

---

## Chapter Summary

- Asynchronous messaging decouples producers and consumers in time, levels load, and fans events out — at the cost of
  eventual completion, duplicates, ordering questions, and a stateful broker.
- Queues distribute work: one consumer per message, per-message acknowledgement and retry, deleted when done. Logs
  record facts: retained, ordered per partition, read by many groups at their own offsets, replayable.
- Acknowledge after the effect is durable and make consumers idempotent: at-least-once plus idempotency is the
  effectively-once default, and auto-commit can silently break it.
- Order is per partition by key; choose the key as the entity whose events must stay ordered, and size partitions for
  consumer parallelism with headroom, because changing the count reshuffles keys.
- Retry transient failures with bounded backoff, dead-letter poison messages, alert on the DLQ, and redrive after
  fixing the cause.
- Watch lag in time, autoscale on it, backpressure producers before backlogs approach retention, and keep messages
  small and schema-checked.

---

## Practice Exercises

1. Run Kafka (or Redpanda) locally, produce keyed messages to a 6-partition topic, and observe how consumers in one
   group share partitions as you add and remove them.
2. Reproduce the auto-commit message-loss bug with a consumer that throws on some records, then fix it with manual
   commits and a DLQ.
3. Read your queue's documentation on visibility timeouts, delayed messages, and redrive policies, and configure a
   DLQ with alerting for a test queue.
4. Measure a consumer's throughput, compute `partitionsNeeded` for a realistic peak, and compare it with the topic's
   actual partition count.
5. Build a retry-topic chain (`retry.10s`, `retry.1m`, `dlq`) for a Kafka consumer and measure its effect on ordering
   for keys with failing messages.

---

## Mini Project: A Reliable Event Pipeline

Build a small pipeline: an order service writes orders and outbox events in PostgreSQL; a relay publishes them to a
log; three consumers (email, analytics, loyalty points) process them independently with the guarantees from this
chapter.

- Outbox relay publishing keyed by order ID; at least one consumer group replays from the beginning on demand.
- Consumers with manual offset commits, inbox-based deduplication, and error classification.
- A retry topic with delayed reprocessing and a DLQ with a redrive command.
- Lag metrics in messages and seconds per consumer group, with an alert threshold.
- A chaos test that kills consumers and the relay during a burst of 50,000 orders and verifies every order is
  processed exactly once in effect by every consumer.

---

**Next:** [Chapter 25 — Stream and Batch Processing →](25-stream-batch-processing.md)
