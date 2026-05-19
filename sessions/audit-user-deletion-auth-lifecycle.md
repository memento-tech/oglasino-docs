# Audit — User Deletion Auth Lifecycle Contract

**Date:** 2026-05-18
**Repo:** `oglasino-backend`
**Branch read:** `dev` (the brief named `feature/user-deletion`; per CLAUDE.md hard rule I did not switch — all user-deletion files are present in the working tree as modified/untracked entries off `dev`. Flagged in the session summary.)
**Type:** read-only audit
**Source contract:** [`oglasino-docs/features/user-deletion-auth-contract.md`](../../oglasino-docs/features/user-deletion-auth-contract.md) (draft)
**Source spec:** [`oglasino-docs/features/user-deletion.md`](../../oglasino-docs/features/user-deletion.md) §3.5, §4.4, §10, §10.1, §10.2

---

## 1. Summary

The audit confirms the contract's central claims. The architectural move proposed in clause C-3 — `FirebaseAuthFilter` stops mutating state on `PENDING_DELETION` and instead drops the auth context so the request continues anonymously — is implementable as drafted, with no surprises in the surrounding code. Specifically:

- Spring Security's authorization layer already returns 401 on `/api/secure/**` when no `Authentication` is set, and serves `/api/public/**` cleanly to anonymous principals, so dropping the auth context is a clean primitive (Q-1).
- The filter is path-aware only for `firebase-sync`; `/api/public/**` is **not** exempted, so the race in `user-deletion-auth-contract.md` §3 is fully explained by the on-disk behaviour, and C-3 fixes it at the right layer (Q-2).
- `requestDeletion` routes its `deletion_status` flip through `userService.saveUser`, which evicts both `redisUserAuth` and `redisUserInfo` via `@CacheEvict`, so the cache is structurally consistent with C-3's assumption that the next request reads fresh state (Q-3).
- `AuthController.firebaseSync` is structurally self-contained — it does its own `verifyToken` and is filter-exempt — so removing the filter's `cancelDeletionOnLogin` call does not orphan any code path (Q-4, Q-5).
- The required test changes are localised: one filter test must be deleted, two must be rewritten, and one new "PENDING_DELETION falls through with no auth context" test must be added. The `firebase-sync` and `DefaultUserDeletionService` test suites are untouched by C-3 (Q-6).

No assumption in the contract turned out to be false. Two **adjacent observations** are flagged at the end of this document for Mastermind's awareness; neither blocks C-3.

---

## 2. Q-1 — Filter behavior when `SecurityContextHolder` is not set

### Walk-through, by endpoint

`SecurityConfig.filterChain` (SecurityConfig.java:53-88) is the authoritative chain. The relevant rules:

```java
auth.requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()       // line 75-76
    .requestMatchers("/api/public/**", "/api/auth/**", "/internal/**").permitAll()  // line 77-78
    .requestMatchers("/api/secure/**").authenticated()             // line 79-80
    .anyRequest().permitAll();                                     // line 81-82
```

`@EnableMethodSecurity` is on the class (SecurityConfig.java:23) — `@PreAuthorize` annotations are evaluated as a separate (method-level) layer on top of the URL rules.

**1. `POST /api/public/product/search` from a PENDING_DELETION user, filter drops auth context.**

`ProductSearchController.java:25` maps `/api/public/product/search`; the `POST /` handler is at line 57-58. The URL rule `permitAll()` (SecurityConfig.java:77) governs. No `@PreAuthorize` on `ProductSearchController` (confirmed by grep — no hits in that file). With `SecurityContextHolder` empty, Spring's `AuthorizationFilter` sees `permitAll` and lets the request through. Expected: **200 with public results.** Confirmed by inspection.

**2. `POST /api/secure/user/me/delete` from a PENDING_DELETION user, filter drops auth context.**

