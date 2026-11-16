# Chapter 36 — Design a News Feed

## Learning Objectives

- Scope a social news feed and quantify posting, fan-out, and feed-read load
- Compare fan-out on write, fan-out on read, and the hybrid, using the follower-count distribution
- Store timelines as capped lists of IDs and hydrate them efficiently, filtering deleted and private posts
- Design a two-stage ranking pipeline and keep pagination stable while new posts arrive
- Identify the hot spots — celebrities, viral posts, backfills — and what changes at 10×

---

## 36.1 Requirements and Scope

A news feed shows each user a stream of posts from the accounts they follow — the home timeline of a social network. The
problem is a classic because its difficulty comes entirely from the **shape of the social graph**: most accounts have a few
hundred followers, a few have tens of millions, and every post must reach every follower's feed quickly.

**Functional requirements**

- Users publish posts (text plus media references; media itself goes through Chapter 17's pipeline).
- Users follow and unfollow other users.
- Users load their home feed: posts from followed accounts, newest or ranked first, paginated.

**Out of scope**: comments and likes beyond displaying counts, search, direct messages, notifications (Chapter 38),
ads insertion, content moderation pipelines (mentioned in §36.8).

**Non-functional requirements**

| Requirement | Target |
|---|---|
| Feed load latency | p99 < 300 ms for the first page |
| Freshness | A post appears in followers' feeds within ~5 s (p95); celebrity posts may take slightly longer |
| Availability | 99.95% for feed reads; posting may degrade (queue) rather than fail |
| Consistency | Eventual; an author sees their own post immediately (read-your-writes) |
| Scale | 300 M DAU, global |

---

## 36.2 Estimation

```js
// News feed: the numbers behind the fan-out decision
const dau = 300e6, postsPerUserPerDay = 0.2, avgFollowers = 200, feedLoadsPerUserPerDay = 10, day = 86_400;
const postsPerSec = (dau * postsPerUserPerDay) / day;
const fanoutWritesPerSec = postsPerSec * avgFollowers;
const feedReadsPerSec = (dau * feedLoadsPerUserPerDay) / day;
console.log(`posts ~${Math.round(postsPerSec)}/s | push fan-out ~${Math.round(fanoutWritesPerSec / 1000)}k timeline writes/s | feed loads ~${Math.round(feedReadsPerSec / 1000)}k/s avg`);
// The skew: one celebrity post under pure fan-out-on-write
const celebFollowers = 50e6, writesPerSecPerWorker = 50_000;
console.log(`one celebrity post = ${celebFollowers / 1e6} M timeline writes ≈ ${(celebFollowers / writesPerSecPerWorker / 60).toFixed(0)} worker-minutes`);
// Pure fan-out-on-read: each feed load merges recent posts from everyone you follow
const avgFollowing = 300;
console.log(`pull on every load: ~${Math.round((feedReadsPerSec * avgFollowing) / 1e6)} M post-list reads/s`);
// posts ~694/s | push fan-out ~139k timeline writes/s | feed loads ~35k/s avg
// one celebrity post = 50 M timeline writes ≈ 17 worker-minutes
// pull on every load: ~10 M post-list reads/s
```

The numbers frame the whole design:

- **Posting is cheap** (~700 posts/s, ~2,000 at peak). Storing posts is not the problem.
- **Fan-out on write** turns those posts into ~140,000 timeline insertions per second on average — large but tractable,
  since timeline writes are small appends to in-memory lists.
- **Fan-out on read** would turn ~35,000 feed loads per second into ~10 million post-list reads per second — far worse,
  and every one of them on the latency-sensitive path.
- **The average hides the problem.** A single post by an account with 50 million followers is 50 million timeline writes;
  even with parallel workers that's minutes of work and a huge burst on the timeline store. Celebrities post often, too.

Feeds are read far more than posts are written, and most authors have few followers — so precompute for the common case,
and handle the skewed tail differently (§36.5).

---

## 36.3 API and Data Model

```text
POST   /v1/posts              Idempotency-Key  { text, mediaIds[] }       → 201 { postId, createdAt }
POST   /v1/follows            { targetUserId }                             → 204
DELETE /v1/follows/:targetUserId                                           → 204
GET    /v1/feed?cursor=<opaque>&limit=20                                   → { posts[], nextCursor }
```

The feed endpoint returns **hydrated** posts (author name and avatar, text, media URLs, counts) but internally works with
post IDs until the last step (§36.6). Cursor pagination (Chapter 8) is mandatory — offsets break as new posts arrive at the
top.

| Data | Store | Key design | Access pattern |
|---|---|---|---|
| **Posts** | Wide-column or sharded relational | Partition by `post_id` (time-ordered, Chapter 22); index by `(author_id, created_at)` | Get by ID (batch); recent posts by author |
| **Follow graph** | Sharded relational or wide-column | Two tables: `followers(user_id → follower_id)` and `following(user_id → followee_id)` | Page through an author's followers (fan-out); list whom a user follows (reads, celebrity pull) |
| **Timelines** | In-memory store (Redis-like), sharded by `user_id` | One capped, time-sorted list of post IDs per user | Push an ID; read the top N |
| **Counters** | Sharded counters (Chapter 12) | Per post | Increment; batch read |

Timelines are **derived data** (Chapter 11): rebuildable from posts and the follow graph, so they can live in memory,
be capped at a few hundred entries, and be lost without losing anything permanent. Inactive users' timelines can be
evicted entirely and rebuilt on their next visit.

---

## 36.4 High-Level Design

```text
POST /posts ─▶ post service ─▶ posts DB ──(outbox)──▶ post-created log
                                                         │
                                       fan-out workers ◀─┘ (page followers, push IDs)
                                              │
GET /feed ─▶ feed service ──▶ timeline store (IDs) ───────┘
                 │ ├─▶ following list → celebrity recent posts (pull)
                 │ ├─▶ ranking service (score candidates)
                 │ └─▶ hydration: post cache → posts DB, user cache, counters (batched)
                 └─▶ response
```

The write path is asynchronous: the post service stores the post and returns; fan-out happens from the log, so a slow
fan-out never slows posting, and a backlog delays freshness rather than causing errors. The read path assembles a page
from the precomputed timeline, pulls in posts that weren't pushed, ranks, and hydrates.

---

## 36.5 Deep Dive: Fan-Out on Write, on Read, or Both

| | Fan-out on write (push) | Fan-out on read (pull) |
|---|---|---|
| When a post is made | Insert its ID into every follower's timeline | Do nothing extra |
| When a feed is read | Read one precomputed list | Fetch recent posts from every followed account and merge |
| Read cost | Very low — one list read | High — proportional to accounts followed |
| Write cost | Proportional to followers — 50 M for a celebrity | Constant |
| Freshness | Delayed by fan-out lag | Always current |
| Wasted work | Timelines updated for users who never log in | None |
| Good for | Ordinary authors (most of them) | Accounts with enormous follower counts |

Neither alone works. Pure push melts down on celebrity posts; pure pull makes every feed load expensive. The standard design
is **hybrid**:

- Authors below a follower threshold (say 10,000–100,000, tuned from metrics) are **pushed**: their posts are fanned out to
  followers' timelines.
- Authors above it are **pulled**: their posts aren't fanned out; at read time, the feed service fetches recent posts from the
  celebrities the user follows (usually a small number) and merges them with the precomputed timeline.

The real-world example in §36.10 implements this. Refinements used in practice:

- **Skip inactive followers.** Don't fan out to users who haven't opened the app in weeks; rebuild their timeline on return.
  This often removes a large fraction of fan-out writes.
- **Prioritise the fan-out queue.** Fan out to recently active followers first, so the people most likely to look see the post
  soonest.
- **Cache celebrity recent posts aggressively.** Every follower's read pulls the same few lists; they're perfect cache entries
  (Chapter 16), and they change only when the celebrity posts.
- **Handle threshold crossings.** When an account crosses the threshold, its existing pushed entries stay; new posts switch to
  pull. Moving back below it is the reverse. Because each post takes exactly one path, a post never appears twice.

> 💼 **Interview** — The fan-out trade-off is *the* question in a news-feed interview. Give the push/pull table, the celebrity
> arithmetic (50 M writes for one post), the hybrid with a threshold, and at least one refinement — skipping inactive users is
> the one interviewers most often hope to hear.

---

## 36.6 Deep Dive: Timeline Storage and Hydration

A timeline is a capped, time-sorted list of post IDs per user — a few hundred entries, each a 16-byte ID plus a timestamp. At
300 million users that's on the order of a few terabytes if every user has a full timeline — held in a sharded in-memory store,
partitioned by user ID so a feed read touches one shard. Capping matters: users rarely scroll past a few hundred posts, and
older history can come from a slower path (pulling the followed authors' archives) on the rare occasions it's needed.

**Hydration** turns IDs into what the client renders. A page of 20 posts needs the posts, their authors, media URLs, and
counts — potentially 60+ lookups. Done naively, it's Chapter 5's N+1 problem. Done right:

- **Batch** every lookup type into one multi-get per store (posts by ID, users by ID, counters by post ID), in parallel.
- **Cache** posts and user profiles (Chapter 16) — they're read by many followers and change rarely. Counts can be slightly
  stale.
- **Filter at read time**: posts deleted since being fanned out, posts from accounts the user has since blocked or muted,
  and posts whose visibility changed. Pushing deletions into millions of timelines is expensive and racy; checking a small
  "deleted/hidden" status during hydration is cheap. Over-fetch slightly (ask for 25 IDs to return 20) to absorb filtered
  entries.

**Unfollows** are handled the same way: rather than scrubbing the old author's posts from the timeline, filter by "do I still
follow this author?" at read time, and let the entries age out of the capped list.

**Read-your-writes for the author** (Chapter 13): the author's own new post is inserted into their own timeline synchronously
(or merged in by the feed service from a "my recent posts" lookup), so they see it immediately even if fan-out is lagging.

---

## 36.7 Deep Dive: Ranking

A chronological feed is simple: merge, sort by time, return. Most products rank instead, which changes the read path into a
**two-stage pipeline**:

1. **Candidate generation.** Collect a few hundred to a few thousand candidate posts cheaply: the precomputed timeline, pulled
   celebrity posts, and possibly recommended posts from outside the follow graph.
2. **Ranking.** Score each candidate with a model using features — author affinity (how often the user interacts with this
   author), post engagement so far, recency, content type, the user's recent behaviour — and sort by score. Heavy models run on
   only the top few hundred candidates.

Then **post-processing**: diversity rules (don't show five posts in a row from one author), freshness boosts, and policy filters.

Two engineering consequences:

- **Latency budget.** Feature fetching and model inference fit inside the 300 ms target only if features are precomputed and
  stored in a low-latency feature store (Chapter 25's stream and batch features) and model serving is close to the feed
  service. Budget each stage (Chapter 3), and fall back to chronological order if ranking times out — a degraded feed beats an
  error.
- **Stable pagination.** A ranked feed reorders as scores change, so page 2 computed later could repeat or skip posts from page
  1. Rank once per session (or per first-page request), cache the ranked list of IDs briefly keyed by a session cursor, and
  serve subsequent pages from it; new posts appear on the next refresh, not mid-scroll.

---

## 36.8 Failure Modes, Bottlenecks, and 10×

| Failure | Effect | Mitigation |
|---|---|---|
| Fan-out workers lag (a burst of posts, a backlog after an outage) | Feeds are stale | Lag (time) alert; prioritise active followers; autoscale workers on lag; pulled celebrities unaffected |
| Timeline store shard lost | Those users' timelines empty | Rebuild on read from followed authors' recent posts (slower first load), then repopulate |
| Ranking service slow or down | Feed latency or errors | Timeout and fall back to chronological order |
| Celebrity post during a major event | Huge read load on that post and author | Cache celebrity recent posts and the post itself at every layer; pull path, not push |
| Mass unfollows, deletions, account takedown | Many timelines contain removed content | Read-time filtering; takedown flags checked during hydration |

**Backfills** are a hidden load source: when a user follows someone new, should their past posts appear? Usually a handful of
recent posts are inserted into the follower's timeline (a small, bounded fan-out); when a new user signs up and follows 200
accounts at once, the timeline is built by pulling once rather than by 200 fan-outs.

**At 10×** (3 billion DAU-equivalent load, or 10× posting):

- Fan-out writes reach ~1.4 million per second: shard the timeline store further, lower the celebrity threshold (moving more
  authors to pull), and invest in skipping inactive users.
- Feed reads at ~350,000 per second need more ranking capacity — the ranking tier, not storage, often becomes the largest
  cost.
- The follow graph's largest rows (celebrities' follower lists, tens of millions of entries) need chunked storage and paging
  rather than single-row reads.
- Multi-region becomes necessary for latency: timelines served from the user's region, posts replicated globally, and fan-out
  run in each region for its own users.

---

## 36.9 Common Mistakes and Best Practices

**Mistake: pure fan-out on write.** One celebrity post generates tens of millions of writes and minutes of lag for everyone else's
posts.

**Mistake: pure fan-out on read.** Every feed load merges hundreds of authors' lists — millions of reads per second on the hottest
path.

**Mistake: storing full posts in timelines.** Edits, deletions, and counts become impossible to keep consistent; store IDs and hydrate.

**Mistake: hydrating one post at a time.** Dozens of sequential lookups per page blow the latency budget; batch and parallelise.

**Mistake: pushing deletions to every timeline.** Expensive and racy; filter deleted and hidden posts at read time.

**Mistake: offset pagination on a live, ranked feed.** New posts and re-ranking cause duplicates and gaps between pages.

**Mistake: fanning out to inactive users.** Most of the write work is done for timelines nobody reads.

**Best practice: hybrid fan-out with a tuned threshold.** Push for the many, pull for the few with huge audiences.

**Best practice: timelines as rebuildable, capped, in-memory derived data.** They can be lost, evicted, or rebuilt without data loss.

**Best practice: rank with a timeout and a chronological fallback.** The feed degrades instead of failing.

---

## 36.10 Real-World Example: A Hybrid Fan-Out Feed Service

This is the core of the feed: fan-out on write for ordinary authors (paging through followers in batches, inserting post IDs into
capped timelines idempotently), fan-out on read for authors above a follower threshold, and a read path that merges the two. The
demo has one celebrity with 5,000 followers (above a threshold of 1,000 for the demo) and two ordinary authors.

```js
// hybrid-feed.mjs — fan-out on write for ordinary authors, fan-out on read for celebrities.
// Timelines hold post IDs (not posts), are capped, and are merged with celebrity posts at read time.

export function createFeedService({ store, celebrityThreshold = 10_000, timelineCap = 800, batch = 1_000 }) {
  const stats = { timelineWrites: 0, celebrityPulls: 0 };
  const isCelebrity = (author) => store.followerCount(author) >= celebrityThreshold;

  return {
    stats,
    // Called by a queue consumer after the post is durably stored (outbox → log, Chapter 23)
    async fanOut(post) {
      if (isCelebrity(post.author)) return;                // pulled at read time instead
      let cursor = null;
      do {                                                 // page through followers in batches
        const { ids, next } = store.followersPage(post.author, cursor, batch);
        for (const f of ids) {
          store.pushTimeline(f, post.id, post.ts, timelineCap); // sorted insert + trim; idempotent by post ID
          stats.timelineWrites++;
        }
        cursor = next;
      } while (cursor !== null);
    },

    async readFeed(user, { limit = 20 } = {}) {
      const pushed = store.timeline(user, limit);          // precomputed: [{ id, ts }], newest first
      const celebs = store.following(user).filter(isCelebrity);
      const pulled = celebs.flatMap((c) => { stats.celebrityPulls++; return store.recentPosts(c, limit); });
      // Merge by time; the same post can't appear twice because each author takes one path
      return [...pushed, ...pulled].sort((a, b) => b.ts - a.ts).slice(0, limit).map((p) => p.id);
    },
  };
}

// ── In-memory store for the demo ─────────────────────────────────────────────────────
function memoryStore() {
  const followers = new Map(), following = new Map(), timelines = new Map(), posts = new Map();
  return {
    follow(a, b) { // a follows b
      if (!followers.has(b)) followers.set(b, []); followers.get(b).push(a);
      if (!following.has(a)) following.set(a, []); following.get(a).push(b);
    },
    addPost(p) { if (!posts.has(p.author)) posts.set(p.author, []); posts.get(p.author).unshift(p); },
    followerCount: (u) => followers.get(u)?.length ?? 0,
    followersPage(u, cursor, n) { const all = followers.get(u) ?? [], i = cursor ?? 0;
      return { ids: all.slice(i, i + n), next: i + n < all.length ? i + n : null }; },
    following: (u) => following.get(u) ?? [],
    pushTimeline(u, id, ts, cap) {
      const t = timelines.get(u) ?? [];
      if (!t.some((e) => e.id === id)) { t.push({ id, ts }); t.sort((a, b) => b.ts - a.ts); t.length = Math.min(t.length, cap); }
      timelines.set(u, t);
    },
    timeline: (u, n) => (timelines.get(u) ?? []).slice(0, n),
    recentPosts: (u, n) => (posts.get(u) ?? []).slice(0, n).map(({ id, ts }) => ({ id, ts })),
  };
}

const store = memoryStore();
const feed = createFeedService({ store, celebrityThreshold: 1_000 });
for (let i = 0; i < 5_000; i++) store.follow(`fan${i}`, 'celebrity');   // 5,000 followers: over the threshold
store.follow('ana', 'bo'); store.follow('ana', 'cy'); store.follow('ana', 'celebrity');

let ts = 0;
const publish = async (author, id) => { const p = { id, author, ts: ++ts }; store.addPost(p); await feed.fanOut(p); };
await publish('bo', 'bo-1'); await publish('celebrity', 'celeb-1'); await publish('cy', 'cy-1'); await publish('celebrity', 'celeb-2');

console.log(await feed.readFeed('ana')); // [ 'celeb-2', 'cy-1', 'celeb-1', 'bo-1' ]
console.log(feed.stats);                 // { timelineWrites: 2, celebrityPulls: 1 }
```

Ana's feed interleaves pushed and pulled posts in time order. The two celebrity posts cost zero timeline writes — under pure push
they would have cost 10,002 (all 5,001 followers, twice) — while each read of Ana's feed costs one extra list read for the one
celebrity she follows.

The decision to defend is **choosing push or pull per author by follower count, rather than per post or per reader**. Alternatives
exist — deciding per post by expected engagement, or per reader by how many celebrities they follow — and they can squeeze out
more efficiency. But a per-author rule has a property the others lack: every post from a given author takes exactly one path, so
the merge can never produce duplicates or gaps, and the system's behaviour is predictable from a single number per author that's
easy to monitor and tune. Fan-out cost is dominated by the handful of accounts with enormous followings, so a threshold that moves
those accounts to pull captures most of the benefit with the least machinery. The threshold itself becomes the tuning knob: lower it
when fan-out lag grows, raise it when feed-read latency does.

---

## 36.11 Questions

**1.** MCQ — Why is pure fan-out on write a problem? a) Feed reads are slow b) A celebrity post generates tens of millions of timeline writes c) It loses posts d) It can't paginate
<details><summary>Answer</summary>**b)** Write cost is proportional to followers, which is enormous for a few accounts.</details>

