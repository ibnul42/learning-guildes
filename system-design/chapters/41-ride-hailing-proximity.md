# Chapter 41 — Design Ride-Hailing and Proximity Search

## Learning Objectives

- Scope a ride-hailing system and recognise that driver location updates dominate its load
- Index constantly moving points with geohash, quadtree, or hierarchical cells, and search outward for the nearest
- Ingest millions of location updates per second into an in-memory, latest-value-only index, sharded by geography
- Match riders to drivers by ETA and claim drivers atomically so no driver is double-booked
- Model the trip lifecycle as a durable state machine and plan for regional hot spots and failures

---

## 41.1 Requirements and Scope

A ride-hailing service connects riders with nearby drivers in real time. Its distinctive problem is **proximity search over
points that never stop moving**: millions of drivers report their position every few seconds, and every ride request asks
"which available drivers are closest right now?"

**Functional requirements**

- Drivers go online and offline; while online, their app reports location every few seconds.
- Riders see nearby cars on a map and request a ride from A to B.
- The system finds suitable nearby drivers, offers the ride, and assigns it to one that accepts.
- Both sides track the trip through its lifecycle: requested, matched, driver arriving, in progress, completed, cancelled.

**Out of scope**: pricing and surge (mentioned in §41.8), payments (Chapter 43), routing and maps (bought, not built), ratings,
driver onboarding.

**Non-functional requirements**

| Requirement | Target |
|---|---|
| Nearby-drivers query | p99 < 100 ms |
| Match time | A ride is offered to a driver within a few seconds of request |
| Location freshness | Driver positions at most a few seconds old |
| Correctness | A driver is never assigned two rides at once; a trip's state is never lost |
| Availability | 99.99% for requesting rides and trip state |
| Scale | 5 M drivers online at peak worldwide; hundreds of cities |

---

## 41.2 Estimation

```js
// Ride-hailing: location updates dominate everything else
const onlineDrivers = 5e6, updateEverySec = 4, bytesPerUpdate = 100;
const rideRequestsPerSec = 2_000, appOpensWithMapPerSec = 100_000;
console.log(`location updates ~${(onlineDrivers / updateEverySec / 1e6).toFixed(2)} M/s, ~${((onlineDrivers / updateEverySec) * bytesPerUpdate / 1e6).toFixed(0)} MB/s ingest`);
console.log(`latest positions in memory: ~${((onlineDrivers * 64) / 1e9).toFixed(2)} GB | history: ~${((onlineDrivers / updateEverySec) * bytesPerUpdate * 86_400 / 1e12).toFixed(1)} TB/day`);
console.log(`proximity queries ~${(rideRequestsPerSec + appOpensWithMapPerSec) / 1000}k/s (map views dominate ride requests ${appOpensWithMapPerSec / rideRequestsPerSec}:1)`);
// location updates ~1.25 M/s, ~125 MB/s ingest
// latest positions in memory: ~0.32 GB | history: ~10.8 TB/day
// proximity queries ~102k/s (map views dominate ride requests 50:1)
```

The shape is unusual:

- **1.25 million writes per second** — each one overwriting a driver's previous position. No disk-based database should take this
  as row updates; it's an in-memory index problem.
- **The current state is tiny**: all drivers' latest positions fit in well under a gigabyte. The challenge is update rate and query
  latency, not size.
- **History is large** (~11 TB/day) but write-only and analytical — trip reconstruction, ETA training, disputes — so it goes to a log
  and cold storage, off the hot path.
- **Map views outnumber ride requests 50:1**, and they need only approximate results ("cars nearby"), which permits caching and coarser
  answers.
- **Everything is local.** A rider in Lisbon never needs drivers in Tokyo, so the whole system partitions naturally by geography.

---

## 41.3 API and Data Model

```text
driver  → WS  location { lat, lng, heading, speed, ts, status }          every ~4 s
rider   → GET /v1/nearby?lat=&lng=                    → { drivers: [{ id, lat, lng, etaSec }] } (approximate, cached briefly)
rider   → POST /v1/rides   Idempotency-Key            { pickup, dropoff, product } → { rideId, status: "requested" }
driver  ← WS  offer { rideId, pickup, expiresInSec: 15 }
driver  → POST /v1/rides/:id/accept | decline
both    ← WS  ride updates { status, driverLocation, eta }
```

