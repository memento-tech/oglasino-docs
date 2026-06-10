# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-06-10
**Task:** READ-ONLY AUDIT — verify the Privacy Policy / Terms of Use draft claims (ban flow, hard/soft deletion, notifications/follows/reports, OpenAI payload, automated enforcement & view counting, ops/infra) against backend code, with twice-verified file:line evidence.

## Implemented

- No code changed (read-only audit). Produced the audit deliverable `.agent/audit-legal-fact-verification.md` answering Q1–Q7 with verdicts, file:line evidence, plain answers, and an adjacent-findings section.
- Every cited claim verified twice (direct `Read` of the region **and** an `rg` confirmation), per the brief's verification discipline.

## Files touched

- `.agent/audit-legal-fact-verification.md` (new — audit deliverable; written on Igor's explicit "write audit file" instruction)
- `.agent/2026-06-10-oglasino-backend-legal-fact-verification-1.md` (new — this summary)
- `.agent/last-session.md` (overwritten — exact copy of this summary)

No source, test, config, or schema file was modified.

## Headline verdicts

- **Q1 ban:** VERIFIED-FALSE vs ToU §10.2 — ban sets `disabled=true` + hides profile/phone/listings + disables Firebase auth; deletes nothing. No auto-deletion timer for banned accounts.
- **Q2 hard delete:** grace 7d / retention 30d / ban 12mo all match; both purge jobs run weekly (Sun 04:00). SHA-256 is **unsalted**. ES docs **not** deleted, **no** CDN purge, chat messages **anonymized not deleted**.
- **Q3 soft delete:** (a) profile visible + badge ✔, (b) phone hidden ✔, (c) listings hidden ✔, (d) reviews visible ✔, (e) PARTIAL — outbound messaging block verified backend-side, inbound is Firestore-rules (other repo), (f) **VERIFIED-FALSE** — anonymization is hard-delete-time `deleted:<uid>` sentinel, not a soft-delete "Deleted User" name, (g) login restore ✔.
- **Q4:** favorite notif does NOT identify favoriter; chat push DOES carry sender name + full text; follow stored in `user_follows`, only caller's own followings exposed; report types PRODUCT/USER/REVIEW (no message type); delete-own-listing exists, **delete-own-review does not**.
- **Q5:** VERIFIED-TRUE — only prompt+text sent to OpenAI; no IDs, no `user` field, no custom headers, `store=false`; chat never sent.
- **Q6:** PARTIAL — content validation never auto-enforces, but the firebase-cascade abuse check auto-bans+deletes (system, no human); view counting keeps a 12h per-device/per-IP Redis dedup key (`view:{productId}:{deviceId|ip}`).
- **Q7:** logs carry IP+userId to stdout, no in-app 90-day purge; alerts/incident_log carry no PII, incident_log never purged; **Redis + Elasticsearch are self-hosted Docker** (ES = `oglasino-es` compose on VPC-private IP, NOT DO-managed); Postgres is DO-managed; email verification/reset links minted by Firebase, **sent by Brevo**.

## Tests

- None run (read-only audit; no code changed).

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change (audit output is a deliverable, not a status flip; Mastermind/Docs-QA decide any state edits after seam analysis)
- issues.md: no change by me — but six adjacent findings below are candidates for `issues.md` entries; drafting is Mastermind's call (see "For Mastermind").

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code written, no debug artifacts.
- Part 4a (simplicity): N/A — no code added. See "For Mastermind" structured evidence.
- Part 4b (adjacent observations): six findings flagged below and in the audit file's "Adjacent findings" section.
- Part 6 (translations): N/A this session.
- Part 11 (trust boundaries): exercised as audit subject — confirmed ban/deletion/messaging trust decisions read server-side state (auth filter, DB), and the OpenAI/notification paths derive identity server-side.

## Known gaps / TODOs

- Q3(e) inbound-messaging block and overall Firestore message-content lifecycle are enforced (if at all) in `oglasino-firestore-rules` — out of this repo; flagged for the Firestore Rules audit/agent.
- DO-platform claims (Postgres at-rest encryption, log-driver retention) are infra/operational, not verifiable from code.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **Note on the brief vs the "write audit file" instruction:** the brief said "do not create any file"; Igor then explicitly instructed "write audit file," so I created the Phase-2 deliverable `.agent/audit-legal-fact-verification.md` (slug `legal-fact-verification`) plus this summary. No code/test/config files were created or changed.

- **Six adjacent findings (candidates for `issues.md`):**
  1. `repository/UserRepository.findUserInfoByFirebaseUid:160-193` lacks the `u.disabled = false` filter that `findUserInfoById:155` has → banned user's public profile leaks via the firebaseUid lookup path. **Severity: medium** (banned-user privacy posture inconsistency). Not fixed — out of scope.
  2. User hard delete removes no Elasticsearch docs (`runHardDelete:303-455` has zero ES calls); product docs orphan until next full reindex. **Severity: medium.** Out of scope.
  3. Chat **message body text** of a deleted user is never anonymized/deleted — the Sunday cron only rewrites sender/receiver IDs to a sentinel (`DefaultMessagingCleanupService`). **Severity: medium** for any PP message-erasure claim. Out of scope.
  4. Transient Redis keys (`view:…`, `product:owner:…`, report dedup) not purged at hard delete; expire by TTL. **Severity: low.**
  5. `User.numOfPenalties` is write-only (incremented at `DefaultAdminReviewService:91`, never read for enforcement). **Severity: low.**
  6. `user.deletion.report.window.days` (config id 50) exists but isn't independently enforced — report handling keys off ban status. **Severity: low.**

- **Doc-accuracy flags the legal-drafts chat should act on (relative to the quoted draft text):** ToU §10.2 ban-deletes-everything is false; PP §9 "all enforcement by a human admin" is overstated (firebase-cascade auto-ban); PP §2.4 "anonymous totals only" omits the 12h per-IP/device view dedup key; PP §8.2 90-day log purge is not implemented in-app; PP §3 "Deleted User at soft delete" is wrong (it's a hard-delete sentinel); PP §4 should reflect ES + Redis as self-hosted (no third-party processor) and Brevo (not Firebase) as the email sender; PP §10 admin-access "logging and monitoring" is only partially code-backed.
