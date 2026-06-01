# Audit — oglasino-web create-flow: pre-validate gate + step-4 failure UX

**Repo:** oglasino-web
**Branch:** `stage`
**Date:** 2026-05-29
**Mode:** READ-ONLY. No code changed, no git ops, no runtime execution.
**Feature spec:** [`oglasino-docs/features/product-validation.md`](../../oglasino-docs/features/product-validation.md)
**Purpose:** Capture the exact web shape of the two behaviors the mobile (Expo)
validation rebuild mirrors — the step-2→step-3 pre-validate gate and the step-4
submit-failure UX — plus the `productStepMapping` and
`parseProductValidationErrors` helpers, beyond the spec's prose.

## Files read

- `src/lib/service/reactCalls/productService.ts` (service layer — both results + cleanup)
- `src/components/popups/components/BasicInfoProductDialog.tsx` (step 2 — pre-validate gate)
- `src/components/popups/components/UploadedProductDialog.tsx` (step 4 — submit + failure UX)
- `src/components/popups/dialogs/CreateNewProductDialog.tsx` (wizard container — step indexing)
- `src/lib/utils/productStepMapping.ts` (step-routing helper)
- `src/lib/utils/parseProductValidationErrors.ts` (parse helper)
- `src/lib/types/product/ProductErrorResponse.ts` (wire type)

---

## Section 1 — Pre-validate call and the step-2→step-3 gate

### 1a. `preValidateProductBasics` — signature, request, result shape

`src/lib/service/reactCalls/productService.ts:259`

```typescript
export const preValidateProductBasics = async (
  name: string,
  description: string
): Promise<PreValidateResult> => {
```

- **Sends:** `POST /secure/products/pre-validate` with body `{ name, description }`
  (exactly two fields; no `language` — resolved server-side). Line 264.
- **Returns** a discriminated union (`productService.ts:34-37`):

```typescript
type PreValidateResult =
  | { type: 'clean' }
  | { type: 'validation'; errors: ParsedProductValidationErrors }
  | { type: 'error' };
```

The exact `type` string values are **`'clean'` | `'validation'` | `'error'`**
(confirmed verbatim — these match the spec). Variant payloads:

- **`clean`** — carries nothing. Returned when `status === 200` and `errors` is empty (line 270).
- **`validation`** — carries `errors: ParsedProductValidationErrors`, i.e. the
  `{byField, list}` dual-view (see Section 4). Returned for `200`-with-violations
  (line 271), `400` (Jakarta, line 274/292), and `429` (rate-limit, line 278/295,
  routed through `parseProductErrorsForStatus` which injects the `__system` entry).
- **`error`** — carries nothing. The fall-through for transport failures, `403`,
  `5xx`, and any unrecognized/unparseable response (lines 283, 298).

Status→variant table as implemented:

| Status / condition                  | Result variant                         |
| ----------------------------------- | -------------------------------------- |
| 200, `errors: []`                   | `{type:'clean'}`                       |
| 200, `errors: [...]`                | `{type:'validation'}`                  |
| 400 (Jakarta), parseable body       | `{type:'validation'}`                  |
| 429, parseable body                 | `{type:'validation'}` (`__system` injected) |
| 403 / 500 / network / unparseable   | `{type:'error'}`                       |

### 1b. The step-2 "Next" handler — `BasicInfoProductDialog.onNextInternal`

`src/components/popups/components/BasicInfoProductDialog.tsx:79`. Gate ordering is
**re-entrancy guard → trim → Zod structural → pre-validate → branch on result**:

```typescript
const onNextInternal = async () => {
  if (preValidating || isRateLimited) return;

  productData.name = trimText(productData.name || '');
  productData.description = trimText(productData.description || '');

  const zodErrors = await validateProduct(
    productData.name, productData.description, productData.price,
    productData.topCategory, productData.subCategory, productData.finalCategory,
    productData.imagesData, false
  );

  if (zodErrors['name'] || zodErrors['description'] || zodErrors['images'] ||
      zodErrors['topCategory'] || zodErrors['subCategory'] ||
      zodErrors['finalCategory'] || zodErrors['price']) {
    setProductErrors(zodErrors);
    return;                       // structural fail → render inline, no pre-validate
  }

  setProductErrors({});
  setPreValidating(true);
  const result = await preValidateProductBasics(productData.name, productData.description);
  setPreValidating(false);

  if (result.type === 'clean') {
    onNextStep();                 // → advance to step 3
    return;
  }

  if (result.type === 'validation') {
    setProductErrors(result.errors.byField);          // inline per-field render
    for (const err of result.errors.list) {
      track('form_submit_failed', { form_name: 'product_create_step_1',
        error_code: err.code, field: err.field });    // analytics from `list`
    }
    if (result.errors.byField[SYSTEM_ERROR_KEY]) {     // rate-limit present
      setIsRateLimited(true);
      if (rateLimitTimerRef.current) clearTimeout(rateLimitTimerRef.current);
      rateLimitTimerRef.current = setTimeout(() => {
        setProductErrors((prev) => {
          if (!prev[SYSTEM_ERROR_KEY]) return prev;
          const next = { ...prev };
          delete next[SYSTEM_ERROR_KEY];
          return next;
        });
        setIsRateLimited(false);
      }, RATE_LIMIT_BACKOFF_MS);
    }
    return;
  }

  // result.type === 'error' — transport/5xx → warn + advance anyway
  notify.warning({
    id: 'product-pre-validate-warning',
    title: tDialog('new.product.pre.validate.warning'),
  });
  onNextStep();
};
```

**On `clean`** → `onNextStep()` (the parent's `handleNext`, increments the
0-indexed step → step 3). Line 114-117.

**On `validation`:**
- `result.errors.byField` is written straight into `productErrors` state →
  each field renders its error **inline** next to that field (e.g. name at
  `:205` `errorMessage={productErrors['name'] ? tErrors(productErrors['name']) : undefined}`,
  collapsed category error at `:193`, etc.). So both `name` and `description`
  violations render at once, each beside its field.
- The **`__system` (rate-limit) entry** renders separately, centered under the
  action bar (`:292-296`):
  ```tsx
  {productErrors[SYSTEM_ERROR_KEY] && (
    <p className="mt-1 text-center text-xs text-red-500">
      {tErrors(productErrors[SYSTEM_ERROR_KEY]!)}
    </p>
  )}
  ```
- **Next disabled?** Yes. The button has `disabled={preValidating || isRateLimited}`
  (`:308`). `isRateLimited` flips true on a `__system` entry and is cleared by a
  `setTimeout` after **`RATE_LIMIT_BACKOFF_MS = 5000`** (`:40`) — a **fixed 5s
  backoff**, after which the `__system` error is also deleted from state.
  (Next is also disabled transiently while `preValidating` is true.)

**On `error`** → **advances to step 3 anyway** via `onNextStep()`, after a warning
toast. Toast key (DIALOG namespace): **`tDialog('new.product.pre.validate.warning')`**
→ fully-qualified **`DIALOG.new.product.pre.validate.warning`** (`:148`). Confirms
the spec: pre-validate is a UX optimization, not a boundary.

> **Step 3 (`MetaDataProductDialog`)** itself runs only reCAPTCHA on Next — no
> content validation — then calls `onNextStep()`. (Out of scope; one-line note.)

---

## Section 2 — Step-4 submit + failure UX (`UploadedProductDialog`)

### 2a. Result classification

`UploadedProductDialog.tsx` fires once on mount (`triggeredRef` guard) and calls
`createNewProduct(productData, {onProgress})`. The create result is the **same
`ParsedProductValidationErrors` failure shape as pre-validate, but a different
happy-path tag** — `success` (with data), not `clean`:

`productService.ts:23-26`

```typescript
type ProductSubmitResult =
  | { type: 'success'; data: NewProductResponseDTO }   // NewProductResponseDTO = {id:number; name:string}
  | { type: 'validation'; errors: ParsedProductValidationErrors }
  | { type: 'error' };
```

Classification in the dialog (`:64-97`):
- `success` → `setUploadSuccess(true)`, build product URL, `track('product_create_completed')`, `onFinish()`.
- `validation` → `setValidationErrors(result.errors.byField)`, `track('form_submit_failed', {form_name:'product_create_step_4', error_code, field})` per `list` item, `setUploadSuccess(false)`.
- `error` → `setValidationErrors({})` (empty), `setUploadSuccess(false)`.
- Thrown `UploadError` (image upload failed, not persistence) → one localized
  toast via `buildUploadErrorTitle(error, tErrors)`, then `setUploadSuccess(false)`
  with empty validation errors (falls into the generic-error UX).

Classification source (`productService.createNewProduct`, `:170-197`): 2xx →
`success`; status in `PARSEABLE_ERROR_STATUSES = {400,403,422,429,500}` with a
contract-shaped body → `validation`; everything else → `error`. The axios
interceptor re-throws non-2xx, so the `catch` block repeats the same
parseable→`validation` / else→`error` split.

