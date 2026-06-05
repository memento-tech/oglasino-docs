# Web Reference Audit — expo-system-theme

**Repo:** oglasino-web
**Branch:** dev (no switch, no edits — read-only)
**Date:** 2026-06-01
**Purpose:** Inventory web's system-theme implementation precisely so the expo
chat can mirror the pattern on mobile. Web does **not** change. Reference
material, not a parity-change audit.

Every file cited below is backed by raw `cat -n` / `grep` shell output proving
it exists and showing the key lines (TOOL-FABRICATION GUARD).

---

## 0. File map

| Role | Path |
| --- | --- |
| Theme store (Zustand) | `src/lib/store/useTheme.ts` |
| Live OS-change listener | `src/components/client/SyncThemeFromSystem.tsx` |
| Cookie hydration / reconciliation | `src/components/client/SyncThemeFromCookie.tsx` |
| SSR pre-paint snippet generator | `src/lib/theme/ssr.ts` |
| SSR `<html>` class + snippet mount | `app/layout.tsx` (lines 58–69) |
| Cookie read/write primitives | `src/lib/service/oglasinoCookies.ts` |
| Consent gate | `src/lib/consent/gating.ts` |
| Theme type | `src/lib/types/cookie/GlobalCookie.ts` |
| Tri-state toggle UI | `src/components/client/buttons/ToggleButton.tsx` |
| Toggle mount point | `src/components/popups/dialogs/PortalConfigDialog.tsx` (line 151) |
| Initializer that mounts both Sync components | `src/components/client/initializers/AppInit.tsx` (lines 57–58) |
| `resolvedTheme` consumers | `src/components/icons/OglasinoIcon.tsx` (line 21) |

Locator grep proving the set is complete:

```
$ grep -rl "SyncTheme" src app
src/components/client/SyncThemeFromCookie.tsx
src/components/client/SyncThemeFromSystem.tsx
src/components/client/initializers/AppInit.tsx
$ grep -rl "resolvedTheme" src app
src/components/icons/OglasinoIcon.tsx
src/lib/store/useTheme.ts
$ grep -rl "setTheme" src app
src/components/client/SyncThemeFromCookie.tsx
src/components/client/buttons/ToggleButton.tsx
src/lib/store/useTheme.ts
$ grep -rn "generateThemeSnippet\|suppressHydrationWarning\|ThemeToggle" app src
app/layout.tsx:6:import { generateThemeSnippet } from '@/src/lib/theme/ssr';
app/layout.tsx:59:  const themeSnippet = generateThemeSnippet(globalCookie.theme);
app/layout.tsx:66:      suppressHydrationWarning>
src/components/popups/dialogs/PortalConfigDialog.tsx:22:import { ThemeToggle } from '../../client/buttons/ToggleButton';
src/components/popups/dialogs/PortalConfigDialog.tsx:151:            <ThemeToggle />
src/components/client/buttons/ToggleButton.tsx:13:export function ThemeToggle() {
src/lib/theme/ssr.ts:10:export function generateThemeSnippet(theme: Theme | undefined): string {
```

The `Theme` type itself (`src/lib/types/cookie/GlobalCookie.ts`):

```
     3	export type Lang = 'sr' | 'en' | 'ru' | 'cnr';
     4	
     5	export type Theme = 'light' | 'dark' | 'system';
```

So `theme` is a **three-state** value (`light | dark | system`); the store's
`resolvedTheme` is a **two-state** value (`light | dark`).

---

## 1. THEME STORE

**File:** `src/lib/store/useTheme.ts` — pasted in full.

