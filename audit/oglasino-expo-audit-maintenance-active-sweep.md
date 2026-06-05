# Audit — `maintenance.active` / `/public/maintenance/active` removal sweep

**Repo:** oglasino-expo
**Branch:** `new-expo-dev` (confirmed via `git branch --show-current`)
**Type:** Read-only audit. No code changed.
**Date:** 2026-05-30
**Feature:** [expo-maintenance-split](../../oglasino-docs/features/expo-maintenance-split.md) (status `planned`)
**Scope:** Repo-wide sweep for every "maintenance" reference. Classify each as (a) a call to the backend `/public/maintenance/active` endpoint that must be removed, (b) the client boot-state-machine `'maintenance'` status / UI (legitimate, stays), (c) the exit poller, or (d) unrelated. Goal: give the backend an independent confirmation of whether the Expo side still calls `/public/maintenance/active`, so it can safely delete that endpoint (spec §7 step 5 is gated on "Expo no longer polls").

> **Context:** the Expo §5 implementation already landed earlier today — session [`2026-05-30-oglasino-expo-maintenance-split-2.md`](2026-05-30-oglasino-expo-maintenance-split-2.md). That session removed the poll and grep-confirmed zero callers. This sweep is the **independent verification artifact** the backend cleanup step depends on; it re-establishes that fact from a clean repo-wide grep and classifies every residual reference.

---

## TL;DR (headline for the backend)

**There are ZERO live calls to `/public/maintenance/active` anywhere in this repo.** From the Expo side, the spec's sequencing step 4 ("Expo stops polling") is **complete** (done in session `-2`) — the backend's step 5 (delete `GET /public/maintenance/active` + the `maintenance.active` config row) is **unblocked**. Deleting that endpoint will break nothing in `oglasino-expo`.

- The service that hit it — `checkIfMaintenance()` in `src/lib/services/maintenanceService.tsx` — has been **deleted** (session `-2`). No file, no function, no caller. (`find -iname "*maintenance*"` and grep for `checkIfMaintenance` / `maintenanceService` / `maintenance.active` all return nothing in source.)
- Gate 1 and the exit poller were re-pointed onto `healthCheck()` → **`GET /public/health/check`**; `healthCheckService.ts` went from dead-code to live (two callers); the worker-`503` + `X-Oglasino-Maintenance` interceptor branch (spec §5.4) exists (`api.ts:81–87`) and is wired via `onMaintenanceDetected` (`authInterceptors.ts:12`).
- The single remaining textual mention of the old endpoint is a **documentary comment**, not a call: `bootStore.ts:172` (`// No dedicated /public/maintenance/active poll anymore`).

**One caveat the backend cleanup brief must honour (see §5):** the Expo exit poller and boot Gate 1 now depend on a *different* backend endpoint, **`/public/health/check`**. The cleanup must delete *only* `/public/maintenance/active` (+ the config row) and **keep `/public/health/check`** — removing it would break Gate 1 and the exit poller (every non-200 reads as "still maintenance" → never-exit).

---

## Classification of every hit

Repo-wide, case-insensitive grep for `maintenance` across `*.ts/*.tsx/*.js/*.json` (excluding `node_modules`, `.git`, `ios`, `android`, `.expo`), plus targeted greps for `/public/maintenance`, `checkIfMaintenance`, `maintenanceService`, `maintenance.active`, and the liveness/health probe. Every hit below.

### (a) Calls to backend `/public/maintenance/active` — must be removed

**NONE.** The only reference to the endpoint string is documentation:

| file:line | line content | note |
|-----------|--------------|------|
| `src/lib/store/bootStore.ts:172` | `// No dedicated /public/maintenance/active poll anymore: a lightweight backend` | **Comment, not a call.** Accurately documents the endpoint's removal. Harmless to keep; optionally trim once the feature ships. |

No `BACKEND_API.get('/public/maintenance/active')`, no `checkIfMaintenance`, no `maintenanceService` file. Category (a) is empty.

### (b) Boot-state-machine `'maintenance'` status / UI — legitimate, STAYS

This is the bulk of the hits. It is the **client maintenance STATE** and the **worker-signal detection mechanism** — explicitly in-scope-to-keep per the brief and spec §5.4/§5.5.

**Status enum, single writer, and gate routing** — `src/lib/store/bootStore.ts`:
- `:73` — `| 'maintenance'` (BootStatus enum member)
- `:93` — `runMaintenanceGate: () => Promise<void>;` (Gate 1 type)
- `:103` — `toMaintenance: () => void;` (the single blessed `status` writer, type)
- `:571` — `toMaintenance: () => set({ status: 'maintenance' }),` (the one writer)
- `:182–193` — `runMaintenanceGate` body (Gate 1; shares the `healthCheck()` probe — see (c))
- `:186,191,243,292,314,401,417,437,470,494,516` — `get().toMaintenance()` calls: every gate routes backend-down/timeout/throw to maintenance (the "universal backend-down state"). Legitimate state transitions.
- `:62,65,127,129,134–150,159,168,171,173,175,177,190,214–215,266–267,288,291,316,360,385,399,408,411,424,458,468,505,514` — comments describing maintenance routing semantics. Documentation; stays.