`UserController.java:28` is annotated `@RequestMapping("/api/secure/user")`; the `@PostMapping("/me/delete")` handler is at line 61-63. URL rule `authenticated()` (SecurityConfig.java:79-80) governs. With no `Authentication` set, the `authenticated()` matcher rejects via `AuthorizationFilter` → `AccessDeniedException` → Spring's default exception translation. The default `AuthenticationEntryPoint` for an `HttpBasic`-disabled, `formLogin`-disabled, stateless chain (SecurityConfig.java:68-72) is `Http403ForbiddenEntryPoint` for `AccessDeniedException` from an unauthenticated principal — i.e. Spring returns **401** when no principal exists at all (no `AnonymousAuthenticationToken` is set either, since this chain configures none explicitly, but Spring's default anonymous-authentication is enabled unless `.anonymous().disable()` is called; it is not). The practical effect: anonymous `AnonymousAuthenticationToken` is set on the security context by Spring's own `AnonymousAuthenticationFilter` if our filter leaves it empty, so `authenticated()` returns false → 401 via `Http403ForbiddenEntryPoint` (which despite the name returns 401 for unauthenticated, 403 for authenticated-but-denied). Net: **401 from the secure-endpoint authorization layer.** Confirmed implementable.

**3. `POST /api/secure/admin/users/{userId}/force-delete` from a PENDING_DELETION user, filter drops auth context.**

`UsersController.java:24-25` is `@RequestMapping("/api/secure/admin/users")` + class-level `@PreAuthorize("hasRole('ADMIN')")`. The `@PostMapping("/{targetUserId}/force-delete")` handler is at lines 69-81. Two layers reject: the URL rule `/api/secure/** authenticated()` rejects first → 401, identical to (2). If, hypothetically, the URL rule were `permitAll` and only the `@PreAuthorize` were the gate, the method-level check on an `AnonymousAuthenticationToken` would also fail → 403 via Spring's default `AccessDeniedHandler`. Either way: **401 or 403.** Confirmed.

### 4. Can the filter skip `setAuthentication` without breaking anything?

Yes — and the current code already does so on multiple paths:

- **Blank token path** (FirebaseAuthFilter.java:79): `if (!request.getServletPath().contains("firebase-sync") && StringUtils.isNotBlank(idToken))` — when the token is blank, the entire `try` block (including `setAuthentication` at line 118) is skipped. The filter falls through to `filterChain.doFilter` (line 132) with no authentication set. This is what happens today for every anonymous request to a public endpoint, and it works correctly.
- **Exception path** (FirebaseAuthFilter.java:122-129): on any failure (`verifyToken`, `getCachedAuthData`, `orElseThrow`), `SecurityContextHolder.clearContext()` is called and **only** a 401 is returned when the path is **not** under `/api/public`. For public paths the filter falls through with cleared context. This is structurally the same as the C-3 proposal — clear/leave-empty and continue. The pattern is already exercised and tested.

No downstream invariant assumes the filter has populated `SecurityContextHolder` for every request. The chain has `RateLimitFilter` (SecurityConfig.java:85, added after `FirebaseAuthFilter`), which inspects the request but does not require authentication to be set. The controller layer uses `@AuthenticationPrincipal OglasinoAuthentication auth` (e.g. `UserController.deleteMyAccount` at line 62-63); on `/api/secure/**` paths, the request never reaches the controller without authentication because the URL-level `authenticated()` rule short-circuits.

### Verdict (Q-1)

**C-3 is implementable as drafted.** The filter can leave `SecurityContextHolder` unset for `PENDING_DELETION` users; Spring's `AnonymousAuthenticationFilter` will install an `AnonymousAuthenticationToken`, after which the `/api/secure/**` `authenticated()` rule returns 401 and the `permitAll` rules let `/api/public/**` and `/api/auth/**` through. No additional principal needs to be constructed; the existing chain handles "request without our auth context" cleanly. The pattern is precedented in the filter's own exception branch (FirebaseAuthFilter.java:122-129).

---

## 3. Q-2 — Filter exemption list and short-circuit predicate

The filter is registered unconditionally on the chain via `HttpSecurity.addFilterBefore(firebaseAuthFilter, UsernamePasswordAuthenticationFilter.class)` (SecurityConfig.java:84). It is a `@Bean` produced by `ApplicationConfig.firebaseAuthFilter` (ApplicationConfig.java:24-39). There is **no** `FilterRegistrationBean` and **no** URL-pattern restriction at registration. The filter therefore runs on every request the security chain processes — which is every request the application serves.

Inside `doFilterInternal` (FirebaseAuthFilter.java:64-133), the exemptions and short-circuits are:

### 1. OPTIONS preflight — full skip

