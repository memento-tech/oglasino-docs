# Audit — Expo readiness: Google Analytics v1

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-05-23
**Mode:** read-only audit. No code changes, no commits.

---

## Section 1 — Current analytics surface

**Is `firebase/analytics` (web SDK) imported anywhere besides the dead `firebaseAnalytics.ts`?**
No. `grep` for `firebase/analytics` across `src/` and `app/` returns exactly one hit: `src/lib/client/firebaseAnalytics.ts:1`. Zero other files import the web analytics module.

**Is any RN-native analytics SDK installed?**
No. `package.json` contains no entry for `@react-native-firebase/analytics`, `expo-firebase-analytics`, `react-native-google-analytics`, or any other analytics SDK. The only Firebase-related packages are the web SDK (`firebase: ^12.10.0`) and the Google Sign-In native module (`@react-native-google-signin/google-signin: ^16.1.2`).

**Is there any `logEvent`, `gtag`, or `analytics().log*` call anywhere?**
No. Zero matches across the entire `src/` and `app/` trees.

**Is there any session-tracking, screen-view-tracking, or user-identifier-tracking code?**
No. Zero matches for `setUserId`, `setUserProperties`, `screen_view`, or any screen-tracking calls. The `session` keyword appears only in non-analytics contexts (image upload sessions, auth session comments).

**Confirmed: mobile has zero analytics instrumentation.**

---

## Section 2 — Dead analytics code inventory

### `src/lib/client/firebaseAnalytics.ts`

**Imports:** `getAnalytics` from `firebase/analytics` (the web SDK).
**Imports from:** `./firebaseClient` (the shared Firebase app instance).
**Exports:** `getFirebaseAnalytics()` — a lazy-init getter.
**Guard:** `typeof window !== 'undefined'` — a web-only pattern. In React Native, `window` is defined but `firebase/analytics` requires a browser environment with `document` and `navigator`; `getAnalytics()` will throw at runtime. The guard never prevents this because `window !== 'undefined'` evaluates `true` in RN.
**Callers:** Zero. `grep` for `getFirebaseAnalytics` across the entire repo returns only the definition site.
**Contains:** `console.warn` fallback — swallows the error if analytics init fails.

**Verdict:** Dead code. The web team deleted its equivalent during Consent Mode v2 cleanup. Mobile retains it. Safe to delete in a future cleanup or the analytics feature chat.

### Other dead analytics artifacts

- No other files reference `firebase/analytics`.
- No `gtag`, `dataLayer`, or web-analytics imports anywhere.
- No commented-out analytics scaffolding found.
- `firebaseConfig` in `firebaseClient.ts:24` includes `measurementId` from `EXPO_PUBLIC_FIREBASE_MEASUREMENT_ID`. This env var feeds `getAnalytics(app)` on web; on RN it is inert (no consumer). The config key is harmless but vestigial in the absence of an analytics SDK.

**Dead code for deletion:**
| File | Reason |
|------|--------|
| `src/lib/client/firebaseAnalytics.ts` | Zero callers, web-only guard, web SDK incompatible with RN |

---

## Section 3 — Event taxonomy from the spec

The GA v1 spec defines 12 events plus automatic `page_view`. Mobile equivalents mapped below.

