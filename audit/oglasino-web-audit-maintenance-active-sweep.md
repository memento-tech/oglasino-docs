# Audit — maintenance-active sweep (oglasino-web)

**Date:** 2026-05-30
**Repo:** oglasino-web
**Feature:** expo-maintenance-split (web slice)
**Type:** read-only audit — no code changed
**Branch:** stage

Repo-wide sweep for every `maintenance` reference, classified per the brief:

- **(a)** deploy-workflow KV flips of the old `maintenance.active` key → must become `maintenance.web.active` (web deploy flips WEB only, never backend).
- **(b)** admin `MaintenanceToggle` UI + the `configService` call to the backend maintenance endpoint → backend toggle moving to `POST /api/secure/maintenance/toggle` (admin-gated).
- **(c)** any runtime read of maintenance state → audit claims none; confirm.
- **(d)** docs.
- **(e)** unrelated.

Search method: `grep -rniI "maintenance"` across the whole tree (excluding `node_modules`, `.next`, `.git`, `dist`, `build`, `coverage`, lockfiles), plus caller traces for `isMaintenanceActive`, `toggleMaintenance`, and `MaintenanceToggle`, plus a server-path check (`app/api`, `proxy.ts`, `middleware.ts`). All line numbers and content below are verified against direct file reads.

