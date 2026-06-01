# Audit — Messaging adoption (Phase 2 re-audit)

**Repo:** oglasino-expo
**Branch:** `new-expo-dev`
**Date:** 2026-05-31
**Mode:** READ-ONLY — no code changes, no git state changes, no installs.
**Slug:** `messaging-adoption` (n=1 — no prior `*-messaging-adoption-*.md` in `.agent/`)
**Going-in audit re-measured:** `.agent/audit-expo-readiness-messaging.md` (2026-05-23, branch `dev`)
**Contract measured against:** `oglasino-docs/features/messaging.md` + the three brief corrections.

> **Method note.** Line numbers in this report are taken from `grep -n` / direct reads on the current `new-expo-dev` tree. The repo's structural foundation work (Φ3 Brief F23 chat-store split, the boot redesign) reshaped the surface substantially since 2026-05-23. **The brief's own file map is itself stale** — most files it names (`useChatStore.ts`, `blockStore.ts`, `chatTempStore.ts`, `ChatList.tsx`, `ChatListItem.tsx`, `ChatMessages.tsx`, `MessageBubble.tsx`, `SingleChatView.tsx`, `BlockBanner.tsx`, `notificationTypes.ts`, `notificationService.ts`) **do not exist**. Real current paths are reported below.

---

## Section A — Module map

### A1 — Files on the messaging surface and their roles

| Role | Current real path | Lines | Notes |
|---|---|---|---|
| Chat-list store | `src/lib/store/useChatListStore.ts` | 164 | Replaces the deleted monolith. `src/lib/store/useChatStore.ts` is **staged-for-deletion** (`git status: D`) and gone from the working tree. |
| Active-chat / messages store | `src/lib/store/useActiveChatStore.ts` | 580 | Untracked (`??`) — new since `dev`. |
| Block store | `src/lib/store/useChatBlockStore.ts` | 101 | |
| Temp/nav store | `src/lib/store/useChatNavStore.ts` | 59 | Holds `tempReceiver` + `tempProductReason`. Untracked. |
| Chat-list component | `src/components/messages/Chats.tsx` | 104 | `FlatList` (`:86`), local display-name search filter. |
| Chat-detail component | `src/components/messages/Messages.tsx` | 255 | `FlatList` (`:210`), block notices, load-older header. |
| Message renderer | `src/components/messages/Message.tsx` | 48 | Text + image blocks. |
| Message input | `src/components/messages/MessageInput.tsx` | 217 | Text + ≤5 images, upload progress, cancel. |
| Image display | `src/components/messages/MessageImages.tsx` | 77 | view-token resolution. |
| Page container | `app/(portal)/(secured)/messages.tsx` | 25 | Switches `<Chats/>` ↔ `<Messages/>` on `activeChatId`. |
| Init/watcher | `src/components/init/ChatsInit.tsx` | — | Wires subscriptions on `user`. |
| Block/delete/report dialog | `src/components/dialog/dialogs/ChatUserFunctionsDialog.tsx` | — | Hosts `deleteChat` (`:27`,`:46`), block/unblock, report. |
| Notification client | `src/lib/client/firebaseNotifications.ts` | — | Holds the `NotificationType` enum (`:16`). |
| Notifications screen | `app/(portal)/(secured)/notifications.tsx` | — | Renders the notifications list. |
| Push init | `src/notifications/components/PushNotificationsInit.tsx`, `NotificationsInit.tsx` | — | Push category routing. |
| User enrichment | `src/lib/services/userService.ts` | — | `getUserForFirebaseUid` → backend HTTP (`:71`). |

### A2 — The chat-store split

The going-in audit knew a single monolithic `useChatStore.ts`. Current `new-expo-dev` has **four** stores (`useChatStore.ts` deleted):

