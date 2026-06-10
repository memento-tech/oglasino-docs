# Audit — Native permissions inventory (oglasino-expo)

**Type:** Phase 2 audit, READ-ONLY. No code changed, nothing installed, no prebuild/eas run, nothing committed.
**Date:** 2026-06-09
**Branch note:** Brief specifies branch `new-expo-dev`; the actual checked-out branch is `dev` (`git branch --show-current` → `dev`). Per the hard rule I stayed on the checked-out branch and did not switch. Flagging the discrepancy for Igor — if this audit was meant to run against `new-expo-dev`, the findings should be re-confirmed there.

Every file:line and present/absent claim below is backed by BOTH a `view` (Read/cat) and an independent `rg`. Where a string is claimed absent, the `rg` command and its empty result are shown. `view` and `rg` agreed on every citation in this report.

---

## ⚠️ BUILD BLOCKERS (HIGH crash risk) — surfaced at top

An iOS app that calls a permission-requesting API with no matching usage-description string in Info.plist crashes hard (SIGABRT/TCC). Two such gaps exist:

| # | iOS API called | File:line | Missing Info.plist key | Severity |
|---|----------------|-----------|------------------------|----------|
| 1 | `ImagePicker.requestCameraPermissionsAsync()` (+ `launchCameraAsync`) | `src/lib/permissions/imagePermissions.ts:7`, `src/components/ImagesImport.tsx:100` | `NSCameraUsageDescription` | **HIGH** |
| 2 | `ImagePicker.requestMediaLibraryPermissionsAsync()` | `src/lib/permissions/imagePermissions.ts:15` | `NSPhotoLibraryUsageDescription` | **HIGH** |

Both are reached by **user action** (image-picker bottom sheet / avatar upload), not at launch — so they crash the first time a user taps "take photo" or "choose from gallery", not on boot. Still build-blockers: the camera/gallery flow is unusable and SIGABRTs on iOS until the strings exist.

**Root cause:** `expo-image-picker` is installed (`~17.0.11`) and used, but (a) its config plugin is **not** in `app.config.ts`'s `plugins` array, and (b) `ios.infoPlist` contains **no** `NS*UsageDescription` keys at all. So nothing injects the camera/photo-library strings at prebuild, and none are hand-written. See Step 3.

Fix is a separate brief (out of scope here): either add the `expo-image-picker` config plugin with `cameraPermission` / `photosPermission` props, or hand-write `NSCameraUsageDescription` + `NSPhotoLibraryUsageDescription` into `ios.infoPlist`.

---

## STEP 1 — Installed permission-gating modules

From `package.json` (Read, lines 16–84; confirmed by `rg`). Present:

| Module | Version | Can prompt for |
|--------|---------|----------------|
| `expo-image-picker` | `~17.0.11` | iOS Camera, iOS Photo Library; Android CAMERA / media |
| `expo-notifications` | `~0.32.16` | iOS push auth; Android 13+ POST_NOTIFICATIONS |
| `expo-device` | `~8.0.10` | none (no runtime permission) |
| `expo-file-system` | `~19.0.21` | none on iOS; no runtime prompt |
| `@react-native-community/netinfo` | `11.4.1` | none on iOS; Android ACCESS_NETWORK_STATE (install-time, no prompt) |
| `expo-image-manipulator` | `~14.0.8` | none (operates on already-picked assets) |

**Absent** (checked because the brief named them) — `rg` over `package.json` returned nothing:

```
$ rg -n 'expo-location|expo-camera|expo-media-library|expo-contacts|expo-calendar|expo-av|expo-audio|expo-local-authentication' package.json
→ (no output)
```

So: no `expo-location`, `expo-camera`, `expo-media-library`, `expo-contacts`, `expo-calendar`, `expo-av`/`expo-audio`, `expo-local-authentication`. And no `expo-tracking-transparency` (see Step 6).

---

## STEP 2 — Runtime permission REQUEST inventory (the crash surface)

