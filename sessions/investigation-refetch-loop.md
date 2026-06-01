# Investigation — refetch loop on `/api/public/app/version/android` and `/api/public/maintenance/active`

**Repo:** `oglasino-expo`
**Branch:** `new-expo-dev`
**Date:** 2026-05-27
**Task:** Read-only investigation of the ~3 req/s refetch loop on two backend endpoints after the `<Slot />` → `<Stack>` change in Φ2.
**Mode:** read-only — no edits

---

## 1. Components responsible for each endpoint

### `/api/public/maintenance/active`

**Service:** `src/lib/services/maintenanceService.tsx:4-18` — `checkIfMaintenance()` calls `BACKEND_API.get<boolean>('/public/maintenance/active')`.

**Consumer:** `src/components/context/AppContext.tsx` — called in TWO places:

1. **`bootstrap()` at line 82** — inside `Promise.all([checkIfMaintenance().catch(() => true), ...])`. Called every time the main effect fires.
2. **Polling interval at line 145** — `const maintenance = await checkIfMaintenance()` inside a `setInterval(..., 5000)`.

No other consumers exist in the codebase (confirmed via grep).

### `/api/public/app/version/android`

**Service:** `src/lib/services/appVersionConfigService.tsx:6-22` — `getAppVersionConfig()` calls `BACKEND_API.get(/public/app/version/${Platform.OS}?currentVersion=...)`.

**Consumer:** `src/components/internals/AppVersionConfigInit.tsx` — called via `checkVersion()` in TWO effects:

1. **Mount effect at line 103-114** — `checkVersion(true)` on mount, plus an `AppState` listener that calls `checkVersion(false)` on foreground return.
2. **Pathname effect at line 116-118** — `checkVersion(false)` whenever `pathname` (from `usePathname()`) changes.

No other consumers exist in the codebase (confirmed via grep).

---

## 2. Current mount location in the tree

```
RootLayout (app/_layout.tsx:25)
└─ GestureHandlerRootView (:56)
   └─ AppContextProvider (:57)              ← owns bootstrap() + maintenance polling
      └─ AppVersionConfigInit (:58)         ← owns version checks, uses usePathname()
         └─ ToastProvider (:59)
            └─ ThemeProvider (:60)
               └─ SafeAreaView (:61)
                  ├─ {!status && <LoadingOverlay />}
                  ├─ {status === 'maintenance' && <BaseSiteSelector />}
                  ├─ {status === 'select-base-site' && <BaseSiteSelector />}
                  └─ {(status === 'ready' || status === 'loading') && (
                        <AppInit />
                        <View>
                          <Stack />            ← the Φ2 change (was <Slot />)
                        </View>
                        <PortalHost />
                        <DialogManager />
                     )}
```

This matches the Φ2 spec §3.1 contract: "Provider stack outside the Stack remains identical… `AppInit`, `PortalHost`, `DialogManager` remain as siblings."

**Key structural fact:** `AppVersionConfigInit` (which calls `usePathname()`) is ABOVE the `<Stack>` in the tree. It wraps the entire conditional block including the Stack. The Stack is a child of `AppVersionConfigInit`'s rendered children.

---

## 3. Effect dependency arrays and what triggers them

### AppContext.tsx — main effect (lines 140-165)

```tsx
useEffect(() => {
    bootstrap();                              // fires on EVERY effect run
    const interval = setInterval(async () => {
        const maintenance = await checkIfMaintenance();
        if (maintenance) {
            setStateWithStatusHook(prev => prev.status !== 'maintenance'
                ? { ...prev, status: 'maintenance' } : prev);
        } else {
            if (state.status === 'maintenance') {  // BUG: stale closure
                await bootstrap();
            }
        }
    }, 5000);
    return () => clearInterval(interval);
}, [bootstrap, state.status, setStateWithStatusHook]);   // <-- state.status is here
```

**Deps:**
- `bootstrap` — `useCallback` with dep `[setStateWithStatusHook]`. Stable (referentially same across renders).
- `state.status` — **changes when `bootstrap()` calls `setStateWithStatusHook()`**. This is the re-trigger source.
- `setStateWithStatusHook` — `useCallback` with dep `[onStatusChanged]`. Stable (`onStatusChanged` is `setStatus` from `useState`, referentially stable).

