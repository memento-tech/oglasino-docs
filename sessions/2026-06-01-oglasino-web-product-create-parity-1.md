# Audit — Web Product Creation Dialog (READ-ONLY reference contract)

**Repo:** oglasino-web
**Branch:** dev (not switched)
**Date:** 2026-06-01
**Feature slug:** product-create-parity
**Task:** Inventory the entire product CREATE flow on web (the new-product wizard, not the update/edit page) so it can serve as the reference contract for the mobile parity fix.

All claims below are read from the code on disk today. I did not trust comments, prior audits, or the brief's assumptions. The create wizard is `CreateNewProductDialog` and its four step components. I deliberately did **not** audit the owner edit page (`updateProductData` / `ProductEditState` / `app/[locale]/owner/products/[productId]`).

---

## 1. Entry point and component tree

### Who opens it, from where

Two buttons can start the flow; only one reaches the create dialog directly:

- **`AuthAddNewProductButton`** (`src/components/client/buttons/AuthAddNewProductButton.tsx`) — the logged-in entry point. Rendered in the header (`src/components/client/HeaderNavButtons.tsx:35`), the mobile footer (`src/components/server/layout/MobileFooterNavigation.tsx:28`), the secure top nav (`src/components/client/secure/TopNavigation.tsx:31`, gated on `allowProductCreation`), and the owner products page (`app/[locale]/owner/products/page.tsx:50`, `useRegularButton`).
  - `openNewProductDialog` (`AuthAddNewProductButton.tsx:23-35`) is the gate:
    - `if (!user) return;` (no-op when logged out).
    - **If `!user.baseSite || !user.regionAndCity`** → opens `USER_BASIC_DATA_SELECTOR_DIALOG` with `shouldOpenDialog: true` (`:27-32`) — the user must set base-site + region/city first.
    - Otherwise → `openDialog(DialogId.CREATE_NEW_PRODUCT_DIALOG)` (`:33`).
- **`AddNewProductButton`** (`src/components/client/buttons/AddNewProductButton.tsx`) — the logged-**out** variant. It opens `LOGIN_OPTIONS_DIALOG` (`:18-23`), not the create dialog. It never reaches create directly.

The pre-entry gate dialog **`UserBasicDataSelectorDialog`** (`src/components/popups/dialogs/UserBasicDataSelectorDialog.tsx`) collects base-site + region/city, calls `assignUserLocation` + `refreshUser`, and — because `shouldOpenDialog` is true — chains into `openDialog(DialogId.CREATE_NEW_PRODUCT_DIALOG)` on success (`UserBasicDataSelectorDialog.tsx:87-89`).

### Dialog dispatch

`DialogId.CREATE_NEW_PRODUCT_DIALOG = 'createNewProductDialog'` (`src/components/popups/dialogRegistry.ts:6`). The `DialogManager` maps that string → the component: `createNewProductDialog: CreateNewProductDialog` (`src/components/popups/DialogManager.tsx:42`, imported `:9`).

### Component tree (mount order)

```
AuthAddNewProductButton            src/components/client/buttons/AuthAddNewProductButton.tsx
  └─ (gate) UserBasicDataSelectorDialog  src/components/popups/dialogs/UserBasicDataSelectorDialog.tsx   (only if baseSite/regionAndCity missing)
  └─ CreateNewProductDialog        src/components/popups/dialogs/CreateNewProductDialog.tsx
       └─ DrawerDialog (wrapper)   src/components/popups/dialogs/DrawerDialog.tsx
            ├─ Progress bar         (CreateNewProductDialog.tsx:215-219, hidden on step index 3)
            ├─ ComponentScrollToTop (CreateNewProductDialog.tsx:222)
            └─ steps[currentStep].component   (one of:)
                 ├─ [0] ImageSelectionProductDialog   src/components/popups/components/ImageSelectionProductDialog.tsx
                 │        └─ ImagesImport              src/components/client/ImagesImport.tsx
                 ├─ [1] BasicInfoProductDialog         src/components/popups/components/BasicInfoProductDialog.tsx
                 │        ├─ CategorySelector          src/components/client/CategorySelector.tsx
                 │        ├─ Input (name) + AISuggestionIcon
                 │        ├─ Textarea (description)
                 │        ├─ Input (price) + Select (currency)
                 │        └─ CitySelector (disabled)   src/components/owner/client/CitySelector.tsx
                 ├─ [2] MetaDataProductDialog          src/components/popups/components/MetaDataProductDialog.tsx
                 │        ├─ MetaDataProduct           src/components/owner/product/MetaDataProduct.tsx
                 │        └─ ReCaptchaWrapper          src/components/recaptcha/ReCaptchaWrapper.tsx
                 └─ [3] UploadedProductDialog          src/components/popups/components/UploadedProductDialog.tsx
                          └─ createNewProduct()        src/lib/service/reactCalls/productService.ts
```

