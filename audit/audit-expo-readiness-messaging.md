# Audit — Expo release readiness: messaging

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-05-23
**Mode:** read-only audit — no code changes

---

## Section 1 — Chat list surface

**File paths:**
- `src/components/messages/Chats.tsx` — chat list component
- `app/(portal)/(secured)/messages.tsx` — page-level container (toggling between list and detail)

**Per-row display:**
- Other participant name (`item.withUser.displayName`)
- Avatar (`item.withUser.profileImageKey`)
- Last message preview (`item.lastMessage`) — single line, truncated via `numberOfLines={1}`
- Unread count badge (red circle with count when `item.unreadCount > 0`)
- No last-activity timestamp rendered (field is on the model but not displayed)

**Ordering, pagination, refresh:**
- Ordered by `lastUpdated desc` via `useChatStore.subscribeToChats` (Firestore snapshot)
- Paginated at 15 per page (`LIMIT = 15`)
- `loadMoreChats` exists in the store but **no UI button exposes it** — users with >15 chats see only the first 15 (spec §5.11 says web added a "Load more" button; mobile has not adopted this)
- Real-time refresh via `onSnapshot` listener

**Data source:** Firestore `userchats/{user.firebaseUid}/chats` subscription only — no backend endpoint for the user-facing chat list.

**Block/archive/deleted distinction:**
- No block badge shown on chat list items (spec §9 F17c says web added this; mobile has not)
- No archived or deleted state distinctions in the UI
- Local search filter by display name

---

## Section 2 — Chat detail surface

**File paths:**
- `src/components/messages/Messages.tsx` — chat detail view
- `src/components/messages/Message.tsx` — single message block renderer
- `src/components/messages/MessageInput.tsx` — input with text + image attachment
- `src/components/messages/MessageImages.tsx` — image display with view-token resolution

**Message list rendering:** `FlatList` with `keyExtractor={(_, i) => i.toString()}` (index-based keys on grouped messages). Scrolls to end on content change.

**Message types supported:**
- Text blocks (`block.type === 'text'`)
- Image blocks (`block.type === 'images'`) via `MessageImages` component
- No system messages rendered
- No auto-linkify for URLs (web has `linkify-it`; mobile renders plain text)

**Message status indicators:**
- No sending/sent/failed/read indicators visible to the user
- Optimistic message appears immediately (identified by `temp-${Date.now()}` ID)
- On failure, optimistic message is silently removed — no toast, no retry UI

**Send input surface:**
- Text (multiline `TextInput`)
- Image attachments via `expo-image-picker` (up to 5 images)
- Image upload progress overlays during send
- Cancel button appears during send
- Disabled state when blocked

---

## Section 3 — Firestore read inventory

