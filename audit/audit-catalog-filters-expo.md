# Audit — Expo Catalog Filters (current-state + gap vs web)

**Repo:** oglasino-expo
**Branch:** `new-expo-dev` (confirmed checked out and clean at audit start). This branch carries the
live filter surface: `src/components/dialog/dialogs/FiltersDialog.tsx` (with `getAppropriateFilter`),
`src/lib/store/useFilterStore.ts` (with `usePortalFilterStore` / `useDashboardFilterStore`), and
`src/components/product/FilteredProductList.tsx` (the store→DTO build + random suppression). All three
exist as named in the brief.
**Type:** read-only audit. No source, test, config, or doc edits; no git writes.
**Mobile code is ground truth.** The web reference is the comparison target only; every point below was
verified against mobile code.

---

## 0. Surfaces and which store each uses

Three screens render the filterable list component `FilteredProductList`:

| Screen | File | Store | `applyRandom` | Category context |
| --- | --- | --- | --- | --- |
| Portal home | `app/(portal)/(public)/index.tsx` | `usePortalFilterStore` | `true` | none |
| Catalog (category) | `app/(portal)/(public)/catalog/[...categories].tsx` | `usePortalFilterStore` | `true` | top/sub/final from path |
| Owner dashboard products | `app/owner/dashboard/products/index.tsx` | `useDashboardFilterStore` | `false` | none |

A fourth list, the public seller page `src/components/product/UserProductsList.tsx`, calls `ProductList`
directly (NOT `FilteredProductList`): it sends only `{ ownerId, applyRandom: true, randomSeed }` and renders
**no filter controls**. It has no `FiltersDialog`, no search box, no count. Out of the filtering surface but
noted for completeness.

The filter dialog itself (`FiltersDialog`) is the same component for both portal and dashboard; it switches
behavior on `usePortalScope().currentPortalScope` (`'dashboard'` vs not).

**Admin absence (Scope-OUT confirmation):** no admin filter surface exists. `useFilterStore.ts` exposes only
`usePortalFilterStore` and `useDashboardFilterStore`. The store still carries `selectedModerationStates` and a
`useDashboardFilterStore(allowProductStateFilter=true, allowModerationStateFilter=false)` factory, but
`allowModerationStateFilter` is `false` for **both** exported stores, so moderation state is never settable
from any live UI. Admin is removed (chat α), confirmed.

---

## 1. Mobile filter contract

**FilterType enum** — `src/lib/types/filter/FilterType.ts`: exactly four members,
`SINGLE_OPTION | MULTI_OPTION | RANGE | DATE`. **No `SELECT`.** Matches the web/backend four-value contract.

**FilterDTO shape** — `src/lib/types/filter/FilterDTO.ts`:
`{ id:number, basic:boolean, order:number, labelKey:string, iconId?:string, filterType:FilterType,
filterKey:string, options:FilterOptionDTO[], multiselection:boolean, filterRange:FilterRangeDTO }`.
- `FilterOptionDTO` = `{ id:number, labelKey:string }`.
- `FilterRangeDTO` = `{ id:number, rangePrefix:string, rangeSuffix:string, options:number[] }`.
This is field-for-field identical to the web FilterDTO shape in the brief.

**Where definitions come from** — `FiltersDialog.advancedFilters` (lines 70–92): starts from
`selectedBaseSite.catalog.topFilters`, then appends `categoriesFromPath.topCategory.filters`,
`.subCategory.filters`, `.finalCategory.filters` (whichever are present), then
`filters.sort((a,b) => (a.order||0) - (b.order||0))`. So **both** site-wide top filters **and** per-category
filters resolved from the category chain are merged and sorted by `order` ascending — same resolution and merge
strategy as web. `categoriesFromPath` comes from `getCategoriesFromPath(catalog, pathname)` — the path is the
mobile analogue of web's category chain.

**Dispatch site** — `FiltersDialog.getAppropriateFilter(filter)` (lines 129–167), four branches:
- `MULTI_OPTION` **or** `SINGLE_OPTION` → **same** component `SimpleCollapsibleFilter` (a multi-select
  checkbox list). They are NOT distinguished. `multiselection` is **not read**.
- `RANGE` → `RangeFilter` (two dropdowns from `filterRange.options`).
- `DATE` → `DateFilter` (two year dropdowns, generated client-side).
- else → fallback `<View><Text>Filter not found {filterKey}</Text></View>`. With only four enum values and all
  four handled, the fallback is **unreachable today**. (Tracked as resolved/deferred in `issues.md`
  2026-05-31 "SELECT-type catalog filter has no dispatch branch" — no `SELECT` in either app or backend enum.)

