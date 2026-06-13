---
paths:
  - "apps/api/src/modules/**"
---

ENFORCEMENT: blocking

# Handbook: One client seam per external service, bound via a NestJS factory (marsa apps/api)

**Scope:** PRs touching `apps/api/src/modules/**` that add access to an external service in the marsa NestJS backend.
**Enforcement:** blocking

## The rule

Don't spin up a new service class per external-facing feature. Define **one** abstract class (the seam) for the external service, add capabilities as methods on it, and bind it to a concrete implementation via a **NestJS factory** in the module — real impl in production, a reusable mock in test/local.

```ts
// github-client.ts — the seam
export abstract class GithubClient {
  abstract convertManifest(code: string): Promise<GitHubAppCredentials>
  abstract getInstallationToken(params: InstallationTokenParams): Promise<string>
}

// github-client.module.ts — factory binding
@Module({
  providers: [
    { provide: GithubClient, useClass: useMockGithubClient() ? MockGithubClient : OctokitGithubClient },
  ],
  exports: [GithubClient],
})
export class GitHubClientModule {}

// mock-github-client.ts — reusable, network-free, override-able
export function createMockGithubClient(overrides: Partial<GithubClient> = {}): GithubClient {
  return Object.assign(new MockGithubClient(), overrides)
}
```

## Why

A separate `XTokenService` / `YManifestClient` per feature scatters one external dependency (GitHub) across many providers, each needing its own mock wiring. A single abstract seam means: consumers inject one thing, new capabilities are methods (not new providers), and the whole external service is faked in one place — `MockGithubClient` bound under `NODE_ENV=test` so e2e/local never hit the network. The abstract class (vs an `interface`) is what NestJS DI can use as an injection token. This is the AgDR-0014 consolidation.

## What Rex flags

- A new `*Service` / `*Client` under `apps/api/src/modules/**` wrapping the **same** external service that an existing seam already covers (should be a method on the seam instead).
- An external client injected as a concrete class rather than through an abstract seam + module factory binding.
- A new external integration with no network-free mock implementation for test/local.

## Sample finding

> blocking: `GitHubInstallationTokenService` is a second per-feature class for the
> same external service (GitHub). Fold `getInstallationToken` into the single
> `GithubClient` seam and bind impls via the module factory (real vs
> `MockGithubClient`). See `handbooks/domain/marsa-api/external-client-seam.md`.

## What's NOT a violation

- The first seam for a genuinely new external service (a different vendor).
- Internal (non-external) support services — this rule is about outbound third-party API access.
- The concrete implementations themselves (`OctokitGithubClient`, `MockGithubClient`) — those are what the seam binds to.

---

_Source: PR #70 comments by @G0maa on 2026-06-11_
_See: https://github.com/marsa-cloud/marsa/pull/70_
