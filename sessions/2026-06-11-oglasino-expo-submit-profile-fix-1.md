# Session summary

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-06-11
**Task:** Fix the empty `submit.production` profile in eas.json that caused `eas submit --profile production --platform ios` to resolve the WRONG bundle (com.oglasino.development instead of com.oglasino) and target a stray "Oglasino Dev" App Store Connect record. Pin the production submit target to the real production App Store Connect app so submit always uploads to the correct app regardless of bundle resolution.

## Implemented

- Replaced the empty `submit.production: {}` block in `eas.json` with a `submit.production.ios` block carrying `ascAppId`, `appleId`, and `appleTeamId`.
- `ascAppId: "6779105068"` is the load-bearing key — it pins `eas submit` to the exact App Store Connect record by numeric ID, so even when submit evaluates `app.config.ts` without `APP_ENV=production` and the bundle ternary falls through to `com.oglasino.development`, the upload still targets the correct production app (bundle `com.oglasino`).
- Scope held tight: only `submit.production.ios` was touched. No `build` profile, no `cli` block, no `env`/`APP_ENV` injected into submit, no `android` submit block (Android submit is a later track, out of scope).

## Files touched

- eas.json (+6 / -1)

## Tests

- Ran: `cat eas.json | python3 -m json.tool` — PASS (valid JSON).
- Ran: `rg ascAppId eas.json` — appears exactly once (count: 1), value `6779105068`.
- lint / `tsc --noEmit` / `npm test` / `expo-doctor`: N/A — the change is a JSON config edit, touches no TypeScript path, and changed no dependencies.

## Verification (per brief)

- JSON valid: **PASS**.
- Full `submit` block after edit:
  ```json
  "submit": {
    "production": {
      "ios": {
        "ascAppId": "6779105068",
        "appleId": "igor.stojanovic.nis@icloud.com",
        "appleTeamId": "44PHQVN8PB"
      }
    }
  }
  ```
- `ascAppId` appears exactly once and reads `6779105068`: **confirmed** (file read + rg, count: 1).
- Independent corroboration of the ascAppId fact: `state.md` (last-updated 2026-06-11) records the iOS force-update store URL as `https://apps.apple.com/app/id6779105068` (Apple ID `6779105068`, bundle `com.oglasino`) — the same numeric ID as the brief's production `ascAppId`.

## Cleanup performed

- none needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — single config edit, no dead code/imports/logging introduced.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one observation flagged in "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: none.

## Known gaps / TODOs

- Android submit block deliberately not added (out of scope per brief; Android submit is a later track).

## For Mastermind

- This change fixes a real submit misfire caught mid-submission: `eas submit --profile production --platform ios` resolved the wrong bundle (`com.oglasino.development`) and was targeting a stray "Oglasino Dev" App Store Connect record. **No production data was uploaded to the wrong record** — the submit was aborted at the API-key prompt before any upload occurred. The `ascAppId` pin prevents recurrence by binding submit to the production ASC app by numeric ID regardless of how the bundle identifier resolves.

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): the three `submit.production.ios` keys (`ascAppId`, `appleId`, `appleTeamId`) — each is a concrete deployment value required by `eas submit` to target the correct ASC app; `ascAppId` is the one that actually fixes the bug.
  - Considered and rejected: injecting `env: { APP_ENV: "production" }` into the submit profile to force correct bundle resolution — rejected because the brief is explicit that the `ascAppId` pin is the correct fix, and pinning by numeric ID is more robust than relying on config-time bundle resolution at submit time.
  - Simplified or removed: nothing.

- **Adjacent observation (Part 4b):** The hardcoded `appleId` (`igor.stojanovic.nis@icloud.com`) and `appleTeamId` (`44PHQVN8PB`) now live in `eas.json`, which is committed to the repo. These are account/team identifiers, not secrets (Apple Team IDs and Apple account emails are not credentials, and storing them in `eas.json` is the standard EAS pattern), so this is not a secret-leak per conventions Part 1 "Sensitive data." Flagging only for awareness. File: `eas.json:53-54`. Severity: low. I did not change this because it is the prescribed fix.

- Hard-rule note: editing `eas.json` is normally forbidden by this repo's CLAUDE.md without explicit brief instruction. The brief explicitly instructs this edit, naming the file and the exact target block, which satisfies the exception. No commit/push performed; change is staged on disk on `dev` only.
