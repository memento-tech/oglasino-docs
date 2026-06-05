# DB Overload Protection

**Status:** planned
**Last updated:** 2026-06-03
**Phase 2 audit:** complete — `oglasino-backend/.agent/audit-db-overload-protection.md` (to be archived to `sessions/`)

Graduated response to database load pressure: slow requests early, shed requests at threshold, auto-flip the maintenance gate at critical saturation. Operator (Igor, solo) is alerted via email + Telegram, with full forensic logging to `incident_log` for post-incident root-cause.

This is structural defense against sustained-load DB exhaustion. The 2026-05-14 connection-pool incident (decisions.md) addressed cold-cache thundering herd; the open `connection-pool-hardening` backlog covers per-key single-flight dedup. This feature is independent of both — it protects against any cause of pool saturation, regardless of mechanism.

The feature is scoped to `oglasino-backend`. No web, mobile, router, or Firestore Rules changes. Cloudflare KV writes are server-to-server via the existing `DefaultCloudflareKvService`.

---

## 1. Goals

1. **Automatic maintenance flip when DB is genuinely overloaded.** Backend self-asserts Cloudflare maintenance (both `maintenance.web.active` and `maintenance.backend.active` → `"true"`) at CRITICAL pressure, sustained 30s. Untrip is manual via the existing three-step restore runbook (decisions.md 2026-05-15).
2. **Graduated response below CRITICAL.** YELLOW = sleep per request. RED = HTTP 503 + `Retry-After` shedding. Each level reduces arrival rate, giving the DB headroom to drain.
3. **Operator alerting.** Email on RED. Email + Telegram on CRITICAL. No alert on YELLOW (too frequent at this threshold).
4. **Full forensic logging.** At CRITICAL trip, capture HikariCP pool snapshot, `pg_stat_statements` slow-query top-N (if available), request context. Persisted to `incident_log` plus application logs.
5. **Two-layer enable control.** A per-environment YAML master switch (off on dev, on in stage/prod) and a live Configuration-table runtime kill-switch (flip on the spot, no redeploy). The throttle and auto-trip each run only when both layers permit.

---

## 2. Non-goals

- **Automatic un-trip.** Site stays in maintenance until operator manually un-trips via the three-step restore. Avoids flap mode where overload returns the moment traffic resumes.
- **Tiered request classification.** All non-excluded endpoints get the same treatment at each pressure level. Revisit if launch traffic patterns suggest otherwise.
- **Web, mobile, router, Firestore Rules changes.** Pure backend feature.
- **External metrics/observability (Prometheus/Grafana).** The throttle reads pool pressure in-process; it does not expose metrics over HTTP. Operator trend-visibility dashboards are a separate future chat (see § 12).
- **Bigger DO Postgres tier.** Out of budget for now. The feature buys time on the connection cap; it doesn't replace scaling. Logged as Risk Watch in state.md.
- **Manual un-trip automation.** The three-step restore runbook (decisions.md 2026-05-15) remains the un-trip path. Not redesigned here.

---

## 3. Architecture

### 3.1 Pressure signal

The signal source is read **in-process from the `HikariDataSource` bean**, not over HTTP and not via Micrometer/Actuator. The Phase 2 audit confirmed there is no `micrometer-registry-prometheus` on the classpath, so the `prometheus` endpoint in the stage/prod exposure list is a no-op, the `metrics` endpoint is not exposed, and dev has no `management` block at all. HikariCP metrics are collected in-process via a fallback `SimpleMeterRegistry` but are not reachable over HTTP. Routing the throttle's own pressure read through an HTTP scrape of its own JVM would be strictly worse than a direct bean read.

`DatabaseHealthMonitor` injects the `DataSource`, casts to `HikariDataSource`, and reads the pool via `getHikariPoolMXBean()`:

- `getActiveConnections()` — primary gradient signal (YELLOW/RED).
- `getThreadsAwaitingConnection()` — CRITICAL discriminator (see § 3.2).
- `getTotalConnections()`, `getIdleConnections()` — forensic snapshot.
- `getMaximumPoolSize()` (via `HikariConfigMXBean`) — the **live** pool size, used to resolve fractional thresholds to absolute counts.

**Why the live pool size, not a literal:** the audit found `maximum-pool-size` differs by environment — dev 20, stage 8, prod 18. Any threshold expressed against a hard-coded 18 would be wrong on dev (too low) and broken on stage (a `critical ≥ 17` count can never fire on an 8-connection pool — the feature would be silently dead exactly where it is tested). Reading the size from the running pool means thresholds re-derive automatically when the pool changes (droplet upgrade, managed-DB move, per-env difference) with no config edit.

