---
name: check-point
type: skill
repo_intent: published
status: live
owner: quality-security
last_reviewed: 2026-05-24
applies_to_projects: all
github_repo: https://github.com/shifteddevlabs/check-point
description: "Independent senior code-review methodology (Generator-Verifier separation, IV&V tradition) that finds race conditions, auth/authz gaps, input-validation holes at trust boundaries, state mutations on error paths, and database concurrency bugs in Next.js + Supabase web apps, iOS Swift apps, and Python tooling. Enforces a strict three-element rule (where: file:line, trigger: how to reproduce, why-not-prevented: why existing code does not block it), a four-question concurrency checklist for any Task/useEffect/onChange/DispatchQueue/actor code, CRITICAL/HIGH/MEDIUM/LOW severity rubric, and mandatory before/after fix code (no hedging, no style nits). Pairs with Semgrep static scanning in the /push gate. Runs a one-time onboarding interview per repo that writes .github/review-context.md so future reviews calibrate to known-safe patterns and sharpen over time."
triggers:
  - check point
  - check-point
  - code review methodology
  - independent code review
  - run check point
  - review this diff
  - review this code
  - find bugs in this
  - code review with onboarding
  - security review methodology
---

# Check Point

Independent code-review methodology with severity rubric, concurrency checklist, and
first-run interview protocol.

## Primary Content

- `identity.md` — purpose and scope of the methodology
- `rules.md` — the review rules and severity rubric
- `examples.md` — worked review examples
- `reference/` — supporting reference material (first-run interview, checklists)

## Usage

Load `identity.md` and `rules.md` as the canonical operating instructions for this capability.
This SKILL.md is a thin wrapper and discovery entry point.
Canonical repo: https://github.com/shifteddevlabs/check-point
