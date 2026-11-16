# Chapter 5 — Networking for System Designers

## Learning Objectives

- Count the round trips a request costs, and explain why round trips matter more than bandwidth
- Explain what TCP slow start, TLS, and QUIC handshakes cost, and how connection reuse removes it
- Choose between HTTP/1.1, HTTP/2, and HTTP/3 for a given client and network
- Configure connection pools and keep-alive timeouts that neither churn nor reset
- Diagnose port exhaustion, keep-alive races, and long-lived-connection load imbalance from symptoms

---

## 5.1 The Round Trip Is the Unit of Cost

Two numbers describe a network path. **Bandwidth** is how many bytes per second it can carry.
**Round-trip time (RTT)** is how long a message takes to go there and back. For almost every
request a user-facing system makes, RTT dominates: a 2 KB API response takes microseconds to
serialise onto a 10 Gbit/s link, and 80 ms to cross the Atlantic and back.

That makes the round trip the natural unit for reasoning about network cost. A design that
needs four sequential round trips to a server 80 ms away has spent 320 ms before doing any work,
no matter how fast the server is. The levers that matter are therefore:

| Lever | Example | Chapter |
|---|---|---|
| Fewer round trips per request | Reuse connections; batch calls; avoid sequential dependencies | this chapter |
| Shorter round trips | Serve from a nearby edge or region | 6, 32 |
| Parallel instead of serial | Issue independent calls concurrently | 3 |
| Fewer bytes when bandwidth *does* bind | Compression, compact encodings, smaller images | this chapter, 6 |

Bandwidth starts to matter for large transfers on long paths, through the **bandwidth-delay
product**: the amount of data that must be "in flight" to keep a link busy. A sender can only have
one window's worth of unacknowledged data outstanding, so throughput on a single connection is
capped at `window / RTT`:

```js
// Bytes that must be in flight to fill a path: bandwidth × RTT
const bdp = (gbps, rttMs) => (gbps * 1e9 / 8) * (rttMs / 1000);
console.log(`${(bdp(10, 80) / 1e6).toFixed(0)} MB in flight to fill 10 Gbit/s at 80 ms`);
// 100 MB in flight to fill 10 Gbit/s at 80 ms

// The ceiling a fixed window imposes, regardless of link speed
const maxMbps = (windowBytes, rttMs) => (windowBytes * 8) / (rttMs / 1000) / 1e6;
console.log(`${maxMbps(64 * 1024, 80).toFixed(1)} Mbit/s with a 64 KB window at 80 ms`);
// 6.6 Mbit/s with a 64 KB window at 80 ms
```

A single TCP connection with a small window across an ocean is slow even on a fast link — which
is why bulk cross-region transfers use window scaling, multiple parallel connections, or both.

---

## 5.2 TCP: What a Connection Costs

TCP gives you a reliable, ordered byte stream over an unreliable packet network. That reliability
costs round trips at the start, a ramp-up period, and some state at the end.

**The handshake.** Before any data flows, the client sends `SYN`, the server replies `SYN-ACK`,
and the client's `ACK` can carry the first request. That's **one RTT** of pure setup per new
connection.

**Slow start.** A new connection doesn't know how much the network can carry, so it starts with a
small **congestion window** — the modern default is 10 segments, about 14.6 KB — and roughly
doubles it every round trip until it sees loss. A response larger than the initial window needs
several round trips to deliver, even on an idle gigabit link:

```js
// Round trips TCP slow start needs to deliver a response, ignoring loss
function slowStartRounds(bytes, { mss = 1460, initCwnd = 10 } = {}) {
  let cwnd = initCwnd, sent = 0, rounds = 0;
  while (sent < bytes) { sent += cwnd * mss; cwnd *= 2; rounds++; } // the window doubles per RTT
  return rounds;
}
for (const kb of [10, 100, 1000]) {
  const n = slowStartRounds(kb * 1000);
  console.log(`${kb} KB → ${n} round trip${n === 1 ? '' : 's'}`);
}
// 10 KB → 1 round trip
// 100 KB → 3 round trips
// 1000 KB → 7 round trips
```

This is why the first ~14 KB of a web page are precious (they arrive in the first round trip),
and why a *warm* connection — one that has already grown its window — delivers large responses
much faster than a fresh one. Reusing connections avoids both the handshake and slow start.

**Congestion control.** When packets are lost, TCP shrinks its window, so a lossy path (mobile,
congested Wi-Fi) has lower throughput than its bandwidth suggests. Modern algorithms like BBR
estimate bandwidth and RTT directly rather than reacting only to loss, and do markedly better on
long or lossy paths. It's a server-side setting worth knowing exists.

