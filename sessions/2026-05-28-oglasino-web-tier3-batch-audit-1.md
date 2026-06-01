# Session summary

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-28
**Task:** READ-ONLY audit of three `issues.md` entries: useAuthResolved adoption, backend errors swallowed as "empty results", home page pagination scope unknown.

## 1. `useAuthResolved` adoption pending across the app

### Hook contract

`src/lib/hooks/useAuthResolved.ts:13-27`

```ts
export function useAuthResolved(): boolean {
  const storeUser = useAuthStore((s) => s.user);
  const [firebaseUser, setFirebaseUser] = useState<FirebaseUser | null>(null);
  const [firebaseReady, setFirebaseReady] = useState(false);

  useEffect(() => {
    const unsub = onAuthStateChanged(auth, (u) => {
      setFirebaseUser(u);
      setFirebaseReady(true);
    });
    return () => unsub();
  }, []);

  return firebaseReady && (!firebaseUser || storeUser !== null);
}
```

Returns `true` once Firebase has reported auth state **and**, if a Firebase user exists, the auth store's backend user has been synced. Three-state gate: Firebase not ready → false; Firebase says anonymous → true immediately; Firebase says authenticated → wait for `storeUser` hydration, then true.

### Current adopters (4 components)

| Component | File:line | Usage |
|---|---|---|
| `UserDetails` / `UserInfoBlock` | `src/components/client/UserDetails.tsx:49` | Gates follow/report/send-message buttons on `authResolved && !iamActive` |
| `ProductFunctions` | `src/components/client/ProductFunctions.tsx:36-38` | Returns `null` until resolved; hides action bar from owner |
| `ProductViewTracker` | `src/components/client/initializers/ProductViewTracker.tsx:31` | Defers `product_view` GA4 event until auth resolves for accurate `is_owner_view` |
| `GA4UserIdSync` | `src/components/client/initializers/GA4UserIdSync.tsx:11` | Defers `gtag('set', { user_id })` until auth resolves |

### Candidates that inline the same pattern

**A. `HeaderNavButtons.tsx` (`src/components/client/HeaderNavButtons.tsx:18-44`)**

Exact same three-state gate, inlined:
```ts
const [firebaseUser, setFirebaseUser] = useState<FirebaseUser | null>(null);
const [firebaseReady, setFirebaseReady] = useState(false);
const storeUser = useAuthStore((s) => s.user);

useEffect(() => {
  const unsub = onAuthStateChanged(auth, (u) => {
    setFirebaseUser(u);
    setFirebaseReady(true);
  });
  return () => unsub();
}, []);

const isReady = firebaseReady && (!firebaseUser || storeUser !== null);
if (!isReady) return <div className="w-45"></div>;
```

**Would `useAuthResolved` simplify?** Yes — strict simplification. Replace 13 lines with `const isReady = useAuthResolved()`. The placeholder `<div className="w-45">` is a layout-preserving shim that stays as the `!isReady` branch; the hook only replaces the gate computation.

**B. `MobileFooterNavigation.tsx` (`src/components/server/layout/MobileFooterNavigation.tsx:17-55`)**

Subscribes to `onAuthStateChanged` for `user` and `loading`:
```ts
const [user, setUser] = useState(null);
const [loading, setLoading] = useState(true);
useEffect(() => {
  const unsub = onAuthStateChanged(auth, (u) => {
    setUser(u);
    setLoading(false);
  });
  return () => unsub();
}, []);
```

Renders nothing while `loading`, then switches between anonymous and auth button sets based on `user` truthiness.

