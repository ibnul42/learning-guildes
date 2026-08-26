# Chapter 23 — Securing a NestJS API

## Learning Objectives

- Apply security headers, CORS, and rate limiting with settings that hold in production
- Prevent SQL and NoSQL injection, prototype pollution, and mass assignment
- Decide whether your API needs CSRF protection, and implement it if so
- Handle file uploads and redirects without creating well-known vulnerabilities
- Build a dependency and secret-handling process that survives contact with a real team

---

## 23.1 The Baseline

Four lines in `main.ts` close most of the low-effort attack surface (→ [Ch 3](03-first-application.md)):

```ts
import helmet from 'helmet';

app.use(helmet());                     // security headers
app.enableCors({
  origin: config.getOrThrow<string>('CORS_ORIGINS').split(','),
  credentials: true,
  maxAge: 86_400,
});
app.set('trust proxy', 1);             // exactly one hop — see §23.3
app.useGlobalPipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true }));
```

Helmet sets a dozen headers; the ones that matter most are `X-Content-Type-Options: nosniff`, `Strict-Transport-Security`, and a Content Security Policy.

**CSP is Helmet's default that people disable.** It is restrictive, it breaks inline scripts, and the common "fix" is to turn it off:

```ts
// ❌ Disables the strongest single defence against XSS
app.use(helmet({ contentSecurityPolicy: false }));

// ✅ Configure it for the app you actually have
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],           // no 'unsafe-inline' — that defeats the point
      objectSrc: ["'none'"],
      frameAncestors: ["'none'"],      // clickjacking
    },
  },
}));
```

For a pure JSON API serving no HTML, CSP matters less — but Swagger UI, error pages, and any redirect target are HTML, and those are exactly where an injected script would run.

> ⚠️ **Gotcha** — `enableCors({ origin: true })` reflects whatever `Origin` the caller sends, which permits *every* origin. With `credentials: true` this exposes authenticated endpoints to any website. It is a development convenience that reaches production regularly because nothing fails when it is wrong.

---

## 23.2 Injection

**SQL injection** is prevented by binding parameters, never by escaping (→ [Ch 17](17-typeorm.md)):

```ts
// ❌ Injection
.where(`task.title = '${search}'`)
await this.prisma.$queryRawUnsafe(`SELECT * FROM tasks WHERE owner = '${id}'`);

// ✅ Bound
.where('task.title = :search', { search })
await this.prisma.$queryRaw`SELECT * FROM tasks WHERE owner = ${id}`;
```

Prisma's tagged-template `$queryRaw` parameterises interpolated values automatically; `$queryRawUnsafe` does not, and its name is the warning.

**Column and table names cannot be parameterised**, so anything client-supplied that lands in `ORDER BY`, a column list, or a table name needs an allowlist. This is the injection vector that survives an ORM.

**Prototype pollution** comes from merging untrusted objects:

```ts
// ❌ A body of {"__proto__": {"isAdmin": true}} pollutes Object.prototype
Object.assign(target, req.body);
merge(defaults, req.body);

// ✅ ValidationPipe with whitelist strips unknown keys, including __proto__
// ✅ Or construct explicitly
const safe = { title: dto.title, description: dto.description };
```

`ValidationPipe` with `whitelist: true` is the systemic fix, because it drops any property the DTO does not declare.

**Mass assignment** is the same class of bug with a business impact. Passing a DTO straight into `save()` lets a client set any column the DTO permits, so the DTO — not the entity — is the security boundary:

```ts
// ❌ If CreateUserDto has a `role` field, a client can make themselves admin
await this.repo.save(dto);

// ✅ Only the fields this operation may set
await this.repo.save({ email: dto.email, name: dto.name, role: Role.Member });
```

---

## 23.3 Rate Limiting

```bash
npm i @nestjs/throttler
```

