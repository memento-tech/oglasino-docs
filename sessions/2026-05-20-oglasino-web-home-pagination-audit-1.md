# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-20
**Task:** Audit brief — Home page pagination broken: scope + root cause (READ-ONLY)

## Implemented

- Traced the home-page pagination data path from SSR fetch through the shared `ProductList` client component to the click handler.
- Catalogued the seven product-list surfaces that share the `ProductList` mechanism vs. the four admin surfaces that use the separate `PaginatedTable` mechanism vs. the three `loadMore`-style surfaces (follows, notifications, reviews).
- Did not edit code, did not run tests, did not touch backend.

## Files touched

- none (read-only audit)

## Tests

- Ran: none (read-only)
- Result: N/A

## Cleanup performed

- none needed (read-only)

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change drafted by this session — the existing 2026-05-19 entry is what triggered the audit. Mastermind may want to amend it with the diagnosis below before scoping the fix; that draft text is included in "For Mastermind."

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): N/A this session (read-only)
- Part 4a (simplicity): N/A this session — no code added; complexity observations in "For Mastermind"
- Part 4b (adjacent observations): one item flagged in "For Mastermind" (catalog deep-link `?page=N` never honored on SSR)
- Part 6 (translations): N/A this session
- Other parts touched: Part 7 (error contract) — N/A; Part 8 — N/A

## Known gaps / TODOs

- The audit cannot conclusively distinguish "backend ignores `page`" from a "web-side pagination UI clipped offscreen" interpretation of the symptom without observing the live page. Both readings are surfaced below with the evidence the code provides; final scoping needs either a manual reproduction step (browser screenshot of the home page with >20 products) or a backend audit.

---

# Audit: home page pagination

## 1. Home page diagnosis (with code-quoted evidence)

### 1.1 How `searchParams.page` flows into the data fetch

It doesn't. The home server component at `app/[locale]/(portal)/(public)/page.tsx:24-37` consumes `searchParams` only to build `filtersData`:

```ts
// app/[locale]/(portal)/(public)/page.tsx:33-37
const filtersData = await filterHydrationSSR(await searchParams, baseSite, t, true);
const initialProductsData: ProductOverviewsDTO = await getPortalProducts(
  filtersData,
  getInitialPage()
);
```

`getInitialPage()` is hardcoded at `src/lib/utils/utils.ts:15-20`:

```ts
export const getInitialPage = (): PagingDTO => {
  return {
    page: 0,
    perPage: PRODUCTS_PER_PAGE, // 20
  };
};
```

And `filterHydrationSSR` → `parseFiltersFromQueryParams` does not read a `page` query param at all (grepped `src/lib/utils/filtersHelper.ts` for `page|paging` — zero hits).

**So `?page=N` in the URL is ignored on every SSR fetch on the home page.** SSR always renders page 0 of the filter set.

### 1.2 Which fetch function does it call

The SSR fetch uses the Next-server variant of `getPortalProducts`:

```ts
// app/[locale]/(portal)/(public)/page.tsx:6
import { getPortalProducts } from '@/src/lib/service/nextCalls/productsSearchService';
```

That function (`src/lib/service/nextCalls/productsSearchService.ts:119-124`) is a thin wrapper around `getProducts` that POSTs `{ productsFilter, paging }` to `/public/product/search` via `FETCH_BACKEND_API.post` (line 94). No `cache: 'no-store'` / `next.revalidate` is set on this POST — POSTs are not cached by Next's Data Cache anyway, and the surrounding route is opted into dynamic rendering by `export const revalidate = 0` (`page.tsx:22`).

### 1.3 Which pagination UI renders, and where

The home page conditionally renders `SelectableFilterProductListWrapper` (`page.tsx:53-60`) when `totalNumberOfProducts > 0`. That wrapper lives at `src/components/client/product/SelectableFilterProductListWrapper.tsx`. For `portalScope="portal"`, it forwards `initialProductsData` plus an `onNextPortalPageInternal` callback into `FilteredProductList` (which adds `key={JSON.stringify(filtersData)}` and forwards into `ProductList`).

The pagination UI itself is **inside `ProductList`**, at `src/components/client/product/ProductList.tsx:150-164`:

```tsx
{showPaging && (
  <div className="flex justify-center gap-2 py-6">
    {Array.from({ length: totalPages }).map((_, index) => (
      <button
        key={index}
        onClick={() => onDisplayPageChange(index)}
        className={cn(
          'rounded border px-3 py-1',
          index === displayPage && 'bg-black text-white'
        )}>
        {index + 1}
      </button>
    ))}
  </div>
)}
```

`showPaging` (line 89) is `totalPages > 1`, where `totalPages = Math.ceil(initialProductsData.totalNumberOfProducts / PRODUCTS_PER_PAGE)`.

### 1.4 What happens when a user clicks "page 2"

Click → `onDisplayPageChange(1)` (`ProductList.tsx:91-104`):

```tsx
const onDisplayPageChange = async (page: number) => {
  setLoading(true);
  setDisplayPage(page);
  const result = await onNextPage({ page, perPage: PRODUCTS_PER_PAGE });
  setProducts(result.products);
  setLoading(false);
  window.scrollTo({ top: 0, behavior: 'instant' });
};
```

`onNextPage` is the `onNextPortalPageInternal` callback from the wrapper (`SelectableFilterProductListWrapper.tsx:32-38`):

```tsx
const onNextPortalPageInternal = (paging: PagingDTO): Promise<ProductOverviewsDTO> => {
  if (paging.page === 0) {
    return Promise.resolve(initialProductsData);
  }
  return getPortalProducts({ ...filtersData }, paging);
};
```

For `page !== 0`, this calls the **client/axios** variant of `getPortalProducts` (`src/lib/service/reactCalls/productsSearchService.ts:161-166`) — `BACKEND_API.post('/public/product/search', { productsFilter, paging })`. The result's `products` array replaces local state. **No URL update.**

So:
- The click handler **exists** and is wired.
- It **does** pass `page: 1` (etc.) up to the backend — there is no hardcoded `page=0` in the click path.
- The page does **not** refetch via Next.js navigation; pagination is a local-state swap.
- The URL is **not** updated, and `searchParams.page` is **never** read anywhere.

### 1.5 Which candidate from the brief matches

- **(a) Pagination controls don't render at all** — possible only if backend returns `totalNumberOfProducts <= 20`. Not verifiable from code alone.
- **(b) Controls render but click handler is missing/no-op** — *not the bug.* Click handler is present, wired, and passes the correct page index.
- **(c) Click updates URL but doesn't refetch** — *not the bug.* Click never updates URL; refetch is client-side axios into the page-2 result.
- **(d) Refetch always passes `page=0` regardless of URL state** — *not the bug.* `index` flows through correctly.
- **(e) Refetch passes the right page but backend returns page 0 anyway** — *plausible, but a backend-side problem.* The brief calls out that the 2026-05-14 sweep found three pagination-related backend issues (`perPage` cap, `parseFiltersFromQueryParams`, search-result swallowing). If the backend is silently returning page 0 for any `page > 0` request — for any of those three reasons or another — the web symptom would be exactly "only loads first page."
- **(f) Something else** — see two structural possibilities below.

**Structural possibility 1 — pagination row overflow.** The numbered-buttons row is `flex justify-center gap-2` with no `flex-wrap`, no `overflow-x`. For any filter set with more than ~20 pages the row overflows the viewport horizontally. With `justify-center` overflow is symmetric, so the page-2 button is clipped behind the left edge on narrow screens. A user can still click whatever sliver of "2" they can see; this matches "only loads first page" only if the buttons render so degenerately that they're effectively unclickable, which I cannot confirm without running the app.

**Structural possibility 2 — URL deep-link silently ignored.** Anyone arriving at the home page via a link like `/sr-rs?page=2` (e.g. from a search engine, share, or back-navigation if some other surface ever writes that param) gets page 0 silently. The symptom "only loads first page" would describe this exactly. The web side has never wired the SSR fetch to `searchParams.page`; the catalog page hasn't either (it uses `getInitialPage()` at `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx:112`). I don't see evidence anywhere that writes `?page=` into the URL, so this is a deep-link gap, not a regression from a recent change.

**Best read of the symptom given just the code I can see:** the click path is correct, so either (i) the backend is returning page 0 for `page > 0` requests, or (ii) the visible pagination UI is structurally unusable for typical product counts. Both deserve verification before scoping the fix.

## 2. Other paginated surfaces

