---
paths:
  - "apps/api/src/app/**"
---

ENFORCEMENT: blocking

# Handbook: Nullable column types reflect nullability (marsa apps/api)

**Scope:** PRs touching `apps/api/src/app/**` entities in the marsa NestJS backend.
**Enforcement:** blocking

## The rule

When a MikroORM `@Property({ nullable: true })` column can hold `null`, the TS field type must say so — use `?: T | null`, not just `?: T`.

```ts
// wrong — the type claims it's only ever string-or-absent
@Property({ type: 'string', length: 255, nullable: true })
accountLogin?: string

// right — the type matches what the column (and a DB read) can return
@Property({ type: 'string', length: 255, nullable: true })
accountLogin?: string | null
```

## Why

`nullable: true` means a hydrated entity (or an `upsert` result) can carry `null`. A bare `?: string` type lies about that — downstream code that does `if (x.accountLogin)` is fine, but code that passes the field on, compares it, or maps it gets a type that excludes the `null` it will actually see at runtime. Matching the type to the column keeps the entity the honest source of truth for the table.

## What Rex flags

- A `@Property({ ..., nullable: true })` (or a `nullable: true` relation) whose TS field type is `?: T` or `: T` without `| null`.
- A builder/`withX` setter for such a field whose parameter type omits `| null`.

## Sample finding

> blocking: `accountLogin` is `nullable: true` but typed `?: string`, so the type
> hides the `null` a DB read can return. Type it `?: string | null` (and match the
> builder setter). See `handbooks/domain/marsa-api/nullable-column-type.md`.

## What's NOT a violation

- A non-nullable column (`nullable` absent or `false`) typed `: T` — correct as-is.
- A field that is genuinely "absent vs present" with no DB-null semantics (rare for persisted columns; common for in-memory DTOs).

---

_Source: PR #70 comment by @G0maa on 2026-06-11_
_See: https://github.com/marsa-cloud/marsa/pull/70_
