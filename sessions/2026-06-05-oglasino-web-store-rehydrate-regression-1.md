# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-06-05
**Task:** READ-ONLY audit — pin the store-rehydrate regression (on return to `/` or `/catalog` from a product page, controls + URL must rebuild FROM THE STORE, not clear it). Write findings to `.agent/audit-store-rehydrate-regression.md` answering Q1–Q5 with file:line + git evidence. No fixes.

## Implemented

- Read-only audit only. Produced `.agent/audit-store-rehydrate-regression.md` with file:line + git evidence for Q1–Q5.
- **Confirmed the regression cause = the FilterManager mount relocation** (`2026-06-04-oglasino-web-filter-mount-fix-1.md`), which is **uncommitted on `dev`** — `HEAD` still holds the BEFORE wiring, so BEFORE is directly inspectable via git.
- **Git evidence:** `HEAD:(public)/layout.tsx:11` (+ admin/owner layouts) mounts FilterManager in the THREE LAYOUTS; `git grep SelectableFilterManagerWrapper HEAD -- app/` returns only layouts. Working tree has the wrapper in FIVE PAGES instead. `HEAD:FilterManager.tsx` still has `isAllowedPath()` (fn :54, guards :79/:232); working tree has it deleted.
- **BEFORE mechanism pinned (Q1):** the product page is a child of `(public)/layout.tsx`, so the layout-mounted FilterManager **persisted across home→product→home** (App Router shared-layout, no remount). `isAllowedPath` blocked HYDRATE/SYNC on the product page (so the store wasn't cleared leaving home and `lastHydratedPathRef` stayed `'/'`); on return, guard `:80` `ref('/')===pathname('/')` early-returned the clearing HYDRATE, and SYNC pushed the surviving store → URL. That emergent property IS Igor's required behavior.
- **AFTER mechanism (Q2):** per-page mount → product→home fresh-mounts (`ref===null`) → HYDRATE runs against the param-less URL (product link strips query) → clears the store → stale-closure SYNC re-writes old params. Both halves of the relocation (mount-level change + `isAllowedPath` removal) are independently causal.
- **Fix (Q3/Q5):** recommended keeping per-page mounting and adding a store-aware fresh-mount guard in HYDRATE — skip the clear when `ref===null && hydrated && storeHasFilters && urlParamless`, let SYNC repopulate. Discriminator (Q3b) is the singleton-level `hydrated` flag (false on hard refresh, true on surviving client-nav). No product-URL params (Q3c). Loop-safe (HYDRATE keys on pathname, SYNC changes only search). Covers `/` and `/catalog` identically (same portal singleton) (Q4). Partial revert (Option A) reintroduces the W7 string-allowlist; the guard (Option B) preserves the cleanup and makes the behavior explicit — recommended.

## Files touched

- `.agent/audit-store-rehydrate-regression.md` (new, audit deliverable)
- `.agent/2026-06-05-oglasino-web-store-rehydrate-regression-1.md` (this summary)
- `.agent/last-session.md` (exact copy of this summary)

No source files modified (read-only brief).

## Tests

- None run — read-only audit, no code touched. lint/tsc/test not applicable.

## Cleanup performed

- none needed (no code changed).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change.
- issues.md: no change applied by me. The URL/control/SSR divergence is already a flagged issues.md candidate (per `audit-hydrate-skip-clientnav.md`); this audit adds the regression-point analysis but authors no entry (Docs/QA write).

## Obsoleted by this session

- nothing.

## Conventions check

- Part 4 (cleanliness): N/A — no code changed (read-only audit).
- Part 4a (simplicity): N/A — no code added. The fix recommendation explicitly weighs simplicity (Option B preserves the W7 cleanup vs Option A re-adding the string allowlist).
- Part 4b (adjacent observations): one re-flagged (pre-existing exhaustive-deps warnings on both FilterManager effects, already on record; the `[baseSite, pathname]` HYDRATE deps omitting the query string is the self-correction gap noted in the prior audit). Not introduced here.
- Part 6 (translations): N/A this session.
- Other parts touched: Part 10 Phase 2 (audit) — followed: read the actual code + git, not pre-existing reports.

## Known gaps / TODOs

- The audit reasons from code + git, not a live browser repro. The BEFORE/AFTER mechanism is internally consistent with all observed facts and directly supported by `HEAD` vs working-tree diffs. If Mastermind wants runtime confirmation, the cheapest check is: on `HEAD`, home(filtered)→product→home shows filters restored; on the working tree, same nav shows empty control + params in URL.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code.
  - Considered and rejected: nothing — no implementation.
  - Simplified or removed: nothing.

- **Brief vs reality:** no mismatch — the brief's PRIME SUSPECT (Q1) is correct and confirmed by git. One refinement worth surfacing: the regression has **two** load-bearing components, not one — the mount-level change (lost persistence) AND the `isAllowedPath` removal (lost the product-page block). Both were deleted in the same session; either alone would also break the BEFORE behavior. Any fix or partial-revert must address both (a layout-only revert without restoring `isAllowedPath` would still clear the store on the product page).

- **Recommendation for the Phase-5 brief:** Option B (per-page mount + store-aware HYDRATE guard). It satisfies Igor's clean-product-URL requirement (no product-link change), restores the BEFORE "rehydrate from store" behavior, and keeps the deliberate W7 simplification. The guard must be gated on `lastHydratedPathRef.current === null` so catalog category-switch re-hydration (`/catalog/x → /catalog/y`, no remount) is untouched. Option A (restore layout mount + `isAllowedPath`) is the zero-new-logic fallback but re-accepts the W7 anti-pattern.

- **Adjacent flags:** (low) the logo→`/` category-scoped-filter drop tracked by `audit-filter-rehydrate-bug.md` is a *separate* bug from this regression; the store-skip fix sidesteps it for the return-nav case (it doesn't read the URL), but it is not fixed by this work. (low) pre-existing `react-hooks/exhaustive-deps` warnings on both FilterManager effects — already on record, behavior-sensitive, own brief.

- **Config-file dependency (closure gate):** none required for this read-only audit to close. No drafted config-file text. If Mastermind opens a tracking entry, it would fold into the existing `audit-hydrate-skip-clientnav` issues.md candidate, not a new one.
