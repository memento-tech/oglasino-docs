# Investigation — Does changing a filter in the dialog refresh the product list?

**Verdict (one line):** **NO.** An in-place `FiltersDialog` filter change does **not** auto-refresh the product list. The new filters are assembled into a fresh fetch closure, but nothing invokes it — the list only updates on pull-to-refresh, pagination, a language switch, a favorites reload, or a screen remount (e.g. search-submit navigation).

**Repo / branch:** `oglasino-expo` / `new-expo-dev`
**Mode:** Read-only. No code changed.
**Date:** 2026-05-31

---

## TL;DR mechanism

1. The dialog writes the new filter values into `useFilterStore` (e.g. `setOrder`, `setPriceRange`, `addRemoveOptionFilter`).
2. `FilteredProductList` **does** observe the store, so it recomputes `filtersData` and rebuilds `fetchPageInternal` with a **new identity** on every filter change. ✅
3. `FilteredProductList` passes that new closure to `<ProductList fetchPage={fetchPageInternal} />` — but with **no `key` prop**, so `ProductList` does not remount.
4. Inside `ProductList`, the only effect that triggers a load (`onRefresh()`) is **ref-guarded to fire exactly once per mount**. When `fetchPage`'s identity changes on a filter change, the effect re-runs but returns early, so **no fetch happens**.
5. There is **no** `useFocusEffect`, no dialog-close refresh callback, and no refresh/version counter in the store that would otherwise force a re-fetch.

The audit's Q5 hypothesis is **confirmed**.

---

## File:line evidence

### Step 1 — Dialog writes to the store, no refresh on close
- `src/components/dialog/dialogs/FiltersDialog.tsx` — filter controls (`ProductOrder`, `PriceFilter`, `RegionCityFilter`, option/range filters) call store setters directly; the close button only calls the `onClose` prop.
- `src/components/dialog/DialogManager.tsx:83-88` — on close, with no custom `onClose` supplied (none is passed where the FILTERS dialog is opened), it just calls `closeDialog()`. **No refresh callback is wired to dialog close.**
- `src/lib/store/useFilterStore.ts:154-159` — `setOrder`, `setPriceRange`, `setRegionCityValues` simply `set({...})`; `:65-152` option/range setters likewise. There is **no** refresh trigger, version counter, or "dirty" flag in the store.

### Step 2 — `FilteredProductList` correctly reacts to the store (this part works)
- `src/components/product/FilteredProductList.tsx:42-58` — subscribes to all filter fields via `useFilterStore(useShallow(...))`.
- `:60-98` — `filtersData` memo depends on every filter field, so it recomputes when any filter changes.
- `:100-119` — `fetchPageInternal` is a `useCallback` with deps `[filtersData, searchText, selectedOrder]`, so its **identity changes** whenever filters change. The new closure correctly carries the new filters.

### Step 3 — No remount path
- `src/components/product/FilteredProductList.tsx:124-136` — `<ProductList ... fetchPage={fetchPageInternal} ... />` is rendered **without a `key` prop**, so a filter change does not remount `ProductList`.

### Step 4 — The decisive guard in `ProductList` (this is why it doesn't refresh)
- `src/components/product/ProductList.tsx:71` — `const hasLoadedInitiallyRef = useRef(false);`
- `:103-117` — the **only** effect that calls `onRefresh()` (full reset + page-0 fetch):
  ```tsx
  useEffect(() => {
    if (!selectedLanguage || hasLoadedInitiallyRef.current) return;  // :111 — early return
    hasLoadedInitiallyRef.current = true;                            // :112
    onRefresh();                                                     // :113
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [selectedLanguage, fetchPage]);                                 // :117
  ```
  `fetchPage` **is** in the dependency array, so the effect *does* re-run when filters change the closure identity — but line 111 returns early because `hasLoadedInitiallyRef.current` is already `true` after the first load. **Net result: no fetch on filter change.**

