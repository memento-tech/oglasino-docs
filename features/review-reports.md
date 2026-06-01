# Review Reports

**Slug:** `review-reports`
**Status:** web-stable
**Repos:** oglasino-backend (primary), oglasino-web (wire + mount)
**Branches:** backend `dev`, web `stage`

## 1. Goal

Reporting a review is wired on web but absent on backend. The web "Report" button on a received review sends `reportType=REVIEW` with the review id in the `reportedProductId` slot; the backend `ReportType` enum has no `REVIEW` constant, so the request fails Jackson deserialization and returns HTTP 500. Build the missing backend half and align the web wire so a user can report a review and the server validates the target like any other report.

## 2. Current behaviour (corrected from issues.md 2026-05-28)

The issue claims a REVIEW report runs `productRepository.getOwnerId(reviewId)` → misfile-on-collision or `REPORTED_PRODUCT_NOT_FOUND` 422. That is not what happens. `ReportType` = `{PRODUCT, USER}`. Web sends the literal string `"REVIEW"`. With no lenient-enum Jackson config and no `HttpMessageNotReadableException` handler, the body fails deserialization → catch-all → HTTP 500 `INTERNAL_ERROR`. The PRODUCT branch never runs; `getOwnerId` is never called; there is no 422 and no misfile. (An issues.md correction is drafted at chat close.)

## 3. End state

A REVIEW report is a first-class report type alongside USER and PRODUCT:

- `ReportType` gains `REVIEW`.
- The request DTO gains `reportedReviewId`.
- `DefaultReportService` gains a REVIEW branch: validate the review exists, block self-reports, persist.
- Per-target dedupe extends to REVIEW for free (the cache key already embeds the type).
- Web sends the review id in a dedicated `reportedReviewId` field, not the product slot.

## 4. Backend — request DTO

`ReportRequestDTO` gains:

- `Long reportedReviewId` — no Jakarta annotation (presence enforced in the service for the REVIEW branch, matching how `reportedUserId` / `reportedProductId` are handled).

`reportType` (`@NotNull`), `reportOption` (`@NotNull`), `description` (`@NotBlank @Size(max=2000)`) unchanged.

## 5. Backend — `ReportType` + schema

- `entity/ReportType.java` gains `REVIEW` (enum now `{PRODUCT, USER, REVIEW}`).
- The `report_type` CHECK constraint at `V1__init_schema.sql:477` widens from `('PRODUCT','USER')` to `('PRODUCT','USER','REVIEW')`. Pre-prod V1 fold — edit V1 in place, no new migration (conventions Part 12).
- `Report` entity (`entity/Report.java`) gains a `reportedReviewId` column to persist the REVIEW target. Mirror the `reportedProductId` column's shape (plain `Long`, no FK — same as the existing product slot per the 2026-05-28 fix's posture). V1 fold adds the column.

## 6. Backend — service branch (the trust boundary)

`DefaultReportService.createReport` gains a REVIEW branch alongside USER and PRODUCT. `reportedReviewId` is client-supplied and is validated server-side — this is a Part 11 trust boundary. The branch:

1. **Existence check.** `reviewRepository.findById(reportedReviewId).orElseThrow(REPORTED_REVIEW_NOT_FOUND)`. A non-existent review is a coded 422, never an uncaught exception.
2. **Required check.** If `reportedReviewId` is null on a REVIEW report → `REPORTED_REVIEW_ID_REQUIRED` (422).
3. **Self-report block.** Reject if the review's `reviewer` is non-null AND `reporterId.equals(review.getReviewer().getId())` → `REPORT_SELF_NOT_ALLOWED` (reuse existing constant). The null-check is required because `Review.reviewer` is nulled on author anonymisation (`ReviewRepository.updateReviewerIdToNullForApproved`); a null author can never be "self," so an anonymised review remains reportable by its `targetUser` — the primary legitimate use. You cannot report your own review. The review's `targetUser` (the seller the review is about) is explicitly allowed to report.
4. **No deletion-state guard for REVIEW.** Unlike the USER branch (`REPORTED_USER_PENDING_DELETION`), a review is its own object; whether its author or subject is pending deletion does not change that the review content is reportable. Decision locked Phase 4.
5. Persist with `reportedReviewId` set, `reportedUser`/`reportedProductId` null.

