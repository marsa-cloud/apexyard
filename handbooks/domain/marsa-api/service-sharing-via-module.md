---
paths:
  - "apps/api/src/app/**"
  - "apps/api/src/modules/**"
---

ENFORCEMENT: advisory

# Handbook: Share a service via its own module (marsa apps/api)

**Scope:** PRs touching `apps/api/src/app/**` or `apps/api/src/modules/**` in the marsa NestJS backend.
**Enforcement:** advisory

## The rule

When a provider (service) is needed by more than one use-case, put it in its **own module** that `providers` **and** `exports` it, and `imports` that module wherever it's needed. Don't re-list the same provider in each consuming use-case module — that creates a separate instance per module and scatters the wiring.

```ts
@Module({ providers: [ManifestStateService], exports: [ManifestStateService] })
export class ManifestStateModule {}

// each consumer:
@Module({ imports: [ManifestStateModule], providers: [GetManifestUseCase] })
export class GetManifestModule {}
```

## Why

Re-declaring a provider in two modules gives two instances — fine for stateless helpers, a latent bug for anything holding state, a connection, or a cache. A single exporting module gives one shared instance, one place to configure it, and a clear dependency edge. It's the idiomatic Nest pattern (mirrors `CryptoModule` / `GitHubClientModule` here).

## What Rex flags

- The same provider class listed in the `providers` array of two or more modules, instead of one module that exports it and the others importing it.

## What's NOT a violation

- A provider used by exactly one module — keep it local until a second consumer appears.
- Request-scoped or deliberately per-module providers where a fresh instance is the intent.

---

_Source: PR #64 comment by @G0maa on 2026-06-10_
_See: https://github.com/marsa-cloud/marsa/pull/64_
