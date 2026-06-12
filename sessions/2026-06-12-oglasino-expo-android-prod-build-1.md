# Session summary

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-06-12
**Task:** Read-only production build-config audit ahead of the Android prod AAB. No edits — report findings only. Every file:line claim verified with BOTH a file read AND rg.

## Implemented

Read-only audit. No code changed. Findings as checklist A–E, each PASS or FLAG with file:line evidence (every claim cross-checked with Read + rg + git).

### A. APP_ENV → identity resolution in app.config.ts
- `process.env.APP_ENV` read at `app.config.ts:4`.
- Android `package` = `com.oglasino` (prod): `app.config.ts:8-9` → assigned `app.config.ts:87`. **PASS**
- Android `googleServicesFile` = `./google-services.prod.json`: `app.config.ts:21-23` → assigned `app.config.ts:94`. **PASS**
- API base URL: NOT resolved in app.config.ts. From `EXPO_PUBLIC_API_URL` (`src/lib/config/api.ts:22`); prod value `https://api.oglasino.com/api` in `.env.production:5`. **PASS on value, see D for delivery.**
- channel: NOT in app.config.ts. `eas.json:36` `"channel": "production"`. **PASS**
- No prod value resolves to stage/dev or undefined inside app.config.ts.

### B. Production google-services.json — **PASS**
- `google-services.prod.json` (ref `app.config.ts:23`): project_id `oglasino-prod-7e5db` (`:4`), package_name `com.oglasino` (`:12`), project_number `958252433998` (`:3`). git-tracked → reaches cloud build.
- Informational: prebuilt `android/app/google-services.json:3-4` is the STAGE file (`oglasino-stage-49abb`). Gitignored generated folder (`.gitignore:44`); EAS prebuild regenerates from prod file. Not a prod-path blocker.

### C. eas.json build.production — **PASS (all five)**
- env.APP_ENV=production `eas.json:38`; channel=production `eas.json:36`; android.buildType=app-bundle `eas.json:44`; cli.appVersionSource=remote `eas.json:4`; autoIncrement=true `eas.json:46`.

### D. Production env vars — **FLAG**
- APP_ENV: `eas.json:38` (production) — reaches build. ✅
- All `EXPO_PUBLIC_*` (API_URL, six FIREBASE_*, WEB_CLIENT_ID, CDN_URL, WATERMARK_ENABLED, GA4_*) sourced only from `.env.production`.
- `.env.production` is **gitignored + untracked** (`.gitignore:35` `.env.*`; only `!.env.example`; `git check-ignore` IGNORED; `git ls-files` shows only `.env.example`). No `.easignore`. → file NOT uploaded to cloud EAS build → `EXPO_PUBLIC_*` undefined at build **unless set as EAS dashboard env vars** (not in eas.json, not verifiable from repo).
- Consequence: prod app throws at boot — `firebaseClient.ts:11-35` (asserts six firebase vars) and `api.ts:24-26` (throws if API URL unset).
- Values in `.env.production` themselves are correct prod values (`oglasino-prod-7e5db`, `api.oglasino.com`, `cdn.oglasino.com`). Gap is delivery to a cloud build, not the values.

### E. Android permissions for core flows — **FLAG**
- Prebuilt `android/app/src/main/AndroidManifest.xml:2-7`: INTERNET, READ_EXTERNAL_STORAGE, RECORD_AUDIO, SYSTEM_ALERT_WINDOW, VIBRATE, WRITE_EXTERNAL_STORAGE.
- `android.permission.CAMERA` ABSENT. expo-image-picker plugin never adds it — only blocks when `cameraPermission===false` (`node_modules/expo-image-picker/plugin/build/withImagePicker.js:15-17,39-41`); adds RECORD_AUDIO (`:11,:35`). READ/WRITE_EXTERNAL_STORAGE from expo-file-system (`withFileSystem.js:7-8`). Config-plugin-deterministic → fresh EAS prod prebuild produces same set.
- App uses camera: `launchCameraAsync` (`ImagesImport.tsx:105`) gated on `ensureCameraPermission()` → `requestCameraPermissionsAsync()` (`imagePermissions.ts:7`), hard deny-and-return at `ImagesImport.tsx:100-103`. On Android, undeclared permission auto-denies → take-photo flow blocked on Android. (high)
- READ_MEDIA_IMAGES (Android 13+) absent; gallery gated on `requestMediaLibraryPermissionsAsync()` (`imagePermissions.ts:15`, `ImagesImport.tsx:113`). Only legacy READ_EXTERNAL_STORAGE present. Risk of gallery gate denying on Android 13+; on-device check needed. (medium)

