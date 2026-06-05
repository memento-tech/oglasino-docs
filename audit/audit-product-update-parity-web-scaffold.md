# Audit — Web Edit Page: category-label + available-filter sourcing (read-only)

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-06-01
**Type:** Phase 2 audit. READ-ONLY. No code changed.
**Feature slug:** product-update-parity
**Task (verbatim from brief):** how does the web edit page obtain (a) the three category labels and (b) the set of available filters to render — given the load endpoint returns neither — and does it actually render them populated, or quietly fall back to empty?

Scope: category-label sourcing and available-filter sourcing on the **owner edit page** (`app/[locale]/owner/products/[productId]/page.tsx`) only. Code is ground truth. Where the answer depends on the backend's serialized response (which I cannot execute or read from this repo), I say so explicitly and name the decisive test for Phase 3.

---

## Headline (read this first)

The contradiction the brief names is real, and the web code resolves it in a direction the brief did not anticipate. **The web edit page has exactly ONE source for both the category labels and the category-specific filters: the loaded product object (`productDetails`).** There is no second fetch, no catalog-by-id lookup, no router param, no Zustand category store feeding them. So whether the page renders them populated or blank is *entirely* determined by whether the load endpoint puts category objects on the response.

The web code then forces the resolution: **the load-response object carries fields that are NOT in `UpdateProductRequestDTO`, and the page demonstrably depends on them.** Specifically `productDetails.productState` (rendered as the ACTIVE/INACTIVE colored label, `page.tsx:331-336`, and the gate for the Activate/Deactivate buttons, `:570`/`:594`) is **not a field on `UpdateProductRequestDTO`** (`UpdateProductRequestDTO.ts:9-17`). If the load returned *only* the seven `UpdateProductRequestDTO` fields the backend audit lists, `productState` would be `undefined`, the colored label would be empty, and neither the Activate nor the Deactivate button would ever render. That page works in production. Therefore the **load-response DTO is strictly richer than the update-*request* DTO** — they are different shapes, and the backend audit's "the load returns only `id, name, description, price, currency, filters, imageKeys`" almost certainly describes the *request* DTO (`UpdateProductRequestDTO`), not the *load-response* DTO.

That reframes §2–§3 below: web renders category labels and the category-filter set **iff** the load-response includes the `topCategory`/`subCategory`/`finalCategory` objects (with `.labelKey` and `.filters`). The web code is built on the assumption that it does. The price/currency block is a second independent proof of that assumption (see §2).

---

## 1. What the load actually returns at runtime

- The page loads via `getDashboardProductDetails(productId)` (`page.tsx:93`) → `productsSearchService.ts:57-69`.
- `getDashboardProductDetails` calls the generic `getProductDetails(productId, 'owner')` (`:58` → `:35-51`), which does `BACKEND_API.get('/secure/products?productId=<id>')` and **returns `res.data` raw** (`:42`). `getProductDetails` has **no return-type annotation** — it returns whatever JSON the backend sent.
- `getDashboardProductDetails` then spreads that raw object and adds a hydrated `imagesData`: `return { ...data, imagesData };` (`:67-68`). **It performs no category mapping, no catalog lookup, no enrichment.** Whatever the backend put on `data.topCategory` etc. is what `productDetails.topCategory` is at runtime.
- The declared return type `Promise<ProductEditState>` (`:57`) is an **unchecked assertion**, not a transform. `ProductEditState` (`ProductEditState.ts:12-31`) declares `topCategory?`, `subCategory?`, `finalCategory?`, `productState?` as **optional** fields. TypeScript is not validating that the backend actually sends them — the `?` means "may be undefined," and at runtime they are exactly as present/absent as the wire JSON.

**Does `productDetails` have the category objects populated at runtime?** I cannot execute the endpoint from this repo, so I cannot read the wire bytes directly. But the web code answers it indirectly and decisively:

- `productDetails.productState` is **rendered and depended on** (`:335`, `:570`, `:594`) and is **not** on `UpdateProductRequestDTO`. Its presence at runtime proves the load-response is richer than `UpdateProductRequestDTO`.
- The price/currency block is gated on `productDetails.topCategory && !productDetails.topCategory.freeZone` (`:402`, `:481`). If `topCategory` were `undefined`, **the price and currency inputs would never render** — a product edit page on which you cannot edit price would be obviously, immediately broken. (See §2 for why this matters.)

