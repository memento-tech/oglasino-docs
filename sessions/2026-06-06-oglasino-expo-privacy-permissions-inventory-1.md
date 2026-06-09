# Session summary

**Repo:** oglasino-expo
**Branch:** dev (brief said `new-expo-dev`; the repo is currently on `dev`. Read-only audit — no checkout performed.)
**Date:** 2026-06-06
**Task:** Enumerate the OS permissions the app requests and any device-level data the app collects that the web version does not (for a Privacy Policy that must name mobile-specific data surfaces).

## Implemented

Read-only audit. No code changes. The inventory below is the core finding, derived from the code and config — not from assumptions.

**Important scope note on source-of-truth:** `/ios` and `/android` are **gitignored, prebuild-generated** folders (`.gitignore` "generated native folders"; `git ls-files` shows 0 tracked files in each). The authoritative source is `app.config.ts` + the Expo config plugins it lists. The native manifests below are the *derived output* a clean `expo prebuild` reproduces. I cite both: app.config.ts (the intent) and the generated manifest (the realized permission). The privacy-relevant conclusions (camera/photos/push declared; **no location**; **no ATT**) are driven by which plugins/deps are present, so they hold regardless of regeneration.

---

## A. OS permissions — declared

### iOS (Info.plist usage-description strings)

