# Chapter 21 — Authentication

## Learning Objectives

- Implement local and JWT strategies with `@nestjs/passport`
- Hash passwords correctly, and explain why the algorithm choice matters
- Design refresh-token rotation with reuse detection
- Choose between tokens and sessions from real constraints, not fashion
- Store credentials on the client without creating an XSS-to-account-takeover path

---

## 21.1 Authentication Is Two Separate Problems

Authentication answers "who is this", authorization answers "may they do this" (→ [Ch 22](22-authorization.md)). Conflating them produces a guard that does both badly.

Authentication itself splits again:

**Establishing identity** — verifying a password, an OAuth callback, an API key. It happens once, at a login endpoint, and it is expensive by design.

**Carrying identity** — proving on each subsequent request that the earlier verification happened. This must be cheap, because it runs on every request.

The credential that carries identity is the interesting part, and it is where the security decisions live.

---

## 21.2 Passport Strategies

`@nestjs/passport` wraps Passport so a strategy becomes a provider:

```bash
npm i @nestjs/passport passport @nestjs/jwt passport-jwt passport-local argon2
npm i -D @types/passport-jwt @types/passport-local
```

A strategy's `validate` method returns the value Nest attaches to `request.user`:

```ts
// src/auth/strategies/local.strategy.ts
@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy, 'local') {
  constructor(private readonly auth: AuthService) {
    super({ usernameField: 'email' });        // default is `username`
  }

  async validate(email: string, password: string): Promise<AuthenticatedUser> {
    const user = await this.auth.verifyCredentials(email, password);
    if (!user) throw new UnauthorizedException('Invalid credentials');
    return user;                              // becomes request.user
  }
}
```

```ts
// src/auth/strategies/jwt.strategy.ts
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy, 'jwt') {
  constructor(
    @Inject(jwtConfig.KEY) config: ConfigType<typeof jwtConfig>,
    private readonly users: UsersService,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,                // never true
      secretOrKey: config.secret,
      algorithms: ['HS256'],                  // pin it — see the gotcha below
    });
  }

  async validate(payload: JwtPayload): Promise<AuthenticatedUser> {
    // The signature is already verified here. This step exists to reject
    // tokens whose subject has since been deleted, banned, or had their
    // permissions revoked — a valid signature is not a valid session.
    const user = await this.users.findActiveById(payload.sub);
    if (!user) throw new UnauthorizedException();
    return { id: user.id, email: user.email, roles: user.roles };
  }
}
```

> ⚠️ **Gotcha** — Always pin `algorithms`. Without it, a library that accepts the token's own `alg` header can be tricked into verifying an `alg: none` token, or into treating an RS256 public key as an HMAC secret. Both are classic JWT vulnerabilities, and both are prevented by naming the algorithm you expect.

The guards come from the strategy names:

```ts
@UseGuards(AuthGuard('local'))
@Post('login')
login(@CurrentUser() user: AuthenticatedUser) {
  return this.auth.issueTokens(user);
}
```

Wrapping them in named classes is worth the three lines — the string disappears from every call site and the guard becomes a place to add behaviour (→ [Ch 12](12-guards.md)):

```ts
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

---

## 21.3 Password Hashing

Passwords must be hashed with an algorithm designed to be slow. A general-purpose hash — SHA-256, MD5 — is a vulnerability regardless of salting, because it is fast enough to brute-force at billions of guesses per second.

```ts
import * as argon2 from 'argon2';

@Injectable()
export class PasswordService {
  private readonly options: argon2.Options = {
    type: argon2.argon2id,        // resists both GPU and side-channel attacks
    memoryCost: 19_456,           // 19 MiB — memory hardness is the point
    timeCost: 2,
    parallelism: 1,
  };

  hash(plain: string): Promise<string> {
    return argon2.hash(plain, this.options);
  }

  async verify(hash: string, plain: string): Promise<boolean> {
    try {
      return await argon2.verify(hash, plain);
    } catch {
      return false;               // malformed hash is a failed verification
    }
  }
}
```

**Argon2id is the current recommendation.** bcrypt remains acceptable and has a specific trap: it silently truncates input at **72 bytes**, so two different long passwords can verify against the same hash. If you use bcrypt, reject over-long passwords explicitly rather than letting them be cut.

The salt is stored inside the hash string by both algorithms — you do not manage it separately, and a schema with a separate `salt` column usually indicates a hand-rolled scheme that should be replaced.

**Login must not reveal whether an account exists.** The naive implementation leaks it two ways:

```ts
// ❌ Different messages, and a fast path when the user is missing —
//    the timing difference alone enumerates accounts.
async verifyCredentials(email: string, password: string) {
  const user = await this.users.findByEmail(email);
  if (!user) throw new UnauthorizedException('No account with that email');
  if (!(await this.passwords.verify(user.passwordHash, password))) {
    throw new UnauthorizedException('Wrong password');
  }
  return user;
}

