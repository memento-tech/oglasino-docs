# Audit — `maintenance.active` Sweep

**Type:** read-only audit. No code, spec, or config-file changes made.
**Date:** 2026-05-30
**Repo:** `oglasino-docs`
**Brief:** grep the entire repo for every "maintenance" reference; report every hit with `file:line` and line content; classify; map every doc the closing maintenance-split docs sweep must update so it misses nothing.

**Context (from the brief).** Maintenance is moving fully to Cloudflare KV with **two keys** — `maintenance.web.active` + `maintenance.backend.active` — composed per-client at the worker. All backend-side maintenance (the DB config row, the `GET /public/maintenance/active` poll, the `POST /maintenance/toggle` admin endpoint) is being **deleted**. The new model is specified in [`features/expo-maintenance-split.md`](../features/expo-maintenance-split.md). The old model was a single `maintenance.active` KV key plus the backend pieces.

---

## 1. Headline numbers

| Scope | Files w/ hits | Total hits | Sweep touches? |
| --- | --- | --- | --- |
| **Live docs** (`features/`, `infra/`, `decisions.md`, `issues.md`, `state.md`, `meta/`, `future/`) | **23** | ~323 | **Yes — this is the map** |
| **`sessions/` archives** | 65 | 402 | **No** — frozen verbatim copies (conventions Part 5) |
| **`.agent/` working files** (this brief, handoffs, Docs/QA session summaries) | 5 | ~22 | No — working/archival, not sweep targets |
| **Total (all `*.md`)** | 88 (29 outside `sessions/`) | **725** | — |

**Of the 23 live docs, 9 carry hits the closing sweep must edit. 14 carry only model-neutral or already-correct references (leave as-is).**

---

## 2. Classification key

The brief defined five categories (a–e). I added **(f) OTHER** because a large share of hits are genuinely unrelated to the KV-flag model and would be miscounted as "stale" otherwise.

| Cat | Meaning | Sweep action |
| --- | --- | --- |
| **(a)** | STALE — references the OLD single `maintenance.active` KV key, the backend DB config row, the `/maintenance/active` poll, or `/maintenance/toggle`, in prose/specs/runbooks | **Update** |
| **(b)** | The `decisions.md` 2026-05-15 entry describing the old single→two-flag split | **Supersede** (see §4) |
| **(c)** | Content in `features/expo-maintenance-split.md` (the new spec) that already describes the two-key model correctly | Leave |
| **(d)** | An `issues.md` entry referencing maintenance | **Append resolution note** (append-only — see §5) |
| **(e)** | Content in `infra/cloudflare/maintenance.md` (the runbook) | **Update in place** (still the live runbook) |
| **(f)** | OTHER — model-neutral; needs no change | Leave |

### Classification correction (important)

The fan-out subagents over-applied **(c)** to stale OLD single-key references inside `infra/cloudflare/maintenance.md` and `infra/oglasino-devops-blueprint-v5.md`. Category (c) is **only** the new spec being correct. I have re-filed those:

- Stale single-key lines **inside the runbook** `infra/cloudflare/maintenance.md` → **(e)** with `needsUpdate=true` (it is the runbook *and* it needs updating to two keys).
- Stale single-key lines **inside the blueprint** `infra/oglasino-devops-blueprint-v5.md` → **(a)** (a runbook/blueprint reference to the old key) — **but see the open question in §8: is the v5 blueprint a live runbook or a frozen historical planning doc?**

---

## 3. THE UPDATE MAP — docs the closing sweep must touch

Ordered by edit weight. Every `needsUpdate=true` line is listed in full in §6.

