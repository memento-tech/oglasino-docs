# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-06-04
**Task:** READ-ONLY audit — Deep Linking current state + `+native-intent` readiness. Output to `.agent/audit-deep-links-2.md`; audit code fresh, do not read prior `audit-deep-links.md` until findings are formed.

## Implemented

- Phase-2 read-only audit. No source, config, or native files changed.
- Wrote `.agent/audit-deep-links-2.md` answering all 9 brief questions with file:line evidence, a Keystone summary, a Diff-vs-prior-audit section, and a For-Mastermind section.
- Verified native claims directly against generated `ios/OglasinoDev/Info.plist`, `OglasinoDev.entitlements`, and `android/app/src/main/AndroidManifest.xml` (not inferred from `app.config.ts`).
- Verified `+native-intent` / `redirectSystemPath` support against the **installed** `expo-router@6.0.24` build in `node_modules` (not from docs).

## Key findings

- **Custom scheme works per-tier** (config: `oglasino`/`oglasino-preview`/`oglasino-dev`), but the **generated native files on disk carry only the dev tier** (`oglasino-dev`, `com.oglasino.development`); prod `oglasino` scheme is absent until a prod prebuild runs.
- **No locale segment in the app route tree** — app paths are `/product/{id}/{slug}`; web emits `https://oglasino.com/{locale}/product/{id}/{slug}`. This is the central mismatch.
- **`+native-intent` is supported** by installed expo-router 6.0.24; `app/+native-intent.ts` does NOT exist (the unbuilt keystone).
- **Universal/App Links absent** — no `ios.associatedDomains`/entitlement, no Android `autoVerify`/`https` intent-filter host. The `https` entry is in `<queries>` (outbound) only — confirmed independently.
- **Notification engine (`PushNotificationsInit.handleNavigation`)** is the proven `router.push` nav layer with `LHNR` cold-start dedup; all `Linking` usage is outbound; expo-router owns inbound routing.
- **Password-reset (`getForgotPasswordUrl`/`LoginDialog`)** is an outbound, tier-aware funnel to web with NO inbound return path — no `appDeepLink`-equivalent on mobile.
- **`expo-linking` already a dep** (8.0.12) — no new JS package needed.
- Diff vs prior audit: agrees on all substantive points; my additions are precision (dev-only scheme on disk, explicit native-intent support confirmation, Google reversed-client-id scheme, password-reset coverage, prod-hardcoded share host). Flagged the prior audit's "10 routing locales" as unverified by me.

## Files touched

- `.agent/audit-deep-links-2.md` (new, audit deliverable)
- `.agent/2026-06-04-oglasino-expo-deep-links-1.md` (new, this summary)
- `.agent/last-session.md` (overwritten, exact copy of this summary)

(No source, config, or native files modified.)

## Tests

- None run — read-only audit, no code changed. (lint/tsc/test not applicable; no touched code paths.)

## Cleanup performed

- none needed (no code changed).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change written. A deep-linking Expo-backlog row is *drafted* in the audit's "For Mastermind" for Docs/QA to add if the team wants it tracked — not written by me.
- issues.md: no change. Re-surfaced the existing open 2026-06-03 item (prod-hardcoded share host in `getNormalizedProductUrl`, `utils.ts:136`) as relevant to a universal-link rollout; no new entry needed.

## Obsoleted by this session

- Nothing. The prior `.agent/audit-deep-links.md` is not obsoleted — `audit-deep-links-2.md` is an independent re-audit (Phase-2 process: re-verify from code). The two agree; mine adds precision. Igor/Mastermind decide which to carry forward.

## Conventions check

- Part 4 (cleanliness): N/A — read-only, no code/imports/logging added.
- Part 4a (simplicity): see structured evidence in "For Mastermind" (nothing added/considered/simplified — audit only).
- Part 4b (adjacent observations): one re-surfaced (prod-hardcoded share host, already in issues.md) — flagged in the audit, not fixed (out of scope, read-only).
- Part 6 (translations): N/A this session.
- Other parts touched: hard rules (no commit/push/config/native/cross-repo/build) — confirmed; native files verified directly per CLAUDE.md "verify before relying on Read."

## Known gaps / TODOs

- "10 routing locales" (cited by prior audit from `audit-routing-locale-parity.md`) not re-verified — locale set is runtime-derived, not a static list in this repo.
- Universal-link end-to-end behavior not verifiable from code / simulator — on-device (Ψ) once native config + web `.well-known` files land.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit.
  - Considered and rejected: nothing — no implementation decisions.
  - Simplified or removed: nothing — no code changed.
- **Deliverable:** `.agent/audit-deep-links-2.md` is the Phase-2 output; paste back to Mastermind for seam analysis.
- **Keystone confirmed available:** expo-router 6.0.24 supports `redirectSystemPath`; `app/+native-intent.ts` (locale-strip) is the cleanest first in-repo brief, parallelizable with web hosting the two `.well-known` files.
- **Cross-repo blockers:** web must host `apple-app-site-association` + `assetlinks.json`; Android needs the release SHA-256 from EAS (Igor); native config edit (`app.config.ts`) needs an explicit brief.
- **Open product decision:** should an inbound `rs-en` link switch the app's language? Recommend strip-and-route v1, language adoption as a separate explicit decision.
- **Drafted `state.md` Expo-backlog row** (Docs/QA to apply, not written by me): see the audit's "For Mastermind" section verbatim.
- **Closure gate:** no config-file edit was made or is required by this session beyond the drafted (optional) `state.md` backlog row noted above.
