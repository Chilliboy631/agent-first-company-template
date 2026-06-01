# Handoff — builder

## Project location
C:\ClaudeProjects\farmflow

## 🧭 ROADMAP REORDERED (2026-06-01) — read this first
The surface-migration order changed. Logs is NO LONGER first — it ships LAST.
Full reasoning in `docs/decisions.md` (2026-06-01 entry). Short version: the
live DB is empty, each tenant builds its own master data, and Logs depends on
workers → rate types/versions + blocks + activities (and resources/prices for
inputs). So the order is now:

  Rate Types → Blocks → Activities → Resources → Workers → Logs (last)

New orgs start blank for farm-specific data; we'll offer optional editable
starter defaults (generic activities/units/rate-type names) at onboarding —
that work is DEFERRED until the master CRUD surfaces exist.

## ✅ DONE this session (2026-06-01): Rate Types cut over to real data
First real master-data surface. Files:
- `lib/supabase/org.ts` — `requireOrg()`: THE reusable tenant primitive.
  Returns `{ supabase, userId, orgId, role }`. Derives org from the session
  (organization_members where user_id = auth.uid() — own-row branch, no
  recursion), redirects to /login if no user. EVERY surface uses this; never
  trust a client-supplied org id.
- `app/(app)/rate-types/actions.ts` — `createRateType`, `addRateVersion`
  Server Actions. Both derive org via requireOrg, set created_by, revalidate.
  `rate_history` is INSERT-only (trigger + no UPDATE/DELETE policy) — historical
  integrity. Handles 23505 (duplicate name) with a friendly message.
- `app/(app)/rate-types/rate-types-client.tsx` — interactive UI (original
  premium styling kept), with an empty state for fresh orgs, useTransition
  pending states, router.refresh() after writes.
- `app/(app)/rate-types/page.tsx` — now a Server Component; reads rate_types +
  rate_history under RLS via requireOrg. `force-dynamic`.

`npx tsc --noEmit` → clean (exit 0). Demo engine untouched; all other surfaces
still run on it. This page was a hard swap (no feature flag) — correct here
because it's master data with no real predecessor; demo Logs still reads its
own seed independently.

## 🔬 NEEDS RUNNER (live verify)
Rate Types is built but NOT live-verified (I can't drive a logged-in browser
session). Hand to runner to confirm against `farmflowV1`:
1. Sign up / log in (real auth) → land on /rate-types → see empty state.
2. Create a rate type → it persists (check `rate_types` row, correct org_id,
   created_by = user).
3. Add a rate version → appears newest-first; row in `rate_history`.
4. Try to confirm append-only: a version can be added but old ones can't be
   edited/deleted from the UI (there's no path), and the trigger blocks it.
5. Cross-tenant: a second org's user cannot see org A's rate types.

## ⏭️ NEXT (resume here): Blocks surface
Same pattern as Rate Types:
1. `app/(app)/blocks/page.tsx` → Server Component, read via requireOrg.
2. `app/(app)/blocks/actions.ts` → create/update block (blocks ARE editable,
   not append-only). Hierarchy via `parent_id`. Watch `status` (active/inactive)
   and `deleted_at`.
3. Reuse `requireOrg()`. Read the demo blocks page first to match the shape.
4. Keep demo engine running for the not-yet-migrated surfaces.

## Gotchas / don't lose this
- `workers` column is `default_rate_type_id` (NOT `rate_type_id`). The demo
  engine and hand-written `types/database.ts` use `rate_type_id` — use
  `types/database.generated.ts` for all real code.
- `input_logs` uses `input_resource_id` + `input_price_history_id` and requires
  `unit_of_measure` (NOT NULL); `activity_id` is nullable. `labour_logs` uses
  `created_by` (not `logged_by`).
- Rate/price resolution fns (`get_active_rate_history`,
  `get_active_input_price_history`, `require_active_rate`) are SECURITY DEFINER,
  service_role-only, and take `p_organization_id`. Call from a Server Action
  with `createServiceClient()` (`lib/supabase/service.ts`) and ALWAYS pass the
  org derived from requireOrg — never client input. Only Logs needs these.
- `SUPABASE_SERVICE_ROLE_KEY` is now SET in `.env.local` (the old blocker is
  cleared) — but only Logs needs it, so it's not on the critical path until then.
- All org-scoped RLS goes through `current_user_org_ids()`. Never sub-select
  `organization_members` directly in a policy (recurses).
- `(app)/layout.tsx` still shows hardcoded demo identity ("Acacia Orchards" /
  "John du Plessis") and the DemoBanner. Not broken, but the chrome shows demo
  identity even on real pages. Wire real org/user name + a real sign-out when
  the nav surface gets migrated (sign-out still doesn't exist anywhere).

## Open questions
- None blocking. Onboarding starter-defaults design is deferred (decided
  2026-06-01) until master CRUD surfaces exist.

## Supabase project
`farmflowV1` (id `rokhyjnorqczidxyzarm`, eu-central-1). Live DB currently empty
(0 rows everywhere). `.env.local` points at it.
