# Expo Boot Redesign

Canonical spec for the `oglasino-expo` cold-start boot rearchitecture. Replaces the current `AppContext.bootstrap` / `apiStore` barrier / `AppVersionConfigInit` render-gate tangle with a single explicit boot state machine. Adopts version-checksum-driven freshness so the cold-start request storm is eliminated.

This spec is authoritative per conventions Part 3. Where it disagrees with any sibling-repo doc or prior session note, this spec wins.

**Status at authoring:** planned. Backend prerequisites (B1, B2) must land before the Expo state machine is built.

**Originating context:** the Φ3 cold-start boot-loading hang ([`../decisions.md`](../decisions.md) 2026-05-28 Entry C). Two read-only audits (`oglasino-backend` and `oglasino-expo`, both `.agent/audit-expo-boot-redesign.md`, 2026-05-28) established the real contracts and the real failure mode. The hang was misdiagnosed as a transport problem; the backend access log proved all four boot requests returned 200. The actual cause is an over-parallel boot (4 boot calls + 19 translation calls firing on every cold start) with a single undifferentiated failure bucket that routes any failure — or any hang — to `maintenance` or to nothing.

---

## Part 1 — The three loop-prevention invariants

These exist because the prior implementation produced a boot→loading→boot→loading death spiral in which the store cleared on each pass. The root pattern: an effect whose dependency array included `status`, so the machine re-triggered itself, and re-entry destroyed established state. These three invariants make that loop impossible by construction. They are non-negotiable and every brief inherits them.

### Invariant 1 — One effect, empty deps

The boot path contains exactly **one** `useEffect`. It has an empty dependency array (`[]`). It runs once on mount. It calls `start()`. No effect anywhere in the boot path lists `status`, `selectedBaseSite`, codes, or any machine-owned value in its dependency array. The machine advances by gate functions calling the next transition on completion — never by an effect reacting to a state change the machine itself produced.

### Invariant 2 — The machine writes status; the view reads status; nothing else touches it

`status` is an output of the machine, consumed only by the renderer to decide which screen to show. `status` is never an input that re-triggers boot. Render reads status. Boot writes status. They do not cross. Re-entry into the machine is always an explicit `start()` call from a named event (maintenance-clear, manual retry) — never a reactive dependency.

### Invariant 3 — Re-entry never destroys progress

`start()` clears nothing. Stored state (base site, codes, translations, checksums) is treated as valid until a gate proves a specific slice stale and replaces that slice only. Clearing stored state is an explicit user action (logout) or a confirmed freshness replacement — never a boot side effect. Re-entering the machine because maintenance cleared must not wipe the base site, must not reset codes, must not clear any store. Each re-entry pass either makes progress or sits at a terminal screen awaiting an external event. No pass re-clears and re-decides the same thing.

### The structural defense behind the invariants

The boot store is a single standalone module. It does **not** import `authStore` or any chat store. It cannot be caught in the `authStore` ↔ chat-store require cycle (confirmed live in the Expo audit §5.1), so it cannot read an uninitialized value at cold-start module-eval order and cannot self-clear as a consequence. The `globalThis`-pinned barrier in `apiStore.ts` is removed; the boot store is a leaf.

---

## Part 2 — The boot state machine

Boot is a sequence of gates. Each gate must pass before the next runs. The portal is reachable only in the terminal `ready` state. There is no transition into a hang: every gate has a timeout, and every timeout has a defined destination.

### States

| `status` | Screen rendered | Meaning |
| --- | --- | --- |
| `booting` | splash | initial; entered on app start and on every re-entry. Owns no data, no fetch, no logic beyond kicking the first gate. The only legitimate pre-gate loading state (the async storage read happens here). |
| `maintenance` | maintenance screen | maintenance is on, OR a gate could not reach the backend within its timeout. Active-checker polls; on clear, re-enter `booting`. |
| `update-required` | hard-update screen | hard update required. Terminal-blocking. Only exit is the user updating the app binary. |
| `intro-picker` | intro + base-site picker | no stored base site. Picker fetches the base-site list (scoped to this screen, own spinner). |
| `updating` | splash | a freshness check found a stale slice; re-fetching it before proceeding. |
| `ready` | **portal** | all gates passed. Stored, fresh base site exists. Portal mounts only here. |

