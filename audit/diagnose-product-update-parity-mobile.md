# Diagnostic — Expo Product Update Screen: Preview crash, images, layout, Delete (read-only)

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-06-01
**Feature slug:** product-update-parity
**Type:** Phase 2 diagnostic. READ-ONLY — no code changed, nothing staged.
**Task (verbatim):** Diagnose four remaining problems on the mobile product UPDATE screen (`app/owner/dashboard/products/[productId].tsx`) from the code — Preview crash, images not visible, layout vs web, missing Delete. Do not fix.

Cross-repo note: this audit cannot read `oglasino-backend`. Where a symptom hinges on the `GET /secure/products?productId=` response shape (the new `ProductForUpdateDTO`), I say so and flag it for the backend seam.

---

## Question 1 — The Preview crash (PRIORITY)

### Root cause (one line)
`PreviewProductDialog` derives `previewProductData` at the **top level of the component body** (runs on every render), and that object reads `ownerId: userInfo.id` (`PreviewProductDialog.tsx:107`) — but `userInfo` is local state that starts `undefined` and is only filled by an **async** `getUserForFirebaseUid` call inside a `useEffect`. On the **first render**, before that effect resolves, `userInfo` is `undefined`, so `userInfo.id` throws `TypeError: Cannot read property 'id' of undefined`. There is no guard for the not-yet-loaded `userInfo`.

### Trace

- **Where does the dialog get `userInfo`?** Not a prop. It is local state — `const [userInfo, setUserInfo] = useState<UserInfoDTO>();` (`PreviewProductDialog.tsx:40`), seeded `undefined`. It is populated asynchronously:
  ```tsx
  // PreviewProductDialog.tsx:52-56
  useEffect(() => {
    if (user) {
      getUserForFirebaseUid(user?.firebaseUid).then((res) => setUserInfo(res));
    }
  }, [user]);
  ```
  `getUserForFirebaseUid` (`userService.ts:93`) is an HTTP call returning `UserInfoDTO | null`. So `userInfo` is `undefined` until that round-trip resolves — i.e. on the first render and every render until the fetch returns.

- **What the update screen passes when opening Preview.** `handlePreview` (`[productId].tsx:281-286`) opens the dialog with **only** `{ productDetails }`:
  ```tsx
  openDialog(DialogId.PREVIEW_PRODUCT_DIALOG, { productDetails: productDetails });
  ```
  It does **not** pass `userInfo`, and is not expected to — the dialog self-fetches it. So this is **not** specific to how the update screen invokes Preview; it is an internal render-ordering bug in `PreviewProductDialog` that fires the moment the dialog mounts. (Preview is update-only, which is why it surfaces only here — but the defect is in the dialog, not in `[productId].tsx`.)

- **Why undefined.** It is read during render before the async fetch that fills it has resolved, with no loading guard. The object literal `previewProductData` (`:105-124`) and the helper `getProductDetailsData` (`:126-162`) both dereference `userInfo.id` (`:107`, `:129`) and run before `userInfo` exists.

- **Compare to what populates correctly.** The dialog *does* read the store user synchronously — `const user = useAuthStore((s) => s.user)` (`PreviewProductDialog.tsx:37`), same store the update screen uses for region/city. `user` is `AuthUserDTO` and **carries `id: number`** (`AuthUserDTO.ts:5`), available synchronously (the store is persisted/hydrated). So the dialog reads the right store for `user`, but then makes a **second**, async fetch for a richer `UserInfoDTO` and dereferences *that* (`userInfo.id`) before it loads. The wrong-source nuance: `ownerId` only needs the numeric id, which `user.id` already provides synchronously; `userInfo` is fetched for the *other* fields (display name, rating, etc. used by `owner: userInfo` at `:146`).

- **The `if (!user) return` guard is too late and guards the wrong value.** It sits at `:164`, **after** `previewProductData` is computed at `:105`, and it tests `user` (populated) not `userInfo` (the undefined one). It cannot prevent the crash.

### Minimal correct source for `userInfo` (not implemented)
Two viable minimal fixes:
1. **Gate the derivation on `userInfo` being loaded** — early-return a loader/null until both `user` and `userInfo` are present, and compute `previewProductData`/`getProductDetailsData` only after that (e.g. `if (!user || !userInfo) return …;` placed *above* line 105). This is the smallest change that addresses the actual crash and also protects line 129.
2. **Source `ownerId` from `user.id`** (the already-loaded `AuthUserDTO`, `:37`) instead of `userInfo.id`. This removes the dependency on the async fetch *for the listing-card path*, but the `details` path still uses `owner: userInfo` (`:146`) and `productDetailsData.owner` (`:206`), so option 1 (guard on `userInfo`) is the more complete minimal fix.