| File:line | Collection | Document pattern | What field(s) read | Owner-only safe? |
|---|---|---|---|---|
| `src/lib/store/useChatStore.ts:128-129` | `userchats` | `userchats/{user.firebaseUid}/chats` (onSnapshot) | chatId, withUserFirebaseUid, unreadCount, lastMessage, lastUpdated | **yes** — reads own sidecar |
| `src/lib/store/useChatStore.ts:179-184` | `userchats` | `userchats/{user.firebaseUid}/chats` (getDocs, paginated) | same fields | **yes** — reads own sidecar |
| `src/lib/store/useChatStore.ts:236-238` | `chats` subcollection | `chats/{chatId}/messages` (onSnapshot) | id, senderId, receiverId, content, seen, createdAt, productId | **yes** — user is participant (rule uses `getAfter()`) |
| `src/lib/store/useChatStore.ts:318-323` | `chats` subcollection | `chats/{chatId}/messages` (getDocs, paginated) | same fields | **yes** — user is participant |
| `src/lib/store/useChatStore.ts:387` | `chats` | `chats/{resolvedChatId}` (getDoc, existence check) | existence only | **yes** — rule allows read if user in `users[]` |
| `src/lib/store/useChatStore.ts:523` | `userchats` | `userchats/{user.firebaseUid}/chats/{chatRef.id}` (getDoc) | existence check for merge decision | **yes** — own sidecar |
| `src/lib/store/useChatStore.ts:681` | `chats` | `chats/{chatId}` (getDoc, in `setTempReceiver`) | existence check | **yes** — allowed per rule |
| `src/lib/store/useChatStore.ts:711-712` | `chats` | `chats/{chatId}` (getDoc, in `getActiveChat` fallback) | withUserFirebaseUid, withUser, unreadCount (reads chat root doc) | **yes** — user is participant |
| `src/lib/store/useChatStore.ts:36-40` | `userblocks` | `userblocks/{userA}/blocked/{userB}`, `userblocks/{userB}/blocked/{userA}` (getDoc × 2, `checkIsBlocked`) | existence check | **unclear** — reads another user's forward block index if userA != auth.uid (rule: `read: if request.auth.uid == userId`). **This function is dead code** — never called externally. No runtime impact. |
| `src/lib/store/useChatBlockStore.ts:20` | `userblocks` | `userblocks/{user.firebaseUid}/blocked` (onSnapshot) | all docs (blockerId, createdAt) | **yes** — own block index |
| `src/lib/store/useChatBlockStore.ts:23` | `userblocksReverse` | `userblocksReverse/{user.firebaseUid}/blockedBy` (onSnapshot) | all docs (blockerId, createdAt) | **yes** — own reverse index |
| `src/lib/services/authService.ts:59` | `users` | `users/{firebaseUser.uid}` (getDoc) | existence check before create | **yes** — reads own doc |
| `src/lib/client/firebaseNotifications.ts:50-54` | `notifications` | `notifications/{firebaseUid}/userNotifications` (onSnapshot) | all notification fields | **yes** — own notifications |
| `src/lib/client/firebaseNotifications.ts:72-80` | `notifications` | `notifications/{firebaseUid}/userNotifications` (getDocs, paginated) | all notification fields | **yes** — own notifications |

### Cross-user `users/{otherUserId}` reads — THE CRITICAL CHECK

**No cross-user Firestore reads of `users/{otherUserId}` found.**

The only read of `users/{userId}` is in `authService.ts:59` which reads `users/{firebaseUser.uid}` — the authenticated user's own document.

User data enrichment for chat participants (display name, avatar) goes through `getUserForFirebaseUid(uid)` which calls the **backend endpoint** `GET /auth/firebase/{uid}` — NOT a direct Firestore read of `users/{otherUserId}`. This is safe under the tightened rules.

The `testConnection` function in `firebaseClient.ts:38-50` uses `collection(db, 'users')` in a snapshot listener but is **commented out** (never called). Would fail under tightened rules if enabled, but has no runtime impact.

**Verdict: No release-blocking cross-user reads.**

---

## Section 4 — Firestore write inventory

