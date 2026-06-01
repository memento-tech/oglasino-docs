# Session summary

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-05-23
**Task:** Audit the full messaging surface on mobile against `features/messaging.md`. Inventory the chat UI, the Firestore read/write patterns, the contract adoption, the trust boundaries, and explicitly catalogue every Firestore read by collection and document. The Firestore-rule check is the highest-priority deliverable.

## Implemented

- Read-only audit producing `.agent/audit-expo-readiness-messaging.md` with all 17 sections per the brief.
- Exhaustive Firestore read inventory (Section 3): 14 read sites catalogued across 5 source files, each classified as owner-only safe or not.
- **Critical finding resolved: no cross-user `users/{otherUserId}` Firestore reads found.** All user-data enrichment goes through the backend endpoint `GET /auth/firebase/{uid}`, not direct Firestore reads. The tightened rules will NOT break mobile messaging.
- Complete write inventory (Section 4): 18 write sites catalogued. No trust-boundary violations found.
- 10 spec-divergence items catalogued where web adopted messaging fixes but mobile has not (atomic existing-chat sends, send-failure toast, `getActiveChat` fallback fix, mark-seen bug, chat-list load-more, etc.).
- 5 hardcoded Serbian strings identified in the messaging surface.
- `NOTMAL` typo confirmed; no current runtime impact (MESSAGE push notifications unwired).

## Files touched

- `.agent/audit-expo-readiness-messaging.md` (new, +350 lines)

## Tests

- No tests run — read-only audit.

## Cleanup performed

none needed — read-only audit.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

nothing — read-only audit.

## Conventions check

- Part 4 (cleanliness): N/A this session.
- Part 4a (simplicity): N/A this session (no code changes).
- Part 4b (adjacent observations): five items flagged in audit "For Mastermind" — dead `checkIsBlocked`, dead `testConnection`, vestigial `FirestoreUser.blocked` field, `loginWithFacebookFirebase` dead code, `var` keyword usage.
- Part 6 (translations): five untranslated Serbian hardcoded strings identified.
- Part 11 (trust boundaries): full Firestore read inventory completed. No release-blocking trust-boundary violations found.
- Other parts touched: Part 7 (error contract) — send-failure UX reviewed, missing toast flagged. Part 8 (architectural defaults) — confirmed shared backend endpoints.

## Known gaps / TODOs

none — all sections completed per brief's definition of done.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit)
  - Considered and rejected: nothing
  - Simplified or removed: nothing

- **HEADLINE: No release-blocking cross-user Firestore reads.** The tightened `users/{userId}` owner-only rule will not break mobile's messaging surface. All user-data enrichment goes through backend's `GET /auth/firebase/{uid}`.

- **10 spec-divergence items** between web's shipped messaging fixes and mobile's current state. These are all items for the future `oglasino-expo-messaging-<n>` adoption chat to consume. Ordered by severity:
  1. `getActiveChat` fallback reads wrong fields from chat root doc (would affect users with >15 chats)
  2. Existing-chat send is not atomic (potential partial-failure data drift)
  3. Mark-seen `seenLocal` mutation before `batch.commit()` (messages stuck unseen on failure)
  4. No send-failure toast (silent failure for users)
  5. No `ChatsWatcher` for temp-state clearing on navigation away
  6. No chat-list "Load more" button (>15 chats inaccessible)
  7. `deleteChat` reorder not applied (ghost removal on failure)
  8. No auto-linkify for URLs
  9. No block badge in chat list
  10. No deleted-user fallback rendering

- **`NOTMAL` typo** — currently harmless but becomes a bug when MESSAGE push notifications are wired. Fix is trivial (one character).

- **5 hardcoded Serbian strings** in the messaging UI violate the four-locale requirement. Need translation key adoption.

- **Forward block field name divergence** — mobile writes `blockedUserId` where spec says `blockerId`. Cosmetic only (redundant with path).

- **Dead code** flagged: `checkIsBlocked`, `testConnection`, `loginWithFacebookFirebase` (commented out), vestigial `blocked: []` field on Firestore user docs.
