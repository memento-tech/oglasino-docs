# Admin App Version Control

Admin-panel view to set the mobile app-update **floor** (`minSupportedVersion`) per platform — the value that forces users below it to hard-update. Styled like `/admin/cache`. Backend + web feature; mobile (`oglasino-expo`) is untouched (it already reads the version gate and obeys the backend-computed booleans).

**Status:** `planned`

## Background — the as-built version model

Three facts per platform, two stored server-side in the `app_version_config` table (one row per platform, `UNIQUE (platform)`; `android` and `ios` seeded at `0.0.0`/`0.0.0`):

- **Running version** — baked into the installed binary. Client-known.
- **Ceiling = `latestVersion`** — newest version available. Written automatically by the EAS post-build hook on every build (`POST /internal/app/version/ceiling`). Drives the **SOFT** popup.
- **Floor = `minSupportedVersion`** — minimum version still allowed. Human policy lever, set rarely. Drives the **HARD** popup. This is the value the admin view sets.

The mobile app does **not** compare versions itself. At boot (Gate 2) it calls `GET /api/public/app/version/{platform}?currentVersion={installed}` and obeys two booleans the backend computes:

- `forceUpdate = running < minSupportedVersion` → HARD blocking screen, boot stops.
- `optionalUpdate = running < latestVersion && !forceUpdate` → SOFT dismissable modal.
- neither → no popup.

The read path **fails open**: unknown platform, unparseable semver, or missing row all return HTTP 200 with a no-update DTO. It can never brick the app.

## The gap this feature closes

The floor is currently writable only via `POST /internal/app/version/floor`, guarded by a machine-to-machine shared-secret token (`X-INTERNAL-TOKEN`). The web admin client (`BACKEND_API`, Firebase-Bearer) cannot reach `/internal/**` and has no internal token. There is **no** admin-namespaced version endpoint today. So the floor cannot be set from the admin panel at all — this feature adds the missing admin-reachable write path and the view.

## Admin workflow

- **Soft popup ("you can update"):** the admin does nothing. The EAS post-build hook bumps the ceiling on every release; users below it see the dismissable modal automatically. The admin's only role is to glance at the view and confirm the ceiling looks fresh (provenance timestamp).
- **Hard popup ("you must update"):** the admin raises the floor for a platform. Every user below the new floor gets the blocking screen. Done rarely and deliberately. The 99% path is a one-click "Require latest version" button; manual entry is the escape hatch.

Per-platform: iOS and Android are independent. Forcing on one does not touch the other.

## Backend contract (to build)

New controller `@RequestMapping("/api/secure/admin/app/version")`, class-level `@PreAuthorize("hasRole('ADMIN')")` (mirrors `CacheAdminController` / `MaintenanceAdminController`). Admin identity derived server-side from `OglasinoAuthentication` in `SecurityContextHolder` (verified Firebase token) — never from a client-supplied role/flag (Part 11).

- **GET `/api/secure/admin/app/version`** — returns both platforms' rows: `platform`, `latestVersion`, `minSupportedVersion`, `updatedAt` (the `BaseEntity` timestamp — drives the provenance display "updated X ago"). No new column needed; `updatedAt` already exists.
- **POST `/api/secure/admin/app/version/floor`** — body `{platform, minSupportedVersion}`. Sets the floor. UPDATE-ONLY (404 on missing platform row, mirroring the internal path). Server-side validation the internal path lacks:
  - `minSupportedVersion` must parse as a strict three-part semver (`Version.parse`, java-semver 0.10.2). Reject otherwise with a Part-7 coded error (code, not message).
  - **Reject floor > ceiling** (`minSupportedVersion > latestVersion`) with a Part-7 coded error. A floor above the ceiling would force every user to a version that does not exist yet — total lockout. The "Require latest version" button sets floor = ceiling exactly, so this guard only ever catches a manual typo.

Extract a small `AppVersionService` so the semver-parse + floor-validation logic is shared rather than duplicated between the new admin write and the existing internal write. (The internal endpoints stay as-is functionally; whether they adopt the shared validation is the engineer's call at brief time — do not change their external behavior in this feature.)

The two `/internal/app/version/{floor,ceiling}` endpoints and the EAS post-build hook are **untouched**.

**Decision: no actor/audit column.** The row carries `updatedAt` (when) but not who. Not added — single-operator system, no second admin to disambiguate, and "in case we need it" does not earn the schema (Part 4a). Revisit if a second admin role appears.

## Web view (to build)

New route `app/[locale]/admin/version/page.tsx` — async server component, `getTranslations(ADMIN_PAGES)`, the `/admin/cache` page shell (`div.flex.w-full.max-w-3xl.flex-col.gap-8.py-4`, header h1 + description), mounting a `'use client'` panel `src/components/admin/version/VersionControlPanel.tsx`.

Panel mirrors `BackendCachePanel`:

- Fetches both platforms on mount via a new client service `src/lib/admin/lib/service/appVersionService.ts` calling `BACKEND_API.get('/secure/admin/app/version')`.
- Per platform, shows: **latest version** (read-only, with "updated X ago" from `updatedAt`), **minimum required version** (current floor), provenance timestamps.
- A **"Require latest version"** button per platform that fills the minimum-required field with the current `latestVersion` (does not save on its own).
- A manual semver input for the floor (the escape hatch). Client-side validate as three-part semver before submit.
- **Save** behind a confirm `Dialog` (mirror `BackendCachePanel`'s destructive-op confirm) reading e.g. "This will force all {platform} users below {version} to update. Continue?" POSTs to `/secure/admin/app/version/floor`.
- `notify.success` / `notify.error` toasts. Surface the backend's coded errors (bad semver, floor>ceiling) as translated messages.

Nav: one `NavItem` in `src/lib/data/sectionNavigation.ts` (`adminSectionNavigations`) — `labelKey` (COMMON namespace, e.g. `admin.version.label`), `url: '/admin/version'`, a `lucide-react` icon. No route registration beyond the folder (App Router).

## Translations

New `ADMIN_PAGES` keys for the panel (labels, button text, confirm copy, toast messages, the two coded errors) + one COMMON nav label. Seeded by the **backend** agent per Part 6 Rule 3; web identifies the exact key list in its brief, Igor passes it on. EN final; RS/RU/CNR placeholder pending native review (standing precedent).

## Trust boundary (Part 11)

- Admin authorization is server-side (`@PreAuthorize` reading the verified-token role). The client sends no role/flag.
- The floor value + platform are admin-supplied data, set by an already-authenticated admin. The backend validates the value (semver format, floor ≤ ceiling) server-side; it does not trust it blindly. The internal floor write does zero validation today — the admin path must not copy that gap.

## Out of scope / known items

- **EAS → version-list integration:** not built. The view drives off the two live backend values; no live call to EAS. A historical version picker is not backable (no history table) and not needed for the floor decision. Revisit only on real demand.
- **Dead mobile `AppVersionConfigurationDialog`** (`oglasino-expo`, registered but never opened) — cleanup for a future Expo Ω sweep, not this feature.
- **Unvalidated `/internal/app/version/floor` write** — pre-existing; not fixed here. The new admin endpoint adds the validation the internal path lacks.

## Owed at feature close (do NOT write now)

- `decisions.md` entry summarizing: backend+web (not web-only), the floor>ceiling reject, the no-actor-column decision, the EAS-integration rejection, the "Require latest" button design.
- `state.md`: flip status from `planned` through the pipeline to `shipped`; Expo backlog row if any mobile adoption is ever needed (none expected).