```
$ rg -n 'requestPermissionsAsync|getPermissionsAsync|requestCameraPermissionsAsync|requestMediaLibraryPermissionsAsync|requestForegroundPermissionsAsync|requestBackgroundPermissionsAsync|requestTrackingPermissionsAsync|getTrackingPermissionsAsync|launchCameraAsync|launchImageLibraryAsync|MediaLibrary\.|usePermissions' src app
```
plus a follow-up `rg` for `getCameraPermissionsAsync|getMediaLibraryPermissionsAsync`. Combined results (every line below confirmed by both `view` of the file and `rg`):

| File:line | Call | Module | iOS implication | Android implication | When |
|-----------|------|--------|-----------------|---------------------|------|
| `src/lib/permissions/imagePermissions.ts:4` | `getCameraPermissionsAsync()` | expo-image-picker | reads status — no string required, no crash | CAMERA | user-action (precedes request) |
| `src/lib/permissions/imagePermissions.ts:7` | `requestCameraPermissionsAsync()` | expo-image-picker | **NSCameraUsageDescription required** | CAMERA | user-action |
| `src/lib/permissions/imagePermissions.ts:12` | `getMediaLibraryPermissionsAsync()` | expo-image-picker | reads status — no string required, no crash | media read | user-action (precedes request) |
| `src/lib/permissions/imagePermissions.ts:15` | `requestMediaLibraryPermissionsAsync()` | expo-image-picker | **NSPhotoLibraryUsageDescription required** | media read | user-action |
| `src/components/ImagesImport.tsx:100` | `launchCameraAsync({quality:0.8})` | expo-image-picker | **NSCameraUsageDescription required** | CAMERA | user-action (`pickFromCamera`, behind sheet) |
| `src/components/ImagesImport.tsx:110` | `launchImageLibraryAsync(...)` | expo-image-picker | PHPicker — no string required (iOS 14+) | media read | user-action (`pickFromGallery`) |
| `src/components/messages/MessageInput.tsx:62` | `launchImageLibraryAsync(...)` | expo-image-picker | PHPicker — no string required | media read | user-action (`pickImages`) |
| `src/components/dashboard/components/AvatarUpload.tsx:46` | `launchImageLibraryAsync(...)` | expo-image-picker | PHPicker — no string required | media read | user-action (`openImagePicker`) |
| `src/notifications/lib/pushNotificationRegister.ts:18` | `Notifications.getPermissionsAsync()` | expo-notifications | reads status — no string required | — | login-time (`registerForPush`) |
| `src/notifications/lib/pushNotificationRegister.ts:23` | `Notifications.requestPermissionsAsync()` | expo-notifications | **no iOS usage string required** for push | POST_NOTIFICATIONS (13+) | user-action (after soft-prompt accept) |
| `src/notifications/lib/pushNotificationRegister.ts:78` | `Notifications.getPermissionsAsync()` | expo-notifications | reads status — no string required | — | AppState `change` listener |
| `src/notifications/components/PushNotificationsInit.tsx:175` | `Notifications.getPermissionsAsync()` | expo-notifications | reads status — no string required | — | near-launch (runs in effect when `user` set) |

**Camera/photo-library call graph (launch vs user-action):**
- `ImagesImport.tsx` `pickFromCamera` (`:96`) → `ensureCameraPermission()` (`:98`) → `requestCameraPermissionsAsync`. Result IS checked (`if (!(await ensureCameraPermission())) return;`).
- `ImagesImport.tsx` `pickFromGallery` (`:106`) and `AvatarUpload.tsx` `openImagePicker` (`:43`) → `ensureGalleryPermission()` → `requestMediaLibraryPermissionsAsync`. Result IS checked.
- `MessageInput.tsx` `pickImages` (`:59`) calls `launchImageLibraryAsync` **without** any `ensure*Permission` gate — relies on PHPicker needing no permission. No crash, but see Step 7 (inconsistency).
- `registerForPush` is called from `PushNotificationsInit.tsx:184` / `:196`, mounted in `src/components/init/AppInit.tsx:28`. The notification request runs only after the user logs in and accepts the in-app soft-prompt dialog (`DialogId.SOFT_PUSH_PERMISSION_DIALOG`); `getPermissionsAsync` (status read only) runs near launch. Neither needs an iOS usage string.

No `MediaLibrary.` (expo-media-library) usage, no `usePermissions`, no foreground/background location, no ATT calls (Step 6).

