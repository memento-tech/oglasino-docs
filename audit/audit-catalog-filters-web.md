# Audit — Web Catalog Filters (reference, read-only)

**Repo:** oglasino-web
**Branch:** `dev`
**Date:** 2026-06-01
**Type:** read-only reference audit (Phase 2). No code changed.

## Branch confirmation

The checked-out branch is `dev`. It carries the current/live catalog filtering code: the
live filter UI (`src/components/client/filters/Filters.tsx` for the public catalog,
`DashboardFilters.tsx` for the owner/admin product list), the filter-type dispatch
(`Filters.tsx` → `getAppropriateFilter`), the URL hydrate/sync manager
(`src/components/client/initializers/FilterManager.tsx`), and the SSR filter parse
(`filterHydrationSSR` / `parseFiltersFromQueryParams`). Per `state.md`, `dev` is the active
branch for the shipped Google Analytics, SEO, and most platform work. This is the production
catalog filtering code.

**Naming note — there is no `FiltersDialog` on web.** The brief refers to a
"FiltersDialog / FilterManager + filter-type dispatch." On web the filter surface is a
**sidebar panel** (`Filters.tsx`), not a dialog; `FilterManager.tsx` is a headless
URL-sync component (renders `null`). `FiltersDialog` is the **mobile** (`oglasino-expo`)
component name. The filter-type dispatch on web lives in `Filters.tsx`'s
`getAppropriateFilter`, not in a dialog. I describe what the code actually has.

---

## 1. Backend filter contract as web consumes it

### The enum

`src/lib/types/filter/FilterType.ts` — web handles exactly four values:

```ts
enum FilterType { SINGLE_OPTION, MULTI_OPTION, RANGE, DATE }
```

There is **no `SELECT`** member (consistent with the `issues.md` 2026-05-31 resolution that
the backend `FilterType.java` also carries only these four). Any other value would fall
through to the dispatch's literal `<div>Filter not found {filterKey}</div>` fallback
(`Filters.tsx:157`).

### Where the filter definitions come from

Filters are read off the **base-site catalog** in the Zustand base-site store
(`useBaseSiteStore().baseSite`), populated from the backend's base-site/catalog response.
Two sources are merged for the public catalog (`Filters.tsx:66-84`, `advancedFilters`):

- `baseSite.catalog.topFilters: FilterDTO[]` — site-wide filters (always present).
- `filterCategoriesProps.topCategory?.filters`, `.subCategory?.filters`,
  `.finalCategory?.filters` — per-category dynamic filters, resolved from the URL path
  slugs (`getCategoriesFromPathSlugs` in the catalog page; `FilterCategoriesProps`).

The merged list is sorted by `FilterDTO.order` ascending. Each `FilterDTO` (the data shape
web consumes, `src/lib/types/filter/FilterDTO.ts`):

```ts
{ id: number; basic: boolean; order: number; labelKey: string; iconId?: string;
  filterType: FilterType; filterKey: string; options: FilterOptionDTO[];
  multiselection: boolean; filterRange: FilterRangeDTO }
```

`FilterRangeDTO` carries `rangePrefix`, `rangeSuffix`, and `options: number[]` (the discrete
range/year choices). `FilterOptionDTO` carries `id` and `labelKey`.

### FilterType → control web renders (dispatch site `Filters.tsx:125-159`)

| FilterType | Data shape used | Control rendered (catalog) |
| --- | --- | --- |
| `SINGLE_OPTION` | `options: FilterOptionDTO[]` | `SimpleCollapsibleFilter` — a collapsible list of **checkboxes** (one per option) |
| `MULTI_OPTION` | `options: FilterOptionDTO[]` | `SimpleCollapsibleFilter` — same checkbox list |
| `RANGE` | `filterRange.options: number[]` + prefix/suffix | `RangeFilter` — two `Select` dropdowns (from / to), each option `${prefix}${v}${suffix}` |
| `DATE` | client-generated year list (`currentYear … 1930`) | `DateFilter` — two `Select` dropdowns (from / to year); ignores `filterRange.options` |

