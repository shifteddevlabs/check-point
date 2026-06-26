# Paginated-API Connector Abuse Checks

Three failure modes specific to connectors that page through a third-party API
(Meta/Graph, Google, X, Stripe — any cursor- or offset-paged endpoint) and write the
results to a DB or a health/deadman signal. They slip past "technically-correct" reviews
because the happy path works. Distilled from the command-point ad-connector
orchestration review (2026-06, prop_f1acd641a3dd61ac / prop_f13ae3a9d030516f). Apply
alongside the universal review when the diff touches a paginated fetch loop, a connector
that stamps `last_success_at` / drives a deadman healthcheck, or any code that follows a
`paging.next` / cursor URL.

Each check obeys the three-element rule: I only raise it if I can name the *where*, the
*trigger*, and the *why-not-prevented*. If I cannot, it is not a finding.

---

## 1. Access token embedded in the paging/cursor URL — HIGH (secret leak)

Many APIs (notably Meta Graph) return a `paging.next` / cursor URL with the
`access_token` baked into the query string. If that URL is logged, surfaced in an error
message, stored, or shipped in a healthcheck payload, the live token leaks.

- **Where:** the fetch-next loop + any `console.*` / logger / `throw new Error(url)` / row that persists the URL.
- **Trigger:** the loop logs or error-surfaces the next-page URL (e.g. on a non-200, `throw new Error(\`failed: ${nextUrl}\`)`), and that log/error sink is shared or shipped off-box.
- **Why-not-prevented:** the token query-param is never redacted before logging; the URL is treated as opaque. **Flag:** strip/redact `access_token` (and any `*token*` / `*secret*` param) before the URL touches a log, error, or stored row.

## 2. Advisory pushed into the same `errors[]` that drives the ok/deadman signal — HIGH (false-fail)

A connector collects non-fatal advisories (rate-limit nearing, partial field, deprecation
notice) into the same array it uses to decide healthy-vs-failed. One advisory then flips
the deadman to "down" and pages the on-call or halts the schedule, though every page
actually succeeded.

- **Where:** the shared `errors[]` / `issues[]` accumulator + the healthcheck/deadman that reads `errors.length === 0`.
- **Trigger:** a benign advisory is `errors.push(...)`-ed; the deadman reads non-empty and reports failure.
- **Why-not-prevented:** advisories and hard errors share one channel; the health signal can't tell them apart. **Flag:** separate `warnings[]` from `errors[]`; only true failures gate the ok signal.

## 3. Hard page cap with no overflow flag — CRITICAL (silent data loss)

A loop caps at N pages (`while (page < 100)`) to bound runtime, exits while a cursor still
points to more data, and stamps `last_success_at` — so the run looks complete while
silently dropping every row past the cap.

- **Where:** the page-cap guard + the success / `last_success_at` stamp.
- **Trigger:** the source has more than N pages; the loop exits at the cap, returns, and is recorded as a clean success.
- **Why-not-prevented:** hitting the cap exits the same way as "cursor exhausted," with no distinguishing flag. **Flag:** when the loop exits with a cursor still present, raise a `truncated: true` / overflow signal and do NOT stamp an unqualified success. Caps must fail loud, never silently truncate.