**What triggers it:** Any change to `state.status`. Since `bootstrap()` sets `state.status` on every call, the effect re-runs after every `bootstrap()` completion where status actually changes value.

### AppContext.tsx — `bootstrap()` (lines 76-136)

```tsx
const bootstrap = useCallback(async () => {
    if (booting.current) return;
    booting.current = true;
    try {
        const [maintenance, configuration, baseSites] = await Promise.all([
            checkIfMaintenance().catch(() => true),
            getAppConfiguration().catch(() => undefined),
            fetchBaseSites().catch(() => []),
        ]);
        if (maintenance || !configuration) {         // <-- !null === true
            setStateWithStatusHook({ status: 'maintenance', ... });
            return;
        }
        // ... storedBaseSite checks ...
        setStateWithStatusHook({ status: 'ready', ... });
    } catch {
        setStateWithStatusHook(prev => ({ ...prev, status: 'maintenance' }));
    } finally {
        booting.current = false;
    }
}, [setStateWithStatusHook]);
```

`bootstrap()` changes `state.status` on every call. After the async `await`, `booting.current` is set to `false` in `finally`, allowing the next `bootstrap()` call (from the re-triggered effect) to enter.

### AppVersionConfigInit.tsx — pathname effect (lines 116-118)

```tsx
useEffect(() => {
    checkVersion(false);
}, [pathname]);   // <-- changes when navigation state changes
```

**What triggers it:** `pathname` from `usePathname()` changing. With `<Stack>`, pathname changes when the navigator initializes (mount) and when navigation occurs. With `<Slot />`, pathname was simpler and more stable — no navigator to manage.

### AppVersionConfigInit.tsx — mount effect (lines 103-114)

```tsx
useEffect(() => {
    checkVersion(true);
    const subscription = AppState.addEventListener('change', ...);
    return () => subscription.remove();
}, []);   // <-- runs once on mount
```

Runs exactly once. Not part of the loop.

---

## 4. Root cause analysis

**The loop is caused by `state.status` in the dependency array of the `AppContextProvider` main effect (`AppContext.tsx:165`), combined with a conditional `<Stack>` that remounts on status changes.**

### The mechanism

The effect at `AppContext.tsx:140` unconditionally calls `bootstrap()` whenever any dep changes. `bootstrap()` changes `state.status`. Since `state.status` IS a dependency of the effect, the effect re-runs, calling `bootstrap()` again. This is a state-update-in-effect-deps cycle.

**Guaranteed double-bootstrap at startup:**

1. `state.status = 'loading'` (initial). Effect runs → `bootstrap()` fires → maintenance request #1.
2. ~300ms later: `bootstrap()` resolves. Sets `state.status = 'ready'`.
3. Effect re-runs (`state.status` changed: `'loading'` → `'ready'`). `bootstrap()` fires → maintenance request #2.
4. ~300ms later: `bootstrap()` resolves. Sets `state.status = 'ready'` again. `prev.status === next.status` — no change. Effect does NOT re-run. Loop breaks.

This alone produces 2 bootstrap calls (2 maintenance + 2 config + 2 baseSites = 6 requests), plus version checks from pathname changes. After that, only the 5-second polling interval continues.

**But the brief describes a sustained ~3 req/s, not just a startup burst. For that, `state.status` must oscillate.**

### The oscillation hypothesis (most likely cause of sustained loop)

`bootstrap()` calls three services in `Promise.all`:
- `checkIfMaintenance().catch(() => true)` — brief confirms 200
- `getAppConfiguration().catch(() => undefined)` — **not confirmed by the brief**
- `fetchBaseSites().catch(() => [])` — **not confirmed by the brief**

The critical check at `AppContext.tsx:87`:
```tsx
if (maintenance || !configuration) {
    setStateWithStatusHook({ status: 'maintenance', ... });
    return;
}
```

`getAppConfiguration()` (`src/lib/services/configurationService.tsx:6-19`) returns `null` on non-200 responses. Since it doesn't throw on non-200, the `.catch(() => undefined)` wrapper in `bootstrap()` does NOT trigger — `configuration` receives `null`. And `!null === true`, so status goes to `'maintenance'`.

