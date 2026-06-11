# Session summary

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-06-10
**Task:** Pre-Build Readiness Audit (READ-ONLY) — audit `oglasino-expo` for preview/production build readiness; report findings, fix nothing.

## Implemented

- Nothing implemented. This was a read-only audit per the brief's hard rules (no mutations, no edits, no `eas env:*` writes, no `expo prebuild`).
- Ran the EAS auth gate (`eas whoami` → `oglasino`, authenticated) and executed all seven audit steps.
- Every file:line claim cross-verified with BOTH `view` (Read/cat) AND `rg`; raw command output captured in the findings report handed to Igor.

## Findings (condensed; full evidence in the report delivered to Igor)

- **Step 1 — EAS env vars:** PASS. preview `BACKEND_BASE_URL=https://api-stage.oglasino.com` (stage), production `BACKEND_BASE_URL=https://api.oglasino.com` (prod) — correctly split, plaintext/visible. Internal-token var present in BOTH environments as `INTERNAL_TOKEN` (secret, value hidden — expected).
- **Step 2 — workflow ↔ env cross-check:** PASS. `.eas/workflows/preview.yml` ceiling jobs reference `${{ env.BACKEND_BASE_URL }}` and `${{ env.INTERNAL_TOKEN }}` (lines 42–43, 63–64); both names exist in EAS `preview`. The HTTP header is `X-INTERNAL-TOKEN` (line 47/68) but the *env var* it reads is `INTERNAL_TOKEN` — consistent, no silent-failure mismatch.
- **Step 3 — .env.preview:** PASS (consistency). Exists; defines `APP_ENV` + the `EXPO_PUBLIC_*` set + GA4 keys. Deliberately does NOT contain `BACKEND_BASE_URL` or `INTERNAL_TOKEN` — those are build-server-only and live in EAS env (correct for SDK 54 cloud builds). Keys line up with EAS preview.
- **Step 4 — OTA/runtimeVersion:** PASS. `app.config.ts`: `version: '1.0.0'` (L58), `runtimeVersion.policy: 'appVersion'` (L159–160), `updates.url: https://u.expo.dev/382cac59-...` (L163), `enabled: true` (L164). `eas.json`: `appVersionSource: remote` (L4), `preview` channel `preview` (L23), `production` channel `production` (L36). Triplet: **version 1.0.0 / policy appVersion / channels preview+production**.
- **Step 5 — iOS permission strings (SOURCE):** PASS. `expo-image-picker` plugin block (`app.config.ts` L131–137) injects `cameraPermission` + `photosPermission` → these become `NSCameraUsageDescription` / `NSPhotoLibraryUsageDescription` at prebuild. No raw `NS*` keys in `infoPlist` (only `UIBackgroundModes`), which is correct — the plugin owns them. Built-`Info.plist` verification NOT possible here (needs the build artifact / on-device).
- **Step 6 — update-screen URLs:** Android PASS, iOS KNOWN-PLACEHOLDER (tracked). Both `HardUpdateScreen.tsx` and `SoftUpdateModal.tsx` route through `getStoreUrl()` in `src/lib/utils/storeUrl.ts`. Android (L6) = real Play listing `https://play.google.com/store/apps/details?id=com.oglasino`. iOS (L12) = `https://memento-tech.com` placeholder, blocked on the numeric Apple ID — already tracked in `issues.md` (open, 2026-06-09 "iOS force-update store URL is still the placeholder").
- **Step 7 — Firebase mapping:** PASS (no project crossing). preview (android `google-services.preview.json` + iOS `GoogleService-Info.preview.plist`) → `oglasino-stage-49abb` / sender `1091292210835`; production → `oglasino-prod-7e5db` / sender `958252433998`. Never crossed → no MismatchSenderId risk. **One adjacent inconsistency flagged** (non-blocking): EAS preview `EXPO_PUBLIC_FIREBASE_APP_ID=1:1091292210835:android:06eca2cd8303b35469cf66` is the `com.oglasino` Android app id within the stage project, NOT the `com.oglasino.preview` app id (`...28b98...`). Same project/sender, so FCM/auth unaffected, but the JS-SDK appId attributes the preview build to the wrong package. See "For Mastermind."

## Files touched

- None (read-only audit).

## Tests

- None run (read-only; no code changed). Lint/tsc/test not applicable to an audit session.

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change. (Audit produces findings only; the `expo-cloud-setup` pre-deploy checklist and `expo-release-readiness` blockers already capture the open items. No status flip earned by an audit.)
- issues.md: no change. The iOS-store-URL placeholder is already an open entry (2026-06-09). The EAS preview `EXPO_PUBLIC_FIREBASE_APP_ID` inconsistency is drafted in "For Mastermind" for triage — not written here (Docs/QA is sole writer).

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code touched.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one flagged (EAS preview Firebase appId points at wrong package) — see "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: none.

## Known gaps / TODOs

- Built-`Info.plist` permission-string verification and on-device permission-dialog/deny-path checks are explicitly out of this audit's reach (need the build artifact / device) — already owed in `state.md` as the pre-build blocker close-out (iPhone take-photo/gallery dialog; deny→toast→Open Settings; Android camera/gallery + POST_NOTIFICATIONS post-prebuild).
- Out-of-scope per brief and not attempted: APNs key redundancy (2HFA5WPPMM vs YC4475JBW7), router `ASSETLINKS_STAGE` SHA-256, prod push credential provisioning.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (no code).
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **Adjacent observation (Part 4b) — EAS preview `EXPO_PUBLIC_FIREBASE_APP_ID` points at the wrong package (severity: low/medium).**
  - File/source: EAS env `preview` → `EXPO_PUBLIC_FIREBASE_APP_ID=1:1091292210835:android:06eca2cd8303b35469cf66`, consumed by `app.config.ts:104` (`extra.firebaseConfig.appId`).
  - What's there: that app id belongs to the `com.oglasino` Android app registered in the **stage** project (`google-services.development.json:10–12`). The preview build's actual Android Firebase app is `com.oglasino.preview` = `1:1091292210835:android:28b9892525fcb20a69cf66` (`google-services.preview.json:54`).
  - Why it matters: same project + same sender id (1091292210835), so Auth and FCM are unaffected and there is NO MismatchSenderId risk. But the JS Firebase SDK (firebaseConfig.appId) attributes the preview build to `com.oglasino` rather than `com.oglasino.preview` — Analytics/Installations app attribution and any appId-keyed Firebase product would be misfiled for preview. Native @react-native-firebase reads the google-services file (correct), so the split is JS-config vs native-config.
  - I did not fix this because it is out of scope (read-only audit, and editing EAS env is a brief hard-rule prohibition). Suggested resolution: update EAS preview `EXPO_PUBLIC_FIREBASE_APP_ID` to the `com.oglasino.preview` app id (`...28b98...`). Needs Igor (EAS env write).

- **Suggested deliverable framing:** Blockers before a PRODUCTION build = none found in the audited surface (production env, OTA, Firebase prod mapping, Android store URL all PASS). The iOS store-URL placeholder is a production-build *iOS* blocker only when iOS ships to App Store, already tracked. The preview Firebase appId item is preview-only and non-blocking.

- No config-file drafts pending application. Closure gate: no unstated config-file dependency — confirmed.
