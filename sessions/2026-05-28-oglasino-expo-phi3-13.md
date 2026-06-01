# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-28
**Task:** Read-only exploration of the right shape for restoring status-gating at the root layout.

## Implemented

- Read-only exploration. No code changes.
- Documented the current (post-Φ2) root layout shape end-to-end.
- Retrieved and documented the pre-Φ2 (committed at HEAD) root layout shape.
- Confirmed the Φ2 constraints from `decisions.md` 2026-05-27.
- Documented the bootstrap flow in `AppContext.tsx` — status state machine, initial value, safe-for-API-calls status.
- Analyzed all six candidates (C1–C5, plus C6 engineer-discovered).
- Produced a recommendation for Mastermind.

## Step 1 — Current root layout shape

Current `app/_layout.tsx` (post-Φ2, uncommitted on `new-expo-dev`):

```tsx
<GestureHandlerRootView className="flex-1">
  <AppContextProvider onStatusChanged={setStatus} onLanguageChanged={setLocale}>
    <AppVersionConfigInit>
      <ToastProvider>
        <ThemeProvider value={NAV_THEME[colorScheme ?? 'light']}>
          <SafeAreaView className="flex-1 bg-background">
            {status === 'ready' && <AppInit />}
            <View className="flex-1">
              <Stack screenOptions={{ headerShown: false }} />    {/* ALWAYS MOUNTED */}
            </View>
            <PortalHost />
            <DialogManager />

            {status !== 'ready' && (
              <View className="absolute inset-0 items-center justify-center bg-background">
                {status === 'maintenance' && <BaseSiteSelector isMaintenance />}
                {status === 'select-base-site' && <BaseSiteSelector isMaintenance={false} />}
                {(!status || status === 'loading') && <LoadingOverlay />}
              </View>
            )}
          </SafeAreaView>
        </ThemeProvider>
      </ToastProvider>
    </AppVersionConfigInit>
  </AppContextProvider>
</GestureHandlerRootView>
```

Key structural facts:

- **Navigator:** `<Stack screenOptions={{ headerShown: false }} />` from `expo-router`. Always mounted. Uses auto-discovery (no explicit `<Stack.Screen>` children). Auto-discovers `(portal)`, `owner`, `+not-found`, `__smoke__`.
- **Provider tree:** `GestureHandlerRootView` → `AppContextProvider` → `AppVersionConfigInit` → `ToastProvider` → `ThemeProvider` → `SafeAreaView`. All OUTSIDE the Stack.
- **AppInit:** Conditionally rendered on `{status === 'ready'}`. Contains auth listener init, ChatsInit, NotificationsInit, ForegroundRevalidationInit, PushNotificationsInit, InitFavoritesStore, CardSizeInit, AccountStateDialogsInit.
- **Overlays:** Non-ready states rendered as absolute-positioned `View` on top of the always-mounted Stack.
- **Local status:** `const [status, setStatus] = useState<AppStateValue['status']>()` — initialized to `undefined`. Set by `AppContextProvider`'s `onStatusChanged` callback.

## Step 2 — Pre-Φ2 root layout shape

Committed at HEAD (retrieved via `git show HEAD:app/_layout.tsx`):

```tsx
{!status && <LoadingOverlay />}
{status === 'maintenance' && <BaseSiteSelector isMaintenance />}
{status === 'select-base-site' && <BaseSiteSelector isMaintenance={false} />}

{(status === 'ready' || status === 'loading') && (
  <>
    <AppInit />
    <View className="flex-1">
      <Slot />
    </View>
    <PortalHost />
    <DialogManager />

    {status === 'loading' && (
      <View className="absolute inset-0 flex-1 items-center justify-center bg-background/80">
        <LoadingOverlay />
      </View>
    )}
  </>
)}
```

Key structural differences from current:

1. **`<Slot />` instead of `<Stack />`**. `Slot` is a pass-through from expo-router — it renders the matched child route without registering a navigator. No native transitions, no stack-based back navigation between root-level routes.
2. **Conditional gate:** `{(status === 'ready' || status === 'loading') && (...)}`. Children do NOT mount when `status` is `undefined` (initial boot), `'maintenance'`, or `'select-base-site'`. They mount when `status` is `'ready'` or `'loading'`.
3. **AppInit inside the gate.** AppInit was rendered alongside Slot inside the same conditional block. In the current layout, AppInit is gated separately on `status === 'ready'`.
4. **Loading overlay inside the gate.** The pre-Φ2 layout showed a semi-transparent overlay during `'loading'` (e.g., base-site switch) ON TOP of the already-rendered children. The current layout shows an opaque overlay when not ready.
5. **Type name:** `OglasinoAppState` (committed) vs `AppStateValue` (current). Same shape.

