# Chapter 33 — Security Architecture

## Learning Objectives

- Draw trust boundaries on an architecture diagram and threat-model the flows that cross them
- Design authentication for users and services, including token validation and service identity
- Choose an authorization model — RBAC, ABAC, or relationship-based — and enforce it on every object
- Manage secrets and keys with short-lived credentials and envelope encryption
- Contain breaches with tenant isolation, egress controls, supply-chain protections, and audit logs

---

## 33.1 Trust Boundaries and Threat Modelling

Security in system design is mostly about **boundaries**: places where data or control passes between parties that
trust each other differently. The internet and your edge. Your edge and your services. A service and its database.
One tenant's data and another's. Your code and a third-party library, a webhook sender, an LLM's output. Every
boundary is a place to authenticate, authorise, validate, and log — and every unmarked boundary is a place an attacker
will find first.

The practical method is to **draw the boundaries on the architecture diagram** and then threat-model each flow that
crosses one. STRIDE is a useful checklist of what can go wrong:

| Threat | Question at each boundary | Typical control |
|---|---|---|
| **S**poofing | Can someone pretend to be another user or service? | Authentication: tokens, mTLS, signed webhooks |
| **T**ampering | Can data be modified in transit or at rest? | TLS, signatures, checksums, immutability |
| **R**epudiation | Can someone deny having done something? | Audit logs that the actor can't alter |
| **I**nformation disclosure | Can data leak to someone who shouldn't see it? | Authorization per object, encryption, minimisation |
| **D**enial of service | Can someone exhaust capacity? | Rate limits, load shedding, quotas (Chapter 29) |
| **E**levation of privilege | Can someone gain rights they shouldn't have? | Least privilege, input validation, isolation |

Threat modelling isn't a one-off document; it's a design-review habit. For each new flow: who is on each side, what
can each side send, what's the worst thing a malicious sender could do, and which control stops it? The rest of this
chapter covers the controls that recur.

> 💼 **Interview** — Security rarely gets its own interview question, but strong candidates mention it unprompted at the
> right moments: "this endpoint takes an order ID, so it must check the order belongs to the caller"; "uploads go to a
> private bucket and are scanned before they're visible"; "the webhook is signed and replay-protected". Specific
> controls at specific boundaries signal far more than "we'd make it secure".

---

## 33.2 Authentication: Users and Services

**User authentication** belongs in a dedicated identity system — an identity provider (IdP) you run or buy — not
re-implemented in every service. It handles credentials (with strong password hashing such as Argon2id, breached-password
checks, and MFA), federation (SSO via OIDC or SAML), and issues tokens that the rest of the system trusts.

The common token architecture:

- A **short-lived access token** (minutes) that services validate locally, without calling the IdP on every request —
  typically a signed JWT carrying the subject, issuer, audience, expiry, and coarse claims.
- A **longer-lived refresh token**, stored securely and exchanged for new access tokens, which can be revoked and should
  be rotated on each use with reuse detection.
- In browsers, prefer **HTTP-only, secure, same-site cookies** (or a backend-for-frontend holding tokens) over tokens in
  JavaScript-readable storage, which any XSS can steal.

Local validation is what makes this scale — and it must be complete. Decoding a JWT is not validating it:

```js
// Validating a JWT means checking everything, not just decoding it. (HS256 here for brevity;
// across services prefer asymmetric keys such as ES256 so verifiers can't mint tokens.)
import { createHmac, timingSafeEqual } from 'node:crypto';
const b64 = (obj) => Buffer.from(JSON.stringify(obj)).toString('base64url');
const sign = (payload, key) => {
  const head = b64({ alg: 'HS256', typ: 'JWT' }), body = b64(payload);
  return `${head}.${body}.${createHmac('sha256', key).update(`${head}.${body}`).digest('base64url')}`;
};

function verifyJwt(token, { key, issuer, audience, now = Date.now() / 1000, leewaySec = 30 }) {
  const [h, p, s] = token.split('.');
  if (!s) throw new Error('malformed');
  const header = JSON.parse(Buffer.from(h, 'base64url'));
  if (header.alg !== 'HS256') throw new Error(`alg ${header.alg} not allowed`);   // never trust the token's choice
  const expected = createHmac('sha256', key).update(`${h}.${p}`).digest();
  const given = Buffer.from(s, 'base64url');
  if (given.length !== expected.length || !timingSafeEqual(given, expected)) throw new Error('bad signature');
  const claims = JSON.parse(Buffer.from(p, 'base64url'));
  if (claims.iss !== issuer) throw new Error('wrong issuer');
  if (![claims.aud].flat().includes(audience)) throw new Error('wrong audience');  // a token for another service
  if (claims.exp + leewaySec < now) throw new Error('expired');
  if (claims.nbf && claims.nbf - leewaySec > now) throw new Error('not yet valid');
  return claims;
}

const key = 'demo-secret', now = 1_800_000_000;
const good = sign({ sub: 'u42', iss: 'https://id.example.com', aud: 'orders-api', exp: now + 300 }, key);
const forOtherService = sign({ sub: 'u42', iss: 'https://id.example.com', aud: 'billing-api', exp: now + 300 }, key);
const [h, , sig] = good.split('.');
const tampered = `${h}.${b64({ sub: 'admin', iss: 'https://id.example.com', aud: 'orders-api', exp: now + 300 })}.${sig}`;
const noneAlg = `${b64({ alg: 'none' })}.${good.split('.')[1]}.x`;
for (const [name, tok] of Object.entries({ good, forOtherService, tampered, noneAlg })) {
  try { console.log(name.padEnd(16), 'ok', verifyJwt(tok, { key, issuer: 'https://id.example.com', audience: 'orders-api', now }).sub); }
  catch (e) { console.log(name.padEnd(16), 'rejected:', e.message); }
}
// good             ok u42
// forOtherService  rejected: wrong audience
// tampered         rejected: bad signature
// noneAlg          rejected: alg none not allowed
```

Each rejected case is a real attack: a token issued for another service replayed against this one (the audience check),
a modified payload (the signature), and the historic `alg: none` trick (an algorithm allow-list). In practice, use a
maintained JWT library configured with an explicit algorithm, issuer, and audience — and use **asymmetric** signatures
(RS256, ES256) when many services verify tokens, so that a compromised verifier can't mint new ones.

Short-lived access tokens trade revocation for scalability: a token stays valid until it expires even if the user logs
out or is disabled. Keep lifetimes short (5–15 minutes), revoke refresh tokens, and for high-risk actions (changing
email, moving money) re-check with the IdP or require re-authentication.

### Service identity and zero trust

Inside the network, the old model — "anything inside the firewall is trusted" — fails the first time an attacker gets
inside. **Zero trust** replaces network location with identity: every service authenticates every call, regardless of
where it comes from.

- **Workload identity**: each service gets a cryptographic identity issued by the platform (SPIFFE IDs, cloud workload
  identity, Kubernetes service account tokens), short-lived and rotated automatically — not a shared API key in an
  environment variable.
- **Mutual TLS** between services (often provided by a service mesh, Chapter 27) authenticates both ends and encrypts the
  traffic.
- **Authorization per call**: the receiving service checks not just "is this a valid service?" but "is *this* service
  allowed to call *this* endpoint?" — and, for user requests, still enforces the end user's permissions.

---

## 33.3 Authorization: Who May Do What to Which Thing

Authentication says who the caller is; **authorization** decides what they may do. The most common serious API
vulnerability — **broken object-level authorization** (BOLA, also called IDOR) — is an authorization check that's
missing for a specific object: the endpoint verifies the user is logged in, then returns order `918273` without checking
that order `918273` belongs to them. Every endpoint that takes an object identifier must authorise access to that object.

Three models, increasingly expressive:

| Model | Decision based on | Example | Strength | Weakness |
|---|---|---|---|---|
| **RBAC** (roles) | The user's roles | "Admins can delete projects" | Simple, auditable | Coarse; role explosion for per-object rules |
| **ABAC** (attributes) | Attributes of user, resource, and context | "Managers can approve expenses under €5,000 in their own department during business hours" | Expressive policies | Policies get hard to reason about and audit |
| **ReBAC** (relationships) | Relationships between subjects and objects | "Users who can view a folder can view its documents; members of a team that owns a project can edit it" | Natural for sharing, hierarchies, groups; scales to billions of objects | Needs a relationship store and evaluation engine |

Most products need RBAC for administrative functions plus per-object permissions for their data — and as soon as there's
sharing, groups, or nested containers (folders, workspaces, organisations), per-object permissions are relationships.
Google's **Zanzibar** paper described the architecture used for sharing across its products: permissions are stored as
relationship tuples (`doc:roadmap#editor@user:ana`), relations imply others (`owner` ⇒ `editor` ⇒ `viewer`), groups are
subjects too, and permissions inherit from parents. Open-source and commercial systems (SpiceDB, OpenFGA, and others)
implement the model. The real-world example in §33.9 builds a small version.

