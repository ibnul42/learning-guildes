# Chapter 10 — Storage Engines and Indexes

## Learning Objectives

- Explain how a write-ahead log makes writes durable, and why `fsync` dominates write latency
- Compare B-trees and LSM trees by read, write, and space amplification, and pick one for a workload
- Design composite, covering, and partial indexes from a query's access pattern
- Predict the write cost of each additional index and of random versus sequential primary keys
- Distinguish row-oriented from column-oriented storage and match each to OLTP or analytics

---

## 10.1 Why the Engine Matters to a Designer

Choosing a database is, to a large degree, choosing a **storage engine** — the component that
decides how bytes are laid out on disk, how a write becomes durable, and how a read finds its
data. Two databases with the same query language can behave completely differently under the
same workload because one updates pages in place and the other appends to a log.

You don't need to build an engine to design systems, but you need its cost model. The questions
this chapter answers recur in every data-heavy design:

- Why does this database handle 50,000 writes per second on modest hardware while that one
  struggles at 5,000?
- Why did adding a fifth index halve insert throughput?
- Why are reads on this table slow even though every query uses an index?
- Why did switching the primary key from an auto-increment integer to a random UUID slow inserts
  by 3× once the table outgrew memory?

All of them come down to the same few mechanisms: the log, the page, the tree, and the index.

---

## 10.2 The Log: Durability by Appending

The foundation of almost every durable store is an **append-only log**. To make a write durable,
an engine appends a record describing it to a log file and calls **`fsync`**, which forces the
operating system to push the bytes through its page cache to the storage device and wait for the
device to confirm. Only then is the write acknowledged to the client.

Appending is the cheapest possible disk operation: sequential, with no reading first, on any
device. That's why engines that *do* update data in place (B-trees) still write a log first — the
**write-ahead log (WAL)** — and apply changes to their data pages later. If the process crashes,
recovery replays the log to reconstruct anything that hadn't reached the data pages yet.

```text
client write ──▶ append to WAL ──▶ fsync ──▶ ACK to client
                                      │
                                      └─(later)──▶ apply to data pages ──▶ checkpoint: older WAL can be deleted
```

**`fsync` is the expensive part,** and its cost depends enormously on the device:

| Device | Typical `fsync` latency | Consequence |
|---|---|---|
| Enterprise NVMe with power-loss protection | tens of µs | Device cache is safe, so it can acknowledge early |
| Consumer SSD | ~1 ms or more | Must actually flush its volatile cache |
| Cloud network block storage | ~0.5–2 ms | The write crosses a network to replicated storage |
| Spinning disk | ~5–10 ms | Physical rotation |

At 1 ms per `fsync`, a single writer doing one `fsync` per transaction caps out near 1,000
transactions per second. **Group commit** breaks that ceiling: concurrent transactions that arrive
while one `fsync` is in progress are written together and made durable by the *next* single
`fsync`. Throughput rises with concurrency while each transaction still waits for real durability.
It's Chapter 3's batching trade-off, applied to the disk.

Some systems let you trade durability for latency explicitly — PostgreSQL's
`synchronous_commit = off`, MongoDB's journal settings, Redis's `appendfsync everysec` — which
acknowledge before `fsync` and accept losing the last fraction of a second on a crash. That's a
legitimate choice for some data and a disaster for others; make it per table or per operation, on
purpose.

> ⚠️ **Gotcha** — A write that reached the OS but not the device is not durable. `write()`
> returning successfully means the bytes are in the page cache; a power loss erases them. Engines
> that skip `fsync` "for speed" — or run on storage that lies about flushes — pass every test and
> lose acknowledged data the first time a machine loses power.

### The simplest log-structured store

If the log *is* the database — every value lives in the log, and an in-memory hash table maps each
key to the byte offset of its latest record — you get a remarkably fast store. Writes are appends;
reads are one hash lookup plus one disk read. Old versions of a key accumulate in the log and are
removed by **compaction**: copy live records to a new file, then swap it in. This design (Bitcask)
is the real-world example in §10.9. Its limitation is that **every key must fit in memory**, and
range queries are impossible because the hash has no order. The two engines below remove those
limits in different ways.

---

## 10.3 B-Trees: Update in Place

The **B-tree** (in practice, the B+tree variant) has been the default index structure of relational
databases for fifty years. Data is organised in fixed-size **pages** — 8 KB in PostgreSQL, 16 KB in
MySQL's InnoDB — arranged as a balanced tree. Interior pages hold keys and pointers to child pages;
leaf pages hold the keys with their values (or pointers to rows), linked in key order so range
scans can walk leaves sequentially.

```text
                      [ 100 | 500 ]                          root
                     /      |      \
          [ 20 | 60 ]  [ 200 | 350 ]  [ 700 | 900 ]          interior
          /   |   \     /    |    \     /    |    \
        leaf  leaf leaf leaf leaf leaf leaf leaf leaf ─▶ ─▶  leaves, linked for range scans
```

