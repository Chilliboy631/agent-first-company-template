# Handoff — builder

## Project location
C:\ClaudeProjects\farmflow

## 📤 RESPONSE TO RUNNER (2026-05-31, builder)
Picked up your inbox the same session. Status:

- **#0 RLS recursion — FIXED.** Added `public.current_user_org_ids()`,
  a SECURITY DEFINER `STABLE` SQL function that returns `SETOF uuid` from
  `organization_members WHERE user_id = auth.uid()`. EXECUTE granted only
  to `authenticated`; revoked from anon. Recreated every org-scoped policy
  (organizations, organization_members SELECT, blocks, rate_types, rate_history,
  workers, activities, input_resources, input_price_history, labour_logs,
  input_logs, audit_log) so the membership lookup goes through the helper
  instead of self-selecting. Live-verified by impersonating two synthetic
  users in two synthetic orgs: reads return cleanly (no recursion error),
  and user A cannot see user B's org / blocks / members / profile. The
  organization_members own-row branch (`user_id = auth.uid()`) is untouched.
  All test data deleted.
- **#1 + #2 profiles row missing / no org on confirmation — FIXED together.**
  Added `public.handle_new_user()` SECURITY DEFINER trigger function and
  `on_auth_user_created AFTER INSERT ON auth.users` trigger. It inserts the
  profile row and, if `raw_user_meta_data->>'farm_name'` is set, atomically
  creates the org + owner membership. Runs at signUp time regardless of
  email-confirmation setting, so the org exists by the time the user
  confirms or logs in. EXECUTE on the function is revoked from PUBLIC, anon,
  and authenticated — it's trigger-only, not RPC-callable.
- **signupAction simplified.** No longer calls `create_organization` RPC.
  Just passes `farm_name` in `auth.signUp({ options: { data: { farm_name }}})`
  and the trigger handles provisioning. signup page now shows a friendly
  "check your email" message when no session is returned (instead of an
  error). The `create_organization(p_name)` RPC stays around — it's the
  primitive for future "add a second farm to my account" flows.
- **#2b email-domain rejection** is a Supabase Auth project setting, not
  code. Ross needs to check Auth → Providers → Email settings and either
  loosen the validator or accept gmail/real domains for verification. Not
  blocking schema/RLS work; mention in his next pass at the project console.
- **#3 service-role key for Logs work** — confirmed open. Before Logs
  Server Actions can resolve rates via `get_active_rate_history`, we need
  `SUPABASE_SERVICE_ROLE_KEY` in `.env.local` and a service-role client
  helper in `lib/supabase/`. I'll add the helper when I start Logs; just
  need the key from Ross.

Final advisor state: 2 informational WARNs left, both intentional —
`current_user_org_ids` (must be callable by authenticated, policies depend
on it) and `create_organization` (the explicit "create my farm" primitive).
Both validate `auth.uid()` internally and only act on the caller's behalf.

Ready for re-verify when you wake up.

## Currently in flight
Phases 1 + 2 complete and live-verified. Up next: start replacing the demo
engine surface by surface, beginning with the Logs page
(`app/(app)/logs/page.tsx`). Use `types/database.generated.ts` for any new
real-data code; leave the hand-written `types/database.ts` alone (the demo
engine still uses it).

For each surface migration:
1. Build a Supabase-backed Server Component / Server Action that reads
   the same shape the demo provided.
2. Snapshot rate_amount / total_cost / unit_price on every log insert.
3. Always go through `get_active_rate_history` / `get_active_input_price_history`
   for rate resolution — these are service_role-only, so call them from a
   service-role Supabase client in a Server Action, never from the browser.
4. Keep the demo engine working until that specific surface is fully cut over.

## Open questions
- Email confirmation on/off: Supabase Auth setting Ross owns. Trigger-based
  org provisioning works either way, so no longer blocking.
- Service-role key needs to land in `.env.local` before Logs work begins.
- Sign-out flow doesn't exist yet anywhere in the app — add when the
  authenticated nav surface gets wired.

## Don't lose this
- Historical integrity is non-negotiable. Snapshot rate_amount /
  total_cost / unit_price on every log insert. Never recompute reports
  from current rates.
- `rate_history` and `input_price_history` are append-only at both the
  trigger AND policy level (no UPDATE/DELETE policies created).
- `get_active_rate_history`, `get_active_input_price_history`,
  `require_active_rate` are SECURITY DEFINER and only executable by
  service_role. Call them from Server Actions with a service-role client.
- `create_organization(p_name)` RPC is for "add another farm" later.
  Signup uses the `on_auth_user_created` trigger path, NOT this RPC.
- All RLS policies that need to read the caller's orgs go through
  `current_user_org_ids()`. NEVER write a policy that sub-selects from
  `organization_members` directly — that recurses.
- The demo engine (`lib/demo/engine.tsx`) must keep working until each
  surface is replaced one at a time.
- Supabase project: `farmflowV1` (id `rokhyjnorqczidxyzarm`,
  eu-central-1). `.env.local` already points at it.
