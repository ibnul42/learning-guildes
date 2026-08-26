# Chapter 16 — Configuration and Environments

## Learning Objectives

- Load and validate configuration so a misconfigured deploy fails at boot
- Namespace configuration with `registerAs` and consume it type-safely
- Choose between Joi and `class-validator` for schema validation
- Explain why `ConfigService.get()` returning `undefined` is a design flaw you can remove
- Handle secrets without putting them in environment variables you also log

---

## 16.1 Configuration Should Fail at Boot

The default failure mode for configuration is the worst possible one: a missing variable produces `undefined`, which flows silently through the application until something dereferences it, usually in production, usually under load.

```ts
// ❌ Silent. A missing DB_HOST becomes `undefined` and surfaces as a
//    connection error to host "undefined" minutes later.
const host = process.env.DB_HOST;
```

The fix is to validate the whole environment once, at startup, and refuse to boot if anything required is missing or malformed. A deploy that cannot start is a visible, immediate, recoverable failure. A deploy that starts and then misbehaves is an incident.

```bash
npm i @nestjs/config joi
```

```ts
// src/config/config.module.ts
import { ConfigModule } from '@nestjs/config';
import * as Joi from 'joi';

ConfigModule.forRoot({
  isGlobal: true,                       // config is the legitimate global (→ Ch 4)
  cache: true,                          // process.env lookups are surprisingly slow
  envFilePath: ['.env.local', '.env'],  // first match wins
  expandVariables: true,                // ${VAR} interpolation inside .env
  validationSchema: Joi.object({
    NODE_ENV: Joi.string().valid('development', 'test', 'production').default('development'),
    PORT: Joi.number().port().default(3000),
    DATABASE_URL: Joi.string().uri().required(),
    JWT_SECRET: Joi.string().min(32).required(),
    REDIS_URL: Joi.string().uri().optional(),
  }),
  validationOptions: {
    abortEarly: false,                  // report EVERY problem, not just the first
    allowUnknown: true,                 // the OS sets hundreds of variables
  },
});
```

`abortEarly: false` matters more than it looks. With the default, fixing a broken deploy becomes a sequence of restart-and-discover-the-next-missing-variable cycles; with it, you get the full list at once.

`allowUnknown: true` is required in practice — the schema describes *your* variables, and the environment always contains `PATH`, `HOME`, and hundreds more.

> ⚠️ **Gotcha** — `ConfigModule.forRoot()` reads `.env` when the module initialises. Anything evaluated earlier — a `useValue` provider built at import time, a module-scope `new Client(process.env.KEY)` — sees `undefined` (→ [Ch 8](08-custom-providers.md)). Configuration-derived values must come from a factory.

---

## 16.2 `get` vs `getOrThrow`

`ConfigService.get()` returns `T | undefined`, which pushes a null check onto every call site — and those checks get skipped.

```ts
// ❌ Type is string | undefined; the non-null assertion is a lie that compiles
const secret = this.config.get<string>('JWT_SECRET')!;

// ✅ Throws at the point of use with a clear message, and the type is string
const secret = this.config.getOrThrow<string>('JWT_SECRET');
```

Use `getOrThrow` for anything required and `get` with an explicit default for anything optional:

```ts
const poolSize = this.config.get<number>('DB_POOL_SIZE', 10);   // default as 2nd arg
```

If the schema already marks a variable `required()`, `getOrThrow` should never fire — which is exactly why it is the right call. It converts "this cannot happen" from a comment into an assertion, and it removes `undefined` from the type.

---

## 16.3 Namespaced Configuration

Flat keys do not scale. `registerAs` groups related settings and gives them a type:

```ts
// src/config/database.config.ts
import { registerAs } from '@nestjs/config';

export default registerAs('database', () => ({
  url: process.env.DATABASE_URL!,
  poolSize: Number(process.env.DB_POOL_SIZE ?? 10),
  ssl: process.env.DB_SSL === 'true',
  logging: process.env.NODE_ENV === 'development',
}));
```

```ts
// src/config/jwt.config.ts
export default registerAs('jwt', () => ({
  secret: process.env.JWT_SECRET!,
  accessTtl: process.env.JWT_ACCESS_TTL ?? '15m',
  refreshTtl: process.env.JWT_REFRESH_TTL ?? '7d',
}));
```