**Verdict for §1:** the load-response DTO is **not** `UpdateProductRequestDTO`. It carries at least `productState` beyond those seven fields, and the page is constructed assuming it also carries the three category objects. If the backend audit examined `UpdateProductRequestDTO`, it examined the *request* type, not the *load-response* type. The TypeScript type is not "lying" about category existence so much as the two audits are describing two different DTOs.

---

## 2. The three category labels — where the rendered value comes from

- Rendered at `page.tsx:447-467`: three `disabled` `Input`s with `value={getTranslation(productDetails.topCategory?.labelKey)}` (and `subCategory?.labelKey`, `finalCategory?.labelKey`).
- `getTranslation` (`:296-300`): `if (!key) return defaultValue || ''; return t(key);` — `t` is the COMMON_SYSTEM translator (`:57`). So a present `labelKey` is translated via COMMON_SYSTEM; an absent one renders the empty string.
- **Source trace — exhaustive.** Every reference to categories on the owner edit page reads off `productDetails`:
  - `:140-142` (passed to the validator),
  - `:402` / `:481` (price/currency gate),
  - `:449` / `:456` / `:463` (the three label inputs).
  - There is **no other source.** I grepped the page and the repo: no second category fetch, no `baseSite.catalog?.categories?.find(...)` in this page, no router param carrying a category (the route is `[productId]` only), no Zustand category store. (Contrast §5: the *admin* page does a catalog-by-id lookup; the owner edit page does not.)

**Runtime behaviour, conditional on the §1 resolution:**

- **If the load-response includes `topCategory`/`subCategory`/`finalCategory` objects** (the assumption the page is built on, and which the `productState` and price/currency tells strongly support): the three labels render populated, translated through COMMON_SYSTEM from each object's `labelKey`. **Source = the load-response DTO, full category objects, directly.**
- **If it does not** (the literal reading of the backend audit applied to the load endpoint): `topCategory?.labelKey` → `undefined` → `getTranslation('')` → `''`. All three category inputs render **blank**, AND — by the same `topCategory` being undefined — the **price and currency inputs never render** (`:402`/`:481` gate is falsy). That second consequence is the tell: a page that silently drops the price field would not have survived to production unnoticed.

**Verdict for §2:** web sources the category labels **only** from `productDetails.{top,sub,final}Category.labelKey` (the load-response DTO), translated via COMMON_SYSTEM. There is no fallback source. The page is built assuming those objects are present, and the price/currency gate makes that assumption load-bearing for a field that demonstrably works. The honest conclusion is that the load-response **does** carry the category objects (so the labels render populated) — which contradicts the backend audit's claim *as applied to the load endpoint*, and is the seam Phase 3 must reconcile (see "For Mastermind"). It is *not* the case that web has some clever second source; it has none. Either the categories come down on the load DTO, or the page is broken in a way that would also kill the price field.

---

## 3. The available-filter set — where `MetaDataProduct` gets it

`MetaDataProduct` is mounted at `page.tsx:526-531` with `productData={productDetails}`, `selectedBaseSite={baseSite}`, `addCheckedFilter={false}`, no `disabled` prop (so editable, default `false`).

The available set is assembled in one `useEffect` (`MetaDataProduct.tsx:41-57`):

```
let filters = selectedBaseSite.catalog?.topFilters || [];     // (A) live base-site catalog
if (productData.topCategory?.filters)   filters = [...filters, ...productData.topCategory.filters];   // (B) load DTO
if (productData.subCategory?.filters)   filters = [...filters, ...productData.subCategory.filters];   // (B)
if (productData.finalCategory?.filters) filters = [...filters, ...productData.finalCategory.filters]; // (B)
setAvailableFilters(filters.filter((f) => f.filterKey !== 'age'));
```

Two distinct sources, traced:

- **(A) `selectedBaseSite.catalog.topFilters`** — `baseSite` comes from `useBaseSiteStore()` (`page.tsx:68`), the live Zustand base-site store; `catalog.topFilters: FilterDTO[]` (`CatalogDTO.ts`). This is always populated independently of the product. So the base-site top-filters **always** render on the edit page.
- **(B) the category objects' `.filters`** — read off `productData.topCategory.filters` etc., i.e. **off `productDetails`, the load-response DTO** — *not* off the live catalog. `MetaDataProduct` does **not** do a `catalog.categories.find(id ===)` lookup; it reads `.filters` straight from the category objects handed in via `productData`. So the category-specific filters appear **iff** the load-response carried `topCategory`/`subCategory`/`finalCategory` objects that themselves carry `.filters` (`CategoryDTO.filters: FilterDTO[]`, `CategoryDTO.ts:9`).

**Verdict for §3:** the available-filter set = **base-site `catalog.topFilters` (always, from the live store) ∪ each category object's `.filters` (from the load-response DTO, if present)**, minus the `age` filter. It is the same conditional as §2: full set if the load DTO carries the category objects; **thin set (topFilters only)** if it does not. Critically, the category filters are sourced from the **load DTO's category objects, NOT from the live catalog keyed by id** — web never keys the catalog by category id on this page (it can't; see §5). So the parity contract for mobile is "category filters come down attached to the loaded product's category objects," not "look them up in the catalog."

---

## 4. How selected values overlay the available set

- The product's selected values come from `productDetails.filters: SelectedFilterDTO[]` — present on the load DTO per **both** audits (this field is uncontested).
- Overlay happens in `MetaDataProduct` via two readers keyed on `filterKey`:
  - `getSelectedOptions(filterKey)` (`:59-61`): `productData.filters?.find(f => f.filter.filterKey === filterKey)?.options`
  - `getSelectedRangeValue(filterKey)` (`:63-65`): same, `?.selectedRangeValue`
- In `getValidFilterSelector` (`:117-166`), for each filter in `availableFilters` the matching selected value is passed into the right selector: `RANGE`/`DATE` get `value={selectedRangeValue}`, `MULTI_OPTION`/`SINGLE_OPTION` get `selectedOptions={selectedOptions}`. So **a filter that (a) appears in `availableFilters` and (b) has a value in `productData.filters` renders pre-filled.** Confirmed.
- **Edge consequence of §3:** the overlay is keyed by `filterKey` against `availableFilters`. A selected value whose filter is **not** in `availableFilters` (e.g. a category-specific selected filter when the category objects did *not* come down, so the set degraded to topFilters only) has its value sitting in `productData.filters` but **no control to render it** — it is invisible in the UI. It is *not* lost on save (it stays in the `filters` array, which `toUpdateWirePayload` passes through untouched), but the user cannot see or edit it. This only bites in the degraded-set scenario; in the full-set scenario every selected value has a control.

---

## 5. The category-id question (for mobile's benefit)

- **The owner edit page uses no category IDs and does no catalog lookup.** Grep of the page confirms it never references `topCategoryId`/`subCategoryId`/`finalCategoryId` and never calls `catalog.categories.find(...)`. It consumes whole `CategoryDTO` objects directly off `productDetails`.
- **The admin page is the contrast that proves the alternative exists.** `app/[locale]/admin/products/product/[productId]/page.tsx:52-78` loads via `getAdminProductDetails` → `ProductDetailsDTO` (the rich portal-style type), which **does** carry `topCategoryId`/`subCategoryId`/`finalCategoryId` (`ProductDetailsDTO.ts:13-15`). The admin page then **reconstructs** the category objects by keying the live catalog: `baseSite.catalog?.categories?.find(cat => cat.id === productDetails.topCategoryId)`, then walks `topCategory.subcategories` for sub, then final (`:61-74`). That is the catalog-by-id pattern.
- **Implication for mobile:** if mobile cannot get full category objects on the owner load DTO and instead wants to reconstruct them from the live catalog (the admin pattern), it needs the **category IDs**, which the owner load endpoint does **not** currently provide (the backend audit confirms no IDs on the owner edit response; the admin path only works because the *admin* endpoint returns the rich `ProductDetailsDTO` with IDs). So mobile has two coherent options, and they map onto the §1 reconciliation:
  - **(i)** the owner load endpoint returns the full category objects (what web's owner page assumes) → mobile reads them directly, no IDs needed; or
  - **(ii)** the owner load endpoint is extended to return category **IDs**, and mobile (and web) reconstruct via the admin's `catalog.categories.find(id ===)` pattern.
  - What mobile must **not** do is assume the owner load endpoint returns IDs today — it does not.

---

## Bottom line

The web edit page has **a single source** for both the three category labels and the category-specific filters: the loaded product object `productDetails`, i.e. the load-response of `GET /secure/products?productId=`. There is no second fetch, no live-catalog-by-id lookup, no router param, no store — the owner page reads `productDetails.{top,sub,final}Category.labelKey` for the labels (translated via COMMON_SYSTEM) and `productDetails.{top,sub,final}Category.filters` (unioned with the live `baseSite.catalog.topFilters`) for the available set. So the page renders category labels and the full category-filter set **iff the load-response carries those category objects**, and renders blank labels + a topFilters-only thin set if it does not. The web code itself supplies the decisive evidence that the former is true: `productDetails.productState` (rendered, and gating the Activate/Deactivate buttons) and the price/currency block (gated on `productDetails.topCategory`) are both load-bearing and both absent from `UpdateProductRequestDTO` — so the **load-response DTO is strictly richer than the update-request DTO the backend audit examined**, and the page would visibly lose the price field if `topCategory` were ever undefined. The honest resolution of the two-audit contradiction is therefore **not** "web silently falls back to empty," but "the backend's load-*response* DTO ≠ its update-*request* DTO; web populates categories and category-filters directly from the richer load-response, and Phase 3 should re-examine the actual serialized owner-load response (not `UpdateProductRequestDTO`) to confirm it carries `topCategory`/`subCategory`/`finalCategory` objects with `.labelKey` and `.filters`." Mobile's reference contract is: source category labels and category filters from the loaded product's category objects (option (i)); do **not** expect category IDs on the owner load endpoint and do **not** model a catalog-by-id lookup unless the backend is first extended to return IDs (option (ii)).

---

# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-06-01
**Task:** Phase 2 audit (read-only) — trace how the web edit page obtains the three category labels and the available-filter set, and whether it renders them populated or falls back to empty, given the load endpoint allegedly returns neither.

## Implemented

- Nothing changed on disk. Read-only Phase 2 audit.
- Traced category-label and available-filter sourcing on `app/[locale]/owner/products/[productId]/page.tsx` end to end. Established that the owner edit page has a single source for both — the load-response object `productDetails` — with no catalog lookup, second fetch, router param, or store fallback.
- Resolved the two-audit contradiction with web-side evidence: the load-response DTO is strictly richer than `UpdateProductRequestDTO` (it carries `productState`, which is rendered and gates the activate/deactivate buttons, and the page's price/currency block is gated on `productDetails.topCategory` — both load-bearing, both absent from the request DTO). The backend audit most likely examined the request DTO, not the load-response DTO.
- Documented the admin page's catalog-by-id pattern (`ProductDetailsDTO.topCategoryId` → `catalog.categories.find`) as the contrast the owner page does NOT use, and laid out the two coherent options for mobile.

## Files touched

- None (read-only). Output written to `.agent/audit-product-update-parity-web-scaffold.md` and `.agent/last-session.md`.

## Files read (evidence base)

- `app/[locale]/owner/products/[productId]/page.tsx`
- `app/[locale]/admin/products/product/[productId]/page.tsx`
- `src/lib/service/reactCalls/productsSearchService.ts`
- `src/components/owner/product/MetaDataProduct.tsx`
- `src/lib/types/product/ProductEditState.ts`, `UpdateProductRequestDTO.ts`, `ProductDetailsDTO.ts`
- `src/lib/types/catalog/CategoryDTO.ts`, `CatalogDTO.ts`, `BaseSiteDTO.ts`
- `src/lib/types/filter/FilterDTO.ts`, `SelectedFilterDTO.ts`
- grep sweeps for `catalog.categories`, `topFilters`, category-by-id lookups across `src/` and `app/`

## Tests

- Not run — read-only audit, no code changed.

## Cleanup performed

- None needed (no code changed).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change (this audit drafts no edit; Mastermind may note its completion against the product-update-parity feature)
- issues.md: no change. The seam this audit surfaces (load-response DTO vs update-request DTO) is for Mastermind's Phase 3 reconciliation, not an issues.md entry authored here (not Docs/QA; this is an audit finding for triage).

## Obsoleted by this session

- Nothing deleted. However, this audit **supersedes the conclusions of the prior `.agent/audit-product-update-parity.md`** on two specific points, and Mastermind should treat the two together:
  - That audit's §2 stated the three category labels are "pre-filled from the loaded product's `topCategory/subCategory/finalCategory` label keys" and its §2 table lists price/currency as editable. Both are correct *only if* the load-response carries the category objects — which that audit assumed from the `ProductEditState` type without tracing the wire, exactly the gap this audit was commissioned to close. This audit confirms there is no other source and makes the dependency explicit (price/currency also depend on `topCategory`).
  - Neither audit can read the backend wire from this repo; the open question (does the owner load-response actually serialize the category objects?) is handed to Phase 3, with the decisive `productState`/price tells documented above.

## Conventions check

- Part 4 (cleanliness): confirmed — no code changed.
- Part 4a (simplicity): N/A — no code added; structured evidence below.
- Part 4b (adjacent observations): one flagged (the price/currency render gate's hidden dependency on `topCategory`); surfaced to "For Mastermind."
- Part 6 (translations): N/A this session — no keys added or changed. (Noted existing usage: category labels via COMMON_SYSTEM `getTranslation`; DASHBOARD_PAGES for the surrounding labels.)
- Other parts touched: Part 11 (trust boundaries) — not re-audited here; categories/region are immutable and not sent on update, settled by the prior audit. Part 10 (feature lifecycle) — this is the Phase 2 web audit re-run to resolve a Phase 3 seam contradiction.

## Known gaps / TODOs

- The single unresolvable-from-this-repo fact: whether `GET /secure/products?productId=` serializes the `topCategory`/`subCategory`/`finalCategory` objects (with `.labelKey` and `.filters`). The web code requires them to be present for the page to function; confirming the actual serialization is a backend read, handed to Phase 3.
- Output filename: the brief explicitly named `.agent/audit-product-update-parity-web-scaffold.md` (+ `last-session.md` copy), so I used those rather than the generic `yyyy-mm-dd-<repo>-<slug>-<n>.md` template; the brief's explicit instruction overrides the Part 5 default naming.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit.
  - Considered and rejected: nothing — no code written.
  - Simplified or removed: nothing — no code touched.
- **The seam, stated for Phase 3 reconciliation (this is the whole point of the re-audit):** The two prior audits are not actually in conflict once you separate two backend DTOs. The web owner edit page consumes the *load-response* of `GET /secure/products?productId=`, which it treats as `ProductEditState` (11 fields incl. `productState` + three category objects). The backend audit reported the *update-request* shape `UpdateProductRequestDTO` (7 fields, no categories, no `productState`). **These are different DTOs.** Web's page renders `productState` and gates buttons on it, and gates price/currency on `topCategory` — fields/objects that exist on the load-response but not on the request DTO. **Recommended resolution: have the backend confirm the actual serialized type returned by the owner load endpoint (`GET /secure/products?productId=`), specifically whether it includes `topCategory`/`subCategory`/`finalCategory` objects carrying `labelKey` and `filters`.** Web's behaviour (populated labels + full filter set, or blank labels + thin filter set + missing price) is a direct function of that one fact.
- **Adjacent observation (Part 4b), out of scope, not fixed:**
  - The price/currency render block is gated on `productDetails.topCategory && !productDetails.topCategory.freeZone` (`page.tsx:402`, `:481`). This couples a core editable field (price) to the presence of the `topCategory` object on the load DTO. If categories ever stop coming down, the page silently loses the price field with no error — a quiet failure mode worth a guard or an explicit "category missing" state. Severity: medium (user-facing field silently disappears if the contract drifts). I did not fix this because the audit is read-only and it is out of scope.
- **Mobile parity headline (ties to the open `issues.md` 2026-06-01 items "filters and the category are missing from the form" and the field-by-field parity audit):** the reference contract is — category labels and category-specific filters are sourced from the **loaded product's category objects** (not a catalog lookup, not category IDs). The base-site `topFilters` come from the live catalog store and always render; the category filters ride on the loaded product. Mobile should not model a catalog-by-id lookup for the update screen unless the backend is first extended to return category IDs on the owner load endpoint (option (ii) in §5).
- **Config-file impact:** none required this session — no drafted edits to any of the four files. Nothing pending.
