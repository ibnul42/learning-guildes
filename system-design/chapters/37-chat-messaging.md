# Chapter 37 — Design a Chat System

## Learning Objectives

- Scope a chat system with one-to-one and group conversations, receipts, presence, and multi-device sync
- Size the connection tier, message throughput, and storage from user numbers
- Guarantee per-conversation ordering and exactly-once display with sequence numbers and client message IDs
- Deliver to online and offline devices, and sync history across devices with per-device cursors
- Implement receipts and presence without multiplying traffic, and know how end-to-end encryption changes the design

---

## 37.1 Requirements and Scope

A chat system lets people exchange messages in real time, one-to-one and in groups, across several devices each.
It combines most of the hard parts in this book: millions of persistent connections (Chapter 9), ordered delivery under
retries (Chapters 22–24), and a write-heavy store partitioned for one dominant access pattern (Chapters 12, 14).

**Functional requirements**

- Send text messages (and media references) in one-to-one and group conversations (up to 1,000 members).
- Deliver messages in real time to online recipients and on reconnect to offline ones; notify offline devices via push.
- Show message status: sent (stored by the server), delivered (reached a recipient device), read.
- Sync conversation history to a new or returning device.
- Show presence: online, or last seen.

**Out of scope**: voice and video calls, stories, payments, message search (mentioned in §37.8), very large broadcast
channels (a different, feed-like design — Chapter 36).

**Non-functional requirements**

| Requirement | Target |
|---|---|
| Delivery latency | p99 < 500 ms sender-to-recipient when both are online in the same region |
| Ordering | Messages in a conversation appear in the same order on every device |
| Delivery guarantee | No acknowledged message is ever lost; none is displayed twice |
| Availability | 99.99% for sending and receiving |
| Scale | 500 M DAU, mostly mobile, global |

The key distinction is between **real-time delivery** (the fast path over live connections) and **durable delivery** (the
guarantee, backed by storage and sync). As in Chapter 9, the live connection is an optimisation; correctness comes from the
stored, sequenced log.

---

## 37.2 Estimation

```js
// Chat: the numbers that size each tier
const dau = 500e6, msgsPerUserPerDay = 40, bytesPerMsg = 200, peak = 3;
const msgsPerSec = (dau * msgsPerUserPerDay) / 86_400;
const concurrent = dau * 0.3;                         // ~30% of DAU connected at peak (sessions, background)
const connsPerGateway = 100_000;
console.log(`messages ~${Math.round(msgsPerSec / 1000)}k/s avg, ~${Math.round((msgsPerSec * peak) / 1000)}k/s peak`);
console.log(`storage ~${((dau * msgsPerUserPerDay * bytesPerMsg) / 1e12).toFixed(0)} TB/day raw, ~${((dau * msgsPerUserPerDay * bytesPerMsg * 365) / 1e15).toFixed(1)} PB/year`);
console.log(`peak connections ~${concurrent / 1e6} M → ${Math.ceil(concurrent / connsPerGateway).toLocaleString('en-US')} gateway nodes at ${connsPerGateway / 1000}k each (before N+1)`);
// messages ~231k/s avg, ~694k/s peak
// storage ~4 TB/day raw, ~1.5 PB/year
// peak connections ~150 M → 1,500 gateway nodes at 100k each (before N+1)
```

What the numbers decide:

- **~700,000 messages per second at peak** is a write-heavy workload for the message store — an LSM-based, partitioned store
  (Chapter 10) rather than a single relational primary.
- **~1.5 PB per year** before replication means retention policy, compression, and tiering to object storage for old history
  are design decisions, not afterthoughts.
- **~150 million concurrent connections** need a dedicated, thin gateway tier of well over a thousand nodes, plus headroom for a
  zone's worth of reconnects (Chapter 9).
- Each message to a group of size *g* becomes up to *g* deliveries, so delivery traffic exceeds message traffic by the average
  effective group size.

---

## 37.3 API and Data Model

Clients hold a persistent connection (WebSocket, or a mobile-optimised protocol over TCP/QUIC) to a gateway, over which both
directions flow:

```text
client → server  send     { conversationId, clientMessageId, body }
server → client  ack      { clientMessageId, seq, sentAt }                 ("sent" ✓)
server → client  message  { conversationId, seq, sender, body, sentAt }
client → server  delivered{ conversationId, upToSeq }                      (per device)
client → server  read     { conversationId, upToSeq }                      ("read" ✓✓)
client → server  sync     { conversationId, afterSeq, limit }  → messages[]
```

`clientMessageId` is generated once per message by the sending device and reused on every retry — it's the idempotency key
(Chapter 8) that makes resending safe.

| Data | Store | Key | Access pattern |
|---|---|---|---|
| **Messages** | Partitioned wide-column / LSM store | Partition: `conversation_id` (time-bucketed for huge conversations); clustering: `seq` | Append; range read by `seq` |
| **Conversations and membership** | Relational or wide-column | `conversation_id`; and `user_id → conversations` | List my conversations; list members |
| **Per-member read cursors** | Wide-column | `(conversation_id, member)` → `read_up_to` | Update; read for receipts |
| **Per-device sync state** | Wide-column | `(user_id, device_id)` → per-conversation `delivered_up_to` | Sync after reconnect |
| **Connection registry** | In-memory with TTL | `user_id → [(device, gateway)]` | Route a delivery to the right gateway |

Partitioning messages by conversation makes the dominant query — "messages in this conversation after seq N" — a single
partition range scan, and makes per-conversation ordering a local property of one partition.

---

## 37.4 High-Level Design

```text
devices ◀═ws═▶ gateways (connections, auth, heartbeats) ──▶ chat service
                    ▲                                         │ 1. dedupe + assign seq (per conversation)
                    │                                         │ 2. append to message store
                    │                                         │ 3. ack sender
                    │                                         ▼
                    │                                   message log (partitioned by conversation)
                    │                                         │
                    └──── delivery workers ◀──────────────────┘ look up members' devices in the registry
                           ├─ online device  → push over its gateway
                           └─ offline device → push notification (APNs/FCM), device syncs on next open
```

The **gateways** hold connections and nothing else (Chapter 9: thin, so they rarely need redeploying). The **chat service**
handles a send: deduplicate by client message ID, assign the next sequence number for the conversation, append to the
message store, and acknowledge the sender — in that order, so an ack always means durably stored. **Delivery workers** consume
the message log, look up each member's connected devices in the registry, and push to the right gateways; for devices without
a connection, they trigger push notifications. A device that reconnects calls `sync` from its last delivered cursor and fills
any gap.

---

## 37.5 Deep Dive: Ordering and Exactly-Once Display

Two problems look similar and are solved differently:

**Ordering.** Messages in a conversation must appear in one order on every device. Wall-clock timestamps can't provide it
(Chapter 22: skew reorders close events, and different senders' clocks disagree). The solution is a **sequence number per
conversation**, assigned by a single owner of that conversation — the partition leader for the conversation's key, or an atomic
increment on the conversation's counter stored with the messages. Every device sorts by `seq`, so everyone agrees. Because
ordering is only needed *within* a conversation, the system needs no global sequencer: each conversation's owner sequences
independently, and the work spreads across partitions.

**Exactly-once display.** Networks lose acknowledgements, so senders retry (Chapter 19), and delivery is at-least-once (Chapter
24), so recipients may receive a message twice. Both are fixed with idempotency:

- The **sender** reuses its `clientMessageId` on every retry; the chat service records `(conversation, sender, clientMessageId)
  → seq` and returns the existing `seq` for a duplicate rather than storing the message again.
- The **recipient** deduplicates by `(conversation, seq)` — trivially, since it stores messages keyed by seq.

Sequence numbers do a third job: **gap detection**. A device that holds seq 41 and receives 43 knows 42 is missing and fetches it
via `sync`, instead of silently displaying an incomplete conversation (Chapter 9's resume, applied per conversation).

> 💼 **Interview** — "How do you ensure messages arrive in order?" The strong answer: per-conversation sequence numbers assigned
> by a single owner of that conversation, not timestamps; clients sort by sequence and fill gaps via sync; retries deduplicated
> by a client-generated message ID. Then note there's no need for global order across conversations.

**Hot conversations.** A single owner per conversation is a throughput limit for that conversation — fine for chats, where even
a busy group sends a few messages per second. Very large, very active groups (thousands of members, hundreds of messages per
second) are better treated as broadcast channels with a different design.

---

## 37.6 Deep Dive: Delivery and Multi-Device Sync

Each user has several devices — phone, laptop, tablet — each with its own connection state and its own view of what it has
received. The design tracks delivery **per device**, as a cursor per conversation: "this device has everything up to seq N". The
real-world example in §37.10 implements it.

**Online delivery.** When a message is stored, delivery workers look up every member's connected devices in the registry
(Chapter 9's routing registry: `user → [(device, gateway)]`, maintained by gateways with heartbeat-refreshed TTLs) and push the
message through those gateways. The device acknowledges receipt by advancing its delivered cursor.

**Offline delivery.** For members with no connected device, the worker sends a platform push notification. The notification is a
nudge — possibly with a preview, possibly coalesced by the OS — and when the app opens, it syncs from its cursors. Nothing depends
on the push arriving; the message log is the source of truth.

**Reconnect and new devices.** A reconnecting device calls `sync` for each conversation with activity since its cursor (the server
can tell it which conversations changed using a per-user "conversation updated" index). A new device starts with no cursors and
pulls recent history — typically the last few hundred messages per conversation, with older history loaded on scroll.

**Group fan-out.** A message is **stored once**, in the conversation's partition — not copied into every member's inbox. What fans
out is *delivery*: one push per connected member device. For a 1,000-member group, that's up to a few thousand pushes per message,
spread across gateways; delivery workers batch pushes per gateway, so each gateway receives one message carrying a list of local
connections (Chapter 9's fan-out once per gateway).

| | Store per conversation (this design) | Copy into each member's inbox |
|---|---|---|
| Writes per group message | 1 | Group size |
| Storage | 1× | Group size × |
| Reading a conversation | One partition range read | One read per member inbox (or a merge) |
| Fits | Groups of any size up to broadcast scale | Very small groups, or when per-user mailbox semantics are required (e.g. end-to-end encrypted per-device copies) |

---

## 37.7 Deep Dive: Receipts and Presence

Both features are easy to build badly — as one event per message per recipient — and multiply traffic many times over.

**Receipts as cursors.** "Read" means "this member has read up to seq N in this conversation". One number per member per
conversation, updated as they scroll, replaces a receipt per message: reading 50 messages at once is one update, not 50. The sender's
✓✓ is computed by comparing a message's seq with members' cursors (in a group, "read by 3 of 5"). Delivered receipts work the same
way, aggregated across a member's devices (delivered if any device has it). Receipt updates are themselves delivered to the sender's
devices, but they can be coalesced and rate-limited — they're the most droppable traffic in the system.

**Presence.** "Online" and "last seen" look trivial and are among the most traffic-heavy features in chat, because a user's status
change is interesting to everyone who has them in a contact list. Designs that scale:

- **Derive presence from connection heartbeats** with a TTL: a user is online while any device's heartbeat is fresh. Gateways
  update a presence store; entries expire on their own when a device disappears without disconnecting.
- **Don't broadcast every change.** Push presence only to users who are currently looking — someone with that conversation open, or
  the contact list on screen — via subscriptions that last as long as the view does. Everyone else fetches presence lazily when they
  open a chat.
- **Coarsen it.** "Last seen recently" rounded to minutes, debounced so flapping connections (a phone switching networks) don't
  generate storms of online/offline events, and respecting privacy settings.

---

## 37.8 Failure Modes, Bottlenecks, and 10×

| Failure | Effect | Mitigation |
|---|---|---|
| Gateway node dies | 100k devices disconnect | Jittered reconnect to other gateways; resume via sync from cursors; nothing lost (Chapter 9) |
| Chat service instance dies mid-send | Sender gets no ack | Sender retries with the same `clientMessageId`; dedupe makes it safe |
| Message store partition unavailable | Sends to conversations on that partition fail | Replicated partitions with fast leader failover (Chapters 13, 21); client queues and retries |
| Delivery workers lag | Messages stored but delivered late | Lag alert; devices also receive messages on their next sync; autoscale on lag |
| Push provider throttling | Offline users not notified | Coalesce notifications per conversation; retry with backoff; sync on app open is unaffected |
| Region outage | Users homed there disconnected | Users re-home to another region (Chapter 32); messages replicated across regions asynchronously |

**End-to-end encryption** changes the design materially. With protocols like Signal's, the server relays ciphertext it can't read:
messages are encrypted per recipient *device*, so a group message becomes one ciphertext per member device (the inbox-copy model from
§37.6 for the encrypted payload, or sender-key schemes that encrypt once per group with distributed keys). The server can still
sequence, store, and deliver — but server-side search, content moderation, and history restoration on new devices must move to the
clients or be given up, and key management (device registration, verification, key changes) becomes a core feature.

**Search** over message history, if offered without end-to-end encryption, is a derived index fed from the message log (Chapter 18),
partitioned by user or conversation so each search touches only the searcher's conversations.

**At 10×** — 7 million messages per second at peak and over a billion concurrent connections:

- The gateway tier grows to tens of thousands of nodes, which pushes connection-routing and presence into regional tiers; users connect
  to their home region (Chapter 32).
- The message store grows by ~15 PB a year: tiered storage (recent messages on SSD, older ones compacted to object storage, fetched on
  demand), and retention policies matter more than raw capacity.
- Delivery fan-out for groups dominates traffic; batching per gateway and suppressing deliveries to devices that are in the
  background (relying on sync) are the main levers.

---

## 37.9 Common Mistakes and Best Practices

**Mistake: ordering by timestamps.** Clock skew between senders and servers reorders messages differently on different devices.

**Mistake: retries without client message IDs.** A lost ack becomes a duplicate message in the conversation.

**Mistake: real-time delivery without durable sync.** Messages sent while a device is offline or reconnecting are lost.

**Mistake: copying every group message into every member's inbox.** Writes and storage multiply by group size; store once per
conversation.

**Mistake: one receipt per message per recipient.** Traffic multiplies; use per-member read cursors.

**Mistake: broadcasting every presence change.** Presence becomes the largest traffic source; push only to active viewers and debounce.

**Mistake: business logic in gateways.** Every change redeploys the connection tier and disconnects millions of devices.

**Best practice: per-conversation sequence numbers from a single owner.** Total order where it's needed, parallelism everywhere else,
and gap detection for free.

**Best practice: ack only after the message is durably stored.** "Sent" then means it can't be lost.

**Best practice: per-device cursors and sync as the source of truth.** Push is a fast path; sync guarantees completeness.

---

## 37.10 Real-World Example: A Sequenced Conversation Log

This is the core of the chat backend: sequence numbers assigned per conversation, sends deduplicated by the client's message ID,
per-device sync cursors, and read receipts as one cursor per member. In production the log is a partitioned store keyed by
conversation, and sequence assignment happens atomically with the append on the conversation's partition leader.

```js
// conversation-log.mjs — the core of a chat backend: per-conversation sequence numbers, idempotent
// sends keyed by the client's message ID, receipts as per-member cursors, and per-device sync.

export class ConversationLog {
  #messages = new Map();     // conversationId → [{ seq, id, sender, body, sentAt }]
  #byClientId = new Map();   // "conv|sender|clientMessageId" → seq  (dedupe retries)
  #readUpTo = new Map();     // "conv|member" → highest seq the member has read
  #deviceCursor = new Map(); // "user|device" → { conv → highest seq delivered to that device }

  // One owner per conversation assigns sequence numbers; in production this is a single partition
  // leader or an atomic increment on the conversation row, inside the same write as the message.
  send({ conversationId, sender, clientMessageId, body, now = Date.now() }) {
    const dedupeKey = `${conversationId}|${sender}|${clientMessageId}`;
    if (this.#byClientId.has(dedupeKey)) {                      // a retry after a lost ack
      return { seq: this.#byClientId.get(dedupeKey), duplicate: true };
    }
    const log = this.#messages.get(conversationId) ?? [];
    const seq = (log.at(-1)?.seq ?? 0) + 1;                     // gap-free within the conversation
    log.push({ seq, id: `${conversationId}:${seq}`, sender, body, sentAt: now });
    this.#messages.set(conversationId, log);
    this.#byClientId.set(dedupeKey, seq);
    this.markRead(conversationId, sender, seq);                 // you've read your own message
    return { seq, duplicate: false };
  }

  // A device asks: "what have I missed in this conversation since the last seq I acknowledged?"
  sync({ user, device, conversationId, limit = 100 }) {
    const cursors = this.#deviceCursor.get(`${user}|${device}`) ?? {};
    const after = cursors[conversationId] ?? 0;
    const batch = (this.#messages.get(conversationId) ?? []).filter((m) => m.seq > after).slice(0, limit);
    return { messages: batch, upTo: batch.at(-1)?.seq ?? after };
  }
  ackDelivered({ user, device, conversationId, upTo }) {        // advance only forward (monotonic)
    const k = `${user}|${device}`, c = this.#deviceCursor.get(k) ?? {};
    c[conversationId] = Math.max(c[conversationId] ?? 0, upTo);
    this.#deviceCursor.set(k, c);
  }

  // "Read up to N" is one number per member — not one receipt per message
  markRead(conversationId, member, seq) {
    const k = `${conversationId}|${member}`;
    this.#readUpTo.set(k, Math.max(this.#readUpTo.get(k) ?? 0, seq));
  }
  readBy(conversationId, seq, members) { return members.filter((m) => (this.#readUpTo.get(`${conversationId}|${m}`) ?? 0) >= seq); }
}

// ── Demo: a group chat, a retried send, two devices, and receipts ────────────────────
const chat = new ConversationLog();
const conv = 'grp-7', members = ['ana', 'bo', 'cy'];
chat.send({ conversationId: conv, sender: 'ana', clientMessageId: 'a-1', body: 'lunch?' });
const retry = chat.send({ conversationId: conv, sender: 'ana', clientMessageId: 'a-1', body: 'lunch?' }); // lost ack, resent
chat.send({ conversationId: conv, sender: 'bo', clientMessageId: 'b-1', body: 'yes, 12:30' });
chat.send({ conversationId: conv, sender: 'ana', clientMessageId: 'a-2', body: 'booked' });
console.log('retry deduplicated:', retry, '| messages stored:', chat.sync({ user: 'cy', device: 'phone', conversationId: conv }).messages.length);
// retry deduplicated: { seq: 1, duplicate: true } | messages stored: 3

const phone = chat.sync({ user: 'cy', device: 'phone', conversationId: conv });
chat.ackDelivered({ user: 'cy', device: 'phone', conversationId: conv, upTo: phone.upTo });
chat.send({ conversationId: conv, sender: 'bo', clientMessageId: 'b-2', body: 'see you there' });
console.log('phone catches up:', chat.sync({ user: 'cy', device: 'phone', conversationId: conv }).messages.map((m) => m.seq));
console.log('laptop (never synced):', chat.sync({ user: 'cy', device: 'laptop', conversationId: conv }).messages.map((m) => m.seq));
// phone catches up: [ 4 ]
// laptop (never synced): [ 1, 2, 3, 4 ]

chat.markRead(conv, 'cy', 2);
console.log('seq 2 read by:', chat.readBy(conv, 2, members), '| seq 3 read by:', chat.readBy(conv, 3, members));
// seq 2 read by: [ 'ana', 'bo', 'cy' ] | seq 3 read by: [ 'ana', 'bo' ]
```

The retried send returned the original sequence number instead of storing a second copy; the phone, having acknowledged up to seq 3,
receives only seq 4 on its next sync, while the laptop that never synced gets everything; and receipts are answered from three
cursors, not from per-message records.

The decision to defend is **sequence numbers per conversation rather than globally unique, time-ordered message IDs** (Snowflake or
UUIDv7, Chapter 22), which would avoid needing a single owner per conversation. Time-ordered IDs from many generators are only
*roughly* ordered — two messages sent within the clock skew of their generators can sort differently from the order they were
accepted — and they can't reveal gaps: after message `…7a1f`, nothing says what should come next. A per-conversation counter gives a
total order that every device agrees on, makes "everything after 41" a precise query, and turns a missing number into a detectable
gap. The cost is that each conversation's sends go through one owner — which a chat can easily afford, because no conversation
between humans sends faster than one partition can sequence.

---

## 37.11 Questions

**1.** MCQ — What gives messages in a conversation one consistent order on every device? a) Server timestamps b) Client timestamps c) A per-conversation sequence number from a single owner d) Random IDs
<details><summary>Answer</summary>**c)** Timestamps suffer clock skew; a single owner's counter defines one order.</details>

