# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-18
**Task:** PR Review — User Deletion (oglasino-backend). Read-only senior-engineer review of every changed file on the branch; surface spec-vs-code divergences and rank by severity. No code changes.

## Implemented

- Read the spec (`oglasino-docs/features/user-deletion.md`), the user-deletion-relevant entries in `decisions.md` (auth-cache TTL constraint, connection-pool incident), and conventions Parts 4, 4a, 7, 11.
- Audited every file in `git status` (39 new + 36 modified). Read whole files where the diff was load-bearing (auth filter, deletion service, scheduled jobs, controllers, schema, configuration, translations); spot-checked the rest.
- Verified the branch state with `./mvnw spotless:check` (passed) and `./mvnw test` (441 passed, 0 failed, 0 errors).
- Wrote `.agent/user-deletion-pr-review.md` with verdict, 18 findings (2 critical, 3 high, 6 medium, 7 low), spec-vs-code divergences, adjacent observations, scope-of-review note, and Mastermind-facing follow-ups.

## Files touched

- `.agent/user-deletion-pr-review.md` (+450 / -0) — new PR review report
- `.agent/2026-05-18-oglasino-backend-user-deletion-pr-review-1.md` (+60 / -0) — this summary
- `.agent/last-session.md` (+60 / -0) — duplicate of this summary

## Tests

- Ran: `./mvnw spotless:check` — **passed** (570 files, 0 needs changes)
- Ran: `./mvnw test` — **passed** (441 tests, 0 failures, 0 errors, 0 skipped)
- Result: branch is green as delivered. No code changes by this session.

## Cleanup performed

- none needed — review only, no code touched.

## Obsoleted by this session

- nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code written, spotless + tests green.
- Part 4a (simplicity) / Part 4b (adjacent observations): confirmed — adjacent observations flagged in §"Adjacent observations" of the PR review.
- Part 6 (translations): confirmed — translations are scope-relevant; English seed file at `data/translations/0004-data-user-deletion-translations-EN.sql` reviewed for namespace correctness (no parent-leaf collisions, `BANNED_DIALOG` added to the namespace CHECK and enum).
- Part 7 (error contract): confirmed — new error codes use the unified `{errors:[{field, code, translationKey}]}` envelope via a fresh `UserDeletionException` hierarchy (not added to `ProductErrorCode`, per the open `issues.md` HIGH entry).
- Part 11 (trust boundaries): confirmed — delete-account endpoint takes an empty body, userId derived from `OglasinoAuthentication`, `auth_time` read from the verified-token credentials slot. Audit covered in the report.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change required by this review; if the C-1/C-2 fixes ship in a follow-up brief, state.md update is owned by that session.
- issues.md: draft below for Docs/QA — see "For Mastermind."

## Known gaps / TODOs

- None for this session. Two follow-up briefs are needed (see "For Mastermind") to close C-1 and C-2; they are not this session's work.

## For Mastermind

### Blocking findings (require a backend follow-up brief)

1. **C-1 — Cron does not re-verify deletion state inside the hard-delete transaction.** A user who signs in after the cron's page load but before the row is processed gets hard-deleted anyway. `runHardDelete` performs no precondition check on `user.getDeletionStatus()`. Recommended fix: re-fetch the user / request row inside the `writeTx.executeWithoutResult` block and abort cleanly if not pending. See PR review §C-1 for fix shape.

2. **C-2 — Disapproved reviews authored by the deleting user are not handled; FK blocks user delete.** `DefaultUserDeletionService.runHardDelete` only handles approved (anonymize), pending (delete), and reviews about user (delete). Reviews where the deleting user is the reviewer and `approved = false` (disapproved by admin) survive, and the `review.reviewer_id` FK has `NO ACTION`, so step 12 (`userRepository.delete(user)`) raises a referential-integrity violation and the transaction rolls back. The cron will keep retrying forever. Spec §3.3 / §5 also need a policy decision for this case.

### High-severity findings (should ship with the same brief)

3. **H-1 — `triggered_by` is inherited from a pre-existing audit row.** Admin force-delete over a user-initiated pending request records `triggered_by = 'user_request'` instead of `'admin_action'`. Same shape for firebase-cascade. Compliance trail risk.

4. **H-2 — Audit purge can erase audit rows before `actual_deletion_at` is written.** The `actual_deletion_at` update is post-commit bestEffort; the purge job's `WHERE retention_until <= :now` does not exclude unfinished rows. Either move the update into the transaction or filter the purge.

5. **H-3 — Firestore deletion runs inside the JPA transaction.** Failure of the user-row delete in step 12 leaves notifications gone and the user still in the DB. Move the Firestore call to the post-commit bestEffort block.

### Spec corrections drafted for Docs/QA

These are documentation fixes only — apply to `oglasino-docs/features/user-deletion.md`:

- **§3.9 / §7:** the three cron knobs (`user.deletion.hard.delete.cron`, `user.deletion.audit.purge.cron`, `user.deletion.firebase.reconciliation.cron`) **cannot** live in the `Configuration` DB table because Spring's `@Scheduled(cron = "${…}")` resolves at bean construction from the Environment, not from a runtime cache. Spec should split the knob inventory into "DB-backed (hot-reloadable)" and "yaml-only (cron expressions, redeploy to change)." Implementation already does this with explanatory comments at both sites; spec is the only thing wrong.

- **§6.4:** `user_deletion_requests.status` values are uppercase (`'PENDING'`, `'CANCELLED'`, `'COMPLETED'`) in the CHECK constraint, the partial-index predicate, and the JPA `EnumType.STRING` storage — matching the `DeletionRequestStatus` enum constants. Spec currently shows lowercase.

- **§3.3 / §5:** define the policy for reviews authored by the deleting user where `approved = false` (admin-disapproved). Implementation does nothing with these and the FK blocks the user delete (C-2). Most natural policy: delete them along with pending reviews, since the user is being erased and a disapproved review carries no public value.

- **§8.1:** clarify whether `markForImmediateDeletion(User user, String reason)` is supposed to flow `reason` into the audit row / ban reason (currently the parameter is ignored, and both `triggered_by` and the auto-ban reason are hard-coded). If not, drop the parameter from the signature.

### Issues.md candidate

- The pre-existing `UserInfoProjection.getBaseSiteCode()` ↔ query alias `baseSiteId` mismatch (L-4 in the review) is a real bug masked by the downstream `Optional.ofNullable`. Worth an `issues.md` entry so it isn't forgotten. Severity TBD by Mastermind; my read is medium — the projection getter is effectively dead, which means UI components that depend on `baseSiteCode` from this path silently get `null`.

### Workflow note

- Spec names `feature/user-deletion`; current work is on `dev`. Confirm whether to rebase the deletion changes onto a feature branch before merge, or update the spec to match the branch reality.

### What I deliberately did not flag

- Naming, formatting, log-message phrasing — per CLAUDE.md "what is not worth challenging."
- The two known Phase 3 adjacent bugs (chat-id self-self review check, self-assignment report bug) — already fixed manually per spec §3.11.
- The `ReviewRequestDTO.skipTrust` known weakness — operator-confirmed per spec §16.7.