**Would `useAuthResolved` simplify?** Partially. The component doesn't wait for `storeUser` — it only waits for Firebase readiness and branches on Firebase user presence. Adopting `useAuthResolved` would add the `storeUser` gate, which is actually correct behavior (matches `HeaderNavButtons`'s pattern — both render auth button clusters that need backend user data). But `MobileFooterNavigation` also passes `user` to its children implicitly via the ternary branch, not via the store. **Recommendation:** adopt `useAuthResolved()` for the ready gate, read `firebaseUser` from the hook (requires widening the hook's return, see below) or keep a local `onAuthStateChanged` for the `user` value. The simpler path is to have it read `useAuthStore((s) => s.user)` for the auth/anon branch, matching `HeaderNavButtons`.

### Components that gate on auth but need a different pattern

**C. `SessionGuard.tsx` (`src/components/client/SessionGuard.tsx:10-59`)**

Uses `auth.authStateReady()` (a one-shot Promise that resolves when Firebase first reports), then reads `auth.currentUser` synchronously. If no user, redirects to home. If admin route, probes `/secure/admin` via `loadIsAdmin`. Sets `loading = false` only after all checks pass.

`SessionGuard` is route-level gating with redirect semantics and admin authorization. `useAuthResolved` is in-component UI gating. They solve different problems at different scopes:
- `useAuthResolved`: "should I render this button yet?" → boolean
- `SessionGuard`: "should I let this route render at all, and should I redirect if not?" → redirect + admin probe

**They should coexist.** `SessionGuard` cannot be replaced by `useAuthResolved` because it performs redirect navigation and admin authorization — behaviors the hook doesn't own. `useAuthResolved` cannot be replaced by `SessionGuard` because it operates inside already-rendered components, not at the layout boundary.

**D. `ChatsInit.tsx` (`src/components/client/initializers/ChatsInit.tsx:9`)**

Gates on `useAuthStore((s) => s.user)` directly. When `user` is null, unsubscribes from Firestore chat listeners. When non-null, subscribes. Does not use Firebase ready state at all.

**Would `useAuthResolved` help?** No. `ChatsInit` doesn't have a hydration-flash problem — it doesn't render UI. It gates side effects (Firestore subscriptions) on user presence. The existing `user !== null` check is correct and sufficient.

**E. `useNotificationsStore` (`src/notifications/hooks/useNotifications.ts:24`)**

Gates Firestore notification subscription on `user?.firebaseUid`. Same reasoning as `ChatsInit` — side-effect gating, not UI gating.

**Would `useAuthResolved` help?** No.

### `useAuthStore` and a synchronous `initialized` boolean

`useAuthStore` (`src/lib/store/useAuthStore.ts`) exposes no `initialized` or `hydrated` flag. The store starts with `user: null, loading: false`. The `UseTokenRefresh` listener (`onIdTokenChanged`) is the sole hydrator of `user`. Before the listener fires its first callback, `user: null` is indistinguishable from "genuinely anonymous" — this is the root cause of the hydration-flash problem that `useAuthResolved` works around.

**Could `useAuthStore` grow `initialized: boolean`?**

Yes, and it would simplify the ecosystem. The flag would be set to `true` by the `UseTokenRefresh` listener after its first `onIdTokenChanged` callback fires (whether the callback reports a user or null). Consumers could then gate on `initialized` instead of running their own `onAuthStateChanged` subscriptions.

**What would set it:**
- `UseTokenRefresh.tsx` — the existing sole-hydrator listener. After `syncUserToBackend` resolves (or after the null-user branch runs), call `useAuthStore.getState().setInitialized(true)`.

**What would benefit:**
- `HeaderNavButtons` — replace inline Firebase listener with `const initialized = useAuthStore((s) => s.initialized); const user = useAuthStore((s) => s.user);`
- `MobileFooterNavigation` — same
- All current `useAuthResolved` consumers — the hook could simplify to `const initialized = useAuthStore((s) => s.initialized); const user = useAuthStore((s) => s.user); return initialized;` (or the hook itself becomes unnecessary if consumers read the store directly)

**Trade-off:** The `initialized` flag is a coupling point — `UseTokenRefresh` becomes the single writer, which it already is for `user`. The flag makes the contract explicit rather than implicit. Downside: the flag must be reset on sign-out-then-sign-in (the listener re-fires, but `initialized` should stay `true` in that case — it only tracks "has the listener ever fired," not "is there a user"). Low risk.

### Estimated blast radius

| Action | Files |
|---|---|
| Adopt `useAuthResolved` in `HeaderNavButtons` | 1 file, ~13 lines removed |
| Adopt `useAuthResolved` in `MobileFooterNavigation` | 1 file, ~8 lines changed (plus switching to store `user`) |
| Add `initialized` to `useAuthStore` + set it in `UseTokenRefresh` | 2 files |
| Simplify `useAuthResolved` to read `initialized` from store | 1 file |
| **Total** | **5 files** (if doing the full pass including `initialized`), **2 files** (if only adopting the hook in the two inlining components) |

### Fix-shape recommendation

**Candidate A (minimal):** Adopt `useAuthResolved` in `HeaderNavButtons` and `MobileFooterNavigation`. Two files, purely mechanical. Leave `SessionGuard`, `ChatsInit`, `useNotificationsStore` unchanged. ~30 minutes of work.

**Candidate B (full):** Add `initialized: boolean` to `useAuthStore`, set it in `UseTokenRefresh`, simplify `useAuthResolved` to `return useAuthStore((s) => s.initialized)`, then adopt the simplified hook (or direct store reads) across all consumers. Eliminates per-component Firebase `onAuthStateChanged` subscriptions. ~2 hours of work, more moving parts, more test surface.

**Recommendation:** Candidate A first (immediate simplification, no new state), Candidate B as a follow-up if the `initialized` flag is needed elsewhere.

---

## 2. Backend errors swallowed as "empty results"

### The try/catch shape in `nextCalls/productsSearchService.ts`

The file has two functions with error handling:

**`getProductDetails` (line 49-60):**
```ts
try {
  const res = await FETCH_BACKEND_API.get<ProductDetailsDTO>(uri, fetchOptions);
  if (res.status >= 200 && res.status < 300) {
    return res.data;
  }
  logServiceWarn('product.next.getDetails', res);
  return null;
} catch (err) {
  logServiceError('product.next.getDetails', err);
  return null;
}
```
Returns `null` on error — not `EMPTY_OVERVIEWS`.

**`getProducts` (line 85-108) — the function the brief targets:**
```ts
try {
  const res = await FETCH_BACKEND_API.post<ProductOverviewsDTO>(uri, { productsFilter, paging }, { ... });
  if (res.status >= 200 && res.status < 300) {
    return res.data;
  }
  logServiceWarn('product.next.search', res);
  return EMPTY_OVERVIEWS;
} catch (err) {
  logServiceError('product.next.search', err);
  return EMPTY_OVERVIEWS;
}
```

### The shape is symmetric across all three scopes

There is only **one** `getProducts` function that takes `portalScope: PortalScope` as a parameter. `getPortalProducts`, `getDashboardProducts`, and `getAdminProducts` are thin wrappers:

```ts
export const getPortalProducts = async (f, p) => getProducts(f, p, 'portal');
export const getDashboardProducts = async (f, p) => getProducts(f, p, 'owner');
export const getAdminProducts = async (f, p) => getProducts(f, p, 'admin');
```

All three share the identical try/catch/return-`EMPTY_OVERVIEWS` shape. No scope has a more granular catch.

### Where `EMPTY_OVERVIEWS` is defined

`src/lib/data/emptyOverviews.ts:3-6`:
```ts
export const EMPTY_OVERVIEWS: ProductOverviewsDTO = {
  products: [],
  totalNumberOfProducts: 0,
};
```

### Logging before the sentinel return

Yes — both branches log:
- Non-2xx response: `logServiceWarn('product.next.search', res)` → `console.warn` with status and errorCode
- Thrown error: `logServiceError('product.next.search', err)` → `console.error` (or `console.warn` for 403s)

**Server-side visibility:** `nextCalls` services run server-side (`'use server'` directive), so logs go to the Node.js process (Vercel serverless function logs). They are observable in Vercel's runtime logs dashboard. **No structured logging pipeline exists** — these are `console.*` calls, not a logging framework with log levels, correlation IDs, or alerting integration.

### Downstream consumers of `EMPTY_OVERVIEWS`

The SSR product search return value flows to:
1. **Home page** (`app/[locale]/(portal)/(public)/page.tsx:68`): `initialProductsData?.totalNumberOfProducts > 0` → renders product list. If `EMPTY_OVERVIEWS`, shows "no products yet" message.
2. **Catalog page** (`app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx:191`): Same pattern — renders product list or "no products" message.
3. **User page** (`app/[locale]/(portal)/(public)/user/[userId]/page.tsx:93`): `productsData.products.length > 0` → renders product list or "no products found" message.
4. **Owner products** (`app/[locale]/owner/products/page.tsx:48`): Same pattern.
5. **Admin products** (`app/[locale]/admin/products/page.tsx:54`): Same pattern.

In every case, a backend outage produces the "no products" empty state. The user sees a functioning page with zero results — indistinguishable from "the category is genuinely empty."

### Whether the same pattern exists elsewhere in `nextCalls/`

**Every `nextCalls/` service uses the same pattern:**

| Service | File | Error return |
|---|---|---|
| `productsSearchService.getProducts` | `nextCalls/productsSearchService.ts:104,107` | `EMPTY_OVERVIEWS` |
| `productsSearchService.getProductDetails` | `nextCalls/productsSearchService.ts:56,59` | `null` |
| `favoritesService.getFavorites` | `nextCalls/favoritesService.ts:30,33` | `EMPTY_OVERVIEWS` |
| `userService.getUserForId` | `nextCalls/userService.ts:27,30` | `null` |
| `productService.markAsSeen` | `nextCalls/productService.ts:9` | (void — swallows silently) |
| `metadataSnapshotService.getMetadataProductsSnapshot` | `nextCalls/metadataSnapshotService.ts:70,92` | `[]` |

**Every `reactCalls/` service uses the same pattern:**

| Service | File | Error return |
|---|---|---|
| `productsSearchService.getProducts` | `reactCalls/productsSearchService.ts:147,150,153` | `EMPTY_OVERVIEWS` |
| `productsSearchService.getProductDetails` | `reactCalls/productsSearchService.ts:45,48` | `null` |
| `productsSearchService.getAutocompleteSuggestions` | `reactCalls/productsSearchService.ts:101,104,107` | `[]` |
| `extraProductsService.fetchExtraProducts` | `reactCalls/extraProductsService.ts:22,25` | `EMPTY_OVERVIEWS` |
| `favoriteService.getAllFavoriteIds` | `reactCalls/favoriteService.ts:16,19` | `[]` |
| `favoriteService.getFavorites` | `reactCalls/favoriteService.ts:41,44,47` | `EMPTY_OVERVIEWS` |
| `baseSiteService.getAllBaseSitesOverviews` | `reactCalls/baseSiteService.ts:15,18` | `[]` |
| `baseSiteService.getBaseSiteRegions` | `reactCalls/baseSiteService.ts:33,36` | `[]` |
| `userService.updateUser` | `reactCalls/userService.ts:30,33` | `false` |
| `userService.getUserPhoneNumber` | `reactCalls/userService.ts:70,73` | `null` |
| `userService.getUserForFirebaseUid` | `reactCalls/userService.ts:88,91` | `null` |
| `userService.getUserFollows` | `reactCalls/userService.ts:122-133` | `{ users: [], totalNumberOfUsers: 0 }` |
| `reviewService.getReviews` | `reactCalls/reviewService.ts:125,128` | `undefined` |
| `reviewService.getMyReviews` | `reactCalls/reviewService.ts:150,152` | `undefined` |
| `reportService.sendReport` | `reactCalls/reportService.ts:30,31` | `{ success: false, alreadyReported: false }` |
| `reviewService.reviewProduct` | `reactCalls/reviewService.ts:94,96` | `false` |

**The pattern is project-wide.** Every service layer function (both SSR and client) catches errors and returns a type-safe empty sentinel. No service propagates errors to consumers.

### Report-submit endpoint independence

The `issues.md` 2026-05-16 "Report-submit endpoint trust-boundary verification unknown" entry references `reactCalls/reportService.ts:12` — the `sendReport` function that POSTs to `/secure/report/add`. This is a completely different endpoint (`/secure/report/add`) from the product search services (`/public/product/search`, `/secure/products`, `/secure/admin/products`). Different controller, different service, different trust-boundary concern (authorization validation vs error propagation). **Fully independent — does not affect this fix.**

### Fix-shape candidates

**Candidate A — Discriminated union, consumer error states:**

Change every service function from:
```ts
// Before
async function getProducts(...): Promise<ProductOverviewsDTO> {
  try { ... } catch { return EMPTY_OVERVIEWS; }
}
```
To:
```ts
// After
type ServiceResult<T> = { ok: true; data: T } | { ok: false; error: string };
async function getProducts(...): Promise<ServiceResult<ProductOverviewsDTO>> {
  try { ... return { ok: true, data: res.data }; }
  catch { return { ok: false, error: 'product.search.failed' }; }
}
```

**Blast radius of A:** Every consumer of every service function must be updated. From the audit above:
- `nextCalls/productsSearchService`: 7 consumer pages (home, catalog, user, owner products, admin products, admin products by user, sitemap)
- `nextCalls/favoritesService`: 1 consumer page (favorites)
- `reactCalls/productsSearchService`: 3 consumer components (`SelectableFilterProductListWrapper` handlers)
- `reactCalls/favoriteService`: 2 consumer components (`FavoriteProductList`, `useFavoritesStore`)
- Plus every other service's consumers across the app

**Estimate: 30+ files.** This is a large refactor. Correct but expensive.

**Candidate B — Keep sentinel returns, add structured server-side logging:**

Keep `EMPTY_OVERVIEWS` returns but replace `console.error/warn` in `logServiceError/logServiceWarn` (`src/lib/utils/serviceLog.ts`) with a structured logger that integrates with Vercel's logging pipeline (or a third-party service). Add operation name, timestamp, request URL, response status, and error class to each log entry. Backend outages become observable without changing any consumer code.

**Blast radius of B:** 1 file (`serviceLog.ts`). No consumer changes.

**Candidate C — Hybrid: error-state propagation for user-facing product lists only:**

Apply Candidate A's discriminated union to `nextCalls/productsSearchService.getProducts` and `reactCalls/productsSearchService.getProducts` only (the product search endpoints that render the primary product lists). Update the 7 SSR consumer pages and 3 client-side consumer components to render an error state ("Something went wrong — try again") instead of the "no products" message. Leave all other services unchanged.

**Blast radius of C:** ~10 files. Targeted, high-impact fix for the most visible symptom.

### Recommendation

**Candidate C** for the fix brief, **Candidate B** as an immediate low-cost improvement to logging. The user-visible product list is the surface where "error looks like empty" matters most. Other services (favorites, user details, reviews) have lower traffic and lower confusion potential.

---

## 3. Home page only loads first page; pagination scope unknown

### Code trace: page 1 → page 2 click on the home page

**SSR render:**
1. `app/[locale]/(portal)/(public)/page.tsx:40-44`: calls `filterHydrationSSR(searchParams, baseSite, t, true)` → returns `{ applyRandom: true, randomSeed: Date.now() }` (no filters on home page)
2. `page.tsx:41-44`: calls `getPortalProducts(filtersData, getInitialPage())` via `nextCalls/productsSearchService` (server-side `FETCH_BACKEND_API`) → returns page 0 data with `totalNumberOfProducts`
3. `page.tsx:69-74`: passes `initialProductsData` and `filtersData` to `SelectableFilterProductListWrapper`

**Client-side component chain:**
4. `SelectableFilterProductListWrapper.tsx:32-41`: creates `onNextPortalPageInternal` closure that returns `Promise.resolve(initialProductsData)` for page 0, calls `getPortalProducts({ ...filtersData }, paging, signal)` from `reactCalls/productsSearchService` for page > 0
5. `FilteredProductList.tsx:31-35`: strips `randomSeed` from key (fix from 2026-05-21), passes `onNextPageInternal` as `onNextPage` to `ProductList`
6. `ProductList.tsx:68-76`: starts with `products = initialProductsData.products`, `displayPage = 0`, calculates `totalPages = Math.ceil(totalNumberOfProducts / 20)`

**User clicks page 2 button:**
7. `ProductList.tsx:80-101`: `onDisplayPageChange(1)`:
   - Aborts any in-flight request
   - Creates new `AbortController`
   - `setLoading(true)`, `setDisplayPage(1)`
   - Calls `onNextPage({ page: 1, perPage: 20 }, signal)`
   - On success: `setProducts(result.products)`, `setLoading(false)`, scrolls to top

8. **The `onNextPage` call chain:**
   - `SelectableFilterProductListWrapper.onNextPortalPageInternal` → page > 0 → calls `getPortalProducts({ ...filtersData }, { page: 1, perPage: 20 }, signal)`
   - `reactCalls/productsSearchService.getPortalProducts` → `getProducts(filter, paging, 'portal', signal)`
   - `getProducts` → `BACKEND_API.post('/public/product/search', { productsFilter: { applyRandom: true, randomSeed: <ts> }, paging: { page: 1, perPage: 20 } }, { signal })`
   - `BACKEND_API` request interceptor adds `Authorization: Bearer <token>` (if logged in), `X-Base-Site`, `X-Lang`
   - On success: returns `res.data`
   - On error: returns `EMPTY_OVERVIEWS`

### Identifying the actual broken behavior

**The web-side code path is structurally correct.** I traced every step and found no code-level bug in the pagination flow. The data flows correctly from button click → service call → state update → render.

**Three hypotheses for the observed symptom:**

**Hypothesis A — Backend doesn't paginate random-ordered results correctly.**

The home page sends `{ applyRandom: true, randomSeed: <ts> }` with `{ page: 1, perPage: 20 }`. If the backend's Elasticsearch query applies random scoring but doesn't support stable pagination with the same seed across pages (e.g., it re-scores and re-sorts on every request, making page 1's results overlap with or duplicate page 0's), the user would see wrong or empty results on page 2.

Evidence for: the home page is the only surface that always uses `applyRandom: true`. If the symptom is home-page-specific, random ordering is the distinguishing factor.

Evidence against: the `randomSeed` is a `Date.now()` timestamp sent as-is to the backend. If the backend uses it as an Elasticsearch `random_score` seed, pagination should be stable. But this requires backend verification.

**Hypothesis B — Client-side API call fails silently, returns `EMPTY_OVERVIEWS`.**

The `reactCalls/productsSearchService.getProducts` catch block returns `EMPTY_OVERVIEWS` on any error. If the client-side POST to `/public/product/search` fails (CORS, network, backend 500, auth interceptor error), the user sees an empty products list on page 2 with no error indicator.

Evidence for: this is consistent with "only loads first page" — page 0 works (SSR), page 1+ silently fails (client-side).

Evidence against: other paginated pages (catalog, user, owner, admin) share the same `getProducts` → `BACKEND_API.post` path. If the client-side call is fundamentally broken, all pagination would fail everywhere, not just the home page.

**Hypothesis C — `initialProductsData` reference change resets pagination.**

If anything triggers a server component re-render (FilterManager `router.replace`, soft navigation, RSC refetch), the `initialProductsData` prop changes by reference. `ProductList`'s SYNC INITIAL DATA effect (`ProductList.tsx:136-139`) fires and resets `products` and `displayPage` to page 0. The user's page 2 click would be overridden.

Evidence for: the SYNC INITIAL DATA effect has `[initialProductsData]` as its dependency, comparing by reference. Any server-component re-render creates a new object.

Evidence against: the FilterManager trailing-slash false-positive was fixed (issues.md 2026-05-21). After that fix, FilterManager should not trigger `router.replace` on the home page with no filters. Pagination clicks don't change the URL. This hypothesis requires a secondary trigger.

### Whether other paginated pages have the same problem

**All pages use the same component chain:** `SelectableFilterProductListWrapper` → `FilteredProductList` → `ProductList`. The only differences are:
- **Home page:** `portalScope='portal'`, `applyRandom=true`, `filtersData = { applyRandom: true, randomSeed: <ts> }`
- **Catalog page:** `portalScope='portal'`, `applyRandom=true` (if no search/order), `filtersData` includes category IDs
- **User page:** `portalScope='portal'`, `filtersData = { ownerId: userId }` — no `applyRandom`
- **Owner products:** `portalScope='owner'`, `applyRandom=false`
- **Admin products:** `portalScope='admin'`, `applyRandom=false`
- **Favorites:** uses `FavoriteProductList` → `ProductList` directly (no page-0 optimization, no `SelectableFilterProductListWrapper`)

If **Hypothesis A** is correct (random ordering + pagination), the catalog page would also be affected (it uses `applyRandom: true` for no-filter/no-search states). User page, owner, admin, and favorites would be unaffected (no `applyRandom`).

If **Hypothesis B** is correct (client-side API failure), all pages would be affected equally.

If **Hypothesis C** is correct (initialProductsData reference reset), all pages would be affected equally — but the FilterManager trailing-slash fix specifically targeted the home page's `pathname === '/'` case, so the home page might be the only one where the trigger existed.

### Whether the AbortController work (Tier 2) is orthogonal

The Tier 2 AbortController addition (`ProductList.tsx:72-73, 81-96`) added proper request cancellation on rapid page clicks and component unmount. The abort ref is shared with the favorites effect (`ProductList.tsx:107-131`), but the favorites effect only runs when `pathname === '/favorites'`, so on the home page the abort ref is only touched by `onDisplayPageChange`.

**The AbortController work is orthogonal.** It prevents race conditions (double-fetching, stale data from a previous page request arriving after a newer one), but it doesn't change whether pagination works at all. If pagination was broken before the AbortController work, it's still broken. If it was working, the AbortController doesn't break it.

### Whether this is a regression or always-broken

**Cannot determine from code alone.** The code path is structurally sound. If the issue is a backend behavior (Hypothesis A), it would have been always-broken for the home page specifically. If the issue is a client-side API failure (Hypothesis B), it could be a regression from API configuration changes. If the issue is a reference-reset (Hypothesis C), the FilterManager fix in 2026-05-21 should have resolved it.

**The most productive next step is a live reproduction test** on the current `stage` branch:
1. Navigate to the home page with > 20 products in the base site
2. Click page 2 button
3. Open browser DevTools Network tab: does a POST to `/public/product/search` fire?
4. If yes: what's the response? Is it empty, a duplicate of page 0, or correct page 1 data?
5. If no: what prevented the request?

### Fix-shape recommendation

**If Hypothesis A (backend random pagination):**
- Backend fix: ensure Elasticsearch `random_score` uses the `randomSeed` parameter as a stable seed across paginated requests. Verify that `from: 20, size: 20` with the same seed returns different documents than `from: 0, size: 20`.
- Web change: none needed.

**If Hypothesis B (client-side API failure):**
- Debug the specific failure (CORS, auth interceptor, backend rejection). Fix the root cause.
- Additionally: this overlaps with Section 2 — adopt error-state propagation (Section 2 Candidate C) so failures are visible rather than silent.

**If Hypothesis C (reference reset):**
- Stabilize the `initialProductsData` reference in `ProductList`. The SYNC INITIAL DATA effect at `ProductList.tsx:136-139` could compare content instead of reference:
  ```ts
  useEffect(() => {
    if (displayPage === 0) {
      setProducts(initialProductsData.products);
    }
  }, [initialProductsData]);
  ```
  Only reset products if the user is on page 0 (meaning a genuine filter change occurred and remounted the component via the key). If on page > 0, ignore the reference change — the user's pagination state is more important than a stale-reference update.

**Recommendation:** Start with a live reproduction test to determine which hypothesis is correct. The web code is structurally sound; the most likely root cause is either backend behavior with random pagination or a transient environmental issue during the original smoke test.

---

## Implemented

- Read-only audit of three `issues.md` entries. No code changes.

## Files touched

- None (read-only audit)

## Tests

- Not applicable (read-only)

## Cleanup performed

- None needed

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change (three entries remain `open` pending fix briefs)

## Obsoleted by this session

- Nothing

## Conventions check

- Part 4 (cleanliness): confirmed (no code changes)
- Part 4a (simplicity): N/A (read-only audit)
- Part 4b (adjacent observations): see "For Mastermind" section
- Part 6 (translations): N/A this session
- Other parts touched: Part 7 (error contract) — consulted for Section 2 analysis; Part 11 (trust boundaries) — consulted for report-submit independence check

## Known gaps / TODOs

- Section 3 requires a live reproduction test to determine root cause. The web code trace is structurally sound; the bug is either in the backend's random-pagination behavior or in a transient condition not reproducible from code reading alone.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only)
  - Considered and rejected: nothing (read-only)
  - Simplified or removed: nothing (read-only)

- **Section 1 — fix brief scoping:** Candidate A (adopt `useAuthResolved` in `HeaderNavButtons` and `MobileFooterNavigation`) is a clean, mechanical 2-file change. Candidate B (add `initialized` to `useAuthStore`) is a design decision worth Mastermind input — it changes the auth contract and should be deliberate.

- **Section 2 — project-wide pattern:** The error-swallowing pattern is not limited to `productsSearchService`. It's in every service layer function across both `nextCalls/` (6 services) and `reactCalls/` (14+ services). A full Candidate A (discriminated union) refactor touches 30+ files. Candidate C (targeted to product search only) is the pragmatic scope for a fix brief. Candidate B (structured logging) should happen regardless — it's independent and low-risk.

- **Section 3 — backend verification needed:** The most likely root cause is Hypothesis A (backend doesn't support stable pagination with random ordering). This requires a backend audit or a live reproduction test. The web code is correct. If Mastermind briefs a fix, it should include a "reproduce first" step before any code changes.

- **Adjacent observation (Part 4b):**
  - `MobileFooterNavigation.tsx` (`src/components/server/layout/MobileFooterNavigation.tsx`) has filename prefix `server/` but is a `'use client'` component. The file is in `src/components/server/layout/` which is misleading — all its siblings may also be client components. Low severity, cosmetic only. I did not fix this because it is out of scope.
