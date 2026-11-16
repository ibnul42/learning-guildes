# Chapter 6 — DNS, CDNs, and the Edge

## Learning Objectives

- Trace a DNS lookup through each cache, and predict how long a record change takes to reach users
- Choose between DNS-based routing, anycast, and a global load balancer for failover and proximity
- Write `Cache-Control` headers that make shared caches fast without ever leaking private data
- Design cache keys and URL schemes that make invalidation unnecessary for most content
- Decide what belongs at the edge — caching, TLS, auth checks, compute — and what doesn't

---

## 6.1 How a DNS Lookup Actually Works

DNS turns a name like `api.example.com` into addresses. It's also the first network operation of
nearly every request, a global distributed cache with its own consistency model, and one of the
most common hidden dependencies in post-mortems.

A lookup passes through several layers, and every one of them caches:

```text
app ──▶ OS stub resolver ──▶ recursive resolver (ISP, corporate, 8.8.8.8, 1.1.1.1)
          (cache)               (cache)      │
                                             ├──▶ root servers         "who handles .com?"
                                             ├──▶ .com TLD servers     "who handles example.com?"
                                             └──▶ example.com's authoritative servers
                                                                       "api.example.com = 203.0.113.10, TTL 60"
```

The **authoritative** servers hold the truth for a zone. **Recursive resolvers** do the walking
and cache each answer for its **TTL** (time to live, in seconds), so the root and TLD steps are
almost always cached and a typical lookup costs one round trip to a nearby resolver — or nothing,
if the OS or application already has the answer.

Record types worth knowing:

| Type | Maps | Notes |
|---|---|---|
| `A` / `AAAA` | Name → IPv4 / IPv6 address | The end of most lookups |
| `CNAME` | Name → another name | Adds a lookup; not allowed at the zone apex (`example.com` itself) |
| `ALIAS` / `ANAME` / flattening | Apex name → another name, resolved by the provider | Vendor feature that works around the apex `CNAME` rule — how `example.com` points at a CDN |
| `NS` | Zone → its authoritative servers | Delegation |
| `TXT` | Name → text | Domain verification, SPF/DKIM for email |
| `SRV` / `HTTPS` / `SVCB` | Name → service endpoints and parameters | `HTTPS` records can advertise HTTP/3 support before the first connection |

**Negative answers are cached too.** "This name doesn't exist" (NXDOMAIN) is cached for a period
set by the zone's SOA record. Look up a name *before* creating it — say, during a deploy script —
and resolvers may keep insisting it doesn't exist for minutes or hours afterwards.

### How long a change takes

A record change propagates as caches expire, so the worst case is roughly the old TTL — *if*
everyone honours it. Many don't:

- Some resolvers enforce a minimum TTL and treat `TTL 5` as 30 or 60 seconds.
- Language runtimes and HTTP clients cache lookups themselves, sometimes for the life of the
  process or connection pool. A pooled connection never re-resolves at all until it's closed.
- Browsers keep their own short cache.

> ⚠️ **Gotcha** — Lowering a TTL only helps for the *next* change. If a record has `TTL 86400`
> (one day) and you lower it to 60 just before a migration, resolvers that cached it yesterday will
> keep the old value for up to a day. Lower TTLs at least one old-TTL period *before* the change,
> then raise them again afterwards.

---

## 6.2 DNS and Anycast as Traffic Tools

Because DNS decides where a client connects, it's also a routing tool. Managed DNS services offer
routing policies on top of plain records:

| Policy | How it works | Good for |
|---|---|---|
| Weighted | Answers in proportion to configured weights | Gradual migrations, canarying a new region |
| Latency / geo-based | Answers based on the resolver's (or client subnet's) location | Sending users to the nearest region |
| Failover with health checks | Primary answer while healthy, secondary otherwise | Simple active-passive regional failover |
| Multi-value | Several healthy addresses; client picks | Crude client-side load spreading |

DNS-based routing has two structural limits. **Granularity**: the authoritative server sees the
*resolver*, not the user, so a user on a distant corporate or public resolver can be routed by the
resolver's location (the EDNS Client Subnet extension mitigates this where resolvers support it).
**Speed**: failover is bounded by TTLs and by clients that ignore them, so realistic DNS failover is
minutes, not seconds.

**Anycast** routes at the network layer instead. The same IP address is announced from many
locations via BGP, and the internet's routing delivers each packet to the topologically nearest
one. If a location fails, it stops announcing and traffic shifts within seconds — no client caches
to wait out. Anycast is how CDNs, public DNS resolvers, and DDoS-mitigation networks put one
address in hundreds of places.

