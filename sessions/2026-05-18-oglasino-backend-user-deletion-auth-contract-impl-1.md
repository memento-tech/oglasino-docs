# Session — User Deletion Auth Contract (C-3 + D-4) — implementation

**Date:** 2026-05-18
**Repo:** `oglasino-backend`
**Branch:** `dev` (verified at session start; all User Deletion working-tree changes are on `dev`)
**Brief:** `.agent/brief.md` — "Engineering Brief — User Deletion Auth Lifecycle (backend)"
**Type:** implementation (code + tests + drafted spec amendment text)
**Source contract:** [`../../oglasino-docs/features/user-deletion-auth-contract.md`](../../oglasino-docs/features/user-deletion-auth-contract.md) §4 (C-3), §7 (adjacent observation A folded as D-4)
**Source audit:** [`audit-user-deletion-auth-lifecycle.md`](audit-user-deletion-auth-lifecycle.md) §2 (Q-1), §3 (Q-2), §7 (Q-6), §8 (observation A)

---

## Brief vs reality

One brief-vs-hard-rule conflict surfaced at session start and was resolved with Igor before any edits:

**Spec amendment scope vs CLAUDE.md hard rule "No edits to other repos."**

- Brief Part 3 ("Spec amendment in `oglasino-docs/features/user-deletion.md`") instructed me to edit §10 step 4, add a cross-reference, and replace §17.14's body.
- CLAUDE.md hard rule: *"No edits to other repos. Never touch `../oglasino-docs/`. If a task seems to require it, stop and tell Igor."*
- The contract document itself (`user-deletion-auth-contract.md` §10) reinforces the rule: *"Docs/QA applies these amendments after the engineering briefs land."*
- Asked Igor; he selected **"Draft text only, Docs/QA applies"** — I do code + tests, and I draft the spec amendment text in this session's "For Mastermind" section verbatim so Docs/QA can paste it in without re-deriving the wording.

No other brief-vs-reality discrepancies. The audit had pre-resolved Q-1 through Q-6 cleanly, line numbers shifted only slightly from the audit's reads, and the code matched the audit's claims exactly at the points being edited.

---

## Implemented

### Part 1 — `FirebaseAuthFilter.java`

**1A. Contract C-3: PENDING_DELETION → drop auth context, continue chain.**

The mutation branch (was: load user, `userDeletionService.cancelDeletionOnLogin(user)`, `X-Account-Restored: true`) is replaced with `SecurityContextHolder.clearContext(); filterChain.doFilter(request, response); return;`. The new shape mirrors the existing exception-branch precedent at lines 115–130 — "clear context, let URL rules decide." Inline comment cites C-3 and names the Spring URL-rule outcomes per endpoint family.

**1B. Adjacent cleanup D-4: broaden 401-suppression to all `permitAll` prefixes.**

The exception branch's path check was `!startsWith("/api/public")` — a malformed token on `/api/auth/**` or `/internal/**` would short-circuit to 401 even though `SecurityConfig.filterChain` permits anonymous on those prefixes. Replaced with an inline three-prefix check (`/api/public`, `/api/auth`, `/internal`). Did not extract a helper because the predicate is used exactly once.

**Field cleanup (cleanliness consequence of C-3, conventions Part 4).**

Removing the mutation branch removes the only usages of `userRepository` (line 101) and `userDeletionService` (line 102) in the filter. Both fields, their constructor parameters, the assignments, and the `UserRepository` import are deleted. `userService` stays — it is still consumed by `validateForPostmanTesting`. `userDeletionService` is now wired solely through `AuthController.firebaseSync` (C-9), unchanged.

### Part 2 — `FirebaseAuthFilterTest.java`

Applied the audit Q-6 inventory plus the constructor-signature fixup forced by Part 1's field cleanup:

| Action | Test | Disposition |
| --- | --- | --- |
| **Deleted** | `pendingDeletionTriggersCancelAndSetsAccountRestoredHeader` | Asserted the exact behavior C-3 removes. |
| **Deleted** | `cancelDeletionOnLoginIsCalledTwiceWhenTwoRequestsRaceTheSameStaleCache` | The race it pinned is structurally impossible under C-3 (neither request reaches `cancelDeletionOnLogin` from the filter). Audit observation B also noted the test's inline comment described the wrong race shape relative to the 2026-05-18 incident. |
| **Rewritten** | `freshCacheAfterRestoreSkipsCancelBranchEntirely` → `pendingDeletionDropsAuthContextAndContinuesAnonymous` | Asserts `filterChain.doFilter` runs, `verify(userDeletionService, never())`, `X-Account-Restored` header is absent, `SecurityContextHolder.getContext().getAuthentication()` is null. |
| **Added** | `pendingDeletionOnPublicPathStillFallsThroughAnonymous` | Public-path variant of the C-3 assertion (the exact 2026-05-18 race shape on `/api/public/product/search`). Pins that the filter does not short-circuit a 200 response on the public endpoint. The brief flagged this test as optional; added because it is essentially free given the existing fixtures and pins the contract's central promise. |
| **Untouched (behavior assertions)** | `disabledUserShortCircuitsWith403UserBanned`, `activeUserDoesNotCallCancelAndOmitsRestoredHeader`, `firebaseSyncPathSkipsTokenVerificationBranch` | All three still pass. The only change in the file outside the deleted/rewritten methods is the `@BeforeEach` constructor call (now 4 args, not 6) and the corresponding removal of the `userRepository` mock + the `User`/`UserRepository` imports + the unused `Mockito.times` import. |

Result: `FirebaseAuthFilterTest` now has 5 tests (was 6); whole suite 443 (was 444).

### Part 3 — Spec amendment (drafted, NOT applied)

Per Igor's session-start decision, I did not edit `oglasino-docs/features/user-deletion.md`. Drafted text is in the **For Mastermind** section below, ready for Docs/QA to paste in.

### Forced collateral edit — `ApplicationConfig.java`

Removing the two filter fields forced a matching constructor-call edit in `ApplicationConfig.firebaseAuthFilter` (the `@Bean` producer): dropped the `UserDeletionService` and `UserRepository` parameters and the corresponding imports. `ApplicationConfig` is a wiring file, not in the brief's auth-area exclusion list (`AuthController`, `DefaultUserDeletionService`, `SecurityConfig`, `DefaultFirebaseAuthService`). No other change to the file.

---

## Files touched

- `src/main/java/com/memento/tech/oglasino/security/filter/FirebaseAuthFilter.java` — C-3 + D-4 + field cleanup
- `src/main/java/com/memento/tech/oglasino/ApplicationConfig.java` — constructor signature fixup
- `src/test/java/com/memento/tech/oglasino/security/filter/FirebaseAuthFilterTest.java` — 2 deletions, 1 rewrite, 1 new test, mock/import cleanup, Javadoc updated

No other production files were edited. No spec files in `oglasino-docs/` were edited (drafted-only per the session-start decision).

---

## Tests

**Baseline (session start, before any edit):**

- `./mvnw spotless:check` → BUILD SUCCESS (clean)
- `./mvnw test` → `Tests run: 444, Failures: 0, Errors: 0, Skipped: 0` — BUILD SUCCESS

**Final (session end):**

- `./mvnw spotless:check` → BUILD SUCCESS (clean)
- `./mvnw test` → `Tests run: 443, Failures: 0, Errors: 0, Skipped: 0` — BUILD SUCCESS

Net delta: −2 (deletes) + 1 (rewrite) + 1 (public-path addition) = +0 from rewrite/add side, −2 from deletes → 444 − 1 = 443. Inside the brief's predicted band of 443–444.

Spot-checks:
- `FirebaseAuthFilterTest` — 5 tests, all green
- `AuthControllerFirebaseSyncTest` — 5 tests, all green (untouched per audit Q-6)
- `DefaultUserDeletionServiceTest` — 24 tests, all green (untouched per audit Q-6)

No new test failures introduced anywhere in the suite.

---

## Cleanup performed

- Removed the `userRepository` field and its constructor parameter from `FirebaseAuthFilter` (the C-3 edit removed its only call site; per conventions Part 4 cleanliness, dead state is deleted in the same session).
- Removed the `userDeletionService` field and its constructor parameter from `FirebaseAuthFilter` (same reason).
- Removed the `UserRepository` import from `FirebaseAuthFilter`.
- Updated `ApplicationConfig.firebaseAuthFilter` to match the new constructor signature; removed the corresponding two imports (`UserRepository`, `UserDeletionService`).
- Removed the `@Mock UserRepository userRepository` field and the `com.memento.tech.oglasino.entity.User` + `com.memento.tech.oglasino.repository.UserRepository` imports from `FirebaseAuthFilterTest` (no remaining test references them).
- Removed the `static org.mockito.Mockito.times` import from `FirebaseAuthFilterTest` (it was only used by the deleted race test).
- Updated the `FirebaseAuthFilterTest` class Javadoc to reflect the new contract (C-3 + C-4 + firebase-sync exemption) and to drop the now-irrelevant "CHECKPOINT 5 race-window" reference.

