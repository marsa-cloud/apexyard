<!-- Source: ApexYard · templates/prd.md · github.com/me2resh/apexyard · MIT -->

# PRD: SQL Arena — SQL solution-runner + execution-time leaderboard

**Status**: Draft
**Author**: Mariam (Product Manager)
**Created**: 2026-06-08
**Last Updated**: 2026-06-09

---

## Overview

### Problem Statement

A mentorship cohort is practicing SQL optimization: write five analytics queries against a
large e-commerce dataset, then optimize them and **show the before/after execution-time
difference** (the assignment deliverable is literally a table of *naive query → time before →
technique → rewritten query → time after*).

Today each mentee runs their queries on their own machine, against their own seed data, and
self-reports timings. That makes results **non-comparable** (different data volumes, different
hardware) and gives the cohort **no shared signal** about who found the fastest approach or what
technique worked. There's no friendly competition and no single source of truth for "what's the
fastest correct query for Q7?"

**SQL Arena** is a tiny hosted tool that fixes this: everyone runs against *the same* seeded
database on *the same* host, the system verifies the answer is correct, times it, and ranks
correct submissions on a per-question leaderboard.

### Target User

**Primary**: Mentees in the cohort — paste a SQL query for a given question, get back correctness +
execution time, and see where they rank.
**Secondary**: The mentor (you) — defines the questions and reference answers, seeds the data, and
watches the leaderboards to see who's optimizing well and what techniques surface.

### Goals

1. **Apples-to-apples timing** — every submission runs on one host against the seed DB **reset to an
   identical index-free baseline**, so execution times are directly comparable and one contestant's
   indexes never advantage another.
2. **Correctness-gated ranking** — only submissions whose result set matches the question's golden
   answer appear on the leaderboard (no gaming with `SELECT 1`).
3. **Per-question leaderboards** — each of Q5–Q9 has its own ranking by fastest correct execution.
4. **Zero-friction entry** — open URL, pick a question, type a display name, paste SQL, submit. No
   login, no install.
5. **Reproducible setup** — the seed data is produced by a committed, parameterized script so the
   whole arena can be torn down and rebuilt identically.

### Non-Goals (Out of Scope)

- **Authentication / accounts / security hardening.** Trusted cohort, throwaway tool. Display name
  only.
- **Visual polish / design system.** Function over form — a plain, usable page is enough.
- **Tasks 1–4 as user-facing challenges.** The bulk data-generation (Tasks 1–4) is how the *seed
  data* is produced (a setup step via the CSV+COPY seeder), not something users submit through the tool.
- **General SQL playground.** The *solution* is a `SELECT`; *setup* SQL exists only to add indexes
  before the solution runs. The DB is reset to baseline after each run, so it's a per-submission
  optimization sandbox, not a persistent scratchpad.
- **Multi-database engines.** Single engine (Postgres) — the engine under study.
- **Showing the EXPLAIN ANALYZE plan in the tool.** Kept out to stay simple — users run EXPLAIN
  themselves as part of the assignment; the tool only judges + times + ranks.
- **Submission history.** We store only the *best* (name, time) per question — we do not keep a
  record of who submitted what over time.
- **Committing the answers.** Reference queries / golden results never enter source control — a
  mentee could just read them. They live gitignored on the server, in the DB only.
- **Long-term persistence guarantees.** Leaderboard survives normal restarts but this is not a
  durable product.

### Success Metrics

| Metric | Target | How Measured |
|--------|--------|--------------|
| Comparable runs | 100% of submissions run on the same seeded DB/host | By construction (single shared DB) |
| Correctness gate works | 0 incorrect submissions on any leaderboard | Spot-check golden vs. ranked entries |
| Adoption | Every active mentee submits ≥1 correct answer per question | Distinct display names per question leaderboard |
| Optimization signal | Visible spread between slowest-correct and fastest-correct per question | Leaderboard min/max execution time |
| Setup reproducibility | Fresh arena rebuilt from script in < ~15 min | Manual rebuild timing |

---

## User Stories

### US-1: Submit a solution (with optional index setup) and get judged + timed
>
> As a mentee, I want to add indexes and paste my solution query, then see whether it's correct and
> how fast it ran, so that I know if my optimization actually helped.

**Acceptance Criteria**:

- [ ] User selects one of Q5–Q9 and sees the question prompt text.
- [ ] User enters a display name, optionally writes **setup SQL** (e.g. `CREATE INDEX`), and pastes
      a solution `SELECT`.
- [ ] On submit the request returns **immediately** with a ticket; the run happens in the background
      and the page polls for the result (no long blocking wait).
- [ ] The setup SQL runs first, then the solution is timed; **only the solution's execution time is
      scored** (index build time is not counted).
- [ ] When processed, the system returns: correct/incorrect, and the solution's execution time.
- [ ] Correctness = the solution's result set matches the question's golden result set
      (order-insensitive, unless the question is flagged `ordered`).