**Maintenance UI** — `app/_layout.tsx`:
- `:120` — `{bootStatus === 'maintenance' && <BaseSiteSelector isMaintenance />}` (the maintenance screen, spec §5.5: "UI unchanged")
- `:87,111,113,121` — comments + the `intro-picker` sibling branch reusing `BaseSiteSelector isMaintenance={false}`. Stays.

`src/components/init/BaseSiteSelector.tsx`:
- `:18` — `isMaintenance?: boolean;` (prop)
- `:38` — `maintenance: { … }` (fallback-copy key)
- `:52,53,56` — `isMaintenance` destructure + `messageOnly` logic (hides picker in maintenance/offline)
- `:119,122` — `tIntro('maintenance.label.1'|'.2')` (hardcoded-Serbian maintenance copy; Gate 1 precedes i18n init). Stays.

**Worker-`503` maintenance detection (spec §5.4)** — `src/lib/config/api.ts`:
- `:9` — `onMaintenanceDetected?: () => void;` (hook type)
- `:78–80` — comment: "Worker maintenance signal: 503 + the discriminating header…"
- `:81–87` — the branch: `if (status === 503 && headers['x-oglasino-maintenance'] === 'true') { authHooks.onMaintenanceDetected?.(); … }`
- `src/lib/init/authInterceptors.ts:12` — `onMaintenanceDetected: () => useBootStore.getState().toMaintenance(),` (wires the hook to the boot machine, keeping `bootStore` a leaf module). Stays — the worker-signal path.

**Gate routing comments in other services** (documentation of (b)):
- `src/lib/init/baseSitesService.ts:37,39` — comments re routing unreachable → maintenance.
- `src/lib/init/versionsService.ts:11` — comment re timeout/throw → maintenance.
- `src/lib/store/bootGate.ts:6` — comment ("…`maintenance`. This wrapper is the one place that rule is expressed.").

**The offline mirror** — `src/components/init/OfflineReconnectInit.tsx:8,11,14,20,31,53` — comments cross-referencing `MaintenancePollInit` as the design template for the `'offline'` reconnect poller. Sibling of (c), not the maintenance path itself; stays.

**Tests of (b)** — legitimate coverage, stays:
- `src/lib/store/bootStore.test.ts` — ~40 hits across `:284–1382` exercising gate ordering, `toMaintenance()`, and every gate's backend-down→maintenance routing.
- `src/lib/config/api.interceptor.test.ts:17,22,43,48–72` — the worker-`503` interceptor branch (fires on 503+header; ignores bare 503; ignores header on non-503).

### (c) The exit poller — STAYS (already re-pointed off the deleted endpoint)

`src/components/init/MaintenancePollInit.tsx` (whole file) — the active-checker. While `status === 'maintenance'`, a 5s interval calls `pollMaintenanceOnce()`; on backend-up it calls `reEnter()` (non-destructive re-entry, Invariant 3). **Migrated in session `-2`**: its probe is now `healthCheck()`, *not* the deleted `checkIfMaintenance()`.
- `:2` — `import healthCheck from '../../lib/services/healthCheckService';`
- `:28–35` — comment describing the probe as a backend-liveness call through the worker's `/api/mobile/*` label
- `:37–46` — `pollMaintenanceOnce()`: `const backendUp = await healthCheck(); if (backendUp) await reEnter();`
- `:48–59` — the component: interval armed only while `status === 'maintenance'`.

`src/lib/services/healthCheckService.ts` (whole file) — the shared liveness probe used by both the exit poller AND Gate 1:
- `:5` — `const res = await BACKEND_API.get('/public/health/check');` → returns `true` on 200, `false` otherwise/on throw.
- **Note:** in the 2026-05-29 planning audit this file was flagged as **dead code (zero callers)**. Session `-2` wired it live (two callers: `MaintenancePollInit` + Gate 1) as the replacement probe.

`app/_layout.tsx:7,78` — `import MaintenancePollInit` + `<MaintenancePollInit />` mount (unconditional; self-gates on status).

`src/components/init/MaintenancePollInit.test.ts:9,17,19,32,37,40,48` — tests `pollMaintenanceOnce` (re-enters on backend-up; does NOT re-enter while still maintenance). Mocks `healthCheckService`. Stays.

### (d) Unrelated

| file:line | reason |
|-----------|--------|
| `src/components/icons/dynamic/RepairsMaintenanceCategoryIcon.tsx:3,19` | A **listing-category icon** ("Repairs & Maintenance"). Nothing to do with system maintenance. |
| `src/components/icons/dynamic/index.ts:103` | Re-export of the above icon. |
| `src/components/icons/dynamic/index.ts:57,58` | `HealthBeautyCategoryIcon` / `HealthCareCategoryIcon` — matched the `health` probe grep only; category icons, unrelated. |

---

## What changed since the 2026-05-29 planning audit (for traceability)