| | DNS-based routing | Anycast | Global L7 load balancer (anycast front + proxy) |
|---|---|---|---|
| Failover speed | Minutes (TTL + client caching) | Seconds (BGP) | Seconds |
| Routing decision based on | Resolver location, health checks | Network topology | Full request: health, load, latency, headers |
| Who can run it | Anyone with managed DNS | Requires your own IP space and BGP — or a provider | Cloud providers and CDNs |
| Typical use | Region selection, migrations | CDN and DNS edges | Multi-region apps on one global IP |

For most teams the practical choice is a CDN or cloud global load balancer that combines both:
an anycast address that terminates connections at the nearest edge, which then proxies to a
healthy origin region it selects per request. You get anycast's fast failover and L7's routing
intelligence without operating BGP. Chapter 32 builds multi-region failover on this.

> 💼 **Interview** — "How do users get routed to the nearest data centre?" A complete answer
> distinguishes DNS geo-routing (simple, coarse, slow to fail over) from anycast (fast, network-
> level) and notes that most real deployments use an anycast edge that proxies to regions.

---

## 6.3 What a CDN Does

A **content delivery network** is a fleet of caching reverse proxies in hundreds of locations
("points of presence", PoPs) near users. Clients connect to the nearest PoP; on a cache **hit**
the PoP responds directly; on a **miss** it fetches from your origin, stores the response if
allowed, and responds.

The benefits compound:

| Benefit | Mechanism |
|---|---|
| Lower latency | Short round trips to a nearby PoP; TLS terminated there (Chapter 5) |
| Origin offload | Hits never reach your servers |
| Bandwidth cost | CDN egress is usually cheaper than cloud egress at volume |
| Burst absorption | A viral page is served from edge caches, not your database |
| DDoS absorption | Attack traffic is spread across a network far larger than your origin |
| Connection efficiency | PoPs hold warm, multiplexed connections to the origin |

**Hit ratio** is the number that matters, and small changes at the top have large effects:

```js
// Origin load with a CDN in front: only misses reach the origin
const originRps = (edgeRps, hitRatio) => Math.round(edgeRps * (1 - hitRatio));
for (const h of [0.9, 0.95, 0.99]) console.log(`hit ratio ${h * 100}% → origin ${originRps(100_000, h)} req/s`);
// hit ratio 90% → origin 10000 req/s
// hit ratio 95% → origin 5000 req/s
// hit ratio 99% → origin 1000 req/s
```

Going from 95% to 99% cuts origin load by 5×. Track two variants separately: **request hit
ratio** (what fraction of requests are hits — drives origin CPU) and **byte hit ratio** (what
fraction of bytes — drives origin bandwidth and egress cost). A site with many small cached API
responses and a few large uncached downloads can have a high request hit ratio and a terrible byte
hit ratio.

### Tiered caching and request collapsing

With hundreds of PoPs, each one misses independently. A newly published asset could produce a
miss from every PoP — hundreds of origin requests for one object. Two features fix this:

- **Tiered caching / origin shield**: PoPs fetch misses from a regional or central cache tier
  rather than the origin, so the origin sees one request per object per shield, not one per PoP.
- **Request collapsing** (collapsed forwarding): when many requests for the same uncached object
  arrive at once, the PoP sends one to the origin and holds the rest until it returns.

Together they turn a cache-miss stampede (Chapter 16) into a single origin fetch.

```text
users ──▶ PoP (Tokyo)  ─┐
users ──▶ PoP (Osaka)  ─┼─▶ shield (Tokyo region) ─┐
users ──▶ PoP (Seoul)  ─┘                          ├─▶ origin
users ──▶ PoP (Paris)  ─┬─▶ shield (Frankfurt)  ───┘
users ──▶ PoP (London) ─┘
```

---

## 6.4 HTTP Caching Semantics

CDNs, browsers, and proxies decide what to cache from HTTP headers — mainly `Cache-Control`. Its
directives are easy to confuse:

| Directive | Meaning |
|---|---|
| `max-age=N` | Fresh for N seconds in any cache (browser and shared) |
| `s-maxage=N` | Overrides `max-age` for **shared** caches (CDNs, proxies) only |
| `public` | Shared caches may store it, even if normally they wouldn't (e.g. with auth) |
| `private` | Only the end user's browser may store it — never a shared cache |
| `no-cache` | *May* be stored, but must be **revalidated** with the origin before each use |
| `no-store` | Must not be stored anywhere |
| `immutable` | Will never change during its freshness lifetime — don't even revalidate on reload |
| `stale-while-revalidate=N` | After expiry, serve stale for up to N seconds while refreshing in the background |
| `stale-if-error=N` | If the origin errors, serve stale for up to N seconds |

