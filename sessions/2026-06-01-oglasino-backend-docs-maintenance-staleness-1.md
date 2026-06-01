# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-06-01
**Task:** AUDIT (read-only) — Stale `maintenance.active` references in `oglasino-backend/docs/` (01/02/04/06/11). Findings only; change nothing.

---

## Implemented

Read-only audit. Nothing changed on disk except this session summary.

**Citation reliability note.** The `Read` tool returned corrupted/truncated
deliveries several times this session (short partial files, and one run that
returned a single line repeated ~100×). Every `file:line` below is therefore
cross-checked against a deterministic `grep -n` across `docs/`, and I only cite
line numbers the grep confirms. Where a read was unreliable I say so rather than
guess.

The authoritative two-flag facts trace to **two** sources, which agree:
`../oglasino-docs/infra/cloudflare/maintenance.md` (the live runbook) and —
importantly — `docs/12-deployment.md` in **this** repo, which was already
migrated to the two-flag model (grep-confirmed below). So corrections to the
five stale pages can point at an in-repo source, not only the sibling runbook.

### The two-flag reality (authoritative)

- The single `maintenance.active` KV key **no longer exists**. The worker reads
  `maintenance.web.active` and `maintenance.backend.active` (plus
  `admin.bypass.disabled` and the pre-existing `use.backend.check`).
- A **backend** deploy/operation sets **both** maintenance flags to `"true"`
  (plus `admin.bypass.disabled=true`). Confirmed in-repo at
  `12-deployment.md:87` (`maintenance-on` job) and `12-deployment.md:73–74`
  (`maintenance-off` sets both back), and the `maintenance-on.sh` /
  `maintenance-off.sh` droplet scripts at `12-deployment.md:123/174/186`.
- Restore is a **three-step manual** procedure per the runbook
  (`admin-bypass-allow.sh` → refresh caches → `maintenance-off.sh`).

**Consequence:** every doc that writes `.../values/maintenance.active` via a raw
`curl` targets a key the worker no longer reads → **functionally broken** (would
not engage/clear maintenance). References that go through the
`maintenance-on.sh` / `maintenance-off.sh` **scripts** are **fine** — those
scripts already set/clear both flags (per `12-deployment.md`).

---

## Findings — per page

Severity key: **broken** = a raw-`curl` write to the dead single key (won't
work); **stale-name** = a prose/diagram mention of the old key (misleading, not
executable); **script-ok** = goes through the two-flag-aware scripts, not stale.

### `01-backend-pre-lunch-tasks.md` — "Pre-launch tasks" (360 lines)

**Classification: (b) INCIDENTAL** — a first-`dev`→`main` launch checklist;
maintenance is task #5 (an E2E worker-contract test) plus deploy steps #12/#13.

Stale references (grep-confirmed):

- `:155` — "doesn't honour `maintenance.active=true` correctly … leaving the
  site **in maintenance mode** until you flip it back manually" (**stale-name** +
  single-step-restore implication)
- `:162` — curl PUT `…/values/maintenance.active … --data "true"` (**broken**)
- `:176` — curl PUT `…/values/maintenance.active … --data "false"` (**broken**)
- `:292` — workflow step "**maintenance-on** — PUT `maintenance.active=true` to
  Cloudflare KV" (**stale-name**; also wrong — a backend deploy now sets **both**
  flags + `admin.bypass.disabled`, per `12-deployment.md:87`)
- `:327` — curl PUT `…/values/maintenance.active … --data "false"` (**broken**)
- `:352` — checklist line "Cloudflare KV namespace contains `maintenance.active`
  key; router Worker reads it" (**stale-name**)
- Conceptual single-flag, no key, low priority: `:46`, `:54`, `:294`, `:302`.

**Disposition: CORRECT IN PLACE (+ pointer).** Genuine launch-specific value
(merge dry-run, secrets, rollback). Fix the broken curls and the stale names; for
§5's manual test, point at `12-deployment.md` / the runbook for the full two-flag
flip rather than re-documenting it (Part 4a).
**Source available: YES** (runbook + `12-deployment.md`).

### `02-deploy-checklist.md` — "Pre-Deployment Checklist" (737 lines)

**Classification: (b) INCIDENTAL, operationally central** — the page is the full
deploy procedure; **Phase 7 ("Maintenance flow rehearsal", lines ~465–514) is a
maintenance-dedicated section**, and maintenance recurs in Phases 9–10 and the
Quick reference.

Stale references (grep-confirmed):

- `:154` — Phase 2 Cloudflare check: "KV namespace bound as `CONFIG` with two
  keys: `maintenance.active=false`, `use.backend.check=false`." **stale-name AND
  wrong count** — the maintenance keys are now `maintenance.web.active` +
  `maintenance.backend.active`, alongside `admin.bypass.disabled` and
  `use.backend.check`.
- `:475` — Phase 7 curl PUT `…/values/maintenance.active … --data "true"`
  (**broken**)
- `:509` — Phase 7 "change `--data "true"` to `--data "false"`" (single-key flip)
- `:578` — Phase 9 curl GET `…/values/maintenance.active` to read state
  (**broken**/stale-name; surrounding "the workflow does NOT flip it off" is
  conceptually correct)