The `SimpleCollapsibleFilter`/`SingleOptionFilterSelector`/`MultiOptionFilterSelector` distinction: the
standalone `SingleOptionFilterSelector`, `MultiOptionFilterSelector`, and `FilterSelector` (the one that DOES
read `filter.multiselection`) are imported **only** by `src/components/MetaDataProduct.tsx` — the
update-product form, which is Scope-OUT. The catalog surface uses `SimpleCollapsibleFilter` exclusively.

---

## 2. Mobile control inventory (by surface)

### Catalog (portal) surface — rendered in `FiltersDialog` when scope ≠ dashboard

| Web catalog control | Mobile component | Data source | What selecting does | Verdict |
| --- | --- | --- | --- | --- |
| search text | `SearchInput.tsx` (in TopBar, not the dialog) | typed input + autocomplete | debounced autocomplete; on "search for X" sets `searchText` in store and navigates to catalog/home | **HAS** (different placement) |
| ordering | `ProductOrder.tsx` | `catalog.orderTypes` | single-select radio; re-tap clears (`code===selected ? undefined : order`) | **HAS** (toggle/re-click-clears matches web) |
| price from/to | `PriceFilter.tsx` | store `selectedPriceRange` | numeric `TextInput` (`replace(/\D/g,'')`) → store `from`/`to` **as strings** | **HAS** (value type differs — strings) |
| price currency | `CurrencySelector.tsx` → `CurrencySelectionDialog` | `baseSite.allowedCurrencies`, defaults to `defaultCurrency` | sets `selectedPriceRange.selectedCurrency` (full `CurrencyDTO`) | **HAS** |
| price free-boolean | `PriceFilter.tsx` free toggle | store | toggles `selectedPriceRange.free` | **HAS** |
| region/city (nested, collapse-all-to-region) | `RegionCityFilter.tsx` | `baseSite.regions[].cities[]` | region check clears its cities + adds region; city check removes region + adds city | **DIFFERENT** — no auto-collapse when all cities individually picked (see §6) |
| dynamic per-category filters (SINGLE/MULTI/RANGE/DATE) | `getAppropriateFilter` → `SimpleCollapsibleFilter` / `RangeFilter` / `DateFilter` | merged `advancedFilters` | option toggles / range pickers update store | **HAS rendering**, but RANGE/DATE selections are **not serialized into the request** (see §3, headline gap) |
| basic/advanced "show more" split | `FiltersDialog` lines 212–223 | `filter.basic` | `basic` always shown; `!basic` shown only when `showAdvanced` | **HAS** |
| active-filter-count badge | `FiltersDialog` lines 110–123 + `FloatingFiltersButton` lines 34–47 | store | numeric badge | **DIFFERENT** count formula (see §4) |
| clear-all | `FiltersDialog` `clearAllFilters` (trash icon, shown when count>0) | store | resets all slices | **HAS** |
| selected-filter chips | `SelectedFiltersDisplay.tsx` (in list header) | store | removable chips | **DIFFERENT** — RANGE/DATE selections render no chip (see §4) |

Catalog-only mobile extra: `FiltersDialog` hides `PriceFilter` entirely when `isFreeCategory`
(`categoriesFromPath.topCategory.id === 1`) — a free-zone behavior. Region/city is hidden on dashboard
(`currentPortalScope !== 'dashboard'` gate).

### Owner dashboard surface — `FiltersDialog` when scope === dashboard

| Web dashboard control | Mobile | Notes | Verdict |
| --- | --- | --- | --- |
| search | `SearchInput` (submit routes to `/owner/dashboard/products`) | shared component | **HAS** |
| ordering | `ProductOrder` | shared | **HAS** |
| price | `PriceFilter` | shared (free-zone gate not relevant) | **HAS** |
| product-state (ACTIVE / INACTIVE) | `SelectFilter<ProductState>` (lines 194–204) | options = `Object.values(ProductState)` = **ACTIVE, INACTIVE, DELETED** | **DIFFERENT** — mobile also offers `DELETED`, which web's ACTIVE/INACTIVE-only control does not |
| (web dashboard has NO region/city) | region/city correctly hidden on dashboard | matches | **MATCH** |
| (web dashboard has NO dynamic per-category filters) | but mobile **still renders** `advancedFilters` (top filters + show-more) on the dashboard too — there is no scope gate around the `getAppropriateFilter` map | **DIFFERENT** — mobile dashboard shows dynamic/show-more filters web omits |
| (web dashboard has NO random) | `applyRandom={false}` on dashboard screen | matches | **MATCH** |

