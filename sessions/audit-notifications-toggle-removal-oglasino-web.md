# Audit — Notifications Toggle Removal

**Date:** 2026-06-06
**Repo:** oglasino-web
**Type:** Read-only audit. No code changed.
**Question driving this audit:** If we remove the "Allow notifications" toggle on
the user settings screen, do we also remove the only path that registers a web
push token? Every file:line below was cross-checked with `grep`/`cat`.

---

## Headline answer

**No.** Removing the toggle would NOT remove the only push-token registration path.
A web push token is attached to the backend on the **auth/boot path**, independent
of the toggle, via `UseTokenRefresh`'s `onIdTokenChanged` listener
(`src/components/client/initializers/UseTokenRefresh.tsx:103`). That listener calls
`initPushForAuthenticatedUser()` on every sign-in / sign-up / token rotation, with
no involvement from the settings screen.

The toggle is therefore a **secondary / redundant** registration trigger, not the
sole one. See the important caveat in Q3 about the permission prompt.

---

## 1. The toggle: render location, backing state, and saveChanges() side effects

**Rendered at:** `app/[locale]/owner/user/page.tsx:331-336` — a shadcn `<Switch id="notifications">`.

**Backing local state:** `app/[locale]/owner/user/page.tsx:52`
`const [allowNotifications, setAllowNotifiations] = useState(false);`
(note: the setter is misspelled `setAllowNotifiations` in the code — cosmetic, not in scope).
- Initialized from backend on load: `:69` `setAllowNotifiations(details.allowNotifications || false);`
- Updated by the switch: `:334` `onCheckedChange={(checked) => setAllowNotifiations(checked)}`

**What `saveChanges()` does with it** (`app/[locale]/owner/user/page.tsx:98-249`):

1. **Change detection** (`:113`): `allowNotifications !== userDetails.allowNotifications`
   contributes to the `userChanged` flag that gates whether a save proceeds at all.
2. **Push side effect** (`:220`): `handleNotificationToggle(allowNotifications);`
   - Defined at `:88-96`. If `checked === true` → `await initPushForAuthenticatedUser()`
     (`:92`). If `false` → `await detachPushToken()` (`:94`).
   - NOTE: it is called **un-awaited** at `:220` (the call is `async` but its promise is
     not awaited), and it fires on **every** save regardless of whether the toggle value
     actually changed — i.e. saving an unrelated field with the toggle ON re-runs
     `initPushForAuthenticatedUser()`, and with the toggle OFF re-runs `detachPushToken()`.
3. **Persist to backend** (`:222-234`): `updateUser({ ..., allowNotifications, ... })`
   sends `allowNotifications` in the `UpdateUserDTO` body so the backend
   `user.allowNotifications` column is updated.

Full list of save-time side effects tied to the toggle: (a) participates in the
`userChanged` gate, (b) triggers attach/detach of the push token via
`handleNotificationToggle`, (c) is persisted to the backend user record.

---

## 2. Token registration: every call site of `initPushForAuthenticatedUser`

Defined at `src/notifications/lib/devicePush.ts:29-50`. It posts the FCM token to
`/secure/push/token` via `attachPushTokenToBackend` (`src/notifications/service/pushTokenService.ts:3-8`).

Two call sites:

| # | File:line | Trigger | On boot/auth path? |
|---|-----------|---------|--------------------|
| 1 | `src/components/client/initializers/UseTokenRefresh.tsx:103` | Inside the `onIdTokenChanged` listener, after a successful `syncUserToBackend`, on every sign-in / sign-up / token rotation | **YES** |
| 2 | `app/[locale]/owner/user/page.tsx:92` | Settings screen, `handleNotificationToggle(true)`, run from `saveChanges()` | No |