The `steps` array is defined at `CreateNewProductDialog.tsx:142-201`. `currentStep` is **0-indexed** state (`:40`); each step object carries a 1-indexed `no` (1–4) used only for the progress bar and the step-mapping contract.

---

## 2. Steps, in order

`currentStep` 0-indexed; `no` 1-indexed. The dialog renders exactly one step at a time (`CreateNewProductDialog.tsx:223`).

### Step index 0 (`no: 1`) — Images — `ImageSelectionProductDialog.tsx`

- **Fields:** image picker via `ImagesImport` (`ImageSelectionProductDialog.tsx:65-71`), `maxNumberOfImages={5}`. A suggestion line + advice text. No text inputs.
- **Validation before Next:** client-side only, `validateImagesData(productData.imagesData ?? [])` (`:43`, defined `src/lib/validators/productValidator.ts:113-145`). Checks per-file MIME (`image/jpeg|png|webp`), size ≤ 5 MB, and SHA-256 duplicate detection. Error keys: `product.image.invalid_type`, `product.image.too_big`, `product.image.duplicate`. No server call here.
- **Skippable:** **YES, effectively.** `validateImagesData` returns `''` (no error) for an empty array — the `if (productImages.length > 0)` guard (`productValidator.ts:118`) is never entered, so zero images passes. The user can advance step 1 with no images selected. (See Drift §A.)

### Step index 1 (`no: 2`) — Basic info — `BasicInfoProductDialog.tsx`

Fields:

| Field | Component | Input type | Required |
| --- | --- | --- | --- |
| Category (top/sub/final) | `CategorySelector` (`:177-186`) | cascading dropdowns | required (`required={true}`) |
| Product name | `Input` (`:200-207`) | text, `maxNumOfChars={80}` | required; has AI-suggest button when non-empty (`:208-216`) |
| Description | `Textarea` (`:222-231`) | textarea, `maxNumOfChars={2000}` | required |
| Price | `Input` (`:238-245`) | numeric (`isNumericOnly`) | required **only if** `topCategory && !topCategory.freeZone` (`:236`) — free-zone categories hide the price+currency block entirely |
| Currency | `Select` (`:246-269`) | dropdown over `selectedBaseSite.allowedCurrencies` | shown with price; pre-defaulted (see §5) |
| Region/City | `CitySelector` (`:272-279`) | **display only — `disabled={true}`, `onChange={() => {}}`** | not user-editable here (see §4) |

- **Validation before Next** (`onNextInternal`, `:79-151`): **both** client Zod and a server pre-validate call.
  1. Trims name/description (`:82-83`).
  2. Client Zod via `validateProduct(...)` (`:85-94`, defined `productValidator.ts:13-92`). `validateImages` is passed **`false`** (`:93`) — so images are NOT re-checked here. Checks: name/description required/too_short/too_long (`createProductSchema`), category required (collapsed to first missing in top→sub→final order), price required when not free-zone. Error keys in the `ERRORS` namespace as `product.<field>.*` / `product.category.required` / `product.price.required`.
  3. If any of name/description/images/top/sub/final/price has a Zod error → `setProductErrors`, stop (`:96-107`).
  4. Otherwise server **pre-validate**: `preValidateProductBasics(name, description)` (`:111`, → `POST /secure/products/pre-validate`, `productService.ts:281`). `clean` → advance; `validation` → show per-field errors + fire `form_submit_failed` analytics + handle `__system`/429 cooldown; transport/5xx → warning toast + advance anyway (`:144-150`, pre-validate is a UX optimization, not the gate).
