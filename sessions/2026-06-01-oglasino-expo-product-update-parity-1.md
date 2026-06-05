# Audit — Expo Product Update/Edit Screen (read-only)

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-06-01
**Feature slug:** product-update-parity
**Type:** Phase 2 audit. READ-ONLY — no code changed, nothing staged.
**Task (verbatim):** Inventory the entire product UPDATE flow on mobile, field-by-field, from the code, so it can be compared against the web reference and brought to parity.

Web is the reference contract; everywhere the brief says "web does X" I treated it as a *question about mobile* and answered from the mobile code. I did not read web. Where a fact depends on the backend response shape, I say so explicitly — this audit cannot read backend, and (per the findings below) several reported symptoms hinge on what `GET /secure/products?productId=` actually returns.

---

## 1. Entry point and component tree

### Route / screen
- **`app/owner/dashboard/products/[productId].tsx`** — `export default function UpdateProductScreen()`. This is the update/edit screen. Confirmed from the code (not just the prior session record).
- The sibling `app/owner/dashboard/products/index.tsx` is the dashboard product **list** (`ProductsScreen`), not the editor.

### How the user reaches it
- **Dashboard product list → functions dialog → "Update".** `src/components/dialog/dialogs/DashboardProductFunctionsDialog.tsx:129` does `router.push(getDashboardNormalizedProductUrl(productId))`, where `getDashboardNormalizedProductUrl` (`src/lib/utils/utils.ts:140-142`) returns `/owner/dashboard/products/${productId}`. Button label `tButtons('update.product')` (`DashboardProductFunctionsDialog.tsx:132`).
- **Notifications.** `app/(portal)/(secured)/notifications.tsx:79` pushes the same `getDashboardNormalizedProductUrl(item.data.productId)` for product-related notifications.

### How the screen loads the existing product
- `productId` comes from `useLocalSearchParams` (`[productId].tsx:47-49`).
- On mount (`useEffect`, `[productId].tsx:114-140`) it calls **`getDashboardProductDetails(parseInt(productId))`** (`src/lib/services/productsSearchService.ts:36-38`).
- That service hits **`GET /secure/products?productId=<id>`** (the `dashboard` branch of `getProductDetails`, `productsSearchService.ts:13-30`).
- The return is **typed `UpdateProductRequestDTO`** (`productsSearchService.ts:36`; type at `src/lib/types/product/UpdateProductRequestDTO.ts`), which `extends NewProductRequestDTO` and adds `productState?`, `moderationState?`, `id`, `oldName`, `oldDescription`. So mobile's load type is the *request* DTO with extra read-only fields — analogous to web's `ProductEditState`-shaped type, but mobile reuses the update-request DTO directly.
- On success: `setProductDetails(data)` and `setOldProductDetails(data)` (the deep-equal baseline), `setProductErrors({})`, and — if `data.imageKeys?.length` — `setVisibleImage({ key: data.imageKeys[0] })` (`[productId].tsx:125-130`). On a falsy/empty response or a throw it sets `error = tDash('product.fail')`.

### Single flat screen or wizard?
- **Single flat screen.** The whole form is one `<ScrollView>` (`[productId].tsx:279`) rendering all sections top-to-bottom. There is no step machinery. (The multi-step wizard — `AddUpdateProductDialog` and its `*ProductDialog` children — is the **create** flow and is explicitly out of scope.)

### Component tree (render order, `[productId].tsx:278-451`)
1. Header row: product-id label + a back `Button` (`ArrowLeft`) → `router.back()` (`:280-288`).
2. Error text block when `!productDetails` (`:290-294`).
3. When `productDetails` present (`:296`):
   - `ImagesImport` (`@/components/ImagesImport`), wrapped in `imagesRef` (`:299-307`).
   - `Input` — name (`nameRef`, `:308-318`) + `AdviceText`.
   - `Textarea` — description (`descriptionRef`, `:319-334`) + `AdviceText`.
   - Region/city block (`:336-349`): an italic `Text` line + `CitySelector` (`@/components/basic/CitySelector`), **disabled**.
   - Price + currency row, gated on `topCategory && !topCategory.freeZone` (`priceRef`, `:351-386`): `Input` (price) + `Select` (currency).
   - `CategorySelector` (`@/components/basic/CategorySelector`), **disabled**, wrapped in `categoryRef` (`:387-401`) + a category error `Text`.
   - `MetaDataProduct` (`@/components/MetaDataProduct`) — the filters block (`:403-408`).
   - Near-Save error block: `systemError` + `otherErrors` (`:411-424`).
   - Action row (`:425-447`): Cancel (`outline`), Preview (blue), Save (green, spinner+disable on `saving`).

