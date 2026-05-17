# Check Point

**Repo name:** `check-point`. **Product name:** Check Point. (Renamed from `code-review-specialist` on 2026-05-14.)

An independent senior code reviewer that finds real bugs in pull requests and code diffs without drowning you in style nits or false positives. Built around one hard-won rule: **the writer cannot review their own code**.

This specialist solves the "AI reviewed its own code and missed the obvious bug" problem. It separates writer from reviewer, applies a strict evidence rule to every finding, and returns either high-confidence bugs or `PASS — 0 issues`. Strongest on race conditions, async lifecycle bugs, authentication gaps, and error-path state mutations.

**The part nobody else does:** a one-time 3-minute onboarding interview that runs before the first review. It calibrates to your experience level (senior dev / hobbyist / non-developer), reads your codebase via grep, then asks the questions only *you* can answer — who uses the product, what data is sensitive, which patterns look like bugs but are intentional. Future reviews read those answers and stop flagging things you already know are fine. Most AI code reviewers (Cursor's bug bot, GitHub's bug bot, Codium, Coursera-grade tools) are stateless and re-flag the same false positives forever; this one gets sharper over time.

## Quick start

1. Copy this folder into your repo, or upload its contents to your agent as project knowledge.
2. Tell your agent:

   > Read `identity.md` and `rules.md`. Review this diff. Report only findings with where, trigger, why-not-prevented, and a concrete fix. If clean, say `PASS — 0 issues`.

3. Paste your diff and the full file content for any changed file.

That's it. The first review in a new project will pause for a 2–3 minute interview to learn your project's conventions; every review after that is immediate.

## Why this works
The writer cannot review their own code. Confirmation bias guarantees blind spots in self-review; structural separation between the writer and the reviewer eliminates them. This pattern has decades of history in software engineering and a recent restatement in AI research as *Generator-Verifier separation*. Full reasoning and citations: [`reference/methodology.md`](reference/methodology.md).

## What good output looks like
For each issue:
```
[HIGH] src/components/edit-form.tsx:3
Trigger: User clicks Save, server returns non-OK, editor closes via setEditing(null) before the error check, draft text is lost.
Why-not-prevented: setEditing(null) runs unconditionally before the res.ok branch.
Fix:
  Before:  setEditing(null); if (!res.ok) toast.error(...);
  After:   if (!res.ok) { toast.error(...); return; } setEditing(null);
```

If the diff is clean: `PASS — 0 issues. Reviewed src/components/edit-form.tsx. Checked: error path state mutations, async race conditions, input validation.`

## How to use this folder in different agents

The methodology is portable. The specialist works in any agent platform that can read folders or accept system prompts.

**Claude Projects (claude.ai)**
1. Clone or download this folder.
2. Create a new Project.
3. Upload the four root markdown files plus the `reference/` folder contents to the Project knowledge.
4. Start a new chat in the Project and paste your diff. Tell the reviewer: *"Review this diff using Check Point. Here is `git diff --staged` and the full file content of `<filename>`."*

**Claude Code, Cursor, or any agent CLI**
1. Clone this folder into your repo (e.g. as `.claude/specialists/code-reviewer/`).
2. Reference it in your `CLAUDE.md`, `.cursorrules`, or system prompt: *"When asked to review code, follow `code-reviewer/rules.md`. Use `code-reviewer/identity.md` for voice. Apply `code-reviewer/reference/concurrency-checklist.md` to any async or lifecycle code in the diff."*
3. Run a review by saying: *"Review the staged diff using Check Point."*

**ChatGPT custom GPTs / Gemini Gems / other LLM tools**
1. Create a custom GPT / Gem / equivalent.
2. In the system instructions, paste the contents of `identity.md` followed by `rules.md`.
3. Upload the `reference/` files as knowledge.
4. Start a chat and paste your diff.

## First-run experience (≈3 minutes, once per project)
The first time you ask the reviewer to review a diff in a new project, it pauses and runs a short interview to learn how your project works. It calibrates to your experience level first (senior dev, hobbyist, or non-developer), then asks the right questions for that audience. A senior dev gets technical questions about auth, patterns, and trust boundaries. A non-developer gets plain-English questions about who uses the product and what could go wrong. Both paths produce the same `.github/review-context.md` so future reviews stop flagging things you already know are intentional. You pay this 3-minute tax once per project. See [`reference/first-run-interview.md`](reference/first-run-interview.md) for the protocol.

**Works best when your project has at least a basic README or `package.json` description.** The reviewer reads project docs (README, CLAUDE.md, AGENTS.md, package.json) before asking anything, so it can present a product summary for you to confirm rather than asking you to describe your own product from scratch. If your project has no docs, the interview still works — it just asks more questions and the first few reviews will be less precise.

## Folder map
- `identity.md` — who the reviewer is, what they cover, what they refuse to do
- `rules.md` — the three-element rule, severity definitions, output format, what to never flag
- `examples.md` — three worked examples (real bug, non-bug, concurrency issue)
- `README.md` — this file
- `reference/methodology.md` — Independent Code Review / Generator-Verifier Separation, with citations
- `reference/first-run-interview.md` — first-run interview protocol that populates project-specific context
- `reference/interview-tone.md` — tone and language rules for the onboarding interview
- `reference/concurrency-checklist.md` — mandatory four-question checklist for any async or lifecycle code
- `reference/known-safe-patterns.md` — universal patterns that look like bugs but are not
- `reference/severity-rubric.md` — when to use CRITICAL / HIGH / MEDIUM / LOW
- `reference/review-context-template.md` — drop-in template for project-specific safe patterns

## Optional: project-specific tuning
Copy [`reference/review-context-template.md`](reference/review-context-template.md) to `.github/review-context.md` in your own repo and fill it in (or let the first-run interview do it for you). The reviewer reads it before every review and stops flagging your project's intentional-but-unusual patterns. This is how reviews get smarter over time without re-litigating the same arguments every week.

## What this specialist will not do
Design critique, refactoring suggestions, performance benchmarking, style and formatting opinions, architecture proposals, documentation review. Those are different jobs and different specialists. This one finds bugs.

## P.S.
The mascot is the **M.A.D. Reviewer** duck. Modified, Added, Deleted: the only git statuses that matter for review. Now you know.

## License
MIT. Copy, modify, ship.
