# Chapter 27 — Service Boundaries and Architecture Styles

## Learning Objectives

- Compare monoliths, modular monoliths, and microservices by what each costs and what each buys
- Find service boundaries from domain cohesion, data ownership, and observed change coupling
- Choose synchronous or asynchronous communication between services, and decide where gateways, BFFs, and meshes fit
- Migrate a monolith incrementally with the strangler pattern, including its data
- Recognise a distributed monolith and the measurements that reveal one

---

## 27.1 What a Boundary Is

A **service boundary** is the line across which code is deployed, owned, and scaled independently, and across
which data is accessed only through an interface. Drawing one is a trade: everything inside a boundary can
change together cheaply — one commit, one deploy, one transaction — and everything that crosses it becomes an
API, a network call, and a coordination problem between teams.

Three properties define a boundary, and a "service" that lacks any of them isn't one:

| Property | Meaning | The test |
|---|---|---|
| **Independent deployability** | It can be released without coordinating with other services | Has it shipped alone, on its own schedule, in the last month? |
| **Data ownership** | It is the only writer (and the source of truth) for its data | Does any other service read or write its tables directly? |
| **Team ownership** | One team builds it, runs it, and is paged for it | Who answers when it breaks at 3 a.m.? |

**Conway's law** observes that systems end up mirroring the communication structure of the organisations that
build them. It cuts both ways: services split along lines where teams don't talk will develop awkward interfaces
there, and a team structure can be shaped deliberately to produce the architecture you want (the "inverse Conway
manoeuvre"). The practical upshot is that service boundaries are as much an organisational decision as a technical
one — and splitting a system into more services than there are teams to own them produces orphaned services that
nobody understands.

---

## 27.2 Monolith, Modular Monolith, Microservices

| | Monolith | Modular monolith | Microservices |
|---|---|---|---|
| Deploy unit | One | One | Many |
| Internal boundaries | Informal — anything can call anything | Enforced modules with public APIs | Network APIs |
| Data | One database, shared freely | One database, tables owned per module | Database per service |
| Cross-boundary call cost | Function call | Function call | Network round trip, serialisation, partial failure |
| Transactions across boundaries | Local | Local | Sagas, outboxes (Chapter 23) |
| Independent scaling | No | No (scale the whole) | Yes, per service |
| Independent deployment and failure | No | No | Yes |
| Operational cost | Low | Low | High: per-service pipelines, observability, on-call, discovery |
| Typical fit | Small teams, early products | Most products, up to many teams | Many teams with divergent release, scaling, or reliability needs |

Moving right buys **independence** — teams deploy without coordinating, services scale separately, one service's
failure needn't take down another — and pays for it with everything Part IV described: network calls that fail
partially, data that's only eventually consistent across services, and transactions that become sagas. Martin
Fowler's phrase for the cost is the **microservice premium**: a fixed overhead that only pays for itself once the
coordination costs of a single deployable outweigh it.

The **modular monolith** is underused and usually the best starting point. It keeps one deployable and one database,
but divides the code into modules with explicit public interfaces, owned tables, and *enforced* dependency rules —
so the boundaries are real and testable, and a module can later be extracted into a service along a seam that
already exists. The enforcement is what distinguishes it from a monolith with folders:

```js
// Enforce module boundaries inside a modular monolith: each module may import only its own
// files, the public entry points of modules it declares as dependencies, and shared kernel code.
const modules = {
  orders:    { deps: ['catalog', 'payments'] },
  catalog:   { deps: [] },
  payments:  { deps: [] },
  shipping:  { deps: ['orders'] },
};
const moduleOf = (path) => path.split('/')[1];                 // src/<module>/...
const isPublic = (path) => /^src\/\w+\/index\.js$/.test(path);  // a module's API is its index.js

function checkImports(imports) {
  const violations = [];
  for (const { from, to } of imports) {
    const a = moduleOf(from), b = moduleOf(to);
    if (a === b || b === 'shared') continue;
    if (!modules[a].deps.includes(b)) violations.push(`${from} → ${to}: ${a} may not depend on ${b}`);
    else if (!isPublic(to)) violations.push(`${from} → ${to}: reaches into the internals of ${b}; use src/${b}/index.js`);
  }
  return violations;
}

console.log(checkImports([
  { from: 'src/orders/place-order.js', to: 'src/catalog/index.js' },          // allowed
  { from: 'src/orders/place-order.js', to: 'src/payments/stripe-client.js' }, // internals
  { from: 'src/catalog/pricing.js', to: 'src/orders/index.js' },              // undeclared, and a cycle
  { from: 'src/shipping/label.js', to: 'src/shared/money.js' },               // shared kernel: fine
]));
// [
//   'src/orders/place-order.js → src/payments/stripe-client.js: reaches into the internals of payments; use src/payments/index.js',
//   'src/catalog/pricing.js → src/orders/index.js: catalog may not depend on orders'
// ]
```

Run a check like this in CI (tools such as dependency-cruiser, ArchUnit, or language-level module systems do it
properly) and module boundaries stop eroding one convenient import at a time. Apply the same rule to the database:
each module's tables are accessed only by that module's code.

> 💼 **Interview** — "Would you build this as microservices?" is often a test of judgement. A strong answer for a new
> product: "A modular monolith, with module boundaries along the domains I'd eventually split on, enforced in CI, each
> module owning its tables. I'd extract a service when a module needs independent scaling, a different reliability
> target, or a separate team's release cadence — not before."

---

## 27.3 Finding Boundaries

Good boundaries put things that change together on the same side and things that change independently on different
sides. Several lenses help find them.

**Business capabilities and bounded contexts.** Domain-driven design's **bounded context** is a part of the domain
within which terms have one consistent meaning. "Product" means something different to the catalogue (descriptions,
images), to pricing (cost, margin, rules), to inventory (stock by warehouse), and to shipping (weight, dimensions).
Each meaning is a candidate boundary; forcing them into one shared `Product` model couples teams that have little to
say to each other. Boundaries drawn around **capabilities** — catalogue, pricing, checkout, fulfilment, billing — age
better than boundaries drawn around **entities** (a "product service", a "user service"), because entity services
become CRUD wrappers that every workflow must call, and every feature change touches several of them.

**Data ownership and transactions.** Data that must change atomically together belongs on the same side of a
boundary (Chapter 23's closing point). If two candidate services would need a distributed transaction on their hot
path, they're one service.

**Change coupling from history.** What the domain model suggests can be checked against what actually happens.
Version-control history records which parts of the code are changed *together*; modules that are nearly always edited
in the same commits are coupled in practice, whatever the diagrams say:

```js
// change-coupling.mjs — mine version-control history for modules that change together.
// Static imports show what code CAN reach; co-change shows what people actually have to edit
// together, which is the coupling a service boundary would turn into cross-team coordination.

export function changeCoupling(commits, moduleOf, { minCommits = 3 } = {}) {
  const changes = new Map();            // module → commits touching it
  const together = new Map();           // "a|b" → commits touching both
  for (const files of commits) {
    const mods = [...new Set(files.map(moduleOf).filter(Boolean))].sort();
    for (const m of mods) changes.set(m, (changes.get(m) ?? 0) + 1);
    for (let i = 0; i < mods.length; i++) {
      for (let j = i + 1; j < mods.length; j++) {
        const key = `${mods[i]}|${mods[j]}`;
        together.set(key, (together.get(key) ?? 0) + 1);
      }
    }
  }
  return [...together]
    .filter(([, n]) => n >= minCommits)
    .map(([key, n]) => {
      const [a, b] = key.split('|');
      // Degree of coupling: of the commits touching the less-frequently-changed module,
      // how many also touched the other?
      return { pair: `${a} ↔ ${b}`, together: n, degree: n / Math.min(changes.get(a), changes.get(b)) };
    })
    .sort((x, y) => y.degree - x.degree)
    .map((r) => ({ ...r, degree: `${Math.round(r.degree * 100)}%` }));
}

// Sample history: each entry is the list of files one commit touched
const commits = [
  ['src/orders/place.js', 'src/pricing/discounts.js'], ['src/orders/place.js', 'src/pricing/rules.js'],
  ['src/pricing/rules.js', 'src/orders/totals.js'], ['src/pricing/tax.js'],
  ['src/orders/totals.js', 'src/pricing/tax.js', 'src/invoices/pdf.js'],
  ['src/shipping/carrier.js'], ['src/shipping/label.js', 'src/orders/status.js'],
  ['src/shipping/rates.js'], ['src/orders/status.js', 'src/pricing/rules.js'],
  ['src/invoices/pdf.js', 'src/orders/totals.js'], ['src/invoices/email.js', 'src/orders/totals.js'],
  ['src/invoices/pdf.js', 'src/orders/totals.js'], ['src/shipping/carrier.js'],
];
for (const r of changeCoupling(commits, (f) => f.split('/')[1], { minCommits: 2 })) console.log(r.pair, r.together, r.degree);
// invoices ↔ orders 4 100%
// orders ↔ pricing 5 83%
```

In this history, every change to invoicing also touched orders, and most pricing changes touched orders — so splitting
pricing or invoicing into separate services from orders would turn most of those commits into coordinated releases
across teams. Shipping, by contrast, changed mostly on its own: a much better candidate for extraction. On a real
repository, feed `git log --name-only` output into the same function.

**The independent-deployment test.** For any proposed boundary, ask: of the last twenty changes on either side, how
many would have required a coordinated release? If the answer is "most of them", the boundary is wrong.

---

## 27.4 Communication Between Services

Once a boundary is a network, every interaction pays Chapter 5's and Chapter 19's costs. Choose the style per
interaction:

| Style | Use when | Costs |
|---|---|---|
| **Synchronous request/response** (REST, gRPC) | The caller needs the answer now to proceed (check stock at checkout, authorise a payment) | Temporal coupling: the caller fails or slows when the callee does; availability multiplies (Chapter 4) |
| **Asynchronous events** (Chapter 26) | Others need to know something happened; the publisher doesn't need them | Eventual consistency; flows harder to trace |
| **Asynchronous commands** (queue + reply) | Work the caller needs done, but not immediately | Correlating replies, timeouts |

Three rules keep synchronous coupling under control:

- **Keep request paths shallow.** Each synchronous hop multiplies latency and failure probability. A user request that
  fans out to twelve services, each calling three more, inherits the tail latency of dozens of calls (Chapter 3) and
  the series availability of all of them (Chapter 4). If a request path is deep, move data closer (event-carried state,
  Chapter 26) or merge services.
- **Make dependencies optional where the product allows**, with timeouts, fallbacks, and circuit breakers (Chapter 28).
- **Avoid chatty interfaces.** An interface that needs ten calls to render one page is a sign that a boundary cuts
  through a single capability, or that the interface needs a batch or aggregate operation (Chapter 5).

### Gateways, BFFs, and meshes

| Component | What it does | Where it helps | Watch out for |
|---|---|---|---|
| **API gateway** | One entry point for clients: routing, TLS, authentication, rate limits, request shaping | Hiding internal topology from clients; cross-cutting edge concerns | Business logic creeping in; becoming a shared bottleneck owned by no team |
| **Backend for frontend (BFF)** | A thin service per client type (web, iOS, partner API) that aggregates internal calls into what that client needs | Different clients needing different shapes; reducing client round trips | Duplicated logic across BFFs; each needs an owner (usually the client team) |
| **Service mesh** (sidecar or proxyless) | mTLS between services, retries, timeouts, load balancing, traffic shifting, telemetry — outside application code | Uniform security and resilience policy across many services and languages | Operational complexity; retries configured in the mesh *and* the app multiply (Chapter 28) |
| **Service discovery** | Maps service names to healthy instances (DNS, a registry, Kubernetes services) | Every multi-instance deployment | Stale entries; caching (Chapter 6) |

---

## 27.5 Migrating From a Monolith

Rewriting a working monolith from scratch as microservices is one of the most reliable ways to lose years. The proven
approach is incremental: the **strangler fig** pattern, named after a vine that grows around a tree until it can stand
on its own.

```text
phase 1:  clients ──▶ facade/proxy ──▶ monolith (everything)
phase 2:  clients ──▶ facade/proxy ──┬─▶ new shipping service   (routes for /shipping/*)
                                     └─▶ monolith (everything else)
phase 3:  clients ──▶ facade/proxy ──┬─▶ shipping  ┬─▶ billing  … monolith shrinks until it can be retired
```

The steps for each extraction:

1. **Find a seam** — a module with clear inputs and outputs and low change coupling (§27.3). If the monolith has no
   seams, create them first by modularising it in place; extraction along a real module boundary is mechanical, and
   extraction through tangled code is a rewrite.
2. **Put a facade in front** — a proxy, gateway, or in-code interface — so callers can be redirected without knowing.
3. **Build the new service behind the interface** and route a small share of traffic to it (by tenant, by percentage),
   comparing results with the monolith's where possible ("shadow" or "parallel run").
4. **Move the data.** This is the hard part. The new service needs to own its tables. Typical sequence: the new service
   reads from its own store, kept in sync from the monolith's tables by CDC; then writes move to the new service, which
   publishes events the monolith consumes for anything that still needs the data; then the monolith's copy is removed.
   During the transition, exactly one side is the writer for each piece of data at any time.
5. **Remove the old path** from the monolith. An extraction that leaves the old code in place "just in case" leaves two
   implementations to keep consistent.

> ⚠️ **Gotcha** — **The shared database is where migrations stall.** Extracting the code but leaving the new service
> reading and writing the monolith's tables produces a service that can't change its own schema, can't be deployed
> independently of the monolith's migrations, and can corrupt data the monolith assumes it owns. A shared database is
> acceptable as a short, explicitly temporary step; as an end state, it's a distributed monolith with extra latency.

---

## 27.6 The Distributed Monolith

A **distributed monolith** has the costs of microservices and the coupling of a monolith: services that must be
deployed together, share databases, call each other synchronously in long chains, and can't change without
coordinating. It's the most common outcome of splitting a system by technical layers or entities instead of by
capability, and it's worse than the monolith it replaced.

Symptoms worth measuring:

| Symptom | Measure | Healthy direction |
|---|---|---|
| Coordinated releases | Share of deploys that required another service to deploy first or together | Near zero |
| Deep synchronous chains | Max and p95 depth of service calls per user request (from tracing — Chapter 30) | Shallow: 2–3 hops |
| Shared data | Services reading or writing tables they don't own | None |
| Change coupling across services | Share of features requiring changes in more than one service | Low |
| Cascading failures | Incidents where one service's failure caused others' | Rare; contained by timeouts and breakers |

**Contract testing** keeps independently deployable services actually independent: each consumer publishes the
requests and response shapes it relies on, and the provider's CI verifies it still satisfies every consumer's
contract before it deploys. It catches breaking API changes before they reach production, without the brittle,
slow end-to-end test environments that distributed monoliths come to depend on.

Teams that recognise a distributed monolith have two honest options: **merge** the services that change together back
into one (publicly documented examples of companies consolidating microservices exist, and they're not failures —
they're corrections), or **invest in real independence** — separate data, events instead of synchronous chains, and
contracts.

---

## 27.7 Common Mistakes and Best Practices

**Mistake: microservices for a small team.** The fixed cost of distribution — pipelines, observability, on-call,
network failures, sagas — exceeds the coordination it saves.

**Mistake: entity services.** A "user service" and "product service" that every workflow must call become chatty CRUD
wrappers coupled to everything.

**Mistake: splitting where change coupling is high.** Modules that change together become services that release
together.

**Mistake: a shared database between services.** Schemas can't evolve independently, and services can't deploy or fail
independently.

**Mistake: deep synchronous call chains.** Latency tails and availability multiply along the chain.

**Mistake: the big-bang rewrite.** Years without delivering value, while the monolith keeps changing underneath.

**Mistake: business logic in the API gateway.** A shared, central component accumulates logic nobody owns and every
team must coordinate on.

**Best practice: modular monolith first, with enforced boundaries.** Boundaries become real and testable, and later
extraction follows existing seams.

**Best practice: boundaries around capabilities and data ownership.** Each service owns its data and a coherent piece
of the business.

**Best practice: strangle incrementally, moving data deliberately.** One extraction at a time, one writer per piece of
data at any moment, and the old path removed when done.

---

## 27.8 Real-World Example: Choosing an Extraction With Evidence

The artefact of this chapter is an extraction proposal — the decision record a team writes before carving a service out
of a monolith. This one uses the change-coupling analysis from §27.3, the communication rules from §27.4, and the
migration steps from §27.5.

```markdown
# Proposal: Extract shipping from the commerce monolith

Status: proposed · Owner: fulfilment team · Reviewers: platform, orders

## Why now
- Shipping deploys are blocked ~2×/week by unrelated monolith release freezes; carrier
  integrations need same-day fixes (carriers change APIs with little notice).
- Shipping rate-quote traffic peaks at 8× the rest of checkout during sales; today the
  whole monolith is scaled for it.

## Evidence the boundary is real
- Change coupling (last 12 months, 2,340 commits): shipping ↔ orders 9% of shipping
  commits; shipping ↔ every other module < 3%. By contrast, pricing ↔ orders is 81% —
  pricing is NOT a candidate and stays in the monolith.
- Shipping tables (shipments, labels, carrier_accounts) are written only by shipping code;
  orders READS shipments in 2 places (order detail page, support tool).
- Module boundary check has passed in CI for 4 months with zero exceptions.

## Interface
- Synchronous: POST /quotes (checkout needs rates to show a total) — p99 budget 150 ms,
  with a fallback of cached flat rates if shipping is unavailable.
- Asynchronous: consumes OrderPaid (event-carried state: address, items, weight);
  publishes ShipmentCreated, ShipmentDelivered via outbox.
- Orders' two reads of `shipments` move to consuming ShipmentCreated/Delivered into a
  local projection on the order (status + tracking URL).

## Migration
1. Proxy /shipping/* through the gateway to the monolith (no behaviour change).
2. New service reads a CDC-fed copy of shipping tables; shadow-compare quotes for 2 weeks.
3. Route quotes for 5% of tenants, then 100%. Writes still in the monolith.
4. Move writes: new service becomes writer; monolith consumes events for its projection.
5. Drop shipping tables and code from the monolith. Target: 10 weeks.

## Risks and reversal
- Quote latency adds a network hop: measured 4 ms p99 in-region; within budget.
- Rollback at steps 1–3 is a routing change. After step 4, rollback requires replaying
  events into the monolith's tables — rehearse it before step 4.
```

The decision this proposal defends is **extracting shipping and explicitly not pricing**, even though pricing is the
module the business talks about most and the one with the most complex logic. Complexity and importance feel like
reasons to give something its own service, but the coupling data says that 81% of pricing changes also change orders:
as a separate service, pricing would turn four out of five of its changes into coordinated cross-team releases and
synchronous calls on the checkout path, which is the distributed monolith in miniature. Shipping is simpler but
genuinely independent — it changes on its own, owns its data, has a different scaling profile, and needs a faster
release cadence — which are the actual reasons services pay for themselves. Extraction should follow independence,
not importance.

---

## 27.9 Questions

**1.** MCQ — Which property most distinguishes a modular monolith from a plain monolith? a) Multiple databases b) Enforced module boundaries with public interfaces and owned tables c) Independent deployment d) A service mesh
<details><summary>Answer</summary>**b)** It's still one deployable and one database, but its internal boundaries are explicit and enforced.</details>

