# Chapter 9 — Real-Time Delivery

## Learning Objectives

- Choose between short polling, long polling, Server-Sent Events, and WebSockets for a use case
- Estimate the cost of polling versus persistent connections at a given scale
- Design a connection-gateway tier with a pub/sub backbone and a routing registry
- Make real-time delivery survive disconnects with sequence numbers, resume, and replay
- Prevent slow consumers and reconnection storms from taking the system down

---

## 9.1 The Problem: The Server Has Something to Say

HTTP's request/response model assumes the client asks and the server answers. Many features need
the opposite: the server learns something — a chat message arrived, a price moved, a build
finished, a collaborator moved their cursor — and the client should know *now*, without asking.

Every technique for this is a trade-off between **latency** (how soon the client hears),
**cost** (requests, connections, and memory the server pays for), and **complexity** (proxies,
reconnection, ordering, delivery guarantees). The right choice depends heavily on how often there's
news and how many clients are listening.

| Use case | Update frequency | Direction | Typical choice |
|---|---|---|---|
| Order status page | Rare (minutes) | Server → client | Polling, or SSE |
| Notifications badge | Occasional | Server → client | SSE, or mobile push when backgrounded |
| Live sports scores, stock tickers | Frequent, broadcast | Server → many clients | SSE with a CDN or fan-out tier |
| Chat | Frequent, both ways | Bidirectional | WebSockets |
| Multiplayer games, collaborative cursors | Very frequent, both ways | Bidirectional, latency-critical | WebSockets (or WebTransport/UDP for games) |

---

## 9.2 The Four Techniques

**Short polling** — the client asks every N seconds. Simple, works through anything, stateless on
the server. Latency is up to N seconds, and most responses are "nothing new".

**Long polling** — the client asks; the server *holds* the request open until there's news or a
timeout (say 30 seconds), then responds; the client immediately asks again. Near-real-time with
plain HTTP, but every message costs a full request cycle and the server holds one pending request
per client.

**Server-Sent Events (SSE)** — the client opens one HTTP request; the server keeps the response
open and writes events into it as a text stream (`text/event-stream`). Browsers have a native
`EventSource` API that **reconnects automatically** and sends a `Last-Event-ID` header so the
server can resume. One direction only — the client sends anything else via ordinary requests.

**WebSockets** — an HTTP request "upgrades" to a persistent, bidirectional, message-framed
connection. Lowest overhead per message in both directions, but it's a different protocol from
HTTP once established: caching, standard middleware, and many proxies don't apply, and
reconnection, resume, and heartbeats are yours to build.

| | Short polling | Long polling | SSE | WebSockets |
|---|---|---|---|---|
| Direction | Client pulls | Client pulls, server delays | Server → client | Both |
| Latency | Up to the interval | ~one RTT after the event | ~immediate | ~immediate |
| Requests per message | Many (mostly empty) | One per message (plus timeouts) | None | None |
| Server state per client | None between polls | One held request | One open stream | One open socket |
| Works through proxies/CDNs | Always | Usually (watch idle timeouts) | Usually — needs unbuffered proxying | Needs explicit upgrade support |
| Auto-reconnect and resume | N/A | Manual | **Built in** (`EventSource`, `Last-Event-ID`) | Manual |
| Binary data | Yes | Yes | Text only (encode) | Yes |
| Over HTTP/2 | Normal requests | Normal requests | Multiplexed — many streams on one connection | Separate connection per socket (RFC 8441 bootstrapping has limited support) |

A practical default: **SSE for server-to-client streams** (notifications, feeds, dashboards,
LLM token streaming), **WebSockets when the client also sends frequently** (chat, games,
collaboration), and **polling when updates are rare or the client count is small**. Don't reach
for WebSockets just because a feature is "real-time": a one-way stream over SSE reuses your HTTP
infrastructure, authentication, and tooling.

> 💼 **Interview** — "Why WebSockets and not polling?" is a cost question in disguise. Give the
> arithmetic: N clients polling every T seconds is N/T requests per second whether or not anything
> happened; persistent connections cost memory per client but no requests until there's news.

---

## 9.3 What Polling Costs

The arithmetic decides more cases than intuition does:

