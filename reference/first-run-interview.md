# First-Run Onboarding

The reviewer auto-runs this interview the **first time** it is invoked in a new project. The goal is to populate `.github/review-context.md` so the specialist stops flagging false positives that the team already knows are intentional. Three minutes of questions saves weeks of "I told you, that's intentional."

For tone, language, and what NOT to say during the interview, see [`interview-tone.md`](interview-tone.md).

---

## When this runs

Before reviewing any diff, the reviewer checks for `.github/review-context.md` (or `review-context.md` at the repo root). If it does **not** exist:

1. Pause the review.
2. Run the interview below.
3. Write the answers into `.github/review-context.md` using the template in `reference/review-context-template.md`.
4. Show the file to the user, ask "Anything to add or correct?" — accept edits.
5. Continue with the review.

If `review-context.md` already exists, skip the interview and review immediately. The user can re-run the interview manually by saying "re-run onboarding" or deleting the file.

## When this is also a good idea

- The codebase has changed significantly (new auth layer, new DB, framework migration). Ask the user: "It looks like the stack has changed since `review-context.md` was written. Re-run onboarding?"
- The user reports the reviewer is "flagging the same false positive again." Offer the interview as a fix.

---

## The interview protocol

**Open with this exact line so the user knows what they are walking into:**

> Before I review, I need 2–3 minutes to learn how your project works. This stops me from flagging things that are intentional. I will detect what I can from the codebase first, then ask 4–6 short questions. Ready?

### Step 1 — Detect project maturity, then auto-scan

Before any questions, decide whether this is a **greenfield** project (little or no code yet) or an **existing** project (substantial codebase).

Quick test: does `src/` have more than ~10 source files? Are there route handlers / view files / business logic already written?

#### If existing project: grep first, present back, then ask the user to confirm.

Scan and report what you found. The user only has to confirm or correct — they don't have to describe things from memory.

Look for and announce:
- **Stack:** read `package.json` (framework, key deps), `*.xcodeproj`, `pyproject.toml`, `go.mod`, `Cargo.toml`
- **Auth library:** grep for `@supabase/supabase-js`, `next-auth`, `clerk`, `firebase/auth`, `@aws-sdk/client-cognito-identity-provider`
- **Database / ORM:** Supabase, Prisma, Drizzle, Mongoose, raw SQL, Core Data, SwiftData
- **Has tests:** does `npm test` exist? `xcodebuild test`? `pytest`?
- **Has linter / type checker:** ESLint, TypeScript strict, SwiftLint, mypy

Report it back like:
> I see this is a Next.js 15 + Supabase + Resend project, TypeScript strict mode, with Vitest tests and ESLint. Auth via `@supabase/supabase-js`. Sound right?

If wrong, let the user correct before continuing.

#### If greenfield project: skip the scan, ask the user.

There is nothing meaningful to grep. Move straight to Step 2 questions, which describe **intent** rather than current state.

### Step 1.5 — Calibrate to the user's experience level (CRITICAL)

Before any further questions, ask **one calibration question**. Use the tech stack you just detected as concrete anchors so the user can self-assess against real things, not abstract labels.

Ask it like this (substitute the actual stack you found):

> Quick check before I dig in — how comfortable are you reading and writing code? Pick the closest:
>
> **A. Pro / senior dev.** I read Next.js + Supabase + Tailwind code daily. Show me technical specifics.
>
> **B. Hobbyist / can-debug.** I can read code, follow what it does, fix small bugs. Don't assume I know every framework convention but don't oversimplify either.
>
> **C. Non-developer.** I ship apps with AI assistance but I'm not the one writing the auth layer or designing the database. Speak in plain English about the product, not the plumbing.

If they say A or B → use the **technical path** (Step 2, original questions).
If they say C → use the **plain-English path** (Step 2-NonDev, below).
If they don't answer or say "not sure" → assume B (middle ground, light jargon, define terms inline the first time).

