# Audit — Admin App Version Control (Mobile Read Contract)

**Repo:** oglasino-expo · **Branch:** main (brief guessed `new-expo-dev`; read-only, no checkout) · **Date:** 2026-06-06 · **Type:** read-only, contract confirmation.

**Scope:** confirm exactly what this app fetches at boot for the app-version-update gate and how it decides hard / soft / no-popup, so a web admin view can write values in a shape this app reads correctly. No mobile code changes.

---

## TL;DR for the admin view

- **Endpoint the popup gate hits:** `GET {API root}/public/app/version/{platform}?currentVersion={installedVersion}`. `{platform}` is `ios` or `android`; `currentVersion` is the installed native app version (semver string).
- **The mobile app does NOT compare versions itself.** It sends its installed version + platform and reads back two booleans — `forceUpdate` and `optionalUpdate` — that the **backend** computes. The admin/back end owns the comparison logic; mobile just obeys the booleans.
- **Response shape (all four fields):**
  ```ts
  type AppVersionConfig = {
    latestVersion: string;        // semver, e.g. "1.4.0" — DISPLAYED to user
    minSupportedVersion: string;  // semver — IGNORED by the client (see §4)
    forceUpdate: boolean;         // true → hard/blocking screen
    optionalUpdate: boolean;      // true → soft modal (if not dismissed)
  };
  ```
- **Decision:** `forceUpdate` → hard. Else `optionalUpdate` → soft (subject to 24h-per-version dismissal). Else nothing.
- **`forceUpdate` wins** if both are true (mobile checks it first and stops).

⚠️ **Do not confuse two endpoints.** There is a *separate* `GET /public/versions` (the i18n/catalog freshness-checksum gate, Gate 4). That is unrelated to update popups. The popup gate is `GET /public/app/version/{platform}` (Gate 2). The admin view must target the **`/public/app/version`** data, not `/public/versions`.

---

## 1. The fetch

- **Caller:** `src/lib/services/appVersionConfigService.tsx` → `getAppVersionConfig()`.
- **Invoked from:** `bootStore.runVersionGate()` (Gate 2), `src/lib/store/bootStore.ts:262`, wrapped in `withGateTimeout`.
- **Method / path:**
  ```
  GET /public/app/version/${Platform.OS}?currentVersion=${Application.nativeApplicationVersion}
  ```
  (`appVersionConfigService.tsx:10-12`.) `Platform.OS` is React Native's `"ios"` / `"android"`. The base URL is `EXPO_PUBLIC_API_URL` (`src/lib/config/api.ts:22`), which includes the `/api` prefix — so the absolute path is `{host}/api/public/app/version/{platform}?currentVersion=...`.
- **Success criteria:** the service returns `res.data` only on `res.status === 200 && res.data`; anything else (non-200, empty body, or thrown error) returns `undefined` — the whole call is in a `try/catch` that swallows errors (`appVersionConfigService.tsx:14-20`).
- **Response DTO the boot logic consumes** — `src/lib/types/AppVersion.ts`:

  | Field | Type | Consumed by mobile? |
  |---|---|---|
  | `latestVersion` | `string` (semver) | **Yes** — stored in `bootStore.latestVersion`, displayed in both update UIs. |
  | `minSupportedVersion` | `string` (semver) | **No** — present in the type, never read in client code (see §4). |
  | `forceUpdate` | `boolean` | **Yes** — triggers hard screen. |
  | `optionalUpdate` | `boolean` | **Yes** — triggers soft modal. |

---

## 2. The decision logic (exact)

`bootStore.ts:260-287`, `runVersionGate()`:

```ts
const config = await withGateTimeout(() => getAppVersionConfig());

if (config !== undefined) {
  set({ latestVersion: config.latestVersion });   // stored regardless of branch
  if (config.forceUpdate) {
    get().toUpdateRequired();                       // → bootStatus 'update-required', STOP
    return;
  }
  if (config.optionalUpdate) {
    set({ softUpdate: true });                      // flag; advances boot
  }
}
// ... falls through to runBaseSiteGate()
```

- **HARD (forced) — `HardUpdateScreen`:** shown when **`config.forceUpdate === true`**. Sets boot status to `update-required` and **stops boot** (`toUpdateRequired()`, `return`). The root layout renders `HardUpdateScreen` as a blocking `Modal` when `bootStatus === 'update-required'`. There is **no client-side version comparison** — the screen is driven purely by the boolean.
- **SOFT (optional) — `SoftUpdateModal`:** shown when `forceUpdate` is falsy **AND** `config.optionalUpdate === true` (sets `softUpdate: true`) **AND** the dismissal memory says show (§5). Boot continues normally; the modal floats over the portal.
- **NEITHER:** both booleans falsy → boot advances, no popup. Also no popup when `config === undefined` (swallowed non-2xx / empty body — treated as "you're fine, don't block"), or when `optionalUpdate` is true but the soft update was dismissed for this same version within 24h.
- **Precedence:** `forceUpdate` is checked first and returns early, so **`forceUpdate` wins over `optionalUpdate`** when both are true.
- **Timeout vs error (edge the admin should know):** if the call *hangs* past the gate timeout (`GateTimeoutError`), the app goes to **maintenance**, not update. A *fast non-2xx / thrown* error is swallowed → `undefined` → no popup, boot continues. So a malformed or 4xx/5xx response from the version endpoint will **not** force or soft-prompt anyone — it silently no-ops.

