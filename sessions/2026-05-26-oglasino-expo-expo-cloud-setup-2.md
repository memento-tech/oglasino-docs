# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-26
**Task:** Create or rewrite `eas.json` to define three build profiles (`development`, `preview`, `production`) matching the `APP_ENV`-driven configuration in `app.config.ts`.

## Audit of existing state

- **`eas.json` existed** at repo root with two profiles (`development`, `production`). Missing `preview`. Minimal config — no image registry, no cache config, no channels, no fields outside the target shape.
- **`cli.version`** was `>= 16.32.0`. Installed EAS CLI is 19.1.0.
- **`cli.appVersionSource: "remote"`** already present.
- **`submit.production: {}`** already present.
- **Bundle IDs in `app.config.ts`** confirmed: `com.oglasino.development` (dev), `com.oglasino.preview` (preview), `com.oglasino` (production). Matches brief.
- **Expo SDK 54** in `package.json` (`"expo": "~54.0.33"`). Compatible with EAS CLI 19.x.
- **No `development-simulator` profile** existed. Not invented per brief instructions.

## Implemented

- Rewrote `eas.json` with three build profiles per the brief's target shape
- Updated `cli.version` from `>= 16.32.0` to `>= 19.0.0` to match installed EAS CLI 19.x
- Added `preview` profile: `distribution: "internal"`, `APP_ENV: "preview"`, `ios.resourceClass: "m-medium"`, `android.buildType: "apk"`
- Extended `development` profile: added `distribution: "internal"`, `ios.resourceClass: "m-medium"`, `ios.simulator: false`, `android.buildType: "apk"`
- Extended `production` profile: added `distribution: "store"`, `ios.resourceClass: "m-medium"`, `android.buildType: "app-bundle"`, `autoIncrement: true`
- Preserved existing `cli.appVersionSource: "remote"` and `submit.production: {}`

## Files touched

- eas.json (+30 / -14)

## Tests

- JSON syntax validation: `python3 -m json.tool` — valid
- `eas.json` location confirmed at repo root alongside `package.json` and `app.config.ts`
- `eas build:inspect --profile development --platform android`: profile resolved, `APP_ENV` loaded from env config, versionCode initialized
- `eas build:inspect --profile preview --platform android`: profile resolved, `APP_ENV` loaded from env config, versionCode initialized
- `eas build:inspect --profile production --platform android`: profile resolved, `APP_ENV` loaded from env config, `autoIncrement` working (versionCode incremented from 1 to 2)
- All three inspections show "Build failed" after the pre-build stage — expected; this is the native project generation / credentials step, not an eas.json parsing failure. No credentials are configured yet (separate brief).
- `npx tsc --noEmit`: clean (zero errors)

## Cleanup performed

- none needed (eas.json is the only file touched; no code changes)

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- The two-profile `eas.json` (missing `preview`, missing `distribution`/`resourceClass`/`buildType`/`autoIncrement` settings, outdated `cli.version`) — overwritten in this session.

## Conventions check

- Part 4 (cleanliness): confirmed — no code changes, JSON file only
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): N/A — only `eas.json` touched, no adjacent code files read
- Part 6 (translations): N/A this session
- Other parts touched: none

## Known gaps / TODOs

- Credentials are not configured — managed by EAS Cloud, separate brief per the task instructions.
- No `.env.preview` file exists. When EAS builds with `APP_ENV=preview`, Expo falls back to `.env.development` (stage Firebase credentials — correct for preview). If preview needs distinct env vars, a `.env.preview` file must be created (separate brief).
- `GoogleService-Info.prod.plist` and `google-services.prod.json` don't exist on disk. Config references them for the production profile. File placement is a separate brief.
- EAS Updates / channel configuration not included — out of scope per the brief.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — the eas.json shape is a direct transcription of the brief's target with no abstraction or indirection.
  - Considered and rejected: nothing — eas.json is a declarative config file with no design decisions beyond the target shape.
  - Simplified or removed: nothing — the previous eas.json was minimal; this session extended it rather than simplifying.
- **`autoIncrement` on production incremented versionCode during inspect.** The `eas build:inspect` command for the production profile incremented the remote versionCode from 1 to 2. This is a side effect of the inspect command reading the remote version source. The increment is harmless (version 2 is the correct next version for any future production build), but worth noting: `eas build:inspect` with `autoIncrement: true` is not purely read-only.
- Nothing else flagged.
