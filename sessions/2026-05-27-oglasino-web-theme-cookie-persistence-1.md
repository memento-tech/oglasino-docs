# Session summary

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-27
**Task:** Read-only audit of theme cookie persistence — verify Path A is still the right choice, produce ground-truth findings for the canonical spec and engineering briefs.

## Audit findings

### 1. `next-themes` version and API surface

- **Installed version:** v0.4.6. Confirmed in `package.json:58` (`"next-themes": "^0.4.6"`), `package-lock.json:41`, and `node_modules/next-themes/package.json:3` (resolved `"version": "0.4.6"`). The handoff's claim is correct.
- **No pluggable storage mechanism.** `ThemeProviderProps` in `node_modules/next-themes/dist/index.d.ts:25-48` exposes `storageKey?: string` (line 37) — this controls the localStorage key name (default `"theme"`), not the storage backend. No `storage`, `storageProvider`, `customStorage`, `onThemeChange`, or any other escape hatch exists in the type definition. The full props list is: `themes`, `forcedTheme`, `enableSystem`, `disableTransitionOnChange`, `enableColorScheme`, `storageKey`, `defaultTheme`, `attribute`, `value`, `nonce`, `scriptProps`.
- **localStorage writes are unconditional.** In the minified source (`node_modules/next-themes/dist/index.mjs:1`), the `setTheme` callback (the `f` variable) is:
  ```
  f = useCallback(o => {
    let r = typeof o == "function" ? o(c) : o;
    n(r);                            // React setState
    try { localStorage.setItem(m, r) } catch(v) {}  // unconditional write
  }, [c])
  ```
  There is no gate, no callback, and no way to intercept or prevent the `localStorage.setItem` call. Any consumer calling `setTheme('dark')` will write `"dark"` to `localStorage["theme"]` immediately, regardless of consent state.
- **Pre-paint script also reads localStorage unconditionally.** The inline `<script>` rendered by `next-themes` (the `M` function and `_` memoized component) executes `localStorage.getItem(i) || s` where `i` is the `storageKey` and `s` is `defaultTheme`. No cookie read, no fallback mechanism.
- **Initial state reads localStorage unconditionally.** The `H` function returns `localStorage.getItem(e) || i` on mount.

**Conclusion: B1-A (custom storage adapter) is genuinely unavailable in v0.4.6. B1-B (mirror pattern) is genuinely unworkable because `setTheme` writes to localStorage unconditionally — there is no way to gate the write on consent.**

### 2. Current theme provider and toggle UI

- **`ThemeProvider` is mounted exactly once** at `app/layout.tsx:80`, inside `<body>`, wrapping `{children}`:
  ```tsx
  <ThemeProvider>{children}</ThemeProvider>
  ```
  It is nested inside `<ConfigProvider>`. No other mount points exist.
- **Props passed** at `src/components/providers/ThemeProvider.tsx:7-11`:
  - `attribute="class"` — sets `class="dark"` or `class="light"` on `<html>`
  - `defaultTheme="system"` — respects `prefers-color-scheme` on first visit
  - `enableSystem` — enables system theme detection
  - `disableTransitionOnChange` — disables CSS transitions during theme switch
