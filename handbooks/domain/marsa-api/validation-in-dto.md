---
paths:
  - "apps/api/src/app/**/use-cases/**"
---

ENFORCEMENT: advisory

# Handbook: Validate at the DTO boundary, not in the use-case (marsa apps/api)

**Scope:** PRs touching `apps/api/src/app/<feature>/use-cases/**` in the marsa NestJS backend.
**Enforcement:** advisory

## The rule

Presence / type / shape validation of the request body lives on the command DTO as `class-validator` decorators, enforced by the global `ValidationPipe`. Don't re-check the same things by hand inside the use-case.

```ts
// convert-manifest.command.ts — the validation lives here
export class ConvertManifestCommand {
  @IsString() @IsNotEmpty() code!: string
  @IsString() @IsNotEmpty() state!: string
}

// use-case — DON'T do this; the ValidationPipe already rejected empty/non-string
if (typeof command.code !== 'string' || command.code.length === 0) {
  throw new BadRequestException('code is required')
}
```

## Why

The `ValidationPipe` runs before the controller, so a manual guard in the use-case is dead code on the request path — it only ever fires for callers that bypass the pipe (i.e. tests). Two validators for one rule drift apart: the DTO says `@IsNotEmpty()` while the use-case re-implements a subtly different check, and now the real contract is split across two files. Keep one source of truth at the boundary; the use-case assumes a validated command.

## What Rex flags

- A use-case manually checking `typeof x !== 'string'`, `x.length === 0`, null/undefined presence, etc. for a field the command DTO already validates with class-validator.

## What's NOT a violation

- **Business-rule** validation that class-validator can't express (cross-field invariants, DB-state checks, "this state token is unexpired") — that belongs in the use-case.
- A use-case reused outside an HTTP boundary where no pipe runs — but prefer validating the DTO once at the edge.

---

_Source: PR #64 comment by @G0maa on 2026-06-10_
_See: https://github.com/marsa-cloud/marsa/pull/64_