| # | Event (spec) | Web trigger surface | Mobile equivalent surface | Cross-platform parameter notes |
|---|-------------|---------------------|---------------------------|-------------------------------|
| 1 | `page_view` | Pathname change via `GA4RouteListener` | `expo-router` route change. Attach at root `_layout.tsx` or `(portal)/_layout.tsx` using `usePathname()` from `expo-router`. | `page_path` must be derived from `expo-router` pathname. Mobile paths differ from web (no locale prefix, no tenant prefix). |
| 2 | `sign_up` | `UseTokenRefresh.tsx` post-`setUser` when `wasRegister === true` | `authStore.register` success path, or `authStore.initAuthListener` if `wasRegister` is surfaced. Currently `syncUserToBackend` does not return `wasRegister` on mobile. | `method` param: same provider mapping needed (`password→email`, `google.com→google`). `user_id` from `AuthUserDTO.id`. |
| 3 | `login` | `UseTokenRefresh.tsx` post-`setUser` when `wasRegister === false` | Same as sign_up — discriminator needed. `authStore.login`, `loginWithGoogle`, `loginWithFacebook` success paths plus `initAuthListener`. | Same as `sign_up`. |
| 4 | `product_view` | `ProductViewTracker` client component on product detail page mount | `app/(portal)/(public)/product/[...productData].tsx` — mount effect after `productDetails` loads. | `product_id`, category IDs, `price`, `currency` — all available from `ProductDetailsDTO`. `is_owner_view` requires comparing `useAuthStore.user.id` to `productDetails.ownerId`. |
| 5 | `product_create_started` | `CreateNewProductDialog.tsx` mount effect at `currentStep === 0` | `src/components/dialog/dialogs/product-creation/AddUpdateProductDialog.tsx` — when `currentStep === 0` (the `ImageSelectionProductDialog` step). | No parameters beyond `user_id`. |
| 6 | `product_create_completed` | `UploadedProductDialog.tsx` success branch | `src/components/dialog/dialogs/product-creation/UploadedProductDialog.tsx:69-72` — inside `if (result)` block when `setUploadSuccess(true)`. | `product_id` from `result.id`; category IDs and `price`/`currency` from `productData`; `image_count` from `productData.imagesData`. |
| 7 | `contact_seller_clicked` | `CallUserButton`, `StartMessageButton`, `FavoriteButton` | Same three components exist on mobile: `src/components/product/CallUserButton.tsx`, `StartMessageButton.tsx`, `FavoriteButton.tsx`. | `method`: `'call'`/`'message'`/`'favorite'`. `product_id`: `CallUserButton` currently receives `userId` only, not `productId` — would need the same prop addition web did. `seller_id` from `userId` prop or `productData.ownerId`. |
| 8 | `message_sent` | `useChatStore.sendMessage` post-`batch.commit()` | `src/lib/store/useChatStore.ts:370` — `sendMessage` function. Fire after successful Firestore write. | `is_new_chat`, `receiver_id`, `product_id`, `block_count`, `has_text`, `has_images` — all derivable from the chat store state. |
| 9 | `search` | `SearchInput.commitSearch` | `src/components/SearchInput.tsx` — the search submission path. Mobile uses the same `SearchInput` component via `TopBar.tsx`. | `search_term` with 100-char truncation. |
| 10 | `view_search_results` | `ViewSearchResultsTracker` on catalog page when search term present | `app/(portal)/(public)/catalog/[...categories].tsx` — mount effect when search filter is active. | `search_term`, `results_count` from the filter response. |
| 11 | `filter_change` | `FilterManager.tsx` URL-sync effect | Mobile filter architecture differs from web. Mobile uses a Zustand `useFilterStore` directly, not URL-sync. The fire point would be wherever filter state commits (the store's filter-update actions). | `filters_active` (category keys only), `sort_order`, `pathname`. |
| 12 | `exception` | `app/error.tsx`, `app/global-error.tsx`, `GA4ErrorListener` | Expo's error boundaries: `app/+not-found.tsx` exists. No global error boundary equivalent exists on mobile. Mobile-specific: `ErrorUtils.setGlobalHandler` for uncaught JS errors, or Expo's `expo-error-reporter`. | `error_name`, `error_message` (200-char truncation), `boundary`. |
| 13 | `form_submit_failed` | Each form's error site | Registration: `src/components/dialog/dialogs/RegisterDialog.tsx`. Login: `LoginDialog.tsx`. Product create: `BasicInfoProductDialog.tsx` / `MetaDataProductDialog.tsx`. | `form_name`, `error_code`, `field`. Mobile uses the same backend wire shape for product validation errors. Client-only forms (register, login) would need synthetic codes. |

**Events with obvious mobile analogs:** all 12 have a direct or near-direct mobile equivalent. No event is web-only or requires redesign.

**Events requiring mobile-specific adaptation:**
- `filter_change`: different mechanism (Zustand store vs URL-sync).
- `exception`: different boundary infrastructure (no Next.js error boundaries; need RN/Expo-specific global error handler).
- `page_view`: different routing primitive (`expo-router` vs Next.js `usePathname`).

---

## Section 4 — SDK choice analysis

### `@react-native-firebase/analytics`

- **Not installed.** Zero references in `package.json`.
- **Requires native modules.** Would need EAS Build (already in use — `eas.json` has `development` and `production` build profiles, CLI version `>= 16.32.0`). Managed Expo workflow with EAS Build supports `@react-native-firebase/analytics` via config plugins.
- **Best parity with web.** Web uses `firebase/analytics` (`gtag.js`). `@react-native-firebase/analytics` is the React Native Firebase SDK's analytics module, which sends events to the same GA4 property via the Firebase Analytics native SDK (Google Analytics for Firebase). Events appear in the same GA4 reports as web events.
- **Requires `@react-native-firebase/app` as a peer.** Neither is installed today.
- **Expo SDK 54 compatibility:** Expo SDK 54 supports React Native Firebase via config plugins. The `expo-dev-client` + EAS Build workflow is required (which the project already uses).

### `expo-firebase-analytics`

- **Not installed.**
- **Status on Expo SDK 54:** `expo-firebase-analytics` was removed from the Expo SDK in SDK 48 (2023). The npm registry shows version 8.0.0 as the latest, published for SDK 47. **It is deprecated and not supported on Expo SDK 54.** Not a viable candidate.

### `expo-tracking-transparency`

- **Not installed.** Zero references in `package.json`.
- **Required regardless of SDK choice on iOS.** Any app that includes a tracking SDK (including `@react-native-firebase/analytics` with IDFA collection) must show the App Tracking Transparency prompt before tracking begins. Without this, Apple will reject the app.
- **Expo SDK 54 compatible.** Listed as a supported Expo module.

### Summary of current state

| Package | Installed? | Expo SDK 54 support | Notes |
|---------|-----------|---------------------|-------|
| `@react-native-firebase/analytics` | No | Yes (via config plugins + EAS Build) | Best parity with web |
| `expo-firebase-analytics` | No | **No** (deprecated since SDK 48) | Not viable |
| `expo-tracking-transparency` | No | Yes | Required for iOS regardless of SDK |
| `@react-native-firebase/app` | No | Yes | Peer dependency of the analytics module |

---

## Section 5 — Consent and ATT prerequisites

### iOS — App Tracking Transparency

- **No ATT prompt code exists on mobile.** `expo-tracking-transparency` is not installed. Zero references to `AppTrackingTransparency`, `ATT`, or `requestTrackingPermissionsAsync` in the codebase.
- **iOS App Store rule:** any app that includes SDKs accessing the IDFA (Identifier for Advertisers) or that perform "tracking" as Apple defines it must present the ATT prompt. `@react-native-firebase/analytics` collects the IDFA by default on iOS; installing it without the ATT prompt will result in App Store rejection.
- **Consequence:** analytics SDK initialization on iOS must be gated on the ATT result. If the user declines, the analytics SDK can still run but must be configured to not collect the IDFA (Firebase Analytics supports this via `setAnalyticsCollectionEnabled`).

### Android

- **No OS-level tracking prompt equivalent.** Android has no ATT equivalent.
- **GDPR / Consent Mode v2:** for users in EU scope, the Consent Mode v2 framework still applies. Mobile currently has no consent UI and no consent state storage. The consent-mode-v2 audit for this repo confirmed this.
- **Consequence:** analytics SDK init on Android for EU users must be gated on a consent signal that does not exist today.

### Mobile consent state today

- **No consent UI exists.** Confirmed by the consent-mode-v2 audit.
- **No consent state is stored.** No `og_consent` equivalent, no AsyncStorage key for consent.
- **`ConsentData` type exists** at `src/lib/types/cookie/ConsentData.ts` — a vestigial type with `{ necessary: boolean; preference: boolean }`. This is the pre-Consent-Mode-v2 shape (web has since moved to a four-signal model). The type is referenced only by `GlobalCookie.ts`, which itself appears to be a web-shaped type retained in mobile.
- **`GlobalCookie.ts` has `cookieConsent: ConsentData`** — but `GlobalCookie` is a web-oriented type. Mobile does not use cookies. This type appears to be dead code carried over from web.

### Prerequisites for iOS analytics shipping

1. `expo-tracking-transparency` must be installed.
2. ATT prompt must be wired at app launch or before analytics SDK initialization.
3. Analytics SDK initialization must be gated on the ATT result:
   - ATT authorized → full analytics with IDFA.
   - ATT denied or not determined → analytics without IDFA (still fires events, just no cross-app identifier).
4. A mobile consent UI must exist for GDPR compliance (separate from ATT — ATT is Apple's mechanism; GDPR consent is the legal requirement for EU users).

---

## Section 6 — User identifier strategy

### Where `user_id` would attach

Web sets `user_id` via `gtag('set', { user_id: String(user.id) })` in `GA4UserIdSync`, gated on `useAuthResolved()`. The identifier is `AuthUserDTO.id` (backend numeric ID, stable, non-PII).

Mobile equivalent:

- **Auth store:** `useAuthStore` at `src/lib/store/authStore.ts`. The `user` field holds `AuthUserDTO | null`. `user.id` is the same backend numeric ID.
- **Auth listener:** `initAuthListener` at `authStore.ts:150-162`. Calls `listenAuthState` (which wraps `onAuthStateChanged` from `firebase/auth` at `authService.ts:240-241`). On auth state change, calls `syncUserToBackend(firebaseUser)`, which POSTs to `/auth/firebase-sync` and returns `AuthUserDTO`.
- **Attach point:** `setUserId` would fire inside the `initAuthListener` callback, after `set({ user: backendUser })` resolves at `authStore.ts:157`. This is the point where the backend-authoritative user identity is available.
- **Clear point:** On logout (`authStore.ts:121-144`), after `logoutFirebase()` and `set({ user: null })`, clear the analytics `userId`.

### `wasRegister` gap

Mobile's `syncUserToBackend` (`authService.ts:125-137`) does not consume or return `wasRegister` from the backend response. The backend has shipped this field on the `/auth/firebase-sync` response (GA v1 brief 3), but mobile's `AuthUserDTO` type does not include it, and `syncUserToBackend` does not extract it.

For mobile to distinguish `sign_up` from `login`, either:
- Add `wasRegister` to `AuthUserDTO` (or return it separately from `syncUserToBackend`).
- Or use a different discriminator (e.g., detect first-time auth via a module-scoped flag in the register flow — less robust).

---

## Section 7 — Screen-view tracking strategy

### Where screen-view tracking would attach

Mobile uses `expo-router` (v6.0.23) for navigation. The file-system-based routing provides `usePathname()` and `useSegments()` hooks.

**Recommended attach points:**

1. **Root layout** — `app/_layout.tsx`. This is the outermost layout and renders for every route. A screen-view tracker component mounted here (sibling to `AppInit`) would capture all route changes.

2. **Portal layout** — `app/(portal)/_layout.tsx`. This covers the main portal routes. If screen-view tracking should be scoped to the portal only (excluding admin/owner dashboard), mount here.

3. **Per-group layouts** — `app/(portal)/(public)/_layout.tsx`, `app/(portal)/(secured)/_layout.tsx`, `app/owner/_layout.tsx`, `app/admin/_layout.tsx`. For scope-specific tracking.

**Recommended pattern for expo-router:**

Expo's documentation recommends using `usePathname()` from `expo-router` in a `useEffect` to fire screen-view events on pathname change. This mirrors web's `GA4RouteListener` pattern. For `@react-native-firebase/analytics`, the equivalent is `analytics().logScreenView({ screen_name, screen_class })`. `useNavigationContainerRef` from `@react-navigation/native` (which expo-router wraps) is the alternative for capturing navigation state changes at a lower level.

---

## Section 8 — Event firing surfaces

For each event with a mobile analog, the file path where the event would fire:

| Event | File path | Attach point |
|-------|-----------|-------------|
| `page_view` | `app/_layout.tsx` (or `app/(portal)/_layout.tsx`) | New tracker component using `usePathname()` |
| `sign_up` | `src/lib/store/authStore.ts:75-86` (register), `src/lib/store/authStore.ts:150-162` (initAuthListener) | After `set({ user: backendUser })` in register or listener callback, gated on `wasRegister` |
| `login` | `src/lib/store/authStore.ts:59-69` (login), `:89-99` (loginWithGoogle), `:105-115` (loginWithFacebook), `:150-162` (initAuthListener) | After `set({ user: backendUser })`, when not a registration |
| `product_view` | `app/(portal)/(public)/product/[...productData].tsx` | Mount effect after `productDetails` and `owner` are loaded |
| `product_create_started` | `src/components/dialog/dialogs/product-creation/AddUpdateProductDialog.tsx` | When `currentStep === 0` and the wizard renders `ImageSelectionProductDialog` |
| `product_create_completed` | `src/components/dialog/dialogs/product-creation/UploadedProductDialog.tsx:69-72` | Inside `if (result)` block, before or after `setUploadSuccess(true)` |
| `contact_seller_clicked` (call) | `src/components/product/CallUserButton.tsx:30` | Top of `handlePress`, after auth and availability checks pass |
| `contact_seller_clicked` (message) | `src/components/product/StartMessageButton.tsx:24` | Inside `handlePress`, happy-path only |
| `contact_seller_clicked` (favorite) | `src/components/product/FavoriteButton.tsx:31` | Inside `handlePress`, add-transition only (when `!isFavorite`) |
| `message_sent` | `src/lib/store/useChatStore.ts:370` | After successful Firestore batch commit in `sendMessage` |
| `search` | `src/components/SearchInput.tsx` | The search submission path (equivalent to `commitSearch` on web) |
| `view_search_results` | `app/(portal)/(public)/catalog/[...categories].tsx` | Mount effect when search filter is active and results are loaded |
| `filter_change` | Filter store update actions (mobile doesn't URL-sync; fire on Zustand state change) | The store's filter-commit point |
| `exception` | Global error handler (new) | `ErrorUtils.setGlobalHandler` or equivalent RN error boundary |
| `form_submit_failed` | `src/components/dialog/dialogs/RegisterDialog.tsx`, `LoginDialog.tsx`, `BasicInfoProductDialog.tsx`, `MetaDataProductDialog.tsx` | Each form's error-handling branch |

---

## Section 9 — Translation / locale considerations

**Does the spec require any analytics events to carry the user's locale?**

No. The GA v1 event catalog does not include a `locale` or `language` parameter on any event. GA4 automatically collects the device's language setting as a user property, so an explicit parameter is unnecessary.

**Where is the locale state today?**

- `src/i18n/i18n.ts` holds the i18next instance. `initI18n(lang, tenant)` sets the active language.
- `src/i18n/storage.ts` persists the selected language to AsyncStorage under the key `app_language` as a `LanguageDTO` object.
- The active language is available via `i18n.language` from the i18next instance.

If a future analytics event needed the locale, `i18n.language` is the source.

---

## Section 10 — Trust boundary check

Analytics is client-side telemetry. Events flow from the mobile client to GA4 via the Firebase Analytics SDK (or `gtag.js` proxy). No event payload feeds any server-side decision.

**Does any analytics-relevant event carry server-trusted state?**
No. All event parameters in the spec are either:
- Client-derived identifiers (`product_id`, `user_id`, `seller_id`) — these are read from backend responses but not used for server-side trust decisions via the analytics path.
- Client-observed state (`is_owner_view`, `is_new_chat`, `filters_active`) — informational, not authoritative.
- No moderation outcomes, no user roles, no server-trusted flags appear in any event payload.

**Does any analytics initialization carry a server-issued token or key?**
No. GA4 measurement IDs are public identifiers. The Firebase config's `measurementId` (already in `firebaseClient.ts`) is a public value. No secret is needed to send events to GA4.

**Verdict: clean.** No trust-boundary concerns for analytics instrumentation.

---

## Section 11 — Dead code

| Item | File | Status |
|------|------|--------|
| `getFirebaseAnalytics()` | `src/lib/client/firebaseAnalytics.ts` | Dead. Zero callers. Web-only `typeof window` guard. Web SDK incompatible with RN runtime. Safe to delete. |
| `measurementId` in `firebaseConfig` | `src/lib/client/firebaseClient.ts:24` | Inert. No consumer on RN. Harmless but vestigial. Can remain (standard Firebase config key) or be removed if the future analytics SDK uses a different config path. |
| `ConsentData` type | `src/lib/types/cookie/ConsentData.ts` | Pre-Consent-Mode-v2 shape (`{ necessary, preference }`). Web has moved to a four-signal model. Only referenced by `GlobalCookie.ts`. Likely dead — consent-mode-v2 audit covers this in detail. |
| `GlobalCookie` type | `src/lib/types/cookie/GlobalCookie.ts` | Web-oriented type with `cookieConsent`, `lang`, card sizes. Mobile does not use cookies. Likely dead — general audit or consent-mode-v2 audit covers this. |
| `allowPreferenceCookies` on `AuthUserDTO` | `src/lib/types/user/AuthUserDTO.ts:13` | Web removed this field end-to-end during Consent Mode v2 (Brief C). Mobile retains it. Backend no longer sends it. Dead field. |

No `gtag` references, no `dataLayer` references, no web-analytics imports beyond the items listed above. No commented-out analytics scaffolding found.

---

## Section 12 — For Mastermind

- **ATT is a hard floor for shipping iOS with analytics.** Without `expo-tracking-transparency` installed and the ATT prompt wired, Apple will reject any build that includes `@react-native-firebase/analytics` (or any SDK that accesses the IDFA). This must precede analytics SDK initialization.

- **There is no current analytics integration.** The future mobile-analytics chat is "design and build from scratch," not "adopt an existing surface." Zero event-logging code exists. Zero analytics SDKs are installed. The dead `firebaseAnalytics.ts` is a web SDK vestige that never worked on RN.

- **`expo-firebase-analytics` is deprecated (removed in SDK 48).** The only viable candidate for GA4 parity is `@react-native-firebase/analytics`, which requires `@react-native-firebase/app` and EAS Build (already in use).

- **`wasRegister` is not consumed on mobile.** The backend ships it on `/auth/firebase-sync`, but mobile's `syncUserToBackend` and `AuthUserDTO` don't include it. The future analytics chat needs to wire this for `sign_up` vs `login` discrimination.

- **`CallUserButton` lacks `productId` prop on mobile.** Web added a `productId` prop to `CallUserButton` during GA v1 brief 6. Mobile's `CallUserButton` (`src/components/product/CallUserButton.tsx`) receives `userId` only. The `contact_seller_clicked` event needs `product_id` — same prop addition needed.

- **Mobile consent UI is a prerequisite.** Both iOS (ATT) and EU users (GDPR/Consent Mode v2) require consent gating before analytics fires. The consent-mode-v2 audit confirmed no consent UI exists on mobile. The consent feature chat must precede or ship concurrently with the analytics feature chat.

- **`allowPreferenceCookies` on `AuthUserDTO` is dead.** Backend removed this field during Consent Mode v2. Mobile retains it. Cross-referenced with the consent-mode-v2 audit findings (not investigated further here — that audit's scope).

- **Cross-feature observations:**
  - `ConsentData` and `GlobalCookie` types appear dead — consent-mode-v2 audit scope.
  - `firebaseClient.ts:38-50` has a commented-out `testConnection()` call with `console.info`/`console.error` — general audit scope.
  - `authService.ts:200-228` (`loginWithFacebookFirebase`) is entirely commented out and returns `null` — general audit scope.

### Questions whose answers would change scope

1. **Does mobile need its own GA4 measurement ID, or does it share web's?** Firebase Analytics for mobile typically reports to the Firebase project's linked GA4 property automatically (via the `google-services.json` / `GoogleService-Info.plist` config). If the same Firebase project is used, events flow to the same GA4 property. If separate stage/prod Firebase projects exist, they'd each have their own linked GA4 property. This affects whether `NEXT_PUBLIC_GA4_MEASUREMENT_ID` has a mobile equivalent or whether it's implicit from the Firebase config.

2. **Should mobile consent follow the same four-signal Consent Mode v2 model as web, or a simplified model?** On mobile, there's no `gtag('consent', ...)` API. Firebase Analytics for mobile has `setAnalyticsCollectionEnabled(bool)` — a binary on/off. The Consent Mode v2 four-signal model may need to be simplified for mobile.

3. **Should mobile analytics ship before or after mobile consent UI?** The brief's Section 5 lists consent as a prerequisite. If analytics must wait for a full consent UI, the dependency chain is: consent UI → ATT prompt → analytics SDK init → event firing. If analytics can ship with ATT-only gating (no GDPR consent UI), the chain is shorter but legally incomplete for EU users.

---

## Section 13 — Cleanup performed

`none needed` — read-only audit.

---

## Section 14 — Obsoleted by this session

`nothing` — read-only audit.

---

## Section 15 — Conventions check

- Part 4 (cleanliness): N/A this session. Adjacent dead-code findings flagged in Section 11.
- Part 4a (simplicity): N/A this session — read-only audit, no code added or removed.
- Part 4b (adjacent observations): flagged in Section 12 (dead `allowPreferenceCookies`, commented-out Facebook login, `testConnection` debug code, dead `ConsentData`/`GlobalCookie` types).
- Part 6 (translations): N/A this session.
- Part 11 (trust boundaries): touched — explicit check in Section 10. Clean finding.
- Other parts touched: Part 8 (architectural defaults — routes are reusable, confirmed mobile reuses `/auth/firebase-sync`), Part 9 (stack reference — Expo SDK 54, expo-router 6.0.23).
