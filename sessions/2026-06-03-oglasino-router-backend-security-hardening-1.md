# Session summary

**Repo:** oglasino-router
**Branch:** stage
**Date:** 2026-06-03
**Task:** Read-only Phase-2 audit (backend-security-hardening, edge addendum): produce a per-path edge-reachability forwarding map — for each security-relevant path, whether the worker forwards it to the backend origin, blocks it, handles it, or never routes it — to resize the H1 default-deny re-allow list and decide whether `/actuator/prometheus`+`/actuator/info` need lockdown.

## Implemented

- Read `src/index.ts` end-to-end (274 lines, the entire product) and `wrangler.toml`; established that the worker's forwarding model is **pass-through-everything partitioned by host**, with **no path-prefix allow-list or block-list**.
- Built the deliverable forwarding table for the eight requested path groups. Verdict for every backend path: **FORWARDED to origin verbatim** on the API host (no edge filtering); forwarded to Vercel on the apex host. Only one rewrite exists (`/api/mobile/*` → `/api/*`); `/media` is not served at the edge (no R2 binding on `Env`).
- Confirmed the load-bearing question: `/actuator/*` (incl. `/prometheus`, `/info`) **does** reach the backend through the worker → `/actuator` lockdown is warranted, not moot.
- Confirmed the edge liveness probe hits `BACKEND_ORIGIN/actuator/health/readiness` as an **unauthenticated GET** (no auth header) → H1 default-deny must keep that path `permitAll`.
- Wrote `.agent/audit-router-edge-reachability.md` (Part 1 routing model → Part 2 table → Part 3 probe → Part 4 admin note → Bottom line for H1 + brief-challenge).

## Files touched

- `.agent/audit-router-edge-reachability.md` (new, deliverable — audit report)
- `.agent/2026-06-03-oglasino-router-backend-security-hardening-1.md` (new, this summary)
- `.agent/last-session.md` (overwritten with this summary's content)
- No `src/**`, `tests/**`, `wrangler.toml`, or config changes (read-only audit).

## Tests

- Ran: `npm run lint` (tsc --noEmit) and `npm test` (vitest) as a sanity check only — no source changed.
- Result: lint clean; **47 passed, 0 failed**.
- New tests added: none (read-only audit).

## Cleanup performed

- none needed (no code changed).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change (the audit deliverable feeds Phase-3 seam analysis; no state edit owed from this read-only session)
- issues.md: no change (no new repo-internal bug found; the one open infra question is routed to seam analysis, not issues.md — see For Mastermind)

## Obsoleted by this session

- nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code touched, lint/test green.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one item flagged in "For Mastermind" (origin-bypass DNS question — out of repo scope).
- Part 6 (translations): N/A this session.
- Other parts touched: Part 8 (architectural defaults — "the Cloudflare router worker is the edge boundary" confirmed); Part 11 (trust boundaries — confirmed the edge adds no auth on forwarded paths, Spring is the gate).

## Known gaps / TODOs

- The audit could not settle whether `api-origin[-stage].oglasino.com` is directly internet-reachable bypassing the worker — that is a Cloudflare DNS/origin-lock question, not answerable from the worker source or `wrangler.toml` (routes/DNS are Dashboard-configured per `wrangler.toml:5-9`). Flagged for seam analysis.
- The docker-healthcheck path equality (vs. the edge probe's `/actuator/health/readiness`) must be confirmed in the backend audit — cross-repo, not readable here.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code or abstractions introduced.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **Headline answer (which world are we in):** the worker is **pass-through-everything, partitioned by host** — NOT allow-list-by-prefix. Per the brief's own framing, this means **the backend's Spring-layer authorization is the only gate and every backend audit finding stands at full severity.** The `/actuator` lockdown does **not** drop off the feature.

- **For H1 (resize the re-allow list):**
  1. `/actuator/prometheus` and `/actuator/info` are internet-reachable via the API host → backend lockdown required.
  2. `/internal/**` is **not** edge-blocked — `InternalTokenFilter` is the sole gate; no edge defense-in-depth to lean on.
  3. The edge liveness probe is an **unauthenticated GET to `/actuator/health/readiness`** (`src/index.ts:143-152`, no `headers` arg). H1 **must** keep that path `permitAll` or the mobile backend-availability gate breaks (probe → 401/403 → `backendDown=true` → mobile 503s on a healthy backend). This is in addition to the docker healthcheck — confirm in the backend audit whether they share the path (likely, per the maintenance-split design, but cross-repo so I did not assert it).

- **Adjacent observation (Part 4b):**
  - **Description:** Cannot confirm from this repo whether the backend origin `api-origin[-stage].oglasino.com` is directly reachable bypassing the worker (orange-cloud-locked vs. gray-cloud/DNS-only public origin). The `src/index.ts:3` comment calls it a "gray-cloud origin," which *hints* at a separately-resolvable hostname.
  - **File path:** `wrangler.toml:18,59` (origin vars); `src/index.ts:3` (comment).
  - **Severity guess:** medium (if the origin host is publicly reachable and not IP-allow-listed to Cloudflare, it is a full worker bypass — but this is unverified and may well be locked down).
  - "I did not investigate further because it is out of repo scope — it needs a Cloudflare DNS/origin check in Phase-3 seam analysis, not a worker code read."

- **Config-file impact:** none required. The deliverable feeds Phase-3 seam analysis and resizes the H1 brief; no edit to `conventions.md`/`decisions.md`/`state.md`/`issues.md` is owed from this read-only audit. No unstated config-file dependency.