```
     1	import { create } from 'zustand';
     2	import { isPreferenceConsentGranted } from '../consent/gating';
     3	import { updateGlobalCookie } from '../service/oglasinoCookies';
     4	import { Theme } from '../types/cookie/GlobalCookie';
     5	
     6	type ResolvedTheme = 'light' | 'dark';
     7	
     8	interface ThemeStore {
     9	  theme: Theme;
    10	  resolvedTheme: ResolvedTheme;
    11	  setTheme: (theme: Theme) => void;
    12	  _setResolvedTheme: (resolved: ResolvedTheme) => void;
    13	}
    14	
    15	function resolveTheme(theme: Theme): ResolvedTheme {
    16	  if (theme === 'light' || theme === 'dark') return theme;
    17	  if (typeof window !== 'undefined') {
    18	    return window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light';
    19	  }
    20	  return 'light';
    21	}
    22	
    23	function applyClass(resolved: ResolvedTheme) {
    24	  if (typeof document === 'undefined') return;
    25	  if (resolved === 'dark') {
    26	    document.documentElement.classList.add('dark');
    27	  } else {
    28	    document.documentElement.classList.remove('dark');
    29	  }
    30	}
    31	
    32	export const useThemeStore = create<ThemeStore>((set) => ({
    33	  theme: 'system',
    34	  resolvedTheme: 'light',
    35	  setTheme: (theme: Theme) => {
    36	    const resolved = resolveTheme(theme);
    37	    set({ theme, resolvedTheme: resolved });
    38	    applyClass(resolved);
    39	    if (typeof window !== 'undefined' && isPreferenceConsentGranted()) {
    40	      try {
    41	        updateGlobalCookie('theme', theme);
    42	      } catch {}
    43	    }
    44	  },
    45	  _setResolvedTheme: (resolved: ResolvedTheme) => {
    46	    set({ resolvedTheme: resolved });
    47	    applyClass(resolved);
    48	  },
    49	}));
    50	(EOF)
```

### State shape

- `theme: Theme` — the user's *choice*, one of `'light' | 'dark' | 'system'`. Initial value `'system'` (line 33).
- `resolvedTheme: ResolvedTheme` — the *effective* appearance, one of `'light' | 'dark'`. Initial value `'light'` (line 34).
- `setTheme(theme)` — public setter, called by the toggle and by cookie reconciliation.
- `_setResolvedTheme(resolved)` — internal setter (underscore-prefixed by convention), called only by the live OS listener. It updates `resolvedTheme` **without** touching `theme` and **without** writing the cookie.

There is nothing else in the store — no `isHydrated` flag, no listener handle. The OS listener lives in its own component (§2), not in the store.

### `setTheme` walked step by step (lines 35–44)

1. `const resolved = resolveTheme(theme)` — compute the two-state appearance from the three-state choice (see `resolveTheme` below).
2. `set({ theme, resolvedTheme: resolved })` — commit both the choice and the resolved value to the store in one update.
3. `applyClass(resolved)` — imperatively toggle the `.dark` class on `<html>` to match (lines 23–30).
4. Cookie write, **gated** (line 39): only if `typeof window !== 'undefined'` **and** `isPreferenceConsentGranted()` is true. The write is `updateGlobalCookie('theme', theme)` — it persists the *choice* (`'system'` included), not the resolved value. Wrapped in `try/catch {}` so a cookie failure never throws into the caller.

### How `resolvedTheme` is computed from `theme` + OS preference — `resolveTheme` (lines 15–21)

```
function resolveTheme(theme: Theme): ResolvedTheme {
  if (theme === 'light' || theme === 'dark') return theme;          // explicit choice wins
  if (typeof window !== 'undefined') {                              // theme === 'system'
    return window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light';
  }
  return 'light';                                                   // SSR / no window → light
}
```

- If `theme` is an explicit `'light'` or `'dark'`, that value *is* the resolved value — OS preference is ignored.
- If `theme` is `'system'`, the resolved value comes from the OS via `window.matchMedia('(prefers-color-scheme: dark)').matches` → `'dark'` when true, else `'light'`.
- On the server (no `window`), it falls back to `'light'`. This is why the store's initial `resolvedTheme` is `'light'` and why the SSR snippet (§4) exists to correct the first paint before React hydrates.

