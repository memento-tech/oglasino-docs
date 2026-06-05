# Session summary

**Repo:** oglasino-docs
**Branch:** main
**Date:** 2026-06-05
**Task:** issues.md only — flip the Status line on five entries and append dated correction notes beneath each (append-only; original body text preserved).

## Implemented

- **FLIP 1** — 2026-05-16 `isAllowedPath()` allowlist anti-pattern (oglasino-web): Status `open → fixed`; appended the filter-batch note (W7 mount relocation deleted isAllowedPath() and moved FilterManager to 5 page-level mounts; store-aware fresh-mount guard fixed the rehydrate regression; Igor-verified live; web `dev`, staged).
- **FLIP 2** — 2026-05-29 backend `/public/maintenance/active` redundant with edge worker (oglasino-expo): Status `open → fixed`; appended a "missed flip" note. The body already carried the expo-maintenance-split resolution sub-note; only the Status line was stale. No new work.
- **FLIP 3** — 2026-06-01 Mobile on-device UI/UX findings (batch) (oglasino-expo): Status kept `open` (standing container); appended a clarifying note that every line item is fixed-in-code ([x]) with only on-device Ψ confirmation outstanding (tracked in state.md Risk Watch).
- **FLIP 4** — 2026-05-31 Mobile Ψ on-device UI findings (batch) (oglasino-expo): Status kept `open` (standing container); appended the same-class clarifying note (all items fixed-in-code, most device-confirmed inline).
- **FLIP 5** — 2026-06-04 password-user email-change gate (oglasino-backend): held on the brief's "CONFIRM WITH IGOR FIRST" gate; asked Igor; Igor confirmed the mechanism. Status `open → fixed` with Igor's two-part note: (1) the block on password users changing email in-app is correct behavior (entry's bug-framing retracted), (2) the misleading "change with your provider" message was removed for password users.

## Files touched

- issues.md (5 Status flips + 5 appended notes; append-only, no original body text deleted or rewritten)

## Tests

- N/A (docs-only; no code, no test harness in this repo)

## Cleanup performed

- None needed. Append-only edits by mandate; no stale references, dead links, or superseded content introduced or exposed. The four-config-file revalidation: state.md / decisions.md / conventions.md unaffected (none reference these entries' open/fixed status); README and other docs do not index issues.md entry statuses.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: 5 entries amended (5 Status flips + 5 dated correction notes appended)

## Obsoleted by this session

- Nothing. Append-only; prior note bodies remain valid as history. The FLIP 2 stale Status line (the one inconsistency) is corrected, not deleted — its body's own resolution sub-note predates and supports the flip.

## Conventions check

- Part 4 (cleanliness): confirmed — append-only, no dead links or stale references created.
- Part 4a (simplicity) / Part 4b (adjacent observations): confirmed — see "For Mastermind."
- Part 6 (translations): N/A this session.
- Other parts touched: Part 3 (config-file writes) — all five flips are upstream-authorized (brief = Igor's briefing; FLIP 5 individually re-confirmed by Igor before flipping, per the brief's own gate and the standing "status flip needs session evidence" rule).

## Known gaps / TODOs

- None. All five entries handled. The brief's contingency (leave FLIP 5 open with a placeholder if Igor had not supplied the mechanism) did not trigger — Igor supplied the mechanism.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- FLIP 5 was the only gated item; resolved by asking Igor rather than guessing or flipping on the placeholder. The brief had pre-authorized a wontfix/placeholder fallback; Igor instead confirmed `fixed` with a concrete two-part mechanism (product decision + message-removal code fix), which is now on disk.
- Adjacent observation (low): the FLIP 2 entry is an example of the drift class the config-file discipline exists to prevent — the body carried a resolution sub-note but the Status line was never flipped. No action needed beyond this flip; noted as a pattern for Mastermind's awareness.
- Nothing else flagged.
