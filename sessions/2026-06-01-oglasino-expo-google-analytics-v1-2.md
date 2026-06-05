# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-06-01
**Task:** GA4 Mobile v1 — Brief 1 (foundation + auth/page_view). Build the analytics foundation (`track`/`trackError` wrapper, init+ATT+consent gating, `user_id` sync), wire `page_view` and `sign_up`/`login` with `wasRegister` plumbing, delete the dead `firebaseAnalytics.ts`. Prove the logic with unit tests; no build (Igor rebuilds after Brief 3).

## Implemented

- **Dependencies (SDK-pinned via `npx expo install`):** `@react-native-firebase/app@^24`, `@react-native-firebase/analytics@^24`, `expo-tracking-transparency@~6.0.8`. No build, no native config touched (see Brief vs reality + Config-file impact).
- **The `track` wrapper (single entry point):** `src/lib/analytics/track.ts` — `track(event, params?)` wraps Firebase `logEvent` and is the ONLY `logEvent` call site. Three ordered no-op guards (never throws): (1) SDK not initialized, (2) `isAnalyticsConsentGranted()` false, (3) analytics instance absent. Also `trackError(err, context)` — normalizes to `Error`, caps message at 200 chars, fires `exception` (consumed by Brief 3; built now so the module is complete).
- **SDK seam:** `src/lib/analytics/analyticsClient.ts` — owns the analytics instance + init state. `syncAnalyticsState()` reconciles the SDK with the gate `(ATT-allows-on-iOS, or Android) AND isAnalyticsConsentGranted()`: lazily inits + enables collection when open, disables collection when closed (reactive consent-off without restart). `setAnalyticsUserId()` and `setAttAllowed()` exposed. No `logEvent` here (kept solely in `track.ts`).
- **Init + ATT + consent reactivity + `user_id`:** `src/components/init/AnalyticsInit.tsx`, mounted from `AppInit` (per consent-mode-mobile decision: AppInit is the analytics-init home). Effect 1: iOS ATT prompt (`requestTrackingPermissionsAsync`) at launch *before* init, then reconcile + seed `user_id`. Effect 2: subscribes to `useConsentStore`, re-reconciles only when `decision.analytics` flips. Effect 3: subscribes to `useAuthStore`, sets the GA4 `user_id` (stringified numeric `AuthUserDTO.id`) on id change, clears on sign-out — mirrors web's `GA4UserIdSync`.
- **`page_view`:** `src/components/analytics/GA4RouteListener.tsx` — `usePathname()` listener firing `track('page_view', { page_path })` on path change. Mounted inside the navigator in BOTH in-navigator layouts (`app/(portal)/_layout.tsx`, `app/owner/_layout.tsx`) so every route tree is covered; a **module-scoped `lastTrackedPath`** dedupes across the two mounts so a path fires exactly once and a layout remount on an unchanged path does not re-fire.
- **`wasRegister` plumbing:** added `wasRegister: boolean` to `AuthUserDTO` (`src/lib/types/user/AuthUserDTO.ts`). `syncUserToBackend` already returns the wire `AuthUserDTO` (the runtime spread carried the field; it was only untyped), so typing the DTO surfaces it to callers with no service-code change.
- **`sign_up`/`login`:** `src/lib/analytics/authEvents.ts` — `trackAuthEvent(user)` fires `sign_up` when `user.wasRegister`, else `login`, with `method` (from `auth.currentUser.providerData[0].providerId`: `password→email`, `google.com→google`, `facebook.com→facebook`) and stringified numeric `user_id`. Wired into the explicit auth methods `login()`/`register()`/`loginWithGoogle()` in `authStore.ts`, **after** the successful `set({ user })` (banned paths throw before `setUser` → no event). The `onIdTokenChanged` listener fires nothing (prevents a spurious `login` on every cold-start rehydration). Google guards `if (backendUser)` because the service returns `undefined` on cancel.
- **Deleted** `src/lib/client/firebaseAnalytics.ts` (dead web-SDK vestige; grep re-confirmed zero importers before deleting).

## Files touched

- `src/lib/analytics/track.ts` (new) — `track` + `trackError`.
- `src/lib/analytics/analyticsClient.ts` (new) — SDK seam: init/gate/collection toggle/`user_id`.
- `src/lib/analytics/authEvents.ts` (new) — `providerIdToMethod` + `trackAuthEvent`.
- `src/components/init/AnalyticsInit.tsx` (new) — ATT + init + consent/user_id reactivity.
- `src/components/analytics/GA4RouteListener.tsx` (new) — `page_view` listener.
- `src/lib/analytics/track.test.ts` (new) — guard + dispatch + trackError tests.
- `src/lib/analytics/authEvents.test.ts` (new) — method mapping + event discrimination + PII-free payload.
- `src/lib/store/authStore.test.ts` (new) — explicit methods fire; listener fires nothing.
- `src/lib/types/user/AuthUserDTO.ts` — `+ wasRegister: boolean`.
- `src/lib/store/authStore.ts` — import `trackAuthEvent`; fire after `setUser` in 3 explicit methods.
- `src/components/init/AppInit.tsx` — mount `<AnalyticsInit />`.
- `app/(portal)/_layout.tsx`, `app/owner/_layout.tsx` — mount `<GA4RouteListener />`.
- `src/lib/services/authService.test.ts` — `+ syncUserToBackend wasRegister` describe.
- `src/lib/client/firebaseAnalytics.ts` — **deleted**.
- `package.json` / `package-lock.json` — the 3 new deps.

