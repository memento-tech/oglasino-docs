# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-22
**Task:** Add an authoritative `wasRegister: boolean` field to the `/auth/firebase-sync` response. The backend already knows whether the request created a new `users` row or matched an existing one — propagate that bit to the response so the frontend has a definitive sign-up-vs-login discriminator for every auth method (email+password, Google, Facebook).

## Implemented

- New record `SyncResult(User user, boolean wasRegister)` in `security.service`, the new public return type of `FirebaseAuthService.getOrCreateUser`. `wasRegister` is derived from backend state (whether `createUserSynchronized` performed the `INSERT`) — never from a client-supplied `LoginRequest` field.
- `DefaultFirebaseAuthService.createUserSynchronized` now returns `SyncResult`. The `wasRegister=true` value is set inside the synchronized block at the exact branch that calls `userService.saveUser(newUser)`, so the concurrent-create race resolves correctly: the losing thread's inner `getUser(token)` finds the just-inserted row and reports `wasRegister=false`. The outer fast-path `getUser(token)` short-circuit also reports `wasRegister=false`.
- `AuthUserDTO` gains a `boolean wasRegister` field with standard `isWasRegister()` / `setWasRegister()` accessors (matching the surrounding `boolean disabled` / `isDisabled()` shape). Jackson serializes it as the JSON key `wasRegister`.
- `AuthController.firebaseSync` consumes `SyncResult`, populates `dto.setWasRegister(syncResult.wasRegister())` after the existing `modelMapper.map(...)` call (ModelMapper would otherwise leave the field at its default `false` since the source `User` entity has no matching column).
- `AuthControllerFirebaseSyncTest`: three existing mocks updated from `.thenReturn(user)` to `.thenReturn(new SyncResult(user, false))`. Two new tests added pinning the contract: `firebaseSyncReturnsWasRegisterTrueWhenServiceCreatedNewRow` and `firebaseSyncReturnsWasRegisterFalseWhenServiceMatchedExistingRow`. Existing PENDING_DELETION and ACTIVE tests extended to assert `$.wasRegister == false`. The USER_BANNED test gains an explicit `$.wasRegister` doesNotExist assertion documenting that the error-response shape does NOT carry the field.

## Files touched

- src/main/java/com/memento/tech/oglasino/security/service/SyncResult.java (new, +11 lines)
- src/main/java/com/memento/tech/oglasino/security/service/FirebaseAuthService.java (+2 / -1)
- src/main/java/com/memento/tech/oglasino/security/service/impl/DefaultFirebaseAuthService.java (+15 / -5)
- src/main/java/com/memento/tech/oglasino/dto/AuthUserDTO.java (+16 / 0)
- src/main/java/com/memento/tech/oglasino/controller/AuthController.java (+8 / -3)
- src/test/java/com/memento/tech/oglasino/controller/AuthControllerFirebaseSyncTest.java (+67 / -7)

## Tests

- Ran: `./mvnw test` (full suite)
- Result: **551 passed, 0 failed**
- Ran: `./mvnw test -Dtest='AuthControllerFirebaseSyncTest'` (touched test class)
- Result: **7 passed, 0 failed** (5 pre-existing + 2 new `wasRegister` branches)
- Compile: `./mvnw clean compile` — **0 `-Xlint` warnings** (the 3 pre-existing warnings tracked in `issues.md` 2026-05-16 were closed in the 2026-05-20 bug-chat closeout per `state.md`; no regression introduced).
- Format: `./mvnw spotless:check` — clean (`spotless:apply` was run once mid-session to reformat the new test methods to single-line `.thenReturn(...)`).
- New tests added: `firebaseSyncReturnsWasRegisterTrueWhenServiceCreatedNewRow`, `firebaseSyncReturnsWasRegisterFalseWhenServiceMatchedExistingRow`; existing PENDING_DELETION / ACTIVE / USER_BANNED tests extended with `$.wasRegister` JSON-path assertions.
- `DefaultFirebaseAuthServiceDisplayNameTest` (4 tests, calls `getOrCreateUser` for side-effects via an `ArgumentCaptor` on `userService.saveUser`) required no source change — the test ignores the return value, so the signature change from `User` → `SyncResult` is transparent. All 4 still pass.

## Cleanup performed

- none needed

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): confirmed
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): confirmed — one small adjacent finding flagged in "For Mastermind"
- Part 6 (translations): N/A this session (no new translation keys)
- Part 7 (error contract): confirmed — the USER_BANNED / EMAIL_BANNED error responses continue to use the existing `ProductErrorResponse` `{ field, code, translationKey }` shape; `wasRegister` does NOT appear on the error DTO (judgment call per the brief — see "For Mastermind")
- Part 11 (trust boundaries): confirmed — `wasRegister` is sourced inside `createUserSynchronized` from the actual `INSERT` branch; the bit cannot be derived from any field on `LoginRequest`. The brief's trust-boundary check passes.

