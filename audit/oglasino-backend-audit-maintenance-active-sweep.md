# Audit — `maintenance.active` deletion sweep

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-30
**Feature:** `expo-maintenance-split` (spec §4.1 / §7 step 5 — the backend-cleanup deletion)
**Type:** read-only audit. No code changed.
**Goal:** map EVERY repo reference to legacy backend-side maintenance state so the follow-up deletion brief misses nothing. Context: we are deleting ALL backend-side maintenance state — maintenance lives ONLY in Cloudflare KV (two keys: `maintenance.web.active`, `maintenance.backend.active`). The backend's only remaining role is the admin toggle that *writes* those two KV keys.

## Method

1. Deterministic ground-truth grep across the entire repo (`.git`/`target` excluded) for: `maintenance.active`, `maintenance_active`, `getMaintenanceActive`, `maintenance/active`, and a full case-insensitive sweep of `maintenance`. Every hit below is from grep, not inference.
2. Read the full dependency chain: `MaintenancePageController` → `CloudflareKvService` (interface) → `DefaultCloudflareKvService` (impl) → `ConfigurationService.getBooleanConfig` → DB row; plus `CurrentLanguageFilter`, `data-configuration.sql`, `ConfigurationSeedTest`.
3. Adversarial verification workflow — 5 independent agents, each verifying one deletion-landmine question. All five returned `confirmed`. One false landmine emitted by an agent is corrected in §6 below.

## Bottom line

The legacy maintenance read-path is a clean, self-contained chain with **exactly one external caller** and **no test depending on it**. Sessions 1–2 of this feature already moved the *write* path (admin toggle → two KV keys) and the deploy-workflow flips to the new keys; what remains for the deletion brief is the *read* path plus its DB seed. **Deleting it is low-risk**, subject to one sequencing dependency: Expo must stop polling `/api/public/maintenance/active` first (spec §7 step 4 before step 5).

---

## The deletion chain (buckets a/b/c) — DELETE these

These four sites are the legacy maintenance STATE. They form one dependency chain:
`GET /api/public/maintenance/active` → `CloudflareKvService.getMaintenanceActive()` → `DefaultCloudflareKvService.getMaintenanceActive()` → `getBooleanConfig("maintenance.active")` → DB row id 11.

### Bucket (c) — the `/active` endpoint, its controller, and the `getMaintenanceActive` read

| file:line | content | action |
| --- | --- | --- |
| `src/main/java/com/memento/tech/oglasino/controller/MaintenancePageController.java:9` | `@RequestMapping("/api/public/maintenance")` | **Delete entire file.** The controller's only endpoint is `/active`; after sessions 1–2 removed `POST /toggle`, nothing else lives here. |
| `…/MaintenancePageController.java:14` | `@GetMapping("/active")` | (same file) the sole endpoint = `GET /api/public/maintenance/active`. |
| `…/MaintenancePageController.java:15-16` | `public ResponseEntity<Boolean> isActive() { return ResponseEntity.ok(cloudflareKvService.getMaintenanceActive()); }` | the **only** caller of `getMaintenanceActive()` in the whole repo. |
| `src/main/java/com/memento/tech/oglasino/service/CloudflareKvService.java:5` | `boolean getMaintenanceActive();` | **Delete** the interface method. |
| `src/main/java/com/memento/tech/oglasino/service/impl/DefaultCloudflareKvService.java:42-45` | `public boolean getMaintenanceActive() { return configurationService.getBooleanConfig(MAINTENANCE_KEY); }` | **Delete** the impl method (and its `@Override`). |

### Bucket (a) — the OLD single KV/DB key literal

| file:line | content | action |
| --- | --- | --- |
| `src/main/java/com/memento/tech/oglasino/service/impl/DefaultCloudflareKvService.java:15-16` | `// DB-backed read source for the legacy public GET /api/public/maintenance/active poll` / `// (removed in a later phase once Expo stops polling). The toggle no longer writes this.` | **Delete** the comment (it documents only the legacy read). |
| `…/DefaultCloudflareKvService.java:17` | `private static final String MAINTENANCE_KEY = "maintenance.active";` | **Delete.** Used only by `getMaintenanceActive()` (line 44). After that method is gone the constant is unused. |