**2.** MCQ — A sender's message is stored but the ack is lost; the client retries. What prevents a duplicate? a) The gateway b) Deduplicating by the client-generated message ID c) Timestamps d) Read receipts
<details><summary>Answer</summary>**b)** The service returns the existing sequence number for a repeated `clientMessageId`.</details>

**3.** MCQ — Where should a group message be stored? a) Once per member inbox b) Once, in the conversation's partition c) Only in the sender's outbox d) In the gateway
<details><summary>Answer</summary>**b)** Store once; fan out delivery, not storage.</details>

**4.** MCQ — How should read receipts be represented? a) One record per message per reader b) A per-member "read up to seq N" cursor c) A boolean per conversation d) Server logs
<details><summary>Answer</summary>**b)** One number per member per conversation, regardless of how many messages were read.</details>

**5.** MCQ — What happens to a message for a recipient whose devices are all offline? a) It's dropped b) It's stored; a push notification nudges the device, which syncs on open c) The sender must resend d) It waits in the gateway
<details><summary>Answer</summary>**b)** The log is the source of truth; push is only a nudge.</details>

**6.** MCQ — Why not broadcast every presence change to all contacts? a) Privacy only b) Presence changes are frequent and contact lists large, so broadcasting multiplies traffic enormously c) Presence can't be computed d) Gateways can't send them
<details><summary>Answer</summary>**b)** Push presence only to active viewers, debounce, and fetch lazily otherwise.</details>

