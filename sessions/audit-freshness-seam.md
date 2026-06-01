# Audit — Freshness seam (Gate 4 inputs)

**Repo:** `oglasino-expo`
**Branch:** `new-expo-dev`
**Date:** 2026-05-28
**Type:** read-only audit
**Scope:** map how translations are fetched, cached, checksum-tracked, and loaded into i18n today, plus the catalog-checksum side and any `/versions` client. Output drives `expo-boot-redesign` step 6 (Gate 4). No code changes.

---

## 1. The `/versions` endpoint — client side

### 1.1 Existing client calls

**None.** Grep across `src/` and `app/`:

- `/public/versions` — zero matches.
- `/versions` — one match only: a comment inside the Gate 4 stub at `src/lib/store/bootStore.ts:254` (`// GET /versions with 5s timeout. All checksums match → set 'ready'.`).
- `checksum` — same one match (line 254 comment); zero matches anywhere else in the working tree.

The only references to "version" in the runtime translation/catalog area are:

- `src/lib/services/appVersionConfigService.tsx:9-13` — `GET /public/app/version/${Platform.OS}?currentVersion=...` (the app-binary force/optional-update endpoint, unrelated to per-namespace checksums).
- `src/i18n/storage.ts:5` — `VERSION: 'translations_version'` AsyncStorage key (inert; see §4).
- `src/i18n/loader.ts:4,7` — `const VERSION_KEY = 'translations_version'` and `const storedVersion = await AsyncStorage.getItem(VERSION_KEY)` (value discarded).

**Conclusion:** no client code consumes `GET /api/public/versions` today. Step 6 will add the service function, the caller in `runFreshnessGate`, and the response TypeScript type.

### 1.2 Matching TS response type

The backend feature ([`features/version-checksums.md`](../../oglasino-docs/features/version-checksums.md) §4.1, §5) ships the response shape:

```json
{
  "translations": { "<22 namespaces>": "<16-hex>" },
  "catalog":      { "rs": "<16-hex>", "rsmoto": "<16-hex>", "me": "<16-hex>" }
}
```

`src/i18n/types.ts` declares two version-flavoured types, but neither matches:

```ts
// src/i18n/types.ts:4-11
export interface TranslationResponse {
  version: string;
  translations: TranslationKeys;
}

export interface VersionResponse {
  version: string;
}
```

These shapes (single `version: string`) describe the OLD per-translation-call version field, not the new `/versions` checksum maps. Neither has any callers (`grep -RIn 'VersionResponse\|TranslationResponse'` returns only the declarations).

**Conclusion:** no matching TS type exists. Step 6 must add a new type (e.g. `VersionsResponse = { translations: Record<TranslationNamespace, string>; catalog: Record<BaseSiteCode, string> }`) and may delete the inert `VersionResponse` / `TranslationResponse` interfaces in the same pass (flagged in §8 for Mastermind, since deletion is out of audit scope).

---

## 2. Translation fetch — the per-namespace path

### 2.1 `fetchNamespace` — exact code

`src/i18n/fetchNamespace.ts:26-48`:

```ts
export async function fetchNamespace(
  namespace: TranslationNamespace,
  lang: string
): Promise<Record<string, any>> {
  try {
    const res = await BACKEND_API.get<TranslationDTO[]>(
      `/public/translations?namespace=${namespace}&lang=${lang}`
    );

    if (res.status === 200 && res.data) {
      const flat: Record<string, string> = {};
      res.data.forEach((t) => {
        flat[t.key] = t.value;
      });

      return toNested(flat);
    }

    return undefined;
  } catch (error) {
    return undefined;
  }
}
```

