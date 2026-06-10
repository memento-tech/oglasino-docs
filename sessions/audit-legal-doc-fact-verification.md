# AUDIT — Legal-document fact verification (Privacy Policy vs Firestore Rules)

**Repo:** oglasino-firestore-rules
**Date:** 2026-06-10
**Type:** READ-ONLY audit. No rules/config/test changes made.
**Verification discipline:** every factual statement cited to file + line, verified twice — direct Read of `firestore.rules` AND `rg` confirmation.

**Artifacts audited:**
- `firestore.rules` (3817 bytes, dated Jun 2)
- `firebase.json`, `.firebaserc`, `firestore.indexes.json`, `package.json`

---

## Q12(a) — Soft-delete read asymmetry (PP §8.3)

**Claim under test:** during the 7-day soft-deletion window, existing messages stay readable by the OTHER participant while the soft-deleted user is locked out (can neither read nor write).

**VERDICT: NOT PRODUCED BY THE RULES — depends on backend/Auth disabling the account.**

**Evidence:**
- The rules contain no soft-delete concept. `rg -in "soft|delete[d]?At|deactiv|disabl|status|active|isDeleted"` over `firestore.rules` → **NONE**.
- Chat read — `firestore.rules:43-44`:
  ```
  allow read: if request.auth != null &&
    (resource == null || request.auth.uid in resource.data.users);
  ```
- Message read — `firestore.rules:53-54`:
  ```
  allow read: if request.auth != null
    && request.auth.uid in getAfter(/databases/$(database)/documents/chats/$(chatId)).data.users;
  ```
- `users` array is immutable on update — `firestore.rules:50`: `request.resource.data.users == resource.data.users`.
- Chat delete forbidden — `firestore.rules:51`: `allow delete: if false;`.

**Reasoning:** Both participants are gated by the identical condition `request.auth.uid in users`. Because `users` is immutable and the chat cannot be deleted, a soft-deleted user's UID cannot be removed from `users`, so that user would still satisfy every read condition. The rules do not distinguish a soft-deleted user from an active one. The claimed asymmetry can only arise if Firebase **Auth disables the soft-deleted account** → `request.auth` becomes null → `request.auth != null` fails for that user only, while the other (still-authenticated, still-in-`users`) participant keeps reading.

**Plain answer:** No — the rules alone do not produce the asymmetry; the other participant keeps access purely via the immutable `users` array, and the soft-deleted user is locked out only if the backend disables their Firebase Auth account, not by any rule.

---

## Q12(b) — Notification isolation

**Claim under test:** a client can read only their OWN notification documents — no cross-user reads.

**VERDICT: YES — own-only.**

**Evidence — `firestore.rules:75-82`:**
```
match /notifications/{userId}/userNotifications/{notificationId} {
  allow read, delete: if request.auth != null
    && request.auth.uid == userId;
  allow update: if request.auth != null
    && request.auth.uid == userId
    && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['seen'])
    && request.resource.data.seen is bool;
  allow create: if false;
}
```
Path is keyed by `{userId}`; read requires `request.auth.uid == userId`. Update is restricted to the same owner and to a `seen`-only diff; create is `if false`.

**Plain answer:** Yes — a client can read only notification documents under its own UID; cross-user notification reads are denied.

---

## Q12(c) — Region config (PP §4 "EU region" for Authentication)

**Claim under test:** is the Firestore location ID pinned in any repo file? Is there any config touching Firebase Auth regionalization?

**VERDICT: NOT FOUND — console check required. Auth region not determinable from this repo.**

**Evidence:**
- `rg -in "location|region|eu|europe|nam5|asia"` over `firebase.json`, `.firebaserc`, `firestore.indexes.json`, `package.json` → **NONE**.
- `firebase.json` — Firestore rules/indexes paths + emulator config only; no `locationId`.
- `.firebaserc` — project aliases only: `stage` → `oglasino-stage-49abb`, `prod` → `oglasino-prod-7e5db`. No location.
- Deploy scripts (`package.json`) — `firebase deploy --only firestore --project {stage|prod}`; no `--location`.
- No Firebase Auth configuration exists anywhere in this repo (it holds only Firestore rules/indexes).

**Plain answer:** No file in this repo pins the Firestore location ID or any Auth region; both are console-side settings and must be verified in the Firebase console. The PP §4 "EU region" claim for Authentication is not determinable from this repo.

---

## Q12(d) — Admin / moderator access path to messages (PP §2.5)

**Claim under test:** do the rules grant an admin/moderator read path to message contents, and how is admin status determined?

**VERDICT: NO rule-level admin path. Any moderation access is via the Admin SDK, which bypasses rules.**

**Evidence:**
- `rg -in "admin|moderator|role|token\.|claim"` over `firestore.rules` → **empty (no matches)**. No custom-claim check, no UID-list, no role field anywhere.
- Message reads are strictly participant-gated — `firestore.rules:53-54` (quoted in Q12a): the only path to read a message is `request.auth.uid in chats/$(chatId).data.users`. A non-participant admin is denied by default-deny.

**Plain answer:** The rules contain no admin/moderator access path to message contents; any PP §2.5 moderation access must run through the Firebase Admin SDK, which bypasses Security Rules entirely and is not granted or observable in these rules.

---

## Trust boundaries (Part 11)

No violations found. Every rule trusts `request.auth.uid` (Firebase-signed, unforgeable) against either the immutable `users` array or the `{userId}` path segment. No rule trusts an unprotected Firestore field as an authorization source.

---

## Adjacent findings (out of scope — not fixed)

1. **Message-read cost** — `firestore.rules:54` — each message read calls `getAfter()` on the parent chat doc, billing one extra document read per message read. `getAfter` vs `get` is equivalent on a pure read (no pending chat write). Severity: **low** (cost/cosmetic).
2. **`firestore-debug.log` in repo root** — 99 KB emulator artifact dated Jun 9; check `.gitignore`. Severity: **low**.
3. **`.agent/last-session.md` was empty (0 bytes)** from the prior session before this audit repopulated it. Severity: **low**.
4. **Chat read allows `resource == null`** — `firestore.rules:44` permits read of a non-existent chat doc to any authenticated user; harmless (a missing doc returns nothing). Severity: **low**.

---

## Headline for Mastermind / Backend

PP §8.3's soft-delete lockout of the deleted user is **not enforced by Firestore rules** — both participants are gated identically on the immutable `users` array. If the backend does not disable the soft-deleted user's Firebase Auth account (or revoke their tokens), that user retains full read/write to their chats and messages throughout the 7-day window. Confirm the disable mechanism with the Backend agent, or the PP wording overstates the rule-level guarantee.
