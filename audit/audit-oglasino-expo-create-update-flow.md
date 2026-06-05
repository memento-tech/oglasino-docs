# Mobile audit — Create + Update product flow (EXHAUSTIVE)

- Repo: `oglasino-expo`
- Branch: `new-expo-dev`
- Mode: READ-ONLY (no code changes)
- Date: 2026-05-30
- Web reference mapped: `../oglasino-web/.agent/audit-create-update-flow.md`

This document traces every user-observable behavior of the mobile product CREATE and UPDATE flows and maps each web behavior (B1–B27) to its mobile status. It is built from source-tracing sub-audits and their verification verdicts. Where a verify verdict corrected a trace finding, the corrected version is used and the correction is noted.

**Out of parity scope (stated once, excluded from the delta table):**
- **reCAPTCHA** (web B8): mobile has **NO reCAPTCHA**. A source search for `reCAPTCHA`/`recaptcha` returns nothing; `MetaDataProductDialog.tsx` step 3 has no token gate — Next directly calls `onNextStep` (`MetaDataProductDialog.tsx:45`). reCAPTCHA is a security/anti-abuse concern, not a parity behavior, so it is excluded from the table below.
- **Analytics** (web B16): mobile has **NO analytics**. No `track('product_create_started')`, `track('product_create_completed')`, or `track('form_submit_failed')` calls exist (grep returns nothing). Mobile's absence of analytics is intentional per brief; excluded from the table.

---

## Part 1 — Entry points

There are **three distinct UI surfaces** that open the CREATE wizard (four call sites total, since the free-zone page renders the create CTA twice), and **one** UPDATE entry path (dashboard card → functions dialog → route push). All CREATE openers call `openDialog(DialogId.NEW_PRODUCT_DIALOG)` with **no props**, mirroring web's prop-less contract.

The dialog store defaults props to `{}` (`src/components/dialog/store/useDialogStore.ts:14`):

```
openDialog: (id, props = {}) => set({ currentDialogId: id, dialogProps: props }),
```

`DialogId.NEW_PRODUCT_DIALOG` is mapped to `AddUpdateProductDialog`, the 4-step wizard container.

### CREATE entry points

