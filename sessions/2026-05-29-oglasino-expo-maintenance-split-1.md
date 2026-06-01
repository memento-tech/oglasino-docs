# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-29
**Task:** Read-only audit of the current boot/maintenance path on `new-expo-dev`; write findings to `.agent/audit-expo-maintenance-split.md`. Inventory the `GET /public/maintenance/active` poll, the `bootStore` gate state machine + `MaintenancePollInit`, API base-URL config, the API chokepoint/interceptors, and where a worker maintenance signal would be detected.

## Implemented

- Read-only audit only — no source changes. Produced `.agent/audit-expo-maintenance-split.md` covering all six brief points with verbatim `file:line` quotes.
- Confirmed branch `new-expo-dev` before starting.
- Key findings: maintenance poll is `checkIfMaintenance()` in `maintenanceService.tsx` (`GET /public/maintenance/active`) with exactly two callers (boot Gate 1 + `MaintenancePollInit`); the boot redesign is a sequential gate machine in `bootStore.ts` where `'maintenance'` is the universal backend-down state (any gate's 5s `withGateTimeout` miss → `toMaintenance`), not only Gate 1; maintenance UI is `<BaseSiteSelector isMaintenance />`, not `HardUpdateScreen`; API base URL is the single env var `EXPO_PUBLIC_API_URL` feeding one axios instance `BACKEND_API` (true single chokepoint, 23 importers, zero bypasses); an existing response interceptor handles 401/403/404 and is the natural seat for a worker `503` maintenance branch.
- Cross-checked a 3-agent read-only workflow's findings against my own direct reads; resolved one sub-agent error (it claimed `.env.*` files are git-tracked — they are not; two are staged deletions, all three are gitignored).

## Files touched

- `.agent/audit-expo-maintenance-split.md` (new, read-only audit output)
- `.agent/2026-05-29-oglasino-expo-maintenance-split-1.md` (this summary)
- `.agent/last-session.md` (exact copy)
- No source files changed.

## Tests

- Not run. Read-only audit; no code changed, so lint/tsc/test are not applicable to touched paths (only `.agent/*.md` written).

## Cleanup performed

- none needed (no code touched).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change required by this session. (This audit is Phase 2 for a not-yet-active feature; it does not adopt anything off the Expo backlog table, so no backlog row is cleared. No implicit config-file dependency exists.)
- issues.md: no change. One adjacent observation flagged below for Mastermind triage (dead `healthCheckService`); not authored into issues.md (Docs/QA is sole writer).

## Obsoleted by this session

- nothing (read-only).

## Conventions check

- Part 4 (cleanliness): confirmed — no code touched; audit doc only.
- Part 4a (simplicity): see structured evidence in "For Mastermind". No abstractions introduced (read-only).
- Part 4b (adjacent observations): one flagged below (dead `healthCheckService.ts`).
- Part 6 (translations): N/A this session (no translation keys touched). Noted in the audit that the maintenance overlay copy is hardcoded Serbian `INTRO_FALLBACK.maintenance.label.*` by design (Gate 1 precedes i18n init).
- Other parts touched: Part 8 (architectural defaults) — confirmed the worker-is-edge-boundary principle is consistent with the proposed `/api/mobile/*` routing, but noted dev/preview tiers hit raw backend IPs with no worker in front.

## Known gaps / TODOs

- none (audit is complete to the brief's six points).

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code added.
  - Considered and rejected: nothing — no implementation choices were in scope.
  - Simplified or removed: nothing — read-only.

- **Seam decisions surfaced (detail in the audit §3, §5, §6):**
  1. **Tier asymmetry (high relevance to the split design).** Per-tier `EXPO_PUBLIC_API_URL`: development `http://10.184.52.91:8080/api` and preview `http://172.21.173.91:8080/api` hit raw backend LAN IPs with **no Cloudflare worker in front**; only production (`https://oglasino.com/api`) traverses the worker. The worker-gated `/api/mobile/*` maintenance signal therefore only exists in production. The plan needs a stance for dev/preview (local worker, worker-fronted host, or an explicit "dev bypasses the gate"). The base URL already ends in `/api`, so inserting `/api/mobile` is a one-line base-URL/instance change inherited by all 23 `BACKEND_API` consumers.
  2. **Recommended worker signal shape:** `503` + a discriminating header (e.g. `X-Oglasino-Maintenance: true`), keyed off the header rather than bare status so a real origin `503` blip isn't conflated with an operator maintenance window. The existing response interceptor (`api.ts:53-135`) is the clean insertion point; mirror the existing `configureAuthInterceptorHooks` pattern with an `onMaintenanceDetected` hook → `useBootStore.getState().toMaintenance()`, keeping `bootStore` a leaf module.
  3. **`MaintenancePollInit` exit mechanism does not survive endpoint removal as-is** — it is hard-wired to `checkIfMaintenance()`. Lowest-risk fix: keep the 5s-while-in-maintenance poller and re-point its probe at a cheap `/api/mobile/*` route. An event-driven exit can't work alone because the portal Stack is unmounted during maintenance, so no app traffic is in flight to carry a "back online" signal — a probe is still required. The `reEnter()` / non-destructive re-entry machinery is independent of detection and survives untouched.

- **Adjacent observation (Part 4b):**
  - `src/lib/services/healthCheckService.ts` (`GET /public/health/check`) has **zero callers** — dead code. Severity: low (cosmetic; no user-facing impact). I did not remove it because it is out of scope for a read-only audit. Possibly relevant to the split, since a worker liveness probe would naturally consume a health route — Mastermind may want to either wire it into the new poll or delete it.

- **Possible cross-repo dependency for the router agent:** the worker's `/api/mobile/*` 503+header contract (seam 6a/6c above) is the wire shape mobile will consume. If/when the router brief is written, the header name and body shape should be frozen there and reflected back so mobile's interceptor branch and the re-pointed exit poll match exactly.
