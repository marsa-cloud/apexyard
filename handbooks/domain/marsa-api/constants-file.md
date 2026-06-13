---
paths:
  - "apps/api/src/**"
---

ENFORCEMENT: blocking

# Handbook: Magic values live in a `.constant.ts` file (marsa apps/api)

**Scope:** PRs touching `apps/api/src/**` in the marsa NestJS backend.
**Enforcement:** blocking

## The rule

Don't inline a magic string/number literal inside a use-case, service, or controller. Pull it into a named export in a `.constant.ts` (or `.constants.ts`) file — co-located in the use-case/module that owns it, or in the relevant feature module.

```ts
// capture-installation.constant.ts
export const INSTALL_SETUP_ACTION = 'install'

// used in the use-case / DTO instead of a bare 'install'
```

## Why

A bare literal duplicated across the use-case, its DTO validator, and its tests drifts when one copy changes. A named constant is the single definition, documents intent at the use site (`INSTALL_SETUP_ACTION` vs `'install'`), and lets the DTO `@IsIn([INSTALL_SETUP_ACTION])` and the use-case share one source.

## What Rex flags

- A domain-significant string/number literal used in logic (a comparison, a validator `@IsIn`/`@Matches` arg, a branch) inlined rather than referenced from a `.constant(s).ts` export — especially when the same literal appears in more than one file.

## Sample finding

> blocking: `'install'` is a magic string used in both the DTO validator and the
> use-case. Extract it to `capture-installation.constant.ts` as
> `INSTALL_SETUP_ACTION` and reference it from both. See
> `handbooks/domain/marsa-api/constants-file.md`.

## What's NOT a violation

- Genuinely local, self-evident literals (`0`, `1`, `''` defaults; an array index).
- Strings that are inherently single-use and self-documenting in place.
- Config/env-derived values (those belong in config, not a constants file).

---

_Source: PR #70 comment by @G0maa on 2026-06-11_
_See: https://github.com/marsa-cloud/marsa/pull/70_