**Finding — SINGLE_OPTION and MULTI_OPTION render identically on the catalog surface.**
Both branch to `SimpleCollapsibleFilter`, which always renders checkboxes and always calls
`addRemoveOptionFilter` (which appends to an options array). So a `SINGLE_OPTION` filter on
the public catalog allows **multiple** options to be checked simultaneously — the
single-vs-multi distinction is **not honored** on this surface. The only component that
honors `FilterDTO.multiselection` (rendering a single-select `Select` when false) is
`src/components/popups/components/FilterSelector.tsx`, and that component is used **only** by
`src/components/owner/product/MetaDataProduct.tsx` (the update-product form) — which is
explicitly **out of scope** for this audit. The catalog/browse surface does not use it.

**DATE finding:** `DateFilter` derives its year options client-side
(`new Date().getFullYear()` down to `1930`); it does not consume `filterRange.options` from
the backend for a `DATE` filter. Only `RANGE` reads `filterRange.options`.

---

## 2. Complete control inventory

Surfaces: **catalog** = public catalog page + home page (`portal` scope); **search** =
same pages with a `search_text` param (there is no separate search-results route — see §5);
**dashboard** = owner product list (`owner` scope); **admin** = admin product list (`admin`
scope, out-of-parity-scope, listed only to exclude).

| Control | Component | Reads | Selecting it does | catalog | search | dashboard | admin |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **Search text** | `SearchInput` (in `Header` for portal; in `DashboardFilters` for owner/admin) | `searchText` from the scope's filter store; autocomplete via `/…/autocomplete` | Debounced (300 ms) autocomplete popup; on Enter / "found" button → `setSearchText` + `router.push(targetPath?search_text=…)` | ✓ (header) | ✓ | ✓ | ✓ |
| **Ordering / sort** | `ProductOrder` | `baseSite.catalog.orderTypes: OrderTypeDTO[]`; `selectedOrder` | Checkbox list; toggles `setOrder` (re-click same code clears) | ✓ | ✓ | ✓ | ✓ |
| **Price (from/to/currency/free)** | `PriceFilter` + `CurrencySelector` | `selectedPriceRange`; `baseSite.allowedCurrencies` | Numeric from/to inputs, currency select, "free" checkbox → `setPriceRange` | ✓ | ✓ | ✓ | ✓ |
| **Region / City** | `RegionCityFilter` | `baseSite.regions` (regions→cities); `selectedRegionsAndCities` | Nested checkboxes; selecting all cities collapses to the region → `setRegionCityValues` | ✓ | ✓ | ✗ | ✗ |
| **Dynamic per-category filters** | `SimpleCollapsibleFilter` / `RangeFilter` / `DateFilter` via `getAppropriateFilter` | `advancedFilters` (catalog `topFilters` + category filters) | Per `FilterType` (see §1) → `addRemoveOptionFilter` / `addRemoveRangeFilter` | ✓ | ✓ | ✗ | ✗ |
| **"Show more" toggle** | inline in `Filters.tsx` | `FilterDTO.basic` | Splits dynamic filters: `basic` always shown, `!basic` shown only when expanded | ✓ | ✓ | ✗ | ✗ |
| **Product-state filter** | `SelectFilter<ProductState>` (in `DashboardFilters`) | `selectedProductStates`; option list passed in | Single `Select`; `setProductStates([v])` or `clearProductStates()` | ✗ | ✗ | ✓ (`ACTIVE`, `INACTIVE`) | ✓ (all of `ACTIVE`/`INACTIVE`/`DELETED`) |
| **Moderation-state filter** | `SelectFilter<ModerationState>` (in `DashboardFilters`) | `selectedModerationStates`; `ModerationState` enum | Single `Select`; `setModerationStates([v])` / clear | ✗ | ✗ | ✗ (renders empty `<div>`) | ✓ (`APPROVED`, `BANNED`) — **admin-only** |
| **Active-filter count badge** | inline | derived (see §4) | Display only | ✓ | ✓ | ✓ | ✓ |
| **Clear-all (trash icon)** | inline | `hasAnyFilterApplied` | `clearAllFilters()` | ✓ | ✓ | ✓ | ✓ |
| **Selected-filter chips** | `SelectedFiltersDisplay` | all selected slices | Each chip's `X` removes that one slice | ✓ | ✓ | ✓ | ✓ |

