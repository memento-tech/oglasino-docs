# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-19
**Task:** Audit the start-message flow and the broader messaging surface on `oglasino-web` to feed Mastermind's cross-repo seam analysis for the `[permission-denied]` defect on stage.

> Read-only audit. No code edits, no commits, no staged changes. The "Implemented" section is replaced with "Findings" organized by the brief's 10 scope items.

## Findings

### 1. The failing flow — snapshot listener (Error 1)

Snapshot listeners attached for any authenticated user, before the user ever clicks "Send Message":

- `src/components/client/initializers/ChatsInit.tsx:17-31` — runs whenever `useAuthStore.user` flips from null → set. Calls:
  - `useChatStore.subscribeToChats()` (`src/messages/store/useChatStore.ts:126-171`) → `onSnapshot` on `query(collection(db, 'userchats', user.firebaseUid, 'chats'), orderBy('lastUpdated','desc'), limit(15))`.
  - `useChatBlockStore.subscribe()` (`src/messages/store/useChatBlockStore.ts:12-43`) → two `onSnapshot` listeners: `collection(db, 'userblocks', user.firebaseUid, 'blocked')` and `collection(db, 'userblocksReverse', user.firebaseUid, 'blockedBy')`.
- `AppInit` (`src/components/client/initializers/AppInit.tsx:22-33`) mounts `ChatsInit` (and `ChatsWatcher`) in the `[locale]` layout — so all three listeners attach on every product detail page load for authenticated users.
- `Messages` (`src/messages/components/Messages.tsx:69-73`) calls `getActiveChat(activeChatId)` on mount. For a brand-new chat (no Firestore doc, only `tempReceiver`), `getActiveChat` (`useChatStore.ts:709-753`) does a one-time `getDoc` on `chats/{chatId}` (not a listener), then returns a synthesized `ChatSummary` from `tempReceiver`. It does **not** call `subscribeToMessages` on this path.
- `subscribeToMessages` (`useChatStore.ts:229-312`) is only invoked from `getActiveChat` paths 1 and 2 (local-cache hit and Firestore doc exists). For the new-chat path on the failing flow, no `chats/{chatId}/messages` listener attaches before the batch write.

Listener-by-Firestore path inventory (auth required for each):

| Path | Where attached | Required auth |
|------|----------------|---------------|
| `userchats/{uid}/chats` (chat-summaries collection) | `subscribeToChats`, line 126 | `request.auth.uid == uid` |
| `userblocks/{uid}/blocked` (subcollection) | `useChatBlockStore.subscribe`, line 25 | `request.auth.uid == uid` |
| `userblocksReverse/{uid}/blockedBy` (subcollection) | `useChatBlockStore.subscribe`, line 31 | `request.auth.uid == uid` |
| `chats/{chatId}/messages` (subcollection) | `subscribeToMessages`, line 243 (lazy) | likely `request.auth.uid in get(chats/{chatId}).data.users` |

**Most likely source of Error 1 — finding `suspicious-critical`:** The three always-on listeners (`userchats/{uid}/chats`, `userblocks/{uid}/blocked`, `userblocksReverse/{uid}/blockedBy`) all fire on every page mount for an authenticated user — including the product detail page. Any one of them firing `permission-denied` would surface as "Uncaught Error in snapshot listener" exactly as the operator described, with no relation to which page the user is on. If the listener is `userchats/{uid}/chats` and the chat doc the listener sees is the new one created by the failing send (after `subscribeToChats` retains its listener and a partial write somehow occurs — though batches should be atomic), the error would be tied to the send action. The seam is "Web assumes the rule for `userchats/{uid}/chats` (list/snapshot) allows the authenticated user to read their own subcollection."

A less likely but possible second source: if the listener fires Error 1 right around send time, it could be `subscribeToMessages` for the new chat — but only if the user has previously visited an existing chat and that listener is still attached. For the *first* time a user starts a conversation from product, `subscribeToMessages` is not yet attached for that chatId.

**Status:** `suspicious-critical` (Error 1) — most likely one of the three always-on listeners attached in `ChatsInit`. Cannot resolve to a single listener from web alone; depends on Firestore rule outcome the audit can't see.

### 2. The failing flow — send (Error 2)

Full path:

1. User clicks `<button>` in `MessageInput.tsx:293-295` (the "Send" Button) → `send()` (`MessageInput.tsx:117-176`).
2. `send()` validates non-empty, optionally uploads images via `uploadImages(images, 'chat', chatId, ...)` (`src/lib/images/uploadImages.ts`), assembles `MessageContent` with `blocks: [{type:'images', imageKeys}, {type:'text', text}]`, then calls `onSend(content)`.
3. `onSend` is supplied by `Messages.tsx:291`: `(content) => sendMessage(activeChatId ?? null, content)`.
4. `sendMessage` (`useChatStore.ts:372-603`):
   - Computes `resolvedChatId` from `tempReceiver`: `[user.firebaseUid, receiver.firebaseUid].sort().join('_')` (deterministic, sorted-pair, `useChatStore.ts:380`).
   - `chatExists` check: local cache OR `getDoc(chatRef)` (`useChatStore.ts:388-389`). `isNewChat = !chatExists`.
   - `receiver = state.tempReceiver` for new chat.
   - Optimistic UI insert.
   - **New-chat write (`useChatStore.ts:458-505`) — `writeBatch` containing four operations:**
     1. `batch.set(doc(db, 'chats', resolvedChatId), { users: [user.firebaseUid, receiver.firebaseUid], createdAt: serverTimestamp(), lastMessage, lastUpdated: serverTimestamp() })`. — Chat doc, identifies participants by **Firebase UID array** under field `users`.
     2. `batch.set(doc(db, 'userchats', user.firebaseUid, 'chats', resolvedChatId), { chatId, withUserFirebaseUid: receiver.firebaseUid, unreadCount: 0, lastMessage, lastUpdated: serverTimestamp() })`. — Sender userchats pointer.
     3. `batch.set(doc(db, 'userchats', receiver.firebaseUid, 'chats', resolvedChatId), { chatId, withUserFirebaseUid: user.firebaseUid, unreadCount: 1, lastMessage, lastUpdated: serverTimestamp() })`. — **Cross-user write** under the receiver's `userchats/{receiverUid}/chats`.
     4. `batch.set(messageRef, messageRequest)` where `messageRequest` (the `MessageRequest` type at `src/lib/types/chat/MessageRequest.ts`) is `{ content, senderId: user.firebaseUid, receiverId: receiver.firebaseUid, seen: false, createdAt: serverTimestamp() }`. **`productId` is added only if `state.tempProductReason` is truthy at the moment `sendMessage` runs.**

Order: all four in a single Firestore batch, atomic, committed together via `await batch.commit()`. If any one is denied, the entire batch fails with `permission-denied` and nothing is written.

Existing-chat write path (`useChatStore.ts:506-557`):
- `addDoc` the message to `chats/{chatId}/messages`.
- Then two independent `setDoc` calls (merge) on `userchats/{sender}/chats/{chatId}` and `userchats/{receiver}/chats/{chatId}`. Not a batch — three independent writes. (Adjacent observation: inconsistent atomicity vs. new-chat path.)

**Status:** `critical`. The exact payload of operation 1 above is the one that the Firestore rule for `chats/{chatId}` create must accept, and operation 4 is the one that may or may not have `productId`. Hypothesis details in items 4 and 5.

### 3. Recent git history on messaging code

`git log --since="90 days ago" --pretty=format:"%h %ad %s" --date=short -- src/messages/ src/components/client/StartMessageButton.tsx src/lib/types/chat/`:

| Hash | Date | Subject | Touches relevant to failing flow |
|------|------|---------|----------------------------------|
| 0c8bf76 | 2026-05-17 | "New changes, many bugs resolved" | **`src/messages/components/Messages.tsx` only**; wired the previously inert Report DropdownMenuItem's `onClick` to `openDialog(DialogId.REPORT_DIALOG, ...)`. No touch to `MessageInput`, `useChatStore`, or `MessageRequest`. |
| 96d8d81 | 2026-05-08 | "IMAGE_PIPELINE_FINISHED" | `MessageImages.tsx`, `MessageInput.tsx` (major rewrite to switch from `uploadChatImagesBatchParallel` → `uploadImages`, plus AbortController/progress UI), `useChatStore.ts` (added `cleanupOrphanImages` in catch). **Critically did NOT touch the `setTempProductReason(undefined)` mount effect in `MessageInput.tsx:57-68`** — that effect existed unchanged before this commit. |
| f69c724 | 2026-05-05 | "Audit cleanup: remove dead code, add SEO + error boundaries, restructure logging" | `MessageInput.tsx` (minor). |
| 5c24776 | 2026-05-05 | "Set up CI/CD and clean up lint baseline" | Some messaging files (lint baseline only). |
| 21efab8 | 2026-05-05 | "lets do it" | Some messaging files. |
| 73ab239 | 2026-04-30 | "Pre production commit" | Whole-`src/messages/` rewrite (Chats.tsx 47±, Messages.tsx 141±, useChatStore.ts 60±, MessageInput.tsx 83±, etc.) and `src/components/client/StartMessageButton.tsx`. |
| 465a328 | 2026-03-15 | "before base site" | Initial messaging commit. |

`git log --all -- src/lib/types/chat/MessageRequest.ts` → only `73ab239` ever touched `MessageRequest.ts`. The `productId?: number` field has not changed since pre-production.

`git log --all -- src/components/client/StartMessageButton.tsx` → only `73ab239` and `465a328`. The `setTempProductReason(product); setTempReceiver(withUser); router.push('/messages')` sequence in `StartMessageButton.tsx:60-70` has been stable since pre-production.

**Currently uncommitted, on `dev`, against the in-flight failing flow:** `M src/messages/components/Chats.tsx` and `M src/messages/components/Messages.tsx`. The Chats.tsx diff adds `chat.withUser?.displayName?.toLowerCase()` (optional chain on displayName) and a deleted-user fallback label. The Messages.tsx diff adds `peerPendingDeletion` / `peerBanned` state-derived UI, optional-chains `activeChat.withUser?`, and guards the dropdown with `{peer && (...)}`. None of these uncommitted edits touch the productId, batch, payload, or listener attach paths.

