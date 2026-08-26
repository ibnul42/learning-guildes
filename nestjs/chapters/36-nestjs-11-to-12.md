# Chapter 36 — NestJS 11 → 12 and the Ecosystem

## Learning Objectives

- Recall what changed in NestJS 11 and what it broke
- Describe the four pillars of NestJS 12 and assess their impact on your codebase
- Plan an ESM migration, which is the hardest part of the upgrade
- Decide whether to adopt Standard Schema validation alongside or instead of `class-validator`
- Choose ecosystem packages that will survive the transition

---

## 36.1 Where NestJS 11 Left Things

This book targets NestJS 11 (11.2.x) on Node 20 or 22. Its breaking changes are worth restating, because most upgrade failures from 10 are one of these four:

| Change | Symptom | Chapter |
|---|---|---|
| Node 20 minimum | obscure native-binding errors on 18 | [1](01-what-nestjs-is.md) |
| Express 5 by default | `path-to-regexp` throws at startup | [5](05-controllers-routing.md) |
| Termination hooks reversed | a dependency is already destroyed during teardown | [9](09-scopes-lifecycle.md) |
| `cache-manager` v6 (Keyv) | `ttl: 60` now means 60 **milliseconds** | [20](20-caching.md) |

The Express 5 route syntax is the one that stops a deploy outright:

```ts
@Get('*')             // ❌ invalid in Express 5
@Get('*splat')        // ✅ named wildcard
@Get(':file{.:ext}')  // ✅ optional segment, braces not `?`
```

---

## 36.2 The Four Pillars of v12

NestJS 12 is in development for approximately Q3 2026, with prereleases published under the `next` npm tag. It is the largest change to the framework's infrastructure since its release, and it rests on four pillars.

**1. Full ESM.** Every official package migrates from CommonJS to ES modules. This is the pillar that will consume your migration time — §36.3.

