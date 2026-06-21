# AgDR-0005 — Seed ~6M rows with generated CSV + Postgres `COPY`

**Status**: Accepted
**Date**: 2026-06-09
**Author**: Hisham (Tech Lead)

> In the context of seeding ~100k products / ~1M customers / ~2–5M order-details against which the
> optimization timings must be meaningful, facing the fact that the existing row-by-row/batched
> INSERT seed (`ecommerce-system`, Kysely + faker) is far too slow at that scale, I decided to
> generate flat **CSV files with faker and bulk-load them via Postgres `COPY`**, to achieve a seed
> that finishes in a sensible time, accepting a two-phase (generate-then-load) flow.

## Context

- The mentor's reference seed (`G0maa/ecommerce-system/sql/seeds/...dummy_data.ts`) builds rows in
  memory and inserts them in 1k batches via Kysely. That's fine for tens of thousands of rows but
  won't finish ~6M rows in reasonable time (per-row/round-trip overhead dominates).
- It is also MySQL-flavoured (`SET FOREIGN_KEY_CHECKS`, `BINARY(16)` UUIDs) — reference only. Our
  schema is Postgres with **integer surrogate keys** (the canonical ERD), which also makes CSV
  generation trivial (sequential IDs) and indexes smaller.
- `COPY` is Postgres's bulk-load path and is one to two orders of magnitude faster than INSERT.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **faker → CSV files → `COPY FROM`** | Fastest practical load; CSVs double as the reset baseline (re-COPY); generation decoupled from load | Two steps; temporary disk for CSVs |
| Keep batched multi-row INSERT (current) | No new code path | Too slow at ~6M rows — the blocker we're solving |
| Pure-SQL `generate_series` + `random()` | No app code at all; very fast | Harder to get faker-quality realistic values; awkward FK fan-out for orders/details |

## Decision

Chosen: **generate CSVs with faker, then bulk-load with `COPY`.**

**Mechanics:**

- A Node/TS generator (reusing `ecommerce-system`'s faker logic) streams rows to
  `category.csv`, `product.csv`, `customer.csv`, `orders.csv`, `order_details.csv`.
- **Integer IDs are assigned sequentially** during generation (category 1..N, product 1..N, …) so FK
  columns are just integers referencing earlier files — no UUID lookup map needed.
- Load order respects FKs: category → product → customer → orders → order_details.
- Load **into tables that have PK only, no secondary indexes** (the index-free baseline is the whole
  point). FK constraints are omitted or added after load — not needed for query correctness and they
  slow COPY.
- Loading uses `pg`'s `COPY ... FROM STDIN` via `pg-copy-streams` (stream the CSV), or `psql \copy`
  in a make target. Run **on the server** (avoids uploading a multi-hundred-MB dump — PRD FR-7).
- The generated CSVs are kept on the server and are the **baseline for the per-run reset**
  ([[AgDR-0004-optimization-sandbox]]): reset = TRUNCATE + drop any contestant indexes + re-`COPY`.

## Consequences

- Seed (and reset reload) complete fast enough to be practical; can be tuned via row-count env knobs
  like the reference script.
- CSVs are an artefact on disk; `.gitignore` them (large + regenerable). Generation is deterministic
  via a fixed faker seed so re-runs reproduce the same data.
- Local-first: generate + load at small volumes locally to validate, then crank the knobs on the
  server (mentor's "test locally first").
- If reset-by-reCOPY proves slow, the template-DB lever from [[AgDR-0004-optimization-sandbox]]
  still applies (clone a pristine DB built once via COPY).

## Artifacts

- Reference seed: `G0maa/ecommerce-system/sql/seeds/1776498199231_dummy_data.ts` (MySQL, batched INSERT)
- Stack: [[AgDR-0001-stack-selection]] · Reset reuse: [[AgDR-0004-optimization-sandbox]]
- PRD: `projects/sql-arena/prds/sql-arena.md`
