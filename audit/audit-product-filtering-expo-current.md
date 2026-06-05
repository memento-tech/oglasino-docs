# Audit — Expo Filtering & Search Current State

**Repo:** oglasino-expo
**Branch:** new-expo-dev (confirmed `git rev-parse --abbrev-ref HEAD` → `new-expo-dev`)
**Mode:** READ-ONLY. No code changed.
**Date:** 2026-05-31

All `file:line` below were read and verified on `new-expo-dev` this session.

---

## Q0 — Admin reality — VERIFIED

**Admin is gone from mobile.** Repo-wide grep for `useAdminFilterStore` /
`AdminFilter` across `src/` and `app/` → **zero** matches.

- **`useAdminFilterStore`:** does not exist / not exported anywhere.
- **Admin product-list screen:** none (admin removed in chat α; `git status`
  shows `app/admin/*` deleted).
- **Filter stores** — `src/lib/store/useFilterStore.ts`:
  - Factory: `createFilterStore = (allowProductStateFilter, allowModerationStateFilter) => ...` — `useFilterStore.ts:41`.
  - `usePortalFilterStore = createFilterStore(false, false)` — `useFilterStore.ts:201`.
  - `useDashboardFilterStore = createFilterStore(true, false)` — `useFilterStore.ts:203`.
  - **Portal** allows neither product- nor moderation-state; **dashboard** allows
    product-state only. **Moderation-state is allowed by NO store.**
- **Moderation-state dropdown:** **renders nowhere.** The only state dropdown is a
  `ProductState` `SelectFilter`, gated on dashboard scope — `FiltersDialog.tsx:179-189`.
  No `ModerationState` UI. `SelectedFiltersDisplay.tsx:137-141` *would* render a
  moderation chip if `selectedModerationStates` were non-empty, but nothing on
  mobile ever populates it. **Moderation-state is not a live surface on mobile** —
  its half of every finding below is vestigial. The `selectedModerationStates`
  field, its setters (`useFilterStore.ts:34-36,173-183`) and the body field
  `moderationStates` (`FilteredProductList.tsx:83`) are dead plumbing.

---

## Q1 — `clearAllFilters` coverage + active-filter count — VERIFIED

### `clearAllFilters` — `useFilterStore.ts:185-198`

```ts
clearAllFilters: () =>
  set({
    preparedFilters: {},
    searchText: undefined,
    selectedFilters: [],
    selectedOrder: undefined,
    selectedPriceRange: { from: '', to: '', selectedCurrency: undefined, free: undefined },
    selectedRegionsAndCities: { regions: [], cities: [] },
  }),
```

Resets: `preparedFilters`, `searchText`, `selectedFilters`, `selectedOrder`,
`selectedPriceRange`, `selectedRegionsAndCities`. **Does NOT reset
`selectedProductStates` or `selectedModerationStates`.** ✗ Divergence from target #1.
(Dedicated `clearProductStates` `:171` / `clearModerationStates` `:183` exist but
`clearAllFilters` does not call them.)

### Active-filter count / "any filter active" — four sites, all exclude the state arrays

1. `FloatingFiltersButton.tsx:26-38` — `selectedFiltersCount + price(from/to/free) + order + regions.length + cities.length`. Excludes both state arrays.
2. `Filters.tsx:98-110` — identical; excludes both.
3. `FiltersDialog.tsx:100-112` — identical; excludes both (even though it subscribes
   to `selectedProductStates` at `:46-52` for the dropdown).
4. `app/(portal)/(public)/catalog/[...categories].tsx:29-37` — `anyFilterActive`
   inspects `selectedFilters.length`, `selectedOrder`, `selectedPriceRange.from/to`,
   `selectedRegionsAndCities.regions/cities`. Excludes both state arrays. (Reads
   `usePortalFilterStore`, which has no product states anyway; used only to choose
   the empty-list message text at `:71-74`.)