| Permission key | Surface | Declared by | Actually used in app code? |
| --- | --- | --- | --- |
| `NSCameraUsageDescription` | Camera | `expo-image-picker` plugin default | **Yes** — `launchCameraAsync` in `ImagesImport.tsx:100` (product photos). |
| `NSPhotoLibraryUsageDescription` | Photo library | `expo-image-picker` plugin default | **Yes** — `launchImageLibraryAsync` in `ImagesImport.tsx:110`, `MessageInput.tsx:62` (chat image), `AvatarUpload.tsx:46` (avatar). |
| `NSMicrophoneUsageDescription` | Microphone | `expo-image-picker` plugin default (video capture) | **No** — no audio/video recording anywhere. Declared-but-unused. |
| `NSFaceIDUsageDescription` | Face ID / biometrics | `expo-secure-store` plugin (app.config.ts:128) | **No** — `expo-secure-store` is never imported in `src/`/`app/`. Declared-but-unused. (Matches the storage-inventory audit's finding F.) |
| `UIBackgroundModes: remote-notification` | Background push wakeups | app.config.ts:73-75 (`infoPlist`) + `expo-notifications` | **Yes** — push handling. |

`ITSAppUsesNonExemptEncryption=false` (app.config.ts:77) — declares no non-exempt crypto; not a permission.

### Android (AndroidManifest.xml — generated)

| Permission | Surface | Declared by | Used? |
| --- | --- | --- | --- |
| `INTERNET` | Networking | baseline / RN | Yes. |
| `READ_EXTERNAL_STORAGE` | Read images/files | `expo-image-picker` | Yes (gallery pick). |
| `WRITE_EXTERNAL_STORAGE` | Write images/files | `expo-image-picker` | Yes (picker scratch). |
| `RECORD_AUDIO` | Microphone | `expo-image-picker` default (video) | **No** — declared-but-unused (mirrors iOS mic). |
| `VIBRATE` | Notification vibration | `expo-notifications` / Firebase messaging | Yes. |
| `SYSTEM_ALERT_WINDOW` | Draw-over-other-apps | React Native dev menu overlay (present in the **dev** build's `main` + `debug` manifests) | Dev tooling only; not an app feature. Release builds strip it. |

- **No `CAMERA` permission on Android** — `expo-image-picker` launches the system camera via intent, which doesn't require the manifest permission. Camera still works.
- **No `ACCESS_FINE_LOCATION` / `ACCESS_COARSE_LOCATION`.**
- `<queries>` for `VIEW`/`BROWSABLE` `https` — link-handling / App Links autoverify, not a runtime permission.

## B. OS permissions — requested at runtime

| Permission | Request call | Where | Trigger |
| --- | --- | --- | --- |
| Camera | `ImagePicker.requestCameraPermissionsAsync()` | `src/lib/permissions/imagePermissions.ts:7` (`ensureCameraPermission`) | User taps "take photo" in image import. |
| Media library | `ImagePicker.requestMediaLibraryPermissionsAsync()` | `imagePermissions.ts:15` (`ensureGalleryPermission`) | User picks from gallery (product images, chat image, avatar). |
| Push notifications | `Notifications.requestPermissionsAsync()` | `src/notifications/lib/pushNotificationRegister.ts:23` | Inside `registerForPush(userId)` — fires only on a **physical device** (`Device.isDevice` guard, line 14) and only **after the user is logged in**. Re-prompt is throttled via the `SPPL` timestamp key (per storage-inventory audit). |

## C. Device-derived location (GPS)

**None.** No `expo-location` dependency, no `getCurrentPositionAsync` / `watchPosition` / geolocation / `coords` / lat-long anywhere in `src/`/`app/`. The app has **no device-GPS surface at all.**

The only "location" concept is the **account-level region/city/base-site the user explicitly picks** — a server-side selection identical to web (stored in `auth-storage` / `base_site`; the user chooses it from a list). It is **not** derived from device GPS. Confirmed: account-level picker, not device location.

## D. Device identifiers read or transmitted

| Identifier | Read? | Transmitted? Where to? |
| --- | --- | --- |
| Hardware/device IDs (`Device.modelId`, `osBuildId`, IDFV, Android ID) | **No.** `expo-device` is imported in exactly two files and used **only** as `Device.isDevice` (boolean: physical vs simulator — `pushNotificationRegister.ts:14`, `PushNotificationsInit.tsx:170`). No identifier property is read. | — |
| **Expo push token** (`getExpoPushTokenAsync`) | Yes — `pushNotificationRegister.ts:30`. | **Yes → Oglasino's own backend**, `POST /secure/push/token` `{ token, platform: 'EXPO' }` (`pushTokenService.ts:4`). Tied to the authenticated user; **detached** on logout / permission-revoke (`detachPushTokenFromBackend`). This is a device-addressable routing identifier and is **mobile-only** (web has no equivalent). Not stored locally (per storage-inventory audit). |
| **Firebase Analytics app-instance ID** | Generated by `@react-native-firebase/analytics` (pseudonymous, device-scoped). | Sent to Firebase/Google as part of analytics. **Consent-gated** — collection only when `consent_decision.analytics` is granted; `setAnalyticsCollectionEnabled(false)` when denied (`analyticsClient.ts:38-69`). `user_id` is set to the backend userId, mirroring web's GA4UserIdSync. This is the mobile analog of web GA4's `client_id`. |
| **IDFA / advertising ID** | **Not read** by app code; no `AdSupport` usage; `NSPrivacyTracking=false`. **But see caveat below** — the iOS build links the ad-id-capable Firebase variant. | Not collected in practice (no ATT → IDFA is all-zeros). |

**IDFA caveat (privacy-questionnaire relevant, not a code bug):** the iOS Pods resolve `FirebaseAnalytics/IdentitySupport` + `GoogleAppMeasurement/IdentitySupport` (`ios/Podfile.lock:359-388`), the IDFA-*capable* GoogleAppMeasurement variant. This is the default because the `$RNFirebaseAnalyticsWithoutAdIdSupport` Podfile flag is **not** set. Because ATT is never requested, the IDFA returns zeros and **no advertising identifier is actually collected**. The linked framework is still a consideration for the App Store privacy questionnaire / Apple privacy manifest, so I'm flagging it rather than silently treating "no ATT" as "no ad-id framework." See "For Mastermind."

## E. App Tracking Transparency (ATT)

**Not requested. Confirmed removed.** Evidence, four independent signals:
1. No `expo-tracking-transparency` (or any ATT) dependency in `package.json`.
2. No `TrackingTransparency` / `requestTrackingPermissionsAsync` import anywhere in `src/`/`app/`.
3. Explicit code comments documenting the deliberate absence: `analyticsClient.ts:17` ("First-party analytics only — no IDFA/ads, so no App Tracking Transparency gate") and `AnalyticsInit.tsx:13` ("First-party analytics only (no IDFA/ads), so no App Tracking Transparency").
4. The app-target Apple privacy manifest `ios/OglasinoDev/PrivacyInfo.xcprivacy` sets `NSPrivacyTracking = false` and `NSPrivacyCollectedDataTypes = <array/>` (empty).

Prior work's removal of the ATT call is confirmed against current code: ATT is absent.

## F. Apple privacy manifest (`PrivacyInfo.xcprivacy`)

App-target manifest declares only **required-reason API** usage (no data collection at the app target level):
- `NSPrivacyAccessedAPICategoryFileTimestamp`, `…UserDefaults`, `…SystemBootTime`, `…DiskSpace` — each with standard reason codes. These are SDK housekeeping APIs (AsyncStorage → UserDefaults, file-system → FileTimestamp, etc.), not user-data collection.
- `NSPrivacyTracking = false`, `NSPrivacyCollectedDataTypes` empty. (Pods carry their own manifests, aggregated at build.)

## G. Mobile-only surfaces vs web (the brief's core question)

Surfaces the **mobile** app has that **web does not**:
- **Native camera permission + capture** (product photos via system camera).
- **Native photo-library permission + picker** (products, chat images, avatar).
- **OS push-notification permission + Expo push token** sent to backend (device-addressable identifier; web has no equivalent push token).
- **Firebase Analytics app-instance ID** (mobile analog of web GA4 `client_id`; consent-gated).

Surfaces that are the **same** as web (not new on mobile): account-level region/city selection; backend `user_id` on analytics; first-party analytics consent model.

**No** device GPS, **no** contacts, **no** advertising-ID collection, **no** ATT — on either platform.

---

## Files touched

- None (read-only audit).

## Tests

- None run (no code changed).

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change (two low-severity observations flagged below for Mastermind triage, not authored into issues.md)

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): N/A — no code written.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): two observations flagged in "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: Part 10/Phase 2 (read-only audit posture) — confirmed; Part 11 (trust boundaries) — N/A, no trust decision touched.