```ts
ThrottlerModule.forRoot({
  throttlers: [
    { name: 'short', ttl: 1_000, limit: 10 },     // burst
    { name: 'long', ttl: 60_000, limit: 100 },    // sustained
  ],
});

// Global, but overridable per route
{ provide: APP_GUARD, useClass: ThrottlerGuard }
```

```ts
@Throttle({ short: { ttl: 60_000, limit: 5 } })   // login is worth 5/minute, not 100
@Post('login')
login() {}

@SkipThrottle()
@Get('health')
health() {}
```

Two things make rate limiting real rather than decorative.

**The default in-memory storage does not work across replicas.** Three instances with a limit of 100 permit 300. Use the Redis storage provider in any multi-instance deployment.

**Trusting `X-Forwarded-For` incorrectly makes it bypassable.** `app.set('trust proxy', true)` trusts the entire chain, so a client can prepend a forged address and get a fresh bucket per request. Trust exactly the number of proxies you actually have:

```ts
app.set('trust proxy', 1);     // one load balancer in front
```

Rate-limit by principal where one exists and by IP otherwise, so a shared corporate NAT does not exhaust one bucket for everybody.

---

## 23.4 CSRF

CSRF matters when the browser attaches credentials **automatically** — that means cookies. A pure token API sending `Authorization: Bearer` is not vulnerable, because the browser does not attach that header to a cross-site request.

If you use cookie authentication (→ [Ch 21](21-authentication.md)), you need protection:

**`SameSite=Strict` or `Lax` on the cookie** is the primary defence and covers most cases.

**A double-submit or synchroniser token** is the belt-and-braces addition:

```bash
npm i csrf-csrf
```

```ts
const { doubleCsrfProtection } = doubleCsrf({
  getSecret: () => config.getOrThrow('CSRF_SECRET'),
  cookieName: '__Host-csrf',
  cookieOptions: { sameSite: 'strict', secure: true, path: '/' },
});
app.use(doubleCsrfProtection);
```

> ⚠️ **Gotcha** — `csurf` is deprecated and unmaintained, and a great deal of NestJS material still recommends it. Use a maintained alternative such as `csrf-csrf`, and check the advisory status of any security middleware before adopting it — security packages that stop being maintained are worse than none, because they create the appearance of protection.

---

## 23.5 The Ones That Get Missed

**Open redirects.** Redirecting to a client-supplied URL turns your domain into a phishing launchpad:

```ts
// ❌ ?next=https://evil.example lands the user there, from your domain
@Redirect() go(@Query('next') next: string) { return { url: next }; }

// ✅ Allowlist, or accept only relative paths
const ALLOWED = new Set(['/dashboard', '/settings']);
if (!ALLOWED.has(next)) throw new BadRequestException('Invalid redirect');
```

**SSRF.** A URL supplied by a client and fetched by your server reaches your internal network — including cloud metadata endpoints that hand out credentials. Resolve the hostname and reject private ranges before fetching, and do it *after* resolution, since a hostname can resolve to `169.254.169.254`.

**Unbounded input.** No body limit, no page-size ceiling, and no array-length cap are each a denial-of-service vector that looks like a missing feature (→ [Ch 14](14-pipes-validation.md)).

**Uploads.** Trusting the client-supplied MIME type or filename allows path traversal and stored XSS. Validate by magic bytes, generate your own filename, and serve user uploads from a separate origin (→ [Ch 31](31-file-uploads-streaming.md)).

**Dependencies.** `npm audit` in CI, a lockfile committed, and automated update PRs. Most real-world compromises arrive through a dependency, not through your code.

**Error responses.** Stack traces and driver messages leak paths, queries, and credentials — return a code and a correlation id (→ [Ch 15](15-exception-filters.md)).

> 💼 **Interview** — "How would you secure a NestJS API?" A list of packages is a weak answer. A strong one walks the request: transport (HSTS, TLS), origin (CORS), rate limit, authentication, authorization, input validation, output encoding, and error handling — naming what each layer catches and what it misses.

---

## 23.6 Common Mistakes and Best Practices

**Mistake: `origin: true` with credentials.** Every origin is permitted, including an attacker's.