Notes:
- **Region/City and the dynamic per-category filters are catalog-only.** The dashboard
  (`DashboardFilters`) renders only: search, product-state, (admin) moderation-state, order,
  and price. It does **not** render region/city, the "free" semantics aside, or the dynamic
  category filters.
- **Product-state is dashboard + admin only.** On the owner dashboard the options are
  `ACTIVE` and `INACTIVE` only; admin gets all three `ProductState` values. The portal
  filter store is created with `allowProductStateFilter=false`, so its product/moderation
  setters are no-ops even if invoked (`useFilterStore.ts:148-169`).
- **Moderation-state is admin-only.** On the owner dashboard the moderation `SelectFilter`
  is replaced by an empty `<div>` (`DashboardFilters.tsx:135-148`). Out-of-parity-scope.

---

## 3. Filter → request mapping

### Request body shape

All product searches POST to one endpoint per scope (`productsSearchService`, both
`nextCalls` and `reactCalls` variants):

- `portal` → `POST /public/product/search`
- `owner` → `POST /secure/products`
- `admin` → `POST /secure/admin/products`

Body: `{ productsFilter: ProductsFilterDTO, paging: PagingDTO }`, with headers
`X-Base-Site` and `X-Lang`. Autocomplete uses the same body shape against the
`…/autocomplete` sibling URIs with `paging {page:0, perPage:5}`.

`ProductsFilterDTO` (`src/lib/types/filter/ProductsFilterDTO.ts`):

```ts
{ applyRandom?: boolean; randomSeed?: number; ownerId?: number;
  topCategoryId?: number; subCategoryId?: number; finalCategoryId?: number;
  excludeIds?: number[]; searchText?: string; filters?: RequestSelectedFilterDTO[];
  orderBy?: OrderTypeDTO; priceRange?: PriceRange;
  selectedRegionsAndCities?: SelectedRegionsAndCities;
  productStates?: ProductState[]; moderationStates?: ModerationState[] }
```

**Important architectural fact — the request DTO is built server-side from the URL, not
from the Zustand store.** The flow is: store → URL (via `FilterManager`) → server parse
(`filterHydrationSSR` → `parseFiltersFromQueryParams`) → `ProductsFilterDTO`. The store's
`selectedFilters` are typed `SearchSelectedFilter[]` (rich UI objects) and are **never**
converted to `RequestSelectedFilterDTO[]` on the client for the list request; the server
re-parses the query string into the DTO. Client-side pagination
(`SelectableFilterProductListWrapper.onNext…PageInternal`) reuses the same server-built
`filtersData` prop. (Autocomplete is the one client-built filter — see §5.)

### Per-control field mapping

