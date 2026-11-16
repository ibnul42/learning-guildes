# Chapter 40 — Design a Video Streaming Platform

## Learning Objectives

- Scope a video-on-demand platform and quantify upload, storage, and — above all — delivery bandwidth
- Design a transcoding pipeline that splits work into parallel, idempotent tasks
- Explain adaptive bitrate streaming with HLS/DASH segments, and choose a client bitrate strategy
- Treat CDN delivery as the core of the architecture, including cache hierarchy and cost
- Identify what changes for live streaming, and what breaks first at 10×

---

## 40.1 Requirements and Scope

A video platform lets creators upload videos and lets viewers watch them smoothly on any device and network. It's
dominated by two facts: video files are huge and must be converted into many formats, and serving them consumes more
bandwidth than almost anything else on the internet.

**Functional requirements**

- Creators upload videos (minutes to hours long, up to tens of GB).
- The platform processes uploads into streamable formats at several qualities, plus thumbnails.
- Viewers play videos with quick start, seeking, and quality adapting to their network.
- View counts and basic metadata (title, description, channel).

**Out of scope**: recommendations (a ranking system like Chapter 36's), comments, monetisation, DRM details beyond a mention,
search (Chapter 18).

**Non-functional requirements**

| Requirement | Target |
|---|---|
| Playback start | p90 < 2 s to first frame |
| Rebuffering | < 0.5% of watch time spent stalled |
| Upload-to-watchable | Minutes for typical videos (lower qualities first) |
| Availability | 99.95% for playback; uploads may queue |
| Durability | Originals never lost |
| Scale | 500 hours uploaded per minute; 1 B hours watched per day |

---

## 40.2 Estimation

```js
// Video: storage is large, egress is enormous
const uploadHoursPerMin = 500;
const ladderMbps = [0.4, 0.8, 1.5, 3, 5, 8];                 // 240p … 1080p-high renditions
const gbPerHour = (mbps) => (mbps * 3600) / 8 / 1000;         // Mbit/s × seconds → GB
const perHourAllRenditions = ladderMbps.reduce((s, m) => s + gbPerHour(m), 0);
const uploadHoursPerDay = uploadHoursPerMin * 60 * 24;
console.log(`one hour, all renditions: ${perHourAllRenditions.toFixed(1)} GB | new storage per day: ${((uploadHoursPerDay * perHourAllRenditions) / 1e6).toFixed(1)} PB`);
const watchHoursPerDay = 1e9, avgMbps = 3;
const egressPBPerDay = (watchHoursPerDay * gbPerHour(avgMbps)) / 1e6;
console.log(`egress: ${egressPBPerDay.toFixed(0)} PB/day ≈ ${((watchHoursPerDay * 3600 * avgMbps) / 86_400 / 1e6).toFixed(0)} Tbit/s average`);
// one hour, all renditions: 8.4 GB | new storage per day: 6.1 PB
// egress: 1350 PB/day ≈ 125 Tbit/s average
```

Three conclusions:

- **Storage grows by petabytes per day** (plus originals), so storage tiering and deciding which renditions to keep for which
  videos are first-order design choices.
- **Egress is about 125 Tbit/s on average** — far beyond what any origin could serve. Delivery is a CDN problem (Chapter 6),
  and the CDN bill (or the cost of running your own) is the largest cost of the business.
- **Transcoding is CPU-heavy**: every uploaded hour becomes several encodes. Spread across a fleet, it's a large but parallel
  batch workload.

Popularity is extremely skewed: a small fraction of videos accounts for most watch time, and most videos are watched rarely. Both
the CDN and storage designs exploit that.

---

## 40.3 API and Data Model

```text
POST /v1/uploads            { filename, sizeBytes, contentType }   → { uploadId, parts: [presigned URLs] }   (Ch 17)
POST /v1/uploads/:id/complete                                       → { videoId, status: "processing" }
GET  /v1/videos/:id          → { title, duration, status, thumbnails, playback: { hls: ".../master.m3u8" } }
GET  /cdn/v/:id/master.m3u8  → manifest listing renditions → per-rendition playlists → segment URLs
```

Uploads go directly to object storage with multipart, resumable, pre-signed URLs (Chapter 17); the API never handles video bytes.

| Data | Store |
|---|---|
| Video metadata (title, owner, status, duration) | Relational database, sharded by video ID |
| Originals | Object storage, never deleted while the video exists; cold tier after processing |
| Renditions (segments + playlists) | Object storage, content-addressed or versioned paths, served via CDN |
| Processing jobs and tasks | A job store with per-task state (§40.5) |
| View counts | Event pipeline → aggregates (Chapter 25), sharded counters for display (Chapter 12) |

---

## 40.4 High-Level Design

```text
creator ─▶ upload API ─▶ object storage (original) ─▶ "uploaded" event
                                                         │
                         transcoding orchestrator ◀──────┘
                           ├─ probe + split into chunks
                           ├─ encode chunk × rendition tasks  ─▶ worker fleet (CPU/GPU, spot capacity)
                           ├─ assemble + package (HLS/DASH segments, playlists)
                           └─ thumbnails, captions, checks ─▶ object storage (renditions) ─▶ status: ready

viewer ─▶ CDN edge ─(miss)─▶ regional cache/shield ─(miss)─▶ origin (object storage)
   └─▶ player: fetch manifest → pick rendition per segment (ABR) → fetch segments
```

The write side is a **batch pipeline** triggered by uploads; the read side is **static file serving at enormous scale**. They meet only
in object storage: once processed, a video is a set of immutable files, which is what makes it so cacheable.

---

## 40.5 Deep Dive: The Transcoding Pipeline

Transcoding converts the uploaded original into a **bitrate ladder** — several renditions at different resolutions and bitrates —
encoded in widely supported codecs (H.264 for compatibility, plus newer codecs such as VP9, HEVC, or AV1 that cut bitrate for the same
quality at the price of more encoding CPU), and **packaged** into short segments for streaming.

Encoding a two-hour video sequentially at several qualities takes many hours. The pipeline makes it minutes by splitting the work:

```js
// Transcoding fan-out: split the source into chunks and encode chunks × renditions in parallel
function plan({ durationSec, chunkSec = 10, renditions = 6, secPerChunkPerRendition = 30 }) {
  const chunks = Math.ceil(durationSec / chunkSec);
  const tasks = chunks * renditions;
  const serialHours = (tasks * secPerChunkPerRendition) / 3600;
  // With enough workers, wall time ≈ one chunk's encode + split + package overheads
  return { chunks, tasks, serialHours: +serialHours.toFixed(1), parallelMinutes: +((secPerChunkPerRendition + 60) / 60).toFixed(1) };
}
console.log(plan({ durationSec: 2 * 3600 }));
// { chunks: 720, tasks: 4320, serialHours: 36, parallelMinutes: 1.5 }
```

A two-hour film becomes 4,320 independent tasks — 36 CPU-hours of work that can finish in minutes on a large enough fleet. The pipeline
is a **DAG** (directed acyclic graph) of stages, run by an orchestrator (a workflow engine, Chapter 23):

1. **Probe and validate**: read the container, codecs, duration, and resolution; reject corrupt or unsupported files early.
2. **Split** the source into chunks at keyframes, so each can be encoded independently.
3. **Encode** each chunk at each rendition — the fan-out — on a worker fleet, often using cheap interruptible (spot) instances, since
   tasks are short and retryable.
4. **Assemble and package**: concatenate encoded chunks per rendition, cut them into streaming segments, and write playlists.
5. **Side tasks** in parallel: thumbnails, preview sprites, captions, content-safety checks.
6. **Publish**: mark the video ready and its playback URLs live.

Design properties that matter:

- **Idempotent tasks with deterministic outputs.** A task's output path is a function of (video, chunk, rendition, encoder settings), so
  a retried or duplicated task overwrites identical bytes. Spot interruptions, worker crashes, and redelivered queue messages are all
  routine (Chapter 24).
- **Progressive availability.** Encode and publish a low and a medium rendition first so the video is watchable within minutes; higher
  renditions and newer codecs follow.
- **Priorities and fairness.** A creator uploading 500 videos shouldn't block everyone else's single upload: per-creator queues or weighted
  fair scheduling (Chapter 29).
- **Encode effort follows popularity.** Spending extra CPU on expensive codecs and per-title optimisation (choosing ladder rungs per video
  based on its complexity) pays off only for videos that will be watched a lot; do a cheap encode first and re-encode popular videos.

---

## 40.6 Deep Dive: Adaptive Bitrate Streaming

Viewers' networks vary from fibre to a train passing through tunnels. **Adaptive bitrate streaming (ABR)** lets the player change quality
continuously:

- Each rendition is split into **segments** of a few seconds (2–6 s is typical) aligned at the same timestamps across renditions, so the
  player can switch between renditions at any segment boundary.
- A **manifest** (an HLS `.m3u8` master playlist, or a DASH MPD) lists the renditions; per-rendition playlists list segment URLs.
- The **player** downloads segments ahead of playback into a buffer, and before each request decides which rendition to fetch next.

Segments are ordinary immutable files over HTTP — which is why ABR streaming works with ordinary CDNs and caches (Chapter 6), unlike older
streaming protocols that needed stateful media servers.

**The bitrate decision** is where viewing experience is won or lost. The goals conflict: high quality, no stalls (rebuffering), and few
visible quality switches. Two families of algorithms:

| Approach | Decides from | Strength | Weakness |
|---|---|---|---|
| **Throughput-based** | Measured download speed of recent segments | Responsive; simple | Bandwidth estimates are noisy; a sudden drop stalls playback before it reacts |
| **Buffer-based** | How many seconds of video are buffered | Stable; avoids stalls by playing safe when the buffer is low | Can be slow to raise quality at startup |
| **Hybrid** (most production players) | Both, plus predictions | Best of both | More tuning |

The real-world example in §40.10 compares a naive throughput rule with a buffer-based controller on a fluctuating network. Startup is a
special case: the buffer is empty, so players start at a modest rendition (or use a bandwidth hint from the previous session) to reach
first frame fast, then climb.

> ⚠️ **Gotcha** — **Segment duration is a trade-off, not a constant.** Longer segments compress slightly better and mean fewer requests,
> but the player can only switch quality at segment boundaries and must download a whole segment before playing it — so long segments
> slow startup and make adaptation sluggish. Short segments adapt quickly and start fast but multiply requests (and CDN costs per request).
> Live streaming pushes segments shorter still to reduce latency.

---

## 40.7 Deep Dive: Delivery and Cost

At ~125 Tbit/s, delivery *is* the system. Every design choice should maximise how much is served from caches close to viewers.

- **Multi-tier caching.** Edge PoPs serve most requests; misses go to regional mid-tier caches (origin shields), and only their misses reach
  object storage (Chapter 6's tiered caching). Because segments are immutable, they can be cached with long lifetimes and no invalidation.
- **Exploit the skew.** The popular head of the catalogue — new releases, viral videos — lives at the edge everywhere. The long tail is
  served from regional tiers or origin, at higher latency, which viewers of obscure videos tolerate. For the most popular content, push it
  to edges *before* demand (pre-positioning a new season overnight).
- **Caches inside ISPs.** The largest video services place their own cache appliances inside internet service providers' networks, so popular
  bytes travel the shortest possible path and never cross transit links. It's the logical extreme of "serve it before it reaches you".
- **Bytes are the cost.** Better codecs (AV1 can save a large fraction of bits versus H.264 at equal quality), per-title encoding, and avoiding
  waste (don't prefetch far ahead for a viewer who'll leave in 10 seconds) translate directly into money.
- **Storage tiering.** Renditions of videos nobody has watched in a year move to colder storage; originals move to archive tiers after
  processing (and are kept for re-encoding with future codecs).

**Playback security** sits at the edge too: signed URLs or tokens (Chapter 17) restrict who can fetch segments, and premium content adds DRM
(encrypted segments whose keys are issued by a license server after authorization), which doesn't change the caching model because encrypted
segments are still immutable files.

---

## 40.8 Failure Modes, Bottlenecks, and 10×

| Failure | Effect | Mitigation |
|---|---|---|
| Transcoding backlog (spike in uploads) | Slower upload-to-watchable | Autoscale on queue age; progressive renditions; priority for small/new creators |
| Worker interruptions (spot reclaim) | Lost task progress | Short, idempotent tasks retried automatically |
| CDN PoP or provider degradation | Slow starts and rebuffering in a region | Multi-CDN with steering based on real-user measurements; player-side failover to an alternate CDN host |
| Origin overload on a viral video's cold start | Misses stampede the origin | Origin shields and request collapsing (Chapter 6) |
| Corrupt or malicious upload | Pipeline crashes or unsafe content | Validate during probe; sandboxed workers; poison tasks to a DLQ (Chapter 24) |

**Live streaming** changes several parts: ingest from the broadcaster over a contribution protocol (RTMP, SRT, WebRTC) to an ingest point near
them; real-time transcoding into the ladder as the stream arrives; very short segments (or chunked segments delivered while still being written)
to keep glass-to-glass latency to seconds; and CDN behaviour tuned for manifests that change every few seconds. Viral live events concentrate
millions of viewers on the same few segments at once — ideal for caching if request collapsing works, catastrophic for the origin if it doesn't.

**At 10×** (1.25 Pbit/s of delivery): the problem is almost entirely delivery capacity and cost — more ISP-embedded caches, more peering, more
efficient codecs, and smarter pre-positioning. Transcoding scales by adding workers (and is cheaper with hardware encoders); storage growth
forces more aggressive tiering and decisions about which renditions to keep for the long tail (or to regenerate on demand).

---

## 40.9 Common Mistakes and Best Practices

**Mistake: uploading through application servers.** Gigabyte uploads tie up servers for minutes; use direct multipart uploads to object storage.

**Mistake: sequential transcoding.** A two-hour video takes many hours; split into chunks and encode in parallel.

**Mistake: non-idempotent encode tasks.** Retries and spot interruptions produce duplicates or corrupt outputs; make outputs deterministic.

**Mistake: one rendition (or no adaptation).** Viewers on poor networks stall constantly; viewers on good ones get poor quality.

**Mistake: segments that aren't aligned across renditions.** The player can't switch cleanly between qualities.

**Mistake: serving from origin.** Origins can't approach video egress; the CDN hierarchy is the delivery system.

**Mistake: spending the same encoding effort on every video.** Expensive codecs only pay off for videos that will be watched often.

**Best practice: progressive availability.** Publish low and medium renditions first; refine popular videos later.

**Best practice: optimise for rebuffering first.** Viewers forgive lower resolution far more readily than stalls; tune ABR accordingly.

**Best practice: measure from the player.** Startup time, rebuffer ratio, and bitrate from real viewers drive CDN steering and ABR tuning.

---

## 40.10 Real-World Example: A Buffer-Based ABR Controller

The bitrate decision runs in every player before every segment, and it's where most of the viewing experience is decided. This compares a naive
rule — fetch the highest rendition below the last measured throughput — with a buffer-based controller that plays safe when the buffer is low,
maps buffer level onto the ladder otherwise, caps the choice at a smoothed throughput estimate, and climbs one rung at a time. The network trace
mimics a train passing through tunnels.

```js
// abr.mjs — choosing the bitrate of the next video segment. A buffer-based controller picks quality
// from how many seconds of video are buffered; a naive one picks from the last measured throughput.

const LADDER = [0.4, 0.8, 1.5, 3, 5, 8];            // Mbit/s per rendition
const SEGMENT_SEC = 4;

// Buffer-based (in the spirit of BBA): below a reservoir play safe, above a cushion play best,
// in between map buffer level onto the ladder — then apply two stabilisers.
export function bufferBased({ bufferSec, smoothedMbps, prev }, { reservoir = 8, cushion = 24 } = {}) {
  if (bufferSec <= reservoir) return 0;
  const fraction = Math.min(1, (bufferSec - reservoir) / (cushion - reservoir));
  let i = Math.floor(fraction * (LADDER.length - 1));
  while (i > 0 && LADDER[i] > smoothedMbps * 0.9) i--;        // stay under the smoothed link estimate
  if (prev !== null && i > prev + 1) i = prev + 1;              // climb one rung at a time: fewer visible switches
  return i;
}
export function throughputBased({ lastThroughputMbps }) {     // the naive rule: highest rung under last measurement
  let i = 0;
  while (i + 1 < LADDER.length && LADDER[i + 1] <= lastThroughputMbps) i++;
  return i;
}

// Simulate 10 minutes of playback over a network that fluctuates between good and poor
function simulate(choose, trace) {
  let buffer = 0, rebufferSec = 0, bitrateSum = 0, switches = 0, prev = null, last = trace[0];
  const recent = [trace[0]];                                   // throughput samples for smoothing
  for (let s = 0; s < 150; s++) {                              // 150 segments × 4 s = 10 min
    // Harmonic mean of recent samples: dominated by the slow ones, as players need it to be
    const smoothedMbps = recent.length / recent.reduce((a, x) => a + 1 / x, 0);
    const i = choose({ bufferSec: buffer, lastThroughputMbps: last, smoothedMbps, prev });
    const bandwidth = trace[s % trace.length];
    const downloadSec = (LADDER[i] * SEGMENT_SEC) / bandwidth;
    if (s > 0 && downloadSec > buffer) rebufferSec += downloadSec - buffer;  // stall (startup excluded)
    buffer = Math.min(Math.max(0, buffer - downloadSec) + SEGMENT_SEC, 30);  // fetch-ahead capped at 30 s
    last = bandwidth; recent.push(bandwidth); if (recent.length > 5) recent.shift();
    bitrateSum += LADDER[i];
    if (prev !== null && prev !== i) switches++;
    prev = i;
  }
  return `avg ${(bitrateSum / 150).toFixed(2)} Mbit/s | rebuffering ${rebufferSec.toFixed(1)} s | switches ${switches}`;
}

// Bandwidth trace (Mbit/s): mostly decent, with sharp drops — a train through tunnels
const trace = [6, 6, 5, 7, 6, 1.2, 0.9, 6, 7, 5, 6, 0.8, 1.0, 1.1, 6, 6, 7, 6, 5, 6];
console.log('throughput-based:', simulate(throughputBased, trace));
console.log('buffer-based:    ', simulate(bufferBased, trace));
// throughput-based: avg 3.96 Mbit/s | rebuffering 124.3 s | switches 30
// buffer-based:     avg 2.43 Mbit/s | rebuffering 0.0 s | switches 40
```

The naive rule achieves a higher average bitrate — and spends over two minutes of a ten-minute session frozen, because each tunnel arrives right
after it chose a high rendition based on good conditions. The buffer-based controller gives up some average quality and never stalls: whenever the
network drops, it has seconds of video in hand and a conservative estimate of the link.

The decision to defend is **accepting a lower average bitrate in exchange for no rebuffering**, which looks like leaving quality on the table — the
link could carry more on average. But viewers don't experience averages: they experience the moment the picture freezes, and studies of real viewing
consistently find that stalls drive abandonment far more strongly than resolution does. A stall also compounds: by the time it happens, the buffer is
empty and the player must refill it at the worst possible moment. Keeping a buffer and measuring the link pessimistically (a harmonic mean is dragged
down by slow samples) turns network variability into a slightly softer picture instead of a frozen one — and the one-rung-at-a-time climb keeps the
quality changes gradual enough that most viewers never notice them.

---

## 40.11 Questions

**1.** MCQ — Why is a video split into short segments for streaming? a) To compress better b) So the player can switch quality at segment boundaries and segments can be served as ordinary cacheable files c) To prevent piracy d) To reduce storage
<details><summary>Answer</summary>**b)** Aligned segments enable adaptive switching, and immutable files work with any HTTP cache.</details>

