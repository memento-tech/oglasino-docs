# Notifications Toggle Removal

**Status:** shipped (2026-06-06)
**Repos:** oglasino-backend, oglasino-web, oglasino-expo
**Schema:** V1-fold edit-in-place (pre-prod), no new migration.

## Goal

Remove the "Allow notifications" preference toggle from the user settings screen
on both clients, and remove the backend `allowNotifications` flag end-to-end. The
toggle is not legally required (legal adviser confirmed, 2026-06-06) and the flag
gates no behavior. After removal, a user's notification opt-out is the OS/browser
permission grant plus push-token registration — there is no server-stored
preference flag.

## Why this is safe (audit findings, 2026-06-06)

Three read-only Phase-2 audits (one per repo) established:

1. **Dispatch gates on nothing.** Backend `fanOutPush`
   (`DefaultNotificationsService`) sends to every push token that exists for a
   user. It never reads `allowNotifications`. The flag is dead with respect to
   delivery.
2. **Web push registration is independent of the toggle.** A web push token is
   attached on the auth/boot path via `UseTokenRefresh`'s `onIdTokenChanged`
   listener (`UseTokenRefresh.tsx:103` → `initPushForAuthenticatedUser`), on every
   sign-in / token rotation. The toggle is a redundant second trigger, not the
   sole one. Removing it does not stop registration.
3. **Mobile already independent.** The mobile toggle is dead (never seeded, never
   sent, per the 2026-05-30 consent-mode-mobile decision). Mobile registers its
   token on the boot/auth path via `PushNotificationsInit`. Mobile reads
   `allowNotifications` nowhere.
4. **No other reader.** Backend has exactly four readers of the flag, all of which
   just round-trip it through DTOs/converters. No cron, query, or serializer reads
   it for a decision. Zero backend tests assert on it.

## Behavior changes accepted (model decision, Igor, 2026-06-06)

After removal:

- **OS/browser permission is the off-switch.** The only user-facing way to stop
  push on a device is to revoke the OS (mobile) or site (browser) notification
  permission. Logout detaches the device's token.
- **No durable in-app silence on web.** Previously the toggle's OFF path detached
  this device's token without logging out. That path is gone. Because web
  auto-registers at boot when permission is `granted`, a user cannot be
  logged-in-and-silent on web without revoking browser permission. This is
  intentional under the permission-is-truth model.
- **Permission-prompt timing is unchanged.** Both clients already prompt on the
  auth/boot path when permission is `default` — the prompt was never driven by the
  toggle. Removing the toggle changes nothing about when the prompt fires.

## Ordering

No hard sequencing gate. Both client DTO fields (`allowNotifications`) are optional
(`?:`), so the backend can ship the field-absent payload and the clients can drop
their reads in any order without a coordination window. Recommended order is
backend → web → mobile → docs, for cleanliness, not correctness.

## Removal sets (from the audits — authoritative for the engineering briefs)

### Backend (`oglasino-backend`)
- `User.java:112` field `allowNotifications` + getter/setter `:304-310`.
- `V1__init_schema.sql:590` column `allow_notifications` (V1-fold, no V2).
- Admin seed inserts — drop the column name AND its `false` VALUES entry in:
  `data-admin-dev.sql:19`, `data-admin-prod.sql:18`, `data-admin-stage.sql:18`.
- DTOs: `UpdateUserDTO.java:22` (+ accessors `:92-97`),
  `AuthUserDTO.java:15` (+ accessors `:104-109`),
  `ImportUserData.java:24` (+ accessors `:157-162`).
- Reader lines (delete in same change): `UpdateUserConverter.java:47`,
  `AuthUserConverter.java:41`, `DefaultUserFacade.java:188`,
  `TestUsersImportService.java:68`.
- Tests: none — `src/test` grep for the flag returned zero hits.

### Web (`oglasino-web`)
- `app/[locale]/owner/user/page.tsx` — remove the `<Switch id="notifications">`
  (`:331-336`), the backing state `:52`, load-seed `:69`, the change-detection
  contribution `:113`, the `handleNotificationToggle` call `:220` and its
  definition `:88-96`, and the `allowNotifications` field in the `updateUser` body
  `:222-234`.
- DTO type fields: `AuthUserDTO.ts:13`, `UpdateUserDTO.ts:12`.
- Do NOT touch `initPushForAuthenticatedUser` / `detachPushToken` / `devicePush.ts`
  / `UseTokenRefresh.tsx` / `useAuthStore.ts` logout detach — the auth-path
  registration and the logout detach stay exactly as they are.

### Mobile (`oglasino-expo`)
- `app/owner/user.tsx` — remove the dead `<Switch>` (`:283-287`), its label section
  (`:277-280`), and the backing state `:50`.
- DTO type fields: `AuthUserDTO.ts:13`, `UpdateUserDTO.ts:12`.
- Do NOT touch the `src/notifications/` module — push registration is independent.
- The COOKIES translation keys `notifications.label` / `.description` / `.warning`
  used by the dead label become unused after this — the engineer flags them for the
  Ω teardown rather than deleting cross-repo backend seeds in a mobile session.

## Definition of done

- Toggle gone from both clients; flag gone from backend entity/column/DTOs/converters/seeds.
- Backend boots (seeds don't 500 on the dropped column).
- `./mvnw test` green (no test asserted on the flag); web `tsc`/`lint`/`test` green;
  mobile `tsc`/`lint`/`test` green.
- Push still registers on auth/boot on both clients (verify on web smoke: log in,
  confirm token attach still fires; permission prompt still appears for a
  `default`-permission user).
- decisions.md entry applied; this spec reflects shipped state at close.

## Out of scope

- The notification permission-prompt UX (contextual prompting, opt-in rate) — the
  prompt timing is unchanged by this feature; any future improvement is its own chat.
- Cleanup of the now-unused mobile COOKIES translation keys — routed to the Ω
  teardown, not done here.

## Shipped (2026-06-06)

Removed across all three repos, all DoD-green:
- **Backend:** entity field + V1 column + 3 admin-seed values + 3 DTO fields
  (`UpdateUserDTO`, `AuthUserDTO`, `ImportUserData`) + 4 reader lines + 3
  `testUsers.json` fixture entries (the fixture source was a Phase-2 audit miss,
  caught and folded in by the backend engineer via the grep-zero DoD). 969 tests
  green, grep-zero across `src/main`+`src/test`.
- **Web:** toggle + its save side effects + dead `devicePush` import + 2 DTO type
  fields. Auth-path registration (`UseTokenRefresh.tsx:103`) and logout detach
  (`useAuthStore.ts:250`) untouched and confirmed still firing. tsc clean, grep-zero.
- **Mobile:** dead toggle + label + 2 DTO type fields. `src/notifications/`
  untouched. tsc clean, grep-zero. (Toggle was already inert per the 2026-05-30
  consent-mode-mobile decision — audit's inert claim confirmed exactly.)

The `allowNotifications` flag no longer exists anywhere. Off-switch is OS/browser
permission; logout detaches the device token; push registers on the auth/boot path
on both clients. No behavioral gate was added — dispatch was already token-only.
