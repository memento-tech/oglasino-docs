# Audit — `maintenance.active` sweep

**Repo:** oglasino-router
**Branch:** stage
**Date:** 2026-05-30
**Type:** Read-only audit. No code changes made.
**Task (verbatim):** Grep the entire repo for `maintenance.active`, `maintenance_active`, and any other `maintenance` reference; report every hit with file:line and surrounding line; classify each as (a) the OLD single KV key that should be gone, (b) one of the NEW keys `maintenance.web.active` / `maintenance.backend.active`, (c) a comment/doc/test mention, (d) unrelated. We are moving maintenance fully to Cloudflare KV with two keys (web + backend); the old single `maintenance.active` key must not exist anywhere.

## Method

```
rg -n --hidden -g '!.git' -g '!node_modules' -g '!package-lock.json' -F 'maintenance.active' .
rg -n --hidden -g '!.git' -g '!node_modules' -g '!package-lock.json' -F 'maintenance_active' .
rg -ni --hidden -g '!.git' -g '!node_modules' -g '!package-lock.json' 'maintenance' .
```

Excluded from the sweep: `.git/`, `node_modules/`, and `package-lock.json` (vendored / generated noise, no project-authored maintenance references). Everything else in the repo was searched, including hidden files, `docs/`, `tests/`, `wrangler.toml`, comments, and strings.

Classification legend:
- **(a) OLD KEY — should be gone.** The literal old single key `maintenance.active`.
- **(b) NEW KEY — correct, leave.** `maintenance.web.active` or `maintenance.backend.active`.
- **(c) COMMENT/DOC/TEST mention.** A reference to maintenance as a concept (prose, test names, the word "maintenance"), not a literal KV key.
- **(d) UNRELATED.** `MAINTENANCE_ORIGIN`, `MAINTENANCE_JSON`, `X-Oglasino-Maintenance`, etc. — env bindings, constants, response headers, page references; nothing to do with the KV key.

---

## Headline verdict

**The old single `maintenance.active` KV key is GONE from all executable surfaces** — `src/index.ts` and `tests/router.test.ts` use only the two new keys (`maintenance.web.active`, `maintenance.backend.active`). `maintenance_active` (underscore form) does not appear anywhere.

**The old `maintenance.active` literal survives only in documentation / runbook text**, in four files:

| File | Lines | What it is |
|------|-------|------------|
| `README.md` | 27, 28 | `wrangler kv key put/delete ... maintenance.active` runbook commands |
| `docs/operations.md` | 41, 42, 45, 46 | same put/delete runbook commands, stage + production |
| `docs/architecture.md` | 14, 50 | ASCII diagram + prose describing the old single-key gating |
| `CLAUDE.md` | 68, 69, 70 | the "maintenance matrix" spec block (also referenced at line 144) |

All four are stale against the two-key model now live in `src/index.ts`. None were changed (read-only audit). See "Recommended follow-up" for who owns each fix.

---

## Exact hits — `maintenance.active`

### `src/` — NONE
The worker contains **zero** occurrences of the old literal `maintenance.active`. ✅ Code is fully migrated.

### `CLAUDE.md`
```
68:- `maintenance.active=false` → allow everyone
69:- `maintenance.active=true, admin.bypass.disabled=false` → allow admin + API only
70:- `maintenance.active=true, admin.bypass.disabled=true` → block everyone (full lockdown)
```
**Classification: (a) OLD KEY (as documentation).** This is the "maintenance matrix" spec block in this repo's own agent instructions. It still describes a single `maintenance.active` flag. The code now composes from two flags (`maintenance.web.active OR maintenance.backend.active` → `webDown`). The matrix block is stale. CLAUDE.md is this repo's instruction file — not in the engineer agent's writable set and not one of the four docs-repo config files; flag for owner (Igor / Mastermind), do not self-edit.

### `README.md`
```
27:npx wrangler kv key put --env stage --binding CONFIG maintenance.active true
28:npx wrangler kv key delete --env stage --binding CONFIG maintenance.active
```
**Classification: (a) OLD KEY.** Runbook commands that write/delete the old single key. Following them would set a key the worker no longer reads — a no-op that looks like it triggers maintenance but does nothing. Should become two commands per key (`maintenance.web.active`, `maintenance.backend.active`).

### `docs/operations.md`
```
41:npx wrangler kv key put --env stage --binding CONFIG maintenance.active true
42:npx wrangler kv key put --env production --binding CONFIG maintenance.active true
45:npx wrangler kv key delete --env stage --binding CONFIG maintenance.active
46:npx wrangler kv key delete --env production --binding CONFIG maintenance.active
```
**Classification: (a) OLD KEY.** Same problem as README, for both stage and production. Highest operational risk of the four: an operator copy-pasting these during an incident would believe the site is in maintenance when it is not.

