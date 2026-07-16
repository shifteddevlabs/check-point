# Money / Derived-Count Fail-Closed Checks

Two failure modes specific to any dashboard or page that computes a financial metric (cost, spend, margin) or a derived count from two independently-sourced values. Distilled from the command-point AI-Health-Export money-page review (2026-07, ahe-trends.ts + app/(dashboard)/ai-health-export/money/page.tsx). Apply alongside the universal review when the diff touches a money/spend/margin/cost calculation, or subtracts/compares two counts sourced from different systems (e.g. ad-platform installs vs. App Store Connect installs).

Each check obeys the three-element rule: only raised if I can name the *where*, the *trigger*, and the *why-not-prevented*.

---

## 1. Unchecked Supabase/query error silently zeroes a money metric — MEDIUM (understated cost, inflated margin)

`const rows = res.data ?? []` used directly on a financial query without first checking `res.error`. A transient error, an RLS misconfig, or a schema change makes the query fail, `res.data` comes back `null`, and the `?? []` silently treats "query failed" the same as "legitimately zero rows."

- **Where:** any `Promise.all([...])`-destructured Supabase result feeding a cost/spend/margin calculation, at the `?? []` (or `?? 0`) fallback.
- **Trigger:** the query errors (RLS change, dropped column, timeout) — `res.data` is `null`, `res.error` is set but unchecked; ad spend / cost reads as 0 for the window.
- **Why-not-prevented:** nothing distinguishes "0 rows because nothing happened" from "0 rows because the query failed." On a financial dashboard this UNDERSTATES all-in cost and INFLATES margin with no operator-visible signal.
- **Fix:** check `res.error` before touching `res.data`. Fail closed — return `null`/`undefined` (or throw) so the caller can render "data unavailable" rather than a wrong number, and `console.error`/Sentry-capture the error. Never let a query error render as a legitimate zero on a money metric.

## 2. Derived count silently clamped to zero — MEDIUM (hides a real mismatch as a bug-shaped non-event)

`Math.max(0, totalCount - subCount)` (e.g. `organicInstalls = Math.max(0, installs - paidInstalls)`) is used to keep a derived count from going negative. When the two counts come from different systems with different attribution windows (ad-platform install count vs. App Store Connect install count), `subCount` can legitimately exceed `totalCount` — the clamp then silently floors the derived value to 0 instead of surfacing that the two sources disagree.

- **Where:** any `Math.max(0, a - b)` (or equivalent floor-at-zero) where `a` and `b` are counts sourced from two different systems/windows.
- **Trigger:** `b > a` due to window skew or attribution lag between the two sources; the UI shows `0` for the derived value with no indication anything is off.
- **Why-not-prevented:** the clamp exists specifically to suppress the negative-number case, which also suppresses the signal that something needs investigating.
- **Fix:** keep the clamp for display (never show a negative), but compute and surface an explicit mismatch flag/disclosure alongside it (e.g. "paid (X) > total (Y); windows differ") rather than letting the clamped value read as an unremarkable zero.

## Wiring (add to check-point/rules.md "Things I always do")
Add one bullet alongside the existing `reference/*.md` triggers (after the `supabase-ai-app-abuse-checks.md` / `paginated-api-connector-checks.md` lines):

> - When the diff touches a cost/spend/margin calculation or subtracts/compares two counts sourced from different systems, apply `reference/money-metric-fail-closed.md` (unchecked query-error zeroing a money metric, derived-count silently clamped to zero hiding a real mismatch).