| File:line | Collection | Document pattern | What field(s) written | Trust concern |
|---|---|---|---|---|
| `src/lib/store/useChatStore.ts:459-465` | `chats` | `chats/{chatId}` (batch.set, new chat) | users, createdAt (serverTimestamp), lastMessage, lastUpdated (serverTimestamp) | None — users contains auth.uid; serverTimestamp for dates |
| `src/lib/store/useChatStore.ts:468-473` | `userchats` | `userchats/{user.firebaseUid}/chats/{chatId}` (batch.set) | chatId, withUserFirebaseUid, unreadCount, lastMessage, lastUpdated | None — own sidecar |
| `src/lib/store/useChatStore.ts:477-483` | `userchats` | `userchats/{receiver.firebaseUid}/chats/{chatId}` (batch.set, cross-user) | chatId, withUserFirebaseUid (= sender.uid), unreadCount, lastMessage, lastUpdated | None — authorized by rule §4.5 cross-user clause |
| `src/lib/store/useChatStore.ts:488-500` | `chats` subcollection | `chats/{chatId}/messages/{newId}` (batch.set) | content, senderId (auth.uid), receiverId, seen (false), createdAt (serverTimestamp), productId (optional) | **`receiverId` is client-supplied** — not verified against `chats.users[]` by rules. Known residual gap documented in spec §4.4/§7.3. Not a mobile-specific issue. |
| `src/lib/store/useChatStore.ts:518` | `chats` subcollection | `chats/{chatId}/messages/{newId}` (addDoc, existing chat) | same as above | Same residual gap |
| `src/lib/store/useChatStore.ts:527-531` | `userchats` | `userchats/{user.firebaseUid}/chats/{chatId}` (setDoc, full shape) | chatId, withUserFirebaseUid, unreadCount, lastMessage, lastUpdated | None — own sidecar |
| `src/lib/store/useChatStore.ts:535-540` | `userchats` | `userchats/{user.firebaseUid}/chats/{chatId}` (setDoc merge) | lastMessage, lastUpdated | None — own sidecar |
| `src/lib/store/useChatStore.ts:545-555` | `userchats` | `userchats/{receiver.firebaseUid}/chats/{chatId}` (setDoc merge) | chatId, withUserFirebaseUid (= sender.uid), lastMessage, lastUpdated, unreadCount (increment) | None — authorized by cross-user clause |
| `src/lib/store/useChatStore.ts:295-303` | `chats` subcollection + `userchats` | mark-as-seen batch: `messages/{id}` update + `userchats/{uid}/chats/{chatId}` update | `seen: true` on messages; `unreadCount: 0` on sidecar | None — receiver flips own messages |
| `src/lib/store/useChatStore.ts:628` | `userchats` | `userchats/{user.firebaseUid}/chats/{chatId}` (deleteDoc) | delete | None — own sidecar |
| `src/lib/store/useChatBlockStore.ts:57-58` | `userblocks` | `userblocks/{user.firebaseUid}/blocked/{targetUid}` (batch.set) | blockedUserId, createdAt | None — own block doc |
| `src/lib/store/useChatBlockStore.ts:62-63` | `userblocksReverse` | `userblocksReverse/{targetUid}/blockedBy/{user.firebaseUid}` (batch.set) | blockerId (= user.uid), createdAt | None — authorized by rule (auth.uid == blockerId) |
| `src/lib/store/useChatBlockStore.ts:80` | `userblocks` | `userblocks/{user.firebaseUid}/blocked/{targetUid}` (batch.delete) | delete | None — own doc |
| `src/lib/store/useChatBlockStore.ts:81` | `userblocksReverse` | `userblocksReverse/{targetUid}/blockedBy/{user.firebaseUid}` (batch.delete) | delete | None — authorized by rule |
| `src/lib/services/authService.ts:99` | `users` | `users/{firebaseUser.uid}` (setDoc) | id, displayName, email, profileImageKey, blocked, createdAt | None — own doc |
| `src/lib/services/authService.ts:100` | `userchats` | `userchats/{firebaseUser.uid}` (setDoc) | chats: [] | None — own sidecar root |
| `src/lib/client/firebaseNotifications.ts:93-98` | `notifications` | `notifications/{uid}/userNotifications/{id}` (updateDoc) | seen: true | None — own notifications |
| `src/lib/client/firebaseNotifications.ts:101-106` | `notifications` | `notifications/{uid}/userNotifications/{id}` (updateDoc) | shown: true | None — own notifications |

**Trust summary:** No client-side write claims to modify another user's data illegitimately. The cross-user writes (receiver's sidecar, reverse block index) are authorized by the rule system. The `receiverId` residual gap is spec-acknowledged and not mobile-specific.

---

## Section 5 — Chat ID scheme and field names

**Chat ID scheme:**
- Mobile: `[user.firebaseUid, receiver.firebaseUid].sort().join('_')` at `useChatStore.ts:379` and `useChatStore.ts:673`
- Spec: `sorted([uid1, uid2]).join('_')` (§2, §3.1)
- **Match: YES**

**Field name comparison:**

| Spec field name (§3.1 chat doc) | Mobile field name | Match |
|---|---|---|
| `users` (array of 2 UIDs) | `users` | YES |
| `createdAt` (Timestamp) | `createdAt` (serverTimestamp) | YES |
| `lastMessage` (string) | `lastMessage` | YES |
| `lastUpdated` (Timestamp) | `lastUpdated` (serverTimestamp) | YES |

