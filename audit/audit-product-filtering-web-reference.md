# Audit — Web filtering & search reference (oglasino-web)

**Repo:** oglasino-web
**Branch at audit time:** `dev` (the brief expected `stage`; I did not switch — read-only, branch is irrelevant to the findings).
**Mode:** READ-ONLY. No code changed, nothing staged, no commit.
**Scope:** the product filtering/search pipeline only, this repo only. Code is ground truth; no other doc was treated as authoritative.

Every factual claim below cites `file:line`. Where a question has no clean answer I say so.

---

## Pipeline map (the files that actually exist)

- Filter store factory: `src/lib/store/useFilterStore.ts` — exports three store instances (`useFilterStore.ts:189-191`).
- Portal filter UI: `src/components/client/filters/Filters.tsx`.
- Dashboard/admin filter UI: `src/components/client/filters/DashboardFilters.tsx`.
- Generic select dropdown: `src/components/admin/SelectFilter.tsx`.
- Text search input: `src/components/client/SearchInput.tsx`.
- Scope routers (pick a store by `portalScope`): `SelectableFiltersWrapper.tsx`, `SelectableSearchInputWrapper.tsx`, `SelectableFilterManagerWrapper.tsx`, `SelectableSelectedFiltersDisplayWrapper.tsx`, `SelectableFilterProductListWrapper.tsx`.
- URL ↔ store sync initializer: `src/components/client/initializers/FilterManager.tsx`.
- SSR filter hydration: `src/components/client/initializers/FilterHydrationSSRInit.tsx` + `src/lib/utils/filtersHelper.ts`.
- Wire type: `src/lib/types/filter/ProductsFilterDTO.ts`.
- Full-search / autocomplete services: `src/lib/service/nextCalls/productsSearchService.ts` (SSR), `src/lib/service/reactCalls/productsSearchService.ts` (client), `src/lib/service/reactCalls/extraProductsService.ts`.
- List + pagination: `src/components/client/product/FilteredProductList.tsx` → `ProductList.tsx`.
- Listing routes:
  - home `/`: `app/[locale]/(portal)/(public)/page.tsx`
  - category: `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx`
  - owner: `app/[locale]/owner/products/page.tsx`
  - admin: `app/[locale]/admin/products/page.tsx`
  - admin-user: `app/[locale]/admin/products/[userId]/page.tsx`
  - public user: `app/[locale]/(portal)/(public)/user/[userId]/page.tsx`

Three store instances, created by one factory parameterized on two booleans (`useFilterStore.ts:45`, `:189-191`):

| Store | `allowProductStateFilter` | `allowModerationStateFilter` |
|-------|---------------------------|------------------------------|
| `usePortalFilterStore` | `false` | `false` |
| `useDashboardFilterStore` (owner) | `true` | `false` |
| `useAdminFilterStore` | `true` | `true` |

---

## Q1 — `clearAllFilters()` field coverage + active-filter boolean/count

### `clearAllFilters()` resets every field, including both state arrays — **YES**.

`useFilterStore.ts:172-186`. It `set(...)`s all seven user-facing fields back to initial:

- `searchText: undefined` (`:174`)
- `selectedFilters: []` (`:175`)
- `selectedOrder: undefined` (`:176`)
- `selectedPriceRange: { from: '', to: '', selectedCurrency: undefined, free: false }` (`:177-182`)
- `selectedRegionsAndCities: { regions: [], cities: [] }` (`:183`)
- **`selectedProductStates: []`** (`:184`) ← confirmed
- **`selectedModerationStates: []`** (`:185`) ← confirmed

Note: `clearAllFilters` resets the state arrays **directly**, bypassing the `allowProductStateFilter` / `allowModerationStateFilter` guards that gate the individual `setProductStates`/`setModerationStates` setters (`:153-157`, `:165-169`). It does **not** reset the `hydrated` flag (`:60`, `:22`) — that is by design (prevents re-hydration).

### "Is any filter active" boolean + "active filter count"

There are **two separate implementations**, one per UI component. They do not share code and they inspect **different** field sets.

**Dashboard / admin** — `DashboardFilters.tsx`:

`hasAnyFilterApplied` (`DashboardFilters.tsx:73-80`) inspects:
- `selectedFilters.length` (`:74`)
- `selectedPriceRange.from` / `.to` / `.free` (`:75-77`)
- `selectedOrder` (`:78`)
- **`selectedProductStates.length > 0`** (`:79`) ← included
- **`selectedModerationStates.length > 0`** (`:80`) ← included