| Surface | File | Mechanism | Same as home? | Same bug? |
| --- | --- | --- | --- | --- |
| Home | `app/[locale]/(portal)/(public)/page.tsx` | `SelectableFilterProductListWrapper` → `ProductList` numbered buttons, client-state pagination via reactCalls `getPortalProducts`, no URL update | (reference) | (reference) |
| Catalog list | `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx` | Same wrapper, same `ProductList`, `portalScope="portal"`. SSR fetch at line 105 hardcodes `getInitialPage()`. | yes — identical pagination path | yes — inherits whatever bug is in `ProductList` and identical SSR-ignores-`?page=` deep-link gap |
| Public user products | `app/[locale]/(portal)/(public)/user/[userId]/page.tsx` | Same wrapper, same `ProductList`, `portalScope="portal"`. SSR `getPortalProducts({ ownerId: userId }, getInitialPage())` at line 45. | yes | yes |
| Search / autocomplete | `src/components/.../GlobalSearchInput*` (autocomplete only — top-5 results dropdown via `getPortalAutocompleteSuggestions`, hardcoded `paging = { page: 0, perPage: 5 }` at `src/lib/service/reactCalls/productsSearchService.ts:77-80`). There is no dedicated `/search` page; search submission routes to the catalog/home with filters applied. | autocomplete is intentionally non-paginated; search-results page = catalog/home | — | inherits home/catalog behavior |
| Favorites | `app/[locale]/(portal)/(protected)/favorites/page.tsx` → `src/components/client/FavoriteProductList.tsx` | Same `ProductList` component but called via `FavoriteProductList`. `fetchPage` always calls `getFavorites(paging)` (no `if (page === 0) return initial`). Pagination otherwise identical. | uses the same `ProductList` mechanism | yes — if the bug is in `ProductList`, favorites inherits it. If the bug is the wrapper's `if (paging.page === 0) return Promise.resolve(initialProductsData)` short-circuit, favorites does **not** inherit it. |
| Owner dashboard products | `app/[locale]/owner/products/page.tsx` | Same wrapper, same `ProductList`, `portalScope="dashboard"`, uses `getDashboardProducts`. | yes | yes |
| Admin products list | `app/[locale]/admin/products/page.tsx` | Same wrapper, `portalScope="admin"`, uses `getAdminProducts`. | yes | yes |
| Admin user-specific products | `app/[locale]/admin/products/[userId]/page.tsx` | Same wrapper, `portalScope="admin"`. | yes | yes |
| Owner dashboard reviews | `app/[locale]/owner/reviews/page.tsx` → `src/components/owner/reviews/OwnerReviewList.tsx` | "Load more" button (`OwnerReviewList.tsx:32-43`) appending to a `useState` array, tracks `page` locally, calls `getMyReviews(pageToLoad, type)`. | no — different mechanism | no — independent of `ProductList` |
| Owner dashboard follows | `app/[locale]/owner/follows/page.tsx` | "Load more" button, appends rows, `getUserFollows(paging)` per click. | no — different mechanism | no |
| Notifications | `app/[locale]/(portal)/(protected)/notifications/page.tsx` → `notifications/hooks/useNotifications` | Firestore-driven store with `loadMore` / `hasMore`. | no — Firestore, not backend search | no |
| Admin users / reports / suggestions / reviews | `app/[locale]/admin/{users,reports,suggestions,reviews}/page.tsx` → `src/components/admin/PaginatedTable.tsx` | shadcn `<Pagination>` with prev / next / windowed page numbers, `useState(currentPage)` + `useEffect([currentPage, ...])` fetch via the table's `onFetchPage`. Hardcodes `page: 0` in the **SSR** props (`UsersPage` etc.), but the client `PaginatedTable` then re-fetches on `currentPage` change. Same "URL never carries `page`" gap exists, but the click path is independent. | no — different mechanism (`PaginatedTable`, not `ProductList`) | independent — would need its own check |
| Admin chats / translations / cache / config / es / statistics | listed; not investigated in depth for this audit beyond confirming they don't share `ProductList`. The `StatsPaginatedTable` exists separately. | various | no | no |

## 3. Shared pagination components

Two distinct shared components:

### 3.1 `ProductList`

