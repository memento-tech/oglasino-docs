# Audit — Expo release readiness: product filtering and search

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-05-23
**Task:** Audit the state of product listing, filtering, and search in `oglasino-expo` against the canonical spec at `../oglasino-docs/features/product-filtering-and-search.md`.

---

## Section 1 — Current state in mobile

### Screens that list products

| Surface | Route file | Endpoint | Filter store |
|---|---|---|---|
| Home | `app/(portal)/(public)/index.tsx` | `POST /public/product/search` | `usePortalFilterStore` |
| Category | `app/(portal)/(public)/catalog/[...categories].tsx` | `POST /public/product/search` | `usePortalFilterStore` |
| Owner dashboard | `app/owner/dashboard/products/index.tsx` | `POST /secure/products` | `useDashboardFilterStore` |
| Admin global | `app/admin/index.tsx` | `POST /secure/admin/products` | `useAdminFilterStore` |
| Admin user | `app/admin/products/[userId].tsx` | `POST /secure/admin/products` | `useAdminFilterStore` |
| Public user | `app/(portal)/(public)/user/[...userData].tsx` | `POST /public/product/search` | `usePortalFilterStore` (read-only — no filter UI) |

Mobile has **six** listing surfaces, matching web's five spec'd surfaces plus an admin global surface (`app/admin/index.tsx`) that web does not have at the same route. The admin global and admin user screens both exist and both use `useAdminFilterStore` + `getAdminProducts`, matching the spec.

### Filters

All six listing surfaces (except public user) render a `FloatingFiltersButton` via `FilteredProductList`, which opens a `FiltersDialog` modal. The dialog provides:

- **Sort** (`ProductOrder`) — single-select from `catalog.orderTypes`.
- **Price** (`PriceFilter`) — from/to/free/currency.
- **Region/City** (`RegionCityFilter`) — multi-select. Hidden for dashboard scope.
- **Catalog filters** (basic + advanced toggle) — `MULTI_OPTION`, `SINGLE_OPTION`, `RANGE`, `DATE` types, pulled from `selectedBaseSite.catalog.topFilters` merged with category-specific filters from the current path.
- **Product state** (`SelectFilter<ProductState>`) — visible for dashboard and admin scopes.
- **Moderation state** (`SelectFilter<ModerationState>`) — visible for admin scope only.

Selected filters are displayed as removable chips via `SelectedFiltersDisplay`, rendered in the `FilteredProductList` header.

### Search

Header-mounted `SearchInput` component. Text input with 300 ms debounce, autocomplete dropdown when `debouncedTerm.length >= 3`. Category-aware: sends `topCategoryId`/`subCategoryId`/`finalCategoryId` from the current path.

Submit handler sets `searchText` on the filter store and navigates to the scope's root (`/`, `/owner/dashboard/products`, or `/admin`).

### Sort

`ProductOrder` component in the filter dialog. Single-select toggle from `selectedBaseSite.catalog.orderTypes`. Each order type has a `code` and `labelKey`.

### Pagination

**Infinite scroll** via `FlatList.onEndReached` with threshold `1.8`. Page size 20 (`PRODUCTS_PER_PAGE`). Pull-to-refresh via `onRefresh`. Tracks loaded pages via `pageRef`, `requestedPagesRef` (Set), and `loadingRef` to prevent duplicate fetches. Shows skeleton + spinner when loading more. "Scroll to top" floating button appears after scrolling past one screen height.

No button-based pagination, no "load more" button — infinite scroll only. This differs from web, which uses button-based pagination.

---

## Section 2 — Backend contract adoption

### Endpoints

Mobile calls all six endpoints the spec describes (three list + three autocomplete):