The prior `audit-expo-maintenance-split.md` (2026-05-29) was the *planning* audit, accurate when written. The §5 implementation landed in session [`2026-05-30-oglasino-expo-maintenance-split-2.md`](2026-05-30-oglasino-expo-maintenance-split-2.md). Deltas:

| Concern | 2026-05-29 planning audit | Current tree (post session `-2`) |
|---------|---------------------------|----------------------------------|
| Maintenance probe service | `checkIfMaintenance()` in `maintenanceService.tsx` → `GET /public/maintenance/active` | **Deleted.** Gate 1 + poller call `healthCheck()` → `GET /public/health/check`. |
| `healthCheckService.ts` | dead code, zero callers | **Live** — two callers (Gate 1, exit poller). |
| Worker-`503` interceptor branch (§5.4) | did not exist; "natural seat" recommended | **Implemented** — `api.ts:81–87` + `onMaintenanceDetected` hook wired in `authInterceptors.ts:12`. |
| `/api/mobile/*` routing (§5.1/§5.2) | base URL ends in `/api` | **Code-complete and intentionally env-only** — `api.ts` is a dumb reader of `EXPO_PUBLIC_API_URL`; the `/mobile` segment is a per-tier env value, not code (see §5). |

This sweep independently confirms session `-2`'s zero-callers result and adds the `/public/health/check` cleanup caveat (below).

---

## §5 — Adjacent observations (Mastermind / cross-repo)

Neither item is a category-(a) hit; neither blocks the backend's endpoint deletion. Surfaced per the closure gate because they bear on whether maintenance detection works end-to-end.

1. **`/api/mobile/*` routing is code-complete and env-only — the pending piece is Igor's env values, not code.** Per session `-2`, `api.ts` reads `EXPO_PUBLIC_API_URL` verbatim; routing mobile traffic through the worker's `/api/mobile/*` label is achieved by setting the per-tier env value, with zero code in the single chokepoint. The on-disk `.env.*` (read-only) still carry the **old** values without `/mobile`:
   - development: `http://10.184.52.91:8080/api` — *stays* `…/api` by design (no worker in dev; Gate 1's 5s `withGateTimeout` shows maintenance when the local backend is down)
   - preview: `http://172.21.173.91:8080/api` — must become `https://api-stage.oglasino.com/api/mobile`
   - production: `https://oglasino.com/api` — must become `https://oglasino.com/api/mobile`

   **Implication:** until Igor sets the preview/prod env values to `…/api/mobile`, the worker never classifies mobile traffic as mobile, so the `api.ts:81–87` worker-`503` branch stays latent in those tiers. This is a **pending operator/env step already tracked in session `-2`'s "For Mastermind,"** not a code gap and not a defect. Noted here only so the backend/router teams know the worker-signal path isn't exercised end-to-end until the env flip + preview rehearsal (DoD §9). (`.env.*`/`app.config.ts`/`eas.json` are outside my hard rules to edit without explicit instruction — flagging, not changing.)

2. **Maintenance detection now depends on a second backend endpoint, `/public/health/check` — keep it during cleanup.** Gate 1 (`bootStore.ts:182–193`) and the exit poller (`MaintenancePollInit.tsx:37–46`) decide maintenance from `healthCheck()` → `GET /public/health/check` (200 = up). This is a valid reading of spec §5.3 ("the implementing brief decides whether Gate 1 keeps a lightweight liveness call … either is acceptable"). Consequence for the **backend cleanup brief (§7 step 5)**: delete *only* `/public/maintenance/active` (+ the config row); **`/public/health/check` must survive**. If it is removed, every poll/Gate-1 call reads non-200 → permanent maintenance / never-exit. (This caveat is the main thing this sweep adds beyond session `-2`.)

Neither point changes the headline: **no Expo code calls `/public/maintenance/active`; the backend may delete it.**

---

## Files inspected (read-only)

- `src/lib/store/bootStore.ts`, `src/lib/store/bootGate.ts`, `src/lib/store/bootStore.test.ts`
- `src/components/init/MaintenancePollInit.tsx`, `src/components/init/MaintenancePollInit.test.ts`, `src/components/init/OfflineReconnectInit.tsx`, `src/components/init/BaseSiteSelector.tsx`
- `src/lib/services/healthCheckService.ts`
- `src/lib/config/api.ts`, `src/lib/config/api.interceptor.test.ts`, `src/lib/init/authInterceptors.ts`, `src/lib/init/baseSitesService.ts`, `src/lib/init/versionsService.ts`
- `app/_layout.tsx`
- `.env.development`, `.env.preview`, `.env.production`, `.env.example`
- Prior `.agent/audit-expo-maintenance-split.md` + `.agent/2026-05-29-oglasino-expo-maintenance-split-1.md` + `.agent/2026-05-30-oglasino-expo-maintenance-split-2.md` (for delta traceability)
- Repo-wide greps: `maintenance` (all source extensions), `/public/maintenance`, `checkIfMaintenance`, `maintenanceService`, `maintenance.active`, liveness/`readiness`/`/health`; `find -iname "*maintenance*"`.

_No source files changed. Output is this audit doc only._
