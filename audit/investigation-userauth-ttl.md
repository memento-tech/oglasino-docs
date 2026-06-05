# Investigation — redisUserAuth TTL and auth-data eviction completeness

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-14
**Type:** Read-only investigation, no code changes.

---

## TL;DR

**Verdict: safe to raise.** Every mutation of an auth-relevant field that exists in current production code routes through `DefaultUserService.saveUser`, which carries `@CacheEvict(value = "redisUserAuth", key = "#currentUser.firebaseUid")`. The eviction key matches the cache key (`#firebaseUid`). The 1-minute TTL is currently a pure safety net with nothing relying on it — the explicit `@CacheEvict` is the only thing that ever invalidates a user's `redisUserAuth` entry in production today, and that has been true for every traced auth-relevant field.

Two findings ride alongside the verdict, neither of which blocks raising the TTL:

1. **The `disabled` field is in the cached DTO but unused.** `AuthenticatedUserDTO.disabled` is fetched from the DB row, serialised into Redis, and never read anywhere on the request path. The admin "Disable user" endpoint flips the DB column and calls Firebase admin SDK to disable the Firebase account, but no in-process code path consults `authData.disabled()` to gate authentication or authorization. **Independent of the TTL question** — and would still be true with a 1-minute TTL — but it means "disable a user" today does **not** lock them out of the running app on its own. They are blocked only when their Firebase ID token expires (~1 h) and Firebase refuses to mint a new one. `verifyIdToken` is called without `checkRevoked`, so token revocation is not enforced on existing tokens either. Flagged for Mastermind; not a cache problem.

2. **Account deletion does not exist.** There is no `userRepository.delete(...)` call anywhere in the codebase. `state.md`'s backlog has "User deletion — planned, GDPR for Croatia (EU)" — so this is a known gap, not an oversight in the investigation. **When account deletion is implemented, it MUST evict `redisUserAuth` for the deleted user's `firebaseUid`.** Without that, a deleted user's stale cache entry would let `FirebaseAuthFilter` keep building an `OglasinoAuthentication` with their (now-orphaned) `userId` until TTL expiry. That risk grows with the TTL. Flag for whoever picks up the user-deletion brief.

Static tracing was self-sufficient. No app run, no SQL logging needed.

---

## Brief vs reality

No structural mismatches between the brief and the code. The brief's assumptions (`getCachedAuthData` returns `AuthenticatedUserDTO`, `redisUserAuth` is keyed by `firebaseUid`, the only programmatic eviction is on `saveUser`, the prior investigation's findings stand) all check out as written.

One observation that's worth surfacing before the questions, because it changes the framing of the answer:

- **The brief frames `redisUserAuth` as caching "auth data — identity, roles, and the user's preferred language."** True for the DTO's fields. But the only fields that any code on the request path actually *reads* from the DTO are `userId`, `userRole`, `subscriptionType`, `subscriptionActive`, `baseSiteId`, and `preferredLanguage`. The `disabled` field — which the brief calls out as the canonical example of "auth-relevant state that should invalidate the cache" — is fetched but never consulted. That's its own finding (see TL;DR finding 1), and it shapes the answer to Q4: the worst-case staleness window for a disabled user's authorization-effective state is *unbounded*, not "1 minute" or "30 minutes," because nothing reads the flag in the first place.

This does not invalidate the TTL investigation. It just means the brief's risk model ("banned, role-changed, or otherwise has their auth state revoked … keep operating on stale cached auth for that whole window") is currently shaped more by `verifyIdToken` semantics and missing `disabled`-checks than by the TTL.

---

## Question 1 — What is cached, what is auth-relevant?

`AuthenticatedUserDTO` (`src/main/java/com/memento/tech/oglasino/security/auth/AuthenticatedUserDTO.java:14-22`) is a flat record with 8 fields. Populated by the JPQL constructor projection at `UserRepository.findAuthDataByFirebaseUid` (`UserRepository.java:46-64`).

