# DB Overload Protection

**Status:** planned
**Last updated:** 2026-05-25
**Phase 2 audit status:** not yet run — see § 11

Graduated response to database load pressure: slow requests early, shed requests at threshold, auto-flip the maintenance gate at critical saturation. Operator (Igor, solo) is alerted via email + Telegram, with full forensic logging to `incident_log` for post-incident root-cause.

This is structural defense against sustained-load DB exhaustion (Shape B from the 2026-05-25 intake). The 2026-05-14 connection-pool incident (decisions.md) addressed cold-cache thundering herd (Shape A); the open `connection-pool-hardening` backlog covers per-key single-flight dedup. This feature is independent of both — it protects against any cause of pool saturation, regardless of mechanism.

The feature is scoped to `oglasino-backend`. No web, mobile, router, or Firestore Rules changes. Cloudflare KV writes are server-to-server.

---

## 1. Goals

1. **Automatic maintenance flip when DB is genuinely overloaded.** Backend self-trips Cloudflare's `maintenance.backend.active` flag at CRITICAL pressure level, sustained 30s. Untrip is manual via the existing three-step restore runbook (decisions.md 2026-05-15).
2. **Graduated response below CRITICAL.** YELLOW = 50ms sleep per request. RED = HTTP 503 + `Retry-After: 2` shedding. Each level reduces arrival rate, giving the DB headroom to drain.
3. **Operator alerting.** Email-only on RED. Email + Telegram on CRITICAL. No alert on YELLOW (too frequent at this threshold).
4. **Full forensic logging.** At CRITICAL trip, capture HikariCP pool snapshot, `pg_stat_statements` slow-query top-N, request rate, thread dump. Persisted to `incident_log` Postgres table plus application logs.
5. **Independent kill-switches.** Two Configuration-tunable flags: `request.throttle.enabled` (filter on/off) and `auto.maintenance.enabled` (auto-trip on/off). Operator can disable either without redeploy.

---

## 2. Non-goals

- **Automatic un-trip.** Site stays in maintenance until operator manually un-trips via the three-step restore. Avoids flap mode where overload returns the moment traffic resumes.
- **Tiered request classification.** All endpoints get the same treatment at each pressure level. Excludes the "expensive routes shed first, cheap routes always admitted" design (tier D from intake). Acceptable cost; revisit if launch traffic patterns suggest otherwise.
- **Web, mobile, router, Firestore Rules changes.** Pure backend feature.
- **Bigger DO Postgres tier.** Out of budget for now. The feature buys time on the 25-connection cap; it doesn't replace scaling. Logged as Risk Watch in state.md.
- **Manual un-trip automation.** The three-step restore runbook (decisions.md 2026-05-15) remains the un-trip path. Not redesigned here.

---

## 3. Architecture

### 3.1 Pressure signal

The signal source is HikariCP's `active connections` count, read via Spring Boot Actuator + Micrometer. HikariCP exposes this through its native MBean (`com.zaxxer.hikari:type=Pool (HikariPool-1)` → `ActiveConnections`) and Micrometer's `hikaricp.connections.active` gauge.

Polling: `DatabaseHealthMonitor` polls every 2 seconds via a `@Scheduled` task. Each poll computes the current pressure level. Sustained-window logic compares against the rolling history.

Why active connections, not pool utilization percentage: with 18-connection pool, each connection is ~5.5% of pool. Percentages are too coarse for clean thresholds. Absolute counts are clearer at this scale.

Why not connection acquisition wait time: lagging indicator — by the time wait time spikes, the pool is already exhausted. Active count is leading.

### 3.2 Pressure levels

| Level | Trigger condition | Action |
|---|---|---|
| **GREEN** | active < 13 | Nothing |
| **YELLOW** | active ≥ 13 sustained 10s | `RequestThrottleFilter` sleeps `throttle.yellow.sleep.ms` (default 50) before passing the request through |
| **RED** | active ≥ 14 sustained 10s | `RequestThrottleFilter` returns HTTP 503 + `Retry-After: 2` instead of forwarding |
| **CRITICAL** | active ≥ 17 sustained 30s | `MaintenanceAutoTripService` flips Cloudflare `maintenance.backend.active = "true"`. Filter continues shedding until Cloudflare propagation (≤30s) takes over |

