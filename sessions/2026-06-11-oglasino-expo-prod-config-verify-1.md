# Session summary

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-06-11
**Task:** Pre-build verification of the PRODUCTION-tier config for the iOS build (read-only audit — confirm production profile resolves to the right bundle, version, and prod (not stage) Firebase/google-services files before cutting a slow iOS production build).

## Implemented

- Nothing changed on disk. This was a read-only VERIFY-AND-REPORT brief.
- Verified the five requested items against `eas.json`, `app.config.ts`, and `GoogleService-Info.prod.plist`, each claim cross-checked with both a file Read and an `rg`/grep (the two agreed on every item).
- Result: all 16 sub-checks PASS, zero FLAGs. The production tier is safe to build from a config standpoint.

## PASS/FLAG table

| # | Item | file:line | Literal value | Verdict |
|---|------|-----------|---------------|---------|
| 1 | Production profile, store distribution | `eas.json:34–35` | `"production": { "distribution": "store"` | PASS |
| 1 | APP_ENV for production | `eas.json:37–39` | `"env": { "APP_ENV": "production" }` | PASS |
| 1 | OTA channel | `eas.json:36` | `"channel": "production"` | PASS |
| 1 | buildReactNativeFromSource | `app.config.ts:156` (expo-build-properties iOS) | `buildReactNativeFromSource: true` | PASS |
| 1 | iOS-specific prod bits | `eas.json:40–42` | `"ios": { "resourceClass": "m-medium" }` (no simulator flag) | PASS |
| 2 | Bundle id expression | `app.config.ts:7–9` | `ENV === 'production' ? 'com.oglasino' : …` | PASS |
| 2 | Resolved prod bundle | `app.config.ts:9` applied at `:71` | `com.oglasino` (not `.preview`/`.development`) | PASS |
| 3 | Marketing version | `app.config.ts:58` | `version: '1.0.0'` | PASS |
| 3 | iOS buildNumber management | `eas.json:4` + `eas.json:46` | no static buildNumber; `appVersionSource: "remote"` + production `autoIncrement: true` → EAS-managed | PASS |
| 4 | iOS googleServicesFile selector | `app.config.ts:14–16` applied at `:73` | prod → `./GoogleService-Info.prod.plist` | PASS |
| 4 | Prod plist BUNDLE_ID | `GoogleService-Info.prod.plist` | `com.oglasino` | PASS |
| 4 | Prod plist PROJECT_ID | `GoogleService-Info.prod.plist` | `oglasino-prod-7e5db` | PASS |
| 4 | Prod plist GCM_SENDER_ID | `GoogleService-Info.prod.plist` | `958252433998` | PASS |
| 4 | Prod plist GOOGLE_APP_ID (iOS) | `GoogleService-Info.prod.plist` | `1:958252433998:ios:c851d3d0d8d7d9e969b91f` | PASS |
| 5 | NSCameraUsageDescription | `app.config.ts:135` (expo-image-picker plugin → injected at prebuild) | `"Allow Oglasino to use your camera to take photos for your listings."` | PASS |
| 5 | NSPhotoLibraryUsageDescription | `app.config.ts:136` (expo-image-picker plugin → injected at prebuild) | `"Allow Oglasino to access your photos to add them to your listings."` | PASS |

### Verification notes

- The two iOS permission strings are NOT in `ios.infoPlist` (which carries only `UIBackgroundModes: ['remote-notification']`). They come from the `expo-image-picker` config plugin (`cameraPermission`, `photosPermission`) and are injected into `Info.plist` at prebuild. This confirms the photo-library string fix the records tracked as "code-complete pending prebuild verification" is in fact wired in config (line 136).
- Brief item 4 asked to confirm Firebase app `oglasino-ios-prod`: the plist carries no app *name* field, but `GOOGLE_APP_ID ...:ios:...` under `PROJECT_ID oglasino-prod-7e5db` confirms it is the iOS app within the prod Firebase project. Eyeball-confirmed prod, not stage.
- The only uncommitted change to `app.config.ts` (git diff) is `+ platforms: ['ios', 'android']` at line 62 — irrelevant to every verification item. Nothing staged.

## Files touched

- None (read-only).

## Tests

- Ran: none required (no code change; verification only). `rg`/grep used as cross-checks alongside file reads per the standing tool-reliability discipline.
- Result: N/A
- New tests added: none

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change required by this session. (Optional, for Mastermind to weigh — see "For Mastermind": a note could be added that the production-tier build config passed pre-build verification on 2026-06-11. Not authored here; drafted below if wanted.)
- issues.md: no change

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no disk changes, nothing to clean.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one low-severity observation flagged in "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: Part 9 (Versioning) — confirmed `expo` marketing version `1.0.0`, iOS buildNumber EAS-managed via `autoIncrement` + remote `appVersionSource`, consistent with the state.md Versions table and decisions.md 2026-06-09.

## Known gaps / TODOs

- This brief covers config presence only. On-device behavior of the camera/photo permission dialogs (the deny→toast→Open Settings flow tracked in state.md 2026-06-09) is explicitly out of scope and still owed at prebuild/on-device.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — no code written.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- **Verdict for the brief's headline question — is the production tier safe to build?** Yes. All five verification items pass with zero FLAGs. The production EAS profile sets `APP_ENV=production`, store distribution, OTA channel `production`, and EAS `autoIncrement`; the bundle resolves to exactly `com.oglasino`; marketing version is `1.0.0` with iOS buildNumber EAS-managed (remote source); the iOS Firebase wiring points at the prod plist (`oglasino-prod-7e5db`, BUNDLE_ID `com.oglasino`, sender `958252433998`), not stage; and both iOS camera/photo-library permission strings are present and non-empty via the `expo-image-picker` plugin. No config blocker remains for cutting the iOS production build.
- **Adjacent observation (Part 4b), low severity:** `app.config.ts` currently has one uncommitted line on `dev` (`+ platforms: ['ios', 'android']`, line 62). Harmless and arguably correct, but it is an uncommitted edit to a config file riding into the production build. Worth a deliberate commit decision by Igor before the build, so the built binary corresponds to a committed config. File: `app.config.ts:62`. I did not change this — out of scope and config-file edits are forbidden without an explicit brief.
- **Optional config-file draft (only if Mastermind wants the verification recorded), for Docs/QA to apply to `state.md`:** under the Expo cloud setup / pre-deploy area, a line such as: "Production-tier iOS build config pre-build-verified 2026-06-11 (`oglasino-expo` prod-config-verify-1): production EAS profile → bundle `com.oglasino`, version `1.0.0` (EAS-managed buildNumber), prod Firebase plist `oglasino-prod-7e5db`, camera/photo permission strings wired — zero FLAGs, safe to build." Not applied here (engineer agents do not write the four config files); drafted for the upstream-drafter path. If Mastermind judges it unnecessary, no edit is required.
