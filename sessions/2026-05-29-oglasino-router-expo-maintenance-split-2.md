# Session summary

**Repo:** oglasino-router
**Branch:** stage
**Date:** 2026-05-29
**Task:** Implement the maintenance-split worker changes (six changes per `.agent/brief.md`, spec `../oglasino-docs/features/expo-maintenance-split.md` §2/§3/§8): flag split, fail-open fix, mobile classification + routing, probe target change, web composition, matrix comment — plus tests.

## Implemented

- **Flag split (change 1):** Replaced the single `maintenance.active` KV read with `maintenance.web.active` + `maintenance.backend.active`, both `{ cacheTtl: 30 }`. `admin.bypass.disabled` and `use.backend.check` unchanged by name. Strict `=== "true"`; absent/null = false = up.
- **Fail-open (change 2):** All four reads sit in one `Promise.all` inside the existing fail-open try/catch; the catch resets all four flags to false. NOTE: `use.backend.check` was *already* inside the try/catch on disk (fixed 2026-05-20 per issues.md), contradicting the brief's "currently sits outside it" framing — see "For Mastermind / Brief vs reality." The brief's desired end-state was therefore already met; this session preserved it and added the test coverage the brief asked for.
- **Mobile classification + routing (change 3):** `isMobile = path.startsWith("/api/mobile/")` (path-based, host-agnostic — mobile arrives on apex in prod, API host in stage). Mobile branch: `backendDown = maintenance.backend.active OR probeFailed`; if down → API-style 503 maintenance response; else strip the `/mobile` segment (`/api/mobile/<rest>` → `/api/<rest>`) and forward to `BACKEND_ORIGIN`. The admin bypass is never consulted on the mobile path (branch returns before the web gate).
- **Probe target (change 4):** Re-pointed from `/health` to `/actuator/health/readiness`, edge-cache opts (`cacheTtl: 30, cacheEverything: true`) preserved. The probe now lives *inside* the mobile branch (gates mobile only), gated by `use.backend.check`, short-circuited when `backend.active` is already true.
- **Web composition (change 5):** `webDown = maintenance.web.active OR maintenance.backend.active`; bypass mechanics unchanged (`shouldBlock = adminBypassDisabled || !isAdminRequest`).
- **Matrix comment (change 6):** Rewrote the top-of-file block to document the two dependency flags, per-client composition (web = web OR backend; mobile = backend OR probe), the `/api/mobile/*` label + segment-stripping, the probe target, and `use.backend.check`.

## Files touched

- `src/index.ts` (+95 / -34)
- `tests/router.test.ts` (+291 / -51)

## Tests

- Ran: `npm run lint` (tsc --noEmit) → clean, 0 errors.
- Ran: `npm test` (vitest run) → 45 passed, 0 failed (`tests/router.test.ts`).
- New / reworked tests: two-flag web matrix (web-only, backend-only, both-off, admin-bypass); full-lockdown extended to `backend.active`; a 10-test mobile-path block (backend.active→503, probe-fail→503, probe-throw→503, both-clear→stripped forward, no-probe→stripped forward, ignores web.active [core decoupling], ignores admin.bypass.disabled, probe target+cacheTtl asserted, probe short-circuit, mobile on API host); fail-open extended to all four reads incl. `use.backend.check` for both web and mobile.
- After an adversarial review workflow flagged the core-decoupling test as weak (status-only assertion), strengthened it to assert the stripped `BACKEND_ORIGIN` forward URL and that the probe ran. Re-ran: still 45/45 green.

## Cleanup performed

- Removed the old single-flag gate, the standalone probe block (`if (!maintenanceActive && useBackendCheck)`), and the `maintenance.active` matrix comment — all replaced, none left dangling.
- No commented-out code, no debug logging, no TODO/FIXME added.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change. (The "Expo Maintenance Split" entry is `planned`; a status flip to reflect router code-complete is Mastermind/Docs-QA territory, not this session — drafted note in "For Mastermind.")
- issues.md: no change. The two relevant entries (2026-05-15 `use.backend.check` fail-open; 2026-05-15 matrix-comment-missing-`use.backend.check`) are already marked `fixed` (2026-05-20); this session does not reopen or amend them.

## Obsoleted by this session

- The single `maintenance.active` flag concept in the worker — deleted this session (replaced by the two dependency flags). The KV key itself is removed from *this* repo's code; per spec §7, the live `maintenance.active` KV row and the backend config row are removed in later backend phases (not this repo, not this session).
- The `/health` probe target — deleted this session, replaced by `/actuator/health/readiness`.
- Pre-split tests keyed on `maintenance.active` and on the probe-forces-web-maintenance behavior — rewritten this session, not left stale.

## Conventions check

