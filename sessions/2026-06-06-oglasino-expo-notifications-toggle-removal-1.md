# Session summary

**Repo:** oglasino-expo
**Branch:** main
**Date:** 2026-06-06
**Task:** Read-only audit. No code changes. Write findings to `.agent/audit-notifications-toggle-removal.md`. Confirm the dead `allowNotifications` toggle state and how mobile registers push tokens, so removing the web toggle + backend flag doesn't surprise mobile. Cross-check every file:line.

## Implemented

- Read-only audit only — no production code touched. Produced `.agent/audit-notifications-toggle-removal.md` answering the brief's four questions with file:line evidence.
- Q1: confirmed the `allowNotifications` toggle is dead exactly as the 2026-05-30 consent-mode-mobile decision recorded — local `useState` (`user.tsx:50`), never seeded (absent from the profile-fetch effect `user.tsx:67-82`), never sent (absent from change-detection `:97-108` and the `updateUser` body `:178-190`).
- Q2: documented mobile's independent, fully-built push-token path — `attachPushTokenToBackend` → `POST /secure/push/token` (`pushTokenService.ts:3-8`), driven by `registerForPush` (`pushNotificationRegister.ts:13-43`), triggered on the boot/auth path by `PushNotificationsInit` mounted in `AppInit` (`AppInit.tsx:28`) keyed on `user` identity change (`PushNotificationsInit.tsx:157-189`). Independent of any toggle.
- Q3: documented the two-stage permission flow — in-app soft prompt (`SOFT_PUSH_PERMISSION_DIALOG`, gated by user + `Device.isDevice` + 5-day throttle + `canAskAgain`, `PushNotificationsInit.tsx:175-186`) then the OS prompt (`requestPermissionsAsync`, `pushNotificationRegister.ts:22-25`). No gate reads `allowNotifications`.
- Q4: confirmed **no mobile code reads `user.allowNotifications`** — grep returns only two optional DTO declarations + one local `useState`. Backend dropping the field is safe for mobile (optional `?:` fields, zero readers).

## Files touched

- .agent/audit-notifications-toggle-removal.md (new, audit deliverable)
- .agent/2026-06-06-oglasino-expo-notifications-toggle-removal-1.md (new, this summary)
- .agent/last-session.md (overwritten with this summary)

No source files modified (read-only audit).

## Tests

- Not run — read-only audit, zero code changes. (`npm run lint` / `tsc` / `npm test` not applicable; nothing was edited.)

## Cleanup performed

- none needed (read-only).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- Nothing. (The audit observes existing local dead code — the dead `allowNotifications` toggle in `user.tsx` and its two optional DTO declarations — but per the consent-mode-mobile spec `:183` that is deliberately owned by future notifications-feature work, not deleted here, and it is independent of the web/backend removal this audit clears.)

## Conventions check

- Part 4 (cleanliness): confirmed — no code changed; no debug logging, commented code, or stray files introduced (the two new files are the audit deliverable + mandatory session summary).
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one low-severity observation flagged in "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: Part 11 (trust boundary) — confirmed: `allowNotifications` is not a trust-bearing value; the audit notes mobile's opt-in is the OS permission grant + token registration, not a server-stored boolean. Part 10 (lifecycle) — this is a Phase-2-style read-only audit per the brief.

## Known gaps / TODOs

- none.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no abstractions, config, or patterns introduced.
  - Considered and rejected: nothing — no implementation choices to weigh.
  - Simplified or removed: nothing — no code changed.
- **Cross-repo verdict:** removing the web toggle + backend `allowNotifications` flag is safe from mobile's side. Mobile never sends, reads, or seeds the field; both mobile DTO declarations are optional (`?:`), so the narrowed wire shape type-checks with no mobile change. Mobile's notification opt-in is the OS permission grant + Expo push-token registration (`/secure/push/token`), a boot/auth-driven lifecycle decoupled from any user-preference flag — this is the pattern web may want to mirror rather than a server boolean.
- **Adjacent observation (Part 4b), low severity:** `app/owner/user.tsx:50,277-287` (dead `allowNotifications` toggle) and `AuthUserDTO.ts:13` + `UpdateUserDTO.ts:12` (dead optional field declarations) are local dead code on mobile. Severity low (purely cosmetic — renders a no-op switch; the COOKIES `notifications.*` keys it uses still resolve, so no broken-label regression). I did not fix this because it is out of scope for a read-only audit, and the consent-mode-mobile spec (`features/consent-mode-mobile.md:183`) explicitly reserves wiring/removal of this toggle for the future notifications feature. Flagging in case Mastermind wants to fold the cleanup into that feature's mobile brief.
- **Config-file impact:** none required — no `conventions.md` / `decisions.md` / `state.md` / `issues.md` edits surfaced by this audit. Stated explicitly per the closure gate.