- **Skippable:** No — Zod + pre-validate must pass (or pre-validate must hard-fail transport-wise) to advance.

### Step index 2 (`no: 3`) — Metadata / filters — `MetaDataProductDialog.tsx`

- **Fields:** `MetaDataProduct` (`:47-51`) renders the catalog filters for the selected base-site + chosen categories (`MetaDataProduct.tsx:43-58`; the `age` filter is filtered out, `:57`). Filter selectors: RANGE, DATE, MULTI_OPTION, SINGLE_OPTION (`MetaDataProduct.tsx:118-167`). Plus an invisible `ReCaptchaWrapper` (`MetaDataProductDialog.tsx:52`).
- **Validation before Next** (`onNextStepInternal`, `:30-40`): runs reCAPTCHA `recaptchaRef.current.execute()`. If not verified → shows `tValidation('suspicion')`; if verified → advance. **No field validation** — filters are not required.
- **Skippable:** filter selection is optional; the step itself requires the reCAPTCHA to resolve verified. Note: if `recaptchaRef.current` is null, `onNextStepInternal` does nothing (cannot advance) — see Drift §D.

### Step index 3 (`no: 4`) — Upload / result — `UploadedProductDialog.tsx`

- **Fields:** none — terminal step. On mount (guarded by `triggeredRef`, `:46-47`) it auto-runs `createNewProduct(productData, {onProgress})` (`:54`). Shows loading, then a success view (product URL + copy + view/close) or a failure view.
- **Validation:** the real server gate — the create POST. No client-advance from here.
- **Skippable:** N/A (terminal).

---

## 3. The dialog title across steps

Yes, the dialog renders a header title via `DrawerDialog`'s `dialogTitle` prop (desktop: `DialogTitle`, `DrawerDialog.tsx:83`; mobile drawer: `DrawerTitle`, `:119`).

The title is **conditional on the step**. `CreateNewProductDialog.tsx:210`:

```tsx
dialogTitle={currentStep === 3 ? '' : tDialog('new.product.title')}
```

So `new.product.title` is shown on steps 0, 1, 2 (Images, Basic info, Metadata) and is **blank on step index 3** (the Upload/result step). Same title text on all three non-final steps. The progress bar is gated by the same condition (`currentStep !== 3`, `:215`) — hidden only on the final step.

---

## 4. Region and city

**The create flow does NOT ask the user to pick region/city. It defaults from the user's saved profile and is not editable in the wizard.**

- The pre-entry gate (`AuthAddNewProductButton.tsx:27-32`) refuses to open the create dialog unless `user.regionAndCity` (and `user.baseSite`) already exist, diverting to `UserBasicDataSelectorDialog` to set them first. So by the time the wizard opens, region/city is guaranteed present on the user.
- Inside the wizard, step 2 renders `CitySelector` purely as a **disabled display** of the saved value (`BasicInfoProductDialog.tsx:272-279`):
  ```tsx
  <CitySelector
    selectedRegionAndCity={user.regionAndCity}   // from useAuthStore()
    onChange={() => {}}                            // no-op
    showOnSelector={false}
    required={true}
    disabled={true}
    regions={selectedBaseSite.regions}
  />
  ```
- **Default source:** `useAuthStore().user.regionAndCity` (`BasicInfoProductDialog.tsx:65` destructures `user`; the field is `RegionAndCityDTO`). `CitySelector` short-circuits any selection while disabled (`CitySelector.tsx` `handleSelect`: `if (disabled) return;`).
- **Pickable?** No — disabled in the wizard. It is only pickable in the separate `UserBasicDataSelectorDialog` (profile/location setup), not in the create flow.
- **Sent in the payload?** **NO.** `regionAndCity` is not a field of `NewProductRequestDTO` and is not added to the request payload (see §5). The server derives the product's location from the authenticated user — consistent with conventions Part 11 (trust boundary). This is a correct trust posture, not a drift.

