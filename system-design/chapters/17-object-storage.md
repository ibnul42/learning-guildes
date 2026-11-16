# Chapter 17 — Object Storage and Large Files

## Learning Objectives

- Explain why large binary data belongs in object storage, not in a database or on app servers' disks
- Design direct-to-storage uploads and downloads with pre-signed URLs that can't be abused
- Handle large and unreliable uploads with multipart and resumable protocols
- Use content addressing and content-defined chunking for deduplication and efficient sync
- Keep database metadata and stored objects consistent, and control storage cost with lifecycle rules

---

## 17.1 Bytes Don't Belong in Your Database

Images, videos, documents, backups, exports, model files, and logs are large, written once, read
many times, and rarely modified in place. Putting them in a relational database's `bytea` or `BLOB`
columns works for a prototype and then becomes a problem on every axis:

| Concern | In the database | In object storage |
|---|---|---|
| Cost per GB | Provisioned SSD, ×3 for replicas, plus backups | A few cents per GB-month, redundancy included |
| Buffer pool | Large values evict the hot rows that queries need | Not involved |
| Backups and replicas | Every backup and replica rebuild copies all of it | Separate lifecycle; versioning and replication per bucket |
| Serving | Every download passes through the database and the app | Served directly, or through a CDN |
| Size limits | Practical limits well below a GB per value | Objects of many gigabytes to terabytes |

