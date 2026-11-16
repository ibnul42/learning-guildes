# Chapter 7 — Load Balancing

## Learning Objectives

- Choose between layer-4 and layer-7 load balancing for a given protocol and routing need
- Explain why load-aware algorithms such as power-of-two-choices beat round robin on real fleets
- Design health checks that remove broken instances without removing the whole fleet
- Deploy and scale in without dropping requests, using readiness, draining, and slow start
- Decide when session affinity is legitimate and when it's hiding state that should move

---

## 7.1 What a Load Balancer Does

A **load balancer** sits between clients and a pool of interchangeable backends and decides which
backend handles each connection or request. That one decision carries several responsibilities:

| Responsibility | What it means |
|---|---|
| Distribution | Spread work so no backend is overloaded while others idle |
| Health | Stop sending traffic to backends that can't serve it |
| Elasticity | Add and remove backends (deploys, autoscaling) without clients noticing |
| A stable entry point | Clients address one name or IP; the backend set changes underneath |
| Termination | Often terminates TLS and enforces timeouts, limits, and request size caps |

Load balancing happens at several places in a typical system, and the same ideas apply at each:

```text
users ─▶ global/edge LB (anycast, Ch 6) ─▶ regional L7 LB ─▶ API instances
                                                              │
                                     client-side LB in each ──┼─▶ service B instances
                                     instance, or a sidecar   └─▶ service C instances
```

**Server-side** load balancers are a separate tier (a cloud load balancer, HAProxy, Envoy, NGINX).
**Client-side** load balancing puts the logic in the caller: it learns the backend list from
service discovery and picks a backend itself, avoiding an extra hop. A **service mesh** puts a
proxy next to every instance (a sidecar) so callers get client-side balancing without
implementing it. All three must answer the same questions — which algorithm, which health signal,
how to drain — and this chapter is about those answers.

---

## 7.2 Layer 4 Versus Layer 7

A load balancer can operate on connections (transport layer, "L4") or on requests (application
layer, "L7").

| | Layer 4 (TCP/UDP) | Layer 7 (HTTP, gRPC) |
|---|---|---|
| Unit of balancing | A connection | A request |
| Sees | IPs and ports | Methods, paths, headers, cookies, bodies |
| Routing by path/header | No | Yes — `/api/*` to one pool, `/static/*` to another |
| TLS | Passes through, or terminates without inspecting | Terminates and inspects |
| Per-request retries, timeouts | No | Yes |
| HTTP/2 / gRPC multiplexing | Balances the connection once — all its requests go to one backend | Balances each request across backends |
| Overhead | Very low; millions of connections | Higher; parses every request |
| Protocols | Anything over TCP/UDP: databases, MQTT, custom | HTTP-family protocols it understands |
| Typical use | Non-HTTP traffic, extreme throughput, the hop in front of an L7 tier | Web and API traffic, microservices |

Two practical consequences:

- **Multiplexed protocols need L7.** Chapter 5's gotcha: with HTTP/2 or gRPC, an L4 balancer pins
  every request on a connection to one backend. Use L7 or client-side balancing for them.
- **The two are often stacked.** A fleet of L7 proxies is itself a pool of backends, so it usually
  sits behind an L4 tier (or anycast + ECMP routing) that spreads connections across the proxies.

> 💼 **Interview** — "Where would you put the load balancer and what kind?" A good answer names
> both layers: "an L4 network load balancer or anycast in front, L7 proxies that terminate TLS and
> route by path, and client-side balancing for internal gRPC."

---

## 7.3 Algorithms: Why Round Robin Isn't Enough

The obvious algorithm sends each request to the next backend in turn. **Round robin** — and its
cousin, uniform **random** choice — work well when every request costs the same and every backend
is equally fast. Real fleets are neither: request costs vary by orders of magnitude, and backends
differ because of noisy neighbours, GC pauses, cold caches, older hardware, or a failing disk.

Algorithms that look at load do better:

| Algorithm | How it picks | Strengths | Weaknesses |
|---|---|---|---|
| Round robin / random | Ignores load | Simple, stateless | Overloads slow backends; ignores request cost |
| Weighted round robin | In proportion to configured weights | Handles known capacity differences | Weights are static; can't react to a sick backend |
| Least connections / least outstanding requests | Backend with fewest in-flight requests | Adapts to slow backends and expensive requests | Needs accurate, current counts; with many balancers, all pile onto the same "least loaded" backend |
| **Power of two choices (P2C)** | Pick two at random; send to the less loaded | Nearly as good as least-loaded, O(1), robust to stale information | Slightly worse than perfect least-loaded |
| Latency-aware (EWMA) | Prefer backends with low recent latency × load | Routes around slow backends quickly | Can be fooled by fast failures (see the gotcha below) |
| Consistent hashing | Hash a key (user, cache key) to a backend | Locality: the same key hits the same backend's cache | Hot keys overload one backend; uneven by design (Ch 14) |

