# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-28
**Task:** Trace the complete app-initialization sequence from process start to first product-search request, and explain why `POST /public/product/search` fires with `X-Base-Site: null` / `X-Lang: null` headers despite the Brief 15 barrier that should hold it.

## Implemented

- Read-only audit. No code changes. No instrumentation added.
- All nine initialization files read end-to-end (`package.json` entry, `app/_layout.tsx`, `AppContext.tsx`, `api.ts`, `apiStore.ts`, `ProductList.tsx`, `FilteredProductList.tsx`, the portal layout chain, `AppInit.tsx` plus `AppVersionConfigInit.tsx`).
- Brief 12–15 session summaries read for prior context.
- Module-evaluation order and require-cycle topology traced.
- Axios 1.13.6 async-interceptor semantics verified against installed source.
- Q1–Q7 answered explicitly below.

## Files touched

None. Read-only.

## Tests

Not run. No code changes.

## Cleanup performed

None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

Nothing.

## Conventions check

- Part 4 (cleanliness): N/A — no code changes
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): three observations flagged in "For Mastermind"
- Part 6 (translations): N/A this session
- Other parts touched: Part 8 (architectural defaults) — the barrier sits at the shared HTTP layer per Part 8's "routes are reusable" intent; conclusions are consistent.

## Known gaps / TODOs

- Q7 cannot be resolved from static reading with full certainty. A single named runtime datum is given in "For Mastermind" to settle it; the fix-shape proposal does not depend on that datum.

## For Mastermind

### Part 4a simplicity evidence (required)

- **Added (earned complexity):** nothing (read-only session).
- **Considered and rejected:** nothing (read-only session).
- **Simplified or removed:** nothing (read-only session).

### Part 4b adjacent observations

1. **Stale TODO in `AppVersionConfigInit.tsx:29`** — `// TODO: Translations PLUS after meintanence page do better check`. Misspelled, no matching `issues.md` entry. Severity: low (cosmetic).
2. **`console.log('[DBG]', ...)` still present in `src/lib/config/api.ts:36-45`.** Per brief, it stays until the fix brief removes it. Severity: low.
3. **`configurationService.tsx` return-type mismatch** — declared `Promise<ConfigMap>`, returns `null` on failure (`src/lib/services/configurationService.tsx:15,18`). Already tracked in `issues.md` (2026-05-27 per Brief 13). No new flag.

---

## Step-by-step trace — module evaluation and runtime execution

### Module evaluation (cold start)

1. `expo-router/entry` (per `package.json:3` `"main"`) is the JS bundle entry. It boots expo-router and resolves `app/_layout.tsx` as the root layout.
2. `app/_layout.tsx` imports `AppContextProvider` (from `src/components/context/AppContext.tsx`), `AppInit`, `BaseSiteSelector`, `AppVersionConfigInit`, the `Stack` from `expo-router`, the dialog manager, etc. The transitive import graph eagerly evaluates `api.ts`, `apiStore.ts`, `authStore.ts`, `authService.ts`, the service modules, and the i18n loader.
3. Require cycle: `api.ts` (line 3) → `authStore.ts` (line 13) → `authService.ts` (line 15) → `api.ts`. Metro emits a warning about this. The cycle is benign for the interceptor: `authService.ts` references `BACKEND_API` only inside function bodies (no top-level call sites that read it at evaluation time). Once all modules finish evaluating, every consumer's later runtime call to `BACKEND_API` sees the fully-configured singleton instance.
4. `apiStore.ts` is a **leaf** module — zero imports. Imported from exactly two places: `api.ts` (the interceptor reads `getCurrentBaseSiteCode`, `getCurrentLangCode`, `waitForBootstrap`) and `AppContext.tsx` (the bootstrap success/action paths call `setCurrentBaseSiteCode`, `setCurrentLangCode`). Both paths resolve to the same absolute file (Babel `module-resolver` alias `@` → `./src` per `babel.config.js`). Module is a Metro singleton; `bootstrapPromise` is a single instance shared by interceptor and setter.
5. `api.ts:25-146` — `createApiInstance` creates the axios instance, attaches the request interceptor (lines 35-59), attaches the response interceptor (lines 61-143), returns. `api.ts:148` `export const BACKEND_API = createApiInstance(BACKEND_API_URL)`. The instance is exported with **both interceptors already attached**.

### Runtime — cold start to first product search