**2.** MCQ — In a hybrid design, which authors are pulled at read time? a) All authors b) Authors with very large follower counts c) Inactive authors d) New authors
<details><summary>Answer</summary>**b)** Their fan-out would be too expensive; readers merge their recent posts instead.</details>

**3.** MCQ — What should a timeline store? a) Full post content b) Post IDs (with timestamps), capped to a few hundred c) Every post ever d) Only celebrity posts
<details><summary>Answer</summary>**b)** IDs keep timelines small and let edits, deletions, and counts be resolved at hydration.</details>

**4.** MCQ — How should deleted posts be kept out of feeds? a) Remove them from every follower's timeline immediately b) Filter them during hydration c) Never delete posts d) Rebuild all timelines
<details><summary>Answer</summary>**b)** Read-time filtering is cheap; pushing deletions to millions of timelines is expensive and racy.</details>

**5.** MCQ — Which refinement most reduces fan-out write volume? a) Bigger timelines b) Skipping inactive followers c) Ranking d) Larger posts
<details><summary>Answer</summary>**b)** Many followers rarely log in; their timelines can be rebuilt when they return.</details>

**6.** MCQ — Why does a ranked feed need special pagination handling? a) Ranking is slow b) Scores change between page requests, causing duplicates or gaps c) IDs aren't sortable d) Cursors don't work with ranking
<details><summary>Answer</summary>**b)** Rank once per session and page through the cached ranked list.</details>

