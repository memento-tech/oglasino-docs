# Audit — Admin App Version Control (web)

**Repo:** oglasino-web
**Branch:** dev (clean working tree)
**Date:** 2026-06-06
**Type:** Read-only reconnaissance. No code changes.
**Brief:** `.agent/brief.md` — inventory the `/admin/cache` page as a styling reference, search for any existing app-version-control surface, and document the admin write-endpoint pattern + trust boundary.

---

## Part A — The `/admin/cache` page (styling reference)

### File inventory

| Role | Path |
| --- | --- |
| Route (server component) | `app/[locale]/admin/cache/page.tsx` |
| Backend-cache panel (client) | `src/components/admin/cache/BackendCachePanel.tsx` |
| Frontend-cache panel (client) | `src/components/admin/cache/CacheEvictionPanel.tsx` |
| Backend cache service (client axios) | `src/lib/admin/lib/service/cacheManagementService.ts` |
| Frontend cache server actions | `app/actions/cacheActions.ts` |
| Service tests | `src/lib/admin/lib/service/cacheManagementService.test.ts` |

The route is one of ten siblings under `app/[locale]/admin/` (products, users, suggestions, reviews, reports, statistics, config, translations, cache, es). A new `version` view would be an eleventh sibling directory `app/[locale]/admin/version/page.tsx`.

### Structure

The page is a **server component** (`AdminCachePage`, `async`) that does almost nothing itself: it pulls translations from the `ADMIN_PAGES` namespace via `getTranslations`, renders a header (`h1` title + description), then mounts two client panels separated by an `<hr>`:

1. **`BackendCachePanel`** — talks to the Spring backend. This is the panel a version-control view most resembles: it fetches a list server-state on mount and offers per-row + global mutations.
2. **`CacheEvictionPanel`** — talks to Next.js's own data cache via server actions (`revalidateTag`/`revalidatePath`). Less relevant to version control, which is backend state.

The page itself reads **no data server-side** — both panels are `'use client'` and fetch on mount via `useEffect`. (Contrast with `/admin/config/page.tsx`, which reads `searchParams` server-side and passes a params object down.)

### `BackendCachePanel` anatomy (the closest template)

- **Initial data:** client-side. `loadDescriptors()` runs in a `useEffect([])`, calls `getAdminCacheList()`, stores `AdminCacheDescriptor[]` in state. Tracks `isLoading` and `loadError` separately.
- **Mutations:** per-row (`onClearOne`, `onClearAndWarmup`) and global (`onClearAll`, `onRefreshAll`, `onWarmupOnly`). Each is an `async` handler that sets a pending flag, awaits the service call, then fires a toast.
- **Pending UX:** `pendingRow` (`{name, action}`) for a single row, `isGlobalPending` for bulk actions; `anyPending` disables every button while any op is in flight. Spinners are `Loader2Icon` with `animate-spin`.
- **Confirmation:** destructive bulk ops (`clearAll`, `refreshAll`) open a shadcn `Dialog` (`ConfirmTarget` state) before executing. Single-row ops do not confirm.
- **Error mapping:** local `describeError(err)` maps `err.status` 403/401/≥500 to translated strings, with an `isAxiosErrorLike` type guard. This is the panel-local pattern — note it differs from the service-layer `logServiceError`/return-sentinel pattern used by `configService.ts` (the cache service re-throws the axios response so the panel can read `.status`).

### Styling approach

All UI is shadcn + Tailwind utility classes (no CSS modules):

- **Page wrapper:** `<div className="flex w-full max-w-3xl flex-col gap-8 py-4">` with a `<header>` holding `h1.text-2xl.font-semibold` + `p.text-primary-mild.text-sm`.
- **Panels** are `<section className="bg-background-strong flex flex-col gap-4 rounded-lg border p-4">`. `CacheEvictionPanel` defines a local `Section` helper with the same shell for sub-sections, plus a `LabeledInput` helper.
- **Buttons:** `Button` from `@/components/shadcn/ui/button`, variants `outline` (default action), `default` (primary, e.g. Refresh All), `ghost` (reload). Icons from `lucide-react` (`RefreshCw`, `Trash2`, `Loader2Icon`) sized `h-4 w-4` / `h-3.5 w-3.5` with `mr-2`.
- **Inputs:** `Input` from `@/components/shadcn/ui/input`.
- **Dialog:** `@/components/shadcn/ui/dialog` (`Dialog`, `DialogContent`, `DialogHeader`, `DialogTitle`, `DialogDescription`, `DialogFooter`).
- **List rows:** `<ul className="divide-y">` with each `<li>` a flex row, the cache name in `<code className="font-mono">`, action buttons right-aligned.
- **Toasts:** `notify` from `@/src/lib/config/toast.tsx` (sonner-backed custom toaster). `notify.success(...)` / `notify.error(...)` take `{ id, title, description? }`. IDs are made unique with `Date.now()`. Titles/descriptions come from the `ADMIN_PAGES` translation namespace (`cache.backend.toast.*`, `cache.frontend.toast.*`).