| Store | State | Functions |
|---|---|---|
| `useChatListStore.ts` | `chats[]` (enriched), `hasMoreChats`, `lastVisibleChat` cursor, `loadingMoreChats`, `newMessagesCount` | `subscribeToChats` (onSnapshot `userchats/{uid}/chats`), `loadMoreChats` (`:97`), `reset` |
| `useActiveChatStore.ts` | `activeChatId`, `messages{}`, `hasMoreMessages{}`, `lastVisibleMessage{}`, `loadingMore{}`, `unsubMessages{}` | `subscribeToMessages` (`:114`), `loadMoreMessages` (`:193`), `sendMessage` (`:258`), `setActiveChatId` (`:476`), `getActiveChat` (`:478`), `deleteChat` (`:522`), `unsubscribeFromMessages` (`:558`), `clearActiveChat` (`:568`). Mark-seen is **inline** inside `subscribeToMessages` (`:169–187`), not a named function. |
| `useChatBlockStore.ts` | `blocking{}`, `blockedBy{}` | `subscribe` (forward `userblocks/{uid}/blocked` + reverse `userblocksReverse/{uid}/blockedBy` onSnapshot), `blockUser` (`:57–65` batch), `unblock` |
| `useChatNavStore.ts` | `tempReceiver`, `tempProductReason` | `setTempReceiver`, `setTempProductReason`, `clearChatNav` |

### A3 — Cycle B (the require cycle)

`issues.md` (2026-05-28) recorded a "chat-store require-cycle **triangle** (Cycle B)." On current `new-expo-dev` it is a **2-node dyad**, not a triangle:

- `useActiveChatStore.ts` → `import { useChatListStore }` (`:32`), `import { useChatNavStore }` (`:33`).
- `useChatNavStore.ts` → `import { useActiveChatStore }` (`:7`), `import { useChatListStore }` (`:8`).
- `useChatListStore.ts` imports **none** of the other chat stores (only `authStore`, `userCache`, `userService`).
- `useChatBlockStore.ts` imports **none** of the other chat stores.

**The closing edge is `useActiveChatStore ↔ useChatNavStore`.** Both directions cross only through dynamic `getState()` inside function bodies (not module-evaluation-time calls), so it is benign at runtime:
- `useActiveChatStore` → `useChatNavStore.getState()` at `:262`, `:450`, `:509`.
- `useChatNavStore` → `useActiveChatStore.getState().clearActiveChat()` and `useChatListStore.getState().reset()` inside `clearChatNav`, and `useActiveChatStore.getState()` inside `setTempReceiver`.

No load-order failure today; structural untidiness only. **Disagreement with going-in:** it was a triangle (monolith era); it is now a dyad, and `useChatBlockStore`/`useChatListStore` are NOT part of it.

---

## Section B — Spec-adoption items (vs verified contract)

### B-1 — Atomic existing-chat `writeBatch` — **DIVERGES**
`sendMessage` (`useActiveChatStore.ts:258`) branches on new vs existing chat.
- **New-chat path is a 4-op atomic `writeBatch`** (matches the going-in claim and the verified shape): `writeBatch(db)` (`:343`); `batch.set` chat-root `chats/{id}` (`:345`), sender sidecar `userchats/{me}` (`:352`), receiver sidecar `userchats/{receiver}` with `unreadCount: 1` (`:360`), message doc (`:383`); `await batch.commit()` (`:385`). One difference from the web 4-op send: the new-chat batch sets `unreadCount: 1` literally and the message `seen:false` — it both **creates** the chat and **sends** the first message in one batch.
- **Existing-chat path is NOT atomic** — sequential awaited writes: `addDoc(messages, …)` (`:399`), then sender sidecar `getDoc` (`:404`) + `setDoc` (new `:407` / merge `:415`), then receiver sidecar `setDoc … { unreadCount: increment(1), merge:true }` (`:425–434`). DIVERGES from the verified single `writeBatch`, 4-op shape. **Note:** the existing-chat path does **not** update the `chats/{id}` root doc's `lastMessage`/`lastUpdated` at all — only the two `userchats` sidecars — whereas the verified web batch updates the chat-root too (op 1 of 4).

### B-2 — Send-failure toast — **DIVERGES (per correction #2)**
The correct pattern is "store throws, screen catches and toasts."
- `useActiveChatStore.sendMessage` does **not** throw on failure — it `catch`es internally, logs `console.error('Send failed', e)` (`:454`), fires orphan-image cleanup (`:459–462`), and rolls back the optimistic message. No re-throw.
- The screen caller `Messages.tsx:247` (`useActiveChatStore.getState().sendMessage(...)`) has **no** surrounding try/catch and no `notify`/toast.
- So a send failure is **silently swallowed** — no user-visible toast. (A toast primitive exists and is used for *image-upload* failures in `MessageInput.tsx:84,:132` via `react-native-toast-notifications`; the `notify` util also exists. The seam for "screen catches and toasts" would sit at `Messages.tsx:247` once `sendMessage` is changed to re-throw.)

