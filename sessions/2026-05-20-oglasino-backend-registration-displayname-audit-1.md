# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-20
**Task:** Read-only audit of the backend side of "Registration `displayName` never persists to backend" (issues.md 2026-05-19) — firebase-sync handler, `users.displayName` column, edit-profile symmetry reference, translation keys.

## Brief vs reality

I read the brief and the code before drafting any fix. Three findings I cannot proceed silently past:

1. **The brief's symmetry premise is false. The edit-profile flow does NOT persist `displayName` to the backend.**

   - Brief says (and issues.md 2026-05-19 echoes): "the edit-profile flow writes only to the backend" / "Symmetric pattern across register + update means one place to maintain" / "What validation runs — ideally exact mirror of the edit-profile flow."
   - Code says: `UpdateUserDTO.java:9` declares `@NotBlank private String displayName;`, and the controller `POST /api/secure/user/update` runs `@Valid` so the field is parsed and validated for non-blank. But `DefaultUserFacade.updateCurrentUserData` (`facade/impl/DefaultUserFacade.java:109-149`) reads `updateData.getEmail()`, `getProfileImageKey()`, `getPhoneNumber()`, `isAllowPreferenceCookies()`, `isAllowNotifications()`, `isAllowEmails()`, `isAllowPromoEmails()`, and `getShortBio()` — and never calls `user.setDisplayName(updateData.getDisplayName())`. The entity loaded by `userService.getUserById(updateData.getId())` keeps its old `displayName`, and `userService.saveUser(user)` re-persists that unchanged value. The DTO's `displayName` field is silently discarded.
   - Exhaustive grep confirms: in `src/main/java/`, `User.setDisplayName(...)` is only called from `DefaultFirebaseAuthService.createUserSynchronized` lines 103 (Firebase token name) and 115 (admin special-case), plus `TestUsersImportService.java:52` (test fixtures). No production update path exists for `displayName` today.
   - Why this matters: (a) the audit's "mirror the edit-profile validation" guidance has nothing real to mirror — the edit-profile validation for `displayName` is only `@NotBlank` and the persistence is broken; (b) any client today that submits a new `displayName` on the edit-profile form gets a 200 OK while the value is silently dropped — that is a user-facing bug separate from registration; (c) the fix for registration cannot lean on an existing pattern that does not exist.
   - Recommended resolution: Mastermind decides scope. Either fold the edit-profile fix into this brief (small — one line in the service plus the same backend validation we choose for registration), or split it out as a sibling brief. Either way, registration cannot mirror what isn't there. I have not started any implementation.

2. **There is no `displayName` translation key seeded in any locale today.**

   - Brief says: "find the error codes and the `product.*` or `ERRORS.*` translation keys" for `displayName` validation on the edit-profile path.
   - Code says: `grep -rni 'display.name\|displayname' src/main/resources/data/translations/*.sql` returns zero hits across all four locale files (EN / RS / RU / CNR). The closest is the generic `user.update.success` key (DASHBOARD_PAGES namespace). The brief's confirmation step ("the keys exist and are seeded in all four locales") cannot be performed because the keys do not exist.
   - Why this matters: the validation strategy chosen for the new registration path will need fresh `ERRORS.*` translation keys seeded across all four locales per conventions Part 6 Rule 3. There is no shortcut to "copy what edit-profile uses." Pre-launch, this is fine (the seed work is small) — I am flagging it so the Phase 4 spec calls it out explicitly rather than the engineering brief discovering it mid-session.