- **Signature:** `(namespace: TranslationNamespace, lang: string) => Promise<Record<string, any>>`. The return type annotation is `Promise<Record<string, any>>`, but every non-success branch returns `undefined` — minor type drift (callers tolerate it; see §3.2).
- **Endpoint:** `GET /public/translations?namespace=${namespace}&lang=${lang}` (line 32). No request body, no headers other than what the shared axios interceptor injects.
- **Params:** both `namespace` and `lang` are required query params (string concatenation, no encoding — values are enum keys and BCP-47-ish lang codes, both safe).
- **Return shape:** the backend emits `TranslationDTO[]` (line 4: `type TranslationDTO = { key: string; value: string }`). `fetchNamespace` flattens that to `Record<string, string>` and then runs `toNested` (lines 9-24) which splits dotted keys into a nested object tree — the i18next-conventional shape that `i18n.init({ resources })` expects per namespace.
- **Fail behavior:** swallows both the `try` body's non-200 and the `catch` — returns `undefined` on any failure. There is no throw, no logging, no toast. The caller (`loadAllNamespaces`) stores the `undefined` into the resources map for that namespace, which then gets registered into i18next as an `undefined` value — components depending on that namespace would silently fall through to keys.

### 2.2 Per-namespace and per-language

The fetch takes **both** a `namespace` and a `lang` — one call returns one namespace for one language. The backend exposes the same wire shape that web consumes (per Part 8 architectural defaults).

### 2.3 `TranslationNamespace` enum — count discrepancy

`src/i18n/types.ts:18-50`:

```ts
export enum TranslationNamespace {
  // CORE
  COMMON = 'COMMON',
  COMMON_SYSTEM = 'COMMON_SYSTEM',
  ERRORS = 'ERRORS',
  VALIDATION = 'VALIDATION',
  PAGING = 'PAGING',

  // UI
  BUTTONS = 'BUTTONS',
  INPUT = 'INPUT',
  DIALOG = 'DIALOG',

  // LAYOUT
  HEADER = 'HEADER',
  FOOTER = 'FOOTER',
  NAVIGATION = 'NAVIGATION',

  // GLOBAL FEATURES
  INTRO = 'INTRO',
  EXTRA_PRODUCTS = 'EXTRA_PRODUCTS',
  COOKIES = 'COOKIES',

  // PAGES
  MESSAGES_PAGE = 'MESSAGES_PAGE',
  DASHBOARD_PAGES = 'DASHBOARD_PAGES',
  ABOUT_PAGE = 'ABOUT_PAGE',
  FREE_ZONE_PAGE = 'FREE_ZONE_PAGE',
  PRICING_PAGE = 'PRICING_PAGE',

  // META
  METADATA = 'METADATA',
}
```

**Count: 20 entries.** Verified mechanically: `grep -c "^\s*[A-Z_]* = '" src/i18n/types.ts` → `20`.

The backend's [`features/version-checksums.md`](../../oglasino-docs/features/version-checksums.md) §4.1 (lines 108-129) lists **22 namespaces** that `/api/public/versions` returns checksums for. The mobile enum is missing two of them:

- `ADMIN_PAGES` — intentionally absent post-admin-removal (chat α, 2026-05-24). Mobile no longer has admin code; the namespace exists only on web. Mobile must skip it; comparing the backend's `ADMIN_PAGES` checksum is wasted bytes (decisive but harmless).
- `BACKEND_TRANSLATIONS` — by design: per conventions Part 6, `BACKEND_TRANSLATIONS` holds strings the backend emits directly to the user (push notification bodies, email content), with no client-side translation step. Mobile has no reason to fetch them.

The prior audit (`audit-expo-boot-redesign.md` §2.3) refers to "the 19 `TranslationNamespace` enum values" — that count is also off by one (the real count is 20). Both audits and the spec must agree on the number before step 6 — flagged in §8.

**Implication for Gate 4:** the freshness gate cannot iterate the backend's 22-key `translations` map and try to fetch all of them; it must intersect with the local `TranslationNamespace` enum (20 values) and only compare/refetch those. The mismatch is on backend keys mobile does not consume — never a missing-on-mobile namespace.

---

## 3. The 19/20-namespace burst — how `initI18n` works today

