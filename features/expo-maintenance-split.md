# Expo Maintenance Split

**Slug:** `expo-maintenance-split`
**Status:** shipped
**Repos:** oglasino-router, oglasino-backend, oglasino-expo, oglasino-web
**Branches:** router `stage`, backend `dev`, expo `new-expo-dev`, web `stage`

---

## 1. Problem

The platform has one maintenance flag, `maintenance.active`, that conflates two
independent concerns: web availability and backend availability. Mobile depends
only on the backend. When web goes into maintenance but the backend is live,
mobile is needlessly locked out. The flag also has two writers (deploy workflows
and a backend toggle) and is stored in two places (Cloudflare KV and a backend DB
config row), which can drift.

This feature splits maintenance into two dependency flags, makes Cloudflare KV the
single store of maintenance state, routes mobile API traffic through a worker-only
`/api/mobile/*` label gated on backend availability, and removes the backend's
dedicated maintenance poll.

## 2. Model

Two dependency flags in the Cloudflare `CONFIG` KV namespace:

- `maintenance.web.active` — web's maintenance state (web deploys, web-specific outages).
- `maintenance.backend.active` — backend's maintenance state (backend deploys, backend outages).

Replaces the single `maintenance.active`. The `admin.bypass.disabled` and
`use.backend.check` flags are unchanged in name and meaning.

Client maintenance is **composed** from the dependency flags, per request, in the worker:

- **Web request** is in maintenance when `maintenance.web.active OR maintenance.backend.active`.
  (Web cannot function without backend, so a backend-down also takes web down.)
- **Mobile request** (`/api/mobile/*`) is in maintenance when
  `maintenance.backend.active OR (probe says backend is down)`.
  Mobile does not depend on web, so `maintenance.web.active` does not affect mobile.

Default-when-absent for every flag is `false` (= surface is up), matching the
worker's existing strict `=== "true"` semantics. Any reader must treat a `null`
KV value as "up," never "down."

## 3. The worker (oglasino-router)

### 3.1 Flag reads

The worker reads `maintenance.web.active` and `maintenance.backend.active`
(replacing the single `maintenance.active` read) plus the unchanged
`admin.bypass.disabled` and `use.backend.check`. All KV reads keep
`{ cacheTtl: 30 }`. All flag reads MUST sit inside the existing fail-open
try/catch — including the `use.backend.check` read, which currently sits outside
it (a pre-existing bug, issues.md). On any KV throw, all flags fall back to `false`
(fail-open).

### 3.2 Request classification

A request is **mobile** when its path starts with `/api/mobile/`. `/api/mobile/*`
is a worker-only label; it never forwards to the backend under that prefix (see 3.4).
All other requests retain their current classification (apex web vs API host vs
admin-locale-prefixed, per the existing `isAdminRequest` predicate).

### 3.3 Composition and gate

- For a **mobile** request: compute `backendDown = maintenance.backend.active OR
  probeFailed`. If `backendDown`, return the maintenance response (3.5). Otherwise
  strip the `/mobile` segment and forward to the backend origin (3.4).
- For a **web** request: compute `webDown = maintenance.web.active OR
  maintenance.backend.active`. The existing admin-bypass logic is unchanged —
  `shouldBlock = adminBypassDisabled OR !isAdminRequest`; if `webDown && shouldBlock`,
  return the maintenance response; otherwise forward.

The admin bypass (`admin.bypass.disabled`) applies to the web/admin path exactly as
today. It does not apply to `/api/mobile/*` — mobile has no admin surface.

### 3.4 Mobile routing and the probe

`/api/mobile/<rest>` maps to backend `/api/<rest>` — the worker strips the `/mobile`
segment before forwarding to `BACKEND_ORIGIN`. The `/mobile` token exists only so the
worker can identify mobile traffic and apply the mobile composition; the backend never
sees it.

The backend liveness probe already exists (gated by `use.backend.check`, edge-cached
at `cacheTtl: 30`, fail-closed-into-maintenance on non-2xx or throw). Two changes:

- The probe target changes from `/health` to `/actuator/health/readiness`. `/health`
  returns 200 even when the backend's dependencies (Postgres/Redis/Elasticsearch) are
  down; `/actuator/health/readiness` (with the backend's readiness group made
  dependency-aware — see 4.4) returns non-2xx when a dependency is down, which is what
  "backend can serve traffic" requires.
- The probe gates `/api/mobile/*` requests. A failed probe (non-2xx or throw) sets
  `backendDown = true` for the mobile composition.

The probe's edge cache (`cacheTtl: 30`, `cacheEverything: true`) is what bounds backend
load: regardless of how many mobile clients hit the worker, the worker probes the
backend at most once per cache-TTL per edge location, not once per client request.

