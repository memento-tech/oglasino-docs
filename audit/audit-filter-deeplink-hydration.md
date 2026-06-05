# Audit — Inbound filter deep-link hydration readiness (home + catalog)

**Repo:** oglasino-expo
**Branch:** new-expo-dev (not switched)
**Type:** READ-ONLY audit. No edits, no builds.
**Date:** 2026-06-04

Companion to the web filter-contract audit. Question: can an inbound `/catalog/...?<filters>`
(locale presumed stripped upstream) land on catalog/home with filters **populated and applied**,
and what does the dormant `parseFiltersFromQueryParams` seam actually give us today?

All findings cross-checked against on-disk source with `cat`/`grep`/`sed` per the standing
tool-reliability note. The headline shape mismatch and the zero-callers fact were both
re-verified with raw reads.

---

## Q1 — The dormant `parseFiltersFromQueryParams`

**Exists.** `src/lib/utils/filtersUtil.ts:34`, prefaced by a stale TODO at line 33:
`//TODO Currently not used... use it for deep links...`

**Callers: none.** `grep -rn parseFiltersFromQueryParams app src` returns only the definition
(`filtersUtil.ts:34`). Zero call sites. Matches the decisions.md 2026-06-01 note: "kept as a
seam, not deleted."

**Input shape:** `(params: { key: string; value: string }[], baseSite: BaseSiteDTO)`
(`filtersUtil.ts:34-37`). It expects an **already-parsed array of key/value pairs** — NOT a raw
query string, NOT a URL, NOT expo-router's `useLocalSearchParams` object (which is
`Record<string, string | string[]>`). So even the input is non-native: a caller would first have
to convert `useLocalSearchParams()` output into this `{key,value}[]` array. It also needs the full
`BaseSiteDTO` (catalog, regions, allowedCurrencies) to resolve label-slugs back to ids.

**Output shape:** `ProductsFilterDTO | undefined` (`filtersUtil.ts:37`, returns `undefined` when
`params.length === 0`). Full body (lines 42-48):

```ts
return {
  searchText: getSearchText(params),
  filters: getSelectedFilters(params, baseSite.catalog),      // RequestSelectedFilterDTO[]
  orderBy: getOrderData(params, baseSite.catalog),            // OrderTypeDTO | undefined
  priceRange: getPriceRangeData(params, baseSite),            // PriceRange
  selectedRegionsAndCities: getRegionAndCitiesData(params, baseSite), // SelectedRegionsAndCities
};
```

The query-string token vocabulary it parses: `search_text`, `f_<filterKey>` (comma-joined option
label-slugs), `from` / `to` / `free` / `currency` (price), `order`, `regions` / `cities`
(comma-joined label-slugs). Label→id resolution uses `toQueryParam(labelKey)` matching
(`filtersUtil.ts:85, 131, 136`).

---

## Q2 — Does its output match the LIVE filter store shape? **No — it is STALE/MISORIENTED.**

This is the critical finding. The parser produces the **wire/request DTO** (`ProductsFilterDTO`),
which is **not** the shape the live store holds for the fields that drive the catalog/home UI and
fetch.

`useFilterStore.ts:12-39` (`FilterState`) holds these as **separate top-level fields**:

| Store field (`FilterState`) | Type | Parser produces | Match? |
| --- | --- | --- | --- |
| `searchText?` (`:15`-ish, decl `:14`) | `string` | `searchText: string` | ✅ shape OK (different container) |
| `selectedFilters` (`:15`) | `SearchSelectedFilter[]` | `filters: RequestSelectedFilterDTO[]` | ❌ **wrong shape + drops range** |
| `selectedOrder?` (`:16`) | `OrderTypeDTO` | `orderBy: OrderTypeDTO` | ✅ shape OK (name differs) |
| `selectedPriceRange` (`:17`) | `PriceRange` | `priceRange: PriceRange` | ✅ match |
| `selectedRegionsAndCities` (`:18`) | `SelectedRegionsAndCities` | same | ✅ match |
| `selectedProductStates` (`:19`) | `ProductState[]` | — (not produced) | ➖ acceptable for public catalog |
| `selectedModerationStates` (`:20`) | `ModerationState[]` | — | ➖ acceptable |

Two distinct problems:

1. **Container mismatch.** The parser packages everything into one `ProductsFilterDTO`. The store
   holds them as five separate fields with different names (`orderBy`→`selectedOrder`,
   `filters`→`selectedFilters`). There is no store action that accepts a `ProductsFilterDTO` and
   fans it out into those fields. The only field typed as `ProductsFilterDTO` is
   `preparedFilters` (`useFilterStore.ts:13`), set via `setPreparedFilters` (`:57`) — and that
   field is **not read by the catalog/home fetch** (see Q5; its only readers are
   `SearchInput.tsx:95` and `HorizontalExtraProductsListView.tsx:55`). So
   `setPreparedFilters(parser_output)` would populate nothing visible and apply nothing.

2. **`selectedFilters` is the wrong shape AND drops range/date.** The store holds
   `SearchSelectedFilter = { filter: FilterDTO, options?: FilterOptionDTO[], rangeFrom?, rangeTo? }`
   (`src/lib/types/filter/SearchSelectedFilter.ts`) — i.e. **full** `FilterDTO`/`FilterOptionDTO`
   objects, the same objects the chips and `getActiveFilterCount` read. The parser produces
   `RequestSelectedFilterDTO = { filterId, filterType, optionIds }` (`filtersUtil.ts:89-93`) —
   ids only, no full objects, **and no `rangeFrom`/`rangeTo`** (raw-verified at
   `filtersUtil.ts:83-94`). This is the exact catalog-filters-2 bug class on the *request* side
   (decisions.md 2026-06-01), here baked into the dormant parser. Reconstructing
   `SearchSelectedFilter` from a `RequestSelectedFilterDTO` requires re-looking-up the `FilterDTO`
   and each `FilterOptionDTO` from the catalog.

**Conclusion:** the parser predates the store reshape that made `selectedFilters` a
`SearchSelectedFilter[]` and added the range/date carry. It is stale against today's store and,
worse, oriented at the wire DTO rather than the store. A hydration path cannot consume it as-is.

---

## Q3 — The filter store's write surface

`useFilterStore.ts:22-38` exposes these actions (both `usePortalFilterStore` and
`useDashboardFilterStore` are the same factory, `:203-205`):

- `setSearchText(searchText?)` — `:59` (UI caller: `SearchInput.tsx:209`)
- `addRemoveOptionFilter(filter: FilterDTO, option?: FilterOptionDTO)` — `:65`, toggle semantics
  (UI: `FiltersDialog.tsx:138`)
- `addRemoveRangeFilter(filter: FilterDTO, from?: number, to?: number)` — `:114`, sets
  `rangeFrom`/`rangeTo` on the `SearchSelectedFilter` (UI: `FiltersDialog.tsx:147,156`)
- `setOrder(order?: OrderTypeDTO)` — `:154` (UI: `ProductOrder.tsx:53`)
- `setPriceRange(range: PriceRange)` — `:156` (UI: `PriceFilter.tsx:46,68,87`)
- `setRegionCityValues(values: SelectedRegionsAndCities)` — `:158`, bulk-set (UI:
  `RegionCityFilter.tsx:51,84`)
- `setProductState` / `setProductStates` / `clearProductStates` — `:161/166/171` (dashboard-gated
  by the `allowProductStateFilter` factory flag; public portal store has it `false`)
- `setModerationState` / `setModerationStates` / `clearModerationStates` — `:173/178/183`
- `setPreparedFilters(filters: ProductsFilterDTO)` — `:57` (NOT a hydration hook for the catalog
  fetch — see Q2/Q5)
- `clearAllFilters()` — `:185` (UI: `FiltersDialog.tsx:186`)

**Hydration implication:** there is **no bulk "load a full FilterState" action.** A hydration
path would either (a) call the per-axis setters — `setSearchText`, `setOrder`, `setPriceRange`,
`setRegionCityValues`, and `addRemoveOptionFilter`/`addRemoveRangeFilter` per filter — or (b) a
new bulk-set action would be added. Note `addRemoveOptionFilter`/`addRemoveRangeFilter` are
**toggles** keyed on filter id; driving them from a clean (post-`clearAllFilters`) state is
additive and safe, but a hydration brief should prefer a dedicated bulk-set to avoid
toggle-on-existing surprises and to set everything in one render.

---

## Q4 — The mount/injection point

**Neither screen reads the query string today.**

- **Home** (`app/(portal)/(public)/index.tsx:7-18`): renders `<FilteredProductList>`
  unconditionally with no params read at all. No `useLocalSearchParams`, no baseSite gate.
- **Catalog** (`app/(portal)/(public)/catalog/[...categories].tsx`): reads `usePathname()`
  (`:21`) only — and `usePathname()` **excludes** the query string. It does **not** call
  `useLocalSearchParams`. Category is resolved via `getCategoriesFromPath(selectedBaseSite.catalog,
  pathname)` in a `useMemo` (`:23-27`), gated on `selectedBaseSite`; the screen returns `null`
  until `categoriesFromPath` resolves (`:39`).