**2. Standard Schema in route decorators.** `@Body()`, `@Query()`, and `@Param()` gain native support for [Standard Schema](https://standardschema.dev), the shared interface implemented by Zod, Valibot, and ArkType. Validation becomes possible without `class-validator` and without decorated DTO classes:

```ts
// Anticipated shape — verify against the release
const CreateTaskSchema = z.object({
  title: z.string().min(1).max(120),
  priority: z.number().int().min(0).max(3).default(0),
});

@Post()
create(@Body(CreateTaskSchema) dto: z.infer<typeof CreateTaskSchema>) {}
```

**3. Vitest replaces Jest** — for new **ESM** projects. CommonJS projects keep Jest. The `@nestjs/testing` API is unchanged; only the runner and its mocking helpers differ (→ [Ch 29](29-unit-testing.md)).

**4. Rspack replaces Webpack, oxlint replaces ESLint.** Both are drop-in-ish replacements aimed at build and lint speed, and both matter mainly to your tooling configuration rather than your application code.

> ⚠️ **Do not build on prereleases.** `next`-tagged packages are for testing the migration, not for production. Everything in Chapters 1–35 targets v11 deliberately, and the sensible plan is to prepare now and adopt after the stable release has had a few patch versions.

---

## 36.3 The ESM Migration

ESM is not a configuration flag. It changes module resolution, and the changes surface in five places.

**`require` is gone.** So are `__dirname`, `__filename`, and `require.resolve`:

```ts
// ❌ Not defined in ESM
const config = require('./config.json');
const here = __dirname;

// ✅
import config from './config.json' with { type: 'json' };
const here = dirname(fileURLToPath(import.meta.url));
```

**Relative imports need file extensions.**

```ts
import { TasksService } from './tasks.service';       // ❌
import { TasksService } from './tasks.service.js';    // ✅ .js even from .ts
```

The `.js` extension on a TypeScript import looks wrong and is correct — the specifier refers to the emitted file.

**`tsconfig.json` and `package.json` change together:**

```jsonc
// package.json
{ "type": "module" }

// tsconfig.json
{ "compilerOptions": { "module": "NodeNext", "moduleResolution": "NodeNext" } }
```

**Some dependencies are CJS-only.** Node can import CommonJS from ESM, but only the default export reliably — named imports from a CJS package may not resolve, and the failure is at runtime:

```ts
import pkg from 'some-cjs-package';
const { namedThing } = pkg;                // the common workaround
```

**Decorator metadata still works.** `experimentalDecorators` and `emitDecoratorMetadata` are unaffected by the module system, so the DI mechanism from [Chapter 2](02-typescript-decorators.md) is unchanged. This is the most common upgrade fear and it is unfounded.

**Prepare now, in v11.** These are all v11-compatible and remove most of the eventual work:

- Replace `__dirname`/`__filename` with `import.meta.url` equivalents behind a helper.
- Remove dynamic `require()` calls, especially conditional ones.
- Audit dependencies for ESM support — `are-the-types-wrong` and `publint` both report this.
- Move to `NodeNext` resolution before switching `type` to `module`, so extension errors surface incrementally.

---

## 36.4 Standard Schema, and Whether to Move

Standard Schema is a common interface several validation libraries implement, so a framework can accept any of them without depending on one.

| | `class-validator` | Standard Schema (Zod, Valibot, ArkType) |
|---|---|---|
| Definition | decorated class | a schema value |
| Type source | the class | inferred from the schema |
| Runtime cost | reflection per request | direct validation |
| Needs decorators | yes | no |
| Swagger integration | mature | improving |
| Composition | mapped types | schema combinators |

Two real advantages: types are **inferred** from the schema rather than declared twice, so they cannot drift; and the schema is a value, so it can be composed, reused between client and server, and used outside a request.

Two real costs: the ecosystem around `class-validator` — particularly Swagger generation and `class-transformer` serialization — is mature, and the fail-open traps in [Chapter 14](14-pipes-validation.md) are replaced by different ones you have not learned yet.

The pragmatic position is that both will be supported. Adopt Standard Schema for new endpoints where inference helps most — heavily nested payloads, shapes shared with a frontend — and leave working `class-validator` DTOs alone. A migration whose only justification is that the new thing is newer is not worth the regression risk on a validation layer.

---

## 36.5 Choosing Packages That Survive

Three signals predict whether a dependency will still work after the transition:

**ESM support already.** A package publishing `exports` with both CJS and ESM entry points has done the work. `publint` reports this in seconds.

**Maintenance, not popularity.** `csurf` was extremely popular and is deprecated and unmaintained (→ [Ch 23](23-security.md)). Check the last release date and open-issue response times, not the download count.

**Official or well-known.** `@nestjs/*` packages move with the framework. Community packages vary, and a `@nestjs/`-shaped name does not mean it is official.

The version-sensitive packages in this book, and their status:

| Package | Note |
|---|---|
| `@nestjs/config` | stable; unaffected by ESM |
| `@nestjs/typeorm`, `typeorm` | stable; verify ESM support at upgrade |
| `@nestjs/bullmq` | stable; BullMQ is ESM-ready |
| `@nestjs/cache-manager` | v6/Keyv in Nest 11 — the TTL unit change bites here |
| `@nestjs/passport` | stable; some strategies are CJS-only |
| `class-validator`, `class-transformer` | supported; joined by Standard Schema in v12 |
| `csurf` | **deprecated** — use `csrf-csrf` |
| `@nestjs/swagger` | stable; Standard Schema integration still maturing |

---

## 36.6 Common Mistakes and Best Practices

**Mistake: upgrading to Nest 11 without reading the Express 5 route changes.** The error names `path-to-regexp`, not your route.

**Mistake: copying `ttl: 60` from a v5 cache-manager example.** In v6 that is 60 milliseconds.

**Mistake: treating ESM as a flag.** It changes resolution, `__dirname`, and every relative import.

**Mistake: omitting `.js` from relative imports in ESM.** Resolution fails at runtime, not at compile time.

**Mistake: building on `next` prereleases.** They exist for migration testing.

**Mistake: assuming ESM breaks decorators.** It does not; the DI mechanism is unaffected.

**Mistake: choosing packages by download count.** Popularity and maintenance are different things.

**Best practice: do the ESM-compatible work now, in v11.** Removing `require` and `__dirname` is most of the migration and needs no upgrade.

**Best practice: upgrade one major version at a time.** 10 → 11 → 12, with the application working in between.

**Best practice: wait for a few patch releases.** The first stable of a major with this much infrastructure change will have follow-ups.

---

## 36.7 Real-World Example: An Upgrade Readiness Audit

A script that reports what will break, run against a v11 codebase before any upgrade work begins. It reads the repository rather than the release notes, so it tells you about *your* code.

```ts
// scripts/upgrade-audit.ts
import { readFileSync } from 'node:fs';
import { globSync } from 'node:fs';
import { join } from 'node:path';

interface Finding {
  severity: 'blocker' | 'warning' | 'info';
  rule: string;
  file: string;
  line: number;
  detail: string;
}

const SOURCE = globSync('src/**/*.ts', { exclude: (p) => p.endsWith('.spec.ts') });

/** Patterns that stop an ESM migration outright. */
const ESM_BLOCKERS: Array<{ rule: string; pattern: RegExp; detail: string }> = [
  {
    rule: 'esm/no-require',
    pattern: /\brequire\s*\(/,
    detail: '`require()` is not available in ESM — use import or createRequire',
  },
  {
    rule: 'esm/no-dirname',
    pattern: /\b__dirname\b|\b__filename\b/,
    detail: 'Use dirname(fileURLToPath(import.meta.url))',
  },
  {
    rule: 'esm/extensionless-relative-import',
    // Relative specifiers without an extension fail at RUNTIME under
    // NodeNext resolution, which is why this is a blocker rather than a hint.
    pattern: /from\s+['"]\.{1,2}\/[^'"]*(?<!\.js|\.json|\.node)['"]/,
    detail: 'Relative imports need an explicit .js extension under NodeNext',
  },
];

/** Already broken on Nest 11, or silently changed meaning. */
const V11_ISSUES: Array<{ rule: string; pattern: RegExp; detail: string; severity: Finding['severity'] }> = [
  {
    rule: 'express5/unnamed-wildcard',
    pattern: /@(Get|Post|Put|Patch|Delete|All)\(\s*['"][^'"]*\*(?![A-Za-z])/,
    detail: 'Express 5 requires named wildcards, e.g. "*splat"',
    severity: 'blocker',
  },
  {
    rule: 'express5/optional-param',
    pattern: /@(Get|Post|Put|Patch|Delete|All)\(\s*['"][^'"]*:\w+\?/,
    detail: 'Express 5 uses brace syntax for optional segments: ":name{.:ext}"',
    severity: 'blocker',
  },
  {
    rule: 'cache/ttl-unit',
    // A number under 1000 is almost certainly seconds copied from v5 docs.
    // Flagged as a warning rather than a blocker because it is a heuristic —
    // a deliberate 500ms cache is legal, just very unusual.
    pattern: /\bttl:\s*(?:[1-9]|[1-9]\d|[1-9]\d{2})\b/,
    detail: 'cache-manager v6 uses milliseconds; a small ttl is probably seconds from v5',
    severity: 'warning',
  },
  {
    rule: 'security/deprecated-csurf',
    pattern: /['"]csurf['"]/,
    detail: 'csurf is deprecated and unmaintained — use csrf-csrf',
    severity: 'blocker',
  },
];

function audit(): Finding[] {
  const findings: Finding[] = [];

  for (const file of SOURCE) {
    const lines = readFileSync(file, 'utf8').split('\n');

    lines.forEach((line, index) => {
      // Skip comments so a code sample in a docblock is not reported.
      if (/^\s*(\/\/|\*|\/\*)/.test(line)) return;

      for (const { rule, pattern, detail } of ESM_BLOCKERS) {
        if (pattern.test(line)) {
          findings.push({ severity: 'blocker', rule, file, line: index + 1, detail });
        }
      }
      for (const { rule, pattern, detail, severity } of V11_ISSUES) {
        if (pattern.test(line)) {
          findings.push({ severity, rule, file, line: index + 1, detail });
        }
      }
    });
  }

  return findings;
}

function auditDependencies(): Finding[] {
  const pkg = JSON.parse(readFileSync('package.json', 'utf8'));
  const deps = { ...pkg.dependencies, ...pkg.devDependencies };
  const findings: Finding[] = [];

  for (const [name, version] of Object.entries<string>(deps)) {
    let manifest: Record<string, unknown>;
    try {
      manifest = JSON.parse(
        readFileSync(join('node_modules', name, 'package.json'), 'utf8'),
      );
    } catch {
      continue;                              // not installed; skip rather than fail
    }

    // No `exports` and no `module` field means CJS-only, which does not block
    // the migration but does mean named imports may not resolve.
    const hasEsm = Boolean(manifest.exports) || manifest.type === 'module' || manifest.module;
    if (!hasEsm) {
      findings.push({
        severity: 'info',
        rule: 'esm/cjs-only-dependency',
        file: 'package.json',
        line: 0,
        detail: `${name}@${version} appears CJS-only — named imports may need a default-import workaround`,
      });
    }
  }
  return findings;
}

const all = [...audit(), ...auditDependencies()];
const blockers = all.filter((f) => f.severity === 'blocker');

for (const f of all) {
  console.log(`${f.severity.padEnd(7)} ${f.rule.padEnd(36)} ${f.file}:${f.line}\n        ${f.detail}`);
}
console.log(`\n${blockers.length} blockers, ${all.length - blockers.length} other findings`);

// Non-zero exit only for blockers, so this can run in CI as a ratchet:
// new blockers fail the build while existing warnings are worked through.
process.exit(blockers.length > 0 ? 1 : 0);
```

The decision worth defending is exiting non-zero only for blockers while still printing warnings and info. A script that fails on every finding cannot be adopted — a large codebase will have hundreds of extensionless imports on day one, so the build breaks immediately and the check gets disabled within a week, which is the worst outcome for a tool whose value is continuous. Failing only on blockers lets the audit run in CI from the moment it is written: existing warnings are visible in every build without stopping anyone, and the categories that genuinely break a deploy — an Express 5 wildcard, a deprecated security package — cannot be newly introduced. The `ttl` heuristic is deliberately a warning rather than a blocker for the same reason: it cannot distinguish a v5 copy-paste from an intentional 500 ms cache, and a check that produces confident false positives trains people to ignore it. An upgrade audit is only useful if it survives contact with a deadline, and that means being honest about which findings are certain.

---

## 36.8 Questions

**1.** MCQ — NestJS 11's minimum Node version is: a) 16 b) 18 c) 20 d) 22
<details><summary>Answer</summary>**c)** Node 18 produces obscure runtime failures rather than a clean error.</details>