**Head-of-line blocking.** TCP delivers bytes in order. If one packet is lost, everything behind
it waits for the retransmission — even bytes belonging to unrelated requests multiplexed onto the
same connection. That matters for HTTP/2 (§5.4).

**Connection teardown and TIME_WAIT.** The side that closes a TCP connection first keeps a record
of it in the `TIME_WAIT` state for a while (60 seconds on Linux) so that stray late packets from
the old connection can't be mistaken for a new one. Each such record pins a local port. Section
5.5 shows how this exhausts ports when connections aren't reused.

> 💼 **Interview** — "Why is the first request to a service slower than the rest?" The answer
> chain is DNS lookup, TCP handshake, TLS handshake, then slow start for the response — and,
> inside the server, cold caches and JIT. Connection reuse removes the middle three.

---

## 5.3 TLS: Encryption's Round Trips

Every external connection — and, in a zero-trust network (Chapter 33), every internal one — is
encrypted with TLS. The handshake that agrees on keys costs round trips on top of TCP's:

| Protocol | Setup round trips before the request | Notes |
|---|---|---|
| TCP + TLS 1.2 | 3 (1 TCP + 2 TLS) | Still found on old clients and internal systems |
| TCP + TLS 1.3 | 2 (1 TCP + 1 TLS) | The default for modern clients |
| TCP + TLS 1.3 with session resumption | 2, with cheaper crypto | Resumption saves CPU, not round trips, without 0-RTT |
| QUIC (HTTP/3) | 1 | Transport and crypto handshakes combined |
| QUIC or TLS 1.3 with 0-RTT | 0 | Request sent in the first flight — replayable (see below) |

```js
// Time to first response byte for a user 80 ms away, by protocol and connection state
const setupRtts = {
  'reused connection': 0,
  'new TCP + TLS 1.3': 2,
  'new TCP + TLS 1.2': 3,
  'new QUIC (HTTP/3)': 1,
  'QUIC 0-RTT resumption': 0,
};
for (const [k, setup] of Object.entries(setupRtts)) {
  console.log(`${k.padEnd(22)} ${(setup + 1) * 80} ms`); // + 1 RTT for the request itself
}
// reused connection      80 ms
// new TCP + TLS 1.3      240 ms
// new TCP + TLS 1.2      320 ms
// new QUIC (HTTP/3)      160 ms
// QUIC 0-RTT resumption  80 ms
```

A fresh TLS 1.2 connection is four times slower to first byte than a reused one. For users far
from your servers, this is the strongest argument for terminating TLS at a nearby edge (Chapter 6):
the expensive handshake happens over a 10 ms path to the edge, and the edge keeps warm, long-lived
connections back to the origin.

> ⚠️ **Gotcha** — **0-RTT data can be replayed.** An attacker who captures the first flight can
> resend it, and the server can't distinguish the copy from the original. Servers must accept
> 0-RTT only for idempotent requests (a `GET` for a page, never a `POST /transfer`). CDNs and
> load balancers that enable 0-RTT typically restrict it to safe methods or mark such requests
> so the origin can refuse them.

**Where TLS terminates is a design decision**:

| Termination point | Benefit | Cost |
|---|---|---|
| Edge / CDN | Shortest handshake path for users | The edge sees plaintext; re-encrypt to origin |
| Load balancer | Central certificate management; L7 routing possible | Internal hop is plaintext unless re-encrypted |
| Each service (end-to-end, or mTLS in a mesh) | Encryption everywhere, service identity | Certificate distribution and rotation per service |

Most production systems terminate at the edge or load balancer and re-encrypt to backends, which
is the default this book assumes.

---

## 5.4 HTTP/1.1, HTTP/2, and HTTP/3

All three carry the same semantics — methods, headers, status codes — and differ in how requests
share connections.

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Transport | TCP | TCP | QUIC (over UDP) |
| Requests per connection, concurrently | One at a time (pipelining is effectively unused) | Many, multiplexed as streams | Many, multiplexed as independent streams |
| Header encoding | Plain text, repeated every request | Compressed (HPACK) | Compressed (QPACK) |
| Head-of-line blocking | At the HTTP layer: a slow response blocks the connection | At the TCP layer: one lost packet stalls all streams | None across streams |
| Connection setup | TCP + TLS | TCP + TLS | 1 RTT, or 0-RTT on resumption |
| Survives client IP change (Wi-Fi → cellular) | No | No | Yes — connection IDs, not IP/port tuples |
| Typical use | Simple clients, internal tools, legacy | Browsers, gRPC, service-to-service | Browsers and mobile apps on lossy networks |

