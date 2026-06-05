# Audit — Expo maintenance-split (backend role)

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-29
**Type:** Read-only audit (Phase 2). No code changes.
**Brief:** `.agent/brief.md` — inventory the backend's role in splitting the maintenance flag into `maintenance.web.active` + `maintenance.backend.active`, adding a worker-side backend liveness probe, and routing mobile through a worker-only `/api/mobile/*` label.

Current code on disk is treated as ground truth. Prior reports about maintenance/deploys were not read (per brief). Every claim below is cited to `file:line` and was verified by direct file read, not only by the parallel investigation.

---

## TL;DR

1. **Deploy KV flips** live in two GitHub Actions workflows (`deploy-backend.yml` → `main`, `deploy-stage.yml` → `stage`). Each flips **two** KV keys to `true`: `maintenance.active` and `admin.bypass.disabled`. `sleep 60` for propagation, then probe the **edge** expecting `503`. Maintenance is **never flipped off by the workflow** — that is a manual step. `ci-dev.yml` writes no KV.
2. **`/api/public/health/check` exists** (`HealthCheckController`), returns a bare `200 OK` empty body, touches **nothing** (no DB/Redis/ES), is unauthenticated, language-filter-exempt, and not rate-limited. It returns `200` even if every dependency is down.
3. **`/actuator/health/liveness` and `/actuator/health/readiness` both exist.** Critically, **readiness is the Spring default group = `readinessState` only** (there is **no** `group.readiness.include` config). It does **not** re-check DB/Redis/ES at runtime. The db/redis/elasticsearch indicators feed the **aggregate** `/actuator/health`, which is the only surface reflecting live dependency health. Docker's container healthcheck targets `/actuator/health/readiness`.
4. **No backend route/controller/security-matcher under `/api/mobile/`.** Confirmed safe as a worker-only label.
5. **The brief's item-5 belief is REFUTED.** The backend *does* write maintenance state outward to Cloudflare KV — via `DefaultCloudflareKvService.toggleMaintenance()`, reachable through the **unauthenticated** `POST /api/public/maintenance/toggle`. There is no *autonomous self-monitoring* that flips KV, but a request-triggered KV write path exists.
6. **Seam:** for "is the origin process alive and routable," `/api/public/health/check` (or `/health`) is the right cheap probe. For "can the backend serve *real* API traffic (deps healthy)," neither `health/check` nor `readiness` answers it as currently configured — only the aggregate `/actuator/health` does. Details in §6.

---

## 1. Deploy workflow Cloudflare KV flips

**Workflows present** (`.github/workflows/`): `ci-dev.yml`, `deploy-backend.yml`, `deploy-stage.yml`.

### KV key strings (exact)

Both deploy workflows declare, at the top-level `env` block:

- `deploy-backend.yml:14` — `MAINTENANCE_KEY: maintenance.active`
- `deploy-backend.yml:15` — `ADMIN_BYPASS_KEY: admin.bypass.disabled`
- `deploy-stage.yml:14-15` — identical.

Both keys you asked us to look for are present and both are flipped to `true`.

### Production — `deploy-backend.yml`

- **Trigger:** `on: push: branches: [main]` plus `workflow_dispatch` (`deploy-backend.yml:3-6`).
- **Job order:** `build` → `maintenance-on` → `deploy` → `notify` (`:97-98` gates `deploy` on `[build, maintenance-on]`).
- **KV write** (`maintenance-on` job, `:55-71`): two sequential `curl ... -X PUT`, one per key, to the Cloudflare KV API:

  ```bash
  curl -fsSL -X PUT \
    "https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/storage/kv/namespaces/${CF_KV_NAMESPACE_ID}/values/${MAINTENANCE_KEY}" \
    -H "Authorization: Bearer ${CF_API_TOKEN}" \
    -H "Content-Type: text/plain" \
    --data "true"
  # then the same shape for ${ADMIN_BYPASS_KEY}
  ```

  - Method: `PUT`. URL: `https://api.cloudflare.com/client/v4/accounts/{CF_ACCOUNT_ID}/storage/kv/namespaces/{CF_KV_NAMESPACE_ID}/values/{KEY}`.
  - Headers: `Authorization: Bearer {CF_API_TOKEN}`, `Content-Type: text/plain`. Body: literal `true`.
  - Secrets: `CF_API_TOKEN`, `CF_ACCOUNT_ID`, `CF_KV_NAMESPACE_ID` (`:57-59`).
