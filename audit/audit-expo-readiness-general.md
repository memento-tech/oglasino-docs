# Audit — Expo release readiness: general project health

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-05-23
**Mode:** read-only audit. No code changes.

---

## Section 1 — Expo workflow

This project uses the **EAS Build managed workflow with prebuild**. Both `ios/` and `android/` directories exist on disk but are gitignored (`/ios` and `/android` in `.gitignore`), indicating they are generated via `expo prebuild` and not maintained by hand. `eas.json` is present with `development` and `production` build profiles. `app.config.ts` (dynamic config) drives the native configuration — there is no static `app.json`. The `commands.json` at root contains shorthand build commands (`dev-build`, `prod-build`, `dev-build-local`, `prod-build-local`) referencing `eas build` and `expo prebuild --clean`. The project entry point is `expo-router/entry` (per `package.json` `"main"`), confirming file-based routing via `expo-router`.

---

## Section 2 — SDK and core versions

| Package         | Installed version | Notes |
|-----------------|-------------------|-------|
| Expo SDK        | ~54.0.33          | Expo SDK 54. |
| React Native    | 0.81.5            | |
| React           | 19.1.0            | |
| React DOM       | 19.1.0            | Peer dependency for Expo web support. |
| TypeScript      | ~5.9.2            | |
| Node            | Not specified      | No `engines` field in `package.json`, no `.nvmrc` or `.node-version` file at the project root. |

---

## Section 3 — Dependency surface

**Counts:** 47 `dependencies`, 11 `devDependencies` — 58 total direct packages.

**Expressly deprecated packages:** None identified from package names alone. (No `npm ls` output available — read-only audit, no installs.)

**Version mismatches with Expo SDK 54:** No obvious mismatches detected from the declared versions. `react-native-reanimated` (~4.1.1), `react-native-screens` (~4.16.0), `react-native-gesture-handler` (~2.28.0), and `react-native-safe-area-context` (~5.6.0) all use the tilde ranges typical of Expo SDK 54's compatibility matrix.

**Packages installed but appearing unused (no imports found in `src/` or `app/`):**

| Package | Likely explanation |
|---------|--------------------|
| `sonner-native` | Zero imports. Duplicate toast library — `react-native-toast-notifications` is the active one (5+ import sites). |
| `expo-haptics` | Zero imports. Expo SDK peer; may have been scaffolded and never wired. |
| `expo-crypto` | Zero imports. Expo SDK peer. |
| `expo-symbols` | Zero imports. iOS-only SF Symbols; Expo SDK peer. |
| `expo-system-ui` | Zero imports. Expo SDK peer, often used internally. |
| `expo-font` | Zero imports. Expo SDK peer, often used internally by `expo-splash-screen`. |
| `expo-linking` | Zero imports. Used internally by `expo-router`. |
| `expo-status-bar` | Zero imports. Expo SDK peer. |
| `expo-secure-store` | Zero imports, but listed as a config plugin in `app.config.ts`. May be used only at the native layer. |
| `expo-auth-session` | Zero imports. |
| `@react-navigation/elements` | Zero imports. Peer of `@react-navigation/native`. |
| `@rn-primitives/label` | Zero imports. |
| `@rn-primitives/separator` | Zero imports. |
| `tailwindcss-animate` | Not referenced in `tailwind.config.js`'s `plugins` array (which is empty). |
| `dotenv` | Zero imports in app/src code. May be used by a build-time script. |
| `react-dom` | Zero imports. Required Expo web support peer. |
| `react-native-worklets` | Zero imports. Peer dependency of `react-native-reanimated`. |

Many of the above are Expo SDK peers or transitive plugin requirements that don't need explicit imports. The clearly superfluous ones are `sonner-native` (duplicate of the active `react-native-toast-notifications`), `expo-auth-session`, `tailwindcss-animate`, and `dotenv`.

**Packages used but not in `package.json`:**