### Other unguarded derefs that would crash for an update-loaded product
Line 110 `productDetails.regionAndCity?.city?.labelKey` is `?.`-guarded (and `regionAndCity` is correctly absent on the update DTO — server omits it). The following are **unguarded** and relevant:

| Line | Expression | Why it crashes |
|------|-----------|----------------|
| `:107` | `ownerId: userInfo.id` (in `previewProductData`) | `userInfo` undefined on first render — **the reported crash**. |
| `:129` | `ownerId: userInfo.id` (in `getProductDetailsData`) | Same undefined `userInfo`; the effect at `:44-50` calls this and would also throw (the render crash at `:107` simply hits first). |
| `:202` | `productDetails.imageKeys.map((key) => publicImageUrl(key, 'hero'))` (details mode) | `imageKeys` is **optional** on the DTO (`NewProductRequestDTO.ts:18`, `imageKeys?`) and per Q2 likely arrives `undefined` on the update load → `.map` of undefined throws when the user switches to the "details" tab. Note this line reads `imageKeys` directly, **not** the hydrated `imagesData`, so it is doubly wrong: vulnerable *and* would show nothing even when `imageKeys` is empty. |
| `:206-207` | `productDetailsData.owner`, `productDetailsData.id`, `.name` (details mode) | `productDetailsData` is state, `undefined` until the async effect resolves; dereferenced unguarded in the details branch. Compounded by the fact that `getProductDetailsData` itself throws at `:129`, so `productDetailsData` may never get set at all. |

`selectedBaseSite` (`:38`, from `bootStore`) is only *assigned* (`:108`, `:130`), never dereferenced, so it does not crash here.

---

## Question 2 — Images still not visible on update

### Most likely cause
**(a) `imageKeys` is not arriving populated from the backend** on the new `ProductForUpdateDTO` returned by `GET /secure/products?productId=`. Hydration (b) and `{ key }` rendering (c) are both confirmed correct in the mobile code, which leaves (a) — and the fact that the prior `imagesData`-hydration fix (session `-2`) is fully in place yet on-device images still don't show is strong evidence the input array is simply empty/absent. **This must be confirmed against the backend `ProductForUpdateDTO`; flag as the likely cause if `imageKeys` was dropped in the DTO split.**

### Trace

- **Load path confirmed.** `getDashboardProductDetails(id)` (`productsSearchService.ts:36-38`) → `getProductDetails(id, 'dashboard')` → `GET /secure/products?productId=<id>` (`:13-30`) → returns `res.data` typed `UpdateProductRequestDTO` (`:36`). That type `extends NewProductRequestDTO` (`UpdateProductRequestDTO.ts:5`), which **still declares `imageKeys?: string[] | undefined`** (`NewProductRequestDTO.ts:18`). So the mobile read still *expects* `productDetails.imageKeys`. The field is **optional** — the type tolerates it being absent, so a backend that dropped it in the `ProductForUpdateDTO` split would compile-pass on mobile and silently yield no images.

- **Hydration confirmed running.** `hydrateImagesData` (`[productId].tsx:41-44`) maps `(data.imageKeys ?? []).map((key) => ({ key }))` into `imagesData`. It runs on load (`:139`, inside `fetchProduct`) and on post-save re-fetch (`:200`, `reseedFromServer`), and both `productDetails` and the deep-equal baseline `oldProductDetails` are hydrated from the same object. `visibleImage` is seeded from `hydrated.imagesData[0]` (`:143-145`). **If `imageKeys` is undefined/empty, `imagesData` becomes `[]` → nothing to show.** Hydration is correct; it is starved of input.