- **Wait:** `sleep 60` for KV edge-POP propagation (`:73-76`).
- **Verify (edge, not origin):** polls `https://api.oglasino.com/health` up to **5 attempts**, expecting HTTP **503** (the Worker returns 503 when `maintenance.active=true`), `sleep 10` between attempts; on non-503 after 5 tries it logs a `::warning::` and **proceeds anyway** — verification is advisory, non-blocking (`:78-95`).
- **Deploy:** SSH to the droplet (`appleboy/ssh-action`), `sed` the `IMAGE_TAG` into `.env`, `docker compose pull backend`, `docker compose up -d --wait backend`, prune (`:97-122`).
- **Maintenance OFF is NOT in the workflow.** `notify` prints "Maintenance and admin bypass are STILL ON" and tells the operator to run `/opt/oglasino/scripts/maintenance-off.sh` on the droplet (`:124-137`). That script is not in this repo (lives on the droplet). `docs/12-deployment.md:117-142` documents the manual off-flip: the same `PUT` with `--data "false"` for `maintenance.active`.

### Stage — `deploy-stage.yml`

Same structure, with these differences:
- **Trigger:** `branches: [stage]` (`:5`).
- **Namespace:** the `env` var `CF_KV_NAMESPACE_ID` is sourced from `secrets.CF_KV_NAMESPACE_ID_STAGE` (`:60`). The curls reference `${CF_KV_NAMESPACE_ID}` (`:64,68`) — which resolves to the stage value. **This is correct**, not a bug (see "Cross-check corrections" below).
- **Verify:** single non-retrying advisory `curl` to `https://api-stage.oglasino.com/health` expecting `503` (`:80-87`).
- KV-off is manual (`:119-125`).

### CI — `ci-dev.yml`

Pure CI on `dev` (push) and `[dev, stage, main]` (PR): compile, Spotless, SpotBugs. **No Cloudflare/KV/maintenance writes whatsoever** (grep-confirmed clean).

### Note for the split

The deploy pipeline writes a single `maintenance.active` key (plus `admin.bypass.disabled`). If the feature introduces `maintenance.web.active` + `maintenance.backend.active`, **every site above that writes/reads `maintenance.active` is a touch point**: `deploy-backend.yml:14`, `deploy-stage.yml:14`, the droplet scripts (`maintenance-on.sh` / `maintenance-off.sh`, not in repo), `docs/12-deployment.md`, **and** the backend's own `DefaultCloudflareKvService` (see §5) which hardcodes `maintenance.active`.

---

## 2. Health-check endpoint `/api/public/health/check`

**Exists.** `HealthCheckController.java:8-16`:

```java
@RestController
@RequestMapping("/api/public/health/check")
public class HealthCheckController {
  @GetMapping
  public ResponseEntity<?> check() {
    return ResponseEntity.ok().build();
  }
}
```

- **Full path:** `/api/public/health/check`. No `server.servlet.context-path` is set in dev/stage/prod, so there is no prefix to add.
- **Returns:** HTTP `200 OK`, **empty body**. No custom headers.
- **Cost:** zero. No injected fields, no DB/Redis/ES/external calls, no `@Transactional`. It is a static `200`.
- **Failure behavior:** returns `200` **regardless** of backend health. If Postgres, Redis, and Elasticsearch are all down, this endpoint still returns `200`. It is a liveness signal ("JVM up, HTTP routing works"), **not** a readiness signal.
- **Reachability for an external worker probe (all confirmed):**
  - **Auth:** `permitAll()` — `/api/public/**` is whitelisted in `SecurityConfig.java:30,77`; `FirebaseAuthFilter.java:121-125` skips the 401 path for `/api/public`. No token needed.
  - **Language filter:** exempt — listed in `CurrentLanguageFilter.ALLOWLIST_EXACT` at `CurrentLanguageFilter.java:47`, so it never returns `400 LANG_MISSING_OR_INVALID` for a missing `X-Lang`.
  - **Rate limit:** `RateLimitFilter.categorize()` returns `null` for it → not rate-limited.
- **Tests:** none for this controller.

