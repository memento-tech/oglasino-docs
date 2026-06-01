# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-30
**Task:** Exhaustive, source-traced READ-ONLY audit of mobile's product CREATE + UPDATE behavior laid against the web reference (`oglasino-web/.agent/audit-create-update-flow.md`), producing a per-behavior delta table and a prioritized parity-gap list. reCAPTCHA and analytics are explicitly out of parity scope.

## Implemented

This was a read-only audit — no product code changed. Output is the audit document
`.agent/audit-create-update-flow.md` (Parts 1–6 + delta table + For Mastermind),
re-derived from current mobile code on `new-expo-dev` and compared behavior-by-behavior
to the frozen web reference.

- Mapped both flows end to end: CREATE = a 4-step wizard dialog (`AddUpdateProductDialog.tsx`),
  UPDATE = a screen route (`app/owner/dashboard/products/[productId].tsx`) — the same
  structural split as web.
- Confirmed the A1–A5 RN rebuild landed at parity on the contract surface: pre-validate
  gate + 5s backoff (B6), step-4 auto-submit + create POST with **identical wire fields**
  to web (B9), the narrowed 7-field `toUpdateWirePayload` that strips
  oldName/oldDescription/state/moderation/category/region (B21), update success
  reseed-from-GET (B22), and the validation/transport/429/upload outcomes for both flows.
- Pinned the one real user-facing divergence: the **product-URL helpers** disagree with
  web and with each other (three distinct domain forms, none matching web's canonical
  `https://oglasino.com/${locale}`).