**Why the pre-Φ2 gate prevented the boot-time API call:**
- Root layout's `status` starts as `undefined` (no initial value in `useState`)
- `onStatusChanged` fires only when AppContext's internal status CHANGES (prev !== next). The initial state (`'loading'` from `defaultState`) does not trigger `onStatusChanged` — it's set via `useState(defaultState)`, not via `setStateWithStatusHook`.
- So root layout's `status` stays `undefined` until bootstrap completes and sets status to `'ready'` (or `'maintenance'` or `'select-base-site'`).
- With `status === undefined`, the gate `(status === 'ready' || status === 'loading')` is `false`. `<Slot />` doesn't render. Child routes don't mount. No effects fire. No API calls.

## Step 3 — Φ2 constraints from decisions.md

### Constraint 1 — Navigators cannot be conditionally rendered

From `decisions.md` 2026-05-27 ("Expo-router `<Stack>` cannot be conditionally rendered"):

> "Conditionally rendering a `<Stack>` causes expo-router to reinitialize the navigation container on every mount, which remounts the parent layout, which resets parent state, which can produce an infinite loop. Bare `<Slot />` doesn't register a navigator, so the conditional-rendering pattern was safe with the pre-Φ2 shape. Switching to `<Stack>` exposed the incompatibility."

And:

> "Any future navigator (`<Stack>` or `<Tabs>`) in this codebase must be always-mounted, not conditionally rendered."

### Constraint 2 — Route-group layout files collapse children

From `decisions.md` 2026-05-25 ("Expo-router route-group layout files collapse children"):

> "A route group with a `_layout.tsx` becomes a single screen of its parent navigator, regardless of how many route files live inside."

Not directly relevant to this fix but noted per brief.

### Constraint 3 — Tab state preservation (the reason for always-mounted Stack)

From `decisions.md` 2026-05-27:

> "The fix: always-mount `<Stack>` and render the non-`ready` states as absolute-positioned overlays on top of it."

The always-mounted root Stack preserves tab state across portal ↔ owner navigation because:
- Navigating from portal to owner PUSHES owner onto the root Stack. Portal's `<Tabs>` stays mounted (in the stack, just not visible).
- Navigating back pops owner. Portal's Tabs is still there with preserved scroll position and tab state.
- With `<Slot />`, navigating from portal to owner SWAPS the child. Portal unmounts. Tabs unmounts. Tab state is lost.

### Additional constraint — `selectedBaseSite` chrome guard

From `features/expo-navigation-foundation.md` §3.2:

> "Because the root Stack is now always mounted (per §3.1), the portal layout renders before `AppContext` bootstrap completes. At that point `selectedBaseSite` is undefined and the chrome above the Tabs (`TopBar`, `CategoryNavigation`, `ConsumerProtectionBanner`) crashes when it reads `selectedBaseSite.iconId` or similar fields. The portal layout therefore guards the chrome rendering on `selectedBaseSite` being defined."

Verified in current `app/(portal)/_layout.tsx:16`: `{selectedBaseSite && (<chrome />)}`.

## Step 4 — Bootstrap flow

### AppContext status state machine

From `src/components/context/AppContext.tsx`:

**States:** `'loading' | 'maintenance' | 'select-base-site' | 'ready'`

**Initial value:** `defaultState = { status: 'loading', baseSites: [] }` (line 48-51). Set via `useState(defaultState)` at line 60. This does NOT trigger `onStatusChanged` — the callback only fires inside `setStateWithStatusHook` when `prev.status !== next.status`.

**Transitions:**

```
'loading' ──(maintenance=true || !config)──→ 'maintenance'
'loading' ──(!storedBaseSite)──→ 'select-base-site'
'loading' ──(success)──→ 'ready'
'ready' ──(base-site switch)──→ 'loading' ──→ 'ready'
'ready' ──(language switch)──→ 'loading' ──→ 'ready'
'ready' ──(polling detects maintenance)──→ 'maintenance'
'maintenance' ──(polling: maintenance clears)──→ reBootstrap ──→ 'loading' ──→ ...
any ──(bootstrap throws)──→ 'maintenance'
```

