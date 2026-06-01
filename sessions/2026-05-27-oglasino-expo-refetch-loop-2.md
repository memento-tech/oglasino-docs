# Session summary

**Repo:** oglasino-expo
**Branch:** `new-expo-dev`
**Date:** 2026-05-27
**Task:** Revert `AppContext.tsx` changes from previous session, then diagnose and fix the infinite bootstrap loop.

## Implemented

**Step 1 — Revert (per brief):** restored `AppContext.tsx` to audit-baseline state. Igor tested — loop persisted, confirming the bug predates this chat.

**Step 2 — Root-cause diagnosis:** added diagnostic `console.log` lines to `AppContext.tsx` and `_layout.tsx` to track render counts, effect runs, and status transitions. Igor's Metro output revealed two independent bugs:

1. **Navigator remount loop (primary).** The `<Stack>` in `_layout.tsx` was conditionally rendered: `{(status === 'ready' || status === 'loading') && <Stack />}`. On first render, `status` is `undefined` → Stack not mounted. Bootstrap completes → `status` becomes `'ready'` → Stack mounts for the first time. Expo-router detects a new navigator appearing in the tree and **reinitializes the navigation container**, which remounts the root layout, resetting all `useState` to initial values (`status` back to `undefined`). Stack unmounts → bootstrap runs again → infinite cycle. Diagnostic proof: `[AppContext] RENDER #1` (render count resetting to 1 = component remounting, not re-rendering) and `[RootLayout] status=undefined` appearing after `status=ready`.

2. **Effect dep write-read cycle (secondary).** The main effect in `AppContext.tsx` had `state.status` in its dependency array. Since `bootstrap()` writes to `state.status`, the effect re-triggered on every bootstrap completion. Would cause double-bootstrap even without the remount loop.

**Step 3 — Fix both bugs:**

- `app/_layout.tsx`: `<Stack>` is now always mounted. `<AppInit>` only mounts when `status === 'ready'`. Loading, maintenance, and base-site-selector are rendered as absolute-positioned full-screen overlays on top of the Stack, instead of replacing it.
- `app/(portal)/_layout.tsx`: added a `selectedBaseSite` guard around the header UI (TopBar, CategoryNavigation, ConsumerProtectionBanner). These components access `selectedBaseSite` which is undefined before bootstrap completes. The `<Tabs>` navigator stays always mounted. Without this guard, the always-mounted Stack renders the portal layout before bootstrap finishes, crashing on `selectedBaseSite.iconId`.
- `src/components/context/AppContext.tsx`: removed `state.status` from the main effect's deps (now `[bootstrap, setStateWithStatusHook]`). Added `statusRef` + sync effect so the polling interval can read current status without triggering effect re-runs.

**Step 4 — Cleanup:** removed all diagnostic `console.log` lines and temporary refs/counters.

## Files touched

- `src/components/context/AppContext.tsx` (+8 / -2 net vs audit baseline)
- `app/_layout.tsx` (+10 / -14)
- `app/(portal)/_layout.tsx` (+5 / -2)

## Tests

- Ran: `npx tsc --noEmit` — exit 0
- Ran: `npm run lint` — 0 errors, 80 warnings (Φ2 baseline, unchanged)
- Ran: `npm test` — 109 passed, 0 failed
- Manual: Igor confirmed the app loads without looping
- New tests added: none

## Cleanup performed

- Removed all diagnostic `console.log` lines and temporary refs/counters from `AppContext.tsx` and `_layout.tsx`.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- The audit-baseline `_layout.tsx` pattern of conditionally rendering `<Stack>` — replaced with always-mounted Stack + overlay pattern. The conditional pattern caused expo-router to remount the navigation tree on every status transition.
- The audit-baseline `AppContext.tsx` effect deps `[bootstrap, state.status, setStateWithStatusHook]` — `state.status` removed, replaced with `statusRef` pattern.

## Conventions check

- Part 4 (cleanliness): confirmed — no commented-out code, no unused imports/variables, no debug logging, no TODOs.
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): flagged in "For Mastermind"
- Part 6 (translations): N/A this session
- Other parts touched: none

## Known gaps / TODOs

- None. Both bugs are fixed and verified by Igor.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): (1) `statusRef` + sync effect in AppContext (1 ref + 3-line effect) — earned because the polling interval runs in a stale closure; the ref is the standard React pattern for reading current state inside stale closures. (2) `selectedBaseSite` guard in `PortalLayout` — earned because the always-mounted Stack renders route layouts before bootstrap completes; the guard prevents crashes on undefined context data.
  - Considered and rejected: wrapping the Stack in a `display: 'none'` container instead of the overlay approach. Rejected because React Native still runs render functions for hidden children — route components would still crash accessing undefined AppContext data.
  - Simplified or removed: the conditional `{(status === 'ready' || status === 'loading') && (<><AppInit /><Stack />...</>)}` block in `_layout.tsx` was replaced with a simpler always-mounted pattern. Net reduction: 4 fewer lines.

- **The brief's theory was wrong.** The brief assumed the refetch-loop-1 `AppContext.tsx` changes caused the loop. In fact, two independent bugs existed in the audit-baseline code: (1) the navigator remount loop in `_layout.tsx` (primary cause), and (2) the effect dep write-read cycle in `AppContext.tsx` (secondary cause). The refetch-loop-1 session correctly fixed bug #2 but couldn't fix bug #1 because it only touched `AppContext.tsx`.

- **Why the bug wasn't visible before Φ2.** Before Φ2, `_layout.tsx` used `<Slot />` instead of `<Stack>`. Bare `<Slot />` does not register a navigator with expo-router, so conditionally rendering it did not trigger the navigation-container reinitialization. The Φ2 work replaced `<Slot />` with `<Stack>` (addressing structural audit finding F12), which exposed the conditional-rendering incompatibility. The bug was latent in the conditional-rendering pattern; Φ2 activated it.

- **Adjacent observations (Part 4b, not fixed this session):**
  1. `AppContext.Provider` value at line 248-256 creates a new object on every render because `setBaseSiteForCode`, `setLanguageForCode`, and `getConfiguration` are regular functions (not `useCallback`). Structural audit finding F10 / Φ3 scope. Severity: low (performance). Path: `src/components/context/AppContext.tsx:248`. I did not fix this because it is out of scope.
  2. `configurationService.tsx:6` declares return type `Promise<ConfigMap>` but returns `null` on failure. The type contract is misleading. Severity: low. Path: `src/lib/services/configurationService.tsx:6`. I did not fix this because it is out of scope.
  3. Multiple require cycles warned at startup (api.ts → authStore → authService → api.ts and variants). Already tracked in `issues.md` 2026-05-25 entry. Severity: low (dynamic only, no runtime issue). I did not fix this because it is out of scope.