**Consequence (✗ target #1):** on the **dashboard**, a product-state-only filter
yields `activeFilterCount === 0`, so the count badge shows nothing and the trash
icon (gated on `activeFilterCount > 0`, `FiltersDialog.tsx:171`) does not appear;
even if reached, `clearAllFilters` would not clear the product-state. Badge and
trash both ignore orphaned product-state filters.

---

## Q2 — Search submit & category context (load-bearing) — VERIFIED ✗ divergence

**On submit in portal scope, mobile navigates to the listing ROOT (`/`) and DROPS
the category.** ✗ Divergence from target #2.

The "full search" trigger is the **"see all results" footer button** inside the
autocomplete dropdown — `src/components/SearchInput.tsx:205-227`:

```ts
onPress={() => {
  setSearchText(debouncedTerm);            // :208 — writes searchText to the shared store
  setFocused(false); setInput(''); Keyboard.dismiss();
  const toRouteMap = { portal: '/', dashboard: '/owner/dashboard/products' } as const; // :213-216
  const toRoute = toRouteMap[currentPortalScope];
  router.push(toRoute);                     // :220 — portal → '/' (ROOT)
}}
```

- The `TextInput` has **no `onSubmitEditing`** (`SearchInput.tsx:166-179`) — the only
  full-search path is this footer button, shown only while the dropdown is open
  (`focused && debouncedTerm.length >= 3`, `:181`).
- **Category screen routing:** `app/(portal)/(public)/catalog/[...categories].tsx`
  is an expo-router **catch-all**. Categories are derived from `usePathname()` via
  `getCategoriesFromPath(catalog, pathname)` (`catalog:21-27`) — **not** from
  `useLocalSearchParams`. (Contrast the user screen, which *does* use
  `useLocalSearchParams` for `userData` — `user/[...userData].tsx:22-27`.)
- **Are category IDs available to the submit handler?** Yes. `SearchInput` computes
  `categories = getCategoriesFromPath(catalog, pathname)` (`SearchInput.tsx:64-68`)
  and `pathname` is in scope (`:53`). It already uses
  `categories?.topCategory?.id` etc. for **autocomplete** (`:89-94`). But the
  full-search navigation (`:213-220`) ignores them and pushes a static `/`.
- **Category IDs to autocomplete vs full search:**
  - Autocomplete: **yes** — `getAutocompleteSuggestions({ searchText, topCategoryId, subCategoryId, finalCategoryId, ...preparedFilters })`, `SearchInput.tsx:89-94`.
    (Endpoints: `productsSearchService.ts:45` portal `/public/product/search/autocomplete`,
    `:46` dashboard `/secure/products/autocomplete`.)
  - Full search: the category body fields are only ever populated by the **catalog
    screen's** `FilteredProductList` props (`catalog:60-64` →
    `FilteredProductList.tsx:62-64`). After submit pushes to `/` (the home/index
    screen), the destination has no category, so the full search runs category-less.
    The `searchText` survives (shared portal store), the category does not.

**The exact seam:** `SearchInput.tsx:213-220`. To preserve category in portal scope,
the submit should push the current category path (e.g. `router.push(pathname)` when
`pathname.startsWith('/catalog')`) instead of `/`. All inputs needed are already in
scope (`pathname`, `categories`).

---

## Q2b — Reconstructable from input vs navigation-only — VERIFIED → reconstructable

**Category context is path-derived everywhere, so it is reconstructable from input
(deep-link-compatible).**

- Every category derivation is `getCategoriesFromPath(catalog, pathname)`:
  `SearchInput.tsx:64-68`, `catalog/[...categories].tsx:23-27`, `Filters.tsx:39,66`,
  `FiltersDialog.tsx:36,68`.
- `getCategoriesFromPath` (`src/lib/utils/utils.ts:76-104`) is a **pure function of
  `(catalog, path)`**: it splits the path, and for each segment finds the catalog
  node whose `route === '/' + segment`, assigning top/sub/final by depth. No
  in-memory navigation state is consulted.
- **Route entry contract:** the catch-all path itself. A cold deep-link to
  `/catalog/<top>/<sub>/...` yields identical categories with no prior navigation.
- **Does the B7 fix stay deep-link-compatible?** Yes — carrying category forward by
  pushing the category **path** produces exactly the same input a deep-link entry
  would supply. No param-only / "must have come from the catalog screen" coupling.

