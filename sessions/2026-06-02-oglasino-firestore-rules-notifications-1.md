# Session summary

**Repo:** oglasino-firestore-rules
**Branch:** stage
**Date:** 2026-06-02
**Task:** Read-only audit. No code changes. Audit every notifications-related Firestore rule for the notifications feature (in-app + push); Igor suspects a bug. Write findings to `.agent/audit-notifications.md`.

## Implemented

- Phase-2 read-only audit only — no rules, tests, indexes, or config changed.
- Audited the single notifications rule (`firestore.rules:74–79`, `notifications/{userId}/userNotifications/{notificationId}`) per operation, plus the adjacent `users/{userId}` rule for the PII question.
- Conclusion: **no security bug.** `create: if false` (Admin-SDK-only) + recipient-only `read`/`update`/`delete` is the correct, consistent pattern; neither "too open" nor "too closed" applies. Flagged one low-severity soft spot: the `update` rule has no field constraints (recipient can rewrite any field, not just `seen`).
- Confirmed no `fcmToken`/push-token field exists anywhere in this repo; documented that if stored on `users/{userId}` it is owner-only-read, and laid out the cross-repo seams to verify.
- Wrote `.agent/audit-notifications.md` with verbatim rule quotes + line numbers, the access matrix, the bug verdict, the write-vs-read consistency analysis, the PII finding, and five seams.

## Files touched

- `.agent/audit-notifications.md` (new, audit deliverable)
- `.agent/2026-06-02-oglasino-firestore-rules-notifications-1.md` (this summary)
- `.agent/last-session.md` (duplicate of this summary)

No source files (`firestore.rules`, `tests/**`, `firestore.indexes.json`, `firebase.json`, `package.json`, `tsconfig.json`) were modified.

## Tests

- Ran: `npx firebase emulators:exec --only firestore --project demo-oglasino "npx vitest run"` (local emulator, demo project — never a real project)
- Result: 68 passed, 0 failed (3 test files). Green baseline confirmed; PERMISSION_DENIED stderr lines are the expected negative-path assertions.
- New tests added: none — read-only audit, no rule change to exercise. The existing notification tests (16, 17, 40–46 in `tests/messaging.test.ts`) already cover every cell of the access matrix.
- Note: bare `npm test` is `vitest run` and needs the emulator already running; the correct local invocation is the `emulators:exec` wrapper above.

## Cleanup performed

- none needed (no code written)

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change. The low-severity unconstrained-`update` observation is raised to Mastermind in this summary (and in the audit file) for triage; per Part 4b, Mastermind decides whether it becomes an `issues.md` entry. No drafted entry is owed from this session.

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): confirmed — no code, no commented-out rules, no debug(), no TODO/FIXME added.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one flagged — unconstrained notifications `update` rule (low). See "For Mastermind".
- Part 6 (translations): N/A this session.
- Part 11 (trust boundaries): confirmed — the notifications rule trusts only `request.auth.uid` vs. the path segment; no Firestore-field or custom-claim trust, no `get()`/`exists()`. No trust-boundary violation found. PII (push-token) storage location is a cross-repo seam, not resolvable here.
- Other parts touched: none.

## Known gaps / TODOs

- The audit names five seams that this repo cannot close (backend write path/collection name, recipient-id namespace = Firebase uid, push-token storage location + its rule, and whether a compound `seen`+`createdAt` query needs a composite index that is currently absent). These require backend/expo reads — out of scope for a rules-repo audit.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no rules/tests/abstractions introduced.
  - Considered and rejected: nothing — no code-shape decisions were in play.
  - Simplified or removed: nothing.
- **Bug verdict:** the suspected notifications bug is not present. `firestore.rules:74–79` is correct — `create: if false` (Admin SDK writes), recipient-only read/update/delete, no any-user access. Tests 16, 17, 40–46 pass and cover the full matrix.
- **Adjacent observation (Part 4b), low severity:** `firestore.rules:76` — the notifications `update` allow has no field-immutability constraints, unlike the messages update rule (`:62–70`). The recipient can rewrite any field (title/body/createdAt), not just toggle `seen`. Blast radius is the user's own private notifications only (no cross-user exposure, not a security hole), so low. If the spec wants notification content immutable-once-written with only `seen` togglable, tighten the update rule to pin non-`seen` fields. **I did not fix this because it is out of scope (read-only audit) and it is a spec decision.**
- **Seams to confirm in Phase 3** (full detail in `.agent/audit-notifications.md` §6): (a) backend writes exactly `notifications/{firebaseUid}/userNotifications/{id}` via Admin SDK; (b) recipient keyed by Firebase uid, not a backend numeric id (a mismatch would mimic the "too closed" bug); (c) push-token storage location + read rule live in backend/expo; (d) if the client queries notifications with a `seen` filter + `createdAt` order, a composite index on `userNotifications` is owed and is currently absent from `firestore.indexes.json`.
- No config-file drafts owed from this session.