| Spec field name (§3.2 message doc) | Mobile field name | Match |
|---|---|---|
| `senderId` | `senderId` | YES |
| `receiverId` | `receiverId` | YES |
| `content.blocks[]` | `content.blocks[]` | YES |
| `productId` (optional) | `productId` (optional) | YES |
| `seen` (boolean) | `seen` | YES |
| `createdAt` (Timestamp) | `createdAt` (serverTimestamp) | YES |

| Spec field name (§3.3 userchats sidecar) | Mobile field name | Match |
|---|---|---|
| `chatId` | `chatId` | YES |
| `withUserFirebaseUid` | `withUserFirebaseUid` | YES |
| `unreadCount` | `unreadCount` | YES |
| `lastMessage` | `lastMessage` | YES |
| `lastUpdated` | `lastUpdated` | YES |

| Spec field name (§3.4 block doc) | Mobile field name | Match |
|---|---|---|
| `blockerId` | forward: `blockedUserId`; reverse: `blockerId` | **PARTIAL MISMATCH** — spec says forward doc carries `blockerId`, mobile writes `blockedUserId` to the forward index. The field is convenience-only (redundant with path), so the mismatch is cosmetic, not functional. |
| `createdAt` | `createdAt` | YES |

---

## Section 6 — Send and receive flow

**Send path:**
1. User taps Send in `MessageInput` → builds `MessageContent` with blocks → calls `onSend(content)` which maps to `useChatStore.sendMessage(activeChatId, content)`.
2. Store function decides new-chat vs existing-chat. New chat: `writeBatch` with 4 operations (chat root + 2 sidecars + message). Existing chat: `addDoc` for message, then sequential `setDoc` merge calls for sidecars — **NOT atomic** for existing chats (web fixed this in Brief 2; mobile still uses the pre-fix non-atomic pattern).
3. Optimistic message rendered immediately.
4. On success: optimistic temp ID replaced with real Firestore-assigned ID; `tempReceiver` and `tempProductReason` cleared.
5. On failure: optimistic message removed from store; `console.error` logged; **no user-visible toast** (web added toast in Brief 2; mobile has not adopted).

**Receive path:**
- Per-chat `onSnapshot` listener (`subscribeToMessages`) attaches when a chat is opened via `getActiveChat`.
- Global chat-list listener (`subscribeToChats`) fires on new messages, updating `lastMessage` and `unreadCount` in the list.
- No always-on global message listener beyond the chat-list sidecar.

**Read receipts / mark-as-read:**
- When the message snapshot fires with unseen messages (where `receiverId == user.firebaseUid && !seen`), a `writeBatch` flips `seen: true` on each message and sets `unreadCount: 0` on the user's sidecar.
- Direct client Firestore write (no backend call).
- `seenLocal` Set prevents re-marking on subsequent snapshot fires.
- **Bug:** `seenLocal` is mutated _before_ `batch.commit()` — on commit failure, those messages won't be retried until page reload (spec §5.14 documents this; web fixed it; mobile hasn't adopted the fix).

**Failed sends:**
- Optimistic message is silently removed.
- `console.error` only — no user-visible retry UI or toast.
- `tempReceiver`/`tempProductReason` are not cleared on failure (retry is theoretically viable but no UI signals the failure state).

---

## Section 7 — Block / unblock

**Block UI location:** `ChatUserFunctionsDialog.tsx` — opened via avatar/name press in chat header. Contains Block/Unblock buttons conditional on `!blockedBy`.

**Blocking client-side:** Direct Firestore `writeBatch` to both forward (`userblocks/{myUid}/blocked/{theirUid}`) and reverse (`userblocksReverse/{theirUid}/blockedBy/{myUid}`) indexes. No backend call.

**Block state on receive side:**
- `useChatBlockStore` subscribes to `userblocks/{myUid}/blocked` and `userblocksReverse/{myUid}/blockedBy` via `onSnapshot`.
- Chat detail (`Messages.tsx`) checks `isBlocking(otherUid)` and `isBlockedBy(otherUid)`:
  - If blocking: shows "can't send because you blocked" message, disables input.
  - If blockedBy: shows "can't send because they blocked you" message, disables input.