Register them with `load`, then inject the namespace directly:

```ts
ConfigModule.forRoot({ isGlobal: true, load: [databaseConfig, jwtConfig], validationSchema });
```

```ts
import { ConfigType } from '@nestjs/config';
import jwtConfig from '../config/jwt.config';

@Injectable()
export class AuthService {
  constructor(
    @Inject(jwtConfig.KEY)
    private readonly config: ConfigType<typeof jwtConfig>,   // fully typed
  ) {}

  sign(payload: JwtPayload): string {
    return this.jwt.sign(payload, {
      secret: this.config.secret,        // string, not string | undefined
      expiresIn: this.config.accessTtl,
    });
  }
}
```

This is the shape worth adopting. The service depends on a small typed object rather than the whole `ConfigService`, so its configuration surface is visible in the constructor, and a test supplies a plain object with no mocking.

---

## 16.4 Validating With `class-validator` Instead

Joi is the documented default. If you already use `class-validator` for DTOs (→ [Ch 14](14-pipes-validation.md)), a `validate` function keeps one validation library in the project:

```ts
// src/config/env.validation.ts
import { plainToInstance } from 'class-transformer';
import { IsEnum, IsInt, IsString, IsUrl, Max, Min, MinLength, validateSync } from 'class-validator';

enum Environment { Development = 'development', Test = 'test', Production = 'production' }

class EnvironmentVariables {
  @IsEnum(Environment)
  NODE_ENV: Environment;

  @IsInt() @Min(1) @Max(65535)
  PORT: number;

  @IsUrl({ protocols: ['postgres', 'postgresql'], require_tld: false })
  DATABASE_URL: string;

  @IsString() @MinLength(32)
  JWT_SECRET: string;
}

export function validate(config: Record<string, unknown>): EnvironmentVariables {
  const validated = plainToInstance(EnvironmentVariables, config, {
    enableImplicitConversion: true,      // env values are always strings
  });
  const errors = validateSync(validated, { skipMissingProperties: false });

  if (errors.length) {
    // Print every problem at once, and never print the values themselves.
    throw new Error(
      `Invalid environment:\n${errors.map((e) => `  - ${e.property}: ${Object.values(e.constraints ?? {}).join(', ')}`).join('\n')}`,
    );
  }
  return validated;
}
```

```ts
ConfigModule.forRoot({ isGlobal: true, validate });
```

| | Joi | `class-validator` |
|---|---|---|
| Extra dependency | yes | already present for DTOs |
| Defaults | built in | via class field initialisers |
| Type inference | weak | the class *is* the type |
| Coercion | automatic | needs `enableImplicitConversion` |

Either is fine. Choosing one and using it everywhere matters more than which.

---

## 16.5 Secrets

Environment variables are convenient and are not a secret store. They appear in `docker inspect`, in crash dumps, in process listings, and in any log line that dumps the environment.

Three rules keep the damage bounded:

**Never log the config object.** A debug line printing `process.env` or a validated config object puts every secret into your log aggregator, where it is retained and widely readable.

```ts
// ❌ ships JWT_SECRET and DATABASE_URL to the log pipeline
this.logger.debug(`Config: ${JSON.stringify(config)}`);

// ✅ redact at the boundary
const REDACT = /(SECRET|PASSWORD|TOKEN|KEY|DSN|URL)/i;
const safe = Object.fromEntries(
  Object.entries(config).map(([k, v]) => [k, REDACT.test(k) ? '***' : v]),
);
```

**Fail closed in production.** A missing secret must throw, never fall back to a development default. A hard-coded fallback secret is a permanent authentication bypass, and it will be committed at some point.

```ts
// ❌ this exact line has shipped to production in real codebases
secret: process.env.JWT_SECRET ?? 'dev-secret',
```

**Load real secrets from a real store.** For anything beyond a hobby deployment, an async factory pulling from AWS Secrets Manager, Vault, or the platform's secret mount is correct — and it fails the boot when unavailable (→ [Ch 8](08-custom-providers.md)).