**2.** MCQ — What does Conway's law predict? a) Services should be small b) Systems mirror the communication structure of the organisation that builds them c) Every service needs a database d) Monoliths can't scale
<details><summary>Answer</summary>**b)** Boundaries tend to appear where team communication boundaries are.</details>

**3.** MCQ — Which is the strongest signal that two modules should stay in one service? a) They use the same language b) Most changes to one also change the other, and they share transactional invariants c) They're both large d) They have different owners
<details><summary>Answer</summary>**b)** High change coupling and shared invariants mean a boundary would force coordinated releases and distributed transactions.</details>

**4.** MCQ — What is the strangler fig pattern? a) Rewriting the system from scratch b) Incrementally routing functionality from a monolith to new services behind a facade until the monolith can be retired c) Splitting a database d) A load-balancing algorithm
<details><summary>Answer</summary>**b)** Extraction happens one capability at a time, with traffic redirected gradually.</details>

**5.** MCQ — What is a backend for frontend (BFF)? a) A database per client b) A thin service per client type that aggregates internal APIs into the shape that client needs c) An API gateway plugin d) A caching layer
<details><summary>Answer</summary>**b)** It's usually owned by the client team and keeps client-specific aggregation out of shared services.</details>

**6.** MCQ — What does consumer-driven contract testing catch? a) Performance regressions b) Provider changes that break the requests or responses a consumer relies on, before deployment c) Database deadlocks d) Network partitions
<details><summary>Answer</summary>**b)** The provider's CI verifies every consumer's published contract.</details>