**2.** MCQ — What dominates the cost of a large video platform? a) Metadata database b) Transcoding CPU c) Delivery bandwidth (egress/CDN) d) Upload API servers
<details><summary>Answer</summary>**c)** Egress at tens to hundreds of Tbit/s dwarfs everything else.</details>

**3.** MCQ — How does the pipeline make a two-hour video watchable in minutes? a) Faster single encoder b) Splitting into chunks and encoding chunks × renditions in parallel, publishing lower renditions first c) Skipping transcoding d) Streaming the original
<details><summary>Answer</summary>**b)** Parallel fan-out plus progressive availability.</details>

**4.** MCQ — Why must encode tasks be idempotent? a) For compression b) Retries, duplicate queue deliveries, and spot interruptions re-run tasks c) For DRM d) CDNs require it
<details><summary>Answer</summary>**b)** Deterministic output paths make re-runs harmless.</details>

**5.** MCQ — What's the main weakness of throughput-based ABR? a) It's slow to start b) Bandwidth estimates are noisy, and a sudden drop can stall playback before it adapts c) It can't use high bitrates d) It needs server support
<details><summary>Answer</summary>**b)** Buffer-based logic protects against stalls by reacting to how much video is buffered.</details>

**6.** MCQ — Why can video segments be cached with very long lifetimes? a) They're small b) They're immutable once published c) CDNs require it d) They're encrypted
<details><summary>Answer</summary>**b)** A published segment never changes, so there's nothing to invalidate.</details>

