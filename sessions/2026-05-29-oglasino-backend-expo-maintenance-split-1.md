# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-29
**Task:** Read-only audit of the backend's role in the expo maintenance-split feature (deploy KV flips, health-check endpoint, actuator liveness/readiness, `/api/mobile/*` absence, backend KV writes, seam). Findings written to `.agent/audit-expo-maintenance-split.md`.

## Implemented

- Nothing implemented. This was a Phase 2 read-only audit; no code changed.
- Produced `.agent/audit-expo-maintenance-split.md` answering all six brief items, each cited to `file:line` and verified by direct read.
- Investigation fanned out across five parallel read-only subagents, then every load-bearing claim was independently re-verified against source before writing the report. Two subagent claims were found wrong and corrected (see "Brief vs reality" and the audit's "Cross-check corrections").

## Key findings (condensed)

1. **Deploy KV flips:** `deploy-backend.yml` (push→`main`) and `deploy-stage.yml` (push→`stage`) each `PUT` `maintenance.active=true` and `admin.bypass.disabled=true` to Cloudflare KV, `sleep 60`, then probe the edge expecting `503`. Maintenance-off is manual (droplet script / curl `--data "false"`). `ci-dev.yml` writes no KV.
2. **`/api/public/health/check`** (`HealthCheckController.java:8-16`): bare `200 OK`, empty body, zero dependencies, unauthenticated, language-exempt (`CurrentLanguageFilter.java:47`), not rate-limited. Returns `200` even if DB/Redis/ES are all down.
3. **Actuator:** liveness + readiness both exposed. Readiness group is the Spring default = `readinessState` only (no `group.readiness.include` config) — it does **not** re-check db/redis/es; the aggregate `/actuator/health` does. `CacheWarmupService.java:76` flips readiness to `ACCEPTING_TRAFFIC` post-warmup. Docker healthcheck targets `/actuator/health/readiness` (`docker-compose.yml:93`).
4. **`/api/mobile/*`:** no backend route/controller/security-matcher. Safe as a worker-only label.
5. **Backend KV writes (brief belief refuted):** `DefaultCloudflareKvService.toggleMaintenance()` writes `maintenance.active` to Cloudflare KV, reachable via the **unauthenticated** `POST /api/public/maintenance/toggle`. No autonomous self-monitoring; the write is request-triggered.
6. **Seam:** `/api/public/health/check` (or `/health`) = "origin alive/routable" probe; aggregate `/actuator/health` = "deps healthy / can serve" probe. Readiness, as configured, answers neither dependency-health question reliably.

## Files touched

- `.agent/audit-expo-maintenance-split.md` (new, audit deliverable)
- `.agent/2026-05-29-oglasino-backend-expo-maintenance-split-1.md` (this summary) + `.agent/last-session.md` (copy)
- No source files touched.

## Tests

- None run. Read-only audit; no code changed, so `./mvnw test` / `spotless:check` were not applicable.

## Cleanup performed

- None needed (no code changed).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change. (The maintenance-split feature is not yet a row in state.md; this is its Phase 2 audit. If/when Mastermind opens it, Docs/QA would add it — drafting that is Mastermind's call, not this session's.)
- issues.md: no change made. **Three candidate entries are DRAFTED in "For Mastermind"** for Docs/QA to apply if Mastermind agrees (unauthenticated maintenance toggle; stale `docs/12-deployment.md` "testing mode" note; readiness-group dependency-awareness gap). Per hard rules I do not write issues.md myself.

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code changed; no debug logging, no commented-out code, no stray files.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): three flagged in "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: Part 8 (edge boundary) and Part 11 (trust boundary) — relevant to the §5 finding that the backend is a second writer of `maintenance.active` KV and exposes an unauthenticated toggle; surfaced, not acted on. Part 12/13: N/A.

## Known gaps / TODOs