- **Messages from blocked users are NOT filtered in the list or detail view.** They still appear in the chat history — blocking only prevents new sends.

**Block state source on read side:** Real-time Firestore snapshot listeners (not backend).

---

## Section 8 — Trust boundary check

### Backend endpoints called from messaging surface:

**1. `GET /auth/firebase/{uid}` — user data resolution**
- Fields sent: `uid` as path parameter
- Source of `uid`: extracted from Firestore message documents (`senderId`, `receiverId`) or sidecar (`withUserFirebaseUid`)
- Trust: the endpoint returns public user info; no sensitive action taken based on the UID. The UID is client-supplied (from Firestore documents that were written by authenticated users under rule enforcement). Backend verifies caller auth via `FirebaseAuthFilter` on the request header.

**2. `POST /api/secure/images/upload-tokens` (via `uploadImages`)**
- Fields sent: `scope: 'chat'`, `chatId`, `count`, `contentTypes`
- `chatId`: client-supplied, derived from the deterministic scheme (`sorted UIDs joined`)
- Trust: backend verifies caller is a participant in the chat via `isUserMemberOfChat` Firestore read. **Server-derived participation check — correct.**

**3. `POST /api/secure/images/view-tokens` (via `useViewTokenStore.getToken`)**
- Fields sent: `scope: 'chat'`, `chatId`
- `chatId`: client-supplied
- Trust: backend re-verifies chat membership. **Server-derived — correct.**

**4. `GET /secure/admin/chats/user/{userId}/chats`** (admin surface only)
- Fields sent: `userId` as path parameter, `lastUpdated` as query param
- Trust: gated by `@PreAuthorize("hasRole('ADMIN')")`. `userId` is the Postgres integer ID of the user whose chats to list. Admin-only.

**5. `GET /secure/admin/chats/{chatId}/messages`** (admin surface only)
- Fields sent: `chatId` as path parameter, `lastCreatedAt` as query param
- Trust: gated by admin role.

### Specific trust-boundary questions from the brief:

- **"Send message" backend endpoint:** Mobile does NOT call a backend endpoint to send messages. Messages are written directly to Firestore. The `senderId` is always set to `user.firebaseUid` (auth UID). The `receiverId` is client-supplied (spec-acknowledged residual gap — not mobile-specific).
- **"Create chat" endpoint:** No backend create-chat endpoint exists. Chat is created via Firestore `writeBatch`. Participation enforced by rules (`request.auth.uid in request.resource.data.users`).
- **"Block user" endpoint:** No backend block endpoint. Block is a Firestore write. Blocker ID is enforced by rule (`request.auth.uid == userId` on the forward path).

**Conclusion:** Trust boundaries are correctly maintained. No mobile-specific trust-boundary violations found.

---

## Section 9 — Push notifications for messages

**Handling location:** `src/notifications/components/PushNotificationsInit.tsx`

**`NotificationType.NORMAL = 'NOTMAL'` typo:** Confirmed at `src/lib/client/firebaseNotifications.ts:17`. The enum value is `'NOTMAL'` (misspelled "NORMAL"). **Consequences:** This enum is used as a type discriminator read from Firestore notification documents' `type` field. If the backend writes notifications with `type: 'NORMAL'` (correct spelling), mobile would fail to match them against `NotificationType.NORMAL` because the mobile enum value is `'NOTMAL'`. However: (1) looking at the usage, `NotificationType` is only defined in the type, and the `type` field on `FirebaseNotification` is typed as `NotificationType` but never used as a filter key or switch discriminator in current code — it's stored but never branched on; (2) messaging push notifications are not wired today per spec §1.2 / §6.4. **Impact: currently none**, but becomes a bug the moment MESSAGE notifications are wired if the backend writes with correct spelling.

