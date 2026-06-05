# Audit — Expo release readiness: backend calls reduction

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-05-23
**Mode:** read-only audit, no code changes
**Auditor scope:** backend call patterns only. Findings in other audits' scopes named in "For Mastermind" and not investigated.

---

## Section 1 — Confirmed-redundant patterns from other audits

### (a) i18n loader fetches 20 namespaces in parallel on every app init

**Files:** `src/i18n/loader.ts`, `src/i18n/fetchNamespace.ts`, `src/i18n/loadNamespaces.ts`, `src/i18n/types.ts`

**Confirmed.** The `TranslationNamespace` enum has 21 values (COMMON, COMMON_SYSTEM, ERRORS, VALIDATION, PAGING, BUTTONS, INPUT, DIALOG, HEADER, FOOTER, NAVIGATION, INTRO, EXTRA_PRODUCTS, COOKIES, MESSAGES_PAGE, DASHBOARD_PAGES, ADMIN_PAGES, ABOUT_PAGE, FREE_ZONE_PAGE, PRICING_PAGE, METADATA). `loadAllNamespaces` at `loadNamespaces.ts:4-7` calls `Promise.all(namespaces.map(ns => fetchNamespace(ns, lang)))`, firing **21 parallel `GET /public/translations?namespace=<NS>&lang=<LANG>` calls** on every init.

**Caching logic is commented out.** At `loader.ts:9-17`, five lines are commented:
- `var validateVersionS = await validateVersion()` — the version-checking call
- `if (validateVersion) { const cached = ... }` — the AsyncStorage cache read
- `await AsyncStorage.setItem(...)` — the cache write
- `await AsyncStorage.setItem(VERSION_KEY, version)` — the version stamp

The `validateVersion()` function at `fetchNamespace.ts:50-56` is a stub that always returns `true` — never implemented.

**Call burst:** 21 HTTP requests fire simultaneously on every app bootstrap (`AppContext.bootstrap` → `initI18n` → `loadTranslations` → `loadAllNamespaces`). This also fires on every language change (`AppContext.setLanguageForCode` → `initI18n`) and every base site change (`AppContext.setBaseSiteForCode` → `initI18n`). Additionally, if the app recovers from maintenance mode, bootstrap runs again, firing another 21.

**Why caching was disabled:** The commented code's intent was: (1) check a backend version endpoint, (2) if versions match, read from AsyncStorage, (3) otherwise fetch fresh and persist. The version endpoint was never implemented (`validateVersion` is a stub returning `true`), so the entire caching path was disabled. The `storedVersion` at `loader.ts:7` is read but never used.

**Impact:** 21 unnecessary backend calls per app init. With caching enabled and a version-check endpoint, this becomes 1 call (version check) on most inits, plus 21 only on version mismatch.

### (b) `POST /auth/firebase-sync` sends `idToken` in the request body AND as Authorization header

**Files:** `src/lib/services/authService.ts:125-137`, `src/lib/config/api.ts:21-29`

**Confirmed.** At `authService.ts:126`, `firebaseUser.getIdToken()` is called explicitly. The result is:
1. Sent in the request body: `{ idToken, profileImageKey: null, providerId: ... }` (line 129)
2. Sent as the Authorization header: `{ headers: { Authorization: 'Bearer ${idToken}' } }` (line 131)

Meanwhile, the global request interceptor at `api.ts:21-29` also calls `auth.currentUser.getIdToken()` and sets `Authorization: Bearer <token>` on every request. The explicit header at line 131 **overrides** the interceptor's header (same key), so there's no functional bug — but `getIdToken()` is called **twice** for this single request (once in `syncUserToBackend`, once in the interceptor).

**What mobile sends:** Both body `idToken` and header `Authorization` carry the same token value. Whether the backend reads from the body, the header, or both cannot be verified from the mobile repo. The body field is redundant if the backend reads the header (which the interceptor already provides).

### (c) Language-change refresh fetches all previously-loaded pages in parallel

**File:** `src/components/product/ProductList.tsx:117-189`

**Confirmed.** The `onRefreshCurrent` function at lines 133-189 runs when `selectedLanguage?.code` changes (detected by the `useEffect` at lines 117-131). It fetches all already-loaded pages in parallel via:

```
for (let page = 0; page <= currentMaxPage; page++) {
  pagePromises.push(fetchPage(page, false));
}
const results = await Promise.all(pagePromises);
```

**Call burst estimate:** If a user has scrolled through 5 pages (pages 0–4) before changing language, this fires **5 parallel `POST /public/product/search`** calls (or `/secure/products` for dashboard, or `/secure/admin/products` for admin). At 10 pages scrolled, it's 10 parallel calls. There is no debounce or throttle on this path.

This is the same pattern the filtering audit confirmed on web. The mobile version is identical in behavior.

### (d) Favorites toggle triggers full product list refresh

**Files:** `src/components/product/FavoriteButton.tsx:26,46`, `src/lib/store/useFavoritesStore.ts:50-53`, `src/components/product/ProductList.tsx:56-61`, `src/components/product/FavoritesProductList.tsx:35`

**Confirmed.** The chain is:

1. `FavoriteButton.handlePress` (line 45) calls `addToFavorites(productId)` — `GET /secure/favorites/addRemove?productId=X` — and then calls `presetClearShouldReload()` (line 46)
2. `presetClearShouldReload()` sets `shouldReload: true` in the favorites store (line 51)
3. `ProductList` watches `shouldReload` (line 57): when true and `refreshOnFavoritesChange` is enabled, calls `onRefresh()` then `clearShouldReload()`
4. `onRefresh()` resets all pagination and refetches page 0

**Who uses `refreshOnFavoritesChange={true}`:** Only `FavoritesProductList.tsx:35`. The favorites list screen passes this prop.

**How many backend calls does a single favorite toggle generate?**
1. `GET /secure/favorites/addRemove?productId=X` — the toggle itself
2. If the user is on the favorites list screen: `POST /secure/favorites/overviews` — the full list refresh (via `fetchPage` → `getFavorites`)

So: 2 calls. On any other screen, only 1 call (the toggle). The `shouldReload` flag persists in memory, so if the user toggles a favorite on the home screen and later navigates to the favorites screen, the favorites screen will fire a refresh on mount.

---

## Section 2 — App init network call audit

Traced every backend call from app launch to first interactive screen.

| # | Call | File:line | When | Required for first paint? | Cacheable? |
|---|---|---|---|---|---|
| 1 | `GET /public/maintenance/active` | `maintenanceService.tsx:4` via `AppContext.tsx:81` | Bootstrap `Promise.all` | Yes (blocks on maintenance gate) | Short TTL (5s poll anyway) |
| 2 | `GET /public/config` | `configurationService.tsx:6` via `AppContext.tsx:82` | Bootstrap `Promise.all` | Yes (null = maintenance mode) | Yes — rarely changes |
| 3 | `GET /public/baseSite/details` | `baseSitesService.ts:19` via `AppContext.tsx:83` | Bootstrap `Promise.all` | Yes (needed for site selection) | Yes — rarely changes |
| 4 | `GET /public/app/version/<platform>?currentVersion=X` | `appVersionConfigService.tsx:10` via `AppVersionConfigInit.tsx:104` | Init + every pathname change + every foreground resume | Yes (blocks children on `preparing`) | Yes — version changes rarely |
| 5–25 | 21× `GET /public/translations?namespace=<NS>&lang=<LANG>` | `fetchNamespace.ts:31` via `loadNamespaces.ts:7` via `i18n.ts:8` via `AppContext.tsx:118` | After base site resolved | Yes (UI strings) | **Yes — caching commented out** |
| 26 | `POST /auth/firebase-sync` | `authService.ts:128` via `authStore.ts:156` | Auth listener fires (persisted session) | No — user can browse without auth | Yes (session-scoped) |
| 27 | `GET /secure/favorites` | `favoritesService.ts:7` via `useFavoritesStore.ts:24` via `InitFavoritesStore.ts:12` | Auth listener fires | No — deferred behind auth | Yes (session-scoped) |
| 28 | `POST /secure/push/token` | `pushTokenService.ts:4` via `pushNotificationRegister.ts:39` | After user identified + permission granted | No | No (side effect) |

**Total at cold start for an authenticated user:** ~28 HTTP calls (3 bootstrap + 1 version check + 21 translations + 1 firebase-sync + 1 favorites + 1 push token).