> ⚠️ **Gotcha** — `no-cache` does **not** mean "don't cache". It means "cache, but check with the
> origin every time". The directive that means "don't cache" is `no-store`. Using `no-cache` on a
> response containing private data still lets caches *store* it.

**Revalidation** makes `no-cache` and expired entries cheap. The origin sends a validator — an
`ETag` (a version identifier) or `Last-Modified` — and the cache asks "has it changed?" with
`If-None-Match` or `If-Modified-Since`. If not, the origin replies `304 Not Modified` with no body.
That costs a round trip but almost no bandwidth or origin work, *if* the origin can answer the
conditional cheaply (from a stored version number, not by rendering the page and hashing it).

**`stale-while-revalidate`** is the single most useful directive for dynamic-but-shared content:
users always get an immediate response from cache, and the edge refreshes asynchronously. It
removes miss latency from the user path entirely, at the cost of serving content up to N seconds
older than it could be. **`stale-if-error`** turns the CDN into an availability layer: during an
origin outage, users keep getting the last good version.

### `Vary` and why it's dangerous

`Vary` lists request headers whose values change the response, so the cache must store a separate
copy per value. `Vary: Accept-Encoding` (gzip vs Brotli vs none) is normal and cheap — a handful of
values. `Vary: User-Agent` creates a separate copy per browser build — thousands — and destroys the
hit ratio. `Vary: Cookie` effectively disables shared caching. If the response depends on device
type, normalise at the edge to a small derived value (`mobile` / `desktop`) and vary on that.

---

## 6.5 Cache Keys, Invalidation, and How Edges Leak Data

The **cache key** is what a cache uses to decide whether two requests are "the same". By default
it's roughly the host plus path plus query string. Getting it wrong goes two ways:

| Key too specific | Key too broad |
|---|---|
| Identical responses stored many times | Different responses share one entry |
| `?utm_source=` and parameter order fragment the cache | User A's response served to user B |
| Low hit ratio, high origin load | Data leak, wrong language, wrong currency |
| Costly | **Catastrophic** |

Normalising the key — dropping tracking parameters, sorting the rest, lower-casing the host — is
pure win. Adding to it must be deliberate: only request properties the response actually depends
on, in coarse buckets. Section 6.8 shows the policy side; the key side looks like this:

```js
// cache-key.mjs — normalise a request URL into a CDN cache key.
// Requests that must get the same response share one key; requests that might differ never do.
const IGNORED_PARAMS = new Set(['utm_source', 'utm_medium', 'utm_campaign', 'fbclid', 'gclid']);

export function cacheKey(rawUrl, { varyOn = {} } = {}) {
  const url = new URL(rawUrl);
  // Tracking parameters never change the response but would fragment the cache
  const params = [...url.searchParams].filter(([k]) => !IGNORED_PARAMS.has(k.toLowerCase()));
  // Sort so ?a=1&b=2 and ?b=2&a=1 share an entry
  params.sort(([a, av], [b, bv]) => (a === b ? av.localeCompare(bv) : a.localeCompare(b)));
  const query = new URLSearchParams(params).toString();
  // Only explicitly listed, coarse request properties join the key — never a raw User-Agent
  const extras = Object.entries(varyOn).sort().map(([k, v]) => `${k}=${v}`).join(';');
  return `${url.host.toLowerCase()}${url.pathname}${query ? `?${query}` : ''}${extras ? `#${extras}` : ''}`;
}

console.log(cacheKey('https://Shop.example.com/p/42?utm_source=mail&color=red&size=m'));
console.log(cacheKey('https://shop.example.com/p/42?size=m&color=red&gclid=xyz'));
console.log(cacheKey('https://shop.example.com/p/42?size=m&color=red', { varyOn: { device: 'mobile' } }));
// shop.example.com/p/42?color=red&size=m
// shop.example.com/p/42?color=red&size=m
// shop.example.com/p/42?color=red&size=m#device=mobile
```

### Invalidation: avoid needing it

Cache invalidation at a CDN — a **purge** — works, but it's eventually consistent: propagation to
every PoP takes seconds to minutes, purge APIs are rate-limited, and during propagation users see a
mix of old and new. The better strategy is to design URLs so most content never needs purging:

- **Immutable, content-addressed URLs for assets.** `app.3f9a1c2e.js` — the hash of the file is in
  the name. A new build produces new names, so the old file can be cached for a year with
  `immutable`, and "invalidation" is just the HTML referencing a different URL. Chapter 1's
  thumbnail design used the same idea.
- **Short TTLs plus `stale-while-revalidate` for HTML and API reads** — the content refreshes on
  its own within seconds without a purge.
- **Purge by tag (surrogate keys)** for the remaining cases: tag every cached response with the
  entities it contains (`product-42`, `category-7`), and purge the tag when an entity changes.
  This replaces "which URLs show product 42?" — usually unanswerable — with one API call.

### How edges leak private data

The worst failure a CDN can cause is serving one user's private response to another. It happens
in a few specific ways:

- **Personalised responses without `private`.** An account page with `Cache-Control: public` — or
  no header plus a CDN rule that caches by default — is stored and served to the next visitor.
- **Caching responses that set cookies.** A response with `Set-Cookie` cached at the edge hands
  that session cookie to everyone who receives the cached copy.
- **Web cache deception.** A CDN rule says "cache everything ending in `.css`". An attacker sends a
  victim to `/account/settings/x.css`; the application ignores the suffix and renders the
  victim's settings page, which the CDN caches as a public CSS file — readable by the attacker.
  Cache by **response** headers the origin sets deliberately, not by URL extension rules.
- **Cache poisoning via unkeyed inputs.** If the origin uses a header that isn't in the cache key
  — say `X-Forwarded-Host` to build absolute URLs — an attacker sends one request with a malicious
  value, the poisoned response is cached, and every other user gets it. Either include such
  headers in the key or make the origin ignore them.

> ⚠️ **Gotcha** — The safe default is that the **origin** decides cacheability with explicit
> headers on every response, and the CDN is configured to *respect origin headers* rather than to
> apply its own broad "cache by path" rules. Any CDN rule that overrides origin headers is a place
> where a new private endpoint can be accidentally cached.

---

## 6.6 Dynamic Content and the Edge

Not everything is a static file, but much more is cacheable than teams assume.

**Micro-caching** caches dynamic, shared responses for a few seconds. A product page that receives
2,000 requests per second, cached for 5 seconds at each shield, produces about one origin request
per shield per 5 seconds — a reduction of four orders of magnitude, with content at most 5 seconds
stale. For any public read endpoint with bursty traffic, this is the cheapest scaling tool there is.

**Separating the shared from the personal** makes more pages cacheable. A product page is 95%
shared (description, images, price) and 5% personal (cart count, "hello Sam"). Cache the shared
page at the edge, and fetch the personal fragment from the browser with a small private API call
— or assemble the two at the edge.

**Edge compute** — small functions running in PoPs — moves logic closer to users:

| Good fit at the edge | Poor fit at the edge |
|---|---|
| Redirects, rewrites, A/B bucket assignment | Anything needing a transactional database |
| Cache-key normalisation, header cleanup | Long-running or CPU-heavy work |
| Verifying a signed token (stateless) before forwarding | Authorization that needs fresh data from the origin |
| Geo-based routing and localisation | Business logic duplicated from the origin (now two copies drift) |
| Rate limiting and bot filtering (approximate) | Strongly consistent counters or limits |

The constraint is data. Edge functions run in hundreds of places; your source of truth runs in one
or a few. Logic that only needs the request (and maybe a signed token or a slowly changing config
blob) is a good fit; logic that needs a consistent read of mutable data pays a round trip to the
origin anyway and gains little.

### Protecting the origin

A CDN only protects an origin that can't be reached around it. If the origin's address is public
and accepts any traffic, an attacker skips the CDN entirely — no caching, no DDoS absorption, no
WAF. Lock the origin down so it only accepts requests from the CDN: allow-list the CDN's address
ranges, require a secret header or mutual TLS from the CDN, or use a private connection offered by
the provider.

---

## 6.7 Common Mistakes and Best Practices

**Mistake: lowering a TTL at the moment of a migration.** Caches still hold the old record for the
old TTL; lower it at least one old-TTL period in advance.

**Mistake: relying on DNS for fast failover.** Resolver minimums, runtime caching, and pooled
connections make DNS failover take minutes; use anycast or a global load balancer for seconds.

**Mistake: confusing `no-cache` with `no-store`.** `no-cache` allows storage with revalidation;
private data needs `private` or `no-store`.

**Mistake: `Vary: User-Agent` or `Vary: Cookie` on shared content.** It fragments the cache into
thousands of copies, or disables it.

**Mistake: CDN rules that cache by URL pattern.** Extension- or path-based rules cache private
pages under crafted URLs (web cache deception); let the origin's response headers decide.