**Mistake: disabling CSP because it broke something.** It is the strongest single defence against XSS.

**Mistake: `trust proxy: true`.** Clients can forge `X-Forwarded-For` and bypass IP rate limiting.

**Mistake: in-memory throttling across replicas.** Three instances multiply your limit by three.

**Mistake: `$queryRawUnsafe` or template-interpolated SQL.** Injection, regardless of the ORM.

**Mistake: passing a DTO into `save()`.** Any field the DTO carries becomes settable by the client.

**Mistake: using `csurf`.** It is deprecated and unmaintained.

**Best practice: `whitelist` and `forbidNonWhitelisted` globally.** They close mass assignment and prototype pollution in one move.

**Best practice: rate-limit by principal, falling back to IP.** IP-only punishes shared networks and is forgeable if proxy trust is wrong.

**Best practice: allowlist anything that reaches SQL identifiers, a redirect, or a fetch.** Values can be bound; identifiers and URLs cannot.

---

## 23.7 Real-World Example: A Security Baseline Module

The controls that belong in one reviewable place, with the settings that are commonly wrong made explicit.

```ts
// src/security/security.config.ts
export default registerAs('security', () => ({
  corsOrigins: (process.env.CORS_ORIGINS ?? '').split(',').filter(Boolean),
  trustedProxyHops: Number(process.env.TRUSTED_PROXY_HOPS ?? 1),
  bodyLimit: process.env.BODY_LIMIT ?? '100kb',
  isProduction: process.env.NODE_ENV === 'production',
}));
```

```ts
// src/security/security.setup.ts
import { INestApplication } from '@nestjs/common';
import { NestExpressApplication } from '@nestjs/platform-express';
import helmet from 'helmet';
import * as express from 'express';

export function applySecurity(
  app: NestExpressApplication,
  config: ConfigType<typeof securityConfig>,
): void {
  app.use(
    helmet({
      contentSecurityPolicy: {
        directives: {
          defaultSrc: ["'self'"],
          scriptSrc: ["'self'"],          // no 'unsafe-inline'
          styleSrc: ["'self'", "'unsafe-inline'"],   // Swagger UI needs this
          imgSrc: ["'self'", 'data:'],
          objectSrc: ["'none'"],
          frameAncestors: ["'none'"],     // clickjacking
          upgradeInsecureRequests: config.isProduction ? [] : null,
        },
      },
      hsts: config.isProduction
        ? { maxAge: 31_536_000, includeSubDomains: true, preload: true }
        : false,                          // never send HSTS from localhost
    }),
  );

  // Deny by default. An empty CORS_ORIGINS must NOT mean "allow everything",
  // which is what `origin: true` would do and what a missing env var invites.
  app.enableCors({
    origin: config.corsOrigins.length > 0 ? config.corsOrigins : false,
    credentials: true,
    methods: ['GET', 'POST', 'PATCH', 'DELETE'],
    maxAge: 86_400,
  });

  // Exactly the number of proxies in front of us. `true` trusts the whole
  // chain, letting a client forge X-Forwarded-For and reset its rate bucket.
  app.set('trust proxy', config.trustedProxyHops);

  // An unbounded body is a denial-of-service vector; the default is 100kb
  // and raising it globally to serve one upload endpoint is a common error.
  app.use(express.json({ limit: config.bodyLimit }));
  app.use(express.urlencoded({ extended: true, limit: config.bodyLimit }));
}
```

