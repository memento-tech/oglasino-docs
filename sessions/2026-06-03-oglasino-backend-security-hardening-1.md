# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-06-03
**Task:** Audit Brief — backend-security-hardening: revalidation. READ-ONLY revalidation of the 2026-06-03 backend security audit's claims against current code; produce the Phase-4 spec input.

## Implemented

- Read-only Phase-2 audit. No code changed. Single deliverable written: `.agent/audit-backend-security-hardening.md`.
- Revalidated all 7 Parts + sub-items against the actual code, each `file:line` opened with Read **and** cross-checked with an independent grep (standing tool-reliability rule). No phantom-content disagreement encountered.
- Produced the complete inventory of HTTP paths outside the four known prefixes (the brief's primary deliverable): the only non-prefixed application controller is `GET /health`; the rest of the non-prefixed surface is framework (`/actuator/*`, `/error`, OPTIONS preflight). `/media/**` has no backend handler at all.
- Identified the boot-loop constraint precisely: prod + stage docker-compose healthchecks hit container-internal `http://localhost:8080/actuator/health/readiness` (`infra/docker-compose.yml:93`, `infra/docker-compose-stage.yml:108-114`) — `/actuator/health/**` and `/error` must stay permitAll after any default-deny flip.
- Recalibrated four over-stated prior-audit severities (Parts 2, 3, 6, M5) and refined two mechanisms (Part 4 registration, L3 config row), with quoted evidence and a "Corrections to the prior audit" section.

## Files touched

- `.agent/audit-backend-security-hardening.md` (new, +~290 lines) — the audit deliverable.
- `.agent/2026-06-03-oglasino-backend-security-hardening-1.md` (new) — this summary.
- `.agent/last-session.md` (overwritten) — exact copy of this summary.

No source files modified (read-only audit).

## Tests

- Not run. Read-only audit, no code change, so `./mvnw test` / `spotless:check` were out of scope and not invoked.

## Cleanup performed

- None needed (no code touched).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change. (Findings may seed a decisions/issues entry at Igor's discretion via Mastermind→Docs/QA; nothing drafted-and-pending from this session.)
- state.md: no change.
- issues.md: no change. (The audit cross-referenced the existing 2026-06-03 double-registration and CloudflareKv entries and confirmed the double-registration one is the correct reading vs the security audit; no new issues authored here — they route through Mastermind/Docs/QA.)

## Obsoleted by this session

- Nothing. (The audit refutes/recalibrates claims in the upstream 2026-06-03 security report, but that report is Phase-1 input material, not a tracked repo artifact this session can delete.)

## Conventions check

- Part 4 (cleanliness): confirmed — no code added, nothing to clean.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): flagged — see "For Mastermind" (TestCreateJSON dev controller in main; CORS UPDATE/missing-PATCH; SuggestionController `/api` class mapping fragility). All also captured in the audit's adjacent-observations section.
- Part 6 (translations): N/A this session.
- Part 7 (error contract): confirmed observed in passing — `/error` is the Spring default dispatch; no custom ErrorController. No change made.
- Part 11 (trust boundaries): central to the audit — auth identity confirmed server-derived; Part 6 sort `field` flagged as a (low-blast-radius) client-trust smell; M1 removal-safety seam analyzed.

## Known gaps / TODOs

- **Part 5 reachability is DB-dependent and unverified.** Whether the live `admin@oglasino.com` row has `subscription_active=false`/`subscription_type=null` (which would make the identity-role lockout live, not theoretical) requires reading the prod admin row. Read-only + no prod DB access this session — I did not query it. The code-level coupling is confirmed regardless.
- **L5 prod CORS origins unverified.** `ALLOWED_CORS` is an env var, not in the repo. Confirm it holds exact `oglasino.com` origins (not broad patterns) given `allowCredentials:true`.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (no code).
  - Considered and rejected: considered delegating the controller-path sweep to a sub-agent for speed; rejected because the brief's tool-reliability rule demands first-party Read+grep cross-checking of every cited line, which a sub-agent's summarized output would undercut.
  - Simplified or removed: nothing (no code).

- **Headline for the Phase-4 spec:** the H1 default-deny flip is safe *if and only if* it explicitly re-allows `/actuator/health/**` (container healthcheck + edge liveness probe), `/error`, and OPTIONS preflight. The complete non-prefixed surface to re-allow is small (see audit Part 1b) — `/health` is the only application controller in it and is arguably redundant with `/actuator/health`.

- **Severity recalibrations for the spec (don't carry the prior audit's severities forward):** Part 3 = cleanliness not perf (OncePerRequestFilter dedupes); Part 6 = ES-sort defense-in-depth not SQLi; Part 2 = latent re-arming trap not live hole; M5 = unnecessary prod jar not open console. Details + quotes in the audit's "Corrections" section.

- **Cross-repo seam needing your analysis (M1):** removing `InputSanitizationFilter`/`Sanitizer` (HTML-encoding at input is the wrong layer and corrupts non-HTML consumers) is safe only after confirming the **email** render path (Brevo SMTP bodies) escapes at its own step — web/mobile already escape on render. This touches web/mobile render behavior; flagging rather than calling it unilaterally.

- **Adjacent observations (Part 4b), severity guesses:**
  - `controller/test/TestCreateJSON.java` — dev-only (`@Profile("dev")`) test controller in `src/main`. Low. Not fixed (out of scope, read-only).
  - `SecurityConfig.java:61-62` CORS lists non-existent `"UPDATE"` and omits `PATCH`. Low (would bite a future `@PatchMapping`). Not fixed.
  - `SuggestionController.java:19` class-maps `/api` and depends on method paths landing inside `/public`/`/secure`. Low/medium under default-deny. Not fixed.
  - `WHITE_LIST_URL` (`SecurityConfig:26-35`) is dead and references absent springdoc + disabled h2-console. Low. Not fixed (delete candidate for the H1 brief).

- **No drafted config-file text from this session.** Read-only; any issues.md/decisions.md entries arising from these findings are yours to draft and Docs/QA's to apply.
