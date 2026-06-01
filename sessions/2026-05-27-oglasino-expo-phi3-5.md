# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-27
**Task:** Split the single `AppContext` into two contexts (`AppStateContext` + `AppActionsContext`), migrate all 26 consumers to the appropriate hook, delete `useAppContext`.

## Implemented

- Split `AppContext` into `AppStateContext` (state fields) and `AppActionsContext` (action functions). Two new hooks: `useAppState()` and `useAppActions()`. Old `useAppContext` deleted.
- Wrapped `setBaseSiteForCode`, `setLanguageForCode`, and `getConfiguration` in `useCallback` with stable deps. Uses `stateRef` pattern so callbacks read current state without re-creating on every render. `reBootstrap` (aliased from `bootstrap`) was already `useCallback`-wrapped.
- Fixed a pre-existing stale-closure bug in `setBaseSiteForCode`: the final `setStateWithStatusHook` call used `...state` (captured at definition time) instead of the functional updater `(prev) => ({ ...prev, ... })`. Now uses the updater form, matching the pattern used elsewhere in the file.
- Both context values are `useMemo`-stabilized. `actionsValue` is stable for the lifetime of the provider (all four actions are stable `useCallback` refs). `stateValue` only changes when state changes (rare: bootstrap, base-site switch, language switch, maintenance flip).
- Migrated all 26 consumers: 20 state-only → `useAppState()`; 1 actions-only (`BasicInfoProductDialog`) → `useAppActions()`; 5 mixed → both hooks.
- Renamed `OglasinoAppState` → `AppStateValue` (new exported type). Updated `app/_layout.tsx` import.

## Consumer migration table

| File | Hook(s) | Fields read |
|---|---|---|
| `app/(portal)/_layout.tsx` | `useAppState` | `selectedBaseSite` |
| `app/(portal)/(public)/catalog/[...categories].tsx` | `useAppState` | `selectedBaseSite` |
| `app/(portal)/(public)/product/[...productData].tsx` | `useAppState` + `useAppActions` | `selectedBaseSite`, `status`, `setBaseSiteForCode` |
| `app/owner/_layout.tsx` | `useAppState` | `selectedBaseSite` |
| `app/owner/dashboard/products/[productId].tsx` | `useAppState` + `useAppActions` | `selectedBaseSite`, `getConfiguration` |
| `src/components/ConsumerProtectionBanner.tsx` | `useAppState` | `selectedBaseSite` |
| `src/components/SearchInput.tsx` | `useAppState` | `selectedBaseSite` |
| `src/components/basic/CategorySelector.tsx` | `useAppState` | `selectedBaseSite` |
| `src/components/basic/CitySelector.tsx` | `useAppState` | `selectedBaseSite` |
| `src/components/dialog/dialogs/CurrencySelectionDialog.tsx` | `useAppState` | `selectedBaseSite` |
| `src/components/dialog/dialogs/FiltersDialog.tsx` | `useAppState` | `selectedBaseSite` |
| `src/components/dialog/dialogs/PortalConfigDialog.tsx` | `useAppState` + `useAppActions` | `selectedBaseSite`, `selectedLanguage`, `baseSites`, `setBaseSiteForCode`, `setLanguageForCode` |
| `src/components/dialog/dialogs/PreviewProductDialog.tsx` | `useAppState` | `selectedBaseSite` |
| `src/components/dialog/dialogs/product-creation/AddUpdateProductDialog.tsx` | `useAppState` | `selectedBaseSite` |
| `src/components/dialog/dialogs/product-creation/BasicInfoProductDialog.tsx` | `useAppActions` | `getConfiguration` |
| `src/components/filters/CurrencySelector.tsx` | `useAppState` | `selectedBaseSite` |
| `src/components/filters/ProductOrder.tsx` | `useAppState` | `selectedBaseSite` |
| `src/components/filters/RegionCityFilter.tsx` | `useAppState` | `selectedBaseSite` |
| `src/components/init/BaseSiteSelector.tsx` | `useAppState` + `useAppActions` | `baseSites`, `setBaseSiteForCode` |
| `src/components/navigation/CategoryNavigation.tsx` | `useAppState` | `selectedBaseSite` |
| `src/components/navigation/Filters.tsx` | `useAppState` | `selectedBaseSite` |
| `src/components/navigation/Footer.tsx` | `useAppState` | `selectedBaseSite` |
| `src/components/navigation/TopBar.tsx` | `useAppState` | `selectedBaseSite` |
| `src/components/product/ProductBreadcrumb.tsx` | `useAppState` | `selectedBaseSite` |
| `src/components/product/ProductList.tsx` | `useAppState` | `selectedLanguage` |
| `src/components/user/UserMenu.tsx` | `useAppState` + `useAppActions` | `selectedBaseSite`, `setBaseSiteForCode` |

