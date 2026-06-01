# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-27
**Task:** Diagnose Φ3 regression — 400 on `POST /api/public/product/search` at boot before user navigation.

## Implemented

- Read suspected files end-to-end: `api.ts`, `productsSearchService.ts`, `AppContext.tsx`, `useFilterStore.ts`, `AppInit.tsx`, `ProductList.tsx`, `FilteredProductList.tsx`, `apiStore.ts`, all six layout files, and the portal home page `index.tsx`.
- Confirmed `products.portal.getList` is at `src/lib/services/productsSearchService.ts:114` — the `getProducts()` function at line 92 posts to `/public/product/search` and uses the tag on warning/error log.
- Traced the full call chain: `ProductList.tsx:101-103` (`useEffect(() => onRefresh(), [fetchPage])`) → `loadNextPage(true)` → `fetchPageInternal(0)` in `FilteredProductList.tsx` → `getPortalProducts()` → `getProducts()` → `BACKEND_API.post('/public/product/search', ...)`.
- Added three temporary instrumentation patches (all marked `// TEMPORARY Φ3 diagnostic — remove in patch session`):
  - **Instrumentation A** — axios request interceptor body + header log (`api.ts:38-44`)
  - **Instrumentation B** — call-site stack trace + inputs log (`productsSearchService.ts:100-105`)
  - **Instrumentation C** — ProductList mount/effect trigger log with state snapshot (`ProductList.tsx:101-107`)
- Igor ran the app on device and captured runtime logs. Diagnosis confirmed.

## Confirmed diagnosis (runtime logs received)

**Root cause: the Φ2 always-mounted `<Stack>` causes the portal home page to mount and fire a product search before bootstrap sets `currentBaseSiteCode`/`currentLangCode` in `apiStore.ts`.**

### Runtime evidence

1. **`[Φ3-diag] ProductList mount/fetchPage change`** — `selectedLanguage: undefined` confirms bootstrap hasn't completed when the effect fires.
2. **`[Φ3-diag] products.portal.getList called`** — request body is structurally valid (default filter state, page 0, perPage 20). Stack trace shows the call originates from React's `commitPassiveMountOnFiber` → `recursivelyTraversePassiveMountEffects` chain — a mount-phase passive effect, not user navigation.
3. **`[Φ3-diag] product/search request`** — `headers: {"X-Base-Site": null, "X-Lang": null}`. The 400 is from the null headers, not from the body.
4. **Error log:** `[products.portal.getList] threw [AxiosError: Request failed with status code 400]`.

### Diagnosis

- **Root cause:** Φ2's always-mounted `<Stack>` in root layout causes the portal home page (`app/(portal)/(public)/index.tsx`) to mount and fire a product search via `ProductList.tsx:101-103`'s `useEffect(() => onRefresh(), [fetchPage])` before `AppContextProvider.bootstrap()` has set `currentBaseSiteCode`/`currentLangCode` in `apiStore.ts`.
- **Failing field:** `X-Base-Site` header is `null`. `X-Lang` header is `null`. The request body is structurally valid — the 400 is entirely from the missing/invalid headers.
- **Call site:** `ProductList.tsx:101-103` → `onRefresh()` → `loadNextPage(true)` → `fetchPageInternal(0)` (FilteredProductList.tsx:100-117) → `getPortalProducts()` (productsSearchService.ts:128-133) → `getProducts()` (productsSearchService.ts:92-126) → `BACKEND_API.post('/public/product/search', ...)`.
- **State source:** `apiStore.ts:1-2` — `currentBaseSiteCode` and `currentLangCode` are initialized to `null`. They are only set by `AppContext.tsx:127-128` during bootstrap's ready path, which runs as a parent effect AFTER the child ProductList effect.
- **Φ2 brief responsible:** The regression was introduced by the Φ2 root layout change (visible in `git diff HEAD -- app/_layout.tsx`). The old layout gated `<Slot />` on `status === 'ready' || status === 'loading'`; since root layout's local `status` starts as `undefined` (from `useState<AppStateValue['status']>()`), children never mounted until bootstrap completed and `onStatusChanged` changed status to `'ready'`. The new layout always mounts `<Stack />`, removing this implicit gate. The brief labels this "Φ3" but the causal change is the Φ2 navigator rewrite.
- **Not a circular-import issue.** The brief suspected circular-import-induced `undefined`. The `apiStore.ts` module has zero imports and cannot participate in a cycle. The actual issue is mount-order timing: child effects fire before parent effects in React, so `ProductList`'s effect fires before `AppContextProvider`'s bootstrap effect.
- **Proposed fix shape:** Gate the `ProductList`'s initial fetch on a valid base site. The simplest fix: add an early return in the `useEffect(() => onRefresh(), [fetchPage])` when `selectedLanguage` is undefined (it already reads `selectedLanguage` from `useAppState()` at line 49). The product search should not fire until the app is in `ready` state with a valid base site and language. Alternative: gate the axios interceptor to reject/queue requests when `currentBaseSiteCode` is null — more defensive but higher blast radius.

