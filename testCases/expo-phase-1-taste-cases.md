# Expo Φ1 — Manual Smoke Test Cases

**Feature:** Expo auth lifecycle foundation (Φ1)
**Spec:** `features/expo-auth-lifecycle.md`
**Date drafted:** 2026-05-25
**Status gate:** Φ1 flips from `shipped (code) / verifying` → `shipped` only after all critical tests pass on at least one real device.

---

## Setup before you start

- Build and run on a real device. Mid-range Android (Pixel 4a or equivalent) plus iPhone if possible. If only one, pick one.
- Have a way to back-end-ban a user: admin tool, direct DB update, or backend API call to `disableUser(userId, banReason)`.
- Have at least two test accounts: one normal user, one you can ban/unban.
- Have network inspection set up (Reactotron, Chrome DevTools over USB, or device-level proxy). You'll need this for Test 3.
- Have web set up too — Test 6 needs the web side to put a user into PENDING_DELETION (mobile delete flow ships in chat E, not Φ1).

---

## Test 1 — F2: Logout properly drains Firebase

**What this tests:** the F2 fix ensuring `auth.signOut()` is called before `GoogleSignin.signOut()`. Before Φ1, the Firebase session persisted after logout and the axios interceptor continued attaching the old user's Bearer token.

**Steps:**
1. Sign in as a normal user.
2. Note any API request being made (e.g., load favorites). Observe the `Authorization: Bearer <token>` header in network inspection.
3. Log out via the user menu.
4. Immediately make another API request — try to load any authenticated page.
5. **Expected:** the request either doesn't fire, or fires without an `Authorization` header. Should not carry the old user's Bearer token.
6. **Bonus check:** kill and reopen the app. **Expected:** you're signed out (not auto-signed-in with a persisted Firebase session).

**Failure signal:** the old token still attaches on requests after logout → F2 is broken.

---

## Test 2 — F3: Stores cleared on logout

**What this tests:** the F3 fix making `authStore.logout()` directly clear chat, favorites, notifications, portal filter, and dashboard filter stores. Before Φ1, these stores relied on fragile component effects.

