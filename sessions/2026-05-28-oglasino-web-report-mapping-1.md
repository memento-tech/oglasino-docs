# Session summary

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-28
**Task:** READ-ONLY AUDIT. No code changes. Stage nothing. Produce a session summary only. — audit the current `POST /api/secure/report/add` failure-handling surface on the web so a follow-up fix brief can be scoped precisely.

## Implemented

- Audit only. Five structured answers below, plus a recommended fix shape and one adjacent observation. No edits, no staging.

## Audit answers (questions 1–5 from the brief)

### 1. Submit call site(s)

Single web call site:

- `src/lib/service/reactCalls/reportService.ts:11` — `sendReport(reportRequest: ReportRequest): Promise<ReportCreateResponse>`.
  - **Request shape** (`src/lib/types/report/ReportRequest.ts`):
    ```ts
    type ReportRequest = {
      reportedUserId?: number;
      reportedProductId?: number;
      description: string;
      reportOption: ReportOption;
      reportType: ReportType;  // 'PRODUCT' | 'USER' | 'REVIEW'
    };
    ```
  - **HTTP call:** `BACKEND_API.post('/secure/report/add', reportRequest)`.
  - **Return shape:** `ReportCreateResponse = { success: boolean; alreadyReported: boolean }`. No `errors`, no per-field information, no `code`.
  - **Error handling (current):**
    ```ts
    catch (err) {
      const status = (err as AxiosError).response?.status;
      if (status === 406) {
        return { success: false, alreadyReported: true };
      }
      logServiceError('report.sendReport', err);
      return { success: false, alreadyReported: false };
    }
    ```
    No body parsing. The `{errors:[{field, code, translationKey}]}` envelope the backend now returns on 400/422 is read into the AxiosError, but `sendReport` discards it. Every non-406 failure collapses to the same `{success: false, alreadyReported: false}`.

### 2. UI consumers

- **`src/components/popups/dialogs/ReportDialog.tsx`** (id `DialogId.REPORT_DIALOG`) is the only component that calls `sendReport`. It is opened via `useDialogStore.openDialog(DialogId.REPORT_DIALOG, ...)` from four trigger sites:
  - **`src/components/client/buttons/ReportButton.tsx:46`** — generic wrapper, mounted from three places:
    - `src/components/client/ProductFunctions.tsx:68` (product detail action bar; `ReportType.PRODUCT`, passes `reportedProductId`).
    - `src/components/client/UserDetails.tsx:210` (public user page actions; `ReportType.USER`, passes `reportedUserId`).
    - `src/components/owner/reviews/ReceivedReviewCard.tsx:18` (owner reviews list; `ReportType.REVIEW`, passes `reportedProductId` — see "For Mastermind" §1).
  - **`src/messages/components/Messages.tsx:199`** — conversation-header kebab dropdown "Report" item (wired 2026-05-16 in session `oglasino-web-messages-report-wiring-1`; `ReportType.USER`, passes `reportedUserId: peer.id`).
- Both `ReportButton.tsx` and `Messages.tsx` open the same dialog — there is exactly one component (`ReportDialog`) that talks to the backend, and exactly one service function (`sendReport`). All UX surfacing of success/failure happens inside `ReportDialog.tsx`.

**How `ReportDialog` surfaces today (`ReportDialog.tsx:67-124`):**

- Single inline `<p>` at `:161-168` (no toast, no per-field). State is `success: boolean` + `errorMessage: string`.
- Success → green text, label `tDialog('report.success')`.
- Client-Zod (pre-submit) violations → red text:
  - `tValidation('report.missing.option')`
  - `tValidation('report.missing.description')`
  - `tValidation('report.short.description')` (< 50 chars)
  - `tValidation('suspicion')` (reCAPTCHA fail)
- Server failures → red text, only two outcomes:
  - 406 → `tErrors('report.one.per.day')` ("already reported")
  - everything else (was 500 pre-2026-05-28; will be 400/422 post-2026-05-28) → `tErrors('report.send.fail')`

No `toast`, no `Sonner`, no per-field annotation; the only render target is `errorMessage`.

### 3. Current failure handling

