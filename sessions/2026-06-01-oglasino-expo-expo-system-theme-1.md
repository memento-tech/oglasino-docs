# Session summary

**Slug:** expo-system-theme · **Session:** 1 (first expo-side session for this slug)
**Repo:** oglasino-expo · **Branch:** new-expo-dev (not switched, not committed)
**Date:** 2026-06-01
**Type:** READ-ONLY current-state audit (no code changes)

## Task (one sentence)

Inventory the current mobile theme implementation — store/state, persistence, default, apply mechanism, OS-appearance usage, toggle UI, translation keys, and the seam for adding `'system'` — so the spec can mirror web's three-state machine.

## Implemented

Nothing implemented (read-only audit). Deliverable is the eight-section inventory at `.agent/audit-expo-system-theme.md`, every cited file backed by raw `cat -n`/`grep`/`sed -n` shell output per the tool-fabrication guard.

**Key findings:**

- **No app-owned theme store/context/hook.** Theme "state" is NativeWind's `useColorScheme()` consumed directly in 20 files. Two-state only (`light`/`dark`); no `'system'`, no choice/resolved split.
- **No persistence.** No AsyncStorage theme key, no `setColorScheme` call, no boot restore. Theme resets to NativeWind's runtime default on every launch. Writers (`toggleColorScheme`) are unconditional/ungated (consistent with consent-mode-mobile's functional-theme premise, but the stronger fact is it isn't persisted at all).
- **No app-level default**; only a `colorScheme ?? 'light'` render fallback for the React Navigation theme in `app/_layout.tsx:77`. Fresh-install value is NativeWind-runtime-owned, not provable from repo source (stated honestly).
- **Apply mechanism:** NativeWind `darkMode: 'class'` + CSS-var tokens (`global.css` `:root` / `.dark:root`), plus `NAV_THEME` fed to RN `ThemeProvider`, plus ~scattered binary `colorScheme === 'dark'` color ternaries.
- **`Appearance` is imported nowhere; `prefers-color-scheme` absent.** The OS-following primitive is net-new.
- **Toggle UI:** two binary icon toggles — `PortalConfigDialog.tsx` (labelled, DIALOG-ns key `portal.config.theme.toggle.label`) and `DashboardSidebar.tsx` (icon-only, no label). Both call `toggleColorScheme` (a flip, not a set). `app/owner/dashboard/user.tsx` has NO theme control today.
- **Translation keys:** exactly one — `portal.config.theme.toggle.label` (DIALOG ns, backend-seeded). No web aria-label-as-raw-value antipattern.
- **Seam:** adding `'system'` is mostly net-new (store + persistence + `Appearance` listener + `setColorScheme` bridge), not a widen-in-place, because there's no app store to widen and `toggleColorScheme` has no third slot. Flagged fall-through risks: `NAV_THEME: Record<'light'|'dark'>` index (`theme.ts:58`), and ~20 binary `=== 'dark'` consumers that must only ever receive the *resolved* value.

## Files touched

- `.agent/audit-expo-system-theme.md` — created (the eight-section inventory).
- `.agent/2026-06-01-oglasino-expo-expo-system-theme-1.md` — this summary.
- `.agent/last-session.md` — exact copy of this summary.

No source files modified.

## Tests

Not run — read-only audit, no source changes. `lint`/`tsc`/`test` not applicable (no touched source paths).

## Cleanup performed

None needed (read-only; no code written).

## Config-file impact

No change. This is an audit, not a feature adoption — nothing to add to or remove from `state.md`'s Expo backlog table, and no `conventions.md`/`decisions.md`/`issues.md` edit is implied. **Closure gate:** confirmed no implicit config-file dependency. (When the spec is later adopted on mobile, the backlog-table row for `expo-system-theme` will be the relevant edit — not this session.)

## Obsoleted by this session

Nothing.

## Conventions check

- **Part 4 (cleanliness):** N/A — no code written; no commented code, unused imports, console.logs, or TODOs introduced.
- **Part 4a (simplicity):** N/A — no implementation.
- **Part 4b (adjacent observations):** While auditing I noted (a) `app/owner/dashboard/user.tsx` has no theme control despite the consent-mode-mobile lineage suggesting it as a settings home, and (b) the `DashboardSidebar` theme toggle is unlabelled (icon-only) while `PortalConfigDialog`'s is labelled — a minor inconsistency the spec may want to unify. Both are reported in the audit (§6); neither is a defect to fix here.
- **Tool-fabrication guard:** every cited file proven to exist via shell output; absent artifacts (theme store, `Appearance`, `setColorScheme`, theme AsyncStorage key) reported as explicitly-absent with their failed/empty greps rather than invented.

## Known gaps / TODOs

- NativeWind v4's fresh-install default color scheme is library-runtime behavior I could not prove from repo source; the audit states this honestly rather than asserting it. The spec author should confirm NativeWind's default-follow-OS behavior against the installed version (`nativewind@^4.2.2`) if it matters to the boot UX.

## For Mastermind / Brief vs reality

No brief-vs-reality conflict to raise — this was an open-ended inventory, not an implementation against a contract. Two pointers for the spec:

1. **`app/owner/dashboard/user.tsx` is NOT the current theme home.** If the spec assumes the settings screen already hosts a theme control (per consent-mode-mobile framing), it does not — the two live controls are `PortalConfigDialog.tsx` and `DashboardSidebar.tsx`. Decide whether the 3-way control consolidates there or stays in those two dialogs.
2. **`toggleColorScheme` cannot be reused for 3-state** — it's a binary flip. Both call sites need replacing with an explicit `setTheme(choice)` selector, and the resolved value (never the choice) must keep driving `NAV_THEME` and the ~20 `=== 'dark'` consumers.
