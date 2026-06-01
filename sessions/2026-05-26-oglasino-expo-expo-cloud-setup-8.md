# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-26
**Task:** Extend `app.config.ts` to branch `name` and `scheme` on `APP_ENV` (per-tier display name and URL scheme)

## Implemented

- Added `appName` const branching on `ENV`: `'Oglasino'` (production), `'Oglasino Preview'` (preview), `'Oglasino Dev'` (development)
- Added `appScheme` const branching on `ENV`: `'oglasino'` (production), `'oglasino-preview'` (preview), `'oglasino-dev'` (development)
- Replaced hardcoded `name: 'Oglasino'` with `name: appName` and `scheme: 'oglasino'` with `scheme: appScheme`
- Matched the existing ternary style used by `bundleId`, `iosGoogleServicesFile`, and `androidGoogleServicesFile`

## Files touched

- app.config.ts (+10 / -2)

## Tests

- Ran: `npx tsc --noEmit` — clean (no output)
- `APP_ENV=development npx expo config --type public`:
  - `name: 'Oglasino Dev'`
  - `scheme: 'oglasino-dev'`
- `APP_ENV=preview npx expo config --type public`:
  - `name: 'Oglasino Preview'`
  - `scheme: 'oglasino-preview'`
- `APP_ENV=production npx expo config --type public`:
  - `name: 'Oglasino'`
  - `scheme: 'oglasino'`

All three tiers resolve to the correct per-tier values from the brief's table.

## Cleanup performed

None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

Nothing. The hardcoded `name` and `scheme` values were replaced inline; no files or functions became dead.

## Conventions check

- Part 4 (cleanliness): confirmed — no commented-out code, no unused imports/variables, no console.log, no TODOs
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): N/A — single-file change, no adjacent issues observed
- Part 6 (translations): N/A this session
- Other parts touched: none

## Known gaps / TODOs

None.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): two new derived consts (`appName`, `appScheme`) — each is a three-branch ternary matching the existing `bundleId` pattern. Justified by the brief's requirement to branch display name and URL scheme per `APP_ENV`.
  - Considered and rejected: a `getEnvSpecific(env, devVal, previewVal, prodVal)` helper — explicitly rejected by the brief per Part 4a; one-line ternaries don't earn an abstraction even though there are now five ternaries in the file.
  - Simplified or removed: nothing
- Nothing else flagged.
