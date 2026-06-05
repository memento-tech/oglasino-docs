# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-06-01
**Feature slug:** product-create-parity
**Task:** Phase 5 fix — bring the mobile product CREATE wizard to web parity on exactly two points: (1) region/city shown display-only from the user's profile instead of collected, and (2) the dialog title shown on every non-terminal step.

## Implemented

### Change 1 — Region/city: stop asking, show from profile (guarded)
- `BasicInfoProductDialog.tsx`: replaced the editable, `required` `CitySelector` (was sourced from `productData.regionAndCity` with a write-back `onChange`) with a **display-only** one sourced from the authenticated user's profile.
  - Added `import { useAuthStore } from '@/lib/store/authStore';` and a narrow selector `const userRegionAndCity = useAuthStore((s) => s.user?.regionAndCity);` (mirrors the Φ3 narrow-selector pattern and the precedent update-screen fix).
  - The picker now renders with `selectedRegionAndCity={userRegionAndCity}`, `onChange={() => {}}` (inert), `disabled={true}`; dropped `required` (it drove only red-border styling — `validateProduct` has no `regionAndCity` branch).
  - **Absent-case guard:** the whole slot is gated on `userRegionAndCity?.region && userRegionAndCity.city`, so when the user has no saved region/city (undefined/null, or missing region or city) it renders **nothing** rather than an empty/`choose…` picker. This is the brief's stronger guard — note the precedent update screen renders the disabled picker unconditionally and only guards a separate text label; here, per the brief, the entire slot is guarded.
- No wire change: `regionAndCity` was already excluded from the create payload by the allow-list (`toCreateWirePayload` / `CreateProductWireDTO`). Left untouched.

### Change 2 — Step title on every non-terminal step
- `AddUpdateProductDialog.tsx`: changed the title render condition from `currentStep === 0` to `currentStep !== steps.length - 1` — the exact predicate the progress bar already uses on the next line. Title now shows on steps 0/1/2, hidden on the terminal step 3. No new constant/helper introduced.

## Files touched

- `src/components/dialog/dialogs/product-creation/BasicInfoProductDialog.tsx` — +`useAuthStore` import, +`userRegionAndCity` selector (+comment), region/city block rewritten display-only + guarded.
- `src/components/dialog/dialogs/product-creation/AddUpdateProductDialog.tsx` — title render condition `currentStep === 0` → `currentStep !== steps.length - 1`.

## Decision: `NewProductRequestDTO.regionAndCity` field KEPT (per brief's reader-check)

The brief said to delete the `regionAndCity` field/init **only if** nothing else reads it. Grep across `src/`+`app/` found another reader: `PreviewProductDialog.tsx:110` reads `productDetails.regionAndCity?.city?.labelKey`, and its `productDetails` is typed `UpdateProductRequestDTO` (which `extends NewProductRequestDTO`). The preview dialog is opened from the **update** screen (`app/owner/dashboard/products/[productId].tsx:267`), not the create wizard — but it reads the shared type's field. The field is also declared **required** (non-optional) on `NewProductRequestDTO`.

Therefore:
- **Field kept** on `NewProductRequestDTO` (Preview reads it; deleting breaks the update-flow preview).
- **Create init kept** (`AddUpdateProductDialog.tsx:97`, `regionAndCity: undefined`). I first removed it, but `tsc` failed because the field is required on the type, so the init object must supply it. Restored. Within the create flow it is now write-once-undefined and read by nothing (the picker no longer writes it).
- The only dead wiring actually removed is the picker's `onChange={(regionAndCity) => onChange({ regionAndCity })}` write-back (replaced by the inert `() => {}`).

## Tests

- `npx tsc --noEmit` → exit 0 (clean).
- `npm run lint` → 0 errors, 82 warnings. Below the state.md Risk-Watch baseline (84/0); no new warnings — none of the 82 are in the two touched files (all pre-existing in unrelated files: `consentStorage.test`, `bootStore`, `viewTokens.test`, `PushNotificationsInit`, etc.).
- `npm test` (vitest) → 26 files, 334 tests passed, 0 failed.
- `npx expo-doctor` → not run; no dependency/native-config change this session.
- New tests added: none — UI-only render-condition changes; no new branch logic warrants a unit test (the existing suite has no render test for this dialog).

### Manual reasoning (on-device deferred — see Known gaps)
- **User WITH saved region/city:** `userRegionAndCity?.region && userRegionAndCity.city` is truthy → the disabled `CitySelector` renders showing the saved city label (green border, non-interactive — taps short-circuit via `disabled` at `CitySelector.tsx:69/127-128/138-139`). Uneditable, not collected.
- **User WITHOUT region/city:** guard is falsy → the slot renders nothing; the step does not crash and shows no empty picker.
- **Title:** `currentStep !== steps.length - 1` ⇒ true for steps 0/1/2, false for step 3 → title shows on 0/1/2, hidden on the terminal upload step. Verified against the identical predicate already governing the progress bar one line below.

## Cleanup performed

- Removed the dead picker write-back (`onChange` that wrote `regionAndCity` into wizard state then got discarded on the wire). No commented-out code, no unused imports left (`CitySelector` and `Text` both still used; `useAuthStore` newly used).
- No `console.log`, no `TODO`/`FIXME` added.

