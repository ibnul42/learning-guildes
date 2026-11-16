# Chapter 43 — Design a Payment System

## Learning Objectives

- Scope a payment platform that accepts card payments through providers, holds balances, refunds, and pays out
- Guarantee exactly-once financial effects across clients, services, and payment providers with idempotency end to end
- Model payments as a durable state machine that handles "unknown" outcomes safely
- Record money movements in a double-entry ledger with invariants that can't be violated
- Reconcile internal records against providers and banks, and keep sensitive card data out of scope

---

## 43.1 Requirements and Scope

A payment system moves money on behalf of a business and its customers. Throughput is modest compared with most systems in this
book; what makes it hard is that **every error is someone's money**: a duplicate charge, a lost refund, a balance that doesn't add
up. Correctness, auditability, and recoverability dominate the design.

The scenario: a marketplace where customers pay by card (through a payment service provider, or PSP), the platform takes a fee,
and sellers are paid out periodically. Customers may also hold a stored balance (wallet).

**Functional requirements**

- Accept a payment for an order: authorise, then capture (immediately or later).
- Refund all or part of a captured payment.
- Track balances: what customers hold, what sellers are owed, what the platform earned.
- Pay out sellers to their bank accounts on a schedule.
- Produce statements and reports, and reconcile with PSPs and banks.

**Out of scope**: building a PSP or connecting directly to card networks, fraud models (an input here), tax calculation, currency
exchange pricing.

**Non-functional requirements**

| Requirement | Target |
|---|---|
| Correctness | No payment captured twice; no money created or lost; every movement traceable |
| Idempotency | Every operation safe to retry from any layer |
| Durability | Committed financial records never lost (RPO ≈ 0 within a region — Chapter 32) |
| Auditability | Immutable history; who did what, when, and why |
| Availability | 99.95% for accepting payments; degraded modes (queue and confirm later) preferred to failure |
| Latency | Dominated by the PSP (hundreds of ms to seconds); our own overhead small |

---

## 43.2 Estimation

```js
// Payments: modest throughput, high stakes
const paymentsPerDay = 50e6, peakFactor = 8, postingsPerPayment = 5;
const avg = paymentsPerDay / 86_400;
console.log(`payments ~${Math.round(avg)}/s avg, ~${Math.round(avg * peakFactor).toLocaleString('en-US')}/s peak | ledger postings ~${Math.round((avg * peakFactor * postingsPerPayment) / 1000)}k/s peak`);
console.log(`ledger rows per year: ~${((paymentsPerDay * postingsPerPayment * 365) / 1e9).toFixed(0)} B`);
console.log(`0.1% of payments with an "unknown" PSP outcome = ${(paymentsPerDay * 0.001).toLocaleString('en-US')} per day to resolve`);
// payments ~579/s avg, ~4,630/s peak | ledger postings ~23k/s peak
// ledger rows per year: ~91 B
// 0.1% of payments with an "unknown" PSP outcome = 50,000 per day to resolve
```

- **Throughput is modest**: thousands of payments per second at a sales peak fits a well-designed relational system, which is good — financial
  data wants transactions and constraints (Chapter 15).
- **The ledger grows large**: tens of billions of immutable rows per year, so it needs partitioning by time or account and an archival strategy, but
  never deletion or in-place updates.
- **Rare outcomes are common at scale**: a 0.1% rate of ambiguous PSP responses is 50,000 payments a day whose fate must be determined. The design
  must handle "we don't know yet" as a normal state, not an exception.

---

## 43.3 API and Data Model

```text
POST /v1/payments              Idempotency-Key: <order-77-attempt-1>
     { orderId, amount: { value: 5000, currency: "EUR" }, paymentMethod: "pm_tok_…", capture: "automatic" }
  → 201 { paymentId, status: "succeeded" | "requires_action" | "processing" | "failed" }
POST /v1/payments/:id/refunds  Idempotency-Key       { amount }        → { refundId, status }
GET  /v1/payments/:id                                                    → current state and history
POST /webhooks/psp                                    (signed)           ← PSP events: captured, failed, disputed, refunded
```

