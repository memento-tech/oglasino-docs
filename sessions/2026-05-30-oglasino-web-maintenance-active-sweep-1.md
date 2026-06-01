# Session summary

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-30
**Task:** Read-only audit — grep the repo for every `maintenance` reference, classify each (deploy-workflow KV flip / admin toggle + configService / runtime read / docs / unrelated), and write the result to `.agent/audit-maintenance-active-sweep.md`.

## Implemented

- Swept the whole tree for `maintenance` (and `maintenance.active`, `/public/maintenance/active`, `/maintenance/toggle`, `MaintenanceToggle`), traced callers (`isMaintenanceActive`, `toggleMaintenance`, `MaintenanceToggle`), and checked the server path (`app/api`, `proxy.ts`, `middleware.ts`).
- Classified all 15 hit-bearing files into the brief's (a)–(e) buckets and wrote the full report to `.agent/audit-maintenance-active-sweep.md`.
- **Confirmed (c): web reads no maintenance state at runtime** — no `middleware.ts`, `proxy.ts` and `app/api` have zero maintenance refs, and the only read (`isMaintenanceActive`) is the admin wrench, rendered only in the admin config page.
- Surfaced the items the web implementation brief must pin down (the admin read after the poll is deleted; the exact secure toggle path; web-side doc debt).
- **No code changed** — read-only audit.

## Files touched

- `.agent/audit-maintenance-active-sweep.md` (new audit report; the file existed but was empty)
- `.agent/2026-05-30-web-maintenance-active-sweep-1.md` + `.agent/last-session.md` (this summary)
- No source, test, config, or workflow files modified. (Verified: `git status` shows no new modifications introduced by this session beyond the pre-existing working-tree changes that were already present at session start.)

## Findings (condensed — full detail + line numbers in the audit file)

- **(a) Deploy-workflow KV flips:** `.github/workflows/deploy-prod.yml` (key def L13) and `.github/workflows/deploy-stage.yml` (key def L14) each define `MAINTENANCE_KEY: maintenance.active` and PUT it to `"true"` on deploy (`${MAINTENANCE_KEY}` used throughout). Rename to `maintenance.web.active` — one line per workflow. Each also flips `admin.bypass.disabled` (unchanged by this feature). No `maintenance.backend.active` flip on a web deploy (spec §6). Prod comment at deploy-prod.yml:84 also names the old key.
- **(b) Admin toggle + service:** `src/lib/admin/lib/service/configService.ts` — `toggleMaintenance` POST `/public/maintenance/toggle` (L66) → must move to the admin-gated secure path (spec §4.3); `isMaintenanceActive` GET `/public/maintenance/active` (L50) → backend endpoint being deleted (spec §4.1). The wrench UI `src/components/client/buttons/MaintenanceToggle.tsx` calls both, and is **mounted** in `app/[locale]/admin/config/page.tsx:24`.
- **(c) Runtime reads:** confirmed none (admin-only read; no public/SSR/edge gate).
- **(d) Docs:** six `docs/*` files (heaviest: `04-deployment.md`, `07-pre-lunch-tasks.md`; plus `README.md`, `02-architecture.md`, `05-environment-variables.md`, `03-development.md`) and two `jobs/*` contract docs document the old single key + manual flip flow.
- **(e) Unrelated:** `RepairsMaintenanceCategoryIcon.tsx` + its barrel export (a product-category icon; coincidental word).

## Tests

- None run — read-only audit, no code changed. `npm run lint` / `tsc` / `npm test` not applicable (and not runnable here: `node_modules` is absent and the sandbox has no network).

## Cleanup performed

- None needed (no code touched).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change. (Expo Maintenance Split already sits at "Phase 5 engineering briefs next"; this audit is an input to the web brief, not a status flip.)
- issues.md: no change required.

## Obsoleted by this session

- Nothing. (The audit *identifies* code/docs the feature will obsolete — the old KV key, the `/public/maintenance/toggle` + `/public/maintenance/active` paths, the stale docs — but the audit itself obsoletes nothing.)

## Conventions check

- Part 4 (cleanliness): confirmed — no code added, no debug logging, no dead code. Scratch greps went to `/tmp`, not the repo.
- Part 4a (simplicity): N/A — no code written.
- Part 4b (adjacent observations): confirmed — observations are in the audit + below, none fixed (read-only).
- Part 6 (translations): N/A this session.
- Other parts touched: Part 10 (feature lifecycle) — Phase 2 read-only audit; output is `.agent/audit-<slug>.md` as specified.

## Known gaps / TODOs

- The audit's three open questions are for the web *implementation* brief: (1) the fate of the admin `isMaintenanceActive` read once `/public/maintenance/active` is deleted; (2) the exact secure toggle path (note the sibling admin calls use `/secure/admin/…`, suggesting `/secure/admin/maintenance/toggle`, not `/secure/maintenance/toggle` — confirm with backend); (3) web-side doc debt not listed in spec §7.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- **Brief vs reality:** the brief and the code agree. Two refinements worth surfacing to the spec, both flagged in the audit's open questions:
  - **Toggle path:** spec §4.3 says `/api/secure/maintenance/toggle`, but the existing admin config calls in the same service use `/secure/admin/config` and `/secure/admin/config/update` — so the established admin namespace is `/secure/admin/…`. The moved toggle may need to be `/secure/admin/maintenance/toggle`. Backend should confirm so web repoints to the right path.
  - **Admin read:** spec §6 only mentions repointing the toggle, but the wrench also calls `isMaintenanceActive()` GET `/public/maintenance/active`, which §4.1 deletes. The spec doesn't say what the read becomes. Needs a decision before the web brief.
- **Adjacent observations (severity):**
  - `toggleMaintenance` currently POSTs to **unauthenticated** `/api/public/maintenance/toggle` (`configService.ts:66`). **Medium** — this is the pre-existing auth hole the feature (§4.3) closes; flagged so it's tracked until the move lands. Not fixed (read-only, and the fix is the feature itself).
  - Web doc debt: `docs/04-deployment.md` and `docs/07-pre-lunch-tasks.md` carry copy-paste `curl …/values/maintenance.active` snippets that will mislead operators after the rename; not in spec §7's touch-point list. **Low.** Decide owner.
- **Suggested next step:** resolve the two spec refinements (toggle path, admin read) and the doc-owner question, then the Phase 5 web brief is a small, well-defined change (rename one env line per workflow; repoint one POST; decide the read).
- All config-file impacts are "no change"; no pending config-file drafts. Closure gate satisfied.
