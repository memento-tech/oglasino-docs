# Expo Cloud Setup

How `oglasino-expo` is configured, what cloud projects it depends on, and how the three build tiers map to backend, Firebase, and EAS infrastructure.

## Scope

The mobile app's cloud setup, EAS project linkage, build profiles, environment configuration, and Firebase app registration. Excludes feature work (auth flows, image pipeline, etc.) and excludes platform-specific runtime concerns (push notifications wiring, deep link handlers).

## Three-tier model

The mobile app builds in three tiers. Each tier has its own bundle identifier, its own Firebase App (within either the stage or prod Firebase project), and its own environment file.

| Tier | EAS profile | iOS bundle ID / Android package | Firebase project | Env file |
|------|-------------|--------------------------------|------------------|----------|
| Development | `development` | `com.oglasino.development` | `oglasino-stage-49abb` | `.env.development` |
| Preview | `preview` | `com.oglasino.preview` | `oglasino-stage-49abb` | `.env.preview` |
| Production | `production` | `com.oglasino` | `oglasino-prod-7e5db` | `.env.production` |

Tiers are not Firebase projects. Two tiers (development, preview) share the stage Firebase project but register as separate Firebase Apps with distinct bundle IDs. Production has its own Firebase project entirely.

The three bundle IDs are independent in Apple Developer and Google Play. A device can have all three tiers installed simultaneously without collision.

## EAS Cloud project

- Account: `oglasino` (org)
- Project: `@oglasino/oglasino-expo`
- Project ID: `382cac59-45fa-420e-ab80-7fc709e6d2f3`
- Stored in `app.config.ts` under `extra.eas.projectId`

The dashboard: https://expo.dev/accounts/oglasino/projects/oglasino-expo

## Build profiles (eas.json)

Three profiles defined in `eas.json` at repo root. Each sets `APP_ENV` as an environment variable, which `app.config.ts` reads to branch all per-tier configuration.

### `development`
- `developmentClient: true` — produces a dev client build with Expo dev menu and Metro JS loader
- `distribution: internal` — Ad Hoc on iOS, raw APK on Android
- `ios.resourceClass: m-medium`
- `ios.simulator: false` — real-device builds only
- `android.buildType: apk`
- Triggered: manually, locally, via `eas build --profile development --platform <ios|android|all>`

### `preview`
- No development client — production-shaped bundle, no dev menu
- `distribution: internal` — same Ad Hoc / APK distribution as development
- `ios.resourceClass: m-medium`
- `android.buildType: apk`
- Triggered: auto on push to `preview` branch via `.eas/workflows/preview.yml`

### `production`
- No development client — production bundle
- `distribution: store` — App Store / Play Store routing
- `ios.resourceClass: m-medium`
- `android.buildType: app-bundle` — AAB for Play Store
- `autoIncrement: true` — EAS bumps remote `versionCode` (Android) / `buildNumber` (iOS) on every build
- Triggered: auto on push to `main` branch via `.eas/workflows/production.yml`

### Shared config
- `cli.version: >= 19.0.0`
- `cli.appVersionSource: remote` — version tracked centrally in EAS Cloud, marketing version lives in `app.config.ts`
- `submit.production: {}` — placeholder for future App Store / Play Store submission config

## EAS Workflows (CI)

Two workflow files in `.eas/workflows/`. Both target both platforms (Android + iOS) with two parallel jobs.

### `preview.yml`
- Trigger: push to `preview` branch
- Path filter: `app/**`, `src/**`, `assets/**`, `app.config.ts`, `package.json`, `package-lock.json`, `eas.json`, `.env.preview`
- Jobs: `build_android` (preview profile) + `build_ios` (preview profile)

### `production.yml`
- Trigger: push to `main` branch
- Path filter: same as preview but watches `.env.production` instead
- Jobs: `build_android` (production profile) + `build_ios` (production profile)

No workflow exists for the `development` profile. Development builds are manual-only.

