# Expo System Theme

Adds a `'system'` (follow-OS) theme option to `oglasino-expo`, bringing mobile to parity with web's tri-state theme. Web already ships this; web does not change. Mobile adopts web's *pattern* (the three-state choice → two-state resolved machine), not its *mechanism* (web persists to a consent-gated cookie with an SSR pre-paint snippet; mobile has neither). Precedent: the 2026-05-30 consent-mode-mobile decision — mobile mirrors web's pattern, not web's mechanism.

## Status

`shipped (code)` / `verifying`. Branch: `new-expo-dev` (Igor's isolation branch, see decisions.md 2026-05-25). On-device verification (Ψ) owed; rides the pending iOS+Android rebuild dependency (see state.md Risk Watch). Do not promote to `mobile-stable` until Ψ passes.

## The machine (mirrors web)

App-owned theme store (`src/lib/store/useThemeStore.ts`) holds two values:

- `theme: 'light' | 'dark' | 'system'` — the user's **choice**. Default `'system'`.
- `resolvedTheme: 'light' | 'dark'` — the **effective** appearance. Never `'system'`.

`'system'` resolves against the OS via react-native `Appearance.getColorScheme()`. A live listener (`Appearance.addChangeListener`) updates `resolvedTheme` on OS-appearance change, but only while `theme === 'system'`.

Two distinct write paths — the load-bearing, non-negotiable split:

- `setTheme(choice)` — user picked a segment. Sets `theme` and `resolvedTheme`, bridges the resolved value into NativeWind, persists the **choice** to AsyncStorage.
- `_setResolvedTheme(resolved)` — internal, OS-listener-only. Updates `resolvedTheme` and bridges. Does not touch `theme`, does not persist.

Collapsing these is the canonical bug: an OS dark-mode flip would overwrite the stored `'system'` choice with `'dark'`, losing the user's intent on next launch.

## Apply bridge

Web applies via `.dark` on `<html>`; the store is the apply mechanism. Mobile can't — NativeWind owns the binary `colorScheme`, and everything downstream (~20 binary consumers, `NAV_THEME`, the CSS-var cascade) reads NativeWind's resolved value. So the store bridges: both writers call NativeWind's `colorScheme.set(resolvedTheme)` (net-new — called nowhere before this feature).

**Invariant: `'system'` lives only in the store. It never reaches NativeWind, `NAV_THEME` (`src/lib/theme.ts`, `Record<'light' | 'dark'>`, indexed at `app/_layout.tsx:77`), or any `=== 'dark'`/`=== 'light'` ternary.** Those see only resolved `'light'`/`'dark'`, so all ~20 existing binary consumers stay correct with zero changes.

## Persistence

Dedicated `src/lib/storage/themeStorage.ts` (key `theme_choice`), mirroring the `consentStorage`/`authStorage` per-concern house style. Ungated — no consent gate, no auth gate; theme is functional data per the consent-mode-mobile classification. Persists the **choice** (`'light' | 'dark' | 'system'`), never the resolved value. Absent → `'system'`. Web's `SyncThemeFromCookie` reconciliation component has no mobile analog — it exists only to flush a pre-consent choice into the cookie once consent lands, and mobile has no consent gate.

## Boot

`src/components/init/ThemeInit.tsx`, mounted unconditionally in `app/_layout.tsx` next to `ConsentInit`. Hydrates the stored choice → resolves against the OS → bridges into NativeWind during the boot overlay, before the portal Stack paints. Subscribes to the OS listener only while the choice is `'system'`. No change to the bootStore gate machine.

## Toggle UI

One 3-segment control: Sun (`light`) / Monitor (`system`) / Moon (`dark`), in `PortalConfigDialog.tsx`, replacing the prior binary flip toggle. Reads `theme` to highlight the active segment; writes via `setTheme(choice)`. **Icon-only, matching web** — no per-segment label translation keys (web's equivalent control carries none). Each segment's `accessibilityLabel` is the raw value string (`'light'`/`'system'`/`'dark'`), matching web's `aria-label={value}`. The section-label row key `portal.config.theme.toggle.label` (DIALOG namespace, pre-existing, seeded) is unchanged.

`DashboardSidebar.tsx`: the prior binary theme toggle is removed and replaced with a button (`MonitorCog` icon) that closes the sidebar and opens `PORTAL_CONFIG_DIALOG`. Theme is switched from one control; the sidebar routes to it. Keeps theme reachable from the dashboard with a single source of truth.

## Do not break — fall-through sites

1. `src/lib/theme.ts` — `NAV_THEME: Record<'light' | 'dark', Theme>`, indexed at `app/_layout.tsx:77`. Fed the resolved value only; a raw `'system'` index returns `undefined`. The `?? 'light'` guards null, not the string `'system'`.
2. The ~20 binary `colorScheme === 'dark'` consumers — safe as long as they read NativeWind's resolved value. Keep them on the resolved scheme.

## No backend involvement

No translation keys, no seed, no DTO, no wire. An earlier implementation pass invented three `portal.config.theme.option.*` keys; they were removed (session `-3`) because web has no per-segment label keys and parity requires none.

## Out of scope

Web changes (web already ships tri-state). The ~20 binary consumers, `theme.ts`, `global.css`, `tailwind.config.js` — all unchanged; they keep working because the store feeds NativeWind only a resolved binary value. The shared web+mobile raw-`aria-label`/`accessibilityLabel` a11y gap (logged separately in issues.md).

## Definition of done

- Choice/resolved store with `Appearance`-backed `'system'` resolution; live OS listener active only while `theme === 'system'`.
- Ungated AsyncStorage persistence of the choice; boot hydrate/resolve/apply; default `'system'`.
- `colorScheme.set` bridge feeds the resolved value to NativeWind on every change.
- Icon-only 3-way control in PortalConfigDialog; sidebar toggle replaced by a config opener.
- No new translation keys; no backend work.
- tsc clean, lint at/under the 84 baseline (82), tests pass (334).
- On-device Ψ deferred; ships `shipped (code)` / `verifying`.

## Platform adoption

Mobile-only feature (web already has it). No further platform adoption.

## Session log

- `oglasino-expo-expo-system-theme-1` — read-only current-state audit.
- `oglasino-expo-expo-system-theme-2` — implementation (store, storage, ThemeInit, control, sidebar opener). Note: this session ran before the spec existed and invented three theme-option keys, corrected in `-3`.
- `oglasino-expo-expo-system-theme-3` — removed the three invented keys; icon-only parity with web; raw-value a11y label.
