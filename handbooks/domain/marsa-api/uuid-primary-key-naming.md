---
paths:
  - "apps/api/src/app/**"
---

ENFORCEMENT: blocking

# Handbook: UUID primary keys are named `uuid` (marsa apps/api)

**Scope:** PRs touching `apps/api/src/app/**` entities in the marsa NestJS backend.
**Enforcement:** blocking

## The rule

When an entity's primary key is a UUID, name the field `uuid` — not `id`.

```ts
// wrong
@PrimaryKey({ type: 'uuid' })
id: string = randomUUID()

// right
@PrimaryKey({ type: 'uuid' })
uuid: string = randomUUID()
```

Apply this to **every** entity in the codebase, not just new ones — fix existing `id`-named UUID PKs when you touch them.

## Why

A field named `id` reads as an opaque/auto-increment surrogate; naming it `uuid` makes the key's type self-documenting at every call site (`app.uuid`, FK `app_uuid`) and keeps the convention uniform so a reviewer never has to check the decorator to know what the key is.

## What Rex flags

- A `@PrimaryKey({ type: 'uuid' })` (or `randomUUID()`-initialised PK) whose field is named `id`.
- A foreign-key column or join reference pointing at a UUID PK but named `*_id` instead of `*_uuid`.

## Sample finding

> blocking: This UUID primary key is named `id`. Rename it to `uuid` (and any
> `*_id` FK columns that reference it to `*_uuid`) so the key type is
> self-documenting. See `handbooks/domain/marsa-api/uuid-primary-key-naming.md`.

## What's NOT a violation

- A non-UUID surrogate key (auto-increment integer) named `id` — that's correct.
- An external/third-party id stored as a column (e.g. `installationId`, `githubAppId`) — those name the upstream concept, not our PK.

---

_Source: PR #70 comment by @G0maa on 2026-06-11_
_See: https://github.com/marsa-cloud/marsa/pull/70_