### Submission
Production builds reach EAS Cloud but do not auto-submit to stores. Submission is a separate manual `eas submit --platform <ios|android> --profile production` step.

## app.config.ts — dynamic configuration

`app.config.ts` is the source of truth for all per-tier configuration. It reads `APP_ENV` and branches on its value (`development | preview | production`).

Per-tier values derived from `APP_ENV`:
- `bundleId` (iOS) and `package` (Android): `com.oglasino.development | com.oglasino.preview | com.oglasino`
- `iosGoogleServicesFile`: `./GoogleService-Info.development.plist | ./GoogleService-Info.preview.plist | ./GoogleService-Info.prod.plist`
- `androidGoogleServicesFile`: same naming pattern with `.json` extension
- `name`: `Oglasino Dev | Oglasino Preview | Oglasino` — home-screen icon label
- `scheme`: `oglasino-dev | oglasino-preview | oglasino` — URL scheme registered for deep links

Project-wide config not branched per tier:
- `extra.eas.projectId`: `382cac59-45fa-420e-ab80-7fc709e6d2f3`
- `extra.env`: passes `APP_ENV` through to runtime (consumable via `Constants.expoConfig.extra.env`)
- `extra.firebaseConfig`: reads six `EXPO_PUBLIC_FIREBASE_*` env vars and exposes them under `Constants.expoConfig.extra.firebaseConfig` for the JS Firebase SDK
- Icon, splash, status bar color: same across tiers (cosmetic per-tier branching not implemented)

## Environment files

Three env files at repo root, gitignored (`.gitignore` uses `.env.*` with `!.env.example` exception).

| File | APP_ENV | API URL | CDN URL | Firebase project | GA4 |
|------|---------|---------|---------|------------------|-----|
| `.env.development` | development | `<local LAN IP>` (dev backend) | `https://cdn-stage.oglasino.com` | `oglasino-stage-49abb` | `G-P0LEVEJ0V9` (debug on) |
| `.env.preview` | preview | `https://api-stage.oglasino.com/api` | `https://cdn-stage.oglasino.com` | `oglasino-stage-49abb` | `G-P0LEVEJ0V9` (debug on) |
| `.env.production` | production | `https://oglasino.com/api` | `https://cdn.oglasino.com` | `oglasino-prod-7e5db` | `G-GNKB4WBNC0` (debug off) |

Each file holds 13 keys: `APP_ENV`, `EXPO_PUBLIC_API_URL`, `EXPO_PUBLIC_CDN_URL`, `EXPO_PUBLIC_WATERMARK_ENABLED`, six `EXPO_PUBLIC_FIREBASE_*`, `EXPO_PUBLIC_WEB_CLIENT_ID`, `EXPO_PUBLIC_GA4_MEASUREMENT_ID`, `EXPO_PUBLIC_GA4_DEBUG_MODE`.

Template at `.env.example` (tracked in git) carries the same key list with placeholder/empty values.

### How EAS picks the right env file

EAS Cloud Build reads `eas.json`'s `env.APP_ENV` for the building profile, then sets the equivalent process env var on the build worker. Expo's auto-loading reads the matching `.env.<APP_ENV>` file. The values land in `process.env.EXPO_PUBLIC_*` at build time and are baked into the JS bundle.

Local `npx expo config` and `npx expo start` always load `.env.development` regardless of `APP_ENV` — a quirk of Expo CLI's local env file resolution. EAS Cloud Build does not have this quirk and correctly loads the per-profile env file.

## Firebase setup

Two Firebase projects, each with multiple Firebase Apps registered against tier-specific bundle IDs.

### `oglasino-stage-49abb` (stage project)

Apps registered:
- Android: `com.oglasino.development` (nickname: Oglasino Development)
- Android: `com.oglasino.preview` (nickname: Oglasino Preview)
- iOS: `com.oglasino.development` (nickname: Oglasino Development iOS)
- iOS: `com.oglasino.preview` (nickname: Oglasino Preview iOS)
- Web: `oglasino-web-stage` (used by `oglasino-web` repo, unrelated to mobile)