**Verdict:** the current category-screen model is **reconstructable from input**
(deep-link-compatible). The only gap is behavioral (search submit discards it,
Q2), not structural.

---

## Q3 — User-products list filter-store participation — VERIFIED ✗ divergence

**The public user-products list DOES participate in the shared portal filter
store.** ✗ Divergence from target #3.

`src/components/product/UserProductsList.tsx` (exports `UserProductList`):

- Reads `usePortalFilterStore` (`:20-32`): `searchText`, `selectedFilters`,
  `selectedOrder`, `selectedPriceRange`, `selectedRegionsAndCities`.
- Builds the request body (`:38-64`): `ownerId: userDetails.id` (`:40`),
  `applyRandom: true` (`:41`), `searchText` (`:42`), `orderBy` (`:43`),
  `priceRange` (`:44`), `selectedRegionsAndCities` (`:45-48`), `filters` (`:49-53`).
  (No product/moderation states — portal store has none.)
- Fetches `getPortalProducts(dataWithSeed, { page, perPage: PRODUCTS_PER_PAGE })` (`:77`).

Because `usePortalFilterStore` is a module-level singleton (`useFilterStore.ts:201`),
**any portal filter set on the home/category screen bleeds onto the user-products
list.** It both reads the shared store **and** sets `ownerId` — target #3 wants
`ownerId`-only with no store participation.

---

## Q4 — Extra-products / "more from this seller" + excludeIds — VERIFIED

**The user screen's second list is the generic `RANDOM_PRODUCTS` carousel (no
`ownerId`, no `excludeIds`) — even though a purpose-built `MORE_FROM_SELLER`
section with both already exists. So `excludeIds` is fully plumbed; the user screen
simply mounts the wrong section.**

- `app/(portal)/(public)/user/[...userData].tsx:64` renders `<UserProductList>`.
- `UserProductList` passes `extraSections = [RANDOM_PRODUCTS]` (`UserProductsList.tsx:66`)
  to `ProductList` (`:87`).
- `ProductList` appends the extra section **only when `!hasMore`** (end of list) —
  `ProductList.tsx:214-225` (list assembly) and `:238-243` (renders
  `<HorizontalExtraProductsView sectionData=… resetSeed=…>`).
- `HorizontalExtraProductsView` (`src/components/product/HorizontalExtraProductsListView.tsx`)
  fetches `getPortalProducts(sectionData.filters, EXTRA_SECTION_PAGING)` (`:33`) —
  it forwards `sectionData.filters` verbatim, adding nothing.
- The section the user screen passes is `RANDOM_PRODUCTS`
  (`productExtraSections.ts:227-234`), whose `filters` are only `{ applyRandom,
  randomSeed }` — **no `ownerId`, no `excludeIds`**. So the carousel pulls random
  products site-wide and **can overlap the seller's own main list**.
- `excludeIds` is otherwise **fully implemented** across the extra-section catalog
  (`grep -rn excludeIds`): `SIMILAR_PRODUCTS` `:72`, **`MORE_FROM_SELLER` `:88`**,
  `MORE_FROM_TOP/SUB/FINAL_CATEGORY` `:142,:164,:186`, `POPULAR_IN_CATEGORY` `:212`,
  and the DTO field `ProductsFilterDTO.ts:15`.
- A ready-made **`MORE_FROM_SELLER(ownerId, excludeProductId)`** factory exists
  (`productExtraSections.ts:83-93`): `{ ownerId, excludeIds: [excludeProductId],
  applyRandom, randomSeed }` + `onSeeAll → /user/${ownerId}`. It is simply **not**
  the section the user screen mounts (the screen mounts `RANDOM_PRODUCTS`,
  `UserProductsList.tsx:66`).
- **Stale-comment flag (LOW):** `SIMILAR_PRODUCTS` `:67-76`'s doc-comment claims
  "Excludes products from the same owner," but its `filters` set only
  `finalCategoryId` + `excludeIds:[productId]` — there is **no** owner exclusion in
  the actual filter. Comment ≠ behavior.