```java
if (HttpMethod.OPTIONS.matches(request.getMethod())) {
  filterChain.doFilter(request, response);
  return;
}
```

(FirebaseAuthFilter.java:68-71.) **Exemption type:** skips filter entirely. No token resolution, no verification, no cache lookup, no `SecurityContextHolder` mutation.

### 2. Postman dev bypass — sets auth from `X-User` header, then skips chain

```java
if (validateForPostmanTesting(request, response, filterChain)) {
  return;
}
```

(FirebaseAuthFilter.java:73-75, with the helper at lines 144-181.) Active only when `spring.profiles.active = dev` AND header `X-Postman: true` AND `postman.testing.active` config flag is true. When all three are true, the filter sets a fake `OglasinoAuthentication` directly from the DB row (no Firebase verification) and short-circuits the rest of the filter. **Exemption type:** verifies via DB only, no Firebase verification. Note: the bypass does **not** stash a verified `FirebaseToken` (`setCredentials` is not called in `validateForPostmanTesting`), which means controllers reading `auth.getCredentials()` see `null` — `UserController.deleteMyAccount` correctly throws `ReauthRequiredException` in that case (see its Javadoc, UserController.java:54-59).

### 3. `firebase-sync` substring match — skip token verification, continue chain

```java
if (!request.getServletPath().contains("firebase-sync") && StringUtils.isNotBlank(idToken)) {
  ...
}
filterChain.doFilter(request, response);
```

(FirebaseAuthFilter.java:79-132.) The `if` predicate has two conditions joined by `&&`. If either is false, the entire token-verification block is skipped and the filter falls through to `filterChain.doFilter`.

- **Sub-case A: path contains `firebase-sync`.** Whether the token is present or not, no verification runs. **Exemption type:** verifies optionally → no, actually skips verification entirely. The `AuthController.firebaseSync` handler runs its own `verifyToken` (AuthController.java:61), so this exemption is the right shape: the filter steps aside, the handler does its own auth check.
- **Sub-case B: token is blank.** Whether the path is `firebase-sync` or anything else, no verification runs. **Exemption type:** verifies optionally if present. For a request with no `Authorization` header, `resolveFirebaseToken` (DefaultFirebaseAuthService.java:138-150) returns `null`, `StringUtils.isNotBlank(null)` is false → entire block skipped → request continues anonymous.

### 4. `/api/public/**` — NOT exempted

There is **no** path-prefix exemption for `/api/public/**` in the filter. If a public endpoint is called with an `Authorization: Bearer <token>` header (the exact case in the 2026-05-18 race per the contract §3 timeline), the filter runs the full verify + cache lookup + ban check + PENDING_DELETION branch. This is the on-disk explanation for why the SSR call to `/api/public/product/search` triggered `cancelDeletionOnLogin` even though the endpoint is public. **Confirmed:** the filter verifies tokens on `/api/public/**` when present.

### 5. Exception fall-through — clear context, 401 only off public

```java
} catch (Exception e) {
  SecurityContextHolder.clearContext();
  if (!request.getServletPath().startsWith("/api/public")) {
    response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
    return;
  }
}
```

(FirebaseAuthFilter.java:122-129.) On any exception — `verifyToken` fail, cache lookup `orElseThrow`, or anything else inside the `try` — context is cleared. For `/api/public/**`, the filter falls through with no auth context. For other paths, 401 + return. This is the **architectural precedent for C-3**: the filter already knows how to "drop context and let public endpoints serve anonymous." C-3 simply extends that pattern from the exception branch to the new PENDING_DELETION branch.

### What happens with no `Authorization` header

Per Sub-case 3B above: the entire token-verification block is skipped via the `StringUtils.isNotBlank(idToken)` guard. The filter falls through to `filterChain.doFilter` cleanly with `SecurityContextHolder` untouched. **Pass-through, no short-circuit.** This is how every anonymous request to a public endpoint succeeds today.

### Verdict (Q-2)

The exemption picture is exactly as `user-deletion-auth-contract.md` §7 Q-5 predicted: only `firebase-sync` short-circuits at the filter level; `/api/public/**` is not exempt and the filter runs on it whenever a token is present. C-3 is implementable as a one-clause amendment to the existing verify-and-load block — specifically removing lines 100-104 (the PENDING_DELETION branch) and replacing them with an early `filterChain.doFilter` + `return`, leaving `SecurityContextHolder` untouched and mirroring the existing exception-branch precedent at lines 122-129.