Each Android app has SHA-1 and SHA-256 fingerprints registered from the EAS-managed keystore. These are required for Google Sign-In on Android. The fingerprints are embedded in the corresponding `google-services.<tier>.json` file.

### `oglasino-prod-7e5db` (prod project)

Apps registered:
- Android: `com.oglasino`
- iOS: `com.oglasino`

Used only by the production tier of the mobile app and the production deployment of `oglasino-web`.

### Google service files

Six files at repo root, tracked in git (not gitignored — Firebase config files are non-secret per Firebase's documentation; they ship inside the compiled app anyway):

- `GoogleService-Info.development.plist` (stage Firebase, dev tier)
- `GoogleService-Info.preview.plist` (stage Firebase, preview tier)
- `GoogleService-Info.prod.plist` (prod Firebase, production tier)
- `google-services.development.json` (stage Firebase, dev tier)
- `google-services.preview.json` (stage Firebase, preview tier)
- `google-services.prod.json` (prod Firebase, production tier)

`app.config.ts` references these by filename per-tier.

### OAuth Web Client IDs

Each Firebase project has one auto-generated OAuth Web Client ID in Google Cloud Console (under `APIs & Services → Credentials`). The Web Client ID is project-level, not per-Firebase-App.

- Stage Web Client ID: shared by `.env.development` and `.env.preview`
- Prod Web Client ID: used by `.env.production`

Consumed by `@react-native-google-signin/google-signin` for Google Sign-In, accessed at runtime via `Constants.expoConfig.extra.firebaseConfig.webClientId`.

## Apple Developer setup

Three bundle IDs registered as App IDs in Apple Developer portal:
- `com.oglasino`
- `com.oglasino.development`
- `com.oglasino.preview`

Each has Push Notifications capability enabled (pre-enabled for future APNs work; no APNs keys uploaded yet).

EAS Cloud manages the Apple Distribution Certificate and per-bundle-ID provisioning profiles automatically on first build. Internal-distribution profiles include registered iOS device UDIDs (Ad Hoc distribution); add new devices via `eas device:create` before they can install dev / preview builds.

## Android keystore

EAS Cloud manages a single Android keystore per profile. Pre-launch state: one keystore exists for the `development` profile, generated during the first dev build.

Keystore is backed up at `~/code/oglasino-private/01-secrets/oglasino-infra/oglasino-expo-android-keystore.jks` along with the password. EAS Cloud holds the canonical copy; the local backup is recovery insurance.

SHA-1 and SHA-256 fingerprints from this keystore are registered against `com.oglasino.development` in the stage Firebase project. When preview and production builds run for the first time, EAS generates separate keystores per profile and the fingerprints from each will need registration against their respective Firebase Apps before Google Sign-In works on those tiers.

## Code-level architecture

### Firebase initialization

`src/lib/client/firebaseClient.ts` reads six `EXPO_PUBLIC_FIREBASE_*` env vars at module-evaluation time, validates none are missing (throws clearly naming any absent vars), and calls `initializeApp()` from the JS Firebase SDK (`firebase/app`).

The JS Firebase SDK is platform-agnostic — same App ID is used for both iOS and Android (the Android App ID from `google-services.<tier>.json`). The native-platform-specific App IDs in `GoogleService-Info.plist` and `google-services.json` are consumed by native Firebase modules (e.g., native push registration, native analytics if added later), not by the JS SDK.

### Bundle ID branching

`app.config.ts` reads `process.env.APP_ENV` at config-evaluation time and branches every per-tier field via ternary. Five ternaries currently: bundle ID/package, iOS Google services file, Android Google services file, name, scheme.

### Per-platform handling

`NavigationBar` (from `expo-navigation-bar`) is Android-only. All call sites in `app/_layout.tsx` are guarded with `Platform.OS === 'android'` checks.

`Device.isDevice` (from `expo-device`) distinguishes physical-device from simulator/emulator builds, not tiers.

## Build, install, run flow

### Development tier (manual)

1. `eas build --profile development --platform <ios|android|all>` from local terminal
2. EAS Cloud builds; produces `.ipa` (iOS) or `.apk` (Android)
3. Install on physical device:
   - iOS: scan QR code from EAS dashboard with iPhone Camera → opens in Safari → installs configuration profile → app appears on home screen
   - Android: scan QR code or download APK link → install (allow unknown sources if prompted)
4. iOS-only first-time steps: enable Developer Mode (Settings → Privacy & Security → Developer Mode) on iOS 16+, then launch the app to land on dev client screen
5. Start Metro: `npx expo start --dev-client` from laptop terminal
6. Connect dev client to Metro: scan QR code from Metro output with iPhone (or enter `exp://<laptop-ip>:8081` manually) → app downloads JS bundle from Metro → launches

Iterating: edit JS/TS code → Metro auto-bundles → dev client receives update → app reloads with new code. Native config changes (e.g., `app.config.ts` name/scheme/bundleId) require a fresh build.

### Preview tier (auto on push to `preview`)

1. Commit changes to `preview` branch
2. Push to remote
3. EAS Workflows triggers preview.yml → builds Android + iOS in parallel
4. Build artifacts appear in EAS dashboard
5. Distribute internal install links to testers (UDID-registered iOS devices required)

### Production tier (auto on push to `main`, manual submission)

1. Commit changes to `main` branch
2. Push to remote
3. EAS Workflows triggers production.yml → builds Android AAB + iOS IPA in parallel
4. Build artifacts appear in EAS dashboard
5. Manually run `eas submit --platform <ios|android> --profile production` to submit to App Store / Play Store

## Deferred / future work

Not implemented at the time of writing. Will require separate work to address:

- APNs key generation in Apple Developer portal and upload to Firebase Cloud Messaging (for iOS push notifications)
- FCM service account credentials upload to EAS Secrets (for server-side push send)
- Per-tier app icon variants (visual distinction between dev / preview / production on home screen)
- Deep link URL types (`infoPlist.CFBundleURLTypes` on iOS, `intentFilters` on Android) — will need per-tier scheme registration when implemented
- Native `react-native-firebase` migration if/when native-platform-specific Firebase analytics attribution becomes important
- SHA fingerprint registration for `com.oglasino.preview` and `com.oglasino` Android apps before Google Sign-In works on those tiers
- EAS Secrets push for any secret values that workflows need at build time (currently env files contain only public-prefixed values, no secret-class material)

## Known constraints

- iOS Ad Hoc distribution limited to 100 registered devices per device type per year per Apple Developer account. Track UDID-registered devices via `eas device:list`.
- EAS Free tier: 30 builds/month, 1 concurrent build. Builds may queue. Concurrent builds require paid tier.
- Local `npx expo config` always loads `.env.development` regardless of `APP_ENV` — does not affect EAS Cloud builds. Use `set -a; source .env.<tier>; set +a` to force-load for local verification of per-tier resolution.
- `eas build:inspect` against the `production` profile with `autoIncrement: true` increments the remote versionCode even though no actual build runs. Use `development` or `preview` profile for inspect-only verification.
- The Android keystore is per-profile, generated lazily on first build of each profile. First preview/production build will require fresh fingerprint registration in Firebase Console before Google Sign-In works for that tier.

## Trust boundaries

No trust boundary surface in cloud setup itself. Environment files contain `EXPO_PUBLIC_*` variables that are baked into the JS bundle and therefore not secret. Bundle IDs, project IDs, and OAuth Client IDs are public identifiers. The Android keystore is the only sensitive artifact; it lives in EAS Cloud (managed) and as a local backup in `oglasino-private`.

Runtime trust boundary enforcement (Firebase Auth tokens, backend authorization, OAuth state) is unchanged from existing platform behavior — covered by other specs.
