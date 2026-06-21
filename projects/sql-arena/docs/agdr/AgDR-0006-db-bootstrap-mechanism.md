# AgDR-0006 — DB bootstrap via idempotent SQL + `db:bootstrap` script (+ compose `initdb.d`)

**Status**: Accepted
**Date**: 2026-06-12
**Author**: Karim (Backend Engineer)

> In the context of standing up SQL Arena's database structure (a `seed` schema for the e-commerce
> dataset under study, an `app` schema for tool state, and two roles wired so `app` is unreachable
> from contestant submission SQL), facing the need for that DDL to be both auto-applied on a fresh
> `docker compose up` **and** re-applied on demand **and** re-executed verbatim by the Step-5
> per-submission reset, I decided to author the structure as **idempotent raw `.sql` files driven by
> a `db:bootstrap` shell wrapper**, auto-mounted into the Postgres `initdb.d`, with **env-driven role
> passwords** and **seed tables owned by `arena_runner`**, to achieve one re-runnable source of truth
> for the schema, accepting that roles/grants live in SQL rather than in the app's migration tool.

## Context

- Step 1 (#2) shipped a typed Kysely connection but **no tables, no migration tooling, no `.sql`**.
  This ticket (#3) creates everything downstream builds on.
- The seed DDL is not write-once: [[AgDR-0004-optimization-sandbox]]'s per-submission reset
  (`DROP SCHEMA seed CASCADE` → reload baseline) **re-executes the seed structure at runtime** in
  Step 5. So the seed schema DDL must be authored as a standalone, re-runnable artefact — not buried
  inside an app migration that only runs once at deploy.
- Two roles are required: `arena_runner` (runs contestant SQL on `seed`; integrity comes from the
  reset, not least-privilege) and `arena_rw` (read/write on `app`). The isolation boundary — `app`
  unreachable from submission SQL — is a **schema-level grant** concern, which reads far more
  naturally in raw SQL than in a TypeScript migration DSL.
- **Postgres ownership constraint (load-bearing):** on Postgres 16/17/**18**, `CREATE INDEX` requires
  *table ownership*. The PG17 `MAINTAIN` privilege covers `REINDEX`/`VACUUM`/`ANALYZE` but **not**
  `CREATE INDEX`. AgDR-0004's headline feature is contestants adding indexes on `seed`, so the seed
  tables must be **owned by** `arena_runner`, not merely granted to it.
- Passwords must not be committed in SQL. The connection user/password for the superuser bootstrap is
  already in compose; the two app-role passwords should follow the same env-driven shape.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **Idempotent `.sql` + `db:bootstrap` wrapper, also mounted into compose `initdb.d`** | One source of truth re-used verbatim by the Step-5 reset; roles/grants read naturally; re-runnable while iterating; auto-applies on fresh volume **and** on demand | DDL lives outside the app's tooling; the model must hand-write idempotency guards |
| Kysely migrations | Versioned; same tool as the app; TS types co-located | Reset can't cheaply re-run a migration; roles/grants/`SET ROLE`/`AUTHORIZATION` are awkward in the DSL; migration runner ≠ the reset path, so the seed DDL would be authored twice |
| `initdb.d` SQL only (no npm script) | Zero extra tooling; auto-runs on empty volume | Only runs **once**, on an empty data volume — no way to re-apply to an existing DB; useless for iterating and for the reset path |

## Decision

Chosen: **idempotent raw SQL + a `db:bootstrap` shell wrapper, with the `db/` dir mounted into the
Postgres container's `/docker-entrypoint-initdb.d`** so a fresh volume self-bootstraps and an existing
DB can be re-bootstrapped with `npm run db:bootstrap`.

Layout (only the top-level `.sh` is auto-run by the Postgres entrypoint; the `.sql` live in a
`sql/` subdir so the entrypoint does **not** run them standalone and double-apply):

```
db/
  00-bootstrap.sh   # initdb.d entry + what `db:bootstrap` calls: reads $ARENA_*_PASSWORD,
                    #   runs `psql -v ... -f sql/bootstrap.sql`
  sql/
    bootstrap.sql   # roles → schemas (AUTHORIZATION) → \ir includes → isolation REVOKEs
    seed.sql        # 5 seed tables, created under SET ROLE arena_runner (→ owned by it)
    app.sql         # questions / submissions / leaderboard, created under SET ROLE arena_rw
```

Supporting decisions:

1. **Env-driven role passwords.** `00-bootstrap.sh` reads `$ARENA_RUNNER_PASSWORD` /
   `$ARENA_RW_PASSWORD` and passes them to `psql -v`. Dev-only defaults live in `.env.example` +
   compose. No password literals in committed SQL. Passwords are set via `ALTER ROLE … PASSWORD :'var'`
   **outside** any `$$…$$` dollar-quote, because psql `:var` interpolation does not reach inside
   dollar-quotes.
2. **Seed tables owned by `arena_runner`** (see Context — the PG18 `CREATE INDEX` ownership rule).
   Achieved by `CREATE SCHEMA seed AUTHORIZATION arena_runner` + `SET ROLE arena_runner` around the
   seed DDL, so the tables are owned by the runner, not the superuser.
3. **Isolation is schema-level + belt-and-suspenders REVOKEs.** Named schemas are private-by-default,
   and we additionally `REVOKE ALL ON SCHEMA app FROM arena_runner, PUBLIC` and
   `REVOKE ALL ON SCHEMA seed FROM arena_rw, PUBLIC` so the golden answers + leaderboard are
   unreachable from submission SQL.

### §4 — Baseline runtime version bump (CEO directive: use latest/LTS)

Folded into this ticket since the bootstrap is being authored fresh and the version choice is part of
the DB baseline:

- **Postgres `16-alpine` → `18-alpine`.** The ownership-for-`CREATE INDEX` analysis above was verified
  against the 18 behaviour. A PG16 `pgdata` volume will not start under 18, so the verification path
  uses `docker compose down -v`.
- **Node `20-alpine` → `22-alpine`** (active LTS). Keeps NestJS 10 — no framework major bump;
  `engines.node` moves `>=20` → `>=22` and `@types/node` stays compatible.

## Consequences

- The seed DDL is a standalone, re-runnable file — directly reusable by the Step-5 reset path, with
  no second authoring.
- Roles, grants, schema ownership, and the isolation boundary live in one readable SQL file that a
  reviewer can audit top-to-bottom.
- `arena_runner` owning the seed tables is what enables the contest's core mechanic (contestant
  `CREATE INDEX`). This is now a documented invariant: anything that recreates `seed` must preserve
  runner ownership.
- DDL lives outside Kysely's tooling, so schema changes touch SQL *and* `src/database/types.ts`
  (the typed schema) — they must be kept in sync by hand. Accepted: the types file is small and the
  divergence is caught by `tsc` the moment a query references a non-existent column.
- Fresh-volume bootstrap and on-demand re-bootstrap share the exact same `bootstrap.sql`, so there is
  one code path to test, not two.

## Artifacts

- Reset path this feeds: [[AgDR-0004-optimization-sandbox]] · Seed load: [[AgDR-0005-seeding-via-copy]]
- Stack: [[AgDR-0001-stack-selection]]
- Tech design (ERD / data model): `projects/sql-arena/architecture/tech-design.md`
- PRD: `projects/sql-arena/prds/sql-arena.md`
- Implements ticket: G0maa/sql-arena#3