**HTTP/1.1's** limitation is one in-flight request per connection, so clients open several
connections per host (browsers use about six) and still queue behind slow responses.

**HTTP/2** multiplexes many concurrent requests over one connection — a large win for browsers
loading dozens of assets, and the basis of gRPC. Its weakness is inherited from TCP: a single lost
packet stalls every stream on the connection until it's retransmitted. On a clean data-centre
network that's rare; on a 2% loss mobile network it can make HTTP/2 slower than several HTTP/1.1
connections.

**HTTP/3** runs over QUIC, a transport built on UDP that implements reliability *per stream*, so
loss on one stream doesn't stall others. It also combines the transport and TLS handshakes and
lets connections survive a change of client IP address. Its benefits are largest for mobile users
on lossy networks; inside a data centre, where loss is near zero and connections are long-lived,
HTTP/2 is usually just as good and better supported by internal tooling.

The practical design rule: **offer HTTP/2 and HTTP/3 at the edge for users, use HTTP/2 (or gRPC)
between services, and keep HTTP/1.1 working for clients that need it.** CDNs and modern load
balancers make the edge part a configuration switch.

> ⚠️ **Gotcha** — HTTP/2 and gRPC connections are long-lived and carry many requests. A layer-4
> load balancer balances *connections*, not requests, so a client that opens one connection to a
> service with ten backends sends all its traffic to one backend — forever. New backends added by
> autoscaling receive nothing until clients reconnect. Use a layer-7 (request-aware) load balancer
> or client-side balancing for HTTP/2, and set a maximum connection age on servers so clients
> periodically reconnect and rebalance (Chapter 7).

---

## 5.5 Connection Pools and Their Failure Modes

Every service-to-service call should reuse connections from a **pool**, so the handshake and slow
start are paid once per connection instead of once per request. Pools have three settings that
matter, and each has a characteristic way of going wrong.

**Maximum connections per upstream.** Bounds how much concurrency one client can put on one
upstream. Little's law (Chapter 2) gives the floor: requests/s × latency. Unbounded pools let a
latency spike in the upstream turn into thousands of connections that overload it further.

**Maximum idle connections.** How many connections stay open between bursts. If it's smaller than
the normal working concurrency, every burst closes some connections and opens new ones — paying
handshakes and slow start repeatedly and generating `TIME_WAIT` entries. The real-world example in
§5.9 measures this: the same traffic opens 10, 28, or 200 connections depending only on these two
settings.

**Idle timeout.** How long an idle pooled connection is kept before the client closes it. This has
a subtle constraint:

> ⚠️ **Gotcha** — **The client's idle timeout must be shorter than the server's keep-alive
> timeout.** If the server (or a load balancer in between) closes an idle connection at 5 seconds
> and the client keeps it in its pool for 30, the client will eventually send a request on a
> connection the server has just closed. The request fails with `ECONNRESET` or "socket hang up" —
> sporadically, under light load, and almost never in tests. Node's HTTP server defaults to a
> 5-second keep-alive timeout; many load balancers default to 60 seconds idle. Chain the timeouts
> so each hop closes *after* the hop in front of it: client < load balancer < backend server.

### Ephemeral port exhaustion

A TCP connection is identified by the four-tuple (source IP, source port, destination IP,
destination port). When a client connects to the same destination repeatedly, only the source port
varies, and Linux allocates those from an ephemeral range of about 28,000 ports by default. Each
connection the client closes holds its port in `TIME_WAIT` for 60 seconds.

```text
28,000 ports ÷ 60 s in TIME_WAIT ≈ 470 new connections/s per destination, sustained
```

A service that opens a fresh connection per request to one upstream at 1,000 requests/s runs out
of ports within a minute and starts failing with `EADDRNOTAVAIL` — "cannot assign requested
address" — while CPU and memory look fine. The same limit appears, often lower, on **NAT
gateways**: everything leaving a private network through one NAT shares its pool of ports per
destination, so many instances each opening "only" a few hundred connections per second to the
same third-party API can exhaust it together.

The fix is almost always connection reuse. Widening the port range and enabling `tcp_tw_reuse`
are mitigations; a pool with keep-alive removes the problem.

### Connection counts at the server

