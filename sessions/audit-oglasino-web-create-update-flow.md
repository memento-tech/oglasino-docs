# Audit — oglasino-web: full product CREATE + UPDATE behavior, EXHAUSTIVE (read-only)

**Repo:** oglasino-web
**Branch:** `stage`
**Date:** 2026-05-30
**Mode:** READ-ONLY. No code changed, no git ops, no runtime.
**Supersedes (for scope):** the two prior narrow audits `audit-create-flow.md`
(pre-validate gate + step-4 failure UX) and `audit-create-success-path.md`
(success path). This audit re-derives every claim from code on `stage`; where it
overlaps those audits it confirms them, where it extends them it says so.

All `file:line` references are to code as read this session. Every claim is
quoted or cited. Where a value is injected at deploy/runtime it is marked
**UNDETERMINED** with the variable/key name and the read site.

---

## TL;DR (the shape mobile must mirror)

- **CREATE** is a **4-step modal wizard** (`CreateNewProductDialog`), opened from
  **3 call sites**, all via `openDialog(CREATE_NEW_PRODUCT_DIALOG)` with **no
  props**. Steps: (1) images → (2) basics + server pre-validate gate → (3)
  filters + reCAPTCHA gate → (4) auto-submit on mount + in-place success/failure
  screen. On success the dialog does **not** navigate; it shows a confirmation
  screen with a copyable absolute URL, a "View link" (new tab), and a "Close"
  that **hard-reloads** the page.
- **UPDATE** is a **single-page route** (`/owner/products/[productId]`), opened
  from a dashboard card → functions drawer → "Update" (`router.push`). One form,
  all fields visible. Categories + region are **shown but disabled**. Save
  diffs via `deepEqualTest`, narrows to **7 wire fields**, POSTs, then **reseeds
  from a GET** and stays on the page with a toast.
- **Canonical product-URL domain: `https://oglasino.com`** (no `www`, `.com`).
  Every hardcoded source agrees. The one runtime value (`base.url` translation)
  is UNDETERMINED from the repo.

---

## Part 1 — Entry points (how each flow starts)

### CREATE — 3 openers, all `openDialog(DialogId.CREATE_NEW_PRODUCT_DIALOG)`, no props

`DialogId.CREATE_NEW_PRODUCT_DIALOG = 'createNewProductDialog'` —
`src/components/popups/dialogRegistry.ts:6`. Registered to the component in the
dialog map at `src/components/popups/DialogManager.tsx` (`createNewProductDialog:
CreateNewProductDialog`).