**7.** Short — Why do entity services ("user service", "product service") tend to age badly?
<details><summary>Answer</summary>They become CRUD wrappers that every workflow needs, so features touch several services at once and those services are called synchronously from everywhere — high coupling, deep call graphs, and coordinated releases.</details>

**8.** Short — Why is a shared database between services a distributed monolith in disguise?
<details><summary>Answer</summary>Services can't evolve their schemas independently, must coordinate migrations and deployments, can break each other's assumptions about data, and fail together — coupling without the simplicity of one codebase.</details>

**9.** Short — Why measure change coupling from version-control history rather than relying only on import graphs?
<details><summary>Answer</summary>Imports show what code can reach; co-change shows what people actually must edit together. A boundary between frequently co-changing modules forces coordinated releases even if their imports look clean.</details>

**10.** Short — What makes data migration the hardest part of extracting a service?
<details><summary>Answer</summary>The new service must become the sole owner of its data while the monolith still uses it. That requires syncing (CDC), shadowing, moving the writer role at a precise moment, and replacing the monolith's direct reads with events or APIs — with exactly one writer at any time and a rehearsed rollback.</details>

**11.** Short — When does a service mesh help, and what's a common pitfall?
<details><summary>Answer</summary>It helps apply mTLS, timeouts, retries, and telemetry uniformly across many services and languages. A common pitfall is retries configured in both the mesh and the application, which multiply into retry storms (Chapter 28).</details>