The difference is easy to underestimate, so here's a simulation: ten backends, two of them three
times slower than the rest, with the fleet at 70% utilisation overall.

```js
// Compare balancing policies on a fleet where two of ten servers are 3× slower
// (a noisy neighbour, a bad disk, an older instance type). Seeded, so output is stable.
function mulberry32(seed) {
  return () => {
    seed |= 0; seed = (seed + 0x6d2b79f5) | 0;
    let t = Math.imul(seed ^ (seed >>> 15), 1 | seed);
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t;
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}
const exp = (rand, mean) => -Math.log(1 - rand()) * mean;

function run(policy, { n = 100_000, seed = 7 } = {}) {
  const rand = mulberry32(seed);
  const serviceMs = [10, 10, 10, 10, 10, 10, 10, 10, 30, 30];   // two slow servers
  const capacity = serviceMs.reduce((s, ms) => s + 1 / ms, 0);   // requests per ms, whole fleet
  const interArrival = 1 / (capacity * 0.7);                     // fleet 70% utilised
  const pending = serviceMs.map(() => []);                       // completion times per server
  let clock = 0, rr = 0;
  const times = [];
  for (let i = 0; i < n; i++) {
    clock += exp(rand, interArrival);
    for (const q of pending) while (q.length && q[0] <= clock) q.shift(); // finished requests leave
    const load = (s) => pending[s].length;
    let s;
    if (policy === 'round-robin') s = rr++ % serviceMs.length;
    else if (policy === 'random') s = Math.floor(rand() * serviceMs.length);
    else if (policy === 'least-outstanding') { s = 0; for (let k = 1; k < pending.length; k++) if (load(k) < load(s)) s = k; }
    else if (policy === 'power-of-two') {
      const a = Math.floor(rand() * serviceMs.length), b = Math.floor(rand() * serviceMs.length);
      s = load(a) <= load(b) ? a : b;
    }
    const q = pending[s];
    const start = q.length ? q[q.length - 1] : clock;             // FIFO: start after the last one
    const done = Math.max(start, clock) + exp(rand, serviceMs[s]);
    q.push(done);
    times.push(done - clock);
  }
  times.sort((a, b) => a - b);
  return { p50: times[Math.floor(n * 0.5)], p99: times[Math.floor(n * 0.99)] };
}

for (const p of ['round-robin', 'random', 'power-of-two', 'least-outstanding']) {
  const r = run(p);
  const fmt = (ms) => (ms >= 10_000 ? `${(ms / 1000).toFixed(0)} s` : `${ms.toFixed(0)} ms`);
  console.log(`${p.padEnd(18)} p50 ${fmt(r.p50).padStart(6)}   p99 ${fmt(r.p99).padStart(6)}`);
}
// round-robin        p50  16 ms   p99  131 s
// random             p50  25 ms   p99  130 s
// power-of-two       p50  15 ms   p99 173 ms
// least-outstanding  p50   9 ms   p99  66 ms
```

The round-robin and random p99s aren't typos. Each slow server receives a tenth of the traffic,
but can only handle about 4% of it — so it's at 180% utilisation, its queue grows without limit
(Chapter 3), and every request routed there waits longer than the last. The *fleet* is 70%
utilised; two of its members are drowning. Load-blind algorithms can't notice. Both load-aware
algorithms keep every queue bounded, and P2C gets most of the benefit of perfect least-loaded with
none of its coordination.

**Why P2C is the usual default.** Perfect least-outstanding needs a current view of every
backend's load. With one balancer that's fine. With fifty balancer instances (or every client
doing client-side balancing), each has a partial, slightly stale view — and when they all see the
same backend as "least loaded", they all send to it at once, overloading it, then all move on
together. This **herding** makes least-loaded oscillate. Picking two random candidates and taking
the better one breaks the herd: different balancers sample different pairs, so no single backend
gets everyone's traffic, yet a badly overloaded backend almost always loses its comparisons. Envoy,
gRPC, and most modern proxies use P2C (with in-flight requests as the load signal) by default.

> ⚠️ **Gotcha** — **Fast failures attract traffic.** A backend that's broken — say it returns
> `500` in 2 ms because its database connection is gone — looks wonderful to latency-aware and
> least-outstanding algorithms: it finishes requests faster than anyone and always has the fewest
> in flight. Without failure-aware ejection, the balancer sends it *more* traffic, turning one bad
> instance into a large error rate. Every load-aware balancer needs passive health checking
> (§7.4) to counter this "black hole".

