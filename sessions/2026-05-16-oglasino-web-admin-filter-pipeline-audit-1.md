# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-16
**Task:** Read-only audit of the admin filter pipeline. Map chip-click → URL → refetch end-to-end and diagnose Bug A (`/admin/products` chip changes don't take effect until hard reload) and Bug B (`/admin/products/[userId]` chip changes do nothing).

## Brief vs reality

The brief is essentially correct. One nuance worth flagging up front: the brief's claim "URL doesn't update on chip click" on Bug A is consistent with what I found, but the reason is not "the URL push fails," it's "the URL push is never attempted." Detailed under Bug A below.

## Pipeline map

End-to-end answers to the six questions in the brief, with file/line citations.

### 1. Chip click handler

Chips live in **`src/components/client/filters/SelectedFiltersDisplay.tsx`** (the X-on-each-chip row at the top of the product list) and in **`src/components/client/filters/DashboardFilters.tsx`** (the side filter panel — checkboxes, price inputs, order, product/moderation state selects).

The chip X-click handlers (`SelectedFiltersDisplay.tsx:47-241`) call setters straight on the filter store:

- `setSearchText(undefined)` — line 51
- `setOrder(undefined)` — line 62
- `clearProductStates()` — line 73
- `clearModerationStates()` — line 84
- `setPriceRange({ ...selectedPriceRange, free: !... })` — line 95
- `setPriceRange({ ...selectedPriceRange, from: '' })` — line 113
- `setPriceRange({ ...selectedPriceRange, to: '' })` — line 130
- `setRegionCityValues({...})` — lines 151, 173
- `addRemoveOptionFilter(selectedFilter.filter)` — line 236
- `addRemoveRangeFilter(selectedFilter.filter)` — line 234

There is no special "admin chip" handler. All chip clicks go to the store.

### 2. State update

The store is at **`src/lib/store/useFilterStore.ts`** — a `createFilterStore` factory (line 45) that returns three named exports (lines 189–191):

- `usePortalFilterStore` (read-only product/moderation states)
- `useDashboardFilterStore` (product states allowed, moderation states not)
- `useAdminFilterStore` (both allowed)

The admin product pages are routed to `useAdminFilterStore` via two wrappers that select by `portalScope`:

- `src/components/client/filters/SelectableFiltersWrapper.tsx:30-37` (side panel)
- `src/components/client/filters/SelectableSelectedFiltersDisplayWrapper.tsx:24-26` (chip row)
- `src/components/client/initializers/SelectableFilterManagerWrapper.tsx:23-25` (URL-sync component)

Both admin product routes mount these wrappers with `portalScope="admin"` (pages at lines 47, 49 of `app/[locale]/admin/products/page.tsx` and `app/[locale]/admin/products/[userId]/page.tsx`). The chip row, the side panel, and the URL-sync component all subscribe to the same `useAdminFilterStore`, so a chip click does correctly mutate the store the URL-sync component reads from.

### 3. URL sync

The URL push lives in **`src/components/client/initializers/FilterManager.tsx`** in the SYNC TO URL effect (lines 223–323). The push call itself is:

```ts
router.replace(newUrl, { scroll: false });   // line 311
```

`router` is from `next/navigation` (line 14), `newUrl` is composed from `useLocale()` + the next-intl locale-stripped `pathname` + the serialised params (line 306). The effect's gates are:

```ts
if (!hydratedRef.current || !hydrated) return;   // line 224
if (!isAllowedPath()) return;                    // line 225
```

The effect dep array (lines 313–323) is:

```
[searchText, selectedFilters, selectedOrder, selectedPriceRange,
 selectedRegionsAndCities, hydrated, pathname,
 selectedProductStates, selectedModeratioStates]
```

so chip-driven store mutations do retrigger the effect. The gating on `hydratedRef.current && hydrated` and `isAllowedPath()` is what determines whether the push fires.

`FilterManager` is mounted in three layouts:

- `app/[locale]/admin/layout.tsx:29` — `portalScope="admin"`
- `app/[locale]/owner/layout.tsx:30` — `portalScope="dashboard"`
- `app/[locale]/(portal)/(public)/layout.tsx` — `portalScope="portal"`

The admin instance persists across all `/admin/*` routes (single layout instance).

### 4. `FilterManager.isAllowedPath()` gate

`FilterManager.tsx:52-66`:

```ts
function isAllowedPath(): boolean {
  if (!pathname) return false;

  const exactAllowed = new Set(['/', '/owner/products', '/admin/products']);
  if (exactAllowed.has(pathname)) return true;

  if (pathname.startsWith('/catalog')) return true;

  return false;
}
```

Allowlist contents:

- Exact: `/`, `/owner/products`, `/admin/products`
- Prefix: `/catalog/...`
- Everything else: false. Including **`/admin/products/<userId>`** — there is no prefix rule for `/admin/products/`, only the exact `/admin/products`. Confirmed.

`pathname` comes from `usePathname` re-exported from `@/src/i18n/navigation` (line 3), which is the next-intl `createNavigation` variant — it returns the path **without** the locale prefix. So a user on `/rs-sr/admin/products` produces `pathname === '/admin/products'`, and a user on `/rs-sr/admin/products/42` produces `pathname === '/admin/products/42'`. Confirmed by reading `src/i18n/navigation.ts` and `src/i18n/routing.ts`.

`isAllowedPath()` is consulted in **two** places, both gates:

- HYDRATE FROM URL effect (line 73) — when false, skip reading params into the store. Result: store stays at defaults; chips render as if no filters are applied.
- SYNC TO URL effect (line 225) — when false, skip pushing store state to the URL. Result: chip clicks mutate the store and update the chip row visually, but the URL and the SSR-fetched product list stay frozen.

### 5. Refetch trigger

There is **no client-side refetch**. The product list re-renders only via SSR-driven remount.

The mechanism is in **`src/components/client/product/FilteredProductList.tsx:28`**:

```tsx
<ProductList
  key={JSON.stringify(filtersData)}
  ... />
```

`filtersData` is the SSR-computed `ProductsFilterDTO`, threaded down from the page server component (`SelectableFilterProductListWrapper` → `FilteredProductList`). When the page is re-rendered server-side with different `searchParams`, `filtersData` changes, `key` changes, `ProductList` unmounts and remounts with the new `initialProductsData` prop. There is no `useEffect` on the store inside `ProductList` that refetches — only paging-2+ does a client call via `onNextPage` (line 95).

So the full chain to refresh visible products on a chip click is:

1. Chip click → store setter.
2. SYNC TO URL effect → `router.replace(newUrl, { scroll: false })`.
3. Next.js App Router → re-execute the page server component with new `searchParams`.
4. Page re-runs `filterHydrationSSR` + `getAdminProducts` → new `initialProductsData` + new `filtersData`.
5. `FilteredProductList` receives new `filtersData` → key changes → `ProductList` remounts.

Step 3 is the brittle one. The `src/components/admin/FiltersPanel.tsx:65-69` pattern used elsewhere in admin (Reports, Reviews) pairs `router.replace` with a `setTimeout(() => router.refresh(), 0)` — strong evidence the team has hit cases where `router.replace` alone did not force a server-component re-render. The `FilterManager` URL push **does not** include this refresh, which is a known-fragile pattern even if it sometimes works on its own.

### 6. SSR vs CSR boundary

`/admin/products/page.tsx` and `/admin/products/[userId]/page.tsx` are async server components. The product fetch (`getAdminProducts`, `src/lib/service/nextCalls/productsSearchService.ts:133`) happens **only** on the server, marked `'use server'`. The list of products in the visible UI comes from `initialProductsData`, threaded through the props.

Client-side, `ProductList` holds local `products` state seeded from `initialProductsData.products` (line 81), and replaces it only on:

- Paging button click → `onNextPage` (line 95)
- The favorites-page path (`/favorites`) when `favoriteIds` changes (lines 110–127)
- `initialProductsData` reference change → resets state (lines 131–134)

There is no "client-side override on filter change." The client filter store never drives a client refetch on the admin product surfaces. The SSR path is the only path.

This is design-correct: filter state → URL → SSR → `key`-driven remount. It just requires every link in the chain to fire.

## Bug A diagnosis — `/admin/products`

**The bug fires whenever the user reaches `/admin/products` via in-app navigation from any other admin route.**

Walk-through, fresh tab loading directly on `/admin/products` (the case where it works):

1. AdminLayout mounts → SessionGuard authStateReady runs → children mount → `SelectableFilterManagerWrapper` mounts → `FilterManager(useAdminFilterStore)` mounts with `pathname === '/admin/products'`.
2. baseSite resolves (`useBaseSiteStore`) → HYDRATE FROM URL effect runs (deps `[baseSite]`, line 218). `isAllowedPath()` returns true (`/admin/products` is in `exactAllowed`). Hydration reads searchParams into the store and flips `hydratedRef.current = true` + `setHydrated(true)` (lines 216–217).
3. User clicks a chip → store updates → SYNC TO URL effect runs. Gates pass (`hydratedRef.current && hydrated && isAllowedPath()` all true). `router.replace(newUrl, { scroll: false })` fires.
4. Next.js navigates → page server component re-executes with new `searchParams` → new `filtersData` → `FilteredProductList` remounts → user sees filtered products. ✓

Walk-through, **in-app nav from any other admin route to `/admin/products`** (the broken case):

1. User lands on, say, `/admin/users` or `/admin/reports`. AdminLayout mounts → `FilterManager` mounts with `pathname === '/admin/users'`. `hydratedRef.current` is `false`.
2. baseSite resolves → HYDRATE effect runs. `isAllowedPath()` returns **false** (`/admin/users` is not in the allowlist). Effect early-returns at line 73. `hydratedRef.current` **stays false**, `hydrated` (store) **stays false**.
3. User clicks "Products" in `AdminSidebar` → Next.js client-side navigation to `/admin/products`. AdminLayout does **not** unmount (it's the layout for both routes). `FilterManager` does **not** unmount. `pathname` updates to `/admin/products`.
4. **Here is the bug:** the HYDRATE FROM URL effect has dep array `[baseSite]` (line 218). `baseSite` did not change. The effect does **not** re-run. `hydratedRef.current` is **still false**. `hydrated` (store) is **still false**.
5. User clicks a chip → store updates → SYNC TO URL effect runs → first gate `if (!hydratedRef.current || !hydrated) return` **early-returns at line 224**. **`router.replace` is never called.**
6. URL stays `/admin/products` (or whatever it was). Server component does not re-execute. List does not refetch. User sees no change.
7. Hard reload → SessionGuard re-runs → FilterManager re-mounts on `pathname === '/admin/products'` from the start of step 1 above → hydration runs, ref flips true → chip click then works. This matches the brief's "hard-reloading after a chip click does show the filtered view" observation, because by the time the user hard-reloads, the previously-clicked chip is still in the **store** (Zustand is in-memory but the in-flight click before the hard reload is gone too — actually no, hard reload clears the store, so the user would only see the filtered view if the URL had been updated). Hmm — the brief's "URL state ends up correct somehow" needs a more careful look. See **Caveat 1** at the bottom.

**Where the chain breaks:** step 5, SYNC effect early-return on `!hydratedRef.current`. The click handler fires, the store updates, but the URL push is gated off because the HYDRATE effect's deps array never re-ran the hydration when pathname changed from a disallowed admin path to `/admin/products`.

## Bug B diagnosis — `/admin/products/[userId]`

The brief noted Bug B looks like "isAllowedPath excludes the route" — that is **one** of two reasons. The second is the same hydration-deps issue as Bug A.

Walk-through (any entry path — direct URL or in-app nav from `/admin/users` row click — same outcome):

- **Direct URL load** of `/admin/products/42`:
  1. AdminLayout mounts → FilterManager mounts with `pathname === '/admin/products/42'`.
  2. baseSite resolves → HYDRATE effect runs. `isAllowedPath()` returns **false** (`/admin/products/42` is not in `exactAllowed`, and the only prefix rule is `/catalog`). Effect early-returns at line 73. `hydratedRef.current` stays false.
  3. Chip click → SYNC effect runs → first gate `!hydratedRef.current` → early-return. Even if it passed, second gate `!isAllowedPath()` would also early-return.
  4. URL unchanged. List unchanged. ✗

- **In-app nav** from `/admin/users` row click via `UsersTable.tsx:49`:
  1. Same as Bug A: FilterManager was mounted on a disallowed path; `hydratedRef.current` is false.
  2. pathname updates to `/admin/products/42`; HYDRATE deps `[baseSite]` did not change; hydration never runs.
  3. Chip click → SYNC effect → `!hydratedRef.current` (true) **OR** `!isAllowedPath()` (true) → either gate early-returns. URL unchanged. ✗

**Adding `/admin/products/[userId]` to the allowlist alone would not fix Bug B.** The hydration gate on `!hydratedRef.current` is the dominant short-circuit. The route is "doubly broken" — the allowlist gap and the deps-array gap each independently prevent the URL push.

It is worth noting that even if **both** issues were fixed, the per-user admin product list has an additional concern: `filtersData.ownerId` is set server-side from the URL path (`page.tsx:27`) and is **not** something the chip row could ever express. Filter chips and `ownerId` are orthogonal — chip changes should re-filter within that user's products, not cross to other users. The current SSR path handles this correctly (the page reads `userId` from `params` independently of `searchParams`), so this is not a blocker, just a noted invariant for whoever writes the fix.

## Shared root cause or not

**Shared root cause: yes**, with one Bug-B-only secondary.

The dominant root cause for **both** bugs is the HYDRATE FROM URL effect's dep array `[baseSite]` at `FilterManager.tsx:218`. It treats hydration as a one-shot at mount-time-once-baseSite-loads, and ignores pathname changes thereafter. Whenever the user enters an allowed path via in-app navigation from a non-allowed path (which is the common case for both `/admin/products` and `/admin/products/[userId]`), hydration is skipped, `hydratedRef.current` stays false, and the SYNC effect early-returns on every subsequent chip click.

The Bug-B-only secondary is that `isAllowedPath()` does not include `/admin/products/<userId>` at all. Even with the hydration deps fixed, the per-user list would still be excluded. But this is the smaller of the two issues; the deps-array fix is the bigger lever.

## Recommended fix shape

One small fix touching `FilterManager.tsx`, addressing both bugs. Bug-chat scoped; not feature-sized.

### Smallest fix (single PR)

1. **Make hydration pathname-aware.** Replace the single `hydratedRef = useRef(false)` boolean with a "last-hydrated path" ref:

   ```ts
   const lastHydratedPathRef = useRef<string | null>(null);

   useEffect(() => {
     if (!baseSite) return;
     if (!isAllowedPath()) return;
     if (lastHydratedPathRef.current === pathname) return;

     // ... existing hydration body ...

     lastHydratedPathRef.current = pathname;
     setHydrated(true);
   }, [baseSite, pathname]);
   ```

   And update the SYNC effect gate to use `lastHydratedPathRef.current === pathname` instead of the old `hydratedRef.current` boolean — both checks mean "we've correctly read the URL for the path we're on." Keep the `&& hydrated` half of that gate; it's still useful for the very first paint after mount.

   *This single change fixes Bug A entirely.* On an in-app nav from `/admin/users` → `/admin/products`, pathname changes from `/admin/users` to `/admin/products`, the effect re-runs, isAllowedPath is now true, hydration runs, the ref updates, and chip clicks thereafter push the URL.

2. **Extend `isAllowedPath()`.** Change line 55 from:
   ```ts
   const exactAllowed = new Set(['/', '/owner/products', '/admin/products']);
   if (exactAllowed.has(pathname)) return true;
   ```
   to additionally accept `/admin/products/<userId>`:
   ```ts
   if (pathname.startsWith('/admin/products')) return true;
   ```
   This subsumes the exact `/admin/products` rule for that path. Keep the `/`, `/owner/products` exact entries and the `/catalog` prefix as they are.

   *This fixes the Bug B allowlist gap.* Combined with change (1), both bugs are addressed.

3. **(Recommended, defensive)** After `router.replace(newUrl, { scroll: false })` at line 311, add the same defensive pair used by `src/components/admin/FiltersPanel.tsx:67-69`:

   ```ts
   setTimeout(() => router.refresh(), 0);
   ```

   The team has consistently paired these two calls in `FiltersPanel.tsx` and similar admin tables (`AdminReportOverviewDialog`, `AdminReviewOverviewDialog`, `ReviewsTable`, `ReportsTable`). The strong implication is they've observed cases where `router.replace` alone did not force the server component to re-execute. Whether this affects the `FilterManager` path or not depends on Next.js App Router caching specifics that are easier to verify in a browser than in a code read. Adding this pair is cheap and matches the established admin pattern — see **Caveat 2** for the unknown.

### Order

Single PR. All three changes are local to `FilterManager.tsx`, around 8–12 changed lines total. The fix is shared, no need to split.

### Bigger than bug-chat?

No. This is bug-chat scoped: a one-file, three-change fix totalling under ~15 lines.

A bigger refactor would be worth considering separately (not in this fix): the `isAllowedPath()` allowlist is an O(n)-paths-known-by-name pattern that has already failed twice (Bug B was already documented in `issues.md` 2026-05-14 "filter chips change without refetch or URL sync"). Each new filter-enabled route requires a code edit here. A different model — e.g., FilterManager mounted per-page rather than per-layout, or `isAllowedPath` driven by an explicit per-route opt-in prop — would remove that footgun. But that's a refactor brief, not a bug fix.

## Caveats and unknowns

1. **Brief's "URL doesn't update on chip click" claim, post-fix.** The code analysis shows `router.replace` is never called pre-fix, which is consistent with "URL doesn't update." After fix (1) applied, the SYNC effect's `router.replace` should fire. Whether the URL in the browser bar visibly updates is then a Next.js / next-intl interplay matter. Confirmed in a browser by Igor would be the proof. The expected outcome after fix (1)+(2): the URL bar changes on chip click; the list refetches (with or without fix (3) depending on whether Next's automatic re-render kicks in).

2. **Whether `router.refresh()` is strictly required (fix 3).** This is the unknown. `router.replace` to the same path with new searchParams *should* re-execute the page server component in Next 15. The fact that `src/components/admin/FiltersPanel.tsx:67-69` defensively schedules a `router.refresh()` is strong but not conclusive evidence that pure `router.replace` was insufficient on those surfaces. **Recommend:** apply fix (1)+(2) first, test in a browser, and add fix (3) only if the list visibly doesn't refetch despite the URL updating correctly.

3. **`/owner/products` may have a latent version of the same Bug A.** The Owner layout (`app/[locale]/owner/layout.tsx`) also mounts FilterManager and has multiple sidebar links to non-allowed paths (`/owner/balance`, `/owner/reviews`, `/owner/follows`, `/owner/analytics`, etc. — `src/lib/data/sectionNavigation.ts:19-85`). By the same mechanism as Bug A, in-app navigation from any of those to `/owner/products` would skip hydration and leave the SYNC effect gated off. This is not in the brief's scope, but the proposed fix (1) would address it as a side-effect — worth noting so Igor isn't surprised.

4. **The home page `/` and `/catalog/...` routes.** These almost certainly do work, because a fresh tab load on them sets `pathname` correctly at FilterManager's mount-time, so the initial-mount hydration succeeds. The deps-array bug only matters for *in-app navigation between two different allowed paths* or *in-app navigation from a disallowed path to an allowed one*. Within the public portal those situations are: home → catalog (in-app nav from `/` to `/catalog/X`). I did not exhaustively walk every such transition; the fix proposal handles them by construction (pathname dep).

5. **`SessionGuard.tsx:42`'s `useAuthStore.getState().isAdmin` read.** Unrelated to this audit, but adjacent. It reads the store imperatively after an `await`. Works, but reads slightly differently from the rest of `SessionGuard`. Not in scope.

6. **`hydrated` (store boolean) is redundant under the proposed fix.** Once we use `lastHydratedPathRef.current === pathname` as the gate, the `hydrated` store boolean is informational only. The current dual-gate (`hydratedRef.current && hydrated`) reads belt-and-suspenders. Whoever implements fix (1) can decide whether to keep `hydrated` (for tests / external consumers) or remove it. Not load-bearing either way.

## Cleanup performed

- none needed (read-only audit, no edits)

## Obsoleted by this session

- nothing — read-only audit. No code paths displaced.
- The `issues.md` 2026-05-14 entry "/admin/products/[userId] filter chips change without refetch or URL sync" is corroborated by this audit; if a fix lands per the recommendation here, that entry would become `fixed`. Same for the entry "Bad catalog slug produces two different failure modes depending on position" — orthogonal but lives in the same `FilterManager` neighborhood.

## Known gaps / TODOs

- The audit relied entirely on code reading. Three statements above would benefit from a one-minute browser test (Igor) before fix is written:
  1. Confirm that direct `/admin/products` load DOES allow chip clicks to update the URL + refetch (predicted: yes).
  2. Confirm that in-app nav from `/admin/users` (sidebar) to `/admin/products` produces broken chip clicks (predicted: yes — this is Bug A's mechanism).
  3. Confirm that `router.refresh()` is or isn't needed alongside the `router.replace` (predicted: probably not strictly required, given the route's `searchParams` Promise should be enough to invalidate, but the FiltersPanel pattern suggests defensive use).

## Conventions check

- Part 4 (cleanliness): N/A — read-only audit, no edits.
- Part 4a (simplicity) / Part 4b (adjacent observations): Part 4b applies. Three adjacent observations are noted in **Caveats and unknowns** items 3 (latent owner-products bug), 5 (SessionGuard imperative read), 6 (redundant `hydrated` store boolean). All low severity, not in scope.
- Part 6 (translations): N/A — no translation keys touched.
- Other parts touched: none.

## For Mastermind

The audit verdict in one line: **both bugs share a root cause** — `FilterManager`'s HYDRATE FROM URL effect's dep array is `[baseSite]` and ignores `pathname` changes — with `isAllowedPath()` being a Bug-B-only secondary. One small fix (under ~15 lines, single file) addresses both. Not feature-sized.

Three things worth Mastermind's attention beyond the verdict:

1. **The owner-products surface may carry the same latent bug** (Caveat 3). If the fix lands as proposed, this gets fixed as a side effect — but worth a quick Igor browser-check to confirm whether it's actually broken today, because if it is and we don't say so in the bug-fix brief, the owner-products fix will look like scope creep rather than the proposed fix's natural consequence.

2. **The `isAllowedPath()` allowlist pattern has now failed twice** (the `[userId]` exclusion, and the bug-A mechanism that depends on the path being in the list at the right time). It's an opt-in-by-string-matching list embedded inside FilterManager, growing one path at a time. This is a known anti-pattern: a refactor that flips it (e.g., a per-route `enableFilterSync` prop, or co-locating FilterManager with the pages that use it instead of the layouts that don't all want it) would remove the footgun. Not scoping the refactor here, but flagging so Mastermind can decide whether to queue a follow-up.

3. **The `router.replace + setTimeout(router.refresh, 0)` pattern in admin** (`src/components/admin/FiltersPanel.tsx:65-69`, plus four other call-sites) suggests the team has observed cases where `router.replace` alone didn't force a server-side re-render. The bug-fix brief should probably mandate that pattern for `FilterManager` too — costs near-zero, matches the established admin convention, removes ambiguity. Up to Mastermind whether to fold (3) into fix (1)+(2) as a single PR or treat as a defensive variant.