### Step 5 — The other fetch paths exist but are not invoked by a filter change
All three capture the latest `fetchPage`, so they *would* use the new filters — but nothing calls them when filters change:
- `:73-101` `loadNextPage` (deps `[fetchPage, hasMore]`) — invoked only by `onEndReached` → `:263`.
- `:119-131` `onRefresh` — invoked only by FlatList pull-to-refresh `:262` (and the favorites effect `:59-64`).
- `:133-147` language-change effect → `:149-205` `onRefreshCurrent` — invoked only when `selectedLanguage?.code` changes.

There is no effect anywhere keyed on `fetchPage` / `filtersData` identity that fires a refresh after the initial load.

---

## Scenario distinction (brief item 3)

- **Search submit → works (via remount).** `hasLoadedInitiallyRef` is a `useRef`, so it resets to `false` on every fresh mount of `ProductList`. Any path that remounts the screen — navigation, including submitting a search that pushes/replaces the catalog route — produces a brand-new `ProductList` whose initial-load effect fires with the **current** store values (including the new `searchText`). This is the mechanical reason search *appears* to work. (Consistent with the prior confirmation noted in the brief; the remount-resets-the-ref mechanism is confirmed directly in `ProductList.tsx:71,111`.)
- **In-place dialog filter change → does NOT work.** No remount, ref already `true`, no other trigger → stale list until a manual pull-to-refresh, pagination, language switch, or navigation away-and-back.

---

## Secondary observation (related, worth flagging for the fix brief)

Because `loadNextPage` (`ProductList.tsx:73-101`) captures the **latest** `fetchPage`, if a user changes a filter in the dialog and then **scrolls to paginate** (without pull-to-refresh), the next page is fetched with the **new** filters and **appended onto the existing old-filter results** — `pageRef`/`requestedPagesRef` are not reset on a filter change. The resulting list is a mix of old-filter and new-filter items. This is a direct consequence of the same missing "refresh-on-filter-change" trigger and should be covered by the same fix.

---

## Where a fix would go (no fix applied this session)

The missing piece is a trigger that, **after** the initial load, resets the feed and re-fetches page 0 when the filter-derived `fetchPage` identity changes. Two viable shapes:

1. **Recommended — a dedicated effect in `ProductList.tsx`** keyed on `fetchPage`, separate from the initial-load effect:
   - After `hasLoadedInitiallyRef.current` is `true`, when `fetchPage` identity changes, call `onRefresh()` (full reset to page 0 — the correct semantics for a *filter* change, distinct from the language-switch path which deliberately uses `onRefreshCurrent` to preserve pagination).
   - Must be guarded so it does **not** double-fire alongside the initial-load effect on first mount (e.g. track the last-seen `fetchPage` in a ref, or only run `onRefresh` when the ref shows a *change* after the first load).
   - This keeps the language-switch behavior (`:133-147` → `onRefreshCurrent`, preserve scroll) intact while making filter changes reset-to-top.

2. **Alternative — remount via `key` in `FilteredProductList.tsx:124`:** pass `key={<stable hash of filtersData>}` to `<ProductList>`. Simplest to reason about, but throws away scroll position and remounts the whole list/FlatList on every filter change (heavier; also resets favorites/extra-section state).

The choice between "reset to page 0" (`onRefresh`) and "refetch loaded pages in place" (`onRefreshCurrent`) is a UX decision for the follow-up brief; for a filter change, reset-to-top (`onRefresh`) is the conventional behavior. The store has no refresh/version field today (`useFilterStore.ts`), so option 1 relies on the existing `fetchPage`-identity signal and needs no store change.

---

## Method note

`ProductList.tsx`, `FilteredProductList.tsx`, and `useFilterStore.ts` were read directly. `FiltersDialog.tsx` and `DialogManager.tsx` line references were obtained via a read-only sub-agent trace and are consistent with the store/component code read directly. The verdict depends on the `ProductList` ref-guarded effect, which was confirmed first-hand.
