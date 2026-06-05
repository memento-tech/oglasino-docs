# Audit — Google Analytics v1 (mobile firing surfaces + `wasRegister` shape)

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-06-01
**Mode:** READ-ONLY. No code changed, no installs, no files created except this deliverable.
**Spec authority:** [`../oglasino-docs/features/google-analytics-v1.md`](../../oglasino-docs/features/google-analytics-v1.md) (web catalog is authoritative; mobile mirrors it).

This audit inventories where each of the 13 events must fire on mobile, the local data available to populate params, the `CallUserButton` prop question, the `wasRegister` wire shape, the dead `firebaseAnalytics.ts`, existing scaffolding, and PII surfaces. It designs nothing.

---

## Summary of headline findings

1. **`wasRegister` is present-on-wire-but-dropped-by-mobile.** Mobile hits the same `/auth/firebase-sync` route web uses (backend `AuthUserDTO` carries the field per docs/decisions 2026-05-23), but the mobile `AuthUserDTO` TS interface does not declare it and `syncUserToBackend` discards it. Zero backend work to discriminate `sign_up` vs `login`; the work is mobile-side type + plumbing only.
2. **Native analytics is greenfield.** `@react-native-firebase/analytics` is NOT installed; only the **web** `firebase` JS SDK (`firebase@^12.10.0`) is present. There is no `track` / `logEvent` wrapper of any kind.
3. **No error boundary and no global error handler exist** anywhere in the repo. `exception` is the one event whose firing surface does not exist yet on mobile and must be built (web had `app/error.tsx` + `app/global-error.tsx` + `window` listeners — none of which translate to RN).
4. **`CallUserButton` needs `productId` plumbed** from `ProductFunctions.tsx` (same as web). It has `userId` (= `seller_id`) but no product context.
5. **The auth firing surface is multi-site on mobile**, unlike web's single `UseTokenRefresh.tsx` hydrator — there are 4 post-sync `setUser` sites. This needs a deliberate chokepoint choice (see §1 row 2/3).
6. **`firebaseAnalytics.ts` is a dead web-SDK vestige with zero importers.** Flagged for deletion (not deleted here).

---

## 1. Firing surface per event

For each event: the mobile file + function where it must fire, and the local data available there. Param names below follow the spec's web catalog.

### 1. `page_view` — screen/route change