The key property is **high fanout**: an 8 KB page holds hundreds of keys, so the tree is very
shallow.

```js
// How many page reads to find one row among N? Height ≈ log_fanout(N).
const height = (rows, fanout) => Math.ceil(Math.log(rows) / Math.log(fanout));
for (const rows of [1e6, 1e9, 1e12]) console.log(`${rows.toExponential()} rows → ${height(rows, 500)} levels`);
// 1e+6 rows → 3 levels
// 1e+9 rows → 4 levels
// 1e+12 rows → 5 levels
```

A billion rows need four levels. The root and interior pages are small and hot, so they stay in the
database's memory (the **buffer pool**); a point lookup typically costs **one disk read** for the
leaf, or none if it's cached. That's why indexed lookups stay fast as tables grow, and why "does the
working set fit in RAM?" is the most important capacity question for a B-tree database.

**Writes update pages in place.** To insert a row, the engine finds the right leaf, modifies it in
the buffer pool, logs the change in the WAL, and writes the dirty page back to disk later. When a
leaf fills, it **splits** into two half-full pages and the parent gains a pointer — which can
cascade upward. Writing whole pages is the B-tree's main cost: changing a 100-byte row eventually
writes an 8 KB page (plus the WAL record), and engines add protection against **torn pages**
(half-written pages after a crash) — PostgreSQL logs a full page image after each checkpoint;
InnoDB uses a doublewrite buffer — which costs more writes still.

> ⚠️ **Gotcha** — **Random primary keys hurt B-trees.** With an auto-increment or time-ordered key,
> every insert lands on the rightmost leaf, which stays hot in memory. With a random UUIDv4, inserts
> land on random leaves across the whole index; once the index outgrows memory, nearly every insert
> must read a leaf from disk first, pages split everywhere, and they end up about half full. Insert
> throughput can drop several-fold and the index grows larger than necessary. Use time-ordered IDs
> (UUIDv7, Snowflake-style — Chapter 22) when you want unique IDs without central coordination.

---

## 10.4 LSM Trees: Never Update in Place

The **log-structured merge tree (LSM)** takes the log idea further: *nothing* is updated in place.

```text
write ──▶ WAL (durability) + memtable (sorted, in memory)
                                │ full (e.g. 64 MB)
                                ▼
                       flush as an immutable sorted file (SSTable) ──▶ L0
                                                                        │ compaction
                                                         L1  [ sorted, non-overlapping files ]
                                                         L2  [ 10× larger ]
                                                         L3  [ 10× larger again ]
```

1. A write goes to the WAL (for durability) and to the **memtable**, an in-memory sorted structure.
2. When the memtable fills, it's written out sequentially as an immutable, sorted file — a
   **sorted string table (SSTable)**.
3. Background **compaction** merges SSTables, discarding overwritten values and deleted keys, so the
   number of files stays bounded.
4. A **read** checks the memtable, then SSTables from newest to oldest, returning the first match.
   Deletes are written as **tombstones** — markers saying "this key is deleted" — that shadow older
   values until compaction removes both.

All disk writes are large and sequential: memtable flushes and compaction output. That's why LSM
engines sustain very high write throughput, and why they're used in write-heavy stores — RocksDB
(and systems built on it), Cassandra, ScyllaDB, HBase, and the storage layers of CockroachDB and
TiDB.

The cost moves to reads. A key could be in any of several files, so naively a read checks each one.
Two mechanisms fix that:

- **Each SSTable has a sparse index** (one entry per block), so checking a file is a binary search
  in memory plus at most one block read.
- **Each SSTable has a Bloom filter** — a compact bit array that answers "this key is *definitely
  not* in this file" or "it *might* be". With ~10 bits per key, false positives are about 1%, so a
  point read skips almost every file that doesn't contain the key.

```js
// bloom.mjs — "definitely not present" or "probably present", in a few bits per key.
import { createHash } from 'node:crypto';

export class BloomFilter {
  constructor(expectedKeys, falsePositiveRate = 0.01) {
    // Optimal sizing: m = −n·ln(p)/ln(2)², k = (m/n)·ln(2)
    this.m = Math.ceil((-expectedKeys * Math.log(falsePositiveRate)) / Math.LN2 ** 2);
    this.k = Math.max(1, Math.round((this.m / expectedKeys) * Math.LN2));
    this.bits = new Uint8Array(Math.ceil(this.m / 8));
  }
  // Double hashing: two hash values generate all k positions (Kirsch–Mitzenmacher)
  #positions(key) {
    const h = createHash('sha256').update(key).digest();
    const h1 = h.readUInt32LE(0), h2 = (h.readUInt32LE(4) | 1) >>> 0; // odd, and unsigned
    return Array.from({ length: this.k }, (_, i) => (h1 + i * h2) % this.m);
  }
  add(key) { for (const p of this.#positions(key)) this.bits[p >> 3] |= 1 << (p & 7); }
  mightContain(key) { return this.#positions(key).every((p) => this.bits[p >> 3] & (1 << (p & 7))); }
}

const bf = new BloomFilter(100_000, 0.01);
for (let i = 0; i < 100_000; i++) bf.add(`key:${i}`);
let falsePositives = 0;
for (let i = 100_000; i < 200_000; i++) if (bf.mightContain(`key:${i}`)) falsePositives++;
console.log(`${(bf.m / 100_000).toFixed(1)} bits/key, k=${bf.k}, present key found: ${bf.mightContain('key:42')}`);
console.log(`false positive rate: ${(falsePositives / 1000).toFixed(2)}%`);
// 9.6 bits/key, k=7, present key found: true
// false positive rate: 1.03%
```