---

## 4. Q-3 — `redisUserAuth` cache eviction on `requestDeletion`

### Eviction trace

**1. `requestDeletion` routes through `saveUser`.**

`DefaultUserDeletionService.requestDeletion` flips the deletion status inside its write-transaction (DefaultUserDeletionService.java:114-159). The relevant lines:

```java
// Flip user state via saveUser → @CacheEvict fires for both Redis caches.
user.setDeletionStatus(DeletionStatus.PENDING_DELETION);
userService.saveUser(user);
```

(DefaultUserDeletionService.java:139-141.) The inline comment names the eviction explicitly. **Confirmed: not a direct repository call.**

**2. `saveUser` evicts both caches.**

`DefaultUserService.saveUser` (DefaultUserService.java:64-83) is wrapped with `@Caching(evict = {...})`:

```java
@Caching(evict = {
    @CacheEvict(value = "redisUserInfo", key = "#currentUser.id"),
    @CacheEvict(value = "redisUserInfo", key = "#currentUser.firebaseUid"),
    @CacheEvict(value = "redisUserAuth", key = "#currentUser.firebaseUid")
})
public User saveUser(User currentUser) { ... }
```

(DefaultUserService.java:65-77.) **Confirmed:** both `redisUserInfo` keys (id and firebaseUid) and the `redisUserAuth` key (firebaseUid) are evicted on every `saveUser` call.

**3. Repopulation on the next request.**

The filter calls `firebaseAuthService.getCachedAuthData(decoded.getUid())` (FirebaseAuthFilter.java:86). The implementation:

```java
@Cacheable(value = "redisUserAuth", key = "#firebaseUid")
public Optional<AuthenticatedUserDTO> getCachedAuthData(String firebaseUid) {
  if (StringUtils.isBlank(firebaseUid)) {
    return Optional.empty();
  }
  return userRepository.findAuthDataByFirebaseUid(firebaseUid);
}
```

(DefaultFirebaseAuthService.java:62-69.) On a cache miss, this runs `userRepository.findAuthDataByFirebaseUid` — a `@Query`-annotated JPQL constructor projection (UserRepository.java:46-65) that selects `u.deletionStatus` directly from the `users` table at line 57. The fresh DB row's `deletion_status` is bound into the `AuthenticatedUserDTO` constructor (DTO record fields at AuthenticatedUserDTO.java:29 include `DeletionStatus deletionStatus`), and Spring Cache then writes the populated DTO back into the `redisUserAuth` cache under the firebaseUid key.

**4. Lookup is `@Cacheable`, not a raw repo call.**

`getCachedAuthData` is `@Cacheable` (DefaultFirebaseAuthService.java:63). The filter calls it through the Spring AOP proxy from outside the bean — exactly the pattern the in-line comment at DefaultFirebaseAuthService.java:54-57 says is required. The filter is not in the same bean, so the proxy applies. **Confirmed: cache lookup goes through `@Cacheable`.**

**5. Race-window cache contents.**

Per the 2026-05-18 contract §3 timeline:

- T+1.5 (`me/delete` commits): the write-tx in `requestDeletion` finishes, `saveUser` is called inside the same tx (DefaultUserDeletionService.java:114-159), `@CacheEvict` fires post-commit (Spring's default for `@CacheEvict` is *before* the method body unless `beforeInvocation=true` is set, but `@Caching` evicts after the method returns successfully on a non-transactional method; `saveUser` itself is not `@Transactional` so the eviction fires as part of the surrounding tx's commit phase). Net effect for the contract: by the time `me/delete` returns 200 to the client, the cache key is gone.
- T+2.0 (SSR call inbound): filter calls `getCachedAuthData` → cache miss → DB fetch → fresh PENDING_DELETION row → cache populated with PENDING_DELETION.
- T+5.7 (filter sees PENDING_DELETION): the freshly-cached entry carries the **correct** PENDING_DELETION. The race is **not** "cache held stale ACTIVE"; it is "fresh PENDING_DELETION cache combined with a state-mutating filter branch." The cache machinery is working correctly. The amplifier is the filter's mutation step.

