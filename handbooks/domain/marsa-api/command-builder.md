---
paths:
  - "apps/api/src/app/**/use-cases/**"
---

ENFORCEMENT: advisory

# Handbook: Test-side command builders (marsa apps/api)

**Scope:** PRs touching `apps/api/src/app/<feature>/use-cases/**` in the marsa NestJS backend.
**Enforcement:** advisory

## The rule

Command DTOs (`<use-case>.command.ts`, class `<Action>Command`) are class-validator DTOs that Nest deserialises from the HTTP body — on the request path you never hand-construct them. **In tests**, construct them through a `<Action>CommandBuilder` (`<use-case>.command.builder.ts`, fluent `withX().build()`) instead of inline object literals.

```ts
// convert-manifest.command.builder.ts
export class ConvertManifestCommandBuilder {
  private readonly command = new ConvertManifestCommand()
  withCode(code: string): this { this.command.code = code; return this }
  withState(state: string): this { this.command.state = state; return this }
  build(): ConvertManifestCommand { return this.command }
}

// in a test:
await usecase.execute(new ConvertManifestCommandBuilder().withCode('x').withState(s).build())
```

## Why

A builder gives tests one intent-first construction point that tracks the command's real shape — when a field is added, the builder (and a quick compile error at each call site) surfaces every test that needs the new field, instead of N scattered object literals silently shipping `undefined`. It mirrors the entity-builder convention so construction reads the same everywhere.

## What Rex flags

- A use-case **unit/e2e test** building a valid `<Action>Command` as an inline object literal (`{ code: 'x', state: s }`) where a `<Action>CommandBuilder` exists or should.

## What's NOT a violation

- The request path — controllers receive a Nest-deserialised DTO; no builder there.
- Deliberately-malformed inputs that bypass the type system (`{ code: 123 as unknown as string }`) to exercise a runtime guard — those stay inline because the builder's typed setters can't express them.

---

_Source: PR #64 comments by @G0maa on 2026-06-10_
_See: https://github.com/marsa-cloud/marsa/pull/64_