---

## STEP 3 — Declared iOS usage strings

`app.config.ts` `ios.infoPlist` block, verbatim (Read lines 73–75):

```ts
infoPlist: {
  UIBackgroundModes: ['remote-notification'],
},
```

There is **no** `app.json` (`ls app.json` → `No such file or directory`). The only Info.plist content is `UIBackgroundModes` — zero `NS*UsageDescription` keys.

Absence proof — `rg` for any usage-description string anywhere in the repo (excluding node_modules) returns nothing:

```
$ rg -n 'UsageDescription' --glob '!node_modules' .
→ (no output)
```

Present/absent checklist (all ABSENT):

| Key | Present? | Source |
|-----|----------|--------|
| `NSCameraUsageDescription` | **No** | MISSING (required — see HIGH #1) |
| `NSPhotoLibraryUsageDescription` | **No** | MISSING (required — see HIGH #2) |
| `NSPhotoLibraryAddUsageDescription` | **No** | not required (app does not save to library) |
| `NSMicrophoneUsageDescription` | **No** | not required (no audio/video capture) |
| `NSUserTrackingUsageDescription` | **No** | not required (no ATT — Step 6) |
| `NSLocationWhenInUseUsageDescription` (+ other `NSLocation*`) | **No** | not required (no location module) |

**Plugin-injection check:** the `plugins` array (Read lines 110–150) is: `expo-router`, `expo-splash-screen`, `@react-native-google-signin/google-signin`, `expo-secure-store`, `expo-web-browser`, `expo-dev-client`, `expo-notifications` (config block lines 131–139), `@react-native-firebase/app`, `expo-build-properties`. There is **no** `expo-image-picker` or `expo-camera` plugin entry:

```
$ rg -n 'image-picker|expo-camera|CameraUsage|PhotoLibrary' app.config.ts
→ (no output)
```

So **no** config plugin injects `NSCameraUsageDescription` or `NSPhotoLibraryUsageDescription` at prebuild, and none are hand-written. Both required strings have **no owner** — that is the HIGH gap. The `expo-notifications` plugin block contains no iOS usage string (push needs none).

---

## STEP 4 — Declared Android permissions

`app.config.ts` `android` block, verbatim (Read lines 85–94):

```ts
android: {
  package: bundleId,
  intentFilters: androidIntentFilters,
  adaptiveIcon: {
    foregroundImage: './assets/images/logo/android-foreground.png',
    backgroundColor: '#E6F4FE',
    monochromeImage: './assets/images/logo/android-monochrome.png',
  },
  googleServicesFile: androidGoogleServicesFile,
},
```

There is **no** `android.permissions` and **no** `android.blockedPermissions`:

```
$ rg -n 'permissions|blockedPermissions' app.config.ts
→ (no output)
```

All Android permissions therefore come from native module / config-plugin manifest merge at prebuild (not visible in `app.config.ts`, not verifiable without prebuild — which is out of scope):
- **`POST_NOTIFICATIONS`** (Android 13+ runtime notification permission) — auto-added by the `expo-notifications` config plugin. Required because `Notifications.requestPermissionsAsync` runs (Step 2). Expected present via plugin; confirm at prebuild.
- **`CAMERA`** and **media read** (`READ_MEDIA_IMAGES` / legacy `READ_EXTERNAL_STORAGE`) — merged by the `expo-image-picker` native module manifest. Not declared in `app.config.ts`.
- **`ACCESS_NETWORK_STATE`** — merged by `@react-native-community/netinfo`.

On Android a missing manifest permission **denies** at runtime (rarely crashes), so these are LOW-risk relative to iOS. Worth a one-line prebuild verification that `POST_NOTIFICATIONS` actually lands.

---

## STEP 5 — Cross-reference table (the deliverable)

One row per permission the app can request at runtime. HIGH rows are the two build blockers surfaced at the top.

| Permission / API | Where requested (file:line, when) | iOS string required? / present? / source | Android perm required? / declared? | CRASH RISK |
|------------------|-----------------------------------|-------------------------------------------|------------------------------------|------------|
| **Camera** (`requestCameraPermissionsAsync`, `launchCameraAsync`) | `imagePermissions.ts:7`, `ImagesImport.tsx:100` — user-action | `NSCameraUsageDescription` — **N** — **MISSING** | CAMERA — module-merged (not in config), verify at prebuild | **HIGH** |
| **Photo library** (`requestMediaLibraryPermissionsAsync`) | `imagePermissions.ts:15` (via AvatarUpload `:44`, ImagesImport gallery `:108`) — user-action | `NSPhotoLibraryUsageDescription` — **N** — **MISSING** | media read — module-merged, verify at prebuild | **HIGH** |
| **Photo library (PHPicker)** (`launchImageLibraryAsync`) | `MessageInput.tsx:62`, `AvatarUpload.tsx:46`, `ImagesImport.tsx:110` — user-action | none (PHPicker, iOS 14+) — N/A — n/a | media read — module-merged | LOW |
| **Push notifications** (`requestPermissionsAsync`) | `pushNotificationRegister.ts:23` — user-action (after soft-prompt); status read near-launch at `PushNotificationsInit.tsx:175` | none required for push — N/A — n/a | `POST_NOTIFICATIONS` (13+) — plugin-added (expo-notifications), verify at prebuild | LOW |

---

## STEP 6 — expo-tracking-transparency disposition

ATT is **fully removed** — all three footprints are clean. Memory ("ATT was removed") is **confirmed** against the actual tree:

1. **Runtime calls:** none.
```
$ rg -n 'tracking-transparency|TrackingPermissions|NSUserTracking|requestTrackingPermissions|getTrackingPermissions' --glob '!node_modules' .
→ (no output)
```
2. **package.json dependency:** absent.
```
$ rg -n 'tracking-transparency' package.json
→ (no output)
```
3. **Config plugin in app.config.ts:** absent (not in the `plugins` list, Read lines 110–150; the `rg` in #1 covers app.config.ts too).
4. **`NSUserTrackingUsageDescription`:** absent (Step 3 `UsageDescription` grep returned nothing).

No cleanup item remains for ATT — there is nothing left to remove. (This differs from the "unused dep, remove at next prebuild" framing in the brief: the dep is already gone, so that follow-up is already closed.)

---

## STEP 7 — Adjacent observations (Part 4b) — not fixed, flagged only

- **`MessageInput.tsx:62`** — `pickImages` calls `launchImageLibraryAsync` with **no** `ensureGalleryPermission` gate, unlike `AvatarUpload.tsx:44` and `ImagesImport.tsx:108` which both gate first. Works today (PHPicker needs no permission) but is an inconsistent pattern; if the picker config ever changes to a permission-requiring mode, this site has no guard. **low**.
- **`PushNotificationsInit.tsx:184` / `:196`** — `registerForPush(user.id)` is called without `await` at `:184` (fire-and-forget) but awaited at `:196`. Inside, `requestPermissionsAsync` failures are handled (early-returns on non-granted), but the un-awaited call at `:184` means any rejection is an unhandled promise. **low**.
- **`pushNotificationRegister.ts:39`, `:48`, `:67`, `:81`** — `attachPushTokenToBackend` / `detachPushTokenFromBackend` are awaited with no try/catch; a backend/network failure during token attach rejects out of `registerForPush` silently (the un-awaited `:184` path swallows it). Deny/failure of the network step has no user-facing handling. **low–med**.
- **Camera/gallery deny path** — `ensureCameraPermission` / `ensureGalleryPermission` return `false` and callers just `return` (`ImagesImport.tsx:98/108`, `AvatarUpload.tsx:44`). A user who denies gets a silent no-op with no toast/explanation directing them to Settings. Functional, but a silent dead-end UX. **low**.
- **No dead iOS usage strings** — every declared Info.plist entry (`UIBackgroundModes` only) is in use (remote push). No NS string is declared for an unused permission (there are none declared at all). Nothing to remove.

---

## Tool-reliability note

Per the brief's mandatory rule, every file:line and every present/absent claim above was confirmed by BOTH `view` (Read / cat -n) and an independent `rg`. The two tools agreed on every citation. All four absence claims (no UsageDescription strings, no android.permissions, no image-picker plugin, no ATT footprint) are backed by the explicit `rg` commands shown inline, each returning empty.
