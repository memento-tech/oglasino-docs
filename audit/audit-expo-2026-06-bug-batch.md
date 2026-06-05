# Audit — oglasino-expo 2026-06 bug-batch (READ ONLY)

**Repo:** oglasino-expo · **Branch:** new-expo-dev (not switched) · **Date:** 2026-06-01
**Mode:** read-only scoping audit. No code changes. Every file:line below was confirmed with `cat -n` / `grep -n` (not Read), per the brief's tool-reliability note. During this session `rg`/`grep` content output was observed mangling certain literal words (e.g. "Facebook", "placeholder" rendered as "n"); all asserted content was therefore re-confirmed via `cat -n`, whose output was accurate.

---

## 1. Facebook login button on the login screen

**Confirmed real.** The login-options UI renders **two** provider buttons.

- File: `src/components/dialog/dialogs/LoginOptionsDialog.tsx`
- Google button: lines **49–55** (`loginWithGoogle()`, `<GoogleIcon />`).
- **Facebook button: lines 57–63** — `onPress={() => useAuthStore.getState().loginWithFacebook()}`, `<FacebookIcon height={25} />`, label `tDialog('login.options.login.facebook.label')`.
- Import to remove: `LoginOptionsDialog.tsx:2` `import FacebookIcon from '@/components/icons/FacebookIcon';`

**Dead-code left behind if the button is removed (shell-confirmed callers):**
| Symbol | Location | Status after button removal |
| --- | --- | --- |
| `FacebookIcon` component | `src/components/icons/FacebookIcon.tsx` (def `:7`, export `:22`) | Only consumer is this button → file becomes fully dead, deletable. |
| `authStore.loginWithFacebook` | `src/lib/store/authStore.ts` — type `:62`, impl `:143` | Only caller is the button → dead. |
| `loginWithFacebookFirebase` | `src/lib/services/authService.ts:215` (stub `return null;` over a fully commented-out FB SDK block, `:216`–`:240`) | Called only by `authStore.ts:146` and the test mock → dead. |
| import in store | `authStore.ts:9` `loginWithFacebookFirebase,` | dead. |
| test mock | `src/lib/store/authStore.test.ts:34` `loginWithFacebookFirebase: vi.fn(),` | becomes stale. |
| translation key | `login.options.login.facebook.label` (backend-seeded) | no client change. |

