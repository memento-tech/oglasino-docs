# Audit — Mobile service layer, error contract, offline detection

**Repo:** oglasino-expo
**Branch:** new-expo-dev (confirmed on branch before reading)
**Date:** 2026-05-29
**Type:** READ-ONLY. No code changed, nothing staged, no installs.
**Scope:** Parts A–E of brief `audit-expo-service-error-contract`. Maps the post-`expo-boot-redesign` code (shipped 2026-05-29 on this branch) against the new four-enum error-code split and the review-report contract. Where the 2026-05-24 structural audit disagrees with current code, current code wins — see "Stale-audit corrections."

All line numbers are as they stand on `new-expo-dev` today.

---

## Part A — The HTTP layer

### A.1 — Location confirmed

The axios instance and its interceptors live at **`src/lib/config/api.ts`** — confirmed, unchanged from the old audit's location. The instance is built by `createApiInstance(baseURL: string)` (`api.ts:30-138`) and exported as `BACKEND_API` (`api.ts:140`). `axios.create` sets `baseURL`, an **8000 ms** timeout (`api.ts:33`), and base headers `Content-Type: application/json` / `Accept: application/json` (`api.ts:34-37`).

### A.2 — Request interceptor (`api.ts:40-51`)

Attaches, per request:
- **`Authorization: Bearer <idToken>`** (`api.ts:48`) from Firebase `auth.currentUser.getIdToken()`.
- **`X-Base-Site`** = `selectedBaseSite.code` when present (`api.ts:44`).
- **`X-Lang`** = `language.code` when present (`api.ts:45`).

`X-Account-Restored` is **not** a request header — it is read off the *response* (see A.3).

### A.3 — Response interceptor (`api.ts:53-135`) — status codes handled today

| Status / case | Lines | Behavior |
|---|---|---|
| 2xx success | 54-58 | If response carries `x-account-restored: true`, fire `authHooks.onAccountRestored?.()`. Return response unchanged. |
| **403 Forbidden** | 77-83 | If `isErrorWithCode(error, 'USER_BANNED')` **or** `isErrorWithCode(error, 'EMAIL_BANNED')`: sign out Firebase, fire `authHooks.onAccountBanned?.()`, return a never-resolving promise. Any other 403 falls through to the generic reject at line 133. |
| **401 Unauthorized** | 85-123 | Single-flight token refresh with a queue. If `_retry` already set → sign out + reject (89-92). If no `auth.currentUser` → reject (94). If a refresh is in flight → queue and await (96-104). Else `getIdToken(true)` (109), replay queued + original with the new token, set `_retry`, retry (110-114); on refresh failure → sign out + reject (115-120). |
| 404 Not Found | 125-131 | Wrap error as `{ errorCode: 'error.not.found', status: 404 }`. |
| Network: `ERR_NETWORK` / `ECONNABORTED` | 61-67 | Wrap as `{ status: 0, errorCode: 'error.connection.timeout' }`. |
| Network: no response at all | 69-75 | Wrap as `{ status: 0, errorCode: 'error.network.unreachable' }`. |
| Everything else | 133 | Passed through unchanged (raw `AxiosError`). |

**Φ1 (401/403) handling is present and matches the brief's expectation.** Confirmed against the `decisions.md` 2026-05-25 Φ1 Brief 7 entry.

### A.4 — Does `api.ts` parse the backend error body?

**No.** The interceptor never extracts or transforms the `{errors:[{field,code,translationKey}]}` array. The 403 branch *reads* `errors[0].code` indirectly via the `isErrorWithCode` helper (A.5), but only to test for two ban codes. Every other error is handed to callers untouched (or wrapped with a synthetic `errorCode` string for the network/404 cases, which are **not** backend wire codes).

### A.5 — The one helper that reads the error shape

**`src/lib/utils/isErrorWithCode.ts` (lines 1-9).** Signature `isErrorWithCode(error: unknown, code: string): boolean`. It reads `error.response.data.errors[0].code` and returns `true` iff the **first** error's `code` string-equals the supplied `code`.

```ts
const e = error as AxiosError<{ errors?: { code?: string }[] }>;
const errors = e.response?.data?.errors;
return errors?.[0]?.code === code;
```

Callers: `api.ts:78` (USER_BANNED / EMAIL_BANNED on 403), `authStore.ts` (same two codes during user sync), `ForegroundRevalidationInit.tsx` (same two codes on foreground revalidation).

