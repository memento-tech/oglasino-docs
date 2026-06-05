# Audit — oglasino-expo production bug sweep (read-only)

**Date:** 2026-06-04
**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Mode:** READ-ONLY — no code changed, nothing staged/committed, no deps, no build.

Five tracked issues verified against current code. Code is ground truth; where the
issue description diverges from the as-built reality, that is called out explicitly.

---

## Item 1 — `getNormalizedProductUrl` hardcodes the production host on all tiers

- **Present?** **YES**
- **As-built location:** `src/lib/utils/utils.ts:148-158` (the implementation overload).
  The hardcoded host is at **`src/lib/utils/utils.ts:156`**.
- **What you found:** The prefixed (outbound) form returns
  `` `https://oglasino.com/${locale}/product/${productId}/${slug}` `` with the apex host
  literal — no tier branch. `extra.env` resolves from `app.config.ts:4,103`
  (`ENV = process.env.APP_ENV ?? 'development'`, surfaced as `extra.env`), but
  `getNormalizedProductUrl` never reads it. So **preview and development** builds emit
  share/outbound links pointing at production; only the `production` tier is coincidentally
  correct. The relative form (`withPrefix=false`, line 157) is host-less and unaffected.
- **`getForgotPasswordUrl` mechanism (`utils.ts:170-178`):** takes `env` as an explicit
  first arg and branches `env === 'production' ? 'https://oglasino.com' : 'https://stage.oglasino.com'`
  (line 174), then appends `compoundLocale` when present, else drops the locale segment. The
  caller supplies the env — `LoginDialog.tsx:83` passes `Constants.expoConfig?.extra?.env`.
  Note the **non-prod host differs**: forgot-password funnels to `stage.oglasino.com`, while
  the product builder has no stage form at all.
- **Recommended fix shape:** Extract one tier→host helper, e.g.
  `getWebBase(env): 'https://oglasino.com' | 'https://stage.oglasino.com'`, and have both
  builders call it. `getNormalizedProductUrl`'s prefixed overload must then accept `env`
  (alongside the existing `locale`), so the two outbound call sites thread
  `Constants.expoConfig?.extra?.env` the same way `LoginDialog` already does.
- **Blast radius — every caller of `getNormalizedProductUrl`:**
  | Caller | file:line | Form | Affected by host bug? |
  |---|---|---|---|
  | ShareProductButton | `src/components/product/ShareProductButton.tsx:34` | prefixed (`true`) | **YES — the share button** |
  | UploadedProductDialog (share URL) | `src/components/dialog/dialogs/product-creation/UploadedProductDialog.tsx:115` | prefixed (`true`) | **YES** |
  | UploadedProductDialog (in-app route) | `…/UploadedProductDialog.tsx:117` | relative | no |
  | SearchInput | `src/components/SearchInput.tsx:152` | relative | no |
  | PortalProductCard | `src/components/product/PortalProductCard.tsx:16` | relative | no |
  | ProductReview | `src/components/product/ProductReview.tsx:56` | relative | no |
  | ExtraProductCard | `src/components/product/ExtraProductCard.tsx:26` | relative | no |
  | PushNotificationsInit | `src/notifications/components/PushNotificationsInit.tsx:84` | relative | no |

  Only **2 outbound call sites** (ShareProductButton, UploadedProductDialog share URL) need
  env threaded; the other 6 use the relative form and are untouched. Tests in
  `utils.test.ts` assert the hardcoded apex output for the prefixed form and would need
  updating to the tier-aware shape.

---

## Item 2 — message-body trim divergence vs web (confirm-only)

- **Present?** **YES** (expo trims) — and **expo is the correct side; no expo change.**
- **As-built location:** trim at **`src/lib/store/useActiveChatStore.ts:452`**
  (`?.text?.trim() ?? ''`), feeding `pingMessageSent(resolvedChatId, messageText)` at
  `useActiveChatStore.ts:454`. The emitter is
  `pingMessageSent` → `POST /secure/notifications/message-sent { chatId, messageText }`
  at `src/notifications/service/messageNotificationService.ts:18-22`.
- **What you found:** Expo extracts the text block, trims it, and falls back to `''` for an
  image-only message (the backend supplies the localized "[sent a photo]" body). The
  surrounding comment (`useActiveChatStore.ts:441-448`) documents this as intentional. The
  trimmed value is only the push-banner body sent to the backend ping; it is not the stored
  message content.
- **Verdict:** **No expo change.** Trimming the push banner is the sensible behavior; the
  divergence is a web-side alignment item (web should trim too). Did not change anything.

---

## Item 3 — auth-listener null-path skips full store cleanup

- **Present?** **YES**
- **As-built location:** null-path branch at **`src/lib/store/authStore.ts:249-255`**.
- **What you found:** On `onIdTokenChanged(null)` the branch resets the three
  `listenerState` fields (`inFlightUid`, `lastSyncedUid`, `lastSyncedAt`) and calls
  `set({ user: null })` — nothing else. It does **not** clear any user-scoped store, so
  sign-outs reaching this branch (the axios 403/401 interceptor calling `auth.signOut()` at
  `api.ts:91,102,130`, and any foreground re-validation) leave stale chat/favorites/
  notification/filter data behind.
- **What `authStore.logout()` clears (`authStore.ts:175-231`), in order:**
  1. `set({ user: null })`
  2. `useViewTokenStore.getState().clear()`
  3. `useChatListStore.getState().clearChatList()`
  4. `useActiveChatStore.getState().clearActiveChat()`
  5. `useChatNavStore.getState().clearChatNav()`
  6. `clearUserCache()`
  7. `useFavoritesStore.getState().clear()`
  8. `useNotificationStore.getState().reset()`
  9. `usePortalFilterStore.getState().clearAllFilters()`
  10. `useDashboardFilterStore.getState().clearAllFilters()`
  11. `await logoutFirebase()`
  (each store call wrapped in its own try/catch + `logServiceError`).
