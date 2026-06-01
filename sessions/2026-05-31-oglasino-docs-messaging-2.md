# Session summary

**Repo:** oglasino-docs
**Branch:** main
**Date:** 2026-05-31
**Task:** Docs/QA close-out: mobile messaging adoption (chat B) — apply the four-config-file + spec edits for the code-complete-pending-Ψ mobile adoption of the frozen `features/messaging.md` contract.

## Implemented

- **`issues.md` — two touches.** (a) Flipped the `2026-05-28 — Chat-store require-cycle triangle (Cycle B)` entry `open → fixed` and appended the engineer's drafted resolution note (2-node dyad not a triangle; broken by lazy `require` on the narrower nav→active edge in `setTempReceiver`; zero behavior change; tsc 0 / 325 tests / lint 80-0). (b) Filed one new `2026-05-31` entry (open/medium): spec §3.6's admin view-token override is not implemented backend-side — a non-participant admin gets 403 `NOT_CHAT_MEMBER`; surface is backend + web-admin, **not** mobile; framed as a decision for separate Mastermind triage (correct the spec vs. open a backend brief). Nothing else filed.
- **`decisions.md` — one closing entry at top.** Records the mobile messaging adoption (code-complete on `new-expo-dev` across five briefs behind the n=1 Phase-2 audit), the parity behavior fixes, the five banked decisions (no new translation seeds; forward block-doc field `blockedUserId`; store-throws/screen-toasts UX; delete-toast reuses `messages.send.failed.toast`; Cycle B lazy-`require` over a new module), the pending-Ψ posture, and the low lint-confirm caveat. Factual vs inferred marked.
- **`state.md` — row + Risk Watch + session-log (no stable flip).** Messaging Expo-backlog row → mobile `in-progress` (adopted in `oglasino-expo-messaging-adoption-2`..`-5`), explicitly NOT `mobile-stable`. Messaging active-feature Tasks-remaining updated to reflect mobile code-complete pending Ψ. New Risk Watch row (Ψ shares the iOS+Android-rebuild device dependency though no new native module is added; + the Brief 3 lint-asserted-not-rendered note). One session-log bullet for this close-out enumerating the five briefs. `Last updated` already 2026-05-31 (no change).
- **`features/messaging.md` — two spec-drift corrections.** §3.4 forward block-doc (`userblocks/{owner}/blocked/{blocked}`) field `blockerId → blockedUserId` (web + mobile both write it; `blockerId` is reverse-index-only — left intact on the reverse doc and the rules wildcard). §5.11 stale seed-row IDs corrected to on-disk EN 3391 / RS 5491 / RU 7591 / CNR 1291. §3.6 deliberately untouched (tracked via the new issues entry).
- **Archival.** Copied the five engineer records (4 build summaries `-2`..`-5` + the n=1 Phase-2 audit deliverable `audit-messaging-adoption.md`) to `sessions/` and deleted the verified sources from `oglasino-expo/.agent/`. Also archived the three sibling-repo Phase-2 spec-validation records that fed this close-out (backend `validate-messaging-backend.md`, rules `validate-messaging-rules.md`, web `validate-messaging-web.md` + its dated session-summary twin `2026-05-30-oglasino-web-validate-messaging-web-1.md`) and deleted their verified sources — same precedent as the consent-mode / product-validation closures.

## Files touched

- issues.md (Cycle B flip + 1 new entry)
- decisions.md (+1 entry at top)
- state.md (Expo-backlog row, active-feature Tasks-remaining, 1 Risk Watch row, 1 session-log bullet)
- features/messaging.md (§3.4 field name, §5.11 IDs)
- sessions/ (+9 archived records, listed above)

## Tests

- N/A (Docs/QA, markdown only). Verification: `cmp` byte-identity check before every source deletion; `grep` confirmation of the §3.4/§5.11 corrections and that reverse-index/rules `blockerId` were left intact.

## Cleanup performed

