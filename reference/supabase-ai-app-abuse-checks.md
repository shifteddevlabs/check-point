# Supabase / AI-App Abuse Checks

Four failure modes specific to AI-built Supabase/Firebase + LLM apps that slip past
"technically-correct" reviews. Distilled from the vibe-security-skill
(github.com/raroque/vibe-security-skill) canon, folded into Check Point's rubric and
three-element rule. Apply these alongside the universal review when the diff touches an
RLS table, a metered/paid endpoint (AI, email, SMS, image gen), a payment flow, or a
frontend/mobile build that calls a third-party provider.

The governing principle (from the source skill): **Never trust the client. Every price,
user ID, role, subscription status, feature flag, and rate-limit counter must be validated
or enforced server-side.**

Each check below still obeys the three-element rule — I only raise it if I can name the
*where*, the *trigger*, and the *why-not-prevented*. If I cannot, it is not a finding.

---

## 1. Entitlement / limit columns on a user-editable RLS table — CRITICAL

The non-obvious one. RLS can be **correct** (the user reads and writes only their own
rows) and the app can still be wide open to privilege escalation, because an entitlement
column lives on a row the user is allowed to `UPDATE`.

**What I flag:** any of these columns on a table where an authenticated-user RLS policy
grants `UPDATE` (or `INSERT` with these fields settable) on the user's own row:
`subscription_status`, `is_premium`, `is_pro`, `plan`, `tier`, `role`, `is_admin`,
`rate_limit`, `daily_limit`, `quota`, `credits`, `balance`, `trial_ends_at`,
`feature_flags`. The classic shape: a calorie/AI app where `profiles` has both
user-editable fields (name, avatar) AND `subscription_status`, with one RLS policy
`USING (auth.uid() = id) WITH CHECK (auth.uid() = id)` covering the whole row.

**Three elements:**
- **Where:** the table definition + the RLS `UPDATE`/`INSERT` policy (cite both file:line).
- **Trigger:** the user sends `PATCH /rest/v1/profiles?id=eq.<own-id>` (or the SDK
  equivalent) setting `subscription_status='premium'` / `role='admin'` / `credits=999999`,
  then calls the paid AI endpoint. The write passes because it is their own row.
- **Why-not-prevented:** the RLS policy authorizes self-`UPDATE` of the whole row; there
  is no column-level `GRANT`, no `WITH CHECK` pinning the entitlement column to its stored
  value, and no `BEFORE UPDATE` trigger pinning it to the old value. "RLS is on" is not the same as
  "this column is server-only."

**Fix (before/after):** move entitlement/limit fields off the user-writable table onto a
server-only table (no anon/authenticated `UPDATE` grant; written only by the service role
or a Stripe/webhook handler), OR revoke column-level UPDATE and add a trigger that rejects
client changes to those columns. Concrete pattern:
```
  -- Before: one table, user can UPDATE the whole row
  -- profiles(id, name, avatar_url, subscription_status, credits)
  -- policy: USING (auth.uid()=id) WITH CHECK (auth.uid()=id)

  -- After: entitlements split out, server-writes only
  -- profiles(id, name, avatar_url)                      <- user UPDATE ok
  -- entitlements(user_id, subscription_status, credits) <- NO authenticated UPDATE grant
  REVOKE UPDATE ON public.entitlements FROM authenticated, anon;
  -- writes happen only via service_role in the Stripe webhook / RPC
```
If splitting is too large for this diff, the minimum fix is a column-pinning trigger:
```
  CREATE FUNCTION pin_entitlements() RETURNS trigger AS $$
  BEGIN
    NEW.subscription_status := OLD.subscription_status;
    NEW.credits            := OLD.credits;
    RETURN NEW;
  END $$ LANGUAGE plpgsql;
  CREATE TRIGGER no_client_entitlement_edit BEFORE UPDATE ON public.profiles
    FOR EACH ROW EXECUTE FUNCTION pin_entitlements();
```

**Severity = CRITICAL.** It is privilege escalation / data-and-money exposure with a
one-sentence attack scenario — same bar as "missing auth check on an endpoint that mutates
other users' data" in the severity rubric.

**Not a finding when:** the entitlement column sits on a table with NO
anon/authenticated write grant, or a `WITH CHECK` / trigger already pins it, or the column
is read-only-derived (a generated column / view). Check the project's
`review-context.md` — if the split is already documented as the project's standard, do not
re-litigate.

---

## 2. Backend rate limits on cost-bearing endpoints — HIGH (CRITICAL if it gates an uncapped paid key)

Frontend rate limits (a disabled button, a `useState` counter, a debounce) are
cosmetic — anyone can call the endpoint directly. For endpoints that **cost money or send
things** (LLM/AI completion, email, SMS, image generation, embeddings, any metered
upstream), the limit must be enforced server-side, keyed on **both the authenticated user
AND the source IP** (IP alone is shared/NAT'd; user-id alone is bypassed by
unauthenticated or multi-account abuse).

**What I flag:** a metered endpoint whose only throttle is client-side, or that has no
per-user/IP server-side limit at all.

**Three elements:**
- **Where:** the route handler / edge function / RPC that calls the paid provider.
- **Trigger:** an attacker (or a runaway client) calls the endpoint in a loop, bypassing
  the UI entirely (curl, replayed request, second tab).
