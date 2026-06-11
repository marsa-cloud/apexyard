---
paths:
  - "apps/api/src/app/**/use-cases/**"
---

ENFORCEMENT: advisory

# Handbook: Document thrown errors on the controller (marsa apps/api)

**Scope:** PRs touching `apps/api/src/app/<feature>/use-cases/**` in the marsa NestJS backend.
**Enforcement:** advisory

## The rule

Every HTTP error a use-case can throw must be declared on its controller with the matching `@Api*Response()` decorator, so the committed `openapi.json` reflects the full response set — not just the 200 happy path.

```ts
// use-case throws BadRequestException + BadGatewayException
@ApiOperation({ operationId: 'convertGithubAppManifestV1' })
@ApiOkResponse({ type: ConvertManifestResponse })
@ApiBadRequestResponse({ description: 'invalid or expired state, or missing code' })
@ApiResponse({ status: 502, description: 'GitHub App creation failed upstream' })
handle(@Body() body: ConvertManifestCommand): Promise<ConvertManifestResponse> { … }
```

## Why

The web generates its types + client behaviour from `openapi.json`. If the contract only advertises 200, the frontend has no typed knowledge of the 400/502 paths and handles them as untyped surprises. Documenting the error responses keeps the contract honest and the failure handling type-checked on the consumer side. Because the SWC build skips the swagger CLI plugin, these must be explicit decorators — they are not inferred from the `throw`.

## What Rex flags

- A controller whose use-case throws an HTTP exception (`BadRequestException`, `BadGatewayException`, `NotFoundException`, …) with no corresponding `@Api*Response()` decorator for that status.

## What's NOT a violation

- Generic framework-level 500s for truly unexpected errors — document the *intentional* error contract, not every theoretical throw.
- Errors raised by the global `ValidationPipe` (400) — fine to document once, but the pipe's 400 is framework-standard.

---

_Source: PR #64 comment by @G0maa on 2026-06-10_
_See: https://github.com/marsa-cloud/marsa/pull/64_