- **File:** `src/components/client/product/ProductList.tsx`
- **Consumers:** home, catalog, public user products, favorites, owner products, admin products, admin user-products — seven surfaces.
- **Props:** `initialProductsData`, `initialCardSize`, `ProductCardComponent`, `onNextPage(paging)`, `isDashboard?`, `isLarge?`.
- **"Next page" computation:** local `useState(displayPage)`, click handler `onDisplayPageChange(index)` calls `await onNextPage({ page: index, perPage: PRODUCTS_PER_PAGE })` then `setProducts(result.products)`. **No URL update. No history push. No `router.push`.**
- **`page: 0` hardcodes** in this component:
  - Line 82: `const [displayPage, setDisplayPage] = useState(0);` — initial render correctly assumes SSR was page 0.
  - Line 116: `await onNextPage({ page: 0, perPage: PRODUCTS_PER_PAGE });` — inside the `/favorites` pathname-gated `useEffect` that refetches when `favoriteIds` changes. This forces favorites page back to page 0 whenever the favorites set changes; that may be intentional (user toggled a favorite, list semantics changed) but it does mean favorites page-2 navigation is wiped any time the favorites Zustand store updates.
  - Line 133: `setDisplayPage(0)` inside `useEffect([initialProductsData])` — resets local pagination state if SSR delivers a new `initialProductsData` reference. Stable in normal navigation; relevant only if the parent re-renders with a new prop reference.

### 3.2 `PaginatedTable`

- **File:** `src/components/admin/PaginatedTable.tsx`
- **Consumers:** admin users, reports, suggestions, reviews — four admin tables.
- **Props:** `params` (with optional `page`), `onFetchPage`, `renderHeader`, `renderRow`, `emptyMessage`.
- **"Next page":** `setCurrentPage(page)` triggers `useEffect([currentPage, onFetchPage, JSON.stringify(params)])` to refetch. shadcn `<Pagination>` UI with windowed page numbers (`VISIBLE_PAGE_NUMBERS = 3`).
- **`page: 0` hardcodes:** none in this component. The SSR pages set `page: 0` in `params` (e.g., `app/[locale]/admin/users/page.tsx:14-15`), and `PaginatedTable` reads `params.page ?? 0` for its initial currentPage. Client clicks update local state and refetch independently.

## 4. Backend interaction sanity check

Grepped the web side for hardcoded `page: 0` against the backend on the page-2 path. Hits:

- `src/lib/utils/utils.ts:17` — `getInitialPage()` returns `{ page: 0, perPage: 20 }`. Used **only** for SSR initial fetches across the seven surfaces above and for `ExtraProductsComponent` (which is not paginated). The click path does not go through `getInitialPage()`.
- `src/lib/service/reactCalls/productsSearchService.ts:77-80` — autocomplete suggestions hardcode `paging = { page: 0, perPage: 5 }`. Intentional; autocomplete dropdown is non-paginated.
- `src/components/client/product/SelectableFilterProductListWrapper.tsx:33,42,50` — `if (paging.page === 0) return Promise.resolve(initialProductsData)`. This short-circuits a network call when the user clicks back to page 1; it correctly forwards non-zero pages.
- `app/[locale]/admin/{users,reports,suggestions,reviews}/page.tsx` — `page: 0` is set in the SSR params object. `PaginatedTable` then uses its own state, so this hardcode does not bleed into page-2 fetches.
- `app/[locale]/(portal)/(public)/user/[userId]/page.tsx:82-86` and `(portal)/(protected)/favorites/page.tsx:55-59,67-71` — `ExtraProductsComponent` paging hardcoded `{ page: 0, perPage: 10 }`. Not paginated, fixed-size related-products list.

**Conclusion:** the web side does **not** hardcode `page: 0` in any path that runs when the user clicks the "page 2" pagination button. The numeric page index reaches the axios call body correctly. If the backend then returns page 0 anyway, the bug is server-side.

## 5. Recent-change correlation

The 2026-05-19 User Deletion ship landed the `/api/revalidate` cache-profile fix (`expire: 0` for immediate tag invalidation). Examined four candidate interaction paths:

1. **`revalidate: N` directives on the home fetch.** Home has `export const revalidate = 0` (`page.tsx:22`) — dynamic, no Data Cache. The product-list POST does not pass `next.revalidate` or `next.tags`. **No interaction.**
2. **`revalidateTag` consumers that touch home page data.** `revalidateTag('product', ...)` and `revalidateTag('product:N', ...)` are emitted by the cache-invalidation endpoint, but the home page's product-list fetch is a POST without `next.tags`, so it isn't tagged into the Data Cache and isn't subject to those invalidations. **No interaction.**
3. **Next.js Data Cache `revalidateTag(tag, { expire: 0 })` semantics.** The 2026-05-19 fix only affects responses that were stored in the Data Cache, which the product-list POST is not. **No interaction.**
4. **`useState` that holds the rendered list and might not reset on URL change.** `ProductList.tsx:81` holds `products` in `useState`. The `useEffect` at line 131-134 resets it when `initialProductsData` reference changes. `FilteredProductList` re-keys `ProductList` on `JSON.stringify(filtersData)` (line 28). Pagination clicks do not change the URL, so this is not exercised by clicking page 2. **No interaction observed.**

Most recent commit touching the home page is `0c8bf76` (2026-05-17 — "New changes, many bugs resolved"). It simplified the SSR fetch by removing the `applyRandom` fallback branch but did not change the pagination path. No pagination-related commits since. **No recent-change correlation found on the web side.**

## 6. Proposed fix

**Diagnosis (one sentence):** the web side passes the page index correctly through the click → axios path with no hardcoded `page=0` along the way, so the home-only symptom is either (a) a backend-side bug returning page 0 for any `page > 0` request — fits the existing pagination-adjacent backend issues — or (b) a structural defect in the `ProductList` numbered-buttons UI that renders all `totalPages` buttons in a non-wrapping centered row, which clips at typical product counts and could be misread as "only page 1 loads." I cannot decide between these without runtime evidence.

**Scope:** if the bug is on the web side, it is **shared-component** — fixing `ProductList` (or its caller-wrapper) cascades to seven surfaces: home, catalog, public user products, favorites, owner products, admin products, admin user-products. The four admin tables (users/reports/suggestions/reviews) and the three load-more surfaces (follows, notifications, reviews) are independent.

**Fix shape — three concrete options to consider, depending on the runtime finding:**

1. **If runtime shows "click page 2 makes a request with `page=1` but backend response carries page-0 product IDs":** scope is backend-only. Route to a backend audit of `/public/product/search` page handling; the web side requires no change. (Brief explicitly out-of-scope for this audit.)

2. **If runtime shows "the page-2 button exists but is clipped/unclickable":** scope is `src/components/client/product/ProductList.tsx:150-164`. Replace the inline `Array.from({length: totalPages}).map(...)` with the same windowed `<Pagination>` (shadcn) already used by `PaginatedTable` — `Pagination`/`PaginationContent`/`PaginationPrevious`/`PaginationLink`/`PaginationNext` from `components/shadcn/ui/pagination.tsx`, with a `VISIBLE_PAGE_NUMBERS = 3` (or 5) window. The click handler stays the same. One file changes; seven surfaces inherit the fix.

3. **If runtime shows "URL `?page=2` is the actual use case":** scope is the home page's SSR fetch (and likely the matching catalog/user/owner/admin pages). Read `searchParams.page`, default to 0 on parse failure, pass it into the SSR `getPortalProducts` call. Then update `ProductList` so the client click also `router.push(?page=N)` and the SSR refetch is the source of truth. This is the larger refactor; recommend only if Mastermind genuinely wants URL-driven pagination (good for SEO + deep links).

**Recommended first step:** Igor (or a follow-up brief) reproduces with browser DevTools Network tab open on the home page with a filter that yields > 20 products, clicks "page 2," and reports (a) does the POST body contain `paging: { page: 1, ... }`, and (b) what page does the response actually carry. That single observation picks (1) vs (2) vs (3) deterministically.

## 7. Risks