| Control | Request field(s) | How populated |
| --- | --- | --- |
| Search text | `searchText` | From `search_text` query param (decoded via `fromQueryParam`) |
| Dynamic option filter (SINGLE/MULTI) | `filters[]` = `{ filterId, filterType, optionIds: number[] }` | URL `f_<filterKey>=<csv of toQueryParam(t(option.labelKey))>` → matched back to option ids in `getSelectedFilters` |
| Dynamic range/date filter | `filters[]` = `{ filterId, filterType, rangeFrom?, rangeTo? }` | URL `f_<filterKey>=from:<prefix><n><suffix>,to:…` → parsed by `extractRangeData` |
| Order | `orderBy: OrderTypeDTO` | URL `order=<code>` → looked up in `catalog.orderTypes` → **full** `OrderTypeDTO` object (`{code, field, labelKey, direction}`) |
| Price | `priceRange: { from: string, to: string, free: boolean, selectedCurrency?: CurrencyDTO }` | URL `from`, `to`, `free=true`, `currency=<code>` → currency resolved to the **full** `CurrencyDTO` from `baseSite.allowedCurrencies` |
| Region/City | `selectedRegionsAndCities: { regions: RegionDTO[], cities: CityDTO[] }` | URL `regions=<csv>`, `cities=<csv>` (csv of `toQueryParam(t(labelKey))`) → matched back to **full** region/city objects |
| Product-state | `productStates: ProductState[]` | URL `productStates=<csv>` → cast to `ProductState[]` (raw enum strings, no translation) |
| Moderation-state | `moderationStates: ModerationState[]` | URL `moderationStates=<csv>` → cast to `ModerationState[]` (admin only) |
| Category context | `topCategoryId` / `subCategoryId` / `finalCategoryId` | Merged in at the page level from the resolved category chain (catalog page), not from the filter UI |
| Owner scoping | `ownerId` | Set by the user page (`{ ownerId: userId }`) |
| Random | `applyRandom`, `randomSeed` | See §5 |

`RequestSelectedFilterDTO` (`{ filterId, filterType, optionIds?, rangeFrom?, rangeTo? }`)
uses `optionIds` for option filters and `rangeFrom`/`rangeTo` for range/date. Note the
request sends **filter ids** (`filterId`, `optionIds`) but the **URL** uses
translated-label slugs (`filterKey` + `toQueryParam(t(labelKey))`), not ids — the server
parse bridges the two.

**Observation:** `orderBy`, `priceRange.selectedCurrency`, and `selectedRegionsAndCities`
send **whole DTO objects** to the backend, not just codes/ids. The backend filter consumer
must accept (or tolerate the extra fields of) these full objects.

---

## 4. State lifecycle

### Where state lives

Three Zustand store instances created from one factory (`useFilterStore.ts`):

- `usePortalFilterStore` = `createFilterStore(false, false)` — catalog/home/search.
- `useDashboardFilterStore` = `createFilterStore(true, false)` — owner dashboard.
- `useAdminFilterStore` = `createFilterStore(true, true)` — admin (out of scope).

The two booleans gate `allowProductStateFilter` / `allowModerationStateFilter`; when false,
the corresponding setters are no-ops. Scope is dispatched by `portalScope` through the
`Selectable*Wrapper` components.

`FilterState` slices: `searchText`, `selectedFilters: SearchSelectedFilter[]`,
`selectedOrder?`, `selectedPriceRange: PriceRange`, `selectedRegionsAndCities`,
`selectedProductStates`, `selectedModerationStates`, plus a `hydrated` boolean guard.

There is **no persistence** (no `persist` middleware); the store is memory-only and the
**URL is the durable carrier**. There is also a parallel **server-side** parse path
(`filterHydrationSSR`) that produces the request DTO directly from `searchParams` — the
store is not consulted for the initial server render.

### Hydrate from URL

`FilterManager` (mounted once per scope via `SelectableFilterManagerWrapper` in the
`(public)`, `owner`, and `admin` layouts) hydrates once per pathname
(`FilterManager.tsx:77-225`):

- Guards: requires `baseSite`, an **allowed path** (`/`, `/owner/products`,
  `/admin/products*`, `/catalog*` — `isAllowedPath`), and that the path hasn't already
  hydrated (`lastHydratedPathRef`).
- Resolves category chain from `/catalog/...` slugs, then reads each query param and writes
  into the store: `search_text` (decoded), `f_*` filters (option filters matched by
  `toQueryParam(t(labelKey))`; range/date by `extractRangeData`), `from`/`to`/`free`/
  `currency`, `order`, `regions`/`cities`, `productStates`, `moderationStates`.