3. **The `firebase-sync` first-create fallback silently defaults `displayName` to `"user"`.**

   - Adjacent observation, not strictly a brief contradiction. `DefaultFirebaseAuthService.resolveDisplayName` (line 126-135) uses `token.getName()`, falls back to the email local-part, and finally to the literal string `"user"`. A token without `name` (today the case for email+password registrations, which is the surface this audit is about) and a malformed-looking email lands the user in the database with `displayName = "user"` and `NOT NULL` satisfied. Practically rare with Firebase email+password (the token's email is always populated and contains `@`), but the literal `"user"` fallback is still an unusual choice to leave silently after the new fix lands. Worth a one-liner in the spec.

Items 1 and 2 should be passed to Mastermind before I am briefed for implementation. Item 3 is a flag, not a blocker.

---

## Implemented

- (Read-only audit — no code changes.) Read the brief, conventions.md, state.md, decisions.md, issues.md, and the backend code paths named below. Produced the four required reports and the recommended-fix block.

## Files touched

- (None — read-only.)

## Tests

- (None — read-only audit.)

## Cleanup performed

- (None needed — no code touched.)

## Config-file impact

- `conventions.md`: no change.
- `decisions.md`: no change required by this audit, but two items in "For Mastermind" are candidates for a future decision entry if the edit-profile-broken finding triggers a scope expansion.
- `state.md`: no change. (If Mastermind opens a `registration-displayname` feature spec, Docs/QA will add the row at that point.)
- `issues.md`: no change required by this audit. A new entry for the edit-profile silent-drop bug is drafted in "For Mastermind" below — Docs/QA to apply if Mastermind agrees.

## Obsoleted by this session

- Nothing. (Read-only audit.)

## Conventions check

- Part 4 (cleanliness): N/A — no code edits.
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): see "For Mastermind" — three items flagged.
- Part 6 (translations): confirmed via grep that no `displayName`-keyed translations exist in any of the four locale SQL files. Captured in finding 2 above.
- Part 11 (trust boundaries): explicitly verified — `displayName` is client-supplied, does not influence moderation/authorization/state-transition, but must be validated server-side per the rule. See Report 1 below.
- Part 12 (schema): the pre-production V1 fold rule applies to any column-level changes the eventual fix introduces (e.g., adding length constraint, default, or check constraint). Audit verified the column today is `varchar(255) NOT NULL` with no default and no unique constraint. No changes proposed in this audit; flagging the rule for the engineering brief.

## Known gaps / TODOs

- None opened in this session.

---

## Report 1 — `firebase-sync` endpoint handler

**Controller.** `src/main/java/com/memento/tech/oglasino/controller/AuthController.java:51-110` — `POST /api/auth/firebase-sync`. Body parameter:

```java
@PostMapping("/firebase-sync")
public ResponseEntity<?> firebaseSync(
    @RequestBody @Valid LoginRequest request, HttpServletRequest servletRequest)
    throws FirebaseAuthException { ... }
```

**Request DTO — `LoginRequest.java`** (complete file, single field):

```java
public class LoginRequest {
  private boolean allowPreferenceCookies;

  public boolean isAllowPreferenceCookies() { return allowPreferenceCookies; }
  public void setAllowPreferenceCookies(boolean allowPreferenceCookies) {
    this.allowPreferenceCookies = allowPreferenceCookies;
  }
}
```

Confirmed: `displayName` is not on the DTO today. There is no shadow field, no `@JsonProperty`, no setter for it anywhere on the class.

**Service path.** Controller calls `firebaseAuthService.getOrCreateUser(request, servletRequest)` at `AuthController.java:78`, which routes into `DefaultFirebaseAuthService.getOrCreateUser` at `security/service/impl/DefaultFirebaseAuthService.java:72-90`:

```java
@Override
public User getOrCreateUser(LoginRequest loginRequest, HttpServletRequest request)
    throws FirebaseAuthException {
  var resolvedToken = resolveFirebaseToken(request);
  var token = this.verifyToken(resolvedToken);
  var claims = token.getClaims();
  var providerId =
      Optional.ofNullable(claims.get("firebase")).map(Object::toString).orElse(StringUtils.EMPTY);

  var user =
      getUser(token).orElseGet(() -> createUserSynchronized(token, loginRequest, providerId));

  if (StringUtils.isBlank(user.getRegisteredWithProvider())
      && StringUtils.isNoneBlank(resolvedToken)) {
    user.setRegisteredWithProvider(providerId);
    userService.saveUser(user);
  }

  return user;
}
```

If the user already exists, the method does not touch `displayName`. The only post-create back-fill path is for `registeredWithProvider`.

**First-create branch — `createUserSynchronized`** (`DefaultFirebaseAuthService.java:92-124`):

