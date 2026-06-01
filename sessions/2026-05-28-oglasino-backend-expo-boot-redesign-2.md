# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-28
**Task:** Make GET /api/public/app/version/{platform} return a real, computed AppVersionResponseDTO in every environment. Uncomment and wire the version logic, seed dev to a no-op, and split the version write surface into floor-only and ceiling-only paths. (Phase 1 of B2; EAS post-build hook out of scope.)

## Implemented

- **Read path (`GET /api/public/app/version/{platform}`).** Replaced the empty `200 OK` stub in `AppVersionController` with the Part 3 classification rule (`forceUpdate = running < floor`; `optionalUpdate = running < ceiling && !forceUpdate`) using the existing `Version.parse(...).isLowerThan(...)` helper. Switched to constructor injection.
- **Fail-open handling.** Unknown platform returns `AppVersionResponseDTO("", "", false, false)` with a WARN log. Unparseable stored semver (`floor` or `ceiling`, including `null`) returns the same no-update DTO with a WARN log — the `Version.parse` calls are wrapped so no parse failure can 500 the gate. Spring's existing required-param `400` on missing `currentVersion` is preserved (no custom handling).
- **Split write surface.** `AppVersionAdminController` now exposes two single-purpose endpoints with constructor injection:
  - `POST /internal/app/version/ceiling` accepting `AppVersionCeilingUpdateRequest{platform, latestVersion}` — writes only `latestVersion`.
  - `POST /internal/app/version/floor` accepting `AppVersionFloorUpdateRequest{platform, minSupportedVersion}` — writes only `minSupportedVersion`.
  - Both are UPDATE-ONLY: a missing platform row returns `404 Not Found` with empty body and does not save. Row creation is owned by the seed.
  - The old `POST /internal/app/version/update`, its `AppVersionUpdateRequest` POJO, and the dead GitHub-Actions `curl` comment block were deleted.
- **Per-platform dev seed.** New `data/configuration/data-app-version-config.sql` seeds `android` and `ios` rows with `minSupportedVersion='0.0.0'` and `latestVersion='0.0.0'`, IDs 1 and 2 (under the `global_seq` start at 1000), `ON CONFLICT (id) DO NOTHING`. Safe-default-in-all-envs pattern (mirrors `configuration.maintenance.active='false'`); operator/EAS-set values are preserved across reboots. No `if (dev)` in code — the difference is data only.
- **Schema constraint (adjacent finding pulled into scope per the brief).** Added `UNIQUE (platform)` on `app_version_config` in `V1__init_schema.sql` (pre-prod V1 schema fold per Part 12). The split-write design and `findByPlatform` both rely on one row per platform; a duplicate would surface as `IncorrectResultSizeDataAccessException` → 500 → maintenance lockout in the Expo app per the Part 2 backend-down rule.
- **Trust boundary (Part 11) confirmed:** `currentVersion` is client-supplied but only feeds a comparison against server-stored floor/ceiling — a client lying about its own version only fools itself. The two write paths sit under `/internal/**`, gated by `InternalTokenFilter` (`X-INTERNAL-TOKEN` shared secret, `401` otherwise) — unchanged. Neither write path accepts a client value that becomes a trust decision; the values they carry **are** the values stored.

## Files touched

- src/main/java/com/memento/tech/oglasino/admin/internal/controller/AppVersionController.java (+45 / -13)
- src/main/java/com/memento/tech/oglasino/admin/internal/controller/AppVersionAdminController.java (rewritten; net +20 / -29)
- src/main/java/com/memento/tech/oglasino/admin/internal/dto/AppVersionCeilingUpdateRequest.java (new, +3)
- src/main/java/com/memento/tech/oglasino/admin/internal/dto/AppVersionFloorUpdateRequest.java (new, +3)
- src/main/java/com/memento/tech/oglasino/admin/internal/dto/AppVersionUpdateRequest.java (deleted, -31)
- src/main/resources/db/migration/V1__init_schema.sql (+8 / -0)
- src/main/resources/data/configuration/data-app-version-config.sql (new, +10)
- src/test/java/com/memento/tech/oglasino/admin/internal/controller/AppVersionControllerTest.java (new, +132)
- src/test/java/com/memento/tech/oglasino/admin/internal/controller/AppVersionAdminControllerTest.java (new, +117)

## Tests

