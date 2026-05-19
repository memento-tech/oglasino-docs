# Session — User deletion (backend), Phase 7

Date: 2026-05-17
Repo: oglasino-backend
Branch: dev
Feature: user-deletion
Phase: 7 (scheduled jobs + folded cleanups)
Slug-sequence: 4th session for this slug

## Status

**Phase 7 complete.** CHECKPOINT 6 cleared. Only Phase 8 (comprehensive tests) remains
before the feature is feature-complete.

## Implemented

- **`UserDeletionScheduledJobs`** in `com.memento.tech.oglasino.jobs` (alongside
  `ProductRemovalJob`). Three `@Scheduled` methods reading their cron from
  `application*.yaml`:
  - `processScheduledDeletions()` — daily 02:00; one page of due rows per tick (batch size
    config-driven via `user.deletion.hard.delete.batch.size`); per-row try/catch with
    counters logged at the end; skips locked users, cleans up orphan request rows.
  - `reconcileFirebaseUsers()` — Sundays 03:00; honours the
    `user.deletion.firebase.reconciliation.enabled` flag (off-by-default config so dev does
    not ping Firebase weekly); paged walk of ACTIVE users; orphan check + immediate
    deletion via `firebase_cascade`.
  - `purgeExpiredAuditRecords()` — Sundays 04:00; two delete statements (audit log + ban-
    hash). FK CASCADE on `report.banned_user_audit_id` deletes linked reports
    automatically.
- **Yaml cron entries** added to `application-dev.yaml`, `application-stage.yaml`,
  `application-prod.yaml` with a header comment explaining why these live in yaml rather
  than the configuration table (`@Scheduled` resolves at bean construction).

### Folded-in cleanups

- **7.2 — Replaced `findAll()` linear scans (DONE).** `DefaultUserDeletionService` had two
  call sites (`cancelDeletionOnLogin` + `ensureAuditLogRow`) walking
  `auditLogRepository.findAll()` to find the open row for a user hash. Replaced both with
  a derived-name method on `UserDeletionAuditLogRepository`:
  `findFirstByUserIdHashAndActualDeletionAtIsNullAndCancelledAtIsNullOrderByCreatedAtDesc`.
  Added a matching `idx_udal_user_id_hash` btree to V1 (the existing indexes covered
  `email_hash` and `retention_until` only — `user_id_hash` lookups would have been
  sequential scans). Also added the corresponding `@Index` to the `UserDeletionAuditLog`
  entity so JPA-side reflects the schema.
- **7.3 — Dropped duplicate `idx_udl_user_active` (DONE).** Removed both the
  `CREATE INDEX idx_udl_user_active …` block from V1 (and its multi-line comment) and the
  `@Index(name = "idx_udl_user_active", columnList = "user_id")` annotation from
  `UserDeletionLock`. The `UNIQUE` constraint on `user_id` already creates a btree that
  `existsActiveLockForUser` uses; the duplicate index was dead weight. Added a short comment
  on the entity's `@Table` and on V1 explaining the rationale.