**Push notification for MESSAGE category:**
- `PushNotificationsInit.tsx:38-44` registers a `MESSAGE` notification category with an "Open" button.
- `handleNavigation` (line 76-77): on `category === 'MESSAGE'`, navigates to `/messages`.
- Deep-links to the messages page generically — does NOT open a specific chat (no `chatId` or sender info is extracted from the notification data).
- No reading of `users/<senderId>` from Firestore to enrich the notification — the notification content (title, body) comes from the push payload, not Firestore document reads.

**Safe from tightened rules:** Push notification handling does NOT read `users/{otherUserId}`.

---

## Section 10 — Message images integration

**When the user attaches images:**
1. Images are picked via `expo-image-picker` in `MessageInput.tsx`.
2. On send, `uploadImages` is called with `scope: 'chat'` and `chatId` **before** the message Firestore write — upload happens first.
3. If upload succeeds, `imageKeys` are placed into `MessageContent.blocks` as an `ImagesMessageBlock`.
4. Then `onSend(content)` fires the Firestore message write.
5. If upload fails, send is aborted entirely (early return).

**Image key field on message doc:** `content.blocks[n].imageKeys` (array of strings) on blocks of `type: 'images'`. Matches spec §3.2.

**Viewing images in received messages:**
- `MessageImages.tsx` calls `useViewTokenStore.getState().getToken(chatId)`.
- `getToken` calls `POST /api/secure/images/view-tokens` with `{ scope: 'chat', chatId }`.
- View-token response carries `token` and `expiresAt`; cached with 60s pre-expiry buffer.
- Image URLs built via `privateImageUrl(key, token)`.

**Does the view-token request include a `chatId`?** Yes — `{ scope: 'chat', chatId }`. The image-pipeline audit correctly identified this as a trust-boundary surface — server verifies participation before issuing the token.

---

## Section 11 — Translation coverage

Spot-check of chat-related strings:

| String purpose | Translated? | Namespace | Key |
|---|---|---|---|
| Empty chat list | YES | MESSAGES_PAGE | `chats.empty.list` |
| Empty filter result | YES | MESSAGES_PAGE | `chats.empty.filter.list` |
| Message input placeholder | YES | MESSAGES_PAGE | `message.input.placeholder` |
| Initial product message | YES | MESSAGES_PAGE | `initial.product.message` |
| Chat functions: profile | YES | MESSAGES_PAGE | `func.profile` |
| Chat functions: delete | YES | MESSAGES_PAGE | `func.delete` |
| Chat functions: report | YES | MESSAGES_PAGE | `func.report` |
| Chat functions: block | YES | MESSAGES_PAGE | `func.block` |
| Chat functions: unblock | YES | MESSAGES_PAGE | `func.unblock` |
| Search placeholder | **NO** | — | Hardcoded Serbian: `"Pretraži ćaskanja..."` |
| Select chat message | **NO** | — | Hardcoded Serbian: `"Izaberite ćaskanje kako bi videli poruke..."` |
| Load older messages | **NO** | — | Hardcoded Serbian: `"Učitaj starije poruke"` |
| Blocking notice | **NO** | — | Hardcoded Serbian: `"Ne mozete poslati poruku ovom korisniku zato sto ste blokirali korisnika..."` |
| Blocked-by notice | **NO** | — | Hardcoded Serbian: `"Ne mozete poslati poruku ovom korisniku zato sto vas je blokirao..."` |
| Image-only preview fallback | **NO** | — | Hardcoded Serbian: `"Nove poruke ..."` in `useChatStore.ts:54` |

**Summary:** 5 hardcoded Serbian strings in the messaging surface. Web fixed the `"Nove poruke ..."` in Brief 2. Mobile has not adopted translation keys for these strings.

---

## Section 12 — Dead code

| Item | File:line | Status |
|---|---|---|
| `checkIsBlocked` function | `useChatStore.ts:35-43` | **Dead** — exported but never called from any file. Spec §9 F17d marks it for deletion. |
| `testConnection` function | `firebaseClient.ts:38-50` | **Dead** — defined but never called (the call is commented out on line 52). Spec §9 F17d marks it for deletion. |
| `FirestoreUser.blocked` field | `src/lib/types/user/FirestoreUser.ts:6` | **Vestigial** — written on user creation as `[]` but never read by any code. Spec §11.11 notes this. |
| `loginWithFacebookFirebase` | `authService.ts:200-228` | **Dead** — entire function body is commented out, returns `null`. |
| `var previewMessage` (var keyword) | `useChatStore.ts:454` | Not dead code but uses `var` instead of `const`/`let` — pre-existing style issue. |