**7.** Short — Why is a global sequence across all conversations unnecessary?
<details><summary>Answer</summary>Users only need order within a conversation. Per-conversation sequencing spreads the work across partitions, while a global sequencer would be a single bottleneck providing an order nobody uses.</details>

**8.** Short — Why must the server acknowledge a send only after storing it?
<details><summary>Answer</summary>So "sent" means durable: if the server crashes after acking but before storing, the client stops retrying and the message is lost.</details>

**9.** Short — How do sequence numbers enable gap detection?
<details><summary>Answer</summary>They're consecutive within a conversation, so a device holding 41 that receives 43 knows 42 is missing and fetches it via sync, rather than showing an incomplete conversation.</details>

**10.** Short — How does end-to-end encryption change storage and delivery?
<details><summary>Answer</summary>The server relays ciphertext, often encrypted per recipient device (or with group sender keys), so it can still sequence and deliver but can't search, moderate, or restore history server-side; key management becomes a core feature.</details>

**11.** Short — Why are delivery cursors tracked per device rather than per user?
<details><summary>Answer</summary>Each device connects, disconnects, and receives independently; a message delivered to the phone isn't on the laptop, so each device needs its own "delivered up to" position to sync correctly.</details>

**12.** Output — Using §37.2's model with 200 M DAU and 60 messages per user per day, what's the peak message rate (3× average), to the nearest thousand per second?
<details><summary>Answer</summary>**~417,000/s.** 200 M × 60 / 86,400 ≈ 138,900/s average × 3.</details>