> 💼 **Interview** — "How do you manage secrets in a NestJS app?" Mentioning `.env` and `ConfigModule` is the baseline answer. Adding that `.env` is for local development only, that production secrets come from a secret manager via an async provider, and that the config object must be redacted before logging demonstrates you have run this in production.

---

## 16.6 Common Mistakes and Best Practices

**Mistake: reading `process.env` directly in services.** It bypasses validation and defaults, and makes the value untestable without mutating the global environment.

**Mistake: `get()` with a non-null assertion.** The assertion is a lie the compiler accepts; `getOrThrow` makes it true.

**Mistake: no validation schema.** A typo in a variable name becomes `undefined` at runtime instead of a startup error.

**Mistake: `abortEarly` left at its default.** You discover missing variables one restart at a time.

**Mistake: fallback secrets.** `?? 'dev-secret'` is an authentication bypass that will reach production.

**Mistake: committing `.env`.** Add it to `.gitignore` on day one and commit `.env.example` instead.

**Mistake: logging the config object.** Every secret lands in the log aggregator, retained and broadly readable.

**Best practice: namespace with `registerAs` and inject `ConfigType`.** Services declare a small typed surface rather than the whole config service.

**Best practice: make `ConfigModule` global.** It is genuine cross-cutting infrastructure and one of the few justified `@Global()` modules.

**Best practice: commit `.env.example` with every key and no values.** It is the only documentation of what the application needs that cannot go stale silently.

---

## 16.7 Real-World Example: Typed, Validated, Secret-Aware Configuration

A configuration layer that validates everything at boot, exposes typed namespaces, loads production secrets from a manager, and cannot leak them into logs.

```ts
// src/config/configuration.ts
import { registerAs } from '@nestjs/config';

export const appConfig = registerAs('app', () => ({
  env: process.env.NODE_ENV as 'development' | 'test' | 'production',
  port: Number(process.env.PORT ?? 3000),
  corsOrigins: process.env.CORS_ORIGINS?.split(',').filter(Boolean) ?? [],
}));

export const databaseConfig = registerAs('database', () => ({
  url: process.env.DATABASE_URL!,
  poolSize: Number(process.env.DB_POOL_SIZE ?? 10),
  ssl: process.env.DB_SSL === 'true',
}));

export const jwtConfig = registerAs('jwt', () => ({
  secret: process.env.JWT_SECRET!,
  accessTtl: process.env.JWT_ACCESS_TTL ?? '15m',
  refreshTtl: process.env.JWT_REFRESH_TTL ?? '7d',
}));
```

```ts
// src/config/env.schema.ts
import * as Joi from 'joi';

export const envSchema = Joi.object({
  NODE_ENV: Joi.string().valid('development', 'test', 'production').default('development'),
  PORT: Joi.number().port().default(3000),
  DATABASE_URL: Joi.string().uri().required(),
  DB_POOL_SIZE: Joi.number().integer().min(1).max(100).default(10),
  DB_SSL: Joi.boolean().default(false),

  // 32 characters minimum: a short HMAC secret is brute-forceable, and this is
  // the cheapest place to make that impossible to get wrong.
  JWT_SECRET: Joi.string().min(32).required(),
  JWT_ACCESS_TTL: Joi.string().default('15m'),
  JWT_REFRESH_TTL: Joi.string().default('7d'),

  CORS_ORIGINS: Joi.string().allow('').default(''),

  // Required only in production — local development runs without Redis.
  REDIS_URL: Joi.string().uri().when('NODE_ENV', {
    is: 'production',
    then: Joi.required(),
    otherwise: Joi.optional(),
  }),
});
```