- **`{ key }` rendering confirmed sound.** `ImagesImport` now reads `images={productDetails.imagesData ?? []}` (`[productId].tsx:317`) — display and edit share one field (the session-`2` fix). A `{ key }` entry renders via `img.key ? publicImageUrl(img.key, 'card') : img.file?.uri` for the main image (`ImagesImport.tsx:127-129`) and thumbnails (`:172`). `publicImageUrl(key, 'card')` builds `${CDN}/cdn-cgi/image/width=400,height=300,…/${key}` (`variants.ts:42-47`). This is the **same** `{ key }` → CDN path the portal product page uses successfully, so the render path is not the problem.
  - One latent gotcha (not the current cause): `publicImageUrl` throws if `EXPO_PUBLIC_CDN_URL` is unset (`variants.ts:11-13`). Since portal images render for Igor, the CDN env is configured — so this is not it.
  - No silent-drop on a `{ key }` entry: `expo-image` with a valid CDN URI either renders or shows blank, but the array would still contain the entries (thumbnail strip would show placeholders/count). The reported symptom is *no images at all*, consistent with `imagesData === []`, i.e. `imageKeys` empty.

- **Conclusion.** (b) runs and (c) renders correctly; therefore the evidence points to **(a)** — `imageKeys` not populated on the new `ProductForUpdateDTO`. The mobile read is correct; the seam is the backend response. Resolve by confirming, in `oglasino-backend`, that `ProductForUpdateDTO` still serializes a populated `imageKeys` for `GET /secure/products?productId=`.

---

## Question 3 — Layout structure vs web

### Current state (one line)
`[productId].tsx` is a **flat `ScrollView`** with **no section/card containers**: every control sits in one `mt-2 gap-4 px-2` column, so the non-filter controls render full-width while the filter rows inside `MetaDataProduct` carry their own internal layout — there is no shared frame normalizing widths or grouping sections the way web's bordered cards do.

### Document

- **Structure (`[productId].tsx:294-470`).**
  - Outer `<ScrollView ref={scrollRef}>` (`:295`) — flat, no cards.
  - Header row `<View className="mt-2 flex-row items-center justify-between px-4">` (`:296-304`): product-id label + back button.
  - The whole form lives in **one** `<View className="mt-2 gap-4 px-2">` (`:314`) that stacks, in order: `imagesRef` View → `ImagesImport` (`:315-323`); name `Input` (`:324-334`); description `Textarea` (`:335-350`); region/city `<View>` (`:352-365`); price+currency row (`:367-406`); `categoryRef` `<View>` → `CategorySelector` (`:407-421`); `MetaDataProduct` (`:423-428`). **No `Kategorija proizvoda` / `Filteri proizvoda` card wrappers, no images block frame — one flat `gap-4` column.**
  - Near-Save error block `<View className="mt-4 gap-1 px-4">` (`:431-444`).
  - Action row `<View className="my-4 flex-row justify-center gap-2">` (`:445-467`).

- **Why field widths differ.**
  - The non-filter controls (name, description, region/city, the price+currency `w-full flex-row` row at `:368`, category) all render **directly in the `px-2` column → full bleed**.
  - The filter controls come from `MetaDataProduct`, which returns a **bare fragment** (`MetaDataProduct.tsx:160-180`) of rows, each `<View className="flex w-full flex-row items-center gap-2">` (`:166`) whose **first child is a leading icon column** `<View className="mt-2 self-start">` (`:167`). On the update screen `addCheckedFilter={false}` (`[productId].tsx:426`), so that leading column renders **empty but still occupies a flex slot**, pushing the actual filter selector (`getValidFilterSelector`, `:175`) inward. So the filter selectors are inset/narrower than the full-width controls above them, and nothing wraps either group in a common-width section. That is the width-mismatch the brief describes.

- **Web's target (from the brief).** Web groups the form into **distinct bordered cards** — a `Kategorija proizvoda` card, a `Filteri proizvoda` card, images as their own block, etc. (No web audit doc exists in `oglasino-docs/`; this is the brief's stated structure.)

- **Existing reusable section/card component? — No.** `src/components/ui/` has `button, checkbox, collapsible, icon, select, skeleton, text` — **no `Card` / `Section` / `Fieldset` primitive**. The `*Card*` files under `src/components/` are product/data cards (`PreviewProductCard`, `DashboardProductCard`, `UserCard`, …), not layout frames. The **create wizard** dialogs (`product-creation/AddUpdateProductDialog.tsx`, `ImageSelectionProductDialog.tsx`, `UploadedProductDialog.tsx`) use **inline** `border border-border rounded-…` Views as ad-hoc frames; there is no shared component to import. So a parity fix would either (i) introduce a small reusable section wrapper (one new component, justified by ≥2 sections here + reuse on create), or (ii) copy the create wizard's inline bordered-View pattern. Flag for fix-time scoping.

---

## Question 4 — Missing Delete button