| Data | Store | Notes |
|---|---|---|
| Latest driver positions | In-memory geo index, sharded by region/cell | Latest value only; expires if a driver goes silent |
| Driver state (`offline`, `available`, `offered`, `on_trip`) | Fast store with atomic conditional updates | The source of truth for dispatch (§41.7) |
| Trips | Replicated relational database, partitioned by city or trip ID | Durable state machine with history |
| Location history | Append-only log → object storage | Analytics, ETA models, disputes |

---

## 41.4 High-Level Design

```text
driver apps ═ws═▶ gateways ─▶ location service ─┬─▶ geo index shard (city/region) — latest positions
                                                └─▶ location log ─▶ history store, ETA training
rider app ─▶ API ─▶ nearby service ─▶ geo index (approximate, cached)
rider app ─▶ API ─▶ ride service ─▶ trips DB (state machine)
                        └─▶ matching service ─▶ geo index (candidates) ─▶ ETA service ─▶ rank
                                   └─▶ claim driver (atomic) ─▶ offer via gateway ─▶ accept/decline/timeout
```

The design is organised around **geographic partitions**: each city (or region of a large city) is served by its own geo index shard and
matching workers, so a request touches only the shard for its location. Gateways (Chapter 9) hold the drivers' and riders' persistent
connections. The trip service holds the durable state; everything about locations is ephemeral and rebuildable within seconds from the
next round of updates.

---

## 41.5 Deep Dive: Indexing Moving Points

A naive "compute the distance to every driver" scans millions of points per query. Spatial indexes divide the map so a query looks only at
nearby areas:

| Index | How | Strengths | Weaknesses |
|---|---|---|---|
| **Geohash** | Interleave latitude and longitude bits into a base32 string; a shared prefix means a shared cell | Simple, string-sortable, works in any key-value store; prefix length sets cell size | Cells are rectangles that vary with latitude; edge cases at cell boundaries require searching neighbours |
| **Quadtree** | Recursively split regions into four until each holds at most N points | Adapts to density (small cells downtown, large in the countryside) | Rebalancing as points move is costly for very high update rates |
| **Hierarchical cells** (S2, H3) | Sphere divided into cells at many resolutions (S2: quadrilaterals; H3: hexagons) | Uniform-ish cells, good neighbour and covering operations; hexagons have equidistant neighbours | Libraries needed; more concepts |
| **R-tree / PostGIS** | Bounding-box tree in a database | Rich queries (polygons, routes) | Too slow for million-per-second updates of the same points |

For moving drivers the standard choice is a **fixed grid of cells** — geohash, S2, or H3 at a resolution where a cell is a few hundred
metres to a kilometre wide — held in memory as `cell → set of drivers` plus `driver → current cell`. An update is O(1): if the driver changed
cell, move it between two sets. A query starts with the rider's cell and its neighbours and **expands ring by ring** until it has enough
candidates close enough, which handles both dense downtowns (one ring suffices) and sparse suburbs (several rings). The real-world example
in §41.10 implements exactly this.

