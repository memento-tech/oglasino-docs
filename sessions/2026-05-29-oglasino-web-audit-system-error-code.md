# Audit — Web: product/system error-code consumption + REVIEW-report wire

**Repo:** `oglasino-web`
**Branch:** `stage`
**Date:** 2026-05-29
**Mode:** READ-ONLY. No code changed.
**Brief:** `.agent/brief.md` — "Web audit: product/system error-code consumption + REVIEW-report wire"

All findings below were read from code directly (not from prior reports or issue text), per the brief's instruction.

---

## TL;DR — the two answers Mastermind needs

1. **Web keys product-validation UI off the wire `translationKey`, not off `code`.** The shared parser writes `byField[field] = err.translationKey` and the UI calls `tErrors(translationKey)` verbatim. `code` is read **only** for analytics (`track('form_submit_failed', { error_code })`). **Consequence:** a backend enum-home move or a `code`-string rename does **not** touch web's product-error rendering. A **`translationKey`** rename **does** — but web hardcodes only one such literal: `'product.system.rate_limited'` in the 429 synth. (One out-of-scope exception: the image-upload subsystem *does* map `code → key` explicitly — see §A.3(c)/Seam 3.)

2. **The REVIEW report wire sends the review id in the `reportedProductId` slot.** `ReceivedReviewCard.tsx:24` passes `reportedProductId={review.id}` with `reportType={ReportType.REVIEW}`. There is **no** `reportedReviewId` field on the web `ReportRequest` type. REVIEW reporting is surfaced in **exactly one** place (`ReceivedReviewCard`).

---

# Part A — how web consumes backend error codes

## A.1 — Envelope parsing: `parseProductValidationErrors` (THE most important finding)

**Parser:** `src/lib/utils/parseProductValidationErrors.ts`

Wire type it consumes (`src/lib/types/product/ProductErrorResponse.ts:1-9`):

```ts
export interface ProductFieldError {
  field: string | null;
  code: string;
  translationKey: string;
}
export interface ProductErrorResponse {
  errors: ProductFieldError[];
}
```

Parser body (`parseProductValidationErrors.ts:31-45`):

```ts
export function parseProductValidationErrors(
  response: ProductErrorResponse
): ParsedProductValidationErrors {
  const byField: Partial<Record<string, string>> = {};
  const list: ParsedProductError[] = [];
  const seen = new Set<string>();
  for (const err of response.errors) {
    const key = err.field ?? SYSTEM_ERROR_KEY;
    if (seen.has(key)) continue;
    seen.add(key);
    byField[key] = err.translationKey;           // <-- display value is translationKey
    list.push({ field: err.field, code: err.code, translationKey: err.translationKey });
  }
  return { byField, list };
}
```

Returned shape (`parseProductValidationErrors.ts:12-21`):

```ts
export type ParsedProductError = {
  field: string | null;
  code: string;
  translationKey: string;
};
export type ParsedProductValidationErrors = {
  byField: Partial<Record<string, string>>;   // field -> translationKey
  list: ParsedProductError[];                   // field + code + translationKey
};
```

`SYSTEM_ERROR_KEY` (`parseProductValidationErrors.ts:6`):

```ts
export const SYSTEM_ERROR_KEY = '__system';
```

`field: null` errors collapse under `__system`; first error per field wins (the `seen` set).

### Does web translate off `code`, off `translationKey`, or both?

**Off `translationKey`.** The `byField` map (the display view) holds `translationKey` strings; the UI passes them straight into the `ERRORS`-namespace translator. `code` is **never** passed to `t()` — it is read only for analytics. Confirmed at every product-error rendering site:

- `src/components/popups/components/BasicInfoProductDialog.tsx` (create wizard step 1)
  - `tErrors = useTranslations(TranslationNamespaceEnum.ERRORS)` (`:58`)
  - `setProductErrors(result.errors.byField)` (`:120`)
  - Per-field render: `errorMessage={productErrors['name'] ? tErrors(productErrors['name']) : undefined}` (`:205`; same shape `:228` description, `:244` price, `:193` category, `:281` images)
  - System render: `{tErrors(productErrors[SYSTEM_ERROR_KEY]!)}` (`:294`)
  - **`code` consumed for analytics only** (`:121-126`):
    ```ts
    for (const err of result.errors.list) {
      track('form_submit_failed', {
        form_name: 'product_create_step_1',
        error_code: err.code,   // analytics/telemetry only — never t()
        field: err.field,
      });
    }
    ```
