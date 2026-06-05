# Audit — Expo Maintenance Split (boot/maintenance/API inventory)

**Repo:** oglasino-expo
**Branch:** `new-expo-dev` (confirmed via `git branch --show-current`)
**Type:** Read-only audit. No code changed.
**Date:** 2026-05-29
**Scope:** Inventory the current boot/maintenance path so the team can move maintenance detection off the dedicated `GET /public/maintenance/active` poll and onto a worker-gated `/api/mobile/*` prefix. Code on `new-expo-dev` is ground truth; prior boot/maintenance reports were not read.

---

## TL;DR

- The maintenance poll is **one** service function, `checkIfMaintenance()` in `src/lib/services/maintenanceService.tsx`, hitting `GET /public/maintenance/active`. It has exactly **two** callers: boot Gate 1 (`runMaintenanceGate`) and the exit poller (`MaintenancePollInit`). No other consumer.
- The boot redesign is a sequential **gate state machine** in `src/lib/store/bootStore.ts` (`useBootStore`). Maintenance is **Gate 1**. Entry to `'maintenance'` happens whenever **any gate cannot reach the backend within 5s** (the "single backend-down rule", `withGateTimeout` in `bootGate.ts`) — not only Gate 1. Exit is driven by `MaintenancePollInit`, a 5s interval that runs **only while `status === 'maintenance'`** and calls `reEnter()` when the backend reports maintenance off.
- The maintenance UI is **not** `HardUpdateScreen` and **not** a dedicated screen — it is `<BaseSiteSelector isMaintenance />` rendered from `app/_layout.tsx`. `HardUpdateScreen` is the separate `'update-required'` terminal screen.
- API base URL is a single env var **`EXPO_PUBLIC_API_URL`**, read once in `src/lib/config/api.ts`. There is **one axios instance** (`BACKEND_API`) — a true single chokepoint; every service imports it; zero other `axios.create`, zero backend `fetch()`, zero hardcoded backend base URLs.
- The base URL already includes a `/api` suffix per tier. Per-tier values live **only in local/EAS `.env.*` files** (now git-ignored and removed from tracking) — not in `app.config.ts` or `eas.json`.
- There is already a **response interceptor** in `api.ts` that handles 401 (token refresh) and 403 (USER_BANNED/EMAIL_BANNED). This is the natural place to detect a worker maintenance signal. Status `0` / `ERR_NETWORK` / no-response branches already exist there too.

---

## 1. The maintenance poll

### Service file / function

`src/lib/services/maintenanceService.tsx` (the entire file):

