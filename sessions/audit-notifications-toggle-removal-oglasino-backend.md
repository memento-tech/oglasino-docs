# Audit — Notifications Toggle Removal (backend blast radius)

**Repo:** oglasino-backend
**Type:** Phase-2 read-only audit. No code changed.
**Date:** 2026-06-06
**Question:** if the account-wide `allowNotifications` flag is removed entirely, what is the blast radius?

Every claim below is grounded in `file:line` from `grep`/`cat` against the working tree on branch `dev`. "Not found" means an exhaustive search returned nothing — no inference.

---

## Headline

The dispatch path **does not gate on `allowNotifications`**. Nothing in the push/in-app fan-out reads the flag (Q2). The flag is purely a stored preference that round-trips through two user DTOs and is written back on profile update — it has no behavioral consumer. Removing it touches: 1 entity field, 1 V1 column, 4 SQL seed inserts, 2 response/request DTOs, 2 ModelMapper converters, 1 facade write, and 1 dev/test import path. No cron, no query, no serializer reads it for any decision.

---

## Q1 — The column: definition, default, where set

**Entity field** — `src/main/java/com/memento/tech/oglasino/entity/User.java:112`

```java
private boolean allowNotifications = false;
```

Java-side default `false`. No `@Column` annotation (implicit name `allow_notifications`). Getter/setter at `User.java:304-310`.

**V1 schema** — `src/main/resources/db/migration/V1__init_schema.sql:590`

```sql
allow_notifications boolean NOT NULL,
```

`NOT NULL` with **no DB-level `DEFAULT`** (contrast `disabled boolean DEFAULT false NOT NULL` two lines down at 593). Every `INSERT` into `users` must supply the value explicitly. (Pre-prod V1-fold convention, conventions Part 12 — edit V1 in place, no V2 file.)

**Where set on the write path** — `src/main/java/com/memento/tech/oglasino/facade/impl/DefaultUserFacade.java:188`

```java
user.setAllowNotifications(updateData.isAllowNotifications());
```

Inside `updateCurrentUserData(UpdateUserDTO)` (method opens `DefaultUserFacade.java:162`). This is the **only** mutation of the flag from user-facing input. The value is taken straight from the request body — it is a stored preference, not a server-derived value.

**Also set on the seed/import paths:**
- Admin seed inserts name the column in their column list and pass `false`:
  `data-admin-dev.sql:19`, `data-admin-prod.sql:18`, `data-admin-stage.sql:18` (value `false` in the VALUES block).
- Dev/test user import: `src/main/java/com/memento/tech/oglasino/data/user/service/TestUsersImportService.java:68` →
  `entity.setAllowNotifications(userData.isAllowNotifications())`, fed from `ImportUserData.java:24`.

---

## Q2 — Dispatch gate: does the push fan-out read `allowNotifications`?

**No. The dispatch path does not read `allowNotifications` anywhere.** Grep for `allowNotifications`/`allow_notifications` across `src/main` returns **zero** hits in any notifications package file (`notifications/**`), and zero in any of the four dispatch callers. Confirmed below by quoting the full path.

**Dispatch code path** — `src/main/java/com/memento/tech/oglasino/notifications/service/impl/DefaultNotificationsService.java`

In-app + push (`sendAsyncNotification`, lines 45-79): writes a Firestore `userNotifications` doc, then:

```java
// :49-50
var pushTokens =
    ListUtils.emptyIfNull(pushTokenService.getUserTokens(notification.getUserId()));
// :78
fanOutPush(notification, pushTokens);
```

Push-only (`sendAsyncPushOnly`, lines 81-91): loads tokens and fans out, no flag check:

```java
// :88-90
var pushTokens =
    ListUtils.emptyIfNull(pushTokenService.getUserTokens(notification.getUserId()));
fanOutPush(notification, pushTokens);
```

`fanOutPush` (lines 94-110) selects a `PushService` per token platform and calls `pushService.sendPush(...)`. **No `allowNotifications` read at any step.** Token presence is the only gate — a user with zero tokens receives nothing; a user with tokens receives everything regardless of the flag.

`getUserTokens` (`DefaultPushTokenService.java:24-30`) is a plain `findAllByUserId` — no flag join, no filter.