### Navigation wiring

- The sidebar is data-driven. `src/lib/data/sectionNavigation.ts` exports `adminSectionNavigations: NavItem[]`. The cache entry:
  ```ts
  { labelKey: 'admin.cache.label', url: '/admin/cache', icon: RefreshCw, collapsible: false }
  ```
  `labelKey` resolves against the **COMMON** namespace (the owner/admin nav labels live there — see `admin.*.label` keys). A new view needs one new array entry with a `labelKey`, `url`, and a `lucide-react` icon.
- `src/components/admin/AdminSidebar.tsx` feeds `adminSectionNavigations` into `NavMain` (`@/components/client/secure/NavMain`). The sidebar shell is shadcn `Sidebar` with `collapsible="icon"`.
- `app/[locale]/admin/layout.tsx` wraps every admin route in `<SessionGuard isAdminRoute={true}>` + `SidebarProvider` + `AdminSidebar` + `TopNavigation portalScope="admin"`. **No route registration beyond creating the folder** — App Router file-system routing. A new view is: (1) create `app/[locale]/admin/version/page.tsx`, (2) add one `NavItem` to `adminSectionNavigations`.

### Fetch/mutation pattern summary

- **Backend state** (the version-control case): client axios service in `src/lib/admin/lib/service/` calling `BACKEND_API`, consumed by a `'use client'` panel that fetches on mount and mutates via handlers + toasts. **No server action involved.**
- **Next.js data cache** (not relevant here): server actions in `app/actions/cacheActions.ts` guarded by `ensureAdmin()`.

A version-control view should mirror the **`BackendCachePanel` + `cacheManagementService` (client axios)** path, not the server-action path.

---

## Part B — Existing version-control surface on web

**Nothing exists.** Confirmed explicitly.

Searches run across `src/` and `app/` (excluding `node_modules`/`.next`) for: `app version`, `version ceiling/floor`, `force-update`, `forceUpdate`, `soft update`, `min version`, `/internal/app/version`, `versionChecksum`, `mustUpdate`, `canUpdate`, `update popup`, and a broad bare `version` sweep.

The only `version` hits are unrelated:

- `src/lib/admin/lib/types/es.ts` + `EsClusterStatePanel.tsx` — Elasticsearch **cluster** version display.
- `src/lib/consent/*`, `src/components/client/consent/*`, `src/lib/store/useConsentStore.ts` — the consent-cookie **schema** `version: 1` literal.
- `app/[locale]/design/topics.ts` — prose using the word "version."

There is:

- **No** service call, type, or component referencing app version / version ceiling / force-update.
- **No** admin route or component that sets version values. (Suspicion confirmed: none exists.)
- **No** TypeScript types/DTOs for any version endpoint — web consumes nothing version-related. There is no `/internal/app/version` reference anywhere in the web repo.

A version-control view would be greenfield on the web side: new route, new panel component(s), new client service, and new DTO type(s).

---

## Part C — The write endpoint, from web's side

### Is there a version-write service wrapper?

No. There is no service that POSTs/PUTs to any version-control write endpoint. (Follows directly from Part B.)

### The established admin-write pattern

Every admin write in this repo goes through the **client axios instance** `BACKEND_API` (`src/lib/config/api.ts`) against paths prefixed **`/secure/admin/...`**. Confirmed across all admin services:

```
/secure/admin/cache/list                     GET
/secure/admin/cache/{name}/clear             POST
/secure/admin/cache/{name}/warmup            POST
/secure/admin/config                         POST  (list)
/secure/admin/config/update                  POST  (write)
/secure/admin/maintenance                    GET
/secure/admin/maintenance/toggle             POST  (state-control write)
/secure/admin/translations/update            POST  (write)
/secure/admin/review/approve-disapprove      POST
/secure/admin/report/resolve                 POST
/secure/admin/es/reindex                     POST
/secure/admin/users, /suggestion, /report, /review, /stats   POST/GET
```

Two service-layer shapes coexist:

1. **`configService.ts` style** (most admin services): `try/catch`, on non-2xx call `logServiceWarn`/`logServiceError` and **return a sentinel** (`false`, `EMPTY_PAGE`). The caller gets a boolean/empty result, never an exception. `updateConfig` and `toggleMaintenance` are the canonical write examples — both POST and return `boolean`.
2. **`cacheManagementService.ts` style**: **re-throws** the axios response (`throw res`) on non-2xx so the panel can branch on `.status` (used for its 403/401/500 toast mapping).

