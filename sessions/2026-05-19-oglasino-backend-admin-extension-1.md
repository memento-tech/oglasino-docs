# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-19
**Task:** Admin Extension Backend (Ban-with-Reason, Lock-from-Deletion, Filters, Info) — six endpoint changes, +Mockito tests, translation seed rows, spec-amendment drafts for Mastermind.

## Schema verification findings (Step 0)

**0.1 Ban reason field.** Already present in two places that this session reuses without change:
- `users.ban_reason VARCHAR(500)` — mirrored on the `User` entity (`User.java:48-49`); kept in sync by `saveUser` (which fires the Redis evictions).
- `banned_user_audit.ban_reason VARCHAR(500) NOT NULL` — written by `DefaultUserAuditService.recordBannedUser`; the canonical record of the ban event.
- No `banned_by_admin_id` column on `banned_user_audit` (asymmetric with the lock side, which has `locked_by_admin_id` — see "For Mastermind").

**0.2 user_deletion_locks table.** Confirmed columns: `id`, `user_id` (UNIQUE, indexed via the implicit btree from the UNIQUE constraint), `locked_by_admin_id NOT NULL`, `reason VARCHAR(500) NOT NULL`, `disclosable_reason VARCHAR(500)`, `notified_at`, `locked_at NOT NULL`, `expires_at` (nullable). Data model is **"one active row per user"** — UNIQUE forbids a second concurrent lock; unlock = delete the row. Existing service methods `lockFromDeletion` (upsert) and `unlockFromDeletion` (no-op when absent) ship from the user-deletion feature; their callers' tests assert these semantics so the controller does the strict pre-checks rather than tightening the service contract.

**0.3 Existing endpoints.** Located in `admin/controller/UsersController.java` (not a separate `AdminUserController`). The current shape that this session replaces:
- `GET /api/secure/admin/users/disable/{targetUserId}` — body-less; controller passed `null` to the facade, which defaulted to placeholder string `"admin disabled (no reason provided)"`.
- `GET /api/secure/admin/users/enable/{targetUserId}` — body-less.
- `POST /api/secure/admin/users` with `UsersFilterRequestDTO` body, filter param `disabledOnly` (not `bannedOnly`).
- Authorization: class-level `@PreAuthorize("hasRole('ADMIN')")`; no method-level gates.

**Brief-vs-reality decisions (cleared with Igor on 2026-05-19 before any code):**
1. Convert ban/unban from `GET` to `POST` with JSON body. Old GET shape obsoleted.
2. Drop `bannedByAdminId` from the state-info response (no column exists); lock side keeps `lockedByAdminId`.
3. Log unban reason via SLF4J only (no audit-log table retains unban events; `banned_user_audit` is deleted on unban per spec §3.7). Spec amendment drafted in "For Mastermind."
4. Translation IDs: insert literal `id=100` on every new row per Igor's explicit instruction; Igor will renumber before any seed run. Each block carries a comment header calling this out.
5. RU `user.locked.indicator` uses `Zaperto` (not `Zablokirovan`) so banned and locked stay visually distinct in the row indicators column.

## Implemented

- **Step 1 (ban/unban with reason).** Converted `disable` and `enable` to `POST` with JSON bodies. `AdminBanRequest.reason` is `@NotBlank @Size(max=500)` enforced at the controller boundary; `AdminUnbanRequest.reason` is optional `@Size(max=500)`. The facade now trusts its inputs (no placeholder fallback); the un-ban path logs the optional reason at INFO via SLF4J because there's no schema location for unban-event audit.
- **Step 2 (lock / unlock).** New `POST .../lock-deletion` (with required `AdminLockDeletionRequest.reason`) and `POST .../unlock-deletion` (no body). Controller does the pre-check against `UserDeletionLockRepository.existsActiveLockForUser`: throws `UserAlreadyLockedException` (HTTP 400, `USER_ALREADY_LOCKED`) on lock-when-already-locked and `UserNotLockedException` (HTTP 400, `USER_NOT_LOCKED`) on unlock-when-not-locked. Admin id read from `OglasinoAuthentication.getUserId()` (never from the request body — Part 11 trust boundary). Service-layer methods (already shipped by user-deletion) are reused as-is.
- **Step 3 (pendingDeletion filter).** `UsersFilterRequestDTO.pendingDeletionOnly` added (boolean, default false), with parallel changes to both predicate paths — the DTO's own `getUserSpec()` and the live `DefaultUsersService.buildPredicates`. Filter composes with `disabledOnly` (and every other field) by AND, matching the existing convention.
- **Step 4 (state-info endpoint).** `GET .../{userId}/state-info` returns `UserStateInfoDTO` with derived `currentState` (precedence `BANNED > LOCKED > PENDING_DELETION > ACTIVE`) plus three nullable sub-payloads (`ban`, `deletion`, `lock`). Ban section reads from `banned_user_audit` keyed by `userAuditService.hashEmail(user.email)`, falling back to the `user.banReason` mirror if the audit row is absent; deletion section reads `user_deletion_requests.findByUserId` and derives `daysRemaining` from `Math.max(0, Duration.between(now, scheduledDeletionAt).toDays())`; lock section reads `user_deletion_locks.findByUserId` and filters out expired rows.
- **Step 5 (translation seeds).** Added 39 DIALOG keys + 9 ADMIN_PAGES keys × 4 languages = **192 rows** appended at the end of each namespace section in `0001-data-web-translations-{EN,RS,CNR,RU}.sql`. SR/CNR ijekavian splits applied (`Vreme`/`Vrijeme`, `Zahtev podnet`/`Zahtjev podnijet`). RU `Zaperto` for `user.locked.indicator`. **All rows use literal `id=100`** per Igor's directive — the seed file must be renumbered before any run; both block comments call this out.
- **Step 6 (Mockito tests).** +23 new tests across two classes. `UsersControllerAdminExtensionTest` (new, 16 tests) covers ban/unban/lock/unlock happy + validation + conflict paths, the list filter forwarding, and the state-info JSON shape. `DefaultUsersFacadeTest` (rewritten, 9 tests) covers ban/unban ordering and the full state-info aggregation including precedence and expired-lock filtering.
- **Adjacent fix (HandlerMethodValidationException handler).** Spring 7's `HandlerMethodValidator` emits `HandlerMethodValidationException` for `@Valid @RequestBody` violations on the method-level validation path. The existing `MethodArgumentNotValidException` handler in `GlobalExceptionHandler` didn't catch it; the catch-all `Exception` handler returned 500. Added a sibling handler that mirrors the existing one — same `{errors:[{field, code, translationKey}]}` shape, 400 status, best-effort `translationKey` resolution via `ProductErrorCode.valueOf`. Necessary to make the new validation tests pass and to keep the new endpoints returning 400 in production rather than 500 when the body is malformed.

