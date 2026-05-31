# Handoff — builder

## Project location
C:\ClaudeProjects\farmflow

## 📨 INBOX FROM RUNNER (2026-05-31) — read before "Currently in flight"
I verified Phase 1-2 live against `farmflowV1`. Schema + historical integrity
are solid and pass. **But Phase 2 is NOT done — auth has a launch blocker.**
Full evidence in `agents/runner/handoff.md`. Tasks for you, in order:

- **#0 🔴🔴 BLOCKER — fix infinite recursion in the `organization_members`
  RLS SELECT policy.** Confirmed LIVE: any authenticated read of
  `organization_members` OR `organizations` fails with
  `infinite recursion detected in policy for relation "organization_members"`.
  Cause: the members SELECT policy's USING clause sub-selects from
  `organization_members` itself; the `organizations` policies depend on that
  same subquery, so they recurse too. Effect: after login the dashboard can't
  read the user's org/membership — the whole authenticated app 500s.
  Fix (per docs/rls.md): put the membership lookup in a SECURITY DEFINER
  helper (e.g. `current_user_org_ids()` or `is_org_member(org_id)`) and
  reference THAT in the policies instead of self-selecting. The own-row branch
  (`user_id = auth.uid()`) is fine. After the fix, EVERY table's policy uses the
  same `organization_id IN (SELECT ... FROM organization_members ...)` pattern —
  so re-test reads on blocks/rate_types/logs/etc. too. Tell me when it's in and
  I'll re-run the live impersonation read test.
- **#1 🔴 signup creates no `profiles` row.** Confirmed live: after a real
  `create_organization`, org + owner-membership exist but `profiles` is empty.
  Add a `handle_new_user` trigger on `auth.users`, or insert the profile in
  `signupAction`.
- **#2b 🟠 Supabase Auth rejects signups as "Email address is invalid"** for
  `@example.com` and `.test` domains — a GoTrue email restriction is on. Before
  launch, confirm a real address (e.g. gmail) actually signs up via the form,
  or check the Auth → email settings. (This blocked me from a pure anon-key
  end-to-end signup; I proved the create_org/RLS path via SQL impersonation
  instead.)
- **#2 🟠 email-confirmation path has no org creation.** `signupAction` only
  calls `create_organization` when a session exists (confirmation OFF). If Ross
  turns confirmation ON, confirmed users log in via `loginAction` (which creates
  no org) and land on /dashboard with no org/membership. Decide the setting; if
  ON, add org creation to the post-confirmation path.
- **#3 🟠 (for the Logs work below)** the rate/price lookup fns are
  service_role-ONLY and there is NO service-role key/client yet (`.env.local`
  has it commented out). You'll need a service-role key in env + a service-role
  Supabase client before Logs Server Actions can resolve rates — otherwise the
  RPCs fail permission-denied.

Do #0 first (it's a launch blocker), then #1. The Logs migration below should
wait until #0/#1 land, since it builds on a working authenticated read path.

## Currently in flight
~~Phases 1 + 2 done.~~ Phase 1 done; **Phase 2 blocked on #0/#1 above
(runner verification, 2026-05-31).** After those are fixed and re-verified:
start replacing the demo engine surface by
surface, beginning with the Logs page (`app/(app)/logs/page.tsx`). Use
`types/database.generated.ts` for any new real-data code; leave the
hand-written `types/database.ts` alone (the demo engine still uses it).

For each surface migration:
1. Build a Supabase-backed Server Component / Server Action that reads
   the same shape the demo provided.
2. Snapshot rate_amount / total_cost / unit_price on every log insert.
3. Always go through `get_active_rate_history` / `get_active_input_price_history`
   for rate resolution — these are now service_role-only, so call them
   from Server Actions, not from the client.
4. Keep the demo engine working until that specific surface is fully cut over.

## What just happened
- Schema migrated to `farmflowV1` (project id `rokhyjnorqczidxyzarm`) via
  the Supabase MCP connector — not via the SQL editor.
- Hardening migration applied: per-table per-operation RLS policies for
  all 13 tables, views recreated with `security_invoker = true`,
  `search_path` pinned on functions, EXECUTE revoked from anon +
  authenticated on the SECURITY DEFINER rate/price lookup functions.
- `create_organization(name)` SECURITY DEFINER function added — this is
  the signup primitive; replaces the original permissive
  `Authenticated users can create organizations` INSERT policy.
- Real Supabase auth wired in `app/login/` and `app/signup/` via Server
  Actions (`useActionState` + `useFormStatus`). Signup calls
  `create_organization` after `signUp`. Typecheck passes.
- Supabase-generated types written to `types/database.generated.ts`.
  The hand-written `types/database.ts` is untouched (demo engine
  depends on it).

## Open questions
- Email confirmation: signup currently surfaces a friendly "check your
  email" message if Supabase returns no session post-signup. Whether
  the project has email confirmation on/off is a Supabase Auth setting
  Ross should pick before launch — not blocking now.
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
  service_role. Call them from Server Actions, not from browser code.
- `create_organization(p_name)` is the ONLY supported way to create an
  org — it inserts the org row and the owner membership atomically.
- The demo engine (`lib/demo/engine.tsx`) must keep working until each
  surface is replaced one at a time.
- Supabase project: `farmflowV1` (id `rokhyjnorqczidxyzarm`,
  eu-central-1). `.env.local` already points at it.
