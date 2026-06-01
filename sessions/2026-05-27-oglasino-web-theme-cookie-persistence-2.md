# Session summary

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-27
**Task:** Replace `next-themes` with a cookie-based theme system: new Zustand store, SSR pre-paint snippet, two sync components, tri-state toggle UI, three consumer migrations, dependency removal.

## Implemented

- Widened `GlobalCookie.theme` type from `'light' | 'dark'` to `'light' | 'dark' | 'system'`.
- Created `src/lib/store/useTheme.ts` — Zustand store exposing `theme`, `resolvedTheme`, `setTheme`, and `_setResolvedTheme`. `setTheme` updates in-memory state unconditionally, applies `.dark` class on `<html>` synchronously, and writes to cookie only when preference consent is granted. Mirrors `useCardSizeStore.setSize` pattern exactly.
- Created `src/lib/theme/ssr.ts` — `generateThemeSnippet(theme)` function that produces an inline IIFE setting the `.dark` class on `<html>` before paint. Sanitizes interpolated value against a `Set` of valid themes; unknown values fall through to `prefers-color-scheme` resolution. Placed as sibling to `src/lib/consent/ssr.ts`.
- Wired the theme snippet in `app/layout.tsx` `<head>` as sibling to the consent snippet. Reads `globalCookie.theme` server-side via `readGlobalCookieForSsr(cookieStore)`.
- Created `src/components/client/SyncThemeFromCookie.tsx` — reconciliation component mirroring `SyncCardSizeFromCookie`. Subscribes to consent grant; in-memory wins when theme is non-default (`!== 'system'`); cookie wins when in-memory is default.
- Created `src/components/client/SyncThemeFromSystem.tsx` — OS preference listener. Active only when `theme === 'system'`. Registers `matchMedia` change listener, calls `_setResolvedTheme` on OS toggle.
- Mounted both sync components in `AppInit.tsx` alongside `SyncCardSizeFromCookie`.
- Migrated `OglasinoIcon.tsx` — `useTheme` from `next-themes` → `useThemeStore`, reads `resolvedTheme` via selector. Effect deps fixed from `[theme, setLogoPath]` to `[resolvedTheme, reverted, extended]` (closes audit observation 2).
- Migrated `sonner.tsx` — `useTheme` from `next-themes` → `useThemeStore`, reads `theme` via selector. Sonner accepts `'light' | 'dark' | 'system'` natively.
- Rewrote `ToggleButton.tsx` — three-icon segmented control (Sun/Monitor/Moon) with active-state highlighting via `bg-accent`. Dropped `mounted` guard, `WithTooltip` wrapper, translation dependency, and `IconButton` wrapper. Reads/writes `useThemeStore`.
- Deleted `src/components/providers/ThemeProvider.tsx`.
- Removed `<ThemeProvider>` wrapper from `app/layout.tsx`.
- Removed `suppressHydrationWarning` from `<html>` in `app/layout.tsx`.
- Uninstalled `next-themes` from `package.json` and `package-lock.json`.
- Added `cookie.theme` to the AppInit self-heal check so stale theme values in cookies without consent are cleaned up alongside other preference fields.

## Files touched

- `app/layout.tsx` (+6 / -5)
- `src/lib/types/cookie/GlobalCookie.ts` (+1 / -1)
- `src/lib/store/useTheme.ts` (+49 / -0) — **new**
- `src/lib/theme/ssr.ts` (+17 / -0) — **new**
- `src/components/client/SyncThemeFromCookie.tsx` (+25 / -0) — **new**
- `src/components/client/SyncThemeFromSystem.tsx` (+27 / -0) — **new**
- `src/components/client/buttons/ToggleButton.tsx` (+28 / -37)
- `src/components/client/initializers/AppInit.tsx` (+5 / -0)
- `src/components/icons/OglasinoIcon.tsx` (+4 / -6)
- `src/components/shadcn/ui/sonner.tsx` (+2 / -2)
- `src/components/providers/ThemeProvider.tsx` (0 / -15) — **deleted**
- `package.json` (+0 / -1)
- `package-lock.json` (+0 / -11)