Product-state control is single-select (`SelectFilter` calls `setProductStates([v])` or `clearProductStates()`).
The store slice `selectedProductStates` is an array but the UI only ever holds 0 or 1 value.

---

## 3. Mobile filter → request mapping

The request body is built in `FilteredProductList.filtersData` (lines 60–98) and posted by
`productsSearchService.getProducts` as `{ productsFilter, paging }` to `/public/product/search` (portal) or
`/secure/products` (dashboard). `BACKEND_API` (axios) attaches `X-Base-Site` / `X-Lang` via interceptors.

```ts
filtersData = {
  topCategoryId, subCategoryId, finalCategoryId,   // from props
  searchText,                                       // store
  ownerId: userId,                                  // prop (dashboard/user)
  filters: selectedFilters.map(sf => ({
    filterId: sf.filter.id,
    filterType: sf.filter.filterType,
    optionIds: sf.options.map(o => o.id),           // <-- ONLY optionIds
  })),
  orderBy: selectedOrder,                           // full OrderTypeDTO
  priceRange: selectedPriceRange,                   // {from:string,to:string,free,selectedCurrency}
  selectedRegionsAndCities: { regions, cities },    // full RegionDTO[]/CityDTO[]
  productStates: selectedProductStates,             // ProductState[]
  moderationStates: selectedModerationStates,       // always [] (no UI)
}
```
`applyRandom`/`randomSeed` are added later in `fetchPageInternal` (see §5), not in `filtersData`.

Field-by-field vs the web ProductsFilterDTO:

| Web field | Mobile | Match? |
| --- | --- | --- |
| `filters[].filterId` | sends `filterId` (id) | **MATCH** (ids, like web) |
| `filters[].optionIds` | sends `optionIds` (ids) | **MATCH** |
| `filters[].rangeFrom` / `rangeTo` | **NOT SENT.** `filtersData` maps only `optionIds`; `sf.rangeFrom`/`sf.rangeTo` (set by `addRemoveRangeFilter`) are dropped. RANGE/DATE filters therefore POST `{filterId, filterType, optionIds:[]}` — an empty option filter that filters nothing. | **MISSING — headline gap** |
| `orderBy` (full OrderTypeDTO) | full `selectedOrder` object | **MATCH** |
| `priceRange.{from,to,free,selectedCurrency}` | full object; `selectedCurrency` full `CurrencyDTO` | **MATCH on shape**, but `from`/`to` are **strings** ("" when empty) vs web's numeric expectation — see §7 |
| `selectedRegionsAndCities` (full DTOs) | full `RegionDTO[]`/`CityDTO[]` | **MATCH** |
| `productStates` | `ProductState[]` (dashboard) | **MATCH** (but DELETED reachable, §2) |
| `moderationStates` | always `[]` | MATCH (web sends `?`; mobile sends empty — harmless) |
| `ownerId` / `topCategoryId` / `subCategoryId` / `finalCategoryId` / `excludeIds` | ids; `excludeIds` not used by these screens | MATCH |

**Shape note:** web emits most of these fields **optionally**; mobile **always** includes `priceRange`,
`selectedRegionsAndCities`, `filters`, `productStates`, `moderationStates` even when empty/default
(`{from:'',to:'',free:undefined,...}`, `{regions:[],cities:[]}`, `[]`). Backend tolerance for empty values is
assumed (no observed failure), but the emitted DTO is not byte-identical to web's "omit when absent" build.

---

## 4. Mobile state lifecycle

**Where state lives:** `src/lib/store/useFilterStore.ts`, a `createFilterStore(allowProductStateFilter,
allowModerationStateFilter)` Zustand factory producing two singletons: `usePortalFilterStore(false,false)` and
`useDashboardFilterStore(true,false)`. Slices: `searchText?`, `selectedFilters:SearchSelectedFilter[]`,
`selectedOrder?`, `selectedPriceRange`, `selectedRegionsAndCities`, `selectedProductStates`,
`selectedModerationStates`, plus a `preparedFilters?` slot (used by `SearchInput`/autocomplete, not the list).

