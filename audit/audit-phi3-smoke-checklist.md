# Φ3 Smoke Verification Checklist

**Date:** 2026-05-27
**Spec:** `features/expo-performance-foundation.md` §7
**Device:** real device (mid-range Android + iPhone)
**Prerequisite:** development build with all Φ3 changes baked in

---

## Product feed

- [ ] **Scroll home feed 30 seconds.** Open the app, land on the home product feed. Scroll continuously for 30 seconds. **Pass:** smooth scrolling, no visible hitches or dropped frames. **Fail:** visible stutters, frame drops, or UI freezing during scroll.

- [ ] **Scroll-back image cache.** Scroll down ~20 products, then scroll back to the top. **Pass:** previously viewed images appear instantly from disk cache — no loading spinners, no re-download flicker. **Fail:** images re-download or show loading placeholders on scroll-back.

- [ ] **Card size toggle isolation.** Tap the card-size toggle (grid/list). **Pass:** only the visible product cards update their layout; the bottom bar, top bar, and other non-card UI elements do not flicker or re-render visibly. **Fail:** bottom bar badges flicker, top bar re-renders, or other unrelated UI elements visibly update.

- [ ] **Category filter smooth render.** Select a category filter from the category navigation. **Pass:** new filtered results render without stutter or visible delay; the transition from old results to new is smooth. **Fail:** visible blank frame, stutter, or dropped frames during filter application.

---

## Messaging

- [ ] **Chat list opens cleanly.** Navigate to the Messages tab. **Pass:** chat list loads and displays; each row shows the correct partner name, last message preview, and unread badge. **Fail:** blank screen, missing data, or crash.

- [ ] **Open a chat and scroll.** Tap a chat to open it. Scroll through message history. **Pass:** messages display correctly, scrolling is smooth. **Fail:** messages missing, wrong order, scroll lag, or crash.

- [ ] **Send a message.** Type and send a message in an open chat. **Pass:** message appears in the chat immediately with correct content. **Fail:** message doesn't appear, appears with wrong content, or error toast.

- [ ] **Receive a message.** Have another user (or use a second device) send a message to the active chat. **Pass:** incoming message appears in real time without manual refresh. **Fail:** message doesn't appear until app restart or manual refresh.