| Package | Import sites |
|---------|-------------|
| `expo-web-browser` | `src/lib/services/authService.ts`, `src/lib/hooks/useGoogleLogin.ts` |

`expo-web-browser` is imported in two files but is not declared in `package.json`. It resolves today through the Expo SDK's transitive dependency tree, but is not guaranteed to remain available.

---

## Section 4 — Local dev setup

**`.env` keys expected (names only):** Both `.env.development` and `.env.production` declare the same 11 keys: `APP_ENV`, `EXPO_PUBLIC_API_URL`, `EXPO_PUBLIC_CDN_URL`, `EXPO_PUBLIC_WATERMARK_ENABLED`, `EXPO_PUBLIC_FIREBASE_API_KEY`, `EXPO_PUBLIC_FIREBASE_AUTH_DOMAIN`, `EXPO_PUBLIC_FIREBASE_PROJECT_ID`, `EXPO_PUBLIC_FIREBASE_APP_ID`, `EXPO_PUBLIC_FIREBASE_STORAGE_BUCKET`, `EXPO_PUBLIC_FIREBASE_MESSAGING_SENDER_ID`, `EXPO_PUBLIC_WEB_CLIENT_ID`. No `.env.example` exists — the `.env.development` and `.env.production` files serve as the implicit template.

**Backend URL configuration:** The backend URL is configured via the `EXPO_PUBLIC_API_URL` env var, read directly in `src/lib/config/api.ts` as the axios base URL. A separate hardcoded fallback exists in `app.config.ts` (`http://10.243.163.91:8080/api`) under `extra.apiBaseUrl`, but this value is only consumed via `Constants.expoConfig.extra` — the main networking layer reads `EXPO_PUBLIC_API_URL` from the environment, not from `extra.apiBaseUrl`. The two sources are independent and could diverge.

**Platform-specific dev quirks:** The README is the default Expo scaffold boilerplate and documents no project-specific quirks. The `.npmrc` sets `node-linker=hoisted` and `enable-pre-post-scripts=true`.

**Additional local processes:** None documented. The project depends on Metro and the backend only. No tunneling, websocket server, or mock service is referenced.

---

## Section 5 — Native config surface

**iOS permissions declared:**
- `UIBackgroundModes: ['remote-notification']` — in `app.config.ts` → `ios.infoPlist`.
- `usesNonExemptEncryption: false` — in `ios.config`.

**Android permissions declared:** None explicitly declared in `app.config.ts`. The `expo-notifications` config plugin and `@react-native-google-signin/google-signin` plugin may inject permissions at prebuild time.

**Deep linking / URL scheme:** `scheme: 'oglasino'` declared at the top level of `app.config.ts`. No associated domains declared.

**Push notification setup:** Present. `expo-notifications` config plugin with:
- Notification icon: `./assets/images/logo/light-oglasino-96.png`
- Color: `#000000`
- Default channel: `default`
- `enableBackgroundRemoteNotifications: true`

The push token registration is wired in `src/notifications/lib/pushNotificationRegister.ts` and `src/notifications/components/PushNotificationsInit.tsx` using `expo-notifications` and `expo-device`. Token is sent to the backend via `src/notifications/service/pushTokenService.ts`.

**Expo config plugins listed:**
1. `expo-router`
2. `expo-splash-screen` (with light/dark image config)
3. `@react-native-google-signin/google-signin`
4. `expo-secure-store`
5. `expo-dev-client`
6. `expo-notifications`

---

## Section 6 — Localization setup

**i18n library:** `i18next` (v25.10.10) + `react-i18next` (v16.6.6) + `i18next-icu` (v2.4.3). The ICU plugin enables ICU message format (plurals, selects). `expo-localization` is in `package.json` but no direct imports were found — locale resolution is not wired through it.