**12.** Output — Using §27.3's `changeCoupling` with `minCommits: 1` on the sample history, what are the `together` count and degree for `orders ↔ shipping`?
<details><summary>Answer</summary>**1, 25%.** One commit touched both; shipping appears in 4 commits and orders in 9, so the degree is 1 / min(4, 9) = 25%.</details>

**13.** Output — Using §27.2's `checkImports`, what does `checkImports([{ from: 'src/shipping/label.js', to: 'src/orders/internal/db.js' }])` return?
<details><summary>Answer</summary>`[ 'src/shipping/label.js → src/orders/internal/db.js: reaches into the internals of orders; use src/orders/index.js' ]` — shipping may depend on orders, but only through its public entry point.</details>

**14.** Output — A user request passes synchronously through 6 services in series, each 99.9% available. What's the request path's availability, to two decimal places of a percent?
<details><summary>Answer</summary>**99.40%.** 0.999⁶ ≈ 0.99401.</details>

**15.** Output — A request fans out to 8 services in parallel, each with a 2% chance of exceeding 300 ms. What fraction of requests exceed 300 ms?
<details><summary>Answer</summary>**~15%.** 1 − 0.98⁸ ≈ 0.149 — Chapter 3's fan-out amplification.</details>

**16.** Debug — A company split its monolith into 14 microservices a year ago. Release frequency has dropped, most releases involve three or more services deployed in a specific order, and the staging environment is constantly broken. What's going on, and what would you do?
<details><summary>Answer</summary>A distributed monolith: boundaries cut across capabilities that change together, so features require coordinated multi-service releases. Measure change coupling and coordinated-deploy rates, merge services that change together, introduce consumer-driven contract tests so services can deploy independently, and replace synchronous chains with events where callers don't need answers.</details>