```js
// Requests per second the server absorbs for each technique, and how many were useful
function pollingCost({ clients, intervalSec, updatesPerClientPerMin }) {
  const shortPollRps = clients / intervalSec;
  const usefulRps = (clients * updatesPerClientPerMin) / 60;
  return {
    shortPollRps: Math.round(shortPollRps),
    emptyFraction: `${Math.max(0, (1 - usefulRps / shortPollRps) * 100).toFixed(1)}%`,
    // Long polling: one request per update, plus one per timeout when nothing happens
    longPollRps: Math.round(usefulRps + clients / 30),
  };
}

console.log(pollingCost({ clients: 1_000_000, intervalSec: 5, updatesPerClientPerMin: 0.5 }));
// { shortPollRps: 200000, emptyFraction: '95.8%', longPollRps: 41667 }
console.log(pollingCost({ clients: 2_000, intervalSec: 10, updatesPerClientPerMin: 0.1 }));
// { shortPollRps: 200, emptyFraction: '98.3%', longPollRps: 70 }
```

A million clients polling every 5 seconds is 200,000 requests per second — each with TLS,
authentication, routing, and a database or cache lookup — and 96% of them say "nothing new". With
persistent connections, the same million clients cost memory, but the server only does work when
there's news. On the other hand, 2,000 internal dashboard users polling every 10 seconds is 200
requests per second: trivial, cacheable, and not worth a persistent-connection tier.

If polling is the right answer, make it cheap: return `304 Not Modified` via `ETag`
(Chapter 6) so empty polls cost almost nothing, let the server tell clients how often to poll
(a `Retry-After` or poll-interval field it can raise under load), and add jitter so clients don't
synchronise.

---

## 9.4 Holding a Million Connections

Persistent connections change the scaling model. A stateless API server is sized by **requests
per second**; a connection server is sized by **concurrent connections** and **messages per
second**, and its main constraint is usually memory.

Per-connection costs, roughly:

| Cost | Typical size | Notes |
|---|---|---|
| Kernel socket buffers | 4–64 KB+ | Tunable; auto-sized by load — mostly idle connections stay small |
| TLS session state | 10–40 KB | Depends on library and buffering |
| Application state | 1–10 KB | Subscriptions, user ID, auth context, a small send queue |
| File descriptor | 1 | Raise `ulimit -n` and kernel limits well above the target |

At ~50 KB per connection, 1 million connections is ~50 GB of memory across the fleet. A tuned
event-driven gateway (Node, Go, Rust, Erlang/Elixir) can hold 100,000–500,000 mostly idle
connections per node; Chapter 2's planning figure of 50,000 per node is deliberately conservative
to leave room for message bursts and reconnection storms. The non-memory limits:

- **Load balancer idle timeouts.** Many balancers close connections idle for 60 seconds. Send a
  **heartbeat** (a WebSocket ping, or an SSE comment line) well inside that window — every 20–30
  seconds — so idle connections stay open, and so dead ones are detected.
- **Per-node connection limits on the balancer and NAT** (Chapter 5).
- **Deploy time.** Draining a node with 200,000 connections means 200,000 clients reconnect
  elsewhere (§9.7).

Keep connection servers **thin**: their job is holding sockets and moving bytes. Authentication
happens at connect time, business logic lives in ordinary services, and the gateway forwards. A
gateway that also runs business logic must be redeployed — dropping every connection — whenever
that logic changes.

---

## 9.5 Architecture: Gateways, a Backbone, and a Routing Registry

With more than one connection server, a message for user Ana must reach whichever gateway node
holds Ana's connection. The standard architecture has three parts:

```text
                       ┌──────────────────────────────────────────────┐
clients ◀═══ws/sse═══▶ │ gateway-1   gateway-2   gateway-3   …        │  thin: sockets, auth, heartbeats
                       └──────┬───────────┬───────────┬───────────────┘
                              │ subscribe │           │
                       ┌──────▼───────────▼───────────▼───────────────┐
                       │      pub/sub backbone (topics/channels)      │  Redis pub/sub, NATS, Kafka…
                       └──────▲───────────────────────────────────────┘
                              │ publish
     chat service, notification service, score feed, …  ──▶  durable store (history, offsets)
```

