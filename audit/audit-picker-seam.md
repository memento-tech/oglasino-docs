# Audit — Intro / Base-Site Picker SEAM

**Repo:** `oglasino-expo`
**Branch:** `new-expo-dev`
**Date:** 2026-05-28
**Type:** read-only audit
**Scope:** map the seam between the existing intro / base-site picker UI and the boot layer so step 5 of the boot redesign (`features/expo-boot-redesign.md`, Part 8) can move the wiring from old `AppContext` to new `bootStore` without touching the UI.

No code changes. Cross-checked against `.agent/audit-expo-boot-redesign.md` §1.5.

---

## 1. The picker component(s)

### 1.1 Inventory — there is exactly ONE component

The intro screen and the base-site picker are **a single component**:

- **File:** `src/components/init/BaseSiteSelector.tsx` (143 lines).
- **Default export:** `BaseSiteSelector({ isMaintenance }: Props)` (line 36).

There is no separate `IntroScreen`, `WelcomeScreen`, or `BaseSitePicker` component. A grep across `src/` and `app/` for `Intro` / `Welcome` / `Picker` returns no other intro-or-picker component (the matches are: a translation namespace `INTRO` at `src/i18n/types.ts`, a category icon and product spec passing the word, the dialog `PortalConfigDialog`, and the bootStore status name `'intro-picker'` — none are screens).

This confirms the prior audit (`.agent/audit-expo-boot-redesign.md` §1.5): the only overlay component for the pre-`ready` "user must pick" path is `BaseSiteSelector`.

### 1.2 Where it is rendered from

Both render sites are in the root layout's overlay branch:

`app/_layout.tsx:72-80`

```
{status !== 'ready' && (
  <View className="absolute inset-0 items-center justify-center bg-background">
    {status === 'maintenance' && <BaseSiteSelector isMaintenance />}
    {status === 'select-base-site' && (
      <BaseSiteSelector isMaintenance={false} />
    )}
    {(!status || status === 'loading') && <LoadingOverlay />}
  </View>
)}
```

- `status === 'maintenance'` → `<BaseSiteSelector isMaintenance />` (line 74)
- `status === 'select-base-site'` → `<BaseSiteSelector isMaintenance={false} />` (line 76)
- `status === 'loading'` / undefined → `<LoadingOverlay />` (line 78). LoadingOverlay is a separate file (`src/components/LoadingOverlay.tsx`) and is not the picker.

### 1.3 Render tree from `_layout.tsx` down to the tap target

```
RootLayout                                  app/_layout.tsx:28
└── GestureHandlerRootView                  app/_layout.tsx:59
    └── AppContextProvider                   app/_layout.tsx:60   (passes onStatusChanged=setStatus, onLanguageChanged=setLocale)
        └── AppVersionConfigInit             app/_layout.tsx:61   (render-gate: returns ActivityIndicator until first version check resolves; cf. prior audit §1.1)
            └── ToastProvider
                └── ThemeProvider
                    └── SafeAreaView         app/_layout.tsx:64
                        ├── {status === 'ready' && <AppInit />}             line 65
                        ├── <View><Stack /></View>     (always mounted)      lines 66-68
                        ├── <PortalHost />                                   line 69
                        ├── <DialogManager />                                 line 70
                        └── {status !== 'ready' && overlay}                  lines 72-80
                                ├── status === 'maintenance'   → <BaseSiteSelector isMaintenance />
                                ├── status === 'select-base-site' → <BaseSiteSelector isMaintenance={false} />
                                └── status === 'loading'/undef → <LoadingOverlay />
```

Inside `BaseSiteSelector` the user-visible "intro card + picker buttons" tree is:

```
View (root bg-color)                          BaseSiteSelector.tsx:76
├── ImageBackground (intro.jpg + black/50)    lines 77-82
└── ScrollView                                lines 84-140
    └── View (centered content column)        lines 85-133
        ├── intro card                        lines 86-113
        │   ├── OglasinoIcon                  line 87
        │   ├── welcome.title                 lines 89-91
        │   ├── welcome.subtitle              lines 92-94
        │   └── EITHER maintenance text       lines 96-104 (when isMaintenance=true)
        │       OR    welcome.description + select.base.site prompt   lines 106-112
        └── {!isMaintenance && picker buttons}    lines 114-132
            └── baseSites.map(...)            line 116
                └── TouchableOpacity          lines 117-129  ◀── THE TAP TARGET
                    ├── Image (flag)          lines 121-125
                    └── Text (label)          lines 126-128
```