**Nothing in the repo reads `field` or `translationKey`, and nothing iterates beyond `errors[0]`.** The helper keys on `code` only — which is exactly the contract the error-code split preserves (the `code` wire value is unchanged). So the Φ1 ban handling is **unaffected** by the four-enum split.

---

## Part B — Service-file inventory

**Path confirmed: `src/lib/services/`. File count: 17 (16 service modules + 1 test file `imageTokensService.test.ts`).** The old audit's "~17 files" is right if the test file is counted; 16 are actual services.

| Service | On backend error | Reads error body? |
|---|---|---|
| `appVersionConfigService.tsx` | swallow → `undefined` | no |
| `authService.ts` | mixed (Firebase-specific): rethrow on build failure; route Google sign-in by `GoogleSignin.statusCodes`; Firestore failures logged then continue; photo-upload swallows | no (Firebase status codes only) |
| `catalogService.ts` | log, then **rethrow** a custom error (no default) | no |
| `configurationService.tsx` | `logServiceError` → `null` | no |
| `favoritesService.ts` | `logServiceError` → default (`[]` / `{products:[], totalNumberOfProducts:0}`) | no |
| `followService.ts` | `logServiceError` → `false` | no |
| `healthCheckService.ts` | swallow → `false` | no |
| **`imageTokensService.ts`** | **reads `body.error.code`, packages into `ImageTokenError`, rethrows** | **YES — `code` only** (`imageTokensService.ts:108-110`, `:121`) |
| `maintenanceService.tsx` | `logServiceError` → `true` (fail-safe = "in maintenance") | no |
| `openAiService.ts` | `logServiceError` → `undefined` | no |
| `productService.ts` | `logServiceError` → default (`null` / `false` / `0`) | no |
| `productsSearchService.ts` | `logServiceError` → default (`null` / empty page) | no |
| **`reportService.ts`** | maps **HTTP status only** (406 vs 2xx) into a flag object | no (status code only — see Part C) |
| `reviewService.ts` | `logServiceError` → `false`; orphan-image cleanup on persist failure | no |
| `suggestionsService.ts` | swallow → `false` | no |
| `userService.ts` | `logServiceError` → default (`true` / `false` / `null`) | no |

**Dominant pattern (~14 of 16):** catch → `logServiceError(...)` → return a sentinel default. The structured backend error body is discarded; callers see only the sentinel.

**Outliers worth naming:**
1. **`imageTokensService.ts`** — the *only* service that inspects the error body. It reads `code` (not `field`, not `translationKey`) from `body.error.code` / `e.response.data.error.code` and rethrows a typed `ImageTokenError` so upload callers can branch on token-specific failures. Note: it reads `data.error.code` (singular `error` object), **not** the `data.errors[]` array shape — a different envelope from the Part-7 contract.
2. **`reportService.ts`** — branches on HTTP **status** (`406` = already-reported) and ignores the body entirely.
3. **`catalogService.ts`** — the only non-image service that rethrows instead of returning a default.
4. **`authService.ts`** — Firebase/Google-SDK error space, not the backend wire contract.

**Does any service read `field`, `code`, or `translationKey` off the wire error body?** Only `imageTokensService.ts`, and only `code` (off the singular `error` object, not the `errors[]` array). **No service reads `field` or `translationKey`. No service consumes the `{errors:[{field,code,translationKey}]}` array.**

---

## Part C — reportService

`src/lib/services/reportService.ts` read in full (28 lines). Call chain: `ReportButton` → opens `REPORT_DIALOG` → `ReportDialog.handleSubmit` → `sendReport`.

### C.1 — Boolean-inversion bug (old Finding 24): CONFIRMED PRESENT

`reportService.ts` in full:

```ts
export type ReportCreateResponse = {
  success: boolean;
  alreadyReported: boolean;
  error: boolean;
};
export const sendReport = async (reportRequest: ReportRequest): Promise<ReportCreateResponse> => {
  try {
    const res = await BACKEND_API.post('/secure/report/add', reportRequest);
    return {
      success: res.status >= 200 && res.status < 300,
      alreadyReported: res.status === 406,
      error: res.status !== 406 || (res.status >= 200 && res.status < 300),   // line 16
    };
  } catch (err) {
    const status = (err as { status?: number; response?: { status?: number } })?.status
      ?? (err as { response?: { status?: number } })?.response?.status;
    return {
      success: false,
      alreadyReported: status === 406,
      error: status !== 406 || (status !== undefined && status >= 200 && status < 300),   // line 24
    };
  }
};
```