| Endpoint | Mobile calls? | Call site |
|---|---|---|
| `POST /api/public/product/search` | Yes | `productsSearchService.ts:141-146` via `getPortalProducts` |
| `POST /api/public/product/search/autocomplete` | Yes | `productsSearchService.ts:86-90` via `getPortalAutocompleteSuggestions` |
| `POST /api/secure/products` | Yes | `productsSearchService.ts:148-153` via `getDashboardProducts` |
| `POST /api/secure/products/autocomplete` | Yes | `productsSearchService.ts:92-96` via `getDashboardAutocompleteSuggestions` |
| `POST /api/secure/admin/products` | Yes | `productsSearchService.ts:155-160` via `getAdminProducts` |
| `POST /api/secure/admin/products/autocomplete` | Yes | `productsSearchService.ts:98-102` via `getAdminAutocompleteSuggestions` |

### Wire shape — `ProductsFilterDTO`

Mobile's `ProductsFilterDTO` (`src/lib/types/filter/ProductsFilterDTO.ts`) matches the spec's shape field-for-field:

| Parameter | Mobile sends? | Spec defines? | Match |
|---|---|---|---|
| `applyRandom` | Yes | Yes | Match |
| `randomSeed` | Yes | Yes | Match |
| `ownerId` | Yes | Yes | Match |
| `topCategoryId` | Yes | Yes | Match |
| `subCategoryId` | Yes | Yes | Match |
| `finalCategoryId` | Yes | Yes | Match |
| `excludeIds` | Yes (type present) | Yes | Match — but mobile never populates it in any call site |
| `searchText` | Yes | Yes | Match |
| `filters` | Yes | Yes | Match — `RequestSelectedFilterDTO[]` with `filterId`, `filterType`, `optionIds` |
| `orderBy` | Yes | Yes | Match — `OrderTypeDTO` |
| `priceRange` | Yes | Yes | Match — `PriceRange` with `from`, `to`, `free`, `selectedCurrency` |
| `selectedRegionsAndCities` | Yes | Yes | Match — `{ regions, cities }` |
| `productStates` | Yes | Yes | Match — `ProductState[]` |
| `moderationStates` | Yes | Yes | Match — `ModerationState[]` |

**Extra parameters mobile sends that aren't in the spec:** none.

**Spec-defined parameters mobile doesn't send:** `excludeIds` is defined in the type but never populated by any mobile call site. Web uses it on the public user page's `ExtraProductsComponent`; mobile's `UserProductsList.tsx` does not use it.

### Response shape

Mobile parses `ProductOverviewsDTO = { products: ProductOverviewDTO[], totalNumberOfProducts: number }` — matches the spec's `{ products, totalNumberOfProducts }`.

### Autocomplete

Autocomplete uses `paging: { page: 0, perPage: 5 }` — matches the spec.

### Paging

Mobile sends `{ page, perPage: 20 }` — 0-indexed, page size 20. Matches spec.

---

## Section 3 — URL / state model

### Filter state persistence

Mobile uses **in-memory Zustand stores** (`usePortalFilterStore`, `useDashboardFilterStore`, `useAdminFilterStore`). There is no URL serialization of filter state. The stores persist in memory for the lifetime of the app process.

- **Back-navigation:** filter state survives back-navigation within the same scope because the Zustand store is a module-level singleton. Navigating from a category page back to home preserves filters (both use `usePortalFilterStore`).
- **App backgrounding:** filter state survives backgrounding on both iOS and Android because the JS process typically stays alive. On process kill, state is lost — no AsyncStorage persistence.
- **Cross-scope navigation:** switching between portal and dashboard resets filters implicitly because different stores are used.

### Deep-link / share-link

A `parseFiltersFromQueryParams` utility exists in `src/lib/utils/filtersUtil.ts` with a `//TODO Currently not used... use it for deep links...` comment. The implementation is complete (parses `search_text`, `f_*`, `from/to/free/currency`, `order`, `regions/cities`) but is never called. Mobile has no deep-link support for filtered views.

### Divergence from web

Web uses URL-driven filter state (filters serialize to query string, deep-linkable). Mobile uses in-memory Zustand state. This is the expected platform-appropriate pattern — mobile does not have URL bars. The unused `parseFiltersFromQueryParams` suggests deep-link support was planned but not implemented.

---

## Section 4 — Search UX

### Debounce

Search is debounced at **300 ms** (`SearchInput.tsx:71`). Matches the spec's web debounce delay.