**Total at cold start for an anonymous user:** ~25 HTTP calls (3 bootstrap + 1 version check + 21 translations).

**Calls deferrable from first paint:** #26–28 (auth-dependent calls). The 21 translation calls (#5–25) are required for first paint but should be 0–1 with caching.

---

## Section 3 — Per-route network call audit

### Route 1: Home (`app/(portal)/(public)/index.tsx`)

- **On mount:** `POST /public/product/search` (page 0) via `FilteredProductList` → `ProductList.loadNextPage`
- **On scroll:** `POST /public/product/search` (page N) per page
- **On language change:** All loaded pages refetched in parallel via `onRefreshCurrent`
- **Duplicates:** The home fires no calls that duplicate init calls.

### Route 2: Product detail (`app/(portal)/(public)/product/[...productData].tsx`)

- **On mount (sequential):**
  1. `GET /public/product/search?productId=X` — product details (`productsSearchService.ts:13-37`)
  2. `GET /public/user?id=Y` — owner info (`userService.ts:52-66`)
  3. `GET /public/product/seen/X` — mark as seen (`productService.ts:79-85`), only if `!fetchedOwner?.iamActive`
- **Extra sections (4 parallel calls):** When categories resolve, 4× `POST /public/product/search` fire for MORE_FROM_SELLER, MORE_FROM_FINAL_CATEGORY, MORE_FROM_SUB_CATEGORY, MORE_FROM_TOP_CATEGORY via `HorizontalExtraProductsListView`
- **Total on mount:** 3 sequential + 4 parallel = **7 backend calls** per product page view
- **Duplicates:** `getPortalProductDetails` calls the same endpoint as the home's product list search but with `?productId=X` (GET not POST). The `getUserForId` call is not cached — visiting the same seller's products twice will re-fetch their info.

### Route 3: Owner dashboard products (`app/owner/dashboard/products/index.tsx`)

- **On mount:** `POST /secure/products` (page 0) via `FilteredProductList` → `ProductList.loadNextPage`
- **On scroll:** `POST /secure/products` (page N) per page
- **On language change:** All loaded pages refetched in parallel
- **Duplicates:** None specific to this route.

### Route 4: Messages (`app/(portal)/(secured)/messages.tsx`)

- **On mount:** No backend HTTP calls. Chat data comes from Firestore via `useChatStore.subscribeToChats`.
- **On user resolution for chats:** `GET /auth/firebase/{uid}` per unique counterparty uid, via `useChatStore.getUserData` → `userService.getUserForFirebaseUid`. Cached in `useChatStore.userCache` (in-memory only, session-scoped).
- **On message subscription:** Firestore `onSnapshot` for messages. Each unique sender/receiver uid triggers a `GET /auth/firebase/{uid}` if not cached.
- **View tokens:** `POST /secure/images/view-tokens` per chat with images, via `useViewTokenStore.getToken`. Cached with TTL + in-flight dedup.
- **Duplicates:** User data fetches across chat list and message views are deduplicated by `userCache`. No duplication with other routes' user fetches (different cache scope — `useChatStore.userCache` vs no cache elsewhere).

### Route 5: User settings (`app/owner/dashboard/user.tsx`)

- **On mount:** `GET /secure/user/update` — fetches current user details for the edit form
- **On save:** `POST /secure/user/update` — submits changes
- **On save with avatar:** Additionally: `POST /secure/images/upload-tokens` + direct R2 upload
- **Duplicates:** The `GET /secure/user/update` call fetches the same user whose auth data was already synced via `POST /auth/firebase-sync` at login. The two endpoints return different DTOs (`UpdateUserDTO` vs `AuthUserDTO`), so both are needed — but the overlap in fields (email, displayName, profileImageKey) means some data is fetched twice.

---

## Section 4 — Polling, intervals, and listeners

### `setInterval` — Maintenance check (5-second poll)

**File:** `src/components/context/AppContext.tsx:143`
**Endpoint:** `GET /public/maintenance/active`
**Frequency:** Every 5 seconds, unconditionally, for the entire app lifetime.
**Impact:** 12 calls/minute, 720 calls/hour. This is the single largest source of unnecessary backend traffic. The 5-second interval is extremely aggressive for a maintenance check. A 30–60 second interval would be ~6–12× more efficient with negligible UX impact.