**Gateways** hold connections and subscribe to the backbone for the topics their connected
clients care about — a user's personal topic, a chat room, a match's score channel.

**The backbone** routes published messages to every gateway subscribed to that topic. Choices
differ in delivery semantics:

| Backbone | Delivery | Retained history | Good for |
|---|---|---|---|
| Redis pub/sub | At-most-once: offline subscribers miss messages | None | Ephemeral signals (typing indicators, presence), with resume handled elsewhere |
| Redis Streams, NATS JetStream | At-least-once with consumer offsets | Configurable | Moderate scale with replay |
| Kafka (or similar log) | At-least-once, ordered per partition | Long | High volume, replay, many consumers (Chapter 24) |

**The routing registry** answers "which gateway holds user X?" — needed when messages are
addressed to users rather than broadcast to topics. Gateways write `user → gateway` entries (with a
TTL refreshed by heartbeats) to a fast store when users connect. The alternative — publish every
user-addressed message to a per-user topic and let the right gateway pick it up — trades the
registry for many more subscriptions on the backbone. Both are common; Chapter 37 builds the
registry version for chat.

### Fan-out

Broadcast to large audiences is where the backbone design matters. A message to a channel with
500,000 subscribers spread across 50 gateways should be published **once to the backbone**,
delivered **once to each of the 50 gateways**, and expanded to individual sockets **locally on each
gateway** — 50 backbone deliveries, not 500,000. For extreme broadcast (a live event with millions
of viewers watching the same score), SSE has an extra option: since every viewer gets the same
stream, a CDN that supports streaming responses can fan it out from its edges.

---

## 9.6 Delivery Guarantees: Disconnects Are Normal

Mobile clients lose connectivity constantly — tunnels, elevators, network handovers, app
backgrounding. A real-time design that only delivers to currently connected clients silently loses
every message sent during a disconnect. Correct delivery needs three things.

**Durable history with sequence numbers.** Every message in a stream (a conversation, a user's
notification feed, a document) gets a monotonically increasing sequence number from the service
that persisted it. The real-time push is a *fast path*; the durable store is the source of truth.

**Resume from the last seen position.** On reconnect, the client says "I have everything up to
sequence 1,041" — SSE's `Last-Event-ID` does this automatically — and the server replays the gap
from a buffer or the durable store. If the gap is too old to replay, the server tells the client to
**reset**: refetch current state through the normal API.

**Idempotent clients.** Between the push path and the replay path, the same message can arrive
twice. Clients deduplicate by sequence number or message ID. This is at-least-once delivery with
deduplication — the same pattern Chapter 24 generalises.

```text
client                         gateway                     durable store
  │──── connect (last_seq=1041) ──▶│                             │
  │                                │──── fetch seq > 1041 ──────▶│
  │◀─── 1042, 1043, 1044 ──────────│◀────────────────────────────│
  │◀─── 1045 (live) ───────────────│   (subscribed to backbone)  │
```

**Ordering** comes from the sequence numbers, not the network: a client that receives 1,045
before 1,044 (one via replay, one via live push) sorts by sequence and waits briefly for gaps.
Order is only meaningful *within* a stream; across streams (two different chats) there's no global
order to preserve, and designs that try to impose one don't scale.

### Backpressure and slow consumers

A client on a bad network reads slower than the server writes. Its socket's send buffer fills,
then the gateway's per-connection queue grows — and a gateway holding 100,000 connections, a few
hundred of them slow, can run out of memory because of clients that aren't even receiving the
messages. Every gateway needs a **bounded per-connection buffer** and a policy for when it fills:

| Policy | Use when |
|---|---|
| Disconnect the client (it will reconnect and resume) | Durable history exists — the default |
| Drop old messages, keep the newest | Only the latest state matters (prices, positions, cursors) |
| Coalesce (replace queued updates for the same key) | Many updates to the same entity (a score, a document field) |
| Block the publisher | Almost never — one slow client must not stall everyone |

---

## 9.7 Reconnection Storms, Deploys, and Mobile

