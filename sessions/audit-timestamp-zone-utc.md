# Audit — timestamp-zone-utc (oglasino-backend)

**Repo:** oglasino-backend · **Branch:** dev · **Mode:** READ-ONLY (Phase 2 audit)
**Date:** 2026-06-06
**Scope:** Confirm the blast radius of flipping the container timezone from `Europe/Belgrade`
to `Etc/UTC` BEFORE any change is made. App is pre-production: schema rebuilds from `V1`
on every reset, no stored data to migrate.

---

## Headline findings (read this first)

1. **The TZ is set in exactly one place** — `Dockerfile:13` `ENV TZ=Europe/Belgrade`. No
   compose file, JVM flag, `application*.yaml` property, or `TimeZone.setDefault()` overrides
   or duplicates it. **There is NO `hibernate.jdbc.time_zone` anywhere** — flipping the
   container TZ alone fully changes what `@CreationTimestamp`/`@UpdateTimestamp` write.
   Uniform across all containerized envs (single Dockerfile; stage/prod/dev all build from it).

2. **The issues.md list of "four call sites" overstates the blast radius.** Only **one**
   site genuinely reads a `BaseEntity` `LocalDateTime` and mis-labels it as UTC
   (`DocumentProductConverter:76`). Its display counterpart (`ProductDetailsConverter:78`)
   reads the *derived* ES `Instant` and is the exact inverse — they flip together. The other
   two named sites — **`EmailDateFormatter` and `ProductIndexer` — do NOT read a `BaseEntity`
   timestamp at all** and are completely immune to the flip (both operate on `Instant`s, which
   are absolute moments). Details in Section 2. This is the most important correction the
   audit produces and is flagged for Mastermind.

3. **No double-correction risk anywhere.** `AppVersionAdminDTO.updatedAt`
   (`DefaultAppVersionService:99`) uses `ZoneId.systemDefault()` — it is correct both before
   and after the flip and is NOT double-corrected. Confirmed per the brief's explicit ask.

4. **Every `timestamp with time zone` column / `Instant` field is TZ-flip-immune.** All the
   user-deletion, audit, messaging-cleanup, lock, and `incident_log.occurred_at` timestamps
   are `Instant`s produced by `Instant.now()` (absolute) or DB `DEFAULT now()` (DB-clock).
   Only `timestamp without time zone` columns (`created_at`/`updated_at`/`changed_at`/
   `last_base_site_change`), written from a JVM `LocalDateTime`, change meaning under the flip —
   and pre-production that change has no migration cost.

**Trust-boundary check (Part 11):** every timestamp examined is server-generated — Hibernate
`@CreationTimestamp`/`@UpdateTimestamp`, explicit `Instant.now()`/`LocalDateTime.now()` in
service code, or Postgres `DEFAULT now()`. **None of these readers consume a client-supplied
timestamp.** No trust-boundary concern in this feature.

---

## 1. TZ setting — where and how uniform

| Location | file:line | Value | Notes |
| --- | --- | --- | --- |
| Container/process TZ | `Dockerfile:13` | `ENV TZ=Europe/Belgrade` | **The only place the runtime zone is set.** Drives `ZoneId.systemDefault()` for the whole JVM. |
| (dependabot scheduling) | `.github/dependabot.yml:11,33` | `timezone: "Europe/Belgrade"` | CI-only; irrelevant to runtime. Listed for completeness. |

- **JVM flags:** `Dockerfile:20-26` `CMD java …` — no `-Duser.timezone`. Stage compose
  `JAVA_TOOL_OPTIONS` (`infra/docker-compose-stage.yml:19`) is `-Xmx…`/`MaxRAMPercentage` only.
- **docker-compose:** none of `infra/docker-compose.yml` (prod), `docker-compose-stage.yml`,
  `docker-compose.local.yml`, `docker-compose.es.yml` sets `TZ` on any service. Confirmed by
  `grep -rn "TZ" infra/docker-compose*.yml` → no matches.