- Part 4 (cleanliness): confirmed — lint + tests green, no dead code, no debug logging, no stray files.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one observation flagged in "For Mastermind".
- Part 6 (translations): N/A this session (no user-facing strings touched; the `MAINTENANCE_JSON` body is unchanged).
- Other parts touched: Part 8 (architectural defaults — "the Cloudflare router worker is the edge boundary; maintenance state … live there") — confirmed, this change keeps maintenance composition at the edge. Part 10 Phase 5 (engineering brief execution) — this is the router Phase-5 brief.

## Known gaps / TODOs

- None deferred in code. The end-to-end rehearsal (spec §9: flip `backend.active` → preview mobile shows maintenance; flip `web.active` only → mobile unaffected) is a cross-repo deploy/QA step owned by Igor, gated on the stage worker deploy (spec §7 step 1) — not an engineer-agent task and not blockable from this repo.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): one new path predicate `isMobile` and one mobile branch — both required by the brief (change 3), single concrete use, no abstraction introduced. The `/mobile` strip is inlined (`"/api" + path.slice("/api/mobile".length)`), not extracted to a helper, since it has exactly one call site — matching the repo's "keep helpers inline if used once" rule.
  - Considered and rejected: (a) a shared `maintenanceResponse`-style helper for the probe — rejected, the probe is one call site; (b) routing mobile through the existing web gate with an `isMobile` exclusion flag — rejected, a dedicated early-return branch is clearer and guarantees the admin bypass is never consulted for mobile; (c) keeping the probe outside the mobile branch (as before) and reading its result in both paths — rejected, web composition no longer uses the probe (spec §3.3), so probing on web requests would be wasted backend load.
  - Simplified or removed: collapsed the old two-step `maintenanceActive` gate + standalone probe block into a single per-client composition; the probe now runs only where its result is used (mobile).

- **Brief vs reality:**
  1. **Change 2 ("FAIL-OPEN FIX") was already done.**
     - Brief says: `use.backend.check` "is read OUTSIDE it (~91-93) — a pre-existing bug" and asks me to move it inside.
     - Code says / I observed: on disk (pre-session `src/index.ts:84-91`) all three reads were already inside one `Promise.all` in the fail-open try/catch. `issues.md` 2026-05-15 entry is marked `fixed (2026-05-20, session oglasino-router-use-backend-check-fail-open-1)`.
     - Why this matters: the brief's CONTEXT treats the bug as live; it was fixed nine days ago. The change-2 deliverable (use.backend.check inside fail-open) was already satisfied, so I preserved it and added the test coverage the brief requested. No behavior change was needed for change 2.
     - Recommended resolution: none required for the code; Mastermind may want to note that the Phase-2 audit/brief framing lagged the 2026-05-20 fix.

- **Behavior change worth surfacing (intended per spec, not a discrepancy):** Previously the liveness probe forced maintenance for *everyone* (web included). Per spec §3.3, the probe now gates *mobile only* — web maintenance is purely `web.active OR backend.active` (no probe input). So if the backend crashes without an operator setting `backend.active`, mobile shows maintenance but web will still forward to Vercel (and its API calls would fail at origin). This is the spec's explicit design (web on Vercel can serve many routes without the backend; gating web on the probe would over-trigger). Implemented as specified; flagging so it's a conscious platform decision, not a surprise.

- **Adjacent observation (Part 4b):** The frontend maintenance HTML response (`maintenanceResponse` non-API branch, `src/index.ts` ~217-227) does not set `X-Robots-Tag: noindex` on stage — only `forwardToOrigin` sets it. So a stage maintenance page is served without the noindex header. File: `oglasino-router/src/index.ts` (maintenanceResponse HTML branch). Severity: low (transient maintenance page, brief window, stage only). Confirmed via `git show HEAD:src/index.ts` to be **pre-existing, not a regression** from this change, and **out of scope** (brief OUT OF SCOPE: "The frontend-path maintenance response … is unchanged"). I did not fix this because it is out of scope.

- **Suggested state.md note (draft for Docs/QA, not applied):** The "Expo Maintenance Split" entry (`state.md`) is `planned`. Router code is now complete on `stage` (uncommitted, awaiting Igor's commit + deploy per spec §7 step 1–2). If Mastermind wants the pipeline to reflect it, a note like *"Router: two-flag composition + `/api/mobile/*` routing + probe re-point implemented on `stage` (oglasino-router-expo-maintenance-split-2), pending Igor commit/deploy; backend/expo/web phases not started."* — drafted here for Docs/QA to apply; not written by this agent.

- **Verification note:** Ran an adversarial multi-dimension review workflow (spec-conformance, care-areas, test-rigor) over the diff. 12 of 13 confirmed findings were positive ("no regression" / "present and strong"); the one actionable finding (weak core-decoupling test assertion) was fixed this session. No correctness bug in `src/index.ts` was found.

- **Pre-existing uncommitted state:** `CLAUDE.md` (modified) and `start-session.sh` (untracked) were present at session start per the brief's note; I did not touch or revert them.
