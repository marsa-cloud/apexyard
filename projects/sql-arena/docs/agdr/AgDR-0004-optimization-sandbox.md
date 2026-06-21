# AgDR-0004 — Per-submission optimization sandbox (setup SQL + DB reset isolation)

**Status**: Accepted
**Date**: 2026-06-09
**Author**: Hisham (Tech Lead)

> In the context of a SQL *optimization* contest where the primary technique is adding indexes,
> facing the fact that a read-only shared DB lets nobody `CREATE INDEX`, I decided to let each
> submission run optional "setup" SQL (e.g. `CREATE INDEX`) with full rights and then **reset the
> seed DB to a pristine baseline before the next submission runs**, to achieve per-submission
> isolation on a single shared DB with minimal moving parts, accepting the reset cost between runs
> (mitigated by reloading a prebuilt baseline rather than regenerating).

## Context

- The mentorship task is "write the query, then **optimize** it." Adding an index is the headline
  optimization — without it the leaderboard can't reward the main skill being taught.
- Contestant A's index must not pollute B's run or accumulate. Every contestant must start from the
  identical pristine, index-free baseline.
- Runs are already strictly serial (single worker, [[AgDR-0003-sequential-execution]]) — so a reset
  between runs is always safe; no one ever observes a half-reset DB.
- The score is the *solution query's* execution time with the optimization in place — not the index
  build or reset time.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **Free setup SQL + reset the seed DB to a baseline after each run** | Dead simple to reason about; bulletproof clean slate (incl. planner stats); no whitelist/rollback edge cases; submission can do anything | Reset costs time between runs; slows an index-heavy queue |
| Setup + solution in one always-rolled-back txn | No reseed cost | `CREATE INDEX CONCURRENTLY` & some ops can't run in a txn; needs a statement whitelist; subtler |
| Per-submission DB clone up front | Isolated | Same reset cost, paid eagerly |

## Decision

Chosen: **free-form setup SQL run with full rights, then reset the seed DB to a pristine baseline
before the next job.** The reset is the isolation guarantee — not a whitelist, not least-privilege.

**Mechanics** (per submission, in the worker; DB is pristine at job start):

```
1. Run setup SQL (optional, free-form — typically CREATE INDEX / ANALYZE).
                                                        setup-phase statement_timeout (e.g. 120s)
2. Correctness: run the solution once → compare to golden (values only; ordered-aware).
3. Timing:     EXPLAIN (ANALYZE, FORMAT JSON) solution ×3 → min Execution Time   [AgDR-0002]
                                                        solution-phase statement_timeout = 30s
4. Reset the seed DB to baseline   (only if this job ran setup / could have dirtied it).
```

**Reset mechanism (v1):** the baseline is generated **once on the server** (Step 3 seed) and saved as
a dump; reset = `DROP SCHEMA seed CASCADE` → reload the baseline dump. The naive
drop-and-regenerate-from-scratch is the fallback; reloading the prebuilt dump is the default because
it's much faster than re-running the insert functions.

**Cheap optimization:** the **solution is restricted to a single `SELECT`/`WITH`** and setup is
optional, so a submission with *no setup* cannot have changed anything → **skip its reset**. Only
submissions that ran setup pay the reset cost.

## Consequences

- The tool can finally reward indexing — its core purpose. A setup field appears in the UI above the
  solution field, labelled e.g. "Setup SQL (optional — add indexes here; not timed)".
- **Index build + reset time are excluded from the score**; only the solution `EXPLAIN ANALYZE` time
  counts (= "execution time after optimization").
- Reset cost is the main perf risk. **Deferred speed levers if it's too slow** (mentor's call): use
  a pristine **template database** (`CREATE DATABASE seed TEMPLATE seed_pristine` — block-level copy)
  or `pg_restore -j`, or fall back to the always-rolled-back-txn isolation. Evaluate later.
- Integrity comes from the reset, so the runner role can hold broad rights on `seed`. The `app`
  schema (questions/queue/leaderboard) stays on a separate role, unreachable from submission SQL.
- Residual: a determined contestant could precompute the answer in free-form setup and have the
  solution read it. Accepted for a trusted cohort; an optional setup whitelist (`CREATE INDEX` /
  `ANALYZE` only) is the lever if it ever matters.

## Artifacts

- Timing this protects: [[AgDR-0002-execution-timing]] · Serialization: [[AgDR-0003-sequential-execution]]
- Baseline reset reuses the seed produced in Step 3 of the tech design.
- PRD: `projects/sql-arena/prds/sql-arena.md`
