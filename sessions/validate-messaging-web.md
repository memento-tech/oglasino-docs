# Spec validation: messaging (web)

**Repo:** `oglasino-web`
**Branch:** `dev` (audited as checked out; not switched)
**Mode:** read-only audit — no code touched
**Date:** 2026-05-30
**Source of truth:** the `oglasino-web` code. Spec claims are taken from Brief 2a; not cross-checked against `oglasino-docs`.

Each claim is marked **MATCHES**, **DRIFTED**, or **NOT FOUND** with a `file:line` citation. The chat store referenced by the spec as `useChatStore.ts` lives at `src/messages/store/useChatStore.ts`.

---

## Data model (spec §3)

### 1. Message doc shape — **MATCHES**

- **Type:** `src/lib/types/chat/MessageRequest.ts:4-11` — `{ content, senderId, receiverId, productId?, seen, createdAt }`. `content.blocks[]` block type is `'text' | 'images'` (`src/lib/types/chat/MessageBlock.ts:2`).
- **Write site:** `src/messages/store/useChatStore.ts:456-462` builds the `MessageRequest`; `:525` writes it via `batch.set(messageRef, messageRequest)`. `createdAt` is `serverTimestamp()` (`:461`), `seen: false` (`:460`).

The on-the-wire doc shape is exactly `senderId`, `receiverId`, `content` (blocks of `text`/`images`), optional `productId`, `seen` (bool), `createdAt`. (The richer `Message` interface in `Message.ts` — with `id`/`sender`/`receiver` objects — is the client-enriched read model, not the write shape.)

### 2. `productId` is optional and a number — **MATCHES**

- `src/messages/store/useChatStore.ts:464-466`: `if (capturedProductContext) { messageRequest.productId = capturedProductContext.id; }`. No `else`, so when there is no product context the field is **omitted entirely** (not null, not 0).
- Type is `productId?: number` (`MessageRequest.ts:8`); `capturedProductContext.id` is `number` (`src/lib/types/product/ProductOverviewDTO.ts:9` `id: number`, inherited by `ProductDetailsDTO`).

### 3. Userchats sidecar shape — **MATCHES**

- `src/messages/store/useChatStore.ts:484-498` (new-chat path) writes exactly `chatId`, `withUserFirebaseUid`, `unreadCount`, `lastMessage`, `lastUpdated` on both sender and receiver sidecars.
- Field is `withUserFirebaseUid` (singular) — `:486`, `:493`; also `ChatSummary.ts:5`. Not `withUser`, not `users`.

### 4. Block index forward-doc field name — **DRIFTED**

- **Spec §3.4 says:** the forward doc `userblocks/{ownerUid}/blocked/{blockedUid}` carries `blockerId`.
- **Code says:** the forward doc carries **`blockedUserId`** — `src/messages/store/useChatBlockStore.ts:57-60` (`batch.set(doc(db,'userblocks',user.firebaseUid,'blocked',uid), { blockedUserId: uid, createdAt })`). The field `blockerId` is written to the **reverse** doc `userblocksReverse/{blockedUid}/blockedBy/{ownerUid}` instead (`:62-65`).
- **Canonical answer for the mobile reconciliation:** web's forward-doc field is **`blockedUserId`** — i.e. web matches the mobile audit's `blockedUserId`, not the spec's `blockerId`. (`blockerId` exists in the system, but only on the reverse index.)

---

## Client flows (spec §5)

### 5. chatId algorithm — **MATCHES** (line numbers drifted)

- Algorithm `[uidA, uidB].sort().join('_')` confirmed.
- **Real current sites:** `src/messages/store/useChatStore.ts:380` (`sendMessage` → `resolvedChatId`) and `:671` (`setTempReceiver`). Spec said `:380` and `:683`; the second is now `:671`. (`:399` uses `split('_')` to *derive* the counterparty, not to compute the id.)

### 6. `tempProductContext` rename + type — **MATCHES**

- `src/lib/types/chat/ChatStore.ts:17`: `tempProductContext: ProductDetailsDTO | null` — exactly this name and type, no `undefined` variant.
- Grep for `tempProductReason` across `src`/`app`: **none found.**