### `setInterval` — App version check on every pathname change

**File:** `src/components/internals/AppVersionConfigInit.tsx:116-118`
**Endpoint:** `GET /public/app/version/<platform>?currentVersion=X`
**Frequency:** Every time `usePathname()` changes + on every app foreground resume (via `AppState.addEventListener`).
**Impact:** On a user navigating through 10 screens, this fires 10 version-check calls. The version cannot change faster than an app store deployment cycle (days/weeks), so checking on every navigation is wasteful. Once per session or once per foreground resume is sufficient.

### Firestore real-time listeners

| Listener | Path | Component | Fan-out to backend? |
|---|---|---|---|
| `onSnapshot` on `userchats/{uid}/chats` | `useChatStore.subscribeToChats` | `ChatsInit.tsx` | Yes — each snapshot triggers `GET /auth/firebase/{uid}` per unique counterparty (cached) |
| `onSnapshot` on `chats/{chatId}/messages` | `useChatStore.subscribeToMessages` | `useChatStore.ts:241` | Yes — each snapshot triggers `GET /auth/firebase/{uid}` per unique sender/receiver (cached) |
| `onSnapshot` on `userblocks/{uid}/blocked` | `useChatBlockStore.subscribe` | `ChatsInit.tsx` | No backend fan-out |
| `onSnapshot` on `userblocksReverse/{uid}/blockedBy` | `useChatBlockStore.subscribe` | `ChatsInit.tsx` | No backend fan-out |
| `onSnapshot` on `notifications/{uid}/userNotifications` | `useNotificationStore.subscribe` | `NotificationsInit.tsx` | No backend fan-out |

**Fan-out pattern:** The chat listeners (`subscribeToChats`, `subscribeToMessages`) call `getUserData(firebaseUid)` which hits `GET /auth/firebase/{uid}`. This is cached in `useChatStore.userCache`, so the fan-out is bounded by the number of unique counterparties, not the number of snapshots. On first load of the chat list, if a user has 15 chats, this fires up to 15 backend calls to resolve user info (one per counterparty, parallel).

### Infinite re-render loops

No infinite re-render loops detected. The `shouldReload` flag in `useFavoritesStore` is cleared synchronously after triggering the refresh, so it cannot loop.

---

## Section 5 — Cache state inventory

### `catalogStorage.ts` — Catalog cache (AsyncStorage)

- **What's cached:** `CatalogDTO` (the full category tree)
- **TTL:** None. The timestamp is stored but never read for staleness checks.
- **Consulted before backend call:** **Not used at all.** `catalogStorage.saveCatalog` and `loadCatalog` exist but are never called by any code path. The catalog is part of the `BaseSiteDTO` returned by `fetchBaseSites()`, which has no caching.

### `useCatalogStore` — In-memory catalog

- **What's cached:** `CatalogDTO` set via `setCatalog`
- **TTL:** None. Persists until `clearCatalog()` is called.
- **Consulted before backend call:** The `setCatalog` implementation has a guard: `if (state.initialized) return state;` — it only accepts the first write. However, grep shows `setCatalog` is never called from any file in the repo. The store exists but is unused.

### `useFilterStore` — Filter state (memory only)

- **What's cached:** Search text, selected filters, price range, regions, product states, moderation states
- **TTL:** None. Memory-only, cleared on navigation or explicit `clearAllFilters()`.
- **Backend impact:** Filter state drives the `ProductsFilterDTO` sent on every product search call. Not a cache in the network-reduction sense — it's UI state.

### `useChatStore.userCache` — User data cache (memory only)

- **What's cached:** `UserInfoDTO` keyed by `firebaseUid`
- **TTL:** None. Persists until `clearChatStore()` or logout.
- **Consulted before backend call:** **Yes.** `getUserData` at `useChatStore.ts:113-122` checks `userCache[firebaseUid]` before calling `getUserForFirebaseUid`. Cache is populated on first fetch and never invalidated (no staleness check, no TTL). User data changes (display name, avatar) won't reflect until the next app restart.
- **Limitation:** This cache is scoped to the chat store. Other components that call `getUserForId` (e.g., the product detail page) have **no cache** — they always hit the backend.