**13.** Output — Using §37.10's `ConversationLog`, after the demo, `chat.send({ conversationId: 'grp-7', sender: 'cy', clientMessageId: 'c-1', body: 'ok' })` is called. What's its seq, and what does `readBy('grp-7', 4, ['ana', 'bo', 'cy'])` return afterwards?
<details><summary>Answer</summary>**seq 5; `[ 'bo', 'cy' ]`.** Bo sent seq 4 (so has read it); Cy sending seq 5 marks Cy as read up to 5; Ana's cursor is at 3.</details>

**14.** Output — A group has 500 members, each with 2 devices, 60% of devices online. How many real-time pushes does one message cause, and how many push notifications if each offline member (no device online) gets one?
<details><summary>Answer</summary>**~600 pushes; notifications depend on members with no online device.** 1,000 devices × 0.6 = 600 live pushes. If device presence is independent, a member has no online device with probability 0.4² = 0.16, so ~80 notifications.</details>

**15.** Output — 150 M concurrent connections; one availability zone holding a third of them fails, and clients reconnect uniformly over 60 seconds. What's the reconnect rate the remaining gateways must absorb?
<details><summary>Answer</summary>**~833,000 connections/s.** 50 M / 60 s — which is why reconnects need jitter, cheap resume tokens, and admission control (Chapter 9).</details>