**Hydration:** none. There is **no persistence and no URL** — mobile builds the request from the store directly.
Initial state is the literal defaults in the factory (`searchText: undefined`, empty arrays,
`selectedPriceRange:{from:'',to:'',selectedCurrency:undefined,free:undefined}`). `parseFiltersFromQueryParams`
exists in `src/lib/utils/filtersUtil.ts` but is dead — it carries a `//TODO Currently not used... use it for
deep links...` comment and has no callers (storeless build; no deep-link hydration).

**Page-reset-on-change:** indirect, via component identity. `FilteredProductList.fetchPageInternal` is a
`useCallback` keyed on `[filtersData, searchText, selectedOrder]`; `filtersData` is a `useMemo` over every
filter slice. Any filter change → new `filtersData` → new `fetchPageInternal` identity. `ProductList` has a
dedicated effect (lines 134–158) that calls `onRefresh()` (reset to page 0, clear products, refetch) whenever
`fetchPage` identity changes — guarded so it does not fire before the initial load and is distinct from the
language-switch path (`onRefreshCurrent`, scroll-preserving). So **yes**, changing any filter resets to page 0
and refetches.

**Active-filter-count** (`FiltersDialog` 110–123, duplicated in `FloatingFiltersButton` 34–47):
```
selectedFiltersCount = Σ over selectedFilters of (filter.options?.length || 0)
activeFilterCount = selectedFiltersCount
  + (price.from?1:0) + (price.to?1:0) + (price.free?1:0)
  + (selectedOrder?1:0)
  + regions.length + cities.length
  + (scope==='dashboard' ? selectedProductStates.length : 0)
```
Term-by-term vs web (`selectedFilters.length + price.from + price.to + price.free + selectedOrder +
regions.length + cities.length`):
- **searchText:** does not count in either. **MATCH.**
- **price sub-fields / order / regions / cities:** same terms. **MATCH.**
- **dynamic filters:** web counts `selectedFilters.length` (1 per filter regardless of #options). Mobile counts
  **total options across all filters**, so a filter with 2 chosen options counts **2**, not 1. **DIFFERENT.**
- **RANGE/DATE filters:** stored with `options:[]`, so they contribute **0** to mobile's count (web counts them
  as 1 each via `selectedFilters.length`). **DIFFERENT** — range/date selections are invisible in the badge.
- **dashboard product-state:** mobile adds `selectedProductStates.length`; web's catalog formula (the only one
  the brief gives) does not include it. Likely fine for the dashboard surface but is an extra term.

**Selected-filter chips** (`SelectedFiltersDisplay`): renders chips for `searchText`, `selectedOrder`,
`price.free`, `price.from`, `price.to`, each region, each city, and `selectedFilters.flatMap(sf =>
sf.options.map(...))`, plus product/moderation-state. Because option chips come from `sf.options`, **RANGE and
DATE selected filters render no chip** (their `options` is `[]`). They can only be cleared from inside the
RangeFilter/DateFilter trash icon, not from the chip row. (Note: mobile shows a `searchText` chip — web's brief
formula doesn't count searchText, but chip presence is a separate UI choice; mobile renders it.)

**Clear-all** (`clearAllFilters`): resets `preparedFilters:{}`, `searchText:undefined`, `selectedFilters:[]`,
`selectedOrder:undefined`, `selectedPriceRange` to defaults, `selectedRegionsAndCities:{regions:[],cities:[]}`,
`selectedProductStates:[]`, `selectedModerationStates:[]`. Resets everything. **HAS.**

---

## 5. Search × filter × ordering × random

`searchText` lives in the store, set by `SearchInput`'s "search for X" footer button (it also routes to the
catalog/home or `/owner/dashboard/products`). It feeds `filtersData.searchText` and coexists with all other
slices (they are independent store fields all merged into one DTO). Ordering is `selectedOrder`
(single, re-tap clears).

**Random:** set only in `FilteredProductList.fetchPageInternal` (lines 104–114):
```
suppressRandom = (searchText ?? '').trim() !== '' || selectedOrder !== undefined;
if (applyRandom && !suppressRandom) { applyRandom=true; randomSeed = (page 0 ? new Date.now() : stored ref) }
```
So mobile emits `applyRandom`/`randomSeed` only when `applyRandom && !searchText && !orderBy` — **identical to
web's suppression rule** (`applyRandom && !searchText && !orderBy`). Other filters (price/region/option) do NOT
suppress random. `applyRandom` is `true` on home/catalog screens, `false` on the dashboard. The
`(searchText ?? '')` guard is present (the `issues.md` 2026-05-31 `.trim()`-on-undefined item is resolved /
already-guarded). `randomSeed` is held in a `useRef`, re-seeded on page-0 reloads, reused for subsequent pages
— so pagination keeps one consistent random order. **MATCH.**

