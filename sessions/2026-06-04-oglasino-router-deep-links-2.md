# Session summary

**Repo:** oglasino-router
**Branch:** stage (unchanged — no commit/push/checkout)
**Date:** 2026-06-04
**Task:** Serve `.well-known` association files directly (two-tier) — `apple-app-site-association` and `assetlinks.json`, short-circuited before maintenance/forward/KV, tier-correct per env, with the maintenance-matrix exemption documented.

## Implemented

- Added a direct-serve short-circuit in `src/index.ts` for `GET /.well-known/apple-app-site-association` and `GET /.well-known/assetlinks.json`. It returns an inline body with `200`, hard-set `Content-Type: application/json`, and `Cache-Control: max-age=3600`. It is **not** routed through `forwardToOrigin`, so no origin redirect and no stage `X-Robots-Tag` attaches.
- Placed the short-circuit **after** the www→apex 301 (`:76-81`) and **before** host classification, the KV reads, the maintenance gate, and the origin forward — the audit's "Option B" placement. We're apex-only, so a www request 301'ing to apex first and being served there is correct.
- Made contents **tier-correct per env**, keyed off the existing `env.ENVIRONMENT === "stage"` signal (computed inline in the branch, since the branch runs before the existing `isStage` at the KV stage) — **not** off the request host: prod serves `44PHQVN8PB.com.oglasino` / package `com.oglasino`; stage serves `44PHQVN8PB.com.oglasino.preview` / package `com.oglasino.preview`.
- Android `sha256_cert_fingerprints`: prod keeps the tracked placeholder `REPLACE_AFTER_PLAY_CONSOLE_SETUP`; stage carries a **placeholder** `REPLACE_AFTER_PREVIEW_KEYSTORE_SETUP` (Igor's instruction this session — the real preview SHA was not yet available, so a clearly-named placeholder rather than an invented value).
- Updated the maintenance-matrix comment block at the top of `src/index.ts` to record the `.well-known` bypass as a deliberate exception, with the one-line why (a 503 or origin redirect during maintenance can de-verify the domain association).
- The 9-element AASA `components` list is a single shared `AASA_COMPONENTS` constant so the prod and stage AASA bodies cannot drift; identifiers are the only per-tier difference.

## Files touched

- src/index.ts (+97 / -0)
- tests/router.test.ts (+144 / -0)

## Tests

- Ran: `npm run lint` (tsc --noEmit) → clean; `npm test` (vitest run) → **54 passed, 0 failed** (was 47; +7 new).
- New tests (in a new `.well-known app-association files (direct-served)` describe block):
  - prod AASA → 200 + `application/json` + `com.oglasino` appID + 9 components, no origin fetch
  - stage AASA → preview appID (tier-correct, env-keyed), no stage `X-Robots-Tag`
  - prod assetlinks → 200 + `application/json` + `com.oglasino` package
  - stage assetlinks → `com.oglasino.preview` package, no `X-Robots-Tag`
  - served with full-lockdown maintenance active (gate bypass; no `X-Oglasino-Maintenance`, no maintenance-origin hit)
  - not redirected on apex (no `Location`)
  - unrelated `.well-known/other.txt` still forwards to FRONTEND_ORIGIN (no regression)

## Cleanup performed

- none needed (net-additive change; no commented-out code, no dead imports/vars, no `console.log`, no TODO/FIXME added).

## Config-file impact

- conventions.md: no change.
- decisions.md: **change needed (drafted below for Docs/QA)** — ownership-shift entry web→router for serving `.well-known` association files. I do not write it; drafted in "For Mastermind."
- state.md: no change required by me. (The deep-links feature's status tracking is Mastermind/Docs/QA's; flagged in "For Mastermind" only as informational.)
- issues.md: **two entries needed (drafted below for Docs/QA)** — (1) the prod Android SHA-256 swap after Play Console setup; (2) the stage preview SHA-256 swap after the EAS preview keystore is registered. Both are placeholders in code today.
- Also flagged for Docs/QA: the `seo-foundation.md:571` reconciliation cited in the brief (not a config file; a feature-spec line).

## Obsoleted by this session

- nothing. This is the first implementation for the deep-links feature; the prior `-deep-links-1` session was a read-only audit and is not obsoleted (this session executes its recommendation).

## Conventions check

- Part 4 (cleanliness): confirmed.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one observation flagged in "For Mastermind".
- Part 6 (translations): N/A this session (no user-facing strings; the JSON bodies are machine-read association files).
- Other parts touched: none.

## Known gaps / TODOs

- **Both Android fingerprints are placeholders.** Prod (`REPLACE_AFTER_PLAY_CONSOLE_SETUP`) and stage (`REPLACE_AFTER_PREVIEW_KEYSTORE_SETUP`). Android App Links verification will NOT pass on either tier until the real SHA-256 values replace the placeholders. iOS AASA is final and complete (Team ID + bundles known). Tracked via the issues.md drafts below.
- AASA `components` use the `/*/` locale wildcard (per brief). It slightly over-claims (matches any first segment, not just the 10 locales), but non-locale first segments aren't valid web URLs, so the over-claim is harmless. No tighter non-enumerating syntax found; using `/*/` as the brief directs.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): `AASA_COMPONENTS` shared constant — earns its place by guaranteeing the prod/stage AASA component lists stay identical (the only legitimate per-tier difference is the appID); duplicating a 9-element list twice is a real drift risk. Six body/path constants (`AASA_PROD/STAGE`, `ASSETLINKS_PROD/STAGE`, two path constants) — these are the "clearly-named constants per tier" the brief asked for, replacing buried magic strings.
  - Considered and rejected: a `buildAasa(appID)` / `buildAssetlinks(pkg, sha)` factory pair — rejected; for two call sites each it adds indirection without removing meaningful duplication once `AASA_COMPONENTS` is already shared. Inlining the four precomputed `JSON.stringify` bodies is simpler and reads top-to-bottom.
  - Simplified or removed: nothing (net-additive change).

- **Brief vs reality:** no discrepancies. The audit-recommended placement, the env signal (`env.ENVIRONMENT`), and the host classification all matched what the brief assumed. Nothing to challenge.

- **Drafted config-file text for Docs/QA (I do not write these):**

  - **decisions.md — new entry (ownership shift web→router):**
    > **2026-06-04 — `.well-known` app-association files are served by the router worker, not the web origin.**
    > `/.well-known/apple-app-site-association` and `/.well-known/assetlinks.json` are now served directly by `oglasino-router` (`src/index.ts`), short-circuited before the maintenance gate, origin forward, and KV reads, and tier-correct per env (prod = `com.oglasino`, stage = `com.oglasino.preview`). Rationale: forwarding to Vercel let a maintenance window 503 them (can de-verify the domain association), passed origin redirects through (verifiers don't follow them), and left Next's dotfile/extensionless serving uncertain. This shifts ownership of these two paths from web to router. Supersedes any prior assumption (incl. `seo-foundation.md:571`) that web serves them.

  - **issues.md — entry 1 (prod SHA swap):**
    > **2026-06-04 — Router: prod `assetlinks.json` carries a placeholder SHA-256 (Android App Links unverified until Play Console setup)**
    > **Repo:** `oglasino-router` · **Severity:** medium · **Status:** open
    > `src/index.ts` `ASSETLINKS_PROD` ships `sha256_cert_fingerprints: ["REPLACE_AFTER_PLAY_CONSOLE_SETUP"]` because the app is not yet in the Play Store. Android App Links verification on `oglasino.com` will fail until the real production signing-cert SHA-256 replaces the placeholder. iOS AASA is final. Fix: swap the placeholder for the real fingerprint once Play Console app signing is set up; a router brief.

  - **issues.md — entry 2 (stage/preview SHA swap):**
    > **2026-06-04 — Router: stage `assetlinks.json` carries a placeholder preview SHA-256**
    > **Repo:** `oglasino-router` · **Severity:** medium · **Status:** open
    > `src/index.ts` `ASSETLINKS_STAGE` ships `sha256_cert_fingerprints: ["REPLACE_AFTER_PREVIEW_KEYSTORE_SETUP"]`. The real preview fingerprint (from `eas credentials` → Android → preview keystore) was not available at implementation time; Igor directed a placeholder rather than an invented value. Android App Links verification on `stage.oglasino.com` for `com.oglasino.preview` will fail until the real preview SHA-256 replaces the placeholder. Fix: swap once Igor provides the value; a router brief (can pair with entry 1).

  - **Adjacent observation (Part 4b):** `2026-06-04-oglasino-router-deep-links-1.md` (the audit) cited the matrix comment block as `:1-43`; after this change the block runs longer (the exemption paragraph was appended before the closing `===`). Low severity, cosmetic line-reference drift in an archived session file — I did not edit the archived summary because it's an immutable record; flagging so any future reader of that audit knows the line range shifted. File: `.agent/2026-06-04-oglasino-router-deep-links-1.md`. Severity: low.

  - **Informational (not a request):** the deep-links feature is now code-complete on the router side (both files served, tier-correct, maintenance-immune) but **blocked on real Android fingerprints** for verification on both tiers, and iOS verification still wants Igor's on-device/AASA-validator check. state.md's deep-links tracking may want this noted; that's Docs/QA's write, drafted nowhere here beyond this pointer.

  - **Closure gate:** the only config-file dependencies this session creates are the three drafts above (1 decisions.md, 2 issues.md). All are drafted here for Docs/QA. No unstated config-file dependency remains.
