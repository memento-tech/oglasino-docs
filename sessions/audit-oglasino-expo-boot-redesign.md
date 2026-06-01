# Audit — Expo cold-start boot, current implementation

**Repo:** `oglasino-expo`
**Branch:** `new-expo-dev`
**Date:** 2026-05-28
**Type:** read-only audit
**Scope:** document the CURRENT cold-start boot exactly as it exists, plus what is missing for a versioning-aware boot. No code changes.

---

## Part 1 — Current boot flow, end to end

### 1.1 Where boot starts (component tree + call order)

Cold start enters at `app/_layout.tsx:28` (`RootLayout`). The provider chain rendered top-down is:

```
RootLayout (app/_layout.tsx)
└── GestureHandlerRootView
    └── AppContextProvider                (src/components/context/AppContext.tsx:55)
        └── AppVersionConfigInit           (src/components/internals/AppVersionConfigInit.tsx:29)
            └── ToastProvider
                └── ThemeProvider
                    └── SafeAreaView
                        ├── {status === 'ready' && <AppInit />}   (src/components/init/AppInit.tsx:11)
                        ├── <View><Stack /></View>                 (always mounted)
                        ├── <PortalHost />
                        ├── <DialogManager />
                        └── {status !== 'ready' && overlay}        (LoadingOverlay / BaseSiteSelector)
```

Two side effects fire before any provider mounts:

- `app/_layout.tsx:22` — `registerAuthInterceptors()` (module-level, sets `configureAuthInterceptorHooks`).
- `app/_layout.tsx:24-26` — Android-only `NavigationBar.setVisibilityAsync('hidden')`.

Call order at cold start:

1. `RootLayout` first render. `useState<AppStateValue['status']>()` initializes `status` to `undefined`. The overlay branch at `app/_layout.tsx:72-80` renders `LoadingOverlay` (since `!status || status === 'loading'`).
2. `AppContextProvider` mounts with `defaultState = { status: 'loading', baseSites: [] }` (`AppContext.tsx:48-51`).
3. `AppVersionConfigInit` mounts and immediately renders only an `ActivityIndicator` (`AppVersionConfigInit.tsx:118-124`) because `preparing` is `true`. **At this point its `children` (everything below it, including `Stack`, `AppInit`, the overlay) are NOT mounted.**
4. `AppVersionConfigInit`'s `useEffect` at line 101 fires `checkVersion(true)`, which awaits `getAppVersionConfig()` (`GET /public/app/version/<Platform.OS>?currentVersion=...`).
5. Independently, `AppContextProvider`'s `useEffect` at `AppContext.tsx:158` fires `bootstrap()` and starts the 5-second maintenance poll.
6. `bootstrap()` (`AppContext.tsx:89`) does:
   ```
   const [maintenance, configuration, baseSites] = await Promise.all([
     checkIfMaintenance().catch(() => true),
     getAppConfiguration().catch(() => undefined),
     fetchBaseSites().catch(() => []),
   ]);
   ```
7. Once `Promise.all` resolves, `bootstrap` either:
   - sets `status: 'maintenance'` if `maintenance || !configuration` (`AppContext.tsx:100`),
   - sets `status: 'select-base-site'` if no stored base site (`AppContext.tsx:111`),
   - or proceeds to `setCodes(...)`, persist `setStoredBaseSite`, `initI18n(...)`, then `status: 'ready'` (`AppContext.tsx:120-139`).
8. Independently, `AppVersionConfigInit.checkVersion(true)` finishes and flips `preparing` to `false` (`AppVersionConfigInit.tsx:97`), which then mounts the children for the first time.

This means **`AppVersionConfigInit` is BOTH a render gate (its `ActivityIndicator` covers everything until version check resolves) AND a sibling of bootstrap**: it owns the app-version response but does not feed it into `bootstrap`. The two flows are concurrent and uncoordinated. If `getAppVersionConfig()` is fast and `bootstrap` is slow, the user sees a spinner that briefly switches from `AppVersionConfigInit`'s `ActivityIndicator` to `LoadingOverlay` then to either content or the maintenance/select-base-site overlay.

### 1.2 The four-parallel boot burst

The four cold-start HTTP requests sent before any user interaction:

| # | Endpoint | Caller file | _bootstrap flag |
|---|----------|-------------|-----------------|
| 1 | `GET /public/maintenance/active` | `src/lib/services/maintenanceService.tsx:6` | `true` |
| 2 | `GET /public/config` | `src/lib/services/configurationService.tsx:8` | `true` |
| 3 | `GET /public/baseSite/details` | `src/lib/init/baseSitesService.ts:21` | `true` |
| 4 | `GET /public/app/version/<Platform.OS>?currentVersion=...` | `src/lib/services/appVersionConfigService.tsx:10-12` | `true` |

Requests 1–3 are dispatched by `AppContextProvider.bootstrap()` (`AppContext.tsx:94-98`) inside a single `Promise.all`. Request 4 is dispatched independently by `AppVersionConfigInit.checkVersion(true)` (`AppVersionConfigInit.tsx:101`, called from `useEffect` on mount).

All four set `{ _bootstrap: true }` so they bypass the bootstrap barrier (see §1.3).

### 1.3 `apiStore.ts` — the bootstrap barrier

File: `src/lib/config/apiStore.ts` (29 lines).

Shape:

```
type ApiStoreState = {
  baseSite: string | null;
  lang: string | null;
  waiters: (() => void)[];
};

const KEY = '__OGLASINO_API_STORE__';
const g = globalThis as typeof globalThis & { [KEY]?: ApiStoreState };
const state: ApiStoreState = (g[KEY] ??= { baseSite: null, lang: null, waiters: [] });
```

The state object is pinned on `globalThis['__OGLASINO_API_STORE__']` (`apiStore.ts:7-9`) and lazily initialized with `??=`. The module export holds a reference to that single object — re-imports do not create a new state because the assignment `(g[KEY] ??= …)` is a no-op when the key already exists. The pinning is what makes the barrier survive Fast Refresh / module re-evaluation.

Public surface:

- `getCurrentBaseSiteCode()` → `state.baseSite` (`apiStore.ts:11`)
- `getCurrentLangCode()` → `state.lang` (`apiStore.ts:12`)
- `setCodes(baseSite, lang)` (`apiStore.ts:14-20`) — assigns both codes, snapshots the waiter list, clears it, then `forEach(r => r())`. "Codes are set" = `state.baseSite !== null`. The barrier resolves the moment `setCodes` runs, by firing the queued resolvers.
- `setLangOnly(lang)` (`apiStore.ts:22-24`) — used by language switch in `AppContext.tsx:236`. Does NOT trigger waiters.
- `waitForBootstrap()` (`apiStore.ts:26-29`):
  ```
  export function waitForBootstrap(): Promise<void> {
    if (state.baseSite !== null) return Promise.resolve();
    return new Promise<void>((r) => state.waiters.push(r));
  }
  ```
  If `baseSite` is null the resolver is appended to `state.waiters`; otherwise it resolves synchronously.

Callers of `waitForBootstrap()` (grep across `src/` and `app/`):

- `src/lib/config/api.ts:53` — the only caller. Inside the request interceptor, gated by `if (!getCurrentBaseSiteCode() && !config._bootstrap)` (`api.ts:52`).

`setCodes` is called from exactly two sites:

- `AppContext.tsx:127` — at the end of `bootstrap()` immediately after `getStoredLanguage()` resolves and before `setStoredBaseSite`, `initI18n`, and the final `setState({ status: 'ready' })`.
- `AppContext.tsx:206` — inside `setBaseSiteForCode`, when the user picks a base site from `BaseSiteSelector`.

No code path resets the barrier back to `null` once codes are set.

### 1.4 The four boot service files

#### configurationService — `src/lib/services/configurationService.tsx`

```
export const getAppConfiguration = async (): Promise<ConfigMap> => {
  try {
    const response = await BACKEND_API.get<ConfigMap>('/public/config', { _bootstrap: true });

    if (response.status === 200) {
      return response.data;
    }

    logServiceWarn('config.getAppConfiguration', response);
    return null;
  } catch (e) {
    logServiceError('config.getAppConfiguration', e);
    return null;
  }
};
```

- Single endpoint: `GET /public/config`.
- Carries `{ _bootstrap: true }` so it bypasses `waitForBootstrap()`.
- Response is the raw `ConfigMap` (an alias for `Record<string, string>` — line 4); no further parsing.
- Awaited in `AppContext.tsx:96` (`getAppConfiguration().catch(() => undefined)`).
- Consumed in `AppContext.tsx:100` (`maintenance || !configuration` → 'maintenance' status) and stored under `state.configuration`. Read back via `getConfiguration(key)` (`AppContext.tsx:255-257`).
- Return type is annotated `Promise<ConfigMap>` but the function returns `null` on non-200 / catch. This is a minor type-safety violation (`null` is not `Record<string, string>`); callers tolerate it because `AppContext.tsx:100` only checks falsy.