| # | Doc | Cat | Hits needing edit | What the sweep does |
| --- | --- | --- | --- | --- |
| 1 | `infra/oglasino-devops-blueprint-v5.md` | (a) | **35** | Replace single `maintenance.active` with the two keys throughout worker source, deploy CI, droplet scripts, smoke tests, KV-init, snapshots — **OR** declare it a frozen historical blueprint (§8) |
| 2 | `infra/cloudflare/maintenance.md` | (e) | **9** | The live runbook → two-key KV table, gate matrix, deploy diagram, restore steps, droplet-script table |
| 3 | `decisions.md` | (b)+(a) | **9** | Supersede the 2026-05-15 entry; fix the `/api/public/maintenance/*` allowlist line (1352) and the `MaintenancePageController`/`/public/maintenance/active` refs (1271, 209) |
| 4 | `issues.md` | (d) | up to 11 | Append a resolution note to the open 2026-05-29 redundancy entry; leave fixed historical entries as accurate records (§5) |
| 5 | `infra/overview/secret-inventory.md` | (a) | **8** | Deploy-workflow rows say "writes `maintenance.active` and `admin.bypass.disabled`" → two-key wording |
| 6 | `features/expo-boot-redesign.md` | (a) | 0 (see note) | 22 hits describe the mobile app's internal `maintenance` boot state — **model-neutral**; 0 flagged. Verify no backend-poll reference survives the spec close |
| 7 | `infra/vercel/deployments.md` | (a) | **5** | "Flip Cloudflare KV maintenance flag ON/OFF" → web deploy flips only `maintenance.web.active` |
| 8 | `state.md` | (a) | **1** | Session-log line 336 ("split-flag maintenance gate") — historical, optional touch |
| 9 | `meta/devops-chat-bootstrap.md` | (a) | **1** | Line 163 names `maintenance.active` as the gating flag example → two keys |
| 10 | `future/db-overload-protection.md` | (a) | **6** | A `future/` doc that self-trips `maintenance.active`; will need the new backend key when built |
| 11 | `features/version-checksums.md` | (a) | **1** | Line 22 cites the `MaintenancePageController.toggle()` deferred bucket (being deleted) |

**`features/expo-maintenance-split.md` (category c, 60 hits)** — the new spec. Correct. **Leave entirely.**

---

## 4. Category (b) — the `decisions.md` 2026-05-15 entry

The entry at **`decisions.md:1261`** — *"Maintenance gate split into two KV flags; deploy flows full-lockdown with manual three-step restore"* — documents the OLD model: a single `maintenance.active` + `admin.bypass.disabled`, backend+web deploy workflows flipping `maintenance.active`, and a droplet three-step restore that flips `maintenance.active` off. The whole entry (lines 1261, 1263, 1265, 1267, 1269, 1271, 1277) is superseded by the two-key model.