Money is always **an integer in minor units plus a currency** (5000 EUR cents = €50.00). Floating-point amounts are a classic source of rounding
errors that reconciliation will eventually find (Chapter 1's "never" list).

| Data | Notes |
|---|---|
| **Payments** | A state machine row per payment attempt: status, amounts, PSP reference, idempotency key, version |
| **Payment events** | Append-only history of every state change and PSP interaction (request, response, webhook) |
| **Ledger: accounts** | Customer wallets, seller payables, platform revenue, PSP clearing, bank, fees, refunds |
| **Ledger: transactions and postings** | Each transaction is a set of postings that sum to zero (§43.6) |
| **Idempotency records** | Request key → stored response (Chapter 8) |
| **Payouts** | Batches of seller payments to banks, with their own state machine |

Card numbers are **not** stored: the client collects them in the PSP's hosted fields or SDK and sends us a token (`pm_tok_…`). That keeps the
platform largely out of PCI DSS scope (§43.8).

---

## 43.4 High-Level Design

```text
checkout ─▶ payment API ─▶ idempotency layer ─▶ payment service (state machine, DB)
                                                   │  ├─▶ fraud/risk check (decline, review, allow)
                                                   │  └─▶ PSP adapter(s) ─▶ PSP ─▶ card networks
                                                   │            ▲ signed webhooks
                                                   │ outbox     │
                                                   ▼            │
                                    ledger service ◀── payment events ── reconciliation jobs ◀── PSP & bank reports
                                          │
                                   payouts service ─▶ bank (scheduled batches)
```

The payment service owns the **state machine** and the conversation with PSPs; the **ledger** owns balances and the immutable record of money
movement; they communicate through events published via an outbox (Chapter 23), so a ledger posting can never be lost or duplicated relative to the
payment state change that caused it. **Reconciliation** jobs compare all of this with the PSPs' and banks' records every day.

---

## 43.5 Deep Dive: Exactly-Once Effects and the Payment State Machine

Retries happen at every layer: the browser resubmits, the checkout service retries a timeout, a worker restarts mid-request. Each one must not charge
the customer again. The chain of protections:

1. **The client's idempotency key** (Chapter 8) — generated once per payment attempt, sent with every retry. The payment API stores the key with the
   eventual response and replays it.
2. **One payment record per key**, created *before* calling the PSP, in state `processing`.
3. **The PSP's idempotency key** — derived from our payment ID, passed on every call to the PSP. Mature PSPs deduplicate on it, so if our service crashes
   after sending the request and retries, the PSP returns the original result instead of charging again. This is the link that makes the chain
   exactly-once end to end.
4. **Conditional state transitions** — `processing → succeeded` only if still `processing` — so a webhook and a synchronous response arriving together
   apply once (Chapter 15).

The payment's states:

```text
created ─▶ processing ─┬─▶ requires_action (3-D Secure etc.) ─▶ processing
                       ├─▶ succeeded ─▶ (partially_)refunded
                       ├─▶ failed
                       └─▶ unknown ──(poll PSP / webhook / reconciliation)──▶ succeeded | failed
```

The **unknown** state is the important one. A timeout or a network error after sending the request to the PSP means the charge may or may not have
happened (Chapter 19). The wrong responses are "assume it failed" (the customer retries and is charged twice) and "assume it succeeded" (the order ships
unpaid). The right response is to record `unknown`, tell the user the payment is being confirmed, and resolve it: query the PSP by idempotency key, wait
for its webhook, and as a backstop let reconciliation settle it. Only a resolved state moves money in the ledger.

> 💼 **Interview** — "How do you prevent double charges?" is the core payments question. The complete answer: client idempotency key → stored payment
> record created before calling the PSP → PSP idempotency key derived from it → conditional state transitions → an explicit unknown state resolved by
> polling, webhooks, and reconciliation — never by guessing.

**Authorise and capture.** Card payments usually happen in two steps: **authorisation** reserves funds on the card, and **capture** actually takes them
(often when the order ships). Authorisations expire after days, partial captures are possible, and an authorised-but-uncaptured payment must be captured
or released. Each is a separate idempotent operation and a separate state transition — and only capture creates ledger postings for revenue.

---

## 43.6 Deep Dive: The Double-Entry Ledger

Storing a `balance` column and updating it with each payment is the obvious design and the wrong one: an update overwrites history, a bug that adds
money to one balance without removing it from another is invisible, and "why is this seller's balance €12 higher than expected?" has no answer.

**Double-entry bookkeeping** — a technique centuries older than computers — fixes all three. Every financial event is a **transaction** made of
**postings** to accounts, and the postings of every transaction **sum to zero** (per currency): money only moves, it's never created or destroyed. A
customer's €50 card payment on a marketplace order:

| Account | Posting | Meaning |
|---|---|---|
| PSP clearing (asset) | +€50.00 | The PSP owes us €50 |
| Seller payable (liability) | −€45.00 | We owe the seller €45 |
| Platform fees (revenue) | −€5.00 | We earned €5 |
| **Sum** | **€0.00** | |

Rules that make the ledger trustworthy:

- **Immutable postings.** Nothing is ever updated or deleted. A mistake or a refund is a **new transaction** that reverses or adjusts the original, so
  history always explains the current state.
- **Balances are derived** from postings — maintained incrementally for speed, but always recomputable, and continuously checked.
- **Invariants enforced at write time**: each transaction balances; accounts that may not go negative (a customer wallet) are checked atomically with the
  posting (Chapter 15's conditional update), so concurrent spends can't overdraw.
- **Idempotent posting** keyed by the business event (`capture:pay_123`), so redelivered events (Chapter 24) don't double-post.
- **Continuous checks**: the sum of all balances per currency is exactly zero (a trial balance); each account's derived balance equals the sum of its
  postings. Either failing is a sev-1 bug.

Real ledgers also track **pending vs available** amounts (an authorisation creates a hold that reduces available balance without moving funds), store
transactions with effective dates as well as recorded dates (for corrections that belong to a previous period), and partition postings by time for
size. The real-world example in §43.10 implements the core.

---

## 43.7 Deep Dive: Reconciliation

Every party in a payment keeps its own records — your ledger, the PSP, the card networks, your bank — and they disagree more often than anyone would like:
a webhook was lost, a refund was processed twice by a manual action, a currency conversion rounded differently, a PSP fee changed, a settlement batch was
split. **Reconciliation** is the daily process of matching your records against everyone else's and explaining every difference.

```js
// Reconciliation: match our records of captured payments against the PSP's settlement report.
// Every "break" is either explained (timing) or investigated (money is wrong somewhere).
function reconcile(ours, theirs, { settlementCutoff }) {
  const byRef = new Map(theirs.map((t) => [t.ref, t]));
  const breaks = [], matched = [];
  for (const o of ours) {
    const t = byRef.get(o.ref);
    byRef.delete(o.ref);
    if (!t) {
      // Captured after the report's cutoff: expected in tomorrow's report, not a break yet
      breaks.push(o.capturedAt > settlementCutoff ? { ref: o.ref, kind: 'timing (next report)' } : { ref: o.ref, kind: 'missing at PSP' });
    } else if (t.amount !== o.amount || t.currency !== o.currency) {
      breaks.push({ ref: o.ref, kind: `amount mismatch: ours ${o.amount} ${o.currency}, theirs ${t.amount} ${t.currency}` });
    } else matched.push(o.ref);
  }
  for (const t of byRef.values()) breaks.push({ ref: t.ref, kind: 'missing in our ledger' }); // money we never recorded
  return { matched: matched.length, breaks };
}

const ours = [
  { ref: 'pay_1', amount: 5000, currency: 'EUR', capturedAt: 10 },
  { ref: 'pay_2', amount: 1299, currency: 'EUR', capturedAt: 11 },
  { ref: 'pay_3', amount: 800, currency: 'EUR', capturedAt: 12 },
  { ref: 'pay_4', amount: 2500, currency: 'EUR', capturedAt: 99 },   // after the cutoff
];
const theirs = [
  { ref: 'pay_1', amount: 5000, currency: 'EUR' },
  { ref: 'pay_2', amount: 1300, currency: 'EUR' },                    // rounding bug somewhere
  { ref: 'pay_9', amount: 4200, currency: 'EUR' },                    // a capture we have no record of
];
const r = reconcile(ours, theirs, { settlementCutoff: 50 });
console.log(`matched ${r.matched}`);
for (const b of r.breaks) console.log(`${b.ref.padEnd(6)} ${b.kind}`);
// matched 1
// pay_2  amount mismatch: ours 1299 EUR, theirs 1300 EUR
// pay_3  missing at PSP
// pay_4  timing (next report)
// pay_9  missing in our ledger
```

Each kind of break means something different: an **amount mismatch** is usually a bug or an unrecorded fee; **missing at the PSP** may be a payment we think
succeeded that didn't (a dangerous one); **missing in our ledger** is money that arrived without a record — often a resolved "unknown" payment whose outcome
never reached us; **timing** differences resolve on their own and must be tracked so they don't become permanent. Production reconciliation is
**three-way** (ledger ↔ PSP reports ↔ bank statements), runs automatically every day, auto-resolves known patterns, and puts the rest in a queue for a
finance operations team. It's also the final safety net for the state machine: an `unknown` payment that no webhook resolved is settled here.

---

## 43.8 Failure Modes, Compliance, and 10×

| Failure | Effect | Mitigation |
|---|---|---|
| PSP timeout after sending a charge | Outcome unknown | `unknown` state; PSP idempotency key; poll and webhook; reconciliation backstop |
| PSP outage | Payments fail | Secondary PSP with routing (tokens must be portable or vaulted); queue "pay later" where the business allows |
| Lost or duplicated webhooks | State not updated, or updated twice | Signed webhooks, idempotent handlers by event ID, polling for stale `processing` payments |
| Ledger posting fails after payment succeeds | Money moved without a record | Outbox from the payment state change; ledger consumer idempotent by event key |
| Double refund by support tooling | Over-refund | Refunds through the same idempotent API with remaining-refundable checks; no direct database edits |
| Chargebacks (disputes) | Funds pulled back weeks later | Dispute state machine; ledger entries for disputed amounts; evidence workflows |

**Compliance shapes the architecture.** Card data is governed by PCI DSS; the less of the system that touches raw card numbers, the less of it is in scope.
Using the PSP's hosted payment fields and tokens means card numbers never reach your servers, shrinking scope dramatically. Payouts and wallets bring
anti-money-laundering and know-your-customer obligations, which add verification states before money can move. Audit logs of every manual action and every
configuration change are a regulatory expectation, not a nice-to-have (Chapter 33).

**At 10×** — tens of thousands of payments per second: the ledger becomes the scaling challenge. Partition it by account (so most transactions touch a
small number of partitions), handle **hot accounts** — the platform's own fee or clearing account is touched by every payment — by sharding them into
sub-accounts summed for reporting (Chapter 12's sharded counter, in accounting form), and batch postings to such accounts. Multi-region adds the hardest
question: balances that must never go negative need a single home (or consensus) per account (Chapter 32), so wallets are homed to a region while card
acceptance can run active-active through PSPs.

---

## 43.9 Common Mistakes and Best Practices

**Mistake: floats for money.** Rounding errors accumulate and reconciliation can never fully match; use integer minor units with a currency.

**Mistake: retrying PSP calls without the PSP's idempotency key.** A crash after sending the charge becomes a double charge on retry.

**Mistake: treating timeouts as failures.** The charge may have succeeded; record `unknown` and resolve it.

**Mistake: mutable balance columns as the source of truth.** History is lost and bugs that create or destroy money go unnoticed; use a double-entry ledger.

**Mistake: editing or deleting ledger rows to fix mistakes.** It destroys auditability; post reversing or adjusting transactions.

**Mistake: no daily reconciliation.** Breaks accumulate silently until an audit or a customer finds them.

**Mistake: storing card numbers.** It drags the whole system into PCI scope; use PSP tokens and hosted fields.

**Best practice: idempotency at every hop, keyed by the business operation.** Client → API → payment record → PSP → ledger posting.

**Best practice: an explicit state machine with conditional transitions.** Including the unknown state and how it resolves.

**Best practice: invariants checked continuously.** Every transaction balances, the trial balance is zero, and no protected account goes negative.

---

## 43.10 Real-World Example: A Double-Entry Ledger Core

This is the core of the ledger: transactions made of postings that must balance per currency, BigInt minor units only, idempotent posting by key,
atomic enforcement of non-negative accounts (all postings apply or none do), immutable history with corrections as reversing transactions, and a trial
balance check. The demo records a marketplace capture, a retry, two rejected bad postings, and a full refund.

```js
// ledger.mjs — a double-entry ledger: every transaction is a set of postings that sum to zero per
// currency; postings are immutable (corrections are new, reversing transactions); posting is
// idempotent by key; and accounts that must never go negative are checked atomically.

export class Ledger {
  #accounts = new Map();      // id → { currency, allowNegative, balance (minor units, BigInt) }
  #transactions = new Map();  // idempotency key → transaction
  #postings = [];             // append-only journal

  open(id, { currency, allowNegative = false }) { this.#accounts.set(id, { currency, allowNegative, balance: 0n }); }

  // postings: [{ account, amount }] — positive = debit, negative = credit (any consistent sign convention works)
  post(key, { description, postings }) {
    if (this.#transactions.has(key)) return { ...this.#transactions.get(key), replayed: true }; // idempotent
    const byCurrency = new Map();
    for (const p of postings) {
      const acct = this.#accounts.get(p.account);
      if (!acct) throw new Error(`unknown account ${p.account}`);
      if (typeof p.amount !== 'bigint') throw new TypeError('amounts are BigInt minor units — never floats');
      byCurrency.set(acct.currency, (byCurrency.get(acct.currency) ?? 0n) + p.amount);
    }
    for (const [cur, sum] of byCurrency) if (sum !== 0n) throw new Error(`unbalanced in ${cur}: ${sum}`);
    // Check every balance invariant BEFORE applying anything: all postings apply, or none do
    for (const p of postings) {
      const acct = this.#accounts.get(p.account);
      if (!acct.allowNegative && acct.balance + p.amount < 0n) throw new Error(`insufficient funds in ${p.account}`);
    }
    for (const p of postings) this.#accounts.get(p.account).balance += p.amount;
    const tx = { key, description, postings, at: this.#postings.length };
    this.#postings.push(...postings.map((p) => ({ ...p, key })));
    this.#transactions.set(key, tx);
    return { ...tx, replayed: false };
  }

  // Mistakes are corrected by a new transaction that reverses the original — history is never edited
  reverse(originalKey, key) {
    const tx = this.#transactions.get(originalKey);
    return this.post(key, { description: `reversal of ${originalKey}`, postings: tx.postings.map((p) => ({ account: p.account, amount: -p.amount })) });
  }

  balance(id) { return this.#accounts.get(id).balance; }
  // Whole-ledger invariant, run continuously in production: every currency sums to exactly zero
  trialBalance() {
    const sums = {};
    for (const [, a] of this.#accounts) sums[a.currency] = (sums[a.currency] ?? 0n) + a.balance;
    return sums;
  }
}

// ── Demo: a marketplace order — customer pays €50, platform keeps a €5 fee, seller is owed €45 ──
const L = new Ledger();
L.open('psp:clearing', { currency: 'EUR', allowNegative: true });   // money the PSP owes us (asset)
L.open('customer:ana:wallet', { currency: 'EUR' });                 // a stored balance that can't go negative
L.open('seller:s9:payable', { currency: 'EUR', allowNegative: true });
L.open('platform:fees', { currency: 'EUR', allowNegative: true });

const charge = { description: 'order o-77 captured', postings: [
  { account: 'psp:clearing', amount: 5000n },          // the PSP will settle €50 to us
  { account: 'seller:s9:payable', amount: -4500n },    // we owe the seller €45
  { account: 'platform:fees', amount: -500n },         // we earned €5
] };
L.post('capture:o-77', charge);
console.log('retry replayed:', L.post('capture:o-77', charge).replayed, '| seller owed:', -L.balance('seller:s9:payable'));
// retry replayed: true | seller owed: 4500n
try { L.post('bad', { description: 'typo', postings: [{ account: 'psp:clearing', amount: 100n }, { account: 'platform:fees', amount: -99n }] }); }
catch (e) { console.log('rejected:', e.message); }                                     // rejected: unbalanced in EUR: 1
try { L.post('w1', { description: 'wallet spend', postings: [{ account: 'customer:ana:wallet', amount: -1000n }, { account: 'platform:fees', amount: 1000n }] }); }
catch (e) { console.log('rejected:', e.message); }                                     // rejected: insufficient funds in customer:ana:wallet
L.reverse('capture:o-77', 'refund:o-77');                   // full refund: reverse every leg
console.log('after refund — seller owed:', -L.balance('seller:s9:payable'), '| trial balance:', L.trialBalance());
// after refund — seller owed: 0n | trial balance: { EUR: 0n }
```

In a database the same rules become: a `transactions` table with a unique idempotency key, a `postings` table that's insert-only (no `UPDATE` or `DELETE`
permission for the application role), balances maintained in the same database transaction as the postings with a `CHECK (balance >= 0)` or conditional
update for protected accounts, and a deferred constraint or trigger verifying each transaction sums to zero.

The decision to defend is **modelling every money movement as balanced postings across several accounts**, instead of recording a payment as one row with an
amount and updating a couple of balances. The single-row approach is simpler to write and to query, and for a while it works. What it can't do is *prove*
anything: if a bug credits a seller without debiting anything, every individual row still looks plausible, and the error surfaces months later as an
unexplained difference in a bank reconciliation. With double entry, money can only move between accounts — a transaction that would create or destroy it is
rejected at write time — and the trial balance must be exactly zero at every moment, so a whole class of bugs becomes impossible rather than merely unlikely,
and every balance can be explained posting by posting. That is the property auditors, regulators, and finance teams actually require, and it costs a few more
rows per payment.

---

## 43.11 Questions

**1.** MCQ — What makes a retried PSP charge safe after our service crashed mid-request? a) Our own retry logic b) Passing the PSP an idempotency key derived from our payment ID c) Waiting longer d) Refunding afterwards
<details><summary>Answer</summary>**b)** The PSP deduplicates on it and returns the original result instead of charging again.</details>

