# Washboard — Build Plan & Status

See [SPEC.md](./SPEC.md) for the full product spec (entities, feature
areas, principles). **This file is the project's memory** — if a session
gets closed mid-work, re-read this file (Status + Decisions log) before
doing anything else; it should be enough to pick back up with zero lost
context.

## Status

Update the checkbox and the one-line note below whenever a phase's
"done when" check passes.

- [~] Phase 1 — scaffold, Supabase dev project, and Playwright harness
      all done and verified locally (`tsc`, `eslint`, `test:e2e` all
      green). Vercel deploy still pending — that's the one piece of
      Phase 1's "done when" left.
- [ ] Phase 2 — Auth
- [ ] Phase 3 — Buildings & onboarding
- [ ] Phase 4 — Machines & slot config (admin)
- [ ] Phase 5 — Booking calendar, read side (TanStack Query + Realtime)
- [ ] Phase 6 — Booking action, the core loop (concurrency)
- [ ] Phase 7 — QR code for the building's laundry room
- [ ] Phase 8 — Email invites (Resend)
- [ ] Phase 9 — Polish pass
- [ ] Phase 10 — Deploy & final QA

## Decisions log

Append here whenever a real decision gets made mid-build that isn't
already in SPEC.md — so it's not lost if context resets.

- **2026-08-20** — washboard is a separate, real (non-study) project,
  deliberately not filed under `projects-to-study-nextjs/`; SPEC+PLAN
  live directly in this folder, code will too. Reuses
  `projects-to-study-nextjs/laundry-booking/SPEC.md`'s technical shape
  (concurrency-safe slot booking, multi-tenant RLS) but that spec stays
  untouched as its own separate study project.
