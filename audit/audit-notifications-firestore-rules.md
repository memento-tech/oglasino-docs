# Audit — Notifications (Firestore Rules)

**Repo:** `oglasino-firestore-rules`
**Branch:** `stage`
**Date:** 2026-06-02
**Type:** Phase-2 read-only audit. No code changed.
**Brief:** `.agent/brief.md` — notifications feature (in-app + push); Igor suspects a bug in the notifications-related Firestore rules. The code is ground truth.

Files read for this audit: `firestore.rules` (every rule + the one global helper), `firebase.json`, `firestore.indexes.json`, `tests/messaging.test.ts` (notification block + `seedNotification` helper). Baseline: `npx firebase emulators:exec --project demo-oglasino "npx vitest run"` → **68/68 pass**.

---

## 1. Every rule touching a notification path (verbatim)

There is exactly **one** notification rule in the file. There is **no** top-level `notifications/{notificationId}` collection, and **no** notification-related subcollection under `users/{userId}`. Grep for `notif` across the rules returns only this block.

`firestore.rules:74–79`:

```
/* NOTIFICATIONS */
match /notifications/{userId}/userNotifications/{notificationId} {
  allow read, update, delete: if request.auth != null
    && request.auth.uid == userId;
  allow create: if false;
}
```

Path shape: `notifications/{userId}/userNotifications/{notificationId}`.

- `{userId}` is the **recipient** — the user the notifications belong to. It is a path segment, so the recipient identity is the document's location, not a field inside the document.
- `{notificationId}` is the individual notification doc id.

The only adjacent rule that matters for the PII question (§5) is the users rule, `firestore.rules:9–11`:

```
match /users/{userId} {
  allow read, write: if request.auth != null && request.auth.uid == userId;
}
```

---

## 2. Who can READ / CREATE / UPDATE / DELETE per path

Path `notifications/{userId}/userNotifications/{notificationId}`:

| Operation | Condition (verbatim) | Who that is |
|-----------|----------------------|-------------|
| **read**   | `request.auth != null && request.auth.uid == userId` | The recipient only. The `{userId}` path segment must equal the caller's uid. |
| **create** | `if false` | Nobody via client SDK. Admin SDK bypasses rules entirely. |
| **update** | `request.auth != null && request.auth.uid == userId` | The recipient only (e.g. mark-as-read). |
| **delete** | `request.auth != null && request.auth.uid == userId` | The recipient only. |

Trust basis (Part 11): all three allowed operations trust `request.auth.uid` (a Firebase-verified, client-unforgeable identity) compared against the `{userId}` path segment. There is no Firestore-field trust here, no custom-claim trust, no `get()`/`exists()` cross-document read. This is the cheapest and safest possible trust basis — no read-cost amplification, no field-protection dependency.

Test coverage confirms each row (`tests/messaging.test.ts`):
- read by owner pass (test 40 :376), read by non-owner fail (test 41 :384), read unauthenticated fail (test 46 :428)
- create as anonymous fail (test 16 :354), create as authed owner fail (test 17 :365) — i.e. even the recipient cannot create
- update by owner pass (test 42 :392), update by non-owner fail (test 43 :402)
- delete by owner pass (test 44 :412), delete by non-owner fail (test 45 :420)

---

## 3. The suspected bug — is it too open, or too closed?

**Finding: the rule is NOT buggy in either direction. The security posture is correct.** Both hypothesized failure modes in the brief are checked and neither applies.

**Too open?** No.
- `create: if false` — no client (anonymous or authenticated) can write a notification into anyone's tree, including their own. Rules out forged/spam notifications from clients (tests 16, 17 confirm).
- `read/update/delete` are gated to `request.auth.uid == userId`, so **no user can touch another user's notifications.** Bob cannot read, mark-read, or delete Alice's notifications (tests 41, 43, 45 confirm), and an anonymous caller gets nothing (test 46). There is no any-authenticated-user read or write path.

