# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-06-02
**Task:** Phase-2 read-only audit for the email-notifications feature — inventory EmailService/Brevo, Firebase Admin SDK, the login/verification gate, the event architecture across ten transitions, BACKEND_TRANSLATIONS, and cross-repo seams. Output `.agent/audit-email-notifications.md`.

## Implemented

- Wrote `.agent/audit-email-notifications.md` (the sole deliverable), six sections per the brief plus a TL;DR, a "Brief vs reality" section, and out-of-scope observations.
- Verified every CRITICAL claim and every "does not exist" firsthand by reading the actual files / grepping — not relying solely on sub-agent search. Key files read directly: `DefaultEmailService.java`, `EmailService.java`, `FirebaseAuthFilter.java`, `DefaultFirebaseAuthService.java`, `DefaultUsersFacade.java`, `UserDeletionScheduledJobs.java`, `DefaultUserDeletionService.runHardDelete`, the three `application-*.yaml` email blocks.
- Key findings: (1) send path is plain-text only (`multipart=false`), a new HTML method is needed; (2) **no** `POST /api/secure/admin/email/test` endpoint exists and `EmailService` has zero production callers — contradicts the brief; (3) email-verification login gate = **(c) nothing**, trust boundary clean (`emailVerified` from authoritative token, never client input); (4) the `UserStateChangedEvent`+AFTER_COMMIT precedent is real but ban/unban run outside any tx (fallbackExecution only), and 3 of 10 transitions (deletion reminder, deletion postponed, product blocked) **do not exist in code**; (5) BACKEND_TRANSLATIONS supports cron/no-request sends via `getBackendTranslation(key, langCode)` + `users.preferred_language_id`, proven by existing push-notification code.

## Files touched

- `.agent/audit-email-notifications.md` (new, audit output — not source code)
- `.agent/2026-06-02-oglasino-backend-email-notifications-1.md` (this summary)
- `.agent/last-session.md` (exact copy of this summary)

No source/test/config code changed (read-only audit).

## Tests

- Ran: none. Read-only audit; no code changed, so no build/test run is warranted.
- Result: N/A
- New tests added: none

## Cleanup performed

- none needed (no code touched).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change (a future `email-notifications` pipeline row is Mastermind/Docs-QA's to draft when the feature is planned, not this audit's).
- issues.md: no change. The non-existent test endpoint and three non-existent transitions are reported in the audit for Mastermind's seam analysis; whether any become issues entries is Mastermind's call.

## Obsoleted by this session

- nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code touched, audit file is the only new artifact and is referenced (the brief's deliverable).
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): three flagged (dead EmailService scaffolding; `getBackendTranslation` `.orElseThrow` with no fallback; `emailVerifiedExternal` captured-once staleness) — in the audit's "Out-of-scope observations" and repeated in "For Mastermind".
- Part 6 (translations): N/A this session (no seeds added; audit documents the seed mechanism for the feature).
- Part 11 (trust boundaries): checked explicitly — `emailVerified` source is the authoritative Firebase token, not client input. Clean. Email envelope is server-controlled.
- Part 13 (transactional patterns): relevant — documented that deletion paths use `TransactionTemplate` (AFTER_COMMIT works) while ban/unban use no tx (fallbackExecution only).

## Known gaps / TODOs

- I did not read sibling repos (web/router/firestore) per CLAUDE.md — Section 6 states assumptions and what's needed from each, rather than asserting their contents.
- The concrete email sender addresses are env-injected; I report the values documented in YAML comments (`noreply@`/`noreply-stage@`) but the actual `reply-to` value is not asserted in code and is left as unknown-from-code.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code or abstractions introduced.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- **Brief-vs-reality (needs your attention before Phase 3/4):**
  - The brief asserts `POST /api/secure/admin/email/test` exists ("do not delete"). It does **not** exist; `EmailService` has no production callers. Decide whether a later brief adds-then-removes a temporary test endpoint, or the first listener brief becomes the first real caller.
  - Transitions 3 (deletion reminder), 7 (deletion postponed by admin), 9 (product blocked) **do not exist in code**. #3 is net-new cron work (data model supports it via `scheduled_deletion_at`; needs a `reminder_sent_at` guard). #7 — nearest is admin "lock from deletion" (`UserDeletionLock`); needs a product definition of "postpone." #9 — `ModerationState.BANNED` is never set; the moderation-block transition must be built before any email listener can hang off it.
- **Biggest design driver:** AFTER_COMMIT is only meaningful where a Spring tx wraps the publish. Deletion request/cancel (TransactionTemplate) qualify; **ban/unban do not** (no `@Transactional`, listener fires via `fallbackExecution=true`, no commit/rollback boundary). And `UserStateChangedEvent` is overloaded across 5 transitions — recommend dedicated per-transition events for email-template routing rather than overloading it.
- **Verification-gate decision:** read the live `decoded.isEmailVerified()` off the FirebaseToken the filter already holds (always current, no cache change) rather than the stored `emailVerifiedExternal` column (captured once, goes stale). A blanket gate only bites email/password registrations (Google tokens arrive verified) — confirm that's the intent.
- **Seams to resolve with web/router (Section 6):** web `/verify` (or equivalent) continue-URL per base-site/locale; whether web already generates Firebase action links client-side (avoid backend duplication); router preserving Firebase `oobCode`/`mode` query params; Firebase console authorized action domains.
- **Adjacent observations (Part 4b):** (low/med) dead EmailService scaffolding — send path has never executed against real SMTP in any profile; (low) `getBackendTranslation` `.orElseThrow()` no-fallback risk for cron batches; (low) `emailVerifiedExternal` captured-once staleness, harmless today (only display reader).
- No drafted config-file text. No config-file dependency created.