`activeFilterCount` (`DashboardFilters.tsx:82-89`) counts:
- `selectedFilters.length` + price(from/to/free) + order
- **`+ selectedProductStates.length`** (`:88`) ← included
- **`+ selectedModerationStates.length`** (`:89`) ← included

So in the dashboard/admin UI the **trash icon and the count badge agree** when only a state filter is active: both include the product-state and moderation-state arrays. (The trash icon renders on `hasAnyFilterApplied`, `:113-120`; the badge on `activeFilterCount > 0`, `:106-110`, calling `clearAllFilters` on click, `:118`.)

**Portal** — `Filters.tsx` (the portal UI never renders state dropdowns, so it does **not** reference the state arrays at all):

`hasAnyFilterApplied` (`Filters.tsx:103-110`): `selectedFilters`, price(from/to/free), order, `selectedRegionsAndCities.regions/.cities`.
`activeFilterCount` (`Filters.tsx:116-123`): same set (+ region/city counts). No state arrays.

**Field neither implementation counts: `searchText`.** Neither `hasAnyFilterApplied` nor `activeFilterCount` (in either component) inspects `searchText`, even though `clearAllFilters` resets it. Also, `DashboardFilters` does not count regions/cities (it renders no region/city filter), and `Filters` does not count states (renders no state filter) — so the two components are mirror-asymmetric. See adjacent observation A1.

---

## Q2 — Search submit category-context preservation

The portal-scope submit handler is `commitSearch` in `SearchInput.tsx:117-142`, invoked on Enter (`:144-150`) and on the "found" button (`:208`).

### Mechanism on a `/catalog/...` page (portal scope)

`commitSearch` does **not** read category IDs from any store. It reads **the current pathname** (`usePathname()`, `SearchInput.tsx:45`) and branches on it:

```
case 'portal' / default:
  return pathname.startsWith('/catalog') ? pathname : '/';   // SearchInput.tsx:137
```

So on a catalog page the navigation target **is the current pathname itself** (which already encodes the category slugs), and it appends the search text:

```
router.push(`${targetPath}?search_text=${toQueryParam(rawTerm)}`)   // SearchInput.tsx:141
```

It also writes `searchText` into the store first (`setSearchText(rawTerm)`, `:123`) and fires a `track('search', ...)` event (`:121`). Category context is preserved **purely by staying on the same path** — no category IDs are placed in the query string.

### Submit from home (not in a category)

`pathname.startsWith('/catalog')` is false, so target = `'/'` (`SearchInput.tsx:137`), i.e. `router.push('/?search_text=...')`. No category context (there is none).

### Dashboard / admin submit targets (one line each)

- owner: `'/owner/products'` (`SearchInput.tsx:133`)
- admin: `'/admin/products'` (`SearchInput.tsx:131-132`)

(then `?search_text=...` appended, `:141`.)

### Category IDs in autocomplete vs full search

- **Autocomplete request body: YES, category IDs are set** — from the pathname-derived `categories` state. `SearchInput.tsx:79-87` spreads `topCategoryId: categories.topCategory?.id`, `subCategoryId`, `finalCategoryId` into the autocomplete `productsFilter`. `categories` is computed from the pathname via `getCategoriesFromPath(baseSite.catalog, pathname)` (`SearchInput.tsx:61-63`). (Autocomplete `perPage` is `5`, `reactCalls/productsSearchService.ts:82-85`.)
- **Full-search request body: YES, but the IDs come from the route, set server-side — not from the search input and not from the store.** On the catalog page the server resolves the chain from the slugs and injects it into the full-search call (`catalog/[[...slugs]]/page.tsx:115-123`) and into the client list's `filtersData` prop (`:199-204`). The `commitSearch` handler itself puts **only** `search_text` in the URL (`SearchInput.tsx:141`); the category IDs re-enter on the next server render of the catalog route. Home/owner/admin full searches set **no** category IDs (`page.tsx:41`, `owner/products/page.tsx:25`, `admin/products/page.tsx:25`).

Note: the `FilterState` store has **no** category-ID fields at all (`useFilterStore.ts:11-43`). Category context lives entirely in the URL path, never in the store.

---

## Q3 — `/user/[userId]` filter-store participation + `excludeIds`

### Does the page read/write any filter Zustand store? — Effectively NO.