**16.** Debug — Users report messages appearing in different orders on their phone and laptop in busy group chats. Messages are sorted by `sentAt` from the server that handled each send. Cause and fix?
<details><summary>Answer</summary>Different chat-service instances have skewed clocks, and devices may receive messages via different paths. Assign a per-conversation sequence number on the conversation's single owner (partition leader) and sort by it everywhere.</details>

**17.** Debug — After a gateway deploy, some users saw a burst of duplicate messages. Logs show clients resent unacknowledged messages on reconnect. What's missing?
<details><summary>Answer</summary>Server-side deduplication by `clientMessageId` (or clients generating a new ID per retry). The server should return the original seq for a repeated client message ID, so resends after a reconnect are harmless.</details>

**18.** Debug — The presence service is the largest consumer of gateway bandwidth and spikes whenever mobile networks are flaky. What do you change?
<details><summary>Answer</summary>Stop broadcasting every change: send presence only to subscribers currently viewing the relevant conversation or contact list, debounce online/offline transitions (e.g. require 30 s of absence before "offline"), coarsen "last seen", and derive presence from heartbeats with TTLs rather than connect/disconnect events.</details>

**19.** Debug — A user who reinstalled the app sees only messages received after reinstalling; older conversations are empty. What's wrong with the sync design?
<details><summary>Answer</summary>A new device has no cursors, and the server only delivers new messages (push path). New devices must pull recent history per conversation (bounded, e.g. last few hundred messages), with older history fetched on scroll — sync, not just live delivery.</details>

**20.** Code — Write `detectGaps(seqs, from)` that, given the sorted seqs a device holds and the seq it expects next, returns the ranges it must fetch.
<details><summary>Answer</summary>

```js
function detectGaps(seqs, from = 1) {
  const gaps = [];
  let expected = from;
  for (const s of seqs) {
    if (s > expected) gaps.push([expected, s - 1]);   // a hole before this message
    expected = Math.max(expected, s + 1);
  }
  return gaps;
}
console.log(detectGaps([1, 2, 5, 6, 9])); // [ [ 3, 4 ], [ 7, 8 ] ]
```
</details>

**21.** Code — Write `deliveryPlan(members, registry)` that groups online devices by gateway (so each gateway receives one batch) and lists members with no online device for push notifications.
<details><summary>Answer</summary>

```js
function deliveryPlan(members, registry) {
  const byGateway = new Map(), notify = [];
  for (const m of members) {
    const devices = registry.get(m) ?? [];                  // [{ device, gateway }]
    if (devices.length === 0) { notify.push(m); continue; }
    for (const { device, gateway } of devices) {
      if (!byGateway.has(gateway)) byGateway.set(gateway, []);
      byGateway.get(gateway).push(`${m}/${device}`);
    }
  }
  return { batches: Object.fromEntries(byGateway), notify };
}
const registry = new Map([['ana', [{ device: 'phone', gateway: 'gw1' }]], ['bo', [{ device: 'web', gateway: 'gw1' }, { device: 'phone', gateway: 'gw2' }]]]);
console.log(JSON.stringify(deliveryPlan(['ana', 'bo', 'cy'], registry)));
// {"batches":{"gw1":["ana/phone","bo/web"],"gw2":["bo/phone"]},"notify":["cy"]}
```
</details>

**22.** Code — Write `presence(heartbeats, user, { now, ttlMs })` returning `'online'` if any of the user's devices sent a heartbeat within the TTL, else `{ lastSeen }` rounded down to the minute.
<details><summary>Answer</summary>