### Endpoint

Autocomplete calls the `/autocomplete` sibling endpoint, not the main list endpoint. Matches spec. Autocomplete fires when `debouncedTerm.length >= 3` (spec says `> 2`, which is equivalent).

### Submit vs autocomplete

Submit sets `searchText` on the filter store and navigates to the scope's root route. Autocomplete suggestions are rendered in a dropdown overlay; tapping a suggestion navigates directly to the product detail page. The submit button at the bottom of the autocomplete dropdown triggers the full search.

### Client-side suggestions / autocomplete / recent-search

- Autocomplete via the backend `/autocomplete` endpoint — yes.
- Client-side suggestions — no.
- Recent-search storage — no.

### Zero-result state

Autocomplete dropdown: shows `navigation.search.not.found` with the search term (`SearchInput.tsx:175`).

Full-search zero results: depends on the surface. Home has no explicit `NoProductsComponent`. Catalog has one that distinguishes filters-applied vs no-filters. Dashboard and admin have none. The distinction between "no results" and "error" is not made — see Section 5.

### Category-scoped search

Mobile sends `topCategoryId`/`subCategoryId`/`finalCategoryId` from the current path to the autocomplete endpoint. Matches spec behavior. **Submit preserves the category context** — the search submit navigates to `/` (root) rather than staying on the catalog page, so category context is lost on full-search submit. This diverges from web's post-fix behavior where search submit preserves the active `/catalog/...` path.

---

## Section 5 — Loading and error handling

### Loading state

- **Initial load:** no explicit skeleton or spinner on the main product list before data arrives. `totalNumberOfProducts` initializes to `-1` and the count is hidden until it becomes `>= 0`.
- **Pagination load:** skeleton placeholder + green `ActivityIndicator` spinner while `hasMore` is true (`ProductList.tsx:267-270`).
- **Pull-to-refresh:** native `FlatList` refresh indicator.
- **Autocomplete loading:** shows italic "Loading..." text via `tCommon('loading')` (`SearchInput.tsx:172`).

### Error handling

Backend errors are **swallowed as empty results** across all product search calls. `productsSearchService.ts:128-138` catches errors and returns `{ products: [], totalNumberOfProducts: 0 }`. No toast, no inline message, no retry button. The user sees an empty list with no indication of failure.

`ProductList.tsx:90-91` has a bare `console.error('Something went wrong!')` in the catch block — no user-facing error. `ProductList.tsx:184` has `console.error('Failed to refresh current products:', error)` in the language-change refresh path.

Autocomplete errors are similarly swallowed — `productsSearchService.ts:81-83` returns `[]` on error.

### "No results" vs "error"

Mobile does **not** distinguish between "no results" (success with empty array) and "error" (network/server failure). Both render identically: an empty list. This matches web's known gap (spec's Known gaps: "Web swallows backend errors and renders them as empty results across every product-search call").

---

## Section 6 — Translation coverage

### Translation infrastructure

Mobile uses `i18next` with `i18next-icu` for ICU message format support. Translations are fetched from the backend per namespace via `GET /public/translations?namespace=<NS>&lang=<LANG>` (`src/i18n/fetchNamespace.ts`). The `TranslationNamespace` enum matches the conventions Part 6 namespace list.

Fallback language is `'sr'` (`i18n.ts:9`).

### Locale support

Mobile's `initI18n(lang, tenant)` accepts any language code the backend serves. The backend seeds EN, SR, RU, and CNR. Mobile should support all four locales the web ships (EN, SR, RU, CNR — conventions Part 9 "Montenegrin (me/cnr) aliases to SR" is stale per the brief's standing context).

### Sample strings from filter/search UI — translation key coverage

