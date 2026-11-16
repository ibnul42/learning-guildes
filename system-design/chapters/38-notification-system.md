# Chapter 38 — Design a Notification System

## Learning Objectives

- Scope a multi-channel notification platform used by many internal services
- Separate priorities so a marketing campaign can never delay a one-time passcode
- Guarantee users see each notification at most once across retries, replays, and multiple producers
- Apply preferences, quiet hours, frequency caps, and digests consistently in one policy layer
- Handle third-party providers' rate limits, failures, and feedback (bounces, invalid tokens)

---

## 38.1 Requirements and Scope

Every product sends notifications — order shipped, someone replied, password changed, a weekly digest, a sale. Without a
platform, each team integrates push, email, and SMS providers on its own, with inconsistent preferences, no shared
rate limits, and users getting the same message three times. A notification system centralises that: services describe
*what happened*, and the platform decides *whether, when, how, and in what words* to tell the user.

**Functional requirements**

- Accept notification requests from internal services (and from scheduled campaigns): recipient, category, template, data.
- Deliver over push (iOS/Android), email, SMS, and in-app inbox.
- Respect user preferences: per-category opt-outs, channel choices, quiet hours in the user's time zone.
- Apply frequency caps and collapse low-priority notifications into digests.
- Render localised templates; track delivery status (sent, delivered, opened, bounced, failed).

**Out of scope**: authoring tools for marketing content, A/B testing of copy (a consumer of this platform), the in-app UI.

**Non-functional requirements**

| Requirement | Target |
|---|---|
| Latency | Critical (passcodes, security alerts): p99 < 5 s to the provider; normal: < 30 s; campaigns: minutes to hours |
| Duplicates | A user should never receive the same notification twice |
| Loss | Critical and transactional notifications must not be lost; marketing may be dropped under policy |
| Isolation | Campaign volume must never delay critical notifications |
| Scale | ~1 B notifications per day; campaigns of tens of millions within minutes |

---

## 38.2 Estimation

```js
// Notifications: steady load is modest; campaigns and provider limits are what size the system
const perDay = 1e9, campaignRecipients = 80e6, campaignWindowMin = 30;
console.log(`steady ~${Math.round(perDay / 86_400 / 1000)}k/s avg`);
console.log(`campaign: ${campaignRecipients / 1e6} M emails in ${campaignWindowMin} min → ~${Math.round(campaignRecipients / (campaignWindowMin * 60) / 1000)}k/s`);
const providerLimit = 20_000;   // a hypothetical per-account send rate at one email provider
console.log(`at ${providerLimit / 1000}k/s per provider account, the campaign needs ${(campaignRecipients / providerLimit / 60).toFixed(0)} min on one account`);
// steady ~12k/s avg
// campaign: 80 M emails in 30 min → ~44k/s
// at 20k/s per provider account, the campaign needs 67 min on one account
```

The numbers say something unusual: our own processing is not the bottleneck. **Third-party providers are.** Push services,
email providers, and SMS carriers all impose rate limits, quotas, and costs (SMS especially — cents per message, which turns
volume into money fast). The design therefore centres on queueing, prioritising, and pacing work against external limits,
and on never spending scarce capacity on the wrong notifications.

---

## 38.3 API and Data Model

Producers describe events; they don't choose channels or format messages:

```text
POST /v1/notifications
Idempotency-Key: order-77:shipped               ← derived from the business event, not random
{ "recipient": "user-42", "category": "orders", "template": "order_shipped",
  "data": { "orderId": "77", "carrier": "DHL", "eta": "2026-11-18" }, "priorityHint": "high" }
→ 202 { "notificationId": "ntf_…", "status": "accepted" }
```

The idempotency key comes from the **business event** (`order-77:shipped`), so if the order service, its retry logic, and a
replayed event stream all request the same notification, the platform recognises them as one (§38.5).

| Data | Notes |
|---|---|
| **Preferences** (per user) | Opt-outs per category, enabled channels, quiet hours, time zone, locale |
| **Device tokens / addresses** | Push tokens per device, email, phone; with validity status (invalidated on provider feedback) |
| **Templates** | Per template × channel × locale, versioned |
| **Notification log** | Each notification: key, recipient, category, decision, per-channel attempts and statuses |
| **Counters** | Per user × category per day (frequency caps), short TTL |
| **Dedupe keys** | Idempotency keys seen recently (days), TTL-bounded |