### Current state
Mobile's update-screen action row has **no Delete**. The row (`[productId].tsx:445-467`) contains exactly three buttons: Cancel (`product.update.cancel`, outline, `:446-453`), Preview (`product.update.preview`, blue, `:454-456`), Save (`product.update.save`, green, spinner+disable on `saving`, `:457-466`). No Delete, and (correctly per the brief) no Activate/Deactivate.

### The existing Delete flow (reuse target)
Delete lives in the product-functions dialog, `DashboardProductFunctionsDialog.handleDelete` (`:90-114`):

- **Service:** `deleteProduct(productId)` (`productService.ts:168-176`) → `BACKEND_API.delete('/secure/products?productId=' + productId)`. This is a **hard delete** (HTTP `DELETE` on the products resource). Soft state changes go through the separate `deactivateProduct` → `GET /secure/products/deactivate?productId=` (`:158-166`), which is a different endpoint — so Delete here is the hard/permanent path.
- **Confirmation dialog (the established destructive pattern):** opens `DialogId.INFO_DIALOG` with:
  - title `tDialog('info.delete.product.alert.title')`
  - two-line description `info.delete.product.alert.description.1` + `.2`
  - continue button `tButtons('delete.product')`
  - `type: 'error'`, `freezeOnContinue: true`
  - `onContinue: async () => { const result = await deleteProduct(productId); … }` — on success a `tCommon('product.function.delete.notification')` success toast, else `tErrors('unknown')`, then `onRequestProductRefresh()`, returns `result`.
- **After delete:** in the functions dialog it does **not** navigate — it calls `onRequestProductRefresh()` to refresh the dashboard list in place. On the *update screen* there is no list to refresh and the edited product no longer exists, so a fix would instead need to `router.back()` (to the dashboard product list) after a successful delete.

### Notes for fix-time scoping
- Delete is permanent (hard `DELETE`); the `INFO_DIALOG` confirm + `freezeOnContinue` is the established pattern — **reuse it, don't invent a new one.** All four labels/keys already exist (`info.delete.product.alert.*`, `tButtons('delete.product')`, `product.function.delete.notification`), so no new translation keys are needed.
- The only delta vs the functions-dialog flow is post-delete navigation: update screen → `router.back()`; functions dialog → list refresh.

---

## Fix scoping notes (what the minimal fix would touch)

1. **Preview crash (Q1) — `src/components/dialog/dialogs/PreviewProductDialog.tsx` only.** Add a loading guard so `previewProductData`/`getProductDetailsData` don't run until `userInfo` is loaded (e.g. early-return above `:105` when `!user || !userInfo`), and guard the details-mode derefs (`productDetails.imageKeys?.map …` at `:202`, `productDetailsData?` at `:206`). Optionally source `ownerId` from `user.id` for the listing path. No change to `[productId].tsx` needed for the crash itself. Highest priority, smallest blast radius.
2. **Images (Q2) — primarily a backend seam, not a mobile fix.** Confirm `ProductForUpdateDTO` returns a populated `imageKeys`. If it does, no mobile change is needed (hydration + render are correct). If the field was intentionally renamed/restructured in the DTO split, the mobile fix is a one-line read change in `hydrateImagesData` (`[productId].tsx:41-44`) to map the new field. Also worth fixing regardless: `PreviewProductDialog.tsx:202` reads `imageKeys` directly instead of the hydrated `imagesData` (Q1 overlap).
3. **Layout (Q3) — `[productId].tsx` (+ optionally one new section component).** Wrap the images / category / filters groups in bordered section frames to mirror web; normalize control widths (notably the empty leading-icon column in `MetaDataProduct.tsx:167` when `addCheckedFilter={false}`). Decide: introduce a reusable `Section`/`Card` wrapper vs copy the create wizard's inline bordered-View pattern. Cosmetic/parity, no contract impact.
4. **Delete (Q4) — `[productId].tsx` action row.** Add a fourth (Delete) button that reuses `deleteProduct` + the `INFO_DIALOG` confirm pattern from `DashboardProductFunctionsDialog.tsx:90-114`, navigating `router.back()` on success. No new service, no new keys, no new confirm dialog.

---

## Session summary (conventions Part 5)

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-06-01
**Task:** Read-only Phase-2 diagnostic of four remaining problems on the mobile product UPDATE screen (Preview crash, images not visible, layout vs web, missing Delete).

### Implemented
- Nothing implemented — read-only diagnostic. Output is this document (Questions 1–4 + Fix scoping notes).

