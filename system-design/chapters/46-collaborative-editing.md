# Chapter 46 — Design a Collaborative Editor

## Learning Objectives

- Scope a real-time collaborative document editor and size its connections, operation rate, and document history
- Explain why concurrent edits can't simply be applied in arrival order, and how operational transformation (OT) fixes it with a server
- Explain how CRDTs make replicas converge without a server, and what they cost in metadata and tombstones
- Design the session layer, operation log, snapshots, presence, offline editing, and access control around either approach
- Choose between OT and CRDTs for a product, and defend the choice

---

## 46.1 Requirements and Scope

A collaborative editor lets several people edit the same document at once and see each other's changes within a fraction of a
second, the way shared documents, design tools, and code-pairing editors do. Every keystroke is a write; every writer is also a
reader; and two people typing in the same sentence at the same moment is not an edge case but the product.

**Functional requirements**

- Create, open, and edit rich-text documents; changes from other editors appear live.
- Show presence: who's in the document, their cursors and selections.
- Edit offline (or on a bad connection) and merge on reconnect.
- Version history: view and restore earlier versions; see who changed what.
- Share with permissions: owner, editor, commenter, viewer.

**Out of scope**: comments and suggestions in depth (they're anchored ranges, covered briefly in §46.7), spreadsheets and drawings (the
same principles over different data types), search across documents (Chapter 18).

**Non-functional requirements**

| Requirement | Target |
|---|---|
| Convergence | All editors who've seen the same edits see the **same document** — always, with no manual merge |
| Intent preservation | An edit lands where its author meant it, even if the text around it changed concurrently |
| Latency | Local edits apply instantly; remote edits visible in p99 < 300 ms |
| Durability | An edit acknowledged to its author is never lost |
| Scale | ~5 million open editing sessions at peak; documents from one editor to a few hundred |

"Applies instantly" rules out sending each keystroke to a server and waiting for permission: the editor must apply edits **optimistically**
and reconcile afterwards. That one requirement is why the rest of the chapter exists.

---

## 46.2 Estimation

```js
// Collaborative editor: connections, operation rate, and why the op log must be compacted
const openSessions = 5e6, typingShare = 0.2, charsPerSec = 4, batchMs = 100;
const typists = openSessions * typingShare;
const opsIn = typists * Math.min(charsPerSec, 1000 / batchMs);      // client batches keystrokes every 100 ms
const avgEditorsPerDoc = 1.6;                                       // most docs have one editor; a few have dozens
const fanOut = opsIn * (avgEditorsPerDoc - 1);
console.log(`${(openSessions / 1e6).toFixed(0)} M WebSockets | ~${(opsIn / 1e6).toFixed(1)} M op batches/s in | ~${(fanOut / 1e6).toFixed(1)} M/s fanned out to co-editors`);
const opsPerDocLifetime = 200_000, bytesPerOp = 60;                 // a long-lived doc edited for months
console.log(`one busy doc's op log: ~${(opsPerDocLifetime * bytesPerOp / 1e6).toFixed(0)} MB vs ~0.05 MB of text — replay needs snapshots`);
// 5 M WebSockets | ~4.0 M op batches/s in | ~2.4 M/s fanned out to co-editors
// one busy doc's op log: ~12 MB vs ~0.05 MB of text — replay needs snapshots
```

What the numbers decide:

- **Millions of long-lived connections** mean a WebSocket gateway tier (Chapter 9) — the same machinery as chat (Chapter 37).
- **Millions of small operations per second in aggregate, but tiny per document.** Even a busy document sees tens of operations per second.
  That makes the natural unit of scale the **document**: route every editor of a document to the same session server, which then handles a
  small, ordered stream in memory. No single document needs distributed coordination; the fleet just needs many documents spread across
  many servers.
- **The operation log grows far larger than the text.** Opening a document by replaying 200,000 operations is too slow, so the log is
  periodically compacted into **snapshots**, with only recent operations replayed on top (§46.7).

---

## 46.3 API and Data Model

```text
GET  /v1/docs/{id}                          → latest snapshot + revision (for the initial load)
WS   /v1/docs/{id}/session                  bidirectional, per open document:
       client → server   { type: 'ops', baseRev, ops: [...], clientId, seq }        (OT)
                         { type: 'update', changes }                                 (CRDT)
                         { type: 'presence', cursor, selection }
       server → client   { type: 'ack', seq, rev } | { type: 'ops', rev, ops, author } | { type: 'presence', ... }