`soft update available` is **not** a status. It is a boolean flag set during the version gate and surfaced as a non-blocking modal over whatever renders. It never blocks boot.

### The gate sequence

```
                         ┌─────────────┐
            ┌───────────▶│  BOOTING    │  (initial; entered on app start
            │            │  (splash)   │   and on every re-entry via start())
            │            └──────┬──────┘
            │                   │ start()
            │                   ▼
            │         ╔═══════════════════╗
            │         ║ GATE 1: MAINTENANCE║
            │         ║ GET /maintenance/  ║
            │         ║ active (timeout 5s)║
            │         ╚═════════╤═════════╝
            │   active=true     │   active=false
            │   OR unreachable  │   AND reachable
            │           ▼       │
            │    ┌────────────┐ │
            │    │ MAINTENANCE │ │   ◀── active-checker polls on interval;
            │    │  (screen)   │─┘        on clear → start() (re-enter BOOTING)
            │    └────────────┘
            │                   ▼
            │         ╔═══════════════════╗
            │         ║ GATE 2: VERSION    ║
            │         ║ GET /app/version/  ║
            │         ║ {platform}         ║
            │         ║ (timeout 5s)       ║
            │         ╚═════════╤═════════╝
            │   forceUpdate     │ ok / optionalUpdate
            │        ▼          │ (optional → set softUpdate flag, continue)
            │ ┌──────────────┐  │
            │ │UPDATE_REQUIRED│  │   ◀── terminal-blocking; only exit
            │ │  (screen)    │  │        is the user updating the binary
            │ └──────────────┘  │
            │                   ▼
            │         ╔═══════════════════╗
            │         ║ GATE 3: BASE SITE  ║
            │         ║ read stored site   ║
            │         ║ (async storage)    ║
            │         ╚═════════╤═════════╝
            │     no stored     │   stored site
            │       site        │   exists
            │         ▼         │
            │  ┌─────────────┐  │
            │  │ INTRO_PICKER │  │   GET /baseSite/overviews
            │  │  (screen)    │  │   (scoped to screen, own spinner)
            │  └──────┬──────┘  │
            │  user picks site  │
            │  GET /baseSite/   │
            │  {code}, store    │
            │  full DTO + codes │
            │         │         │
            │         └────┬────┘
            │              ▼
            │    ╔═══════════════════════╗
            │    ║ GATE 4: FRESHNESS      ║
            │    ║ GET /versions          ║
            │    ║ compare catalog +      ║
            │    ║ translation checksums  ║
            │    ║ (timeout 5s)           ║
            │    ╚═══════════╤═══════════╝
            │   stale slice  │  all current
            │     ▼          │
            │ ┌─────────┐    │
            │ │ UPDATING │    │   re-fetch ONLY the stale slice(s):
            │ │ (splash) │    │   stale catalog → GET /baseSite/{code}
            │ └────┬────┘    │   stale namespace(s) → GET /translations
            │      └────┬────┘   per stale namespace; store new checksums
            │           ▼
            │     ┌──────────┐
            │     │  READY   │ ──▶ PORTAL mounts (only here)
            │     └──────────┘
            │
            └──── any gate hits timeout/unreachable ───▶ MAINTENANCE
                  (per the rule: backend unreachable = maintenance)
```

### Transition table

