# Audit — Post-Pick Consumer Surface (AppContext vs bootStore)

> **Archival note (added 2026-05-29 by Docs/QA at feature close — read before trusting this audit's store column):** This audit maps consumers of the OLD `AppContext` path (`useAppState` / `useAppActions`). That attribution is now **stale**: step 7b of the `expo-boot-redesign` feature re-pointed every one of these consumers to `useBootStore`, and `AppContext.tsx` was deleted. The per-file `useTranslations` facts and the mount-tree mapping below still hold; only the store-attribution column is out of date. Where this audit says `AppContext`, the code now reads `bootStore`. Trust the code over this column. See [decisions.md](../decisions.md) 2026-05-29 entry.

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-29
**Type:** read-only audit. No code changes, no design proposals, no A/B recommendation.
**Brief:** `.agent/brief.md` (post-pick-consumers)

## Scope note

This audit maps every in-app consumer of the OLD path (`AppContext` via
`useAppState` / `useAppActions`) so Mastermind can size Option A (re-point
consumers to bootStore) vs Option B (bridge state from `pickBaseSite` + Gate 3
into AppContext). It does NOT recommend A or B.

The working hypothesis in the brief is **confirmed by the code** (evidence in §0
and §4). The divergence is real and in-memory: `bootStore.pickBaseSite` populates
`bootStore.selectedBaseSite` + `language` and persists to AsyncStorage, but never
writes to `AppContext`. The portal chrome and product lists read
`AppContext.selectedBaseSite`, which on a fresh install stays `undefined` until
the next cold boot.

---

## 0. Confirmation of the divergence (the mechanism)

`app/_layout.tsx` mounts BOTH systems side by side (the 7a "preserve-old-path" rule):

- `app/_layout.tsx:40-43` — the ONE empty-deps effect calls `useBootStore.getState().start()`.
- `app/_layout.tsx:47` — the overlay reads `bootStore.status`.
- `app/_layout.tsx:76` — `<AppContextProvider onLanguageChanged={setLocale}>` wraps the whole tree; AppContext runs its own `bootstrap()` independently (`AppContext.tsx:155-156`).
- `app/_layout.tsx:96-102` — when `bootStatus !== 'ready'` the overlay covers the portal; `intro-picker` renders `<BaseSiteSelector>`.

`src/components/init/BaseSiteSelector.tsx:38,119` wires the picker tap to the NEW path:

```tsx
const setBaseSiteForCode = useBootStore((s) => s.pickBaseSite);
...
onPress={() => setBaseSiteForCode(baseSite.code)}
```

`src/lib/store/bootStore.ts:240-266` — `pickBaseSite` writes ONLY bootStore + AsyncStorage:

```ts
set({ selectedBaseSite: fullDto, language: lang });
await setStoredBaseSite(fullDto);
await setStoredLanguage(lang);
await get().runFreshnessGate();   // → set({ status: 'ready' }) at line 451
```

No write to AppContext anywhere in this path. AppContext's own `bootstrap`
(`AppContext.tsx:108-117`) only populates `selectedBaseSite` when a STORED site
already exists at boot — on a fresh install it returns at `status:'select-base-site'`
with `selectedBaseSite` undefined. So after a first-install pick: `bootStore.status`
flips to `ready`, the overlay clears, the portal paints — but every consumer below
still reads `AppContext.selectedBaseSite === undefined`. On a later cold restart
the stored site exists, AppContext.bootstrap reads it, both stores agree, symptom
gone. This matches the brief exactly.

---

## 1. What the OLD path (AppContext) exposes

Source: `src/components/context/AppContext.tsx:20-34`. Verbatim:

```ts
export type AppStateValue = {
  status: 'loading' | 'maintenance' | 'select-base-site' | 'ready';
  baseSites: BaseSiteDTO[];
  selectedBaseSite?: BaseSiteDTO;
  selectedLanguage?: LanguageDTO;
  configuration?: ConfigMap;
  getCurrentLocale?: () => string;
};

export type AppActionsValue = {
  setBaseSiteForCode: (siteCode: string, withLoading?: boolean) => Promise<void>;
  setLanguageForCode: (langCode: string) => Promise<void>;
  getConfiguration: (key: string) => string | undefined;
  reBootstrap: () => Promise<void>;
};
```

**State fields (6):** `status`, `baseSites`, `selectedBaseSite`, `selectedLanguage`, `configuration`, `getCurrentLocale`.
**Actions (4):** `setBaseSiteForCode`, `setLanguageForCode`, `getConfiguration`, `reBootstrap`.

Hooks exposed: `useAppState()` (`AppContext.tsx:279`), `useAppActions()` (`AppContext.tsx:285`). There is no `useAppContext` export and no consumer calls `useContext(AppStateContext|AppActionsContext)` directly — all consumption is through these two hooks. (`useContext` is used only internally at `AppContext.tsx:280,286`.)

---

## 2. useAppState / useAppActions call-site inventory

Exhaustive grep of `src/` and `app/`. Two definition files excluded as non-consumers:
`AppContext.tsx` (the provider) is the source; every other hit is a consumer
(including `RequireBaseSite.tsx`, which is itself a consumer — see §3).

### 2a. `useAppState()` read sites (25 sites)

| # | File | Line | Destructured | Use |
|---|------|------|--------------|-----|
| 1 | `src/components/context/RequireBaseSite.tsx` | 11 | `selectedBaseSite` | Gate: renders `fallback` (default `null`) when no base site; else children. See §3. |
| 2 | `src/components/basic/CategorySelector.tsx` | 60 | `selectedBaseSite` | `catalog.categories` to build category hierarchy / search (72, 129-133, 212). |
| 3 | `src/components/navigation/CategoryNavigation.tsx` | 22 | `selectedBaseSite` | Null-check (32) + `catalog` categories for nav menu (40). |
| 4 | `src/components/navigation/Filters.tsx` | 40 | `selectedBaseSite` | `catalog.topFilters` + category filters; null-check (61-88). |
| 5 | `src/components/user/UserMenu.tsx` | 31 | `selectedBaseSite` | Compare `.domain` to `user.baseSite.domain` (51) → conditional switch. |
| 6 | `src/components/product/ProductList.tsx` | 49 | `selectedLanguage` | Refetch products when `selectedLanguage?.code` changes (102, 125-139). |
| 7 | `src/components/dialog/dialogs/CurrencySelectionDialog.tsx` | 21 | `selectedBaseSite` | `allowedCurrencies` to render currency buttons (29). |
| 8 | `src/components/SearchInput.tsx` | 49 | `selectedBaseSite` | Categories for search placeholder (64-68, 110-125). |
| 9 | `src/components/dialog/dialogs/FiltersDialog.tsx` | 37 | `selectedBaseSite` | `catalog.topFilters` advanced filters; null-check (63-90). |
| 10 | `src/components/filters/ProductOrder.tsx` | 20 | `selectedBaseSite` | `catalog.orderTypes` to render order options (26-47). |
| 11 | `src/components/navigation/Footer.tsx` | 17 | `selectedBaseSite` | `catalog?.categories` for footer links (32-33). |
| 12 | `src/components/product/ProductBreadcrumb.tsx` | 13 | `selectedBaseSite` | `catalog.categories` to resolve breadcrumb chain (17-21, 43). |
| 13 | `src/components/dialog/dialogs/PreviewProductDialog.tsx` | 38 | `selectedBaseSite` | `baseSiteOverview` field in preview/details data (108, 130). |
| 14 | `src/components/dialog/dialogs/PortalConfigDialog.tsx` | 39 | `selectedBaseSite`, `selectedLanguage`, `baseSites` | Language + base-site switcher UI (see §5, §7). |
| 15 | `src/components/ConsumerProtectionBanner.tsx` | 14 | `selectedBaseSite` | Compare `.domain` to user's baseSite domain → banner visibility (19-26). |
| 16 | `src/components/filters/RegionCityFilter.tsx` | 22 | `selectedBaseSite` | `regions` list (29 null-check, 90 map). |
| 17 | `src/components/basic/CitySelector.tsx` | 31 | `selectedBaseSite` | `regions`/cities section list (48-60). |
| 18 | `src/components/navigation/TopBar.tsx` | 28 | `selectedBaseSite` | Header icon: `iconId` → `DynamicIcon`, else `code` uppercase (36-42). |
| 19 | `src/components/dialog/dialogs/product-creation/AddUpdateProductDialog.tsx` | 29 | `selectedBaseSite` | `catalog` top filters for initial filters (38-56, 73). |
| 20 | `src/components/filters/CurrencySelector.tsx` | 19 | `selectedBaseSite` | `defaultCurrency` to init price-range state (28-31). |
| 21 | `app/owner/_layout.tsx` | 21 | `selectedBaseSite` | `labelKey` for "Marketplace {…}" header label (35). |
| 22 | `app/(portal)/_layout.tsx` | 12 | `selectedBaseSite` | **Gates the entire portal header chrome** (TopBar + CategoryNavigation + banner) on truthiness (16-25). See §4. |
| 23 | `app/(portal)/(public)/catalog/[...categories].tsx` | 21 | `selectedBaseSite` | `catalog` categories-from-path (25-28). Also wraps content in `<RequireBaseSite>` (60-80). |
| 24 | `app/(portal)/(public)/product/[...productData].tsx` | 48 | `selectedBaseSite`, `status` | Category resolution (68-70); `status` for boot-state checks (100, 150). Also wraps content in `<RequireBaseSite>` (37-39). |
| 25 | `app/owner/dashboard/products/[productId].tsx` | 41 | `selectedBaseSite` | `allowedCurrencies` (291) + passed to `MetaDataProduct` (324); guard (203). |

### 2b. `useAppActions()` call sites (5 sites)

| # | File | Line | Destructured | Use |
|---|------|------|--------------|-----|
| A1 | `src/components/user/UserMenu.tsx` | 32 | `setBaseSiteForCode` | Switch base site when user's `baseSite.domain` differs (52). |
| A2 | `src/components/dialog/dialogs/product-creation/BasicInfoProductDialog.tsx` | 43 | `getConfiguration` | Read 6 regex-validation patterns for product validation (49-54). |
| A3 | `src/components/dialog/dialogs/PortalConfigDialog.tsx` | 40 | `setBaseSiteForCode`, `setLanguageForCode` | Base-site switch (115) + language switch (95). See §7. |
| A4 | `app/(portal)/(public)/product/[...productData].tsx` | 49 | `setBaseSiteForCode` | Switch base site if product response carries a different `baseSiteOverview.code` (114). |
| A5 | `app/owner/dashboard/products/[productId].tsx` | 42 | `getConfiguration` | Read 6 regex-validation patterns (58-63). |

> Note on `BasicInfoProductDialog` (A2): it does NOT call `useAppState`. It
> receives `selectedBaseSite` as a **prop** (line 29/37). Its only AppContext
> coupling is `getConfiguration`.

### 2c. Grouped by field/action

**By state field read:**

| Field | # files | Files |
|-------|---------|-------|
| `selectedBaseSite` | **24** | #1–5, 7–25 (every `useAppState` site except ProductList) |
| `selectedLanguage` | **2** | ProductList (#6), PortalConfigDialog (#14) |
| `baseSites` | **1** | PortalConfigDialog (#14) |
| `status` | **1** | product/[...productData] (#24) |
| `configuration` (field) | **0** | — read indirectly via `getConfiguration` action only |
| `getCurrentLocale` | **0** | no consumer reads it |

**By action called:**

| Action | # files | Files |
|--------|---------|-------|
| `setBaseSiteForCode` | **3** | UserMenu (A1), PortalConfigDialog (A3), product/[...productData] (A4) |
| `setLanguageForCode` | **1** | PortalConfigDialog (A3) |
| `getConfiguration` | **2** | BasicInfoProductDialog (A2), owner/dashboard/products/[productId] (A5) |
| `reBootstrap` | **0** | exposed (`AppContext.tsx:33,261`) but no consumer calls it |

---

## 3. The `<RequireBaseSite>` component

Location: `src/components/context/RequireBaseSite.tsx`. Full body (lines 1-14, verbatim):

```tsx
import { useAppState } from '@/components/context/AppContext';
import { ReactNode } from 'react';

export function RequireBaseSite({
  children,
  fallback = null,
}: {
  children: ReactNode;
  fallback?: ReactNode;
}) {
  const { selectedBaseSite } = useAppState();
  if (!selectedBaseSite) return <>{fallback}</>;
  return <>{children}</>;
}
```

- **Reads:** `AppContext.selectedBaseSite` (via `useAppState`).
- **Gates on:** truthiness of `selectedBaseSite`. When falsy it renders `fallback`
  (default `null` → renders nothing).
- It is itself a consumer (counted as site #1 above).

**Screens wrapping content in `<RequireBaseSite>` (5, confirmed against current code):**

| Screen | Lines | Wraps |
|--------|-------|-------|
| `app/(portal)/(public)/index.tsx` | 10-19 | `<View>` → `<FilteredProductList … CardContentComponent={PortalProductCard}>` (the **portal home product list**) |
| `app/(portal)/(public)/catalog/[...categories].tsx` | 60-80 | `<FilteredProductList>` + empty-state `NoProdctsComponent` |
| `app/(portal)/(public)/product/[...productData].tsx` | 37-39 | `<ProductScreenContent />` |
| `app/(portal)/(public)/user/[...userData].tsx` | 16-18 | `<UserScreenContent />` |
| `app/(portal)/(public)/blog/free-zone.tsx` | 128-130 | `<ExtraProductsList extraSections={[MORE_FROM_FREE_CATEGORY]} />` |

This is the prior-audit reference (`audit-expo-boot-redesign` §5.3 / `audit-picker-seam` §5) confirmed current: 5 portal screens, each gating its product/content on `AppContext.selectedBaseSite`.

---

## 4. The header + product screens that fail the live test

Two load-bearing consumers explain the reported "no header / no products" symptom directly:

### Header chrome — `app/(portal)/_layout.tsx:12-25`

```tsx
const { selectedBaseSite } = useAppState();
...
{selectedBaseSite && (
  <View …>
    <ConsumerProtectionBanner … />
    <TopBar … />
    <CategoryNavigation … />
  </View>
)}
```

- **Needs:** `AppContext.selectedBaseSite`.
- **Reads from:** AppContext (`useAppState`, line 12).
- The entire header block (banner + `TopBar` + `CategoryNavigation`) is conditional
  on `selectedBaseSite`. With AppContext at `undefined`, **none of the header renders**.

`TopBar` itself (`src/components/navigation/TopBar.tsx:28`) also reads
`AppContext.selectedBaseSite` (for `iconId`/`code`, lines 36-42) — so even if it
mounted it would have no site.

### Portal product list — `app/(portal)/(public)/index.tsx:10-19` via `RequireBaseSite`

```tsx
<RequireBaseSite>
  <View style={{ flex: 1 }}>
    <FilteredProductList useFilterStore={usePortalFilterStore}
      fetchPage={getPortalProducts} applyRandom={true}
      CardContentComponent={PortalProductCard} />
  </View>
</RequireBaseSite>
```

- **Needs:** `AppContext.selectedBaseSite` (read inside `RequireBaseSite`).
- **Reads from:** AppContext.
- With AppContext `undefined`, `RequireBaseSite` returns `null` → **no product list**.

Together these two are the visible failure: empty header (gate in `(portal)/_layout.tsx`)
and empty product area (`RequireBaseSite` in `index.tsx`). Both read AppContext, which
the NEW pick path never populated.

---

## 5. Language / i18n consumption (non-i18n consumers of the language slot)

i18n itself is served by the `i18n` singleton / `useTranslations` (`i18n.t`) and is
registered by bootStore Gate 4 (`bootStore.ts:420-446`) — out of scope per brief.
The **non-i18n** consumers that read `AppContext.selectedLanguage` directly are exactly **two**:

| File | Line | Read | Use |
|------|------|------|-----|
| `src/components/product/ProductList.tsx` | 49 | `selectedLanguage` | NOT rendering — used as a reactive trigger. Mount guard at 102; on `selectedLanguage?.code` change (125-139) it refetches the product list so cards re-render in the new language. |
| `src/components/dialog/dialogs/PortalConfigDialog.tsx` | 39 | `selectedLanguage` | Rendering — compares `selectedLanguage.code` to each `lang.code` to highlight the currently-active language in the switcher (93-100). |

No consumer reads `getCurrentLocale` (the locale-string accessor). All other
"language-dependent" rendering across the app goes through `useTranslations` /
`i18n.t`, not through the AppContext language slot.

---

## 6. The opposite direction — what writes `selectedBaseSite` / `selectedLanguage` today, and its side effects

Source: `AppContext.tsx`. Both action bodies do MORE than set state — relevant to Option B bridge sizing.

### `setBaseSiteForCode` (lines 184-215, verbatim)

```ts
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

**Side effects beyond setting state:**
- Resolves `site` from the **in-memory `baseSites` array** (full `BaseSiteDTO` list — not overviews). If the code isn't in `baseSites`, it silently returns (no fetch).
- Computes `lang` (keep current language if allowed by the new site, else `site.defaultLanguage`) — same rule as `bootStore.pickBaseSite:254-258`.
- `setStoredBaseSite(site)` — persists DTO to AsyncStorage.
- `setStoredLanguage(lang)` — persists language to AsyncStorage.
- `initI18n(lang.code, site.code)` — registers translations into i18n. **(bootStore defers this to Gate 4 via `addResourceBundle`; the two i18n-load mechanisms differ.)**
- Drives `status` through `loading` → `ready` (the AppContext status machine).
- **No `setCodes`** (confirmed gone post-7a.1 — not present in this body).
- No navigation.

### `setLanguageForCode` (lines 217-246, verbatim)

```ts
const setLanguageForCode = useCallback(
  async (langCode: string) => {
    const currentState = stateRef.current;
    const site = currentState.selectedBaseSite;
    if (!site) return;

    const lang = site.allowedLanguages.find((l) => l.code === langCode);
    if (!lang) return;

    setStateWithStatusHook((prev) => ({
      ...prev,
      status: 'loading',
    }));

    await setStoredLanguage(lang);
    await initI18n(lang.code, site.code);

    const newLocale = `${site.code}-${lang.code}`;

    onLanguageChangedRef.current?.(newLocale);

    setStateWithStatusHook((prev) => ({
      ...prev,
      status: 'ready',
      selectedLanguage: lang,
      getCurrentLocale: () => newLocale,
    }));
  },
  [setStateWithStatusHook]
);
```

**Side effects beyond setting state:**
- Requires an already-selected `selectedBaseSite`; resolves `lang` from its `allowedLanguages`.
- `setStoredLanguage(lang)` — persists to AsyncStorage.
- `initI18n(lang.code, site.code)` — re-registers translations for the new language.
- Fires `onLanguageChangedRef.current?.(newLocale)` — the `onLanguageChanged` prop, wired in `app/_layout.tsx:76` to `setLocale` (the formatter/locale state). This is an extra cross-system effect a bridge would have to account for.
- Sets `getCurrentLocale` to a fresh closure.
- Drives `status` `loading` → `ready`.

**Implication for Option B (factual, not a recommendation):** a bridge that mirrors
the pick into AppContext cannot be a plain `setState` — `setBaseSiteForCode` resolves
its site from the in-memory `baseSites` list (which the boot/picker path does not
populate — it uses `baseSiteOverviews`), and both actions also persist to AsyncStorage,
call `initI18n`, drive the AppContext status machine, and (for language) fire
`onLanguageChanged`. Several of these effects (AsyncStorage persistence, i18n
registration) are ALSO already performed by the bootStore path, so a naive bridge
would double-persist / double-register.

---

## 7. Stored-language switching at runtime

**Yes — runtime language switching exists, and it routes through `AppContext.setLanguageForCode`.**

- `src/components/dialog/dialogs/PortalConfigDialog.tsx` is the switcher. Line 40
  destructures `{ setBaseSiteForCode, setLanguageForCode }`; line 95 calls
  `setLanguageForCode(lang.code)` when the user taps a language; line 115 calls
  `setBaseSiteForCode(baseSite.code)` for base-site switching.
- This dialog is opened from the header (it's the portal config dialog — the same
  switcher the header chrome surfaces). It reads `selectedBaseSite.allowedLanguages`,
  `selectedLanguage`, and `baseSites` from AppContext (§2a #14).
- `setLanguageForCode` is exposed only here; `setBaseSiteForCode` is also called by
  `UserMenu` (A1) and the product screen (A4).

There is no other runtime language switcher (no settings screen, no separate header
toggle). The only path is `PortalConfigDialog → AppContext.setLanguageForCode`. So
runtime language switching IS in scope for whichever fix is chosen: it currently
relies entirely on the AppContext action + the `selectedLanguage` reactivity that
`ProductList` (§5) keys off of.

---

## 8. Size summary

- **N_components** (distinct files reading `useAppState` and/or `useAppActions`): **26**
  (25 `useAppState` files — including `RequireBaseSite.tsx` — plus `BasicInfoProductDialog.tsx`,
  which uses only `useAppActions`. The four other `useAppActions` files already appear
  in the `useAppState` set.)
- **N_state_reads** (total `useAppState()` call sites): **25**
- **N_action_calls** (total `useAppActions()` call sites): **5**

Field-level read distribution (from §2c): `selectedBaseSite` = 24 files,
`selectedLanguage` = 2, `baseSites` = 1, `status` = 1, `configuration`/`getCurrentLocale` = 0.
Action-level: `setBaseSiteForCode` = 3, `setLanguageForCode` = 1, `getConfiguration` = 2,
`reBootstrap` = 0.

---

## Ambiguities / caveats (stated plainly)

- `RequireBaseSite.tsx` is counted as a consumer (it reads `useAppState`). If
  Mastermind prefers to count it as infrastructure rather than a "component," subtract
  1 from N_components (→ 25) and 1 from N_state_reads (→ 24). It is included here
  because it literally calls `useAppState` and is the gate behind the symptom in §4.
- `getConfiguration` (action, 2 sites) and `reBootstrap` (0 sites) are NOT base-site/
  language reads — they're listed for completeness because they ride the same
  `useAppActions` hook. `configuration` is sourced from `getAppConfiguration()` during
  AppContext bootstrap; it is independent of the base-site pick and is not part of the
  divergence. If Option A re-points only base-site/language consumers, the 2
  `getConfiguration` sites and the `configuration` field are a separate concern.
- `BasicInfoProductDialog` receives `selectedBaseSite` as a prop, not from the hook —
  its parent (the product-creation flow) is the actual `selectedBaseSite` reader feeding
  it; that parent is among the 24 above only if it reads via `useAppState` (the dialog
  itself does not).
- All 28 analyzed files were read directly this session (per-file, line-verified).
  The only `useAppState`/`useAppActions` hits in the repo outside this set are the
  definitions in `AppContext.tsx` itself.