**7.** Short — Why do segments need to be aligned across renditions?
<details><summary>Answer</summary>So every rendition has a segment boundary at the same timestamps; the player can then switch to another quality for the next segment without gaps or overlaps.</details>

**8.** Short — What's the trade-off in choosing segment duration?
<details><summary>Answer</summary>Longer segments mean fewer requests and slightly better compression, but slower startup and coarser adaptation; shorter ones adapt and start faster but multiply requests. Live streaming favours shorter segments for latency.</details>

**9.** Short — Why spend more encoding effort on popular videos only?
<details><summary>Answer</summary>Better codecs and per-title optimisation save bandwidth on every view, so the extra CPU pays off in proportion to watch time; for rarely watched videos the CPU cost exceeds the bandwidth saved.</details>

**10.** Short — How does an origin survive a viral video's first minutes?
<details><summary>Answer</summary>Origin shields funnel edge misses through regional caches, and request collapsing sends one origin request per segment while holding concurrent requests, so the origin sees a handful of fetches per segment rather than millions.</details>

**11.** Short — What changes for live streaming compared with VOD?
<details><summary>Answer</summary>Ingest from broadcasters over contribution protocols, real-time transcoding as the stream arrives, much shorter (or chunked) segments for low latency, manifests that update constantly, and huge concurrent audiences on the same segments.</details>