The 2026-05-18 logs the brief cites (`select u1_0.deletion_status from users` on the SSR path) corroborate this — the SQL ran on a cache miss, produced a fresh PENDING_DELETION row, and the filter then called `cancelDeletionOnLogin` on the fresh-but-correctly-PENDING_DELETION state. **No cache-eviction bug.** The eviction worked; the filter's response to the eviction is what's wrong, and that is what C-3 addresses.

### Verdict (Q-3)

Cache eviction on `requestDeletion` is structurally correct and consistent with the 2026-05-14 connection-pool decision. The race in the contract §3 timeline is **not** caused by a stale cache; it is caused by the filter's design to mutate state on `PENDING_DELETION`. C-3 is the right fix — removing the mutation, leaving the cache machinery alone.

---

## 5. Q-4 — `firebase-sync` handler — current restore logic

### Step-by-step trace

`AuthController.firebaseSync` (AuthController.java:51-110):

| Step | Code | What it does |
|---|---|---|
| 1 | Lines 60-62 | Resolve `Authorization` header → `resolveFirebaseToken(servletRequest)` → `verifyToken(idToken)`. Returns a decoded `FirebaseToken`. |
| 2 | Line 62 | Extract `email = decoded.getEmail()`. |
| 3 | Lines 68-76 | If `email` non-blank AND `userAuditService.isEmailBanned(email)` → best-effort `firebaseUserService.deleteFirebaseUser(decoded.getUid())` (try/catch, warns and continues on failure) → return 403 `EMAIL_BANNED` / `email.banned`. |
| 4 | Line 78 | `getOrCreateUser(request, servletRequest)` — re-verifies the token internally (per inline comment at lines 56-59, "a second verifyIdToken call is cheap") and inserts a local user row if absent. |
| 5 | Lines 80-82 | If returned user is null → 500. |
| 6 | Lines 87-89 | If `user.isDisabled()` → return 403 `USER_BANNED` / `user.banned`. |
| 7 | Lines 94-98 | If `user.getDeletionStatus() == DeletionStatus.PENDING_DELETION` → `userDeletionService.cancelDeletionOnLogin(user)`; set local `boolean restored = true`. |
| 8 | Line 103 | `AuthUserDTO dto = modelMapper.map(user, AuthUserDTO.class)`. |
| 9 | Lines 105-108 | Build `HttpHeaders`; if `restored`, `headers.set("X-Account-Restored", "true")`. |
| 10 | Line 109 | `return new ResponseEntity<>(dto, headers, HttpStatus.OK)`. |

### Specific answers

**1. Steps in order:** the table above.

**2. Where is `cancelDeletionOnLogin` called?** AuthController.java:96, inside the `if (user.getDeletionStatus() == DeletionStatus.PENDING_DELETION)` branch. This is step 6 in spec §10.1's numbering (the spec lists 7 steps with "step 5" being the disabled check; the handler matches the spec exactly).

**3. `X-Account-Restored: true` timing.** Set at line 107, **after** `cancelDeletionOnLogin` runs at line 96. The `restored` flag captures the decision at step 7; the header is written into the response at step 9 just before `ResponseEntity` construction. No race — `cancelDeletionOnLogin` is synchronous and completes (or throws and rejects the response) before the header is set.

**4. Does the handler depend on the filter having flipped `deletion_status` to ACTIVE first?**

No. The handler is self-contained. Several pieces of evidence:

- The handler calls `firebaseAuthService.verifyToken` directly at line 61 — it does not rely on the filter to do so.
- The handler reads `user.getDeletionStatus()` from the just-loaded `User` entity at line 95 (loaded by `getOrCreateUser` at line 78). If the entity carries `PENDING_DELETION`, the handler itself flips it via `cancelDeletionOnLogin`. There is no read of `SecurityContextHolder` in this branch.
- The filter exempts `firebase-sync` precisely so the filter cannot have flipped state before the handler runs (see Q-2 sub-case 3A). Spec §10.1 step 1 ("Verify Firebase ID token") is the handler's own work, not the filter's.

If C-3 removes the filter's `cancelDeletionOnLogin` call, the handler keeps working unchanged — it is the only remaining restorer (per C-2).

**5. Filter vs handler on `/api/auth/firebase-sync`.**

