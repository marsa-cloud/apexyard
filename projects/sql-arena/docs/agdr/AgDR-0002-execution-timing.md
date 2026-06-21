# AgDR-0002 — How SQL Arena measures the ranked execution time

**Status**: Accepted (mentor approved 2026-06-09)
**Date**: 2026-06-09
**Author**: Hisham (Tech Lead)

> In the context of ranking correct SQL submissions by speed, facing the need for a *fair* and
> *assignment-aligned* timing number, I decided to rank on PostgreSQL's own `EXPLAIN (ANALYZE)`
> "Execution Time", taking the minimum of 3 runs, to achieve a metric that excludes
> network/fetch noise and matches the technique the cohort is already using, accepting one extra
> in-DB execution per submission.

## Context

- The leaderboard's entire purpose is to compare how fast a query runs — a noisy or gameable metric
  defeats it.
- The assignment itself tells mentees to *"use the EXPLAIN analyze plan"* to measure before/after.
  Ranking on the same number the assignment uses keeps the tool honest and pedagogically aligned.
- Correctness still requires actually running the query to fetch its result set (see the runner
  flow in the tech design).

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **`EXPLAIN (ANALYZE, FORMAT JSON)` → "Execution Time", min of 3 runs** | Postgres-measured; excludes client fetch + network; matches the assignment's own method; min-of-3 damps cold-cache/jitter | Executes the query an extra few times; warm-cache effects still possible |
| Wall-clock around the client execute+fetch | Dead simple | Includes network + client-side fetch overhead; punishes large result sets unfairly; noisier |
| `pg_stat_statements` total/mean time | Aggregated, low overhead | Needs an extension + per-submission isolation gymnastics; awkward to attribute one submission |

## Decision

Chosen: **rank on `EXPLAIN (ANALYZE, FORMAT JSON, TIMING ON)` "Execution Time", taking the minimum across 3 runs**, because it is the fairest readily-available number and is exactly what the mentorship assignment asks mentees to read.

**Flow per submission** (run by the worker on the seed schema via `arena_runner`, after any setup SQL,
with `statement_timeout = 30s` on the solution):

1. Run the submitted SQL once normally → fetch rows → compare to the golden result (values only,
   order-insensitive unless the question is `ordered`).
2. If incorrect/error/timeout → report, do **not** rank.
3. If correct → run `EXPLAIN (ANALYZE, FORMAT JSON) <sql>` three times, parse "Execution Time",
   keep the **minimum** → that's the ranked `exec_ms`.
4. After the run, the worker resets the seed DB to baseline (see
   [[AgDR-0004-optimization-sandbox]]) so the next submission starts index-free.

## Consequences

- **Only the solution query is timed** — the contestant's setup indexes
  ([[AgDR-0004-optimization-sandbox]]) are built *before* the timed runs, so build cost is excluded
  from the score (= "execution time after optimization").
- ~4 in-DB executions per correct submission (1 correctness + 3 timing). Fine at cohort scale; the
  30s statement timeout bounds worst case.
- The leaderboard number is directly reproducible by a mentee running EXPLAIN ANALYZE themselves —
  builds trust in the ranking.
- Warm-cache advantage for later runs in the batch is accepted as negligible for a learning tool;
  min-of-3 is the pragmatic fairness knob, not a benchmark-grade harness.

## Dependency

This metric is only fair if **no other submission runs concurrently** — otherwise queries contend
for CPU/IO/buffer cache and skew each other's "Execution Time". Serialized execution is therefore a
prerequisite, recorded in [[AgDR-0003-sequential-execution]].

## Artifacts

- Parent decision: [[AgDR-0001-stack-selection]]
- PRD: `projects/sql-arena/prds/sql-arena.md`
