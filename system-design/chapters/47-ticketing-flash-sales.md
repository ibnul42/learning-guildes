# Chapter 47 — Design Ticketing and Flash Sales

## Learning Objectives

- Scope an on-sale where demand exceeds supply by orders of magnitude, and size the spike against what the inventory path can sustain
- Sell limited inventory under extreme contention without overselling, using conditional updates, holds, and sharded stock
- Design a waiting room that protects the backend, admits buyers at a sustainable rate, and is fair to people rather than to bots
- Tie holds to checkout and payment so abandoned carts return stock and paid orders never lose their tickets
- Degrade gracefully, communicate honestly, and survive the minutes that matter most

---

## 47.1 Requirements and Scope

A ticketing on-sale — a stadium tour, a festival, a limited sneaker drop — is the most hostile traffic pattern in this book. Millions of
people arrive at the same second, for inventory that will be gone in minutes, and every one of them refreshes when the page is slow. The
system is idle for weeks and then has to be right, fast, and fair for ten minutes in front of an audience that's furious if it isn't.

**Functional requirements**

- Browse an event; see seat maps (reserved seating) or ticket types (general admission) with availability.
- Select seats or quantities and **hold** them while checking out; pay; receive tickets.
- Queue fairly when demand exceeds what the system admits at once; see queue position and an honest estimate.
- Enforce per-buyer limits; resist bots and scalpers.

**Out of scope**: event setup, ticket transfer and resale markets, venue entry scanning, payment internals (Chapter 43).

**Non-functional requirements**

| Requirement | Target |
|---|---|
| Correctness | **Never** sell a seat twice or more tickets than exist |
| Spike tolerance | ~2 million arrivals in two minutes, 40× more buyers than tickets, without falling over |
| Fairness | Position doesn't depend on refreshing, arriving minutes early, or running a faster bot |
| Latency | Holds succeed or fail in < 1 s for admitted buyers; queue pages are static and served from the edge |
| Honesty | Buyers learn quickly when inventory is gone, instead of waiting in a queue for nothing |

The first row is non-negotiable and the last is often forgotten. An oversold seat is a refund, a furious customer, and a news story; a
queue that holds a million people for an hour after the last ticket sold is a different news story.

---

## 47.2 Estimation

```js
// Ticketing on-sale: demand vs supply, and the rate the inventory path actually needs to handle
const fans = 2_000_000, seats = 50_000, arrivalWindowSec = 120;     // most arrive in the 2 minutes around opening
const pollEverySec = 2, checkoutMinutes = 5, conversion = 0.7;
const arrivalsPerSec = fans / arrivalWindowSec;
const naiveRps = fans / pollEverySec;                                // everyone refreshing the seat page
console.log(`demand/supply ${fans / seats}:1 | ~${Math.round(arrivalsPerSec).toLocaleString('en-US')} arrivals/s | ~${(naiveRps / 1e6).toFixed(0)} M req/s if everyone just refreshes`);
const admitPerMin = 3_000;                                           // sized to what checkout + payment sustain
const sellOutMin = seats / (admitPerMin * conversion * 2.5);         // ~2.5 tickets per admitted buyer
console.log(`admit ${admitPerMin.toLocaleString('en-US')}/min → ~${Math.round(admitPerMin * checkoutMinutes).toLocaleString('en-US')} in checkout at once; sells out in ~${sellOutMin.toFixed(0)} min; last in queue admitted after ~${Math.round(fans / admitPerMin / 60)} h`);
// demand/supply 40:1 | ~16,667 arrivals/s | ~1 M req/s if everyone just refreshes
// admit 3,000/min → ~15,000 in checkout at once; sells out in ~10 min; last in queue admitted after ~11 h
```

What the numbers decide:

- **A million requests per second of refreshing** is not a load to serve; it's a load to *prevent*. The waiting room's first job is to turn
  refreshing into one cheap, cacheable poll.
- **The inventory path only needs a few thousand buyers at a time.** Admitting 3,000 per minute keeps ~15,000 in checkout — a load the
  inventory store and payment provider can handle comfortably. The rest of the two million wait in a queue that costs almost nothing.
- **Most of the queue will never buy anything.** At 40:1, the event sells out in ~10 minutes while the queue would take ~11 hours to drain.
  Telling the remaining 1.9 million people "sold out" the moment it's true is a requirement, not a courtesy.
- **Contention concentrates.** Reserved-seat buyers all want the same front sections; general-admission buyers all decrement one number.
  Both are hot spots that ordinary scaling doesn't fix (§47.5).

---

## 47.3 API and Data Model