**Why active connections + threads-awaiting, not pool-utilization percentage or acquisition wait time:** absolute active count gives a clean gradient for graduated response. Threads-awaiting is the leading indicator of genuine starvation — active-count alone saturates at the pool max and cannot distinguish "busy" (active at max, zero waiting) from "starving" (active at max, threads queuing). Acquisition wait time is a lagging indicator. `leak-detection-threshold` is disabled in all environments (audit Q11), so there is no existing leak instrumentation to lean on; threads-awaiting is the available starvation signal.

Polling: `DatabaseHealthMonitor` polls every `monitor.poll.interval.seconds` (default 2) via a `@Scheduled` task. The interval lives in `application.yaml`, **not** the Configuration table — `@Scheduled` resolves its interval from the Spring Environment at bean construction and cannot read the runtime Configuration cache (audit Q13; same constraint the user-deletion crons document, decisions.md 2026-05-19). Each poll computes the current pressure level against the rolling history.

### 3.2 Pressure levels

Thresholds are stored as **fractions of the live pool size** and resolved per-poll to absolute counts: `triggerCount = ceil(ratio × livePoolSize)`.

| Level        | Trigger condition                                                                                            | Action                                                                                                                                   |
| ------------ | ------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| **GREEN**    | active < yellow count                                                                                        | Nothing                                                                                                                                  |
| **YELLOW**   | active ≥ `ceil(yellowRatio × poolSize)` sustained `yellow.window.seconds`                                    | `RequestThrottleFilter` sleeps `throttle.yellow.sleep.ms` (default 50) before passing the request through                                |
| **RED**      | active ≥ `ceil(redRatio × poolSize)` sustained `red.window.seconds`                                          | `RequestThrottleFilter` returns HTTP 503 + `Retry-After: 2` instead of forwarding                                                        |
| **CRITICAL** | active ≥ `ceil(criticalRatio × poolSize)` **AND** `threadsAwaiting > 0`, sustained `critical.window.seconds` | `MaintenanceAutoTripService` asserts Cloudflare maintenance ON. Filter continues shedding until Cloudflare propagation (≤30s) takes over |

