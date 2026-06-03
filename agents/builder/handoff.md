# Handoff — builder

## 🌅 TOMORROW (2026-06-03) — Ross's browser pass + 1 bug to chase
**Ross is doing the live browser-verify pass tomorrow (he asked to be reminded).**
He has the logged-in session; builder can't drive it. Checklist:
- [ ] **Blocks CRUD** on `/blocks` — his org has 0 blocks, so he'll see the new
      empty state; create one, set a parent on a 2nd, edit, remove.
- [ ] **Signup hardening** — weak pw (`12345678`) + mismatched confirm both blocked.
- [ ] **Rate Types** — empty-state + add-version affordance clarity.
- [ ] **Number-input spacing** (see bug below).

🐞 **BUG Ross already spotted (2026-06-02 eve) — chase tomorrow: number-input
STEPPER "not working".** He saw a value `93,6` on a number field (comma decimal
= en-ZA browser locale; matches the rate-types add-version default 90×1.04 =
93.6). **Most likely cause:** my `app/globals.css` fix set
`input[type="number"] { appearance: textfield }` + `::-webkit-*-spin-button {
-webkit-appearance: none }`, which **removes the up/down steppers by design** (to
fix the "squished" report). So "stepper not working" = they're intentionally
gone. **NEEDS ROSS'S INTENT before fixing:** (a) "no steppers, clean number box"
→ current behaviour is correct, the squish is fixed; or (b) "keep working
steppers, just not cramped" → revert the spinner-removal and instead give the
field padding/min-width so native steppers aren't squeezed. Also verify (don't
assume) the comma-decimal value parses on submit — `type=number` `.value`
should normalise to a dot, and the actions use `parseFloat`, so it *should* be
fine; confirm in the browser. Ross will look at the rest tomorrow.

🐞 **BUG #2 (same field, 2026-06-02 eve): the "R" prefix sits OVER/UNDER the
number — vertical alignment / formatting problem.** This is the prefixed amount
input in `app/(app)/rate-types/rate-types-client.tsx` (add-version modal):
`<div className="relative"><div className="absolute left-4 top-3.5 ...">R</div>
<input className="input pl-8 text-xl font-semibold" /></div>`. The absolutely-
positioned `R` (`top-3.5`) doesn't line up with the `text-xl` value baseline, so
it reads as overlapping. FIX next session: align the prefix to the input's
vertical centre (e.g. wrap input row in `flex items-center`, or position the `R`
with `top-1/2 -translate-y-1/2`, and make sure `pl-8` clears it at `text-xl`).
Check the same prefixed-input pattern isn't reused elsewhere with the same
misalignment. Low-risk cosmetic; bundle with the stepper decision above.

---

