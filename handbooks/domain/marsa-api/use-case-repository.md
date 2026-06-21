---
paths:
  - "apps/api/src/app/**/use-cases/**"
---

ENFORCEMENT: advisory

# Handbook: Use-case-scoped repositories (marsa apps/api)

**Scope:** PRs touching `apps/api/src/app/<feature>/use-cases/**` in the marsa NestJS backend.
**Enforcement:** advisory

## The rule

A use-case that touches the database does so through **its own** repository —
`<use-case>.repository.ts`, class `<Action>Repository`, co-located in the slice —
not by injecting `EntityManager` and inlining queries/persistence in the use-case.

```ts
// convert-manifest.repository.ts
@Injectable()
export class ConvertManifestRepository {
  constructor(private readonly em: EntityManager) {}

  async upsertByGithubAppId(app: GitHubApp): Promise<void> {
    const em = this.em.fork() // request-isolated unit of work
    // …find-or-insert, race recovery, etc. live here
  }
}

// convert-manifest.use-case.ts — depends on the repository, not the EM
constructor(private readonly repository: ConvertManifestRepository) {}
```

The repository is **use-case-scoped**, mirroring the vertical-slice layout where
each use-case owns its controller / command / response. It is **not** a
feature-wide aggregate repository. The repository wraps `this.em.fork()` internally
to keep request isolation (the unit of work), so the use-case never sees a raw EM.

AgDR: `docs/agdr/AgDR-0011-github-app-repository.md`.

## Why

- **Mockability** — the use-case unit test stubs one method (`upsertByGithubAppId`)
  instead of hand-rolling an `EntityManager` whose `.fork()` returns a sub-object
  with `findOne/persistAndFlush/assign/flush/clear/findOneOrFail`.
- **Layering** — persistence orchestration (idempotency, race recovery, forking)
  lives behind a persistence seam, not in the application layer.
- **Why not MikroORM `@InjectRepository`?** Its custom repositories bind to the
  **root** EM (shared identity-map / UoW across requests). We rely on `em.fork()`
  for request isolation, so a plain injectable repository that forks internally is
  both simpler and a better fit — see AgDR-0011.

## What Rex flags

- A use-case (`*.use-case.ts`) injecting `EntityManager` and calling `findOne` /
  `persist*` / `flush` / `nativeDelete` directly, where a `<use-case>.repository.ts`
  should hold that logic.

## What's NOT a violation

- A repository injecting `EntityManager` — that's the one place the EM belongs.
- A read-only support service under `src/modules/**` that legitimately wraps the EM.
- Genuinely trivial single-line reads where a repository adds no value — but prefer
  the repository once there's any insert/update/transaction logic.
- Sharing one repository across use-cases **after** real duplication appears
  (promote to a feature-level module then; don't pre-share).

---

_Source: PR #64 comments by @G0maa on 2026-06-10_
_See: https://github.com/marsa-cloud/marsa/pull/64_