## Files touched

Modified:
- `src/main/java/com/memento/tech/oglasino/admin/controller/UsersController.java` (+78 / -10)
- `src/main/java/com/memento/tech/oglasino/admin/dto/UsersFilterRequestDTO.java` (+14)
- `src/main/java/com/memento/tech/oglasino/admin/facade/UsersFacade.java` (+19 / -3)
- `src/main/java/com/memento/tech/oglasino/admin/facade/impl/DefaultUsersFacade.java` (+103 / -18)
- `src/main/java/com/memento/tech/oglasino/admin/service/impl/DefaultUsersService.java` (+4)
- `src/main/java/com/memento/tech/oglasino/exception/GlobalExceptionHandler.java` (+26)
- `src/main/resources/data/translations/0001-data-web-translations-EN.sql` (+52)
- `src/main/resources/data/translations/0001-data-web-translations-RS.sql` (+52)
- `src/main/resources/data/translations/0001-data-web-translations-CNR.sql` (+52)
- `src/main/resources/data/translations/0001-data-web-translations-RU.sql` (+55)
- `src/test/java/com/memento/tech/oglasino/admin/facade/impl/DefaultUsersFacadeTest.java` (+200 / -50)

Created:
- `src/main/java/com/memento/tech/oglasino/admin/dto/AdminBanRequest.java`
- `src/main/java/com/memento/tech/oglasino/admin/dto/AdminUnbanRequest.java`
- `src/main/java/com/memento/tech/oglasino/admin/dto/AdminLockDeletionRequest.java`
- `src/main/java/com/memento/tech/oglasino/admin/dto/UserStateInfoDTO.java`
- `src/main/java/com/memento/tech/oglasino/exception/UserAlreadyLockedException.java`
- `src/main/java/com/memento/tech/oglasino/exception/UserNotLockedException.java`
- `src/test/java/com/memento/tech/oglasino/admin/controller/UsersControllerAdminExtensionTest.java`

## Tests