**Is `useLocalSearchParams` available here?** Yes. It's expo-router and is already used on three
sibling dynamic-route screens (`product/[...productData].tsx:55`, `user/[...userData].tsx:22`,
`owner/products/[productId].tsx:62`). Both home and catalog can read it; they simply don't.

**Mount → fetch sequence (current):**
1. Screen renders. Catalog gates on `selectedBaseSite` + resolved category (returns null until
   both); home renders immediately.
2. `<FilteredProductList>` builds `filtersData` from the live store via `useShallow`
   (`FilteredProductList.tsx:53-71`), then `fetchPageInternal` (`:115-142`).
3. `<ProductList>` (grandchild) gates its render on `bootStatus === 'ready'`
   (`ProductList.tsx:295`) and fires the **initial full load exactly once**, via the effect keyed
   `[selectedLanguage, fetchPage]` guarded by `hasLoadedInitiallyRef` (`ProductList.tsx:106-120`).

**Where a clean inject goes — and the double-fetch hazard.** React fires effects **bottom-up**:
`ProductList` (grandchild) effects run **before** the screen's (parent) effects. So if hydration
is placed in a screen `useEffect`, `ProductList`'s initial-load effect will have already fetched
with the **empty** store; hydration then mutates the store → `filtersData` → `fetchPage` identity
changes → the filter-change reset effect (`ProductList.tsx:136-160`) fires `onRefresh()` → a
**second, redundant fetch**. To inject cleanly (single fetch), the store must be hydrated
**before `ProductList`'s initial-load effect reads it** — i.e. synchronously during the screen's
first render (e.g. a once-guarded `usePortalFilterStore.getState().setX(...)` / bulk-set in a lazy
`useState` initializer or a render-time ref guard), not in a post-mount effect. This is the
single most important coordination point for the hydration brief.

**Boot gating** does not fight hydration but constrains its timing: hydration needs the full
`BaseSiteDTO` (for label→id resolution), which is `useBootStore(s => s.selectedBaseSite)`. Catalog
already waits for `selectedBaseSite` before mounting `FilteredProductList`; **home does not** and
would need a baseSite-ready guard added before it can hydrate. `BootStatus` is
`booting | offline | maintenance | update-required | intro-picker | updating | ready`
(`bootStore.ts:71-78`); `ProductList` only renders on `ready`.

---

## Q5 — The fetch trigger

The fetch is driven by **`fetchPage` identity change**, not by an explicit call or a store
subscription inside the list:

- **Initial load:** `ProductList.tsx:106-120` — one-shot when `selectedLanguage` is satisfied,
  `hasLoadedInitiallyRef`-guarded. Reads current store state through `filtersData`/`fetchPage`.
- **Filter-change reset:** `ProductList.tsx:136-160` — when `fetchPage` gets a new identity it
  calls `onRefresh()` (reset to page 0). `fetchPage` (`fetchPageInternal`) is memoized on
  `filtersData` (`FilteredProductList.tsx:115-142`), and `filtersData` is memoized on the store
  fields (`:100-113`). So **any store filter change → new `filtersData` → new `fetchPage` →
  refetch.**

**Answer to the brief's desired property:** if hydration sets the store **before** the initial
load runs (Q4), the existing initial load reads the populated state and there is **no second
fetch** — exactly the desired "populate store → existing fetch uses populated state." If hydration
sets the store **after** the initial load (e.g. from a screen effect), you get the double fetch
described in Q4. Hydration does **not** need to call the fetch itself; it only needs to win the
race against `ProductList`'s first-load effect.

---

## Q6 — Range / date / region-city round-trip risk

These were the catalog-filters bug sources. Per-axis:

- **Range / date (`rangeFrom`/`rangeTo`).**
  - Store CAN represent them: `SearchSelectedFilter.rangeFrom/rangeTo` and the
    `addRemoveRangeFilter(filter, from?, to?)` action (`useFilterStore.ts:114-152`), which sets
    them on the `SearchSelectedFilter`; the fetch carries them (`FilteredProductList.tsx:88-89`).
  - **Risk: the dormant parser DROPS them.** `getSelectedFilters` only emits `optionIds`
    (`filtersUtil.ts:89-93`) — no range. The hydration brief must (a) parse range tokens from the
    query string and (b) drive `addRemoveRangeFilter` (or the bulk-set equivalent), reconstructing
    the full `FilterDTO`. Note also a unit mismatch to watch: `addRemoveRangeFilter` takes
    `number`, while the parser's price helper reads raw string params — range/date hydration must
    coerce to `number`.