The most dangerous moment for a connection tier is when a lot of clients reconnect at once: a
gateway crashes, a deploy restarts a node, a network blip drops a region. Hundreds of thousands of
clients reconnect within the same second — every one doing a TLS handshake, authentication, a
registry write, subscription setup, and a history replay. This **thundering herd** can overload the
remaining gateways and the auth and history services behind them, causing more disconnects, and a
self-sustaining storm.

Defences, layered:

- **Exponential backoff with jitter on the client** (Chapter 28): first retry after a random 0–2
  seconds, doubling the range each time. Jitter spreads reconnects over time; without it, every
  client retries in lockstep. SSE's `retry:` field lets the server set the delay — and the server
  can raise it during an incident.
- **Server-directed reconnect during deploys.** Drain a gateway gradually: close connections in
  batches over a few minutes, with a reconnect hint, rather than all at once.
- **Cheap reconnects.** Resume tokens or short-lived session tickets so a reconnecting client skips
  full authentication; replay from a gateway-local or cache-backed buffer rather than the database.
- **Admission control.** Gateways cap the rate of new connections they accept per second and reject
  the excess with a retry hint, protecting the services behind them (Chapter 29).
- **Headroom.** The remaining nodes must absorb the connections of a failed node — N+1 per zone,
  as in Chapter 4.

**Mobile apps in the background** can't hold connections — the operating system suspends them. For
those clients, real-time delivery means **platform push notifications** (APNs for iOS, FCM for
Android): the backend sends to the push service, which wakes the device. Push has its own
semantics — best-effort, rate-limited, possibly coalesced — so it's a nudge ("you have a new
message"), and the app fetches the authoritative data when it opens. Chapter 38 designs the
notification pipeline that sits behind it.

---

## 9.8 Common Mistakes and Best Practices

**Mistake: WebSockets for one-way streams.** SSE reuses HTTP infrastructure and has reconnect and
resume built in; WebSockets add protocol complexity for no benefit when only the server talks.

**Mistake: fire-and-forget pushes with no history.** Every message sent while a client is
disconnected is lost; real-time delivery must be backed by a durable, sequenced store.

**Mistake: no heartbeat.** Load balancers close idle connections at their idle timeout, and dead
connections are never detected.

**Mistake: unbounded per-connection send queues.** A few slow clients exhaust the gateway's
memory and take every other connection down with it.

**Mistake: reconnecting without jitter.** After a node failure, every client reconnects at the
same instant and the herd overloads the rest of the tier.

**Mistake: business logic in the gateway.** Every logic change requires redeploying the
connection tier, which disconnects every client.

**Mistake: polling without conditional requests.** Most polls return nothing; without `ETag` and
`304`, each one costs a full response.

**Best practice: real-time push as a fast path over a durable source of truth.** Clients resume
by sequence number and reset when the gap is too old.

**Best practice: fan out once per gateway, not once per subscriber.** Publish to the backbone once
and expand to sockets locally.

**Best practice: let the server control the client's reconnect delay.** It can slow clients down
during an incident without shipping an app update.

---

## 9.9 Real-World Example: A Resumable SSE Hub

This SSE server implements the delivery properties from §9.6 and §9.7 for a single node: sequence
numbers per channel, resume via `Last-Event-ID`, a bounded replay buffer with a reset event when the
gap is too old, heartbeats, a jittered reconnect delay, and disconnection of slow consumers. In a
multi-node deployment, `publish` would be driven by a backbone subscription and the replay buffer
backed by the durable store.