- **Recommended fix shape:** Two options:
  - **(a)** Inline the cleanup (steps 2–10) into the listener null-path.
  - **(b)** Extract steps 2–10 into a shared `clearUserScopedStores()` helper called by both
    `logout()` and the listener null-path.
  **(b) is smaller and safer.** It removes the duplication risk (a future store added to
  `logout()` but forgotten in the listener) instead of creating it. Do **not** route the
  interceptor/listener through `logout()` itself: `logout()` also calls `logoutFirebase()`,
  which would re-trigger `onIdTokenChanged(null)` and recurse / double-fire. The helper must
  contain only the store-clearing steps, not the Firebase sign-out. Blast radius: one new
  helper + two call sites in `authStore.ts`; no external callers.

---

## Item 4 — two Firebase auth listeners on overlapping events

- **Present?** **YES** (both exist; benign today)
- **As-built location:**
  - Main: `initAuthListener` → `onIdTokenChanged` at `src/lib/store/authStore.ts:243-307`.
  - Favorites: `InitFavoritesStore` → `listenAuthState` (wraps `onAuthStateChanged`) at
    `src/components/init/InitFavoritesStore.ts:11-18`; wrapper defined at
    `src/lib/services/authService.ts:270-271`.
- **What you found:** On a non-null user the main listener runs the
  email-verification gate + dedup + `syncUserToBackend`; the favorites listener calls
  `useFavoritesStore.init()`. On null, main sets `user: null` (see Item 3) and favorites
  calls `clear()`. They fire on overlapping events but touch disjoint state, and the
  favorites listener does **not** call `syncUserToBackend` — so it is structurally untidy
  but not currently harmful. (`onIdTokenChanged` additionally fires on token refresh, which
  `onAuthStateChanged` does not — a real behavioral difference, not just duplication.)
- **Recommended consolidation shape:** Have favorites subscribe to `useAuthStore` after
  `_hasHydrated` (`useAuthStore.subscribe` on `user` transitions null↔non-null) instead of
  opening its own Firebase listener — favorites then keys off the same resolved auth state
  the rest of the app uses, and the app keeps exactly one Firebase listener. Folding `init()`
  directly into the main listener is the alternative but couples favorites' lifecycle into
  the sync path.
- **Risk now vs later:** **Leave it for now is the safer call.** It is benign pre-prod, and
  touching the favorites init path risks a load-order regression (favorites init currently
  runs independently of the sync dedup window). Bundle this with the Item 3 cleanup-helper
  work, or schedule it post-prod-stabilization — it is tidiness, not a live bug.

---

## Item 5 — circular module dependency in auth wiring

- **Present?** **NO — the reported cycle does not exist in current code.** The issue
  description is wrong against the as-built code (already refactored away).
- **Edge-by-edge trace:**
  - `api.ts → authStore`: **NOT PRESENT.** `src/lib/config/api.ts:1-4` imports only `axios`,
    `../client/firebaseClient` (`auth`), `../store/bootStore`, `../utils/isErrorWithCode`.
    No `authStore` / `authService` import anywhere in the file.
  - `authStore → authService`: present — `src/lib/store/authStore.ts:7-14`.
  - `authService → api.ts`: present — `src/lib/services/authService.ts:15`
    (`import { BACKEND_API } from '../config/api'`). `authService` does **not** import
    `authStore`.
- **What you found:** The chain is the linear `authStore → authService → api.ts`, not a
  cycle — `api.ts` is already the leaf. The decoupling the brief recommends is **already
  implemented** via dependency injection: `api.ts` exposes `configureAuthInterceptorHooks`
  (`api.ts:6-14`) and calls `auth.signOut()` directly for ban/401 paths rather than reaching
  into `authStore`. The bridge `src/lib/init/authInterceptors.ts:1-14` imports both
  `api` (the hook configurer) and `authStore`, wiring `onAccountRestored/onAccountBanned/
  onMaintenanceDetected` at app init — that is the seam that would otherwise have closed the
  loop.
- **Module-eval-time access?** None. The only cross-module reach from `api.ts` into store
  state is `useBootStore.getState()` inside the request interceptor callback (`api.ts:44`)
  and the injected `authHooks.*` callbacks — all runtime, none at evaluation time. No latent
  bug either, because the cycle itself is gone.
- **Recommended fix shape:** **No action.** File touch count: 0. Recommend closing this issue
  as already-resolved (the hooks/DI pattern in `authInterceptors.ts` is exactly the
  leaf-module extraction the issue asked for).

---

## Summary table

| Item | Present? | Live bug? | Smallest fix |
|---|---|---|---|
| 1 — product URL hardcoded host | YES | yes (preview/dev share links → prod) | shared `getWebBase(env)`, thread env into 2 outbound callers |
| 2 — message trim divergence | YES (expo trims) | no | none (web-side alignment) |
| 3 — listener null-path no cleanup | YES | yes (stale stores after interceptor sign-out) | extract `clearUserScopedStores()`, call from logout + null-path |
| 4 — dual Firebase listeners | YES | no | leave for now / subscribe-to-store later |
| 5 — circular dep | **NO** | no | none — already fixed via DI hooks; close issue |

## Notes on issue-description accuracy
- **Item 5's reported cycle is stale** — `api.ts` no longer imports `authStore`; the DI hooks
  pattern (`authInterceptors.ts`) already broke it. Recommend the tracker mark it resolved.
- **Item 1** line drift: hardcoded host is `utils.ts:156` (impl overload), not `:136`
  (`:136` is the first type-overload signature).
- **Item 3** line drift: null-path is `authStore.ts:249-255`.