```java
private User createUserSynchronized(
    FirebaseToken token, LoginRequest loginRequest, String providerId) {
  synchronized (token.getUid().intern()) {
    return getUser(token)
        .orElseGet(() -> {
          User newUser = new User();
          newUser.setFirebaseUid(token.getUid());
          newUser.setEmail(token.getEmail());
          newUser.setDisplayName(resolveDisplayName(token));
          newUser.setProfileImageKey(token.getPicture());
          newUser.setEmailVerifiedExternal(token.isEmailVerified());
          newUser.setRegisteredWithProvider(providerId);
          newUser.setAllowPreferenceCookies(loginRequest.isAllowPreferenceCookies());
          newUser.setSubscriptionType(UserSubscription.SUBSCRIPTION_FREE);
          newUser.setSubscriptionActive(true);
          newUser.setPreferredLanguage(
              entityManager.getReference(Language.class, languageContext.getCurrentLanguageId()));

          if (token.getEmail().equals("admin@oglasino.com")) {
            newUser.setDisplayName("Admin User");
            newUser.setUserRole(UserRole.ROLE_ADMIN);
          } else {
            newUser.setUserRole(UserRole.ROLE_BASIC);
          }

          return userService.saveUser(newUser);
        });
  }
}
```

`resolveDisplayName(token)` is the source of the value:

```java
private String resolveDisplayName(FirebaseToken token) {
  return Optional.ofNullable(token.getName())
      .filter(StringUtils::isNoneBlank)
      .orElseGet(() ->
          Optional.ofNullable(token.getEmail())
              .filter(email -> email.contains("@"))
              .map(email -> email.substring(0, email.indexOf("@")))
              .orElse("user"));
}
```

For email+password registrations (the surface this audit targets), Firebase Auth's `token.getName()` is null because the web `createUserWithEmailAndPassword(email, password)` call does not write a displayName. So the effective default for a fresh registration today is **the email local-part** (e.g. `oglasino` for `oglasino@gmail.com`).

**Idempotency.** Strictly first-create on the displayName surface. Subsequent `firebase-sync` calls hit the `getUser(token).orElseGet(...)` short-circuit — the existing user is returned unchanged. Only `registeredWithProvider` may be back-filled. If the fix carries `displayName` on every call (including re-logins), the handler will need an explicit decision: "update on every sync" (idempotent overwrite from client) vs. "create-only" (registration writes, login does not). My recommendation is below.

---

## Report 2 — `users` row, `displayName` column

**JPA entity** — `src/main/java/com/memento/tech/oglasino/entity/User.java:37-38`:

```java
@Column(nullable = false)
private String displayName;
```

No `@Size`, no `@Pattern`, no `length = N` (Hibernate defaults to 255 for String columns). The column-level annotation on the entity does not declare uniqueness, and `@Table.indexes` (lines 21-29) lists only `email`, `firebaseUid`, `base_site_id`, `region_id`, `city_id` — no index on `display_name`.

**Database column** — `src/main/resources/db/migration/V1__init_schema.sql:608` (inside `CREATE TABLE public.users`):

```sql
display_name character varying(255) NOT NULL,
```

No `DEFAULT`. No `UNIQUE`. No `CHECK`. Verified by grepping the whole V1 file for `display_name` — single hit on line 608.

**Unique constraint?** None today. Oglasino does not require unique display names; the natural primary key for a user is `firebase_uid` (unique) and `email` (unique). Two users with `displayName = "Marko"` is permitted. Confirmed by the absence of any `UNIQUE` constraint on the column and the absence of a uniqueness index in the `@Table` annotation.

---

## Report 3 — Existing `displayName` write paths

**Production paths that call `User.setDisplayName(...)`** — exhaustive grep on `src/main/java`:

1. `security/service/impl/DefaultFirebaseAuthService.java:103` — first-create branch, value from `resolveDisplayName(token)`.
2. `security/service/impl/DefaultFirebaseAuthService.java:115` — admin special-case, hardcoded `"Admin User"`.

That is the complete production list. No update path. No admin update path. No batch path.

**Edit-profile endpoint.**

- Controller: `controller/UserUpdateController.java:18-50` — `@RequestMapping("/api/secure/user/update")` → `@PostMapping public ... updateUserData(@RequestBody @Valid UpdateUserDTO updateData)`.
- DTO: `dto/UpdateUserDTO.java` — `displayName` is field 3, annotated `@NotBlank` (line 9). No `@Size`, no `@Pattern`.
- Service: `facade/impl/DefaultUserFacade.updateCurrentUserData` (`facade/impl/DefaultUserFacade.java:109-149`). Quoting the relevant block:

```java
@Override
public boolean updateCurrentUserData(UpdateUserDTO updateData) {
  var currentUserId = currentUserService.getCurrentUserIdStrict();
  var user = userService.getUserById(updateData.getId()).orElseThrow();

  if (!currentUserId.equals(user.getId())) {
    return false;
  }

  if (StringUtils.isBlank(user.getRegisteredWithProvider())) {
    user.setEmail(updateData.getEmail());
  }

  if (Objects.nonNull(user.getProfileImageKey())
      && !user.getProfileImageKey().equals(updateData.getProfileImageKey())) {
    imageService.deleteImageAndClearOwnership(user.getProfileImageKey());
  }

  user.setProfileImageKey(updateData.getProfileImageKey());
  user.setPhoneNumber(updateData.getPhoneNumber());
  user.setAllowPreferenceCookies(updateData.isAllowPreferenceCookies());
  user.setAllowNotifications(updateData.isAllowNotifications());
  user.setAllowEmails(updateData.isAllowEmails());
  user.setAllowPromoEmails(updateData.isAllowPromoEmails());

  userService.saveUser(user);

  userTranslationsService.generateUserTranslations(user, updateData.getShortBio());

  userService.evictUserInfoCache(user);

  return true;
}
```

No `user.setDisplayName(updateData.getDisplayName())` line. **The DTO's `displayName` is parsed, `@NotBlank`-validated, then silently dropped.** This is the brief-vs-reality finding 1 above.

**Admin-side edit path?** `admin/controller/UsersController.java` (full read confirms) has no displayName edit endpoint. Admin paths on this controller are limited to ban/unban/lock-deletion/unlock-deletion/state-info/force-delete. There is no admin "edit user profile" surface anywhere in `src/main/java/com/memento/tech/oglasino/admin/`.

**Cache eviction.** `userService.saveUser` is annotated with `@Caching(evict = { ... })` in `service/impl/DefaultUserService.java:64-75`:

```java
@Caching(
    evict = {
      @CacheEvict(value = "redisUserInfo", key = "#currentUser.id"),
      @CacheEvict(value = "redisUserInfo", key = "#currentUser.firebaseUid"),
      @CacheEvict(value = "redisUserAuth", key = "#currentUser.firebaseUid")
    })
public User saveUser(User currentUser) { ... }
```

Plus the explicit `evictUserInfoCache(user)` at line 146 of the facade (post-translations write — pre-2026-05-17 fix was an ordering bug). Net: every `saveUser` evicts `redisUserInfo` (both id-keyed and firebaseUid-keyed entries) plus `redisUserAuth` (firebaseUid-keyed). Any future `displayName` write that flows through `userService.saveUser` inherits this plumbing for free. The `redisUserInfo` cache holds `UserInfoDTO`, whose `displayName` field is populated from `User.displayName` via the `EntityUserInfoConverter` (line 55) and `mapProjectionToUserInfo` (`DefaultUserService.java:119`). So evicting `redisUserInfo` is exactly the right surface for a displayName write.

**Note on `redisUserAuth`.** The `redisUserAuth` cache holds `AuthenticatedUserDTO`, which carries `userId`, `userRole`, `subscriptionType`, `subscriptionActive`, `baseSiteId`, `preferredLanguage` — **not** `displayName` (the trust-decision fields per Part 11, not the display payload). Evicting it on a displayName-only write is over-eviction but cheap and safe; the `saveUser` annotation already does this and we should not split it.

---

## Report 4 — Translation keys for `displayName` validation

**Grep result.** `grep -rni 'display.name\|displayname' src/main/resources/db/migration/ src/main/resources/data/translations/*.sql` — zero hits in any translation file across EN, RS, RU, CNR. The only `display_name` hit is the schema column at `V1__init_schema.sql:608`.

**Consequence.** No `product.<field>.<code>` mapping (per conventions Part 6 Rule 4) exists for `displayName` today on either the register path (no validation) or the edit-profile path (`@NotBlank`-only — the Jakarta default constraint failure does not currently surface as a coded `ERRORS.*` translation key, and Mastermind/Igor have not approved any). The new keys we need will be greenfield additions in the `ERRORS` namespace.

**Note on namespace.** Conventions Part 6 Rule 1 freezes the `VALIDATION` namespace — new validation keys go to `ERRORS`. Any `displayName.<code>` key the fix introduces should be seeded under `ERRORS`.

---

## Recommended fix (call to propose)

Subject to Mastermind's resolution of brief-vs-reality findings 1 and 2 above. With the caveat that mirroring the edit-profile path is not possible because the edit-profile path is broken and unvalidated, here is the shape I would implement if briefed:

**Fix shape (backend side).**

