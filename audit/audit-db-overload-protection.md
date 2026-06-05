# Audit — DB Overload Protection (read-only, oglasino-backend)

**Repo:** oglasino-backend
**Branch:** `dev` (unchanged — no checkout)
**Type:** READ-ONLY current-state inventory. No code changed.
**Date:** 2026-06-03

## Scope note — the spec file does not exist yet

The brief references `oglasino-docs/features/db-overload-protection.md` as "the spec." **That file does not exist on disk.** `oglasino-docs/features/` has no `db-overload-protection.md` (closest neighbour: `connection-pool-hardening.md`). This is expected and not a blocker: per conventions Part 10, a Phase-2 audit precedes the Phase-4 canonical spec. The brief's 17 questions are self-contained and answered below from code. Every "the spec assumes…" statement is taken from the brief's wording, not from a file I read.

Each answer gives the **actual file path + actual value**. Where the brief's assumption and the code disagree, it's marked **⚠ DISCREPANCY**.

---

## A. Pool size and environment configuration (HIGHEST PRIORITY)

### 1. Real pool size, per environment

There is **no base `application.yaml`** — only three profile files. Pool size is a **literal in each profile file** (NOT an env var):

| Env | File | `maximum-pool-size` | `minimum-idle` |
|---|---|---|---|
| dev | `src/main/resources/application-dev.yaml:18` | **20** | 10 |
| stage | `src/main/resources/application-stage.yaml:18` | **8** | 2 |
| prod | `src/main/resources/application-prod.yaml:15` | **18** | 10 |

**⚠ DISCREPANCY with the brief's "spec assumes 18":** 18 is correct for **prod only**. **dev = 20, stage = 8.** Any threshold expressed as a fraction of a hard-coded 18 would be wrong on dev (too low) and stage (way too high — 8-connection pool). This is the strongest argument for deriving thresholds from the *live* pool size (Q2), not a literal.

prod comment (`application-prod.yaml:15`): `# tuned for DO Basic 1GB Postgres (25 conn limit)`. stage comment (`:14-16`): tight pool to leave headroom on a 2 GB droplet that also runs Dockerized Postgres/Redis/ES.

### 2. Can pool size be read at runtime from HikariCP itself?

**Yes, in-process — and this is the robust route.** No custom `DataSource` bean exists (`grep` for `HikariDataSource`/`HikariConfig`/`DataSource` bean overrides in `src/main/java` → none), so Spring Boot's default datasource is a **`HikariDataSource`** (confirmed by the fact that `spring.datasource.hikari.*` keys are honoured). That bean exposes `getMaximumPoolSize()` directly and `getHikariPoolMXBean()` (live `getActiveConnections()`, `getTotalConnections()`, `getIdleConnections()`, `getThreadsAwaitingConnection()`). Injecting the `DataSource` and casting is Actuator-independent.

**Micrometer gauge `hikaricp.connections.max` / `.active`:** registered **in-process** but **NOT reachable over HTTP** today. Detail under Q7 — short version: there is **no Micrometer registry dependency** in `pom.xml`, so Spring Boot auto-configures only a fallback `SimpleMeterRegistry`; HikariCP metrics bind to it and are readable via an injected `MeterRegistry`, but no `/actuator/prometheus` (no registry dep) and no `/actuator/metrics` (not in the exposure list) make them visible externally.

**Recommendation for the feature:** read live pool size from the `HikariDataSource` bean (or its MXBean), not a literal and not Actuator HTTP.

### 3. Configuration-table seeding, per environment

**Single shared seed set for all environments — no per-environment seed.**
- Files: `src/main/resources/data/configuration/data-configuration.sql` (the `configuration` key/value table) and `data-app-version-config.sql`.
- All three profiles load them identically via `spring.sql.init.data-locations: classpath:data/configuration/*.sql` (`application-dev.yaml:28`, `application-stage.yaml:34`, `application-prod.yaml:31`). `mode: always`.
- `find src/main/resources/data -iname "*dev*"/"*stage*"/"*prod*"` → **nothing**. There is no profile-specific SQL.
- Seed row shape: `configuration (id, key, value, description, created_at)`, idempotent via `ON CONFLICT (id) DO NOTHING` / `DO UPDATE`.