Default ratios (reproduce the original 13/14/17 intent against prod's pool of 18):

| Ratio key                  | Default | prod (18) | dev (20) | stage (8) |
| -------------------------- | ------- | --------- | -------- | --------- |
| `threshold.yellow.ratio`   | `0.72`  | 13        | 15       | 6         |
| `threshold.red.ratio`      | `0.80`  | 15        | 16       | 7         |
| `threshold.critical.ratio` | `0.94`  | 17        | 19       | 8         |

**Threshold semantics:** floor-inclusive (active ≥ N triggers the level). On stage's pool of 8 the defaults resolve to **distinct adjacent counts** — `ceil(0.72×8)=6`, `ceil(0.80×8)=7`, `ceil(0.94×8)=8` (6 / 7 / 8, as the table above shows). A genuine collapse (two levels resolving to the same integer) only occurs on a much smaller pool — e.g. a pool of 2, where all three resolve to 2. The monitor resolves ties **higher-level-wins**: if `red count ≤ yellow count`, a request at that count is treated as RED. This keeps the feature correct on a collapsed small pool rather than letting a rounding collapse disable a level.

**CRITICAL is the only level gated on `threadsAwaiting > 0`** in addition to the count. Rationale: active-at-max under healthy load is normal and should not auto-trip; active-at-max _with threads queuing for a connection_ is genuine exhaustion. The destructive action (maintenance flip) fires only on real starvation. YELLOW and RED key on active-count alone because below saturation `threadsAwaiting` is usually 0 and gives no gradient.

All ratios and window durations live in the `Configuration` table and are re-read each poll. Operator tunes without redeploy.

### 3.3 Trip policy

**Auto-trip, manual untrip.** CRITICAL fires once; the site stays in maintenance until the operator runs the three-step restore manually (decisions.md 2026-05-15). After auto-trip, a lockout (`auto.maintenance.locked_until` in Configuration) prevents re-trip for 5 minutes after the next manual untrip, preventing trip-untrip-trip cascades.

**The trip asserts maintenance ON — it does not blind-toggle.** The existing `DefaultCloudflareKvService.toggleMaintenance()` computes `next = !(webActive || backendActive)` — a toggle. Used raw, it would turn maintenance **off** if maintenance happened to be on when CRITICAL fired (mid-deploy, or a manual flip). The trip must _assert_ maintenance-on, never flip-whatever's-there. `MaintenanceAutoTripService` therefore:

1. Reads current maintenance state (`isMaintenanceActive()`).
2. If already on, no-ops the KV write (logs, records the incident, does not re-write).
3. If off, writes **both** `maintenance.web.active` and `maintenance.backend.active` → `"true"`.

Writing both flags is deliberate (operator decision, 2026-06-03): when the backend is overloaded the web tier is unusable anyway, so asserting both is the honest state and reuses the existing both-flags PUT path.

**Both control layers must permit the trip** (see § 3.8):

- YAML master `auto.maintenance.enabled` resolves `true` for this environment, AND
- Configuration runtime switch `throttle.runtime.enabled` is `true`.

**Timeout-bounded KV client (required, on the critical path).** The existing `DefaultCloudflareKvService` uses `new RestTemplate()` with no connect/read timeout. The auto-trip calls Cloudflare KV precisely when the system is degraded; a hung KV call with no read timeout would block the scheduled monitor thread indefinitely — the trip mechanism hanging on the very API it depends on. The auto-trip's KV write path **must** use a timeout-bounded client (mirror the 3s-timeout `RestTemplate` in `ApplicationConfig`, or a dedicated short-timeout client). The KV-write call is also off the monitor's poll thread (see § 4.1) so a slow trip cannot stall polling. Fixing the existing `toggleMaintenance()` no-timeout `RestTemplate` is logged separately as an issue (it serves the admin wrench, a different path); the auto-trip path must not inherit it.

**Cloudflare credentials already exist.** `DefaultCloudflareKvService` reads `CLOUDFLARE_ACC_ID`, `CLOUDFLARE_KV_CONFIG_TOKEN`, `CLOUDFLARE_NAMESPACE_ID` from env and Bearer-auths. No new token, URL, or auth work — the auto-trip reuses this service.

### 3.4 Filter placement

`RequestThrottleFilter` is registered into the Spring Security filter chain via `SecurityConfig`:

```java
http.addFilterAfter(requestThrottleFilter, RateLimitFilter.class)
```

This mirrors exactly how `RateLimitFilter` itself is placed (`addFilterAfter(rateLimitFilter, FirebaseAuthFilter.class)`) and gives the intended position deterministically: after `FirebaseAuthFilter` (auth runs first), after `RateLimitFilter` (per-client limit is upstream of the global throttle), before `DispatcherServlet`.

**Why the Security chain, not a bare `@Component`:** the audit found no servlet filter in the app declares `@Order`. The only deterministic ordering is the Security chain trio (`InternalTokenFilter → FirebaseAuthFilter → RateLimitFilter`). The four `@Component` filters sit in an undefined `LOWEST_PRECEDENCE` bucket with no guaranteed order among themselves. Registering the throttle as a `@Component` would land it in that undefined bucket _after_ the security chain. The Security-chain registration sidesteps the pre-existing `@Order`-less arrangement rather than depending on it.

**Filter exclusion list** — these paths bypass throttling entirely:

- `/actuator/health/*` (health probes — Better Stack and DO — must succeed)
- `/api/public/maintenance/*` (the maintenance page itself must render)
- `/api/secure/admin/**` (admin must operate during overload, same posture as the admin bypass on the Cloudflare maintenance gate)

### 3.5 Filter behavior

Per request:

1. Read the YAML master `throttle.enabled` (resolved at boot). If `false`, pass through (no-op for this environment).
2. Read the Configuration runtime switch `throttle.runtime.enabled` (cached, re-read live). If `false`, pass through.
3. Check exclusion list. If matched, pass through.
4. Read current pressure level from `DatabaseHealthMonitor` (in-memory volatile state, no DB hit).
5. GREEN: pass through.
6. YELLOW: `Thread.sleep(yellowSleepMs)`, then pass through.
7. RED or CRITICAL: write 503 response with `Retry-After: 2`, log at INFO. Do not pass through.

The sleep at YELLOW holds the Tomcat request thread for the sleep duration. Audit-confirmed thread budgets: dev 200, stage 50, prod 100, against pools of 20/8/18. With 50ms sleep and prod's 100 threads, this scales well past anything this tier will see. Sleep duration is Configuration-tunable; if it ever needs to grow past ~200ms, switch that level to shed instead of sleep.

The shed response body mirrors the as-built `RateLimitFilter` 429 body — status set, then `Retry-After`, then `Content-Type`/encoding, then the JSON written directly (the throttle runs before `DispatcherServlet`, so no `@ControllerAdvice` is involved; writing directly is correct):

```json
{
  "errors": [
    {
      "field": null,
      "code": "SERVICE_DEGRADED",
      "translationKey": "system.service_degraded"
    }
  ]
}
```

**Note on the envelope:** conventions Part 7's canonical envelope is `{field, code}` only, but the filter layer in practice extends it with `translationKey` — this is pre-existing (`RateLimitFilter`'s 429 body already does it) so web/mobile clients can render a localized string the way they already do for 429. This feature matches the as-built filter convention rather than the written Part 7 shape. Not a Part 7 amendment; a one-line acknowledgment of the existing filter-layer extension.