> Excluded from the report: `.agent/` (this audit + session files) and `.claude/settings.local.json` lines 197/200/202 (those are this session's own grep commands recorded in the permission allowlist — a tooling artifact, not a product reference).

---

## Summary table

| # | File | Lines | Class | Change needed |
|---|------|-------|-------|---------------|
| 1 | `.github/workflows/deploy-prod.yml` | 13 (key def); 50–80, 84, 116, 121 (usage) | **(a)** | `MAINTENANCE_KEY: maintenance.active` → `maintenance.web.active` |
| 2 | `.github/workflows/deploy-stage.yml` | 14 (key def); 52–82, 123, 128 (usage) | **(a)** | `MAINTENANCE_KEY: maintenance.active` → `maintenance.web.active` |
| 3 | `src/lib/admin/lib/service/configService.ts` | 48–62 (`isMaintenanceActive`), 64–78 (`toggleMaintenance`) | **(b)** | `toggleMaintenance` POST `/public/maintenance/toggle` → `/secure/maintenance/toggle`; `isMaintenanceActive` GET `/public/maintenance/active` → endpoint being deleted (see open Q) |
| 4 | `src/components/client/buttons/MaintenanceToggle.tsx` | 3, 8, 9, 11–13, 16–18 | **(b)** | admin wrench UI; calls the two `configService` fns. No change beyond the service. **Mounted** (item 5) |
| 5 | `app/[locale]/admin/config/page.tsx` | 3 (import), 24 (mount) | **(b)** | mount site of the wrench; no change needed |
| 6 | `docs/04-deployment.md` | 3,19,43,44,46,53,61,62,115,121,135,141,148,152,156,157,266,271,272,276,289,298,300,304,309,323,325 | **(d)** | documents old key + manual flip flow |
| 7 | `docs/07-pre-lunch-tasks.md` | 38,85,87,94,103,104,112,116,121,220,246,256,258,265,269,290,291 | **(d)** | documents old key + worker-contract test |
| 8 | `docs/README.md` | 36,107,123,126 | **(d)** | describes single `maintenance.active` KV flag |
| 9 | `docs/02-architecture.md` | 39,52,58 | **(d)** | worker reads `maintenance.active` |
| 10 | `docs/05-environment-variables.md` | 158,162 | **(d)** | `CF_KV_NAMESPACE_ID` holds `maintenance.active` |
| 11 | `docs/03-development.md` | 99,161 | **(d)** | worker reads `maintenance.active` |
| 12 | `jobs/image_pipeline/IMAGE-PIPELINE-WORKER-CONTRACT.md` | 1203,1210 | **(d)** | "Maintenance flag ON/OFF" in a worker-contract doc |
| 13 | `jobs/rn_sync/FRONTEND-CHANGES-FOR-RN.md` | 51,89,215,539 | **(d)** | RN-sync change-log entries about the deploy flip |
| 14 | `src/components/icons/dynamic/RepairsMaintenanceCategoryIcon.tsx` | 1 | **(e)** | "Repairs & Maintenance" product-category icon — coincidental word |
| 15 | `src/components/icons/dynamic/index.ts` | 103 | **(e)** | re-export of that icon |

**Runtime reads (c): CONFIRMED NONE.** No `middleware.ts`; `proxy.ts` and `app/api` have zero maintenance references; the only maintenance read in the codebase is `isMaintenanceActive()` in the admin wrench (item 4), which renders inside the admin config page (item 5) — admin tooling, not a public-site gate. Details in §(c).

---

## (a) Deploy-workflow KV flips — must become `maintenance.web.active`

Both web deploy workflows flip the KV flag ON (and never automatically OFF — the OFF flip is manual, by design). Each defines a single `MAINTENANCE_KEY` env var and references it everywhere via `${MAINTENANCE_KEY}`, so the **rename is a one-line change per workflow** (the `env:` definition). Both workflows also flip `admin.bypass.disabled` — that flag is **unchanged** by this feature.

### 1. `.github/workflows/deploy-prod.yml` (prod KV namespace `CF_KV_NAMESPACE_ID`)

- **L13** `  MAINTENANCE_KEY: maintenance.active`  ← **the rename point** → `maintenance.web.active`
- **L50** `      - name: Maintenance ON — flip Cloudflare KV`
- **L53** `          echo "Setting ${MAINTENANCE_KEY}=true in KV..."`
- **L56** `            "https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/storage/kv/namespaces/${CF_KV_NAMESPACE_ID}/values/${MAINTENANCE_KEY}" \`  (PUT `--data "true"`, L59)
- **L63** `            echo "::error::Cloudflare KV PUT for ${MAINTENANCE_KEY} failed with HTTP $response"`
- **L66–L78** also flips `${ADMIN_BYPASS_KEY}` (= `admin.bypass.disabled`) to true — **unchanged by this feature**.
- **L84** `        # — Worker returns 503 when maintenance.active=true, 200 otherwise.`  ← literal in a comment; update for accuracy.
- **L116 / L121** success/failure notices: `"…manually flip both ${MAINTENANCE_KEY} and ${ADMIN_BYPASS_KEY} to false…"` (use the var, so they follow the rename automatically).

**Action for the web brief:** change L13 to `maintenance.web.active`; fix the L84 comment. Do **not** add a `maintenance.backend.active` flip — a web deploy must not signal a backend outage (spec §6). KV value semantics (`"true"`) unchanged.

> Note: the prod verify step (L82–L99) probes `https://api.oglasino.com/health` for a 503. The spec changes the **worker's** probe target to `/actuator/health/readiness` (router slice, §3.4), not this CI verify. Out of scope for the web rename, but worth knowing the CI verify still hits `/health`.

### 2. `.github/workflows/deploy-stage.yml` (stage KV namespace `CF_KV_NAMESPACE_ID_STAGE`)

- **L14** `  MAINTENANCE_KEY: maintenance.active`  ← **the rename point** → `maintenance.web.active`
- **L16** `  MAINTENANCE_HEADER: X-Oglasino-Maintenance`  (used by the verify step; not the KV key)
- **L52** `      - name: Maintenance ON — flip Cloudflare KV`
- **L55** `          echo "Setting ${MAINTENANCE_KEY}=true in stage KV..."`
- **L58** `            "…/namespaces/${CF_KV_NAMESPACE_ID}/values/${MAINTENANCE_KEY}" \`  (PUT `--data "true"`, L61)
- **L65** error echo referencing `${MAINTENANCE_KEY}`.
- **L68–L80** also flips `${ADMIN_BYPASS_KEY}` to true — **unchanged**.
- **L123 / L128** success/failure notices referencing both vars (follow the rename automatically).

**Action:** change L14 to `maintenance.web.active`. The stage verify step (L84–L106) reads the `X-Oglasino-Maintenance` response header — header name, not the KV key, so no change there.

---

## (b) Admin `MaintenanceToggle` UI + backend endpoint call

### 3. `src/lib/admin/lib/service/configService.ts`

Admin config service (under `src/lib/admin/`). Two maintenance functions, both hitting `BACKEND_API` (base = `NEXT_PUBLIC_API_URL`, i.e. `…/api`), so the wire paths are `/api/public/maintenance/active` and `/api/public/maintenance/toggle`.

- **L48–L62** `export async function isMaintenanceActive(): Promise<boolean>` — **L50** `const res = await BACKEND_API.get<boolean>('/public/maintenance/active');` (logs `config.isMaintenanceActive` at L56/L59).
- **L64–L78** `export async function toggleMaintenance(): Promise<boolean>` — **L66** `const res = await BACKEND_API.post<boolean>('/public/maintenance/toggle');` (logs `config.toggleMaintenance` at L72/L75).

**Actions for the web brief:**
- **L66** `toggleMaintenance` → repoint from `/public/maintenance/toggle` to the moved, admin-gated `/secure/maintenance/toggle` (spec §4.3 / §6). This is the one unambiguous web wire change. (Currently the toggle POSTs to an **unauthenticated** `/api/public/` path — the very hole §4.3 closes.)
- **L50** `isMaintenanceActive` GET `/public/maintenance/active` — spec §4.1 **deletes** this backend endpoint (KV becomes the sole store). The web admin "read current state" call therefore needs a decision (drop it / derive from the toggle response / KV-backed read). See "Open questions."

### 4. `src/components/client/buttons/MaintenanceToggle.tsx`

The admin "wrench" toggle (`'use client'`, `lucide-react` `Wrench`, green/red).

- **L3** `import { isMaintenanceActive, toggleMaintenance } from '@/src/lib/admin/lib/service/configService';`
- **L8** `export default function MaintenanceToggle() {`
- **L9** `const [isActive, setIsActive] = useState(false);`
- **L11–L13** `useEffect(() => { isMaintenanceActive().then(setIsActive); }, []);`  ← reads current state on mount
- **L16** `<div … onClick={() => toggleMaintenance().then(setIsActive)}>`  ← writes on click
- **L17** tooltip `…maintenance (${isActive})`
- **L18** `<Wrench color={isActive ? 'red' : 'green'} />`

Updating `configService` (item 3) covers the wire change; this component needs no edit (unless the read in item 3 is dropped, in which case the `useEffect`/`isActive` display would change — a follow-on of the open question).

### 5. `app/[locale]/admin/config/page.tsx`

The admin config page — **the mount site** (this is a real, live admin surface, not orphaned).

- **L3** `import MaintenanceToggle from '@/src/components/client/buttons/MaintenanceToggle';`
- **L24** `        <MaintenanceToggle />`  (rendered next to `<ConfigFilter />`, above `<ConfigTable />`)

No change needed here.

---

## (c) Runtime maintenance reads — CONFIRMED NONE

The spec's claim ("Web reads no maintenance state at runtime (confirmed)", §6) holds:

- **No Next.js middleware:** `middleware.ts` does not exist at the repo root.
- **`proxy.ts`:** exists, **0** maintenance references.
- **`app/api/**`:** **0** maintenance references (checked).
- The **only** maintenance read anywhere in the code is `isMaintenanceActive()` (item 3), whose **only caller** is `MaintenanceToggle` (item 4) — verified by caller trace. `MaintenanceToggle`'s only mount is the admin config page (item 5). So the read is admin tooling that displays the wrench's current state; it does not gate any public request, layout, page, or proxy path.
- `toggleMaintenance()`'s only caller is the same admin component.

No public/SSR/edge code in web consults maintenance state. The audit's "none" stands.

---

## (d) Docs

All hits below document the **current** single-flag flow (`maintenance.active`, the `X-Oglasino-Maintenance` header, the manual flip-OFF). They describe behavior, not code, so they don't block the rename — but they go stale the moment the key becomes `maintenance.web.active` and the toggle moves to `/api/secure/`. Existing `docs/` files may be edited (not created) per repo rules; whether the web engineer updates them or Docs/QA owns it is the brief author's call. Spec §7's rename touch-point list names `docs/12-deployment.md` (a **backend**-repo doc), not these web docs — so these are an *additional* web-side doc-debt the brief should account for.

- **`docs/04-deployment.md`** — the deployment guide; heaviest concentration. Lines 3, 19, 43, 44, 46, 53, 61, 62, 115, 121, 135, 141, 148, 152, 156, 157, 266, 271, 272, 276, 289, 298, 300, 304, 309, 323, 325. Includes the mermaid flow, the manual flip-OFF section, and copy-paste `curl …/values/maintenance.active` snippets (L266, L276, L304).
- **`docs/07-pre-lunch-tasks.md`** — pre-launch checklist + worker-contract hand-test. Lines 38, 85, 87, 94, 103, 104, 112, 116, 121, 220, 246, 256, 258, 265, 269, 290, 291. Includes `curl …/values/maintenance.active` (L94, L116, L269) and a checklist item "KV namespace contains `maintenance.active` key" (L290).
- **`docs/README.md`** — repo overview. Lines 36, 107, 123, 126. L123/L126 describe the "single Cloudflare KV flag (`maintenance.active`)" shared with the backend pipeline.
- **`docs/02-architecture.md`** — Lines 39, 52, 58. "Worker reads `maintenance.active` KV flag."
- **`docs/05-environment-variables.md`** — Lines 158, 162. `CF_KV_NAMESPACE_ID` = "KV namespace holding `maintenance.active`."
- **`docs/03-development.md`** — Lines 99, 161. "Worker reads maintenance.active with cacheTtl=60s."
- **`jobs/image_pipeline/IMAGE-PIPELINE-WORKER-CONTRACT.md`** — Lines 1203, 1210. "Maintenance flag ON/OFF" steps in a worker-contract doc (not app code; a job-runner spec doc).
- **`jobs/rn_sync/FRONTEND-CHANGES-FOR-RN.md`** — Lines 51, 89, 215, 539. Historical change-log describing the deploy-prod KV flip and the `X-Oglasino-Maintenance` header as CI/CD-only and RN-irrelevant.

---

## (e) Unrelated

- **`src/components/icons/dynamic/RepairsMaintenanceCategoryIcon.tsx:1`** — a product-category icon for the "Repairs & Maintenance" category. Coincidental word match; nothing to do with the maintenance gate.
- **`src/components/icons/dynamic/index.ts:103`** — `export { default as RepairsMaintenanceCategoryIcon } …`. Same, the barrel re-export.

---

## Open questions for the brief author / Mastermind

1. **`isMaintenanceActive` (the admin read) after the backend poll is removed.** Spec §4.1 deletes `GET /public/maintenance/active`, but §6 only specifies repointing the **toggle**. The web admin wrench reads current state via `isMaintenanceActive()` on mount (`MaintenanceToggle.tsx:12` → `configService.ts:50`). The brief must say what that read becomes: drop it (wrench renders from last toggle result only), derive from the toggle endpoint's response, or a KV-backed read. The **write** side (`toggleMaintenance` → `/secure/maintenance/toggle`) is unambiguous.
2. **Toggle endpoint path precision.** Spec §4.3 hedges between `POST /api/secure/maintenance/toggle` and "the established admin namespace." The web call is relative (`/public/maintenance/toggle` against the `…/api` base), so the implementer needs the exact secure path. Note the existing admin config calls in the same file use `/secure/admin/config` (L17) and `/secure/admin/config/update` (L33) — so the admin namespace convention here is `/secure/admin/…`, which may imply `/secure/admin/maintenance/toggle` rather than `/secure/maintenance/toggle`. Confirm with backend.
3. **Web doc debt.** Six `docs/*` files (esp. `04-deployment.md`, `07-pre-lunch-tasks.md`) and two `jobs/*` contract docs hard-document `maintenance.active`, the manual flip-OFF, and copy-paste curl snippets. Spec §7's rename list does not include these web docs (only the backend's `docs/12-deployment.md`). Decide owner (web engineer vs Docs/QA) and scope.

These are flagged, not resolved — this is a read-only audit.