**2.** MCQ — In `cache-manager` v6, `ttl: 60` means: a) 60 seconds b) 60 milliseconds c) 60 minutes d) no expiry
<details><summary>Answer</summary>**b)** The unit changed from seconds in v5.</details>

**3.** MCQ — Which is NOT a pillar of NestJS 12? a) full ESM b) Standard Schema c) dropping decorators d) Rspack
<details><summary>Answer</summary>**c)** Decorators are unaffected; the DI mechanism is unchanged.</details>

**4.** MCQ — In ESM, a relative import of a TypeScript file must end with: a) `.ts` b) `.js` c) nothing d) `.mjs`
<details><summary>Answer</summary>**b)** The specifier refers to the emitted file.</details>

**5.** MCQ — Vitest becomes the default runner for: a) all v12 projects b) new ESM projects c) CJS projects d) tests using `@nestjs/testing`
<details><summary>Answer</summary>**b)** CommonJS projects keep Jest.</details>

**6.** MCQ — Standard Schema lets route decorators accept: a) only Zod b) any implementing library c) only class-validator d) JSON Schema only
<details><summary>Answer</summary>**b)** Zod, Valibot, and ArkType all implement it.</details>

**7.** Short — Which Nest 11 change most often stops a deploy outright?
<details><summary>Answer</summary>Express 5 route syntax — a bare `*` wildcard or a `?` optional parameter throws during route registration, and the error names `path-to-regexp` rather than your controller.</details>