### `docs/architecture.md`
```
14:                │        ├── reads KV (maintenance.active)
50:`maintenance.active` instantly puts the entire site into a 503 state
```
**Classification: (a) OLD KEY.** Diagram + prose still describe the single-key, whole-site-503 model. The current model is per-client composition (web vs mobile) from two keys; "instantly puts the entire site into a 503" is no longer accurate (mobile is gated on backend only; admin bypass shapes who is blocked).

---

## Exact hits — `maintenance_active`

**NONE.** The underscore variant does not appear anywhere in the repo. ✅

---

## All other `maintenance` references (case-insensitive)

### `src/index.ts`
```
5:  // maintenance based on KV flags.                                   (c) comment
9:  //   - maintenance.web.active      — web's own maintenance state    (b) NEW KEY (in comment)
10: //   - maintenance.backend.active  — backend's maintenance state    (b) NEW KEY (in comment)
14: // Per-client maintenance is COMPOSED from the dependency flags...  (c) comment
17: //     webDown = maintenance.web.active OR maintenance.backend.active (b) NEW KEY (in comment)
25: //     backendDown = maintenance.backend.active OR probeFailed       (b) NEW KEY (in comment)
26: //       (mobile depends only on the backend; maintenance.web.active does NOT (b) NEW KEY (in comment)
29: //     When backendDown → 503 maintenance JSON ...                   (c) comment
30: //        X-Oglasino-Maintenance header, not the bare 503).          (d) header name
49:  MAINTENANCE_ORIGIN: string;                                        (d) env binding
56:  const MAINTENANCE_JSON = JSON.stringify({                          (d) constant
57:    status: "maintenance",                                           (d) response body literal
59:    "Oglasino is undergoing maintenance. Please try again..."        (d) response body literal
95:    // through when maintenance is active but the admin bypass...     (c) comment
107: // Read all four maintenance flags (30s edge cache each)...         (c) comment
110: let maintenanceWebActive = false;                                   (b) NEW KEY var (maintenance.web.active)
111: let maintenanceBackendActive = false;                              (b) NEW KEY var (maintenance.backend.active)
117: env.CONFIG.get("maintenance.web.active", { cacheTtl: 30 }),        (b) NEW KEY — live KV read
118: env.CONFIG.get("maintenance.backend.active", { cacheTtl: 30 }),    (b) NEW KEY — live KV read
122: maintenanceWebActive = webRaw === "true";                          (b) NEW KEY var
123: maintenanceBackendActive = backendRaw === "true";                  (b) NEW KEY var
129: maintenanceWebActive = false;                                      (b) NEW KEY var (fail-open)
130: maintenanceBackendActive = false;                                  (b) NEW KEY var (fail-open)
138: // availability only — never on web maintenance or the admin bypass (c) comment
140: let backendDown = maintenanceBackendActive;                        (b) NEW KEY var
142: // "skip probe when maintenance is set" optimization...            (c) comment
155: return maintenanceResponse(true, path, url.search, env);           (d) helper fn name
172: const webDown = maintenanceWebActive || maintenanceBackendActive;  (b) NEW KEY vars
176: return maintenanceResponse(isApi, path, url.search, env);          (d) helper fn name
199: function maintenanceResponse(                                      (d) helper fn name
206: return new Response(MAINTENANCE_JSON, {                            (d) constant
212: "X-Oglasino-Maintenance": "true",                                  (d) header name
217: return fetch(`${env.MAINTENANCE_ORIGIN}${path}${search}`)...       (d) env binding
219: headers.set("X-Oglasino-Maintenance", "true");                    (d) header name
```
**No old key. The only literal KV keys read are the two new ones (lines 117–118).** ✅

### `CLAUDE.md` (concept references, non-key)
```
10:  - KV-backed maintenance mode with admin bypass                     (c) prose
64:  ### The maintenance matrix                                         (c) heading (block below is stale — see (a) above)
144: - **The brief proposes a change that breaks the maintenance matrix.** ... (c) prose
```
**Classification: (c).** Concept references; the matrix block they point at (lines 68–70) is the stale (a) item.

### `wrangler.toml`
```
19:MAINTENANCE_ORIGIN = "https://oglasino-maintenance.pages.dev"        (d) env binding (default)
41:MAINTENANCE_ORIGIN = "https://oglasino-maintenance.pages.dev"        (d) env binding (stage)
60:MAINTENANCE_ORIGIN = "https://oglasino-maintenance.pages.dev"        (d) env binding (production)
```
**Classification: (d) UNRELATED.** The maintenance-page origin binding. No KV key here. Correct, leave.

### `package.json`
```
5:"description": "Cloudflare Worker that routes oglasino.com traffic to Vercel/API/maintenance"  (c) prose
```
**Classification: (c).** Describes the maintenance-page route target. Not a key. Fine as-is.

