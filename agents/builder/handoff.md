# Handoff — builder

## ▶️ RESUME HERE (session ended 2026-06-02)
**Everything dispatched to builder is DONE, committed, and pushed. No work in
flight, no blockers.** A fresh session should:
1. Run the wake-up routine (pull, read docs/decisions, read this + lead handoff).
2. **Start the Blocks surface** — the next build step (details in "⏭️ NEXT" below).
   Same `requireOrg()` pattern as Rate Types; blocks are editable (not
   append-only); hierarchy via `parent_id`.

State of the world right now:
- ✅ **Rate Types** — shipped to real data; runner live-verified the **DB layer
  GREEN** (persist / append-only / cross-tenant isolation) in commit `c084080`.
  Only a **browser UI pass** remains (fresh-org empty state + clarity of the
  add-version affordance). Not a builder blocker.
- ✅ **Signup password hardening + global number-input CSS** (the two 🟢 items) —
  done, `tsc` clean, pushed (`24a2c07`). Needs the same browser verify pass.
- ✅ **Demo hydration mismatch** — fixed + pushed (`eb8fa9a`), Ross confirmed gone.
- 🔬 **Outstanding = browser-only verification** (runner/Ross, not builder):
  Rate Types empty-state + add-version clarity; weak-pw blocked client+server;
  pw mismatch blocked; valid signup completes; number inputs no longer cramped.
- 🧊 Deferred (lead, `e25570c`): a post-Phase-3 UI/UX polish pass (specialist
  agent). Not now.

---

## 📥 INBOX — FROM LEAD (2026-06-02) — UI feedback from Ross's first real signup
Ross signed up live (real Gmail, email-confirm ON — both confirmed working) and
walked the app. His feedback, triaged. Do the 🟢 items, then resume Blocks.

🟢 **DONE (2026-06-02) — both shipped, tsc clean, /signup renders green:**
1. ✅ **Password hardening on `/signup`** — rule chosen by Ross: **8+ chars,
   must contain a letter AND a number**. Enforced server-side in
   `app/signup/actions.ts` (authoritative) and mirrored client-side on the form
   via HTML5 `minLength={8}` + `pattern="(?=.*[A-Za-z])(?=.*\d).{8,}"` + `title`.
   Added a **Confirm password** field (`confirmPassword`); mismatch → server
   returns "Passwords do not match." Added a calm hint line under the password
   field. `/login` left untouched (existing users). Match is enforced
   server-side only (no client-side live match yet — possible future polish).
2. ✅ **Number inputs squished** — added a global rule in `app/globals.css`
   (after the `input[type="date"]` block) stripping the native number steppers
   (`::-webkit-*-spin-button` + `appearance: textfield`). Applies everywhere.