**All four dispatch entry points** call the same ungated methods:
- `admin/facade/impl/DefaultAdminReportFacade.java:119` → `sendAsyncNotification`
- `admin/facade/impl/DefaultAdminProductFacade.java:125` → `sendAsyncNotification`
- `facade/impl/DefaultFavoriteProductFacade.java:120` → `sendAsyncNotification`
- `facade/impl/DefaultUserFacade.java:90` → `sendAsyncNotification`
- `controller/MessageNotificationController.java:30` → `notifyMessageSent` → `sendAsyncPushOnly` (`DefaultMessageNotificationService.java:103`)

None of those five call sites reads `allowNotifications` (verified: not in the grep hit list). **The flag is dead with respect to delivery.**

---

## Q3 — The flag on the wire: which DTOs carry it, any other reader

`allowNotifications` is present on **two** user DTOs, both round-tripped through ModelMapper converters:

1. **`UpdateUserDTO`** — `src/main/java/com/memento/tech/oglasino/dto/UpdateUserDTO.java:22` (field), getter/setter `:92-97`.
   This DTO is **both** the GET response **and** the update request body for the profile screen:
   - GET `/api/secure/user/update` returns it — `UserUpdateController.java:22-25` (`ResponseEntity<UpdateUserDTO> getUser()`), built by `DefaultUserFacade.getCurrentUserUpdateData()` (`:154-158`) via `modelMapper.map(user, UpdateUserDTO.class)`, populated in `UpdateUserConverter.java:47` (`destination.setAllowNotifications(source.isAllowNotifications())`).
   - POST `/api/secure/user/update` consumes it — `UserUpdateController.java:27-28` → `updateCurrentUserData` → written at `DefaultUserFacade.java:188`.

2. **`AuthUserDTO`** — `src/main/java/com/memento/tech/oglasino/dto/AuthUserDTO.java:15` (field), getter/setter `:104-109`.
   This is the **login/auth response**. `AuthController.java:163` does `modelMapper.map(user, AuthUserDTO.class)`, populated in `AuthUserConverter.java:41` (`destination.setAllowNotifications(source.isAllowNotifications())`).

3. **`ImportUserData`** — `src/main/java/com/memento/tech/oglasino/data/user/dto/ImportUserData.java:24` (field), getter/setter `:157-162`. Dev/test fixture import DTO only (consumed by `TestUsersImportService.java:68`). Not a public API DTO.

**Any other reader?** No. The complete set of code that reads the value (`isAllowNotifications()` / `source.isAllowNotifications()`):
- `UpdateUserConverter.java:47` (User → UpdateUserDTO)
- `AuthUserConverter.java:41` (User → AuthUserDTO)
- `DefaultUserFacade.java:188` (UpdateUserDTO → User, the write)
- `TestUsersImportService.java:68` (ImportUserData → User, dev/test import)

No cron, no scheduled job, no repository query (no `@Query` references `allow_notifications`), no serializer, no notification code reads it. **Not found** anywhere outside the four readers above.

---

## Q4 — Push token endpoints

Both live on `PushTokenController` (`@RequestMapping("/api/secure/push/token")`, `PushTokenController.java:16`). Note the path prefix is `/api/secure/push/token` (the backend security-hardening move to `/api/secure/`), not a bare `/secure/...`.

**Attach** — `POST /api/secure/push/token` (`PushTokenController.java:22-26`)
→ `pushTokenService.createOrUpdatePushToken(dto, currentUserService.getCurrentUserIdStrict())`.
In `DefaultPushTokenService.createOrUpdatePushToken` (`:60-77`): looks up the row **by token value** (`findByToken`, `:70`). If absent → `createNewPushToken` inserts exactly **one** row (`:79-88`). If present → `updateExistingPushToken` re-points the existing row's `user` + `token` and saves (`:90-97`) — still one row, no insert. So attach is upsert-by-token-value: **one row inserted or one row re-owned**, never bulk. The owning `userId` is the server-derived authenticated id (Part 11), never read from the body.

