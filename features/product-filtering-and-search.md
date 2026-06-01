# Product Filtering and Search

Current-state reference for product filtering and text search across the web
app. Five surfaces share one filtering pipeline, one wire shape, and one
backend query builder. The web is the reference implementation; this doc
captures that reality so mobile can be aligned to it later.

**Status:** shipped on `feature/validation-refactor` (post product-filtering bug sweep, 2026-05-14)
**Source repos:** `oglasino-web`, `oglasino-backend`

---

## Table of contents

- [Scope](#scope)
- [Implicit design intents](#implicit-design-intents)
- [The five surfaces](#the-five-surfaces)
  - [Home — `/`](#home--)
  - [Category — `/catalog/[[...slugs]]`](#category--catalogslugs)
  - [Owner dashboard — `/owner/products`](#owner-dashboard--ownerproducts)
  - [Admin product list — `/admin/products` and `/admin/products/[userId]`](#admin-product-list--adminproducts-and-adminproductsuserid)
  - [Public user-products page — `/user/[userId]`](#public-user-products-page--useruserid)
- [Shared machinery (web)](#shared-machinery-web)
- [Backend pipeline](#backend-pipeline)
- [Trust boundaries summary](#trust-boundaries-summary)
- [Known gaps](#known-gaps)
- [Platform adoption](#platform-adoption)

---

## Scope

| Surface | Route | Endpoint | Filter store |
|---|---|---|---|
| Home | `/` | `POST /api/public/product/search` | `usePortalFilterStore` |
| Category | `/catalog/[[...slugs]]` | `POST /api/public/product/search` | `usePortalFilterStore` |
| Owner dashboard | `/owner/products` | `POST /api/secure/products` | `useDashboardFilterStore` |
| Admin global | `/admin/products` | `POST /api/secure/admin/products` | `useAdminFilterStore` |
| Admin user | `/admin/products/[userId]` | `POST /api/secure/admin/products` | `useAdminFilterStore` |
| Public user | `/user/[userId]` | `POST /api/public/product/search` | none (no filter UI) |

Each list endpoint has a `/autocomplete` sibling with the same body shape
and `paging: { page: 0, perPage: 5 }`. Source paths in the backend are
mounted under `/api/...`; web service code references them without the
prefix and the HTTP layer adds it transparently.

---

## Implicit design intents

Stated explicitly because they govern several decisions below.

- **Page size is 20 everywhere.** Driven by one constant
  (`PRODUCTS_PER_PAGE = 20` in `src/lib/utils/utils.ts`) and routed
  through `getInitialPage()` on every SSR call.
- **Filter changes reset pagination to page 0.** Mutating any URL-mirrored
  filter triggers SSR re-fetch via Next routing → new `filtersData` prop →
  `FilteredProductList` remounts because its `key={JSON.stringify(filtersData)}`
  flips → page resets.
- **Filters persist across same-scope page navigations.** Home and category
  share `usePortalFilterStore`; admin global and admin user share
  `useAdminFilterStore`. Sibling surfaces in the same scope keep each
  other's filter state until the URL on the new surface overwrites it.
- **The owner dashboard exposes no category, region, or attribute
  filters** — only price, sort, product-state, and text search.
  Intentional minimal surface.
- **`PORTAL_SEARCH` is hard-pinned to `productState=ACTIVE AND
  moderationState=APPROVED`.** Public callers cannot see PENDING/REJECTED
  products regardless of body-supplied states.
- **`applyRandom` is suppressed when `searchText` is non-blank or
  `orderBy` is set.** Random ordering is for browsing only — it does not
  override explicit sort, and it does not overwrite text-relevance
  scoring. Suppression is enforced both server-side
  (`DefaultProductsFilterQueryBuilder.applyRandomIfNeeded`) and at the
  SSR DTO layer (`parseFiltersFromQueryParams`).

---

## The five surfaces

Every surface posts `{ productsFilter: ProductsFilterDTO, paging: { page, perPage } }`
and consumes `ProductOverviewsDTO = { products, totalNumberOfProducts }`.

### Home — `/`

- **Route:** `app/[locale]/(portal)/(public)/page.tsx`
- **SSR call:** `getPortalProducts(filtersData, getInitialPage())` → `POST /api/public/product/search`
- **Filter UI** (rendered by `Filters.tsx`): sort, price (from / to / "free" / currency),
  region/city, the catalog's "basic" filters by default, "advanced" filters on toggle.
  No product/moderation state filter.
- **Text search:** header-mounted `SearchInput`; submit goes to `/?search_text=...`.
- **Filter state:** `usePortalFilterStore`. URL-mirrored via `FilterManager` (allowed).
- **`applyRandom`:** set to `true` in `parseFiltersFromQueryParams` **only when
  `searchText` is empty and `orderBy` is unset**. With either present, the SSR
  DTO has no `applyRandom`/`randomSeed`, so the wrapper key does not flip
  per request.
- **Empty state:** `home.page.no.products.yet`. Does not distinguish "no
  products in the catalog" from "no products matching your filters" — see
  Known gaps (backend errors swallowed as empty results).

### Category — `/catalog/[[...slugs]]`

- **Route:** `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx`
  (catch-all up to three slug segments: top / sub / final).
- **SSR call:** same as home (`getPortalProducts` → `POST /api/public/product/search`).
- **Filter UI:** same `Filters.tsx`, plus the merged-in filters from
  `topCategory.filters` / `subCategory.filters` / `finalCategory.filters`.
- **Text search:** header-mounted. Autocomplete is category-scoped (sends
  the active `topCategoryId` / `subCategoryId` / `finalCategoryId`).
  **Submit preserves the active `/catalog/...` path** (post-fix) — searching
  inside a category no longer drops the category context.
- **Filter state:** `usePortalFilterStore`, shared with home.
- **Categories are in the path**, not query params: `/catalog/top/sub/final`.
  Unknown top slug → `redirect` to home.
- **Filter pruning on category navigation:** `Filters.tsx` removes selected
  filters that are not present in the destination category's filter list.
- **Empty state:** `catalog.no.products.with.filters` vs `.without.filters`,
  selected by a `filtersApplied` check covering `searchText`,
  `filters.length`, `priceRange.from/to/free`, and
  `selectedRegionsAndCities.regions.length`/`.cities.length` (post-fix).

### Owner dashboard — `/owner/products`

- **Route:** `app/[locale]/owner/products/page.tsx`
- **SSR call:** `getDashboardProducts(filtersData, getInitialPage())` → `POST /api/secure/products`
- **Filter UI** (`DashboardFilters.tsx`): inline text search, single-select
  `SelectFilter<ProductState>` over `[ACTIVE, INACTIVE]`, price, sort.
  **No region/city, no category, no attribute filters, no chip mechanic.**
- **Text search:** inside the filter panel; submit goes to `/owner/products?search_text=...`.
- **Filter state:** `useDashboardFilterStore` (`allowProductStateFilter=true`,
  `allowModerationStateFilter=false`). URL-mirrored via `FilterManager`.
- **Ownership:** web does **not** set `ownerId`. Backend derives it from the
  Firebase-authenticated identity. Body `ownerId` is dropped server-side
  (post-fix: `UserQueryGenerator` is skipped for `DASHBOARD_SEARCH`).
- **Empty state:** `products.empty.list` (no filters) vs
  `products.filters.empty.list` (filters applied). Post-fix `filtersApplied`
  check includes `searchText`, `orderBy`, `filters.length`,
  `productStates.length`, and `priceRange.from/to/free`.

### Admin product list — `/admin/products` and `/admin/products/[userId]`

**Global admin (`/admin/products`)**

- **Route:** `app/[locale]/admin/products/page.tsx`
- **SSR call:** `getAdminProducts(filtersData, getInitialPage())` → `POST /api/secure/admin/products`
- **Filter UI:** same `DashboardFilters.tsx` as owner, but the product-state
  dropdown exposes **all** states (not just ACTIVE/INACTIVE) and a
  `SelectFilter<ModerationState>` dropdown is added.
- **Filter state:** `useAdminFilterStore` (both state-filter flags `true`).
  URL-mirrored via `FilterManager`.
- **Ownership:** body `ownerId` is honored — admins legitimately scope to
  any user. Admin role is auth-derived
  (`@PreAuthorize("hasRole('ADMIN')")` on the controller +
  `isCurrentUserAdmin()` in `SearchModeQueryGenerator.ADMIN_SEARCH`).
- **Empty state:** post-fix `filtersApplied` uses inner-field checks only
  (`priceRange.from/to/free`, etc.).

**User-specific admin (`/admin/products/[userId]`)**

- **Route:** `app/[locale]/admin/products/[userId]/page.tsx`
- **SSR call:** `getAdminProducts({ ...filtersData, ownerId: parseNumber(userId) }, getInitialPage())`.
  `ownerId` is set server-side from the URL path segment.
- **Filter UI:** same as global admin.
- **Filter state:** shares `useAdminFilterStore` with global admin.
- **`FilterManager.isAllowedPath()` excludes this route.** Filter mutations
  change the chip row but do not refetch the list and do not sync to URL.
  Carried over for now; if filter-driven refetching is wanted on this
  surface, it needs a dedicated client hook plus an `isAllowedPath` entry.
- **Empty state:** renders `<></>` (empty fragment) when zero products — see Known gaps.

### Public user-products page — `/user/[userId]`

- **Route:** `app/[locale]/(portal)/(public)/user/[userId]/page.tsx`
- **SSR call:** `getPortalProducts({ ownerId: userId }, getInitialPage())` → `POST /api/public/product/search`.
  No `randomSeed`, no `applyRandom` (dead seed removed post-fix).
- **Filter UI:** **none.** No sidebar, no chip row, no on-page search. The
  header's portal `SearchInput` is visible but its submit leaves the
  seller profile.
- **Filter state:** no participation in any Zustand store. Not in
  `FilterManager.isAllowedPath()` — but there is no filter UI to drive
  anything anyway.
- **Below the list:** `ExtraProductsComponent` renders a "more from this
  seller" row with `{ ownerId, excludeIds: <initial page ids> }` and
  `perPage: 10`.
- **Empty state:** centered `user.page.no.products.found`. Unknown user → `<NotFound />`.

---

## Shared machinery (web)

### Zustand filter stores

`src/lib/store/useFilterStore.ts` exports a factory with three instances:

- `usePortalFilterStore` — home + category
- `useDashboardFilterStore` — owner dashboard
- `useAdminFilterStore` — admin global + admin user

Two factory booleans gate writes per scope:

- `allowProductStateFilter` (dashboard + admin) — gates `selectedProductStates`
- `allowModerationStateFilter` (admin only) — gates `selectedModerationStates`

`clearAllFilters()` resets every writeable field including
`selectedProductStates` and `selectedModerationStates` (post-fix).
`hasAnyFilterApplied` and `activeFilterCount` in `DashboardFilters.tsx`
both include the state arrays so the trash icon and the count badge agree
when only a state filter is active.

### FilterManager

`src/components/client/initializers/FilterManager.tsx`, mounted at each
scope's layout via `SelectableFilterManagerWrapper`:

- Hydrates the store from `window.location.search` once on mount.
- Syncs the store back to URL via `router.replace(..., { scroll: false })`
  when watched fields change. Filter mutations use `replace`, so they do
  not create back-stack entries within a page.
- Gated by `isAllowedPath()`: exact match on `/`, `/owner/products`,
  `/admin/products`; prefix match on `/catalog`. `/admin/products/[userId]`
  and `/user/[userId]` are deliberately excluded.

### SSR hydration

`filterHydrationSSR(searchParams, baseSite, t, applyRandom, categories?)`
in `src/components/client/initializers/FilterHydrationSSRInit.tsx` and
`src/lib/utils/filtersHelper.ts` builds a `ProductsFilterDTO` from URL
params for the initial SSR fetch. It mirrors the client `FilterManager`
hydration logic.

`parseFiltersFromQueryParams` mixes in `applyRandom: true, randomSeed: Date.now()`
**only when `searchText` is empty and `orderBy` is unset** (post-fix).
This keeps the SSR DTO hash — and the client `FilteredProductList` key —
stable across SSR runs on search or explicit-sort pages.

### URL query-param contract

Round-tripped via `toQueryParam` / `fromQueryParam` in `src/lib/utils/filtersHelper.ts`:

| Param | Meaning |
|---|---|
| `search_text` | Free-text query, kebab-cased (`-` → `_d_`, spaces → `-`, NFD-normalized, lowercased) |
| `f_<filterKey>` | Selected options for a catalog filter, or `from:<v>,to:<v>` for range/date; comma-separated for multi-option |
| `from`, `to`, `free` | Price range (raw numeric strings; `free=true` for "only free") |
| `currency` | Currency code; only set when `from` or `to` is set |
| `order` | Order type `code` |
| `regions`, `cities` | Comma-separated, kebab-cased label values |
| `productStates`, `moderationStates` | Comma-separated enum names (no transformation) |

### Wire shape — `ProductsFilterDTO`

`src/lib/types/filter/ProductsFilterDTO.ts`:

```ts
{
  applyRandom?, randomSeed?,
  ownerId?, topCategoryId?, subCategoryId?, finalCategoryId?,
  excludeIds?,
  searchText?,
  filters?,
  orderBy?,
  priceRange?,
  selectedRegionsAndCities?,
  productStates?,
  moderationStates?
}
```

Posted alongside `paging: { page, perPage }` to every list endpoint.
Surfaces differ only in which fields they populate.

### Pagination model

- `PRODUCTS_PER_PAGE = 20`. `getInitialPage()` returns
  `{ page: 0, perPage: 20 }` and is the source for every SSR call.
- `ProductList.tsx`: `totalPages = ceil(totalNumberOfProducts / 20)`;
  button-based pagination; no infinite scroll.
- `SelectableFilterProductListWrapper.tsx`: UI page 1 (display index 0)
  serves the cached `initialProductsData`; UI page N (for N > 0) calls
  the backend with `{ page: N, perPage: 20 }`. **No decrement** (post-fix);
  UI page maps 1:1 to backend page. The `initialRandom`-gated `excludeIds`
  branch was removed in the same fix — overlap is now prevented by the
  page numbering, not by client-side de-duplication.
- No `AbortController` on paging requests; see Known gaps.

### Text search input

`src/components/client/SearchInput.tsx`, instantiated per scope via
`SelectableSearchInputWrapper`:

- Portal scope: mounted in the header (visible on home and catalog).
- Dashboard / admin: inline at the top of the filter panel inside `DashboardFilters.tsx`.

Local state: `term` (raw input) → 300 ms debounce → `debouncedTerm`.
Autocomplete fires when `debouncedTerm.length > 2` with
`paging: { page: 0, perPage: 5 }`. `AbortController` cancels in-flight
autocomplete requests; aborted results are silently dropped (the service's
catch block returns `[]` without logging).

Submit handler:

- **Portal scope:** preserves the active `/catalog/...` path if present
  (otherwise routes to `/`).
- **Dashboard scope:** `/owner/products?search_text=...`.
- **Admin scope:** `/admin/products?search_text=...`.

---

## Backend pipeline

### Endpoints and search modes

Six endpoints — three list + three autocomplete — funnel through one
common pipeline. Each controller calls
`productsSearchFacade.getProductOverviews(searchProducts, MODE)` or
`getAutocompleteProductsSearch(searchProducts, MODE)`, where `MODE` is
determined by the controller:

| Endpoint | Mode |
|---|---|
| `POST /api/secure/products` (+ `/autocomplete`) | `DASHBOARD_SEARCH` |
| `POST /api/secure/admin/products` (+ `/autocomplete`) | `ADMIN_SEARCH` |
| `POST /api/public/product/search` (+ `/autocomplete`) | `PORTAL_SEARCH` |

Both facade methods delegate to
`DefaultProductsSearchService.getProductsFor` →
`DefaultProductsFilterQueryBuilder.buildQuery(searchProducts, mode)`.

### Query builder

`DefaultProductsFilterQueryBuilder.buildQuery` builds the result in three
distinct steps:

1. **`buildBaseQuery`** — iterates the autowired `List<QueryGenerator>`
   (BaseSite, Category, ExcludeIds, Price, RegionAndCity, SpecFilter,
   Text, User) and lets each contribute clauses from `productsFilter`.
   **`UserQueryGenerator` is skipped when the mode is `DASHBOARD_SEARCH`**
   (post-fix); for other modes it adds `filter: term(ownerId = body.ownerId)`
   when the body sets `ownerId`. Then `ProductStateQueryGenerator.wrapWithStateSearchMode`
   wraps for state/moderation filtering, and `SearchModeQueryGenerator.wrapWithSearchMode`
   wraps for ownership/authorization.
2. **`applyRandomIfNeeded`** — wraps the base query in
   `function_score(random_score, boostMode=Replace)` only when all three
   hold: `applyRandom == true`, `searchText` is null/blank, and `orderBy`
   is null. With any of those absent, the base query is returned
   unwrapped (post-fix). The random score uses `productsFilter.randomSeed`
   and `field("id")`.
3. **`addSortToQuery`** — explicit `Sort` from `orderBy` if present.
   Otherwise: `ADMIN_SEARCH` / `DASHBOARD_SEARCH` fall back to
   `Sort.by(DESC, "id")` (stable); `PORTAL_SEARCH` adds no explicit sort
   and results sort by `_score` desc.

### Search-mode wrap (ownership / authorization)

`SearchModeQueryGenerator.wrapWithSearchMode`:

| Mode | Behavior |
|---|---|
| `PORTAL_SEARCH` | No ownership clause. Public access. |
| `DASHBOARD_SEARCH` | Adds `must: term(ownerId = currentUserService.getCurrentUserId().orElseThrow())`. The `orElseThrow()` (post-fix) makes the auth-derived clause non-optional — the query either runs scoped to the caller or throws `NoSuchElementException`. Combined with the `UserQueryGenerator` skip in `buildBaseQuery`, body `ownerId` cannot contribute. |
| `ADMIN_SEARCH` | Asserts `isCurrentUserAdmin()`. Does not constrain `ownerId` — admin role permits scoping to any user via body or path. |

### State filtering

`ProductStateQueryGenerator.wrapWithStateSearchMode`:

| Mode | Behavior |
|---|---|
| `PORTAL_SEARCH` | Hard-pins `productState=ACTIVE AND moderationState=APPROVED`. Body `productStates` / `moderationStates` are silently ignored (intentional). |
| `DASHBOARD_SEARCH` | Body `productStates` intersected with `{ACTIVE, INACTIVE}` (DELETED/PENDING/EXPIRED unreachable). `moderationStates` honored as-is; no exposure because the search-mode wrap already restricts to the caller. |
| `ADMIN_SEARCH` | Body `productStates` and `moderationStates` honored. |

### Text search (Elasticsearch)

`TextQueryGenerator` adds, when `searchText` is non-blank: a `must`
nested fuzzy match on `nameTranslations.translation`, plus `should`
clauses for nested exact match and `match_bool_prefix` on name and a
match on `descriptionTranslations.translation`, with
`minimumShouldMatch("0")`. The fuzzy `must` survives any later
`function_score` wrap as a filter, so non-matching documents are
eliminated regardless of `applyRandom`. With the post-fix suppression of
`function_score` when `searchText` is non-blank, relevance ranking is
preserved.

### Pagination semantics

- **0-indexed.** `PageRequest.of(page, perPage)`. Page 0 is the first
  page on every endpoint.
- **Default page size 20.** `PagingDTO` defaults `page=0, perPage=20`.
  `SearchProductsDTO.getPagingSafe()` falls back to `Pageable.ofSize(10)`
  only if the entire `paging` object is null.
- **No server-side `perPage` cap.** `RateLimitFilter` is the only abuse
  mitigation; an explicit cap is tracked in `issues.md`.
- Same pipeline for all three list endpoints and all three autocomplete
  endpoints — no controller-specific shim.
- `NativeQuery.withTrackTotalHits(true)` is set on every search.

---

## Trust boundaries summary

Per `meta/conventions.md` Part 11, for each ownership/scoping input:

| Surface | Ownership input on the wire | Source | Verdict |
|---|---|---|---|
| Home / Category | None (public listing) | n/a | n/a |
| Owner dashboard | None (web omits `ownerId`) | Auth-derived server-side (Firebase identity from `SecurityContextHolder`, populated by `FirebaseAuthFilter`) | Safe. Body `ownerId` is dropped at the query layer (`UserQueryGenerator` skipped for `DASHBOARD_SEARCH`). `orElseThrow()` fails loud on missing auth. |
| Admin global | Optional body `ownerId` | Client-supplied | Safe — admin role is auth-derived (`@PreAuthorize` + `isCurrentUserAdmin()`); admins are explicitly permitted to scope to any user. |
| Admin user | Path segment `[userId]`, copied to body server-side | Client-supplied (URL) | Safe — same admin-role gate as global admin. |
| Public user-products | Path segment `[userId]`, set on body server-side | Client-supplied (URL) | Safe — public data, no authorization decision rides on this. |

No client-supplied "before" or "previous" values are sent on any
search/filter call. No moderation or state-transition decisions ride on
body fields.

---

## Known gaps

Tracked in [`../issues.md`](../issues.md), not part of this feature's
active work. One line each; see the linked entries for detail.

- `perPage` is uncapped server-side. Medium.
- `PORTAL_SEARCH` silently drops body `productStates` / `moderationStates` (intentional behavior; wire shape doesn't communicate it).
- `baseSite` tenant scoping has not been audited. Medium.
- Web swallows backend errors and renders them as empty results across every product-search call. Medium.
- No request cancellation on the paging path (autocomplete only).
- `parseFiltersFromQueryParams` typed `ProductsFilterDTO | undefined` but never returns `undefined` — dead branch on the home page.
- `/admin/products/[userId]` renders `<></>` on zero products (no message, no context).

Manual-test reminders (also in `issues.md`):

- Pagination overlap verification across all five surfaces with 25+ products.
- Random-ordering stability across pages within a session, reshuffle on refresh.

## Platform adoption

Mobile (`oglasino-expo`) aligns to the web/backend filtering and search behavior
documented above. This section records what mobile must match, where mobile
legitimately diverges by platform, and the seams a future deep-link feature
plugs into. Authored from three read-only audits (web reference, backend
reference, expo current-state) run 2026-05-31; resolutions confirmed by Igor in
the chat-D Mastermind session.

Status: chat D (`oglasino-expo`, on `new-expo-dev`) is the adoption chat. Backend
and web are unchanged by chat D — it is mobile-only client work against the
already-shipped backend contract.

### What mobile must match (the contract)

These are the points where mobile diverged and is being brought to parity. Each
is a behavioral match to web; the mechanism may differ (mobile has no URL bar),
but the user-visible result is identical to web.

- **`clearAllFilters` resets every filter field, including the product-state and
  moderation-state arrays.** The dashboard "active filter count" and "any filter
  active" indicator include the product-state array, so the trash icon and the
  count badge appear when only a state filter is set — matching web's
  dashboard/admin behavior. The portal count excludes state arrays (portal renders
  no state filter), matching web's portal behavior.

- **Search submit preserves category context.** On a category screen, submitting a
  text search keeps the user in that category with the search applied; it does not
  drop them to the listing root. Web does this by staying on the category URL path;
  mobile does it by navigating to the same category path (which re-derives the
  category from the path on the destination render) while carrying the search term
  in the shared portal filter store. Same user-visible result as web.

- **The public user-products list participates in no filter store.** It is driven
  solely by the seller's `ownerId`. Portal filters set on the home or category
  screen do not bleed onto the seller's listings — matching web, whose user page
  reads from no filter store.

- **The public user/seller screen has no extra-products section.** Web renders a
  "more from this seller" carousel below the main list because web paginates one page
  at a time, so there is always other inventory to surface. Mobile uses exhaustive
  infinite scroll: the main list loads the seller's entire catalog before any
  below-the-fold section would mount, so a seller-scoped carousel there is structurally
  duplicate-or-empty. Mobile therefore renders only the main list on this screen, with
  no carousel. (The `MORE_FROM_SELLER` section remains correct and in use on the
  product-detail screen, which paginates.)

- **`applyRandom` / `randomSeed` are suppressed when `searchText` is non-blank or
  `orderBy` is set.** The backend enforces this regardless of the client (confirmed
  in the backend reference audit), so this is a client-side correctness/parity
  change, not a behavior change. Its visible effect is that mobile stops
  re-randomizing result order on a manual refresh while a search or sort is active
  — matching web, which does not reshuffle under an active search.

### Platform divergences (intentional, not defects)

These differ from web and are correct for the platform. Do not "fix" them toward
web.

- **Pagination model.** Web uses button-based numbered pages; mobile uses infinite
  scroll (`FlatList.onEndReached`). Both honor page size 20 and 0-indexed paging
  against the same endpoints. The difference is platform-idiomatic.

- **Filter state persistence.** Web mirrors filter state to the URL query string
  via `FilterManager`; mobile holds it in in-memory Zustand stores (no URL bar).
  This is the expected platform-appropriate pattern.

- **State-filter option labels render the raw enum string (`ACTIVE`, `INACTIVE`).**
  This matches web, which also renders raw enum strings — the backend has no
  per-value translation keys for `ProductState` / `ModerationState`. Untranslated
  state labels are parity, not a gap. A future i18n pass (backend seed +
  client lookup, both platforms) could translate them; it is out of scope here.

- **Moderation-state filter has no mobile surface.** Admin was removed from mobile
  (chat α). The moderation-state dropdown rendered only on the admin surface, so it
  renders nowhere on mobile. The `selectedModerationStates` field and its plumbing
  remain in the store as vestigial dead code, flagged for the Ω structural sweep —
  `clearAllFilters` still resets it for completeness, but nothing populates it.

### Deep-link seam (not implemented; must not be foreclosed)

Mobile does not use a URL as its source of truth. A future feature may need to
reconstruct filter/search state from a one-time external input (a deep-link URL's
params) at app entry. Nothing implements this today, but the filter-state model
must stay populatable from such an input. Chat D's category-preservation fix
(navigating to the category path) is deep-link-compatible by construction: a cold
deep-link to a category path produces the same input the in-app navigation does.

The seam itself is `parseFiltersFromQueryParams` (`src/lib/utils/filtersUtil.ts`),
present and uncalled (grep-confirmed zero callers). It is deliberately retained,
not deleted. When deep-linking is built, that feature will need to:

- add a wire-DTO-to-FilterState adapter (the parser emits the request
  `ProductsFilterDTO` shape, not the store's `FilterState` shape — field names
  differ: `filters` vs `selectedFilters`, `orderBy` vs `selectedOrder`,
  `priceRange` vs `selectedPriceRange`);
- source category context from the path (via `getCategoriesFromPath`), since the
  parser does not produce category IDs;
- extend the parser for product-state filters if/when a portal product-state
  filter ships (the parser omits the state arrays today).

### Chat D scope boundary

Chat D adopts the four parity fixes and the `applyRandom` suppression above, plus
two mobile-only correctness fixes with no web counterpart: the `<div>` runtime
crash in the filter fall-through branch (RN has no `<div>`; both sites become
`<View>`/`<Text>`), and the `setState`-during-render patterns in the two filter
components. It also deletes a user-visible `Test123` debug placeholder on the
catalog screen. Items deliberately left for the Ω structural sweep or separate
triage: the vestigial moderation-state plumbing, the `SelectFilter`-not-dispatched
gap that makes the `<div>` branch reachable, additional `console.error` logger
violations, and the question of whether dialog-set filter changes should
auto-refresh the list without a pull-to-refresh.