- **Router-level observable:** expo-router `usePathname()` (or `useSegments()`). There is **no existing route-listener component** (web's `GA4RouteListener` has no mobile counterpart).
- **Where it can be observed:** a listener component using `usePathname()` mounted **inside the navigation context**. The portal `<Stack>` is rendered in `app/_layout.tsx:102`, gated to `bootStatus === 'ready' | 'updating'` (`app/_layout.tsx:100`). A clean router-level home is a layout inside the navigator — e.g. `app/(portal)/_layout.tsx` (covers all portal screens) and/or `app/owner/dashboard/_layout.tsx` for dashboard — or a single `usePathname()` listener.
- **Local data:** `pathname` string (from `usePathname()`). Spec's web payload is `{ page_path: pathname }` — no PII, no numeric IDs needed.
- **Platform gotcha to flag:** `usePathname()` requires the expo-router navigation context. `AppInit` (`app/_layout.tsx:80`) is a **sibling rendered before** `<Stack>`; do not assume `usePathname()` resolves there. Mount the listener within the navigator subtree (a `_layout` under the Stack is safest). Because the Stack mounts only in `ready`/`updating`, `page_view` naturally won't fire pre-boot — consistent with the spec's "page_view may land without user_id" allowance.

### 2 & 3. `sign_up` / `login` — post-`syncUserToBackend`

- **File:** `src/lib/store/authStore.ts`.
- **Reality — multiple post-sync `setUser` sites (not web's single hydrator):**
  - `login()` → `loginUserFirebase` → `setUser` (`authStore.ts:96-97`)
  - `register()` → `registerUserFirebase` → `setUser` (`authStore.ts:112-114`)
  - `loginWithGoogle()` → `loginWithGoogleFirebase` → `setUser` (`authStore.ts:126-127`)
  - `initAuthListener()` `onIdTokenChanged` success branch → `setUser` (`authStore.ts:257-258`) — the convergence point, with a 2 s same-uid dedup (`SAME_UID_REPEAT_WINDOW_MS`, `authStore.ts:29`, `247-253`).
- **Closest mirror of web's "listener-is-sole-hydrator":** the `initAuthListener` success branch (`authStore.ts:257`). But note the explicit `login`/`register`/`loginWithGoogle` methods each run their own `syncUserToBackend` + `setUser` first and the listener then *dedups* — so firing **only** in the listener risks missing the explicit-method logins, and firing in **both** risks double-counting. This is a real design decision for the F brief, not something the code resolves today.
- **Discriminator:** `wasRegister` — **not currently threaded** (see §3). `method` is derivable from `firebaseUser.providerData?.[0]?.providerId` (already read in `authService.ts:146`): `password → email`, `google.com → google`, `facebook.com → facebook`. `user_id` = `backendUser.id` (numeric, `AuthUserDTO.id`).
- **Gotcha:** Facebook login is a no-op stub (`authService.ts:215-243` returns `null`); only `email` + `google` are reachable today. Banned sign-ins return via the `USER_BANNED`/`EMAIL_BANNED` catch (`authStore.ts:262-266`) and never reach `setUser` — so no spurious event, matching web.
- **Cold-start caveat to flag:** `onIdTokenChanged` fires on app relaunch with the persisted Firebase user, running `syncUserToBackend` again. A naive listener-based `login` fire would emit a spurious `login` on every cold start. The F brief must gate on a genuine-auth-event signal, not merely "listener produced a user."

### 4. `product_view` — product detail screen mount

- **File/function:** `app/(portal)/(public)/product/[...productData].tsx` → `ProductScreenContent`, after the load effect sets `productDetails` + `owner` (`product/[...productData].tsx:113-117`). No `ProductViewTracker` equivalent exists; an effect keyed on the loaded product is the surface.
- **Local data (all present):**
  - `product_id` = `productDetails.id` (numeric, `ProductOverviewDTO.id`).
  - `top_category_id` / `sub_category_id` / `final_category_id` = `productDetails.topCategoryId` / `subCategoryId` / `finalCategoryId` (numeric, `ProductDetailsDTO`).
  - `price` = `productDetails.price` — **typed `string`** in `ProductOverviewDTO`; needs `Number()` coercion to satisfy the spec's numeric `price`.
  - `currency` = `productDetails.currency` (string ISO).
  - `is_owner_view` = `owner?.iamActive` (`UserInfoDTO.iamActive` is true when the viewer owns the product). Available from the `owner` state set at `product/[...productData].tsx:117`.
- **PII:** none required; all numeric/enum.

### 5. `product_create_started` — create wizard first step mount

- **File/function:** `src/components/dialog/dialogs/product-creation/AddUpdateProductDialog.tsx`. First user-facing step is `currentStep === 0` → `ImageSelectionProductDialog` (`AddUpdateProductDialog.tsx:123-134`). A mount effect (fresh state) is the surface, mirroring web's `CreateNewProductDialog.tsx` mount effect.
- **Local data:** none needed beyond global `user_id` (funnel entry).
- **Ambiguity to confirm (flag):** the component is named **AddUpdate** and is shared-shaped, but as wired here it is the **create** flow — `UploadedProductDialog` hard-codes `mode: 'create'` (`UploadedProductDialog.tsx:99`). The edit/update entry point is elsewhere (`app/owner/dashboard/products/[productId].tsx`). The F brief must confirm this dialog instance is create-only (or add a mode discriminator) so `product_create_started` does not fire on edit.

### 6. `product_create_completed` — create success

- **File/function:** `src/components/dialog/dialogs/product-creation/UploadedProductDialog.tsx` → `uploadProductInternal`, success branch `setOutcome({ kind: 'success' })` (`UploadedProductDialog.tsx:123`), after `uploadProduct(...)` resolves (`:98`). Mirrors web's `UploadedProductDialog.tsx:62-65`. Re-entry guard already short-circuits after a prior success (`:86`), so it fires once.
- **Local data:**
  - `product_id` = `result.id` (persisted response, `:120`).
  - `top_category_id` / `sub_category_id` / `final_category_id` = `productData.topCategory?.id` / `subCategory?.id` / `finalCategory?.id` (`CategoryDTO.id`).
  - `price` = `productData.price` — **`string`** (`NewProductRequestDTO.price`); coerce to numeric.
  - `currency` = `productData.currency` — this is a **`CurrencyDTO` object**, not a string (`NewProductRequestDTO.currency: CurrencyDTO`); the F brief must read the ISO code off the object (e.g. its `.code`), not pass the object.
  - `image_count` = `productData.imageKeys?.length ?? productData.imagesData?.length`.
- **PII:** `result.name` / `productData.name` are present but not requested by the spec — do not include.

### 7. `contact_seller_clicked` — Call / Message / Favorite (all three call sites)

- **Call:** `src/components/product/CallUserButton.tsx` → `handlePress` (`:30`). Has `userId` (= `seller_id`, numeric). **Missing `product_id`** — see §2.
- **Message:** `src/components/product/StartMessageButton.tsx` → `handlePress` (`:24`). Has `product: ProductDetailsDTO` → `product.id` (`product_id`) and `withUser: UserInfoDTO` → `withUser.id` (`seller_id`, numeric). Both present.
- **Favorite:** `src/components/product/FavoriteButton.tsx` → `handlePress` (`:27`). Has `productData: ProductOverviewDTO` → `productData.id` (`product_id`) and `productData.ownerId` (`seller_id`, numeric). Both present.
- **Reuse note (flag):** `FavoriteButton` is **not product-detail-only** — it is rendered on product cards / lists too (separate from `ProductFavoriteButton.tsx`). So `contact_seller_clicked{method:'favorite'}` will fire from many surfaces, not just the detail screen. This is fine for the event semantics but worth knowing for QA expectations.
- **Params:** `method` ∈ `'call' | 'message' | 'favorite'`; `product_id`, `seller_id` numeric. **No seller name, no phone** (see §6).

### 8. `message_sent` — chat send commit resolve

- **File/function:** `src/lib/store/useActiveChatStore.ts` → `sendMessage`, after `await batch.commit()` resolves. Two commit paths: new chat (`useActiveChatStore.ts:385`) and existing chat (`:428`); both converge before `return chatRef.id` (`:445`). Mirrors web's `useChatStore.ts:sendMessage` post-`batch.commit()`.
- **Local data:**
  - `is_new_chat` = `isNewChat` (`:281`).
  - `receiver_id` = `receiver.id` (numeric `UserInfoDTO.id`) — **not** `receiver.firebaseUid` (the Firestore string key used throughout this store).
  - `product_id` = `tempProductContext?.id` (numeric), set only when the chat was opened from a product (`:379-381`, `:398-400`).
  - `block_count` = `content.blocks.length`; `has_text` = blocks include a `'text'` block; `has_images` = blocks include an `'images'` block.
- **PII (flag, see §6):** `content` carries the **message body text** (`TextMessageBlock.text`, surfaced via `getPreview`) and **image keys** (`ImagesMessageBlock.imageKeys`). Neither may go in the payload — only counts/booleans + the numeric `receiver.id`.

### 9. `search` — header search submit

- **File/function:** `src/components/SearchInput.tsx`. There is **no `onSubmitEditing`** on the `TextInput`; the commit is the "search for X" footer `Pressable` `onPress` (`SearchInput.tsx:205-221`) which calls `setSearchText(debouncedTerm)` then navigates. That is the `commitSearch` analog.
- **Local data:** `search_term` = `debouncedTerm` (`SearchInput.tsx:57`, trimmed at `:72`). Spec: truncate to 100 chars. Allowed value per spec.
- **Note:** tapping an autocomplete suggestion navigates straight to a product (`:150`) — that is a product navigation, not a search submit; do not fire `search` there.

### 10. `view_search_results` — catalog render with a search term present

- **File/function:** the catalog renders `FilteredProductList` (`app/(portal)/(public)/catalog/[...categories].tsx:59`). The results live in `src/components/product/FilteredProductList.tsx` (it owns `fetchPageInternal` and the fetched page).
- **Local data:**
  - `search_term` = `searchText` from the filter store (`FilteredProductList.tsx:43`, also `useFilterStore.searchText`); fire only when non-empty.
  - `results_count` = `totalNumberOfProducts` from the fetch response `ProductOverviewsDTO` (`src/lib/types/product/ProductOverviewsDTO.ts`) — returned by `fetchPage` → `getPortalProducts`.
- **Surface:** when the first page resolves with a non-empty `searchText`. No dedicated component exists; the F brief adds the fire inside `FilteredProductList` (or `ProductList`) on page-0 resolution.

### 11. `filter_change` — filter mutation on catalog/home

- **Mutation sites:** the filter-store setters in `src/lib/store/useFilterStore.ts` (`addRemoveOptionFilter`, `addRemoveRangeFilter`, `setOrder`, `setPriceRange`, `setRegionCityValues`, `clearAllFilters`), invoked from `src/components/dialog/dialogs/FiltersDialog.tsx` and its child filter components, plus `clearAllFilters` (`FiltersDialog.tsx:184`).
- **No URL-sync chokepoint exists** (web fired from `FilterManager.tsx`'s `router.replace` URL-sync effect; mobile keeps filters in Zustand, not the URL). The single observable convergence point is `FilteredProductList.tsx`'s `filtersData` memo (`:60-100`), which recomputes on any selection change and drives the refetch. A debounced effect keyed on `filtersData` (excluding `searchText`, which belongs to `search`) is the natural chokepoint — this is a design choice the F brief must make explicitly.
- **Local data:**
  - `filters_active` = categories of set filters (derivable from `selectedFilters` + price/order/region presence; cf. the new `src/lib/utils/getActiveFilterCount.ts`, which counts the same set). Categories only, not values.
  - `sort_order` = `selectedOrder` (`OrderTypeDTO`).
  - `pathname` = `usePathname()` (already used in `FiltersDialog.tsx:37` and the catalog screen).
- **PII:** none (categories + sort + pathname).

### 12. `exception` — error boundary + global handler

- **Reality: neither exists.** Repo-wide grep for `ErrorBoundary` / `componentDidCatch` / `ErrorUtils` / `setGlobalHandler` / `unhandledrejection` returns **nothing**. `app/+not-found.tsx` is a 404 route, not an error boundary.
- **What the F brief must build (mobile-specific, web's surfaces do not port):**
  - A React error boundary — expo-router supports exporting an `ErrorBoundary` from a route file (e.g. `app/_layout.tsx`); RN has no `app/error.tsx`/`app/global-error.tsx` split.
  - A global JS error hook via `ErrorUtils.setGlobalHandler(...)` (the RN analog of `window.onerror`), and unhandled-promise-rejection tracking (Hermes/RN rejection-tracking), as the catch-all. There is no `window` object.
- **Local data (per spec, once built):** `error_name` (`error.name`), `error_message` (`error.message`, truncate 200), `boundary` ∈ `'global' | 'route' | ...`. **No stack trace.**

### 13. `form_submit_failed` — form validation error surfaces

Two distinct error shapes on mobile:

- **Structured `{field, code, translationKey}` (backend, via `parseServiceError`) — fire with the conventions Part 7 code:**
  - **Product create / update:** `UploadedProductDialog.tsx` — the `validation` outcome carries `byField: Record<string, ServiceFieldError>` + `systemError` (`UploadedProductDialog.tsx:48-55`, set at `:143-144`). `ServiceFieldError` has `field`, `code`, `translationKey` (`src/lib/utils/parseServiceError.ts`). `error_code` = `code`; `field` = `field` (or `null`/`__system`). This is the clean structured surface.
  - **Profile/settings update** (`app/owner/dashboard/user.tsx`) and **product update** (`app/owner/dashboard/products/[productId].tsx`) go through the Φ4 surface-via-throw + `parseServiceError` contract, so they expose the same structured shape. (Not deep-read this session — flagged to confirm in the F brief, but they follow the Φ4 caller list.)
- **Client-side, field-keyed but message-valued (NOT a `{field, code}` shape) — synthetic codes needed:**
  - **Register:** `RegisterDialog.tsx` — `validateForm()` (`:55-78`) sets `errors: Record<keyof RegisterUserRequestDTO, string>` where values are **translated message strings** (`tValidation('email.bad')` etc.), keyed by field. Failure surface: `validateForm()` returns false in `registerInternal` (`:81`). Uses imperative `validateEmail`/`validatePassword` helpers — **no Zod** on mobile.
  - **Login:** `LoginDialog.tsx` — same pattern (`validateForm()` `:52-76`, fails in `loginInternal` `:79`); plus a single `loginError` string from `authStore.error` for backend/Firebase failures.
- **Platform difference to flag:** the spec's web plan derives `form_submit_failed.error_code` for register/login from the **Zod failure path**. Mobile has no Zod here — it uses imperative `validateForm` branches. The F brief must define the synthetic code set from those branches (e.g. `EMAIL_EMPTY`, `EMAIL_FORMAT`, `PASSWORD_EMPTY`, `PASSWORD_TOO_SHORT`, `DISPLAY_NAME_EMPTY`) rather than reusing a Zod-path mapping.
- **Backend auth failures** (`authStore.login`/`register`/`loginWithGoogle`) collapse to a **single string** via `mapAuthError` (`authStore.ts:100`, `117`, `129`) — not a `{field, code}` shape. If those need a `form_submit_failed`, the code would have to come from `mapAuthError`'s classification, not a field shape.

---

## 2. `CallUserButton` prop plumbing

- **Confirmed: `CallUserButton` does NOT have `productId`.** Its props are `{ userId, callingAllowed, disabled }` (`CallUserButton.tsx:12-20`). `userId` serves `seller_id`; there is no product context.
- **Parent call site:** `src/components/product/ProductFunctions.tsx:33-37`, which has `productDetails: ProductDetailsDTO` (→ `productDetails.id`) in scope. So `product_id` must be plumbed `ProductFunctions` → `CallUserButton` as a new prop — **exactly the plumbing web needed** (web plumbed it from `ProductFunctions.tsx` too).
- `ProductFunctions` is the single call site of `CallUserButton` (grep confirms no other importer), so the plumbing is one-hop and unambiguous.

---

## 3. `wasRegister` wire shape

**Verdict: present-on-wire-but-dropped-by-mobile.**

- **Type:** `src/lib/types/user/AuthUserDTO.ts` declares no `wasRegister` field (id, firebaseUid, displayName, email, baseSite, regionAndCity, profileImageKey, providerId?, allow*, allowPhoneCalling, disabled, banReason, deletionStatus, scheduledDeletionAt).
- **Service:** `syncUserToBackend` (`src/lib/services/authService.ts:143-152`) types the response as `AuthUserDTO`, does `const userData: AuthUserDTO = result.data; return { ...userData, firebaseUid }`, and returns `Promise<AuthUserDTO>`. It does **not** read, type, or surface `wasRegister`. (At runtime the spread happens to copy any extra wire field, but it is untyped and no caller reads it — so it is dropped for all practical purposes.)
- **Caller:** `authStore.ts` (`login`/`register`/`loginWithGoogle`/`initAuthListener`) consumes only the `AuthUserDTO` and calls `setUser` — nothing reads `wasRegister`.
- **Backend side:** mobile reuses web's `/auth/firebase-sync` route (POST in `authService.ts:144`). Per docs (`decisions.md` 2026-05-23 GA4 entry), the backend `AuthUserDTO` carries `wasRegister: boolean` on the wire (the error envelope does not). So the bit is on the response mobile already receives.
- **Consequence:** discriminating `sign_up` vs `login` mobile-side needs **zero backend work**. The F brief must (a) add `wasRegister: boolean` to the mobile `AuthUserDTO` type, and (b) surface it from `syncUserToBackend` to the chosen firing chokepoint (see §1 rows 2/3 — the multi-`setUser` decision).

---

## 4. Dead `firebaseAnalytics.ts`

- **Exists:** `src/lib/client/firebaseAnalytics.ts` (17 lines).
- **Web-SDK vestige:** imports `getAnalytics` from `firebase/analytics` (the web JS SDK) and guards on `typeof window !== 'undefined'` (`firebaseAnalytics.ts:6-7`) — `window` never exists in RN, so `getFirebaseAnalytics()` would always return `null` on device. It is web-shaped dead code.
- **Importers: zero.** Repo-wide grep for `firebaseAnalytics` / `getFirebaseAnalytics` matches only the file's own definition; nothing imports it.
- **Flagged for deletion** (not deleted in this read-only audit). Consistent with the consent-mode-mobile decision (`decisions.md` 2026-05-30): "F replaces it with a native init." The F brief should delete it.

## 5. Existing analytics scaffolding

- **`@react-native-firebase/analytics`: NOT a dependency.** `package.json` carries only `firebase@^12.10.0` (the web JS SDK, used for Auth + Firestore). No `expo-firebase-analytics`, no `@react-native-firebase/*`.
- **`track` / `logEvent` wrapper: none.** Greenfield — there is no analytics dispatch helper of any kind.
- **What *does* exist (the consent foundation, ready for F):** `src/lib/consent/analyticsGate.ts` → `isAnalyticsConsentGranted()` (synchronous, default-deny, reads `useConsentStore`). It has **no live caller by design** — chat F is the intended consumer (`decisions.md` 2026-05-30/31). Its only references are its own test (`analyticsGate.test.ts`) and doc comments in `useConsentStore.ts` / `ConsentInit.tsx`.
- **ATT: not present.** No `expo-tracking-transparency` / `requestTrackingPermissions` anywhere. The F brief owns the iOS ATT prompt and gates init on `(ATT-allows-on-iOS, or Android) AND isAnalyticsConsentGranted()` per the consent spec.

## 6. PII check

Surfaces where the obvious local data includes name / email / phone / message body / image URL, with the numeric-ID alternative:

| Surface | PII present locally | Use instead |
| --- | --- | --- |
| `CallUserButton.tsx` (`contact_seller_clicked`) | **phone number** via `getUserPhoneNumber(userId)` (`:51`) | `seller_id` = `userId` (numeric); no phone in payload |
| `StartMessageButton` / `FavoriteButton` / `ProductFunctions` (`contact_seller_clicked`) | seller `displayName` available on `UserInfoDTO` | `seller_id` = `withUser.id` / `productData.ownerId` (numeric) |
| `useActiveChatStore.sendMessage` (`message_sent`) | **message body text** (`TextMessageBlock.text`, `getPreview`), **image keys** (`ImagesMessageBlock.imageKeys`), receiver `displayName` | `block_count`, `has_text`, `has_images`, `is_new_chat` (booleans/counts); `receiver_id` = `receiver.id` (numeric) |
| `product_view` / `product_create_completed` | product `name` on the DTO | `product_id` (numeric) + category IDs; omit name |
| `sign_up` / `login` | `email`, `displayName` on `AuthUserDTO` | `user_id` = `id` (numeric); `method` enum only |
| `search` / `view_search_results` | `search_term` is the user's typed text — spec **explicitly allows** it (truncate 100) | n/a (allowed by spec) |
| `exception` (once built) | stack traces / URL fragments may carry PII | `error_name` + `error_message` (200-char cap), **no stack** |

All required numeric IDs are available at every surface (verified against `AuthUserDTO`, `UserInfoDTO`, `ProductOverviewDTO`/`ProductDetailsDTO`, `NewProductRequestDTO`, `ProductOverviewsDTO`). The only type coercions needed: `price` (string → number) at `product_view` and `product_create_completed`, and `currency` read off `CurrencyDTO` (object) at `product_create_completed`.

---

## Open questions for the F brief (not decided here)

1. **Auth chokepoint:** which of the 4 `setUser` sites fires `sign_up`/`login`, and how cold-start `onIdTokenChanged` re-syncs are excluded (§1 rows 2/3).
2. **`AddUpdateProductDialog` create-vs-edit:** confirm this dialog is create-only (or add a mode flag) so `product_create_started` doesn't fire on edit (§1 row 5).
3. **`filter_change` chokepoint:** there is no URL-sync equivalent; pick the `FilteredProductList.filtersData` effect vs a FiltersDialog apply point (§1 row 11).
4. **Register/login synthetic codes:** derive from imperative `validateForm` branches, not a Zod path (§1 row 13).
5. **Profile/product-update form shapes:** confirm `app/owner/dashboard/user.tsx` and `products/[productId].tsx` expose the structured `parseServiceError` shape (§1 row 13) — flagged, not deep-read.
