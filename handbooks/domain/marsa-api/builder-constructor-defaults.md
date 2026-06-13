---
paths:
  - "apps/api/src/app/**"
---

ENFORCEMENT: blocking

# Handbook: Builders declare a constructor with sensible defaults (marsa apps/api)

**Scope:** PRs touching `apps/api/src/app/**` that add or change a `<Entity>Builder` or `<Action>CommandBuilder` in the marsa NestJS backend.
**Enforcement:** blocking

## The rule

Every builder (`<Entity>Builder`, `<Action>CommandBuilder`) declares an explicit **constructor** that seeds the wrapped object with **sensible defaults** for its required fields. A test that only cares about one field should still get a buildable, valid object without calling every `withX()`.

```ts
export class GitHubInstallationBuilder {
  private readonly installation: GitHubInstallation

  constructor() {
    this.installation = new GitHubInstallation()
    this.installation.installationId = '1'
    this.installation.accountLogin = null
  }

  withInstallationId(installationId: string): this {
    this.installation.installationId = installationId
    return this
  }
  // ... one withX per field

  build(): GitHubInstallation {
    return this.installation
  }
}
```

When you add this convention to one builder, **update the sibling builders in the same PR** so the pattern stays uniform (the CEO called this out explicitly on #70).

## Why

A builder whose fields start `undefined` forces every call site to spell out every required `withX()` just to get a valid object — which defeats the "intent-first, set only what this test cares about" point of the builder. Seeding required fields in the constructor means `new XBuilder().build()` is always valid, and a test overrides only the field under test.

## What Rex flags

- A `<Entity>Builder` / `<Action>CommandBuilder` under `apps/api/src/app/**` with **no constructor** (relying on `private readonly x = new Entity()` field initialisation) where required fields are left `undefined`.
- A PR that adds the constructor-defaults pattern to one builder but leaves sibling builders in the same module untouched.

## Sample finding

> blocking: `GitHubInstallationBuilder` has no constructor, so `installationId`
> starts `undefined` and every test must call `.withInstallationId(...)` to get a
> valid object. Add a constructor seeding sensible defaults for required fields,
> and apply the same to the sibling builders in this PR. See
> `handbooks/domain/marsa-api/builder-constructor-defaults.md`.

## What's NOT a violation

- Optional/nullable fields left unset in the constructor (e.g. `accountLogin = null` is fine, but it need not be seeded to a fake value).
- A builder whose only required field is already set by a `withX()` that every realistic call must pass — though seeding a default is still preferred for consistency.
- The constructor assigning fields directly — that's the one place inline assignment belongs (see `handbooks/domain/marsa-api/entity-builder.md`).

---

_Source: PR #70 comment by @G0maa on 2026-06-11_
_See: https://github.com/marsa-cloud/marsa/pull/70_
