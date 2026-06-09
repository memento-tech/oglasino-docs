# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-06-06
**Task:** Read-only audit. No code changes. Write report to `.agent/audit-admin-app-version-control.md` and summarize in response. Five things: the store, the read, the writes, the admin-endpoint gap, semver parsing (+ trust boundary note).

## Implemented

- Read-only Phase-2 audit only. No source changes.
- Produced `.agent/audit-admin-app-version-control.md` answering all five questions plus the Part 11 trust-boundary note, with quoted code and absolute file paths.

## Files touched

- `.agent/audit-admin-app-version-control.md` (new, audit report)
- `.agent/2026-06-06-oglasino-backend-admin-app-version-control-1.md` (this summary)
- `.agent/last-session.md` (copy of this summary)

No source/test/config files modified.

## Tests

- Not run. Read-only audit; no code changed.

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change (Mastermind may add an "admin app version control" feature row after Phase 3/4; that is Docs/QA's write, not drafted here)
- issues.md: no change required. Two adjacent flags raised in "For Mastermind" (no value-validation on the internal floor write; no actor/history on the row) — Mastermind decides whether either becomes an issues.md entry.

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code touched, no debug logging, no dead code.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): two flagged in "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: Part 7 (error contract) — referenced in the audit's recommendation that a new admin endpoint return codes not messages. Part 11 (trust boundaries) — explicitly checked, the brief's required note. Part 12 (V1 schema fold) — referenced re: cheap pre-prod schema add if an actor column is wanted.

## Key findings (mirrors the audit report)

1. **Store:** `AppVersionConfig` = `platform`, `latestVersion`, `minSupportedVersion`, extending `BaseEntity` → carries `createdAt` + `updatedAt` (auto-bumped on save; no actor column). Table `app_version_config` has `UNIQUE (platform)` → one live row per platform. No history/audit table; only the two live values per platform. Seeded: android + ios at 0.0.0/0.0.0.
2. **Read:** `GET /api/public/app/version/{platform}?currentVersion=`. `forceUpdate` = current < floor(`minSupportedVersion`); `optionalUpdate` = current < ceiling(`latestVersion`) AND NOT force (mutually exclusive). Fails open: missing row → 200 NO_UPDATE; unparseable semver → caught, 200 NO_UPDATE. Never throws.
3. **Writes:** `POST /internal/app/version/floor` (DTO `{platform, minSupportedVersion}`, writes floor only) and `POST /internal/app/version/ceiling` (DTO `{platform, latestVersion}`, writes ceiling only, the EAS post-build hook target). Both guarded by `InternalTokenFilter` (X-INTERNAL-TOKEN, constant-time compare). Both **UPDATE-ONLY** → 404 on missing platform row, no upsert. No `/api/secure/admin/**` version endpoint exists.
4. **Admin gap:** admin gate = `@PreAuthorize("hasRole('ADMIN')")` at the controller class level (e.g. `CacheAdminController`, `MaintenanceAdminController`), NOT a SecurityConfig path matcher — SecurityConfig only gates `/api/secure/**` as `authenticated()`. No version admin endpoint today; a new `/api/secure/admin/...` controller with the class-level `@PreAuthorize` must be added.
5. **Semver:** `com.github.zafarkhaja:java-semver` 0.10.2; `Version.parse(...).isLowerThan(...)`. Strict three-part `MAJOR.MINOR.PATCH` (e.g. `1.4.0`); two-part/empty throws. Admin view must validate three-part semver.

Trust boundary: admin identity must come from `OglasinoAuthentication` in `SecurityContextHolder` (verified Firebase token) via `@PreAuthorize`, never a client flag. The internal floor write does **no** server-side value validation (no semver/range check, no Jakarta constraints) — a new admin endpoint should mirror+exceed it by validating the semver format.

## Known gaps / TODOs

- None. Audit is complete and read-only per brief.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit, no code).
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- **Flag (medium):** `/internal/app/version/floor` (and `/ceiling`) store the version string verbatim with zero server-side validation — no semver parse, no range/monotonicity check, no Jakarta constraints on the DTO records. A malformed floor silently disables the gate for that platform (read path fails open). The new admin floor endpoint should validate semver format server-side and consider rejecting floor > ceiling. File: `src/main/java/com/memento/tech/oglasino/admin/internal/controller/AppVersionAdminController.java`. Not fixed — out of audit scope.
- **Flag (low/medium):** the row records `updatedAt` (when) but no actor and no history table — there is no record of WHICH admin set the floor or any past values. If that matters for a force-update control, it is net-new schema. Cheap to add pre-prod under the Part 12 V1 fold. Spec decides.
- **Observation (architecture):** no service layer — both controllers hit `AppVersionConfigRepository` directly. A new admin floor write sharing validation/monotonicity with the internal write may warrant a small extracted service to avoid duplicating the rule across `/internal` and `/api/secure/admin`. Spec call.
- **Config-file impact:** none required this session. No config-file edit is implied by a read-only audit; the eventual state.md feature row is Mastermind→Docs/QA's to author post-spec.
