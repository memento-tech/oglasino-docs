# Audit — Notifications Toggle Removal (mobile read-only)

**Repo:** oglasino-expo
**Branch:** main (working tree; foundation work historically on `new-expo-dev`)
**Date:** 2026-06-06
**Type:** read-only audit. No code changes.
**Brief:** Confirm the current state of mobile's dead `allowNotifications` toggle and how mobile registers push tokens, so removing the web toggle + backend flag doesn't surprise mobile. Cross-check every file:line.

---

## TL;DR

- The `allowNotifications` toggle is **dead exactly as the 2026-05-30 consent-mode-mobile decision recorded it**: rendered on the settings screen, backed by a local `useState` that is **never seeded** from the profile fetch and **never sent** in any update request. Default-on-screen value is always `false`.
- Mobile push delivery does **not** depend on this toggle at all. There is a **fully-built, independent notifications module** (`src/notifications/`) that registers an Expo push token on the **boot/auth path**, keyed on the logged-in user — completely orthogonal to `allowNotifications`.
- **No mobile code reads `user.allowNotifications`** (the AuthUserDTO field) or `details.allowNotifications` (the UpdateUserDTO field). The field is declared on both DTOs but has zero readers. Dropping it from the backend user DTO is **safe for mobile** — TypeScript optional field, no consumer.

The web toggle + backend-flag removal is therefore safe from mobile's side. The one thing to flag for whoever plans the web/backend change: mobile's push token path is its own first-class feature, not gated on any user-preference flag — see Q2.

---

## Q1 — The dead toggle: rendered where, and confirmed never seeded / never sent?

**Rendered:** `app/owner/user.tsx`

- Local state declared, default `false`: `app/owner/user.tsx:50`
  ```ts
  const [allowNotifications, setAllowNotifications] = useState(false);
  ```
- Label section (uses COOKIES keys `notifications.label` / `.description` / `.warning`): `app/owner/user.tsx:277-280`
- The `Switch` itself: `app/owner/user.tsx:283-287`
  ```tsx
  <Switch
    className="self-start"
    value={allowNotifications}
    onValueChange={setAllowNotifications}
  />
  ```

**Never seeded — confirmed.** The profile-fetch effect seeds every other field but **not** `allowNotifications`: `app/owner/user.tsx:67-82`. The `.then((details) => {...})` block calls `setEmail`, `setDisplayName`, `setShortBio`, `setPhoneNumber`, `setSelectedRegionAndCity`, `setAllowEmails`, `setAllowPromoEmails`, `setAllowPhoneCalling`, `setProfileImageKey` — there is **no `setAllowNotifications(...)` call**. So the toggle always renders `false` regardless of the backend value. (Lines 76-78 seed the three sibling flags; `allowNotifications` is conspicuously absent.)

**Never sent — confirmed.** Two places that would carry it, both omit it:
- Change-detection comparison block: `app/owner/user.tsx:97-108` — compares `email`, `displayName`, `shortBio`, `phoneNumber`, `selectedRegionAndCity`, `allowEmails`, `allowPromoEmails`, `allowPhoneCalling`. **`allowNotifications` is not in the comparison.** So flipping it never even marks the form dirty.
- The `updateUser({...})` save body: `app/owner/user.tsx:178-190` — sends `id`, `firebaseUid`, `email`, `displayName`, `profileImageKey`, `shortBio`, `phoneNumber`, `regionAndCity`, `allowEmails`, `allowPromoEmails`, `allowPhoneCalling`. **`allowNotifications` is not in the body.**

Net: the toggle is a pure no-op control. It holds a local boolean the user can flip on screen, but the value is born `false` on every screen mount and dies on unmount — never read from the server, never written back. Matches the consent-mode-mobile spec's Part C note verbatim ("never seeded, never sent", `features/consent-mode-mobile.md:183`).

---

## Q2 — Mobile push-token registration: how/where does mobile attach its token to the backend? Boot/auth path, independent of any toggle?

**Yes — it is on the boot/auth path and independent of `allowNotifications`.** Mobile has a complete `src/notifications/` module. The token-attach flow:

**The wire call (token → backend):** `src/notifications/service/pushTokenService.ts:3-8`
```ts
export const attachPushTokenToBackend = async (token: string) => {
  await BACKEND_API.post('/secure/push/token', {
    token,
    platform: 'EXPO',
  });
};
```
Detach counterpart (on logout / permission-revoked): `pushTokenService.ts:10-20` → `POST /secure/push/token/detach` with `{ pushToken }`.

**Where the token is obtained and attached:** `src/notifications/lib/pushNotificationRegister.ts:13-43` (`registerForPush(userId)`):
- guards `Device.isDevice` (`:14`)
- reads/refreshes OS permission (`:18-27`)
- fetches the Expo push token via `Notifications.getExpoPushTokenAsync({ projectId: ... })` (`:29-33`)
- dedups against `currentToken`/`currentUserId` (`:35`)
- calls `attachPushTokenToBackend(token)` (`:39`)
- wires a token-refresh listener (`:60-70`, re-attaches on rotation) and a permission-revoked AppState listener (`:72-85`, detaches if the user revokes OS permission while the app is backgrounded/foregrounded).

