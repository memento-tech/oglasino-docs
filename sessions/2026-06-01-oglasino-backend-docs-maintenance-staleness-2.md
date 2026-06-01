# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-06-01
**Task:** FIX — Correct stale single-flag `maintenance.active` references across five
`docs/` pages (01/02/04/06/11) to the two-flag model. Docs-only. Follows the
read-only audit `2026-06-01-oglasino-backend-docs-maintenance-staleness-1.md`.

---

## Implemented

Corrected every stale `maintenance.active` reference in the five pages to the
two-flag gate model (`maintenance.web.active` + `maintenance.backend.active`,
plus `admin.bypass.disabled`), and corrected the restore framing to the 3-step
manual procedure. All corrected wording traces to the in-repo authoritative
source `docs/12-deployment.md` (already migrated) and the sibling runbook
`../oglasino-docs/infra/cloudflare/maintenance.md` — nothing invented from memory.

Per the brief's tool caution, every target line was re-grepped for the exact
string before editing, and re-grepped after to confirm zero `maintenance.active`
references remain anywhere in `docs/`.

### Per-file changes

**`docs/01-backend-pre-lunch-tasks.md`**

- `:155` (§5 "Why") — stale name + single-step-restore implication corrected:
  now names both flags and states there is **no auto-off**; restore is a manual
  procedure.
- §5 "How" (was the `:162`/`:176` broken curls) — removed the two raw
  `maintenance.active` curls. Per the brief (Part 4a), §5 now points at
  `12-deployment.md#flipping-maintenance-off` and names the `oglasino-docs`
  runbook for the canonical two-flag flip, keeping only the contract-specific
  `X-Oglasino-Maintenance` header check. (This is the same worker-contract test
  as frontend #4.)
- `:278` (workflow step "maintenance-on") — corrected: a backend deploy PUTs
  **both** `maintenance.web.active=true` and `maintenance.backend.active=true`
  (plus `admin.bypass.disabled=true`), per `12-deployment.md:87`.
- `:313` (was `:327`, §13 step 5 broken OFF curl) — replaced the raw
  `maintenance.active --data "false"` curl with the 3-step manual restore
  (admin-bypass-allow.sh → refresh caches → maintenance-off.sh) as a pointer to
  `12-deployment.md` / the runbook.
- `:337` (was `:352`, "Things I can't check" checklist) — corrected the KV-key
  name to the two keys.

**`docs/02-deploy-checklist.md`**

- `:154` — KEY INVENTORY corrected: was "two keys: `maintenance.active=false`,
  `use.backend.check=false`"; now "four keys: `maintenance.web.active=false`,
  `maintenance.backend.active=false`, `admin.bypass.disabled=false`,
  `use.backend.check=false`." `:156` ("the two KV keys") updated to "the four KV
  keys" for inventory consistency.
- Phase 7 ON flip (`:475` broken curl) — replaced the single `maintenance.active`
  PUT with a both-key loop (`maintenance.web.active` + `maintenance.backend.active`),
  noting a backend deploy also sets `admin.bypass.disabled`.
- Phase 7 OFF flip (`:509` single-key flip) — corrected to flip both flags, with
  an explicit note that a real post-deploy restore is the 3-step manual procedure
  (pointer to `12-deployment.md`); the rehearsal only exercises flag mechanics.
- Phase 9 state check (`:578` broken GET) — key corrected to
  `maintenance.backend.active`; kept the conceptually-correct "the workflow does
  NOT flip them off" point; added that a backend deploy also sets
  `maintenance.web.active` and `admin.bypass.disabled`.
- Phase 10 restore (`:620`) — re-titled to "Restore traffic (3-step manual
  procedure — there is no single-step off)"; the How now lists
  admin-bypass-allow.sh → refresh caches → maintenance-off.sh and points at
  `12-deployment.md` / the runbook rather than re-documenting. If-fails updated to
  reference `maintenance-off.sh`'s underlying both-key curls.
- Quick reference (`:706` broken curl) — ON is now a both-key loop; the OFF line
  is replaced by a pointer to the 3-step manual restore (no single-step off).

**`docs/04-db-reset-runbook.md`**

- `:445` (Troubleshooting "stuck ON" fallback) — the single broken
  `maintenance.active --data "false"` curl replaced with two curls flipping
  **both** `maintenance.backend.active` and `maintenance.web.active` to `"false"`,
  mirroring the laptop-CF-API fallback in `12-deployment.md:138–150`. The Step 1 /
  Step 9 / quick-ref `maintenance-on.sh` / `maintenance-off.sh` references were
  **not** touched (already two-flag-correct).

**`docs/06-architecture.md`** (stale-name only)

- `:30` (mermaid KV node) — `maintenance.active` → `maintenance.web.active` +
  `maintenance.backend.active`.