---

## 38.4 High-Level Design

```text
producers ─▶ notification API ─▶ dedupe ─▶ policy engine ──┬─▶ critical queue ─▶ push/SMS/email workers ─┐
 (services,   (validate, 202)   (keys)    (prefs, quiet,  ├─▶ high queue     ─▶ …                        ├─▶ providers
  campaigns)                               caps, digest)   ├─▶ normal queue   ─▶ …                        │  (APNs, FCM,
                                                           ├─▶ bulk queue     ─▶ paced workers            │   email, SMS)
                                                           └─▶ deferred store (quiet hours, digests) ─────┘
                                                                                                   provider webhooks
in-app inbox ◀─────────────────────────────────── renderer (template × locale) ◀──┘          (delivered, bounced,
                                                                                               token invalid) ─▶ status log
```

The pipeline is asynchronous end to end: the API accepts and returns `202`, so producers never wait on providers. Each stage
has one job — dedupe, decide, enqueue by priority, render, send, record — and each queue can be scaled and rate-limited
independently.

---

## 38.5 Deep Dive: Never Twice

Duplicate notifications are the most visible failure of the platform — two identical SMS passcodes, three "your order shipped"
emails — and they come from everywhere: producers retrying after timeouts, event streams redelivering (Chapter 24), two services
both deciding to notify about the same thing, workers retrying a send whose acknowledgement was lost.

Defences at each stage:

- **At ingestion: business idempotency keys.** Every request carries a key derived from the triggering event. The API records keys
  in a TTL-bounded store and treats a repeat as the same notification (returning the original `notificationId`). This also merges
  overlapping producers if they derive the same key.
- **Between stages: at-least-once with idempotent consumers.** Queues redeliver; each worker checks the notification's per-channel
  status before acting (`already sent on email → skip`), using the inbox pattern from Chapter 23.
- **At providers: provider idempotency where available.** Some providers accept an idempotency key or a deduplication ID; pass the
  notification ID through. Where they don't, record "attempting" before the call and reconcile ambiguous outcomes (a timeout after
  sending) via provider status APIs or webhooks, rather than blindly resending.
- **Collapse keys on devices.** Push services support a collapse/thread identifier so a newer notification replaces an older one on
  the lock screen (`order-77-status`) — the device-side backstop for updates that supersede each other.

There's a trade-off at the provider boundary that must be decided per category: after an ambiguous failure, **resend** (risk a
duplicate) or **don't** (risk a loss)? For a passcode, a duplicate is harmless and a loss blocks login — resend. For a marketing
email, a loss is fine and a duplicate is annoying — don't.

---

## 38.6 Deep Dive: Preferences, Quiet Hours, Caps, and Digests

Rules about *whether and when* to notify must be applied **in one place**, or every team implements them differently and users
experience them inconsistently. The policy engine evaluates each notification against:

1. **Opt-outs** per category — except categories that can't be disabled (security alerts, legally required notices).
2. **Frequency caps** per user per category (at most N social notifications per day), so a busy thread doesn't produce 200 pushes.
3. **Channel selection** — the category's allowed channels, filtered by what the user enabled and what addresses are valid; usually
   the first available channel, and several for critical categories.
4. **Quiet hours** in the user's own time zone — deferring non-critical notifications until morning.
5. **Digests** — when a cap is reached, or for categories configured that way, notifications are batched into a periodic summary
   ("12 new likes on your photo") instead of being sent individually or dropped.

The real-world example in §38.10 implements this decision function. Two details matter in practice:

- **Time zones are per user, not per server.** Store the user's IANA time zone and compute local time at decision time; daylight
  saving changes shift quiet hours automatically. Deferred notifications are stored with a "deliver after" time and released by a
  scheduler (Chapter 45).
- **Decide late.** A notification deferred overnight should be **re-evaluated** when released — the user may have opted out, the order
  may have been cancelled, or the event may no longer be relevant ("your driver is arriving" is useless eight hours later). Store
  expiry times for time-sensitive categories.