**17.** Debug — A newly extracted billing service reads and writes the monolith's `invoices` table directly. After a monolith migration renamed a column, billing failed in production. What should have been done?
<details><summary>Answer</summary>The extraction stopped at the shared-database stage. Billing must own its data: give it its own store synchronised by CDC during transition, move the writer role to billing, and have the monolith consume billing's events or API — so neither side's schema changes can break the other.</details>

**18.** Debug — The checkout page makes 23 calls to internal services and its p99 is 2.4 seconds, though each service's p99 is under 150 ms. What's happening and how do you fix it?
<details><summary>Answer</summary>Sequential and fanned-out calls compound: tails amplify across parallel calls and add across serial ones. Introduce a BFF or aggregate endpoint that parallelises independent calls and batches others, move rarely-changing data (product details, customer tier) into local copies via events, and make non-critical calls optional with timeouts.</details>

**19.** Debug — After adding a service mesh with retries (3 attempts) to a system whose services already retried 3 times in code, a brief database slowdown turned into a full outage. Why?
<details><summary>Answer</summary>Retries multiplied: each layer's 3 attempts compounded across hops and between mesh and application, so one failing call became dozens, overloading the already slow database. Retry in exactly one layer, with budgets and backoff (Chapter 28).</details>

**20.** Code — Write `findCycles(deps)` that, given `{ module: [dependencies] }`, returns one dependency cycle as an array of module names if any exists, else `null`.
<details><summary>Answer</summary>