// ✅ One message, and always do the hash work
async verifyCredentials(email: string, password: string): Promise<User | null> {
  const user = await this.users.findByEmail(email);

  // Verify against a dummy hash when the user is absent, so the response
  // time does not distinguish "no such account" from "wrong password".
  const hash = user?.passwordHash ?? this.dummyHash;
  const ok = await this.passwords.verify(hash, password);

  return ok && user ? user : null;
}
```

---

## 21.4 Refresh Token Rotation

A short-lived access token limits the damage of a leak; a refresh token lets a session outlive it. The pairing only works if the refresh token is handled carefully.

**Access token:** 5–15 minutes, stateless, carries the claims a guard needs.
**Refresh token:** days to weeks, stored server-side, single-use.

Single-use is what makes theft detectable. Each refresh issues a new pair and invalidates the old refresh token; if an old one is presented again, either the legitimate client is retrying or an attacker is using a stolen copy — and you cannot tell which, so you must assume the worst:

```ts
async refresh(presentedToken: string): Promise<TokenPair> {
  const record = await this.tokens.findByHash(sha256(presentedToken));
  if (!record) throw new UnauthorizedException('Invalid refresh token');

  if (record.usedAt) {
    // Reuse: this token was already exchanged. Either it was stolen and
    // replayed, or the real user's replacement was stolen. Revoking the
    // whole family logs everyone out, which is the safe failure.
    await this.tokens.revokeFamily(record.familyId);
    throw new UnauthorizedException('Refresh token reuse detected');
  }

  if (record.expiresAt < new Date()) throw new UnauthorizedException('Expired');

  await this.tokens.markUsed(record.id);
  return this.issueTokens(record.user, record.familyId);
}
```

> ⚠️ **Gotcha** — Store a **hash** of the refresh token, never the token itself. Refresh tokens are password-equivalent: a database leak containing raw refresh tokens is a full account compromise for every active session, and the tokens are long-lived enough for that to matter.

---

## 21.5 Where the Client Keeps the Token

This decision matters more than the token format, and it is usually made by accident.

| Storage | XSS exposure | CSRF exposure | Works cross-origin |
|---|---|---|---|
| `localStorage` | **Full** — any script reads it | none | yes |
| Memory only | lost on reload | none | yes |
| `httpOnly` cookie | **none** — script cannot read it | needs mitigation | with config |

`localStorage` is the common choice and the weakest: any XSS anywhere on the origin — including in a third-party script — reads the token and exfiltrates it. `httpOnly` cookies cannot be read by script at all, which converts an XSS from account takeover into a much smaller problem.

The usual production shape is a hybrid: access token in memory, refresh token in an `httpOnly`, `Secure`, `SameSite=Strict` cookie.

```ts
res.cookie('refresh_token', token, {
  httpOnly: true,          // invisible to JavaScript
  secure: true,            // HTTPS only
  sameSite: 'strict',      // not sent on cross-site requests — CSRF mitigation
  path: '/auth/refresh',   // sent only to the endpoint that needs it
  maxAge: 7 * 24 * 60 * 60 * 1000,
});
```

**Tokens versus sessions** is a genuine trade, not a settled question:

| | JWT | Server session |
|---|---|---|
| Revocation | hard — valid until expiry | immediate |
| Statelessness | no lookup needed | a store lookup per request |
| Size | large, on every request | a short id |
| Cross-service | verifiable anywhere | needs shared session storage |

The revocation row is the one that decides most cases. A JWT cannot be un-issued, so "log out everywhere" and "ban this user now" are impossible without a denylist — at which point you are doing a lookup per request and have most of a session system anyway. Choose JWTs when independent services must verify identity without a shared store, and sessions when immediate revocation matters and everything runs behind one boundary.

> 💼 **Interview** — "How do you revoke a JWT?" There is no clean way, and saying so is the correct answer. The follow-ups are: keep access tokens short so the window is small, hold revocation state on the refresh token, and use a denylist only for emergencies — accepting that it reintroduces the statefulness JWTs were chosen to avoid.

---

## 21.6 Common Mistakes and Best Practices

**Mistake: not pinning `algorithms`.** It enables `alg: none` and key-confusion attacks.

**Mistake: distinguishing "no such user" from "wrong password".** Both messages and timing enumerate accounts.

**Mistake: storing raw refresh tokens.** A database leak becomes a full compromise of every live session.

**Mistake: long-lived access tokens.** With no revocation path, a leaked token is valid for its whole lifetime.

**Mistake: `localStorage` for tokens.** Any XSS on the origin becomes account takeover.

**Mistake: a fast hash for passwords.** SHA-256 is brute-forceable at billions of guesses per second regardless of salt.

**Mistake: trusting a valid signature alone.** A signature proves the token was issued, not that the account still exists or is still permitted.

**Best practice: verify the subject still exists in `validate`.** It is one indexed lookup and it closes the gap between issuance and revocation.

**Best practice: rotate refresh tokens and detect reuse.** It is the only mechanism that makes theft observable.

**Best practice: rate-limit login.** Without it, the slow hash protects the database and nothing protects the endpoint.

---

## 21.7 Real-World Example: Rotating Tokens With Reuse Detection

A complete auth service: constant-time credential checking, hashed single-use refresh tokens grouped into families, and automatic revocation on reuse.

```ts
// src/auth/refresh-token.entity.ts
@Entity('refresh_tokens')
@Index(['familyId'])
export class RefreshToken {
  @PrimaryGeneratedColumn('uuid') id: string;