> ⚠️ **Gotcha** — **The nearest driver may be in the next cell.** A rider standing near a cell's edge can be 20 metres from a driver in the
> adjacent cell and 600 metres from every driver in their own. Always search the neighbouring cells (the ring around the rider's cell), and
> only stop expanding once the candidates found are closer than the distance to the next ring.

> 💼 **Interview** — Interviewers usually want the comparison: geohash (simple, prefix-based, boundary issues), quadtree (density-adaptive,
> costly to rebalance under constant motion), and S2/H3 (production-grade cells). Then the key insight: with constant updates, keep only the
> latest position per driver in an in-memory grid and expand rings at query time.

---

## 41.6 Deep Dive: High-Frequency Location Ingest

1.25 million updates per second, each overwriting the last, needs a path designed for it:

- **Persistent connections.** Drivers' apps send updates over the WebSocket (or a lightweight protocol) they already hold for offers and trip
  updates, not as separate HTTP requests.
- **Geographic sharding.** The location service routes each update to the geo index shard owning that location — by city or by a coarse cell
  prefix. A shard holds tens of thousands of drivers and handles tens of thousands of updates per second in memory.
- **Latest value only.** The index stores one position per driver. There's no need to persist each update on the hot path; if a shard restarts,
  it rebuilds within one update interval as drivers report again. A short replica (or two) per shard avoids even that gap.
- **Staleness is a signal.** Phones lose signal, apps get killed. A driver whose last update is older than ~30 seconds should drop out of results;
  the index evicts or ignores stale entries rather than offering rides to ghosts.
- **History off the hot path.** Updates are also appended to a log (batched, compressed) for trip reconstruction, ETA model training, and
  disputes — consumed asynchronously into cold storage (Chapters 24–25).
- **Adaptive update rates.** Drivers on a trip, or stationary in a quiet area, can report less often; drivers approaching a pickup more often.
  Halving the average rate halves the dominant load.

**Hot cells** — an airport, a stadium after an event, a train station at rush hour — concentrate thousands of drivers and requests in a few
cells. Finer cell resolution in those areas (hierarchical cells make this natural) and dedicated queueing systems (airport driver queues) keep a
single shard from saturating.

---

## 41.7 Deep Dive: Matching and Dispatch

**Distance isn't the metric; time is.** The nearest driver by straight line may be across a river or facing the wrong way on a one-way street.
Matching takes the nearest K candidates from the geo index (cheap), computes **ETA** for each with a routing engine (expensive — so only for a
short list), and ranks by ETA, driver acceptance likelihood, and fairness among drivers.

**Batching improves global outcomes.** Assigning each request instantly to its nearest driver is greedy; collecting requests for a second or two
per area and solving the assignment jointly (minimising total pickup time) produces better matches overall, at the cost of a small delay. Many
systems batch in dense areas and match immediately in sparse ones.

**Never double-dispatch.** Two matchers working on two riders can both pick the same driver. The driver's state must change atomically from
`available` to `offered` before the offer is sent — a conditional update (Chapter 15) that only one matcher can win:

```js
// Dispatch must never give one driver two rides: claim the driver with an atomic state transition
const drivers = new Map([['d1', { state: 'available', version: 7 }]]);
function claim(driverId, rideId, expectedVersion) {      // in SQL: UPDATE … WHERE id = $1 AND state = 'available' AND version = $2
  const d = drivers.get(driverId);
  if (d.state !== 'available' || d.version !== expectedVersion) return false;
  drivers.set(driverId, { state: 'offered', rideId, version: d.version + 1 });
  return true;
}
// Two matchers saw d1 as the best driver for two different riders at the same moment
console.log(claim('d1', 'ride-A', 7), claim('d1', 'ride-B', 7), drivers.get('d1'));
// true false { state: 'offered', rideId: 'ride-A', version: 8 }
```

The losing matcher moves on to its next candidate. The offer has a **timeout** (say 15 seconds): if the driver declines or doesn't respond, the
state returns to `available` (again atomically, conditioned on still being `offered` for that ride) and the ride is offered to the next candidate.

**The trip is a state machine** stored durably: `requested → matched → arriving → in_progress → completed`, with `cancelled` reachable from early
states. Every transition is a conditional update on the trip's current state and version, so retries and races (a rider cancelling as a driver
accepts) resolve deterministically — one transition wins, the other is rejected with the current state. Events for each transition go through an
outbox (Chapter 23) to notifications, payments, and analytics.

---

## 41.8 Failure Modes, Bottlenecks, and 10×

| Failure | Effect | Mitigation |
|---|---|---|
| Geo index shard restarts | City briefly shows no drivers | Replica per shard; repopulates within one update interval |
| Gateway node dies | Drivers disconnect | Reconnect with jitter (Chapter 9); positions resume; offers in flight time out and re-dispatch |
| Routing/ETA service slow | Matching delays | Timeout and fall back to straight-line ranking; cache ETAs per cell pair |
| Trips database failover | New rides briefly fail | Short in-region failover; rider app retries with idempotency key |
| Phone clock or GPS errors | Teleporting or stale drivers | Server-side timestamps; sanity checks on speed and jumps (Chapter 19) |