**2.** MCQ — A PSP call times out. What state should the payment move to? a) failed b) succeeded c) unknown, to be resolved by polling, webhooks, or reconciliation d) created
<details><summary>Answer</summary>**c)** The charge may or may not have happened; guessing either way causes double charges or unpaid orders.</details>

**3.** MCQ — In double-entry bookkeeping, what must hold for every transaction? a) Exactly two postings b) Postings sum to zero per currency c) Only debits d) A positive balance on every account
<details><summary>Answer</summary>**b)** Money moves between accounts; it's never created or destroyed.</details>

**4.** MCQ — How should a mistaken ledger entry be corrected? a) Edit the row b) Delete the row c) Post a new reversing or adjusting transaction d) Change the balance directly
<details><summary>Answer</summary>**c)** History stays immutable and explains the current state.</details>

**5.** MCQ — How should money amounts be represented? a) Floating-point numbers b) Integers in minor units plus a currency code c) Strings d) Decimal floats without currency
<details><summary>Answer</summary>**b)** Exact arithmetic, with the currency always explicit.</details>

**6.** MCQ — What keeps most of the platform out of PCI DSS scope? a) Encrypting card numbers in our database b) Collecting cards via the PSP's hosted fields and storing only tokens c) Using HTTPS d) Deleting cards after use
<details><summary>Answer</summary>**b)** Raw card numbers never touch our servers.</details>