- **Price range.**
  - Store: `selectedPriceRange: PriceRange { from: string; to: string; selectedCurrency; free }`
    set via `setPriceRange` (`useFilterStore.ts:156`). `from`/`to` are **strings**.
  - Parser's `getPriceRangeData` (`filtersUtil.ts:101-112`) produces the matching `PriceRange`
    with string `from`/`to`, `free` boolean, and resolves `selectedCurrency` from
    `baseSite.allowedCurrencies`. **Shape matches** — the only gap is it lands in
    `ProductsFilterDTO.priceRange`, which must be routed to `setPriceRange` (not
    `setPreparedFilters`).

- **Region / city.**
  - Store: `selectedRegionsAndCities: { regions: RegionDTO[]; cities: CityDTO[] }` set via the
    bulk `setRegionCityValues` (`useFilterStore.ts:158`).
  - Parser's `getRegionAndCitiesData` (`filtersUtil.ts:122-142`) produces full `RegionDTO[]` /
    `CityDTO[]` resolved from `baseSite.regions` via label-slug matching. **Shape matches**; route
    to `setRegionCityValues`. One behavioral note: the catalog-filters-4 region/city
    **auto-collapse** (decisions.md 2026-06-01) lives only in the city-add path inside
    `RegionCityFilter`, not in `setRegionCityValues`; a hydrated set of regions+cities is applied
    verbatim, so the inbound URL must already encode the collapsed end-state (web's canonical URL
    does, so this is low-risk if web is the link source).

---

## Q7 — Catalog category resolution interaction

**Category is resolved BEFORE any filter application — favorable ordering.** The catalog screen
computes `categoriesFromPath` from `getCategoriesFromPath(selectedBaseSite.catalog, pathname)`
(`catalog/[...categories].tsx:23-27`, def `src/lib/utils/utils.ts:76`) and **returns `null` until
it resolves** (`:39`), so `<FilteredProductList>` (and its hydration injection point) only mount
once the category is known. A category-specific filter in the URL would have its category in hand.

Two caveats for the hydration brief:
1. The dormant parser resolves filter keys by scanning **`baseSite.catalog.topFilters` plus every
   `categories[].filters`** (`filtersUtil.ts:79-81`) — it does **not** scope to the resolved
   category. It relies on `filterKey` global uniqueness. Hydration should either keep that
   global-lookup behavior (if filterKeys are unique catalog-wide) or scope to the resolved
   category's filter set; flag for the spec.
2. Home has no category and no baseSite gate (Q4) — its hydration is the simpler case (global
   feed) but needs the baseSite-ready guard added.

---

## Hydration readiness verdict

- **Parser exists?** Yes — `filtersUtil.ts:34`, uncalled, with a stale TODO.
- **Current or stale?** **STALE and mis-oriented.** It targets the wire DTO
  (`ProductsFilterDTO`), not the live store fields; its `selectedFilters` equivalent is the wrong
  type (`RequestSelectedFilterDTO`, ids-only) and **drops `rangeFrom`/`rangeTo`**. The
  scalar axes (search/price/order/region-city) are produced in store-compatible shapes, but inside
  the wrong container with no action to unpack it. The one `ProductsFilterDTO`-typed store field
  (`preparedFilters`) is not read by the catalog/home fetch.
- **Injection point clean?** No injection exists. Neither screen reads `useLocalSearchParams`;
  catalog reads only `usePathname` (query-string-free). A naive screen-effect inject **double-
  fetches** due to bottom-up effect ordering; a clean single-fetch inject must hydrate
  synchronously on first render before `ProductList`'s initial-load effect.