---

## 3. Per-platform

- **Request:** platform **is** sent, as a **path segment**: `.../app/version/${Platform.OS}` → `ios` or `android` (`appVersionConfigService.tsx:11`). The installed version is sent as the `currentVersion` query param.
- **Response:** the app reads `forceUpdate` / `optionalUpdate` / `latestVersion` directly from the body, with **no further platform branching** on the client. The platform-specific decision is therefore expected to be made **server-side**, keyed off the `{platform}` path segment + `currentVersion`.
- **Implication for the admin view:** version controls must be **stored and resolved per platform** (iOS and Android independently). The backend receives the platform in the URL and is expected to return that platform's booleans. The client gives the admin no platform-agnostic shortcut.

---

## 4. Version format

- **Type the app parses:** `latestVersion` is a **`string`** and is used only as **display text** — interpolated into the Serbian UI copy (`HardUpdateScreen.tsx:44`, `SoftUpdateModal.tsx:57`: "...najnoviju verziju ({latestVersion})"). The client never parses it numerically.
- **What it's compared against:** the dismissal memory does a **string equality** check `parsed.version !== latestVersion` (`softUpdateDismissal.ts:30`) — so `latestVersion` is also the **dismissal key**. It must be a stable, canonical string per release (e.g. `"1.4.0"`), because any change in the string re-prompts dismissed users (see §5).
- **`currentVersion` sent up:** `Application.nativeApplicationVersion` — the iOS `CFBundleShortVersionString` / Android `versionName`, i.e. a **semver string** like `"1.4.0"` (not the integer build number).
- **`minSupportedVersion`:** declared `string` in the type but **never read** by the client. Whatever format the admin uses for it does not affect mobile behavior; it appears to exist for the backend's own force-update computation.
- **Admin validation recommendation:** validate `latestVersion` (and `minSupportedVersion`) as **semver strings matching `nativeApplicationVersion`** (the `versionName` / `CFBundleShortVersionString`), not as integer build numbers.

---

## 5. Soft-update dismissal (24h-per-version)

**Confirmed — the app remembers soft-update dismissals per version, with a 24h window.** `src/lib/store/softUpdateDismissal.ts`:

- **AsyncStorage key:** `dismissed_optional_update_version_data`.
- **Stored shape:** `{ version: string; lastChecked: number }` (`version` = the `latestVersion` that was dismissed; `lastChecked` = `Date.now()` at dismissal).
- **`shouldShowSoftUpdate(latestVersion)`** returns `true` (show modal) when **any** of:
  - no stored record, OR
  - record older than 24h (`Date.now() - lastChecked > 86_400_000`), OR
  - record's `version !== latestVersion` (different version).

  On expiry or version mismatch it clears the stored record and returns `true`.
- **Dismissal:** "Kasnije" (Later) calls `dismissSoftUpdate(latestVersion)` storing the current `latestVersion` + timestamp.

**Answer to the admin's question:** **Yes — bumping the soft (`latestVersion`) value re-prompts users who dismissed a prior version**, immediately (different-version check), regardless of the 24h timer. If the admin re-sends the **same** `latestVersion` string, a user who dismissed it stays un-nagged for 24h, after which it re-shows. So: the **`latestVersion` string is the soft-prompt identity** — change it per release to re-engage dismissers; keep it stable to honor the 24h quiet period.

---

## For Mastermind (notes, not fixes)

1. **Orphaned parallel read path — `AppVersionConfigurationDialog`.** `src/components/dialog/dialogs/AppVersionConfigurationDialog.tsx` is an older dialog that consumes the same `AppVersionConfig` shape (`forceUpdate`/`optionalUpdate`/`latestVersion`) and is registered in `DialogManager.tsx:50` / `dialogRegistry.ts:19` (`APP_VERSION_CONFIGURATION_DIALOG`), but **nothing ever opens it** (no `openDialog(APP_VERSION_CONFIGURATION_DIALOG)` call anywhere). The **live** path is bootStore Gate 2 → `HardUpdateScreen` / `SoftUpdateModal`. Not a contract defect and out of this audit's write scope — flagging as dead code for a future cleanup. It does confirm the same field contract, so it's harmless to the admin work.
2. **Silent no-op on bad responses (by design, but worth the admin knowing).** A 4xx/5xx or malformed body from `/public/app/version/{platform}` is swallowed → no popup, boot continues (`appVersionConfigService.tsx:14-20`, `bootStore.ts:282-285`). Only a *hang* (gate timeout) routes to maintenance. So a buggy admin write that returns an error will **fail open** (users see no prompt), not fail closed. The admin view should treat a successful, well-formed `200` with correct booleans as the only way to actually drive a prompt.
3. **`minSupportedVersion` is server-only.** Confirmed unused on the client; if the admin form exposes it, its only consumer is the backend's force-update computation.
