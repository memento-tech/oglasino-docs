# Session summary

**Repo:** oglasino-web
**Branch:** dev (HEAD 90c74ab)
**Date:** 2026-05-31
**Task:** Read-only reference audit of the as-built web user-deletion / ban UX, for mobile (`oglasino-expo`) parity adoption. Output `.agent/audit-user-deletion-mobile-reference.md`, structured Q1–Q12.

## Implemented

- Read-only Phase-2 audit. No code touched. Produced `.agent/audit-user-deletion-mobile-reference.md` answering Q1–Q12 with file:line citations, PORTABLE/WEB-ONLY labels on the Q2 submit steps and the Q12 list, and CODE-vs-DOC divergences flagged.
- **Q4 doc conflict resolved against code:** the post-deletion trigger (and ban + restore) is an on-disk **Zustand flag on `useAuthStore`** (`accountJustDeleted` / `accountBanned` / `restored`), **not** `sessionStorage`. Spec §14.4 matches; spec §14.3/§4.1/§4.7/§14.10 and auth-contract C-5 still reference the retired `sessionStorage` mechanism.
- Confirmed C-5/C-6 ordering on disk: `setDeletionInFlight(true)` immediately before `getIdToken(true)`; success path submit → `setAccountJustDeleted` → `await signOut` → `await clearFirebaseTokenCookie()` (swallowed) → `router.replace`; flag cleared in `finally`.
- Provider detection is `auth.currentUser.providerData[0].providerId` (not `AuthUserDTO.providerId`); 403 `USER_BANNED` is handled only by the global axios interceptor (never-resolving promise), not the dialog catch; BANNED peer-state lives on `UserInfoDTO.state` while the self DTO uses `AuthUserDTO.disabled`.

## Files touched

- `.agent/audit-user-deletion-mobile-reference.md` (new, audit deliverable)
- `.agent/2026-05-31-oglasino-web-user-deletion-mobile-reference-1.md` (this summary)
- `.agent/last-session.md` (copy of this summary)

No source/test/config files changed (read-only audit).

## Tests

- Not run — read-only audit, no code changes.

## Cleanup performed

- none needed.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change.
- issues.md: no change. (The CODE-vs-DOC `sessionStorage`→Zustand staleness in the spec/auth-contract is a docs-correction candidate, surfaced in "For Mastermind" below — Mastermind/Docs/QA's call, not an issues.md entry authored here.)

## Obsoleted by this session

- nothing (audit only).

## Conventions check

- Part 4 (cleanliness): confirmed — no code written.
- Part 4a (simplicity): N/A — no code introduced. See "For Mastermind" structured evidence.
- Part 4b (adjacent observations): flagged in "For Mastermind".
- Part 6 (translations): N/A this session (no keys added; existing keys catalogued in the audit).
- Other parts touched: Part 7 (error contract) — confirmed; the deletion/ban paths use the codes-only `{errors:[{code}]}` envelope and `isErrorWithCode`. Part 11 (trust boundaries) — confirmed; deletion authority is server-side (Postgres `deletion_status` per auth-contract C-1); the client only mints a fresh token and reads response flags.

## Known gaps / TODOs

- The audit is scoped to web (the parity reference). It does not assert anything about `oglasino-expo`'s current state.
- `InfoDialog`'s internal `autoDismissAfterMs` timer implementation was not separately quoted (the init component only supplies the 10_000ms value); not needed for parity since mobile reimplements the timer.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- **Adjacent observations (Part 4b):**
  - **Docs drift (medium — could mislead a future reader/mobile engineer):** `user-deletion.md` §14.3 step 4, §4.1 step 4, §4.7, §14.10 and `user-deletion-auth-contract.md` C-5 step 2 + the §5 timeline rows still describe `sessionStorage.setItem('account-just-deleted')` / `('account-banned')` and "reads sessionStorage", but the as-built mechanism is Zustand store flags on `useAuthStore` (spec §14.4 already documents the replacement). Recommend a Docs/QA correction pass so the stale sessionStorage prose doesn't get mirrored into mobile. I did not change docs (sibling repo, config files — out of my write scope).
  - **`AuthUserDTO.allowPreferenceCookies?` (low):** still declared (`AuthUserDTO.ts:13`). This matches the existing open issues.md entry (2026-05-30 "Web `allowPreferenceCookies` dead type fields"); noted only so it isn't re-surfaced.
- Suggested next step: Mastermind feeds this audit (esp. Q4/Q8 store-flag mechanism, Q12 web-only list, and the divergence summary) into the mobile (chat E) user-deletion adoption briefs; the mobile flow mirrors store flags + surface-via-throw, dropping all Q12 web-only behaviors.
- Config-file dependency check (closure gate): none required. The docs-drift item above is a *recommendation* for Docs/QA, not a blocking config-file draft owed by this session.
</content>