- Ran: `./mvnw test` (full suite)
- Baseline at session start: **446 passed, 0 failed**
- End of session: **469 passed, 0 failed** (+23 net new tests across two classes)
- `./mvnw spotless:check`: clean at session end (one `./mvnw spotless:apply` pass during development to fix the 5 new files' wrapping; re-verified after).
- New tests added:
  - `UsersControllerAdminExtensionTest` (16): disable/enable/lock/unlock happy + validation + conflict, list filter forwarding, state-info JSON.
  - `DefaultUsersFacadeTest` net delta +9: ban happy + trim, unban with-reason + without-reason, state-info ACTIVE / banned / banned-with-fallback / pending-deletion / locked / expired-lock / banned-precedence-over-lock-and-deletion.

## Cleanup performed

- Removed `DEFAULT_BAN_REASON` constant and the null/blank-reason fallback branches from `DefaultUsersFacade.disableUser` — obsoleted because the controller's Jakarta `@NotBlank` is now the validation boundary. The placeholder string had no remaining role.
- Removed `DefaultUsersFacadeTest.disableUserWithNullReasonFallsBackToPlaceholder` and `…WithBlankReasonFallsBackToPlaceholder` — both asserted the obsoleted fallback. Replaced with `disableUserTrimsWhitespaceFromReason` which asserts the new normalised-input behaviour.
- The previous `enableUserClearsFieldsAndOrdersCallsSaveUserAuditFirebase` was updated for the new `enableUser(Long, String)` signature rather than deleted.

## Config-file impact

- `conventions.md`: **no change.** Part 6 Rule 3 (translation-seed numbering / ON-CONFLICT-collision discipline) is touched in spirit by Igor's "id=100 placeholder" directive, but the directive is an explicit one-off, not a convention amendment. No edit drafted.
- `decisions.md`: **no change.**
- `state.md`: **no change.** User-deletion feature already at `backend-stable`; this session is a follow-up Phase 1 intake (admin extension) on the existing feature, not a status flip. The Mastermind chat that owns admin-extension intake can decide whether to log an entry once the frontend lands.
- `issues.md`: **no change.** Adjacent observations from this session are routed below for Mastermind triage, not authored directly.

## Obsoleted by this session

- The body-less `GET /api/secure/admin/users/disable/{userId}` and `GET .../enable/{userId}` endpoints — deleted in this session; replaced with `POST` shapes carrying a JSON body.
- The placeholder default-reason path in `DefaultUsersFacade.disableUser` and `DEFAULT_BAN_REASON` constant — deleted in this session.
- The two `DefaultUsersFacadeTest` fallback-placeholder tests — deleted in this session.
- The frontend brief's expectation that the existing admin disable/enable buttons are body-less GETs — the next frontend session needs to rewrite those calls. Frontend brief is owned by a different agent; this engineer can't update it. Flagged in "For Mastermind."

## Conventions check

- **Part 4 (cleanliness):** confirmed. No commented-out blocks, no unused imports, no debug logging added (only one INFO log line on the un-ban path, fitting the existing SLF4J pattern). Both spotless and test green.
- **Part 4a (simplicity):** confirmed. New exceptions extend the existing `UserDeletionException` hierarchy rather than introducing a parallel admin-exception hierarchy. State-info aggregator lives in the facade (the existing trust-boundary integration point) rather than a new orchestrator. Filter expansion follows the existing `disabledOnly` pattern verbatim.
- **Part 4b (adjacent observations):** flagged in "For Mastermind."
- **Part 5 (session summary):** this file + `.agent/last-session.md` mirror.
- **Part 6 (translations):** Rule 1 namespaces honoured (DIALOG + ADMIN_PAGES). Rule 2 parent/child collision check: `user.info` (ADMIN_PAGES) and `user.info.dialog.*` (DIALOG) are in different namespaces — no collision because the frontend translation library separates namespaces. Rule 3 ID-numbering: violated by directive — Igor explicitly accepted `id=100` placeholders. Flagged.
- **Part 7 (error contract):** confirmed. All new validation errors return 400 with `{errors:[{field, code, translationKey}]}`. Custom business-rule errors (USER_ALREADY_LOCKED, USER_NOT_LOCKED) flow through `UserDeletionException` and reuse the existing handler. `translationKey` is null on the new validation codes because they aren't registered in `ProductErrorCode` — flagged.
- **Part 11 (trust boundaries):** confirmed. Admin id for the lock endpoint is read from `OglasinoAuthentication.getUserId()` (server-derived from `SecurityContextHolder`), never from the request body. Target user id from the path is gated by the class-level `@PreAuthorize("hasRole('ADMIN')")`. All sensitive reads (`banned_user_audit`, `user_deletion_requests`, `user_deletion_locks`) are server-side lookups.

## Known gaps / TODOs

- **Translation IDs are placeholders.** Every new row uses literal `id=100`. The seed file in its current state would only persist one row per language (the last one written) on its way through `ON CONFLICT DO UPDATE`. Igor will renumber before any run — explicit, accepted at 2026-05-19. Block comments call this out at each insertion point.
- **`HttpMessageNotReadableException` not handled.** Sending no body at all (Content-Length: 0) to the new ban / lock endpoints still produces HTTP 500 via the catch-all handler because Spring fails to deserialise the missing body before `@Valid` ever fires. Frontend will always send a body so practical impact is nil; flagged.
- **Validation codes (BAN_REASON_REQUIRED, etc.) have no translation key.** `resolveTranslationKey` returns null for unknown `ProductErrorCode` names, so the wire response carries `translationKey: null`. Frontend handles ban/unban dialog text via DIALOG namespace keys directly, so this is acceptable.
- **`UsersFilterRequestDTO.getUserSpec()` is unused.** The DTO carries a `Specification<User>` builder that no caller invokes (the live path is `DefaultUsersService.buildPredicates`). I updated it in parallel so it stays consistent if a future caller switches over, but the dead-code question stands. Flagged.

## For Mastermind

### Drafted spec amendment — `user-deletion.md` §11 / §3.5 (admin ban / unban contract)

Add or extend a section near §11 / §3.5:

```markdown
### Admin ban / unban — wire contract (added 2026-05-19)

**List.** `POST /api/secure/admin/users` with `UsersFilterRequestDTO` body. Response is a `PageDTO<UserOverviewDTO>` where each row carries `{id, displayName, email, disabled, deletionStatus, lockedFromDeletion, baseSiteOverview}`. `deletionStatus` mirrors `users.deletion_status` (enum: `ACTIVE` | `PENDING_DELETION`). `lockedFromDeletion` is `true` when an active row exists in `user_deletion_locks` for this user — computed per page via a single batch lookup against `UserDeletionLockRepository.findActivelyLockedUserIds`, not via N+1 per-row queries. Filter params include `disabledOnly`, `pendingDeletionOnly`, `displayName`, `email`, `subscription`, `regionId`, `cityId`. The admin Users page row indicators (Banned / Pending deletion / Locked) and the row-level Lock/Unlock icon read from these two new fields without any per-row state-info fetch.

**Ban.** `POST /api/secure/admin/users/{userId}/disable`. Body: `{"reason": "..."}` — required, non-blank, ≤ 500 chars (max-length matches `user.ban_reason` and `banned_user_audit.ban_reason` column widths). Validation failures return HTTP 400 with code `BAN_REASON_REQUIRED` (missing/blank) or `BAN_REASON_TOO_LONG`. On success: `user.disabled=true`, `user.banReason=<trimmed reason>` (persisted via `saveUser` which evicts both Redis caches); `banned_user_audit` row inserted (idempotent on email-hash collision; new inserts also persist `banned_by_admin_id` read from `OglasinoAuthentication`); `firebaseUserService.disableUser` + `revokeRefreshTokens`. Returns 200 with empty body.

**Un-ban.** `POST /api/secure/admin/users/{userId}/enable`. Body: `{"reason": "..."}` — optional. Empty body / missing body / `{}` all accepted. If present, max 500 chars (code `UNBAN_REASON_TOO_LONG`). On success: `user.disabled=false`, `user.banReason=null`; `banned_user_audit` row deleted (cascades to linked reports per `report.banned_user_audit_id ON DELETE CASCADE`); `firebaseUserService.enableUser`. The optional reason is **logged at INFO via SLF4J** for operator trace and **not persisted to any table** — no audit-log table retains unban events with a reason. Returns 200.

**Audit-log gap.** Adding a generic ban/unban audit-log table (e.g. `user_admin_action_audit` with action, target, reason, timestamp, acting admin id) is deferred. The current model is "the absence of a `banned_user_audit` row is the unban record"; this is sufficient for compliance but leaves the operator without a reason trail on the unban side.
```

### Drafted spec amendment — `user-deletion.md` §11 / new §12 (admin lock / unlock contract)

```markdown
### Admin lock-from-deletion / unlock-from-deletion (added 2026-05-19)

**Lock.** `POST /api/secure/admin/users/{userId}/lock-deletion`. Body: `{"reason": "..."}` — required, non-blank, ≤ 500 chars (codes `LOCK_REASON_REQUIRED` / `LOCK_REASON_TOO_LONG`). The controller verifies no active lock exists for this user (`existsActiveLockForUser`) and throws `USER_ALREADY_LOCKED` (HTTP 400, key `user.already.locked`) if one does — preserves the UNIQUE-on-user_id constraint's intent rather than silently upserting. Acting admin id is read server-side from `OglasinoAuthentication`. Persists via `userDeletionService.lockFromDeletion(userId, adminId, reason, null, Optional.empty())`. Returns 200.

**Unlock.** `POST /api/secure/admin/users/{userId}/unlock-deletion`. No body. The controller verifies an active lock exists; throws `USER_NOT_LOCKED` (HTTP 400, key `user.not.locked`) if not. Persists via `userDeletionService.unlockFromDeletion(userId)` (deletes the row). Returns 200.

**Admin state-info popup.** `GET /api/secure/admin/users/{userId}/state-info`. Returns `UserStateInfoDTO`:
{
  "currentState": "ACTIVE" | "PENDING_DELETION" | "LOCKED" | "BANNED",
  "ban":      { "bannedAt", "bannedByAdminId", "reason", "expiresAt" }      // or null
  "deletion": { "requestedAt", "scheduledDeletionAt", "daysRemaining" }     // or null
  "lock":     { "lockedAt", "lockedByAdminId", "reason" }                   // or null
}
Precedence for `currentState` when multiple states coexist: BANNED > LOCKED > PENDING_DELETION > ACTIVE.

**`bannedByAdminId` nullability.** The `banned_user_audit.banned_by_admin_id` column is **nullable** (FK to `users(id)` ON DELETE SET NULL), explicitly to accommodate the spec §3.12 Firebase-orphan-cascade auto-ban call site in `DefaultUserDeletionService.markForImmediateDeletion`, which is system-initiated and has no admin actor. This is asymmetric with `user_deletion_locks.locked_by_admin_id` (NOT NULL) because lock-from-deletion has no system-initiated call site. Frontend renders the "Banned by" field with the actual admin's identity when populated, or the translation key `info.dialog.field.banned.by.system` ("System" / "Sistem" / "Sistema") when null.
```

### Decisions / flags surfaced

1. **~~`bannedByAdminId` is asymmetric vs `lockedByAdminId`.~~** **Resolved 2026-05-19 by the follow-up patch below** (V1 column folded inline; FK ON DELETE SET NULL; field populated in `UserStateInfoDTO.BanInfo`). The column is nullable for the system-initiated §3.12 cascade path, which is asymmetric with the NOT-NULL `locked_by_admin_id` on the lock side — that asymmetry is intentional and now documented in the §11 / §12 spec drafts above.

2. **No audit-log table for un-ban events.** The frontend will display "Banned by / on / reason" in the info popup but no parallel "Unbanned on / by / reason" because there's nowhere to read it from after the `banned_user_audit` row is deleted. Slight operator-trace gap. Mastermind to decide whether to add an audit table (probably feature-sized) or live with logs.

3. **`UsersFilterRequestDTO.getUserSpec()` is dead code.** The DTO ships a `Specification<User>` builder that no caller invokes — the live path is `DefaultUsersService.buildPredicates`, a parallel implementation. I updated `getUserSpec` to stay consistent (per Part 4a "match the surrounding code's style") but the dead-code question stands. **Severity:** low. **File:** `admin/dto/UsersFilterRequestDTO.java`. **I did not delete this because it is out of scope.** Mastermind decision: delete `getUserSpec`, or migrate `DefaultUsersService` to use it (one source of truth)?

4. **`HandlerMethodValidationException` was not handled.** Added a sibling handler in `GlobalExceptionHandler` because Spring 7's `HandlerMethodValidator` emits this for `@Valid @RequestBody` violations on the new method-level path. The existing `MethodArgumentNotValidException` handler stays as-is — it covers the legacy path. **Severity:** medium pre-fix (would have returned 500 instead of 400 on validation failures across all `@Valid @RequestBody`-using endpoints that don't pair with `BindingResult`). Fixed in this session because it directly blocked the brief's Step 6 validation tests; the fix is a clean mirror of an existing handler. Worth Mastermind awareness because the same exception type would have fired for any other controller using `@Valid @RequestBody` without `BindingResult`.