## Config-file impact

- **conventions.md:** no change.
- **decisions.md:** no change.
- **state.md:** no change required by me (Docs/QA owns the file). **Draft for Mastermind (see "For Mastermind"):** the two `issues.md` 2026-06-01 product-creation items (region/city defaulting; title-not-visible-past-step-1) are now addressed in code — they may be checked off / annotated by Docs/QA. This feature is not in the state.md Expo backlog table as a discrete row, so no backlog-row removal is owed.
- **issues.md:** no self-edit. The two relevant open items under the 2026-06-01 "Mobile on-device UI/UX findings (batch)" are the natural carriers; resolution annotations are Docs/QA's to write (drafted below).

### Closure gate
No implicit config-file dependency beyond the issues.md annotations drafted for Mastermind. No state.md backlog row exists for this slug, so none is owed for removal.

## Obsoleted by this session

- The collected-but-discarded region/city picker in the create wizard (user friction with no effect) is obsoleted — region/city is now display-only and never collected in create.

## Conventions check

- **Part 4 (cleanliness):** tsc/lint/test all green; no commented-out code, no unused imports/vars, no debug logging, no unmatched TODO/FIXME. Dead write-back removed in the same session.
- **Part 4a (simplicity):** structured evidence in "For Mastermind." Net change is small and removes UI/logic (a picker write-back) rather than adding it. Deliberately did NOT replicate the update screen's extra region/city `Text` label or a new COMMON_SYSTEM translator — the brief asked only for a display-only `CitySelector`, and the disabled picker already surfaces the saved city; adding the label would be unrequested UI plus a new translator import.
- **Part 4b (adjacent observations):** one noted — the create/update screens now display the user's region/city slightly differently (update screen pairs a `region/city` text line with the disabled picker and renders the picker even when absent; create renders only the disabled picker and hides the whole slot when absent). Not reconciled here (out of scope; update screen is a separate brief). Flagged for Mastermind.
- **Part 6 (translations):** N/A — no keys added or changed. `new.product.title` (already seeded) now simply renders on more steps.
- **Part 11 (trust boundary):** unchanged and confirmed — region/city still excluded from the create wire by allow-list; server derives location from the authenticated user. This change is UI-only.

## Known gaps / TODOs

- **On-device confirmation owed.** Per the brief, device verification is not part of this session — it rides the pending iOS+Android rebuild / Ψ pass (the same dependency carried by product-validation and the other `new-expo-dev` `verifying` items). Verify on a build with: (a) a user who HAS a saved region/city → step shows it disabled/uneditable; (b) a user with NONE → slot renders nothing, no crash; (c) title visible on steps 0/1/2, hidden on step 3.
- No TODO/FIXME left in code.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - **Added (earned complexity):** one narrow `useAuthStore` selector + a presence guard in `BasicInfoProductDialog.tsx`. Earned: it's the minimum needed to source the value from the profile and not crash when the profile lacks region/city (mobile has no pre-entry gate, unlike web).
  - **Considered and rejected:** (1) replicating the update screen's extra `region/city` `Text` line + a new COMMON_SYSTEM translator — rejected as unrequested UI/scope creep; the disabled picker already shows the saved city. (2) Deleting `NewProductRequestDTO.regionAndCity` and its create init — rejected because `PreviewProductDialog.tsx:110` reads the field via the shared `UpdateProductRequestDTO` type and the field is required; deletion would break the update-flow preview and fail `tsc`. (3) Adding a pre-entry "set your region/city first" gate — explicitly out of scope; this is a UI-only parity change.
  - **Simplified or removed:** removed the picker's write-back `onChange` (collected-then-discarded value); dropped the cosmetic `required` flag; title gate now reuses the existing progress-bar predicate rather than a step-0 special case.
- **issues.md annotation drafts (Docs/QA to apply — I do not write issues.md):**
  - 2026-06-01 batch item *"Product creation dialog — region/city is asked even though the user already has a region and city set"* → **Fixed (code) 2026-06-01** (`oglasino-expo` `new-expo-dev`, session `product-create-parity-1`). Create wizard now sources region/city display-only from `useAuthStore().user.regionAndCity`, disabled + inert, guarded to render nothing when absent (mirrors the update-screen fix `update-product-region-crash-2`). Not collected, not on the wire. On-device confirmation owed (rides the iOS+Android rebuild / Ψ).
  - 2026-06-01 batch item *"Product creation dialog — the dialog title is not visible on step 2 and higher"* → **Fixed (code) 2026-06-01** (same session). Title gate changed from `currentStep === 0` to `currentStep !== steps.length - 1`; shows on steps 0/1/2, hidden on terminal step 3. On-device confirmation owed.
- **Adjacent (Part 4b), not actioned:** create vs update screens display the saved region/city slightly differently (text-line-plus-picker-always vs picker-only-hidden-when-absent). A future pass could reconcile to one shared display-only region/city presenter. Update screen is a separate brief.
- **Config-file impact:** no edits to the four config files by me. No state.md Expo backlog row exists for this slug. Closure gate clean — only the two issues.md annotation drafts above are owed (Docs/QA's to write).