- `app/[locale]/owner/products/[productId]/page.tsx` (owner edit page)
  - `tErrors = useTranslations(TranslationNamespaceEnum.ERRORS)` (`:63`)
  - `setProductErrors(result.errors.byField)` (`:239`); per-field `tErrors(productErrors['name'])` etc. (`:369`,`:382`,`:412`,`:491`,`:358`); system `{tErrors(productErrors[SYSTEM_ERROR_KEY]!)}` (`:539`)
  - Inline comment confirming the design (`:127-131`): *"…the backend ships `product.*` on `translationKey`. One translator covers both sources; no discriminator needed."*
- `src/components/popups/components/UploadedProductDialog.tsx` (create wizard final step)
  - `tErrors = useTranslations(TranslationNamespaceEnum.ERRORS)` (`:33`)
  - `setValidationErrors(result.errors.byField)` (`:85`); per-field `<li>{tErrors(key!)}</li>` (`:197`); system `{tErrors(systemError)}` (`:199`)

**The field actually passed to next-intl's `t()` is `translationKey`. `code` is never translated.**

## A.2 — The 429 synth: `parseProductErrorsForStatus`

**Location:** `src/lib/service/reactCalls/productService.ts`

Hardcoded constant (`:59`):

```ts
const RATE_LIMITED_TRANSLATION_KEY = 'product.system.rate_limited';
```

NOTE comment documenting the scope (`:52-58`):

```ts
// NOTE: 429-only synth. The Cloudflare edge router worker (per conventions
// Part 8, the edge boundary) may emit a 429 that does not pass through
// Spring's `RateLimitFilter` and so may lack the unified body shape — no
// `{field: null, translationKey: ...}` entry. Without this guard, the create
// wizard's cooldown gate (`BasicInfoProductDialog`) silently no-ops and the
// user is stuck on step 1. The synth is 429-only; other statuses
// (400/422/403/500) still require a contract-shaped body per Part 7.
```

Function body (`:61-74`):

```ts
const parseProductErrorsForStatus = (
  status: number,
  response: ProductErrorResponse
): ParsedProductValidationErrors => {
  const parsed = parseProductValidationErrors(response);
  if (status !== 429 || parsed.byField[SYSTEM_ERROR_KEY]) return parsed;
  return {
    byField: { ...parsed.byField, [SYSTEM_ERROR_KEY]: RATE_LIMITED_TRANSLATION_KEY },
    list: [
      ...parsed.list,
      { field: null, code: 'RATE_LIMITED', translationKey: RATE_LIMITED_TRANSLATION_KEY },
    ],
  };
};
```

**Exact hardcoded literals it injects (only when status === 429 AND no `__system` entry already present):**
- `code` = `'RATE_LIMITED'` (`:71`)
- `translationKey` = `'product.system.rate_limited'` (via `RATE_LIMITED_TRANSLATION_KEY`, `:59` + `:68`,`:71`)

**Six call sites** (all in `productService.ts`):
1. `:178` — `createNewProduct`, resolved-response branch
2. `:194` — `createNewProduct`, `catch` branch
3. `:235` — `updateProductData`, resolved-response branch
4. `:253` — `updateProductData`, `catch` branch
5. `:279` — `preValidateProductBasics`, resolved-response branch
6. `:296` — `preValidateProductBasics`, `catch` branch

## A.3 — Repo sweep: hardcoded `product.system.*`, hardcoded backend codes, code→key mappings

### (a) Hardcoded `product.system.*` literals (non-test)

| file:line | line |
|---|---|
| `src/lib/service/reactCalls/productService.ts:59` | `const RATE_LIMITED_TRANSLATION_KEY = 'product.system.rate_limited';` |