- **Old 500 path (pre-2026-05-28).** `sendReport`'s catch saw `status === 500`, fell through the 406 branch, logged via `logServiceError`, and returned `{success: false, alreadyReported: false}`. `ReportDialog.handleOnClick` `:108-112` saw `!result.success && !result.alreadyReported` → set `errorMessage = tErrors('report.send.fail')`. Same generic message for every non-406 failure; no per-code distinction; the 500 stack trace body was thrown away.
- **406 "already reported" path.** Body-less. Handled explicitly in `reportService.ts:24-28` by reading `response.status`. `ReportDialog` checks `result.alreadyReported` and shows `tErrors('report.one.per.day')`. Unchanged by the backend hardening.
- **New 400/422 path (today).** The new `{errors:[{field, code, translationKey}]}` envelope is delivered in `err.response.data`, but `sendReport` does not inspect `response.data` at all. From the UI's perspective, the new typed-code responses surface identically to the old 500 — `errorMessage = tErrors('report.send.fail')`. There is no existing per-code mapping on the report path.
- **Parser availability.** `src/lib/utils/parseProductValidationErrors.ts` exists and parses exactly the Part-7 wire shape, but it is wired only into `src/lib/service/reactCalls/productService.ts` (see §5). The report path does not call it.

### 4. Translation keys

The web does not ship locale JSON files — translations are runtime-loaded from the backend SQL seed via the `useTranslations(NAMESPACE)('key.path')` hook (`src/translations/...`). Whatever the backend has seeded for the active locale is resolvable from the web; whatever is missing is not.

**Confirming the codes against `oglasino-backend`.** Per the brief, exact constant names were read from the backend ground-truth file `oglasino-backend/src/main/java/com/memento/tech/oglasino/exception/ReportErrorCode.java` (read-only; no edit). The 10 constants and their translationKey + httpStatus:

| Code | translationKey | HTTP |
| --- | --- | --- |
| `REPORT_TYPE_REQUIRED` | `report.type.required` | 400 |
| `REPORT_OPTION_REQUIRED` | `report.option.required` | 400 |
| `REPORT_DESCRIPTION_REQUIRED` | `report.description.required` | 400 |
| `REPORT_DESCRIPTION_TOO_LONG` | `report.description.too_long` | 400 |
| `REPORTED_USER_ID_REQUIRED` | `report.reported_user_id.required` | 422 |
| `REPORTED_PRODUCT_ID_REQUIRED` | `report.reported_product_id.required` | 422 |
| `REPORTED_USER_NOT_FOUND` | `report.reported_user.not_found` | 422 |
| `REPORTED_PRODUCT_NOT_FOUND` | `report.reported_product.not_found` | 422 |
| `REPORT_SELF_NOT_ALLOWED` | `report.self_not_allowed` | 422 |
| `REPORTED_USER_PENDING_DELETION` | `report.reported_user.pending_deletion` | 422 |

Note: this is **ten** codes, not eight — the brief's "four `REPORT_*_REQUIRED` / `_TOO_LONG`" understates by two (the two `REPORTED_*_ID_REQUIRED` cross-field codes are extra to the four Jakarta-DTO codes). Not a "brief vs reality" challenge, just a count correction the brief explicitly anticipated by saying "exact names to be confirmed by you against the backend."

**Seeded values on the backend.** Per `oglasino-docs/sessions/2026-05-28-oglasino-backend-report-trust-boundary-fix-1.md` and `state.md` Risk Watch, all 10 keys live in the `ERRORS` namespace (IDs EN 3149-3158, RS 5249-5258, RU 7349-7358, CNR 1049-1058). EN values are final; RS / RU / CNR are placeholders pending native-translator review.

**What the web can already resolve.** Because translations are fetched from the backend at runtime, the web can resolve all 10 keys today via `useTranslations(TranslationNamespaceEnum.ERRORS)(<translationKey>)`. There is **no missing translation file** on the web — the gap is purely the code → key mapping in `sendReport` / `ReportDialog`, not translation data.

**Existing report-related keys on the web (for context, not confused with the new ones):**

- `ERRORS` namespace: `report.one.per.day`, `report.send.fail` (used today).
- `VALIDATION` namespace: `report.missing.option`, `report.missing.description`, `report.short.description` (client-Zod messages — `VALIDATION` is frozen per conventions Part 6, no new keys go there; pre-existing keys stay).
- `DIALOG` namespace: `report.success`, `report.user.title`, `report.user.description`, `report.product.title`, `report.product.description`, `report.review.title`, `report.review.description`, etc. (dialog chrome strings).
- `BUTTONS` namespace: `report.label`, `report.user.tooltip`, `report.product.tooltip`, `report.review.tooltip`, etc.
- `INPUT` namespace: `report.reason`, `report.description.label`.

