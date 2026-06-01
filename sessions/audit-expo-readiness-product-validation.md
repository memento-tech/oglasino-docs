# Audit — Expo release readiness: product validation

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-05-23
**Mode:** read-only audit — no code changes

---

## Section 1 — Current state in mobile

### Create-product surface

**Location:** `src/components/dialog/dialogs/product-creation/AddUpdateProductDialog.tsx`

A four-step wizard rendered inside a dialog, mirroring the web's structure:

| Step | Component | Purpose |
|------|-----------|---------|
| 1 | `ImageSelectionProductDialog.tsx` | Select up to 5 images |
| 2 | `BasicInfoProductDialog.tsx` | Name, description, category (top/sub/final), city, price, currency |
| 3 | `MetaDataProductDialog.tsx` | Filter selection (delivery, condition, availability, custom) |
| 4 | `UploadedProductDialog.tsx` | Image upload to R2, then product creation POST |

State is held locally in `AddUpdateProductDialog` as `NewProductRequestDTO`, not in a Zustand store. A progress bar shows 25%/50%/75%/100%.

### Edit-product surface

**Location:** `app/owner/dashboard/products/[productId].tsx` (348 lines)

A single-screen form. Fetches the product via `getDashboardProductDetails(productId)` → `GET /secure/products?productId=...`. All fields displayed at once. Category and location are disabled (read-only). Name, description, price, currency, images, and filters are editable. Validates via `validateProduct()` + `isMassiveChange()`. Submits via `uploadProduct()`.

### Fields presented

**Create:** name (max 80), description (max 2000), topCategory, subCategory, finalCategory, regionAndCity, price (conditional on non-free-zone), currency, images (max 5), filters.

**Edit:** same fields, but category and regionAndCity are disabled/read-only.

### Client-side validation today

**`productValidator.ts` (240 lines):** runs the following checks on name and description using client-side logic:

- Name: required, repeating chars (regex from backend config), excessive punctuation (regex), all caps (length > 3), keyword stuffing (word repeated 8+ times with length > 3), banned words (hardcoded 70-word list), promotions (regex), links (hardcoded regex), contacts (hardcoded regex).
- Description: required, banned words, spammy description (regex + keyword stuffing), links, contacts.
- Images: file size ≤ 5 MB, URI-based dedup.

**`productUpdateNameValidator.ts` (62 lines):** massive-change detection using Levenshtein distance, word overlap, and length-change ratio — thresholds hardcoded at 0.5 similarity, 40% length delta.

### Wire request shape

Both create and edit call the **same endpoint**: `POST /secure/product/addUpdate` via `addUpdateProductData()` in `productService.ts:91`.

The payload is `NewProductRequestDTO` with all fields included:

```
name, description, price, currency, free, topCategory, subCategory,
finalCategory, regionAndCity, filters, imageKeys, imagesData (stripped to undefined before send)
```

For edit, the DTO is `UpdateProductRequestDTO extends NewProductRequestDTO` which adds:

```
productState, moderationState, id, oldName, oldDescription
```

---

## Section 2 — Gap against the spec

### Error contract (Part 7 + spec)

**Position: `wrong`**

Mobile does **not** handle the `{errors: [{field, code, translationKey}]}` wire shape. The `addUpdateProductData` function (`productService.ts:87-101`) treats any non-2xx response as a generic failure, returning `null`. No response body is parsed. No per-field error codes are extracted. The `UploadedProductDialog` (step 4) shows a generic "failed" message with a single "Go back" button — it does not display which fields failed or why.

Translation keys used by mobile are in the `product.internal.*` namespace (e.g., `product.internal.name.required`, `product.internal.name.abuse`) — these are **not** the spec's `product.<field>.<code_lowercase>` pattern (e.g., `product.name.required`, `product.name.repeating_chars`). Mobile uses the `VALIDATION` namespace for rendering errors; the spec requires the `ERRORS` namespace.

### HTTP status mapping

**Position: `missing`**

Mobile does not distinguish HTTP status codes at all. `addUpdateProductData` (`productService.ts:91-101`) wraps the POST in a try/catch. Any status ≥ 300 → `return null`. Any thrown error → `return null`. There is no distinction between 400 (Jakarta), 422 (business rule), 429 (rate limit), 403 (auth), or 500 (server error). All failures are presented identically as a generic error.

### Pre-validate endpoint

**Position: `missing`**

