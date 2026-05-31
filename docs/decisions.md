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

(Add new decision entries above this line. Do not delete this template
section; it's the reference for how new entries should look.)