- Deleted all nine archived source files from sibling `.agent/` folders after `cmp`-verified copies landed in `sessions/`.
- Left in place (out of scope): `oglasino-expo/.agent/audit-expo-readiness-messaging.md` (2026-05-23 pre-build readiness audit, superseded — same handling as the consent-mode readiness audit); `oglasino-firestore-rules/.agent/audit-messaging.md` (45722 B, 2026-05-20 — the *original* messaging-feature rules Phase-2 audit; NOT byte-identical to the already-archived `sessions/audit-messaging.md` (33953 B), so not a verified duplicate; predates this adoption — flagged below, not deleted).

## Config-file impact

- conventions.md: no change
- decisions.md: new entry titled "2026-05-31 — Mobile messaging adoption onto the frozen `messaging.md` contract — code-complete on `new-expo-dev`, pending Ψ"
- state.md: Expo-backlog Messaging row → `in-progress`; Messaging active-feature Tasks-remaining updated; +1 Risk Watch row; +1 session-log bullet
- issues.md: 1 entry amended (Cycle B `open → fixed`); 1 new entry (§3.6 admin view-token gap, open/medium)

## Obsoleted by this session

- The `useActiveChatStore ↔ useChatNavStore` static require-cycle as an open issue — flipped to `fixed` (resolved by `oglasino-expo-messaging-adoption-4`).
- The stale §3.4 `blockerId` forward-doc field name and §5.11 stale ID values in `features/messaging.md` — corrected.
- The "mobile adoption (queued in the Expo backlog)" framing in the Messaging active-feature block — superseded by `in-progress` / code-complete-pending-Ψ.

## Conventions check

- Part 4 (cleanliness): confirmed — sources deleted only after `cmp`-verified archival; no dead links introduced; stale spec field/IDs corrected; the superseded readiness audit and the out-of-scope original rules audit left in place with reasons recorded.
- Part 4a (simplicity): N/A (no abstractions/config introduced — markdown edits).
- Part 4b (adjacent observations): one flagged (the un-archived original messaging rules audit; see "For Mastermind").
- Part 6 (translations): confirmed — banked the no-new-seeds decision; no namespace/key changes authored here.
- Other parts touched: Part 5 (session template, `<n>` numbering — see note below); Part 3 (config-file sole-writer + cross-repo `.agent/` archival exception — both honored).

## Known gaps / TODOs

- §3.6 admin view-token gap is filed, not resolved — awaits a separate Mastermind triage decision (spec-correction vs backend brief).
- Brief 3 lint baseline (80/0) is asserted-not-rendered; Igor confirms on his own `npm run lint` at commit.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — markdown edits to existing files only.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- **Session `<n>` numbering note.** A prior `2026-05-20-oglasino-docs-messaging-1.md` already exists in `sessions/` for the (oglasino-docs, messaging) pair, so this session is numbered `-2` (not `-1`) to preserve per-(repo, slug) uniqueness — the literal "count your own `.agent/` folder" heuristic would have mis-numbered it `1` because the predecessor was archived out of `.agent/`. The engineer build sessions used the slug `messaging-adoption`; this docs close-out continues the `messaging` docs-slug lineage.
- **Adjacent observation (low).** `oglasino-firestore-rules/.agent/audit-messaging.md` (45722 B, 2026-05-20) is the original messaging-feature Phase-2 rules audit and was never archived under that name — it is NOT byte-identical to `sessions/audit-messaging.md` (33953 B, which is a different repo's audit). It predates this adoption work, so I did not archive or delete it (deleting an un-archived record would lose data). Recommend a one-line decision: archive it as e.g. `sessions/2026-05-20-oglasino-firestore-rules-messaging-audit-2.md` or confirm it is an intentional leftover.
- **Closure gate.** This session was opened to apply the chat-B close-out drafts; all are on disk (issues Cycle B flip + §3.6 entry, decisions entry, state row/Risk-Watch/session-log, messaging.md §3.4/§5.11). The only un-applied substantive item is the §3.6 triage, which is correctly filed-not-resolved (needs an upstream drafter — backend brief or spec correction). No pending config-file draft remains un-applied.