**Too closed?** No.
- The brief's specific worry — "create is `if false` AND there's no read/update path, so the recipient can't read or mark-as-read their own notifications" — does **not** hold here, because read and update **are** allowed for the recipient. The recipient can read their notifications (test 40) and flip `seen` to mark-as-read (test 42). The `if false` on create only blocks client-side creation; it does not strand the recipient, because the recipient never needs to create — the backend does (§4).

So the rule lands exactly on the textbook pattern for server-written / user-read notifications: **server creates (Admin SDK), recipient reads + marks-read + deletes, nobody else sees anything.** It is internally consistent and matched by the tests.

**One genuine soft spot (not the "bug," but worth flagging) — the UPDATE rule has no field constraints.** Contrast the messages update rule (`firestore.rules:62–70`), which pins `senderId`, `receiverId`, `content`, `createdAt`, `productId` to their stored values and only lets `seen` change. The notifications update rule allows the recipient to rewrite **any** field to **any** value — title, body, createdAt, not just `seen`. 

- **Why this is low severity, not high:** the only person who can do this is the recipient, and the only data they can corrupt is their **own** private notifications. There is no cross-user blast radius and no trust-boundary violation — a user tampering with what they themselves will see is self-inflicted. It is not a security hole.
- **Why it's still worth noting:** if the notifications feature ever relies on the integrity of `createdAt` (e.g. ordering, "new since" badges) or on `title`/`body` being server-authored, a client could locally rewrite them. If the spec wants notification content to be immutable-once-written and only `seen` togglable, the update rule should be tightened to lock the non-`seen` fields (mirroring the messages pattern). That is a spec decision — flagged for Mastermind, not fixed here (read-only audit).

---

## 4. How notifications are written vs. read / marked-read — is it consistent?

Yes, consistent, and the create-side assumption the brief names is correct.

- **Write path:** `create: if false` means no client SDK can create a notification. The only writer that can populate `notifications/{uid}/userNotifications/*` is the **Firebase Admin SDK**, which runs in the backend with full privileges and **bypasses security rules entirely**. This is the intended and only viable writer. (The test suite simulates exactly this: `seedNotification` at `tests/messaging.test.ts:99` writes via `testEnv.withSecurityRulesDisabled(...)` — i.e. the rules-bypassing path that stands in for Admin SDK.)
- **Read path:** the recipient reads its own subtree via the `read` allow (`uid == userId`).
- **Mark-as-read path:** the recipient updates the doc (typically `seen: false → true`) via the `update` allow. The test `seedNotification` doc shape (`title`, `body`, `seen: false`, `createdAt`) and test 42's `update({ seen: true })` show the expected mark-as-read mechanic.
- **Delete path:** the recipient can delete its own notifications (`delete` allow).