**7.** Short — List the idempotency protections from the client to the PSP.
<details><summary>Answer</summary>A client idempotency key per payment attempt; the API storing and replaying responses for that key; a payment record created before calling the PSP; a PSP idempotency key derived from the payment ID; and conditional state transitions so synchronous responses and webhooks apply once.</details>

**8.** Short — Why separate authorisation from capture?
<details><summary>Answer</summary>Authorisation reserves funds without taking them, so the business can capture when it fulfils (or capture less, or release) — e.g. capture on shipment. They're separate idempotent operations and state transitions, and only capture moves money in the ledger.</details>

**9.** Short — What does a trial balance check, and why run it continuously?
<details><summary>Answer</summary>That the sum of all account balances per currency is exactly zero. Any non-zero result means money was created or destroyed by a bug, which should be caught immediately rather than at month-end.</details>

**10.** Short — What are the main kinds of reconciliation breaks?
<details><summary>Answer</summary>Amount mismatches (bugs, fees, rounding), records missing at the PSP (payments we think succeeded), records missing in our ledger (money we never recorded, often a resolved unknown), and timing differences that resolve in a later report.</details>

**11.** Short — Why is the platform's fee account a scaling problem at 10×, and what's the fix?
<details><summary>Answer</summary>Every payment posts to it, making it a hot row whose lock serialises the whole system. Shard it into several sub-accounts (posting to one chosen by hash or at random) and sum them for reporting, or batch postings to it.</details>

