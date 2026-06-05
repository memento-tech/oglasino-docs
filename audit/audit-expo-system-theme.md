# EXPO CURRENT-STATE AUDIT — expo-system-theme

Repo: `oglasino-expo`. Branch: `new-expo-dev` (read-only, no edits, no commits). Date: 2026-06-01.

**Headline:** Mobile theming today is **entirely NativeWind-driven and two-state (`light`/`dark`)**. There is **no app-owned theme store, no choice/resolved split, and no persistence** — `useColorScheme().toggleColorScheme` from `nativewind` is the only writer, and it is in-memory. There is **no `'system'` value, no `Appearance` usage, and no OS-appearance read anywhere in app code.** Adding `'system'` is largely net-new: it needs an app-owned choice store + persistence + an OS listener, because NativeWind's binary `toggleColorScheme` has nowhere to hold a third state.

Every claim below is backed by raw shell output (`cat -n` / `grep` / `sed -n`), per the tool-fabrication guard.

---

## 1. THEME STORE / STATE

**There is no app-owned theme store, context, or hook.** The "state" is NativeWind's internal `useColorScheme()` hook, consumed directly in components. There is no Zustand slice, no React context, no resolved-vs-choice distinction — a single binary value (`'light' | 'dark'`, possibly `null`/`undefined` before first paint) provided by the library.

Proof there is no theme store file (searched the store dir and lib):

```
$ ls src/lib/store/
authStore.ts  bootStore.ts  ... useCardSizeStore.ts  useConsentStore.ts  usePortalScope.ts ...
  (no themeStore / useThemeStore / colorSchemeStore)
```

The only static theme artifact is a **color-token table** at `src/lib/theme.ts` — it is NOT a store; it's a frozen `THEME` object plus a `NAV_THEME` map for React Navigation. It hardcodes exactly two keys, `light` and `dark`:

```
$ cat -n src/lib/theme.ts   (key lines)
     3	export const THEME = {
     4	  light: { background: 'hsl(195 21% 94%)', ... },
    30	  dark:  { background: 'hsl(0 0% 14%)', ... },
    56	};
    58	export const NAV_THEME: Record<'light' | 'dark', Theme> = {
    59	  light: { ...DefaultTheme, colors: {...} },
    70	  dark:  { ...DarkTheme,    colors: {...} },
    81	};
```

Note line 58: the type is literally `Record<'light' | 'dark', Theme>`. **A third value indexing this map (`NAV_THEME['system']`) would be `undefined`** — see §8.

The "current theme" is read at the consumption site via NativeWind:

```
$ grep -rn "useColorScheme" src app | grep nativewind   (representative)
app/_layout.tsx:19:import { useColorScheme } from 'nativewind';
app/_layout.tsx:34:  const { colorScheme } = useColorScheme();
src/components/dialog/dialogs/PortalConfigDialog.tsx:39:  const { colorScheme, toggleColorScheme } = useColorScheme();
src/components/dashboard/layout/DashboardSidebar.tsx:34:  const { colorScheme, toggleColorScheme } = useColorScheme();
src/components/ThemeSupportiveIcon.tsx:28:  const { colorScheme } = useColorScheme();
  ... (20 total consumer files — full list in §5)
```