- The page component `user/[userId]/page.tsx` is a server component and imports/uses **no** filter store (`user/[userId]/page.tsx:1-20`, full file). It renders only `UserDetails`, `SelectableFilterProductListWrapper`, and `ExtraProductsComponent` — it does **not** render `SelectableFiltersWrapper`, `SelectableSearchInputWrapper`, or `SelectableSelectedFiltersDisplayWrapper`.
- One subtlety: the page lives under `app/[locale]/(portal)/(public)/layout.tsx`, which **does** mount `<SelectableFilterManagerWrapper portalScope="portal" />` (`(portal)/(public)/layout.tsx:11`). That mounts `FilterManager` against `usePortalFilterStore`. **But `FilterManager` short-circuits on this route**: its `isAllowedPath()` allows only `/`, `/owner/products`, `/admin/products*`, `/catalog*` (`FilterManager.tsx:54-72`); `/user/[userId]` is not in that set, so both the hydrate effect (`:77-79`, returns early) and the sync effect (`:230-232`, returns early) no-op. So no URL↔store sync runs on the user page.
- The product list (`SelectableFilterProductListWrapper` → `FilteredProductList` → `ProductList`) is driven by the **`filtersData` prop** (`{ ownerId: userId }`, `user/[userId]/page.tsx:101-104`), not by a store. `ProductList.tsx` subscribes to no filter store (verified: no `useFilterStore`/`usePortalFilterStore`/etc. import). Its pagination callbacks reuse the prop (`SelectableFilterProductListWrapper.tsx:56`).

**Conclusion:** the user page participates in no filter store for its behavior. The portal store object is mounted by the shared layout but FilterManager is inert on this path, and the list reads its filter from props.

### `ExtraProductsComponent` ("similar products") — YES, present.

Rendered at `user/[userId]/page.tsx:121-131`:

- Request body filter (`:123-126`): `{ ownerId: userId, excludeIds: productsData?.products?.map((prod) => prod.id) ?? [] }` — **`excludeIds` IS set**, to the IDs of the products shown in the initial main-list page (`getInitialPage()` first page, fetched at `:77`).
- Paging (`:127-130`): `{ page: 0, perPage: 10 }` — **`perPage` is `10`**.
- It POSTs to `/public/product/search` with `{ productsFilter, paging }` (`extraProductsService.ts:13-16`), fired in a `requestIdleCallback` + `startTransition` on mount (`ExtraProductsComponent.tsx:26-34`).

---

## Q4 — `applyRandom` / `randomSeed` suppression

Set in `parseFiltersFromQueryParams` (`filtersHelper.ts:38-73`), called by `filterHydrationSSR` (`FilterHydrationSSRInit.tsx:7-25`).

`randomAddOn = { applyRandom: true, randomSeed: Date.now() }` (`filtersHelper.ts:45-48`). It is included in exactly two places:

1. No query params at all + caller passed `applyRandom`: `if (params.length === 0 && applyRandom) return { ...randomAddOn };` (`filtersHelper.ts:49-51`).
2. Otherwise, only when neither searchText nor orderBy is present:
   ```
   if (applyRandom && !result.searchText && !result.orderBy) {
     result = { ...result, ...randomAddOn };   // filtersHelper.ts:68-70
   }
   ```

**So `applyRandom`/`randomSeed` are suppressed (omitted) when `searchText` is non-blank OR when `orderBy` is set.** That is the exact condition (`:68`).

Who passes `applyRandom = true`: only home (`page.tsx:40`) and catalog (`catalog/[[...slugs]]/page.tsx:113`). Owner/admin/admin-user pass `false` (`owner/products/page.tsx:24`, `admin/products/page.tsx:23`, `admin/products/[userId]/page.tsx:27`); the user page never calls the helper.

**Client-side suppression is in addition to server-side suppression.** The web code's own comment states the backend also ignores these fields, so emitting them is wire-noise, and adds a second client reason (seed instability remounting the list): `filtersHelper.ts:63-67`. So the web **both** relies on the server and suppresses client-side. (Backend not read — this is per the web-side comment.) Relatedly, `randomSeed` is stripped from the list-remount key but still travels to the backend (`FilteredProductList.tsx:29-35`).

---

## Q5 — State-filter label rendering (load-bearing B9 answer)

### Both dropdowns render **raw enum values** as option labels — **NOT translated.**

The product-state and moderation-state dropdowns are `SelectFilter` instances inside `DashboardFilters.tsx`. The option list is built with `label: ns` where `ns` is the raw enum string:

- **Product-state dropdown** (`DashboardFilters.tsx:125-134`):
  ```
  options={productStateOptions.map((ns) => ({ value: ns, label: ns }))}   // :130-133
  ```
  `label` is the raw enum value (`"ACTIVE"`, `"INACTIVE"`, `"DELETED"`). No translation lookup, no key pattern.
- **Moderation-state dropdown** (`DashboardFilters.tsx:136-145`, admin only):
  ```
  options={Object.values(ModerationState).map((ns) => ({ value: ns, label: ns }))}   // :141-144
  ```
  `label` is the raw enum value (`"APPROVED"`, `"BANNED"`). No translation lookup.

