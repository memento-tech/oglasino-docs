# Session summary — oglasino-firestore-rules — legal-document fact verification (audit)

**Date:** 2026-06-10
**Slug:** legal-doc-audit (`-1`, first session for this slug in this repo)
**Session type:** READ-ONLY AUDIT. No rules/config/test changes. Output = answers + this summary.

## Task
Verify four Privacy-Policy claims against the *current* `firestore.rules` and repo config:
(a) soft-delete read asymmetry (PP §8.3), (b) notification isolation, (c) Firestore/Auth region config (PP §4), (d) admin message-access path (PP §2.5). Every factual statement cited to file+line and verified twice (direct Read + `rg`).

## Findings (verdicts)

**(a) Soft-delete read asymmetry — NOT PRODUCED BY RULES.**
The rules have no soft-delete concept (`rg "soft|deleted|disabl|status|isDeleted"` over `firestore.rules` → NONE). Access is gated solely on membership in the immutable `users` array:
- Chat read — `firestore.rules:43-44`: `request.auth.uid in resource.data.users`.
- Message read — `firestore.rules:53-54`: `request.auth.uid in getAfter(...chats/$(chatId)).data.users`.
- `users` is immutable on update (`firestore.rules:50`) and chat delete is `if false` (`firestore.rules:51`), so a soft-deleted UID cannot be removed from `users`.
Both participants satisfy the identical `uid in users` condition. A soft-deleted user whose UID remains in `users` would still pass every read. The PP §8.3 asymmetry can only come from Firebase **Auth disabling** the soft-deleted account (→ `request.auth == null` → `request.auth != null` fails for that user only). The rules alone do not lock the soft-deleted user out.

**(b) Notification isolation — YES, own-only.**
`firestore.rules:75-77`: path keyed by `{userId}`; `allow read, delete: if request.auth != null && request.auth.uid == userId`. Update restricted to same owner + `seen`-only diff (lines 78-81); create `if false` (line 82). No cross-user read path.

**(c) Region config — NOT FOUND (console check).**
`rg "location|region|eu|europe|nam5|asia"` over `firebase.json`, `.firebaserc`, `firestore.indexes.json`, `package.json` → NONE. No `locationId` pinned. `.firebaserc` holds project aliases only (`stage` → `oglasino-stage-49abb`, `prod` → `oglasino-prod-7e5db`). No deploy script pins `--location`. **Auth region (PP §4 "EU region") not determinable from this repo** — no Auth config exists here (Firestore-rules repo only).

**(d) Admin message-access path — NONE in rules; Admin SDK bypass.**
`rg "admin|moderator|role|token\.|claim"` over `firestore.rules` → empty. No custom-claim or UID-list check anywhere. Message reads strictly participant-gated (`firestore.rules:53-54`); a non-participant admin is default-denied. Any PP §2.5 moderation access to message contents must run through the Firebase Admin SDK, which bypasses Security Rules and is not visible here.

## Adjacent findings (out of scope, not fixed)
- `firestore.rules:54` — message read calls `getAfter()` on parent chat, billing one extra read per message read; `getAfter` vs `get` equivalent on a pure read. Severity low (cost/cosmetic).
- `firestore-debug.log` (99 KB) present in repo root — emulator artifact; check `.gitignore`. Severity low.
- `.agent/last-session.md` was empty (0 bytes) from prior session; repopulated this session. Severity low.
- `firestore.rules:44` — chat read allows `resource == null` (read of non-existent doc) to any authed user; harmless. Severity low.

## Trust boundaries (Part 11)
No violations. Every rule trusts `request.auth.uid` (Firebase-signed, unforgeable) against the immutable `users` array or the `{userId}` path segment. No rule trusts an unprotected Firestore field as an authorization source.

## For Mastermind

### Brief vs reality
1. **PP §8.3 soft-delete asymmetry attributed to rules**
   - Brief/PP says: existing messages stay readable by the OTHER participant while the soft-deleted user is locked out (implying rule-enforced).
   - Rules say: no soft-delete logic exists; both participants gated identically on the immutable `users` array (`firestore.rules:43-44`, `53-54`, `50`, `51`).
   - Why this matters: the claimed lockout of the soft-deleted user is NOT enforced by Firestore rules. If the backend does not disable the account's Firebase Auth, a soft-deleted user retains full read/write to their chats and messages. The PP wording could overstate what the rules guarantee.
   - Recommended resolution: confirm with Backend agent that soft-delete disables the Firebase Auth account (or removes session tokens). If lockout must be rule-enforceable, a soft-delete flag check would be required — but that needs a rule-protected field and a brief.

### Part 4a evidence
- Rules added: none (read-only audit).
- Rules considered and rejected: none.
- Rules simplified: none.

### Config-file impact
None. No edits to `firestore.rules`, `firestore.indexes.json`, `firebase.json`, `.firebaserc`, `package.json`, `tsconfig.json`, or tests.

### Cleanup performed
None (audit only). Noted `firestore-debug.log` and empty `last-session.md` as adjacent findings.

### Obsoleted by this session
None.

### Conventions check
- No commits/pushes/merges/deploys/emulator-against-real-project. ✅
- No cross-repo edits (oglasino-docs not reachable; not touched). ✅
- No edits beyond session summaries. ✅ (READ-ONLY brief)
- `npm test` not run — no code change made; running it would be the only side effect, and the brief forbids modifications. Tests unchanged and presumed green from prior session. Flagging that I deliberately did not run it given the read-only scope.

## Known gaps
- Firestore location ID and Auth region must be verified in the Firebase console — not determinable from this repo (see Q12c).
- Whether soft-delete lockout is enforced (via Auth account disable) is a Backend-repo question (see Brief vs reality #1).