### 2b. Validation-failure list rendering

`UploadedProductDialog.tsx:189-200`:

```tsx
{uploadSuccess === false && (
  <div className="flex w-full flex-col items-center justify-center gap-5">
    <TriangleAlert color="red" className="scale-250" height={50} />
    {hasValidationErrors ? (
      <div className="flex w-full flex-col items-center gap-3">
        <p className="text-center">{tDialog('new.product.create.failed.header')}</p>
        <ul className="w-full list-disc pl-6 text-sm text-red-500">
          {errorEntries.map(([field, key]) => (
            <li key={field}>{tErrors(key!)}</li>
          ))}
          {systemError && <li>{tErrors(systemError)}</li>}
        </ul>
      </div>
    ) : (
      <p className="text-center">
        {tDialog('new.product.success.failed.1')}
        <br />
        {tDialog('new.product.success.failed.2')}
      </p>
    )}
    ...
```

- **Header key:** `tDialog('new.product.create.failed.header')` →
  **`DIALOG.new.product.create.failed.header`** (matches spec).
- **Each error's translated message is listed**, one `<li>` per entry, via
  `tErrors(key)` (ERRORS namespace). It is **first-per-field** — `validationErrors`
  is the `byField` map, which already collapsed to one entry per field (see §4).
- The list is built from (`:128-132`):
  ```typescript
  const hasValidationErrors = Object.keys(validationErrors).length > 0;
  const errorEntries = Object.entries(validationErrors).filter(([field]) => field !== SYSTEM_ERROR_KEY);
  const systemError = validationErrors[SYSTEM_ERROR_KEY];
  ```

### 2c. The two buttons

`:209-215`:

```tsx
<Button variant="outline" onClick={handleGoBackAndFix}>
  {tDialog('new.product.create.failed.fix')}
</Button>
<Button variant="outline" onClick={onClose}>
  {tDialog('new.product.create.failed.exit')}
</Button>
```

- **"Go back and fix":** `DIALOG.new.product.create.failed.fix`. Handler (`:134-140`):
  ```typescript
  const handleGoBackAndFix = () => {
    if (hasValidationErrors) {
      onJumpToStep(resolveTargetStep(validationErrors));   // earliest mapped step
    } else {
      onJumpToStep(DEFAULT_FALLBACK_STEP);                 // = 3
    }
  };
  ```
  `onJumpToStep` is the parent's `handleJumpToStep`, which converts 1-indexed→0-indexed (see §3).
- **"Exit":** `DIALOG.new.product.create.failed.exit` → `onClose()` (closes popup).

### 2d. The `error` (transport/5xx) path — orphan cleanup + same two-button UX

- **Cleanup is SERVICE-owned, not dialog-owned.** `createNewProduct` holds the
  `newKeys` (freshly-uploaded R2 keys this session) and calls
  `cleanupOrphanImages(newKeys).catch(() => {})` itself before returning `error`
  (`productService.ts:182-185` for non-2xx, `:188-190` in the catch). The dialog
  never calls `cleanupOrphanImages` and never sees the keys.
- On `error`, `validationErrors` is `{}` → `hasValidationErrors` is false → the
  generic failure copy renders (`DIALOG.new.product.success.failed.1` + `.2`),
  with the **same two-button bar**. "Go back and fix" → `handleGoBackAndFix` →
  empty-errors branch → `onJumpToStep(DEFAULT_FALLBACK_STEP)` = **fixed step-3
  jump**. Confirms the spec.

> ⚠ **Divergence (see For Mastermind D1):** cleanup also runs on the
> **`validation`** branch (`:177`), not only on `error`. Web cleans uploaded keys
> on *every* non-2xx create result.

### 2e. How a `__system` (429) entry interleaves with field errors

- Injected by `parseProductErrorsForStatus` (`productService.ts:61-74`) — only on
  `429` and only if no `__system` entry already exists — appended to the **end** of
  both `byField` (as `SYSTEM_ERROR_KEY`) and `list`:
  ```typescript
  return {
    byField: { ...parsed.byField, [SYSTEM_ERROR_KEY]: RATE_LIMITED_TRANSLATION_KEY },
    list: [...parsed.list, { field: null, code: 'RATE_LIMITED', translationKey: RATE_LIMITED_TRANSLATION_KEY }],
  };
  ```