**Translation files:** No local translation files exist in the repo. All translations are fetched from the backend at runtime, per-namespace, via `src/i18n/fetchNamespace.ts` → `GET /public/translations?namespace=<NS>&lang=<lang>`. The `TranslationNamespace` enum in `src/i18n/types.ts` lists 20 namespaces matching conventions Part 6 Rule 1 (except `BACKEND_TRANSLATIONS` is absent from the mobile enum).

**Locales with files today:** N/A — no local locale files. The backend serves whatever locales it has seeded (EN, SR, RU, CNR per conventions).

**Active locale resolution:** The `initI18n(lang, tenant)` function in `src/i18n/i18n.ts` is called with a `lang` parameter from the app initialization flow. The fallback language is hardcoded to `'sr'`. Preferred language is persisted to AsyncStorage via `src/i18n/storage.ts` (`setStoredLanguage`, `getStoredLanguage`). No `expo-localization` call was found to detect the device locale — the language appears to be selected by the user via the base-site/language selector during app init, not auto-detected from the device.

---

## Section 7 — Auth wiring

**Firebase Auth integration:** Firebase JS SDK (`firebase/auth`). Initialized in `src/lib/client/firebaseClient.ts` with `initializeAuth(app, { persistence: getReactNativePersistence(ReactNativeAsyncStorage) })`. Not `@react-native-firebase/auth` — this uses the modular web SDK with React Native persistence adapter.

**User token storage:** Firebase Auth persists its own tokens via the `getReactNativePersistence(AsyncStorage)` adapter. A separate `src/lib/storage/authStorage.ts` stores an `access_token` key in AsyncStorage, though no callers of `setAuthToken` were found via the import analysis — this may be vestigial.

**Token attachment to backend requests:** Via an axios request interceptor in `src/lib/config/api.ts`. On every request, the interceptor calls `auth.currentUser.getIdToken()` and sets the `Authorization: Bearer <token>` header. The `getIdToken()` call automatically refreshes the Firebase ID token if it's expired.

**Token-refresh path on 401:** There is no explicit 401 retry interceptor. The axios response interceptor handles `ERR_NETWORK`, `ECONNABORTED`, and `404` but does not intercept `401` responses. Token refresh is implicit through Firebase's `getIdToken()` auto-refresh, but if a request arrives at the backend with an expired token that wasn't refreshed in time, the 401 is propagated to the caller without retry.

---

## Section 8 — State management

**State library:** Zustand (v5.0.11) — matches the web project.

**Top-level stores/slices (12):**

| Store | Location |
|-------|----------|
| `authStore` | `src/lib/store/authStore.ts` |
| `useChatStore` | `src/lib/store/useChatStore.ts` |
| `useChatBlockStore` | `src/lib/store/useChatBlockStore.ts` |
| `useCatalogStore` | `src/lib/store/useCatalogStore.ts` |
| `useFilterStore` | `src/lib/store/useFilterStore.ts` |
| `useCardSizeStore` | `src/lib/store/useCardSizeStore.ts` |
| `usePortalScope` | `src/lib/store/usePortalScope.ts` |
| `useFavoritesStore` | `src/lib/store/useFavoritesStore.ts` |
| `useNotificationStore` | `src/notifications/store/useNotificationStore.ts` |
| `viewTokens` | `src/lib/stores/viewTokens.ts` |
| `uploadProgress` | `src/lib/stores/uploadProgress.ts` |
| `useDialogStore` | `src/components/dialog/store/useDialogStore.ts` |

Note: stores are split across two directories — `src/lib/store/` (singular) and `src/lib/stores/` (plural). The image-pipeline stores live in `stores/`; the rest live in `store/`.

**Persistence:** AsyncStorage is used for auth token persistence (`authStorage.ts`), catalog cache (`catalogStorage.ts`), and language preference (`i18n/storage.ts`). No MMKV usage. No `expo-secure-store` imports found despite the package being installed and declared as a config plugin — `expo-secure-store` is not used at the application layer.

---

## Section 9 — Networking layer

**HTTP client:** axios (v1.13.6).