```js
// sse-hub.mjs — Server-Sent Events with resume (Last-Event-ID), heartbeats, a bounded
// replay buffer per channel, and disconnection of slow clients. Dependency-free.
import http from 'node:http';

export class SseHub {
  #channels = new Map(); // channel → { seq, buffer: [{ id, data }], clients: Set<res> }
  #opts;

  constructor({ replay = 1_000, heartbeatMs = 25_000, maxBufferedBytes = 256 * 1024 } = {}) {
    this.#opts = { replay, heartbeatMs, maxBufferedBytes };
  }

  #channel(name) {
    if (!this.#channels.has(name)) this.#channels.set(name, { seq: 0, buffer: [], clients: new Set() });
    return this.#channels.get(name);
  }

  publish(name, data) {
    const ch = this.#channel(name);
    const event = { id: ++ch.seq, data: JSON.stringify(data) };
    ch.buffer.push(event);
    if (ch.buffer.length > this.#opts.replay) ch.buffer.shift();  // bounded memory per channel
    for (const res of ch.clients) this.#send(res, event);
  }

  #send(res, { id, data, event }) {
    // Slow consumer: socket and Node buffers are full. Buffering more just moves the
    // problem into our memory, so cut the client loose; it will reconnect and resume.
    if (res.writableLength > this.#opts.maxBufferedBytes) { res.destroy(); return; }
    res.write(`${event ? `event: ${event}\n` : ''}id: ${id}\ndata: ${data}\n\n`);
  }

  // Handle GET /events/:channel
  subscribe(name, req, res) {
    const ch = this.#channel(name);
    res.writeHead(200, {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-store',      // never let a proxy or CDN cache a live stream
      'X-Accel-Buffering': 'no',        // ask buffering proxies (e.g. NGINX) to pass bytes through
    });
    res.write(`retry: ${2_000 + Math.floor(Math.random() * 3_000)}\n\n`); // jittered reconnect delay

    const lastId = Number(req.headers['last-event-id'] ?? 0);
    if (lastId) {
      const oldest = ch.buffer[0]?.id ?? ch.seq + 1;
      if (lastId < oldest - 1) {
        // The gap is older than the replay buffer: tell the client to refetch full state
        this.#send(res, { id: ch.seq, event: 'reset', data: '{}' });
      } else {
        for (const e of ch.buffer) if (e.id > lastId) this.#send(res, e);
      }
    }

    ch.clients.add(res);
    // Comment lines keep idle connections alive through load balancers with idle timeouts
    const heartbeat = setInterval(() => res.write(': hb\n\n'), this.#opts.heartbeatMs);
    req.on('close', () => { clearInterval(heartbeat); ch.clients.delete(res); });
  }
}

// ── Demo: a live subscriber sees 4–5, misses 6–7 while disconnected, then resumes ──────
const hub = new SseHub({ replay: 100 });
const server = http.createServer((req, res) => {
  const m = req.url.match(/^\/events\/([\w-]+)$/);
  if (m) return hub.subscribe(m[1], req, res);
  res.writeHead(404).end();
});
await new Promise((r) => server.listen(0, r));
const base = `http://127.0.0.1:${server.address().port}`;

async function readEvents(lastEventId, count) {
  const res = await fetch(`${base}/events/match-42`, {
    headers: lastEventId ? { 'Last-Event-ID': String(lastEventId) } : {} });
  const seen = [];
  let buf = '';
  for await (const chunk of res.body) {
    buf += Buffer.from(chunk).toString();
    let i;
    while ((i = buf.indexOf('\n\n')) !== -1) {
      const block = buf.slice(0, i); buf = buf.slice(i + 2);
      const id = block.match(/^id: (\d+)$/m)?.[1];
      if (id) seen.push(Number(id));
    }
    if (seen.length >= count) break;      // leaving the loop cancels the body and closes the stream
  }
  return seen;
}

