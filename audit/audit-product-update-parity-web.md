# Audit — Web Product UPDATE/Edit Page (read-only)

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-06-01
**Type:** Phase 2 audit, READ-ONLY. No code changed.
**Feature slug:** product-update-parity
**Order number:** `<n>=1` (no prior `*-product-update-parity-*.md` in `.agent/`).
**Task:** Inventory the entire product UPDATE flow on web from the code, as the reference contract against which mobile's update screen will be fixed.

Scope note: this audits the **edit page for an existing product only**. The 4-step create wizard is out of scope and not described here. Catalog/browse filters are out of scope. The code is ground truth; comments and prior decisions were cross-checked, not trusted.

---

## 1. Entry point and component tree

**The update page is a single flat page (NOT a wizard).**

- **Route:** `app/[locale]/owner/products/[productId]/page.tsx` — the default export `ProductDetailsPage` (`'use client'`). One page, all fields visible at once. (Create is a 4-step dialog wizard; update is a single scrollable form.)
- **How the user reaches it:**
  - Owner dashboard product list at `app/[locale]/owner/products/page.tsx` renders product cards.
  - `src/components/client/product/DashboardProductCard.tsx:28` — clicking a dashboard card opens `DialogId.DASHBOARD_PRODUCT_FUNCTIONS_DIALOG` with `{ productOverview }`.
  - `src/components/popups/dialogs/DashboardProductFunctionsDialog.tsx:66-68` — the "Update product" button (`tButtons('update.product')`) calls `router.push('/owner/products/' + productOverview.id)`.
  - Helper `src/lib/utils/utils.ts:125` also builds `/owner/products/${productId}`.
- **How it loads existing data:** `useEffect` on `[user, productId]` (`page.tsx:89-120`) calls:
  - `getDashboardProductDetails(productId)` → `src/lib/service/reactCalls/productsSearchService.ts:57-69` → `GET /secure/products?productId=<id>` (owner scope, `productSearchUri('owner')` = `/secure/products`, `productsSearchService.ts:13-22,35-51`). Returns the persisted product, typed as `ProductEditState`.
  - `getUserForFirebaseUid(user.firebaseUid)` → `UserInfoDTO` (used only to source the display-only region/city — see §2).
  - On load it sets both `productDetails` and `oldProductDetails` to the same loaded object (`page.tsx:104-106`); `oldProductDetails` is the client-side baseline for the "no changes" diff (§4/§5).
  - The loaded `imageKeys: string[]` are hydrated into `imagesData: ImageData[]` by mapping each key to `{ key }` (`productsSearchService.ts:67`); the first image becomes `visibleImage` (`page.tsx:108-110`).
- **DTO that comes back:** the owner detail endpoint response is consumed directly as `ProductEditState` (`src/lib/types/product/ProductEditState.ts`): `id, name, description, price, currency, filters[], imageKeys[], productState, topCategory, subCategory, finalCategory`. (The public/portal detail endpoint returns a richer `ProductDetailsDTO`, but the owner edit page does NOT use that type — it uses the `ProductEditState` shape.)

**Component tree (render order, `page.tsx:323-650`):**

1. Header row (`:325-345`) — product id (`tDash('product.id.label')`), `productState` colored label, "Back" button (`router.back()`).
2. Images + name/description row (`:347-391`):
   - `ImagesImport` (`@/components/client/ImagesImport`, `src/components/client/ImagesImport.tsx`) — `:349-355`, `maxNumberOfImages={5}`.
   - `Input` name (`@/components/server/Input`) — `:363-373`.
   - `Textarea` description (`@/components/server/Textarea`) — `:374-389`.