- **`<html>` attributes today:** `next-themes` adds/removes the `dark` or `light` class on `document.documentElement`. The `<html>` tag in `app/layout.tsx:58-61` has `className={\`${inter.variable} overscroll-none font-sans antialiased\`}` plus `suppressHydrationWarning` (required by `next-themes`' pre-paint script).
- **Three consumers of `useTheme()`:**
  1. `src/components/client/buttons/ToggleButton.tsx:14` — the `ThemeToggle` component. Uses `theme`, `setTheme`, `resolvedTheme`. **This is the only call site that mutates theme** via `setTheme(isDark ? 'light' : 'dark')` at line 34.
  2. `src/components/icons/OglasinoIcon.tsx:21` — reads `theme` and `resolvedTheme` to select light/dark logo path. Read-only.
  3. `src/components/shadcn/ui/sonner.tsx:14` — reads `theme` to pass to Sonner toast library. Read-only.
- **`ThemeToggle` is rendered from** `src/components/popups/dialogs/PortalConfigDialog.tsx:151`, inside the settings dialog. This is the sole user-facing theme control.

### 3. Tailwind v4 dark-mode infrastructure

- **Dark-mode selector:** `.dark` class on `<html>`, class-based. Defined at `app/globals.css:5`:
  ```css
  @custom-variant dark (&:where(.dark, .dark *));
  ```
  This is the Tailwind v4 CSS-first way to declare the dark variant. Any `dark:` utility in the codebase resolves against the `.dark` class.
- **CSS variable token system:** `:root` (lines 136–174) defines light-mode variables. `.dark` (lines 176–206) overrides them for dark mode. This matches the handoff's description exactly.
- **No `tailwind.config.ts` or `tailwind.config.js`.** Tailwind v4 uses CSS-first configuration. The `@theme inline` block at `globals.css:70-120` maps CSS custom properties to Tailwind tokens.
- **No dark-mode logic outside `globals.css`.** Grep for `dark` and `theme` across all `.css` files in `app/` returned only `globals.css` matches.

**The pre-paint script must set `class="dark"` on `document.documentElement` — this is the exact attribute Tailwind reads.**

### 4. Cookie infrastructure

- **`GlobalCookie` type** at `src/lib/types/cookie/GlobalCookie.ts:5-6`: `theme?: Theme` where `Theme = 'light' | 'dark'`. The field exists and is typed. The handoff's type claim is correct. Adding `'system'` would require widening this type.

- **Cookie read helpers:**
  - Client-side: `getGlobalCookie(): GlobalCookie | null` at `oglasinoCookies.ts:18-20`. Parses `document.cookie` via regex match and `JSON.parse(decodeURIComponent(...))`. Returns `null` on server or if cookie is absent/malformed.
  - Server-side: `readGlobalCookieForSsr(cookieStore: CookieStore): GlobalCookie` at `oglasinoCookies.ts:62-78`. Takes the Next.js `cookies()` store, reads `GLOBAL_COOKIE_NAME`, returns parsed cookie with defaults for card sizes. Already reads `theme` from the parsed cookie at line 74: `theme: parsed.theme`.

- **Cookie write helper:** `updateGlobalCookie<K>(key: K, value: GlobalCookie[K], days = 365)` at `oglasinoCookies.ts:22-28`. Generic per-field writer — accepts any `GlobalCookie` key and its typed value. Internally calls `updateCookie` which reads the full cookie, spreads `{ ...cookie, [key]: value }`, and writes back via `setCookie`. Theme writes would be: `updateGlobalCookie('theme', 'dark')`.

- **`useCardSizeStore.setSize` pattern** at `useCardSize.ts:35-51`:
  1. Updates React/Zustand state unconditionally: `set((state) => ({ sizes: { ...state.sizes, [scope]: size } }))`.
  2. Checks consent: `if (typeof window !== 'undefined' && isPreferenceConsentGranted())`.
  3. Writes to cookie only on consent: `updateGlobalCookie('portalCardSize', size)`.
  Theme's store will mirror this exact pattern: in-memory always, cookie only when preference consent is granted.

- **`SyncCardSizeFromCookie` reconciliation pattern** at `src/components/client/SyncCardSizeFromCookie.tsx:8-28`:
  - Subscribes to `useConsentStore((s) => s.consent?.preference === 'granted')`.
  - When `preferenceGranted` transitions to `true`, the effect fires.
  - Reads both in-memory state (`useCardSizeStore.getState().sizes`) and cookie state (`getGlobalCookie()`).
  - **In-memory wins:** if the in-memory value is set (user toggled before granting consent), writes it to cookie.
  - **Cookie wins:** if in-memory is unset but cookie has a value (returning visitor who granted consent previously), reads cookie into store.
  - This is the exact reconciliation semantics the theme store will need.

- **Cookie clearance on consent decline** at `useConsentStore.ts:44-46`:
  ```typescript
  if (prev?.preference !== 'denied' && next.preference === 'denied') {
    clearGlobalCookie();
  }
  ```
  `clearGlobalCookie()` at `oglasinoCookies.ts:51-54` sets `Max-Age=0` on the entire `globalCookie` — it nukes the whole cookie, not individual fields. **A new `theme` field is automatically cleared on consent decline with zero new code.** No per-field enumeration exists; no extension is needed.

### 5. Edge / SSR readability of the cookie

- **`proxy.ts` reads `globalCookie` at the edge** at `proxy.ts:48-65`. It calls `request.cookies.get(GLOBAL_COOKIE_NAME)?.value`, then `JSON.parse(decodeURIComponent(raw))` and reads `parsed.lang`. Reading `parsed.theme` would follow the identical pattern — the cookie is fully edge-readable and already parsed in the proxy.

- **Existing inline pre-paint `<script>` pattern:** the Consent Mode v2 snippet at `app/layout.tsx:46-55` is rendered inside `<head>` at line 63:
  ```tsx
  <script dangerouslySetInnerHTML={{ __html: consentDefaultSnippet }} />
  ```
  The snippet sets up `window.dataLayer`, defines `gtag()`, calls `gtag('consent', 'default', {...})`, and sets `window.__og_consent_loaded = true`. The consent values are derived server-side from `readConsentForSsr(cookieStore)` and interpolated into the snippet via `sanitizeForSnippet()`. **This is the exact template the theme pre-paint script would follow** — a server-side cookie read feeding values into an inline `<script dangerouslySetInnerHTML={...}>` in `<head>`.

- **`next-themes`' own pre-paint script:** rendered by the `_` memoized component inside the `V` (`ThemeProvider`) component. It creates a `<script>` element with `dangerouslySetInnerHTML` containing the self-invoking `M` function. This script **renders inside `<body>`** (because `ThemeProvider` is mounted inside `<body>` at `layout.tsx:80`), not inside `<head>`. The `M` function reads `localStorage.getItem(storageKey)`, resolves system preference if needed, and applies the class/attribute to `document.documentElement`. Under Path A, the replacement script would:
  1. Render in `<head>` (alongside the consent snippet), closing the FOUC window that `next-themes`' body-rendered script leaves open.
  2. Read theme from the `globalCookie` cookie instead of localStorage.
  3. Fall back to `prefers-color-scheme` media query if no cookie value exists.
  4. Set `document.documentElement.classList.add('dark')` or remove it.

## Files touched

None — read-only audit.

## Tests

- Ran: N/A (read-only audit)
- Result: N/A
- New tests added: none

## Cleanup performed

None needed — read-only audit.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

Nothing.

## Conventions check

- Part 4 (cleanliness): N/A (read-only, no code changes)
- Part 4a (simplicity): N/A (no code added/removed)
- Part 4b (adjacent observations): all observations listed in "For Mastermind" below
- Part 6 (translations): N/A this session
- Part 11 (trust boundaries): no theme-state trusted from an unsafe source identified. Theme is a UI preference, not a trust-boundary value — it is not used in moderation, authorization, or state-transition decisions.

## Known gaps / TODOs

None — this is an audit, not an implementation.

## For Mastermind

### Five structured answers

**1. Is B1-A genuinely unavailable in the currently-installed `next-themes` version?**

Yes. The installed version is v0.4.6, exactly as the handoff claimed. The `ThemeProviderProps` type in `node_modules/next-themes/dist/index.d.ts:25-48` exposes no pluggable storage mechanism. The `storageKey` prop (line 37) controls only the localStorage key name (default `"theme"`), not the storage backend. The `setTheme` callback in the source (`node_modules/next-themes/dist/index.mjs:1`) unconditionally calls `localStorage.setItem(m, r)` after setting React state — there is no gate, no callback hook, and no way to intercept or prevent the write. B1-A is unavailable. B1-B (mirror pattern) is unworkable for the same reason: `setTheme` writes to localStorage unconditionally, and `next-themes` provides no mechanism to suppress or defer the write.

**2. What is the exact `<html>` attribute that Tailwind reads for dark mode?**

`class="dark"` on `<html>`. Two confirming sources: (a) `ThemeProvider.tsx:8` passes `attribute="class"` to `next-themes`, which adds/removes the `dark` class on `document.documentElement`; (b) `globals.css:5` declares `@custom-variant dark (&:where(.dark, .dark *))`, the Tailwind v4 CSS-first dark variant selector matching the `.dark` class. The pre-paint script must call `document.documentElement.classList.add('dark')` or `document.documentElement.classList.remove('dark')`.

**3. What is the exact shape of the consent-gated cookie write?**

Write helper: `updateGlobalCookie<K extends keyof GlobalCookie>(key: K, value: GlobalCookie[K], days = 365)` at `oglasinoCookies.ts:22-28`. Accepts any typed `GlobalCookie` field. Internally reads the existing cookie, spreads the new value, writes back as `document.cookie`. Gating pattern from `useCardSizeStore.setSize` at `useCardSize.ts:35-51`: (1) update Zustand state unconditionally via `set(...)`, (2) check `isPreferenceConsentGranted()` from `src/lib/consent/gating.ts:11-17` (which reads `useConsentStore.getState().consent?.preference === 'granted'`), (3) call `updateGlobalCookie(...)` only if granted. Theme's store mirrors this: in-memory always, cookie on consent only.

**4. Does the existing cookie-clearance-on-decline mechanism handle a new field automatically?**

Yes, automatically — no extension needed. `useConsentStore.setConsent` at `useConsentStore.ts:44-46` calls `clearGlobalCookie()` when consent transitions to denied. `clearGlobalCookie()` at `oglasinoCookies.ts:51-54` sets `Max-Age=0` on the entire `globalCookie` — it destroys the whole cookie, not specific fields. Any new field added to `GlobalCookie` (including `theme`, which already exists in the type at `GlobalCookie.ts:12`) is automatically cleared. There is no per-field enumeration to extend.

**5. Where exactly does the existing pre-paint SSR snippet sit, and what is its template?**

The Consent Mode v2 snippet is at `app/layout.tsx:46-55` (snippet construction) and `app/layout.tsx:63` (rendering). It sits inside `<head>`:
```tsx
<script dangerouslySetInnerHTML={{ __html: consentDefaultSnippet }} />
```
The `consentDefaultSnippet` string is built server-side: `readConsentForSsr(cookieStore)` reads the consent cookie, `sanitizeForSnippet()` sanitizes the values, and the snippet is interpolated as a template literal. The theme pre-paint script would be a sibling `<script dangerouslySetInnerHTML={...}>` at the same location — server-side cookie read of `readGlobalCookieForSsr(cookieStore).theme`, interpolated into an inline script that sets `document.documentElement.classList.add('dark')` (or not) before paint.

`next-themes` currently renders its own pre-paint script inside `<body>` (because `ThemeProvider` is mounted at `layout.tsx:80` inside `<body>`). That script reads from localStorage. Under Path A, the replacement in `<head>` reads from the cookie, and `next-themes` is removed entirely — its body-rendered script goes away.

### Part 4a simplicity evidence (required)

- Added (earned complexity): nothing — read-only audit
- Considered and rejected: nothing — no code decisions made
- Simplified or removed: nothing — read-only audit

### Adjacent observations (Part 4b)

1. **`disableTransitionOnChange` set but no theme-specific CSS transitions exist.** `ThemeProvider.tsx:11` passes `disableTransitionOnChange` to `next-themes`. This prop injects a temporary `<style>` that disables all CSS transitions during theme switch. The only CSS transitions in `globals.css` are `transition-transform` for button hover scale (line 22) and `transition-transform duration-100` in `ToggleButton.tsx:37-39` for icon hover. Neither is theme-dependent. The prop is harmless but unnecessary — Path A can omit it. **Severity: low.** I did not fix this because it is out of scope.

2. **`OglasinoIcon.tsx` `useEffect` deps missing `resolvedTheme`.** At `src/components/icons/OglasinoIcon.tsx:38`, the effect depends on `[theme, setLogoPath]` but reads `resolvedTheme` at line 27. If `theme` is `"system"` and the system color scheme changes (e.g., user toggles OS dark mode), `resolvedTheme` updates but `theme` stays `"system"`, so the effect doesn't re-run and the logo stays on the previous color variant until the next render for other reasons. Path A needs to decide whether to preserve `resolvedTheme` or simplify the icon to a direct theme read. **Severity: low.** I did not fix this because it is out of scope.

3. **`ThemeProvider` renders inside `<body>`, so `next-themes`' pre-paint script is body-rendered.** The pre-paint script injected by `next-themes` (the `_` memoized component in the source) renders wherever `ThemeProvider` is mounted — which is inside `<body>` at `layout.tsx:80`. This means there is a FOUC window between `<head>` (where the browser starts rendering) and the point in `<body>` where the script executes and applies the `dark` class. Path A's replacement script in `<head>` (alongside the consent snippet) closes this window. **Severity: low** (FOUC is brief because the script executes synchronously and is near the top of `<body>`, but it exists). I did not fix this because it is out of scope.

4. **`sonner.tsx` passes `theme` (which can be `"system"`) to Sonner's `theme` prop.** At `src/components/shadcn/ui/sonner.tsx:14`, `const { theme = 'system' } = useTheme()` and line 18 `theme={theme as ToasterProps['theme']}`. If `next-themes` is removed (Path A), this consumer needs a replacement source for the current resolved theme value. Sonner accepts `"light"`, `"dark"`, or `"system"`. **Severity: low.** I did not fix this because it is out of scope, but it's a consumer that Path A must address.

5. **`ToggleButton.tsx` mounted-guard pattern.** At `src/components/client/buttons/ToggleButton.tsx:15-27`, the component uses a `mounted` state to avoid hydration mismatch (renders a placeholder until `useEffect` fires). Path A's custom theme store can avoid this if the server and client agree on the initial theme via the cookie — the mounted-guard may become unnecessary. **Severity: low.** I did not fix this because it is out of scope.