**Mistake: purging as the primary invalidation strategy.** Purges are slow, rate-limited, and
eventually consistent; immutable URLs and short TTLs avoid most of them.

**Mistake: a publicly reachable origin.** Attackers bypass the CDN's caching, WAF, and DDoS
protection by connecting to the origin directly.

**Best practice: content-hashed asset URLs with year-long `immutable` caching.** Deploys never need
a purge, and HTML is the only thing that must refresh quickly.

**Best practice: `stale-while-revalidate` and `stale-if-error` on shared dynamic content.** Users
never wait on a miss, and an origin outage degrades to slightly stale instead of down.

**Best practice: one origin-side function that sets `Cache-Control` for every response.** A single
reviewed policy, defaulting to conservative, is far safer than headers scattered across handlers.

---

## 6.8 Real-World Example: A Central Cache Policy

The safest way to run a CDN is to have the origin decide cacheability for every response in one
place, with the most dangerous rule first. This policy function is what a middleware would call
just before sending headers.

```js
// cache-policy.mjs — one place that decides how every response may be cached.
// Run: node cache-policy.mjs

const YEAR = 31_536_000;
const HASHED_ASSET = /\.[0-9a-f]{8,}\.(?:js|css|woff2|png|jpe?g|webp|avif|svg)$/i;

export function cachePolicy({ method, path, status, authenticated, setsCookie, kind }) {
  // 1. Never let a shared cache store anything user-specific. This rule runs first because
  //    a mistake here leaks one user's data to another — the worst failure a CDN can cause.
  if (authenticated || setsCookie) {
    return { 'Cache-Control': 'private, no-store' };
  }
  // 2. Only cache safe methods and responses that are stable to replay
  if (method !== 'GET' && method !== 'HEAD') return { 'Cache-Control': 'no-store' };
  if (status >= 500) return { 'Cache-Control': 'no-store' };            // don't pin an outage at the edge
  if (status === 404) return { 'Cache-Control': 'public, max-age=60' }; // absorb scans, recover fast

  // 3. Content-addressed assets: the URL changes when the bytes change, so cache forever
  if (HASHED_ASSET.test(path)) {
    return { 'Cache-Control': `public, max-age=${YEAR}, immutable` };
  }
  // 4. HTML: must reflect deploys quickly, because it's what points at the hashed assets.
  //    Edges keep it briefly and refresh in the background; browsers always revalidate.
  if (kind === 'html') {
    return { 'Cache-Control': 'public, max-age=0, s-maxage=60, stale-while-revalidate=300, stale-if-error=86400' };
  }
  // 5. Public API reads: micro-cache at the edge to collapse bursts on hot keys
  if (kind === 'api') {
    return { 'Cache-Control': 'public, max-age=0, s-maxage=5, stale-while-revalidate=30', Vary: 'Accept-Encoding' };
  }
  // 6. Anything unclassified: be conservative — revalidate every time
  return { 'Cache-Control': 'no-cache' };
}

const cases = [
  { method: 'GET', path: '/assets/app.3f9a1c2e.js', status: 200 },
  { method: 'GET', path: '/products/42', status: 200, kind: 'html' },
  { method: 'GET', path: '/api/products/42', status: 200, kind: 'api' },
  { method: 'GET', path: '/account', status: 200, kind: 'html', authenticated: true },
  { method: 'GET', path: '/api/cart', status: 200, kind: 'api', setsCookie: true },
  { method: 'POST', path: '/api/orders', status: 201, kind: 'api' },
  { method: 'GET', path: '/products/42', status: 503, kind: 'html' },
];
for (const c of cases) console.log(`${c.method} ${c.path} ${c.status}`.padEnd(32), cachePolicy(c)['Cache-Control']);
// GET /assets/app.3f9a1c2e.js 200  public, max-age=31536000, immutable
// GET /products/42 200             public, max-age=0, s-maxage=60, stale-while-revalidate=300, stale-if-error=86400
// GET /api/products/42 200         public, max-age=0, s-maxage=5, stale-while-revalidate=30
// GET /account 200                 private, no-store
// GET /api/cart 200                private, no-store
// POST /api/orders 201             no-store
// GET /products/42 503             no-store
```