- Sets `hydrated = true` and records `lastHydratedPathRef = pathname`.

The server build (`parseFiltersFromQueryParams`) mirrors this independently for the initial
fetch; it also short-circuits to `{applyRandom, randomSeed}` when there are **no** params
and `applyRandom` is requested.

### Sync back to URL

`FilterManager`'s second effect (`:230-355`) runs on any selected-slice change once
hydrated and on an allowed path:

- Rebuilds the query string from the store, fires a `track('filter_change', …)` analytics
  event (only when the URL actually changes), and `router.replace(newUrl, {scroll:false})`.
- The URL is `/${locale}${pathSegment}?…`. Option filters serialize as
  `f_<filterKey>=<csv of toQueryParam(t(labelKey))>`; range/date as `from:…,to:…` with
  prefix/suffix; currency only emitted when a from/to value exists.

### Page reset on change

There is **no explicit page-reset counter**. Page reset is achieved structurally:
`FilteredProductList` keys `ProductList` on `JSON.stringify(keyFiltersData)` where
`keyFiltersData` is `filtersData` **minus `randomSeed`** (`FilteredProductList.tsx:31`). Any
meaningful filter change changes `filtersData` (server-rebuilt from the new URL on
navigation), changes the key, and **remounts** `ProductList` from page 0. `randomSeed` is
deliberately excluded so a per-render seed change doesn't spuriously remount.

### Active-filter count

Computed inline, differs by surface:

- **Catalog** (`Filters.tsx:116-123`): `selectedFilters.length` + 1 each for
  price.from / price.to / price.free / selectedOrder + `regions.length` + `cities.length`.
- **Dashboard/admin** (`DashboardFilters.tsx:82-89`): `selectedFilters.length` + the same
  price/order terms + `selectedProductStates.length` + `selectedModerationStates.length`.

So each price sub-field counts separately, each region and each city count as 1, and
`searchText` does **not** count toward the badge on either surface. (`searchText` *does*
show as a removable chip in `SelectedFiltersDisplay`.) On the dashboard, `selectedFilters`
is always empty (no dynamic filters rendered there) and region/city don't count.

### Clear-all

`clearAllFilters()` resets every slice to its initial value (searchText→undefined,
selectedFilters→[], order→undefined, priceRange→empty with `free:false` and no currency,
regions/cities→[], product/moderation states→[]). The trash icon shows when
`hasAnyFilterApplied`. After the store clears, the URL-sync effect strips the params.

---

## 5. Search × filter × ordering interaction

### Search coexistence

`searchText` is just another slice on the same store and another query param
(`search_text`); it coexists with all other filters and with ordering. There is **no
dedicated `/search` route** — `SearchInput.commitSearch` pushes to the **current** catalog
path (or `/` on the portal home, `/owner/products`, `/admin/products`) with a
`search_text` query param. So "search results" = the catalog/home page filtered by
`search_text`. `SearchInput` writes `searchText` raw into the store but `toQueryParam`-encodes
it into the URL; `FilterManager`/the server decode with `fromQueryParam`.

Autocomplete is the one **client-built** filter object: `SearchInput` calls
`getAutocompleteSuggestions({ ...productsFilter, searchText: debouncedTerm,
topCategoryId/subCategoryId/finalCategoryId })`. It does **not** include the currently
selected option/range/price/region filters — autocomplete is scoped only by search text +
category context.

### applyRandom / randomSeed

`randomAddOn = { applyRandom: true, randomSeed: Date.now() }`. Set only in
`parseFiltersFromQueryParams` (`filtersHelper.ts`). The caller passes an `applyRandom`
flag, which is `true` on the **home page** and the **catalog page**, and `false` on the
**owner** and **admin** product lists and in the metadata snapshot service. (The favorites
page hard-codes `applyRandom:true` on a non-filter-UI request and is outside the filter
surface.)

Suppression rules (`filtersHelper.ts:49,68`):