---

## 2. Fields — the full edit form, field by field

> "Initial value" = the field of the loaded `productDetails` (the `GET /secure/products?productId=` response) unless stated. State is held in one object `productDetails: NewProductRequestDTO` (`[productId].tsx:54`); edits go through `handleChange` (`:142-144`) which shallow-merges a partial.

| # | Field | Label (key) | Input type | Editable on update? | Initial value | Required |
|---|-------|-------------|-----------|---------------------|---------------|----------|
| 1 | **Images** | — (`ImagesImport`) | image picker (camera/gallery), thumbnails, remove `X` | **Editable, but display is broken — see §7** | `productDetails.imageKeys` (display) | not enforced client-side on update (`validateImages` passes `true`, but empty images yields no key) |
| 2 | **Name** | `tInput('product.name')` | `Input` (text, `maxNumOfChars={80}`) | Yes | `productDetails.name` | Yes (`required={true}`) |
| 3 | **Description** | `tInput('product.description')` | `Textarea` (`rows={8}`, `maxNumOfChars={2000}`) | Yes | `productDetails.description` | Yes (`required={true}`) |
| 4 | **Region/city** | — | italic `Text` + `CitySelector` | **No — `disabled={true}`** | `useAuthStore().user.regionAndCity` (the **USER PROFILE**, NOT the product) | `required={true}` (cosmetic; can't change) |
| 5 | **Price** | `tInput('price.label')` | `Input` (`isNumericOnly`) | Yes (only when shown) | `productDetails.price` | Yes when shown |
| 6 | **Currency** | — | `Select` | Yes (only when shown) | `productDetails.currency` (`{code, symbol}`); options from `selectedBaseSite.allowedCurrencies` | `required={true}` |
| 7 | **Top/sub/final category** | — (`CategorySelector`) | modal picker | **No — `disabled={true}`** | `productDetails.topCategory / subCategory / finalCategory` | `required` (cosmetic) |
| 8 | **Filters** | per-filter (`MetaDataProduct`) | per type: range/date/multi/single | **Yes (editable)** — no `disabled` prop passed | `productDetails.filters` (pre-selected); available set from category `.filters` + `topFilters` | not required |

Notes:
- **Price/currency are conditionally rendered**: `productDetails.topCategory && !productDetails.topCategory.freeZone` (`:351`). A free-zone product (or one whose `topCategory` is absent on load) shows **no price/currency row at all**.
- Currency `selectedValue` reads `productDetails.currency.code/.symbol` **without a guard** (`:380-383`) — if the loaded product lacks a `currency` object this throws. (Same class of bug as the now-fixed `regionAndCity` crash; flagged in §"Drift / issues noticed".)

### Per the brief's explicit-attention list

- **Images** — see the §7 deep dive. Short version: existing images are *wired* to display from `productDetails.imageKeys`, but (a) display is decoupled from edits, and (b) the keys may not be hydrated by the backend on this endpoint — see §7 and §"Web-drift checks".
- **Category (top/sub/final)** — **shown and locked.** `CategorySelector` is rendered with `disabled={true}` (`:394`) and pre-filled from `productDetails.topCategory/subCategory/finalCategory` (`:389-391`). Its visible label is `t(topCategory.labelKey) / … / …` (`CategorySelector.tsx:207-210`) — so it renders the category **only if the loaded product carries the category objects**. If absent, it shows the `category.selector.select.label` placeholder (looks "missing/empty").
- **Filters** — **shown and editable.** `MetaDataProduct` (`:403-408`) is passed **no `disabled` prop** → defaults to `disabled=false` (`MetaDataProduct.tsx:28`) → filters are editable. The set of filters shown is **derived from the category chain on the loaded product** (see §"Web-drift checks → filters" for why some can be missing). The `age` filter is explicitly excluded (`MetaDataProduct.tsx:48`). Current values pre-filled from `productData.filters` via `getSelectedOptions`/`getSelectedRangeValue` (`MetaDataProduct.tsx:51-57`).
- **Region/city** — **shown, display-only, sourced from the USER PROFILE.** `userRegionAndCity = useAuthStore((s) => s.user?.regionAndCity)` (`:45`). Rendered guarded: the italic label only renders when `userRegionAndCity?.region && userRegionAndCity.city` (`:337`); the `CitySelector` is `disabled={true}` and fed `selectedRegionAndCity={userRegionAndCity}` with a no-op `onChange` (`:342-348`). This matches the `oglasino-expo-update-product-region-crash-2` fix described in issues.md (2026-05-31).
- **Price / currency / name / description** — all editable and pre-filled from the loaded product (table rows 2,3,5,6).

---

## 3. Which fields are immutable on update

| Field | How it's locked | Code |
|-------|-----------------|------|
| **Region/city** | `disabled={true}` on `CitySelector`; `onChange={() => {}}` no-op; sourced from user profile not product | `[productId].tsx:344-347` |
| **Top/sub/final category** | `disabled={true}` on `CategorySelector`; `onChange={() => {}}` no-op | `[productId].tsx:392-394` |
| **`id`, `oldName`, `oldDescription`, `productState`, `moderationState`** | Present on the loaded `UpdateProductRequestDTO` but never rendered as inputs, and stripped from the wire by the allow-list (`toUpdateWirePayload`) | `UpdateProductRequestDTO.ts:6-10`; `productWirePayload.ts:43-57` |
| **`free` / `freeZone`** | Not an input; drives whether the price row renders; never sent on the wire | `[productId].tsx:351`; `productWirePayload.ts` (omitted) |

Quoted locks:
- `CitySelector … disabled={true}` — `[productId].tsx:347`; inside `CitySelector` every interaction is gated `if (disabled) return` / `onPress={() => !disabled && …}` / `editable={!disabled}` (`CitySelector.tsx:69, 96, 127, 138`).
- `CategorySelector … disabled={true}` — `[productId].tsx:394`; inside, the trigger is `disabled={disabled}` and the label dims (`CategorySelector.tsx:221-222`).

**Editable on update:** name, description, price, currency, and filters. Categories and region/city are display-only. This matches web's "categories locked, filters editable" model.

---

## 4. The update request payload (trust boundary — Part 11)

- **Endpoint / method:** `POST /secure/products/update` (`productService.ts:117-127`, `updateProduct`).
- **Payload builder:** **`toUpdateWirePayload(product, id, imageKeys)`** (`src/lib/services/productWirePayload.ts:43-57`), called from `uploadProduct` when `options.mode === 'update'` (`productService.ts:88`). `id` is the authenticated route param (`parseInt(productId)`), passed explicitly — not sniffed off the object.
- **Wire type:** `UpdateProductWireDTO` (`src/lib/types/product/UpdateProductWireDTO.ts`) = `Pick<NewProductRequestDTO, 'name'|'description'|'price'|'currency'|'filters'> & { id: number; imageKeys: string[] }`. Structurally cannot carry forbidden fields.

### Exact body, field by field

| Field | Source | Class |
|-------|--------|-------|
| `id` | route param (`options.id`) | (a) user context / server-validated FK |
| `name` | `product.name` | (a) user-edited |
| `description` | `product.description` | (a) user-edited |
| `price` | `product.price` | (a) user-edited |
| `currency` | `product.currency` (full `{code, symbol}` object) | (a) user-edited — **full object, not an ID** |
| `filters` | `product.filters ?? []` (array of `SelectedFilterDTO`, each with a full `filter` object + `options`/`selectedRangeValue`) | (a) user-edited — **full objects, not IDs** |
| `imageKeys` | computed by `uploadProduct` (`allKeys`): retained existing keys + newly-uploaded keys, in slot order | (b) derived client-side |

**Not sent / stripped by the allow-list:** `free`, `regionAndCity`, `topCategory`/`subCategory`/`finalCategory`, `imagesData`, `oldName`, `oldDescription`, `productState`, `moderationState`. Covered by `productWirePayload.test.ts:68` (asserts each is absent) and the doc comments at `productWirePayload.ts:32-42` and `UpdateProductWireDTO.ts:1-15`.

**Full objects vs IDs:** mobile sends **full `currency` and full `filter` objects**, not IDs. (This is a candidate web-drift item — web may send IDs. Cannot confirm web here; flagged for seam analysis.)

**Client-supplied "before"/"old" value? — NO (confirmed).** `oldName`/`oldDescription` exist on the *loaded* `UpdateProductRequestDTO` but are **excluded** from the wire by the `Pick`-based allow-list. The server compares against its own stored copy. Conventions Part 11 satisfied. (The screen's own `oldProductDetails` state, `[productId].tsx:53`, is a **client-only** deep-equal baseline for the "no changes" guard — it is never sent.)

---

## 5. Validation

- **Client validator:** `validateProduct(productData, true)` (`src/lib/validators/productValidator.ts:25`), invoked from `validateProductInternal` (`[productId].tsx:146-177`). **Structural-only** — required/length/category-presence/conditional-price/image MIME+size+dedup. No content moderation client-side (validator header comment `productValidator.ts:4-14`). Emits `ERRORS`-namespace `product.<field>.<code>` keys.
  - name: required / `too_short` (<3) / `too_long` (>80)
  - description: required / `too_short` (<10) / `too_long` (>2000)
  - category: top+sub+final all required → one `product.category.required`
  - price: required+range only when `topCategory.freeZone === false`
  - images: MIME allowlist, ≤5 MB, dedup — only over `imagesData` entries that have a `.file` (new picks); existing keys are not re-validated.
- **No pre-validate on update (confirmed).** `handleUpdate` (`[productId].tsx:194-263`) runs the local `validateProductInternal(true)` then goes straight to `uploadProduct({ mode: 'update' })` → `POST /secure/products/update`. The `preValidateProduct` service (`productService.ts:135-146`, `POST /secure/products/pre-validate`) exists but is **not called** on this screen (its comment notes it's wired into the create wizard, not update). Matches web (no pre-validate on update).
- **Change-detection guard:** `validateProductInternal(true)` also runs `deepEqualTest(productDetails, oldProductDetails)` (`[productId].tsx:166`, helper at `utils.ts:16-35`); if unchanged it shows a `product.unchanged.*` warning toast and aborts before any network call. Preview (`handlePreview`) calls `validateProductInternal(false)` — structural check, no deep-equal.
- **Content moderation for name/description:** entirely **server-side**, surfaced on submit via the thrown error → `classifyUpdateError` → per-field translation keys (see §6). The client never inspects content.

---

## 6. Success and failure handling

### Success (`[productId].tsx:227-231`)
- Success toast `product.update.success.title` / `.description` (`type: 'success'`).
- **Stays on the edit screen** and calls **`reseedFromServer()`** (`:182-192`) → re-fetches `getDashboardProductDetails` and resets both `productDetails` and `oldProductDetails` (the deep-equal baseline) so a subsequent identical save is correctly caught as "no changes". **Matches web** (stay + re-fetch to reset the diff baseline). No navigation away, no dashboard reload triggered from here.

### Per-field server errors (`:241-251`)
- `classifyUpdateError(err)` (`src/lib/services/updateSubmitOutcome.ts:69-86`) → uses `parseServiceError` + `findSystemError` (`src/lib/utils/parseServiceError.ts`).
- `validation` outcome: `mapByFieldToSlots` (`updateSubmitOutcome.ts:52-61`) maps `name/description/price/(top|sub|final)Category/images` server fields onto inline slots; category levels collapse into one `categoryErrorKey`.
- Inline render: each `…ErrorKey` rendered via `tError(...)` next to its field (`:314, 327, 361, 397`).
- **Catch-all `otherErrors`** — any field-level error not in `MAPPED_FIELDS` (`updateSubmitOutcome.ts:36-44`) renders near Save so nothing is silently dropped (`:418-422`).
- After setting errors, **`scrollToFirstError`** (`:83-112`) scrolls the first errored field (top-to-bottom order: images→name→description→price→category) into view, or to the bottom if only near-Save messages exist.
- `error` outcome (no parseable body): generic `product.update.fail.*` danger toast (`:253-257`).

### 429 / rate limit
- A `field: null` object-level error becomes `systemError` (`findSystemError`) and renders near Save (`:413-417`). **No backoff / no disabling on update** — the screen comment is explicit: "No backoff on update (N1): the message shows, Save stays live" (`:63-65`). So a 429 surfaces its translated message and Save remains pressable.

### Image-upload failure
- `upload` outcome (an `UploadError`, not persistence): keeps the toast surface via `buildUploadErrorTitle(err, tError)`; `CANCELLED` returns silently (`:234-239`). Not folded into field errors.

---

## 7. Images on update — the deep dive

### How the image component is wired (the core finding)
`[productId].tsx:300-306`:
```tsx
<ImagesImport
  images={(productDetails.imageKeys ?? []).map((key) => ({ key }))}   // DISPLAY source = imageKeys
  setImages={(imagesData) => handleChange({ imagesData: imagesData ?? [] })}  // EDIT target = imagesData
  visibleImage={visibleImage}
  setVisibleImage={setVisibleImage}
  maxNumberOfImages={5}
/>
```

The **display list is derived from `productDetails.imageKeys`**, but **`setImages` writes to `productDetails.imagesData`** — a *different field*. The two are never reconciled. Contrast the create flow (`ImageSelectionProductDialog.tsx:25,48-54`), where `images={productData.imagesData || []}` and `setImages` also writes `imagesData` — display and edit share one field, the correct pattern. **The update screen half-converted this and the two fields are decoupled.** Three consequences:

1. **On load, display works *only if* `imageKeys` is populated by the backend.** `ImagesImport` turns a `{ key }` into a URL via `publicImageUrl(key, 'card')` (`ImagesImport.tsx:127-133, 170-178`; `variants.ts:42-47` → `${CDN}/cdn-cgi/image/.../${key}`). `visibleImage` is also seeded from `imageKeys[0]` (`[productId].tsx:128-129`). So the render path itself is sound.
2. **After add/remove, the display does NOT update.** `ImagesImport.addImages`/`removeImage` call `setImages(merged)` → that writes `imagesData`. But on the next render the `images` prop is recomputed from `productDetails.imageKeys`, which **never changes**. So the user's add/remove is invisible in the thumbnail strip and main image — the editor looks frozen.
3. **Saving without touching images WIPES all images.** On load `imagesData` is `undefined` (the backend response carries `imageKeys`, never the client-only `imagesData`; `NewProductRequestDTO.ts:18-19`). `uploadProduct` builds `allKeys` solely from `productData.imagesData` (`productService.ts:51-75`); with `imagesData === undefined`, `slots` is empty → `allKeys = []` → `toUpdateWirePayload(..., [])` sends **`imageKeys: []`**, deleting every image server-side on any save that didn't re-touch images.

### Are existing images displayed on load? — depends on the backend; likely NO today
- **Per mobile code:** yes *iff* `GET /secure/products?productId=` returns a populated `imageKeys`.
- **Strong signal it is NOT populated:** the `regionAndCity` crash fix (issues.md 2026-05-31, `oglasino-expo-update-product-region-crash-2`) reportedly "hardened `productDetails.imageKeys.map` with `?? []`" — and the current code carries exactly that guard (`[productId].tsx:301`, `:128`, `:188`). A `?? []` defensive guard is added precisely because `imageKeys` was coming back **undefined** at runtime. If `imageKeys` is undefined/absent on this endpoint, the display list is empty and **no images render** — which matches Igor's on-device report ("product images are not displayed", issues.md 2026-06-01).
- **This audit cannot confirm the backend response** (mobile-only). **The single most important seam to resolve:** does `GET /secure/products?productId=` return `imageKeys` (and the category objects)? Note the *portal* product detail is a **different type** — `ProductDetailsDTO` (`src/lib/types/product/ProductDetailsDTO.ts`) carries `imageKeys` and the portal product page renders them (`app/(portal)/(public)/product/[...productData].tsx:220`), and Igor reported no portal image problem. The dashboard detail is typed `UpdateProductRequestDTO`, a separate shape. So "images render on portal but not on dashboard edit" is consistent with the dashboard endpoint not hydrating `imageKeys`.

### Add / remove / state model
- **Add:** `ImagesImport.addImages` merges `images(prop) + newOnes` and calls `setImages` (`ImagesImport.tsx:53-65`). Because the prop is the `{key}`-mapped existing keys, a successful add *would* carry existing keys + new `{file}` into `imagesData` — but see consequence #2: the display won't reflect it.
- **Remove:** `removeImage` filters by **referential identity** (`i !== img`, `ImagesImport.tsx:70`). The screen recreates the `{key}` objects every render (`.map((key) => ({ key }))`), so identities are fresh each render — removal works within one render pass but, again, the display is driven by `imageKeys` so the removal isn't reflected.
- **State model:** existing images live as `imageKeys: string[]` (display) and any user edits live as `imagesData: ImageData[]` where `ImageData = { key?, file? }` (`src/lib/types/ui/ImageData.ts`). Existing = `{key}`; newly-picked = `{file}`. The decoupling means these two never merge into a single source of truth on this screen.

### On submit — payload representation
- `uploadProduct` (`productService.ts:42-95`): walks `imagesData`, uploading `{file}` slots via `uploadImages` (R2 direct upload, `scope: 'product'`) and keeping `{key}` slots as-is, then reassembles **`allKeys` = final ordered key list** (retained + newly-uploaded). This is a **final key list**, not a diff. Passed as `imageKeys` to the wire payload.

### When do new image bytes upload to R2?
- During `handleUpdate`, inside `uploadProduct`, **before** the `POST /secure/products/update` persist call (`productService.ts:65-71` then `:88`). Progress is surfaced via `useUploadProgressStore` (`[productId].tsx:201-225`).

### Orphan cleanup for removed/failed images
- If the persist step throws after uploads succeeded, `uploadProduct` fires `cleanupOrphanImages(newKeys)` (DELETE) before rethrowing (`productService.ts:91-94`) — but only for **newly-uploaded** keys in a failed save.
- **Images the user *removed* from an existing product are not explicitly cleaned up by the client** — they simply fall out of `allKeys`; reclaiming the now-unreferenced R2 object is left to the backend (the update endpoint receiving the reduced `imageKeys`, plus the server-side sweeper backstop). No client-side DELETE for keys dropped from an existing product.

---

## Web-drift checks (mobile's actual behavior vs the web reference)

- **Region/city — MATCHES web.** Mobile reads `useAuthStore((s) => s.user?.regionAndCity)` (the USER PROFILE), display-only (`CitySelector disabled`), **not** from the product DTO; the absent case is guarded (`[productId].tsx:337` only renders the label when both region and city exist; `CitySelector` tolerates `undefined`). This is the `oglasino-expo-update-product-region-crash-2` fix. ✅
- **Images displayed on load — LIKELY DIVERGES.** Mobile *wires* display from `productDetails.imageKeys` and would show them if hydrated, but (a) the `?? []` guard strongly implies `imageKeys` arrives undefined on `GET /secure/products?productId=`, so nothing renders; and (b) even when present, edits don't reflect and an untouched save wipes images (§7). Web reportedly hydrates `imageKeys` into image state and renders/edits them. **Drift — needs backend-shape confirmation + the §7 code fixes.** ⚠️
- **Category shown and locked — MATCHES web structurally.** Mobile renders `CategorySelector` with `disabled={true}`, pre-filled from the product's `topCategory/subCategory/finalCategory`. Caveat: it renders the label only if those objects are present on the loaded DTO (same backend-hydration dependency as images). ✅ (with hydration caveat)
- **Filters shown and pre-filled — PARTIAL / LIKELY DIVERGES.** Mobile renders `MetaDataProduct` editable, pre-filled from `productData.filters`. **But the *available* filter set is built from the loaded product's category objects' `.filters`** (`MetaDataProduct.tsx:36-49`: `topFilters` + `topCategory.filters` + `subCategory.filters` + `finalCategory.filters`, minus `age`). In the *create* flow the category objects come from the live catalog (`selectedBaseSite.catalog.categories`, fully populated with `.filters`). On *update* they come from the backend product DTO — **if those category objects are slim (no nested `.filters`), most category-specific filters won't render**, leaving only `topFilters` minus `age`. This is the most likely code-level cause of "some filters missing." The `age` filter is also unconditionally excluded. **Drift — likely backend-hydration + the `age` exclusion.** ⚠️
- **Payload allow-list — MATCHES the field set, but full objects not IDs.** Mobile sends exactly `{id, name, description, price, currency, filters, imageKeys}` and strips `oldName/oldDescription/categories/regionAndCity/free/imagesData/productState/moderationState` (`productWirePayload.ts:43-57`, `UpdateProductWireDTO.ts`). Web's set is reportedly the same. **Open delta:** mobile sends the **full `currency` and full `filter` objects**, where web may send IDs — confirm in seam analysis. ✅ (field set) / ⚠️ (object-vs-ID)
- **Pre-validate on update — MATCHES web.** Mobile does **not** pre-validate on update; only the local structural check then final submit (`[productId].tsx:194-211`). ✅
- **Success behavior — MATCHES web.** Mobile stays on the edit screen and re-fetches via `reseedFromServer()` to reset the deep-equal baseline; success toast only, no navigation (`[productId].tsx:227-231, 182-192`). ✅

---

## Drift / issues noticed (out of scope — Part 4b)

1. **Untouched-save image wipe** — `[productId].tsx:300-306` + `productService.ts:51-75`. **Severity: high** (user-facing data loss). Saving an edit without touching images sends `imageKeys: []`. Root cause is the `imageKeys` (display) vs `imagesData` (edit) decoupling. *Not fixed — read-only audit.*
2. **Image editor display frozen** — same decoupling; add/remove never reflected in the UI. **Severity: high** (the headline "images not displayed" plus an unusable editor). *Not fixed — read-only audit.*
3. **Unguarded `productDetails.currency.code/.symbol`** — `[productId].tsx:380-383`. **Severity: medium** (same crash class as the fixed `regionAndCity` bug; if a loaded product lacks a `currency` object, the price/currency row throws). The render is already gated on `topCategory && !freeZone`, so free-zone products avoid it, but a non-free product missing `currency` would crash. *Not fixed — out of scope.*
4. **`age` filter unconditionally excluded** — `MetaDataProduct.tsx:48` (`f.filterKey !== 'age'`). Shared with the create flow. If web shows an age filter on edit, this is a parity gap. **Severity: low-medium.** *Flag for seam analysis.*
5. **Backend-shape dependency (the meta-issue)** — multiple reported symptoms (images, category, filters "missing") plausibly trace to `GET /secure/products?productId=` not hydrating `imageKeys` and/or slim category objects without `.filters`. **This is a cross-repo seam, not purely a mobile bug.** Must be resolved against the backend audit before scoping mobile fixes. **Severity: high (blocks parity).**
6. **Full objects on the wire** — `currency` and `filters` are sent as full objects (§4). Defensible (server can ignore extra fields), but a web-drift candidate if web sends IDs. **Severity: low.** *Flag for seam analysis.*

---

## Session summary (conventions Part 5)

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-06-01
**Task:** Read-only Phase-2 audit of the mobile product UPDATE/edit screen for the `product-update-parity` feature.

### Implemented
- Nothing implemented — this is a read-only audit. Output is this document (sections 1–7, Web-drift checks, Drift/issues).

### Files touched
- None (read-only). Files **read**: `app/owner/dashboard/products/[productId].tsx`, `app/owner/dashboard/products/index.tsx`, `src/components/ImagesImport.tsx`, `src/components/MetaDataProduct.tsx`, `src/components/basic/CategorySelector.tsx`, `src/components/basic/CitySelector.tsx`, `src/components/dialog/dialogs/DashboardProductFunctionsDialog.tsx`, `src/components/dialog/dialogs/product-creation/{AddUpdateProductDialog,ImageSelectionProductDialog}.tsx`, `src/lib/services/{productService,productsSearchService,productWirePayload,updateSubmitOutcome}.ts`, `src/lib/validators/productValidator.ts`, `src/lib/images/variants.ts`, `src/lib/types/product/{NewProductRequestDTO,UpdateProductRequestDTO,UpdateProductWireDTO,ProductDetailsDTO}.ts`, `src/lib/types/catalog/CategoryDTO.ts`, `src/lib/types/ui/ImageData.ts`, `src/lib/utils/utils.ts`.

### Tests
- Not run — no code changed. (Existing related suites observed: `productWirePayload.test.ts`, `productService.test.ts`, `productValidator.test.ts`.)

### Cleanup performed
- None needed (read-only).

### Config-file impact
- conventions.md: no change.
- decisions.md: no change.
- state.md: no change required by this audit. (The `product-update-parity` work is tracked via the 2026-06-01 issues.md "Mobile on-device UI/UX findings (batch)" update-product items; no Expo backlog table row needs flipping by a read-only audit. If Mastermind opens this as a numbered feature, that's a docs/state edit for Docs/QA — not drafted here.)
- issues.md: no change. Findings here feed Phase 3 seam analysis; the three open update-product items in the 2026-06-01 batch already cover the user-visible symptoms. New code-level findings (untouched-save image wipe, display decoupling, unguarded `currency`) are surfaced in "Drift / issues noticed" for Mastermind to triage into issues.md if desired — not authored by this audit.

### Obsoleted by this session
- Nothing.

### Conventions check
- Part 4 (cleanliness): N/A — no code changed.
- Part 4a (simplicity): N/A — no code added. See "For Mastermind" categories.
- Part 4b (adjacent observations): six items flagged in "Drift / issues noticed".
- Part 6 (translations): N/A this session.
- Part 11 (trust boundaries): confirmed — update payload allow-lists fields, sends no client "before" values; `id` from authenticated route param. Region/city sourced from authenticated user, not client-supplied product field.
- Other parts: Part 8 (routes reusable) — mobile reuses web's `/secure/products`, `/secure/products/update` endpoints; no mobile-specific route.

### Known gaps / TODOs
- This audit cannot read the backend, so the central question — does `GET /secure/products?productId=` return `imageKeys` and fully-hydrated category objects (with `.filters`)? — is unresolved here and must be answered by the backend/web audits in Phase 3 seam analysis.

### For Mastermind
- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only).
  - Considered and rejected: nothing (read-only).
  - Simplified or removed: nothing (read-only).
- **Highest-priority findings for the spec / seam analysis:**
  1. **Backend seam (blocker):** confirm the `GET /secure/products?productId=` response shape — specifically whether `imageKeys` is populated and whether the category objects carry nested `.filters`. The "images/category/filters missing" reports plausibly all trace here. Compare against what web's edit page consumes.
  2. **Mobile code fix needed regardless (high):** the image editor decouples display (`imageKeys`) from edit (`imagesData`). This causes (a) a frozen editor and (b) an untouched-save image wipe (`imageKeys: []`). The create flow's single-`imagesData` pattern is the model to converge on (hydrate `imagesData` from `imageKeys` on load; drive both display and submit from `imagesData`).
  3. **Filters availability (high for parity):** `MetaDataProduct` builds the available-filter set from the loaded product's category `.filters`. On update those come from the backend DTO, not the live catalog — if slim, filters vanish. Decide whether the edit screen should source available filters from the live catalog (`selectedBaseSite.catalog`) keyed by the product's category IDs, the way create does.
  4. **`age` filter exclusion (low-med):** confirm against web whether `age` should appear on edit.
  5. **Wire shape delta (low):** mobile sends full `currency`/`filter` objects; confirm web parity (IDs vs objects) so the spec freezes one.
  6. **Unguarded `currency` deref (medium):** `[productId].tsx:380-383` is the same crash class as the fixed `regionAndCity` bug; worth a guard when the parity fixes land.
- No config-file edits drafted; none required by a read-only audit. Closure gate: no unstated config-file dependency.