```js
function presence(heartbeats, user, { now = Date.now(), ttlMs = 60_000 } = {}) {
  const times = (heartbeats.get(user) ?? []).map((h) => h.at);
  if (times.length === 0) return { lastSeen: null };
  const latest = Math.max(...times);
  if (now - latest <= ttlMs) return 'online';
  return { lastSeen: new Date(Math.floor(latest / 60_000) * 60_000).toISOString() }; // coarse: minutes only
}
const hb = new Map([['ana', [{ device: 'phone', at: Date.UTC(2026, 10, 15, 9, 41, 37) }]]]);
console.log(presence(hb, 'ana', { now: Date.UTC(2026, 10, 15, 10, 0, 0) })); // { lastSeen: '2026-11-15T09:41:00.000Z' }
```
</details>

**23.** Scenario — Product wants "message editing and deletion for everyone" within 15 minutes of sending. How do you fit it into the sequenced log?
<details><summary>Answer</summary>Don't mutate history in place: an edit or delete is a new event in the conversation with its own seq (`edit` or `delete` referencing the original seq), so it's ordered, delivered, and synced like any message, and devices apply it to their local copy. The server validates the time window and the sender, stores the latest content for sync of new devices (or applies edits during sync), and devices that were offline receive the edit after the original in order. With end-to-end encryption, the edit is an encrypted message too.</details>

**24.** Scenario — A customer-support product wants chat between customers and agents with transcripts retained for 7 years for compliance, full-text search for agents, and very different load (5 M conversations/day, short-lived). How would you adapt this design?
<details><summary>Answer</summary>Conversations are short and two-to-few-party, so the same sequenced, per-conversation log fits; retention changes storage: tier closed conversations to compressed object storage (Chapter 17) with legal-hold and immutability, and index transcripts into a search engine fed by CDC, partitioned by tenant (Chapter 18). End-to-end encryption is usually not used, because the business (and agents) must read transcripts. Add routing of conversations to agents (a queue with skills and capacity), and per-tenant isolation throughout.</details>

---

## Chapter Summary

- A chat system is a real-time delivery fast path over a durable, sequenced message log; correctness comes from storage and sync,
  not from the live connection.
- Size the gateway tier by concurrent connections, the message store by write rate and retention, and delivery by group fan-out.
- Order messages with per-conversation sequence numbers from a single owner, deduplicate sends by client message ID, and use the
  sequence to detect and fill gaps.
- Store each message once per conversation and fan out only delivery, batched per gateway; offline devices get a push nudge and
  sync from per-device cursors.
- Represent receipts as per-member read cursors and presence as heartbeat-derived state pushed only to active viewers, debounced and
  coarsened.
- Plan for gateway failures as reconnect storms, treat end-to-end encryption as a design change rather than a feature, and at 10×
  go regional for connections and tier storage aggressively.

---

## Practice Exercises

1. Extend §37.10's log with message edits and deletions as sequenced events, and show devices converge to the same state regardless of
   when they sync.
2. Build a WebSocket gateway and chat service (using Chapter 9's hub as a starting point) that delivers sequenced messages and resumes
   via sync after a disconnect.
3. Read the Signal protocol documentation's overview of multi-device and group messaging, and describe what the server stores for one
   group message.
4. Simulate presence for 100,000 users with flaky mobile connections, and measure presence messages per second with naive broadcast
   versus viewer-only subscriptions with debouncing.
5. Model storage tiering for a year of messages: hot SSD for 30 days, compressed object storage afterwards, and estimate cost and the
   latency of fetching old history.

---

## Mini Project: A Multi-Device Chat Backend

Build a chat backend that supports groups, multiple devices per user, and correct behaviour under disconnects and retries.

- WebSocket gateways (thin), a chat service assigning per-conversation sequence numbers atomically in PostgreSQL or a key-value store,
  and a delivery worker reading a message log.
- Idempotent sends by client message ID; per-device delivered cursors; sync with gap filling.
- Read receipts as per-member cursors and presence from heartbeats with TTLs, pushed only to active viewers.
- Offline notification stubs triggered for members with no connected device.
- A chaos test: two clients per user, random gateway kills, dropped acks, and reconnects during a 10-minute burst — asserting every device
  ends with every message exactly once, in the same order.

---

**Next:** [Chapter 38 — Design a Notification System →](38-notification-system.md)
