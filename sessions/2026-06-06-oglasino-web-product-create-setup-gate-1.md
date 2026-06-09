# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-06-06
**Task:** READ-ONLY AUDIT — Audit the `USER_BASIC_DATA_SELECTOR_DIALOG` setup-gate behavior so mobile can mirror the behavior (not the mechanism). Mobile has no equivalent dialog and is failing because the create flow has no baseSite/regionAndCity gate.

## Implemented

- Read-only audit; no code changed. Traced the full web setup-gate from the `+` button through the dialog, the persist endpoint, the user-refresh path, and the product-create request shape.
- Confirmed the gate: `AuthAddNewProductButton` opens `USER_BASIC_DATA_SELECTOR_DIALOG` when `!user.baseSite || !user.regionAndCity`, else opens `CREATE_NEW_PRODUCT_DIALOG`.
- Established that the dialog collects base-site (single choice, `domain===code` only) then region+city, persists only the `RegionAndCityDTO` via `POST /secure/user/update/region-city` (base site is server-derived), refreshes the user via `refreshUser()` → `POST /auth/firebase-sync`, and auto-advances into the create dialog when `shouldOpenDialog` is true (the only caller passes `true`).
- Established the trust boundary on the web side: the product-create payload (`NewProductRequestDTO`) carries no location; the gate is UX only. Server-side enforcement of location-eligibility at create time is backend-owned and flagged as the open question for the backend audit.
- Wrote `.agent/audit-product-create-setup-gate.md` with the five answers plus a platform-neutral "What mobile must replicate" behavior contract.

## Files touched

- `.agent/audit-product-create-setup-gate.md` (new, audit output — not source)
- `.agent/2026-06-06-oglasino-web-product-create-setup-gate-1.md` (this summary)
- `.agent/last-session.md` (exact copy of this summary)

No source files modified.

## Tests

- Not run. Read-only audit, zero source changes — no touched paths to lint/typecheck/test.

## Cleanup performed

- none needed (no code written).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change — but see "For Mastermind": this audit is Phase 2 of a new feature (`product-create-setup-gate`) not yet listed under Active features; a state.md entry may be warranted once Mastermind opens the feature. Drafting that entry is not this session's job (Docs/QA writes it from a Mastermind draft).
- issues.md: no change — one adjacent observation flagged below for Mastermind to triage into issues.md if it chooses.

## Obsoleted by this session

- nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code, nothing to clean.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one observation flagged in "For Mastermind".
- Part 6 (translations): N/A this session (no keys added; audit only references existing keys).
- Other parts touched: Part 11 (trust boundaries) — explicitly audited; web carries no location in the create payload, server derives it; server-side create-time enforcement flagged as backend-audit-owned. Part 10 (feature lifecycle) — this is a Phase 2 audit; I read code only, not pre-existing reports.

## Known gaps / TODOs

- The server-side answer to brief Q5 (does `/secure/products/create` reject a location-less create, and with what code) cannot be determined from this repo. Documented in the audit §5 as the open question for the backend audit. This is not a gap in the web audit — it is out of this repo's scope.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **Adjacent observation (Part 4b):** `JoinFreeZoneButton` (`src/components/client/JoinFreeZoneButton.tsx:18–22`) opens `CREATE_NEW_PRODUCT_DIALOG` directly for any logged-in user with **no baseSite/regionAndCity gate** — a second create entry point that bypasses the setup gate that `AuthAddNewProductButton` enforces. A user without location reaching create via the free-zone page would hit whatever the server does on a location-less create.
  - File: `src/components/client/JoinFreeZoneButton.tsx:21`
  - Severity: medium (could mislead; user-facing impact depends on backend create-time enforcement — see audit §5).
  - I did not fix this because it is out of scope (read-only audit). Flagging for triage into `issues.md` and as relevant context for the seam analysis — the web gate is not applied uniformly even on web, which strengthens the case that the server must be the trust boundary.

- **Open question for the seam analysis (brief Q5):** the backend audit must answer whether `/secure/products/create` enforces location-eligibility server-side and with which error code. Web sends no location in the create payload and consumes no specific "location-not-set" code. Per Part 11 the gate alone is insufficient.

- **state.md note:** `product-create-setup-gate` is a new feature slug with no Active-features entry yet. If Mastermind formalizes the feature, a state.md row + (likely) a `features/<slug>.md` spec are owed before Phase 5 engineering per Part 10. Not drafted here — flagged so closure isn't mistaken for "no config dependency." No config-file edit is required *by this audit itself*.
