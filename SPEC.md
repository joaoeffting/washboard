# Product Overview — Washboard

A real product name, not a placeholder (unlike `laundry-booking`'s working
title over in `projects-to-study-nextjs/`) — this folder lives on its own
directly under `personal/`, doc-and-code together, not split across a
study-repo + separate-code-repo like this workspace's other projects.

## What this is

A multi-tenant laundry-booking platform for apartment buildings: each
**building** manages its own machines and booking rules, **residents** book
fixed time slots to do their laundry, an **admin** per building configures
machines/schedule/rules. Multiple buildings sign up independently, fully
isolated from each other.

**Default booking day** (admin-configurable per building, but this is the
seed/default and the concrete example the product is built around):
closes at 22:00 so laundry doesn't run late and bother neighbors, opens at
07:00, in 3-hour slots:

- 07:00–10:00
- 10:00–13:00
- 13:00–16:00
- 16:00–19:00
- 19:00–22:00

Five slots per machine per day.

## Why this project

Two goals at once, both explicit:

1. **More Next.js + Supabase practice**, same as `coffeedex` and
   `storyloom` on this workspace.
2. **A real portfolio piece for freelancing** — not a study-only spec.
   Booking/reservation systems were flagged (during `wattpad-platform`
   freelance-positioning discussion) as the most directly sellable
   category to demonstrate, and this is that category, concretely.

This evolved from `projects-to-study-nextjs/laundry-booking/SPEC.md`,
which stays untouched as its own separate study spec — washboard reuses
its core technical shape but is the one actually getting built as a real
product, with a wider brief:

- **Time-slot scheduling under real write concurrency** — two residents
  can click "book" on the same slot at the same instant. A Postgres
  unique constraint on `(machine_id, date, slot_index)` is the entire
  double-booking prevention mechanism — the second concurrent insert just
  fails cleanly, no explicit locking needed. This is also the one
  scenario in the whole app that's worth an actual automated test rather
  than manual two-tab clicking (see Testing).
- **True multi-tenant RLS** — row visibility scoped by *which building a
  user belongs to*. A resident must never see or book another building's
  machines.
- **Server-enforced business rules** — configurable per-building limits
  (max bookings per resident per week, how far ahead you can book) that
  hold even against a client trying to bypass the UI.
- **Client-side interactivity worth TanStack Query specifically** (new
  vs. the laundry-booking study spec, which explicitly deferred it): the
  booking calendar needs an optimistic "yours (pending)" state on
  slot-click that resolves to confirmed or rolls back to the clean
  conflict message, plus Supabase Realtime pushing other residents'
  book/cancel events into the same client cache so the grid updates live
  without a full page reload. That's exactly the "initial server-fetched
  list + client-driven updates" case the root `CLAUDE.md` calls out as
  the trigger for bringing TanStack Query in.
- **Integration test coverage from day one** (also new vs. the study
  spec) — see Testing below.

## Tech stack

Repo defaults: Next.js App Router (v16+), Supabase (Postgres/Auth/RLS),
TypeScript, Tailwind, shadcn/ui, Vercel.

Plus, deliberately (not default-by-habit — each one earns its place):

- **TanStack Query** — optimistic booking mutations + reconciling
  Supabase Realtime booking/cancel events into the calendar's client
  cache. See "Why this project" above.
- **Resend** — transactional email for admin-sent building invites (an
  actual "you've been invited to join [Building]" email, not just a
  shareable code).
- **Playwright** — integration tests from Phase 1, not deferred. See
  Testing.

Not needed: Stripe, PostHog — revisit only if the brief actually grows to
need them.

## Core entities

- **Building** — the tenant. `name`, `address`, a shareable `join_code`
  residents use to self-signup into the right building.
- **Profile** — extends `auth.users`. `building_id` (nullable until
  joined), `is_building_admin` (scoped to one building — a person admins
  at most one building; see Future ideas for multi-admin).
- **Machine** — belongs to a building. `label` (e.g., "Washer 1"), `type`
  (`washer` | `dryer`), `is_out_of_service`.
- **Slot config** — per-building schedule definition: `slot_duration_minutes`
  (default 180), `day_start_time` (default 07:00), `slots_per_day`
  (default 5). A small config row, not pre-generated slot rows — "which
  slot indices exist for machine X on date Y" is computed from this
  config plus the actual `bookings` for that day. Also carries the
  business rules: `max_bookings_per_resident_per_week`,
  `booking_window_days` (how far ahead a resident can book).
- **Booking** — `machine_id`, `user_id`, `date`, `slot_index`. The
  **unique constraint on `(machine_id, date, slot_index)`** is the entire
  double-booking prevention mechanism.

## Feature areas

### 1. Building & resident onboarding

- An admin signs up and creates a building — becomes its
  `is_building_admin`, building gets a generated `join_code`.
- Residents sign up normally, then enter a `join_code` to attach
  themselves to a building (self-service, matches how a code posted on an
  actual laundry-room noticeboard works).
- Admin can send an email invite (Resend) containing the join code
  directly.
- **QR code for the physical laundry room**: the admin's building page
  shows a printable QR code encoding a direct link to that building's
  booking calendar (post it on the actual noticeboard next to the
  machines). Low effort, genuinely how a real building would roll this
  out, and a nice concrete portfolio/demo touch.

### 2. Machine & rule management (admin)

- Add/edit/remove machines for their building.
- Configure the slot schedule (start time, slot duration, slots per day)
  — changing this doesn't retroactively touch existing bookings, only
  affects slots offered going forward.