The reporter identity is server-derived (`currentUserService.getCurrentUserIdStrict()` from `SecurityContextHolder`) — unchanged from the existing flow. Only the target id is client-supplied, and it is now validated.

## 7. Backend — dedupe key

`DefaultReportFacade.buildCacheKey` (lines 47-52) currently extracts `targetId` via a binary ternary (`USER ? reportedUserId : reportedProductId`). Extend to a 3-way resolution so REVIEW uses `reportedReviewId`. The key format `report:user:<reporterId>:target:<type>:<targetId>` already embeds the type, so `report:user:<id>:target:REVIEW:<reviewId>` comes free — only the `targetId` extraction changes. 24h TTL unchanged. A reporter can report distinct reviews within 24h; the same review twice → 406.

## 8. Backend — new error codes + translations

Two new `ReportErrorCode` constants:

| Constant | translationKey | httpStatus |
|----------|----------------|------------|
| REPORTED_REVIEW_ID_REQUIRED | report.reported_review_id.required | UNPROCESSABLE_ENTITY |
| REPORTED_REVIEW_NOT_FOUND | report.reported_review.not_found | UNPROCESSABLE_ENTITY |

Self-report reuses the existing `REPORT_SELF_NOT_ALLOWED` (`report.self_not_allowed`).

Seed rows — 2 keys × 4 locales = 8 rows, appended to the `ERRORS` namespace group in the four `0001-data-web-translations-<LOCALE>.sql` files, next available IDs per locale (engineer reads the current max, reports collision rather than overwriting — conventions Part 6 Rule 3).

Translation drafts (EN final; RS/RU/CNR are Mastermind drafts pending native-translator review — Risk Watch at close):

`report.reported_review_id.required`
- EN: `Please select a review to report.`
- RS: `Izaberite recenziju koju prijavljujete.`
- RU: `Pozhaluysta, vyberite otzyv dlya zhaloby.`
- CNR: `Izaberite recenziju koju prijavljujete.`

`report.reported_review.not_found`
- EN: `This review no longer exists.`
- RS: `Ova recenzija više ne postoji.`
- RU: `Etot otzyv bol''she ne sushchestvuyet.`
- CNR: `Ova recenzija više ne postoji.`

(RU uses Latin transliteration with `''` for the soft sign per the established convention — decisions.md 2026-05-27. RS/CNR identical here, consistent with the cnr→sr aliasing in conventions Part 9; the translator confirms or splits them.)

## 9. Backend — tests

- `DefaultReportServiceTest` — REVIEW branch: existence-check miss → `REPORTED_REVIEW_NOT_FOUND`; null id → `REPORTED_REVIEW_ID_REQUIRED`; self-report (reporter == reviewer) → `REPORT_SELF_NOT_ALLOWED`; target-user reporting → success; happy path persists with `reportedReviewId` set.
- `DefaultReportFacadeTest` — per-target dedupe for REVIEW: same reporter + two different reviews both succeed within 24h; same reporter + same review twice → 406.
- `ReportErrorCodeTest` (authored in the `system-error-code-split` spec) covers the two new constants' seed presence automatically once it exists. If the split spec ships after this one, this spec authors the two new rows and the test catches them when it lands. Sequencing in §12.

## 10. Web changes (on `stage`)

- `src/lib/types/report/ReportRequest.ts` — add `reportedReviewId?: number`.
- `src/components/owner/reviews/ReceivedReviewCard.tsx:24` — change `reportedProductId={review.id}` → `reportedReviewId={review.id}`.
- `ReportButton` / `ReportDialog` — thread `reportedReviewId` through the same way `reportedUserId` / `reportedProductId` are threaded (props → dialog → `sendReport` body). No per-type routing logic; the mounting component sets the slot, the dialog forwards it.
- Adjacent (fold in): `reportService.ts` maps only HTTP 406 today; the typed `REPORT_*` 4xx codes (including the two new REVIEW codes) collapse to a generic `report.send.fail` toast. This is the pre-existing W6 follow-up (issues.md 2026-05-28). In scope here for the REVIEW codes at minimum — surface `REPORTED_REVIEW_NOT_FOUND` and `REPORTED_REVIEW_ID_REQUIRED` as their translated messages rather than the generic toast. The engineer decides whether to do the full typed-code mapping or just the two REVIEW codes; the brief states the minimum.