| # | Field | Type | Used on request path (where) | Security-relevant? | Notes |
|---|---|---|---|---|---|
| 1 | `userId` | `Long` | `FirebaseAuthFilter.java:71,79` → `OglasinoAuthentication.userId` + MDC `userId` | **Identity** (immutable per user) | Never changes after row creation. |
| 2 | `firebaseUid` | `String` | (cache key only; also written to `OglasinoAuthentication` from `decoded.getUid()` directly, not from the DTO) | **Identity** (immutable per user) | Never changes after row creation. The DTO field is effectively redundant. |
| 3 | `disabled` | `boolean` | **Nowhere on the request path.** Grep of `authData.disabled()` / `\.disabled\(\)` in `security/` and `filter/` returns zero hits. | **DESIGNED to be security-relevant; CURRENTLY unused.** | See TL;DR finding 1. |
| 4 | `userRole` | `UserRole` | `AuthUtils.subscriptionToAuthorities(authData)` (`AuthUtils.java:18-21`) → `SimpleGrantedAuthority(userRole.name())` → guards every `@PreAuthorize("hasRole('ADMIN')")` (`AdminController.java:13`, `UsersController.java:21`, `CacheAdminController.java:31`, +10 more) | **YES — gates all admin endpoints.** | |
| 5 | `subscriptionType` | `UserSubscription` | `AuthUtils.subscriptionToAuthorities(authData)` → `SimpleGrantedAuthority("ROLE_" + subscriptionType.name())` | **YES — produces a `ROLE_*` granted authority.** | No `@PreAuthorize` currently consults the subscription authority that I can find (only `hasRole('ADMIN')` exists), but the authority is emitted into the security context, so future tier-gated endpoints would lean on it. Today's risk is latent. |
| 6 | `subscriptionActive` | `boolean` | `AuthUtils.subscriptionToAuthorities(authData)` line 25-27 — if false (or `subscriptionType == null`), returns `List.of()` (no authorities at all) | **YES — flipping false strips ALL authorities, including `ROLE_BASIC`/`ROLE_ADMIN`.** | More load-bearing than it looks. |
| 7 | `baseSiteId` | `Long` | `FirebaseAuthFilter.java:73` → `OglasinoAuthentication.baseSiteId` | **Tenant scoping, not authz.** Read on dashboard search via `SecurityContextHolder` (per the prior product-filtering audit and `BaseSiteQueryGenerator`). | Misalignment here would let a user's dashboard scope a different `baseSite`, not let them bypass authentication. Borderline-relevant. |
| 8 | `preferredLanguage` | `LanguageDTO` | `FirebaseAuthFilter.java:74` → `OglasinoAuthentication.preferredLanguage` → consumed by `CurrentLanguageFilter` (per the prior caching investigation, Step 5 / `redisLanguage` table) | **Not security-relevant.** Locale only. | A user can override this per request via `X-Lang` anyway. |

### What "auth-relevant state" means for the rest of this investigation

Fields whose change *must* invalidate the cache to avoid a security-shaped staleness window: **`userRole`, `subscriptionType`, `subscriptionActive`, `disabled`**.

Borderline / not security-relevant: `baseSiteId` (tenant scoping), `preferredLanguage` (locale), `userId`/`firebaseUid` (immutable identity).

The four security-relevant fields are what Questions 2-3 trace.

---

## Question 2 — Every eviction site for `redisUserAuth`

`grep -rn "redisUserAuth" src/main/java` returns 9 hits. Most are documentation references (`@code`, javadoc); the only annotation- or code-driven evictions are:

| # | Site | Type | Key | Matches cache key (`#firebaseUid`)? |
|---|---|---|---|---|
| 1 | `DefaultUserService.saveUser` (`DefaultUserService.java:73`), inside the `@Caching(evict = {...})` block | `@CacheEvict(value = "redisUserAuth", key = "#currentUser.firebaseUid")` | `#currentUser.firebaseUid` (the `User.firebaseUid` of the entity being saved) | ✅ Direct match. `getCachedAuthData(firebaseUid)` keys on the input string; `saveUser` evicts using the saved entity's `firebaseUid`. The `User.firebaseUid` column has `unique = true` and is set once at row creation, so the eviction key equals the cache key for any user whose row goes through `saveUser`. |
| 2 | `RedisCacheEvictConfig.evictCachesOnStartup` (`RedisCacheEvictConfig.java:25,40-53`) | Programmatic `cache.clear()` at `ApplicationRunner` time | All entries cleared | ✅ Wholesale clear. Cache starts cold every boot. |
| 3 | `CacheAdminController.evictAll/evictOne/refresh` (`CacheAdminController.java:47-89`) | Admin-only programmatic `cache.clear()` (admin endpoint, `@PreAuthorize("hasRole('ADMIN')")`) | All entries (or one specific cache) cleared | ✅ Wholesale clear. Manual operator lever. |