### `useViewTokenStore` — View token cache with TTL

- **What's cached:** Per-chat JWT view tokens for image access, keyed by `chatId`
- **TTL:** Yes — `expiresAt > Date.now() + 60_000` (60-second pre-expiry buffer). Backend issues 4h tokens per contract.
- **Consulted before backend call:** **Yes.** `getToken` at `viewTokens.ts:48-50` checks the cache first. In-flight dedup prevents concurrent duplicate requests for the same `chatId`.
- **Cleared on logout:** Yes, via `useViewTokenStore.getState().clear()` in `authStore.logout`.

---

## Section 6 — Token-refresh, request interceptor patterns

### Interceptor token pattern

**File:** `src/lib/config/api.ts:21-29`

The request interceptor calls `auth.currentUser.getIdToken()` on **every** request where `currentUser` is non-null. Firebase's `getIdToken()` (without `true`) returns the cached token if it hasn't expired and refreshes transparently if it has. This is the standard Firebase pattern — no waste.

### Out-of-band token reads

1. **`syncUserToBackend` at `authService.ts:126`** calls `firebaseUser.getIdToken()` explicitly, then passes the token both as body field and as Authorization header. The interceptor also calls `getIdToken()`, so the token is fetched **twice** for this single request. The second call is a Firebase SDK cache hit (same millisecond), so no network cost — but it's redundant code.

2. **No other out-of-band token reads found.** All other services use `BACKEND_API` which relies on the interceptor.

### Wasteful token-fetch patterns

The only wasteful pattern is the double `getIdToken()` call in `syncUserToBackend`. The explicit call + explicit header at `authService.ts:131` is redundant because the interceptor would set the header anyway. The body `idToken` field may be needed by the backend (cannot verify from mobile), but the explicit `Authorization` header is always redundant.

---

## Section 7 — Cross-fetch coordination

### Same data fetched from two places

1. **User info duplication:** `getUserForId(userId)` (no cache) is called from the product detail page (`product/[...productData].tsx:109`). `getUserForFirebaseUid(uid)` (cached in `useChatStore.userCache`) is called from chat operations. These two functions fetch the same underlying user data via different endpoints (`/public/user?id=X` vs `/auth/firebase/{uid}`) and different cache strategies (none vs in-memory). A user visiting a product page and then messaging that seller fetches the seller's info twice from two different endpoints.

2. **Auth user sync duplication:** `syncUserToBackend` is called from:
   - `authStore.initAuthListener` (line 151-162) — on every `onAuthStateChanged` emission
   - `authStore.login` → `loginUserFirebase` → `buildUserSession` → `syncUserToBackend`
   - `authStore.register` → `registerUserFirebase` → `buildUserSession` → `syncUserToBackend`
   - `authStore.loginWithGoogle` → `loginWithGoogleFirebase` → `buildUserSession` → `syncUserToBackend`
   
   When a user logs in, `buildUserSession` fires `syncUserToBackend`. Then `onAuthStateChanged` fires (because Firebase auth state changed) and `initAuthListener` fires `syncUserToBackend` again. **This is a double-sync on every login.** The same pattern web fixed with a single-flight guard in `UseTokenRefresh.tsx` (decisions.md 2026-05-21).

3. **`ensureUserInFirestore` is called on every login, not just registration.** `buildUserSession` at `authService.ts:142-145` calls `ensureUserInFirestore` then `syncUserToBackend`. The `ensureUserInFirestore` function reads from Firestore (`getDoc`) on every login to check if the user doc exists. For returning users, this is a wasted Firestore read.

### In-flight dedup

- **Present:** `useViewTokenStore` has proper in-flight dedup (`inFlight` map with shared Promise).
- **Absent everywhere else.** No in-flight dedup on:
  - `fetchBaseSites` — could fire twice if bootstrap is triggered during recovery from maintenance
  - `syncUserToBackend` — fires on both explicit login and auth state listener
  - `loadAllNamespaces` — no dedup, but unlikely to fire concurrently
  - `getPortalProducts` / `getPortalProductDetails` — no dedup at the service level

### Missing caches that would reduce backend calls

