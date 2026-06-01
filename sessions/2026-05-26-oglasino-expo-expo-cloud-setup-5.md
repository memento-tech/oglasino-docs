# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-26
**Task:** Read-only audit — find stale references and inconsistencies in the codebase against the just-completed Expo cloud setup (new project ID, ternary bundle IDs, three EAS profiles, new Google service files, env files).

## Implemented

- Read-only audit only. No code changes.

## Audit 1 — Stale Expo project ID references

**No findings.** `grep -rn "cbc496aa-bbd8-4c8e-b46e-c4c70a61f867"` returned zero matches in source code. The dead project ID has been fully removed. `app.config.ts:63` correctly references the new project ID `382cac59-45fa-420e-ab80-7fc709e6d2f3`.

## Audit 2 — Stale bundle ID references

**4 findings in 3 files:**

| # | File | Line(s) | What's stale | Severity | Recommended fix |
|---|------|---------|-------------|----------|----------------|
| 2a | `GoogleService-Info.dev.plist` | 12 | Contains `com.oglasino.dev` — legacy iOS Google service file no longer referenced by `app.config.ts` (config now points to `*.development.*`, `*.preview.*`, `*.prod.*`) | medium | Delete the file; it's orphaned |
| 2b | `google-services.dev.json` | 12, 20, 28 | Contains `com.oglasino.dev` — legacy Android Google service file, same situation as 2a | medium | Delete the file; it's orphaned |
| 2c | `jobs/image_pipeline/IMAGE-PIPELINE-RN-AUDIT.md` | 41–42 | References `com.oglasino.dev` as the dev bundle ID in a documentation table | low | Update to `com.oglasino.development` or delete if the audit doc is fully superseded |

**Notes:** The legacy `*.dev.*` files still exist on disk (dated Feb 28 and Mar 7 respectively) but `app.config.ts` was updated to reference `*.development.*`, `*.preview.*`, and `*.prod.*`. The legacy files are unreferenced dead weight that could cause confusion. They are not gitignored (`.gitignore` has no pattern for Google service files, only `.env.*`).

## Audit 3 — Two-tier assumptions in source code

**No findings worth flagging.** Four matches found; all are benign:

| File | Line | Pattern | Verdict |
|------|------|---------|---------|
| `src/components/messages/MessageInput.tsx` | 89 | `if (__DEV__) console.warn(...)` | Correct — gates debug logging. Preview build has `__DEV__` = false (production bundle), which is desired behavior. |
| `src/components/dialog/dialogs/product-creation/UploadedProductDialog.tsx` | 83 | `if (__DEV__) console.error(error)` | Same pattern, correct. |
| `src/notifications/components/PushNotificationsInit.tsx` | 146 | `Device.isDevice` | Not tier-related — checks physical device vs simulator. |
| `src/notifications/lib/pushNotificationRegister.ts` | 14 | `Device.isDevice` | Same, not tier-related. |

No `APP_ENV` checks exist in `src/` or `app/` — all `APP_ENV` branching lives in `app.config.ts`, which properly handles all three tiers (`development`, `preview`, `production`). No two-tier `isDev`/`isProd` booleans found.

## Audit 4 — Hardcoded Firebase project references

**No findings.** Zero matches for `oglasino-stage-49abb`, `oglasino-prod-7e5db`, or `oglasino-dev` in `src/` or `app/`.

## Audit 5 — firebaseClient.ts review

**Confirmed correct:**
- ✅ Reads all six Firebase config keys from env vars (apiKey, authDomain, projectId, appId, storageBucket, messagingSenderId) at lines 11–16.
- ✅ No fallback values pointing at stale projects — config object is a direct passthrough of env vars.
- ✅ No hardcoded webClientId in this file.

**webClientId consumer:** Found at `src/lib/services/authService.ts:170`. Reads via `Constants.expoConfig?.extra?.firebaseConfig?.webClientId`, sourced from `app.config.ts:73` which reads `process.env.EXPO_PUBLIC_WEB_CLIENT_ID`. The optional chaining means a missing env var degrades to `undefined` rather than crashing — Google Sign-In would silently fail. Pattern works for all three tiers as long as each env file provides `EXPO_PUBLIC_WEB_CLIENT_ID`.

**2 findings:**

| # | File | Line(s) | What's stale/wrong | Severity | Recommended fix |
|---|------|---------|---------------------|----------|----------------|
| 5a | `src/lib/client/firebaseClient.ts` | 11–27 | **No missing-env-var guard.** If any of the six env vars is missing, `initializeApp()` receives `undefined` values and Firebase fails opaquely later (unhelpful auth errors, Firestore connection failures). | medium | Add a guard before `initializeApp()` that checks all six are truthy and throws a clear error naming the missing var(s). |
| 5b | `src/lib/client/firebaseClient.ts` | 36–50 | **Dead `testConnection()` function.** Commented-out call at line 50, but the function body (with `console.info`/`console.error`) is still defined. Never called from anywhere. | low | Delete the function and the commented-out call. |

## Audit 6 — Test configuration

