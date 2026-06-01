# Session summary

**Repo:** oglasino-expo
**Branch:** `new-expo-dev`
**Date:** 2026-05-27
**Task:** Fix `AppContextProvider` effect re-triggers on `state.status`, causing refetch loop on `/api/public/maintenance/active` and `/api/public/app/version/android`.

## Implemented

- Removed `state.status` from the main effect's dependency array at `AppContext.tsx:179` (was line 165 before edits). The effect now runs once on mount (deps: `[bootstrap, setStateWithStatusHook]`), not on every status change. This stops the re-trigger loop where `bootstrap()` changed status → effect re-ran → `bootstrap()` fired again.
- Added `statusRef = useRef(state.status)` at line 60, synced via a dedicated `useEffect` at lines 148-150 (`[state.status]` dep). The polling interval at line 166 now reads `statusRef.current` instead of the stale `state.status` closure, making the maintenance-recovery path actually functional — when the app is in maintenance and the backend recovers, the interval correctly triggers `bootstrap()`.

## Files touched

- `src/components/context/AppContext.tsx` (+12 / -2)

## Tests

- Ran: `npx tsc --noEmit` — exit 0
- Ran: `npm run lint` — 0 errors, 80 warnings (Φ2 baseline)
- Ran: `npm test` — 109 passed, 0 failed
- New tests added: none

## Cleanup performed

None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change (the investigation's drafted `issues.md` entry from the previous session was a draft for Mastermind — this fix session resolves the bug, so the entry should be authored as `fixed` if Mastermind wants to record it)

## Obsoleted by this session

- The stale-closure bug in the polling interval (line 152 before edits) is fixed — `statusRef.current` replaces the captured `state.status`. The recovery path is no longer dead code.
- The double-bootstrap-at-startup behavior is eliminated — the effect runs once, `bootstrap()` runs once, status settles without re-triggering the effect.

## Conventions check

- Part 4 (cleanliness): confirmed — no commented-out code, no unused imports/variables, no debug logging, no TODOs.
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): flagged in "For Mastermind"
- Part 6 (translations): N/A this session
- Other parts touched: none

## Known gaps / TODOs

- Backend log verification pending — Igor needs to observe the app on a real device/simulator and confirm the expected request pattern described in the brief's Verification section. This session cannot self-verify runtime behavior.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): `statusRef` + sync effect (2 lines of hook + 3 lines of effect). Earned because the polling interval runs in a closure that never re-creates after the mount-only effect change. Without the ref, the recovery path (`if status === 'maintenance' then re-bootstrap`) would be permanently dead. A ref is the standard React pattern for reading current state inside stale closures.
  - Considered and rejected: separating `bootstrap()` and the polling interval into two distinct effects. Rejected because the brief explicitly says "do not refactor beyond what's needed for Changes 1 and 2." The single-effect structure works correctly after the fix.
  - Simplified or removed: nothing — the brief forbids refactoring beyond the two changes.

- **Verification step (from brief):** Igor should start the app on a real device or simulator and watch the backend access log. Expected after fix:
  - `/api/public/maintenance/active` — one call at startup, then exactly one every 5 seconds (the polling interval).
  - `/api/public/app/version/android` — one call on initial mount, then one on each foreground return or pathname change. NOT continuous.
  - `/api/public/config` — one call at startup, not repeated.

- **Adjacent observations (Part 4b, carried from investigation, not fixed this session):**
  1. `AppContext.Provider` value at line 247-255 creates a new object on every render because `setBaseSiteForCode`, `setLanguageForCode`, and `getConfiguration` are regular functions (not `useCallback`). This causes every `useAppContext()` consumer to re-render on every AppContextProvider re-render. Structural audit finding F10 / Φ3 scope. Severity: low (performance). Path: `src/components/context/AppContext.tsx:247`. I did not fix this because it is out of scope.
  2. `configurationService.tsx:6` declares return type `Promise<ConfigMap>` but returns `null` on failure (line 15, 17). The `.catch(() => undefined)` wrapper in `bootstrap()` doesn't catch non-thrown `null` returns. The `!configuration` check treats both `null` and `undefined` as failure, which is correct at runtime, but the type contract is misleading. Severity: low. Path: `src/lib/services/configurationService.tsx:6`. I did not fix this because it is out of scope.

- **Lint/tsc/test gates:** all pass. 0 errors, 80 warnings (unchanged from Φ2 baseline), 109 tests passed.
