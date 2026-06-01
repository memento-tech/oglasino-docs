# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-30
**Task:** Brief A3 — wire the CREATE wizard's two server round-trips: the pre-validate gate at the step-2 → step-3 transition, and the step-4 submit-failure UX (translated error list + step-routing). Plus port web's `productStepMapping` to RN. CREATE wizard only (EDIT is A4).

## Implemented

- **`productStepMapping.ts` ported to RN** (`src/lib/utils/`). Pure, zero-mock.
  `STEP_FOR_FIELD` (images→1, name/description/price/currency/topCategory/
  subCategory/finalCategory→2), `DEFAULT_FALLBACK_STEP = 3`, `stepForField`
  (exact match; `filters`/`filter`/`filter.*` → 3; else `undefined`),
  `resolveTargetStep(byField)` (first mappable key wins; empty/unmappable → 3).
  1-indexed, mirroring web's code.
- **`preValidateOutcome.ts`** (`src/lib/services/`) — a pure classifier
  `classifyPreValidate(name, description)` returning the discriminated outcome the
  brief invited (`clean` | `validation` | `error`). It runs `preValidateProduct`
  and **never throws**: a resolved 200-with-violations → `validation`; a thrown
  error whose body parses (429 `__system`, 400/422) → `validation`; a thrown error
  with no parseable body (transport/5xx) → `error`. Extracted as a pure module so
  the branch logic is unit-testable in the repo's node-env vitest (no RN render
  infra exists — see "For Mastermind").