```ts
// src/security/url-safety.service.ts
import { Injectable, BadRequestException } from '@nestjs/common';
import { lookup } from 'node:dns/promises';
import { isIP } from 'node:net';

/**
 * SSRF defence. The check must happen AFTER DNS resolution: an attacker
 * controls the hostname, and `internal.evil.com` can resolve to
 * 169.254.169.254 — the cloud metadata endpoint that returns credentials.
 */
@Injectable()
export class UrlSafetyService {
  private static readonly BLOCKED = [
    /^127\./, /^10\./, /^192\.168\./, /^169\.254\./,
    /^172\.(1[6-9]|2\d|3[01])\./, /^0\./, /^::1$/, /^fc00:/, /^fe80:/,
  ];

  async assertSafe(rawUrl: string): Promise<URL> {
    let url: URL;
    try {
      url = new URL(rawUrl);
    } catch {
      throw new BadRequestException('Invalid URL');
    }

    if (!['http:', 'https:'].includes(url.protocol)) {
      throw new BadRequestException('Only http and https are permitted');
    }

    const address = isIP(url.hostname)
      ? url.hostname
      : (await lookup(url.hostname)).address;

    if (UrlSafetyService.BLOCKED.some((r) => r.test(address))) {
      throw new BadRequestException('URL resolves to a private address');
    }
    return url;
  }
}
```

```ts
// src/security/redirect.controller.ts
@Controller()
export class RedirectController {
  // Relative paths only. An allowlist of absolute URLs is the alternative;
  // accepting a client-supplied absolute URL makes your domain a phishing hop.
  private static readonly ALLOWED = new Set(['/dashboard', '/settings', '/tasks']);

  @Public()
  @Get('go')
  @Redirect()
  go(@Query('next') next: string) {
    if (!next?.startsWith('/') || next.startsWith('//') || !RedirectController.ALLOWED.has(next)) {
      throw new BadRequestException('Invalid redirect target');
    }
    return { url: next, statusCode: 302 };
  }
}
```

The decision worth defending is treating an empty `CORS_ORIGINS` as *deny all* rather than *allow all*. Allowing everything when configuration is absent is the friendlier behaviour during development and it is precisely the failure mode that reaches production, because nothing breaks when the variable is missing — the API works, the frontend works, and the only observable difference is that every website on the internet can now make credentialed requests to it. Denying by default breaks loudly and immediately in whatever environment forgot the variable, which is the correct place to discover it. The `next.startsWith('//')` check in the redirect controller is the same principle in miniature: `//evil.example` is a protocol-relative URL that passes a naive `startsWith('/')` test and navigates off-site, so the guard has to reject the case that *looks* safe. Security defaults should fail closed, and validation should reject the near-miss rather than the obvious case.

---

## 23.8 Questions

**1.** MCQ — `enableCors({ origin: true })` permits: a) no origins b) same origin c) every origin d) configured origins
<details><summary>Answer</summary>**c)** It reflects whatever `Origin` is sent.</details>

**2.** MCQ — `app.set('trust proxy', true)` allows a client to: a) nothing b) forge `X-Forwarded-For` c) bypass CORS d) set cookies
<details><summary>Answer</summary>**b)** Which defeats IP-based rate limiting.</details>

**3.** MCQ — Which is safe from SQL injection? a) `$queryRawUnsafe` with interpolation b) tagged `$queryRaw` c) template literals in `where()` d) escaping quotes
<details><summary>Answer</summary>**b)** Prisma's tagged template parameterises interpolated values.</details>

**4.** MCQ — Prototype pollution is systemically prevented by: a) `Object.freeze` b) `ValidationPipe` with `whitelist` c) HTTPS d) CSP
<details><summary>Answer</summary>**b)** Unknown properties, including `__proto__`, are stripped.</details>

**5.** MCQ — CSRF protection is needed when credentials are carried by: a) bearer headers b) cookies c) query strings d) custom headers
<details><summary>Answer</summary>**b)** The browser attaches cookies automatically to cross-site requests.</details>

**6.** MCQ — `csurf` should not be used because it is: a) slow b) deprecated and unmaintained c) Express-only d) paid
<details><summary>Answer</summary>**b)** Use a maintained alternative such as `csrf-csrf`.</details>

**7.** Short — Why must an SSRF check happen after DNS resolution?
<details><summary>Answer</summary>The attacker controls the hostname, and a public-looking name can resolve to a private or metadata address — so only the resolved IP tells you where the request will actually go.</details>

