# Audit — Catalog Filters Backend (Price Bound + Random Suppression)

**Repo:** oglasino-backend
**Branch:** dev (confirmed checked out)
**Type:** read-only. No code changed, no git writes.
**Date:** 2026-06-01

Scope: how the product-search backend consumes `ProductsFilterDTO` for the two POST
search endpoints, so mobile parity work sends the right wire types. Code is ground truth.

## Endpoints and request shape (orientation)

- `POST /api/public/product/search` — `ProductSearchController.getAllProducts`
  (`controller/ProductSearchController.java:64-78`), body `SearchProductsDTO`,
  mode `PORTAL_SEARCH`.
- `POST /api/secure/products` — `DashboardProductController.getAllProducts`
  (`controller/DashboardProductController.java:69-79`), body `SearchProductsDTO`,
  mode `DASHBOARD_SEARCH`.
- `SearchProductsDTO` (`dto/SearchProductsDTO.java`) wraps `ProductsFilterDTO productsFilter`
  + `PagingDTO paging`. The filter object the questions are about is `productsFilter`.
- Both bodies are deserialized by Spring's default Jackson `ObjectMapper` (no custom global
  `ObjectMapper` bean, no `Jackson2ObjectMapperBuilderCustomizer`, no `spring.jackson.*`
  coercion keys in `application-{dev,prod,stage}.yaml` — grep clean). So Jackson framework
  defaults apply. Caveat per the brief: Jackson coercion behavior below is framework-level,
  not repo code; I state it as default behavior and flag where that is the only basis.

---

## Q1 — PRICE BOUND TYPE

**Declared type:** `BigDecimal`.
`dto/PriceRangeFilterDTO.java:6-7`:
```java
private BigDecimal from;
private BigDecimal to;
```
(`selectedCurrency` is `CurrencyDTO`, `free` is `Boolean`.)

**Where the price filter is applied:** `PriceQueryGenerator`
(`elasticsearch/generator/impl/PriceQueryGenerator.java`).
- `addQuery` (:20-45) null-guards `priceRange`, then branches on `free` and on
  `from`/`to` presence.
- `getPriceRangeQuery2` (:62-98) is the range generator: converts `from`/`to` to base
  currency via `baseCurrencyService.convertToBaseCurrency(...)` and emits an ES `range`
  on `basePrice` using `fromConverted.doubleValue()` / `toConverted.doubleValue()`
  (`gte` / `lte`). So the `BigDecimal` is converted, then collapsed to a `double` for ES.

**Does a JSON string `"1000"` filter, or must it be the JSON number `1000`?**
- **Both work.** Jackson's default scalar coercion (`MapperFeature.ALLOW_COERCION_OF_SCALARS`,
  on by default) converts a JSON string `"1000"` to `BigDecimal(1000)`. So
  `priceRange.from = "1000"` (JSON string) and `priceRange.from = 1000` (JSON number)
  both deserialize to the same `BigDecimal` and filter identically. A mobile client can
  send either; the string form is accepted.
- **Non-numeric strings fail.** A `from`/`to` that is a non-numeric, non-empty string
  (e.g. `"abc"`) is not coercible to `BigDecimal` and would raise a deserialization error
  → the request fails before reaching the generator (HTTP 400-class). Empty string `""`
  is the special case handled in Q2 (coerces to `null`, not an error).
- **Basis:** this rests on Jackson defaults, not repo code (no test in `src/test/java`
  exercises a string-valued price; the only price-related test hit is unrelated). The
  `InputSanitizationFilter` (see Q2) does not change this — it preserves JSON scalar types
  and only HTML-encodes/trims string values, leaving `"1000"` as `"1000"`.

**Concrete answer:** YES — `priceRange.from = "1000"` (JSON string) filters correctly; it
does not need to be the JSON number `1000`.

---

## Q2 — EMPTY/DEFAULT TOLERANCE

Mobile always emits `priceRange` (`from:""`, `to:""`, `free:undefined`), `filters:[]`,
`selectedRegionsAndCities:{regions:[],cities:[]}`, `productStates:[]`, `moderationStates:[]`.

**Overall: tolerated — no NPE, no 400, all treated as "no filter."** Per-field:

- **`priceRange.from:"" / to:""`** → Jackson coerces empty string to `null` for scalar
  number targets (default `CoercionInputShape.EmptyString` → `AsNull` for numbers). So
  `from`/`to` arrive `null`. `free:undefined` is simply omitted → `Boolean free` stays
  `null`. In `PriceQueryGenerator.addQuery` (:24-43): `priceRange` non-null but `free` null
  and both bounds null → no branch fires → no price filter. Tolerated.
  - *Basis caveat:* the `""` → `null` step is Jackson default behavior, not repo code, and
    is not covered by a repo test. If that default were ever overridden to reject empty
    strings, `from:""` would 400 instead. No such override exists in the repo today.
- **`filters:[]`** → `SpecFilterQueryGenerator.addQuery`
  (`.../SpecFilterQueryGenerator.java:17`) guards with `CollectionUtils.isNotEmpty(...)`;
  empty/null list → no filter. Tolerated.
- **`selectedRegionsAndCities:{regions:[],cities:[]}`** → tolerated for the empty case, but
  see the **wire-name mismatch** flagged below. `RegionAndCityQueryGenerator.addQuery`
  (`.../RegionAndCityQueryGenerator.java:17-27`) null-guards the object and uses
  `CollectionUtils.isNotEmpty(...)` on `regionIds`/`cityIds`; empty → no filter. No NPE.
