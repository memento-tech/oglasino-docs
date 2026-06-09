# Re-audit — timestamp-zone-utc (cold recheck of the as-applied flip)

**Repo:** oglasino-backend
**Branch:** dev
**Mode:** read-only
**Date:** 2026-06-06
**Contract:** `../oglasino-docs/features/timestamp-zone-utc.md`
**Method:** `git diff HEAD` against the committed baseline + full-tree greps. Fresh session — did not perform the original flip; audited cold.

---

## Verdict

**PASS — the as-applied change matches the spec.** No CRITICAL findings. No reader logic moved. No migration, no behavior code, no test added for the flip. The single genuine `BaseEntity`-`LocalDateTime` skew site is re-confirmed and correct post-flip.

One **LOW** doc-consistency observation (a sibling cron comment left zoneless inside an otherwise-UTC-annotated block) and one **audit-hygiene** note (the `dev` working tree commingles this feature with ≥3 unrelated uncommitted features). Neither is a defect in the flip.

---

## 1. The flip — MATCHES spec

`Dockerfile:13`:

```
-ENV TZ=Europe/Belgrade
+ENV TZ=Etc/UTC
```

One line, exactly as the spec's Resolution / Brief-1.1 names. No competing timezone source anywhere:

| Competing source checked | Result |
| --- | --- |
| Compose files (`infra/docker-compose*.yml`, `…-stage.yml`, `…es.yml`, `…local.yml`) | no `TZ` env set in any |
| `hibernate.jdbc.time_zone` / `time-zone` in `src/main/resources` | none |
| JVM `-Duser.timezone` (Dockerfile, `pom.xml`, yaml) | none |
| `TimeZone.setDefault` / `TimeZone.getDefault` in `src/main/java` | none (the `setDefault*` grep hits are `BaseSiteDTO.setDefaultLanguage/Currency` etc. and one `HttpClients.setDefaultRequestConfig` — unrelated) |

The JVM default zone is therefore driven solely by the container `TZ`. The flip is the only lever, as the spec asserts.

---

## 2. No reader code changed — CONFIRMED

`git diff HEAD` is **empty** for both ES converters; the lines are byte-identical to baseline:

- `elasticsearch/converters/DocumentProductConverter.java:76`
  `destination.setCreatedAt(source.getCreatedAt().toInstant(ZoneOffset.UTC));` — unchanged.
- `elasticsearch/converters/ProductDetailsConverter.java:78`
  `formatDateTime(LocalDateTime.ofInstant(source.getCreatedAt(), ZoneOffset.UTC)))` — unchanged.

`DefaultAppVersionService.toInstant` (line 99, `…atZone(ZoneId.systemDefault()).toInstant()`) and `AppVersionAdminDTO.updatedAt` carry **no UTC double-correction** — they read `ZoneId.systemDefault()`, correct under any container zone, exactly as the spec requires. Note these two are **new untracked files** belonging to the concurrent AppVersion-admin feature (see §4 hygiene note), so `git diff HEAD` shows no base for them; their current content was read directly and is zone-correct (the `AppVersionAdminDTO` javadoc explicitly states the container runs `TZ=Etc/UTC` and the service uses the same zone that wrote the value).

No reader logic moved. **No CRITICAL.**

**Path note (naming only):** the brief/spec cite `…/converter/DocumentProductConverter:76`; the files actually live under `…/elasticsearch/converters/`. Line numbers match. No logic concern — flagged only so future readers find the files.

---

## 3. Doc-sync scope — only stale/zoneless wording touched

Every comment edit attributable to the flip, each listed:

| File:line | Before | After |
| --- | --- | --- |
| `jobs/ProductBaseCurrencyUpdater.java:38` | `Runs every day at 03:00 AM server time` | `Runs every day at 03:00 UTC` |
| `jobs/ProductRemovalJob.java:20` | `Runs every Sunday at 02:00 AM` | `Runs every Sunday at 02:00 UTC` |
| `application-{dev,stage,prod}.yaml` (chat removal) | `Sundays at 03:00 (server tz)` | `Sundays at 03:00 (UTC)` |
| `application-{dev,stage,prod}.yaml` (hard.delete) | `daily 02:00 server-local` | `daily 02:00 UTC` |
| `application-{dev,stage,prod}.yaml` (reminder) | `daily 13:00 server-local` | `daily 13:00 UTC` |
| `docs/16-image-pipeline.md:388` | `ChatImagesRemovalJob … Sundays 03:00` | `… Sundays 03:00 UTC` |
| `admin/dto/AppVersionAdminDTO.java` (javadoc, untracked) | n/a (new file) | reads UTC-correct (`TZ=Etc/UTC`) |

All edits are comment/doc-only. No churn of already-correct wording: the `docs/16-image-pipeline.md` table row `ProductImagesRemovalJob … Daily 03:00 UTC` was already UTC and is **left untouched** (verified — it appears as an unchanged context line in the diff).

**Stale wall-clock re-grep** (`Belgrade`, `server time`, `server-local`, `server tz`, bare `AM`): every hit is in `dataJSON/testProducts.json` listing descriptions or `data/location|translations` seed rows — **none in code or cron comments**. The image-job `@Scheduled` sites carry no wall-clock comment (timing lives in the yaml + docs table, both corrected). The four `UserDeletionScheduledJobs` cron javadocs and `MessagingCleanup` describe behavior ("Daily sweep", "Weekly", "deleted tomorrow") with no zone claim — correctly left alone. No stale wall-clock comment remains in code.

**LOW — intra-block zoneless sibling:** in all three `application-*.yaml`, the `user.deletion.audit.purge` comment two lines below the edited pair was left zoneless:
```
audit:
  purge:
    cron: "0 0 4 * * SUN"        # Sundays 04:00 — audit + ban-hash purge
```
It never claimed a wrong zone, so it is outside the brief's flag patterns and was not "stale". But it is now the only zoneless cron comment in a block whose neighbors (`hard.delete`, `reminder`) gained explicit `UTC`. Purely cosmetic consistency; not a correctness issue (the cron itself, `0 0 4 * * SUN`, is in the spec's accepted-shift table as Sun 04:00 UTC). Flagging per the brief's "list each touched comment line; flag any … stale wall-clock comment still uncorrected" — recording it as a consistency nit, not a stale-zone defect.

---

## 4. No migration, no behavior code — CONFIRMED (for the flip)

- **No new Flyway migration.** `src/main/resources/db/migration/` still holds only `V1__init_schema.sql`. `V1` is modified in the working tree, but its entire diff vs HEAD is a **single deletion** — `allow_notifications boolean NOT NULL` from `public.users` — which belongs to the concurrent user-notifications feature, **not** the timezone change (the V1 diff contains zero `timestamp`/`timezone`/`created_at`/`updated_at` lines).
- **No test asserts timezone behavior.** Working-tree test changes (`TestUsersImportService.java`, `testUsers.json`, and the new `AdminAppVersionController*Test`, `DefaultAppVersionServiceTest`, `AppVersionErrorCodeTest`) all belong to concurrent features; none assert TZ/timestamp behavior.
- **No functional code beyond the flip + comments.** The timestamp-attributable diff is exactly: `Dockerfile` (1 line) + `ProductBaseCurrencyUpdater` / `ProductRemovalJob` (1 comment each) + 3 × `application-*.yaml` (comment-only) + `docs/16-image-pipeline.md` (1 line). Matches the spec's Brief-1 named set.