---

## 7.4 Health Checks

A balancer must stop sending traffic to backends that can't serve it — but every health-check
design is a trade-off between removing broken instances quickly and *not* removing healthy ones by
mistake. The second failure is often worse: a bad health check can take a healthy fleet out of
service all at once.

### Active and passive checks

| | Active health checks | Passive health checks (outlier detection) |
|---|---|---|
| How | The balancer probes `GET /healthz` every few seconds | The balancer watches real traffic for errors and timeouts |
| Detects | Process down, listener closed, explicit "not ready" | Real failures on real requests, including partial ones |
| Speed | Probe interval × failure threshold (e.g. 5 s × 3 = 15 s) | As fast as errors accumulate — often a few requests |
| Blind spot | A backend that answers `/healthz` but fails real requests | A backend that receives no traffic |
| Use | Both, together | Both, together |

### Liveness, readiness, and how deep to check

Two different questions are often confused:

- **Liveness** — "is this process stuck beyond recovery?" If not live, *restart it*.
- **Readiness** — "should this instance receive traffic right now?" If not ready, *stop routing to
  it* but leave it running — it may be warming up, draining, or temporarily overloaded.

The difficult design choice is **how deep** a check goes:

| | Shallow check | Deep check |
|---|---|---|
| What it verifies | The process is up and its event loop/thread pool responds | The process *and* its dependencies (database, cache, downstream services) |
| Catches | Crashed or wedged processes | Instances that are up but can't do useful work |
| Danger | Routes traffic to an instance that can't reach its database | **When a shared dependency fails, every instance fails its check at once** |

The deep-check failure mode is one of the classic cascading outages. The database has a 20-second
blip. Every app instance's deep health check fails. The load balancer marks every instance
unhealthy and removes them all — so now, instead of returning errors for 20 seconds, the service
returns nothing, and when the database recovers the instances must pass several consecutive checks
before receiving traffic again. A brief dependency hiccup has become a full outage that outlasts
its cause.

The usual resolution:

- **Readiness checks verify only what's local to the instance** — it's started, its caches are
  warm, it isn't draining, it isn't overloaded. They don't fail because a *shared* dependency is
  down; that's not something removing the instance can fix.
- **Dependency health is reported separately** for dashboards and alerts, and handled per request
  with timeouts, fallbacks, and circuit breakers (Chapter 28).
- **Balancers fail open.** If more than some fraction (say 50%) of backends look unhealthy, the
  balancer ignores health status and routes to all of them, on the theory that a mass failure is
  more likely a broken health signal than a broken fleet. Envoy calls this the "panic threshold";
  cloud load balancers have equivalents.

**Hysteresis** prevents flapping: require several consecutive failures to mark an instance
unhealthy, and several consecutive successes to restore it.

---

## 7.5 Deploys, Draining, and Slow Start

Most dropped requests in a healthy system happen at the edges of an instance's life: when it
starts, and when it stops. Getting both right is what makes deploys and autoscaling invisible.

### Graceful shutdown

When an instance is told to stop (a `SIGTERM` from the orchestrator), the correct sequence is:

```text
1. SIGTERM received
2. Mark not-ready → health checks start failing; balancers stop sending NEW requests
3. Keep serving for a grace period → requests already routed here (and in-flight in the LB) finish
4. Stop accepting connections; close idle keep-alive connections
5. Wait for in-flight requests to complete (bounded by a deadline)
6. Close pools, flush logs and metrics, exit
```

The step most often missing is **3**. Balancers and service-discovery caches take a few seconds to
notice the instance is going away; an instance that stops listening the moment it gets `SIGTERM`
refuses connections that were already routed to it. The fix is a short delay after marking
not-ready (often 5–15 seconds) before closing the listener.

```js
// graceful-shutdown.mjs — the order matters more than the code
import http from 'node:http';

let ready = true;
const server = http.createServer((req, res) => {
  if (req.url === '/readyz') { res.statusCode = ready ? 200 : 503; return res.end(); }
  setTimeout(() => res.end('ok'), 50); // stand-in for real work
});
server.listen(8080);

process.on('SIGTERM', async () => {
  ready = false;                                   // 2. readiness fails → LB stops routing here
  await new Promise((r) => setTimeout(r, 10_000)); // 3. let in-flight routing decisions land
  server.close(() => process.exit(0));             // 4–5. stop accepting; exit when requests finish
  server.closeIdleConnections();                   //      idle keep-alive sockets close now
  setTimeout(() => process.exit(1), 20_000).unref(); // 5. hard deadline for stragglers
});
```