Mobile does **not** call `POST /api/secure/products/pre-validate` at any point. The `BasicInfoProductDialog`'s "Next" handler (`onNextInternal`, line 57-67) runs only client-side validation via `validateProduct()` and then immediately advances to step 3. There is no server-side content-moderation check between steps 2 and 3. Users reach step 4 (submit) before any backend moderation runs.

### `oldName` and change detection (trust boundary)

**Position: `wrong`** — **see Section 3 for full analysis.**

`UpdateProductRequestDTO.ts:9-10` declares `oldName: string` and `oldDescription: string`. These fields are populated from the backend response when the product is loaded for editing (`getDashboardProductDetails` returns data typed as `UpdateProductRequestDTO` — see `productsSearchService.ts:43`). The client-side `isMassiveChange()` validator in `productUpdateNameValidator.ts` uses `oldProductDetails.name` (the loaded-from-server baseline) for change detection.

However, the entire `UpdateProductRequestDTO` (which includes `oldName` and `oldDescription`) is sent as the wire payload via `uploadProduct()` → `addUpdateProductData()`. The spread at `productService.ts:63-67` builds the request from `productData` without stripping `oldName` or `oldDescription`. **These client-supplied "before" values are on the wire.**

Whether the current backend endpoint (`/secure/product/addUpdate`) reads them is unknown from this repo alone — the spec says the **new** backend endpoints (`/secure/products/update`) do NOT read them (they were removed from the DTO). But mobile calls the **old** endpoint, not the new one.

### Translation coverage for ProductErrorCode

**Position: `missing`**

Mobile's translation keys are entirely in the `product.internal.*` pattern under the `VALIDATION` namespace. The spec's `ProductErrorCode` enum uses the `product.<field>.<code_lowercase>` pattern under the `ERRORS` namespace. Sampling 10 spec codes:

| Spec code | Spec translation key | Mobile equivalent | Present? |
|-----------|---------------------|-------------------|----------|
| `NAME_REQUIRED` | `product.name.required` | `product.internal.name.required` | wrong key |
| `NAME_REPEATING_CHARS` | `product.name.repeating_chars` | `product.internal.name.abuse` | wrong key |
| `NAME_BANNED_WORDS` | `product.name.banned_words` | `product.internal.name.forbidden.words` | wrong key |
| `NAME_MASSIVE_CHANGE` | `product.name.massive_change` | `product.internal.name.exessive.change` | wrong key |
| `DESCRIPTION_REQUIRED` | `product.description.required` | `product.internal.description.required` | wrong key |
| `RATE_LIMITED` | `product.system.rate_limited` | (none) | missing |
| `NOT_OWNER` | `product.system.not_owner` | (none) | missing |
| `PRODUCT_NOT_FOUND` | `product.id.not_found` | (none) | missing |
| `PRICE_REQUIRED` | `product.price.required` | (none) | missing |
| `CATEGORY_NOT_FOUND` | `product.category.not_found` | (none) | missing |

**Coverage: 0/10.** Zero keys match the spec's pattern. Mobile uses a completely different key scheme that predates the validation refactor, and the `VALIDATION` namespace is frozen per conventions Part 6 Rule 1.

Additionally, translations are fetched from the backend at runtime via `fetchNamespace()` (`i18n/fetchNamespace.ts`), so if the backend seeds the spec-pattern keys in the `ERRORS` namespace, mobile can access them — but no code in the product create/edit surface uses `TranslationNamespace.ERRORS` for validation error rendering. `BasicInfoProductDialog` uses `tValidation()` (the frozen `VALIDATION` namespace). The edit screen (`[productId].tsx`) also uses `tValidation()`.

---

## Section 3 — Trust boundary check (explicit)

### Create flow — `POST /secure/product/addUpdate`

Request body fields (from `NewProductRequestDTO` after `uploadProduct` builds the wire payload at `productService.ts:63-67`):