Threshold semantics: thresholds are *floor-inclusive* (active ≥ N triggers the level). The boundary at 13 = "real traffic above the 10-connection idle floor"; 14 = "pool starting to stretch"; 17 = "one connection from exhaustion."

All thresholds and window durations are stored in the `Configuration` table and read on each poll. Operator can tune without redeploy.

### 3.3 Trip policy

**Auto-trip, manual untrip.** CRITICAL fires once and the site stays in maintenance until operator runs the three-step restore manually. After auto-trip, a lockout flag in Configuration (`auto.maintenance.locked_until`) prevents re-trip for 5 minutes after the next manual untrip, preventing trip-untrip-trip cascades.

**Both flags must be ON for trip to fire:**
- `auto.maintenance.enabled = "true"` (master flag for the trip subsystem)
- Cloudflare `admin.bypass.disabled` flag remains at its existing default (so operator can still reach admin during auto-tripped maintenance, same as manual deploy maintenance per decisions.md 2026-05-15)

The trip call writes Cloudflare KV via the REST API. Mechanism: HTTP POST with `CF_API_TOKEN` from env, against the same KV namespace the deploy workflows already write (per decisions.md 2026-05-15). **VERIFY DURING AUDIT:** does `oglasino-backend` already have a Cloudflare KV client (for the existing maintenance gate's deploy flow, or for any other purpose), or does this feature need to add one? If add: a thin `RestClient`-based wrapper is sufficient; no new dependency needed.

### 3.4 Filter placement

`RequestThrottleFilter` sits in the Spring filter chain at a specific position:

- **After** `FirebaseAuthFilter` (auth must run; throttle is for authenticated and anonymous alike)
- **After** `RateLimitFilter` (rate limit is per-client; throttle is global)
- **Before** `DispatcherServlet` (throttle decides whether to even enter Spring MVC)

Filter exclusion list: requests matching these paths bypass throttling entirely.

- `/actuator/health/*` (Better Stack and DO health probes must succeed)
- `/api/public/maintenance/*` (the maintenance page itself must render)
- `/api/secure/admin/**` (admin must operate during overload, same posture as the admin bypass on the Cloudflare maintenance gate)

**VERIFY DURING AUDIT:** current explicit `@Order` of the four `@Component` filters (`FirebaseAuthFilter`, `CurrentLanguageFilter`, `BaseSiteFilter`, `RateLimitFilter`). The existing connection-pool-hardening backlog item flags missing `@Order` on these. This feature should not block on that fix, but the audit must establish the current ordering so `RequestThrottleFilter`'s position is correct.

### 3.5 Filter behavior

Per request:

1. Read `request.throttle.enabled`. If `false`, pass through (no-op).
2. Check exclusion list. If matched, pass through.
3. Read current pressure level from `DatabaseHealthMonitor` (in-memory volatile state, no DB hit).
4. GREEN: pass through.
5. YELLOW: `Thread.sleep(yellowSleepMs)`, then pass through.
6. RED or CRITICAL: write 503 response with `Retry-After: 2` header, log at INFO. Do not pass through.

The sleep at YELLOW holds the Tomcat thread for the sleep duration. With 50ms sleep and default Tomcat `max-threads=200`, this scales to ~4,000 req/s before thread exhaustion — well past anything this tier will see. Sleep duration is Configuration-tunable; if it ever needs to grow past ~200ms, switch to shed instead.

The shed response includes a minimal JSON body matching the Part 7 error contract:

```json
{
  "errors": [
    { "field": null, "code": "SERVICE_DEGRADED" }
  ]
}
```

New error code `SERVICE_DEGRADED` added to whichever cross-cutting error-code enum exists post-`ProductErrorCode` refactor (currently lives in `ProductErrorCode`'s "Cross-cutting" section per issues.md 2026-05-14 HIGH entry). Translation key `service.degraded` seeded in 4 locales (`ERRORS` namespace, conventions Part 6 Rule 1).

**Trust boundary check.** The throttle decision is server-derived from HikariCP metrics. No client-supplied values are consulted. The shed response carries only an error code, not message text (per conventions Part 8). Clean.

### 3.6 Alerting

`AlertService` is a Spring component called by `DatabaseHealthMonitor` on level transitions. Two channels:

- **Email**: via existing `EmailService` (Brevo SMTP, per decisions.md 2026-05-22). Reuses the existing `sendPlainText` interface. **VERIFY DURING AUDIT:** the existing `EmailService` interface is sufficient for plain-text alerts. If structured (HTML, attachments) is wanted later, extend the interface in a follow-up.
- **Telegram**: via Telegram Bot API. New `TelegramAlertService` component. Bot token + chat ID from env (`TELEGRAM_BOT_TOKEN`, `TELEGRAM_OPERATOR_CHAT_ID`). Token has zero cost; bot creation is a 10-minute BotFather setup by operator.

Alert taxonomy:

| Trigger | Email | Telegram |
|---|---|---|
| YELLOW transition (entry) | no | no |
| RED transition (entry) | yes | no |
| CRITICAL transition + auto-trip fired | yes | yes |
| GREEN transition (recovery from RED/CRITICAL) | yes | no |
| Auto-trip fired but Cloudflare API call failed | yes | yes |
| Auto-trip skipped because `auto.maintenance.enabled=false` | yes | no |

Alert content:

- Subject / first line: severity + level + timestamp
- Body: pool snapshot (active, idle, total), recent active history (last 60s), top 5 slow queries from `pg_stat_statements` (CRITICAL only), pointer to `incident_log` row ID for forensics

Alerting failure mode: alerts are best-effort. Email or Telegram failure logs at WARN but does not block the trip. The trip itself is the primary mitigation; the alert is informational.

### 3.7 Forensic logging

A new `incident_log` Postgres table captures each transition for post-incident analysis.

Schema (pre-production V1 fold, per conventions Part 12):

```sql
CREATE TABLE incident_log (
    id BIGSERIAL PRIMARY KEY,
    occurred_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    level VARCHAR(16) NOT NULL,  -- YELLOW / RED / CRITICAL / GREEN_RECOVERY
    pool_active INTEGER NOT NULL,
    pool_idle INTEGER NOT NULL,
    pool_total INTEGER NOT NULL,
    pool_wait_queue_depth INTEGER,
    auto_trip_fired BOOLEAN NOT NULL DEFAULT FALSE,
    auto_trip_succeeded BOOLEAN,
    slow_queries_top_n JSONB,  -- captured at CRITICAL only
    request_rate_per_sec NUMERIC(10,2),
    notes TEXT
);

CREATE INDEX idx_incident_log_occurred_at ON incident_log (occurred_at DESC);
CREATE INDEX idx_incident_log_level ON incident_log (level);
```

`slow_queries_top_n` is populated at CRITICAL trip time by querying `pg_stat_statements`. **Operator must enable `pg_stat_statements.track = top` in DO control panel** before this feature ships (currently set to `None` per intake). Without it, slow-query forensics are empty.

Retention: keep forever. Rows are small, one per incident. The Privacy Policy 90-day application-log retention does not apply — this is operational forensics, not user data. **VERIFY DURING AUDIT:** confirm that incident_log rows contain no PII (no user IDs, no IP addresses). If they do, retention policy needs revision.

### 3.8 Configuration table entries

New rows seeded into `Configuration`:

| Key | Default | Description |
|---|---|---|
| `request.throttle.enabled` | `true` | Master flag for the filter |
| `auto.maintenance.enabled` | `true` | Master flag for the auto-trip subsystem |
| `threshold.yellow.active.connections` | `13` | YELLOW boundary |
| `threshold.red.active.connections` | `14` | RED boundary |
| `threshold.critical.active.connections` | `17` | CRITICAL boundary |
| `threshold.yellow.window.seconds` | `10` | YELLOW sustained window |
| `threshold.red.window.seconds` | `10` | RED sustained window |
| `threshold.critical.window.seconds` | `30` | CRITICAL sustained window |
| `throttle.yellow.sleep.ms` | `50` | Sleep duration at YELLOW |
| `monitor.poll.interval.seconds` | `2` | How often `DatabaseHealthMonitor` polls HikariCP |

Reads on each poll for thresholds and on each request for `request.throttle.enabled`. Standard `getConfig` cache eviction applies (per the translations precedent in decisions.md 2026-05-14).

### 3.9 External dead-man's-switch

Better Stack uptime monitor (free tier) polls `/actuator/health/readiness` from outside DO infrastructure. Configured to alert operator when health check fails for 2+ minutes. This is the independent fallback — if the entire backend goes down (not just DB), internal alerting is dead, Better Stack notices.

Operator action: sign up for Better Stack free tier, add monitor pointing at production health endpoint, set alert channels (email + Telegram). 10-minute setup. No backend code involvement. Documented in the operator runbook.

---

## 4. Components

Five new Spring components in `oglasino-backend`:

### 4.1 `DatabaseHealthMonitor`
- `@Component` `@Scheduled` polling HikariCP metrics every 2s
- Maintains rolling history of active-connection counts (last 60s)
- Computes current pressure level via sustained-window logic
- Exposes `getCurrentLevel()` for filter consumption
- Fires level transitions to `AlertService` and `MaintenanceAutoTripService`
- Persists each transition to `incident_log`

### 4.2 `RequestThrottleFilter`
- `OncePerRequestFilter`
- Reads pressure level from `DatabaseHealthMonitor`
- Applies sleep / shed per § 3.5
- Respects exclusion list and master flag

### 4.3 `MaintenanceAutoTripService`
- Listens for CRITICAL transitions
- Reads `auto.maintenance.enabled` flag
- Writes Cloudflare KV via REST API
- Records lockout flag to prevent re-trip
- Updates `incident_log` row with trip result

### 4.4 `AlertService`
- Email alerts via existing `EmailService` (Brevo)
- Telegram alerts via new `TelegramAlertService`
- Best-effort; alerting failures don't block trips

### 4.5 `TelegramAlertService`
- `RestClient`-based wrapper around Telegram Bot API
- Token + chat ID from env vars
- Single method: `sendMessage(String text)`

---

## 5. Trust boundaries

Every value used in throttle / shed / trip decisions is server-derived:

- Pressure level: derived from HikariCP metrics (server-internal)
- Thresholds: read from Configuration table (server-owned)
- Master flags: read from Configuration table (server-owned)
- Cloudflare API token: env var (server-owned)
- Alert recipient addresses: env var (server-owned)

No client-supplied values consulted. Per conventions Part 11: clean.

---

## 6. Error contract

Per conventions Part 7:

| Endpoint | Status when throttled | Body |
|---|---|---|
| Any non-excluded endpoint, RED or CRITICAL | 503 | `{ "errors": [{ "field": null, "code": "SERVICE_DEGRADED" }] }` |
| `Retry-After: 2` header on all 503 responses | | |

Frontend handling: any `axios` interceptor for 503 status should respect `Retry-After`. **VERIFY DURING AUDIT:** does the web app's axios setup currently retry on 503? If yes, the throttle will naturally smooth client load. If not, that's a follow-up web feature (not part of this backend-only feature).

---

## 7. Translations

One new key in the `ERRORS` namespace per conventions Part 6 Rule 4:

| Key | Backend code | Languages |
|---|---|---|
| `service.degraded` | `SERVICE_DEGRADED` | EN, SR, RU, CNR |

Seeded inline-append into the existing four locale SQL files per conventions Part 6 Rule 3.

---

## 8. Operational concerns

### 8.1 DO connection budget

Current state: 25-connection cluster cap, 18 HikariCP pool, ~5 DO-reserved. Operator has ~2 connections for `psql` admin access during incidents. Tight but workable.

Risk: If operator needs to run admin queries during CRITICAL state and DO-reserved + HikariCP-active + ongoing-backups consume 25, operator cannot connect to diagnose. Mitigation: the maintenance gate is auto-flipped at CRITICAL, so user traffic stops, and HikariCP idle-timeout (default 600s) eventually releases active connections. Within minutes, operator has connections again.

**Risk Watch entry recommended in state.md:** 25-connection ceiling. The feature buys time, doesn't replace scaling. Operator should plan to upgrade DO tier (Basic 2GB → 25 connections still, jump to General Purpose 8GB → 75 connections) when budget allows.

### 8.2 `pg_stat_statements` enablement

Currently `track = None`. Operator action required before feature ships: change to `track = top` in DO control panel. One control-panel click, requires DB restart. Schedule during planned maintenance window.

If operator forgets to enable: feature works, but slow-query forensics in `incident_log` are empty. Alerting and trip behavior unaffected.

### 8.3 Telegram bot setup

Operator action: BotFather → `/newbot` → name + handle → receive token. Then message the bot once to populate `getUpdates`, find own chat ID. Set env vars `TELEGRAM_BOT_TOKEN`, `TELEGRAM_OPERATOR_CHAT_ID`. 10-minute task, documented in the operator runbook.

### 8.4 Better Stack setup

Operator action: sign up for free tier, add a monitor pointing at `https://oglasino.com/actuator/health/readiness`, configure email + Telegram channels for that monitor (Better Stack supports Telegram natively), set check interval to 30s, set alert threshold to 2 consecutive failures. 10-minute task, documented in the operator runbook.

### 8.5 Cloudflare API token

Backend needs a Cloudflare API token with permission to write the maintenance KV. **VERIFY DURING AUDIT:** does the backend already have such a token configured (would be unexpected — the existing maintenance gate is written from GitHub Actions deploy workflows, not from the running backend), or is this a new env var (`CF_API_TOKEN`, with appropriate KV write scope)? Token created in Cloudflare dashboard, scoped to the specific KV namespace, stored as env var.

### 8.6 Runbook

New file `oglasino-docs/infra/runbooks/database-overload.md` documenting:

- Alert taxonomy: what each email/Telegram message means
- Diagnostic steps: query `incident_log`, check DO metrics dashboard, examine `pg_stat_statements`
- Mitigation steps: confirm cause, run three-step restore (`admin-bypass-allow.sh` → cache refresh → `maintenance-off.sh`)
- Tuning: when to adjust thresholds in Configuration
- Disable procedures: how to flip `request.throttle.enabled` or `auto.maintenance.enabled` off during planned operations

Created by Docs/QA as part of the feature's Phase 5 close-out.

---

## 9. Platform adoption

Backend-only feature. No web, mobile, router, or Firestore Rules adoption.

The frontend axios setup may benefit from 503 + `Retry-After` handling, but that's a separate decision belonging to a future web chat. The feature ships fully functional without it (browsers natively respect `Retry-After` for many cases; ad-hoc requests will fail loudly, which is acceptable under overload).

---

## 10. Definition of done

- Five Spring components implemented and tested in `oglasino-backend`
- `incident_log` table added via V1 schema fold (pre-prod) or new Flyway migration (post-prod) per conventions Part 12
- 10 Configuration rows seeded with documented defaults
- One translation key seeded in 4 locales
- `RequestThrottleFilter` placed correctly in filter chain (audit-verified position)
- Backend lint + test green per conventions Part 4
- Operator runbook authored at `infra/runbooks/database-overload.md`
- Operator has enabled `pg_stat_statements.track = top` in DO control panel
- Operator has created Telegram bot and set env vars
- Operator has wired Better Stack monitor with email + Telegram channels
- Operator has confirmed Cloudflare KV API token is configured
- Manual smoke: trigger YELLOW (load test in stage), verify 50ms sleep applied
- Manual smoke: trigger RED (heavier load), verify 503 + `Retry-After`
- Manual smoke: trigger CRITICAL (saturate pool in stage), verify auto-trip fires, Cloudflare flag flips, email + Telegram arrive
- Manual smoke: verify manual untrip via three-step restore works post-auto-trip
- state.md row created for the feature

---

## 11. Open items for Phase 2 audit

The spec was drafted without a Phase 2 read-only audit. The following items are explicit unknowns that the audit must establish before Phase 5 implementation begins. None block spec acceptance; all block implementation:

1. **Cloudflare KV API client.** Does `oglasino-backend` already have a Cloudflare KV REST client (for any purpose), or does this feature add one? Affects scope of the KV-write path in `MaintenanceAutoTripService`.

2. **Filter chain ordering.** Current explicit `@Order` on the four `@Component` filters (`FirebaseAuthFilter`, `CurrentLanguageFilter`, `BaseSiteFilter`, `RateLimitFilter`). Connection-pool-hardening backlog flags this as missing. `RequestThrottleFilter` placement depends on the resolution.

3. **HikariCP metrics exposure.** Confirm Spring Boot Actuator + Micrometer + HikariCP metric registry are already wired and reachable, or whether the feature must add `management.endpoints.web.exposure.include=metrics` or similar.

4. **`EmailService` interface sufficiency.** Does the existing `EmailService.sendPlainText` interface cover the alerting use case, or does it need extension for richer alerting (markdown, attachments)? Plain-text is fine for v1; want explicit confirmation.

5. **`incident_log` table conflicts.** Verify no naming or column conflict with anything already in V1 schema.

6. **Web axios 503 retry handling.** Does the web app already respect `Retry-After` headers on 503 responses? If not, this is a follow-up web feature; the throttle works without it but UX during shed is degraded.

7. **Tomcat thread pool config.** Current `server.tomcat.threads.max`. Default is 200. Confirm it hasn't been tuned down — if it has, the YELLOW sleep math needs revisiting (50ms × N req/s must fit within max-threads).

8. **HikariCP timing config.** Current `connection-timeout`, `idle-timeout`, `leak-detection-threshold`. These interact with the throttle in non-obvious ways — for instance, if `connection-timeout` is very high, requests will queue at HikariCP even after the filter passes them through, partially undermining the shed.

9. **No PII in `incident_log`.** Confirm none of the captured fields (slow query text, request rate, pool metrics) contain user identifiers, IPs, or other PII. If they do, retention policy must change.

10. **`pg_stat_statements` query format.** Confirm the SQL to read top-N slow queries from `pg_stat_statements` in DO managed PG (some managed providers restrict access to extension internals).

Audit output: `.agent/audit-db-overload-protection.md` in `oglasino-backend`. Phase 5 cannot begin until audit is complete.

---

## 12. Risks and follow-ups

**Risk: thresholds wrong at launch.** Starting values (13/14/17) are educated guesses against the 18-connection pool. Real traffic may exceed these calmly (over-alerts) or stay below them during real distress (under-protects). All values are Configuration-tunable; operator monitors `incident_log` for false positives and adjusts.

**Risk: auto-trip during legitimate spike.** A genuine traffic surge (organic launch press, viral post) could trigger CRITICAL. The trip protects the DB, but it also stops the spike from converting. Operator can disable auto-trip via `auto.maintenance.enabled = false` proactively during expected high-traffic events.

**Risk: bot scraping triggers CRITICAL.** Aggressive scraping could exhaust the pool. `RateLimitFilter` per conventions Part 8 should catch this earlier, but if it doesn't, throttle + auto-trip is the next line of defense. Followed up in the `connection-pool-hardening` backlog (`RateLimitFilter` categorization for public reference-data endpoints).

**Risk: Cloudflare KV write latency.** The auto-trip path makes an HTTP call to Cloudflare. Typical latency is 50-200ms. During the call, the backend is still under load. Acceptable: the call happens once per trip, and the filter is already shedding to keep load off the DB.

**Follow-up: tiered request classification.** Excluded from v1. If operator observes that some endpoints are repeatedly the cause of saturation (search, autocomplete), tier-D request classification could shed them first while keeping cheap routes open. Future Mastermind chat.

**Follow-up: 503 + `Retry-After` handling on web.** Frontend axios should retry with backoff on 503 responses carrying `Retry-After`. Out of scope here; future web chat.

**Follow-up: DO tier upgrade plan.** 25 connections is the launch ceiling. Operator should have a written plan for when to upgrade and to what tier. Not part of this feature; goes in state.md Risk Watch.