**Consequence for "default-OFF on dev, opt-in per environment":** there is **no existing per-environment Configuration-table mechanism to copy.** Every environment gets the identical `configuration` rows. To vary the throttle's enable flag by environment you must use one of the two precedents in Q4 — not a Configuration-table seed split (which doesn't exist).

### 4. Precedent for per-environment config differences

Per-environment variance today is achieved **entirely through env-var-overridden YAML per profile**, never through the `configuration` table. Named mechanisms + examples:

1. **Literal-differs-per-profile-file** (the value is hard-coded but each profile file holds a different one): `maximum-pool-size` 20/8/18; `server.tomcat.max-threads` (dev unset → 200 / stage 50 / prod 100); `minimum-idle` 10/2/10.
2. **`${ENV_VAR:default}` with different defaults per profile:** `app.images.sweeper.enabled` = `${IMAGE_SWEEPER_ENABLED:false}` on dev (`application-dev.yaml:223`) vs `:true` on stage/prod (`:246`/`:235`); `app.images.cdn-base-url` defaults `cdn-staging`/`cdn-stage`/`cdn` per file.
3. **Seed-neutral-then-operator-mutates** (Configuration-adjacent, the closest thing to a runtime toggle): `data-app-version-config.sql` seeds floor/ceiling `0.0.0` with `ON CONFLICT (id) DO NOTHING`, and operators raise it at runtime via `POST /internal/app/version/floor|ceiling`; `DO NOTHING` preserves the operator value across reboots.

So a "default-OFF dev, ON elsewhere" throttle flag fits mechanism **#2** (a YAML `${THROTTLE_ENABLED:false}` defaulted per profile) or **#3** (seed disabled, operator enables via an admin endpoint). It does **not** fit any existing Configuration-seed split, because none exists.

---

## B. The ten spec unknowns (§11)

### 5. Cloudflare KV client — EXISTS, writes both maintenance flags

`DefaultCloudflareKvService` exists and is a live backend KV writer (the expo-maintenance-split work, as issues.md hinted).

- Interface: `src/main/java/com/memento/tech/oglasino/service/CloudflareKvService.java` — two methods: `boolean toggleMaintenance()`, `boolean isMaintenanceActive()`.
- Impl: `src/main/java/com/memento/tech/oglasino/service/impl/DefaultCloudflareKvService.java`.
- **What it writes:** both `maintenance.web.active` AND `maintenance.backend.active` (constants at `:15-16`). `toggleMaintenance()` reads both keys (no short-circuit), computes `next = !(webActive || backendActive)`, then `setMaintenanceCloudflareValue(...)` PUTs **both** keys to `next` (`:43-44`). There is currently **no public method that writes only `maintenance.backend.active`.**
- **Token/env:** `cloudflare.account.id` ← env `CLOUDFLARE_ACC_ID`; `cloudflare.config.token` ← env `CLOUDFLARE_KV_CONFIG_TOKEN`; `cloudflare.config.namespace-id` ← env `CLOUDFLARE_NAMESPACE_ID` (mapping in all three `application-*.yaml` under `cloudflare.config.*` / `cloudflare.account.*`). Bearer-auth header (`:82-86`).
- **Can it write `maintenance.backend.active`?** **Yes — the plumbing already does.** The private `setMaintenanceCloudflareValue(key, value)` (`:69-76`) PUTs an arbitrary key via `URI_TEMPLATE` (`:20-21`, `/storage/kv/namespaces/{ns}/values/{key}`). An auto-trip writer for **only** `maintenance.backend.active` is a thin new method over the existing PUT; no new token, URL, or auth work.
- **Only caller:** `admin/controller/MaintenanceAdminController` (`:21` reads `isMaintenanceActive`, `:26` calls `toggleMaintenance`) — the admin wrench. Nothing else touches it.
- **Part 4b flag (low/medium):** the impl uses `private RestTemplate restTemplate = new RestTemplate();` (`:23`) — a **no-timeout** `RestTemplate`, not the 3 s-timeout one in `ApplicationConfig.restTemplate()`. A hung Cloudflare API call would block the calling thread indefinitely. Relevant because the auto-trip would call KV on a degraded path; an auto-trip should not be able to hang on the very API it depends on. Out of scope to fix here — flagged.