| Field | Source | Trust concern |
|-------|--------|---------------|
| `name` | User input (text typed by user) | None — moderated server-side |
| `description` | User input (text typed by user) | None — moderated server-side |
| `price` | User input (text typed by user) | None — validated server-side |
| `currency` | User selection from `baseSite.allowedCurrencies` | None — validated server-side |
| `free` | **Set from client-side logic** (`AddUpdateProductDialog.tsx:101` sets it to `undefined` initially; `BasicInfoProductDialog` does not set it explicitly; it may remain `undefined` or be populated from `topCategory.freeZone` — unclear) | **Flag**: spec says `free` is NOT part of the wire DTO. Mobile still includes `free: boolean` on `NewProductRequestDTO.ts:12`. Web has dropped it. |
| `topCategory` | User selection via `CategorySelector` | None — id validated via catalog lookup |
| `subCategory` | User selection via `CategorySelector` | None — id validated via catalog lookup |
| `finalCategory` | User selection via `CategorySelector` | None — id validated via catalog lookup |
| `regionAndCity` | User selection via `CitySelector` | None — validated server-side |
| `filters` | User selection via `MetaDataProduct` | None — structurally validated server-side |
| `imageKeys` | Computed from upload results (R2 keys from `uploadImages()`) | None — ownership check is a known gap |
| `imagesData` | Set to `undefined` in wire payload | None — stripped before send |

**No "before" values on the create flow.** Create is clean.

### Edit flow — `POST /secure/product/addUpdate`

Request body fields (from `UpdateProductRequestDTO extends NewProductRequestDTO`):

| Field | Source | Trust concern |
|-------|--------|---------------|
| All `NewProductRequestDTO` fields | Same as create | Same concerns as create |
| `id` | From loaded product (`getDashboardProductDetails` response) | None — product looked up server-side |
| `productState` | From loaded product response | **Flag**: client sends product state; spec says this field was removed from `UpdateProductRequestDTO` |
| `moderationState` | From loaded product response | **Flag**: client sends moderation state; spec says this field was removed |
| `oldName` | **From loaded product response** | **CRITICAL FLAG** — see below |
| `oldDescription` | **From loaded product response** | **CRITICAL FLAG** — see below |

### CRITICAL finding: `oldName` and `oldDescription`

`UpdateProductRequestDTO.ts:9-10` declares:
```typescript
oldName: string;
oldDescription: string;
```

These fields are populated from the backend response when the product is loaded for editing. The edit screen stores the loaded product as `oldProductDetails` (`[productId].tsx:49`) and uses it for client-side change detection. When the product is submitted, the entire `UpdateProductRequestDTO` is sent as the wire payload. **`oldName` and `oldDescription` are client-supplied "before" values on the wire.**

**Mitigating context:** mobile calls `POST /secure/product/addUpdate`, not the spec's new `POST /secure/products/update`. The spec documents that `oldName` and `oldDescription` were removed from the **new** `UpdateProductRequestDTO` and that MASSIVE_CHANGE detection was moved server-side. If the backend's `/secure/product/addUpdate` endpoint (the old endpoint) still reads `oldName`/`oldDescription` for change detection, then this is a **live trust-boundary violation** — a client could fabricate `oldName` to match the new name and bypass moderation.

**However**, whether mobile actually populates `oldName`/`oldDescription` on the wire payload is architecture-dependent. The `uploadProduct` function (`productService.ts:63-67`) spreads `productData` into the request. The edit screen passes `productDetails` (which is typed as `NewProductRequestDTO` in state — see `[productId].tsx:50`), but `getDashboardProductDetails` returns `UpdateProductRequestDTO` at `productsSearchService.ts:43`. The runtime type includes `oldName` and `oldDescription` from the backend response. These values flow through `uploadProduct` → `addUpdateProductData` → wire.

**Verdict:** The type declares the fields. The backend populates them in the GET response. The POST includes them on the wire. Whether the old endpoint reads them is a backend question this audit cannot answer. The field's presence on the wire DTO is a trust-boundary surface that the spec explicitly removed. This is the highest-priority finding.

### Additional "before" value fields

- `productState` (`UpdateProductRequestDTO.ts:6`): client-supplied on the wire. Spec says removed.
- `moderationState` (`UpdateProductRequestDTO.ts:7`): client-supplied on the wire. Spec says removed.

Neither is used for moderation, authorization, or state-transition decisions per the spec — but their presence on the wire is unnecessary and contradicts the spec's DTO shape.

---

## Section 4 — Image pipeline integration with create flow

The create flow holds images in memory as `ImageData[]` (with `file?: File` and `key?: string`) through all wizard steps. At step 4 (`UploadedProductDialog`), `uploadProduct()` in `productService.ts` separates new files from existing keys, uploads new files to R2 via `uploadImages()` (which requests upload tokens from `/secure/images/upload-tokens` and PUTs directly to R2), then constructs the final `imageKeys` array and POSTs the product data. This is functionally equivalent to web's Model C: images are held in memory through the wizard and uploaded at the final step. If the persist POST fails after images uploaded successfully, orphan cleanup runs via `cleanupOrphanImages(newKeys)`.

