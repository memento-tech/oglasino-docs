# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-29
**Task:** Read-only Phase-2 audit of the current `oglasino-expo` product create/edit/validation surface (post-Φ4), inventoried across sections A–G against the frozen `product-validation.md` `## Platform adoption` contract.

## Implemented

Read-only audit; no code changed. Deliverable is `.agent/audit-expo-product-validation.md`, structured by the brief's sections A–G. Highlights:

- **A/B (surfaces):** create wizard survives as 4 steps but reordered to Images → BasicInfo → Meta → Upload; final step auto-fires upload+POST on mount (no submit button). Edit screen did not move — still `app/owner/dashboard/products/[productId].tsx`; loads via `GET /secure/products?productId=`, detects changes with `deepEqualTest` + on-device `isMassiveChange`, submits through the same service path as create.
- **C (load-bearing):** both create and edit POST to the OLD `/secure/product/addUpdate` (`productService.ts:93`); the contract's three routes (`/api/secure/products/create`, `/update`, `/pre-validate`) are nowhere in the repo and no pre-validate exists. Service does throw (Φ4); `parseServiceError` returns `{ errors, byField }` and never throws — but no product screen consumes it. All three carry-forward callers (`user.tsx` `updateUser`, `BasicInfoProductDialog` `getOpenAiSuggestionForProduct`, `FollowUserButton` `markFollowUser`) confirmed uncaught/unsurfaced.
- **D (DTOs):** no wire-payload narrowing; create leaks `free`+`regionAndCity`, update leaks all nine forbidden fields incl. `oldName`/`oldDescription`. `productUpdateNameValidator.isMassiveChange` still exists with one live caller (the edit screen).
- **E (i18n):** surface is client-only on `product.internal.*` under frozen `VALIDATION`; 0 of 10 sampled `ERRORS` spec keys resolve.
- **F (bugs):** B13 (wrong-field error key, `productValidator.ts:189,191`), B14 (hardcoded 62-word `BANNED_WORDS`, ignores `regexData.bannedWords`), B16 (3 `console.error` sites) all present.
- **G (trust):** CREATE clean; UPDATE is a Part 11 violation (client-supplied `oldName`/`oldDescription` "before" values + immutable fields on the wire).

## Files touched

- `.agent/audit-expo-product-validation.md` (new — the audit deliverable)
- `.agent/2026-05-29-oglasino-expo-product-validation-1.md` (new — this summary)
- `.agent/last-session.md` (overwritten — twin of this summary)

No source files modified (read-only brief).

## Tests

- Not run. Read-only audit; no code change to lint, type-check, or test.

## Cleanup performed

- none needed (no code changed).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change required by this session. (The Expo backlog row for Product validation is already `not-started`; it flips only when mobile adoption reaches `mobile-stable`, which is downstream of the rebuild this audit precedes. Drafted note for Mastermind below, but no edit owed now.)
- issues.md: no change. Bucket-1 bugs (B13/B14/B16) and adjacent observations are surfaced to Mastermind below for triage into `issues.md` by Docs/QA if Mastermind so decides; this session does not author entries.

## Obsoleted by this session

- The prior mobile audit `.agent/audit-expo-readiness-product-validation.md` (dated 2026-05-23, pre-Φ) is now superseded for the product-validation feature by `.agent/audit-expo-product-validation.md`. Not deleted — archival/retirement is Docs/QA's call (cannot delete cross-repo state, and it may carry readiness-program context beyond this feature). Flagged for Mastermind.

## Conventions check

- Part 4 (cleanliness): confirmed — no code added; the `console.error`/`// TODO`/dead-state items found are pre-existing and flagged (Part 4b), not introduced.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): flagged in "For Mastermind".
- Part 6 (translations): confirmed analyzed — surface is on `product.internal.*`/`VALIDATION`, not the frozen `ERRORS` contract (Section E).
- Part 7 (error contract): confirmed analyzed — `parseServiceError` matches the wire shape but is unused on the product surface (Section C).
- Part 11 (trust boundaries): confirmed analyzed — CREATE clean, UPDATE violation (Section G).

## Known gaps / TODOs