**Pricing and surge** (out of scope, but asked about) is a stream-processing problem (Chapter 25): count requests and available drivers per cell over
short windows, compute a supply/demand ratio, and publish multipliers per cell — consistent for a rider's session so the price shown is the price
charged.

**At 10×** — 50 million drivers or far more frequent updates: location ingest grows to 12+ million updates per second, which the geographic sharding
absorbs by adding shards per city; adaptive update rates become essential; and matching moves further toward batched optimisation per area. The
system is naturally **cellular** — each city or region is an independent cell with its own shards, matchers, and even trip database — so growth adds
cells rather than enlarging a global bottleneck, and an incident in one city doesn't touch another.

---

## 41.9 Common Mistakes and Best Practices

**Mistake: storing every location update as a database row.** Millions of overwrites per second overwhelm any disk-based store; keep latest positions in
memory and history in a log.

**Mistake: searching only the rider's own cell.** The nearest driver is often just across a cell boundary; search neighbours and expand rings.

**Mistake: ranking by straight-line distance.** Rivers, one-way streets, and traffic make ETA the right metric; compute it for a short list.

**Mistake: offering a driver without claiming them atomically.** Two matchers double-book the same driver.

**Mistake: trusting stale positions.** Drivers whose phones went silent keep receiving offers; evict or ignore entries older than a staleness threshold.

**Mistake: a global geo index.** Every query and update crosses the world; partition by geography so everything stays local.

**Mistake: trip state transitions without conditions.** Racing cancel and accept produce inconsistent trips; make every transition a conditional update.

**Best practice: latest-value, in-memory grid with ring expansion.** O(1) updates and fast nearest-neighbour queries for constantly moving points.

**Best practice: cheap candidate generation, expensive ranking on a short list.** Geo index for K candidates, routing ETA for those only.

**Best practice: cells by city.** Independent geographic cells localise load, failures, and growth.

---

## 41.10 Real-World Example: A Proximity Index for Moving Drivers

This is the in-memory index behind "nearest available drivers": geohash cells, one latest position per driver, O(1) updates that move drivers between
cells, eviction of drivers whose updates stopped, and a query that expands ring by ring around the rider's cell until it has enough candidates closer
than the next ring could contain.