### `applyClass` (lines 23–30)

The single place the store touches the DOM: adds `.dark` to `document.documentElement` when resolved is `'dark'`, removes it otherwise. No-op when `document` is undefined (SSR).

---

## 2. SYSTEM-FOLLOWING (live OS changes)

`'system'` resolves to light/dark via **`window.matchMedia('(prefers-color-scheme: dark)').matches`** — used in two places: `resolveTheme` in the store (§1, line 18) for the synchronous compute, and the listener component below for the live subscription.

**Component confirmed to still exist — `SyncThemeFromSystem`** (was named as of 2026-05-27, still present):

```
$ grep -rl "SyncTheme" src app
src/components/client/SyncThemeFromCookie.tsx
src/components/client/SyncThemeFromSystem.tsx          ← present
src/components/client/initializers/AppInit.tsx
```

**File:** `src/components/client/SyncThemeFromSystem.tsx` — pasted in full.

```
     1	'use client';
     2	
     3	import { useThemeStore } from '@/src/lib/store/useTheme';
     4	import { useEffect } from 'react';
     5	
     6	export default function SyncThemeFromSystem() {
     7	  const isSystem = useThemeStore((s) => s.theme === 'system');
     8	
     9	  useEffect(() => {
    10	    if (!isSystem) return;
    11	
    12	    const mql = window.matchMedia('(prefers-color-scheme: dark)');
    13	
    14	    const handler = (e: MediaQueryListEvent) => {
    15	      const resolved = e.matches ? 'dark' : 'light';
    16	      useThemeStore.getState()._setResolvedTheme(resolved);
    17	    };
    18	
    19	    const resolved = mql.matches ? 'dark' : 'light';
    20	    useThemeStore.getState()._setResolvedTheme(resolved);
    21	
    22	    mql.addEventListener('change', handler);
    23	    return () => mql.removeEventListener('change', handler);
    24	  }, [isSystem]);
    25	
    26	  return null;
    27	}
    28	(EOF)
```

- **When active:** only while `theme === 'system'`. The effect subscribes to `s.theme === 'system'` as a boolean selector (line 7); the effect body early-returns when `!isSystem` (line 10). Switching the toggle away from `system` flips `isSystem` to false, re-runs the effect, the cleanup removes the listener, and the body returns without re-adding it. Switching back to `system` re-adds it.
- **What it does on mount (while system):** lines 19–20 immediately push the *current* OS value into `resolvedTheme` via `_setResolvedTheme` — a one-shot sync so the store reflects reality even if no `change` event ever fires.
- **What it does on an OS-appearance change:** the `change` handler (lines 14–17) maps `e.matches` → `'dark'`/`'light'` and calls `_setResolvedTheme(resolved)`, which updates the store and re-applies the `.dark` class. It does **not** call `setTheme`, so `theme` stays `'system'` and **no cookie is written** — live OS following never persists.
- **Cleanup:** `removeEventListener` on unmount or whenever `isSystem` changes (line 23).

Where it's mounted — `AppInit.tsx` (line 58):

```
    57	      <SyncThemeFromCookie />
    58	      <SyncThemeFromSystem />
```

---

## 3. PERSISTENCE + RECONCILIATION (web-only — catalogue of what NOT to port)

### The cookie write and what gates it

The only theme cookie write is in the store's `setTheme` (§1, lines 39–43):

```
    39	    if (typeof window !== 'undefined' && isPreferenceConsentGranted()) {
    40	      try {
    41	        updateGlobalCookie('theme', theme);
    42	      } catch {}
    43	    }
```

Gate = **preference consent granted**. `isPreferenceConsentGranted()` is the synchronous, non-React read in `src/lib/consent/gating.ts`:

```
    11	export function isPreferenceConsentGranted(): boolean {
    12	  try {
    13	    return useConsentStore.getState().consent?.preference === 'granted';
    14	  } catch {
    15	    return false;
    16	  }
    17	}
```