**12.** Output — Using §43.10's ledger, after the capture of order o-77 (before the refund), what is `L.balance('platform:fees')` and what does the trial balance show?
<details><summary>Answer</summary>**`-500n`, and `{ EUR: 0n }`.** Revenue sits as a credit of €5; the three postings sum to zero.</details>

**13.** Output — In §43.7's reconciliation, if `pay_4`'s `capturedAt` were 40 instead of 99, how would it be classified?
<details><summary>Answer</summary>**`missing at PSP`.** It was captured before the report's cutoff (50) but isn't in the report — a real break, not timing.</details>

**14.** Output — 50 million payments per day, 0.3% of PSP calls time out. How many payments per day enter the unknown state, and how many per second on average?
<details><summary>Answer</summary>**150,000 per day, ~1.7 per second.** 50 M × 0.003; ÷ 86,400 ≈ 1.74.</details>

**15.** Output — What does this print, and why is it a problem?
```js
console.log(0.1 + 0.2 === 0.3, 10 + 20 === 30);
```
<details><summary>Answer</summary>`false true` — binary floating point can't represent 0.1 or 0.2 exactly, so money in floats accumulates rounding errors; integer minor units don't.</details>

**16.** Debug — Some customers were charged twice during a PSP latency incident. The checkout service retried timed-out charge requests, and the PSP adapter generated a new idempotency key on each attempt. What's the fix?
<details><summary>Answer</summary>Derive the PSP idempotency key from the payment record (created once, before the first attempt), so every retry of that payment carries the same key and the PSP returns the original outcome. Also record timeouts as `unknown` and resolve them rather than retrying blindly.</details>