**8.** Short — Why is ESM the expensive part of the v12 upgrade?
<details><summary>Answer</summary>It changes module resolution itself: `require`, `__dirname`, and extensionless relative imports all stop working, and some dependencies are CJS-only.</details>

**9.** Short — Does ESM break decorator metadata?
<details><summary>Answer</summary>No. `experimentalDecorators` and `emitDecoratorMetadata` are independent of the module system, so dependency injection is unaffected.</details>

**10.** Short — What can you do now, on v11, to reduce the eventual migration?
<details><summary>Answer</summary>Remove `require()` and `__dirname` usage, move to `NodeNext` resolution so extension errors surface incrementally, and audit dependencies for ESM support.</details>

**11.** Short — What is the main advantage of Standard Schema over `class-validator`?
<details><summary>Answer</summary>Types are inferred from the schema rather than declared alongside it, so the runtime rules and the static type cannot drift apart.</details>

**12.** Output — `@Get('files/*')` on NestJS 11. What happens?
<details><summary>Answer</summary>Startup fails with an error from `path-to-regexp`; Express 5 requires a named wildcard such as `*path`.</details>

**13.** Output — `import { TasksService } from './tasks.service'` under `NodeNext`. When does it fail?
<details><summary>Answer</summary>At runtime, on module resolution — not at compile time, which is what makes it easy to miss.</details>

