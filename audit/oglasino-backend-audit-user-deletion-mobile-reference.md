# Audit — user-deletion backend contract (mobile-adoption reference)

**Repo:** `oglasino-backend`
**Branch:** `dev`
**HEAD:** `85ed51a`
**Type:** read-only reference audit. No code changed, nothing staged, no commits.
**Method:** code is ground truth; the two reference docs (`oglasino-docs/features/user-deletion.md`, `oglasino-docs/features/user-deletion-auth-contract.md`) are the *expected*. Divergences are flagged `CODE SAYS / DOC SAYS`; none are resolved here.

---

## TL;DR for the mobile engineer

- **Delete:** `POST /api/secure/user/me/delete`, **no request body at all**, fresh-reauth Firebase token required. Success → `200 {"scheduledDeletionAt": "<ISO-8601 UTC string>"}`.
- **Errors mobile must parse on delete:** 403 `REAUTH_REQUIRED` (`reauth.required`), 403 `USER_LOCKED_FROM_DELETION` (`user.locked.from.deletion`), 403 `NOT_AUTHENTICATED` (`error.not.authenticated` via Spring 401/403 machinery if no principal).
- **Filter on PENDING_DELETION:** drops auth, request proceeds **anonymous** (C-3 landed). A pending user's authenticated calls behave exactly as if logged out. Restoration is sign-in-only via `firebase-sync`.
- **Ban:** filter returns hard **403 `USER_BANNED`** (`user.banned`). `firebase-sync` returns 403 `USER_BANNED` (`user.banned`) or 403 `EMAIL_BANNED` (`email.banned`).
- **Public profile (`/api/public/user?id=`):** banned and hard-deleted users → **404** (empty body via `ResponseEntity.ofNullable`). Pending users → 200 with `state: "PENDING_DELETION"` + `scheduledDeletionAt`.
- **All deletion/ban errors use the standard `{errors:[{field, code, translationKey}]}` envelope** → `parseServiceError` works unchanged. (One special case: the filter's `USER_BANNED` is hand-written but byte-identical to the envelope.)
- **Backend↔web-only, mobile ignores:** the `X-Revalidate-Secret` → web `/api/revalidate` cache-revalidation call fired on every user-state change.

---

## Q1 — The delete endpoint

**Controller:** `UserController.deleteMyAccount` — `controller/UserController.java:61-90`.
**Path / method:** `POST /api/secure/user/me/delete` (class `@RequestMapping("/api/secure/user")` line 28 + `@PostMapping("/me/delete")` line 61). Matches spec §10.2. ✓

**Auth requirement:** the principal `OglasinoAuthentication` is injected via `@AuthenticationPrincipal` (line 63) and must be present — the route is under `/api/secure/**`, which Spring Security requires authenticated. The endpoint additionally requires the **verified `FirebaseToken`** to be stashed on the principal's credentials slot (set by the filter at `FirebaseAuthFilter.java:109`). If credentials are null (only reachable via the dev Postman bypass) the endpoint throws `REAUTH_REQUIRED` (lines 65-70).

**Request body shape:**
> **CODE SAYS:** there is **no request body** — `deleteMyAccount` has no `@RequestBody` parameter. The method takes only the `@AuthenticationPrincipal`.
> **DOC SAYS:** spec §10.2 describes an empty `{}` body.
>
> Practical effect for mobile: send the POST with **no body** (or an empty/ignored one). Nothing in the body is read; an empty `{}` is harmless but unnecessary. Not a behavioral bug — flagged only so mobile doesn't expect a body schema.

**Success response:** `DeletionRequestResultDTO` — `dto/DeletionRequestResultDTO.java:10`:
```java
public record DeletionRequestResultDTO(Instant scheduledDeletionAt) {}
```
- Single field: **`scheduledDeletionAt`**, Java type `java.time.Instant`.
- **Serialized type:** ISO-8601 UTC string. No `spring.jackson.*` overrides exist in any `application*.yaml`, and no custom web `ObjectMapper`/`WRITE_DATES_AS_TIMESTAMPS` bean is registered (the only `JsonFormat`/mapper hits are Redis + OpenAI internals, not the web message converter). So Spring Boot's default `JavaTimeModule` with `WRITE_DATES_AS_TIMESTAMPS=false` applies → `Instant` is written as an ISO-8601 instant string ending in `Z`. The web spec corroborates: §14.10 types it `scheduledDeletionAt: string | null` (spec lines 1529, 1540).
- Derivation (`DefaultUserDeletionService.requestDeletion`, `service/impl/DefaultUserDeletionService.java:116-117`): `now + user.deletion.grace.period.days` (default 7), no truncation. Sample wire value: `{"scheduledDeletionAt":"2026-06-07T14:23:45.123456789Z"}` (fractional seconds present when `Instant.now()` carries them).

**Every error the endpoint can emit** (all 403, all via the `{errors:[{field, code, translationKey}]}` envelope from `GlobalExceptionHandler.handleUserDeletion`, `exception/GlobalExceptionHandler.java:157-165`):

| HTTP | code | translationKey | Source |
| ---- | ---- | -------------- | ------ |
| 403 | `REAUTH_REQUIRED` | `reauth.required` | `exception/ReauthRequiredException.java:24,29` — thrown on null token (line 69), missing `auth_time` (line 76), or stale `auth_time` (line 84). |
| 403 | `USER_LOCKED_FROM_DELETION` | `user.locked.from.deletion` | `exception/UserLockedFromDeletionException.java:23,28` — thrown in `requestDeletion` when `isLocked(userId)` (`DefaultUserDeletionService.java:101-103`). |
| 403 | `NOT_AUTHENTICATED` | `error.not.authenticated` (from `SystemErrorCode`) | If the principal is missing entirely, Spring Security rejects before the controller; `GlobalExceptionHandler.handleAccessDenied` (lines 47-56) emits this. |

> **CODE-vs-DOC divergence on translationKeys (REAUTH_REQUIRED, USER_LOCKED_FROM_DELETION):**
> **CODE SAYS:** `reauth.required` and `user.locked.from.deletion` (no `errors.` prefix).
> **DOC SAYS:** spec §8.8 table (`features/user-deletion.md:955,958`) and the §15 copy table (lines 1605,1608) list `errors.reauth.required` and `errors.user.locked.from.deletion`.
> Mobile must key its translations off the **code-emitted** strings (`reauth.required`, `user.locked.from.deletion`), not the spec's `errors.`-prefixed ones. (Note: the same table already shows the *corrected* `user.banned` / `email.banned` for the ban codes — only these two delete-endpoint codes still carry the stale `errors.` prefix in the doc.)

**`auth_time` freshness check — where it lives:** in the **controller**, not a filter — `UserController.java:72-85`.
- Reads `auth_time` from `token.getClaims()` where `token` is the verified `FirebaseToken` from the principal credentials (lines 65, 72).
- Config key: **`user.deletion.reauth.max.age.seconds`** via `configurationService.getRequiredIntConfig(...)` (line 81). DB-backed `Configuration`; default **300** per spec §7.
- Missing claim → treated as stale → `ReauthRequiredException` (lines 73-77). Null token → `ReauthRequiredException` (lines 66-70). Computed as `Instant.now().getEpochSecond() - authTimeSeconds > maxAgeSeconds` (lines 79-85).

---

## Q2 — FirebaseAuthFilter behavior on PENDING_DELETION (load-bearing)

**File:** `security/filter/FirebaseAuthFilter.java`.

**PENDING_DELETION branch (verbatim, lines 88-97):**
```java
// Per auth lifecycle contract C-3: restoration is a sign-in event handled exclusively
// by AuthController.firebaseSync. The filter must not restore on side-channel requests
// (e.g., SSR calls carrying a stale post-deletion cookie). Drop the auth context and
// continue anonymously; Spring's URL rules decide the response per endpoint
// (/api/public/** + /api/auth/** → anonymous, /api/secure/** → 401).
if (authData.deletionStatus() == DeletionStatus.PENDING_DELETION) {
  SecurityContextHolder.clearContext();
  filterChain.doFilter(request, response);
  return;
}
```
**C-3 landed.** The filter does **not** call `cancelDeletionOnLogin`, does **not** 403, does **not** 401. It clears the context and continues the chain anonymously. There is no remaining pre-contract restore branch in this file. ✓

**`disabled = true` branch (C-4) — lines 83-86:**
```java
if (authData.disabled()) {
  writeUserBanned(response);
  return;
}
```
`writeUserBanned` (lines 136-143) writes **HTTP 403** with hand-built body `USER_BANNED_BODY` (lines 32-34):
```json
{"errors":[{"field":null,"code":"USER_BANNED","translationKey":"user.banned"}]}
```
Hard 403 `USER_BANNED` / `user.banned`. ✓ matches C-4.

**Ordering note:** the disabled check (line 83) runs **before** the PENDING_DELETION check (line 93). So a user who is both banned and pending gets the 403 ban response from the filter — consistent with "BANNED wins" (Q4).

**Mobile implication:** a PENDING_DELETION user's authenticated requests are indistinguishable from anonymous — public endpoints serve public data, secure endpoints return 401 via Spring's own machinery (`GlobalExceptionHandler.handleAccessDenied` → 403 `NOT_AUTHENTICATED`, or a raw 401 depending on path). Restoration happens only by signing in (`firebase-sync`, Q3). Mobile must not expect any side-channel auto-restore.

---

## Q3 — firebase-sync response contract

**Handler:** `AuthController.firebaseSync` — `controller/AuthController.java:52-117`. Path `POST /api/auth/firebase-sync`.

**Success DTO — `AuthUserDTO`** (`dto/AuthUserDTO.java`), deletion/ban fields:

| Wire field | Java type | Serialized | Source |
| ---------- | --------- | ---------- | ------ |
| `disabled` | `boolean` | `true`/`false` | line 21; ModelMapper from `User.disabled`. |
| `banReason` | `String` (nullable) | string or null | line 24. Null unless disabled. (DTO javadoc: "Frontend does not display this today.") |
| `deletionStatus` | enum `DeletionStatus` | **string** `"ACTIVE"` or `"PENDING_DELETION"` | line 27; field-initialised to `ACTIVE`. |
| `scheduledDeletionAt` | `Instant` (nullable) | ISO-8601 string or null | line 30. |

**`deletionStatus` enum string values:** `entity/DeletionStatus.java` defines exactly `ACTIVE`, `PENDING_DELETION` (only two). Serialized as the constant name (no `@JsonValue`). Confirmed.

> **Note (not a divergence, but mobile-relevant):** `AuthUserDTO.scheduledDeletionAt` is mapped by ModelMapper from the `User` entity, which has **no** `scheduledDeletionAt` column — so on the `firebase-sync` response this field is effectively **always null** (AuthController comment, lines 104-105). This is benign: a PENDING user hitting `firebase-sync` is restored to `ACTIVE` in the same call (below), so a null `scheduledDeletionAt` is correct. Mobile should read `scheduledDeletionAt` from the **public-user DTO** (Q4) / the delete response (Q1), not from `firebase-sync`.

**Restoration path:** lines 94-101:
```java
boolean restored = false;
if (user.getDeletionStatus() == DeletionStatus.PENDING_DELETION) {
  userDeletionService.cancelDeletionOnLogin(user);
  restored = true;
}
```
On restore, the response header is set (lines 112-115):
```java
HttpHeaders headers = new HttpHeaders();
if (restored) {
  headers.set("X-Account-Restored", "true");
}
```
**Exact header:** `X-Account-Restored: true`. ✓ matches spec §10.1 / contract C-2, C-9. Set only when a PENDING user was restored; absent otherwise.

**Ban rejections** (both via `errorResponse(...)` → `ProductErrorResponse` envelope, lines 135-141):

| HTTP | code | translationKey (ACTUAL emitted) | Source |
| ---- | ---- | -------------------------------- | ------ |
| 403 | `EMAIL_BANNED` | **`email.banned`** | line 76 — when `userAuditService.isEmailBanned(email)` matches (pre-`getOrCreateUser`); best-effort deletes the freshly-created Firebase user first (lines 70-75). |
| 403 | `USER_BANNED` | **`user.banned`** | line 91 — when the matched/created `user.isDisabled()`. |

> **Φ1 note resolved:** the brief flagged spec §8.8 documenting these as `errors.user.banned`. In the **current** spec the §8.8 table (lines 956-957) and the §10.1 pseudo-code (lines 1075, 1097) already show the corrected `user.banned` / `email.banned`, which **match the code**. No divergence remains on the two ban codes. (The residual `errors.`-prefix divergence is only on `REAUTH_REQUIRED` / `USER_LOCKED_FROM_DELETION` — see Q1.)

---

## Q4 — UserInfoDTO.state and composition

**Public user DTO:** `dto/UserInfoDTO.java`. Wire fields:
- **`state`** — enum `UserState`, serialized as string `"ACTIVE"` / `"PENDING_DELETION"` / `"BANNED"` (field line 47, field-initialised to `ACTIVE` for cache back-compat).
- **`scheduledDeletionAt`** — `Instant`, ISO-8601 string or null (line 50; "set only when state is PENDING_DELETION").

**`UserState.resolve` algorithm (verbatim, `entity/UserState.java:20-28`):**
```java
public static UserState resolve(boolean disabled, DeletionStatus deletionStatus) {
  if (disabled) {
    return BANNED;
  }
  if (deletionStatus == DeletionStatus.PENDING_DELETION) {
    return PENDING_DELETION;
  }
  return ACTIVE;
}
```
**Composition rule confirmed: BANNED wins over PENDING_DELETION** on the wire. A user who is both disabled and pending serializes as `BANNED`. ✓ (class javadoc lines 3-9 states the same.)

**Where `state` / `scheduledDeletionAt` get populated:** there are **two** mapping paths — they are not equivalent:
1. **Projection path** (cached `redisUserInfo`) — `DefaultUserService.mapProjectionToUserInfo`, `service/impl/DefaultUserService.java:114-164`. Sets **both** `state` (line 127) and `scheduledDeletionAt` (line 128, sourced from a `UserDeletionRequest` PENDING-status subquery). Used by `getUserInfoForId` (line 45) and `getUserInfoForFirebaseUid` (line 35) → i.e. the public profile endpoint and `/api/auth/firebase/{uid}`.
2. **Entity/ModelMapper path** — `EntityUserInfoConverter.convert`, `converter/EntityUserInfoConverter.java:43-82`. Sets `state` (line 61) but **does NOT set `scheduledDeletionAt`**. Used by `DefaultUserFacade.getMyFollowings` (`facade/impl/DefaultUserFacade.java:62`).

> **CODE-vs-DOC / internal-inconsistency flag (low severity, mobile-relevant only if mobile renders the follows list badge):**
> **CODE SAYS:** in the **follows-list** response (`/api/secure/user/follows` → `getMyFollowings`), each `UserInfoDTO.state` is populated but `scheduledDeletionAt` is always **null** (the entity converter never sets it).
> **DOC SAYS:** spec treats `state` + `scheduledDeletionAt` as a pair on `UserInfoDTO`.
> Public profile and chat-user lookups (the projection path) carry both correctly. Only the follows list omits `scheduledDeletionAt`. If mobile renders only the badge (driven by `state`) this is invisible; if it shows the date in a follows list it would be missing. Not changed — flagged for the mobile chat to decide whether it matters.

**Public profile endpoint for banned / hard-deleted users:**
- Endpoint: `UserDataController.getUser` — `controller/UserDataController.java:19-24`: `return ResponseEntity.ofNullable(userFacade.getUserData(id));`. `ofNullable(null)` → **HTTP 404, empty body.**
- **Banned user → 404:** the backing query `findUserInfoById` (`repository/UserRepository.java:137-170`) has `WHERE u.id = :userId AND u.disabled = false` (line 167). A banned user returns `Optional.empty()` → `getUserData` returns null → 404. The query javadoc (lines 131-136) states this explicitly: "banned users return Optional.empty(), so `/api/public/user?id=…` yields 404 — the client cannot distinguish 'user banned' from 'user never existed.'" ✓ matches spec §3.5a.
- **Hard-deleted user → 404:** row absent → `Optional.empty()` → null → 404. ✓
- **Pending-deletion user → 200** with `state: "PENDING_DELETION"` + `scheduledDeletionAt` (the `disabled = false` filter still passes them). ✓

> **Asymmetry worth knowing:** `findUserInfoByFirebaseUid` (lines 172-205, used by `GET /api/auth/firebase/{uid}`) has **no** `disabled = false` filter — so a **banned** user IS returned there with `state: "BANNED"` (the projection javadoc, `UserInfoProjection.java:23-30`, confirms `disabled`/`BANNED` is "meaningful only on the firebase-uid projection path today"). Hard-deleted users still 404 there (row gone). So: id-path hides banned (404); firebase-uid-path surfaces banned (200 + BANNED). Mobile's chat renderer should rely on the BANNED state from the firebase-uid path and the null/404 from the id path.

---

## Q5 — Phone-number gating

**Endpoint:** `UserController.getUserPhoneNumber` — `controller/UserController.java:36-39`: `GET /api/secure/user/phoneNumber?userId=` → `userFacade.getUserPhoneNumber(userId)`.

> **CODE-vs-DOC (path):** **CODE SAYS** the route is `/api/secure/user/phoneNumber` (under the `/api/secure/user` class mapping, line 28) — i.e. it requires an authenticated caller. **DOC SAYS** spec §14.7 names `/secure/user/phoneNumber`. Same route (the `/api` prefix is implicit in the spec shorthand). No behavioral divergence; noted for path-exactness.

**The gate (server-side, in the JPQL):** `DefaultUserFacade.getUserPhoneNumber` (line 49-51) → `userService.getUserPhoneNumberIfUserAllows` (`DefaultUserService.java:59-61`) → `userRepository.getUserPhoneNumberAllowed` (`repository/UserRepository.java:79-91`):
```sql
SELECT u.phoneNumber FROM User u
WHERE u.id = :userId
  AND u.allowPhoneCalling = true
  AND u.disabled = false
  AND u.deletionStatus = com.memento.tech.oglasino.entity.DeletionStatus.ACTIVE
```
The query **nulls** the number (returns `Optional.empty()` → facade `.orElse(null)` → `200` with null body) for any of: phone-calling not allowed, **disabled (banned)**, or **PENDING_DELETION**. The gate is purely server-side and load-bearing — it does not depend on any client field. ✓ matches spec §14.7 + B2 ban-enforcement. Mobile's call-button UX gate mirrors web, but the backend gate stands on its own.

---

## Q6 — Trust boundary on the destructive op (Part 11)

Re-verified against `UserController.deleteMyAccount` (lines 61-90) and `DefaultUserDeletionService.requestDeletion` (lines 97-174).

- **User identity:** derived from the authenticated principal. `auth.getUserId()` (controller line 87) → loaded `User` via `userService.getUserById(...)`. The request carries **no body and no `userId`** — there is nothing client-supplied to spoof. ✓
- **`auth_time`:** read from the verified `FirebaseToken` stashed by the filter after `verifyToken` (`FirebaseAuthFilter.java:73,109`), not from any client field (controller lines 65, 72). A client cannot forge it without Google's signing key. ✓
- **Lock check (`USER_LOCKED_FROM_DELETION`):** `isLocked(userId)` reads `user_deletion_locks` from the DB (`requestDeletion` lines 101-103), keyed by the server-derived `userId`. Server-only data. ✓
- **Any client-trusted value in the destructive path:** none. The empty body means the only inputs are the principal-derived `userId` and the token-derived `auth_time`.

**Verdict: CLEAN.** The endpoint reads identity, freshness, and lock state entirely from the authenticated principal / verified token / DB. (The controller's own javadoc, lines 46-60, documents the same trust posture.)

---

## Q7 — Error envelope shape

Every deletion/ban error in this feature uses the unified `{errors:[{field, code, translationKey}]}` envelope, so mobile's `parseServiceError` parses them unchanged:

| Surface | Code(s) | Mechanism | File:line |
| ------- | ------- | --------- | --------- |
| delete endpoint | `REAUTH_REQUIRED`, `USER_LOCKED_FROM_DELETION` | `UserDeletionException` → `GlobalExceptionHandler.handleUserDeletion` builds a `ProductErrorResponse` | `GlobalExceptionHandler.java:157-165` |
| delete endpoint (no principal) | `NOT_AUTHENTICATED` / `ACCESS_DENIED` | `handleAccessDenied` → `ProductErrorResponse.single` | `GlobalExceptionHandler.java:47-56` |
| firebase-sync | `USER_BANNED`, `EMAIL_BANNED` | `AuthController.errorResponse` → `ProductErrorResponse` | `AuthController.java:135-141` |
| auth filter | `USER_BANNED` | **hand-written JSON constant** (filter runs before `@RestControllerAdvice`) | `FirebaseAuthFilter.java:32-34, 136-143` |

`ProductErrorResponse` (`exception/ProductErrorResponse.java:15-17`) is `record(List<FieldError> errors)` with `FieldError(String field, String code, String translationKey)` — exactly the envelope.

> **One structural note (not a deviation, but mobile should know):** the filter's `USER_BANNED` body is a **hand-built string constant**, not produced by `ProductErrorResponse`, because the servlet filter runs ahead of Spring MVC's exception handling. It is byte-for-byte identical to the envelope (`{"errors":[{"field":null,"code":"USER_BANNED","translationKey":"user.banned"}]}`), so `parseServiceError` handles it the same. The only practical consequence: if the envelope shape ever changes, this constant must be updated by hand. No deviation in the wire shape today.

No endpoint in this feature deviates from the envelope.

---

## Q8 — Backend behavior with no mobile analog

- **Web cache revalidation (`X-Revalidate-Secret` → `/api/revalidate`).** On every user-state change (deletion request, restore, ban, unban), the backend publishes `UserStateChangedEvent`; `UserStateChangedEventListener` (`listeners/UserStateChangedEventListener.java:20-23`, AFTER_COMMIT) calls `DefaultWebRevalidationService.revalidateUserCache`, which POSTs to the web revalidate endpoint (`web.revalidate.endpoint.url`) with header **`X-Revalidate-Secret`** (`service/impl/DefaultWebRevalidationService.java:21,44-45`). This exists purely to bust Next.js's server cache. **Mobile has no server cache and simply ignores this** — it is a backend→web concern.
- **`X-Account-Restored: true` response header (firebase-sync).** Mobile *can* consume this to open a "restored" dialog (the Expo `AccountStateDialogsInit` scaffold has a `restored` slot), but it is optional — the restoration itself is server-side and complete regardless. Listed here because it is a header-channel signal, not a body field.
- **The dev-only Postman bypass** (`FirebaseAuthFilter.validateForPostmanTesting`, lines 145-182) is a local-only test affordance (`dev` profile + `X-Postman` header + config flag). No mobile participation.
- **Scheduled jobs / crons, R2 deletion, hard-delete cascade** (per the brief's out-of-scope): exist in `DefaultUserDeletionService` (`executeScheduledDeletion`, Firebase reconciliation, audit purge) — **not mobile-relevant**; mobile never triggers or observes them directly.

---

## Divergence summary (CODE vs DOC)

1. **Q1 — delete request body:** CODE has no `@RequestBody` at all; DOC §10.2 says empty `{}`. Mobile sends a bodiless POST.
2. **Q1 — translationKeys:** CODE emits `reauth.required` / `user.locked.from.deletion`; DOC §8.8 + §15 tables still show `errors.reauth.required` / `errors.user.locked.from.deletion`. Mobile keys off the code strings.
3. **Q4 — follows-list `scheduledDeletionAt`:** CODE's entity-converter path (follows list) sets `state` but leaves `scheduledDeletionAt` null; the projection path (public profile / firebase-uid) sets both. DOC treats them as an always-paired field.
4. **Q3 — note (not strictly a divergence):** `AuthUserDTO.scheduledDeletionAt` is always null on `firebase-sync` (no User-side source); benign because PENDING→restored→ACTIVE in the same call.
5. **Q5 — path shorthand:** route is `/api/secure/user/phoneNumber`; DOC §14.7 shorthand `/secure/user/phoneNumber`. Same route.

The ban-code translationKeys (`user.banned` / `email.banned`) that Φ1 previously flagged are now **aligned** between code and the current spec — no divergence remains there.

---

*End of audit. No code changed; nothing staged; no commits. Branch `dev` @ `85ed51a`.*
