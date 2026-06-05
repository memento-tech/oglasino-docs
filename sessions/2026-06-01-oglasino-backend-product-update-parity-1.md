# Audit — `GET /secure/products?productId=` Owner Product-Detail Response Shape

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-06-01
**Type:** Phase 2 audit — READ-ONLY. No code changes.
**Feature slug:** product-update-parity
**Task (verbatim from brief):** Document the shape and content of the response body from the owner/dashboard-scoped single-product fetch (`GET /secure/products?productId=<id>`) — every field, and which nested objects are populated, with the headline questions being `imageKeys` population and category `.filters`.

---

## 1. The endpoint

**Controller:** `DashboardProductController` (`src/main/java/com/memento/tech/oglasino/controller/DashboardProductController.java:53-67`).

**Exact mapping:** class-level `@RequestMapping("/api/secure/products")` (line 43) + method-level `@GetMapping` with no sub-path (line 53). So the wire path is **`GET /api/secure/products?productId=<id>`** — the clients' `/secure/products` with the `/api` prefix. Single required query param `@RequestParam @NotNull Long productId` (line 55). Confirmed: this is the owner-scoped single-product fetch — it is the only `@GetMapping` with no sub-path on the secured products controller, distinct from the public detail (§6) and from list/search (`@PostMapping` `getAllProducts`, line 69).

**Declared response DTO:** `ResponseEntity<UpdateProductRequestDTO>` (line 54). The response DTO is the *same class used as the update request body* — `UpdateProductRequestDTO`.

**Auth / ownership gating (lines 56-66):**
1. `productFacade.getProductForUpdate(productId)` → `null` if the product doesn't exist or is `DELETED` (`DefaultProductFacade.getProductForUpdate` / `isValidProduct`, lines 39-51). Null ⇒ throws `ProductValidationException("id", PRODUCT_NOT_FOUND)`.
2. Ownership: `currentUserService.getCurrentUserIdStrict()` (server-derived identity per Part 11) compared to `product.getOwner().getId()`. Mismatch ⇒ throws `ProductValidationException(SystemErrorCode.NOT_OWNER)`. Trust boundary is correct — owner identity is read from the security context, not the request.
3. On success: `productFacade.toUpdateData(product)` → `modelMapper.map(product, UpdateProductRequestDTO.class)` (line 66; `DefaultProductFacade.toUpdateData`, lines 44-47).

The ModelMapper has a dedicated registered `Converter<Product, UpdateProductRequestDTO>` — `UpdateProductRequestConverter` (registered in `MapperConfiguration.java:59`). That converter, not default field mapping, builds this response. It is the source of truth for what's populated.

---

## 2. The response DTO — full field list

`UpdateProductRequestDTO` (`src/main/java/com/memento/tech/oglasino/dto/UpdateProductRequestDTO.java`). **Seven fields, total:**

| Field | Type | Populated on owner fetch? | Source |
| --- | --- | --- | --- |
| `id` | `Long` | yes | `source.getId()` (converter:58) |
| `name` | `String` | yes | resolved translation, preferred-language-first (converter:64, `resolveField`) |
| `description` | `String` | yes | resolved translation (converter:65) |
| `price` | `BigDecimal` | yes | `source.getPrice()` (converter:59) |
| `currency` | `CurrencyDTO` (record `{id, code, symbol}`) | yes | built from `source.getCurrency()` (converter:52-56, 60) |
| `filters` | `List<SelectedFilterDTO>` | yes — see §5 | from `source.getFilterValues()` (converter:33-50, 61) |
| `imageKeys` | `Set<String>` | yes — see §3 | `source.getImageKeys()` (converter:62) |

**Brief's checklist — presence/absence on this DTO:**

- `id` — **present** (`Long`).
- `name` — **present** (`String`).
- `description` — **present** (`String`).
- `price` — **present** (`BigDecimal`).
- `currency` — **present** (`CurrencyDTO` object, not a bare code).
- `imageKeys` — **present** (`Set<String>`). See §3.
- `topCategory` / `subCategory` / `finalCategory` — **ABSENT.** No such fields on the DTO, in any form (not as objects, not as IDs, not as labels). The converter never sets them.
- `filters` — **present** (`List<SelectedFilterDTO>`) — these are the product's *selected* filter values, not a category's available-filter catalog. See §5.
- `productState` — **ABSENT.** Not a field on the DTO.
- `moderationState` — **ABSENT.** Not a field on the DTO.
- `regionAndCity` — **ABSENT.** No region/city field in any form.
- `free` / `freeZone` — **ABSENT.** No such field.

The DTO's own Javadoc (lines 13-19) states this by design: *"Categories, region/city, and baseSite are immutable post-creation and are therefore absent from this DTO — they are read from the persisted entity."* The DTO carries only the mutable update surface. (`free` is derived from category/price server-side at create/update; it is not part of the editable surface and is not returned here.)

---

## 3. `imageKeys` — IS IT POPULATED?  (headline)

**Yes. `imageKeys` is a field on the response DTO and it is populated directly from the persisted product.**

