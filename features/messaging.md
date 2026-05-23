# Messaging

Production-ready user-to-user messaging on Oglasino. Text, images, auto-linkified URLs, block/unblock, admin read-only view, scheduled cleanup of hard-deleted users' chat data.

This spec describes the feature as it will be after the engineering work in Phase 5. It reflects the audited reality (three Phase 2 audits dated 2026-05-20) plus the seam resolutions from Phase 3. It is the canonical reference for the messaging surface going forward; any change to messaging-related code updates this document in the same session.

Spec lives at `oglasino-docs/features/messaging.md` per the existing convention (features in progress and shipped both live in `features/`).

---

## 1. Scope

### 1.1 In scope

- Firestore Security Rules hardening for the messaging surface and the notification create rule (rules repo closure).
- Web client send-flow fixes: atomic batches, `productId` preservation, send-failure toast, `getActiveChat` fallback fix, navigation-aware temp state cleanup.
- Auto-linkify URLs in message text (URL displayed = URL clicked; no masking, no preview cards).
- Chat list "load more" affordance.
- Admin chat-list receiver bug fix.
- Polish bundle: hardcoded literals to translation keys, locale fix in admin date formatting, dead exports removed, block badge in chat list, `deleteChat` reorder, mark-seen failure handling.
- Sunday cron job that anonymizes hard-deleted users' chat data.
- Removal of the public `/api/public/notification/test` endpoint.

### 1.2 Out of scope

Each becomes its own future Mastermind chat.

- Push notifications for incoming messages. The `NotificationCategoryId.MESSAGE` enum value stays unwired.
- Admin welcome chat for new users (single designated admin, system chat that can't be blocked or removed). Design notes below in §13.
- Message deletion (currently messages are immortal). Future feature; likely a 5-minute soft-delete window.
- Message content moderation. Different abuse profile from product moderation; needs its own design with real abuse data.
- Mobile (`oglasino-expo`) adoption. Mobile catch-up in its own chat post-merge.
- Typing indicators, read receipts beyond `seen`, reactions, edit, threading, search, voice/video messages.
- Link previews (Open Graph cards).
- Rate limiting on chat writes.
- Per-conversation mute/archive.
- Receiver-in-`users[]` rule tightening (closes a residual trust-boundary gap on `receiverId`; documented in §7.3).

---

## 2. Architecture at one glance

**Web ↔ Firestore directly** for chat data (chat documents, messages subcollections, user sidecars, blocks). The web client uses the Firebase Web SDK; Firestore Security Rules enforce authorization server-side.

**Backend is read-only on chat data.** Admin chat-list and message-list endpoints, trust-review queries, image upload-token chat-membership checks, and the Sunday cleanup cron. Backend never writes to `chats/**` except for the cron's anonymization passes (and those are Admin SDK writes that bypass rules).

**Identity is Firebase UID everywhere in Firestore.** The backend integer user ID is for backend-internal use only (Postgres FKs, admin lookups, profile routing). Firestore writes never carry backend integer IDs.

**Auth is one source of truth.** Firebase Auth issues the ID token. Web SDK auto-attaches it on every Firestore request. Backend's `FirebaseAuthFilter` verifies the same token on its own endpoints and populates `OglasinoAuthentication` with both `userId` (Postgres) and `firebaseUid` (Firebase). Per conventions Part 11.

**Image attachments route through R2** via `POST /api/secure/images/upload-tokens` with `scope=chat`. Backend verifies the caller is in the chat's `users[]` before issuing the token. The web client then uploads images directly to R2; the resulting `imageKeys` are stored inside `MessageContent.blocks[].imageKeys`.

**Chat IDs are deterministic** from the two participants' Firebase UIDs: `sorted([uid1, uid2]).join('_')`. Same UIDs → same chat ID. No "find chat between A and B" query needed; no race on first-message creation; no duplicate chats. This convention is shared between web and backend (`DefaultTrustReviewService.getChatId`); diverging from it silently breaks trust-review.

---

## 3. Data model

### 3.1 Chat document — `chats/{chatId}`

One document per pair of users. Created on first message; updated on every subsequent message.

```
chats/{chatId}: {
  users: [string, string],     // exactly 2 Firebase UIDs, sorted to match chatId
  createdAt: Timestamp,         // server timestamp on first message
  lastMessage: string,          // short preview, last message's plain text or sentinel for image-only
  lastUpdated: Timestamp        // server timestamp on every message create
}
```

`chatId` is computed as `sorted([uid1, uid2]).join('_')`. Web computes at `useChatStore.ts:380` and `:683`. Backend computes at `DefaultTrustReviewService.java:114-116`. Algorithm matches.

No `productId` on the chat document. Product context is per-message, not per-chat — a single conversation may span multiple products.

No `participants`, `members`, `withUser`, or `withUserFirebaseUid` field on the chat root doc. Only `users[]`. The chat root doc and the userchats sidecar (§3.3) use different field names; clients write the correct shape to each.

### 3.2 Message document — `chats/{chatId}/messages/{messageId}`

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
  productId?: number,            // OPTIONAL — backend integer product ID if message has product context
  seen: boolean,                 // false on create, flipped to true by receiver
  createdAt: Timestamp           // server timestamp
}
```

`productId` is optional and may appear on any message in any conversation:

- First message of a new conversation from a product page → carries `productId`.
- Later messages in that conversation, even about a different product if the users navigate back to messages from another product page → can carry `productId`.
- Messages started from a user profile page (no product context) → no `productId`.

Content blocks support text and images only. Image rendering uses the existing view-token mechanism (§3.6). Auto-linkified URLs are rendered client-side from text blocks; no separate `link` block type.

`seen` is the only mutable field after create. Rules enforce this (§4.3). Only the receiver may flip `seen`. Sender, anyone else, admin — none can mutate.

No editing, no deletion, no edit history.

### 3.3 Userchats sidecar — `userchats/{ownerUid}/chats/{chatId}`

Per-user pointer subcollection. Powers the chat-list snapshot on `/messages`.

```
userchats/{ownerUid}/chats/{chatId}: {
  chatId: string,                       // redundant with path; convenience
  withUserFirebaseUid: string,           // OTHER participant's Firebase UID
  unreadCount: number,                   // incremented on incoming message, zeroed on read
  lastMessage: string,
  lastUpdated: Timestamp
}
```

The sidecar exists because querying "all chats for user X ordered by recency" against the top-level `chats` collection would require an index on `users array-contains + lastUpdated desc`. The sidecar denormalizes that read.

The field is `withUserFirebaseUid` (singular, the other participant). The chat root doc uses `users` (array). The asymmetry is intentional but a known foot-gun for refactor.

`unreadCount` is owner-tracked, not derived. When the owner reads messages, their snapshot listener zeroes `unreadCount` and flips each unseen message's `seen` to true in one batch. When a new message arrives, the sender's `sendMessage` increments the recipient's `unreadCount`.

### 3.4 Block indexes — `userblocks` and `userblocksReverse`

Two-collection forward/reverse index.

```
userblocks/{ownerUid}/blocked/{blockedUid}: {
  blockerId: string,          // ownerUid; redundant with path; convenience
  createdAt: Timestamp
}

