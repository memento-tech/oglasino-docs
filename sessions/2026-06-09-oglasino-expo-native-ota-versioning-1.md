# Session summary

**Repo:** oglasino-expo
**Branch:** dev (brief named `new-expo-dev`; Igor confirmed `new-expo-dev` was removed and `dev` is now the live branch — treated as overriding the brief's stale branch line)
**Date:** 2026-06-09
**Task:** Prepare native + OTA versioning for the first production build. Two phases — verify first, change second.

## Outcome

**Phase A (verify): complete. All three STOP-checks (A1, A3, A4) PASS.**
**Phase B (implement): STOPPED at B1, before any edit. No files touched.**

Reason: B1 instructs "Leave updates.url / updates.enabled as-is unless A1 found them unset; if unset, STOP and report (enabling OTA wiring is a separate decision, not this brief)." A1 found both unset. Underneath that, a more fundamental discovery: **`expo-updates` is not installed at all** (absent from `package.json` and `node_modules`). There is no OTA system in this app. `runtimeVersion: { policy: "appVersion" }` governs which OTA JS bundles a binary accepts — with no OTA system, it is inert. Setting it would imply a capability the app does not have and is exactly the OTA-coupled decision the B1 STOP defers. So I did not apply it. Every other Phase B target was already satisfied (see below), leaving runtimeVersion as the only substantive change — and it is the gated one.

## Phase A report (full)

### A1 — app.config.ts (single config; tiers branch on APP_ENV)

Marketing version is a **single shared string** across all three tiers. Tiers differ only by `bundleId`, `scheme`, `name`, `googleServicesFile`, and `universalLinkHost` — not by version. So the "same `1.0.0` across development/preview/production" requirement is already structurally true.

- `expo.version`: **`"1.0.0"`** (`app.config.ts:58`), identical across dev/preview/production.
- `runtimeVersion`: **ABSENT** — no `runtimeVersion` key anywhere in the file.
- `ios.buildNumber` / `android.versionCode`: **not in config** — delegated to EAS remote (`appVersionSource: remote` + production `autoIncrement`).
- `updates`: present but only `{ fallbackToCacheTimeout: 0 }` (`app.config.ts:151–153`). **`updates.url`: ABSENT. `updates.enabled`: ABSENT.**
- `expo-updates` package: **NOT installed** (absent from `package.json`; `node_modules/expo-updates` does not exist). Expo SDK `~54.0.33`.
- **A1 STOP** (runtimeVersion already a deliberate FIXED string or non-appVersion policy): **NOT triggered → PASS.** It is simply absent, not a deliberate competing choice.

### A2 — eas.json

- `cli.appVersionSource`: **`"remote"`** (`eas.json:4`).
- production profile: **`autoIncrement: true`** (`eas.json:44`), `distribution: "store"`, android `app-bundle`. **No `channel`.**
- preview profile: `distribution: "internal"`, android `apk`. **No `channel`.**
- development profile: `developmentClient: true`, `distribution: "internal"`. **No `channel`.**
- channel→profile mapping: **ABSENT entirely** — no `channel` key on any profile. Consistent with no OTA / EAS Update wiring.

### A3 — force-update gate (system B) — **answers the brief's critical question**

Path: bootStore Gate 2 `runVersionGate` (`src/lib/store/bootStore.ts:260`) → `getAppVersionConfig()` (`src/lib/services/appVersionConfigService.tsx`) → `GET /public/app/version/{Platform.OS}?currentVersion=${Application.nativeApplicationVersion}` (`appVersionConfigService.tsx:8,11`). Backend returns `{ latestVersion, minSupportedVersion, forceUpdate, optionalUpdate }` (`src/lib/types/AppVersion.ts`). `forceUpdate` → `toUpdateRequired` (HardUpdateScreen); `optionalUpdate` → `softUpdate` flag (SoftUpdateModal); `latestVersion` stored for both screens (`bootStore.ts:268`).

- **Value the gate compares: the MARKETING version** — `Application.nativeApplicationVersion` (= `expo.version` `"1.0.0"`), **NOT** `nativeBuildVersion`. The comparison itself runs server-side; the client supplies the marketing version.
- **A3 STOP** (gate compares the BUILD NUMBER): **NOT triggered → PASS.** The appVersion-policy plan and the "floor = 1.0.0" assumption hold.
- Minor note: the client read path is `/public/app/version/{OS}` (not `/internal/...`). The `/internal/app/version/ceiling` endpoint is the **write** side (A4). The brief's line-14 reference to the gate reading `/internal/...` slightly conflated read and write paths — no action, just clarity.

### A4 — EAS post-build ceiling hook

- Location: `.eas/workflows/production.yml` (jobs `publish_ceiling_android` / `publish_ceiling_ios`, each `needs` its build job) and an **identical** pair in `.eas/workflows/preview.yml`.
- Value sent: `latestVersion` = `npx expo config --json → .version` = the **marketing version** (currently `"1.0.0"`). `POST $BACKEND_BASE_URL/internal/app/version/ceiling`, header `X-INTERNAL-TOKEN`, body `{ "platform", "latestVersion" }`. Failure is non-fatal (`|| echo …`).
- Triggers: production profile (push to `main`) and preview profile (push to `preview`).
- **A4 STOP** (cannot locate hook, or it sends a non-marketing value): **NOT triggered → PASS.**

### A5 — store-redirect URLs (report only, do not fix)

- `HardUpdateScreen.tsx:18` → `Linking.openURL('https://memento-tech.com')` — **placeholder**.
- `SoftUpdateModal.tsx:18` → `Linking.openURL('https://memento-tech.com')` — **placeholder**.
- Both still the placeholder, exactly as state.md predicted. Not fixed — real App Store / Play Store URLs come from Igor once the app is published.

## Phase B status (no edits made)

- **B1 marketing version `1.0.0`:** already satisfied (single shared string) — no edit needed.
- **B1 runtimeVersion appVersion:** **NOT applied** — gated by the updates.url STOP + the expo-updates-absent discovery (see Outcome).
- **B1 updates.url / enabled:** both unset → **B1 STOP triggered.**
- **B2 `cli.appVersionSource: "remote"`:** already correct — no edit needed.
- **B2 production `autoIncrement: true`:** already correct — no edit needed.
- **B2 channel mappings:** **ABSENT** on all profiles. Reported, not fixed; no `eas` command run. Consistent with no OTA wiring.
- **B4 tsc/lint/expo-doctor:** not run — no edits to verify (B4 is conditional on Phase B edits).

## Files touched

- None.

## Tests

- Ran: none (read-only verification session; no code changed).

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change required from this session. (Note for Mastermind below: the OTA-absence finding may warrant reconciling the release-readiness planning model, but there is nothing concrete to flip — no edit drafted.)
- issues.md: no change.

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code touched, nothing to clean.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): two flagged in "For Mastermind" (low severity).
- Part 6 (translations): N/A this session.
- Other parts touched: brief's hard rule (code-is-ground-truth) — every cited file:line was opened with `view`/`cat` AND independently cross-checked with `grep`/`rg`.

