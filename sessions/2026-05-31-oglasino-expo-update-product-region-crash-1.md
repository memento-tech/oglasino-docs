# Session — AUDIT: update-product screen crash (`regionAndCity` undefined)

**Repo:** oglasino-expo
**Branch:** new-expo-dev (no switch, no commit, no push)
**Date:** 2026-05-31
**Slug:** update-product-region-crash (order 1)
**Type:** read-only audit — no code changed, nothing staged. Findings only.

---

## Task (one sentence)

Trace why `productDetails.regionAndCity` is undefined at render in the update-product screen (`app/owner/dashboard/products/[productId].tsx:333`) and answer the brief's six questions with file:line evidence — read-only, no fix.

---

## What I did

Read-only inspection of:
- `app/owner/dashboard/products/[productId].tsx` (the screen + crash site)
- `src/lib/services/productsSearchService.ts` (data source / endpoint)
- `src/lib/types/product/NewProductRequestDTO.ts`, `UpdateProductRequestDTO.ts`, `src/lib/types/catalog/RegionAndCityDTO.ts` (types)

Produced findings doc at `.agent/audit-update-product-region-crash.md` answering Q1–6 with citations. No source files modified.

---

## Findings (condensed; full doc in audit-update-product-region-crash.md)

1. **Source:** `productDetails` state (`:49`, init undefined) ← `getDashboardProductDetails(productId)` (`:112`), set whole-object once at `:120`; re-seeded at `:181`. Render double-gated: `loading` early-return `:267-269` + `{productDetails && …}` `:291`. Region block (`:331-343`) **not** gated on `regionAndCity`.
2. **Crash site:** `:333-334` deref `productDetails.regionAndCity.region.labelKey` / `.city.labelKey` **raw** — no optional chaining, no guard.
3. **Type:** `regionAndCity: RegionAndCityDTO | undefined` (`NewProductRequestDTO.ts:16`) — explicitly optional; inner `region`/`city` are `KeyLabelPair | null` (`RegionAndCityDTO.ts:6-9`).
4. **Endpoint:** `GET /secure/products?productId=` (`productsSearchService.ts:36-38`), cast to `UpdateProductRequestDTO` which extends `NewProductRequestDTO` → inherits the optional field. No runtime shape validation; `res.data` set straight into state.
5. **Verdict: (a)** — `regionAndCity` was absent/null in the response payload for the crashing product; a data-shape exposure, NOT a render-before-hydration race. `productDetails` is only ever set as a complete object and render is double-gated, so no half-hydrated window exists. (b) ruled out.
6. **Adjacent:** `.city` line (`:334`) same exposure as `:333`; inner `region`/`city` null also crashes at `.labelKey`; `productDetails.imageKeys.map` at `:296` is the same unguarded-optional pattern (`imageKeys?: string[]`, `NewProductRequestDTO.ts:18`) and would crash earlier if absent.

---

## Brief vs reality

Nothing to challenge. The brief's reported stack trace matches the code exactly (crash at `:333`, raw deref of an optional field). One refinement, not a contradiction: the brief framed Q5 as (a) "legitimately absent" vs (b) "timing/hydration race." The code rules out (b) firmly (no partial-hydration path); whether the absence is *legitimate* is a Backend question this repo can't answer — captured in the audit's "For Mastermind".

---

## Verification

Read-only audit — no tsc/lint/test run (no code touched; running them would prove nothing about an unchanged tree). Findings verified by direct file:line reading.

---

## Cleanup performed

None needed — read-only session, no code changed.

---

## Obsoleted by this session

Nothing. No code or prior doc superseded.

---

## Conventions check

- **Part 4 (cleanliness):** N/A — no code changed; no TODO/FIXME/console.log introduced.
- **Part 4a (simplicity):** N/A — no implementation.
- **Part 4b (adjacent observations):** logged — `.city` line + inner-null exposure (`:334`, RegionAndCityDTO null members) and the `imageKeys.map` raw-optional deref (`:296`). All recorded in the audit's Q6 / "For Mastermind", none fixed (read-only brief).
- **Stack reminders:** confirmed the screen reuses the web endpoint `GET /secure/products`; no mobile-specific route involved.

---

## Config-file impact

No edit to any of the four config files. Read-only audit; adopts/retires no `state.md` Expo-backlog row. The bug is already tracked in `issues.md` (the source line); any update there is Docs/QA's to make. No closure-gate dependency surfaced — stated explicitly.

---

## For Mastermind

1. **Confirmed mobile render bug (in-scope, not fixed):** `[productId].tsx:333-334` deref `regionAndCity.region.labelKey` / `.city.labelKey` raw against a `RegionAndCityDTO | undefined` field whose `region`/`city` are `| null`. Three nullable hops, zero guards = the reported crash.
2. **Needs Backend answer before a fix is designed:** is `regionAndCity` legitimately absent/null on a valid product's `GET /secure/products` response, or a backend regression? Fix shape depends on it — graceful render-degrade (if legitimately optional) vs. backend contract fix (if it should always be present).
3. **Web parity check:** confirm whether web's update-product screen guards this same field (same endpoint). If web guards and mobile doesn't, this is mobile-only render hardening.
4. **Adjacent latent crash:** `imageKeys.map` at `:296` is the same unguarded-optional pattern — candidate to harden in the same pass.
5. Suggested issues.md follow-up wording (Docs/QA to write): "Mobile update-product render derefs optional `regionAndCity` (and inner null `region`/`city`) and optional `imageKeys` without guards; pending Backend confirmation on whether `regionAndCity` is legitimately optional on the details response."