---

## 5. The create request payload (trust boundary — Part 11)

- **Endpoint / method:** `POST /secure/products/create` (`productService.ts:175`, `BACKEND_API.post('/secure/products/create', requestPayload)`).
- **Builder:** `createNewProduct(productData, uploadOptions)` in `src/lib/service/reactCalls/productService.ts:155-201`. Images are resolved to keys first by `extractAndUploadImages` (`:88-148`), then the payload is built as a **new object** (`:165-169`):
  ```ts
  const requestPayload: NewProductRequestDTO = {
    ...productData,
    imageKeys: allKeys,
    imagesData: undefined,
  };
  ```
- **Wire shape** (`NewProductRequestDTO`, `src/lib/types/product/NewProductRequestDTO.ts`), field by field:

| Field | Origin | Notes |
| --- | --- | --- |
| `name` | **(a) user-entered** | step 2; trimmed (`BasicInfoProductDialog.tsx:82`) |
| `description` | **(a) user-entered** | step 2; trimmed; optional AI-suggest (`:153-163`) |
| `price` | **(a) user-entered** | step 2; sent as string; only collected when category not free-zone |
| `currency` | **(a) user-entered, pre-defaulted** | default `selectedBaseSite.allowedCurrencies[0]` (`CreateNewProductDialog.tsx:116`); full `CurrencyDTO` object sent |
| `topCategory` / `subCategory` / `finalCategory` | **(a) user-entered** | step 2 `CategorySelector`; **full `CategoryDTO` objects** sent, not just IDs |
| `filters` | **(a) user-entered, pre-seeded** | step 3; initialized with defaults for delivery/condition/availability in `getInitialFilters` (`CreateNewProductDialog.tsx:48-86`); full `SelectedFilterDTO[]` sent |
| `imageKeys` | **(b) derived client-side** | R2 keys produced by uploading `imagesData[].file` in `extractAndUploadImages`; positional array (`productService.ts:165`) |
| `imagesData` | **NOT sent** | explicitly set to `undefined` in the payload (`:168`); axios drops undefined keys from the JSON body |
| region / city | **(c) NOT sent — server-derived** | not a DTO field; server reads it from the authenticated user (see §4) |

Trust-boundary read (Part 11): the only client-supplied values that cross are content fields (name/description/price/currency/categories/filters) plus `imageKeys`. Location is server-derived. No client-supplied "previous"/"old" value is sent for change detection (this is create, not update). Server remains the moderation gate; client Zod is structural only (`productValidator.ts:6-7`).

---

## 6. Success and failure handling

All in `UploadedProductDialog.tsx`.

### Success
- `result.type === 'success'` (`:64-83`): fires `track('product_create_completed', {...})` with product id, category ids, price, currency, image_count; computes `productUrl` via `getNormalizedProductUrl(id, name, true, locale)` (`:81`); sets `uploadSuccess = true`; calls `onFinish?.()`.
- Success view (`:152-188`): success title with `productName`, the product URL with a copy-to-clipboard button (`handleCopy`, `:120-126`), a "view" button that `window.open`s the URL, and a Close button. **Close behavior:** `onClose()`, and **if `onFinish` was NOT provided, `window.location.reload()`** (`:178-183`) — i.e. a hard reload to refresh the dashboard when no finish callback was wired.