**8.** Short — Why is in-memory rate limiting wrong across replicas?
<details><summary>Answer</summary>Each instance keeps its own counter, so N instances permit N times the intended limit. Use a shared store.</details>

**9.** Short — Why is a DTO the security boundary for mass assignment?
<details><summary>Answer</summary>Passing a DTO into `save()` lets a client set any column the DTO declares, so what the DTO permits is exactly what the client can write.</details>

**10.** Short — Why does `next.startsWith('/')` fail to prevent an open redirect?
<details><summary>Answer</summary>`//evil.example` is protocol-relative: it starts with `/` and navigates off-site. The check must also reject a leading `//`.</details>

**11.** Short — Why is disabling CSP a significant loss for a JSON API with Swagger?
<details><summary>Answer</summary>Swagger UI, error pages, and redirect targets are HTML rendered on your origin, and those are exactly where an injected script would execute.</details>

**12.** Output — Three replicas, `ThrottlerModule` with the default storage and `limit: 100`. How many requests get through per window?
<details><summary>Answer</summary>Up to 300 — each instance counts independently.</details>

**13.** Output — A DTO includes `role` and the service calls `repo.save(dto)`. A client posts `{"email":"a@b.c","role":"admin"}`. What happens?
<details><summary>Answer</summary>They become an admin. The DTO permitted the field and it was written straight through.</details>

**14.** Output — With `whitelist: true`, a body of `{"__proto__": {"isAdmin": true}}` is posted. What reaches the handler?
<details><summary>Answer</summary>An empty object for that key — the undeclared property is stripped before it can pollute anything.</details>

**15.** Output — `CORS_ORIGINS` is unset and the code uses `origin: config.corsOrigins.length ? config.corsOrigins : false`. What can a browser on another origin do?
<details><summary>Answer</summary>Nothing — CORS is denied. That is the intended fail-closed behaviour.</details>

**16.** Debug — Rate limiting has no effect against a determined attacker. Which two settings do you check?
<details><summary>Answer</summary>The storage (in-memory across replicas multiplies the limit) and `trust proxy` (trusting the whole chain lets the attacker forge their IP).</details>

**17.** Debug — A user reports a link on your domain sent them to a phishing site. What is the vulnerability?
<details><summary>Answer</summary>An open redirect taking a client-supplied target. Restrict to an allowlist of relative paths and reject protocol-relative `//` values.</details>

**18.** Debug — An internal service was reached from the public API by supplying a URL. What is missing?
<details><summary>Answer</summary>SSRF protection: resolve the hostname and reject private, loopback, and link-local ranges before fetching.</details>

**19.** Debug — CSP was disabled because Swagger UI failed to render. Better fix?
<details><summary>Answer</summary>Keep CSP and permit exactly what Swagger needs — typically `'unsafe-inline'` for styles only — rather than removing the policy entirely.</details>

**20.** Debug — A JSON API using bearer tokens added CSRF tokens and broke every client. Was the change necessary?
<details><summary>Answer</summary>No. Bearer tokens are not attached automatically by the browser, so there is no CSRF exposure. The protection was unnecessary and the breakage was pure cost.</details>

**21.** Code — Write a CORS configuration that denies by default.
<details><summary>Answer</summary>

```ts
const origins = (process.env.CORS_ORIGINS ?? '').split(',').filter(Boolean);
app.enableCors({
  origin: origins.length > 0 ? origins : false,   // missing config → deny
  credentials: true,
  maxAge: 86_400,
});
```
</details>

**22.** Code — Write an SSRF guard that rejects private addresses after resolution.
<details><summary>Answer</summary>

```ts
async assertSafe(raw: string): Promise<URL> {
  const url = new URL(raw);
  if (!['http:', 'https:'].includes(url.protocol)) throw new BadRequestException();
  const { address } = await lookup(url.hostname);        // resolve first
  if (/^(127\.|10\.|192\.168\.|169\.254\.|::1)/.test(address)) {
    throw new BadRequestException('Private address');
  }
  return url;
}
```
</details>

