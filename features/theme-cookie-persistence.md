# Theme cookie persistence

**Slug:** `theme-cookie-persistence`
**Branch:** `feature/theme-cookie-persistence` (off `dev`)
**Repos affected:** `oglasino-web` only
**Status:** `shipped`
**Authoritative source for engineers:** this file

---

## 1. Scope

Replace `next-themes` with a minimal cookie-based theme system that persists user theme preference through `globalCookie.theme`, gated on preference consent. Adds a `'system'` option to the toggle UI.

**In scope:**

- New Zustand store `useThemeStore` (cookie-backed, consent-gated, exposes `theme` and `resolvedTheme`).
- Inline pre-paint `<script>` in `<head>` that sets the `.dark` class on `<html>` before paint, read from `globalCookie.theme` with `prefers-color-scheme` fallback.
- Tri-state toggle UI (light / system / dark) in `PortalConfigDialog`.
- Three consumer migrations (`ToggleButton`, `OglasinoIcon`, `sonner`).
- Cookie-decline auto-clearance (inherits from `useConsentStore.setConsent` — zero new code).
- Reconciliation component `SyncThemeFromCookie` mirroring `SyncCardSizeFromCookie`'s in-memory-wins-on-grant semantics.
- System-preference listener component `SyncThemeFromSystem` updating `resolvedTheme` when `theme === 'system'` and the OS color scheme changes.
- Removal of `next-themes` dependency.

**Out of scope:**

- Mobile (`oglasino-expo`). Mobile has its own theme story (React Native `Appearance` API + NativeWind). If mobile mirrors web later, it's a separate Expo chat.
- Backend changes. No backend involvement.
- Router / Firestore Rules / docs reorganization.

---

## 2. Locked decisions

### D1 — `'system'` is included

`GlobalCookie.theme` widens from `'light' | 'dark'` to `'light' | 'dark' | 'system'`. Default when cookie is absent: `'system'`. The pre-paint script resolves `'system'` to a concrete `'light'` / `'dark'` via `prefers-color-scheme`.

### D2 — Toggle UI shape: three-icon segmented control

Three buttons in a row inside `PortalConfigDialog`: sun (light) / monitor (system) / moon (dark). Click to select. The currently-active button gets a visual indicator (background or border). Mirrors patterns used by other theme switchers in the wild; high discoverability.

### D3 — Store exposes both `theme` and `resolvedTheme`

`theme: 'light' | 'dark' | 'system'` — the user's stored preference. Drives toggle highlighting.

`resolvedTheme: 'light' | 'dark'` — what's actually on `<html>`. Drives any binary visual decision (logo variant, sonner theme).

When `theme === 'system'`, `resolvedTheme` tracks `prefers-color-scheme`. When `theme` is concrete, `resolvedTheme` equals `theme`.

### D4 — Store location and shape

File: `src/lib/store/useTheme.ts`.

```ts
type Theme = 'light' | 'dark' | 'system';
type ResolvedTheme = 'light' | 'dark';

interface ThemeStore {
  theme: Theme;
  resolvedTheme: ResolvedTheme;
  setTheme: (theme: Theme) => void;
  _setResolvedTheme: (resolved: ResolvedTheme) => void; // internal, called by SyncThemeFromSystem
}
```

`setTheme`:

1. Updates `theme` in store unconditionally.
2. Recomputes `resolvedTheme`:
   - If `theme === 'light'` or `'dark'`, `resolvedTheme = theme`.
   - If `theme === 'system'`, read `window.matchMedia('(prefers-color-scheme: dark)').matches` and set accordingly.
3. Applies `.dark` class on `document.documentElement` synchronously based on the new `resolvedTheme`.
4. Checks `isPreferenceConsentGranted()`. If granted, calls `updateGlobalCookie('theme', theme)`.

Mirrors `useCardSizeStore.setSize` exactly: in-memory always, cookie on consent only.

### D5 — System-preference listener

File: `src/components/client/SyncThemeFromSystem.tsx`.

Subscribes to `useThemeStore((s) => s.theme === 'system')`. When `true`:

- Registers a `matchMedia('(prefers-color-scheme: dark)')` change listener.
- On change, calls `useThemeStore.getState()._setResolvedTheme(newResolved)` and toggles `.dark` on `<html>`.
- Cleans up the listener when `theme` is no longer `'system'` (effect cleanup).

When `false`: no listener active.

### D6 — Cookie reconciliation component

File: `src/components/client/SyncThemeFromCookie.tsx`. Mirrors `SyncCardSizeFromCookie`:

- Subscribes to `useConsentStore((s) => s.consent?.preference === 'granted')`.
- On consent-grant transition (or on initial mount if already granted):
  - Read in-memory `theme` from `useThemeStore.getState()`.
  - Read cookie `theme` from `getGlobalCookie()?.theme`.
  - **In-memory wins:** if in-memory `theme !== 'system'` (user toggled before granting consent), write it to cookie via `updateGlobalCookie('theme', inMemory)`.
  - **Cookie wins:** if in-memory `theme === 'system'` (default — user hasn't toggled) but cookie has a value, call `setTheme(cookieValue)`.
- Mounted alongside `SyncCardSizeFromCookie` in `AppInit.tsx`.

The "in-memory wins" trigger is `theme !== 'system'` because `'system'` is the default state and any non-default value means the user clicked.

### D7 — SSR pre-paint snippet

File: `src/lib/theme/ssr.ts` (sibling to `src/lib/consent/ssr.ts`).

Helper signature: `generateThemeSnippet(theme: Theme | undefined): string`.

Snippet template:

```js
(function() {
  try {
    var t = THEME_FROM_COOKIE_OR_UNDEFINED;
    var resolved;
    if (t === 'dark') resolved = 'dark';
    else if (t === 'light') resolved = 'light';
    else resolved = window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light';
    if (resolved === 'dark') document.documentElement.classList.add('dark');
  } catch (e) {}
})();
```

Built server-side: `readGlobalCookieForSsr(cookieStore).theme` → sanitize against the valid-theme `Set` → interpolate → render as `<script dangerouslySetInnerHTML={{ __html: themeSnippet }} />` in `<head>` at `app/layout.tsx`, sibling to the existing consent snippet.

Sanitization defends against the interpolated value being anything other than `'light'`, `'dark'`, `'system'`, or `undefined`. Mirrors the Consent Mode v2 `sanitizeForSnippet` discipline.

The snippet is idempotent: `classList.add('dark')` on an element that already has the class is a no-op. This matters for the SSR-rendered-class case below.

### D8 — Server-side class rendering on `<html>`

When `globalCookie.theme === 'dark'`, `app/layout.tsx` renders `<html class="... dark ...">` server-side. For `'light'`, `'system'`, or absent cookie, no class is added (light is the Tailwind default; `'system'` and absent are deferred to the client snippet).

This eliminates hydration mismatch on the cookie-concrete branches. The snippet handles `'system'` and absent-cookie cases on the client, where the resulting class change relative to server-rendered HTML is suppressed by `suppressHydrationWarning` (see §8).

### D9 — Removal scope

**Delete:**

- `src/components/providers/ThemeProvider.tsx`
- `next-themes` from `package.json` and `package-lock.json`
- The `<ThemeProvider>{children}</ThemeProvider>` wrapper at `app/layout.tsx`

**Rewrite:**

- `src/components/client/buttons/ToggleButton.tsx` — three-state segmented control, reads/writes `useThemeStore`. Drops the `mounted` guard, `WithTooltip` wrapper, translation dependency.
- `src/components/icons/OglasinoIcon.tsx` — `useTheme` → `useThemeStore`, reads `resolvedTheme`. Effect deps updated to track `resolvedTheme`. Incidentally fixes the dependency bug surfaced in audit observation 2.
- `src/components/shadcn/ui/sonner.tsx` — `useTheme` → `useThemeStore`, passes `theme` (Sonner accepts `'light' | 'dark' | 'system'`).

**Add:**

- `src/lib/store/useTheme.ts`
- `src/components/client/SyncThemeFromCookie.tsx`
- `src/components/client/SyncThemeFromSystem.tsx`
- `src/lib/theme/ssr.ts`
- The `<script>` block in `app/layout.tsx`'s `<head>`
- Server-side class resolution in `app/layout.tsx`'s `<html>`.

**Widen:**

- `src/lib/types/cookie/GlobalCookie.ts` — `theme` field type widens from `'light' | 'dark'` to `'light' | 'dark' | 'system'`.

---

## 3. Reconciliation semantics

Per the 2026-05-22 cookies-closing decision, theme reconciliation is **in-memory-wins-on-grant** (mirrors card-size).

| Scenario | Behavior |
| --- | --- |
| First visit, consent not yet granted | `theme = 'system'`, `resolvedTheme` follows OS, no cookie write |
| User toggles to `'dark'` before granting consent | `theme = 'dark'` in memory, `.dark` class applied, no cookie write |
| User grants consent while in-memory is `'dark'` | `'dark'` written to cookie (in-memory wins) |
| Returning visitor with `cookie.theme = 'dark'` and consent granted | Server renders `<html class="dark">`; cookie value reads into store on `SyncThemeFromCookie` mount |
| User declines consent | `clearGlobalCookie()` nukes the whole cookie including `theme`. In-memory state unchanged. Next session: `theme = 'system'` again. |
| User has `theme === 'system'`, OS toggles dark | `resolvedTheme` updates via `SyncThemeFromSystem`, `.dark` class toggles |
| User explicitly picks `'system'` after having had `'dark'` | `setTheme('system')` resolves to current OS preference, cookie write `'system'` (if consent granted) |

---

## 4. Definition of done

The feature ships when all of the following hold:

1. `next-themes` is absent from `package.json` and `package-lock.json`.
2. `globalCookie.theme` is populated on user toggle, gated on preference consent.
3. Theme persists across page loads via cookie (verified with DevTools: clear `localStorage`, reload, theme survives).
4. SSR sets the correct `.dark` class on `<html>` before paint (verified: no FOUC, even on hard refresh, even with cold cache).
5. No React hydration warnings on any reload, for any of the four cookie states (concrete dark, concrete light, system, absent).
6. Cookie is cleared on consent decline (verified: toggle to dark, decline consent in the banner, reload — cookie is gone, theme back to default).
7. `'system'` option is selectable in `PortalConfigDialog`'s tri-state toggle, and OS-level dark mode toggle is reflected live without page reload.
8. `OglasinoIcon` updates correctly on OS dark-mode toggle when `theme === 'system'` (closes audit observation 2).
9. `sonner` toasts render with the correct theme.
10. `npm run lint`, `npx tsc --noEmit`, and `npm test` all green.

---

## 5. Items deliberately dropped

- **`disableTransitionOnChange` prop.** `next-themes` had it; Path A drops it. The audit confirmed no theme-dependent CSS transitions exist in the codebase — the prop was a no-op. Removed deliberately, not by oversight.
- **`mounted`-guard in `ToggleButton`.** No longer needed: server and client agree on initial state via the SSR cookie read for concrete cookie values; the segmented control renders the correct active state from first paint.

---

## 6. Test surfaces

`npm test` passes unchanged (the existing 229 tests don't cover theme directly). New tests are not required for this feature — the verification is manual + DevTools. The store and the snippet helper are unit-test-friendly targets if a future engineer wants to add coverage.

The SSR snippet and the hydration behavior are verified by manual hard-refresh inspection across the four cookie states.

---

## 7. Engineering brief sequence (Phase 5)

Shipped via two briefs:

- **Brief 1 (single-shot):** entire feature foundation — store, two sync components, snippet helper, layout wiring, three consumer migrations, removals, type widening. Landed 2026-05-27. Session `oglasino-web-theme-cookie-persistence-2`.
- **Brief 1b (hydration fix):** manual smoke surfaced a React hydration warning on `<html>` not anticipated by the original spec's risk assessment. Fix added server-side class rendering (D8) and restored `suppressHydrationWarning`. Two lines of code. Session `oglasino-web-theme-cookie-persistence-3`.

Net engineering footprint: ~150 LOC across both briefs.

---

## 8. Risks (post-shipping notes)

- **`<html>` hydration mismatch on the system / absent-cookie branches.** Anticipated and handled by `suppressHydrationWarning` on `<html>`. The server cannot resolve `prefers-color-scheme` (no media query exists server-side), so when the cookie is `'system'` or absent, the client snippet may add the `dark` class that the server didn't render. This is the documented use case for `suppressHydrationWarning`. The attribute scopes only to `<html>`'s own attributes; it does not suppress warnings for the rest of the tree.

  Earlier drafts of this spec incorrectly framed `<html>` as outside React's hydration tree. In Next.js App Router, `<html>` IS rendered by React (it's `app/layout.tsx`'s root element). Brief 1b corrected this — the attribute was restored after Brief 1 dropped it on the wrong reasoning. Recording here so the next feature touching `<html>` doesn't repeat the analysis error.

- **The two sync components racing.** `SyncThemeFromCookie` writes to the store on consent grant; `SyncThemeFromSystem` writes to the store on OS preference change. Both update `resolvedTheme`. If they fire simultaneously, last-write-wins — which is correct, because both writes converge on the same value. No real race.

- **Consent transition while `theme === 'system'`.** User has `'system'` in memory, grants consent → `SyncThemeFromCookie` sees in-memory is `'system'`, falls through to cookie-wins branch, finds no cookie value, does nothing. Correct: `'system'` stays unsaved (it's the default). If the user later picks light or dark, that writes.

---

## 9. Verification checklist at chat close

- [x] All ten items in Definition of Done (§4) pass.
- [ ] Engineer session summaries (`-2` and `-3`) archived to `oglasino-docs/sessions/`.
- [ ] Closing entry in `decisions.md` summarizing what shipped (including the Brief 1b hydration fix).
- [ ] `state.md` feature row reflects `shipped` status.
- [ ] `state.md` Risk Watch entry "Theme persistence remains on `next-themes` localStorage" closed.
- [ ] `oglasino-docs/.agent/handoffs/theme-path-a.md` deleted.
- [ ] `issues.md` 2026-05-22 entry "Theme switcher should support 'system' option" flipped to `fixed`.
