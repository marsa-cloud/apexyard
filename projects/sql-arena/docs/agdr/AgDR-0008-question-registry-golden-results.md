# AgDR-0008 — Question Registry & Golden Results Loader

> In the context of bootstrapping the grading oracle for Q5–Q9, facing the need
> to keep reference queries secret while making golden results queryable, I decided
> to split committed metadata from gitignored reference SQL and join them by code at
> load time, to achieve an idempotent loader that the runner reuses, accepting that
> the secrets file must be provisioned separately on every environment.

## Context

Before any submission can be graded, two things must exist in `app.questions`:
- The **question metadata** (code, title, prompt, ordered flag) — safe to commit.
- The **golden result** (normalised reference output) and the **reference query** that
  produced it — these are the answer and must never be readable by mentees.

A mentee with `SELECT` access to `app.questions` must not be able to retrieve the
reference query.  The isolation boundary (`REVOKE` on `app` from `arena_runner` /
`PUBLIC`) already handles the DB-level guard.  The question here is how the *loader*
gets the reference queries into the DB in the first place.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **Committed registry + gitignored secrets file joined at load time** | Clear separation; reference SQL never in git history; loader is a single script; Step-5 runner imports the same `normalise.ts` | Secrets file must be provisioned manually on every environment (dev + prod) |
| Encrypt reference queries in the repo | No out-of-band provisioning step | Key management complexity; git history retains ciphertext; overkill for a learning tool |
| Store reference queries only in the DB (no file) | Nothing to provision | Chicken-and-egg: how does the first row get in? Still need a bootstrap file |
| Hardcode queries in `load-questions.ts` | Simplest | Queries are committed; any `git log` viewer sees the answer |

## Decision

Chosen: **committed registry + gitignored secrets file joined at load time**, because:

- The committed file (`src/seed/questions.ts`) contains only metadata the mentee
  can legitimately see (code, title, prompt, ordered flag).
- The gitignored file (`secrets/reference_queries.sql`) is provisioned once
  server-side.  An `.example` template with explicit parser-contract documentation
  ships in the repo so setup is straightforward.
- Section format `-- Q<n>:` (matching the example already committed in Step 3) is
  parsed by a simple split — no YAML/JSON overhead, easy for a human to author.
- The `normalise.ts` module is pure and importable by both the loader (this step)
  and the runner (Step 5), making the golden result and the graded result byte-identical
  for the same input.  This is load-bearing: any divergence between how the golden was
  computed and how a submission is scored is a silent false-negative.

## Consequences

- `secrets/reference_queries.sql` must be provisioned on every new environment
  before `npm run db:questions` can succeed.  The loader fails loudly with a
  "copy the .example" message if the file is missing.
- `src/grading/normalise.ts` is the canonical source of truth for the correctness
  rule. If the rule changes (e.g. column-order sensitivity), both the loader and
  the runner are updated in one place.
- The upsert (`ON CONFLICT (code) DO UPDATE`) makes the loader idempotent —
  re-running after a schema wipe or a reference query update is safe.

## Artifacts

- `src/seed/questions.ts` — committed question metadata registry (Q5–Q9)
- `src/grading/normalise.ts` — shared normalisation module
- `src/seed/load-questions.ts` — idempotent loader
- `secrets/reference_queries.sql.example` — parser-contract-documented template
- `package.json` `db:questions` script

## Related

- AgDR-0002 — normalisation rule (values-only, order-insensitive unless `ordered`)
- AgDR-0007 — seed loader mechanism (pattern this loader mirrors)