#### baseSiteService — `src/lib/init/baseSitesService.ts`

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

- Endpoint: `GET /public/baseSite/details`.
- `{ _bootstrap: true }` set on request config.
- Returns raw `res.data` assumed-typed `BaseSiteDTO[]` — no validation, no transformation.
- Awaited in `AppContext.tsx:97` (`fetchBaseSites().catch(() => [])`).
- Result lands in `AppStateValue.baseSites` for all three status branches (maintenance / select-base-site / ready). Used by `BaseSiteSelector` and `setBaseSiteForCode`.
- The same file owns the AsyncStorage persistence helpers `getStoredBaseSite` (`baseSitesService.ts:10-13`) and `setStoredBaseSite` (`baseSitesService.ts:15-17`) — see Part 2.

#### maintenanceService — `src/lib/services/maintenanceService.tsx`

```
export const checkIfMaintenance = async (): Promise<boolean> => {
  try {
    const response = await BACKEND_API.get<boolean>('/public/maintenance/active', { _bootstrap: true });

    if (response.status === 200) {
      return response.data;
    }

    logServiceWarn('config.isMaintenanceActive', response);
    return true;
  } catch (e) {
    logServiceError('config.isMaintenanceActive', e);
    return true;
  }
};
```

- Endpoint: `GET /public/maintenance/active`.
- `{ _bootstrap: true }`.
- Returns the raw boolean payload. Fail-closed: `return true` on non-200 OR catch (so a network error puts the app into maintenance status).
- Awaited in `AppContext.tsx:95` (`checkIfMaintenance().catch(() => true)` — the catch is redundant since the function never throws, but it's there as a belt-and-braces).
- Also polled every 5 s by `AppContext.tsx:161-180` (the `setInterval` block) to detect maintenance flips at runtime. The poll does NOT carry `_bootstrap: true` — it relies on codes already being set (or, pre-`setCodes`, on the request interceptor's `waitForBootstrap()` gate). In practice, post-`setCodes`, the poll uses the same `_bootstrap`-less codepath as ordinary requests.

#### appVersionConfigService — `src/lib/services/appVersionConfigService.tsx`

```
export default async function getAppVersionConfig(): Promise<AppVersionConfig | undefined> {
  try {
    const currentVersion = Application.nativeApplicationVersion;

    const res = await BACKEND_API.get(
      `/public/app/version/${Platform.OS}?currentVersion=${currentVersion}`,
      { _bootstrap: true }
    );

    if (res.status === 200 && res.data) {
      return res.data;
    }

    return undefined;
  } catch (error) {
    return undefined;
  }
}
```

- Endpoint: `GET /public/app/version/<Platform.OS>?currentVersion=<expo-application native version>`.
- `{ _bootstrap: true }`.
- Response shape `AppVersionConfig = { latestVersion: string; minSupportedVersion: string; forceUpdate: boolean; optionalUpdate: boolean }` (`src/lib/types/AppVersion.ts`).
- Awaited only by `AppVersionConfigInit.checkVersion` (`AppVersionConfigInit.tsx:59`).
- The result feeds `AppVersionConfigInit`'s local React state (`config`, `showOptional`). **It is NEVER fed back into `AppContext`'s `bootstrap` or status machine.** `bootstrap` does not know whether the version check succeeded, failed, or said `forceUpdate: true`.

### 1.5 How the root `<Stack>` decides what to render

`app/_layout.tsx:58-86`. The `Stack` is always mounted (`<View className="flex-1"><Stack screenOptions={{ headerShown: false }} /></View>` at line 67). On top of it, an overlay is conditionally rendered:

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

`AppInit` is mounted only when `status === 'ready'` (line 65). Children of `AppVersionConfigInit` (everything in the tree, including the `Stack`) are NOT rendered while `preparing` is true (`AppVersionConfigInit.tsx:118-124` early-returns an `ActivityIndicator`).

Status values:

- `AppStateValue['status']` = `'loading' | 'maintenance' | 'select-base-site' | 'ready'` (`AppContext.tsx:22`).
- `defaultState.status === 'loading'` (`AppContext.tsx:49`).
- `RootLayout` mirrors this via `useState<AppStateValue['status']>()` (initially `undefined`) and `setStatus` is wired through `AppContextProvider.onStatusChanged` (`app/_layout.tsx:31, 60`). The `setStateWithStatusHook` wrapper (`AppContext.tsx:70-85`) fires the callback in a `setTimeout(..., 0)` whenever status changes.

The status lives in two places: `AppContext` state (canonical) and `RootLayout` local state (mirror). `RootLayout` reads the mirror; the rest of the app reads `useAppState()`.

`<RequireBaseSite>` is a per-screen render gate, not a navigator-level gate — see Part 5.

---

## Part 2 — Storage

### 2.1 Base site persistence

File: `src/lib/init/baseSitesService.ts:6-17`.

```
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

- **AsyncStorage key: `'base_site'`.**
- **The FULL `BaseSiteDTO` is persisted, not just the code.** The catalog (categories, order types, etc.), `defaultLanguage`, `allowedLanguages`, currency, and every other field on the DTO are all written as one JSON blob.
- The read in `getStoredBaseSite` is async (`await AsyncStorage.getItem`) and is awaited by `bootstrap` at `AppContext.tsx:109` (`const storedBaseSite = await getStoredBaseSite();`).
- Written by `bootstrap` at `AppContext.tsx:129` (`await setStoredBaseSite(storedBaseSite)` — re-persists what was just read, which is a no-op overwrite) and by `setBaseSiteForCode` at `AppContext.tsx:208` (writes the freshly selected site).
- No version, checksum, or timestamp is co-stored with the base site. The read returns whatever was last written — there is no staleness check.

### 2.2 Language persistence

File: `src/i18n/storage.ts:4-24`.

```
const KEYS = {
  VERSION: 'translations_version',
  LANGUAGE: 'app_language',
};
…
export async function getStoredLanguage(): Promise<LanguageDTO | null> {
  const data = await AsyncStorage.getItem(KEYS.LANGUAGE);
  return data ? JSON.parse(data) : null;
}

export async function setStoredLanguage(language: LanguageDTO) {
  await AsyncStorage.setItem(KEYS.LANGUAGE, JSON.stringify(language));
}
```

- **AsyncStorage key: `'app_language'`.**
- The full `LanguageDTO` blob is persisted (not just the code).

### 2.3 Translation payload cache — declared but inert

File: `src/i18n/storage.ts:26-33`.

```
export async function getTranslations<T = unknown>(lang: string): Promise<T | null> {
  const data = await AsyncStorage.getItem(`translations_${lang}`);
  return data ? (JSON.parse(data) as T) : null;
}

export async function setTranslations<T>(lang: string, data: T): Promise<void> {
  await AsyncStorage.setItem(`translations_${lang}`, JSON.stringify(data));
}
```

**These functions exist but have ZERO callers.** Grep across `src/` and `app/`:

- `getTranslations` / `setTranslations` — defined here, called nowhere.

File: `src/i18n/loader.ts` (the only loader path that runs at boot):

```
const VERSION_KEY = 'translations_version';

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

- `storedVersion` is read but **never used** (assigned and discarded).
- The cache read, the cache write, and the version write are all commented out.
- Every boot calls `loadAllNamespaces(lang)` (`src/i18n/loadNamespaces.ts`) which does `Promise.all` of `fetchNamespace(ns, lang)` across the 19 `TranslationNamespace` enum values (`src/i18n/types.ts:18-50`). So **on every cold start the app fires 19 parallel `GET /public/translations?namespace=<NS>&lang=<LANG>` calls** from `fetchNamespace.ts:31-33`.
- The `[BOOT] /public/translations?namespace=INTRO` log line referenced in the brief is one of these 19 requests passing through the `api.ts` request interceptor's `console.log('[BOOT]', config.url, ...)` instrumentation (`api.ts:44-51`).
- These translation requests do NOT carry `_bootstrap: true`. They go through `waitForBootstrap()` if codes are not yet set — and `loadTranslations` is called from `initI18n` (`src/i18n/i18n.ts:7-25`), which is called from `bootstrap` at `AppContext.tsx:130` **after** `setCodes` at line 127. So in cold-start the barrier is already open by the time these fire, and they piggy-back on the codes set in step 7 of §1.1.

### 2.4 Translation version storage — declared but inert

- `KEYS.VERSION = 'translations_version'` (`src/i18n/storage.ts:5`).
- Helpers `getStoredVersion` (`storage.ts:9-11`) and `setStoredVersion` (`storage.ts:13-15`) exist.
- `getStoredVersion` has **zero callers**.
- `setStoredVersion` has **zero callers**.
- The only reference to the `translations_version` key outside `storage.ts` is `loader.ts:7` (`const storedVersion = await AsyncStorage.getItem(VERSION_KEY);`) where the value is read but discarded.

### 2.5 Confirmed: nothing version-aware is stored

No version, checksum, or `versions` payload is stored client-side. The `translations_version` key is read at boot but its value is never used. No code path writes to it.

The full storage inventory at boot:

| Key | Purpose | Read at boot? | Written at boot? |
|-----|---------|--------------|------------------|
| `base_site` | full `BaseSiteDTO` blob (incl. catalog) | yes (`getStoredBaseSite`) | yes (`setStoredBaseSite` re-persist at `AppContext.tsx:129`) |
| `app_language` | full `LanguageDTO` blob | yes (`getStoredLanguage` at `AppContext.tsx:120`) | conditionally (`AppContext.tsx:124` only if absent) |
| `translations_version` | (intended for translation version) | read into a discarded local (`loader.ts:7`) | never |
| `translations_<lang>` | (intended for cached translation payload) | never (commented out) | never (commented out) |
| `dismissed_optional_update_version_data` | dismissed version + dismissal timestamp | yes (`AppVersionConfigInit.tsx:78`) | yes on user-dismiss (`AppVersionConfigInit.tsx:51`) |
| `auth-storage` (Zustand persist) | `{ user }` slice of `useAuthStore` | yes (rehydrate) | on auth change |
| `card_size_<scope>` | card-size preference (handled by `useCardSizeStore`) | yes (manual hydration) | on user toggle |

---

## Part 3 — What exists for versioning on the client

### 3.1 `GET /api/public/versions` — NOT consumed

Grep across `src/`, `app/`, `assets/`, and the rest of the working tree:

- `/public/versions` — zero matches.
- `versions` as a route fragment — only `/public/app/version/<Platform.OS>` (the app-version endpoint, see §1.4) and `getStoredVersion` / `VERSION_KEY` (the inert translation-version helpers, see §2.4).
- `checksum` — zero matches.
- `version_checksum` — zero matches.

**Confirmed: no client code consumes `/api/public/versions` today.** The handoff brief and `state.md`'s Expo backlog entry ("Adopt `/api/public/versions` endpoint…") are correct — this is unwritten.

### 3.2 Checksum / staleness logic

There is no checksum-comparison or "is my cached data stale" logic anywhere in the repo.

- `loader.ts:7` reads `translations_version` and discards it — closest thing to a staleness check, and even the commented-out scaffolding (`loader.ts:11-17`) only branches on a boolean `validateVersion()` (`fetchNamespace.ts:50-56`) that hard-returns `true` and never queries the server.
- The base site blob in AsyncStorage is read and trusted unconditionally. `bootstrap` does not compare the stored catalog against anything from `fetchBaseSites()`, even though both are available in the same scope (`AppContext.tsx:97, 109`).
- `AppVersionConfigInit` compares the dismissed version against the server's `latestVersion` (`AppVersionConfigInit.tsx:88`) but that is for the app-binary update dismissal, not catalog/translation freshness.

### 3.3 How translations are fetched and refreshed today

The translation fetch path:

1. `initI18n(lang, tenant)` (`src/i18n/i18n.ts:7`) — called from `AppContext.tsx:130` at the end of `bootstrap` (`await initI18n(language.code, storedBaseSite.code);`) and from `AppContext.tsx:210` at the end of `setBaseSiteForCode` and `AppContext.tsx:239` at the end of `setLanguageForCode`.
2. `initI18n` calls `loadTranslations(lang, tenant)` (`loader.ts:6`).
3. `loadTranslations` calls `loadAllNamespaces(lang)` (`loadNamespaces.ts:4`).
4. `loadAllNamespaces` does `Promise.all(namespaces.map(ns => fetchNamespace(ns, lang)))` over all 19 enum values in `TranslationNamespace` (`types.ts:18-50`).
5. Each `fetchNamespace(ns, lang)` (`fetchNamespace.ts:26-48`) does:
   ```
   const res = await BACKEND_API.get<TranslationDTO[]>(
     `/public/translations?namespace=${namespace}&lang=${lang}`
   );
   ```
   then flattens the `[{key, value}]` array into a dotted map and runs `toNested` (`fetchNamespace.ts:9-24`) to build a nested-object tree per i18next conventions.
6. `initI18n` calls `i18n.use(ICU).use(initReactI18next).init({ lng: lang, fallbackLng: 'sr', ns: Object.keys(resources), defaultNS: TranslationNamespace.COMMON_SYSTEM, resources: { [lang]: resources }, … })`.
7. The `tenant` argument is accepted by `initI18n` and `loadTranslations` but **never used by either function** (no `X-Base-Site` partitioning of the resources object). The axios request interceptor independently sets the `X-Base-Site` header from `getCurrentBaseSiteCode()` (`api.ts:58`), so the backend can scope the response per base site, but the loader does not key its result on tenant.

Refresh triggers (when the full 19-namespace burst re-fires):

- Every cold start, post-`setCodes`, in step 7 of §1.1.
- On base-site change: `setBaseSiteForCode` → `initI18n(lang.code, site.code)` at `AppContext.tsx:210`.
- On language change: `setLanguageForCode` → `initI18n(lang.code, site.code)` at `AppContext.tsx:239`.
- On re-bootstrap: `reBootstrap` exposed at `AppContext.tsx:268` (alias for `bootstrap`); called by maintenance-recovery flow at `AppContext.tsx:171` and possibly by other consumers via `useAppActions().reBootstrap` (grep returned no callers outside the context itself).

There is no per-namespace fetch; there is no incremental refresh; there is no caching layer.

---

## Part 4 — Update check (soft / hard)

### 4.1 What exists

The app-version response IS classified into soft (`optionalUpdate`) vs hard (`forceUpdate`) and IS used to gate UI. The logic lives entirely in `AppVersionConfigInit.tsx`.

Response shape (`src/lib/types/AppVersion.ts`):

```
export type AppVersionConfig = {
  latestVersion: string;
  minSupportedVersion: string;
  forceUpdate: boolean;
  optionalUpdate: boolean;
};
```

Hard update (`forceUpdate: true`) handling, `AppVersionConfigInit.tsx:126-156`:

```
if (config?.forceUpdate) {
  return (
    <Modal transparent>
      <View className="flex-1 items-center justify-center bg-background px-6">
        …
        <Text className="text-xl font-bold">Ažuriranje je obavezno</Text>
        <Text className="text-center">
          Vaša verzija aplikacije više nije podržana. Da biste nastavili korišćenje, potrebno je
          da instalirate najnoviju verziju ({config.latestVersion}).
        </Text>
        <Button onPress={openStore}>
          <Text …>Ažuriraj aplikaciju</Text>
        </Button>
      </View>
    </Modal>
  );
}
```

`forceUpdate: true` short-circuits the render: children (the rest of the app) are never returned. The "store" button opens `https://memento-tech.com` (`AppVersionConfigInit.tsx:39-41`) — that placeholder URL is a known gap, not a store deeplink.

Soft update (`optionalUpdate: true`) handling, `AppVersionConfigInit.tsx:76-95, 162-186`:

- Shown as a `Modal` over the children (children stay mounted).
- Suppressed by `dismissed_optional_update_version_data` AsyncStorage entry until either 24 hours pass (`ONE_DAY = 24 * 60 * 60 * 1000`, line 26) OR the `latestVersion` differs from the dismissed one (line 87-92).
- Buttons: "Kasnije" → `dismissOptional()` writes the dismissal record; "Ažuriraj" → same `openStore()` placeholder URL.

Re-check triggers:

- On mount, `checkVersion(true)` (`AppVersionConfigInit.tsx:102`).
- On `AppState` change from background/inactive → active, `checkVersion(false)` (`AppVersionConfigInit.tsx:104-109`).
- On every `pathname` change, `checkVersion(false)` (`AppVersionConfigInit.tsx:114-116`).

### 4.2 What is missing

- The app-version response feeds **only** `AppVersionConfigInit`'s local state. It does not flow into `AppContext`, `apiStore`, or anywhere else. If versioning-aware boot wants to use, say, the `latestVersion` field as an input to a checksum compare, that wiring does not exist.
- The block on hard update is a render-side gate, not a request-side gate. Network requests fired by `AppVersionConfigInit`'s children before `preparing` flips to `false` are not blocked — but in practice the early-return at line 118-124 prevents children from mounting at all until the first version check resolves, so this is a non-issue today. Once children are mounted and `forceUpdate` arrives on a later poll (`AppState` change or pathname change), children are unmounted at the next render but in-flight requests are not cancelled.
- There is no telemetry / log signal when `forceUpdate` or `optionalUpdate` fires — the modal is the only side effect.
- The "store" link is `https://memento-tech.com`, not an App Store / Play Store deeplink (`AppVersionConfigInit.tsx:40`).

---

## Part 5 — Known fragilities, confirmed against code

### 5.1 `authStore` ↔ chat-stores require-cycle triangle (Cycle B)

**Confirmed.** The triangle is:

- `src/lib/store/authStore.ts:21` — `import { useActiveChatStore } from './useActiveChatStore';`
- `src/lib/store/authStore.ts:22` — `import { useChatListStore } from './useChatListStore';`
- `src/lib/store/authStore.ts:23` — `import { useChatNavStore } from './useChatNavStore';`

And going back the other direction:

- `src/lib/store/useActiveChatStore.ts:30` — `import { useAuthStore } from './authStore';`
- `src/lib/store/useChatListStore.ts:17` — `import { useAuthStore } from './authStore';`
- `src/lib/store/useChatNavStore.ts:6` — `import { useAuthStore } from './authStore';`
- `src/lib/store/useChatBlockStore.ts:4` — `import { useAuthStore } from './authStore';`

Each of the three chat stores imports `useAuthStore` (and uses `useAuthStore.getState().user` inside store methods, e.g. `useActiveChatStore.ts:114, 195, 258, 514`; `useChatListStore.ts:52, 98`; `useChatNavStore.ts:25`). `authStore.logout()` (`authStore.ts:155-194`) calls `useChatListStore.getState().clearChatList()`, `useActiveChatStore.getState().clearActiveChat()`, `useChatNavStore.getState().clearChatNav()`.

`useChatNavStore` also imports `useActiveChatStore` (`useChatNavStore.ts:7`), so the cycle is multi-edged.

At runtime the cycle is benign — all cross-store reads use `.getState()` lazily inside callbacks, so Metro's lazy-init handles the bootstrap order. But the import graph itself is cyclic and any code-mod that swaps a `.getState()` for a top-of-module destructure would break boot.

### 5.2 The `globalThis`-pinned barrier

**Confirmed.** `src/lib/config/apiStore.ts:7-9`:

```
const KEY = '__OGLASINO_API_STORE__';
const g = globalThis as typeof globalThis & { [KEY]?: ApiStoreState };
const state: ApiStoreState = (g[KEY] ??= { baseSite: null, lang: null, waiters: [] });
```

The state object is owned by `globalThis['__OGLASINO_API_STORE__']`. The module-scope `state` binding is just a reference to that same heap object. Re-importing the module (Fast Refresh, test isolation, accidental dual import) hits the `??=` and finds the key already set, so the state is preserved and the waiters / codes do not get reset.

The "pin" is the `??=` plus the `globalThis` key. No teardown / reset is exposed.

### 5.3 Always-mounted `<Stack>` + `<RequireBaseSite>` per-screen render gate

**Confirmed.**

The `<Stack>` is mounted unconditionally at `app/_layout.tsx:67` (`<View className="flex-1"><Stack screenOptions={{ headerShown: false }} /></View>`). Pre-`ready` overlays sit on top of it (`app/_layout.tsx:72-80`) but the navigator and its screens remain mounted underneath.

`<RequireBaseSite>` (`src/components/context/RequireBaseSite.tsx`) is the per-screen render gate:

```
export function RequireBaseSite({ children, fallback = null }: …) {
  const { selectedBaseSite } = useAppState();
  if (!selectedBaseSite) return <>{fallback}</>;
  return <>{children}</>;
}
```

It wraps the screen body and short-circuits to `fallback` (default `null`) until `selectedBaseSite` is set in `AppContext` state.

Placements (5 screens — all base-site-dependent portal screens):

- `app/(portal)/(public)/index.tsx:10-19` — home feed.
- `app/(portal)/(public)/product/[...productData].tsx:37-39` — product detail.
- `app/(portal)/(public)/catalog/[...categories].tsx:60-80` — catalog.
- `app/(portal)/(public)/blog/free-zone.tsx:128-130` — free-zone.
- `app/(portal)/(public)/user/[...userData].tsx:16-18` — user detail.

`<RequireBaseSite>` is per-screen by design — Φ2 forbids conditional navigators, so the gating moved into screen bodies. Other base-site-dependent screens (secured tabs: favorites, notifications, messages; owner dashboard) do NOT use `<RequireBaseSite>` — they rely on the upstream overlay covering the screen before status is `ready` (the overlay at `app/_layout.tsx:72-80` is positioned `absolute inset-0` on top of the `Stack`, so user interaction with the screens beneath is blocked until status flips).

Net: the `Stack` and its screens render and may fire effects (and HTTP requests) before `selectedBaseSite` exists. `<RequireBaseSite>` only blocks the render of five screen bodies; it does NOT block effects in screens that don't use it, nor in shared chrome (`(portal)/_layout.tsx:11-35` reads `selectedBaseSite` to gate its `<TopBar>` / `<CategoryNavigation>` chrome but not the `<Tabs>` navigator itself).

### 5.4 Live `[BOOT]` instrumentation in `api.ts` request interceptor

**Confirmed.** `src/lib/config/api.ts:43-51`:

```
instance.interceptors.request.use(async (config) => {
  console.log(
    '[BOOT]',
    config.url,
    'flag:',
    config._bootstrap,
    'willAwait:',
    !getCurrentBaseSiteCode() && !config._bootstrap
  );
  if (!getCurrentBaseSiteCode() && !config._bootstrap) {
    await waitForBootstrap();
  }
  …
});
```

This is the live diagnostic instrumentation referenced in `issues.md` (2026-05-28 entry) and in the Φ3 handoff (`oglasino-docs/.agent/handoffs/phi3-bootloading.md` §5). It must be removed as part of the resolving fix.

The interceptor logs every outbound request — not only boot-time ones — so it currently spams every namespaced translation fetch, every search call, every product call, etc.

---

## Adjacent observations (out of scope for this audit, flagged for Mastermind)

- **`getAppConfiguration` return-type mismatch.** Declared `Promise<ConfigMap>` but returns `null` on the non-200 / catch branches (`configurationService.tsx:14, 17`). Low severity (callers tolerate it) but the type annotation is wrong. File: `src/lib/services/configurationService.tsx:6`.

- **`tenant` argument unused inside i18n loaders.** `initI18n(lang, tenant)` (`i18n.ts:7`) accepts `tenant` and passes it to `loadTranslations(lang, tenant)` (`loader.ts:6`), but neither function uses `tenant` — the only base-site partitioning happens via the request interceptor's `X-Base-Site` header. The arg is decorative as written. The cache key (when caching is uncommented) currently keys only on `lang` (`translations_<lang>`, `storage.ts:27, 32`), which would collide across base sites if caching is enabled. Medium severity if/when caching is turned back on. Files: `src/i18n/i18n.ts`, `src/i18n/loader.ts`, `src/i18n/storage.ts`.

- **`AppVersionConfigInit` re-fires `checkVersion` on every pathname change.** `AppVersionConfigInit.tsx:114-116` reads `usePathname()` and calls `checkVersion(false)` on every change. The `checkingRef` guard prevents overlap but does not debounce; rapid navigation triggers many version checks. Low severity. File: `src/components/internals/AppVersionConfigInit.tsx`.

- **`AppVersionConfigInit` store link is a placeholder.** `Linking.openURL('https://memento-tech.com')` (`AppVersionConfigInit.tsx:40`). Both forceUpdate and optionalUpdate "Update" buttons point here. Pre-launch blocker, not a redesign concern, but should be tracked. Medium severity. File: `src/components/internals/AppVersionConfigInit.tsx:39-41`.

- **`AppContext.bootstrap` re-writes `base_site` to AsyncStorage on every cold start.** Line 129 calls `await setStoredBaseSite(storedBaseSite)` where `storedBaseSite` came directly from `getStoredBaseSite()` on line 109. A no-op write that costs ~10–30 ms of AsyncStorage I/O per boot. Low severity. File: `src/components/context/AppContext.tsx:109, 129`.

- **`AppContext.tsx:142-144` catch-all swallows everything to `'maintenance'`.** Even though each of the three boot calls has its own `.catch(() => fallback)` (lines 95-97), the outer try/catch around `getStoredLanguage`, `setStoredLanguage`, `setCodes`, `setStoredBaseSite`, `initI18n` would translate any failure in those into `maintenance` status. A failure in `initI18n` (e.g. a single namespace fetch hanging past the axios timeout) puts the app into a misleading "maintenance" overlay. Medium severity. File: `src/components/context/AppContext.tsx:89-148`.

- **`fetchBaseSites` failure is silent: empty array.** Line 22-27 of `baseSitesService.ts` returns `[]` on non-2xx. Combined with line 97 of `AppContext.tsx` (`fetchBaseSites().catch(() => [])`), a network failure produces an empty selector list on `select-base-site`. The user sees a screen with no choices. Medium severity. File: `src/lib/init/baseSitesService.ts:19-32`.

---