`/api/auth/firebase-sync` matches the substring check at FirebaseAuthFilter.java:79. The filter **does not run token verification** on this path; the handler runs its own `verifyToken` at AuthController.java:61. This is "in lieu of" the filter, exactly as spec §10.1 and the contract §7 Q-5 expected. **Confirmed.**

### Verdict (Q-4)

`firebase-sync` is structurally self-contained and matches spec §10.1 step-for-step. Under C-3 it becomes the only restorer; the handler needs no code change to take on that role.

---

## 6. Q-5 — `cancelDeletionOnLogin` callers and audit-log shape

### Callers (today)

Grep across `src/main/java`:

| Caller | File:line | Removed by C-3? |
|---|---|---|
| `FirebaseAuthFilter` | FirebaseAuthFilter.java:102 | **Yes** (this is the call C-3 removes) |
| `AuthController.firebaseSync` | AuthController.java:96 | No (this is the call C-2 / C-9 preserves) |

Exactly the two callers the contract anticipated. No tests, no other services, no scheduled jobs reach `cancelDeletionOnLogin`. (Test-side references — five mock-`verify` calls across three test classes — are listed in Q-6.)

### `cancelDeletionOnLogin` implementation walk-through

`DefaultUserDeletionService.cancelDeletionOnLogin` (DefaultUserDeletionService.java:168-212):

```java
public void cancelDeletionOnLogin(User user) {
  if (user.getDeletionStatus() != DeletionStatus.PENDING_DELETION) {
    // Idempotent: already active, nothing to restore.
    return;
  }
  ...
}
```

**Early-return guard (lines 170-173):** the method returns immediately if the user is not `PENDING_DELETION`. The guard is **before** any DB write, so a second concurrent call against an already-restored user is a clean no-op — no spurious row updates, no double audit-log writes, no product re-flips.

**Inside the tx (lines 178-211):**

- `deletionRequestRepository.findByUserId(userId).ifPresent(req -> req.setStatus(CANCELLED); req.setCancelledAt(now); req.setCancelledReason("user_login"); save)` (lines 180-188). Per the early-return guard, this path only runs when the user is PENDING_DELETION, so the request row is expected to be PENDING; the `ifPresent` is defensive in case of inconsistent state.
- `auditLogRepository.findFirstByUserIdHashAndActualDeletionAtIsNullAndCancelledAtIsNullOrderByCreatedAtDesc(userIdHash).ifPresent(a -> a.setCancelledAt(now); save)` (lines 193-201). Matches the open audit-log row by `user_id_hash`; the predicate (`actual_deletion_at IS NULL AND cancelled_at IS NULL`) ensures only the open row is touched. The query is backed by `idx_udal_user_id_hash` per spec §6.
- `user.setDeletionStatus(DeletionStatus.ACTIVE); userService.saveUser(user)` (lines 203-204). The `saveUser` call triggers the same `@CacheEvict` chain analysed in Q-3 — both `redisUserAuth` and `redisUserInfo` evicted on restoration. **Confirmed: cache eviction handled via `saveUser`.**
- `productService.findProductIdsByOwnerAndState(userId, ProductState.INACTIVE).forEach(...changeProductStateAsSystem(..., ACTIVE))` (lines 206-210). Re-activates products that `requestDeletion` had flipped to INACTIVE. Uses the system-context state-change to bypass owner-checks.

### What does the existing test pin?

`DefaultUserDeletionServiceTest.cancelDeletionOnLoginIsNoOpWhenUserAlreadyActive` (test file lines 222-234) directly asserts the early-return: no tx interaction, no repository save, no product flip. **The service-level idempotency is covered by a unit test.**

### Does removing the filter caller break any current scenario?

No. The legitimate scenarios that restore are:

- **User signs back in (Google, email, OAuth).** Frontend → `firebase-sync` → handler restores. The filter's call was never on this path because `firebase-sync` is filter-exempt.
- **User signs back in and the *next* authenticated request hits a non-firebase-sync endpoint.** This is the *side-channel* path the contract §4 wants to eliminate — and it is exactly the race. After C-3, this scenario serves the request anonymously (no restoration), and the user is forced through the explicit `firebase-sync` sign-in to restore. That is the intended new behaviour.

There is no production code path where the filter's `cancelDeletionOnLogin` was the only restoration trigger. Mobile/web sign-in flows already use `firebase-sync` as the explicit handshake. The `firebase-sync` round-trip is unavoidable for any "sign in" event the contract considers legitimate.