The orchestrator's termination grace period must be longer than the sum of these waits, or it
kills the process mid-drain. And the delay in step 3 must be longer than the balancer's health
check interval × failure threshold, or the balancer won't have noticed yet.

### Connection draining at the balancer

The balancer's side of the same contract: when a backend is deregistered, stop sending new
requests but let existing ones finish within a **draining timeout** (also called deregistration
delay). Long-lived connections — WebSockets, streaming RPCs — never finish on their own, so the
backend must actively close them (with a reconnect hint) during shutdown, and clients must
reconnect gracefully. Chapter 9 covers this for real-time systems.

### Slow start

A freshly started instance has cold caches, an unwarmed JIT, empty connection pools, and possibly a
lazily initialised dependency. If a least-outstanding balancer sees it as "zero in-flight" and
sends it a full share of traffic immediately, its first requests are slow, it builds a queue, and
in bad cases it fails readiness and is removed — then re-added, cold again. **Slow start** ramps a
new backend's share of traffic up over a configured window (say 30–60 seconds). Combine it with an
explicit warm-up in the instance's startup: prime caches and open pools *before* reporting ready.

---

## 7.6 Session Affinity and Hash-Based Routing

**Session affinity** (sticky sessions) routes all requests from one client to the same backend,
usually via a cookie set by the balancer. Chapter 1 showed why it's the wrong fix for in-memory
session state. There are legitimate reasons to route requests for the same *key* to the same
backend, though:

| Reason | Legitimate? | Better alternative if not |
|---|---|---|
| In-memory login sessions | No — users lose state on every deploy | Shared session store or stateless tokens |
| A local cache that's expensive to rebuild (per-user or per-tenant data) | Yes, as an optimisation — correctness must not depend on it | — |
| Long-lived connection state (WebSocket, game session) | Yes, inherently — the connection *is* on one backend | — |
| In-process aggregation (a stream processor's partition) | Yes, but via partitioning, not cookies (Ch 14, 25) | — |

When affinity is for cache locality, use **consistent hashing** on the key rather than a cookie:
the same tenant or object ID maps to the same backend, and adding or removing a backend only remaps
a small fraction of keys (Chapter 14 builds the hash ring). Two refinements matter in practice:

- **Bounded loads.** Pure consistent hashing sends all traffic for a hot key to one backend.
  *Consistent hashing with bounded loads* caps each backend at, say, 125% of the average; once a
  backend is full, keys that hash to it spill to the next backend on the ring.
- **Affinity must be advisory.** If the preferred backend is unhealthy or full, the request goes
  elsewhere and pays a cache miss. A design that *breaks* when affinity breaks has turned an
  optimisation into hidden state.

---

## 7.7 Scaling and Protecting the Load Balancer

A load balancer is on the path of every request, so it must not become the single point of failure
or the bottleneck it was supposed to remove.

- **Redundancy.** Managed cloud load balancers are internally redundant across zones. Self-run
  proxies (HAProxy, Envoy, NGINX) run as a fleet behind an L4 tier, DNS with several addresses, or
  ECMP/anycast routing, so losing one proxy loses only its in-flight connections.
- **Capacity.** L7 proxies are CPU-bound on TLS and parsing. Size them like any other tier —
  peak requests and connections, at a target utilisation (Chapter 3) — and note that cloud load
  balancers can need time (or a pre-warming request to the provider) to scale up for sudden 10×
  spikes.
- **Limits as protection.** The balancer is the natural place for coarse protection: maximum
  connections, request-rate limits per client (Chapter 29), header and body size caps, and
  timeouts that stop slow clients from holding connections indefinitely (the "slowloris" attack).
- **Timeouts that match the stack.** The balancer's idle timeout must fit Chapter 5's ordering —
  shorter than backends' keep-alive timeouts, longer than clients' — and its request timeout must
  fit the latency budget (Chapter 3).
- **Zone awareness.** Prefer backends in the caller's zone when they're healthy and not overloaded,
  falling back across zones otherwise. It cuts latency and cross-zone data charges (Chapter 5),
  but only works if each zone has enough capacity for its own traffic.

---

## 7.8 Common Mistakes and Best Practices

**Mistake: round robin on a heterogeneous fleet.** A slow backend receives the same share as a
fast one, its queue grows without bound, and the tail explodes while fleet utilisation looks fine.

**Mistake: deep health checks on shared dependencies.** One database blip fails every instance's
check at once, and the balancer removes the entire fleet.

**Mistake: no fail-open threshold.** When most backends look unhealthy, routing to the few
"healthy" ones crushes them; a panic threshold spreads load across all.

**Mistake: closing the listener on `SIGTERM`.** Requests the balancer already routed are refused;
mark not-ready, wait, then drain.

**Mistake: termination grace period shorter than the drain.** The orchestrator kills the process
mid-request, dropping the in-flight work draining was supposed to protect.

**Mistake: latency-aware balancing without outlier ejection.** Fast-failing backends look healthiest
and attract more traffic.

**Mistake: L4 balancing for gRPC or HTTP/2.** Each client's connection pins all its requests to one
backend.

**Best practice: power-of-two-choices on in-flight requests as the default.** It adapts to slow
backends and expensive requests without herding.

**Best practice: readiness for local state, separate signals for dependencies.** Removing an
instance should only happen for a problem removing it fixes.

**Best practice: slow start plus explicit warm-up.** New instances ramp up instead of being
flattened by a full share of cold traffic.

---

## 7.9 Real-World Example: A Client-Side P2C Balancer With Outlier Ejection

Client-side balancing is how most service-to-service traffic is spread in practice — in gRPC
clients, mesh sidecars, and service libraries. This one picks with power-of-two-choices on
in-flight requests, ejects backends after consecutive failures, never ejects more than half the
fleet, and falls back to all backends ("panic mode") if too many look unhealthy.

```js
// p2c-balancer.mjs — client-side load balancer: power-of-two-choices on in-flight requests,
// with passive outlier ejection and a panic threshold. Dependency-free.

export class P2CBalancer {
  #backends; #rand; #now; #opts;

  constructor(addresses, { rand = Math.random, now = Date.now, ejectAfterFailures = 5,
    ejectMs = 30_000, maxEjectedFraction = 0.5 } = {}) {
    if (addresses.length === 0) throw new Error('at least one backend required');
    this.#rand = rand;
    this.#now = now;
    this.#opts = { ejectAfterFailures, ejectMs, maxEjectedFraction };
    this.#backends = addresses.map((address) => ({
      address, inFlight: 0, ewmaMs: 0, consecutiveFailures: 0, ejectedUntil: 0, picks: 0,
    }));
  }

  #candidates() {
    const t = this.#now();
    const healthy = this.#backends.filter((b) => b.ejectedUntil <= t);
    // Panic mode: if too few backends look healthy, the health signal is more likely wrong
    // than the whole fleet being down — spread load over everyone rather than crushing a few
    return healthy.length >= Math.ceil(this.#backends.length * (1 - this.#opts.maxEjectedFraction))
      ? healthy : this.#backends;
  }

  pick() {
    const pool = this.#candidates();
    const a = pool[Math.floor(this.#rand() * pool.length)];
    let b = pool[Math.floor(this.#rand() * pool.length)];
    if (pool.length > 1) while (b === a) b = pool[Math.floor(this.#rand() * pool.length)];
    // Fewer in-flight requests wins; break ties on recent latency
    const chosen = a.inFlight < b.inFlight ? a : b.inFlight < a.inFlight ? b : (a.ewmaMs <= b.ewmaMs ? a : b);
    chosen.inFlight++;
    chosen.picks++;
    let finished = false;
    return {
      address: chosen.address,
      // Callers must report every outcome, or in-flight counts drift upward forever
      done: ({ ok, latencyMs }) => {
        if (finished) return;
        finished = true;
        chosen.inFlight--;
        chosen.ewmaMs = chosen.ewmaMs ? 0.8 * chosen.ewmaMs + 0.2 * latencyMs : latencyMs;
        if (ok) { chosen.consecutiveFailures = 0; return; }
        if (++chosen.consecutiveFailures >= this.#opts.ejectAfterFailures && this.#canEject()) {
          chosen.ejectedUntil = this.#now() + this.#opts.ejectMs;
          chosen.consecutiveFailures = 0;
        }
      },
    };
  }

  #canEject() {
    const t = this.#now();
    const ejected = this.#backends.filter((b) => b.ejectedUntil > t).length;
    return (ejected + 1) / this.#backends.length <= this.#opts.maxEjectedFraction;
  }

  snapshot() {
    const t = this.#now();
    return this.#backends.map(({ address, inFlight, picks, ejectedUntil }) =>
      ({ address, inFlight, picks, ejected: ejectedUntil > t }));
  }
}

// Demo: five backends; 'c' fails every request — fast, in 2 ms, the black-hole case
let seed = 1;
const rand = () => ((seed = (seed * 16807) % 2147483647) - 1) / 2147483646; // deterministic
let t = 0;
const lb = new P2CBalancer(['a', 'b', 'c', 'd', 'e'], { rand, now: () => t });
for (let i = 0; i < 1_000; i++, t += 10) {
  const h = lb.pick();
  h.done({ ok: h.address !== 'c', latencyMs: h.address === 'c' ? 2 : 20 });
}
console.log(lb.snapshot().map((b) => `${b.address}:${b.picks}${b.ejected ? ' (ejected)' : ''}`).join('  '));
// a:254  b:240  c:5 (ejected)  d:263  e:238
```

The broken backend `c` fails fast, which would win every latency tie-break — but after five
consecutive failures it's ejected, so it receives 5 of 1,000 requests instead of the lion's share.

The decision to defend is the **cap on ejection (`maxEjectedFraction`) and the panic fallback**,
which deliberately keep sending traffic to backends that look broken. It feels wrong: if half the
fleet is failing, why route anything to it? Because the balancer can't tell "half the backends are
broken" from "a shared dependency blipped and every backend failed a few requests". Without the
cap, a 10-second database hiccup ejects every backend for 30 seconds, and a client-side outage
outlives the real one threefold; with uncapped ejection on fifty clients, the healthy remainder
gets the whole load and fails next. Outlier ejection is for *outliers* — a minority of instances
behaving differently from the rest — and the cap enforces exactly that definition.

---

## 7.10 Questions

**1.** MCQ — Which capability requires a layer-7 load balancer? a) Balancing raw TCP connections b) Routing `/api/*` and `/static/*` to different pools c) Handling millions of idle connections cheaply d) Load balancing a PostgreSQL wire protocol
<details><summary>Answer</summary>**b)** Path-based routing needs the balancer to parse HTTP requests. The others are L4 strengths.</details>