for (let n = 1; n <= 3; n++) hub.publish('match-42', { score: n });
const first = readEvents(0, 2);               // a live subscriber, connected after event 3
await new Promise((r) => setTimeout(r, 50));
hub.publish('match-42', { score: 4 }); hub.publish('match-42', { score: 5 });
console.log('live:', await first);            // live: [ 4, 5 ]
hub.publish('match-42', { score: 6 }); hub.publish('match-42', { score: 7 });   // while disconnected
console.log('resumed after 5:', await readEvents(5, 2)); // resumed after 5: [ 6, 7 ]
server.closeAllConnections(); server.close();
```

The decision to defend is **disconnecting slow clients instead of buffering for them.** It feels
hostile: the client is merely on a bad network, and buffering a little more would let it catch up.
But the gateway's memory is shared by every connection on it, and a slow client's queue grows for
as long as it stays slow — buffering turns one user's bad cellular link into an out-of-memory crash
that disconnects 100,000 healthy users at once. Because delivery is backed by sequence numbers and
replay, disconnecting costs the slow client almost nothing: it reconnects, sends its
`Last-Event-ID`, and receives exactly what it missed, from storage instead of from the gateway's
heap. The replay mechanism is what makes the harsh policy safe.

---

## 9.10 Questions

**1.** MCQ — Which technique provides automatic reconnection and resume in browsers without extra client code? a) Short polling b) Long polling c) Server-Sent Events via `EventSource` d) WebSockets
<details><summary>Answer</summary>**c)** `EventSource` reconnects on its own and sends `Last-Event-ID`, so a server that assigns event IDs can resume the stream.</details>

**2.** MCQ — 400,000 clients poll every 4 seconds. How many requests per second does the server absorb? a) 1,600 b) 10,000 c) 100,000 d) 1.6 M
<details><summary>Answer</summary>**c)** 400,000 / 4 = 100,000 requests per second, regardless of whether anything changed.</details>

**3.** MCQ — A load balancer closes connections idle for 60 seconds. How often should an SSE server send heartbeats? a) Every 5 minutes b) Every 60 seconds c) Every 20–30 seconds d) Never — SSE doesn't need them
<details><summary>Answer</summary>**c)** Comfortably inside the idle timeout, so idle-but-healthy streams are never cut.</details>

**4.** MCQ — Which pub/sub backbone delivers at-most-once, so subscribers that are offline miss messages? a) Kafka b) Redis pub/sub c) NATS JetStream d) Redis Streams
<details><summary>Answer</summary>**b)** Redis pub/sub keeps no history; messages published while a subscriber is disconnected are gone.</details>

**5.** MCQ — For a live score update sent to 600,000 viewers across 60 gateway nodes, how many backbone deliveries should occur? a) 1 b) About 60 c) 600,000 d) 36 million
<details><summary>Answer</summary>**b)** One publish, one delivery per subscribed gateway, then local fan-out to sockets on each gateway.</details>

**6.** MCQ — What's the best policy when a client's per-connection buffer fills and the stream has durable, sequenced history? a) Buffer without limit b) Block the publisher c) Disconnect the client so it reconnects and resumes d) Silently drop messages
<details><summary>Answer</summary>**c)** Disconnecting protects the gateway, and resume by sequence number means the client loses nothing.</details>

**7.** Short — Why is real-time push described as a "fast path" over a durable source of truth?
<details><summary>Answer</summary>Connections drop routinely, so anything sent only over the live connection can be lost. The durable, sequenced store is authoritative; the push just delivers it sooner, and clients fill gaps from the store on reconnect.</details>

**8.** Short — Why should connection gateways be kept "thin"?
<details><summary>Answer</summary>Redeploying a gateway disconnects every client on it. If business logic lives elsewhere, the gateway rarely needs to change, and logic can be deployed without causing reconnection storms.</details>

**9.** Short — What is a reconnection storm, and name two defences?
<details><summary>Answer</summary>A large number of clients reconnecting at once after a failure or deploy, overloading gateways and the services behind them. Defences: jittered exponential backoff (server-controlled via `retry:`), gradual draining, cheap resume tokens, admission control, and N+1 headroom.</details>

**10.** Short — When does short polling beat persistent connections?
<details><summary>Answer</summary>When updates are rare and client counts are small enough that N/T requests per second is trivial — e.g. a few thousand internal users — especially with `ETag`/`304` making empty polls cheap and responses cacheable.</details>

**11.** Short — Why do mobile apps need platform push (APNs/FCM) in addition to WebSockets or SSE?
<details><summary>Answer</summary>The OS suspends backgrounded apps and their connections; only the platform push service can wake the device. Push is best-effort and coalesced, so it nudges the app, which then fetches authoritative data.</details>

**12.** Output — What does this print?
```js
const cost = (clients, intervalSec) => clients / intervalSec;
console.log(cost(1_200_000, 15), cost(1_200_000, 60));
```
<details><summary>Answer</summary>`80000 20000` — the request rate scales inversely with the interval, and latency scales with it.</details>

**13.** Output — Using the §9.9 hub with `replay: 100`, channel `c` has had 500 events published (ids 1–500). A client reconnects with `Last-Event-ID: 450`. Which events does it receive?
<details><summary>Answer</summary>**451–500.** The buffer holds ids 401–500; 450 is within it, so the server replays everything after 450.</details>

**14.** Output — Same hub and channel, but the client reconnects with `Last-Event-ID: 120`. What does it receive?
<details><summary>Answer</summary>**A single `reset` event (id 500).** The oldest buffered id is 401, so the gap 121–400 can't be replayed; the client must refetch full state.</details>

**15.** Output — A gateway node holds 120,000 connections at about 40 KB each. How much memory do connections use, in GB?
<details><summary>Answer</summary>**4.8 GB.** 120,000 × 40 KB = 4,800,000 KB ≈ 4.8 GB.</details>

**16.** Debug — Users of a dashboard using SSE behind NGINX see no updates for minutes, then a burst of them at once. Direct connections to the app work fine. Cause and fix?
<details><summary>Answer</summary>The proxy is buffering the response and flushing only when its buffer fills. Disable proxy buffering for the stream (`proxy_buffering off`, or the `X-Accel-Buffering: no` response header) and ensure no compression middleware buffers it either.</details>

**17.** Debug — After each gateway deploy, the authentication service's error rate spikes and some users can't reconnect for several minutes. Cause and fixes?
<details><summary>Answer</summary>A reconnection storm: draining a node disconnects all its clients at once, and every reconnect performs full authentication. Drain gradually in batches, add jittered backoff (server-set `retry:`), issue short-lived resume tickets so reconnects skip full auth, and rate-limit new connections per gateway.</details>

**18.** Debug — Chat users occasionally report missing messages that "appear after restarting the app". The system pushes messages via Redis pub/sub to WebSocket gateways. What's happening and how do you fix it?
<details><summary>Answer</summary>Messages published while a client was disconnected (or while its gateway was restarting) are lost by at-most-once pub/sub; restarting the app triggers a full history fetch that finds them. Assign per-conversation sequence numbers in the durable store, have clients resume from their last sequence on reconnect, and deduplicate by message ID.</details>

**19.** Debug — A WebSocket gateway's memory climbs steadily during a major live event and it crashes with OOM, though connection count is stable. Cause and fix?
<details><summary>Answer</summary>Slow consumers: high message volume plus clients on poor networks cause per-connection send queues to grow unboundedly. Bound each queue, and disconnect, drop-oldest, or coalesce when it fills — coalescing is ideal for score-style updates where only the latest value matters.</details>

**20.** Code — Write `backoffDelay(attempt, { baseMs, capMs, rand })` implementing "full jitter": a random delay between 0 and `min(cap, base × 2^attempt)`.
<details><summary>Answer</summary>

```js
function backoffDelay(attempt, { baseMs = 1_000, capMs = 60_000, rand = Math.random } = {}) {
  const ceiling = Math.min(capMs, baseMs * 2 ** attempt);
  return Math.floor(rand() * ceiling); // full jitter: spreads a herd evenly over the window
}
console.log(backoffDelay(0, { rand: () => 0.5 }), backoffDelay(10, { rand: () => 0.5 })); // 500 30000
```
</details>

**21.** Code — Write a `Coalescer` that holds pending updates per key for one connection and, on `drain()`, returns only the latest value for each key in first-queued order.
<details><summary>Answer</summary>

```js
class Coalescer {
  #pending = new Map();
  push(key, value) {
    if (this.#pending.has(key)) this.#pending.get(key).value = value; // replace in place: keeps order
    else this.#pending.set(key, { value });
  }
  drain() {
    const out = [...this.#pending].map(([key, { value }]) => ({ key, value }));
    this.#pending.clear();
    return out;
  }
}
const c = new Coalescer();
c.push('score', 1); c.push('clock', '10:02'); c.push('score', 2);
console.log(c.drain()); // [ { key: 'score', value: 2 }, { key: 'clock', value: '10:02' } ]
```
</details>

**22.** Code — Write `reorder(buffer, nextExpected)` for a client that receives messages possibly out of order: given an array of `{ seq }` messages and the next sequence it expects, return `{ deliver, keep }` — the contiguous run starting at `nextExpected`, and the rest to hold.
<details><summary>Answer</summary>

```js
function reorder(buffer, nextExpected) {
  const bySeq = new Map(buffer.map((m) => [m.seq, m]));
  const deliver = [];
  while (bySeq.has(nextExpected)) { deliver.push(bySeq.get(nextExpected)); bySeq.delete(nextExpected++); }
  // Anything below nextExpected is a duplicate and can be discarded
  const keep = [...bySeq.values()].filter((m) => m.seq > nextExpected).sort((a, b) => a.seq - b.seq);
  return { deliver, keep };
}
console.log(reorder([{ seq: 7 }, { seq: 5 }, { seq: 4 }, { seq: 3 }], 4));
// { deliver: [ { seq: 4 }, { seq: 5 } ], keep: [ { seq: 7 } ] }
```
</details>

**23.** Scenario — A team building LLM-powered chat wants WebSockets to stream tokens to the browser. Users send a prompt and receive a streamed answer. What do you recommend?
<details><summary>Answer</summary>SSE (or a streamed HTTP response read with `fetch`): the prompt is an ordinary `POST`, and the answer streams one way. It reuses existing HTTP auth, load balancing, and observability, works through most proxies with buffering disabled, and supports resume if generations are persisted with event IDs. WebSockets make sense only if the client must send frequently during the stream (e.g. interrupting, collaborative sessions).</details>

**24.** Scenario — A sports app expects 5 million concurrent viewers for a final, each receiving the same score and commentary updates a few times per second. How do you architect delivery?
<details><summary>Answer</summary>Treat it as broadcast: one stream per match, identical for all viewers. Serve it over SSE through a CDN that supports streaming fan-out, or through a gateway tier fed by a backbone with one publish per update and local fan-out. Use coalescing so slow clients only get the latest score, sequence numbers so reconnects resume cheaply, server-controlled jittered reconnects, and pre-scaled capacity with N+1 headroom. Personalised features (bets, notifications) go over a separate, smaller channel.</details>

---

## Chapter Summary

- Pick real-time techniques by update frequency, direction, and client count: polling when updates
  are rare, SSE for one-way streams, WebSockets when the client sends frequently too.
- Polling costs clients ÷ interval requests per second regardless of activity; persistent
  connections cost memory per client but no requests until there's news.
- Connection tiers are sized by concurrent connections and memory, need heartbeats inside load
  balancer idle timeouts, and should be thin so deploys are rare.
- Gateways, a pub/sub backbone, and a routing registry let any service reach any connected client,
  with fan-out once per gateway rather than once per subscriber.
- Disconnects are normal, so push must be a fast path over durable, sequenced history: clients
  resume by sequence number, reset when the gap is too old, and deduplicate.
- Bound every per-connection buffer and disconnect, drop, or coalesce slow consumers; spread
  reconnects with server-controlled jittered backoff and gradual draining.

---

## Practice Exercises

1. Use the §9.3 cost function to find the polling interval at which short polling and long polling
   cost the same number of requests for your own product's update rate.
2. Connect to the §9.9 hub from a browser with `EventSource`, kill the server, restart it, and
   watch the browser reconnect and resume (persist the buffer across restarts first).
3. Read your load balancer's documentation on WebSocket and streaming support: idle timeouts,
   maximum connection duration, and whether it buffers responses.
4. Write a load generator that opens 20,000 SSE connections to one node and measure memory per
   connection, then repeat with TLS.
5. Add a Redis (or in-process) pub/sub backbone so two hub processes share channels, with replay
   served from a shared store rather than per-process buffers.

---

## Mini Project: A Live Event Feed

Build a multi-node live-update service for a sports-match page: viewers subscribe to a match's
stream and see score and commentary updates within a second, survive disconnects without missing
anything, and the system survives a node being killed during the event.

- At least two gateway processes behind a load balancer, sharing a pub/sub backbone.
- Durable per-match history with sequence numbers; resume via `Last-Event-ID`; reset on stale gaps.
- Score updates coalesced per connection; commentary delivered in full.
- Heartbeats, a bounded per-connection buffer, and slow-client disconnection.
- A chaos script that kills one gateway under 10,000 simulated viewers and reports reconnect time
  distribution, missed messages (must be zero), and peak load on the history store.

---

**Next:** [Chapter 10 — Storage Engines and Indexes →](10-storage-engines.md)
