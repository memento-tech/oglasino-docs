# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-06-01
**Task:** GA4 Mobile v1 — Brief 3 (last code brief): the two remaining events. (1) `exception` — build the RN catch-all surface from scratch (route `ErrorBoundary` + global `ErrorUtils` handler + promise-rejection tracking) and fire via the Brief-1 `trackError`. (2) `form_submit_failed` — fire the real backend `code` on the structured product create/update paths and a synthetic code on the client-side register/login/profile paths. No device verification (Igor's rebuild + DebugView smoke after this brief); prove logic with unit tests.

## Implemented

### `exception`

- **Route-level error boundary** — `app/_layout.tsx` now exports a named `ErrorBoundary` (expo-router's RN analog of `app/error.tsx`; confirmed `app/_layout.tsx` is the root layout / top-level home). On catch it fires `trackError(error, { boundary: 'route' })` **in a `useEffect` keyed on `error`** (not during render → no double-fire, no render side effect), then renders a minimal fallback (centered message + retry `Pressable`). The fallback uses **plain RN primitives + hardcoded Serbian copy, no themed components and no i18n** — the same last-resort discipline `HardUpdateScreen` already uses, so the fallback can't itself throw via the UI/translation layer that produced the error.
- **Global JS handler + promise-rejection tracking** — `src/lib/analytics/globalErrorHandler.ts` → `installGlobalErrorHandlers()`. Reads `globalThis.ErrorUtils`, captures the previous handler, installs one that fires `trackError(err, { boundary: 'global' })` **then chains to the previous handler** (never replace-and-swallow — a handler that doesn't call through hides crashes from the OS/crash reporting). Also enables RN's bundled `promise/setimmediate/rejection-tracking` with `onUnhandled → trackError(err, { boundary: 'global' })`. Returns an uninstall fn (restores prior handler, disables tracking). Registered once via `src/components/init/GlobalErrorHandlerInit.tsx` (headless), mounted as an **ungated sibling in `app/_layout.tsx`** next to `ThemeInit`/`ConsentInit` — early, always-on, and (deliberately) NOT inside the ready-gated `AppInit`, which unmounts on the `ready ↔ updating` transition and would drop the crash catch-all (see Brief vs reality #2).
- **Payload** — uses Brief-1 `trackError` unchanged: `error_name` = `error.name`, `error_message` = `error.message.slice(0,200)`, `boundary`, **no stack trace**. Confirmed `trackError` already threads `context.boundary` into the `exception` params and caps the message — **no extension needed** (the brief flagged this to confirm).

### `form_submit_failed`

- **Structured path (backend `{field, code, translationKey}` via `parseServiceError`)** — `src/lib/analytics/formEvents.ts` → `trackStructuredFormSubmitFailed(err, formName)`: parses the caught error, fires the **first wire error** (`errors[0].code` + `.field`), no-ops when there are no structured entries (network/generic/UploadError → not a validation failure). Wired:
  - **Product create** — `UploadedProductDialog.tsx`, in the persistence-`validation` branch (`error_code` = first code, `form_name: 'product_create'`).
  - **Product update** — `app/owner/dashboard/products/[productId].tsx`, in the `classifyUpdateError` `validation` branch (`form_name: 'product_update'`). **Confirmed** this surface exposes the structured shape (`classifyUpdateError` → `parseServiceError`).
- **Client-side path (synthetic codes)** — register/login use imperative `validateForm` (no Zod on mobile), profile-update uses imperative `!email`/`!displayName` guards. Defined the synthetic set from the **actual branches**: `DISPLAY_NAME_EMPTY · EMAIL_EMPTY · EMAIL_FORMAT · PASSWORD_EMPTY · PASSWORD_INVALID` (`PASSWORD_INVALID`, not `PASSWORD_TOO_SHORT`, because the single `validatePassword` branch covers length AND letter/number — one branch, one code).
  - **Register** (`RegisterDialog.tsx`) + **Login** (`LoginDialog.tsx`): extracted a pure `src/lib/validators/authFormValidation.ts` (`registerValidationFailures` / `loginValidationFailures`) as the **single source of truth** for both the inline field messages and the synthetic code — `validateForm` now maps the ordered failures to messages and returns them; the submit fires `failures[0]`'s code (first-error rule, matching the structured path). `form_name: 'register' | 'login'`.
  - **Profile** (`app/owner/dashboard/user.tsx`): fires inline at the existing `!email` / `!displayName` early-returns (`EMAIL_EMPTY` / `DISPLAY_NAME_EMPTY`, `form_name: 'profile_update'`). The first-error rule is naturally satisfied by the sequential early-returns.
- **Single typed entry point** — both paths route through `trackFormSubmitFailed({ form_name, error_code, field })` so the param keys never drift across the five surfaces. `field` is included (per the spec catalog: field name or `null` for object-level violations).
- **Scope held:** backend auth failures that collapse to a single `mapAuthError` string fire **nothing** (we only fire on the client `validateForm` rejection — valid input → empty failures → no event); profile's generic `updateUser` catch (`tError('unknown')`) and the product client-side pre-validation (`validateProductInternal`/`validateProduct`) are **not** `form_submit_failed` surfaces.

## Files touched

- `src/lib/analytics/formEvents.ts` (new) — `trackFormSubmitFailed` + `trackStructuredFormSubmitFailed`.
- `src/lib/analytics/globalErrorHandler.ts` (new) — `installGlobalErrorHandlers` (ErrorUtils chain + rejection tracking).
- `src/lib/validators/authFormValidation.ts` (new) — pure ordered register/login validation failures + synthetic codes.
- `src/components/init/GlobalErrorHandlerInit.tsx` (new) — headless once-registration.
- `src/types/promise-rejection-tracking.d.ts` (new) — ambient types for RN's bundled rejection tracker (keeps `tsc` clean).
- `src/lib/analytics/formEvents.test.ts` (new) — first-error rule, object-level `field: null`, no-op on non-structured.
- `src/lib/analytics/globalErrorHandler.test.ts` (new) — fires `boundary: 'global'` then chains; rejection `onUnhandled` fires; cleanup restores + disables.
- `src/lib/validators/authFormValidation.test.ts` (new) — synthetic codes, branch order / first-error, valid→empty (the mapAuthError path fires nothing).
- `app/_layout.tsx` — `ErrorBoundary` export; mount `<GlobalErrorHandlerInit />`.
- `src/components/dialog/dialogs/product-creation/UploadedProductDialog.tsx` — fire structured `product_create`.
- `app/owner/dashboard/products/[productId].tsx` — fire structured `product_update`.
- `app/owner/dashboard/user.tsx` — fire synthetic `profile_update`.
- `src/components/dialog/dialogs/RegisterDialog.tsx` — `validateForm` via pure validator; fire synthetic `register`.
- `src/components/dialog/dialogs/LoginDialog.tsx` — same for `login`; removed now-unused `setError` helper.

## Tests

- `npx tsc --noEmit`: **clean** (exit 0).
- `npx vitest run`: **374/374 green** (32 files), including +14 new specs (formEvents 4, globalErrorHandler 4, authFormValidation 6).
- `npx expo lint`: **no new errors.** My touched files add **2 warnings, both `import/first` in the two new mock-before-import test files** — the repo's established vitest pattern (explicitly allowed by the DoD). All other warnings on my touched files (`exhaustive-deps` on `[productId].tsx`/`LoginDialog`/`RegisterDialog`/`UploadedProductDialog`) are **pre-existing** — on effects I did not modify. The single lint `error` (`react/jsx-key` in `DashboardSidebar.tsx`) is **pre-existing** in the working tree (that file is `M` from earlier uncommitted work; untouched by me). Source (non-test) files I added emit zero warnings.
- `npx expo-doctor`: not run — no dependency changes this brief (Brief 1 installed the SDK; this brief is JS-only). The `promise/setimmediate/rejection-tracking` module is already bundled with `react-native`; no new package.

## Brief vs reality

1. **Profile update does NOT expose the structured `parseServiceError` shape — it's a client-side surface.**
   - Brief says (Task 2a): profile/settings update goes through the Φ4 structured `{field, code, translationKey}` contract; "confirm each exposes the structured shape; if one doesn't, treat it like the client-side shape below and say so."
   - Code says: `app/owner/dashboard/user.tsx:saveChanges` validates client-side (`!email` → `tError('email.required')`, `!displayName` → `tError('username.required')`), and the `updateUser` failure `catch` sets a single generic `tError('unknown')` string — it never calls `parseServiceError`. No `{field, code}` reaches this surface.
   - Why it matters: firing a backend `code` here is impossible; the surface only has client-side emptiness guards.
   - Resolution: **as the brief instructed for the not-exposed case** — treated profile_update as the client-side synthetic path (`EMAIL_EMPTY` / `DISPLAY_NAME_EMPTY`), and did **not** fire on the generic `updateUser` catch (mirrors the mapAuthError "single-string backend failure isn't a form-validation failure" rule). Product **update**, by contrast, *does* expose the structured shape (`classifyUpdateError` → `parseServiceError`) and was wired structured as the brief specified. Not a blocking challenge (the brief pre-authorized this fallback); flagged per its instruction.

2. **The global crash handler belongs at the ungated layout root, not inside `AppInit`.**
   - Brief says: "mount it alongside the other init (per the audit, `AppInit` is the headless init slot; place it where it runs once and early)."
   - Code says: `AppInit` is rendered `{bootStatus === 'ready' && <AppInit />}` (`app/_layout.tsx`) — it **unmounts** whenever boot leaves `ready` (e.g. the `updating` freshness transient), which would tear down a crash catch-all registered inside it and reinstall it on return. The ungated init siblings (`ThemeInit`, `ConsentInit`, …) render at every boot state.
   - Why it matters: a crash catch-all must persist across boot states, especially the transitions where things are most likely to break.
   - Resolution: mounted `<GlobalErrorHandlerInit />` as an **ungated sibling at the layout root** (alongside `ThemeInit`/`ConsentInit`) rather than in `AppInit`. This satisfies "alongside the other init / once and early" while honoring the persistence requirement and the existing `registerAuthInterceptors()`-at-`_layout` precedent for early global side effects. (Minor deviation from the literal "AppInit" wording; flagged for transparency, not a contract break.)

3. **`trackError` needed no change.** The brief asked to confirm `trackError` forwards `context.boundary` and caps the message. It already does both (`src/lib/analytics/track.ts`: `boundary: context.boundary`, `error.message.slice(0,200)`, no stack field). No extension made.

## Cleanup performed

- Removed the now-unused `setError` helper from `LoginDialog.tsx` (its only caller was the refactored `validateForm`; `handleChange` uses `setErrors` directly). Removed the now-unused `validateEmail`/`validatePassword` imports from both auth dialogs (the pure validator owns those predicates now). No commented-out code, no debug logging, no `TODO`/`FIXME` left.

## Config-file impact

- **conventions.md:** no change.
- **decisions.md:** no change owed. The synthetic-code definition and the route-boundary/global-handler design are this brief's implementation of decisions already set by the spec; the closure brief carries the feature's decisions entry.
- **state.md:** this is the **last code brief** — after it lands, mobile GA4 is code-complete pending Igor's native rebuild + DebugView smoke. The Expo-backlog row / Platform-adoption-mobile flip to "code-complete-pending-Ψ" is explicitly the **closure brief's** job (per this brief's "Note for the closure brief"); **no `state.md` edit made by me.** Draft pointer in "For Mastermind."
- **issues.md:** no new defect entries. (The pre-existing `react/jsx-key` error in `DashboardSidebar.tsx` is someone else's in-flight working-tree change, not a logged bug from this session.)

## Obsoleted by this session

- Nothing. (No prior code removed beyond the dead-helper cleanup noted above; the Brief-1 `trackError` gains its first call sites here, as designed.)

## Conventions check

- **Part 4 (cleanliness):** tsc/test green; lint adds only the allowed test `import/first` pattern; unused helper + imports removed; no dead/commented code, no debug logging.
- **Part 4a (simplicity):** see three-category evidence in "For Mastermind."
- **Part 4b (adjacent observations):** the pre-existing `react/jsx-key` error and the `exhaustive-deps` warnings in the touched form files are noted (not mine, out of scope — I did not widen scope to "fix" them).
- **Part 6 (translations):** the route-boundary fallback intentionally uses hardcoded Serbian copy (last-resort screen, `HardUpdateScreen` precedent) rather than i18n; all backend `form_submit_failed` codes render via existing `translationKey` lookups, untouched. No new translation keys.
- **Part 7 (error contract):** structured `form_submit_failed` reports the wire `code` straight off `parseServiceError` (`ServiceFieldError.code`); no code→message coercion.
- **Part 11 (trust boundary):** **confirmed N/A risk.** Every payload is write-only GA4 telemetry via `track`/`trackError` → `logEvent`; no `form_submit_failed`/`exception` field is read by the oglasino backend or feeds a server-side decision. PII held: `exception` carries `error_name`/`error_message` (200-cap) + `boundary`, **no stack trace**; `form_submit_failed` carries `form_name`/`error_code`/`field` only — no free-text, no user input values.

## Known gaps / TODOs

- **Hermes promise-rejection caveat (documented in code + here, no TODO left):** RN's default engine is Hermes, whose native `Promise` has its own rejection tracker; the bundled `promise/setimmediate/rejection-tracking` we enable fully attaches on the polyfill path and is **best-effort on Hermes**. Escalated (fatal) rejections still reach the `ErrorUtils` global handler. The brief's testable requirement (ErrorUtils chaining) is met and unit-proven; full rejection coverage is part of Igor's on-device DebugView smoke.
- No native config touched (Brief 1's rebuild checklist still stands). Nothing fires until Igor's rebuild.

## For Mastermind

- **Closure brief (next, docs-only) — drafts:**
  - Flip `state.md` Platform-adoption (Mobile) to **code-complete-pending-Ψ** (native rebuild + DebugView smoke); the Expo-backlog GA4 row likewise should reflect code-complete, not yet adopted. No edit made by me.
  - Risk Watch should carry the **`firebase.json` `google_analytics_automatic_screen_reporting_enabled: false`** item (from Brief 1) — without it the native SDK double-counts `screen_view` against our custom `page_view`.
  - decisions entry summarizing mobile GA4 v1 (13 events mirrored, ATT+consent gating, route+global `exception` surfaces built RN-native, synthetic auth/profile codes).
- **Synthetic `form_submit_failed.error_code` set defined this brief (document inline + here):** `DISPLAY_NAME_EMPTY`, `EMAIL_EMPTY`, `EMAIL_FORMAT`, `PASSWORD_EMPTY`, `PASSWORD_INVALID` — one per imperative `validateForm` branch (register/login) and the profile `!email`/`!displayName` guards. These join the structured Part-7 codes (real backend `code`) on the product create/update paths in the GA4 `error_code` custom dimension.
- **Confirmation results (brief asked):** product **update** exposes the structured `parseServiceError` shape (wired structured); **profile** does **not** (wired synthetic — see Brief vs reality #1).
- **Part 4a simplicity evidence (required):**
  - *Added (earned complexity):* (a) `authFormValidation.ts` pure module — earned because it makes the synthetic-code/first-error rule unit-testable AND removes duplication (it's the single source for both inline messages and the analytics code, so the two can't drift). The generic `AuthValidationFailure<F>` is the minimal way to give the login dialog precise `email|password` field types so its `next` map type-checks. (b) `globalErrorHandler.ts` as a separate pure-ish installer (returns uninstall) — earned so the chain/cleanup logic is testable without a device and the headless component stays a one-liner. (c) the `promise-rejection-tracking.d.ts` ambient — minimal, the only way to keep `tsc` clean for the untyped RN-bundled module without `@ts-expect-error`.
  - *Considered and rejected:* (a) extending `UpdateSubmitOutcome` / the create outcome to carry the first error code — rejected; it would thread an analytics concern through shared UI-state types, and the mapped field errors there have already lost their `code`. Re-parsing via one uniform `trackStructuredFormSubmitFailed(err, …)` (one extra cheap pure `parseServiceError` call) keeps the analytics logic in one tested place. (b) firing `exception` during the boundary's render — rejected for a `useEffect` (no render side effects, no double-fire). (c) registering the global handler in `AppInit` — rejected for the ungated layout root (Brief vs reality #2). (d) i18n in the route fallback — rejected for hardcoded copy (the layer that errored may be i18n itself; `HardUpdateScreen` precedent).
  - *Simplified or removed:* `LoginDialog`'s `validateForm` collapsed from ~24 imperative lines to a map over the pure validator; dead `setError` + unused predicate imports removed from both dialogs.
