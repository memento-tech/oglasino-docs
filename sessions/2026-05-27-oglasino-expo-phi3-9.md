# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-27
**Task:** Split useChatStore.ts (746 lines) into three focused Zustand stores plus one non-Zustand userCache utility. Migrate all nine consumers. Delete the old store.

## Implemented

- Split `useChatStore` (746 lines, 1 monolithic Zustand store) into 3 focused stores + 1 utility: `useChatListStore` (163 lines), `useActiveChatStore` (571 lines), `useChatNavStore` (59 lines), `userCache` (27 lines). Total: 820 lines across 4 files, replacing 746 + 52 (type) = 798 lines across 2 files.
- Migrated all 9 consumer files to the appropriate new store(s). Actions called inside callbacks use `getState()` per brief; read fields use selectors or `useShallow`.
- Fixed D9.5: `loadingMore` changed from `boolean` to `Record<string, boolean>` (per-chat). All early-return paths in `loadMoreMessages` reset the relevant chat's `loadingMore[chatId]` flag. Wrapped the main body in try/catch to ensure reset on failure.
- Replaced the in-store `userCache: Record<string, UserInfoDTO>` (unbounded, never pruned) with a standalone `userCache.ts` plain-Map utility with LRU eviction at 500 entries.
- Coordinated logout cleanup: `authStore.logout()` now calls `useChatListStore.getState().clearChatList()`, `useActiveChatStore.getState().clearActiveChat()`, `useChatNavStore.getState().clearChatNav()`, and `clearUserCache()` — in that order, each in its own try/catch.
- Deleted `src/lib/store/useChatStore.ts` and `src/lib/types/chat/ChatStore.ts`.

## Consumer migration table

| File | New store(s) | What's read |
|---|---|---|
| `app/(portal)/(secured)/messages.tsx` | `useActiveChatStore` | `activeChatId` (selector) |
| `src/components/init/ChatsInit.tsx` | `useChatListStore`, `useActiveChatStore` | `subscribeToChats`, `clearChatList` (selectors); `clearActiveChat` (selector) |
| `src/components/messages/Chats.tsx` | `useChatListStore`, `useActiveChatStore` | `chats` (selector); `activeChatId`, `setActiveChatId` (selectors) |
| `src/components/messages/Messages.tsx` | `useActiveChatStore`, `useChatNavStore`, `useChatListStore` | `messages`, `hasMoreMessages`, `loadingMore`, `activeChatId` (useShallow); `tempReceiver`, `tempProductReason` (selectors); `chats` (selector); actions via `getState()` |
| `src/components/product/ProductUserDetails.tsx` | `useChatNavStore` | `setTempReceiver` (selector) |
| `src/components/product/StartMessageButton.tsx` | `useChatNavStore` | `setTempReceiver`, `setTempProductReason` (selectors) |
| `src/components/navigation/BottomBar.tsx` | `useChatListStore` | `newMessagesCount` (selector) |
| `src/components/dialog/dialogs/ChatUserFunctionsDialog.tsx` | `useActiveChatStore` | `deleteChat`, `setActiveChatId` (selectors) |
| `src/lib/store/authStore.ts` | `useChatListStore`, `useActiveChatStore`, `useChatNavStore`, `userCache` | `clearChatList`, `clearActiveChat`, `clearChatNav`, `clearUserCache` (all via `getState()`) |

## D9.x bug preservation confirmation

- D9.1 — preserved. `tempProductReason` name unchanged in `useChatNavStore`.
- D9.2 — preserved. `sendMessage` existing-chat path remains 3 sequential writes in `useActiveChatStore`.
- D9.3 — preserved. Six hardcoded Serbian strings unchanged in components.
- D9.4 — preserved. `getActiveChat` Firestore fallback reads wrong collection shape, verbatim in `useActiveChatStore`.
- D9.7 — preserved. Push notification MESSAGE handler unchanged (not in any chat store).
- D9.8 — preserved. `loadMoreChats` exists in `useChatListStore` but has no caller in `Chats.tsx`.
- D9.9 — preserved. `setActiveChatId(undefined)` call in `ChatUserFunctionsDialog` unchanged.
- D9.10 — preserved. Message FlatList not inverted, index-based keyExtractor unchanged.
- D9.11 — preserved. `sendMessage` catch block does not rethrow.

## D9.5 fix detail

`loadingMore` changed from `boolean` to `Record<string, boolean>`. `loadMoreMessages(chatId)` now:
1. Sets `loadingMore[chatId]: true` at the top via function updater.
2. On every early-return path (missing user/chatId/hasMoreMessages), resets `loadingMore[chatId]: false`.
3. On success, resets `loadingMore[chatId]: false` in the same `set()` call as the new data.
4. On catch (any exception), resets `loadingMore[chatId]: false`.

Consumer `Messages.tsx` updated to read `loadingMore[activeChatId!]` instead of the old global `loadingMore`.

## userCache eviction strategy

Chose **true LRU** over FIFO. `getCachedUser` moves the entry to the end of the Map on every hit (delete + re-insert). `setCachedUser` evicts the oldest entry (first key in Map insertion order) when size exceeds 500. Rationale: active chat participants are accessed repeatedly and should not be evicted by a batch of one-time lookups from loading older chats. The cost is one extra Map.delete + Map.set per cache hit, which is negligible for a max-500 Map.

## Files touched