1. Add `displayName` to `LoginRequest.java` as `private String displayName;` with the same validation we choose for the canonical input shape (see point 3 below). Per Part 11, the field is client-supplied input; no trust decision reads from it.
2. Plumb `LoginRequest.displayName` into `DefaultFirebaseAuthService.getOrCreateUser`. Two decisions, both for Mastermind:
   - **First-create only vs. update-on-every-sync.** First-create is the cleaner default — Firebase Auth re-syncs (subsequent logins) should not overwrite a displayName that the user may have edited via the profile flow later. Recommended: first-create only on this endpoint.
   - **Empty/blank handling.** If the client sends a blank `displayName` (unlikely after frontend validation, but Part 11 still applies), the server must reject or fall back. Recommendation: validate `@NotBlank` (and length/pattern per point 3) on `LoginRequest`, so a blank client value produces a 400 before the service runs. The frontend's Observation-1 patch (issues.md 2026-05-19) makes the client the only source of `displayName` post-fix — failing closed on a missing field is safer than the current `resolveDisplayName(token)` fallback to `"user"` or email-local-part.
3. **Validation.** Mirror the spirit of product-field validation, scaled down (`displayName` is not subject to spam/banned-words moderation at registration today, per the brief's out-of-scope note on moderation lifecycle):
   - `@NotBlank` (Jakarta).
   - `@Size(min = 2, max = 60)` — DB cap is 255, but a 2-60 range is the sane UX cap for a display name across all four locales. Mastermind to confirm the upper bound.
   - Optionally `@Pattern` for character class — Mastermind to decide if we restrict (e.g. letters/digits/space/`-`/`.`/`'`) or accept all printable. The brief's "ideally mirror the edit-profile flow" gives no answer because edit-profile doesn't mirror anything.
   - Translation keys: seed `ERRORS.displayName.required`, `ERRORS.displayName.size`, `ERRORS.displayName.pattern` (or whichever subset the validation chooses) across EN / RS / RU / CNR per Part 6 Rule 3 — append to the existing 0001/0002 files at the end of the `ERRORS` namespace block, take next IDs, alphabetical within group.
4. **Persistence.** Inside `createUserSynchronized`, prefer `loginRequest.getDisplayName()` over `resolveDisplayName(token)` when non-blank. Keep the admin-special-case override at line 115 unchanged. This keeps a single defensive fallback path for non-registration entries (legacy data, edge tokens) without breaking the new contract.
5. **Cache plumbing.** First-create runs through `userService.saveUser`, which already evicts `redisUserInfo` + `redisUserAuth`. No new cache eviction wiring needed. (`redisUserInfo` evict matters; `redisUserAuth` evict is over-eviction but harmless.)
6. **Edit-profile parity (proposed scope expansion — Mastermind to confirm).** Add `user.setDisplayName(updateData.getDisplayName())` to `DefaultUserFacade.updateCurrentUserData` and apply the same validation rules to `UpdateUserDTO.displayName` (replace bare `@NotBlank` with the `@Size` + `@Pattern` set). Without this, register and edit will not be symmetric and the spec's "Symmetric pattern across register + update" claim cannot be true even after the registration fix. Logged as draft `issues.md` entry below.

**Trust-boundary check (Part 11).** `displayName` is client-supplied input. It does not influence moderation, authorization, or state-transition decisions. The backend persists it as display content only. Validation runs server-side per Part 11 — the new Jakarta constraints on `LoginRequest.displayName` are the trust boundary, identical in posture to how `LoginRequest.allowPreferenceCookies` works today. No claim is made about the value beyond format/length; it is not compared to a server-side "old value" (no change-detection here, this is first-create).

**Backwards compatibility.** Pre-launch, no production data. Dev/local DBs may have `displayName="user"` or email-local-part values from prior firebase-sync calls (e.g. Igor's local fixtures). No backfill needed; existing rows are valid (`NOT NULL` is satisfied) and the next time the affected user hits the edit-profile flow, the new value lands cleanly **provided point 6 is also done**. Without point 6, those rows are stuck because edit-profile silently drops the field. This is the strongest argument for folding point 6 into the same brief.

**Open questions (for Igor / Mastermind).**

1. Resolve brief-vs-reality finding 1: is edit-profile parity in scope for the same brief, or split out?
2. Validation pattern: `@Size(2, 60)` range OK? Pattern restriction yes/no? If yes, which characters?
3. On idempotency: confirm "first-create only" on `firebase-sync`. Alternative is "client may always send current value, backend persists if non-blank" — slightly more code, slightly safer if the frontend ever loses sync.
4. Should the `resolveDisplayName(token)` fallback to `"user"` / email-local-part stay as a defensive path after the fix? My recommendation: leave it in place for any non-registration entry that lacks a body-supplied `displayName`, but the new registration path makes the field required, so the fallback is no longer reachable from the web form. Keep for safety.

---

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit).
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **Brief-vs-reality findings (priority — please resolve before issuing the engineering brief):**

  1. **Edit-profile silently drops `displayName`.** `DefaultUserFacade.updateCurrentUserData` validates the field via `@NotBlank` on the DTO but never writes it to the entity. The brief and issues.md 2026-05-19 both refer to the edit-profile flow as the symmetric reference for "writes only to the backend" — that claim is wrong as currently coded. The engineering brief cannot meaningfully say "mirror the edit-profile validation" because the edit-profile path has only `@NotBlank` and discards the value. Two scope options: (a) fold the edit-profile fix into the same brief (one-line service edit plus the same validation set we choose for register), or (b) split it out. Recommend (a) — same surface, same validation, same translation keys.
  2. **No `displayName` translation keys exist today.** Greenfield seed work across EN / RS / RU / CNR is required per conventions Part 6 Rule 3. Not a blocker, just flagged so the spec carries it explicitly.

- **Adjacent observations (Part 4b):**
  - **Low.** `DefaultFirebaseAuthService.resolveDisplayName` silently falls back to the literal string `"user"` for tokens without `name` and without `@` in the email. Pre-launch and rare with email+password flows, but unusual. File: `src/main/java/com/memento/tech/oglasino/security/service/impl/DefaultFirebaseAuthService.java:126-135`. I did not fix this because it is out of scope and the registration fix will short-circuit this branch for the web surface anyway.
  - **Medium.** `User.displayName` is `@Column(nullable = false)` on the entity but has no `length = N` annotation, no `@Size`, no `@Pattern`. The DB column is `varchar(255) NOT NULL`. Effective max length is 255 characters at the boundary, which is far past any sane display name. Worth pinning to a sensible cap (e.g. 60) on both layers when the fix lands. File: `src/main/java/com/memento/tech/oglasino/entity/User.java:37-38` and `src/main/resources/db/migration/V1__init_schema.sql:608` (pre-production V1 fold per Part 12 — edit in place if Mastermind agrees on the cap).
  - **Medium.** Same observation as above for `User.email`, `User.phoneNumber`, and other free-text fields — but firmly out of scope here; flagging only that the displayName fix is a natural opportunity to set the precedent.

- **Drafted `issues.md` entry — Docs/QA to apply if Mastermind agrees with finding 1:**

  ````markdown
  ## 2026-05-20 — Edit-profile `displayName` change is silently dropped server-side

  **Severity:** medium
  **Status:** open
  **Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/facade/impl/DefaultUserFacade.java:109-149` (`updateCurrentUserData`); `oglasino-backend/src/main/java/com/memento/tech/oglasino/dto/UpdateUserDTO.java:9`.
  **Detail:** `UpdateUserDTO` declares `@NotBlank private String displayName;`, the controller `POST /api/secure/user/update` parses and validates it via `@Valid`, but `DefaultUserFacade.updateCurrentUserData` never copies the value onto the loaded `User` entity. The DTO field is silently discarded; `userService.saveUser(user)` re-persists the unchanged old value. Any user editing their displayName via the profile flow gets a 200 OK while the server keeps the original. Discovered during the 2026-05-20 backend audit for the registration `displayName` persistence fix; the registration fix's "mirror the edit-profile flow" guidance has nothing real to mirror because of this gap.

  **Fix scope:** add `user.setDisplayName(updateData.getDisplayName())` in the facade method, alongside the same Jakarta validation set chosen for the new registration path (likely `@NotBlank` + `@Size(2, 60)` + optional `@Pattern`). Cache plumbing already in place via `userService.saveUser`'s existing `@CacheEvict` set. Likely folded into the registration brief rather than landing as a separate session — Mastermind to decide.
  ````

- **Drafted entry — none needed for `decisions.md` until Mastermind sets validation rules and scope.**

- **Closure gate confirmation:** drafted `issues.md` text is in this "For Mastermind" section; no other config-file edits are required by this audit. Confirmed no pending implicit config-file dependency.