**States that exist today:** `'light'` and `'dark'` only (NativeWind's `ColorSchemeName`, which can also be `null`/`undefined` before hydration — the code defends against that with `colorScheme ?? 'light'`, see §3). **`'system'` is NOT present anywhere.**

---

## 2. PERSISTENCE

**There is no theme persistence.** Nothing writes the chosen scheme to AsyncStorage (or anywhere) on change, and nothing reads it on boot. The choice lives only in NativeWind's in-memory runtime and is **lost on every app restart**.

Proof — exhaustive AsyncStorage inventory shows a key for every persisted concern *except theme*:

```
$ grep -rn "AsyncStorage" src | grep -iE "\.(ts|tsx):"   (the persisting modules)
src/lib/init/baseSitesService.ts        → KEYS.BASE_SITE
src/i18n/storage.ts                      → KEYS.LANGUAGE
src/lib/storage/consentStorage.ts        → CONSENT_KEY
src/lib/storage/authStorage.ts           → ACCESS_TOKEN_KEY
src/lib/storage/firestoreEnsuredStorage.ts
src/lib/store/useCardSizeStore.ts        → STORAGE_KEY (card size)
src/lib/store/checksumStorage.ts         → translations/catalog checksums
src/lib/store/softUpdateDismissal.ts     → STORAGE_KEY (soft-update dismissal)
src/lib/store/authStore.ts               → zustand persist(AsyncStorage)
src/lib/store/userPreferenceStorage.ts   → generic PREFIX'd kv helper
src/notifications/components/PushNotificationsInit.tsx → LAST_PROMPT_KEY
```

No `theme`/`colorScheme` key in any of them:

```
$ grep -rniE "STORAGE_KEY|_KEY =|KEYS\." src/lib/store src/lib/storage src/i18n | grep -i theme
  (no output)
```

No `setColorScheme` call exists anywhere (so nothing programmatically restores a saved value on boot):

```
$ grep -rn "setColorScheme" src app | grep -iE "\.(ts|tsx):"
NO setColorScheme MATCHES
```

**Consent/gating question (from brief):** Moot — since the theme is not persisted at all, there is no write to gate. The two writers (`toggleColorScheme` in PortalConfigDialog and DashboardSidebar, §6) are **unconditional** `onPress={toggleColorScheme}` with no consent or auth check. This is consistent with the `consent-mode-mobile` premise that theme is functional/ungated, but the stronger fact is: **mobile does not persist theme today at all.** Adding `'system'` and persistence is the same piece of work.

(`src/lib/store/userPreferenceStorage.ts` exists as a generic `PREFIX`'d AsyncStorage kv helper — a natural home for a future theme key — but it is not wired to theme today.)

---

## 3. DEFAULT-WHEN-ABSENT

There is **no app-level default**. The boot path never sets an initial color scheme — no `setColorScheme`, no `Appearance.getColorScheme()`, no stored-value restore (proven in §2). The effective default is therefore **whatever NativeWind's runtime yields**, which the app does not control from source.

The only app-source "default" is a **render-time fallback** when NativeWind's value is nullish, in `app/_layout.tsx`:

```
$ sed -n '34p;77p' app/_layout.tsx
    34	  const { colorScheme } = useColorScheme();
    77	        <ThemeProvider value={NAV_THEME[colorScheme ?? 'light']}>
```

So **React Navigation chrome falls back to `light`** when `colorScheme` is `null`/`undefined`. NativeWind's `dark:` utility classes (the rest of the UI) have their own internal default and are not governed by this `?? 'light'`.

**Honesty note (fabrication guard):** NativeWind v4's documented behavior is to follow the device color scheme by default with `darkMode: 'class'`, but I **cannot prove the fresh-install boot value from this repo's source** — no app code sets or reads it. What I *can* prove: the app supplies no explicit default of its own beyond the `?? 'light'` fallback for the nav theme.

---

## 4. APPLY MECHANISM

**Mechanism: NativeWind `darkMode: 'class'` + CSS custom properties.** This is the RN analog of web's `.dark`-on-`<html>`. NativeWind toggles a dark class internally; the Tailwind color tokens are CSS variables that swap under a `.dark:root` selector. There is NO React token-context and NO StyleSheet swap for the bulk of styling.

Config (`tailwind.config.js`):

```
$ sed -n '1,7p' tailwind.config.js
     1	const { hairlineWidth } = require('nativewind/theme');
     4	module.exports = {
     5	  darkMode: 'class',
     6	  content: ['./app/**/*.{ts,tsx}', './src/components/**/*.{ts,tsx}'],
     7	  presets: [require('nativewind/preset')],
```

Color tokens are CSS variables with a light `:root` set and a dark `.dark:root` set (`global.css`):

```
$ sed -n '9,16p;46,52p' global.css
     9	@layer base {
    10	  :root {
    11	    /* Background / foreground */
    12	    --background: 195 21% 94%;
    ...
    46	  .dark:root {
    47	    /* Background / foreground */
    48	    --background: 0 0% 14%;
    ...
```

Tailwind classes reference those vars (`tailwind.config.js`), e.g. `background: 'hsl(var(--background))'` (line 12). So a component writing `className="bg-background"` automatically re-colors when the dark class flips. `global.css` is imported once at the top of `app/_layout.tsx` (`import '../global.css';`, line 1).

**Second, parallel apply path — React Navigation chrome:** the `NAV_THEME` map (§1) is fed to `@react-navigation/native`'s `ThemeProvider` in `app/_layout.tsx:77` (`value={NAV_THEME[colorScheme ?? 'light']}`). This colors navigation containers/headers, which Tailwind classes don't reach.

**Third path — imperative per-component color picks:** many components read `colorScheme` and branch a literal color (icons, placeholders, toasts). These are scattered ternaries like `colorScheme === 'dark' ? '#22c55e' : '#028431'` — full list in §5. Each is a binary `=== 'dark'` check (relevant to §8).

The app does **not** ignore appearance — it actively themes through all three paths above. What it does NOT do is read the OS appearance itself (§5).

---

## 5. OS APPEARANCE — CURRENT USAGE

**`react-native`'s `Appearance` API is NOT imported anywhere. `prefers-color-scheme` does not appear. The OS-following primitive is net-new to app code.**

```
$ grep -rn "Appearance" src app | grep -iE "\.(ts|tsx):"
NO Appearance MATCHES

$ grep -rn "prefers-color-scheme" src app global.css
NONE
```

The only OS-appearance-adjacent primitive in use is **NativeWind's** `useColorScheme` (NOT `react-native`'s). Every hit (`grep -rniE "useColorScheme|colorScheme"`), all 20 consumer files:

```
src/components/DynamicIcon.tsx:1,18,29
src/components/Rating.tsx:2,26,57
src/components/MarkdownViewer.tsx:3,17,74
src/components/ThemeSupportiveIcon.tsx:1,28,30
src/components/SearchInput.tsx:8,52,171
src/components/init/HardUpdateScreen.tsx:3,23,31
src/components/navigation/BottomBar.tsx:9,26,51
src/components/product/CallUserButton.tsx:4,25,68
src/components/product/ShareProductButton.tsx:4,24,43
src/components/product/ProductReviewButton.tsx:5,23,74
src/components/product/StartMessageButton.tsx:8,22,34
src/components/product/ProductFunctions.tsx:14,27,43
src/components/dialog/dialogs/ReportDialog.tsx:12,38,150
src/components/dialog/dialogs/ProductReviewDialog.tsx:15,49,174
src/components/dialog/dialogs/PortalConfigDialog.tsx:15,39 (+toggleColorScheme:88,90)
src/components/dashboard/layout/DashboardSidebar.tsx:24,34 (+toggleColorScheme:109/85,111/87)
src/components/icons/OglasinoIcon.tsx:1,22,26,28,43
src/lib/provider/ToastProvider.tsx:4,9,26,28,30,32
app/_layout.tsx:19,34,77
```

All 20 import `from 'nativewind'`. Only **two** of them write (`toggleColorScheme`, §6); the other 18 are read-only consumers. **None read `Appearance` or the system scheme directly** — they read NativeWind's already-resolved binary value.

---

## 6. TOGGLE / SETTINGS UI

The user changes theme in **two** places, both a **binary icon toggle** (Sun ⇄ MoonStar), both calling NativeWind's `toggleColorScheme` directly. There is **no three-way control, no segmented control, no `'system'` option, and no light/dark *labels*** (the icon implies the *target*, not the current state).

> ⚠️ The `consent-mode-mobile` audit lineage points at `app/owner/dashboard/user.tsx` as a settings home, but **that file contains no theme control today** — confirmed: `grep -niE "theme|colorScheme" app/owner/dashboard/user.tsx` → no output. The theme controls live in the two files below.

**(a) `PortalConfigDialog.tsx`** — the public-portal config dialog. This is the labelled one:

```
$ sed -n '39p;85,96p' src/components/dialog/dialogs/PortalConfigDialog.tsx
    39	  const { colorScheme, toggleColorScheme } = useColorScheme();
    85	        <View className="flex-row items-center justify-between gap-2">
    86	          <Text className="italic">{tDialog('portal.config.theme.toggle.label')}</Text>
    87	          <Pressable
    88	            onPress={toggleColorScheme}
    89	            className="h-10 w-10 items-center justify-center rounded-full border border-border bg-background-mild">
    90	            {colorScheme === 'dark' ? (
    91	              <ThemeSupportiveIcon Icon={Sun} size={20} />
    92	            ) : (
    93	              <ThemeSupportiveIcon Icon={MoonStar} size={20} />
    94	            )}
    95	          </Pressable>
    96	        </View>
```

Reads current value: `colorScheme === 'dark'` (binary ternary). Writes: `onPress={toggleColorScheme}` (no argument — NativeWind flips light⇄dark). Imports: `useColorScheme` from `nativewind` (line 15); icons `MoonStar, Sun` from `lucide-react-native` (line 14); `ThemeSupportiveIcon`; `useTranslations` (line 5).

**(b) `DashboardSidebar.tsx`** — the owner-dashboard slide-out sidebar. **Unlabelled** (icon-only):

```
$ sed -n '34p;84,92p' src/components/dashboard/layout/DashboardSidebar.tsx   (real line numbers)
    34	  const { colorScheme, toggleColorScheme } = useColorScheme();
   ...
    84	              <Pressable
    85	                onPress={toggleColorScheme}
    86	                className="h-10 w-10 items-center justify-center rounded-full bg-background-mild">
    87	                {colorScheme === 'dark' ? (
    88	                  <ThemeSupportiveIcon Icon={Sun} size={20} />
    89	                ) : (
    90	                  <ThemeSupportiveIcon Icon={MoonStar} size={20} />
    91	                )}
    92	              </Pressable>
```

Same pattern: binary read `colorScheme === 'dark'`, unconditional `toggleColorScheme` write, no text label.

**Implication for a 3-state control:** `toggleColorScheme` is a *flip*, not a *set* — it has no concept of choosing among three values. A `'system'` control cannot reuse it; it needs an explicit `set(choice)` writer. Both call sites would have to change from a flip-toggle to a 3-way selector (or be replaced).

---

## 7. TRANSLATION KEYS — CURRENT

Translations are **backend-seeded** (no local JSON in repo; `src/i18n/` holds only `fetchNamespace.ts`, `storage.ts`, `types.ts`, `useTranslations.ts`). Exactly **one** theme-related key exists today:

```
$ grep -rniE "'[^']*theme[^']*'" src app | grep -iE "\.(ts|tsx):" | grep "t.*("
src/components/dialog/dialogs/PortalConfigDialog.tsx:86:  ...{tDialog('portal.config.theme.toggle.label')}
```

- **Key:** `portal.config.theme.toggle.label`
- **Namespace:** `DIALOG` (`tDialog = useTranslations(TranslationNamespace.DIALOG)`, PortalConfigDialog line 28)
- **Pattern:** dot-namespaced under the dialog's `portal.config.*` family (siblings: `portal.config.title`, `portal.config.description`, `portal.config.card.size.label`, `portal.config.language.label`, `portal.config.domain.label`). New `'system'`-related option keys for *this dialog* would naturally follow `portal.config.theme.*` in the `DIALOG` namespace.
- **DashboardSidebar's toggle uses NO translation key** (icon-only, §6b).

**Antipattern check (brief):** mobile does **not** use the web aria-label-as-raw-value antipattern. The single key is a normal human label string fetched via `tDialog(...)`; the current/target state is conveyed by swapping the Sun/MoonStar icon, not by a raw-value label. New keys for a 3-way control (e.g. light/dark/system option labels) would be net-new in the `DIALOG` namespace and must be backend-seeded.

---

## 8. SEAM NOTE

Adding `'system'` is **mostly net-new, not a widen-in-place.** Today there is no app-owned theme state to widen — the "state" is NativeWind's binary `useColorScheme`, whose `toggleColorScheme` is a light⇄dark *flip* with no slot for a third value and no persistence (§1, §2, §6). To mirror web's choice/resolved machine you must add, net-new: (a) an app-owned store holding `theme ∈ {light,dark,system}` (the choice) plus a derived `resolvedTheme ∈ {light,dark}`; (b) AsyncStorage persistence of the *choice* (a `userPreferenceStorage` key is the obvious host); (c) an OS listener — `Appearance.getColorScheme()` + `Appearance.addChangeListener` — which is **net-new** because `Appearance` is imported nowhere today (§5); and (d) a bridge that pushes the resolved value into NativeWind via `setColorScheme(resolved)` (also net-new — `setColorScheme` is called nowhere today). The two existing toggle call sites (§6) must change from `toggleColorScheme` to a 3-way `setTheme(choice)` selector.

**What breaks if `'system'` is added naively — concrete fall-through sites to flag:**

1. **`src/lib/theme.ts:58`** — `NAV_THEME: Record<'light' | 'dark', Theme>` is indexed in `app/_layout.tsx:77` as `NAV_THEME[colorScheme ?? 'light']`. If a raw `'system'` ever reaches this index it returns `undefined` → React Navigation gets an undefined theme. The `?? 'light'` guards `null`/`undefined` but **NOT** the string `'system'`. The index must be fed the **resolved** value, never the choice.

2. **Persistence shape.** There is no persistence today, so there is no "raw resolved value stored" hazard yet — but the design must persist the **choice** (`light|dark|system`), and feed NativeWind/`NAV_THEME` only the **resolved** value. Persisting a resolved value (as a binary-only design would) would silently lose the user's `system` intent on the next launch — the exact choice/resolved split the brief flags.

3. **~20 binary `colorScheme === 'dark' ? … : …` consumers** (§4, §5) — these read NativeWind's *resolved* value, so they are SAFE **iff** the design keeps pushing a resolved `light|dark` into NativeWind (path (d) above). They become wrong only if any of them is refactored to read the *choice* store directly and then does `=== 'dark'`, in which case `'system'` falls through to the light branch. Keep these reading the resolved scheme.

4. **Every `colorScheme === 'dark'` / `colorScheme === 'light'` ternary** (PortalConfigDialog:90, DashboardSidebar:87, ThemeSupportiveIcon:30, OglasinoIcon:28, BottomBar:51, ToastProvider:26–32, etc.) assumes exactly two states. None handle a third value, so the **choice** value must never be handed to these — only the resolved binary.

**Bottom line:** the seam is clean *if* the new store keeps choice and resolved strictly separated and continues to drive the existing binary world (NativeWind + `NAV_THEME` + the ternaries) with the **resolved** value only. The `'system'` value should never reach `theme.ts`, `_layout.tsx`'s `NAV_THEME` index, or any `=== 'dark'/'light'` consumer.

---

### File index (all shell-verified to exist this session)

| File | Role |
|---|---|
| `src/lib/theme.ts` | Color-token table + `NAV_THEME` (`Record<'light'\|'dark'>`). Not a store. |
| `tailwind.config.js` | `darkMode: 'class'`, CSS-var color tokens. |
| `global.css` | `:root` (light) + `.dark:root` (dark) CSS-var sets. |
| `app/_layout.tsx` | Reads `colorScheme`; feeds `NAV_THEME[colorScheme ?? 'light']` to RN `ThemeProvider`. |
| `src/components/dialog/dialogs/PortalConfigDialog.tsx` | Labelled binary theme toggle (DIALOG ns key). |
| `src/components/dashboard/layout/DashboardSidebar.tsx` | Unlabelled binary theme toggle. |
| `src/components/ThemeSupportiveIcon.tsx` | Reads `colorScheme` to pick icon color (binary). |
| `src/lib/store/userPreferenceStorage.ts` | Generic AsyncStorage kv helper — natural future home for a theme key (not wired today). |
| (absent) theme store / `Appearance` / `setColorScheme` / theme AsyncStorage key | **Do not exist** — net-new for `'system'`. |