**The boot-path one (call site #1) is the answer to "the single most important
question."** `UseTokenRefresh` is the sole `onIdTokenChanged` listener
(`UseTokenRefresh.tsx:23-47`). It is mounted app-wide: `AppInit`
(`src/components/client/initializers/AppInit.tsx:54`) renders `<UseTokenRefresh />`,
and `AppInit` is mounted in the root locale layout (`app/[locale]/layout.tsx:48`).
So on every authenticated load/login, `initPushForAuthenticatedUser()` runs
**without the user touching the toggle**.

(`AuthInit` — `app/[locale]/layout.tsx:44` → `AuthInit.tsx` — only sets Firebase
persistence; it does not register push. The listener lives in `UseTokenRefresh`.)

---

## 3. Permission prompt: where `Notification.requestPermission()` is called and its gating

**Only call site:** `src/notifications/lib/devicePush.ts:36`, inside
`initPushForAuthenticatedUser`.

Gating (`devicePush.ts:30-40`):
- `:32` returns early if `Notification.permission === 'denied'`.
- `:35-38` calls `Notification.requestPermission()` **only when permission is `'default'`**,
  and bails if the result is not `'granted'`.
- `:40` final guard: returns unless `hasPushPermission()` (i.e. `=== 'granted'`).

So the browser permission prompt is correctly gated on `default` state and is never
re-shown once granted or denied.

**Does anything prompt at login/boot today? YES.** Because call site #1
(`UseTokenRefresh.tsx:103`) runs on the auth/boot path and calls
`initPushForAuthenticatedUser`, a first-time authenticated user whose permission is
still `'default'` is prompted by the browser at login/boot — independent of the
toggle. This is the behavior to be aware of when reasoning about toggle removal: the
permission prompt and the token registration already happen at boot, not (only) at
the toggle.

`ForegroundPushInit.tsx:16` reads `Notification.permission` but only to decide
whether to display a foreground notification — it does **not** call `requestPermission`.

---

## 4. The flag: where `allowNotifications` is read on the web side beyond the toggle

Grep for `allowNotifications` / `.allowNotifications` across `*.ts`/`*.tsx`:

- `app/[locale]/owner/user/page.tsx:52, 69, 113, 220, 232, 333` — the settings screen
  only (local state, load, change-detection, save-time side effect, persist, render).
- `src/lib/types/user/AuthUserDTO.ts:13` — type field declaration (`allowNotifications?: boolean`).
- `src/lib/types/user/UpdateUserDTO.ts:12` — type field declaration.

**No consumer of `allowNotifications` exists beyond the settings screen.** Nothing
in the push-init path, the messaging path, or anywhere else reads the flag to decide
whether to register/show notifications. In particular,
`initPushForAuthenticatedUser` does **not** check `allowNotifications` — it only
checks the browser `Notification.permission`. So the backend `allowNotifications`
column is, on the web side, written-and-displayed only; it does not currently gate
any web behavior. (Whether the backend uses it to gate push delivery is a
backend-repo question, out of scope here.)

---

## 5. detachPushToken: every call site

Defined at `src/notifications/lib/devicePush.ts:52-65` → posts to
`/secure/push/token/detach` via `detachPushTokenFromBackend`
(`src/notifications/service/pushTokenService.ts:10-20`).

Two call sites:

| # | File:line | Trigger | Beyond the toggle's OFF path? |
|---|-----------|---------|-------------------------------|
| 1 | `app/[locale]/owner/user/page.tsx:94` | `handleNotificationToggle(false)` from `saveChanges()` — the toggle OFF path | toggle |
| 2 | `src/lib/store/useAuthStore.ts:250` | `logout()` — detaches the token before signing out of Firebase | **YES — logout** |

So besides the toggle's OFF path, `detachPushToken` is also called on **logout**
(`useAuthStore.ts:248-264`).

---

## Implication for the removal decision (not a recommendation, just what the code says)

- Removing the toggle and its `handleNotificationToggle` side effects would **not**
  stop web push registration: the boot path (`UseTokenRefresh.tsx:103`) still attaches
  the token, and `logout` (`useAuthStore.ts:250`) still detaches it.
- The boot path would still prompt for permission at login when permission is
  `'default'` (`devicePush.ts:35-36`).
- The only user-facing way to **detach** a token while staying logged in (the toggle
  OFF path, `page.tsx:94`) would disappear; the remaining detach trigger is logout.
- The backend `user.allowNotifications` field would no longer be writable from web,
  and (per Q4) nothing on web reads it anyway — but the backend may. Flag for
  Mastermind/Backend to confirm before removal.

---

## Items I could not find / explicitly not asserted

- No place other than the settings screen reads `allowNotifications` — **not found**
  (confirmed by grep, Q4).
- No `requestPermission` call other than `devicePush.ts:36` — **not found** any other.
- Whether the backend gates push delivery on `allowNotifications` — **out of scope**
  (backend repo); not asserted here.
