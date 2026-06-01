# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-29
**Task:** Backend audit: `ProductErrorCode` / system-code split + REVIEW-report gap (Phase 2, read-only). Output `.agent/audit-system-error-code.md`.

## Implemented

- Read-only audit only; no production code touched. Produced `.agent/audit-system-error-code.md` covering Part A (`ProductErrorCode` inventory + cross-cutting classification + consumers + trust boundaries; `ReportErrorCode` shape; `GlobalExceptionHandler` lookup mechanism + extension point; seed-enforcement test), Part B (`product.system.*` seed-row inventory across 4 locales + full rename impact), Part C (report-submit surface + REVIEW-gap trace + REVIEW-branch requirements), and the two cross-repo seams.
- Inventoried all 47 `ProductErrorCode` constants: 36 product-specific, 11 cross-cutting (7 under `product.system.*`, 4 user/profile-level).
- Confirmed `resolveTranslationKey` is a linear `ProductErrorCode.valueOf` → `ReportErrorCode.valueOf` chain; identified the exact one-block extension point at `GlobalExceptionHandler.java:189` for a third enum.
- Quantified the rename: 28 seed rows + 7 enum fields + 3 hardcoded test-assertion literals = 38 edits over 6 files; the seed-enforcement test self-validates (reads the enum, no literal).
- **Refuted the issue's REVIEW-gap premise** with a verified code trace (see "For Mastermind").

## Files touched

- `.agent/audit-system-error-code.md` (new, +~330 lines) — the audit deliverable
- `.agent/2026-05-29-oglasino-backend-system-error-code-1.md` (this summary)
- `.agent/last-session.md` (exact copy of this summary)

No `src/` files modified (read-only audit).

## Tests

- Ran: none. Read-only audit; no code change to validate. `./mvnw test` / `spotless:check` not applicable (no source edits).
- New tests added: none.

## Cleanup performed