The decision to defend is **`max-age=0` for HTML in browsers while the CDN keeps it for 60
seconds.** It's tempting to give HTML a long browser cache too, since it rarely changes. But HTML
is the one resource that tells the browser which hashed asset URLs to load, so a browser holding a
stale copy of it keeps loading the previous release's JavaScript — possibly against an API that has
since changed — and no purge can reach into users' browsers to fix that. Keeping HTML at
`max-age=0` costs a cheap conditional request per navigation; in exchange, a deploy reaches every
user within the CDN's 60-second window, and a bad release can be rolled back just as fast. The
long-lived caching happens where it's safe — on the immutable assets — and the short-lived caching
happens where it can be controlled — at the edge.

---

## 6.9 Questions

**1.** MCQ — A DNS record has a TTL of 3,600 seconds. You change it and lower the TTL to 60 at the same moment. What's the worst-case time for a resolver that cached it just before the change to see the new value? a) 60 seconds b) About an hour c) Instantly d) 24 hours
<details><summary>Answer</summary>**b)** The resolver cached the old record with its old TTL of 3,600 s; the new TTL only applies once it re-fetches.</details>

**2.** MCQ — Which `Cache-Control` directive forbids storing a response anywhere? a) `no-cache` b) `private` c) `no-store` d) `max-age=0`
<details><summary>Answer</summary>**c)** `no-store` prohibits storage. `no-cache` allows storage with revalidation, `private` allows the browser to store it, and `max-age=0` allows storage but makes it immediately stale.</details>

**3.** MCQ — Why can't `example.com` (the zone apex) be a `CNAME` to a CDN hostname? a) CDNs don't support it b) DNS forbids a `CNAME` alongside the apex's required `SOA` and `NS` records c) `CNAME`s only work for IPv6 d) Browsers reject it
<details><summary>Answer</summary>**b)** A `CNAME` can't coexist with other records at the same name, and the apex must have `SOA` and `NS`. Providers offer `ALIAS`/flattening to work around it.</details>

**4.** MCQ — What makes anycast failover faster than DNS failover? a) Anycast uses UDP b) Routing changes propagate via BGP in seconds, with no client caches to expire c) Anycast has lower TTLs d) Anycast only works within one region
<details><summary>Answer</summary>**b)** A failed location withdraws its route and packets go elsewhere; there are no resolver or application caches holding the old answer.</details>

**5.** MCQ — A CDN's request hit ratio is 98% but origin bandwidth is still high. What's the likely explanation? a) The CDN is down b) Large objects are missing the cache, so the byte hit ratio is low c) TTLs are too long d) `Vary: Accept-Encoding` is set
<details><summary>Answer</summary>**b)** Many small hits and a few large misses give a high request hit ratio but a low byte hit ratio — bandwidth follows bytes.</details>

**6.** MCQ — Which header choice most damages a CDN's hit ratio for a public page? a) `Vary: Accept-Encoding` b) `Vary: User-Agent` c) `ETag` d) `s-maxage=60`
<details><summary>Answer</summary>**b)** Thousands of User-Agent strings each get their own cached copy. Normalise to a coarse device class at the edge instead.</details>

**7.** Short — What do `stale-while-revalidate` and `stale-if-error` each buy you?
<details><summary>Answer</summary>`stale-while-revalidate` removes miss latency from users by serving the stale copy while refreshing in the background. `stale-if-error` keeps serving the last good copy when the origin fails, turning an origin outage into slightly stale content.</details>

**8.** Short — Why do content-hashed asset URLs make CDN invalidation unnecessary?
<details><summary>Answer</summary>Any change to the file produces a new URL, so cached copies of the old URL are never wrong — they're just no longer referenced. Assets can be cached for a year with `immutable`, and only the HTML that references them needs to refresh quickly.</details>

**9.** Short — Explain web cache deception.
<details><summary>Answer</summary>A CDN rule caches by URL pattern (e.g. anything ending in `.css`); the application ignores the suffix and renders a private page at `/account/x.css`. A victim lured to that URL causes their private page to be cached publicly, where an attacker can fetch it. Let origin headers, not URL patterns, decide cacheability.</details>

**10.** Short — What problem do origin shields and request collapsing solve?
<details><summary>Answer</summary>Without them, each PoP misses independently and many concurrent requests for an uncached object all reach the origin. Shields funnel PoP misses through one regional cache, and collapsing sends one origin request per object while holding the rest.</details>

**11.** Short — Why must the origin be locked down to accept traffic only from the CDN?
<details><summary>Answer</summary>Otherwise attackers connect to the origin directly, bypassing the CDN's caching, WAF, and DDoS absorption; the CDN protects only the traffic that goes through it.</details>

**12.** Output — What does this print?
```js
const origin = (rps, hit) => Math.round(rps * (1 - hit));
console.log(origin(50_000, 0.96), origin(50_000, 0.99));
```
<details><summary>Answer</summary>`2000 500` — moving from 96% to 99% hit ratio cuts origin load 4×.</details>

