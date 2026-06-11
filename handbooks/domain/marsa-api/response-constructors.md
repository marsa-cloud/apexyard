---
paths:
  - "apps/api/src/app/**/use-cases/**"
---

ENFORCEMENT: blocking

# Handbook: Response DTOs use constructors (marsa apps/api)

**Scope:** PRs touching `apps/api/src/app/<feature>/use-cases/**` in the marsa NestJS backend.
**Enforcement:** blocking

## The rule

Response DTOs (`<use-case>.response.ts`) declare an explicit constructor that takes the response fields, and use-cases return them via `new <Action>Response(...)`.

- Do **not** leave a response class with only definite-assignment fields (`appSlug!: string`) and no constructor.
- Do **not** build the response by assembling an object literal and casting (`return { ... } as ConvertManifestResponse`) or by mutating fields one at a time after construction.

```ts
export class ConvertManifestResponse {
  @ApiProperty(...) readonly appSlug: string
  // ...
  constructor(appSlug: string, appName: string, htmlUrl: string, installUrl: string) {
    this.appSlug = appSlug
    // ...
  }
}

// in the use-case:
return new ConvertManifestResponse(slug, name, htmlUrl, installUrl)
```

## Why

A constructor makes the response's required fields explicit and checked by the compiler — you cannot forget one. Object-literal-plus-cast bypasses the type check (`as` silences missing fields), so a partially-built response compiles and ships a `undefined` field to the web client. Centralising construction in the constructor also gives one place to add invariants later.

## What Rex flags

- A `*.response.ts` class with `@ApiProperty` fields but **no constructor**.
- A use-case returning `{ ... } as <Action>Response`, or assigning response fields one-by-one (`const r = new Response(); r.appSlug = ...`).

## Sample finding

> blocking: `ConvertManifestResponse` has no constructor — add one taking
> the response fields and `return new ConvertManifestResponse(...)` from the
> use-case instead of building the object inline. See
> `handbooks/domain/marsa-api/response-constructors.md`.

## Also — take the type, not loose fields (advisory)

_Advisory: flag as a `suggestion:`, not blocking, until the existing responses migrate._

When the response's fields are exactly (or mostly) a domain entity or an existing shared type, the constructor should take **that type**, not a long list of separated primitives:

```ts
// prefer
constructor(app: GitHubApp) { this.appSlug = app.slug; /* … */ }
// over
constructor(appSlug: string, appName: string, htmlUrl: string, installUrl: string) { … }
```

And a **nested object** field must be its own `@ApiProperty()`-decorated class, never an exported `interface` typed with `additionalProperties: true` — an interface produces no OpenAPI schema, so the web's generated types degrade to `Record<string, unknown>`. (`GetManifestResponse.manifest` is the current offender: typed `GitHubAppManifest` interface; the fix is a `ManifestDto` class.)

## What's NOT a violation

- An empty response (no body fields) — no constructor needed.
- A static factory method (`ConvertManifestResponse.fromEntity(app)`) that internally calls the constructor — that's the constructor pattern, just named.
- Request/command DTOs deserialised by Nest from the HTTP body — those are populated by the framework, not constructed by us.
- A response that genuinely composes fields from several sources (not a single entity) — separated constructor params are fine there.

---

_Source: PR #64 comment by @G0maa on 2026-06-08_
_See: https://github.com/marsa-cloud/marsa/pull/64_