### 6. Filter chain ordering — NO filter has an explicit `@Order`

**Not one servlet filter declares `@Order`.** (`grep @Order` across `filter/`, `security/filter/`, `logging/` → empty.) The eight filters and how each is wired:

| Filter | File | Registration | Explicit order? |
|---|---|---|---|
| `RequestLoggingFilter` | `logging/RequestLoggingFilter.java` | `LoggingConfig` `FilterRegistrationBean`, `setOrder(Ordered.HIGHEST_PRECEDENCE)` | **Yes — HIGHEST_PRECEDENCE (`Integer.MIN_VALUE`)** |
| `InternalTokenFilter` | `security/filter/InternalTokenFilter.java` | `@Bean` in `ApplicationConfig:33`; added to Security chain `addFilterBefore(..., UsernamePasswordAuthenticationFilter)` (`SecurityConfig:83`) | No |
| `FirebaseAuthFilter` | `security/filter/FirebaseAuthFilter.java` | `@Bean` in `ApplicationConfig:22`; `addFilterBefore(..., UsernamePasswordAuthenticationFilter)` (`SecurityConfig:84`) | No |
| `RateLimitFilter` | `security/filter/RateLimitFilter.java` | `@Bean` in `RateLimitConfig:52`; `addFilterAfter(rateLimitFilter, FirebaseAuthFilter.class)` (`SecurityConfig:85`) | No |
| `BaseSiteFilter` | `filter/BaseSiteFilter.java` | `@Component` (auto-registered servlet filter) | No |
| `CurrentLanguageFilter` | `filter/CurrentLanguageFilter.java` | `@Component` | No |
| `BotFilter` | `security/filter/BotFilter.java` | `@Component` | No |
| `InputSanitizationFilter` | `security/filter/InputSanitizationFilter.java` | `@Component` + `@WebFilter("/*")` | No |

**Resolved execution order (as far as determinable):**
1. `RequestLoggingFilter` first (HIGHEST_PRECEDENCE).
2. The **Spring Security `FilterChainProxy`** at the default order `-100`, and **inside it** the explicit relative order from `SecurityConfig`: `InternalTokenFilter` → `FirebaseAuthFilter` → `RateLimitFilter` (the only deterministically-ordered trio in the app).
3. The four `@Component` filters (`BaseSiteFilter`, `CurrentLanguageFilter`, `BotFilter`, `InputSanitizationFilter`), auto-registered at the default order `LOWEST_PRECEDENCE`. **Their order relative to each other is undefined** (no `@Order`; Spring Boot falls back to bean-name ordering, which is not a contract).
4. `DispatcherServlet`.

**⚠ Two gotchas the placement brief must account for:**
- **`FirebaseAuthFilter` and `InternalTokenFilter` are double-registered.** They are plain `@Bean` `OncePerRequestFilter`s with **no `FilterRegistrationBean` disabling servlet auto-registration** (contrast `RateLimitConfig:61-67`, which *does* disable it for `RateLimitFilter` with `reg.setEnabled(false)`). So Spring Boot also auto-registers them as standalone servlet filters at `LOWEST_PRECEDENCE`, *in addition to* their place in the Security chain. `OncePerRequestFilter`'s dedupe guard means the body runs once per request, but the wiring is duplicated. **Adjacent observation (medium)** — pre-existing, out of scope.
- **`@WebFilter("/*")` on `InputSanitizationFilter` is inert** — there is no `@ServletComponentScan` (`OglasinoApplication` is `@SpringBootApplication` only). The filter registers solely because it is `@Component`.

**Placement implication for `RequestThrottleFilter` ("after auth + rate-limit, before DispatcherServlet"):** the clean, deterministic way to get exactly that is to add it to the Security chain with `http.addFilterAfter(requestThrottleFilter, RateLimitFilter.class)` in `SecurityConfig` (mirroring how `RateLimitFilter` itself is placed) — **not** as a bare `@Component`, which would land it in the undefined `LOWEST_PRECEDENCE` bucket *after* the security chain and with no guaranteed order vs the four `@Component` filters.

