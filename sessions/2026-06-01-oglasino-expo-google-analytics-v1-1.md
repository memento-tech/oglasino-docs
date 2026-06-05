# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-06-01
**Task:** Read-only audit — GA4 mobile firing surfaces (13 events) + `wasRegister` wire shape; produce `.agent/audit-google-analytics-v1.md`.

## Implemented

- Read-only Phase-2 audit. No source touched. Produced `.agent/audit-google-analytics-v1.md` answering all six brief items.
- Inventoried the mobile firing surface + locally-available params (and PII alternative) for each of the 13 events against the web catalog in `features/google-analytics-v1.md`.
- Traced `syncUserToBackend` / `AuthUserDTO` and classified the `wasRegister` wire shape.
- Confirmed the dead `firebaseAnalytics.ts`, the greenfield native-analytics state, and the readiness of the `isAnalyticsConsentGranted()` gate.

## Files touched

- `.agent/audit-google-analytics-v1.md` (new, +~210 lines) — the deliverable.
- (Plus this summary + `last-session.md`.) No source/test/config files changed.

## Tests

- Not run — read-only audit, zero code changes. Nothing to lint/typecheck/test.

## Headline findings (full detail in the audit)

- **`wasRegister` = present-on-wire-but-dropped-by-mobile.** Mobile `AuthUserDTO` type lacks the field and `syncUserToBackend` (`authService.ts:143-152`) discards it; backend already emits it on the shared `/auth/firebase-sync` route. Discriminating `sign_up`/`login` is zero-backend-work — mobile type + plumbing only.
- **Native analytics is greenfield:** only the web `firebase@^12.10.0` JS SDK is installed; no `@react-native-firebase/analytics`, no `track`/`logEvent` wrapper, no ATT.
- **`exception` has no surface today:** no error boundary and no global error handler exist anywhere; the F brief must build both (RN-specific — web's `error.tsx`/`global-error.tsx`/`window` listeners don't port).
- **`CallUserButton` needs `productId` plumbed** from `ProductFunctions.tsx:33` (same as web); it has `userId` (= seller_id) only.
- **Auth firing is multi-site** (4 `setUser` sites in `authStore.ts`), unlike web's single hydrator — a chokepoint + cold-start-resync exclusion must be chosen in the F brief.
- **`firebaseAnalytics.ts`** is a web-SDK vestige (`typeof window` guard) with zero importers — flagged for deletion.

## Cleanup performed

- None needed (read-only audit; no code touched).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change required by this session. The Expo-backlog "Google Analytics v1" row stays `not-started` — this is the Phase-2 audit, not adoption; no status flip is warranted until Phase-5 mobile code lands. Explicitly: no config-file edit is owed by this session.
- issues.md: no change. (Findings are captured in the audit deliverable for the F brief; no standalone bug entries warranted — they are scoped feature work, not loose defects.)

## Obsoleted by this session

- Nothing. (The audit *recommends* deleting `src/lib/client/firebaseAnalytics.ts` — a pre-existing dead web-SDK vestige with zero importers — but per the brief's read-only scope it is flagged, not deleted. The F-implementation session deletes it.)

## Conventions check

- Part 4 (cleanliness): confirmed — no code added, nothing to clean.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): flagged in "For Mastermind" (dead file; price string→number; CurrencyDTO object).
- Part 6 (translations): N/A this session (no keys touched).
- Part 11 (trust boundaries): confirmed N/A — GA4 is client telemetry; no event payload feeds a server-side decision (consistent with web's GA4 close-out).
- Other parts: Part 8 (routes reusable) — confirmed; mobile reuses web's `/auth/firebase-sync`, no mobile-specific route implied.

## Known gaps / TODOs

- Three surfaces were not deep-read, only inferred, and are flagged in the audit for the F brief to confirm: `app/owner/dashboard/user.tsx` and `app/owner/dashboard/products/[productId].tsx` (assumed structured `parseServiceError` form-error shape per the Φ4 contract), and the create-vs-edit discrimination of `AddUpdateProductDialog`. No TODO/FIXME left in code.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no abstractions/config/code introduced.
  - Considered and rejected: nothing — no implementation choices made this session.
  - Simplified or removed: nothing — flagged the dead `firebaseAnalytics.ts` for the F brief rather than deleting it (read-only scope).
- **Adjacent observations (Part 4b):**
  - `src/lib/client/firebaseAnalytics.ts` — dead web-SDK vestige, zero importers, would always return `null` on device. Severity low (dead, unreferenced). Not fixed: out of scope (read-only) — delete in the F-implementation session.
  - `ProductOverviewDTO.price` is typed `string` and `NewProductRequestDTO.currency` is a `CurrencyDTO` object; the spec's `product_view`/`product_create_completed` expect numeric `price` and ISO-string `currency`. Severity low (the F brief handles the coercion). Flagged so it isn't missed.
- **Key decisions the F brief must make (surfaced, not decided here):** (1) the auth firing chokepoint among 4 `setUser` sites + cold-start `onIdTokenChanged` re-sync exclusion; (2) `filter_change` has no URL-sync equivalent — pick the `FilteredProductList.filtersData` effect vs a FiltersDialog apply point; (3) register/login synthetic `error_code` set must come from imperative `validateForm` branches (no Zod on mobile); (4) `exception` requires building an RN error boundary + `ErrorUtils.setGlobalHandler` + rejection tracking from scratch.
- **Config-file drafts pending:** none. No edits to any of the four files are owed by this session (verified at closure per the Part 5 gate).