| String | Key | Namespace | Source file |
|---|---|---|---|
| Filter panel title | `filter.side.title` | `COMMON_SYSTEM` | `FiltersDialog.tsx:160` |
| Price from label | `filter.side.price.from.label` | `COMMON_SYSTEM` | `SelectedFiltersDisplay.tsx:63` |
| "Free" filter label | `filter.side.price.free.label` | `COMMON_SYSTEM` | `SelectedFiltersDisplay.tsx:48` |
| "Show more filters" button | `filter.side.show.more.button.label` | `COMMON_SYSTEM` | `FiltersDialog.tsx:213` |
| Total products count | `total.number.of.products` | `COMMON` | `ProductList.tsx:259` |

All five keys use backend-seeded translation keys via the standard `useTranslations(namespace)` hook, so coverage across all four locales depends on the backend's SQL seed files, which are authoritative. Mobile does not ship its own translation files — it fetches everything from the backend. The same keys web uses are available to mobile because both consume the same backend translation API.

**Product state and moderation state filter labels are NOT translated.** `FiltersDialog.tsx:181-184` renders `Object.values(ProductState).map(ns => ({ value: ns, label: ns }))` — the raw enum value string (e.g., `"ACTIVE"`, `"INACTIVE"`) is used as the label, not a translation key. Same for `ModerationState` at line 190-193. Web's `DashboardFilters.tsx` likely uses translation keys for these; mobile hardcodes the raw enum.

---

## Section 7 — Dead code

1. **`parseFiltersFromQueryParams` in `src/lib/utils/filtersUtil.ts:34-49`** — complete deep-link filter parser, never called. Has a `//TODO Currently not used... use it for deep links...` comment (line 33). The supporting functions (`getSearchText`, `getSelectedFilters`, `getPriceRangeData`, `getOrderData`, `getRegionAndCitiesData`) are also unused.

2. **`<div>` element in `FiltersDialog.tsx:148`** — `return <div key={filter.id}>Filter not found {filter.filterKey}</div>`. This is a web HTML element in React Native code. React Native does not have `<div>`. This would crash at runtime if a filter with an unrecognized `FilterType` were encountered. Should be `<View>` with `<Text>`.

3. **`excludeIds` on `ProductsFilterDTO`** — defined in the type but never populated by any mobile call site. Web uses it on the user products page via `ExtraProductsComponent`; mobile does not.

4. **Unused `suffix` variable in `SearchInput.tsx:111`** — `let suffix = '';` is declared in the `placeholder` useMemo but never read.

5. **Unused imports in `catalog/[...categories].tsx`** — `tCommon` and `tDash` are imported via `useTranslations` but `tDash` is only used for the empty state; `tCommon` does not appear to be used in the rendered output.

---

## Section 8 — Performance smells (observational only)

1. **No `AbortController` on search/filter-driven refetches.** When filters change rapidly, `FilteredProductList` creates a new `fetchPageInternal` callback (via `useMemo` dependency change), which triggers `ProductList.onRefresh` (via the `useEffect` on `fetchPage`). Each change fires a new backend request with no cancellation of the previous one. Results from stale requests could arrive after newer ones.

2. **`applyRandom` not suppressed when `searchText` or `orderBy` is set.** `FilteredProductList.tsx:95` sets `applyRandom = true` unconditionally when the prop is true, regardless of search text or sort order. The spec explicitly states: "`applyRandom` is suppressed when `searchText` is non-blank or `orderBy` is set." Mobile sends both `applyRandom: true` and `searchText: "foo"` simultaneously, which the backend handles correctly (server-side `applyRandomIfNeeded` checks both), but the mobile client still generates a `randomSeed` and sends it unnecessarily, causing the `fetchPageInternal` callback to change identity on every refresh even when it shouldn't. Same issue in `UserProductsList.tsx:34`.

3. **`FiltersDialog` calls `addRemoveOptionFilter` during render.** Lines 91-95 iterate `selectedFilters` and call `addRemoveOptionFilter` (a store setter) for filters not present in `advancedFilters`. This is a state update during render — React warns about this pattern, and it can cause cascading re-renders. Should be in a `useEffect`.

4. **Language-change refresh fetches all loaded pages in parallel.** `ProductList.tsx:156-160` fires `Promise.all` over every previously loaded page when the language changes. For a user who scrolled through many pages, this could be a burst of simultaneous requests. (Overlaps with the backend-calls-reduction brief — noting briefly here.)