3. Categories + price + region/city + filters block (`:392-535`). The layout splits into a mobile-only variant (`xl:hidden`, `:393-444`) and a desktop variant (`xl:flex`, `:469-534`) carrying the same controls:
   - `CitySelector` (`@/src/components/owner/client/CitySelector`) — region/city, `disabled={true}` (`:394-401` mobile, `:472-479` desktop).
   - Price `Input` + currency `Select` (rendered only when `topCategory && !topCategory.freeZone`) — `:402-443` mobile, `:481-521` desktop.
   - Categories card (top/sub/final) — three `Input`s, all `disabled={true}` (`:445-468`).
   - Filters card → `MetaDataProduct` (`@/src/components/owner/product/MetaDataProduct`) — `:523-533`, `addCheckedFilter={false}`.
4. System-error line, submit-failed line, no-changes line (`:536-550`).
5. Action buttons (`:551-648`): Cancel, Preview, **Save** (`handleUpdate`), then Activate/Deactivate (state-dependent), Delete.

Form pattern matches the repo convention: `useState` holding `productDetails`, with a single `handleChange(partial)` (`page.tsx:122-125`) spread-merging partials — `setProductDetails((prev) => ({ ...prev, ...data }))`. Not React Hook Form.

---

## 2. Fields — the full edit form, field by field

| Field | Label | Input | Editable on update? | Initial value source | Required |
|---|---|---|---|---|---|
| **Images** | — (drop zone) | `ImagesImport` picker (`:349`) | **Yes** — add/remove | `productDetails.imagesData`, hydrated from loaded `imageKeys` (`productsSearchService.ts:67`) | Yes (≥1, validated) |
| **Name** | `tInput('product.name')` | `Input` text, `maxNumOfChars={80}` (`:364-371`) | **Yes** | `productDetails.name` | Yes (`required={true}`) |
| **Description** | `tInput('product.description')` | `Textarea`, `rows={8}`, `maxNumOfChars={2000}` (`:375-385`) | **Yes** | `productDetails.description` | Yes (`required={true}`) |
| **Price** | `tInput('price.label')` | `Input` `isNumericOnly` (`:405-414` / `:483-493`) | **Yes** (hidden if free-zone top category) | `productDetails.price` | Yes when `!topCategory.freeZone` |
| **Currency** | `tInput('currency.select.header')` | shadcn `Select` from `baseSite.allowedCurrencies` (`:416-441` / `:494-519`) | **Yes** (hidden if free-zone) | `productDetails.currency` | Implicitly (paired with price) |
| **Filters** | `tDash('product.filters.label')` | `MetaDataProduct` (`:526-531`) | **Yes** | `productDetails.filters[]` | No (structural) |
| **Top category** | `tDash('product.top.category.label')` | `Input` `disabled` (`:447-453`) | **No — locked** | `productDetails.topCategory.labelKey` | n/a (display) |
| **Sub category** | `tDash('product.sub.category.label')` | `Input` `disabled` (`:454-460`) | **No — locked** | `productDetails.subCategory.labelKey` | n/a (display) |
| **Final category** | `tDash('product.category.label')` | `Input` `disabled` (`:461-467`) | **No — locked** | `productDetails.finalCategory.labelKey` | n/a (display) |
| **Region/City** | `CitySelector` | `disabled`, `onChange={() => {}}` (`:394-401`/`:472-479`) | **No — locked, display-only** | `userInfo.regionAndCity` (from the **user profile**, NOT the product) | shown `required` but inert |
| **Product state** | — | colored text label (`:331-336`) | **No — display only** (changed via Activate/Deactivate buttons, separate endpoints) | `productDetails.productState` | n/a |
| **Product id** | `tDash('product.id.label')` | text (`:328-330`) | **No — display only** | `productDetails.id` | n/a |

**Headline (mobile-gap) fields:**

