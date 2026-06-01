# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-26
**Task:** Rewrite app.config.ts to support three build profiles (development / preview / production) instead of binary isDev branching.

## Brief vs reality

1. **`apiBaseUrl` in `extra` is dead config**
   - Brief says: extend all `isDev`-branching config to three-way, which implicitly includes `apiBaseUrl`
   - Code says: `src/lib/config/api.ts:13` reads `process.env.EXPO_PUBLIC_API_URL` directly from the environment; the `.env.development` and `.env.production` files already provide per-profile API URLs. `extra.apiBaseUrl` has zero consumers across the codebase (grep confirmed).
   - Why this matters: extending dead config to three-way adds unused code; Part 4 says no unused code left behind.
   - Resolution: removed `apiBaseUrl` entirely instead of extending to three-way.

2. **`GoogleService-Info.prod.plist` and `google-services.prod.json` don't exist on disk**
   - Brief says: "The actual file names you find on disk are ground truth"
   - Code says: only `GoogleService-Info.dev.plist` and `google-services.dev.json` exist at root. The `.prod` variants are absent.
   - Why this matters: informational only. The current config already referenced these absent files before this session. The brief says file management is a separate brief.
   - Resolution: no change — config continues referencing the expected filenames. File placement is out of scope.

3. **Bundle ID changes from `com.oglasino.dev` to `com.oglasino.development`**
   - Brief is explicit about this in the three-profile table. Not a contradiction; noted as a deliberate change.

## Implemented

- Replaced binary `isDev` branching with typed three-way `APP_ENV` switch using `type AppEnv = 'development' | 'preview' | 'production'` and nullish-coalescing default
- Bundle ID / package: `com.oglasino.development` (dev), `com.oglasino.preview` (preview), `com.oglasino` (production)
- GoogleService files: development + preview both point at stage Firebase (`*.dev.*`); production points at prod Firebase (`*.prod.*`)
- Removed dead `apiBaseUrl` from `extra` (zero consumers — `api.ts` reads `EXPO_PUBLIC_API_URL` from env directly)
- Collapsed `androidPackage` into `bundleId` (same value for all profiles, per brief: `const androidPackage = bundleId`)

## Files touched

- app.config.ts (+18 / -12)

## Tests

- Ran: `npx tsc --noEmit`
- Result: clean (zero errors)
- Ran: `APP_ENV=development npx expo config --type public`
  - bundleIdentifier: `com.oglasino.development`, package: `com.oglasino.development`, env: `development`, googleServicesFile: `./GoogleService-Info.dev.plist` / `./google-services.dev.json`, Firebase projectId: `oglasino-stage-49abb`
- Ran: `APP_ENV=preview npx expo config --type public`
  - bundleIdentifier: `com.oglasino.preview`, package: `com.oglasino.preview`, env: `preview`, googleServicesFile: `./GoogleService-Info.dev.plist` / `./google-services.dev.json`, Firebase projectId: `oglasino-stage-49abb`
- Ran: `APP_ENV=production npx expo config --type public`
  - bundleIdentifier: `com.oglasino`, package: `com.oglasino`, env: `production`, googleServicesFile: `./GoogleService-Info.prod.plist` / `./google-services.prod.json`, Firebase projectId: `oglasino-stage-49abb`
- Note: all three runs loaded `.env.development` locally (Expo env file auto-loading in local shell defaults to development). Firebase projectId shows stage for all three in this local context. In actual EAS builds, `APP_ENV` is set per build profile in `eas.json` and the matching `.env.<APP_ENV>` file is loaded, so production builds will pick up `.env.production` with the prod Firebase credentials. The `app.config.ts` logic is correct; the local verification limitation is expected.

## Trust boundary check

No trust boundary surface. `APP_ENV` is a build-time constant set by EAS. No values flow from a request body or from auth into any decision.

## Cleanup performed

- Removed dead `apiBaseUrl` variable and its `extra.apiBaseUrl` field (zero consumers in codebase)
- Removed intermediate `isDev` boolean (replaced by direct `ENV` comparisons)
- Removed intermediate `androidPackage` variable (collapsed into `bundleId` since values are identical)

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- The `isDev` binary branching pattern in `app.config.ts` — replaced by three-way `ENV` switch. Deleted in this session.
- The `apiBaseUrl` hardcoded config in `extra` — dead code, deleted in this session.

## Conventions check

- Part 4 (cleanliness): confirmed — no commented-out code, no unused imports/variables, no console.log, no TODO/FIXME
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): confirmed — no adjacent issues found in `app.config.ts`
- Part 6 (translations): N/A this session
- Other parts touched: Part 8 (architectural defaults) — confirmed, no mobile-specific routes or backend changes

## Known gaps / TODOs

- `GoogleService-Info.prod.plist` and `google-services.prod.json` don't exist on disk. The config references them for the production profile. File placement is a separate brief per the task instructions.
- No `.env.preview` file exists. When EAS builds with `APP_ENV=preview`, Expo will fall back to `.env.development` (which has stage Firebase credentials — correct for preview). If preview needs distinct env vars (e.g., a different API URL), a `.env.preview` file must be created (separate brief per task instructions).

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): `type AppEnv` union type — three-variant type constraint on `APP_ENV` prevents silent misuse with arbitrary strings. One type, one line, used at one site.
  - Considered and rejected: extracting a `getGoogleServicesFile(env, platform)` helper — two call sites with identical two-way logic (production vs not-production) don't earn a function. Inline ternary is clearer.
  - Simplified or removed: collapsed `androidPackage` into `bundleId` (were always identical); removed dead `apiBaseUrl` config; removed intermediate `isDev` boolean.

- **Local verification note:** `npx expo config` in a local shell loads `.env.development` for all three `APP_ENV` values because Expo's env file auto-loading in local context defaults to the development env file. The `app.config.ts` branching on `APP_ENV` is correct — verified by the `extra.env` and `bundleIdentifier` / `package` outputs resolving to the expected tier in all three runs. Firebase `projectId` showing stage for all runs is a local-shell artifact; EAS builds load the correct `.env.<APP_ENV>` file per profile.
