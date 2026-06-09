# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-06-06
**Task:** Read-only audit. Confirm the blast radius of removing the account-wide `allowNotifications` flag. Cross-check every file:line with grep/cat. Write findings to `.agent/audit-notifications-toggle-removal.md`.

## Implemented

- Read-only Phase-2 audit; no source changed. Wrote findings to `.agent/audit-notifications-toggle-removal.md`.
- **Key finding (Q2):** the push/in-app dispatch path does **not** gate on `allowNotifications`. `DefaultNotificationsService.sendAsyncNotification`/`sendAsyncPushOnly` load tokens by userId and `fanOutPush` unconditionally — token presence is the only gate. The flag is a stored preference with zero behavioral consumer.
- Mapped the full reader set: entity field (`User.java:112`), V1 column (`V1__init_schema.sql:590`, `NOT NULL`, no DB default), 4 SQL seed inserts, 2 wire DTOs (`UpdateUserDTO` = profile GET+POST, `AuthUserDTO` = login response), 2 ModelMapper converters, the facade write (`DefaultUserFacade.java:188`), and the dev/test import path (`TestUsersImportService.java:68`).
- Confirmed push-token endpoints: attach = upsert-by-token-value (one row), detach = single ownership-scoped delete by token value — no cascade/bulk.
- Listed the exact compile-time + seed breakage and the net removal set for a future implementing brief. No backend test references the flag (zero `src/test` hits).

## Files touched

- `.agent/audit-notifications-toggle-removal.md` (new, audit deliverable)
- `.agent/2026-06-06-oglasino-backend-notifications-toggle-removal-1.md` (this summary)
- `.agent/last-session.md` (copy of this summary)
- No source files touched (read-only audit).

## Tests

- Ran: none (read-only audit, no code change).
- Result: N/A
- New tests added: none

## Cleanup performed

- none needed

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change (audit is Phase-2 input to Mastermind; no status flip owed by this session)
- issues.md: no change

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): confirmed — no source changed, no debug logging, no stray files.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one observation flagged in "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: Part 10 (feature lifecycle — Phase-2 audit format `audit-<slug>.md`) confirmed; Part 11 (trust boundaries) — verified push-token attach/detach derive userId server-side; Part 12 (V1-fold) — noted in audit for the removal path.

## Known gaps / TODOs

- none. The audit deliberately does not implement the removal — it is read-only Phase-2 input.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- **Headline for seam analysis:** `allowNotifications` is functionally dead — nothing in the dispatch path reads it (Q2, with the full `DefaultNotificationsService` path quoted in the audit). Removing it is low-risk on the backend: 4 Java readers, 1 entity field, 1 V1 column, 4 SQL seeds, 2 wire DTOs. Zero backend tests reference it.
- **Adjacent observation (Part 4b):** `AuthUserDTO` and `UpdateUserDTO` are wire contracts shared with `oglasino-web` and `oglasino-expo`. Dropping `allowNotifications` is a breaking contract change for those repos (auth/profile payload read + the profile-settings toggle that POSTs it). File paths: `dto/AuthUserDTO.java:15`, `dto/UpdateUserDTO.java:22`. Severity: medium (coordinated cross-repo change, not a backend bug). Not actioned — cross-repo, flagged for routing to web/mobile briefs.
- **Note on brief wording:** the brief's preamble says "this is a sibling repo — read only," but every audit question targets `oglasino-backend` code (entity, V1 schema, `DefaultUserFacade`, `DefaultNotificationsService`, push-token endpoints). I read it as a read-only audit of *this* repo and proceeded accordingly. If a different repo was intended, that's outside my scope — flag for Igor.
- **Config-file dependency check (closure gate):** none required. No `conventions.md`/`decisions.md`/`state.md`/`issues.md` edit is implied by a read-only audit.