```js
// proximity-index.mjs — "nearest available drivers" over constantly moving points: geohash cells,
// latest-position-only storage, staleness eviction, and ring-by-ring expansion until k are found.

const BASE32 = '0123456789bcdefghjkmnpqrstuvwxyz';
export function geohash(lat, lng, precision = 6) {
  let latR = [-90, 90], lngR = [-180, 180], bits = 0, bit = 0, even = true, out = '';
  while (out.length < precision) {
    const r = even ? lngR : latR, v = even ? lng : lat, mid = (r[0] + r[1]) / 2;
    if (v >= mid) { bits = (bits << 1) | 1; r[0] = mid; } else { bits <<= 1; r[1] = mid; }
    even = !even;
    if (++bit === 5) { out += BASE32[bits]; bits = 0; bit = 0; }      // 5 bits per character
  }
  return out;
}
// Cell size at a precision: longitude and latitude bits alternate, starting with longitude
const cellDeg = (p) => ({ lat: 180 / 2 ** Math.floor((5 * p) / 2), lng: 360 / 2 ** Math.ceil((5 * p) / 2) });

export const haversineKm = (a, b) => {
  const R = 6371, rad = (d) => (d * Math.PI) / 180;
  const dLat = rad(b.lat - a.lat), dLng = rad(b.lng - a.lng);
  const h = Math.sin(dLat / 2) ** 2 + Math.cos(rad(a.lat)) * Math.cos(rad(b.lat)) * Math.sin(dLng / 2) ** 2;
  return 2 * R * Math.asin(Math.sqrt(h));
};

export class ProximityIndex {
  #cells = new Map();    // geohash → Set(driverId)
  #drivers = new Map();  // driverId → { lat, lng, cell, at, available }
  constructor({ precision = 6, staleMs = 30_000, now = Date.now } = {}) { Object.assign(this, { precision, staleMs, now }); }

  update(id, lat, lng, { available = true } = {}) {     // called ~every 4 s per driver
    const cell = geohash(lat, lng, this.precision), prev = this.#drivers.get(id);
    if (prev && prev.cell !== cell) this.#cells.get(prev.cell)?.delete(id);   // moved to another cell
    if (!this.#cells.has(cell)) this.#cells.set(cell, new Set());
    this.#cells.get(cell).add(id);
    this.#drivers.set(id, { lat, lng, cell, at: this.now(), available });
  }

  nearest(lat, lng, { k = 3, maxRings = 4 } = {}) {
    const { lat: dLat, lng: dLng } = cellDeg(this.precision);
    const found = new Map();
    for (let ring = 0; ring <= maxRings; ring++) {
      for (let i = -ring; i <= ring; i++) for (let j = -ring; j <= ring; j++) {
        if (Math.max(Math.abs(i), Math.abs(j)) !== ring) continue;        // only the new ring's cells
        for (const id of this.#cells.get(geohash(lat + i * dLat, lng + j * dLng, this.precision)) ?? []) {
          const d = this.#drivers.get(id);
          if (this.now() - d.at > this.staleMs) { this.#cells.get(d.cell).delete(id); this.#drivers.delete(id); continue; } // phone went silent
          if (d.available) found.set(id, haversineKm({ lat, lng }, d));
        }
      }
      // Cells beyond this ring are at least `ring` cell-widths away; stop once k are found within that
      const ringKm = ring * Math.min(dLat * 111, dLng * 111 * Math.cos((lat * Math.PI) / 180));
      const closeEnough = [...found.values()].filter((km) => km <= ringKm).length;
      if (closeEnough >= k) break;
    }
    return [...found].sort((a, b) => a[1] - b[1]).slice(0, k).map(([id, km]) => `${id} ${km.toFixed(2)} km`);
  }
}

// ── Demo: drivers around central Lisbon ──────────────────────────────────────────────
let t = 0;
const idx = new ProximityIndex({ now: () => t });
const rider = { lat: 38.7139, lng: -9.1394 };
idx.update('d1', 38.7150, -9.1400); idx.update('d2', 38.7300, -9.1500); idx.update('d3', 38.7000, -9.1300);
idx.update('d4', 38.7140, -9.1390, { available: false });              // on a trip
idx.update('d5', 38.8000, -9.2000);                                      // far away
console.log(geohash(rider.lat, rider.lng), idx.nearest(rider.lat, rider.lng));
// eycs0p [ 'd1 0.13 km', 'd3 1.75 km', 'd2 2.01 km' ]
t += 40_000; idx.update('d2', 38.7135, -9.1398);                         // d2 drove over; others went silent
console.log(idx.nearest(rider.lat, rider.lng));
// [ 'd2 0.06 km' ]
```

The driver on a trip is skipped, the far-away driver is never examined, and forty seconds later the drivers that stopped reporting have been evicted
while the one that kept reporting — and moved closer — is found in the rider's own cell.

The decision to defend is **a fixed grid of cells instead of a density-adaptive structure like a quadtree**, given that quadtrees use small cells downtown
and large ones in the suburbs, which looks strictly better for proximity search. The deciding factor is the update rate. Every location update in a
quadtree can move a point across node boundaries and trigger splits or merges, and with 1.25 million updates per second concentrated in the densest
areas, the tree is rebalancing constantly exactly where it's busiest. A fixed grid makes every update O(1) — at most a delete from one set and an insert
into another — and pushes the cost of uneven density onto queries, which handle it by expanding a few rings in sparse areas. Queries are far fewer than
updates here, so moving work from updates to queries is the right trade; where density varies extremely, hierarchical cells (S2, H3) let the grid use
a finer resolution in hot areas without giving up O(1) updates.

---

## 41.11 Questions