  // The hash, never the token. A leak of this table must not be usable.
  @Column({ unique: true }) tokenHash: string;

  // All descendants of one login share a family, so detecting reuse can
  // revoke the entire lineage rather than a single token.
  @Column({ type: 'uuid' }) familyId: string;

  @Column({ type: 'uuid' }) userId: string;
  @Column({ type: 'timestamptz' }) expiresAt: Date;
  @Column({ type: 'timestamptz', nullable: true }) usedAt: Date | null;
  @Column({ type: 'timestamptz', nullable: true }) revokedAt: Date | null;
}
```

```ts
// src/auth/auth.service.ts
import { createHash, randomBytes, randomUUID } from 'node:crypto';

@Injectable()
export class AuthService implements OnModuleInit {
  private readonly logger = new Logger(AuthService.name);
  private dummyHash!: string;

  constructor(
    private readonly users: UsersService,
    private readonly passwords: PasswordService,
    private readonly jwt: JwtService,
    private readonly tokens: RefreshTokenRepository,
    @Inject(jwtConfig.KEY) private readonly config: ConfigType<typeof jwtConfig>,
  ) {}

  async onModuleInit(): Promise<void> {
    // Computed once at boot so the "no such user" path costs the same as a
    // real verification without hashing on every failed attempt.
    this.dummyHash = await this.passwords.hash(randomBytes(32).toString('hex'));
  }

  async verifyCredentials(email: string, password: string): Promise<User | null> {
    const user = await this.users.findByEmail(email);
    const hash = user?.passwordHash ?? this.dummyHash;
    const ok = await this.passwords.verify(hash, password);
    return ok && user ? user : null;         // one outcome, one timing profile
  }

  async issueTokens(user: User, familyId = randomUUID()): Promise<TokenPair> {
    const accessToken = await this.jwt.signAsync(
      { sub: user.id, email: user.email, roles: user.roles },
      { secret: this.config.secret, expiresIn: this.config.accessTtl, algorithm: 'HS256' },
    );

    // Opaque and random: a refresh token carries no claims, so it needs no
    // signature and cannot leak anything if intercepted in a log.
    const refreshToken = randomBytes(48).toString('base64url');

    await this.tokens.create({
      tokenHash: this.hashToken(refreshToken),
      familyId,
      userId: user.id,
      expiresAt: new Date(Date.now() + this.config.refreshTtlMs),
    });

    return { accessToken, refreshToken };
  }