### B-3 — Deleted-user fallback rendering — **ABSENT**
No `'Deleted user'` string, no `COMMON.user.deleted` lookup, and **no `deleted:`-prefix detection** (`startsWith('deleted:')`) anywhere on the surface. `Chats.tsx`'s row renders `chatSummary.withUser.displayName` (`:34`) and `withUser.profileImageKey` (`:29`) with **no null/deleted guard** — it will throw if `withUser` is `undefined` (e.g. a 404/deleted counterparty). Chat detail has no deleted handling either.

### B-4 — `tempProductReason` → `tempProductContext` rename + clearing — **rename NOT done; productId NOT dropped via the web mechanism**
- (a) **Field name:** mobile still uses **`tempProductReason`** (`useChatNavStore.ts:14`), not `tempProductContext`. The rename has NOT been adopted.
- (b) **The mount-clear / productId question:** there is **no** `MessageInput` mount effect that clears temp product context — `clearTempProductContext` does not exist; `MessageInput`'s only effect reacts to `[disabled, suggestion]` (`MessageInput.tsx:53–60`). Temp state is cleared once, on send **success**, via `useChatNavStore.getState().clearChatNav()` (`useActiveChatStore.ts:450`). The send path **captures `tempProductReason` before any clear**: `sendMessage` reads it at `:264` and stamps `messageRequest.productId = tempProductReason.id` in both the new-chat (`:379–381`) and existing-chat (`:395–397`) branches, before `clearChatNav()` runs at `:450`. **Mobile does NOT have the web productId-drop bug** — productId is attached to the message doc itself (not the chat root) and is captured pre-clear. The risk is only the cosmetic rename gap. (See Section E-1.)

### B-5 — Mark-seen `seenLocal` mutation — **DIVERGES**
Inline in `subscribeToMessages`: `unseen.forEach((m) => seenLocal.add(m.id))` runs at `:174`, **before** `batch.commit()` at `:186`. Verified correct shape is "mutate after commit (`.then`), clean on `.catch`." Mobile mutates **before** commit; the `.catch` (`:186`) only `console.warn`s — it does **not** remove the ids from `seenLocal` on failure. So a failed commit leaves those messages permanently un-retried for the session. DIVERGES.

### B-6 — Auto-linkify message URLs — **ABSENT**
`Message.tsx:38–40` renders message text as plain `<Text>{block.text}</Text>`. No `Linking`, no `Hyperlink`, no linkify library anywhere in the messages components. URLs are inert text.

### B-7 — Block badge on the chat list — **ABSENT**
`Chats.tsx` (`ChatListRow`) renders no block indicator. There is no `showBlockBadge`, no `!peerDeleted` guard, no `t('blocking.label')`/`t('blocked.by.label')` use on the list. Block UI exists **only inside the open conversation** (`Messages.tsx:224–239`) as inline hardcoded Serbian notices (see B-C3). DIVERGES from the verified web behavior (badge on list, guarded by `!peerDeleted`).

### B-8 — "Load more" affordance on the chat list — **ABSENT (UI)**
`useChatListStore.loadMoreChats` exists (`:97`) but the `Chats.tsx` `FlatList` (`:86`) has **no `onEndReached`** and no button — nothing in the UI calls `loadMoreChats`. Users past the first page cannot reach older chats. (Contrast: the message-list *load-older* affordance DOES exist — `Messages.tsx:152–168` header button calling `loadMoreMessages` — but that is the in-chat list, not the chat list.) The key to consume is `MESSAGES_PAGE.chats.load.more`.

