# Messaging

The user-to-user messaging feature on Oglasino. Text, images, auto-linkified URLs. Production-ready scope for launch, with deliberate room left for additions post-launch (push notifications, deletion, reactions, moderation, mobile).

This is a single canonical spec. It replaces the planned reference + fix doc shape from the 2026-05-19 audits. It describes the feature end-to-end: data model, security rules, client flows, identity invariants, admin surfaces, validation, and the engineering work to land it. Any change to messaging-related code updates this document in the same session (conventions amendment in this work).

---

## 1. Scope and intent

### 1.1 What this feature is

A user-to-user messaging surface on `oglasino-web`, backed by Firestore for realtime data and Firebase Auth for identity. Users in any base-site cluster can start a conversation with another user, either from a product detail page (carrying product context) or from a user profile page (no product context). Conversations support text messages, image attachments via the existing R2 pipeline, and auto-linkified URLs.

Messages persist forever (deletion deferred). The sender's UI optimistically shows their message; failures surface as a toast. The recipient's UI updates via Firestore snapshot listeners. Block / unblock is server-enforced and prevents message creation. Admin has read-only views.

### 1.2 What this feature is not

Out of scope for this spec. Each becomes its own future Mastermind chat when its time comes.

- **Push notifications for incoming messages.** Separate task per operator decision. The `NotificationCategoryId.MESSAGE` enum value stays in place but unwired.
- **Typing indicators.**
- **Read receipts beyond the single `seen` boolean per message.** No per-character delivery state, no "read at X:XX timestamp" display.
- **Reactions, emoji responses, message edit.**
- **Threading.** Conversations are flat.
- **Message search.**
- **Voice messages, video messages, file attachments other than images.**
- **Message deletion.** Currently messages are immortal (`allow delete: if false`). Future feature will likely add a 5-minute soft-delete window. Left out of v1 to keep scope narrow.
- **Message content moderation.** Today messages bypass the product-style moderation pipeline. A scammer can write `"call me at +381..."` or `"send money to..."` without server intervention. Real concern but a fundamentally different abuse profile from product publishing — folding the existing pipeline into messages now is the kind of decision that compounds badly. Scope it post-launch with real abuse data. The data model in this spec leaves room (server-side hooks at message-create time can be added without changing client contracts).
- **Mobile (Expo).** Mobile catch-up adopts this contract in its own Mastermind chat. Not in launch scope unless the operator decides otherwise. The contracts in this spec are designed to be platform-agnostic.
- **Link previews (Open Graph cards).** Auto-linkify renders URLs as clickable anchors; no preview card.
- **Per-conversation muting / archiving.** Possible later.

### 1.3 Why this spec exists

Three 2026-05-19 audits surfaced a production defect (start-message permission-denied), a latent bug (productId clearing on `MessageInput` mount), several rule-hardening gaps (open notification create, loose chat-doc update, sender-can-flip-seen), and a critical missing capability (push notifications for messages — deferred per operator). They also documented that messaging has worked partially-by-accident since the 2026-04-30 pre-production commit: the rules and the client were never designed against a written contract.

Pre-launch is the window to fix this cleanly. There's no production data to migrate; data-model changes are cheap. The spec captures the contract going forward.

---

## 2. Architecture at one glance

**Web ↔ Firestore directly** for chat data (chat documents, messages subcollections, user sidecars, blocks). The web client uses the Firebase Web SDK; Firestore Security Rules enforce authorization server-side.

**Backend is read-only on chat data** — admin chat-list and message-list endpoints (`/api/secure/admin/chats/**`), trust-review queries, and image upload-token membership checks. Backend never writes to `chats/**`. Backend writes to `notifications/**` for non-message in-app notifications (reviews, favorites) — a different Firestore subtree.

**Identity is Firebase UID everywhere in Firestore.** The backend integer user ID is for backend-internal use only (admin lookups, profile routing, report payloads). Firestore writes never carry backend integer IDs. The two never cross.

**Auth is one source of truth.** Firebase Auth issues the ID token. Web SDK auto-attaches it on every Firestore request. Backend's `FirebaseAuthFilter` verifies the same token on its own endpoints and populates `OglasinoAuthentication.userId` and `OglasinoAuthentication.firebaseUid` (per conventions Part 11). Token revocation fires only from admin ban and user hard-delete.

**Image attachments route through R2** via the existing `POST /api/secure/images/upload-tokens` endpoint with `scope=chat`. Backend verifies the caller is in the chat's `users[]` before issuing the token (server-side trust boundary). The web client then uploads images directly to R2; the resulting `imageKeys` are stored inside `MessageContent.blocks[].imageKeys`.

**Chat IDs are deterministic** from the two participants' Firebase UIDs: `sorted([uid1, uid2]).join('_')`. Same UIDs → same chat ID, always. No "find chat between A and B" query needed; no race on first-message creation; no duplicate chats.

---

## 3. Data model

### 3.1 The chat document — `chats/{chatId}`

A single document per pair of users. Created on first message of a new conversation; updated on every subsequent message (last-message snippet, last-updated timestamp).

```
chats/{chatId}: {
  users: [string, string],     // exactly 2 Firebase UIDs, sorted to match chatId
  createdAt: Timestamp,         // server timestamp on first message
  lastMessage: string,          // short preview text, last message's plain-text content
  lastUpdated: Timestamp        // server timestamp on every message create
}
```

`chatId` itself is computed from the participants: `sorted([uid1, uid2]).join('_')`. The first participant alphabetically goes first; the underscore separator is fixed.

**No `productId` on the chat document.** Product context is per-message, not per-chat — a single conversation may span multiple products if the users continue chatting about other listings later.

**No `participants`, `members`, or `withUser` field on the chat doc.** Only `users[]`. The field name `users` is the contract; the rule enforces it. Rule audit finding 7 confirmed clients today write this shape; the spec locks it.

### 3.2 The message document — `chats/{chatId}/messages/{messageId}`