The other side of the pool is the server's total connection count: every client instance × its
pool size. Fifty client instances with pools of 100 connections each can open 5,000 connections to
a database that performs best with a few dozen active sessions (Chapter 2's Little's-law example).
The usual fix is a **connection pooler or proxy** in front of the database (PgBouncer for
PostgreSQL, RDS Proxy, ProxySQL) that multiplexes many client connections onto a small number of
server connections.

---

## 5.6 Payloads, Serialisation, and Chattiness

Once round trips are under control, bytes and encoding are the next costs.

**Serialisation format.** JSON is human-readable, universally supported, and verbose; binary
formats are compact and fast but need schemas and tooling.

| | JSON | Protocol Buffers / similar binary |
|---|---|---|
| Size | Field names repeated in every message | Field numbers; typically 3–10× smaller before compression |
| Parse cost | Text parsing; significant CPU at high volume | Much faster |
| Schema | Optional (JSON Schema, OpenAPI) | Required; enables generated clients |
| Evolution | Add fields freely; consumers must tolerate unknown ones | Add fields with new numbers; never reuse a number |
| Debuggability | Read it with `curl` | Needs tooling |
| Good default for | Public APIs, browsers, low-to-moderate volume | High-volume internal RPC, streaming, mobile on bad networks |

**Compression** narrows the size gap: gzip or Brotli on JSON typically cuts 70–90% of bytes. It
costs CPU on both ends, so compress responses above a threshold (~1 KB) and leave small ones raw.

**Chattiness** costs far more than encoding. A client that makes one call to list 50 orders and
then one call per order for its details has made 51 round trips — the network version of the N+1
query problem. Three fixes, in order of preference:

1. **Return what the caller needs in one response** (include order details in the list, or let the
   caller request fields).
2. **Batch endpoints** — `GET /orders?ids=1,2,3…` or a batch RPC — so N lookups become one call.
3. **Parallelise** the N calls when they can't be batched, which cuts wall-clock time to roughly
   one round trip but still costs N requests of server work.

Chapter 8 returns to this in API design, and GraphQL's DataLoader pattern exists entirely to fix
it.

---

## 5.7 The Network Inside a Region

Inside a cloud region, a few properties of the network shape designs:

- **Availability zones** are separate data centres a few kilometres apart. Round trips between
  them are ~0.5–2 ms, versus ~0.1–0.5 ms within one. A synchronous chain that crosses zones on
  every hop adds up quickly.
- **Cross-zone traffic is usually billed per GB**, in both directions, at around a cent per GB on
  major providers. A chatty service that sends 50 TB a month between zones pays for it. Zone-aware
  routing — prefer a backend in the caller's zone when healthy — cuts both latency and cost.