(Header comment, lines 3–10: "Absent consent … returns false — defaults are denied per Consent Mode v2.")

The cookie machinery — `src/lib/service/oglasinoCookies.ts`. The theme lives as one key inside a single shared `globalCookie` blob (`GLOBAL_COOKIE_NAME = 'globalCookie'`), alongside card sizes and lang:

```
     5	export const GLOBAL_COOKIE_NAME = 'globalCookie';
    22	export const updateGlobalCookie = <K extends keyof GlobalCookie>(
    23	  key: K,
    24	  value: GlobalCookie[K],
    25	  days = 365
    26	) => {
    27	  updateCookie(GLOBAL_COOKIE_NAME, key, value, days);
    28	};
    56	export const updateCookie = (cookieName: COOKIE_NAMES, key: string, value: any, days: number) => {
    57	  const cookie = getCookie(cookieName) || {};
    58	  const updated = { ...cookie, [key]: value };
    59	  setCookie(GLOBAL_COOKIE_NAME, updated, days);
    60	};
    43	export const setCookie = (cookieName: COOKIE_NAMES, data: any, days: number) => {
    44	  if (typeof document === 'undefined') return;
    45	  const expires = new Date(Date.now() + days * 864e5).toUTCString();
    46	  const value = encodeURIComponent(JSON.stringify(data));
    47	  document.cookie = `${cookieName}=${value}; expires=${expires}; path=/`;
    48	};
```

So the write is a 365-day, `path=/`, URL-encoded-JSON document cookie; theme is merged into the existing blob (read-modify-write), never its own cookie.

### Default-when-absent

- In-memory store default: `theme: 'system'`, `resolvedTheme: 'light'` (§1, lines 33–34).
- SSR default blob (`readGlobalCookieForSsr`, lines 62–77): when the cookie is absent it returns `DEFAULT_GLOBAL_COOKIE` (lines 12–16) which sets the three card sizes to `'small'` and leaves **`theme` and `lang` undefined**. An undefined `theme` flows into the SSR snippet as `undefined`, where the snippet treats it as "system" (§4).

```
    12	const DEFAULT_GLOBAL_COOKIE: GlobalCookie = {
    13	  portalCardSize: DEFAULT_CARD_SIZE,
    14	  ownerCardSize: DEFAULT_CARD_SIZE,
    15	  adminCardSize: DEFAULT_CARD_SIZE,
    16	};
    62	export function readGlobalCookieForSsr(cookieStore: CookieStore): GlobalCookie {
    63	  const raw = cookieStore.get(GLOBAL_COOKIE_NAME)?.value;
    64	  if (!raw) return DEFAULT_GLOBAL_COOKIE;
    ...
    72	      lang: parsed.lang,
    73	      theme: parsed.theme,
```

### Hydration / reconciliation component — `SyncThemeFromCookie` (confirmed present)

**File:** `src/components/client/SyncThemeFromCookie.tsx` — pasted in full.

```
     1	'use client';
     2	
     3	import { getGlobalCookie, updateGlobalCookie } from '@/src/lib/service/oglasinoCookies';
     4	import { useThemeStore } from '@/src/lib/store/useTheme';
     5	import { useConsentStore } from '@/src/lib/store/useConsentStore';
     6	import { useEffect } from 'react';
     7	
     8	export default function SyncThemeFromCookie() {
     9	  const setTheme = useThemeStore((s) => s.setTheme);
    10	  const preferenceGranted = useConsentStore((s) => s.consent?.preference === 'granted');
    11	
    12	  useEffect(() => {
    13	    if (!preferenceGranted) return;
    14	    const inMemory = useThemeStore.getState().theme;
    15	    const cookieTheme = getGlobalCookie()?.theme;
    16	
    17	    if (inMemory !== 'system') {
    18	      updateGlobalCookie('theme', inMemory);
    19	    } else if (cookieTheme) {
    20	      setTheme(cookieTheme);
    21	    }
    22	  }, [setTheme, preferenceGranted]);
    23	
    24	  return null;
    25	}
    26	(EOF)
```

