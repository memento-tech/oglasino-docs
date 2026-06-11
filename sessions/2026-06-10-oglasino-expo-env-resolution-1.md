# Session summary

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-06-10
**Task:** Determine, definitively, where the PRODUCTION and PREVIEW builds get their `EXPO_PUBLIC_*` variables (Firebase appId, GA4, etc.) — EAS environment vars, a committed .env file, or eas.json `env`. (READ-ONLY investigation, no fixes.)

## Implemented

- Nothing changed on disk. This was a READ-ONLY investigation per the brief. Below is the evidence-backed conclusion.

### Step 1 — eas.json build profiles
`eas.json` (52 lines, no `extends`/`base`/shared profile). **No `"environment"` field on any profile** (`rg '"environment"' eas.json` → none).
- **preview** (`eas.json:21-33`): `distribution: internal`, `channel: preview`, `env: { APP_ENV: "preview" }`.
- **production** (`eas.json:34-47`): `distribution: store`, `channel: production`, `env: { APP_ENV: "production" }`, `autoIncrement: true`.
- Only var injected by either profile is `APP_ENV`. Because neither sets `"environment": "preview"/"production"`, no EAS-hosted environment variables are bound to these profiles.

### Step 2 — .env* files present
`ls .env*`: `.env.development`, `.env.preview`, `.env.production`, `.env.example`. No base `.env` / `.env.local`.
- All three real env files contain `EXPO_PUBLIC_FIREBASE_APP_ID` (line 19) and GA4 keys `EXPO_PUBLIC_GA4_MEASUREMENT_ID` / `EXPO_PUBLIC_GA4_DEBUG_MODE` (lines 25-26).
- `.env.production` read in full: values are **real and complete**, not placeholders — prod Firebase project `oglasino-prod-7e5db`, all six Firebase vars + `EXPO_PUBLIC_WEB_CLIENT_ID` + both GA4 keys populated.
- `.env.example` (tracked) holds the same keys with `dummy` values.

### Step 3 — gitignore / easignore
No `.easignore` exists. `.gitignore:33-36`:
```
# local env files
.env*.local
.env.*
!.env.example
```
`git check-ignore -v`:
- `.env.development` → `.gitignore:35:.env.*` (ignored)
- `.env.preview` → `.gitignore:35:.env.*` (ignored)
- `.env.production` → `.gitignore:35:.env.*` (ignored)
- `.env.example` → not ignored / tracked (only `.env.example` appears in `git ls-files`).

→ The three real env files are gitignored, and with no `.easignore` re-include, EAS Build does **not** upload them to the cloud build server. Only `.env.example` (dummy values) is in the repo.

### Step 4 — NODE_ENV / APP_ENV wiring
`rg NODE_ENV` → not set anywhere (repo, eas.json, app.config.ts). `rg APP_ENV`:
- `app.config.ts:4` reads it: `const ENV = (process.env.APP_ENV as AppEnv) ?? 'development'`.
- `eas.json:11/25/38` sets it per profile; `commands.json:5-6` sets it for local prebuild.
- Expo's dotenv loader keys off **`NODE_ENV`**, not `APP_ENV`/profile. EAS bundling defaults `NODE_ENV=production`, so it would only look for `.env.production` (+ `.env`, `.env.local`) — never `.env.preview` (there is no `NODE_ENV=preview`; the preview profile sets `APP_ENV=preview`). So `.env.preview` is inert to dotenv even if it were uploaded.

### Step 5 — app.config.ts consumption
- `firebaseConfig` reads `process.env.EXPO_PUBLIC_*` at `app.config.ts:100-108` (appId at `:104`).
- `src/lib/client/firebaseClient.ts:11-16` reads the same vars and **throws at module load if any required one is missing** (`firebaseClient.ts:31-35`).
- GA4 keys (`EXPO_PUBLIC_GA4_MEASUREMENT_ID`, `EXPO_PUBLIC_GA4_DEBUG_MODE`) are referenced **only in `.env*` files** — no source file consumes them (`.env.production:24` comment confirms: "not currently wired into mobile code"). Mobile GA4 runs through native `@react-native-firebase/analytics` + per-env `google-services`/`GoogleService-Info` files, not these JS vars.

## Deliverable (the brief's required one-liners)

- **PREVIEW:** `EXPO_PUBLIC_*` resolves from **none of the in-repo sources** — no EAS env binding (no `"environment"` field, `eas.json:21-33`), no committed/uploaded `.env` (all `.env.*` gitignored `.gitignore:35`, no `.easignore`; `.env.preview` is never matched by dotenv since `NODE_ENV≠preview`), and `eas.json env` carries only `APP_ENV`.
- **PRODUCTION:** same — no EAS env binding (`eas.json:34-47`), `.env.production` gitignored and not uploaded, `eas.json env` carries only `APP_ENV`.

