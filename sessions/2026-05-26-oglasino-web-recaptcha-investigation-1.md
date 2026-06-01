# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-26
**Task:** Read-only audit of reCAPTCHA frontend integration — investigate two problems (dev: "not verified" after successful solve; stage: "key is bad" widget error)

## Implemented

- Read-only audit covering sections A–G per the brief. No code changes.
- Output written to `.agent/audit-recaptcha-investigation.md`.

## Files touched

- None (read-only audit).

## Tests

- Ran: none (per brief: "No tests run")
- Result: N/A
- New tests added: none

## Cleanup performed

None needed (read-only audit).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

Nothing.

## Conventions check

- Part 4 (cleanliness): N/A — read-only audit, no code changes
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): five observations flagged in "For Mastermind"
- Part 6 (translations): N/A this session
- Part 11 (trust boundaries): confirmed reCAPTCHA token is sent to backend for verification; no dev-mode bypass; architectural note on decoupled verification recorded in audit

## Known gaps / TODOs

- None. Audit is complete per the brief's scope.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit)
  - Considered and rejected: nothing (read-only audit)
  - Simplified or removed: nothing (read-only audit)

- **Key findings for seam analysis:**

  1. **Problem 1 (dev — "not verified") is almost certainly backend-side.** The frontend obtains a fresh token on every `execute()` call, resets the widget immediately after `executeAsync()` resolves, and sends the token to the backend in a dedicated `POST /public/verify-recaptcha` call. Token-reuse, token-expiry, and missing-reset are all ruled out on the frontend. The most likely root causes: site-key / secret-key mismatch in the reCAPTCHA admin console, wrong `RECAPTCHA_SECRET` env var on the backend, or the backend's Google `siteverify` call failing for a configuration reason.

  2. **Problem 2 (stage — "key is bad") is a configuration issue.** The frontend uses a single `NEXT_PUBLIC_RECAPTCHA_SITE_KEY` env var with no environment-specific logic. The stage Vercel project's value for this env var either (a) doesn't exist / is the "dummy" build placeholder, (b) is a key not registered in the Google reCAPTCHA admin console, or (c) is registered but without the stage domain whitelisted.

  3. **The library is v2 invisible, not v3 score-based.** `react-google-recaptcha` ^3.1.0 is the v2 library (`size="invisible"`). The `.env.local.example` comment incorrectly says "reCAPTCHA v3." This matters for the backend audit — the backend should be calling the v2 `siteverify` API, not the v3 score-based API.

  4. **reCAPTCHA verification is architecturally decoupled from business endpoints.** The frontend verifies the token via `POST /public/verify-recaptcha`, then makes a separate call to the business endpoint (e.g., `POST /secure/products/create`). The backend business endpoints do not check whether reCAPTCHA was verified. This is currently mitigated by Firebase auth on `/secure/*` endpoints, but the reCAPTCHA-protected public flows (`suggest`, `report`) have no server-side enforcement that the reCAPTCHA step actually ran.

- **Adjacent observations (Part 4b):**
  1. `.env.local.example:48` — "reCAPTCHA v3" comment is wrong; library is v2. Severity: low. Out of scope.
  2. `NEXT_PUBLIC_RECAPTCHA_SITE_KEY` and `RECAPTCHA_SECRET` not in `secret-inventory.md`. Severity: low. Out of scope (other repo).
  3. `MetaDataProductDialog.tsx` — no loading/disabled guard during async reCAPTCHA flow (only consumer without one). Severity: low. Out of scope.
  4. Decoupled verification architecture. Severity: medium (see finding 4 above). Out of scope.
  5. `ProductReviewDialog.tsx:51,79` — caches reCAPTCHA result in state; skips re-execution on retry. Severity: low. Out of scope.

- Nothing else flagged.