5. **Translation seed IDs are placeholders.** Per Igor's explicit instruction (2026-05-19), every new row uses `id=100`. The block comments call this out, but a future engineer running the seed file unpatched will silently lose 47 of the 48 rows per language (only the last id=100 INSERT survives the `ON CONFLICT DO UPDATE`). Recommend Mastermind chat next session: "fix admin-extension seed IDs" mini-brief before any seed run.

6. **Brief's stated assumption that `bannedOnly` filter exists.** It is actually `disabledOnly`. New filter is `pendingDeletionOnly` matching the existing convention. Frontend brief will need to know that admin filter chip naming follows `*Only` not `bannedOnly`. Flagged.

7. **The existing `lockFromDeletion` Javadoc claims it throws when a lock already exists, but the implementation upserts.** The Javadoc at `UserDeletionService.java:69-77` ("If a lock already exists for this user, throws") doesn't match the implementation in `DefaultUserDeletionService.lockFromDeletion`, which upserts (and is asserted by existing tests `lockFromDeletionUpdatesExistingLockRow`). Either the Javadoc is stale or the implementation is wrong. **Severity:** low — only one admin caller exists today and it does its own pre-check. **File:** `service/UserDeletionService.java:69-77`. **I did not fix this because it is out of scope.** Mastermind decision: tighten the implementation to throw (and update tests), or fix the Javadoc to say "upsert"?