GET  /v1/docs/{id}/history?before=rev       → version list; POST /v1/docs/{id}/restore { rev }
```

| Data | Store | Notes |
|---|---|---|
| **Operation log** | Append-only, per document, ordered by revision | The source of truth for "what happened"; durable before ack |
| **Snapshots** | Object storage (Chapter 17), keyed by doc and revision | Full document state every *N* ops or minutes |
| **Document metadata** | Relational (Chapter 11) | Title, owner, permissions, latest revision, session server |
| **Presence** | Memory on the session server only | Cursors are ephemeral; losing them on a crash is fine |

The client's `seq` makes resent operations idempotent after a reconnect (Chapter 8's idea, applied per session): the server ignores a
`(clientId, seq)` it has already accepted and re-acknowledges it.

---

## 46.4 High-Level Design

```text
editors ─WS─▶ gateways ─▶ session server for doc D (chosen by consistent hashing on doc ID)
                              │   in memory: current document, recent ops, connected editors, presence
                              │   orders (OT) or relays (CRDT) operations; broadcasts to co-editors
                              ▼
                        operation log (durable append, then ack) ─▶ snapshotter ─▶ snapshot store
                              │
                              └─▶ change events (Chapter 26): search indexing, notifications, history UI
```

- **One session server per active document.** Gateways route by document ID (Chapter 14's consistent hashing, with a registry for
  overrides). The session server loads the latest snapshot plus subsequent ops, then serves every editor of that document from memory.
- **Durability before acknowledgement.** An operation is appended to the document's log before it's acknowledged to its author and
  broadcast — so an acked edit survives a session-server crash.
- **Failover.** If a session server dies, its documents are reassigned; clients reconnect, the new server rebuilds from snapshot + log, and
  each client resends any un-acknowledged operations (idempotent by `seq`).

This shape is the same for OT and CRDTs. What differs is what the session server does with each incoming operation — §46.5 and §46.6.

---

## 46.5 Deep Dive: Operational Transformation

Two editors start from `"Hello world"`. Alice inserts `,` at position 5. Bob, at the same moment, deletes the character at position 0.
Each applies their own edit locally, then receives the other's. Applied as-is, the positions are wrong for the document they now hold:

```js
// Operational transformation for character-level insert/delete. transform(a, b) rewrites `a` so it can be
// applied *after* `b` while keeping a's intent. Range operations follow the same rules with more cases.
const apply = (s, op) => !op ? s
  : op.type === 'ins' ? s.slice(0, op.pos) + op.ch + s.slice(op.pos)
  : s.slice(0, op.pos) + s.slice(op.pos + 1);

function transform(a, b) {
  if (!a || !b) return a;
  if (b.type === 'ins') {
    // b's insert sits before a's position (ties broken by site, so both sides make the same choice)
    const before = b.pos < a.pos || (b.pos === a.pos && (a.type === 'del' || b.site < a.site));
    return before ? { ...a, pos: a.pos + 1 } : a;
  }
  if (b.pos < a.pos) return { ...a, pos: a.pos - 1 };          // b deleted a character before a
  if (b.pos === a.pos && a.type === 'del') return null;        // both deleted the same character
  return a;
}

const doc = 'Hello world';
const alice = { type: 'ins', pos: 5, ch: ',', site: 'alice' };   // wants "Hello, world"
const bob = { type: 'del', pos: 0, site: 'bob' };                // wants "ello world"
console.log('naive:      ', JSON.stringify(apply(apply(doc, alice), bob)), JSON.stringify(apply(apply(doc, bob), alice)));
console.log('transformed:', JSON.stringify(apply(apply(doc, alice), transform(bob, alice))), JSON.stringify(apply(apply(doc, bob), transform(alice, bob))));

