---
paths:
  - "apps/api/src/**"
---

ENFORCEMENT: blocking

# Handbook: Test the right layer — use-cases unit, endpoints e2e, not repositories (marsa apps/api)

**Scope:** PRs touching `apps/api/src/**` tests in the marsa NestJS backend.
**Enforcement:** blocking

## The rule

- **Use-cases** get **unit tests** (`<use-case>.use-case.unit.test.ts`) — collaborators stubbed.
- **Endpoints** get **e2e tests** (`<use-case>.e2e.test.ts`) — full app, real DB.
- **Repositories** do **not** get dedicated unit tests. Their behaviour is exercised through the use-case unit tests and the e2e tests. Don't add `*.repository.unit.test.ts` until explicitly asked to.

## Why

A repository is a thin wrapper over `em.fork()` + a query or two — a dedicated unit test mostly re-asserts the ORM, is brittle to mock, and duplicates coverage the use-case unit tests and e2e tests already provide. Concentrating effort on the use-case (logic) and the endpoint (contract) layers keeps the suite meaningful and cheap; over-testing the persistence wrapper is wasted ceremony.

## What Rex flags

- A new `*.repository.unit.test.ts` file under `apps/api/src/**` (the repository layer being unit-tested directly).
- A use-case left without a `*.use-case.unit.test.ts`, or an endpoint without a `*.e2e.test.ts`.

## Sample finding

> blocking: `capture-installation.repository.unit.test.ts` unit-tests the repository
> directly — the repo is a thin `em.fork()` wrapper already covered by the use-case
> unit tests and the e2e tests. Drop it unless repository tests were explicitly
> requested. See `handbooks/domain/marsa-api/test-layer-boundaries.md`.

## What's NOT a violation

- Repository tests that were **explicitly requested** (the rule is "not until told to").
- A repository with genuinely non-trivial query logic the operator has asked to cover — call it out and get sign-off.
- Integration tests that exercise a repository **through** a use-case or endpoint.

---

_Source: PR #70 comment by @G0maa on 2026-06-11_
_See: https://github.com/marsa-cloud/marsa/pull/70_