- Configure `max_bookings_per_resident_per_week` and `booking_window_days`.
- Mark a machine `is_out_of_service` — its slots stop being bookable
  until cleared.

### 3. Booking calendar (resident)

The core loop, and the one place both the concurrency handling and the
TanStack Query/Realtime wiring matter:

- A day/week view per machine (or all machines for a day) showing each
  slot as open, taken (by someone else), or "yours."
- Clicking an open slot shows it as "yours (pending)" immediately
  (TanStack Query optimistic update), then resolves: either it stays
  confirmed, or — if someone else's insert landed first — reverts with a
  clean "someone just booked this — pick another slot," never a crash or
  a stuck pending state.
- A Supabase Realtime subscription on `bookings` (scoped to the viewer's
  `building_id`) pushes other residents' book/cancel events into the same
  query cache, so the grid updates live for everyone watching it, not
  just the person who clicked.
- Both business rules (weekly cap, booking window) are checked
  **server-side** before the insert — a request crafted to skip the UI
  must still be rejected.
- Residents can cancel their own upcoming bookings, freeing the slot
  (also reflected live via the same Realtime channel).

## Product principles worth carrying forward

- **Fixed slots, not arbitrary time ranges.** Makes double-booking
  prevention a simple unique constraint instead of interval-overlap
  logic, and matches how real shared-laundry booking boards work.
- **Tenant isolation is load-bearing, not cosmetic.** Every query must be
  provably scoped by `building_id`, not just filtered in the UI.
- **Business rules are enforced where a bypassed UI can't skip them** —
  weekly cap and booking window live in server-side checks, not just
  client-side form validation.
- **Optimistic UI must always reconcile with server truth.** A pending
  client-side state (TanStack Query) is a UX nicety, never the source of
  truth — the unique constraint and Realtime feed are what's actually
  real; the UI just tries to feel instant while waiting to hear back.
- **Laundry is the first resource type, not a hardcoded assumption.**
  Keep `machine`/`type` naming laundry-specific for now (see Future
  ideas for generalizing), don't bake laundry-only logic into the
  booking/slot mechanism itself.
- **Test coverage is part of the portfolio demonstration, not optional
  polish** — see Testing.

## Accessibility & SEO

Both are standing requirements from `projects-to-study-nextjs/CLAUDE.md`
(the "shared conventions" every project in that repo's kickoff flow
carries forward, even one like this that lives outside the repo) — not
optional polish, and not deferred to a final pass:

- **Accessibility** — real `<button>`/`<label htmlFor>` elements, not
  `<div onClick>`; icon-only controls get `aria-label`, decorative icons
  get `aria-hidden="true"`; `lang` set on `<html>`; shadcn's
  `focus-visible` ring left intact. The one thing washboard needs more
  than a typical project: the booking calendar's live state (Realtime
  book/cancel events, optimistic "yours (pending)" states) needs an
  `aria-live` region — without one, a screen reader user gets no signal
  that a slot they're looking at just changed out from under them.
- **SEO** — mostly narrow here, since the booking calendar itself sits
  behind auth (nothing to index once a resident's inside a building).
  What's actually public: the marketing/landing page and signup flow —
  those get real `generateMetadata`/Open Graph, same as any other
  project. `robots.ts` should explicitly disallow the authenticated
  building/calendar routes; `sitemap.ts` only needs the public marketing
  surface, not per-building calendar pages (those aren't content anyone
  should be searching for).

## Testing

Playwright integration tests from Phase 1, not deferred to a later pass
(overrides the laundry-booking study spec's "not needed yet — revisit if
the brief grows to need it," since this project's brief explicitly
includes it). Add a spec in `tests/` alongside each feature as it's
built, not as a separate follow-up.

- No direct DB seeding/reads — no service-role key in this project (see
  root `CLAUDE.md`'s "Schema changes" section) — specs interact only
  through the real UI, same as a real resident/admin would.
- Auth: decide at Phase 1/2 how test accounts get created (a
  `tests/global-setup.ts` that signs up fresh throwaway accounts each
  run, if dev has email confirmation off — same pattern as `coffeedex`).
- The double-booking race (Phase 6) is the one scenario worth its own
  dedicated test: two Playwright browser contexts, two logged-in
  residents, both submit a booking for the same slot as close together
  as the test can manage — assert exactly one succeeds and the other
  gets the clean conflict message.
- Only ever run against the dev Supabase project — never point it at
  prod (except the one explicit re-run called out in Phase 10's "done
  when," to catch behavior differences between local dev and a real
  deployed Postgres connection).

## Future ideas (explicitly deferred, not forgotten)

- **Booking reminder emails/push** ("your slot starts in 30 minutes").
- **Recurring bookings** ("same slot every week") — real convenience,
  but the core loop should work for one-off bookings first.
- **No-show / auto-release** — free a slot automatically if nobody's
  laundry shows up (needs some signal of "used," which doesn't exist
  yet).
- **Generalizing beyond laundry** — renaming "machine" to a generic
  "bookable resource" (shared guest parking, a common room). Explicitly
  not done now — see product principles.
- **Waitlists** for fully-booked slots.
- **Admin analytics** — utilization per machine, no-show tracking.
- **Multiple admins per building**, or a regional/property-manager role
  spanning several buildings — current model caps at one admin per
  building deliberately.

---

See [PLAN.md](./PLAN.md) for the step-by-step build order and current
status.