## Known gaps / TODOs

- I did not exhaustively enumerate every transitive Pod's individual `PrivacyInfo.xcprivacy`; I read the app-target manifest (the one that governs `NSPrivacyTracking` and app-level collected types) and the Firebase/GoogleAppMeasurement Pod resolution relevant to the IDFA question. A full per-Pod manifest sweep is a separate, larger task if the App Store privacy questionnaire needs line-by-line backing.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — no code written.
  - Considered and rejected: nothing — no code written.
  - Simplified or removed: nothing — no code written.

- **Direct answers to the brief's four questions:**
  1. **Permissions:** camera + photo library (image-picker, runtime-requested via `imagePermissions.ts`); push notifications (requested post-login on physical devices); microphone (iOS + Android RECORD_AUDIO) and Face ID (iOS) are **declared by plugin defaults but never exercised** by app code. No location, no contacts. Full table in §A/§B.
  2. **Device location:** none — no GPS/geolocation API present. The region/city is the account-level user-picked value, same as web. §C.
  3. **Device identifiers:** no hardware IDs read (`expo-device` used only for the `isDevice` boolean). The **Expo push token** is transmitted to Oglasino's backend (`/secure/push/token`) and the **Firebase Analytics app-instance ID** goes to Google (consent-gated). No IDFA collected. §D.
  4. **ATT:** confirmed not present — four independent signals (no dep, no import, explicit code comments, `NSPrivacyTracking=false`). §E.

- **Privacy-relevant highlights for the policy author:**
  1. Camera and photos are mobile-specific data surfaces the web policy wouldn't have named — both gated behind a runtime OS prompt and used only for user-initiated uploads.
  2. The push token is a device-addressable identifier sent to and stored by Oglasino's backend; it is detached on logout / permission revocation.
  3. Analytics (app-instance ID + backend user_id) is the only third-party (Google) data flow and is consent-gated; it parallels web GA4.
  4. No device location, no advertising identifier, no cross-app tracking.

- **Part 4b adjacent observations (low severity, out of scope — not fixed):**
  1. **Declared-but-unused permission descriptors.** `NSMicrophoneUsageDescription` + Android `RECORD_AUDIO` (from `expo-image-picker`'s video defaults) and `NSFaceIDUsageDescription` (from the unused `expo-secure-store` plugin) are present in the built app though no code uses microphone/biometrics. These can trip an App Store reviewer's "why does this app need the microphone?" question and slightly over-state the app's data access in the privacy questionnaire. File: `app.config.ts:128` (`expo-secure-store` plugin) and the `expo-image-picker` dependency. Not fixed — out of scope for a read-only audit; also note CLAUDE.md forbids editing `app.config.ts` without an explicit brief instruction. (Tie-in: storage-inventory audit already flagged `expo-secure-store` as installed-but-unused.)
  2. **IDFA-capable Firebase variant linked.** `ios/Podfile.lock:359-388` resolves `FirebaseAnalytics/IdentitySupport` because `$RNFirebaseAnalyticsWithoutAdIdSupport` is not set in the Podfile. Functionally harmless today (no ATT → zeroed IDFA), but if Igor wants the App Store privacy questionnaire to honestly answer "no, we don't link the ad-identifier framework," setting that Podfile flag would drop the IdentitySupport variant. Decision for Mastermind/Igor — I did not change it (native config, out of scope, and the generated `/ios` folder isn't the source of truth).

- **Config-file dependency closure:** none required. No state.md Expo-backlog row is implicated (this is a privacy-policy fact-finding audit, not a feature adoption). Explicitly: no config-file edit is owed by this session.

- Nothing else flagged.
