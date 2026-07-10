---
name: Generator-Verifier separation — value log
description: "Concrete catches where an independent reviewer (Check Point) found real regressions the writer's own known-readers/known-callers list missed."
type: reference
applies_to: check-point
status: active
date_started: 2026-07-06
---

# Generator-Verifier separation — value log

Concrete evidence for the mandatory independent-reviewer gate (see identity.md, reference/methodology.md). Append new catches as they happen.

## Bluesky Chunk-2 review — caught a real BLOCKER
An independent review of the Bluesky integration's Chunk 2 caught a real BLOCKER-severity issue, fixed by deferring UI activation to Chunk 3 rather than shipping broken.

## Review method notes
Reviewers apply the Three-Element Rule (Where: file:line / Trigger: how to reproduce / Why-not-prevented: why existing code doesn't block it) and verify behavior against ground truth (e.g. Node's actual URL/isIP semantics) rather than from memory. Security-critical diffs are reviewed on Opus, default-to-skeptical, never rubber-stamped.

## ai-health-export session — reviewer caught bugs on nearly every commit
Independent Opus review caught real HIGH/CRITICAL issues across a full session: P0c wrong CSV column name (`sleep_hours` vs `sleep_duration_hr`) that would have silently broken sleep follow-ups for all users; P0b a missing 6th personalInfo disclosure category (App Store risk); P1a a `requires` marker mismatched against DailyAggregator's header; P1b 3 HIGH + 4 MEDIUM (save-error swallowing, missing consent disclosure, swipe-dismiss state loss, cutoff off-by-one); P2 HIGH-1 cache hash missing profile/units/date; a live-reproduced JSONSerialization NaN crash. An earlier Session-25 reviewer caught a CRITICAL severity-mapping inversion + a LAContext race. The reviewer also correctly returned NO-OP verdicts when a flagged bug wasn't real, and cross-checked `.github/review-safe-patterns.md` first to avoid re-litigating verified-safe code. Owner is non-technical (Jay-J) and relies on this gate as his primary trust mechanism for shipped code.

## quicksinq waitlist self-read regression — caught by the reviewer, not the writer, twice
The 2026-06-12 `20260612000100_harden_pii_metrics_queue_grants.sql` migration dropped `waitlist_leads`' broad authenticated-SELECT policy (a real HIGH fix — see `docs/reviews/2026-06-12-security-audit.md` QS-H-series). But `app/dashboard/page.tsx` read the caller's own pending-waitlist row via that same anon-key client; after the drop, the read returned null, flipping `isPendingUser` to false, so un-approved beta users fell through to the FULL dashboard — a real auth-gate bypass introduced BY the security fix. An independent Check Point reviewer caught it, not the writer; a second re-review of the same diff then found a THIRD reader of the dropped policy (`hero-section.tsx`) missing from the writer's own known-readers list. Fix: a scoped self-read RLS policy (`FOR SELECT TO authenticated USING (email = auth.email())`) restored the own-row read without reopening cross-tenant exposure; position/total counts moved to count-only SECURITY DEFINER RPCs. Now enforced by a source-level regression guard (`scripts/verify-security-invariants.ts`: "waitlist_leads has exactly 1 SELECT/ALL policy (self-read)"). Lesson: a writer's "exhaustive reader inventory" for a policy being dropped/scoped is not reliable on the first pass, even from the same agent that wrote the original audit — this is why the independent-reviewer gate is mandatory on every security diff, not just novel ones.