| Data | Current cache | Recommended | Estimated savings |
|---|---|---|---|
| Translations | None (commented out) | AsyncStorage with version check | 20 calls/init → 0–1 |
| Base sites | None | AsyncStorage with TTL | 1 call/init → 0 on warm start |
| App configuration | None | AsyncStorage with TTL | 1 call/init → 0 on warm start |
| User info (by ID) | None | In-memory with TTL | 1 call/product page → 0 for repeat visits |
| Catalog | Written to storage but never read | Wire up existing `catalogStorage` | Already built, just unused |

---

## Section 8 — Summary of findings by severity

### High impact (address in the adoption feature)

| # | Finding | Calls saved | Effort |
|---|---|---|---|
| H1 | Enable translation caching (uncomment + implement version endpoint) | ~20 calls/init | Medium — backend needs version endpoint |
| H2 | Reduce maintenance polling from 5s to 30–60s | ~600 calls/hour | Trivial — change one constant |
| H3 | Remove version check on every pathname change | ~10 calls/session | Trivial — remove the `useEffect` on `pathname` |
| H4 | Add single-flight guard to `initAuthListener` (mirror web's fix) | 1 call/login | Small — match web's `UseTokenRefresh` pattern |
| H5 | Remove double `getIdToken()` and body `idToken` in `syncUserToBackend` (pending backend verification) | 0 network calls (code cleanup) | Trivial |

### Medium impact (worth addressing)

| # | Finding | Calls saved | Effort |
|---|---|---|---|
| M1 | Cache `getUserForId` results in memory (or unify with chat's `userCache`) | 1+ calls/product page | Small |
| M2 | Skip `ensureUserInFirestore` for returning users | 1 Firestore read/login | Small — check `exists()` result caching or a local flag |
| M3 | Language-change multi-page refetch could be debounced or limited | N calls (N = pages loaded) | Small |
| M4 | Cache base sites and app configuration in AsyncStorage | 2 calls/cold start | Small |

### Low impact (future cleanup)

| # | Finding | Notes |
|---|---|---|
| L1 | `catalogStorage` + `useCatalogStore` are dead code | Not causing extra calls, but misleading |
| L2 | Product detail page fires 7 calls (3 sequential + 4 extra sections) | Normal for the feature; batching the 4 extra section calls into one backend call would be a backend change |
| L3 | `useChatStore.userCache` has no TTL or invalidation | Stale data risk, not extra calls |
| L4 | Favorites toggle → list refresh fires even when not on the favorites screen (flag persists) | Deferred refresh; not extra calls, but unnecessary work on navigate |

---

## For Mastermind

- **No code changes made.** This is a read-only audit.
- **The i18n caching gap (H1) is the single biggest win.** 21 calls per init, every init, with the caching infrastructure already half-built (the AsyncStorage helpers exist, the commented code shows the intent). The missing piece is a backend version endpoint that returns a version number — mobile compares, skips the 21 fetches if versions match. Web has already eliminated this burst.
- **The 5-second maintenance poll (H2) is the most wasteful ongoing pattern.** 720 calls/hour for a check that's meaningful only during deploy windows. Recommend 30–60 seconds, or better: use a push notification or WebSocket for maintenance-mode entry, poll only for recovery.
- **The auth double-sync (H4) is the mobile equivalent of web's `onIdTokenChanged` double-fire.** Web fixed this in the 2026-05-21 `UseTokenRefresh` single-flight session. Mobile's `initAuthListener` has the same gap — both `buildUserSession` and the auth state listener call `syncUserToBackend`. The fix is a single-flight guard matching web's pattern.
- **The `ensureUserInFirestore` call on every login (M2) is a Firestore read that's wasted for 99% of logins** (only the first login creates the user doc). A local flag or a check against the auth store's persisted state could skip it.
- **Cross-audit finding (for the general audit):** `catalogStorage.ts` and `useCatalogStore.ts` are dead code — `saveCatalog`, `loadCatalog`, `setCatalog`, and `clearCatalog` have zero callers. Not backend-calls-related, but worth noting.
- **Cross-audit finding (for the consent-mode audit):** The user settings screen at `app/owner/dashboard/user.tsx:46` still has `allowPreferenceCookies` as a server-persisted field. The Consent Mode v2 feature removed the backend mirror (`allowPreferenceCookies` dropped from `User` entity and DTOs) — mobile will need to align when adopting consent-mode.
