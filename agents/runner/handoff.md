# Handoff — runner

## Project location
C:\ClaudeProjects\farmflow
Supabase project: farmflowV1 (id `rokhyjnorqczidxyzarm`, eu-central-1)

## Last session (2026-05-31)
Ran the Phase 1-2 verification checklist against the live `farmflowV1`
project after builder shipped schema migration + auth wiring.

### VERIFIED PASS (direct DB access via Supabase MCP)
- **Schema structure** — all 13 tables exist, RLS enabled on every one.
- **Functions** — `get_active_rate_history`, `get_active_input_price_history`,
  `require_active_rate`, `create_organization` all present, SECURITY DEFINER,
  `search_path=public` pinned. Trigger fns present.
- **Immutability triggers** — `trg_rate_history_immutable` and
  `trg_input_price_history_immutable` fire BEFORE UPDATE *and* DELETE.
  `updated_at` triggers on the mutable tables.
- **RLS policies** — per-table per-operation (matches docs/rls.md).
  `rate_history` + `input_price_history` are append-only at policy level too
  (INSERT + SELECT only; no UPDATE/DELETE policy). labour_logs/input_logs
  have full CRUD. audit_log SELECT-only.
- **Indexes** — `idx_rate_history_lookup`, `idx_input_price_lookup` present.
- **Views** — `v_blocks_flat`, `v_labour_costs` both `security_invoker=true`.
- **Function grants** — lookups (`get_active_*`, `require_active_rate`) are
  service_role-ONLY (revoked from anon + authenticated). `create_organization`
  is callable by authenticated (correct — it's the signup primitive).
- **HISTORICAL INTEGRITY (live functional test, rolled back, zero residue —
  org_count still 0 after):**
  - as-of lookup returns the correct version for the work date
  - adding a newer rate (R2 @ later date) does NOT change the older date's
    resolved rate — D1 still resolves R1
  - labour_log snapshot (rate_amount/total_cost = 50/400) unchanged after R2
  - input-price parallel behaviour correct
  - `require_active_rate` raises when no rate configured
  - UPDATE amount / DELETE on rate_history BLOCKED; UPDATE on
    input_price_history BLOCKED; notes-only update ALLOWED
- **Typecheck** — `npx tsc --noEmit` exits 0.
- **Auth code wiring** — login/signup Server Actions present; middleware
  protects /dashboard,/rate-types,/blocks,/logs,/reports and bounces
  logged-in users off /login,/signup.
- **Env keys** — `.env.local` has a REAL anon key (decodes to role `anon`,
  ref `rokhyjnorqczidxyzarm`) and the correct project URL. (Earlier
  "placeholder keys" note was a misread — retracted.) Service-role key is
  NOT set (commented out: "optional for now").

### AUTH END-TO-END (live, against farmflowV1 — two methods, all rolled back / 0 residue)
Method A: replicated the Server Actions' calls via @supabase/supabase-js + the
real anon key (auth.signUp → rpc create_organization → signInWithPassword).
Method B: SQL impersonation — set role `authenticated` + a JWT `sub` claim,
called `create_organization`, then read back through RLS. Both inside rollback;
final counts users/orgs/members/profiles all 0.

- ✅ `create_organization` (the signup primitive) works for an authenticated
  user: creates the org + an `organization_members` row with role=`owner`,
  and the user can SEE both through RLS (org_visible=1, mem_visible=1,
  role=owner). RLS read path CONFIRMED live.
- ❌ **profiles row NOT created** — confirmed live (profiles_for_user=0 right
  after create_organization). See #1.
- ✅ service-role-only lookup denied to authenticated (RLS/grants enforced).
- ⚠️ **Anon-key signUp is REJECTED by Supabase Auth with "Email address is
  invalid"** for `@example.com` AND a `.test` domain. This is GoTrue-level
  email validation/restriction on the project (not format). Couldn't complete
  a pure end-to-end signup via the anon key without registering a real
  external address — did NOT do that. See #2b. The create_organization +
  RLS substance was proven via impersonation instead.
- Login correct→session and wrong→error: NOT re-confirmed this run (the
  earlier apparent run was buffered-output replay; the real signUp errored on
  email validation before login could be reached). Logic in `loginAction` is
  a thin wrapper over `signInWithPassword`; low risk but UNTESTED live.
- ⏸️ middleware redirects: CODE-verified only, not click-verified.
- ℹ️ `app/dashboard/page.tsx.disabled` is inert — no route conflict with
  `app/(app)/dashboard`. Real dashboard is the (app) one.

## BUGS / FINDINGS FOR BUILDER (in priority order)
1. **🔴 signup never creates a `profiles` row.** CONFIRMED LIVE (profiles
   empty immediately after a real create_organization call). No trigger on
   `auth.users`; `create_organization` only writes org + member.
   Fix: add a `handle_new_user` trigger on auth.users, or insert the profile
   in `signupAction`.
2. **🟠 email-confirmation path has no org creation.** `signupAction` only
   calls `create_organization` when a session exists (i.e. confirmation OFF).
   If Supabase email confirmation is ON, the confirmed user signs in via
   `loginAction`, which creates no org → they reach /dashboard with no
   organization/membership and every org-scoped query is empty. Decide the
   confirmation setting; if ON, add org-creation to the post-confirmation path.
   (Could not auto-detect the setting live — signUp was blocked by 2b before a
   session could be observed.)
2b. **🟠 Supabase Auth rejects signups as "Email address is invalid."** Seen
   live for `@example.com` and `@*.test`. Some GoTrue email restriction is on
   (domain allow/blocklist or strict validator). Ross/builder must confirm
   real signups (e.g. gmail) actually succeed via the live signup form before
   launch — otherwise real users may be blocked. Needs a browser test with a
   real address, or a check of the Auth → Providers/email settings.
3. **🟠 Logs-surface landmine.** `get_active_rate_history` /
   `get_active_input_price_history` / `require_active_rate` are
   service_role-ONLY, but `lib/supabase/` has only anon-key clients
   (server.ts = anon key + cookies = `authenticated` role) AND no service-role
   key is set in `.env.local` yet. Before the Logs Server Actions can resolve
   rates, builder must (a) get a service-role key into the env and (b) add a
   service-role Supabase client. Otherwise the RPC calls fail permission-denied.
4. **ℹ️ Advisor WARN (accepted):** `create_organization` SECURITY DEFINER is
   callable by authenticated. By design — it checks `auth.uid()` internally
   and is the signup primitive. No action needed.

## NOT YET VERIFIED
- A full UI signup with a REAL email through a running `next dev` (blocked by
  2b without registering a real external address). This is the one gap left in
  auth — would confirm: the email validator accepts real domains, the
  confirmation on/off setting, login→/dashboard, and middleware redirects, all
  through the rendered pages.
- Login wrong-creds→error and correct→/dashboard, live (logic-reviewed only).
- Middleware redirects via a real browser (code-verified only).
- Demo-layer regression (loads, banner, add log, rate resolution, reports) —
  not run; needs a browser session. Demo engine uses localStorage (no
  Supabase) so it should be unaffected, but unconfirmed.

## Don't lose this
- Historical integrity verification must re-run after every log-related
  change. The DO-block test I used asserts then RAISEs `VERIFY_PASS_ALL_CHECKS`
  to roll everything back — leaves no residue. Reuse it.
- rate_history cannot be DELETEd (trigger) — so an org with rate history can't
  be cascade-deleted normally. Keep test data out of the real project or use
  the rollback pattern.
- This template lives in its OWN nested git repo (under
  `_Agents/agent-first-company-template`). The farmflow root repo does NOT
  track `_Agents/`. Commit handoffs from inside the template repo. It has no
  remote configured (local-only; humans relay).