### Where decisions are made and enforced

- **Enforcement points** — the code paths that must ask "is this allowed?" — live in every service, close to the data.
- **The decision** can be embedded (a library evaluating local policy), or delegated to a central authorization service
  or policy engine (OPA, Cedar, a ReBAC service). Centralising decisions gives consistency and auditability; it also adds
  a dependency on the request path, which needs caching, low latency, and a fail-closed policy if it's unavailable.
- **Defence in depth**: coarse checks at the gateway (authenticated, right scopes), fine-grained checks in the service
  (this user, this object), and data-layer constraints where possible (row-level security for tenant isolation, Chapter 12).

> ⚠️ **Gotcha** — **Filtering lists is authorization too.** An endpoint that checks permissions on `GET /docs/:id` but
> returns every document in `GET /docs?query=…` — or in search results, exports, autocomplete suggestions, or
> notification emails — leaks the same data through a side door. Search indexes and caches (Chapters 16, 18) are common
> culprits: they're built for speed, not permission checks, and must be filtered per caller or partitioned by tenant.

---

## 33.4 Secrets and Keys

**Secrets** — database passwords, API keys, signing keys, certificates — are the credentials an attacker most wants. The
practices:

- **Never in source code, images, or plain environment files committed anywhere.** Store them in a secrets manager
  (Vault, cloud secret managers) with access control and audit logs, and inject them at runtime.
- **Prefer short-lived, automatically issued credentials** over long-lived static secrets: database credentials generated
  per service instance with an expiry, cloud credentials from workload identity, certificates issued for hours or days.
  A leaked credential that expires in an hour is a much smaller incident than one valid for years.
- **Rotate everything that can't be short-lived**, on a schedule and on demand, without downtime (support two valid
  versions during rotation).
- **Least privilege**: each service gets only the secrets it needs, scoped to only the actions it performs.
- **Detect leaks**: secret scanning in repositories and CI, and alerts on unusual use of credentials.

**Encryption keys** follow the same principles, with one more pattern worth knowing: **envelope encryption**. Encrypting
large amounts of data directly with a master key in a key management service (KMS) would mean sending all the data to the
KMS. Instead, each object or record is encrypted locally with its own **data key**, and only the small data key is
encrypted ("wrapped") by the master key, which never leaves the KMS or HSM:

```js
// Envelope encryption: each record gets its own data key; only the data key is encrypted by the
// master key (which, in production, never leaves a KMS or HSM).
import { randomBytes, createCipheriv, createDecipheriv } from 'node:crypto';

const seal = (key, plaintext, aad) => {
  const iv = randomBytes(12), c = createCipheriv('aes-256-gcm', key, iv);
  c.setAAD(Buffer.from(aad));                     // binds the ciphertext to its context (e.g. the record ID)
  const data = Buffer.concat([c.update(plaintext), c.final()]);
  return { iv, data, tag: c.getAuthTag() };
};
const open = (key, { iv, data, tag }, aad) => {
  const d = createDecipheriv('aes-256-gcm', key, iv);
  d.setAAD(Buffer.from(aad)); d.setAuthTag(tag);
  return Buffer.concat([d.update(data), d.final()]);
};

const kms = { masterKey: randomBytes(32),           // stand-in for a KMS: wrap/unwrap only, key never exported
  wrap(dataKey, ctx) { return seal(this.masterKey, dataKey, ctx); },
  unwrap(wrapped, ctx) { return open(this.masterKey, wrapped, ctx); } };

function encryptRecord(id, plaintext) {
  const dataKey = randomBytes(32);
  return { id, wrappedKey: kms.wrap(dataKey, `customer:${id}`), body: seal(dataKey, Buffer.from(plaintext), `customer:${id}`) };
}
const decryptRecord = (rec) => open(kms.unwrap(rec.wrappedKey, `customer:${rec.id}`), rec.body, `customer:${rec.id}`).toString();

const rec = encryptRecord('c-7', JSON.stringify({ taxId: 'PT123456789' }));
console.log(decryptRecord(rec));                                    // {"taxId":"PT123456789"}
try { decryptRecord({ ...rec, id: 'c-8' }); } catch { console.log('ciphertext moved to another record: rejected'); }
// ciphertext moved to another record: rejected
```