- **Images** — existing images ARE displayed. `ImagesImport` renders one large `visibleImage` plus a scrollable thumbnail column (`ImagesImport.tsx:137-188`). Existing keys are turned into URLs by `publicImageUrl(key, 'card')` (`ImagesImport.tsx:96-99`; helper at `src/lib/images/variants.ts:41-46` → Cloudflare `/cdn-cgi/image/...` edge URL). The user can add (file picker, `:47-83`) and remove (red X per thumb, `:85-94`); no explicit reorder UI. Existing image keys come from the loaded product's `imageKeys`, hydrated to `imagesData` at `productsSearchService.ts:67`. Cap is 5 (`page.tsx:354`).
- **Category (top/sub/final)** — all three ARE shown, all three are **locked** (`disabled={true}`) with an `infoText={tDash('product.category.unchangeable')}` (`page.tsx:447-467`). Pre-filled from the loaded product's `topCategory/subCategory/finalCategory` label keys, resolved through `getTranslation` (`page.tsx:296-300`, COMMON_SYSTEM namespace).
- **Filters** — ALL filters for the product's category chain are shown, pre-filled with the product's current values, and editable. `MetaDataProduct` (`MetaDataProduct.tsx:41-57`) assembles the available filter set from `baseSite.catalog.topFilters` + the top/sub/final category `.filters` (minus the `'age'` filter), and reads current selections from `productData.filters` (`:59-65`). It is NOT passed `disabled`, so on the edit page filters are editable (the `disabled` prop defaults to `false`, `:30`). Supported filter types: `RANGE`, `DATE`, `MULTI_OPTION`, `SINGLE_OPTION` (`:121-165`). `addCheckedFilter={false}` on the edit page suppresses the green-check/X selection markers used in the create wizard.
- **Region/City** — shown via `CitySelector` but **`disabled={true}` and `onChange={() => {}}`** (`page.tsx:394-401`, `:472-479`). **Sourced from `userInfo.regionAndCity` (the authenticated user's profile), NOT from the product.** This is the key contract point for the mobile crash (issues.md 2026-05-31 `regionAndCity` undefined): web never reads region/city off the product DTO.
- **Price / currency / name / description** — all editable and pre-filled from the loaded product (see table). Price+currency are hidden entirely when the top category is free-zone (`page.tsx:402`, `:481`).

---

## 3. Which fields are immutable on update

Shown but NOT editable:

1. **Top category** — `page.tsx:447-453`: `<Input ... disabled={true} infoText={tDash('product.category.unchangeable')} onChange={() => {}}/>`.
2. **Sub category** — `page.tsx:454-460`: same `disabled={true}` + unchangeable info text.
3. **Final category** — `page.tsx:461-467`: same `disabled={true}` + unchangeable info text.
4. **Region/City** — `page.tsx:394-401` (mobile) and `:472-479` (desktop): `<CitySelector ... onChange={() => {}} disabled={true} />`. Sourced from `userInfo.regionAndCity`, not the product. `CitySelector` itself early-returns on `disabled` in its select handler (`CitySelector.tsx:58-60`).
5. **Product state** — `page.tsx:331-336`: rendered as a read-only colored label. State changes go through separate `activateProduct`/`deactivateProduct` GET endpoints (`page.tsx:570-617`), not the update payload.
6. **Product id** — `page.tsx:328-330`: display only (but `id` IS sent in the payload as the product key — see §4).

Categories and region/city are also structurally absent from the wire DTO (`UpdateProductRequestDTO`, §4), so even if a client tampered with them they would not be sent.

---

## 4. The update request payload (trust boundary — Part 11)

- **Endpoint / method:** `POST /secure/products/update` (`productService.ts:228`).
- **Payload builder:** `toUpdateWirePayload(state, imageKeys)` (`productService.ts:203-214`), called at `:225`.
- **Exact body shape** (`UpdateProductRequestDTO`, `src/lib/types/product/UpdateProductRequestDTO.ts`):

| Field | Source | Classification |
|---|---|---|
| `id` | `state.id` | user-loaded, identifies the product (server validates ownership) |
| `name` | `state.name` | (a) user-edited |
| `description` | `state.description` | (a) user-edited |
| `price` | `state.price` | (a) user-edited (optional) |
| `currency` | `state.currency` | (a) user-edited (optional) |
| `filters` | `state.filters` | (a) user-edited (optional) |
| `imageKeys` | `allKeys` from `extractAndUploadImages` | (b) derived client-side (existing keys + freshly-uploaded keys, positional) |

- **Stripped / NOT sent (allow-list confirmation):** `toUpdateWirePayload` is a positive allow-list — it constructs a brand-new object naming only the 7 fields above. Everything else on `ProductEditState` is dropped by omission. Confirmed stripped: `imagesData` (the UI working array), `productState`, `topCategory`, `subCategory`, `finalCategory`. **There is no `oldName`, `oldDescription`, `moderationState`, `regionAndCity`, or `free` field anywhere on `ProductEditState` or the wire DTO** — they do not exist on the client model, so there is nothing to strip (the prior-decision allow-list framing is satisfied; those fields simply never enter the client state). The `UpdateProductRequestDTO` doc comment (`UpdateProductRequestDTO.ts:4-8`) states categories/regionAndCity/state/moderationState are loaded server-side and `oldName`/`oldDescription` are forbidden per Part 11.
- **Client-supplied "before"/"old" value for change detection?** **None sent.** Confirmed compliant with Part 11. The only "old" value is `oldProductDetails`, held **client-side** purely to short-circuit a no-op save (`deepEqualTest`, `page.tsx:169-172`) and show the "no changes" message. It never crosses the wire. The server is responsible for its own change detection against its stored copy.
- **Mutation safety:** `toUpdateWirePayload` builds a fresh object and `extractAndUploadImages` never mutates the caller's `imagesData` (`productService.ts:136-140` splices into a fresh `allKeys` array), so the React state reference is not mutated during submit.

---

## 5. Validation

- **Client-side validator:** `validateProduct(...)` (`src/lib/validators/productValidator.ts:13-90`), called via `validateProductInternal(withDeepEqual)` in `page.tsx:133-176`. Structural-only (Zod `createProductSchema`/`priceSchema` from `productSchemas`): required + length for name/description, required categories, price required when `!topCategory.freeZone`, and image checks (MIME type, ≤5 MB, duplicate-by-SHA-256) via `validateImagesData` (`productValidator.ts:106-137`). Emits ERRORS-namespace keys `product.<field>.required|too_short|too_long`, `product.image.invalid_type|too_big|duplicate`, `product.category.required`, `product.price.required`.
- **Note:** on update, categories are locked/pre-filled so the category-required branch effectively never triggers; name/description/price/images are the live client checks. `validateProductInternal` only surfaces `name|description|images|price` errors (`page.tsx:147-165`) and scrolls to the first offending field.
- **No-changes gate:** when `withDeepEqual` is true (the Save path), `deepEqualTest(productDetails, oldProductDetails, [])` short-circuits with `showNoChangesMessage` if nothing changed (`page.tsx:169-172`). Preview passes `withDeepEqual=false` (`page.tsx:289`), so it never blocks on "no changes".
- **Pre-validate on update?** **No.** `preValidateProductBasics` (`POST /secure/products/pre-validate`, `productService.ts:259`) is a create-wizard Step-2 UX optimization only; the update page does not import or call it. Update has exactly one server round-trip: the final `POST /secure/products/update`.
- **Content moderation on name/description:** entirely server-side. The client never checks banned words/spam/links. The server returns per-field codes (e.g. `NAME_BANNED_WORDS` → `product.name.banned_words`) on the update response, surfaced through the same error pipeline (§6). Matches Part 8 ("server is the single source of truth for content moderation") and the create flow's contract.

---

## 6. Success and failure handling

- **Success** (`page.tsx:203-237`): `result.type === 'success'` → success toast (`tDash('product.update.success.title'/'.description')`), then a **re-fetch** of the persisted product (`getDashboardProductDetails(productData.id)`) to reset both `oldProductDetails` and `productDetails` to the post-save state (so subsequent saves diff against saved state, not the page-load snapshot), and resets `visibleImage`. If the re-fetch returns null/throws → a warning toast (`tDash('product.update.refresh.fail')`); the save itself is still treated as successful. **No navigation away** — the user stays on the edit page.
- **Per-field server errors** (`page.tsx:238-258`): `result.type === 'validation'` → `setProductErrors(result.errors.byField)`, fires a `form_submit_failed` analytics event per error, and scrolls to the first offending field (name → description → price → system-error). Errors are rendered inline via `tErrors(productErrors[field])` (e.g. name `:369`, description `:381`, price `:412/:491`, images `:356-360`, system `:536-542`). Parse helper: `parseProductValidationErrors` (`src/lib/utils/parseProductValidationErrors.ts:31-45`) — first-error-per-field wins; `field:null` errors collapse under `SYSTEM_ERROR_KEY` (`'__system'`).
- **Generic failure** (`page.tsx:259-261`): `result.type === 'error'` → `setSubmitFailed(true)` renders `tDash('product.update.fail.message')` (`:543-545`).
- **Image-upload failure** (`page.tsx:266-281`): `updateProductData` re-throws `UploadError`; the page catches it and shows ONE localized toast via `buildUploadErrorTitle(err, tErrors)` (skipping `CANCELLED`); non-`UploadError` throws fall through to the generic update-fail toast.
- **429 handling:** `productService.ts` treats 400/403/422/429/500 as parseable error bodies (`PARSEABLE_ERROR_STATUSES`, `:42`). For 429, `parseProductErrorsForStatus` (`:61-74`) synthesizes a `SYSTEM_ERROR_KEY` → `system.rate_limited` entry if the body lacked a `field:null` rate-limit error (the edge worker can emit a 429 without Spring's unified shape). On the update page this surfaces as the system-error line (`:536-542`).

---

## 7. Images on update — the deep dive

- **On load:** the loaded product's `imageKeys: string[]` (full R2 keys with prefix, per the Phase-4 pipeline) are mapped to `imagesData: ImageData[]` as `{ key }` entries (`productsSearchService.ts:64-68`). `ImagesImport` renders them: keys become URLs via `publicImageUrl(key, 'card')` (`ImagesImport.tsx:96-99`) → `${CDN_BASE}/cdn-cgi/image/width=400,height=300,fit=cover,.../{key}` (`variants.ts:41-46`). Large preview is `visibleImage`; the rest are a thumbnail column (`ImagesImport.tsx:137-188`).
- **State model:** a single `imagesData: ImageData[]` array where each entry is **either** an existing image (`{ key }`, no `file`) **or** a newly-picked image (`{ file }`, no `key`). Add appends `{ file }` entries (`ImagesImport.tsx:79-81`); remove filters the array by reference (`:85-94`). The picker also auto-drops entries whose `<img>` fails to load (`onError`, `:171-182`).
- **Remove / add:** yes to both. Removing an existing image just drops its `{ key }` from `imagesData`, so that key is simply absent from the next submitted `imageKeys` (no explicit delete call at remove-time). HEIC/HEIF picks are converted to JPEG at pick time (`preparePickerFiles`, `ImagesImport.tsx:65-75`). Cap 5 enforced at pick time (`:53-60`).
- **On submit — payload representation:** the **final positional key list**, not a diff. `extractAndUploadImages` (`productService.ts:96-141`) walks `imagesData`: entries with `.key` are kept as-is; entries with `.file` are uploaded and their returned keys spliced back into the same positions (`:135-140`). The result `allKeys` (existing + newly uploaded, in order) is written to `imageKeys` on the wire (`toUpdateWirePayload`, `:213/:225`). It returns `newKeys` = only the freshly-uploaded subset for orphan cleanup. So update sends a complete replacement list; the server diffs against its stored set.
- **When new bytes upload to R2:** during `updateProductData`, before the `POST .../update` call — `extractAndUploadImages` → `uploadImages(files, 'product', ...)` (`uploadImages.ts`): client processes each file (resize/re-encode/HEIC→JPEG), `POST /api/secure/images/upload-tokens` for per-file `{token, key, uploadUrl}`, then **PUTs raw bytes directly to R2 via the Worker** (`x-upload-token` + `Content-Type`). This is the direct-to-R2 pipeline (Part 8) — bytes never proxy through the backend. Upload failures fail-fast as `UploadError`. Progress is driven into `useUploadProgressStore` (`page.tsx:186-201`) which `ImagesImport` subscribes to for per-thumbnail status overlays.
- **Orphans:** if `POST .../update` fails (non-2xx body, thrown error, or network failure after upload), `updateProductData` fires `cleanupOrphanImages(newKeys)` (`productService.ts:234,239-241,244-246`) — **only the newly-uploaded keys**, never existing keys. `cleanupOrphanImages` (`uploadImages.ts:401-413`) issues best-effort `DELETE /secure/images/{key}` per key (`Promise.allSettled`; logs failures; the backend sweeper is the backstop). **Images the user *removed* from an existing product are NOT proactively deleted at submit time** — the removed key is just omitted from `imageKeys`; the now-unreferenced R2 object is left for the backend sweeper. (Client-side orphan cleanup covers only *newly-uploaded-but-not-persisted* keys, not *previously-persisted-now-dereferenced* keys.)

---

## Drift / issues noticed (out of scope)

These are observations for Mastermind's seam analysis vs. the expo audit — flagged, not fixed (read-only audit).

1. **Region/city source is the user profile, not the product (the mobile-crash contract).** Web sources region/city from `userInfo.regionAndCity` (`page.tsx:394-401`, `:472-479`), display-only and guarded by `disabled`. The mobile crash logged in `issues.md` (2026-05-31, `regionAndCity` undefined on `productDetails`) was caused by mobile reading it off the product DTO; the resolution note already says mobile was re-pointed to `useAuthStore().user.regionAndCity` to match web. This audit **confirms** web's behavior is exactly that. Severity: informational — the reference contract is "region/city = authenticated user's profile, display-only, render nothing if absent."

2. **`getDashboardProductDetails` return type vs. error path (low).** `productsSearchService.ts:57` types the return as `Promise<ProductEditState>`, but on a null backend response it returns `data` (which is `null`) — `:59 if (!data) return data;`. The caller guards (`page.tsx:96-99 if (!data) ...`), so it's safe, but the type signature lies about nullability (no `| null`). Could mislead a future reader. Severity: low.

3. **Currency `Select` round-trips the whole `CurrencyDTO` through `JSON.stringify`/`JSON.parse` as the option value** (`page.tsx:417-422`, `:495-500`). Works, but it's an unusual pattern (value is a serialized object, parsed in `onValueChange`). Not a bug; noting for the parity comparison in case mobile models currency differently. Severity: low.

4. **Removed-existing-image R2 objects rely solely on the backend sweeper.** When a user removes an image that was already persisted and then saves, the client sends a shorter `imageKeys` and does **not** proactively `DELETE` the dereferenced key (client cleanup targets only `newKeys`). The object lingers until the backend sweeper. This is consistent with the documented contract (client cleanup = orphans from *this* upload only), but worth confirming the backend dereference-and-sweep actually reclaims it. Severity: low (cost/storage, not user-facing). Out of scope.

5. **`page.tsx` carries `console.error` calls** at `:112`, `:232`, `:277`. These predate this audit and sit on genuine error paths (fetch failure, refresh failure, non-UploadError submit failure). Per conventions Part 4 ad-hoc `console.*` is discouraged in favor of the project logger (`logServiceError`); the service layer already uses it, but these page-level catches do not. Severity: low. Not fixed — read-only audit and out of scope.

---

# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-06-01
**Task:** Phase 2 audit (read-only) — inventory the entire web product UPDATE/edit flow from the code as the reference contract for fixing mobile drift.

## Implemented

- Nothing changed on disk. This is a read-only Phase 2 audit.
- Produced sections 1–7 plus a drift section documenting the web product update page: single flat page at `app/[locale]/owner/products/[productId]/page.tsx`; loads via `GET /secure/products?productId=` (consumed as `ProductEditState`); region/city sourced from the user profile and display-only; categories locked; filters + images + name/description/price/currency editable; payload `POST /secure/products/update` allow-listed to 7 fields via `toUpdateWirePayload`; no client-supplied "old" values (Part 11 compliant); images modeled as a positional `imagesData` array (existing keys vs new files) submitted as a full key list, with direct-to-R2 upload and `newKeys`-only orphan cleanup.

## Files touched

- None (read-only). Output written to `.agent/audit-product-update-parity.md` and `.agent/last-session.md`.

## Files read (evidence base)

- `app/[locale]/owner/products/[productId]/page.tsx`, `app/[locale]/owner/products/page.tsx`
- `src/components/client/product/DashboardProductCard.tsx`, `src/components/popups/dialogs/DashboardProductFunctionsDialog.tsx`
- `src/lib/service/reactCalls/productsSearchService.ts`, `src/lib/service/reactCalls/productService.ts`
- `src/lib/types/product/ProductEditState.ts`, `UpdateProductRequestDTO.ts`, `ProductDetailsDTO.ts`
- `src/components/owner/product/MetaDataProduct.tsx`, `src/components/client/ImagesImport.tsx`, `src/components/owner/client/CitySelector.tsx`
- `src/lib/validators/productValidator.ts`, `src/lib/utils/parseProductValidationErrors.ts`
- `src/lib/images/variants.ts`, `src/lib/images/uploadImages.ts`

## Tests

- Not run — read-only audit, no code changed.

## Cleanup performed

- None needed (no code changed).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change (Mastermind may wish to note the audit's completion against the product-update-parity feature, but this audit drafts no edit)
- issues.md: no change. The §"Drift / issues noticed" items are surfaced here for Mastermind's Phase 3 seam analysis per Part 10; I did not author issues.md entries (not Docs/QA, and these are audit findings for triage).

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code changed.
- Part 4a (simplicity): N/A — no code added; see structured evidence below.
- Part 4b (adjacent observations): five observations flagged in "Drift / issues noticed" and surfaced to "For Mastermind."
- Part 6 (translations): N/A this session — no keys added or changed. (Documented the existing key usage: ERRORS `product.*`, DASHBOARD_PAGES `product.*`, INPUT `product.*`/`price.*`/`currency.*`, BUTTONS, DIALOG.)
- Other parts touched: Part 11 (trust boundaries) — confirmed: update payload sends no client "old" value; Part 8 (direct-to-R2, server-side moderation) — confirmed.

## Known gaps / TODOs

- None. The audit covers the full update surface requested. Server-side behavior (how the backend diffs/moderates/dereferences images) is the backend audit's scope, not web.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit.
  - Considered and rejected: nothing — no code written.
  - Simplified or removed: nothing — no code touched.
- **Reference-contract headline for the expo seam analysis:** region/city on web update = the authenticated user's profile (`userInfo.regionAndCity`), display-only and disabled, NOT read from the product DTO. Categories are locked. Filters, images, name, description, price, currency are editable. This directly answers the three open mobile items in `issues.md` (2026-06-01 batch: "images not displayed", "filters and category missing", "field-by-field parity audit") and aligns with the already-resolved 2026-05-31 mobile `regionAndCity` crash fix.
- **Adjacent observations (Part 4b), all out of scope, not fixed:**
  - `getDashboardProductDetails` return type omits `| null` while returning null on failure (`productsSearchService.ts:57-59`) — file:as cited — severity low (misleading type, caller is guarded).
  - Currency `Select` serializes the whole `CurrencyDTO` as the option value (`page.tsx:417-422`) — severity low (works; unusual).
  - Removed-existing images are not proactively DELETE'd on save; rely on the backend sweeper (`productService.ts` cleanup targets `newKeys` only) — severity low.
  - Three page-level `console.error` calls (`page.tsx:112,232,277`) instead of the project logger — severity low.
  - All four: "I did not fix these because the audit is read-only and they are out of scope."
- **Config-file impact:** none required this session — no drafted edits to any of the four files. Nothing pending.