- In the rendered list (§2b), field errors render first (`errorEntries`, with
  `__system` filtered out), then the single `{systemError && <li>...}` renders
  **last**. There is **no backoff / disable** at step 4 (unlike step 2) — the
  rate-limit message simply appears as the last bullet and the two buttons stay live.

---

## Section 3 — `productStepMapping.ts` (verbatim, full file)

```typescript
const STEP_FOR_FIELD: Record<string, number> = {
  images: 1,
  name: 2,
  description: 2,
  price: 2,
  currency: 2,
  topCategory: 2,
  subCategory: 2,
  finalCategory: 2,
};

export const DEFAULT_FALLBACK_STEP = 3;

export const stepForField = (field: string): number | undefined => {
  if (STEP_FOR_FIELD[field] !== undefined) return STEP_FOR_FIELD[field];
  // Filter errors arrive keyed as `filters`, `filter`, or `filter.<id>` etc.
  if (field === 'filters' || field === 'filter' || field.startsWith('filter.')) return 3;
  return undefined;
};

export const resolveTargetStep = (errors: Partial<Record<string, string>>): number => {
  for (const field of Object.keys(errors)) {
    const step = stepForField(field);
    if (step !== undefined) return step;
  }
  return DEFAULT_FALLBACK_STEP;
};
```

- **Signatures:**
  - `stepForField(field: string): number | undefined`
  - `resolveTargetStep(errors: Partial<Record<string, string>>): number` — note it
    consumes the **`byField` map** (keys only), not the `list`.
  - `DEFAULT_FALLBACK_STEP = 3` (exported const). `STEP_FOR_FIELD` is module-private.
- **Field→step (as implemented):** `images`→1; `name`/`description`/`price`/
  `currency`/`topCategory`/`subCategory`/`finalCategory`→2; filter fields→3.
- **Filter matching is BROADER than the spec.** Spec says "any field starting with
  `filter.`"; code also matches exact `'filters'` and exact `'filter'`. (For Mastermind D3.)
- **`__system`-only / unmappable → fallback step 3.** `stepForField` returns
  `undefined` for `'__system'` and any unknown field (there is no explicit
  `__system` branch); `resolveTargetStep` skips undefined and returns
  `DEFAULT_FALLBACK_STEP = 3` when nothing maps. Confirms spec.
- **1-indexed.** File header comment states callers on 0-indexed step state subtract
  1. Confirmed at the consumer: `CreateNewProductDialog.handleJumpToStep` does
  `setCurrentStep(Math.max(0, step - 1))` (`CreateNewProductDialog.tsx:138-140`),
  and `currentStep` state is **0-indexed** (`useState(0)`, `:40`; `handleNext` =
  `prev + 1`, `handlePrevious` = `prev - 1`). So the subtract-1 happens exactly
  once, at the wizard container's jump handler.

---

## Section 4 — `parseProductValidationErrors.ts` (verbatim, full file)

```typescript
export const SYSTEM_ERROR_KEY = '__system';

export type ParsedProductError = {
  field: string | null;
  code: string;
  translationKey: string;
};

export type ParsedProductValidationErrors = {
  byField: Partial<Record<string, string>>;
  list: ParsedProductError[];
};

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
    byField[key] = err.translationKey;
    list.push({ field: err.field, code: err.code, translationKey: err.translationKey });
  }
  return { byField, list };
}
```

Wire type it consumes (`ProductErrorResponse.ts`): `{ errors: ProductFieldError[] }`
where `ProductFieldError = { field: string | null; code: string; translationKey: string }`.

- **Return shape:** `{ byField, list }`.
  - **`byField: Partial<Record<string, string>>`** — field name → `translationKey`.
    A `field: null` entry collapses under **`SYSTEM_ERROR_KEY` (`'__system'`)**.
    This map is what the **render layer** (step-2 inline + `__system` line; step-4
    `<ul>`) and the **step-router** (`resolveTargetStep`) both consume.
  - **`list: ParsedProductError[]`** — `{field, code, translationKey}` per entry.
    `field` is **preserved as-is** here (stays `null`, NOT mapped to `__system` —
    an intentional asymmetry vs `byField`). This is what the **analytics** layer
    consumes (`track('form_submit_failed', {error_code: err.code, field: err.field})`).
- **First-per-field wins across BOTH views** via the shared `seen` Set keyed by
  `field ?? '__system'`. So at most one `__system` bucket survives, and one entry per field.
- **Exports:** `SYSTEM_ERROR_KEY`, `ParsedProductError`, `ParsedProductValidationErrors`,
  `parseProductValidationErrors`.