- Ran: `./mvnw test -Dtest='AppVersionControllerTest,AppVersionAdminControllerTest'` → 12 passed, 0 failed.
- Ran: `./mvnw test` → 678 passed, 0 failed, 0 skipped (was 666 before this session; +12 new).
- Ran: `./mvnw spotless:check` → green (after one `spotless:apply` to format generated test files).
- New tests added:
  - `AppVersionControllerTest` (8): `forceUpdate_whenRunningBelowFloor`, `optionalUpdate_whenRunningAtOrAboveFloorButBelowCeiling`, `neither_whenRunningAtOrAboveCeiling`, `devSeed_neverFiresEitherFlag` (0.0.0/0.0.0 contract), `unknownPlatform_returnsNoUpdateDtoNot500`, `unparseableStoredFloor_returnsNoUpdateDtoNot500`, `unparseableStoredCeiling_returnsNoUpdateDtoNot500`, `nullStoredFields_returnsNoUpdateDtoNot500`.
  - `AppVersionAdminControllerTest` (4): `ceilingWrite_leavesFloorUnchanged`, `floorWrite_leavesCeilingUnchanged`, `ceilingWrite_missingPlatform_returns404AndDoesNotSave`, `floorWrite_missingPlatform_returns404AndDoesNotSave`.

## Cleanup performed

- Deleted `AppVersionUpdateRequest.java` (obsoleted by the split).
- Deleted the multi-line GitHub-Actions `curl` comment block at the bottom of `AppVersionAdminController.java` (documented the deleted `/update` endpoint).
- Switched `AppVersionController` and `AppVersionAdminController` from `@Autowired` field injection to constructor injection. Only these two — no broader sweep.

## Config-file impact

- conventions.md: no change
- decisions.md: no change (Mastermind will likely want a B2 wrap-up entry at feature close, drafted in "For Mastermind" below — not a blocker for this session)
- state.md: no change in this session; eventual Expo-boot-redesign feature row update will follow when the full feature ships (out of scope here)
- issues.md: no change

## Obsoleted by this session

- `POST /internal/app/version/update` endpoint and its `AppVersionUpdateRequest` DTO — deleted in this session.
- The GitHub-Actions `curl` comment block at the bottom of `AppVersionAdminController.java` — deleted in this session.
- The "AppVersionController shipped as a stub" / "high-severity" flag from `audit-expo-boot-redesign.md` §2 — closed by uncommenting `checkVersion`. The audit's other adjacent observation about `admin.internal.controller` package vs `/api/public` mapping remains open and out of scope per the brief.

## Conventions check

- Part 4 (cleanliness): confirmed — no commented-out code left; no unused imports/files; no `System.out.println`; no `TODO`/`FIXME` added; obsoleted DTO and dead comment block deleted in the same session.
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): confirmed — the brief explicitly pulled three adjacent observations into scope (unique constraint, constructor injection on the two controllers, dead comment block); all three handled. Nothing else surfaced.
- Part 6 (translations): N/A this session — no translation keys added or removed. (See "For Mastermind" note on the 404 envelope decision.)
- Part 7 (error contract): confirmed for the read path (returns the typed `AppVersionResponseDTO` shape, never 500). For the write paths' 404, see "For Mastermind" — chose bare `ResponseEntity.notFound().build()` to match the `InternalTokenFilter` bare-401 precedent on /internal/**, rather than introducing a new error code + translation seed for a machine-to-machine path with no UI consumer today.
- Part 11 (trust boundaries): confirmed — see "Implemented" section.
- Part 12 (schema patterns): confirmed — `UNIQUE (platform)` added by editing `V1__init_schema.sql` in place (pre-prod V1 schema fold).

## Known gaps / TODOs

