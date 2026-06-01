# Session summary

**Slug:** expo-system-theme · **Session:** 2 (first implementation session; `-1` was the read-only audit)
**Repo:** oglasino-expo · **Branch:** new-expo-dev (not switched, not committed)
**Date:** 2026-06-01
**Task:** Add a `'system'` (follow-OS) theme option to mobile, bringing it to parity with web's tri-state theme — app-owned choice/resolved store, `Appearance`-backed system resolution + live OS listener, ungated AsyncStorage persistence of the choice, boot hydrate/apply bridging the resolved value into NativeWind, a 3-segment Sun/Monitor/Moon control in `PortalConfigDialog`, and replacing the `DashboardSidebar` binary toggle with a button that opens that dialog.

> **Mid-session note (for Igor/Mastermind):** `.agent/brief.md` was overwritten partway through this session with an unrelated M1/M2 bug-batch brief. Igor confirmed System Theme is the real task and the swap was unintended. A one-line M1 edit to `ProductUserDetails.tsx` made during the confusion was fully reverted (`git diff` against that file shows only pre-existing branch work, none of mine). The System Theme brief content now lives only in this session's transcript — the canonical spec (`features/expo-system-theme.md`) is Docs/QA's to author in Phase 4.

## Implemented

- **App-owned theme store with the load-bearing choice/resolved split** (`src/lib/store/useThemeStore.ts`), mirroring web's `useTheme` plus the house AsyncStorage-hydrated store shape (`useConsentStore`/`useCardSizeStore`): `theme` (the choice, default `'system'`), `resolvedTheme` (effective binary, never `'system'`), `hydrated`. Two distinct write paths — `setTheme(choice)` (user pick: sets both, bridges, persists the choice) and `_setResolvedTheme(resolved)` (internal, OS-listener-only: updates resolved + bridges, never touches/persists the choice). `'system'` resolves against `Appearance.getColorScheme()`.
- **Apply bridge into NativeWind.** Both writers call NativeWind's imperative `colorScheme.set(resolved)` (from `nativewind`) as the apply step. The `'system'` value is confined to the store — only resolved `'light'`/`'dark'` ever reaches NativeWind, so `NAV_THEME[...]` (`app/_layout.tsx:77`) and the ~20 binary `=== 'dark'` consumers stay correct with zero changes.
- **Ungated persistence of the choice** (`src/lib/storage/themeStorage.ts`), a dedicated per-concern module mirroring `consentStorage` (own key `theme_choice`, direct AsyncStorage, validates-then-defaults). Persists `'light'|'dark'|'system'`, never the resolved value. Absent → `'system'`.
- **Boot hydrate + live OS listener** (`src/components/init/ThemeInit.tsx`), mounted unconditionally in `app/_layout.tsx` next to `ConsentInit` so the resolved theme is applied during the boot overlay, before the portal Stack paints. Hydrates the stored choice → resolves → bridges on mount; subscribes to `Appearance.addChangeListener` only while the choice is `'system'` (effect keyed on `isSystem`, mirroring web's `SyncThemeFromSystem`).
- **3-segment Sun/Monitor/Moon control** in `PortalConfigDialog.tsx` (web's order/icons via `lucide-react-native`), replacing the binary flip toggle. Highlights the active segment against the choice; writes via `setTheme`. Accessibility labels go through DIALOG translation keys (not raw value strings — mobile starts correct vs. web's raw `aria-label`). Visually consistent with the dialog's adjacent language picker (`border-green-400` active state).
- **`DashboardSidebar.tsx` binary toggle removed**, replaced with a button that closes the sidebar and opens `PORTAL_CONFIG_DIALOG` — theme is now switched from the one real control, single source of truth. (Final icon `MonitorCog`, a11y label via `portal.config.title`.)
- Introduced `Theme` / `ResolvedTheme` types (`src/lib/types/ui/Theme.ts`).

## Files touched

New:
- `src/lib/types/ui/Theme.ts` (+15)
- `src/lib/storage/themeStorage.ts` (+46)
- `src/lib/store/useThemeStore.ts` (+78)
- `src/components/init/ThemeInit.tsx` (+40)

Edited (theme-scoped portions only; these files also carry pre-existing uncommitted branch work):
- `app/_layout.tsx` (+2: `ThemeInit` import + unconditional mount)
- `src/components/dialog/dialogs/PortalConfigDialog.tsx` (swap `useColorScheme`→`useThemeStore`; `MoonStar/Sun`→`Monitor/Moon/Sun`+`LucideIcon`; add `Theme` import + `THEME_OPTIONS`; replace toggle block with 3-segment map)
- `src/components/dashboard/layout/DashboardSidebar.tsx` (drop `useColorScheme`/`Sun`/`MoonStar`; add `MonitorCog`+`tDialog`; replace toggle with config opener)

Reverted to baseline (no net change): `src/components/product/ProductUserDetails.tsx` (the stray M1 edit was undone).

## Tests

- Ran: `npx tsc --noEmit` → exit 0 (clean).
- Ran: `npx eslint .` → 82 problems, **0 errors / 82 warnings** (state.md Risk-Watch baseline was 84; held/under — the removed binary-toggle code dropped two unused-symbol warnings).
- Ran: `npx vitest run` → **26 files, 334 tests, all passing.**
- New tests added: none — see "Known gaps / TODOs" for the choice/resolved guard test deliberation.

## Cleanup performed

- Removed the now-dead `useColorScheme` import + `colorScheme`/`toggleColorScheme` destructure from both `PortalConfigDialog.tsx` and `DashboardSidebar.tsx`, and the `MoonStar`/`Sun` (sidebar) lucide imports that the deleted binary toggles used. No commented-out code, no `console.log`, no TODO/FIXME introduced.

## Config-file impact

- **conventions.md:** no change.
- **decisions.md:** no change required by me. (Mastermind may want a closing entry mirroring the consent-mode-mobile precedent — mobile mirrors web's *pattern* not its *mechanism* — but that is Mastermind/Docs/QA's call, not a contract I changed.)
- **state.md:** **change needed (drafted below in "For Mastermind").** No Expo-backlog row exists for system-theme yet; one should be appended (feature is web-stable; mobile now code-complete on `new-expo-dev`, pending Ψ). Closure gate: this is the implicit config dependency — stated, not silently left.
- **issues.md:** **change needed (drafted below).** The line-57 parity-gap item ("No system-theme (follow-OS) option on mobile") should flip to fixed-in-code / pending-Ψ.

## Obsoleted by this session

- The two binary `toggleColorScheme` call sites (the flip toggles in `PortalConfigDialog` and `DashboardSidebar`) — both deleted in this session. `toggleColorScheme` (a light⇄dark flip with no third slot) is no longer called anywhere; the explicit `setTheme(choice)` selector + the `colorScheme.set` bridge replace it.

## Conventions check

- **Part 4 (cleanliness):** confirmed — dead imports/destructures from the removed toggles deleted; tsc clean; lint 0 errors / 82 warnings (≤ baseline); tests green.
- **Part 4a (simplicity):** see structured evidence in "For Mastermind."
- **Part 4b (adjacent observations):** flagged in "For Mastermind" (pre-existing dead vars in `PortalConfigDialog`).
- **Part 6 (translations):** new keys are net-new and **backend-seeded** (DIALOG namespace, `portal.config.theme.*` family) — I consume them, I do not author the seed. No parent/child collision: `portal.config.theme.toggle.label` (existing section label) and `portal.config.theme.option.{light,system,dark}.label` (new) are sibling leaves under `portal.config.theme.*`, none is a leaf-then-nested parent. Keys drafted for the backend in "For Mastermind."
- **Other parts touched:** Part 8 (architectural defaults) — no new mobile-specific routes; reused the existing dialog system. The choice/resolved invariant and the "resolved-only reaches NativeWind/`NAV_THEME`/ternaries" rule from the audit are upheld.

## Known gaps / TODOs

- **New translation keys not yet seeded.** Until the backend seeds `portal.config.theme.option.{light,system,dark}.label` across the four locales, the three segments render their raw key strings on device. This is a cross-repo dependency (backend seed), not a mobile bug — drafted in "For Mastermind."
- **No unit test for the choice/resolved split.** The canonical bug (collapsing the two setters) is exactly the kind the per-language-checksum work guarded with a dedicated test. I did **not** add one this session: the expo suite is node-env (`vitest.config.ts` `environment: 'node'`), and a store test needs `nativewind`'s imperative `colorScheme` and RN `Appearance` mocked — non-trivial mock surface with no existing precedent in the suite for mocking NativeWind. Flagged for Mastermind to decide whether a guard test is worth the mock scaffolding (recommend yes, as a small follow-up).
- **On-device verification (Ψ) owed**, per the branch posture — feature ships `shipped (code)` / `verifying` and rides the existing pending iOS+Android rebuild + on-device smoke dependency rather than getting its own device pass. [This matches the brief's INFERRED close-out item; confirm.]

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - **Added (earned complexity):**
    - `useThemeStore` (new store) — earned: there is no app-owned theme state today to widen, and the choice/resolved split + persistence + OS listener cannot live in NativeWind's binary `useColorScheme`. One concrete caller today (the control), with the boot path + listener as immediate second/third consumers.
    - `themeStorage.ts` (dedicated module) — earned: matches the established per-concern storage house pattern (`authStorage`/`consentStorage`); the generic `userPreferenceStorage` is a zero-caller wrapper the codebase explicitly does not use as the convention (see `consentStorage`'s own header comment). [Resolves the brief's INFERRED host question in favor of a dedicated module.]
    - `Theme`/`ResolvedTheme` types — earned: the choice/resolved distinction is the load-bearing invariant; naming the two as distinct types makes "resolved never carries `'system'`" enforceable at the type level.
    - `THEME_OPTIONS` table in `PortalConfigDialog` — earned: mirrors web's `ToggleButton` `OPTIONS` array; drives the 3-segment map and keeps icon/label/value colocated.
  - **Considered and rejected:**
    - A standalone `SyncThemeFromSystem`-style component separate from hydration — folded the listener into `ThemeInit` (one headless init component) since both are mount-lifecycle concerns; avoids a second near-empty component.
    - Touching the boot state machine (`bootStore`) to slot theme into a gate — rejected; the unconditional `ThemeInit` mount applies the resolved theme under the boot overlay before the portal paints, with zero risk to the gate machine's three invariants. (Matches the brief's INFERRED placement latitude.)
    - Using NativeWind's native `colorScheme.set('system')` (it supports `'system'`) — rejected per the invariant: the store owns system resolution so `NAV_THEME` and every `=== 'dark'` ternary only ever see a resolved binary.
  - **Simplified or removed:** deleted both binary `toggleColorScheme` toggles and their now-dead imports; theme is now switched from a single control.

- **Drafted config-file edits (Docs/QA to apply):**
  - **state.md — new Expo-backlog row** (append to the table at `## Expo backlog`):
    `| [System theme](features/expo-system-theme.md) | web-stable | in-progress | oglasino-expo-expo-system-theme-1..-2 | Mobile tri-state theme (light/system/dark) code-complete on new-expo-dev (`-1` read-only audit; `-2` implementation): app-owned choice/resolved store, Appearance-backed system resolution + live OS listener, ungated AsyncStorage persistence of the choice, boot hydrate/apply bridging resolved into NativeWind, 3-segment Sun/Monitor/Moon control in PortalConfigDialog, sidebar toggle → config opener. Pending: backend seed of the three `portal.config.theme.option.*.label` keys (4 locales) + on-device Ψ (shares the pending iOS+Android-rebuild dependency). Do not promote to adopted/mobile-stable until Ψ passes. |`
  - **issues.md — flip the line-57 parity item** ("No system-theme (follow-OS) option on mobile") `open` → fixed-in-code on `new-expo-dev` (`oglasino-expo-expo-system-theme-2`), pending backend seed + Ψ.
  - **state.md — Risk Watch** (if maintained): the three new DIALOG keys (`portal.config.theme.option.{light,system,dark}.label`) need RS/RU/CNR native-translator review (EN final), same precedent as consent-mode-mobile / image-source keys.

- **Backend seed needed (cross-repo, backend agent):** DIALOG namespace, `portal.config.theme.*` family, alphabetical-where-reasonable, EN final / RS·RU·CNR placeholder per precedent:
  - `portal.config.theme.option.light.label`
  - `portal.config.theme.option.system.label`
  - `portal.config.theme.option.dark.label`

- **Adjacent observations (Part 4b):**
  - `PortalConfigDialog.tsx:44–45` — `const router = useRouter()` and `const segments = useSegments()` are assigned but never used (lint warnings, pre-existing, not introduced by me). Severity low (cosmetic dead vars). I did not fix this because it is out of scope for the theme work and removing the `expo-router` imports touches lines unrelated to the feature.
  - `DashboardSidebar.tsx:41` — the slide animation `useEffect` is missing `screenWidth`/`slideAnim` deps (pre-existing `react-hooks/exhaustive-deps` warning). Severity low. Not fixed — out of scope.

- **Brief vs reality:** none. The audited code matched the brief on every cited fall-through site (`NAV_THEME` index, the ~20 binary consumers, the two toggle call sites, the single existing theme key). No wire-shape, platform, or contract conflict to raise. The only deviation from the brief's literal text is the `DashboardSidebar` opener icon (`MonitorCog` vs. the brief's unspecified icon) — an intentional in-session choice, no behavior impact.