1. If there are **no** query params and `applyRandom` is requested → returns just
   `{applyRandom:true, randomSeed}` (random browse of an unfiltered catalog).
2. Otherwise random is appended **only if** `applyRandom && !result.searchText &&
   !result.orderBy` — i.e. random is **suppressed when a search text or an explicit order
   is present**. Other filters (price/region/option) do **not** suppress random.

The in-code comment (`filtersHelper.ts:63-67`) states the rationale: "The backend ignores
`applyRandom`/`randomSeed` when `searchText` or `orderBy` is present, so emitting them here
is wire-noise," and the non-deterministic seed would otherwise mutate `filtersData` and
trigger spurious `FilteredProductList` remounts. So web's suppression is asserted to
**match the backend's own ignore behavior** — the suppression is an optimization mirroring
backend semantics, per the code's own claim. (This audit reads only web code; the matching
backend behavior is the web code's stated assumption, to be confirmed against the backend
audit — see §7.)

`randomSeed` is excluded from the `ProductList` remount key but still travels to the backend
in `filtersData` at the page-0 fetch call sites.

---

## 6. Scope differences (public catalog vs dashboard owner)

| Capability | Public catalog (`portal`) | Owner dashboard (`owner`) | Admin (out-of-parity-scope) |
| --- | --- | --- | --- |
| Filter component | `Filters.tsx` (sidebar) | `DashboardFilters.tsx` | `DashboardFilters.tsx` |
| Search text | ✓ (header) | ✓ (in panel) | ✓ |
| Order / sort | ✓ | ✓ | ✓ |
| Price (from/to/currency/free) | ✓ | ✓ | ✓ |
| Region / City | ✓ | ✗ | ✗ |
| Dynamic per-category filters | ✓ (`topFilters` + category filters) | ✗ | ✗ |
| Basic/advanced "show more" split | ✓ | ✗ | ✗ |
| Product-state filter | ✗ (store-gated off) | ✓ `ACTIVE`/`INACTIVE` | ✓ all 3 states |
| Moderation-state filter | ✗ | ✗ (empty `<div>`) | ✓ `APPROVED`/`BANNED` — **admin-only** |
| applyRandom | ✓ (suppressed by search/order) | ✗ (`false`) | ✗ (`false`) |
| Endpoint | `/public/product/search` | `/secure/products` | `/secure/admin/products` |
| Store | `usePortalFilterStore` | `useDashboardFilterStore` | `useAdminFilterStore` |

**Out-of-parity-scope (admin-only):** the moderation-state `SelectFilter` (and its
`/secure/admin/products` endpoint, `useAdminFilterStore`, and the third `DELETED`
product-state option). Marked here only to exclude from parity work.

Region/city, the dynamic per-category filters, the basic/advanced split, and random
ordering are the public-catalog-only capabilities the dashboard does not have.

---

## 7. Seams (web's assumptions about the backend filter contract)

1. **FilterType enum is exactly four values.** Web's dispatch handles `SINGLE_OPTION`,
   `MULTI_OPTION`, `RANGE`, `DATE`; anything else renders a "Filter not found" placeholder.
   A counterpart must emit only these four (the backend `FilterType.java` does today).

2. **Catalog response carries `topFilters` + nested category `filters`.** Web reads
   `baseSite.catalog.topFilters`, `catalog.orderTypes`, `catalog.categories[].route`,
   `.subcategories[]`, and per-category `.filters`. A counterpart must shape filters under
   the catalog and per-category so web can merge them by path.

3. **`FilterDTO` shape is fixed.** Web depends on `id`, `filterKey`, `filterType`,
   `labelKey`, `iconId`, `order`, `basic`, `options[].{id,labelKey}`, and
   `filterRange.{rangePrefix,rangeSuffix,options:number[]}`. A counterpart must populate
   `filterRange.options` for `RANGE` (web ignores it for `DATE`).

