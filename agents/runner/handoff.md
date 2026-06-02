# Handoff — runner

## Project location
C:\ClaudeProjects\farmflow
Supabase project: farmflowV1 (id `rokhyjnorqczidxyzarm`, eu-central-1)

## RATE TYPES LIVE-VERIFY (2026-06-02) — ✅ DB-LAYER GREEN (assigned by lead)
Verified the Rate Types real-data surface against farmflowV1. The DB is no
longer empty: Ross's live walkthrough left **1 user / 1 org / 1 member / 2
rate_types ("Long","mechanic") / 1 rate_history (mechanic R900 eff 2026-06-02)**
— i.e. he already exercised the real UI write path. Findings:
- **#2 createRateType persists ✅** — both real rate_types rows have
  org_matches_member=true AND created_by=the real user. Org derived server-side
  (requireOrg), not client input.
- **#3 addRateVersion ✅** — the real rate_history row carries correct org +
  created_by. Page reads `rate_history ORDER BY effective_from DESC` →
  newest-first confirmed in `app/(app)/rate-types/page.tsx:23`.
- **#4 append-only ✅** — live attempts BOTH blocked by
  `prevent_rate_history_mutation`: UPDATE→"rows are immutable", DELETE→"cannot
  be deleted… permanent audit records". rate_history also has no UPDATE/DELETE
  RLS policy (INSERT+SELECT only).
- **#5 cross-tenant isolation ✅** — impersonation test (real owner vs a genuine
  2nd signup in its own org, all rolled back via final RAISE, zero residue
  reconfirmed 1/1/1/2/1): owner_sees=2, stranger_total=0, stranger_sees_orgA=0.
  RLS via `current_user_org_ids()` — no recursion.
- **#1 empty-state UI — NOT verifiable here.** The live org now has data, so the
  fresh-org empty state can't be observed on it. Needs a fresh-org browser
  session. Populated render is implied working (Ross created the data via the
  real UI). The only outstanding gap is browser-level: empty state + the
  add-version affordance clarity Ross flagged.
Net: Rate Types surface is functionally correct at the data layer. Cleared from
runner's side pending the cosmetic/UI browser pass.

### ⏭️ RESUME HERE NEXT SESSION (paused 2026-06-02 EOD)
Rate Types DB-layer verify is DONE. One open item, browser-only:
- Drive a real logged-in `next dev` session and confirm: (a) a FRESH org shows
  the Rate Types empty state, (b) versions render newest-first on-screen, (c)
  there is NO edit/delete path on existing versions (append-only by UI), and (d)
  the add-version affordance is clear (the clarity item from Ross's UX triage).
- Note for lead: this overlaps builder's open add-version-clarity inbox item —
  suggested folding the empty-state check into that rather than a separate pass.
  Awaiting lead's call before doing the browser run.
- After that: next surface to verify is **Blocks** (builder's NEXT build step),
  not yet built/handed to runner. Nothing to do until builder ships it.

## RE-VERIFY (2026-05-31, after builder's fixes) — ✅ ALL CLEAR
Builder fixed #0/#1/#2; I re-verified live against farmflowV1 (impersonation,
rolled back, 0 residue):
- **#0 RLS recursion FIXED.** `current_user_org_ids()` exists (SECURITY DEFINER,
  EXECUTE = authenticated only). NO policy self-selects from
  organization_members anymore. Authenticated reads of organization_members /
  organizations / blocks all succeed — no recursion. Cross-tenant isolation
  proven: user A sees its own 1 org/1 block/1 member and 0 of user B's.
- **#1/#2 profiles + org provisioning FIXED.** `on_auth_user_created` trigger
  on auth.users + `handle_new_user()` (SECURITY DEFINER, trigger-only — EXECUTE
  revoked from anon+authenticated). Inserting an auth user with
  `raw_user_meta_data.farm_name` auto-created profile + org + owner membership
  at signup time. Confirmed live (profA=1, role=owner).
- **STILL OPEN (not code bugs, need Ross):**
  - #2b email-domain rejection — Supabase Auth project setting. Ross must check
    Auth → Providers → Email and confirm real domains (gmail) can sign up.
  - #3 service-role key — `SUPABASE_SERVICE_ROLE_KEY` must land in `.env.local`
    + a service-role client in lib/supabase/ before Logs work resolves rates.
    Builder will add the helper when starting Logs.