**7.** Short — Why are timelines treated as derived data?
<details><summary>Answer</summary>They can be rebuilt from posts and the follow graph, so they can live in memory, be capped, be evicted for inactive users, and be lost in a failure without losing anything permanent.</details>

**8.** Short — How does the author see their own post immediately when fan-out is asynchronous?
<details><summary>Answer</summary>The post is added to the author's own timeline synchronously (or the feed service merges the author's recent posts for their own feed), giving read-your-writes regardless of fan-out lag.</details>

**9.** Short — Describe two-stage ranking.
<details><summary>Answer</summary>Candidate generation gathers hundreds or thousands of posts cheaply (timeline, pulled celebrities, recommendations); a ranking model scores them with precomputed features and sorts, followed by diversity and policy post-processing.</details>

**10.** Short — What happens when an account crosses the celebrity threshold?
<details><summary>Answer</summary>Posts already fanned out stay in timelines; new posts are pulled instead of pushed. Because each post takes exactly one path, the merge never duplicates or drops posts.</details>

**11.** Short — Why fall back to a chronological feed when ranking times out?
<details><summary>Answer</summary>A slightly less relevant feed is far better than an error or a slow page; the fallback keeps the feed within its latency SLO when the ranking dependency is degraded.</details>

**12.** Output — Using §36.2's formulas with 100 M DAU, 0.5 posts per user per day, and 150 average followers, how many timeline writes per second does pure push require?
<details><summary>Answer</summary>**~86,800/s.** 50 M posts/day ÷ 86,400 ≈ 579 posts/s × 150 ≈ 86,800.</details>

