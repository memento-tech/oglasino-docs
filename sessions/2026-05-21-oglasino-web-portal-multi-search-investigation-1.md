# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-21
**Task:** Read-only investigation: 4× product search on hard refresh, 3× on normal refresh, with portal "jerk"

## Implemented

- Read-only audit. No code changes, no staged edits, no test runs.
- Mapped every code path on the portal home route (`app/[locale]/(portal)/(public)/page.tsx`) that reaches `POST /api/public/product/search`, every site that calls `syncUserToBackend` / posts `/api/auth/firebase-sync`, and every site that posts `/api/secure/push/token`.
- Mapped the data-flow into `ProductList` so the "jerk" can be attributed to a specific differ between SSR responses.
- Output is this summary; nothing else changed on disk.

## Q1 — Every code path that triggers `POST /api/public/product/search` on the portal home

Two SSR call sites fire from the route. **No client component mounted on the home page issues an unconditional product-search POST on mount** — every client path is either gated to a non-home pathname (`pathname === '/favorites'`) or to a user action (pagination button click).

### Call site A — page-render fetch (always fires per SSR render)

File: `app/[locale]/(portal)/(public)/page.tsx:34-37`.

```ts
const initialProductsData: ProductOverviewsDTO = await getPortalProducts(
  filtersData,
  getInitialPage()
);
```

