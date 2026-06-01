# Spec validation — messaging Firestore rules (Brief 2c)

**Branch:** `stage`
**Mode:** READ-ONLY — no rule edits, no test edits, no deploys, no git changes.
**Date:** 2026-05-30
**Validated against:** committed `firestore.rules`, `tests/messaging.test.ts`, `tests/users.test.ts`, `package.json`
**Spec source:** `../oglasino-docs/features/messaging.md` §4

---

## Verdict on claim 6 (load-bearing)

**MATCHES — all three message-subcollection rules use `getAfter()`, none reverted to `get()`.**
read → `firestore.rules:54`, create → `firestore.rules:57`, update → `firestore.rules:63`. Zero `get(` calls against the parent chat doc anywhere in the messages block.

---

## Claim-by-claim

### 1. `/users/{userId}` owner-only (spec §4.2) — **MATCHES**
`firestore.rules:10` → `allow read, write: if request.auth != null && request.auth.uid == userId;`
It is **not** `allow read: if true`. PII path is owner-gated as specified.

### 2. `/chats/{chatId}` read with `resource == null` short-circuit (spec §4.3) — **MATCHES**
`firestore.rules:43-44` → `allow read: if request.auth != null && (resource == null || request.auth.uid in resource.data.users);`
The `resource == null` short-circuit (subscribe-before-exists) is present and exactly matches the spec.

### 3. `/chats/{chatId}` create (spec §4.3) — **MATCHES**
`firestore.rules:45-47` → requires `request.resource.data.users.size() == 2` **and** `request.auth.uid in request.resource.data.users`.

### 4. `/chats/{chatId}` update freezes `users[]` (spec §4.3) — **MATCHES**
`firestore.rules:48-50` → `request.auth.uid in resource.data.users && request.resource.data.users == resource.data.users`.
Participant-list immutability enforced; updater must already be a participant.

### 5. `/chats/{chatId}` delete is `if false` (spec §4.3) — **MATCHES**
`firestore.rules:51` → `allow delete: if false;`

### 6. Messages use `getAfter()` on read, create, update — NOT `get()` (spec §4.4) — **MATCHES**
- read: `firestore.rules:54` → `request.auth.uid in getAfter(/databases/$(database)/documents/chats/$(chatId)).data.users`
- create: `firestore.rules:57` → `request.auth.uid in getAfter(/databases/$(database)/documents/chats/$(chatId)).data.users`
- update: `firestore.rules:63` → `request.auth.uid in getAfter(/databases/$(database)/documents/chats/$(chatId)).data.users`

All three are `getAfter(`. The production-defect fix is intact. No reversion to `get()`.

### 7. Message-create gating + `isBlocked` both directions (spec §4.4) — **MATCHES**
Create rule `firestore.rules:55-61`:
- `request.resource.data.senderId == request.auth.uid` (`:56`)
- sender in `getAfter(...).data.users` (`:57`)
- `!isBlocked(request.resource.data.senderId, request.resource.data.receiverId)` (`:58-61`)

`isBlocked` helper `firestore.rules:4-7` reads the **forward** index in **both** directions:
`exists(/userblocks/$(userA)/blocked/$(userB)) || exists(/userblocks/$(userB)/blocked/$(userA))`. As specified (reverse index not consulted).

### 8. Message update — receiver-only `seen` flip + field freezes via `get('productId', null)` (spec §4.4) — **MATCHES**
Update rule `firestore.rules:62-70`:
- receiver-only: `request.auth.uid == resource.data.receiverId` (`:64`)
- `seen is bool` (`:65`)
- freezes `senderId` (`:66`), `receiverId` (`:67`), `content` (`:68`), `createdAt` (`:69`)
- `productId` frozen via `request.resource.data.get('productId', null) == resource.data.get('productId', null)` (`:70`) — the `get(field, null)` form is used on **both** sides, not plain `==`. Handles absent-on-both-sides as required for mobile's no-`productId` messages.

### 9. Userchats cross-user write (spec §4.5) — **MATCHES**
`/userchats/{userId}/chats/{chatId}`:
- create + update (`firestore.rules:18-20`) → `request.auth.uid == userId || request.auth.uid == request.resource.data.withUserFirebaseUid`. **Both clauses present.**
- read owner-only (`:16`), delete owner-only (`:17`).
This authorizes mobile's receiver-sidecar write in the new-chat batch exactly.
Test coverage confirms both branches: `tests/messaging.test.ts:298` (matching `withUserFirebaseUid` → pass) and `tests/messaging.test.ts:312` (mismatched → fail).

### 10. Notifications create is `if false` (spec §4.7) — **MATCHES**
`/notifications/{userId}/userNotifications/{notificationId}`:
- `allow create: if false;` (`firestore.rules:78`)
- read, update, delete owner-only: `request.auth != null && request.auth.uid == userId` (`firestore.rules:76-77`)

### 11. `receiverId` NOT verified against `users[]` at create — residual gap still open (spec §4.4 / §7.3) — **MATCHES**
The message-create rule (`firestore.rules:55-61`) contains **no** `receiverId in getAfter(...).data.users` clause. The documented residual trust-boundary gap on `receiverId` is **still present** — it was not tightened. (Confirming the gap exists as documented; nothing changed for mobile.)

### 12. Test suite (spec §4.8) — **MATCHES**
- `tests/messaging.test.ts` exists; `tests/users.test.ts` exists.
- `package.json:8` → `"test": "vitest run"` — **no `--passWithNoTests`** flag. Confirmed.
- **Actual test count: 24 `it()` blocks** — 22 in `tests/messaging.test.ts`, 2 in `tests/users.test.ts`. Spec §4.8 lists 23; the extra is case "24" (`tests/messaging.test.ts:232`, receiver flips `seen` on a message with no `productId` field — the Brief 1b absent-`productId` guard). "Roughly 23" holds; the live count is 24 and matches §10.1's "24 rule tests passing."
- `getAfter()` atomic-create regression guard (§4.8 case 7) **present**: `tests/messaging.test.ts:143` — `"7. atomic chat + first-message create as participant → pass (getAfter regression guard)"`.

---

## Drift summary

**No DRIFTED claims. No NOT_FOUND claims.** All 12 claims MATCH the committed rules and tests.

The one cosmetic note (not drift): the test count is **24**, not the "~23" the brief estimated — the extra is the documented Brief 1b absent-`productId` guard (`tests/messaging.test.ts:232`). This is additive coverage, not a discrepancy against any rule mobile must satisfy.

**Bottom line for mobile:** the shipped rules match `messaging.md` §4 exactly. Mobile can mirror the documented write patterns without risk of building against drifted rules. In particular: `getAfter()` ×3 is intact (atomic new-chat batch will pass), the userchats cross-user write clause holds exactly (receiver-sidecar write authorized), `productId` uses the absent-safe `get(...,null)` comparison, and the `receiverId`-not-verified gap is still open (mobile must continue writing the correct `receiverId`, as the web client does).

---

## Cleanup performed

None needed — read-only session. No rule, test, config, or git changes made.