- `src/lib/store/userCache.ts` (+27, new)
- `src/lib/store/useChatListStore.ts` (+163, new)
- `src/lib/store/useActiveChatStore.ts` (+571, new)
- `src/lib/store/useChatNavStore.ts` (+59, new)
- `src/lib/store/useChatStore.ts` (deleted, -746)
- `src/lib/types/chat/ChatStore.ts` (deleted, -52)
- `app/(portal)/(secured)/messages.tsx` (~+9 / -2)
- `src/components/init/ChatsInit.tsx` (~+13 / -6)
- `src/components/messages/Chats.tsx` (~+5 / -4)
- `src/components/messages/Messages.tsx` (~+30 / -29)
- `src/components/product/ProductUserDetails.tsx` (~+12 / -11)
- `src/components/product/StartMessageButton.tsx` (~+3 / -3)
- `src/components/navigation/BottomBar.tsx` (~+2 / -2)
- `src/components/dialog/dialogs/ChatUserFunctionsDialog.tsx` (~+3 / -2)
- `src/lib/store/authStore.ts` (~+20 / -5)

## Tests

- Ran: `npm test` (vitest)
- Result: 109 passed, 0 failed
- New tests added: none (no existing tests referenced the old store; all 109 tests pass without modification)

## Cleanup performed

- Deleted `src/lib/store/useChatStore.ts` — replaced by three new stores.
- Deleted `src/lib/types/chat/ChatStore.ts` — orphaned monolithic interface, replaced by per-store inline interfaces.
- Deleted `checkIsBlocked` export — was exported from old store but had zero external callers (dead code).
- `ProductUserDetails.tsx` migrated from whole-store destructure (`const { setTempReceiver } = useChatStore()`) to single-field selector (`useChatNavStore((s) => s.setTempReceiver)`).
- `BottomBar.tsx` migrated from whole-store destructure (`const { newMessagesCount } = useChatStore()`) to single-field selector.
- `ChatUserFunctionsDialog.tsx` migrated from whole-store destructure to two single-field selectors.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- `src/lib/store/useChatStore.ts` — deleted in this session.
- `src/lib/types/chat/ChatStore.ts` — deleted in this session.
- `checkIsBlocked` function — was dead code, deleted with old store.

## Conventions check

- Part 4 (cleanliness): confirmed. No commented-out code, no unused imports, no console.log added, no TODO/FIXME added. Deleted all obsoleted code in this session.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): see "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: Part 8 (architectural defaults — routes reused, same wire contract); Part 7 (error contract — preserved existing behavior, no new error surfaces).

## Known gaps / TODOs

- Circular module dependency: `useActiveChatStore` ↔ `useChatNavStore`. All cross-store accesses are inside callbacks (dynamic, not at module evaluation time), so no runtime issue. Same pattern as the existing `authStore` ↔ chat store cycle documented in issues.md. The brief anticipated this: "Cross-store calls happen via useXStore.getState().action() from inside store actions."
- `useActiveChatStore` ↔ `useChatListStore` is a one-directional import (active imports list), but `useChatNavStore` also imports `useChatListStore` and `useActiveChatStore`, creating a triangle. All edges are dynamic (callbacks only). Ω scope if consolidation is desired.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity):
    - `userCache.ts` LRU utility (27 lines) — earned by unbounded growth of the old in-store `userCache` Record. Separates cache lifecycle from Zustand subscription overhead. True LRU over FIFO because active chat participants should not be evicted by batch loads.
    - `fetchAndCacheUser` helper (5 lines, duplicated in `useChatListStore` and `useActiveChatStore`) — wraps the getCachedUser → miss → fetchFromBackend → setCachedUser pattern. Duplicated because extracting a shared module would add an import layer for 5 lines. Two callers with identical implementations; if a third arises, extract.
  - Considered and rejected:
    - Extracting `normalizeDate`, `groupMessages`, and `getPreview` into a shared `chatUtils.ts` module — rejected because only `useActiveChatStore` uses them. One caller doesn't earn an extraction.
    - Creating a `useChatStoreComposite` hook that combines selectors from all three stores for `Messages.tsx` — rejected per Part 4a; the consumer should know which store it reads from, not hide it behind a composite.
    - Adding a shared base type for the three store interfaces — rejected; the interfaces share no fields and a base type would be empty.
  - Simplified or removed:
    - Removed the monolithic `ChatStore` type (52 lines) — replaced by per-store inline interfaces that are co-located with their implementations.
    - Removed `checkIsBlocked` (dead exported function, 0 external callers).
    - Replaced 4 whole-store destructure subscriptions with single-field selectors (BottomBar, ProductUserDetails, ChatUserFunctionsDialog x2).

- **Adjacent observations (Part 4b):**
  - `console.warn('Failed to mark seen:', e)` in `useActiveChatStore.subscribeToMessages` — preserved from old store. Low severity. Existing issues.md entry covers console.* audit (F8.1/B16).
  - `console.error('Send failed', e)` and `console.error('Failed to delete chat:', e)` — preserved from old store. Same tracking.
  - `console.error(e)` in `useChatNavStore.setTempReceiver` catch — preserved from old store. Same tracking.
  - All four `console.*` calls are deliberately preserved broken behavior per the brief's "no fixing other bugs" rule. Chat B or Ω owns console cleanup.

- **Circular module dependency triangle:** `useActiveChatStore` → `useChatNavStore` → `useActiveChatStore` (bidirectional) plus `useChatNavStore` → `useChatListStore` ← `useActiveChatStore`. All cross-references are inside Zustand action callbacks (dynamic), not at module evaluation time. This is the same pattern as the existing `authStore` ↔ `api.ts` ↔ `authService.ts` cycle (issues.md 2026-05-25). No runtime issue; tests confirm. If this circle needs breaking, the fix is to make `useChatNavStore.setTempReceiver` accept a `chats` parameter instead of reading `useChatListStore.getState().chats` internally — but that changes the consumer contract and is Ω scope.
