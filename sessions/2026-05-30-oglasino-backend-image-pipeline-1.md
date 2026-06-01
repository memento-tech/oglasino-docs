# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-30
**Task:** READ-ONLY audit of the backend's image-pipeline surface (the part mobile calls directly), across the 11 areas in the brief, output to `.agent/audit-image-pipeline.md`.

## Implemented

- Read-only audit only — no source code changed. Produced `.agent/audit-image-pipeline.md` structured under the 11 brief headings, every claim grounded in code with `file:line`, spec treated as a claim to verify.
- Inventoried the three endpoints (upload-tokens, view-tokens, DELETE `{*key}`), with DTO field/validation tables, exact as-coded validation order (the DELETE chain pinned by tests: auth → format → rate-limit → ownership → in-use → age → delete), and per-branch HTTP status + error code.
- Covered JWT signing (claims, TTLs, secret startup guard), scope/count/content-type validation, Firestore chat-membership, server-side R2 key construction, Redis ownership + DELETE authorization, rate limiting (before/after validation), entity field semantics, the dual error-envelope contract, the translation-seed state, and the mobile seams.
- Delivered the §10 translation verdict unambiguously and flagged three code-vs-spec divergences (D1–D3) in a rollup at the top of the audit.

## Files touched

- `.agent/audit-image-pipeline.md` (new, +~230 lines) — the single audit deliverable.
- `.agent/2026-05-30-oglasino-backend-image-pipeline-1.md` (this summary).
- `.agent/last-session.md` (exact copy of this summary).

No files under `src/` were modified (read-only audit).

## Tests

- Not run. Read-only audit; no code change. (`./mvnw test` not applicable.)

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change. (Image-pipeline status `web-stable` already reflects this audit's premise; no flip warranted from a read-only audit.)
- issues.md: no change required from me. Three divergences are flagged below in "For Mastermind" for triage; whether any become `issues.md` entries is Mastermind's call.

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no debug output, no dead code, single referenced output file.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): flagged in "For Mastermind" (D1–D3).
- Part 6 (translations): N/A this session for seeding — but the audit's §10 verifies seed state and surfaces 4 active-key-name divergences (D3). No seeds added.
- Other parts touched: Part 7 (error contract) — audited, §9 documents the intentional image-vs-Part-7 envelope split. Part 11 (trust boundaries) — audited, §5/§11 confirm keys server-constructed and auth read from `SecurityContextHolder`; no violation.

## Known gaps / TODOs

- The audit reports line numbers as read by the exploration pass; a couple of ranges (e.g. the DELETE in-use block) were reported slightly differently by two readers (83–86 vs 83–139). I used the more detailed range. Exact line numbers may shift with edits — class/method names are the durable anchors.
- `oglasino-expo` was deliberately not read (brief out-of-scope rule). Seam claims (§11) describe what the backend requires; they do not assert what mobile currently sends.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no abstractions/config/patterns introduced in source.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **Divergences for triage (Part 4b):**
  - **D3 — Translation key names drift from spec (severity: medium).** All 17 image keys are seeded ×4 languages (68 rows) in `0001-data-web-translations-*.sql`, but 4 of the 10 active keys are named differently than the spec: `image.processing.converting-heic`→`converting_heic`; `image.processing.complete`→`...complete.label`; `image.uploading`→`image.processing.uploading.label`; `image.uploading.with.size`→`image.processing.uploading.with.size`. Likely Part 6 Rule 2 (parent/child collision) renames — `image.processing.complete` would collide if a child existed, hence `.label`. **Impact:** if web's inline `TODO(phase 7)` fallbacks or the forthcoming mobile adoption hard-code the spec's literal key strings, lookups miss. Recommend the canonical spec's "Translation keys" table be reconciled to the seeded names (a Docs/QA edit to `features/image-pipeline.md`, which is a feature spec, not one of the four config files). I did not change anything — out of scope and cross-repo.
  - **D1 — `REPORT` scope rejected late, not in the validation fence (severity: low).** `ImageScope.from()` accepts `REPORT`; an upload-tokens request with `scope:"report"` passes all named validations and is only rejected in `buildKey()` (`DefaultImageTokensFacade` ~230–232), re-using `INVALID_SCOPE` (400). Functionally fine (still 400), but the rejection lives outside the validation gate. View-tokens reject it earlier. Cosmetic/structural; flag for awareness, not a user-facing bug.
  - **D2 — Dual error envelope (severity: informational, by design, but load-bearing for mobile).** Image endpoints emit `{"error":{code,message,details,retryable}}` (no `field`, no `translationKey`); everything else emits Part 7 `{"errors":[{field,code,translationKey}]}`. This is intentional and documented in `GlobalExceptionHandler`. It is the single most important fact for mobile's Φ4 `parseServiceError`: image errors must be parsed by a different branch and i18n keyed off `error.code` → the `image.*` reserved keys, since the backend sends no `translationKey` for image errors.

- **Config-file impact restatement:** no edits to `conventions.md`/`decisions.md`/`state.md`/`issues.md` are required from this session. The only documentation reconciliation I'd recommend (D3) targets the `features/image-pipeline.md` feature spec, owned by Docs/QA — drafted above for routing. No implicit config-file dependency is left unstated.