- Phase: SSR (server component, `export const revalidate = 0;` at line 22, so the page is fully dynamic and the route's data cache is bypassed).
- `getPortalProducts` is from the server flavor `src/lib/service/nextCalls/productsSearchService.ts:111-116`, which delegates to `getProducts` at `:75-109`:

```ts
const res = await FETCH_BACKEND_API.post<ProductOverviewsDTO>(
  uri,
  { productsFilter, paging },
  { headers: { 'X-Base-Site': tenant, 'X-Lang': lang } }
);
```

- No `skipAuth: true` and no `next: { revalidate, tags }`. Per `src/lib/config/fetchApi.ts:32-45`, that means the call **reads cookies, attaches the `firebase_token` Bearer header, and is NOT served from the Next.js Data Cache**. Every SSR render is one fresh backend POST.
- Trigger: SSR render (initial GET, RSC re-render via `router.refresh()`, or `revalidatePath('/')`).
- Payload: `productsFilter = filtersData` from `filterHydrationSSR(...)` at line 33. For the no-filter home, `parseFiltersFromQueryParams` in `src/lib/utils/filtersHelper.ts:45-51, 68-70` synthesises:

```ts
const randomAddOn = {
  applyRandom: true,
  randomSeed: Date.now(),
};
if (params.length === 0 && applyRandom) {
  return { ...randomAddOn };
}
```

So `productsFilter = { applyRandom: true, randomSeed: <Date.now() at SSR time> }`. `paging = { page: 0, perPage: 20 }` (from `getInitialPage()` + `PRODUCTS_PER_PAGE` at `src/lib/utils/utils.ts:8, 15-20`).
- Receiver: `initialProductsData` prop into `SelectableFilterProductListWrapper` (page.tsx:50-55), then `FilteredProductList` (`src/components/client/product/FilteredProductList.tsx:30-39`), then seed for `useState(initialProductsData.products)` in `ProductList` (`src/components/client/product/ProductList.tsx:68`). Not stored in Zustand, not stored in the Next.js Data Cache.

### Call site B — `generateMetadata` snapshot fetch (cache-gated)

File: `app/[locale]/(portal)/(public)/page.tsx:16-20` → `src/metadata/generateHomePageMetadata.ts:23` → `src/lib/service/nextCalls/metadataSnapshotService.ts:55-66`.

```ts
const res = await FETCH_BACKEND_API.post<ProductOverviewsDTO>(
  '/public/product/search',
  { productsFilter, paging: { page: 0, perPage: limit } },
  {
    headers: { 'X-Base-Site': tenant, 'X-Lang': lang },
    skipAuth: true,
    next: {
      revalidate: 3600,
      tags: buildSnapshotTags(filters),
    },
  }
);
```

- Phase: SSR (`generateMetadata`).
- Trigger: every SSR render of the home route — but **the call hits the Next.js Data Cache** (1-hour `revalidate`, tag `metadata-snapshot-home` per `metadataSnapshotService.ts:31`), so a cache hit completes locally without a backend POST. The first uncached call after a 1-hour-or-longer gap, or after a tag-bust on `metadata-snapshot-home`, fires a backend POST.
- Payload: forced `applyRandom: false`, `orderBy: NEWEST_FIRST_ORDER (createdAtDesc)`, `paging.perPage = 10`. Anonymous (`skipAuth: true`).
- Receiver: shaped into `MetadataProductSnapshot[]` used only for OG image + `application/ld+json`. Does not feed `ProductList`.

### Call site C — client pagination fetch (only on page > 0)

File: `src/components/client/product/SelectableFilterProductListWrapper.tsx:32-38`.

```ts
const onNextPortalPageInternal = (paging: PagingDTO): Promise<ProductOverviewsDTO> => {
  if (paging.page === 0) {
    return Promise.resolve(initialProductsData);
  }
  return getPortalProducts({ ...filtersData }, paging);
};
```

- Resolves from the client flavor `src/lib/service/reactCalls/productsSearchService.ts:156-161` → `getProducts` at `:132-154` → `BACKEND_API.post('/public/product/search', ...)`.
- Trigger: `ProductList.onDisplayPageChange` (`src/components/client/product/ProductList.tsx:78-91`) — wired to the pagination button at `:140-149`. Page 0 short-circuits to the in-memory `initialProductsData`, so this site cannot reach the backend on a fresh load with no user action.
- Auth: axios pipeline at `src/lib/config/api.ts:94-139` attaches the cached Bearer token if a Firebase user is present.

### Call site D — favorites-route refetch (NOT on home)

File: `src/components/client/product/ProductList.tsx:97-114`.

```ts
useEffect(() => {
  const loadFavorites = async () => {
    ...
    const result = await onNextPage({ page: 0, perPage: PRODUCTS_PER_PAGE });
    setProducts(result.products);
    setLoading(false);
  };

  if (pathname === '/favorites') {
    loadFavorites();
  }
}, [favoriteIds]);
```

- This effect re-runs when `favoriteIds` updates (so it WILL fire post-`InitFavoritesStore.init` on a signed-in user). The `if (pathname === '/favorites')` gate excludes the home route; the effect returns early on `/`. Captured here because it's an `[favoriteIds]`-keyed re-fetcher in the same `ProductList` instance — but on the portal home it is a no-op.

### Other potential search call sites — ruled out for the home route

- `src/components/client/SearchInput.tsx:78-91` posts to `/public/product/search/autocomplete`, not `/public/product/search`.
- `src/lib/service/reactCalls/extraProductsService.ts:8-27` (`fetchExtraProducts`) posts to `/public/product/search` but its only mount sites are `/favorites` (`app/[locale]/(portal)/(protected)/favorites/page.tsx`) and `/user/[userId]` (`app/[locale]/(portal)/(public)/user/[userId]/page.tsx`). Not mounted on `/`.
- `app/sitemap.ts:61` posts to `/public/product/search` server-side, but only when a crawler hits `/sitemap.xml`.
- No store action (`useFilterStore`, `useFavoritesStore`, `useAuthStore`, `useBaseSiteStore`, `useCardSizeStore`, `useConsentStore`, `useChatStore`, `useChatBlockStore`) issues a product-search POST.

### Reconciliation against the 4-call / 3-call evidence

Code-reading gives us **two distinct SSR call sites**: A (always one POST per render) and B (one POST per cache miss).

- 4 calls on hard refresh = 1× A (initial SSR) + 1× B (initial SSR, metadata cache cold) + 2× A (two later RSC re-renders; metadata cache now warm so B does not fire on the re-renders).
- 3 calls on normal refresh = 1× A (initial SSR; metadata cache already warm so B does not fire) + 2× A (two later RSC re-renders).

**This is the most consistent fit with the access-log evidence, but the code-reading did not produce a deterministic identification of what triggers the two later RSC re-renders.** See "Known gaps" below — this is a real gap in the audit, not papered over.

What I found and ruled in / out for the RSC-re-render trigger:

- `setTimeout(() => router.refresh(), 0)` at `src/components/client/initializers/FilterManager.tsx:317`, inside the SYNC TO URL effect (`:228-329`). The effect runs after HYDRATE FROM URL flips `hydrated = true` (`:222`). Body builds `newUrl = `/${locale}${pathname}` + …` (`:311`) and compares to `current = window.location.pathname + window.location.search` (`:313`). When the URL is canonical, `newUrl === current` and no refresh fires; when it isn't (e.g. visiting `/rs-sr` without trailing slash, or any URL with a non-canonical query-param order), `router.replace` + `router.refresh` fire **once**. This is a plausible single-RSC-re-render source on the home but does not explain two re-runs.
- `router.refresh()` in `AccountStateDialogsInit.tsx:93, :123` is gated by `accountJustDeleted` / `restored` flags — not set on a normal home refresh.
- `router.refresh()` everywhere else (`FavoriteButton.tsx:64`, dialog `onClose` handlers, admin tables, `AuthUserProfileButton` menu items) is user-action gated.
- No effect deps on `useAuthStore.user` issue `router.refresh()` (audited every `useAuthStore` consumer; only `loadIsAdmin` and `init` favorites fire, and those are not product searches).

I could not pinpoint a second auto-firing `router.refresh()` site in code. The candidate hypotheses I could not refute from code alone:

1. The single `FilterManager` refresh path actually fires twice in a row because the SYNC TO URL effect's deps churn twice — `selectedFilters`, `selectedPriceRange`, `selectedRegionsAndCities` are reassigned to fresh empty-state references inside HYDRATE FROM URL (`:179, :189, :207`), so the effect re-runs at least once even when all values are content-equal to initial state. A run that produces a non-canonical-form `newUrl` would refresh once; a subsequent run could in principle fire a second refresh if a downstream state update (e.g. `setHydrated`, `setOrder(undefined)`) lands in a separate React render and re-evaluates `newUrl !== current` differently.
2. The two extra RSC re-renders are not from `router.refresh()` at all but from something other than the user-facing app code path — e.g. Next.js's automatic `<Link>` RSC prefetching for `<HomeLink>` (`Header.tsx:19-22`) firing a route-data request when the header hydrates, or a `link prefetch={true}` in the `CategoryNavigation`. Backend logs would record those as identical search POSTs (RSC payload fetches go through the same RSC tree as `router.refresh`).

Either way, the upper bound on what code can do without explicit instrumentation is "two RSC re-renders happen post-mount." Determining the actual trigger requires either (a) a `console.log` in the page component to count SSR renders during one hard refresh, or (b) network-tab + React DevTools tracing on the live page.

## Q2 — Why `firebase-sync` fires twice on hard refresh

Single explicit caller in code:

- `src/components/client/initializers/UseTokenRefresh.tsx:38` — inside the `onIdTokenChanged` listener.

```ts
return onIdTokenChanged(auth, async (firebaseUser) => {
  ...
  const backendUser = await syncUserToBackend(firebaseUser);
  useAuthStore.getState().setUser(backendUser);
  if (backendUser) {
    await initPushForAuthenticatedUser(backendUser.id);
  }
});
```

`UseTokenRefresh` is mounted exactly once — in `AppInit.tsx:28`, mounted at `[locale]/layout.tsx:43`. There is no second component that imports or calls `syncUserToBackend` on mount.

Other `syncUserToBackend` reference: `src/lib/store/useAuthStore.ts:281` inside `refreshUser()`. Only called by `UserBasicDataSelectorDialog.tsx:85` on a user click. Not auto-fired on the home route.

The five other Firebase auth listeners in the codebase — `src/lib/config/api.ts:87-92`, `src/lib/service/reactCalls/authTokenCookie.ts:21-25`, `src/components/server/layout/MobileFooterNavigation.tsx:22`, `src/components/client/HeaderNavButtons.tsx:32`, `src/lib/hooks/useAuthResolved.ts:19` — do **not** call `syncUserToBackend`. They each maintain local state (cached-token clearance, lastWrittenToken cache invalidation, mobile/header button auth-aware UI, gate-resolution hook).

So there is one code-level caller. Yet the backend logs show two real syncs (both run `banned_user_audit` + `users` lookup). On normal refresh only one fires.

The mechanism this audit can confirm in code: **the firebase-sync POST has no single-flight guard.** `syncUserToBackend` at `src/lib/service/reactCalls/authService.ts:132-163` runs straight through every invocation of the listener callback. The display-name one-shot cell (`nextRegisterDisplayName` at `:27`) is consumed-and-nulled on the first run, but the network call still goes out on every subsequent call.

The most consistent reading from code alone: **Firebase's SDK emits `onIdTokenChanged` twice during initial session restoration when the persisted token is near or past expiry** — once with the persisted token, then again ~milliseconds later after the SDK transparently refreshes it. The 8 ms separation on hard refresh and the absence of a duplicate on normal refresh (warmer in-memory token state, single emission) match that behaviour. The single listener emits two backend syncs because nothing dedupes back-to-back same-uid invocations within `UseTokenRefresh`.

This is *consistent with* the documented "sole hydrator" contract in `UseTokenRefresh.tsx:13-17` and the closing entry in `decisions.md` 2026-05-21. The contract is not broken — there is exactly one listener that calls `syncUserToBackend`. The duplicate is the SDK emitting twice through that one listener.

I cannot rule out from code alone that some interaction with `auth.signOut()` from the response-interceptor 403 path (`src/lib/config/api.ts:51-66`) or the deletion-handshake `getIdToken(true)` re-emits a fresh event — but neither of those is on the cold-refresh path for a non-banned user.

## Q3 — Why `push/token` POSTs twice on hard refresh

Single call chain:

- `src/notifications/service/pushTokenService.ts:3-9` — `attachPushTokenToBackend` POSTs `/secure/push/token`.
- `src/notifications/lib/devicePush.ts:29-50` — `initPushForAuthenticatedUser` is the only caller of `attachPushTokenToBackend`.
- `src/components/client/initializers/UseTokenRefresh.tsx:41` — the only caller of `initPushForAuthenticatedUser`, executed inside the same `onIdTokenChanged` callback after `syncUserToBackend` returns.

```ts
const backendUser = await syncUserToBackend(firebaseUser);
useAuthStore.getState().setUser(backendUser);
if (backendUser) {
  await initPushForAuthenticatedUser(backendUser.id);
}
```

Same root cause as Q2: one listener, two emissions → two `attachPushTokenToBackend` POSTs with the same token. Same shape, same fix surface — single-flight at the listener level (or at the syncUserToBackend / initPush level) would collapse both duplicates at once.

`initPushForAuthenticatedUser` does not check whether the same token has already been registered with the backend in the current session. Both calls go through with the same payload (`token`, `userId`, `platform: 'FIREBASE'`).

## Q4 — Why hard refresh = 4 search calls and normal refresh = 3 search calls

The one call missing on normal refresh is **the `generateMetadata` snapshot call** (Call site B in Q1):

- `src/lib/service/nextCalls/metadataSnapshotService.ts:60-64` declares `next: { revalidate: 3600, tags: buildSnapshotTags(filters) }`. For the home, `buildSnapshotTags({})` returns `['metadata-snapshot-home']` (`:31`).
- The Next.js Data Cache keys on URL + body + headers. The metadata call's body is fixed (no filters, orderBy newest, perPage 10) so successive home-page requests collide on the same cache key.
- Hard refresh (cold cache) → cache miss → backend POST. Normal refresh inside the 1-hour window → cache hit → no backend POST.

The page-render call (Call site A) **cannot** be the missing one — it carries `export const revalidate = 0` (`app/[locale]/(portal)/(public)/page.tsx:22`) and no `next.tags`, so it never enters the Data Cache and fires on every SSR render regardless of refresh kind.

Auth-state warmth in `useAuthStore` is not the differentiator either: on both hard and normal refresh, the JS runtime restarts from scratch, the in-memory Zustand state is gone, the `onIdTokenChanged` listener subscribes fresh, and the SDK restores the persisted user from IndexedDB. The Q2 SDK-double-emit reasoning applies equally to both — but the brief notes only one firebase-sync on normal refresh, suggesting the SDK's near-expiry refresh path did not trigger on the warmer second case. That's consistent with a token refresh having just happened on the prior page life (so the persisted token still has > 5 min of life left, no refresh needed) versus the cold case where the persisted token had drifted past the SDK's preempt-refresh threshold.

The "warm-path optimisation" worth preserving is the 1-hour `revalidate` on `metadata-snapshot-home`. Any fix that addresses the post-mount refetch cascade should not also force the metadata snapshot to fire on every visit.

## Q5 — The "jerk": what changes between renders

The specific differ is **the `randomSeed: Date.now()` regenerated on every SSR render** at `src/lib/utils/filtersHelper.ts:45-48`:

```ts
const randomAddOn = {
  applyRandom: true,
  randomSeed: Date.now(),
};
if (params.length === 0 && applyRandom) {
  return { ...randomAddOn };
}
```

This is the seed passed to the backend in `productsFilter` on the home page (no query params, `applyRandom = true`). Each SSR render produces a different `Date.now()` → the backend orders the result set differently → a different product slate (different first-20 IDs or same IDs in a different order) lands at the client.

The remount mechanism that turns the new payload into a *visible* swap (not just a silent state replace):

- `src/components/client/product/FilteredProductList.tsx:31` keys the inner `ProductList` on the entire `filtersData`:

```tsx
<ProductList
  key={JSON.stringify(filtersData)}
  initialProductsData={initialProductsData}
  ...
/>
```

- A new `randomSeed` in `filtersData` flips the key → React unmounts the previous `ProductList` and mounts a fresh one with the new `initialProductsData`. The freshly-mounted `ProductList.tsx:68` seeds `useState(initialProductsData.products)` with the new array; the previous state is discarded. The "sync initial data" effect at `:118-121` would handle the same in-place if the key didn't change, but the key change forces the harder remount.
- Cards render in the new order. Visible swap.

There is a comment at `filtersHelper.ts:63-67` that acknowledges this mechanism for the case where `searchText`/`orderBy` are present and the seed is therefore wire-noise; it does not address the no-criteria home case, where the seed is intentionally emitted and is intentionally per-request. Each SSR re-run is by design a different product slate.

Secondary differ — **`favorite` flag on `ProductOverviewDTO`** (`src/lib/types/product/ProductOverviewDTO.ts:18`). The backend computes this from the authenticated user's favourites. SSR call #1 with no `firebase_token` cookie → all `favorite: false`. SSR re-run after the `firebase_token` cookie has been written by `writeFirebaseTokenCookie` (during firebase-sync) → backend has the auth identity → some `favorite: true`. Heart-icon swaps on those cards. Not "products replaced," but visually overlaps with the seed-driven main jerk.

Tertiary differ — **client-side favourites overlay** in `src/components/client/buttons/FavoriteButton.tsx:37-38`:

```ts
const favoriteIds = useFavoritesStore((s) => s.favoriteIds);
const isFavorite = favoriteIds.includes(productData.id);
```

When `InitFavoritesStore.init()` (`src/components/client/initializers/InitFavoritesStore.tsx:10-14`) populates `favoriteIds` from `/secure/favorites`, every mounted `FavoriteButton` re-renders. This independently flips heart icons on top of the SSR-rendered baseline. Same direction as the secondary differ; happens on a slightly different beat.

Neither `iamActive` nor `isFollowingCurrent` appears on `ProductOverviewDTO` — those live on `UserInfoDTO` per the `issues.md` 2026-05-17 entry. They don't contribute to the product-list jerk on the home route.

**The dominant differ is `randomSeed: Date.now()` × `key={JSON.stringify(filtersData)}`.** That is the precise mechanism that swaps products. The favourites surface is an additional, smaller-scale visual churn that lands on top.

## Reconciliation — mapping each access-log call

Hard refresh, 4 calls:

| # | Time | Mechanism                                                                                              | File                                                                  |
| - | ---- | ------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- |
| 1 | T+0      | Initial SSR — page render `getPortalProducts(filtersData, getInitialPage())`                       | `app/[locale]/(portal)/(public)/page.tsx:34`                          |
| 2 | T+170ms  | Initial SSR — `generateMetadata` → `getMetadataProductsSnapshot({}, 10)` (data-cache cold)         | `src/lib/service/nextCalls/metadataSnapshotService.ts:55`             |
| 3 | T+2600ms | Post-firebase-sync RSC re-render — page render again (metadata cached, only Call site A re-fires) | `app/[locale]/(portal)/(public)/page.tsx:34`                          |
| 4 | T+3650ms | Second post-mount RSC re-render — page render again                                                | `app/[locale]/(portal)/(public)/page.tsx:34`                          |

Normal refresh, 3 calls:

| # | Time | Mechanism                                                                  | File                                                                  |
| - | ---- | -------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| 1 | T+0      | Initial SSR — page render `getPortalProducts(...)`. Metadata snapshot is a cache HIT, no POST. | `app/[locale]/(portal)/(public)/page.tsx:34`                          |
| 2 | partway  | Post-firebase-sync RSC re-render — page render                          | `app/[locale]/(portal)/(public)/page.tsx:34`                          |
| 3 | partway  | Second post-mount RSC re-render — page render                           | `app/[locale]/(portal)/(public)/page.tsx:34`                          |

Honest gap: I can map calls #1 and #2 of hard refresh directly from code (two SSR call sites firing in parallel; cold metadata cache). I can map calls #3, #4 and the corresponding normal-refresh #2, #3 only by analogy — they are RSC re-renders of Call site A, but the trigger that produces *two* of them post-mount is not deterministically identifiable from code alone (see Q1 closing).

I checked specifically: there is no third or fourth SSR call site for `/public/product/search` on the portal home. The 4 / 3 totals reconcile cleanly to 2 SSR-call-sites × N renders.

## Root cause(s)

Three root causes, independent:

1. **`randomSeed: Date.now()` regenerated per SSR render + key-based remount in `FilteredProductList`** — `src/lib/utils/filtersHelper.ts:45-48` and `src/components/client/product/FilteredProductList.tsx:31`. This is the cause of the "jerk." Every SSR re-render of the home produces a freshly-randomised product slate; the inner `ProductList` is keyed on the stringified filter, so the new slate is presented via remount.

2. **No de-duplication of `onIdTokenChanged` callback invocations for the same user** — `src/components/client/initializers/UseTokenRefresh.tsx:18-43`. The SDK can emit twice on initial restoration (persisted-then-refreshed token); the listener fires `syncUserToBackend` and `initPushForAuthenticatedUser` independently for each emission. The "sole listener" contract is not violated — but the listener has no idempotency guard. This causes the duplicate firebase-sync POST AND the duplicate push/token POST.

3. **Two post-mount RSC re-renders of the home route per visit** (mechanism not deterministically identified from code). Each re-render fires Call site A again (Call site B is data-cached). The user-visible cost is two extra backend product searches plus the two seed-driven jerks.

Root causes 1 and 3 compound: root cause 3 produces the extra renders; root cause 1 turns each into a visible swap. Fixing either eliminates the user-visible jerk. Fixing root cause 3 also collapses the call count to 1 (normal refresh) / 2 (hard refresh, until metadata cache decays). Root cause 2 is independent of both and lives on its own fix axis (firebase-sync + push/token duplicate POSTs).

## Fix shape sketch

For each root cause:

1. **Random-seed-driven jerk.** Two routes worth considering. (a) Move `randomSeed` out of `filtersHelper.parseFiltersFromQueryParams` and into `getPortalProducts`'s call site so the seed is stable across the lifecycle of a single user visit (e.g. derive from a short-lived cookie or a per-route ETag). (b) Remove `randomSeed` from `filtersData` before it reaches `JSON.stringify(filtersData)` in the key — keep the seed for the backend but exclude it from the key calculation so `ProductList` does not remount on seed-only changes. Either fix takes the visible churn off the home page; (a) also stabilises which products land in any subsequent paginate-from-zero. (b) is lower-blast-radius (one prop key, two lines).

2. **Listener-side duplicate firebase-sync / push-token.** Single-flight in `UseTokenRefresh` — track `lastSyncedUid` + `inFlight` on the closure or via a `useRef`. Same uid + still-in-flight → drop. Same uid + just-finished within N seconds → drop. Drop only the listener-level POST; do not gate `refreshUser` or sign-out paths.

3. **Two post-mount RSC re-renders.** Cannot be sketched precisely until the trigger is found. Three plausible shapes: (a) if FilterManager's SYNC TO URL is the source, add a deeper equality check before `router.replace(...) + setTimeout(refresh)` so canonical-but-different-reference URLs don't cause a refresh, and remove `setTimeout(0)` (it exists because `router.replace` doesn't refetch RSC server data — but `router.replace` followed by `router.refresh` is reachable directly without the setTimeout in current Next 16). (b) if it's a prefetch / route-data fetch, audit `<Link prefetch={...}>` on `Header.tsx` / `CategoryNavigation` and the navigation-progress wiring. (c) if it's an effect that depends on `user` or `favoriteIds` and refreshes the route to repaint auth-aware UI (none found in this audit, but might exist in code I didn't reach), remove the refresh and let the server data flow through SSR-once instead.

## Files touched

- (none — read-only audit)

## Tests

- (none — read-only audit per the brief)

## Cleanup performed

- none needed

## Config-file impact

- `conventions.md`: no change
- `decisions.md`: no change
- `state.md`: no change
- `issues.md`: no change (the brief explicitly directs surfaced bugs to "For Mastermind" with severity guess + out-of-scope line — see below)

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): confirmed (no code changes; no commented-out blocks added)
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): confirmed — adjacent findings flagged in "For Mastermind"
- Part 6 (translations): N/A this session
- Other parts touched: Part 3 (engineer agent hard rules — read-only, no commits, no cross-repo edits, no writes to the four config files)