- **When active / what gates it:** runs its effect only when `preferenceGranted` is true (line 13). It re-runs when `preferenceGranted` flips — i.e. **on consent-grant** (the user accepting preference cookies in the banner flips the consent store, re-running this effect).
- **What it does on consent-grant** (lines 14–21):
  - Reads the current in-memory `theme` and the persisted `cookieTheme`.
  - If the user already changed the theme this session (`inMemory !== 'system'`, line 17): write that in-memory choice into the cookie — the choice made *before* consent existed is now persisted (forward-flush).
  - Else, if the in-memory value is still the default `'system'` and a `cookieTheme` exists (line 19): adopt the persisted value via `setTheme(cookieTheme)` — restores a prior session's choice.
  - This ordering deliberately lets an active-session choice win over a stale cookie.

### Related self-heal (adjacent, not theme-specific) — `AppInit.tsx` lines 32–45

When preference consent is **not** granted but a `globalCookie` still carries `theme`/`lang`/card-size data, `AppInit` clears the whole blob (`clearGlobalCookie()`), so no preference data lingers without consent. Theme is one of the keys it checks (line 38). Catalogue-only; not part of the theme machine proper.

---

## 4. SSR PIECES (web-only — catalogue of what NOT to port)

### (a) The pre-paint `<script>` generator — `src/lib/theme/ssr.ts`

```
     1	import { Theme } from '../types/cookie/GlobalCookie';
     2	
     3	const VALID_THEMES: ReadonlySet<string> = new Set(['light', 'dark', 'system']);
     4	
     5	function sanitizeTheme(theme: Theme | undefined): string | undefined {
     6	  if (theme && VALID_THEMES.has(theme)) return theme;
     7	  return undefined;
     8	}
     9	
    10	export function generateThemeSnippet(theme: Theme | undefined): string {
    11	  const sanitized = sanitizeTheme(theme);
    12	  const themeValue = sanitized ? `'${sanitized}'` : 'undefined';
    13	
    14	  return `(function(){try{var t=${themeValue};var r;if(t==='dark')r='dark';else if(t==='light')r='light';else r=window.matchMedia('(prefers-color-scheme: dark)').matches?'dark':'light';if(r==='dark')document.documentElement.classList.add('dark');}catch(e){}})();`;
    15	}
```

An IIFE string injected before paint. It re-implements `resolveTheme` in vanilla JS: reads the server-sanitized cookie theme, and if it isn't an explicit `dark`/`light` (i.e. `system` or undefined) it consults `matchMedia` *on the client at first paint* and adds `.dark` if needed. Sanitization (lines 5–8) guards against a tampered cookie injecting script (only `light|dark|system` pass; anything else → `undefined`). **SSR-bound:** its sole purpose is to run before React hydrates to avoid a flash; it has no role after hydration (the store's `applyClass` takes over). Confirmed used only in `app/layout.tsx:59,69`.

### (b) The server-side `.dark` class on `<html>` — `app/layout.tsx` lines 58–66

```
    58	  const globalCookie = readGlobalCookieForSsr(cookieStore);
    59	  const themeSnippet = generateThemeSnippet(globalCookie.theme);
    60	  const ssrHtmlClass = globalCookie.theme === 'dark' ? 'dark' : '';
    61	
    62	  return (
    63	    <html
    64	      className={`${inter.variable} ${ssrHtmlClass} overscroll-none font-sans antialiased`}
    65	      lang={htmlLang}
    66	      suppressHydrationWarning>
```

`ssrHtmlClass` is `'dark'` **only** when the cookie theme is an explicit `'dark'` (line 60). For `'system'` or absent, the server emits no `.dark` class and the snippet (run at first paint, item a) is what adds it — because the server cannot know the device's OS preference. **SSR-bound.**