The user sees and taps the picker at `BaseSiteSelector.tsx:117-129`.

---

## 2. Where the list comes from (the data-in seam)

### 2.1 Source — `useAppState().baseSites`

`BaseSiteSelector.tsx:37`:

```
const { baseSites } = useAppState();
```

`useAppState()` (`AppContext.tsx:286-290`) returns the `AppStateValue` exposed by `AppStateContext.Provider`. The `baseSites` field is populated by `AppContextProvider.bootstrap()`:

`AppContext.tsx:94-98`

```
const [maintenance, configuration, baseSites] = await Promise.all([
  checkIfMaintenance().catch(() => true),
  getAppConfiguration().catch(() => undefined),
  fetchBaseSites().catch(() => []),
]);
```

`fetchBaseSites()` (`src/lib/init/baseSitesService.ts:19-32`) is:

```
export async function fetchBaseSites(): Promise<BaseSiteDTO[]> {
  try {
    const res = await BACKEND_API.get('/public/baseSite/details', { _bootstrap: true });
    if (res.status >= 200 && res.status < 300) {
      return res.data;
    }
    logServiceWarn('baseSite.fetchBaseSites', res);
    return [];
  } catch (err) {
    logServiceError('baseSite.fetchBaseSites', err);
    return [];
  }
}
```

Endpoint: **`GET /public/baseSite/details`** (NOT `/baseSite/overviews`). The result lands in `baseSites` for all three downstream branches in `bootstrap` (`AppContext.tsx:103, 114, 134`).

No prop. No direct service call inside `BaseSiteSelector`. The component reads `baseSites` from the AppContext state slice produced by `bootstrap()`.

### 2.2 Item TYPE — full `BaseSiteDTO`

`BaseSiteDTO` (`src/lib/types/catalog/BaseSiteDTO.ts:6-19`):

```
export type BaseSiteDTO = {
  id: number;
  code: string;
  domain: string;
  iconId?: string;
  labelKey: string;
  flagImageKey: string;
  defaultLanguage: LanguageDTO;
  allowedLanguages: LanguageDTO[];
  catalog: CatalogDTO;
  defaultCurrency: CurrencyDTO;
  allowedCurrencies: CurrencyDTO[];
  regions: RegionDTO[];
};
```

Each item is the **full** `BaseSiteDTO`. The picker today consumes the heavy DTO, not an overview. Gate 3 of the spec (`features/expo-boot-redesign.md` Part 2, transition table row "no stored base site → `intro-picker` … GET `/baseSite/overviews`") plans to switch the picker's list source to `/baseSite/overviews`. Whether `/baseSite/overviews` carries the exact three fields the existing UI renders is the seam concern in §2.3 below.

### 2.3 Fields READ by the picker to render

Exhaustive list — grep of `baseSite\.` inside `BaseSiteSelector.tsx`:

| # | Line | Code | Field | Used for |
|---|------|------|-------|----------|
| 1 | 118 | `key={baseSite.code}` | `code` | React list key |
| 2 | 119 | `onPress={() => setBaseSiteForCode(baseSite.code)}` | `code` | argument to tap handler |
| 3 | 122 | `source={{ uri: publicImageUrl(baseSite.flagImageKey, 'card') }}` | `flagImageKey` | flag image (resolved through `publicImageUrl` → R2) |
| 4 | 127 | `{tIntro(baseSite.labelKey)}` | `labelKey` | translation key looked up in the `INTRO` namespace |

**The picker reads exactly three fields: `code`, `flagImageKey`, `labelKey`.** It does not read `id`, `domain`, `iconId`, `defaultLanguage`, `allowedLanguages`, `catalog`, `defaultCurrency`, `allowedCurrencies`, or `regions`.