**No findings.** Vitest config (`vitest.config.ts`) uses `environment: 'node'` with no env file loading and no Firebase project assumption. Tests that need env vars (`src/lib/images/variants.test.ts`) set them inline via `process.env.EXPO_PUBLIC_*` assignments. No test references a specific Firebase project ID.

## Audit 7 — Scripts and tooling

**No findings.** `package.json` scripts are generic Expo commands: `start`, `reset-project`, `android`, `ios`, `lint`, `test`, `test:watch`. No hardcoded bundle IDs, project IDs, or tier names. No `scripts/` directory beyond `scripts/reset-project.js` (standard Expo template file).

## Audit 8 — Native config in app.config.ts

`app.config.ts` properly branches `bundleId`, `iosGoogleServicesFile`, and `androidGoogleServicesFile` on `APP_ENV` across all three tiers. Several other fields are NOT branched:

| # | Field | Current value | Severity | Observation |
|---|-------|--------------|----------|-------------|
| 8a | `name` (line 24) | `'Oglasino'` for all tiers | low | If dev, preview, and production builds are installed on the same device, they're indistinguishable by name. Consider `Oglasino Dev` / `Oglasino Preview` / `Oglasino`. |
| 8b | `scheme` (line 28) | `'oglasino'` for all tiers | medium | All tiers register the same `oglasino://` URL scheme. With multiple tiers on one device, deep links are ambiguous — the OS picks arbitrarily. Consider `oglasino-dev` / `oglasino-preview` / `oglasino`. |
| 8c | `icon` (line 30) | Same icon for all tiers | low | Standard practice is to badge dev/preview icons for visual distinction. Not urgent. |
| 8d | `splash` (lines 31–35) | Same for all tiers | low | Minor — some teams tint the splash per tier. Cosmetic only. |
| 8e | `infoPlist` (lines 40–42) | Only `UIBackgroundModes` — no `CFBundleURLTypes` | low | No deep link URL types registered. When deep links are added, they'll need per-tier branching. |
| 8f | `android` (lines 52–60) | No `intentFilters` | low | Same as 8e — no Android deep link intent filters. Will need per-tier config when added. |

**`updates.fallbackToCacheTimeout: 0` (line 107):** This is the EAS Updates config. Value of 0 means the app won't wait for an OTA update check before launching. Present for all tiers — reasonable default. No tier concern.

## Files touched

None — read-only audit.

## Tests

No tests run — read-only audit.

## Cleanup performed

None needed (read-only session).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- **Legacy Google service files** (`GoogleService-Info.dev.plist`, `google-services.dev.json`): orphaned by the new ternary naming (`*.development.*`, `*.preview.*`, `*.prod.*`). Left for follow-up deletion — this is a read-only audit.
- **`jobs/image_pipeline/IMAGE-PIPELINE-RN-AUDIT.md` bundle ID references:** stale documentation. Left for follow-up — may be superseded entirely by newer audit files.

## Conventions check

- Part 4 (cleanliness): N/A — no code changes this session
- Part 4a (simplicity): N/A — read-only audit
- Part 4b (adjacent observations): see "For Mastermind" below
- Part 6 (translations): N/A this session
- Other parts touched: none

## Known gaps / TODOs

None — all 8 audits completed and reported.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit
  - Considered and rejected: nothing
  - Simplified or removed: nothing

- **Summary of findings by severity:**

  | Severity | Count | Findings |
  |----------|-------|----------|
  | medium | 3 | 2a (legacy iOS Google service file), 2b (legacy Android Google service file), 5a (no missing-env-var guard in firebaseClient.ts), 8b (URL scheme not per-tier) |
  | low | 5 | 2c (stale doc bundle ID), 5b (dead testConnection function), 8a (app name not per-tier), 8c (icon not per-tier), 8d–8f (splash/deep links not per-tier) |

- **Recommended follow-up brief priority:**
  1. Delete `GoogleService-Info.dev.plist` and `google-services.dev.json` (2a, 2b) — immediate, no code risk.
  2. Add missing-env-var guard to `firebaseClient.ts` (5a) — medium priority, prevents opaque failures on misconfigured builds.
  3. Branch `scheme` per tier (8b) — medium priority, blocks deep link work if multiple tiers coexist on test devices.
  4. Branch `name` per tier (8a) — low priority, quality-of-life for testers.
  5. Delete dead `testConnection()` (5b) — low priority, pure cleanup.
  6. Update or delete `IMAGE-PIPELINE-RN-AUDIT.md` (2c) — low priority, stale doc.

- **Adjacent observation (Part 4b):** `.gitignore` has `.env.*` but no pattern for Google service files. The `*.development.*`, `*.preview.*`, and `*.prod.*` files are tracked (not gitignored). These files contain Firebase project config (API keys, project IDs) that are embedded in the app binary regardless, so tracking them is the standard Firebase recommendation. However, if the project's security posture prefers gitignoring them (and loading via EAS Secrets or similar), that's a separate decision. Severity: low. I did not change this because it is out of scope.

- **Adjacent observation (Part 4b):** `firebaseClient.ts:50` has a commented-out `// testConnection();` line — a conventions Part 4 violation (no commented-out code). Severity: low. I did not fix this because this is a read-only audit.