**2.** MCQ — Why does power-of-two-choices avoid the herding that least-loaded suffers from with many balancers? a) It uses more accurate load data b) Different balancers sample different random pairs, so they don't all pick the same backend c) It ignores load entirely d) It uses consistent hashing
<details><summary>Answer</summary>**b)** Randomised sampling spreads balancers' choices while still steering away from overloaded backends.</details>

**3.** MCQ — A backend returns HTTP 500 in 2 ms on every request. Under a pure least-outstanding-requests balancer with no outlier detection, what happens? a) It receives no traffic b) It receives about the same traffic as others c) It receives more traffic than healthy backends d) The balancer crashes
<details><summary>Answer</summary>**c)** It completes requests fastest and so always has the fewest in flight — the "black hole" effect.</details>

**4.** MCQ — What's the main danger of a health check that queries the shared database? a) It's slower b) A database blip fails every instance's check simultaneously, removing the whole fleet c) It leaks credentials d) It increases database storage
<details><summary>Answer</summary>**b)** Removing instances can't fix a shared dependency, and removing all of them turns a blip into an outage.</details>

**5.** MCQ — In a graceful shutdown, why wait several seconds after marking an instance not-ready before closing its listener? a) To flush logs b) Because balancers and discovery caches take time to notice and stop routing to it c) To save CPU d) To let the JIT cool down
<details><summary>Answer</summary>**b)** Requests already routed — or routed before the next health check — would otherwise be refused.</details>

**6.** MCQ — Which use of request affinity is legitimate? a) Keeping login sessions in process memory b) Routing a tenant's requests to the backend holding its warm cache, falling back elsewhere if needed c) Pinning users so deploys don't affect them d) Avoiding a shared database
<details><summary>Answer</summary>**b)** Affinity as an optimisation is fine, provided correctness doesn't depend on it.</details>

**7.** Short — What's the difference between liveness and readiness?
<details><summary>Answer</summary>Liveness asks whether the process is stuck and should be restarted; readiness asks whether it should receive traffic right now. A warming or draining instance is live but not ready.</details>

**8.** Short — Why do load-blind algorithms fail badly when one backend is slower, even if the fleet is lightly loaded?
<details><summary>Answer</summary>The slow backend receives the same share as fast ones; if that share exceeds its capacity, its queue grows without bound regardless of fleet-wide utilisation, and every request routed to it waits longer and longer.</details>

**9.** Short — What is slow start, and what problem does it solve?
<details><summary>Answer</summary>Ramping a new backend's share of traffic up over a window. It prevents a cold instance — empty caches, cold JIT, no pooled connections — from being flooded by a load-aware balancer that sees it as idle.</details>

**10.** Short — What does "consistent hashing with bounded loads" add to plain consistent hashing?
<details><summary>Answer</summary>A cap on each backend's load relative to the average; once a backend is full, keys that hash to it spill to the next backend on the ring, so a hot key can't overload one backend.</details>

**11.** Short — Why should long-lived WebSocket connections be closed actively during shutdown?
<details><summary>Answer</summary>They never finish on their own, so draining would wait until the hard deadline and then cut them abruptly. Closing them with a reconnect hint lets clients reconnect to other instances gracefully.</details>

