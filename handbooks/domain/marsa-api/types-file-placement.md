---
paths:
  - "apps/api/src/**"
---

ENFORCEMENT: blocking

# Handbook: Shared types live in a `.types.ts` file (marsa apps/api)

**Scope:** PRs touching `apps/api/src/**` in the marsa NestJS backend.
**Enforcement:** blocking

## The rule

Standalone `type` / `interface` declarations that are shared across files (or describe a module's public shapes) live in a `.types.ts` file at a sensible placement — co-located with the module/use-case that owns them — not inline at the top of a service/use-case.

```ts
// github-client.types.ts
export interface InstallationTokenParams {
  installationId: string
  privateKeyPem: string
}
export interface GitHubAppCredentials { /* ... */ }
```

## Why

A type declared inline in a service file gets imported from that service by other modules, coupling them to the implementation file rather than to a types module. A dedicated `.types.ts` gives shared shapes one obvious home, keeps the service file about behaviour, and avoids import cycles when the type and the impl reference each other.

## What Rex flags

- An exported `type`/`interface` declared at the top of a `*.service.ts` / `*.use-case.ts` / `*.client.ts` that is imported by another file (should be in `*.types.ts`).
- A cluster of related shared shapes scattered inline rather than gathered in a `.types.ts`.

## Sample finding

> blocking: `InstallationTokenParams` is declared inline in the service but consumed
> elsewhere. Move it (and the related shapes) to `github-client.types.ts`. See
> `handbooks/domain/marsa-api/types-file-placement.md`.

## What's NOT a violation

- A small, file-private type used only within its declaring file (not exported, not shared).
- DTO classes (`@ApiProperty` command/response classes) — those follow the command/response conventions, not `.types.ts`.
- Entities — those live in `entities/`.

---

_Source: PR #70 comment by @G0maa on 2026-06-11_
_See: https://github.com/marsa-cloud/marsa/pull/70_
