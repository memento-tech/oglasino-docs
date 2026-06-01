# Session summary

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-05-23
**Task:** Read-only audit of the Google Analytics state on mobile against `features/google-analytics-v1.md`. Inventory what would be needed to ship analytics — SDK choice, event taxonomy parity, consent gating, ATT prompt, identifier strategy.

## Implemented

- Produced `.agent/audit-expo-readiness-google-analytics.md` with all 15 required sections.
- Confirmed zero analytics instrumentation exists on mobile (no SDKs, no event logging, no screen tracking).
- Inventoried dead analytics code (`firebaseAnalytics.ts`, vestigial `measurementId` config, dead `allowPreferenceCookies` field).
- Mapped all 12 GA v1 events to their mobile equivalent surfaces with file paths.
- Documented SDK choice inputs: `@react-native-firebase/analytics` is the only viable candidate; `expo-firebase-analytics` is deprecated since SDK 48.
- Documented ATT and consent prerequisites for iOS and Android.
- Identified `wasRegister` gap (backend ships it, mobile doesn't consume it).
- Identified `CallUserButton` `productId` prop gap (web added it, mobile hasn't).
- Confirmed trust boundaries clean for analytics instrumentation.

## Files touched

- `.agent/audit-expo-readiness-google-analytics.md` (new, +~250 lines)

## Tests

- N/A — read-only audit, no code changes.

## Cleanup performed

none needed — read-only audit.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

nothing — read-only audit.

## Conventions check

- Part 4 (cleanliness): N/A this session. Adjacent dead-code findings flagged in the audit's Section 11.
- Part 4a (simplicity): N/A — read-only audit, no code added or removed.
  - Added (earned complexity): nothing
  - Considered and rejected: nothing
  - Simplified or removed: nothing
- Part 4b (adjacent observations): flagged in audit Section 12 — dead `allowPreferenceCookies` on `AuthUserDTO`, commented-out Facebook login in `authService.ts`, `testConnection` debug code in `firebaseClient.ts`, dead `ConsentData`/`GlobalCookie` types. All out of scope for this audit; assigned to their respective parallel audits (consent-mode-v2, general).
- Part 6 (translations): N/A this session.
- Part 11 (trust boundaries): touched — explicit check in audit Section 10. Clean finding.
- Other parts touched: Part 8 (architectural defaults — confirmed mobile reuses same backend routes), Part 9 (stack reference — Expo SDK 54, expo-router 6.0.23).

## Known gaps / TODOs

- none — all 15 sections populated per the brief's definition of done.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **ATT is the hard floor for iOS analytics.** Without `expo-tracking-transparency` and the ATT prompt, Apple rejects any build with tracking SDKs. This must ship before or concurrently with the analytics SDK.

- **Mobile consent UI is a blocking dependency.** Both ATT (iOS) and GDPR (EU users) require consent gating. No consent surface exists on mobile today. The consent feature chat must precede or parallel the analytics feature chat.

- **`expo-firebase-analytics` is dead.** Removed from Expo SDK in SDK 48 (2023). The only viable SDK is `@react-native-firebase/analytics` via config plugins + EAS Build.

- **Two gaps need closing before mobile analytics can mirror web's event taxonomy:**
  1. `wasRegister` from `/auth/firebase-sync` — mobile's `syncUserToBackend` and `AuthUserDTO` don't include it.
  2. `CallUserButton` needs a `productId` prop — web added this in GA v1 brief 6; mobile hasn't followed.

- **Three questions for the future mobile-analytics chat:**
  1. Does mobile share web's GA4 measurement ID (via Firebase project linking), or does it need its own?
  2. Should mobile consent follow the four-signal Consent Mode v2 model or a simplified binary model (Firebase Analytics only has `setAnalyticsCollectionEnabled(bool)`)?
  3. Should analytics wait for a full consent UI, or can it ship with ATT-only gating (legally incomplete for EU users)?

- **Adjacent dead-code observations (not fixed, out of scope):**
  - `allowPreferenceCookies` on `AuthUserDTO` — dead, backend removed it during Consent Mode v2. Consent-mode-v2 audit scope.
  - `ConsentData` and `GlobalCookie` types — pre-v2 web shapes, likely dead on mobile. Consent-mode-v2 audit scope.
  - `loginWithFacebookFirebase` entirely commented out in `authService.ts:200-228`. General audit scope.
  - `testConnection()` commented-out call with `console.info`/`console.error` in `firebaseClient.ts:38-50`. General audit scope.