## Known gaps / TODOs

- none

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity):
    - `record SyncResult(User user, boolean wasRegister)` — the brief explicitly suggested this shape; it is the only mechanic that surfaces the create-vs-match decision to the controller without leaking it via instance state or an out-parameter. The Javadoc on the record names the consumer (`FirebaseAuthService.getOrCreateUser`) and the source-of-truth contract so the next reader does not re-litigate.
    - One new `boolean wasRegister` field on `AuthUserDTO` with standard accessors. The field has exactly one writer (the controller) and exactly one consumer (web GA4 per `features/google-analytics-v1.md`); no config value, no helper.
  - Considered and rejected:
    - Re-using `Pair<User, Boolean>` or `Map.Entry<User, Boolean>` in lieu of a record — rejected because the boolean's meaning is then carried only by call-site comments. A named `record` self-documents at every read.
    - Naming the record's boolean `created` (matching the brief's `record SyncResult(User user, boolean created)` sketch) and translating `created` → `wasRegister` at the controller — rejected. Two names for the same bit is friction; the response field's name (`wasRegister`) is the contract with the web client, and the in-server name should match.
    - Setting `wasRegister` on the outer `getUser(token).isEmpty()` check in `getOrCreateUser` and passing nothing extra into `createUserSynchronized` — rejected because the concurrent-create race would cause the loser to claim `wasRegister=true` despite having no `INSERT` to its name. Sourcing the bit from inside the synchronized block's actual `INSERT` branch is the only correct shape.
    - Adding `wasRegister` to the `ProductErrorResponse` body so banned-sign-in attempts also carry the discriminator — rejected. The error response is a separate DTO with its own contract (`{ errors: [{ field, code, translationKey }] }`); adding a discriminator field would either pollute every other 4xx response in the codebase or fork the auth error shape away from the project-wide error contract (Part 7). Web GA4 fires `sign_up` / `login` only when `backendUser` is non-null (per the spec's "Events fire only when `backendUser` is non-null, so banned-sign-in attempts produce no `login` or `sign_up` event"), so the bit is not needed on the error path. The new USER_BANNED test asserts `$.wasRegister.doesNotExist()` to make this contract decision visible to the next reader.
  - Simplified or removed: nothing this session.
- **Part 4b adjacent observation (low severity, not fixed):** `AuthController` retains `@Autowired private CurrentUserService currentUserService;` (line 43) that is never referenced in the controller body. Same is true for `userFacade` *for* `firebaseSync`, but `userFacade` is used in `getUserForFirebaseId` so it is live. `currentUserService` appears unused in this file — it may be a leftover. Out of scope for this brief (touching it would also force test-mock updates and is unrelated to `wasRegister`). Flag-only.
- **Brief vs reality:** no discrepancies. The brief's assumptions all held:
  - `DefaultFirebaseAuthService.createUserSynchronized` exists at the path the brief implies, with the same signature shape the `issues.md` 2026-05-19 entry described.
  - The response DTO returned by `/auth/firebase-sync` is `AuthUserDTO`, exactly as the brief states (per the `decisions.md` 2026-05-21 Consent Mode v2 closing entry that removed `allowPreferenceCookies` from this DTO end-to-end).
  - The create-vs-match seam is locatable: `getUser(token).orElseGet(() -> createUserSynchronized(...))` was the existing pattern; replacing the bare `User` return with a `SyncResult` wrapper was a clean mechanical change.
- **Trust-boundary check (required):** confirmed. `wasRegister` is derived from backend state at the exact point of the `INSERT` (`userService.saveUser(newUser)` inside `createUserSynchronized`). No code path reads from `loginRequest`, the Firebase token claims, or any other client-influenceable source when computing the bit. The `LoginRequest` DTO was not modified.
- **Suggested next step:** Brief 4 (web) — `oglasino-web` updates `syncUserToBackend` to return `{ user, wasRegister }` consuming the new response field, and `UseTokenRefresh.tsx` fires `track('sign_up' | 'login', ...)` accordingly per § Architecture → Sign-up vs login discriminator in the feature spec.
- **No drafted config-file text.** No `conventions.md`, `decisions.md`, `state.md`, or `issues.md` change is required for this session — the wire-format addition is captured in the feature spec already; the `state.md` `Last updated` flip and the closing `decisions.md` entry happen at feature close after Briefs 4–11.
