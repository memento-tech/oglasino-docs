# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-28
**Task:** Read-only audit of oglasino-backend. Document the exact contracts for the four boot-time concerns the Expo app gates on. No code changes. Output to .agent/audit-expo-boot-redesign.md.

## Implemented

- Read-only audit (no code changes). Produced `.agent/audit-expo-boot-redesign.md` documenting the exact wire contracts of the five boot-time backend endpoints the Expo app gates on: `/api/public/maintenance/active`, `/api/public/app/version/{platform}`, `/api/public/baseSite/overviews`, `/api/public/baseSite/details`, `/api/public/baseSite/{code}`, and `/api/public/versions`.
- For each endpoint, documented method, path, params, headers (with explicit `X-Base-Site` / `X-Lang` posture per `BaseSiteFilter` / `CurrentLanguageFilter`), exact response shape with field types quoted from the DTO/entity code, status codes, trust-boundary posture, and rate-limit posture.
- Surfaced two load-bearing findings that affect the Expo boot redesign: (a) `/api/public/versions` requires `X-Lang` and will 400 at cold boot before a language is known, (b) `/api/public/app/version/{platform}` is shipped as a stub (body commented out, returns empty 200).
- Verified the 22 + 3 checksum keys exposed by `/api/public/versions` against the `TranslationNamespace` enum and the seed rows in `data-configuration.sql`. Documented the lightweight vs heavy base-site distinction explicitly.

## Files touched

- `.agent/audit-expo-boot-redesign.md` (new file, +540 / -0)
- `.agent/2026-05-28-oglasino-backend-expo-boot-redesign-audit-1.md` (new, this file)
- `.agent/last-session.md` (overwrite with exact copy of this file per Part 5)

No source files modified.

## Tests

- Did not run. Read-only audit; no code changes; nothing to verify with the test suite.

## Cleanup performed

- None needed (read-only audit, no code changes).

## Config-file impact

- `conventions.md`: no change.
- `decisions.md`: no change.
- `state.md`: no change. (`version-checksums` already listed as `shipped`; the audit findings about `X-Lang` on `/versions` and the `AppVersionController` stub are routed via "For Mastermind" below, not via `state.md`.)
- `issues.md`: no change drafted from this session. The "For Mastermind" flags below are recommendations to add issues — Mastermind decides whether they go into `issues.md` or get rolled into the next brief, per `conventions.md` Part 4b.

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code touched, no comments/imports/files added beyond the two `.agent/` outputs the workflow requires.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): two adjacent findings flagged in "For Mastermind" (one high-severity, one medium) — both explicitly out of scope for this audit.
- Part 11 (trust boundaries): each of the five endpoints checked explicitly. Roll-up table in `.agent/audit-expo-boot-redesign.md` "Trust-boundary roll-up" section. No violations found.
- Other parts touched: Part 7 (error contract) — documented the `LANG_MISSING_OR_INVALID` envelope shape returned by `CurrentLanguageFilter`. Part 8 (architectural defaults) — referenced the worker boundary in the maintenance answer.

## Known gaps / TODOs

- None. The brief asked for documentation; the documentation is complete. The two surfaced findings (X-Lang on `/versions`, app-version stub) are explicitly flagged for Mastermind triage, not left as TODOs.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing. Read-only audit; no abstractions, configs, or patterns introduced.
  - Considered and rejected: nothing. The audit follows the brief's structure verbatim; no shape decisions to make.
  - Simplified or removed: nothing.

- **High-severity finding — `/api/public/versions` requires `X-Lang`.** `CurrentLanguageFilter.ALLOWLIST_PREFIXES` covers `/api/public/baseSite/`, `/api/public/maintenance/`, `/api/public/app/version/`, and `ALLOWLIST_EXACT` covers `/api/public/config`, `/api/public/translations`, `/api/public/health/check`, `/api/public/verify-recaptcha`. `/api/public/versions` is on neither list, so the filter returns `400 LANG_MISSING_OR_INVALID` when called without a resolvable `X-Lang`. The endpoint's response body is language-independent — the filter requirement is incidental. The Expo boot redesign cannot rely on `/versions` at cold boot before a language is known unless one of: (a) backend adds `/api/public/versions` to `ALLOWLIST_EXACT` (one-line change), or (b) the app sequences `/versions` after a default `X-Lang` is established. Recommend (a). The `features/version-checksums.md` spec implies the endpoint is always-200; adding it to the allowlist matches the documented contract.

- **High-severity finding — `AppVersionController` is a stub.** `controller/AppVersionController.java:23` returns `ResponseEntity.ok().build()` (empty body); the actual decision logic that would populate `AppVersionResponseDTO(latestVersion, minSupportedVersion, forceUpdate, optionalUpdate)` is commented out at lines 25-33. The DTO, the entity, the repository, and the admin POST endpoint at `/internal/app/version/update` all exist and are wired. Any Expo boot logic written against the documented `AppVersionResponseDTO` shape will receive an empty body and either fail JSON parsing or misinterpret as "no update required." Uncommenting the controller body would restore the contract, but that's a code change explicitly out of audit scope. If the boot-redesign feature needs a working soft/hard-update gate, this must be picked up as its own backend brief.

- **Adjacent observation (medium) — `BaseSite not found` throws raw `RuntimeException` at `DefaultBaseSiteService.java:77-78`.** Surfaces as 500 instead of 404 for unknown codes on `/api/public/baseSite/{code}`. Not blocking for the boot redesign because the app's selected `code` always comes from `/overviews` and is valid by construction. Pre-existing; flagging for Part 4b.

- **Adjacent observation (low) — `AppVersionController` lives under package `admin.internal.controller` but is mapped to `/api/public/app/version`.** Cosmetic packaging mismatch. Easy to miss when grepping for public endpoints by package.

- **Config-file drafts pending:** none. This session's findings go to Mastermind for routing; nothing is drafted for direct application to `conventions.md` / `decisions.md` / `state.md` / `issues.md`.