---

## 38.7 Deep Dive: Priorities and Providers

**Priority isolation** is the platform's most important property. If an 80-million-recipient campaign and a password-reset code share
a queue and a provider account, the code waits behind 80 million emails. Isolation is structural:

- **Separate queues per priority** (critical, high, normal, bulk), each with its own consumers, so a backlog in one never delays another.
- **Reserved provider capacity**: dedicated provider accounts, sending IPs, or rate-limit allocations for critical traffic, with bulk
  traffic paced to stay under its own share.
- **Bulk pacing**: campaigns are sent by paced workers using token buckets (Chapter 29) sized to the provider's limit and to warm-up
  constraints (email providers and mailbox operators throttle senders whose volume jumps suddenly).

**Provider handling:**

| Concern | Approach |
|---|---|
| Rate limits | Client-side token buckets per provider account; honour `429`/`Retry-After` |
| Transient failures | Retries with jittered backoff and a retry budget (Chapter 28), bounded by the notification's expiry |
| Provider outage | Failover to a secondary provider for email and SMS (keep both warm with a share of traffic); push has no alternative provider, so queue and retry |
| Invalid push tokens | Remove tokens the provider reports as unregistered — sending to dead tokens wastes quota and can harm sender reputation |
| Email bounces and complaints | Suppress addresses that hard-bounce or mark mail as spam; required to keep deliverability |
| SMS cost and fraud | Per-country routing and pricing, rate limits per destination, and protection against SMS pumping fraud (attackers triggering passcodes to premium numbers) |

> 💼 **Interview** — The two follow-ups that separate strong answers are "what stops a campaign from delaying password resets?"
> (separate priority queues and reserved provider capacity) and "how do you avoid sending duplicates?" (business idempotency keys,
> idempotent workers, and provider-level dedupe or collapse keys).

---

## 38.8 Failure Modes, Bottlenecks, and 10×

| Failure | Effect | Mitigation |
|---|---|---|
| Push provider degraded | Push delays | Queue with retries until expiry; fall back to in-app inbox or email for important categories |
| Email provider outage | Email delays | Fail over to a secondary provider |
| Policy engine's preference store down | Can't evaluate opt-outs | Fail safe: hold non-critical notifications; send critical ones with cached preferences |
| Runaway producer (a bug sends 1 M notifications) | User spam, cost | Per-producer rate limits and quotas at the API; per-user caps in policy; kill switch per producer and template |
| Campaign mis-targeted | Wrong audience | Staged sends (1% first, review metrics), cancellation that drains the bulk queue |

**At 10×** (10 B per day, campaigns of hundreds of millions): the platform's own tiers scale horizontally; the constraints become
provider quotas and deliverability. More provider accounts, dedicated sending infrastructure for email, regional SMS routes, and
smarter suppression (not sending to users who never engage) matter more than more workers. The notification log grows to trillions
of rows a year — partition it by time, keep detailed records for weeks, and aggregate the rest into analytics.

---

## 38.9 Common Mistakes and Best Practices

**Mistake: one queue for all notifications.** Campaign backlogs delay passcodes and security alerts.

**Mistake: random idempotency keys.** Keys generated per request don't merge retries from different producers or replayed events;
derive them from the business event.

**Mistake: preferences checked in each producing service.** Rules drift, and users get inconsistent behaviour; centralise policy.

**Mistake: quiet hours in server time.** Users in other time zones get woken up; use each user's IANA time zone.

**Mistake: deferred notifications sent without re-evaluation.** Stale or cancelled events reach users hours later.

**Mistake: ignoring provider feedback.** Dead tokens and bounced addresses waste quota and damage deliverability.

**Mistake: no per-producer limits.** One buggy service can spam every user and run up an SMS bill.

**Best practice: producers describe events; the platform decides delivery.** Channels, timing, wording, and limits live in one place.

**Best practice: isolate priorities in queues and provider capacity.** Critical traffic gets reserved paths that bulk traffic can't touch.

**Best practice: decide per category between duplicate and loss.** Passcodes resend on ambiguity; marketing doesn't.

---

## 38.10 Real-World Example: A Notification Policy Engine