**Base URL:** Centralized. `EXPO_PUBLIC_API_URL` env var → `BACKEND_API` singleton instance in `src/lib/config/api.ts`. All backend calls go through this instance.

**Error handling:** Single centralized error-handling path in the axios response interceptor (`src/lib/config/api.ts:33-61`). The interceptor catches network errors (`ERR_NETWORK`, `ECONNABORTED`), no-response errors, and 404s, enriching each with a synthetic `errorCode` string. All other error responses are passed through to callers. Individual services have their own try/catch patterns (e.g., `configurationService.tsx`, `authService.ts`) but the interceptor is the single shared boundary.

**Request headers:** The request interceptor attaches `X-Base-Site` (from `apiStore.ts` module-level variable), `X-Lang` (same), and `Authorization: Bearer <token>` (from Firebase Auth) on every request. Default timeout is 8 seconds.

---

## Section 10 — Project-level dead code

**Stale files in the repo root:**

| File | Issue |
|------|-------|
| `commands.json` | Contains four build commands not referenced by `package.json` scripts or any code. Appears to be a manual reference file. |
| `oglasino-dev-firebase-adminsdk-fbsvc-002e6b2f58.json` | Firebase Admin SDK service account key file. **Git-tracked.** Not imported by any source file — this is a backend/server credential that has no purpose in a client-side Expo app. |
| `@oglasino__oglasino-expo.jks` | Android keystore file. Present on disk but gitignored (`*.jks` in `.gitignore`). Not tracked. |
| `start-session.sh` | Shell script at root, untracked (listed in git status). |

**Missing files referenced by config:**

| Reference | Expected file | Status |
|-----------|---------------|--------|
| `app.config.ts:12` | `GoogleService-Info.prod.plist` | **Missing.** Referenced for production iOS builds. |
| `app.config.ts:14` | `google-services.prod.json` | **Missing.** Referenced for production Android builds. |
| `package.json:6` (`reset-project` script) | `scripts/reset-project.js` | **Missing.** The `scripts/` directory does not exist. |

**Dead/vestigial code in source files:**

| Location | Issue |
|----------|-------|
| `src/lib/client/firebaseClient.ts:38-52` | `testConnection()` function and its commented-out invocation (`// testConnection()`). Dead code — function defined, never called. |
| `src/lib/client/firebaseAnalytics.ts:7` | `typeof window !== 'undefined'` guard. This is a web-only pattern that never evaluates meaningfully in React Native (RN has no `window` in the browser sense — the global is polyfilled differently). The entire file's approach (using `firebase/analytics` `getAnalytics()`) may not work in RN without `@react-native-firebase/analytics`. |
| `src/i18n/loader.ts:8-17, 22-23` | Extensive commented-out cache/version logic. The `storedVersion` variable on line 7 is assigned but never read. |
| `src/i18n/fetchNamespace.ts:50-56` | `validateVersion()` function — always returns `true`, never called. |
| `src/lib/services/authService.ts:200-228` | `loginWithFacebookFirebase()` — entire function body is commented out, returns `null`. |
| `src/lib/storage/authStorage.ts` | `setAuthToken`, `getAuthToken`, `clearAuthStorage` — no callers found. Firebase Auth handles its own persistence via `getReactNativePersistence(AsyncStorage)`. May be vestigial from a pre-Firebase auth approach. |
| `app/__smoke__/upload.tsx` | Smoke test harness for image upload. README documents it as "Deletable as a unit (`rm -r app/__smoke__/`)". |

**Unused Expo config plugins:** None conclusively unused — all six plugins in `app.config.ts` correspond to installed packages, though `expo-secure-store`'s plugin is declared without any application-layer usage of the package.

**Scripts referencing nonexistent files:** `npm run reset-project` → `node ./scripts/reset-project.js`. The file and directory do not exist.

**TODOs/FIXMEs in entry points:** None found in `app/_layout.tsx`, `app/+not-found.tsx`, or `app/(portal)/_layout.tsx`.

---