**17.** Debug — Month-end reconciliation shows €18,400 more in the bank than the ledger accounts for. The ledger has never had a failed posting. Where do you look?
<details><summary>Answer</summary>Money that arrived without a corresponding ledger entry: payments stuck in `unknown` that the PSP actually captured, webhooks lost before the outbox, manual operations done in the PSP dashboard, or fees and FX differences not modelled. Run daily three-way reconciliation, resolve unknowns, and post adjusting entries with an explanation for each.</details>

**18.** Debug — Two concurrent purchases from the same wallet both succeeded, leaving it at −€12. Each purchase read the balance, checked it, then posted. Cause and fix?
<details><summary>Answer</summary>A check-then-act race (Chapter 15). Enforce the invariant atomically with the posting — a conditional update (`balance = balance - $amount WHERE balance >= $amount`) or a `CHECK (balance >= 0)` constraint within the same transaction — and retry the loser.</details>

**19.** Debug — A support engineer fixed a refund mistake by running `UPDATE ledger_postings SET amount = …` in production. Balances now disagree with postings, and auditors want an explanation. What should the process be?
<details><summary>Answer</summary>Ledger rows must be immutable: revoke update and delete permissions for application and support roles, and correct mistakes only by posting reversing or adjusting transactions through the same API, with an audit-logged reason and approver. Recompute balances from postings to find and document the damage.</details>