## ▶️ RESUME HERE (session 2026-06-03) — Activities DONE, next = Input Resources (#4)
**Activities (#3) is shipped + DB-layer verified. No work in flight, no blockers.**
A fresh session should:
1. Run the wake-up routine (pull, read docs/decisions, read this + lead handoff).
2. **Start the Input Resources surface** (#4) — the next build step. Same
   `requireOrg()` + Server-Component-reads / Server-Action-writes pattern.
   GOTCHA: Resources have **append-only price versions** via
   `input_price_history` (like rate_types/rate_history) — the create flow must
   let the user set an initial price (writes the first price version), with an
   add-version flow after (NO edit-in-place on prices = historical integrity).
   Check real columns in `types/database.generated.ts`
   (`input_resources` + `input_price_history`) — don't trust demo field names.
   Resolves Ross's "new resource is R0, no way to set price" feedback.

## ✅ DONE this session (2026-06-03): Activities cut over to real data (#3)
Third master-data surface. Activities are editable + soft-deletable current-state
data (NOT append-only). Files:
- `app/(app)/activities/actions.ts` — `createActivity`, `updateActivity`,
  `deleteActivity` (soft). All derive org via `requireOrg` (never client input);
  RLS double-enforces. Handles the partial unique index on
  `(organization_id, name) WHERE deleted_at IS NULL` → friendly 23505 message on
  duplicate active name. Soft-delete sets `deleted_at` (labour_logs FK
  `activity_id`, so history must keep pointing at its activity); soft-delete also
  frees the name for reuse (partial index).
- `app/(app)/activities/page.tsx` — now a Server Component; reads activities under
  RLS (`deleted_at IS NULL`, ordered by created_at), `force-dynamic`.
- `app/(app)/activities/activities-client.tsx` — table (name / category / status
  + edit/remove), add-form, edit modal, empty state, `is_active` toggle, category
  free-text with a `<datalist>` of common suggestions. Shared `ActivityFields`
  sub-component. useTransition + router.refresh().
- Schema (`types/database.generated.ts`): activities = id, name, **category**
  (free text, nullable), **is_active** (bool, default true), metadata, deleted_at,
  created_by, org_id. No `parent`/`area`/`crop` — simpler than blocks.
`npx tsc --noEmit` → clean (exit 0). Committed `e039dd9`. Demo engine untouched
for the other surfaces; the demo activities page was hard-swapped (no flag).
**DB-layer verified live by builder** (farmflowV1, rolled-back impersonation,
zero residue): insert persists with correct org + created_by + is_active default;
duplicate active name blocked (23505); em user sees 0 of Academia's activities
AND cross-tenant UPDATE affects 0 rows. All 4 per-op RLS policies confirmed
present via `current_user_org_ids()`.
**Still browser-only (needs an authed session):** empty-state render, add/edit/
remove click-through, category datalist UX.

## ▶️ (superseded) prior resume — Activities was NEXT (now done above)
The block below was the 2026-06-02 resume note. Kept for history.

State of the world right now:
- ✅ **Rate Types** (#1) — shipped to real data; runner live-verified the **DB
  layer GREEN** (persist / append-only / cross-tenant isolation) in `c084080`.
  Only a browser UI pass remains (empty state + add-version clarity).
- ✅ **Blocks** (#2) — shipped this session (real data, full CRUD: create / edit
  / soft-delete). `tsc` clean; route compiles (`/blocks` → 307 to /login
  unauth'd, no 500). NOT yet browser-verified (needs an authed session).
- ✅ **Signup password hardening + global number-input CSS** — pushed (`24a2c07`).
- ✅ **Demo hydration mismatch** — fixed + pushed (`eb8fa9a`), Ross confirmed gone.
- 🔬 **Outstanding = browser-only verification** (runner/Ross, not builder):
  see the "🔬 NEEDS RUNNER / BROWSER" section — now includes Blocks CRUD.
- 🧊 Deferred (lead, `e25570c`): a post-Phase-3 UI/UX polish pass. Not now.

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
- **Blocks CRUD** (this session): create a block (persists, org_id + created_by
  correct); set a parent → hierarchy shows; edit name/code/area/parent/status;
  soft-delete (block disappears, row keeps `deleted_at`); removing a block with
  active sub-blocks is refused; cross-tenant isolation (org B can't see/edit
  org A's blocks). Empty state shows for a fresh org.

## ✅ DONE this session (2026-06-02): Blocks cut over to real data (#2)
Second master-data surface. Full CRUD (blocks are mutable + soft-deletable —
NOT append-only like rates). Files:
- `app/(app)/blocks/actions.ts` — `createBlock`, `updateBlock`, `deleteBlock`.
  All derive org via `requireOrg` (never client input); RLS double-enforces.
  `deleteBlock` is a SOFT delete (sets `deleted_at`) — labour/input logs FK to
  `block_id`, so historical logs must keep pointing at their block; it also
  refuses to remove a block that still has active children (no orphans).
  `updateBlock` blocks self-parent; status validated to active/inactive/archived.
- `app/(app)/blocks/page.tsx` — Server Component, reads blocks under RLS
  (`deleted_at IS NULL`, ordered by created_at), `force-dynamic`.
- `app/(app)/blocks/blocks-client.tsx` — table (name/code/parent/area/status +
  edit/remove actions), add-form, edit modal, empty state, status badges,
  useTransition + router.refresh(). Shared `BlockFields` sub-component.
- Real schema gotcha handled: column is **`area_ha`** (not the demo's
  `area_hectares`); real blocks have `code` + `status` and NO `crop`/`variety`
  (those were demo-only). No unique constraint on name → duplicates allowed
  (no 23505 handling needed). Verified live: blocks has all 4 per-operation RLS
  policies via `current_user_org_ids()`.
`tsc` clean; `/blocks` compiles (307 → /login unauth'd). Demo engine untouched
for the other surfaces. The demo `blocks` page was hard-swapped (no flag).

## ⏭️ NEXT (resume here): Activities surface (#3)
Same pattern as Rate Types / Blocks:
1. Read the demo `app/(app)/activities/page.tsx` to match shape, and the real
   `activities` columns in `types/database.generated.ts` (don't trust demo
   field names — same lesson as Blocks' `area_ha`).
2. `app/(app)/activities/page.tsx` → Server Component, read via requireOrg.
3. `app/(app)/activities/actions.ts` → create/update (activities are editable;
   check for soft-delete + any unique constraint before assuming 23505 handling).
4. `app/(app)/activities/activities-client.tsx` → UI in the same premium style.
5. Reuse `requireOrg()`. Keep the demo engine running for not-yet-migrated pages.

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
