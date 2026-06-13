---
paths:
  - "apps/api/src/**"
---

ENFORCEMENT: blocking

# Handbook: Stub collaborators with sinon `createStubInstance` in unit tests (marsa apps/api)

**Scope:** PRs touching `apps/api/src/**` unit tests in the marsa NestJS backend.
**Enforcement:** blocking

## The rule

In use-case unit tests, build collaborator doubles with sinon's `createStubInstance(Class)` — not hand-rolled object literals cast through `as unknown as Class`.

```ts
import { createStubInstance } from 'sinon'

const repository = createStubInstance(CaptureInstallationRepository)
repository.loadProvisionedApp.resolves(app)
repository.upsertByInstallationId.resolves(installation)

const github = createStubInstance(OctokitGithubClient)
github.getInstallationToken.resolves('ghs_ok')
```

## Why

`createStubInstance` produces a real instance of the class with every method auto-stubbed, so the double stays in sync with the class signature — add a method and the stub has it; rename one and the test fails to compile. A hand-rolled `{ foo: () => ... } as unknown as Repository` silently drifts from the real type, defeats the type-checker via the `as unknown` cast, and re-implements call-tracking that sinon gives for free (`.calledOnceWithExactly`, `.firstCall.args`).

## What Rex flags

- A unit test constructing a collaborator double as an object literal cast with `as unknown as <Class>` (or `as <Class>`).
- Manual call-tracking arrays (`const calls: ...[] = []`) re-implementing what a sinon stub records.

## Sample finding

> blocking: This builds the repository double as `{ ... } as unknown as
> CaptureInstallationRepository`, which drifts from the real type and hand-rolls
> call tracking. Use `createStubInstance(CaptureInstallationRepository)` and
> `.resolves(...)`. See `handbooks/domain/marsa-api/sinon-stub-instance.md`.

## What's NOT a violation

- e2e tests, which use the real wired dependencies (and the `MockGithubClient` DI binding), not stubs.
- A plain value/DTO built via its builder (`new XBuilder().build()`) — that's not a collaborator double.
- A standalone fake whose behaviour is the thing under test.

---

_Source: PR #70 comments by @G0maa on 2026-06-11_
_See: https://github.com/marsa-cloud/marsa/pull/70_