- **Egress to the internet is billed per GB** and is often a top-three cost line for media-heavy
  products (Chapter 2's bandwidth estimate).
- **Private networks** (VPCs) put most services on private addresses. Outbound internet traffic
  goes through NAT gateways — a shared, port-limited, per-GB-billed resource (§5.5) — unless
  routed via private endpoints for cloud services.
- **DNS inside the region** resolves service names to instances or load balancers; Chapter 6
  covers DNS caching behaviour, which matters for failover.

---

## 5.8 Common Mistakes and Best Practices

**Mistake: a new connection per request.** Each one pays the TCP and TLS handshakes plus slow
start, and at a few hundred per second to one destination it exhausts ephemeral ports.

**Mistake: client idle timeout longer than the server's.** The client reuses connections the
server already closed, producing sporadic `ECONNRESET` errors under light load.

**Mistake: idle pool smaller than working concurrency.** Every burst closes and reopens
connections, paying handshakes and creating `TIME_WAIT` churn.

**Mistake: layer-4 balancing for HTTP/2 or gRPC.** Long-lived multiplexed connections pin all of a
client's traffic to one backend, and new backends receive nothing.

**Mistake: unbounded connection pools.** A slow upstream causes the pool to grow until the
upstream (often a database) is overwhelmed by connections.

**Mistake: chatty APIs across a high-RTT path.** N sequential calls cost N round trips; at 80 ms
each, ten calls are most of a second.

**Mistake: accepting 0-RTT for non-idempotent requests.** Early data can be replayed by an
attacker, executing a state change twice.

**Best practice: chain keep-alive timeouts outward-in.** Client idle timeout < load balancer idle
timeout < backend keep-alive timeout, so the closing side is always the one that isn't about to
send.

**Best practice: terminate TLS close to users.** The expensive handshake travels a short path, and
the edge holds warm connections to the origin.

**Best practice: count round trips in every design review.** Cold and warm, per user-facing
operation — they're the most common hidden latency.

---

## 5.9 Real-World Example: A Bounded Keep-Alive Upstream Client

Every service needs a correctly configured client for each upstream it calls. This one uses only
`node:http`, caps concurrency, keeps connections warm, applies one deadline to the whole exchange,
and — run as-is — starts a local upstream and counts the TCP connections each configuration
opens.

```js
// upstream-client.mjs — a bounded, keep-alive HTTP client for calling one upstream service.
// Dependency-free (node:http). Run as-is: it starts a local upstream and compares
// connection counts across pool configurations.
import http from 'node:http';

export function createUpstreamClient({
  host, port,
  maxSockets = 16,              // hard cap on concurrent connections to this upstream
  maxFreeSockets = maxSockets,  // idle sockets kept warm; fewer than normal concurrency = churn
  idleTimeoutMs = 30_000,       // must be SHORTER than the server's keep-alive timeout
  requestTimeoutMs = 2_000,
  keepAlive = true,
} = {}) {
  const agent = new http.Agent({ keepAlive, maxSockets, maxFreeSockets, timeout: idleTimeoutMs });
  const stats = { requests: 0, failures: 0 };

  function request(path, { method = 'GET', body, signal } = {}) {
    stats.requests++;
    // One deadline for the whole exchange — waiting for a pooled socket, connecting,
    // and reading the body all count against it, so the caller's budget is honoured
    const deadline = AbortSignal.timeout(requestTimeoutMs);
    const combined = signal ? AbortSignal.any([signal, deadline]) : deadline;

    return new Promise((resolve, reject) => {
      const req = http.request({ host, port, path, method, agent, signal: combined }, (res) => {
        const chunks = [];
        res.on('data', (c) => chunks.push(c));
        res.on('end', () => resolve({ status: res.statusCode, body: Buffer.concat(chunks).toString() }));
        res.on('error', reject);
      });
      req.on('error', (err) => { stats.failures++; reject(err); });
      if (body) req.write(body);
      req.end();
    });
  }

  return {
    request,
    stats: () => ({ ...stats, openSockets: Object.values(agent.sockets).flat().length,
      idleSockets: Object.values(agent.freeSockets).flat().length }),
    close: () => agent.destroy(),
  };
}

// ── Demo: an upstream that counts the TCP connections it accepts ──────────────────────
let connections = 0;
const server = http.createServer((req, res) => setTimeout(() => res.end('ok'), 5));
server.keepAliveTimeout = 60_000;          // the server keeps idle sockets longer than clients do
server.on('connection', () => connections++);
await new Promise((r) => server.listen(0, r));
const { port } = server.address();

const configs = {
  'no keep-alive, unbounded': { keepAlive: false, maxSockets: Infinity },
  'keep-alive, 10 max, 4 idle': { keepAlive: true, maxSockets: 10, maxFreeSockets: 4 },
  'keep-alive, 10 max, 10 idle': { keepAlive: true, maxSockets: 10, maxFreeSockets: 10 },
};
for (const [label, cfg] of Object.entries(configs)) {
  connections = 0;
  const client = createUpstreamClient({ host: '127.0.0.1', port, ...cfg });
  // 200 requests in 4 waves of 50 concurrent calls, as bursty traffic arrives
  for (let wave = 0; wave < 4; wave++) {
    await Promise.all(Array.from({ length: 50 }, () => client.request('/')));
  }
  console.log(`${label.padEnd(28)} → ${connections} connections`);
  client.close();
}
server.close();
// no keep-alive, unbounded     → 200 connections
// keep-alive, 10 max, 4 idle   → 28 connections
// keep-alive, 10 max, 10 idle  → 10 connections
```

The decision most readers would question is **capping `maxSockets` at 10 when 50 requests arrive
at once** — the other 40 wait in the client's local queue instead of going straight to the
upstream. That looks like self-inflicted latency, but the alternative is worse in exactly the
situation that matters. When the upstream slows down, an uncapped client opens more and more
connections (Little's law again), multiplying the upstream's load at the moment it can least
absorb it, and every client instance does the same. The cap turns "upstream is slow" into a
bounded, local queue that the request deadline drains — callers time out fast and predictably
instead of the upstream collapsing. Size the cap from Little's law at expected peak with headroom,
and treat time spent waiting for a socket as a metric worth alerting on: it's the earliest signal
that the upstream is saturated.

---

## 5.10 Questions

**1.** MCQ — A 3 KB API response is sent over a 10 Gbit/s link with an 80 ms RTT on an existing warm connection. What dominates its latency? a) Serialisation onto the link b) Bandwidth c) The round trip d) TCP slow start
<details><summary>Answer</summary>**c)** 3 KB takes microseconds to transmit at 10 Gbit/s and fits in the initial congestion window; the 80 ms round trip is nearly all of it.</details>