## Known gaps / TODOs

- **Cannot deterministically identify from code alone what triggers the two post-mount RSC re-renders that produce the extra 2 search calls on each visit.** Code-reading establishes that they must be SSR re-runs of Call site A (since Call site B is data-cached and Call sites C/D cannot reach a backend POST on a fresh home load), but the *cause* of two RSC re-renders is not unambiguous. Most plausible single-source candidates: `FilterManager.tsx:317`'s `setTimeout(() => router.refresh(), 0)` inside SYNC TO URL when `newUrl !== current`, possibly firing twice as the post-HYDRATE state churn ripples through React. Confirming requires one of: instrumented page-component render counter, network-tab trace pairing client events to backend POST timestamps, or a React DevTools render trace.
- The `firebase-sync` 8ms-apart pair is *consistent with* the Firebase SDK's near-expiry persisted-token refresh path emitting twice through one listener. I cannot prove this from code alone; the alternative (a second listener somewhere) is ruled out by audit but not the SDK internals. To confirm, add temporary `console.log` in the `onIdTokenChanged` callback and observe the order / spacing of emissions for the same uid.

## For Mastermind

### Part 4a simplicity evidence (required)

- **Added (earned complexity):** nothing (read-only audit).
- **Considered and rejected:** nothing (read-only audit).
- **Simplified or removed:** nothing (read-only audit).