**13.** Output — Using the §6.5 `cacheKey`, what does `cacheKey('https://A.com/x?b=2&utm_campaign=z&a=1')` return?
<details><summary>Answer</summary>`a.com/x?a=1&b=2` — the host is lower-cased, the tracking parameter dropped, and the remaining parameters sorted.</details>

**14.** Output — Using the §6.8 `cachePolicy`, what `Cache-Control` value is returned for `{ method: 'GET', path: '/assets/logo.png', status: 200 }`?
<details><summary>Answer</summary>`no-cache` — the filename has no content hash, so it isn't treated as immutable, and with no `kind` it falls through to the conservative default.</details>

**15.** Output — A product page receives 3,000 requests/s spread evenly across 4 origin shields, each of which micro-caches it for 5 seconds with request collapsing. Roughly how many origin requests per second does this page cause?
<details><summary>Answer</summary>**~0.8/s.** Each shield fetches about once per 5 seconds: 4 / 5 = 0.8 requests/s, versus 3,000 without caching.</details>

**16.** Debug — After adding a "cache everything" CDN rule to improve the hit ratio, some users report seeing another person's name in the site header. What happened and how do you fix it?
<details><summary>Answer</summary>Personalised HTML (and possibly `Set-Cookie` responses) is now cached and served to other users. Remove the override, have the origin send `private, no-store` on personalised responses, never cache responses that set cookies, and split shared from personal content so the shared part can be cached safely.</details>

**17.** Debug — A team moved their API to a new load balancer by changing a DNS record with a 300-second TTL. An hour later, 15% of traffic still hits the old load balancer. Why?
<details><summary>Answer</summary>Clients with long-lived pooled connections never re-resolve while connections stay open, some runtimes cache DNS beyond the TTL, and some resolvers clamp TTLs. Keep the old endpoint serving (or proxying) until traffic drains, and cap connection age so clients reconnect and re-resolve.</details>

**18.** Debug — After a release, some users get a blank page with JavaScript errors referencing functions that no longer exist. The CDN serves HTML with `max-age=86400`. Explain.
<details><summary>Answer</summary>Browsers and edges hold yesterday's HTML, which references the previous release's asset bundle or calls APIs whose shape changed. Serve HTML with `max-age=0` (and a short `s-maxage` at the CDN), keep old asset files available for a while after deploys, and keep APIs backward-compatible across one release.</details>

**19.** Debug — An attacker briefly made a site's homepage load scripts from their domain for all users, without compromising the origin. The origin builds absolute URLs from `X-Forwarded-Host`. What happened?
<details><summary>Answer</summary>Cache poisoning via an unkeyed input: the attacker sent a request with a malicious `X-Forwarded-Host`, the origin embedded it, and the CDN cached the response under the normal key for everyone. Ignore or validate forwarded-host headers at the origin (use configured hostnames), or include them in the cache key.</details>

**20.** Code — Write `isStaleServable({ ageSeconds, maxAge, swr, sie, originUp })` returning whether a cache may serve a stored response, given its age, `max-age`, `stale-while-revalidate`, and `stale-if-error` values.
<details><summary>Answer</summary>

```js
function isStaleServable({ ageSeconds, maxAge, swr = 0, sie = 0, originUp = true }) {
  if (ageSeconds <= maxAge) return true;                          // still fresh
  if (ageSeconds <= maxAge + swr) return true;                    // stale, refresh in background
  if (!originUp && ageSeconds <= maxAge + sie) return true;       // origin failing: serve last good
  return false;
}
console.log(isStaleServable({ ageSeconds: 70, maxAge: 60, swr: 30 }),
  isStaleServable({ ageSeconds: 500, maxAge: 60, swr: 30, sie: 86_400, originUp: false })); // true true
```
</details>

**21.** Code — Write `assetUrl(path, contentBuffer)` that returns a content-addressed URL by inserting the first 10 hex characters of the content's SHA-256 before the extension.
<details><summary>Answer</summary>

```js
import { createHash } from 'node:crypto';
function assetUrl(path, content) {
  const hash = createHash('sha256').update(content).digest('hex').slice(0, 10);
  const dot = path.lastIndexOf('.');
  return dot === -1 ? `${path}.${hash}` : `${path.slice(0, dot)}.${hash}${path.slice(dot)}`;
}
console.log(assetUrl('/assets/app.js', Buffer.from('console.log(1)')).replace(/\.[0-9a-f]{10}\./, '.<hash>.')); // /assets/app.<hash>.js
```
</details>

