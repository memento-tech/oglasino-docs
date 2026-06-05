# Runbook — Database Overload

Operator's incident-handling guide for the DB Overload Protection feature. This page is
operational only — for the design (why each threshold exists, how the signal is read, the
trust-boundary reasoning), see the spec at
[`../../features/db-overload-protection.md`](../../features/db-overload-protection.md).

## What this feature does

The backend watches its own database-connection pressure and responds in graduated steps. At
**YELLOW** it slows each request by a short sleep; at **RED** it sheds with HTTP 503 +
`Retry-After`; at **CRITICAL** it auto-asserts Cloudflare maintenance ON (both
`maintenance.web.active` and `maintenance.backend.active`), taking the site into maintenance. The
signal comes from `DatabaseHealthMonitor`, which polls the HikariCP pool MXBean every 2 seconds and
resolves the thresholds against the live pool size. The auto-trip is one-way: it asserts maintenance
on, but recovery is **manual** (see [Recovery](#recovery--the-un-trip-procedure)).

## Alert taxonomy

Each level transition can raise an alert. What you receive tells you what happened and what to do.

| Alert (channel)                          | What it means                                                                                   | What to do                                                                                                   |
| ---------------------------------------- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| **RED entry** (email)                    | The system is shedding load with 503s. The DB is under sustained pressure but not yet critical. No automated action was taken. | Investigate (see [Diagnostics](#diagnostic-steps)). Decide whether to act or let it drain.                   |
| **CRITICAL + trip fired** (email + Telegram) | Maintenance was auto-asserted **ON** — the site is now in maintenance. This is the destructive action. | Go to [Recovery](#recovery--the-un-trip-procedure). Diagnose before un-tripping.                             |
| **CRITICAL + trip FAILED** (email + Telegram) | CRITICAL pressure hit, but the maintenance assertion **did not succeed** (Cloudflare KV write failed). The site is still live and overloaded. | **URGENT.** Assert maintenance by hand and investigate the KV failure — see [Trip failed](#if-the-trip-failed). |
| **CRITICAL + trip skipped** (email)      | CRITICAL pressure hit, but a control gate was off (runtime kill-switch, YAML master, or an active lockout window), so no auto-trip happened. | Decide whether to flip maintenance manually. Check why the gate was off (see [Disabling](#disabling-the-feature)). |
| **GREEN recovery** from RED/CRITICAL (email) | Pressure subsided back to normal. Informational.                                                | None required. Note it in the incident record if you are writing one up.                                     |

By design, **YELLOW does not alert**, and a recovery from a YELLOW-only blip does not alert — that
level is too frequent to be worth paging on.

## Diagnostic steps

The forensic record lives in the `incident_log` table. Each level transition writes one row.

Read the most recent rows and the pool snapshot:

```sql
SELECT occurred_at, level, pool_active, pool_idle, pool_total, pool_max,
       threads_awaiting, auto_trip_fired, auto_trip_succeeded,
       request_rate_per_sec, slow_queries_top_n, notes
FROM incident_log
ORDER BY occurred_at DESC
LIMIT 20;
```

The columns are: `occurred_at`, `level` (YELLOW / RED / CRITICAL / GREEN_RECOVERY), the four pool
columns `pool_active` / `pool_idle` / `pool_total` / `pool_max`, `threads_awaiting`,
`auto_trip_fired`, `auto_trip_succeeded`, `slow_queries_top_n` (JSONB, populated at CRITICAL only
when `pg_stat_statements` is available), `request_rate_per_sec`, and `notes`.

- `slow_queries_top_n` carries the captured slow-query top-N at trip time. If it is null,
  `pg_stat_statements` was not readable — query the view directly (below) or see
  [Operator setup](#operator-setup-one-time-out-of-band).
- Cross-reference the DigitalOcean managed-database metrics (connection count, CPU, disk I/O) for
  the same window to confirm the pressure was real and find the cause.
- If the captured top-N is not enough, read `pg_stat_statements` directly for the live picture:

```sql
SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

## Recovery — the un-trip procedure

The auto-trip asserts maintenance on; it never recovers on its own. The site stays in maintenance
until you un-trip manually. This is deliberate (auto-trip, manual untrip) — it avoids a flap where
overload returns the instant traffic resumes.

To bring the site back, run the existing three-step restore (admin-bypass → cache refresh →
maintenance-off), documented in [`../cloudflare/maintenance.md`](../cloudflare/maintenance.md)
(Restore runbook) and decisions.md 2026-05-15. Do not duplicate those steps here — follow them
there.

**Set the re-trip lockout first (IMPORTANT, feature-specific).** Before (or as part of) the un-trip,
set the lockout so the monitor does not immediately re-trip on residual pressure as traffic resumes.
There is **no generic config-write endpoint** for this (established in Session 3a), so set it with a
direct DB UPDATE on the `configuration` table. Set `auto.maintenance.locked_until` to
`now + window` as an **epoch-millis** value (e.g. now + 5 minutes):

```sql
UPDATE configuration SET value = '<epoch_ms>' WHERE key = 'auto.maintenance.locked_until';
```

Compute the epoch-ms value for "5 minutes from now" with: `echo $(( ($(date +%s) + 300) * 1000 ))`
(seconds-since-epoch + 300, times 1000). The monitor reads `auto.maintenance.locked_until` live on
every poll and skips the trip while `now < locked_until`. The trip itself never writes this value —
only you do, here.

### If the trip failed

A **CRITICAL + trip FAILED** alert means the site is still live and overloaded — the KV write to
assert maintenance did not land. Assert maintenance by hand: run the three-step restore's
maintenance flip **in reverse** (turn the two `maintenance.*` flags ON rather than OFF) per
[`../cloudflare/maintenance.md`](../cloudflare/maintenance.md), then investigate why the KV write
failed (Cloudflare API reachability, token, namespace id).

## Tuning the thresholds

The threshold rows in the `configuration` table are read live on every poll — no redeploy needed:

| Key                                 | Default | Effect of raising it             |
| ----------------------------------- | ------- | -------------------------------- |
| `threshold.yellow.ratio`            | `0.72`  | YELLOW fires later               |
| `threshold.red.ratio`               | `0.80`  | RED (shedding) fires later       |
| `threshold.critical.ratio`          | `0.94`  | CRITICAL (auto-trip) fires later |
| `threshold.yellow.window.seconds`   | `10`    | YELLOW needs a longer sustain    |
| `threshold.red.window.seconds`      | `10`    | RED needs a longer sustain       |
| `threshold.critical.window.seconds` | `30`    | CRITICAL needs a longer sustain  |
| `throttle.yellow.sleep.ms`          | `50`    | Longer per-request slow-down at YELLOW |

The ratios are fractions of the **live** pool size — the monitor resolves `ceil(ratio × poolSize)`
each poll — so they self-adjust if the pool changes (droplet upgrade, per-env difference) with no
config edit. Raising a ratio makes that level fire later (at a higher active-connection count).

## Disabling the feature

Two independent levers; the throttle and the auto-trip run only when **both** layers permit.

- **Live kill-switch (no redeploy).** Set `throttle.runtime.enabled` to `false` in the
  `configuration` table. Takes effect on the next poll and disables **both** the throttle and the
  auto-trip immediately. This is the "kill it right now on the spot" lever.
- **Per-environment YAML masters (require a redeploy).** The `THROTTLE_ENABLED` and
  `AUTO_MAINTENANCE_ENABLED` env vars set the per-profile defaults (off on dev, on in stage/prod).
  This is the "off in this environment entirely" mechanism.

## Operator setup (one-time, out-of-band)

These must be done outside the codebase for the feature to be fully live.

- **Telegram.** Create a bot via BotFather, message it once, capture the chat id. Set
  `TELEGRAM_BOT_TOKEN` and `TELEGRAM_OPERATOR_CHAT_ID` in the droplet env file. These are plaintext
  secrets on the host — keep the env file **outside the git work-tree** and readable only by the
  deploy user, and make sure the compose/run step references the env file so the vars reach the
  container. Until set, Telegram alerts skip silently (email still works).
- **Alert email recipient.** Set `ALERT_OPERATOR_EMAIL`. Until set, email alerts skip silently.
- **`pg_stat_statements`.** Enable `track = top` in DO managed Postgres and confirm the app DB user
  can read the view. Until done, the slow-query forensic section is omitted from CRITICAL alerts and
  `slow_queries_top_n` stays null — trip and alerting are otherwise unaffected.
- **Better Stack.** Wire an external uptime monitor on `/actuator/health/readiness` (free tier,
  ~30s interval, 2-failure threshold, email + Telegram) as the dead-man's-switch for a total
  backend-down — when the backend is fully down, internal alerting cannot fire, so this external
  check is the only thing that notices (spec §3.10).

## Known limitations / pre-existing issues in this path

- The legacy admin maintenance toggle (`DefaultCloudflareKvService.toggleMaintenance()`) uses a
  no-timeout HTTP client and can hang on a slow Cloudflare API (issues.md 2026-06-03). The auto-trip
  path does **not** share this — it uses a timeout-bounded client — but if you flip maintenance by
  hand via the admin wrench, know that the legacy path can hang.
- The auth-filter double-registration (issues.md 2026-06-03) is unrelated to this feature's
  operation but is noted there.

## References

- [`../../features/db-overload-protection.md`](../../features/db-overload-protection.md) — the design spec
- [`../cloudflare/maintenance.md`](../cloudflare/maintenance.md) — the maintenance gate + three-step restore
- [`../../decisions.md`](../../decisions.md) — 2026-05-15 (maintenance split + restore), 2026-06-03 (this feature)
- [`../../issues.md`](../../issues.md) — 2026-06-03 (no-timeout KV `RestTemplate`; auth-filter double-registration)
