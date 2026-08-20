<!-- BEGIN:nextjs-agent-rules -->

# This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` (resolved from this file's directory; in monorepos the `next` package may not be visible from the repo root) before writing any code. Heed deprecation notices.

This block is written and re-added by `next dev` — verify at `node_modules/next/dist/server/lib/generate-agent-files.js`. Removing it from a diff only re-creates the uncommitted change; committing it with your work keeps the tree clean.

<!-- END:nextjs-agent-rules -->

# Supabase is central to this project

Auth, the database, RLS — everything backend here runs on Supabase. Before writing or changing anything that touches it (schema/migrations, RLS policies, security-definer functions, indexes, auth flows, Storage, Realtime, debugging a Postgres/RLS/auth error), load the `supabase` skill, and for anything that lives in the database specifically (tables, policies, functions, migrations, queries) also load `supabase-postgres-best-practices` before writing the SQL.

## Two Supabase projects, dev and prod, from day one

- `washboard-dev` (ref `cxxsmzkatddjaxmszwfo`, region `eu-north-1`) — `.env.local`, everyday work.
- `washboard-prod` — not created yet, deferred to Phase 10 (deploy) per `PLAN.md`. Will load from `.env.production.local`, only for `next build`/`next start` in production and Vercel's Production environment.
- Both live under org `washboard` (`xghhayakyvtifqtbllna`), a dedicated org created specifically for this project under account `joaoeffting1990@gmail.com` — its own fresh free-tier 2-active-project cap, same reasoning `coffeedex` used (separate org, not separate paid tier) to avoid a Pro upgrade on a hobby project. Note this is a **different** Supabase account than `coffeedex`'s (`joaopauloeffting@gmail.com`) — not a mistake, just worth knowing if the two projects are ever compared side by side.

## No DB write credentials, no service-role key

There's no `SUPABASE_SERVICE_ROLE_KEY` in this project, on purpose — don't add one. (The dev project's `service_role`/`secret` keys were fetched once during Phase 1 setup via a temporary Management API token and immediately discarded — never written to `.env.local` or anywhere else.)

- **Applying schema changes**: write the change as a standalone `supabase-<feature-name>.sql` file at the repo root, ask the user to run it in the Supabase SQL editor (dev first, then prod once `washboard-prod` exists), then run `npm run gen:types` to pick up the new schema before continuing. Code that depends on a not-yet-applied column/function is expected to show a type error in the meantime — that's normal, not a bug to work around with `any`.
- **Anything needing elevated privileges** — write a `security definer` Postgres function that captures `auth.uid()` into a local variable once and scopes every operation to that one id, granted via `grant execute on function ... to authenticated`.

## Verifying changes: never touch the user's running dev server

If the user already has `npm run dev` running (port 3300 — see `package.json`), don't kill it, restart it, or reuse its port to test something. Use an isolated `git worktree` on a separate port instead: `git worktree add /tmp/washboard-test-wt -b test-<feature>`, copy over the changed files plus `.env.local` and a copy of `node_modules` (rsync, not a symlink — Turbopack panics on a symlink pointing outside the worktree root), then `PORT=<other> npx next dev -p <other>` from that worktree directory.

# New features need Playwright integration test coverage

This is a portfolio project — test coverage is part of what it's demonstrating, not optional polish, and it's in scope from Phase 1 (not deferred, unlike the `laundry-booking` study spec this project evolved from — see `SPEC.md`'s "Why this project"). When adding a feature (a new page, a Server Action that mutates data, a meaningful interaction), add a spec in `tests/` alongside it rather than treating tests as a separate follow-up pass. See `tests/README.md` for how the suite is set up and run (`npm run test:e2e`).

A few things that make this suite different from a typical setup, worth knowing before adding to it:

- No direct DB seeding or reads — no service-role key exists here (see above), so specs interact only through the real UI, same as a real resident/admin would.
- Multi-actor by nature, unlike `coffeedex`'s single-shared-account suite: most scenarios need an admin and at least one resident in the same building, and the Phase 6 concurrency spec specifically needs **two** residents booking the same slot at once. `tests/global-setup.ts` (not written yet — build it when Phase 2/3 need it) will need to manage more than one throwaway account/`storageState`.
- Only ever run this against `washboard-dev` (the suite is hardcoded to `localhost:3300`, which always runs against `.env.local`) — never point it at prod, except the one explicit Phase 6 re-run against prod called out in `PLAN.md`'s Phase 10.

# Never commit or push without being explicitly asked

Finishing and verifying a change is not the same as being told to commit it. Leave the working tree as-is and say the change is ready — let the user decide when it becomes a commit, and never push without being asked either.

# Git identity

Checked and set correctly before the first commit: `git config user.email` → `joaoeffting@gmail.com` (local override, not the work email that leaks in via global config), `git config user.name` → `joaeff`. Matches the already-corrected history on both `storyloom` and `coffeedex` — not the near-miss dotted variant that caused `coffeedex`'s original misattribution incident. If a fresh clone or a new machine ever shows a different email, fix it with a **local** override (no `--global`) before the first commit from that clone.

# The booking loop is the load-bearing mechanism — don't "fix" it into something worse

Two things make the core booking loop correct, and both are deliberate, not accidental simplicity:

- **The double-booking guard is a Postgres unique constraint on `bookings (machine_id, date, slot_index)`, not app-layer locking.** Two concurrent inserts for the same slot: one succeeds, one hits a unique-violation that the Server Action catches and turns into "someone just booked this — pick another slot." If a future change seems to need a lock, a transaction-level `SELECT ... FOR UPDATE`, or a pre-check-then-insert pattern, that's very likely reintroducing the exact race condition this constraint exists to prevent — the fix is almost always "let the constraint do it," not more code.
- **Optimistic UI (TanStack Query) is never the source of truth.** A slot showing "yours (pending)" on click is a UX nicety layered on top of the constraint + Supabase Realtime feed, which are what's actually real. Don't let a future refactor start trusting client state before the server confirms it — see `SPEC.md`'s product principles.