- **Why-not-prevented:** the only limiter is in the React/Swift client; the server accepts
  every request and forwards it to the metered upstream.

**Fix:** add a server-side counter keyed on `(user_id, ip, window)` — Upstash/Redis,
a Postgres `INSERT … ON CONFLICT` counter row, or Vercel/Cloudflare edge middleware — and
return `429` past the threshold *before* calling the upstream.

**Severity = HIGH** by default (cost abuse and degraded service *will* happen under
realistic use, not merely "might"). Escalate to **CRITICAL** only when the unthrottled
endpoint forwards to an uncapped paid key (the runaway-bill / DoW "denial of wallet"
scenario) and there is no spend cap behind it (see check 4).

**Not a finding when:** the project enforces the limit at the edge
(Vercel/Cloudflare/API gateway) — the universal `known-safe-patterns.md` "Missing rate
limiting" suppressor still applies for *ordinary* endpoints. This check is narrower: it is
about **cost-bearing** endpoints specifically, where "the edge probably handles it" is not
good enough without evidence.

---

## 3. Sensitive provider calls reachable from the frontend — CRITICAL

In a browser or mobile build, **an env var is not a secret.** `NEXT_PUBLIC_*`,
`VITE_*`, and `EXPO_PUBLIC_*` values are inlined into the shipped JS/IPA bundle at build
time and readable by anyone. Any call to a provider that bills, sends, or grants access
(OpenAI/Anthropic/Gemini, Stripe secret key, Resend/SendGrid, Twilio, AWS S3 signing,
Supabase `service_role`) must originate from a server (Route Handler, edge function, RPC),
never the client.

**What I flag:** a paid/privileged provider SDK or `fetch` initialized in a client
component / React Native screen with its key sourced from a `NEXT_PUBLIC_`/`VITE_`/
`EXPO_PUBLIC_` (or hardcoded) value; a `'use client'` file importing a server-only SDK.

**Three elements:**
- **Where:** the client file + the key reference (cite the import and the env access).
- **Trigger:** the attacker opens devtools / unzips the bundle, reads the key, and calls
  the provider directly on the victim's account.
- **Why-not-prevented:** the build inlines the value; the `NEXT_PUBLIC_`/`EXPO_PUBLIC_`
  prefix *guarantees* client exposure — it is not an accident the bundler can fix.

**Fix:** move the call behind a server route; expose only a thin authenticated endpoint to
the client; keep the real key in a non-`PUBLIC` server env var.

**Severity = CRITICAL** — same bar as the rubric's "service role / admin key exposed to
the client bundle."

**Not a finding when:** the key is genuinely a publishable/anon key designed for the
client (Stripe **publishable** `pk_`, Supabase **anon** key, a Mapbox public token,
PostHog project key). Those are public by design — do not flag. The line is
*publishable-by-design* vs *secret-leaked-via-PUBLIC-prefix*; this matches
`known-safe-patterns.md` (`SUPABASE_SERVICE_ROLE_KEY` in Route Handlers is fine; the same
key in a `NEXT_PUBLIC_` var is the bug).

---

## 4. Missing budget cap / spend alert on metered usage — MEDIUM

AI/usage-metered features without a hard spend ceiling turn a bug or an abuser into a
five-figure surprise bill. This is the review-time hook for the same risk the
gcp-cost-monitor killswitches cover at the infra layer — here I check that **the code
path that meters spend has a cap or alert**, not just that infra exists.

**What I flag:** a diff that adds or expands a metered call (per-token LLM, per-image gen,
per-email) with no per-user/global usage ceiling in the code path and no documented
provider-side budget cap or alert.

**Three elements:**
- **Where:** the metered call site + the absence of a guard before it.
- **Trigger:** abuse (see checks 1–3) or an accidental loop drives usage far past expected
  volume.
- **Why-not-prevented:** nothing in the path checks cumulative spend/usage; there is no
  cap row, no provider hard limit, no alert wired up.

**Fix:** add a per-user and/or global usage ceiling checked before the metered call
(reuse the check-2 counter), AND confirm a provider-side hard budget/alert exists (e.g.
the GCP killswitch / Gemini monthly cap pattern, or an OpenAI usage limit). Surface this
as a finding the user must acknowledge, not a silent pass.

**Severity = MEDIUM.** Real, more-than-cosmetic impact under a realistic edge case
(runaway usage / abuse), but it is not itself a directly-exploitable code vulnerability the
way checks 1 and 3 are — it is the missing backstop. Escalate to **HIGH** only if the diff
also removes an existing cap.

---

## Quick severity map

| Check | Default severity | Escalates to |
|---|---|---|
| 1. Entitlement/limit column on user-editable RLS table | CRITICAL | — |
| 2. Cost-bearing endpoint, no backend per-user+IP limit | HIGH | CRITICAL (gates an uncapped paid key) |
| 3. Paid/privileged provider key reachable from frontend | CRITICAL | — |
| 4. Metered usage with no budget cap / alert | MEDIUM | HIGH (diff removes an existing cap) |

Deliberately scoped to what check-point did NOT already cover. Generic SQL injection,
service-role-key-in-bundle, webhook signature verification, unverified `jwt.decode()`, and
`dangerouslySetInnerHTML` are already handled by `rules.md`, `reference/severity-rubric.md`,
and `reference/known-safe-patterns.md` — not repeated here.