### Files touched
- None (read-only). Files **read:** `app/owner/dashboard/products/[productId].tsx`, `src/components/dialog/dialogs/PreviewProductDialog.tsx`, `src/components/dialog/dialogs/DashboardProductFunctionsDialog.tsx`, `src/components/ImagesImport.tsx`, `src/components/MetaDataProduct.tsx`, `src/lib/services/productsSearchService.ts`, `src/lib/services/productService.ts`, `src/lib/images/variants.ts`, `src/lib/store/authStore.ts`, `src/lib/types/product/{UpdateProductRequestDTO,NewProductRequestDTO}.ts`, `src/lib/types/user/{UserInfoDTO,AuthUserDTO}.ts`, and `src/components/ui/` listing.

### Tests
- Not run — no code changed.

### Cleanup performed
- None needed (read-only).

### Config-file impact
- conventions.md: no change.
- decisions.md: no change.
- state.md: no change required by a read-only diagnostic. The four findings feed the next fix brief; the open 2026-06-01 issues.md "Mobile on-device UI/UX findings (batch)" update-product items already cover the user-visible symptoms. No Expo backlog row flips.
- issues.md: no change (Docs/QA is sole writer). New code-level findings (the `PreviewProductDialog` render-ordering crash + its unguarded `imageKeys`/`productDetailsData` derefs; the `MetaDataProduct` empty-icon-column width inset) are surfaced in "For Mastermind" for triage if desired — not authored here.

### Obsoleted by this session
- Nothing.

### Conventions check
- Part 4 (cleanliness): N/A — no code changed.
- Part 4a (simplicity): N/A — no code added (see "For Mastermind" categories).
- Part 4b (adjacent observations): items flagged in "For Mastermind".
- Part 6 (translations): N/A this session — confirmed no new keys needed for any of the four fixes (Q4 reuses existing keys).
- Part 11 (trust boundaries): confirmed — no payload change examined beyond the prior audit; `deleteProduct` sends only the route `productId`; Preview is display-only.
- Other parts: Part 8 (routes reusable) — confirmed all paths (`GET /secure/products`, `DELETE /secure/products`, `deactivate`) are reused web endpoints; no mobile-specific route.

### Known gaps / TODOs
- The Q2 verdict depends on the backend `ProductForUpdateDTO` shape, which this repo cannot read. The mobile-side read/hydrate/render is confirmed correct; whether `imageKeys` arrives populated must be answered by a backend check before scoping the (possibly zero) mobile change.

### For Mastermind
- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only).
  - Considered and rejected: nothing (read-only).
  - Simplified or removed: nothing (read-only).
- **Highest-priority finding (Q1, high):** `PreviewProductDialog` crashes on open because `previewProductData` (`:105-124`) reads `userInfo.id` (`:107`) during the first render, before the async `getUserForFirebaseUid` effect (`:52-56`) fills `userInfo`. The `if (!user) return` guard (`:164`) is below the derivation and guards the wrong value. Minimal fix is local to the dialog (guard on `userInfo`); does not touch `[productId].tsx`. Same file also has unguarded `imageKeys.map` (`:202`, would crash the "details" tab) and `productDetailsData.*` derefs (`:206-207`).
- **Q2 seam (high, cross-repo):** confirm `ProductForUpdateDTO.imageKeys` is populated on `GET /secure/products?productId=`. Mobile hydrate (`[productId].tsx:41-44`) + render (`ImagesImport`/`publicImageUrl`) are correct; the prior session-`2` fix is in place, so empty images strongly implies the field isn't arriving. If confirmed dropped, the mobile fix is a one-line field remap.
- **Part 4b adjacent observations:**
  - `PreviewProductDialog.tsx:202` reads `productDetails.imageKeys` directly instead of the hydrated `imagesData` — file: `src/components/dialog/dialogs/PreviewProductDialog.tsx`. Severity: medium — crashes the details tab when `imageKeys` is undefined and shows no images even when empty. Out of scope to fix in a read-only pass.
  - `MetaDataProduct.tsx:167` renders an empty leading `self-start` icon column when `addCheckedFilter={false}` (the update screen's case, `[productId].tsx:426`), insetting/narrowing every filter row vs the full-width controls. File: `src/components/MetaDataProduct.tsx`. Severity: low (cosmetic / parity). Out of scope.
- **Config-file dependency (closure gate):** none required by a read-only diagnostic. No drafted config edits.
</content>
</invoke>