**Recommended sweep action (decision is Mastermind's, not mine):** do **not** rewrite the historical entry — `decisions.md` is a decision log. The clean move is a **new dated entry** recording the move to two keys, with a back-reference superseding the 2026-05-15 one (precedent: the issues.md "> Resolved" / "> Fixed" pattern). The 2026-05-15 entry's line 1271 also defers the `MaintenancePageController` trust-boundary defect, which the split now closes by deleting the controller — the new entry should note that.

---

## 5. Category (d) — `issues.md` (append-only)

`issues.md` is an **append-only log of out-of-scope findings** (header line 3). Its maintenance hits fall in two buckets:

- **Open, directly superseded by the feature:** the 2026-05-29 entry *"Mobile's backend `/public/maintenance/active` check is redundant with the edge worker"* (lines 7, 11, 12, 14, 16) and the 2026-05-28 Φ3 cold-start entry referencing the same poll (lines 53, 54). The expo-maintenance-split feature **is** the resolution of the 2026-05-29 entry. Sweep action: **append** a `> Resolved by expo-maintenance-split` note — do not delete or rewrite the finding.
- **Closed historical records:** the two 2026-05-15 router-worker findings (lines 924, 926, 935, 937) reference single `maintenance.active` and are marked **fixed**. These are accurate records of what was found and fixed at the time. **Leave as-is** — rewriting closed log entries to a later model falsifies the record.

So the subagent's "all 11 → needsUpdate=true" overstates it: realistically **one append** (the open 2026-05-29 entry) plus an optional cross-reference. Flagged for Mastermind to confirm.

---

## 6. Live docs — every hit, full content

Legend: ✏️ = `needsUpdate=true`; — = leave.

### `features/expo-maintenance-split.md` — category (c), 60 hits — LEAVE ENTIRELY
The new spec; correctly describes the two-key model end to end (problem statement, worker compose logic, `maintenance.web.active`/`maintenance.backend.active` definitions, KV-as-sole-store, toggle rework to `/api/secure/`, deploy-flow flips, mobile boot/exit-poller re-point). All 60 hits `needsUpdate=false`. Lines (sample): 1, 3, 12, 19, 28–29, 31, 36, 38–40, 50, 67, 70–73, 86, 105–115, 126, 130, 132, 138, 140, 146, 151, 154, 176, 182, 214, 217, 220, 225–226, 235–236, 244, 250, 281, 298, 303, 307, 309, 311–312, 322.

### `decisions.md` — categories (b),(a),(f) — 9 ✏️
| Line | Cat | Content (trimmed) | |
| --- | --- | --- | --- |
| 17 | f | "...Gate 0 added ahead of the maintenance gate in `bootStore`..." (mobile boot gate, two-key-compatible) | — |
| 49 | f | "...still means 'namespace absent from the response' → maintenance..." (mobile checksum semantics) | — |
| 115 | f | "`MaintenancePollInit.tsx` (the active-checker poll...)" (mobile component) | — |
| 142 | f | "Option A — parked maintenance-poll decision..." (mobile poll, deferred) | — |
| 203 | f | "...resolved on terminal *status* (`ready`/`maintenance`/...)" (mobile boot status) | — |
| 209 | **a** | "...`Promise.all([checkIfMaintenance, ...])` ... `GET /public/maintenance/active` ... consistently do." | ✏️ |
| 375 | f | "...render the non-`ready` states (loading, maintenance, ...) as overlays..." (mobile arch) | — |
| 481 | f | "...not maintenance burden." (generic prose, unrelated) | — |
| 638 | f | "...NetInfo for offline-vs-maintenance distinction..." (mobile) | — |
| 645 | f | "F6 maintenance-vs-offline..." (mobile QA) | — |
| 1182 | f | "...care areas (maintenance matrix, fail-open KV reads...)" (router care-area list) | — |
| 1184 | f | "The 2026-05-15 maintenance-gate split decision retroactively validates this..." | — |
| **1261** | **b** | `## 2026-05-15 — Maintenance gate split into two KV flags; ...` | ✏️ |
| **1263** | **b** | "The Cloudflare router worker's single-flag maintenance gate is now a two-flag gate..." | ✏️ |
| **1265** | **b** | "...reads two string-typed KV keys ...: `maintenance.active` and `admin.bypass.disabled`..." | ✏️ |
| **1267** | **b** | "...deploy workflows flip **both** `maintenance.active = \"true\"` and `admin.bypass.disabled`..." | ✏️ |
| **1269** | **b** | "...`maintenance-off.sh` ... flips `maintenance.active` to `\"false\"`..." | ✏️ |
| **1271** | **a** | "...backend `MaintenancePageController` trust-boundary defect ... deferred..." (controller being deleted) | ✏️ |
| **1277** | **b** | "Per-surface flags (`portal.maintenance.active` + `admin.maintenance.active`). Rejected..." | ✏️ |
| **1352** | **a** | "...allowlist of language-independent public routes (... `/api/public/maintenance/*` ...)..." | ✏️ |

> Note on (b): "update" here means **supersede via a new entry** (§4), not rewrite in place.

### `issues.md` — category (d) — see §5 for the append-only nuance
| Line | Content (trimmed) | Sweep |
| --- | --- | --- |
| 7, 11, 12, 14, 16 | 2026-05-29 entry: "Mobile's backend `/public/maintenance/active` check is redundant with the edge worker" | **append "Resolved by expo-maintenance-split"** |
| 53, 54 | 2026-05-28 Φ3 cold-start entry: `checkIfMaintenance` / `GET /public/maintenance/active` boot request | leave (historical) or cross-ref |
| 924, 926, 935, 937 | 2026-05-15 router-worker findings re single `maintenance.active` — **fixed** | **leave** (closed records) |

### `infra/cloudflare/maintenance.md` — category (e), runbook — 9 ✏️
| Line | Content (trimmed) | |
| --- | --- | --- |
| 15 | `| maintenance.active | string | "true" | Engages the maintenance gate. |` | ✏️ |
| 27 | gate matrix header `| maintenance.active | admin.bypass.disabled | Effect |` | ✏️ |
| 79 | deploy diagram `GH->>CF: PUT maintenance.active = "true"` | ✏️ |
| 91 | "Both `maintenance.active` and `admin.bypass.disabled` are set to `\"true\"`..." | ✏️ |
| 105 | restore step 3 "...Flips `maintenance.active` to `\"false\"`..." | ✏️ |
| 110 | "...flipping `maintenance.active` off before the caches are warm..." | ✏️ |
| 121 | droplet-script table `maintenance-off.sh ... Flips maintenance.active to "false"` | ✏️ |
| 16, 17, 30, 31, 39, 43, 44, 49, 53, 55, 127 | runbook prose/response-shape/`use.backend.check`/`admin.bypass.disabled` lines — accurate under two-key model | — |

> The runbook stays the live runbook. It needs the single key replaced by the two-key compose model (the deploy diagram, the gate matrix, and the two droplet scripts are the load-bearing edits).

### `infra/oglasino-devops-blueprint-v5.md` — category (a) (re-filed from subagent's (c)) — 35 ✏️
Stale single-key references concentrated in: the TL;DR diagram (73, 99, 100), the embedded worker source listing (46, 1417, 1453, 1456, 1459, 1460), KV-init (1404), manual-flip docs (1622, 1627, 1642), droplet scripts (1675, 1682, 1687, 1690, 1697, 1702), CI workflow jobs (2094, 2098, 2106, 2114, 2130, 2159, 2160, 2162, 2166, 2283, 2287, 2295, 2303, 2319, 2347, 2348, 2350, 2354), smoke tests (2416, 2417, 2420), post-deploy/rollback (2547, 2553, 2575), checklists (2681, 2918, 2919), and infra snapshots (2829, 2975). Model-neutral hits (the `oglasino-maintenance` 503 origin, `X-Oglasino-Maintenance` header, response JSON, "maintenance window" prose): 5, 7, 45, 1350, 1421, 1423–1425, 1540, 1548, 1553, 1555, 1635, 1638, 1706, 1763, 2120, 2127, 2412, 2831 → (f), leave.

> **This file is the single largest edit target. See §8 — confirm whether it's a live runbook or a frozen historical blueprint before editing.**

### `infra/overview/secret-inventory.md` — category (a) — 8 ✏️
Lines 68, 69, 70, 71, 72, 73, 74, 75 — the `CF_*` deploy-workflow secret rows all describe the workflow as writing `maintenance.active` and `admin.bypass.disabled` to KV. Under the new model the deploy workflows write `maintenance.web.active` / `maintenance.backend.active`. All ✏️.

### `infra/vercel/deployments.md` — category (a) — 5 ✏️
Lines 29, 32, 37, 41, 45 — "Maintenance gating is manual/auto-restore", "Flip Cloudflare KV maintenance flag ON/OFF". Web deploy now flips only `maintenance.web.active`. All ✏️.

### `future/db-overload-protection.md` — category (a)/(f) — 6 ✏️
| Line | Cat | Content (trimmed) | |
| --- | --- | --- | --- |
| 7 | a | "...auto-flip the maintenance gate at critical saturation..." | ✏️ |
| 17 | a | "...self-trips Cloudflare's `maintenance.active` flag at CRITICAL..." | ✏️ |
| 54 | a | "...flips Cloudflare `maintenance.active = \"true\"`..." | ✏️ |
| 68 | a | "...the existing maintenance gate's deploy flow..." | ✏️ |
| 297 | a | "...the existing maintenance gate is written from GitHub Actions..." | ✏️ |
| 346 | a | "...KV-write path in `MaintenanceAutoTripService`..." | ✏️ |
| 21, 27, 62, 65, 66, 81, 82, 129, 175, 277, 283, 305, 307, 374 | f | internal `auto.maintenance.*` Configuration flags, runbook step names, "maintenance window" prose — model-independent | — |

> This is a `future/` (not-yet-built) doc. Its target backend key becomes `maintenance.backend.active` when the feature is built. Sweep can update the references now or note the dependency.

### `features/expo-boot-redesign.md` — category (a)/model-neutral — 22 hits, 0 ✏️
All 22 hits (lines 9, 23, 27, 44, 62, 63, 70, 125, 135, 136, 137, 141, 145, 148, 150, 151, 155, 159, 308, 329, 338, 347) describe the **mobile app's internal `maintenance` boot state** (a `BootStatus`), the boot-gate state machine, and `GET /maintenance/active` as it existed pre-redesign. These are the app's runtime state, not the KV-flag deployment model. The expo-maintenance-split spec re-points the mobile boot/exit poller away from the backend poll; this spec's prose is historical-at-close and **model-neutral for the KV sweep**. 0 flagged. **Verify** during the sweep that no live backend-poll instruction survives.

### `features/expo-service-error-contract.md` — category (a)/model-neutral — 9 hits, 0 ✏️
Lines 13, 21, 66, 70, 81, 114, 119, 122 — the offline-vs-maintenance Gate 0 work and "Φ4 keeps the backend maintenance poll" (line 81 explicitly defers the poll-vs-edge question to the open issues.md entry, which the split now resolves). Shipped spec describing mobile boot structure; model-neutral. 0 flagged.

### `features/version-checksums.md` — category (a)/(f) — 1 ✏️
| Line | Cat | Content | |
| --- | --- | --- | --- |
| 22 | a | "`MaintenancePageController.toggle()` trust-boundary. Pre-existing; tracked in the deferred bucket from 2026-05-15." | ✏️ |
| 171 | f | "...the Cloudflare router worker serves a maintenance page." (REFUSING_TRAFFIC, model-neutral) | — |

### `features/version-checksums-per-language.md` — category (f) — 5 hits, 0 ✏️
Lines 61, 63, 113, 125, 143 — all "backend-down → maintenance" / "not a maintenance trigger" mobile-gate fail-safe semantics. Model-neutral.

### `features/seo-foundation.md` — category (f) — 1 hit, 0 ✏️
Line 534 — `MAINTENANCE_ORIGIN` 503 missing `X-Robots-Tag`. Worker-origin behavior, model-neutral.

### `features/expo-structural-foundation.md` — category (f) — 4 hits, 0 ✏️
Lines 67, 73, 116, 123 — F6 offline-vs-maintenance, `maintenanceService.tsx` as a non-JSX file. Model-neutral.

### `features/expo-navigation-foundation.md` — category (f) — 3 hits, 0 ✏️
Lines 47, 55, 278 — maintenance *overlay* rendering pattern (always-mounted Stack). Architectural, model-neutral.

### `features/expo-release-readiness.md` — category (a) — 2 ✏️
| Line | Content (trimmed) | |
| --- | --- | --- |
| 309 | "...Reduce maintenance polling from 5s to a saner interval..." | ✏️ |
| 371 | "5-second maintenance poll on mobile. Confirmed 720 calls/hour. Deferred to chat I." | ✏️ |
> The 5s backend poll is exactly what the split removes (replaced by the worker `X-Oglasino-Maintenance` signal + exit poller). The split supersedes "chat I."

### `state.md` — categories (c-correct)/(a)/(d)/(f) — 1 ✏️
| Line | Cat | Content (trimmed) | |
| --- | --- | --- | --- |
| 97 | f | "...connectivity Gate 0 ahead of the maintenance gate..." (mobile, two-key-compatible) | — |
| 123 | c✓ | "### Expo Maintenance Split" (active-features entry header) | — |
| 128 | c✓ | "...splits the single `maintenance.active` ... into two dependency flags ..." — correctly describes the new model | — |
| 275 | d | session-log: "...the 2026-05-29 maintenance-redundancy entry left untouched..." | — |
| 336 | a | "**2026-05-15** — DevOps closeout: ... split-flag maintenance gate propagated..." | ✏️ (optional, historical) |

### `meta/conventions.md` — category (f) — 3 hits, 0 ✏️
Lines 104 ("maintenance matrix" router care-area), 123 ("brief-authorized maintenance flips"), 463 ("Maintenance state ... live there" — the edge-boundary principle). All model-neutral architecture/policy. **Leave.**

### `meta/devops-chat-bootstrap.md` — category (a) — 1 ✏️
Line 163 — "A KV flag that gates access (`maintenance.active`, `admin.bypass.disabled`) is a trust boundary." The example names the old single key. ✏️ (update to `maintenance.web.active` / `maintenance.backend.active`).

### `infra/cloudflare/workers.md` / `infra/github/workflows.md` / `infra/github/repos.md` / `infra/README.md` — category (f) — 1 hit each, 0 ✏️
All four refer to **`oglasino-maintenance`**, the static-HTML 503 page worker/repo (the `MAINTENANCE_ORIGIN`). That worker is unchanged by the KV-flag split. **Leave.** (Lines: workers.md:9, workflows.md:32, repos.md:10, README.md:13.)

---

## 7. `sessions/` archives — frozen, NOT edited

Per conventions Part 5, session files in `sessions/` are **straight verbatim copies, never reformatted or edited**. All **402 hits across 65 files** are `needsUpdate=false` regardless of content. They are reported for completeness only; the closing sweep must **not** touch them.

Two flavors of hit appear:

- **OLD KV/backend model (category a-flavored):** files documenting the single `maintenance.active` key, the `/api/public/maintenance/active` poll, the `/maintenance/toggle` endpoint, the backend DB config row, or the deploy-workflow flips. Highest-density: `audit-deploy-workflow.md` (61), `audit-expo-service-error-contract.md` (51), `investigation-refetch-loop.md` (40), `2026-05-28-oglasino-expo-boot-redesign-8.md` (39), `audit-oglasino-backend-boot-redesign.md` (35), `2026-05-19-oglasino-router-seo-discovery-1.md` (29), `audit-oglasino-expo-boot-redesign.md` (29), plus ~15 more backend/router/expo sessions.
- **Mobile app internal `maintenance` boot-state (category f-flavored):** files about the `bootStatus='maintenance'` state machine, `toMaintenance()`, `MaintenancePollInit`, `withGateTimeout`. E.g. `2026-05-28-oglasino-expo-boot-redesign-4.md` (34), `2026-05-29-oglasino-expo-boot-redesign-11.md` (28), `audit-picker-seam.md` (28), `2026-05-29-oglasino-expo-service-error-contract-4.md` (25), `2026-05-28-oglasino-expo-boot-redesign-5.md` (22), `audit-expo-structural.md` (19), and ~20 more.

(The fan-out enumerated ~40 of the 65 files individually; the remaining lower-count files follow the same two patterns. None are editable, so per-line enumeration adds nothing.)

---

## 8. For Mastermind / Igor — observations and open questions

1. **`infra/oglasino-devops-blueprint-v5.md` is the biggest decision, not just the biggest edit (35 hits).** It contains a full embedded worker source listing, CI workflow YAML, and droplet scripts all on the single-key model. **Is v5 a live operational runbook the sweep must update, or a frozen historical planning blueprint?** If live → it needs the largest edit of any doc. If historical → leave it and rely on `infra/cloudflare/maintenance.md` as the authoritative live runbook. I did not edit it; I need your call. (Note its worker listing also predates `use.backend.check`, which the current runbook documents — another sign it may be stale-by-design.)

2. **`decisions.md` 2026-05-15 → supersede, don't rewrite.** Recommend a new dated entry recording the two-key move + KV-as-sole-store + backend deletion + the `MaintenancePageController` trust-boundary closure, back-referencing 2026-05-15. Confirm before I (in a future briefed session) apply.

3. **`issues.md` is append-only.** Recommend: append `> Resolved by expo-maintenance-split` to the open 2026-05-29 redundancy entry; **leave** the closed 2026-05-15 router-worker findings (924/935) untouched as accurate records. The subagent's "update all 11" overstates the work.

4. **Two specs explicitly hand the poll-vs-edge question to this feature.** `features/expo-service-error-contract.md:81` ("Φ4 keeps the backend maintenance poll ... do not touch the maintenance gate's existing behavior") and `issues.md` 2026-05-29 both defer to a future Mastermind/mobile chat — which is exactly `expo-maintenance-split`. When the split ships, both should get resolution notes so the deferral chain closes.

5. **`features/expo-release-readiness.md` "chat I"** (the 5s maintenance-poll reduction, lines 309/371) is **absorbed** by the split, the same way Φ4 absorbed the review-reports chat. Worth striking from the A–I queue when the split ships.

6. **Config-file impact of *this* session: none.** This is a read-only audit; I wrote only `.agent/audit-maintenance-active-sweep.md`. No edits to `conventions.md`, `decisions.md`, `state.md`, `issues.md`, or any spec. All edits mapped above are for a future briefed sweep, after Mastermind resolves Q1–Q3.

---

## Appendix — patterns to grep in the closing sweep

To re-find the live-doc edit targets quickly (excludes frozen archives):

```
find . -name "*.md" -not -path "./sessions/*" -not -path "./.agent/*" \
  -exec grep -in "maintenance\.active\|/maintenance/active\|/maintenance/toggle\|maintenance flag\|maintenance gate" {} +
```

The two new keys to introduce: `maintenance.web.active`, `maintenance.backend.active`.
