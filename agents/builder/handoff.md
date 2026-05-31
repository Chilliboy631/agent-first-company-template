# Handoff — builder

## Project location
C:\ClaudeProjects\farmflow

## Currently in flight
Phases 1 + 2 done. Up next: start replacing the demo engine surface by
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