**1.** MCQ — What dominates the load of a ride-hailing system? a) Ride requests b) Payments c) Driver location updates d) Map tile downloads
<details><summary>Answer</summary>**c)** Millions of drivers updating every few seconds produce over a million writes per second.</details>

**2.** MCQ — Why store only each driver's latest position in the hot index? a) Privacy b) Queries need only current positions, and the index rebuilds within one update interval if lost c) Disk is expensive d) GPS is inaccurate
<details><summary>Answer</summary>**b)** History goes to a log off the hot path.</details>

**3.** MCQ — A rider is near a cell boundary. Why search neighbouring cells? a) To find more drivers b) The closest driver may be just across the boundary, in another cell c) Cells overlap d) Geohash requires it
<details><summary>Answer</summary>**b)** Cell membership doesn't equal proximity at the edges.</details>

**4.** MCQ — What prevents two matchers from assigning the same driver to two rides? a) Faster matching b) An atomic conditional update of the driver's state from available to offered c) Ranking by ETA d) Geohash precision
<details><summary>Answer</summary>**b)** Only one conditional update can succeed.</details>

**5.** MCQ — Why rank candidates by ETA rather than distance? a) It's cheaper b) Roads, rivers, one-way streets, and traffic make straight-line distance a poor predictor of pickup time c) GPS is more accurate for ETA d) Distance can't be computed
<details><summary>Answer</summary>**b)** ETA is computed only for a short list from the cheap geo query.</details>

**6.** MCQ — Why partition the system by city or region? a) Legal reasons only b) Queries and updates are local, so geographic shards keep work local and isolate failures c) To reduce storage d) Maps require it
<details><summary>Answer</summary>**b)** A Lisbon request never needs Tokyo drivers.</details>

**7.** Short — Compare geohash and quadtrees for moving drivers.
<details><summary>Answer</summary>Geohash is a fixed grid with O(1) updates and simple keys but needs neighbour searches at boundaries and has uneven cell sizes by latitude. Quadtrees adapt to density but must rebalance as points move, which is costly at very high update rates.</details>

**8.** Short — Why should stale driver entries be evicted?
<details><summary>Answer</summary>A driver whose app stopped reporting (no signal, killed app) shouldn't receive offers; ignoring or evicting entries older than a threshold keeps results to drivers who are really there.</details>

**9.** Short — What does batching ride requests buy, and what does it cost?
<details><summary>Answer</summary>Solving assignments jointly over a short window reduces total pickup time compared with greedy nearest-driver matching, at the cost of a small delay before each request is matched.</details>

**10.** Short — Why model the trip as a state machine with conditional transitions?
<details><summary>Answer</summary>So races (rider cancels as driver accepts) and retries resolve deterministically: each transition is applied only from the expected state, one wins, and the trip never ends up in an impossible combination.</details>

**11.** Short — How would you reduce the location-update load without hurting matching?
<details><summary>Answer</summary>Adaptive update rates: less frequent updates for drivers who are stationary, on a trip far from completion, or in quiet areas; more frequent when approaching pickups. Since updates dominate load, halving the average rate halves the dominant cost.</details>

**12.** Output — Using §41.2's model with 2 M online drivers updating every 5 s, how many updates per second must the index absorb?
<details><summary>Answer</summary>**400,000/s.** 2,000,000 / 5.</details>

**13.** Output — Using §41.10's `cellDeg`, what are the cell dimensions in degrees at geohash precision 5?
<details><summary>Answer</summary>**About 0.044° latitude × 0.044° longitude.** 25 bits: 12 for latitude (180 / 4,096 ≈ 0.0439°) and 13 for longitude (360 / 8,192 ≈ 0.0439°) — roughly 4.9 km × 3.8 km at Lisbon's latitude.</details>

**14.** Output — Using §41.7's `claim`, after `claim('d1', 'ride-A', 7)` succeeds, what does `claim('d1', 'ride-A', 8)` return, and why might a matcher call it?
<details><summary>Answer</summary>**`false`.** The driver's state is now `offered`, not `available`, so even the correct new version doesn't allow a second claim — a retried or duplicated claim can't double-book.</details>