| From | Trigger | To | Notes |
| --- | --- | --- | --- |
| (app start) | mount effect calls `start()` | `booting` | one effect, empty deps |
| `booting` | `start()` begins gate 1 | (gate 1 runs) | |
| gate 1 | `maintenance.active === true` OR unreachable/timeout | `maintenance` | fail-closed |
| gate 1 | `maintenance.active === false` AND reachable | (gate 2 runs) | |
| `maintenance` | active-checker poll returns false | `start()` → `booting` | explicit re-entry; non-destructive |
| gate 2 | `forceUpdate === true` | `update-required` | terminal-blocking |
| gate 2 | `optionalUpdate === true` (and not force) | (gate 3 runs) | set `softUpdate` flag; continue |
| gate 2 | neither | (gate 3 runs) | |
| gate 2 | timeout/unreachable | `maintenance` | backend-down rule |
| gate 3 | no stored base site | `intro-picker` | |
| gate 3 | stored base site exists | (gate 4 runs) | stored DTO trusted until gate 4 proves stale |
| `intro-picker` | user picks site; `GET /baseSite/{code}` succeeds; store DTO + `setCodes` | (gate 4 runs) | |
| `intro-picker` | `GET /baseSite/overviews` fails/times out | `maintenance` | empty picker is never shown; unreachable = maintenance |
| gate 4 | all checksums match local | `ready` | |
| gate 4 | one or more slices stale | `updating` | |
| gate 4 | timeout/unreachable | `maintenance` | backend-down rule |
| `updating` | stale slices re-fetched and stored | `ready` | |
| `updating` | re-fetch fails/times out | `maintenance` | backend-down rule |
| `ready` | active-checker detects maintenance on | `start()` → `booting` | runtime maintenance flip; re-entry |

### Timeout

Per-gate timeout `T = 5000ms`. Shorter than the current axios default (8000ms) so worst-case time-to-maintenance is bounded and humane. Backend-unreachable routes to `maintenance` fast. Every gate that can't reach the backend within `T` transitions to `maintenance` — one rule, applied uniformly, not per-gate error handling.

### The single backend-down rule

Any gate that cannot reach the backend within its timeout transitions to `maintenance`. This generalizes the stated maintenance rule ("backend not active → maintenance") to every gate, because an unreachable backend at any gate means the same thing and should produce the same screen. There is no separate "network error" state.

---

## Part 3 — The version model (Gate 2)

### Two facts, two sources

- **Running version** — `Application.nativeApplicationVersion`. Baked into the binary at build time. The app knows it; no backend involved.
- **The floor** (`minSupportedVersion`) — the minimum version still allowed. A deliberate policy decision, set rarely (only when a release genuinely breaks old clients). Source: **admin panel**, human-set, per platform.
- **The ceiling** (`latestVersion`) — the newest version available. Changes on every release. Source: **derived from the build** via an EAS post-build hook, never hand-typed.

### The classification rule

Computed server-side in `AppVersionController`:

```
forceUpdate    = running < minSupportedVersion          (below the floor → HARD, block)
optionalUpdate = running < latestVersion && !forceUpdate (below ceiling, above floor → SOFT, notify)
neither        = running >= latestVersion                (current → no prompt)
```

The major version number does **not** trigger a force by itself. Force is driven only by the running version being below the floor. Shipping a new major is a soft event until the operator deliberately raises the floor.

### Two independent write paths

The floor and ceiling are two fields on the same per-platform `AppVersionConfig` row. They are written by two separate, non-overlapping surfaces so neither can clobber the other:

- **Ceiling write** — machine-driven. The EAS post-build hook POSTs the just-built version to a ceiling-only update endpoint on every build. The operator never touches it.
- **Floor write** — human-driven. The admin panel POSTs to a floor-only update endpoint. Rare. The only version value the operator ever types, and only when forcing old clients off.

Split write surface means the EAS hook can never lower the floor, and the admin can never un-set the ceiling.

### Per-platform

`android` and `ios` each have their own floor and ceiling (the `AppVersionConfig` is keyed by `platform`). A breaking change may land on one platform before the other. The EAS hook writes the ceiling for whichever platform it just built. The floor is set per-platform in admin.

### Operator flows

**Normal release (patch/minor):** bump version, push, EAS builds, ceiling auto-set. Existing users get a soft nudge. Operator does nothing else.

**Major release:**
1. Bump to e.g. `3.0.0`, push, EAS builds, ceiling auto-set to `3.0.0`. All `2.x` users get a soft "update available" nudge. *(automatic)*
2. Wait. Let adoption climb. Watch `3.0.0` health. *(operator does nothing)*
3. When confident and when old clients must be forced off, open admin, set floor = `3.0.0`. Remaining `2.x` users now hard-blocked. *(one deliberate action, one place)*