`error` (lines 16 and 24) is inverted. Truth table:
- **2xx success:** `status !== 406` is `true` → `error = true`. **Should be `false`.**
- **406 already-reported:** `status !== 406` is `false`, second clause `false` → `error = false`.
- **Other error (e.g. 422/500):** `status !== 406` is `true` → `error = true` (incidentally correct).

So `error` is effectively `!alreadyReported` — `true` on success and on real errors, `false` only on 406. **Runtime impact today is nil:** the sole consumer, `ReportDialog.handleSubmit` (`ReportDialog.tsx:80-86`), branches on `result.success` then `result.alreadyReported` and **never reads `result.error`**. The field is dead but wrong. (Logged as an adjacent observation, not fixed — read-only brief.)

### C.2 — Does the request carry `reportType`, `reportedUserId`, `reportedProductId`?

Yes. `ReportRequest` (`src/lib/types/report/ReportRequest.ts:4-10`):

```ts
export type ReportRequest = {
  reportedUserId?: number;
  reportedProductId?: number;
  description: string;
  reportOption: ReportOption;
  reportType: ReportType;
};
```

`ReportDialog.handleSubmit` sends `{ reportType, reportOption, description, reportedUserId, reportedProductId }` (`ReportDialog.tsx:72-78`).

### C.3 — Does it carry `reportedReviewId` or send literal `"REVIEW"`?

- **`reportedReviewId`: ABSENT.** Repo-wide search for `reportedReviewId` returns zero hits. The field exists on neither `ReportRequest`, `ReportButton`'s props, nor `ReportDialog`'s props.
- **`ReportType.REVIEW` enum value EXISTS** (`src/lib/types/report/ReportType.ts:1-5`: `PRODUCT`, `USER`, `REVIEW`). It is passed as `reportType={ReportType.REVIEW}` from the review cards (C.4). No code sends the bare string `"REVIEW"` — only the enum member, which serializes to `"REVIEW"` on the wire. So the wire **does** carry `reportType: "REVIEW"`, but with **no review-id field accompanying it.**

### C.4 — Report-review completeness check (this decides scope)

**Verdict: ~35% wired — a report-review surface exists on dashboard review cards, but the review id is sent in the *wrong field* and there is no `reportedReviewId` field at all. Public reviews have no report surface.**

End-to-end walk:

**Surfaces that mount a review report:**
- `src/components/dashboard/components/ReceivedReviewCard.tsx:17-22` — mounts `<ReportButton reportType={ReportType.REVIEW} ... reportedProductId={review.id} />`.
- `src/components/dashboard/components/GivenReviewCard.tsx:24-29` — same, on unapproved given reviews.
- Both are reached only via `src/components/dashboard/components/OwnerReviewList.tsx` (the owner dashboard "my reviews" list).

**Surface that does NOT exist:**
- `src/components/product/ProductReview.tsx` — the card used by the public `ReviewsList` (on the user/product page) — mounts **no `ReportButton`**. So a logged-in user browsing another user's public reviews has **no way to report a review.**

**Where the review id flows (the bug):**
- `ReceivedReviewCard.tsx:22` and `GivenReviewCard.tsx:29` pass **`reportedProductId={review.id}`** — i.e. the `ReviewDTO.id` is smuggled into the **product-id slot.**
- Neither card passes `reportedUserId`. `ReportButton` declares it `reportedUserId?: number` (optional) so this compiles, but `ReportDialog` declares it **required** (`reportedProductId: number`, `reportedUserId: number` — `ReportDialog.tsx:20-21`). So a review report sends `reportType: "REVIEW"`, `reportedProductId: <reviewId>`, and `reportedUserId: undefined`.
- `sendReport` posts that to `/secure/report/add`.

**Against the new backend contract** (`ReportType.REVIEW` + dedicated `reportedReviewId`, plus `REPORTED_REVIEW_ID_REQUIRED` / `REPORTED_REVIEW_NOT_FOUND` codes): the backend now expects the review id in `reportedReviewId`. Mobile sends it in `reportedProductId` and leaves `reportedReviewId` unset, so a REVIEW report will fail backend validation with `REPORTED_REVIEW_ID_REQUIRED` (and, since mobile never reads `field`/`code`/`translationKey`, it will surface only the generic `report.send.fail` toast).

**What is present:** the `REVIEW` enum value; two dashboard report buttons wired with `reportType={ReportType.REVIEW}`.
**What is missing:** the `reportedReviewId` field on `ReportRequest` / `ReportButton` / `ReportDialog`; the re-pointing of the cards from `reportedProductId` to `reportedReviewId`; a public-review report surface in `ProductReview.tsx`; and mapping of the two new REVIEW error codes to user-facing text.

