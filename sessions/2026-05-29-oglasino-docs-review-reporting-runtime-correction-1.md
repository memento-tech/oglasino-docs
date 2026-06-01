# Session summary

**Repo:** oglasino-docs
**Branch:** main
**Date:** 2026-05-29
**Task:** Correct the issues.md 2026-05-28 "Review-reporting wire" entry — replace the wrong "Runtime behaviour today / two outcomes" block with the audit-corrected HTTP 500 description.

## Implemented

- Replaced the "Runtime behaviour today" paragraph (the "Two outcomes, both wrong" misfile/422 block) in the 2026-05-28 "Review-reporting wire is an unfinished feature" entry with the corrected text from the 2026-05-29 backend audit (`.agent/audit-system-error-code.md`, Part C.10).
- Corrected behaviour: `ReportType` has only `PRODUCT`/`USER`; web's literal `"REVIEW"` fails Jackson deserialization before any service code runs → catch-all → HTTP 500 `INTERNAL_ERROR`. The PRODUCT existence check is never reached, so there is no silent mis-file and no `REPORTED_PRODUCT_NOT_FOUND` 422.
- Used the translation key `product.system.internal_error` (NOT the renamed `system.internal_error`) because `system-error-code-split` is still `planned`, not shipped — per the draft's explicit Factual-note conditional.
- Left the rest of the entry unchanged: severity (medium), status (parked), the a/b/c resolution options, the wire-mismatch framing. The audit confirmed only the runtime-behaviour description was wrong.

## Files touched

- issues.md (~+1 / -4 lines — one block replaced)

## Tests

- N/A (markdown-only repo).

## Cleanup performed

- None needed — single in-place block replacement; no dead links or stale references introduced.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: 1 entry amended (2026-05-28 "Review-reporting wire" — runtime-behaviour block corrected). Substantive edit with an upstream drafter (2026-05-29 backend audit, via Igor's brief).

## Obsoleted by this session

- The prior "two outcomes, both wrong" misfile/422 runtime description in that entry — deleted in this session (it described a code path that does not exist).

## Conventions check

- Part 1 (doc style): confirmed — body-text formatting matched what was replaced; no blockquote substitution.
- Part 3 (config-file writes): confirmed — substantive issues.md edit applied from an upstream draft, not a Docs/QA independent fix.
- Part 4 (cleanliness): confirmed.
- Part 4a / 4b: N/A — verbatim drafted correction, no Docs/QA-authored substance; no adjacent observations.
- Part 6 (translations): N/A.

## Known gaps / TODOs

- None. The correction is self-contained.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing.
  - Considered and rejected: nothing.
  - Simplified or removed: removed the incorrect misfile/422 description.
- Key-name note carried into the applied text: it reads `product.system.internal_error` because `system-error-code-split` has not shipped. If/when that feature ships and renames the key, this entry's `product.system.internal_error` reference becomes stale and should flip to `system.internal_error` — worth folding into the `system-error-code-split` docs-cleanup brief so it isn't missed.
- issues.md is edited on disk and ready for Igor to commit.