- It is declared on `UpdateProductRequestDTO` as `Set<String> imageKeys` (DTO:39).
- The converter sets it unconditionally from the entity: `destination.setImageKeys(source.getImageKeys());` (`UpdateProductRequestConverter.java:62`).
- `Product.imageKeys` is an `@ElementCollection` of the full R2 keys (entity `Product.java:85-88`; "R2 keys for product photos, full path including prefix").

**Plain statement:** on a normal owner fetch of a product that HAS images, the response **carries the image keys verbatim** (the same `public/products/<uuid>.jpg` strings stored at create/update). They are not nulled or slimmed. If a client edit screen shows no images, that is **not** because this endpoint omits `imageKeys`.

---

## 4. Category objects — are they hydrated?

**There are no category objects on this response at all.** `topCategory`, `subCategory`, and `finalCategory` are **absent from `UpdateProductRequestDTO`** entirely (§2) — neither as full objects, nor as IDs, nor as labels. The converter reads `source.getTopCategory()` / `getSubCategory()` / `getCategory()` **never** (they exist on the `Product` entity — `Product.java:36-44` — but the converter ignores them).

Consequently the brief's sub-question — *does each returned category object carry its nested `filters` collection?* — **does not apply**: there is no category object on this response to carry a `.filters` collection. The category-and-its-available-filters catalog is simply not part of this endpoint's contract. It is omitted by design because category is immutable post-create (DTO Javadoc, lines 13-19).

---

## 5. `filters` — the product's selected filter values

**Yes — the product's own selected filter values ARE returned, as reasonably full objects.**