The worker MUST be able to route `/actuator/health/readiness` to the backend origin for
the probe (it is an outbound `fetch` from the worker, not a client-facing route, so this
is the worker calling origin directly — confirm the probe URL resolves against
`BACKEND_ORIGIN`).

### 3.5 Maintenance response shape

Mobile maintenance response (the `/api/mobile/*` blocked case):
- Status `503`.
- Header `X-Oglasino-Maintenance: true`.
- Header `Retry-After: 120`, `Cache-Control: no-store`.
- JSON body `{"status":"maintenance","message":"...","retryAfter":120}` (the existing
  `MAINTENANCE_JSON` constant shape).

This matches the worker's existing API-path maintenance response. Mobile keys off the
`X-Oglasino-Maintenance` header, not the bare 503, so a genuine backend 5xx blip is not
mistaken for an operator maintenance window.

### 3.6 Matrix comment

The top-of-file matrix comment must be updated to document the two dependency flags,
the per-client composition, the `/api/mobile/*` label, the probe target, and
`use.backend.check` (currently undocumented).

## 4. The backend (oglasino-backend)

### 4.1 Cloudflare KV is the sole maintenance store

The backend no longer stores or reads maintenance state of its own. Specifically:

- Delete the `maintenance.active` row from the backend `Configuration` seed
  (`data-configuration.sql`).
- Delete `GET /api/public/maintenance/active` (the endpoint Expo polls today). It is
  dead once Expo stops polling (web does not consume it; confirmed). **Sequencing:**
  Expo must stop polling before this endpoint is deleted (see 7).

### 4.2 The admin wrench writes both KV flags

`DefaultCloudflareKvService.toggleMaintenance()` is kept and reworked:

- It writes **both** `maintenance.web.active` and `maintenance.backend.active` to
  Cloudflare KV. Toggle-on writes both `"true"`; toggle-off writes both `"false"`.
  Pressing the wrench = "platform maintenance, everything down."
- It reads current state from Cloudflare KV (not from the backend DB config, which is
  being removed). Toggle logic: read both flags from KV; if either is on, turn both
  off; if both off, turn both on. This adds a KV `GET` to the service.
- The hardcoded `MAINTENANCE_KEY = "maintenance.active"` literal is replaced by the two
  new key strings.

### 4.3 The toggle endpoint moves to secure + admin-gated

`POST /api/public/maintenance/toggle` moves to `POST /api/secure/maintenance/toggle`
(or the established admin namespace) with `@PreAuthorize("hasRole('ADMIN')")`. This
closes the pre-existing hole where the toggle was unauthenticated under `/api/public/`.
`MaintenancePageController` shrinks accordingly (the `/active` read endpoint is deleted
per 4.1; only the moved, gated toggle remains, in whatever controller is appropriate
for the secure namespace).

### 4.4 Readiness becomes dependency-aware

Add to each profile's actuator config (`application-prod.yaml`, `application-stage.yaml`;
dev has no actuator block and is out of scope for the probe):

    management.endpoint.health.group.readiness.include: readinessState,db,redis,elasticsearch

This makes `/actuator/health/readiness` reflect real dependency health (Postgres, Redis,
Elasticsearch), so the worker probe against it answers "can the backend serve traffic,"
not merely "is the JVM up." It also fixes the latent Docker-healthcheck gap where the
container reports healthy with a dead dependency (the compose healthcheck already targets
readiness).

### 4.5 Deploy workflow KV flips

Both backend deploy workflows (`deploy-backend.yml` → `main`, `deploy-stage.yml` → `stage`)
flip KV on deploy. After this feature:

- Backend deploy flips **both** `maintenance.backend.active = "true"` AND
  `maintenance.web.active = "true"`. (A backend deploy almost always rides with a web
  change — new translations, republish — so both go down together; web can stay down
  past backend's return on its own flag.)
- `admin.bypass.disabled` continues to be flipped as today (unchanged).

The flips are one-way (on only); maintenance-off stays manual via the Cloudflare console.

## 5. Mobile (oglasino-expo, branch new-expo-dev)

### 5.1 Route all API traffic through `/api/mobile/*`

The single axios instance (`BACKEND_API`, the sole chokepoint, 23 importers) gets its
base URL changed so every backend call goes through the worker's `/api/mobile/*` label.
The base URL already ends in `/api`; it becomes `…/api/mobile`. No per-call changes;
all 23 consumers inherit it.

### 5.2 Per-tier API URLs

- **development:** local backend directly, no worker, no `/mobile` (or `/mobile`
  harmlessly unstripped against local backend — see note). Dev relies on the boot
  machine's 5s `withGateTimeout` fallback to show maintenance when local backend is
  unreachable; there is no worker signal in dev. Keep dev pointed at the local backend.
- **preview:** `https://api-stage.oglasino.com/api/mobile` — through the **stage worker**.
  This replaces the current raw LAN IP (`http://172.21.173.91:8080/api`). Preview is the
  stage environment in mobile terms; this is where the full maintenance-split flow is
  rehearsed before production.
- **production:** `https://oglasino.com/api/mobile` — through the prod worker. Replaces
  `https://oglasino.com/api`.

Note on dev: dev hits the local backend with no worker, so `/api/mobile/*` has no
stripping layer. Dev's base URL stays at the local `…/api` (no `/mobile`), since there
is no worker to interpret the label. The `/mobile` prefix is a production/preview
concern only. The implementing brief decides whether this is a per-tier base-URL
difference or a conditional prefix; either is acceptable.

### 5.3 Remove the dedicated maintenance poll

`checkIfMaintenance()` (`maintenanceService.tsx`, `GET /public/maintenance/active`) is
removed as the boot-time and exit-poll mechanism. Its two callers change:

- **Boot Gate 1** (`runMaintenanceGate` in `bootStore.ts`): no longer calls
  `checkIfMaintenance()`. Maintenance during boot is detected the same way every other
  backend-down is detected — a gate's backend call returns the worker's `503` +
  `X-Oglasino-Maintenance: true`, or times out via `withGateTimeout`. Gate 1's dedicated
  maintenance probe is replaced by the worker signal arriving on whatever the first
  backend call is. The implementing brief decides whether Gate 1 keeps a lightweight
  liveness call through `/api/mobile/*` or folds into the next gate's call; either is
  acceptable as long as a worker `503`/maintenance header routes to `toMaintenance()`.
- **Exit poller** (`MaintenancePollInit.tsx`): re-pointed from `checkIfMaintenance()` to
  a cheap `/api/mobile/*` request through the worker. A non-maintenance response (not a
  503-with-maintenance-header) means "clear" → `reEnter()`. The 5s-while-in-maintenance
  interval and the `reEnter()` non-destructive re-entry are unchanged. A poll is
  required (not event-driven exit) because the portal Stack is unmounted during
  maintenance, so no app traffic carries a "back online" signal.

### 5.4 Detect the worker maintenance signal in the response interceptor

The existing axios response interceptor (`api.ts`, which already handles 401/403/404)
gains a maintenance branch: when a response is `503` with header
`X-Oglasino-Maintenance: true`, it calls a new hook `onMaintenanceDetected` →
`useBootStore.getState().toMaintenance()`. The hook is wired via the existing
`configureAuthInterceptorHooks` pattern (alongside `onAccountRestored`,
`onAccountBanned`), keeping `bootStore` a leaf module that imports no HTTP/auth code.
Branch on the header, not the bare 503.

### 5.5 Maintenance UI unchanged

The maintenance state still renders `<BaseSiteSelector isMaintenance />` from
`app/_layout.tsx`. No UI change.

## 6. Web (oglasino-web)

Two deploy workflows flip KV: `deploy-stage.yml` (→ `stage`) and `deploy-prod.yml`
(→ `main`). Each flips **only** `maintenance.web.active = "true"` (replacing the
`maintenance.active` flip). Mobile is unaffected by a web deploy — the whole point of the
split. `admin.bypass.disabled` continues unchanged. The flip is one-way; off is manual.

The web admin `MaintenanceToggle` UI button now drives the reworked backend toggle
(4.2/4.3) — which writes both KV flags and is admin-gated. No web-side logic change
beyond pointing at the moved endpoint path.

Web reads no maintenance state at runtime (confirmed); no other web change.

## 7. Sequencing

The rename and the endpoint removal are coordinated across repos. Order:

1. **Stage worker deploy.** The stage worker is running pre-split code (last deploy
   predates the split). Igor deploys the current worker to stage first, so stage has the
   two-flag logic before anything writes the new keys. (Engineer agents do not deploy;
   Igor runs it.)
2. **Router:** implement two-flag composition, `/api/mobile/*` routing, probe-target
   change, matrix comment. Igor deploys to stage, then prod.
3. **Backend:** readiness group (4.4), `DefaultCloudflareKvService` rework (4.2), toggle
   move + gate (4.3). Deploy-workflow key changes (4.5). Do NOT yet delete
   `GET /public/maintenance/active` or the config row.
4. **Expo:** route through `/api/mobile/*` (5.1/5.2), remove the poll and re-point the
   exit poller (5.3), interceptor maintenance branch (5.4). This is what stops the
   `/public/maintenance/active` calls.
5. **Backend cleanup:** once Expo no longer polls, delete `GET /public/maintenance/active`
   and the `maintenance.active` config row (4.1).
6. **Web:** deploy-workflow key change (6); point the admin toggle at the moved endpoint.

The rename touch-points across all repos (every site that writes/reads the old
`maintenance.active` string): backend `deploy-backend.yml` + `deploy-stage.yml` env keys,
backend `DefaultCloudflareKvService` literal, web `deploy-stage.yml` + `deploy-prod.yml`
env keys, router worker reads + matrix comment, `docs/12-deployment.md`, and the droplet
scripts (operator handles by hand). The KV namespace itself: the new keys must exist (or
default-absent = up) in both stage and prod CONFIG namespaces.

## 8. Trust boundary

Maintenance flags are operator-supplied KV state, not user-supplied input — no Part 11
trust-boundary concern on the flags themselves. The one trust fix in scope: the toggle
endpoint moves from unauthenticated `/api/public/` to admin-gated `/api/secure/` (4.3).
The `/api/mobile/*` label carries no authority — it is a routing hint the worker applies;
the backend never trusts it (the worker strips it before forwarding).

## 9. Definition of done

- Worker reads two dependency flags, composes per-client, routes `/api/mobile/*`, probes
  `/actuator/health/readiness`, returns the documented 503+header on mobile maintenance;
  matrix comment updated; `use.backend.check` read moved inside fail-open; tests cover
  the two-flag matrix and the mobile path. Deployed to stage and prod.
- Backend readiness group is dependency-aware; `DefaultCloudflareKvService` writes both
  flags and reads from KV; toggle moved to `/api/secure/` with `@PreAuthorize`;
  `/public/maintenance/active` and the config row deleted; deploy workflows flip the new
  key names (backend: both flags; — see 4.5).
- Expo routes all API traffic through `/api/mobile/*` per tier (preview →
  api-stage.oglasino.com, prod → oglasino.com, dev → local); the dedicated poll is gone;
  the interceptor detects the worker maintenance signal; the exit poller is re-pointed;
  boot and exit both verified against the stage worker on a preview build.
- Web deploy workflows flip only `maintenance.web.active`; the admin toggle drives the
  moved endpoint.
- End-to-end rehearsal on preview against stage: flip `maintenance.backend.active` in
  stage KV → preview mobile shows maintenance; flip `maintenance.web.active` only →
  preview mobile unaffected, web shows maintenance; clear → mobile exits maintenance via
  the re-pointed poll.

## 10. Adjacent items (logged to issues.md, fixed by this feature)

- Unauthenticated `POST /api/public/maintenance/toggle` — closed by 4.3 (moved to secure,
  admin-gated).
- `use.backend.check` read outside the worker's fail-open try/catch — closed by 3.1.
- Worker matrix comment missing `use.backend.check` — closed by 3.6.
- `docs/12-deployment.md` stale "testing mode" note and the maintenance-off curl — updated
  for the new key names as part of the backend rename.

## Session log

- **2026-05-29 → 05-30** — Implemented across all four repos (sequencing per §7). Archived engineer sessions: router `oglasino-router-expo-maintenance-split-1..3` (+ `oglasino-router-maintenance-active-sweep-1`); backend `oglasino-backend-expo-maintenance-split-1..3` (+ `oglasino-backend-maintenance-active-sweep-1..2`); web `oglasino-web-expo-maintenance-split-1..3` (+ `oglasino-web-maintenance-active-sweep-1`); expo `oglasino-expo-maintenance-split-1..3`. Two-flag worker composition, `/api/mobile/*` routing, `/actuator/health/readiness` probe, KV-as-sole-store, admin-toggle move to `/api/secure/`, and the mobile poll removal all landed.
- **2026-05-30** — Docs sweep, PART 1 (`oglasino-docs-expo-maintenance-split-1` audit, `-2` content sweep): eight live docs moved to the two-flag model; `issues.md` updated. PART 2 (status flip + a paired `decisions.md` supersession entry) deferred behind the stage-rehearsal gate.
- **2026-05-31** — End-to-end stage rehearsal passed (Igor-confirmed): backend-flag flip shows mobile maintenance; web-flag-only flip leaves mobile unaffected; clear exits mobile maintenance. Status → `shipped` on Igor's direct confirmation. **Still owed:** the paired `decisions.md` supersession entry (superseding the 2026-05-15 single-flag entry) — Mastermind-drafted, not yet available; the 2026-05-15 entry remains accurate-as-history until then.