- Found two places where mobile is *better* than web: it renders real "Copied!" feedback
  (web's `copied` state is dead), and it adds an `otherErrors` catch-all so unmapped
  server field-errors aren't silently dropped on update.

### Method note (workflow)

Per the session request, the audit ran as a multi-agent workflow: 6 parallel source-tracers
(entry points / create wizard / create submit / update flow / shared mechanics / dedicated
product-URL deep-dive) + 6 concurrent adversarial verifiers re-opening files to confirm or
refute the known-divergence claims, then a synthesis pass that wrote the audit doc. The
adversarial pass earned its keep: it **corrected** the create-payload finding from an
initial DIVERGES to MATCHES (create wire fields are identical to web) and independently
**confirmed** the product-URL divergence, the 7-field update narrowing, the reseed-from-GET,
and the scroll-to-error mechanism. 13 agents total.

## Files touched

- `.agent/audit-create-update-flow.md` (new audit deliverable — not product code)
- `.agent/2026-05-30-oglasino-expo-product-validation-8.md` (this summary)
- `.agent/last-session.md` (exact copy of this summary)

No source, test, or config files were modified. (`brief.md` and `last-session.md` were the
only pre-existing `.agent/` files in scope; the brief was read, not written.)

## Tests

- Not run — read-only audit, no code changed. `npm run lint` / `npx tsc --noEmit` / `npm test`
  not applicable (no touched source paths).

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change. **Note:** this audit is the read-only phase feeding the
  `product-validation` mobile create/update parity fix; it does not itself flip any status
  or clear any Expo-backlog row. If Mastermind opens the parity-fix brief off this audit,
  the status/backlog edit belongs to that work, not this session.
- issues.md: no change. (The product-URL divergence and the seeding/domain questions are
  raised in "For Mastermind" below for triage; per the hard rules I do not write issues.md.)

## Obsoleted by this session

- Nothing deleted. The prior `.agent/audit-expo-product-validation.md` (the original
  Phase-2 validation audit) is **not** obsoleted — it covers the validation contract broadly.
  This new `audit-create-update-flow.md` supersedes it only for the *create/update behavioral
  parity* slice (detail-for-detail vs web), and stands alongside it. Left in place
  deliberately; not dead code.

## Conventions check

- Part 4 (cleanliness): confirmed — no code touched; no debug logging, dead code, or TODOs added.
- Part 4a (simplicity): N/A — no abstractions or config introduced (read-only audit). See
  "For Mastermind" structured evidence ("nothing in this category" × 3).
- Part 4b (adjacent observations): flagged in "For Mastermind" (product-URL divergence +
  translation-seeding question).
- Part 6 (translations): N/A this session — no keys added. Mobile-emitted keys that may not
  be seeded are listed as an Undetermined item for Backend to confirm.
- Part 7 (error contract): confirmed — both flows parse the `{errors:[{field,code,translationKey}]}`
  envelope via the shared helper and render codes through translationKey lookups, matching web.
- Part 11 (trust boundaries): confirmed — `toUpdateWirePayload` sends no client-supplied
  `oldName`/`oldDescription` (B21), matching web's server-as-trust-boundary narrowing.

## Known gaps / TODOs

- None added to the codebase. Five open questions are recorded as Undetermined in the audit
  doc and surfaced below — they need a product/backend answer, not mobile code, and are out
  of scope for this read-only session.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

### The one real parity gap to fix (user-facing divergence)

- **Product-URL divergence — HIGH.** Mobile has THREE distinct public product-URL domain
  forms, none matching web's canonical `https://oglasino.com/${locale}/product/{id}/{slug}`:
  1. `getNormalizedProductUrl` (`src/lib/utils/utils.ts:114`) → `https://www.oglasino.rs`
     (**www, .rs, no locale segment**), used for the create-success copy/share string
     (`UploadedProductDialog.tsx:109`).
  2. `ShareProductButton` (`src/components/product/ShareProductButton.tsx:29`) →
     hardcoded `https://www.oglasino.com` (**www, .com**) **and embeds the RAW,
     un-normalized `productName` as the slug** (no `normalizeProductName`). Call sites:
     `ProductFunctions.tsx:47-51`, `DashboardProductFunctionsDialog.tsx:139-144`,
     `ProductUserDetails.tsx:199-205`.
  3. Web canonical (reference): `https://oglasino.com/${locale}` with a normalized slug.
  - In-app navigation (the `withPrefix=false` relative path) is unaffected — this is purely
    the **copy/share** strings. The create-success copy string is the public web URL form
    (helper 1); the share button is helper 2. Recommended fix: unify both helpers to web's
    canonical form and route the share slug through `normalizeProductName`. Blast radius =
    the 4 call sites above plus the create-success screen.

### Already at parity (no action)

- Entry points (B1/B2), wizard structure (B3), step-1 images (B4), step-2 basics + validation
  keys (B5), pre-validate gate + 5s backoff (B6), step-3 filters (B7), step-4 auto-submit +
  create POST + **identical create wire fields** (B9, corrected to MATCHES by adversarial
  verify), create validation/transport/429/upload outcomes (B11–B14), orphan cleanup (B15),
  and the full update flow — load/two-snapshots (B17), editable-vs-disabled fields (B18),
  client validation (B19), deep-equal change detection (B20), narrowed 7-field payload (B21),
  success reseed-from-GET (B22), 429 (B24), upload+transport errors (B25). Full per-row table
  with file:line in the audit doc, Part 6.

### Mobile already does it better than web (do not "fix" toward web)

- **Copy feedback (B10):** web computes a `copied` state but never renders it (dead code);
  mobile renders a real 2s "Copied!" label (`UploadedProductDialog.tsx:199-203`). Keep mobile's.
- **`otherErrors` catch-all (B23):** web silently drops server field-errors that map to no
  inline slot; mobile collects them into `otherErrors` and renders them near Save
  (`updateSubmitOutcome.ts:74-77`; `[productId].tsx:403-418`). Keep mobile's.

### Out-of-parity-scope status (noted once, per brief)

- **reCAPTCHA:** web gates step-3 with an invisible reCAPTCHA; mobile has **no** reCAPTCHA
  gate. Deliberately excluded from parity — informational only, not a gap.
- **Analytics:** web fires `track(...)` events across both flows; mobile has **none**.
  Deliberately excluded from parity — informational only, not a gap.

### Open questions for Igor / Mastermind / Backend (Undetermined — need a product answer)

1. **Which product-URL domain is correct?** `www.oglasino.rs` (helper 1) vs `www.oglasino.com`
   (helper 2) vs web's `oglasino.com`. All three diverge; only code is visible, no brief
   confirms the intended mobile canonical. This is the decision the parity fix hangs on.
2. **Should the mobile copy URL include the locale segment** the way web's does?
   `getNormalizedProductUrl` takes no locale param and omits it.
3. **429 handling:** does the backend emit `system.rate_limited` (or another key) for a 429?
   Mobile only *parses*; it has no client-side synthesis, so a body-less 429 routes to the
   generic-error path (web synthesizes the system key). Confirm the backend contract.
4. **Translation-key seeding (Part 6 hand-off to Backend):** confirm these mobile-emitted keys
   are seeded — client ERRORS keys (`product.name/description.required|too_short|too_long`,
   `product.category.required`, `product.price.required`,
   `product.image.invalid_type|too_big|duplicate`) and mobile-only DIALOG/DASHBOARD keys
   (`new.product.success.copy.done.label`, `new.product.success.finish.label`,
   `new.product.success.failed.back`, `product.update.success.title/description`,
   `product.unchanged.title/description`, `product.update.fail.title/description`).
   No seed file exists in this repo to verify against.
5. **"View link" target:** does the create-success "View" route resolve if the freshly
   created product isn't yet publicly visible? Not exercisable in a read-only audit.

### Config-file dependency (closure gate)

- No config-file edit is required by this session. The product-URL divergence and the
  seeding/domain questions above are raised here for Mastermind to triage into `issues.md`
  or the next brief — I do not write those files (hard rule). Stated explicitly: **none required.**