This decision function is the heart of the platform: every notification passes through it after deduplication, and it returns one of
send, defer, digest, or drop — with the channels to use. Categories carry their own priority, caps, channels, and whether they may
bypass quiet hours; users carry opt-outs, enabled channels, quiet hours, and a time zone.

```js
// notification-policy.mjs — one place that decides whether, when, and how a notification is sent:
// deduplication, opt-outs, quiet hours in the user's time zone, frequency caps, and channel choice.

const CATEGORY = {
  security: { priority: 'critical', bypassQuietHours: true, capPerDay: Infinity, channels: ['push', 'sms', 'email'] },
  orders:   { priority: 'high', bypassQuietHours: false, capPerDay: 20, channels: ['push', 'email'] },
  social:   { priority: 'normal', bypassQuietHours: false, capPerDay: 10, channels: ['push', 'inapp'] },
  marketing:{ priority: 'low', bypassQuietHours: false, capPerDay: 1, channels: ['email'] },
};

const localHour = (date, timeZone) =>
  Number(new Intl.DateTimeFormat('en-GB', { hour: 'numeric', hourCycle: 'h23', timeZone }).format(date));

export function decide(n, { prefs, sentToday, seenKeys, now }) {
  const cat = CATEGORY[n.category];
  if (!cat) return { action: 'reject', reason: `unknown category ${n.category}` };
  if (seenKeys.has(n.idempotencyKey)) return { action: 'drop', reason: 'duplicate' };       // same event twice
  if (prefs.optedOut.includes(n.category) && cat.priority !== 'critical') {
    return { action: 'drop', reason: 'user opted out' };                                     // security can't be disabled
  }
  if ((sentToday[n.category] ?? 0) >= cat.capPerDay) {
    return cat.priority === 'low' ? { action: 'drop', reason: 'daily cap reached' }
      : { action: 'digest', reason: 'daily cap reached: batch into the evening digest' };
  }
  // Channels the category allows, in order, that the user has enabled and can receive
  const channels = cat.channels.filter((c) => prefs.channels[c]);
  if (channels.length === 0) return { action: 'drop', reason: 'no enabled channel' };

  const hour = localHour(now, prefs.timeZone);
  const quiet = prefs.quietHours && (hour >= prefs.quietHours.start || hour < prefs.quietHours.end);
  if (quiet && !cat.bypassQuietHours) {
    return { action: 'defer', reason: `quiet hours (${hour}:00 local)`, until: `${prefs.quietHours.end}:00 local`, channels };
  }
  return { action: 'send', priority: cat.priority, channels: cat.priority === 'critical' ? channels : [channels[0]] };
}

// ── Demo: one user in Lisbon, evaluated at 23:30 local time ───────────────────────────
const prefs = { timeZone: 'Europe/Lisbon', quietHours: { start: 22, end: 8 }, optedOut: ['marketing'],
  channels: { push: true, email: true, sms: true, inapp: true } };
const now = new Date('2026-11-15T23:30:00Z');                     // Lisbon is UTC+0 in November
const ctx = { prefs, sentToday: { social: 10 }, seenKeys: new Set(['evt-41']), now };
for (const n of [
  { idempotencyKey: 'evt-1', category: 'security' },              // new-device login
  { idempotencyKey: 'evt-2', category: 'orders' },                // "your parcel shipped"
  { idempotencyKey: 'evt-3', category: 'social' },                // 11th like notification today
  { idempotencyKey: 'evt-4', category: 'marketing' },
  { idempotencyKey: 'evt-41', category: 'orders' },               // replayed event
]) {
  const d = decide(n, ctx);
  console.log([n.category.padEnd(9), d.action.padEnd(6), d.channels && `[${d.channels}]`, d.reason].filter(Boolean).join(' '));
}
// security  send   [push,sms,email]
// orders    defer  [push,email] quiet hours (23:00 local)
// social    digest daily cap reached: batch into the evening digest
// marketing drop   user opted out
// orders    drop   duplicate
```

At 23:30 in Lisbon, the security alert goes out immediately on every channel; the shipping update waits until 08:00; the eleventh
social notification of the day becomes part of the evening digest; marketing is dropped because the user opted out; and the replayed
event is recognised as a duplicate.