There is also a **second** static health endpoint: `HealthController.java:9-16` maps `GET /health` and returns `200 {"status":"ok"}`. `/health` is `permitAll()` via the `anyRequest().permitAll()` fallback (`SecurityConfig.java:81-82`) and is skipped by `CurrentLanguageFilter` (which only acts on paths starting with `/api`, `CurrentLanguageFilter.java:70`). The deploy workflows' edge-verify step probes `/health` (§1) — but only to read the **Worker's** 503, not backend health.

---

## 3. Liveness / readiness surfaces

`spring-boot-starter-actuator` is on the classpath (`pom.xml:53`).

### Configuration (prod, `application-prod.yaml:111-127`; stage identical except detail visibility)

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus   # only these — never "*" in prod
      base-path: /actuator
  endpoint:
    health:
      show-details: never                   # prod; stage uses "when-authorized"
      probes:
        enabled: true
  health:
    livenessstate: { enabled: true }
    readinessstate: { enabled: true }
    db: { enabled: true }
    redis: { enabled: true }
    elasticsearch: { enabled: true }
```

`application-dev.yaml` has **no** `management` block (Spring defaults). **There is no `management.endpoint.health.group.*` configuration in any profile** (grep-confirmed; the only `include:` lines are the `endpoints.web.exposure.include` above, which controls HTTP exposure, not group membership).

### What each probe actually checks

Because `probes.enabled: true` auto-creates the `liveness` and `readiness` groups but **no group membership is customized**, Spring's defaults apply:

- **`/actuator/health/liveness`** = `livenessState` only → "is the process alive / not in a broken internal state." Says nothing about dependencies.
- **`/actuator/health/readiness`** = `readinessState` only → reflects the `ReadinessState` (`ACCEPTING_TRAFFIC` vs `REFUSING_TRAFFIC`). **It does NOT include the db/redis/elasticsearch indicators.**
- **`/actuator/health`** (aggregate; exposed via `include: health`) = rolls up `livenessState`, `readinessState`, **db, redis, elasticsearch**. This is the **only** surface that reflects live dependency connectivity. (`show-details: never` in prod hides the per-indicator breakdown but the top-level `UP/DOWN` rollup still flips `DOWN` if any indicator is down.)

### Readiness state is gated at boot, then static

`CacheWarmupService.warmup()` (`CacheWarmupService.java:60-78`, `@EventListener(ApplicationReadyEvent.class)` `@Order(10)`) starts with readiness in `REFUSING_TRAFFIC` and, in a `finally`, publishes `AvailabilityChangeEvent ... ReadinessState.ACCEPTING_TRAFFIC` (`:76`) once cache warmup completes. So readiness flips to UP after boot+warmup. **Nothing flips it back to `REFUSING_TRAFFIC` at runtime** on a dependency outage — only application shutdown would. No custom `HealthIndicator` beans exist; all indicators are Spring auto-configured.

### Docker healthcheck

No `HEALTHCHECK` in the `Dockerfile` (it only installs `curl`/`tini`, exposes 8080, runs `java`). The healthcheck is defined in compose and **targets readiness**:

- `infra/docker-compose.yml:92-97` — `wget -qO- http://localhost:8080/actuator/health/readiness || exit 1`, `interval 30s / timeout 5s / retries 3 / start_period 120s`.
- `infra/docker-compose-stage.yml:108-117` — identical.

So the container is gated on readiness = "booted, warmed, accepting traffic," which is correct for `docker compose up --wait` ordering. The `start_period 120s` covers warmup.

### Which probe best answers "is backend able to serve real API traffic right now"

- **Liveness** — no (process-alive only).
- **Readiness** (as configured) — partially. It answers "finished boot/warmup and not shutting down," but because db/redis/es are **not** in the readiness group, a backend whose Postgres/Redis/ES died *after* boot would still report readiness UP. So readiness does **not** reliably answer "can serve real traffic right now."
- **Aggregate `/actuator/health`** — **yes**, this is the most accurate: it rolls up db/redis/elasticsearch. (Caveat: it is an actuator path, targeted by Docker on `localhost`; whether the edge Worker can/should reach `/actuator/**` from outside is a routing/exposure decision — `/actuator/**` is **not** in the `permitAll` `/api/public/**` allowlist, though `anyRequest().permitAll()` currently makes it reachable if routed.)