- None for this brief. The EAS post-build hook (B2 step 4) is the next session per the original brief; this session leaves the ceiling-write endpoint ready for it.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity):
    - Two single-field DTO records (`AppVersionCeilingUpdateRequest`, `AppVersionFloorUpdateRequest`) — the wire shape itself enforces "ceiling writer cannot lower the floor; floor writer cannot un-set the ceiling." A single nullable-partial DTO would have pushed that invariant into runtime code rather than the contract. Two records, one line each.
    - A `NO_UPDATE` static `AppVersionResponseDTO` constant inside `AppVersionController` — reused by both fail-open paths (unknown platform, parse failure). One named constant beats two literal instantiations with the same semantics.
    - WARN logging on both fail-open paths so a misconfigured platform / corrupted stored version surfaces in logs instead of silently no-op'ing forever.
  - Considered and rejected:
    - **Introducing an `AppVersionErrorCode` enum + `AppVersionValidationException` family for the 404.** Rejected — one code for one machine-to-machine endpoint is the canonical "no foreseeable second caller" case (Part 4a). Used bare `ResponseEntity.notFound().build()` instead, matching the `InternalTokenFilter` bare-401 precedent on `/internal/**`.
    - **Adding a `PLATFORM_NOT_FOUND` constant to `ProductErrorCode`** with the unified `{errors:[{field,code,translationKey}]}` envelope. Rejected — emitting a translation key (`product.system.platform_not_found`) that isn't seeded in the ERRORS namespace would be a wire-contract dangler; the only consumers (EAS hook, admin panel curl) consume status code, not envelope. Bare 404 is honest about the surface.
    - **Jakarta `@NotBlank` validation on the two request DTOs.** Rejected — the brief didn't request it, the InternalTokenFilter already gates access, and adding validation just to add it is the kind of speculative defense Part 4a discourages. If a future caller sends a null `platform`, the `findByPlatform(null)` lookup returns empty and we 404 — same observable behavior.
    - **An `if (dev)` branch in the controller for dev/no-op behavior.** Rejected per the spec — the difference lives in seeded data, not code. The 0.0.0/0.0.0 seed gives the same no-op behavior with one less branch.
    - **Echoing the (possibly garbage) stored version strings back in the parse-failure response** rather than empty strings. Rejected — `NO_UPDATE` is a single deterministic shape; the operator diagnoses by reading the WARN log, not the wire response.
  - Simplified or removed:
    - Deleted `AppVersionUpdateRequest.java` (one-purpose DTO that's now structurally wrong; two single-field DTOs replace it).
    - Deleted the multi-line GitHub-Actions comment block at `AppVersionAdminController.java:39-49` documenting the deleted `/update` endpoint.
    - Removed `@Autowired` field injection from both controllers (constructor injection is the surrounding style, e.g. `VersionController`).

- **Draft `decisions.md` entry (for feature close, when B2 + B2-hook are both shipped).** Mastermind may want to merge with the B2-hook session's note rather than drop two entries.

  > ### YYYY-MM-DD — App-version gate uncommented; floor/ceiling write surfaces split per platform
  >
  > `AppVersionController.checkVersion` no longer returns an empty 200; it computes `AppVersionResponseDTO` per the spec rule (`forceUpdate = running < floor`, `optionalUpdate = running < ceiling && !forceUpdate`) using `Version.parse(...).isLowerThan(...)`. The dev/prod difference is data, not code: per-platform `AppVersionConfig` rows are seeded with `0.0.0/0.0.0`, which makes the gate a real-contract no-op in every environment until an operator raises the floor or the EAS post-build hook raises the ceiling. The single `POST /internal/app/version/update` endpoint that wrote both fields in one payload was split into `POST /internal/app/version/ceiling` (machine-driven, EAS hook target) and `POST /internal/app/version/floor` (human-driven, admin target), each with a single-field DTO so the wire shape itself enforces that neither writer can touch the other field. Both write paths are UPDATE-ONLY — a missing platform row returns 404 and does not create; row creation is owned by the seed. `UNIQUE (platform)` added to `app_version_config` in `V1__init_schema.sql` (pre-prod V1 schema fold) so duplicate rows can't silently cause an `IncorrectResultSizeDataAccessException` 500 → maintenance lockout in the Expo gate. Unknown platforms and unparseable stored semver both fail open to the no-update DTO with a WARN log, never 500 — the gate is incapable of bricking the app on a misconfig.

- **404 envelope decision worth a sanity check.** The brief said "use the project's standard error envelope / not-found path." I read the slash as "envelope OR not-found path." The unified `{errors:[{field,code,translationKey}]}` envelope is the standard for user-facing surfaces; `/internal/**` is machine-to-machine (the existing `InternalTokenFilter` returns a bare `401` with no body). I went with `ResponseEntity.notFound().build()` (bare 404) for the same reason: the consumers are the EAS post-build hook (CI) and a future admin panel (which can read the status code). If you want the unified envelope here too, the diff is a new `PLATFORM_NOT_FOUND` constant on `ProductErrorCode` + matching translation seed in the ERRORS namespace (4 rows: EN/RS/RU/CNR per Part 6 Rule 3) + a 2-line change to the two write methods. Flag if you want me to switch.

- **Dev seed is in `data/configuration/*.sql` and runs in every environment.** This is deliberate per Item 3 of the go-ahead: the 0.0.0/0.0.0 seed is the safe default everywhere; `ON CONFLICT (id) DO NOTHING` means subsequent operator/EAS writes survive boots. Worth surfacing once for the record: the `data/configuration/` folder is loaded in prod and stage too (per `application-prod.yaml:22-33`), so on a fresh prod DB the rows will exist with 0.0.0/0.0.0 floor and ceiling on first boot. The gate returns "you're fine" until the EAS hook (B2-hook session) sets a real ceiling and admin raises the floor. Operators need to know this to avoid being surprised that a freshly-deployed prod backend reports no pending updates for an installed app.

- **No drafts pending for `conventions.md`, `state.md`, `issues.md`.**