- `:620` — Phase 10 "run `/opt/oglasino/scripts/maintenance-off.sh` … or the curl
  from Phase 7 with `--data "false"`" (**script-ok**, but the curl alternative is
  broken; and presented as a single-step off, omitting the 3-step restore)
- `:706` — Quick-reference curl PUT `…/values/maintenance.active … --data "true"`
  (**broken**)

**Disposition: CORRECT IN PLACE.** Operator-facing; must reflect the two-flag
keys (esp. `:154`) and either flip via the scripts or via both-key curls. Trim
the restore detail to a pointer to `12-deployment.md` / the runbook (Part 4a).
**Source available: YES.**

### `04-db-reset-runbook.md` — "Database Reset Runbook" (497 lines)

**Classification: (b) INCIDENTAL** — maintenance brackets a DB reset (Step 1
enter, Step 9 exit).

References (grep-confirmed):

- `:81` — Step 1 `/opt/oglasino/scripts/maintenance-on.sh` (**script-ok** — the
  script sets both flags; `12-deployment.md:120–131` documents exactly this
  "entering maintenance manually outside a deploy" case)
- `:310` — Step 9 `/opt/oglasino/scripts/maintenance-off.sh` (**script-ok**)
- `:460–:461` — quick-ref `maintenance-on.sh` (**script-ok**)
- `:487` — quick-ref `maintenance-off.sh` (**script-ok**)
- `:445` — Troubleshooting "stuck ON" fallback: curl PUT
  `…/values/maintenance.active … --data "false"` (**broken** — the only
  functionally-broken ref in this file)
- `:40` — "Cloudflare API token available (for maintenance flip)" (conceptual,
  minor)

**Disposition: CORRECT IN PLACE — small.** Earlier-in-session I suspected this
page was "blocked" because the runbook has no `maintenance-on.sh`; that was
**wrong** — `12-deployment.md:120–131` documents `maintenance-on.sh` setting both
flags for exactly the ad-hoc (non-deploy) case a DB reset needs. So 04 is nearly
correct: only the `:445` fallback curl needs fixing (flip **both** keys, or
replace with a pointer to the script).
**Source available: YES** (`12-deployment.md` covers the ad-hoc-entry case).

### `06-architecture.md` — "Architecture" (250 lines)

**Classification: (b) INCIDENTAL** — architecture overview.

Stale references (grep-confirmed, all **stale-name**, harmless at runtime):

- `:30` — mermaid KV node `kv[("KV<br/>maintenance.active<br/>config.*")]`
- `:104` — "**Maintenance flip.** Reads `maintenance.active` from KV; if `true`,
  returns a maintenance page (HTTP 503)…"