## Section 11 — For Mastermind

**Structural-level findings (build/start blockers):**

- **Production Firebase config files missing.** `GoogleService-Info.prod.plist` and `google-services.prod.json` are referenced in `app.config.ts` for production builds but do not exist on disk. Production EAS builds will fail until these are supplied. Dev config files (`GoogleService-Info.dev.plist`, `google-services.dev.json`) exist and are git-tracked.
- **`scripts/reset-project.js` missing.** `npm run reset-project` will error. Low severity — this is a scaffolding artifact, not a production path.

**Cross-cutting findings affecting multiple feature audits:**

- **Firebase Analytics may not work on RN.** `src/lib/client/firebaseAnalytics.ts` uses `firebase/analytics` (the web SDK's analytics module) with a `typeof window !== 'undefined'` guard. The web SDK's analytics module is not designed for React Native — it depends on browser APIs (`document.cookie`, `navigator`, etc.). The GA4 v1 mobile adoption (Expo backlog) will need to evaluate `@react-native-firebase/analytics` or `expo-firebase-analytics` as noted in the backlog entry.
- **Two toast libraries installed.** `react-native-toast-notifications` is actively used (5+ import sites). `sonner-native` is installed but has zero imports. Any feature audit touching notifications/toasts should know which library is canonical.
- **`expo-secure-store` installed and plugin-declared but unused.** Any feature audit touching secure storage (auth tokens, sensitive data) should know that SecureStore is available at the native layer but not wired at the application layer. Auth tokens are in plain AsyncStorage.
- **Firebase Admin SDK service account key is git-tracked.** `oglasino-dev-firebase-adminsdk-fbsvc-002e6b2f58.json` is committed to the repo. This is a server-side credential with no purpose in a client app. Not a mobile feature concern — flagging for the admin-removal audit or security review.
- **`.env.development` and `.env.production` are git-tracked with values.** Contains Firebase API keys (which are client-side and semi-public by design in Firebase's model, but still credentials). Standard Expo practice for development; worth awareness.

**One-liners for other audit scopes:**

- **Image pipeline audit:** `app/__smoke__/upload.tsx` exists as a deletable day-one smoke harness per the README module map.
- **Messaging audit:** `src/lib/client/firebaseClient.ts` initializes Firestore (`getFirestore`) and Storage (`getStorage`). The `useChatStore` and `useChatBlockStore` Zustand stores exist. `firebaseNotifications.ts` has a typo in `NotificationType.NORMAL = 'NOTMAL'`.
- **User deletion audit:** `loginWithFacebookFirebase` returns `null` and is fully commented out — no Facebook auth path exists to test against deletion flows. `logoutFirebase` calls `GoogleSignin.signOut()` only — no `auth.signOut()` from Firebase Auth is called, which means Firebase Auth state persists after "logout".
- **Backend calls reduction audit:** The i18n loader fetches all 20 translation namespaces individually via 20 parallel `GET /public/translations` calls on every app init (no caching is active — the cache logic in `loader.ts` is commented out).
- **Consent mode v2 audit:** No consent mechanism exists on mobile. `COOKIES` namespace is declared in `TranslationNamespace` but no consent UI or gating is implemented.

---

## Section 12 — Cleanup performed

`none needed` — read-only audit.

---

## Section 13 — Obsoleted by this session

`nothing` — read-only audit.

---

## Section 14 — Conventions check

- Part 4 (cleanliness): N/A this session — no code modified.
- Part 4a (simplicity): N/A this session — no code modified.
- Part 4b (adjacent observations): Findings routed to Section 11 "For Mastermind" per convention.
- Part 6 (translations): N/A this session — structural localization inventory only, no key audit.
- Other parts touched: Part 11 (trust boundaries) — noted that `authStorage.ts` stores tokens in plain AsyncStorage rather than SecureStore; the Firebase Admin SDK service account key is committed to the repo. Both are structural observations, not trust-boundary violations in the mobile client itself.