**15.** Output — A query expands rings of geohash-6 cells (each roughly 1.2 km × 0.6 km) up to 4 rings around the rider's cell. How many cells does it examine in total?
<details><summary>Answer</summary>**81.** A (2 × 4 + 1)² = 9 × 9 square of cells, including the centre.</details>

**16.** Debug — Riders near the edges of some neighbourhoods are matched with distant drivers while closer ones sit idle. The index searches only the rider's own geohash cell. Cause and fix?
<details><summary>Answer</summary>Boundary effects: close drivers sit in adjacent cells. Search the eight neighbours (and expand rings), and stop only when found candidates are closer than the next ring could hold.</details>

**17.** Debug — Drivers occasionally receive two ride offers at once, and some riders are told "driver found" twice. What's missing?
<details><summary>Answer</summary>An atomic claim of the driver's state before sending an offer, and a conditional trip state transition on acceptance. Implement the claim as a conditional update (`state = 'available'` and version match) and route ride-state changes through conditional transitions.</details>

**18.** Debug — After a geo index shard restart, its city showed zero available drivers for about two minutes. Updates arrive every 4 seconds. Why so long, and how do you fix it?
<details><summary>Answer</summary>The shard probably waited for full warm-up or rejected updates while restarting, or clients backed off after the disconnect. Accept updates immediately on restart (the index rebuilds within one interval), keep a replica to serve during restart, and make drivers' reconnect backoff short with jitter.</details>

**19.** Debug — The matching service's p99 is 8 seconds during peaks; traces show it computing routed ETAs for 200 candidate drivers per request. What do you change?
<details><summary>Answer</summary>Limit routed ETA to a short list (say the 10 nearest by straight line from the geo index), cache ETAs between cell pairs, and time out to straight-line ranking if routing is slow.</details>

**20.** Code — Write `neighbours(lat, lng, precision)` returning the geohashes of the 8 cells surrounding the cell containing a point, using `geohash` and `cellDeg` from §41.10.
<details><summary>Answer</summary>

```js
function neighbours(lat, lng, precision = 6) {
  const { lat: dLat, lng: dLng } = cellDeg(precision);
  const out = [];
  for (const i of [-1, 0, 1]) for (const j of [-1, 0, 1]) {
    if (i === 0 && j === 0) continue;
    out.push(geohash(lat + i * dLat, lng + j * dLng, precision)); // step one cell width in each direction
  }
  return out;
}
```
Libraries compute neighbours directly from the hash string, which also handles the poles and the antimeridian.</details>

**21.** Code — Write `transition(trip, to, expectedVersion)` enforcing allowed trip state transitions with optimistic versioning.
<details><summary>Answer</summary>

```js
const ALLOWED = { requested: ['matched', 'cancelled'], matched: ['arriving', 'cancelled'], arriving: ['in_progress', 'cancelled'], in_progress: ['completed'] };
function transition(trip, to, expectedVersion) {
  if (trip.version !== expectedVersion) return { ok: false, reason: 'stale version', trip };
  if (!(ALLOWED[trip.state] ?? []).includes(to)) return { ok: false, reason: `${trip.state} → ${to} not allowed`, trip };
  return { ok: true, trip: { ...trip, state: to, version: trip.version + 1 } };
}
const t0 = { state: 'arriving', version: 3 };
console.log(transition(t0, 'in_progress', 3).trip.state, transition(t0, 'completed', 3).reason); // in_progress arriving → completed not allowed
```
In the database, the version check and the allowed-state check go into one conditional `UPDATE`.</details>

**22.** Code — Write `isPlausibleUpdate(prev, next, { maxSpeedKmh })` rejecting location updates that imply impossible speeds (GPS jumps).
<details><summary>Answer</summary>

