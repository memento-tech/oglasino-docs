# App Version Control — Documentation Audit

Read-only audit of how the mobile app-version-control feature drives the soft
("you can update") and hard ("you must update") popups. Source: the docs only.
Date: 2026-06-06.

---

## 1. Where it's documented

There is **no standalone `features/app-version-control.md`**. The mechanism is
spread across:

| File | What it covers |
| --- | --- |
| **`features/expo-boot-redesign.md`** | **Canonical source.** Part 2 (Gate 2 in the boot state machine), **Part 3 "The version model (Gate 2)"** — full model, classification rule, write paths, per-platform, operator flows, dev/prod posture, store deep-link. Part 6 (B2 backend prerequisite), Part 8 (build order), Part 9 (DoD). |
| **`decisions.md`** (2026-05-29 boot-redesign close, lines 829–868) | Confirms the model shipped; amendment #1 (seed posture: `0.0.0/0.0.0` safe-default-all-envs, `ON CONFLICT DO NOTHING`); the two pre-launch carve-outs (deep-link URLs, EAS ceiling-write smoke). Line 539 notes the per-pathname version check was dropped (now once-per-boot at Gate 2). |
| **`state.md`** Risk Watch (lines 502–503) | The two open follow-ups (EAS ceiling-write live smoke; placeholder deep-link URLs). |
| **`sessions/2026-05-28-oglasino-backend-expo-boot-redesign-2.md`** | Backend B2 implementation record — actual endpoint paths, DTOs, guard, trust-boundary note. |
| **`sessions/2026-05-28-oglasino-expo-boot-redesign-2.md`** | EAS post-build hook implementation (the ceiling writer). |
| **`sessions/2026-05-28-oglasino-backend-expo-boot-redesign-audit-1.md`** | Pre-work audit: documents the controller was a stub before B2. |

`features/version-checksums*.md` are **not** about app-version control — they
cover Gate 4 freshness (translation/catalog checksums), a separate concern. The
infra blueprint and `infra/expo/` docs do **not** describe the version-control
surface (the blueprint "ceiling" hits are unrelated).

---

## 2. The model

Three facts, two stored server-side (`expo-boot-redesign.md` Part 3):

- **Running version** — `Application.nativeApplicationVersion`, baked into the
  binary. Client-known, no backend.
- **Floor = `minSupportedVersion`** — minimum version still allowed. Human policy
  decision, set rarely. **Triggers the HARD popup.**
- **Ceiling = `latestVersion`** — newest version available. Changes every
  release, derived from the build. **Triggers the SOFT popup.**

**Classification (computed server-side in `AppVersionController`):**

```
forceUpdate    = running < minSupportedVersion            -> HARD, block
optionalUpdate = running < latestVersion && !forceUpdate  -> SOFT, notify
neither        = running >= latestVersion                 -> no prompt
```

Difference between soft and hard: **below the ceiling but above the floor = soft;
below the floor = hard.** A new major version is only ever a *soft* event until
the operator deliberately raises the floor.

**Where stored:** a per-platform `AppVersionConfig` DB row (Postgres,
`app_version_config` table, `UNIQUE (platform)` added in `V1__init_schema.sql`).
Floor and ceiling are two fields on the same row. Seeded `0.0.0/0.0.0` in all
environments (`ON CONFLICT DO NOTHING`), making Gate 2 a no-op until real values
land.

**Per-platform:** Yes — separate. `android` and `ios` each have their own floor
and ceiling (row keyed by `platform`). A breaking change can land on one platform
before the other.

---

## 3. The endpoints

| Endpoint | Method | Who calls it today | Admin-reachable per docs? |
| --- | --- | --- | --- |
| `/api/public/app/version/{platform}` | GET (READ) | Mobile app at boot, Gate 2. Returns `AppVersionResponseDTO(latestVersion, minSupportedVersion, forceUpdate, optionalUpdate)`. On the `CurrentLanguageFilter` allowlist (language-independent). | N/A (read path; mobile-consumed) |
| `/internal/app/version/ceiling` | POST (WRITE) | **EAS post-build hook** — fires on every preview/prod build, POSTs `{platform, latestVersion}` with `X-INTERNAL-TOKEN`. Writes `latestVersion` only. | Under `/internal/**`, guarded by `InternalTokenFilter` (shared-secret, bare 401). Machine-to-machine. |
| `/internal/app/version/floor` | POST (WRITE) | **Intended for the operator/admin** — POSTs `{platform, minSupportedVersion}`. Writes `minSupportedVersion` only. | Same `/internal/**` + `X-INTERNAL-TOKEN` guard. |

Notes:

- Both writes are **UPDATE-ONLY** (missing platform row → 404, never creates —
  the seed owns creation).
- Split single-field DTOs mean the EAS hook can never touch the floor and the
  admin writer can never touch the ceiling.
- The old combined `POST /internal/app/version/update` was deleted in B2.
- Both write paths fail open to a no-update DTO with WARN on unknown platform /
  unparseable semver — the gate cannot 500 the app into a maintenance lockout.

---

## 4. The gap — admin-panel VIEW to set version values manually

**A floor-write *endpoint* exists and is documented as the admin target — but no
admin-panel *UI/VIEW* is documented as built.**

- Part 3 says the floor is set via "the **admin panel** POSTs to a floor-only
  update endpoint." That describes the *intended* writer.
- The *implemented* endpoint is `POST /internal/app/version/floor`, under
  `/internal/**` and guarded by a **shared-secret `X-INTERNAL-TOKEN`** — a
  machine-to-machine endpoint hit by curl, **not** a route under
  `/api/secure/admin/*` wired to a web admin screen.
- The backend B2 session is explicit: consumers are "the EAS post-build hook (CI)
  and a **future admin panel** (which can read the status code)," and the bare
  404 was kept because it has **"no UI consumer today."**

So what exists today is the *backend endpoint* the admin VIEW would call. The
admin-facing **view itself does not appear anywhere in the docs** — no web session
wired a version-config admin screen (the only admin-panel web session,
`2026-05-19-oglasino-web-admin-extension-1.md`, is user enable/disable +
lock-from-deletion, unrelated). **Docs are silent on any web admin-panel screen
for floor/ceiling.**

Building Igor's view means a web admin screen that POSTs to
`/internal/app/version/floor`. The docs do **not** address how a browser admin
session would authenticate to an `/internal/**` shared-secret route (today it's
curl-with-token, not a logged-in admin path). That auth seam is a gap the docs
don't cover.

---

## 5. Risk-watch / known pending items

From `state.md` Risk Watch (502–503) and the `decisions.md` 2026-05-29 carve-outs:

- **[low] EAS post-build ceiling-write hook — live smoke not yet done.** First
  real write to `/internal/app/version/ceiling` happens on the next preview build
  cut; not yet verified live. The EAS-side session notes it never read backend to
  confirm the endpoint accepts the body — if backend B2 weren't live, the first
  POST would 404 to a non-fatal echo. Track until smoke-tested.
- **[low] Store deep-link URLs not wired.** `HardUpdateScreen` and
  `SoftUpdateModal` currently open the placeholder `https://memento-tech.com`.
  Real Play Store / App Store URLs must land before any production force-update
  flow has real value. (`decisions.md` 855, 865; Risk Watch 503.)
- **Parked maintenance-poll item** (adjacent, not strictly version-control):
  runtime maintenance-detection-from-`ready` is unwired (`decisions.md` 864
  Option A). Mentioned for completeness; not a version-control gap.

Everything else (the model, classification, split write paths, per-platform, dev
seed) is documented as **shipped** — boot-redesign closed 2026-05-29, real-device
regression passed.