1. **Bottom-bar FAB** — `src/components/navigation/BottomBar.tsx:144`
   `onPress={() => openDialogSafe(() => openDialog(DialogId.NEW_PRODUCT_DIALOG))}`
   Plus-icon floating button; `openDialogSafe` checks auth, then opens the create wizard with no props. (Equivalent to web's auth-gated add-product button on a different UI surface.)

2. **Dashboard sidebar** — `src/components/dashboard/layout/DashboardSidebar.tsx:65`
   `<Pressable onPress={() => openDialog(DialogId.NEW_PRODUCT_DIALOG)}>`
   PlusCircle icon. No explicit auth guard (dashboard context is already authenticated).

3. **Free-zone page header CTA** — `app/(portal)/(public)/blog/free-zone.tsx:62`
   `!user ? openDialog(DialogId.LOGIN_OPTIONS_DIALOG) : openDialog(DialogId.NEW_PRODUCT_DIALOG)`
   Unauthenticated → login options; authenticated → create wizard. Mirrors web's `JoinFreeZoneButton`.

4. **Free-zone page support-section CTA** — `app/(portal)/(public)/blog/free-zone.tsx:118`
   Same auth-guard logic as #3 (the page renders the CTA twice).

These map to web **B1** (three prop-less openers; `onFinish` dead). Mobile's `AddUpdateProductDialog` receives only `onClose` (`AddUpdateProductDialog.tsx:25`) — there is no `onFinish` in the type signature, so the dead-callback concern does not exist on mobile.

### UPDATE entry point

- Dashboard product card press opens the functions dialog with product metadata — `src/components/dashboard/components/DashboardProductCard.tsx:17`:
  `openDialog(DialogId.DASHBOARD_PRODUCT_FUNCTIONS_DIALOG, { productId, productName, productState, moderationState, onRequestProductRefresh })`
- In that dialog, the "Update" button pushes the edit route — `src/components/dialog/dialogs/DashboardProductFunctionsDialog.tsx:129`:
  `router.push(getDashboardNormalizedProductUrl(productId));` → `/owner/dashboard/products/{productId}`
- The edit screen reads the route param — `app/owner/dashboard/products/[productId].tsx:42-44`:
  `const { productId } = useLocalSearchParams<{ productId: string; }>();`

This maps to web **B2** (`router.push('/owner/products/' + id)`). Same intent — route by id; mobile's path is `/owner/dashboard/products/{id}`.

---

## Part 2 — Create flow, step by step

Container: `src/components/dialog/dialogs/product-creation/AddUpdateProductDialog.tsx`. It holds a single `productData: NewProductRequestDTO` state, 0-indexed `currentStep` (`:32`), and a 4-step `steps[]` array (`:123-171`). The progress bar (`:181-183`) hides on the final step. Steps communicate via `handleChange` (`:104`), a spread-merge into the shared object. Seeding on base-site resolve (`:68-106`) sets default currency (`allowedCurrencies[0]`) and default filters (delivery[1], condition[2], availability[0]) — same as web. Maps to web **B3**.

### Step 1 — Images
File: `ImageSelectionProductDialog.tsx`. Field `imagesData: ImageData[]`, max 5 (line 53). On Next, `validateImagesData` (`productValidator.ts:91-120`) checks MIME ∈ `['image/jpeg','image/png','image/webp']`, size ≤ 5 MB, URI dedup; returns a single `product.image.invalid_type|too_big|duplicate` key (ERRORS namespace). No server call (upload deferred to step 4). **Only a Next button, no Back** (`ImageSelectionProductDialog.tsx:60-64`). Maps to web **B4**.

### Step 2 — Basics + pre-validate gate
File: `BasicInfoProductDialog.tsx`. Fields: category top/sub/final (`:170-178`), name max 80 (`:187-195`), description max 2000 (`:208-221`), price numeric & conditionally required for non-free-zone (`:223-253`), currency Select (`:236-251`), city disabled/read-only (`:255-259`). Client validation (`validateProduct(productData, false)`, `:104`) keys (ERRORS namespace, `productValidator.ts:25-89`): name 3–80 (`required|too_short|too_long`), description 10–2000 (`required|too_short|too_long`), `product.category.required`, `product.price.required` (non-free-zone only, price 1–9999999). Maps to web **B5**.

**Pre-validate gate** (`onNextInternal`, `:97-135`): Zod-style validation first; only if it passes, probe `classifyPreValidate(name, description)`. Outcomes:
- `clean` → `onNextStep()` (advance) (`:125`)
- `validation` → render inline violations + optional rate-limit backoff (`:127`)
- `error` → warning toast + advance (pre-validate is UX, not a hard gate) (`:130`)

Server call: `POST /secure/products/pre-validate {name, description}` (`productService.ts:135-146`). Maps to web **B6**.

**5s rate-limit backoff** (verified): `RATE_LIMIT_BACKOFF_MS = 5000` (`:31`). `startRateLimitBackoff` (`:72-81`) sets `rateLimited=true`, stores the system error, and auto-clears after 5000 ms. Triggered only when pre-validate returns a `__system` entry (`:94` `if (sysError) startRateLimitBackoff(sysError);`). Disables Next (`:280` `disabled={preValidating || rateLimited}`) and renders the system error below the button bar (`:289-293`). Matches web **B6** exactly (verify verdict CONFIRMED).

### Step 3 — Filters
Files: `MetaDataProductDialog.tsx`, `MetaDataProduct.tsx`. Dynamic filter set from `selectedBaseSite.catalog.topFilters` minus `'age'` (`MetaDataProduct.tsx:48`) plus per-category filters, rendered by filter type. Stored at `productData.filters: SelectedFilterDTO[]`. **No client validation** (server validates at step 4). Maps to web **B7**. **reCAPTCHA gate (web B8): NOT PRESENT** — out of scope, see header.

### Step 4 — Submit
File: `UploadedProductDialog.tsx`. **Auto-fires on mount** via `useEffect(() => { uploadProductInternal(); }, [productData])` (`:143-145`), guarded against re-entry by `if (outcome?.kind === 'success') return;` (`:79`) so "Go back and fix" re-uploads fresh. Maps to web **B9**. Full submit behavior is in Part 3.

---

## Part 3 — Create submit + every outcome

### Endpoint, method, request body

- Endpoint/method: `POST /secure/products/create` — `productService.ts:105-110`. `X-Base-Site` and `X-Lang` headers auto-added by the axios interceptor (`api.ts:44-46`).
- Image upload happens **before** the POST inside `uploadProduct` (`productService.ts:42-95`): `uploadImages` runs first, slots are mapped to `allKeys`, then `createProduct(toCreateWirePayload(productData, allKeys))` is called.

**Request body — NARROW allow-list DTO (this is a wire-shape difference from web, but produces the same fields):** `toCreateWirePayload` (`productWirePayload.ts:15-30`) builds a fresh object via an allow-list:

```
name, description, price, currency, topCategory, subCategory, finalCategory, filters, imageKeys
```

The type `CreateProductWireDTO` (`CreateProductWireDTO.ts:13-25`) is a `Pick<NewProductRequestDTO, ...> & { imageKeys: string[] }`, structurally excluding `free`, `regionAndCity`, and `imagesData`.

**Verify verdict correction:** the trace finding initially labeled the request body as `DIVERGES` (web spreads a wide DTO, mobile narrows via allow-list). The verify verdict CONFIRMED that **both flows send the identical 9 fields on the wire** — web by spreading `{...productData, imageKeys, imagesData: undefined}` then nulling `imagesData`, mobile by allow-list. The user-observable wire payload is the same. Therefore the delta-table status for the create wire shape is **MATCHES** (the allow-list vs spread is an internal implementation detail, not a user-facing or contract difference). Maps to web **B9**.

### SUCCESS outcome

On 2xx (`UploadedProductDialog.tsx:109-116`):
```
setProductUrl(getNormalizedProductUrl(result.id, result.name, true));   // share URL (with prefix)
setProductRoute(getNormalizedProductUrl(result.id, result.name));        // in-app route (no prefix)
setOutcome({ kind: 'success' });
triggerDashboardReload();
```

Success screen (`:180-214`):
- Title with product name — `tDialog('new.product.success.title', { productName })` (`:182-186`). Maps to web **B10** title.
- Description line 1 — `tDialog('new.product.success.description.1')` (`:187-189`).
- Description line 2 — `tDialog('new.product.success.description.2')` (`:190`).
- Bordered URL box with copy icon (`:192-204`): shows `productUrl` italic + `CopyIcon`.
- **Copy action** (`handleCopy`, `:147-157`): `Clipboard.setStringAsync(productUrl)` (expo-clipboard), sets `copied=true` for 2 s, silent on failure.
- **Copy feedback render** (`:199-203`): when `copied`, renders `tDialog('new.product.success.copy.done.label')`. **This is where mobile differs from web: web computes `copied` state but never renders it (B10, dead code / smell). Mobile renders real "Copied!" feedback — a mobile-better improvement.**
- **"View link" button** (`:206-209`) → `viewProduct()` (`:162-166`): `onClose()` then `router.push(productRoute)` — in-app navigation. Web opens a new browser tab (`window.open`). DELIBERATE-RN-DIFF.
- **"Close" button** (`:210-212`): label `tDialog('new.product.success.finish.label')`, calls `onClose()`. Web's label is `button.close.label` (DIALOG namespace) and web's close does `window.location.reload()`. Mobile uses a different label key and does NOT reload — instead `triggerDashboardReload()` (already called at `:116`) bumps `useDashboardProductsStore` `reloadNonce` (`useDashboardProductsStore.ts:17`), remounting/refetching the owner products list. DELIBERATE-RN-DIFF (RN has no `window.location.reload`).

### VALIDATION failure (400/422)

Parse (`:127-135`): `parseServiceError(error)` → `findSystemError(parsed.errors)`. If field errors or a system error exist → `outcome = { kind: 'validation', byField, systemError }`, else `{ kind: 'error' }`.

Render (`:231-259`): red `TriangleAlert`, `failureHeader` (`new.product.create.failed.header`), one bullet per field via `tError(translationKey)`, plus a system-error bullet if present (`:241-245`). Buttons: **"Go back and fix"** → `onJumpToStep(resolveTargetStep(outcome.byField))` (`:250`) and **"Exit"** → `onClose()`. `resolveTargetStep` (`productStepMapping.ts:32-52`) maps images→1, name/description/price/currency/category→2, filters→3, unmapped→3. Maps to web **B11**.

### TRANSPORT / 5xx (non-parseable)

`outcome = { kind: 'error' }` → render (`:261-281`): red `TriangleAlert`, `new.product.success.failed.1` + newline + `.2`, "Go back and fix" → `onJumpToStep(DEFAULT_FALLBACK_STEP)` (= 3, `productStepMapping.ts:25`), and "Exit". Maps to web **B12**.

### 429 rate-limit on step-4 POST

Parsed as a `field: null` entry → rendered as a single system-error bullet on the step-4 validation screen (`:241-245`). **No 5s backoff on step 4** — the backoff exists only on step-2 pre-validate. Maps to web **B13**.

### Upload error (image upload fails before POST)

`if (error instanceof UploadError)` (`:120-126`): one localized toast via `buildUploadErrorTitle(error, tError)` (skipped if `error.code === 'CANCELLED'`), then `outcome = { kind: 'upload' }`. Render (`:217-229`): `TriangleAlert`, `new.product.success.failed.1/2`, single "Go back" button → `onBackStep()`. Maps to web **B14**.

### Orphan cleanup

On any persistence throw, if images were uploaded, cleanup runs before rethrow (`productService.ts:86-94`): `if (newKeys.length > 0) void cleanupOrphanImages(newKeys); throw err;`. Maps to web **B15**.

### Product-URL copy-vs-nav specifics (pinned)

- **Copy/share string** uses `getNormalizedProductUrl(result.id, result.name, true)` → `https://www.oglasino.rs/product/{id}/{slug}` (`UploadedProductDialog.tsx:109`, helper `utils.ts:114`). This is the string shown in the box AND copied to clipboard.
- **In-app nav target** uses `getNormalizedProductUrl(result.id, result.name)` (default `withPrefix=false`) → relative `/product/{id}/{slug}` (`:110`), consumed by `router.push` in `viewProduct` (`:165`).
- The copied string and the nav target therefore use the **same helper** but the copied one is prefixed with `https://www.oglasino.rs` while web's copied string is `https://oglasino.com/${locale}`. See Part 5 for the full product-URL divergence. Copy feedback ("Copied!") is rendered on mobile, unlike web.

---

## Part 4 — Update flow, complete

Screen: `app/owner/dashboard/products/[productId].tsx`.

### Load
`fetchProduct` (`:109-135`) calls `getDashboardProductDetails(parseInt(productId))` → `GET /secure/products?productId={id}` (`productsSearchService.ts:36-38`), then seeds **two snapshots**: `productDetails` (working copy) and `oldProductDetails` (baseline), and sets `visibleImage` from the first image key. Maps to web **B17**.

### Editable vs disabled fields
- **Editable** (`:294-400`): images, name, description, price, currency, filters — all call `handleChange` (spread-merge). Maps to web **B18** editable set.
- **Disabled / read-only** (`:331-393`): `CitySelector` and `CategorySelector` both `disabled={true}` with `onChange={() => {}}` no-ops; region/city shown as display text. Maps to web **B18** disabled set.

### Client validation
`validateProduct(productDetails, true)` (`:141-156`) checks name (3–80), description (10–2000), price (non-free-zone, >0), category (all three required), images (MIME / ≤5 MB / dedup), returning `AddUpdateProductErrors` with `nameErrorKey`, `descriptionErrorKey`, `categoryErrorKey`, `priceErrorKey`, `imagesErrorKey`. Same keys as create. Maps to web **B19**.

### Deep-equal change detection
`if (withDeepEqual && deepEqualTest(productDetails, oldProductDetails))` (`:161`) — empty skip-keys. On exact match: warning toast `product.unchanged.title` + `product.unchanged.description`, returns `false`, blocks save. Maps to web **B20**.

### Submit
`uploadProduct(productDetails, { mode: 'update', id })` → `updateProduct(toUpdateWirePayload(productData, id, allKeys))` → `POST /secure/products/update` (`productService.ts:87-90, 117-126`). `toUpdateWirePayload` (`productWirePayload.ts:43-57`) is a narrowed **7-field** DTO: `id, name, description, price, currency, filters, imageKeys`. Type `UpdateProductWireDTO` is a `Pick` that structurally excludes forbidden fields. Tests (`productWirePayload.test.ts:88-117`) verify the exact 7 fields and that it **never carries** `oldName, oldDescription, productState, moderationState, topCategory, subCategory, finalCategory, regionAndCity, free, imagesData`. Verify verdict CONFIRMED. Maps to web **B21** (identical 7-field shape and forbidden list).

### Success
`:222-226` then `reseedFromServer` (`:177-187`): success toast `product.update.success.title` + `.description`, then GET the product again and reset **both** snapshots (`setProductDetails` + `setOldProductDetails`), clear errors, refresh `visibleImage`. **No navigation, no reload, stays on screen.** Verify verdict CONFIRMED. Maps to web **B22**.

### Validation failure
`classifyUpdateError` (`updateSubmitOutcome.ts`) routes a validation outcome to three buckets, handled at `:236-246`:
- `fieldErrors` → inline slots via `errorMessage` props: name (`:309`), description (`:321-323`), price (`:354`), category (`:388-391`), images (`imagesErrorKey`). Maps to web **B23** inline behavior.
- `systemError` (field `null`, e.g. rate-limit) → near-Save block (`:403-418`).
- `otherErrors` → unmapped server field errors collected (`updateSubmitOutcome.ts:74-77`, `MAPPED_FIELDS` at `:36-44`) and rendered near Save. **Web has NO `otherErrors` catch-all (B23); web silently drops unmapped field errors. Mobile adds this — a mobile-better improvement.** Verify verdict CONFIRMED.

Then `scrollToFirstError(fieldErrors, hasNearSaveMessages)` (`:78-107`): scans images→name→description→price→category, uses `findNodeHandle` + `measureLayout` + `ScrollView.scrollTo({ animated: true })`; falls back to `scrollToEnd` if only near-Save messages exist. RN-native equivalent of web's DOM `scrollIntoView` (B27). DELIBERATE-RN-DIFF. Verify verdict CONFIRMED.

### 429 on update
Parsed as a validation outcome with `systemError`; rendered near Save (`:403-418`). **Save stays live** — no backoff, no disable (comment N1 at `:59-60`; Save `disabled={saving}` only, `:431-440`). Maps to web **B24** exactly.

### Upload error / generic transport error
- Upload error (`:229-234`): if `CANCELLED`, silent; else one toast via `buildUploadErrorTitle`. Not folded into field UX.
- Generic / transport error (`:247-252`): toast `product.update.fail.title` + `.description`.

Maps to web **B25**.

---

## Part 5 — Shared mechanics

### parseServiceError
`src/lib/utils/parseServiceError.ts:40-75`. Extracts `response.data.errors`, normalizes each entry to `{ field, code, translationKey? }` (field defaulting to `null`), returns `{ errors, byField }` where `byField` keeps first-per-field and excludes `field: null` entries. `findSystemError` (`:84-86`) returns the first `field === null` entry or `null`. Used identically by:
- Create persistence failure — `UploadedProductDialog.tsx:129`
- Update persistence failure — `updateSubmitOutcome.ts:72`
- Pre-validate probe — `preValidateOutcome.ts:56`

Translation-agnostic (returns raw codes/keys; rendered via `tError` at consume sites). Maps to web's `parseProductValidationErrors`.

### Translations
Namespaces used: DIALOG (`new.product.*`, `product.update.*`), INPUT (`product.name`, `product.description`, `price.label`), VALIDATION (`form.incomplete`), ERRORS (backend violations + client-generated keys), DASHBOARD_PAGES (update screen). All keys fetched at runtime from the backend (no seed in this repo). Client-generated ERRORS keys emitted by the validator (`productValidator.ts:36-59, 106-114`): `product.name.required|too_short|too_long`, `product.description.required|too_short|too_long`, `product.category.required`, `product.price.required`, `product.image.invalid_type|too_big|duplicate`. These must be backend-seeded (see Undetermined).

### Clipboard / nav / refresh
- **Clipboard:** `expo-clipboard` `setStringAsync` on create success (`UploadedProductDialog.tsx:22, 151`); silent on failure; mobile renders a 2 s "Copied!" label (web does not). DELIBERATE-RN-DIFF / mobile-better.
- **Navigation:** `expo-router`. Create "View link" → `router.push(productRoute)` in-app (`:165`). Update back button → `router.back()` (`[productId].tsx:424`). Web uses `window.open` (new tab) for view-link. DELIBERATE-RN-DIFF.
- **Refresh:** Create success → `triggerDashboardReload()` bumps Zustand `reloadNonce` (`useDashboardProductsStore.ts:17`), remounting the owner list (RN equivalent of web `window.location.reload`). Update success → `reseedFromServer()` GET refetch resetting both snapshots, stays on page (matches web). DELIBERATE-RN-DIFF for create; MATCHES for update.

### Product-URL helpers — exact strings + every call site

There are **two** public product-URL builders in mobile, and they disagree with each other AND with web. (Verify verdict CONFIRMED all three forms.)

**Helper 1 — `getNormalizedProductUrl`** (`src/lib/utils/utils.ts:109-114`), exact string:
```
return `${withPrefix ? 'https://www.oglasino.rs' : ''}/product/${productId}/${normalizeProductName(productName)}`;
```
- Prefix domain: `https://www.oglasino.rs` (HAS `www`, `.rs` TLD).
- Slug via `normalizeProductName` (`utils.ts:121-140`): lowercase → NFD → strip diacritics → đ/Đ→d → `[^a-z0-9]+`→`-` → collapse → trim. Matches web's slug normalization.
- **Web equivalent string:** `https://oglasino.com/${locale}` (NO `www`, `.com` TLD, includes locale segment) — `../oglasino-web/.agent/audit-create-update-flow.md` B26 (verified at web source line 465).

**Helper 2 — `ShareProductButton`** (`src/components/product/ShareProductButton.tsx:29`), exact string:
```
message: `${title}\nhttps://www.oglasino.com/${locale}/product/${productId}/${productName}`,
```
- Domain: `https://www.oglasino.com` (HAS `www`, `.com` TLD) — a **third** distinct form.
- **Does NOT call `normalizeProductName`** — embeds the raw `productName` as the slug. So id 42 / "Opel Astra" shares `.../product/42/Opel Astra` (raw) while in-app nav produces `.../product/42/opel-astra` (normalized). Slug mismatch.

**`getDashboardNormalizedProductUrl`** (`utils.ts:117-119`): `/owner/dashboard/products/${productId}` — internal edit route only, not a public URL (out of scope).

**Call sites:**

`getNormalizedProductUrl`:
| Call site | file:line | withPrefix | Effect |
|---|---|---|---|
| Create success copy URL | `UploadedProductDialog.tsx:109` | `true` | `https://www.oglasino.rs/...` shown + copied — **DIVERGES** from web |
| Create success in-app route | `UploadedProductDialog.tsx:110` | `false` | relative `/product/...` — MATCHES (nav) |
| Search suggestion tap | `SearchInput.tsx:151` | `false` | relative nav — MATCHES |
| Portal product card tap | `PortalProductCard.tsx:16` | `false` | relative nav — MATCHES |
| Review product link | `ProductReview.tsx:56-58` | `false` | relative nav — MATCHES |
| Extra product card tap | `ExtraProductCard.tsx:22` | `false` | relative nav — MATCHES |
| Push-notification tap | `PushNotificationsInit.tsx:83-85` | `false` | relative nav — MATCHES |

`ShareProductButton` (all pass raw `productName`, all DIVERGE on domain + slug):
| Call site | file:line |
|---|---|
| Product detail functions | `ProductFunctions.tsx:47-51` |
| Dashboard product functions dialog | `DashboardProductFunctionsDialog.tsx:139-144` |
| User's product details | `ProductUserDetails.tsx:199-205` |

Mobile also omits the `locale` segment that web includes in the prefixed URL.

---

## Part 6 — Delta table

reCAPTCHA (B8) and analytics (B16) are excluded per the header note. One row per remaining web behavior.

| Web behavior | Mobile status | Detail / file:line |
|---|---|---|
| B1: 3 prop-less `openDialog(CREATE)` openers; `onFinish` dead | MATCHES | 4 prop-less call sites: `BottomBar.tsx:144`, `DashboardSidebar.tsx:65`, `free-zone.tsx:62`, `free-zone.tsx:118`; no `onFinish` in `AddUpdateProductDialog.tsx:25` |
| B2: Update via `router.push('/owner/products/'+id)` | MATCHES | `DashboardProductFunctionsDialog.tsx:129` pushes `/owner/dashboard/products/{id}`; param read `[productId].tsx:42-44` |
| B3: 4-step wizard, 0-indexed, images→basics→filters→upload | MATCHES | `AddUpdateProductDialog.tsx:32,123-171` |
| B4: Step1 images max5, MIME/size/dedup, single key, no Back | MATCHES | `ImageSelectionProductDialog.tsx:53,60-64`; `productValidator.ts:91-120` |
| B5: Step2 fields + keys (name 3–80, desc 10–2000, price, category) ERRORS ns | MATCHES | `BasicInfoProductDialog.tsx:170-259`; `productValidator.ts:25-89` |
| B6: Step2 pre-validate gate (Zod→probe), clean/validation/error, 5s backoff | MATCHES | `BasicInfoProductDialog.tsx:97-135,31,72-81,280`; `productService.ts:135-146` (verify CONFIRMED) |
| B7: Step3 filters dynamic (topFilters minus 'age' + per-category), no client validation | MATCHES | `MetaDataProduct.tsx:24-49`; `MetaDataProductDialog.tsx` |
| B9: Step4 auto-submit on mount, guard on success, upload then POST create | MATCHES | auto-fire `UploadedProductDialog.tsx:143-145`, guard `:79`; `POST /secure/products/create` `productService.ts:105-110`; wire fields identical (verify CONFIRMED — corrected from initial DIVERGES) |
| B10: Success screen title/2 desc/URL box+copy icon/View link/Close; copy state never rendered | DIVERGES | Mobile RENDERS "Copied!" feedback (`UploadedProductDialog.tsx:199-203`) where web does not (web dead state). View link = in-app nav; Close = no reload. Title/desc keys match `:182-190` |
| B11: Validation failure red alert, per-field + system, Go-back-and-fix/Exit | MATCHES | `UploadedProductDialog.tsx:231-259`; `resolveTargetStep` `productStepMapping.ts:32-52` |
| B12: Transport/5xx generic, Go-back→fallback step 3, Exit | MATCHES | `UploadedProductDialog.tsx:261-281`; `DEFAULT_FALLBACK_STEP=3` `productStepMapping.ts:25` |
| B13: 429 on step-4 POST as system error line, no backoff | MATCHES | `UploadedProductDialog.tsx:241-245` |
| B14: Upload error single toast (CANCELLED silent), generic surface | MATCHES | `UploadedProductDialog.tsx:120-126,217-229` |
| B15: Orphan cleanup on persistence failure before rethrow | MATCHES | `productService.ts:86-94` |
| B17: Update loads via GET, two snapshots | MATCHES | `[productId].tsx:109-135`; `productsSearchService.ts:36-38` |
| B18: Editable images/name/desc/price/currency/filters; disabled categories+region/city | MATCHES | `[productId].tsx:294-400` editable; `:331-393` disabled |
| B19: Update client validation same keys as create | MATCHES | `[productId].tsx:141-156`; `validateProduct` |
| B20: Deep-equal change detection, no-change warning toast blocks save | MATCHES | `[productId].tsx:161` |
| B21: POST update narrowed 7 fields, no oldName/oldDesc/state/moderation/category/region | MATCHES | `toUpdateWirePayload` `productWirePayload.ts:43-57`; tests `productWirePayload.test.ts:88-117` (verify CONFIRMED) |
| B22: Update success toast, reseed from GET (both snapshots), stays on page | MATCHES | `[productId].tsx:222-226,177-187` (verify CONFIRMED) |
| B23: Validation failure per-field inline + system near Save; NO otherErrors catch-all | DELIBERATE-RN-DIFF | Mobile ADDS `otherErrors` catch-all so unmapped fields are not dropped (`updateSubmitOutcome.ts:74-77`; render `[productId].tsx:403-418`) — mobile-better. Inline+system parity otherwise (verify CONFIRMED) |
| B24: 429 on update message near Save, Save stays live (no backoff) | MATCHES | `[productId].tsx:59-60,403-418,431-440` |
| B25: Update upload error toast; generic transport error toast | MATCHES | `[productId].tsx:229-234,247-252` |
| B26: Product URL `https://oglasino.com/${locale}` (no www, .com), slug normalized | DIVERGES | `getNormalizedProductUrl` → `https://www.oglasino.rs` (www, .rs, no locale) `utils.ts:114`; `ShareProductButton` → `https://www.oglasino.com` raw slug `ShareProductButton.tsx:29` (verify CONFIRMED) |
| B27: Scroll-to-first-error via DOM getElementById + scrollIntoView | DELIBERATE-RN-DIFF | RN `findNodeHandle`+`measureLayout`+`scrollTo` `[productId].tsx:78-107` (verify CONFIRMED) |

---

## For Mastermind

### (a) User-facing divergence to fix

- **HIGH — Product-URL divergence (pinned exactly).** Three different domain forms are in play, none matching web's canonical `https://oglasino.com/${locale}`:
  1. `getNormalizedProductUrl` (`src/lib/utils/utils.ts:114`) returns **`https://www.oglasino.rs`** (has `www`, `.rs` TLD, no locale segment) when `withPrefix=true`. This is the **create-success copy/share string** (`UploadedProductDialog.tsx:109`) shown in the box and written to the clipboard.
  2. `ShareProductButton` (`src/components/product/ShareProductButton.tsx:29`) hardcodes **`https://www.oglasino.com`** (has `www`, `.com` TLD) AND embeds the **raw, un-normalized `productName`** as the slug. Used at `ProductFunctions.tsx:47-51`, `DashboardProductFunctionsDialog.tsx:139-144`, `ProductUserDetails.tsx:199-205`.
  3. Web canonical: `https://oglasino.com/${locale}` (no `www`, `.com`, with locale), normalized slug.
  Impact: every copied/shared product URL from mobile points at a different domain than web; the share-button slug is also malformed (raw name with spaces). In-app navigation (all `withPrefix=false` call sites) is unaffected. Recommended: unify both helpers to web's `https://oglasino.com/${locale}` form and route share through `normalizeProductName`. (Read-only audit — not fixed here.)

### (b) Already at parity (no action)

- Entry points (B1, B2), 4-step wizard structure (B3), step 1 images (B4), step 2 basics + keys (B5), step-2 pre-validate gate + 5s backoff (B6, verify-confirmed), step 3 filters (B7), step-4 auto-submit + create POST + identical wire fields (B9, status corrected to MATCHES by verify), validation/transport/429/upload outcomes (B11–B14), orphan cleanup (B15), all update behaviors except otherErrors and scroll mechanism (B17–B22, B24, B25, verify-confirmed where applicable).

### (c) Mobile already does it better

- **Copy feedback on create success (B10).** Web computes `copied` state but never renders it (dead code / smell). Mobile renders a real 2 s "Copied!" label (`UploadedProductDialog.tsx:199-203`).
- **`otherErrors` catch-all on update (B23).** Web silently drops server field errors that map to no inline slot. Mobile collects them into `otherErrors` and renders them near Save (`updateSubmitOutcome.ts:74-77`; `[productId].tsx:403-418`), so nothing is dropped.

---

## Config-file impact

None — this is a READ-ONLY audit. No edits to `app.config.ts`, `eas.json`, `app.json`, native config, or any of the four governed docs-repo files.

---

## Undetermined

1. Whether the `https://www.oglasino.rs` (and `ShareProductButton`'s `https://www.oglasino.com`) domain is the intended mobile canonical or a bug — only code is visible; no brief confirms the correct domain. (Both diverge from web; trace and verify agree this is a real divergence.)
2. Whether the create-success copy URL should include the `locale` segment that web includes — mobile's `getNormalizedProductUrl` takes no locale parameter and omits it.
3. Whether the backend emits `system.rate_limited` (or another key) for a 429 — mobile only parses, has no synthesis logic. If the backend does not return a parseable `{errors:[...]}` body for a 429, mobile routes it to the generic-error path.
4. Whether all client-generated ERRORS-namespace keys (`product.name.required|too_short|too_long`, `product.description.*`, `product.category.required`, `product.price.required`, `product.image.invalid_type|too_big|duplicate`) and the mobile-only DIALOG/DASHBOARD keys (`new.product.success.copy.done.label`, `new.product.success.finish.label`, `new.product.success.failed.back`, `product.update.success.title/description`, `product.unchanged.title/description`, `product.update.fail.title/description`) are seeded in the backend translation table — this repo has no seed file to confirm.
5. Whether the "View link" target route resolves correctly if the freshly created product is not yet publicly visible — page-level routing was not exercised in a read-only audit.