5. **`searchText` filter changes trigger full list refresh.** Every keystroke that survives the 300ms debounce updates `searchText` on the filter store, which changes `filtersData`, which changes `fetchPageInternal`, which triggers `onRefresh` in `ProductList`. This is correct behavior but redundant with the autocomplete fetch that fires on the same debounced term.

---

## Section 9 — For Mastermind

### Structurally notable findings

1. **`clearAllFilters()` does not clear `selectedProductStates` or `selectedModerationStates`.** `useFilterStore.ts:185-198` resets all fields except these two. The spec states web's `clearAllFilters()` "resets every writeable field including `selectedProductStates` and `selectedModerationStates`." Mobile's trash-icon clear leaves product/moderation state filters orphaned. Similarly, `FloatingFiltersButton.tsx:20-32` and `FiltersDialog.tsx:97-109` do not include product/moderation states in their `activeFilterCount` calculation.

2. **Search submit drops category context.** `SearchInput.tsx:193-198` maps portal scope to `'/'`. Web preserves the active `/catalog/...` path on search submit (post-fix). Mobile navigates to home, losing the category filter. The category IDs were only sent to the autocomplete endpoint, not carried forward to the full search.

3. **`UserProductsList` reads from `usePortalFilterStore` and applies all portal filters.** `UserProductsList.tsx:20-25` reads `searchText`, `selectedFilters`, `selectedOrder`, `selectedPriceRange`, `selectedRegionsAndCities` from the portal store and includes all of them in the request body. The spec says the public user page has "no filter UI" — but mobile applies whatever the portal store happens to contain. If a user browses home with filters active, then navigates to a user profile, those filters are silently applied to the user's product listing.

4. **`<div>` in `FiltersDialog.tsx:148`** — a web HTML element in React Native code. Would crash at runtime on an unrecognized filter type.

5. **`console.error` calls in `ProductList.tsx:91` and `:184`** — ad-hoc debug logging. Part 4 violation in existing code. Not introduced by this audit.

### Cross-feature observations (one line each)

- The `logServiceWarn` / `logServiceError` utilities in `serviceLog.ts` use `console.warn` / `console.error` throughout the service layer — this is a project-wide logging pattern question, not specific to filtering. Defer to the general project health audit.
- Image pipeline: product cards render `topImageKey` but the image pipeline audit covers that surface.
- Favorites: `ProductList` has a `refreshOnFavoritesChange` prop that triggers a full refresh on favorite toggle — this is a backend-calls question. Defer to the backend-calls-reduction audit.

### Questions whose answers would change mobile-filtering adoption scoping

1. Should mobile implement `excludeIds` on the user products page (like web's `ExtraProductsComponent`)? Currently mobile's `UserProductsList` does not exclude initial-page products from the extra sections.
2. Should the deep-link filter parser (`parseFiltersFromQueryParams`) be wired up for incoming links, or is deep-linking out of scope for v1?
3. Should product state / moderation state labels be translated, or is the raw enum string acceptable for admin/dashboard surfaces?

---

## Section 10 — Cleanup performed

`none needed` — read-only audit.

---

## Section 11 — Obsoleted by this session

`nothing` — read-only audit.

---

## Section 12 — Conventions check

- Part 4 (cleanliness): N/A this session. Two existing violations noted in Section 7 (dead code) and Section 9 (`console.error` calls). Not fixed per audit scope.
- Part 4a (simplicity): N/A this session.
- Part 4b (adjacent observations): flagged in Section 9 — `<div>` in RN code, `clearAllFilters` gap, search-submit category drop, `UserProductsList` filter bleed-through.
- Part 6 (translations): touched — translation coverage spot-checked in Section 6. Product/moderation state labels not translated.
- Part 7 (error contract): touched — error handling reviewed in Section 5. Backend errors swallowed as empty results; matches web's known gap.
- Part 8 (architectural defaults): touched — verified same endpoints, same wire shape, same `PRODUCTS_PER_PAGE = 20`.

---

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change