4. **URL round-trip relies on translated labels being stable and unique per filter.** The
   URL stores `f_<filterKey>=<toQueryParam(t(option.labelKey))>`, and the server parse
   matches options by re-translating each `labelKey` and comparing slugs. If two options
   under one filter translate to the same slug (or a translation changes), the wrong option
   id is selected or the value is dropped. A counterpart must guarantee distinct, stable
   option labels per filter (same constraint applies to region/city labels).

5. **Request DTO field names + value formats.** The backend search consumer must accept the
   `{productsFilter, paging}` envelope with `filters[].{filterId, filterType, optionIds,
   rangeFrom, rangeTo}` (ids, not slugs), and tolerate **full objects** in `orderBy`
   (`OrderTypeDTO`), `priceRange.selectedCurrency` (`CurrencyDTO`), and
   `selectedRegionsAndCities` (`RegionDTO[]`/`CityDTO[]`), plus `productStates` /
   `moderationStates` as raw enum-string arrays.

6. **Order is keyed by `OrderTypeDTO.code`.** Web looks up the selected order by `code`
   against `catalog.orderTypes` and sends the full object back. A counterpart must key
   ordering on `code` and supply `field`/`direction`/`labelKey`.

7. **Random-ordering suppression mirrors the backend.** Web asserts (in code comment) that
   the backend **ignores** `applyRandom`/`randomSeed` whenever `searchText` or `orderBy` is
   present, and suppresses emitting them accordingly. A counterpart relying on this must
   confirm the backend actually ignores random under search/explicit-order; if the backend
   instead honored random there, web would silently differ. (To be verified against the
   backend audit — web only reads web code here.)

8. **Price `free` is a boolean within `priceRange`, not a separate "free-zone" filter.**
   The brief's "free-zone" maps to `priceRange.free` (URL `free=true`). The
   `/blog/free-zone` route is a **static marketing page** (`FREE_ZONE_PAGE` namespace) with
   no product filtering — it is **not** a filter surface. A counterpart must treat
   "free" as the price-range boolean.

---

## Appendix — files read for this audit

- Types: `FilterType.ts`, `FilterDTO.ts`, `ProductFilterDTO.ts`, `ProductsFilterDTO.ts`,
  `RequestSelectedFilterDTO.ts`, `SearchSelectedFilter.ts`, `PriceRange.ts`,
  `OrderTypeDTO.ts`, `ProductState.ts`, `ModerationState.ts`.
- Store: `useFilterStore.ts`.
- Helpers/SSR: `filtersHelper.ts`, `FilterHydrationSSRInit.tsx`.
- UI: `Filters.tsx`, `DashboardFilters.tsx`, `SimpleCollapsibleFilter.tsx`, `RangeFilter.tsx`,
  `DateFilter.tsx`, `ProductOrder.tsx`, `PriceFilter.tsx`, `RegionCityFilter.tsx`,
  `SelectedFiltersDisplay.tsx`, `SearchInput.tsx`, `admin/SelectFilter.tsx`,
  `popups/components/FilterSelector.tsx` (confirmed used only by out-of-scope
  `MetaDataProduct.tsx`).
- Wrappers: `SelectableFiltersWrapper.tsx`, `SelectableFilterManagerWrapper.tsx`,
  `SelectableSelectedFiltersDisplayWrapper.tsx`, `SelectableSearchInputWrapper.tsx`,
  `SelectableFilterProductListWrapper.tsx`, `FilteredProductList.tsx`, `FilterManager.tsx`.
- Services: `nextCalls/productsSearchService.ts`, `reactCalls/productsSearchService.ts`.
- Pages/layouts: `(public)/page.tsx`, `(public)/catalog/[[...slugs]]/page.tsx`,
  `owner/products/page.tsx`, `(public)/user/[userId]/page.tsx`,
  `(public)/blog/free-zone/page.tsx`; layout mount points for `FilterManager` confirmed in
  `(public)/layout.tsx`, `owner/layout.tsx`, `admin/layout.tsx`; portal search mount in
  `server/layout/Header.tsx`.