**Detach** — `POST /api/secure/push/token/detach` (`PushTokenController.java:30-34`)
→ `pushTokenService.detachUserFromPushToken(dto.getPushToken(), currentUserService.getCurrentUserIdStrict())`.
In `DefaultPushTokenService.detachUserFromPushToken` (`:32-48`): `findByToken(token)` then `.filter(pt -> Objects.equals(pt.getUser().getId(), userId))` then `.ifPresent(delete)`. Deletes **exactly one** row, by token value, **scoped to the calling user** (ownership guard, `:46`). A missing or not-owned token is a safe no-op. **No cascade, no bulk delete.**

**Other deletes of push tokens (not these endpoints):**
- `DefaultFirebasePushService.java:99` deletes a single invalid token discovered during send (`deletePushToken`).
- `DefaultUserDeletionService.java:362` does `pushTokenRepository.deleteByUserId(userId)` — bulk delete of all the user's tokens, but only inside the hard-delete flow, unrelated to the toggle.

Behavior matches the brief: attach inserts/upserts one row; detach deletes one row by token value scoped to the caller. Confirmed.

---

## Q5 — If `allowNotifications` were dropped from DTO/entity/column: compile-time and consumer breakage

**Compile-time breakage** (Java), by removal target:

- **Remove getter `isAllowNotifications()` from `UpdateUserDTO`** → breaks `DefaultUserFacade.java:188` (`updateData.isAllowNotifications()`). Must delete that line.
- **Remove getter from `AuthUserDTO`** → no direct caller of `authUserDto.isAllowNotifications()` in `src/main` (the converter is the only writer via setter). Removing the field also requires deleting `AuthUserConverter.java:41`.
- **Remove the field from entity `User`** (and its getter/setter `User.java:304-310`) → breaks all four readers of `source.isAllowNotifications()` / `user.setAllowNotifications(...)`:
  - `UpdateUserConverter.java:47`
  - `AuthUserConverter.java:41`
  - `DefaultUserFacade.java:188`
  - `TestUsersImportService.java:68`
  Each line must be deleted in the same change.
- **Remove the field from `ImportUserData`** → breaks `TestUsersImportService.java:68` (`userData.isAllowNotifications()`). Delete the field (`:24`), accessors (`:157-162`), and that line together.

**Schema/seed breakage** (SQL):

- **Remove `allow_notifications` from `V1__init_schema.sql:590`** → the four admin seed inserts that name the column in their column list fail at startup (column does not exist):
  - `data-admin-dev.sql:19`, `data-admin-prod.sql:18`, `data-admin-stage.sql:18` — each names `allow_notifications` and passes `false` in VALUES.
  - All three must drop the column name **and** its corresponding `false` value from the VALUES tuple in the same change, or seeding 500s on boot.
  - (Convention reminder: V1-fold edit-in-place during pre-prod, conventions Part 12.)

**Non-toggle consumer that reads it?** **None found.** No cron/scheduled task, no `@Query`, no JPQL/native query references `allow_notifications`, no Elasticsearch/Redis serializer reads it, and — critically (Q2) — no notification dispatch code reads it. The flag has exactly four Java readers (all listed above) and zero behavioral consumers.

**Net removal set** (for the implementing brief, not done here):
- Java: `User.java` field+accessors; `UpdateUserDTO.java` field+accessors; `AuthUserDTO.java` field+accessors; `ImportUserData.java` field+accessors; the 4 reader lines (`UpdateUserConverter:47`, `AuthUserConverter:41`, `DefaultUserFacade:188`, `TestUsersImportService:68`).
- SQL: `V1__init_schema.sql:590`; column+value in `data-admin-{dev,prod,stage}.sql`.
- Tests: grep of `src/test` for `allowNotifications`/`allow_notifications` returned **zero** hits — no backend test asserts on the flag, so no test updates are forced by the removal.

---

## Cross-repo note (not actioned)

`allowNotifications` is on `AuthUserDTO` (login response) and `UpdateUserDTO` (profile GET/POST) — both are wire contracts consumed by `oglasino-web` and `oglasino-expo`. Dropping the field is a contract change those repos must absorb (read of `allowNotifications` on the auth/profile payload, and the profile-settings toggle that POSTs it back). That is web/mobile scope — flagged for routing, not audited here (per the no-cross-repo-audit rule).