### Per-field server errors
- Parsed by **`parseProductValidationErrors`** (`src/lib/utils/parseProductValidationErrors.ts:36-56`), via `parseProductErrorsForStatus` in the service (`productService.ts:60-69`). Produces `{ byField, list }`; first error per field wins; `field: null` entries collapse under `SYSTEM_ERROR_KEY = '__system'` (`parseProductValidationErrors.ts:5`). UI renders `translationKey` through `tErrors(...)`.
- On `result.type === 'validation'` (`UploadedProductDialog.tsx:84-93`): `setValidationErrors(byField)`, fires `form_submit_failed` per error (`form_name: 'product_create_step_4'`), `uploadSuccess = false`.
- Failure view (`:189-218`): lists each per-field error key (`errorEntries`, excludes `__system`) plus the `__system` line if present; a **"go back and fix"** button calls `handleGoBackAndFix` → `onJumpToStep(resolveTargetStep(validationErrors))` (`:134-140`). `resolveTargetStep`/`stepForField` (`src/lib/utils/productStepMapping.ts`) map a failing field to its 1-indexed step (images→1, name/desc/price/currency/categories→2, filters→3); `DEFAULT_FALLBACK_STEP = 3` for `__system`-only/unknown. `onJumpToStep` then does `setCurrentStep(Math.max(0, step - 1))` (`CreateNewProductDialog.tsx:138-140`).
- Non-validation server failure (`type === 'error'`): generic failed message (`:202-208`), fix button falls back to step 3.

### Rate-limit (429)
- In the service: `parseProductErrorsForStatus` (`productService.ts:60-69`) is the 429 synth. If a 429 body lacks a `field:null` system entry (e.g. a Cloudflare-edge 429 that didn't pass Spring's `RateLimitFilter`), it injects `byField[__system] = 'system.rate_limited'` and a `{field:null, code:'RATE_LIMITED', translationKey:'system.rate_limited'}` list item. `PARSEABLE_ERROR_STATUSES` includes 429 (`:46`).
- **At step 4** (`UploadedProductDialog`): a 429 surfaces as a `validation` result and the `__system` line is rendered in the failure list (`:199`); the fix button routes to `DEFAULT_FALLBACK_STEP` (3).
- **At step 2 pre-validate** (`BasicInfoProductDialog.tsx:128-140`): when `result.errors.byField[SYSTEM_ERROR_KEY]` is present, it sets `isRateLimited = true`, **disables the Next button** (`:308`), shows the `__system` error line (`:292-296`), and starts a fixed `RATE_LIMIT_BACKOFF_MS = 5000` (`:40`) cooldown timer (`:131-139`) that clears the system error and re-enables Next. The fixed back-off is the brief-accepted fallback because Retry-After headers aren't reliably exposed through the axios interceptor (`:36-40`).

---

## 7. Images

- **Collection:** held **in memory through the wizard**. `ImagesImport` (mounted by `ImageSelectionProductDialog.tsx:65`) writes selected files into `productData.imagesData` (an `ImageData[]`, each carrying a `.file`) via `onChange → handleChange({ imagesData })` in `CreateNewProductDialog.tsx:148`. No upload happens during steps 1–3; only client-side validation (MIME/size/dup) runs at step 1.
- **When bytes hit R2:** at the **final create step (step 4)** only. `UploadedProductDialog` calls `createNewProduct` on mount (`:54`), which calls `extractAndUploadImages` (`productService.ts:88-148`) → `uploadImages(files, 'product', ...)` (`:140`). Progress is streamed back through `onProgress` into `useUploadProgressStore` (`UploadedProductDialog.tsx:55-62`, `progress.start/setState/finish`). The resulting keys become `imageKeys` in the payload (`productService.ts:165-169`).
- **Confirms the image-pipeline contract** ("hold in memory, upload at final create step"): **YES.** The component that performs the deferred upload is **`UploadedProductDialog`** (`src/components/popups/components/UploadedProductDialog.tsx`), delegating to `createNewProduct` → `extractAndUploadImages` → `uploadImages`. Orphan cleanup on persistence failure: `cleanupOrphanImages(newKeys)` (`productService.ts:184,189,194`). Upload failures throw `UploadError` and are toasted once at the UI layer (`UploadedProductDialog.tsx:104-108`).

---

## Drift / issues noticed (out of scope)