NEEDS RUNNER live-verify (I can't drive the logged-in browser): weak pw blocked
both client+server, mismatch blocked, valid signup still completes; and number
inputs (e.g. rate-types add-version) no longer cramped.

🟡 **WORKING AS DESIGNED — do NOT add edit-in-place:**
- Ross reported "can't change rates or prices." Correct — that's the #1 rule
  (append-only `rate_history` / `input_price_history` = historical integrity).
  The amount is set ONLY via "add new version," never edited. The only allowed
  improvement is CLARITY of the add-version flow on the real Rate Types surface
  (e.g. clearer affordance/label). Editability stays off. Don't "fix" this.

🔵 **DEMO→REAL SEAM — resolves on migration, do NOT patch demo:**
- "New resource is R0, no way to set price" → Resources page is still the demo
  engine. Fix lands when **Input Resources (#4)** is migrated: the create flow
  must let the user set an initial price (writes the first
  `input_price_history` version), with proper add-version UX after.
- "New rate type can't be applied to a new worker" → Workers page is still demo
  (and uses the wrong `rate_type_id` col). A demo dropdown can't see a real
  rate type. Resolves when **Workers (#5)** is migrated.
- **Workers table formatting** (`app/(app)/workers/page.tsx`) — fold these into
  the #5 migration, don't patch demo standalone:
  - 4th column has a blank `<th>` with a faint right-aligned "Active" — orphaned.
    Make it a labelled **Status** column with a proper badge, OR an actions
    (edit/remove) column. Right now it's neither.
  - "Active" is HARDCODED (line ~66), not read from worker data.
  - "Employment" header → rename to "Type" (shows `worker_type`).
  - No edit/remove affordance despite the empty column implying one.

DROPPED: the "country code / 27 on signup" report — there is no phone/country
field anywhere in the codebase; Ross confirmed he misread (autofill/other tab).

⏭️ **After the 🟢 fixes: resume Blocks** (your existing NEXT, unchanged).

## ✅ DONE — FROM LEAD (2026-06-01), completed 2026-06-02: signup boot-check
**Task was: confirm the signup flow boots & renders so Ross can run a real-email
signup test.** RESULT: confirmed green. Ross can run his test.

- `npm run dev` boots clean: Next.js 16.2.6 (Turbopack), `Ready in ~0.6s`.
  Only benign warnings (Turbopack cache reset; `middleware`→`proxy` deprecation
  notice). Port is whatever's free — I tested on 3100; default is 3000.
- `/signup` → HTTP 200, renders all three fields: `farmName`, `email`,
  `password` (+ "Create my account" / "Start your farm"). `/login` → HTTP 200.
- `app/signup/actions.ts` is correctly wired: `auth.signUp({ options: { data:
  { farm_name }}})`, NO old `create_organization` RPC. Trigger provisions
  profile + org from metadata.
- **What success looks like for Ross** (the action branches on `data.session`):
  - If email confirmation is ON → he sees the "Check your email to confirm…"
    info message and NO redirect.
  - If email confirmation is OFF → he's redirected to **`/onboarding`** (NOT
    `/dashboard` — Lead's dispatch guessed dashboard; the code says onboarding).
  - So his test result itself answers #2b: which screen he lands on tells us
    whether confirmation is on, and whether his real Gmail is accepted at all.
- No fixes were needed — the render path was already clean. No code changed.

⏭️ After Ross's test: resume Blocks per the plan below.

## 🩹 SIDE FIX (2026-06-02): demo hydration mismatch — DONE (commit eb8fa9a)
Ross hit a Recoverable hydration error on `/dashboard` after relog. Root cause
was in the shared demo engine, NOT recent migration work: `DemoProvider`
(`lib/demo/engine.tsx`) initialised `useState` from `localStorage`, so the
server rendered seed data while the client rendered persisted state → value +
locale mismatch. Fix: seed on server + first client render, load saved state in
a post-mount `useEffect`, and gate the persist effect behind a `loaded` flag so
first mount can't clobber saved state with the seed. Also pinned
`toLocaleString('en-ZA')` on SSR-rendered amounts in `dashboard/page.tsx` (3) and
`reports/page.tsx` (6) so number format is deterministic across server/client.
`tsc` clean, Ross confirmed the error is gone. Format is now SA-style
(space-thousands, comma-decimal) — flip to `en-US` if ever desired.

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

## 🔬 NEEDS RUNNER / BROWSER (live verify)
Rate Types **DB layer was verified GREEN by runner** (`c084080`): persist (#2),
add-version newest-first (#3), append-only blocked (#4), cross-tenant isolation
(#5) — all confirmed live against `farmflowV1`. The ONLY remaining gaps are
browser-level (no one has driven the rendered pages yet):
- Rate Types: fresh-org **empty state** render + clarity of the **add-version**
  affordance Ross flagged (the live org now has data, so empty state needs a
  brand-new org session to observe).
- Signup hardening (`24a2c07`): weak pw blocked **client AND server**, pw
  mismatch blocked, a valid 8+letter+number signup still completes end-to-end.
- Number inputs (e.g. rate-types add-version) no longer cramped by steppers.

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
