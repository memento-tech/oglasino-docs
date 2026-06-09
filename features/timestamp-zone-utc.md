# Timestamp Zone — UTC

**Status:** shipped
**Repos:** oglasino-backend (the flip + own-repo docs), oglasino-docs (config files + this spec)
**Branch:** backend `dev`

## Problem

`BaseEntity.createdAt` / `updatedAt` are `@CreationTimestamp` / `@UpdateTimestamp`
`LocalDateTime`, stored in `timestamp without time zone` columns. The container ran
`TZ=Europe/Belgrade` (`Dockerfile:13`), so Hibernate wrote Belgrade wall-clock. One reader
(`elasticsearch/converters/DocumentProductConverter:76`) re-labelled that value as UTC, producing an Elasticsearch
`Instant` skewed by the Belgrade offset (+1h winter / +2h summer), with a paired display
inverse at `elasticsearch/converters/ProductDetailsConverter:78`. The fix makes the container write UTC so the stored
values and the UTC-reading sites agree.

## Resolution

Set `ENV TZ` in `Dockerfile` from `Europe/Belgrade` to `Etc/UTC`. With no
`hibernate.jdbc.time_zone` pinned anywhere (audit-confirmed), this alone makes
`@CreationTimestamp` / `@UpdateTimestamp` write UTC wall-clock into the
`timestamp without time zone` columns. The app is pre-production — schema rebuilds from
`V1` on every reset, no stored rows to migrate — so the re-base has no data cost.

**The flip is the fix. No reader code changes.**

- `elasticsearch/converters/DocumentProductConverter:76` (`toInstant(ZoneOffset.UTC)`) becomes correct as-written.
- `elasticsearch/converters/ProductDetailsConverter:78` (the display inverse) moves in lock-step — correct, consistent.
- `AppVersionAdminDTO.updatedAt` via `DefaultAppVersionService:99` (`ZoneId.systemDefault()`)
  stays correct under any container zone — NOT double-corrected. Left alone.

## What this fixes

- The single genuine skew: ES-stored `ProductDocument.createdAt` `Instant` and the
  product-page displayed date both become true-UTC.

## What this does NOT touch (audit-confirmed)

- **`EmailDateFormatter` and `ProductIndexer` are flip-immune** — both operate on `Instant`s,
  not `BaseEntity` `LocalDateTime`s. The issue's "skewed email dates" symptom does not exist;
  email dates were already correct. (Corrects the issues.md 2026-06-06 "four call sites" framing
  to one genuine site.)
- **No business-logic math changes.** Every grace-period / retention / age-cutoff uses
  `Instant.now()` + constant-offset arithmetic (absolute). The lone `LocalDate.now().getYear()`
  site (`DefaultProductService:801`) has a New-Year-midnight cosmetic edge only.
- **All `timestamptz` / `Instant` columns are immune** (user-deletion, audit, messaging-cleanup,
  locks, `incident_log.occurred_at`). Only `timestamp`-without-tz columns shift, and pre-prod
  that shift is migration-free.

## Cron timing — accepted UTC shift

No `@Scheduled` cron sets a `zone=`. After the flip, each wall-clock expression is interpreted
as UTC. Operator-accepted (no zone pinning). New effective times:

| Job | Expression | New time (UTC) |
| --- | --- | --- |
| ProductImagesRemovalJob | `0 0 3 * * *` | 03:00 daily |
| ChatImagesRemovalJob | `0 0 3 * * SUN` | Sun 03:00 |
| UserDeletion hard-delete | `0 0 2 * * *` | 02:00 daily |
| UserDeletion reminder | `0 0 13 * * *` | 13:00 daily (= 15:00 Belgrade summer / 14:00 winter) |
| UserDeletion firebase reconcile | `0 0 3 * * SUN` | Sun 03:00 |
| UserDeletion audit purge | `0 0 4 * * SUN` | Sun 04:00 |
| ProductRemovalJob | `0 0 2 ? * SUN` | Sun 02:00 |
| ProductBaseCurrencyUpdater | `0 0 3 * * *` | 03:00 daily |
| MessagingCleanup | `0 0 3 ? * SUN` | Sun 03:00 |

No cron's correctness depends on zone — each recomputes its own cutoffs from `Instant.now()`.

## Tasks

### Brief 1 — backend (oglasino-backend, `dev`) — DONE / shipped 2026-06-06

1. `Dockerfile:13` — `ENV TZ=Europe/Belgrade` → `ENV TZ=Etc/UTC`. **Done.**
2. Own-repo docs-sync (same session — Part 4). Corrected the genuinely stale/zoneless
   wall-clock wording at `ProductBaseCurrencyUpdater.java:38`, `ProductRemovalJob.java:20`,
   `AppVersionAdminDTO.java:13`, the three `application-{dev,stage,prod}.yaml` cron comments,
   and `docs/16-image-pipeline.md:388`. Comments that already read "03:00 UTC" (inaccurate
   before the flip, true after) were left unedited — no-op churn avoided. **Done.**
3. No reader code changes. No new migration. **Confirmed.**
   Landed 2026-06-06; `spotless` clean, 969 tests green. See
   [sessions/2026-06-06-oglasino-backend-timestamp-zone-utc-2.md](../sessions/2026-06-06-oglasino-backend-timestamp-zone-utc-2.md).

### Brief 2 — Docs/QA (oglasino-docs)

1. This spec written.
2. `decisions.md` — close-out entry + amend the sweeper-timing note (its premise inverts:
   the flip makes "03:00 UTC" true).
3. `issues.md` 2026-06-06 `BaseEntity` timestamp entry → `fixed`, with the one-genuine-skew-site
   correction recorded.
4. New `issues.md` entry — `ProductAudit` has no application writer (medium; out of scope here,
   separate triage).
5. `state.md` — feature block flipped to `shipped (code)` / `verifying`.

## Trust boundary (Part 11)

Every timestamp here is server-generated (Hibernate, `Instant.now()`, or Postgres `DEFAULT now()`).
No reader consumes a client-supplied timestamp. No trust-boundary concern.

## Definition of done

- `Dockerfile` flipped; backend boots; `./mvnw test` green. **Done.**
- Backend own-repo doc wording matches UTC reality (same session as the flip). **Done.**
- Docs/QA config-file edits applied; `issues.md` 2026-06-06 entry `fixed`; `ProductAudit`
  entry logged; sweeper note amended; `state.md` updated. **Done.**
- **Stage boot spot-check:** stamped one product, read `created_at`, confirmed UTC
  wall-clock not Belgrade. **Done** (Igor confirmed 2026-06-06).