None of the 10 new `ERRORS`-namespace keys (`report.type.required`, `report.option.required`, `report.description.required`, `report.description.too_long`, `report.reported_user_id.required`, `report.reported_product_id.required`, `report.reported_user.not_found`, `report.reported_product.not_found`, `report.self_not_allowed`, `report.reported_user.pending_deletion`) are referenced from any web component or service today.

### 5. Reusable parsing

- `src/lib/utils/parseProductValidationErrors.ts` is named "Product" but its body is wire-shape generic: input type is `ProductErrorResponse = { errors: ProductFieldError[] }` (`field, code, translationKey`) — exactly the Part-7 envelope. The dedupe ("first error per field wins"), the `SYSTEM_ERROR_KEY = '__system'` fallback for `field: null`, and the dual `{byField, list}` output shape are not product-specific.
- The product-surface specialization lives in `productService.ts:42-74`: `PARSEABLE_ERROR_STATUSES = Set<400, 403, 422, 429, 500>`, `isProductErrorResponse` typeguard, and `parseProductErrorsForStatus` which adds a 429-only synthetic `RATE_LIMITED` entry (`product.system.rate_limited`) for the Cloudflare-edge 429 case. That synth is product-specific (translation key namespace is `product.system.*`); the typeguard and the 400/422 base path are not.
- For the report path:
  - The base parser (`parseProductValidationErrors`) is reusable as-is. No semantic conflict — `field` values from the report endpoint are `description` / `reportType` / `reportOption` / `reportedUserId` / `reportedProductId` / `null`, all compatible with `byField` keying. The "Product" in the function name is misleading but mechanical.
  - The 429-synth in `productService.ts` is not relevant here — the report endpoint does not have a 429 surface today, and even if Cloudflare-edge 429 shows up, the synth translation key (`product.system.rate_limited`) is wrong namespace for the report dialog.
  - A small report-specific mapping helper is **not** warranted. The backend ships `translationKey` directly in the envelope (e.g. `code: REPORT_SELF_NOT_ALLOWED` → `translationKey: report.self_not_allowed`), and all 10 keys live under one namespace (`ERRORS`). The dialog can call `tErrors(translationKey)` directly off the parsed list — no code → key map needed in TypeScript. Surface routing (which codes go to the inline `<p>` vs. which trigger different UX) is the only choice the dialog has to make.
  - **Mild rename worth considering** (flag for Mastermind, not in scope here): `parseProductValidationErrors.ts` → `parseValidationErrorResponse.ts`, plus matching type renames `ProductErrorResponse` → `ValidationErrorResponse` and `ParsedProductValidationErrors` → `ParsedValidationErrors`. Two surfaces now use it (product, report) and the "Product" branding misleads readers. Not blocking — the follow-up fix can ship under the existing name.

## Recommended fix shape (DO NOT implement in this session)

Two files plus an optional name cleanup; no new helper file needed.

**`src/lib/service/reactCalls/reportService.ts`**