---

## 4. `/api/mobile/*` — absence confirmed

**No backend surface exists under `/api/mobile/`.** Verified by repo-wide grep over `src/main` and `src/test`:

- Zero hits for the literal `/api/mobile` anywhere (Java, resources, configs, tests).
- The only `"mobile"` token in Java source is a **comment** at `RateLimitFilter.java:145` (`// X-Device-Id (set by mobile clients) ...`). All other `mobile`/`snowmobile`/`mobile_development` hits are translation seeds, catalog JSON, and test product copy — none are routes.
- All controller mappings use exactly four prefixes: `/api/auth/**`, `/api/public/**`, `/api/secure/**`, `/internal/**`. No controller maps `/api/mobile`.
- Spring Security (`SecurityConfig.java:73-82`) defines matchers only for `/api/public/**`, `/api/auth/**`, `/internal/**`, `/api/secure/**`, and an `anyRequest().permitAll()` fallback — **no `/api/mobile` matcher**.
- No `WebMvcConfigurer` / interceptor / view-controller path mappings exist.

**Conclusion:** `/api/mobile/*` as a pure Worker-only label is safe — there is no backend collision today. (Standing risk: if a future controller ever maps `/api/mobile/...`, it would silently fall under `anyRequest().permitAll()`. Worth a guard/awareness, not a current issue.)

---

## 5. Backend code that flips Cloudflare KV — **brief belief REFUTED**

The brief states: *"We believe none exists; confirm."* **It exists.**

`DefaultCloudflareKvService.java` writes the `maintenance.active` KV key outward to the Cloudflare API:

```java
private static final String MAINTENANCE_KEY = "maintenance.active";                 // :15
private static final String URI_TEMPLATE =
    "https://api.cloudflare.com/client/v4/accounts/%s/storage/kv/namespaces/%s/values/%s"; // :16-17
private final RestTemplate restTemplate = new RestTemplate();                        // :19

public boolean toggleMaintenance() {                                                 // :38
  var current = getMaintenanceActive();
  var next = !current;
  configurationService.updateConfiguration(
      new ConfigurationDTO(MAINTENANCE_KEY, Boolean.toString(next)));                 // writes local DB config
  setMaintenanceCloudflareValue(next);                                               // :44 → writes CF KV
  return next;
}

private void setMaintenanceCloudflareValue(boolean value) {                          // :49-58
  // PUT https://api.cloudflare.com/.../namespaces/{config.namespace-id}/values/maintenance.active
  // Authorization: Bearer ${cloudflare.config.token}; Content-Type: text/plain; body = "true"/"false"
  restTemplate.exchange(buildUrl(), HttpMethod.PUT, request, String.class);          // :57
}
```

- **Config wiring** (`application-prod.yaml:129-143`, also stage/dev): a `cloudflare:` block beyond R2 — `cloudflare.api.token`, `cloudflare.worker.url`, and a **`cloudflare.config`** block with `token` (`CLOUDFLARE_KV_CONFIG_TOKEN`) and `namespace-id` (`CLOUDFLARE_NAMESPACE_ID`). `DefaultCloudflareKvService` reads `cloudflare.account.id`, `cloudflare.config.token`, `cloudflare.config.namespace-id` (`:21-28`).
- **Exposure** (`MaintenancePageController.java`, `@RequestMapping("/api/public/maintenance")`):
  - `GET /api/public/maintenance/active` (`:14-17`) — **reads** `maintenance.active` from the **local DB config** (`configurationService.getBooleanConfig`), *not* from Cloudflare. (Read source can drift from the KV value the Worker reads.)
  - `POST /api/public/maintenance/toggle` (`:19-24`) — calls `toggleMaintenance()`, which writes **both** the local DB config **and** Cloudflare KV.

### Precise answer to the literal question

- **Autonomous self-monitoring that flips KV on detecting its own unhealthiness:** **none.** No `@Scheduled` task, exception handler, or health indicator calls `CloudflareKvService`. (Verified across the scheduled jobs and `GlobalExceptionHandler`.)
- **Any code that calls the Cloudflare API / writes maintenance state outward:** **yes** — the request-triggered `toggleMaintenance` path above.

### Security concern (flagged for Mastermind)