- **2026-08-20** — TanStack Query brought in specifically for optimistic
  booking + reconciling Supabase Realtime events into the calendar's
  cache (see SPEC's "Why this project").
- **2026-08-20** — Playwright integration tests from Phase 1, not
  deferred — this project's brief explicitly calls for it, unlike the
  laundry-booking study spec.
- **2026-08-20** — default slot schedule: 07:00–22:00, 3-hour slots, 5
  per machine per day. Admin-configurable per building, but this is the
  seed default and the concrete example the product is designed around.
- **2026-08-20** — accessibility and SEO added as explicit standing
  requirements (SPEC.md's Accessibility & SEO section, Phase 9's audit
  bullets below) after being caught only in retrospect — flagged the same
  day for `coffeedex` too (which had real gaps: no `sitemap.ts`/
  `robots.ts`, no JSON-LD) and for the shared kickoff doc/CLAUDE.md so
  future projects don't repeat the miss.
- **2026-08-20** — `washboard-dev` Supabase project created: org
  `washboard` (`xghhayakyvtifqtbllna`), project ref `cxxsmzkatddjaxmszwfo`,
  region `eu-north-1`. Created via a temporary Personal Access Token
  (Management API, used only for this one session, never written to
  disk) under account `joaoeffting1990@gmail.com` — note this is a
  **different** account than `coffeedex`'s (`joaopauloeffting@gmail.com`);
  flagged to the user, not silently assumed to match. `washboard-prod`
  not created yet — deferred to Phase 10 per plan.

---

**How to use this plan:** each phase is a small, single-sitting-sized
chunk ending in something clickable in a real browser — not just code
that compiles. Don't move to the next phase until its "done when" check
passes. This is a spec-only phase list (what to build and why), not a
line-by-line tutorial — write/adapt the actual code when you get there.

Every route needs real nav to reach it, every state-changing action needs
its full counterpart, every page needs at least basic Tailwind styling —
the standing lesson from this repo family's earlier tutorials (see
`projects-to-study-nextjs/CLAUDE.md`'s "Tutorial/project completeness").

---

## Phase 1 — Project scaffold, deploy, Playwright setup

- `create-next-app`, new Supabase project (dev), the three-client pattern
  (browser/server/proxy), Tailwind + shadcn/ui init.
- Playwright installed and configured (`playwright.config.ts`,
  `tests/README.md` documenting how the suite runs and how test accounts
  get created — decide the approach now, e.g. a `global-setup.ts` that
  signs up fresh throwaway accounts each run, same as `coffeedex`).
- Deploy to Vercel.

**Done when:** a live Vercel URL shows a styled homepage placeholder, and
`npx playwright test` runs green against an empty test suite (proves the
harness itself works before any real feature needs it).

## Phase 2 — Auth

- Signup/login/logout via Supabase Auth, protected-route pattern.
- `profiles` table created now (even though `building_id`/
  `is_building_admin` don't land until Phase 3) — same reasoning as the
  laundry-booking study spec: it's not doing anything yet, but it's where
  those columns belong.
- Playwright spec: signup → login → protected page → logout → login
  loop.

**Done when:** the loop above works both by clicking through the browser
and as a passing Playwright spec.

## Phase 3 — Buildings & onboarding

- `buildings` table (`name`, `address`, `join_code`) + RLS.
- `profiles` gains `building_id` (nullable) and `is_building_admin`.
- "Create a building" flow: a signed-in user with no `building_id` can
  create one, becomes its admin, gets a generated `join_code`.
- "Join a building" flow: a signed-in user with no `building_id` enters a
  `join_code` to attach to that building.
- RLS discipline starts here: every table added from this phase on
  filters by "same `building_id` as the current user."
- Playwright spec: two throwaway accounts each create their own building
  (two different `join_code`s); a third joins one by code and lands
  correctly scoped to it.

**Done when:** the scenario above passes both manually and as a
Playwright spec.

## Phase 4 — Machines & slot config (admin)

- `machines` table (`building_id`, `label`, `type`, `is_out_of_service`)
  + RLS scoped to `building_id`.
- `slot_config` table (`building_id`, `slot_duration_minutes`,
  `day_start_time`, `slots_per_day`, `max_bookings_per_resident_per_week`,
  `booking_window_days`) — one row per building, seeded with the default
  schedule (07:00 start, 180-minute slots, 5/day) on building creation.
- Admin-only dashboard: add/edit/remove machines, edit slot config.
- Playwright spec: an admin can add a machine and edit their building's
  schedule/rules; a resident in the same building cannot reach those
  controls; a resident of a *different* building can't see this
  building's machines at all.

**Done when:** the spec above passes, both manually and automated.

## Phase 5 — Booking calendar, read side (TanStack Query + Realtime)

- A day (or week) view per building: for each machine, compute which
  slot indices exist from `slot_config` (not stored rows), cross-
  referenced against real `bookings` to mark each slot open /
  taken-by-someone / yours.
- Wire up TanStack Query for the calendar's data fetching (initial
  server-fetched grid handed to the client as the query's initial data).
- Wire up a Supabase Realtime subscription on `bookings` (scoped to
  `building_id`) that pushes changes into the same query cache — no
  booking action exists yet, so verify this by manually inserting/
  deleting a test booking row in the Supabase dashboard and watching the
  grid update live without a page refresh.
- Playwright spec: manually-inserted booking row shows as taken; grid
  reflects a changed `slot_config` (different slot count/duration)
  without code changes.

**Done when:** both checks above pass, including the live-without-
refresh Realtime behavior.

## Phase 6 — Booking action, the core loop (concurrency)

- `bookings` table (`machine_id`, `user_id`, `date`, `slot_index`) with
  the **unique constraint on `(machine_id, date, slot_index)`**.
- Server Action: insert a booking, catch a unique-violation cleanly,
  surface "someone just booked this — pick another slot" rather than a
  raw error.
- Client: TanStack Query optimistic update — slot shows "yours
  (pending)" immediately on click, then confirms or rolls back based on
  the Server Action's result.
- Server-side checks before insert: `booking_window_days` (reject a date
  too far out) and `max_bookings_per_resident_per_week` (count the
  user's bookings that week) — both must reject a request that skips the
  UI.
- Cancel-own-booking: scoped so a resident can only cancel their own.
- **Flagship Playwright spec**: two browser contexts, two logged-in
  residents, both submit a booking for the same slot as close together
  as the test can manage; assert exactly one succeeds and the other
  receives the clean conflict message. Also assert: a booking past the
  window is rejected, and a booking beyond the weekly cap is rejected.

**Done when:** the concurrency spec passes reliably (run it a few times —
race conditions can be flaky to catch even when the fix is correct), plus
the window/cap rejections.

## Phase 7 — QR code for the building's laundry room

- Generate a QR code (client or server-rendered) encoding a direct link
  to the building's booking calendar, shown on the admin's building page
  with a "print this for the laundry room" affordance.
- Playwright spec: the QR code renders and encodes the expected URL
  (decode it in the test, don't just assert an `<img>` exists).

**Done when:** the QR renders correctly and the spec passes.

## Phase 8 — Email invites (Resend)

- Admin enters a resident's email, sends an invite containing the
  building's `join_code` and a direct join link.
- Playwright spec: triggering the invite calls Resend (mock/intercept
  the network call rather than sending a real email in CI — document the
  approach in `tests/README.md`); manually verify one real send to an
  actual inbox outside the automated suite.

**Done when:** a real invite delivers an actual email with a working
link, and the automated spec passes.

## Phase 9 — Polish pass

- Empty/loading/error states — an empty calendar (no machines configured
  yet), a full week (nothing bookable), a failed booking attempt.
- Full click-through completeness audit: every route reachable via real
  nav, every action has its counterpart, consistent styling across admin
  and resident views.
- **Accessibility & SEO audit** (see SPEC.md's Accessibility & SEO
  section — don't treat this as new scope invented here, it should
  already be mostly true by this point from building each feature
  correctly the first time):
  - `aria-live` region on the booking calendar for Realtime/optimistic
    state changes.
  - Keyboard-only click-through: every action reachable and completable
    (create/join building, add machine, book/cancel a slot) using only
    Tab/Enter/Space, no mouse.
  - `robots.ts` disallows authenticated building/calendar routes;
    `sitemap.ts` only lists the public marketing surface.
  - `generateMetadata`/Open Graph present on the public marketing/signup
    pages.
- Playwright spec(s) covering at least one empty-state and one
  error-state scenario per major page.

**Done when:** a cold click-through — signup, create or join a building,
(as admin) add a machine and set rules, (as resident) book and cancel a
slot — works with no dead ends for either role, and the polish specs
pass.

## Phase 10 — Deploy & final QA

- Final Vercel deploy, real environment variables (including Resend), a
  last pass on the live (not local) deployment.
- Re-run the Phase 6 concurrency test against production specifically —
  local dev and a real deployed Postgres connection can behave
  differently under near-simultaneous requests. This is the one
  explicitly-allowed prod run.

**Done when:** the live Vercel URL supports the full loop end to end, and
the two-simultaneous-bookings race test still resolves to exactly one
winner in production.

---

## Future ideas

See [SPEC.md](./SPEC.md#future-ideas-explicitly-deferred-not-forgotten) —
booking reminder emails/push, recurring bookings, no-show/auto-release,
generalizing beyond laundry, waitlists, admin analytics, and multi-admin
buildings are all explicitly deferred, not in-scope-but-forgotten.