### Adjacent observations surfaced during the audit (Part 4b)

1. **`reactCalls/productsSearchService.getProducts` (`:132-154`) swallows backend errors into `EMPTY_OVERVIEWS`.** Severity: low. The catch block at `:150-153` returns `EMPTY_OVERVIEWS` for every error. A failed paginated fetch silently presents "0 results" instead of an error. The home-page initial load uses the SSR variant (different file) which has the same shape (`nextCalls/productsSearchService.ts:103-107`). Out of scope. (Same pattern across `extraProductsService.ts:21-26`, `favoriteService.ts:17-20`, etc. — would warrant a centralised handler discussion.)

2. **`FilteredProductList.tsx:31` keys on `JSON.stringify(filtersData)` including the seed.** Severity: medium — visible regression for end users on the home page (the jerk). Out of scope per brief ("Do not start fixing things you find."). Fix shape sketched in section 1 of "Fix shape sketch."

3. **`syncUserToBackend` has no idempotency / single-flight.** Severity: medium — extra backend write of `banned_user_audit` + `users` lookup on every emission. Out of scope. Fix shape sketched in section 2 of "Fix shape sketch."

4. **`writeFirebaseTokenCookie` (`src/lib/service/reactCalls/authTokenCookie.ts:35-39`) compares tokens by string identity for its dedupe.** Severity: low. Two distinct callers within the same handshake can both pass the same token and rely on the guard; if Firebase rotates the token between the two listener emissions in the 8ms window, the guard does NOT dedupe (different token strings). This is correct behaviour (a refreshed token should be re-mirrored to the SSR cookie) but worth noting because the same window is the most likely root cause of the duplicate sync — there are *two* tokens in play, which is also why the backend logs two real syncs rather than seeing the dedupe.