**12.** Output — Using the §7.9 `P2CBalancer` with 4 backends and the default `maxEjectedFraction` of 0.5, what is the largest number of backends that can be ejected at once?
<details><summary>Answer</summary>**2.** `#canEject` allows an ejection only if (ejected + 1) / 4 ≤ 0.5, so the third ejection is refused.</details>

**13.** Output — Round robin sends 1/10 of 600 requests/s to a backend that can serve 40 requests/s. What is that backend's utilisation, and is its queue stable?
<details><summary>Answer</summary>**150%, unstable.** It receives 60 requests/s against a capacity of 40; the queue grows by ~20 requests every second until requests time out.</details>

**14.** Output — A balancer probes every 5 seconds and marks a backend unhealthy after 3 consecutive failures. What's the worst-case time from a backend failing to being removed, ignoring probe timeouts?
<details><summary>Answer</summary>**~15 seconds.** The failure may occur just after a successful probe; three failed probes at 5-second intervals follow (between 10 and 15 s, worst case 15 s).</details>

**15.** Output — What does this print?
```js
const pickTwo = (loads, i, j) => (loads[i] <= loads[j] ? i : j);
const loads = [7, 2, 9, 2];
console.log(pickTwo(loads, 0, 2), pickTwo(loads, 2, 3), pickTwo(loads, 1, 3));
```
<details><summary>Answer</summary>`0 3 1` — the less-loaded candidate wins each comparison; on a tie the first candidate wins.</details>

**16.** Debug — Every deploy causes a spike of ~200 `502` errors at the load balancer, lasting a few seconds per instance replaced. The app exits immediately on `SIGTERM`. Cause and fix?
<details><summary>Answer</summary>Instances stop listening before the balancer notices they're going away, so requests routed in that window get connection refused or reset. On `SIGTERM`, fail readiness, wait longer than the health-check detection time, then stop accepting and drain in-flight requests; set the orchestrator's grace period above the total.</details>

**17.** Debug — During a 15-second cache-cluster outage, the API returned no responses at all for about 2 minutes, though the API's own code handles cache errors by falling back to the database. What likely happened?
<details><summary>Answer</summary>The readiness check pinged the cache, so every instance failed it and the balancer removed the whole fleet; instances then needed several consecutive passes to be re-added. Remove shared dependencies from readiness, report them separately, and enable a fail-open/panic threshold.</details>

**18.** Debug — After autoscaling adds instances during a spike, the new instances fail readiness within a minute and are removed, then re-added and removed again. CPU on them spikes immediately on registration. Cause and fix?
<details><summary>Answer</summary>Cold instances receive a full share of traffic immediately (load-aware balancers see them as idle), overload while warming, and fail readiness. Enable slow start on the balancer, and warm caches and connection pools before reporting ready.</details>

**19.** Debug — A client-side balancer across 20 backends works well with 3 client instances. After scaling clients to 60 instances using least-outstanding-requests with load data refreshed every second, backend load oscillates wildly. Why, and what's the fix?
<details><summary>Answer</summary>Herding: every client sees the same stale "least loaded" backend and sends to it simultaneously, overloading it, then all move on together. Switch to power-of-two-choices on each client's own in-flight counts, which needs no shared state and doesn't herd.</details>

**20.** Code — Write `weightedPicker(backends)` where `backends` is `[{ address, weight }]`, returning a `pick(rand)` function that selects in proportion to weight using a single random number in [0, 1).
<details><summary>Answer</summary>

```js
function weightedPicker(backends) {
  const total = backends.reduce((s, b) => s + b.weight, 0);
  if (!(total > 0)) throw new RangeError('weights must sum to > 0');
  return (rand = Math.random) => {
    let r = rand() * total;
    for (const b of backends) { if ((r -= b.weight) < 0) return b.address; }
    return backends.at(-1).address; // floating-point safety
  };
}
const pick = weightedPicker([{ address: 'a', weight: 3 }, { address: 'b', weight: 1 }]);
console.log(pick(() => 0.5), pick(() => 0.8)); // a b
```
</details>

**21.** Code — Write `isReady({ warmedUp, draining, inFlight, maxInFlight })` implementing a local-only readiness check: not ready while warming, while draining, or when saturated.
<details><summary>Answer</summary>

```js
function isReady({ warmedUp, draining, inFlight, maxInFlight }) {
  if (draining) return false;                  // shutting down: let the LB route elsewhere
  if (!warmedUp) return false;                 // caches and pools not primed yet
  return inFlight < maxInFlight;               // saturated: shed new traffic to other instances
  // Deliberately absent: shared database/cache checks — removing this instance can't fix those
}
console.log(isReady({ warmedUp: true, draining: false, inFlight: 10, maxInFlight: 100 })); // true
```
</details>