```js
const haversineKm = (a, b) => {                                   // as in §41.10
  const R = 6371, rad = (d) => (d * Math.PI) / 180, dLat = rad(b.lat - a.lat), dLng = rad(b.lng - a.lng);
  const h = Math.sin(dLat / 2) ** 2 + Math.cos(rad(a.lat)) * Math.cos(rad(b.lat)) * Math.sin(dLng / 2) ** 2;
  return 2 * R * Math.asin(Math.sqrt(h));
};
function isPlausibleUpdate(prev, next, { maxSpeedKmh = 200 } = {}) {
  if (!prev) return true;
  const hours = (next.ts - prev.ts) / 3_600_000;
  if (hours <= 0) return false;                                  // out-of-order or duplicate timestamp
  return haversineKm(prev, next) / hours <= maxSpeedKmh;
}
console.log(isPlausibleUpdate({ lat: 38.71, lng: -9.14, ts: 0 }, { lat: 38.75, lng: -9.14, ts: 4_000 })); // false — ~4.4 km in 4 s
```
Rejected updates are logged, and the previous position is kept.</details>

**23.** Scenario — A city wants all drivers at the airport to queue first-in, first-out for airport pickups. How does this fit the design?
<details><summary>Answer</summary>Define the airport as a geofenced zone with its own dispatch policy: drivers entering the zone join a durable FIFO queue (ordered by entry time, with rules for leaving and re-entering), and airport ride requests are matched from the queue head instead of by proximity. The geo index still tracks positions (to detect zone exit), but matching for that zone bypasses nearest-driver ranking. Queue state needs the same atomic claims as normal dispatch.</details>

**24.** Scenario — The product team wants shared rides: one car picking up several riders going in similar directions. What changes in matching?
<details><summary>Answer</summary>Matching becomes a routing optimisation: candidate trips must be evaluated for detour cost, pickup-time constraints, and capacity, often over batched windows, and a new request may join an in-progress trip. Driver and trip state machines gain multi-rider states; claims must reserve seats atomically on a trip; and ETAs must account for intermediate stops. It's a different, harder optimiser on the same location and dispatch infrastructure.</details>

---

## Chapter Summary

- Ride-hailing load is dominated by location updates — over a million per second — while the current state they produce is tiny, so the hot path is
  an in-memory, latest-value index.
- Index moving points with a fixed grid of cells (geohash, S2, H3) for O(1) updates, search neighbouring cells, and expand ring by ring until candidates
  are provably closest.
- Shard everything by geography, evict stale drivers, push history to a log, and adapt update rates to reduce the dominant cost.
- Generate a short list of candidates cheaply and rank by routed ETA; batch matches in dense areas for better global outcomes.
- Claim drivers with atomic conditional updates and model trips as durable state machines with conditional transitions, so races and retries never
  double-book or corrupt a trip.
- Treat cities as independent cells for load, failure isolation, and growth; at 10× add shards per city and lean harder on adaptive updates and batched
  optimisation.

---

## Practice Exercises

1. Replace geohash in §41.10 with H3 or S2 (via a library) and compare neighbour handling and cell-size uniformity at different latitudes.
2. Benchmark §41.10's index with 100,000 simulated drivers moving randomly and 10,000 queries per second, and measure update and query latency.
3. Read a public engineering post about a ride-hailing or delivery company's dispatch system, and map its components to this chapter's design.
4. Implement batched matching for a small simulation: collect requests for 2 seconds, solve assignment greedily and with a simple optimal method, and
   compare total pickup time.
5. Build the trip state machine in PostgreSQL with conditional transitions and test races between cancel and accept with concurrent clients.

---

## Mini Project: A City Dispatch Simulator

Build a simulator of one city's ride-hailing backend with realistic movement and demand, and measure how design choices affect rider experience.

- Simulated drivers moving along a grid of streets, reporting positions every few seconds, with random disconnects.
- The §41.10 proximity index (or an S2/H3 version) with staleness eviction, sharded into two halves of the city.
- Matching with K-nearest candidates, a simple ETA model, atomic driver claims, offers with timeouts, and trip state machines.
- Metrics: time to match, pickup time, driver utilisation, double-dispatch count (must be zero), and match quality with and without batching.
- Experiments: vary update interval, cell size, and batching window, and report the trade-offs.

---

**Next:** [Chapter 42 — Design Search Autocomplete →](42-search-autocomplete.md)
