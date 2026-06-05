# Session summary

**Repo:** oglasino-firestore-rules
**Branch:** stage
**Date:** 2026-06-02
**Task:** notifications — tighten the notifications update rule so the recipient may change ONLY the `seen` field; all other fields pinned to their stored values; read/create/delete unchanged.

## Implemented

- Split the notifications rule's `update` away from `read`/`delete`. `read` and `delete` keep the unchanged recipient-only condition (`request.auth != null && request.auth.uid == userId`); `update` now carries an additional field-lock.
- The new `update` rule requires, on top of recipient-only: `request.resource.data.diff(resource.data).affectedKeys().hasOnly(['seen'])` and `request.resource.data.seen is bool`. Net effect: the recipient may flip `seen` and nothing else — every other field is pinned to its stored value, and *adding* a new field is also rejected.
- `create: if false` unchanged (Admin-SDK-only writes; clients never create). Security posture otherwise untouched — this is a field-constraint, not a re-architecture.
- Idiom choice (`affectedKeys().hasOnly` rather than the messages-style per-field comparison) was an explicit, Igor-approved override of the brief — see "For Mastermind."

## Files touched

- firestore.rules (+5 / -2) — notifications block: split `update` from `read,delete`; added the `hasOnly(['seen'])` + `seen is bool` field-lock.
- tests/messaging.test.ts (+22 / -0) — two new notification field-lock tests (54, 55).

## Tests

- Ran: `npx firebase emulators:exec --project demo-oglasino "npx vitest run"`
- Result: 70 passed, 0 failed (messaging.test.ts 50, blocks.test.ts 15, users.test.ts 5). Baseline was 68; +2 added = 70, all green.
- New tests added:
  - `54. notification update mutating a non-seen field by owner → fail` — owner updates `title` only; denied by the field-lock (would have PASSED the write under the old unconstrained rule, so it genuinely exercises the change).
  - `55. notification update flipping seen AND a non-seen field by owner → fail` — owner sends `{seen:true, title:'tampered'}`; denied because `affectedKeys()` is `{seen, title}`, not `hasOnly(['seen'])`. Confirms the lock rejects mixed updates, not just pure non-seen ones.
- Existing seen-only-flip pass case is already covered by test `42. notification update by owner → pass` (`update({seen:true})`); kept and still green. The recipient-only access matrix tests (16, 17, 40–46) all still pass.

## Cleanup performed

- None needed. No commented-out rules, no `debug()` calls, no TODO/FIXME left in `firestore.rules`. No unused helpers introduced (no helper added — the lock is inline, matching the file's no-one-expression-wrapper style).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change. (Idiom override is recorded here for Mastermind; if Mastermind wants the "correctness-over-consistency / prefer `affectedKeys().hasOnly` when the schema is cross-repo-unverifiable" rule promoted to a standing decision, drafted text is in "For Mastermind" — otherwise no change required.)
- state.md: no change.
- issues.md: no change. (The §3 audit "soft spot" — notifications update had no field constraints — is now resolved by this session; no new issue authored.)

## Obsoleted by this session

- The audit's §3 "one genuine soft spot" (notifications `update` rule has no field constraints) is now closed by the field-lock. The audit file (`.agent/audit-notifications.md`) remains accurate as a historical record of the pre-change state; not deleted.
- Nothing else.

## Conventions check

- Part 4 (cleanliness): confirmed — `npm test`/emulator run green, no commented rules, no debug, no orphan test files (both new tests live in the existing `messaging.test.ts` and run under the suite).
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): one schema-mismatch finding flagged in "For Mastermind" (Brief vs reality #1). No new out-of-scope rule problems observed.
- Part 6 (translations): N/A this session.
- Part 11 (trust boundaries): the `update` rule trusts `request.auth.uid == {userId}` (Firebase-verified, client-unforgeable) plus a structural constraint on the written payload. No Firestore-field trust, no custom-claim trust, no `get()`/`exists()`. Blast radius is the recipient's own private notifications. No trust boundary loosened — only tightened.

## Known gaps / TODOs

- The `seen is bool` guard enforces the type but the spec doesn't require `seen` to monotonically go `false → true` (a recipient could set it back to `false`, i.e. mark-as-unread). That is harmless and likely desired; not constrained. Noted, not changed.
- The live notification document schema is not verifiable from this repo (see Brief vs reality #1). The `hasOnly(['seen'])` idiom was chosen specifically so the rule is correct regardless of the full field set, so this gap does not affect correctness — but the seam (what the backend actually writes) remains a backend/expo verification item, consistent with the audit's SEAMS §6.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): one `affectedKeys().hasOnly(['seen'])` clause + one `seen is bool` clause on notifications `update`, and a one-line split of `update` out of the shared `read, update, delete`. Earned: it closes the audited soft spot (recipient could rewrite any field of their own notification) with the minimum surface that expresses exactly "only `seen` may differ."
  - Considered and rejected: (a) the messages-style **per-field** pin (`request.resource.data.X == resource.data.X` for title/body/createdAt/…). Rejected because it requires enumerating the full non-`seen` field set into a security rule, and the field set is not verifiable from this repo (Brief vs reality #1); it also does **not** block *adding* a new field. (b) Adding a scoped helper to wrap the lock — rejected as a one-expression wrapper (Part 4a) and inconsistent with the file's inline style.
  - Simplified or removed: nothing removed. The change is net-additive but minimal.

- **Brief vs reality #1 — notification field set differs from the brief.**
  - Brief says: the in-app notification doc carries `title, description, seen, createdAt, type, categoryId, data`.
  - Rules/tests say / I observed: the only verifiable shape in this repo is `seedNotification` (`tests/messaging.test.ts`), which writes `title, body, seen, createdAt` — `body` (not `description`), and no `type`/`categoryId`/`data`. The backend write path is cross-repo and not readable from here.
  - Why this matters: a per-field pin would have baked an unverifiable field enumeration into a security rule. Either the brief's list or the test seed (or both) could be incomplete relative to production, leaving unpinned fields a recipient could tamper with.
  - Recommended resolution: chosen — use `affectedKeys().hasOnly(['seen'])`, which is correct for any schema. Separately, the test seed and the backend write should be reconciled (is it `body` or `description`? do `type/categoryId/data` exist?) so the `seedNotification` fixture matches production; that reconciliation is a backend/Docs seam item, not a rules change.

- **Brief vs reality #2 — idiom instruction overridden (Igor-approved).**
  - Brief says: "Mirror the EXACT pattern the messages update rule uses… match whichever the messages rule already uses, for consistency" → the file only contains the per-field idiom, so the brief points at per-field.
  - Decision: used `request.resource.data.diff(resource.data).affectedKeys().hasOnly(['seen'])` + `request.resource.data.seen is bool` instead. Surfaced the collision to Igor before editing; Igor confirmed "Option 1 — correctness beats consistency here." Rationale: per-field can't reliably enumerate the schema cross-repo (#1) and doesn't block added fields; `hasOnly` does both. This is a new idiom in the file (messages still uses per-field) — flagged so Mastermind is aware the two sibling rules now use different field-lock idioms by design.
  - Optional drafted decisions.md entry (apply only if Mastermind wants it standing): titled **"Notification update field-lock uses affectedKeys().hasOnly over per-field pinning"** — "When the document's full field set cannot be verified within the rules repo (cross-repo-owned schema), prefer `diff().affectedKeys().hasOnly([...])` over per-field equality pins for update field-locks: it enforces 'only these keys may change' for any schema and additionally rejects added fields. Messages-update retains per-field pinning (its schema is repo-local and fixed)."

- Nothing else flagged.