**20.** Code — Write `formatMoney(minor, currency, locale)` that formats integer minor units correctly for currencies with different decimal places (EUR 2, JPY 0, BHD 3).
<details><summary>Answer</summary>

```js
function formatMoney(minor, currency, locale = 'en-GB') {
  const digits = new Intl.NumberFormat(locale, { style: 'currency', currency }).resolvedOptions().maximumFractionDigits;
  return new Intl.NumberFormat(locale, { style: 'currency', currency }).format(Number(minor) / 10 ** digits);
}
console.log(formatMoney(5000n, 'EUR'), formatMoney(5000n, 'JPY'), formatMoney(5000n, 'BHD')); // €50.00 JP¥5,000 BHD 5.000
```
Formatting is the only place minor units become decimals — arithmetic stays in integers.</details>

**21.** Code — Write `splitAmount(totalMinor, shares)` that splits an amount into parts proportional to `shares`, distributing remainder cents so the parts always sum exactly to the total.
<details><summary>Answer</summary>

```js
function splitAmount(total, shares) {
  const sum = shares.reduce((a, b) => a + b, 0n);
  const parts = shares.map((s) => (total * s) / sum);               // BigInt division rounds down
  let remainder = total - parts.reduce((a, b) => a + b, 0n);
  for (let i = 0; remainder > 0n; i = (i + 1) % parts.length) { parts[i] += 1n; remainder -= 1n; } // hand out leftover cents
  return parts;
}
console.log(splitAmount(1000n, [1n, 1n, 1n])); // [ 334n, 333n, 333n ]
```
Rounding each share independently would create or lose a cent; distributing the remainder keeps the transaction balanced.</details>

**22.** Code — Write `transitionPayment(payment, event)` for the state machine, accepting PSP events idempotently: repeated events for an already-applied state return the payment unchanged.
<details><summary>Answer</summary>

