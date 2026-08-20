# Playwright integration tests

Run with `npm run test:e2e` (loads `.env.local`, starts the dev server on
port 3300 if one isn't already running, runs the suite against it).

Rules, per `SPEC.md`'s Testing section and the root workspace conventions:

- No direct DB seeding or reads — no service-role key in this project.
  Specs interact only through the real UI, same as a real
  resident/admin would.
- Only ever run against the dev Supabase project. Never point this suite
  at prod, except the one explicit re-run of the Phase 6 concurrency test
  called out in `PLAN.md`'s Phase 10.

## Test accounts

Not wired up yet — Phase 1 only has a homepage placeholder, nothing to
log into. Once Phase 2 (auth) lands, add a `global-setup.ts` here.

Unlike `coffeedex` (one shared throwaway account is enough — everything
there is single-user), washboard's core scenarios are inherently
multi-actor: an admin and at least one resident in the same building, and
for the Phase 6 concurrency spec specifically, **two** residents booking
the same slot at once. So `global-setup.ts` will need to sign up (or
reuse) more than one throwaway account and save more than one
`storageState` — decide the exact shape when Phase 2/3 actually need it,
document it here once built.