Storing files on application servers' local disks is worse still: it breaks horizontal scaling
(Chapter 1's avatar bug) and loses files with the instance.

**Object storage** — Amazon S3 and the many S3-compatible services (Google Cloud Storage, Azure Blob
Storage, Cloudflare R2, MinIO, Ceph) — is built for exactly this data. Its model is deliberately
simple:

- **Buckets** contain **objects**, each identified by a **key** (`uploads/u1/9f2c…`) in a flat
  namespace. Slashes are just characters; "folders" are a UI convention over key prefixes.
- Objects are **immutable as a whole**: you `PUT` a new version or `DELETE` it; there's no in-place
  partial update.
- Access is over HTTP: `PUT`, `GET` (including byte ranges), `HEAD` for metadata, `DELETE`, and
  `LIST` by prefix.
- **Durability** is very high — S3 advertises eleven nines per object per year — via replication or
  erasure coding across devices and facilities, and **availability** is high but not as high (around
  99.9–99.99% depending on class).
- **Consistency**: S3 has provided strong read-after-write consistency for all operations since
  2020 — a `GET` after a successful `PUT` returns the new object, and listings reflect it. Some
  S3-compatible stores are weaker; check before relying on it.

The design pattern that follows is the one Chapter 1's thumbnail service used: **bytes in object
storage, metadata in the database**. The database row holds the key, size, content type, checksum,
owner, and status; the object store holds the bytes.

---

## 17.2 Uploads: Don't Proxy the Bytes

The naive upload path streams the file through your application:

```text
client ──(50 MB)──▶ app server ──(50 MB)──▶ object storage
```

Every byte crosses your servers twice, holding a connection and memory for the whole transfer; slow
mobile uploads tie up app capacity for minutes; and app servers need generous request-size limits
that also help attackers. The better path is **direct upload with a pre-signed URL**:

```text
1. client ──"I want to upload a 50 MB PNG"──▶ app ──▶ db: insert upload(status=pending)
2. client ◀── pre-signed PUT URL (key, type, size, expiry) ── app
3. client ──(50 MB)────────────────────────────────────────▶ object storage (verifies signature)
4. client ──"done" (or bucket event) ──▶ app ──HEAD──▶ object storage; verify; status=active
```

A **pre-signed URL** carries a signature, computed by your server with credentials the client never
sees, that authorises one specific operation — `PUT` this key, with this content type, until this
time. The object store verifies the signature and rejects anything else. Your servers handle a few
hundred bytes of JSON per upload; the storage service handles the bytes, at its own scale.

Pre-signed URLs are bearer tokens: anyone holding one can use it until it expires. The design choices
that keep them safe:

| Choice | Why |
|---|---|
| **The server picks the key** (`uploads/<user>/<random id>`) | A client-chosen key can overwrite other objects or collide; user filenames go in metadata, not keys |
| **Short expiry** (minutes) | Limits the damage of a leaked URL |
| **Bind content type, and size where the API allows** (S3's POST policies support a size range; signed PUTs can bind headers and checksums) | Stops a 20 MB "avatar" URL being used for a 5 GB video |
| **Record the upload before issuing the URL** | Every object in the bucket has an owning database row — so nothing is orphaned silently |
| **Verify on completion** (`HEAD` the object: size, type, checksum) | The client's "done" message is a claim, not a fact |
| **Scan before activation** | Malware and file-type sniffing happen before anyone else can download it |
| **Upload into a private bucket or prefix** | Uploaded objects are never publicly readable by default |

> ⚠️ **Gotcha** — **Never trust the content type the client declares.** A file uploaded as
> `image/png` can be HTML with a script in it; if you later serve it from your own domain with the
> declared type (or let the browser sniff it), you've built a stored cross-site-scripting vector.
> Verify the actual file signature server-side, serve user uploads from a separate domain, and set
> `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff` for anything that isn't a
> verified image.

---

## 17.3 Large and Unreliable Uploads

A single `PUT` of a multi-gigabyte file over a mobile connection will fail somewhere in the middle,
and starting over is unacceptable. Two mechanisms fix this.

**Multipart upload.** The client initiates an upload, sends the file as independently uploaded
**parts**, and then completes it with the list of parts. In S3, parts are 5 MB to 5 GB each (the last
may be smaller), with up to 10,000 parts per object. The benefits:

- **Parallelism** — upload 8 parts at once and saturate the link, especially on long fat paths
  (Chapter 5's bandwidth-delay product).
- **Retry granularity** — a failed part is retried alone, not the whole file.
- **Resumability** — the client can ask which parts arrived and continue after a crash or network
  change.

Each part gets its own pre-signed URL, so the multipart flow stays direct-to-storage. Choose the part
size from the file size and the part limit:

```js
// Pick a part size: at least the minimum, few enough parts to stay under the limit,
// and small enough that a failed part is cheap to retry on a bad connection.
function partPlan(fileBytes, { minPart = 5 * 2 ** 20, maxParts = 10_000, preferred = 16 * 2 ** 20 } = {}) {
  const partSize = Math.max(minPart, preferred, Math.ceil(fileBytes / maxParts));
  return { partSize, parts: Math.ceil(fileBytes / partSize) };
}
console.log(partPlan(200 * 2 ** 20));        // { partSize: 16777216, parts: 13 }
console.log(partPlan(500 * 2 ** 30).parts);  // 10000
```

> ⚠️ **Gotcha** — **Abandoned multipart uploads are billed.** Parts uploaded for a multipart upload
> that's never completed or aborted are stored — and charged — indefinitely, and don't appear in
> ordinary object listings. Every bucket that accepts multipart uploads needs a lifecycle rule that
> aborts incomplete uploads after a few days.

**Resumable upload protocols** (tus is the open standard; cloud providers offer their own resumable
sessions) generalise this for clients: the client creates an upload session, sends chunks with
offsets, and on reconnect asks the server for the current offset. They're common in mobile and
browser upload libraries.

---

## 17.4 Downloads and Serving

The same principle applies in reverse: don't stream large downloads through application servers.

- **Public content** (product images, published videos) is served through a **CDN** with the bucket
  as origin, using immutable, content-addressed keys and long cache lifetimes (Chapter 6). Lock the
  bucket so only the CDN can read it (origin access controls), so nobody bypasses the CDN or lists the
  bucket.
- **Private content** (a user's documents, invoices) is served with a **pre-signed GET URL** generated
  after an authorization check, valid for minutes. The app decides *who* may read; the object store
  serves the bytes. For private content that's also hot, CDNs support signed URLs or signed cookies.
- **Range requests** (`Range: bytes=…`) let clients fetch part of an object: video players seek
  without downloading everything, download managers fetch ranges in parallel, and readers of large
  columnar files (Parquet) read only the column chunks they need.

> 💼 **Interview** — In almost every case study that involves media — Chapter 40's video platform,
> a photo app, a file-sharing service — the expected answer includes "uploads go directly to object
> storage via pre-signed URLs, metadata goes in the database, and downloads are served from a CDN
> backed by the bucket". Saying it early saves the interviewer from asking.

---

## 17.5 Content Addressing, Chunking, and Deduplication

A **content-addressed** object is stored under a key derived from a hash of its bytes —
`sha256/9f86d081…`. Three useful properties follow:

- **Deduplication is automatic.** Two users uploading the same file produce the same key; store it
  once and reference it twice.
- **Integrity is checkable.** Re-hashing the downloaded bytes and comparing with the key detects
  corruption anywhere in the pipeline.
- **Immutability is guaranteed.** A key can only ever refer to one content, so caches can keep it
  forever (Chapter 6).

File-sync and backup systems go further and split files into **chunks**, each content-addressed, so
an edit only uploads the chunks that changed. How you split matters enormously. **Fixed-size chunks**
(every 4 KB) fail on insertions: adding ten bytes near the start shifts every later byte, so every
later chunk boundary — and every chunk hash — changes. **Content-defined chunking (CDC)** places
boundaries where a rolling hash of the last few bytes matches a pattern, so boundaries move *with* the
content and an insertion only disturbs the chunks around it:

```js
// Fixed-size vs content-defined chunking: how much survives a small insert?
import { createHash } from 'node:crypto';

const sha = (buf) => createHash('sha256').update(buf).digest('hex');

function fixedChunks(buf, size = 4096) {
  const out = [];
  for (let i = 0; i < buf.length; i += size) out.push(sha(buf.subarray(i, i + size)));
  return out;
}

// Gear-hash content-defined chunking: a boundary is wherever the rolling hash of the last
// few bytes matches a mask, so boundaries move WITH the content instead of sitting at
// fixed offsets. Average chunk ≈ 2^bits bytes; min/max bounds stop pathological sizes.
const GEAR = Array.from({ length: 256 }, (_, i) => createHash('sha256').update(`g${i}`).digest().readUInt32LE(0));
function cdcChunks(buf, { bits = 12, min = 1024, max = 16384 } = {}) {
  const mask = (1 << bits) - 1, out = [];
  let start = 0, h = 0;
  for (let i = 0; i < buf.length; i++) {
    h = ((h << 1) + GEAR[buf[i]]) >>> 0;
    const len = i - start + 1;
    if ((len >= min && (h & mask) === 0) || len >= max) {
      out.push(sha(buf.subarray(start, i + 1)));
      start = i + 1; h = 0;
    }
  }
  if (start < buf.length) out.push(sha(buf.subarray(start)));
  return out;
}

const shared = (a, b) => { const s = new Set(a); return b.filter((x) => s.has(x)).length / b.length; };

// A 1 MB file, then the same file with 10 bytes inserted near the start
let seed = 42;
const original = Buffer.from(Array.from({ length: 1_000_000 }, () => (seed = (seed * 1103515245 + 12345) & 0x7fffffff) & 0xff));
const edited = Buffer.concat([original.subarray(0, 5_000), Buffer.from('HELLO-EDIT'), original.subarray(5_000)]);

console.log(`fixed 4 KB chunks reused after a 10-byte insert: ${(shared(fixedChunks(original), fixedChunks(edited)) * 100).toFixed(1)}%`);
console.log(`content-defined chunks reused:                  ${(shared(cdcChunks(original), cdcChunks(edited)) * 100).toFixed(1)}%`);
// fixed 4 KB chunks reused after a 10-byte insert: 0.4%
// content-defined chunks reused:                  99.0%
```

A ten-byte edit forces fixed chunking to re-upload essentially the whole file; content-defined
chunking re-uploads about 1% of it. This is the technique behind efficient sync and backup tools
(rsync's rolling checksums are a cousin; restic, borg, and many sync services use CDC), and it's why
"upload only what changed" is feasible for large files.

Deduplicated storage needs **reference counting and garbage collection**: a chunk can be deleted only
when no file version references it. Do GC as a background mark-and-sweep against the metadata database
with a grace period, never inline with deletes — a chunk "unreferenced" at the moment of a delete may be
referenced by an upload that's in flight.

---

## 17.6 Metadata, Lifecycle, and Cost

**Metadata lives in the database**, indexed for the queries the product needs: by owner, by status, by
upload time, by content hash. The object store's own metadata (user-defined headers, tags) is useful
for lifecycle rules and debugging, but it isn't queryable and shouldn't be the system of record.

**Storage classes** trade retrieval cost and latency for storage cost:

| Class (S3 names as the example) | Relative storage cost | Retrieval | Use for |
|---|---|---|---|
| Standard | 1× | Immediate, no fee | Hot and recently created data |
| Infrequent access | ~0.5× | Immediate, per-GB retrieval fee; minimum storage duration and object size | Data read a few times a month |
| Archive (Glacier tiers) | ~0.05–0.2× | Minutes to hours; per-GB fee | Compliance archives, old backups |
| Intelligent tiering | Varies | Immediate (for non-archive tiers) | Unknown access patterns, moved automatically |

**Lifecycle rules** move objects between classes and delete them by age or prefix — "move to infrequent
access after 30 days, to archive after 180, delete after 7 years" — and abort incomplete multipart
uploads. For a photo service storing petabytes (Chapter 2's estimate), tiering old, rarely viewed
originals is often the largest cost lever in the whole system.

Costs people miss:

- **Request charges.** Billing is per request as well as per GB. Billions of tiny objects (a few KB
  each) can cost more in `PUT`s and `GET`s than in storage — and minimum-object-size rules in cheaper
  tiers make small objects disproportionately expensive. Batch small records into larger objects
  (log files, Parquet files) instead of one object per record.
- **Egress.** Downloading to the internet costs per GB; serving through a CDN usually reduces it.
- **`LIST` operations.** Listing millions of keys to find something is slow and costs requests; keep an
  index in the database or use the provider's inventory reports.

**Protecting against deletion.** Eleven nines of durability protects against hardware, not against an
engineer deleting the wrong prefix or ransomware with stolen credentials. **Versioning** keeps previous
versions on overwrite and delete; **object lock** (write-once-read-many retention) makes versions
undeletable for a period, even by administrators; and **replication to a bucket in another account**
protects against compromised credentials in the first. Backups (Chapter 32) need all three.

---

## 17.7 Keeping the Database and the Bucket Consistent

An upload touches two systems — the metadata database and the object store — with no transaction
spanning them (Chapter 15's limit). Each ordering has a failure mode:

| Order | Crash between the steps leaves | Visible effect |
|---|---|---|
| Write object, then insert the row | An **orphan object** with no row | Silent storage leak, invisible to the product |
| Insert the row as active, then write the object | A **dangling reference** to a missing object | Broken images, failed downloads |
| Insert the row as *pending*, write the object, mark it *active* | A pending row and maybe an object | Nothing user-visible; a reaper cleans up |

The third ordering is the reliable one. The upload is a small **state machine** —
`pending → active` (or `rejected`, or `abandoned`) — the product only ever shows `active` objects, and a
periodic job reaps pending uploads whose URLs have expired, deleting any object they left behind. Deletes
follow the same idea in reverse: mark the row deleted (so the product stops showing it), delete the object
asynchronously, and retry until it's gone.

A second safety net, for the inevitable bugs: a periodic **reconciliation** job that compares a bucket
inventory listing against the database, reporting objects with no row (orphans) and active rows with no
object (dangling references). Run it, and alert on the counts.

---

## 17.8 Common Mistakes and Best Practices

**Mistake: files in database columns.** Large values bloat backups and replicas, evict hot data from
memory, and push every download through the database.

**Mistake: uploads and downloads proxied through app servers.** App capacity is tied up by slow
transfers; pre-signed URLs let the object store carry the bytes.

**Mistake: client-chosen object keys.** Clients can overwrite or collide with other objects; the server
should generate unguessable keys and keep filenames in metadata.

**Mistake: trusting declared content types.** Uploaded "images" containing HTML or scripts become stored
XSS when served from your domain.

**Mistake: no lifecycle rule for incomplete multipart uploads.** Abandoned parts are stored and billed
forever, invisibly.

**Mistake: one object per tiny record.** Request charges and minimum-size rules make billions of small
objects expensive; batch them into larger files.

**Mistake: activating a row before the object exists, or writing objects with no row.** Either produces
dangling references or orphans; use a pending → active state machine.

**Best practice: bytes in object storage, metadata in the database.** Each system does what it's good at,
and the database row is the source of truth for whether an object is live.

**Best practice: content-addressed keys for immutable content.** Deduplication, integrity checks, and
infinite caching come for free.

**Best practice: versioning, object lock, and cross-account replication for anything you can't lose.**
Durability protects against hardware; these protect against people and stolen credentials.

---

## 17.9 Real-World Example: A Pre-Signed Upload Service

This service implements §17.2's flow and §17.7's state machine: the server picks the key and limits,
records a pending upload *before* issuing the URL, verifies the stored object on completion, and reaps
abandoned uploads. The signer here is a simple HMAC scheme for a storage gateway you control; with S3 or
GCS you'd call the SDK's presigner instead, and the rest of the service stays the same.

```js
// upload-service.mjs — direct-to-object-storage uploads with pre-signed URLs, a server-side
// record of every upload, verification on completion, and cleanup of abandoned uploads.
import { createHmac, randomUUID, timingSafeEqual } from 'node:crypto';

const ALLOWED = { 'image/jpeg': 20e6, 'image/png': 20e6, 'application/pdf': 50e6 }; // type → max bytes

export function createUploadService({ db, objects, signingKey, urlTtlSec = 900, now = () => Date.now() }) {
  const sign = (s) => createHmac('sha256', signingKey).update(s).digest('hex');

  return {
    // 1. Client asks to upload. The server decides the key, limits, and expiry — never the client.
    async requestUpload({ userId, contentType, size }) {
      const max = ALLOWED[contentType];
      if (!max) throw Object.assign(new Error('unsupported type'), { status: 415 });
      if (!(size > 0 && size <= max)) throw Object.assign(new Error('bad size'), { status: 413 });
      const id = randomUUID();
      const key = `uploads/${userId}/${id}`;                 // unguessable; no user-supplied filename
      const expires = Math.floor(now() / 1000) + urlTtlSec;
      // Record intent BEFORE handing out the URL, so every object in the bucket has an owner row
      await db.insert({ id, userId, key, contentType, size, status: 'pending', createdAt: now() });
      const sig = sign(`PUT\n${key}\n${contentType}\n${size}\n${expires}`);
      return { id, url: `https://storage.example.com/${key}?expires=${expires}&sig=${sig}`, method: 'PUT',
        headers: { 'Content-Type': contentType, 'Content-Length': String(size) } };
    },

    // What the storage gateway checks on each PUT (S3 does the equivalent for its own signatures)
    verifySignature({ key, contentType, size, expires, sig }) {
      if (Math.floor(now() / 1000) > expires) return false;
      const expected = Buffer.from(sign(`PUT\n${key}\n${contentType}\n${size}\n${expires}`), 'hex');
      const given = Buffer.from(sig, 'hex');
      return given.length === expected.length && timingSafeEqual(given, expected);
    },

    // 2. Client says "done" (or a bucket event arrives). Trust nothing: check what was stored.
    async completeUpload({ id, userId }) {
      const rec = await db.get(id);
      if (!rec || rec.userId !== userId) throw Object.assign(new Error('not found'), { status: 404 });
      if (rec.status === 'active') return rec;                // idempotent: completing twice is fine
      const head = await objects.head(rec.key);
      if (!head) throw Object.assign(new Error('object not uploaded'), { status: 409 });
      if (head.size !== rec.size || head.contentType !== rec.contentType) {
        await objects.delete(rec.key);                        // mismatched object: reject and remove
        await db.update(id, { status: 'rejected' });
        throw Object.assign(new Error('object does not match request'), { status: 422 });
      }
      // Real systems also scan content (malware, file-type sniffing) before activation
      return db.update(id, { status: 'active', activatedAt: now() });
    },

    // 3. Periodic cleanup: pending uploads past their URL expiry are abandoned
    async reapAbandoned({ graceMs = 3_600_000 } = {}) {
      const cutoff = now() - urlTtlSec * 1000 - graceMs;
      let reaped = 0;
      for (const rec of await db.pendingOlderThan(cutoff)) {
        await objects.delete(rec.key);                        // may or may not exist: delete is idempotent
        await db.update(rec.id, { status: 'abandoned' });
        reaped++;
      }
      return reaped;
    },
  };
}

// ── Demo with in-memory stand-ins for the database and the bucket ──────────────────
const rows = new Map(), bucket = new Map();
const db = {
  async insert(r) { rows.set(r.id, r); }, async get(id) { return rows.get(id); },
  async update(id, patch) { const r = { ...rows.get(id), ...patch }; rows.set(id, r); return r; },
  async pendingOlderThan(t) { return [...rows.values()].filter((r) => r.status === 'pending' && r.createdAt < t); },
};
const objects = {
  async head(k) { return bucket.get(k) ?? null; }, async delete(k) { bucket.delete(k); },
};
let t = Date.UTC(2026, 10, 15);
const svc = createUploadService({ db, objects, signingKey: 'k3y', now: () => t });

const a = await svc.requestUpload({ userId: 'u1', contentType: 'image/png', size: 2_000_000 });
const u = new URL(a.url);
const params = { key: u.pathname.slice(1), contentType: 'image/png', size: 2_000_000,
  expires: Number(u.searchParams.get('expires')), sig: u.searchParams.get('sig') };
console.log('signature valid:', svc.verifySignature(params), '| tampered size:', svc.verifySignature({ ...params, size: 9e9 }));
// signature valid: true | tampered size: false
bucket.set(params.key, { size: 2_000_000, contentType: 'image/png' });   // the client's PUT lands
console.log('completed:', (await svc.completeUpload({ id: a.id, userId: 'u1' })).status); // completed: active

await svc.requestUpload({ userId: 'u1', contentType: 'application/pdf', size: 1e6 }); // never uploaded
t += 3 * 3_600_000;
console.log('reaped:', await svc.reapAbandoned());                                    // reaped: 1
```

The decision worth defending is **inserting the pending row before the client has uploaded anything** —
which means the database fills with rows for uploads that never happen (users who pick a file and then
close the tab), and a reaper has to clean them up. The alternative — creating the row only after the
upload completes — looks tidier but inverts the failure mode: the object exists before anything in your
system knows about it, so every crash, timeout, or abandoned completion call leaves an object nobody owns
and no process will ever find, because the only record of it is in the bucket itself. A pending row is
cheap, visible, and queryable; it turns "objects we might have leaked" into "rows with status pending and
an old timestamp", which a ten-line job can reap. The reaper is the price of making every object
accountable, and it's a small one.

---

## 17.10 Questions

**1.** MCQ — Why are pre-signed URLs preferred over streaming uploads through app servers? a) They're encrypted b) The object store carries the bytes, so app servers handle only small control requests c) They make uploads synchronous d) They avoid needing a database
<details><summary>Answer</summary>**b)** Slow, large transfers no longer consume app-server connections, memory, and bandwidth.</details>

**2.** MCQ — Who should choose the object key for a user upload? a) The client, using the filename b) The server, generating an unguessable key c) The object store, at random d) The CDN
<details><summary>Answer</summary>**b)** Client-chosen keys can overwrite or collide with other objects; filenames belong in metadata.</details>

**3.** MCQ — What's the main benefit of multipart upload for a 4 GB file on a flaky connection? a) Smaller storage cost b) Failed parts can be retried individually and the upload resumed c) Automatic compression d) Stronger consistency
<details><summary>Answer</summary>**b)** Each part is independent; a network failure costs one part, not the whole file — and parts can be uploaded in parallel.</details>

**4.** MCQ — A 10-byte insertion near the start of a large file is synced. Which chunking scheme re-uploads the least? a) Fixed-size chunks b) Content-defined chunks c) One chunk per file d) They're equivalent
<details><summary>Answer</summary>**b)** Content-defined boundaries move with the content, so only chunks near the edit change; fixed boundaries shift every later chunk.</details>

**5.** MCQ — Which ordering avoids both orphaned objects and dangling references when an upload can crash midway? a) Write object, then insert row b) Insert active row, then write object c) Insert pending row, write object, mark active d) Write object and row in parallel
<details><summary>Answer</summary>**c)** Only active rows are visible, and pending rows let a reaper find and clean up anything left behind.</details>

**6.** MCQ — What protects stored objects from an attacker with valid credentials deleting them? a) Eleven nines of durability b) Erasure coding c) Versioning with object lock, and replication to another account d) A CDN
<details><summary>Answer</summary>**c)** Durability protects against hardware loss; deletion protection needs retention locks and copies outside the compromised account.</details>

**7.** Short — Give three reasons large files don't belong in a relational database.
<details><summary>Answer</summary>Any three of: they're expensive on provisioned, replicated SSD; they evict hot rows from the buffer pool; every backup and replica rebuild copies them; downloads pass through the database and app; practical value-size limits.</details>

**8.** Short — What three properties does content addressing give you?
<details><summary>Answer</summary>Automatic deduplication (same content, same key), integrity checking (re-hash and compare with the key), and immutability (a key always means the same bytes, so it can be cached forever).</details>

**9.** Short — Why must a client's "upload complete" message be verified?
<details><summary>Answer</summary>It's a claim from an untrusted party. The server must check that the object exists and matches what was authorised — size, type, checksum — and scan it, before making it visible to anyone else.</details>

**10.** Short — Why is garbage collection of deduplicated chunks done as a background mark-and-sweep with a grace period?
<details><summary>Answer</summary>A chunk unreferenced at the moment of a delete may be referenced by an upload in flight that hasn't recorded its references yet. A delayed sweep against the metadata avoids deleting chunks a concurrent upload is about to use.</details>

**11.** Short — Why can a billion 2 KB objects cost more than their storage suggests?
<details><summary>Answer</summary>Object storage charges per request, so writing and reading each object costs far more than its tiny storage, and cheaper tiers impose minimum object sizes and durations. Batching small records into larger objects avoids both.</details>

**12.** Output — Using §17.3's `partPlan`, what does `partPlan(2 * 2 ** 30)` return?
<details><summary>Answer</summary>**`{ partSize: 16777216, parts: 128 }`.** 2 GiB with the preferred 16 MiB part size is 128 parts, well under the 10,000-part limit.</details>

**13.** Output — What does this print?
```js
const parts = (bytes, part) => Math.ceil(bytes / part);
const MiB = 2 ** 20;
console.log(parts(100 * MiB, 8 * MiB), parts(100 * 1024 * MiB, 8 * MiB));
```
<details><summary>Answer</summary>`13 12800` — a 100 GiB file at 8 MiB parts needs 12,800 parts, exceeding a 10,000-part limit, so the part size must grow for large files (as `partPlan` does).</details>

**14.** Output — A pre-signed URL is issued at 12:00:00 with a 15-minute expiry. The client starts a 30-minute upload at 12:10:00. Does the upload succeed on S3?
<details><summary>Answer</summary>**Generally yes.** The signature is checked when the request begins; a request started before expiry is accepted even if it finishes after. (A *multipart* upload whose later parts start after 12:15 would need fresh URLs for those parts.)</details>

**15.** Output — 300 million objects averaging 4 KB are moved to an infrequent-access class that bills a minimum of 128 KB per object. How much storage is billed, versus the actual data?
<details><summary>Answer</summary>**~38.4 TB billed for ~1.2 TB of data (32×).** 300 M × 128 KB ≈ 38.4 TB; 300 M × 4 KB ≈ 1.2 TB. Small objects should stay in standard storage or be batched.</details>

**16.** Debug — The bucket's storage bill grows 15% a month, but the product's count of active files grows 3%. Listing the bucket shows roughly what the database expects. What's the likely cause, and the fix?
<details><summary>Answer</summary>Incomplete multipart uploads (billed but not listed as objects), and possibly old versions if versioning is on. Add a lifecycle rule that aborts incomplete multipart uploads after a few days and expires noncurrent versions after the retention you need; check with the provider's storage-class and inventory reports.</details>

**17.** Debug — Users occasionally see broken image icons for photos they just uploaded; reloading a minute later fixes it. The app inserts the photo row as visible as soon as the client requests an upload URL. Cause and fix?
<details><summary>Answer</summary>The row is visible before the object exists (and permanently dangling if the upload fails). Insert rows as pending, make them visible only after verifying the uploaded object on completion, and reap pending rows that never complete.</details>

**18.** Debug — A security review finds that an attacker uploaded an "avatar" that executes JavaScript when its URL is opened. Uploads are served from `https://app.example.com/uploads/…` with the client-declared content type. What went wrong, and what changes?
<details><summary>Answer</summary>The file was HTML declared (or sniffed) as an image and served from the app's own origin, so its script runs with the app's cookies — stored XSS. Verify file signatures server-side and re-encode images, serve user content from a separate, cookieless domain (or via pre-signed URLs on the storage domain), and set `Content-Disposition` and `X-Content-Type-Options: nosniff`.</details>

**19.** Debug — Mobile users on poor connections report that uploads of 300 MB videos usually fail and restart from zero. The app uses a single pre-signed `PUT`. What do you change?
<details><summary>Answer</summary>Switch to multipart (or a resumable protocol): split into parts with individual pre-signed URLs, upload a few in parallel, retry failed parts only, and on reconnect query which parts arrived and resume. Persist upload state on the device so an app restart can resume too.</details>

**20.** Code — Write `contentKey(buffer)` returning a content-addressed key `sha256/<hex>` and `verify(buffer, key)` that checks it.
<details><summary>Answer</summary>

```js
import { createHash } from 'node:crypto';
const contentKey = (buf) => `sha256/${createHash('sha256').update(buf).digest('hex')}`;
const verify = (buf, key) => contentKey(buf) === key;
const k = contentKey(Buffer.from('hello'));
console.log(k.slice(0, 15), verify(Buffer.from('hello'), k), verify(Buffer.from('hellO'), k)); // sha256/2cf24dba true false
```
</details>

**21.** Code — Write `missingParts(totalParts, uploaded)` returning the part numbers (1-based) still to upload, given the set of part numbers the server reports as received.
<details><summary>Answer</summary>

```js
function missingParts(totalParts, uploaded) {
  const have = new Set(uploaded);
  return Array.from({ length: totalParts }, (_, i) => i + 1).filter((n) => !have.has(n));
}
console.log(missingParts(8, [1, 2, 4, 5, 8])); // [ 3, 6, 7 ]
```
</details>

**22.** Code — Write `reconcile(dbKeys, bucketKeys)` that returns `{ orphans, dangling }`: bucket keys with no active database row, and active database keys with no object.
<details><summary>Answer</summary>

```js
function reconcile(dbKeys, bucketKeys) {
  const inDb = new Set(dbKeys), inBucket = new Set(bucketKeys);
  return {
    orphans: bucketKeys.filter((k) => !inDb.has(k)),   // storage leak: candidates for deletion after review
    dangling: dbKeys.filter((k) => !inBucket.has(k)),  // broken references: alert, investigate, repair
  };
}
console.log(reconcile(['a', 'b', 'c'], ['b', 'c', 'd'])); // { orphans: [ 'd' ], dangling: [ 'a' ] }
```
At scale, compare a bucket inventory report against a database export rather than listing live.</details>

**23.** Scenario — You're designing document storage for a legal SaaS: files up to 2 GB, must be retained for 10 years, must be provably unaltered, and customers occasionally need old documents within a day. What storage design do you propose?
<details><summary>Answer</summary>Direct multipart uploads to a private bucket via pre-signed URLs; content-addressed keys with checksums recorded in the database for tamper evidence; versioning plus object lock in compliance mode for the retention period; replication to a bucket in a separate account; lifecycle tiering to infrequent access after 90 days and to an archive tier with retrieval within hours after a year (meeting the one-day requirement). Downloads via short-lived pre-signed GET URLs after authorization, with every access audited.</details>

**24.** Scenario — A team wants to store each analytics event (~1 KB, 50,000 per second) as its own object in S3 "because S3 is infinitely scalable". What do you advise?
<details><summary>Answer</summary>50,000 `PUT`s per second is over 4 billion requests a day — a large request bill and per-prefix throughput concerns — and later queries would need billions of `GET`s. Buffer events (in a queue or stream, Chapter 24) and write them in batches as compressed columnar files (e.g. Parquet) every minute or two, partitioned by date and hour, which is cheap to store and efficient for analytics engines to query.</details>

---

## Chapter Summary

- Large binary data belongs in object storage — cheap, durable, HTTP-native — with metadata and status in
  the database, never in database columns or on app servers' disks.
- Upload and download directly between clients and the object store with short-lived pre-signed URLs; the
  server chooses keys and limits, records the upload first, and verifies and scans on completion.
- Multipart and resumable uploads make large transfers parallel and retryable; abandoned parts must be
  aborted by lifecycle rules or they're billed forever.
- Content-addressed keys give deduplication, integrity, and immutability; content-defined chunking makes
  small edits to large files cheap to sync.
- Lifecycle tiering is a major cost lever, while request charges and minimum-size rules punish many tiny
  objects; versioning, object lock, and cross-account replication protect against deletion.
- Keep the bucket and database consistent with a pending → active state machine, a reaper for abandoned
  uploads, and periodic reconciliation.

---

## Practice Exercises

1. Implement pre-signed uploads against a real S3-compatible store (MinIO in a container is fine) using
   the provider's SDK, and verify that a URL can't be reused after expiry or for a different key.
2. Add multipart upload with per-part pre-signed URLs and resume-after-crash to your implementation, and
   test it by killing the client midway.
3. Read your provider's documentation on storage classes and compute the yearly cost of 500 TB under a
   tiering policy of your choice versus keeping everything in standard storage.
4. Use §17.5's chunkers on two versions of a real file you edit often (a document, a database dump) and
   compare how much each would need to upload.
5. Write the reconciliation job from Q22 against a real bucket inventory report and a database export,
   and run it on a test bucket where you've deliberately created orphans and dangling rows.

---

## Mini Project: A Deduplicating File Sync Service

Build a small file-sync service: a client watches a folder and uploads changes; the server stores files
as content-defined chunks in object storage and file manifests in a database, so editing a large file
uploads only the changed chunks.

- Client-side content-defined chunking; the client asks the server which chunk hashes it already has and
  uploads only the missing ones via pre-signed URLs.
- File versions as manifests (ordered lists of chunk hashes) in the database; restore any version.
- Reference counting with a background mark-and-sweep GC and a grace period.
- Resumable uploads that survive a client restart.
- A report showing bytes uploaded per edit versus file size for a set of realistic edits.

---

**Next:** [Chapter 18 — Search →](18-search.md)
