# Session summary

**Repo:** oglasino-backend
**Branch:** dev (HEAD 85ed51a)
**Date:** 2026-05-31
**Task:** Read-only reference audit — capture the as-built backend user-deletion contract (Q1–Q8) so mobile (`oglasino-expo`) mirrors it precisely; flag every CODE-vs-DOC divergence. No code changes.

## Implemented

- Nothing implemented — this is a Phase 2 read-only audit. Deliverable is `.agent/audit-user-deletion-mobile-reference.md`, structured Q1–Q8 with `file:line` citations.
- Verified the as-built delete endpoint (`POST /api/secure/user/me/delete`), the FirebaseAuthFilter PENDING_DELETION (C-3) and disabled (C-4) branches, the `firebase-sync` response contract + `X-Account-Restored` header, `UserState.resolve` composition, the public-profile/phone-number server-side gates, the destructive-op trust boundary, the error envelope, and the web-only revalidation path.

## Files touched

- `.agent/audit-user-deletion-mobile-reference.md` (new, audit deliverable)
- `.agent/2026-05-31-oglasino-backend-user-deletion-mobile-reference-1.md` (this summary)
- `.agent/last-session.md` (copy of this summary)
- No source files touched.

## Source files read (ground truth for the audit)

- `controller/UserController.java`, `controller/AuthController.java`, `controller/UserDataController.java`
- `security/filter/FirebaseAuthFilter.java`
- `dto/DeletionRequestResultDTO.java`, `dto/AuthUserDTO.java`, `dto/UserInfoDTO.java`
- `entity/UserState.java`, `entity/DeletionStatus.java`, `repository/projections/UserInfoProjection.java`
- `converter/EntityUserInfoConverter.java`, `service/impl/DefaultUserService.java`, `service/impl/DefaultUserDeletionService.java`
- `repository/UserRepository.java`, `facade/impl/DefaultUserFacade.java`
- `exception/GlobalExceptionHandler.java`, `exception/UserDeletionException.java`, `ReauthRequiredException.java`, `UserLockedFromDeletionException.java`, `ProductErrorResponse.java`
- `listeners/UserStateChangedEventListener.java`, `service/impl/DefaultWebRevalidationService.java`
- Reference docs (read-only): `features/user-deletion.md`, `features/user-deletion-auth-contract.md`

## Tests

- None run — read-only audit, no code changed. (Per brief: do not run state-mutating tests.)

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change
- (All audit findings are CODE-vs-DOC observations for Mastermind's Phase 3 seam analysis, not config-file edits. The two delete-endpoint translationKey divergences and the spec body-shape wording belong to `features/user-deletion.md` §8.8/§10.2/§15, not the four config files — surfaced in the audit and below for Mastermind to route to Docs/QA if it decides the spec should be corrected.)

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code touched, no debug artifacts, deliverable is a referenced doc.
- Part 4a (simplicity): N/A — no code added. See "For Mastermind" structured evidence.
- Part 4b (adjacent observations): two low-severity adjacent observations surfaced inside the audit (follows-list `scheduledDeletionAt` omission; filter's hand-built USER_BANNED constant). Both flagged in the deliverable; neither fixed (read-only audit, out of scope).
- Part 6 (translations): N/A this session.
- Part 7 (error contract): confirmed — every deletion/ban error uses the `{errors:[{field, code, translationKey}]}` envelope.
- Part 11 (trust boundaries): explicitly checked for the destructive `/me/delete` op — verdict CLEAN.

## Known gaps / TODOs

- None. The brief's out-of-scope items (scheduled jobs, R2 deletion, hard-delete cascade) were noted as "exist, not mobile-relevant" per instruction, not audited in depth.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **Divergences for Phase 3 seam analysis (full detail + file:line in the deliverable):**
  1. **delete request body** — code has NO `@RequestBody`; spec §10.2 says empty `{}`. Mobile should send a bodiless POST. (low — doc wording)
  2. **translationKeys `REAUTH_REQUIRED` / `USER_LOCKED_FROM_DELETION`** — code emits `reauth.required` / `user.locked.from.deletion`; spec §8.8 (`features/user-deletion.md:955,958`) + §15 copy table (1605,1608) still show the `errors.`-prefixed variants. Mobile must key off the code strings. (medium — a wrong key means an untranslated delete-flow error on mobile) — candidate `features/user-deletion.md` correction, Docs/QA territory if Mastermind agrees.
  3. **follows-list `scheduledDeletionAt`** — `EntityUserInfoConverter` (the `getMyFollowings` path) sets `state` but never `scheduledDeletionAt`; the projection path (public profile / firebase-uid) sets both. (low — invisible if mobile renders only the badge)
  4. **`AuthUserDTO.scheduledDeletionAt` always null on `firebase-sync`** — no User-side source; benign (PENDING→restored→ACTIVE same call). (informational)
  5. **phone path shorthand** — `/api/secure/user/phoneNumber` vs spec §14.7 `/secure/user/phoneNumber`; same route. (cosmetic)

- **Confirmed-aligned (previously flagged by Φ1):** the ban-code translationKeys `user.banned` / `email.banned` now match between code and the current spec §8.8 — no divergence remains there.

- **Suggested next step:** the two delete-endpoint translationKey divergences (#2) are the only ones with user-facing impact on the mobile flow. If Mastermind wants the spec to be authoritative-correct before mobile adoption, a Docs/QA pass on `features/user-deletion.md` §8.8 + §15 would align the two `errors.`-prefixed keys to the code. Not drafted here (engineer does not write the spec); flagged for routing.