**Classification: MEDIUM–LARGE (closer to LARGE).** It is not a clean "small" (parser is not
current and there's no inject), and it exceeds a pure "medium reshape" because three of the four
brief criteria for "large" are present: no inject point, real fetch-coordination work, and
category-ordering handling. It avoids full "large" only because (a) a parser skeleton exists and
(b) the scalar axes are already shape-compatible, so the work is a focused rewrite + wiring rather
than a from-scratch parser. Budget it as **large** if a bulk-set store action and a robust
no-double-fetch inject are both wanted; **medium** only if scope is trimmed to home (no category)
with the per-axis setters.

---

## Gap list — what the hydration brief must build

1. **Rewrite `parseFiltersFromQueryParams` into a store-targeting adapter** (or pair it with a
   `wire→FilterState` adapter). It must emit `SearchSelectedFilter[]` (full `FilterDTO` +
   `FilterOptionDTO` objects re-looked-up from the catalog), including **`rangeFrom`/`rangeTo`**
   for RANGE/DATE filters — the current ids-only, range-dropping mapping is the catalog-filters-2
   bug re-frozen.
2. **An input adapter** from expo-router `useLocalSearchParams()` (`Record<string,string|string[]>`)
   to the parser's `{key,value}[]` input, or change the parser's input contract.
3. **The inject call**, placed to win the race against `ProductList`'s initial-load effect —
   synchronous on first render, once-guarded — to avoid the double fetch (Q4/Q5). Decide:
   per-axis setters vs. a new **bulk-set FilterState action** (recommended — one render, no toggle
   surprises, no `preparedFilters` confusion).
4. **A baseSite-ready guard on the home screen** (catalog already gates); hydration needs the full
   `BaseSiteDTO` for label→id resolution.
5. **Per-axis routing:** search→`setSearchText`, price→`setPriceRange`, order→`setOrder`,
   region/city→`setRegionCityValues`, option filters→`addRemoveOptionFilter` (or bulk),
   range/date→`addRemoveRangeFilter` (or bulk; coerce string→number).
6. **Category-scope decision** for filter-key lookup (global vs category-scoped) per Q7.
7. **No-double-fetch verification** as an explicit acceptance criterion (one network call on a
   filtered deep-link cold-start).
8. **Decide the fate of `setPreparedFilters`/`preparedFilters`** so hydration doesn't accidentally
   route through it (it's for SearchInput / extra-products, not the catalog fetch).

---

## For Mastermind

- **Cross-repo dependency — the web contract audit.** The query-string token vocabulary the
  mobile parser expects (`search_text`, `f_<filterKey>`, `from`/`to`/`free`/`currency`, `order`,
  `regions`/`cities`, comma-joined `toQueryParam(labelKey)` slugs — `filtersUtil.ts`) must match
  byte-for-byte what web **emits** in the canonical filtered URL, or label→id resolution silently
  yields empty selections. The web filter-contract audit should be diffed against this token list
  before the hydration brief freezes a parser format. In particular confirm web encodes filter
  options and region/city as **label-slugs** (not ids) and that the slug transform matches
  `toQueryParam` (NFD-normalize, `-`→`_d_`, spaces→`-`, lowercase).

- **`+native-intent` premise could not be verified — it does not exist in this repo.** The brief
  states "locale already stripped by `+native-intent`." `find`/`grep` for `native-intent`,
  `redirectSystemPath` across `app/` and `src/` returns **nothing** — there is no
  `app/+native-intent.ts(x)`. The mobile route tree is locale-free (`/catalog/...`, no `{locale}`
  segment; `getCategoriesFromPath` slices off only the `catalog` segment, `utils.ts:82`), so an
  inbound web-canonical URL carrying `/sr/catalog/...` would NOT map without locale-stripping that
  currently has no home. Either the stripping lives elsewhere (router/linking config I should be
  pointed at) or `+native-intent` is itself part of the future deep-link work. Flagging as a
  cross-repo/scope gap, not a code defect.

- **Category-resolution-ordering risk: LOW (favorable).** The catalog screen resolves category and
  returns null until ready (Q7), so the category is available before the hydration injection point
  mounts. The only open item is whether the parser should scope filter-key lookup to the resolved
  category; today it scans the whole catalog and trusts `filterKey` uniqueness.

- **Couldn't verify:** (a) `filterKey` global uniqueness across `topFilters` + all
  `categories[].filters` — the parser assumes it; the web/backend audit should confirm. (b) Whether
  web's canonical filtered URL already encodes the region/city **collapsed** end-state (Q6) — if
  not, hydration must replicate the catalog-filters-4 collapse, which currently lives only in
  `RegionCityFilter`'s city-add path, not in `setRegionCityValues`.

- **Adjacent observation (Part 4b), low:** stale TODO at `filtersUtil.ts:33`
  (`//TODO Currently not used... use it for deep links...`) — it has no matching tracked entry;
  it's the seam decisions.md already records, so it's documented, just informal. Not fixed
  (read-only audit, out of scope).

- **Part 4a simplicity evidence:** N/A — read-only audit, no code added/changed/removed.

## Conventions check

- Part 4 (cleanliness): N/A — read-only audit, no code changed.
- Part 4a (simplicity) / Part 4b (adjacent observations): one low adjacent observation flagged
  above (stale TODO); nothing added.
- Part 6 (translations): N/A this session.
- Part 11 (trust boundaries): N/A — no request DTO changed; hydration is client-side store
  population of values the backend re-validates on the existing filter POST.