### 7. HikariCP metrics exposure — Actuator yes, Micrometer registry NO

- **Actuator present:** `spring-boot-starter-actuator` (`pom.xml:53`).
- **No Micrometer registry dependency.** Full `pom.xml` artifactId sweep contains **no** `micrometer-registry-prometheus` (or any `micrometer-registry-*`). The only `io.micrometer` references in `src/main/java` are two `import io.micrometer.common.util.StringUtils;` (in `DefaultUserFacade.java:26`, `TextQueryGenerator.java:9`) — a string utility, **not** metrics.
- **Actuator exposure (`management.endpoints.web.exposure.include`):**
  - **dev:** **no `management:` block at all** (`grep management application-dev.yaml` → none). Default exposure = `health` only; no `prometheus`, no `metrics`.
  - **stage** (`application-stage.yaml:117-143`) and **prod** (`application-prod.yaml:111-133`): `include: health, info, prometheus`. Plus `health.show-details` (`when-authorized` stage / `never` prod), probes enabled, and a dependency-aware `readiness` group = `readinessState,db,redis,elasticsearch`.
- **Net:** `hikaricp.connections.active`/`.max` are **collected in-process** (Spring Boot auto-configures a fallback `SimpleMeterRegistry` under Actuator when no registry impl is present, and `DataSourcePoolMetricsAutoConfiguration` binds Hikari to it), but they are **not exposed over HTTP**: the `prometheus` endpoint listed in `include` won't exist without `micrometer-registry-prometheus` on the classpath (no-op include), and the `metrics` endpoint is not in the include list. → For the throttle, read the pool from the `HikariDataSource` bean directly (Q2), not via an HTTP metrics scrape.

### 8. EmailService interface — `sendPlainText` confirmed

`src/main/java/com/memento/tech/oglasino/service/EmailService.java`, **two methods**:
- `void sendPlainText(String to, String subject, String body)` — **exists, exact signature the brief expects.** Envelope (From/From-name/Reply-To) from `EmailProperties`, never client-supplied; throws `org.springframework.mail.MailException` on SMTP failure.
- `void sendHtml(String to, String subject, String htmlBody, String textFallback)`.

`sendPlainText` is ready to use for alert emails as-is.

### 9. `incident_log` table — name is FREE

`grep -i incident src/main/resources/db/migration/V1__init_schema.sql` → **no match** (no `incident_log` table, no `incident` column). The 42 existing tables (sample: `users`, `product`, `configuration`, `banned_user_audit`, `product_audit`, `user_admin_action_audit`, `user_deletion_audit_log`, `messaging_cleanup_locks`, `pending_chat_cleanup`, `push_token`, `report`, `review`, …) contain no `incident*` collision. **`incident_log` is available.** Note Part 12: pre-production, new tables edit `V1__init_schema.sql` in place (no `V2__`).

### 10. Tomcat thread pool (`server.tomcat.threads.max`)

The project uses the **`server.tomcat.max-threads`** spelling (Boot still binds it), not `threads.max`:
- **dev:** **unset** → default **200**. (`application-dev.yaml` has a `server.tomcat` block at `:165` but only `accesslog`; no `max-threads`.)
- **stage:** `max-threads: 50`, `accept-count: 25`, `connection-timeout: 20000` (`application-stage.yaml:109-111`).
- **prod:** `max-threads: 100`, `accept-count: 50`, `connection-timeout: 20000` (`application-prod.yaml:103-105`).

Relevant interaction: prod has **100 Tomcat threads against an 18-connection pool** — at saturation, up to ~82 request threads can be parked waiting on a DB connection. That ratio is exactly what the throttle is meant to protect.

### 11. HikariCP timing config, per environment