userblocksReverse/{blockedUid}/blockedBy/{blockerUid}: {
  blockerId: string,          // blockerUid; redundant with path; convenience
  createdAt: Timestamp
}
```

Each index has a distinct consumer:

- **Forward (`userblocks/A/blocked/B`)**: read by the `isBlocked()` rule helper for message-create gating, and read by A's own UI to show their blocked-users list.
- **Reverse (`userblocksReverse/B/blockedBy/A`)**: read by B's UI to know they have been blocked by A — drives the "you have been blocked" banner in the chat header and the disabled input.

The blocker writes to both atomically via `writeBatch` (forward + reverse). Unblock deletes both atomically.

`isBlocked()` reads only the forward index, in both directions: `exists(/userblocks/A/blocked/B) || exists(/userblocks/B/blocked/A)`. The reverse index is NOT consulted by `isBlocked()`. The two indexes have different consumers; both are required.

### 3.5 Notification document — `notifications/{firebaseUid}/userNotifications/{notificationId}`

Out of scope for messaging itself, but the create rule is fixed in this feature (§4.7). The collection is written by backend only via `DefaultNotificationsService.sendAsyncNotification`, which uses the Firebase Admin SDK (bypasses rules). Used today for review notifications, favorite-by-someone notifications, admin report notifications. Not used for messaging (deferred per §1.2).

### 3.6 Image attachment storage

Image keys live inside `MessageContent.blocks[]` of type `images`. Resolution of keys to viewable URLs happens client-side via the view-token mechanism:

- Upload: `POST /api/secure/images/upload-tokens` with `{ scope: 'chat', count, contentTypes, chatId }`. Backend verifies chat membership before issuing tokens. Tokens authorize a single PUT to a specific R2 key (`private/chats/{chatId}/{uuid}.{ext}`).
- View: `POST /api/secure/images/view-tokens` with `{ scope: 'chat', chatId }`. Backend re-verifies chat membership. Token authorizes GETs scoped to the `private/chats/{chatId}/` prefix. TTL is short; client caches view tokens with pre-expiry refresh.
- Admin viewing of chat images uses the same view-token endpoint; backend additionally permits admins to mint view tokens for arbitrary chats.

### 3.7 Cross-user write pattern

The new-chat send is an atomic `writeBatch` touching four documents at three ownership boundaries:

1. `chats/{chatId}` — neither user "owns"; both are in `users[]`.
2. `userchats/{senderUid}/chats/{chatId}` — sender owns.
3. `userchats/{receiverUid}/chats/{chatId}` — **receiver owns; sender writes.** Cross-user write.
4. `chats/{chatId}/messages/{messageId}` — neither user "owns"; rule checks chat membership.

Op 3 is enabled by the userchats rule's secondary clause (§4.5): a non-owner write is permitted if the doc declares `withUserFirebaseUid` equal to the writer's UID. When sender writes to receiver's path, the doc says "the other party (from receiver's perspective) is the sender." The rule infers ownership intent from the field value.

The existing-chat send also uses `writeBatch` to update three documents atomically (chat root merge, both sidecars).

---

## 4. Firestore Security Rules

Lives in `oglasino-firestore-rules`. The fixes below are deployed to the stage project (`oglasino-stage-49abb`) first via the `deploy-stage.yml` workflow; prod (`oglasino-prod-7e5db`) deploys via `deploy-main.yml` after stage verification.

### 4.1 Helper functions

```
function isBlocked(userA, userB) {
  return exists(/databases/$(database)/documents/userblocks/$(userA)/blocked/$(userB))
      || exists(/databases/$(database)/documents/userblocks/$(userB)/blocked/$(userA));
}
```

Used only by `chats/{chatId}/messages/{messageId}` create. Two `exists()` reads per message-create.

### 4.2 `/users/{userId}` — tightened to owner-only

Pre-launch, the rule was `allow read: if true`. The Firestore docs at this path carry PII (email, fcmToken, displayName, profileImageKey) — world-readable was a privacy leak. Fixed:

```
match /users/{userId} {
  allow read, write: if request.auth != null && request.auth.uid == userId;
}
```

No code in `oglasino-web` or `oglasino-backend` reads from `users/{userId}` in Firestore (audits confirmed). The mobile app may or may not — out of scope; if mobile reads cross-user, mobile breaks and is fixed in the mobile catch-up chat.

Adjacent: the `blocked` field present on `users/{userId}` documents looks vestigial — no code reads it. Logged separately in `issues.md` for investigation.

### 4.3 `/chats/{chatId}` document rules

```
match /chats/{chatId} {
  allow read: if request.auth != null &&
    (resource == null || request.auth.uid in resource.data.users);

  allow create: if request.auth != null
    && request.resource.data.users.size() == 2
    && request.auth.uid in request.resource.data.users;

  allow update: if request.auth != null
    && request.auth.uid in resource.data.users
    && request.resource.data.users == resource.data.users;  // immutability

  allow delete: if false;

  match /messages/{messageId} { ... }
}
```

Changes from pre-fix rules:

- **Update rule freezes `users[]`.** Either participant can update `lastMessage`/`lastUpdated`/etc., but cannot mutate the participant list.
- **Delete is hardened to `false`.** Cascade-orphaning of the `messages` subcollection (rule audit observation 10) is no longer possible from the client side. If deletion ever becomes a feature, it's a deliberate spec change.

The `resource == null` short-circuit on read remains — supports the "subscribe before doc exists" pattern.

### 4.4 `/chats/{chatId}/messages/{messageId}` subcollection rules

```
match /messages/{messageId} {
  allow read: if request.auth != null
    && request.auth.uid in getAfter(/databases/$(database)/documents/chats/$(chatId)).data.users;

  allow create: if request.auth != null
    && request.resource.data.senderId == request.auth.uid
    && request.auth.uid in getAfter(/databases/$(database)/documents/chats/$(chatId)).data.users
    && !isBlocked(request.resource.data.senderId, request.resource.data.receiverId);

  allow update: if request.auth != null
    && request.auth.uid in getAfter(/databases/$(database)/documents/chats/$(chatId)).data.users
    && request.auth.uid == resource.data.receiverId
    && request.resource.data.seen is bool
    && request.resource.data.senderId == resource.data.senderId
    && request.resource.data.receiverId == resource.data.receiverId
    && request.resource.data.content == resource.data.content
    && request.resource.data.createdAt == resource.data.createdAt
    && request.resource.data.get('productId', null) == resource.data.get('productId', null);

  allow delete: if false;
}
```

Changes from pre-fix rules:

- **`get()` → `getAfter()` on all three rules (read, create, update).** This is the production-defect fix. `getAfter()` evaluates against post-batch state, so the new-chat `writeBatch` (creates chat doc and first message atomically) succeeds. With `get()`, the message-create rule sees pre-batch state where the chat doc doesn't exist yet, dereferences null, and denies.
- **Update rule restricts `seen` flipping to the receiver.** `request.auth.uid == resource.data.receiverId` clause added.
- **Update rule freezes `senderId`, `receiverId`, `content`, `createdAt`, `productId`.** Only `seen` is mutable.
- **`productId` immutability uses `get('productId', null)`** on both sides of the comparison because `productId` is optional on the message wire shape (§3.2). Messages without product context (conversations started from a user profile page) have no `productId` field at all; comparing absent-to-absent via plain `==` is engine-version-dependent in Firestore Rules. The `get(field, default)` form yields a defined value on both sides regardless of presence, so the comparison is stable.

Residual trust-boundary gap on `receiverId` at create time: the rule does NOT verify `receiverId` is the other participant in `chats[chatId].users`. A sender could write a message with an arbitrary `receiverId`, which would silently break the block check. Web client always writes the correct `receiverId`. Tightening to `request.resource.data.receiverId in getAfter(...).data.users && request.resource.data.receiverId != request.auth.uid` is the right fix. **Deferred** to a post-launch one-line rule change (logged in §11).

### 4.5 `/userchats/{userId}/chats/{chatId}` sidecar rules

```
match /userchats/{userId} {
  allow create, read, write: if request.auth != null && request.auth.uid == userId;

  match /chats/{chatId} {
    allow read: if request.auth != null && request.auth.uid == userId;
    allow delete: if request.auth != null && request.auth.uid == userId;

    allow create, update: if request.auth != null
      && (request.auth.uid == userId
          || request.auth.uid == request.resource.data.withUserFirebaseUid);
  }
}
```

Changes from pre-fix rules:

- **Read and delete are owner-only.** Previously the rule was shared across `create, read, update, delete`; the non-owner cross-user clause leaked reads of the other party's sidecar.
- **Create and update still allow the cross-user write pattern.** When sender writes to `userchats/{receiver}/chats/{chatId}`, the doc declares `withUserFirebaseUid == sender.uid` and the rule accepts. Enables the new-chat send batch.

### 4.6 `/userblocks/**` and `/userblocksReverse/**` rules

Unchanged from current. The forward index is read by `isBlocked()` and the owner. The reverse index is read by the blocked user.

```
match /userblocks/{userId}/blocked/{blockedId} {
  allow read: if request.auth != null && request.auth.uid == userId;
  allow create: if request.auth != null && request.auth.uid == userId && blockedId != userId;
  allow delete: if request.auth != null && request.auth.uid == userId;
}

match /userblocksReverse/{userId}/blockedBy/{blockerId} {
  allow read: if request.auth != null && request.auth.uid == userId;
  allow create, delete: if request.auth != null && request.auth.uid == blockerId;
}
```

### 4.7 `/notifications/**` rules — folded fix

The pre-fix rule had a critical trust-boundary violation: `allow create: if request.auth == null || request.auth.token.admin == true` — anonymous OR admin-claim holder. Anyone on the internet could create notification documents under any user's path. Fixed:

```
match /notifications/{userId}/userNotifications/{notificationId} {
  allow read, update, delete: if request.auth != null && request.auth.uid == userId;
  allow create: if false;
}
```

Backend writes notifications via Firebase Admin SDK, which bypasses rules entirely. No client has ever been the writer. `allow create: if false` is the correct posture.

This is folded into the same engineering brief as the messaging fixes because it's the same file, the same deploy cycle, and the same Mastermind decision pass.

### 4.8 Rule tests

The rules repo has scaffolding for `@firebase/rules-unit-testing` and `vitest` but **zero tests today**. `npm test` passes trivially via `--passWithNoTests`. The PR gate runs `firebase deploy --dry-run` which catches syntax errors only.

This feature seeds the test suite with the minimum messaging coverage:

1. Chat create as participant → pass
2. Chat create as non-participant → fail
3. Chat read as participant → pass
4. Chat read as non-participant → fail
5. Chat update mutating `users[]` → fail
6. Chat delete → fail (immortal)
7. Atomic chat+first-message create as participant → pass (regression guard for `getAfter()`)
8. Message create with pre-existing chat as participant → pass
9. Message create as non-participant → fail
10. Message create where sender has blocked receiver → fail
11. Message create where receiver has blocked sender → fail
12. Message update flipping `seen` as receiver → pass
13. Message update flipping `seen` as sender → fail
14. Message update mutating `content` → fail
15. Message update mutating `senderId` → fail
16. Notification create as anonymous → fail
17. Notification create as authed non-admin → fail
18. Userchats `/chats/{chatId}` read by owner → pass
19. Userchats `/chats/{chatId}` read by non-owner → fail
20. Userchats `/chats/{chatId}` cross-user write with `withUserFirebaseUid == auth.uid` → pass
21. Userchats `/chats/{chatId}` cross-user write with mismatched `withUserFirebaseUid` → fail
22. `users/{userId}` read by owner → pass
23. `users/{userId}` read by non-owner → fail

Test files live in `tests/messaging.test.ts` and `tests/users.test.ts`. The `npm test` script drops `--passWithNoTests` so empty test runs fail. The `validate-pr.yml` workflow gates on actual test results, not just rules-syntax dry-run.

---

## 5. Client flows

### 5.1 Identity and auth setup

Firebase Auth on web (`firebaseClient.ts`) auto-attaches the ID token on every Firestore request. The token is the same one verified by backend's `FirebaseAuthFilter`.

`useAuthStore.user` carries the resolved user. Messaging listeners (`ChatsInit` in `app/[locale]/layout.tsx`) attach when `user` flips from null to set, detach when it flips back to null. No `useAuthResolved` is used — the gate is store-state-based.

### 5.2 New-chat send flow

1. User clicks `StartMessageButton` on a product page. Handler dispatches based on auth and block state. Happy path:
   - `setTempProductContext(product)` — sets `tempProductContext` to the originating `ProductDetailsDTO`.
   - `setTempReceiver(seller)` — computes `chatId = sorted([currentUid, sellerUid]).join('_')`, checks if a chat with that id already exists locally or on Firestore; if it does, sets `activeChatId` and clears `tempReceiver`; otherwise sets both `activeChatId` and `tempReceiver`.
   - `router.push('/messages')`.
2. `/messages` mounts. `Messages` component detects `tempReceiver` is set, renders `MessageInput` with a pre-populated suggestion text built from `tempProductContext.name`.
3. **`MessageInput` mount effect does NOT clear `tempProductContext`.** This is the fix for the productId-clearing bug. The store field stays set so `sendMessage` can read it on Send.
4. User clicks Send. `MessageInput.send` builds `MessageContent`, calls `sendMessage(activeChatId, content)`.
5. `useChatStore.sendMessage`:
   - Reads `state.tempProductContext` synchronously at the top, captures as a local const.
   - Builds `MessageRequest`: `{ content, senderId: currentUid, receiverId: receiverUid, seen: false, createdAt: serverTimestamp() }`. Adds `productId: tempProductContext.id` if set.
   - Issues `writeBatch` with four operations:
     - `set(chats/{chatId}, { users: [senderUid, receiverUid], createdAt, lastMessage, lastUpdated })`
     - `set(userchats/{senderUid}/chats/{chatId}, { chatId, withUserFirebaseUid: receiverUid, unreadCount: 0, lastMessage, lastUpdated })`
     - `set(userchats/{receiverUid}/chats/{chatId}, { chatId, withUserFirebaseUid: senderUid, unreadCount: 1, lastMessage, lastUpdated })`
     - `set(chats/{chatId}/messages/{newId}, MessageRequest)`
   - `await batch.commit()`.
   - On success: replace optimistic tempId with realId, clear `tempReceiver` and `tempProductContext` (§5.6).
   - On failure: rollback optimistic message, fire toast, leave `tempReceiver` and `tempProductContext` intact (so retry from same context works).

The batch is atomic. The message-create rule uses `getAfter()` so it sees the in-flight chat-doc creation in the same batch.

### 5.3 Renames and field cleanup

`tempProductReason` → `tempProductContext` in `useChatStore` and all consumers. "Reason" is misleading; "Context" is accurate (the originating product). Mechanical rename; type signature changes from `ProductDetailsDTO | null | undefined` to `ProductDetailsDTO | null` (drop the `undefined` variant — it was a type-contract violation from callers).

### 5.4 Existing-chat send flow

Same store function, different branch. Wraps the four writes in `writeBatch`:

- `set(chats/{chatId}, { lastMessage, lastUpdated }, { merge: true })`
- `set(userchats/{senderUid}/chats/{chatId}, { lastMessage, lastUpdated }, { merge: true })`
- `set(userchats/{receiverUid}/chats/{chatId}, { lastMessage, lastUpdated, unreadCount: increment(1) }, { merge: true })`
- `set(chats/{chatId}/messages/{newId}, MessageRequest)`

All atomic. If any write fails, all fail — no partial-update drift between the chat root and sidecars.

The pre-fix "create sidecar with full shape if it doesn't exist, otherwise merge" branch is removed. `setDoc(..., { merge: true })` handles both create and update; the `withUserFirebaseUid` and `chatId` fields are already set on the sender's sidecar by the original new-chat batch and don't need to be re-asserted. If the sender previously `deleteChat`'d their sidecar and the chat is now resurrected via a new message, the merge-create case re-establishes the doc with just `lastMessage`/`lastUpdated`/`unreadCount`; the receiver's sidecar already exists and the merge updates it. Engineer in Phase 5 verifies the exact merge shape; the contract is "all four writes atomic, sender sidecar resurrected if absent."

### 5.5 Send-failure UX

`useChatStore.sendMessage` catch block:

- `console.error` for the engineer log.
- `notify.error()` toast with the new translation key (§8): "Failed to send message. Try again."
- Optimistic UI rollback (filter tempId message out of `messages[chatId]`).
- Orphan-image cleanup if the failed message carried image keys.
- **`tempReceiver` and `tempProductContext` are not cleared.** Retry from the same context works.

Pre-fix behavior was `console.error` only — silent vanish for the user. The toast is the user-facing fix.

### 5.6 Temp state lifecycle

`tempReceiver` and `tempProductContext` exist to bridge the "user clicks Message on a product page" → "user clicks Send on `/messages`" gap. They live in the Zustand store, not in URL state.

Lifecycle rules:

- **Set:** by `StartMessageButton.onClick` (product page) or `UserDetails` Send-Message button (user profile page), immediately before `router.push('/messages')`.
- **Read:** by `Messages` to render the conversation UI for a not-yet-existing chat. By `useChatStore.sendMessage` to attach `productId` to the first message.
- **Cleared on successful send:** at the end of `sendMessage`'s success branch.
- **Cleared on navigation away from `/messages`:** the `ChatsWatcher` component watches `pathname`. When pathname transitions to anything other than `/{locale}/messages`, both fields are cleared.
- **Not cleared on send failure:** retry stays viable.

The watcher must not clear when transitioning _into_ `/messages` from `/product/...` (the new-chat flow sets temp state then immediately navigates). Engineer in Phase 5 verifies React's state-update batching guarantees the path transition arrives at `/messages` with state intact; if not, the watcher needs a "only clear when previous path was `/messages`" guard.

Pre-fix `ChatsWatcher` only cleared on navigation to `/favorites`. The fix generalizes to "any non-`/messages` route."

### 5.7 Snapshot listeners

Four messaging-related listeners attach when `useAuthStore.user` becomes truthy via `ChatsInit`:

1. `userchats/{user.firebaseUid}/chats` — `orderBy('lastUpdated', 'desc')`, `limit(15)`. Powers chat list and header unread badge.
2. `userblocks/{user.firebaseUid}/blocked` — populates `blocking` map.
3. `userblocksReverse/{user.firebaseUid}/blockedBy` — populates `blockedBy` map.
4. `chats/{chatId}/messages` — `orderBy('createdAt', 'desc')`, `limit(15)`. Per active chat, attached lazily by `getActiveChat`.

Listeners 1-3 detach when `user` flips to null (sign-out). Listener 4 is per-chat; detached when leaving that chat or on sign-out via `unsubscribeAll`.

All four guard on `useAuthStore.getState().user` and short-circuit on null. No race with pre-auth render.

### 5.8 Auto-linkify

`Message.tsx`'s text-block render path detects URLs and renders them as anchor tags:

- **What's linkified:** http(s) URLs and www-prefixed URLs. The linkify library determines the boundary; engineer in Phase 5 audits `linkify-it` vs `linkifyjs` for size, license, and dep weight, picks one.
- **Anchor attributes:** `target="_blank"`, `rel="noopener noreferrer"`. The `noreferrer` prevents `window.opener.location` phishing attacks against the messaging tab.
- **Anchor text:** the URL itself, verbatim. No masking. What the user sees is what they click.
- **What's NOT linkified:** `javascript:`, `data:`, `vbscript:` schemes — most libraries default to safe schemes only; verified in Phase 5.
- **No preview cards.** URLs render as inline links.

Edge cases (IDN homograph attacks, look-alike domains) render as-is. The phishing surface exists at the level a URL-aware user can spot. Future work: link safety / URL blocklist (§11).

### 5.9 `getActiveChat` Firestore fallback

The pre-fix fallback read `chats/{chatId}` and tried to extract `data.withUserFirebaseUid` and `data.withUser` — neither field exists on the chat root doc. Silent failure on any chat opened that wasn't in the most-recent-15 listener cache (deep links, post-`deleteChat` resurrection, chat lists with >15 entries).

The fix derives the other UID from `data.users` (filter out the current user's UID), then calls `getUserData(otherUid)` to hydrate `withUser`:

```ts
const otherUid = chatData.users.find((u) => u !== currentUser.firebaseUid);
const withUser = await getUserData(otherUid);
```

### 5.10 Block / unblock

Unchanged from current behavior, confirmed correct by audit. `useChatBlockStore.blockUser` and `unblockUser` issue atomic `writeBatch` pairs over forward + reverse indexes. UI surfaces (Block dropdown, Unblock dropdown, banner, disabled input) work as audited.

Polish items in scope:

- Block badge in `Chats.tsx` so blocked threads are visible in the conversation list (§9).
- Dead helper `checkIsBlocked` deleted from `useChatStore.ts`.

### 5.11 Chat list "load more"

`useChatStore.loadMoreChats` exists and works (per audit). `Chats.tsx` exposes no UI to call it. Pre-fix: users with >15 chats see only the most-recent 15. Fix: add a "Load more" button at the bottom of the list, calling `loadMoreChats`, matching the existing pattern from `Messages.tsx`'s `loadMoreMessages`.

The dedicated key `MESSAGES_PAGE.chats.load.more` is seeded in all four locales (EN/RS/RU/CNR; RU in Latin transliteration per the existing seed file's convention). Per-locale row IDs: EN=3363, RS=5463, CNR=1263, RU=7563. The web call site in `Chats.tsx` consumes the dedicated key. The earlier placeholder reuse of `messages.load.more` (in Brief 2 W8) was replaced by Brief 6a.

### 5.12 Image attachments

Unchanged from current behavior. `uploadImages('chat', chatId, ...)` → backend mints upload tokens after `isUserMemberOfChat` check → direct PUT to R2 → resulting `imageKeys` go into `MessageContent.blocks[].imageKeys`. `MessageImages.tsx` resolves keys via view tokens with retry-on-401.

### 5.13 deleteChat reorder

Pre-fix: `deleteChat` mutates local store first, then awaits Firestore delete. On Firestore failure, the chat disappears locally but reappears on next listener fire. Fix: await Firestore delete first; on success, mutate local store. On failure: toast (reuse the messages send-failure key or add a delete-specific one — engineer judgment in Phase 5).

### 5.14 Mark-seen failure handling

Pre-fix: `subscribeToMessages` mutates the local `seenLocal` Set _before_ `batch.commit()`. On batch failure, the local Set is poisoned (those IDs marked seen locally even though Firestore disagrees), and the next listener fire's unseen filter skips them. Messages stay marked unseen on disk until page reload.

Fix: mutate `seenLocal` only after `batch.commit()` resolves. On failure, the Set stays clean, the next listener fire correctly re-queues those messages for the next mark-seen attempt.

---

## 6. Backend involvement

### 6.1 Read-only chat surfaces

Admin endpoints (`/api/secure/admin/chats/**`):

- `GET /user/{userId}/chats` — list a user's chats, paginated by `lastUpdated`. Returns `ChatDTO` with both participants resolved to `ChatUserDTO`.
- `GET /{chatId}/messages` — list a chat's messages, paginated by `createdAt`. Read-only; no moderation actions.

Both gated by `@PreAuthorize("hasRole('ADMIN')")`.

Trust-review (`DefaultTrustReviewService`) reads `chats/{chatId}/messages` to compute review eligibility. Chat-ID computation matches web (`sorted().join('_')`).

### 6.2 Admin receiver bug fix

`DefaultFirebaseChatService.getChatsForUserPaged` pre-fix: `getReceiverFromChatDoc(docs, ...)` reads `docs.stream().findFirst()` — every chat row in the page response carries the same receiver (whoever was the counterparty in the page's first chat). User-facing bug in the admin chat list.

Fix: pass the current `doc` to the helper; read the other UID from that doc's `users` array. Per-row receiver resolution. Backend test added covering the regression.

### 6.3 Image token endpoints

Unchanged. `POST /api/secure/images/upload-tokens` and `/view-tokens` with `scope=chat` verify chat membership via `isUserMemberOfChat` (single Firestore read of `chats/{chatId}.users`). Audit confirmed correct.

### 6.4 Notification service

Unchanged for messaging. Backend never writes message notifications today (deferred per §1.2). The `NotificationCategoryId.MESSAGE` enum value stays in place but unwired.

### 6.5 `/api/public/notification/test` endpoint — removed

The `NotificationsControllerTest` class in `src/main/java` registers a public, anonymous, body-controlled endpoint that writes notifications to arbitrary users. Pre-fix this is a notification-spam attack surface that also bypasses the F1 rule fix (because Admin SDK writes bypass rules).

This feature deletes:

- `controller/test/NotificationsControllerTest.java`
- `NotificationTestDTO` (if no other callers)

If any test in `src/test/java` references the endpoint, the test is rewritten to call `DefaultNotificationsService.sendAsyncNotification` directly (it's a Spring bean; test fixtures can inject it).

### 6.6 Sunday cleanup cron — new

Spring `@Scheduled` cron in a new service `DefaultMessagingCleanupService`. Runs Sundays at 03:00 UTC (configurable via YAML — see decisions.md 2026-05-18 for the precedent).

YAML keys:

- `messaging.cleanup.cron` — cron expression, default `"0 0 3 ? * SUN"`
- `messaging.cleanup.enabled` — boolean, default `true`
- `messaging.cleanup.batch.size` — int, default 100 (number of users processed per run)
- `messaging.cleanup.lookback.days` — int, default 30 (window of hard-deleted users to scan; older deletions are assumed already-cleaned or out of reach)

Algorithm:

1. Query Postgres for users hard-deleted in the lookback window where Firestore cleanup has not yet completed. The candidate-tracking approach is the `pending_chat_cleanup` table (§13). The User Deletion hard-delete path writes a row to this table at the moment of hard-delete, carrying the firebaseUid. Cron consumes from there. Cron deletes the row after success.

2. For each candidate user (batch size capped per run):
   a. Read the firebaseUid from the `pending_chat_cleanup` row.
   b. Use Firebase Admin SDK to scan `chats` collection for documents where `users` array contains the firebaseUid: `db.collection("chats").whereArrayContains("users", firebaseUid).get()`.
   c. For each matching chat document, in a Firestore transaction (or batch — engineer judgment based on document count per user):

   - Update `chats/{chatId}` `users[]` — replace the deleted UID with a sentinel string (exact format chosen by engineer in Phase 5, e.g. `"deleted:<original-uid>"`). The sentinel is **not** a real Firebase UID, so no future auth.uid will match it, and the rules' participant checks will reject the deleted user (which is correct — they don't exist anymore).
   - Scan `chats/{chatId}/messages` for messages where `senderId == firebaseUid` or `receiverId == firebaseUid`. Update each message's `senderId`/`receiverId` to the same sentinel.
   - Optionally update a `lastMessage` containing the deleted user's display name — out of scope for v1 (the preview string is approximate anyway; if the deleted user's name leaks into a 50-char preview, that's tolerable).
     d. Delete the deleted user's sidecars (`userchats/{firebaseUid}/**`) and block indexes (`userblocks/{firebaseUid}/**`, `userblocksReverse/{firebaseUid}/**`).
     e. On success: delete the `pending_chat_cleanup` row.
     f. On per-user failure: log ERROR, leave the `pending_chat_cleanup` row in place, continue with the next user. Next Sunday retries.

3. Cron run is idempotent — running it twice in a row is a no-op on the second run (no pending rows).

4. Concurrent safety: a new sibling table `messaging_cleanup_locks` carries the singleton job-lock. Initial design considered reusing `user_deletion_locks`, but on inspection that table is a per-user admin-action table (locks-a-specific-user-from-deletion) keyed by `user_id` with admin-action columns (`locked_by_admin_id`, `disclosable_reason`, `notified_at`, `expires_at`) — not a singleton cron lock primitive. The `messaging_cleanup_locks` table is keyed by `job_name` (UNIQUE) and uses atomic `INSERT … ON CONFLICT (job_name) DO UPDATE … WHERE expires_at <= NOW()` to let a new acquirer take over an expired lock from a crashed prior run. If multiple crons later need singleton locks, a shared `scheduled_job_locks` table can be extracted in a future refactor brief; today only this cron uses the new table.

### 6.7 Anonymization, not deletion

Per Phase 3 decision: messages from deleted users are **anonymized**, not deleted. The recipient's chat history reads coherently — they see "deleted user said X, then I said Y" instead of a missing message. Consistent with how User Deletion anonymizes reviews rather than deleting them.

The sentinel UID render: the web client's `getUserData(sentinelUid)` will fail (it's not a real UID). The client must handle the case gracefully — render `displayName: "Deleted user"` (localized) when the UID is a sentinel or when `getUserData` 404s. Engineer in Phase 5 adds this fallback in `Chats.tsx` and `Messages.tsx`.

Translation key: `user.deleted` already exists in `COMMON` namespace (used in `Chats.tsx:58`, `Messages.tsx:151, 155`). Reuse.

### 6.8 Translation seeds

19 existing `MESSAGES_PAGE` keys per locale, all in `0001-data-web-translations-{EN,RS,RU,CNR}.sql`. New keys for this feature:

- `messages.send.failed.toast` — "Failed to send message. Try again." (and locale equivalents)
- `messages.image.preview.fallback.label` — "Sent an image" — replaces hardcoded `'Nove poruke ...'` in `useChatStore.ts:56`. Engineer in Phase 5 may rename the key.

Total new keys: ~2-3. Under the 10-key threshold from Part 6 Rule 3, so they append inline to the existing `MESSAGES_PAGE` namespace group in the four locale files.

---

## 7. Trust boundaries

Per conventions Part 11.

### 7.1 Chat document writes

- `users[]`: **client-trusted + enforced by rule.** Rule checks `request.auth.uid in users` and `users.size() == 2`. Rule does NOT verify the _other_ UID is a real user. Known limitation; matches the existing block model where any user can be a target.
- `createdAt`, `lastUpdated`: server-derived via `serverTimestamp()`.
- `lastMessage`: client-supplied free text; not used for security decisions.

### 7.2 Userchats sidecar writes

- Path `ownerUid`: enforced by rule. Either writer owns the path, or the doc declares `withUserFirebaseUid == writer.uid`.
- `withUserFirebaseUid`: client-trusted + enforced by rule (cross-user write case).
- `chatId`, `unreadCount`, `lastMessage`, `lastUpdated`: not used for security decisions.

### 7.3 Message document writes

- `senderId`: **client-trusted + enforced by rule.** Rule checks `senderId == request.auth.uid`. Client cannot impersonate.
- `receiverId`: client-trusted, **not server-verified at create time**. Rule does not check `receiverId in chats[chatId].users`. A malicious client could write a message with arbitrary `receiverId`, breaking the block check. Web client always writes the correct other-participant UID. **Residual gap**; post-launch one-line fix logged in §11.
- `productId`: client-supplied if a product-context message. Not used for security decisions.
- `content`: client-supplied free text and image keys. Not validated by rules.
- `seen`: created as `false`. On update, rule enforces receiver-only flip.
- `createdAt`: server-derived via `serverTimestamp()`.

### 7.4 Block index writes

- Forward and reverse paths enforced by rule — only the blocker writes.
- Self-block prevented by rule (`blockedId != userId`).

### 7.5 Image upload and view

- Image keys are server-derived (backend issues them as part of `POST /api/secure/images/upload-tokens`).
- Chat membership is server-verified on both upload-token and view-token issuance.
- View-token mint is re-verified on every request — no caching.

### 7.6 What is NOT a trust boundary today

- **Content moderation.** Message text not screened. Future feature (§11).
- **Link safety.** Linkified URLs not screened against blocklists. Future concern.
- **Rate limiting.** No application-level rate limiter on chat writes. Future concern.

---

## 8. Translations

Per conventions Part 6. Namespace: `MESSAGES_PAGE`.

New keys this feature:

| Key                                     | EN                                   | Notes                                                          |
| --------------------------------------- | ------------------------------------ | -------------------------------------------------------------- |
| `messages.send.failed.toast`            | "Failed to send message. Try again." | Toast on `sendMessage` catch                                   |
| `messages.image.preview.fallback.label` | "Sent an image"                      | Replaces hardcoded `'Nove poruke ...'` in `useChatStore.ts:56` |

Seeded in EN, RS, RU, CNR per Part 6 Rule 3. Inline-appended to the existing `MESSAGES_PAGE` namespace group in `0001-data-web-translations-{EN,RS,RU,CNR}.sql`.

The `delete-failure` case (§5.13) reuses `messages.send.failed.toast` or adds a `messages.delete.failed.toast` — engineer judgment in Phase 5.

Polish item: replace `'sr-RS'` hardcoded locale in admin `Chat.tsx:8` with `useLocale()`. Not a translation seed change.

---

## 9. Polish bundle (§F17)

In scope. One brief at the end of Phase 5:

- **F17a** — Replace hardcoded `'Nove poruke ...'` in `useChatStore.ts:56` with `messages.image.preview.fallback.label` from `MESSAGES_PAGE`. Render at display time in `Chats.tsx` rather than persisting in `lastMessage` (engineer decides — persist or render-time, both work; render-time is cleaner).
- **F17b** — Replace `'sr-RS'` in `src/components/admin/chats/Chat.tsx:8` with `useLocale()`.
- **F17c** — Add block badge in `Chats.tsx` for threads where `useChatBlockStore.blocking[counterpartyUid]` or `blockedBy[counterpartyUid]` is true. Match the `ScheduledForDeletionBadge` visual pattern.
- **F17d** — Delete dead exports: `checkIsBlocked` (`useChatStore.ts:37-45`), `testConnection` (`firebaseClient.ts:66`), `getNextAdminChatMessages` (admin service).
- **F17e** — `deleteChat` reorder: await Firestore delete first, then mutate local store. Toast on Firestore failure.
- **F17f** — `subscribeToMessages` mark-seen: move `seenLocal` Set mutation to after `batch.commit()` resolves.
- **F17g** — Document the chatId convention (`sorted([uidA, uidB]).join('_')`) in §3.1 above. **Done in this spec.** No code change.

Plus deletion of the `tempProductReason`-related dead type variants once the rename lands.

---

## 10. Engineering work

Phase 5 briefs, in shipped order. All shipped on 2026-05-20.

### 10.1 Brief 1 — Firestore rules + tests (`oglasino-firestore-rules`) — shipped

Foundation. Everything downstream depended on rules being correct. Brief 1 + the mid-stream Brief 1b amendment (added the `get('productId', null)` shape and a 24th test for the absent-`productId` case) shipped together. 24 rule tests passing on the emulator; deployed to stage (`oglasino-stage-49abb`) and smoke-verified on 2026-05-20 (new-chat send batch lands cleanly, existing-chat sends work, block/unblock work).

- Apply rule changes per §4.2 (`users` owner-only), §4.3 (chat update freezes `users[]`, delete `if false`), §4.4 (`getAfter` × 3, receiver-only `seen` flip, field-shape immutability including `get('productId', null)`), §4.5 (userchats read/delete owner-only), §4.7 (notifications `create: if false`).
- Seed test suite per §4.8: ~23 tests across `tests/messaging.test.ts` and `tests/users.test.ts`.
- `npm test` script drops `--passWithNoTests`.
- `validate-pr.yml` gates on actual test results.
- Deploy to stage (manual `workflow_dispatch` or branch push, per existing setup).
- Smoke verification on stage by Igor: new-chat from product page, existing-chat send, block/unblock, message-update-as-sender denied.

**Brief is sole writer of rule + test changes. Does not touch backend or web.**

### 10.2 Brief 2 — Web messaging core (`oglasino-web`) — shipped

Depends on Brief 1.

- §5.2-§5.4 — productId fix, rename `tempProductReason` → `tempProductContext`, atomic `writeBatch` on both new-chat and existing-chat paths.
- §5.5 — send-failure toast.
- §5.6 — temp state cleared on navigation away from `/messages` (`ChatsWatcher` rewrite).
- §5.9 — `getActiveChat` fallback fix.
- §5.11 — chat list load-more button.
- §5.13 — `deleteChat` reorder.
- §5.14 — mark-seen failure fix.
- Polish §9 a-f.
- Deleted-user fallback rendering (`getUserData` 404 → render "Deleted user" via `COMMON.user.deleted`).
- New translation key consumption.
- Tests: web test suite, including unit tests for `sendMessage` batch correctness, `ChatsWatcher` navigation behavior, `getActiveChat` Firestore fallback.

Shipped 2026-05-20; 154 web tests passing post-change.

### 10.3 Brief 3 — Auto-linkify (`oglasino-web`) — shipped

Independent of Brief 2; can run in parallel after Brief 1.

- Audit `linkify-it` vs `linkifyjs` for size and license. Pick.
- Modify `Message.tsx` text-block rendering. Anchor attributes per §5.8.
- Tests: URL detection, anchor attribute correctness, `javascript:` not linkified.

Shipped 2026-05-20; `linkify-it@5.0.0` chosen (size + license + spec-named); 12 tokenizer tests added; 166 total web tests.

### 10.4 Brief 4 — Backend admin + endpoint cleanup (`oglasino-backend`) — shipped

Independent of Brief 1 and 2.

- §6.2 — fix `getReceiverFromChatDoc` to read per-doc.
- §6.5 — delete `NotificationsControllerTest` and its DTO.
- Translation seeds per §8.
- Tests: backend admin chat-list regression, translation seed integrity.

Shipped 2026-05-20; 539 backend tests passing; `NotificationCategoryId.MESSAGE` confirmed zero callers and left as intentional stub.

### 10.5 Brief 5 — Backend cleanup cron (`oglasino-backend`) — shipped

Depends on Brief 1 (rules must permit Admin SDK writes — they always have, since Admin SDK bypasses).

- §6.6 — new `DefaultMessagingCleanupService`, `@Scheduled` cron, YAML config keys.
- New `pending_chat_cleanup` table (schema decision per §13).
- User Deletion hard-delete path updated to write `pending_chat_cleanup` rows. **Cross-feature touch** — this brief modifies `DefaultUserDeletionService.runHardDelete`. Acceptable because messaging cleanup _is_ the missing User Deletion piece; the change is small and additive (one repository insert).
- Sentinel UID format chosen (e.g. `"deleted:<uid-prefix>"`).
- Tests: cron logic with mocked Firestore, sentinel write correctness, idempotency, batch limits, lock contention.

Shipped 2026-05-20; 11 cron + integration tests passing; first-production-Sunday live smoke deferred per Igor's choice (logged in `issues.md`).

### 10.6 Brief 6 — split into 6a (engineer: `chats.load.more` key seed + web swap) and 6b (Docs/QA close-out) — shipped

**Brief 6a — engineer.** Backend (`oglasino-backend`): seeded `MESSAGES_PAGE.chats.load.more` in all four locale SQL files. Web (`oglasino-web`): swapped placeholder `messages.load.more` to `chats.load.more` in `Chats.tsx`. Shipped 2026-05-20.

**Brief 6b — Docs/QA.** Three `issues.md` flips, two new `issues.md` entries, closing `decisions.md` entry, `state.md` flip + Expo backlog row + Risk Watch update + session log entries, this set of spec amendments. Shipped 2026-05-20.

### 10.7 Order

Brief 1 (rules) ships first. After stage smoke passes, Briefs 2, 3, 4 can run in parallel. Brief 5 ships after Brief 1 (depends on rules being correct; doesn't depend on web). Brief 6 ships last.

---

## 11. Out of scope, captured for post-launch backlog

Each becomes its own future Mastermind chat or `issues.md` entry as appropriate.

### 11.1 Admin welcome chat for new users

Sketch (carried from this chat's intake discussion):

- Single designated admin Firebase user, UID configured via env var or Postgres `Configuration` table.
- New user registers → backend writes a welcome `chats/{chatId}` doc, both `userchats/` sidecars, and a first message via Firebase Admin SDK on the user's behalf.
- The welcome chat is marked `isSystem: true` on both sidecars. Firestore rules refuse delete on `isSystem` chats (sidecar delete rule), refuse blocks against the admin UID (userblocks create rule).
- Web UI hides the Delete / Block / Report dropdown items when the counterparty UID equals the configured admin UID.
- Welcome message text translated via `BACKEND_TRANSLATIONS` namespace, picked by user's `preferredLanguage`.
- Idempotent — re-running for an existing user is a no-op.

Composes cleanly on top of this feature's data model. Open as a separate Mastermind chat post-launch.

### 11.2 Push notifications for incoming messages

Wire `NotificationCategoryId.MESSAGE` through `DefaultNotificationsService.sendAsyncNotification`. Triggered either by backend endpoint called post-Firestore-commit by the sender's client, or by a Cloud Function listening on `chats/{chatId}/messages` writes. Architectural decision deferred.

### 11.3 Receiver-in-`users[]` rule check

Tighten the message-create rule to additionally verify `receiverId in getAfter(...).data.users && receiverId != senderId`. One-line rule change; closes the residual trust-boundary gap on `receiverId` (§7.3).

### 11.4 Message deletion

5-minute soft-delete window. Sender can delete their own message within 5 minutes. Renders as "Message deleted" client-side.

### 11.5 Message content moderation

Banned words, spam, scam detection, link safety. Different abuse profile from product moderation.

### 11.6 Mobile adoption (`oglasino-expo`)

Mobile catch-up adopts this spec's contracts. Chat ID scheme, field names, rule expectations are all platform-agnostic.

### 11.7 Rate limiting on chat writes

Application-level rate limiter via Firestore counter doc or Cloud Functions.

### 11.8 Link previews (Open Graph cards)

Server-side OG metadata fetch with caching. Out of scope for v1 per §5.8.

### 11.9 Search

In-conversation search and global search across conversations.

### 11.10 Typing indicators, read receipts beyond `seen`, reactions, edit, threading

Each is its own feature.

### 11.11 `users/{userId}.blocked` field investigation

Audit found a `blocked` field on `users/{userId}` Firestore docs separate from the `userblocks/` and `userblocksReverse/` collections. No code reads it. Likely vestigial. Investigation chat or one-shot cleanup.

### 11.12 `fcmToken` location

Per §4.2 adjacent finding: FCM tokens currently live on `users/{userId}` Firestore docs. Even with the rule locked down to owner-only, this is the wrong storage location — backend already has push-token rows in Postgres for this. Migration / cleanup chat post-launch.

---

## 12. Closure criteria

Feature is shipped when:

1. All 5 engineering briefs (§10) are merged.
2. The validation steps embedded in each brief pass.
3. Stage Firebase carries the fixed rules. Production deployment is operator's decision before opening traffic.
4. The post-launch backlog entries (§11) are reflected in `state.md`.
5. Closing `decisions.md` entry applied by Docs/QA.
6. `issues.md` entries that closed on this work, flipped to `fixed` by Brief 6b:
   - 2026-05-17 — `tempProductReason` clearing → closed by Brief 2 W2 + W3.
   - 2026-05-14 — Messages chat list capped at 15 → closed by Brief 2 W8 + Brief 6a.
   - 2026-05-14 — Messages silent text-only send failure → closed by Brief 2 W5.

   Two new `issues.md` entries created by Brief 6b:
   - 2026-05-20 — Brief 5 cleanup cron smoke not run on dev; first production Sunday is the live test. Severity low, open.
   - 2026-05-20 — `TestCreateJSON.java` is an anonymous public endpoint triggering catalog-JSON regeneration. Severity medium, open. Separate post-launch fix brief.

   The following messaging-adjacent `issues.md` entries stayed open because their subject matter is not the messaging surface itself, per the messaging-subject-strict closure rule Igor confirmed on 2026-05-20:
   - 2026-05-19 — Manual testing of messaging gating for deleted and banned users not yet performed. (User Deletion verification; not closed by messaging.)
   - 2026-05-17 — `UserInfoDTO.isFollowingCurrent` non-authoritative seed. (Follow-flow concern.)
   - 2026-05-16 — Report-submit endpoint trust-boundary verification. (Backend concern surfaced by Messages.tsx Report dropdown work, but the endpoint validation lives in the backend not in messaging behavior.)
7. Session-closure gate per conventions Part 3 met for the Mastermind chat producing this spec.

---

## 13. Open items / inferred decisions

Marked per conventions: inferred items confirmed by Igor before Phase 5 brief drafting.

- **Confirmed:** `pending_chat_cleanup` table for cron candidate tracking (§6.6). Hard-delete writes a row; cron consumes and deletes the row after success. Clean separation between User Deletion's hard-delete and the messaging cleanup pass.
- **Confirmed:** Sentinel UID format — engineer in Phase 5 picks exact format (e.g. `"deleted:<original-uid>"`).
- **Confirmed:** Sunday 03:00 UTC default cron time. YAML-configurable.
- **Confirmed:** `messages.image.preview.fallback.label` translation key name as placeholder; engineer in Phase 5 may rename.
- **Confirmed:** Phase 5 brief ordering per §10.7. Brief 1 first; Briefs 2/3/4 in parallel after stage smoke; Brief 5 after Brief 1; Brief 6 last.

All other content is factual (grounded in audits) or pre-resolved in Phase 3 with Igor's confirmation.