**12.** Output — Using §40.2's `gbPerHour`, how much storage does one hour of a single 5 Mbit/s rendition take?
<details><summary>Answer</summary>**2.25 GB.** 5 Mbit/s × 3,600 s ÷ 8 = 2,250 MB.</details>

**13.** Output — Using §40.5's `plan`, how many tasks does a 45-minute video produce with 6-second chunks and 5 renditions?
<details><summary>Answer</summary>**2,250.** 2,700 s ÷ 6 = 450 chunks × 5 renditions.</details>

**14.** Output — 20 million concurrent viewers watch at an average of 4 Mbit/s. What's the total delivery bandwidth, in Tbit/s?
<details><summary>Answer</summary>**80 Tbit/s.** 20 × 10⁶ × 4 × 10⁶ bit/s = 8 × 10¹³ bit/s.</details>

**15.** Output — Using §40.10's `bufferBased` with default reservoir 8 s and cushion 24 s, what rendition index is chosen for `bufferSec: 16`, `smoothedMbps: 10`, `prev: 4`?
<details><summary>Answer</summary>**2.** Fraction = (16 − 8) / 16 = 0.5; floor(0.5 × 5) = 2 (1.5 Mbit/s), well under the throughput cap, and no climb restriction applies when going down.</details>

**16.** Debug — Upload-to-watchable time grew from 5 minutes to 3 hours after one creator bulk-uploaded 2,000 videos. Cause and fix?
<details><summary>Answer</summary>A single FIFO queue let one creator's backlog delay everyone. Use per-creator queues or weighted fair scheduling, prioritise first renditions of every video over higher renditions of a backlog, and autoscale on queue age.</details>

