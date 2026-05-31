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

## BUGS / FINDINGS FOR BUILDER (in priority order)
1. **🔴 signup never creates a `profiles` row.** There is NO trigger on
   `auth.users`, and `create_organization` only inserts `organizations` +
   `organization_members`. The checklist requires a profiles row on signup.
   Fix: add a `handle_new_user` trigger on auth.users, or insert the profile
   in `signupAction`.
2. **🟠 email-confirmation path has no org creation.** `signupAction` only
   calls `create_organization` when a session exists (i.e. confirmation OFF).
   If Supabase email confirmation is ON, the confirmed user signs in via
   `loginAction`, which creates no org → they reach /dashboard with no
   organization/membership and every org-scoped query is empty. Decide the
   confirmation setting; if ON, add org-creation to the post-confirmation path.
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
- Auth end-to-end (signup→user/org/member/profile rows, login→/dashboard,
  wrong creds→error, unauth→redirect). The anon key is real, so this IS now
  testable — needs a dev-server run + form submit (writes real auth rows).
  Offered to Ross as the next step.
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