## Files touched

- `src/components/context/AppContext.tsx` (refactored — two contexts, memoized values, useCallback actions)
- `app/_layout.tsx` (type import: `OglasinoAppState` → `AppStateValue`)
- 20 state-only consumers (import + usage: `useAppContext` → `useAppState`)
- 6 mixed/action consumers (import + usage split across `useAppState` + `useAppActions`)
- **28 files total**

## Tests

- Ran: `npx tsc --noEmit` — exit 0
- Ran: `npm run lint` — 0 errors, 75 warnings (matches Φ1 baseline)
- Ran: `npm test` — 109 passed, 0 failed
- Ran: `npx expo-doctor` — pre-existing 8-package version mismatch only, no new failures
- Ran: `grep useAppContext src/ app/` — zero results

## Cleanup performed

- Deleted `useAppContext` hook and the `AppContextType` type (replaced by `useAppState` + `useAppActions` and `AppStateValue` + `AppActionsValue`)
- Deleted the `AppContext` single context variable (replaced by `AppStateContext` + `AppActionsContext`)
- Renamed `OglasinoAppState` → `AppStateValue` — the old name had zero references outside AppContext.tsx except `app/_layout.tsx`, which was updated

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: the `2026-05-27 — AppContext.Provider value not memoized` entry (F10) is now resolved by this session. Draft for Docs/QA in "For Mastermind" below.

## Obsoleted by this session

- `useAppContext` hook — deleted in this session
- `AppContextType` type — deleted in this session (replaced by separate `AppStateValue` and `AppActionsValue`)
- `OglasinoAppState` type name — renamed to `AppStateValue`, updated in the one external consumer (`app/_layout.tsx`)
- The issues.md `2026-05-27 — AppContext.Provider value not memoized` entry — resolved by this session's memoization work

## Conventions check

- Part 4 (cleanliness): confirmed — no commented-out code, no unused imports, no console.log, no TODO/FIXME added
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): confirmed — one observation flagged in "For Mastermind"
- Part 6 (translations): N/A this session
- Other parts touched: Part 8 (architectural defaults) — confirmed, no new routes or contracts

## Known gaps / TODOs

- none

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity):
    - `stateRef` + `onLanguageChangedRef` refs in AppContextProvider — required to make the three action `useCallback`s truly stable (no state in deps). Without refs, the actions would change identity on every state change, defeating the split's purpose. Standard React pattern for stable callbacks that read current state.
  - Considered and rejected:
    - Selector-style hook (`useAppState(s => s.selectedBaseSite)`) — React Context doesn't natively support selectors; `use-context-selector` library or `useSyncExternalStore` hand-rolling is overkill for Φ3. State changes are rare. Plain destructure is sufficient per the brief's §3 guidance.
    - Custom `areEqual` comparator on state context — unnecessary since `AppStateValue` identity only changes when `setState` produces a new value (already rare).
  - Simplified or removed:
    - Fixed stale-closure bug in `setBaseSiteForCode`: the final `setStateWithStatusHook` call used `...state` (captured at function-definition time, stale if state had changed between definition and call). Replaced with functional updater `(prev) => ({ ...prev, ... })`. This is a correctness fix, not just a simplification.
    - Removed the `AppContextType` union type (was `OglasinoAppState & actions`). The two-context model uses two separate, focused types instead of a single merged type.

- **Part 4b adjacent observation:**
  - `setLanguageForCode` currently reads `state.selectedBaseSite` to resolve the language against the site's allowed languages. With the ref pattern, this reads correctly. However, the function doesn't guard against `selectedBaseSite` being `undefined` at the type level — it early-returns with `if (!site) return;` at runtime. This is identical to the pre-refactor behavior and is fine.

- **Audit consumer count discrepancy (minor, non-blocking):** The audit states "22 of 26 consumers read only `selectedBaseSite`" but actual count is 19 state-only-selectedBaseSite consumers + 1 selectedLanguage-only (ProductList) + 1 actions-only (BasicInfoProductDialog) + 5 mixed. The 22 number appears to include some mixed consumers. This didn't affect the implementation — the per-file migration table is the ground truth.

- **Config-file draft — issues.md:** The `2026-05-27 — AppContext.Provider value not memoized` entry should be marked `fixed` with note: "Fixed in session `2026-05-27-oglasino-expo-phi3-5`. AppContext split into two memoized contexts; all four action functions wrapped in `useCallback`; `actionsValue` stable via `useMemo`; `stateValue` stable via `useMemo`. The provider value is no longer a new object every render."
