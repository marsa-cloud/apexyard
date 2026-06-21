---
paths:
  - "apps/api/src/app/**/use-cases/**"
---

ENFORCEMENT: blocking

# Handbook: Use-case folder naming (marsa apps/api)

**Scope:** PRs touching `apps/api/src/app/<feature>/use-cases/**` in the marsa NestJS backend.
**Enforcement:** blocking

## The rule

Inside a `use-cases/<use-case>/` folder, names follow the folder, not HTTP transport:

- The application-logic class is `<Action>UseCase` (file `<use-case>.use-case.ts`) — **not** `<Action>Service`.
- The HTTP controller stays `<use-case>.controller.ts` exporting `<Action>Controller`, and injects the use-case as `private readonly usecase: <Action>UseCase`.
- The input DTO is `<use-case>.command.ts` exporting `<Action>Command` — **not** `<use-case>.request.ts` / `<Action>Request`.
- The output DTO stays `<use-case>.response.ts` exporting `<Action>Response`.

Example: in `use-cases/convert-manifest/` the class is `ConvertManifestUseCase`, the input is `ConvertManifestCommand`.

## Why

The folder is literally named `use-cases/`. Calling the class `…Service` contradicts the folder and blurs the line between a *use-case* (one application operation) and a shared *service* (cross-cutting infrastructure under `src/modules/`). "Request" framing leaks HTTP-transport vocabulary into the application layer — the input is a command the use-case executes, so the name should describe intent, not transport. Predictable names let humans and the LLM derive a file/class name without reading every file, which is the failure mode that produced this PR's drift.

## What Rex flags

- A class under `apps/api/src/app/**/use-cases/**` whose name ends in `Service` (e.g. `export class ConvertManifestService`). Suggest `…UseCase`.
- A controller in a use-case folder injecting a dependency typed `…Service` or named `service`. Suggest `usecase: …UseCase`.
- A file named `*.request.ts` under a `use-cases/` folder, or a class named `…Request`. Suggest `*.command.ts` / `…Command`.

## Sample finding

> blocking: This class lives in a `use-cases/` folder, so it should be
> `ConvertManifestUseCase` (in `convert-manifest.use-case.ts`), not
> `ConvertManifestService`. The controller should inject it as
> `private readonly usecase: ConvertManifestUseCase`. See
> `handbooks/domain/marsa-api/use-case-naming.md`.

## What's NOT a violation

- Classes named `…Service` under `src/modules/` — those are correctly shared/support services, not use-cases.
- The `…Controller` suffix on controllers, and the `…Response` suffix on output DTOs — only the application class (`…UseCase`) and the input DTO (`…Command`) are renamed.
- A use-case that injects a real shared `…Service` from `src/modules/` — depending on a support service is fine.

---

_Source: PR #64 comment by @G0maa on 2026-06-08_
_See: https://github.com/marsa-cloud/marsa/pull/64_
