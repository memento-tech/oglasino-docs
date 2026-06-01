# Session summary

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-29
**Task:** Read-only audit. Inventory web's role in the `maintenance.active` → `maintenance.web.active` Cloudflare KV rename: deploy-workflow KV flips, every maintenance/admin.bypass reference, and confirm web does not read maintenance state from the worker/KV at runtime.

## Implemented

- Read-only audit. No code changes.
- Findings written to `.agent/audit-expo-maintenance-split.md` per the brief.
- Key results: (1) web flips the KV maintenance flag from **two** workflows — `deploy-stage.yml` (push `stage`) and `deploy-prod.yml` (push `main`) — each writing `maintenance.active` AND `admin.bypass.disabled` to `"true"` via Cloudflare KV PUT, then `sleep 60`; the flip is one-way (OFF is a manual operator step). (2) The KV key string lives in one `env:` line per workflow; the rename is a single-line edit in each of the two files. (3) Web does **not** read maintenance state from the worker/KV at runtime — confirmed no maintenance route, no `X-Oglasino-Maintenance`/`503` handling. (4) Nuance flagged: the in-app admin `MaintenanceToggle` drives a *separate* backend `/public/maintenance/*` flag, not the edge KV, and is not in scope of the rename.

## Files touched

- `.agent/audit-expo-maintenance-split.md` (new, audit deliverable)
- `.agent/2026-05-29-oglasino-web-expo-maintenance-split-1.md` (new, this summary)
- `.agent/last-session.md` (overwritten with a copy of this summary)
- No source files changed (read-only audit).

## Tests

- None run. Read-only audit; no code touched, so the lint/tsc/test gate does not apply to any changed path.

## Cleanup performed

- none needed (no code changes).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

(No config-file edit is required by this session. The 2026-05-29 issues.md entry on mobile's redundant backend maintenance check is adjacent context but needs no amendment from web's side.)

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): confirmed — no code changes, nothing left behind.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one observation flagged in "For Mastermind" (admin MaintenanceToggle ↔ edge KV are unwired; not a bug, a clarification).
- Part 6 (translations): N/A this session.
- Other parts touched: Part 8 (architectural defaults — edge boundary) — confirmed web has no runtime maintenance read, consistent with "maintenance state lives at the worker."

## Known gaps / TODOs

- none. The audit is complete against the current code.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code added.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- **Brief vs reality (worth surfacing before the rename brief is written):**
  - The brief says "the web deploy workflow" (singular) and "we believe web deploys from `stage`." Reality: there are **two** workflows that flip the same KV key — `deploy-stage.yml` (push `stage`) and `deploy-prod.yml` (push `main`). Both must be renamed together, or stage/prod diverge on the key name. (`deploy-stage.yml:4-5,14`, `deploy-prod.yml:4-5,13`.)
  - The maintenance OFF-flip is **manual** (no automated `false` PUT anywhere). Any runbook step that flips it off must also adopt the new key name. (`deploy-*.yml` notify steps.)
- **Part 4b adjacent observation (low severity):** The admin `MaintenanceToggle` (`src/components/client/buttons/MaintenanceToggle.tsx` → `configService.ts:48-78`) calls the **backend** `/public/maintenance/active` + `/public/maintenance/toggle` endpoints, which are a different flag from the Cloudflare KV edge maintenance key. The admin wrench does **not** flip the edge KV. This is not a bug in scope of the rename, but if anyone assumes "the admin maintenance toggle controls the edge gate," that wiring does not exist in web. Flagging so the rename design doesn't conflate the two. I did not change anything — out of scope and read-only.
- No drafted config-file text. No pending config-file dependency.