No stale Firestore subscription code found. No commented-out message handlers beyond the Facebook login (unrelated to messaging).

---

## Section 13 — Stores: `useChatStore` and `useChatBlockStore`

### `useChatStore`

**Holds:**
- `chats: ChatSummary[]` — current user's chat list (enriched with `withUser` data)
- `hasMoreChats: boolean` — pagination flag
- `lastVisibleChat` — Firestore cursor for pagination
- `loadingMoreChats: boolean`
- `messages: Record<string, MessageGroup[]>` — messages per chatId, grouped by day+sender
- `tempReceiver: UserInfoDTO | null` — target user for new chat flow
- `tempProductReason: ProductDetailsDTO | null` — product context for new chat
- `activeChatId: string | null` — currently viewed chat
- `newMessagesCount: number` — total unread across all chats
- `unsubChats: (() => void) | null` — chat list listener cleanup
- `unsubMessages: Record<string, () => void>` — per-chat message listener cleanups
- `userCache: Record<string, UserInfoDTO>` — in-memory cache of user data by firebaseUid
- `hasMoreMessages: Record<string, boolean>` — per-chat message pagination flags
- `lastVisibleMessage: Record<string, any>` — per-chat Firestore cursors
- `loadingMore: boolean`

**Initialized:** `subscribeToChats()` called from `ChatsInit.tsx` when `user` becomes truthy.

**Cleared:** `clearChatStore()` or `unsubscribeAll()` when `user` becomes null (sign-out) via `ChatsInit.tsx` effect cleanup.

**Persistence:** None — no AsyncStorage, no Zustand persist middleware. Fully in-memory, reconstructed from Firestore on each app launch.

### `useChatBlockStore`

**Holds:**
- `blocking: Record<string, boolean>` — users I blocked (keyed by their firebaseUid)
- `blockedBy: Record<string, boolean>` — users who blocked me
- `unsub: (() => void) | null` — combined listener cleanup

**Initialized:** `subscribe()` called from `ChatsInit.tsx` when `user` becomes truthy.

**Cleared:** `unsubscribe()` when `user` becomes null via `ChatsInit.tsx` cleanup.

**Persistence:** None.

---

## Section 14 — For Mastermind

### Cross-user `users/{userId}` reads — CLEAR

**No release-blocking cross-user Firestore reads found.** All user data enrichment goes through the backend endpoint `GET /auth/firebase/{uid}`, not direct Firestore document reads. The tightened `users/{userId}` rule (owner-only) will NOT break mobile's messaging surface.

### Trust-boundary findings

No mobile-specific trust-boundary violations. The `receiverId` residual gap (spec §4.4 / §7.3) exists identically on mobile as on web — it's a rule-level concern, not a client-level one.

### Spec-divergence findings

1. **Existing-chat send is not atomic on mobile.** Web Brief 2 wrapped the existing-chat path in a single `writeBatch`. Mobile still uses sequential `addDoc` + `setDoc` + `setDoc` calls. Partial failure can leave sidecars stale.

2. **`tempProductReason` not renamed to `tempProductContext`.** Spec §5.3 documents the rename that web adopted. Mobile retains the old name. Functional (the field works), but naming diverges from the canonical spec.

3. **No send-failure toast.** Spec §5.5 says the web shows a translated toast. Mobile silently rolls back the optimistic message.

4. **No `ChatsWatcher` equivalent.** Spec §5.6 says temp state is cleared on navigation away from `/messages`. Mobile does not clear temp state on navigation away.

5. **No chat-list "Load more" button.** Spec §5.11 — web adopted; mobile hasn't. Users with >15 chats cannot access older ones.

