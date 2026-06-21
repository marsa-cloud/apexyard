---
paths:
  - "apps/api/src/**"
---

ENFORCEMENT: advisory

# Handbook: No redundant comments (marsa apps/api)

**Scope:** PRs touching `apps/api/src/**` in the marsa NestJS backend.
**Enforcement:** advisory

## The rule

Comments earn their place by explaining non-obvious **why** — a constraint, a trade-off, a gotcha. Don't narrate **what** the code already states. If a line needs a comment to be understood, prefer a clearer name or smaller function; trust the reader to ask in review.

```ts
// redundant — the code already says this
// fork() gives a clean EM independent of request-context middleware
const em = this.em.fork()

// earns its place — non-obvious why
// Atomic conditional delete → verifies at most once, no replay.
const deleted = await em.nativeDelete(State, { id, expiresAt: { $gt: now } })
```

## Why

Redundant comments are upkeep with no payoff: they drift out of sync with the code, add noise that buries the comments that matter, and signal mistrust of the reader. A multi-line doc block restating a builder's field-by-field assignment, or `// loop over items` above a `for`, is pure cost. Spend the comment budget on the one line where intent isn't recoverable from the code.

## What Rex flags

- A comment that paraphrases the very next statement (`// increment counter` / `counter++`).
- A multi-line doc block on a class/function whose body is self-evident, where one line (or none) would do.

## What's NOT a violation

- Comments capturing a non-obvious why, an invariant, a workaround, a spec/ticket reference, or a security rationale.
- Public-API / handbook-referenced doc comments that a consumer reads without opening the implementation.
- TODOs with an owner/ticket.

---

_Source: PR #64 comment by @G0maa on 2026-06-10_
_See: https://github.com/marsa-cloud/marsa/pull/64_