**14.** Output — `console.log(__dirname)` in an ESM module. What happens?
<details><summary>Answer</summary>A `ReferenceError` — `__dirname` is not defined in ESM.</details>

**15.** Output — A cache configured with `ttl: 30` on Nest 11. How long do entries live?
<details><summary>Answer</summary>30 milliseconds, so effectively nothing is cached.</details>

**16.** Debug — After upgrading to Nest 11, teardown throws because a dependency is already destroyed. Cause?
<details><summary>Answer</summary>Nest 11 reversed the module traversal order in the termination phases. Move the work to `beforeApplicationShutdown` or stop depending on another module during teardown.</details>

**17.** Debug — A named import from a dependency is `undefined` after switching to ESM. Fix?
<details><summary>Answer</summary>The package is CJS-only, so named exports may not resolve. Import the default and destructure from it.</details>

**18.** Debug — A codebase upgraded from 10 to 12 in one step and nothing works. What should have happened?
<details><summary>Answer</summary>One major at a time — 10 → 11, verify, then 11 → 12 — so failures are attributable to a single set of changes.</details>

**19.** Debug — CSRF protection stopped receiving security updates. Which package is it?
<details><summary>Answer</summary>`csurf`, which is deprecated and unmaintained. Replace it with `csrf-csrf` (→ Ch 23).</details>

**20.** Debug — An upgrade audit script was added to CI and disabled within a week. What was likely wrong with it?
<details><summary>Answer</summary>It failed the build on every finding, so a codebase with hundreds of pre-existing warnings could not adopt it. Fail only on blockers and report the rest.</details>

**21.** Code — Replace `__dirname` in a way that works in ESM.
<details><summary>Answer</summary>

```ts
import { dirname } from 'node:path';
import { fileURLToPath } from 'node:url';

const __dirname = dirname(fileURLToPath(import.meta.url));
```
Put it in one helper rather than repeating it in every file that needs a path.
</details>

**22.** Code — Configure `tsconfig.json` and `package.json` for ESM.
<details><summary>Answer</summary>