- [ ] Incorrect submissions are reported as incorrect and are NOT placed on the leaderboard.
- [ ] After the run the seed DB is reset to baseline, so the next contestant starts index-free.

### US-2: See the per-question leaderboard
>
> As a mentee, I want to see the ranked list of fastest correct queries for a question, so that I
> can gauge how my approach compares and stay motivated to optimize.

**Acceptance Criteria**:

- [ ] Each question has its own leaderboard, ranked by execution time ascending (fastest first).
- [ ] Each entry shows: rank, display name, execution time.
- [ ] Only correct submissions appear.
- [ ] One entry per display name — a faster correct submission replaces that name's existing entry.
- [ ] The leaderboard reflects new submissions without a manual rebuild (refresh is fine).

### US-3: Mentor defines a question
>
> As the mentor, I want to register a question with its reference ("golden") query, so that the
> system can auto-derive the correct answer without me hand-writing expected output.

**Acceptance Criteria**:

- [ ] A question is defined by: `title`, `prompt`, `reference_query`, `ordered: true|false`.
- [ ] On setup, the system executes `reference_query` against the seed DB and stores its result set
      as the golden answer **in the database**.
- [ ] Q5–Q9 from the assignment are pre-loaded as the initial question set.
- [ ] **The reference queries / golden answers are never committed to the repo** — they live in a
      gitignored file on the server, and the API never exposes them. (The naive baseline query is
      itself the reference, doubling as the time-to-beat.)

### US-4: Stand the arena up from a prebuilt dataset
>
> As the mentor, I want to load a prebuilt dataset onto the server once, so that I can stand up the
> arena without re-running heavy generation each time.

**Acceptance Criteria**:

- [ ] The dataset is produced by a faker **CSV generator** to known volumes and bulk-loaded with
      Postgres **`COPY`** — run on the server (no large upload). Batched INSERT is too slow at ~6M rows.
- [ ] The generated CSVs are kept on the server and double as the **reset baseline**.
- [ ] Volumes: ~100k products, ~1M customers, ~100 categories, and ~2–5M order-details rows — large
      enough that a missing index is visibly slower under EXPLAIN ANALYZE.
- [ ] A **baseline copy** (dump) is kept on the server so the seed can be reset to pristine, index-
      free state after each submission.

### Edge Cases

| Scenario | Expected Behavior |
|----------|-------------------|
| Solution field contains a non-`SELECT` | Rejected — the scored solution must be a single `SELECT`/`WITH` |
| Setup SQL mutates or corrupts the seed | Allowed to run, but the seed is reset to baseline after the run — next contestant is unaffected |
| Query never returns / runs too long | Killed by a per-query statement timeout (30s); job marked "timed out", not ranked; queue moves on |
| Many submit at once | Requests all return instantly; jobs queue and run one at a time; page shows queue position |
| Query errors (syntax, bad column) | Error message shown to user; not ranked |
| Two correct submissions with identical time | Both shown; stable order |
| Result correct but row order differs on an `ordered` question | Marked incorrect (ordering is part of that question) |
| Result values correct but with extra/renamed columns | Compared by values only — extra columns make it not match; aliases are ignored |
| Same name resubmits a faster query | Replaces that name's existing entry; slower resubmits are ignored |
| Concurrent submissions | Runs are **serialized** — only one query executes at a time; others wait in line, so no submission skews another's timing |

---

## Requirements

### Functional Requirements

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| FR-1 | Run a user-submitted solution `SELECT` against the seed DB | Must | Statement timeout enforced |
| FR-2 | Measure server-side execution time for the solution | Must | Basis for ranking; setup/index build excluded |
| FR-3 | Compare result set to golden answer **by values only**; order-insensitive unless question is `ordered` | Must | The correctness gate. Extra columns ⇒ no match; aliases ignored |
| FR-4 | Solution must be a single `SELECT`/`WITH` (guarded) | Must | Keeps the ranked artefact a query |
| FR-4a | Accept optional **setup SQL** (e.g. `CREATE INDEX`) run before the solution; reset the seed to baseline after the run | Must | The optimization sandbox (AgDR-0004); index build not timed |
| FR-4b | Async submission: enqueue and return immediately; one background worker runs jobs one at a time; client polls for the result | Must | Non-blocking request + serial execution = fair timing (AgDR-0003) |
| FR-5 | Persist the best correct submission per name and render per-question leaderboards | Must | Stores (name, time) only — no history |
| FR-6 | Question registry with `title`/`prompt`/`reference_query`/`ordered`; golden derived from reference | Must | Q5–Q9 preloaded |
| FR-7 | Seed the DB on the server via faker CSV generator + `COPY`; keep CSVs as the reset baseline | Must | Bulk-load, not batched INSERT (AgDR-0005); no large-dump upload |
| FR-8 | Single page: pick question → enter name → paste SQL → submit → see result + leaderboard | Must | No login |
| FR-9 | Keep only the best (fastest correct) entry per display name per question | Must | One row per name on the board |
| FR-10 | Combined / overall leaderboard across questions | Could | Nice-to-have view |