1. **`AuthAddNewProductButton`** — `src/components/client/buttons/AuthAddNewProductButton.tsx:22-32`.
   ```ts
   function openNewProductDialog() {
     if (!user) return;
     if (!user.baseSite || !user.regionAndCity) {
       openDialog(DialogId.USER_BASIC_DATA_SELECTOR_DIALOG, {
         dialogTitle: 'base.site.select.required.title',
         shouldOpenDialog: true,
       });
       return;
     }
     openDialog(DialogId.CREATE_NEW_PRODUCT_DIALOG);   // :32 — no props
   }
   ```
   Rendered as an icon button (default) or a regular button (variant). If the
   user has no `baseSite`/`regionAndCity`, it first opens the base-data selector
   with `shouldOpenDialog: true`, which chains to create (see #3).

2. **`JoinFreeZoneButton`** — `src/components/client/JoinFreeZoneButton.tsx:18-21`.
   ```ts
   onClick={() =>
     !user
       ? openDialog(DialogId.LOGIN_OPTIONS_DIALOG)
       : openDialog(DialogId.CREATE_NEW_PRODUCT_DIALOG)   // :21 — no props
   }
   ```
   Unauthenticated → login options; authenticated → create wizard directly.

3. **`UserBasicDataSelectorDialog`** (chained, not a standalone button) —
   `src/components/popups/dialogs/UserBasicDataSelectorDialog.tsx:87-88`.
   ```ts
   if (shouldOpenDialog) {
     openDialog(DialogId.CREATE_NEW_PRODUCT_DIALOG);   // :88 — no props
   }
   ```
   Fires only when opened with `shouldOpenDialog: true` (from opener #1), after
   the user picks base site + region.

**Confirmed: exactly three `openDialog(CREATE_NEW_PRODUCT_DIALOG)` call sites, all
prop-less.** This is the basis for the "`onFinish` is always `undefined`" finding
in Part 3 — `DialogManager` renders
`<CreateNewProductDialog isOpen onClose={closeDialog} {...dialogProps} />` and
`dialogProps` is empty for all three.

### UPDATE — dashboard card → functions drawer → "Update" route push

1. **`DashboardProductCard`** (a product tile on the owner products list page
   `app/[locale]/owner/products/page.tsx`) — `src/components/client/product/DashboardProductCard.tsx:27-29`:
   ```ts
   onClick={() => {
     openDialog(DialogId.DASHBOARD_PRODUCT_FUNCTIONS_DIALOG, { productOverview });
   }}
   ```
   Passes the full `productOverview: ProductOverviewDTO` (carries `id`).

2. **`DashboardProductFunctionsDialog` → "Update" button** —
   `src/components/popups/dialogs/DashboardProductFunctionsDialog.tsx:67-69`:
   ```ts
   onClick={() => {
     router.push('/owner/products/' + productOverview.id);   // :68
     onClose();
   }}
   ```
   Label `tButtons('update.product')`. **The product id is passed only as a URL
   path segment** (`/owner/products/{id}`); no state/props carry it.

3. **Direct URL / programmatic** — the edit page
   `app/[locale]/owner/products/[productId]/page.tsx` is also reachable by typing
   the URL. It reads the id from the route param (`use(params).productId`,
   `:70`). No dedicated button beyond the two above.

### Structural answer (mirrors mobile's question)

| | CREATE | UPDATE |
|---|---|---|
| Container | Modal wizard (`CreateNewProductDialog.tsx`) | Page route (`app/[locale]/owner/products/[productId]/page.tsx`) |
| Shape | 4-step wizard, 0-indexed `currentStep` (`:40`), rendered `steps[currentStep].component` (`:223`) | Single-page form, all fields visible |
| Submit | Auto-fires on step-4 mount | Explicit "Save" button |
| Categories | **Editable** (CategorySelector, step 2) | **Disabled / read-only** |
| Gates | step-2 server pre-validate, step-3 reCAPTCHA | none (client validation only) |

So web matches mobile's "create = wizard, update = single form" split.

---

## Part 2 — The create flow, step by step

Container: `CreateNewProductDialog.tsx`. Shared product-data object is
`productData: NewProductRequestDTO` held in the container (`:88`); steps push
partials up via `handleChange` spread-merge (`:125-127`,
`setProductData(prev => ({ ...prev, ...data }))`). This matches the repo's
"useState + prop-drilled `onChange(partial)` spread-merge" form pattern. Initial
`productData` is seeded on base-site resolve (`:108-123`) with default currency
`selectedBaseSite.allowedCurrencies[0]` and default filters (`getInitialFilters`,
`:48-86`: delivery option[1], condition option[2], availability option[0]).
`track('product_create_started')` fires once when `isOpen && currentStep === 0`
(`:102-106`).

### Step 1 (index 0) — Images — `ImageSelectionProductDialog.tsx`

- **Field:** images. File multi-select, **max 5**. Stored at
  `productData.imagesData: ImageData[]` (`ImageData = { key?: string; file?: File }`,
  `src/lib/types/ui/ImageData.ts`), via `onChange(imagesData)` wired in the
  container at `CreateNewProductDialog.tsx:148`.
- **Validation (on Next, not on change):** `validateImagesData` in
  `src/lib/validators/productValidator.ts:106-137` — MIME ∈
  `['image/jpeg','image/png','image/webp']` else `product.image.invalid_type`;
  size ≤ 5 MB else `product.image.too_big`; SHA-256 duplicate check else
  `product.image.duplicate`. Keys are **single strings** in the ERRORS
  namespace, surfaced via `tErrors(imageErrorKey)`.
- The image error key is **lifted to the container** (`imageErrorKey`,
  `CreateNewProductDialog.tsx:46`) so it survives step unmount/return.
- **Server call:** none here. Upload is deferred to step 4.
- **Nav:** only a Next button (no Back on step 1).

### Step 2 (index 1) — Basics + pre-validate gate — `BasicInfoProductDialog.tsx`

- **Fields & storage** (all via `onChange(partial)` → container spread-merge):
  - Category top/sub/final — `CategorySelector` (`:177-186`), stored as
    `topCategory/subCategory/finalCategory: CategoryDTO`.
  - Name — `Input`, max 80 (`:200-207`), `productData.name`.
  - Description — `Textarea`, max 2000 (`:222-231`), `productData.description`.
    An optional **OpenAI suggestion** button calls
    `getOpenAiSuggestionForProduct({title})` and writes `description` (`:153-163,
    208-216`).
  - Price — numeric `Input`, shown only when `topCategory && !topCategory.freeZone`
    (`:236-245`), `productData.price` (string).
  - Currency — `Select` over `selectedBaseSite.allowedCurrencies` (`:246-269`),
    `productData.currency: CurrencyDTO`.
  - City — `CitySelector` **disabled**, shows `user.regionAndCity` (`:272-279`).
- **Client validation (on Next, `onNextInternal` `:79-151`):** name/description
  trimmed (`:82-83`), then `validateProduct(..., validateImages=false)`
  (`productValidator.ts`). Guard checks `name | description | images |
  topCategory | subCategory | finalCategory | price` (`:96-104`). Keys (ERRORS
  namespace): `product.name.required|too_short|too_long`,
  `product.description.required|too_short|too_long`, `product.category.required`
  (first missing of the three), `product.price.required` (only non-free-zone).
- **Server call — pre-validate** (only if Zod passes): `preValidateProductBasics(name, description)`
  (`:111`) → `POST {NEXT_PUBLIC_API_URL}/secure/products/pre-validate` body
  `{ name, description }` (`productService.ts:264`). Handling:
  - `clean` → `onNextStep()` (`:114-117`).
  - `validation` → `setProductErrors(byField)`, fire
    `track('form_submit_failed', { form_name: 'product_create_step_1', error_code,
    field })` per error (`:121-127`); if `byField[__system]` set, arm a **5 s
    rate-limit backoff** (`RATE_LIMIT_BACKOFF_MS = 5000`, `:40,128-140`) that
    disables Next and auto-clears.
  - `error` (transport/5xx) → warn toast `tDialog('new.product.pre.validate.warning')`
    and **advance anyway** (`:146-150`) — pre-validate is UX, not a boundary.
- **Nav:** Back `tDialog('new.product.image.step.back')` (disabled while
  pre-validating); Next `tDialog('new.product.image.step.forward')` (disabled
  while pre-validating or rate-limited; spinner shown).

### Step 3 (index 2) — Filters + reCAPTCHA gate — `MetaDataProductDialog.tsx`

- **Fields:** the dynamic filter set, rendered by `MetaDataProduct`
  (`src/components/owner/product/MetaDataProduct.tsx`). Filters come from
  `selectedBaseSite.catalog.topFilters` (minus `age`) plus per-category filters;
  types map to `SingleOptionFilterSelector` (Select), `MultiOptionFilterSelector`
  (checkboxes), `RangeFilterSelector` (numeric), `DateFilterSelector` (year
  1930→now). Stored at `productData.filters: SelectedFilterDTO[]` via `onChange`.
- **Client validation:** none on filters (optional; server validates at step 4).
- **reCAPTCHA gate (on Next, `:30-40`):** `recaptchaRef.current.execute()`;
  inside `ReCaptchaWrapper` this gets an invisible-widget token and validates it
  via `validateReCaptcha(token)` → `POST {NEXT_PUBLIC_API_URL}/public/verify-recaptcha`
  body `{ token }`, returns `res.data.success`
  (`src/lib/service/reactCalls/recaptchaService.ts:4-14`). If not verified →
  inline `tValidation('suspicion')`; no auto-retry (user clicks Next again). If
  verified → `onNextStep()`.
- **Nav:** Back/Next reuse `new.product.image.step.back/forward`. Header copy
  `tDialog('new.product.meta.suggestion')`.

### Step 4 (index 3) — Submit — `UploadedProductDialog.tsx`

Fires `createNewProduct(productData, { onProgress })` **on mount** (guarded by
`triggeredRef`, `:43-118`). Image upload happens inside the service before the
POST. Outcomes are Part 3.

---

## Part 3 — The create submit + every outcome

### Endpoint & request body (wide DTO)

`createNewProduct` — `src/lib/service/reactCalls/productService.ts:152-198`.
`POST {NEXT_PUBLIC_API_URL}/secure/products/create` (`:171`). `BACKEND_API`
baseURL is `process.env.NEXT_PUBLIC_API_URL` (`api.ts:7,17`; example value
`https://api.oglasino.com/api`, `.env.local.example:42`); the Firebase ID token
is attached as `Authorization: Bearer`, plus `X-Base-Site` / `X-Lang` headers.

The payload is a **wide object** — it spreads the whole working DTO, only nulling
`imagesData` and adding `imageKeys` (`:160-168`):
```ts
const requestPayload: NewProductRequestDTO = {
  ...productData,            // name, description, price, currency,
                             // topCategory, subCategory, finalCategory, filters
  imageKeys: allKeys,        // from extractAndUploadImages()
  imagesData: undefined,     // working array stripped
};
```
Images: `extractAndUploadImages` (`:96-141`) keeps entries that already have
`.key`, uploads entries with `.file` via `uploadImages(..., 'product', ...)`,
and returns `{ allKeys (positional), newKeys (freshly uploaded) }`. For create,
`allKeys === newKeys`. Throws `UploadError` on upload failure (no cleanup —
nothing persisted yet).

### SUCCESS (2xx) — in order (`UploadedProductDialog.tsx:64-83`)

1. Service returns `{ type: 'success', data: { id, name } }` (`productService.ts:172-173`).
2. `track('product_create_completed', { product_id, top_category_id?,
   sub_category_id?, final_category_id?, price?, currency?, image_count })`
   (`:66-80`) — category/price/currency keys conditionally spread.
3. `setProductUrl(getNormalizedProductUrl(result.data.id, result.data.name, true, locale))`
   (`:81`); `locale = useRoutingLocale()` (`:35`).
4. `setUploadSuccess(true)` (`:82`).
5. `onFinish?.()` (`:83`) — **dead on web**: never supplied by any of the 3
   openers (Part 1), so this is a no-op. The success-screen Close therefore
   always takes the `if (!onFinish) window.location.reload()` branch.
6. `setLoading(false)` (`:99`).

**Success screen** (`:152-188`) — all DIALOG namespace except the URL string:
- Heading `tDialog('new.product.success.title', { productName: productData.name })`.
- Blurb `tDialog('new.product.success.description.1') + ' ' + …'.2'`.
- Bordered URL box: `<span>{productUrl}</span>` + `CopyIcon` whose `onClick`
  (`handleCopy`, `:120-126`) does `navigator.clipboard.writeText(productUrl)`,
  sets `copied` for 2 s (note: `copied` is **not surfaced in the render** — no
  "Copied!" feedback).
- Button **"View link"** `tDialog('new.product.success.view.link')` →
  `window.open(productUrl, '_blank')` (new tab; dialog stays open behind it).
- Button **"Close"** `tDialog('button.close.label')` →
  `onClose(); if (!onFinish) window.location.reload();` — i.e. **always closes
  and hard-reloads the underlying page** (the only thing that surfaces the new
  product into the list behind the dialog — no targeted refetch, no router nav).

  > Correction to a workflow draft: the Close label is `tDialog('button.close.label')`
  > — it resolves in the **DIALOG** namespace (the component's only button
  > translator is `tDialog`), not `BUTTONS`.

### VALIDATION FAILURE (400/422, parseable body) (`:84-93`)

Service path: `PARSEABLE_ERROR_STATUSES = {400,403,422,429,500}` +
`isProductErrorResponse` → `parseProductErrorsForStatus(status, body)` →
`{ type: 'validation', errors }` (`productService.ts:176-178`). **Orphan cleanup**
fires first if `newKeys.length > 0`: `cleanupOrphanImages(newKeys).catch(()=>{})`
(`:177`). Dialog: `setValidationErrors(errors.byField)`, then per error
`track('form_submit_failed', { form_name: 'product_create_step_4', error_code,
field })`, `setUploadSuccess(false)`.
Render (`:189-217`): red `TriangleAlert`; if field errors exist, header
`tDialog('new.product.create.failed.header')` + a `<ul>` of `tErrors(key)` per
non-system field (+ a trailing `<li>` for `__system` if present). Buttons:
**"Fix"** `tDialog('new.product.create.failed.fix')` →
`onJumpToStep(resolveTargetStep(validationErrors))`; **"Exit"**
`tDialog('new.product.create.failed.exit')` → `onClose()`.
Step routing (`src/lib/utils/productStepMapping.ts`): `images→1`,
`name/description/price/currency/topCategory/subCategory/finalCategory→2`,
`filters/filter*→3`, fallback `DEFAULT_FALLBACK_STEP=3`; `onJumpToStep` converts
1-indexed→0-indexed via `setCurrentStep(Math.max(0, step-1))`
(`CreateNewProductDialog.tsx:138-140`).

### TRANSPORT / 5xx (non-parseable or thrown) (`productService.ts:181-196`)

`logServiceWarn/Error('product.create', …)`, orphan cleanup if `newKeys>0`,
return `{ type: 'error' }`. Dialog: `setValidationErrors({})`,
`setUploadSuccess(false)` → the **no-field** branch renders
`tDialog('new.product.success.failed.1')` `<br/>` `'.2'` + the same Fix/Exit
buttons (Fix routes to `DEFAULT_FALLBACK_STEP=3`).

### RATE-LIMIT (429) (`productService.ts:52-74`)

`parseProductErrorsForStatus` synthesizes `byField[__system]='system.rate_limited'`
+ a `{ field:null, code:'RATE_LIMITED', translationKey:'system.rate_limited' }`
list entry when a 429 lacks the unified body (the Cloudflare edge may emit a
bodyless 429). Routed as `{ type: 'validation' }`. On the **step-4** screen this
renders as a single system `<li>` (no per-field errors); **no backoff/disable on
step 4** — the only timed backoff is the step-2 pre-validate gate (5 s).

### UPLOAD ERROR (image upload threw, pre-POST) (`UploadedProductDialog.tsx:100-113`)

`createNewProduct` re-throws `UploadError`; the dialog shows **one** localized
toast `buildUploadErrorTitle(error, tErrors)` (skipped if `errorCode==='CANCELLED'`)
and falls into `uploadSuccess=false` (the no-field failure screen). No orphan
cleanup (no keys persisted).

---

## Part 4 — The update flow, complete

Edit page: `app/[locale]/owner/products/[productId]/page.tsx` (`ProductDetailsPage`,
`:56-651`).

### Load (`:89-120`)

On `user` present, `getDashboardProductDetails(productId)` →
`getProductDetails(id, 'owner')` → `GET {NEXT_PUBLIC_API_URL}/secure/products?productId={id}`
(`src/lib/service/reactCalls/productsSearchService.ts:18,35-36,57-58`). Backend
`ProductDetailsDTO` is transformed into **`ProductEditState`** (image `key[]`
hydrated to `imagesData: [{key}]`). Stored into **two** snapshots:
`productDetails` (working copy) and `oldProductDetails` (baseline). Also fetches
`getUserForFirebaseUid(user.firebaseUid)` → `userInfo` (for the read-only
region/city display).

### Change detection (`:169`)

`deepEqualTest(productDetails, oldProductDetails, [])` (impl
`src/lib/utils/utils.ts:26-51`, **empty skip-keys**). Runs inside
`validateProductInternal(withDeepEqual=true)`. If equal → `setShowNoChangesMessage(true)`,
returns `false` (submit blocked); message
`tDash('product.unchanged.description')` (`:546-549`). Any field edit clears the
flag (`handleChange`, `:122-125`).

### Editable vs disabled fields

**Editable:** images (`ImagesImport`, `#field-images` `:348-360`, max 5), name
(`Input` max 80, `#field-name` `:363-373`), description (`Textarea` max 2000,
`#field-description` `:374-389`), price (numeric, only if
`topCategory && !topCategory.freeZone`; **rendered twice** — mobile `:403-415`,
desktop `:482-493`; both wrapped in `data-field="price"`), currency (`Select`,
`:416-441` / `:494-519`), filters (`MetaDataProduct`, `addCheckedFilter={false}`,
`:526-531`).

**Disabled / read-only (shown, not editable):** top/sub/final category
(`Input disabled` + `infoText 'product.category.unchangeable'`, `:447-467`);
region/city (`CitySelector disabled`, `onChange={()=>{}}`, `:394-401` /
`:472-479`).

**Are disabled values sent?** **No.** The body is built by `toUpdateWirePayload`
(`productService.ts:203-214`) which constructs a fresh object of exactly 7 fields
(below); `productState`, `moderationState`, `topCategory`, `subCategory`,
`finalCategory`, `regionAndCity`, and the working `imagesData` never enter it.

### Client validation before save (`validateProductInternal` `:133-176`)

`validateProduct(name, description, price, topCategory, subCategory,
finalCategory, imagesData, true)` (`validateImages=true`). Same ERRORS keys as
create: `product.name.*`, `product.description.*`, `product.price.required`
(non-free-zone), `product.image.invalid_type|too_big|duplicate`,
`product.category.required`.

### Update endpoint & request body (narrowed — 7 fields)

`updateProductData` — `productService.ts:216-257`. `POST {NEXT_PUBLIC_API_URL}/secure/products/update`
(`:228`). Body shape `UpdateProductRequestDTO`
(`src/lib/types/product/UpdateProductRequestDTO.ts:9-17`), built at `:225`:
```ts
{ id, name, description, price?, currency?, filters?, imageKeys? }
```
**No `oldName`/`oldDescription`** (forbidden per conventions Part 11 — comment in
the DTO file), **no state/moderation/category/region**. Narrowing site:
`toUpdateWirePayload` `productService.ts:203-214`. Image handling identical to
create (`extractAndUploadImages`), but `newKeys ⊆ allKeys` so cleanup on failure
only targets freshly-uploaded keys (never the entity's existing images).

### Update SUCCESS — reseeds from GET, stays on page (`:203-237`)

1. `notify.success({ title: tDash('product.update.success.title'), description:
   tDash('product.update.success.description') })`.
2. **Reseed:** `getDashboardProductDetails(productDetails.id)` again; on success
   `setOldProductDetails(refreshed)` + `setProductDetails(refreshed)` +
   refresh `visibleImage` (`:217-224`). On null/throw → warning toast
   `tDash('product.update.refresh.fail')`.
3. **No navigation, no dialog close, no page reload.** The user stays on the
   edit page with post-save data so the next diff is against the saved state.
   (This matches mobile's "reseed from GET on success".)

### Update FAILURE outcomes (`:238-285`)

- **Validation (400/422/429/500, parseable):** `setProductErrors(byField)`; per
  error `track('form_submit_failed', { form_name: 'product_update', error_code,
  field })`. Errors render **inline per field** via each input's `errorMessage`
  prop (name `:369`, description `:381`, price `:411/491`, images `:356-360`),
  and the system error under `#field-system-error` (`:536-541`). Then
  **scroll-to-first-error** (next section).
- **429 (rate limit):** same `system.rate_limited` synth as create
  (`productService.ts:59-74`); surfaces as the `#field-system-error` text. No
  client backoff/disable on the update page.
- **Upload error (`UploadError`, ≠ CANCELLED):** single toast
  `buildUploadErrorTitle(err, tErrors)` (`:270-274`).
- **Transport / other thrown:** `{ type:'error' }` path → `setSubmitFailed(true)`
  (`:259-261`) renders the page-level `tDash('product.update.fail.message')`
  (`:543-545`); the non-UploadError catch also fires a toast
  `tDash('product.update.fail.message')` (`:275-281`).

Other page actions (out-of-band, all `window.location.reload()` or nav):
Preview (opens `PREVIEW_PRODUCT_DIALOG` with unsaved data, `:288-294`), Cancel
(`router.back()`), Activate/Deactivate (`activate/deactivateProduct` + reload,
`:570-617`), Delete (`INFO_DIALOG` confirm → `deleteProduct` →
`router.replace('/owner/products')`, `:619-647`).

### Scroll-to-error — WORKS (prior "no-op" hint REFUTED)

Two identical mechanisms — client validation (`:154-163`) and server-validation
response (`:247-258`):
```ts
const target = byField['name'] ? getElementById('field-name')
  : byField['description'] ? getElementById('field-description')
  : byField['images'] ? getElementById('field-images')        // client path only
  : byField['price'] ? querySelectorAll('[data-field="price"]').find(el => el.offsetParent !== null)
  : byField[SYSTEM_ERROR_KEY] ? getElementById('field-system-error')  // server path only
  : null;
target?.scrollIntoView({ behavior: 'smooth', block: 'center' });
```
All `getElementById` targets are real rendered ids: `field-name` (`:363`),
`field-description` (`:374`), `field-images` (`:348`), `field-system-error`
(`:536`). The **price** target is the only viewport-conditional one: there are
two `data-field="price"` wrappers (`:403` mobile, `:482` desktop) and the
selector picks the **visible** one via `offsetParent !== null` (null under
`display:none`). **Verdict: functional, including price — NOT a no-op.** The
prior audit's hint does not hold against current code on `stage`.
(`MetaDataProduct.tsx` is a pure filter UI; no scroll logic lives there.)

---

## Part 5 — Shared mechanics both flows rely on

### Product-URL helper — one helper, used everywhere

`getNormalizedProductUrl(productId, productName, withPrefix=false, locale?)` —
`src/lib/utils/utils.ts:115-122`:
```ts
return `${withPrefix ? `https://oglasino.com/${locale}` : ''}/product/${productId}/${normalizeProductName(productName)}`;
```
- Domain hardcoded `https://oglasino.com` — **no `www`, `.com`** — emitted only
  when `withPrefix === true` (else a relative path).
- Locale segment is whatever string the caller passes (create passes
  `useRoutingLocale()`, e.g. `rs-sr`).
- Slug: `normalizeProductName` (`:128-147`) — lowercase → NFD → strip diacritics
  → `đ/Đ→d` → `[^a-z0-9]+ → '-'` → collapse `-` → trim `-`. E.g. `"Opel Astra"`
  id 42 locale `en` → `https://oglasino.com/en/product/42/opel-astra`.

Callers: create success screen `UploadedProductDialog.tsx:81` (`withPrefix=true`);
`app/sitemap.ts:179` (path form, prefixed by `${SITE_URL}/${bs.code}-${lang.code}`);
`src/metadata/generateProductPageMetadata.ts` (both prefix variants);
dashboard share helper. **No divergent product-URL builder exists** — same
function, same output for same inputs. (Sibling `getDashboardNormalizedProductUrl(id)`
`:124-126` returns the internal `/owner/products/{id}` path — a different,
intentional purpose, not a public product URL.)

### Error parser — single helper, both flows

`parseProductValidationErrors(response)` —
`src/lib/utils/parseProductValidationErrors.ts:31-45`. First-error-per-field
wins; `field:null` collapses to `SYSTEM_ERROR_KEY='__system'`; returns
`{ byField, list }`. Both flows go through it (create + pre-validate via
`productService.ts:65,271,275,293`; update via `updateProductData` →
`productService.ts:65` → consumed at `page.tsx:239-240`). **Confirmed: same
helper, both flows.**

### Translation namespaces & keys (both flows)

- **DIALOG** (`tDialog`): `new.product.title`, `new.product.basic.suggestion`,
  `new.product.meta.suggestion`, `new.product.product.name.advice`,
  `new.product.product.description.advice`, `open.ai.suggest.label`,
  `new.product.image.step.back`, `new.product.image.step.forward`,
  `new.product.pre.validate.warning`, `new.product.success.title`,
  `new.product.success.description.1`, `new.product.success.description.2`,
  `new.product.success.view.link`, `button.close.label`,
  `new.product.create.failed.header`, `new.product.create.failed.fix`,
  `new.product.create.failed.exit`, `new.product.success.failed.1`,
  `new.product.success.failed.2`. Update page also uses
  `info.delete.product.alert.*`.
- **INPUT** (`tInputs`): `product.name`, `product.description`, `price.label`,
  `currency.select.placeholder`, `currency.select.header`.
- **VALIDATION** (`tValidation`): `form.incomplete`, `suspicion`.
- **ERRORS** (`tErrors`): all dynamic backend `product.*` / `image.*` /
  `system.rate_limited` keys + client Zod/image keys
  (`product.name|description.*`, `product.category.required`,
  `product.price.required`, `product.image.invalid_type|too_big|duplicate`),
  plus `unknown`.
- **DASHBOARD_PAGES** (`tDash`, update page): `product.fail`, `product.id.label`,
  `product.name.advice`, `product.description.advice`, `product.categories.label`,
  `product.top.category.label`, `product.sub.category.label`,
  `product.category.label`, `product.category.unchangeable`,
  `product.filters.label`, `product.unchanged.description`,
  `product.update.success.title`, `product.update.success.description`,
  `product.update.refresh.fail`, `product.update.fail.message`,
  `product.update.cancel`, `product.update.preview`, `product.update.save`,
  `back.button.label`.
- **BUTTONS** (`tButtons`, update page): `update.product`, `activate.product`,
  `deactivate.product`, `delete.product`.
- **COMMON / COMMON_SYSTEM** (`tCommon`/`t`): `product.function.activate.notification`,
  `product.function.deactivate.notification`; category label keys passed through `t(...)`.

**Seeding:** this repo holds **no** translation seed/JSON — namespaces are fetched
at runtime from `GET {NEXT_PUBLIC_API_URL}/public/translations?namespace=&lang=`
(`src/translations/lib/translationsCache.ts`). Therefore whether any specific key
above is seeded is **UNDETERMINED from this repo** — it must be confirmed against
the backend SQL seed. The client-origin Zod/image keys (`product.<field>.*`,
`product.image.*`) and the `system.rate_limited` synth are web-authored strings
that the backend must seed in ERRORS for them to render.

### Analytics — every `track(...)` in scope

`track(event, params)` → `window.gtag('event', …)` gated on consent
(`src/lib/analytics/track.ts`). Mobile has none of these.

| Event | Site | Payload |
|---|---|---|
| `product_create_started` | `CreateNewProductDialog.tsx:104` | `{}` |
| `form_submit_failed` | `BasicInfoProductDialog.tsx:122` | `{ form_name:'product_create_step_1', error_code, field }` (per pre-validate error) |
| `product_create_completed` | `UploadedProductDialog.tsx:66` | `{ product_id, top_category_id?, sub_category_id?, final_category_id?, price?, currency?, image_count }` |
| `form_submit_failed` | `UploadedProductDialog.tsx:87` | `{ form_name:'product_create_step_4', error_code, field }` (per submit error) |
| `form_submit_failed` | `app/[locale]/owner/products/[productId]/page.tsx:241` | `{ form_name:'product_update', error_code, field }` (per update error) |

There is **no `product_update_completed` success event** — update fires analytics
only on failure.

### Web-only mechanisms (deliberate RN translation needed)

- Clipboard: `navigator.clipboard.writeText(productUrl)` — `UploadedProductDialog.tsx:123`.
- New tab: `window.open(productUrl, '_blank')` — `UploadedProductDialog.tsx:173`.
- Hard reload: `window.location.reload()` — create Close `UploadedProductDialog.tsx:181`;
  update activate/deactivate `page.tsx:588,611`; dashboard functions dialog.
- Router nav: `router.push('/owner/products/'+id)` `DashboardProductFunctionsDialog.tsx:68`;
  `router.back()` / `router.replace('/owner/products')` on the edit page.
- DOM scroll: `getElementById`/`querySelectorAll` + `scrollIntoView` — edit page
  `:154-163, 247-258`.

---

## Part 6 — The canonical domain (settle it completely)

| Source | file:line | Value | Notes |
|---|---|---|---|
| URL helper | `src/lib/utils/utils.ts:121` | `https://oglasino.com` | hardcoded, no www, product URLs |
| Sitemap | `app/sitemap.ts:9` | `const SITE_URL='https://oglasino.com'` | hardcoded |
| Robots | `app/robots.ts:3` | `const SITE_URL='https://oglasino.com'` | hardcoded |
| Hreflang verifier | `scripts/verify-hreflang.ts:222` | `https://oglasino.com` | hardcoded canonical assert |
| Metadata baseUrl | `src/metadata/generalMetadata.ts:5` | `t('base.url')` | **UNDETERMINED** — METADATA namespace key `base.url`, runtime-seeded from backend `/public/translations` |
| `BasicMetadata.baseUrl` | `src/lib/types/configuration/BasicMetadata.ts:13` | (type only) | value flows from `generalMetadata` above |
| `next.config.ts` | `:19-23` | `oglasino.com`, `www.oglasino.com`, `stage.oglasino.com`, `www.stage.oglasino.com` | **CSRF Server-Action allowlist**, not a canonical URL — includes a `www` variant for origin matching |
| `NEXT_PUBLIC_SITE_URL` | `.env.local.example:53-54` | `http://localhost:3000` | comment marks it **NOT used**; dev-only |
| `NEXT_PUBLIC_API_URL` | `.env.local.example:42` | `https://api.oglasino.com/api` | backend API base, not the site domain |

**Verdict:** Web's canonical product-URL domain is **`https://oglasino.com`
(no `www`, `.com`)**. Every **hardcoded** source — the product-URL helper,
sitemap, robots, and the hreflang verifier — is **internally consistent** on that
exact string. The single non-hardcoded value, `generalMetadata.baseUrl` from the
`base.url` METADATA translation, is **UNDETERMINED from the repo** (deploy-seeded);
it is the only place a divergent domain (`www`, `.rs`) could be introduced
without a code change, so it should be confirmed against the seed when
standardizing mobile. The `next.config.ts` `www.oglasino.com` entry is a CSRF
allowlist, not a URL source, and does not make the code inconsistent.

---

## The user's complete journey

### CREATE (first action → final screen)

User taps "+" (`AuthAddNewProductButton`) or "Join free zone"
(`JoinFreeZoneButton`). If logged-out → login dialog; if missing base-site/region
→ base-data selector first, which chains into the wizard. The 4-step modal opens
(`product_create_started`). **Step 1:** pick up to 5 images; Next runs MIME/size/
duplicate checks. **Step 2:** category, name (with optional AI description),
description, price+currency (non-free-zone), disabled city; Next runs Zod then a
server pre-validate of name+description — clean advances, violations show inline
(+ a 5 s backoff on 429), transport errors warn-and-advance. **Step 3:** filters,
then an invisible reCAPTCHA must verify before advancing. **Step 4:** the dialog
auto-submits — uploads new images, POSTs the wide DTO to `/secure/products/create`.
On success: `product_create_completed`, and the same dialog becomes a
confirmation screen with the absolute `https://oglasino.com/{locale}/product/{id}/{slug}`
URL (copy icon + "View link" opens a new tab); "Close" closes and **hard-reloads**
the page behind it. On validation failure: a red error list + "Fix" (jumps back
to the offending step) / "Exit". On transport/429/upload error: a generic failure
screen (+ a toast for upload errors). The user is never auto-navigated to the
product.

### UPDATE (first action → final screen)

From the owner products list, the user taps a product card → a functions drawer →
"Update", which `router.push`es to `/owner/products/{id}`. The page GETs the
product (`/secure/products?productId=`), seeds working + baseline `ProductEditState`,
and renders one form: editable images/name/description/price/currency/filters;
**disabled** categories ("unchangeable") and region/city. The user edits and taps
"Save": client Zod runs (scrolling smoothly to the first bad field, price
included), then a `deepEqualTest` against the baseline — identical → "no changes"
message, no request. Otherwise it uploads any new images and POSTs the **narrowed
7-field** DTO to `/secure/products/update`. On success: a toast, then a **GET
reseed** of both snapshots — the user stays on the page with saved data (no nav,
no reload). On validation failure: inline per-field errors + a scroll to the
first; `form_submit_failed` per error. On 429: a system-error line. On transport/
upload error: a page-level message and/or toast. Activate/Deactivate reload the
page; Delete confirms then routes back to the list.

---

## For Mastermind

### Web-only mechanisms needing a deliberate RN translation
- Clipboard copy of the product URL (success screen) — `UploadedProductDialog.tsx:123`.
- "View link" new browser tab (`window.open(_,'_blank')`) — `:173`.
- "Close" → full-page reload as the *only* list-refresh mechanism — `:181`.
- Update activate/deactivate → `window.location.reload()` — `page.tsx:588,611`.
- Smooth DOM `scrollIntoView` to first error (create routes by step; update
  scrolls within the page) — `page.tsx:154-163,247-258`.

### Web behaviors that DIVERGE from a naive mobile rebuild (candidate parity fixes)
- **Create success is in-place, not a navigation.** Web shows a confirm screen +
  reload; it does NOT route to the product. Mobile should mirror intent (confirm +
  opt-in view), not auto-navigate, unless deliberately diverging.
- **`onFinish` is dead on web** (no opener passes it). The reload-on-close branch
  *is* the web behavior. Don't model mobile on the unused callback.
- **Create POSTs a WIDE DTO** (`{...productData, imageKeys, imagesData:undefined}`),
  but **Update POSTs a NARROW 7-field DTO** (`toUpdateWirePayload`). Mobile must
  replicate this asymmetry: update strips state/moderation/category/region and
  never sends `oldName`/`oldDescription` (trust boundary, conventions Part 11).
- **Update reseeds from GET on success and stays put**; create reloads. Two
  different "refresh" strategies — mobile should match each per flow.
- **Two timed/gated behaviors exist only where web puts them:** step-2 pre-validate
  with a **5 s** 429 backoff, and step-3 reCAPTCHA. There is **no** backoff/disable
  on the create step-4 screen or on the update page.
- **Analytics:** mobile has none. To reach parity it needs 4 create events +
  1 update-failure event (table in Part 5). Note there is **no update-success
  event** on web — don't invent one for parity.
- **Pre-validate is UX-only** (transport error → warn + advance). Mobile must not
  treat it as a gate.

### Web keys not confirmed seeded (web frontend additions)
This repo has no seed; all keys are runtime-fetched, so seeding is UNDETERMINED
here and must be checked against the backend SQL seed. The web-authored strings
most at risk of being unseeded are the client Zod/image keys
(`product.name|description.required|too_short|too_long`,
`product.category.required`, `product.price.required`,
`product.image.invalid_type|too_big|duplicate`) and the rate-limit synth
`system.rate_limited`. Hand this list to Backend to confirm/seed.

### Possible bug / smell (mirror intent, not the bug)
- The create **copy icon sets `copied` state but never renders any "Copied!"
  feedback** (`UploadedProductDialog.tsx:39,122-124,164-167`) — the success state
  is computed and discarded. Low severity (cosmetic); flag so mobile gives real
  copy feedback rather than copying a dead state. **Not fixed — read-only audit,
  out of scope.**

### Config-file impact
None — read-only audit. No `conventions.md` / `decisions.md` / `state.md` /
`issues.md` edit is required by this work.
