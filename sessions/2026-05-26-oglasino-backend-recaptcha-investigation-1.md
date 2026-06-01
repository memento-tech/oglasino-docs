# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-26
**Task:** Read-only investigation of the reCAPTCHA verification implementation — sections A–F per brief.

## Implemented

- Read-only audit. No code changes.
- Investigated all six sections (A–F) of the brief.
- Documented findings in `.agent/audit-recaptcha-investigation.md`.

## Files touched

- `.agent/audit-recaptcha-investigation.md` (new, +audit output)

## Tests

- Not run (read-only audit, per brief scope).

## Cleanup performed

- None needed (read-only audit).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change (findings are in the audit deliverable for Mastermind to route)

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): N/A — read-only audit, no code changes.
- Part 4a (simplicity): N/A — no code written; see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): confirmed — no adjacent observations outside reCAPTCHA scope. The `TestCreateJSON.java` anonymous endpoint is already logged in `issues.md` (2026-05-20 entry).
- Part 7 (error contract): the reCAPTCHA controller returns `Map.of("success", validated)` — this does not match the Part 7 error contract (`{errors: [{field, code}]}`). Flagged in audit Finding 1 context.

## Known gaps / TODOs

- None. All six brief sections fully investigated.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit.
  - Considered and rejected: nothing — no code written.
  - Simplified or removed: nothing — read-only audit.

- **Critical: trust boundary violation (Finding 1, section F2).** reCAPTCHA enforcement is client-side only. The `POST /api/secure/products/create` endpoint has no reCAPTCHA verification. Any client bypassing the frontend can create products without solving a CAPTCHA. Per Part 11, this is the same class of violation as the `oldName` client-trust issue. Two fix options detailed in the audit. Recommend Option A (add `recaptchaToken` to `NewProductRequestDTO`, verify server-side before create).

- **Critical: zero observability (Findings 2-4).** Google's `error-codes` are discarded. `RestClientException` is silently swallowed. No logging anywhere in the reCAPTCHA path. Diagnosing Problem 1 is impossible from backend logs alone.

- **Likely root cause of Problem 1:** single-use token consumed on verification, then frontend re-sends the same consumed token on retry → Google returns `success: false` with `error-codes: ["timeout-or-duplicate"]` → backend returns `{ "success": false }` with no diagnostic info. The web audit should confirm whether the frontend regenerates tokens or reuses stale ones.

- **Likely root cause of Problem 2:** frontend/config — site key misconfigured for the stage domain in the reCAPTCHA admin console. Backend-side: confirm `RECAPTCHA_SECRET` value matches the site key configured for stage. The "key is bad" error is rendered by the reCAPTCHA JavaScript widget before any backend call.

- **Secret inventory gap:** `RECAPTCHA_SECRET` and `RECAPTCHA_URL` are not documented in `../oglasino-docs/infra/overview/secret-inventory.md`. Draft for Docs/QA to add.

- **Error contract note:** the reCAPTCHA controller at `ReCaptchaController.java:21` returns `{ "success": true/false }`, not the Part 7 `{ "errors": [...] }` shape. Pre-existing, not introduced by this audit. If reCAPTCHA verification moves server-side (Option A), the error should use the Part 7 contract (422 with error code).
