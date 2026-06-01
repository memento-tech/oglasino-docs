# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-29
**Task:** READ-ONLY audit — mobile service layer, error contract, offline detection (Parts A–E), mapped against the post-`expo-boot-redesign` code and the new four-enum error-code / review-report contracts.

## Implemented

Read-only audit only — no code changed. Deliverable: `.agent/audit-expo-service-error-contract.md`, covering Parts A–E plus a stale-audit-corrections section and adjacent observations. Key findings:

- **HTTP layer (A):** axios + interceptors confirmed at `src/lib/config/api.ts`. Φ1 401 (single-flight refresh/retry) and 403 (USER_BANNED/EMAIL_BANNED) handling present. The interceptor does **not** parse `{errors:[{field,code,translationKey}]}`; the only helper that touches the shape is `isErrorWithCode` (`src/lib/utils/isErrorWithCode.ts`), which reads `errors[0].code` only — so the error-code split (`code` unchanged) does not affect it.
- **Services (B):** path `src/lib/services/` confirmed, 16 services + 1 test = 17 entries. Dominant pattern (~14): catch → `logServiceError` → sentinel default, discarding the body. Only outlier reading the body is `imageTokensService.ts` (reads `code` off the singular `data.error.code`, rethrows typed). No service reads `field` or `translationKey`.
- **reportService (C):** Finding 24 boolean-inversion bug **confirmed present** (`reportService.ts:16,24`; `error` true on success, but dead — no consumer reads it). Review-reporting is **~35% wired**: `ReportType.REVIEW` exists and two dashboard cards mount it, but the review id is sent in `reportedProductId` (no `reportedReviewId` field exists anywhere), `reportedUserId` is omitted, and public reviews (`ProductReview.tsx`) have no report surface. Against the new backend contract, REVIEW reports will fail with `REPORTED_REVIEW_ID_REQUIRED`.
- **Trust boundaries (E):** mobile sends all report ids as-is, makes no local trust decision — expected posture, nothing to remove.
- **Offline vs maintenance (D):** maintenance now decided at `bootStore.ts:137-148` Gate 1 (`GET /public/maintenance/active`, 5 s timeout, unreachable → maintenance). `netinfo`/any network-state lib is **absent** (package.json + source confirmed). Cold start in airplane mode lands on the **maintenance** screen (`BaseSiteSelector isMaintenance`, via `app/_layout.tsx:116`). Insertion point for an offline check: `bootStore.ts:120`, before `runMaintenanceGate()` (new "Gate 0" → new `'offline'` BootStatus → new overlay branch).

## Files touched

- `.agent/audit-expo-service-error-contract.md` (new, deliverable)
- `.agent/2026-05-29-oglasino-expo-service-error-contract-1.md` (new, this summary)
- `.agent/last-session.md` (overwritten with a copy of this summary)
- No source files changed.

## Method note

Audit fan-out run as a 5-agent read-only Workflow (Parts A, B, C+E, D, and a stale-audit extraction), then synthesized and independently verified against the source: `reportService.ts`, `ReportRequest.ts`, `ReportType.ts`, `ReportButton.tsx`, `ReportDialog.tsx`, the two dashboard review cards, `bootStore.ts`, `maintenanceService.tsx`, and `app/_layout.tsx` were all read first-hand to confirm the agents' line citations before they went into the deliverable.

## Tests

- None run. Read-only audit; no code touched, so the Part-4 lint/tsc/test gates do not apply to any changed path.

## Cleanup performed

- None needed (no code changed).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change required by this session. (The audit is Phase 2 of Φ4; no feature reached `mobile-stable`, so no Expo-backlog row moves. See closure-gate note in "For Mastermind.")
- issues.md: no change authored by this session. Three adjacent observations are surfaced in the audit + below for Mastermind to triage into `issues.md` if desired — I do not write `issues.md`.

## Obsoleted by this session

- Nothing. The audit *corrects* stale claims in the archived `audit-expo-structural.md` (2026-05-24) — see the deliverable's "Stale-audit corrections" — but that file is an immutable archived session record in `oglasino-docs/sessions/`, not something this repo deletes.

## Conventions check

- Part 4 (cleanliness): confirmed — no code added, nothing to clean.
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): three flagged below and in the deliverable.
- Part 6 (translations): N/A this session (no keys touched). Noted for downstream: a future REVIEW-report fix will need user-facing text for `REPORTED_REVIEW_ID_REQUIRED` / `REPORTED_REVIEW_NOT_FOUND` (ERRORS namespace) — not added here.
- Part 7 (error contract): confirmed as the audit's spine — mobile keys on `code` (unaffected by the enum split) but reads no `field`/`translationKey`; most services discard the body.
- Part 11 (trust boundaries): confirmed — Part E checked explicitly; mobile makes no local trust decision off report ids.

## Known gaps / TODOs

- None added to code. The audit itself is the deliverable; no TODO/FIXME comments introduced.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code introduced.
  - Considered and rejected: nothing — no implementation choices to weigh this session.
  - Simplified or removed: nothing — no code changed.

- **Scope decision you asked the audit to settle (Part C):** review-reporting on mobile is ~35% wired and **broken end-to-end against the new backend contract**. The dashboard wire exists but the review id goes into `reportedProductId`; there is no `reportedReviewId` field on `ReportRequest`/`ReportButton`/`ReportDialog`; and public reviews (`ProductReview.tsx`) have no report surface at all. This is a field-contract change + a missing surface + error-code mapping — **more than a one-line finish**. Recommend treating it as its own small `oglasino-expo-review-reports` adoption (or folding into a broader report/messaging mobile chat), not a tail-end of Φ4.

- **Adjacent observations (Part 4b) for `issues.md` triage — I did not fix these (read-only + out of scope):**
  1. `reportService.ts:16,24` — `error` flag inverted (true on success). Dead today (no consumer reads it), so no runtime impact, but wrong and will mislead the next consumer. Severity: **low**. (Old Finding 24, still open.)
  2. `ReceivedReviewCard.tsx:22` / `GivenReviewCard.tsx:29` — review id sent in `reportedProductId`, no `reportedReviewId`, `reportedUserId` omitted though `ReportDialog` types it required. Makes REVIEW reports fail backend validation. Severity: **high** (user-facing; review reporting broken). This is the core Φ4/review-reports finding.
  3. `ProductReview.tsx` — public reviews have no report button; only dashboard "my reviews" cards do. Severity: **medium** (parity/feature gap with web).

- **Offline-vs-maintenance (Part D), pinned for the next brief:** `netinfo` is absent; offline currently shows the maintenance screen; the connectivity-check insertion point is `bootStore.ts:120` (a new pre-maintenance Gate 0 → new `'offline'` BootStatus → new overlay branch in `app/_layout.tsx`); the offline screen would mirror `BaseSiteSelector`'s `isMaintenance` branch. This is the natural Φ4 (or a small follow-up) implementation seam — flagging that it needs translation keys for the offline copy (ERRORS namespace) when it lands.

- **Closure gate:** I checked for an implicit config-file dependency. None required by this audit session — it is Phase 2 (audit) of Φ4; no feature reached `mobile-stable`, so no Expo-backlog row moves and no `state.md` edit is owed. The three adjacent observations above are *candidates* for `issues.md` but I do not assert rows; Docs/QA writes `issues.md` on Mastermind's instruction.