// Fuzz: random concurrent pairs must converge at both sites
let diverged = 0;
for (let i = 0; i < 10_000; i++) {
  const s = 'abcdef';
  const rnd = (site) => Math.random() < 0.5
    ? { type: 'ins', pos: Math.floor(Math.random() * (s.length + 1)), ch: site[0].toUpperCase(), site }
    : { type: 'del', pos: Math.floor(Math.random() * s.length), site };
  const a = rnd('alice'), b = rnd('bob');
  if (apply(apply(s, a), transform(b, a)) !== apply(apply(s, b), transform(a, b))) diverged++;
}
console.log('diverged in 10,000 random pairs:', diverged);
// naive:       "ello, world" "ello ,world"
// transformed: "ello, world" "ello, world"
// diverged in 10,000 random pairs: 0
```

Untransformed, Bob's copy puts the comma in the wrong place, and the two copies have **diverged** — permanently, since every later edit
builds on different text. Transformed, Bob's view of Alice's insert shifts left by one to account for the character he deleted, and both
converge on the text both intended.

**How OT systems use this.** The server holds the canonical document at revision *r*. A client sends its pending operations tagged with the
revision they were made against. If other operations were accepted since then, the server transforms the incoming ones against them,
applies the result as revision *r* + 1, persists, acknowledges, and broadcasts. Each client does the mirror image: incoming server operations
are transformed against its own not-yet-acknowledged ones. The server's total order means transforms only ever need to be correct for
**two** concurrent streams, which is the property that makes OT tractable in practice.

**Why OT is hard.** Each pair of operation types needs a transform, and rich text has many: insert, delete, format a range, split a paragraph,
insert a table row. The number of cases grows quadratically, each must be right, and bugs show up as rare divergence in production. Peer-to-peer
OT (no central order) needs an extra property that most published algorithms were later shown to violate. In practice, OT works when there's
**one server ordering each document** — and then it's efficient, well-understood, and has decades of production use.

> ⚠️ **Gotcha** — A transform that's "obviously right" for single characters can be wrong for ranges. A delete of `[0, 5)` concurrent with an
> insert at position 2 must decide whether the inserted text survives — and *both sides must make the same decision*. Expanding the delete to
> swallow the insert on one side while the other keeps it is a divergence bug that a two-character test never finds. Fuzz-test transforms
> with random concurrent operations, as the block above does.

---

## 46.6 Deep Dive: CRDTs

A **CRDT** (conflict-free replicated data type) takes the opposite approach: instead of rewriting operations to fit, it designs the data so
that operations **commute** — applying the same set of operations in any order, any number of times, yields the same state. Then replicas
converge by simply exchanging operations, with no server needed to order or transform them (Chapter 20's strong eventual consistency).

**The CRDT family.** Text is the hardest case; simpler ones show the idea:

| CRDT | State | Merge | Used for |
|---|---|---|---|
| **G-Counter** | A count per replica | Per-replica maximum; value is the sum | Likes, views, distributed counters |
| **PN-Counter** | Two G-Counters (increments, decrements) | Merge each; value is the difference | Counters that go down |
| **LWW-Register** | Value + timestamp | Larger timestamp wins | A field where losing a concurrent write is acceptable |
| **OR-Set** | Elements tagged with unique add IDs; removes record the tags they saw | Union of adds minus removed tags | Shopping carts, tags, membership (Chapter 39's resurrected item) |
| **Sequence (RGA, YATA, Fugue)** | Elements with unique IDs and a reference to their neighbour | Deterministic placement rule | Text, lists |

**Sequence CRDTs for text.** Positions like "index 5" change meaning under concurrent edits, so sequence CRDTs give every character a
**permanent, unique ID** and express inserts relative to IDs: "insert `,` after the character with ID 5@alice". Concurrent inserts after the same
character are ordered by a deterministic rule on their IDs, so every replica places them identically. Deletes mark the character as a
**tombstone** rather than removing it, because a concurrent insert may still refer to it. §46.10 implements this as an RGA.

**What CRDTs cost.**

- **Metadata.** Every character carries an ID and a neighbour reference — far more than one byte. Modern libraries compress runs of characters
  typed in sequence into single entries, which brings overhead down to a small multiple of the text size for typical documents.
- **Tombstones.** Deleted text stays in the structure. Removing tombstones safely requires knowing that every replica has seen the delete —
  coordination the CRDT was supposed to avoid — so systems either keep them (compacted) or garbage-collect when a server can confirm all
  replicas are past them.
- **Intent anomalies.** Convergence is guaranteed; *sensible* results are not. Two people typing words at the same spot can produce
  interleaved characters in some algorithms; newer sequence CRDTs (YATA, Fugue) are designed specifically to avoid interleaving.
- **Validation.** With no server deciding, a malicious or buggy client can broadcast operations that are valid CRDT operations but violate
  application rules. Production systems still route updates through a server that checks permissions and schema.

**When CRDTs win.** Offline-first and local-first apps, peer-to-peer collaboration, and multi-region active-active editing — anywhere there's no
single server that can order every operation, or edits must merge after hours apart. Even with a server, a CRDT makes the server simpler: it
relays and persists updates instead of transforming them.

---

## 46.7 Deep Dive: Persistence, Presence, Offline, and Permissions

**The log and snapshots.** The operation log is the durable record; snapshots make loading fast. A snapshotter periodically (every few hundred
operations or minutes of activity) materialises the document at revision *r* and writes it to object storage; opening the document loads the
snapshot and replays only the operations after *r*. Old operations can then be compacted — merged into coarser history entries ("Alice edited
from 10:02 to 10:41") — to keep version history useful without storing every keystroke forever.

**Version history and restore.** A version is a snapshot plus a name. Restoring doesn't rewind the log; it appends a new operation (or set of
operations) that changes the current document to match the old one, so restore is itself an edit — collaborative, undoable, and visible to
everyone in the session.

**Presence.** Cursors and selections are sent on the same WebSocket but treated differently: they're **ephemeral** (kept in the session server's
memory, never logged), **lossy** (only the latest position matters, so they're throttled and coalesced), and **anchored** — a cursor at "index 40"
moves when someone types before it, so cursors are expressed relative to the document model (a character ID in a CRDT, or transformed like an
operation in OT).

**Offline editing.** The client keeps a local copy and a queue of unsent operations. On reconnect, it sends them with the revision (OT) or
version vector (CRDT, Chapter 22) it last synced. With OT, a long offline session means transforming many operations against many — workable but
expensive, and the server should reject sessions that are too stale and fall back to a merge view. With a CRDT, merging is the normal path.

**Permissions.** Checked when a session opens, and again on each operation — a permission revoked while someone is editing must take effect in
seconds, so the session server subscribes to permission changes for its documents and closes or downgrades affected sessions. Viewers receive
operations but can't send them; commenters send only comment operations, which are **anchored ranges** that move with the text like cursors do.

> 💼 **Interview** — Interviewers want to hear the concurrency problem stated precisely ("positions change under concurrent edits, so naive
> application diverges"), then a choice: OT with a central server per document, or CRDTs for offline and peer-to-peer. The strongest answers
> explain why per-document session routing makes either approach scale, and treat presence as ephemeral data on a different path.

---

## 46.8 Failure Modes, Bottlenecks, and 10×

| Failure | Effect | Mitigation |
|---|---|---|
| Session server crash | Editors disconnected; in-memory state lost | Ops durable before ack; reassign the document; clients reconnect and resend unacked ops by `seq` |
| Two servers think they own a document | Two divergent orders (OT) | Ownership is a lease with a fencing token (Chapter 21); the log rejects appends with a stale token |
| A document with 500 editors | One session server and fan-out hot | Batch broadcast, coalesce presence, cap live editors (others view-only with near-real-time updates) |
| Slow client falls behind | Memory grows on the server buffering for it | Bounded per-client buffers; drop and resync from snapshot + log |
| Transform or merge bug | Silent divergence between clients | Periodic checksums of document state compared across clients and server; force resync on mismatch; fuzz tests |
| Huge document (book-length, years of history) | Slow open, big snapshots | Load by section; compact history; tombstone GC behind a server-confirmed version |

**At 10×** — 50 million open sessions: more gateways and session servers, with the same per-document routing; the operation log becomes a
partitioned log keyed by document (Chapter 24); snapshotting and history compaction become a large batch workload of their own; and very large
documents get sub-document sharding (each section or page its own CRDT or OT stream), because one document is the one unit that doesn't
spread across machines.

---

## 46.9 Common Mistakes and Best Practices

**Mistake: last-write-wins on the whole document.** Two people edit; one person's work disappears. Collaboration requires operation-level merging.

**Mistake: locking paragraphs or documents for editing.** It avoids conflicts by preventing collaboration, and stale locks from closed tabs block
everyone.

**Mistake: waiting for the server before applying local edits.** Typing latency becomes network latency. Apply locally, reconcile afterwards.

**Mistake: OT without a single ordering server per document.** Peer-to-peer OT is notoriously hard to get right; most designs that try it diverge.

**Mistake: ignoring tombstones and metadata growth in a CRDT.** Long-lived documents grow without bound.

**Mistake: logging presence.** Cursor movements outnumber edits by far and have no lasting value.

**Mistake: acknowledging before persisting.** A session-server crash loses edits the author saw confirmed.

**Best practice: route each document to one session server.** Ordering, broadcast, and presence become local and in-memory.

**Best practice: fuzz-test convergence.** Random concurrent operations and random delivery orders, checking that every replica ends identical.

**Best practice: checksum documents across replicas in production.** Divergence bugs are rare, silent, and catastrophic; detect and resync.

---

## 46.10 Real-World Example: A Text CRDT

This is a Replicated Growable Array (RGA), one of the simplest correct sequence CRDTs. Each character gets a unique ID — a Lamport counter
(Chapter 22) paired with the site name — and records the ID of the character it was typed after. Concurrent inserts after the same character are
ordered by ID, deletes leave tombstones, duplicate operations are ignored, and operations whose dependency hasn't arrived are buffered until it
does. Production libraries add run-length compression, rich-text formatting, and binary encoding, but rest on the same rules.

```js
// rga.mjs — a Replicated Growable Array: a sequence CRDT for collaborative text. Every character has a
// globally unique ID (Lamport counter, site) and remembers the character it was typed after. Replicas that
// apply the same set of operations, in any order, end up with the same text — no server needs to transform.