**Fix size: small** (mechanical, multi-file, no logic). 
**Design fork:** *cosmetic hide* (delete only the button + `FacebookIcon` import) vs. *full teardown* (also delete `FacebookIcon.tsx`, the `loginWithFacebook` store action, the `loginWithFacebookFirebase` stub + its commented-out body, the test mock, and reconcile `DeleteAccountConfirmationDialog`'s now-fully-unreachable `'facebook.com'` provider branch — see Adjacent #1). Recommend full teardown so no orphaned FB scaffolding remains.

---

## 2. Cold start / reload lands on notifications instead of home

**Confirmed real — and NOT a tab-config / initial-route problem.**

- The portal tab navigator (`app/(portal)/_layout.tsx:26–31`) registers tabs in order: `(public)` = home (`:27`), favorites (`:28`), notifications (`:29`), messages (`:30`). There is **no `initialRouteName` / `unstable_settings` anywhere** in `app/` or `src/` (grep returned zero hits). So the navigator's natural initial route is the first screen, `(public)` = home.

- The notifications landing is an **active `router.push('/notifications')`** fired from the boot-mounted push handler, not the tab order:
  - `AppInit` (mounts on every `bootStatus === 'ready'`, `app/_layout.tsx:80`) renders `<PushNotificationsInit />` (`src/components/init/AppInit.tsx:28`).
  - `PushNotificationsInit` runs `checkInitialNotification()` on mount (`src/notifications/components/PushNotificationsInit.tsx:118`, invoked `:130`), which reads `Notifications.getLastNotificationResponse()` (`:119`) and feeds it to `handleNavigation`.
  - `handleNavigation`'s switch has a **catch-all `default:` → `router.push('/notifications')`** (`:99–100`), shared with `PRODUCT_EXPIRATION` / `PRODUCT_EXPIRED` (`:97–98`).

  `getLastNotificationResponse()` returns the **last** notification the user ever interacted with and is not cleared across launches. So on a plain cold start / Fast-Refresh reload (no fresh tap), if any prior response is still cached and its category isn't `MESSAGE`/`SAVED_PRODUCT`/`NAVIGATION`, the `default` branch force-navigates to `/notifications` — overriding home.

**Fix size: small–structural** (not a one-liner). The tab initial route is already home; forcing home does nothing. The fix lives in the notification boot redirect: (a) only navigate when the response is a genuine fresh tap (dedupe by tracking the handled response identifier so a stale `getLastNotificationResponse()` doesn't re-fire on every boot), and/or (b) stop the `default` branch blanket-pushing `/notifications` for unknown/expiration categories. 
**Design fork:** dedupe-the-last-response (cleanest) vs. drop/limit the `default` redirect vs. both.

---

## 3. Dashboard product list — activate/deactivate gives no immediate feedback

**Partially reproduced — the refresh callback IS wired; the gap is data freshness, not a missing handler.**

- **Toggle handlers:** `src/components/dialog/dialogs/DashboardProductFunctionsDialog.tsx` — `handleActivate` (`:50–68`) and `handleDeactivate` (`:70–88`). Both `await activateProduct/deactivateProduct(productId)`, toast, then call `onRequestProductRefresh()` (`:66`, `:86`) and `onClose()`.
- **Services return only `boolean`**, not the updated product: `src/lib/services/productService.ts:148` (`activateProduct → Promise<boolean>`) and `:158` (`deactivateProduct → Promise<boolean>`).
- **List state source:** `products` `useState` in `src/components/product/ProductList.tsx:52`.
- **`onRequestProductRefresh` is wired to a real refetch:** the card receives it from `ProductList.tsx:261` `onRequestProductRefresh={onRefreshCurrent}`. `onRefreshCurrent` (`:176–232`) re-fetches every already-loaded page (`fetchPage(page, false)` for `page 0..currentMaxPage`, `:199–200`), merges, and `setProducts(mergedProducts)` (`:215`). Path: `DashboardProductCard` (`src/components/dashboard/components/DashboardProductCard.tsx:9–24`) → dialog params → handlers.

So after a toggle a full backend refetch of the loaded pages already fires. If the row still shows the old state until a manual pull-to-refresh, the cause is **read-after-write lag on the dashboard-products list endpoint** — the immediate refetch returns the pre-toggle state. The code itself flags this: `ProductList.tsx:211` comment "*Use latest total count from first page (or last — depends on backend consistency)*".

**Can it be fixed client-only?** Yes — an **optimistic update** masks any backend lag instantly. A local single-row refetch is awkward: the only single-product owner-view fetch is **`getDashboardProductDetails(productId): Promise<UpdateProductRequestDTO>`** (`src/lib/services/productsSearchService.ts:36`), which returns `UpdateProductRequestDTO`, **not** the `ProductOverviewDTO`/`ProductDetailsDTO` the list row renders — so it would need shape adaptation. (`getPortalProductDetails`, `:32`, is the public-side equivalent.)

**Fix size: small** (optimistic flip of the toggled row's `productState` in `products`). 
**Design fork:** *optimistic update* (flip the row in `products` immediately, no extra fetch — instant, masks lag) **vs.** *rely on the existing `onRefreshCurrent` list refetch* (already wired; if it under-delivers, the issue is backend consistency, not client) **vs.** *local single-row refetch* (needs the `UpdateProductRequestDTO` → row-DTO adaptation above). Recommend optimistic update.

---

## 4. Catalog category page — search placeholder shows the full category path

**Confirmed real, and SCOPE is independent of the placeholder TEXT.**

- **Placeholder construction:** `src/components/SearchInput.tsx:110–124` (`const placeholder = useMemo(...)`). For ≥2 category levels it returns `tHeader('navigation.search.label.multi', { value: names.join('/') })` (`:123`), where `names` is the translated `topCategory/subCategory/finalCategory` labels (`:114–116`) joined by `/` — i.e. the `Top/Sub/Final` string that overflows. (`names.length === 1` → `...label.one`, `:121`; none → `...label.basic`, `:111`/`:120`.)
- **Search SCOPE is separate:** the query is `getAutocompleteSuggestions({ searchText, topCategoryId: categories?.topCategory?.id, subCategoryId: categories?.subCategory?.id, finalCategoryId: categories?.finalCategory?.id, ...preparedFilters })` (`SearchInput.tsx:89–94`). Scope is driven by the category **IDs** derived from the path (`categories` from `getCategoriesFromPath(...)`, `:66–68`) plus the filter store — **never** by the placeholder string. On submit it just `setSearchText(debouncedTerm)` and stays on the catalog pathname (`:206`, `:217–219`).

**Confirmed:** changing the placeholder TEXT (e.g. to show only the deepest/current category) does **not** change `topCategoryId`/`subCategoryId`/`finalCategoryId` or anything searched.

**Fix size: trivial** (alter only the `names`/branch selection in the `:110` memo to show the current/deepest category, or truncate). 
**Design fork:** show deepest-only vs. show top-only vs. CSS truncate/ellipsis the existing path.

---

## 5. ProductCard renders raw enum as state badge

**Confirmed real** at the reported lines.

- File: `src/components/product/ProductCard.tsx` (96 lines total).
- Badge block gated by `productState !== ProductState.ACTIVE || moderationState === ModerationState.BANNED` (`:79–80`).
- **`:84`** `<Text>{productOverview.productState}</Text>` — prints the literal `ProductState` enum value.
- **`:88`** `<Text>{productOverview.moderationState}</Text>` — prints the literal `ModerationState` enum value (only when `=== BANNED`, `:86`).
- Enums are plain string enums: `ProductState` = `ACTIVE | INACTIVE | DELETED` (`src/lib/types/product/ProductState.ts`); `ModerationState` = `APPROVED | BANNED` (`.../ModerationState.ts`). So the badge shows e.g. `INACTIVE`, `BANNED`.

**Translation keys:** **none exist for these states** — and there is no local mapping pattern to reuse. The only other place that surfaces these enums as user text, the dashboard product-state dropdown, **also renders the raw enum**: `src/components/dialog/dialogs/FiltersDialog.tsx:196–199` maps `[ProductState.ACTIVE, ProductState.INACTIVE]` to `{ value: ns, label: ns }` (raw). Translations are backend-seeded (no local JSON), so the implementer must introduce an enum→key mapping and ensure the keys (e.g. `product.state.*`, `moderation.state.*`) are seeded backend-side — there are **no existing keys to reuse** on the client.

**Fix size: small** (enum→translation-key lookup at both `:84`/`:88`; new backend-seeded keys needed). 
**Design fork:** whether to also fix the same raw-enum render in the dashboard dropdown (`FiltersDialog.tsx:198`) in the same pass — see Adjacent #2.

---

## 6. Preview Details tab reads `imageKeys` directly, not hydrated `imagesData`

**Confirmed real, and guarded (cannot crash).**

- File: `src/components/dialog/dialogs/PreviewProductDialog.tsx`.
- Details-mode carousel: **`:220`** `images={productDetails.imageKeys?.map((key) => publicImageUrl(key, 'hero')) ?? []}` — reads `imageKeys` directly. Guarded with `?.map(...) ?? []`, so a missing/undefined `imageKeys` yields `[]`, **no crash**.
- The listing-mode derivation in the same component reads the hydrated model instead: `topImageKey` is built from `productDetails.imagesData[0].key`/`.file.uri` (`:132–139`).
- **Why this is the wrong source during create-preview:** in the creation flow `imageKeys` is initialized `undefined` (`AddUpdateProductDialog.tsx:90`) and only `imagesData` (`ImageData[]` = `{key?, file?}`, `src/lib/types/ui/ImageData.ts`) holds the in-memory selected images (`onChange={(imagesData) => handleChange({ imagesData })}`, `AddUpdateProductDialog.tsx:130`). The edit flow even explicitly hydrates `imagesData` *from* `imageKeys` to converge everything on one array — `app/owner/dashboard/products/[productId].tsx:43` `imagesData: (data.imageKeys ?? []).map((key) => ({ key }))` with the convergence rationale at `:36–39`. So during create-preview the Details tab's `imageKeys` is empty while `imagesData` has the images → details carousel shows nothing.

**What re-pointing to `imagesData` touches:** only `:220`. It must map `ImageData → url` handling both branches, mirroring the existing `topImageKey` logic (`:132–139`): `key` → `publicImageUrl(key, 'hero')`, else local `file?.uri`. Keep the `?? []` guard. Nothing else references the details-mode image array.

**Fix size: small** (one-line expression swap + the two-branch map). 
**Note:** `app/(portal)/(public)/product/[...productData].tsx:220` reads `imageKeys` too but is **correct** there — that's the live product page where `imageKeys` is backend-hydrated. Only the create-preview path is affected.

---

## 7. Unused `getTranslation` helper in `PreviewProductDialog.tsx`

**Confirmed dead.**

- Definition: `src/components/dialog/dialogs/PreviewProductDialog.tsx:64–68` (`const getTranslation = (key, defaultValue) => { if (!key) return defaultValue; return t(key); }`).
- **Zero callers:** `grep -n "getTranslation"` in the file returns only the `:64` definition. Repo-wide, the only other matches are `getTranslationOrEmpty` in `app/owner/dashboard/user.tsx:216` — a different, separately-used function, not this one.
- Deletion touches nothing else: it uses `t` (still used elsewhere in the file) and takes no other dependency. Removing it leaves no dangling reference. (ESLint `no-unused-vars` would flag it.)

**Fix size: trivial** (delete `:64–68`). No fork.

---

## 8. `configurationService.tsx` return-type contract

**Confirmed — type-only; runtime already safe.**

- File: `src/lib/services/configurationService.tsx`.
- **`:6`** `export const getAppConfiguration = async (): Promise<ConfigMap>` — declares non-nullable `ConfigMap` (`= Record<string, string>`, `:4`).
- Returns `null` on the non-200 path (`:15`) and on the catch path (`:18`). So the declared type is violated; actual return is `ConfigMap | null`.
- **Runtime already tolerates null:** the sole production caller is `src/lib/store/bootStore.ts:203–204` — `const config = await withGateTimeout(() => getAppConfiguration()); set({ config: config ?? {} });`. The `?? {}` treats null/undefined as failure and substitutes an empty map (rationale comment `:198–201`; catch fallback `:206`). The test (`bootStore.test.ts:96`, `:613`) mocks the service.

**Fix size: trivial** (change the annotation to `Promise<ConfigMap | null>`; no runtime change). No fork.

---

## Adjacent observations (conventions Part 4b) — flagged, not fixed

1. **`DeleteAccountConfirmationDialog` Facebook branch** — `src/components/dialog/dialogs/DeleteAccountConfirmationDialog.tsx`: a `'facebook.com'` provider case plus comments (`:22`) explaining it's structurally unreachable because `loginWithFacebookFirebase` returns null / FB isn't wired. **Severity: low.** If item 1 fully tears down FB sign-in, this branch + comments become unambiguously dead and should be reconciled. *Out of scope.*
2. **Dashboard product-state dropdown renders raw enum** — `src/components/dialog/dialogs/FiltersDialog.tsx:198` `label: ns` shows `ACTIVE`/`INACTIVE` unlocalized, same class as item 5. **Severity: low.** Natural to fix in the same pass as item 5. *Out of scope.*
3. **Commented-out FB SDK block** — `src/lib/services/authService.ts:216–240` is a large commented-out implementation under the `loginWithFacebookFirebase` stub (conventions Part 4 forbids commented-out code). Pre-existing. **Severity: low.** Removed naturally by item 1 full teardown. *Out of scope.*
4. **`ProductReviewDialog.tsx:103` also sets `imageKeys: []`** — same DTO-image family as item 6; worth a glance when touching the preview image model to confirm it isn't the same wrong-source pattern. **Severity: low / informational.** *Out of scope.*

---

## Summary table

| # | Item | Verdict | Fix size |
| --- | --- | --- | --- |
| 1 | Facebook login button | Real — 2 provider buttons, FB at `LoginOptionsDialog.tsx:57–63` | small (teardown cascade) |
| 2 | Cold-start lands on notifications | Real — `PushNotificationsInit.tsx:99–100` `default → /notifications` via stale `getLastNotificationResponse()` (`:119`), not tab config | small–structural |
| 3 | Toggle no immediate feedback | Refetch wired (`onRefreshCurrent`); gap is backend read-after-write lag | small (optimistic) |
| 4 | Search placeholder = full path | Real — `SearchInput.tsx:123`; scope (IDs) independent of placeholder text | trivial |
| 5 | ProductCard raw enum badge | Real — `ProductCard.tsx:84`, `:88`; no existing keys (dropdown also raw) | small (+new keys) |
| 6 | Preview Details reads `imageKeys` | Real, guarded — `PreviewProductDialog.tsx:220`; should read `imagesData` | small |
| 7 | Unused `getTranslation` | Dead — `PreviewProductDialog.tsx:64–68`, zero callers | trivial |
| 8 | `configurationService` return type | Real, type-only — `:6` vs `null` at `:15`/`:18`; caller `?? {}`-safe | trivial |