5. **`FilterManager.tsx:317` uses `setTimeout(() => router.refresh(), 0)` after `router.replace`.** Severity: low (questionable in Next 16). The historical reason for `setTimeout` was that `router.replace` did not consistently refetch RSC server data in earlier Next versions; in Next 16 the App Router refetches RSC on `replace` for the affected route. The `setTimeout(0)` may now be redundant or actively-double-fetching. Worth confirming when the broader fix lands.

6. **`PRODUCTS_PER_PAGE = 20` in `src/lib/utils/utils.ts:8` is used by client `ProductList.tsx:74, :84, :104` and by SSR `getInitialPage` (`:15-20`).** Severity: low. The metadata snapshot independently uses `limit = 10` (`generateHomePageMetadata.ts:23`). The two constants drift and the discrepancy is intentional (snapshot is OG image source, list is the full grid). Noting because the audit walked all three to confirm the totals.

### Drafted config-file text

- None drafted this session. The behaviour surfaced by this audit (the four-call cascade) does not yet require a `decisions.md` entry — the Mastermind decision for the fix shape would. No `issues.md` entries drafted inline per the brief's "Do not draft issues.md entries inline."

### Questions for Mastermind

- The audit produces a strong (but not bulletproof) read that two RSC re-renders fire post-mount. Want to confirm with instrumentation before the bug-fix brief is scoped? A 5-minute instrumentation pass (add `console.log('SSR render', Date.now())` in `app/[locale]/(portal)/(public)/page.tsx`, hard-refresh once, capture client console + backend access log) would close the deterministic gap.
- Q5's "favourites-icon flip during the same paint window as the random-seed jerk" is a secondary visible churn that compounds with the main jerk. Worth folding into the same fix brief (e.g. server-side render with the cookie-attached auth so the SSR response already reflects `favorite: true`), or carve it out as a separate item?

### Suggested next steps

1. Confirm the RSC-re-render trigger via instrumentation (5 min).
2. Pick fix shape for root cause 1 (`randomSeed` → key, two options sketched above).
3. Independently land a single-flight guard for root cause 2 (firebase-sync + push/token duplicates).
4. Re-measure with the same access-log methodology to confirm the cascade is gone.