- `:104` (read-path prose) — now describes reading both flags and composing them
  per client, with a pointer to `12-deployment.md`.
- `:150` (sequence step) — `GET maintenance.active` → `GET maintenance.* flags`.

**`docs/11-environment-variables.md`** (stale-name only)

- `:212` (`CF_API_TOKEN` purpose) and `:214` (`CF_KV_NAMESPACE_ID` purpose) —
  `maintenance.active` → both keys.

---

## Files touched

- `docs/01-backend-pre-lunch-tasks.md`
- `docs/02-deploy-checklist.md`
- `docs/04-db-reset-runbook.md`
- `docs/06-architecture.md`
- `docs/11-environment-variables.md`
- `.agent/2026-06-01-oglasino-backend-docs-maintenance-staleness-2.md` (new — this summary)
- `.agent/last-session.md` (overwritten — exact copy)

No code changed. No new `docs/` files created.

## Tests

- No code touched; no test/lint/tsc applies (docs-only, per the brief).
- Verified post-edit: `grep -rn "maintenance\.active" docs/` → **NONE**.
- Verified both `06-architecture.md` mermaid blocks still have balanced fences
  (two `` ```mermaid `` opens, two closes at lines 24/61 and 141/170); both edited
  lines (`:30` cylinder node, `:150` sequence message) are inside valid blocks and
  use mermaid-legal label syntax.

## Cleanup performed

- Removed two dead `maintenance.active` curl blocks from `01` §5 (replaced by a
  pointer) and the dead OFF curl from `01` §13. No commented-out code, no orphaned
  references left. No `TODO`/`FIXME` added.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change.
- issues.md: **no change made by me.** This FIX resolves the corrective half of
  the 2026-05-30 entry ("Backend docs 01/02/04/06/11 still reference the single
  `maintenance.active` KV key"). A drafted close-out note for Docs/QA is in
  "For Mastermind" below — draft only; I did not edit the file.

## Obsoleted by this session

- The audit `2026-06-01-oglasino-backend-docs-maintenance-staleness-1.md` is now
  acted upon — its per-page findings are all corrected on disk. The audit remains
  as the historical record; nothing else is obsoleted.

## Conventions check

- **Part 1 (docs):** confirmed — only existing `docs/` files edited; no new docs
  file created. The brief's "editing existing `<repo>/docs/` files is allowed"
  scope was respected.
- **Part 4 (cleanliness):** confirmed — dead curls removed in the same session;
  no commented-out code, no stray debug, no unmatched TODO.
- **Part 4a (simplicity):** confirmed — for the restore/flip procedures I pointed
  at the single in-repo source (`12-deployment.md`) and the runbook instead of
  re-documenting the 3-step restore in each page, per the brief. Where the brief
  explicitly permitted "both-key curls" (Phase 7 rehearsal, quick-ref, `04`
  fallback) I kept them runnable rather than over-stripping.
- **Part 4b (adjacent observations):** see "For Mastermind."
- **Part 6 (translations):** N/A.
- **Part 8 (edge boundary):** the five pages now agree with the "maintenance lives
  at the edge worker, two composed flags, manual restore" model already reflected
  in `12-deployment.md` and the runbook.

## Known gaps / TODOs

- None for the in-scope fix. The `02` Phase 9 If-fails line ("if the flag is
  somehow OFF...") still reads singular in prose; it does not reference the dead
  key, so I left it untouched to avoid scope creep.

## For Mastermind

- **Out-of-scope finding left untouched (as instructed):** the
  `CLOUDFLARE_KV_CONFIG_TOKEN` write-scope question (`11:136`, `06:109–110`) — the
  brief flagged it as an unconfirmed separate finding. Not changed. Still open as a
  possible follow-up (whether the backend retains any runtime KV-**write**
  capability post-split).

- **Drafted `issues.md` close-out (for Docs/QA to apply, not me):** amend the
  2026-05-30 entry to: "FIXED 2026-06-01 (session
  `docs-maintenance-staleness-2`). All stale `maintenance.active` refs in
  `01/02/04/06/11` corrected to the two-flag model and traced to
  `docs/12-deployment.md` + `infra/cloudflare/maintenance.md`. `02` additionally
  corrected: `:154` key inventory (now four keys) and the restore model (Phase 7
  rehearsal + Phase 10 now reflect the 3-step manual restore). `04` was the single
  `:445` fallback curl. `grep -rn maintenance.active docs/` → none. Remaining
  follow-up: `CLOUDFLARE_KV_CONFIG_TOKEN` runtime KV-write scope (`11:136`),
  tracked separately." (Closure gate: draft only; no config file edited.)

- **Closure gate confirmation:** no implicit config-file dependency beyond the
  drafted issues.md close-out above. The fix is self-contained in `docs/`.
