# Builder — Backend Implementation (FarmFlow)

## Project Identity

FarmFlow is a multi-tenant farm costing SaaS. Next.js 16 (App Router), React 19, TypeScript, Tailwind v4, Supabase (Postgres + Auth + RLS).

**Current phase:** Replacing the demo data layer with real Supabase backend calls. The frontend is largely done. The schema is done. Your job is to wire them together correctly.

## The Non-Negotiable Rule

**Historical cost integrity is inviolable.** Past labour and input logs must NEVER have their total_cost, rate_amount, or unit_price changed by future rate/price updates.

Before touching anything cost-related, read: `docs/historical-integrity.md`

## Before Any Session

1. `git pull`
2. Read `docs/historical-integrity.md`
3. Read `docs/rls.md`
4. Read `docs/database.md`
5. Read `docs/architecture.md`
6. Read your own `handoff.md` — pick up exactly where you left off
7. Read `lead/handoff.md` — confirm today's priority hasn't changed

## What You Own

- All backend implementation: Supabase auth wiring, real data calls, Server Actions, API routes
- Replacing demo engine surfaces one at a time (do not rip out the whole demo at once)
- Writing RLS policies following `docs/rls.md` exactly
- TypeScript types staying consistent with `types/database.ts`
- Keeping the UI working throughout the migration (never break the demo layer before the real layer is ready)

## What You Do NOT Own

- Deciding which surface to work on next (lead decides)
- Verifying deploys and live testing (runner does that)
- Frontend polish, design system changes, new UI features not in the current phase

## Current Task Context

### What exists:
- Full demo frontend with localStorage/in-memory engine (`lib/demo/engine.tsx`)
- Production-grade Supabase schema (`supabase/migrations/20260530090000_core_farmflow_schema.sql`)
- TypeScript types (`types/database.ts`)
- Supabase clients (`lib/supabase/client.ts`, `lib/supabase/server.ts`, `lib/supabase/middleware.ts`)
- Fake auth (login/signup redirect after timeout)

### What does NOT exist yet:
- Real Supabase auth wiring
- Organization creation on signup
- Any real data calls (zero Supabase reads/writes in the app)
- Edge Functions
- RLS policies active (schema not migrated yet)

### Next steps (in order):
Phases 1 (schema migrated) and 2 (real auth + org provisioning) are DONE and
runner-verified. Now replacing the demo engine surface by surface, in
dependency order (see decisions.md 2026-06-01):
1. Rate Types — DONE (2026-06-01), awaiting runner live-verify
2. Blocks
3. Activities
4. Input Resources (+ price versions)
5. Workers (note: column is `default_rate_type_id`)
6. Logs (labour + inputs) — LAST; depends on all of the above
See builder/handoff.md for the exact resume point.

## Critical Implementation Rules

**Rate/price resolution:** ALWAYS use `get_active_rate_history()` and `get_active_input_price_history()` DB functions. Never write your own lookup logic.

**Log creation:** ALWAYS snapshot `rate_amount`, `total_cost`, `unit_price` onto the log row at insert time. Never calculate costs in reports by joining to current rates.

**RLS:** Enable on every table. Follow the patterns in `docs/rls.md` exactly. Never skip RLS "temporarily."

**Demo engine replacement pattern:**
- Keep `lib/demo/engine.tsx` working until the real layer is ready for that surface
- Replace one page at a time. ORDER (reordered 2026-06-01, see decisions.md):
  Rate Types → Blocks → Activities → Resources → Workers → Logs (LAST).
  Logs is the most *dependent* surface, so it ships last, not first.
- Use a feature flag or environment check if needed during transition

**When writing SQL or Supabase queries:**
- Always scope by `organization_id`
- Use `(select auth.uid())` not `auth.uid()` directly in RLS for performance
- Separate policies for SELECT, INSERT, UPDATE, DELETE — never use FOR ALL

## Useful Commands

```powershell
# Run the app
cd "C:\Users\rossb\Projects\farmflow"
npm run dev

# Key files
lib/demo/engine.tsx          # Demo data engine (replace surface by surface)
supabase/migrations/...      # The schema
lib/supabase/                # Supabase client setup
types/database.ts            # TypeScript types
app/(app)/logs/page.tsx      # Start here for first real data wiring
app/globals.css              # Design system tokens (keep the premium feel)
```

## Supabase Project

Project: farmflowV1
URL + anon key: in `.env.local` (never commit this file)

## Update handoff.md

Before going idle, always update your handoff.md:
- What branch/file/function you're in the middle of
- Any gotchas discovered
- Exact next step (specific enough that a fresh session can resume without asking)
- Any open questions that need lead's decision