`userDeletionService` is kept as a `@Mock` in `FirebaseAuthFilterTest` because two remaining tests use `verify(userDeletionService, never()).cancelDeletionOnLogin(any())` to pin the negative-behavior contract at the filter boundary. The mock is no longer wired into the filter; the `verify` assertions trivially pass, but they preserve readable intent ("the filter does not call this service") for future engineers.

---

## Config-file impact

| File | Impact |
| --- | --- |
| `../oglasino-docs/meta/conventions.md` | **no change** |
| `../oglasino-docs/decisions.md` | **no change** |
| `../oglasino-docs/state.md` | **no change** |
| `../oglasino-docs/issues.md` | **no change** |

The session's spec amendment is to `oglasino-docs/features/user-deletion.md`, which is a feature spec, not one of the four config files. Per the session-start decision with Igor (CLAUDE.md hard rule "no edits to other repos"), I have not edited that file either — the amendment text is drafted in "For Mastermind" below for Docs/QA to apply.

---

## Obsoleted by this session

- **Code:** the PENDING_DELETION mutation branch in `FirebaseAuthFilter.doFilterInternal` (old lines 96–104, including the `userRepository.findById(...)` lookup, the `userDeletionService.cancelDeletionOnLogin(user)` call, and the `response.setHeader("X-Account-Restored", "true")`).
- **Code:** the narrow `!startsWith("/api/public")` exception-branch path check (old line 125).
- **Code:** the `UserRepository userRepository` field, parameter, assignment, and import in `FirebaseAuthFilter`; corresponding `UserRepository` and `UserDeletionService` parameters/imports in `ApplicationConfig.firebaseAuthFilter`.
- **Tests:** `FirebaseAuthFilterTest.pendingDeletionTriggersCancelAndSetsAccountRestoredHeader` and `FirebaseAuthFilterTest.cancelDeletionOnLoginIsCalledTwiceWhenTwoRequestsRaceTheSameStaleCache` are deleted entirely.
- **Tests:** `FirebaseAuthFilterTest.freshCacheAfterRestoreSkipsCancelBranchEntirely` is renamed and rewritten as `pendingDeletionDropsAuthContextAndContinuesAnonymous`.
- **Spec (drafted, not applied):** the body of `user-deletion.md` §17.14 ("Concurrent restore-on-login and admin-ban race") — the race it describes is structurally impossible under C-2 + C-3. Replacement text drafted in "For Mastermind" below.
- **Spec (drafted, not applied):** `user-deletion.md` §10 step 4's previous wording ("If `authData.deletionStatus == 'PENDING_DELETION'` → call `cancelDeletionOnLogin`, set `X-Account-Restored`") — replacement text drafted below.

---

## Conventions check

- **Part 4 (cleanliness):** ✅ No commented-out code. No unused imports/variables/fields/parameters left behind by the C-3 refactor — `userRepository` and `userDeletionService` field + ctor + imports cleared from the filter and from `ApplicationConfig`'s `@Bean` producer; obsolete test mock and imports cleared. No `System.out.println`. No `TODO`/`FIXME` introduced. Spotless and tests pass.
- **Part 4a (simplicity):** ✅ The path check in the exception branch is inlined (one call site) rather than extracted into a helper. C-3's three-line replacement of the mutation branch mirrors an existing precedent in the same method rather than introducing a new abstraction.
- **Part 4b (adjacent observations):** ✅ One adjacent observation surfaced (`modelMapper` is injected into `FirebaseAuthFilter` but unused in the file — pre-existing, not a consequence of this session). Not fixed; flagged below under "Adjacent observations." Audit observation A was implemented as adjacent cleanup D-4 per the brief.
- **Part 5 (session summary):** ✅ Written to both `.agent/2026-05-18-oglasino-backend-user-deletion-auth-contract-impl-1.md` and `.agent/last-session.md`. All required sections present including the explicit values in "Config-file impact."
- **Part 7 (error contract):** ✅ No new error codes. The filter still emits the existing `USER_BANNED` JSON envelope (codes only, no message); the C-3 branch returns no response body or status of its own — it hands the request to the chain.
- **Part 11 (trust boundaries):** ✅ The contract C-3 change *strengthens* the trust boundary: the filter no longer mutates auth state on a side-channel request. `SecurityContextHolder` remains the single source of trusted identity, populated exclusively from a verified Firebase token, and the filter no longer treats "token verifies + PENDING_DELETION" as license to flip the user's deletion state.

---

## Known gaps / TODOs

None introduced this session.

The standing testing-infrastructure gap (no integration coverage that exercises filter + service + cache + DB at HTTP-request granularity) is unchanged — the audit Q-6 noted it, and the brief explicitly stated this brief does not close it.

