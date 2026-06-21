# AgDR-0001 — SQL Arena stack & architecture

**Status**: Accepted (revised 2026-06-09 — mentor chose TS/Node/NestJS to align with the existing `ecommerce-system` repo; supersedes the earlier Python/FastAPI pick)
**Date**: 2026-06-09
**Author**: Hisham (Tech Lead)

> In the context of a weekend mentorship tool that runs user-submitted SQL against a large
> Postgres dataset, times it, and ranks it, facing a stack-choice and a wish to reuse the mentor's
> existing TypeScript e-commerce code, I decided to build it as a NestJS (Node/TS) service with
> Kysely over a single PostgreSQL instance with role-separated schemas and a static page, to achieve
> a familiar stack the mentor already runs (`ecommerce-system` is TS + Kysely), accepting NestJS's
> modest ceremony for what is a small tool.

## Context

- **Postgres is non-negotiable** — it is the engine the cohort is studying and optimizing. The
  whole tool exists to surface its EXPLAIN ANALYZE timing differences.
- The backend's real work is narrow: run setup + a solution query under a timeout, compare the
  result to a stored golden answer, parse an execution time, upsert a leaderboard row, serve a page.
- **The mentor already has a TypeScript e-commerce system** (`G0maa/ecommerce-system`: TS + Kysely +
  faker) whose schema and seed logic this tool mirrors. Reusing that stack (TS + Kysely) lets the
  schema types and seed-generation code carry over rather than being re-implemented in another
  language. This reverses the earlier "Python is the faster path" call — mentor preference + code
  reuse win.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **NestJS (Node 20 + TypeScript) + Kysely + `pg`** | Same stack/idiom as the mentor's `ecommerce-system` (TS + Kysely); schema types + faker seed logic carry over; Kysely runs raw `EXPLAIN (ANALYZE, FORMAT JSON)` fine | NestJS adds some boilerplate for a small tool |
| Python 3.12 + FastAPI + psycopg3 | Very terse SQL plumbing | New language for the mentor; no reuse of the existing TS code |
| Go + pgx | Single static binary | Slowest to author; no reuse |

## Decision

Chosen: **NestJS (Node/TS) + Kysely + `pg`**, **single PostgreSQL instance**, **static HTML+vanilla-JS page**, **Docker Compose** deploy — because it matches the mentor's existing `ecommerce-system` stack, lets the schema/seed code carry over, and keeps everything in one language the mentor maintains.

**Architecture shape:**

- **One Postgres instance, two schemas, two roles:**
  - `seed` schema — the e-commerce data (products, customers, categories, orders, order_details).
  - `app` schema — `questions` + `leaderboard` tables (tool's own state).
  - Role `arena_runner` — broad rights on `seed`; runs each submission's optional setup SQL (e.g.
    `CREATE INDEX`) and the solution, after which the worker **resets `seed` to a pristine baseline**
    before the next job. Integrity comes from the reset, not least-privilege — see
    [[AgDR-0004-optimization-sandbox]]. (Earlier draft used a `SELECT`-only `arena_ro` role;
    superseded once index optimization became a requirement.)
  - Role `arena_rw` — read/write on `app` only; used by the service for questions, the submissions
    queue, and the leaderboard.
- **Service** (NestJS): serves the static page + a tiny JSON API (`/api/questions`, `/api/submit`,
  `/api/submission/:id`, `/api/leaderboard/:code`). The background submission worker is a Nest
  provider started on bootstrap (a single in-process loop — see the serialization note below).
- **Data access**: Kysely over `pg`, carrying the schema types from `ecommerce-system`.
- **Frontend**: one `index.html` + vanilla JS `fetch` — no build step.
- **Deploy**: Docker Compose (`postgres` + `app`) on the existing VPS.

> **Single process matters**: the serial-execution guarantee ([[AgDR-0003-sequential-execution]])
> needs exactly one consumer of the queue, so the NestJS app runs as a **single instance / no
> clustering** (one Node process). Don't add PM2 cluster mode or multiple replicas.

## Consequences

- The tool stays in the mentor's existing language (TypeScript) and reuses `ecommerce-system`'s
  schema types + faker seed logic — no second language to maintain.
- Role separation keeps the `app` state (questions/queue/leaderboard) unreachable from submission
  SQL. Seed-data integrity is guaranteed by **resetting the seed to baseline after each run** rather
  than by a read-only role (see [[AgDR-0004-optimization-sandbox]]).
- Single Postgres instance keeps ops trivial (one container to back up / restore).
- Seeding ~6M rows is too slow via row-by-row/batched INSERT — a separate decision picks CSV + COPY:
  see [[AgDR-0005-seeding-via-copy]].
- Timing-measurement method is a separate decision — see [[AgDR-0002-execution-timing]].

## Artifacts

- PRD: `projects/sql-arena/prds/sql-arena.md`
- Repo: https://github.com/G0maa/sql-arena