**Scope call (for the seam analysis):** this is *not* a "finish the last 10%" job — it is a field-contract change (new DTO field + re-point two call sites) plus an absent public surface plus error-code mapping. The dashboard wire exists but points at the wrong field. Reasonable either as its own small `oglasino-expo-review-reports` adoption or folded into a broader report/messaging mobile chat; it is more than a one-line fix.

---

## Part E — Trust boundaries (report call)

Mobile makes **zero local trust, moderation, or authorization decisions** off any report id; all are sent as-is and the backend is the sole trust boundary. `reportedUserId` originates as a `ReportButton` prop from the caller (e.g. `ProductUserDetails.tsx` passes `userDetails.id`, view data from an API response) → `ReportDialog` prop → `sendReport` body, with no ownership/eligibility check. `reportedProductId` likewise flows caller → prop → body untouched (and on review reports carries `review.id`, per C.4). `reportedReviewId` does not exist; the review id is currently smuggled through `reportedProductId`. All three are opaque numbers taken from API responses or user-selected UI elements and forwarded verbatim — mobile relies entirely on the backend to validate existence, ownership, and self-report rules. This is the expected posture (consistent with conventions Part 11): no client-side trust decision to remove.

---

## Part D — Offline vs maintenance

### D.1 — Where maintenance is decided now (Gate 1)

The old `AppContext.tsx` 5-second poll is **deleted**. Maintenance is now decided at **Gate 1 of the boot state machine**, `runMaintenanceGate` in **`src/lib/store/bootStore.ts:137-148`**:

```ts
runMaintenanceGate: async () => {
  try {
    const maintenanceOn = await withGateTimeout(() => checkIfMaintenance());
    if (maintenanceOn) { get().toMaintenance(); return; }
  } catch {
    // Timeout (GateTimeoutError) or any other throw → maintenance.
    get().toMaintenance(); return;
  }
  ...
```

- **Endpoint:** `GET /public/maintenance/active` via `checkIfMaintenance` (`maintenanceService.tsx:6`). (Brief shorthand "`/maintenance/active`" — actual path is `/public/maintenance/active`.)
- **Timeout:** `GATE_TIMEOUT_MS = 5000` (`bootGate.ts:13`), applied by `withGateTimeout` (`bootGate.ts:31-48`), which `reject`s a `GateTimeoutError` if the call never settles.
- **Unreachable → maintenance:** two layers conspire. `checkIfMaintenance` returns `true` on *any* caught error (`maintenanceService.tsx:13,16`) — fail-safe "in maintenance." For a genuine hang that never settles, the 5 s timer fires `GateTimeoutError`, caught by the gate's `catch` (`bootStore.ts:144-147`) → `toMaintenance()`. `toMaintenance` is just `set({ status: 'maintenance' })` (`bootStore.ts:524`).

### D.2 — netinfo / network-state library

**ABSENT.** Not in `package.json` (verified — no `netinfo` match) and no `NetInfo` / `@react-native-community/netinfo` import anywhere in `src/` (verified by recursive grep). The app has no network-state awareness.

### D.3 — Cold start in airplane mode (traced)

1. `app/_layout.tsx:40` calls `useBootStore.getState().start()`.
2. `start()` (`bootStore.ts:115-121`) sets `status: 'booting'` then `await runMaintenanceGate()`.
3. Gate 1 calls `withGateTimeout(() => checkIfMaintenance())`. With no network the request to `/public/maintenance/active` never settles (or rejects as a network error → `checkIfMaintenance` catch → `true`).
4. Either way the gate routes to maintenance: the catch at `bootStore.ts:144-147` (on `GateTimeoutError`) or the `maintenanceOn === true` branch at 140-142.
5. `toMaintenance()` sets `status: 'maintenance'` (`bootStore.ts:524`).
6. `app/_layout.tsx:116` renders `{bootStatus === 'maintenance' && <BaseSiteSelector isMaintenance />}`.