8. **Frontend brief impact.** The frontend brief that runs after this one will need to (a) change the existing disable/enable buttons from GET to POST with body; (b) wire up four new admin dialogs (ban / unban / lock / unlock); (c) wire up the new info popup against `state-info`; (d) wire the new filter chip against `pendingDeletionOnly`. The translation keys are seeded but with placeholder IDs — the frontend chat will need to either wait for Igor to renumber the IDs, or trust that the keys exist by name (the frontend resolves by `namespace + key`, not by id).

---

## Follow-up patch — 2026-05-19 (banned_by_admin_id column)

Closes the asymmetry flagged in "For Mastermind" item 1 and "Decisions / flags surfaced" item 2 above. Igor's directive (round-3 Mastermind, option a) was "add the column inline to V1." During implementation the brief had to be revised mid-session because the spec §3.12 Firebase-orphan-cascade in `DefaultUserDeletionService.markForImmediateDeletion` also calls `recordBannedUser` without an admin actor; a NOT-NULL column would have made that call site fail. Igor's revised instruction (logged in the AskUserQuestion exchange): make the column nullable, FK `ON DELETE SET NULL`, cascade caller passes null, render "System" in the frontend for null values.

### Implemented (patch)

- **V1 schema fold.** `banned_user_audit.banned_by_admin_id BIGINT` (nullable) added with `FK fk_bua_admin → users(id) ON DELETE SET NULL`. The nullable choice is documented inline at the column definition and the FK definition.
- **`BannedUserAudit` entity** gained a nullable `Long bannedByAdminId` field with getter/setter, JPA `@Column(name = "banned_by_admin_id")` (no `nullable = false`).
- **`UserAuditService.recordBannedUser` signature** changed to `recordBannedUser(String email, String banReason, Long adminId)`. New inserts persist `adminId`; the idempotent existing-row branch is unchanged — original admin id is preserved on re-ban.
- **`UsersFacade.disableUser`** signature changed to `disableUser(Long userId, String banReason, Long adminId)`. The controller reads `auth.getUserId()` via `@AuthenticationPrincipal OglasinoAuthentication auth` (already in scope for the lock endpoint; mirrored here) and threads it through. Trust-boundary check: admin id is server-derived, never from the body.
- **`DefaultUserDeletionService.markForImmediateDeletion`** passes `null` to `recordBannedUser` and now carries an inline comment explaining the system-initiated semantics.
- **`UserStateInfoDTO.BanInfo`** gained a nullable `Long bannedByAdminId` field (record component slot 2 of 4). DTO Javadoc rewritten to reflect that both ban and lock sides now carry an admin attribution, with the ban side legitimately null on cascade-inserted rows or legacy rows.
- **Frontend rendering hook.** New translation key `info.dialog.field.banned.by.system` (DIALOG namespace) added to all four 0001 SQL seed files: EN `System`, SR `Sistem`, CNR `Sistem`, RU `Sistema`. Root prefix is `info.*` (not `user.info.*`) specifically to avoid a Conventions Part 6 Rule 2 parent/child collision with the existing `user.info.dialog.field.banned.by` leaf key — i18next would reject the latter as both a string and an object. IDs use the same `id=100` placeholder as the rest of the admin-extension seed batch.

