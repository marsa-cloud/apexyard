# AgDR-0007 — Seed loader = server-side `COPY FROM file`; reset stays re-COPY

**Status**: Accepted
**Date**: 2026-06-12
**Author**: Karim (Backend Engineer)

> In the context of loading the faker-generated CSVs into the `seed` schema (and reusing that load
> path for the Step-5 per-submission reset), facing a choice of COPY delivery mechanism that
> AgDR-0005 left open, I decided to use **server-side `COPY … FROM '/seed-data/x.csv' CSV HEADER`**
> issued as plain SQL through the existing `pg` client, with the CSV directory mounted into the
> Postgres container, to achieve the fastest load with no new runtime dependency and a reset path
> that reuses the identical SQL, accepting that the CSVs must live on the DB host (true by design).

## Context

- [[AgDR-0005-seeding-via-copy]] settled "faker → CSV → `COPY`" but named the delivery mechanism only
  by example (`pg-copy-streams` or `psql \copy`) — it did not pick one.
- Everything runs on **one host** ([[AgDR-0001-stack-selection]] single process,
  [[AgDR-0003-sequential-execution]] single worker), and AgDR-0005 already mandates loading **on the
  server**. So "the CSV is on the DB host" is a safe, designed-in assumption, which unlocks
  server-side `COPY FROM file` — the variant I'd initially overlooked.
- The load path is reused at runtime by the Step-5 reset ([[AgDR-0004-optimization-sandbox]]):
  TRUNCATE + drop contestant indexes + re-COPY. Whatever we pick here, Step 5 inherits.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **Server-side `COPY FROM '/seed-data/x.csv'`** (plain SQL via existing `pg` client) | Fastest (no client-side row framing); **no new runtime dep**; Step-5 reset reuses the *exact same SQL*; trivially callable from app code | CSV must be readable by the postgres process (host-local + a mount); needs `pg_read_server_files` or superuser |
| `pg-copy-streams` (Node streams CSV over the wire) | Works even when CSV isn't on the DB host | Adds a dependency; marginally slower (streams over the connection); more code |
| `psql \copy` (shell/make target) | Fewest Node deps | Needs a `psql` binary present; **not** callable from the Step-5 runtime reset — would be re-implemented in Node later |

## Decision

Chosen: **server-side `COPY … FROM file`** via the existing `pg` client. The generated CSV dir is
mounted into the `postgres` service at `/seed-data` (read-only); `loadSeed(pool)` runs `TRUNCATE`
(reverse FK order) then one `COPY seed.<t> FROM '/seed-data/<t>.csv' WITH (FORMAT csv, HEADER)` per
table in FK order. `loadSeed` is exported so Step 5 imports it directly.

**Reset baseline stays re-COPY** (per AgDR-0004): the CSVs are the baseline; reset = TRUNCATE + drop
contestant indexes + re-COPY. `pg_dump`/`pg_restore -j` and a pristine **template database**
(`CREATE DATABASE … TEMPLATE`) remain the documented **Step-5 speed levers**, to be adopted only if
re-COPY is measured too slow. They are *not* alternatives to generation — they operate on
already-loaded data — so they don't belong in this ticket.

## Consequences

- Zero new runtime dependency for loading; generation adds dev-deps `@faker-js/faker` + `tsx` only.
- The Step-5 reset is "issue the same `loadSeed` SQL again," not a second implementation.
- Loader role needs `pg_read_server_files` (dev `arena` superuser already qualifies; production grants
  it to the loader role).
- Off-host loading is unsupported by design — acceptable given the single-host architecture. If that
  ever changes, `pg-copy-streams` is the drop-in fallback.

## Artifacts

- Generation decision: [[AgDR-0005-seeding-via-copy]] · Reset reuse: [[AgDR-0004-optimization-sandbox]]
- Schema this fills: [[AgDR-0006-db-bootstrap-mechanism]] · Stack: [[AgDR-0001-stack-selection]]
- Implements ticket: G0maa/sql-arena#4
