# sql-arena #8 — Step 4: Question registry + golden results

## Context

sql-arena is a SQL mentorship arena: a mentee submits a query for a question, and the
runner grades it by comparing the **normalised** output against a pre-computed **golden
result**. Before any submission can be scored, the questions (Q5–Q9) and their golden
results must exist. The golden results are derived from trusted **reference queries** (the
mentor's naive-but-correct baselines) which are kept out of the repo — a mentee who could
read them would have the answer.

Step 4 (issue #8, blocked-by #4 seed, now **merged**) builds that grading oracle:
load Q5–Q9 metadata into `app.questions`, run each reference query against the seeded
`seed` schema, normalise the result, and store it as `golden_result` (jsonb) via a
re-runnable `npm run db:questions` loader wired exactly like `db:bootstrap` / `db:seed`.

**Outcome:** `app.questions` populated with 5 rows, each carrying its `ordered` flag and a
normalised `golden_result`; a shared normalise module the Step-5 runner will reuse; an
idempotent loader; secret reference queries kept server-only.

## Pre-work

- `git fetch origin && git checkout main && git pull` — local `main` is stale (PR #7 merged
  on GitHub at 16:45, not yet pulled). Then branch: `task/GH-8-question-registry-golden-results`.
- Active ticket marker already written (`.claude/session/tickets/sql-arena` → #8).

## The five questions

| Code | Prompt (mentee-facing) | `ordered` |
|------|------------------------|-----------|
| Q5 | Total number of products in each category | `false` |
| Q6 | Top customers by total spending | `true` |
| Q7 | 1000 most recent orders with customer information | `true` |
| Q8 | Products with low stock (`stock_quantity` < 10) | `false` |
| Q9 | Revenue generated from each product category | `false` |

`ordered` flags per the ACs: Q6/Q7 `true`, Q5/Q8/Q9 `false`.

## Files to create / modify

All paths under `workspace/sql-arena/` (the live working copy; this is the real git repo
for `G0maa/sql-arena`).

### 1. `src/seed/questions.ts` (new) — committed question registry
Non-secret metadata only. Exported `QUESTIONS` array of
`{ code, title, prompt, ordered }` for Q5–Q9. The reference query is **not** here — it is
parsed from the gitignored secrets file at load time and joined by `code`.

### 2. `src/grading/normalise.ts` (new) — shared normalisation, runner reuses it
Single source of truth for the correctness rule. Per tech-design.md § "Submission Runner"
and AgDR-0002: **values only** (column aliases/names ignored, extra columns ⇒ mismatch),
**order-insensitive except** `ordered` questions.

- `normaliseRows(rows, fields, ordered)` → returns array-of-arrays (each inner array = a
  row's column **values** in column order), sorted by a deterministic row comparator unless
  `ordered`.
- A `canonicalValue()` helper makes value comparison type-stable across the pg driver's
  representations (numeric → string is left as-is, `Date` → ISO string, `null` explicit) so
  the loader and the future runner produce byte-identical JSON for equal result sets. This
  determinism is load-bearing: the runner compares submission output to `golden_result`
  using this exact function.
- Pure, dependency-free → trivially unit-testable when test infra lands in Step 5.

### 3. `src/seed/load-questions.ts` (new) — the loader (mirrors `src/seed/load.ts`)
Pattern copied from `load.ts:34-77`:
- Export `loadQuestions(pool: Pool)` (so Step-5 reset/setup can reuse it), plus a
  `require.main === module` CLI wrapper that builds a `pg.Pool` from `$DATABASE_URL`,
  logs progress, sets `process.exitCode` on failure, and `pool.end()`s in `finally`.
- Read `secrets/reference_queries.sql` (dir overridable via `SECRETS_DIR`, default
  `./secrets`). Parse into a `code → query` map by splitting on `-- Q<n>:` header markers
  (block = everything up to the next marker). Fail loudly if a question in the registry has
  no matching reference query, or the secrets file is missing (clear "copy the .example"
  message).
- One transaction (`BEGIN`/`COMMIT`/`ROLLBACK` like `load.ts`): for each question, execute
  its reference query against `seed`, call `normaliseRows(...)`, then **upsert**:
  `INSERT INTO app.questions (code,title,prompt,reference_query,ordered,golden_result)
   VALUES (...) ON CONFLICT (code) DO UPDATE SET ...` — idempotent re-run.
- Connection role: same as `loadSeed` — `$DATABASE_URL` (dev superuser `arena`, which can
  read `seed` and write `app`). No new role needed.

### 4. `secrets/reference_queries.sql` (new, **gitignored** — local/server only)
The five naive-but-correct reference queries authored against the real `seed.*` columns
(category/product/customer/orders/order_details — see `db/sql/seed.sql`). "Naive" =
straightforward, un-optimised (no manual index hints, simple joins/subqueries) so mentees
have a real baseline to beat. Already covered by `.gitignore` (`/secrets/*` except
`*.example`) — will NOT be committed.

### 5. `secrets/reference_queries.sql.example` (modify) — committed template
Tighten the existing template so the `-- Q<n>:` marker format the parser expects is
explicit and documented (the loader's contract). Keep queries as placeholders only.

### 6. `package.json` (modify) — wire the script
Add alongside the existing `db:*` scripts:
```json
"db:questions": "tsx src/seed/load-questions.ts"
```

### 7. `projects/sql-arena/docs/agdr/AgDR-0008-*.md` (new) — decision record
Short AgDR per the per-step cadence (0006 bootstrap, 0007 seed → 0008 questions). Records:
(a) non-secret metadata in committed `questions.ts` vs answer-revealing query in gitignored
secrets, joined by `code`; (b) reference-query parse-by-`-- Q<n>:`-marker; (c) the shared
`src/grading/normalise.ts` module the runner reuses, and why determinism matters. Links
AgDR-0002 (normalisation rule) and AgDR-0007 (loader mechanism).

### 8. `README.md` (modify) — bump status Step 3 → Step 4, document `npm run db:questions`.

## Verification (manual — no test infra until Step 5, consistent with #4)

1. `docker compose up -d db` then `npm run db:bootstrap && npm run db:seed` (small scale,
   e.g. `SEED_CUSTOMERS=200 SEED_ORDERS=1000`) to get a populated `seed`.
2. Create `secrets/reference_queries.sql` from the template with the five queries.
3. `npm run db:questions` → expect 5 `✓` log lines, no errors.
4. Assert in psql:
   - `SELECT count(*) FROM app.questions;` → **5**.
   - `SELECT code, ordered, jsonb_typeof(golden_result), jsonb_array_length(golden_result)
      FROM app.questions ORDER BY code;` → every `golden_result` is a non-null `array`;
      `ordered` true only for Q6/Q7; Q7 length ≤ 1000.
   - Spot-check one `golden_result` against the reference query run by hand → values match,
     and an unordered question's rows are sorted deterministically.
5. Re-run `npm run db:questions` → still 5 rows, no duplicate-key error (idempotent upsert).
6. `npm run lint && npm run typecheck && npm run format` clean before PR.

## Notes / gates

- **sql-arena skips the QA gate** and allows ops-repo doc commits + close-on-merge (per
  saved memory). Code review (Rex) + per-PR CEO approval still apply.
- Coverage gate: project has deliberately deferred test infra to Step 5; #4 merged without
  unit tests. Verification is the manual sequence above. `normalise.ts` is written
  pure/dependency-free specifically so Step 5 can unit-test it.
- PR: `task(#8): question registry + golden results`. Body needs Summary (narrative
  bullets) + Glossary (golden result / reference query / ordered question / normalisation).
