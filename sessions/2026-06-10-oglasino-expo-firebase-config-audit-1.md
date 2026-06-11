# Session summary

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-06-10
**Task:** Decide whether the JS Firebase setup (`EXPO_PUBLIC_FIREBASE_*` → `firebaseConfig` in `app.config.ts` → `src/lib/client/firebaseClient.ts`) is LOAD-BEARING or VESTIGIAL, and identify which SDK powers each Firebase feature (auth / messaging / analytics / firestore / storage). READ-ONLY.

## Implemented

- Nothing implemented. Read-only audit per the brief's hard rules (no edits, no commits, no installs).
- Every file:line claim cross-verified with BOTH `rg` AND `view`/`sed`; raw output captured in the report delivered to Igor.

## Findings (condensed; full evidence in the report delivered to Igor)

- **VERDICT: LOAD-BEARING.** `firebaseClient.ts` is imported by 11+ non-test src files. Its `auth` and `db` exports power authentication and chat/notifications Firestore. Strongest single proof: `src/lib/services/authService.ts:14` imports `{ auth, db }` and uses them for email/password + Google sign-in and the user doc read/write.
- **Per-feature SDK:**
  - **Auth → JS `firebase/auth`.** `firebaseClient.ts:48` `initializeAuth`; consumed in `authService.ts`, `authStore.ts:2`, `DeleteAccountConfirmationDialog.tsx:11`, `ForegroundRevalidationInit.tsx`, etc. No `@react-native-firebase/auth` installed.
  - **Messaging/push → `expo-notifications`.** `pushNotificationRegister.ts:3`, `PushNotificationsInit.tsx:7`. No `@react-native-firebase/messaging`, no JS `firebase/messaging` anywhere.
  - **Analytics → native `@react-native-firebase/analytics`.** `analyticsClient.ts:1`, `track.ts:1`. Reads google-services/plist, NOT the JS config.
  - **Firestore → JS `firebase/firestore`.** `firebaseClient.ts:51` `getFirestore`; consumed in `useChatBlockStore.ts`, `useChatListStore.ts`, `useActiveChatStore.ts`, `useChatNavStore.ts`, `useNotificationStore.ts`, `authService.ts`. No native firestore.
  - **Storage → NOT USED.** `storage` export at `firebaseClient.ts:52` is imported by zero files. Image uploads go through `expo-file-system` PUT (`uploadPrimitive.ts`), not Firebase Storage.
- **JS `appId` specifically:** the JS config object is load-bearing, but the **`appId` value has no runtime effect**. JS Auth + Firestore consume only `apiKey`/`authDomain`/`projectId`; `appId` is used by JS Analytics/Installations, neither of which runs (analytics is native, reading google-services/plist). The env var `EXPO_PUBLIC_FIREBASE_APP_ID` must merely be **non-empty** or `firebaseClient.ts:31-35` throws on import. The `extra.firebaseConfig.appId` in `app.config.ts:104` is read by nobody — `authService.ts:192-195` reads only `.webClientId` from `extra.firebaseConfig`.
- **google-services wiring confirmed:** `app.config.ts:72` `ios.googleServicesFile` (prod/preview/development plist switch, lines 14-19) and `app.config.ts:93` `android.googleServicesFile`; `@react-native-firebase/app` plugin at line 147.

## Files touched

- None. Read-only.

## Tests

- None run / none changed. Read-only audit; no touched code paths.

## Cleanup performed

- None needed.

## Config-file impact

- No change. No edit to any of the four `oglasino-docs` config files is required by this audit. This was a decision-support investigation; no feature adopted, so no Expo backlog row changes.

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): N/A — no code written.
- Part 4a (simplicity): N/A.
- Part 4b (adjacent observations): two dead-weight items observed but NOT changed (read-only): (a) the `storage` export in `firebaseClient.ts:52` plus its `getStorage` import (line 9) are unused; (b) `extra.firebaseConfig` in `app.config.ts` carries six fields but only `webClientId` is ever read. Flagged here for Mastermind; no action taken.
- Hard rules honored: no commits/pushes/branch changes, no `app.config.ts` edits, no cross-repo edits, no config-file writes.

## Known gaps / TODOs

- None. No `TODO`/`FIXME` introduced.

## For Mastermind

- **The appId concern is moot at runtime.** Whatever cross-platform / preview-vs-prod mismatch exists in `EXPO_PUBLIC_FIREBASE_APP_ID`, it has zero functional effect: the JS SDK never uses `appId` for the two things it actually does (auth + firestore), and the only hard requirement is that the var be non-empty (else `firebaseClient.ts` throws at import). Native analytics gets its app id from google-services.json / GoogleService-Info.plist, not from this var. So `appId` can be deprioritized as a build blocker.
- **Optional cleanup candidates (not done — read-only, and `app.config.ts` is config-locked for me):**
  1. `firebaseClient.ts` `storage`/`getStorage` (lines 9, 52) are dead — uploads use `expo-file-system`. Safe to drop.
  2. `app.config.ts:100-107` `extra.firebaseConfig` only needs `webClientId`; the other five fields are unread (the live config comes from `process.env` directly in `firebaseClient.ts`). If trimmed, that touches `app.config.ts` — needs an explicit brief since it's a config-locked file for Mobile.