  async refresh(presented: string): Promise<TokenPair> {
    const record = await this.tokens.findByHash(this.hashToken(presented));

    // Unknown token: could be forged, could be from a revoked family that
    // was pruned. Either way it is not usable.
    if (!record || record.revokedAt) throw new UnauthorizedException('Invalid refresh token');

    if (record.usedAt) {
      // The critical branch. A single-use token presented twice means a copy
      // exists somewhere it should not. We cannot tell whether the presenter
      // is the victim or the attacker, so both lose the session.
      this.logger.warn(`Refresh reuse detected for family ${record.familyId}`);
      await this.tokens.revokeFamily(record.familyId);
      throw new UnauthorizedException('Refresh token reuse detected');
    }

    if (record.expiresAt < new Date()) throw new UnauthorizedException('Refresh token expired');

    const user = await this.users.findActiveById(record.userId);
    if (!user) throw new UnauthorizedException();

    await this.tokens.markUsed(record.id);
    return this.issueTokens(user, record.familyId);   // same family: lineage preserved
  }

  async logout(presented: string): Promise<void> {
    const record = await this.tokens.findByHash(this.hashToken(presented));
    if (record) await this.tokens.revokeFamily(record.familyId);
  }

  private hashToken(token: string): string {
    // SHA-256 is correct here and wrong for passwords: the input is 48 random
    // bytes, so there is nothing to brute-force and speed costs nothing.
    return createHash('sha256').update(token).digest('hex');
  }
}
```

```ts
// src/auth/auth.controller.ts
@Controller('auth')
export class AuthController {
  constructor(private readonly auth: AuthService) {}

  @Public()
  @UseGuards(ThrottlerGuard, AuthGuard('local'))    // rate limit BEFORE hashing
  @HttpCode(HttpStatus.OK)                          // a login creates nothing
  @Post('login')
  async login(@CurrentUser() user: User, @Res({ passthrough: true }) res: Response) {
    const { accessToken, refreshToken } = await this.auth.issueTokens(user);
    this.setRefreshCookie(res, refreshToken);
    return { accessToken };                         // access token stays in memory
  }

  @Public()
  @HttpCode(HttpStatus.OK)
  @Post('refresh')
  async refresh(@Cookies('refresh_token') token: string, @Res({ passthrough: true }) res: Response) {
    if (!token) throw new UnauthorizedException();
    const pair = await this.auth.refresh(token);
    this.setRefreshCookie(res, pair.refreshToken);
    return { accessToken: pair.accessToken };
  }