### (c) The snippet mount + `suppressHydrationWarning`

```
    67	      <head>
    68	        <script dangerouslySetInnerHTML={{ __html: consentDefaultSnippet }} />
    69	        <script dangerouslySetInnerHTML={{ __html: themeSnippet }} />
```

- `themeSnippet` is injected as a raw `<script>` in `<head>` (line 69), so it executes before body paint.
- `suppressHydrationWarning` on `<html>` (line 66) tells React not to warn about the server/client `className` mismatch — expected, because the pre-paint snippet mutates `documentElement.classList` before React reconciles. **SSR/hydration-bound.**

All three are strictly web/DOM/SSR concerns.

---

## 5. APPLY MECHANISM

The resolved theme drives styling through a single lever: the **`.dark` class on `<html>`** (`document.documentElement`). Tailwind's `dark:` variants are keyed off that class, so adding/removing it flips every `dark:`-prefixed utility across the app at once. The class is set in three coordinated places — the SSR snippet at first paint (§4a), the server-rendered `ssrHtmlClass` for explicit-dark cookies (§4b), and the store's `applyClass` for every runtime change (§1, lines 23–30) — but they all converge on the same one toggle. Additionally, a few components read `resolvedTheme` directly for non-CSS rendering decisions (e.g. `OglasinoIcon.tsx:21` picks a light/dark icon variant), confirmed by:

```
$ grep -rl "resolvedTheme" src app
src/components/icons/OglasinoIcon.tsx
src/lib/store/useTheme.ts
```

This contrasts with mobile, whose apply mechanism the expo audit surfaces separately (RN has no `<html>` class / no Tailwind `dark:` cascade).

---

## 6. TOGGLE UI

**Tri-state control confirmed (Sun / Monitor / Moon)** — `src/components/client/buttons/ToggleButton.tsx`, exported as `ThemeToggle`, pasted in full:

```
     1	'use client';
     2	
     3	import { useThemeStore } from '@/src/lib/store/useTheme';
     4	import { Theme } from '@/src/lib/types/cookie/GlobalCookie';
     5	import { Monitor, Moon, Sun } from 'lucide-react';
     6	
     7	const OPTIONS: { value: Theme; icon: React.ReactNode }[] = [
     8	  { value: 'light', icon: <Sun className="size-4" /> },
     9	  { value: 'system', icon: <Monitor className="size-4" /> },
    10	  { value: 'dark', icon: <Moon className="size-4" /> },
    11	];
    12	
    13	export function ThemeToggle() {
    14	  const theme = useThemeStore((s) => s.theme);
    15	  const setTheme = useThemeStore((s) => s.setTheme);
    16	
    17	  return (
    18	    <div className="flex gap-1 rounded-md border p-0.5">
    19	      {OPTIONS.map((opt) => (
    20	        <button
    21	          key={opt.value}
    22	          type="button"
    23	          aria-label={opt.value}
    24	          onClick={() => setTheme(opt.value)}
    25	          className={`rounded-sm p-1.5 transition-colors ${
    26	            theme === opt.value
    27	              ? 'bg-accent text-accent-foreground'
    28	              : 'text-muted-foreground hover:text-foreground'
    29	          }`}>
    30	          {opt.icon}
    31	        </button>
    32	      ))}
    33	    </div>
    34	  );
    35	}
    36	(EOF)
```

- **Three options, fixed order** (lines 7–11): `light` → `<Sun>`, `system` → `<Monitor>`, `dark` → `<Moon>`. All icons `size-4` lucide-react.
- **Labels:** there is no visible text label per button; the accessible label is `aria-label={opt.value}` (line 23), i.e. the raw value string `"light"` / `"system"` / `"dark"` (not translated). The dialog provides a single section label above the control (see mount below).
- **Reads the current value:** `theme = useThemeStore((s) => s.theme)` (line 14); the selected button is styled via `theme === opt.value` (line 26) → `bg-accent text-accent-foreground`, others muted.
- **Writes a new value:** `onClick={() => setTheme(opt.value)}` (line 24) — calls the store's public `setTheme` with the chosen three-state value. It always writes the raw choice (including `'system'`); resolution + class + gated-cookie all happen inside `setTheme` (§1).