### Dev vs prod — difference lives in data, not code

The controller body is uncommented and identical in every environment. The dev/prod difference is the seeded floor/ceiling values, not an `if (dev)` branch and not commented-out code.

- **Dev:** seed `AppVersionConfig` with `minSupportedVersion: 0.0.0` and `latestVersion: 0.0.0` (or the dev binary's reported version). Floor `0.0.0` → nothing is ever below it → `forceUpdate` never fires. Ceiling at-or-below running → `optionalUpdate` never fires. The gate runs, the contract is real, the endpoint returns a proper `AppVersionResponseDTO`, and always says "you're fine." This is what unblocks local testing without commenting out the controller.
- **Prod:** floor is admin-set; ceiling auto-syncs from EAS. The endpoint computes the two booleans.

The state machine never knows or cares which environment it is in. It reads the booleans.

### Store deep-link (out of scope for boot, tracked separately)

The current hard/soft update buttons open the placeholder URL `https://memento-tech.com` (Expo audit obs #4). The real App Store / Play Store deep-link is a pre-launch task, not a boot-architecture concern. Out of scope for this feature; tracked separately.

---

## Part 4 — The freshness model (Gate 4)

### One call answers everything

`GET /api/public/versions` returns both maps in one response:

```json
{
  "translations": { "COMMON": "<16-hex>", ... 22 namespaces },
  "catalog": { "rs": "<16-hex>", "rsmoto": "<16-hex>", "me": "<16-hex>" }
}
```

One call tells the client whether its stored catalog AND its stored translations are stale. No separate calls. This is the lever that kills the cold-start request storm.

### The compare-and-refetch flow

1. Gate 4 calls `GET /versions` once.
2. For the selected base site's code, compare the returned `catalog[code]` to the locally-stored catalog checksum. If different → re-fetch the catalog via `GET /baseSite/{code}`, store the new `BaseSiteDTO`, store the new checksum.
3. For each of the 22 translation namespaces, compare the returned checksum to the locally-stored checksum. For each mismatch → re-fetch that namespace via `GET /translations?namespace=<NS>&lang=<lang>`, store it, store the new checksum.
4. If nothing is stale, proceed straight to `ready`. A returning user whose content has not changed fetches **zero** namespaces and **zero** catalog. The 19-namespace-every-boot storm is gone.

### Granularity

- Translation checksums are per-namespace, collapsed across all languages. A namespace's checksum changes if any key/value in that namespace changes in any language. There is no per-namespace-per-language checksum.
- Catalog checksums are per-base-site. They cover catalog structure and key references; the strings the keys point to live in the translation namespaces and are covered by translation checksums.

### Empty-checksum sentinel

A checksum value of `""` (empty string) from the backend means not-yet-computed (a narrow boot window on the server). The client treats empty as "re-fetch this slice." Empty is never treated as "matches."

---

## Part 5 — Storage model

The boot store and AsyncStorage hold exactly what the gates need and nothing version-blind.

### Persisted keys (AsyncStorage)

| Key | Holds | Written when |
| --- | --- | --- |
| `base_site` | full `BaseSiteDTO` (incl. catalog) for the selected site | on pick; on catalog re-fetch in gate 4 |
| `base_site_catalog_checksum` | the catalog checksum for the stored base site | alongside `base_site`, every time it is written |
| `app_language` | full `LanguageDTO` for the active language | on first resolve; on language change |
| `translations_<NS>` | cached translation payload for namespace `NS` | on namespace fetch / re-fetch in gate 4 |
| `translations_checksum_<NS>` | the checksum for namespace `NS` | alongside `translations_<NS>` |

The two inert keys in the current code (`translations_version` read-then-discarded; `translations_<lang>` cache never wired) are replaced by the per-namespace checksum scheme above. The dead `getStoredVersion`/`setStoredVersion`/`getTranslations`/`setTranslations` helpers (Expo audit §2.3, §2.4) are removed.

### Co-storage rule

A checksum is always written in the same operation as the payload it describes. The stored catalog and its checksum are written together; a namespace payload and its checksum are written together. This guarantees the stored checksum always describes the stored payload — there is no window where they disagree.

### The honest first paint

Reading `base_site` is async. So there is one unavoidable instant on launch before the machine knows which world it is in. That instant is `booting`. It owns no data and no fetch. It exists only while `start()` does its first synchronous setup and kicks Gate 1. The moment `booting` acquires logic, it becomes a place to hang — so it stays empty by rule.

---

## Part 6 — Backend prerequisites

Both are pure-backend, both land before the Expo state machine is built (conventions Part 10: backend before web/mobile). Both are small.

### B1 — Allowlist `/api/public/versions` for `X-Lang`

`GET /api/public/versions` currently returns `400 LANG_MISSING_OR_INVALID` when called without a resolvable `X-Lang`, because it is not on the `CurrentLanguageFilter` allowlist (backend audit §5, high-severity finding). At cold start, before a base site is picked, the app has no language, so Gate 4 would 400. The endpoint's response is language-independent — the filter requirement is incidental.

Fix: add `/api/public/versions` to `CurrentLanguageFilter.ALLOWLIST_EXACT` (exact, not prefix — single endpoint). One line. Confirmed safe by Igor.

### B2 — Uncomment `AppVersionController` and wire the version model

`AppVersionController.checkVersion` currently returns an empty `200` (body commented out). The implementation exists; it was commented out to dodge a dev-testing annoyance (no valid published version, versions never matched). That dev problem is solved by the dev-seed-`0.0.0` data approach (Part 3), not by commenting out the controller.

B2 delivers:

1. Uncomment `AppVersionController.checkVersion`. It returns `AppVersionResponseDTO(latestVersion, minSupportedVersion, forceUpdate, optionalUpdate)` computed per the Part 3 classification rule.
2. Seed dev `AppVersionConfig` rows (per platform) with floor `0.0.0` and ceiling `0.0.0` so the gate is a real-contract no-op in dev.
3. Split the write surface: a **ceiling-only** update endpoint (for the EAS hook) and a **floor-only** update endpoint (for admin). The existing `/internal/app/version/update` is repointed or split so the EAS hook cannot lower the floor and admin cannot un-set the ceiling. Per platform.
4. The EAS post-build hook: on build of each platform, POST the just-built version as that platform's ceiling. Added to the existing EAS workflow YAMLs.

Adjacent backend findings surfaced by the audits but **out of scope** for this feature (route to `issues.md` for separate triage): `BaseSite not found` throwing raw `RuntimeException` → 500 instead of 404 (backend audit §4, medium); `AppVersionController` packaged under `admin.internal.controller` despite a `/api/public` mapping (cosmetic).

---

## Part 7 — What is torn out

The redesign deletes, not works around, the current boot tangle. The Expo audit confirms each item exists.

- `AppContext.bootstrap`'s `Promise.all([checkIfMaintenance, getAppConfiguration, fetchBaseSites])` and its outer catch-all-to-`maintenance` try/catch (audit §1.1, obs #6).
- The `globalThis`-pinned barrier in `apiStore.ts` and `waitForBootstrap()` (audit §1.3, §5.2). The state machine's sequential gates replace it; no request needs a barrier because no request fires before its gate.
- `AppVersionConfigInit` as a competing render-gate above the tree (audit §1.1). Its soft/hard classification logic moves into Gate 2 of the machine. Its dismissal/modal UI for the soft case is retained as the `softUpdate` flag's view, but it is no longer a boot gate.
- The 19-namespace `loadAllNamespaces` burst on every cold start (audit §2.3). Replaced by checksum-driven per-namespace refetch (Part 4).
- The inert version/cache helpers (`getStoredVersion`, `setStoredVersion`, `getTranslations`, `setTranslations`, the commented scaffolding in `loader.ts`) (audit §2.3, §2.4).
- The no-op `setStoredBaseSite` re-write on every cold start (audit obs #5).
- The live `[BOOT]` `console.log` in `api.ts:43-51` request interceptor (audit §5.4). Removed as part of the redesign, per the standing `issues.md` 2026-05-28 cleanup obligation.

The `<RequireBaseSite>` per-screen render gate (audit §5.3) is reviewed during the redesign: with the machine guaranteeing the portal mounts only in `ready` (a state unreachable without a stored base site), per-screen gating may become redundant. The portal-mounts-only-in-`ready` invariant replaces the need for per-screen base-site guards. Whether `<RequireBaseSite>` is fully removed or kept as defense-in-depth is an implementation call made during the portal-wiring brief; the spec's position is that it should not be load-bearing.

The `authStore` ↔ chat-store require cycle (audit §5.1, "Cycle B") is **not** in scope for this feature — it is deferred to chat B (mobile messaging adoption) per [`../decisions.md`](../decisions.md). The boot store's independence from it (Part 1, structural defense) is what makes the cycle irrelevant to boot. The redesign does not touch the cycle; it routes around it.

---

## Part 8 — Build order

Sequential. Each step is its own engineer session with its own summary. Backend first.

1. **Backend B1** — allowlist `/versions`. One line + a test that `/versions` returns 200 without `X-Lang`.
2. **Backend B2** — uncomment `AppVersionController`; classification logic; per-platform dev seed (`0.0.0`/`0.0.0`); split floor/ceiling write endpoints; EAS post-build hook for ceiling. Tests for force/soft/neither classification and for the two write paths.
3. **Expo — boot store** — the standalone state-machine store: `status`, `softUpdate` flag, the gate functions, the transition logic. No imports of `authStore` or chat stores. One mount effect with empty deps calling `start()`. Unit-test the transition table.
4. **Expo — gates 1+2** — maintenance gate and version gate wired into the store, with timeouts and the backend-down-to-maintenance rule. `softUpdate` modal retained as view.
5. **Expo — gate 3** — base-site gate: async storage read; intro/picker path (`/baseSite/overviews`); pick → `/baseSite/{code}` → store DTO + checksum + `setCodes`.
6. **Expo — gate 4** — freshness gate: `/versions` compare; slice-only refetch of stale catalog and stale namespaces; co-stored checksums.
7. **Expo — portal wiring + teardown** — root layout switches on `status`; portal mounts only in `ready`; remove the `Promise.all`, the `globalThis` barrier, `AppVersionConfigInit` as a gate, the 19-namespace burst, the inert helpers, the no-op re-write, the `[BOOT]` logs; review `<RequireBaseSite>`.

### Regression test for the boot loop

Built and run before the feature closes. The exact scenario that broke the prior implementation:

> Cold start → reach `ready` → toggle maintenance on (backend) → app goes to `maintenance` → toggle off → app returns to `ready` **without clearing the stored base site** and **without a single extra boot pass beyond the one re-entry**.

If re-entry is non-destructive (Invariant 3) and non-reactive (Invariants 1 and 2), this sequence is smooth. If it loops or clears the base site, the invariants were violated and this test catches it before it ships.

---

## Part 9 — Definition of done

- All four gates run in sequence; the portal mounts only in `ready`.
- No transition leads to a hang: every gate has a 5s timeout routing to `maintenance`.
- A returning user with current content reaches `ready` with zero translation fetches and zero catalog fetch.
- The boot-loop regression test passes.
- Gate 2 returns a real `AppVersionResponseDTO` in dev (no-op by data) and computes force/soft correctly in prod.
- The EAS hook sets the ceiling on build; the admin floor write is per-platform and independent.
- Everything in Part 7 is deleted, including the `[BOOT]` instrumentation.
- The three invariants hold: one effect with empty deps; status written by machine and read by view only; re-entry destroys no progress.
- `tsc --noEmit`, lint, and the test suite pass per conventions Part 4.
- Config-file impact applied by Docs/QA: this spec on disk; a `decisions.md` entry at feature close; `state.md` status flips; the Expo backlog row for version-checksums updated (mobile now adopts `/versions`).