**Root layout's local `status`:** Starts as `undefined`. Receives updates via `onStatusChanged` callback, which fires inside `setStateWithStatusHook` via `setTimeout(() => onStatusChanged?.(next.status), 0)`. The `setTimeout` defers the callback to after the React state update commits.

**When headers become valid:**

`setCurrentBaseSiteCode(storedBaseSite.code)` and `setCurrentLangCode(language.code)` are called at `AppContext.tsx:127-128`, BEFORE setting status to `'ready'` at line 133. So by the time `status === 'ready'`, the apiStore headers are guaranteed valid.

During base-site and language switches, the apiStore values are updated before status returns to `'ready'` (e.g., `setBaseSiteForCode` sets codes at lines 207-208 before calling `setStateWithStatusHook` to set `'ready'` at line 214).

**Safe status for API calls:** `'ready'`. The `'loading'` status is safe for SUBSEQUENT loads (base-site/language switch) because apiStore values are already set from the previous `'ready'` state. But the INITIAL `'loading'` (from `defaultState`) has null headers.

### Bootstrap calls and the shared axios instance

All three bootstrap services use `BACKEND_API` — the same axios instance as product search:
- `checkIfMaintenance()` → `BACKEND_API.get('/public/maintenance/active')` (`maintenanceService.tsx:6`)
- `getAppConfiguration()` → `BACKEND_API.get('/public/config')` (`configurationService.tsx:8`)
- `fetchBaseSites()` → `BACKEND_API.get('/public/baseSite/details')` (`baseSitesService.ts:21`)

