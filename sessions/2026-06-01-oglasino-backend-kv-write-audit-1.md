# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-06-01
**Task:** AUDIT (read-only) — Does the backend retain runtime KV-write capability post-maintenance-split? (issues.md adjacent finding, 2026-06-01 bug chat, bug 4 — UNCONFIRMED)

READ-ONLY. No code or docs changed. Nothing staged.

---

## TL;DR / Verdict

**Verdict = (a).** The backend **does** perform runtime Cloudflare KV **writes** today. The admin maintenance "wrench" issues two HTTP `PUT`s to the Cloudflare KV API at request time, authenticated with the `CLOUDFLARE_KV_CONFIG_TOKEN`. The docs write-scope wording at `11-environment-variables.md:136` and `06-architecture.md` is therefore **accurate as to write capability — nothing to fix on the write-scope claim.**

The brief's framing — "after the maintenance split, maintenance KV writes are CI/droplet-script driven (edge-only)" — is **incomplete/refuted by the code**. CI/droplet scripts do flip KV for deploy windows, but that is *in addition to* a live, request-triggered backend KV-write path that still exists. See "Brief vs reality" below.

One secondary doc-accuracy nuance (the `06` "optionally **read** by the backend" phrasing under-describes the write path, and the cited line number is off) is noted in §4 and "For Mastermind" — it is not the write-scope staleness the brief was hunting, so I flag rather than fix.

---

## Findings (answers to brief Q1–Q5, grep-verified)

### Q1 — Does the backend make ANY Cloudflare KV write at runtime?

**Yes.** One runtime KV-write path exists:

- `src/main/java/com/memento/tech/oglasino/service/impl/DefaultCloudflareKvService.java:69-76` — `setMaintenanceCloudflareValue(String key, boolean value)` builds the Cloudflare KV value URL and calls `restTemplate.exchange(buildUrl(key), HttpMethod.PUT, request, String.class)` — an HTTP **PUT** with body `Boolean.toString(value)` and `Content-Type: text/plain`.
- It is invoked from `DefaultCloudflareKvService.java:43-44` inside `toggleMaintenance()`:
  - `:43` `setMaintenanceCloudflareValue(MAINTENANCE_WEB_KEY, next);`
  - `:44` `setMaintenanceCloudflareValue(MAINTENANCE_BACKEND_KEY, next);`
- The endpoint template is `DefaultCloudflareKvService.java:20-21`:
  `"https://api.cloudflare.com/client/v4/accounts/%s/storage/kv/namespaces/%s/values/%s"` — the Cloudflare KV `/values/{key}` write endpoint.

Reachability (request-triggered, not autonomous):

- `src/main/java/com/memento/tech/oglasino/admin/controller/MaintenanceAdminController.java:24-27` — `@PostMapping("/toggle")` → `cloudflareKvService.toggleMaintenance()`.
- Route: `POST /api/secure/admin/maintenance/toggle` (`MaintenanceAdminController.java:13` `@RequestMapping("/api/secure/admin/maintenance")` + `:24`).
- Gate: `@PreAuthorize("hasRole('ADMIN')")` at `MaintenanceAdminController.java:14` (class level). **Admin-only.** (Note: this is stricter than the *unauthenticated* `POST /api/public/maintenance/toggle` that the 2026-05-29 `audit-expo-maintenance-split.md` §5 described — the toggle has since been moved under `/api/secure/admin` and re-gated. The write path itself still exists; only its mount point and auth changed.)

`grep -rn "CloudflareKvService\|toggleMaintenance\|isMaintenanceActive\|setMaintenanceCloudflareValue" src/main` confirms `MaintenanceAdminController` is the **only** caller of the service in `src/main`. No other runtime write path exists.

### Q2 — Reads (GET) or writes (PUT/DELETE)? Which keys?

**Both reads and writes. No DELETE.**

- **Write (PUT)** — `DefaultCloudflareKvService.java:75`:
  `restTemplate.exchange(buildUrl(key), HttpMethod.PUT, request, String.class);`
  Keys written: both, every toggle:
  - `MAINTENANCE_WEB_KEY = "maintenance.web.active"` (`:15`)
  - `MAINTENANCE_BACKEND_KEY = "maintenance.backend.active"` (`:16`)
  Body is literal `"true"` or `"false"`. `toggleMaintenance()` (`:35-47`) reads both flags, computes `next = !(webActive || backendActive)`, then PUTs `next` to **both** keys (the "platform maintenance, everything down" semantics noted in the `:13-14` comment).