- The readiness-group composition (§3/§6) was determined by reading config + Spring defaults, not by curling a running instance. Confidence is high, but a 30-second runtime check (`curl localhost:8080/actuator/health/readiness` vs `.../actuator/health` with one dependency stopped) would make it incontrovertible. Recommended before the feature commits to a probe target. I could not do this read-only without running the app (dev profile has no actuator config anyway, so a local run would not reflect prod behavior).
- `maintenance-on.sh` / `maintenance-off.sh` live on the droplet, not in this repo — their exact contents were not auditable here (the off-curl shape is documented in `docs/12-deployment.md:133-142`).

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **Brief vs reality (resolved within audit, no blocker):**
  1. Brief item 5 said "we believe no backend code flips Cloudflare KV; confirm." **Refuted** — `DefaultCloudflareKvService.toggleMaintenance()` writes `maintenance.active` to CF KV via `POST /api/public/maintenance/toggle`. Reported in audit §5. (No autonomous self-monitoring, which is what the literal wording implied — but a real write path exists.)
  2. A subagent's "stage workflow writes to prod KV namespace" claim was **wrong** and dismissed (env var `CF_KV_NAMESPACE_ID` is sourced from `secrets.CF_KV_NAMESPACE_ID_STAGE`).
  3. A subagent's "readiness probe checks db/redis/es" claim was **wrong** and corrected (default readiness group = `readinessState` only). This is the crux of the seam answer, so flagging explicitly.

- **Adjacent observations (Part 4b):**
  1. **Unauthenticated maintenance toggle — severity HIGH.** `POST /api/public/maintenance/toggle` (`MaintenancePageController.java:19-24`) is `permitAll()` with no `@PreAuthorize`; anyone reaching the origin can flip platform maintenance and write to Cloudflare KV. The edge Worker may gate it, but the backend doesn't. Tensions with conventions Part 8 (Worker is the edge boundary / sole owner of maintenance state) — backend is currently a **second writer** of `maintenance.active`. Did not fix (out of scope; read-only). *Draft issues.md entry:* title "Unauthenticated `POST /api/public/maintenance/toggle` can flip platform maintenance + write CF KV", severity high, status open, found-in `controller/MaintenancePageController.java:19`, `service/impl/DefaultCloudflareKvService.java:38-58`, `security/config/SecurityConfig.java:77`.
  2. **Stale deployment doc — severity LOW.** `docs/12-deployment.md:85` describes the prod `maintenance-on` job as "Currently in testing mode: wait is 0 s and verify is a single non-retrying check." The actual `deploy-backend.yml:73-95` already has `sleep 60` + a 5-attempt retry loop. The doc is stale relative to code. (This is an editable existing `docs/` file, not a config file — Mastermind may route a docs fix.) *Draft issues.md entry:* severity low, status open, found-in `docs/12-deployment.md:85`.
  3. **Readiness not dependency-aware — severity LOW/MEDIUM (design).** `/actuator/health/readiness` reflects only `readinessState`; a runtime DB/Redis/ES outage will not flip it to DOWN, yet it is the Docker container healthcheck target (`docker-compose.yml:93`). For container orchestration this means a dependency-dead-but-process-alive backend stays "healthy." Consider `management.endpoint.health.group.readiness.include: readinessState,db,redis,elasticsearch`. Surfaced as design input for the seam decision. *Draft issues.md entry:* severity low, status open, found-in `application-prod.yaml:111-127`, `docker-compose.yml:93`.

- **Seam recommendation (audit §6):** if the worker-side backend liveness probe must mean "backend can serve API traffic," probe the aggregate `/actuator/health` (and ensure the edge can route `/actuator/**`), or make readiness dependency-aware (obs. 3) and probe readiness. If it only needs "origin is up," `/api/public/health/check` is already the correct token-free target.

- **Closure gate:** No config-file edit was made by this session. Three issues.md entries are drafted above for Docs/QA to apply *only if* Mastermind agrees; none is "drafted but silently pending." conventions/decisions/state require no edit from this session.