### Files touched (patch)

Modified:
- `src/main/resources/db/migration/V1__init_schema.sql` (+5 / -1 on the table definition; +5 on the FK section)
- `src/main/java/com/memento/tech/oglasino/entity/BannedUserAudit.java` (+15 / -1)
- `src/main/java/com/memento/tech/oglasino/service/UserAuditService.java` (+5 / -1 — Javadoc + signature)
- `src/main/java/com/memento/tech/oglasino/service/impl/DefaultUserAuditService.java` (+2 / -1 — signature + setter call)
- `src/main/java/com/memento/tech/oglasino/service/impl/DefaultUserDeletionService.java` (+2 / -1 — pass null + comment)
- `src/main/java/com/memento/tech/oglasino/admin/facade/UsersFacade.java` (+2 / -1 — signature + Javadoc)
- `src/main/java/com/memento/tech/oglasino/admin/facade/impl/DefaultUsersFacade.java` (+3 / -1 — signature + adminId thread-through + state-info populate)
- `src/main/java/com/memento/tech/oglasino/admin/controller/UsersController.java` (+4 / -2 — add `@AuthenticationPrincipal` + Javadoc)
- `src/main/java/com/memento/tech/oglasino/admin/dto/UserStateInfoDTO.java` (+5 / -4 — field + Javadoc rewrite)
- `src/main/resources/data/translations/0001-data-web-translations-EN.sql` (+4)
- `src/main/resources/data/translations/0001-data-web-translations-RS.sql` (+2)
- `src/main/resources/data/translations/0001-data-web-translations-CNR.sql` (+2)
- `src/main/resources/data/translations/0001-data-web-translations-RU.sql` (+2)
- `src/test/java/com/memento/tech/oglasino/admin/facade/impl/DefaultUsersFacadeTest.java` (+30 / -5 — signature updates + null-admin coverage + adminId-forwarding test)
- `src/test/java/com/memento/tech/oglasino/admin/controller/UsersControllerAdminExtensionTest.java` (+5 / -5 — verify 3-arg signature; JSON `bannedByAdminId` assertion)
- `src/test/java/com/memento/tech/oglasino/service/impl/DefaultUserAuditServiceTest.java` (+25 / -3 — signature updates + system-cascade null-admin test)
- `src/test/java/com/memento/tech/oglasino/service/impl/DefaultUserDeletionServiceTest.java` (+5 / -5 — verify 3-arg signature on cascade tests, assert `null` admin)

### Tests (patch)

- Ran: `./mvnw spotless:check && ./mvnw test`
- Before patch: 469 passed.
- After patch: **472 passed, 0 failed** (+3 net: `disableUserForwardsAdminIdToAuditService` in facade, `stateInfoBanInfoNullAdminIdOnLegacyAuditRow` in facade, `recordBannedUserPersistsNullAdminIdForSystemInitiatedCascade` in audit-service).
- Spotless: clean.

### Cleanup performed (patch)

- None needed.

### Config-file impact (patch)

- `conventions.md`: no change.
- `decisions.md`: no change. (The round-3 Mastermind decision that selected option a is documented in this session summary, not in `decisions.md`. Mastermind may choose to log a brief entry once the frontend lands.)
- `state.md`: no change.
- `issues.md`: no change.

