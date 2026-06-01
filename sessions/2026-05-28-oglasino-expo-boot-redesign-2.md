# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-28
**Task:** Phase 1 — add per-platform post-build ceiling-POST jobs to `.eas/workflows/preview.yml` and `production.yml`, gated on build success, hitting `$BACKEND_BASE_URL/internal/app/version/ceiling` with the resolved Expo `version` and the `X-INTERNAL-TOKEN` header.

## Implemented

- Added two new custom jobs (`publish_ceiling_android`, `publish_ceiling_ios`) to each of `preview.yml` and `production.yml`. Each new job carries `needs: [build_<platform>]` for the success-gate (no `if:` condition; Phase 0a confirmed skipped/failed upstream blocks downstream) and `environment: preview` / `environment: production` at the job level so `${{ env.BACKEND_BASE_URL }}` and `${{ env.INTERNAL_TOKEN }}` resolve against the right EAS environment.
- Each ceiling job runs three steps: `eas/checkout` (repo on disk), `eas/install_node_modules` (so `npx expo config` can resolve `app.config.ts`'s `expo/config` import), and a POST step that derives the version and curls the backend.
- Version extraction matches the brief verbatim — `npx expo config --json` piped into `node -e` to write the resolved `.version` to stdout. This reads the same value Expo stamps into the binary as `nativeApplicationVersion` (per the `eas.json` `appVersionSource: "remote"` posture, only `versionCode` / `buildNumber` are remote-managed; the semver `version` field continues to flow from `app.config.ts`, so there is no drift).
- POST shape: `POST $BACKEND_BASE_URL/internal/app/version/ceiling` with `X-INTERNAL-TOKEN: $INTERNAL_TOKEN`, `Content-Type: application/json`, body `{"platform":"android|ios","latestVersion":"<resolved version>"}`. Platform strings lowercase. `-sS` keeps curl quiet on success but surfaces transport errors. The token is passed only via the env-derived header — never echoed, never in `-d`, never printed.
- POST failure is non-fatal via `|| echo "ceiling POST failed (non-fatal); <platform> ceiling not updated this build"`. The failure echo contains neither the token nor the header value.

## Files touched

- .eas/workflows/preview.yml (+40 / -0)
- .eas/workflows/production.yml (+40 / -0)

## Tests

- Ran: `npx eas workflow:validate .eas/workflows/preview.yml` — `✔ Workflow configuration YAML is valid.`
- Ran: `npx eas workflow:validate .eas/workflows/production.yml` — `✔ Workflow configuration YAML is valid.`
- No new code; no unit-test suite applies to YAML workflow files.
- The first real EAS build of each tier is the live smoke (see verification gap below); per the brief's hard rule, no real EAS build was triggered.

## Cleanup performed

- None needed. No prior ceiling-hook scaffolding existed in either workflow file; this is a pure addition.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: one drafted Risk Watch row — see "For Mastermind" → "Drafted state.md edit." No edit applied; Docs/QA writes.
- issues.md: no change

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed. No commented-out code, no debug `echo` of secrets, no TODOs added. No formatter/lint passes apply to YAML (the repo's lint targets are TS/JS).
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): one minor flagged in "For Mastermind" — `paths:` filter still references `.env.preview` / `.env.production` after the 2026-05-26 `.gitignore` switch to wildcard `.env.*` + `!.env.example` (those files are gitignored and so cannot trigger a `push` filter match). Low severity; out of scope for this brief.
- Part 6 (translations): N/A — no user-facing strings.
- Other parts touched: Part 3 hard rules — confirmed (no commit, no push, no deploy, no real EAS build, no edits outside `oglasino-expo`, no edits to `app.config.ts`/`eas.json`/native config).

## Known gaps / TODOs

- **Verification gap (stated per brief).** This hook cannot be fully verified without a real EAS build against a deployed backend. Phase 1 done = YAML is schema-valid and correct-by-construction; POST target, headers, body, and platform strings match the Phase 0 backend B2 contract; `BACKEND_BASE_URL` and `INTERNAL_TOKEN` are referenced as EAS env vars with `environment: preview|production` binding the right EAS environment per workflow. The FIRST REAL BUILD of each tier is the live smoke. Recommended Risk Watch row drafted in "For Mastermind."

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity):
    - Two new custom jobs per workflow (`publish_ceiling_android`, `publish_ceiling_ios`). Earned: distinct platform-keyed POSTs (`platform: "android"` vs `"ios"`) plus independent failure semantics — an Android build failure must not block the iOS ceiling write, and vice versa (Brief decision 3).
    - `eas/install_node_modules` step in each ceiling job. Earned: `npx expo config --json` reads `app.config.ts`, which imports `ConfigContext, ExpoConfig` from `expo/config`; that import requires the `expo` package resolved from `node_modules`. Without an explicit install step, the POST step throws `Cannot find module 'expo/config'` and the `|| echo` fallback fires on every build. The brief's pasted recipe at line 31 showed only `eas/checkout` before the run step; the brief also instructed "report what's needed rather than guessing" if node/expo are not available by default. I am reporting this here: `eas/install_node_modules` is required, and I added it. If Mastermind has a reason to keep it out, drop it and the ceiling write becomes a non-fatal no-op on every build until reinstated — easy to revert.
  - Considered and rejected:
    - A single `ceiling-publish` custom job needing both `build_android` and `build_ios`. Rejected: Brief decision 3 explicitly requires platform-independent gating so a one-platform failure doesn't block the other tier's ceiling write.
    - YAML anchors / `&ceiling-steps` reuse across the four near-identical step blocks. Rejected: the four jobs differ only in `platform` literal and the failure-echo string, and EAS Workflows YAML schema acceptance of anchors is undocumented; the duplicated form is schema-valid by direct test (`workflow:validate` accepted it) and trivial to maintain.
    - Adding a retry loop on the curl POST. Rejected: the brief's `|| echo "..."` non-fatal pattern is explicit, and retries would silently extend job duration on a backend outage with no observable upside (the build is already done).
  - Simplified or removed: nothing — pure addition.