### Mapping to mobile's `parseServiceError`

Mobile's helper returns `{errors, byField}`; web returns `{byField, list}`. They
are isomorphic: **mobile `errors` ↔ web `list`**, **`byField` identical in intent**.
Web's consumption pattern mobile can mirror 1:1 with its own helper:
- **Render** off `byField` (per-field inline + a `__system`/`SYSTEM_ERROR_KEY` line/bullet).
- **Route** off `byField` keys (`resolveTargetStep(byField)`).
- **Analytics** off `list`/`errors` (raw `code` + nullable `field`).

The only thing to match exactly is the reserved key string **`'__system'`** and the
first-per-field collapse semantics.

---

## For Mastermind — divergences (code vs spec prose) + notes

The brief asks which authority wins on each point. Mobile mirrors the **code**;
these are flagged so the spec can be corrected or the code treated as the bug.

- **D1 — Orphan cleanup fires on `validation` failures, not only `error`.**
  Spec §"Step 4 → On failure" prose attaches `cleanupOrphanImages` only to
  `result.type === 'error'`. Code (`productService.ts:177`, inside the
  `validation` branch; also `:182-185` and the `:188-190` catch) cleans `newKeys`
  on **every non-2xx create result**, and it's **service-owned**, not done in
  `UploadedProductDialog`. Severity: low-medium — the code behavior is safer
  (a 422 also orphans the just-uploaded R2 objects). Recommend: spec prose updated
  to "clean uploaded keys on any non-success create result"; mobile mirrors the code.
  Consequence mobile must know: after a step-4 `validation` failure the uploaded
  images are already gone, so "Go back and fix" → resubmit re-uploads.

- **D2 — 429-synth fallback translation key is `'system.rate_limited'`, not
  `'product.system.rate_limited'`.** `productService.ts:59`
  `RATE_LIMITED_TRANSLATION_KEY = 'system.rate_limited'`. Per the spec error-code
  table and Part 7, the rate-limit key is **`product.system.rate_limited`**, and
  the render is `tErrors(translationKey)` where `tErrors = useTranslations(ERRORS)`
  is invoked with the **fully-qualified** key. The synth only fires for an
  edge-router 429 that lacks a Spring-shaped body (the `:52-58` comment documents
  this), so a real Spring 429 (which carries the proper key on the wire) renders
  fine — but on the edge-429 path `tErrors('system.rate_limited')` will not resolve
  to the seeded message. This looks like a **real, pre-existing web bug**, not just
  a spec mismatch. I could not confirm the backend seed key from this repo
  (web-only, read-only) — Mastermind should confirm the seeded key and decide
  whether web's synth key is corrected to `product.system.rate_limited` (and mobile
  uses the corrected value). Severity: medium.

- **D3 — `stepForField` filter-matching is broader than spec.**
  Spec: "any field starting with `filter.`". Code (`productStepMapping.ts:26`) also
  matches exact `'filters'` and exact `'filter'`. Harmless superset; code wins;
  mobile mirrors all three forms. Severity: low (spec wording).

- **D4 — pre-validate vs create discriminated unions are NOT identical.**
  Brief §2 asks whether step-4 uses "the same discriminated shape as pre-validate".
  Answer: they **share** the `validation` and `error` tags (same
  `errors: ParsedProductValidationErrors`) but **differ** on the happy path:
  pre-validate → `{type:'clean'}` (no payload); create → `{type:'success'; data:
  {id, name}}`. Not a spec error — just the precise shape mobile must reproduce.

- **Note N1 — step-4 has no rate-limit backoff/disable.** Step 2 disables Next for
  a fixed 5s on a `__system` entry (`RATE_LIMIT_BACKOFF_MS = 5000`); step 4 has no
  equivalent — the 429 message just renders as the last `<li>` and both buttons
  stay live. Mobile decides parity; the spec's rate-limit-UX note (§Platform
  adoption Part B) is written about the step-2/`BasicInfoProductDialog` surface only.

- **Note N2 — step-4 generic-error copy keys.** The non-validation failure message
  uses `DIALOG.new.product.success.failed.1` and `.2` (two lines), keys the spec
  doesn't name. Recorded for completeness so mobile reuses the same keys.

### Part 4b adjacent observations (out of scope — not fixed)

- `productService.ts:59` — the D2 key mismatch is also an adjacent code-quality
  issue regardless of mobile; medium; not fixed (read-only audit, and it's the
  central divergence already flagged for Mastermind).

No other adjacent issues found in the files read.