### Obsoleted by this patch

- The original ban-side asymmetry ("no `banned_by_admin_id`") — eliminated by the column add. The §11 / §12 spec amendment drafts above were updated in place to reflect the column existing.
- ~~"For Mastermind" item 1 from this session~~ — resolved by this patch; struck through above.

### Conventions check (patch)

- **Part 4 (cleanliness):** confirmed. No commented-out code, no debug logging beyond the existing INFO line on the un-ban path.
- **Part 4a (simplicity):** confirmed. The column is nullable for the smallest possible reason — exactly one existing call site cannot supply an admin id. No new abstractions; signatures grow by one parameter at three interface boundaries.
- **Part 4b (adjacent observations):** noted — the existing `DefaultUserDeletionServiceTest` had several `recordBannedUser` verify calls using positional matchers; each was updated minimally rather than refactoring the test class.
- **Part 6 (translations):** Rule 1 namespace (DIALOG) honoured. Rule 2 parent/child collision **explicitly avoided** by choosing root prefix `info.*` rather than `user.info.*` for the new system-actor key — documented in the seed file's inline comment so the next reader understands the prefix choice. Rule 3 ID-numbering: still placeholder=100 per Igor's standing directive.
- **Part 11 (trust boundaries):** confirmed. Admin id flows server-side from `OglasinoAuthentication.getUserId()` to the facade to the audit service. Never accepted from the request body.
- **Part 12 (schema patterns):** N/A — no partial-index predicates touched.
- **Part 13 (transactional / cache self-call patterns):** N/A — no transactional self-calls touched.

### For Mastermind (patch addendum)

- **Asymmetric nullability is intentional and now documented.** Ban side: nullable `banned_by_admin_id` (system-cascade can insert without an admin actor). Lock side: NOT NULL `locked_by_admin_id` (no system-initiated lock call site exists today). If a future feature adds a system-initiated lock — e.g. an automated abuse-detection lock — that column would need to be made nullable too. Worth knowing.
- **Translation key root prefix divergence.** `info.dialog.field.banned.by.system` (no `user.` prefix) sits outside the rest of the admin-extension keys (all `user.*`). The frontend will need to remember that the system-actor fallback is rooted at `info.*`, not `user.info.*`. If a future cleanup pass renames the existing leaf `user.info.dialog.field.banned.by` → `user.info.dialog.field.banned.by.label`, the system-actor key could move under the same parent (`user.info.dialog.field.banned.by.system`). Out of scope for this session.
- **Database migration note.** Pre-prod V1 fold convention applied. Once prod is up, any further column additions on `banned_user_audit` need a new Flyway migration file rather than an inline V1 edit.

---

## Follow-up patch — 2026-05-19 (UserOverviewDTO row-state fields)

Closes the row-state visibility gap surfaced by the frontend engineer's Step-0 investigation. The admin Users page needs per-row visibility into deletion and lock state to render the row indicators (Banned / Pending deletion / Locked) and toggle the Lock/Unlock icon. The state-info endpoint exists but is per-user — using it per row would force N+1 fetches on the admin Users page. The list endpoint now carries the data already.

### Implemented (overview-DTO patch)

- **`UserOverviewDTO` extended** with two new fields:
  - `DeletionStatus deletionStatus` — mirrors `users.deletion_status` (enum: `ACTIVE` | `PENDING_DELETION`). Serializes as the enum name.
  - `boolean lockedFromDeletion` — true iff an active `user_deletion_locks` row exists for this user at the moment of the query (i.e. `expires_at IS NULL OR expires_at > now()`).
- **`UserOverviewProjection` extended** with `deletionStatus` (the User entity already exposes it; one extra column on the SELECT). `lockedFromDeletion` is intentionally not on the projection — the JPA Criteria + correlated-subquery + CASE-EXISTS combination is awkward to embed in a `construct()` call. Documented in the projection's class Javadoc.
- **`DefaultUsersService.getUsersOverviewFiltered` reworked** to: (a) add `root.get("deletionStatus")` to the projection select; (b) after the main page fetch, run a single batch query against `UserDeletionLockRepository.findActivelyLockedUserIds(pageUserIds, now)` and build a `Set<Long>` of locked user ids; (c) set `lockedFromDeletion` on each DTO based on Set membership. Two roundtrips per page request (main paged query + one batch lock lookup); page size is bounded and the lock lookup is by indexed PK so the cost is predictable. Brief's stated preference for "JOIN/projection or per-row EXISTS" — chose batch lookup as a third option that's cleaner than either: a single small query for the whole page, no Criteria gymnastics, no N+1.
- **`UserDeletionLockRepository.findActivelyLockedUserIds(Collection<Long>, Instant)` added** — backs the batch lookup. JPQL IN-clause + the standard active-lock time predicate.
- **`UserOverviewConverter` extended** for the ModelMapper path (used when `User` entities are nested inside `AdminReviewDTO` / `ReportDTO` / `UserDetailsDTO`):
  - `deletionStatus` set from `source.getDeletionStatus()` — free.
  - `lockedFromDeletion` computed via per-row `deletionLockRepository.existsActiveLockForUser`. These pages are paged in their respective list endpoints; the per-row repo call is bounded but flagged below.
  - Cleaned up: removed the unused `ModelMapper` autowire field that the original carried but didn't use.