### 3.1 Trace

`src/i18n/i18n.ts:1-25`:

```ts
import i18n from 'i18next';
import ICU from 'i18next-icu';
import { initReactI18next } from 'react-i18next';
import { loadTranslations } from './loader';
import { TranslationNamespace } from './types';

export async function initI18n(lang: string, tenant: string) {
  const resources = await loadTranslations(lang, tenant);

  await i18n
    .use(ICU)
    .use(initReactI18next)
    .init({
      lng: lang,
      fallbackLng: 'sr',
      ns: Object.keys(resources),
      defaultNS: TranslationNamespace.COMMON_SYSTEM,
      resources: {
        [lang]: resources,
      },
    });
}
```

`src/i18n/loader.ts:1-26`:

```ts
import AsyncStorage from '@react-native-async-storage/async-storage';
import { loadAllNamespaces } from './loadNamespaces';

const VERSION_KEY = 'translations_version';

export async function loadTranslations(lang: string, tenant: string) {
  const storedVersion = await AsyncStorage.getItem(VERSION_KEY);
  // ...commented-out cache-check scaffolding (see §4.2)...

  const resources = await loadAllNamespaces(lang);

  // ...commented-out cache-write scaffolding...

  return resources;
}
```

`src/i18n/loadNamespaces.ts:1-17`:

```ts
import { fetchNamespace } from './fetchNamespace';
import { TranslationNamespace } from './types';

export async function loadAllNamespaces(lang: string) {
  const namespaces = Object.values(TranslationNamespace);

  const results = await Promise.all(namespaces.map((ns) => fetchNamespace(ns, lang)));

  const resources: Record<string, any> = {};

  namespaces.forEach((ns, i) => {
    resources[ns] = results[i];
  });

  return resources;
}
```

**How many, for which language:** every cold start fires `Promise.all` of `fetchNamespace(ns, lang)` over `Object.values(TranslationNamespace)` → **20 parallel `GET /public/translations?namespace=<NS>&lang=<LANG>` requests** for the single active language. Not for all `allowedLanguages` — just the one selected by `bootstrap`.

**How it loads into i18n:** `initI18n` calls `i18n.use(ICU).use(initReactI18next).init({ ns, defaultNS, resources: { [lang]: resources }, ... })`. The mechanism is `init`, not `addResourceBundle`. There are **zero** uses of `i18n.addResourceBundle`, `i18n.addResource`, or any other incremental register API anywhere in `src/` or `app/` (grep `addResourceBundle\|i18n\.init\|i18next` confirmed). `init` is called once per `initI18n` invocation — and `initI18n` is invoked from three sites that all re-do the full burst (see §3.3).

### 3.2 Fetch vs register — separating the two operations

The brief's critical separation:

- **FETCH** (gets a namespace's strings from the backend):
  - `fetchNamespace(ns, lang)` at `src/i18n/fetchNamespace.ts:26` is the single per-namespace fetch primitive. Called only by `loadAllNamespaces` (`src/i18n/loadNamespaces.ts:7`).
  - No incremental "fetch one namespace" caller exists outside `loadAllNamespaces`.

- **REGISTER** (puts strings into the i18n instance so `t()` resolves them):
  - Today, the only register call is `i18n.init({ resources: { [lang]: resources }, ... })` at `src/i18n/i18n.ts:13`, which atomically registers the entire result of `loadAllNamespaces`.
  - There is no separate `register(namespace, lang, payload)` function. `init` does both "configure the instance" and "register bundles" in one call.

**For step 6:** the FETCH side is separable (use `fetchNamespace` per stale namespace). The REGISTER side is NOT separable today — there is no `addResourceBundle` call site to mirror. Step 6 must EITHER:

1. introduce per-namespace `i18n.addResourceBundle(lang, ns, payload, true /* deep */, true /* overwrite */)` calls for stale namespaces, plus a one-time `i18n.init` for the first ready transition; OR
2. always rebuild the full per-language `resources` object from local cache (loading every namespace from AsyncStorage, replacing only the stale ones with freshly-fetched payloads) and call `i18n.init` once per active-language-change.

The choice is a step-6 design decision, not an audit finding. Either way, `initI18n`'s current "one shot, everything from network" body is what step 6 replaces.

### 3.3 `initI18n` call sites — when the burst re-fires today

Three callers, all in `src/components/context/AppContext.tsx`:

- `AppContext.tsx:130` — end of `bootstrap()`, after `setCodes` and `setStoredBaseSite`, before flipping status to `'ready'`. Cold-start burst.
- `AppContext.tsx:210` — end of `setBaseSiteForCode`, after `setCodes` + persistence, before flipping status back to `'ready'`. Base-site-change burst.
- `AppContext.tsx:239` — end of `setLanguageForCode`, after `setLangOnly` + persistence, before flipping status back to `'ready'`. Language-change burst.

All three paths gate render on status, which is set to `'loading'` while `initI18n` runs (e.g. line 200, line 233), so components do not paint with the new language until the new bundle is registered. This is the closest thing to a render-side i18n gate (see §6.2).

The new boot machine in `src/lib/store/bootStore.ts` has no `initI18n` call site yet — `runBaseSiteGate` (lines 184-212) explicitly does NOT fire `initI18n`; the stub `runFreshnessGate` (lines 257-259) just `set({ status: 'ready' })`. So on the NEW path today, translations are NOT loaded for the boot store — the old `AppContext.bootstrap` is still what drives `initI18n`. Step 6 closes this gap.

---

## 4. Translation cache + checksum storage — what's persisted today

### 4.1 Inventory of translation-related AsyncStorage usage

The translation layer touches three keys (or key shapes):

| Key | Read | Written |
|---|---|---|
| `translations_version` | `src/i18n/storage.ts:10` (`getStoredVersion`) and `src/i18n/loader.ts:7` (read-then-discarded) | `src/i18n/storage.ts:14` (`setStoredVersion`) and commented-out at `src/i18n/loader.ts:23` |
| `translations_<lang>` | `src/i18n/storage.ts:27` (`getTranslations`) and commented-out at `src/i18n/loader.ts:13` | `src/i18n/storage.ts:32` (`setTranslations`) and commented-out at `src/i18n/loader.ts:22` |
| `app_language` | `src/i18n/storage.ts:18` (`getStoredLanguage`) | `src/i18n/storage.ts:23` (`setStoredLanguage`) |

The first two are **inert** at runtime (declared, never called from anywhere live). The third (`app_language`) is live and used by both the old path (`AppContext.tsx:120, 124`) and the new path (`src/lib/store/bootStore.ts:188, 248`).

**No checksum is stored today, at any granularity.** Grep `checksum` returns zero matches outside the one comment in `bootStore.ts:254`. The spec's storage model (`expo-boot-redesign.md` Part 5) keys `translations_<NS>` and `translations_checksum_<NS>` do not exist on disk. The existing `translations_<lang>` (per-language, not per-namespace) is an architectural mismatch — step 6 supersedes it.

### 4.2 Inert helpers — current state

#### `getStoredVersion` / `setStoredVersion` — `src/i18n/storage.ts:9-15`

```ts
export async function getStoredVersion(): Promise<string | null> {
  return AsyncStorage.getItem(KEYS.VERSION);
}

export async function setStoredVersion(version: string): Promise<void> {
  await AsyncStorage.setItem(KEYS.VERSION, version);
}
```

- `getStoredVersion`: **zero callers.** Grep `getStoredVersion` → only the declaration.
- `setStoredVersion`: **zero callers.** Grep `setStoredVersion` → only the declaration.

State: **dead.** Defined, never called.

#### `getTranslations` / `setTranslations` — `src/i18n/storage.ts:26-33`

```ts
export async function getTranslations<T = unknown>(lang: string): Promise<T | null> {
  const data = await AsyncStorage.getItem(`translations_${lang}`);
  return data ? (JSON.parse(data) as T) : null;
}

export async function setTranslations<T>(lang: string, data: T): Promise<void> {
  await AsyncStorage.setItem(`translations_${lang}`, JSON.stringify(data));
}
```

- `getTranslations`: **zero callers.** Grep `getTranslations` → only the declaration.
- `setTranslations`: **zero callers.** Grep `setTranslations` → only the declaration.

State: **dead.** Defined, never called.

#### `translations_version` key — read-then-discarded

`src/i18n/loader.ts:6-26` (full body):

```ts
export async function loadTranslations(lang: string, tenant: string) {
  const storedVersion = await AsyncStorage.getItem(VERSION_KEY);

  // var validateVersionS = await validateVersion();

  // if (validateVersion) {
  //   // load from cache
  //   const cached = await AsyncStorage.getItem(`translations_${lang}`);
  //   if (cached) {
  //     return JSON.parse(cached);
  //   }
  // }

  // fetch all namespaces
  const resources = await loadAllNamespaces(lang);

  // await AsyncStorage.setItem(`translations_${lang}`, JSON.stringify(resources));
  // await AsyncStorage.setItem(VERSION_KEY, version);

  return resources;
}
```

State: **half-wired**. `storedVersion` is assigned (line 7) and never read again. The validate-and-cache-hit block (lines 9-17) is commented out, as are the post-fetch writes (lines 22-23). The function always falls through to `loadAllNamespaces` — the full 20-namespace burst on every call.

#### `validateVersion` — also inert

`src/i18n/fetchNamespace.ts:50-56`:

```ts
export async function validateVersion(): Promise<boolean> {
  try {
    return true;
  } catch (error) {
    return undefined;
  }
}
```

State: **dead.** Hard-returns `true`; no callers (`grep validateVersion` returns only this declaration and the commented-out reference in `loader.ts:9`). The `try` body cannot throw (only contains `return true`), so the `catch` branch is unreachable.

### 4.3 Summary — Part 5 storage model vs current reality

The spec ([`features/expo-boot-redesign.md`](../../oglasino-docs/features/expo-boot-redesign.md) Part 5, lines 261-267) asks for co-stored `translations_<NS>` + `translations_checksum_<NS>` keys per namespace, and a `base_site_catalog_checksum` key alongside `base_site`. **Today the entire checksum scheme is absent.** The half-wired `translations_<lang>` payload key (commented out) is per-language and would need to be removed in step 6 in favour of the per-namespace scheme; the `translations_version` key has no replacement (it was a single string, not a map).

---

## 5. Catalog checksum — base-site side

### 5.1 Nothing writes `base_site_catalog_checksum` today

- `grep -RIn 'base_site_catalog_checksum\|base_site_catalog\|catalog_checksum\|catalogChecksum\|catalog\.checksum'` across `src/` and `app/` → **zero matches.**
- The only checksum-related token anywhere in the working tree is the one-line comment in `src/lib/store/bootStore.ts:254`.

**Confirmed:** the `base_site_catalog_checksum` AsyncStorage key does not exist on disk today. Step 5 (already shipped) deliberately deferred it; step 6 introduces it.

### 5.2 `base_site` key holds the full `BaseSiteDTO`

`src/lib/init/baseSitesService.ts:7-18`:

```ts
const KEYS = {
  BASE_SITE: 'base_site',
};

export async function getStoredBaseSite(): Promise<BaseSiteDTO | null> {
  const data = await AsyncStorage.getItem(KEYS.BASE_SITE);
  return data ? JSON.parse(data) : null;
}

export async function setStoredBaseSite(site: BaseSiteDTO) {
  await AsyncStorage.setItem(KEYS.BASE_SITE, JSON.stringify(site));
}
```

The value written under `'base_site'` is the **full `BaseSiteDTO`** (`JSON.stringify(site)`), not just the code or a partial view.

`src/lib/types/catalog/BaseSiteDTO.ts:6-19`:

```ts
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

So the stored blob includes `catalog: CatalogDTO` — the very payload the catalog checksum tracks. The same shape is written by the picker path (`src/lib/store/bootStore.ts:247` — `await setStoredBaseSite(fullDto)` after `fetchBaseSiteByCode`) and by the old path (`src/components/context/AppContext.tsx:129, 208`).

### 5.3 Catalog refresh = re-store of the same shape

`src/lib/init/baseSitesService.ts:48-56`:

```ts
export async function fetchBaseSiteByCode(code: string): Promise<BaseSiteDTO> {
  const res = await BACKEND_API.get(`/public/baseSite/${encodeURIComponent(code)}`, {
    _bootstrap: true,
  });
  if (res.status >= 200 && res.status < 300) {
    return res.data;
  }
  throw new Error(`baseSite.fetchBaseSiteByCode(${code}): non-2xx status ${res.status}`);
}
```

The endpoint returns `BaseSiteDTO` — the same shape that lives under `'base_site'` in AsyncStorage. A catalog-staleness refetch in Gate 4 is therefore:

1. `const fresh = await fetchBaseSiteByCode(storedSite.code);`
2. `await setStoredBaseSite(fresh);` — overwrites the existing `'base_site'` blob.
3. Write the new checksum under `'base_site_catalog_checksum'` (key does not exist yet — step 6 introduces).
4. Update the boot store's `selectedBaseSite` slot in memory.

No new endpoint, no DTO shape change. The catalog field on the in-memory `BaseSiteDTO` updates automatically because the whole DTO is replaced.

---

## 6. Where i18n is read at render

### 6.1 Single instance, react-i18next consumption

There is exactly one i18n instance: the default `i18next` singleton, configured in `src/i18n/i18n.ts:10-24` via `i18n.use(ICU).use(initReactI18next).init(...)`. The `initReactI18next` plugin wires `react-i18next`'s context to that instance.

Components consume translations through a thin wrapper:

`src/i18n/useTranslations.ts:1-6` (full file):

```ts
import { useTranslation } from 'react-i18next';

export function useTranslations(namespace?: string) {
  const { t } = useTranslation(namespace);
  return t;
}
```

There are **291 call sites** of `useTranslation` / `useTranslations` across `src/` and `app/` (mechanical count). Spot check confirms every site uses `useTranslations(TranslationNamespace.XYZ)` and reads via `t('some.key')`. No component bypasses react-i18next; no component constructs its own i18n instance.

**Conclusion:** "load into i18n" = register bundles on the single default `i18next` singleton. Gate 4 registering the active language's namespaces there will let all 291 sites resolve correctly.

### 6.2 Render gating

There is **no explicit i18n render gate** anywhere in the tree. No `Suspense` around content, no `i18n.isInitialized` check, no `<I18nextProvider>` wrapper. Components render whatever `t('key')` returns; if i18n is not yet initialized, react-i18next falls through to the missing-key handler and returns the key itself.

The OLD path's implicit gate is the status overlay in `app/_layout.tsx:72-80`:

```tsx
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

Because `AppContext.bootstrap` `await`s `initI18n(...)` (line 130) BEFORE setting status to `'ready'` (line 132-139), the overlay covers the always-mounted `<Stack>` until i18n is initialized. The user never sees the screens beneath while keys are unresolved.

Two caveats:

- The `<Stack>` is always mounted — its screens may render and fire effects before status flips. `<RequireBaseSite>` (`src/components/context/RequireBaseSite.tsx`) is a per-screen render gate for 5 base-site-dependent screens, but it gates on `selectedBaseSite`, not on `i18n.isInitialized`. A screen that uses `useTranslations` but does NOT use `<RequireBaseSite>` (most secured tabs, owner dashboard, free-zone, etc.) can technically render with raw keys for a frame before the overlay paints over them.
- `setBaseSiteForCode` and `setLanguageForCode` flip status to `'loading'` BEFORE calling `initI18n` (`AppContext.tsx:200, 233`), so the overlay returns during those transitions — same implicit gate, second-occurrence.

**Conclusion for step 6:** Gate 4 must register the active language's namespaces into the singleton BEFORE flipping `bootStatus` to `'ready'` (or, for the `'updating'` transient, before returning to `'ready'`). The render-side gating is already in place via the status overlay; step 6 just needs to ensure register-before-ready, exactly like the old path does. There is no separate i18n-readiness flag to wire — the status overlay carries that load.

The brief asks whether components show raw keys if the portal renders before namespaces are registered: yes, react-i18next's default for a missing key on a missing namespace is to return the key string. The OLD path avoids this by sequencing `initI18n` before `status = 'ready'`. The NEW boot machine must preserve that sequencing.

---

## 7. Language scope

### 7.1 One language at a time

At any moment the i18n instance holds exactly **one language's** translations. Evidence:

- `initI18n` is always called with a single `lang` and passes `{ [lang]: resources }` to `i18n.init` (`src/i18n/i18n.ts:18-20`). The `lng` and `fallbackLng` arguments are single strings, not arrays.
- `loadAllNamespaces(lang)` runs the 20-parallel burst for that one `lang` only (`src/i18n/loadNamespaces.ts:4-15`).
- Re-initialisation on language change: `setLanguageForCode` (`AppContext.tsx:222-253`) flips status to `'loading'`, calls `setStoredLanguage(lang)`, then `await initI18n(lang.code, site.code)` (line 239), which re-runs the full 20-namespace fetch for the new `lang.code`. The previous language's bundles are not retained on the i18n instance after `init` re-runs with new `resources`.
- AsyncStorage stores exactly one language under `'app_language'` (`src/i18n/storage.ts:17-24`) — the currently-selected one, not a list.

The spec's number of locales is "EN, SR, RU. Montenegrin (me/cnr) aliases to SR" (`meta/conventions.md` Part 9, line 479) — so a base site's `allowedLanguages` is typically 2–3 codes. Mobile holds **only the selected one at a time**; it has never held more.

### 7.2 What changes a namespace's checksum vs what mobile must refetch

Per the version-checksums spec ([`features/version-checksums.md`](../../oglasino-docs/features/version-checksums.md) §4, lines 244-245):

> Translation checksums are per-namespace, collapsed across all languages. A namespace's checksum changes if any key/value in that namespace changes in any language. There is no per-namespace-per-language checksum.

This means: a Russian-only translation change to namespace `COMMON` bumps `translations.checksum.COMMON` once, and a Serbian user on the app sees that bumped checksum even though the change wasn't to their language. The conservative read of the spec: refetch the active language for that namespace.

**Implication for step 6:** because mobile holds only the active language at any time, a stale namespace = "refetch the active language's payload for that namespace." There is no scenario today where refetching all `allowedLanguages` for a stale namespace is needed — none of them are loaded into i18n simultaneously. The audit confirms the simpler model is sufficient: per stale namespace → fetch one language (the active one), register one bundle.

If future work decides to pre-warm a second language (e.g. silently fetch the alternate `allowedLanguages[1]` so language switches are instant), the per-namespace-per-active-language storage scheme would need extension. Not in scope today.

---

## 8. Adjacent observations (flagged for Mastermind)

- **Dead i18n types.** `TranslationResponse` and `VersionResponse` (`src/i18n/types.ts:4-11`) have zero callers and describe the old single-`version`-string contract. Safe to delete when step 6 introduces the new `VersionsResponse` type. Low severity, scope-adjacent.

- **Dead i18n helpers.** `getStoredVersion`, `setStoredVersion`, `getTranslations`, `setTranslations` (`src/i18n/storage.ts`), `validateVersion` (`src/i18n/fetchNamespace.ts:50-56`), and the entire commented-out scaffolding in `src/i18n/loader.ts:9-23` are dead — see §4.2. Step 6 will replace `loadTranslations`'s body and is the natural place to delete them; the spec already lists them for removal ([`features/expo-boot-redesign.md`](../../oglasino-docs/features/expo-boot-redesign.md) Part 5, line 267). Low severity, scope-adjacent.

- **`tenant` argument unused inside i18n loaders.** `initI18n(lang, tenant)` and `loadTranslations(lang, tenant)` accept `tenant` and pass it down, but nothing inside either function reads it. The active base-site code reaches the backend via the request interceptor's `X-Base-Site` header (`src/lib/config/api.ts`), not via the loader. The arg is decorative as written. Per-language cache keys (when caching lands in step 6) should consider per-base-site partitioning to avoid the collision the prior audit's `audit-expo-boot-redesign.md` §2.3 flagged. Medium severity if the per-NS cache lands as `translations_<NS>` (no language, no base site) — same key would be reused across language switches; mitigated only because the spec already keys per-NS-per-language-per-base-site (§4 freshness model). Flagged for clarity, no action this audit.

- **`TranslationNamespace` enum count drift.** Mobile has 20 enum entries; backend spec lists 22 (`ADMIN_PAGES` and `BACKEND_TRANSLATIONS` missing). Both omissions are intentional, but the prior audit `audit-expo-boot-redesign.md` §2.3 calls it "19 enum values" — wrong by one. Worth a one-line correction in that audit when it next gets touched, plus a defensive comment on the mobile enum naming the two backend namespaces it deliberately omits. Low severity.

- **`fetchNamespace` return type lies.** Annotated `Promise<Record<string, any>>` (line 29) but every failure branch returns `undefined`. Both `loadAllNamespaces` and downstream `i18n.init` tolerate this (the resources entry becomes `undefined` and react-i18next falls through to the key on missing-namespace lookups). Worth tightening to `Promise<Record<string, any> | undefined>` when step 6 touches this function. Low severity.

- **`fetchNamespace` failures are silent.** No log, no toast, no error count. A backend hiccup during the cold-start burst quietly leaves one namespace empty, and the user sees raw keys for every component depending on it until the next `initI18n` runs. Adjacent to step 6's responsibilities (Gate 4's per-NS refetch should at least log on failure). Medium severity, scope-adjacent.