The decision to defend is **putting all of these rules in the platform's policy engine instead of in the services that trigger
notifications**. Producing teams know their domain best, and letting each decide "this one's important, send it now" seems
reasonable. But users experience the *sum* of all producers: quiet hours that some teams honour and others don't, caps that apply to
one team's notifications but not another's, and an opt-out that stops marketing email but not the "recommended for you" push a
different team built. Only a central policy sees everything a user is about to receive, so only it can enforce a coherent experience —
one quiet-hours rule, one daily budget, one set of preferences. Producers keep control where it belongs: they choose the category, the
template, and the data, and the category's configuration encodes the priority they negotiated with the platform.

---

## 38.11 Questions

**1.** MCQ — What most directly prevents a marketing campaign from delaying password-reset codes? a) Faster workers b) Separate priority queues with reserved provider capacity c) Bigger servers d) Deduplication
<details><summary>Answer</summary>**b)** Isolation must be structural: queues and provider capacity the bulk traffic can't consume.</details>

**2.** MCQ — Where should a notification's idempotency key come from? a) A random UUID per API call b) The triggering business event (e.g. `order-77:shipped`) c) The provider d) The timestamp
<details><summary>Answer</summary>**b)** Then retries, replays, and overlapping producers all map to the same key.</details>

**3.** MCQ — A notification was deferred overnight by quiet hours. What should happen when it's released? a) Send it as-is b) Re-evaluate it against current preferences, relevance, and expiry c) Drop it d) Send it twice
<details><summary>Answer</summary>**b)** Things change in eight hours: opt-outs, cancellations, and time-sensitive content.</details>

**4.** MCQ — What should happen when a push provider reports a device token as unregistered? a) Retry later b) Remove the token and stop sending to it c) Switch to SMS for all users d) Ignore it
<details><summary>Answer</summary>**b)** Dead tokens waste quota and can harm sender reputation.</details>

**5.** MCQ — After an ambiguous provider failure (timeout after the request was sent), what should the platform do for a one-time passcode? a) Never resend b) Resend — a duplicate is harmless, a loss blocks login c) Switch to email d) Wait a day
<details><summary>Answer</summary>**b)** Per category, choose between duplicate and loss; for passcodes, loss is worse.</details>

**6.** MCQ — Why compute quiet hours from the user's time zone rather than the server's? a) Servers are in UTC b) "Night" is local to each user; server time wakes users in other zones c) Performance d) Providers require it
<details><summary>Answer</summary>**b)** Store each user's IANA time zone and compute their local hour at decision time.</details>

**7.** Short — Why should producers describe events rather than choose channels and wording?
<details><summary>Answer</summary>So preferences, quiet hours, caps, channel choice, templates, and localisation are applied consistently in one place, and producers don't each re-implement (and diverge on) delivery rules.</details>

**8.** Short — What is a digest, and when is it used?
<details><summary>Answer</summary>A periodic summary that batches many low-priority notifications ("12 new likes") into one message, used when a category's frequency cap is reached or when the category is configured for digests.</details>

**9.** Short — What are push collapse keys for?
<details><summary>Answer</summary>They let a newer notification replace an older one with the same key on the device (e.g. successive order-status updates), so the lock screen shows the latest state instead of a stack — a device-side guard against duplicates and noise.</details>

**10.** Short — Why are third-party providers the real bottleneck?
<details><summary>Answer</summary>Internal processing scales horizontally, but providers impose rate limits, quotas, warm-up constraints, and per-message costs, so throughput and reliability depend on pacing, prioritising, and failing over across them.</details>

**11.** Short — Name two protections against a runaway producer.
<details><summary>Answer</summary>Per-producer rate limits and quotas at the API; per-user frequency caps in policy; a kill switch per producer or template; anomaly alerts on volume.</details>

**12.** Output — Using §38.10's `decide` with the same context, what is the decision for `{ idempotencyKey: 'evt-9', category: 'security' }` if the user had opted out of `security`?
<details><summary>Answer</summary>**send on push, sms, email.** Security is critical, so the opt-out check is skipped and critical notifications use every enabled channel, regardless of quiet hours.</details>