  private setRefreshCookie(res: Response, token: string): void {
    res.cookie('refresh_token', token, {
      httpOnly: true,
      secure: true,
      sameSite: 'strict',
      path: '/auth/refresh',       // never sent to any other endpoint
      maxAge: 7 * 24 * 60 * 60 * 1000,
    });
  }
}
```

The decision worth defending is revoking the entire token family on reuse, which logs out a legitimate user whenever a network retry or a race between two browser tabs replays a refresh token. That is a real usability cost and it is the correct trade, because the server genuinely cannot distinguish the two cases: if a single-use token is presented twice, a copy exists somewhere, and the possibilities are "harmless retry" and "attacker holds a stolen token". Choosing the lenient interpretation means an attacker who steals a refresh token keeps a renewable session indefinitely and invisibly, which is precisely the outcome rotation exists to prevent. The cost is bounded — the user logs in again — and the alternative is unbounded. The mitigation is to reduce spurious reuse rather than to soften the response: a short grace window where the immediately-previous token returns the *same* new pair rather than issuing another handles genuine retries without weakening detection.

---

## 21.8 Questions

**1.** MCQ — A Passport strategy's `validate` return value becomes: a) the response b) `request.user` c) the token d) discarded
<details><summary>Answer</summary>**b)** Nest attaches it to the request for guards and decorators to read.</details>

**2.** MCQ — Not pinning `algorithms` in a JWT strategy enables: a) replay b) `alg: none` and key confusion c) CSRF d) timing attacks
<details><summary>Answer</summary>**b)** Both classic JWT vulnerabilities.</details>

**3.** MCQ — bcrypt silently truncates input at: a) 32 bytes b) 55 bytes c) 72 bytes d) 128 bytes
<details><summary>Answer</summary>**c)** So two different long passwords can verify against one hash.</details>

**4.** MCQ — Refresh tokens should be stored: a) in plaintext b) hashed c) encrypted with a reversible key d) not at all
<details><summary>Answer</summary>**b)** They are password-equivalent; a leak of the table must not be usable.</details>

**5.** MCQ — Which storage makes XSS unable to read a token? a) `localStorage` b) `sessionStorage` c) `httpOnly` cookie d) IndexedDB
<details><summary>Answer</summary>**c)** Script cannot access it at all.</details>

**6.** MCQ — Revoking a JWT before expiry requires: a) nothing b) a denylist c) a shorter secret d) rotation alone
<details><summary>Answer</summary>**b)** Which reintroduces the per-request state JWTs were chosen to avoid.</details>

**7.** Short — Why check the user still exists inside `validate`?
<details><summary>Answer</summary>A valid signature proves the token was issued, not that the account still exists or is still permitted — deletion, bans, and revoked permissions all happen after issuance.</details>

**8.** Short — Why verify against a dummy hash when the user is not found?
<details><summary>Answer</summary>Skipping the hash makes the missing-user path measurably faster, and that timing difference alone lets an attacker enumerate which emails have accounts.</details>

**9.** Short — What does refresh-token rotation detect that a static refresh token cannot?
<details><summary>Answer</summary>Theft. A single-use token presented twice proves a copy exists, which is the only signal you get that a token was stolen.</details>

**10.** Short — Why is SHA-256 correct for hashing a refresh token but wrong for a password?
<details><summary>Answer</summary>A refresh token is 48 random bytes with no structure to brute-force, so speed costs nothing; a password is low-entropy and guessable, so the hash must be deliberately slow.</details>

**11.** Short — When are sessions a better choice than JWTs?
<details><summary>Answer</summary>When immediate revocation matters and all consumers sit behind one boundary — a session id can be invalidated instantly, while a JWT stays valid until it expires.</details>

**12.** Output — A user is deleted, but their unexpired JWT is presented. What happens if `validate` only decodes the payload?
<details><summary>Answer</summary>The request succeeds. The signature is still valid, so without a lookup the deleted user remains authenticated until the token expires.</details>

**13.** Output — Login returns "No account with that email" for unknown users and "Wrong password" otherwise. What has been leaked?
<details><summary>Answer</summary>Which email addresses have accounts — an enumeration oracle usable for targeted phishing and credential stuffing.</details>

**14.** Output — A refresh token is used twice. With family revocation, what is the state of the user's sessions?
<details><summary>Answer</summary>All revoked. Every token descended from that login is invalidated and the user must authenticate again.</details>

**15.** Output — A 90-character password is hashed with bcrypt, then the user logs in with only its first 72 characters. Does it succeed?
<details><summary>Answer</summary>Yes — bcrypt truncated the input at 72 bytes, so the remainder never contributed to the hash.</details>

**16.** Debug — Tokens are accepted after logout. Cause?
<details><summary>Answer</summary>Logout revoked the refresh token but the access token is stateless and remains valid until expiry. Shorten the access TTL, and use a denylist if immediate revocation is required.</details>

**17.** Debug — An XSS in a third-party analytics script led to account takeovers. What was the storage choice?
<details><summary>Answer</summary>`localStorage`. Any script on the origin can read it, so an XSS anywhere exfiltrates the token. An `httpOnly` cookie would not have been readable.</details>

**18.** Debug — Credential-stuffing succeeds despite Argon2 hashing. What is missing?
<details><summary>Answer</summary>Rate limiting on the login endpoint. A slow hash protects the stored hashes if the database leaks; it does nothing to stop online guessing.</details>

**19.** Debug — Users are randomly logged out, and logs show "refresh token reuse detected". Likely cause?
<details><summary>Answer</summary>Two tabs or a client retry refreshing concurrently, so the same token is presented twice legitimately. Add a short grace window where the immediately-previous token returns the same new pair.</details>

**20.** Debug — The refresh cookie is not sent by the browser on `/auth/refresh`. What do you check?
<details><summary>Answer</summary>The `path` (it must cover the endpoint), `sameSite` against the request's origin, and `secure` against whether the page is HTTPS.</details>

**21.** Code — Write a constant-outcome credential check.
<details><summary>Answer</summary>

```ts
async verifyCredentials(email: string, password: string): Promise<User | null> {
  const user = await this.users.findByEmail(email);
  const hash = user?.passwordHash ?? this.dummyHash;   // always hash
  const ok = await this.passwords.verify(hash, password);
  return ok && user ? user : null;                     // one failure mode
}
```
</details>

**22.** Code — Implement reuse detection on refresh.
<details><summary>Answer</summary>

```ts
const record = await this.tokens.findByHash(sha256(presented));
if (!record || record.revokedAt) throw new UnauthorizedException();
if (record.usedAt) {
  await this.tokens.revokeFamily(record.familyId);     // assume compromise
  throw new UnauthorizedException('Refresh token reuse detected');
}
if (record.expiresAt < new Date()) throw new UnauthorizedException();
await this.tokens.markUsed(record.id);
return this.issueTokens(user, record.familyId);
```
</details>

**23.** Code — Set a refresh cookie with correct security attributes.
<details><summary>Answer</summary>

```ts
res.cookie('refresh_token', token, {
  httpOnly: true,            // unreadable by script
  secure: true,              // HTTPS only
  sameSite: 'strict',        // not sent cross-site
  path: '/auth/refresh',     // scoped to the one endpoint that needs it
  maxAge: 7 * 24 * 60 * 60 * 1000,
});
```
</details>

**24.** Scenario — A teammate wants a 24-hour access token to avoid refresh complexity. Evaluate it.
<details><summary>Answer</summary>It trades a bounded engineering cost for an unbounded security one. A stateless token cannot be revoked, so a token leaked through a log, a browser extension, or a proxy grants full access for a day, and neither logout nor a password change stops it. It also removes the only theft signal you would otherwise have, since rotation is what makes reuse detectable. The complexity being avoided is real but small — a token table, a rotation endpoint, and reuse detection — and it is the mechanism that keeps the exposure window at minutes. If the driver is genuinely a client that cannot refresh, the right answer is a longer-lived opaque token checked against a store, which is revocable, rather than a long-lived JWT, which is not.</details>

**25.** Scenario — You must support both a first-party web app and third-party API integrations. How do you structure authentication?
<details><summary>Answer</summary>Use different credential types for different threat models rather than stretching one to cover both. The web app gets short access tokens plus rotating refresh tokens in `httpOnly` cookies, because it runs in a browser where XSS and CSRF are the dominant risks. Third-party integrations get long-lived API keys or OAuth client credentials, since there is no browser, no user session, and no place to run a refresh loop — but those must carry their own scopes rather than reusing user roles, because a key is not a person and will otherwise inherit permissions nobody intended to grant. Normalise both onto one principal object with a `type` field at the guard boundary so downstream code stays uniform, and default to refusing API-key principals on endpoints that have not explicitly opted in, so adding a route fails closed.</details>

---

## Chapter Summary

- Authentication splits into establishing identity once and carrying it cheaply on every request; the credential that carries it is where the security decisions live.
- Pin the JWT `algorithms` list, and treat a valid signature as proof of issuance only — `validate` should confirm the subject still exists and is permitted.
- Passwords need a deliberately slow, memory-hard hash (Argon2id); bcrypt's silent 72-byte truncation must be handled explicitly if you use it.
- Login must be indistinguishable between "no such account" and "wrong password" in both message and timing, which means hashing even when the user is absent.
- Refresh tokens must be stored hashed and be single-use, because rotation is the only mechanism that makes theft observable — and reuse must revoke the whole family.
- Token storage decides the blast radius of an XSS: `localStorage` turns any script injection into account takeover, while an `httpOnly` cookie cannot be read at all.

---

## Practice Exercises

1. Measure login response times for an existing and a non-existent email, then add the dummy-hash path and measure again.
2. Hash a 100-character password with bcrypt and verify with its first 72 characters. Then repeat with Argon2id.
3. Implement rotation with reuse detection and prove that replaying a used token revokes the family.
4. Store a token in `localStorage`, then in an `httpOnly` cookie, and attempt to read each from the console.
5. Remove `algorithms` from a JWT strategy and craft a token that exploits the omission.

---

## Mini Project: A Complete Auth Module

Build authentication you would be willing to ship.

**Requirements:**

- Registration and login with Argon2id, rate limiting, and no account-enumeration signal in message or timing.
- Access tokens under 15 minutes; refresh tokens opaque, hashed at rest, single-use, family-grouped.
- Reuse detection revoking the family, with a short grace window so honest retries do not trigger it.
- Refresh token in an `httpOnly` `SameSite=Strict` cookie scoped to the refresh path; access token never persisted.
- Tests covering: reuse revokes the family, a deleted user's valid token is rejected, and timing does not distinguish unknown emails.

---

**Next:** [Chapter 22 — Authorization →](22-authorization.md)