## Tests

- `npx tsc --noEmit`: **clean** (exit 0).
- `npx vitest run`: **360/360 green** (29 files), including +21 new specs (track 10, authEvents 9 [excl. shared describe split], authStore 5, authService +2).
- `npx expo lint`: **no new errors.** Net delta = **+4 warnings, all `import/first` in test files** — identical to the repo's established vitest mock-before-import pattern (e.g. the pre-existing `analyticsGate.test.ts`, `authService.test.ts` already emit it) and inherent to the required tests. My source (non-test) files add zero warnings. The single `error` reported (`react/jsx-key` in `DashboardSidebar.tsx`) is **pre-existing** in the working tree (that file is `M` from earlier uncommitted work; I did not touch it).
- `npx expo-doctor`: 17/18 pass. The one failure (`packages match versions required by installed Expo SDK`) lists **patch** drift on 11 pre-existing Expo packages (`expo`, `expo-router`, …) — **none of my 3 new packages**, which installed SDK-pinned and clean. Pre-existing, unrelated to this session.

## Brief vs reality

1. **"Disable automatic `screen_view` collection at init" is build-time native config, not a runtime call.**
   - Brief/spec says: disable auto `screen_view` "at init."
   - Reality: `@react-native-firebase/analytics` has **no runtime API** to toggle only automatic screen reporting. It is controlled by `firebase.json` → `react-native.google_analytics_automatic_screen_reporting_enabled: false`, read at prebuild/build time. There is no `firebase.json` in the repo (native analytics is greenfield).
   - Why it matters: without this flag the native SDK auto-collects `screen_view`, double-counting against our custom `page_view`. But `firebase.json` is native config, and the hard rules forbid native-config edits without explicit file-level instruction; the brief defers the build to Igor after Brief 3.
   - Resolution: **not created this session.** Added to the rebuild checklist below for Igor. (No runtime code was written for it — writing a call to a nonexistent API would be dead code.)