### B-9 — `deleteChat` reorder — **DIVERGES (and incomplete)**
`deleteChat` (`useActiveChatStore.ts:522`) mutates **local store first** (`set(...)` clearing messages/cursors/unsub at `:530–550`) and **then** awaits the Firestore delete (`:552`). Verified correct shape is "await Firestore delete first, then mutate local; toast on failure." Mobile is local-first, and on failure only `console.error`s (`:554`) — no toast and the local removal is not reverted. Additionally, the delete only removes the caller's own sidecar (`userchats/{me}/chats/{id}`, `:552`) — there is no toast/rollback and the chat reappears on the next list-listener fire if the delete throws.

### B-10 — Forward-block field-name alignment — **MATCHES (no-op, per correction #1)**
`useChatBlockStore.blockUser` writes the **forward** doc `userblocks/{me}/blocked/{target}` with `{ blockedUserId: <target>, createdAt }` (`:57–60`) — the canonical field. The **reverse** index `userblocksReverse/{target}/blockedBy/{me}` carries `{ blockerId: <me>, createdAt }` (`:62–65`). Mobile is already correct; this item closes as a no-op.

---

## Section C — Carried bugs

### C-1 — B10 `NotificationType.NORMAL = 'NOTMAL'` typo — **PRESENT, harmless (not branched on)**
`src/lib/client/firebaseNotifications.ts:17` — `NORMAL = 'NOTMAL',` inside `enum NotificationType` (`:16–21`). (Going-in cited `:17` on the old file path `src/lib/client/firebaseNotifications.ts` — same path, same line; confirmed.)
- The enum `NotificationType` is **imported nowhere** (only self-referenced at `firebaseNotifications.ts:40`, `type: NotificationType`). The notifications screen discriminates `item.type` against `'SUCCESS'`/`'DANGER'`/`'WARNING'` string literals for border color (`notifications.tsx:53,55,57`) and an `else` (`:59`) — `'NOTMAL'`/`'NORMAL'` is never compared, so the typo is inert.
- There is **no separate `NotificationKind` enum** (the brief's going-in note and my own earlier draft were wrong about this). Push navigation is driven by `categoryId` (`PushNotificationsInit.tsx` `case 'MESSAGE'` `:76`, `'SAVED_PRODUCT'` `:80`, `'NAVIGATION'` `:91`, …), not by the `type` discriminator. **Verdict: stored-but-never-branched-on, confirmed.**

### C-2 — B11 `getActiveChat` fallback reads wrong fields — **PRESENT (in `useActiveChatStore.getActiveChat`)**
`getActiveChat` (`useActiveChatStore.ts:478`) — when the chat is not in the local list, it `getDoc`s the chat **root** doc `chats/{id}` (`:488`) and reads `data.withUserFirebaseUid` (`:495`) and `data.withUser` (`:496`). **Neither field exists on the chat root doc** (the root holds `users[]`, `createdAt`, `lastMessage`, `lastUpdated`; `withUserFirebaseUid`/`withUser` live only on the `userchats` sidecar). So an out-of-cache chat resolves with `withUserFirebaseUid: undefined`, `withUser: undefined`. The verified correct shape is `otherUid = data.users.find(u => u !== currentUid)` then `getUserForFirebaseUid(otherUid)`.
- **Disagreement with going-in:** it placed this at `useChatStore.ts:711-712/:718-719` in the monolith. Post-split, the correct-pattern derivation now lives in `useChatListStore` (counterparty from sidecar `withUserFirebaseUid` `:77`, enriched via `getUserForFirebaseUid` `:78`), but the **buggy root-doc read survives in `useActiveChatStore.getActiveChat:495–496`**. The bug moved, it did not get fixed.

### C-3 — B12 five hardcoded Serbian strings — **PARTIAL (3 of 5 still hardcoded; 2 reshaped)**

| Going-in string | Status on `new-expo-dev` | Current file:line | Key to consume |
|---|---|---|---|
| Search placeholder | **STILL hardcoded** `"Pretraži ćaskanja..."` | `Chats.tsx:81` | `MESSAGES_PAGE.search.placeholder` |
| Select-chat placeholder | **STILL hardcoded** `"Izaberite ćaskanje kako bi videli poruke..."` | `Messages.tsx:138` | `MESSAGES_PAGE.messages.select.placeholder` |
| Load-older affordance | **STILL hardcoded** `"Učitaj starije poruke"` | `Messages.tsx:163` | `MESSAGES_PAGE.messages.load.more` |
| You-blocked-them | hardcoded Serbian inline notice | `Messages.tsx:224–239` (block notice block) | `MESSAGES_PAGE.blocking.label` |
| They-blocked-you | hardcoded Serbian inline notice | `Messages.tsx:224–239` (block notice block) | `MESSAGES_PAGE.blocked.by.label` |

(The going-in's sixth string, the image-preview fallback `"Nove poruke ..."`, also still exists — `useActiveChatStore.ts:85`, in `getPreview`. Not one of the brief's five; noted for completeness.) **No new keys proposed** (per correction #3). The two block-notice strings are rendered inline in `Messages.tsx`, not on the chat-list row.

### C-4 — B16 ad-hoc `console.*` on the messaging surface — inventory
| File:line | Call | Note |
|---|---|---|
| `useActiveChatStore.ts:186` | `console.warn('Failed to mark seen:', e)` | mark-seen `.catch` |
| `useActiveChatStore.ts:454` | `console.error('Send failed', e)` | send `catch` |
| `useActiveChatStore.ts:554` | `console.error('Failed to delete chat:', e)` | deleteChat `catch` |
| `useChatNavStore.ts:49` | `console.error(e)` | nav catch |
| `MessageInput.tsx:90` | `if (__DEV__) console.warn(...)` | dev-gated, HEIC convert |
| `authService.ts:47,115,186,188` | `console.error/warn/info` | auth surface (adjacent) |

The repo convention is `logServiceError` / `logServiceWarn` (used in `authService.ts:85,102,192,237`). The Φ4 service-layer work would have replaced the bare `console.*` in the store catches with the project loggers. The four store/nav `console.*` (`useActiveChatStore.ts:186,454,554`, `useChatNavStore.ts:49`) are the in-scope candidates.

### C-5 — B18 `var` → `const`/`let` — **ABSENT (no `var` on the surface)**
No `var` declarations on the messaging/notifications/auth surface. The going-in's `useChatStore.ts:454 var previewMessage` is gone — the monolith was deleted; the equivalent now reads `let previewMessage = getPreview(...)` (`useActiveChatStore.ts:340`). **Disagreement with going-in:** the `var` is resolved/obsolete.

---

## Section D — Trust-boundary re-confirm

### D-1 — Grep of `users/{...}` reads — **HOLDS**
The **only** Firestore access to the `users` collection anywhere in `src/`+`app/` is `authService.ts:56` `doc(db, 'users', firebaseUser.uid)` — the authenticated user's **own** doc, used by `ensureUserInFirestore` (getDoc `:59`, setDoc `:99`). No `collection(db, 'users')` reads. **No cross-user `users/{otherUid}` reads on the messaging surface.** (Going-in cited the own-doc read at `authService.ts:59`; current own-doc ref is `:56`/`:59` — same intent, confirmed.)

### D-2 — Participant enrichment goes through the backend — **HOLDS**
Chat-participant display name/avatar resolves via `getUserForFirebaseUid` → `BACKEND_API.get('/auth/firebase/' + firebaseUid)` (`userService.ts:71`), consumed by `fetchAndCacheUser` in `useActiveChatStore.ts:38–44` and `useChatListStore.ts:78`. No direct `users/{otherUid}` Firestore read for enrichment.

### D-3 — No illegitimate cross-user writes — **HOLDS**
The only cross-user writes are the two known-authorized patterns: the **receiver sidecar** (`userchats/{receiver}/chats/{id}`, `useActiveChatStore.ts:360` new-chat / `:425` existing-chat) and the **reverse block index** (`userblocksReverse/{target}/blockedBy/{me}`, `useChatBlockStore.ts:62`). No other write targets another user's documents.

### D-4 — Correct `receiverId` always written — **HOLDS (with the documented residual gap)**
`sendMessage` always sets `receiverId: receiver.firebaseUid` where `receiver` is derived from the resolved chat counterparty (`:283–295`, message requests `:374/:390`). `senderId` is always `user.firebaseUid` (auth uid). The `receiverId` value is client-supplied and not verified by the rule (the documented residual gap, spec-level, identical to web) — mobile does **not** regress it (it computes the counterparty from the deterministic chatId / sidecar, not from arbitrary input).

---

## Section E — Open questions (answered)

### E-1 — Does mobile have the real productId bug, or only the rename gap?
**Only the rename gap.** `MessageInput` does **not** mount-clear temp context (no such effect; `clearTempProductContext` doesn't exist). `tempProductReason` is captured in `sendMessage` (`:264`) and written onto the **message doc** `productId` (`:379–381` new, `:395–397` existing) **before** `clearChatNav()` runs (`:450`, success-only). Because mobile attaches productId to the message (not to a chat-creation step that races a mount-clear), the web productId-drop mechanism does not exist here. The only gap is the field is still named `tempProductReason` (not `tempProductContext`).

### E-2 — Post-split chat-store module graph + Cycle B's real shape
Four stores (A2). Cycle B is a **2-node dyad** `useActiveChatStore ↔ useChatNavStore` (A3), closed only through dynamic `getState()` calls — benign at runtime. `useChatListStore` and `useChatBlockStore` are leaves (import no other chat store). Not the going-in "triangle."

### E-3 — Is there an RN toast/`notify` primitive for the send-failure path?
**Yes — one exists and is already in use.** `react-native-toast-notifications` (`useToast()`) is used in `MessageInput.tsx:38,84,132` for image-upload errors; a `notify` util (`src/lib/utils/notify.ts`) is used across the app (e.g. `authService.ts:190`). The send-failure path simply doesn't use either — `sendMessage` swallows the error (C-2 / B-2). No new primitive needs introducing; the seam is to re-throw from `sendMessage` and catch+toast at `Messages.tsx:247`.

### E-4 — Does forward-block field alignment make correction #1 a no-op?
**Yes.** Forward doc writes `blockedUserId` (`useChatBlockStore.ts:57–60`); mobile is already canonical (B-10). No-op for mobile.

---

## Divergence summary (flat list of every DIVERGES / ABSENT / PRESENT-bug)

1. **B-1 DIVERGES** — existing-chat send is sequential (`addDoc` + sidecar `setDoc`s), not a 4-op `writeBatch`; chat-root doc not updated on existing-chat send. (`useActiveChatStore.ts:399,407,415,425`)
2. **B-2 DIVERGES** — no send-failure toast; `sendMessage` swallows + rolls back, screen doesn't catch. (`useActiveChatStore.ts:454`; `Messages.tsx:247`)
3. **B-3 ABSENT** — no deleted-user fallback, no `deleted:` detection; `Chats.tsx:34` will crash on undefined `withUser`.
4. **B-4 (rename) DIVERGES** — field still `tempProductReason`, not `tempProductContext`. (`useChatNavStore.ts:14`) — but no productId-drop bug (E-1).
5. **B-5 DIVERGES** — `seenLocal` mutated before `commit`, no cleanup on failure. (`useActiveChatStore.ts:174` vs `:186`)
6. **B-6 ABSENT** — no URL linkify; plain text. (`Message.tsx:38–40`)
7. **B-7 ABSENT** — no block badge on the chat list. (`Chats.tsx`)
8. **B-8 ABSENT** — `loadMoreChats` exists but no UI calls it. (`useChatListStore.ts:97`; `Chats.tsx:86`)
9. **B-9 DIVERGES** — `deleteChat` is local-first then Firestore; no toast/rollback on failure; deletes only own sidecar. (`useActiveChatStore.ts:530–552`)
10. **C-1 PRESENT (harmless)** — `NORMAL = 'NOTMAL'` typo, never branched on. (`firebaseNotifications.ts:17`)
11. **C-2 PRESENT (bug)** — `getActiveChat` reads non-existent `data.withUserFirebaseUid`/`data.withUser` off the chat root. (`useActiveChatStore.ts:495–496`)
12. **C-3 PARTIAL** — 3 hardcoded Serbian strings remain (`Chats.tsx:81`, `Messages.tsx:138`, `Messages.tsx:163`); 2 block-notice strings hardcoded inline (`Messages.tsx:224–239`); plus `getPreview` fallback `useActiveChatStore.ts:85`.
13. **C-4** — bare `console.*` in store catches: `useActiveChatStore.ts:186,454,554`, `useChatNavStore.ts:49`.

**Trust boundary (Section D): clean.** No release-blocking cross-user reads; enrichment via backend; cross-user writes limited to the two authorized patterns; `receiverId`/`senderId` correct.

**Matches / no-ops:** B-10 (forward field `blockedUserId`), new-chat 4-op atomic batch (B-1 first half), B-4 productId capture (E-1), C-5 (`var` gone).

---

## Disagreements with the going-in audit (2026-05-23, `dev`)

1. **Store split.** Going-in knew one `useChatStore.ts` (~760 lines). It is **deleted**; the surface is four stores (A2). Every `useChatStore.ts:<n>` cite in the going-in is dead.
2. **Cycle B is a dyad, not a triangle** (A3). Only `useActiveChatStore ↔ useChatNavStore`; block/list stores are leaves.
3. **`var previewMessage` is gone** (going-in `useChatStore.ts:454`). Now `let previewMessage` (`useActiveChatStore.ts:340`). C-5 resolved.
4. **B11 bug moved, not fixed** — the wrong-field root-doc read is now at `useActiveChatStore.getActiveChat:495–496` (going-in placed it at `useChatStore.ts:711-712/:718-719`). The chat-list enrichment path was rebuilt correctly in `useChatListStore`, but `getActiveChat`'s fallback still reads `data.withUser`/`data.withUserFirebaseUid` off the chat root.
5. **`tempProductReason`** still un-renamed (going-in flagged it; still true) — but the going-in never checked the mount-clear; this audit confirms there is **no** mount-clear and **no** productId-drop bug (E-1).
6. **No `NotificationKind` enum** exists (contrary to an assumption that a second correctly-spelled enum carries navigation). Navigation keys off `categoryId`, and the `NotificationType` enum (with the typo) is imported nowhere.
7. **Going-in file paths for notifications** (`src/notifications/lib/firebaseNotifications.ts`) are wrong; the file is `src/lib/client/firebaseNotifications.ts`.
8. **Brief's file map is stale too** — `useChatStore.ts`, `blockStore.ts`, `chatTempStore.ts`, `ChatList.tsx`, `ChatListItem.tsx`, `ChatMessages.tsx`, `MessageBubble.tsx`, `SingleChatView.tsx`, `BlockBanner.tsx`, `notificationTypes.ts`, `notificationService.ts` do not exist; real paths are in Section A.

---

## Cleanup performed

none needed — read-only.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change — findings here feed the Phase 3/4 messaging-adoption seam analysis; they are deliberately not filed as standalone `issues.md` entries (Phase 2 audit output).

## Obsoleted by this session

nothing — read-only audit. (It does, however, supersede the 2026-05-23 going-in audit's `file:line` map for `new-expo-dev`; see Disagreements.)

## Conventions check

- Part 4 (cleanliness): N/A — read-only, no code touched.
- Part 4a (simplicity) / Part 4b (adjacent observations): adjacent observations recorded in Divergence summary + C-3/C-4 (hardcoded strings, bare `console.*`, `Chats.tsx:34` unguarded `withUser`).
- Part 6 (translations): three+ hardcoded Serbian strings flagged with their existing keys (C-3); no new keys proposed (correction #3).
- Part 7 (error contract): N/A to messaging (Firestore-direct surface; the send path uses no backend error envelope).
- Part 11 (trust boundaries): explicit re-confirm in Section D — clean.

## For Mastermind

- **Part 4a simplicity evidence:** Added — nothing (read-only). Considered and rejected — nothing. Simplified or removed — nothing.
- Highest-leverage findings for the reshape: C-2 (`getActiveChat` root-doc read, real bug for out-of-cache chats), B-1 (existing-chat non-atomic + chat-root not updated on existing-chat send), B-3 (`Chats.tsx:34` will crash on a deleted/404 counterparty — no `withUser` guard).
- Tooling caveat: the local shell/Read tooling was intermittently dropping/corrupting output during this session; all cited `file:line` were re-confirmed via single-file `grep -n` (and a clean-context sub-agent cross-read) to avoid the corruption. If any single cite looks off, re-grep the named symbol.
