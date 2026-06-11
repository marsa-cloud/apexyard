---
paths:
  - "apps/api/src/app/**"
---

ENFORCEMENT: blocking

# Handbook: Build complex entities via a Builder (marsa apps/api)

**Scope:** PRs touching `apps/api/src/app/**` that construct domain entities in the marsa NestJS backend.
**Enforcement:** blocking

## The rule

**Every** domain entity is constructed through a dedicated fluent `<Entity>Builder` — there is no field-count threshold. Don't assign entity fields inline (`const x = new Entity(); x.a = …`) in a use-case or service, regardless of how few fields the entity has.

```ts
export class GitHubAppBuilder {
  private readonly app = new GitHubApp()

  withGithubAppId(id: string): this {
    this.app.githubAppId = id
    return this
  }
  // ... one withX per field

  build(): GitHubApp {
    return this.app
  }
}
```

The builder lives next to the entity it constructs (e.g. `entities/github-app.builder.ts`, `entities/manifest-state.builder.ts`).

## Why

A builder centralises construction, reads intent-first at the call site, and is straightforward to unit-test. For a secret-bearing entity like `GitHubApp` (~10 columns, three of them AES-GCM ciphertext) it also gives one place to enforce "every secret column is the encrypted form, never plaintext". Applying the rule uniformly — including small entities like `ManifestState` — keeps the construction pattern consistent across the codebase, so a reviewer never has to re-litigate "is this entity big enough to warrant a builder?" and a small entity that later grows a field already has the seam.

## What Rex flags

- A use-case/service under `apps/api/src/app/**` constructing **any** entity by inline field mutation (`const x = new Entity(); x.a = …; x.b = …`).
- An object literal (`em.create(Entity, { … })`) assembling an entity where the `<Entity>Builder` should be used instead.

## Sample finding

> blocking: This assembles `GitHubApp` (10 fields, 3 of them encrypted
> secrets) inline. Introduce a `GitHubAppBuilder` (`entities/github-app.builder.ts`)
> with `withX().build()` so construction is centralised and the
> encrypted-column contract is enforced in one place. See
> `handbooks/domain/marsa-api/entity-builder.md`.

## What's NOT a violation

- Test fixtures that set a field or two on an already-built entity for a specific assertion.
- The builder itself assigning fields internally — that's the one place inline assignment belongs.
- ORM hydration (MikroORM populating an entity from a DB row) — that's the framework, not us.

---

_Source: PR #64 comments by @G0maa on 2026-06-08 (introduced) and 2026-06-10 (strengthened from "≈5+ fields" to "always")_
_See: https://github.com/marsa-cloud/marsa/pull/64_