### Non-Functional Requirements

| Category | Requirement | Target |
|----------|-------------|--------|
| Performance | Per-query statement timeout to bound runaway queries | 30s hard cap |
| Isolation | User queries cannot mutate or read outside the seed DB | DB user is read-only on the seed schema |
| Fairness | Query runs are serialized — one submission executes at a time | Single background worker draining a queue; no concurrent execution skews timing |
| Responsiveness | Submitting never blocks on the query run | Async enqueue + poll; request returns in ms |
| Reliability | Leaderboard survives process/container restart | Backed by persistent store |
| Security | Explicitly minimal — trusted cohort, no auth | Read-only DB role is the real guardrail |
| Operability | Stand up / tear down from committed scripts | Single-host deploy |

---

## Design

### User Flow

```
[Open URL]
    |
    v
[Pick question Q5-Q9]  --> shows prompt text
    |
    v
[Enter display name + paste SELECT]
    |
    v
[Submit] --> server runs query (read-only, timed, timeout-bounded)
    |
    +--> [Correct]   --> show exec time --> update best-per-name on leaderboard --> show board
    |
    +--> [Incorrect] --> show "result didn't match" --> NOT ranked
    |
    +--> [Error/Timeout] --> show message --> NOT ranked
```

### Wireframes / Mockups

None — intentionally plain (see Non-Goals). A single page with a question dropdown, name field,
SQL textarea, a result panel, and a leaderboard table.

Source assignment (defines Q5–Q9 and the before/after deliverable):
`/home/gomaa/Pictures/Screenshot_20260608_182806.png`

---

## Technical Notes

### Dependencies

| Dependency | Type | Status | Owner |
|------------|------|--------|-------|
| Postgres (engine under study) | External | Ready | Mentor |
| Faker CSV generator + COPY loader (ports `ecommerce-system` seed logic) | Internal | To produce | Mentor |
| Single host / VPS to deploy on | Internal | TBD | Mentor |

### Technical Constraints

- **The whole point is timing spread**, so the seed volume must be large enough for index/plan
  differences to show in EXPLAIN ANALYZE. Target: ~100k products, ~1M customers, ~100 categories,
  ~2–5M order-details rows — large enough to matter, easily seedable on one host.
- **Seeding uses CSV + `COPY`, generated on the server.** Row-by-row/batched INSERT (the existing
  `ecommerce-system` seed) is too slow at ~6M rows; faker writes CSVs and Postgres `COPY` bulk-loads
  them (AgDR-0005). Generated on the server, so no large dump upload. CSVs double as the reset
  baseline.
- Execution times are taken from `EXPLAIN ANALYZE` and runs are serialized (one at a time), so the
  ranking is fair and reproducible.
- Per-submission integrity comes from resetting the seed after each run (not a read-only role).
- **Stack: NestJS (Node/TS) + Kysely + `pg`**, Postgres, static page, Docker Compose — chosen to
  match the mentor's existing `ecommerce-system` repo (AgDR-0001).

---

## Launch Plan

### Rollout Strategy

- [x] All users at once (small trusted cohort, single shared instance)
- [ ] Phased rollout
- [ ] Beta program first

---

## Open Questions

| Question | Owner | Status | Resolution |
|----------|-------|--------|------------|
| Exact seed volumes? | Mentor | **Resolved** | ~100k products, ~1M customers, ~100 categories, ~2–5M order-details |
| How to seed ~6M rows fast? | Tech Lead | **Resolved** | Faker → CSV → `COPY`, generated on the server (AgDR-0005); batched INSERT too slow |
| Correctness strictness? | Mentor | **Resolved** | Compare by **values only**; aliases ignored, extra columns ⇒ no match |
| Best-per-name vs. full history? | Mentor | **Resolved** | Best-per-name only; store (name, time), no per-submission history |
| Statement timeout value? | Mentor | **Resolved** | 30s |
| Show EXPLAIN ANALYZE in the tool? | Mentor | **Resolved** | No — out of scope; keep it simple |
| Stack choice? | Mentor | **Resolved** | NestJS (Node/TS) + Kysely + `pg` — matches `ecommerce-system` (AgDR-0001) |
| Integer keys vs. reference UUIDs? | Tech Lead → Mentor | Open | Using integer `*_id` per canonical ERD; confirm or switch to mirror the UUID schema |

---

## Timeline

| Milestone | Target Date | Status |
|-----------|-------------|--------|
| PRD Approved | 2026-06-08 | Draft |
| Tech Design (stack + AgDR) | TBD | |
| Seed + runner working | TBD | |
| Leaderboard + page | TBD | |
| Launch to cohort | TBD | |

---

## Approvals

| Role | Name | Date | Status |
|------|------|------|--------|
| Product Manager | Mariam | 2026-06-08 | Author |
| Head of Product | | | Pending |
| Tech Lead | | | Pending |
| Head of Design | | | N/A (no UI design) |