2. **The 3 new deps do not function on a build until native wiring lands — all of it native config (Igor's rebuild).** `expo install` itself printed that `@react-native-firebase/app` and `expo-tracking-transparency` must be added to `app.config.ts` `plugins`. Per the hard rules I did **not** edit `app.config.ts` or any native config. The JS layer is complete and unit-proven; nothing fires until Igor's rebuild wires the native side (consistent with the brief's "nothing fires until a later rebuild"). Full checklist in "For Mastermind."

3. **`wasRegister` surfaces with a type-only change.** The brief says "type the response to include it and return it to callers (today the field arrives on the wire but is dropped)." In fact the runtime spread `{ ...userData, firebaseUid }` already copies it — it was only *untyped*. Adding `wasRegister` to `AuthUserDTO` makes it typed end-to-end; no `authService.ts` logic change was needed. (Implemented exactly the intended outcome — noted only so the "no service change" isn't read as a miss.)

## Cleanup performed

- Deleted the dead `src/lib/client/firebaseAnalytics.ts` (zero importers, re-confirmed by grep). No commented-out code, no debug logging, no unused imports/vars introduced. No `TODO`/`FIXME` left in code.

## Config-file impact

- **conventions.md:** no change.
- **decisions.md:** no change owed by this session (the auth-chokepoint and listener-fires-nothing decisions were already set in the brief/spec; this session implements them).
- **state.md:** the Expo-backlog "Google Analytics v1" row should **not** flip to adopted yet — this is Brief 1 of 3 and no event fires until Briefs 2–3 + Igor's rebuild land. Recommend it move (or stay) at an in-progress marker, not `stable`/adopted. Draft for Docs/QA in "For Mastermind"; I did not edit `state.md`.
- **issues.md:** no new defect entries. The native-wiring checklist is planned rebuild work, not a loose bug; captured here for Igor.

## Obsoleted by this session

- `src/lib/client/firebaseAnalytics.ts` — deleted (the `-1` audit flagged it; this session removes it, fulfilling that recommendation).

## Conventions check

- **Part 4 (cleanliness):** lint/tsc/test green for touched paths; dead file removed; no debug logging or dead code. The only lint residue is the test-file `import/first` pattern that is house style.
- **Part 4a (simplicity):** see three-category evidence in "For Mastermind."
- **Part 4b (adjacent observations):** the pre-existing `react/jsx-key` error in `DashboardSidebar.tsx` and the patch-version drift on 11 Expo packages are noted (not mine, not fixed — out of scope).
- **Part 6 (translations):** N/A — no user-facing strings; GA4 errors render via existing `translationKey` lookups elsewhere, untouched here.
- **Part 8 (routes reusable):** confirmed — reuses web's `/auth/firebase-sync`; no mobile-specific route added.
- **Part 11 (trust boundary):** **confirmed N/A risk.** GA4 is write-only client telemetry: `track` → `logEvent` ships to Google Analytics only. No event payload is read by the oglasino backend or feeds any server-side decision; `user_id` is set on the analytics SDK alone. PII discipline held — payloads carry numeric IDs + enums only (no email/displayName/phone/body/image keys); unit-asserted for the auth payload.

## Known gaps / TODOs

- **Native rebuild wiring is required before any event fires** (all Igor / native config — see checklist). No code TODO/FIXME left.
- `product_view`, create/contact/message/search/filter events (Brief 2), `exception` boundary + global handler and `form_submit_failed` (Brief 3) are out of scope and untouched. `trackError` exists but has no call site yet (by design — Brief 3 wires it).

## For Mastermind

- **Native-rebuild checklist (Igor, after Brief 3 — none done this session, all native config):**
  1. `app.config.ts` `plugins`: add `@react-native-firebase/app` and `expo-tracking-transparency` (the latter with an `NSUserTrackingUsageDescription` string, or set it in `ios.infoPlist`). `expo install` explicitly requested the first two.
  2. iOS: `@react-native-firebase/*` requires static frameworks — add `expo-build-properties` with `ios.useFrameworks: 'static'` (this is the usual RNFirebase+Expo requirement; confirm at prebuild).
  3. `firebase.json` at repo root: `{ "react-native": { "google_analytics_automatic_screen_reporting_enabled": false } }` — disables auto `screen_view` (the "at init" item from Brief 1 §3; build-time, not runtime — see Brief vs reality #1).
  4. `GoogleService-Info.*.plist` / `google-services.*.json` are already wired in `app.config.ts` per env — confirm they correspond to the GA4-linked Firebase project before the rebuild.
  5. New mobile GA4 data stream in the **same** web property (Igor, admin-side), then Firebase DebugView smoke on a real device after Brief 3.
- **DebugView note:** on `@react-native-firebase`, DebugView is enabled by the native debug flag (`-FIRDebugEnabled` iOS launch arg / `adb setprop debug.firebase.analytics.app <pkg>`), **not** by a `debug_mode` event param. So the web `EXPO_PUBLIC_GA4_DEBUG_MODE`→`debug_mode` param (present in `.env`) does **not** apply on mobile; I deliberately did not add it to the wrapper (it would be a no-op junk param on native).
- **Part 4a simplicity evidence (required):**
  - *Added (earned complexity):* a 3-file analytics module (`track`/`analyticsClient`/`authEvents`) — the seam split is load-bearing: it keeps `logEvent` to one call site (brief mandate), isolates the native SDK behind a mockable boundary (the unit tests depend on it), and keeps `authStore`→`analytics` acyclic. The module-scoped `lastTrackedPath` in the page_view listener is earned: it is the minimal way to dedupe across the two required mount points without a shared store.
  - *Considered and rejected:* (a) a single `GA4RouteListener` mount — rejected, no single in-navigator layout covers both `(portal)` and `owner`, and the brief warns `usePathname()` doesn't resolve in `AppInit`; two mounts + module dedup is simpler than restructuring routing. (b) Web's `{ user, wasRegister }` tuple return from `syncUserToBackend` — rejected in favor of the brief's "put it on `AuthUserDTO`" (zero service-code change; the spread already carried it). (c) Reading `method` by threading it through `buildUserSession` — rejected for `auth.currentUser.providerData`, which the audit confirms is the canonical source and avoids touching 4 service signatures.
  - *Simplified or removed:* deleted the dead `firebaseAnalytics.ts`; no `authService.ts` logic change (type-only `wasRegister` surfacing).
- **state.md draft (Docs/QA):** keep "Google Analytics v1" mobile at an in-progress marker; do not mark adopted until Briefs 2–3 + the native rebuild land. No edit made by me.
- **Open decision confirmed downstream:** `register()`/`login()`/`loginWithGoogle()` all route through the single `trackAuthEvent` which derives `sign_up` vs `login` from the authoritative `wasRegister` — so a Google first-time sign-in correctly emits `sign_up`, a returning Google login emits `login`, with no per-method hardcoding.