### `README.md` (concept references, non-key)
```
8:  - Maintenance page when KV flag is set                             (c) prose — slightly stale ("flag" singular)
23: Trigger maintenance mode locally by writing to KV (requires       (c) prose
57: Maintenance mode toggling, KV management, debugging, route migration (c) prose
```
**Classification: (c).** Prose around the (a) runbook commands at lines 27–28. Line 8 ("when KV flag is set", singular) reads as the old single-flag model.

### `docs/operations.md` (concept references, non-key)
```
37: ## Toggle maintenance mode                                         (c) heading
59: forwarding. If the probe fails, requests get the maintenance response (c) prose — still accurate (probe path)
```
**Classification: (c).** Line 59 describes the backend liveness-probe → maintenance response and is still accurate. Line 37 heads the (a) runbook commands.

### `docs/architecture.md` (concept references, non-key)
```
17: │   maintenance? ──yes──► maintenance page                         (c) diagram prose
49: Single source of truth for maintenance gating. The KV flag         (c) prose ("The KV flag", singular — stale)
```
**Classification: (c).** Lines 17 and 49 are the prose around the (a) hits at lines 14 and 50; "The KV flag" (singular) reflects the old model.

### `tests/router.test.ts`
All matches are either the new keys (b), the response-header / origin assertions (d), or test/describe names (c). Representative breakdown:
```
35,51:        MAINTENANCE_ORIGIN: "https://oglasino-maintenance.pages.dev"   (d) test env binding
99:           describe("matrix: maintenance off — allow everyone")           (c) test name
146,162,176,177,192,209,224,238,252,266,280,294,308,334,351,362,381,397:
              prodEnv({ "maintenance.web.active": ... })                     (b) NEW KEY
412,424,544,586,682:  "maintenance.backend.active": ...                     (b) NEW KEY
155,171,320,342,373,392,418,434,452,467: X-Oglasino-Maintenance header      (d) header assertion
157,325,375: https://oglasino-maintenance.pages.dev/...                     (d) origin assertion
191,207,331,360,379,423,512,527:  describe/it names containing "maintenance" (c) test name
345: body.status).toBe("maintenance")                                        (d) response body assertion
512: "ignores maintenance.web.active — web down, backend up..."              (b) NEW KEY (in test name)
619,620,633,634,673:  KV-throw / fail-open tests on the two new keys         (b) NEW KEY
```
**No old key anywhere in tests.** Tests assert the two-key model, the fail-open behavior on each new key, and the mobile-decoupling (web maintenance must not block mobile). ✅

---

## Summary table

| File | (a) OLD key hits | (b) NEW key hits | (c) concept/doc/test | (d) unrelated |
|------|:---:|:---:|:---:|:---:|
| `src/index.ts` | 0 | 13 | 7 | ~14 |
| `tests/router.test.ts` | 0 | ~30 | ~12 | ~20 |
| `wrangler.toml` | 0 | 0 | 0 | 3 |
| `package.json` | 0 | 0 | 1 | 0 |
| `CLAUDE.md` | 3 (L68–70, doc) | 0 | 3 | 0 |
| `README.md` | 2 (L27–28) | 0 | 3 | 0 |
| `docs/operations.md` | 4 (L41,42,45,46) | 0 | 2 | 0 |
| `docs/architecture.md` | 2 (L14,50) | 0 | 2 | 0 |
| `maintenance_active` (any file) | 0 | 0 | 0 | 0 |

---

## Recommended follow-up (not done — read-only audit; ownership noted)

The old `maintenance.active` literal is dead in code/tests but lives on in four docs. None are in this engineer agent's writable set as docs-content updates; routing each to its owner:

1. **`docs/operations.md` (L41,42,45,46)** — highest priority. Incident runbook commands write/delete a key the worker ignores; an operator would think the site is in maintenance when it is not. Owner: Docs/QA (per CLAUDE.md, repo `docs/` content is not written by this agent).
2. **`docs/architecture.md` (L14,49,50,17)** — diagram + prose describe a single-key whole-site-503 model that no longer matches the per-client two-key composition. Owner: Docs/QA.
3. **`README.md` (L8,27,28)** — local-dev runbook + "when KV flag is set" (singular). README is editable by this agent for "how to work here" guidance, but this is a brief-scoped read-only audit, so no change was made; flag for a follow-up brief.
4. **`CLAUDE.md` (L64–70)** — the "maintenance matrix" spec block still encodes the single `maintenance.active` flag. This is the agent's own instruction file; only Igor/Mastermind should rewrite the matrix. Flag in "For Mastermind."

No edits were made to any file. Nothing in `oglasino-docs/` was touched.
