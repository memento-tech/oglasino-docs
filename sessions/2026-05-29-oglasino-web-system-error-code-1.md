# Session summary

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-29
**Task:** Web audit: product/system error-code consumption + REVIEW-report wire (READ-ONLY)

## Implemented

- Read-only audit only — no code changed. Produced `.agent/audit-system-error-code.md`.
- **Part A (error codes):** Established that web keys product-validation UI off the wire `translationKey` (parser writes `byField[field] = err.translationKey`; UI calls `tErrors(translationKey)`), with `code` read only for analytics. Documented the 429 synth `parseProductErrorsForStatus` and its only hardcoded literals (`code: 'RATE_LIMITED'`, `translationKey: 'product.system.rate_limited'`) plus its six call sites. Swept the repo for `product.system.*` literals (one: `productService.ts:59`), hardcoded backend codes (control-flow branches in api/auth/delete + the image-pipeline switch), and code→key mappings (none for product; the lone exception is image-upload `errorMapping.ts`). Namespace is `ERRORS`.
- **Part B (REVIEW wire):** Confirmed `ReceivedReviewCard.tsx:24` sends `reportedProductId={review.id}` with `reportType: REVIEW`; `ReportRequest` has no `reportedReviewId`; REVIEW is surfaced only in `ReceivedReviewCard`. Documented full USER/PRODUCT/REVIEW request shapes and the submit service (`POST /secure/report/add`, only 406 specially handled).
- Surfaced three seams (translationKey-vs-code rename impact; the REVIEW field contract a/b/c decision; the image-pipeline code→key exception) and one Part 4b adjacent observation (report failure codes not mapped).

## Files touched

- `.agent/audit-system-error-code.md` (new, audit deliverable)
- `.agent/2026-05-29-oglasino-web-system-error-code-1.md` + `.agent/last-session.md` (session summary)
- No source files touched.

## Tests

- Not run — read-only audit, no code changed. (`npm run lint` / `tsc` / `npm test` N/A: zero source edits.)

## Cleanup performed

- none needed (no code changed).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change. (This is a Phase 2 audit; Mastermind decides any state flip after seam analysis.)
- issues.md: no change required from web. The audit's REVIEW-wire finding matches the existing `issues.md` 2026-05-28 entry ("Review-reporting wire is an unfinished feature," `parked`); the adjacent observation (report failure codes unmapped) is the existing W6 follow-up already noted in `decisions.md`/`issues.md` 2026-05-28. No new entry is owed.

## Obsoleted by this session

- nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code changed; no debug logging/TODOs introduced.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one flagged in "For Mastermind".
- Part 6 (translations): N/A this session (no keys added/changed). Audit documents that `ERRORS`-namespace `product.*` keys are backend-seeded; web references only.
- Part 7 (error contract): confirmed as the audit's subject — web consumes `{field, code, translationKey}`, renders off `translationKey`.

## Known gaps / TODOs

- The audit does not inspect the backend; the REVIEW field-contract resolution (a/b/c) is a backend/product decision. Web execution of whichever option is chosen is a future brief.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit, no code).
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- **Headline for seam analysis:** web translates product errors off `translationKey`, not `code`. So a backend enum-home move / `code` rename does **not** touch web rendering. The only `translationKey` web hardcodes is `product.system.rate_limited` (`productService.ts:59`) — if that key is renamed, web must change in lockstep; all other `translationKey` renames are web-safe because web passes the key through verbatim.
- **REVIEW wire (Seam 2):** web sends the review id in `reportedProductId`; no `reportedReviewId` exists. Resolution is a product call between (a) build backend REVIEW support + add `reportedReviewId` on web, (b) remove the web mount, (c) repurpose to a PRODUCT report against `review.reviewedProduct.productId`. Web can execute any of the three. Aligns with `issues.md` 2026-05-28 (parked).
- **Seam 3 (out of brief scope, flagged):** `src/lib/images/errorMapping.ts` is the one place web maps backend `code → key`. An image-pipeline code rename would silently fall through to a default key there — opposite rename-impact profile from product errors. Worth noting if any cross-cutting "error-code enum home" move touches image codes too.
- **Part 4b adjacent observation:** `src/lib/service/reactCalls/reportService.ts` maps only HTTP 406 → "already reported"; the backend's newer typed `REPORT_*` 4xx codes (2026-05-28 trust-boundary fix) collapse to a generic `tErrors('report.send.fail')` toast (`ReportDialog.tsx:111`). Severity **low** (UX). Not fixed — out of read-only scope. This is the W6 follow-up already on record.