1. React mounts `RootLayout`. `useState<AppStateValue['status']>()` initializes the local `status` to `undefined` (`app/_layout.tsx:28`).
2. `AppContextProvider` mounts. `useState(defaultState)` sets internal `state.status` to `'loading'` (`AppContext.tsx:60`). This does **not** call `onStatusChanged` (the callback only fires inside `setStateWithStatusHook` when `prev.status !== next.status`, and the initial render has no prior status).
3. `AppContextProvider`'s init effect (`AppContext.tsx:159-184`) fires. `bootstrap()` is called. `Promise.all([checkIfMaintenance(), getAppConfiguration(), fetchBaseSites()])` kicks off three parallel `BACKEND_API.get` calls, each with `{ _bootstrap: true }`.
4. `AppVersionConfigInit` mounts. Its `useState(true)` for `preparing` initializes the spinner. Its mount effect (`AppVersionConfigInit.tsx:103`) calls `checkVersion(true)` → `getAppVersionConfig()` → `BACKEND_API.get('/public/app/version/android?...', { _bootstrap: true })`.
5. Four pre-bootstrap requests now in flight. Each one hits the request interceptor:
   - `[DBG]` logs the URL, presence of `_bootstrap` flag, and whether codes are null.
   - The barrier check `if (!getCurrentBaseSiteCode() && !config._bootstrap)` evaluates `(true && false) = false` for all four (they carry `_bootstrap: true`). The barrier is skipped.
   - Headers are set: `X-Base-Site` → `null`, `X-Lang` → `null`. `auth.currentUser` is `null` at cold start, so the Authorization block is skipped.
   - The interceptor returns; axios dispatches the request.