### 7. MessageInput mount does NOT clear temp context — **MATCHES**

- `src/messages/components/MessageInput.tsx:57-68`: the effect reads `tempProductContext` to build the `suggestion` string and explicitly does **not** clear it (comment `:60-62`). The mount effect at `:70` only sets `mounted`. Nothing in the component clears `tempProductContext`.

### 8. `sendMessage` reads temp context synchronously at top — **MATCHES**

- `src/messages/store/useChatStore.ts:371-375`: `const state = get();` then `const capturedProductContext = state.tempProductContext;` at the top of the function; attached to the message at `:464-466`.

### 9. New-chat send is one atomic `writeBatch` of 4 ops — **MATCHES**

Single batch declared at `src/messages/store/useChatStore.ts:472`, committed once at `:527`. The 4 `set` shapes (new-chat branch):

1. **chat root** (`:477-482`): `{ users: [user.firebaseUid, receiver.firebaseUid], createdAt: serverTimestamp(), lastMessage: previewMessage, lastUpdated: serverTimestamp() }`
2. **sender sidecar** (`:484-490`): `{ chatId, withUserFirebaseUid: receiver.firebaseUid, unreadCount: 0, lastMessage, lastUpdated }`
3. **receiver sidecar** (`:492-498`): `{ chatId, withUserFirebaseUid: user.firebaseUid, unreadCount: 1, lastMessage, lastUpdated }`
4. **message doc** (`:525`): the `MessageRequest` (`{ content, senderId, receiverId, seen:false, createdAt:serverTimestamp(), productId? }`)

Note: the chat-root doc carries `users[]` (not `withUserFirebaseUid`); the sidecar fields live on `userchats`.

### 10. Existing-chat send is one atomic `writeBatch` — **MATCHES**

Same single batch object (declared `:472`, committed `:527`); the `else` branch (`:499-523`) differs only in the three non-message sets. Exact shape — **4 ops total, 3 with `{merge:true}`, 1 without:**

1. **chat root** (`:500-504`): `set(chatRef, { lastMessage, lastUpdated: serverTimestamp() }, { merge: true })`
2. **sender sidecar** (`:508-512`): `set(senderChatRef, { lastMessage, lastUpdated }, { merge: true })`
3. **receiver sidecar** (`:514-522`): `set(receiverChatRef, { lastMessage, lastUpdated, unreadCount: increment(1) }, { merge: true })` — **`unreadCount` uses `increment(1)` on the receiver sidecar** (`:519`).
4. **message doc** (`:525`): `set(messageRef, messageRequest)` — **no `{merge}`** (fresh doc).

The pre-fix "create-if-missing-else-merge" branch is gone; the code comment at `:506-507` notes `setDoc`-with-merge handles both create and update. Fully atomic.

### 11. Send-failure handling — **DRIFTED** (4 of 5 hold; the toast is not in the store catch)

Catch block `src/messages/store/useChatStore.ts:561-587`:

- (a) **logs** — `console.error('Send failed', e)` (`:562`). ✔
- (b) **fires `notify.error()` with a translation key** — **NOT in the catch.** The store catch *re-throws* (`:586`); the `notify.error({ title: tMessages('messages.send.failed.toast') })` actually fires in the React caller `src/messages/components/Messages.tsx:315-318` (the `MessageInput onSend` wrapper). ✘ (relative to "the sendMessage catch block fires it")
- (c) **rolls back the optimistic message** — `:574-582` filters out `tempId`. ✔
- (d) **orphan-image cleanup if image keys** — `:567-569` (`cleanupOrphanImages(imageBlock.imageKeys)` when the message had an images block). ✔
- (e) **does NOT clear `tempReceiver`/`tempProductContext`** — confirmed; comment `:572-573` says they stay set; no clear in the catch. ✔

So the only drift: notify.error is delegated to the caller via re-throw, not performed inside `sendMessage`'s catch (by design — comment `:584-585` notes translations only exist at the React boundary). Same pattern as `deleteChat`.

### 12. `ChatsWatcher` temp-state clearing — **MATCHES**

