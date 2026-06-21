---
paths:
  - "apps/api/src/**"
---

ENFORCEMENT: advisory

# Handbook: Place utility functions by reach (marsa apps/api)

**Scope:** PRs touching `apps/api/src/**` in the marsa NestJS backend.
**Enforcement:** advisory

## The rule

A standalone helper function gets a home chosen by how far it reaches:

- Used by a single use-case / feature module → a feature-local `utils/` directory inside that feature (e.g. `src/app/github-app/utils/app-name.ts`).
- Genuinely cross-cutting (used by, or useful to, multiple features) → `src/utils/`.

Don't leave a general-purpose helper defined inline at the bottom of a use-case or config file.

```ts
// get-manifest.use-case.ts — don't leave appName() trailing here
function appName(webUrl: string): string { … }

// move to src/app/github-app/utils/app-name.ts (feature-local)
// move stripTrailingSlash() from github-app.config.ts to src/utils/ (cross-cutting)
```

## Why

A helper trailing a use-case is invisible to the next person who needs it — they re-implement it. Placing it by reach makes it discoverable, testable on its own, and keeps the use-case file about application logic. The reach test (one feature vs many) decides local-`utils/` vs `src/utils/`, so helpers don't all pile into a global bucket nor hide inside unrelated files.

## What Rex flags

- A non-trivial, reusable pure function defined inline in a `*.use-case.ts`, `*.controller.ts`, or `*.config.ts` instead of a `utils/` module.
- A clearly cross-cutting helper (string/url/date manipulation) living in a feature folder where another feature would have to reach into it.

## What's NOT a violation

- A tiny one-off closure used once, local to a single function.
- A helper genuinely specific to one use-case and not reused — a feature-local `utils/` file is enough; it need not go to `src/utils/`.

---

_Source: PR #64 comments by @G0maa on 2026-06-10_
_See: https://github.com/marsa-cloud/marsa/pull/64_