`filters` is `List<SelectedFilterDTO>`, built from `source.getFilterValues()` (the product's persisted `ProductFilterValue` rows; `Product.java:71-77`). Each `SelectedFilterDTO` (`dto/SelectedFilterDTO.java`) carries:

- **`filter`** — a `FilterDTO` (the catalog filter definition), via `modelMapper.map(productFilter.getFilter(), FilterDTO.class)` (converter:37). `FilterDTO` (`dto/FilterDTO.java`) = `{id, basic, order, labelKey, iconId, filterType, filterKey, options:List<FilterOptionDTO>, filterRange:FilterRangeDTO}`. **Caveat (see "Drift" below):** `productFilter.getFilter()` returns a `Filter` *entity*, but the registered `FilterConverter` is `Converter<CategoryFilter, FilterDTO>` (`converter/FilterConverter.java:18`) — type mismatch, so that converter does **not** run here. ModelMapper falls back to default field mapping `Filter → FilterDTO`. Default mapping populates the name-matching fields — `id, labelKey, iconId, filterType, filterKey, options, filterRange` (the `Filter` entity has all of these, including its full `options` list, `Filter.java:40`). The two FilterDTO fields with **no** counterpart on `Filter` — `basic` and `order` (those live on `CategoryFilter`) — are left at defaults (`false` / `0`). So the nested `filter.options` (the full catalog option list for that filter) **is** present, but `basic`/`order` are unreliable on this path.
- **`options`** — `List<FilterOptionDTO>`, the option(s) the owner actually selected, mapped from `productFilter.getFilterOptions()` (converter:38-41). Each `FilterOptionDTO` = `{id, labelKey}`.
- **`selectedRangeValue`** — `Long`, the selected range value for RANGE-type filters (converter:46).

So per selected filter the response gives: the catalog filter definition (incl. its available options) + the owner's selected option(s) + any selected range value. This is **full enough to re-render the controls for filters the product already has values for.** What it does **not** give is the set of *unselected* filters available for the category — because (per §4) there is no category object and no category-filter catalog on this response. A form that needs "all filters for this category, pre-filled where selected" cannot reconstruct the unselected ones from this endpoint alone.

---

## 6. Compare to the public/portal product detail

**Different DTO, different mapper, different data source.**

- **Owner fetch (this audit):** `UpdateProductRequestDTO`, built by `UpdateProductRequestConverter` **from the `Product` JPA entity** (Postgres).
- **Public/portal detail:** `ProductDetailsDTO` (`dto/ProductDetailsDTO.java`), built by `ProductDetailsConverter` (`elasticsearch/converters/ProductDetailsConverter.java`) **from a `ProductDocument` (Elasticsearch)**, not from the entity. (Public single-product detail is served off the ES read model via `ProductsSearchFacade`; `PublicProductController` only carries seen/views/count.)

Specifically, for the two headline concerns:

- **`imageKeys`:** **both** populate it. Public: `destination.setImageKeys(source.getImageKeys())` from the ES document (`ProductDetailsConverter.java:70`). Owner: from the entity (§3). Both return the same `Set<String>` of R2 keys. So images are available on both responses — the public page rendering images "fine" is consistent with the owner fetch *also* carrying the keys.
- **Categories:** Public returns **bare IDs** — `topCategoryId` / `subCategoryId` / `finalCategoryId` as `long` (`ProductDetailsDTO.java:15-17`; converter:72-74). Owner returns **nothing** for category (§4). Neither returns full category objects, and **neither carries a category's nested available-filters catalog.**
- **Filters:** Public exposes a *reference/display* projection — `availability`, `deliveries` (`ProductFilterDTO`) and `otherFilters` (`List<ProductFilterDTO>`), derived from ES `filterReferences` (converter:42-56, 75-80). Owner exposes the product's *selected* values as `List<SelectedFilterDTO>` (§5). These are two different filter shapes for two different purposes (public read-only display vs. owner edit form).

**Net:** the public detail does not populate anything the owner fetch is "missing" for `imageKeys` — both carry the keys. The only category information the public detail adds over the owner fetch is the three category **IDs**; neither endpoint returns the per-category filter catalog needed to render an empty editable filter form.

---

## Bottom line

The owner fetch `GET /api/secure/products?productId=` returns `UpdateProductRequestDTO`, and it **does populate `imageKeys`** (verbatim R2 keys, straight from the persisted product — `UpdateProductRequestConverter.java:62`), so missing images on an edit screen are not caused by this endpoint dropping the keys. Its response **carries no category objects of any kind** — `topCategory`/`subCategory`/`finalCategory` (and region/city, productState, moderationState, free) are absent by design (immutable post-create), so the "does each category object carry nested `.filters`" question is moot: there is no category object on this response, and therefore no per-category available-filters catalog. The product's **selected** filter values *are* returned (`filters: List<SelectedFilterDTO>`, with the catalog filter definition + selected options + range), but the **unselected** filters available for the category are not, because the category-filter catalog isn't part of this contract.

---

## Drift / issues noticed (flagged, not fixed — READ-ONLY)

1. **`FilterConverter` type-mismatch on the owner-fetch filter path (low).** `UpdateProductRequestConverter.java:37` calls `modelMapper.map(productFilter.getFilter(), FilterDTO.class)` where `productFilter.getFilter()` is a `Filter` entity, but the only registered `FilterDTO` converter is `FilterConverter implements Converter<CategoryFilter, FilterDTO>` (`converter/FilterConverter.java:18`). The converter does not fire for `Filter → FilterDTO`; ModelMapper uses default field mapping. Effect: `FilterDTO.basic` and `FilterDTO.order` (which only exist on `CategoryFilter`, not `Filter`) are left at `false`/`0` on the selected-filter objects this endpoint returns, and the catalog option ordering logic in `FilterConverter` (the `useDefaultOptionsOrder` sort, lines 42-51) is bypassed — options come back in default mapping order. Whether that matters depends on how clients render already-selected filters; it is not a crash and not a security issue. **Not fixed — out of scope (read-only audit).**

2. **Endpoint contract vs. an editable filter/category form (informational, not a bug).** If the product-update-parity goal is for a client edit screen to show the category and the full filter set (selected + unselected) pre-filled, this endpoint alone cannot supply it — category and the category-filter catalog are deliberately absent. Whatever populates the category label and the available-filter list on web must come from a *separate* catalog/category endpoint keyed off data the client already holds, not from this response. This is a seam observation for Phase 3, not a defect in this endpoint. **Not fixed — design/seam decision, out of audit scope.**

---

## Cleanup performed

- none needed (read-only audit; no code touched).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change.
- issues.md: no change. (The two items in "Drift / issues noticed" are surfaced here for Mastermind to triage in Phase 3; per Part 3 I do not write to `issues.md`. No config-file edit is required by this read-only audit.)

## Obsoleted by this session

- nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no files created except this summary pair; nothing to clean.
- Part 4a (simplicity): N/A — no code added; see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): two observations flagged in "Drift / issues noticed" and "For Mastermind."
- Part 6 (translations): N/A this session.
- Part 11 (trust boundaries): confirmed — owner identity on this endpoint is server-derived (`getCurrentUserIdStrict`), not client-supplied; documented in §1.
- Other parts touched: Part 10 (this is the Phase 2 audit deliverable, `.agent/audit-<slug>.md`).

## Known gaps / TODOs

- none. The audit answers Sections 1–6 fully from the code.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code added.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- **Headline for Phase 3 seam analysis:** this endpoint *does* return `imageKeys` populated. If mobile's edit screen shows no images, the gap is on the client side (or in how the client reads `imageKeys` off this DTO), **not** in this response's content. Worth correlating with the open issues.md item "Dashboard update-product page — product images are not displayed" (2026-06-01 mobile batch).
- **Category/filters gap is real but by-design:** the "empty-looking category" and "missing filters" symptoms (same issues.md batch) line up with this endpoint carrying **no** category object and **no** unselected-filter catalog. The client edit form must source category + available filters from elsewhere (a catalog/category endpoint), pre-filling from this endpoint's `filters` (selected values). This is the central seam to resolve in Phase 3 — decide whether the edit form's category/filter scaffold is a client responsibility against the catalog, or whether the owner-fetch contract should be widened. I did not assume an answer.
- **Drift #1 (FilterConverter type mismatch):** `UpdateProductRequestConverter.java:37` — `Filter`-typed source bypasses the `CategoryFilter`-typed `FilterConverter`, so `FilterDTO.basic`/`order` and the option-ordering logic don't apply to selected filters on this endpoint. Severity low; file path and effect above. I did not fix this because it is out of scope for a read-only audit.
- (nothing else flagged)