**17.** Debug — Viewers on mobile networks see frequent stalls while average bitrate looks healthy on dashboards. The player uses the last segment's download speed to choose the next rendition. What do you change?
<details><summary>Answer</summary>The throughput rule overshoots after good samples and stalls when the network drops. Use a buffer-aware hybrid: play safe below a buffer reservoir, use a smoothed (harmonic-mean) throughput estimate as a cap, and climb gradually. Track rebuffer ratio, not just bitrate.</details>

**18.** Debug — After spot-instance reclaims, some videos play with glitches at certain timestamps; logs show some encode tasks ran twice and their outputs were appended to the same file. Cause and fix?
<details><summary>Answer</summary>Non-idempotent outputs: duplicate task runs appended data. Write each task's output to a deterministic path (video, chunk, rendition, settings), overwrite rather than append, and assemble from a manifest of completed task outputs.</details>

**19.** Debug — A new release caused origin storage request rates to spike and origin errors for the first 15 minutes, until edges warmed up. What would prevent it?
<details><summary>Answer</summary>Pre-position the release's segments at edges and shields before launch, enable request collapsing at every tier, and keep origin shields in front of object storage so cold misses from many PoPs become one origin fetch per segment.</details>

**20.** Code — Write `masterPlaylist(renditions)` that produces a minimal HLS master playlist from `[{ name, bandwidth, resolution }]`.
<details><summary>Answer</summary>