- **Read (GET)** — `DefaultCloudflareKvService.java:56-67`, `readMaintenanceFlag(String key)`:
  `restTemplate.exchange(buildUrl(key), HttpMethod.GET, new HttpEntity<>(authHeaders()), String.class)` — same two keys. Called by `isMaintenanceActive()` (`:50-54`, served by `GET /api/secure/admin/maintenance`) and by `toggleMaintenance()` to read current state before flipping.
- **No DELETE** anywhere — `grep` for `HttpMethod.DELETE` / `DELETE` in the file: none. Absent keys are treated as off (`:64` `HttpClientErrorException.NotFound → false`).

### Q3 — Where is `CLOUDFLARE_KV_CONFIG_TOKEN` actually consumed in code (not docs)?

It is consumed for **both the read and the write** path (it is the bearer token on every KV call this service makes):

- Property binding: `DefaultCloudflareKvService.java:28-29`
  `@Value("${cloudflare.config.token}") private String TOKEN_ID;`
- Config wiring (all three profiles map `cloudflare.config.token` → `CLOUDFLARE_KV_CONFIG_TOKEN`):
  - `src/main/resources/application-prod.yaml:147-148` (`config: token: ${CLOUDFLARE_KV_CONFIG_TOKEN}`)
  - `src/main/resources/application-stage.yaml:158`
  - `src/main/resources/application-dev.yaml:87`
- Used in `authHeaders()` (`DefaultCloudflareKvService.java:82-86`): `headers.setBearerAuth(TOKEN_ID);`
- `authHeaders()` is called by **both** `readMaintenanceFlag` (GET, `:60`) **and** `setMaintenanceCloudflareValue` (PUT, `:70`).

Companion KV props consumed by the same service:
- `cloudflare.config.namespace-id` → `CLOUDFLARE_NAMESPACE_ID` (`DefaultCloudflareKvService.java:31-32`; yaml prod `:149`, stage `:159`, dev `:88`) — used in `buildUrl` (`:78-80`).
- `cloudflare.account.id` → `CLOUDFLARE_ACC_ID` (`DefaultCloudflareKvService.java:25-26`; yaml prod `:143-144`) — also used in `buildUrl`.

So `CLOUDFLARE_KV_CONFIG_TOKEN` is **not** read-only-wiring: it directly authenticates the runtime PUT write. The token genuinely needs KV **write** scope for the admin toggle to function.

### Q4 — VERDICT

**(a)** — the backend performs runtime KV **writes**; the docs write-scope wording is accurate; nothing to fix on the write-scope claim.

Secondary nuance (not the write-staleness the brief targeted, flagged not fixed):

- `06-architecture.md` describes the backend's KV role as read-only — `"...optionally read by the backend via CLOUDFLARE_KV_CONFIG_TOKEN"`. That **under-describes** reality: the backend also *writes* KV via the admin toggle. So `06` is arguably *inaccurate in the opposite direction* from what the brief expected (it under-claims the write, rather than over-claiming it). `11:136` ("KV reads/writes") is the accurate one.
- **Line-number correction:** the brief cites `06-architecture.md:109-110` for the "optionally read by the backend" phrasing. That phrasing is actually at **`06-architecture.md:112-113`** on disk. Lines 109-110 are the "Routing & image proxying" bullet, unrelated. (Possible drift since the brief was written, or a miscount — flagging so Docs/QA looks at the right line.)

### Q5 — Exact current docs wording (before-text capture)

Captured verbatim from disk so a follow-up has the before-text (Docs/QA cannot read backend code, but these are docs in *this* repo — included for completeness, since the brief asked):

- `docs/11-environment-variables.md:136` (table row):
  > `| `CLOUDFLARE_KV_CONFIG_TOKEN` | optional | dev KV token                    | prod KV token | Token scoped for KV reads/writes (config + maintenance) |`

- `docs/06-architecture.md:112-113` (the actual "optionally read" sentence; brief cited 109-110):
  > `KV is also used for runtime config keys (read by the Worker, optionally`
  > `read by the backend via `CLOUDFLARE_KV_CONFIG_TOKEN`).`