(`UserProductsList` sends `applyRandom:true` unconditionally with its own seed; it has no search/order, so the
suppression rule is moot there.)

---

## 6. GAP TABLE

CATALOG (portal):

| Capability | Web behavior | Mobile current behavior | Gap | Parity action |
| --- | --- | --- | --- | --- |
| FilterType enum | 4 values, no SELECT | 4 values, no SELECT | **MATCH** | none |
| Definition source + merge | top filters + category-chain filters, sorted by `order` | same (top + path categories, sorted by `order`) | **MATCH** | none |
| SINGLE vs MULTI dispatch | both → same multi-select checkbox list; `multiselection` ignored | both → `SimpleCollapsibleFilter` multi-select; `multiselection` ignored | **MATCH** | none (decision point (a) resolved below) |
| RANGE control | two dropdowns from `filterRange.options`, `${prefix}${v}${suffix}` | `RangeFilter` renders exactly that | **MATCH (render)** | — but see request row |
| DATE control | two year dropdowns, currentYear→1930, ignores `filterRange.options` | `DateFilter` generates currentYear→1930, ignores options | **MATCH (render)** | — but see request row |
| RANGE/DATE → request | sends `rangeFrom`/`rangeTo` | **drops them** — `filtersData` maps only `optionIds` | **DIFFERENT (broken)** | add `rangeFrom: sf.rangeFrom, rangeTo: sf.rangeTo` to the `filters` map in `FilteredProductList.filtersData` |
| search text | sets searchText | `SearchInput` sets `searchText`, coexists | **MATCH** | none |
| ordering | from `orderTypes`, toggle, re-click clears | `ProductOrder`, toggle, re-click clears | **MATCH** | none |
| price from/to | numeric | strings (`replace(/\D/g,'')`) → DTO as strings | **DIFFERENT (type)** | confirm backend coerces string→number; if not, parse to number in the build |
| price currency (full DTO) | full `CurrencyDTO` | full `CurrencyDTO`, defaults to `defaultCurrency` | **MATCH** | none |
| price free boolean | boolean | boolean toggle | **MATCH** | none |
| region/city nested + collapse-all-to-region | selecting all cities collapses to region | region-select collapses; but selecting each city individually does NOT auto-collapse to region | **DIFFERENT** | add all-cities-selected → collapse-to-region logic in `RegionCityFilter.toggleCity` if parity wanted |
| region/city → request (full DTOs) | full `RegionDTO`/`CityDTO` | full DTOs | **MATCH** | none |
| basic/advanced show-more split | `basic` always; `!basic` on expand | identical | **MATCH** | none |
| active-filter count | `selectedFilters.length` (1/filter) + price/order/region/city | Σ option counts + price/order/region/city; RANGE/DATE count 0 | **DIFFERENT** | recompute: count 1 per `selectedFilters` entry (or per-option, but then align web); include range/date |
| clear-all | resets all | resets all | **MATCH** | none |
| selected-filter chips | chips incl. range/date | chips for options/price/order/region/city/search; **no chip for RANGE/DATE** | **DIFFERENT** | render a chip for `sf.rangeFrom`/`sf.rangeTo` entries in `SelectedFiltersDisplay` |
| random + suppression | `applyRandom && !searchText && !orderBy` | identical | **MATCH** | none |

DASHBOARD (owner):