> Note on terminology: in this codebase `maintenance.active` is **not** a live KV key — it is the legacy *DB config* key. The single-KV-key string `"maintenance.active"` that the deploy workflows used to PUT is already gone (sessions 1–2 replaced both workflows' flips with the two new keys). So bucket (a) here is the DB-key constant, not a KV write.

### Bucket (b) — the backend DB Configuration seed row

| file:line | content | action |
| --- | --- | --- |
| `src/main/resources/data/configuration/data-configuration.sql:23` | `-- Maintenance` (section comment) | **Delete** the comment header with the row. |
| `…/data-configuration.sql:24` | `(11, 'maintenance.active', 'false', 'Configuration that is used for mobile check', CURRENT_TIMESTAMP),` | **Delete** row id 11. |

Row-deletion safety (verified):
- **No id collision / no renumber needed.** ids are already sparse (the file jumps 57→80 and 79→83). Neighbours are id 10 (line 21) and id 12 (line 27). The insert uses `ON CONFLICT (id) DO NOTHING`; config is looked up by **string key**, never by id (`DefaultConfigurationService` HashMap cache). Removing line 24 (and the `-- Maintenance` comment at line 23) is sufficient.
- **No seed test breaks.** `src/test/java/com/memento/tech/oglasino/moderation/ConfigurationSeedTest.java` asserts a `REQUIRED_KEYS` allowlist (validation.* keys) and does **not** include `maintenance.active`, nor does it assert a row count. Verified by reading the file.

---

## Cleanup required by the deletions (bucket c, second-order) — UPDATE these

These are not "maintenance state," but they reference the deleted endpoint/controller and go dead or stale on deletion. The deletion brief should handle them in the same session to satisfy conventions Part 4 (no dead code left behind).

| file:line | content | action | why |
| --- | --- | --- | --- |
| `src/main/java/com/memento/tech/oglasino/filter/CurrentLanguageFilter.java:38` | `"/api/public/maintenance/",` (an `ALLOWLIST_PREFIXES` entry) | **Remove this list entry.** | After `/active` is gone and the toggle lives at `/api/secure/admin/maintenance`, **no route remains under `/api/public/maintenance/`**. The allowlist entry becomes dead. The sibling entries (`/api/public/baseSite/`, `/api/public/app/version/`, `/api/public/product/seen/`) stay. |
| `src/main/resources/data/configuration/data-app-version-config.sql:5` | `-- those operator-set values across reboots, same pattern as configuration.maintenance.active.` | **Edit the comment** — drop the `configuration.maintenance.active` example (pick another seeded key, or generalize). | The row it cites as a pattern example will no longer exist; comment becomes misleading. Code-comment, zero behavior. |
| `docs/05-getting-started.md:153` | `| Cloudflare KV (maintenance) | Yes | `MaintenancePageController` returns the cached value or false |` | **Edit** the integration-matrix row — `MaintenancePageController` will no longer exist. | Names a class being deleted; existing `docs/` file so editing is allowed (no new doc). |

---

## NEW keys (bucket d) — KEEP, do not touch

These are the correct two-flag model. They are KV-only (correctly absent from `data-configuration.sql`).

| file:line | content |
| --- | --- |
| `src/main/java/com/memento/tech/oglasino/service/impl/DefaultCloudflareKvService.java:19-22` | the matrix comment + `MAINTENANCE_WEB_KEY = "maintenance.web.active"` / `MAINTENANCE_BACKEND_KEY = "maintenance.backend.active"` |
| `…/DefaultCloudflareKvService.java:48-82` | `toggleMaintenance()` + `readMaintenanceFlag` + `setMaintenanceCloudflareValue` (the sole maintenance writer; reads/writes both KV keys) |
| `src/main/java/com/memento/tech/oglasino/admin/controller/MaintenanceAdminController.java:12,18-20` | `@RequestMapping("/api/secure/admin/maintenance")`, `@PostMapping("/toggle")` → `toggleMaintenance()` (admin-gated replacement) |
| `.github/workflows/deploy-backend.yml:14-15,56,67-73` | `MAINTENANCE_BACKEND_KEY` / `MAINTENANCE_WEB_KEY` env + the two KV PUT curls |
| `.github/workflows/deploy-stage.yml:14-15,57,68-73` | same, for stage |
| `src/test/java/…/MaintenanceAdminControllerTest.java` (whole file) | tests the new admin toggle endpoint |
| `src/test/java/…/DefaultCloudflareKvServiceTest.java` (whole file) | tests `toggleMaintenance()` against the two new keys (`WEB_URL`/`BACKEND_URL` at :45-46) |

Test-comment caveat (KEEP, harmless): `MaintenanceAdminControllerTest.java:24` mentions the old `/api/public/maintenance/toggle` as historical context for why the admin controller exists. It is a javadoc string, references neither `/active` nor the DB key, and stays accurate (that path *was* removed). No action.

---

## Docs prose still on the OLD single key (bucket e) — OUT OF SCOPE here, already tracked

These docs describe the pre-split single `maintenance.active` KV flag / curl. They do **not** reference the backend `/active` endpoint or the DB row, so they are not part of *this* deletion's code blast-radius. They are stale relative to the two-flag split and were already flagged by session 2's Part 4b adjacent observation (a `docs` follow-up sweep is drafted for issues.md). Spec §7 scopes only `docs/12-deployment.md` to this feature's doc work — and that file already uses the new keys.

Recorded here for completeness so the deletion-brief author can decide whether to fold the docs sweep in:

- `docs/01-backend-pre-lunch-tasks.md` — lines 155, 162, 176, 292, 327, 352 (single-key curls + KV-key prose).
- `docs/02-deploy-checklist.md` — lines 154, 475, 578, 706 (`maintenance.active=false` setup + curls).
- `docs/04-db-reset-runbook.md` — line 445 (single-key curl; the `maintenance-on.sh`/`maintenance-off.sh` script references are operator-owned).
- `docs/06-architecture.md` — lines 30, 104, 150 (KV `maintenance.active` in the diagram + worker read prose).
- `docs/11-environment-variables.md` — lines 212, 214 (CF token / KV-namespace descriptions naming `maintenance.active`).
- `docs/12-deployment.md` — already on the new keys (lines 61-62, 73-74, 87, 126-127, 141, 146); no old-key residue. KEEP.
- `jobs/image_pipeline/IMAGE-PIPELINE-WORKER-CONTRACT.md:1216,1223` and `…IMAGE-CLEANUP-INVESTIGATION.md:242` — generic "maintenance flag ON/OFF" / "maintenance endpoint" prose, no key string. Cosmetic.

---

## Unrelated "maintenance" hits (bucket f) — KEEP, never touch

The word "maintenance" appears widely in product-domain and infra contexts that have nothing to do with the deployment flag. Listed so the deletion brief does not over-reach:

- **INTRO maintenance-page UI copy** (the strings the worker's maintenance page shows): `maintenance.label.1` / `maintenance.label.2` in all four `0001-data-web-translations-{EN,RS,RU,CNR}.sql:27-28`. These are *kept by design* — the worker still renders a maintenance page.
- **Product category / filter labels:** `category.repairs_maintenance` and children, `category.cleaning_home_maintenance`, `filter.options.maintenance_equipment` across `catalogJSON/categories/services.json`, `…/hobbies.json`, and the `0002`/`0003` translation seeds (all four languages). Product taxonomy, unrelated.
- **Postgres tuning:** `infra/docker-compose-stage.yml:144` `maintenance_work_mem=64MB`.
- **Free-text:** `dataJSON/testProducts.json:1953` ("maintenance costs" in a listing), `golden-moderation-cases.json:224` ("maintenance note"). Test fixtures.
- **Deploy/runbook "maintenance mode" prose & scripts** generally (`maintenance-on.sh`/`maintenance-off.sh`, "put site in maintenance mode" headings) — operational concept, not the backend state being deleted.
- `.claude/settings.local.json:189` — a permission entry naming a past session file. Ignore.

---

## §6 — Verification note: one false landmine corrected

The adversarial workflow's "language-filter" agent asserted: *"`getMaintenanceActive()` will still be called by `MaintenanceAdminController.toggle()` — do NOT delete this method."* **This is wrong.** Confirmed against the code (grep + read):

- `MaintenanceAdminController:20` calls `cloudflareKvService.toggleMaintenance()`.
- `toggleMaintenance()` (`DefaultCloudflareKvService:48-60`) reads current state via `readMaintenanceFlag(MAINTENANCE_WEB_KEY)` / `readMaintenanceFlag(MAINTENANCE_BACKEND_KEY)` — i.e. **direct Cloudflare KV reads**, never `getMaintenanceActive()`.
- An authoritative whole-`src` grep for `getMaintenanceActive` returns exactly three hits: the interface decl, the impl, and the single caller `MaintenancePageController:16`.

**Conclusion:** `getMaintenanceActive()` and `MAINTENANCE_KEY` are safe to delete in full. The other four agents independently agreed; ground-truth grep is authoritative over the dissenting agent.

---

## Sequencing dependency (carry into the deletion brief)

Per spec §7: **Expo (step 4) must stop polling `GET /api/public/maintenance/active` before this backend cleanup (step 5).** While `/active` lives but the toggle no longer writes the DB row (post-session-2), the endpoint is frozen at whatever `maintenance.active` currently holds (`'false'`) — a known, spec-accepted interim staleness (session 2 "Known gaps"). Deleting the endpoint before Expo migrates would 404 every mobile boot poll. The brief states pre-launch there is no client that needs `/active` kept alive — so the only gate is the Expo change landing first; no production client depends on it.

## Deletion checklist (hand to the deletion-brief author)

Code (Java + SQL):
1. Delete file `controller/MaintenancePageController.java`.
2. Remove `getMaintenanceActive()` from `service/CloudflareKvService.java` (line 5).
3. Remove `getMaintenanceActive()` impl + `MAINTENANCE_KEY` constant + their comment from `service/impl/DefaultCloudflareKvService.java` (lines 15-17, 42-45).
4. Remove the `"/api/public/maintenance/"` entry from `filter/CurrentLanguageFilter.java` `ALLOWLIST_PREFIXES` (line 38).
5. Delete the `-- Maintenance` comment + row id 11 from `data/configuration/data-configuration.sql` (lines 23-24).

Same-session cleanup (avoid leaving stale refs):
6. Edit the `configuration.maintenance.active` example out of `data/configuration/data-app-version-config.sql:5`.
7. Edit `docs/05-getting-started.md:153` (drop the `MaintenancePageController` reference).

Verify after: `./mvnw spotless:check` + `./mvnw test` (expect green — no test references the deleted symbols; 701 currently pass).

Optional / separate brief (already tracked): the bucket (e) docs sweep (01/02/04/06/11) onto the two-key model.
