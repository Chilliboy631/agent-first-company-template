# Decisions

Append-only log of decisions that change direction. Newest first.
Every entry has a date, who decided, what changed, why, and what
surfaces it affects. Reference the relevant commit if applicable.

This log is read on wake-up: any agent reads entries newer than their
last handoff so they're current on direction changes before acting.

## Template

When adding a new entry, copy this shape:

```markdown
## YYYY-MM-DD — Short summary

**Decided by:** Name(s)
**Commit:** abc1234 (if applicable)
**Status:** active / superseded / rolled back

**What changed:**
One or two sentences on the substantive change.

**Why:**
The reasoning — what evidence prompted this, what alternatives were
weighed, why this call.

**Affects:**
Which surfaces / agents / files this changes for.
```

---

## 2026-05-31 — Schema migrated to farmflowV1 + real auth wired

**Decided by:** Ross + builder
**Status:** active

**What changed:**
Applied the core schema to the `farmflowV1` Supabase project via the MCP
connector (not the SQL editor). Followed with a cleanup migration that
adds per-table per-operation RLS policies, recreates the two views with
`security_invoker = true`, pins `search_path` on all functions, and
revokes EXECUTE on `get_active_*` + `require_active_rate` from anon and
authenticated (server-side / service_role only). Replaced the original
permissive `organizations` INSERT policy with a `create_organization(name)`
SECURITY DEFINER function that atomically creates the org + assigns the
caller as `owner`. Wired real Supabase auth: `/login` and `/signup` use
Server Actions; signup calls `create_organization` after `signUp`.

**Why:**
- MCP tool was available, so guiding the user through copy-pasting into
  the SQL editor would have been needless friction.
- Per-table per-operation policies are required by `docs/rls.md` (the
  one-line `FOR ALL` policies the original schema shipped with violated
  the rules).
- An atomic SECURITY DEFINER function is the cleanest signup primitive
  and removes the `WITH CHECK (true)` advisor warning.

**Affects:**
- Supabase project `farmflowV1` (id `rokhyjnorqczidxyzarm`) now holds
  the full schema + RLS + functions.
- `types/database.generated.ts` (new) is the Supabase-generated type
  source for all real-data code. Hand-written `types/database.ts` stays
  for the demo engine until each surface is replaced.
- `app/login/`, `app/signup/` (page + new `actions.ts`) — real auth.
- Next builder step: start replacing the demo engine surface by surface,
  beginning with Logs.

## 2026-05-31 — RLS recursion fix + signup via auth.users trigger

**Decided by:** Builder, after runner verification flagged blockers
**Status:** active

**What changed:**
1. Introduced `public.current_user_org_ids()` (SECURITY DEFINER, STABLE,
   SETOF uuid) and recreated every org-scoped RLS policy to look up
   memberships through it. This breaks the infinite-recursion path where
   `organization_members` policies sub-selected from `organization_members`.
2. Moved signup-time provisioning (profile + organization + owner
   membership) out of `signupAction` and into an
   `on_auth_user_created AFTER INSERT ON auth.users` trigger that calls
   `public.handle_new_user()`. The trigger reads `farm_name` from
   `auth.users.raw_user_meta_data`. This fires regardless of whether email
   confirmation is on, so the org is provisioned at the moment of signup.
3. `signupAction` now just calls `supabase.auth.signUp` with
   `options.data.farm_name` and shows a friendly "check your email"
   message when no session is returned.
4. Revoked EXECUTE on `handle_new_user()` from PUBLIC/anon/authenticated
   (trigger-only, not RPC-callable).

**Why:**
- Runner verified that the original Phase-2 policies caused
  `infinite recursion detected in policy for relation "organization_members"`
  on any authenticated read of `organization_members` or `organizations`,
  which would 500 the whole authenticated app. SECURITY DEFINER helper is
  the standard Supabase-recommended fix.
- Runner verified that signup created an org + membership but no profile
  row, and that the confirmation-on path would skip org creation entirely.
  An `auth.users` trigger fixes both with one mechanism and removes a
  whole class of "what if confirmation is on" branching from the app code.

**Affects:**
- Every RLS policy on `organizations`, `organization_members` (SELECT
  only), `blocks`, `rate_types`, `rate_history`, `workers`, `activities`,
  `input_resources`, `input_price_history`, `labour_logs`, `input_logs`,
  `audit_log`.
- `app/signup/actions.ts` + `app/signup/page.tsx` (now handles
  `{info: string}` state alongside `{error: string}`).
- Future: any new policy that needs caller-org filtering MUST use
  `current_user_org_ids()` — sub-selecting `organization_members`
  directly will recurse.

## 2026-06-01 — Reorder roadmap: master-data surfaces before Logs

**Decided by:** Ross + builder
**Status:** active

**What changed:**
Reversed the "Logs first" surface-migration order. Real DB-backed CRUD for
the master-data surfaces ships first, in dependency order — Rate Types
(+ rate versions) → Blocks → Activities → Input Resources (+ price
versions) → Workers — and Logs (labour + inputs) ships LAST.
Also: new orgs start blank for farm-specific data (blocks, workers, actual
rates) but will be offered optional, fully-editable starter defaults for
generic scaffolding (common activities, units, starter rate-type names) at
onboarding time. The demo seed in `lib/demo/engine.tsx` (the "Acacia"
farm) is throwaway demo content, NOT a production seed.

**Why:**
- The live DB is empty (0 rows everywhere; runner deleted the synthetic
  auth test data). A real, multi-tenant client builds its own rate types,
  blocks, workers, etc. — there is no universal seed.
- Logs is the most *dependent* surface, not the most independent: a
  labour log requires a worker → which references a rate type → which
  needs ≥1 rate version, plus a block and an activity; an input log needs
  a resource with a price version. "Logs first" only ever worked because
  the demo seed was faking all of that underneath it.
- Therefore Logs must come last. Building it first would mean either
  seeding throwaway data into real orgs or shipping empty, unverifiable
  dropdowns.

**Affects:**
- Build order for all remaining surface migrations. Supersedes the
  "Logs first" instruction in `lead/handoff.md`, `builder/handoff.md`,
  and `builder/CLAUDE.md` "Current Task Context".
- Next builder step: real DB-backed Rate Types surface
  (`app/(app)/rate-types/page.tsx`), append-only rate versions via
  `rate_history` (no UPDATE/DELETE — enforced at trigger + policy level).
- Onboarding/starter-defaults work is deferred until the master CRUD
  surfaces exist.

(Add new decision entries above this line. Do not delete this template
section; it's the reference for how new entries should look.)