```js
function findCycles(deps) {
  const state = new Map(); // undefined = unvisited, 1 = in progress, 2 = done
  const stack = [];
  const visit = (m) => {
    if (state.get(m) === 2) return null;
    if (state.get(m) === 1) return [...stack.slice(stack.indexOf(m)), m]; // back edge: a cycle
    state.set(m, 1); stack.push(m);
    for (const d of deps[m] ?? []) { const c = visit(d); if (c) return c; }
    stack.pop(); state.set(m, 2);
    return null;
  };
  for (const m of Object.keys(deps)) { const c = visit(m); if (c) return c; }
  return null;
}
console.log(findCycles({ orders: ['catalog'], catalog: ['pricing'], pricing: ['orders'], shipping: [] }));
// [ 'orders', 'catalog', 'pricing', 'orders' ]
```
</details>

**21.** Code — Write `callDepth(spans)` that, given trace spans `{ id, parentId, service }`, returns the deepest chain of distinct service-to-service hops.
<details><summary>Answer</summary>

```js
function callDepth(spans) {
  const children = new Map();
  for (const s of spans) { if (!children.has(s.parentId)) children.set(s.parentId, []); children.get(s.parentId).push(s); }
  const depth = (span) => {
    const kids = children.get(span.id) ?? [];
    // Only count a hop when the child runs in a different service
    return Math.max(0, ...kids.map((k) => depth(k) + (k.service !== span.service ? 1 : 0)));
  };
  return Math.max(0, ...(children.get(null) ?? []).map(depth));
}
console.log(callDepth([
  { id: 'a', parentId: null, service: 'gateway' }, { id: 'b', parentId: 'a', service: 'checkout' },
  { id: 'c', parentId: 'b', service: 'pricing' }, { id: 'd', parentId: 'c', service: 'pricing' },
  { id: 'e', parentId: 'd', service: 'tax' },
])); // 3
```
</details>