```ts
// src/config/secrets.provider.ts
import { Provider, Logger } from '@nestjs/common';
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

export const RESOLVED_SECRETS = Symbol('RESOLVED_SECRETS');

/**
 * In production, secrets come from the secret manager and env vars hold only
 * non-sensitive settings. Locally, .env is authoritative. The branch is on an
 * explicit allowlist so an unset NODE_ENV fails closed into the strict path.
 */
export const secretsProvider: Provider = {
  provide: RESOLVED_SECRETS,
  useFactory: async (): Promise<Record<string, string>> => {
    const logger = new Logger('Secrets');
    const local = ['development', 'test'];

    if (local.includes(process.env.NODE_ENV ?? '')) {
      logger.warn('Using .env for secrets — local development only');
      return {};
    }

    const secretId = process.env.SECRETS_ID;
    if (!secretId) throw new Error('SECRETS_ID is required outside development');

    const client = new SecretsManagerClient({});
    const timeout = new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error('Secrets fetch timed out after 10s')), 10_000),
    );

    const result = await Promise.race([
      client.send(new GetSecretValueCommand({ SecretId: secretId })),
      timeout,
    ]);

    if (!result.SecretString) throw new Error(`Secret ${secretId} is empty`);
    const parsed = JSON.parse(result.SecretString) as Record<string, string>;

    // Merge into process.env so ConfigModule's schema validates them too,
    // rather than having two parallel sources of configuration.
    Object.assign(process.env, parsed);
    logger.log(`Loaded ${Object.keys(parsed).length} secrets`);
    return parsed;
  },
};
```

```ts
// src/config/config.module.ts
@Module({
  imports: [
    NestConfigModule.forRoot({
      isGlobal: true,
      cache: true,
      envFilePath: ['.env.local', '.env'],
      expandVariables: true,
      load: [appConfig, databaseConfig, jwtConfig],
      validationSchema: envSchema,
      validationOptions: { abortEarly: false, allowUnknown: true },
    }),
  ],
  providers: [secretsProvider, SafeConfigLogger],
  exports: [secretsProvider],
})
export class AppConfigModule {}
```

```ts
// src/config/safe-config-logger.ts
@Injectable()
export class SafeConfigLogger implements OnApplicationBootstrap {
  private readonly logger = new Logger('Config');
  private static readonly SENSITIVE = /(SECRET|PASSWORD|TOKEN|KEY|CREDENTIAL|DSN|URL)/i;

  constructor(private readonly config: ConfigService) {}

  onApplicationBootstrap(): void {
    // Logging the effective config is genuinely useful for diagnosing a bad
    // deploy — but only if it can never print a secret. Redaction is by key
    // pattern and deliberately over-broad: URLs often embed credentials.
    const snapshot = Object.entries(process.env)
      .filter(([key]) => envSchema.describe().keys[key])
      .map(([key, value]) => [
        key,
        SafeConfigLogger.SENSITIVE.test(key) ? '***redacted***' : value,
      ]);

    this.logger.log(`Effective configuration: ${JSON.stringify(Object.fromEntries(snapshot))}`);
  }
}
```

The decision worth defending is redacting by key *pattern* rather than by an explicit list of secret keys. A curated list is more precise and it is wrong the first time someone adds a variable and does not think about the logger — which is exactly the moment redaction matters. The pattern over-redacts on purpose: `DATABASE_URL` is matched by `URL` and hidden even though only part of it is sensitive, because a connection string with an embedded password is the single most common way credentials reach a log aggregator. Losing the ability to see a hostname in the startup log is a small cost; the alternative fails open, silently, and only becomes visible during an audit. Redaction should be the default for anything ambiguous, and the list of things you deliberately choose to print should be the short one.

---

## 16.8 Questions

**1.** MCQ — `ConfigService.get<string>('KEY')` returns: a) `string` b) `string | undefined` c) `string | null` d) throws
<details><summary>Answer</summary>**b)** Which is why `getOrThrow` is preferable for required values.</details>

**2.** MCQ — `abortEarly: false` causes validation to: a) stop at the first error b) report every error c) ignore errors d) retry
<details><summary>Answer</summary>**b)** Essential when fixing a misconfigured deploy.</details>

**3.** MCQ — `registerAs('db', …)` provides a token accessible via: a) `'db'` b) `dbConfig.KEY` c) `ConfigService` only d) `@Inject('db')`
<details><summary>Answer</summary>**b)** `dbConfig.KEY`, typed with `ConfigType<typeof dbConfig>`.</details>

**4.** MCQ — `allowUnknown: true` is needed because: a) Nest adds variables b) the OS sets many variables c) Joi requires it d) it improves speed
<details><summary>Answer</summary>**b)** The environment always contains `PATH`, `HOME`, and hundreds more.</details>

**5.** MCQ — Where should `.env` files be used? a) all environments b) local development only c) production only d) tests only
<details><summary>Answer</summary>**b)** Production secrets belong in a secret manager or platform secret mount.</details>