**The trigger (boot/auth path):** `src/notifications/components/PushNotificationsInit.tsx`
- Component is mounted as a child of `AppInit`: `src/components/init/AppInit.tsx:28` (`<PushNotificationsInit />`). `AppInit` runs on the always-mounted init tree; per the boot redesign its visible siblings run once `bootStatus === 'ready'`.
- The registration effect keys on **`user`** (auth state), `PushNotificationsInit.tsx:157-189`:
  - computes `currentUserId = user?.id ?? null` (`:159`), bails if unchanged (`:161`)
  - if no user → `unregisterFromPush()` (`:165-168`) — i.e. logout detaches
  - if a user is present and OS permission is already grantable/granted → `registerForPush(user.id)` (`:184`)
  - otherwise opens the soft-permission dialog first (see Q3)

**Trigger summary:** registration fires on **user identity change** (login / boot-with-stored-session), not on any settings toggle. The only inputs to the decision are: presence of an authenticated `user`, `Device.isDevice`, OS permission state, and a 5-day soft-prompt throttle. **`allowNotifications` is never consulted anywhere in this path.** This is the pattern web may need to mirror: push subscription is its own boot/auth-driven lifecycle, decoupled from the user-preferences flag.

(Context, not required by the brief: the module also handles deep-link navigation from tapped notifications — `PushNotificationsInit.tsx:75-155` — and a separate `NotificationsInit` / `useNotificationStore` drives the in-app Firestore notifications feed, `src/lib/client/firebaseNotifications.ts`. Neither reads `allowNotifications`.)

---

## Q3 — OS notification permission: where is it requested, and what gates it?

**Two-stage prompt: an in-app "soft" prompt, then the OS prompt.**

**Soft prompt (in-app dialog, shown first):** `src/notifications/components/PushNotificationsInit.tsx:175-186`
- `DialogId.SOFT_PUSH_PERMISSION_DIALOG` (defined `src/components/dialog/dialogRegistry.ts:20`)
- Gated by, in order (`PushNotificationsInit.tsx:157-182`):
  1. authenticated `user` present / `user.id` changed (`:159-168`)
  2. `Device.isDevice` (`:170`)
  3. `shouldShowPrompt()` — a **5-day throttle** keyed on AsyncStorage `LAST_PROMPT_KEY` (`'SPPL'`), `:55-64` and `:172-173`
  4. current OS permission `!== 'granted'` **and** `canAskAgain` (`:177`)
- On those conditions it stamps `LAST_PROMPT_KEY` (`:178`) and opens the soft dialog with an `onAccept` that sets `softPermissionAccepted` (`:180-182`). If permission is already granted (else branch), it skips straight to `registerForPush(user.id)` (`:183-185`).

**OS prompt (the real system permission dialog):** `src/notifications/lib/pushNotificationRegister.ts:22-25`
```ts
if (status !== 'granted') {
  const permission = await Notifications.requestPermissionsAsync();
  finalStatus = permission.status;
}
```
- Reached only via `registerForPush`, which is invoked either directly (permission already granted branch, `PushNotificationsInit.tsx:184`) or after the user accepts the soft prompt → effect at `PushNotificationsInit.tsx:191-201` → `registerForPush(user.id)` (`:196`).

**What gates the OS prompt overall:** authenticated user + real device + (already-granted OR user accepted the soft prompt). **No gate reads `allowNotifications`.** The settings toggle plays no role in whether the OS prompt fires.

---

## Q4 — If the backend drops `allowNotifications` from the user DTO, does any mobile code read `user.allowNotifications`?

**Not found.** No mobile code reads the field.

Exhaustive grep of `allowNotifications` across `src/` and `app/` returns exactly four hits, none of which is a read of the DTO field:
- `src/lib/types/user/AuthUserDTO.ts:13` — `allowNotifications?: boolean;` (optional field **declaration** only)
- `src/lib/types/user/UpdateUserDTO.ts:12` — `allowNotifications?: boolean;` (optional field **declaration** only)
- `app/owner/user.tsx:50` — local `useState` (not derived from `user`/`details`)
- `app/owner/user.tsx:285-286` — the `Switch` bound to that local state

A targeted grep for `.allowNotifications` (property access on any object) returns **zero** hits outside the two DTO declarations. Nothing reads `user.allowNotifications` or `details.allowNotifications`.

**Conclusion:** dropping `allowNotifications` from the backend user DTO is safe for mobile. Both DTO declarations are `?:` optional, so the wire shape narrowing (field absent) type-checks with no change; and since nothing reads the field, there is no runtime consumer to break. The two dead DTO declarations and the dead toggle in `user.tsx` could be cleaned up in a future mobile session, but that is **not required** for the web/backend removal to be safe — it is independent local dead code, owned by the future notifications-feature work per the consent-mode-mobile spec (`features/consent-mode-mobile.md:183`).

---

## Cross-repo safety verdict (for the web/backend removal planner)

| Concern | Finding | file:line |
| --- | --- | --- |
| Does mobile send `allowNotifications` to `/secure/user/update`? | No | `app/owner/user.tsx:178-190` (absent) |
| Does mobile read `allowNotifications` off the GET response? | No | `app/owner/user.tsx:67-82` (not seeded) |
| Does mobile read `user.allowNotifications` anywhere? | No | grep: only 2 DTO decls + 1 local useState |
| Does mobile push delivery depend on the flag? | No — independent boot/auth token path | `pushNotificationRegister.ts:13-43`, `PushNotificationsInit.tsx:157-189`, `pushTokenService.ts:3-8` |
| Will dropping the field from the backend DTO break mobile types? | No — both DTO fields are optional (`?:`) | `AuthUserDTO.ts:13`, `UpdateUserDTO.ts:12` |

Removing the web toggle and the backend flag will not surprise mobile. Mobile's notification opt-in is the OS permission grant + the device's push-token registration, not a server-stored boolean.