**22.** Code — Write `routeFor(tenantId, path, rollout)` for a strangler facade: route `/shipping/*` to the new service for tenants in the rollout set (or a hashed percentage), everything else to the monolith.
<details><summary>Answer</summary>

```js
import { createHash } from 'node:crypto';
function routeFor(tenantId, path, { tenants = new Set(), percent = 0 } = {}) {
  if (!path.startsWith('/shipping/')) return 'monolith';
  if (tenants.has(tenantId)) return 'shipping-service';
  // Stable hashing: a tenant stays on the same side as the percentage grows
  const bucket = createHash('md5').update(tenantId).digest().readUInt16BE(0) % 100;
  return bucket < percent ? 'shipping-service' : 'monolith';
}
console.log(routeFor('t-1', '/shipping/quotes', { tenants: new Set(['t-1']) }), routeFor('t-2', '/orders/7', { percent: 100 })); // shipping-service monolith
```
</details>

**23.** Scenario — A 12-engineer startup's monolith is "getting messy". The CTO proposes splitting it into 10 microservices over the next quarter. What do you recommend?
<details><summary>Answer</summary>Modularise first: define module boundaries along capabilities, assign table ownership, enforce dependencies in CI, and fix the worst coupling — which addresses the mess without the distribution costs. Measure change coupling to find seams. Extract a service only for a concrete reason — a module needing independent scaling, a different reliability target, or a separate team's release cadence — and one at a time with the strangler pattern. Ten services for twelve engineers means each person owns about one service on call, with a premium the company can't afford yet.</details>