**Steps:**
1. Sign in as user A.
2. Apply some filters on the portal (e.g., price range, region).
3. Favorite a product.
4. Open a chat with someone.
5. Log out.
6. Sign in as user B (different account).
7. **Expected:** user B sees their own favorites (not user A's), their own filter state reset (not user A's), their own chat list (not user A's). No flash of user A's data.

**Failure signal:** user B sees any of user A's data → F3 is broken.

---

## Test 3 — F22: No double-init on cold start

**What this tests:** the F22 fix gating `ChatsInit` and `NotificationsInit` on `_hasHydrated` before reading `user`. Before Φ1, these components fired cleanup against pre-hydration null user, then re-fired setup when hydration delivered the real user — double-init on every cold start.

**Steps:**
1. Sign in as a user.
2. Force-quit the app fully (swipe away from app switcher; iOS: double-tap home; Android: clear from recent apps).
3. Open Reactotron / DevTools / network inspector.
4. Cold-start the app.
5. **Expected:** the initial network calls for favorites, notifications, and chat subscriptions fire **once each**, not twice.

**Failure signal:** you see `firebase-sync` or favorites/notifications endpoints called twice in quick succession → F22 didn't take effect.

**Note:** the auth listener's single-flight 2-second window may absorb some doubles — that's correct behavior, not a failure. The failure mode is repeated subscribe → unsubscribe → subscribe cycles on Firestore listeners or repeated REST calls separated by more than ~100ms.

---

## Test 4 — F1 + F5: Auth listener catches banned users mid-session

**What this tests:** the F1 (listener wired) and F5 (axios 403 interceptor) plumbing together. A user banned mid-session should be force-logged-out with the ban dialog, not see scattered errors.

**Steps:**
1. Sign in as a test user.
2. Browse around — make sure the app is working normally.
3. From the backend (admin tool or direct DB update), ban that user. Set `User.disabled = true`, `User.banReason = '<some reason>'`. Run `DefaultUsersFacade.disableUser(userId, banReason)` if calling via API.
4. Wait briefly for backend cache eviction (or restart backend if your local environment requires it).
5. Make any API request from the app — open product detail, send message, anything authenticated.
6. **Expected:** the response interceptor catches `403 USER_BANNED`, signs you out, the ban-notice dialog appears with:
   - Title: "Your account is banned."
   - Body: three paragraphs explaining the situation, the appeals path (support@oglasino.com), and the 12-month duration.
   - Close button labeled "Go to home".

**Failure signals:**
- Scattered "something went wrong" errors instead of the dialog → F5 interceptor isn't matching the response shape.
- The dialog appears but contains placeholder text or English when device is in another locale → translation keys aren't resolving on mobile's i18n.
- App allows continued use after ban → F1 listener and F5 interceptor both failed.

---

## Test 5 — F1: Banned-email re-registration block

**What this tests:** the EMAIL_BANNED enforcement at the `firebase-sync` endpoint. A user with a banned email hash cannot register again.

**Steps:**
1. Use the banned user from Test 4.
2. Sign out (already signed out from Test 4's force-logout).
3. Try to sign in or register with that same email.
4. **Expected:** ban dialog appears. You can't get into the app.
5. From the backend, unban the user (`DefaultUsersFacade.enableUser`).
6. Try signing in again.
7. **Expected:** works normally.

**Failure signal:** sign-in succeeds while the user is still banned → backend ban-hash check failed, or mobile's interceptor missed `EMAIL_BANNED`.

---

## Test 6 — F1 + Brief 4B: Restoration dialog on sign-in for PENDING_DELETION user

**What this tests:** the F1 listener's handling of `X-Account-Restored: true` response header, plus the `AccountStateDialogsInit` restoration-dialog rendering.

**Setup requires web:** the mobile delete-account flow ships in chat E, not Φ1. So use web to put a user into PENDING_DELETION.

**Steps:**
1. On web, sign in as a test user and request account deletion via the Danger Zone on the user settings page (per `user-deletion.md` §4.1).
2. On mobile, attempt to sign in as that same user.
3. **Expected:** sign-in succeeds. The backend's `firebase-sync` handler restores the account (per auth-contract C-2/C-9) and sets `X-Account-Restored: true` response header. Mobile's axios interceptor reads the header, sets `restored: true` flag, `AccountStateDialogsInit` opens the restoration dialog with:
   - Title: "Welcome back"
   - Subtitle: "Your account has been restored. The pending deletion has been cancelled."

**Failure signals:**
- Sign-in succeeds but no dialog appears → header reading or `AccountStateDialogsInit` is broken.
- Sign-in fails entirely → the auth listener is incorrectly rejecting PENDING_DELETION users (it should let them in so restoration can fire).
- Wrong dialog appears or wrong text → translation key resolution issue.

---

## Test 7 — F4: Foreground re-validation after 5+ minutes

**What this tests:** the F4 `ForegroundRevalidationInit` component that calls `syncUserToBackend` on foreground resume after >5 minutes of backgrounding.

**Steps:**
1. Sign in as a test user.
2. Send the app to background (home button, app switcher, lock screen).
3. **Wait at least 5 minutes.** Yes, actually 5 minutes. The threshold is locked at `5 * 60 * 1000` ms.
4. From the backend during the wait, ban that user (`disableUser(userId, banReason)`).
5. After 5+ minutes elapsed, bring the app back to foreground.
6. **Expected:** within a second or two, `syncUserToBackend` fires automatically. Backend returns 403 USER_BANNED. App signs you out, ban dialog appears.

**Failure signal:** you can keep using the app after foregrounding despite being banned → F4 listener didn't fire, or the foreground threshold gate is wrong.

### Variant 7b — Under-5-minute background does NOT re-validate

**Steps:**
1. Sign in as a test user.
2. Background the app for only ~30 seconds.
3. Background-ban the user (same as Test 7 step 4).
4. Bring the app back to foreground.
5. **Expected:** no `syncUserToBackend` re-validation fires. The ban is not detected until the next deliberate API call.

**This is intentional behavior** — the 5-minute threshold avoids burning network on every quick app-switch. If under-5-minute foregrounding does fire `syncUserToBackend`, the threshold gate is wrong.

---

## Test 8 — F13: Auth guards on secured routes

**What this tests:** the F13 layout-level redirects for unauthenticated users. Three layouts gated: `(portal)/(secured)/_layout.tsx`, `owner/_layout.tsx`, `owner/dashboard/_layout.tsx`.

**Steps:**
1. Sign out (be on the public surface).
2. Try to navigate directly to a secured route. Options:
   - **Deep link** if you have one set up: `oglasino://favorites`, `oglasino://messages`, `oglasino://notifications`, or `oglasino://owner/dashboard`.
   - **Programmatic navigation** via dev tools, or temporarily modify a button to push to a secured route.
3. **Expected:** you're immediately redirected to home (`/`). No flash of the secured screen content. No crash.

### Variant 8b — Sign back in and reach the route normally

1. Still signed out from Test 8 step 1.
2. Sign in.
3. Navigate to `/favorites` (or any other secured route) the normal way.
4. **Expected:** route loads correctly. The guard correctly allows authenticated users through.

**Failure signals:**
- You reach the secured screen as an unauthenticated user → F13 guard didn't fire.
- Crash on `DashboardSidebar.tsx` accessing `user.profileImageKey` → owner layout guard didn't take effect or didn't redirect before render.
- Authenticated user is redirected away → guard logic is wrong.

---

## Test 9 — F1 + F5: Token rotation during long session

**What this tests:** Firebase ID tokens expire after ~1 hour. F1's listener handles rotation via `onIdTokenChanged`; F5's axios interceptor handles 401 with force-refresh-and-retry on per-request basis.

**Steps:**
1. Sign in.
2. Use the app actively for >1 hour. Two options:
   - Wait for real-time expiry (recommended but tedious).
   - Manipulate device time forward (riskier — may cause other timing-sensitive bugs).
3. **Expected:** when the token expires, the next API call returns 401, the interceptor force-refreshes via `getIdToken(true)`, retries the original request, success. The user notices nothing — no sign-out, no dialog, no error.

**Failure signal:** you get signed out after ~1 hour of normal use → F5's 401 retry path is broken.

---

## Quick-pass test set (~10–15 minutes)

If short on time, run only these five critical tests. They cover the security/trust failures (banned users get out, sessions clean up, routes guard correctly):

1. **Test 1** — Logout drains Firebase
2. **Test 2** — Stores cleared on logout
3. **Test 4** — Banned user mid-session triggers ban dialog
4. **Test 7** — Foreground re-validation after 5 minutes
5. **Test 8** — Auth guards on secured routes (signed-out attempt)

Tests 3, 5, 6 are valuable verifications but not blocking. Test 9 needs a device-time-manipulation or hour-long wait.

---

## Recording results

For each test, record:
- Pass / Fail
- Device (e.g., "Pixel 6, Android 14" or "iPhone 13, iOS 17")
- App build version
- If fail: what you saw vs what was expected, plus any relevant network logs or error messages.

---

## If a test fails

1. Record the failure in detail.
2. Bring the failure report to a new Mastermind chat (or a continuation of this one if it's still open).
3. We open a fix brief targeted at the specific finding.
4. **Do not commit Φ1 closure to `main` until the smoke pass is complete.**

The status in `state.md` stays at `shipped (code) / verifying` until you confirm the smoke pass. Then it flips to fully `shipped` and Φ2 opens.