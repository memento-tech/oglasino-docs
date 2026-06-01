# Session summary

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-05-23
**Task:** Inventory every piece of the admin surface in mobile. Identify entry points, files to delete, coupling to non-admin code, and the role-gating mechanism. Estimate removal complexity.

## Implemented

- Completed read-only audit of the admin surface in `oglasino-expo`
- Inventoried 48 admin-only files across routes (12), components (21), services (7), types (7), and navigation (1)
- Identified 13 coupling cases across 11 shared files where admin code intersects non-admin code
- Documented the role-gating mechanism (`isAdminUser()` backend call, no client-side role storage)
- Estimated removal complexity at 1 session (low-medium) — all coupling is surgical line deletions, no structural redesign

## Files touched

- `.agent/audit-expo-readiness-admin-removal.md` (new, +audit output)

## Tests

- No tests run (read-only audit, no code changes)

## Cleanup performed

- none needed (read-only audit)

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): confirmed — no code changes, no cleanup to perform
- Part 4a (simplicity): N/A — read-only audit
- Part 4b (adjacent observations): 3 observations flagged in "For Mastermind"
- Part 6 (translations): N/A this session
- Part 11 (trust boundaries): confirmed — admin access is gated by `isAdminUser()` backend call; no client-side trust assumptions

## Known gaps / TODOs

- `src/lib/services/admin/suggestionsService.ts` vs `src/lib/services/suggestionsService.ts` — both contain `getReactAdminSuggestions`. Could not determine if one imports from the other or if they are independent duplicates without reading both files' full import chains. The removal brief should verify which is canonical and delete accordingly.
- `SelectFilter.tsx` uses `ADMIN_PAGES` namespace for `filter.all.label` — the removal brief must either seed this key in a non-admin namespace (backend cross-repo) or verify it already exists there.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit)
  - Considered and rejected: nothing
  - Simplified or removed: nothing

- **Adjacent observation (Part 4b):** `oglasino-dev-firebase-adminsdk-fbsvc-002e6b2f58.json` — a Firebase Admin SDK service account key file at the repo root. NOT part of the admin UI surface. Should NOT be deleted in admin removal. However, it appears to be a credential file committed to the repo. File path: repo root. Severity: medium. I did not fix this because it is out of scope.

- **Adjacent observation (Part 4b):** `src/components/admin/reviews.tsx/` — a directory named with a `.tsx` extension. Unusual naming that could confuse file explorers. File path: `src/components/admin/reviews.tsx/`. Severity: low. Moot after admin removal (entire directory deletes). I did not fix this because it is out of scope.

- **Adjacent observation (Part 4b):** `SelectFilter.tsx` uses the `ADMIN_PAGES` translation namespace for a semantically generic key (`filter.all.label`). After admin removal, the namespace reference must change. Requires a backend translation seed (one key, four locales). File path: `src/components/filters/SelectFilter.tsx:35`. Severity: low. I did not fix this because it is out of scope.

- **Removal brief scope recommendation:** The removal brief should include: (1) delete the 48 admin-only files, (2) perform the 13 coupling edits, (3) seed `filter.all.label` in `DASHBOARD_PAGES` or `COMMON_SYSTEM` namespace (backend cross-repo, or flag for Igor), (4) run `npx tsc --noEmit`, `npm run lint`, `npm test` to verify clean removal. One session is sufficient.