| Key | dev (`-dev.yaml:16-21`) | stage (`-stage.yaml:13-23`) | prod (`-prod.yaml:13-20`) |
|---|---|---|---|
| `maximum-pool-size` | 20 | 8 | 18 |
| `minimum-idle` | 10 | 2 | 10 |
| `connection-timeout` | **unset → default 30000** | 5000 | 5000 |
| `idle-timeout` | 600000 | 300000 | 300000 |
| `max-lifetime` | 1800000 | 1200000 | 1200000 |
| `leak-detection-threshold` | **unset → 0 (disabled)** | **unset → 0 (disabled)** | **unset → 0 (disabled)** |
| `initialization-fail-timeout` | unset (default 1) | 10000 | 10000 |
| `auto-commit` | true | true | true |

Notes for the throttle: **`leak-detection-threshold` is disabled in every environment** — no existing leak instrumentation to lean on. **`connection-timeout` on dev is the 30 s default** (5 s on stage/prod), so a saturated dev pool makes callers wait up to 30 s before `SQLTransientConnectionException` — long enough that a throttle reacting to `getThreadsAwaitingConnection()` would fire well before the timeout.

### 12. `pg_stat_statements` access

**No code anywhere touches it.** `grep -ri pg_stat src/main src/test` → nothing. No native query, no `@Query(nativeQuery=true)` against `pg_stat_*`, no JdbcTemplate usage. The DB user is whatever `${DATASOURCE_USER}` resolves to (env-supplied; not in the repo). **Cannot confirm from code** whether the DO managed-Postgres app user has `pg_stat_statements` read access or whether the extension is enabled DO-side — that is a DO control-panel/database setting outside this repo. Reportable fact: today the app reads zero query-stats; any `pg_stat_statements`-based feature would be greenfield and gated on a DO-side grant we cannot verify here.

---

## C. Design-correction confirmations

### 13. `@Scheduled` mechanism — resolves from YAML (Spring Environment), NOT the Configuration table

Confirmed. `@Scheduled` cron/interval values come from the **Spring Environment (`application*.yaml`)**, resolved at bean construction — they **cannot** read the `configuration` table.

Cited precedent (the user-deletion crons, per decisions.md 2026-05-19): `jobs/UserDeletionScheduledJobs.java`, with an explicit class-doc statement of exactly this constraint (`:23-29`): *"Cron expressions are resolved from the Spring Environment at bean construction (the `application*.yaml` cron keys), not from the `configuration` table — Spring's `@Scheduled` only reads from the property source. The other knobs (batch size, retention, enabled flag) are read from the configuration table at each tick."*
- `:51` `@Scheduled(cron = "${user.deletion.hard.delete.cron:0 0 2 * * *}")`, `:117` reconciliation, `:171` audit purge — all property placeholders with literal defaults; the runtime knobs (`getRequiredIntConfig("user.deletion.hard.delete.batch.size")`, `:54`) come from the Configuration table at each tick.

Other `@Scheduled` sites confirm the same split: `messaging/.../DefaultMessagingCleanupService.java:82` (`${messaging.cleanup.cron}`), `images/job/ChatImagesRemovalJob.java:41` (`${app.images.chat.removal}`), `images/job/ProductImagesRemovalJob.java:57` (`${app.images.sweeper.cron}`); a few use literal crons (`jobs/ProductRemovalJob.java:21` `"0 0 2 ? * SUN"`, `jobs/ProductBaseCurrencyUpdater.java:39`).

**→ The throttle's poll interval MUST live in YAML** (env-resolved), exactly as the brief states. Runtime-tunable thresholds may live in the Configuration table and be re-read each tick (the user-deletion pattern).

### 14. `SystemErrorCode` enum (post-split) — `SERVICE_DEGRADED` fits cleanly

- Location: `src/main/java/com/memento/tech/oglasino/exception/SystemErrorCode.java`.
- Shape: `enum SystemErrorCode implements ErrorCode`; each constant carries **`(String translationKey, HttpStatus httpStatus)`** and exposes `name()` (the wire code), `getTranslationKey()`, `getHttpStatus()`.
- Current constants: `RATE_LIMITED("system.rate_limited", TOO_MANY_REQUESTS)`, `NOT_AUTHENTICATED(..FORBIDDEN)`, `ACCESS_DENIED(..FORBIDDEN)`, `INTERNAL_ERROR(..INTERNAL_SERVER_ERROR)`, `LANG_MISSING_OR_INVALID(..BAD_REQUEST)`, `BASE_SITE_MISSING_OR_INVALID(..BAD_REQUEST)`, `NOT_OWNER(..FORBIDDEN)`.
- **`ErrorCode` interface** (`src/main/java/com/memento/tech/oglasino/exception/ErrorCode.java`): `String name()`, `String getTranslationKey()`, `HttpStatus getHttpStatus()`. Implemented by `ProductErrorCode`, `SystemErrorCode`, `UserErrorCode`, `ReportErrorCode`.
- **Fit:** a new `SERVICE_DEGRADED("system.service_degraded", HttpStatus.SERVICE_UNAVAILABLE)` is a one-line addition matching the existing pattern exactly. `HttpStatus.SERVICE_UNAVAILABLE` = 503. ✓