**13.** Output — With §36.10's service and `celebrityThreshold: 1_000`, if `bo` gains 2,000 followers and then posts `bo-2`, how many timeline writes does `bo-2` cause?
<details><summary>Answer</summary>**0.** Bo now has more than 1,000 followers, so his new posts are pulled at read time rather than pushed.</details>

**14.** Output — A user follows 400 accounts, 6 of them above the celebrity threshold. How many list reads does one feed load cost under the hybrid design, versus pure pull?
<details><summary>Answer</summary>**7 versus 400.** One precomputed timeline plus 6 celebrity lists under hybrid; one list per followed account under pure pull.</details>

**15.** Output — 300 M users each have a timeline of 800 entries at 24 bytes per entry (16-byte ID, 8-byte timestamp). Roughly how much memory is that, before overhead and replication?
<details><summary>Answer</summary>**~5.8 TB.** 300 M × 800 × 24 B ≈ 5.76 × 10¹² bytes — a reason to evict inactive users' timelines.</details>

**16.** Debug — During a major sports final, posts from ordinary users took 20 minutes to appear in followers' feeds. Fan-out queue lag spiked whenever a few large sports accounts posted. What's happening and how do you fix it?
<details><summary>Answer</summary>Large accounts are below the celebrity threshold (or there's no hybrid), so their posts flood the fan-out queue and everyone else's posts wait behind them. Lower the threshold so these accounts are pulled, give fan-out a separate queue or priority for small authors, and alert on fan-out lag in seconds.</details>

**17.** Debug — Feed loads have a p99 of 1.8 s. Traces show 60 sequential cache and database calls during hydration. What do you change?
<details><summary>Answer</summary>Batch hydration: one multi-get per store (posts, users, counters) in parallel, with caches in front; over-fetch a few IDs to absorb filtered posts; and set a deadline with partial results for non-critical fields like counts.</details>

**18.** Debug — Users report seeing posts from accounts they blocked hours ago. Timelines were built by fan-out before the block. Why, and what's the fix?
<details><summary>Answer</summary>Timelines contain IDs pushed before the block, and the read path doesn't re-check relationships. Filter during hydration against the user's current block and mute lists (and the author's visibility settings), rather than relying on timeline contents.</details>

**19.** Debug — After a timeline-store shard failed, affected users saw empty feeds for an hour until fan-out gradually repopulated them. What should happen instead?
<details><summary>Answer</summary>A missing or empty timeline should trigger an on-read rebuild: pull recent posts from the user's followed authors (as fan-out on read does), serve the page, and repopulate the timeline in the background — slower first load, but no empty feeds.</details>

**20.** Code — Write `mergeFeeds(lists, limit)` that merges several newest-first lists of `{ id, ts }` into one newest-first list of at most `limit` IDs, dropping duplicate IDs.
<details><summary>Answer</summary>

```js
function mergeFeeds(lists, limit) {
  const seen = new Set(), out = [];
  for (const p of lists.flat().sort((a, b) => b.ts - a.ts)) {
    if (seen.has(p.id)) continue;            // defensive: duplicates can appear after threshold changes
    seen.add(p.id); out.push(p.id);
    if (out.length === limit) break;
  }
  return out;
}
console.log(mergeFeeds([[{ id: 'a', ts: 5 }, { id: 'b', ts: 2 }], [{ id: 'c', ts: 4 }, { id: 'a', ts: 5 }]], 3)); // [ 'a', 'c', 'b' ]
```
A k-way heap merge avoids the full sort when lists are long.</details>

**21.** Code — Write `hydrate(ids, { posts, users, isVisible })` that batch-loads posts and their authors with one call each, filters invisible posts, and preserves order.
<details><summary>Answer</summary>

```js
async function hydrate(ids, { posts, users, isVisible }) {
  const postMap = await posts.getMany(ids);                                   // one batch call
  const authorIds = [...new Set([...postMap.values()].map((p) => p.authorId))];
  const userMap = await users.getMany(authorIds);                            // one batch call
  return ids
    .map((id) => postMap.get(id))
    .filter((p) => p && !p.deleted && isVisible(p))                          // deleted, blocked, hidden
    .map((p) => ({ ...p, author: userMap.get(p.authorId) }));
}
```
</details>

**22.** Code — Write `fanOutTargets({ followerCount, threshold }, followers, { now, activeDays })` returning which followers to push to: none for celebrities, otherwise only followers active in the last 30 days.
<details><summary>Answer</summary>

```js
function fanOutTargets({ followerCount, threshold = 10_000 }, followers, { now = Date.now(), activeDays = 30 } = {}) {
  if (followerCount >= threshold) return [];                       // pulled at read time
  const cutoff = now - activeDays * 86_400_000;
  return followers.filter((f) => f.lastActiveAt >= cutoff).map((f) => f.id); // inactive users rebuild on return
}
console.log(fanOutTargets({ followerCount: 3 }, [{ id: 'a', lastActiveAt: Date.now() }, { id: 'b', lastActiveAt: 0 }])); // [ 'a' ]
```
</details>

**23.** Scenario — Product wants to add "posts liked by people you follow" to the feed. How does this change the design?
<details><summary>Answer</summary>Likes are far more frequent than posts, so fanning out each like to the liker's followers would multiply write volume many times. Treat it as a candidate source at read time instead: for the user's most-interacted-with followees, fetch their recent likes (cached per user), add those posts to the candidate set, and let ranking decide. Bound the number of followees consulted to keep the read cost predictable, and deduplicate against posts already in the timeline.</details>

**24.** Scenario — You're asked to design the feed for an enterprise collaboration tool: 50,000 companies, each with up to 100,000 employees, where people follow channels rather than individuals, and channels can have 100,000 members. Does the same design apply?
<details><summary>Answer</summary>The shape changes: channels are the "authors", and large channels behave like celebrities, so pull (or a per-channel feed merged at read time) is the default for big channels and push only for small ones. Tenant isolation becomes a hard requirement — partition timelines and channel data by tenant, and filter by tenant and permissions at hydration. Freshness expectations are higher (near real-time), so combine the feed with real-time delivery (Chapter 9) for active users.</details>

---

## Chapter Summary

- A news feed's difficulty comes from the follower distribution: posting is cheap, feed reads are frequent, and a few accounts
  have audiences millions of times larger than average.
- Fan-out on write makes reads cheap but celebrity posts catastrophic; fan-out on read makes writes cheap but every read expensive;
  a hybrid with a per-author follower threshold takes the best of both.
- Timelines are capped, in-memory lists of post IDs — derived, rebuildable data — that skip inactive users and are rebuilt on demand.
- Hydrate in batches with caching, and filter deleted, blocked, and hidden posts at read time rather than editing millions of
  timelines.
- Rank in two stages within a strict latency budget, fall back to chronological order on timeout, and paginate from a per-session
  ranked list for stability.
- Watch fan-out lag, treat backfills and timeline loss as rebuild operations, and at 10× lower the threshold, shard further, and go
  multi-region.

---

## Practice Exercises

1. Simulate a follower graph with a power-law distribution of follower counts, and compute fan-out writes per post and per day
   for thresholds of 1,000, 10,000, and 100,000.
2. Implement read-time filtering for deleted posts and blocks in §36.10's read path, with over-fetching, and measure how often a
   page comes back short.
3. Read a public engineering write-up about a large social network's timeline architecture and identify where it uses push, pull,
   and caching.
4. Build batched hydration over a cache and a database and measure feed-page latency with sequential versus batched lookups.
5. Add a ranking stage with a timeout and chronological fallback, and test that feed latency stays within budget when ranking is
   slowed artificially.

---

## Mini Project: A Feed Service With Hybrid Fan-Out

Build a feed service for a small social app with realistic data skew, and show it keeps freshness and latency targets as a celebrity
posts repeatedly.

- Posts and follow graph in PostgreSQL; timelines in Redis (capped, sorted by time); posts published via an outbox to a log.
- Fan-out workers that page followers, skip inactive users, and prioritise recently active ones.
- A hybrid read path that merges pulled celebrity posts, hydrates in batches, filters deletions and blocks, and supports cursor
  pagination.
- Metrics: fan-out lag (seconds), timeline writes per post, feed p50/p99, and cache hit ratios.
- A load test with a Zipf follower distribution and a celebrity posting every 30 seconds, comparing pure push, pure pull, and hybrid.

---

**Next:** [Chapter 37 — Design a Chat System →](37-chat-messaging.md)