`SERVICE_DEGRADED` is added to `SystemErrorCode` (the post-`system-error-code-split` home, decisions.md 2026-05-29) as `SERVICE_DEGRADED("system.service_degraded", HttpStatus.SERVICE_UNAVAILABLE)` — a one-line fit to the existing `(translationKey, httpStatus)` constant shape. Translation key `system.service_degraded` seeded in 4 locales (`ERRORS` namespace).

**Trust boundary check.** The throttle decision is server-derived from HikariCP metrics + Configuration + env. No client-supplied values are consulted. Unlike `RateLimitFilter` (whose bucket key reads client-controlled `X-Device-Id` / `CF-Connecting-IP` — acceptable there because spoofing only resets the attacker's own bucket), the global throttle/trip is a system-state decision and **must not** read any request header, body, or param for its decision. The shed response carries only an error code + translation key, no message text. Per conventions Part 11: clean.

### 3.6 Alerting

`AlertService` is a Spring component called by `DatabaseHealthMonitor` on level transitions. Two channels:

- **Email**: via the existing `EmailService.sendPlainText(to, subject, body)` (Brevo SMTP, decisions.md 2026-05-22; audit Q8 confirmed the exact signature). Envelope is server-side, never client-supplied.
- **Telegram**: via the Telegram Bot API. New `TelegramAlertService`, timeout-bounded (reuses the 3s `ApplicationConfig.restTemplate()` bean, as the auto-trip's KV path does). Bot token + chat ID from env (`TELEGRAM_BOT_TOKEN`, `TELEGRAM_OPERATOR_CHAT_ID`).

Alert taxonomy:

| Trigger                                           | Email | Telegram |
| ------------------------------------------------- | ----- | -------- |
| YELLOW transition (entry)                         | no    | no       |
| RED transition (entry)                            | yes   | no       |
| CRITICAL transition + auto-trip fired             | yes   | yes      |
| GREEN transition (recovery from RED/CRITICAL)     | yes   | no       |
| Auto-trip fired but Cloudflare KV write failed    | yes   | yes      |
| Auto-trip skipped because a control layer was off | yes   | no       |

Alert content:

- First line / subject: severity + level + timestamp.
- Body: pool snapshot (active, idle, total, threads-awaiting), recent active history (last 60s), top 5 slow queries from `pg_stat_statements` (CRITICAL only, if available), pointer to the `incident_log` row id.

Alerting is best-effort. Email or Telegram failure logs at WARN but does not block the trip. The trip is the primary mitigation; the alert is informational.

### 3.7 Forensic logging

A new `incident_log` Postgres table captures each transition. The audit confirmed the name is free in V1.

Schema (pre-production V1 fold, conventions Part 12 — edit `V1__init_schema.sql` in place, no `V2__`):

```sql
CREATE TABLE incident_log (
    id BIGSERIAL PRIMARY KEY,
    occurred_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    level VARCHAR(16) NOT NULL,  -- YELLOW / RED / CRITICAL / GREEN_RECOVERY
    pool_active INTEGER NOT NULL,
    pool_idle INTEGER NOT NULL,
    pool_total INTEGER NOT NULL,
    pool_max INTEGER NOT NULL,
    threads_awaiting INTEGER,
    auto_trip_fired BOOLEAN NOT NULL DEFAULT FALSE,
    auto_trip_succeeded BOOLEAN,
    slow_queries_top_n JSONB,  -- captured at CRITICAL only, when pg_stat_statements is available
    request_rate_per_sec NUMERIC(10,2),
    notes TEXT
);

CREATE INDEX idx_incident_log_occurred_at ON incident_log (occurred_at DESC);
CREATE INDEX idx_incident_log_level ON incident_log (level);
```

`slow_queries_top_n` is populated at CRITICAL trip time by querying `pg_stat_statements`. The audit could not confirm DO-side extension state (Q12) — no code reads `pg_stat_statements` today, and whether the managed-Postgres app user can read it is a DO control-panel setting outside the repo. **Operator must enable `pg_stat_statements.track = top` in DO and confirm the app user can read it** before this forensic field is populated. If unavailable, the slow-query field is left null; alerting and trip behavior are unaffected.

Retention: keep forever. Rows are small, one per transition. The Privacy Policy 90-day application-log retention does not apply — this is operational forensics, not user data. **The captured fields contain no PII** (pool metrics, level, request rate, and query _text_ from `pg_stat_statements` which is parameterized SQL, not bound values). If a future change adds user identifiers or IPs to a row, retention must be revisited.

### 3.8 Two-layer enable control

The audit found **no per-environment Configuration-table seed mechanism** — all three environments load the identical `data/configuration/*.sql`. So a Configuration row alone cannot express "off on dev, on in prod." Two existing precedents are combined:

**Layer 1 — per-environment YAML master switches** (the `app.images.sweeper.enabled` precedent: `${ENV_VAR:default}` with a different default per profile):

| Key (YAML)                 | dev default                         | stage default                      | prod default                       |
| -------------------------- | ----------------------------------- | ---------------------------------- | ---------------------------------- |
| `throttle.enabled`         | `${THROTTLE_ENABLED:false}`         | `${THROTTLE_ENABLED:true}`         | `${THROTTLE_ENABLED:true}`         |
| `auto.maintenance.enabled` | `${AUTO_MAINTENANCE_ENABLED:false}` | `${AUTO_MAINTENANCE_ENABLED:true}` | `${AUTO_MAINTENANCE_ENABLED:true}` |

Read at boot. This is the "does this feature exist in this environment at all" gate. Dev is inert because its default is `false` and the env var is unset there.

**Layer 2 — live Configuration-table runtime kill-switch** (shared seed, read per-poll/per-request, mutated on the spot):

| Key (Configuration)        | Default | Meaning                                                                                     |
| -------------------------- | ------- | ------------------------------------------------------------------------------------------- |
| `throttle.runtime.enabled` | `true`  | Live kill-switch. Flip to `false` to disable throttle + auto-trip immediately, no redeploy. |

The throttle filter runs, and the auto-trip fires, only when **both** layers permit: the environment opted in (YAML) AND the runtime switch is on (Configuration). Killing it on stage = flip the one Configuration row. Turning it off on dev = already off via YAML default. Reusing your on/off judgment across environments = the YAML defaults encode the per-env stance; the Configuration row is the live override.

### 3.9 Configuration and YAML entries

**Configuration table** (shared seed, re-read live):

| Key                                 | Default | Description                                                     |
| ----------------------------------- | ------- | --------------------------------------------------------------- |
| `throttle.runtime.enabled`          | `true`  | Layer-2 live kill-switch (throttle + auto-trip)                 |
| `threshold.yellow.ratio`            | `0.72`  | YELLOW boundary as fraction of live pool size                   |
| `threshold.red.ratio`               | `0.80`  | RED boundary                                                    |
| `threshold.critical.ratio`          | `0.94`  | CRITICAL boundary                                               |
| `threshold.yellow.window.seconds`   | `10`    | YELLOW sustained window                                         |
| `threshold.red.window.seconds`      | `10`    | RED sustained window                                            |
| `threshold.critical.window.seconds` | `30`    | CRITICAL sustained window                                       |
| `throttle.yellow.sleep.ms`          | `50`    | Sleep duration at YELLOW                                        |
| `auto.maintenance.locked_until`     | `0`     | Re-trip lockout timestamp (epoch ms); set after a manual untrip |

**YAML** (`application*.yaml`, per-environment, read at boot):

| Key                             | dev                                 | stage                              | prod                               |
| ------------------------------- | ----------------------------------- | ---------------------------------- | ---------------------------------- |
| `throttle.enabled`              | `${THROTTLE_ENABLED:false}`         | `${THROTTLE_ENABLED:true}`         | `${THROTTLE_ENABLED:true}`         |
| `auto.maintenance.enabled`      | `${AUTO_MAINTENANCE_ENABLED:false}` | `${AUTO_MAINTENANCE_ENABLED:true}` | `${AUTO_MAINTENANCE_ENABLED:true}` |
| `monitor.poll.interval.seconds` | `2`                                 | `2`                                | `2`                                |

### 3.10 External dead-man's-switch

Better Stack uptime monitor (free tier) polls `/actuator/health/readiness` from outside DO. Alerts the operator when the health check fails for 2+ minutes. Independent fallback — if the entire backend goes down (not just the DB), internal alerting is dead and Better Stack notices. Operator setup, no backend code; documented in the runbook.

---

## 4. Components

Five new Spring components in `oglasino-backend`:

### 4.1 `DatabaseHealthMonitor`

- `@Component` `@Scheduled` polling the `HikariDataSource` MXBean every `monitor.poll.interval.seconds`.
- Maintains a rolling history of active-connection counts (last 60s).
- Resolves fractional thresholds against the live `getMaximumPoolSize()` each poll.
- Computes the current pressure level via sustained-window logic (and the `threadsAwaiting > 0` discriminator for CRITICAL).
- Exposes `getCurrentLevel()` for filter consumption (in-memory volatile, no DB hit on the request path).
- Fires level transitions to `AlertService` and `MaintenanceAutoTripService`.
- Persists each transition to `incident_log`.
- The CRITICAL → KV-write hand-off to `MaintenanceAutoTripService` runs **off the poll thread** (async / separate executor) so a slow KV call cannot stall polling.

### 4.2 `RequestThrottleFilter`

- `OncePerRequestFilter`, registered via `SecurityConfig.addFilterAfter(..., RateLimitFilter.class)`.
- Reads both enable layers, the exclusion list, and the pressure level.
- Applies sleep / shed per § 3.5.

### 4.3 `MaintenanceAutoTripService`

- Listens for CRITICAL transitions.
- Checks both enable layers and the lockout window.
- Asserts maintenance ON (no-op if already on) via `DefaultCloudflareKvService`, using a timeout-bounded client.
- Records the trip result into the `incident_log` row.

### 4.4 `AlertService`

- Email via existing `EmailService.sendPlainText`.
- Telegram via `TelegramAlertService`.
- Best-effort; alerting failures log WARN, never block the trip.

### 4.5 `TelegramAlertService`

- timeout-bounded (reuses the 3s `ApplicationConfig.restTemplate()` bean, as the auto-trip's KV path does), wrapping the Telegram Bot API.
- Token + chat id from env.
- Single method: `sendMessage(String text)`.

---

## 5. Trust boundaries

Every value used in throttle / shed / trip decisions is server-derived:

- Pressure level: `HikariDataSource` MXBean (server-internal).
- Thresholds, windows, kill-switch: Configuration table (server DB).
- Master switches, poll interval: YAML/env (server-owned).
- Cloudflare credentials: env (server-owned).
- Alert recipients: env (server-owned).

No client-supplied values consulted. The throttle filter reads no request header, body, or param for its decision (contrast `RateLimitFilter`'s client-header bucket key, which the throttle must not copy). Per conventions Part 11: clean.

---

## 6. Error contract

Per conventions Part 7 (with the filter-layer `translationKey` extension noted in § 3.5):

| Endpoint                                   | Status when throttled | Body                                                                                                         |
| ------------------------------------------ | --------------------- | ------------------------------------------------------------------------------------------------------------ |
| Any non-excluded endpoint, RED or CRITICAL | 503                   | `{ "errors": [{ "field": null, "code": "SERVICE_DEGRADED", "translationKey": "system.service_degraded" }] }` |

`Retry-After: 2` header on all 503 throttle responses.

`SERVICE_DEGRADED` lives in `SystemErrorCode` with `HttpStatus.SERVICE_UNAVAILABLE` (503).

Web axios 503-with-`Retry-After` handling is a possible future web enhancement; the feature ships fully functional without it (browsers natively respect `Retry-After` for many cases; ad-hoc requests fail loudly under overload, which is acceptable). Not in scope.

---

## 7. Translations

One new key in the `ERRORS` namespace (conventions Part 6 Rule 4):

| Key                       | Backend code       | Languages       | IDs (implemented)                      |
| ------------------------- | ------------------ | --------------- | -------------------------------------- |
| `system.service_degraded` | `SERVICE_DEGRADED` | EN, SR, RU, CNR | EN 3167 / RS 5267 / RU 7367 / CNR 1067 |

Translation IDs are provisional and subject to the operator's post-feature ID-renumber job — do not treat them as fixed. (The originally audit-reserved IDs — EN 3162 / RS 5262 / RU 7362 / CNR 1062 — were consumed by the email-notifications feature before this feature's seeds were written, so the implemented seeds use the IDs above.)

Seeded inline-append into the existing four `0001-data-web-translations-*.sql` files, at the end of the ERRORS group before the `-- ERRORS END` marker (conventions Part 6 Rule 3). IDs are within the reserved gap (audit Q16). EN final; RS/RU/CNR drafted, pending native-translator review (same precedent as Consent Mode v2, User Deletion).

---

## 8. Operational concerns

### 8.1 DO connection budget

Prod: 25-connection cluster cap, 18 HikariCP pool, ~5 DO-reserved, ~2 for operator `psql`. Tight. At CRITICAL the maintenance gate is flipped, user traffic stops, and HikariCP idle-timeout eventually releases active connections, restoring operator access within minutes. **Risk Watch entry recommended:** 25-connection ceiling; the feature buys time, not scale.

### 8.2 `pg_stat_statements` enablement

Currently unused by code; DO-side state unconfirmed (audit Q12). Operator must enable `track = top` in DO and confirm the app user can read the view, or the slow-query forensic field stays null. Alerting and trip are unaffected either way.

### 8.3 Telegram bot setup

Operator: BotFather `/newbot`, message the bot once to find the chat id, set `TELEGRAM_BOT_TOKEN` / `TELEGRAM_OPERATOR_CHAT_ID`. ~10 min. Documented in the runbook.

### 8.4 Better Stack setup

Operator: free tier, monitor on `https://oglasino.com/actuator/health/readiness`, email + Telegram channels, 30s interval, 2-failure threshold. ~10 min. Documented in the runbook.

### 8.5 Cloudflare credentials

Already configured (`CLOUDFLARE_ACC_ID`, `CLOUDFLARE_KV_CONFIG_TOKEN`, `CLOUDFLARE_NAMESPACE_ID`) and used by `DefaultCloudflareKvService`. The auto-trip reuses them; no new env var.

### 8.6 Runbook

New file `oglasino-docs/infra/runbooks/database-overload.md`: alert taxonomy, diagnostic steps (query `incident_log`, DO metrics, `pg_stat_statements`), mitigation (three-step restore), threshold tuning, disable procedures (flip `throttle.runtime.enabled`, or the YAML master per env). Authored by Docs/QA at Phase 5 close-out.

---

## 9. Platform adoption

Backend-only. No web/mobile/router/Firestore Rules adoption. Web 503 + `Retry-After` handling is a separate future decision.

---

## 10. Definition of done

- Five Spring components implemented and tested in `oglasino-backend`.
- `incident_log` table via V1 schema fold (pre-prod, Part 12).
- Configuration rows seeded; YAML master switches added per profile (dev false / stage+prod true) plus poll interval.
- `SERVICE_DEGRADED` added to `SystemErrorCode`; `system.service_degraded` seeded in 4 locales at the reserved IDs.
- `RequestThrottleFilter` placed via `SecurityConfig.addFilterAfter(..., RateLimitFilter.class)`.
- Auto-trip asserts-on (no-op if already on), writes both flags, uses a timeout-bounded KV client, off the poll thread.
- Backend lint + test green (`./mvnw spotless:check`, `./mvnw test`).
- Operator runbook authored at `infra/runbooks/database-overload.md`.
- Operator: `pg_stat_statements.track = top` enabled in DO (or slow-query field accepted as null).
- Operator: Telegram bot created, env vars set.
- Operator: Better Stack monitor wired with email + Telegram.
- Manual smoke (stage, pool of 8): trigger YELLOW (load test), verify sleep applied.
- Manual smoke: trigger RED, verify 503 + `Retry-After`.
- Manual smoke: trigger CRITICAL (saturate the 8-pool with threads waiting), verify auto-trip asserts maintenance, both flags flip, email + Telegram arrive, `incident_log` row written.
- Manual smoke: verify auto-trip no-ops when maintenance already on.
- Manual smoke: verify manual untrip via three-step restore works post-auto-trip, and the 5-min lockout holds.
- state.md row created.

---

## 11. Phase 2 audit — resolved

The audit (`oglasino-backend/.agent/audit-db-overload-protection.md`) resolved every open item. Key findings folded into this spec:

1. **Cloudflare KV client exists** (`DefaultCloudflareKvService`) and already PUTs arbitrary keys + writes both maintenance flags. Auto-trip reuses it; asserts-on rather than toggles; uses a timeout-bounded client (§ 3.3). The existing no-timeout `RestTemplate` on `toggleMaintenance` is logged as a separate issue.
2. **Filter ordering** — no filter has `@Order`; place via the Security chain (§ 3.4).
3. **HikariCP metrics** — no HTTP exposure; read in-process from the bean (§ 3.1).
4. **`EmailService.sendPlainText`** confirmed (§ 3.6).
5. **`incident_log`** name free (§ 3.7).
6. **Tomcat threads** dev 200 / stage 50 / prod 100; sleep math holds (§ 3.5).
7. **Hikari timing** — `leak-detection-threshold` disabled all envs; threads-awaiting is the available starvation signal (§ 3.1–3.2).
8. **`pg_stat_statements`** unused by code; DO-side grant unconfirmed (§ 8.2).
9. **`@Scheduled` resolves from YAML** — poll interval to YAML (§ 3.1).
10. **`SystemErrorCode`** is the home; body carries `translationKey` (§ 3.5, § 6).
11. **Pool size differs per env** (dev 20 / stage 8 / prod 18) — thresholds are live-pool fractions (§ 3.1–3.2).
12. **No per-env Configuration seed** — two-layer YAML + Configuration enable (§ 3.8).

---

## 12. Risks and follow-ups

**Risk: thresholds wrong at launch.** Ratios (0.72/0.80/0.94) are educated guesses. Real traffic may exceed them calmly (over-alerts) or stay below during distress (under-protects). All Configuration-tunable; operator monitors `incident_log` and adjusts.

**Risk: auto-trip during a legitimate spike.** A genuine surge could trigger CRITICAL and stop the spike converting. Operator can pre-disable auto-trip (`throttle.runtime.enabled = false`, or the YAML master) during expected high-traffic events.

**Risk: small-pool threshold collapse.** On the 8-connection stage pool the defaults resolve to distinct adjacent counts (6 / 7 / 8), so no level is lost there; a genuine collapse (two levels on the same integer) needs a much smaller pool (e.g. a pool of 2), where the higher-level-wins tie rule (§ 3.2) handles it. Stage is still a coarse test of the gradient; prod (18) and dev (20) have more room.

**Risk: Cloudflare KV write latency on the trip path.** Mitigated by the timeout-bounded client and the off-poll-thread hand-off (§ 3.3, § 4.1).

**Follow-up: tiered request classification.** Excluded from v1; revisit if specific endpoints repeatedly cause saturation.

**Follow-up: 503 + `Retry-After` handling on web.** Future web chat.

**Follow-up: external metrics/observability (Prometheus/Grafana).** The throttle reads pressure in-process and exposes nothing over HTTP. If operator trend-visibility is wanted, a Prometheus registry + Grafana is a clean separate feature — it does not belong on the throttle's decision path. Future chat.

**Follow-up: DO tier upgrade plan.** 25 connections is the launch ceiling. Written upgrade plan goes in state.md Risk Watch.

---

## Spec decisions (Phase 3 seam resolutions)

Recorded so the reasoning survives without re-reading the chat. Each resolved a seam between the spec's going-in assumptions and the audited code.

1. **Thresholds as fractions of the live pool size, not absolute counts.** The audit found pool size differs per env (dev 20 / stage 8 / prod 18); absolute 13/14/17 was prod-only and would make CRITICAL unreachable on stage's pool of 8. Read `getMaximumPoolSize()` live; resolve `ceil(ratio × size)` per poll. _Rejected:_ absolute counts as Configuration rows — every pool-size change becomes two coordinated edits, and a forgotten one silently mis-calibrates protection.

2. **In-process bean read for the pressure signal, no Prometheus.** No `micrometer-registry-prometheus` on the classpath; the throttle runs in the same JVM as the pool. _Rejected:_ adding Prometheus to expose metrics over HTTP that the throttle would scrape from itself — strictly worse than a direct bean read. Prometheus/Grafana remains a good _operator-observability_ feature, queued separately (§ 12).

3. **Active-count gradient + threads-awaiting CRITICAL discriminator.** Active-count saturates at pool max and can't distinguish "busy" from "starving"; threads-awaiting is the leading starvation signal (and `leak-detection-threshold` is disabled everywhere, so no leak instrumentation to lean on). CRITICAL gates on count AND `threadsAwaiting > 0`. _Rejected:_ threads-awaiting as the sole signal — poor gradient below saturation.

4. **Two-layer enable: per-env YAML master + live Configuration kill-switch.** The audit found no per-env Configuration seed (all envs load the same SQL), so a Configuration row alone can't express "off on dev." YAML carries the per-env stance (`${ENV_VAR:default}`, the `app.images.sweeper.enabled` precedent); Configuration carries the live override. Both must permit. _Rejected:_ Configuration-only flags (spec's original §3.8) — same value everywhere, can't express per-env.

5. **Poll interval in YAML, not Configuration.** `@Scheduled` resolves from the Spring Environment at bean construction and can't read the runtime Configuration cache (audit Q13; user-deletion crons precedent). Thresholds stay in Configuration (read per-poll, not at construction).

6. **`SERVICE_DEGRADED` in `SystemErrorCode`; 503 body carries `translationKey`.** The `system-error-code-split` shipped, so `SystemErrorCode` is the home (not the old `ProductErrorCode` dumping ground). The as-built `RateLimitFilter` 429 body includes `translationKey`; the 503 mirrors it so clients localize for free. Noted as a filter-layer extension of Part 7's `{field, code}` envelope, not a Part 7 amendment.

7. **Filter placement via the Security chain.** No filter declares `@Order`; only the Security chain trio is deterministic. `addFilterAfter(..., RateLimitFilter.class)` gives the intended position without depending on the undefined `@Component` ordering bucket.

8. **Auto-trip asserts maintenance-on (not raw toggle), writes both flags, timeout-bounded, off the poll thread.** `toggleMaintenance()` is a flip — raw use would turn maintenance _off_ if it were already on when CRITICAL fired. The trip asserts on (no-op if already on). Writes both web + backend flags (operator decision: backend overloaded ⇒ web unusable anyway). The KV client must be timeout-bounded — the trip calls Cloudflare precisely when degraded, and the existing no-timeout `RestTemplate` could hang the monitor thread on the API the trip depends on. The KV write runs off the poll thread so it can't stall polling.