---

## 9. What step 6 has to land — derived from this audit

A compressed list of capabilities Gate 4 needs that do not exist today. Not a design — just the gap inventory.

| Capability | Today | Needed for Gate 4 |
|---|---|---|
| `GET /api/public/versions` client | none | service function returning typed response |
| `VersionsResponse` TS type | none (old `VersionResponse` mismatched) | new type matching backend §4.1 shape |
| Per-namespace checksum read/write | none | `translations_checksum_<NS>` key helpers |
| Per-namespace payload read/write | dead `translations_<lang>` helpers only | `translations_<NS>` key helpers (or per-NS-per-lang) |
| Per-base-site catalog checksum read/write | none | `base_site_catalog_checksum` key helpers |
| Catalog refetch primitive | `fetchBaseSiteByCode` exists | (already in place — re-use) |
| Per-namespace fetch primitive | `fetchNamespace` exists | (already in place — re-use) |
| Incremental "register one namespace into i18n" call site | none — only `init` does it | `i18n.addResourceBundle` OR rebuild-then-`init` pattern |
| Render gate on i18n readiness before `bootStatus = 'ready'` | implicit via status overlay; old path `await initI18n` before `setStatus('ready')` | preserve: Gate 4 must await register-into-i18n before setting `'ready'` (Invariant 2 in `bootStore.ts:33-42`) |
| Active language is the unit of "loaded into i18n" | confirmed | confirmed; refetch only the active language per stale NS |

End of audit.
