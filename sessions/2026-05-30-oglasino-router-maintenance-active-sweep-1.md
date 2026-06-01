# Session summary

**Repo:** oglasino-router
**Branch:** stage
**Date:** 2026-05-30
**Task:** Read-only audit. Grep the entire repo for `maintenance.active` / `maintenance_active` / any `maintenance` reference; report EVERY hit with file:line and classification; report, do not change.

## Implemented

- Ran three repo-wide greps (`maintenance.active` literal, `maintenance_active` literal, and case-insensitive `maintenance`), excluding only `.git/`, `node_modules/`, and `package-lock.json`.
- Classified every hit per the brief's scheme: (a) old single key, (b) new keys, (c) comment/doc/test, (d) unrelated.
- Wrote the full audit to `.agent/audit-maintenance-active-sweep.md` with every file:line, the surrounding line, and a per-file summary table.
- **Key finding:** the old single `maintenance.active` KV key is GONE from all executable surfaces (`src/index.ts`, `tests/router.test.ts` use only `maintenance.web.active` + `maintenance.backend.active`). `maintenance_active` (underscore) appears nowhere. The old literal survives ONLY as stale documentation in four files: `docs/operations.md` (L41,42,45,46), `docs/architecture.md` (L14,50), `README.md` (L27,28), and `CLAUDE.md` (L68–70, the maintenance matrix block).

## Files touched

- `.agent/audit-maintenance-active-sweep.md` (new, +~190 lines) — the audit report
- No source, test, config, or docs file was changed. This was a read-only audit.

## Tests

- Ran: `npm run lint` (tsc --noEmit) → clean
- Ran: `npm test` (vitest run) → 45 passed, 0 failed
- New tests added: none (read-only audit)

## Cleanup performed

- None needed — read-only audit, no code touched.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change required by this session. The four stale-doc references the audit surfaced are repo-local docs/README/CLAUDE.md, not the four docs-repo config files. If Mastermind wants the stale runbook tracked as a known issue, draft text is in "For Mastermind" — but no config-file edit is implied by the audit itself.

## Obsoleted by this session

- Nothing was obsoleted by code change. The audit *identifies* already-obsolete content (the old `maintenance.active` key references in `docs/operations.md`, `docs/architecture.md`, `README.md`, `CLAUDE.md`) but did not remove it — out of scope for a report-only brief, and the docs/ + CLAUDE.md edits are not this agent's to make.

## Conventions check

- Part 4 (cleanliness): confirmed — no code added, nothing left behind.
- Part 4a (simplicity): confirmed — no abstractions or dependencies added; see "For Mastermind."
- Part 4b (adjacent observations): flagged in "For Mastermind" (stale docs + stale matrix block).
- Part 6 (translations): N/A this session.

## Known gaps / TODOs

- The audit only reports; the four stale-doc fixes are deliberately not done (read-only brief). Each needs a follow-up brief routed to its owner (Docs/QA for `docs/`, Igor/Mastermind for `CLAUDE.md`).

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing
  - Considered and rejected: nothing
  - Simplified or removed: nothing

- **Adjacent observations (Part 4b) — stale `maintenance.active` in docs, severity-ordered:**

  1. **Stale incident runbook — old KV key**
     - File: `docs/operations.md` lines 41,42,45,46
     - Observed: `wrangler kv key put/delete ... CONFIG maintenance.active` for stage + production. The worker no longer reads `maintenance.active`; it reads `maintenance.web.active` and `maintenance.backend.active`.
     - Severity guess: **high** — an operator following this during an incident would set a key the worker ignores and believe the site is in maintenance when it is not.
     - I did not fix this because it is out of scope (read-only brief; repo `docs/` content is Docs/QA's to write).

  2. **Stale architecture doc — single-key whole-site-503 model**
     - File: `docs/architecture.md` lines 14, 49, 50 (and prose at 17)
     - Observed: diagram "reads KV (maintenance.active)" and prose "`maintenance.active` instantly puts the entire site into a 503 state." Current model is per-client composition (web = web.active OR backend.active; mobile gated on backend only; admin bypass shapes who is blocked).
     - Severity guess: **medium** — misleads anyone reasoning about behavior; not directly operational.
     - I did not fix this because it is out of scope.

  3. **Stale README local-dev runbook**
     - File: `README.md` lines 8, 27, 28
     - Observed: `wrangler kv key put/delete ... maintenance.active` and "Maintenance page when KV flag is set" (singular).
     - Severity guess: **medium** — local-dev only, lower blast radius than ops.md.
     - README is editable by this agent for "how to work here" guidance, but I did not change it because the brief is explicitly report-only.

  4. **Stale maintenance matrix in CLAUDE.md**
     - File: `CLAUDE.md` lines 64–70 (referenced again at 144)
     - Observed: the "maintenance matrix" critical-care block still encodes a single `maintenance.active` flag:
       - `maintenance.active=false` → allow everyone
       - `maintenance.active=true, admin.bypass.disabled=false` → allow admin + API only
       - `maintenance.active=true, admin.bypass.disabled=true` → block everyone
       The code's actual matrix composes two keys (`webDown = maintenance.web.active OR maintenance.backend.active`) and adds a separate mobile path gated on `maintenance.backend.active` (+ optional probe) where the admin bypass does not apply.
     - Severity guess: **medium-high** — this block is the stated spec future briefs are checked against; leaving it single-key risks a future change "restoring" the old behavior to match the doc.
     - I did not fix this because CLAUDE.md is this agent's own instruction file; rewriting the matrix is Igor/Mastermind's call. **Recommend Igor update the CLAUDE.md matrix block to the two-key model in lockstep, since CLAUDE.md explicitly says the matrix is the spec.**

- **Suggested next step:** issue a small follow-up brief to refresh all four docs to the two-key model in one pass (Docs/QA for `docs/`; Igor for `CLAUDE.md`; this agent can take `README.md`).

- No config-file (`conventions.md`/`decisions.md`/`state.md`/`issues.md`) edit is required by this session. Stated explicitly per the closure gate.