**Does the PRODUCTION (EAS cloud) build have a complete `EXPO_PUBLIC_*` set at build time?**
**NO** from any source present in the repo. The local `.env.production` is complete and real, but it is gitignored with no `.easignore`, so `eas build --profile production` (`commands.json:3`) never receives it. The only conceivable remaining source is EAS-hosted environment variables, but (a) without `"environment": "production"` on the profile they are not injected, and (b) their existence is not verifiable read-only from the repo. Residual **UNKNOWN** is closable only via `eas env:list --environment production` or the EAS dashboard (outside this read-only scope). If no EAS vars are injected, `firebaseClient.ts:31-35` throws on startup and Firebase/GA4 ship unconfigured. A **local** build (file present, `NODE_ENV=production`) would inline the vars correctly — the gap is cloud-build delivery, not file completeness.

## Files touched

- None (read-only investigation).

## Tests

- Ran: none (no code changed; read-only).
- Result: N/A.
- New tests added: none.

## Cleanup performed

- none needed.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change. (This is an investigation, not a feature adoption — nothing to remove from the Expo backlog table.)
- issues.md: no change authored by me. Two candidate issues drafted in "For Mastermind" for Docs/QA to file if Igor agrees (build-time env delivery gap; Android-only Firebase appId used for both platforms). I do not write issues.md.

## Obsoleted by this session

- nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code written, nothing to clean.
- Part 4a (simplicity): N/A — no code added.
- Part 4b (adjacent observations): two observations flagged in "For Mastermind" (Android-only appId; GA4 keys unconsumed).
- Part 6 (translations): N/A this session.
- Other parts touched: Part 5 (this summary, written to both files). Hard rule honored — no edits, no commits, no config-file writes; every file:line verified with both Read and `rg`/`cat`.

## Known gaps / TODOs

- The single residual UNKNOWN (whether EAS-hosted env vars exist for the production environment) requires `eas env:list --environment production` or the EAS dashboard — outside this read-only, no-`eas env:*` scope. Igor can run it.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- **Primary finding / risk:** The EAS cloud PRODUCTION build has no in-repo source for `EXPO_PUBLIC_*`. `.env.production` is complete locally but gitignored with no `.easignore`, and the production profile has no `"environment"` field to pull EAS-hosted vars. Unless EAS-stored env vars exist for the `production` environment (unverified), the prod build bundles `process.env.EXPO_PUBLIC_FIREBASE_*` as undefined and `firebaseClient.ts:31-35` throws at startup. Same structural gap for PREVIEW.
  - Suggested next step for Igor: run `eas env:list --environment production` (and `--environment preview`) to confirm whether vars are stored on EAS. If they are, add `"environment": "production"`/`"preview"` to the respective profiles in `eas.json` so they're injected (note: `eas.json` edits are a hard-rule no-go for me without an explicit brief). If they are not, the fix is to populate the EAS environment and/or wire delivery — a separate, non-read-only task.
- **Adjacent observation 1 (Part 4b):** `EXPO_PUBLIC_FIREBASE_APP_ID` in `.env.production:19` is an **Android** appId (`1:958252433998:android:...`). `firebaseConfig` (`app.config.ts:100-108`, `firebaseClient.ts:37-44`) uses a single `appId` for both platforms, so an iOS build would carry the Android appId. May be intentional for the Firebase JS SDK web-config shape, but flagging as a possible cross-platform correctness issue.
- **Adjacent observation 2 (Part 4b):** GA4 keys (`EXPO_PUBLIC_GA4_MEASUREMENT_ID`, `EXPO_PUBLIC_GA4_DEBUG_MODE`) exist in all `.env.*` files but are consumed by **no source file** — confirmed by the `.env.production:24` comment. Dead config until GA4 JS adoption lands (if ever; native analytics may make them permanently unused).
- **Draft issues for Docs/QA** (I do not write `issues.md`):
  1. Title: "Expo prod/preview EAS cloud builds have no `EXPO_PUBLIC_*` delivery path." Body: `.env.*` gitignored (`.gitignore:35`), no `.easignore`, no `"environment"` field on `eas.json` profiles → EAS cloud build receives no Firebase/GA4 config; `firebaseClient.ts:31-35` would throw at startup unless EAS-hosted env vars exist (unverified). Evidence in this session file.
  2. Title: "Firebase appId in `.env.production` is Android-only but used for both platforms." Evidence: `.env.production:19`, `app.config.ts:104`, `firebaseClient.ts:41`.