- `:150` — request-flow sequence "`W->>KV: GET maintenance.active (cached, TTL
  30s)`"
- Generic, **not** stale (no key name): `:186` (data-ownership "Maintenance flag
  | Cloudflare KV"), `:203`, `:240`.

**Disposition: CORRECT IN PLACE.** Update the KV node + the two read-path
mentions to the two-flag model (or `maintenance.*` + a pointer). Trivial.
**Source available: YES.**

### `11-environment-variables.md` — "Environment Variables" (266 lines)

**Classification: (b) INCIDENTAL** — env-var reference.

Stale references (grep-confirmed, **stale-name**):

- `:212` — `| CF_API_TOKEN | deploy-backend.yml (maintenance-on) | … flip
  maintenance.active in KV |`
- `:214` — `| CF_KV_NAMESPACE_ID | … KV namespace containing maintenance.active |`
- Generic, **not** stale: `:136` (`CLOUDFLARE_KV_CONFIG_TOKEN` "config +
  maintenance").

**Disposition: CORRECT IN PLACE.** Update `:212`/`:214` to the two keys.
**Source available: YES** for the key names.

---

## Summary table

| Page | Lines | Class | Functionally-broken refs | Disposition |
| ---- | ----- | ----- | ------------------------ | ----------- |
| 01-backend-pre-lunch-tasks.md | 360 | INCIDENTAL | :162, :176, :327 (+ :155/:292/:352 stale-name) | CORRECT IN PLACE (+pointer) |
| 02-deploy-checklist.md | 737 | INCIDENTAL (Phase 7 maint-dedicated) | :475, :578, :706 (+ :154 wrong keys, :620 curl alt) | CORRECT IN PLACE |
| 04-db-reset-runbook.md | 497 | INCIDENTAL | :445 only (scripts are fine) | CORRECT IN PLACE (small) |
| 06-architecture.md | 250 | INCIDENTAL | none (all prose/diagram :30/:104/:150) | CORRECT IN PLACE |
| 11-environment-variables.md | 266 | INCIDENTAL | none (:212/:214 prose) | CORRECT IN PLACE |

All five → CORRECT IN PLACE. None is a wholesale REPLACE-WITH-POINTER candidate
(each carries real non-maintenance backend content); none is a DELETE candidate.

Brief item-1 cross-check (grep-confirmed across **all** `docs/`): **no** page
references the "old maintenance DB config", a `/public/maintenance/active` poll,
`MaintenanceController`, or `MaintenanceService`. The only staleness is the
single `maintenance.active` KV key (and, in 02, the single-step-restore /
two-keys framing).

Item-4 (source for CORRECT-IN-PLACE): satisfied for all five — the two-flag
facts are available **in-repo** at `12-deployment.md` and in the sibling runbook.

---

## Files touched

- `.agent/2026-06-01-oglasino-backend-docs-maintenance-staleness-1.md` (new — this findings doc)
- `.agent/last-session.md` (overwritten — exact copy)

No `docs/` files changed (read-only audit).

## Tests

- None run. Read-only docs audit; no code touched.

## Cleanup performed

- none needed (no code or docs edited).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change.
- issues.md: **no change made by me.** Recommended amendment to the 2026-05-30
  entry drafted in "For Mastermind" (a Docs/QA edit, not mine).

## Obsoleted by this session

- Nothing. The audit identifies stale docs but does not change them; the obsolete
  references remain on disk for a future correction brief.

## Conventions check

- Part 1 (docs style): confirmed — no new `docs/` files created.
- Part 4 (cleanliness): confirmed — nothing edited.
- Part 4a (simplicity): see "For Mastermind."
- Part 4b (adjacent observations): see "For Mastermind."
- Part 6 (translations): N/A this session.
- Other parts: Part 8 (edge boundary) — the audit confirms five docs lag the
  "maintenance lives at the edge worker" principle that `12-deployment.md`
  already reflects.

## Known gaps / TODOs

- The `maintenance-on` CI job-name parenthetical at `11:212` was not verified
  against the actual `deploy-backend.yml` (out of a docs audit's read-scope).

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit.
  - Considered and rejected: REPLACE-WITH-POINTER for 02/04 (strip the
    maintenance prose, point at `12-deployment.md`) — rejected because both pages
    interleave maintenance steps with non-maintenance operational steps, so a
    clean strip isn't possible; CORRECT IN PLACE with the restore detail trimmed
    to a pointer is the minimal change.
  - Simplified or removed: nothing.

- **Scope note (broader than the issues.md framing):** the 2026-05-30 entry says
  the pages "still reference the single `maintenance.active` KV key." True, but
  `02` is additionally stale on the **restore model** (Phase 10 single-step off;
  the real procedure is the 3-step admin-bypass → cache-refresh → maintenance-off)
  and on the **key inventory** (`:154` says "two keys"). A key-name-only fix would
  leave those wrong.

- **Positive finding:** `12-deployment.md` is **already** migrated to the
  two-flag model (grep-confirmed: `:73–74`, `:87`, scripts at `:123/:174/:186`,
  and the "maintenance-on / human-verify / maintenance-off" section that
  documents `maintenance-on.sh` setting both flags + `admin.bypass.disabled`).
  A clean re-read of `:104–:233` confirmed the section is intact (the
  `maintenance-off.sh` script writes both `maintenance.backend.active=false` and
  `maintenance.web.active=false` at `:126–:127`). It is the in-repo authoritative
  reference; the five stale pages should point at it. (`13-database-migrations.md:56/:294` and `15-troubleshooting.md:67/:140`
  already correctly cross-ref `12-deployment.md` and the scripts — not stale.)

- **Adjacent observations (Part 4b):**
  1. `11-environment-variables.md` — possible **non-maintenance** staleness:
     `11:136` (`CLOUDFLARE_KV_CONFIG_TOKEN` scoped for KV reads/**writes**) and
     `06:109–110` ("optionally read by the backend via `CLOUDFLARE_KV_CONFIG_TOKEN`").
     After the maintenance split, maintenance KV writes are CI/droplet-script
     driven (edge-only). Whether the backend retains any runtime KV-write
     capability is unconfirmed; if not, the write-scope wording is stale.
     Severity: low–medium. Out of this read-only docs audit's scope to confirm
     (needs a backend config/code read).

- **Drafted `issues.md` amendment (for Docs/QA to apply, not me):** amend the
  2026-05-30 entry to add: "Audited 2026-06-01. Grep-confirmed stale
  `maintenance.active` refs: `01` (:155, :162, :176, :292, :327, :352), `02`
  (:154, :475, :509, :578, :620, :706), `04` (:445 only — its `maintenance-on/off.sh`
  script steps are already two-flag-correct), `06` (:30, :104, :150), `11`
  (:212, :214). All are CORRECT-IN-PLACE; the authoritative two-flag source is
  in-repo at `12-deployment.md` (already migrated) plus
  `infra/cloudflare/maintenance.md`. `02` is additionally stale on the
  single-step-restore framing and on `:154`'s 'two keys' inventory. No page
  references a maintenance DB config or `/public/maintenance/active` poll.
  Possible follow-up: whether `CLOUDFLARE_KV_CONFIG_TOKEN`'s runtime KV-**write**
  scope (`11:136`) is still real post-split." (Closure gate: draft only; no
  config file edited this session.)

- **Suggested next step:** one backend correction brief covering 01/02/04/06/11,
  pointing corrections at `12-deployment.md` + the runbook; `04` is a one-line
  fix (`:445`).
