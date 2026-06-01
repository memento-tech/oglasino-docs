# Session summary

**Repo:** oglasino-router
**Branch:** stage
**Date:** 2026-05-30
**Task:** Update the router repo's own docs to the two-key maintenance model. The CODE is already migrated (src/index.ts + tests use only maintenance.web.active / maintenance.backend.active); this brief fixes the stale DOCS that still describe the old single maintenance.active key.

## Implemented

- Rewrote the **CLAUDE.md** "maintenance matrix" block (the agent-instruction spec) from the old 3-row single-`maintenance.active` matrix to the two-flag, per-client composition: web (`maintenance.web.active OR maintenance.backend.active`, admin-bypass-shaped) vs mobile (`maintenance.backend.active OR probeFailed`, no web flag, no admin bypass), plus the `/api/mobile/*` label, the `/actuator/health/readiness` probe target (mobile-only), and `use.backend.check`. Matches the top-of-file comment in `src/index.ts`.
- Fixed **README.md**: the singular "when KV flag is set" prose (L8) now names both flags and the per-client composition; the local-dev runbook commands now write/delete `maintenance.web.active` instead of the dead `maintenance.active`, with a note that web maintenance is set via `maintenance.web.active`.
- Fixed **docs/operations.md** (highest operational risk): the incident runbook put/delete commands for stage+production now target the two real keys, split into a "web maintenance window" group (`maintenance.web.active`) and a "backend maintenance window" group (`maintenance.backend.active`), with a per-client note (web flag = web only; backend flag = web AND mobile) and an explicit warning that any other key has no effect.
- Fixed **docs/architecture.md**: the ASCII diagram now reads both flags and notes per-client composition + mobile 503; the "Why the same Worker" prose replaced the "single `maintenance.active` flag instantly 503s the entire site" model with the per-client two-key composition (web vs mobile, admin-bypass shaping, backend-down takes both down).
- Reworded the two surviving negative references to the old key (CLAUDE.md, operations.md) so the literal token `maintenance.active` no longer appears in any doc/code file, satisfying the brief's grep DoD while keeping the "no single combined key" warning.

## Files touched

- CLAUDE.md (matrix block rewrite; ~+30 / -6)
- README.md (L8 prose + runbook commands; ~+9 / -5)
- docs/operations.md (Toggle maintenance mode section; ~+30 / -12)
- docs/architecture.md (diagram + "why same worker" prose; ~+22 / -7)

No code, test, or config files touched (per brief: src/index.ts, tests/router.test.ts, wrangler.toml, package.json all out of scope).

## Tests

- Ran: `npm run lint` (tsc --noEmit) → passed, no output (clean).
- Ran: `npm test` (vitest run) → 45 passed, 0 failed (1 file).
- New tests added: none (docs-only change; no automated test surface for docs).
- DoD grep: `rg 'maintenance\.active'` across repo (excl. `.git`, `node_modules`, `.agent`) → NO MATCHES. The only remaining references live in `.agent/` (the audit + this brief), which correctly document the old key.

## Cleanup performed

- Removed the last `maintenance.active` literal from all doc/code files (rephrased two intentional negations rather than deleting the warning). None needed beyond the brief's scope.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change. (The "Expo Maintenance Split" entry already says `planned` / Phase 5 engineering next; this docs cleanup is part of that feature's router work and does not by itself warrant a status flip — that is Mastermind/Docs-QA's call when the router code lands on prod.)
- issues.md: no change. (Two adjacent observations flagged below for Mastermind triage; none authored by this agent.)

## Obsoleted by this session

- The stale single-key documentation in CLAUDE.md, README.md, docs/operations.md, docs/architecture.md — all rewritten in this session. Nothing left for follow-up in these four files.
- The `.agent/audit-maintenance-active-sweep.md` "Recommended follow-up" items 1–4 (the four doc fixes) are now done; the audit file itself is a historical record and is left in place.

## Conventions check

- Part 4 (cleanliness): confirmed. No commented-out blocks, no TODO/FIXME, no console.log, no new files (edited existing docs only — Part 1 long-term-goal / Part 3 "no new files in <repo>/docs/" respected: zero files created in docs/).
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): two flagged in "For Mastermind".
- Part 6 (translations): N/A this session (no user-facing strings touched).
- Other parts touched: Part 1 (doc style — ATX headings, kebab-case, fenced blocks preserved); Part 8 ("Cloudflare router worker is the edge boundary; maintenance state lives there" — the rewritten docs reinforce this).

## Known gaps / TODOs

- none.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — prose/comment edits only, no abstractions, no config, no code.
  - Considered and rejected: keeping the explicit `maintenance.active` literal in the two "the old key is gone" warnings (it aids an operator who greps for the old key from muscle memory). Rejected because the brief's DoD is an explicit zero-literal grep; reworded to "no single combined maintenance key" to satisfy it while preserving the warning. Flagging the tradeoff here so you can decide if a one-line "(formerly `maintenance.active`)" pointer is worth re-adding to operations.md for searchability.
  - Simplified or removed: removed the old 3-row single-key matrix from CLAUDE.md and the single-key whole-site-503 prose from architecture.md, replacing both with the per-client composition that matches `src/index.ts`.

- **Adjacent observation 1 (out of scope, not fixed):** `docs/operations.md` "Toggle backend liveness probe" section (~L58) still says the worker probes `${BACKEND_ORIGIN}/health`. The live probe target in `src/index.ts:146` is `/actuator/health/readiness` (changed by the maintenance-split feature, spec §3.4). The brief scoped me to the runbook commands (L41–46) and said to leave the probe prose (L59), so I did not touch it. Severity: medium — an operator reading this would expect the wrong probe endpoint. I did not fix this because it is out of scope.
- **Adjacent observation 2 (out of scope, not fixed):** same section (~L58–60) says "the Worker probes ... before forwarding. If the probe fails, requests get the maintenance response automatically." Post-split the probe gates **only** `/api/mobile/*` requests (not web/apex/API-host traffic) — see `src/index.ts:139–156`. The prose reads as whole-site. Severity: medium — misleads on which clients the probe affects. I did not fix this because it is out of scope (brief said leave the probe prose). Both observations are the same feature's residue; suggest folding both into a one-line follow-up brief for operations.md.

- **Adjacent observation 3 (low):** `CLAUDE.md` "Fail-open on KV errors" section (~L76) says the worker "treats both flags as false" — there are now four flags (`maintenance.web.active`, `maintenance.backend.active`, `admin.bypass.disabled`, `use.backend.check`), and `src/index.ts:128` itself says "all four flags fall back to false." Stale count in a file I touched. Out of the brief's stated line range (64–70); I did not fix it. Trivial reword ("all four flags") if you want it in a follow-up.

- **Config-file dependency check (closure gate):** none. This session requires no edit to conventions.md, decisions.md, state.md, or issues.md. The three adjacent observations above are flagged for your triage; if you want any logged to issues.md, that is a Docs/QA action, not drafted here.

- **Brief vs reality:** none — the brief matched the code exactly. `src/index.ts` implements precisely the two-key per-client composition the brief described (web = web OR backend; mobile = backend OR probe; mobile ignores web.active; admin bypass web-only; probe mobile-only at `/actuator/health/readiness`). No discrepancy to challenge.