```js
const NEXT = { processing: ['succeeded', 'failed', 'unknown', 'requires_action'], unknown: ['succeeded', 'failed'],
  requires_action: ['processing', 'failed'], succeeded: ['refunded', 'partially_refunded'] };
function transitionPayment(payment, { type, eventId }) {
  if (payment.appliedEvents.includes(eventId)) return { payment, applied: false };          // duplicate webhook
  if (payment.status === type) return { payment: { ...payment, appliedEvents: [...payment.appliedEvents, eventId] }, applied: false };
  if (!(NEXT[payment.status] ?? []).includes(type)) throw new Error(`${payment.status} → ${type} not allowed`);
  return { payment: { ...payment, status: type, appliedEvents: [...payment.appliedEvents, eventId] }, applied: true };
}
const p = { status: 'unknown', appliedEvents: [] };
const r1 = transitionPayment(p, { type: 'succeeded', eventId: 'evt_1' });
console.log(r1.payment.status, transitionPayment(r1.payment, { type: 'succeeded', eventId: 'evt_1' }).applied); // succeeded false
```
In storage, the transition is one conditional update on status and version, plus an insert into a processed-events table.</details>

**23.** Scenario — The business wants to add a second PSP for resilience and lower fees. What has to change?
<details><summary>Answer</summary>Card tokens are usually PSP-specific, so either vault cards with a provider-neutral token service (taking on more PCI scope) or tokenise with both. Add a routing layer choosing a PSP per payment (cost, health, card type), keeping each payment's PSP fixed for its lifetime (captures and refunds must go to the same PSP). Reconciliation, webhooks, fee accounts, and clearing accounts become per PSP; health-based failover needs care so an `unknown` payment isn't retried on the other PSP.</details>

**24.** Scenario — A product manager wants instant seller payouts ("withdraw your earnings now") instead of weekly batches. What do you need to consider?
<details><summary>Answer</summary>Available vs pending balances become critical: only funds past refund and chargeback risk windows (or covered by a reserve) are withdrawable. Each payout is an idempotent, ledger-backed transaction that moves funds from seller payable to a payout clearing account before calling the bank or payout provider, with the same unknown-outcome handling as charges. Add KYC and anti-fraud checks (account takeover makes instant payouts attractive to attackers), per-seller limits, and reconciliation against payout provider reports. Fees for instant payouts are posted as their own ledger lines.</details>

---

## Chapter Summary

- Payments are low-throughput, high-stakes: correctness, auditability, and recoverability matter more than scale, and rare failure modes happen thousands of
  times a day.
- Exactly-once effects need idempotency at every hop — client key, stored payment record, PSP idempotency key derived from it, conditional state transitions,
  idempotent ledger postings.
- Model payments as an explicit state machine with an `unknown` state for ambiguous outcomes, resolved by polling, webhooks, and reconciliation — never by
  guessing.
- Record money in a double-entry ledger: balanced, immutable postings in integer minor units; corrections as new transactions; protected balances enforced
  atomically; trial balance checked continuously.
- Reconcile daily, three ways, against PSPs and banks; classify and resolve every break.
- Keep card data out of your systems with PSP tokens, audit every manual action, and at 10× partition the ledger and shard hot system accounts.

---

## Practice Exercises

1. Implement §43.10's ledger on PostgreSQL with insert-only postings, a unique idempotency key, a balancing check, and atomic non-negative balances; test with
   100 concurrent spends from one wallet.
2. Build the payment state machine with a fake PSP that randomly times out, declines, or succeeds slowly; prove no payment is ever charged twice and every
   unknown resolves.
3. Read your PSP's documentation on idempotency keys, webhooks (signing, retries), and settlement reports, and design the reconciliation job for its report
   format.
4. Model a marketplace refund after a seller has already been paid out: which accounts move, and how is the seller's negative balance recovered?
5. Write a property-based test that generates random sequences of captures, refunds, and payouts and checks the trial balance and non-negative invariants after
   every step.

---

## Mini Project: A Marketplace Payments Core

Build the core of a marketplace payment platform with a simulated PSP, proving correctness under failures.

- A payment API with idempotency keys, a state machine including `unknown`, and a PSP adapter using PSP idempotency keys.
- A simulated PSP with authorisation, capture, refunds, webhooks (sometimes duplicated or lost), timeouts, and a daily settlement report.
- A double-entry ledger fed by payment events through an outbox, with holds for authorisations and available/pending balances for sellers.
- Daily reconciliation against the simulated settlement report, with break classification and a resolution queue.
- A chaos test running thousands of payments with random crashes, timeouts, and duplicate webhooks, asserting: no double charges, trial balance always zero,
  every unknown resolved, reconciliation fully matched.

---

**Next:** [Chapter 44 — Design a Web Crawler →](44-web-crawler.md)