Envelope encryption makes key rotation cheap (re-wrap data keys, not data), limits the blast radius of one data key, and
enables **crypto-shredding** (Chapter 26): delete a customer's key, and all their encrypted data becomes unreadable. The
additional authenticated data (the record ID) means a ciphertext copied into another record fails to decrypt instead of
silently decrypting in the wrong place.

---

## 33.5 Protecting Data

- **Encrypt in transit everywhere**, including inside the network (§33.2's mTLS) — "internal" networks get breached and
  mirrored.
- **Encrypt at rest** with provider-managed keys as the baseline (it's nearly free), and with customer-managed or
  per-tenant keys where contracts or regulations require control.
- **Field-level encryption** or **tokenisation** for the most sensitive values (government IDs, card numbers, health data):
  encrypt or replace them with tokens at the edge of the system, so most services, logs, and analytics never see the raw
  value. Payment card data is the classic case — tokenising through a payment provider keeps most of the system out of
  compliance scope.
- **Minimise.** The safest data is data you don't collect or don't keep. Collect only what's needed, set retention
  periods, and delete on schedule (Chapter 12's partition-drop retention).
- **Mind the copies.** Data protection applies to logs, traces, caches, search indexes, analytics pipelines, backups, and
  support tools — the derived copies that outnumber the source of truth (Chapters 11, 30).

---

## 33.6 Isolation and Containment

Assume something will be breached, and design so the breach is small:

- **Tenant isolation** (Chapter 12): tenant IDs leading every key, row-level security, per-tenant encryption keys for
  sensitive tenants, and dedicated infrastructure for those who need it. Test isolation explicitly: run queries as one
  tenant and assert nothing from another comes back.
- **Least privilege for services**: each service's credentials allow only its own data and actions. A compromised
  recommendations service shouldn't be able to read payment records.
- **Network segmentation and egress controls**: services can reach only what they need, and outbound internet access goes
  through an allow-listed proxy. Egress control turns many exploits — exfiltration, command-and-control, **server-side
  request forgery (SSRF)** — from catastrophic into blocked.
- **SSRF in particular**: any feature that fetches a user-supplied URL (webhooks, link previews, image imports) can be
  aimed at internal services or the cloud metadata endpoint. Resolve and check the destination against an allow-list or
  deny private ranges *after* DNS resolution, block redirects to internal addresses, and run fetchers in isolated network
  segments.
- **The confused deputy**: a service acting on behalf of a user must use the *user's* permissions for data access, not its
  own broader ones. Pass the end-user identity (or a scoped token) downstream, and authorise against it.

---

## 33.7 The Edge, the Supply Chain, and Audit

**At the edge**, a CDN or WAF absorbs DDoS traffic and blocks known attack patterns; bot management and rate limits
(Chapter 29) slow credential stuffing and scraping. These are filters, not guarantees — the services behind them must
still validate input and enforce authorization.

**Validate every input at every boundary**: types, ranges, lengths, formats, and sizes, with allow-lists rather than
deny-lists; parameterised queries (never string-built SQL); output encoding for HTML; strict parsers for uploaded files
(Chapter 17). Treat data from other services, webhooks, queues, and LLMs as untrusted input too — trust boundaries exist
inside the system as well as at its edge.

**The software supply chain** is now a primary attack route: a compromised dependency, build system, or base image runs
inside every service that uses it. Controls:

- Pin and review dependencies; scan them for known vulnerabilities; limit install-time scripts.
- Build in isolated, reproducible pipelines; **sign artefacts** and verify signatures at deploy time.
- Produce a **software bill of materials (SBOM)** so "are we affected by this CVE?" can be answered in minutes.
- Give CI/CD the least privilege it needs — a pipeline that can deploy to production is a high-value target.

**Audit logs** record security-relevant actions — logins, permission changes, data exports, administrative actions, key
use — with who, what, when, from where, and the outcome. They must be append-only from the actor's point of view (shipped
to separate storage the actor can't modify), retained per policy, and actually monitored for anomalies. An audit log
that's written but never read only helps the post-mortem.

---

## 33.8 Common Mistakes and Best Practices

**Mistake: authorization checks only on authentication.** "Logged in" isn't "allowed to see order 918273"; every
object-taking endpoint needs an object-level check.

**Mistake: decoding tokens instead of validating them.** Signature, algorithm, issuer, audience, and expiry must all be
checked.

**Mistake: trusting the internal network.** Once an attacker is inside, unauthenticated internal APIs are open doors.

**Mistake: long-lived static secrets.** Leaked credentials stay valid for years; prefer short-lived, automatically issued
ones.

**Mistake: permissions enforced in the API but not in search, exports, and caches.** Derived copies leak what endpoints
protect.

**Mistake: fetching user-supplied URLs without SSRF protection.** The fetcher becomes a proxy into internal services and
metadata endpoints.

**Mistake: services using their own privileges for user requests.** The confused deputy lets users access data through a
service's broader rights.

**Best practice: draw trust boundaries on every design diagram.** Each crossing gets authentication, authorization,
validation, and logging.

**Best practice: identity everywhere.** Users via a central IdP with short-lived tokens; services via workload identity and
mTLS; every call authorised.

**Best practice: contain by default.** Least privilege, tenant isolation, egress control, and per-tenant keys keep a breach
small.

---

## 33.9 Real-World Example: A Relationship-Based Authorization Check

This is the core of a Zanzibar-style authorization engine: permissions stored as relationship tuples, relation rewrites
("owners are editors, editors are viewers"), groups as subjects ("members of team:design"), and inheritance from parent
objects ("viewers of a folder are viewers of its documents"), with a bounded, default-deny evaluation.

```js
// rebac.mjs — relationship-based authorization in the style of Google's Zanzibar: permissions are
// derived from relationship tuples (object#relation@subject), relation rewrites, group membership,
// and inheritance from parent objects.

export class Authz {
  #tuples = new Set();                                  // "doc:roadmap#editor@user:ana"
  constructor(schema) { this.schema = schema; }         // per type: relation → how else it's granted

  write(object, relation, subject) { this.#tuples.add(`${object}#${relation}@${subject}`); }
  delete(object, relation, subject) { this.#tuples.delete(`${object}#${relation}@${subject}`); }

  #subjectsOf(object, relation) {
    const prefix = `${object}#${relation}@`;
    return [...this.#tuples].filter((t) => t.startsWith(prefix)).map((t) => t.slice(prefix.length));
  }

  check(object, relation, user, depth = 0) {
    if (depth > 10) return false;                       // bound the walk: cycles and runaway nesting fail closed
    for (const subject of this.#subjectsOf(object, relation)) {
      if (subject === user) return true;                                       // direct grant
      const [group, groupRel] = subject.split('#');                            // "team:design#member"
      if (groupRel && this.check(group, groupRel, user, depth + 1)) return true; // via group membership
    }
    const type = object.split(':')[0];
    const rule = this.schema[type]?.[relation] ?? {};
    // Rewrites: "editors are also viewers", "owners are also editors"
    for (const implied of rule.impliedBy ?? []) if (this.check(object, implied, user, depth + 1)) return true;
    // Inheritance: "viewers of the parent folder are viewers of the doc"
    if (rule.fromParent) {
      for (const parent of this.#subjectsOf(object, 'parent')) {
        if (this.check(parent, rule.fromParent, user, depth + 1)) return true;
      }
    }
    return false;                                       // default deny
  }
}

const authz = new Authz({
  folder: { viewer: { impliedBy: ['editor'] }, editor: { impliedBy: ['owner'] } },
  doc: { viewer: { impliedBy: ['editor'], fromParent: 'viewer' }, editor: { impliedBy: ['owner'], fromParent: 'editor' } },
  team: {},
});
authz.write('team:design', 'member', 'user:bo');
authz.write('folder:q4', 'viewer', 'team:design#member');    // the whole design team can view the folder
authz.write('folder:q4', 'owner', 'user:ana');
authz.write('doc:roadmap', 'parent', 'folder:q4');
authz.write('doc:roadmap', 'editor', 'user:cy');

const ask = (u, rel) => `${u} ${rel}: ${authz.check('doc:roadmap', rel, u)}`;
console.log([ask('user:ana', 'editor'), ask('user:bo', 'viewer'), ask('user:bo', 'editor'), ask('user:cy', 'viewer'), ask('user:dee', 'viewer')].join(' | '));
// user:ana editor: true | user:bo viewer: true | user:bo editor: false | user:cy viewer: true | user:dee viewer: false
authz.delete('team:design', 'member', 'user:bo');            // Bo leaves the team…
console.log(ask('user:bo', 'viewer'));                       // …and loses access everywhere at once
// user:bo viewer: false
```

Ana edits the document because she owns its folder (owner ⇒ editor, inherited from the parent); Bo views it because his
team can view the folder; Cy views it because editors are viewers; Dee has no path and is denied. Removing one tuple —
Bo's team membership — revokes his access to every object the team could reach. A production system adds an indexed tuple
store, caching, consistency tokens so a check reflects a recent permission change, and "list objects user X can view" for
filtering search results.

The decision to defend is **deriving permissions from relationships at check time rather than storing each user's
effective permissions per object**. Precomputing "Bo can view doc:roadmap" makes checks a single lookup, which is tempting
at scale. But effective permissions change whenever anything along the path changes — a team membership, a folder's
sharing, a document moved to another folder — and keeping millions of precomputed rows in sync with every such change is
exactly the derived-data problem from Chapter 11, with the twist that a stale row is now a security hole: revoked access
that still works. Storing only the relationships people actually create (shares, memberships, parents) keeps one source of
truth, makes revocation immediate and complete, and leaves performance to caching and indexing — which is how
Zanzibar-style systems serve millions of checks per second without precomputing the closure.

---

## 33.10 Questions

**1.** MCQ — An endpoint verifies the user's token, then returns `/invoices/:id` for any ID. What's the vulnerability? a) XSS b) Broken object-level authorization (IDOR/BOLA) c) SQL injection d) CSRF
<details><summary>Answer</summary>**b)** Authentication isn't authorization: the endpoint must check the invoice belongs to (or is shared with) the caller.</details>

**2.** MCQ — Which JWT check stops a token issued for the billing service being replayed against the orders service? a) Expiry b) Signature c) Audience d) Algorithm
<details><summary>Answer</summary>**c)** The audience claim names the intended recipient; each service accepts only tokens for itself.</details>

**3.** MCQ — What does zero trust replace network location with? a) VPNs b) Identity, authenticated on every call c) Firewalls d) IP allow-lists
<details><summary>Answer</summary>**b)** Every request, internal or external, is authenticated and authorised by identity.</details>

**4.** MCQ — Which authorization model fits "anyone who can view a folder can view its documents, and members of a team that owns a project can edit it"? a) RBAC b) ABAC c) ReBAC d) ACL files on disk
<details><summary>Answer</summary>**c)** Permissions follow relationships: inheritance, groups, and ownership.</details>

**5.** MCQ — What's the main advantage of envelope encryption? a) Faster disks b) Data is encrypted locally with data keys; only small keys go to the KMS, so rotation and per-record isolation are cheap c) No keys needed d) Compression
<details><summary>Answer</summary>**b)** The master key never leaves the KMS, and rotating it means re-wrapping data keys, not re-encrypting data.</details>

**6.** MCQ — Which control most directly limits SSRF damage? a) WAF rules on inbound traffic b) Egress allow-listing and blocking private address ranges after DNS resolution c) Longer timeouts d) More replicas
<details><summary>Answer</summary>**b)** The fetcher can't reach internal services or metadata endpoints.</details>

**7.** Short — Why prefer asymmetric signatures (RS256, ES256) for tokens verified by many services?
<details><summary>Answer</summary>With a shared HMAC secret, every verifier can also mint tokens; compromising any one service lets an attacker forge identities. With asymmetric keys, verifiers hold only the public key.</details>

**8.** Short — What's the trade-off of short-lived access tokens validated locally?
<details><summary>Answer</summary>Local validation scales and avoids an IdP dependency per request, but a token stays valid until expiry even after logout or disablement. Keep lifetimes short, revoke refresh tokens, and re-check for high-risk actions.</details>

**9.** Short — What is the confused deputy problem?
<details><summary>Answer</summary>A service with broad privileges acting on behalf of a user is tricked into using its own privileges for an action the user isn't allowed to perform. The fix is to propagate the end user's identity and authorise data access against it.</details>

**10.** Short — Why is filtering search results and exports an authorization concern?
<details><summary>Answer</summary>They return the same data as object endpoints through a different path; if they don't apply per-caller permission filters, they leak whatever the object endpoints protect.</details>

**11.** Short — Name three supply-chain controls.
<details><summary>Answer</summary>Any three of: pinned and reviewed dependencies with vulnerability scanning; isolated, reproducible builds; signed artefacts verified at deploy; an SBOM; least-privilege CI/CD credentials; limiting install-time scripts.</details>

**12.** Output — Using §33.2's `verifyJwt`, a token has `exp: now - 10` and the default leeway. Is it accepted?
<details><summary>Answer</summary>**Yes.** With a 30-second leeway, `exp + 30 < now` is false (it expired only 10 s ago). At `exp: now - 31` it would be rejected as expired.</details>

**13.** Output — Using §33.9's engine (after Bo was removed from the team), add `authz.write('folder:q4', 'editor', 'team:design#member')` and `authz.write('team:design', 'member', 'user:dee')`. What are `check('doc:roadmap', 'editor', 'user:dee')` and `check('doc:roadmap', 'viewer', 'user:dee')`?
<details><summary>Answer</summary>**`true` and `true`.** Dee is a team member; the team edits the folder; the doc inherits folder editors; and editors are viewers.</details>

**14.** Output — A permission check walks at most 10 levels in §33.9's engine. A document is nested 12 folders deep, with access granted only at the top folder. What does `check` return for that user?
<details><summary>Answer</summary>**`false`.** The depth limit is exceeded and the check fails closed. Real systems set limits above realistic nesting and alert when they're hit.</details>

**15.** Output — An access token lives 15 minutes; a user is disabled at 10:02, and their last token was issued at 09:55. Until when could that token still be accepted (ignoring leeway)?
<details><summary>Answer</summary>**10:10.** It expires 15 minutes after issue, so it stays valid 8 minutes past the disablement unless high-risk actions re-check with the IdP.</details>

**16.** Debug — Users could see other tenants' records in the new "global search" feature, though the record API correctly enforced tenancy. The search index contains all tenants' documents. Cause and fix?
<details><summary>Answer</summary>The search path skipped authorization: queries ran across the whole index without a mandatory tenant filter. Apply the tenant filter server-side from the authenticated context on every search (or partition indexes per tenant), and test search as a different tenant.</details>

**17.** Debug — A link-preview feature was used to fetch `http://169.254.169.254/...`, exposing cloud credentials. What happened, and what are the fixes?
<details><summary>Answer</summary>SSRF: the server fetched an attacker-chosen URL targeting the instance metadata service. Resolve and validate destinations (deny private, link-local, and metadata ranges after DNS resolution and on every redirect), enforce metadata-service protections (session tokens/hop limits), run fetchers in an isolated network with egress allow-lists, and give the fetcher no cloud credentials.</details>

**18.** Debug — A leaked API key for a third-party email service, committed to a public repository two years ago, was used to send phishing emails from the company's domain. The key had never been rotated. What should change?
<details><summary>Answer</summary>Secrets belong in a secrets manager, not code; secret scanning should run in CI and on repositories; keys should be scoped narrowly and rotated on a schedule; and usage anomalies should alert. Immediately: revoke the key, rotate, review sending logs, and check for other committed secrets.</details>

**19.** Debug — An internal admin service accepts requests from anything on the private network without authentication. An attacker who compromised a marketing web app used it to export customer data. What's the architectural fix?
<details><summary>Answer</summary>Zero trust: require workload identity and mTLS on every internal call, authorise which services may call which endpoints, require end-user identity for user-data access, segment the network so the marketing app can't reach admin services, and audit-log exports.</details>

**20.** Code — Write `isPrivateAddress(ip)` for IPv4 that returns `true` for loopback, private (RFC 1918), link-local, and the cloud metadata address.
<details><summary>Answer</summary>

```js
function isPrivateAddress(ip) {
  const p = ip.split('.').map(Number);
  if (p.length !== 4 || p.some((n) => !(n >= 0 && n <= 255))) return true;   // unparsable: treat as unsafe
  return p[0] === 10 || p[0] === 127 || p[0] === 0 ||
    (p[0] === 172 && p[1] >= 16 && p[1] <= 31) ||
    (p[0] === 192 && p[1] === 168) ||
    (p[0] === 169 && p[1] === 254);                                            // link-local, incl. metadata
}
console.log(['10.1.2.3', '172.20.0.1', '169.254.169.254', '93.184.216.34'].map(isPrivateAddress)); // [ true, true, true, false ]
```
Check the *resolved* address (and every redirect's), handle IPv6 too, and prefer an allow-list where the feature permits.</details>

**21.** Code — Write `authorizeObject(user, obj, action)` for an orders API: admins may do anything; owners may read and cancel their own orders; support staff may read orders in their region; everything else is denied.
<details><summary>Answer</summary>

```js
function authorizeObject(user, order, action) {
  if (user.roles.includes('admin')) return true;
  if (order.customerId === user.id && ['read', 'cancel'].includes(action)) return true;
  if (user.roles.includes('support') && action === 'read' && order.region === user.region) return true;
  return false; // default deny: anything not explicitly allowed is refused
}
const order = { customerId: 'u1', region: 'eu' };
console.log(authorizeObject({ id: 'u1', roles: [] }, order, 'cancel'), authorizeObject({ id: 'u2', roles: ['support'], region: 'us' }, order, 'read')); // true false
```
</details>

**22.** Code — Write `rotateWrappedKey(record, oldKms, newKms, ctx)` that re-wraps a record's data key under a new master key without touching the encrypted body.
<details><summary>Answer</summary>

```js
function rotateWrappedKey(record, oldKms, newKms, ctx) {
  const dataKey = oldKms.unwrap(record.wrappedKey, ctx);   // the only time the data key is in memory
  return { ...record, wrappedKey: newKms.wrap(dataKey, ctx), keyVersion: (record.keyVersion ?? 1) + 1 };
}
```
Rotating the master key means re-wrapping a few bytes per record — the body, possibly gigabytes, is untouched.</details>

**23.** Scenario — You're designing sharing for a document product: personal docs, team workspaces, nested folders, and link sharing ("anyone with the link can view"). What authorization architecture do you choose?
<details><summary>Answer</summary>ReBAC: a relationship store (SpiceDB/OpenFGA or a Zanzibar-style service) with types for user, team, workspace, folder, and doc; relations owner/editor/viewer with rewrites; inheritance from parents; team membership as group subjects; and link sharing as a special subject (`doc#viewer@link:token`) checked when a valid link token is presented. Every service checks before reading a doc; search and listing use "list objects the user can view" or permission-aware filtering; consistency tokens ensure revocations apply promptly.</details>

**24.** Scenario — A healthcare startup stores patient records in a multi-tenant SaaS and must limit the damage of any single breach. What do you put in place?
<details><summary>Answer</summary>Per-tenant encryption keys with envelope encryption (and crypto-shredding on offboarding); row-level security and tenant-leading keys; field-level encryption or tokenisation for the most sensitive fields so most services and logs never see them; least-privilege service identities with mTLS; egress controls; short-lived credentials from a secrets manager; audit logs of every record access shipped to separate storage and monitored; data minimisation and retention; and regular isolation tests run as other tenants.</details>

---

## Chapter Summary

- Security architecture is boundary work: draw trust boundaries on the diagram, threat-model every crossing with STRIDE,
  and give each crossing authentication, authorization, validation, and logging.
- Centralise user authentication in an IdP with short-lived, fully validated tokens (signature, algorithm, issuer,
  audience, expiry); give services workload identity and mTLS, and trust identity rather than network location.
- Authorise every object access — including in lists, search, exports, and caches — using RBAC for administration and
  relationship-based models where sharing, groups, and hierarchy exist.
- Keep secrets out of code, prefer short-lived automatically issued credentials, rotate the rest, and use envelope
  encryption with a KMS for data keys.
- Protect data in transit and at rest, minimise and tokenise sensitive fields, and remember that derived copies need the
  same protection.
- Contain breaches with tenant isolation, least privilege, egress controls, SSRF defences, and end-user identity propagated
  downstream; secure the supply chain; and keep audit logs that actors can't alter and someone actually reads.

---

## Practice Exercises

1. Draw the trust boundaries on an architecture diagram you know and run STRIDE on every crossing; list the controls in place
   and the gaps.
2. Audit ten endpoints that take object IDs and confirm each performs an object-level authorization check; write a test that
   calls each as a different user.
3. Configure your JWT library to enforce algorithm, issuer, audience, and expiry, and write tests for each rejection case
   from §33.2.
4. Read the Zanzibar paper's sections on relation tuples, userset rewrites, and the "new enemy" problem (consistency tokens),
   and explain how a stale check could grant access after revocation.
5. Implement SSRF protection for a URL-fetching feature: resolution-time checks, redirect checks, and an isolated egress
   path; test it against internal and metadata addresses.

---

## Mini Project: A Permissions Service

Build a small Zanzibar-style permissions service that other services call to authorise access to documents in a sharing
product, with the operational properties a production system needs.

- A tuple store (PostgreSQL) with indexes for "subjects of object#relation" and "objects related to subject", and a schema
  with rewrites, groups, and parent inheritance.
- `check`, `expand` (why does this user have access?), and `listObjects` (which docs can this user view?) APIs.
- Consistency tokens: writes return a token, and checks can require "at least as fresh as this token" to avoid stale grants.
- Caching of check results with invalidation on relevant tuple changes, and a depth limit that fails closed.
- Tests for revocation via group removal and folder moves, plus an audit log of every permission change.

---

**Next:** [Chapter 34 — The Design Interview Method →](34-design-interview-method.md)