- **Verification posture (verbatim per brief):** This hook cannot be fully verified without a real EAS build against a deployed backend. Phase 1 is done in the sense of schema-valid + correct-by-construction. Live smoke is the first preview build (writes the preview ceiling on `oglasino-stage-49abb` via `https://api-stage.oglasino.com`) and the first production build (writes the prod ceiling on `oglasino-prod-7e5db` via `https://api.oglasino.com`).

- **Drafted state.md edit (Risk Watch row, for Docs/QA to apply):**

  ```markdown
  - **Ceiling-hook live smoke pending first preview build and first production build.** The EAS post-build ceiling-POST jobs (`publish_ceiling_android`, `publish_ceiling_ios`) were added to `.eas/workflows/preview.yml` and `.eas/workflows/production.yml` 2026-05-28 in session `oglasino-expo-boot-redesign-2`. YAML is schema-valid (`eas workflow:validate` clean on both files); POST target, headers, body, and platform strings match the Phase 0 backend B2 contract; `BACKEND_BASE_URL` and `INTERNAL_TOKEN` resolve via per-job `environment: preview|production` binding. First real EAS build of each tier is the live smoke. Close this row when the first preview build's ceiling-POST job lands a row on `AppVersionConfig.latestVersion` for both platforms on stage, AND the first production build does the same on prod.
  ```

  Suggested placement: under "Risk watch" in `state.md`, grouped with the other Expo-cloud-setup readiness rows (the existing "Final Android + iOS rebuild pending..." and "Expo cloud setup is uncommitted on `new-expo-dev` branch" rows).

- **Brief vs reality (single substantive point):** The brief's pasted recipe showed only `uses: eas/checkout` before the POST step. As covered in the Part 4a evidence above, `npx expo config --json` will not resolve without `node_modules` present. I added `eas/install_node_modules`. Validation passed on both files. Mastermind should explicitly bless or reject the addition before the first real build (the only way to know definitively whether `eas/install_node_modules` is the right step ID and whether `expo config --json` works correctly in an EAS custom-job runner).

- **Adjacent observation (Part 4b, low):** Both `preview.yml` and `production.yml` still list `.env.preview` / `.env.production` in their `on.push.paths` filter. The 2026-05-26 `.gitignore` switch to wildcard `.env.*` + `!.env.example` means those env files are not tracked in git, so they cannot drive a `push`-path filter match anyway. Not a defect, not actionable in this brief; flagging in case the file-watch list is reviewed in a later cloud-setup chat. File paths: `.eas/workflows/preview.yml:14`, `.eas/workflows/production.yml:14`.

- **Cross-repo verification not requested.** I did not read `oglasino-backend` to confirm the `/internal/app/version/ceiling` endpoint exists or accepts the body shape; per the brief, the Phase 0 backend B2 contract is the source of truth and Mastermind has locked the wire shape. If backend B2 has not landed yet, the first real build's ceiling-POST will 404 and fall to the non-fatal echo — no harm to the build, but the row in `AppVersionConfig` won't be written until both sides are live.

- **Closure gate (verified):** The only config-file dependency this session surfaces is the Risk Watch row drafted above. No `conventions.md`, `decisions.md`, or `issues.md` change required. No Expo backlog row applies (this is in-progress feature work on `expo-boot-redesign`, not a mobile adoption of a `web-stable`/`shipped` feature).