**13.** Output — Same function and preferences, but `now = new Date('2026-07-15T21:30:00Z')` (Lisbon is UTC+1 in summer). What's the decision for an `orders` notification?
<details><summary>Answer</summary>**defer** — the local time is 22:30, inside the 22:00–08:00 quiet hours.</details>

**14.** Output — An SMS costs $0.008 on average. A bug triggers 1.2 million passcode SMS in an hour before a kill switch is used. What did it cost?
<details><summary>Answer</summary>**~$9,600.** 1,200,000 × 0.008 — the reason for per-producer quotas and SMS-specific rate limits.</details>

**15.** Output — A campaign of 30 million emails is paced at 15,000 per second across two provider accounts (7,500/s each). How long does it take?
<details><summary>Answer</summary>**~33 minutes.** 30,000,000 / 15,000 = 2,000 s.</details>

**16.** Debug — Users report receiving two identical "your order shipped" emails. Both the order service and the warehouse service send a notification when a parcel ships, each with a random idempotency key. What's the fix?
<details><summary>Answer</summary>Derive the idempotency key from the business event (`order-77:shipped`), so both producers' requests deduplicate to one notification — and agree which service owns that event, so only one produces it.</details>

**17.** Debug — During a Black Friday campaign, two-factor codes took 15 minutes to arrive, and many users couldn't log in. Codes and campaign emails share a queue and provider account. What do you change?
<details><summary>Answer</summary>Separate priority queues with their own consumers, reserved provider capacity (separate account or allocation) for critical traffic, pacing for bulk sends below their share, and an SLO on critical-queue latency with alerts.</details>

**18.** Debug — An email provider started rejecting a growing share of mail, and deliverability dropped for all users. Bounce webhooks were being received but not processed. What happened?
<details><summary>Answer</summary>Hard-bounced and complaining addresses kept receiving mail, damaging sender reputation until mailbox operators throttled or rejected it. Process bounce and complaint feedback into a suppression list, stop sending to those addresses, and monitor bounce and complaint rates.</details>

**19.** Debug — Users in Australia complain about notifications at 3 a.m., though quiet hours are configured. The policy engine uses `new Date().getHours()`. Cause and fix?
<details><summary>Answer</summary>`getHours()` uses the server's time zone, not the user's. Store each user's IANA time zone and compute local time with `Intl.DateTimeFormat` (or a time-zone library) at decision time.</details>

**20.** Code — Write `businessKey(event)` that derives an idempotency key from an event's type, entity, and the state transition, so producers of the same event agree.
<details><summary>Answer</summary>

```js
const businessKey = ({ type, entityType, entityId, toState }) =>
  `${entityType}:${entityId}:${type}${toState ? `:${toState}` : ''}`.toLowerCase();
console.log(businessKey({ type: 'StatusChanged', entityType: 'Order', entityId: '77', toState: 'Shipped' })); // order:77:statuschanged:shipped
```
Deliberately excludes timestamps and random values: the same business event must always produce the same key.</details>

**21.** Code — Write `nextQuietEnd(now, timeZone, endHour)` returning the next moment when local time reaches `endHour:00` in `timeZone` (scanning forward in 15-minute steps is acceptable).
<details><summary>Answer</summary>

```js
function nextQuietEnd(now, timeZone, endHour) {
  const fmt = new Intl.DateTimeFormat('en-GB', { hour: 'numeric', minute: 'numeric', hourCycle: 'h23', timeZone });
  for (let t = now.getTime(); t < now.getTime() + 2 * 86_400_000; t += 15 * 60_000) {
    const [h, m] = fmt.format(new Date(t)).split(':').map(Number);
    if (h === endHour && m === 0) return new Date(t);
  }
  throw new Error('no quiet-hours end found within 48 h');
}
console.log(nextQuietEnd(new Date('2026-11-15T23:30:00Z'), 'Europe/Lisbon', 8).toISOString()); // 2026-11-16T08:00:00.000Z
```
Stepping through local time handles daylight-saving transitions without special cases.</details>

**22.** Code — Write `pickProvider(providers, { channel })` that returns the first healthy provider for a channel whose token bucket has capacity, or `null` to queue.
<details><summary>Answer</summary>