`POST /api/public/maintenance/toggle` sits under `/api/public/**` = `permitAll()` (`SecurityConfig.java:77`) and has **no `@PreAuthorize`**. As written, it is an **unauthenticated** endpoint that can put the whole platform into maintenance (and write to Cloudflare KV) for anyone who can reach the origin. Whether the edge Worker blocks `/api/public/maintenance/toggle` is a `oglasino-router` concern outside this repo; the backend itself does not gate it. This also tensions with conventions Part 8 ("the Cloudflare router Worker is the edge boundary; maintenance state … lives there") — the backend currently holds a second writer of that same KV key. See "For Mastermind" in the session summary.

---

## 6. Seam — the right probe target

Two distinct questions, two answers:

- **"Is the origin process alive and routable?"** → **`/api/public/health/check`** (or `/health`). Healthy: `200` (empty body for `health/check`; `{"status":"ok"}` for `/health`). Unhealthy (process down / not routing): connection failure / no response. It is purpose-built for this: public, no token, language-exempt, not rate-limited, zero cost. **Limitation:** it returns `200` even when DB/Redis/ES are down, so it cannot detect an "up-but-can't-serve" backend.

- **"Can the backend serve *real* API traffic (dependencies healthy)?"** → as configured, **neither `health/check` nor `/actuator/health/readiness` answers this.** `health/check` ignores dependencies; readiness (default group = `readinessState` only) won't flip on a runtime DB/Redis/ES outage. The **aggregate `/actuator/health`** is the only surface that rolls up db/redis/elasticsearch — Healthy: `{"status":"UP"}`; Unhealthy (any dep down): top-level `{"status":"DOWN"}`/HTTP 503 (details hidden in prod via `show-details: never`).

**Recommendation (design input for Mastermind, not asserted current behavior):** if the worker-side backend liveness probe is meant to mean "backend can serve API traffic," the cleanest options are (a) probe the aggregate `/actuator/health` (ensure the edge can route `/actuator/**`), or (b) add `management.endpoint.health.group.readiness.include: readinessState,db,redis,elasticsearch` so `/actuator/health/readiness` gains real dependency-awareness, then probe readiness. If the probe only needs "origin is up," `/api/public/health/check` is already the correct, cheapest, token-free target.

---

## Cross-check corrections (adversarial verification of the parallel investigation)

Two subagent claims were checked against the source and **corrected**:

1. **"Stage workflow namespace bug" — DISMISSED.** A subagent claimed `deploy-stage.yml` defines `CF_KV_NAMESPACE_ID_STAGE` but curls `${CF_KV_NAMESPACE_ID}`, writing to prod KV. Reading `deploy-stage.yml:60`: the **env var** is named `CF_KV_NAMESPACE_ID` and its **value** is `${{ secrets.CF_KV_NAMESPACE_ID_STAGE }}`. The curl's `${CF_KV_NAMESPACE_ID}` resolves to the stage secret. **No bug.** (The agent confused the secret name with the env-var name.)

2. **"Readiness checks db/redis/es" — CORRECTED.** A subagent asserted `/actuator/health/readiness` includes the db/redis/elasticsearch indicators. It does not: there is no `management.endpoint.health.group.readiness.include`, so the readiness group is Spring's default (`readinessState` only). Those indicators live in the aggregate `/actuator/health`. This correction is load-bearing for §3 and §6.

---

## Files examined (primary)

- `.github/workflows/deploy-backend.yml`, `deploy-stage.yml`, `ci-dev.yml`
- `docs/12-deployment.md`
- `controller/HealthCheckController.java`, `health/HealthController.java`
- `controller/MaintenancePageController.java`
- `service/CloudflareKvService.java`, `service/impl/DefaultCloudflareKvService.java`
- `security/config/SecurityConfig.java`, `security/filter/FirebaseAuthFilter.java`, `security/filter/RateLimitFilter.java`
- `filter/CurrentLanguageFilter.java`
- `config/CacheWarmupService.java`
- `src/main/resources/application-{dev,stage,prod}.yaml`
- `infra/docker-compose.yml`, `infra/docker-compose-stage.yml`, `Dockerfile`
- repo-wide grep for `/api/mobile`, `mobile`, `cloudflare`, `maintenance.active`, KV/namespace references