- [ ] **D9.x bugs still present.** Verify these known bugs are NOT accidentally fixed (chat B owns them):
  - D9.1: `tempProductReason` name not renamed to `tempProductContext` — the product reason field in new-chat flow should still work but the internal name is the old one.
  - D9.3: Hardcoded Serbian strings in chat UI are still Serbian regardless of selected language.
  - D9.4: Opening a chat via deep link or direct ID lookup may show incomplete partner data (the Firestore fallback reads from the wrong document shape).
  - D9.9: `setActiveChatId(undefined)` may not clear properly in some navigation paths.
  - D9.10: Message list is NOT inverted — oldest messages at top, newest at bottom (scroll to bottom to see latest).
  - **Pass:** all these quirks are still present (they haven't been accidentally fixed). **Fail:** any of these behaviors has changed compared to pre-Φ3.

- [ ] **D9.5 fix: pagination spinner.** Open a chat with enough history to paginate. Scroll up to trigger pagination. **Pass:** loading spinner appears while paginating, disappears when new messages load, and disappears immediately if there are no more messages (no stuck spinner). **Fail:** spinner stays visible after pagination completes, or spinner remains permanently ("stuck true" regression).

- [ ] **Logout clears chat data.** Log out. Then log in as a different user (or check before logging back in). **Pass:** after logout, the Messages tab badge shows 0, the chat list is empty, and no previous user's chat data is visible. **Fail:** previous user's chat list, messages, or badge count persists after logout.

---

## City and category selection

- [ ] **City selector scroll performance.** Navigate to product creation (or wherever the city selector opens). Open the city selector modal. Scroll through the full city list. **Pass:** scrolling is smooth with no JS thread freeze; the list renders via SectionList with region headers. **Fail:** JS thread freezes (UI becomes unresponsive), visible lag, or noticeable delay when scrolling.

- [ ] **Category selector expand/collapse.** Open the category selector modal. Expand several top-level categories. **Pass:** expand/collapse animation is smooth, subcategories appear without freezing the JS thread. **Fail:** JS thread freeze, visible lag, or delayed rendering of subcategories.

- [ ] **Critical visual identity — city selector.** Compare the city selector to the pre-Φ3 app (screenshots or memory). **Pass:** region headers are in the same position, item padding is ~identical, search behavior filters correctly, the modal animation (`animationType="slide"`) feels the same. **Fail:** layout is noticeably different — headers in wrong position, padding significantly changed, search broken, or animation different.

- [ ] **Critical visual identity — category selector.** Compare the category selector to the pre-Φ3 app. **Pass:** top-level category headers in same position, subcategory indentation ~identical, 3-level expand/collapse works, search behavior unchanged. **Fail:** visible layout regression — different spacing, different order, broken expand/collapse, or search regression.

---

## AppContext consumers

- [ ] **Base site switch.** Open the settings/config dialog. Switch the base site (e.g., rs → me or vice versa). **Pass:** top bar logo updates to reflect the new base site, currency on product prices updates, the app doesn't crash or show stale data. **Fail:** logo doesn't update, currency stays as the old base site's, crash, or stale data.

- [ ] **Language switch.** Switch the language (e.g., Serbian → English). **Pass:** all translatable strings in the UI refresh to the new language; navigation labels, button labels, and page content update. **Fail:** some strings stay in the old language, crash, or partial update.

---

## Foundation sanity (Φ1 + Φ2 still working)

- [ ] **Auth guards redirect — (secured)/.** While logged out, navigate to Favorites, Notifications, or Messages (the secured tabs). **Pass:** immediately redirected to `/` (home/login surface). **Fail:** the secured screen renders without a logged-in user, or crashes.

- [ ] **Auth guards redirect — owner/.** While logged out, attempt to navigate to any owner route. **Pass:** redirected to `/`. **Fail:** owner screen renders or crashes.

- [ ] **Auth guards redirect — owner/dashboard/.** While logged out, attempt to navigate to the dashboard. **Pass:** redirected to `/`. **Fail:** dashboard renders or crashes.

- [ ] **Tab switches preserve scroll position (Φ2).** Scroll down on the home feed. Switch to another tab (e.g., Favorites). Switch back to home. **Pass:** scroll position is preserved — you're at the same scroll position as before the tab switch. **Fail:** home feed resets to the top.

- [ ] **iOS swipe-back gesture (Φ2).** On iOS, navigate into a product detail or a nested screen. Swipe from the left edge to go back. **Pass:** native swipe-back animation works, previous screen slides in from the left. **Fail:** swipe-back doesn't work, or crashes. (Skip this item on Android.)

- [ ] **Foreground re-validation (Φ1).** Background the app for >5 minutes. Resume it. **Pass:** the app re-validates auth silently (no visible interruption for a valid session). **Fail:** crash on resume, or no re-validation (would only be detectable via backend logs or banned-user scenario).

- [ ] **Ban dialog (Φ1).** If possible: have an admin ban the test user while the app is open or backgrounded. Resume. **Pass:** a dialog appears informing the user they are banned; the user is signed out. **Fail:** no dialog, user stays signed in with scattered API errors. (Skip if not feasible to test — requires admin action on a test user.)

---

## Notes for Igor

- Each checkbox is independent. Check off as you go.
- If a D9.x bug appears to be accidentally fixed, stop and report to Mastermind — that's a behavior change outside spec §5's working-condition contract.
- The critical visual identity checks (city/category selectors) are the highest-risk items from Φ3 Brief 8. These carry the most visual-regression risk since SectionList replaced ScrollView.
- Φ3 does NOT change any visual design. Every screen should look identical to pre-Φ3. The only observable changes are: faster image loads, smoother scrolling, and the D9.5 stuck-spinner fix.