const cmp = (a, b) => a[0] - b[0] || (a[1] < b[1] ? -1 : a[1] > b[1] ? 1 : 0);   // total order on IDs
const key = (id) => id ? `${id[0]}@${id[1]}` : 'root';

export class RGA {
  #site; #clock = 0;
  #items = [];                      // in document order: { id, after, ch, deleted }
  #index = new Map();               // id key → item, to find origins and targets quickly
  #pending = [];                    // ops whose dependency hasn't arrived yet (causal buffering)

  constructor(site) { this.#site = site; }

  // ── Local edits produce operations to broadcast ──────────────────────────────────
  insert(pos, ch) {                 // pos counts visible characters
    const left = pos === 0 ? null : this.#visible()[pos - 1].id;
    const op = { type: 'ins', id: [++this.#clock, this.#site], after: left, ch };
    this.apply(op);
    return op;
  }
  delete(pos) {
    const op = { type: 'del', target: this.#visible()[pos].id };
    this.apply(op);
    return op;
  }

  // ── Remote (or local) operations, in any order, any number of times ─────────────
  apply(op) {
    if (!this.#ready(op)) { this.#pending.push(op); return; }
    this.#integrate(op);
    for (let progress = true; progress; ) {             // an arrival may unblock buffered ops
      progress = false;
      for (const p of [...this.#pending]) {
        if (this.#ready(p)) { this.#pending.splice(this.#pending.indexOf(p), 1); this.#integrate(p); progress = true; }
      }
    }
  }

  #ready(op) { return op.type === 'ins' ? op.after === null || this.#index.has(key(op.after)) : this.#index.has(key(op.target)); }

  #integrate(op) {
    if (op.type === 'del') { this.#index.get(key(op.target)).deleted = true; return; }   // tombstone, never remove
    if (this.#index.has(key(op.id))) return;            // duplicate delivery is harmless
    this.#clock = Math.max(this.#clock, op.id[0]);
    let i = op.after === null ? 0 : this.#items.indexOf(this.#index.get(key(op.after))) + 1;
    // Concurrent inserts after the same character: the larger ID goes first, so every replica agrees
    while (i < this.#items.length && cmp(this.#items[i].id, op.id) > 0) i++;
    const item = { id: op.id, after: op.after, ch: op.ch, deleted: false };
    this.#items.splice(i, 0, item);
    this.#index.set(key(op.id), item);
  }

  #visible() { return this.#items.filter(x => !x.deleted); }
  text() { return this.#visible().map(x => x.ch).join(''); }
  get tombstones() { return this.#items.length - this.#visible().length; }
}

// ── Demo: three replicas, concurrent edits, out-of-order and duplicate delivery ───────
const alice = new RGA('alice'), bob = new RGA('bob'), carol = new RGA('carol');
const log = [];
const type = (r, pos, s) => [...s].forEach((ch, k) => log.push(r.insert(pos + k, ch)));

type(alice, 0, 'cat');
for (const op of log) bob.apply(op);                      // bob is in sync: "cat"
const synced = log.length;

type(alice, 3, 's');                                      // alice: "cats"
log.push(bob.delete(0)); type(bob, 0, 'b');               // bob, concurrently: "bat"
type(bob, 3, '!');                                        // ...then "bat!", also typed right after the "t"

for (const op of log.slice(synced)) { alice.apply(op); bob.apply(op); }   // exchange; own ops are no-ops
for (const op of [...log].reverse()) carol.apply(op);     // carol receives everything backwards
for (const op of log) carol.apply(op);                    // ...and then all of it again
console.log([alice, bob, carol].map(r => JSON.stringify(r.text())).join(' '));
console.log('tombstones kept:', carol.tombstones);
// "bat!s" "bat!s" "bat!s"
// tombstones kept: 1
```

All three replicas converge on `"bat!s"` even though Carol received every operation backwards and then again. Alice's `s` and Bob's `!` were
both typed after the `t`; Bob's has the larger ID (`5@bob` vs `4@alice`), so it goes first on every replica — `"bats!"` would have been
equally valid, and the rule's only job is that everyone picks the same one. The deleted `c` remains as a tombstone, because another replica could
still be sending an insert that was typed after it.

The decision to defend is **choosing a CRDT over OT for this editor**, given that §46.4 already routes each document through one session server —
the setup where OT is at its best. OT with a central server is proven, compact (no per-character IDs or tombstones), and simpler to reason about
for formatting-heavy rich text. But the requirements include offline editing that merges on reconnect, and that's where OT gets expensive and
fragile: a client returning after hours must transform its whole queue against everything that happened meanwhile, and the server must keep
enough history to do it. A CRDT makes that path the normal path — the same merge whether the edits are 50 ms or 5 days apart — and it reduces
the session server to relaying, validating, and persisting updates, so a server failover can't produce a divergent order. The cost is metadata
and tombstone growth, which modern run-length-compressed CRDT libraries keep to a small multiple of the text, and which snapshot compaction
bounds further. For an editor that promised offline editing, that's a better trade than an algorithm whose hardest case is a feature on the
roadmap.

---

## 46.11 Questions

**1.** MCQ — Why can't concurrent edits be applied in arrival order? a) They're too large b) Positions refer to the document as each author saw it; after a concurrent edit, the same position means a different place, so replicas diverge c) WebSockets reorder d) Unicode
<details><summary>Answer</summary>**b)** §46.5's naive output shows two different documents.</details>

**2.** MCQ — What makes OT tractable in production? a) Peer-to-peer exchange b) A single server that orders each document's operations, so transforms only reconcile two streams c) Tombstones d) Version vectors
<details><summary>Answer</summary>**b)** Peer-to-peer OT needs properties most algorithms failed to satisfy.</details>

**3.** MCQ — What property lets CRDT replicas converge without coordination? a) Operations commute and are idempotent, so the same set in any order gives the same state b) A global lock c) Synchronized clocks d) Last write wins
<details><summary>Answer</summary>**a)** That's strong eventual consistency.</details>

**4.** MCQ — Why does a sequence CRDT keep deleted characters as tombstones? a) For undo only b) A concurrent insert may reference the deleted character as its neighbour c) Auditing d) Compression
<details><summary>Answer</summary>**b)** Removing it would leave that insert with nowhere to go.</details>

**5.** MCQ — How should presence (cursors) be handled? a) Logged with operations b) Ephemeral, throttled, kept in the session server's memory, anchored to the document model c) Stored in the database d) Sent by email
<details><summary>Answer</summary>**b)** Only the latest position matters.</details>

**6.** MCQ — Why route all editors of a document to one session server? a) Security b) Ordering, broadcast, and presence become local in-memory operations on a small stream c) Cheaper DNS d) Required by WebSockets
<details><summary>Answer</summary>**b)** The document is the natural unit of scale.</details>

**7.** Short — Why must edits be applied locally before the server responds?
<details><summary>Answer</summary>Typing must feel instant; waiting for a round trip per keystroke makes latency visible. So edits are optimistic, and the system reconciles concurrent ones afterwards — the reason OT or CRDTs are needed at all.</details>

**8.** Short — Name three costs of CRDTs relative to OT.
<details><summary>Answer</summary>Per-element metadata (IDs, neighbour references); tombstones that grow with deletions and need coordination to collect; and possible intent anomalies like interleaving, plus the need for server-side validation since clients can send any valid operation.</details>

**9.** Short — How does restoring an old version work in a collaborative editor?
<details><summary>Answer</summary>By appending new operations that change the current document to match the old version, not by rewinding the log — so the restore is itself an edit that other editors receive and that can be undone.</details>

**10.** Short — How should a permission revoked mid-session take effect?
<details><summary>Answer</summary>The session server subscribes to permission changes for its documents and closes or downgrades affected sessions within seconds; operations are checked against current permissions, not just at session start.</details>

**11.** Short — Why are operations persisted before they are acknowledged?
<details><summary>Answer</summary>The author treats an ack as "saved". If the session server crashes after acking but before persisting, the edit is lost even though the author saw it confirmed. Persisting first means an acked edit survives, and an unacked one is resent by the client.</details>

**12.** Output — Using §46.5's `transform`, what does `transform({ type: 'del', pos: 3, site: 'a' }, { type: 'del', pos: 3, site: 'b' })` return?
<details><summary>Answer</summary>**`null`.** Both deleted the same character, so the second delete becomes a no-op.</details>

**13.** Output — Using §46.5's `transform`, Alice inserts `X` at 2 and Bob inserts `Y` at 2 in `"abcd"`, with sites `'alice'` and `'bob'`. What's the converged text?
<details><summary>Answer</summary>**`"abXYcd"`.** `'alice' < 'bob'`, so Alice's insert is treated as first: Bob's shifts to 3 on Alice's side, and Alice's stays at 2 on Bob's.</details>

**14.** Output — In §46.10's RGA, `alice` inserts `x` at position 0 of an empty document (ID `1@alice`) and `bob` concurrently inserts `y` at position 0 (ID `1@bob`). After exchanging, what's the text?
<details><summary>Answer</summary>**`"yx"`.** Both are inserted after the root; IDs tie on the counter, `'bob' > 'alice'`, so `1@bob` is larger and goes first.</details>

**15.** Output — Using §46.2's model, with 10 million open sessions, 30% typing, and 4 characters per second, how many op batches arrive per second?
<details><summary>Answer</summary>**12 million.** 10 M × 0.3 × 4.</details>

**16.** Debug — Two users report seeing different text in the same document after a session-server failover; refreshing fixes it. What happened?
<details><summary>Answer</summary>During failover, two servers briefly both accepted operations for the document and ordered them differently (OT), so clients diverged. Make ownership a fenced lease so the log rejects appends from the stale owner; add periodic document checksums to detect and resync divergence.</details>

**17.** Debug — Opening a two-year-old document takes 40 seconds. Profiling shows the server replaying 3 million operations. Fix?
<details><summary>Answer</summary>Snapshot periodically and load the latest snapshot plus only the operations after it; compact old operations into coarser history entries; for CRDTs, collect tombstones behind a server-confirmed version.</details>

**18.** Debug — The database storing operations is overwhelmed; 90% of writes are cursor movements. What's wrong?
<details><summary>Answer</summary>Presence is being treated as document operations. Send cursors on the same connection but keep them in session-server memory, throttle and coalesce them, and never persist them.</details>

**19.** Debug — A user who edited offline on a flight reconnects; the server spends minutes transforming and then times out, and their edits are lost. What do you change?
<details><summary>Answer</summary>With OT, long offline sessions mean transforming many operations against many. Bound it: reject sessions beyond a staleness limit into an explicit merge view rather than losing work, keep the client's queue until the server confirms it, or move to a CRDT where offline merge is the normal path.</details>

**20.** Code — Write a G-Counter CRDT with `increment(replica, n)`, `merge(other)`, and `value()`.
<details><summary>Answer</summary>

```js
class GCounter {
  #counts = new Map();                                  // replica → its own monotonically growing count
  increment(replica, n = 1) { this.#counts.set(replica, (this.#counts.get(replica) ?? 0) + n); return this; }
  merge(other) {                                        // per-replica max: commutative, associative, idempotent
    for (const [r, c] of other.#counts) this.#counts.set(r, Math.max(c, this.#counts.get(r) ?? 0));
    return this;
  }
  value() { return [...this.#counts.values()].reduce((a, b) => a + b, 0); }
}
const a = new GCounter().increment('a', 3), b = new GCounter().increment('b', 2);
a.merge(b); b.merge(a); a.merge(b);                     // merging repeatedly changes nothing
console.log(a.value(), b.value());                      // 5 5
```
</details>

**21.** Code — Write `transformCursor(cursor, op)` that moves a cursor index correctly when a character-level operation (from §46.5) is applied before it.
<details><summary>Answer</summary>

```js
function transformCursor(cursor, op) {
  if (op.type === 'ins') return op.pos <= cursor ? cursor + 1 : cursor;   // text inserted before the cursor pushes it right
  return op.pos < cursor ? cursor - 1 : cursor;                          // deleted text before it pulls it left
}
console.log(transformCursor(5, { type: 'ins', pos: 2 }), transformCursor(5, { type: 'del', pos: 2 }), transformCursor(5, { type: 'ins', pos: 7 })); // 6 4 5
```
</details>

**22.** Code — Write `checksum(text)` that the server and clients can compare every few seconds to detect divergence, and show it differs for §46.5's naive outputs.
<details><summary>Answer</summary>

```js
import { createHash } from 'node:crypto';
const checksum = (text) => createHash('sha256').update(text).digest('hex').slice(0, 12);  // short is fine: it's a mismatch detector
console.log(checksum('ello, world') === checksum('ello ,world'), checksum('ello, world') === checksum('ello, world')); // false true
```
Clients send the checksum together with the revision it applies to, so a mismatch at the same revision reliably means divergence rather than lag.</details>

**23.** Scenario — A design tool wants multiplayer editing of shapes on a canvas (position, colour, size, z-order), not text. Does it need OT or a sequence CRDT?
<details><summary>Answer</summary>Usually neither in full. Most properties are independent registers per object, so per-property last-writer-wins (ordered by the server) is acceptable — two people dragging the same shape is rare and resolves sensibly. Object creation and deletion is a map or set CRDT, z-order needs a sequence or fractional indices, and text inside shapes uses a text CRDT. The session-server-per-document design still applies.</details>

**24.** Scenario — Your editor must support 300 people in one document during a company all-hands, all typing notes. What breaks, and what do you do?
<details><summary>Answer</summary>Fan-out: 300 editors × their operations × 299 recipients, plus presence for 300 cursors, all on one session server. Batch broadcasts (e.g., every 50–100 ms), coalesce and throttle presence (show only nearby cursors, or a count), cap simultaneous editors with the rest viewing near-real-time, and encourage structure (each team in its own section, or its own sub-document) so the hot spot is split.</details>

---

## Chapter Summary

- Concurrent edits can't be applied in arrival order: positions change meaning, and replicas diverge permanently without OT or a CRDT.
- OT rewrites operations against concurrent ones; it's efficient and proven with one ordering server per document, and fragile without one.
- CRDTs make operations commute, so replicas converge by exchanging them in any order; they pay with metadata, tombstones, and validation work.
- Route each document to one session server, persist operations before acknowledging, and rebuild from snapshot plus recent log on failover.
- Presence is ephemeral and throttled; permissions are enforced continuously; restore is a new edit, not a rewind.
- Fuzz-test convergence and checksum documents in production, because divergence bugs are rare, silent, and permanent.

---

## Practice Exercises

1. Extend §46.5's OT to range inserts and deletes, and fuzz it until it converges — including the insert-inside-deleted-range case from the Gotcha.
2. Add run-length compression to §46.10's RGA (consecutive characters from one site as one entry) and measure memory against plain RGA on a typed document.
3. Implement an OR-Set CRDT and show that it doesn't resurrect removed items the way a plain set union does (Chapter 39).
4. Build a WebSocket session server for one document that relays §46.10's operations between browser clients, persists them, and snapshots every 100 operations.
5. Simulate two users editing offline for an hour each, then merging, with OT and with the RGA, and compare merge time and results.

---

## Mini Project: A Collaborative Text Editor

Build a minimal collaborative editor with a browser client, a session server, and durable storage, and test its convergence and failure handling.

- A textarea-based client that applies edits locally and exchanges RGA operations over a WebSocket, with remote cursors shown.
- A session server per document (route by document ID across two server processes) that validates, persists, acknowledges, and broadcasts.
- Snapshots every *N* operations, load from snapshot + log, and a version history with restore-as-edit.
- Offline mode: queue operations while disconnected and merge on reconnect, with idempotent resend by sequence number.
- A test harness with simulated clients typing randomly through disconnects and a session-server restart, asserting identical text and checksums at
  the end.

---

**Next:** [Chapter 47 — Design Ticketing and Flash Sales →](47-ticketing-flash-sales.md)