**Decision input:** the `excludeIds`-deduped "more from this seller" section is **NOT
net-new infrastructure** — `MORE_FROM_SELLER` already exists end-to-end. The user
screen just mounts `RANDOM_PRODUCTS` instead. The fix is essentially a section swap
(`RANDOM_PRODUCTS` → `MORE_FROM_SELLER(userDetails.id, …)` at
`UserProductsList.tsx:66`), which also removes the current site-wide-random overlap
with the seller's main list. Open question for the fix chat: `MORE_FROM_SELLER`
takes an `excludeProductId`, but a seller page has no single "current product" —
that argument's source (or a no-exclude variant) needs deciding.

---

## Q5 — applyRandom / randomSeed suppression — VERIFIED ✗ (no suppression) + concrete refetch finding

**No client-side suppression. `applyRandom`/`randomSeed` are sent regardless of
`searchText` or `orderBy`, in both list components.** ✗ Divergence from target #4.

1. **`FilteredProductList.tsx:100-117`** — `fetchPageInternal`: `if (applyRandom) { fetchFilters.applyRandom = true; randomSeed = (page===0 && reinit) ? Date.now() : ref }`. `applyRandom` is a **prop** (`:31`); the catalog screen passes `applyRandom={true}` (`catalog:65`). No `searchText`/`selectedOrder` guard.
2. **`UserProductsList.tsx:38-77`** — `applyRandom: true` is **hard-coded** (`:41`); `fetchPage` sets `randomSeed: page===0 ? Date.now() : ref` (`:70-77`). No guard.

**Refetch wiring (this matters for the "redundant refetch" hypothesis):**
`ProductList`'s initial full load fires **once**, guarded by `hasLoadedInitiallyRef`
(`ProductList.tsx:102-116`, deps `[selectedLanguage, fetchPage]` but the ref returns
early on re-run, `:110-111`). There is **no effect keyed on `fetchPage`/`filtersData`
identity that re-fires the load.** Refetch happens only via: pull-to-refresh
`onRefresh` (`:118-130`), pagination `onEndReached → loadNextPage` (`:262`),
favorites change (`:58-63`), or language switch `onRefreshCurrent` (`:132-146`,
which re-fetches loaded pages with `reinitRandomSeed=false`, preserving the seed).

→ **A fresh `randomSeed` does NOT drive an automatic refetch storm in this
implementation** — the redundant-refetch smell the brief hypothesizes is not present
here. Suppressing the seed is still correct (parity, and it stops re-randomizing the
order on every manual refresh while a search/order is active), but it does not remove
an existing auto-refetch loop because there isn't one.

> **Adjacent observation (out of scope):** the same ref-guard means changing a
> filter via the dialog on the *same* screen does **not** auto-refresh the list —
> the new `filtersData`/`fetchPage` identity is never re-invoked. A new search
> appears to apply only because the submit *navigates* (remount), and dialog filter
> changes would need a pull-to-refresh. Verify this is intended; possible UX gap.
> (`ProductList.tsx:102-116`.)

---

## Q6 — State-label rendering — VERIFIED (parity)

**Parity, no action.** Product-state options and the selected-filter chips render
**raw enum strings**, not translation lookups.

- Dropdown — `FiltersDialog.tsx:184-187`: `Object.values(ProductState).map((ns) => ({ value: ns, label: ns }))` → `label` is the raw enum member. Field label uses `tDash('product.state.filter.label')` (`:181`) — a normal label key, not a per-value lookup.
- Chips — `SelectedFiltersDisplay.tsx:131-141`: product states via `ps.toString()` (`:133`), moderation states via `ps.toString()` (`:139`) — raw enum strings.
- **No blank/missing-key tripwire** — labels are the enum values themselves; they cannot resolve to empty.

**Inventory** (for a future i18n pass):
- `ProductState` (`src/lib/types/product/ProductState.ts:1-5`): `ACTIVE`, `INACTIVE`, `DELETED`. The dashboard dropdown offers all three (`Object.values`).
- `ModerationState` (`src/lib/types/product/ModerationState.ts:1-5`): `APPROVED`, `BANNED`. No dropdown (Q0) — would only ever appear as a chip, which nothing populates.