- **`productStates:[]`** → `ProductStateQueryGenerator.wrapWithStateSearchMode`
  (`.../ProductStateQueryGenerator.java:38-69`):
  - `PORTAL_SEARCH` ignores client `productStates` entirely (hard-pins ACTIVE + APPROVED —
    trust boundary, :52-55). Empty list irrelevant.
  - `DASHBOARD_SEARCH`: empty `productStates` + allowed set {ACTIVE,INACTIVE} →
    `getProductStatesFieldValues` (:86-107) keeps `finalAllowed = allowed`, so the query is
    pinned to ACTIVE/INACTIVE (by design). No error.
  - `ADMIN_SEARCH`: empty `productStates`, `allowed` null → returns null → no filter.
  - Tolerated in all modes.
- **`moderationStates:[]`** → `addModerationStatesFilterTerms` (:71-84) guards with
  `CollectionUtils.isNotEmpty(...)`; empty → no filter. Tolerated.
- **Unknown/extra properties** → Spring Boot default
  `spring.jackson.deserialization.fail-on-unknown-properties=false` (not overridden in the
  repo), so any extra JSON keys mobile emits are silently ignored rather than 400.

**Field where an empty value is "mishandled" — NOT a hard failure, but a silent parity trap
(FLAG):** the region/city filter has a **double wire-name mismatch** between the brief's
described mobile payload and the backend DTO:

| Layer | Backend expects (`SelectedRegionsAndCitiesFilterDTO` + `ProductsFilterDTO`) | Brief says mobile emits |
| ----- | ----- | ----- |
| Outer key on `ProductsFilterDTO` | `selectedRegionAndCityValues` (`dto/ProductsFilterDTO.java:19`) | `selectedRegionsAndCities` |
| Inner list keys | `regionIds`, `cityIds` (`List<Long>`, `SelectedRegionsAndCitiesFilterDTO.java`) | `regions`, `cities` |

With empty arrays this is harmless (unknown key ignored → `selectedRegionAndCityValues`
stays null → no filter). **But if mobile ever sends populated region/city selections under
those names, the backend silently ignores them and applies no region/city filter** — no
error, just wrong results. This is exactly the "send the right wire types" concern the
audit exists to surface. Mobile must send `selectedRegionAndCityValues: { regionIds: [...],
cityIds: [...] }` with `Long` ids for region/city filtering to take effect. (Confirm the
brief's `{regions,cities}` description against the actual mobile payload — if mobile already
sends `selectedRegionAndCityValues`/`regionIds`/`cityIds`, this is a non-issue; the brief's
naming is what doesn't match the code.)

---

## Q3 — RANDOM SUPPRESSION

**Confirmed: the backend DROPS random ordering when `searchText` is non-blank OR `orderBy`
is set.** The frontends' suppression is a correct no-op match of backend behavior, not a
divergence.

**Generator / branch:** `DefaultProductsFilterQueryBuilder.applyRandomIfNeeded`
(`elasticsearch/generator/impl/DefaultProductsFilterQueryBuilder.java:122-145`), called from
`buildQuery` (:80). Logic:
```java
if (productsFilter == null || !productsFilter.isApplyRandom()) return baseQuery;   // :123
String searchText = productsFilter.getSearchText();
boolean hasSearchText = searchText != null && !searchText.isBlank();               // :127-128
if (hasSearchText || productsFilter.getOrderBy() != null) return baseQuery;        // :129-131
// otherwise wrap baseQuery in function_score random_score(seed=randomSeed, field="id")  :133-144
```
So random (`function_score` / `random_score` with `randomSeed` over field `id`) is applied
**only** when `applyRandom == true` AND `searchText` is null/blank AND `orderBy == null`.
- Non-blank `searchText` → random suppressed (relevance/`_score` ordering stands).
- Any non-null `orderBy` → random suppressed; explicit sort applied in `addSortToQuery`
  (:147-167).
- Blank/empty `searchText` does NOT suppress (`isBlank()` check) — random still applies.

**Test-backed:** `DefaultProductsFilterQueryBuilderTest`
(`src/test/java/.../DefaultProductsFilterQueryBuilderTest.java`) asserts this directly:
`applyRandom_withSearchText_skipsFunctionScoreWrap` (:124),
`applyRandom_withOrderBy_skipsFunctionScoreWrap` (:137),
`applyRandom_withBlankSearchText_appliesFunctionScoreWrap` (:154),
`applyRandom_withoutSearchTextOrOrderBy_appliesFunctionScoreWrap` (:167),
`applyRandomFalse_neverWraps` (:179).

**Concrete answer:** YES — the search path drops random ordering when `searchText`
(non-blank) or an explicit `orderBy` is set. Branch: `applyRandomIfNeeded` in
`DefaultProductsFilterQueryBuilder`.

---

## Summary table

| # | Question | Answer |
| - | -------- | ------ |
| 1 | `priceRange.from/to` type; does `"1000"` (string) filter? | `BigDecimal`. YES — JSON string `"1000"` coerces to `BigDecimal` (Jackson default) and filters identically to number `1000`. Non-numeric non-empty strings would 400. |
| 2 | Empty/default tolerance | Tolerated everywhere (no NPE/400; `""` price → null; empty lists → no filter; unknown keys ignored). FLAG: region/city wire-name mismatch (`selectedRegionAndCityValues`/`regionIds`/`cityIds` vs brief's `selectedRegionsAndCities`/`regions`/`cities`) — harmless empty, silent drop if populated. |
| 3 | Random suppressed when `searchText`/`orderBy` present? | YES — `applyRandomIfNeeded` (`DefaultProductsFilterQueryBuilder:122-145`) drops `function_score` random when non-blank `searchText` OR non-null `orderBy`. Test-backed. |