```js
function masterPlaylist(renditions) {
  const lines = ['#EXTM3U'];
  for (const r of [...renditions].sort((a, b) => a.bandwidth - b.bandwidth)) {
    lines.push(`#EXT-X-STREAM-INF:BANDWIDTH=${r.bandwidth},RESOLUTION=${r.resolution}`, `${r.name}/index.m3u8`);
  }
  return lines.join('\n');
}
console.log(masterPlaylist([{ name: '720p', bandwidth: 3_000_000, resolution: '1280x720' }, { name: '360p', bandwidth: 800_000, resolution: '640x360' }]));
// #EXTM3U
// #EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
// 360p/index.m3u8
// #EXT-X-STREAM-INF:BANDWIDTH=3000000,RESOLUTION=1280x720
// 720p/index.m3u8
```
</details>

**21.** Code — Write `taskKey({ videoId, chunk, rendition, settingsHash })` returning a deterministic object path, and explain why the settings hash is part of it.
<details><summary>Answer</summary>

```js
const taskKey = ({ videoId, chunk, rendition, settingsHash }) =>
  `renditions/${videoId}/${rendition}/${settingsHash}/chunk-${String(chunk).padStart(5, '0')}.mp4`;
console.log(taskKey({ videoId: 'v9', chunk: 42, rendition: '720p', settingsHash: 'a1b2' })); // renditions/v9/720p/a1b2/chunk-00042.mp4
```
Re-encoding with new settings writes to a new path, so old and new outputs never mix during a re-encode, and the switch happens by publishing a new playlist.</details>

**22.** Code — Write `harmonicMean(samples)` and show why it suits throughput estimation better than the arithmetic mean for `[8, 8, 8, 1]`.
<details><summary>Answer</summary>

```js
const harmonicMean = (xs) => xs.length / xs.reduce((a, x) => a + 1 / x, 0);
const mean = (xs) => xs.reduce((a, x) => a + x, 0) / xs.length;
console.log(harmonicMean([8, 8, 8, 1]).toFixed(2), mean([8, 8, 8, 1]).toFixed(2)); // 2.91 6.25
```
Download time is proportional to 1/throughput, so the harmonic mean predicts average download time; one slow sample pulls it down sharply, which is the conservative behaviour a player wants.</details>

**23.** Scenario — A startup with 10,000 videos and modest traffic asks whether to build its own transcoding pipeline and CDN. What do you advise?
<details><summary>Answer</summary>Buy, don't build: use a managed video service or a cloud transcoding service plus a commercial CDN. The engineering value of a custom pipeline, ISP caches, or per-title encoding appears only at scale, where bandwidth and encoding costs dominate. Keep originals in your own object storage so you can switch providers or re-encode later, and measure player-side quality from day one.</details>

**24.** Scenario — Leadership wants to cut delivery costs by 30% without hurting quality. What levers do you evaluate, in order?
<details><summary>Answer</summary>First, efficient codecs (AV1/HEVC) for the popular head, where most bytes are served, and per-title encoding to drop unnecessary high rungs for simple content. Second, waste: limit prefetch for viewers likely to abandon, cap resolution to the device's screen. Third, delivery economics: more traffic through cheaper paths — ISP-embedded caches and peering, multi-CDN steering by price and performance. Measure rebuffer ratio and startup time throughout, since savings that increase stalls cost more in engagement than they save.</details>

---

## Chapter Summary

- Video platforms are dominated by delivery bandwidth — over a hundred Tbit/s at scale — and by petabytes of daily storage growth, so the CDN
  and storage tiering are the architecture.
- Uploads go directly to object storage; a transcoding DAG probes, splits, encodes chunks × renditions in parallel on idempotent, retryable
  tasks, packages segments, and publishes progressively.
- Adaptive bitrate streaming relies on short segments aligned across renditions and a player that picks each segment's quality; buffer-aware,
  conservatively estimated choices avoid the stalls viewers hate most.
- Immutable segments cache for a long time at every tier; skew lets the popular head live at edges, with shields and request collapsing protecting
  the origin and ISP caches minimising transit.
- Spend encoding effort and codec sophistication where watch time justifies it, and measure quality from real players.
- Live streaming adds real-time ingest and transcoding and shorter segments; at 10× the problem is overwhelmingly delivery capacity and cost.

---

## Practice Exercises

1. Use `ffmpeg` to encode a short video into a three-rung HLS ladder with aligned 4-second segments, and play it in a browser with an HLS player.
2. Extend §40.10's simulation with a startup phase and a bandwidth hint from a previous session, and measure time to first frame for each controller.
3. Read a CDN provider's documentation on video delivery features (tiered caching, request collapsing, signed URLs) and map each to §40.7.
4. Build the chunked transcoding DAG with a local worker pool: split, encode in parallel, assemble, and verify the output plays; kill workers
   mid-run and confirm idempotent recovery.
5. Using a public dataset or a synthetic Zipf distribution of video popularity, estimate what fraction of bytes is served by caching the top 1%,
   5%, and 20% of videos.

---

## Mini Project: A Small Video Pipeline With ABR Playback

Build an end-to-end VOD prototype: upload a video, process it into an adaptive ladder, and play it with a player whose bitrate logic you control.

- Resumable multipart upload to an S3-compatible store (MinIO).
- An orchestrated, idempotent transcoding DAG (probe → split → parallel encode → package → thumbnails), with progressive publishing.
- HLS output served through a local caching proxy (Chapter 6's mini project works), with long-lived cache headers on segments.
- A browser player using an HLS library with a custom ABR rule implementing §40.10's buffer-based logic.
- Player telemetry — startup time, rebuffer ratio, average bitrate, switches — recorded while throttling the network with browser dev tools.

---

**Next:** [Chapter 41 — Design Ride-Hailing and Proximity Search →](41-ride-hailing-proximity.md)