- none needed (no code changed).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change (engineer agents do not write; any decision is Mastermind's to draft).
- state.md: no change required by this session. (Audit may inform a future feature-status entry, but that is Mastermind/Docs-QA's call, not an implicit dependency of this work.)
- issues.md: no change. **Note:** the audit's C.10 finding materially refines the open `issues.md` 2026-05-28 "Review-reporting wire" entry (it claims a 422/misfile; the code produces a 500 at deserialization). I have **not** edited `issues.md` (not my file). Draft correction text is in "For Mastermind" for Docs/QA to apply if Mastermind agrees.

## Obsoleted by this session

- nothing. (Read-only audit produces no dead code. It does surface that the `issues.md` 2026-05-28 REVIEW entry's stated runtime behavior is inaccurate — flagged below, not edited.)

## Conventions check

- Part 4 (cleanliness): confirmed — no code added, no debug logging, no TODOs.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): two flagged in "For Mastermind" (missing `ReportErrorCodeTest`; `resolveTranslationKey` linear-chain trend).
- Part 6 (translations): N/A — no seed rows added. (Audit documents existing seed rows; no new keys.)
- Part 7 (error contract): confirmed/observed — the wire shape `{field, code, translationKey}` and the codes-only contract are documented in the audit (Seam 1); no contract change made.
- Part 11 (trust boundaries): confirmed — all 7 `product.system.*` cross-cutting codes verified clean (none read as a moderation/authz/state-transition decision input); explicit `LANG_MISSING_OR_INVALID` and rate-limit checks done per brief A.6.
- Part 12 (schema): noted — a future REVIEW branch needs the `report_type` CHECK constraint widened in the pre-prod V1-fold (`V1__init_schema.sql:477`); documented in audit C.11, no edit made.

## Known gaps / TODOs

- **Seam 2 is unresolved by design** — the exact serialized `reportType` string web sends on a review report cannot be confirmed from the backend repo. The C.10 conclusion (500 vs 422) hinges on it. Needs the web engineer / Mastermind to confirm. Documented in the audit.
- The four user/profile cross-cutting codes (`USER_SETUP_INCOMPLETE`, `DISPLAY_NAME_*`) were classified but their full consumer map and seed rows were not exhaustively inventoried — they sit outside the `product.system.*` rename set. If Mastermind folds them into a `SystemErrorCode`/`UserErrorCode` migration, that inventory is a follow-up.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code or abstractions introduced.
  - Considered and rejected: nothing — no implementation decisions were in scope.
  - Simplified or removed: nothing.

- **🔴 Brief vs reality — Part C.10 (the load-bearing finding).** The brief (sourced from `issues.md` 2026-05-28) asks me to confirm that a `reportType=REVIEW` request runs `productRepository.getOwnerId(<reviewId>)`, producing a collision-misfile or `REPORTED_PRODUCT_NOT_FOUND` 422.
  - **Brief/issue says:** backend runs the PRODUCT existence check against the review id → misfile (collision) or 422 (no collision).
  - **Code says:** `ReportType` = `{PRODUCT, USER}` only (`entity/ReportType.java`). No Jackson enum-leniency config in any profile, no custom deserializer, and `GlobalExceptionHandler` is a plain `@RestControllerAdvice` with no `HttpMessageNotReadableException` arm (verified by direct read: hierarchy + catch-all at `GlobalExceptionHandler.java:164–169`). So `reportType:"REVIEW"` fails Jackson deserialization → `HttpMessageNotReadableException` → catch-all → **HTTP 500 `INTERNAL_ERROR` (`product.system.internal_error`)**. The PRODUCT branch never runs; `getOwnerId` is never called; there is no 422 and no misfile.
  - **Why it matters:** the feature framing changes. Today's symptom is a generic 500 with an ERROR-logged stack trace (looks like a server bug), not a quiet mis-file or a clean 422. That affects both the urgency framing and the "remove the wire vs build it vs repurpose" decision in the `issues.md` entry's options (a)/(b)/(c).
  - **The one caveat:** this depends on web literally sending `"REVIEW"` (or any non-`PRODUCT`/`USER` value) in `reportType`. I can't read web. If web actually sends `"PRODUCT"` with the review id in `reportedProductId`, then the issue's 422/misfile is real. **Seam 2 must be closed by web before this is settled.**
  - **Suggested `issues.md` correction draft (for Docs/QA, not edited by me):** in the 2026-05-28 "Review-reporting wire" entry, replace the "Runtime behaviour today" two-outcomes paragraph with: "On the stated premise (web sends `reportType=REVIEW`), the backend `ReportType` enum has no `REVIEW` constant; with no lenient-enum Jackson config and no `HttpMessageNotReadableException` handler, the request fails deserialization and returns HTTP 500 `INTERNAL_ERROR` — the PRODUCT existence check is never reached, so there is neither a misfile nor a `REPORTED_PRODUCT_NOT_FOUND` 422. The exact wire value of `reportType` from web is unconfirmed (Seam 2); if web in fact sends `PRODUCT`, the prior misfile/422 description would apply instead." — pending Seam 2 confirmation.

- **Adjacent observations (Part 4b):**
  1. **No `ReportErrorCodeTest` exists** (`src/test/.../exception/` has only `GlobalExceptionHandlerTest` + `ProductErrorCodeTest`). `ReportErrorCode`'s 10 constants have no seed-coverage enforcement; a constant could be added without an `ERRORS` seed row and CI wouldn't catch it. Severity: **medium** (could mislead — a future engineer assumes parity with `ProductErrorCode`). Not fixed — out of audit scope. Recommend authoring it alongside any `SystemErrorCode` work.
  2. **`resolveTranslationKey` linear-chain trend** (`GlobalExceptionHandler.java:178–193`) — fine for the third enum, but each enum adds a `try/valueOf/catch` block; at 4–5 enums a single registry/map would read better. Severity: **low** (cosmetic/structural). Not fixed — out of scope.

- **Scope question for the feature (needs your call, not assumed):** the cross-cutting set splits into 7 `product.system.*` codes (system/request-level, the clean `SystemErrorCode` candidates) and 4 user/profile codes (`product.user.setup_incomplete`, `displayName.*`). The audit treats only the 7 as the rename/migration set. Do you want the 4 user/profile codes folded into `SystemErrorCode`, given their own `UserErrorCode`, or left in `ProductErrorCode`? The audit flags this rather than deciding.

- **Rename is backend-cheap but possibly cross-repo (Seam 1):** 38 backend edits, self-validating test. Whether it's backend-only depends on whether `oglasino-web` keys UI translation off `translationKey` and hardcodes any `product.system.*` literal. Needs web confirmation before scoping the rename as backend-only.