**Status of "what recent commit caused the regression":** None of the listed commits modified the `setTempProductReason(undefined)` mount effect — it has existed since the 2026-04-30 pre-production commit and is the root cause of the missing `productId` (item 4). So this is a long-standing latent bug, not a regression. **However**, the most recent commit `0c8bf76` (2026-05-17) touched only `Messages.tsx` and only added the Report dropdown handler. **It did not regress anything on the failing flow.** If stage's defect appeared "recently" without a code change to the path, the regression source is **not** in `oglasino-web`.

### 4. Hypothesis check — productId dropped from first message

**Verdict: CONFIRMED. `productId` is NOT in the new-chat first-message payload when the flow originates from a product page.**

Trace through the actual code:

1. `StartMessageButton.tsx:68-70` (no-block path) or `StartMessageButton.tsx:60-62` (post-unblock path): synchronous `setTempProductReason(product)` followed by an async, **un-awaited** `setTempReceiver(withUser)`, immediately followed by `router.push('/messages')`. After these lines: store has `tempProductReason = product` (set synchronously) and `tempReceiver = withUser` (after the `getDoc(chatRef)` promise inside `setTempReceiver` resolves, since the new-chat doc doesn't exist).
2. Route navigation to `/messages` → `ChatsPage` (`app/[locale]/(portal)/(protected)/messages/page.tsx`) renders. On desktop and mobile (when `activeChatId` is set), `Messages` renders.
3. `Messages` (`Messages.tsx:69-73`) effect calls `getActiveChat(activeChatId)`. For the brand-new chat (no Firestore doc), returns the `tempReceiver`-based synthetic `ChatSummary`. `activeChat.chatId` is the deterministic sorted-pair ID, so `MessageInput` is mounted with `chatId` set (`Messages.tsx:292`).
4. `MessageInput.tsx:57-68` mount effect:
   ```tsx
   useEffect(() => {
     if (!tempProductReason) return;
     setSuggestion(tMessages('initial.product.message', { value: tempProductReason.name }));
     setTempProductReason(undefined);  // <-- clears the store value
   }, [tempProductReason]);
   ```
   Fires synchronously on mount because `tempProductReason` is set. **It clears the store's `tempProductReason` to `undefined`** before the user even sees the textbox.
5. User edits the auto-populated suggestion (or doesn't), clicks Send. `MessageInput.send()` calls `onSend(content)` → `sendMessage(activeChatId, content)`.
6. `sendMessage` (`useChatStore.ts:373-374`): `const state = get();`. At this moment `state.tempProductReason` is already `undefined` (cleared in step 4, possibly minutes earlier).
7. `useChatStore.ts:499-501` (new chat path) and `useChatStore.ts:516-518` (existing chat path):
   ```ts
   if (state.tempProductReason) {
     messageRequest.productId = state.tempProductReason.id;
   }
   ```
   The guard is false. `productId` is **never** attached to the first message of a new conversation that originates from a product page.

The store API contributes to fragility: `setTempProductReason`'s type is `(product: ProductDetailsDTO | null) => void` but the caller passes `undefined`. Zustand accepts it at runtime — the store ends up with `tempProductReason: undefined`. The subsequent truthy-check still rejects undefined identically, so the bug is real regardless.

**Cross-repo seam:** Web assumes the Firestore rule for `chats/{chatId}/messages` create does NOT require `productId` on the message doc, OR that the rule for `chats/{chatId}` create does NOT require `productId` to be carried via the chat doc. If either rule requires `productId`, this is the root cause of Error 2.

**Note:** The hypothesis text in the brief mentions "the prior issues.md 2026-05-17 entry's described mechanism" — I did not read `issues.md`. I confirmed the mechanism from the code alone: the `setTempProductReason(undefined)` line in `MessageInput.tsx:67` runs at MessageInput mount, well before `sendMessage` reads `state.tempProductReason`.

### 5. Hypothesis check — participant field shape

**Verdict: DISCONFIRMED on web's side — the participant fields are consistent Firebase UIDs everywhere. However the seam to Firestore rules is unverified.**

Inventory of every participant field web writes or reads:

- **`chats/{chatId}` (chat doc), field `users`:** array of two Firebase UIDs, `[user.firebaseUid, receiver.firebaseUid]` (`useChatStore.ts:463`). Chat document ID is `[uid1, uid2].sort().join('_')`.
- **`userchats/{ownerUid}/chats/{chatId}`, field `withUserFirebaseUid`:** Firebase UID of the *other* party. Sender's doc has `withUserFirebaseUid = receiver.firebaseUid`; receiver's doc has `withUserFirebaseUid = user.firebaseUid`. The `{ownerUid}` path segment is also a Firebase UID.
- **`chats/{chatId}/messages/{msgId}`, fields `senderId` and `receiverId`:** Firebase UIDs (`useChatStore.ts:493-494`, `useChatStore.ts:510-511`).
- **`userblocks/{uid}/blocked/{blockedUid}`** and **`userblocksReverse/{uid}/blockedBy/{blockerUid}`:** all Firebase UIDs (`useChatBlockStore.ts:57-66`).

The recipient `firebaseUid` flows from `UserInfoDTO` (which the product page renders the owner via `ProductFunctions` → `StartMessageButton withUser={owner}` — `owner` is `UserInfoDTO`). `UserInfoDTO.firebaseUid` is the Firebase UID; `UserInfoDTO.id` is the backend integer ID. The two are distinct and the web code consistently uses `firebaseUid` for Firestore writes. There is no path in the failing flow where backend `id` would be written to Firestore where `firebaseUid` is expected.

`AuthUserDTO.firebaseUid` (the sender's identifier from `useAuthStore.user`) is also the Firebase UID and matches what the Firestore SDK has authenticated as (i.e., `request.auth.uid` server-side).

Hash this change recently? `git log --all -- src/messages/store/useChatStore.ts` shows `73ab239` (pre-production, 2026-04-30) and `96d8d81` (image pipeline, 2026-05-08). Neither commit altered the participant identifier scheme — the `users: [uid1, uid2]` shape and `withUserFirebaseUid` field have been stable since pre-production.

**Cross-repo seams (web assumes; cannot verify here):**
- Web assumes the Firestore rule for `chats/{chatId}` create accepts `users: [uid, uid]` as the participant identifier (specifically, that the rule checks `request.auth.uid in request.resource.data.users` rather than expecting fields named `participants`, `senderId`+`receiverId`, or `members`).
- Web assumes the rule for `userchats/{ownerUid}/chats/{chatId}` create allows the sender to also write to the **receiver's** subcollection (`userchats/{receiverUid}/...`). This is the only cross-user write in the new-chat batch — operations 1, 2, 4 are all writes by the sender to documents the sender owns; operation 3 (`useChatStore.ts:479-485`) is a write by the sender to the receiver's `userchats` subcollection. If the rule for `userchats/{ownerUid}/chats/...` requires `request.auth.uid == ownerUid` for writes, **operation 3 fails and the entire batch fails with `permission-denied`** regardless of any `productId`/`participants` shape question.

The receiver-subcollection write (op 3) is a strong candidate for Error 2 even if Hypothesis 4 (productId missing) is also true. The two can both be true; the batch fails on whichever rule denies first.

### 6. The broader messaging feature — full inventory

#### Routes / pages

- `app/[locale]/(portal)/(protected)/messages/page.tsx` — user-facing chat UI. Mounts `<Chats />` and `<Messages />`. Mobile: shows one or the other based on `activeChatId`. Desktop: both.
- `app/[locale]/admin/chats/[userId]/page.tsx` — admin: list of chats for a given user. Renders `ChatsListing` (no Firestore — uses backend HTTP service).
- `app/[locale]/admin/chats/messages/[userId]/[chatId]/page.tsx` — admin: messages of a specific chat. Renders `MessagesList`.

There is no owner-dashboard-specific messages view; user-facing `/messages` is the only customer surface.

#### Components

User-facing (`src/messages/components/`):
- `Chats.tsx` — sidebar list of `ChatSummary`s with search-by-displayName. Reads `chats`, `activeChatId` from `useChatStore`. Bug-shaped: `handleSelectChat` toggles activeChatId on any click rather than switching to the clicked chat — see "For Mastermind" Part 4b.
- `Messages.tsx` — chat header + message list + input. Manages active chat lifecycle, scroll, peer-state-driven send blocking (`cannotSend = blocking || blockedBy || peerPendingDeletion || peerBanned`). Wires `MessageInput.onSend → sendMessage(activeChatId, content)`. Dropdown actions: View profile, Delete chat, Report user, Block/Unblock.
- `MessageInput.tsx` — text + multi-image composer with HEIC conversion, AbortController-controlled image upload, optimistic suggestion auto-population from `tempProductReason`. **Contains the productId-clearing bug at lines 57-68.**
- `Message.tsx` — renders one `MessageContent`'s blocks (text + images).
- `MessageImages.tsx` — renders chat-image thumbnails using `useViewTokenStore.getToken(chatId)` to resolve private R2 URLs, with single-retry on `<img onError>` (401 fallback per contract §C.4).

Messaging-touching components outside `src/messages/`:
- `src/components/client/StartMessageButton.tsx` — the entry from product pages (used in `ProductFunctions.tsx`). Three behaviors: open login dialog if unauthenticated, open Info dialog with unblock CTA if blocking/blocked, else set temp state + navigate.
- `src/components/client/UserDetails.tsx` — also has a "Send message" CTA (`onSendMessage` at lines 62-102) that sets only `tempReceiver` (no `tempProductReason`) and navigates to `/messages`. Used on user profile pages.
- `src/components/client/buttons/AuthChatButton.tsx` — header chat icon with unread badge using `useChatStore.newMessagesCount`.
- `src/components/client/initializers/ChatsInit.tsx` — global subscriber.
- `src/components/client/initializers/ChatsWatcher.tsx` — clears chat state on `/favorites` navigation only.
- `src/components/admin/chats/{Chat.tsx,ChatsListing.tsx,MessagesList.tsx}` — admin views (backend HTTP-driven, not Firestore).

#### Stores

- `src/messages/store/useChatStore.ts` (Zustand) — the central messaging store. Slices: `chats` (ChatSummary list), `messages` (MessagesMap by chatId, grouped by sender+day), `tempReceiver` (UserInfoDTO during pre-existence-of-chat phase), `tempProductReason` (ProductDetailsDTO for first-message productId attachment — affected by the bug), `activeChatId`, `newMessagesCount` (sum of `unreadCount` across chats), `unsubChats`, `unsubMessages` (per-chat), `userCache` (firebaseUid → UserInfoDTO).
- `src/messages/store/useChatBlockStore.ts` (Zustand) — `blocking` and `blockedBy` maps, each subscribed via dual `onSnapshot` listeners.
- `src/lib/stores/viewTokens.ts` — view-token cache for private chat-image URLs (referenced by `MessageImages.tsx`).
- `src/lib/stores/uploadProgress.ts` — chat-image upload progress UI state.

#### Services

Firestore (direct, via `@firebase/firestore` from `firebaseClient`):
- `subscribeToChats` / `loadMoreChats` — read `userchats/{uid}/chats` (snapshot + paginated `getDocs`).
- `subscribeToMessages` / `loadMoreMessages` — read `chats/{chatId}/messages` (snapshot + paginated).
- `sendMessage` — write `chats/{chatId}` + two `userchats/.../chats/{chatId}` + `chats/{chatId}/messages/{msgId}`. Mix of `writeBatch` (new chat) and independent `setDoc`/`addDoc` (existing chat).
- `deleteChat` — `deleteDoc` on `userchats/{uid}/chats/{chatId}` only (does NOT delete the `chats/{chatId}` doc or messages — soft, sender-side only).
- Mark-seen — inside `subscribeToMessages`, a `writeBatch` flips `seen: true` on incoming unseen messages and zeros `unreadCount` on the user's `userchats/.../{chatId}` doc.
- `useChatBlockStore.{blockUser,unblockUser}` — `writeBatch` on `userblocks` + `userblocksReverse`.
- `checkIsBlocked` (`useChatStore.ts:37-45`) — two `getDoc` reads on `userblocks/{userA}/blocked/{userB}` and `userblocks/{userB}/blocked/{userA}`. **Note:** this function is defined and exported but **not referenced anywhere in src/** — see "For Mastermind" Part 4b.

Backend HTTP (via `BACKEND_API`):
- `src/lib/admin/lib/service/chatsService.ts`:
  - `getReactAdminUserChats(userId, lastUpdated?)` → `GET /secure/admin/chats/user/{userId}/chats?lastUpdated=...`
  - `getReactAdminChatMessages(chatId, lastCreatedAt)` → `GET /secure/admin/chats/{chatId}/messages?lastCreatedAt=...`
- `src/lib/admin/lib/service/next/chatsService.ts`:
  - `getNextAdminChatMessages(chatId, lastCreatedAt?)` — same endpoint as above, used by the next-router admin variant.
- `src/lib/service/reactCalls/userService.ts:getUserForFirebaseUid(firebaseUid)` → `GET /auth/firebase/{firebaseUid}` — used by `useChatStore.getUserData` to hydrate `UserInfoDTO` for `withUser` from a Firebase UID. Maps Firebase UID → backend `UserInfoDTO` (which has both `id` and `firebaseUid`).
- `src/lib/images/uploadImages.ts` — `uploadImages(files, 'chat', chatId, ...)` triggers `POST /api/secure/images/upload-tokens` (backend) and then PUTs to Cloudflare Worker. Image keys are then stored inside `MessageContent.blocks[].imageKeys`.

#### Types

- `src/lib/types/chat/MessageRequest.ts` — `{ content, senderId, receiverId, productId?, seen, createdAt }`. `senderId`/`receiverId` are Firebase UIDs; `productId` is a backend integer. The only optional field.
- `src/lib/types/chat/ChatSummary.ts` — `{ chatId, withUserFirebaseUid, withUser, lastMessage?, lastUpdated?, unreadCount }`. Web's in-memory shape; Firestore chat doc shape is different (`users[]` array, no `withUser*`).
- `src/lib/types/chat/Message.ts` — `{ id, content, senderId, receiverId, sender, receiver, seen, createdAt }`. Carries hydrated `sender`/`receiver` UserInfoDTO/AuthUserDTO post-fetch.
- `src/lib/types/chat/MessageContent.ts` — `{ blocks: MessageBlock[] }`.
- `src/lib/types/chat/MessageBlock.ts` — discriminated `{ type: 'text'|'images', ... }`.
- `src/lib/types/chat/MessageGroup.ts` — `{ dayString, sender, messages }`; UI-only aggregation type.
- `src/lib/types/chat/ChatStore.ts` — Zustand store shape (`tempReceiver`, `tempProductReason`, `newMessagesCount`, etc.).
- `src/lib/types/chat/ChatBlockStore.ts` — `useChatBlockStore` shape.
- `src/lib/types/chat/admin/{ChatUser,ChatsDataResponse,ChatMessage,ChatMessagesResponse,MessageGroup,ChatData}.ts` — admin HTTP response types. Admin reads use `sender`/`receiver: ChatUser` (not the user-facing `senderId`/`receiverId` Firebase UIDs).

#### Surfaces in other features

- `StartMessageButton` — instantiated in `src/components/client/ProductFunctions.tsx:49` on every product detail page (when `user?.id !== owner.id`). The only failing-flow entry point.
- `UserDetails.tsx` — "Send message" CTA on user profile page (`/user/{id}`); calls `setTempReceiver` (no product). On a successful send via this path, the message has no `productId` — by design.
- `AuthChatButton` — header unread badge (`newMessagesCount` from `useChatStore`).
- Admin: `ChatsListing`, `Chat`, `MessagesList` — read-only admin surfaces, no Firestore involvement.

#### Realtime mechanisms

- `onSnapshot` x 4 attach points: `subscribeToChats` (1), `useChatBlockStore.subscribe` (2 listeners), `subscribeToMessages` (1 per active chat).
- Dispose semantics: `unsubChats` and `unsubMessages` are stored as cleanup functions on the store; `clearChatStore`, `unsubscribeAll`, `unsubscribeFromMessages`, and `useChatBlockStore.unsubscribe` invoke them. `ChatsInit`'s effect cleanup calls `unsubscribeAll` and `unsubscribeBlocks` on user-flip-to-null.
- The mark-seen `writeBatch` inside `subscribeToMessages` (`useChatStore.ts:297-308`) issues writes to `chats/{chatId}/messages/{msgId}` (flip `seen: true`) and `userchats/{user.firebaseUid}/chats/{chatId}` (zero `unreadCount`).

### 7. Identity in messaging context

- **Chat document participants:** Firebase UIDs only (`users: [uid1, uid2]`, `useChatStore.ts:463`).
- **Chat ID:** `[uid1, uid2].sort().join('_')` — deterministic from Firebase UIDs.
- **What the Firestore client sends:** the authenticated Firebase user via `FirebaseClient.auth` (Firebase SDK auto-attaches the ID token; `request.auth.uid` server-side is the Firebase UID). Web uses `useAuthStore.user.firebaseUid` for write payloads — this value comes from the backend `AuthUserDTO` (`syncUserToBackend` returns it) which is **the same** Firebase UID as the SDK's `auth.currentUser.uid`. The sync invariant relies on backend echoing the verified `firebaseUid`.
- **Recipient identifier on product page:** `ProductFunctions` receives `owner: UserInfoDTO` (which carries both `id` and `firebaseUid`); `StartMessageButton` uses `owner.firebaseUid` for `tempReceiver` and `owner.id` is never written to Firestore. Backend integer `id` is only used for non-Firestore concerns (admin lookups, user profile routing, report payloads).
- **Risk of UID/ID confusion:** Low on web's side. All Firestore write call sites use `.firebaseUid`. The only places backend `id` is referenced anywhere near messaging are admin UI paths (HTTP-only) and dropdown navigation (`/user/{peer.id}`).

### 8. Recently fixed bugs in messaging

I did not read prior session summaries or `issues.md`; this section is based on git log of code paths in the failing flow.

- **0c8bf76 (2026-05-17):** Wired Report DropdownMenuItem in `Messages.tsx` to `openDialog(REPORT_DIALOG, ...)`. Touches only the Messages component dropdown; does not affect send path or productId/listener attach. No regression risk to the failing flow.
- **96d8d81 (2026-05-08):** Image-pipeline rewrite (HEIC support, AbortController, progress UI, orphan-cleanup on send failure in catch block of `sendMessage`). Did not alter the `MessageRequest` shape, the productId guard, the `tempProductReason` clearing, or the participant fields. Did add `cleanupOrphanImages` on send-failure path (`useChatStore.ts:587-590`), which is defensive and only fires after a write throws. No regression to first-message creation.
- **73ab239 (2026-04-30):** Pre-production rewrite — established all current patterns: `users[]` chat doc shape, `withUserFirebaseUid` field, deterministic chatId, `MessageRequest.productId?`, `MessageInput` mount-clear effect. The latent productId bug originates here.

The "silent text-only-send failure" and "tempProductReason clearing" issues referenced in the brief (2026-05-14 and 2026-05-17 entries in issues.md, which I did not read) — based on the code alone, the `tempProductReason` clearing pattern at `MessageInput.tsx:67` matches the symptom described in the brief and is unchanged by recent commits. The "silent text-only-send failure" symptom is not visible in current code (`MessageInput.send()` does not silently exit on text-only; the `if (!text.trim() && images.length === 0) return` guard correctly allows text-only). If the prior "silent text-only-send" issue was the broader `permission-denied` swallowed in the `catch` of `sendMessage` (`useChatStore.ts:581-602` — only `console.error` + optimistic rollback, no toast), that pattern is **still present** and would suppress the user-visible error from Error 2. See "For Mastermind" Part 4b.

### 9. Deploy state and environment differences

I cannot determine the stage deploy state from web's source code alone. The defect is reproducible on stage; the same code on `dev` (current branch) is what stage runs from if Vercel auto-deploys `dev` to stage (Vercel project config is not in this repo). The repo has:

- `.env.local` / `.env.development.local` / etc. — gitignored, not visible.
- `.env.production` — gitignored, not visible.
- Firebase config is read from `NEXT_PUBLIC_FIREBASE_*` env vars (`src/lib/config/firebaseClient.ts:15-23`). Different Firebase **project IDs** between local and stage would point at different Firestore databases with possibly different security rules and data shapes.

No conditional code in `src/messages/` reacts to `NODE_ENV` or stage-vs-prod. Behavioral divergence between local and stage on this code is unlikely to be code-level — it would be config-level (env vars → different Firebase project → different rules).

**Cannot answer from web alone:** what SHA stage runs, what Firebase project stage points at, what rules are deployed to that project. These belong in the firestore-rules audit and Mastermind's seam analysis.

### 10. Local-vs-stage reproducibility

Same answer as item 9: web code is environment-agnostic. The only environment-specific configuration is `NEXT_PUBLIC_FIREBASE_*` (which Firebase project), the backend API base URL, and FCM keys. If stage and local use the same Firebase project, the same rules apply and the same data is read. If they differ, web's code does not branch — but the rules deployed to each project may differ.

To answer the "stage fails / local works (untested)" framing definitively, the operator should reproduce on local against the same Firebase project stage uses. If local fails the same way → it's a code or rules issue (not env). If local passes → it's a rules-deployment or data-shape difference between projects, not a web code issue.

---

## Files touched

- none (read-only)

## Tests

- n/a

## Cleanup performed

- none needed

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): N/A this session (read-only)
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): flagged in "For Mastermind"
- Part 6 (translations): N/A this session
- Part 7 (error contract): N/A — the failing flow's errors are Firestore SDK errors, not the backend codes contract
- Other parts touched: Part 11 (trust boundaries) — covered implicitly in item 7 (Firebase UID is the trust-bearing identifier server-side)

## Known gaps / TODOs

- None. The audit is bounded by the brief's read-only constraint and by the prohibition on reading the four config files and prior session summaries. Anything in those files would refine but not contradict the findings above.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only)
  - Considered and rejected: nothing (read-only)
  - Simplified or removed: nothing (read-only)

- **Hypothesis verdicts (brief items 4 and 5):**
  1. **Hypothesis 4 (productId dropped) — CONFIRMED.** `MessageInput.tsx:57-68`'s mount effect clears `tempProductReason` on mount, which is well before `sendMessage` reads it. The current code at `useChatStore.ts:499-501` and `:516-518` therefore omits `productId` from the first message of a new conversation that originated from a product page. The bug has existed since the 2026-04-30 pre-production commit — it is latent, not a regression. If stage's Firestore rule for `chats/{chatId}` create or `chats/{chatId}/messages` create requires `productId`, this is the root cause of Error 2.
  2. **Hypothesis 5 (participant field shape) — DISCONFIRMED on web's side.** Participants are consistent Firebase UIDs, written as `users: [uid1, uid2]` on the chat doc, `senderId`/`receiverId` on the message doc, and `withUserFirebaseUid` on the userchats pointer. Shape has been stable since pre-production. **However**, the new-chat batch includes a cross-user write (operation 3: `setDoc` on `userchats/{receiverUid}/chats/{chatId}`) by the sender. If the rule for `userchats/{ownerUid}/chats/{...}` requires `request.auth.uid == ownerUid` for writes, the sender cannot create the receiver's pointer doc, and the batch fails with `permission-denied`. This is independent of Hypothesis 4 and could be the (or another) root cause.

- **Cross-repo seams web assumes (each must be verified against `oglasino-firestore-rules`):**
  - S1: `chats/{chatId}` create allows the sender when payload is `{ users: [auth.uid, otherUid], createdAt, lastMessage, lastUpdated }` with no `productId` field on the chat doc.
  - S2: `chats/{chatId}/messages` create allows the sender when message payload is `{ content, senderId: auth.uid, receiverId: otherUid, seen: false, createdAt }` with `productId` **optional**.
  - S3: `userchats/{senderUid}/chats/{chatId}` create allows `request.auth.uid == senderUid`.
  - S4: **`userchats/{receiverUid}/chats/{chatId}` create allows `request.auth.uid == senderUid` (cross-user write).** This is the unusual one — strong candidate for Error 2.
  - S5: `userchats/{uid}/chats` snapshot (listened from `subscribeToChats`) allows `request.auth.uid == uid` reads.
  - S6: `userblocks/{uid}/blocked` and `userblocksReverse/{uid}/blockedBy` snapshots (listened from `useChatBlockStore.subscribe`) allow `request.auth.uid == uid` reads.
  - S7: The chat-doc shape uses `users[]` not `participants[]` / `members[]`. Rules must check `auth.uid in resource.data.users`, not a differently-named field.

- **Cross-repo seam (backend):**
  - B1: `GET /auth/firebase/{firebaseUid}` (used by `useChatStore.getUserData`) returns a `UserInfoDTO` with `firebaseUid` and `id`. Web treats `id` as the backend integer (used for navigation/admin only) and `firebaseUid` as the Firestore identifier. Backend must keep this contract; in particular, `firebaseUid` must match the value the Firebase SDK has authenticated as for the same user.

- **Recent-commit regression assessment:** None of the messaging-touching commits in the last 90 days modified the failing-flow code (`StartMessageButton`, `MessageInput.tsx:57-68`, `useChatStore.sendMessage`'s payload, the listener attach points). If stage's defect appeared without a corresponding web commit, the regression is in `oglasino-firestore-rules` or in stage env/data, **not in this repo**. If the defect has always been present and the operator only now noticed it, the latent productId bug at `MessageInput.tsx:67` is the most likely cause.

- **Recommended next-brief priorities for Mastermind:**
  1. Get the firestore-rules audit verdict on rules for `chats/{chatId}` create, `chats/{chatId}/messages` create, and especially `userchats/{ownerUid}/chats/{chatId}` create (cross-user write at S4). Resolve the seam.
  2. If rules require `productId` on first message of new chats originating from product pages: fix Hypothesis 4 by either (a) reading `tempProductReason` *inside* `sendMessage` before any state-cleanup runs (eliminating the gap), or (b) plumbing `productId` through `MessageContent`/`onSend`/`sendMessage` arguments rather than via a separate transient store slice. Option (b) eliminates a class of bugs.
  3. If rules forbid cross-user `userchats` writes (S4): backend must own creation of both userchats pointers via a server endpoint, or rules must permit the sender-writes-receiver case (least-privilege rule with constraints on `withUserFirebaseUid == request.auth.uid` and `chatId.split('_')` containing `request.auth.uid`).

- **Part 4b adjacent observations (out-of-scope, not fixed):**
  - **Severity high — silent swallow of send errors in `useChatStore.ts:581-602`:** the `catch` only `console.error`s and rolls back optimistic UI. The user sees their message disappear with no toast or dialog. This is consistent with the "silent text-only-send failure" symptom in the brief's item 8. Recommend adding a user-visible error notification (toast or dialog) in this catch.
  - **Severity high — type signature of `setTempProductReason` accepts only `ProductDetailsDTO | null` but the caller in `MessageInput.tsx:67` passes `undefined`.** Loose at runtime, type-unsafe. The `tempProductReason` mutation pattern (set-in-callsite-then-consume-in-effect-then-clear) is fragile; the `MessageInput` mount effect is the entire bug surface for the productId issue. Refactor opportunity: pass productId via `MessageContent` or `onSend(content, { productId })`.
  - **Severity high — exported function `checkIsBlocked` (`useChatStore.ts:37-45`) is not referenced anywhere in src/.** Dead code; either consume it from `sendMessage` (currently uses `useChatBlockStore.blocking/blockedBy` snapshots — race-y for a one-shot guard) or delete it.
  - **Severity high — `Messages.tsx:114` calls `setActiveChatId(undefined)` but the signature is `(chatId: string | null) => void`.** TypeScript doesn't catch it because of the destructure assignment loosening; the store value becomes `undefined`. Use `null`.
  - **Severity high — `Chats.tsx:33-39` `handleSelectChat` toggles `activeChatId` on click rather than switching to the clicked chat.** If a chat is already active, clicking *any* chat in the list deselects rather than switching. User-facing bug.
  - **Severity medium — `useChatStore.ts:730` `getActiveChat` Firestore-exists path reads `data.withUser` from the chat doc, but the chat doc only stores `users[]`, `createdAt`, `lastMessage`, `lastUpdated` (see `useChatStore.ts:462-467`).** `data.withUser` will always be `undefined`. The synthesized `ChatSummary` from that path will have `withUser: undefined`, which the UI then renders via the optional-chaining fallbacks added in the in-flight uncommitted diff on `Messages.tsx`. Effectively dead branch; should hydrate via `getUserData(otherFirebaseUid)` from the `users` array.
  - **Severity medium — `useChatStore.subscribeToMessages` mark-seen path** (`useChatStore.ts:297-308`) issues writes for every snapshot tick, including the user's own messages. The `unseen` filter (`m.receiverId === user.firebaseUid && !m.seen`) correctly filters to incoming-only, but the `seenLocal` Set lives in the closure of one `subscribeToMessages` invocation and is reset on each re-subscribe. Idempotent enough, but worth checking.
  - **Severity medium — `deleteChat` (`useChatStore.ts:605-642`) deletes only `userchats/{currentUser}/chats/{chatId}`** — not the chat doc, not the messages, not the other user's pointer. The chat persists on the other side; sender no longer sees it. Matches a common "soft-delete on my side only" pattern but is subtle and worth documenting in `oglasino-docs/reference/messaging.md`.
  - **Severity low — `subscribeToChats` orderBy/limit + `subscribeToMessages` orderBy/limit set LIMIT to 15**, hardcoded. `chats/{chatId}/messages` is queried as `orderBy('createdAt','desc')` — Firestore requires `createdAt` to be a server-timestamp on every message; new-chat first message uses `serverTimestamp()` (`useChatStore.ts:496`, `:513`) so this holds.
  - **Severity low — getPreview** (`useChatStore.ts:47-57`) hardcodes a Serbian fallback string "Nove poruke ..." rather than going through `next-intl`. Translation seam.

- **Definitive answer to "is this fixable in `oglasino-web` alone?":** Probably not. If S4 (rules forbid cross-user userchats writes) is the issue, the fix is in rules and/or backend. If S2 (`productId` required on first-message create) is the issue, the fix involves rules accepting messages without `productId` **and/or** the productId-clearing bug at `MessageInput.tsx:67` (which is fixable in this repo). Both can be true; sequence the fixes only after the rules audit returns.

- **No drafted config-file text. No config-file changes required by this session.**
