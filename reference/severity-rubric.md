# Severity Rubric

Four levels. No "INFO" or "NIT." If a finding does not fit one of these four, it is not a finding.

---

## CRITICAL
**Definition:** Exploitable security vulnerability, data exposure, or cryptographic weakness. An attacker can reach this and cause real harm.

**Examples:**
- SQL injection via raw string concatenation in a query
- Missing auth check on an endpoint that mutates other users' data
- Service role / admin key exposed to the client bundle
- Webhook handler with no signature verification
- Password / token comparison with `==` instead of `timingSafeEqual` on a timing-sensitive boundary
- `dangerouslySetInnerHTML` rendering unescaped user input
- Open redirect via an unsanitized `next`/`returnTo` redirect param — a prefix check like `next.startsWith('/') && !next.startsWith('//')` is NOT sufficient: `/\evil.com` or its encoded form `/%5cevil.com` still passes (browsers/the WHATWG URL spec normalize a leading backslash to `/`, turning it into a protocol-relative `//evil.com` once followed). Validate with `new URL(next, origin)` and an explicit same-origin check on `.origin`, not a string prefix test.

**Bar:** I can describe a concrete attack scenario in one sentence.

---

## HIGH
**Definition:** A bug that **will** cause incorrect behavior in production. Not a remote possibility — an actual outcome under realistic use.

**Examples:**
- State mutation on the wrong code path (e.g. `setEditing(null)` before checking `res.ok`)
- Race condition between two lifecycle handlers writing to the same state
- Missing `await` causing a function to return before the promise resolves
- Wrong column referenced in a query (returns wrong data, not an error)
- Off-by-one in pagination (skips or duplicates rows at page boundaries)
- Using `.single()` on a query that can legitimately return zero rows (throws on every empty result)

**Bar:** I can describe an input or sequence that produces the wrong output, and that input or sequence is plausible in normal use.

---

## MEDIUM
**Definition:** Missing error handling that could crash, an edge case with real user impact, or a bug that fires only under uncommon-but-realistic conditions.

**Examples:**
- Network call with no timeout — fine on a fast connection, hangs forever on a flaky one
- `.map()` over a value that could be `undefined` after an API change
- File upload handler that does not check size — works until someone uploads a 5GB file
- Date parsing that assumes UTC but receives local time strings
- Cache key collision when two different inputs map to the same key

**Bar:** I can describe a realistic edge case where this breaks, and the impact is more than cosmetic.

---

## LOW
**Definition:** Code quality issue with a concrete before/after fix and a clear consequence. Not a style preference.

**Examples:**
- Dead code that confuses future readers (an exported function with no callers, a branch that cannot be reached)
- N+1 query that runs on every request and is trivial to batch
- Confusingly named variable where the name actively misleads (e.g. `userId` that holds an email)
- Magic number used in a security-relevant comparison without a named constant

**Bar:** Removing or renaming costs less than five minutes and the codebase is meaningfully cleaner. If the fix is "extract this into a helper," it is not a finding.

---

## When in doubt, downgrade
- Possible CRITICAL but exploitability requires an unrealistic precondition → HIGH.
- Possible HIGH but the broken path is rare → MEDIUM.
- Possible MEDIUM but no concrete consequence → LOW.
- Possible LOW but the fix is a style preference → not a finding. Drop it.

A small number of high-confidence findings beats a wall of hedged ones every time.
