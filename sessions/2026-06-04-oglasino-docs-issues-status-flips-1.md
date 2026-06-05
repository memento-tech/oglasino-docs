# Session summary

**Repo:** oglasino-docs
**Branch:** main
**Date:** 2026-06-04
**Task:** Apply the confirmed 2026-06-04 `issues.md` status flips (append-only: flip the Status field + add a dated one-line resolution note; no deletions), leaving the two PENDING-confirmation items and the deferred store-listing items untouched.

## Implemented

- Mapped the brief's W#/E# codes onto `issues.md` entries (the file is dated-header, not code-indexed) by matching each brief description + date.
- **W5** — 2026-05-27 "No DOM test environment in `oglasino-web`": `open` → `parked`. Note: "Own infra decision, not a bug — adding a DOM test stack is a scope choice, not a defect."
- **W10** — 2026-05-19 "Backend → web cache revalidation may not work in production": `open` → `fixed`. Note: "Verified working in prod via direct-Vercel-URL (Igor, 2026-06-04)." Also tidied the now-stale severity tag `medium (verification pending)` → `medium` (verification is done).
- **E2** — 2026-06-01 "Orphaned Facebook sign-in scaffolding": `open` → `wontfix`. Note: "Scaffolding intentionally retained for future Facebook login; not a defect."
- **W8** (2026-05-16 211 ESLint) and **E5** (2026-05-25 two Firebase listeners): confirmed both already `parked` with current 2026-06-04 prod-bug-sweep notes — no action, as the brief expected.
- **E1** — 2026-06-02 "unused dependency `expo-tracking-transparency`": initially **held** for the contradiction below; **Igor confirmed the override**, so flipped `open` → `wontfix`. Note: "Removed at next prebuild as a build step; not separately tracked — a rebuild is owed regardless, so the drop rides it. Supersedes the prod-bug-sweep 'stays open' note." Igor's reason: a rebuild is required either way, so no separate tracker is needed.
- Left untouched per the brief: W4 and E4 (PENDING Igor confirmation), and W1/W3/W6 + E3 (deferred, blocked on store listings).

## Brief vs reality

1. **E1 close-as-wontfix contradicted a same-day "stays open" note — surfaced, then resolved by Igor**
   - Brief said: E1 (`expo-tracking-transparency`, 2026-06-02) → wontfix/closed; note "Removed at next prebuild as a build step; not separately tracked."
   - I saw: the entry already carried a **2026-06-04 (prod-bug-sweep)** note arguing the opposite — "stays open ... Not closeable as not-an-issue — there is a concrete pending action, and an unused ATT native module is App Store reviewer-bait. Fold into the next prebuild." Deliberately kept `open` that same day.
   - Why it mattered: closing as `wontfix` / "not separately tracked" risked losing the concrete pending action that prod-bug-sweep kept open for App-Store-review reasons.
   - **Resolution:** I held the flip and asked. Igor confirmed (1) yes, make it `wontfix`, and (2) no separate tracking needed — a rebuild is owed regardless of this dep, so the removal rides the rebuild and can't be "lost." Flip applied; the new note records the override and supersedes the prod-bug-sweep note.

## Files touched

- issues.md — four status flips + four dated resolution notes (W5 `parked`, W10 `fixed`, E2 `wontfix`, E1 `wontfix`); W10 severity tag tidied.

## Tests

- N/A — markdown only.
- Verification: post-edit grep confirmed the four new resolution notes present and the four Status fields flipped; W8/E5 confirmed already `parked` with current notes.

## Cleanup performed

- W10's stale `(verification pending)` severity qualifier removed (now verified). No dead links or stale references introduced; resolution notes are self-contained one-liners.
- No superseded content of mine to delete.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change. These are routine issue-status hygiene flips, not a project-state change; state.md's "Last updated" line already records the broader prod-bug-sweep flip round. No substantive state.md content to add without an upstream draft.
- issues.md: 4 status flips (W5 → `parked`, W10 → `fixed`, E2 → `wontfix`, E1 → `wontfix`), each with a dated one-line note; W10 severity tag tidied. E1 applied after Igor confirmed the override of the prod-bug-sweep "stays open" note (see Brief vs reality).

## Obsoleted by this session

- W10's `(verification pending)` severity qualifier — removed as superseded by the verification.
- Nothing else.

## Conventions check

- Part 3 (config-file writes): all four `issues.md` edits trace to the brief (upstream-drafted). The E1 flip — which contradicted an existing same-day entry — was surfaced and held per the challenge rules, then applied only after Igor explicitly confirmed the override. No unsourced substantive edit.
- Part 4 (cleanliness): stale severity qualifier removed; no dead links.
- Part 4a (simplicity): no new structures; reused the existing blockquote-note + Status-field pattern already used throughout issues.md for flips.
- Part 4b (adjacent observations): none new this session beyond the E1 contradiction, which is surfaced above rather than buried.
- Part 5 (numbering / archival): summary + last-session twin; `<n>=1` (first session for the `issues-status-flips` slug).
- Other parts touched: none.

## Known gaps / TODOs

- W4 and E4 remain `open` pending Igor confirmation, as the brief directs (W4 → recommend close as SEO decision; E4 → recommend keep parked/close as harmless). No action taken this session.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — doc edits only.
  - Considered and rejected: bumping state.md's "Last updated" / adding a session-log line for three routine status flips (rejected — not a project-state change; would be a substantive config-file edit without an upstream draft for state.md).
  - Simplified or removed: W10 stale severity qualifier.
- **Closure status.** All four flips from the brief are applied (W5, W10, E2, E1). The E1 contradiction was surfaced and resolved in-session by Igor's explicit override; no upstream config-file draft remains un-applied. W4/E4 are the only `open` items left, awaiting Igor's confirmation per the brief.
- Nothing else flagged.