**No other eviction site.** No other class carries `@CacheEvict("redisUserAuth", ...)`. No service calls `cacheManager.getCache("redisUserAuth").evict(...)` programmatically except admin/startup wholesale clears.

### Key-alignment check

- **Cache key** at the `@Cacheable` site (`DefaultFirebaseAuthService.getCachedAuthData`, `:63`): `#firebaseUid` — the method's `String` parameter. SpEL stringifies to the same key used by the Redis key serialiser.
- **Eviction key** at the `@CacheEvict` site (`DefaultUserService.saveUser`, `:73`): `#currentUser.firebaseUid` — the `User` entity's `firebaseUid` getter. SpEL stringifies the same way.
- **Identical strings** for the same user. ✅

(For completeness: `User.firebaseUid` is nullable per the entity definition — `@Column(unique = true)` with no `nullable = false`. In practice it's only blank between row construction and `setFirebaseUid` inside `DefaultFirebaseAuthService.createUserSynchronized`, both of which run before `saveUser` is called. If `firebaseUid` were ever null when `saveUser` runs, `@CacheEvict` with a null key throws — this is a latent crash, not a security issue, and is not a current bug since the only path that reaches `saveUser` always has a populated `firebaseUid`.)

---

## Question 3 — Every auth-relevant mutation path

Hunted four security-relevant fields plus the catch-alls the brief asked about. For each, traced from "what code can change this in the database" down to whether the path crosses `DefaultUserService.saveUser` (which evicts) or bypasses it via direct repository writes / dirty-checking.

### Method used

1. `grep -rn "setDisabled|setUserRole|setSubscriptionType|setSubscriptionActive|setPreferredLanguage" src/main/java` — every setter call.
2. `grep -rn "saveUser|userRepository.save|userRepository.delete|@Modifying" src/main/java` on `User` — every write entry point.
3. For each setter call, traced the enclosing method to confirm whether `saveUser` is called after the setter (eviction fires) or whether the entity is left to dirty-check (eviction would NOT fire).
4. Looked for `@Modifying` JPQL/native queries on `User` — none exist (`UserRepository.java` has only read queries).

### `userRole` — role grants/changes

Setters: 4 sites total.

| # | File:line | Path | Calls `saveUser`? | Evicts? |
|---|---|---|---|---|
| 1 | `DefaultFirebaseAuthService.java:116` | `createUserSynchronized` — sets `ROLE_ADMIN` for `admin@oglasino.com` at first-login provisioning | YES, via `userService.saveUser(newUser)` on `:121` | ✅ Evicts (but cache is necessarily empty for a brand-new user — the eviction is a no-op the first time and harmless). |
| 2 | `DefaultFirebaseAuthService.java:118` | `createUserSynchronized` — sets `ROLE_BASIC` for everyone else, first-login | YES, via `:121` | ✅ Same as above. |
| 3 | `TestUsersImportService.java:54` | Test data import | NO — direct `userRepository.save(entity)` on `:74`, bypasses `saveUser` and therefore the eviction | **Does not evict.** Profile-gated `!prod` (`TestDataImportService.java:16`). Runs once at startup, before any user has called `getCachedAuthData`, so the cache is cold. Not a production concern. |
| 4 | `User.java:150` | Bare setter, only called from the 3 sites above | n/a | n/a |

**No production code path mutates `userRole` after a user's first login.** No admin "promote to admin" endpoint exists. The "Active bug queue" in `state.md` is empty for this; the only admin user-management endpoints in `UsersController` are `disableUser` / `enableUser`.

Verdict for `userRole`: **the field is effectively write-once at provisioning.** The cache cannot become stale for `userRole` because there's no way to change it after the entry is populated.

### `subscriptionType` / `subscriptionActive` — subscription tier and activity

Setters: 3 sites for each (entity setter + 2 callers).

| # | File:line | Path | Calls `saveUser`? | Evicts? |
|---|---|---|---|---|
| 1 | `DefaultFirebaseAuthService.java:108-109` | `createUserSynchronized` — defaults new users to `SUBSCRIPTION_FREE`, `subscriptionActive=true` at first-login | YES via `:121` | ✅ (same first-login eviction; cache empty). |
| 2 | `TestUsersImportService.java:59-60` | Test data | NO — bypasses `saveUser` | **Does not evict.** `!prod` profile, startup-only. Same as `userRole` row 3. |

**No production code path mutates `subscriptionType` or `subscriptionActive` after a user's first login.** No Stripe / webhook handler, no admin "extend subscription" endpoint, no scheduled job — `grep -rn "stripe\|Stripe"` returns zero hits in `src/main/java`. The `UserDetailsConverter.setSubscriptionActive` at `:40` is a DTO-side write for a read-only admin view, not a DB mutation.

Verdict for subscription fields: **same as `userRole` — write-once at provisioning.** Even with an unbounded TTL, the cache cannot diverge from the DB for these fields because no code can flip them after the entry is populated.

### `disabled` — admin ban/unban

Setters: 4 sites total.

| # | File:line | Path | Calls `saveUser`? | Evicts? |
|---|---|---|---|---|
| 1 | `DefaultUsersFacade.java:42` | `disableUser` — admin endpoint `GET /api/secure/admin/users/disable/{targetUserId}` | YES via `userService.saveUser(user)` on `:44` | ✅ Evicts. |
| 2 | `DefaultUsersFacade.java:52` | `enableUser` — admin endpoint `GET /api/secure/admin/users/enable/{targetUserId}` | YES via `:54` | ✅ Evicts. |
| 3 | `TestUsersImportService.java:55` | Test data: always sets `disabled=false` | NO — bypasses | `!prod`, startup-only. Not a production concern. |
| 4 | `User.java:158` | Bare setter | n/a | n/a |

**Every production path that mutates `disabled` evicts.** Note the broader finding from Q1/TL;DR: the flag is never consulted on the request path, so even a perfectly-correct cache entry for this field wouldn't lock out a disabled user. But that's a separate issue from the TTL — the eviction picture for `disabled` is complete.

`DefaultUsersFacade.disableUser` also calls `firebaseUserService.disableUser(firebaseUid)` (`:46`), which calls Firebase admin SDK `UserRecord.UpdateRequest(firebaseUid).setDisabled(true)`. This disables the user on Firebase's side too — Firebase refuses to mint new ID tokens for them — but existing ID tokens remain valid for their normal ~1-hour lifetime, and `FirebaseAuthFilter` calls `verifyToken(idToken)` which is `FirebaseAuth.getInstance().verifyIdToken(idToken)` *without* `checkRevoked=true` (`DefaultFirebaseAuthService.java:39`). So a disabled user with a still-valid token can keep making authenticated requests for up to 1 hour after being disabled, regardless of the cache state.

### `baseSiteId` — borderline; tenant scoping

Setters: lots in test data + product/catalog (not User-side). On `User`, the relevant write is in `DefaultUserFacade.assignUserRegionAndCity` (`:201`), which calls `userService.saveUser(user)` on `:206`. ✅ Evicts.

The user-self update flow (`DefaultUserFacade.updateCurrentUserData`, `:124-159`) does NOT touch `baseSite`. The admin user-edit endpoints don't mutate it either (no admin-edit-user controller exists; the admin only has disable/enable).

Verdict for `baseSiteId`: **every mutation path evicts.** Not security-relevant; included for completeness.

### `preferredLanguage` — borderline; locale only

Setter sites: 2 total (entity + 1 caller).

| # | File:line | Path | Calls `saveUser`? | Evicts? |
|---|---|---|---|---|
| 1 | `DefaultFirebaseAuthService.java:110-112` | `createUserSynchronized` — first-login default | YES via `:121` | ✅ (first-login). |
| 2 | `TestUsersImportService.java:63-64` | Test data | NO — bypasses | `!prod`, startup-only. |

**No production code path mutates `preferredLanguage` after a user's first login.** Same write-once shape as `userRole`. There is `UpdateUserDTO` and `DefaultUserFacade.updateCurrentUserData`, but the field is not in the update flow.

### `getOrCreateUser` self-invocation check

Per the brief, confirm the defensive comment in `DefaultFirebaseAuthService.java:54-57` still holds and `getOrCreateUser` has not been changed to self-invoke `getCachedAuthData`. **Confirmed.** `getOrCreateUser` (`:72-90`) calls `userRepository.findByFirebaseUid` (via `getUser(token)`) directly — *not* `getCachedAuthData`. No proxy-bypass risk introduced. The provisioning path correctly skips the cache entirely; the cache populates on the *next* request once the user exists and `FirebaseAuthFilter` calls `getCachedAuthData` for the first time.

The `firebase-sync` endpoint (`AuthController.java:35`) is the public route that drives provisioning. `FirebaseAuthFilter.java:60` explicitly skips its auth path with `if (!request.getServletPath().contains("firebase-sync"))` — so `getCachedAuthData` is never called during provisioning, and there's no chicken-and-egg with a missing cache entry.

### Account deletion

`grep -rn "userRepository\.delete\|deleteUser\|deleteByFirebaseUid" src/main/java` returns zero hits. No code path deletes a `User` row.

`state.md` confirms: "User deletion — `planned`. Igor has extensive documentation. GDPR considerations for Croatia (EU)." Not implemented yet. **Out of scope for "current behavior."** But it must evict `redisUserAuth` when implemented — see TL;DR finding 2.

### Firebase-side state (token revocation, Firebase user disable)

Listed for completeness, since the brief asks: a category the local TTL **cannot** be the safety net for in either direction.

- **Firebase-side `disabled` (set by `FirebaseUserService.disableUser`):** Firebase refuses to mint *new* tokens for the disabled user, but existing tokens stay valid until expiry (~1 h). The local cache cannot see Firebase state; the local DB's `disabled` column is the in-process mirror, evicted correctly as per Q3.
- **Firebase ID token revocation / password reset / external-revocation:** `FirebaseAuthFilter.verifyToken` does not pass `checkRevoked=true` (`DefaultFirebaseAuthService.java:39`), so revoked tokens still pass verification until they expire. The local cache plays no role here — `getCachedAuthData` only runs *after* `verifyToken` succeeds.

**The local TTL cannot mitigate or aggravate either of these.** Whatever TTL is chosen, Firebase-side state changes propagate at the token-expiry boundary, not the cache-TTL boundary.

### Dirty-checking bypass check

Could a `@Transactional` method elsewhere load a `User`, mutate one of the security-relevant fields, and let Hibernate dirty-check flush the change without going through `saveUser` — bypassing `@CacheEvict`?

Searched all setter call sites for `userRole`, `subscriptionType`, `subscriptionActive`, `disabled`, `preferredLanguage`, `baseSite` on `User` (grep results in Q3 above). Every production-path call to one of these setters is followed by an explicit `userService.saveUser(user)` in the same method body. No dirty-checking-only path exists for any auth-relevant field in production code.

(One bypass exists for non-auth-relevant fields and is documented in `DefaultUserFacade.java:108-122` as a known cache-staleness corner for `redisUserInfo.shortBio` — not `redisUserAuth`. Out of this brief's scope.)

---

## Question 4 — Verdict

**Safe to raise.** Specifically:

- Every mutation path for a security-relevant field in the current production code routes through `saveUser`, which provably evicts. Verified field-by-field in Q3.
- The two cases the brief asked about specifically (role changes, ban/suspend/deactivate) are both handled: role *cannot* change after provisioning (no code path exists), ban/unban routes through `saveUser`.
- The one bypass found (`TestUsersImportService.userRepository.save`) is profile-gated `!prod` and runs at startup before any cache entries exist. No production cache-staleness window.
- No `@Modifying` queries on `User`. No direct dirty-checking-only mutation paths in production. No event listeners that mutate `User` rows.
- The `@CacheEvict` key (`#currentUser.firebaseUid`) matches the cache key (`#firebaseUid`) string-for-string.

### What "safe to raise" does and does NOT mean

It means: **raising the TTL to, say, 30 minutes or longer will not introduce a *new* staleness window for any field that any production code reads.** The TTL is doing nothing today; raising it changes nothing about cache correctness.

It does NOT mean: "disabling a user via the admin endpoint takes effect immediately." It doesn't take effect at all in-process today (TL;DR finding 1) — regardless of TTL.

### Downsides of a longer TTL beyond eviction correctness

- **Memory.** `AuthenticatedUserDTO` is small (~200-300 bytes serialised). With a 30-minute TTL and ~N active users per 30-minute window, Redis holds ~N entries instead of ~N-per-minute. Marginal at any realistic launch scale.
- **Per-user `preferredLanguage` going stale on first language change.** Not currently mutable in production code (Q3), so not a real downside today. If a "change preferred language" endpoint is added later, it will need an explicit `@CacheEvict` or to route through `saveUser` — which is exactly the same constraint that already governs every other field in the DTO.
- **Cold cache after deploy.** Same as today. Restarting clears all caches (per prior investigation); each active user pays one DB hit on their next request. Independent of TTL value.
- **The `disabled` and `verifyIdToken(checkRevoked=false)` findings.** Already present at 1 min TTL, would remain present at any TTL — they're orthogonal. But raising the TTL *would* compound them in one specific scenario: if `authData.disabled()` is eventually wired into the auth path AND the eviction on `saveUser` for some reason fails to fire (it does today), the staleness window stretches from 1 min to whatever the new TTL is. Mitigation: when `disabled` is wired in, also verify `saveUser`'s eviction path still works (it does). Not a TTL blocker.

### What the verdict assumes

- Account deletion is not implemented (verified). When it is, the implementing brief must evict `redisUserAuth` for the deleted user. (Flag in TL;DR finding 2.)
- No new code path that mutates `userRole`, `subscriptionType`, `subscriptionActive`, `disabled`, or `preferredLanguage` outside of `saveUser` lands without a corresponding eviction. The current convention (Part 11 trust boundaries, plus the existing `@CacheEvict` block on `saveUser` as the canonical example) holds the line.

---

## Question 5 — Fix shape

Not applicable — Q4 lands on "safe to raise," not "partially" or "not safe." No eviction gap to close.

The two adjacent findings (TL;DR 1 and 2) are separate-brief work, not TTL-fix work:

- **`disabled` unused:** would be its own bug-fix brief — decide whether to check `authData.disabled()` in `FirebaseAuthFilter` (cheap, local; closes the in-process loophole but token-revocation gap remains) and/or to set `verifyIdToken(idToken, true)` to enforce revocation (more expensive — adds a Firebase admin SDK round-trip per request, defeats some of the point of the cache).
- **Account deletion:** part of the planned User-deletion feature already in `state.md` backlog. Its implementing brief must include `@CacheEvict("redisUserAuth", key = "#user.firebaseUid")` on whatever deletion entry point is wired.

---

## For Mastermind (core question answered plainly)

**Would raising the redisUserAuth TTL leave any user able to operate on stale auth after their state changed?**

**No.** Every production-code mutation of an auth-relevant field on a `User` row routes through `DefaultUserService.saveUser`, whose `@CacheEvict(value = "redisUserAuth", key = "#currentUser.firebaseUid")` invalidates that user's cache entry. The eviction key matches the cache key. There is no `@Modifying` JPQL, no direct-repository write of an auth-relevant field outside `saveUser`, and no dirty-checking-only path that mutates one of these fields in production code. The 1-minute TTL has been a safety net with nothing to catch.

Two adjacent findings that are independent of the TTL but worth Mastermind attention:

1. **The `AuthenticatedUserDTO.disabled` field is fetched but never read anywhere on the request path.** Disabling a user via `GET /api/secure/admin/users/disable/{id}` flips the DB column and calls Firebase admin SDK, but no in-process code consults the flag. Combined with `verifyIdToken` being called without `checkRevoked=true`, a disabled user's existing token remains usable until it expires (~1 h). This is a real security gap unrelated to the TTL — its own bug-fix conversation.

2. **Account deletion does not exist yet** (consistent with `state.md` backlog "User deletion — planned"). When the user-deletion brief is drafted, it MUST include an eviction of `redisUserAuth` for the deleted user's `firebaseUid`. With a longer TTL, the cost of forgetting that eviction grows from "1 minute stale" to whatever the new TTL is.

Recommended next step on the TTL itself: pick a value (30 min matches `redisUserInfo`'s TTL, which is the natural symmetry argument; longer is defensible but offers diminishing returns since per-user caches are cold after every deploy anyway), and let the follow-up brief for the `disabled` / `checkRevoked` gap run independently.
