# Handoff — lead

## Project location
C:\ClaudeProjects\farmflow

## DEFERRED INITIATIVES (not roadmap items yet — don't start early)
- **Dedicated UI/UX polish pass via a specialist agent** (Ross, 2026-06-02).
  Goal: make FarmFlow look/feel like a high-end SaaS costing system and be
  "extremely user-friendly." TIMING: only AFTER Phase 3 (all surfaces migrated
  to real data) is complete — ideally folded in with Phase 5 onboarding polish.
  Reason to wait: polishing demo-engine screens that will be torn out during
  migration is wasted work. Brief should absorb Ross's standing UX notes:
  quantity/number inputs feel small, numbers hard to read (NOTE: builder shipped
  a global number-input CSS fix 2026-06-02 — stepper strip — which may partially
  address this; verify before re-listing). Do NOT spin this up mid-migration.
- **Newsletter / mailing list** — deferred (see decisions.md 2026-06-02). Emails
  already captured in auth.users; external tool, post-backend.

## Currently in flight (updated 2026-06-01, lead)
- Phase 1 (schema migration) ✅ done + live-verified by runner.
- Phase 2 (auth + org creation on signup) ✅ done + live-verified. RLS
  recursion #0, missing-profiles #1, no-org-on-confirm #2 all fixed
  (builder) and re-verified live (runner). Historical-integrity tests pass.
- Phase 3 (replace demo layer) 🔄 IN PROGRESS. REORDERED 2026-06-01 (Ross +
  builder, recorded in docs/decisions.md): master-data surfaces ship FIRST in
  dependency order — Rate Types → Blocks → Activities → Input Resources →
  Workers — and Logs ships LAST. Reason: Logs is the most dependent surface
  (needs worker→rate type→rate version + block + activity), and new orgs start
  blank, so Logs-first meant empty dropdowns or seeding throwaway data.
  Rate Types surface: ✅ builder shipped it to real data this session
  (`requireOrg()` tenant primitive added in `lib/supabase/org.ts`; tsc clean;
  demo engine untouched). NEEDS RUNNER live-verify (see builder/handoff).
  NEXT BUILDER BUILD STEP: Blocks surface — but paused (see dispatch below).

## RESOLVED (2026-06-02) — signup verified + UX feedback triaged
- Builder confirmed `/signup` boots (commit 2d2a071). Ross then ran a real-Gmail
  signup live: **accepted, confirmed via email, logged in.** So **#2b is closed**
  (real domains pass; earlier rejects were fake-domain only) and **email
  confirmation is ON**. Recorded in decisions.md (2026-06-02 #2b entry, builder).
- Ross's first-walkthrough UX feedback triaged → builder inbox + decisions.md
  (2026-06-02 UX triage entry). Build-now: password hardening (min strength +
  confirm field) on /signup, and a global number-input CSS fix. Confirmed
  working-as-designed (leave uneditable): rate/price append-only. Demo→real seam
  (fix via migration, not demo patch): resource-R0 (#4 Input Resources),
  rate-type-not-in-worker-dropdown + Workers table formatting (#5 Workers).
  Dropped: "phone/country on signup" — no such field exists.
- Mailing-list feature **deferred** (not a roadmap item): auth.users already
  holds customer emails; distribution work for later, external tool.
- NEXT BUILDER: do the two 🟢 fixes, then resume **Blocks** (unchanged NEXT).

## Service-role key — RESOLVED (2026-06-01)
Ross added `SUPABASE_SERVICE_ROLE_KEY` to `.env.local`. NOTE: with Logs now
last, the service-role resolution RPCs (get_active_rate_history etc.) aren't
needed until the Logs surface — the master-data surfaces (incl. Rate Types
writing to rate_history) use the authenticated client (rate_history has
INSERT+SELECT policies for authenticated). So the key unblocks the *eventual*
Logs work; builder is free to proceed on Rate Types now regardless.

## STILL open on Ross (not blocking Rate Types)
- Confirm Supabase Auth → Providers → Email accepts real domains (gmail).
  Runner saw GoTrue reject @example.com / .test signups (#2b); real-user
  signup is unverified until a browser test with a real address.

## Open questions
- Email-confirmation ON/OFF: Ross's Auth setting. Trigger-based provisioning
  works either way, so not blocking — but affects whether a post-confirm org
  path is ever needed.
- Sign-out flow doesn't exist anywhere yet — builder to add when the
  authenticated nav surface gets wired (not blocking, log it).

## Housekeeping
- Working tree has uncommitted edits to all three agents' CLAUDE.md
  (template → FarmFlow personalization). Already in effect. Whoever authored
  them should commit; flagging so they aren't lost.

## Don't lose this
- Roadmap order is fixed: schema → auth → real data layer → Google OAuth → onboarding
- Do not allow any work on phases 4-5 until 1-3 are solid
- The demo layer must keep working throughout the transition (replace surface
  by surface, never patch the demo engine in place)
- Historical integrity is non-negotiable: snapshot rate_amount/total_cost/
  unit_price on every log insert; never recompute reports from current rates
- No decision needed this session — team is following the roadmap in order,
  no scope creep, no agent disagreement to break.