Per verdict (a), `11:136` "reads/writes" needs **no** correction — it is accurate. `06:112-113`'s "optionally read by the backend" is the line that under-describes the write capability; that is a Docs-repo / doc-accuracy call for Mastermind/Docs-QA, **not** the stale write-scope the brief hypothesised. I am not proposing wording (per brief: "Do not propose final doc wording").

---

## Tool-caution log (per brief)

The brief warned of phantom/corrupted Read output. Every `file:line` above was cross-checked with `grep -n` / `cat`, not Read alone:
- `DefaultCloudflareKvService.java`, `MaintenanceAdminController.java`, `CloudflareKvService.java` — full contents confirmed via `cat`/Read and the grep caller-sweep agree on line numbers.
- yaml `config.token`/`namespace-id` mappings confirmed by direct `grep -rn` across all three profile files.
- The `06` line-number discrepancy was caught precisely because Read line numbers were cross-checked against the brief's claim. No re-reads were needed beyond the initial cross-check; no phantom content observed this session.

---

## Implemented

- Nothing. Read-only audit.

## Files touched

- None (no code or docs modified; nothing staged).

## Tests

- Ran: none (read-only audit; no code change to validate).
- Result: N/A
- New tests added: none

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change.
- issues.md: no change by me. The bug-4 "UNCONFIRMED" item this audit resolves is owned by Docs/QA — resolution summary drafted in "For Mastermind" for them to apply. I did not edit issues.md.

## Obsoleted by this session

- Nothing in code. This audit supersedes the **factual premise** of bug 4 (that the backend KV-write path is gone / edge-only) and updates the stale detail in `audit-expo-maintenance-split.md` §5 (the toggle is now admin-gated at `/api/secure/admin/maintenance/toggle`, no longer the unauthenticated `/api/public/maintenance/toggle`). Those are prior-audit notes, not live code — left as historical record.

## Conventions check

- Part 4 (cleanliness): N/A — no code changed.
- Part 4a (simplicity): N/A — no code added.
- Part 4b (adjacent observations): one adjacent doc-accuracy item surfaced (`06` under-describes the write path + wrong cited line) — flagged in "For Mastermind", not fixed.
- Part 6 (translations): N/A this session.
- Part 11 (trust boundaries): observed in passing — the write path is correctly behind `@PreAuthorize("hasRole('ADMIN')")`; admin role is read from `SecurityContextHolder`, not client input. No issue.
- Other parts touched: Part 5 (this summary, dual-write).

## Known gaps / TODOs

- I did not verify the *Cloudflare-side token scope* (whether the actual `CLOUDFLARE_KV_CONFIG_TOKEN` secret in prod has KV write permission) — that lives in the Cloudflare dashboard / secrets, outside this repo. The code *requires* write scope; whether the deployed token grants it is a DevOps check.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- **Bug 4 resolution (for Docs/QA to apply to issues.md):** CONFIRMED FALSE ALARM on the write-scope staleness. The backend *does* retain a runtime KV-write path: `POST /api/secure/admin/maintenance/toggle` (`hasRole('ADMIN')`) → `DefaultCloudflareKvService.toggleMaintenance()` → two HTTP PUTs to Cloudflare KV `/values/{maintenance.web.active, maintenance.backend.active}`, authed with `CLOUDFLARE_KV_CONFIG_TOKEN`. `11-environment-variables.md:136` "reads/writes" is **accurate** — do not change it. CI/droplet scripts are an *additional* writer (deploy windows), not the *only* writer.
- **Adjacent doc-accuracy item (separate from bug 4, Docs/QA + Mastermind call):** `docs/06-architecture.md:112-113` says KV is "optionally **read** by the backend via `CLOUDFLARE_KV_CONFIG_TOKEN`" — this *under-describes* reality (the backend also writes via the admin toggle). If you want `06` to reflect the admin write path, that is a new doc-edit brief. I deliberately did not draft wording (brief said not to). Also note the brief's cited line `06:109-110` is off by ~3 — the phrasing is at `06:112-113` on disk; point any follow-up brief at the correct line.
- **DevOps verification suggested:** confirm the deployed `CLOUDFLARE_KV_CONFIG_TOKEN` secret actually carries KV *write* scope (the admin toggle silently depends on it; a read-only token would make the wrench fail at runtime with a 4xx from Cloudflare). Out of scope for this repo — routing note only.
