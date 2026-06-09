# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-06-06
**Task:** Read-only audit. No code changes. Write findings to `.agent/audit-notifications-toggle-removal.md` — establish whether removing the "Allow notifications" toggle would also remove the only path that registers a web push token.

## Implemented

- Nothing implemented — this was a read-only audit. No source files were modified.
- Produced `.agent/audit-notifications-toggle-removal.md` answering all five brief questions, each with cross-checked `file:line` references.
- Headline finding: removing the toggle would **not** remove the only push-token registration path. The auth/boot path (`UseTokenRefresh.tsx:103`, the sole `onIdTokenChanged` listener, mounted app-wide via `AppInit` → root layout) calls `initPushForAuthenticatedUser()` on every login/sign-up/token rotation, independent of the toggle. The browser permission prompt also already fires at boot when permission is `default`.
- `detachPushToken` has a second non-toggle call site: `logout()` (`useAuthStore.ts:250`).
- `allowNotifications` is read nowhere on web beyond the settings screen (and two DTO type declarations); nothing uses it to gate push behavior on the client.

## Files touched

- `.agent/audit-notifications-toggle-removal.md` (new, audit deliverable)
- No source files changed.

## Tests

- Not run — read-only audit, no code changed, no touched source paths to lint/typecheck/test.

## Cleanup performed

- none needed (no code changed)

## Config-file impact

- conventions.md: no change
- decisions.md: no change. (The removal decision itself, if taken, would warrant a decisions.md entry — but that is Mastermind/Docs/QA's call, drafted post-decision, not this audit's output.)
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): confirmed — no code added, nothing to clean.
- Part 4a (simplicity): N/A — no code written.
- Part 4b (adjacent observations): two non-scope observations noted below in "For Mastermind" (misspelled state setter; un-awaited side-effect call). Neither was changed (read-only audit).
- Part 6 (translations): N/A this session.
- Other parts touched: none.

## Known gaps / TODOs

- Whether the **backend** gates push delivery on `user.allowNotifications` was not determined — it is out of this repo's scope. Flagged for Mastermind/Backend before any removal proceeds.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing
  - Considered and rejected: nothing
  - Simplified or removed: nothing
- **Decision input:** Removing the toggle is safe with respect to push *registration* — the boot path keeps registering tokens. Two consequences to weigh: (1) the only in-session way to *detach* a token (toggle OFF, `page.tsx:94`) disappears; the remaining detach trigger is logout (`useAuthStore.ts:250`). (2) The web side would stop writing `user.allowNotifications`, and nothing on web reads it — but **Backend must confirm** it isn't relied on server-side for delivery gating before removal.
- **Adjacent observation (Part 4b), not in scope, not changed:** in `app/[locale]/owner/user/page.tsx`, (a) the toggle's state setter is misspelled `setAllowNotifiations` (`:52`, used at `:69`, `:334`); (b) `handleNotificationToggle(allowNotifications)` at `:220` is `async` but called un-awaited and fires on *every* save regardless of whether the toggle value changed — so any profile save re-runs attach (token ON) or detach (token OFF). Both are pre-existing; surfacing only, no change made.
- Nothing else flagged.