### 15. 503 wire shape — and a discrepancy in the proposed body

**How `RateLimitFilter` writes its body directly** (the mechanism to mirror), `security/filter/RateLimitFilter.java:30-35` + `:72-77`:
```java
response.setStatus(429);
response.setHeader("Retry-After", String.valueOf(retryAfterSec));
response.setContentType("application/json");
response.setCharacterEncoding("UTF-8");
response.getWriter().write(RATE_LIMITED_BODY);
```
where `RATE_LIMITED_BODY` =
```json
{"errors":[{"field":null,"code":"RATE_LIMITED","translationKey":"system.rate_limited"}]}
```
So a throttle filter writing 503 + `Retry-After` + this body shape directly is **fully consistent** with the existing filter (same status-then-header-then-write pattern, set before the body is written; the throttle runs before `DispatcherServlet`, so no handler/`@ControllerAdvice` is involved — writing directly is correct).

**⚠ DISCREPANCY (low) — the brief's proposed body omits `translationKey`.** The brief proposes `{ "errors": [{ "field": null, "code": "SERVICE_DEGRADED" }] }`. The as-built filter body **also carries `translationKey`** (`...,"translationKey":"system.rate_limited"`). To mirror `RateLimitFilter` (and so the web/mobile clients can render a localized string the way they already do for 429), the throttle body should be `{"errors":[{"field":null,"code":"SERVICE_DEGRADED","translationKey":"system.service_degraded"}]}`. Note this also means the canonical Part 7 envelope (`{field, code}` only) is, in practice, *extended* by the filter layer with `translationKey` — worth Mastermind reconciling the spec's stated shape against the as-built filter convention.

### 16. Translation seed shape — ERRORS namespace, next IDs reserved

System-level error codes are seeded under the **`ERRORS`** namespace (so `system.service_degraded` belongs there, alongside `system.rate_limited`). One `INSERT` per file into `oglasino_translation (id, language_id, translation_namespace, translation_key, translation_value, created_at) VALUES (...)`, closed with an `ON CONFLICT ... DO UPDATE`. Recent precedent — `system.rate_limited` is seeded identically across all four locale files (the maintenance/rate-limit work):
- EN `0001-data-web-translations-EN.sql:662` → `(3089, 3, 'ERRORS', 'system.rate_limited', 'You''re making requests too quickly...', CURRENT_TIMESTAMP)`
- RS `...-RS.sql` → `(5189, 1, 'ERRORS', 'system.rate_limited', ...)`
- RU `...-RU.sql` → `(7289, 4, 'ERRORS', 'system.rate_limited', ...)`
- CNR `...-CNR.sql` → `(989, 2, 'ERRORS', 'system.rate_limited', ...)`

Each locale uses a **disjoint ID block** keyed to its `language_id`, and the `ERRORS` group ends with a marker (`-- ERRORS END` / `--increaseby(20)`) followed by a **reserved gap** before the next namespace (`EXTRA_PRODUCTS`). **Next-available ID to append at the end of the ERRORS group, per locale (collision-free — the gap is ~38 IDs wide):**

| Locale | File | `language_id` | Last ERRORS row id | Next namespace starts | **Next-available ERRORS id** |
|---|---|---|---|---|---|
| EN | `0001-data-web-translations-EN.sql` | 3 | 3161 (`product.activation.banned`) | 3200 (`EXTRA_PRODUCTS`) | **3162** |
| RS | `0001-data-web-translations-RS.sql` | 1 | 5261 | 5300 | **5262** |
| RU | `0001-data-web-translations-RU.sql` | 4 | 7361 | 7400 | **7362** |
| CNR | `0001-data-web-translations-CNR.sql` | 2 | 1061 | 1100 | **1062** |