**If `/api/public/config` is intermittently failing (returning non-200, timing out, or throwing), the oscillation is:**

1. `bootstrap()` → config fails → `configuration = null` → `!null === true` → status = `'maintenance'`
2. Effect re-runs (status changed). `bootstrap()` → config succeeds → status = `'ready'`
3. Effect re-runs (status changed). `bootstrap()` → config fails → status = `'maintenance'`
4. Repeat at ~300-400ms per cycle (network round-trip time)

Each cycle fires one `checkIfMaintenance()` request (maintenance endpoint hit). Each status change from maintenance→ready or ready→maintenance also causes `RootLayout.status` to change (via the `onStatusChanged` setTimeout callback at `AppContext.tsx:63`), which toggles the conditional block at `app/_layout.tsx:66`. This unmounts/remounts the `<Stack>`. Each `<Stack>` mount triggers navigation state initialization, changing `pathname` in `AppVersionConfigInit`, which fires `checkVersion(false)` (version endpoint hit).

**Result:** ~1 maintenance + ~1 version per cycle = ~2 requests per ~350ms = ~5-6/s to the two monitored endpoints. With network jitter and the `checkingRef.current` guard on version checks (which drops some calls), the observed rate of ~3/s is consistent.

### Why `<Slot />` masked the problem and `<Stack>` exposed it

With `<Slot />`:
- `<Slot />` renders the matched route directly without a navigator. It doesn't create navigation state events on mount.
- The double-bootstrap still happened (the effect bug exists regardless of the navigator type).
- But `usePathname()` in `AppVersionConfigInit` didn't see pathname changes from `<Slot />` mounting — there was no navigator to trigger navigation events.
- The version endpoint was called on mount (once) and on foreground return. Not on every status oscillation.
- The extra maintenance requests from the double-bootstrap were silent (no visible version endpoint hits).

With `<Stack>`:
- `<Stack>` creates a `NativeStackNavigator`. Mounting it initializes navigation state: resolving the initial route, pushing it onto the stack, and updating the global navigation state.
- This navigation state update changes `pathname` (from `usePathname()` in `AppVersionConfigInit`).
- If `<Stack>` remounts on every status oscillation (because the conditional block toggles), pathname changes on every cycle → version check fires on every cycle.
- Both endpoints are now hit in the loop, making the problem visible in backend logs.

### The stale closure bug (secondary, not the loop cause)

`AppContext.tsx:152`: `if (state.status === 'maintenance')` inside the `setInterval` callback reads the stale closure value of `state.status` from when the effect last ran. After the effect re-runs with `state.status = 'ready'`, the old interval is cleared and a new one is created. The new interval's closure captures `'ready'`, so the `if` check never matches. This bug means the interval can never trigger a recovery `bootstrap()` after the status settles — the recovery path at line 153 is effectively dead code.

---

## 5. Summary statement

**The most likely cause is:** the `state.status` dependency in the `AppContextProvider` main effect (`AppContext.tsx:165`), which creates a re-trigger loop: `bootstrap()` changes status → effect re-runs → `bootstrap()` fires again. At minimum this produces a double-bootstrap at startup. If `/api/public/config` (called by `getAppConfiguration()` inside `bootstrap()`) is intermittently failing, `state.status` oscillates between `'ready'` and `'maintenance'`, sustaining the loop at network-round-trip frequency. The `<Slot />` → `<Stack>` change amplified this from a silent double-bootstrap into a visible loop hitting BOTH endpoints, because each `<Stack>` mount/unmount (triggered by the status oscillation) fires navigation events that trigger the version check via `usePathname()`.

**Alternates, if `/api/public/config` is not failing:**

- **A. React development mode (Strict Mode) interaction.** Strict Mode double-mounts components and re-runs effects. Combined with the async `bootstrap()` and the `booting.current` guard, Strict Mode could produce a third bootstrap call if timing allows. This would explain a larger startup burst but not a sustained loop.

- **B. Expo-router `usePathname()` instability outside a navigator.** `AppVersionConfigInit` calls `usePathname()` but sits ABOVE the `<Stack>`. If `usePathname()` returns unstable values when called outside a navigator (e.g., before the Stack mounts or between Stack unmount/remount cycles), the pathname effect could fire repeatedly. This is an expo-router-specific behavior that cannot be verified from code reading alone.

