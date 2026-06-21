---
paths:
  - "apps/api/src/**/entities/**"
  - "apps/api/src/**/*.entity.ts"
---

ENFORCEMENT: blocking

# Handbook: MikroORM `@Property()` needs explicit options (marsa apps/api)

**Scope:** PRs touching MikroORM entities (`apps/api/src/**/*.entity.ts`) in the marsa NestJS backend.
**Enforcement:** blocking

## The rule

Every `@Property()` declares explicit column options. A bare empty `@Property()` is not allowed — at minimum state the `type`, and add `nullable` / `length` / `unique` / `default` wherever the column has those semantics.

```ts
// avoid — vague, leans entirely on inference
@Property()
githubAppId!: string

// prefer — explicit intent, the column shape is reviewable from the entity
@Property({ type: 'string', unique: true })
githubAppId!: string
```

## Why

A bare `@Property()` makes the column shape invisible at the entity — reviewers and the next author can't tell the type, length, nullability, or uniqueness without running a migration diff. Leaning on MikroORM's TS-type inference also silently couples the DB column to whatever the field's TS type happens to be, so a later type change mutates the schema by accident. Explicit options make the entity the readable source of truth for the table and make migration review meaningful.

## What Rex flags

- A bare `@Property()` (no options object) on a column in any `*.entity.ts` under `apps/api/src/**`.
- A `@Property()` whose column clearly needs constraints the decorator omits (a unique identifier without `unique`, a free-text blob without `type: 'text'`).

## Sample finding

> blocking: bare `@Property()` on `githubAppId` is vague — declare explicit
> options so the column shape is reviewable from the entity, e.g.
> `@Property({ type: 'string', unique: true })`. See
> `handbooks/domain/marsa-api/mikroorm-property-options.md`.

## What's NOT a violation

- `@PrimaryKey(...)` and relation decorators (`@ManyToOne`, `@OneToMany`) — different decorators with their own conventions.
- A `@Property({ ... })` that already carries a meaningful options object (e.g. `@Property({ nullable: true })`, `@Property({ onUpdate: () => new Date() })`).
- Virtual / non-persisted properties (`@Property({ persist: false })`) and formula columns.

---

_Source: PR #64 comment by @G0maa on 2026-06-08_
_See: https://github.com/marsa-cloud/marsa/pull/64_