---

## Section 5 — Mobile-specific concerns

### Wizard vs single-form

Mobile uses a **4-step wizard** for create (matching web's structure). Edit uses a **single-form** (matching web's single-page edit). The spec leaves "wizard vs single form" as an open question for mobile — mobile has already answered it by mirroring web's pattern exactly.

### Error display surface

**Create flow:** client-side validation errors render **inline next to each field** via the `errorMessage` prop on `Input` and `Textarea` components (`BasicInfoProductDialog.tsx:119`, `:141`). A `form.incomplete` message renders below the action bar when `allFieldsRequired` is true. Image errors show as red italic text. Step-4 server failures show a **generic error with a triangle alert icon** and a single "Go back" button — no per-field error list, no translated error codes.

**Edit flow:** client-side validation errors render **inline next to each field** (`[productId].tsx:239`, `:254`). Massive-change errors render inline on the name field. Server-side failures show a **generic danger toast** (`product.update.fail.title` / `product.update.fail.description`). No per-field server error parsing.

### "Jump back to relevant step" on step-4 failure

The create wizard's step-4 failure UI has a single "Go back" button that calls `onBackStep` — this goes back exactly one step (to step 3, filters). There is no `productStepMapping.ts` equivalent, no `resolveTargetStep`, no routing to the step containing the earliest failed field. Web routes to the step containing the error; mobile always routes to step 3.

---

## Section 6 — Dead code

1. **`free: boolean` on `NewProductRequestDTO.ts:12`** — spec says this field was dropped from the wire DTO. Web has removed it. Mobile still declares and sends it. The `AddUpdateProductDialog.tsx:101` initializes it to `undefined`. No code in the create flow explicitly sets it to `true` or `false` based on category — it is likely always `undefined` on the wire from create. For edit, the backend response may populate it, and it flows through.

2. **`oldName: string` and `oldDescription: string` on `UpdateProductRequestDTO.ts:9-10`** — spec says these fields were removed from the backend's `UpdateProductRequestDTO`. The backend's new endpoint does not accept them. Mobile still declares and sends them. Also a trust-boundary concern (Section 3).

3. **`productState` and `moderationState` on `UpdateProductRequestDTO.ts:6-7`** — spec says these were removed from the update DTO. Mobile still includes them.