- **Pre-validate gate in `BasicInfoProductDialog`.** Step-2 "Next" now: structural
  validation first (A2's `validateProduct`); on a structural pass it calls
  `classifyPreValidate`. `clean` → `onNextStep()`; `validation` → writes the parsed
  `byField` name/description `translationKey`s into `productErrors` (rendered inline
  by A2's existing slots) and, if a `__system` entry is present, renders a centered
  message under the action bar AND starts the 5 s backoff; `error` → warning toast +
  advance.
- **"Next" button busy states from ONE in-flight flag.** `preValidating` drives
  both the `ActivityIndicator` spinner and the disable (they can't drift).
  `rateLimited` is a separate fixed-window disable (no spinner — nothing is
  running). `disabled = preValidating || rateLimited`. `RATE_LIMIT_BACKOFF_MS =
  5000` (named constant, matches web); after the window the `__system` entry is
  cleared and the disable lifts. The backoff timer is cleared on unmount.
- **Step-4 failure UX in `UploadedProductDialog`.** Auto-fire-on-mount preserved.
  Replaced the single boolean `uploadSuccess` with a `CreateOutcome` discriminated
  state. On the create call: `success` → existing URL/finish UI (unchanged);
  `validation` (thrown error parses to field/`__system` entries) → header + a
  bulleted list of `tError(translationKey)` messages with the `__system` entry
  rendered **last**, and two buttons ("Go back and fix" → `onJumpToStep(
  resolveTargetStep(byField))`, "Exit" → `onClose`); `error` (parses to nothing) →
  generic copy + the same two buttons, "Go back and fix" → `DEFAULT_FALLBACK_STEP`;
  `upload` (`UploadError`) → existing toast + existing single-back generic surface,
  **not** folded into the error-list UX (CANCELLED stays silent). No backoff/disable
  at step 4 (per the N1 decision).
- **Step-jump plumbing in `AddUpdateProductDialog`.** Added `jumpToStep(
  step1Indexed)` doing `setCurrentStep(Math.max(0, step1Indexed - 1))` (the
  subtract-1 happens exactly once, here) and threaded it into `UploadedProductDialog`
  as `onJumpToStep`. The dialog previously exposed only next/previous.

## The `__system` decision — chose option (A)

Read `parseServiceError.ts:69`: object-level (`field: null`) entries are **excluded
from `byField`** and kept only in `errors`. I took **option (A)**: scan `errors`
for the `field === null` entry for the `__system`/rate-limit render and the
"is there a system error" check; `resolveTargetStep` still consumes `byField`
(field-keyed), so a `__system`-only failure yields an empty `byField` →
`resolveTargetStep` returns the fallback step 3 — exactly web's behavior.
**`parseServiceError` is unchanged** (option B would have changed a Φ4 shared helper
`ReportDialog` also uses — larger blast radius, unjustified here).

## Files touched

- src/lib/utils/productStepMapping.ts (new, +57)
- src/lib/utils/productStepMapping.test.ts (new, +60)
- src/lib/services/preValidateOutcome.ts (new, +73)
- src/lib/services/preValidateOutcome.test.ts (new, +95)
- src/components/dialog/dialogs/product-creation/BasicInfoProductDialog.tsx (~133 changed)
- src/components/dialog/dialogs/product-creation/UploadedProductDialog.tsx (~125 changed)
- src/components/dialog/dialogs/product-creation/AddUpdateProductDialog.tsx (~16 changed)

(The EDIT screen `app/owner/dashboard/products/[productId].tsx` was **not** touched
— it is single-form, does not use the step mapping, and per the brief A3 touches it
only if it imports a shared helper, which it does not.)

## Tests

- Ran: `npx vitest run`
- Result: **295 passed, 0 failed** (19 files). Baseline after A2 was 277; +18 in two
  new files.
- `productStepMapping.test.ts` (10 tests): each field→step, all three filter forms
  → 3, unmappable → `undefined`, `resolveTargetStep` first-key-wins / empty-map →3 /
  unmappable →3.
- `preValidateOutcome.test.ts` (7 tests, `preValidateProduct` mocked): clean→advance,
  200-with-violations→validation (per-field, first-per-field), thrown-429-`__system`
  →validation with systemError, thrown-400-field-errors→validation,
  transport-no-body→error, 500-no-errors-array→error.
- `npx tsc --noEmit` — clean (exit 0).
- `npm run lint` — **0 errors, 73 warnings** — identical to the A2 baseline (0/73).
  No new violations: the dialog `react-hooks/exhaustive-deps` warnings are
  pre-existing (HEAD's `UploadedProductDialog` already had the
  `uploadProductInternal()` `useEffect([productData])`; `AddUpdateProductDialog`'s
  three already existed), and the one `import/first` in `preValidateOutcome.test.ts`
  matches the repo's established `vi.mock`-then-import test convention
  (`productService.test.ts` emits 3 of the same).
- `npx expo-doctor` — N/A, no dependency touched (the pure classifier was extracted
  specifically to avoid adding RN-render test deps).

## D1 re-upload-on-retry — verified

Confirmed by reading the container: `AddUpdateProductDialog` renders only
`steps[currentStep].component`, so "Go back and fix" (`jumpToStep`) moves
`currentStep` away from index 3 and **unmounts** `UploadedProductDialog`, discarding
its `outcome` state. Returning to step 4 **remounts** it with `outcome = null`, and
the `useEffect` re-fires `uploadProductInternal` → `uploadProduct(productData, {mode:
'create'})`, which rebuilds `imageKeys` fresh from the in-memory `productData.
imagesData`. The success guard (`if (outcome?.kind === 'success') return`) is null on
a fresh mount, so it proceeds. The failed session's `newKeys` were local to the prior
`uploadProduct` call and cleaned via `cleanupOrphanImages` (A1); nothing writes stale
keys back to `productData.imageKeys` (it stays `undefined` on create). No stale keys
are cached or reused. ✓

## Cleanup performed

- Removed the DEV-guarded `if (__DEV__) console.error(error)` from
  `UploadedProductDialog`'s catch (it sat in the block I restructured; the failure is
  now surfaced via the `outcome` UI and `productService` already logs via
  `logServiceError` before rethrow — no debug log needed).
- No commented-out code, no unused imports/variables introduced (tsc + lint clean).
- The duplicated inline pre-validate branching I first wrote in `BasicInfoProductDialog`
  was collapsed into the shared `classifyPreValidate` classifier before final — no
  dead duplicate left.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: **no change owed from A3.** Product-validation stays `web-stable`; the
  Expo backlog row flips only when mobile adoption (A1–A4) reaches `mobile-stable`.
  A3 wires CREATE server round-trips; EDIT server-error display (A4) is still open,
  so adoption is not complete. (Stated explicitly per the closure gate — no implicit
  config dependency.)
- issues.md: no change authored by me. The DIALOG seed-gap below is a backend-seed
  escalation candidate for Mastermind, not a config edit I make.

## Obsoleted by this session

- The single-button ("Back") generic failure surface in `UploadedProductDialog` is
  replaced by the two-button (Go back and fix / Exit) UX for the `validation` and
  `error` outcomes. The `UploadError` (`upload`) outcome keeps the old single-back
  generic block intentionally.
- Nothing deleted wholesale; no module obsoleted.

## Conventions check

- **Part 4 (cleanliness):** confirmed. 0 lint errors / no new warnings; tsc clean;
  295 tests green; the one removed console site cleaned in-session; no TODO/FIXME
  added.
- **Part 4a (simplicity):** see structured evidence in "For Mastermind". One named
  constant (`RATE_LIMIT_BACKOFF_MS`) — one value, no foreseeable second, a constant
  not config. The pure classifier extraction is earned complexity (testability +
  removes duplication between the inline branch and what would otherwise be untested).
- **Part 4b (adjacent observations):** one flagged in "For Mastermind".
- **Part 6 (translations):** confirmed — the only client-constructed keys are A2's
  `product.<field>.*` structural keys and the generic `ERRORS.unknown` fallback. All
  server-origin messages render the backend-sent `translationKey` (rate-limit
  included; never reconstructed as `product.system.rate_limited`). No keys invented;
  no SQL touched. DIALOG chrome keys: see the seed-gap flag.
- **Part 7 (error contract):** confirmed — server errors consumed via
  `parseServiceError` (`{errors, byField}`), rendered off `byField` + the object-level
  `errors` scan, routed off `byField` keys; `translationKey` rendered, codes never
  turned into messages.
- **Part 11 (trust boundaries):** N/A new — the wire narrowing is A1's; A3 only
  reads responses.

## Known gaps / TODOs

- No `// TODO`/`// FIXME` added this session. The pre-existing bare `// TODO` in
  `UploadedProductDialog`'s "finish" handler (success block) was LEFT untouched —
  assigned to Ω by the A2 brief, unrelated to this change.

## For Mastermind

- **Brief-vs-reality confirmations (brief asked to verify before coding — all green,
  no halt warranted):**
  1. **`parseServiceError` `field: null` / `byField`** — confirmed at
     `parseServiceError.ts:69`: `field: null` is **excluded** from `byField`, kept
     only in `errors`. This *differs from web* (which collapses to
     `byField['__system']`). Resolved via **option (A)** (scan `errors`; helper
     unchanged) — see "The `__system` decision" above.
  2. **`preValidateProduct` resolve/throw shape** — confirmed at
     `productService.ts:135`: resolves 200 with `res.data.errors ?? []`; **throws on
     any non-2xx** (incl. 429). So in the wizard the only non-throw path is a 200;
     429/400/422 arrive as throws and are routed to `validation` by parsing the
     thrown body — matching web's 429→validation routing. Captured in
     `classifyPreValidate`.
  3. **Step-control exposure** — `AddUpdateProductDialog` exposed only
     `handleNext`/`handlePrevious` to its step children (no jump). Added
     `jumpToStep(step1Indexed)` (subtract-1 once) and threaded it into
     `UploadedProductDialog` as `onJumpToStep`.

- **DIALOG UX seed-gap (the brief's flag-and-proceed rule applied — needs a backend
  seed decision):** the four contract chrome keys are **web frontend additions** that
  the web session `2026-05-13-oglasino-web-product-validation-3.md` (lines 88–91)
  listed as *"Suggested EN"* seed candidates — there is **no confirmation in the docs
  that the backend seeded them**, and mobile fetches DIALOG from the backend at
  runtime so I cannot verify resolution statically. Per the brief I used the nearest
  **confirmed-resolving** mobile keys (all already used in shipped mobile code) and
  flag for a seed follow-up:

  | Contract key (web) | Purpose | Mobile fallback used (resolves today) |
  |---|---|---|
  | `DIALOG.new.product.pre.validate.warning` | pre-validate transport warning toast | `ERRORS.unknown` |
  | `DIALOG.new.product.create.failed.header` | step-4 failure list header | `DIALOG.new.product.success.failed.1` |
  | `DIALOG.new.product.create.failed.fix` | "Go back and fix" button | `DIALOG.new.product.success.failed.back` |
  | `DIALOG.new.product.create.failed.exit` | "Exit" button | `DIALOG.button.close.label` |

  If backend confirms these four DIALOG keys ARE seeded, swapping each fallback for
  the contract key is a one-token change per site (the render structure is already in
  place). The error/field `translationKey`s and the rate-limit `system.rate_limited`
  key are ERRORS-namespace and confirmed seeded (spec: all 42 constants), so those
  render the real backend copy already — only the four DIALOG *chrome* keys use
  fallbacks.

- **Part 4a simplicity evidence (required):**
  - Added (earned): `classifyPreValidate` pure module (testable + removes the
    duplicate inline branch in two call paths); `productStepMapping` (the brief's
    required port); `RATE_LIMIT_BACKOFF_MS` (one-value constant); the `CreateOutcome`
    discriminated state replacing a boolean (the boolean could not express
    validation-vs-error-vs-upload).
  - Considered and rejected: (1) a full `BasicInfoProductDialog` render test via
    `@testing-library/react-native` — rejected; the repo's vitest is `environment:
    'node'` with no RN testing-library and zero `.test.tsx` precedent, so it would
    have meant adding deps the brief said not to expect and native-mock setup.
    Extracting `classifyPreValidate` gives the same branch coverage as a pure unit
    test. (2) Option (B) (mutating `parseServiceError`) — rejected per the `__system`
    decision (shared-helper blast radius).
  - Simplified: collapsed the first-draft inline pre-validate try/catch in the dialog
    into the shared classifier.

- **Adjacent observation (Part 4b):** `UploadedProductDialog`'s `uploadProduct`
  return type is `NewProductResponse | null`, but A1's `uploadProduct` always returns
  the resolved `createProduct`/`updateProduct` body or throws — it never resolves
  `null`. The `else { setOutcome({ kind: 'error' }) }` branch is therefore dead but
  typed; I kept it as a defensive default rather than narrow A1's service type (out
  of A3 scope per "no change to A1's service layer"). Low severity; flag for triage if
  the nullable return is ever tightened.

- **Config-file impact:** no config edits owed from A3 (stated per-file above). The
  DIALOG seed-gap is a backend-seed escalation candidate, not a config edit. No
  pending drafts at closure.