---

## Q7 — The `<div>` — VERIFIED (two sites, not one)

- `src/components/navigation/Filters.tsx:148` — `return <div key={filter.id}>Filter not found {filter.filterKey}</div>;`
- `src/components/dialog/dialogs/FiltersDialog.tsx:150` — same.

Both are the `else` branch of `getAppropriateFilter` (Filters.tsx:116-150;
FiltersDialog.tsx:118-152) — the fall-through when `filter.filterType` is not
`MULTI_OPTION`/`SINGLE_OPTION`/`RANGE`/`DATE`. Must become `<View>`+`<Text>`.

**Reachability:** triggered if a catalog filter has a `filterType` outside those
four. Note `SelectFilter` is imported in `FiltersDialog.tsx:8` but **not** dispatched
in `getAppropriateFilter` — so a `SELECT`-typed advanced filter would hit the
`<div>`. Low-frequency but reachable. (Prior hypothesis said only
`FiltersDialog.tsx:148`; the real lines are `Filters.tsx:148` **and**
`FiltersDialog.tsx:150`.)

---

## Q8 — setState-during-render — VERIFIED (real lines differ from old hypothesis)

Both filter components write state during render:

1. **`setIsFreeCategory` inside the `useMemo` body:**
   - `Filters.tsx:69` and `Filters.tsx:72` (inside `advancedFilters = useMemo(...)`, `:60-86`).
   - `FiltersDialog.tsx:71` and `FiltersDialog.tsx:74` (inside the `useMemo`, `:62-88`).
2. **Store setter called from the render body:**
   - `Filters.tsx:92-96` — `selectedFilters.forEach(... addRemoveOptionFilter(filterOption.filter, undefined))` (store setter `useFilterStore.ts:65`), runs every render.
   - `FiltersDialog.tsx:94-98` — identical.

Produces React update-during-render warnings / extra passes; (2) mutates the store
mid-render. (Prior `issues.md` hypothesis `Filters.tsx:86-90` / `FiltersDialog.tsx:87-91`
was in the right region; real lines above.)

> Note: `SearchInput.tsx:64-68` and `catalog/[...categories].tsx:23-27` also derive
> categories in a `useMemo`, but they do **not** call `setState` inside it — they are
> clean. The offenders are only the two filter components.

---

## Q9 — `parseFiltersFromQueryParams` as the deep-link seam — VERIFIED (caller-grep to confirm)

**Present, self-declared unused, shape-mismatched against the live store. Keep as a
seam — do not delete.**

- `src/lib/utils/filtersUtil.ts:33-49`, with `//TODO Currently not used... use it for deep links...` at `:33`.
- Supporting symbols (same file): `toQueryParam` `:9-18`, `fromQueryParam` `:20-31`,
  `getSearchText` `:51-61`, `getSelectedFilters` `:63-99`, `getPriceRangeData` `:101-112`,
  `getOrderData` `:114-120`, `getRegionAndCitiesData` `:122-142`.
- **Callers:** **zero — confirmed.** `grep -rn parseFiltersFromQueryParams src app`
  returns only the definition itself (`filtersUtil.ts:34`). Uncalled, as the TODO says.
- **Input:** `params: { key, value }[]` + `BaseSiteDTO` (`:34-37`) — exactly a decoded
  deep-link query-param list.
- **Output:** a **`ProductsFilterDTO`** (`:42-48`) with only
  `{ searchText, filters (RequestSelectedFilterDTO[]), orderBy, priceRange, selectedRegionsAndCities }`.