```js
function pickProvider(providers, { channel }) {
  for (const p of providers.filter((x) => x.channel === channel)) {
    if (p.health !== 'healthy') continue;          // circuit open or degraded (Chapter 28)
    if (p.bucket.tryTake()) return p.name;         // within this account's rate limit (Chapter 29)
  }
  return null;                                     // all saturated or unhealthy: stay queued, retry later
}
const bucket = (n) => ({ tryTake() { return n-- > 0; } });
console.log(pickProvider([
  { name: 'mailA', channel: 'email', health: 'healthy', bucket: bucket(0) },
  { name: 'mailB', channel: 'email', health: 'healthy', bucket: bucket(5) },
], { channel: 'email' })); // mailB
```
</details>

**23.** Scenario — Product wants every notification to also appear in an in-app inbox, with read/unread state synced across devices. How do you add it?
<details><summary>Answer</summary>Treat the inbox as a channel that always receives (subject to policy) and is stored durably: one inbox record per user notification, partitioned by user and ordered by time, with read state as a per-user cursor or per-item flag (Chapter 37's receipts approach). Devices sync the inbox like chat history; push notifications deep-link to inbox items; marking read on one device propagates to others. Retain inbox items per category policy, and let the inbox absorb notifications that were deferred or digested elsewhere.</details>

**24.** Scenario — A fraud team notices thousands of passcode SMS sent to premium-rate numbers in a few countries, triggered from the signup form. What do you put in place?
<details><summary>Answer</summary>This is SMS pumping fraud. Rate-limit passcode requests per IP, device, phone number, and destination prefix (Chapter 29); block or require extra verification for high-risk countries and number ranges; add bot protection to the form; prefer non-SMS methods (authenticator apps, email, push) where possible; set per-country daily spend caps with alerts; and monitor conversion — passcodes sent but never entered are the signature.</details>

---

## Chapter Summary

- A notification platform turns producers' events into delivered messages; its constraints are third-party provider limits, cost,
  and deliverability rather than its own processing.
- Producers describe events with business-derived idempotency keys; the platform deduplicates at ingestion, makes every stage
  idempotent, and uses provider dedupe or collapse keys at the edge.
- One central policy engine applies opt-outs, frequency caps, channel choice, quiet hours in each user's time zone, and digests — and
  re-evaluates deferred notifications when they're released.
- Priorities are isolated structurally: separate queues and reserved provider capacity so bulk campaigns never delay critical traffic.
- Providers need pacing, retries within expiry, failover where alternatives exist, and feedback processing for invalid tokens,
  bounces, and complaints.
- Protect users and budgets from runaway producers and SMS fraud with quotas, caps, kill switches, and spend monitoring.

---

## Practice Exercises

1. Extend §38.10's policy with per-category expiry and re-evaluation on release, and test a deferred "driver arriving" notification
   that should be dropped.
2. Build the ingestion API with a TTL-bounded idempotency store and verify that 1,000 concurrent duplicate requests produce one
   notification.
3. Read the documentation for APNs or FCM on collapse identifiers, priorities, and error responses for invalid tokens, and implement
   handling for each error type.
4. Implement a paced bulk sender with a token bucket per provider account, and measure how close it stays to the configured rate under
   bursts.
5. Design the schema and retention policy for the notification log at 1 B rows per day, and estimate storage for 30 days of detail plus
   a year of aggregates.

---

## Mini Project: A Notification Platform

Build a small notification platform that two or three toy services use, with the isolation and correctness properties from this chapter.

- An ingestion API with business idempotency keys and per-producer rate limits.
- The §38.10 policy engine with user preferences, time-zone-aware quiet hours, caps, digests, and re-evaluation of deferred notifications.
- Priority queues (critical, normal, bulk), channel workers with fake providers that rate-limit, fail, and report invalid tokens.
- A digest job, a deferred-release scheduler, and a status log with per-channel attempts.
- A test that runs a large simulated campaign while critical passcodes are requested, asserting critical latency stays under target and no
  user receives a duplicate.

---

**Next:** [Chapter 39 — Design a Distributed Key-Value Store →](39-key-value-store.md)