4. **`regionAndCity` on `NewProductRequestDTO.ts:14`** — spec's `NewProductRequestDTO` does not include `regionAndCity` (it's server-derived). Web has a dead `regionAndCity` field on `createProductSchema` (tracked in spec's known gaps). Mobile includes it in the DTO and sends it on the wire.

5. **Hardcoded `BANNED_WORDS` array in `productValidator.ts:7-70`** — 70+ words hardcoded in the client. The spec's content moderation chain runs server-side only; client-side checks are structural only (length, required, type). Mobile duplicates server-side content moderation with a stale, hardcoded word list that diverges from the backend's configurable banned-words lists (which are per-language and admin-editable).

6. **`containsBannedWords` parameter `bannedWords: string` (`productValidator.ts:119`)** — the `bannedWords` parameter from `RegexData` is accepted but never used. The function always checks against the hardcoded `BANNED_WORDS` array.

7. **Bug in `productValidator.ts:189,191`** — description link/contact errors are written to `errors.nameErrorKey` instead of `errors.descriptionErrorKey`. If a description contains a link or contact, the error appears on the **name** field, not the description field.

8. **`console.error` calls:**
   - `UploadedProductDialog.tsx:83` — `if (__DEV__) console.error(error)` (gated, acceptable for dev)
   - `UploadedProductDialog.tsx:104` — `console.error('Copy failed', e)` (not gated, ad-hoc debug logging)
   - `[productId].tsx:83` — `console.error(err)` (not gated, ad-hoc debug logging)

9. **`TODO` comment at `UploadedProductDialog.tsx:160`** — `// TODO` with comment about Expo navigation, no matching entry in any session summary.

---

## Section 7 — For Mastermind

### CRITICAL — `oldName` / `oldDescription` on the wire (Section 3)

Mobile's `UpdateProductRequestDTO` still carries `oldName` and `oldDescription`. These fields are populated from the backend GET response and sent back on the POST to `/secure/product/addUpdate`. The spec explicitly removed these from the update DTO and moved change-detection server-side (conventions Part 11, `decisions.md` 2026-05-13 entry). Whether the **old** backend endpoint (`/secure/product/addUpdate`) still reads them is a backend question — but the field's presence on the wire is a trust-boundary surface.

**Recommended resolution:** the mobile adoption brief must (a) remove `oldName` and `oldDescription` from `UpdateProductRequestDTO`, (b) remove the client-side `isMassiveChange()` validator entirely (server handles this), and (c) switch from the old `/secure/product/addUpdate` endpoint to the spec's separate `/secure/products/create` and `/secure/products/update` endpoints.

### Wrong endpoint

Mobile calls `POST /secure/product/addUpdate` for both create and update. The spec defines three separate endpoints: `POST /secure/products/create`, `POST /secure/products/update`, and `POST /secure/products/pre-validate`. The old combined endpoint predates the validation refactor.

**Question for Mastermind:** does the old `/secure/product/addUpdate` endpoint still exist on the backend? Was it removed during the validation refactor, or left as a deprecated alias? If removed, mobile is currently broken for product create/update against the `feature/validation-refactor` branch. If still present, mobile works against the old behavior but misses all validation-refactor improvements.

### No server error parsing

Mobile treats all server errors as generic failures (`return null`). The entire `{errors: [{field, code, translationKey}]}` error contract is unused. The adoption brief must implement a response parser (equivalent to web's `parseProductValidationErrors`) and wire per-field error display for server-returned validation errors.

### Wrong translation key pattern

Mobile uses `product.internal.*` keys under the frozen `VALIDATION` namespace. The spec uses `product.<field>.<code_lowercase>` keys under the `ERRORS` namespace. The adoption brief must switch all product validation error rendering to `TranslationNamespace.ERRORS` and use the spec's key pattern.

### Client-side content moderation should be removed

Mobile duplicates server-side content moderation (banned words, links, contacts, promotions, keyword stuffing) with a hardcoded word list that diverges from the backend's configurable, per-language lists. Per the spec: "client-side checks are structural only (length, required, type). No content moderation client-side." The adoption brief should strip all content-moderation logic from `productValidator.ts` and rely on the pre-validate endpoint and server-side validation.

### Items relevant to other audits' scopes

- **Image pipeline audit:** `ImagesImport` component, `uploadImages()`, `processImage()`, `uploadPrimitive.ts` — the full image upload pipeline. This audit only covered the integration point (Section 4).
- **Backend calls reduction audit:** mobile calls `POST /secure/product/addUpdate` (a single combined endpoint) rather than the spec's three separate endpoints. The endpoint story is entangled with the validation refactor, not the backend-calls-reduction feature.
- **General project health audit:** two ungated `console.error` calls (`UploadedProductDialog.tsx:104`, `[productId].tsx:83`) and one `TODO` without a session-summary entry (`UploadedProductDialog.tsx:160`).

### Questions whose answers would meaningfully change adoption scope

1. **Does `POST /secure/product/addUpdate` still exist on the backend's `feature/validation-refactor` branch?** If removed, mobile create/update is currently broken and the adoption brief is a full rewrite of the service layer. If present, the adoption brief can migrate incrementally.

2. **Does the backend response from `GET /secure/products?productId=...` still include `oldName` and `oldDescription` fields?** If the backend removed them from the response DTO, the `UpdateProductRequestDTO` type on mobile will have runtime `undefined` for those fields (harmless on the wire but the type is still wrong).

3. **Does the edit screen's `getDashboardProductDetails` return a shape that matches `UpdateProductRequestDTO`?** The function is typed to return `UpdateProductRequestDTO` at `productsSearchService.ts:43`, but the backend may now return a different shape after the validation refactor.

---

## Section 8 — Cleanup performed

`none needed` — read-only audit.

---

## Section 9 — Obsoleted by this session

`nothing` — read-only audit.

---

## Section 10 — Conventions check

- Part 4 (cleanliness): N/A this session.
- Part 6 (translations): touched — translation-key coverage was checked but no keys were modified.
- Part 7 (error contract): touched — verified against spec. Mobile does not implement the spec's error contract.
- Part 11 (trust boundaries): touched — explicit check performed. `oldName`/`oldDescription` on the wire is a trust-boundary surface.
- Other parts touched: Part 4b (adjacent observations) — three items flagged in Section 7 under other audits' scopes; Part 8 (architectural defaults) — mobile uses a different endpoint than the spec's platform-neutral contract.
