---
paths:
  - "apps/api/src/app/**"
---

ENFORCEMENT: blocking

# Handbook: Model relations with an explicit foreign-key reference (marsa apps/api)

**Scope:** PRs touching `apps/api/src/app/**` entities in the marsa NestJS backend.
**Enforcement:** blocking

## The rule

An entity that belongs to another models the relation explicitly as a typed FK reference. Use `@ManyToOne(() => Parent, { ref: true })` typed `Ref<Parent>` so the foreign key is first-class (`child.app.uuid` without loading the row), and don't add a duplicate scalar FK column alongside it (MikroORM would double-map).

```ts
import { type Ref, ManyToOne } from '@mikro-orm/core'

@ManyToOne(() => GitHubApp, { nullable: false, ref: true })
app!: Ref<GitHubApp>
```

## Why

Without an explicit relation the FK is implicit or missing, and code can't read the parent key without a join/load. A typed `Ref` makes the column (`app_uuid` → `github_app.uuid`) explicit, lets callers read the key without hydrating the parent, and keeps the relation the readable source of truth. Modelling the FK via the reference (not a parallel scalar column) avoids MikroORM mapping the same column twice.

## What Rex flags

- A child entity that conceptually belongs to a parent but has no `@ManyToOne`/relation decorator (the FK is missing).
- A `@ManyToOne` without `ref: true` where the code reads only the parent key (forces a needless load), or a relation paired with a hand-rolled duplicate `parentUuid` scalar column.

## Sample finding

> blocking: `GitHubInstallation` belongs to a `GitHubApp` but the foreign key isn't
> modelled. Add `@ManyToOne(() => GitHubApp, { nullable: false, ref: true })` typed
> `Ref<GitHubApp>` so the FK (`app_uuid`) is explicit and the parent key is readable
> without a load. See `handbooks/domain/marsa-api/explicit-fk-relation.md`.

## What's NOT a violation

- A relation that genuinely needs the full parent loaded eagerly (then a non-`ref` `@ManyToOne` is a deliberate choice — note why).
- `@OneToMany` / `@ManyToMany` inverse sides, which don't own the FK column.
- A standalone entity with no parent.

---

_Source: PR #70 comment by @G0maa on 2026-06-11_
_See: https://github.com/marsa-cloud/marsa/pull/70_