```text
GET  /events/{id}                         static page + availability summary (cached at the edge, refreshed every few seconds)
POST /events/{id}/queue                   → { queueId, position, estimateSec }   (only before or during the on-sale)
GET  /queue/{queueId}                     → { position, estimateSec } | { admitted: token } | { soldOut: true }
POST /events/{id}/holds                   Authorization: admission token
                                          { seats: [...] } or { ticketType, qty }  → { holdId, expiresAt } | 409 unavailable
POST /holds/{holdId}/checkout             { paymentMethod, idempotencyKey }  → order
```

```sql
-- Reserved seating: one row per seat. The state machine is available → held → sold, with holds expiring back.
CREATE TABLE seats (
  event_id        bigint NOT NULL,
  seat_id         text   NOT NULL,                       -- e.g. 'FLOOR-A-12-7'
  section         text   NOT NULL,
  price_cents     int    NOT NULL,
  state           text   NOT NULL CHECK (state IN ('available', 'held', 'sold')),
  hold_id         uuid,
  hold_expires_at timestamptz,
  order_id        uuid,
  PRIMARY KEY (event_id, seat_id)
);

-- Hold several seats atomically: succeeds only if every one is free (or its hold expired).
-- The application compares the returned row count to the number requested and rolls back on a shortfall.
UPDATE seats
SET state = 'held', hold_id = $1, hold_expires_at = now() + interval '5 minutes'
WHERE event_id = $2 AND seat_id = ANY($3)
  AND (state = 'available' OR (state = 'held' AND hold_expires_at < now()))
RETURNING seat_id;
```

The `WHERE` clause is the whole correctness story for reserved seats: it's a **compare-and-set** (Chapter 15) that only changes rows that
are still free, in one statement, under the row locks the database takes anyway. Two buyers racing for seat `FLOOR-A-12-7` both run it; one
gets the row back, the other gets nothing and is shown a different seat.

**General admission** has no seats — just "2,000 floor tickets" — and that's harder, because every buyer updates the **same row**. §47.5 and
§47.10 split it.

---

## 47.4 High-Level Design

```text
buyers ─▶ CDN (static event page, queue page, availability JSON cached ~2 s) ─────────────────────────┐
   │                                                                                                  │
   └─▶ waiting room (edge + queue service): random order for early arrivals, FIFO after;              │
         admits N/min, issues signed admission tokens ─────────────────┐                              │
                                                                       ▼                              │
                                   inventory service (verifies token) ─▶ seat / stock store (holds)   │
                                                                       │                              │
                                   checkout ─▶ payments (Chapter 43) ─▶ confirm hold → sold ─▶ tickets│
                                                                       │                              │
                                   hold reaper: expired holds → available ─▶ availability summary ────┘
```

- **The edge absorbs the crowd.** Event pages, queue pages, and availability summaries are static or cached for a couple of seconds at the
  CDN (Chapter 6), so millions of viewers cost the origin a handful of requests.
- **The waiting room admits at a fixed rate**, sized to what checkout and payment sustain, and hands each admitted buyer a **signed, expiring
  token**. The inventory service accepts only requests carrying a valid token, so nothing reaches it except admitted buyers — even if someone
  finds its URL.
- **The inventory service** runs holds as conditional updates against the seat or stock store, and confirms them after payment.
- **The hold reaper** returns expired holds to stock, and the availability summary feeds back to the edge so "sold out" propagates within
  seconds — including to the queue, which stops admitting and tells everyone.

---

## 47.5 Deep Dive: Inventory Under Contention

**Reserved seats** contend per seat, and the compare-and-set update in §47.3 handles it: row locks are held for one short statement, buyers
mostly want *different* seats, and a failed hold just means "pick another". Two refinements matter at on-sale scale:

- **Don't let buyers pick from a stale map.** If 15,000 people see the same seat map and all click the best seat, one wins and 14,999 fail.
  Offering **best available** — the system assigns seats in the requested section — spreads requests across seats instead of piling them on one.
- **Partition by event, and section if needed.** One event's seats on one database partition keeps multi-seat holds in a single transaction;
  the hottest events can be spread by section across partitions, since a multi-seat hold rarely spans sections.

**General admission** contends on one counter. `UPDATE stock SET left = left - $qty WHERE type_id = $1 AND left >= $qty` is correct, but every
buyer serialises on that row's lock, and a single row sustains perhaps a few thousand updates per second — less across a network with
replication. Options:

| Approach | How | Trade-off |
|---|---|---|
| **One row, conditional decrement** | As above | Simplest and exact; capped by one row's lock throughput |
| **Sharded stock** | Split 2,000 tickets into *N* rows of 2,000/*N*; each buyer decrements a random shard, trying others when it's empty | *N*× throughput; a request for 4 may fail on every shard near the end even though 4 remain in total |
| **In-memory atomic counter** | A Redis-style `DECRBY` (or a script that checks and decrements) in front of the database | Very fast; needs durable reconciliation with the database, and a crash can lose or double-count in-flight holds |
| **Pre-allocated tokens** | 2,000 ticket tokens in a queue; each hold pops *qty* | Natural fit for queues; same reconciliation concerns |

Sharded stock is §47.10's choice. The shards aren't independent databases, just separate rows (or keys), so the database's ordinary atomic
updates keep each shard exact, and the sum of shards is always the truth. Near the end of the sale, when most shards are empty, a request
falls back to checking every shard, and can borrow across them.

> ⚠️ **Gotcha** — "Check availability, then insert an order" as two steps oversells under load: two buyers both read `left = 1`, both insert.
> The check and the decrement must be **one** atomic operation — a conditional `UPDATE … WHERE left >= qty` that reports how many rows it changed,
> a `SELECT … FOR UPDATE` in the same transaction, or an atomic script in an in-memory store. Load testing with one client at a time never finds
> this; production finds it in the first minute.

**Holds, not sales.** A buyer who clicks "buy" hasn't paid. The system takes a **hold** — inventory reserved for this buyer for a few minutes —
and converts it to a sale only when payment succeeds. Holds expire, and a reaper returns their stock (and an expired hold can't be confirmed, so
a slow payment can't sell a seat that's been re-sold to someone else). The hold's length is a product decision with a capacity cost: long holds
are kind to slow typists and leave tickets locked up by buyers who walked away; short holds return stock fast and pressure everyone.

---

## 47.6 Deep Dive: The Waiting Room

The waiting room is a **load shedder** (Chapter 29) designed to be fair. It has three jobs: keep the crowd off the backend, decide the order
in which people get in, and admit them at a rate the backend can sustain.

**Order.** Strict first-come-first-served sounds fair, and isn't: it rewards whoever's clock, connection, or bot is fastest at 10:00:00.000.
Most large on-sales instead give everyone who arrives **before the on-sale opens** a **random** position, and queue later arrivals FIFO behind
them. Arriving 30 minutes early buys nothing over arriving 30 seconds early, so there's no reason to hammer the site before it opens.

**Admission.** Admitted buyers get a signed, short-lived token that the inventory service verifies **without a database lookup** — the queue and
the inventory service share a key, not a table:

```js
// Waiting room: everyone who arrives before the on-sale gets a random place (arriving early, or with a
// faster bot, buys nothing); later arrivals queue FIFO behind them. Admission is a signed, short-lived
// token the inventory service can verify without a database lookup.
import { createHmac, randomInt, timingSafeEqual } from 'node:crypto';

const SECRET = 'rotate-me';                                    // in production: from a secrets manager, rotated
const sign = (body) => createHmac('sha256', SECRET).update(body).digest('base64url');

function orderQueue(arrivals, openAt) {
  const early = arrivals.filter(a => a.at < openAt);
  const late = arrivals.filter(a => a.at >= openAt).sort((a, b) => a.at - b.at);
  for (let i = early.length - 1; i > 0; i--) {                 // Fisher–Yates with a CSPRNG: no predictable order
    const j = randomInt(i + 1);
    [early[i], early[j]] = [early[j], early[i]];
  }
  return [...early, ...late].map((a, i) => ({ ...a, position: i + 1 }));
}

function admissionToken(user, eventId, expiresAt) {
  const body = `${user}.${eventId}.${expiresAt}`;
  return `${body}.${sign(body)}`;
}
function verifyToken(token, eventId, now) {
  const i = token.lastIndexOf('.');
  const body = token.slice(0, i), mac = Buffer.from(token.slice(i + 1));
  const expected = Buffer.from(sign(body));
  if (mac.length !== expected.length || !timingSafeEqual(mac, expected)) return false;   // forged or altered
  const [, ev, exp] = body.split('.');
  return ev === eventId && Number(exp) > now;
}

const openAt = 1_000;
const arrivals = [
  ...Array.from({ length: 1_000 }, (_, i) => ({ user: `early-${i}`, at: i % 900 })),       // queued before open
  ...Array.from({ length: 1_000 }, (_, i) => ({ user: `late-${i}`, at: openAt + i })),
];
const q = orderQueue(arrivals, openAt);
const early = q.filter(a => a.at < openAt);
const corr = (xs, ys) => {                                      // Pearson correlation of arrival time vs position
  const mx = xs.reduce((a, b) => a + b) / xs.length, my = ys.reduce((a, b) => a + b) / ys.length;
  let sxy = 0, sxx = 0, syy = 0;
  xs.forEach((x, i) => { sxy += (x - mx) * (ys[i] - my); sxx += (x - mx) ** 2; syy += (ys[i] - my) ** 2; });
  return sxy / Math.sqrt(sxx * syy);
};
console.log('arriving earlier predicts a better place:', Math.abs(corr(early.map(a => a.at), early.map(a => a.position))) > 0.15);
console.log('first late arrival is at position', q.find(a => a.user === 'late-0').position);

const t = admissionToken('early-7', 'tour-2026-london', openAt + 600_000);
console.log(verifyToken(t, 'tour-2026-london', openAt + 1), verifyToken(t.replace('early-7', 'early-8'), 'tour-2026-london', openAt + 1), verifyToken(t, 'tour-2026-london', openAt + 600_001));
// arriving earlier predicts a better place: false
// first late arrival is at position 1001
// true false false
```

Among early arrivals, arrival time and queue position are uncorrelated; everyone who arrived after opening queues behind them in order; and the
token verifies for its owner, fails if the user is swapped, and fails after expiry. (This token format assumes IDs contain no dots; a production
token would use a structured encoding such as a JWT, Chapter 33.)

**The queue page polls cheaply.** Position updates come from a small JSON document cached at the edge per queue segment, or from a server-sent
events stream (Chapter 9) — not from the origin per user. Estimates are computed from the measured admission rate, and shown as ranges, because
a precise estimate that slips is worse than an honest range.

**Admission rate is a control loop.** Start at a planned rate, then adjust from the inventory and checkout services' latency and error rates — admit
faster while they're healthy, slower when payment latency climbs. And stop entirely, with a "sold out" message to everyone still waiting, when
available stock plus expiring holds can no longer satisfy anyone.

> 💼 **Interview** — The strongest ticketing answers separate the problem into two: a **correctness core** that's small (conditional updates on seats or
> sharded stock, holds with expiry) and a **traffic problem** that's huge (a waiting room in front, cached pages at the edge, admission tokens). Candidates
> who try to scale the inventory database to two million users have missed that only a few thousand ever need to touch it.

---

## 47.7 Deep Dive: Holds, Payment, Bots, and Fairness

**Holds and payment.** Checkout ties a hold to a payment with an idempotency key (Chapter 8): the payment is authorised, the hold is confirmed to
*sold* in a transaction that checks it hasn't expired, and the payment is captured. If the hold expired first — a very slow 3-D Secure challenge, say —
the authorisation is voided and the buyer is told, rather than charged for a seat that's gone. Some systems extend holds briefly while a payment is
in flight, which is safe because the extension is bounded and the reaper honours it.

**Per-buyer limits.** "Four tickets per customer" is enforced at hold time, keyed on the account — and, since scalpers make many accounts, also on
payment instrument, device, and address signals, checked at checkout when they're available.

**Bots** are the reason fairness is hard. Defences, in layers:

- **Make speed useless**: random order for pre-open arrivals, so the fastest script gets the same odds as everyone else.
- **Make accounts expensive**: verified accounts, phone verification, and a history of genuine purchases — some on-sales are open only to fans registered
  days in advance, then admitted by **lottery**, which removes the on-sale rush entirely.
- **Detect automation**: challenges at the waiting-room entrance (not at checkout, when the damage is done), rate limits per IP and device (Chapter 29),
  and behavioural signals.
- **Make resale less profitable**: tickets bound to the buyer's identity or delivered late to the buyer's app, and capped-price official resale.

**Fairness in the result**, not just the queue: a lottery for high-demand events lets fans register over days and gives every registrant the same odds,
which is fairer than any queue and turns a ten-minute peak into a quiet background job.

---

## 47.8 Failure Modes, Bottlenecks, and 10×

| Failure | Effect | Mitigation |
|---|---|---|
| Origin overwhelmed by refreshes | Site down at the worst moment | Waiting room in front; static pages and cached availability at the CDN; token-gated inventory API |
| Hot GA counter | Hold latency climbs, timeouts, retries make it worse | Sharded stock; one hold attempt per click with idempotency; admission rate tied to inventory latency |
| Hold reaper stalls | Stock locked up by abandoned holds; false "sold out" | Expiry checked in the hold query itself (an expired hold counts as available); reaper alerts on lag |
| Payment provider slow | Holds expire mid-payment | Bounded hold extension during in-flight payment; void authorisations for expired holds |
| Queue service fails | Nobody admitted, or everyone admitted at once | Queue state replicated; on failure, fail **closed** (pause admission) rather than open |
| Oversell from a bug | Refunds and angry customers | Database constraints as the last line (e.g., a `CHECK (left >= 0)`), invariant checks on sold + held + available, kill switch to pause sales |

**Load testing is the real defence.** On-sales fail in ways only realistic load finds: lock contention, connection-pool exhaustion, CDN cache keys that
accidentally vary per user. Rehearse the on-sale at full scale against a staging copy, and pre-warm everything that autoscales, since autoscaling reacts only after the spike has arrived.

**At 10×** — 20 million arrivals for a global tour: the waiting room and edge scale horizontally and cheaply, which is the point of the design. The
inventory path doesn't need 10× — admission is still sized to checkout and payment capacity — but a sale spanning many events at once runs each event's
inventory on its own partition, and pre-registration with lotteries becomes the better product, since a queue of 20 million for 500,000 tickets serves
almost nobody well.

---

## 47.9 Common Mistakes and Best Practices

**Mistake: read availability, then write the sale.** Two buyers read the same last ticket. Make the check and the decrement one atomic conditional
update.

**Mistake: scaling the inventory database to the crowd.** The crowd should never reach it; a waiting room and an edge cache should.

**Mistake: first-come-first-served at millisecond precision.** It rewards bots and the people closest to the data centre.

**Mistake: holds that never expire, or expire only via a reaper.** Abandoned carts lock up stock; a stalled reaper causes false sell-outs.

**Mistake: confirming a hold after it expired.** The seat may have been held by — and sold to — someone else.

**Mistake: a queue that keeps people waiting after a sell-out.** Tell everyone as soon as stock plus expiring holds can't serve them.

**Mistake: failing open when the waiting room breaks.** Two million people hit the backend at once.

**Best practice: separate the correctness core from the traffic problem.** Small, exact inventory logic behind a large, cheap admission layer.

**Best practice: shard hot counters, assign best-available seats.** Spread contention across rows instead of piling it on one.

**Best practice: rehearse at full scale.** On-sales fail on contention and configuration mistakes that only real load reveals.

---

## 47.10 Real-World Example: Sharded General-Admission Stock with Holds

This is the inventory core for general admission: stock split across shards so no single counter is a hot row, holds with expiry that return abandoned
stock, a per-buyer limit, and an invariant that sold + held + available always equals the total. Each hold is a check-and-decrement on one shard — the
in-memory equivalent of `UPDATE stock_shards SET left = left - $qty WHERE shard = $s AND left >= $qty`.

```js
// inventory.mjs — general-admission stock under extreme contention: stock split across shards so no single
// counter is a hot row, holds that reserve tickets for a few minutes during checkout, expiry that returns
// abandoned holds to stock, and a per-buyer limit. Every operation is a check-and-decrement on one shard.

export class Inventory {
  #shards; #holds = new Map(); #byBuyer = new Map(); #sold = 0; #nextHold = 1;

  constructor({ total, shards = 8, holdMs = 5 * 60_000, perBuyerLimit = 4, random = Math.random }) {
    // Spread stock evenly; in production each shard is its own row or key, updated atomically
    this.#shards = Array.from({ length: shards }, (_, i) => Math.floor(total / shards) + (i < total % shards ? 1 : 0));
    Object.assign(this, { total, holdMs, perBuyerLimit, random });
  }

  hold(buyer, qty, now) {
    const already = this.#byBuyer.get(buyer) ?? 0;
    if (already + qty > this.perBuyerLimit) return { ok: false, reason: 'limit' };
    const start = Math.floor(this.random() * this.#shards.length);    // random start spreads contention
    for (let k = 0; k < this.#shards.length; k++) {
      const s = (start + k) % this.#shards.length;
      if (this.#shards[s] >= qty) {                                   // the atomic step: UPDATE … WHERE left >= qty
        this.#shards[s] -= qty;
        const id = this.#nextHold++;
        this.#holds.set(id, { buyer, qty, shard: s, expiresAt: now + this.holdMs });
        this.#byBuyer.set(buyer, already + qty);
        return { ok: true, holdId: id, expiresAt: now + this.holdMs };
      }
    }
    // No single shard has qty left, though the sum might: a real system then tries a multi-shard hold
    return { ok: false, reason: this.available() >= qty ? 'fragmented' : 'sold-out' };
  }

  confirm(holdId, now) {                                              // called after payment succeeds
    const h = this.#holds.get(holdId);
    if (!h || h.expiresAt <= now) return false;                      // expired holds can't be bought
    this.#holds.delete(holdId);
    this.#sold += h.qty;
    return true;
  }

  expire(now) {                                                       // the reaper: return abandoned stock
    let returned = 0;
    for (const [id, h] of this.#holds) {
      if (h.expiresAt > now) continue;
      this.#holds.delete(id);
      this.#shards[h.shard] += h.qty;
      this.#byBuyer.set(h.buyer, this.#byBuyer.get(h.buyer) - h.qty);
      returned += h.qty;
    }
    return returned;
  }

  available() { return this.#shards.reduce((a, b) => a + b, 0); }
  get sold() { return this.#sold; }
  get held() { let n = 0; for (const h of this.#holds.values()) n += h.qty; return n; }
}

// ── Demo: 1,000 tickets, 5,000 buyers wanting 1–4 each, 30% abandon checkout ───────────
let seed = 42;
const rand = () => ((seed = (seed * 1103515245 + 12345) % 2 ** 31) / 2 ** 31);   // seeded, so the run is repeatable
const inv = new Inventory({ total: 1_000, shards: 8, random: rand });
const outcomes = { held: 0, 'sold-out': 0, fragmented: 0, limit: 0 };
const pending = [];
for (let b = 0; b < 5_000; b++) {
  const r = inv.hold(`buyer-${b}`, 1 + Math.floor(rand() * 4), 0);
  if (r.ok) { outcomes.held++; pending.push(r.holdId); } else outcomes[r.reason]++;
}
for (const id of pending) if (rand() >= 0.3) inv.confirm(id, 60_000);           // 70% pay within a minute
console.log('first wave:', JSON.stringify(outcomes), '| sold', inv.sold, '| held', inv.held, '| available', inv.available());
console.log('returned at expiry:', inv.expire(5 * 60_000 + 1), '| available now', inv.available());
console.log('invariant sold + held + available = total:', inv.sold + inv.held + inv.available() === inv.total);
// first wave: {"held":385,"sold-out":4614,"fragmented":1,"limit":0} | sold 667 | held 333 | available 0
// returned at expiry: 333 | available now 333
// invariant sold + held + available = total: true
```

The first 385 buyers take every ticket; 4,614 are told "sold out" immediately rather than queuing for nothing; one request is refused as *fragmented*
(enough tickets in total, but not in any one shard). When the 30% who abandoned checkout hit their five-minute expiry, 333 tickets return — and the
waiting room, which stopped admitting at sell-out, can resume for a second wave. The invariant holds throughout.

The decision to defend is **sharding the stock counter**, rather than keeping one exact row and accepting its throughput limit. One row is simpler, has no
fragmentation, and is enough for most events — and with a waiting room in front, admission could simply be throttled to what one row sustains. But the
row's limit then becomes the on-sale's limit: every buyer waits longer in the queue so that a single lock can be taken one at a time, and lock waits
under load are where timeouts, retries, and connection-pool exhaustion begin. Sharding multiplies throughput by the shard count with no loss of exactness —
each shard is still an ordinary atomic row, and the sum is still the truth — at the cost of a fragmentation edge case that affects a handful of requests
in the final seconds and is fixed by a fallback multi-shard hold. That's a small, well-understood complexity bought for the one resource every buyer
contends on.

---

## 47.11 Questions

**1.** MCQ — Why does "check availability, then insert an order" oversell? a) Slow disks b) Two buyers can both read the last ticket as available before either writes c) Caching d) Time zones
<details><summary>Answer</summary>**b)** The check and decrement must be one atomic conditional update.</details>

**2.** MCQ — What is the waiting room's main architectural purpose? a) Advertising b) Keeping the crowd away from the inventory path and admitting buyers at a rate the backend sustains c) Payment d) Analytics
<details><summary>Answer</summary>**b)** Only a few thousand buyers ever need to touch inventory at once.</details>

**3.** MCQ — Why randomise the order of people who arrive before the on-sale opens? a) Load balancing b) So arriving early or running a faster bot gives no advantage, removing the incentive to hammer the site c) Caching d) Compliance
<details><summary>Answer</summary>**b)** Speed at the millisecond stops mattering.</details>

**4.** MCQ — What does a hold do? a) Charges the card b) Reserves inventory for one buyer for a limited time during checkout, returning it if not confirmed c) Prints tickets d) Locks the whole event
<details><summary>Answer</summary>**b)** It converts to a sale only on successful payment.</details>

**5.** MCQ — Why can the inventory service verify admission tokens without a database lookup? a) It caches all users b) Tokens are signed by the queue with a shared key and carry their own expiry c) It skips verification d) Tokens are random
<details><summary>Answer</summary>**b)** Verification is a local HMAC check.</details>

**6.** MCQ — What's the trade-off of sharding a general-admission counter? a) Lost exactness b) Higher throughput, but a request may fail on every shard near the end even though enough tickets remain in total c) More storage only d) None
<details><summary>Answer</summary>**b)** Fragmentation, handled by a multi-shard fallback.</details>

**7.** Short — Why is "best available" seat assignment better than letting everyone pick from a seat map during an on-sale?
<details><summary>Answer</summary>Everyone sees the same map and clicks the same best seats, so almost all holds fail and retry. Assigning seats within a requested section spreads holds across different rows and turns contention into successful sales.</details>

**8.** Short — Why must the hold query itself treat expired holds as available, rather than relying on a reaper?
<details><summary>Answer</summary>If the reaper stalls or lags, stock would stay locked behind expired holds and the event could show a false sell-out. Checking `hold_expires_at < now()` in the hold's `WHERE` clause makes expiry effective immediately; the reaper is cleanup, not correctness.</details>

**9.** Short — What should happen when a payment completes after its hold expired?
<details><summary>Answer</summary>The confirm step fails (the hold is gone or reassigned), so the authorisation is voided or refunded and the buyer is told. A system that confirms anyway risks selling a seat twice.</details>

**10.** Short — Why should the waiting room fail closed?
<details><summary>Answer</summary>If it fails open, everyone waiting is admitted at once — exactly the load the waiting room exists to prevent — and the inventory path falls over. Pausing admission is visible but recoverable.</details>

**11.** Short — Why is a lottery often fairer than a queue for very high-demand events?
<details><summary>Answer</summary>Registration spans days, so no one needs to be online at an exact second; every registrant has equal odds regardless of connection speed or bots; and sales happen as a quiet background process rather than a ten-minute spike.</details>

**12.** Output — Using §47.2's model, with 6,000 admitted per minute, how many buyers are in checkout at once, and how fast does it sell out?
<details><summary>Answer</summary>**~30,000 in checkout; ~5 minutes.** 6,000 × 5 minutes; 50,000 / (6,000 × 0.7 × 2.5) ≈ 4.8 minutes.</details>

**13.** Output — In §47.10's `Inventory`, a buyer holds 3 tickets and then tries to hold 2 more with `perBuyerLimit: 4`. What's returned?
<details><summary>Answer</summary>**`{ ok: false, reason: 'limit' }`.** 3 + 2 exceeds 4; the limit is checked before stock.</details>

**14.** Output — 10 tickets in 4 shards with §47.10's constructor: how many tickets does each shard start with?
<details><summary>Answer</summary>**3, 3, 2, 2.** `floor(10 / 4) = 2`, and the first `10 % 4 = 2` shards get one extra.</details>

**15.** Output — If §47.10's demo had 50% of buyers abandon checkout instead of 30%, roughly how many tickets would return at expiry?
<details><summary>Answer</summary>**~500.** About half of the 1,000 held tickets; the exact figure depends on which holds (of 1–4 tickets) were abandoned.</details>

**16.** Debug — During an on-sale, hold latency climbs from 20 ms to 4 s, and timeouts trigger client retries that make it worse. Only general-admission holds are affected. Cause and fix?
<details><summary>Answer</summary>Every GA hold serialises on one stock row's lock. Shard the stock across rows, make each hold attempt idempotent and retried at most once, and tie the admission rate to inventory latency so the queue slows down before the database does.</details>

**17.** Debug — Thirty minutes after a sell-out, the event shows "sold out" but the seat table shows 2,000 seats still `held`, all with holds that expired long ago. What failed?
<details><summary>Answer</summary>Expired holds weren't being returned — the reaper stalled — and the availability logic counted any held seat as unavailable. Treat expired holds as available in the hold query, alert on reaper lag, and run the reaper to release them (then resume admission for a second wave).</details>

**18.** Debug — The CDN's hit rate for the event page is 3% during the on-sale, and the origin falls over. The page includes a "Hi, {name}" greeting. What's wrong?
<details><summary>Answer</summary>The page varies per user (cookie in the cache key or `Cache-Control: private`), so every request reaches the origin. Serve an identical static page from the edge and load personalised bits from a separate, small, cacheable-or-cheap endpoint (Chapter 6).</details>

**19.** Debug — Investigators find 2% of tickets went to 300 accounts that all joined the queue at exactly 09:59:59.950, and got the first positions. What's the design flaw?
<details><summary>Answer</summary>The queue is strict FIFO by arrival time, rewarding precise automation. Randomise positions for everyone who joins before opening, add bot challenges at queue entry, and enforce limits across linked accounts, payment instruments, and devices.</details>

**20.** Code — Write `holdSeats(seatMap, seatIds, holdId, now, holdMs)` that holds all requested seats atomically or none, treating expired holds as available.
<details><summary>Answer</summary>

```js
function holdSeats(seatMap, seatIds, holdId, now, holdMs = 300_000) {
  const free = (s) => s.state === 'available' || (s.state === 'held' && s.expiresAt <= now);
  if (!seatIds.every(id => seatMap.has(id) && free(seatMap.get(id)))) return false;   // all or nothing
  for (const id of seatIds) Object.assign(seatMap.get(id), { state: 'held', holdId, expiresAt: now + holdMs });
  return true;
}
const seats = new Map([['A1', { state: 'available' }], ['A2', { state: 'held', expiresAt: 100 }], ['A3', { state: 'sold' }]]);
console.log(holdSeats(seats, ['A1', 'A2'], 'h1', 50), holdSeats(seats, ['A1', 'A2'], 'h1', 150), holdSeats(seats, ['A3'], 'h2', 150)); // false true false
```
In the database, the same check and write are one conditional `UPDATE`, with the row count compared to the request (§47.3).</details>

**21.** Code — Write `estimateWait(position, admitPerMin)` that returns a range in minutes (±20%) for the queue page.
<details><summary>Answer</summary>

```js
function estimateWait(position, admitPerMin) {
  const mid = position / admitPerMin;
  return { low: Math.floor(mid * 0.8), high: Math.ceil(mid * 1.2) };   // a range that may hold beats a number that slips
}
console.log(JSON.stringify(estimateWait(45_000, 3_000))); // {"low":12,"high":18}
```
</details>

**22.** Code — Write `nextAdmitRate(current, { p99Ms, errorRate }, { targetMs = 500, min = 500, max = 10_000 })` that raises the rate by 10% when healthy and halves it when latency or errors are too high.
<details><summary>Answer</summary>

```js
function nextAdmitRate(current, { p99Ms, errorRate }, { targetMs = 500, min = 500, max = 10_000 } = {}) {
  const unhealthy = p99Ms > targetMs || errorRate > 0.01;
  const next = unhealthy ? current / 2 : current * 1.1;               // additive-ish increase, multiplicative decrease
  return Math.round(Math.min(max, Math.max(min, next)));
}
console.log(nextAdmitRate(3_000, { p99Ms: 200, errorRate: 0 }), nextAdmitRate(3_000, { p99Ms: 900, errorRate: 0 })); // 3300 1500
```
Backing off fast and growing slowly is the same shape as TCP congestion control, for the same reason: overload is much costlier than slight under-use.</details>

**23.** Scenario — A sneaker brand will drop 10,000 pairs in 12 sizes at noon; 3 million people will try. Design the sale.
<details><summary>Answer</summary>Prefer a raffle: registrations over a few days with verified accounts, one entry per person, winners drawn randomly and given a timed purchase window — no spike at all. If it must be a live drop: a waiting room with random pre-open order and bot challenges at entry; stock per size sharded and held during checkout with short holds; a per-account and per-card limit of one pair; and "sold out" per size pushed to everyone immediately.</details>

**24.** Scenario — Leadership asks why the on-sale needs a waiting room "if we just add more servers". How do you answer?
<details><summary>Answer</summary>More servers scale the stateless parts but not the inventory: every buyer contends on the same seats and counters, and payment providers have their own limits. Two million people can't all buy 50,000 tickets; the queue decides who tries, in a fair order, and keeps the contended core at a load where it stays fast and correct. Without it, everyone gets errors at random, including the people who would have bought — the worst possible fairness.</details>

---

## Chapter Summary

- An on-sale is a small correctness problem inside a huge traffic problem: only a few thousand buyers need to touch inventory, out of millions who arrive.
- Inventory is sold with atomic conditional updates — compare-and-set on seats, sharded counters for general admission — never check-then-write.
- Holds reserve stock during checkout and expire; expiry is enforced in the hold query itself, and expired holds can't be confirmed.
- A waiting room in front, with edge-cached pages and signed admission tokens, admits buyers at a rate tied to backend health and fails closed.
- Fairness means making speed useless — random order for early arrivals, bot challenges at entry, cross-account limits — or a lottery that removes the rush.
- Tell people the truth fast: sell-outs propagate to the queue at once, and estimates are ranges; rehearse the whole on-sale at full scale.

---

## Practice Exercises

1. Implement §47.3's seat schema in PostgreSQL and run 200 concurrent clients holding random seats; verify no seat is ever held or sold twice.
2. Benchmark a single-row general-admission counter against 8 and 32 shards under concurrent conditional decrements, and plot throughput and failures.
3. Add a multi-shard fallback to §47.10's `hold` so fragmented requests succeed when enough stock remains in total.
4. Build a waiting room page that polls an edge-cacheable position JSON, with admission tokens verified by a separate inventory service.
5. Simulate an on-sale with bots that join at precise times, comparing FIFO with randomised pre-open ordering for the share of tickets bots win.

---

## Mini Project: An On-Sale Platform

Build a ticketing system for one event with reserved and general-admission sections, and survive a simulated on-sale.

- Static event and queue pages behind a caching proxy; a queue service with randomised pre-open ordering, a controlled admission rate, and signed tokens.
- An inventory service with seat holds as conditional updates, sharded GA stock, hold expiry in the query, and a reaper.
- Checkout with an idempotency key, a simulated payment provider with variable latency, and void-on-expired-hold handling.
- Per-buyer limits, a sell-out broadcast to the queue, and a kill switch that pauses sales.
- A load test with 100,000 simulated buyers (including bots), reporting peak origin load, hold latency, oversells (must be zero), and the share of tickets
  won by bots under each ordering.

---

**Next:** [Chapter 48 — Design a Metrics and Monitoring System →](48-metrics-monitoring.md)