- **Re-rendering with `PaginatedTable`-style pagination on the seven product-list surfaces** (option 2) is low-risk — `PaginatedTable` is already in production for the four admin tables and behaves identically. Only the visual chrome differs.
- **Wiring `?page=` into SSR on the home page** (option 3) interacts with `revalidate = 0` cleanly (the page is dynamic anyway), but on the catalog page interacts with metadata generation (`generateCatalogPageMetadata` reads `searchParams` for SEO). Need to confirm `?page=N` does not cause duplicate-canonical SEO issues across paginated views before shipping option 3.
- **Favorites' `pathname === '/favorites'` `useEffect` in `ProductList.tsx:124-126`** resets pagination to page 0 whenever `favoriteIds` changes. If option 2 (or 3) is taken, that branch should be re-validated — toggling a favorite while on page 2 of favorites currently kicks the user back to page 1 silently. Out of scope here; medium severity if anyone has noticed.
- **Catalog and public-user-products pages share `ProductList` and the SSR-ignores-`?page=` gap** identically. Whatever decision is made for home applies to those two. The brief flagged this possibility; the code confirms it.

## 8. Open questions

1. **Symptom precision.** Does "only loads first page" mean (a) clicking page 2 changes the highlighted button but the products are still page-1 items, (b) clicking page 2 does nothing visible at all, (c) there are no pagination buttons visible, or (d) the URL `/sr-rs?page=2` shows page 1? The web code reads differently against each.
2. **`totalNumberOfProducts` in production.** If the dataset on the testing environment had `<= 20` products, `showPaging` would be `false` and there would simply be no pagination UI to test. Worth confirming before treating this as a "page 2 broken" bug — it might be "page 2 never rendered because the test data has too few products."
3. **Is URL-driven pagination intended?** The product team may have an opinion on `?page=` for SEO / deep-link / back-button behavior. The current pure-client-state pagination has none of that; option 3 adds it but is wider.
4. **Was anything observed in DevTools at smoke-test time** — network tab, console error, response shape — that would narrow this past code inspection? The audit assumes none was captured.

---

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit)
  - Considered and rejected: nothing
  - Simplified or removed: nothing

- **Recommended `issues.md` 2026-05-19 entry amendment (draft, for Docs/QA to apply if Mastermind agrees):**
  > Web-side click path correctly passes the numeric page index from `ProductList.onDisplayPageChange` through `SelectableFilterProductListWrapper.onNextPortalPageInternal` into the axios POST body — no hardcoded `page=0` on the page-2 path. If the symptom is "the request leaves the browser with `paging.page=1` but the response carries page-0 products," route to a backend audit (potentially related to the 2026-05-14 sweep entries on `parseFiltersFromQueryParams` and search-result swallowing). If the symptom is "the page-2 button isn't reachable in the UI," scope a fix on `src/components/client/product/ProductList.tsx:150-164` to replace the un-windowed flex row with the shadcn `<Pagination>` already used by `src/components/admin/PaginatedTable.tsx`. Seven surfaces share the `ProductList` mechanism and would inherit either fix: home, catalog, public user-products, favorites, owner products, admin products, admin user-products.

- **Adjacent observation (Part 4b, medium severity — out of scope for this audit, not fixed):**
  - **No surface on the web side reads `?page=` from `searchParams` when SSR-fetching products.** Affects home, catalog (`[[...slugs]]`), and public user-products. Deep-links to `?page=2` silently render page 1. This is not a regression — it has always been this way. Worth a separate `issues.md` entry if and only if the product team wants URL-driven pagination. If not, the gap is a deliberate non-feature and the audit shouldn't escalate it. Flagging here so Mastermind can decide.

- **Adjacent observation (Part 4b, low severity — out of scope, not fixed):**
  - **Favorites `useEffect([favoriteIds])` in `ProductList.tsx:110-127` resets pagination to page 0 when the user toggles any favorite while on a non-first page.** Pathname-gated to `/favorites` so it does not affect home/catalog. Likely intentional (favorites set changed → restart from page 0) but the silent reset may surprise users. Mastermind judgment on whether to flag in `issues.md`.

- **Adjacent observation (Part 4b, low severity — out of scope, not fixed):**
  - **Owner reviews "Load more" mechanism** (`src/components/owner/reviews/OwnerReviewList.tsx`) and **owner follows "Load more" mechanism** (`app/[locale]/owner/follows/page.tsx`) are structurally similar but not shared. If a future brief consolidates load-more patterns, these two would be the seed.

- **Closure gate confirmation:** no config-file edits required from this session. The `issues.md` amendment above is a *draft for Mastermind to consider*, not a pending unstated dependency — if Mastermind chooses not to amend, no Docs/QA work is queued. Stating explicitly to clear the gate.