### Verdict (Q-5)

- Exactly two production callers today; only the filter's call is removed by C-3.
- The service is fully idempotent on already-ACTIVE users (DefaultUserDeletionService.java:170-173), and the test suite pins this behaviour.
- No current scenario depends on the filter's side-channel call. The architectural change is clean.

---

## 7. Q-6 — Test coverage that will need updating

### `FirebaseAuthFilterTest` (6 tests total)

| # | Test method | File:line | What it asserts | Disposition under C-3 |
|---|---|---|---|---|
| 1 | `disabledUserShortCircuitsWith403UserBanned` | line 81 | Disabled user → 403 USER_BANNED, no chain proceeded. | **Unchanged** — C-4 preserves this branch. |
| 2 | `pendingDeletionTriggersCancelAndSetsAccountRestoredHeader` | line 101 | PENDING_DELETION → `cancelDeletionOnLogin(user)` called + `X-Account-Restored: true` header + auth context set. | **Delete.** Directly asserts the behaviour C-3 removes. |
| 3 | `activeUserDoesNotCallCancelAndOmitsRestoredHeader` | line 122 | ACTIVE user → no cancel, no restore header, chain proceeds. | **Unchanged.** |
| 4 | `cancelDeletionOnLoginIsCalledTwiceWhenTwoRequestsRaceTheSameStaleCache` | line 140 | Two concurrent requests both call `cancelDeletionOnLogin`. Pins the "filter doesn't dedupe; service does" contract. | **Delete.** Whole test is meaningless under C-3 — neither request calls `cancelDeletionOnLogin`. The race-window asserted no longer exists at the filter layer. |
| 5 | `freshCacheAfterRestoreSkipsCancelBranchEntirely` | line 171 | After cache refresh to ACTIVE, no DB hit, no service call, no restore header. | **Replace** with a "PENDING_DELETION falls through with no auth context" test (per C-3): assert `userDeletionService.cancelDeletionOnLogin` is never called, `X-Account-Restored` header is absent, `SecurityContextHolder` is empty, and `filterChain.doFilter` runs. |
| 6 | `firebaseSyncPathSkipsTokenVerificationBranch` | line 191 | `/api/auth/firebase-sync` → no `verifyToken` call, chain proceeds. | **Unchanged** — C-3 does not touch the firebase-sync exemption. |

**New tests required** (engineering brief's call, but the audit names them so they aren't missed):