The request interceptor at `api.ts:35-36` sets `X-Base-Site` and `X-Lang` from apiStore for ALL requests, including these bootstrap calls. The backend accepts `null` for these public endpoints (they don't require the headers). The 400 comes specifically from product search, which validates the headers.

## Step 5 — Candidate analysis

### C1 — Stack with children gated

**Verdict: NOT APPLICABLE.**

The root `<Stack>` uses auto-discovery — no explicit `<Stack.Screen>` children are rendered. In expo-router, `<Stack.Screen>` children are configuration declarations (setting `headerShown`, `title`, etc.), not content gates. Routes exist by virtue of the file system, not by being declared as `<Stack.Screen>` children. Adding or removing `<Stack.Screen>` declarations does not prevent a route's component from mounting when that route is the active route.

Even if we added explicit `<Stack.Screen>` entries and conditionally rendered them, the underlying file-system route would still exist and render when matched by expo-router's auto-discovery.

**JSX sketch:** N/A
**Φ2 constraint compatibility:** N/A
**Tab state risk:** N/A
**Expo-router crash risk:** N/A

### C2 — Overlay above always-mounted navigator

**Verdict: CONFIRMED DEAD END.**

An overlay (visual blocking layer) does not prevent React from rendering the components underneath it. The Stack is always mounted, so child routes mount, effects fire, and API calls go out — regardless of any visual overlay. The current layout already has an overlay for non-ready states; the 400 error happens DESPITE the overlay being visible.

The problem is mount-phase effects, not visual rendering.

**JSX sketch:** Already implemented (the current `{status !== 'ready' && <absolute overlay />}` pattern).
**Φ2 constraint compatibility:** Compatible (Stack stays always-mounted).
**Tab state risk:** None.
**Expo-router crash risk:** None.
**Why dead end:** Does not solve the problem.

### C3 — Slot-pattern at root, Stack inside

**Verdict: CAUSES TAB STATE REGRESSION. Not recommended.**

Restoring `{(status === 'ready' || status === 'loading') && <Slot />}` at the root would prevent children from mounting before bootstrap. The child layouts (portal's `<Tabs>`, owner's `<Stack>`) would be the navigators.

**Does the conditional `<Slot />` itself violate Φ2's constraint?** No. `<Slot />` is not a navigator — it doesn't register with the navigation container. Conditionally rendering `<Slot />` was the pre-Φ2 pattern and worked correctly. The constraint is specifically about `<Stack>` and `<Tabs>`.

**But conditionally rendering `<Slot />` INDIRECTLY conditionally renders child navigators.** When `<Slot />` unmounts, the portal's `<Tabs>` and owner's `<Stack>` unmount. When `<Slot />` mounts, they mount fresh. Whether this triggers the same expo-router navigation container reinitialization loop is uncertain — the Φ2 discovery was with a ROOT-LEVEL Stack, not a child-level navigator being indirectly gated. It might work. But it's untested.

**The definitive problem: tab state loss.** `<Slot />` renders one child at a time. When the user navigates from portal to owner, portal unmounts (including its Tabs). When the user navigates back, portal remounts fresh — tab scroll position, fetched data, and screen state are lost. This is a direct regression from the always-mounted root Stack pattern.

The root Stack preserves portal state during portal ↔ owner navigation by keeping both routes in the stack. Slot cannot do this.

```
Scenario: user scrolls home feed → opens dashboard → goes back
  Root Stack: portal stays mounted, scroll position preserved ✓
  Root Slot: portal unmounts, remounts from scratch ✗
```

**JSX sketch:**
```tsx
<SafeAreaView className="flex-1 bg-background">
  {(status === 'ready' || status === 'loading') && (
    <>
      <AppInit />
      <View className="flex-1">
        <Slot />   {/* pass-through, child layouts provide navigators */}
      </View>
      <PortalHost />
      <DialogManager />
    </>
  )}
  {/* overlays for non-ready states */}
</SafeAreaView>
```

**Φ2 constraint compatibility:** Slot itself doesn't violate the constraint. Indirect navigator mount/unmount is uncertain.
**Tab state risk:** HIGH — tab state lost on portal ↔ owner navigation.
**Expo-router crash risk:** Low for Slot; uncertain for indirect child navigator cycling.

### C4 — Placeholder loading route

**Verdict: PARTIALLY VIABLE but hacky. Deep links bypass the gate.**

Keep Stack always mounted. Add a `/loading` route. Set `initialRouteName="loading"`. When bootstrap completes, `router.replace('/(portal)')`.

**Cold start without deep link:** Works. The loading screen renders first, portal doesn't mount, no pre-bootstrap API calls.

**Deep link at cold start:** Does NOT work. If the user taps a notification that deep-links to `/(portal)/(public)/product/123`, expo-router renders that route directly — the loading screen is bypassed. The product screen mounts before bootstrap. If it fires an API call, it hits the same 400.

**URL/navigation impact:** The loading route would appear in the navigation state. Using `router.replace` would clear it, but there could be a flash or a brief visible transition. The loading route's file (`app/loading.tsx`) would exist in the file system, adding a route that has no user-facing purpose after boot.

**JSX sketch:**
```tsx
// app/_layout.tsx — unchanged (Stack always mounted)
<Stack screenOptions={{ headerShown: false }} initialRouteName="loading" />

// app/loading.tsx
export default function LoadingScreen() {
  const status = /* read from root or context */;
  useEffect(() => {
    if (status === 'ready') {
      router.replace('/(portal)');
    }
  }, [status]);
  return <LoadingOverlay />;
}
```

**Φ2 constraint compatibility:** Compatible (Stack stays always-mounted).
**Tab state risk:** Low (Tabs mounts normally once navigated to portal).
**Expo-router crash risk:** Low.
**Why not recommended:** Deep links bypass the gate. The real-world scenario of a notification tap cold-starting the app directly into a secured/product route would hit the same 400. Not truly architectural.

### C5 — Gate at AppContextProvider's children prop

**Verdict: CONFIRMED DEAD END.**

This conditionally renders the `<Stack>` based on `status` — exactly the pattern that caused the infinite loop in Φ2 smoke. `AppContextProvider` wraps `<Stack>`, so gating `<Stack>` from within the provider is structurally identical to gating at the root layout.

**JSX sketch:**
```tsx
// Inside AppContextProvider
return (
  <AppStateContext.Provider value={stateValue}>
    <AppActionsContext.Provider value={actionsValue}>
      {state.status === 'ready' ? children : <LoadingOverlay />}
    </AppActionsContext.Provider>
  </AppStateContext.Provider>
);
```

**Φ2 constraint compatibility:** VIOLATES. Children include `<Stack>`. Conditionally rendering → navigator mount/unmount → infinite loop.
**Tab state risk:** Moot (crashes first).
**Expo-router crash risk:** HIGH — infinite loop per Φ2 discovery.

### C6 — Engineer-discovered: API-layer request barrier

**Verdict: VIABLE. The most architecturally sound approach that doesn't touch the navigator tree.**

Keep the always-mounted `<Stack>` unchanged. Add a Promise-based barrier in the axios request interceptor that holds non-bootstrap requests until `currentBaseSiteCode` is set.

**How it works:**

1. `apiStore.ts` adds a `baseSiteReadyPromise` that resolves when `setCurrentBaseSiteCode` is first called.
2. The request interceptor at `api.ts:32-52` checks `getCurrentBaseSiteCode()` before setting headers. If null AND the request URL is not a bootstrap endpoint, the interceptor awaits the `baseSiteReadyPromise`.
3. When bootstrap calls `setCurrentBaseSiteCode` at `AppContext.tsx:127`, the Promise resolves.
4. Queued requests proceed with valid headers.

**Bootstrap deadlock prevention:** All three bootstrap services use `BACKEND_API`. The interceptor must whitelist their URLs to avoid deadlock:
- `/public/maintenance/active` (`maintenanceService.tsx:6`)
- `/public/config` (`configurationService.tsx:8`)
- `/public/baseSite/details` (`baseSitesService.ts:21`)

Alternative: pass a `_bootstrap: true` flag in the axios config for bootstrap calls, and check that flag in the interceptor.

**Concrete JSX/code sketch:**

```typescript
// apiStore.ts additions (6 lines)
let baseSiteReadyResolve: (() => void) | null = null;
const baseSiteReadyPromise = new Promise<void>((r) => { baseSiteReadyResolve = r; });

export function setCurrentBaseSiteCode(code: string) {
  currentBaseSiteCode = code;
  baseSiteReadyResolve?.();
  baseSiteReadyResolve = null;
}
export function getBaseSiteReadyPromise() { return baseSiteReadyResolve ? baseSiteReadyPromise : null; }

// api.ts request interceptor change (~5 lines added)
instance.interceptors.request.use(async (config) => {
  if (!getCurrentBaseSiteCode() && !isBootstrapUrl(config.url)) {
    const p = getBaseSiteReadyPromise();
    if (p) await p;
  }
  config.headers.set('X-Base-Site', getCurrentBaseSiteCode());
  config.headers.set('X-Lang', getCurrentLangCode());
  // ... rest unchanged
});

function isBootstrapUrl(url: string | undefined): boolean {
  return ['/public/maintenance', '/public/config', '/public/baseSite'].some(
    (p) => url?.includes(p)
  );
}
```

**Φ2 constraint compatibility:** Fully compatible. Stack stays always-mounted. No navigator changes.
**Tab state risk:** NONE. Navigator tree is unchanged.
**Expo-router crash risk:** NONE. No navigator lifecycle changes.
**Risk profile:**
- Bootstrap deadlock if the URL whitelist misses a bootstrap endpoint. Mitigable with a timeout on the Promise (e.g., reject after 15 seconds with a clear error).
- All pre-bootstrap requests queue and fire simultaneously when bootstrap completes. For cold start, this is typically just the home screen's product search — one request. Burst risk is low.
- Future bootstrap endpoints added without updating the whitelist would deadlock. Document the pattern in the code.

**Scope:** ~15-20 lines across `apiStore.ts` and `api.ts`. Potentially a helper file for `isBootstrapUrl`. No layout files touched. No test changes expected (existing tests don't exercise the cold-start timing).

## Step 6 — Recommendation

### Primary recommendation: C6 (API-layer request barrier)

**Why C6 over others:**

1. **Does not touch the navigator tree.** No risk to tab state preservation. No risk of expo-router crashes. The Φ2 always-mounted Stack pattern is fully preserved.
2. **Architectural.** Protects ALL API calls at the single point where headers are attached. Future components that fire API calls on mount are automatically protected — no per-component gates needed.
3. **Small scope.** ~15-20 lines across 2 files. No layout files, no component files, no test changes.
4. **Composable with future work.** When Φ4 (service layer + error contract) lands, the API-layer barrier is a natural fit. If the error contract work introduces a more sophisticated request lifecycle, the barrier integrates cleanly.

**What could break:**
- A new bootstrap endpoint added without updating `isBootstrapUrl` → deadlock at boot. **Mitigation:** add a 15-second timeout on the Promise. If the timeout fires, reject queued requests with a clear error rather than hanging forever. Document the whitelist.
- If `setCurrentBaseSiteCode` is never called (e.g., bootstrap fails and status goes to `'maintenance'`) → queued requests hang until the timeout. **Mitigation:** also resolve the barrier on `'maintenance'` status, allowing requests to proceed (they'll fail with null headers, which is the existing behavior for maintenance mode).

**Verification needed:**
- Confirm that axios supports async request interceptors in the React Native environment (it does per axios docs — interceptors can return Promises).
- Confirm that the three bootstrap URLs are the ONLY pre-baseSiteCode API calls.
- Run the app on device and verify: (a) product search loads correctly after bootstrap, (b) bootstrap itself is not delayed, (c) no console errors from queued requests.

### Fallback recommendation: Option A (per-component gate)

If Mastermind judges C6 as too complex or risky for a Φ3 closeout patch:

```tsx
// ProductList.tsx — 2-line change
useEffect(() => {
  if (!selectedLanguage) return;    // ← add
  onRefresh();
}, [fetchPage, selectedLanguage]);   // ← add selectedLanguage to deps
```

This solves the immediate symptom (home screen 400 at boot) with zero risk. But it does NOT protect future components that fire API calls on mount — each would need its own gate. Igor previously stated he wants the architectural fix, so this is the fallback if C6 is rejected.

### If both are rejected

If neither C6 nor Option A is acceptable for Φ3, the architectural fix moves to a dedicated foundation chat per the brief's out-of-scope section. The product search 400 persists until that chat ships.

## Files touched

None. Read-only session.

## Tests

Not run (no code changes). Per brief: "not required to be re-run."

## Cleanup performed

None needed. No code changes.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

Nothing. Read-only exploration.

## Conventions check

- Part 4 (cleanliness): N/A — no code changes
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): one observation flagged in "For Mastermind"
- Part 6 (translations): N/A this session
- Other parts touched: none

## Known gaps / TODOs

- The C6 API-layer approach has not been tested against expo-router's async interceptor behavior at runtime. The patch session (Brief 14) should include a device verification step.
- The bootstrap URL whitelist in C6 is fragile — it must be updated if new bootstrap endpoints are added. A timeout-based safety net is recommended.

## For Mastermind

### Part 4a simplicity evidence (required)

- **Added (earned complexity):** nothing (read-only session).
- **Considered and rejected:**
  - C3 (root `<Slot />` gate) — would restore the pre-Φ2 gate but regresses tab state on portal ↔ owner navigation. The always-mounted root Stack is load-bearing for tab state preservation; `<Slot />` can't replace it.
  - C4 (placeholder loading route) — achievable but hacky and deep links bypass it. Not truly architectural.
  - A second axios instance for bootstrap calls (to avoid the whitelist in C6) — adds more complexity than the whitelist. The whitelist is three stable URLs.
- **Simplified or removed:** nothing (read-only session).

### Part 4b adjacent observations

1. **`configurationService.tsx` return type — `Promise<ConfigMap>` but returns `null` on failure.** Already tracked in `issues.md` 2026-05-27. No action needed here.

### Analysis — why no layout-level approach works

The fundamental tension:

- **Φ2 constraint:** `<Stack>` and `<Tabs>` must be always-mounted. No conditional rendering of navigators.
- **Pre-Φ2 gate:** Worked because `<Slot />` is not a navigator. Conditionally rendering `<Slot />` doesn't trigger expo-router's navigation container reinitialization.
- **Post-Φ2 architecture:** Root `<Stack>` provides (a) native transitions between portal and owner, (b) tab state preservation across portal ↔ owner navigation by keeping both routes in the stack.

Going back to `<Slot />` at root (C3) solves the gate problem but loses (a) and (b). Keeping `<Stack>` at root (current) preserves (a) and (b) but loses the gate. No layout-level approach provides both.

The API-layer approach (C6) sidesteps the tension entirely by moving the gate below the React tree into the HTTP layer. It doesn't care what navigator is rendered — it gates at the point where headers are attached to requests.

### Recommendation summary for Mastermind

**C6 (API-layer request barrier) is the recommended approach for Brief 14.** It's the only approach that:
- Preserves the always-mounted Stack (Φ2 constraint)
- Preserves tab state (Φ2 rationale)
- Prevents ALL pre-bootstrap API calls (architectural, not per-component)
- Has a small scope (~15-20 lines, 2 files)

If Mastermind judges C6 as out of scope for Φ3, fall back to Option A (per-component gate in `ProductList.tsx`, 2 lines) and move the architectural fix to a dedicated chat.