| Capability | Web behavior | Mobile current behavior | Gap | Parity action |
| --- | --- | --- | --- | --- |
| search | yes | yes | **MATCH** | none |
| ordering | yes | yes | **MATCH** | none |
| price | yes | yes | **MATCH** | none |
| product-state | ACTIVE / INACTIVE only | `SelectFilter` over `Object.values(ProductState)` = ACTIVE/INACTIVE/**DELETED** | **DIFFERENT** | restrict dashboard product-state options to ACTIVE/INACTIVE |
| region/city | absent | absent (hidden on dashboard) | **MATCH** | none |
| dynamic per-category filters | absent | **present** — `advancedFilters` map + show-more render on dashboard too | **DIFFERENT** | gate the dynamic-filter map + show-more to `currentPortalScope !== 'dashboard'` |
| basic/advanced split | absent | present (because dynamic filters render) | **DIFFERENT** | same gate as above |
| random | absent (`applyRandom=false`) | `applyRandom=false` | **MATCH** | none |

### Flagged decision points

**(a) `multiselection` honoring.** Mobile does **NOT** honor `multiselection` on the catalog surface.
`getAppropriateFilter` routes both `SINGLE_OPTION` and `MULTI_OPTION` to `SimpleCollapsibleFilter`, which always
allows multiple selections (its `onFilterCheck` → `addRemoveOptionFilter` appends to an options array). This
**matches** web (web also collapses both to multi-select and ignores `multiselection` on the catalog). The only
mobile component that reads `filter.multiselection` is `FilterSelector`, used solely by the out-of-scope
update-product form (`MetaDataProduct.tsx`). **MATCH.**

**(b) store→DTO request build.** Mobile's mapping produces the same **field names** and the same **id-vs-object
discipline** as web (ids for filterId/optionIds; full objects for orderBy/selectedCurrency/regions+cities), so
the DTO is structurally web-shaped. Three value-level divergences: (1) **RANGE/DATE `rangeFrom`/`rangeTo` are
dropped** — the single most impactful gap, range/date filters do not actually filter; (2) `priceRange.from`/`to`
are strings, not numbers; (3) mobile always emits empty default sub-objects where web omits absent fields.
**DIFFERENT** (driven mainly by (1)).

---

## 7. Mobile-specific seams & risks

- **RANGE/DATE serialization gap (high).** `FilteredProductList.tsx:69–75` builds `filters` with only
  `optionIds`; `addRemoveRangeFilter` stores `rangeFrom`/`rangeTo` on the same `SearchSelectedFilter` but the
  build never reads them. Net effect: range and date filters render, are selectable, but silently filter
  nothing, show no chip, and add 0 to the count. This is the concrete substance behind the open `issues.md`
  item "Catalog filters — the mobile filters are not right and have filters missing versus web" (2026-05-31,
  medium). Phase 3 should treat this as the catalog-filter parity headline.
- **Price `from`/`to` are strings (medium).** Stored as `''`/digit-strings and sent verbatim in `priceRange`.
  Relies on backend coercing string→number on the `RANGE` price path. If web sends numbers, this is a wire-type
  divergence to confirm against the backend `ProductsFilterDTO.priceRange` deserialization.
- **Dashboard renders catalog-only controls (medium).** The dynamic per-category filter map, the show-more
  split, and the `DELETED` product-state option all appear on the owner dashboard, where web's dashboard is
  intentionally narrower (search/order/price/state-ACTIVE-INACTIVE). No scope gate around the
  `getAppropriateFilter` map.
- **Count formula divergence (low/medium).** `selectedFiltersCount` sums option counts and excludes range/date;
  the badge can read differently from web for the same selection. Duplicated in two files (`FiltersDialog` and
  `FloatingFiltersButton`) — any fix must touch both to stay consistent.
- **Region/city no auto-collapse (low).** Selecting every city in a region individually does not collapse to the
  region (web does). Functionally still correct (sends the cities), just a wire/label difference.
- **Render-time setState (resolved).** The 2026-05-28 `issues.md` "setState-during-render in
  Filters.tsx/FiltersDialog.tsx" item is resolved (2026-05-31, `product-filtering-3`); the current dialog uses a
  `useEffect` (lines 96–104) to prune selected filters not in the active category — no render-time setState.
  `Filters.tsx` is deleted dead code per that entry.
- **searchText `.trim()` guard (resolved).** `FilteredProductList.tsx:104` uses `(searchText ?? '').trim()`; the
  2026-05-31 latent-crash item is closed as already-guarded.
- **SELECT fallback (resolved/deferred).** `getAppropriateFilter`'s "Filter not found" fallback is a safe
  `<View>`/`<Text>` and unreachable — no `SELECT` in the contract (`issues.md` 2026-05-31, resolved-deferred).
- **Dead deep-link path (low).** `filtersUtil.parseFiltersFromQueryParams` and its helpers are unused
  (`//TODO ... use it for deep links`). Storeless build has no URL hydration; this code is inert.
- **Storeless build (structural).** No persistence, no URL/SSR — the store is the single source for the request.
  Parity is purely a store→DTO mapping question (§3), which is where the gaps concentrate.

---

## Closing note

All seven sections complete; §6 GAP TABLE covers every web catalog and dashboard capability plus both flagged
decision points. No mobile code changed. Two already-tracked items (SELECT dispatch, searchText `.trim()`) are
noted as resolved so Phase 3 does not re-litigate them; the open catalog-filter parity item is grounded to the
RANGE/DATE serialization gap in §3/§7.
