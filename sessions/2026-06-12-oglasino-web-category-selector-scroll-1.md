# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-06-12
**Task:** BRIEF 3 — Web: category selector not scrollable in create-product dialog (audit + fix in one)

## Outcome: stopped at brief step 4 (NOT obvious → findings, no code shipped)

The brief authorized an in-session fix only if the cause was obvious (missing
`overflow-y-auto`, missing `max-height`, or a flex `min-h-0`). It is not. The
real cause is a multi-mechanism interaction (scroll-locked modal dialog + a
non-modal portaled popover + an ineffective bespoke wheel handler) whose fix
cannot be verified without a running, authenticated browser session against
the actual dialog. Per the brief, I stopped and wrote findings rather than ship
an unverified, likely-incomplete change. **Working tree is clean — no code
changes left on disk.**

I did trial a one-line fix mid-session (collapsing the nested scroller — see
Findings #3) and reverted it once the analysis below showed it is necessary
cleanup but not sufficient on its own.

## Findings — root cause

Component chain for the create-product category step:

- `DrawerDialog.tsx` — desktop branch renders a Radix `Dialog` (modal by
  default). `DialogContent` at `DrawerDialog.tsx:67-69`.
- `BasicInfoProductDialog.tsx:179` renders `CategorySelector` (the only caller).
- `CategorySelector.tsx:149` — `<Popover open={open} onOpenChange={setOpen}>`
  with **no `modal` prop** → Radix `Popover` defaults to `modal={false}`.
- `CategorySelector.tsx:159` — `DialogPopoverContent` (a bespoke wrapper in
  `popover.tsx`) → `Command` → `CommandList`.

Three interacting issues, in order of importance:

1. **(Primary) Non-modal popover is clipped out of the dialog's scroll-lock.**
   A modal Radix `Dialog` activates `react-remove-scroll`, which installs a
   document-level non-passive `wheel` listener that `preventDefault`s wheel
   events whose target is **outside** the dialog's locked subtree. The
   `Popover` content renders via `PopoverPrimitive.Portal` to `document.body`
   — outside that subtree — and, being `modal={false}`, never registers itself
   as a scroll-lock *shard*. Result: native wheel scrolling over the popover
   list is blocked by the dialog's scroll lock. This is the canonical
   "dropdown/popover won't scroll inside a Radix Dialog" situation.

2. **(Why the existing workaround doesn't save it) The custom wheel handler's
   fallback is dead code.** `DialogPopoverContent` (`popover.tsx:40-115`) was
   clearly built to work around exactly this — it has an `onWheel` fallback
   (`popover.tsx:63-79`) that manually does `el.scrollTop += e.deltaY` on its
   `max-h-[60vh]` scroll `div`. But the same element also has an
   `onWheelCapture` (`popover.tsx:50-54`) that calls `e.stopPropagation()` in
   the **capture phase**. React dispatches capture- and bubble-phase listeners
   for one event in a single ordered pass; `stopPropagation()` in the
   first (outermost capture) handler halts that pass before any bubble-phase
   `onWheel` runs. So the manual-scroll fallback never executes. The handler's
   only live effect is the capture-phase `stopPropagation`, which does nothing
   to `react-remove-scroll`'s separate native document listener.

3. **(Secondary) Redundant nested scroll containers.** Even if scrolling were
   unblocked, there are two competing scrollers: `DialogPopoverContent`'s
   `max-h-[60vh] overflow-y-auto` div (`popover.tsx:101-111`) wrapping shadcn's
   `CommandList` with its own `max-h-[300px] overflow-y-auto`
   (`command.tsx:78-86`). The `Command` between them is `overflow-hidden`
   (`command.tsx:21`). The inner `CommandList` is what actually overflows; the
   outer `60vh` div never does — so the (dead) fallback, which targets the
   outer div, would have nothing to scroll anyway.

## Recommended fix (for a follow-up brief, with browser verification)

Primary candidate — minimal and canonical:

- Add `modal` to the `Popover` in `CategorySelector.tsx:149`:
  `<Popover open={open} onOpenChange={setOpen} modal>`. A modal Radix Popover
  wraps its own content in `react-remove-scroll` and registers as the active
  shard, which is the documented way to let a popover scroll inside a
  scroll-locked dialog. Nested Radix locks (Dialog + Popover) are designed to
  cooperate. **Must verify:** (a) the list now wheel-scrolls inside the dialog;
  (b) outside-click / escape still close the popover and not the dialog
  unexpectedly; (c) focus behavior is acceptable.

If `modal` alone is insufficient or regresses dismissal behavior, the
fallback-repair route is to fix `DialogPopoverContent` so its manual scroll
actually runs (move the `preventDefault` + `scrollTop` logic into the
capture-phase handler, or stop killing the bubble pass) **and** collapse the
nested scroller (Finding #3) so the `60vh` div is the single scroll container.
That touches the shared `popover.tsx`, which also backs `CitySelector` (see
"For Mastermind"), so it needs broader regression and is the heavier option.

I recommend trying the `modal` prop first; it is the smallest change and the
standard remedy. I did not implement it because it needs live verification in
the actual modal dialog, which I cannot run in this session.

## Files touched

- none (trial change applied to `CategorySelector.tsx` and reverted; working
  tree clean per `git status`).

## Tests

- Ran (sanity, against the trial change before reverting): `npx tsc --noEmit`
  (exit 0), `npx eslint src/components/client/CategorySelector.tsx` (exit 0),
  `npx vitest run` → 35 files / 364 tests passed.
- No test files reference the category-selector / popover / command / dialog
  chain, so there is no targeted unit coverage to add for a CSS/scroll fix
  (jsdom does no layout, so wheel/scroll-lock behavior is not unit-testable
  here — verification must be in a real browser).
- New tests added: none.

## Cleanup performed

- none needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change. (This bug fix is not yet code-complete; no status flip.
  When fixed, it likely belongs as a note under the Contact/Support or a new
  create-product polish item — Mastermind's call. Not drafted here because the
  fix is not done.)
- issues.md: no change authored by me (I cannot write it). See "For Mastermind"
  for a draft entry on the dead wheel-handler fallback, which is a latent issue
  independent of this bug.

## Obsoleted by this session

- nothing. (The session produced findings only; no code was changed, so nothing
  was made dead. Note: if the `modal`-prop fix lands in a follow-up, the bespoke
  `DialogPopoverContent` wheel handler in `popover.tsx` would become dead and
  removable — flagged below, not acted on this session.)

## Conventions check

- Part 4 (cleanliness): confirmed — working tree clean, no stray code/comments.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): two flagged in "For Mastermind".
- Part 6 (translations): N/A this session — no strings touched.
- Other parts touched: none.

## Known gaps / TODOs

- The bug is not fixed; only diagnosed. A follow-up brief is needed to apply and
  browser-verify the recommended fix.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — no code shipped.
  - Considered and rejected: the single-line `CommandList` className override
    (`max-h-none overflow-x/y-visible`) to collapse the nested scroller — sound
    as cleanup but does not address the primary scroll-lock blocker, so shipping
    it alone would be a misleading half-fix; reverted.
  - Simplified or removed: nothing this session. (Future: the `modal`-prop fix
    would let us delete the entire bespoke `DialogPopoverContent` wheel handler
    in `popover.tsx` — a real simplification, deferred to the fix brief.)

- **Decision needed:** approve a follow-up fix brief. Recommended scope: add
  `modal` to the `Popover` in `CategorySelector.tsx:149`, then Igor verifies
  the list wheel-scrolls in the desktop create-product dialog and that popover
  dismissal/focus is unaffected. Same fix almost certainly needed for
  `CitySelector` (see below).

- **Adjacent observation (Part 4b) — high:** `CitySelector.tsx:123` uses the
  exact same `DialogPopoverContent` → `Command` → `CommandList` pattern with a
  non-modal `Popover`, so the city selector has the same non-scrolling bug
  wherever it appears inside a modal dialog. File:
  `src/components/owner/client/CitySelector.tsx`. Out of this brief's scope
  (category selector only); did not fix. Recommend folding into the same fix.

- **Adjacent observation (Part 4b) — medium / draft issues.md entry:** the
  `DialogPopoverContent` wheel-handler fallback in `popover.tsx:63-79` is dead
  code — its sibling capture handler's `stopPropagation` (`popover.tsx:50-54`)
  prevents the bubble-phase `onWheel` from ever firing. The component reads as
  a working scroll workaround but isn't one. Suggested issues.md entry (for
  Docs/QA to apply, since I cannot write the file):
  - Title: "DialogPopoverContent wheel-scroll fallback is dead code"
  - Severity: medium
  - Body: "`popover.tsx` `DialogPopoverContent` has an `onWheel` manual-scroll
    fallback intended to scroll the popover inside a scroll-locked Radix Dialog.
    It never runs: the element's `onWheelCapture` calls `stopPropagation()` in
    the capture phase, halting React's dispatch before the bubble-phase
    `onWheel`. The component therefore does not actually enable popover
    scrolling inside a modal dialog. Used by `CategorySelector` and
    `CitySelector`. Likely removable once the popovers are made `modal`."

- **Config-file closure gate:** the only config-file change this session would
  imply is the issues.md draft above (medium, latent). It is drafted here for
  Docs/QA. No other config-file edit is required by this session.
