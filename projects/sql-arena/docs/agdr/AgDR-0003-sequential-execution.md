# AgDR-0003 — Submissions run strictly one at a time

**Status**: Accepted
**Date**: 2026-06-09
**Author**: Hisham (Tech Lead)

> In the context of timing SQL submissions for a leaderboard, facing the fact that concurrent
> queries corrupt each other's measured execution time, I decided to make submission **asynchronous**
> — enqueue each submission in a DB table and process the queue with a single background worker that
> runs one job at a time — to achieve fair, comparable timings without ever blocking the HTTP
> request, accepting that results arrive a moment later (the client polls).

## Context

- The leaderboard ranks on `EXPLAIN ANALYZE` "Execution Time" ([[AgDR-0002-execution-timing]]).
  That number is only meaningful if the query had the box to itself — two contestants running at
  once would slow each other down and produce unfair, irreproducible times.
- The mentor's explicit requirement: *no two contestants run at the same time.*
- **But blocking the HTTP request while it waits its turn (up to ~2 min) is bad UX.** The request
  should return immediately; the run happens in the background.
- Single host, small cohort — heavy concurrency infrastructure is unwarranted.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **DB-backed queue table + single background worker; `/submit` enqueues and returns; client polls** | Serial by construction (one worker = one at a time); non-blocking request; no new infra (queue is a table); naturally gives queue position/status | Client must poll; results arrive a beat later |
| Blocking request + Postgres advisory lock | One round-trip, result inline | Holds the HTTP request up to ~2 min — the UX the mentor explicitly rejected |
| External job queue (Redis/RQ, Celery) | Scales, real queue semantics | Overkill for a weekend single-host tool; new infra |

## Decision

Chosen: **DB-backed queue table processed by a single in-process background worker**, because the
single consumer *is* the serialization guarantee (no lock needed) and the request never blocks.

**Mechanics:**

- `POST /api/submit` validates input, inserts a row into `app.submissions` with `status=queued`, and
  returns `{submission_id, status: queued}` immediately.
- One background worker loop (started on app startup) pops the oldest `queued` row
  (`... ORDER BY created_at FOR UPDATE SKIP LOCKED LIMIT 1`), marks it `running`, executes it
  (read-only txn, correctness compare, EXPLAIN-ANALYZE timing — see runner flow), writes the result
  back to the row (`status=done`), and on `correct` upserts the leaderboard. Then it pops the next.
- Because there is exactly **one** worker, runs are strictly sequential — the fairness invariant —
  with no advisory lock and no blocked requests.
- `GET /api/submission/{id}` returns the current `status` and, when `done`, the result.
- The 30s `statement_timeout` still bounds each run; a stuck job can't wedge the queue forever.

## Consequences

- Timings are fair and reproducible *and* the request returns instantly — both goals met.
- The client polls `/api/submission/{id}` (every ~1s) and can show `queued (position N)` → `running`
  → result.
- `app.submissions` rows are **transient** — purge after the result is delivered (or on a short TTL);
  the durable leaderboard still stores only `(name, exec_ms)`, so "no submission history" holds.
- Throughput is intentionally one query at a time — correct for a fairness tool, not a QPS service.
- A single worker is a single point of processing; if it dies the queue stalls until restart
  (acceptable; jobs persist in the table and resume).

## Artifacts

- Timing metric this protects: [[AgDR-0002-execution-timing]]
- Stack: [[AgDR-0001-stack-selection]]
- PRD: `projects/sql-arena/prds/sql-arena.md`