**Confirmed: with no network, the user lands on the MAINTENANCE screen. There is no offline-specific screen — "offline" and "backend down" are indistinguishable to the user today.** (This is exactly old Finding 6's concern, now relocated from `AppContext` to `bootStore` Gate 1.)

### D.4 — Where a connectivity check would insert (located, not designed)

The check must run **before** Gate 1 maps unreachable → maintenance. Concrete insertion point: **`bootStore.ts:120`**, immediately before `await get().runMaintenanceGate()` inside `start()` — either an inline connectivity probe there, or a new "Gate 0" function chained ahead of `runMaintenanceGate`. It would route to a new `'offline'` `BootStatus` (the `BootStatus` union is defined near `bootStore.ts:70`), which `app/_layout.tsx`'s overlay block (the same `bootStatus === ...` chain at lines 116-123) would render as a dedicated offline screen. No fix designed here — just the seam.

### D.5 — The existing maintenance screen (offline screen will mirror it)

**`src/components/init/BaseSiteSelector.tsx`**, the `isMaintenance={true}` branch. It renders: the Oglasino icon, a welcome title, a subtitle, and two maintenance copy lines (≈ "We're improving our services" / "Please try again later — thank you for your patience"), over the `intro.jpg` background with a dark overlay and a centered card. The base-site selection buttons are gated by `{!isMaintenance}` so they do not render in maintenance mode. An offline screen would reuse this exact layout with offline-specific text keys. Note `app/_layout.tsx:117` already reuses `BaseSiteSelector` for the `intro-picker` status, so this component is the established "full-screen boot message" surface.

---

## Stale-audit corrections (vs the 2026-05-24 structural audit)

Places where current `new-expo-dev` code differs from the archived `audit-expo-structural.md` on the files touched here:

1. **Maintenance location (old Finding 6).** Old audit: a 5-second poll in `src/components/context/AppContext.tsx:143-162`, catch at 157-160 sets `status:'maintenance'` on any error; "720 calls/hour regardless of app state." **Now:** `AppContext.tsx` is deleted. Maintenance is a one-shot boot gate (`bootStore.ts:137-148`) with a 5 s timeout; the steady-state poll while *in* maintenance is a separate component, `MaintenancePollInit.tsx` (runs only while `status === 'maintenance'`, to detect maintenance-clear). The "720 calls/hour from ready" cost is gone.
2. **401/403 handling (old Finding 5).** Old audit: "the response interceptor handles `ERR_NETWORK`, `ECONNABORTED`, missing response, and 404. It does not handle 401 or 403." **Now:** `api.ts` handles 401 (single-flight refresh+retry, `:85-123`) and 403 (USER_BANNED/EMAIL_BANNED, `:77-83`) as well — added by Φ1 Brief 7 (decisions.md 2026-05-25). The network/404 handling the old audit named is still present.
3. **"No service reads the error body" (old Finding 18).** Old audit: "No service parses `field`, `code`, or `translationKey`." **Now mostly still true, with one exception:** `imageTokensService.ts` reads `code` (off the singular `data.error.code`, not the `errors[]` array) and rethrows a typed error. `field` and `translationKey` are still read by nothing. (The image-pipeline work added this after the old audit.)
4. **reportService boolean inversion (old Finding 24).** Old audit pointed at `reportService.ts:14-16` and `:22-24`. **Still present, same logic, lines 16 and 24 on this branch.** Unchanged — confirmed verbatim in C.1.
5. **Services path/count.** Old audit: `src/lib/services/`, "17 service files." **Confirmed path; 16 services + 1 test file** = 17 entries. No drift.
6. **Axios location.** Old audit: `BACKEND_API` single global instance in `src/lib/config/api.ts`. **Confirmed, unchanged.**

---

## Adjacent observations (read-only — not fixed)

- **`reportService.ts:16,24` — `error` flag is inverted and dead.** `error` is `true` on success. No consumer reads it (`ReportDialog` uses only `success`/`alreadyReported`), so no runtime impact today, but the field is wrong and will mislead the next consumer. Severity: low. Out of scope for this read-only audit. (Old Finding 24, still open.)
- **Review-report field mismatch — `ReceivedReviewCard.tsx:22`, `GivenReviewCard.tsx:29`.** Review id is sent in `reportedProductId`; no `reportedReviewId` exists; `reportedUserId` is not passed though `ReportDialog` types it as required. Against the new backend contract this makes REVIEW reports fail validation. Severity: high (user-facing — review reporting is broken end-to-end). This is the core finding for Φ4 / a review-reports adoption chat, not an incidental fix.
- **No public-review report surface — `ProductReview.tsx`.** Public reviews (user/product page) cannot be reported at all; only dashboard "my reviews" cards carry a button. Severity: medium (feature gap, parity with web's `ReceivedReviewCard` report surface). Flag for Mastermind to decide whether mobile review-reporting parity includes public reviews.
- **Generic error surfacing.** Because no service (bar `imageTokensService`) reads `code`/`translationKey`, every backend validation/business error collapses to a sentinel and a generic toast (e.g. `report.send.fail`). This is the structural gap Φ4 exists to address; noted here as the through-line behind Parts A, B, and C.