- **Drift a deep-link feature must reconcile:**
  1. **Wire shape ≠ store shape.** It emits the request DTO, not the store's
     `FilterState`. Field names differ: `filters` vs `selectedFilters`
     (`SearchSelectedFilter[]`, `useFilterStore.ts:15`); `orderBy` vs `selectedOrder`;
     `priceRange` vs `selectedPriceRange`. So the output is **not directly writable
     into the store** — a DTO→FilterState adapter is required.
  2. **Omits fields the store holds:** `selectedProductStates`,
     `selectedModerationStates` (`useFilterStore.ts:19-20`).
  3. **Omits fields the wire DTO supports:** `ownerId`,
     `topCategoryId/subCategoryId/finalCategoryId`, `excludeIds`, `applyRandom`,
     `randomSeed` (`ProductsFilterDTO.ts:9-15,21-22`). Category context (Q2b) is
     **not** produced here — it would still come from the path via
     `getCategoriesFromPath`.
- **Fitness verdict:** valid deep-link entry point, but currently emits the wire DTO
  and omits state filters + category IDs. A deep-link feature would add a
  DTO→FilterState adapter, decide category carriage (path vs params — Q2b says path
  works), and extend for product-state if/when a portal product-state filter ships.

---

## Trust-boundary note — VERIFIED (clean)

Mobile makes **no local trust decision** from a filter field. `FilteredProductList.tsx:60-114`
and `UserProductsList.tsx:38-77` assemble a `ProductsFilterDTO` (incl.
`productStates`/`moderationStates`/`ownerId`) and forward it to `getPortalProducts`
(the server). State/ownership/moderation are server-enforced; the client only
forwards. One line: clean.

---

## Also report

- **Debug placeholder shipped on the catalog screen (MEDIUM):**
  `app/(portal)/(public)/catalog/[...categories].tsx:67` passes
  `HeaderComponent={<Text>Test123</Text>}` — a literal "Test123" renders above the
  category product list. User-visible leftover debug content. Out of audit scope;
  flag for the fix chat / D.
- **`console.error` violating the logger convention (LOW):**
  `HorizontalExtraProductsListView.tsx:37`, `ProductList.tsx:94`, `ProductList.tsx:199`.
  (The two `ProductList` sites are already tracked in `issues.md` 2026-05-28; the
  `HorizontalExtraProductsListView:37` site is additional.) Out of scope.
- **Pagination model (VERIFIED divergence, not a bug):** infinite scroll —
  `ProductList.tsx` loads page-by-page (`loadNextPage`, `:72-100`), `perPage =
  PRODUCTS_PER_PAGE`, `onEndReached → loadNextPage(false)` (`:262`),
  `onEndReachedThreshold={1.8}`. Language switch re-fetches already-loaded pages
  preserving scroll (`onRefreshCurrent`, `:148-204`). Documented platform divergence
  from web's pagination buttons.
- **Vestigial moderation-state plumbing (LOW):** `selectedModerationStates` + its
  setters (`useFilterStore.ts:34-36,173-183`) + body field `moderationStates`
  (`FilteredProductList.tsx:83`) have no UI writer and no allowing store (Q0).
  Removal or future-admin candidate; flag for Ω.
- **`SelectFilter` imported but not dispatched (LOW→MEDIUM):** `FiltersDialog.tsx:8`
  imports it, used only for the dashboard product-state dropdown (`:180`); it is not
  a branch in `getAppropriateFilter`, so a `SELECT`-typed advanced filter falls
  through to the `<div>` (Q7). Couples to the `<div>` fix.
- **`ProductList` ref-guarded load (see Q5 adjacent):** dialog-set filters may not
  auto-refresh the list; verify intended.
- **Surprises vs the brief's prior hypotheses (all corrected above):** `<div>` is at
  `Filters.tsx:148` AND `FiltersDialog.tsx:150` (not `:148`); the setState-during-render
  is `setIsFreeCategory`-in-`useMemo` + render-body `forEach` setter; Q2 confirmed
  (portal submit → `/`, category dropped); Q3 confirmed (UserProductList reads the
  portal store); Q4 shows only a generic random carousel, no seller-scoped second
  list.

---

## Definition-of-done

Q0–Q9 (incl. Q2b), the trust-boundary one-liner, and the "also report" section are
answered with `file:line` evidence verified on `new-expo-dev`. All residuals are
closed: Q4 `excludeIds` is confirmed implemented end-to-end (incl. a ready
`MORE_FROM_SELLER` factory), and Q9 has zero callers (grep-confirmed). No code was
changed.