The frontend half of the contract (web brief — `clearFirebaseTokenCookie` await, `deletionInFlight` flag, axios `cachedToken` reset) is landing in parallel in `oglasino-web` and is not in this session's scope.

---

## Adjacent observations (Part 4b)

### `modelMapper` is injected into `FirebaseAuthFilter` but unused

`FirebaseAuthFilter` declares a `private final ModelMapper modelMapper` field (line 39) and accepts it in the constructor, but no method in the class reads it. The field is wired in `ApplicationConfig.firebaseAuthFilter` and consumes a `ModelMapper` bean for no purpose.

**Why this matters:** small dead wiring, but per conventions Part 4 cleanliness it qualifies as an unused field. Not flagged by the audit; surfaced here only because the C-3 cleanup made it visually obvious in the constructor.

**Why I did not fix it:** it is pre-existing — it was unused before this brief and is unrelated to the C-3 change. The brief's scope is intentionally narrow ("Do not touch other files in the auth area"; this is inside the file but is unrelated to the contract work). Out-of-scope cleanup is properly handled by a future dedicated brief rather than smuggled into this session.

**Severity:** low. A future "filter constructor cleanup" brief can pull this thread together with anything similar that surfaces.

---

## For Mastermind

### A. Spec amendment text (drafted for Docs/QA to apply to `oglasino-docs/features/user-deletion.md`)

Per the session-start decision (CLAUDE.md hard rule "no edits to `oglasino-docs/`"; contract §10 also says Docs/QA applies the amendments), here is the verbatim text Docs/QA should paste in. These match the brief's Part 3 instructions exactly.

#### A.1 — §10 cross-reference (add near the top of §10, before the numbered list)

> See [`user-deletion-auth-contract.md`](user-deletion-auth-contract.md) for the full auth lifecycle contract that this section participates in.

#### A.2 — §10 step 4 (replace the existing step 4 in full)

> 4. If `authData.deletionStatus == 'PENDING_DELETION'` → clear `SecurityContextHolder` and continue the filter chain anonymously, per auth contract C-3. Do **not** call `cancelDeletionOnLogin`. Do **not** set any response header. The user's state is restored exclusively via the `firebase-sync` handshake (§10.1, unchanged), not as a side effect of any other authenticated request.

The numbered list's subsequent steps (was 5, 6, 7 — `Build OglasinoAuthentication`, `Set SecurityContextHolder`, `filterChain.doFilter`) remain unchanged, but they now apply only to the ACTIVE-user branch (which is exactly how the code flows after C-3).

The paragraph below the list that begins *"`cancelDeletionOnLogin` is idempotent..."* should be **deleted** — under C-3 the filter never calls `cancelDeletionOnLogin`, so the idempotency observation is no longer the right thing to say at the filter layer. The service-level idempotency is still real and still load-bearing for the `firebase-sync` path; it is documented adjacent to the service implementation and pinned by `DefaultUserDeletionServiceTest.cancelDeletionOnLoginIsNoOpWhenUserAlreadyActive`.

#### A.3 — §10.1 — no change

Confirmed unchanged in the brief; the `firebase-sync` handler retains its full responsibility per C-9.

#### A.4 — §10.2 — no change

Confirmed unchanged in the brief; the delete-account endpoint is unchanged.

#### A.5 — §17.14 — replace the body (keep the heading)

Replace the three paragraphs ("Theoretical race: ... Acceptable.") with:

> **Historical — superseded by auth contract.** The race described here (concurrent restoration via the filter while an admin ban is in flight) is structurally impossible under the auth lifecycle contract C-2 (restoration is firebase-sync only) and C-3 (filter drops auth context on PENDING_DELETION). See [`user-deletion-auth-contract.md`](user-deletion-auth-contract.md).

Heading stays as `### 17.14 Concurrent restore-on-login and admin-ban race` so existing inbound references don't break.

### B. Adjacent observation surfaced this session

Filed under "Adjacent observations" above: `modelMapper` is an unused injected field in `FirebaseAuthFilter`. Pre-existing, low severity, out-of-scope for this brief. Could be folded into a future "filter constructor cleanup" or "auth area dependency audit" brief.

### C. Confidence on closure

The 2026-05-18 race shape is structurally eliminated by the implemented C-3. The new unit test `pendingDeletionOnPublicPathStillFallsThroughAnonymous` reproduces the exact path (`/api/public/product/search`) the race ran on and pins that the filter no longer mutates state. The `firebase-sync` handler retains the only restore path, and `DefaultUserDeletionServiceTest.cancelDeletionOnLoginIsNoOpWhenUserAlreadyActive` continues to pin service-level idempotency for any rare double-firing of `firebase-sync` itself.

The frontend half (web brief) and Igor's manual verification close the loop.