So the UI itself can render from any shape that includes `code`, `flagImageKey`, `labelKey` — including `BaseSiteOverviewDTO`, if that DTO carries those three. (Confirming the overview DTO's fields is a backend audit concern, out of scope here.)

### 2.4 An adjacent in-component data dependency — intro translations

`BaseSiteSelector.tsx:42-54` performs a direct service call independent of AppContext:

```
useEffect(() => {
  let mounted = true;

  fetchNamespace(TranslationNamespace.INTRO, 'sr').then((res) => {
    if (mounted) {
      setIntroTranslations(res || INTRO_FALLBACK);
    }
  });

  return () => { mounted = false; };
}, []);
```

- Calls `fetchNamespace(TranslationNamespace.INTRO, 'sr')` directly (`src/i18n/fetchNamespace.ts`) — a hardcoded `'sr'` language; this is a deliberate fallback because no language is resolved when the picker shows.
- The early return at `BaseSiteSelector.tsx:71-73` (`if (!introTranslations) return <ActivityIndicator />;`) means the picker briefly shows a spinner until the INTRO namespace fetch resolves.
- `tIntro(key)` (lines 56-69) is a local nested-object reader over `introTranslations`.

This is not data-in from AppContext — but it IS a backend call fired by the picker. It is independent of the seam this audit maps, but flagged so step 5 can decide whether to leave it in-component or fold it into Gate 3.

---

## 3. What happens on tap (the action-out seam — the critical one)

### 3.1 The tap handler in the component

`BaseSiteSelector.tsx:117-129`:

```
<TouchableOpacity
  key={baseSite.code}
  onPress={() => setBaseSiteForCode(baseSite.code)}
  className="..."
>
  <Image source={{ uri: publicImageUrl(baseSite.flagImageKey, 'card') }} ... />
  <Text ...>{tIntro(baseSite.labelKey)}</Text>
</TouchableOpacity>
```

`setBaseSiteForCode` is destructured from `useAppActions()` at `BaseSiteSelector.tsx:38`:

```
const { setBaseSiteForCode } = useAppActions();
```

`useAppActions()` is the hook in `AppContext.tsx:292-296` that reads `AppActionsContext`. **The tap handler is NOT a prop.** It is read from `AppContext` inside the component body.

### 3.2 `setBaseSiteForCode` body — quoted from `AppContext.tsx:187-220`

```
const setBaseSiteForCode = useCallback(
  async (siteCode: string, withLoading = true) => {
    const currentState = stateRef.current;
    const site = currentState.baseSites.find((s) => s.code === siteCode);
    if (!site) return;

    const lang =
      site.allowedLanguages.find((l) => l.code === currentState.selectedLanguage?.code) ??
      site.defaultLanguage;

    if (withLoading) {
      setStateWithStatusHook((p) => ({
        ...p,
        status: 'loading',
        selectedBaseSite: site,
        selectedLanguage: lang,
      }));
    }

    setCodes(site.code, lang.code);

    await setStoredBaseSite(site);
    await setStoredLanguage(lang);
    await initI18n(lang.code, site.code);

    setStateWithStatusHook((prev) => ({
      ...prev,
      status: 'ready',
      selectedBaseSite: site,
      selectedLanguage: lang,
    }));
  },
  [setStateWithStatusHook]
);
```

In order, the handler:

1. **Looks up the full DTO** in `currentState.baseSites` (line 190). Uses the already-loaded DTO; **no separate `GET /baseSite/{code}` fetch happens today.** (Gate 3 of the spec plans to add this fetch — picker today does not.)
2. **Picks a language** — keep the currently selected one if `site.allowedLanguages` contains it, else `site.defaultLanguage` (lines 193-195).
3. **Optionally flips status to `'loading'`** with `selectedBaseSite` + `selectedLanguage` prefilled (lines 197-204). `withLoading` defaults to `true`.
4. **Opens the apiStore barrier** via `setCodes(site.code, lang.code)` (line 206 → `src/lib/config/apiStore.ts:14-20`). This is the call that resolves any queued `waitForBootstrap()` waiters.
5. **Persists the full DTO** to AsyncStorage key `base_site` via `setStoredBaseSite(site)` (line 208 → `baseSitesService.ts:15-17`).
6. **Persists the language** to AsyncStorage key `app_language` via `setStoredLanguage(lang)` (line 209 → `src/i18n/storage.ts:21-23`).
7. **Fires the 19-namespace translation burst** via `initI18n(lang.code, site.code)` (line 210 → `src/i18n/i18n.ts:7-25` → `loader.ts:6` → `loadAllNamespaces.ts:4` → 19× `fetchNamespace`). Confirmed in prior audit §2.3.
8. **Flips status to `'ready'`** with `selectedBaseSite` + `selectedLanguage` set (lines 212-217).

### 3.3 Where the handler is closed over — prop vs hook

- **Hook**: `useAppActions()` at `BaseSiteSelector.tsx:38`. Read inside the component body.
- **Import**: `AppContext` is imported at `BaseSiteSelector.tsx:14`: `import { useAppActions, useAppState } from '../context/AppContext';`
- **Not a prop.** `Props` at `BaseSiteSelector.tsx:17-19` is `{ isMaintenance?: boolean }` — nothing else.

**Implication for step 5:** the component is coupled to `AppContext` for the action-out. Swapping the handler to a `bootStore` equivalent requires either (a) editing `BaseSiteSelector.tsx:14, 37, 38` to read from `bootStore` hooks, (b) making `useAppActions().setBaseSiteForCode` delegate to bootStore under the hood (no source change in the component), or (c) refactoring the component to receive `baseSites` + `onPick` as props from a parent that bridges bootStore. See §5.

---

## 4. The intro → picker → done flow

### 4.1 Intro vs picker — one screen, not two

There is **no separate intro screen before the picker**. The "intro" content (welcome card with title, subtitle, description) and the picker buttons render **at the same time, in the same component, on one scroll**:

- Intro card: `BaseSiteSelector.tsx:86-113` (always rendered while the component is mounted).
- Picker buttons: `BaseSiteSelector.tsx:114-132` (rendered only when `!isMaintenance`).

When `isMaintenance` is true, the buttons block is suppressed (line 114 conditional) and the description text inside the intro card is replaced by maintenance copy (lines 96-104). When `isMaintenance` is false, the welcome description (line 107-109) and the `select.base.site` prompt (line 110) sit above the buttons.

There is no "Continue" / "Get started" affordance separating intro from picker. The user reads the intro card and taps a base-site button on the same screen.

### 4.2 Post-pick transition — status flip, no navigation

The transition after a successful pick is **purely a status flip in `AppContext`**:

1. User taps `<TouchableOpacity>` at `BaseSiteSelector.tsx:117-129`.
2. `setBaseSiteForCode(site.code)` runs (`AppContext.tsx:187-220`, traced in §3.2).
3. Final step of the handler (lines 212-217) sets `status: 'ready'` on `AppContext` state.
4. `setStateWithStatusHook` (`AppContext.tsx:70-85`) fires `onStatusChanged?.('ready')` in a `setTimeout(..., 0)`.
5. `RootLayout`'s `setStatus` updates the local mirror (`app/_layout.tsx:31, 60`).
6. The overlay branch at `app/_layout.tsx:72-80` no longer renders (its condition is `status !== 'ready'`). `BaseSiteSelector` unmounts.
7. The always-mounted `<Stack>` at `app/_layout.tsx:67` becomes visible underneath. `<AppInit />` at line 65 mounts.

**No `router.push`, `router.replace`, or any navigation call is involved in the post-pick transition.** The picker is an overlay above the always-mounted Stack; removing the overlay reveals the portal. The `<RequireBaseSite>` per-screen render gates (prior audit §5.3) then pass because `selectedBaseSite` is populated.

---

## 5. Coupling verdict

### 5.1 Verdict: the UI is internally coupled to `AppContext`

`BaseSiteSelector.tsx` reads `useAppState()` and `useAppActions()` directly inside the component body. The tap handler is not a prop; the list source is not a prop. Step 5 cannot swap the wiring to `bootStore` while leaving `BaseSiteSelector.tsx` completely untouched — **unless** the new `bootStore` is bridged into the existing `AppContext` shape (so `useAppActions().setBaseSiteForCode` and `useAppState().baseSites` keep working with their current consumer signature).

### 5.2 Every line where the picker reads from `AppContext`

Inside `src/components/init/BaseSiteSelector.tsx`:

| Line | Code | Reads from |
|------|------|------------|
| 14 | `import { useAppActions, useAppState } from '../context/AppContext';` | the AppContext module |
| 37 | `const { baseSites } = useAppState();` | `useAppState()` → `AppStateContext` |
| 38 | `const { setBaseSiteForCode } = useAppActions();` | `useAppActions()` → `AppActionsContext` |

Those three lines are the entire coupling surface to `AppContext` inside the picker.

### 5.3 Adjacent coupling above the component (not in `BaseSiteSelector` itself)

These are not lines step 5 must edit in the picker — they are facts about how the overlay decides to render the picker at all:

- `app/_layout.tsx:60`: `<AppContextProvider onStatusChanged={setStatus} ...>` — the root layout learns `status` from `AppContext` via callback.
- `app/_layout.tsx:31`: `const [status, setStatus] = useState<AppStateValue['status']>();` — the local mirror.
- `app/_layout.tsx:74, 76`: the overlay chooses `<BaseSiteSelector isMaintenance />` vs `<BaseSiteSelector isMaintenance={false} />` based on `status === 'maintenance'` vs `status === 'select-base-site'`.

Step 5 must decide whether the new `bootStore.status` feeds the overlay through the existing `AppStateValue['status']` (no `_layout.tsx` change) or directly via `useBootStore()` (one site to edit in `_layout.tsx`, plus mapping the new status values `'booting' | 'maintenance' | 'update-required' | 'intro-picker' | 'updating' | 'ready'` to the old `'loading' | 'maintenance' | 'select-base-site' | 'ready'` used by the overlay branches). This is a wiring choice, not a component-internal one.

### 5.4 In-component data dependency that is NOT AppContext

For completeness (see §2.4):

- `BaseSiteSelector.tsx:1` — `import { fetchNamespace } from '@/i18n/fetchNamespace';`
- `BaseSiteSelector.tsx:45` — `fetchNamespace(TranslationNamespace.INTRO, 'sr').then(...)`

These are an independent service call from inside the component. Step 5 does not need to touch them to swap the picker's data-in / action-out wiring, but is free to fold this fetch into Gate 3 if desired.

### 5.5 Summary table for step 5

| Concern | Currently coupled to | Component-internal line(s) | Swap surface if bootStore is bridged into AppContext | Swap surface if step 5 reads bootStore directly |
|---|---|---|---|---|
| List source (`baseSites`) | `useAppState()` | `BaseSiteSelector.tsx:37` | none in the component | line 37 + import on line 14 |
| Tap handler (`setBaseSiteForCode`) | `useAppActions()` | `BaseSiteSelector.tsx:38` | none in the component | line 38 + import on line 14 |
| Overlay status decision | `AppStateValue['status']` via `onStatusChanged` callback | n/a (lives in `_layout.tsx:31, 60, 72-80`) | none in the component or layout | `_layout.tsx:31, 60, 72-80` |
| Intro translations | direct `fetchNamespace(INTRO, 'sr')` | `BaseSiteSelector.tsx:1, 45` | unchanged (orthogonal to AppContext seam) | unchanged |

**Bottom line.** The existing intro+picker UI can be driven by the new `bootStore` without editing `BaseSiteSelector.tsx` **only** if `AppContext` keeps exposing a `baseSites` array and a `setBaseSiteForCode(code)` action with the same call signatures (proxying to bootStore underneath). If step 5 instead has the picker read from `bootStore` directly, the swap surface inside the component is exactly the three lines in §5.2 (lines 14, 37, 38), plus the overlay-status site in `_layout.tsx` listed in §5.3.

---

## Adjacent observations (flagged for Mastermind, out of scope for this audit)

- **`BaseSiteSelector` ActivityIndicator early-return at line 71-73.** While `introTranslations` is unresolved, the component renders bare `<ActivityIndicator />` with no surrounding view. Under the current overlay (`_layout.tsx:73`'s `bg-background` `items-center justify-center` wrapper) it appears centered, but the picker has no fallback if `fetchNamespace(INTRO, 'sr')` hangs — the spinner is indefinite. Low severity today (the INTRO fetch is one of the 19 boot translations and tends to be fast), but if step 5 expects the picker to drive its own data with its own timeout, this is the slot where a timeout/retry would live. File: `src/components/init/BaseSiteSelector.tsx:42-54, 71-73`.

- **`fetchNamespace(INTRO, 'sr')` is hardcoded to `'sr'`.** `BaseSiteSelector.tsx:45`. This is deliberate because no language is selected yet, but if intro translations should ever follow device-locale before pick, this is the call site that hardcodes the fallback. Low severity. Out of scope for the seam.

- **The two render variants of `BaseSiteSelector` (maintenance vs picker) share the welcome card.** `BaseSiteSelector.tsx:86-113`. The component is doing double duty: it is the maintenance screen AND the picker. The boot redesign spec separates `maintenance` and `intro-picker` as distinct states (`features/expo-boot-redesign.md` Part 2). Whether to split this one component into two or keep the shared shell is a design call for step 5 / step 7; the seam audit just notes that the current single-component design is what step 5 inherits. Low severity, structural observation only.

- **No `GET /baseSite/{code}` fetch happens today on pick.** The picker already has the full `BaseSiteDTO` (loaded via `/baseSite/details` in `bootstrap`) and uses `Array.find` to pick it (`AppContext.tsx:190`). The spec's Gate 3 plans to swap data-in to `/baseSite/overviews` and add a per-pick `GET /baseSite/{code}` to fetch the full DTO. The handler in `setBaseSiteForCode` will need that fetch step added (or moved into a bootStore action). Not in scope for this audit; flagged for step 5 brief drafting.

---