- **NOT verified (needs browser):** real UI signup/login click-through,
  middleware redirects, demo-layer regression. Substance is proven at the DB
  layer; the gap is purely the rendered-page path.
Phase 1-2 backend: GREEN. Builder cleared to start Logs once Ross supplies
the service-role key (#3).

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
- **RLS policies** — per-table per-operation (matches docs/rls.md shape).
  `rate_history` + `input_price_history` are append-only at policy level too
  (INSERT + SELECT only; no UPDATE/DELETE policy). labour_logs/input_logs
  have full CRUD. audit_log SELECT-only. NOTE: structurally present ≠ correct —
  the `organization_members`/`organizations` SELECT policies have a recursion
  bug that only shows at query time (see #0). Other tables' policies depend on
  the same `organization_id IN (SELECT ... FROM organization_members ...)`
  pattern, so they may recurse too once data is read — RE-TEST after #0 is fixed.
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

- ✅ `create_organization` (the signup primitive) RUNS without error for an
  authenticated user (execution reached later steps; it bypasses RLS as
  SECURITY DEFINER, so the org + owner-membership writes happen).
- 🔴 **CRITICAL: authenticated reads of `organization_members` AND
  `organizations` fail with `infinite recursion detected in policy for
  relation "organization_members"`.** CONFIRMED LIVE under impersonation
  (role=authenticated + JWT sub). See #0 — this breaks the whole logged-in app.
- ❌ **profiles row NOT created** — confirmed live (`profiles` read returned
  count=0 right after create_organization; profiles has no recursion, so this
  read succeeded and is trustworthy). See #1.
- ⚠️ **Anon-key signUp is REJECTED by Supabase Auth with "Email address is
  invalid"** for `@example.com` AND a `.test` domain. GoTrue-level email
  restriction (not format). Did NOT register a real external address. See #2b.
- service-role-only lookup denied to authenticated: expected, but NOT cleanly
  re-confirmed this run (earlier apparent confirmation was buffered-output
  replay). The grants query (session 1) proved anon+authenticated lack EXECUTE,
  which is the real evidence — treat the grant check as the source of truth.
- Login correct→session / wrong→error: UNTESTED live (signUp blocked by 2b
  before login could run). `loginAction` is a thin wrapper over
  `signInWithPassword`; low risk but unverified.
- ⏸️ middleware redirects: CODE-verified only, not click-verified.
- ℹ️ `app/dashboard/page.tsx.disabled` is inert — no route conflict with
  `app/(app)/dashboard`. Real dashboard is the (app) one.

### CORRECTION (honesty note)
An earlier commit (29f410e) claimed the RLS read path was "CONFIRMED live
(org_visible=1, mem_visible=1)". That was WRONG — that query had errored with
the recursion bug before returning any rows; I misattributed buffered output.
Retracted and corrected here. The real result is #0 below.

## BUGS / FINDINGS FOR BUILDER (in priority order)
0. **🔴🔴 BLOCKER — infinite recursion in `organization_members` RLS policy.**
   CONFIRMED LIVE. The SELECT policy "Users can select members of their
   organizations" has USING:
     `(user_id = auth.uid()) OR (organization_id IN
        (SELECT organization_id FROM organization_members
         WHERE user_id = auth.uid()))`
   The subquery selects from `organization_members` *inside* the
   `organization_members` policy → Postgres infinite recursion. And the
   `organizations` SELECT/UPDATE/DELETE policies all do
   `id IN (SELECT organization_id FROM organization_members WHERE ...)`, so
   reading `organizations` hits the recursive members policy too.
   IMPACT: every authenticated read of `organizations` or `organization_members`
   500s. After login the dashboard cannot load the user's org/membership at
   all — the entire authenticated app is non-functional. (Demo layer unaffected;
   it doesn't touch Supabase.)
   FIX (per docs/rls.md guidance): move the membership lookup into a SECURITY
   DEFINER helper, e.g. `is_org_member(org_id uuid)` /
   `current_user_org_ids()`, and reference THAT in the policies instead of a
   self-select. The own-row branch (`user_id = auth.uid()`) is fine; it's the
   cross-member subquery that must not read the same table under RLS.
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