**2.** MCQ — How many round trips does a brand-new HTTPS request over TCP with TLS 1.3 take before the first response byte? a) 1 b) 2 c) 3 d) 4
<details><summary>Answer</summary>**c)** One for the TCP handshake, one for the TLS 1.3 handshake, and one for the request and response.</details>

**3.** MCQ — Which problem does HTTP/3 solve that HTTP/2 does not? a) Header compression b) Multiplexing multiple requests on one connection c) Head-of-line blocking across streams when a packet is lost d) Encryption
<details><summary>Answer</summary>**c)** HTTP/2 multiplexes over TCP, so one lost packet stalls all streams. QUIC handles loss per stream. HTTP/2 already has multiplexing and header compression.</details>

**4.** MCQ — Why should 0-RTT early data only be accepted for idempotent requests? a) It's unencrypted b) It can be replayed by an attacker c) It's limited to 1 KB d) It bypasses the load balancer
<details><summary>Answer</summary>**b)** The first flight can be captured and resent; the server can't tell the replay from the original, so a state change could execute twice.</details>

**5.** MCQ — A client sends gRPC traffic through a layer-4 load balancer to 8 backends. After autoscaling adds 4 more backends, they receive almost no traffic. Why? a) gRPC doesn't support load balancing b) Long-lived HTTP/2 connections are balanced once, at connect time c) The new backends failed health checks d) Layer-4 balancers only support UDP
<details><summary>Answer</summary>**b)** L4 balances connections, and a multiplexed HTTP/2 connection carries all the client's requests for its lifetime. Use L7 balancing or client-side balancing, and cap connection age on the server.</details>

**6.** MCQ — Which pairing of idle timeouts avoids sporadic connection resets? a) Client 60 s, server 5 s b) Client 5 s, server 60 s c) Both 30 s d) Client none, server 5 s
<details><summary>Answer</summary>**b)** The client must close idle connections before the server does; otherwise it sends requests on sockets the server has just closed.</details>

**7.** Short — What is the bandwidth-delay product, and why does it matter for cross-region transfers?
<details><summary>Answer</summary>Bandwidth × RTT: the data that must be in flight to keep a path busy. A connection whose window is smaller than the BDP can't use the link's bandwidth, so long paths need window scaling or parallel connections to transfer bulk data quickly.</details>

**8.** Short — Why does a warm connection deliver a 500 KB response faster than a new connection on the same path?
<details><summary>Answer</summary>The new connection pays handshake round trips and starts with a small congestion window that must double over several round trips; the warm connection has skipped the handshake and has already grown its window.</details>

**9.** Short — What is `TIME_WAIT` and how does it lead to port exhaustion?
<details><summary>Answer</summary>The side that closes a TCP connection first keeps its four-tuple reserved (60 s on Linux) so late packets aren't misattributed. Each reservation pins a local ephemeral port; opening and closing connections to one destination faster than ~28,000 per 60 s runs out of ports.</details>

**10.** Short — Why does a connection pooler like PgBouncer help a database with many application instances?
<details><summary>Answer</summary>Instances × pool size can far exceed the number of sessions the database runs efficiently. The pooler multiplexes thousands of client connections onto a few dozen server connections, keeping database concurrency near the Little's-law optimum.</details>

**11.** Short — Give two reasons to terminate TLS at an edge location near users.
<details><summary>Answer</summary>The handshake's round trips cross a short path, cutting time to first byte; and the edge maintains warm, long-lived connections to the origin, so users never pay the long-path handshake or slow start.</details>

**12.** Output — What does this print?
```js
const rtt = 60;
const firstByte = (setupRtts) => (setupRtts + 1) * rtt;
console.log(firstByte(3), firstByte(2), firstByte(0));
```
<details><summary>Answer</summary>`240 180 60` — TLS 1.2 on a new connection, TLS 1.3 on a new connection, and a reused connection, at 60 ms RTT.</details>

**13.** Output — With an initial congestion window of 10 segments of 1,460 bytes that doubles each round trip, how many round trips does a 60 KB response need?
<details><summary>Answer</summary>**3.** Cumulative capacity after 1, 2, 3 RTTs is 14.6 KB, 43.8 KB, 102.2 KB; 60 KB fits within three.</details>

