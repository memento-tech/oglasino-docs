# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-28
**Task:** Move barrier resolve from status transitions to code-set, mark version check as bootstrap.

## Implemented

- Moved the barrier resolve point from `setStateWithStatusHook` in `AppContext.tsx` to `setCurrentBaseSiteCode` in `apiStore.ts`. The barrier now resolves when the base-site code is set (the actual invariant the interceptor needs), not when status reaches a terminal value. This fixes the first-time-user bug where `select-base-site` status resolved the barrier with null codes.
- Deleted the `resolveBootstrap()` and `isBootstrapResolved()` exports from `apiStore.ts` — both are dead code after the resolve logic moved into the setter.
- Changed the interceptor's barrier check from `!isBootstrapResolved()` to `!getCurrentBaseSiteCode()`. Post-bootstrap requests pay zero cost (codes are non-null, check is `false`, no await).
- Marked `appVersionConfigService.tsx`'s version check with `{ _bootstrap: true }`. All four pre-bootstrap calls are now consistently marked.
- Removed the `[BARRIER]` diagnostic `console.log` from the request interceptor in `api.ts`.

## Files touched

- src/lib/config/apiStore.ts (+3 / -11) — resolve moved into setter, dead exports deleted
- src/components/context/AppContext.tsx (+0 / -5) — `resolveBootstrap` import and call removed
- src/lib/config/api.ts (+1 / -11) — `isBootstrapResolved` import removed, BARRIER log removed, barrier check changed to code-based
- src/lib/services/appVersionConfigService.tsx (+1 / -0) — `_bootstrap: true` added

## Tests

- Ran: `npx tsc --noEmit` — exit 0
- Ran: `npm run lint` — 0 errors, 73 warnings (matches Brief 14 baseline)
- Ran: `npm test` — 109 passed, 0 failed
- Ran: `grep -rn 'BARRIER' src/ app/` — zero results
- Ran: `grep -rn 'resolveBootstrap\|isBootstrapResolved' src/ app/` — zero results
- No new tests added. No existing tests broke.

## Cleanup performed

- Removed `[BARRIER]` diagnostic `console.log` (7 lines) from `api.ts` request interceptor.
- Deleted `resolveBootstrap()` export (5 lines) and `isBootstrapResolved()` export (3 lines) from `apiStore.ts`.
- Removed unused `resolveBootstrap` import from `AppContext.tsx`.
- Removed unused `isBootstrapResolved` import from `api.ts`.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- `resolveBootstrap()` export in `apiStore.ts` — deleted in this session.
- `isBootstrapResolved()` export in `apiStore.ts` — deleted in this session.
- The status-transition-based resolve pattern from Brief 14 — replaced by code-set-based resolve. The Brief 14 session summary's "Barrier resolve-on-terminal-status pattern" section is now outdated; the barrier resolves on code-set, not on status transition.

## Conventions check

- Part 4 (cleanliness): confirmed. All diagnostic code removed. No console.log added. No commented-out code. No unused imports.
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): confirmed — no new observations.
- Part 6 (translations): N/A this session
- Other parts touched: Part 8 (architectural defaults) — the barrier continues to protect at the shared HTTP layer.

## Known gaps / TODOs

- None.

## For Mastermind

### Part 4a simplicity evidence (required)

- **Added (earned complexity):** nothing. The resolve logic moved into an existing function (`setCurrentBaseSiteCode`), adding 4 lines. No new exports, no new abstractions.
- **Considered and rejected:**
  - Keeping `isBootstrapResolved()` as a helper based on `currentBaseSiteCode !== null`. Rejected: the interceptor can call `getCurrentBaseSiteCode()` directly — same semantics, one fewer export to maintain.
  - Resolving from both `setCurrentBaseSiteCode` and `setCurrentLangCode`. Rejected: codes are always set in pairs at all three bootstrap call sites (AppContext.tsx lines 130-131, 210-211), or only lang is set on the language-switch path (line 241) where base site is already set. Resolving from the base-site setter is sufficient.
- **Simplified or removed:**
  - Deleted `resolveBootstrap()` export (5 lines) — callers: zero after this session.
  - Deleted `isBootstrapResolved()` export (3 lines) — callers: zero after this session.
  - Removed the `resolveBootstrap()` call and its surrounding status check (3 lines) from `setStateWithStatusHook`.
  - Removed the BARRIER diagnostic log (7 lines) from the interceptor.

### Semantic shift — barrier resolve invariant

The barrier's resolve semantics changed from "status reached a terminal value" to "base-site code is set." This is the correct invariant because:

1. The interceptor needs valid `X-Base-Site` and `X-Lang` headers. It doesn't care about status.
2. On the `select-base-site` path, status is terminal but codes are null. The old model resolved the barrier; the new model holds it.
3. On the `maintenance` and catch-block paths, codes are never set, so the barrier never resolves — correct, because no requests should fire when the app is in maintenance or error state.

### Four bootstrap exit paths under the new model

| Path | `setCurrentBaseSiteCode` called? | Barrier resolves? | Behavior |
|---|---|---|---|
| Success (stored base site) | Yes (line 130) | Yes | Queued requests proceed with valid headers |
| Select-base-site (no stored site) | No | **No** (holds) | Overlay renders; user picks → `setBaseSiteForCode` → `setCurrentBaseSiteCode` → resolves |
| Maintenance | No | **No** (holds) | Maintenance overlay renders; no requests fire |
| Exception (catch block) | No | **No** (holds) | Falls to maintenance status; no requests fire |

### Verification of code-set pairing

Grep-verified: `setCurrentBaseSiteCode` and `setCurrentLangCode` are called together at lines 130-131 (bootstrap success), 210-211 (`setBaseSiteForCode`). Line 241 (`setLanguageForCode`) sets only lang — correct, because this path requires `selectedBaseSite` to be defined (line 230 guard), meaning the base-site code was already set in a prior call.