One document per message. Ordered by `createdAt`. Server-side `seen` flag managed by the recipient's snapshot listener.

```
chats/{chatId}/messages/{messageId}: {
  senderId: string,             // Firebase UID of the writer
  receiverId: string,            // Firebase UID of the other participant
  content: {                     // MessageContent — discriminated blocks
    blocks: [
      { type: 'text', text: string }
      | { type: 'images', imageKeys: string[] }
    ]
  },
  productId?: number,            // OPTIONAL — backend integer product ID if message originated from a product context
  seen: boolean,                 // false on create, flipped to true by receiver's snapshot listener
  createdAt: Timestamp           // server timestamp
}
```

**`productId` is optional and may be present on any message in any conversation.** This is a change from current behavior — see Section 5.3 (productId clearing fix). The spec allows: first message of a new conversation started from a product page carries `productId`; later messages from that conversation about a different product can also carry `productId`; messages started from a user profile page (no product context) carry no `productId`. The rule does not require `productId`.

**Content blocks support text and images only.** The `MessageBlock` type stays the same as today's `src/lib/types/chat/MessageBlock.ts`. Auto-linkified URLs are rendered client-side from text blocks; no separate `link` block type. (Why: keeping the wire shape minimal means future additions don't require schema migration.)

**No `imageUrl` or `mediaUrl` fields.** Image keys go inside `content.blocks`. Resolution of keys to viewable URLs happens client-side via the existing view-token mechanism.

**No editing, no deletion, no edit history.** `seen` is the only mutable field. Rule enforces this — see Section 4.3.

### 3.3 The userchats sidecar — `userchats/{ownerUid}/chats/{chatId}`