So the three roles are coherent: **backend (Admin SDK) is the sole creator; the recipient is the sole reader/marker/deleter; no one else has any access.** The `if false` create does not contradict the read/update model — it is what makes the model safe (clients can't forge notifications), and the recipient's read+update give them everything they need to consume and clear them.

---

## 5. `fcmToken` / push-token exposure (PII)

**No push-token field appears anywhere in this repo.** A repo-wide grep for `fcm`, `fcmToken`, `pushToken`, `push_token`, `deviceToken` across `*.rules`, `*.ts`, `*.json` returns **zero** matches in the rules and tests (only an unrelated npm `update-notifier` package-lock hit). So the rules layer does not name, read, or gate any push token today.

What that means for the PII concern:

- **If push tokens are stored on `users/{userId}`** (a common pattern): the users rule (`firestore.rules:10`) is `allow read, write: if request.auth != null && request.auth.uid == userId` — **owner-only read.** A user can only read their own user doc, so an fcmToken stored there would be readable **only by its owner**, not by other users. **No cross-user PII exposure** in that case. (Caveat: that same rule also lets the owner `write` any field on their own user doc with no field validation — so if the backend, not the client, is meant to own the fcmToken field, a malicious owner could overwrite/clear their own token. Self-inflicted only; pre-existing and outside notifications scope, noted for completeness.)
- **If push tokens are stored in a separate collection** (e.g. `deviceTokens/*`, `fcmTokens/*`): there is **no rule for it**, so default-deny applies — no client can read or write it at all. The backend Admin SDK would read/write it, bypassing rules. Also safe from client exposure.
- **Either way, the rules do not leak push tokens to other users.** The open question is purely *where* the token lives, which this repo cannot answer — see SEAMS.

---

## 6. SEAMS — what the rules layer assumes, and what to verify

The rules are one side of three contracts. Each assumption below needs confirmation against the other repos before the notifications feature is called done.

1. **Backend (Admin SDK) is the sole notification writer.** The rule (`create: if false`) only makes sense if the backend writes notifications to `notifications/{recipientUid}/userNotifications/{autoId}` via the Firebase Admin SDK. **Verify:** `oglasino-backend` actually writes to this exact path (collection name `notifications`, subcollection `userNotifications`, recipient uid as the parent doc id) via Admin SDK. If the backend writes a different path (e.g. top-level `notifications/{id}` with a `recipientUid` field), the current rule does not cover it and those docs would be default-denied to the client.

2. **The recipient is keyed by Firebase uid in the path, not by a field.** The rule trusts `request.auth.uid == {userId path segment}`. **Verify:** the client reads/queries `notifications/{currentUserUid}/userNotifications` using the Firebase uid (not a backend numeric `userId` or any other id). A mismatch between the Firebase uid and whatever id the backend uses as the parent segment would lock the recipient out (would *look* like the "too closed" bug, but the root cause would be an id-namespace seam, not the rule).

3. **Notification document shape is not enforced by rules.** Create is server-side (no client schema to enforce) and update has no field validation (§3). The rules neither require nor protect `title` / `body` / `seen` / `createdAt`. **Verify:** the client tolerates whatever the backend writes, and the spec is comfortable that the recipient can mutate any field (or decides to tighten the update rule). The test seed shape (`title`, `body`, `seen`, `createdAt`) is a test fixture, **not** a contract the rules guarantee.

4. **Push-token storage location is unknown from this repo (§5).** **Verify:** where `oglasino-backend` / `oglasino-expo` store the FCM/push token, and confirm the governing rule there is owner-only-read (if on `users/{userId}`) or default-denied (if a separate collection). This audit can only assert that *this repo's* rules don't expose it.

5. **No notification index exists.** `firestore.indexes.json` defines indexes only for `chats` and `messages` — none for `userNotifications`. A simple per-user list ordered by a single field (e.g. `createdAt DESC`) is served by Firestore's automatic single-field indexes, so no composite index is needed for that. But a **compound** query — e.g. `where('seen','==',false).orderBy('createdAt','desc')` for an unread feed/badge — would require a composite index that is **not** present. **Verify:** the client's notification query shape. If it filters + orders, an index is owed (a separate, brief-authorized change — out of scope for this read-only audit).

---

## Summary for Mastermind

- **No security bug in the notifications rule.** `create: if false` (Admin-SDK-only writes) + recipient-only `read`/`update`/`delete` is the correct, internally-consistent pattern. Neither the "too open" nor the "too closed" hypothesis holds: no user can touch another user's notifications, and the recipient *can* read and mark-as-read their own. 9 tests (16, 17, 40–46) exercise every cell of the access matrix and pass.
- **Adjacent observation (low, in scope of the suspicion):** the `update` rule has **no field constraints** (`firestore.rules:76`), unlike the messages update rule. The recipient can rewrite any field (title/body/createdAt), not just toggle `seen`. Not a security hole (self-only blast radius), but if the spec wants notification content immutable-once-written, the rule should pin the non-`seen` fields. **I did not change this — read-only audit.**
- **Top seams to confirm before "done":** (a) backend writes to exactly `notifications/{firebaseUid}/userNotifications/{id}` via Admin SDK; (b) recipient is keyed by Firebase uid, not a backend id; (c) push-token storage location + its read rule live in backend/expo, not here; (d) if the client runs a `seen`-filtered + `createdAt`-ordered query, a composite index on `userNotifications` is owed and currently absent.
- **Config-file impact:** none. This is an audit deliverable (`.agent/audit-notifications.md`); the rules, tests, indexes, and config are unchanged.