## 11. Brief order

1. Backend brief: `ReportType.REVIEW`, DTO field, V1 CHECK + column, service REVIEW branch, dedupe extension, two `ReportErrorCode` constants + 8 seed rows, service + facade tests. One brief — schema, enum, DTO, service, dedupe, codes, seeds, tests interlock.
2. Web brief: `ReportRequest` field, `ReceivedReviewCard` mount swap, prop threading, REVIEW-code toast mapping. Separate (different repo).

Backend lands first. Web must not ship the `ReceivedReviewCard` mount swap before backend, or `reportType=REVIEW` still 500s on the old backend. Sequence: backend brief → merge → web brief.

## 12. Dependency on `system-error-code-split`

These two specs share `ReportErrorCode` and `GlobalExceptionHandler`. If `system-error-code-split` ships first (recommended — it's the smaller, lower-risk one), `ReportErrorCodeTest` already exists and the two new REVIEW constants slot into the enforced enum cleanly. If this spec ships first, the two new constants land in an unenforced enum and the split spec's `ReportErrorCodeTest` picks them up when it arrives. Either order works; recommend split-first.

## 13. Definition of done

- Backend: `ReportType.REVIEW` exists; `reportedReviewId` on DTO + `Report` entity + V1; REVIEW branch validates existence + self-report server-side; dedupe extended; two new codes + 8 seed rows; tests green. `./mvnw spotless:check` + `./mvnw test` green.
- Web: `reportedReviewId` threaded; `ReceivedReviewCard` swapped; REVIEW codes surfaced as translated messages. `npm run lint`, `npx tsc --noEmit`, `npm test` green.
- A logged-in user reports a review; backend validates the review exists and the reporter is not the review's author; report persists; same review reported twice within 24h → 406.

## 14. Trust boundaries (Part 11)

`reportedReviewId` is client-supplied. The server: (a) validates it resolves to a real review (`reviewRepository.findById`), (b) blocks the reviewer from reporting their own review, with the reviewer-id read null-guarded — `Review.reviewer` is nulled on author anonymisation, so a null (anonymised) reviewer is treated as "not self" and the review stays reportable by its `targetUser`, (c) does not trust any client-supplied "is this my review" claim — when the reviewer is non-null it reads `review.getReviewer().getId()` from the persisted row and compares to the authenticated principal. Same posture as the 2026-05-28 USER/PRODUCT trust-boundary fix.

## Factual vs inferred

- **Factual** (from audits): current 500 behaviour and its cause; `ReportType` = `{PRODUCT, USER}`; no `reportedReviewId`; `Review` carries `reviewer` + `targetUser`; the dedupe key embeds type; the CHECK constraint at V1:477; web sends `review.id` in `reportedProductId`; REVIEW surfaced only in `ReceivedReviewCard`; `reportService.ts` maps only 406.
- **Inferred** (Mastermind framing / Phase 4 decisions, confirmed by Igor): build-not-remove; self-report blocks reviewer only, allows targetUser; no deletion-state guard for REVIEW; two new error codes + reuse `REPORT_SELF_NOT_ALLOWED`; `reportedReviewId` as a dedicated field vs repurposing `reportedProductId`. The RS/RU/CNR translation drafts are Mastermind placeholders, not final.

## Session log

- **2026-05-29** — Code-complete across backend (`dev`) and web (`stage`). Backend (`2026-05-29-oglasino-backend-review-reports-1`): `ReportType.REVIEW`, `reportedReviewId` on DTO + `Report` entity + V1 CHECK/column, REVIEW service branch (existence + null-safe reviewer-only self-report), 3-way dedupe, two new `ReportErrorCode` constants + 8 seed rows; 661 tests passing. Web (`2026-05-29-oglasino-web-review-reports-1`): `reportedReviewId` threaded, `ReceivedReviewCard` mount swapped, two REVIEW codes surfaced as translated messages; 229 tests passing. §6 step 3 + §14 amended this session to match the shipped null-safe self-report (engineer caught the NPE the literal spec expression would have produced on an anonymised review). Status flipped `planned` → `web-stable`. See [decisions.md](../decisions.md) 2026-05-29 and [issues.md](../issues.md) 2026-05-28 REVIEW entry (now `fixed`).