**23.** Code — Rate-limit login harder than the global default.
<details><summary>Answer</summary>

```ts
@Throttle({ short: { ttl: 60_000, limit: 5 } })
@UseGuards(ThrottlerGuard, AuthGuard('local'))    // throttle BEFORE hashing
@Post('login')
login() {}
```
Ordering matters: the throttle guard must run before the expensive password verification.
</details>

**24.** Scenario — A penetration test reports "missing security headers" and a teammate proposes adding Helmet with all defaults and closing the ticket. Evaluate it.
<details><summary>Answer</summary>Adding Helmet is right and closing the ticket there is premature. The defaults are a good baseline, but two of them need decisions rather than acceptance: HSTS must not be sent from non-production origins, and CSP needs directives matching what the application actually loads, or someone will disable it the first time Swagger breaks — which is worse than never enabling it. Headers are also the cheapest finding in any report and rarely the most serious; a test that surfaced only headers either had limited scope or the reviewer stopped early. Use the ticket to check the categories a header scan cannot see: authorization on object identifiers, rate limiting under a forged `X-Forwarded-For`, and what your error responses reveal.</details>

**25.** Scenario — Your API allows users to supply a webhook URL that you call on events. What is the full threat model?
<details><summary>Answer</summary>SSRF is the headline: the URL must be resolved and checked against private, loopback, link-local, and metadata ranges, and re-checked on every call rather than once at registration, because DNS can be repointed afterwards. Redirects must be disabled or followed manually with the same check at each hop, since a public URL can 302 to an internal one. Beyond SSRF, treat it as an outbound denial-of-service surface: bound the timeout, cap response size, and cap concurrency per tenant, or a slow endpoint ties up your workers. Sign the payload with a per-subscriber secret so the receiver can verify authenticity, and never forward internal identifiers or auth headers. Finally, deliver from a queue with retries and a dead-letter path rather than inline, so a hostile or dead endpoint cannot slow the request that triggered it.</details>

---

## Chapter Summary

- Helmet, an explicit CORS origin list, correct proxy trust, and a whitelisting `ValidationPipe` close most of the cheap attack surface in four lines.
- `origin: true` and `trust proxy: true` are the two settings that silently permit everything and forge everything — both are development conveniences that reach production because nothing fails.
- Values in SQL can be bound; identifiers, redirect targets, and fetched URLs cannot, so each needs an allowlist.
- `whitelist: true` closes mass assignment and prototype pollution together, which makes the DTO the real security boundary.
- CSRF applies to cookie-carried credentials only — adding it to a bearer-token API is pure cost — and `csurf` is deprecated.
- SSRF checks must run after DNS resolution and on every call, because the attacker controls the hostname and DNS can be repointed later.

---

## Practice Exercises

1. Set `origin: true` with credentials and make a credentialed request from a different origin to see what is permitted.
2. Deploy three instances with in-memory throttling and measure the effective limit.
3. Post `{"__proto__": {"polluted": true}}` to an endpoint without `whitelist`, then with it.
4. Build an endpoint that fetches a client-supplied URL and reach a local service through it, then add resolution-based blocking.
5. Enable CSP, break Swagger UI, and fix it by narrowing a directive rather than disabling the policy.

---

## Mini Project: A Security Baseline You Can Audit

Build a security module that makes the dangerous settings visible.

**Requirements:**

- One `applySecurity()` function owning headers, CORS, proxy trust, and body limits, with every value from validated config.
- CORS and CSP both fail closed when configuration is absent, proven by tests.
- Distributed rate limiting with a per-principal key and an IP fallback, correct behind one proxy hop.
- An SSRF-safe fetch helper used by every outbound call to a client-supplied URL, re-checked per call.
- A CI job running `npm audit`, plus a test asserting no error response contains a stack trace or connection string.

---

**Next:** [Chapter 24 — GraphQL →](24-graphql.md)