**22.** Code — Write `slowStartWeight({ addedAt, now, windowMs })` returning a weight multiplier that ramps linearly from 0.1 to 1 over the window after a backend is added.
<details><summary>Answer</summary>

```js
function slowStartWeight({ addedAt, now, windowMs = 60_000, floor = 0.1 }) {
  const progress = Math.min(1, Math.max(0, (now - addedAt) / windowMs));
  return floor + (1 - floor) * progress; // never zero, so the instance still gets some traffic to warm on
}
console.log(slowStartWeight({ addedAt: 0, now: 30_000 }).toFixed(2), slowStartWeight({ addedAt: 0, now: 90_000 })); // 0.55 1
```
</details>

**23.** Scenario — A team runs a stateful in-memory recommendation model per tenant on each API instance and wants sticky sessions so tenants hit "their" instance. Tenants vary in traffic by 1,000×. What do you advise?
<details><summary>Answer</summary>Use consistent hashing on tenant ID with bounded loads rather than cookies, so locality survives client changes and a big tenant spills to additional instances instead of overloading one. Make affinity advisory: a request routed elsewhere must still work, loading the model lazily (or from a shared cache). If the largest tenants need dedicated capacity, give them their own pool. If models are small, consider loading all of them everywhere and dropping affinity entirely.</details>

**24.** Scenario — You're choosing a health-check design for a payment API whose instances depend on a database, a fraud service, and a cache. The fraud service is occasionally slow for a minute at a time. What do you check where?
<details><summary>Answer</summary>Readiness checks only local state: process started, pools initialised, not draining, not saturated. Liveness checks that the event loop responds. Database, fraud, and cache health go to dashboards and alerts, not readiness — removing instances can't fix them. Per request, the fraud call has a timeout and a defined fallback policy (e.g. queue for review or decline above a threshold), and a circuit breaker. Add passive outlier detection at the balancer for instances that fail differently from their peers, with a panic threshold.</details>

---

## Chapter Summary

- L4 balances connections and L7 balances requests; multiplexed protocols like HTTP/2 and gRPC need
  L7 or client-side balancing, and the two layers are often stacked.
- Load-blind algorithms let a single slow backend's queue grow without bound; power-of-two-choices
  on in-flight requests adapts to slow backends and expensive requests without herding.
- Load-aware balancers are attracted to fast-failing backends, so they need passive outlier
  ejection — capped, so a shared blip can't eject the fleet.
- Readiness should reflect only what removing the instance can fix; deep checks on shared
  dependencies and no fail-open threshold turn dependency blips into total outages.
- Deploys stay invisible when shutdown is ordered — not-ready, wait, drain, exit — within the
  orchestrator's grace period, and when new instances warm up and ramp in with slow start.
- Affinity is legitimate as an optimisation or for connection state, implemented with consistent
  hashing and bounded loads, never as a place to hide state correctness depends on.

---

## Practice Exercises

1. Extend the §7.3 simulation with a "least-outstanding with stale data" policy, where each of 20
   simulated balancers refreshes its view every 100 ms, and show herding.
2. Add slow-start ramping to the §7.9 balancer and simulate adding a cold backend whose first 500
   requests take 10× longer.
3. Read your cloud load balancer's documentation for its health-check defaults, deregistration
   delay, and whether it fails open when all targets are unhealthy.
4. Run an app behind a local proxy (HAProxy, Envoy, or NGINX), kill it mid-load-test with
   `SIGTERM`, and count dropped requests before and after implementing §7.5's shutdown sequence.
5. Implement consistent hashing with bounded loads and measure the maximum backend load for a
   Zipf-distributed key workload, with and without the bound.

---

## Mini Project: A Small L7 Load Balancer

Build an HTTP reverse proxy in Node that balances across a configurable set of backends and
implements the behaviours in this chapter, then use it to reproduce and fix each failure mode.

- Pluggable algorithms: round robin, random, least-outstanding, and power-of-two-choices.
- Active health checks with hysteresis, passive outlier ejection, and a panic threshold.
- Backend registration and deregistration at runtime, with connection draining and slow start.
- Per-request timeouts and a single retry for idempotent methods only, on a different backend.
- A `/__lb` endpoint showing per-backend in-flight counts, ejection state, and picks, plus a test
  script that starts backends of different speeds and shows each algorithm's p99.

---

**Next:** [Chapter 8 — API Design →](08-api-design.md)