- **`application*.yaml`:** no `spring.jpa.properties.hibernate.jdbc.time_zone`, no
  `spring.jackson.time-zone`, no `user.timezone`. The `spring.jpa.properties.hibernate.jdbc`
  trees (dev `:43-44`, stage `:54-55`, prod `:51-52`) contain only `batch_size` — no
  `time_zone`. Confirmed by repo-wide grep returning nothing.
- **`TimeZone.setDefault()`:** none in `src/main`. (The `setDefault*` grep hits are unrelated
  bean setters: `setDefaultLanguage`/`setDefaultCurrency`/`setDefaultRequestConfig`.)

**Uniformity:** dev / stage / prod all run the same image built from the single `Dockerfile`,
so all inherit `TZ=Europe/Belgrade` identically — **no divergence**. (Local non-Docker runs via
IDE/Maven inherit the developer's host zone; not a deployed concern.)

**CRITICAL answer:** there is **no existing `hibernate.jdbc.time_zone`** pinned anywhere.
Flipping `ENV TZ` to `Etc/UTC` is sufficient: with no jdbc time-zone override, Hibernate uses
the JVM default zone for `LocalDateTime` ↔ `timestamp without time zone`, so the flip makes
`@CreationTimestamp`/`@UpdateTimestamp` write UTC wall-clock. **Recommendation noted but out of
scope:** Option A (the planned fix) is clean given the absence of a jdbc time-zone pin.

---

## 2. BaseEntity timestamp readers — the full set

`BaseEntity` (`entity/BaseEntity.java:26-30`) declares `createdAt` (`@CreationTimestamp`) and
`updatedAt` (`@UpdateTimestamp`) as `LocalDateTime`. Both back `timestamp without time zone`
columns (Section 3). Below is **every** zone-applying site, classified for the post-UTC-flip world.

### 2.1 Genuine `BaseEntity` `LocalDateTime` readers that apply a zone

| Site | file:line | Code | Pre-flip | Post-flip |
| --- | --- | --- | --- | --- |
| `DocumentProductConverter` | `:76` | `source.getCreatedAt().toInstant(ZoneOffset.UTC)` | **STILL-WRONG** — `source` is `Product` (BaseEntity); `getCreatedAt()` is the Belgrade wall-clock `LocalDateTime`. Re-labelling it as UTC produces an ES `Instant` skewed by the offset. | **CORRECT** — `LocalDateTime` now holds UTC wall-clock, so `toInstant(UTC)` yields the true UTC `Instant`. This is the one site Option A actually fixes. |
| `DefaultAppVersionService` (feeds `AppVersionAdminDTO.updatedAt`) | `:99` | `updatedAt.atZone(ZoneId.systemDefault()).toInstant()` | **CORRECT** — interprets the `LocalDateTime` with the *same* zone that wrote it (Belgrade), recovering the true moment. | **CORRECT, NOT double-corrected** — after the flip `systemDefault()` becomes UTC and the value was written as UTC wall-clock, so it still recovers the true moment. The `systemDefault()` choice is self-correcting under any container zone. |

### 2.2 Reader of the *derived* ES `Instant` (inverse of `DocumentProductConverter`)

| Site | file:line | Code | Pre-flip | Post-flip |
| --- | --- | --- | --- | --- |
| `ProductDetailsConverter` | `:78` | `formatDateTime(LocalDateTime.ofInstant(source.getCreatedAt(), ZoneOffset.UTC))` | `source` is `ProductDocument`; `getCreatedAt()` is the ES `Instant` (`ProductDocument.java:39`, type `Instant`), **not** a `BaseEntity` value. This is the exact inverse of `:76`, so the round-trip recovers the original Belgrade wall-clock numbers and formats them as `dd/MM/yyyy` (`DateTimeUtil`). Net effect: displays the Belgrade-local date. | **CORRECT/consistent, NOT double-corrected** — it inverts the now-true-UTC `Instant`, displaying the UTC date. It moves in lock-step with `:76`; no independent fix needed. |

### 2.3 Named sites that are NOT `BaseEntity` readers (issues.md mischaracterization)

| Site | file:line | What it actually does | Effect of flip |
| --- | --- | --- | --- |
| `ProductIndexer` | `:87` (used at `:202`) | `DateTimeFormatter.ofPattern("yyyyMMddHHmmss").withZone(ZoneOffset.UTC)` formats **`Instant.now()`** into a *physical index name* suffix (`products_<ts>`). Reads no entity timestamp. | **UNAFFECTED** — `Instant.now()` is absolute and the formatter pins UTC explicitly, independent of the container zone. Index-name digits are identical before/after. |
| `EmailDateFormatter` | `:41` | `.withZone(ZoneOffset.UTC)` formatting an **`Instant`** argument. Its only callers pass `req.getRequestedAt()` / `req.getScheduledDeletionAt()` (`UserDeletionScheduledJobs.java:156-157`, `UserLifecycleEmailEventListener.java:63-64`), which are `Instant` fields computed from `Instant.now()` (`DefaultUserDeletionService.java:119-120,129-130`). Reads no `BaseEntity` `LocalDateTime`. | **UNAFFECTED** — `Instant` source (absolute) + fixed-UTC render. Email dates are correct today and stay correct. |

**Conclusion for Section 2:** the real skew is a single site (`DocumentProductConverter:76`) plus
its paired display inverse (`ProductDetailsConverter:78`). `AppVersionAdminDTO` is already correct
and immune to double-correction; `EmailDateFormatter` and `ProductIndexer` are immune and were
misattributed in issues.md. **There is no double-correction risk anywhere under Option A.**

*Sort note:* the ES `ProductDocument.createdAt` `Instant` is written with one consistent
conversion for every product, so relative sort ordering is preserved both before and after the
flip. No ES range filter compares `createdAt` to a true-UTC "now" (none found), so the only
observable pre-flip defect is the *displayed* date (`:78`) and the *absolute* stored `Instant`
being offset — both resolved by the flip.

---

## 3. Non-BaseEntity timestamp columns

Schema split is clean (`V1__init_schema.sql`):

- **`timestamp without time zone` (LocalDateTime, JVM-stamped → AFFECTED by flip):**
  - `created_at` / `updated_at` on every `BaseEntity` table (≈30 tables; e.g. lines 51/53, 69/73 …).
    Produced by Hibernate `@CreationTimestamp`/`@UpdateTimestamp` using the JVM default zone.
  - `user.last_base_site_change` (`:602`) — set by `DefaultUserFacade.java:247`
    `LocalDateTime.now()` (JVM zone). See Section 4.
  - `product_audit.changed_at` (`:373`, NOT NULL) — `ProductAudit.changedAt` is a `LocalDateTime`,
    but **no application code writes it** (the entity is only ever *deleted*, via
    `ProductAuditRepository.deleteByUserId` from `DefaultUserDeletionService.java:364`; `grep
    setChangedAt` → setter definition only, no caller; no `new ProductAudit`). So nothing JVM-stamps
    this column today. Flagged as an adjacent observation (Section "For Mastermind").
  - `product.updated_at` is additionally **manually** stamped in one place:
    `DefaultProductService.java:247` `product.setUpdatedAt(LocalDateTime.now())` (JVM zone) — same
    affected-but-consistent behavior as `@UpdateTimestamp`.

  All of these shift their stored wall-clock by the offset under the flip, but because the app is
  pre-production (V1 rebuild, no rows) there is **no migration cost** and all readers that compare
  them do so against same-zone values (Section 4).

- **`timestamp with time zone` (Instant / DB-clock → NOT AFFECTED by flip):**
  - JVM-stamped `Instant`s via `Instant.now()`: `user_deletion_request.requested_at`/
    `scheduled_deletion_at`/`cancelled_at`/`actual_deletion_at`/`retention_until` (`:1891-1895`,
    set in `DefaultUserDeletionService`), `user_deletion_audit_log.*` (`:1922-1934`, incl.
    `reminder_sent_at` set at `UserDeletionScheduledJobs.java:161`), `banned_user_audit.retention_until`/
    `reblock_notified_at` (`:1838,1849`, `reblock_notified_at` set at `DefaultUserAuditService.java:108`),
    `user_admin_action_audit.retention_until` (`:1876`), `user_deletion_lock.locked_at`/`expires_at`
    (`:1910-1911`, set `DefaultUserDeletionService.java:535`), `pending_chat_cleanup.last_attempt_at`
    (`:2058`, set `DefaultMessagingCleanupService.java:114`), `messaging_cleanup_lock.locked_at`/
    `expires_at` (`:2085-2086`), `incident_log.occurred_at` (`:2109`, set
    `DatabaseHealthMonitor.java:238` `Instant.ofEpochMilli(nowMs)`).
    `Instant.now()` / `Instant.ofEpochMilli(System.currentTimeMillis())` return the same absolute
    moment regardless of `TZ` → **flip-immune**.
  - DB-clock via `DEFAULT now()`: `banned_at` (`:1837`), `performed_at` (`:1875`), `requested_at`
    (`:1927`), `hard_deleted_at` (`:2057`), `locked_at` (`:1911,2085`), `incident_log.occurred_at`
    default (`:2109`). `now()` is evaluated by the **Postgres** server (DigitalOcean managed PG, a
    separate process), stored as an absolute `timestamptz` → independent of the app container's TZ.

**Answer to the brief's distinction:** the JVM stamps `Instant`s into `timestamptz` columns
(flip-immune) and `LocalDateTime`s into `timestamp`-without-tz columns (flip-affected but
migration-free pre-prod); DB-defaulted `now()` columns are DB-clock and a separate concern the
container flip does not touch. `IncidentLog.occurred_at` is the `Instant` case (semantic poll-snapshot
time), confirmed flip-immune.

---

## 4. now()-dependent business logic

| Site | file:line | What it decides | Effect of flip on the *result* |
| --- | --- | --- | --- |
| 6-month base-site-change gate | `DefaultUserFacade.java:247` (write `LocalDateTime.now()`) + `:261` (`getLastBaseSiteChange().isBefore(LocalDateTime.now().minusMonths(6))`) | Whether a user may change base site again. | **No change to correctness.** Both the stored value and the comparison `now` are `LocalDateTime` in the JVM zone — they shift together. Pre-prod there are no pre-flip rows to compare against post-flip `now`. A 6-month window is unaffected by a ≤2h wall-clock relabel. |
| Year-of-manufacture filter bound | `DefaultProductService.java:801` `filterData.getSelectedRangeValue() > LocalDate.now().getYear()` | Rejects future manufacture years. | **Effectively unchanged.** `getYear()` only differs between Belgrade and UTC in the ~1–2h window straddling Dec 31 / Jan 1. Cosmetic edge, not a real-world correctness issue. |
| Deletion grace / retention math | `DefaultUserDeletionService.java:119-120,144-145,508`; `DefaultUserAuditService.java:55-64,80-88` | `scheduledDeletionAt = now+graceDays`; `retentionUntil = …+retention`. | **No change.** `now` is `Instant.now()` (absolute); `.atOffset(ZoneOffset.UTC).plusDays/Months(…).toInstant()` is constant-offset arithmetic → the resulting `Instant` is identical regardless of container TZ. |
| Image grace-period / age cutoffs | `ProductImagesRemovalJob.java:66,110`; `DefaultImageDeletionFacade.java:93`; `ChatImagesRemovalJob.java:51`; `DefaultProductService.java:624` | "older than N hours/days" sweeps. | **No change.** All use `Instant.now()` (absolute) and `Duration`/`ChronoUnit` math. |
| DB-overload incident timing | `DatabaseHealthMonitor.java` (`System.currentTimeMillis()`, `Instant.ofEpochMilli`); `MaintenanceAutoTripService.java:149`; window/rate math | Pressure-window and rate computations; `occurred_at`. | **No change.** Epoch-millis / `Instant` based — absolute. |
| Deletion-lock / reminder windows | `UserDeletionScheduledJobs.java:66,85,131,161,243`; `*.existsActiveLockForUser(…, Instant.now())` | Lock-active checks, reminder eligibility. | **No change.** `Instant.now()` compared to `timestamptz` columns — both absolute. |
| Firestore trust-review window | `DefaultTrustReviewService.java:62,68-70,85` | Firestore `createdAt` range query. | **No change.** Operates on Firestore `Timestamp`s (client/Firestore-written), independent of the JVM zone. |
| Auth recency | `UserController.java:75` `Instant.now().getEpochSecond() - authTimeSeconds` | Token freshness. | **No change.** Epoch seconds — absolute. |

**Net:** no `now()`-dependent **business decision** changes its computed result under the flip.
The only `LocalDate.now().getYear()` site (`:801`) has a theoretical New-Year-midnight edge that is
immaterial. (Cron *timing* shifts are Section 5, already accepted.)

---

## 5. @Scheduled cron inventory (timing intent already decided: UTC is fine)

**None set a `zone=` attribute** (`grep` for `zone` in `@Scheduled` context → no matches). After
the flip each cron's clock-hour is interpreted as UTC, so the listed wall-clock simply *becomes*
UTC (absolute firing instant moves +1h winter / +2h summer vs. today).

| Job | file:line | Expression | Source | `zone=` | New effective time |
| --- | --- | --- | --- | --- | --- |
| `ProductImagesRemovalJob.sweep` | `:57` | `0 0 3 * * *` | `app.images.sweeper.cron` (yaml) | none | **03:00 UTC** daily |
| `ChatImagesRemovalJob.removeOldChatImages` | `:46` | `0 0 3 * * SUN` | `app.images.chat.removal` (yaml) | none | **Sun 03:00 UTC** |
| `UserDeletionScheduledJobs.processScheduledDeletions` | `:62` | `0 0 2 * * *` | `user.deletion.hard.delete.cron` (yaml) | none | **02:00 UTC** daily |
| `…​.sendDeletionReminders` | `:129` | `0 0 13 * * *` | `user.deletion.reminder.cron` (yaml) | none | **13:00 UTC** daily |
| `…​.reconcileFirebaseUsers` | `:187` | `0 0 3 * * SUN` | `user.deletion.firebase.reconciliation.cron` (yaml) | none | **Sun 03:00 UTC** |
| `…​.purgeExpiredAuditRecords` | `:241` | `0 0 4 * * SUN` | `user.deletion.audit.purge.cron` (yaml) | none | **Sun 04:00 UTC** |
| `ProductRemovalJob.removeOldDeletedProducts` | `:21` | `0 0 2 ? * SUN` | hardcoded | none | **Sun 02:00 UTC** |
| `ProductBaseCurrencyUpdater.updateBasePrices` | `:39` | `0 0 3 * * *` | hardcoded | none | **03:00 UTC** daily |
| `DefaultMessagingCleanupService.runCleanup` | `:82` | `0 0 3 ? * SUN` | `messaging.cleanup.cron` (yaml) | none | **Sun 03:00 UTC** |

Interval-based (no wall-clock; flip-irrelevant): `DefaultScheduledRedisFlushService.periodicFlush`
(`:26`, `fixedDelayString ${app.views.flush-delay-ms}`=20000ms), `DatabaseHealthMonitor.poll`
(`:115`, `fixedRateString ${monitor.poll.interval.seconds:2}000`), `DefaultBaseCurrencyService.updateBaseCurrency`
(`:59`, `fixedRate 86400000`).

**Correctness-vs-timing:** no cron's *correctness* depends on the zone. Each job recomputes its own
cutoffs internally from `Instant.now()` (absolute), so only the firing wall-clock shifts. Confirmed —
none rely on the zone for anything but schedule timing.

---

## 6. Docs that state a wall-clock time (docs-sync targets on the flip)

These go stale (or already are) when the flip lands; listed for the Docs/QA sync task.

**Code comments / javadoc:**
- `images/job/ProductImagesRemovalJob.java:26` — "(default 03:00 UTC)" — *currently inaccurate*
  (fires 03:00 Belgrade today); *becomes accurate* after the flip.
- `properties/ImageProperties.java:129` — "Default daily 03:00 UTC" — same as above.
- `jobs/ProductBaseCurrencyUpdater.java:38` — "Runs every day at 03:00 AM server time".
- `jobs/ProductRemovalJob.java:20` — "Runs every Sunday at 02:00 AM".
- `admin/dto/AppVersionAdminDTO.java:13` — references "`TZ=Europe/Belgrade`" — **becomes stale**
  (the whole point of the explicit-zone conversion); the value stays correct but the comment's
  premise changes.
- `admin/service/impl/DefaultAppVersionService.java:94-96` — "the same zone that wrote it … regardless
  of which zone the deployment runs under" — stays accurate (zone-agnostic by design); no edit needed
  but noted as zone-related.
- `util/EmailDateFormatter.java:20-21` — "Dates are rendered in UTC, matching how
  `scheduled_deletion_at` is computed" — stays accurate.
- `elasticsearch/.../ProductIndexer.java:85,50` — "UTC" index-name suffix / "every wall-clock instant"
  — stays accurate (explicit-UTC formatter).

**`application*.yaml` comments (dev/stage/prod, identical):**
- `…-dev.yaml:234`, `…-stage.yaml:258`, `…-prod.yaml:247` — chat sweep "Sundays at 03:00 (server tz)".
- `…-dev.yaml:265,267`, stage `:290,292`, prod `:276,278` — "02:00 server-local" / "13:00 server-local".
- `…-dev.yaml:275`, stage `:300`, prod `:286` — messaging cleanup "Sundays at 03:00 UTC" — currently
  inaccurate (03:00 Belgrade); becomes accurate after flip.

**Repo `docs/` (backend-owned):**
- `docs/16-image-pipeline.md:364` — "Daily at 03:00 UTC (`app.images.sweeper.cron`)".
- `docs/16-image-pipeline.md:388` — "ChatImagesRemovalJob | Sundays 03:00".
- `docs/16-image-pipeline.md:389` — "ProductImagesRemovalJob | Daily 03:00 UTC".

**Config-file (`oglasino-docs`) note — not edited here:** `decisions.md:684` already records that the
sweeper "fires at 03:00 **Europe/Belgrade** … the '03:00 UTC' wording in the spec and code javadocs
is inaccurate for this deployment." The flip *resolves* that inaccuracy (makes 03:00-UTC wording
true). The `issues.md` 2026-06-06 entry's enumeration of the "four call sites" needs the Section 2
correction (only `DocumentProductConverter` is a genuine skew site; `EmailDateFormatter` and
`ProductIndexer` are flip-immune non-readers). Drafted for Mastermind below — not applied (read-only).

---

## Adjacent observations (Part 4b — flagged, not fixed)

- **`ProductAudit` has no application writer.** `entity/ProductAudit.java` (`changed_at` NOT NULL,
  `LocalDateTime`) is only ever deleted (`ProductAuditRepository.deleteByUserId`); no code calls
  `setChangedAt` or constructs the entity. Either a DB trigger populates it (none found in `V1`) or
  the audit-write path is currently dead. **Severity: medium** (a NOT NULL audit table with no writer
  is a latent contract gap; also means its `changed_at` zone question is moot today). Out of scope —
  not fixed.
- **`DefaultProductService.java:247` manually stamps `product.setUpdatedAt(LocalDateTime.now())`**
  alongside `@UpdateTimestamp` on the same field. Harmless (same zone, same semantics) but redundant
  with Hibernate's auto-stamp. **Severity: low.** Out of scope — not fixed.