For a version-control **write**, the `configService.updateConfig` / `toggleMaintenance` shape is the closest precedent (POST a DTO to `/secure/admin/...`, return success boolean). `MaintenanceToggle.tsx` (`src/components/client/buttons/`) is the single most analogous existing surface: a state-control toggle that reads current state on mount (`isMaintenanceActive()`) and flips it (`toggleMaintenance()`) — exactly the read-current-value-then-write shape a version-control panel needs.

### Auth on the write path

`BACKEND_API` (`src/lib/config/api.ts`) attaches auth via a **request interceptor**: it reads `auth.currentUser` (Firebase), mints/caches the ID token (`getIdTokenResult`, refreshed 1 min early), and sets `Authorization: Bearer <token>` plus `X-Base-Site` and `X-Lang` headers. `withCredentials: true`. The base URL is `NEXT_PUBLIC_API_URL`, which **already includes `/api`** (`.env.local.example`: `https://api.oglasino.com/api`; `.env.local`: `http://localhost:8080/api`). So a web path of `/secure/admin/version/...` resolves to backend `/api/secure/admin/version/...`.

### Backend namespace where a version-write endpoint must live

Admin writes reach the backend under **`/api/secure/admin/**`**, never `/internal/**`. For a version-write endpoint to be admin-reachable through the existing `BACKEND_API` + token pattern, it must live at **`/api/secure/admin/version/...`** (web path `/secure/admin/version/...`). An `/internal/**` endpoint would not be reachable by this client pattern and would not carry the admin Bearer token.

---

## Trust boundary note

**The web admin-write path is server-trusted, not client-trusted — by construction.** No admin identity is asserted from the client:

- The client only attaches a **Firebase ID token** (`Authorization: Bearer`). It does not send any "I am admin" flag, role, or userId that the backend would trust.
- The backend verifies the token (`FirebaseAuthFilter` → `SecurityContextHolder` with `OglasinoAuthentication`, per conventions Part 11) and gates `/secure/admin/**` server-side. The web `isAdmin()` check (`serverAuthCheck.ts`) and the layout `<SessionGuard isAdminRoute>` are **UX gates only** — they hit `/secure/admin` (200+`true` for admins) to decide whether to render, but they are not the security boundary. The backend's authorization on the endpoint is.
- `cacheActions.ts` server actions additionally re-verify via `ensureAdmin()` server-side before acting, but those touch only Next.js's own cache, not backend state.

**Implication for the new feature:** a version-control write endpoint placed at `/api/secure/admin/version/...` inherits the same server-side admin gate (assuming the backend applies its standard `@PreAuthorize`/admin check on that namespace — that is the **backend agent's** responsibility to confirm; this audit cannot see backend code). From the web side there is **nothing client-trusted** to flag: the panel sends the new ceiling/floor/force-update values as a request body, and the authenticated admin identity is derived entirely server-side from the verified token. The version values themselves (the ceiling/floor numbers) are admin-supplied input — they are a state-control payload, so the backend must validate/authorize them server-side, but they are *data* set by an already-authenticated admin, not an identity/authorization claim the client is forging.

The one thing to verify cross-repo (out of web's scope): that the chosen backend endpoint actually sits under the admin-gated namespace and carries the `@PreAuthorize` admin check, rather than being exposed on an ungated `/internal/**` or `/public/**` path. If it lands on `/internal/**`, web cannot reach it through `BACKEND_API` anyway, which would force a different (and likely weaker) integration — flag to Mastermind/Backend.

---

## Summary for a mirror implementation

To build the new admin version view mirroring `/admin/cache`:

1. **Route:** `app/[locale]/admin/version/page.tsx` — `async` server component, `getTranslations(ADMIN_PAGES)`, wrapper `div.flex.w-full.max-w-3xl.flex-col.gap-8.py-4`, header, mount a client panel.
2. **Panel:** `src/components/admin/version/VersionControlPanel.tsx` — `'use client'`, fetch current values on mount (`useEffect`), inputs (shadcn `Input`) for ceiling/floor + a force-update control, save button, `notify.success`/`notify.error` toasts, pending/disabled state, optional confirm `Dialog` for the force-update flip (it force-logs-out users — destructive, mirror `BackendCachePanel`'s confirm pattern).
3. **Service:** `src/lib/admin/lib/service/versionService.ts` — `BACKEND_API.get`/`.post` against `/secure/admin/version/...`, `configService`-style (return boolean/sentinel) or `cacheManagementService`-style (re-throw for status-based toasts) — pick to match the panel's error UX.
4. **Types:** new DTO(s) for the read/write shape (none exist today).
5. **Nav:** one `NavItem` in `adminSectionNavigations` (`src/lib/data/sectionNavigation.ts`) with a new `labelKey` (COMMON namespace, e.g. `admin.version.label`) + a `lucide-react` icon.
6. **Translations:** new `ADMIN_PAGES` keys for the panel + one COMMON nav label. Per conventions Part 6, the keys are seeded by the **Backend** agent; web identifies the list, Igor passes it on.