6. While the four bootstrap requests are in flight, `AppVersionConfigInit` is rendering `<View><ActivityIndicator /></View>` (lines 120-126). **The children — including `<Stack>`, `<DialogManager />`, and the overlays — are not rendered.** This is crucial: the portal home screen does not mount yet.
7. `getAppVersionConfig()` resolves (success or failure — `checkVersion` always reaches `setPreparing(false)` at line 99). `preparing` flips to `false`. `AppVersionConfigInit` re-renders, returning `<>{children}<Modal/></>`.
8. Now the children mount. `<Stack screenOptions={{ headerShown: false }} />` mounts (`app/_layout.tsx:64`). Expo-router resolves the default route. Since the user has no override, this is `(portal)/(public)/index.tsx`.
9. `(portal)/_layout.tsx` mounts. `selectedBaseSite` is undefined (bootstrap hasn't reached the success branch), so the chrome (`TopBar`, `CategoryNavigation`, `ConsumerProtectionBanner`) is guarded out (`(portal)/_layout.tsx:16`). The `<Tabs>` is **not** guarded — it mounts unconditionally.
10. `(portal)/(public)/_layout.tsx` mounts (a `<Stack>`). The default route `index.tsx` mounts. `<FilteredProductList useFilterStore={usePortalFilterStore} fetchPage={getPortalProducts} applyRandom={true} CardContentComponent={PortalProductCard} />` mounts. Inside, `<ProductList />` mounts (line 122).
11. `ProductList.tsx:101-103` — `useEffect(() => { onRefresh(); }, [fetchPage])` fires unconditionally on mount (no guard on `selectedLanguage`, `selectedBaseSite`, or status). `onRefresh` → `loadNextPage(true)` → `fetchPage(0)` → `fetchPageInternal(0)` (in `FilteredProductList.tsx:100-117`) → `getPortalProducts(filters, { page: 0, perPage: 20 })` → `getProducts(...)` (`productsSearchService.ts:92-126`) → `BACKEND_API.post('/public/product/search', { productsFilter, paging })` (line 105).
12. The product-search request reaches the interceptor. `_bootstrap` is undefined. The DBG log records this. The barrier check `if (!getCurrentBaseSiteCode() && !config._bootstrap)` evaluates `(true && true) = true`. Control reaches `await waitForBootstrap()`.

Steps 1–11 are the unambiguous path. Step 12 is where the static reading and the runtime observation disagree — see Q7.

---

## Explicit answers — Q1–Q7

### Q1 — Interceptor registration timing

**Answered: the interceptor is attached before `BACKEND_API` is observable to any consumer.**

`createApiInstance` (`api.ts:25-146`) is a pure function. It (a) calls `axios.create`, (b) attaches the request interceptor, (c) attaches the response interceptor, (d) returns the instance. `BACKEND_API` is the function's return value (`api.ts:148`). Both interceptors are baked into the instance before `BACKEND_API` exists as a binding.

The require cycle (`api.ts → authStore.ts → authService.ts → api.ts`) does not affect interceptor timing. `authService.ts` references `BACKEND_API` only inside async function bodies (e.g., `loginUserFirebase`, `syncUserToBackend`), never at module top level. ES-module live bindings ensure that by the time any of those functions actually runs (well after all modules finish evaluating), `BACKEND_API` is the fully-configured instance.

The cycle does NOT pass through `apiStore.ts` (zero imports — leaf module). Cycle-induced re-evaluation cannot produce two interceptor sets or two barrier instances by this route.

### Q2 — Does the async interceptor actually block?

**Answered: yes. Axios 1.13.6 honors async request interceptors.**

`package.json:29` declares `"axios": "^1.13.6"`; `node_modules/axios/package.json` confirms installed `"version": "1.13.6"`. Async request interceptors have been supported since axios ~0.27 and are fully integrated in 1.x: axios builds a Promise chain that includes interceptors, `dispatchRequest`, and response interceptors via `.then` reductions. An interceptor returning a Promise pauses the chain until that Promise settles. The request itself is not dispatched until the request-interceptor Promise resolves.

`await waitForBootstrap()` inside the interceptor (`api.ts:47`) therefore blocks dispatch. There is no axios-level mechanism in 1.13.6 that would dispatch around a pending interceptor.

### Q3 — Is the barrier promise the same instance across all callers?

**Answered: yes — `apiStore.ts` is a Metro singleton with one `bootstrapPromise`.**

- `apiStore.ts` is a leaf module (zero imports).
- Two import paths to it: `./apiStore` (relative from `api.ts`) and `@/lib/config/apiStore` (alias from `AppContext.tsx`). Both resolve to the same absolute file via Babel `module-resolver` (`babel.config.js`: `alias: { '@': './src' }`).
- Metro keys modules by canonical absolute path. One module instance. One `bootstrapPromise`.
- Grep confirmed: `apiStore.ts` is imported from exactly these two files in the entire `src/` and `app/` tree.

Caveat (dev only): under Metro Fast Refresh, modules can be re-evaluated in piecemeal fashion. A scenario where `apiStore.ts` is re-evaluated alone — producing a fresh unresolved `bootstrapPromise` — while `api.ts`'s already-attached interceptor closure still references the OLD `waitForBootstrap` is theoretically possible. The inverse (interceptor sees new module while AppContext sees old) is also theoretically possible. In a true cold start (R-R reload, full app restart), the singleton invariant holds.

### Q4 — What resolves the barrier, and when?

**Answered: only `setCurrentBaseSiteCode`. On the select-base-site path, it is not called.**

Grep verified: `setCurrentBaseSiteCode` is called at exactly three sites, all in `AppContext.tsx`:

- `AppContext.tsx:127` — bootstrap success branch (after a non-null `getStoredBaseSite()` and before transitioning to `status: 'ready'`).
- `AppContext.tsx:207` — `setBaseSiteForCode` action (fired when the user taps a country in `BaseSiteSelector`).
- (`AppContext.tsx:238` sets only `currentLangCode` — `setLanguageForCode` action — which does not resolve the barrier.)

On the no-stored-base-site branch, `bootstrap()` returns at `AppContext.tsx:117` (`status: 'select-base-site'`) without reaching line 127. There is no AsyncStorage hydration, no rehydration of a Zustand store, no default-value path, and no other site in the codebase that calls `setCurrentBaseSiteCode`. The barrier stays unresolved until the user picks a country.

### Q5 — Why do `config` and `baseSite/details` not reach the backend?

**Answered: not from any JS-layer cause. The interceptor passes them through cleanly. The cause is outside the code under audit.**

All three bootstrap calls plus `app/version/android` carry `_bootstrap: true` (verified in `maintenanceService.tsx:6`, `configurationService.tsx:8`, `baseSitesService.ts:21`, `appVersionConfigService.tsx:12`). The DBG log confirms `flagVal: true` for all four at the moment of interceptor entry. With the flag set, the barrier check is `(true && false) = false` and the await is skipped.

After the barrier check, the request interceptor body has no other `await` that could hang for unauthenticated requests:
- `auth.currentUser` is `null` at cold start (Firebase persistence hasn't restored yet for the very first run). The Authorization branch (`api.ts:55-57`) is skipped.
- Headers are set to null/null and the interceptor returns config.

The response interceptor has only two hanging paths:
- `api.ts:89` `return new Promise(() => {})` (USER_BANNED / EMAIL_BANNED on 403). Requires a 403 from the backend, which requires the request to have reached the backend first — so this cannot explain a request that "never arrived."
- The 401 token-refresh single-flight queue. Requires `auth.currentUser`, which is null. Not applicable.

There is no code path in `api.ts`, `apiStore.ts`, or any of the four bootstrap services that would cause `config` and `baseSite/details` to silently fail to leave the device while `maintenance/active` and `app/version/android` succeed. The discrepancy must be **outside** the JS code path. Candidates: device-level network behavior (TCP backlog, simultaneous-connection limits), the router/edge (`oglasino-router`) dropping or rejecting before logging at the backend, the backend access-log capture being incomplete or filtered, or transient network failure. The brief notes this is intermittent ("on the most recent reload"), which is consistent with environmental causes rather than a deterministic code bug.

### Q6 — The product-search path

**Answered: there is exactly one code path and it goes through `BACKEND_API` (the instance with the interceptor).**

Trace from mount to network:

1. `app/(portal)/(public)/index.tsx:8-17` mounts `<FilteredProductList useFilterStore={usePortalFilterStore} fetchPage={getPortalProducts} applyRandom={true} CardContentComponent={PortalProductCard} />`.
2. `FilteredProductList.tsx:122-134` mounts `<ProductList fetchPage={fetchPageInternal} ... />`.
3. `ProductList.tsx:101-103` — `useEffect(() => { onRefresh(); }, [fetchPage])`. Unconditional. No guard on `selectedLanguage`, no guard on bootstrap state.
4. `onRefresh` (`ProductList.tsx:105-117`) → `loadNextPage(true)` (`ProductList.tsx:71-99`) → `fetchPage(0)` → `fetchPageInternal(0)` (`FilteredProductList.tsx:100-117`) → `fetchPage(filtersData, { page: 0, perPage: 20 })` (the original `getPortalProducts`) → `getPortalProducts` (`productsSearchService.ts:128-133`) → `getProducts(productsFilter, paging, 'portal')` (`productsSearchService.ts:92-126`) → `BACKEND_API.post('/public/product/search', { productsFilter, paging })` (`productsSearchService.ts:105`).

Verified by grep: `BACKEND_API` is the only axios instance in the codebase (`axios.create` appears exactly once, at `api.ts:26`). No raw `fetch` is used for product search. No bypass.

### Q7 — Reconcile the contradiction

**The crux. Restated:**
- The interceptor's DBG log fired for `/public/product/search` with `hasFlag: false flagVal: undefined codeNull: true`.
- Therefore the `if (!getCurrentBaseSiteCode() && !config._bootstrap)` was true and `await waitForBootstrap()` was reached.
- The backend access log shows the request arrived with `X-Base-Site: null` / `X-Lang: null` and returned 400.
- The barrier resolves only via `setCurrentBaseSiteCode` (Q4). On the select-base-site path, `setCurrentBaseSiteCode` is never called this lifetime.

**The static-reading conclusion is unambiguous: with one apiStore module instance and one `bootstrapPromise`, the await should hold indefinitely on the select-base-site path. The request should never reach the backend.** The fact that it does reach the backend means one of the following runtime conditions held:

#### Mechanism A — Metro Fast Refresh stale state (most likely)

`apiStore.ts`'s module-level state is plain `let` declarations:

```ts
let currentBaseSiteCode: string | null = null;
let currentLangCode: string | null = null;
let bootstrapResolve: (() => void) | null = null;
const bootstrapPromise = new Promise<void>((resolve) => { bootstrapResolve = resolve; });
```

Under a true cold start, all four bindings reset. `currentBaseSiteCode === null` ⇔ `setCurrentBaseSiteCode` has not been called ⇔ `bootstrapResolve !== null` ⇔ `bootstrapPromise` is unresolved. These are coupled.

Under Metro Fast Refresh during dev iteration, this coupling can break. The two scenarios:

- **Partial module re-evaluation.** If `api.ts` retains its old interceptor closure (referencing the OLD `apiStore` exports including a resolved `bootstrapPromise`) while a separate hot-edit caused `apiStore.ts` to re-evaluate with fresh state, the interceptor `await`s a resolved promise from a prior lifetime. After the await, `getCurrentBaseSiteCode()` from the NEW apiStore returns `null`. Headers go out null.
- **Cross-bundle observation.** If the user observed the backend log entry from a prior dev iteration that ran a different bundle (e.g., before Brief 14/15 landed), the JS code under audit is innocent and the log entry is from older code. The brief is explicit that the symptom persists after Brief 14/15, so this is less likely but not impossible.

Mechanism A is consistent with the brief's intermittency ("on the most recent reload, the backend received only…") and with the fact that the select-base-site path requires the user to have deliberately cleared their stored base site — a state more naturally reached in dev than in a clean production cold start.

#### Mechanism B — Two distinct `apiStore` module instances

If Metro (or any tooling) produced two module records for `src/lib/config/apiStore.ts` — perhaps via Fast Refresh patch boundaries — `api.ts`'s closure binds to one set of getters/waitForBootstrap, while `AppContext.tsx` binds to the other set of setters. The first instance's `bootstrapPromise` may have been resolved by a prior lifetime's `setCurrentBaseSiteCode` call; the second instance is fresh and untouched.

Static analysis disfavors this — one absolute path, one cached module — but cannot rule it out at runtime without instrumentation.

#### Mechanism C — Interceptor body returned without awaiting

Would require `waitForBootstrap()` to return something that is not a Promise, or for axios to dispatch around the await. Both ruled out by Q2 and by the source of `waitForBootstrap` (always returns the module-scope Promise). Not plausible.

**The verdict that the audit can offer from reading alone: the contradiction is a runtime artifact of partial-state retention across reloads (Mechanism A, with Mechanism B as a secondary possibility), not a defect in the code under audit on a cold-start trajectory.** The code as written holds the barrier correctly on a true cold start with no Fast Refresh.

#### The one named runtime datum that would settle Q7

Add a **single** instrumentation line to `apiStore.ts` `waitForBootstrap` (or just inline at the interceptor's await point):

```ts
console.log('[BARRIER-IDENTITY]', {
  promiseRef: bootstrapPromise === lastSeenPromise ? 'SAME' : 'NEW',
  resolveIsNull: bootstrapResolve === null,
  codeAtAwait: currentBaseSiteCode,
  ts: Date.now(),
});
```

(Keep a module-scope `lastSeenPromise` ref initialized on first call.) And one log in `setCurrentBaseSiteCode`:

```ts
console.log('[SET-BASE-SITE]', { code, ts: Date.now() });
```

Three outcomes:

1. If `[BARRIER-IDENTITY]` for `/public/product/search` shows `resolveIsNull: true` and no `[SET-BASE-SITE]` has logged yet, but the request still reaches the backend, then **axios is not awaiting the interceptor** — Q2 is wrong for this runtime. Investigate axios.
2. If `[BARRIER-IDENTITY]` shows `resolveIsNull: true` (still unresolved) AND the request never reaches the backend in that run, then the brief's prior backend-log entry was from a different bundle/run — Mechanism A's "cross-bundle observation" sub-case.
3. If `[BARRIER-IDENTITY]` shows `resolveIsNull: false` (resolved) but `codeAtAwait: null` and no `[SET-BASE-SITE]` log this lifetime, then **the barrier was resolved by a prior lifetime's state** — Mechanism A confirmed. This is the expected outcome.

A single bundle reload with these two logs in place plus the existing `[DBG]` will collapse the three possibilities to one.

---

## Proposed fix shape (one or two sentences, not the patch)

**Two complementary changes — pick either or both.**

1. **Make the barrier robust to module-level state retention.** Re-couple the barrier and the code: have `waitForBootstrap()` check that `currentBaseSiteCode !== null` AFTER the promise resolves and, if not, re-create the promise and re-await. This eliminates Mechanism A regardless of whether it's Fast-Refresh- or instance-divergence-driven, and adds ~5 lines to `apiStore.ts`.

2. **Component-level guard for the product-search mount effect.** Gate `ProductList.tsx:101-103`'s `onRefresh` effect on `selectedLanguage?.code` (already read at line 49). This eliminates the cold-start race independent of the barrier, restores defense-in-depth, and is a 3-line change. It is what Brief 12 originally proposed as Option A and what Brief 13 named as the fallback.

The minimum patch is (2). It would fix the user-facing symptom regardless of Q7's exact mechanism and re-creates the pre-Φ2 implicit gate at the right scope (per component, not per layout — Φ2 constraints satisfied).