## Files touched

- None (read-only audit).

## Tests

- Not run — read-only audit, no code changed.

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change applied by me. Suggested issues.md entries drafted in "For Mastermind" (D and E findings). I do not write the four config files.
- issues.md: no change applied by me. Two drafts proposed below for Docs/QA.

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code changed.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): two flagged in "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: Part 10 Phase 2 (read-only audit); Part 2/Part 3 hard rules (no edits, no cross-repo writes, no config-file writes) — confirmed.

## Known gaps / TODOs

- Cannot verify EAS dashboard Environment Variables from the repo — the D flag's resolution depends on whether the `EXPO_PUBLIC_*` prod values are set there. On-device verification owed for the E camera/gallery gating on real Android (esp. Android 13+).

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — no code changed.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **FLAG D (high) — prod EXPO_PUBLIC_* env vars do not reach a cloud EAS build.** `.env.production` is gitignored/untracked (`.gitignore:35`) and no `.easignore` exists, so EAS excludes it from the upload. Unless the values are configured as EAS dashboard env vars for the `production` environment, the prod AAB throws at boot (`firebaseClient.ts:31-35`, `api.ts:24-26`). File path: `.gitignore:35`, `.env.production`, `firebaseClient.ts:11-35`, `api.ts:22-26`. "I did not fix this because it is out of scope (read-only audit, and the fix may be a dashboard change, not code)." Recommended resolutions: (a) confirm EAS env vars are set on the dashboard; or (b) add `.easignore` un-ignoring `.env.production`; or (c) move the values into `eas.json` `build.production.env` / EAS secrets.

- **FLAG E (high/medium) — Android camera permission missing; Android 13+ media permission incomplete.** `android.permission.CAMERA` absent from manifest; expo-image-picker does not add it. The app's camera flow is gated on `requestCameraPermissionsAsync()` which auto-denies on Android when the permission is undeclared → take-photo broken on Android. `READ_MEDIA_IMAGES` also absent (gallery gate risk on Android 13+). File paths: `android/app/src/main/AndroidManifest.xml:2-7`, `src/lib/permissions/imagePermissions.ts:3-17`, `src/components/ImagesImport.tsx:100-105,111-118`, `app.config.ts:132-138`. "I did not fix this because it is out of scope (read-only audit; also touches app.config.ts which is brief-gated)." Recommended resolution: add `android.permissions: ['android.permission.CAMERA', 'android.permission.READ_MEDIA_IMAGES']` to the android block in `app.config.ts`, re-prebuild, verify on a physical Android device (esp. 13+).

- **Adjacent (low) — misleading comment in `.env.production:13`:** comment reads `# Firebase (stage project: oglasino-prod-7e5db)` but the project is the PROD project; values are correct, only the "stage" label is wrong. Did not fix (out of scope, no-code-impact).

- **Drafted issues.md entries (for Docs/QA to apply; I do not write config files):**
  1. `[ ]` (high) Mobile prod build: `EXPO_PUBLIC_*` values live only in gitignored/untracked `.env.production`; no `.easignore`. Cloud EAS `production` build will not receive them unless set as EAS dashboard env vars → app throws at boot (`firebaseClient.ts:31-35`, `api.ts:24-26`). Confirm dashboard env vars or add `.easignore`/`eas.json env`. Surfaced by oglasino-expo android-prod-build audit 2026-06-12.
  2. `[ ]` (high) Mobile Android: `android.permission.CAMERA` not declared (expo-image-picker doesn't add it); camera-capture flow gated on `requestCameraPermissionsAsync()` is blocked on Android. `READ_MEDIA_IMAGES` also absent (Android 13+ gallery-gate risk). Add to `app.config.ts` android.permissions and re-verify on device. Surfaced by oglasino-expo android-prod-build audit 2026-06-12.

- **Config-file dependency check (closure gate):** no config-file edit is required *by me* to close this audit. The two issues.md entries above are recommendations for Mastermind/Docs/QA, not a blocking pending draft from this session's own code changes (there were none).