- **A. Images are effectively optional client-side in the create flow.** Step 1's `validateImagesData` returns `''` for an empty array (`productValidator.ts:118` guard), and step 2 calls `validateProduct(..., validateImages=false)` (`BasicInfoProductDialog.tsx:93`), so no client step ever requires at least one image. A user can reach step 4 with zero images; only the server can reject. If the product contract requires ≥1 image, that gate is server-only on web. (Flagging only — the brief lists images as max 5 but says nothing about a client-side minimum, and this is the documented reference behavior.)
- **B. Full category/currency objects are sent, not IDs.** The payload spreads whole `CategoryDTO`/`CurrencyDTO`/`SelectedFilterDTO` objects (`productService.ts:166`, `...productData`). Per Part 11 the server must validate these against its own catalog (FKs the client can't misrepresent); web trusts the client to send the full shape. Not a web bug, but a parity/trust point worth stating in the cross-repo contract.
- **C. `imagesData: undefined` relies on JSON serialization dropping the key.** Correct with axios JSON, but it is an implicit contract — the field exists on the DTO type (`NewProductRequestDTO.ts`) yet is never meant to reach the wire. Noted for clarity, not a fix.
- **D. Step 3 cannot advance if reCAPTCHA never initializes.** `onNextStepInternal` only advances inside `if (recaptchaRef.current)` (`MetaDataProductDialog.tsx:31-39`); if the widget fails to mount, the Next button silently no-ops with no error shown. Edge case, flagged not fixed.
- **E. `getInitialFilters` returns `undefined` (not `[]`) before a base-site is selected** (`CreateNewProductDialog.tsx:48-49`), and seeds defaults by positional index (`options[1]`, `options[2]`, `options[0]`) for delivery/condition/availability (`:59,71,80`). Positional indexing into option arrays is brittle if the backend reorders options. Flagged, not in scope.

---

# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-06-01
**Task:** Inventory the entire product CREATE flow on web (the new-product wizard, not update/edit) as the reference contract for mobile parity.

## Implemented

- Read-only audit. No code changed.
- Documented the create wizard end to end: entry button + pre-entry location gate, the four-step component tree, per-step fields/validation/skippability, the step-conditional dialog title, the region/city default-from-profile behavior, the exact create payload field-by-field with trust-boundary classification, success/failure/429 handling, and the in-memory-then-upload-at-step-4 image pipeline.
- Findings written to `.agent/audit-product-create-parity.md` and copied to `.agent/last-session.md`.

## Files touched

- none — read-only audit. Two report files written under `.agent/` (not source).

## Tests

- Ran: none (read-only; no source touched).
- Result: N/A.
- New tests added: none.

## Cleanup performed

- none needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code changes.
- Part 4a (simplicity): N/A — read-only audit, no abstractions added/considered/removed.
- Part 4b (adjacent observations): five drift items flagged in "Drift / issues noticed," none fixed (out of scope).
- Part 6 (translations): N/A this session — translation keys were read as subject matter, not changed.
- Part 7 (error contract): exercised as subject matter — confirmed create/pre-validate consume the `{field, code, translationKey}` shape via `parseProductValidationErrors`; the 429 synth in `parseProductErrorsForStatus` was documented, not assessed.
- Part 11 (trust boundary): explicitly checked for every create-payload field (§5) — region/city is server-derived and not sent; categories/currency/filters cross as full objects the server must validate (Drift §B).

## Known gaps / TODOs

- I did not audit the update/edit page (`updateProductData`, `ProductEditState`, `app/[locale]/owner/products/[productId]`) — out of scope per the brief; a separate audit covers it.
- I cannot assert the backend's actual create-DTO acceptance shape or whether it ignores the full category/currency objects vs IDs — that lives in oglasino-backend.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- This is a Phase-2 reference audit; the consumer is the mobile (oglasino-expo) product-create parity work. The five drift items in "Drift / issues noticed" are factual observations of web behavior, not fix requests. The two most parity-relevant: **(A)** web does not enforce a client-side minimum image count in the create flow, and **(B/§5)** web sends full category/currency/filter objects and does NOT send region/city (server-derived). If mobile must match web exactly, mirror those two; if either reflects a defect rather than intended behavior, that needs a separate web brief.
- No config-file edit is required by this session. No implicit config-file dependency.