**Audit-hygiene note (not a flip defect):** the `dev` working tree is **not isolated** to this feature — `git status --porcelain` shows 36 changed/untracked paths spanning at least three concurrent uncommitted features (AppVersion admin: new `admin/controller`, `admin/dto`, `admin/service`, `exception/AppVersionErrorCode`, tests, `V1` table; user email/notifications: `User.java`, converters, `ImportUserData`, `testUsers.json`, the `V1` column drop; DB-overload guard per recent commits). The timestamp-zone-utc change cannot be cleanly isolated by `git diff HEAD` for that reason, but each timestamp-attributable hunk was inspected individually and all fall inside the spec's Brief-1 set. Nothing timezone-related lies outside it. Recorded so whoever commits is aware the branch carries multiple intermingled features.

---

## 5. Residual `ZoneOffset.UTC` / `toInstant` sites — "one genuine site" re-confirmed cold

Full inventory of `ZoneOffset.UTC` / `.toInstant(` / `LocalDateTime.ofInstant(` / `.atZone(` / `withZone(` across `src/main/java` (17 hits), classified against `BaseEntity`-`LocalDateTime`:

| Site | Pattern | Source type | Flip-relevant? |
| --- | --- | --- | --- |
| `DocumentProductConverter:76` | `getCreatedAt().toInstant(ZoneOffset.UTC)` | `Product` → `BaseEntity.createdAt` = **`LocalDateTime`** (`BaseEntity.java:28`) | **YES — the single genuine skew site. Correct post-flip.** |
| `ProductDetailsConverter:78` | `LocalDateTime.ofInstant(getCreatedAt(), UTC)` | `ProductDocument.createdAt` = `Instant` (`ProductDocument.java:39`) | display inverse; immune |
| `DefaultAppVersionService:99` | `atZone(ZoneId.systemDefault()).toInstant()` | `BaseEntity.updatedAt` = `LocalDateTime`, but read via `systemDefault()` | zone-agnostic by construction; **not** a skew site |
| `EmailDateFormatter:21` (javadoc), `:41` | `withZone(ZoneOffset.UTC)` | formats an `Instant` | immune |
| `ProductIndexer:87` | `…withZone(ZoneOffset.UTC)` | formats an `Instant` | immune |
| `DefaultUserAuditService:64,88` | `now.atOffset(UTC).plusMonths(...)` | `Instant.now()` arithmetic | immune (absolute) |
| `DefaultUserDeletionService:120,145,508` | `…atOffset(UTC).plusDays(...)` | `Instant` arithmetic | immune (absolute) |
| `DefaultTrustReviewService:85` | `Timestamp.toDate().toInstant()` | Firestore `Timestamp` | immune |

`DocumentProductConverter:76` is the **only** call that converts a `BaseEntity` `LocalDateTime` to an `Instant` via a hard `ZoneOffset.UTC` — pre-flip it re-labelled Belgrade wall-clock as UTC (the skew); post-flip the stored value is already UTC wall-clock, so the label is now true. The original audit's "one genuine site" conclusion holds, verified cold.

---

## Trust boundary (Part 11)

Every timestamp in scope is server-generated — Hibernate `@CreationTimestamp` / `@UpdateTimestamp`, `Instant.now()`, or Postgres `DEFAULT now()`. No reader consumes a client-supplied timestamp; no DTO field carries an inbound time used in any moderation / authorization / state-transition decision. No trust-boundary concern.

---

## Summary of flags

| # | Severity | Finding |
| --- | --- | --- |
| 1 | LOW | `application-*.yaml` `audit.purge` cron comment left zoneless while its block-siblings gained explicit `UTC` — cosmetic consistency nit, not a stale-zone defect. |
| 2 | INFO | Spec/brief cite `converter/…`; files actually under `elasticsearch/converters/…`. Naming only. |
| 3 | INFO (hygiene) | `dev` working tree commingles this feature with ≥3 unrelated uncommitted features; timestamp diff cannot be isolated by `git diff HEAD` but each hunk was inspected and all fall inside the spec's Brief-1 set. |

No CRITICAL, no HIGH, no MEDIUM. The as-applied change conforms to the spec.