**6.** MCQ — `cache: true` on `ConfigModule` avoids: a) disk reads b) repeated `process.env` lookups c) validation d) parsing
<details><summary>Answer</summary>**b)** `process.env` access is surprisingly slow at high request rates.</details>

**7.** Short — Why should configuration validation happen at startup?
<details><summary>Answer</summary>A deploy that refuses to start is visible and recoverable; one that starts with `undefined` config misbehaves later, usually in production and usually under load.</details>

**8.** Short — Why prefer injecting `ConfigType<typeof someConfig>` over `ConfigService`?
<details><summary>Answer</summary>The service's configuration surface becomes visible in its constructor and fully typed, and tests supply a plain object instead of mocking a service.</details>

**9.** Short — Why is `?? 'dev-secret'` dangerous?
<details><summary>Answer</summary>It is an authentication bypass with a known value that will eventually run in production, because the fallback hides the missing variable rather than reporting it.</details>

**10.** Short — Why redact by key pattern rather than an explicit list?
<details><summary>Answer</summary>A curated list is wrong the moment someone adds a variable without updating it, so it fails open; a pattern over-redacts, which is the safe direction.</details>

**11.** Short — Why must configuration-derived providers use `useFactory`?
<details><summary>Answer</summary>`useValue` is evaluated at import time, before `ConfigModule` has loaded `.env`, so the value is `undefined`.</details>

**12.** Output — `DATABASE_URL` is missing and the schema marks it `required()`. What happens on `npm start`?
<details><summary>Answer</summary>The application throws during bootstrap with a Joi validation error naming `DATABASE_URL`, and never listens.</details>

**13.** Output — With `abortEarly` at its default and three variables missing, how many are reported?
<details><summary>Answer</summary>One — the first failure aborts validation, so fixing the config takes three restarts.</details>

**14.** Output — A provider uses `useValue: new Client(process.env.API_KEY)` and `.env` defines `API_KEY`. What does the client receive?
<details><summary>Answer</summary>`undefined`. Module-scope evaluation happens before `ConfigModule` loads the file.</details>

**15.** Output — `config.get<number>('PORT')` where `.env` has `PORT=3000` and no validation schema is present. What is the type of the value at runtime?
<details><summary>Answer</summary>A string, `'3000'`. The generic is a compile-time assertion only; without a schema performing coercion, environment values stay strings.</details>

**16.** Debug — An app works locally and fails in production with "connect ECONNREFUSED undefined:5432". Diagnosis?
<details><summary>Answer</summary>`DATABASE_URL` (or the host variable) is unset in production and there is no validation schema, so `undefined` reached the driver. Add the variable to a `required()` schema so this fails at boot instead.</details>

**17.** Debug — A secret appeared in the log aggregator. Where do you look?
<details><summary>Answer</summary>Any line serialising the config object or `process.env`, and any error handler printing a connection string. Add pattern-based redaction and a test asserting a known secret never appears in log output.</details>

**18.** Debug — `getOrThrow('REDIS_URL')` throws in development but the app should run without Redis. Fix?
<details><summary>Answer</summary>It is optional locally, so use `get()` with a default and make the schema require it only when `NODE_ENV` is production.</details>

**19.** Debug — Tests fail because config values are missing, though the app runs. Cause?
<details><summary>Answer</summary>The testing module does not import `ConfigModule`, so the global registration from `AppModule` never happens — the cost of `@Global()` (→ Ch 4). Import it explicitly in the test, or provide the namespace as a plain object.</details>

**20.** Debug — After adding `expandVariables: true`, a password containing `$` stopped working. Why?
<details><summary>Answer</summary>Variable expansion interprets `${...}` and `$NAME` sequences, so part of the password is substituted. Escape it or disable expansion for that file.</details>

**21.** Code — Write a namespaced config and inject it type-safely.
<details><summary>Answer</summary>

```ts
export default registerAs('mail', () => ({
  host: process.env.MAIL_HOST!,
  port: Number(process.env.MAIL_PORT ?? 587),
  from: process.env.MAIL_FROM!,
}));

@Injectable()
export class MailService {
  constructor(
    @Inject(mailConfig.KEY)
    private readonly config: ConfigType<typeof mailConfig>,
  ) {}
}
```
</details>