**24.** Scenario — Two teams share a "customer" service that both need to change weekly, causing release conflicts. One team works on billing, the other on marketing. What do you suggest?
<details><summary>Answer</summary>"Customer" means different things in the two contexts: billing cares about payment methods, tax IDs, and invoices; marketing about preferences, segments, and consent. Split along bounded contexts — billing owns its customer billing data, marketing its profile and preferences — with a small shared identity (customer ID, name, email) published as events. Each team then changes its own model independently, and the shared core changes rarely.</details>

---

## Chapter Summary

- A service boundary means independent deployment, sole data ownership, and team ownership; crossing it turns
  function calls into networked APIs and transactions into sagas.
- Microservices buy independence at a fixed premium; a modular monolith with enforced module boundaries and owned
  tables is the right starting point for most products and makes later extraction mechanical.
- Draw boundaries around business capabilities and bounded contexts, keep transactional invariants inside one
  boundary, and check proposed boundaries against change coupling measured from history.
- Use synchronous calls only where answers are needed, keep request paths shallow, and place gateways, BFFs, and
  meshes deliberately — without business logic in shared edges or retries in two layers.
- Migrate with the strangler pattern: facade, seam, shadow, gradual routing, deliberate data migration with one writer
  at a time, and removal of the old path.
- A distributed monolith — coordinated releases, shared databases, deep synchronous chains — is worse than the monolith
  it replaced; measure the symptoms, and merge or decouple honestly.

---

## Practice Exercises

1. Run `git log --name-only` on a repository you work on, feed it to §27.3's analysis, and list the three most coupled
   module pairs and the three most independent modules.
2. Add a module-boundary check to a codebase (dependency-cruiser, ArchUnit, or a script like §27.2's) and record every
   existing violation as a baseline to burn down.
3. Read a public engineering post about consolidating microservices back into fewer services, and identify which
   symptoms from §27.6 drove the decision.
4. From distributed traces of one user-facing request, compute its call depth and fan-out, and estimate the latency and
   availability penalty using Chapters 3 and 4.
5. Write an extraction proposal like §27.8's for a module in a system you know, including change-coupling evidence, the
   interface, and a data migration plan.

---

## Mini Project: Strangling a Monolith

Start from a small monolithic application (orders, catalogue, shipping, invoicing in one codebase and database) and
extract one capability into a service using everything in this chapter.

- Modularise first: module boundaries, owned tables, and a CI boundary check that passes.
- Use change-coupling analysis on the project's own history (or a supplied synthetic history) to choose what to extract.
- A facade routing by tenant with a percentage rollout, and a shadow mode that compares old and new responses.
- Data migration via CDC into the new service's store, a cut-over of the writer role, and events replacing the
  monolith's direct reads.
- Consumer-driven contract tests between the monolith and the new service, and a rehearsed rollback before the write
  cut-over.

---

**Next:** [Chapter 28 — Resilience Patterns →](28-resilience-patterns.md)