`SelectFilter` renders the option label verbatim — `{opt.label}` (`SelectFilter.tsx:57-61`). The **only** translated string inside `SelectFilter` is the synthetic "All" option, `t('filter.all.label')` from the `COMMON` namespace (`SelectFilter.tsx:38`, `:54`).

There is no key pattern such as `product.state.${state.toLowerCase()}` anywhere — the option text is the enum literal.

The **field labels** (the dropdown's own caption/placeholder) **are** translated, separately from the options:
- Product-state field label/placeholder: `tDash('product.state.filter.label')` (`DashboardFilters.tsx:126-127`).
- Moderation-state field label/placeholder: `tDash('moderation.state.filter.label')` (`DashboardFilters.tsx:137-138`).
- Namespace: `tDash = useTranslations(TranslationNamespaceEnum.DASHBOARD_PAGES)` (`DashboardFilters.tsx:30`). (`t` on the same component is `COMMON_SYSTEM`, `:29`, used for `filter.side.title`.)

### Full set of state values each dropdown offers

Driven by the scope router `SelectableFiltersWrapper.tsx`:

- **Owner** product-state dropdown: `[ProductState.ACTIVE, ProductState.INACTIVE]` (`SelectableFiltersWrapper.tsx:45`). No moderation dropdown (owner branch renders an empty `<div>` in place of it, `DashboardFilters.tsx:146-148`).
- **Admin** product-state dropdown: `Object.values(ProductState)` = `[ACTIVE, INACTIVE, DELETED]` (`SelectableFiltersWrapper.tsx:35`; enum at `ProductState.ts:1-5`).
- **Admin** moderation-state dropdown: `Object.values(ModerationState)` = `[APPROVED, BANNED]` (`DashboardFilters.tsx:141`; enum at `ModerationState.ts:1-4`).
- **Portal** (home/catalog/user): no state dropdowns at all (portal uses `Filters.tsx`, which renders none).

Each dropdown also prepends an "All" sentinel option (`includeAllOption` defaults `true`, `SelectFilter.tsx:35`, `:53-55`); selecting it calls back with `undefined`, which `DashboardFilters` maps to `clearProductStates()` / `clearModerationStates()` (`DashboardFilters.tsx:129`, `:140`).

---

## Trust-boundary pass (conventions Part 11) — fresh

Fields with ownership/scoping meaning on `ProductsFilterDTO`: `ownerId`, `productStates`, `moderationStates`. The web client never derives identity itself — scope is enforced by which backend endpoint a given page calls (`nextCalls/productsSearchService.ts:14-23`):

- portal → `POST /public/product/search`
- owner → `POST /secure/products`
- admin → `POST /secure/admin/products`

### `ownerId`

| Surface | Web sets `ownerId`? | Source | Endpoint |
|---|---|---|---|
| home `/` | no | — | `/public/product/search` |
| catalog | no | — | `/public/product/search` |
| owner `/owner/products` | **no** | derived server-side from auth | `/secure/products` |
| admin `/admin/products` | no | — | `/secure/admin/products` |
| admin-user `/admin/products/[userId]` | **yes** | URL path param: `filtersData.ownerId = parseNumber(userId)` (`admin/products/[userId]/page.tsx:28`) | `/secure/admin/products` (auth-gated) |
| public user `/user/[userId]` | **yes** | URL path param: `{ ownerId: userId }` (`user/[userId]/page.tsx:77`, `:104`, `:124`) | `/public/product/search` |

The owner dashboard deliberately does **not** send `ownerId` (`owner/products/page.tsx:25` passes `filtersData` with no ownerId) — the server scopes results to the authenticated owner. That is correct: the web does not let the client name "whose products" on the owner endpoint. On the admin-user and public-user surfaces `ownerId` is a client-supplied path integer; for admin it is gated by the `/secure/admin/*` endpoint, and for the public user page it is the intended public "this seller's listings" query against the public search.

### `productStates` / `moderationStates`

These flow from the URL query string, parsed **unconditionally** by `filtersHelper.ts:246-258` (`getProductStatesData` / `getModerationStatesData`) regardless of `portalScope`, then handed to whichever endpoint the page calls. The store's `allowProductStateFilter`/`allowModerationStateFilter` guards only gate the **UI setters** (`useFilterStore.ts:148-169`) and the `FilterManager` URL→store hydration (`setProductStates`/`setModerationStates`, `FilterManager.tsx:220-221`, which respect the guards) — they do **not** gate the SSR `filterHydrationSSR` parse.

**Part 11 flag (web side):** on the **portal/public** surfaces (home, catalog, user) the SSR parse will read and forward `?productStates=...` / `?moderationStates=...` from a hand-crafted URL straight into the body of `POST /public/product/search` — there is no client-side restriction to `ACTIVE`/`APPROVED` on the public endpoint. A user could append `?productStates=DELETED` or `?moderationStates=BANNED` to a public catalog URL and the web layer would forward it. The web does **not** make any authorization/moderation/state-transition decision from these — it only forwards them — so the server is relied upon to ignore/authorize client-supplied state arrays on the public endpoint. **This is the one place where a client-supplied scoping value reaches a public endpoint unfiltered on the web side; it is safe only if the backend restricts public search to visible states.** (Backend not read per brief — reported as "web relies on the server here.")

No web code reads `productStates`/`moderationStates`/`ownerId` to make a local trust decision; all three are forwarded to the server. Aside from the public-endpoint forwarding note above, **no other Part 11 concern found on the web side.**

---

## Also report

- **`ProductsFilterDTO` field list (as it exists today)** — `ProductsFilterDTO.ts:8-23`: `applyRandom?`, `randomSeed?`, `ownerId?`, `topCategoryId?`, `subCategoryId?`, `finalCategoryId?`, `excludeIds?`, `searchText?`, `filters?` (`RequestSelectedFilterDTO[]`), `orderBy?` (`OrderTypeDTO`), `priceRange?` (`PriceRange`), `selectedRegionsAndCities?`, `productStates?` (`ProductState[]`), `moderationStates?` (`ModerationState[]`). All optional.
- **Page-size constant** — `PRODUCTS_PER_PAGE = 20` (`utils.ts:8`); `getInitialPage()` returns `{ page: 0, perPage: 20 }` (`utils.ts:15-20`). The "similar products" carousel overrides this with `perPage: 10` (`user/[userId]/page.tsx:129`); autocomplete uses `perPage: 5` (`reactCalls/productsSearchService.ts:84`).
- **Pagination model** — **button-based numbered pages, 0-indexed internally, displayed `index + 1`.** `ProductList.tsx:155-168` renders one button per page; `totalPages = Math.ceil(total / PRODUCTS_PER_PAGE)` (`:76`); paging shows only when `totalPages > 1` (`:78`). Page 0 returns the SSR `initialProductsData` without a refetch (`SelectableFilterProductListWrapper.tsx:52-53`, `:64-65`, `:76-77`); other pages refetch via the scope's `onNextPage` with an `AbortController` (`ProductList.tsx:80-101`), then scroll to top (`:100`). Not infinite scroll. The `/favorites` route has a special re-fetch-page-0 effect keyed on `favoriteIds` (`ProductList.tsx:107-132`).

### Adjacent observations (Part 4b)

- **A1 — `searchText` is reset by `clearAllFilters` but excluded from both `hasAnyFilterApplied` and `activeFilterCount` in both `Filters.tsx` and `DashboardFilters.tsx`.** Files: `Filters.tsx:103-123`, `DashboardFilters.tsx:73-89`. Effect: if the only active criterion is a text search, the sidebar trash icon does not appear and the count badge stays at 0, yet a search chip shows elsewhere (`SelectedFiltersDisplay`) and the search clears the search text too. Severity: low (could mildly mislead; searchText has its own chip affordance). Out of audit scope.
- **A2 — Mirror asymmetry between the two count implementations.** `Filters.tsx` (portal) counts region/city but not states; `DashboardFilters.tsx` (owner/admin) counts states but not region/city. This is fine given each renders only its own filters, but it means the "active filter" semantics differ by scope and are duplicated rather than shared. Files: `Filters.tsx:103-123`, `DashboardFilters.tsx:73-89`. Severity: low. Out of scope.
- **A3 — `getProductStatesData`'s internal variable is misnamed `moderationStates`** while parsing the `productStates` param (`filtersHelper.ts:246-250`). Cosmetic; behavior correct (it reads `'productStates'`). Severity: low. Out of scope.
- **A4 — SSR state-array parse is not scope-gated** (see Part 11 flag). Files: `filtersHelper.ts:246-258` vs the per-scope endpoint in `nextCalls/productsSearchService.ts:14-23`. Severity: medium if the backend public search does not independently restrict states; low if it does. Out of scope (backend not read).

---

## Notes on what could not be determined

- Whether the backend actually ignores client-supplied `applyRandom`/`randomSeed` (Q4) and restricts public-search state arrays (Part 11) was **not verified** — the brief forbids reading the backend. Reported strictly as what the web code does and what its comments assert.

No code was changed during this audit.