- `src/components/client/initializers/ChatsWatcher.tsx` exists; watches `pathname` via `usePathname()` (`:10`).
- **Previous-path guard: yes** — `prevPathnameRef` (`:15`, updated `:18-19`).
- **Clear condition** (`:24`): `if (prev === '/messages' && pathname !== '/messages')` → clears `setActiveChatId(null)`, `clearTempReceiver()`, `setTempProductContext(null)` (`:25-27`). So it clears on the transition *out of* `/messages`, and does not clear when navigating *into* `/messages`.
- One nuance vs spec wording: `MESSAGES_PATH` is `/messages` (no locale) because the i18n `usePathname` (`@/src/i18n/navigation`) returns a locale-stripped path; the comparison is correct. The trigger is the leave-transition, not "any non-messages route" (it won't re-clear while already off `/messages`).

### 13. `getActiveChat` Firestore fallback — **MATCHES**

- `src/messages/store/useChatStore.ts:719-729`: reads `data.users` (`:721`), `otherUid = users.find((u) => u !== user.firebaseUid)` (`:722`), then `withUser = await state.getUserData(otherUid)` (`:729`). It does **not** read `withUserFirebaseUid`/`withUser` off the chat root (comment `:710-713` documents that the root carries `users[]` only).

### 14. `deleteChat` order — **MATCHES**

- `src/messages/store/useChatStore.ts:594-623`: `await deleteDoc(...)` (`:598`) **first**, then the local-store mutation (`:603-623`). Catch (`:624-629`) logs and re-throws so the caller toasts (`Messages.tsx:117-124`, `onDelete`). Firestore-first, toast-on-failure confirmed.

### 15. mark-seen `seenLocal` ordering — **MATCHES**

- `src/messages/store/useChatStore.ts:300-303`: `batch.commit().then(() => unseen.forEach((m) => seenLocal.add(m.id)))`. The `seenLocal` Set mutation runs in the `.then` **after** `batch.commit()` resolves; on `.catch` it stays clean so the next listener fire re-queues (comment `:298-299`).

---

## Auto-linkify (spec §5.8)

### 16. Library + config — **MATCHES**

- **Library/version:** `linkify-it ^5.0.0` (+ `@types/linkify-it ^5.0.0`) in `package.json`. Imported as `LinkifyIt` in `src/messages/utils/linkify.ts:1`.
- **Schemes linkified:** `http://`, `https://`, and `www.`-style fuzzy links. The defaults `mailto:` (`:10`), `ftp:` (`:11`), and protocol-relative `//` (`:12`) are explicitly disabled (`linkify.add(scheme, null)`); `fuzzyEmail` is off (`:13`) so bare `foo@bar.com` is not linkified.
- **`javascript:` / `data:` / `vbscript:` excluded:** yes — not in linkify-it's default schema, so they remain unmatched (comment `:8-9`; verified by tests `src/messages/utils/linkify.test.ts:55-74`).
- **Anchor attributes (render site `src/messages/components/Message.tsx:29-34`):** `target="_blank"`, `rel="noopener noreferrer"`, `className="break-all underline"`.
- **Anchor text:** verbatim URL, no masking — token `value` is `m.raw` (`linkify.ts:33`), rendered as the anchor child `{token.value}` (`Message.tsx:34`). The `href` uses `m.url` (linkify-it normalized — e.g. `www.example.com` → `http://www.example.com`), but the displayed text is the raw string as typed. `break-all` is CSS wrapping only, not truncation.

---

## Backend-facing / admin (spec §6.2)

### 17. Admin receiver — **MATCHES** (backend-resolved per row; no first-doc-receiver bug)

- The web admin chat list is fully backend-resolved per row. `src/components/admin/chats/ChatsListing.tsx:72-74` maps each `ChatData` to `<Chat chatData={chat}>`; `src/components/admin/chats/Chat.tsx:22,24` renders `chatData.sender.displayName` / `chatData.receiver.displayName` **per row**. Each `ChatData` carries its own `sender`/`receiver` (`src/lib/types/chat/admin/ChatData.ts:5-6`, `ChatUser.ts`). There is no client-side receiver derivation, so the web admin surface does not carry a shared first-doc-receiver bug.

---

## Translations (spec §8, §9)

> Translations in this repo are backend-seeded and fetched at runtime (no local JSON). "Key exists and is consumed" = the key string is passed to a `next-intl` `t()` call in web code. All MESSAGES_PAGE keys below are under the `MESSAGES_PAGE` namespace unless noted.

### 18. MESSAGES_PAGE keys

| String | Verdict | Web key (exact) | Citation |
|---|---|---|---|
| Send-failure toast | **MATCHES** | `messages.send.failed.toast` | `Messages.tsx:317` (send), `:122` (delete) |
| Image-only preview fallback | **MATCHES** | `messages.image.preview.fallback.label` | `Chats.tsx:53` |
| Chat-list "Load more" | **MATCHES** | `chats.load.more` (in MESSAGES_PAGE) | `Chats.tsx:133` |
| Search placeholder | **MATCHES** (web has a key) | `search.placeholder` | `Chats.tsx:62` |
| Select-chat-to-see-messages | **MATCHES** (web has a key) | `messages.select.placeholder` | `Messages.tsx:146` |
| Load-older-messages | **MATCHES** (web has a key) | `messages.load.more` | `Messages.tsx:246` |
| Block notice — you-blocked-them | **MATCHES** | `blocking.label` | `Messages.tsx:295` (notice); `Chats.tsx:109` (badge) |
| Block notice — they-blocked-you | **MATCHES** | `blocked.by.label` | `Messages.tsx:302` (notice); `Chats.tsx:109` (badge) |

All eight have keys in web. The four strings mobile currently hardcodes (`search.placeholder`, `messages.select.placeholder`, `messages.load.more`, and the two block notices) already have web keys mobile can reuse — none need new backend seeds. The spec's `MESSAGES_PAGE.chats.load.more` is just the namespaced form of `chats.load.more`.

### 19. Deleted-user fallback render — **MATCHES**

- `user.deleted` exists in the **COMMON** namespace and is consumed: `Messages.tsx:161,165` (`tCommon('user.deleted')` for avatar + header name) and `Chats.tsx:77,78` (chat-list row name). Triggered by `isDeletedPeer` (`src/messages/utils/deletedUser.ts:7-14`): true when `withUser` is missing (i.e. `getUserData` 404'd / returned undefined) **or** the firebaseUid starts with the `deleted:` sentinel.

### 20. Block badge — **MATCHES** (with a visual nuance)

- Block badge exists in the chat list: `src/messages/components/Chats.tsx:107-111` — `{(blocking || blockedBy) && <Badge variant="secondary" className="text-red-600">{t(blocking ? 'blocking.label' : 'blocked.by.label')}</Badge>}`.
- **Blocked-ness decided by** `useChatBlockStore`'s `isBlocking` / `isBlockedBy`, keyed by `chat.withUserFirebaseUid` (`:81-82`), with a `!peerDeleted` guard.
- **Visual nuance:** it uses the shadcn `Badge variant="secondary"` primitive (the same base `ScheduledForDeletionBadge.tsx:10` uses) plus `text-red-600` — i.e. it's styled like the BANNED badge, not literally the `ScheduledForDeletionBadge` component. Same primitive family, different styling. (`ScheduledForDeletionBadge` is imported and used separately at `Chats.tsx:101` for `PENDING_DELETION`.)

---

## Drift summary (DRIFTED / NOT FOUND only)

- **Claim 4 — DRIFTED.** Forward block doc field is **`blockedUserId`** (`useChatBlockStore.ts:58`), not the spec's `blockerId`. `blockerId` is written only to the reverse index (`userblocksReverse`, `:63`). Canonical forward field web uses = `blockedUserId` (matches mobile).
- **Claim 11 — DRIFTED (partial).** 4 of the 5 catch behaviors are in `sendMessage`'s catch (log, rollback, orphan cleanup, no temp-clear), but `notify.error(messages.send.failed.toast)` is **not** in the store catch — the catch re-throws and the toast fires in the caller `Messages.tsx:315-318`.

No NOT FOUND items. Minor (non-blocking) notes: claim 5's second line moved `:683 → :671`; claim 20's badge is the shadcn `Badge` primitive (red text), not the `ScheduledForDeletionBadge` component.

---

## Cleanup

None needed — read-only audit. No source, test, or git state touched.