- **C. The `getCurrentLocale: () => ...` new-function-on-every-bootstrap subtle issue.** Each `bootstrap()` call creates a new `getCurrentLocale` arrow function in the state object. While this causes a re-render (new state object), it does NOT change `state.status` (the string value comparison is stable). So this causes unnecessary re-renders but does not trigger the effect re-run.

---

## Files touched

None (read-only investigation).

## Tests

None (read-only investigation).

## Cleanup performed

None needed (read-only).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: the fix for this bug should be tracked there once the fix brief is authored. **Draft for Mastermind** — a new `issues.md` entry:

> ## 2026-05-27 — `AppContextProvider` effect re-triggers on every `bootstrap()` completion, causing refetch loop
>
> **Severity:** medium
> **Status:** open
> **Found in:** `oglasino-expo/src/components/context/AppContext.tsx:140-165` (the main effect's `state.status` dependency).
> **Detail:** The main `useEffect` has `state.status` in its dependency array. `bootstrap()` — called unconditionally at the top of the effect — changes `state.status` on every call. This creates a guaranteed double-bootstrap at startup and, if any of the three `bootstrap()` services (`checkIfMaintenance`, `getAppConfiguration`, `fetchBaseSites`) fails intermittently, a sustained oscillation loop at ~300-400ms per cycle. The `<Slot />` → `<Stack>` change in Φ2 amplified this into a visible refetch loop because each `<Stack>` mount/unmount (triggered by status changes propagating to `RootLayout.status`) fires navigation events that trigger version checks via `usePathname()`. Secondary bug: the polling interval at line 143 captures a stale `state.status` closure, making the recovery path at line 152-154 dead code. Investigation at `.agent/investigation-refetch-loop.md`.

## Obsoleted by this session

Nothing (read-only investigation).

## Conventions check

- Part 4 (cleanliness): N/A — no code changes
- Part 4a (simplicity): N/A — no code changes
- Part 4b (adjacent observations): flagged in "For Mastermind" below
- Other parts: N/A this session

## Known gaps / TODOs

- Cannot confirm whether `/api/public/config` is actually failing intermittently. The brief only confirms the two observed endpoints (`maintenance/active` and `app/version/android`) return 200. Verifying the config endpoint's behavior would confirm or rule out the oscillation hypothesis.
- Cannot verify expo-router `usePathname()` behavior outside a navigator at runtime — this requires running the app with debug logging.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only)
  - Considered and rejected: nothing (read-only)
  - Simplified or removed: nothing (read-only)

- **Root cause confirmed from code reading.** The `state.status` dependency in the AppContext effect is the structural bug. The fix brief should:
  1. Remove `state.status` from the effect's dependency array (the effect should run once on mount, not on every status change). Use a ref for status if the interval needs the current value.
  2. Replace the stale `state.status` closure at line 152 with a ref that tracks the latest status, so the recovery path works correctly.
  3. Consider separating the one-time `bootstrap()` call from the polling interval into distinct effects.

- **Adjacent observation (Part 4b):** The `AppContext.Provider` value at line 235-243 creates a new object on every render because `setBaseSiteForCode`, `setLanguageForCode`, and `getConfiguration` are regular functions (not `useCallback`). This causes every `useAppContext()` consumer to re-render on every AppContextProvider re-render. Low severity (performance, not correctness). This is structural audit finding F10 / Φ3 scope. I did not fix this because it is out of scope.

- **Adjacent observation (Part 4b):** `configurationService.tsx:6` declares return type `Promise<ConfigMap>` but returns `null` on failure (line 15, 17). TypeScript sees this as `null`, but the `.catch(() => undefined)` wrapper in `bootstrap()` doesn't catch non-thrown `null` returns. The `!configuration` check at `AppContext.tsx:87` treats both `null` and `undefined` as failure, which is correct at runtime, but the type contract is misleading. Low severity. I did not fix this because it is out of scope.

- **Verification step for Igor:** check whether `/api/public/config` returns 200 consistently when the app launches. If it sometimes returns non-200, that confirms the oscillation hypothesis. If it's always 200, the sustained loop requires further investigation at runtime (e.g., adding a `console.log` in the effect to log `state.status` on each run).