```jsonc
// package.json
{ "type": "module" }

// tsconfig.json
{
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "experimentalDecorators": true,     // still required — ESM does not change this
    "emitDecoratorMetadata": true
  }
}
```
</details>

**23.** Code — Convert two Express 4 routes to Express 5 syntax.
<details><summary>Answer</summary>

```ts
@Get('files/*path')            // was 'files/*'
findFile(@Param('path') path: string) {}

@Get('report/:name{.:ext}')    // was 'report/:name.:ext?'
getReport(@Param('name') name: string, @Param('ext') ext?: string) {}
```
</details>

**24.** Scenario — Your team wants to upgrade to v12 the week it is released. Evaluate it.
<details><summary>Answer</summary>Wait, and use the time to prepare. A major release changing the module system, the test runner, the bundler, and the linter simultaneously will have follow-up patches in its first weeks, and being the team that finds those bugs costs more than the delay. The preparation is also independent of the release: removing `require` and `__dirname`, moving to `NodeNext` resolution, and auditing dependencies for ESM support are all v11-compatible and constitute most of the work. Run the prereleases against a branch to discover what breaks in *your* code, keep that branch rebased, and adopt after a few patch versions when the ecosystem packages you depend on have published compatible releases. Being early on a framework major buys nothing that being second does not, and the risk is entirely one-sided.</details>

**25.** Scenario — With Standard Schema arriving, should you migrate an existing `class-validator` codebase to Zod?
<details><summary>Answer</summary>Not as a project in its own right. The validation layer is one where regressions fail open — an accidentally weakened rule accepts bad data silently rather than erroring (→ Ch 14) — so a wholesale rewrite carries a specific, high-consequence risk in exchange for benefits that are real but incremental. Both approaches will be supported, so there is no forcing function. Use Zod for new endpoints where inference genuinely helps: deeply nested payloads, schemas shared with a TypeScript frontend, or validation needed outside a request context. Leave working DTOs alone, and if you do convert one, keep the old tests and add tests asserting rejection for every rule, since that is the only way to detect a rule that quietly stopped applying.</details>

---

## Chapter Summary

- NestJS 11's breaking changes are Node 20, Express 5 route syntax, reversed termination hook order, and `cache-manager` v6's millisecond TTLs — and the route syntax is the one that stops a deploy.
- NestJS 12 rests on full ESM, Standard Schema in route decorators, Vitest for new ESM projects, and Rspack plus oxlint in the toolchain.
- ESM is the expensive pillar: it removes `require` and `__dirname` and requires explicit `.js` extensions on relative imports, which fail at runtime rather than at compile time.
- Decorators and dependency injection are unaffected by ESM, which is the most common and least founded upgrade fear.
- Most of the migration can be done now on v11 — removing `require`, moving to `NodeNext`, auditing dependencies — without touching the framework version.
- Standard Schema is worth adopting for new endpoints where type inference helps, and rewriting a working `class-validator` layer is a fail-open risk taken for an incremental gain.

---

## Practice Exercises

1. Write the upgrade audit script from §36.7 against a real project and triage what it reports.
2. Switch one small project to `"type": "module"` and fix every resolution error, timing the work.
3. Replace every `__dirname` in a codebase with an `import.meta.url` helper.
4. Run `publint` over your dependency list and count the CJS-only packages.
5. Implement one endpoint twice — `class-validator` and Zod — and compare the type-safety and the failure messages.

---

## Mini Project: An Upgrade Readiness Report

Produce the document your team would actually use to plan the v12 upgrade.

**Requirements:**

- An automated audit reporting ESM blockers, Express 5 route issues, and deprecated packages, exiting non-zero only on blockers.
- A dependency table listing ESM support and last release date for every direct dependency.
- A branch running the `next` prereleases with a record of exactly what broke.
- An effort estimate per category, separating work that can be done now on v11 from work that requires v12.
- A recommended sequence and a go/no-go criterion for adoption.

---

**Next:** [Appendix A — Interview Questions →](../appendices/A-interview-questions.md)