**22.** Code — Write a Joi schema requiring a variable only in production.
<details><summary>Answer</summary>

```ts
Joi.object({
  NODE_ENV: Joi.string().valid('development', 'test', 'production').required(),
  SENTRY_DSN: Joi.string().uri().when('NODE_ENV', {
    is: 'production',
    then: Joi.required(),
    otherwise: Joi.optional(),
  }),
});
```
</details>

**23.** Code — Write a redaction helper that hides sensitive values before logging.
<details><summary>Answer</summary>

```ts
const SENSITIVE = /(SECRET|PASSWORD|TOKEN|KEY|CREDENTIAL|DSN|URL)/i;

export function redact(config: Record<string, unknown>): Record<string, unknown> {
  return Object.fromEntries(
    Object.entries(config).map(([k, v]) => [k, SENSITIVE.test(k) ? '***' : v]),
  );
}
```
Over-redacting is deliberate: URLs frequently embed credentials.
</details>

**24.** Scenario — A teammate proposes reading feature flags from environment variables so they can be changed without a code deploy. Evaluate it.
<details><summary>Answer</summary>It solves half the problem and creates a new one. Environment variables are read at boot, so changing one still requires a restart — which on most platforms is a deploy in everything but name, and a rolling restart means both values are live simultaneously. That is acceptable for infrastructure switches and dangerous for anything a user can observe mid-session. If flags genuinely need to change at runtime, they need a flag service with a cache and a subscription, which is a different component with different testing needs. Keep environment variables for configuration that is fixed for a process's lifetime — endpoints, pool sizes, credentials — and treat "can change while running" as the property that determines whether something is configuration at all.</details>

**25.** Scenario — Your application runs in three environments with substantially different configuration, and `.env.example` has drifted from reality. How do you fix it durably?
<details><summary>Answer</summary>Make the schema the single source of truth and generate everything else from it. `.env.example` drifts because it is a second list maintained by hand, so delete it as a hand-written file and generate it from the Joi schema in a build step, including which variables are required per environment. Then add a CI check that fails if the generated file differs from the committed one, which turns drift into a build error rather than a discovery during an incident. The deeper fix is that the schema must express environment differences itself — `when('NODE_ENV', …)` rather than three separate documents — because any design with three parallel lists will have three states of staleness. Documentation that cannot be verified will always be wrong eventually; documentation generated from the thing it describes cannot be.</details>

---

## Chapter Summary

- Validate the entire environment at boot with `abortEarly: false`, so a misconfigured deploy fails visibly and reports every problem at once.
- `getOrThrow` removes `undefined` from the type and converts an impossible case into an assertion; reserve `get` for optional values with explicit defaults.
- `registerAs` plus `ConfigType` lets services declare a small typed configuration surface instead of injecting the whole `ConfigService`.
- Anything derived from configuration must come from `useFactory` — `useValue` evaluates before `.env` is loaded.
- Environment variables are not a secret store: never log the config object, never provide fallback secrets, and load production secrets from a real manager.
- Redact by key pattern rather than a curated list, because a list fails open the first time someone forgets to update it.

---

## Practice Exercises

1. Remove a required variable and compare the failure with and without a validation schema.
2. Convert a service from `ConfigService.get()` to an injected `ConfigType` namespace and compare the tests.
3. Build a provider with `useValue` reading `process.env`, prove it is `undefined`, then fix it with `useFactory`.
4. Write a test asserting that a known secret never appears anywhere in captured log output.
5. Generate `.env.example` from a Joi schema and add a CI check that fails on drift.

---

## Mini Project: A Configuration Subsystem

Build configuration for a service deployed to three environments.

**Requirements:**

- One schema validating every variable, with per-environment requirements expressed in it.
- Namespaced, typed configuration consumed via `ConfigType` — no `ConfigService` in domain services.
- Production secrets loaded from a secret manager via a bounded async provider that fails the boot on error.
- A startup log of the effective configuration with pattern-based redaction, plus a test proving no secret leaks.
- `.env.example` generated from the schema, with CI failing if it drifts.

---

**Next:** [Chapter 17 — Databases with TypeORM →](17-typeorm.md)