Per-user pointer subcollection that powers the chat-list snapshot on `/messages`. Each pointer references the underlying `chats/{chatId}` document and stores per-user state (unread count, last-updated, last-message preview, the counterparty's UID for quick display).

```
userchats/{ownerUid}/chats/{chatId}: {
  chatId: string,                       // same as the path segment, for convenience
  withUserFirebaseUid: string,           // the OTHER participant's Firebase UID
  unreadCount: number,                   // incremented on incoming message, zeroed on read
  lastMessage: string,                   // mirror of chat doc's lastMessage
  lastUpdated: Timestamp                 // mirror of chat doc's lastUpdated
}
```

The sidecar exists because querying "all chats for user X ordered by recency" against the top-level `chats` collection would require either a composite index on `users array-contains + lastUpdated desc` (works but couples list query to participants check), or a denormalized per-user view (this — cleaner, scales).

**The field is `withUserFirebaseUid`, singular, the other participant.** Different field name than the chat doc's `users` array. This asymmetry is intentional (per-user denormalization) but worth documenting because it's a common source of confusion.

**`unreadCount` is owner-tracked, not derived.** When the owner reads messages, their snapshot listener flips `unreadCount` to 0 and flips each unseen message's `seen` to true. When a new message arrives, the sender's `sendMessage` increments the recipient's pointer's `unreadCount` (op 3 of the new-chat batch and the existing-chat batch).

### 3.4 The block model — `userblocks` and `userblocksReverse`

Two-collection forward/reverse index. The forward index lives at the blocker's path; the reverse index lives at the blocked user's path, written by the blocker.

```
userblocks/{ownerUid}/blocked/{blockedUid}: {
  blockedAt: Timestamp,
  reason?: string                // optional — not used in v1 UI but reserved
}

userblocksReverse/{blockedUid}/blockedBy/{blockerUid}: {
  blockedAt: Timestamp
}
```

The forward index is read by the blocker's own UI (`/messages` shows a "Blocked" badge on blocked users) and consulted by the `isBlocked()` rules helper on message create. The reverse index is read by the blocked user's UI (so they can see "you have been blocked by this user" and stop attempting to send messages). The reverse index gates the message-create rule via the same `isBlocked()` helper.

Why two indexes: a single index couldn't be read efficiently from both sides. The blocker needs to list their own blocks; the blocked user needs to know they're blocked. Two collections, each owned by the user who reads it, but cross-written by the blocker (sender writes to receiver's reverse index, which is the cross-user write pattern documented in Section 3.6).

### 3.5 The notification document — `notifications/{firebaseUid}/userNotifications/{notificationId}`

Out of scope for this messaging spec but listed here because the rule audit found it co-resident in the rules file and because the notification rule fix lands in the same engineering brief (Section 9.1).

```
notifications/{firebaseUid}/userNotifications/{notificationId}: {
  type: NotificationCategoryId,        // enum: NAVIGATION | SAVED_PRODUCT | MESSAGE (unwired) | ...
  title: string,
  body: string,
  // ... other fields per category
}
```

Documents written by backend only via `DefaultNotificationsService.sendAsyncNotification`. Read by the recipient client. Used today for review notifications, favorite-by-someone notifications, admin report notifications. Not used for messaging yet (the `MESSAGE` enum value has zero callers).

### 3.6 The cross-user write pattern

The new-chat send is an atomic `writeBatch` that touches four documents at three different ownership boundaries:

1. `chats/{chatId}` — neither user "owns" this in the rules sense; both are in `users[]`.
2. `userchats/{senderUid}/chats/{chatId}` — sender owns.
3. `userchats/{receiverUid}/chats/{chatId}` — **receiver owns; sender writes.** This is the cross-user write.
4. `chats/{chatId}/messages/{messageId}` — neither user "owns"; the rule checks chat membership.

Op 3 is enabled by the userchats rule's second clause: a non-owner write is allowed if the document declares `withUserFirebaseUid` equal to the writer's UID. Translation: when sender writes to receiver's path, the doc must say "the other party (from your perspective, receiver) is the sender." The rule infers ownership intent from the field value.

This is the standard "create-both-sides-of-a-chat" pattern in Firestore. It looks unusual on first read but it's correct and it's documented (rule audit adjacent observation 6 confirmed this is intentional).

---

## 4. Security rules

Lives in the `oglasino-firestore-rules` repo. The current rule file (`firestore.rules`) was bootstrapped on 2026-05-10 and is the source of truth for both the stage Firebase project (`oglasino-stage-49abb`) and the prod project (`oglasino-prod-7e5db`).

### 4.1 Helper functions

Top-level functions, in-scope across all collection matches.

```
function isBlocked(userA, userB) {
  return exists(/databases/$(database)/documents/userblocks/$(userA)/blocked/$(userB))
      || exists(/databases/$(database)/documents/userblocks/$(userB)/blocked/$(userA));
}
```

Used by `chats/{chatId}/messages/{messageId}` create. Two `exists()` checks (forward + reverse). Costs two reads per message-create.

### 4.2 The `chats/{chatId}` document rules

```
match /chats/{chatId} {
  allow read: if request.auth != null &&
    (resource == null || request.auth.uid in resource.data.users);

  allow create: if request.auth != null
    && request.resource.data.users.size() == 2
    && request.auth.uid in request.resource.data.users;

  allow update: if request.auth != null
    && request.auth.uid in resource.data.users
    && request.resource.data.users == resource.data.users;  // FIELD-SHAPE GUARD: users[] is immutable after create

  allow delete: if false;  // chat docs are immortal at v1

  match /messages/{messageId} { ... }
}
```

**Changes from current rules** (resolves rule audit findings 2 and adjacent observation 2):

- **Update rule gains the immutability guard on `users[]`.** Either participant can update `lastMessage`, `lastUpdated`, or other rear-of-document state, but cannot change the participant list. (Closes the "malicious participant kicks the other party" gap.)
- **Delete is hardened to `false`.** Currently the rule was a copy of update; making it explicit prevents accidental loosening. If deletion becomes a feature, it's a deliberate spec change.

Snapshot listener for the chat doc: the `resource == null` short-circuit allows the listener to attach before the doc is created. This is intentional — the new-chat send flow attaches a listener on the (about-to-be-created) chat doc as part of pre-warming the messages view. Without the short-circuit, the listener would throw on attach.

### 4.3 The `chats/{chatId}/messages/{messageId}` subcollection rules

The fix for the production defect lives here.

```
match /messages/{messageId} {
  allow read: if request.auth != null &&
    request.auth.uid in getAfter(/databases/$(database)/documents/chats/$(chatId)).data.users;

  allow create: if request.auth != null
    && request.resource.data.senderId == request.auth.uid
    && request.auth.uid in getAfter(/databases/$(database)/documents/chats/$(chatId)).data.users
    && !isBlocked(request.resource.data.senderId, request.resource.data.receiverId);

  allow update: if request.auth != null
    && request.auth.uid in getAfter(/databases/$(database)/documents/chats/$(chatId)).data.users
    && request.auth.uid == resource.data.receiverId  // only receiver flips seen
    && request.resource.data.seen is bool
    && request.resource.data.senderId == resource.data.senderId
    && request.resource.data.receiverId == resource.data.receiverId
    && request.resource.data.content == resource.data.content
    && request.resource.data.createdAt == resource.data.createdAt;

  allow delete: if false;
}
```

**Changes from current rules** (resolves rule audit findings 8 and adjacent observations 3, 4):

- **`get()` → `getAfter()` on all three rules (read, create, update).** This is the production-defect fix. `getAfter()` evaluates against the post-batch state, so when the new-chat write batch creates the chat doc and the first message atomically, the message-create rule sees the chat doc as existing.
- **Update rule restricts who flips `seen` to the receiver only.** `request.auth.uid == resource.data.receiverId` clause added. (Closes the "sender can mark their own outgoing message seen" gap.)
- **Update rule guards immutability of `senderId`, `receiverId`, `content`, `createdAt`.** Only `seen` is mutable. (Closes the "participant can rewrite message content after the fact" gap.)
- **Delete stays `false`.** Messages are immortal in v1; deletion is a future feature.

### 4.4 The `userchats/{userId}/chats/{chatId}` sidecar rules

```
match /userchats/{userId}/chats/{chatId} {
  allow read: if request.auth != null && request.auth.uid == userId;
  allow delete: if request.auth != null && request.auth.uid == userId;

  allow create, update: if request.auth != null
    && (request.auth.uid == userId
        || request.auth.uid == request.resource.data.withUserFirebaseUid);
}
```

**Changes from current rules**:

- **Read and delete are owner-only.** Currently the rule was shared across `create, read, update, delete` — restricting read and delete to the owner closes a leak where the non-owner could read the other party's sidecar by querying with the right path.
- **Create and update still allow the cross-user write pattern.** When sender writes to `userchats/{receiver}/...`, the doc declares `withUserFirebaseUid == sender.uid` and the rule accepts.

### 4.5 The userblocks rules

```
match /userblocks/{userId}/blocked/{blockedId} {
  allow read: if request.auth != null && request.auth.uid == userId;
  allow create: if request.auth != null
    && request.auth.uid == userId
    && userId != blockedId;  // can't block self
  allow delete: if request.auth != null && request.auth.uid == userId;
}

match /userblocksReverse/{userId}/blockedBy/{blockerId} {
  allow read: if request.auth != null && request.auth.uid == userId;
  allow create, delete: if request.auth != null && request.auth.uid == blockerId;
}
```

Unchanged from current rules. The block/unblock client (`useChatBlockStore.blockUser` and `unblockUser`) uses `writeBatch` to write both indexes atomically.

### 4.6 The notifications rules (folded fix)

Current rule contains a critical bug — `create` allows unauthenticated callers per the `request.auth == null` clause.

```
// FIX from rule audit adjacent observation 1
match /notifications/{userId}/userNotifications/{notificationId} {
  allow read, update, delete: if request.auth != null
    && request.auth.uid == userId;
  allow create: if request.auth != null
    && request.auth.token.admin == true;  // admin custom claim required
}
```

**Change**: replaced `request.auth == null || request.auth.token.admin == true` with `request.auth != null && request.auth.token.admin == true`. Now requires authentication AND admin claim. Backend service accounts that write notifications mint this claim via Firebase Admin SDK (per conventions Part 11). Clients never have this claim and can never create notifications.

This is folded into the same engineering brief as the messaging fix because it's the same file and the same deploy cycle.

### 4.7 The users collection (PII audit gate)

The current rule:

```
match /users/{userId} {
  allow read: if true;   // OPEN — pre-audit
  allow write: if request.auth.uid == userId;
}
```

**Pre-launch action required** (not engineering, but landing it in this spec because it touches the rules deploy): the operator audits what fields live in `users/{userId}` Firestore documents. If only public-profile fields (display name, profile image key), the open read is intentional. If any PII (email, phone, address), the read rule tightens to:

```
allow read: if request.auth != null;
```

Decision deferred to the operator pending the field audit. Engineer in the rules-fix brief gates the change on the operator's answer.

### 4.8 Rule tests

The rules repo has scaffolding for `@firebase/rules-unit-testing` and `vitest` but **zero tests** (rule audit finding 9). The rules-fix engineering brief seeds the test suite with a minimum set covering the messaging surface. Each test runs against the rules emulator with `firebase-tools` and either asserts a permission grant or a denial.

Minimum test set:

1. **Chat create as participant** — passes.
2. **Chat create with self in `users[]` twice** — fails (`users.size() == 2` is satisfied numerically; the rule allows it; this test documents a known limitation and is `expected: pass` with a comment).
3. **Chat create as non-participant** — fails (auth.uid not in users[]).
4. **Chat read as participant** — passes.
5. **Chat read as non-participant** — fails.
6. **Chat update mutating `users[]`** — fails (immutability guard).
7. **Message create with parent chat existing, as participant, not blocked** — passes.
8. **Message create with parent chat NOT existing (atomic batched write)** — passes (the new-chat happy path; `getAfter()` sees in-flight write).
9. **Message create from non-participant** — fails.
10. **Message create when sender has blocked receiver** — fails.
11. **Message create when receiver has blocked sender** — fails.
12. **Message update flipping `seen` as receiver** — passes.
13. **Message update flipping `seen` as sender** — fails (only receiver may).
14. **Message update mutating `content`** — fails (immutability).
15. **Notification create as authed user without admin claim** — fails (auth.token.admin not true).
16. **Notification create as unauthed user** — fails (closes the bug).
17. **Notification create as admin custom token** — passes.

Test files live in `tests/messaging.test.ts` etc. CI workflow (`validate-pr.yml`) runs `npm test`; the gate becomes meaningful instead of trivially-passing.

---

## 5. Client flows

### 5.1 Identity and auth setup

Firebase Auth on web (`firebaseClient.ts`) auto-attaches the ID token on every Firestore request. The token is the same one verified by backend's `FirebaseAuthFilter`. Token refresh is automatic via Firebase Web SDK.

`useAuthStore.user` carries the resolved user. `useAuthResolved()` is the gate hook for "Firebase ready AND backend sync complete." Components that subscribe to Firestore on mount should gate on `useAuthResolved` to avoid the user-id-less window flagged in the GA4 spec.

### 5.2 The new-chat flow — start message from product page

1. User on product detail page clicks the Message button on `ProductFunctions`.
2. `StartMessageButton.onClick`:
   - If unauthenticated → open login dialog. Stop.
   - If blocking the seller or blocked by the seller → open block info dialog. Stop.
   - Otherwise:
     - `useChatStore.setTempReceiver(seller)` — sets `tempReceiver` to the seller's `UserInfoDTO`.
     - **`useChatStore.setTempProductContext(product)`** — replaces today's `setTempProductReason`. The new method writes to a separate store slice that `sendMessage` reads synchronously. (See Section 5.3 for why this rename matters.)
     - `router.push('/messages')` — navigate to the messages page.
3. `/messages` page (`Messages` component) detects `tempReceiver` is set with no existing chat:
   - Computes `chatId = sorted([currentUser.firebaseUid, tempReceiver.firebaseUid]).join('_')`.
   - Renders `MessageInput` with a pre-populated suggestion based on the product (uses `tempProductContext.name` if set, generic if not).
   - **Does NOT clear `tempProductContext` on mount.** This is the productId clearing fix. See Section 5.3.
4. User edits/accepts the suggestion, clicks Send.
5. `MessageInput.send` builds a `MessageContent` (text block, possibly images block) and calls `onSend(content)`.
6. `useChatStore.sendMessage(chatId, content)`:
   - **Reads `tempProductContext` synchronously from the store at the moment of send** (not from a closure or a stale ref).
   - Builds the `MessageRequest`: `{ content, senderId, receiverId, productId?, seen: false, createdAt }`. `productId` is included if `tempProductContext` is set.
   - Issues a `writeBatch` with four operations:
     - `set(chats/{chatId}, { users: [senderUid, receiverUid], createdAt: serverTimestamp(), lastMessage, lastUpdated: serverTimestamp() })`.
     - `set(userchats/{senderUid}/chats/{chatId}, { chatId, withUserFirebaseUid: receiverUid, unreadCount: 0, lastMessage, lastUpdated: serverTimestamp() })`.
     - `set(userchats/{receiverUid}/chats/{chatId}, { chatId, withUserFirebaseUid: senderUid, unreadCount: 1, lastMessage, lastUpdated: serverTimestamp() })`.
     - `set(chats/{chatId}/messages/{messageId}, MessageRequest)`.
   - `await batch.commit()` — all four ops succeed or all fail atomically.
   - On success: clear `tempReceiver` and `tempProductContext`. The conversation is now an "existing chat" for all subsequent messages.
   - On failure: roll back optimistic UI insert; **surface a toast to the user**. (See Section 5.5.)

The batch is atomic because `getAfter()` in the messages create rule (Section 4.3) sees the chat doc creation that's part of the same batch. The defect fix.

### 5.3 The productId clearing fix

**Current bug** (rule audit finding 6 + web audit hypothesis 4 + 2026-05-17 `issues.md` entry): `MessageInput.tsx:57-68` has a mount effect that synchronously calls `setTempProductReason(undefined)` after reading the product name into the suggestion text. By the time `sendMessage` runs later, `tempProductReason` is undefined and `productId` is omitted from the message.

**Fix**: refactor the temp product context handling.

- Rename `tempProductReason` → `tempProductContext` for clarity. (Same shape: `ProductDetailsDTO | null`.)
- Rename setter accordingly: `setTempProductContext`. Type signature accepts `ProductDetailsDTO | null` only (not `undefined`).
- `MessageInput` mount effect **no longer clears the context**. It reads the product name once to pre-populate the suggestion; the store slice stays set.
- `useChatStore.sendMessage` reads `state.tempProductContext` synchronously at the top of the function (before any optimistic UI work). The value is captured as a local const for the rest of the function.
- After a successful send (chat created, message landed), `sendMessage` clears `tempProductContext` to null. Subsequent messages in the same chat don't carry stale context.

**Alternative considered and rejected**: plumb `productId` through `MessageContent` (add a metadata object to the content shape, threaded from `StartMessageButton` through to `sendMessage`). This is cleaner architecturally because it eliminates the transient store slice entirely. Rejected for v1 because it touches more files for the same outcome; the rename + sync-read approach achieves the same correctness with smaller surface area. **Mark as a candidate for v2 cleanup** if the messaging code is revisited.

### 5.4 The existing-chat flow — send subsequent message

1. User on `/messages` with an active chat clicks Send.
2. `useChatStore.sendMessage(chatId, content)`:
   - Reads `tempProductContext` synchronously (may be null for an established chat).
   - Builds the `MessageRequest`.
   - **Uses `writeBatch` (not three independent writes)** to update three documents atomically:
     - `set(chats/{chatId}, { lastMessage, lastUpdated: serverTimestamp() }, { merge: true })`.
     - `update(userchats/{senderUid}/chats/{chatId}, { lastMessage, lastUpdated, /* unreadCount unchanged for sender */ })`.
     - `update(userchats/{receiverUid}/chats/{chatId}, { lastMessage, lastUpdated, unreadCount: increment(1) })`.
     - `set(chats/{chatId}/messages/{messageId}, MessageRequest)`.

**Change from current behavior** (web audit Section 2): existing-chat path is now atomic. Today's path uses three independent writes (rule audit finding 2 confirmed this), which means a partial failure leaves one pointer updated and the other stale. The fix is one line — wrap in `writeBatch`.

`getAfter()` in the rules handles the read-side correctly. The message create rule sees the (possibly-pre-existing) chat doc and (now-updated) chat doc identically.

### 5.5 Send failure UX

Currently `useChatStore.ts:581-602` swallows send failures with `console.error` only. Optimistic UI rolls back; user sees their message disappear with no explanation.

**Fix**: add a toast notification on the catch.

- Use the existing `notify.error()` toast pattern from `oglasinoToast.ts` (or equivalent).
- Toast text translated via the `MESSAGES_PAGE` namespace: a new key `messages.send.failed.toast` ("Failed to send message. Try again.") seeded in all four locales.
- The catch block still does the optimistic rollback (filtering out the temp message). After rollback, fires the toast.
- For image uploads that failed mid-batch: also surface, ideally including a "Retry" affordance. v1 can skip the retry button and just show the toast — the user retypes/retaps Send to retry.

**Why this matters**: today's silent failure is exactly what users see during the permission-denied bug. They think the platform is broken and they bounce. The toast doesn't fix the underlying bug but it tells the user what's happening.

### 5.6 Snapshot listeners

Three always-on listeners attach when `useAuthStore.user` flips from null → set (via `ChatsInit` mounting in the `[locale]` layout):

1. `userchats/{user.firebaseUid}/chats` — orderBy `lastUpdated desc`, limit 15. Powers the chat-list sidebar and the unread badge in the header. Rule allows: owner-only read.
2. `userblocks/{user.firebaseUid}/blocked` — for the blocker's own list. Rule allows: owner-only read.
3. `userblocksReverse/{user.firebaseUid}/blockedBy` — for "you have been blocked by" awareness. Rule allows: owner-only read.

One lazy listener attaches when a chat is opened:

4. `chats/{chatId}/messages` — orderBy `createdAt desc`, limit 15. Powers the active message stream. Rule allows: any participant.

All four listeners gate on `useAuthResolved()` to avoid the user-id-less window.

The first three listeners run on every page — including product detail pages — because the `[locale]` layout owns them. They are the most likely source of Error 1 in the audited symptom, though under the fixed rules (with `getAfter()` and the immutability guards) they should not fire `permission-denied` for any authenticated user.

### 5.7 Link auto-linkify

Add to the message rendering layer.

- Use `linkify-it` (npm package, small, well-maintained, zero dependencies). Already-installed alternatives may exist; engineer audits before adding a new dep.
- In `Message.tsx` where text blocks render, replace the plain text node with a mix of text segments and `<a>` tags.
- Anchor attributes: `target="_blank"`, `rel="noopener noreferrer"`. The `noreferrer` is important — without it the linked page can `window.opener.location = ...` and phishing-redirect the messaging tab.
- Styling: anchors use the existing link styling from the design system. No new CSS.
- Display: show the URL as-is, not as a "click here" mask. **What you see is what you click.** Reduces phishing surface.
- No preview cards. URLs render as inline links only.

Edge cases:

- Crafted URLs that look like one domain but are another (`https://oglasino.com.evil.example/...`). The `linkify-it` library detects the URL boundary correctly; the user sees the full URL.
- `javascript:` URLs. `linkify-it` doesn't match these by default; verify in the engineer brief.
- Unicode / IDN domains (`https://оgla...` with Cyrillic 'о'). Linkify detects; render as-is. Phishing concern remains but at the level a URL-aware user can spot.

### 5.8 Block / unblock flow

Unchanged from current behavior except for the rule audit confirming the existing rules are correct (Section 4.5).

- User clicks "Block" on the chat header dropdown or user profile.
- `useChatBlockStore.blockUser(blockedUid)` issues a `writeBatch`:
  - `set(userblocks/{currentUid}/blocked/{blockedUid}, { blockedAt: serverTimestamp() })`.
  - `set(userblocksReverse/{blockedUid}/blockedBy/{currentUid}, { blockedAt: serverTimestamp() })`.
- The next message-send attempt from either side fails at the `isBlocked()` rule check. The send-failure toast fires (Section 5.5).
- Unblock: delete both index entries.

### 5.9 Delete chat (one-sided soft delete)

Unchanged from current behavior. `deleteChat` deletes only `userchats/{currentUid}/chats/{chatId}` — not the chat doc, not the messages, not the other user's pointer. The chat persists on the other side; the current user just doesn't see it in their list.

The audited UX is intentional. Documenting it here so future readers don't "fix" it into a destructive operation.

### 5.10 Image attachments

Unchanged from current behavior except the underlying `POST /api/secure/images/upload-tokens` flow already routes through backend's membership check (rule audit Section 1). Web's `MessageInput` continues to call `uploadImages(files, 'chat', chatId, ...)` and stores the resulting `imageKeys` inside `MessageContent.blocks[].imageKeys`.

`MessageImages.tsx` renders attached images using view tokens for the private R2 bucket. View token fetch is gated on chat membership server-side (backend issues the token after Firestore `isUserMemberOfChat` check). The 401 fallback on the `<img>` element retries with a fresh token.

---

## 6. Backend involvement

### 6.1 What backend does for messaging today

- Verifies Firebase ID tokens (`FirebaseAuthFilter` on all `/api/secure/**`).
- Issues R2 upload tokens for chat-scope images (`POST /api/secure/images/upload-tokens` + `scope=chat`).
- Issues R2 view tokens for chat-scope images.
- Reads chat data for admin views (`/api/secure/admin/chats/**`).
- Reads chat data for trust-review queries (out-of-scope-for-this-feature, but exists).

### 6.2 What backend does NOT do

- Write to `chats/{chatId}` or any descendant.
- Mint custom Firebase tokens.
- Maintain Firestore mirrors of user data (users live in Postgres on backend; Firestore `users/{uid}` collection is separately maintained client-side or out of scope).
- Trigger push notifications for messages. (Future feature — `NotificationCategoryId.MESSAGE` enum exists with zero callers; spec leaves it unwired.)

### 6.3 Chat ID computation

`DefaultTrustReviewService.getChatId(uidA, uidB)` in `oglasino-backend` computes `sorted([uidA, uidB]).join('_')` and is the canonical backend implementation. Mobile, when it arrives, will need its own implementation. **Consider extracting to a shared package post-launch** if the count of clients grows. v1: each client re-implements the same one-line scheme. The spec is the canonical reference.

### 6.4 Admin endpoints

Read-only.

- `GET /api/secure/admin/chats/user/{userId}/chats` — list a given user's chats, paginated by `lastUpdated`. Uses `DefaultFirebaseChatService`.
- `GET /api/secure/admin/chats/{chatId}/messages` — list a chat's messages, paginated by `createdAt`.

Both endpoints require `ROLE_ADMIN`. Used by `oglasino-web` admin views in `app/[locale]/admin/chats/**`.

No changes in this spec.

### 6.5 Trust-review surface

`DefaultTrustReviewService` reads `chats/{sorted_uid_a_uid_b}/messages` to compute review eligibility. Used by other features (reviews). Not changed by this spec.

A recent fix (commit `f03ea20` on 2026-05-17) corrected a self-reference bug in `getChatId(currentUid, currentUid)` → `getChatId(currentUid, targetUid)`. Documented here for completeness; not in this spec's scope.

---

## 7. Trust boundaries

Per conventions Part 11.

For every field on a Firestore document written by the client, the spec states whether it is **trusted from the client**, **server-derived**, or **read from another authoritative store**.

### 7.1 Chat document write (`chats/{chatId}`)

- `users[]`: **trusted from client + enforced by rule.** Rule checks `request.auth.uid in users` and `users.size() == 2`. The client supplies both UIDs; the rule enforces the writer is one of them but does **not** verify the *other* UID is a real user or consents to be added. Known limitation; trade-off accepted (matches the existing block model where any user can be a target).
- `createdAt`, `lastUpdated`: server-derived via `serverTimestamp()`.
- `lastMessage`: client-supplied free text (mirror of the message content's first text block, truncated). Not used for security decisions.

### 7.2 Userchats sidecar write (`userchats/{ownerUid}/chats/{chatId}`)

- Path `ownerUid`: enforced by rule. Either the writer owns the path, or the doc's `withUserFirebaseUid` declares the writer as the other party (cross-user write pattern).
- `withUserFirebaseUid`: trusted from client + enforced by rule (specifically for the cross-user write case).
- `chatId`, `unreadCount`, `lastMessage`, `lastUpdated`: not used for security decisions.

### 7.3 Message document write (`chats/{chatId}/messages/{messageId}`)

- `senderId`: **trusted from client + enforced by rule.** Rule checks `senderId == request.auth.uid`. Client cannot impersonate.
- `receiverId`: trusted from client. Not server-verified — the rule only checks the sender, not that the receiver is in `users[]`. This is a known limitation; in practice, the client only ever writes the other participant's UID, but a malicious client could write any UID. Impact: limited because the receiver doesn't trust the `receiverId` field — they discover messages by being in `chats/{chatId}/messages` via their `users[]` membership. The block check (`isBlocked(senderId, receiverId)`) uses both; an attacker could route a message past a block by misrepresenting `receiverId`. **Worth tightening post-launch**: add a rule clause `receiverId in getAfter(chats/$chatId).data.users && receiverId != senderId`.
- `productId`: client-supplied if a product-context message; not used for security decisions.
- `content`: client-supplied free text and image keys. Not validated by rules for now (moderation is a separate future feature per Section 1.2).
- `seen`: created as `false`. On update, rule enforces receiver-only flip.
- `createdAt`: server-derived via `serverTimestamp()`.

### 7.4 Block index writes (`userblocks/**`, `userblocksReverse/**`)

- Forward index path: enforced — only the blocker writes.
- Reverse index path: enforced — only the blocker writes.
- Self-block prevented by rule (`userId != blockedId`).

### 7.5 Image upload (R2)

- Image keys are server-derived (backend issues them as part of `POST /api/secure/images/upload-tokens`).
- Chat membership is server-verified (`isUserMemberOfChat` reads `chats/{chatId}.users` server-side).
- Upload URL is one-shot, scoped to the issued key.

### 7.6 What is NOT a trust boundary today

- **Content moderation.** Message text is not screened for banned words, scam patterns, or phishing. Future feature (Section 1.2 out-of-scope item).
- **Link safety.** Auto-linkified URLs are not screened against URL blocklists. Future concern, especially as the link surface scales.
- **Rate limiting.** Firestore has no application-level rate limiter on chat writes. A user can flood another user with messages (limited only by Firestore's own per-document write throttling). Future concern.

---

## 8. Translations

Per conventions Part 6.

New translation keys for the spec's scope. Namespace: `MESSAGES_PAGE` (existing).

- `messages.send.failed.toast` — error toast on send failure.
- `messages.send.image.upload.failed.toast` — error toast on image upload failure mid-send (sub-case of above; may use the same generic message).
- `messages.product.context.initial.suggestion.tpl` — template for the auto-populated first-message suggestion when started from a product (existing key may be reused; engineer audits).

All four locales (EN, SR, RU, CNR) seeded per conventions Part 6 Rule 3.

---

## 9. Engineering work

One feature, multiple briefs. Sequenced.

### 9.1 Brief 1 — Firestore rules fixes (`oglasino-firestore-rules`)

Foundation. Everything downstream depends on the rules being correct.

- `get()` → `getAfter()` on all three messages subcollection rules (read, create, update).
- `users[]` immutability guard on the chat update rule.
- Receiver-only `seen` flip + field-shape immutability on the message update rule.
- Lock the notification create rule (`request.auth != null && request.auth.token.admin == true`).
- Tighten userchats read/delete to owner-only.
- Gate the users-collection read tightening on operator's PII audit (Section 4.7).
- Seed the rule test suite (17 tests per Section 4.8) in `tests/messaging.test.ts`.
- Verify `npm test` runs the new tests; verify `validate-pr.yml` workflow respects the gate.
- Deploy to stage via the `deploy-stage.yml` workflow (push to `stage` branch).

Estimated effort: 1-2 days for an engineer familiar with Firestore rules.

### 9.2 Brief 2 — Web `sendMessage` and store refactor (`oglasino-web`)

- Rename `tempProductReason` → `tempProductContext` in `useChatStore`, all consumers, and `MessageInput`.
- Remove the `setTempProductContext(undefined)` mount effect in `MessageInput`.
- `useChatStore.sendMessage`: read `tempProductContext` synchronously at the top of the function; capture as a local const for the duration.
- After successful send, clear `tempProductContext` to null.
- Existing-chat path: wrap the three writes (chat doc merge, two userchats pointers, message create) in a single `writeBatch`.
- Add `notify.error()` toast call in the catch block of `sendMessage`. Use the new translation key.
- Verify behavior: start new chat from product → message carries `productId`; start chat from user profile → message has no `productId`; send subsequent messages → all atomic; failed sends → toast.

Estimated effort: 1-2 days. Includes tests.

### 9.3 Brief 3 — Auto-linkify in message rendering (`oglasino-web`)

- Add `linkify-it` dependency (or verify existing alternative).
- Modify `Message.tsx`'s text-block rendering to detect URLs and emit `<a>` tags with appropriate attributes.
- Style anchors per the existing design system.
- Tests: text without URLs renders as plain; text with a URL renders a clickable anchor; the URL displayed matches the URL clicked; `javascript:` URLs are not linkified.
- Manual verification: paste various URL forms (with and without scheme, IDN, query strings) and confirm correct rendering.

Estimated effort: half a day to a day.

### 9.4 Brief 4 — Translation seeding (`oglasino-backend`)

- Append new keys to the existing translation seed SQL files in all four locales per conventions Part 6 Rule 3.
- Verify keys resolve in development with the appropriate locale.

Estimated effort: 1-2 hours.

### 9.5 Brief 5 — Documentation and convention amendment (`oglasino-docs`)

- This spec exists on disk at `oglasino-docs/future/messaging.md`.
- Conventions amendment (drafted in this Mastermind chat) extends the update-on-change discipline to this spec.
- Closing `decisions.md` entry documents the spec creation and the eight seam resolutions.
- `state.md` updates: Messaging feature as active; the deferred items added to backlog.
- `issues.md` housekeeping: close entries that resolve via this work; flag entries that don't.

Estimated effort: handled by Docs/QA in the closeout brief from this Mastermind chat.

### 9.6 Total

Brief 1: 1-2 days. Brief 2: 1-2 days. Brief 3: 0.5-1 day. Brief 4: 1-2 hours. Brief 5: closeout work.

**Realistic range: 3-5 engineer-days.** Plus Docs/QA closeout. With Claude AI engineer agents (per the operator's prior measurement), shorter.

---

## 10. Validation

After each brief lands.

- **Brief 1**: rule emulator tests pass (the new test suite). Stage deploy succeeds via `deploy-stage.yml`. Operator manually verifies a start-message from a product page succeeds on stage. Operator manually verifies a non-participant cannot read a chat. Operator manually verifies notification create with no auth fails.
- **Brief 2**: web test suite passes. Start a new chat from product → first message has `productId` in Firestore. Send a subsequent message → existing-chat batch writes all four docs atomically. Trigger a failure (e.g., temporarily revoke a rule) → toast appears.
- **Brief 3**: text-only messages render unchanged. URL-containing messages render anchors. Click an anchor → opens in new tab with `noopener noreferrer`. `javascript:` URLs are not linkified.
- **Brief 4**: all four locales render the new keys.
- **Brief 5**: spec is on disk at `oglasino-docs/future/messaging.md`. Conventions amendment is on disk in `meta/conventions.md`. `decisions.md` carries the closing entry.

---

## 11. Out of scope, captured for the post-launch backlog

Each becomes its own future Mastermind chat.

### 11.1 Push notifications for incoming messages

Operator's separate task per intake. Wire `NotificationCategoryId.MESSAGE` through the existing backend `DefaultNotificationsService.sendAsyncNotification`. Triggered either by a backend endpoint called by the sender post-Firestore-commit, or by a Cloud Function listening on `chats/{chatId}/messages` writes. Estimated 3-5 days backend + 1 day web once decided.

### 11.2 Message deletion

5-minute soft-delete window. Sender can delete their own message within 5 minutes of send. Rule: `allow update` on messages adds a clause `request.auth.uid == resource.data.senderId && resource.data.createdAt > timestamp.value(now() - duration.value(5, 'm'))` for the deleted-flag flip. Renders as "Message deleted" client-side. Hard delete is a separate decision. Estimated 2-3 days.

### 11.3 Message content moderation

Banned words, spam patterns, scam detection, link safety. Different abuse profile from product moderation; needs its own design with real abuse data. Estimated 1-2 weeks once designed.

### 11.4 Mobile adoption (`oglasino-expo`)

Mobile catch-up adopts this spec's contracts. Same chat ID scheme, same field names, same rule expectations. Mobile uses Firebase JS SDK or Firebase native SDKs. Listener and write paths translate directly from web. Estimated 1-2 weeks once the operator decides.

### 11.5 Receiver-in-`users[]` rule check

Tighten the message create rule to additionally verify `receiverId in getAfter(chats/$chatId).data.users && receiverId != senderId`. Closes the trust-boundary gap on `receiverId` (Section 7.3). One-line rule change post-launch.

### 11.6 Rate limiting on chat writes

Firestore-level rate limiting is limited. Application-level rate limiter via a Firestore counter document or via Firebase Cloud Functions. Estimated 3-5 days.

### 11.7 `productId` plumbing via `MessageContent`

Alternative architecture mentioned in Section 5.3: thread `productId` through `MessageContent` rather than via a transient store slice. Cleaner. Punt for v1; revisit during a future messaging refactor.

### 11.8 Link previews (Open Graph cards)

Render small preview cards for linked pages. Requires server-side fetch of OG metadata (caching, rate limits, security). Estimated 3-5 days.

### 11.9 Search

In-conversation search and global search across all conversations. Requires Firestore indexes or a separate search service. Estimated 1 week.

### 11.10 Typing indicators

Real-time "user is typing" affordance. Requires Firestore presence or a separate WebSocket/SSE channel. Estimated 3-5 days.

### 11.11 Reactions, edit, threading

Each is a feature in its own right. Sequential additions, not bundled.

### 11.12 Chat ID computation extracted to a shared package

When the count of client implementations grows (web + mobile + future), extract `sorted([uidA, uidB]).join('_')` to a shared package. v1 is fine with three independent implementations (web inline, backend inline, mobile inline).

### 11.13 Users-collection PII audit

Operator audits what fields live in `users/{firebaseUid}` Firestore documents. If PII, tighten the users-collection read rule. Section 4.7.

### 11.14 Soft-deletion / archival of old chats

Currently `deleteChat` is one-sided soft-delete. A user can't archive a chat without losing visibility. Archival semantics worth considering.

### 11.15 Read receipts beyond `seen`

Per-message "read at X:XX" display. Currently `seen` is a boolean; no timestamp. Future addition if user feedback wants it.

---

## 12. Closure criteria

This feature is shipped when:

1. All 4 engineering briefs (Sections 9.1 through 9.4) are merged.
2. The validation steps in Section 10 pass.
3. The post-launch backlog (Section 11) entries are in `state.md`.
4. The conventions amendment (update-on-change discipline) is applied.
5. The closing `decisions.md` entry is applied (this spec creation, seam resolutions, eight-question architectural decisions).
6. The `issues.md` entries that close on this work are flipped to `fixed`:
   - 2026-05-17 `tempProductReason` clearing → closed by Brief 2.
   - 2026-05-14 silent text-only-send failure → closed by Brief 2.
   - 2026-05-14 inert Report dropdown (already fixed; reaffirming) → already closed.
7. Stage Firebase deployment carries the fixed rules. Production deployment is operator's decision (pre-launch the operator presumably deploys before opening traffic).
8. The session-closure gate per conventions Part 3 is met for this Mastermind chat.

---

## 13. Assumptions and open items

- Assumes the conventions amendment for messaging update-on-change discipline is applied before any messaging code change merges.
- Assumes the operator audits the `users/{firebaseUid}` Firestore collection contents before the rules-fix brief deploys; the read-rule tightening lands accordingly.
- Assumes the rules repo CLAUDE.md question (untracked file, audit Section 1) is resolved by Mastermind decision separately from this spec.
- Assumes `getAfter()` rule behavior is supported by the current Firestore rules version in use (`rules_version = '2'`). Confirmed in Firebase docs; engineer re-verifies during Brief 1.
- Assumes `linkify-it` (or chosen alternative) is acceptable to add as a web dependency; engineer in Brief 3 audits package size and license.
- Assumes the spec lives in `oglasino-docs/future/messaging.md` per the operator's folder-semantics rule (`features/` = finished, `future/` = unfinished including in-progress). The folder rule is codified in a closing `decisions.md` entry from this Mastermind chat.