6. **`deleteChat` reorder not applied.** Spec §5.13 — mobile still deletes local state first, then awaits Firestore. Failure causes ghost removal (chat reappears on next listener fire).

7. **Mark-seen `seenLocal` mutation bug.** Spec §5.14 — mobile mutates the Set before `batch.commit()`. On failure, those messages won't be retried.

8. **No auto-linkify for URLs in messages.** Spec §5.8 / web Brief 3 added `linkify-it`. Mobile renders plain text.

9. **No block badge in chat list.** Spec §9 F17c — web added; mobile hasn't.

10. **No deleted-user fallback rendering.** Spec §6.7 — web renders "Deleted user" when `getUserData` 404s. Mobile would show blank/missing user info.

11. **Forward block field name mismatch.** Spec §3.4 says `blockerId` on forward docs. Mobile writes `blockedUserId`. Cosmetic (field is redundant with path) but diverges from spec.

### Runtime bugs noticed

1. **`NotificationType.NORMAL = 'NOTMAL'` typo** (`firebaseNotifications.ts:17`). Currently no consequence because messaging push notifications are unwired. Will become a bug if MESSAGE notifications are wired and backend uses correct spelling `'NORMAL'`.

2. **Hardcoded Serbian strings** — 5 instances in the messaging surface (Section 11). Violates the spec's locale-matching requirement (web ships four locales; mobile should match).

3. **`getActiveChat` fallback reads wrong fields from chat root doc** (`useChatStore.ts:718-719`). Reads `data.withUserFirebaseUid` and `data.withUser` which don't exist on the chat root document (only on the userchats sidecar). Web fixed this in Brief 2 (§5.9) by deriving `otherUid` from `data.users`. Mobile hasn't adopted the fix. Effect: any chat not in the most-recent-15 cache returns `chatFromDb` with `undefined` for `withUserFirebaseUid` and `withUser`.

4. **`ensureUserInFirestore` writes vestigial `blocked: []` field** (`authService.ts:96`). Firestore user docs carry this field but nothing reads it. Harmless but vestigial.

### Cross-feature observations (one line each)

- Image-pipeline audit scope: `MessageInput.tsx` image upload flow uses the image pipeline with `scope: 'chat'` — confirmed working per the image-pipeline integration point.
- User-deletion scope: `ChatsInit.tsx` correctly unsubscribes all listeners when `user` becomes null (confirmed by the user-deletion audit).
- General health: `loginWithFacebookFirebase` is dead code (entire body commented out) — general audit territory.

### Questions whose answers would change scope of a future messaging adoption chat

1. Should mobile adopt the atomic `writeBatch` for existing-chat sends (matching web Brief 2), or is the sequential approach acceptable for mobile given lower concurrency expectations?
2. Should mobile implement the `ChatsWatcher` pattern for temp-state clearing, or does expo-router's navigation model provide a different approach?
3. Should the `NOTMAL` typo be fixed in this mobile chat, or is it owned by the general audit?
4. Does the `getActiveChat` fallback bug affect real users today (i.e., do any mobile users have >15 chats)?

---

## Section 15 — Cleanup performed

`none needed` — read-only audit.

---

## Section 16 — Obsoleted by this session

`nothing` — read-only audit.

---

## Section 17 — Conventions check

- Part 4 (cleanliness): N/A this session (read-only audit).
- Part 4a (simplicity): N/A this session.
- Part 4b (adjacent observations): flagged in "For Mastermind" (dead code, hardcoded strings, `getActiveChat` bug, `NOTMAL` typo, `blocked` field vestigial).
- Part 6 (translations): five untranslated Serbian strings flagged.
- Part 7 (error contract): send-failure handling reviewed — no toast surfaced to user (spec divergence).
- Part 11 (trust boundaries): explicit check in Section 8. Full Firestore read inventory in Section 3. **No release-blocking trust-boundary violation found.**
- Other parts touched: Part 8 (architectural defaults) — confirmed mobile reuses same backend endpoints as web for admin and image surfaces.

---

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change (findings in this audit are for the future messaging adoption chat to consume, not for standalone issues.md entries — they are all spec-acknowledged mobile-not-yet-adopted items)