So one `system.service_degraded` row per file, inserted just after the current last ERRORS row (before the `-- ERRORS END` marker), using IDs **3162 / 5262 / 7362 / 1062** — well within the reserved gap, no collision (satisfies conventions Part 6 Rule 3). Note CNR = Montenegrin (aliases SR per Part 9). This matches the memory note that translation seeds are inline-appended into the existing `0001` files.

---

## D. Trust boundary check

### 17. Client-supplied values reaching the throttle/trip decision — NONE (by construction, if built as designed)

Confirmed against the as-built inputs the feature would consume:
- **Pool pressure:** `HikariDataSource` MXBean (`getActiveConnections`, `getThreadsAwaitingConnection`, `getMaximumPoolSize`) — server-internal, no request influence (Q2).
- **Config/thresholds:** Configuration table (server DB) + YAML/env (Q3, Q13) — server-internal.
- **Maintenance trip target:** Cloudflare KV via `DefaultCloudflareKvService` (Q5) — server-internal, server-credentialed.
- **Identity (if the throttle ever exempts admins or keys by caller):** would read `OglasinoAuthentication` from `SecurityContextHolder`, populated by `FirebaseAuthFilter` — the Part 11 trust source, not raw client claims.

`RateLimitFilter` is the cautionary contrast and the one place to watch: its **bucket key** reads client-controlled headers — `X-Device-Id` and `CF-Connecting-IP` (`RateLimitFilter.java:148-159`). That is acceptable there only because spoofing resets *the attacker's own* bucket (noted in-code, `:145-147`). **A `RequestThrottleFilter` must NOT take the same liberty for its trip/throttle decision:** the global degrade/trip is a system-state decision and must be driven solely by HikariCP metrics + Configuration + env. As long as the filter reads **no request header/body/param** for the throttle decision, Part 11 is satisfied. There is no existing code that would force client input into such a decision — the boundary is clean to begin with; the spec just must not introduce one.

---

## Summary of discrepancies (spec/brief assumption vs code)

1. **Pool size 18 is prod-only** — dev = 20, stage = 8 (Q1). Strongest reason to derive thresholds from the live `HikariDataSource`, not a literal.
2. **No HTTP-exposed HikariCP metrics** — no `micrometer-registry-prometheus`; `prometheus` is a no-op include; `metrics` endpoint not exposed; dev has no `management` block at all. Read the pool in-process from the `HikariDataSource` bean (Q2, Q7).
3. **No per-environment Configuration-table seed exists** — single shared `data/configuration/*.sql` for all profiles. "Default-OFF dev / opt-in prod" must use a per-profile YAML env-var flag or the seed-neutral-then-operator-mutate pattern, not a seed split (Q3, Q4).
4. **Proposed 503 body omits `translationKey`** — the as-built `RateLimitFilter` body includes it; mirror that (Q15).
5. **No filter has `@Order`; `FirebaseAuthFilter`/`InternalTokenFilter` are double-registered; `@WebFilter` is inert** — add the throttle to the Security chain via `addFilterAfter(..., RateLimitFilter.class)` for deterministic placement, don't rely on `@Component` ordering (Q6).
6. **`leak-detection-threshold` disabled in all envs; dev `connection-timeout` is the 30 s default** (Q11) — no existing leak instrumentation; throttle should key off `getThreadsAwaitingConnection()`.

## Cannot be answered from code (reported as such)

- **Q12 — DO-side `pg_stat_statements` grant/extension state.** No code reads it; whether the DO managed-Postgres app user can is a DB/control-panel setting outside this repo.
- **Q1/Q11 — resolved env-var values** (`DATASOURCE_*`, `ALLOWED_CORS`, etc.) come from `.env`/docker-compose/host env; only `.env`'s `DATASOURCE_DRIVER=org.postgresql.Driver` is in-repo. Pool *literals*, however, are in the YAML and are reported above.