```ts
import { BACKEND_API } from '../config/api';
import { logServiceError, logServiceWarn } from '../utils/serviceLog';

export const checkIfMaintenance = async (): Promise<boolean> => {
  try {
    const response = await BACKEND_API.get<boolean>('/public/maintenance/active');

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

- **Path:** `GET /public/maintenance/active`, issued through the shared `BACKEND_API` axios instance (so it goes out with the same base URL + interceptors as every other call).
- **Returns:** a `boolean` — `true` = maintenance is ON. On HTTP 200 it returns `response.data` directly. On any non-200, or any thrown error/timeout, it **swallows the error and returns `true`** (fail-safe: "assume maintenance"). The function never throws.

### Where it is called in the boot sequence

`checkIfMaintenance()` has exactly **two** call sites (confirmed by repo-wide grep):

**(a) Boot Gate 1** — `src/lib/store/bootStore.ts:179-190`:

```ts
runMaintenanceGate: async () => {
  try {
    const maintenanceOn = await withGateTimeout(() => checkIfMaintenance());
    if (maintenanceOn) {
      get().toMaintenance();
      return;
    }
  } catch {
    // Timeout (GateTimeoutError) or any other throw → maintenance.
    get().toMaintenance();
    return;
  }
  // … then populates the `config` slot and advances to Gate 2 (version).
```

**(b) The exit poller** — `src/components/init/MaintenancePollInit.tsx:34-43` (see §2).

### How the result is consumed

- Gate 1: if `checkIfMaintenance()` returns `true` (or the 5s timeout fires, or it throws), the machine calls `toMaintenance()` → `set({ status: 'maintenance' })` and **stops the boot sequence**. If `false` and within timeout, boot advances to Gate 2.
- Note the **belt-and-suspenders** design: the service already returns `true` on error, *and* Gate 1 wraps it in `withGateTimeout` so a genuine hang (which the service's own catch could never see) still routes to maintenance.

---

## 2. The boot gate state machine (`bootStore`)

### Shape

`src/lib/store/bootStore.ts` defines `useBootStore` (Zustand). The status enum (`bootStore.ts:71-78`):

```ts
export type BootStatus =
  | 'booting'
  | 'offline'
  | 'maintenance'
  | 'update-required'
  | 'intro-picker'
  | 'updating'
  | 'ready';
```

It is a **sequential gate machine**, kicked once from `app/_layout.tsx`'s single empty-deps effect (`useBootStore.getState().start()`), and it advances by each gate calling the next on success. The three documented invariants: (1) one effect, empty deps; (2) machine writes `status`, view reads it; (3) re-entry destroys no resolved slot.

Gate order (`start()` → `runConnectivityGate` → …):

| Gate | Function | Purpose | Routes to `maintenance` when… |
|------|----------|---------|-------------------------------|
| 0 | `runConnectivityGate` (`:153`) | NetInfo offline check | never (offline → `'offline'`; ambiguous → proceed) |
| 1 | `runMaintenanceGate` (`:179`) | `checkIfMaintenance()` + `getAppConfiguration()` | maintenance ON, timeout, or throw |
| 2 | `runVersionGate` (`:221`) | force/optional update check | **timeout only** (`GateTimeoutError`) |
| 3 | `runBaseSiteGate` (`:265`) | stored site or overviews fetch | overviews fetch fails/times out |
| 4 | `runFreshnessGate` (`:389`) | `/versions`, catalog + per-namespace refetch, i18n register | any wrapped call times out or a required namespace can't be resolved |

### Which Gate handles maintenance / what triggers entry

Gate 1 is the *dedicated* maintenance gate, but **maintenance is the universal backend-down state**. The "single backend-down rule" is expressed once in `src/lib/store/bootGate.ts`:

```ts
export const GATE_TIMEOUT_MS = 5000;

export const withGateTimeout = <T>(fn: () => Promise<T>): Promise<T> => {
  return new Promise<T>((resolve, reject) => {
    const timerId = setTimeout(() => { reject(new GateTimeoutError()); }, GATE_TIMEOUT_MS);
    fn().then(
      (value) => { clearTimeout(timerId); resolve(value); },
      (err)   => { clearTimeout(timerId); reject(err); }
    );
  });
};
```

Every backend call inside a gate is wrapped with `withGateTimeout`; on timer-wins the gate catches `GateTimeoutError` and calls `toMaintenance()`. So Gates 2, 3, and 4 also enter `'maintenance'` on backend unreachability — not just Gate 1. The single writer is `toMaintenance: () => set({ status: 'maintenance' })` (`bootStore.ts:568`).

### What triggers exit — the `MaintenancePollInit` active-checker

`src/components/init/MaintenancePollInit.tsx` (mounted unconditionally in `app/_layout.tsx:78`):

```ts
const POLL_INTERVAL_MS = 5000; // matches the per-gate timeout (spec Part 2)

export async function pollMaintenanceOnce(): Promise<void> {
  try {
    const maintenanceOn = await checkIfMaintenance();
    if (!maintenanceOn) {
      await useBootStore.getState().reEnter();
    }
  } catch {
    // Error/timeout during the poll → treat as still maintenance; do nothing.
  }
}

export default function MaintenancePollInit() {
  const status = useBootStore((s) => s.status);

  useEffect(() => {
    if (status !== 'maintenance') return;

    const interval = setInterval(pollMaintenanceOnce, POLL_INTERVAL_MS);
    return () => clearInterval(interval);
  }, [status]);

  return null;
}
```

- **When it runs:** the interval is armed **only while `status === 'maintenance'`** (the effect early-returns otherwise) and torn down when status leaves maintenance. It is the one external event source allowed to re-drive the machine; it never writes `status` itself.
- **What it calls:** `checkIfMaintenance()` every 5s. When the backend reports maintenance **off** (`false`), it calls `useBootStore.getState().reEnter()`.
- `reEnter()` (`bootStore.ts:127-132`) is a non-destructive alias for `start()`: it resets `status` to `'booting'` and re-runs the gates from Gate 0, **without** clearing `selectedBaseSite` / `language` / `config` (Invariant 3), so the user's prior progress survives.

### How the UI renders maintenance

`app/_layout.tsx:117-127` — the overlay block renders for every non-`ready`, non-`update-required` status. The maintenance branch is **`<BaseSiteSelector isMaintenance />`**, *not* `HardUpdateScreen`:

```tsx
{bootStatus !== 'ready' && bootStatus !== 'update-required' && (
  <View className="absolute inset-0 items-center justify-center bg-background">
    {bootStatus === 'offline' && <BaseSiteSelector isOffline />}
    {bootStatus === 'maintenance' && <BaseSiteSelector isMaintenance />}
    {bootStatus === 'intro-picker' && <BaseSiteSelector isMaintenance={false} />}
    {(bootStatus === 'booting' || bootStatus === 'updating') && <LoadingOverlay />}
  </View>
)}

{/* Terminal-blocking hard-update screen (boot-redesign Gate 2). */}
{bootStatus === 'update-required' && <HardUpdateScreen />}
```

So three boot states (`offline`, `maintenance`, `intro-picker`) all reuse `BaseSiteSelector`, distinguished by props. `HardUpdateScreen` is a *separate* terminal screen for `'update-required'` (Gate 2 force-update) — unrelated to maintenance.

In maintenance mode `BaseSiteSelector` (`src/components/init/BaseSiteSelector.tsx`) hides the picker buttons (`messageOnly = isMaintenance || isOffline`) and shows two lines of copy. The maintenance copy is **hardcoded Serbian** (`INTRO_FALLBACK.maintenance.label.1/2`), because Gate 1 runs before Gate 4 inits i18n, so no backend-seeded string can resolve at that point. It does still try to fetch the `INTRO` namespace (`sr`) at mount and prefers that if it arrives.

---

## 3. API base URL configuration

### Where it is configured

`src/lib/config/api.ts:21-25` — a single env var, validated at module load:

```ts
const BACKEND_API_URL = process.env.EXPO_PUBLIC_API_URL;

if (!BACKEND_API_URL) {
  throw new Error('EXPO_PUBLIC_API_URL is not defined');
}
```

`app.config.ts` does **not** set `EXPO_PUBLIC_API_URL` (it only threads `EXPO_PUBLIC_FIREBASE_*` into `extra.firebaseConfig`). `eas.json` only sets `APP_ENV` per build profile (`development`/`preview`/`production`) — it does **not** set the API URL. So `EXPO_PUBLIC_API_URL` is resolved purely from Expo's `.env.<tier>` loading (and/or EAS environment variables at build time).

### Per-tier values

The `.env.*` files exist on disk but are **not in git** — `.gitignore` ignores `.env.*` (keeping only `.env.example`), and `git status` shows `.env.development` and `.env.production` as staged deletions (`D`); `.env.preview` was never tracked. `git ls-files` lists no `.env` files. The values therefore live only on Igor's machine / in EAS env, not in the committed tree:

| Tier | `EXPO_PUBLIC_API_URL` | Notes |
|------|----------------------|-------|
| development | `http://10.184.52.91:8080/api` | LAN IP, direct to backend `:8080` — **no Cloudflare in front** |
| preview | `http://172.21.173.91:8080/api` | LAN IP, direct to backend `:8080` — **no Cloudflare in front** |
| production | `https://oglasino.com/api` | public apex — **this is the only tier that traverses the Cloudflare worker** |

**Seam-critical observation:** the base URL already ends in `/api`. To route mobile through `/api/mobile/*`, the natural insertion point is this base URL (`…/api` → `…/api/mobile`) or the path prefix in `createApiInstance`. But note dev/preview hit a raw backend IP with no worker — the worker-gating mechanism only exists in production. Dev/preview would need either a local worker, a pointer at a worker-fronted host, or a documented "dev bypasses the gate" stance. Flagged to Mastermind in the session summary.

---

## 4. How API calls are made

**Single chokepoint — confirmed.** All backend traffic flows through one axios instance built by a factory (`src/lib/config/api.ts:30-38, 140`):

```ts
const createApiInstance = (baseURL: string): AxiosInstance => {
  const instance = axios.create({
    baseURL,
    timeout: 8000,
    headers: { 'Content-Type': 'application/json', Accept: 'application/json' },
  });
  // … request + response interceptors …
  return instance;
};

export const BACKEND_API = createApiInstance(BACKEND_API_URL);
```

Repo-wide grep found:
- `axios.create`: only the one call inside `createApiInstance`.
- backend `fetch()` calls / hardcoded backend base URLs: **none**. (The only literal URLs are non-API: `memento-tech.com` update links, `oglasino.com`/`oglasino.rs` product-share deep links, and `raw.githubusercontent.com` for privacy/terms markdown — none use `BACKEND_API` and none are backend API calls.)
- 23 modules import `BACKEND_API` (all `src/lib/services/*`, plus `i18n/fetchNamespace.ts`, `notifications/service/pushTokenService.ts`, `lib/init/*`, `lib/stores/viewTokens.ts`).

So routing everything through `/api/mobile/*` is a **single-point change** (the base URL / instance config), inherited by all 23 consumers automatically.

### Request interceptor (`api.ts:40-51`)

```ts
instance.interceptors.request.use(async (config) => {
  const currentUser = auth.currentUser;
  const { selectedBaseSite, language } = useBootStore.getState();
  if (selectedBaseSite) config.headers.set('X-Base-Site', selectedBaseSite.code);
  if (language) config.headers.set('X-Lang', language.code);
  if (currentUser && config.headers) {
    config.headers.set('Authorization', `Bearer ${(await currentUser.getIdToken()).toString()}`);
  }
  return config;
});
```

Adds `X-Base-Site`, `X-Lang`, and (when signed in) the Firebase bearer token.

---

## 5. Maintenance signal handling — where a worker 503 would be detected

There is already a **response interceptor** in `src/lib/config/api.ts:53-135` (added by the boot redesign, per the brief). Its structure, in order:

1. **Success handler:** checks `x-account-restored` header → `authHooks.onAccountRestored?.()`.
2. **Error handler** (in priority order):
   - `error.code === 'ERR_NETWORK' || 'ECONNABORTED'` → reject with `{ status: 0, errorCode: 'error.connection.timeout' }`.
   - `!error.response` → reject with `{ status: 0, errorCode: 'error.network.unreachable' }`.
   - `status === 403` + `isErrorWithCode(USER_BANNED|EMAIL_BANNED)` → `auth.signOut()` + `authHooks.onAccountBanned?.()` + return a never-resolving promise.
   - `status === 401` → Firebase token-refresh with a module-level `isRefreshing` flag + `pendingRequests` dedupe queue, single-retry guarded by `_retry`.
   - `status === 404` → reject with `{ status: 404, errorCode: 'error.not.found' }`.
   - else → `Promise.reject(error)`.

The interceptor wires into the auth store via a small hook indirection (`configureAuthInterceptorHooks` in `api.ts`, registered by `src/lib/init/authInterceptors.ts`, called at module load in `app/_layout.tsx:25`):

```ts
configureAuthInterceptorHooks({
  onAccountRestored: () => useAuthStore.getState().setRestored(true),
  onAccountBanned: () => useAuthStore.getState().setAccountBanned(true),
});
```

**This response interceptor is the natural seat for worker maintenance detection.** A worker `503` (with a known header/body) on `/api/mobile/*` would land in the error handler exactly alongside the existing 403/401/404 branches. The cleanest wiring mirrors the existing pattern: add a `503`/maintenance branch that calls a new hook (e.g. `onMaintenanceDetected`) wired via `configureAuthInterceptorHooks` to `useBootStore.getState().toMaintenance()`. That keeps `bootStore` free of the auth/HTTP layer (preserving its leaf-module status and Invariant 2 — `toMaintenance` is already a blessed `status` writer).

One caveat: the boot **gates** call services directly and already convert backend-down into maintenance via `withGateTimeout`. A worker `503` during a gate would arrive as a *fast* rejection, not a timeout — `checkIfMaintenance` swallows it to `true` (→ maintenance) so Gate 1 is already covered, but the other gates' service functions (`fetchVersions`, `fetchBaseSiteOverviews`, etc.) reject on non-2xx; their gate `catch` blocks already route to `toMaintenance` for "any throw" in the base-site/freshness gates, and to maintenance only on *timeout* in the version gate. A response-interceptor maintenance branch would make this uniform.

---

## 6. Seams

**(a) Cleanest signal shape for the worker to return.** A dedicated HTTP **status code the app does not already overload** plus a discriminating header. `503` is the semantically correct code, but the interceptor currently treats *only* `401/403/404` specially and lets everything else fall through to `Promise.reject`. Recommend the worker return **`503` with an explicit header such as `X-Oglasino-Maintenance: true`** (and optionally a small JSON body `{ "maintenance": true }`). The app should key off the header rather than the bare status, because a generic origin `503` (a real backend 500-class blip) is semantically different from an operator-flagged maintenance window — the header lets mobile distinguish "show the maintenance gate" from "retryable error". Status `503` alone is acceptable as a fallback if a header is impractical, but the header is more robust.

**(b) Orphaned consumers of `/public/maintenance/active`.** **None beyond the two known call sites.** `checkIfMaintenance` is imported only by `bootStore.ts` (Gate 1) and `MaintenancePollInit.tsx` (exit poll). Removing the endpoint affects only those two. All other "maintenance" references in the repo are the `status` enum value, state transitions, UI props (`isMaintenance`), and comments — none call the endpoint. **Related note:** `src/lib/services/healthCheckService.ts` (`GET /public/health/check`) has **zero callers** — it is already dead code; mentioned here because a worker liveness probe would more naturally consume something like it, but as-is it is unused (flagged as an adjacent observation).

**(c) Does the `MaintenancePollInit` exit mechanism still work under the new design?** **Not as-is** — it is hard-wired to `checkIfMaintenance()` → `/public/maintenance/active`. If that endpoint is removed and maintenance is detected via the `/api/mobile/*` gate, the exit poller must be re-pointed. Two clean options:
   1. **Keep the poller, change its probe:** have `pollMaintenanceOnce` issue a cheap `/api/mobile/*` request (e.g. a lightweight liveness/health route through the worker) and treat a non-maintenance response as "clear" → `reEnter()`. This preserves the existing 5s-while-in-maintenance interval, the `reEnter()` contract, and Invariant 3 untouched.
   2. **Event-driven exit:** if the worker returns a "back online" signal on the next allowed request, drive `reEnter()` from that. But while the machine sits in `'maintenance'` the portal Stack is unmounted (`app/_layout.tsx:95`), so **no app traffic is in flight to carry that signal** — meaning some probe is still required. Option 1 (re-pointed poll) is therefore the lower-risk path; the poller's structure stays, only its target URL changes.

   Either way, the `reEnter()` / non-destructive re-entry machinery is independent of *how* maintenance-clear is detected, so the state-machine half survives; only the probe (`maintenanceService`) needs to change.

---

## Files inspected (read-only)

- `src/lib/services/maintenanceService.tsx`
- `src/lib/store/bootStore.ts`, `src/lib/store/bootGate.ts`
- `src/components/init/MaintenancePollInit.tsx`
- `app/_layout.tsx`
- `src/components/init/BaseSiteSelector.tsx`
- `src/lib/config/api.ts`, `src/lib/init/authInterceptors.ts`, `src/lib/utils/isErrorWithCode.ts`
- `src/lib/services/healthCheckService.ts`
- `app.config.ts`, `eas.json`, `.env.development`, `.env.preview`, `.env.production`, `.gitignore`
- Repo-wide greps for `maintenance`, `checkIfMaintenance`, `BACKEND_API`, `axios.create`, backend `fetch`/URLs, `EXPO_PUBLIC_API_URL`, `health`.