**Mounted in `PortalConfigDialog`** (confirmed — the brief's guess was correct):

```
$ grep -n "ThemeToggle" src/components/popups/dialogs/PortalConfigDialog.tsx
22:import { ThemeToggle } from '../../client/buttons/ToggleButton';
151:            <ThemeToggle />
```

Context around the mount (lines 146–153), showing the section label:

```
   146	          <div className="flex items-center justify-between gap-4">
   147	            <div className="text-left">
   148	              <p>{tDialog('portal.config.theme.toggle.label')}</p>
   149	            </div>
   150	
   151	            <ThemeToggle />
   152	          </div>
```

The translated row label is `portal.config.theme.toggle.label` (next-intl), with the toggle right-aligned in a justify-between row. `PortalConfigDialog` is the portal settings modal (DrawerDialog) that also hosts card-size, language, and base-site controls.

---

## 7. SEAM NOTE — portable vs strictly-web

**Portable to RN (the conceptual machine):**

- The **three-state → resolved-two-state machine**: `theme ∈ {light, dark, system}` as the stored choice, `resolvedTheme ∈ {light, dark}` as the effective value, with `system` resolving against the OS. On RN the OS read is `Appearance.getColorScheme()` instead of `matchMedia`, but the store shape, the `setTheme` → resolve → apply flow, and the separate internal `_setResolvedTheme` are all directly mirror-able.
- The **live OS listener**: `SyncThemeFromSystem`'s "subscribe only while `theme === 'system'`, push current value on mount, update `resolvedTheme` on change without touching `theme` or persistence" logic maps cleanly to `Appearance.addChangeListener`.
- The **toggle UX**: a three-segment Sun/Monitor/Moon control that writes the raw choice and highlights the active segment.

**Strictly web (do NOT port):**

- The **cookie** persistence (`globalCookie` blob, `updateGlobalCookie`, 365-day document cookie) — RN persists via its own storage (e.g. AsyncStorage), not cookies, and is not subject to the same wire/SSR constraints.
- The **consent gate** on persistence (`isPreferenceConsentGranted`) — web ties the cookie write to Consent Mode v2; whether mobile gates theme persistence on consent is a product decision for the expo chat, not an automatic carry-over.
- The **SSR snippet** (`generateThemeSnippet` / `ssr.ts`), the **server-side `.dark` class** on `<html>` (`ssrHtmlClass`), and **`suppressHydrationWarning`** — all exist only to defeat first-paint flash in a server-rendered DOM. RN has no SSR, no `<html>`, no hydration, so none apply.
- The **apply mechanism** — `.dark` class on `<html>` + Tailwind `dark:` cascade — is DOM-specific. RN applies theme via its own mechanism (surfaced in the expo audit), not a document class.

---

## Cross-check ledger (TOOL-FABRICATION GUARD)

Every file cited above was dumped with `cat -n` or located/confirmed with `grep -rn`/`grep -rl` in this session; the raw output is inlined at each section. Files proven to exist: `src/lib/store/useTheme.ts`, `src/lib/theme/ssr.ts`, `src/components/client/SyncThemeFromSystem.tsx`, `src/components/client/SyncThemeFromCookie.tsx`, `src/components/client/buttons/ToggleButton.tsx`, `src/components/client/initializers/AppInit.tsx`, `src/lib/service/oglasinoCookies.ts`, `src/lib/consent/gating.ts`, `src/lib/types/cookie/GlobalCookie.ts`, `app/layout.tsx`, `src/components/popups/dialogs/PortalConfigDialog.tsx`, `src/components/icons/OglasinoIcon.tsx`. No Read-only claims; no fabricated content.