- The single biggest open question — **is `/secure/product/addUpdate` still live on the backend?** — cannot be answered from this repo. It gates whether mobile create/edit is merely non-conformant or outright broken today. Backend chat must confirm.
- `BANNED_WORDS` exact count reported as 62 (with `heroin` duplicated lines 24/53); the brief's "~70" was approximate. Not material to the rebuild (the list is deleted in favor of backend `regexData`).

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **BLOCKER to resolve before the rebuild brief is written:** mobile create AND edit POST to the OLD `/secure/product/addUpdate` (`src/lib/services/productService.ts:93`); the frozen contract's `/api/secure/products/create`, `/update`, `/pre-validate` are not in the repo, and no pre-validate round-trip exists. **High severity** — if the backend retired `addUpdate`, mobile create/edit is broken in production today, independent of any conformance work. Needs a backend route-table confirmation (backend chat) before scoping. I did not fix this — out of scope (read-only).

- **Scope-shaping findings (each would change the rebuild brief):**
  - **Trust-boundary violation on UPDATE (high):** `oldName`/`oldDescription` (client "before" values) + `productState`/`moderationState`/categories/`regionAndCity` all ride the update wire because the full `UpdateProductRequestDTO` from `getDashboardProductDetails` is blind-spread into the POST with no `toWirePayload`. The rebuild must add an explicit allow-list narrowing in `productService` (or stop typing the dashboard fetch as the wire DTO). `app/owner/dashboard/products/[productId].tsx`, `src/lib/services/productService.ts`.
  - **No pre-validate + no structural category/price/MIME checks (medium):** mobile has no server pre-validate call and `AddUpdateProductErrors` has no price/category slot, so `product.category.required`/`product.price.required` have no emission site. The rebuild adds the structural→pre-validate→submit sequence, not just a key swap.
  - **Translation migration is real work, not a rename (medium):** surface is 100% client-side on `product.internal.*`/`VALIDATION`; 0/10 contract `ERRORS` keys resolve; `parseServiceError` is unused on the product surface (only `ReportDialog` uses it). Adoption = re-point keys+namespace AND wire server-driven error rendering.
  - **On-device `isMassiveChange` (medium):** `productUpdateNameValidator.ts` is still called from the edit screen (`:113`). The contract makes `NAME_MASSIVE_CHANGE` a 422 server decision — the rebuild should delete the on-device check.

- **Bucket-1 bugs (the rebuild deletes these; logged for `issues.md` triage):**
  - **B13 (medium):** description link/contact branches write `errors.nameErrorKey` instead of `descriptionErrorKey` — `src/lib/validators/productValidator.ts:189,191`. Wrong field shows the error.
  - **B14 (medium):** hardcoded `BANNED_WORDS` (62 entries) at `productValidator.ts:7-70`; `containsBannedWords` accepts `regexData.bannedWords` but ignores it, so the backend-configured list is discarded.
  - **B16 (low):** 3 `console.error` sites — `UploadedProductDialog.tsx:83` (DEV-guarded), `:104` (unguarded), `app/owner/dashboard/products/[productId].tsx:83` (unguarded).

- **Adjacent observations (Part 4b, out of scope, not fixed):**
  - Dead `errorMessage` state in `MetaDataProductDialog.tsx` (never set) — step 3 surfaces no validation feedback. Low.
  - Untracked bare `// TODO` at `UploadedProductDialog.tsx:161`. Low.
  - Misspelled key literal `product.internal.name.exessive.change` (`[productId].tsx:116`, "exessive"). Low.
  - `productService` methods annotate `Promise<…|null>`/`Promise<boolean>` but rethrow on error, so the `null`/`false` return is unreachable on the error path — annotations are misleading. Low.
  - Edit screen state typed `NewProductRequestDTO` while holding the wider `UpdateProductRequestDTO` at runtime — hides the leaked fields from tsc. Medium (it is the mechanism behind the Part 11 violation).

- **Config-file impact (closure gate):** none required this session (read-only). Drafted notes above are for Mastermind triage, not edits owed by this session. Suggested (Mastermind's call, not drafted as final text): when the rebuild lands and reaches `mobile-stable`, Docs/QA updates the Product validation Expo-backlog row; and the stale `.agent/audit-expo-readiness-product-validation.md` may be retired in favor of this audit.

- **No challenge to a brief instruction:** the brief is read-only and internally consistent with the code; the only "brief vs reality" item is informational (Section A — the create-wizard step order changed from the brief's stated pre-Φ sequence; the brief explicitly asked me to confirm whether it survived, so this is the requested finding, not a contradiction).
