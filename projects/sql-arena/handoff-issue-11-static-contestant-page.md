# Handoff — sql-arena #11: Static Contestant Page

> **For a fresh Claude Code instance with no conversation history.** This file is
> self-contained: it has the full API contract, the file to edit, the agreed design
> decisions, and the verification steps. Read it top to bottom and you can execute #11
> without re-exploring.

## Before you touch code

1. The active-ticket hook (`require-active-ticket.sh`) blocks edits to non-`*.md` paths
   until a ticket marker exists. Run: **`/start-ticket G0maa/sql-arena#11`**
   (a marker may already exist at `<ops_root>/.claude/session/tickets/sql-arena` — it's
   gitignored and per-machine, so it likely survived the account switch; re-run if unsure).
2. Suggested branch: `feature/GH-11-static-contestant-page`.
3. Ticket: <https://github.com/G0maa/sql-arena/issues/11> · Epic #1 · blocked-by #10 (API, already shipped).

## Context

Contestants need a no-build, single-page UI to pick a SQL question, optionally add
index/setup SQL (untimed), submit a solution, watch it run, and see where they rank. This
is the human-facing end of the submit→rank loop and the last piece before deploy
(tech-design Step 7). The backend (#10) is in place and serves the page same-origin via
`ServeStaticModule`.

## Agreed decision (already made with the operator)

**Plain HTML/JS/CSS with an inline `<style>` block — no framework, no build step.**
Tailwind was explicitly considered and **rejected**: the page is small and a CDN dependency
contradicts the ticket's "no framework / works offline" intent. Do not introduce Tailwind.

## File to modify

- **`workspace/sql-arena/src/public/index.html`** — replace the current placeholder
  scaffold (~52 lines, "submission UI ships in a later step") with the full contestant page.
  This is the *source* file; the Nest build copies `public/` → `dist/public/`.
  **Do NOT edit `dist/public`** (build output).

Backend reference files (read-only, for confirming shapes):
- `workspace/sql-arena/src/arena/arena.controller.ts` — `@Controller('api')`, the 4 routes.
- `workspace/sql-arena/src/arena/arena.service.ts` — response types / DTOs.
- `workspace/sql-arena/src/app.module.ts` — `ServeStaticModule` (rootPath `../public`, excludes `/health`, `/api/*`).
- `workspace/sql-arena/src/main.ts` — **no** global prefix (the `/api` is from the controller), **no** CORS (same-origin, not needed).

## API contract (relative same-origin fetches — no base URL, no CORS)

| Call | Shape |
|------|-------|
| `GET /api/questions` | `[{ code, title, prompt }]` — **`code`** is the identifier (e.g. Q5–Q9); there is no `id`/`slug` |
| `POST /api/submit` | body `{ question_code, display_name, setup_sql?, sql }` → `{ submission_id, status:'queued' }` |
| `GET /api/submission/{id}` | `{ status:'queued'\|'running'\|'done', position?, result?, exec_ms?, ranked?, message? }` |
| `GET /api/leaderboard/{code}` | `[{ rank, display_name, exec_ms }]` — `{code}` = the question `code` |

Gotchas (these bit during exploration — honor them):
- The verdict field is **`result`** (NOT `verdict`); values `correct | incorrect | error | timeout`.
- **`position`** is present **only** while `status === 'queued'`.
- **`exec_ms` is a string** in every response (PG numeric/bigint) — `Number()` before math, render directly otherwise.
- Submit body: `setup_sql` optional (string|null); `sql` required, max 32 KiB, must be a single
  SELECT/WITH. A bad solution comes back `status:'done', result:'error'` with a `message` — show it.
- `404` from `/api/submission/{id}` on unknown id; `400` from submit on empty/oversized fields.

## Page structure (single file, inline `<style>`)

Reuse the existing scaffold's dark-slate tokens for continuity: bg `#0f172a`, text
`#e2e8f0`, muted `#94a3b8`, panel `#1e293b`, accent `#7dd3fc`, system-ui font, 4px radius.

1. **Header** — "🏟️ SQL Arena".
2. **Question picker** — `<select>` from `/api/questions`; on change render the selected
   question's `prompt` into a read-only panel AND refresh the leaderboard for that `code`.
3. **Display-name** — text input.
4. **Setup-SQL textarea** — labelled exactly **"Setup SQL (optional — add indexes here; not timed)"**
   (AgDR-0004 wording), placed *above* the solution field.
5. **Solution-SQL textarea** — the timed query.
6. **Submit button** — disabled while a submission is in flight; also gate on
   (question selected + non-empty name + non-empty solution).
7. **Status panel** — poll progression: `queued (position N)` → `running` → done
   (verdict badge + `exec_ms` when correct; `message` for `error`/`timeout`).
8. **Leaderboard panel** — table (rank / name / exec_ms) from `/api/leaderboard/{code}`;
   refetched on question-change and after a `correct` run.

### JS behavior (vanilla, inline `<script>`)
- `loadQuestions()` on load → populate picker, select first, render its prompt + leaderboard.
- `submit()` → POST `/api/submit`, then `pollLoop(submission_id)`.
- `pollLoop(id)` → ~1000ms `setTimeout` recursion on `/api/submission/{id}` until
  `status==='done'`; update the status panel each tick; on done+`correct` refresh the
  leaderboard; always re-enable submit when done/errored.
- Wrap all fetches in try/catch; surface network/HTTP errors in the status panel.

## Verification (do this before claiming done — evidence, not assertion)

```bash
cd workspace/sql-arena/src
# Ensure DB is up + seeded (docker-compose) — see project README; then:
npm run build && npm run start:prod    # serves dist/public/index.html at http://localhost:3000/
# (or npm run start:dev — confirm it also serves public/)
```

Browser at `http://localhost:3000/`, walk the ACs:
- [ ] Picker populates from `/api/questions`; selecting a question shows its prompt.
- [ ] Setup-SQL textarea present, optional, marked "not timed".
- [ ] Submit enqueues; status shows `queued (position N)` → `running` → result.
- [ ] Result shows verdict + `exec_ms`; an erroring/timeout solution shows `message`.
- [ ] Leaderboard for the active question renders and refreshes after a correct run.
- [ ] Works as a single static file, no build step for the page itself.

Smoke **both** a correct solution and an intentionally-broken one to confirm both result paths render.

## Out of scope
- No new backend endpoints / DTO changes (#10 shipped them).
- No auth, no websockets (poll loop is the agreed model — AgDR-0003).

## PR / merge notes (apexyard SDLC)
- One ticket = one PR. PR title format: `feat(#11): static contestant page`.
- PR body needs a **Summary** (narrative bullets, what + why) and a **Glossary** (mandatory).
- This PR touches UI → a design-review gate applies (`/approve-design <pr>` before merge).
- Merge needs Rex (code review) + explicit per-PR CEO approval via `/approve-merge <pr>`.
  Do **not** merge on an umbrella "go".