**22.** Code — Write `surrogateKeys(response)` that returns a `Surrogate-Key` header value listing the entity tags for a product page response `{ product: { id, categoryId }, related: [{ id }] }`, so that changing any of those entities can purge the page.
<details><summary>Answer</summary>

```js
function surrogateKeys({ product, related = [] }) {
  const keys = new Set([`product-${product.id}`, `category-${product.categoryId}`]);
  for (const r of related) keys.add(`product-${r.id}`); // related items also appear on the page
  return [...keys].join(' ');
}
console.log(surrogateKeys({ product: { id: 42, categoryId: 7 }, related: [{ id: 9 }, { id: 42 }] }));
// product-42 category-7 product-9
```
</details>

**23.** Scenario — A news site expects a 50× traffic spike when a major story breaks. The article pages are rendered dynamically from a CMS and currently uncached because "articles get corrected". What do you propose?
<details><summary>Answer</summary>Micro-cache article HTML at the edge for 10–30 seconds with `stale-while-revalidate` and request collapsing, so the origin sees a handful of requests per article regardless of traffic; corrections appear within the TTL, or immediately via a surrogate-key purge triggered by the CMS on publish. Add `stale-if-error` so an origin failure under load serves the last good version. Personal elements (login state, paywall meter) move to a small client-side call so the article itself stays shareable.</details>

**24.** Scenario — Your company wants to put its whole API behind edge functions "for performance", including authorization checks that read user permissions from the main database. What's your view?
<details><summary>Answer</summary>Edge functions help for what needs only the request: token signature verification, routing, normalisation, coarse rate limiting, and caching public reads. Authorization that needs fresh permissions requires a round trip to the database's region anyway, so running it at the edge adds a hop and a second copy of security logic that can drift. Recommend verifying tokens and rejecting obviously bad requests at the edge, caching public endpoints there, and keeping data-dependent authorization at the origin, close to the data.</details>

---

## Chapter Summary

- DNS is a hierarchy of caches; changes propagate as TTLs expire — and slower, because resolvers,
  runtimes, and pooled connections often ignore TTLs — so lower TTLs well before a migration.
- DNS routing is simple but coarse and slow to fail over; anycast fails over in seconds, and most
  real systems use an anycast edge that proxies to healthy regions.
- A CDN's value scales with hit ratio, where the last few percent matter most; shields and request
  collapsing stop misses from stampeding the origin.
- `Cache-Control` is precise: `no-cache` revalidates, `no-store` forbids storage, `private` keeps
  data out of shared caches, and `stale-while-revalidate`/`stale-if-error` buy latency and
  availability.
- Content-hashed URLs make most invalidation unnecessary; the remaining content gets short TTLs
  and tag-based purges.
- Edge caching leaks data through personalised responses, cached cookies, URL-pattern rules, and
  unkeyed inputs; let the origin set explicit headers through one central policy, and lock the
  origin to CDN traffic only.

---

## Practice Exercises

1. Use `dig +trace` to follow a lookup for a domain you use from the root down, and note each
   level's TTL.
2. Audit the `Cache-Control` headers on ten responses from a site you work on (`curl -sI`), and
   classify each as correct, too conservative, or dangerous.
3. Read your CDN provider's documentation on how it builds cache keys by default and which request
   headers it ignores, then check for unkeyed headers your origin uses.
4. Measure your CDN's request and byte hit ratios for a week, find the ten largest sources of
   misses, and estimate the origin load saved by fixing each.
5. Implement the HTML/asset split for a small static site: content-hashed asset filenames with
   year-long caching, HTML at `max-age=0`, and a deploy that never needs a purge.

---

## Mini Project: A Caching Reverse Proxy

Build a small caching reverse proxy in Node that sits in front of an origin and implements the
HTTP caching semantics in this chapter. Its point is to make every directive's behaviour something
you can observe.

- Honours `max-age`, `s-maxage`, `private`, `no-store`, `no-cache`, and `Vary` on responses.
- Revalidates with `If-None-Match` and serves `304`s from the origin as cache refreshes.
- Implements `stale-while-revalidate` (background refresh) and `stale-if-error` (serve stale on
  origin failure).
- Collapses concurrent misses for the same key into one origin request.
- Normalises the cache key (sorted query, dropped tracking parameters), refuses to cache responses
  with `Set-Cookie`, and exposes hit/miss/stale counters at `/__stats`.

---

**Next:** [Chapter 7 — Load Balancing →](07-load-balancing.md)