Bloom filters never give false negatives, which is what makes them safe to skip files with. They
can't help **range scans** (every file may hold keys in the range) or reads of keys that *do* exist
in many versions, which is why LSM range reads are more expensive than B-tree ones.

### Compaction strategy is a workload decision

| | Leveled compaction (RocksDB default) | Size-tiered compaction (Cassandra's classic default) |
|---|---|---|
| How | Each level is a set of non-overlapping files ~10× the previous level | Merge several similarly sized files into one larger file |
| Write amplification | Higher (data rewritten once per level, often 10–30× total) | Lower |
| Read amplification | Low — roughly one file per level | Higher — many overlapping files |
| Space amplification | Low (~10%) | High — up to 2× during large merges |
| Suits | Read-heavy or mixed workloads, limited disk | Write-heavy, read-light workloads |

> ⚠️ **Gotcha** — **Tombstones are writes.** Deleting a million rows in an LSM store writes a
> million tombstones, and reads that scan the deleted range must step over every one of them until
> compaction removes them — which, by design, waits for the tombstones to propagate. Queue-like
> access patterns (insert, read once, delete) on LSM stores are a classic cause of reads that get
> slower every day. Use TTL-based expiry that drops whole files, or a store designed for queues
> (Chapter 24).

---

## 10.5 Amplification: Comparing the Two Honestly

Every engine pays for its writes and reads in extra work. Three ratios capture it:

- **Write amplification** — bytes written to disk ÷ bytes written by the application.
- **Read amplification** — disk reads (or bytes read) per logical read.
- **Space amplification** — bytes on disk ÷ bytes of live data.

| | B-tree | LSM tree |
|---|---|---|
| Write amplification | Whole pages per small change, plus WAL, plus torn-page protection | Compaction rewrites data several times; but all writes are sequential |
| Write throughput | Good; limited by random page writes once data exceeds RAM | Excellent; sequential, batched |
| Point reads | 1 leaf read (interior pages cached) | Memtable + Bloom-filtered files; usually ~1 read |
| Range scans | Excellent — walk linked leaves | Merge across levels — more expensive |
| Space | Pages partly empty after splits (fill factor) | Obsolete versions until compaction; tiered can be ~2× |
| Latency predictability | Steady | Compaction competes for I/O; can cause latency spikes |
| Typical engines | PostgreSQL, MySQL/InnoDB, SQL Server, Oracle, SQLite | RocksDB, Cassandra, ScyllaDB, HBase, Pebble (CockroachDB), TiKV |

The honest summary: **B-trees are the better default for read-heavy, mixed transactional workloads
with range queries; LSM trees win for write-heavy workloads, especially on large datasets and on
storage where sequential writes are much cheaper than random ones.** The gap has narrowed on fast
NVMe, and modern engines borrow from each other — B-trees with log-structured buffering, LSM trees
with aggressive caching — so measure with your own access pattern before committing.

> 💼 **Interview** — "Why does Cassandra handle writes so well?" Answer with the LSM write path:
> every write is a WAL append plus a memtable insert, with no read-before-write and no random I/O,
> and durability comes from replication plus the commit log. Then name the cost: reads may touch
> several SSTables, mitigated by Bloom filters, and deletes create tombstones.

---

## 10.6 Indexes: Fast Reads, Paid for by Every Write

A table without indexes answers every query by scanning every row. An **index** is a separate,
sorted structure (usually a B-tree) that maps column values to rows, so a query can seek directly
to what it needs. Each index is a partial copy of the table, kept in sync on every write.

### Clustered and secondary indexes

In **InnoDB** (and SQL Server's clustered tables), rows are stored *inside* the primary-key B-tree,
in primary-key order — the table *is* the clustered index. Secondary indexes store the indexed
columns plus the primary key, so a lookup by a secondary index costs two tree descents: find the
primary key, then find the row. In **PostgreSQL**, rows live in an unordered heap and every index
(including the primary key's) points to a row's physical location. Both models make the same
point: the primary key's shape affects every index.

### Composite indexes and the leftmost-prefix rule

An index on `(a, b, c)` is sorted by `a`, then by `b` within equal `a`, then by `c`. It can serve
queries that constrain a **leftmost prefix** of its columns:

```sql
CREATE INDEX orders_by_customer ON orders (customer_id, status, created_at);

-- ✅ Uses the index fully: equality on the first two columns, range on the third
SELECT * FROM orders WHERE customer_id = 42 AND status = 'open' AND created_at > now() - interval '7 days';

-- ✅ Uses the index's first column: all of customer 42's orders, in (status, created_at) order
SELECT * FROM orders WHERE customer_id = 42;

-- ❌ Can't seek: status is not a leftmost prefix, so this is a scan (of the index or the table)
SELECT * FROM orders WHERE status = 'open';
```

The rule for ordering columns: **equality conditions first, then the range or sort column.** A range
condition in the middle stops the index from narrowing on the columns after it.

### Covering, partial, and expression indexes

- **Covering index** — includes every column the query needs, so the database answers from the
  index alone without touching the table (an "index-only scan"). In PostgreSQL:
  `CREATE INDEX … ON orders (customer_id, created_at) INCLUDE (total);`
- **Partial index** — indexes only rows matching a condition:
  `CREATE INDEX … ON orders (created_at) WHERE status = 'open';` — small and fast when queries
  always target the minority of open orders.
- **Expression index** — indexes a computed value: `CREATE INDEX … ON users (lower(email));` makes
  case-insensitive lookups seekable.

### What an index costs

Every insert writes the row *and* an entry in every index; every update to an indexed column
updates that index; every index competes for buffer-pool memory. A table with eight indexes does
roughly nine B-tree writes per insert. Indexes also need maintenance (bloat, vacuum, rebuilds) and
make schema changes slower. So:

| Add an index when | Don't, when |
|---|---|
| A frequent query filters or sorts on the columns and is slow | The column has very low selectivity (a boolean on a large table — unless a partial index fits) |
| It supports a foreign key used in joins or cascading deletes | It duplicates a leftmost prefix of an existing composite index |
| It enforces a uniqueness invariant (`UNIQUE`) | The table is write-heavy and the query is rare or offline |

Use the query planner (`EXPLAIN ANALYZE`) to see whether a query actually uses an index and how many
rows it reads. "There's an index on that column" and "this query uses it efficiently" are different
claims.

---

## 10.7 Row Stores and Column Stores

Everything so far has been **row-oriented**: all columns of a row are stored together, which is
ideal for transactional workloads (OLTP) that read or write whole rows by key.

Analytics (OLAP) has the opposite shape: scan billions of rows but read only a few columns
(`SELECT sum(amount) FROM sales WHERE region = 'EU' GROUP BY month`). A **column store** keeps each
column's values together:

| | Row-oriented | Column-oriented |
|---|---|---|
| Layout | Row 1 (all columns), row 2 (all columns), … | Column A (all rows), column B (all rows), … |
| Read one row by key | One page | One read per column |
| Scan 3 of 50 columns over 1 B rows | Reads all 50 columns | Reads only the 3 |
| Compression | Moderate (mixed types per page) | Excellent — similar values together (run-length, dictionary, delta) |
| Single-row writes | Cheap | Expensive; batched writes preferred |
| Examples | PostgreSQL, MySQL, most OLTP | ClickHouse, BigQuery, Snowflake, Redshift, DuckDB; Parquet files |

Column stores often compress data 5–10× and scan only the needed columns, so analytical queries
read a small fraction of the bytes a row store would — the difference between seconds and hours.
The design consequence is a standard split: **transactional data lives in a row store; analytical
copies flow into a column store** via change data capture or batch pipelines (Chapter 25), and
analytics never runs against the production OLTP database.

---

## 10.8 Common Mistakes and Best Practices

**Mistake: trusting `write()` as durability.** Without `fsync` — on storage that honours it — an
acknowledged write disappears on power loss.

**Mistake: random UUIDv4 primary keys on large B-tree tables.** Inserts scatter across the index,
causing page reads, splits, and half-empty pages once it outgrows memory.

**Mistake: indexing every column "just in case".** Each index adds a write per insert and competes
for cache; unused indexes are pure cost.

**Mistake: composite indexes in the wrong column order.** A range column before an equality column
stops the index from narrowing; put equality columns first.

**Mistake: delete-heavy queue patterns on LSM stores.** Tombstones accumulate and scans step over
them, making reads slower every day.

**Mistake: running analytics on the OLTP row store.** Wide scans evict the transactional working set
from cache and compete for I/O with production traffic.

**Mistake: assuming an index is used.** The planner may choose a scan; verify with `EXPLAIN ANALYZE`
on production-shaped data.

**Best practice: keep the working set in memory.** For B-tree databases, the ratio of hot data and
indexes to RAM predicts performance better than any other number.

**Best practice: choose the engine by write/read mix and access pattern.** B-trees for mixed
transactional workloads with range queries; LSM for write-heavy, large datasets.

**Best practice: covering and partial indexes for the hottest queries.** Index-only scans on a small,
targeted index are often the cheapest large win available.

---

## 10.9 Real-World Example: A Log-Structured Key-Value Store

This is the Bitcask design from §10.2: a single append-only data file, an in-memory index from key
to record offset, checksummed records so recovery can detect a torn final write, tombstones for
deletes, and compaction that rewrites only live records and swaps the file in atomically.

```js
// logkv.mjs — a log-structured key-value store (the Bitcask design): every write is an
// append to a data file; an in-memory hash maps each key to the offset of its latest value.
import { openSync, writeSync, readSync, fsyncSync, closeSync, fstatSync, ftruncateSync,
  renameSync, existsSync, rmSync, statSync } from 'node:fs';
import { crc32 } from 'node:zlib';

// Record layout: [crc32 u32][keyLen u32][valLen i32 (-1 = tombstone)][key][value]
const HEADER = 12;

export class LogKV {
  #fd; #path; #index = new Map(); // key → { offset, length } of the whole record
  #size = 0;

  constructor(path, { sync = true } = {}) {
    this.#path = path;
    this.sync = sync;
    this.#fd = openSync(path, existsSync(path) ? 'r+' : 'w+');
    this.#recover();
  }

  // Rebuild the index by scanning the log. A torn final write (crash mid-append) fails its
  // checksum; everything from there on is discarded, because nothing after it was acknowledged.
  #recover() {
    const total = fstatSync(this.#fd).size;
    const header = Buffer.alloc(HEADER);
    let pos = 0;
    while (pos + HEADER <= total) {
      readSync(this.#fd, header, 0, HEADER, pos);
      const crc = header.readUInt32LE(0), kLen = header.readUInt32LE(4), vLen = header.readInt32LE(8);
      const bodyLen = kLen + Math.max(vLen, 0);
      if (pos + HEADER + bodyLen > total) break;                 // truncated record
      const body = Buffer.alloc(bodyLen);
      readSync(this.#fd, body, 0, bodyLen, pos + HEADER);
      if (crc32(Buffer.concat([header.subarray(4), body])) !== crc) break; // corrupt record
      const key = body.subarray(0, kLen).toString();
      if (vLen < 0) this.#index.delete(key);
      else this.#index.set(key, { offset: pos, length: HEADER + bodyLen });
      pos += HEADER + bodyLen;
    }
    if (pos < total) ftruncateSync(this.#fd, pos);               // drop the torn tail
    this.#size = pos;
  }

  #append(key, value) {
    const k = Buffer.from(key), v = value === null ? null : Buffer.from(value);
    const rec = Buffer.alloc(HEADER + k.length + (v?.length ?? 0));
    rec.writeUInt32LE(k.length, 4);
    rec.writeInt32LE(v ? v.length : -1, 8);
    k.copy(rec, HEADER);
    v?.copy(rec, HEADER + k.length);
    rec.writeUInt32LE(crc32(rec.subarray(4)), 0);
    writeSync(this.#fd, rec, 0, rec.length, this.#size);
    if (this.sync) fsyncSync(this.#fd); // what makes the write survive power loss — and slow
    const loc = { offset: this.#size, length: rec.length };
    this.#size += rec.length;
    return loc;
  }

  set(key, value) { this.#index.set(key, this.#append(key, value)); }
  delete(key) { if (this.#index.has(key)) { this.#append(key, null); this.#index.delete(key); } }

  get(key) {
    const loc = this.#index.get(key);
    if (!loc) return undefined;
    const rec = Buffer.alloc(loc.length);
    readSync(this.#fd, rec, 0, loc.length, loc.offset);         // exactly one disk read
    return rec.subarray(HEADER + rec.readUInt32LE(4)).toString();
  }

  // Compaction: copy only live records to a new file, then atomically swap it in
  compact() {
    const tmp = `${this.#path}.compact`;
    rmSync(tmp, { force: true });                                // leftovers from a crashed compaction
    const next = new LogKV(tmp, { sync: false });
    for (const key of this.#index.keys()) next.set(key, this.get(key));
    fsyncSync(next.#fd);                                         // durable before the swap
    closeSync(next.#fd);
    closeSync(this.#fd);
    renameSync(tmp, this.#path);                                 // atomic replace
    this.#fd = openSync(this.#path, 'r+');
    this.#index = new Map(); this.#recover();
  }

  get bytesOnDisk() { return this.#size; }
  get keys() { return this.#index.size; }
  close() { closeSync(this.#fd); }
}

// ── Demo: 1,000 writes to 10 keys, a simulated crash, then compaction ────────────────
const file = 'demo.log';
rmSync(file, { force: true });
let db = new LogKV(file, { sync: false });
for (let i = 0; i < 1_000; i++) db.set(`user:${i % 10}`, JSON.stringify({ visits: i }));
db.delete('user:9');
console.log(db.get('user:3'), db.keys, db.bytesOnDisk);            // {"visits":993} 9 31908

const raw = openSync(file, 'r+');                                  // crash mid-append: half a record
writeSync(raw, Buffer.from([1, 2, 3, 4, 5, 6, 7]), 0, 7, db.bytesOnDisk);
closeSync(raw);
console.log('file with torn tail:', statSync(file).size);         // file with torn tail: 31915
db.close();
db = new LogKV(file, { sync: false });
console.log('after crash:', db.get('user:3'), db.keys, db.bytesOnDisk);   // after crash: {"visits":993} 9 31908

db.compact();
console.log('after compaction:', db.get('user:3'), db.keys, db.bytesOnDisk); // after compaction: {"visits":993} 9 288
db.close(); rmSync(file);
```

The 1,000 writes left 31,908 bytes for 9 live keys; compaction shrank that to 288 — the space
amplification an append-only design accumulates between compactions, made visible.

The decision to defend is **truncating the log at the first bad checksum instead of skipping the
bad record and continuing.** Skipping looks more resilient — why discard good records after one
corrupt one? Because in an append-only log, a bad record at position *p* almost always means the
process crashed while writing it, so nothing after *p* was ever acknowledged; there *are* no good
records after it, only garbage that happens to parse. Skipping risks resurrecting partial bytes as
a valid-looking record, silently corrupting a value. And if a bad record ever appears in the
*middle* of a log — real media corruption — the right response is to stop and restore from a
replica or backup, not to guess. Real engines follow the same rule: recovery replays the WAL up to
the first record that fails validation and stops. (A production version would also `fsync` the
directory after the rename, which POSIX requires for the rename itself to be durable.)

---

## 10.10 Questions

**1.** MCQ — Why does a B-tree database still write a WAL even though it updates pages in place? a) To support SQL b) So committed changes can be recovered after a crash before dirty pages reach disk c) To compress data d) To replicate to followers only
<details><summary>Answer</summary>**b)** The WAL is appended and `fsync`ed at commit; pages are written later. After a crash, replaying the WAL reconstructs committed changes. (It's also commonly used for replication, but durability is its primary purpose.)</details>

**2.** MCQ — Roughly how many levels does a B-tree with fanout 500 need for 1 billion rows? a) 2 b) 4 c) 10 d) 30
<details><summary>Answer</summary>**b)** 500⁴ = 6.25 × 10¹⁰ ≥ 10⁹, while 500³ = 1.25 × 10⁸ is too small.</details>

**3.** MCQ — What does a Bloom filter in an LSM tree let a read skip? a) The memtable b) SSTables that definitely don't contain the key c) Compaction d) The WAL
<details><summary>Answer</summary>**b)** A negative answer is certain, so files whose filter says "no" need not be read.</details>

**4.** MCQ — Which compaction strategy has the lowest write amplification? a) Leveled b) Size-tiered c) Both are equal d) Neither compacts
<details><summary>Answer</summary>**b)** Size-tiered rewrites data less often, at the cost of more overlapping files (read amplification) and temporary space amplification.</details>

**5.** MCQ — With an index on `(tenant_id, created_at)`, which query can't seek on the index? a) `WHERE tenant_id = 7` b) `WHERE tenant_id = 7 AND created_at > X` c) `WHERE created_at > X` d) `WHERE tenant_id = 7 ORDER BY created_at`
<details><summary>Answer</summary>**c)** `created_at` alone isn't a leftmost prefix of the index.</details>

**6.** MCQ — Which workload best suits a column store? a) Fetching one user's profile by ID b) Summing order amounts by region across 2 billion rows c) Updating a single row's status thousands of times per second d) Point lookups with strict latency targets
<details><summary>Answer</summary>**b)** Scans over a few columns of many rows, where columnar layout and compression minimise bytes read.</details>

**7.** Short — What is group commit, and why does it raise throughput without weakening durability?
<details><summary>Answer</summary>Transactions that arrive while an `fsync` is in progress are written together and made durable by one subsequent `fsync`. Each transaction still waits for a real flush before acknowledgement, but one flush covers many, so throughput rises with concurrency.</details>

**8.** Short — Why do random UUIDv4 primary keys slow inserts on a large B-tree table?
<details><summary>Answer</summary>Each insert targets a random leaf, so once the index exceeds memory most inserts must read a page from disk, splits happen throughout the tree, and pages end up half full. Sequential keys keep inserting into one hot rightmost page.</details>

**9.** Short — What is a tombstone, and why can many of them slow reads?
<details><summary>Answer</summary>A record marking a key as deleted in a log-structured store; it shadows older values until compaction removes both. Reads scanning a range must process every tombstone in it, so large numbers of recent deletes make scans slow.</details>

**10.** Short — What makes an index "covering", and what does it save?
<details><summary>Answer</summary>It contains every column the query needs, so the database answers from the index alone (an index-only scan) and skips the lookup into the table for each row.</details>

**11.** Short — Why is a log-structured hash store (Bitcask) unsuitable for range queries, and what limits its key count?
<details><summary>Answer</summary>Its in-memory index is an unordered hash, so there's no way to iterate keys in order; and every key must fit in that in-memory index.</details>

**12.** Output — What does this print?
```js
const height = (rows, fanout) => Math.ceil(Math.log(rows) / Math.log(fanout));
console.log(height(1e8, 100), height(1e8, 1000));
```
<details><summary>Answer</summary>`4 3` — 100⁴ = 10⁸ needs four levels; with fanout 1,000, 1,000³ = 10⁹ ≥ 10⁸ needs three.</details>

**13.** Output — A storage device's `fsync` takes 2 ms. With one `fsync` per transaction and no group commit, what's the maximum commit rate for a single writer? With group commit averaging 40 transactions per `fsync`?
<details><summary>Answer</summary>**500/s and ~20,000/s.** 1 / 0.002 = 500; 500 × 40 = 20,000 — assuming enough concurrent transactions to fill each group.</details>

**14.** Output — Using the §10.4 sizing formula, about how many bits per key does a Bloom filter need for a 0.1% false-positive rate?
<details><summary>Answer</summary>**~14.4 bits per key.** m/n = −ln(0.001)/ln(2)² ≈ 6.91 / 0.48 ≈ 14.4. Each 10× reduction in false positives costs ~4.8 more bits per key.</details>

**15.** Output — A table has 6 secondary indexes plus its primary key index. Ignoring the WAL and page granularity, how many index structures does each insert modify?
<details><summary>Answer</summary>**7.** The primary key index (or the clustered table itself) plus all six secondary indexes.</details>

**16.** Debug — After switching IDs from `bigserial` to random UUIDv4 strings, insert throughput on a 400 GB table dropped by about two-thirds, and the primary key index grew 2.5× larger than expected. Explain both effects and propose a fix.
<details><summary>Answer</summary>Random keys scatter inserts across the B-tree, forcing disk reads for leaves outside memory and splitting pages everywhere, leaving them about half full; text storage roughly doubles key size in every index. Use time-ordered UUIDv7 (or Snowflake-style IDs) stored as a native 16-byte `uuid` type.</details>

**17.** Debug — A Cassandra table used as a work queue (insert job, read oldest, delete it) has reads that get slower every day and occasionally time out with warnings about scanned tombstones. Cause and fix?
<details><summary>Answer</summary>Each completed job leaves a tombstone, and "read oldest" scans over all recent tombstones until compaction purges them after the grace period. Don't use an LSM table as a queue; use a queue or log (Chapter 24), or partition by time bucket so whole partitions expire and are dropped instead of deleted row by row.</details>

**18.** Debug — A query `WHERE status = 'pending' AND created_at > now() - interval '1 hour'` is slow despite an index on `(created_at, status)`. Only 0.1% of rows are pending. What would you change?
<details><summary>Answer</summary>The range column is first, so the index scans every row of the last hour and filters status afterwards. Use `(status, created_at)` — equality first — or better, a partial index `ON (created_at) WHERE status = 'pending'`, which is tiny and exactly matches the query.</details>

**19.** Debug — A nightly reporting job runs large aggregation queries on the production PostgreSQL primary. Every night, API p99 latency triples for two hours. Why, and what's the fix?
<details><summary>Answer</summary>The scans read large portions of tables, evicting the API's hot working set from the buffer pool and saturating I/O, so API queries hit disk. Move reporting to a replica at minimum, and preferably to a column store fed by CDC or batch exports.</details>

**20.** Code — Write `bloomSize({ keys, falsePositiveRate })` returning `{ bits, hashes, bytes }` using the optimal formulas.
<details><summary>Answer</summary>

```js
function bloomSize({ keys, falsePositiveRate }) {
  const bits = Math.ceil((-keys * Math.log(falsePositiveRate)) / Math.LN2 ** 2);
  const hashes = Math.max(1, Math.round((bits / keys) * Math.LN2));
  return { bits, hashes, bytes: Math.ceil(bits / 8) };
}
console.log(bloomSize({ keys: 1_000_000, falsePositiveRate: 0.01 })); // { bits: 9585059, hashes: 7, bytes: 1198133 }
```
About 1.2 MB to filter a million keys at 1%.</details>

**21.** Code — Write `canSeek(indexColumns, equalityCols, rangeCol)` returning how many leading index columns a query can use to narrow the search, given the columns it constrains by equality and an optional range column.
<details><summary>Answer</summary>

```js
function canSeek(indexColumns, equalityCols, rangeCol = null) {
  let used = 0;
  for (const col of indexColumns) {
    if (equalityCols.includes(col)) { used++; continue; }
    if (col === rangeCol) used++;   // a range can use this column…
    break;                          // …but nothing after it, and a gap stops the prefix
  }
  return used;
}
console.log(canSeek(['a', 'b', 'c'], ['a', 'b'], 'c'), canSeek(['a', 'b', 'c'], ['b'], 'c'), canSeek(['a', 'b', 'c'], ['a'], 'b')); // 3 0 2
```
</details>

**22.** Code — Write `mergeSorted(...runs)` that merges several sorted arrays of `{ key, value, seq }` (as SSTables would be) into one sorted array, keeping only the highest-`seq` version of each key and dropping keys whose newest version is a tombstone (`value === null`).
<details><summary>Answer</summary>

```js
function mergeSorted(...runs) {
  const newest = new Map();
  for (const run of runs) for (const r of run) {
    const cur = newest.get(r.key);
    if (!cur || r.seq > cur.seq) newest.set(r.key, r);
  }
  return [...newest.values()]
    .filter((r) => r.value !== null)               // tombstones and what they shadow both vanish
    .sort((a, b) => (a.key < b.key ? -1 : a.key > b.key ? 1 : 0));
}
console.log(mergeSorted(
  [{ key: 'a', value: 1, seq: 1 }, { key: 'c', value: 3, seq: 2 }],
  [{ key: 'a', value: 9, seq: 5 }, { key: 'c', value: null, seq: 6 }, { key: 'b', value: 2, seq: 4 }],
).map((r) => `${r.key}=${r.value}`).join(' ')); // a=9 b=2
```
A real compaction streams a k-way merge with a heap rather than loading everything; and it may only drop a tombstone once no older file could still hold the key.</details>

**23.** Scenario — You're choosing a store for an IoT platform ingesting 400,000 sensor readings per second, queried mostly as "the last 24 hours for device X" and occasionally "all devices in region Y, last hour". What engine characteristics do you look for?
<details><summary>Answer</summary>Write-heavy, append-mostly, time-ordered data favours an LSM-based or purpose-built time-series store: sequential writes, compression, and TTL-based expiry that drops whole files. Key data by (device, time bucket) so the common query is one partition scanned in order. The regional query is a fan-out across devices — serve it from a pre-aggregated or columnar copy rather than scanning raw data. Avoid row-by-row deletes; let retention expire whole time partitions (Chapter 48 designs this).</details>

**24.** Scenario — A product team wants to add a sixth index to a write-heavy `events` table to speed up a dashboard query run a few dozen times a day. Inserts run at 8,000/s. How do you decide?
<details><summary>Answer</summary>Weigh a permanent cost on 8,000 writes/s (an extra B-tree write each, more memory pressure, more WAL and replication traffic) against a benefit to a few dozen queries a day. Measure the dashboard query with `EXPLAIN ANALYZE` and the insert path's headroom first. Usually better: serve the dashboard from a replica, a pre-aggregated rollup table updated periodically, or a column store. Add the index only if the query is user-facing and latency-critical and the write path has clear headroom.</details>

---

## Chapter Summary

- Durability comes from appending to a log and calling `fsync`; `fsync` latency dominates commit
  cost, and group commit amortises it across concurrent transactions.
- B-trees update fixed-size pages in place with very high fanout, so a billion-row lookup is a few
  levels and usually one disk read — as long as the working set fits in memory.
- LSM trees never update in place: writes are sequential flushes and compactions, reads are
  rescued by Bloom filters, and deletes become tombstones that cost until compacted.
- Choose engines by amplification: B-trees for mixed transactional workloads with range scans, LSM
  for write-heavy and large datasets — and measure with your own access pattern.
- Every index is paid for on every write; order composite columns equality-first, and use covering
  and partial indexes for the hottest queries.
- Row stores serve transactions and column stores serve analytics; copy data from one to the other
  instead of running analytics on production.

---

## Practice Exercises

1. Run the §10.9 store with `sync: true` and measure writes per second on your machine; then
   implement group commit (batch writes arriving within 2 ms into one `fsync`) and measure again.
2. Insert 20 million rows into two PostgreSQL tables, one with a `bigint` identity key and one with
   random UUIDv4 keys, and compare insert time and index size with `pg_relation_size`.
3. Read the RocksDB documentation on leveled and universal compaction and summarise when each is
   recommended.
4. Take the five slowest queries in a database you work with, run `EXPLAIN ANALYZE`, and design an
   index (composite, covering, or partial) for each; measure the before/after cost.
5. Extend the §10.9 store into a tiny LSM: a sorted memtable flushed to sorted files, a Bloom filter
   per file, and a merge-based compaction.

---

## Mini Project: A Mini LSM Engine

Build a small LSM-tree key-value store in Node and measure its amplification under different
workloads — the point is to feel the trade-offs in §10.5 rather than memorise them.

- WAL with checksums and crash recovery; a memtable flushed to immutable sorted files at a size
  threshold.
- Per-file sparse index and Bloom filter; reads check memtable, then files newest to oldest.
- Tombstones for deletes; a compaction that merges files and drops shadowed versions.
- Range scans that merge iterators across the memtable and all files.
- Report write, read, and space amplification for three workloads: insert-only, update-heavy on a
  small key set, and insert-then-delete (queue-like).

---

**Next:** [Chapter 11 — Choosing a Data Store →](11-choosing-a-data-store.md)
