# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev (unchanged — no commit/push/checkout)
**Date:** 2026-06-04
**Slug / order:** deep-links / 3
**Task:** Native config for Universal/App Links in `app.config.ts` (two-tier): add `ios.associatedDomains` + `android.intentFilters`, tier-branched (prod `oglasino.com`, preview `stage.oglasino.com`, dev omitted), apex-only. Config edit explicitly authorized by this brief; no `eas.json`, no native-file hand-edit, no builds/`eas`/prebuild.

## Implemented

- Added a `universalLinkHost` derivation directly below the existing `appScheme` tier ternary, using the **same** `ENV` (`APP_ENV`) signal: production → `'oglasino.com'`, preview → `'stage.oglasino.com'`, development → `undefined`. No parallel env mechanism introduced.
- Built both native values from that single host so prod/preview stay in lockstep and dev gets neither:
  - `associatedDomains = universalLinkHost ? ['applinks:<host>'] : undefined` → wired into the `ios` block.
  - `androidIntentFilters = universalLinkHost ? [{ action: 'VIEW', autoVerify: true, data: [{ scheme: 'https', host: <host> }], category: ['BROWSABLE', 'DEFAULT'] }] : undefined` → wired into the `android` block.
- Dev tier yields `undefined` for both, so Expo omits the keys from generated native config (custom-scheme only preserved) — matching the brief's "development → omit."
- Apex host only; no `www` entry (worker 301s www→apex; verifiers don't follow redirects, so associated domains must point at the host serving `.well-known` directly).

## Files touched

- `app.config.ts` (+24 / -0)
- `.agent/2026-06-04-oglasino-expo-deep-links-3.md` (this summary)
- `.agent/last-session.md` (exact copy)

No other source/config/native files modified. The pre-existing uncommitted diffs (`LoginDialog.tsx`, `utils.ts`, `utils.test.ts`) and the prior deep-links-2 additions (`app/+native-intent.ts`, `redirectSystemPath.ts/.test.ts`) are not mine — untouched.

## Tests

- `npx tsc --noEmit` — exit 0 (the config is TS; this is the primary gate for an `app.config.ts` change).
- `npx eslint app.config.ts` — exit 0 (clean; lint baseline held — added no warnings/errors).
- `npx expo-doctor` — 18/18 checks pass. Notably it evaluated the config under the **dev** tier (`APP_ENV=development` from `.env.development`), where both new values are `undefined` — confirming they serialize cleanly and the config stays valid with the keys omitted.
- No runtime test exercises `app.config.ts`; the full vitest suite (40 files / 457 tests as of deep-links-2) is unaffected by a build-time config change. Cross-check from the brief satisfied: tier conditional shape followed precisely; no existing `associatedDomains`/`intentFilters` key existed to merge with (verified on disk before editing).

## Cleanup performed

- none needed (additive config; no debug logging, no dead code, no commented-out blocks, no TODO/FIXME; the one comment block explains *why* — apex-only / www-redirect rationale — per Part 4a).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- issues.md: no change
- state.md: no change written. This session delivers the **config only**; it is inert until a production/preview-profile prebuild + rebuild regenerates the native entitlement (iOS `com.apple.developer.associated-domains`) and intent-filter (Android `<intent-filter android:autoVerify="true">`), which is out of scope here and the current on-disk build is dev-tier. The deep-linking feature still depends on cross-repo blockers (web/router serving `.well-known` per tier, release SHA-256). So **no Expo-backlog row should be removed/promoted.** If Docs/QA tracks a deep-linking row, the native-config line item can be marked done; the row stays open pending the prebuild+build (Ψ) and the `.well-known` hosting. (Closure gate: no implicit config-file dependency from this session beyond this note.)

## Obsoleted by this session

- Nothing. (Additive; the deep-links-2 `+native-intent` locale-strip hook is the JS complement to this native config — both are now in place, neither obsoletes the other.)

## Conventions check

- **Part 4 (cleanliness):** confirmed — no console/debug logging, no dead code, no commented-out blocks, no unused vars, no TODO/FIXME. tsc + eslint + expo-doctor green.
- **Part 4a (simplicity):** see structured evidence in "For Mastermind."
- **Part 4b (adjacent observations):** one carried forward (the prod-hardcoded share host in `utils.ts:136`, already in issues.md) — see "For Mastermind." Nothing new.
- **Part 6 (translations):** N/A — no user-facing strings.
- **Other parts touched:** none. Native config only; no error contract, no trust boundary.
- **Hard rules:** no commit/push/checkout; only `app.config.ts` edited (no `eas.json`, no native-file hand-edit); no builds/`eas`/prebuild run; no cross-repo edits. Expo `AndroidIntentFilter`/`associatedDomains` shapes verified directly against `node_modules/@expo/config-types/build/ExpoConfig.d.ts` (lines 492, 792-800), not assumed from docs.

## Known gaps / TODOs

- **Cannot verify under this brief (by design).** The entitlement/intent-filter are native and only take effect after a production/preview-profile prebuild + rebuild (out of scope). On-device verification is a later Ψ step, and additionally gated on the **router/web serving `.well-known` per tier** — `apple-app-site-association` (iOS) and `assetlinks.json` (Android), `application/json`, no redirect, on `oglasino.com` and `stage.oglasino.com` respectively. Android `autoVerify` also needs the release-cert SHA-256 registered in `assetlinks.json` (EAS/Igor). None of these are mobile-side.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): a single `universalLinkHost` tier const + two derived values (`associatedDomains`, `androidIntentFilters`). Earned: the host varies per tier (config, not constant); deriving both native values from one host keeps prod/preview in lockstep and gives dev `undefined` in one place — no duplicated magic strings, matching the brief's cleanliness requirement.
  - Considered and rejected: (a) inlining the host into two separate ternaries (would duplicate `'oglasino.com'`/`'stage.oglasino.com'` across iOS+Android — rejected for the single-source reason above); (b) conditional object spread (`...(associatedDomains && { associatedDomains })`) to "omit" on dev — rejected because assigning `undefined` to the optional key is what Expo already expects (omits it from generated native config) and reads consistently with the file's existing "compute a tier value above, reference it below" pattern.
  - Simplified or removed: nothing (additive config).
- **Tier coverage note:** the brief specifies preview → `stage.oglasino.com` (audit-deep-links-2 had earlier reasoned "production only" for v1). I followed the **brief** — both prod and preview claim their apex; dev omitted. Flagging only so the divergence from the audit's earlier framing is on the record; the brief is the authority and this is correct as written.
- **Part 4b carry-forward (unchanged, not fixed — out of scope):** share-URL host is prod-hardcoded across all tiers — `getNormalizedProductUrl` (`utils.ts:136`) returns `https://oglasino.com` regardless of tier, unlike the tier-aware forgot-password URL builder. Already in issues.md (2026-06-03, medium, open). Relevant to a universal-link rollout because a preview build's "open in app" share URL would point at the prod apex (`oglasino.com`), not `stage.oglasino.com` — i.e. a preview-tier share link could deep-link mismatch. Severity: medium (could mislead during stage testing). I did not fix it — out of scope for this config-only brief; the natural fix is to unify the two URL builders on the same tier host this session just introduced.
- **Ψ verification dependency chain (for whoever runs the on-device pass):** prod/preview prebuild+build → `.well-known` hosted per tier (web/router) → Android `assetlinks.json` carries the release SHA-256 → then verified `https://<apex>/{locale}/product/...` opens the app and the deep-links-2 `+native-intent` hook strips the `{locale}` segment. All four must be in place; this session delivers only the first prerequisite's config.