## Tests

- Ran: `npx tsc --noEmit` — clean
- Ran: `npm run lint` — 0 errors, 147 warnings (all pre-existing)
- Ran: `npm test` — 229 passed, 0 failed
- New tests added: none (manual verification per spec §6; store and snippet are unit-test-friendly targets if desired)

## Cleanup performed

- Deleted `ThemeProvider.tsx` (obsoleted by the new store + snippet).
- Removed `next-themes` dependency.
- Removed `suppressHydrationWarning` from `<html>` (was required only by `next-themes`).
- Removed `mounted` guard, `WithTooltip` wrapper, translation dependency from `ToggleButton.tsx`.
- Fixed `OglasinoIcon.tsx` effect deps (was `[theme, setLogoPath]`, now `[resolvedTheme, reverted, extended]`).

## Config-file impact

- conventions.md: no change
- decisions.md: closing entry drafted below in "For Mastermind"
- state.md: no change (Mastermind handles status flip)
- issues.md: 2026-05-22 "Theme switcher should support 'system' option" flips to `fixed` — draft below

## Obsoleted by this session

- `src/components/providers/ThemeProvider.tsx` — deleted in this session.
- `next-themes` dependency — uninstalled in this session.
- `suppressHydrationWarning` on `<html>` — removed in this session.
- `ToggleButton.tsx` `mounted` guard and `WithTooltip` wrapper — removed in this session.
- `OglasinoIcon.tsx` stale deps `[theme, setLogoPath]` — replaced with correct deps `[resolvedTheme, reverted, extended]`.
- `localStorage["theme"]` entries from prior `next-themes` usage — harmless cruft from prior visits; no new localStorage writes occur.

## Conventions check

- Part 4 (cleanliness): confirmed — no commented-out code, no unused imports, no console.log, no TODO/FIXME. tsc/lint/test all green.
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): confirmed — audit observations 2 and 5 fixed in this session as specified by the brief. No new out-of-scope observations.
- Part 6 (translations): N/A this session — no translation keys added or modified.
- Other parts touched: Part 11 (trust boundaries) — confirmed N/A per brief; theme is a UI preference not used in moderation/authorization/state-transition decisions.

## Known gaps / TODOs