That is the **only** non-test `product.system.*` literal in the repo. (Tests reference `product.system.rate_limited` and `product.system.internal_error` as fixtures — `parseProductValidationErrors.test.ts:56,60,61,68,69,73,74,82,88`, `productService.test.ts:110,120,122,136,138,161,169,207,226 — but no other production literal exists.)

### (b) Hardcoded backend error-code string literals (non-test)

These are used as **control-flow branches**, not for translation (web never builds a `product.*` key from them):

| file:line | code literal | purpose |
|---|---|---|
| `src/lib/service/reactCalls/productService.ts:71` | `'RATE_LIMITED'` | 429 synth (analytics `code` field) |
| `src/lib/config/api.ts:54` | `errors?.[0]?.code === 'USER_BANNED'` | axios interceptor — global banned-user handling |
| `src/lib/service/reactCalls/authService.ts:157` | `isErrorWithCode(error, 'EMAIL_BANNED')`, `isErrorWithCode(error, 'USER_BANNED')` | auth flow branch |
| `src/components/popups/dialogs/DeleteAccountConfirmationDialog.tsx:131` | `isErrorWithCode(error, 'REAUTH_REQUIRED')` | account-deletion reauth branch |
| `src/lib/images/errorMapping.ts:38-87` | `CHAT_ID_REQUIRED`, `CHAT_ID_NOT_ALLOWED`, `DIMENSIONS_TOO_LARGE`, `CONTENT_TYPE_NOT_ALLOWED`, `FILE_TOO_LARGE`, `RATE_LIMITED`, `IMAGE_TOO_OLD`, …(image-pipeline codes) | **code→key mapping** — see (c) |

Client-side analytics `error_code` literals (these are web-originated Zod failures, **not** backend codes — they are emitted into `track()` only): `RegisterDialog.tsx:76,84,104,124,132` (`DISPLAY_NAME_REQUIRED`, `DISPLAY_NAME_TOO_SHORT`, `EMAIL_REQUIRED`, `PASSWORD_REQUIRED`, `PASSWORD_TOO_SHORT`); `LogInDialog.tsx:62,82,90`; `app/[locale]/owner/user/page.tsx:137,149,159`.

User-state string literals (`'PENDING_DELETION'`, `'BANNED'`, `'ACTIVE'`, `'LOCKED'`) appear across user DTO types and UI guards (`UserInfoDTO.ts:18`, `DeletionStatus.ts:1`, `Chats.tsx:79`, `Messages.tsx:97`, `UserStateIndicators.tsx:10`, etc.) — these are enum **state** values, not error codes; out of this audit's scope but listed for completeness.

### (c) Places that map a backend `code` → a web translation key

**Product/system flow:** **none.** Web does not map product `code → key` anywhere. The translation key arrives ready-made on `translationKey` and is used verbatim (§A.1).

**Image-upload flow (separate subsystem, out of brief scope but a real seam):** `src/lib/images/errorMapping.ts:30+` — `errorCodeToTranslationKey(errorCode: string): ImageErrorTranslationKey` is an explicit N→1 `switch` mapping Worker/backend image codes onto web `image.*` keys (`image.invalid`, `image.forbidden`, `image.bad.format`, `image.too.big`, `image.rate.limited`, `image.server.error`, …). This is the **only** place in the repo where web translates UI off a backend `code` rather than a `translationKey`.

**Client structural-check key building (not backend-derived):** `src/lib/validators/productValidator.ts:52-56` builds `product.${field}.required | .too_long | .too_short` from Zod results for client-side checks. The comment (`:48`) notes these intentionally collide with the backend-seeded `product.<field>.*` keys in the `ERRORS` namespace so one translator covers both. These keys are derived from the **field name + the failing Zod rule**, never from a backend `code`.

## A.4 — Web-side translation key namespace

- **Namespace:** product/system error keys resolve through the **`ERRORS`** next-intl namespace. Every rendering site uses `useTranslations(TranslationNamespaceEnum.ERRORS)` (aliased `tErrors`): `BasicInfoProductDialog.tsx:58`, `owner/products/[productId]/page.tsx:63`, `UploadedProductDialog.tsx:33`, `ReportDialog.tsx:57`.
- **Key pattern:** `product.<field>.<rule|code_lowercase>` (e.g. `product.name.banned_words`, `product.system.rate_limited`). For backend errors the full key text is supplied by the backend on `translationKey`; web does not assemble it. For client Zod errors web assembles `product.<field>.<rule>` itself (`productValidator.ts:52-56`).
- **Is there a web code→key mapping table?** For product/system errors, **no** — web has no lookup table; it consumes `translationKey` directly. The only code→key table in the repo is the image-pipeline `errorCodeToTranslationKey` switch (`errorMapping.ts`), which is a different subsystem.
- **Where do the `ERRORS` strings live?** Not in this repo as static JSON — translations are backend-seeded and fetched at runtime (the repo has no `messages/`/locale JSON tree; `src/messages/` is a components directory, not message catalogs). Per conventions Part 6 + the project model, the `ERRORS`-namespace values (including `product.system.rate_limited`, `product.system.internal_error`, and every `product.<field>.*`) are seeded by the **Backend engineer agent** in the SQL seed. Web only references the keys.

---

# Part B — REVIEW-report wire

## B.5 — `ReceivedReviewCard.tsx` ReportButton mount

**File:** `src/components/owner/reviews/ReceivedReviewCard.tsx`

```tsx
9   export default function ReceivedReviewCard({ review }: { review: ReviewDTO }) {
...
18        <ReportButton
19          tooltip={tButtons('report.review.tooltip')}
20          reportType={ReportType.REVIEW}
21          reportOptions={Object.values(ReviewReportOption)}
22          reportTitle={tDialog('report.review.title')}
23          reportDescription={tDialog('report.review.description')}
24          reportedProductId={review.id}     // <-- REVIEW id into the PRODUCT slot
25          useIcon={false}
26        />
```

- `reportType`: `ReportType.REVIEW`
- id passed: `review.id` — this is the **review's own id** (`ReviewDTO.id`, `src/lib/types/review/ReviewDTO.ts:13-14`), **not** the reviewed product's id (which would be `review.reviewedProduct.productId`, `ReviewDTO.ts:7-8`).
- request field it lands in: **`reportedProductId`**.

## B.6 — Submit service + ReportButton + full request shapes

**Service:** `src/lib/service/reactCalls/reportService.ts`

```ts
export const sendReport = async (reportRequest: ReportRequest): Promise<ReportCreateResponse> => {
  try {
    const res = await BACKEND_API.post('/secure/report/add', reportRequest);
    return { success: res.status >= 200 && res.status < 300, alreadyReported: false };
  } catch (err) {
    const status = (err as AxiosError).response?.status;
    if (status === 406) return { success: false, alreadyReported: true };
    logServiceError('report.sendReport', err);
    return { success: false, alreadyReported: false };
  }
};
```

- Endpoint: `POST /secure/report/add`
- The 406 → "already reported" path is handled; otherwise success is decided on 2xx. No other status codes are interpreted (the new backend typed `REPORT_*` codes from the 2026-05-28 trust-boundary fix are **not** mapped here — failure collapses to a generic toast; see §Seam 2 / known gap).

**Request type:** `src/lib/types/report/ReportRequest.ts`

```ts
export type ReportRequest = {
  reportedUserId?: number;
  reportedProductId?: number;
  description: string;
  reportOption: ReportOption;
  reportType: ReportType;
};
```

**There is no `reportedReviewId` field.** The only target-id slots are `reportedUserId` and `reportedProductId`.

**Body assembly:** `ReportButton` (`src/components/client/buttons/ReportButton.tsx`) forwards `reportedUserId` / `reportedProductId` props into the `REPORT_DIALOG` (`:46-53`). `ReportDialog` (`src/components/popups/dialogs/ReportDialog.tsx:98-104`) builds the request:

```ts
const result = await sendReport({
  reportType,
  reportOption: option,
  description: reportDescription,
  reportedUserId,
  reportedProductId,
});
```

`ReportButton`/`ReportDialog` do **no** per-type routing — whatever id the mounting component put in `reportedUserId`/`reportedProductId` is sent verbatim alongside the `reportType`.

### Full request shape per report type (from the actual mount sites)

| Report | reportType | `reportedUserId` | `reportedProductId` | description | reportOption |
|---|---|---|---|---|---|
| **USER** (`UserDetails.tsx:213`, `Messages.tsx:202`) | `ReportType.USER` | `userDetails.id` / `peer.id` | *(unset → undefined)* | textarea | `UserReportOption` |
| **PRODUCT** (`ProductFunctions.tsx:70`) | `ReportType.PRODUCT` | *(unset → undefined)* | `productDetails.id` | textarea | `ProductReportOption` |
| **REVIEW** (`ReceivedReviewCard.tsx:18`) | `ReportType.REVIEW` | *(unset → undefined)* | **`review.id`** | textarea | `ReviewReportOption` |

**Which field carries the review id today: `reportedProductId`.** (There is no review-specific slot.)

## B.7 — `ReportType` enum + where REVIEW reporting is surfaced

**Enum** (`src/lib/types/report/ReportType.ts`):

```ts
export enum ReportType {
  PRODUCT = 'PRODUCT',
  USER = 'USER',
  REVIEW = 'REVIEW',
}
```

`REVIEW` is present (alongside `USER`, `PRODUCT`).

**Every `ReportType.*` usage in the repo:**
- `ReportType.USER` → `src/components/client/UserDetails.tsx:213`, `src/messages/components/Messages.tsx:202`
- `ReportType.PRODUCT` → `src/components/client/ProductFunctions.tsx:70`
- `ReportType.REVIEW` → `src/components/owner/reviews/ReceivedReviewCard.tsx:20` **(the only REVIEW site)**

**REVIEW reporting is surfaced in exactly one place: `ReceivedReviewCard`** (the owner's "received reviews" list). Nowhere else.

Report DTO/body the service sends = `ReportRequest` (quoted in §B.6); related option types: `src/lib/types/report/ReportOption.ts`, `ReviewReportOption.ts`, `UserReportOption.ts`, `ProductReportOption.ts`.

---

# Seams to surface

### Seam 1 — Web keys product translation off `translationKey`, not `code`

- **What I found (fact):** the parser writes `byField[field] = err.translationKey`; the UI calls `tErrors(translationKey)`. `code` is analytics-only.
- **Implication for the rename question:** a backend **enum-home move** or a **`code`-string rename** does **not** touch web's product-error rendering. A **`translationKey` rename** is what touches web — and the only web-side hardcoded `translationKey` is `'product.system.rate_limited'` (`productService.ts:59`). So a `translationKey` rename is web-safe **except** if it renames `product.system.rate_limited`, in which case `productService.ts:59` must change in lockstep.
- **What I'd need from backend to confirm:** confirmation that the backend continues to emit a populated `translationKey` on every `{field, code, translationKey}` entry for 400/422/403/500 (web 100% depends on `translationKey` being present and correct), and that the `ERRORS`-namespace seed key `product.system.rate_limited` is not being renamed without a coordinated web change.

### Seam 2 — REVIEW report field contract (the core wire mismatch)

- **What I found (fact):** web sends the **review id** in `reportedProductId` with `reportType: REVIEW`. Web's `ReportRequest` has no `reportedReviewId`. REVIEW is mounted in one place only (`ReceivedReviewCard`).
- **My assumption:** the wire was scoped before the backend REVIEW branch existed; today the backend has only USER/PRODUCT branches, so a REVIEW submit is processed as a PRODUCT report against `getOwnerId(review.id)` — a silent mis-file or a `REPORTED_PRODUCT_NOT_FOUND` 422.
- **What I'd need from backend to confirm — the product decision (a/b/c):**
  - **(a) Build it:** does the backend get a `reportedReviewId` slot + a REVIEW branch (existence check against the reviews table, self-report guard, per-target dedupe key, a `REPORTED_REVIEW_ID_REQUIRED`-style code)? If so, web adds `reportedReviewId?: number` to `ReportRequest` and `ReceivedReviewCard` switches `reportedProductId={review.id}` → `reportedReviewId={review.id}`.
  - **(b) Remove the wire:** delete the `ReportButton` mount in `ReceivedReviewCard.tsx` (and optionally the `REVIEW` enum member + `ReviewReportOption`). Web-only.
  - **(c) Repurpose to PRODUCT:** if the intent is "this review is on a sketchy product," change the mount to `reportType={ReportType.PRODUCT}` + `reportedProductId={review.reviewedProduct.productId}` (the field exists on `ReviewDTO`). Web-only, one-line.
- **Decision owner:** product/Mastermind, not web. Web can execute any of (a)/(b)/(c) once chosen. (Matches `issues.md` 2026-05-28 "Review-reporting wire is an unfinished feature," status `parked`.)

### Seam 3 — Image-upload subsystem maps `code → key` (the one exception to Seam 1)

- **What I found (fact):** `src/lib/images/errorMapping.ts` (`errorCodeToTranslationKey`) is an explicit `code → image.* key` switch. Here web **does** branch on backend/Worker `code`. An image-pipeline **code rename** would break this mapping silently (unknown code → falls through to the default key).
- **Out of the brief's `product.system` scope**, surfaced because it is the lone place in the repo where the "translate off `code`" pattern lives, and the rename-impact answer for image codes is the opposite of the product answer.

### Adjacent observation (Part 4b)

- **`ReportRequest` failure responses are not mapped to typed codes.** `reportService.ts` only special-cases 406; the new backend `REPORT_*` codes (2026-05-28 trust-boundary fix) collapse to a generic `tErrors('report.send.fail')` toast in `ReportDialog.tsx:111`. Severity **low** (UX, not correctness). Not fixed — out of this audit's read-only scope; tracked as the W6 follow-up referenced in `issues.md`/`decisions.md` 2026-05-28.