- **No frontend coordination** — the frontend session reads the new wire shape from this patch.

### Files touched (overview-DTO patch)

Modified:
- `src/main/java/com/memento/tech/oglasino/admin/dto/UserOverviewDTO.java` (+22 / -0 — two fields + accessors)
- `src/main/java/com/memento/tech/oglasino/repository/projections/UserOverviewProjection.java` (+15 / -3 — new column + Javadoc + DTO populate)
- `src/main/java/com/memento/tech/oglasino/repository/UserDeletionLockRepository.java` (+17 / -0 — batch query method)
- `src/main/java/com/memento/tech/oglasino/admin/service/impl/DefaultUsersService.java` (+22 / -1 — select + post-page batch lookup + helper)
- `src/main/java/com/memento/tech/oglasino/admin/converter/UserOverviewConverter.java` (+10 / -4 — deletionStatus + lockedFromDeletion populate; dropped unused ModelMapper field)

Created:
- `src/test/java/com/memento/tech/oglasino/admin/converter/UserOverviewConverterTest.java` — 4 tests covering ACTIVE-unlocked, PENDING_DELETION-unlocked, ACTIVE-locked, and the mixed-state PENDING_DELETION-AND-locked case.
- `src/test/java/com/memento/tech/oglasino/repository/projections/UserOverviewProjectionTest.java` — 2 tests verifying `toDTO()` copies `deletionStatus` and leaves `lockedFromDeletion` at its default (the service layer fills it).

### Tests (overview-DTO patch)

- Ran: `./mvnw spotless:check && ./mvnw test`
- Before this patch: 472 passed.
- After this patch: **478 passed, 0 failed** (+6 net new tests across the converter and projection test classes; brief targeted "+2–4" but the four state combinations + the two projection assertions are all distinct behaviours worth pinning).
- Spotless: clean.

### Cleanup performed (overview-DTO patch)

- Removed unused `@Autowired private ModelMapper modelMapper` field from `UserOverviewConverter` (carried in the original but never referenced). Dependency list is now tight against actual usage.

### Config-file impact (overview-DTO patch)

- `conventions.md`: no change.
- `decisions.md`: no change.
- `state.md`: no change.
- `issues.md`: no change.

### Obsoleted by this patch

- The implicit "frontend must call `state-info` per row to render row indicators" assumption — eliminated. The list endpoint carries `deletionStatus` + `lockedFromDeletion` on every row.

### Conventions check (overview-DTO patch)

- **Part 4 (cleanliness):** confirmed. Removed the unused `ModelMapper` autowire while in the file.
- **Part 4a (simplicity):** confirmed. Batch lookup chosen over (a) Criteria CASE/EXISTS embedded in `construct()` and (b) per-row EXISTS in the page-processing loop. The batch lookup is one well-bounded query (page size cap) by indexed PK — predictable, readable, no Criteria gymnastics.
- **Part 4b (adjacent observations):** flagged below.
- **Part 7 (error contract):** N/A — no new error codes.
- **Part 11 (trust boundaries):** confirmed. No new client inputs; all new fields are server-derived from `User` entity or the lock table.
- **Part 12 (schema patterns):** N/A — no schema change.
- **Part 13 (transactional / cache patterns):** N/A — no cache or self-call.

### For Mastermind (overview-DTO patch addendum)

- **`UserOverviewConverter` does a per-row repo call.** The ModelMapper path used by nested users in `AdminReviewDTO` / `ReportDTO` / `UserDetailsDTO` does one `existsActiveLockForUser` call per nested User. For the admin Reviews and Reports pages this means N (or 2N for reports that have both reporter and reportedUser) extra small queries per page. Page sizes are bounded so this is acceptable, but worth Mastermind awareness if either page grows or starts displaying many users per row. The list path itself (admin Users page) uses the batch lookup and avoids this. Fix options if it bites: (a) thread a precomputed `Set<Long> lockedUserIds` through ModelMapper's context — possible but ugly; (b) move the converter to a JPQL projection like the users-list path; (c) accept the cost. **Severity:** low — bounded by existing page sizes.
- **`UserOverviewProjection` mismatched-cardinality smell.** The projection record now has 6 components but the DTO has 7 fields. `lockedFromDeletion` is the asymmetry — DTO field with no projection slot. The class Javadoc explains why, but a reader might find this surprising. Acceptable trade-off vs adding a 7th column that's computed in a different query anyway.
- **List endpoint wire-shape change is additive.** Existing clients that ignore the two new fields keep working. The frontend session will pick up the fields to drive the row indicators and Lock/Unlock icon toggle. No coordination required between this session and the frontend session beyond the spec-amendment text above.