- `pendingDeletionDropsAuthContextAndContinuesAnonymous` (replaces #2): assert no `cancelDeletionOnLogin`, no `X-Account-Restored`, `SecurityContextHolder` empty, chain proceeds.
- Optionally a public-path variant asserting that a PENDING_DELETION request to `/api/public/...` still completes the chain (this can also be done as an integration test if the infrastructure for it lands).

### `AuthControllerFirebaseSyncTest` (5 tests total)

Tests at lines 76, 96, 114, 135, 160:

| # | Test method | What it asserts | Disposition under C-3 |
|---|---|---|---|
| 1 | `firebaseSyncReturns403EmailBannedAndDeletesFirebaseUser` (line 77) | Banned email → 403, Firebase user deleted. | **Unchanged.** |
| 2 | `firebaseSyncEmailBannedStillReturns403EvenIfFirebaseDeleteFails` (line 97) | Banned email → 403 even when Firebase delete throws. | **Unchanged.** |
| 3 | `firebaseSyncReturns403UserBannedWhenUserDisabled` (line 115) | Disabled user → 403 USER_BANNED, no cancel call. | **Unchanged.** |
| 4 | `firebaseSyncReturns200WithRestoredHeaderAndCallsCancelOnPendingUser` (line 136) | PENDING user → 200 + `X-Account-Restored` + `cancelDeletionOnLogin(user)`. | **Unchanged** — C-9 preserves the handler's restore path exactly. |
| 5 | `firebaseSyncReturns200WithoutRestoredHeaderForActiveUser` (line 161) | ACTIVE user → 200 + no header + no cancel. | **Unchanged.** |

**Confirmed: every test in this file is unaffected by C-3.**

### `DefaultUserDeletionServiceTest` (`cancelDeletionOnLogin` group, 2 tests)

| # | Test method | Line | What it asserts | Disposition under C-3 |
|---|---|---|---|---|
| 1 | `cancelDeletionOnLoginFlipsStateAndRestoresProducts` | 193 | Happy path: status flips, request CANCELLED, audit cancelled_at, products restored, `saveUser` called. | **Unchanged.** |
| 2 | `cancelDeletionOnLoginIsNoOpWhenUserAlreadyActive` | 223 | Already-ACTIVE user → no tx, no saves, no product flips. | **Unchanged.** This test is the load-bearing guarantee for the service-level idempotency that C-9 still relies on (for the rare case of a double-firing `firebase-sync`). |

**Confirmed: every test in this group is unaffected.**

### Integration coverage of the cancel-race

There is no integration test of the cancel-race in the current suite. All relevant tests are unit-level (Mockito-based, MockMvc standalone-setup). The race in `user-deletion-auth-contract.md` §3 only manifests across the filter + service + cache + DB at HTTP-request granularity, which the current backend test infrastructure does not exercise. This matches the standing testing-infrastructure gap noted in the round-2 handoff. The engineering brief should not assume integration coverage exists.

### Verdict (Q-6)

- **Delete:** `pendingDeletionTriggersCancelAndSetsAccountRestoredHeader` (line 101), `cancelDeletionOnLoginIsCalledTwiceWhenTwoRequestsRaceTheSameStaleCache` (line 140).
- **Replace:** `freshCacheAfterRestoreSkipsCancelBranchEntirely` (line 171) — rewrite as a "drops auth context, falls through anonymous" assertion.
- **Add:** one or two unit tests covering the C-3 fall-through behaviour (filter shape and `SecurityContextHolder` emptiness).
- **Unchanged:** the disabled-user filter test (#1), the active-user filter test (#3), the firebase-sync exemption filter test (#6), all 5 `AuthControllerFirebaseSyncTest` tests, and both `DefaultUserDeletionServiceTest.cancelDeletionOnLogin*` tests.

---

## 8. Adjacent observations (Part 4b)

These do not block C-3. Flagged for Mastermind's awareness.

### A. Filter's exception branch returns 401 for `/api/auth/**` and `/internal/**` paths, despite those being `permitAll`

`FirebaseAuthFilter.java:125` checks only `request.getServletPath().startsWith("/api/public")` when deciding whether to suppress the 401 on token-verification failure. If a request to `/api/auth/firebase/{firebaseId}` (a `GET` endpoint at `AuthController.java:112-119`, under `/api/auth/**` permitAll) arrives with a malformed token, the filter returns 401 even though the URL rule would have served it as anonymous. Severity: **low** — `/api/auth/firebase/{id}` is used for cross-user lookups and the frontend likely always sends a valid token or no token at all. But the asymmetry with `/api/public/**` is worth noting because once C-3 reshapes the filter, future engineers may extend the "drop context, continue" pattern and accidentally not handle `/api/auth/**`. Recommendation: when implementing C-3, consider broadening the exception-branch suppression to all `permitAll` paths, or — simpler — drop context unconditionally and let the URL rules decide.

### B. The race-test rationale in `FirebaseAuthFilterTest.cancelDeletionOnLoginIsCalledTwiceWhenTwoRequestsRaceTheSameStaleCache` (line 140) describes a *different* race than the one the contract is fixing

The test's inline comment (FirebaseAuthFilterTest.java:141-145) describes a race where "two near-simultaneous authenticated requests can both see the same PENDING_DELETION cache snapshot **before** the first one's `saveUser` eviction propagates." That is the *intra-restoration* race (two requests both trying to restore). The 2026-05-18 SSR race is *post-deletion* (a request arriving after `requestDeletion` already evicted, seeing the fresh PENDING_DELETION, and inappropriately restoring). The test's framing was always slightly off-aim — it pinned the wrong race shape. Severity: **low** (documentation/comment, no behavioural impact). The whole test is being deleted anyway under C-3, so this is mostly historical. Worth noting in case the engineering brief reuses the comment text.

---

## 9. Open questions for Mastermind

None. The audit resolved all six numbered questions cleanly against the on-disk code. The two adjacent observations are flagged but do not need a decision before C-3 implementation begins.

---

End of audit.
