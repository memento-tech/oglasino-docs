# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-29
**Task:** Read-only audit mapping the AppContext consumer surface (useAppState/useAppActions + RequireBaseSite) so Mastermind can choose between Option A (re-point consumers to bootStore) and Option B (bridge pick → AppContext). Output to `.agent/audit-post-pick-consumers.md`.

## Implemented

- Read-only audit only — no code changed.
- Confirmed the brief's working hypothesis against the code: `bootStore.pickBaseSite` (`bootStore.ts:240-266`) writes `selectedBaseSite`/`language` to bootStore + AsyncStorage but never to AppContext; AppContext.bootstrap only populates `selectedBaseSite` from a STORED site, so a fresh-install pick leaves AppContext at `undefined`. Both systems are mounted side-by-side in `app/_layout.tsx:42,76`.
- Produced an exhaustive call-site inventory: 25 `useAppState()` sites and 5 `useAppActions()` sites across 26 distinct files, grouped by field and action.
- Pinned the two load-bearing failure points: `app/(portal)/_layout.tsx:12-25` gates the whole header chrome on `AppContext.selectedBaseSite`, and `app/(portal)/(public)/index.tsx:10-19` gates the portal product list via `RequireBaseSite` (which reads the same field).
- Mapped the non-i18n language consumers (ProductList, PortalConfigDialog), quoted both `setBaseSiteForCode`/`setLanguageForCode` bodies with their side effects, and confirmed runtime language switching exists only via `PortalConfigDialog → AppContext.setLanguageForCode`.

## Files touched

- `.agent/audit-post-pick-consumers.md` (new, audit deliverable)
- `.agent/2026-05-29-oglasino-expo-post-pick-consumers-1.md` (new, this summary)
- `.agent/last-session.md` (overwritten copy of this summary)

No source files modified.

## Tests

- Not run. Read-only audit; no code changes, so lint/tsc/test gates do not apply.

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change. This audit is a sub-step of the active `expo-performance-foundation` (Φ3) boot-redesign work on `new-expo-dev` (state.md §"Expo performance foundation"); it does not change any feature status or backlog row, and no Expo-backlog row is adopted/removed by an audit.
- issues.md: no change

## Obsoleted by this session

- Nothing. The audit confirms (does not contradict) the `RequireBaseSite` mapping from `audit-expo-boot-redesign` §5.3 / `audit-picker-seam` §5 against current code.

## Conventions check

- Part 4 (cleanliness): confirmed — no code touched; no debug logging, dead code, or stray TODOs introduced.
- Part 4a (simplicity): N/A — no code added; see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): two observations flagged in "For Mastermind."
- Part 6 (translations): N/A this session — no translation keys touched.
- Other parts touched: Part 11 (trust boundaries) — N/A (no DTO/trust surface changed); the audit only reads client-side in-memory state divergence.

## Known gaps / TODOs

- None. The audit answers all 8 brief questions with file paths, line ranges, and quoted code, and states its caveats (RequireBaseSite counting, getConfiguration/reBootstrap being non-base-site concerns) explicitly in the audit's "Ambiguities" section.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code.
  - Considered and rejected: nothing — no implementation choices were made.
  - Simplified or removed: nothing.

- **Adjacent observations (Part 4b):**
  1. **Live `console.log` boot instrumentation still present.** `app/_layout.tsx:41,49` (`[MOUNT-EFFECT]`, `[OVERLAY]`) and `bootStore.ts:112` (`[BOOT-START]`). Severity: low (debug noise, not user-facing). state.md §"Expo performance foundation" already lists removing the `[BOOT]` diagnostics as a tracked follow-up for the boot chat, so this is known — flagging only to confirm it spans `_layout.tsx`/`bootStore.ts`, not just `api.ts`. I did not fix this because it is out of scope (read-only audit).
  2. **`reBootstrap` action is exposed but has zero consumers** (`AppContext.tsx:33,261`; no call sites in `src/`/`app/`). Severity: low (dead-ish surface). Likely becomes relevant only to the A/B decision; not fixing — out of scope.

- **Decision-relevant facts surfaced (not a recommendation):**
  - N_components = 26, N_state_reads = 25, N_action_calls = 5. `selectedBaseSite` dominates (24 of 25 read sites); `selectedLanguage` only 2; `baseSites`/`status` 1 each; `getCurrentLocale` 0.
  - Option B is NOT a plain `setState` bridge: `setBaseSiteForCode` resolves its site from the in-memory `baseSites` (full DTO list) which the picker path does not populate (it uses `baseSiteOverviews`), and both actions also persist to AsyncStorage, call `initI18n`, drive the AppContext status machine, and (language) fire the `onLanguageChanged` prop wired to `setLocale` in `app/_layout.tsx:76`. Several of these effects overlap with what bootStore already does (double-persist / double-i18n risk). Detail in audit §6.
  - Runtime language switching (PortalConfigDialog → `setLanguageForCode`) is in scope for either fix; `ProductList` keys its refetch off `AppContext.selectedLanguage` reactivity (audit §5, §7).

- Nothing else flagged.