**14.** Output — A single connection has a 256 KB window over a 100 ms RTT path. What is its maximum throughput, in Mbit/s to the nearest whole number?
<details><summary>Answer</summary>**21 Mbit/s.** 262,144 bytes × 8 / 0.1 s ≈ 20.97 Mbit/s, regardless of link speed.</details>

**15.** Output — A client opens a new connection per request to one upstream and closes it afterwards. With 28,000 ephemeral ports and a 60-second `TIME_WAIT`, roughly what sustained request rate exhausts the ports?
<details><summary>Answer</summary>**~470 requests/s.** 28,000 / 60 ≈ 467 new connections per second to that destination.</details>

**16.** Debug — A service calling an internal API sees ~0.1% of requests fail with "socket hang up", mostly at night when traffic is low. Retrying always succeeds. Cause and fix?
<details><summary>Answer</summary>A keep-alive race: the client's pool keeps idle connections longer than the server's keep-alive timeout, so after quiet periods it reuses sockets the server just closed. Set the client's idle timeout below the server's (and the load balancer's), and retry idempotent requests once on this specific error.</details>

**17.** Debug — A worker fleet calling a third-party API starts failing with `EADDRNOTAVAIL` after a traffic increase, though CPU and memory are low. All workers sit behind one NAT gateway. Cause and fix?
<details><summary>Answer</summary>Ephemeral port exhaustion at the NAT: every worker's connections to the same destination share the NAT's port pool, and opening new connections per request burns through it while `TIME_WAIT` holds old ports. Reuse connections with keep-alive pools; if still insufficient, add NAT capacity (more IPs) or spread egress.</details>

**18.** Debug — After switching a mobile app's API from HTTP/1.1 to HTTP/2, median latency improved but p99 got worse for users on cellular networks. Why?
<details><summary>Answer</summary>HTTP/2 multiplexes everything on one TCP connection, so a single lost packet — common on cellular — stalls all in-flight requests until retransmission; HTTP/1.1's several connections isolated losses. Enable HTTP/3 (QUIC), which recovers loss per stream.</details>

**19.** Debug — A page renders in 900 ms for users in Asia and 150 ms in the US. The server-side handler takes 40 ms everywhere. Tracing shows the browser makes 4 sequential API calls after load. What's going on and what do you change?
<details><summary>Answer</summary>Each sequential call costs a trans-Pacific round trip (~180 ms), so four calls dominate. Collapse them into one call (or parallelise them) and serve the API from a region or edge closer to Asian users; terminating TLS at a nearby edge also removes handshake cost.</details>

**20.** Code — Write `roundTrips({ newConnection, tls, http3, zeroRtt })` returning the round trips to the first response byte, following the table in §5.3.
<details><summary>Answer</summary>

```js
function roundTrips({ newConnection = true, tls = '1.3', http3 = false, zeroRtt = false }) {
  if (!newConnection) return 1;                 // just the request
  if (zeroRtt) return 1;                        // request rides in the first flight
  if (http3) return 2;                          // combined QUIC handshake + request
  return 1 /* TCP */ + (tls === '1.2' ? 2 : 1) /* TLS */ + 1 /* request */;
}
console.log(roundTrips({ tls: '1.2' }), roundTrips({ http3: true }), roundTrips({ newConnection: false })); // 4 2 1
```
</details>

**21.** Code — Write `poolSize({ rps, latencyMs, headroom })` that returns the connection pool size for one upstream using Little's law, rounded up.
<details><summary>Answer</summary>

```js
const poolSize = ({ rps, latencyMs, headroom = 1.5 }) => Math.ceil(rps * (latencyMs / 1000) * headroom);
console.log(poolSize({ rps: 800, latencyMs: 25 })); // 30
```
</details>

**22.** Code — Write `batchLoader(fetchMany, { maxBatch, waitMs })` returning a `load(id)` function that collects ids requested within `waitMs`, fetches them with one `fetchMany(ids)` call (which resolves to a `Map` of id → value), and resolves each caller with its own value.
<details><summary>Answer</summary>

```js
function batchLoader(fetchMany, { maxBatch = 100, waitMs = 2 } = {}) {
  let pending = new Map(), timer = null;
  const flush = async () => {
    const batch = pending; pending = new Map(); timer = null;
    try {
      const results = await fetchMany([...batch.keys()]);
      for (const [id, waiters] of batch) waiters.forEach((w) => w.resolve(results.get(id)));
    } catch (err) {
      for (const waiters of batch.values()) waiters.forEach((w) => w.reject(err));
    }
  };
  return (id) => new Promise((resolve, reject) => {
    if (!pending.has(id)) pending.set(id, []);
    pending.get(id).push({ resolve, reject });   // duplicate ids share one fetch
    if (pending.size >= maxBatch) { clearTimeout(timer); flush(); }
    else timer ??= setTimeout(flush, waitMs);
  });
}

const load = batchLoader(async (ids) => { console.log('fetch', ids); return new Map(ids.map((i) => [i, i * 10])); });
console.log(await Promise.all([load(1), load(2), load(1)]));
// fetch [ 1, 2 ]
// [ 10, 20, 10 ]
```
</details>

**23.** Scenario — Your API serves users worldwide from one US region. Asian users report slowness; the product team wants to deploy the whole stack to an Asian region. What do you try first, and when is the full multi-region deployment justified?
<details><summary>Answer</summary>First, measure where the time goes; then put a CDN/edge in front to terminate TLS near users, cache what's cacheable, and keep warm connections to the origin — this often removes most handshake and static-asset latency for little effort. Reduce sequential API calls. A full regional deployment is justified when dynamic, uncacheable calls still dominate, when data residency requires it, or when regional availability matters — and it brings the data-replication problems of Chapters 13 and 32.</details>

**24.** Scenario — An internal team wants to move service-to-service communication from JSON over HTTP/1.1 to gRPC everywhere, citing performance. Traffic between the services is about 300 requests/s. How do you advise?
<details><summary>Answer</summary>At 300 requests/s, serialisation and connection overhead are negligible if connections are pooled; the performance argument is weak. gRPC's real benefits are schemas, generated clients, streaming, and deadline propagation — worth it if the team values those. Its costs are tooling, debuggability, and the need for L7 or client-side load balancing. Recommend fixing pooling and keep-alive first, and adopting gRPC where streaming or strict contracts justify it, not as a blanket performance migration.</details>

---

## Chapter Summary

- Round trips, not bandwidth, dominate most request latency; count them per operation, cold and
  warm, and cut them by reusing connections, batching, and moving closer to users.
- A new HTTPS connection costs two to three round trips of handshake plus a slow-start ramp; a
  warm, pooled connection costs neither.
- HTTP/2 multiplexes requests but inherits TCP head-of-line blocking; HTTP/3 over QUIC fixes it and
  matters most on lossy mobile networks.
- Pools need a concurrency cap sized by Little's law, an idle count at least equal to normal
  concurrency, and an idle timeout shorter than every server behind them.
- Opening connections per request exhausts ephemeral ports at a few hundred per second per
  destination — on hosts and, collectively, on NAT gateways.
- Long-lived HTTP/2 and gRPC connections defeat layer-4 load balancing; balance per request and
  cap connection age.

---

## Practice Exercises

1. Use `curl -w` with `time_namelookup`, `time_connect`, `time_appconnect`, and
   `time_starttransfer` against a distant site to measure each phase, then repeat with a reused
   connection.
2. Run the §5.9 demo with a server `keepAliveTimeout` of 1 second and a client idle timeout of 5
   seconds, pausing 2 seconds between waves, and reproduce the `ECONNRESET` race.
3. Read your load balancer's documentation and find its idle timeout default, then check whether
   your backend's keep-alive timeout is longer.
4. Count `TIME_WAIT` sockets on a busy host (`ss -s` on Linux) and work out which destination
   generates most of them.
5. Take an endpoint whose client makes N sequential calls, rewrite it to make one batched call,
   and measure the latency difference from a machine at least 50 ms away.

---

## Mini Project: A Connection Behaviour Lab

Build a small harness with a configurable upstream server and client that lets you reproduce, on
one machine, every connection-level failure in this chapter — and then fix each one by changing
configuration only.

- Inject artificial RTT and packet loss (via a local proxy you write, or `tc netem` on Linux).
- Report per-run: connections opened, `TIME_WAIT` count, resets, and latency p50/p99.
- Reproduce the keep-alive race, port exhaustion, and HTTP/2 head-of-line blocking under loss.
- Demonstrate L4 imbalance with a long-lived multiplexed connection and fix it with a maximum
  connection age.
- Write a one-page "connection settings" guide for your team based on the results.

---

**Next:** [Chapter 6 — DNS, CDNs, and the Edge →](06-dns-cdn-edge.md)
