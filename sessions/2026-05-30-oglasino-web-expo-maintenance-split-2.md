# Session summary

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-30
**Task:** Web slice of the maintenance split + delete the orphaned admin toggle (expo-maintenance-split §6).

## Status: STOPPED before implementation — Brief vs reality blocker

No code was changed. I read the brief, the spec (`features/expo-maintenance-split.md`), the feature-scoped audit (`.agent/audit-maintenance-active-sweep.md`), and the actual code. The brief's central premise is factually false and every concrete file target in it is wrong. The brief itself instructs me to STOP in exactly this situation (lines 28–31: *"If any mount/import exists that the audit missed, STOP and report rather than ripping out a live reference."*). So I stopped.

## Brief vs reality

I read the brief and the code. Before starting work, I found:

1. **The admin `MaintenanceToggle` is MOUNTED and live — it is not orphaned.**
   - Brief says: *"the orphaned admin MaintenanceToggle UI is DELETED … the button was never mounted"* (lines 3–5) and *"It is not mounted/imported anywhere (audit-confirmed: only self-reference)"* (line 20).
   - Code says: it is imported and rendered on the admin config page — `app/[locale]/admin/config/page.tsx:3` (`import MaintenanceToggle …`) and `:24` (`<MaintenanceToggle />`). Grep for `MaintenanceToggle` returns the component definition plus this live import + mount.
   - The audit the brief cites says the **same thing the code does**, not what the brief claims: audit item 5 — *"the admin config page — the mount site (this is a real, live admin surface, not orphaned)."* The brief's "audit-confirmed: only self-reference / never mounted" misreads the audit.
   - Why this matters: deleting the file removes a live, working admin control and breaks the import at `page.tsx:3` → TypeScript/build error. This is "ripping out a live reference," which the brief's own line 30 says to stop on.

2. **The spec contradicts the brief: spec §6 says KEEP the toggle and repoint it, not delete it.**
   - Brief says: DELETE the toggle (decision (a)).
   - Spec says (`features/expo-maintenance-split.md` §6, lines 254–256): *"The web admin `MaintenanceToggle` UI button now drives the reworked backend toggle (4.2/4.3) — which writes both KV flags and is admin-gated. No web-side logic change beyond pointing at the moved endpoint path."*
   - Why this matters: the brief claims this is a "Decision locked by Mastermind" that supersedes the spec, but the locked decision rests on the false "never mounted / orphaned" premise in finding #1. With that premise gone, the delete-vs-repoint decision needs to be re-made by Mastermind against reality. Per CLAUDE.md I implement locked decisions as written — but not when the decision's stated factual basis is contradicted by both the code and the audit.

3. **Every file path and symbol name in the brief's deletion list is wrong.**
   - `MaintenanceToggle.tsx`: brief says `src/components/admin/maintenance/MaintenanceToggle.tsx` → **does not exist**. Actual: `src/components/client/buttons/MaintenanceToggle.tsx`.
   - `maintenanceService.ts`: brief says delete `src/lib/service/nextCalls/maintenanceService.ts` → **does not exist anywhere**; grep for `maintenanceService` across `app/` + `src/` returns nothing. There is no such wrapper.
   - `configService.ts`: brief says `src/lib/service/configService.ts` with methods `getMaintenanceStatus()` / `setMaintenanceStatus()` → **wrong path and wrong names**. Actual: `src/lib/admin/lib/service/configService.ts`, methods `isMaintenanceActive()` (`:48`) and `toggleMaintenance()` (`:64`). Grep for `getMaintenanceStatus`/`setMaintenanceStatus` returns nothing.
   - Why this matters: the deletion instructions cannot be executed as written; following them literally would do nothing (paths absent) while leaving the real, live code in place.

4. **Deploy-workflow filenames and edit shape are wrong.**
   - Brief says: `.github/workflows/deploy-production.yml` and `deploy-staging.yml`, "lines ~33 and ~39", change two `…/values/maintenance.active` occurrences.
   - Code says: files are `.github/workflows/deploy-prod.yml` and `deploy-stage.yml`. Each defines the key **once** as an env var — `MAINTENANCE_KEY: maintenance.active` (prod `:13`, stage `:14`) — and references it everywhere via `${MAINTENANCE_KEY}`. So the rename is a **single-line change per workflow**, not two. (Plus a stale literal comment: prod `:84` `# — Worker returns 503 when maintenance.active=true …`.)
   - Why this matters: the line numbers and "two occurrences" don't match; an agent following them literally edits the wrong spots. The intent (web deploy flips web-only) is achievable, but via the env-var line, and the brief's anti-goal (never flip `maintenance.backend.active`) is already satisfied — neither workflow touches a backend flag.

5. **`.env.local.example` has no maintenance line to delete.**
   - Brief item 3 says: delete the `NEXT_PUBLIC_MAINTENANCE_KV_KEY` line (~L6–8) and two comment lines.
   - Code says: a case-insensitive grep for `maintenance` in `.env.local.example` returns **zero** hits. There is no such line. (The audit also never listed this file as containing a maintenance reference.)
   - Why this matters: nothing to do; the instruction targets text that isn't there.