- None.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity):
    - `useThemeStore` (Zustand store) — mirrors the established `useCardSizeStore` pattern; required for consent-gated cookie persistence and reactive consumer updates.
    - `generateThemeSnippet` (SSR helper) — mirrors `sanitizeForSnippet`/`readConsentForSsr` pattern; required to eliminate FOUC by setting `.dark` class before paint.
    - `SyncThemeFromCookie` (reconciliation component) — mirrors `SyncCardSizeFromCookie`; required for in-memory-wins-on-grant semantics on consent transition.
    - `SyncThemeFromSystem` (OS listener component) — required for live `resolvedTheme` updates when `theme === 'system'` and OS preference changes.
  - Considered and rejected:
    - A shared `resolveTheme` utility exported from the store for the snippet — rejected; the snippet runs pre-React and needs inlined logic (it's a raw `<script>` string), while the store's `resolveTheme` uses `window.matchMedia`. Two independent resolution sites, each minimal.
    - `sanitizeThemeForSnippet` as a separately-exported function — kept as a private helper inside `ssr.ts`; one caller, no reuse.
    - Extracting a `ThemeOption` component from the `ToggleButton` map — rejected; three items, six lines each, no foreseeable reuse.
  - Simplified or removed:
    - `next-themes` dependency (entire library).
    - `ThemeProvider.tsx` wrapper component.
    - `suppressHydrationWarning` on `<html>` (no longer needed since SSR snippet and client agree on initial state).
    - `mounted` guard in `ToggleButton` (no longer needed for same reason).
    - `disableTransitionOnChange` prop (was a no-op per audit observation 1 — no theme-specific CSS transitions exist).
    - `WithTooltip` and `IconButton` wrappers removed from `ToggleButton` — replaced with plain `<button>` elements in the segmented control.

- **Audit observation 2 (OglasinoIcon deps) — closed.** Fixed by updating effect deps to `[resolvedTheme, reverted, extended]`. Logo now updates correctly when OS dark mode toggles while `theme === 'system'`.

- **Audit observation 5 (ToggleButton mounted-guard) — closed.** Dropped in the rewrite.

- **Open question resolutions (from spec §7):**
  1. Snippet helper placed at `src/lib/theme/ssr.ts` — sibling directory to `src/lib/consent/ssr.ts`. Both are SSR-only snippet helpers.
  2. Sync components mounted in `AppInit.tsx` alongside `SyncCardSizeFromCookie`.
  3. `setTheme` computes `resolvedTheme` inline via a private `resolveTheme` helper rather than calling `_setResolvedTheme`. Reads cleaner as a single atomic update.

### Config-file drafts

**`decisions.md` — new entry:**

```markdown
## 2026-05-27 — Theme cookie persistence shipped (replaces next-themes)

Replaced `next-themes` with a minimal cookie-based theme system in `oglasino-web`. The system mirrors the established card-size cookie pattern (Zustand store + consent-gated cookie writes + in-memory-wins-on-grant reconciliation + SSR pre-paint snippet).

**What shipped:**

- New Zustand store `useThemeStore` (`src/lib/store/useTheme.ts`) exposing `theme: 'light' | 'dark' | 'system'` and `resolvedTheme: 'light' | 'dark'`. `setTheme` updates in-memory state unconditionally and writes to `globalCookie.theme` only when preference consent is granted. Mirrors `useCardSizeStore.setSize` exactly.
- SSR pre-paint `<script>` in `<head>` (`src/lib/theme/ssr.ts`, wired at `app/layout.tsx`) that reads `globalCookie.theme` server-side and sets `.dark` on `<html>` before paint. Eliminates the FOUC that `next-themes`' body-rendered script left open.
- `SyncThemeFromCookie` — reconciliation on consent-grant transition. In-memory wins when non-default; cookie wins when in-memory is default (`'system'`).
- `SyncThemeFromSystem` — OS `prefers-color-scheme` listener active only when `theme === 'system'`.
- Tri-state toggle (Sun/Monitor/Moon) in `PortalConfigDialog`.
- Three consumer migrations (`OglasinoIcon`, `sonner`, `ToggleButton`).
- `next-themes` uninstalled; `ThemeProvider.tsx` deleted; `suppressHydrationWarning` removed from `<html>`.

`GlobalCookie.theme` widened from `'light' | 'dark'` to `'light' | 'dark' | 'system'`. Default when absent: `'system'` (resolved via `prefers-color-scheme`).

**Alternatives considered and rejected:**

- **B1-A (next-themes custom storage adapter):** unavailable in v0.4.6 — `ThemeProviderProps` exposes no pluggable storage mechanism. Confirmed by the read-only audit.
- **B1-B (next-themes mirror pattern):** `setTheme` writes to localStorage unconditionally — no way to gate on consent. Confirmed by the read-only audit.
- **Keeping `disableTransitionOnChange`:** no theme-specific CSS transitions exist (audit observation 1); the prop was a no-op.
- **Keeping `suppressHydrationWarning`:** the SSR snippet ensures server and client agree from first paint; the attribute is unnecessary.
```

**`issues.md` — flip status on 2026-05-22 entry:**

Change status of "Theme switcher should support 'system' option" from `open` to `fixed`. Add note: `Fixed (2026-05-27, session oglasino-web-theme-cookie-persistence-2). Tri-state toggle (light/system/dark) shipped in PortalConfigDialog. 'system' option resolves to OS preference via prefers-color-scheme and updates live via matchMedia listener.`