## Known gaps / TODOs

- runtimeVersion not set; OTA not wired — deferred to Mastermind/Igor decision (see below). None added as TODO comments in code.

## For Mastermind

### Brief vs reality

1. **The brief presupposes an OTA system that does not exist.**
   - Brief says: system (A) is "EAS native runtimeVersion = which OTA JS bundles a binary will accept"; B1 sets `runtimeVersion: { policy: "appVersion" }` and treats `updates.url` as likely already wired.
   - Code says: `expo-updates` is not in `package.json` and not in `node_modules`; `updates` block is only `{ fallbackToCacheTimeout: 0 }` (`app.config.ts:151`); no `updates.url`, no `updates.enabled`, no `runtimeVersion`, no `channel` on any eas.json profile. There is no OTA pipeline at all.
   - Why this matters: `runtimeVersion` policy `appVersion` only governs OTA-bundle acceptance. With no OTA system it is functionally inert and would imply a capability the app lacks. The B1 STOP ("enabling OTA wiring is a separate decision") is exactly this case.
   - Recommended resolution: treat OTA wiring (install `expo-updates`, set `updates.url`/`enabled`, define channels per profile, then `runtimeVersion: appVersion`) as one deliberate decision/brief. **Decision needed:** either (a) defer `runtimeVersion` with the rest of the OTA wiring (my recommendation — keeps config honest), or (b) apply `runtimeVersion: { policy: "appVersion" }` now in isolation to pre-position the native runtime boundary, accepting it is inert until OTA lands. I did not guess — left unapplied pending your call.

### Phase A STOP-check summary (for the record)
- A1: **PASS** (runtimeVersion absent, not a deliberate competing choice).
- A3: **PASS** — **the gate compares the MARKETING version** (`Application.nativeApplicationVersion`), not the build number. The appVersion-policy plan and floor=1.0.0 assumption are safe.
- A4: **PASS** (ceiling hook in both `.eas/workflows/*.yml`, sends the marketing version via `expo config .version`).

### Already-satisfied Phase B targets (no work was outstanding except the gated runtimeVersion)
- version `1.0.0` (shared across tiers), `appVersionSource: remote`, production `autoIncrement: true` — all already in place.

### Part 4a simplicity evidence (required)
- Added (earned complexity): nothing — no code changed.
- Considered and rejected: applying `runtimeVersion: { policy: "appVersion" }` in isolation — rejected for now because it is inert without an OTA system and the brief's STOP defers OTA wiring; surfaced as a decision instead of guessing.
- Simplified or removed: nothing.

### Adjacent observations (Part 4b)
- **Placeholder store URLs (low, expected):** `HardUpdateScreen.tsx:18` and `SoftUpdateModal.tsx:18` both `Linking.openURL('https://memento-tech.com')`. Already tracked as a pre-launch task; needs real published-app URLs. Did not fix — out of scope and unresolvable without published URLs.
- **Brief read/write path conflation (low):** brief line 14 describes the gate as reading `/internal/app/version/ceiling`; the client actually reads `/public/app/version/{OS}` and `/internal/...ceiling` is the workflow write side. Cosmetic; flagged so a future reader isn't misled.

### Config-file impact restated
- None required. The OTA-absence finding is informational for planning; no `state.md`/`decisions.md` edit is drafted because nothing concrete flips this session.