6. **`docs/admin-tools.md` does not exist.**
   - Brief item 4 says: update `docs/admin-tools.md`.
   - Code says: `docs/` contains `01-getting-started.md`, `02-architecture.md`, `03-development.md`, `04-deployment.md`, `05-environment-variables.md`, `06-pre-commit.md`, `07-pre-lunch-tasks.md`, `README.md` — no `admin-tools.md`. The maintenance docs debt actually lives in `04-deployment.md`, `07-pre-lunch-tasks.md`, `README.md`, `02-architecture.md`, `03-development.md`, `05-environment-variables.md` (see audit §(d)). Note CLAUDE.md forbids creating new files in `docs/`, so a new `admin-tools.md` could not be created anyway.

I have not started the implementation. Please pass these to Mastermind before I continue.

## What I believe the real web slice is (for Mastermind to confirm)

Grounded in the code + audit + spec §6, the genuinely-needed, low-risk web changes are:
- **Workflows (web-only KV rename):** `deploy-prod.yml:13` and `deploy-stage.yml:14` — `MAINTENANCE_KEY: maintenance.active` → `maintenance.web.active`; fix the stale prod comment `deploy-prod.yml:84`. No backend flag added (already absent). This matches the brief's intent and the spec.
- **Admin toggle (decision needed):** per spec §6, repoint `toggleMaintenance()` (`src/lib/admin/lib/service/configService.ts:66`) from `POST /public/maintenance/toggle` to the moved admin-gated path, and decide what `isMaintenanceActive()` (`:50`, `GET /public/maintenance/active`) becomes once spec §4.1 deletes that backend endpoint (drop the read / derive from toggle response / KV-backed read — audit open question #1). The exact secure path is unconfirmed (audit open question #2: `/secure/maintenance/toggle` vs `/secure/admin/maintenance/toggle` — sibling admin calls in this file use `/secure/admin/…`). This requires the backend contract, which is sequenced **before** web (spec §7). If Mastermind instead reaffirms DELETE, that is a real decision to make now that we know the toggle is live and the spec says keep — but it would delete a live admin control and diverge from spec §6.
- **Docs:** update the real maintenance-bearing `docs/*` files (audit §(d)); owner (web vs Docs/QA) is Mastermind's call.

I did not implement any of the above — listing it only so re-planning is fast.

## Implemented

- Nothing. Read-only verification only; stopped at the Brief vs reality gate.

## Files touched

- None.

## Tests

- Not run (no code changed). The lint/tsc/test gate is moot until there is a corrected brief.

## Cleanup performed

- None needed (no code changed).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change required from this session. (The delete-vs-repoint decision for the toggle is Mastermind's to confirm; if it changes, Mastermind drafts and Docs/QA applies — not me.)
- state.md: no change required from this session.
- issues.md: no change required from this session. (The path/premise errors are brief errors, not code findings; they belong in the Mastermind re-plan, not the issues backlog.)

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): N/A — no code changed.
- Part 4a (simplicity): see "For Mastermind".
- Part 4b (adjacent observations): one flagged below (stale comment in deploy-prod.yml). N/A otherwise.
- Part 6 (translations): N/A this session.
- Other parts touched: Part 11 (trust boundary) — noted only: spec's one trust fix (toggle public→secure) is real and lives behind the backend contract, sequenced before web.

## Known gaps / TODOs

- The web slice is not implemented, deliberately, pending a corrected brief.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- **Primary blocker:** the brief says the admin `MaintenanceToggle` is orphaned/never-mounted and should be deleted; it is in fact mounted and live at `app/[locale]/admin/config/page.tsx:24`, and spec §6 says to KEEP and repoint it. The "delete" decision needs re-confirmation against reality. (Findings #1, #2.)
- **Brief path corrections:** the brief's delete-list paths/symbols are all wrong (Finding #3); the workflow names/lines are wrong (Finding #4); the `.env.local.example` line doesn't exist (Finding #5); `docs/admin-tools.md` doesn't exist and can't be created per CLAUDE.md (Finding #6). A corrected brief should target: `src/components/client/buttons/MaintenanceToggle.tsx`, `src/lib/admin/lib/service/configService.ts` (`isMaintenanceActive`/`toggleMaintenance`), `deploy-prod.yml:13` + `deploy-stage.yml:14`, and the real `docs/*` files in audit §(d).
- **Sequencing note:** spec §7 puts web LAST and the backend toggle move (the secure path web must point at) BEFORE web. If the intent is for web to repoint the toggle now, the exact secure endpoint path must come from the backend (audit open questions #1 and #2). If the intent is web-only-workflow-rename-now-and-toggle-later, that's cleanly doable today in isolation.
- **Part 4b adjacent observation:** `deploy-prod.yml:84` has a stale literal comment `# — Worker returns 503 when maintenance.active=true …` (references the old key by literal, not via the env var, so it won't follow a rename). Severity: low (comment only). Not fixed — out of scope until there's a corrected brief.
- **Recommendation:** confirm (a) keep-and-repoint vs delete for the toggle, and (b) whether web should do the workflow rename now and defer the toggle repoint until the backend secure endpoint lands. I'll execute immediately on a corrected brief.