- Import `ProductErrorResponse`, `parseProductValidationErrors`, `ParsedProductValidationErrors` from the existing util.
- Widen `ReportCreateResponse` to:
  ```ts
  type ReportCreateResponse =
    | { success: true }
    | { success: false; alreadyReported: true }
    | { success: false; alreadyReported: false; errors?: ParsedProductValidationErrors };
  ```
  (or keep the flat shape and add an optional `errors?` field — caller convenience choice; the discriminated union mirrors `productService.ts`'s `type: 'validation'` pattern.)
- In the catch block, between the 406 short-circuit and the `logServiceError` fall-through:
  - If `status ∈ {400, 422}` and the body is shape-valid (reuse `isProductErrorResponse` from `productService.ts` — promote it to a shared util, or inline the 4-line typeguard locally), parse via `parseProductValidationErrors(err.response.data)` and return `{success: false, alreadyReported: false, errors: parsed}`.
  - Otherwise keep the existing `logServiceError` + generic `{success: false, alreadyReported: false}` return — covers network failures, 5xx, and shape-invalid bodies.

**`src/components/popups/dialogs/ReportDialog.tsx`**

- Read `result.errors?.list` (or `byField`) in the failure branch at `:108-114`.
- For each surface, pick from the parsed list and render via `tErrors(translationKey)` (the backend ships the key; no client-side map). The dialog has one render target (`errorMessage`), so first-wins (which `parseProductValidationErrors` already does) is sufficient — no UI restructure needed.

**Code → UI surface mapping (all 10 codes render in the existing inline red `<p>` via `tErrors(translationKey)`):**

| Code | UI surface (inline `<p>`, copy = `tErrors(translationKey)`) | Reachability today |
| --- | --- | --- |
| `REPORT_TYPE_REQUIRED` | inline error | unreachable from current callers (every trigger site passes a `reportType` literal) — defense in depth |
| `REPORT_OPTION_REQUIRED` | inline error | unreachable — `handleOnClick` returns early on missing `option` via `tValidation('report.missing.option')` |
| `REPORT_DESCRIPTION_REQUIRED` | inline error | unreachable — client-Zod blocks empty description |
| `REPORT_DESCRIPTION_TOO_LONG` | inline error | reachable if client maxlength (1000) diverges from server limit (2000) — currently client is stricter, so unreachable, but harmless to map |
| `REPORTED_USER_ID_REQUIRED` | inline error | unreachable for USER trigger sites; could surface if a caller forgets the id |
| `REPORTED_PRODUCT_ID_REQUIRED` | inline error | unreachable for PRODUCT/REVIEW trigger sites; same defense-in-depth |
| `REPORTED_USER_NOT_FOUND` | inline error | reachable: target user hard-deleted between page load and submit |
| `REPORTED_PRODUCT_NOT_FOUND` | inline error | reachable: target product hard-deleted between page load and submit |
| `REPORT_SELF_NOT_ALLOWED` | inline error | reachable: user navigates to own profile/product (today the buttons are gated by `iamActive`, so this is defense-in-depth — but tracks the backend rule) |
| `REPORTED_USER_PENDING_DELETION` | inline error | reachable: target user initiated self-deletion during the 7-day grace period |

No code needs a *different* surface (toast vs. inline, redirect, etc.). Single inline target, first-wins, copy from `translationKey` — that's the entire mapping.

**Optional helper-promotion** (single line, no logic change): move `isProductErrorResponse` from `productService.ts:44` and `PARSEABLE_ERROR_STATUSES` from `productService.ts:42` into a shared util (e.g. alongside `parseProductValidationErrors.ts`) so the report path doesn't have to either duplicate the typeguard or cross-import from product. Not strictly required — duplicating the 4-line typeguard inline in `reportService.ts` is also defensible per conventions Part 4a.

## Files touched

- None (read-only audit). Files read: `src/lib/service/reactCalls/reportService.ts`, `src/lib/types/report/ReportRequest.ts`, `src/lib/types/report/ReportType.ts`, `src/components/popups/dialogs/ReportDialog.tsx`, `src/components/client/buttons/ReportButton.tsx`, `src/messages/components/Messages.tsx`, `src/components/client/ProductFunctions.tsx`, `src/components/client/UserDetails.tsx`, `src/components/owner/reviews/ReceivedReviewCard.tsx`, `src/lib/utils/parseProductValidationErrors.ts`, `src/lib/types/product/ProductErrorResponse.ts`, `src/translations/types/TranslationNamespaceEnum.ts`, `src/lib/service/reactCalls/productService.ts` (excerpt for reuse-pattern check). Cross-repo read (ground-truth confirmation per the brief): `oglasino-backend/src/main/java/com/memento/tech/oglasino/exception/ReportErrorCode.java`.

## Tests

- N/A — read-only audit, no code changes. No tests run.

## Cleanup performed

- None needed (audit only, no diff).

## Config-file impact

- `conventions.md`: no change.
- `decisions.md`: no change.
- `state.md`: no change. (The Risk Watch row for native-translator review of `report.*` placeholders is already authored by the backend session and applied by Docs/QA; this audit does not need to amend it.)
- `issues.md`: no change. (The 2026-05-16 "Report-submit endpoint trust-boundary verification unknown" entry was already flipped to `fixed` by Docs/QA in the backend close-out; the follow-up "web mapping pending" is the *subject* of this audit and the next brief, not a separate `issues.md` entry to file.)

## Obsoleted by this session

- Nothing — audit only.

## Conventions check

- Part 4 (cleanliness): confirmed. No code change, nothing to clean.
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): one flagged in "For Mastermind."
- Part 6 (translations): confirmed in §4 — namespaces correct (all 10 new keys are `ERRORS`, not `VALIDATION` which is frozen), key pattern matches Rule 4 with the `report.*` surface prefix (precedent: `product.<field>.<code>`), no parent/child collision (every new key has a leaf segment).
- Part 7 (error contract): confirmed in §3 — the new envelope is the unified `{errors:[{field, code, translationKey}]}` shape; the parser handles it; first-error-per-field-wins applies; the dialog's single-error inline `<p>` matches the contract's "first error per field wins on the client" semantics.
- Part 11 (trust boundaries): N/A for the web side (backend is the trust boundary; this audit's recommended fix does not touch trust decisions, only display).

## Known gaps / TODOs

- None deliberately deferred.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - **Added (earned complexity):** nothing — audit only.
  - **Considered and rejected:** (a) a new `parseReportValidationErrors.ts` helper. Rejected — existing `parseProductValidationErrors` is wire-shape generic, and all 10 codes route to one inline target with copy = `tErrors(translationKey)`. A second helper would add a file for zero behavioural difference. (b) A client-side code → key mapping table in `reportService.ts`. Rejected — backend ships `translationKey` in the envelope; the table would duplicate the backend contract and rot the first time a key is renamed.
  - **Simplified or removed:** nothing — audit only.

- **Part 4b adjacent observations (one):**
  - **`ReportType.REVIEW` is wired on the web but is not part of the backend's 2026-05-28 hardening narrative.** `src/lib/types/report/ReportType.ts` declares `PRODUCT | USER | REVIEW`. `ReceivedReviewCard.tsx:18` mounts a `ReportButton` with `reportType: ReportType.REVIEW` and `reportedProductId: review.id` (note: passes the review id into the `reportedProductId` slot — possibly the original intent, possibly drift). The decisions.md 2026-05-28 entry and the backend session summary only treat `USER` and `PRODUCT` paths. Backend `ReportErrorCode` only emits `REPORTED_USER_ID_REQUIRED` / `REPORTED_PRODUCT_ID_REQUIRED` (no `REPORTED_REVIEW_ID_REQUIRED`), and the backend service's self-report check only handles USER and PRODUCT branches. **File:** `src/components/owner/reviews/ReceivedReviewCard.tsx:18-25`. **Severity:** medium — submitting a review report today may either succeed (silently treated as a PRODUCT report against the review id), or hit a new typed-code path the backend doesn't have a self-report block for, or land in an `else` branch of the new typed-code service. I did not investigate the backend's REVIEW behaviour because that is cross-repo. Did not fix because out of scope; flagging for Mastermind to triage (probable outcomes: REVIEW path was always a partial wire and the fix brief should either remove the wire or scope a backend ticket).

- **Suggested next-step brief shape (for Mastermind to author or revise):**
  - Title: "Map new `ReportErrorCode` family to UI copy in `ReportDialog` (web)."
  - Scope: extend `sendReport` to parse `{errors:[...]}` on 400/422 and return parsed list; consume it in `ReportDialog.handleOnClick` failure branch; render via `tErrors(translationKey)` in the existing inline `<p>`. No new files. Existing 406 + reCAPTCHA + client-Zod paths untouched.
  - Out of scope (worth flagging to keep the brief tight): the `ReportType.REVIEW` triage above; the optional `parseProductValidationErrors` → `parseValidationErrorResponse` rename; the optional `isProductErrorResponse` helper promotion (inline duplication is acceptable per Part 4a).
  - Tests: unit test the new catch branch in `reportService.ts` (mock AxiosError with 400 + valid envelope, 422 + valid envelope, 422 + malformed body, 406 unchanged, 5xx unchanged). No new component tests needed unless Mastermind wants render-level coverage of the failure copy.

- **Closure gate.** No pending config-file edits — no drafts produced. Backend's close-out already covers `decisions.md`, `state.md` Risk Watch, and `issues.md` flips. This audit produces no new config-file dependency.