- **7.4 — Removed dead `deleteCompletedOlderThan` (DONE).** Dropped the method from
  `UserDeletionRequestRepository` (and the `LocalDateTime` + jakarta `@Transactional` +
  `@Modifying` imports it required). `purgeExpiredAuditRecords` shipped as the two-query
  version per the brief, so there was no call site to remove. Added an inline comment in
  the repository explaining why no purge is needed (FK cascade + Mastermind's Option B).

## Files touched

| File | Δ | Notes |
|------|---|-------|
| `jobs/UserDeletionScheduledJobs.java` | +175 (new) | three `@Scheduled` methods |
| `application-dev.yaml` | +18 | top-level `user:` block with three cron keys |
| `application-stage.yaml` | +18 | same |
| `application-prod.yaml` | +18 | same |
| `repository/UserDeletionAuditLogRepository.java` | +15 net | derived-name method + javadoc |
| `repository/UserDeletionRequestRepository.java` | -14 net | dead method removed |
| `entity/UserDeletionAuditLog.java` | +1 | `idx_udal_user_id_hash` index annotation |
| `entity/UserDeletionLock.java` | -3 net | `@Index` annotation removed + explanatory comment |
| `service/impl/DefaultUserDeletionService.java` | -4 net | two `findAll().stream()` walks → derived call |
| `db/migration/V1__init_schema.sql` | net -9 | dropped `idx_udl_user_active` + comment block; added `idx_udal_user_id_hash` + short comment |

Totals: 1 new file (the job class) + 9 modified.

## Tests

- `./mvnw spotless:check` — pass (formatter touched the import order of two files and
  reflowed two javadoc paragraphs; no manual fixups needed).
- `./mvnw test` — **360 tests, 0 failures, 0 errors.**

No new tests added in this phase; Phase 8 is the dedicated test phase.

## Bootstrap verification (V1 schema)

I did NOT run `mvn flyway:clean` or rebootstrap the local DB. CLAUDE.md's hard rules
prohibit destructive DB operations, and the brief's note ("Igor will repeat") confirms
schema verification belongs to Igor's environment. What I did do:

- Sanity-checked V1 by grepping for `idx_udl_user_active` repo-wide — zero references
  remain.
- Confirmed `idx_udal_user_id_hash` appears exactly once in V1 and is positioned with the
  other `user_deletion_audit_log` indexes.
- Counted DDL statements in V1 (208 `CREATE INDEX|CREATE TABLE|ALTER TABLE` lines) as a
  proxy for "I didn't accidentally delete an unrelated index."
- Confirmed `UserDeletionLockRepository.existsActiveLockForUser` query plan: the JPQL
  filters on `userId` and either-null-or-future `expiresAt`. The UNIQUE constraint's btree
  on `user_id` is the only index Postgres needs for the equality predicate; the `expiresAt`
  filter is row-level.

**Igor: please run `mvn flyway:clean && mvn flyway:migrate` (or your usual rebootstrap) to
confirm the index drop + rename land cleanly.** The test suite uses H2 / in-memory paths
where applicable, so V1 changes don't actually run against a real Postgres in
`./mvnw test`.

## Cleanup performed

- Removed unused imports in `UserDeletionRequestRepository.java` (`LocalDateTime`,
  `jakarta.transaction.Transactional`, `Modifying`) that became orphaned when
  `deleteCompletedOlderThan` was dropped.
- No commented-out code, no debug `System.out`, no `TODO`/`FIXME` introduced.

## Obsoleted by this session

- `UserDeletionRequestRepository.deleteCompletedOlderThan` — dead method, removed.
- `idx_udl_user_active` (V1 SQL + JPA annotation) — duplicate of the UNIQUE-implied btree,
  removed.
- Two `auditLogRepository.findAll().stream()` walks in `DefaultUserDeletionService` —
  replaced with a single indexed lookup.

## Known gaps / TODOs

- **Phase 8 work outstanding.** Comprehensive tests across Phases 1-7 (unit, repository,
  service, controller, scheduled-job) per the original brief's §8.
- The brief's `getRequiredBooleanConfig` did not exist on `ConfigurationService` — I used
  `getBooleanConfig` instead (defaults to `false` on missing, which is the safe choice for
  the firebase-reconciliation enabled flag). Phase 3 seed sets it to `'true'` explicitly so
  the default never applies in normal operation. Flagged below for "For Mastermind."

## For Mastermind

1. **`getBooleanConfig` vs `getRequiredBooleanConfig`.** Spec §9.2 and the brief both
   reference `getRequiredBooleanConfig` but the `ConfigurationService` interface only
   exposes `getBooleanConfig` (which silently treats missing as `false`). I went with
   `getBooleanConfig` because (a) it exists, (b) silent-false is the safer failure mode
   for the reconciliation toggle than throwing at runtime in a cron tick, and (c) Phase 3
   seeded the key explicitly so the default-path never fires in normal operation. If
   Mastermind wants the required-config semantics (boot-time audit), the lift is small:
   add `boolean getRequiredBooleanConfig(String)` to the interface, audit it in
   `ContentValidationConfig.auditRequiredConfig`, swap the call site. Not in scope for
   this session.

2. **`reconcileFirebaseUsers` and `markForImmediateDeletion` reason string.** The spec
   uses the literal string `"firebase_cascade"` as the trigger constant inside the service
   and also as the `reason` argument passed from the job. The service internally maps to
   `TRIGGER_FIREBASE_CASCADE` (same value). Passing the literal as a reason is fine —
   `markForImmediateDeletion` ignores its `reason` parameter today (the implementation
   always uses `TRIGGER_FIREBASE_CASCADE` for the trigger column and the audit log doesn't
   store a free-text reason). The `reason` parameter is effectively a doc string. If the
   audit log later grows a `reason` column, this is the wiring it'd use. Not a bug — flag
   for awareness only.

3. **`@Scheduled` cron at three yaml locations.** The codebase has no base `application.yaml`
   — every profile is standalone. I added the same `user.deletion.*` cron block to all
   three profile files (dev/stage/prod). If a base `application.yaml` is ever introduced
   for cross-profile defaults, these three blocks should consolidate there.

4. **Adjacent observation: `currentUserService` autowire in `AuthController`.** Still
   unused (carried from Phase 6's For Mastermind). Not user-deletion's to fix; flag once
   more so it's not lost.

5. **Adjacent observation: `BannedUserAudit.retentionUntil` set on Instant via
   `now.atOffset(...).plusMonths(...).toInstant()`.** Same OffsetDateTime hop that fixed
   the Phase 5 `Instant.plusMonths` bug. Works correctly, but `Period`/`Duration` cannot
   express variable-length months on a pure `Instant` — this is the right pattern, not a
   smell. Mentioned because it would be the first thing a reviewer flags as "weird Instant
   arithmetic."

6. **Phase 7 trust-boundary audit (Conventions Part 11).** Scheduled jobs have NO trust
   surface — they take no client input, no path variables, no request headers. They read
   from the DB (which is the trust anchor) and act on rows the DB tells them are due.
   `markForImmediateDeletion` accepts a `reason` string that is hardcoded to
   `"firebase_cascade"` at the call site. CHECK across all three jobs. No CRITICAL.

## Conventions check

- **Part 4 (cleanliness)** — confirmed. Dead method removed, dead index removed, two
  linear scans removed, no new TODO/FIXME. Imports cleaned in `UserDeletionRequestRepository`.
- **Part 4a (simplicity)** — confirmed. The job class is field-injected like the existing
  `ProductRemovalJob` (the codebase has no Lombok). The `purgeExpiredAuditRecords` job is
  two queries, not three (after the 7.4 cleanup). The index-rename + drop is the simplest
  possible schema cleanup.
- **Part 4b (adjacent observations)** — flagged in "For Mastermind" §4-§5.
- **Part 5 (session summary)** — this file + a mirror at `last-session.md`.
- **Part 6 (translations)** — no new translation rows in this phase.
- **Part 7 (error contract)** — N/A (no new endpoints in this phase; scheduled jobs do not
  produce wire responses).
- **Part 11 (trust boundaries)** — audited above. CHECK; no CRITICAL.

## Config-file impact

No required edits to `conventions.md`, `decisions.md`, `state.md`, or `issues.md`.

Optional draft for Docs/QA if Mastermind decides item §1 ("introduce
`getRequiredBooleanConfig`") is worth pursuing — that would warrant a `conventions.md`
note alongside the existing `getRequiredConfig` / `getRequiredIntConfig` /
`getRequiredDoubleConfig` family. Not blocking.

## Next checkpoint

**Phase 8 — Comprehensive tests.** All implementation work for the user-deletion feature
is done. Phase 8 is the dedicated test session covering unit, repository, service,
controller, cache-eviction, and end-to-end-with-scheduled-job coverage per the original
brief's §8 checklist.