## Files touched

- src/lib/config/api.ts (+7 / -0) — Instrumentation A
- src/lib/services/productsSearchService.ts (+6 / -0) — Instrumentation B
- src/components/product/ProductList.tsx (+5 / -0) — Instrumentation C

## Tests

- Ran: `npx tsc --noEmit` — exit 0
- Ran: `npm run lint` — 0 errors, 73 warnings (pre-existing baseline)
- Ran: `npm test` — 109 passed, 0 failed
- No new tests added (diagnostic-only session)

## Cleanup performed

Three `console.log` calls added as temporary diagnostic instrumentation, each marked with `// TEMPORARY Φ3 diagnostic — remove in patch session`. These are intentional per brief scope and tracked for removal in the patch session.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

Nothing.

## Conventions check

- Part 4 (cleanliness): three `console.log` calls added intentionally as temporary diagnostic instrumentation per brief scope. All marked for removal. No other violations.
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): two observations flagged in "For Mastermind"
- Part 6 (translations): N/A this session
- Other parts touched: Part 7 (error contract) — the 400 is a backend response to a request with null `X-Base-Site`/`X-Lang` headers, consistent with the error contract (400 = Jakarta constraint failed on a required field the backend expects in the header)

## Known gaps / TODOs

- Instrumentation stays in place per brief. Removed in the patch session alongside the fix.

## For Mastermind

### Part 4a simplicity evidence (required)

- **Added (earned complexity):** nothing. Three `console.log` lines are temporary instrumentation, not architectural additions.
- **Considered and rejected:** adding a guard in `ProductList` to skip `onRefresh()` when `selectedLanguage` is undefined — this would mask the symptom, not diagnose it. Out of scope per brief's "no fixes" rule.
- **Simplified or removed:** nothing.

### Confirmed diagnosis — for patch brief

- **Root cause:** Φ2's always-mounted `<Stack>` in root layout removes the pre-Φ2 implicit gate that prevented child routes from mounting before bootstrap. `ProductList`'s mount effect fires a product search with `X-Base-Site: null` and `X-Lang: null` headers. Backend returns 400.
- **Failing field:** `X-Base-Site` header = `null`. `X-Lang` header = `null`. Body is valid.
- **Call site:** `ProductList.tsx:101-103` (`useEffect(() => onRefresh(), [fetchPage])`) during React mount-phase passive effects.
- **State source:** `apiStore.ts:1-2` — module-scope variables initialized to `null`, only set during bootstrap's ready path at `AppContext.tsx:127-128`.
- **Φ brief responsible:** Φ2 (navigation foundation). Root layout change from conditional `<Slot />` to always-mounted `<Stack />`.
- **Proposed fix shape:** Gate `ProductList`'s initial fetch on `selectedLanguage` being defined (early return when undefined). This already-available state from `useAppState()` at `ProductList.tsx:49` is `undefined` before bootstrap and defined after — a natural gate. Alternatively, guard at the axios interceptor level (reject requests when `currentBaseSiteCode` is null), but that's broader scope and could interfere with other pre-bootstrap requests like the bootstrap calls themselves.

### Part 4b adjacent observations

1. **`console.error('Something went wrong!')` in `ProductList.tsx:93`** — bare error log with no context (no error object, no URL, no status code). Low severity. Path: `src/components/product/ProductList.tsx:93`. Not fixed because out of scope.

2. **`console.error('Failed to refresh current products:', error)` in `ProductList.tsx:186`** — slightly better (includes the error object) but still ad-hoc logging rather than using the `logServiceError` pattern from the service layer. Low severity. Path: `src/components/product/ProductList.tsx:186`. Not fixed because out of scope.

### Instrumentation cleanup tracking

Three instrumentation sites to remove in the patch session:
1. `src/lib/config/api.ts:38-44` — request interceptor log
2. `src/lib/services/productsSearchService.ts:100-105` — call-site stack trace
3. `src/components/product/ProductList.tsx:101-107` — mount trigger log

All marked with `// TEMPORARY Φ3 diagnostic — remove in patch session`.