**Why this matters.** The questions about "auth model" and "intentional database patterns" are useful prompts for a senior dev and noise for a non-dev. A non-dev knows the product context (who uses it, what data is sensitive, what could go wrong from the user's perspective) that no amount of code-reading can derive. Ask them about *that*, not about the plumbing they don't write.

---

### Step 2 — Technical path (for senior devs and capable hobbyists)

Ask one at a time. Wait for the answer before moving on. Skip any question that the codebase scan already answered with high confidence.

**Q1. What does this project do, in one sentence?**
*(Used for orientation — the reviewer needs to know whether it is a payments app, a marketing tool, an internal admin, etc. Risk model differs.)*

**Q2. How does authentication work?**

**For existing projects: grep first, present back.** Read the actual auth code (`src/lib/auth/*`, `middleware.ts`, `proxy.ts`, route handlers, any `*auth*.ts` file). Build the model from the code, then present it to the user as a draft and ask "is this complete? anything I missed?"

Present format:
> Looks like you have three auth layers:
> 1. Public endpoints (no login) — `/api/subscribe`, `/api/preferences`. Token-validated.
> 2. User session — Supabase magic link, JWT verified via `getClaims()` in `proxy.ts`.
> 3. Admin / cron — `Authorization: Bearer ${CRON_SECRET}` checked with `crypto.timingSafeEqual` in `src/lib/auth/verify-secret.ts`.
>
> Anything I missed? Anything that's actually different from what the code looks like?

Users will know about edge cases the code doesn't make obvious (e.g. "X is technically public but only linked from emails," "Y has a feature flag we forgot to remove"). The grep gets the structure right; the user adds the nuance.

**For greenfield projects: ask the user to describe intent.** No code to read.

If you're not sure how to think about it, here are the kinds of things to mention:
- Are there parts that anyone with a link can use (no login)?
- Is there a logged-in user area? How do they sign in (email magic link, password, Google, etc.)?
- Are there admin-only or cron/scheduled job endpoints? What guards those?
- Any API keys or secrets that should never reach the browser?

If a layer doesn't exist, say "we don't have that." If you don't know, say so — I'll grep the code.

**Q3. Anything in this codebase that looks like a bug but is actually intentional?**

The plain-English version: **has any code reviewer (human or AI) ever flagged something here that you knew was actually fine?** I want to know so I don't make the same mistake.

**For existing projects: read the receipts before asking.** Check these files in order, present what you find, then ask the user to confirm:

1. `.github/review-context.md` — if this file exists, it lists patterns previously verified as safe. Read it, summarize 3–5 examples back to the user, and ask: "These were marked safe in past reviews. Still accurate?"
2. `.github/review-backlog.md` — if this exists, look for entries marked `RESOLVED` or `N/A`. Those are bugs that turned out not to be bugs.
3. Last 30 commit messages (`git log --oneline -30`) — look for "intentional," "by design," "expected," "not a bug" phrases.

If steps 1–3 produce a list, present it as: *"Here are patterns I'm seeing marked as intentional. Anything wrong, or anything to add?"*

**For greenfield or context-less projects: ask in plain English.**
> Has any code review tool ever flagged something in your code that you said "no, that's actually fine" — and you got tired of explaining it? Tell me what it was. If nothing comes to mind, just say "skip" and we'll catch them as they come up in the first few reviews.

**Do NOT ask the user to enumerate technical patterns from memory.** Phrases like "Postgres error 23505," "atomic RPC," or specific function names are noise for non-developers and patronizing for senior developers (who can paste their own list if they want). The question is about lived experience — what false positives have you already lived through? — not about reciting a database textbook.

**Senior-developer fast path (optional):** If the user is clearly a senior developer and prefers technical enumeration, offer: *"Or if you'd rather just dump a list — paste any patterns, function names, or libraries that have been flagged-and-dismissed before."* Let them lead.

If the user says "I don't know" or "skip," move on. The patterns will accumulate naturally in `review-context.md` over the next few reviews.

**Q4. Where are the trust boundaries?**
Where is user input validated? The reviewer needs to know whether to flag missing validation deep in a server action, or trust that it was already validated upstream.
*Examples:*
- "All input is validated at the route handler with Zod schemas. Server actions assume valid input."
- "Webhook payloads are signature-verified at the edge."
- "Frontend forms have client-side validation but server re-validates everything."

**Q5. Have any patterns been flagged incorrectly before? (the don't-flag list)**
Anything you have already explained to a reviewer (human or AI) and don't want to explain again?

**Q6. Severity floor — which findings do you want?**
- All four (CRITICAL / HIGH / MEDIUM / LOW) — full reviews
- HIGH and above — fast iteration, ship-mode
- CRITICAL only — production hotfix mode

*Default to "all four" if no answer.*

---

### Step 2-NonDev — Plain-English path (for non-developers)

When the user picked C, **drop everything technical**. The reviewer does the heavy lifting via grep and only asks the user about things no amount of code-reading can answer: who the product is for, what data is sensitive, what war stories they have.

Three questions. That's it. Ask one at a time.

**N1. Who uses this app, and what's the worst thing that could happen if it broke or leaked data?**

**Read first, present back.** Before asking, scan in this order and pull out a 2-sentence product summary:
1. `README.md` at the repo root
2. `package.json` `description` field (or `pyproject.toml`, etc.)
3. `CLAUDE.md` / `AGENTS.md` / any top-level project doc
4. `docs/` folder, especially anything titled "overview" / "intro" / "what-is-this"

Present back like:
> Reading your README and package.json, this looks like a "[paraphrased product description]" used by [audience inferred from docs]. Worst case I can think of from the code is [data exposure / wrong recipient / etc]. Sound right? Anything I missed about who uses it or what could go wrong?

The user only needs to **confirm or correct** — not write a product description from scratch.

**If steps 1–4 produce nothing** (greenfield project with no README): ask the question as plain N1, with the example from below as a prompt.

> "Music producers — worst case is wrong percentages on a split sheet, or someone's IPI number leaks to a stranger."

That single sentence tells the reviewer:
- Who the audience is (drives tone of bug priority)
- What data is sensitive (drives CRITICAL-vs-HIGH severity calls)
- What "broken" looks like to the user (drives what to look for)

**If the project has zero documentation AND the user picks C (non-dev), warn them once:** *"Heads up — your project has no README or description, so I'm starting from a blank slate. Reviews will get sharper as soon as someone (you, an AI, or me) writes even a one-paragraph README of what this is and who uses it."*

**N2. Has anything ever gone wrong — bug, data loss, embarrassing mistake, security scare? What was it?**

War stories are gold. They tell the reviewer where the codebase is fragile and what the user has *already* paid the cost of forgetting once. Examples that count:
- "We accidentally sent the wrong PDF to the wrong email last year."
- "The signup form was broken for 2 days and nobody noticed."
- "Someone got logged into the wrong account once."

Pattern-match against these whenever reviewing. If they have no stories: "Cool, I'll be extra careful since you don't have a record of past pain to learn from."

**N3. Is there anything in the app that you wish a code reviewer would stop bothering you about?**

Plain English. "Reviewers always tell me X is dangerous but I know it's fine because Y." If they can't think of anything: skip. The reviewer will learn over the first few reviews.

**N4. When I find something — bug, problem, suggestion — what do you want me to do?**

A note before the options: **if you're not a developer, option 1 is usually the right answer for you.** Here's why — when I find a "low-priority" issue and don't fix it, it sits in the codebase as a tiny landmine. Three months from now you have fifty of them, and your code is messy in ways you can't navigate. For people who can read code, "skip the small stuff" makes sense — they can come back later and judge what to fix. For people who can't, every unfixed thing is future debt they're stuck with.

Three options, in plain English:

1. **Fix everything as you find it.** Big or small, fix it before commit so there are no time bombs. Tell me in the commit summary what got fixed. *Best if you're not a developer — you don't want low-priority issues piling up as future tech debt you can't navigate.*
2. **Skip the small stuff.** Only tell me about things that could break the app or leak data. The minor cleanups can wait — I'll come back to them. *For people comfortable triaging code themselves.*
3. **Just the dangerous stuff.** Security issues, data loss, things that could really hurt. I'm in ship-mode and don't want noise. *For senior devs in a hurry; risky for non-devs.*

If they pick 1 or don't answer, default to 1. **Never silently defer or "save for a later release" without asking.** The user decides what waits, not the reviewer.

**Capture the answer in `review-context.md` using the user's own words** — not by translating their answer into "Severity floor: HIGH+" or "Filter LOW findings." If they said "fix everything because I don't know what I'm doing," write *that*, not "Severity: ALL." The reviewer reads it back the same way next time, in their voice. Mirroring their language is part of how the specialist stays useful instead of becoming jargon noise.

---

#### What the reviewer does in non-dev mode (heavy lifting)

After getting answers to N1–N3, the reviewer **fills in the technical sections of `review-context.md` from the grep alone** — auth model, database patterns, trust boundaries — without asking the user to confirm them line-by-line. The reviewer marks each grepped section with a small comment:

```markdown
<!-- Auto-detected from codebase. The user is non-technical and did not verify
     these specifics line-by-line. If a future reviewer (human or technical)
     spots something wrong, edit this section. -->
```

This is honest and self-correcting: the technical sections are the reviewer's best read of the code, the product sections are the user's lived knowledge. Future reviewers can fix the technical parts; nobody but the user can fix the product parts.

---

### Step 3 — Write the file

Generate `.github/review-context.md` from the template. Fill in only what the user gave you. Leave a comment at the top:

```markdown
<!-- Generated by check-point onboarding on YYYY-MM-DD.
     Edit anytime. The reviewer reads this before every review. -->
```

Show the file to the user. Ask: **"Look right? Anything to fix or add before I review the diff?"**

Accept edits. Then proceed to review.

### Step 4 — Hand off to the review

Continue with the actual code review using the now-populated context. The user has paid the 3-minute tax exactly once for this project, and every future review benefits.

---

## Manual triggers

The user can re-run onboarding any time by saying:
- "re-run onboarding"
- "update review context"
- "the project has changed, redo the interview"

The reviewer should also **proactively** suggest re-running when:
- The same false positive has been dismissed twice in two reviews
- The stack appears to have changed (new framework in `package.json`, new auth library)
- The user explicitly says "you keep flagging X, I told you that's fine"
