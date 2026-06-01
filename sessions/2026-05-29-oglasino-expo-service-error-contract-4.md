# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev (confirmed before starting)
**Date:** 2026-05-29
**Task:** Make airplane-mode cold start show an offline screen instead of the maintenance screen, by adding a connectivity check in front of the maintenance gate.

## Implemented

- **Installed `@react-native-community/netinfo` via `npx expo install`** — landed at `11.4.1`, the exact SDK 54-pinned version (Expo chose it; it is *not* in expo-doctor's version-mismatch list). One dependency, surgical: the only manifest change is netinfo (verified against `git diff package.json` / `package-lock.json`).
- **Gate 0 (connectivity) added to `bootStore.ts`**, ahead of the maintenance gate. `start()` now calls `runConnectivityGate()` first; it reads current connectivity once via `NetInfo.fetch()` (not a subscription) and, when NetInfo is *certain* there is no connection (`isConnected === false`), sets a new `'offline'` `BootStatus` and stops — the maintenance gate is never run against no network. Online / unknown (`null`) / a thrown NetInfo read all fall through to the existing maintenance gate unchanged.
- **`'offline'` status + offline screen.** `'offline'` added to the `BootStatus` union and a `toOffline()` transition added (mirrors `toMaintenance()`). `app/_layout.tsx` renders the offline screen via the existing overlay block. The screen reuses `BaseSiteSelector` with a new `isOffline` prop — same full-screen boot-message layout as the maintenance branch, with offline copy and no base-site buttons.
- **Reconnect → non-destructive re-entry.** New `OfflineReconnectInit` component mirrors `MaintenancePollInit`: while `status === 'offline'` it subscribes to `NetInfo.addEventListener` and, on connectivity returning, calls the machine's existing `reEnter()` (which is `start()`), clearing nothing. It uses a change-event subscription rather than a poll because connectivity emits a signal; the re-entry path is identical to maintenance-clear.
- **Offline copy** is a minimal hardcoded Serbian fallback (the brief's one justified exception) — see the translation-copy finding in "For Mastermind"; a backend-seeded key cannot resolve at the `'offline'` point by construction.

## Files touched

- package.json (+1 / -0) — `@react-native-community/netinfo: 11.4.1`
- package-lock.json (netinfo entry only)
- src/lib/store/bootStore.ts (+~45 / -3) — `'offline'` status, `runConnectivityGate` (Gate 0), `toOffline`, `start()` rewired to call Gate 0 first
- src/components/init/OfflineReconnectInit.tsx (new, ~55 lines) — reconnect listener + extracted `handleConnectivityChange`
- src/components/init/OfflineReconnectInit.test.ts (new) — 3 unit tests for the reconnect callback
- src/components/init/BaseSiteSelector.tsx (+~18 / -6) — `isOffline` prop, `OFFLINE_FALLBACK`, `messageOnly` branch
- app/_layout.tsx (+3 / -2) — mount `OfflineReconnectInit`, offline overlay branch, overlay comment
- src/lib/store/bootStore.test.ts (+~55) — NetInfo mock + Gate 0 test block

## How the offline path honors each boot invariant (required by the brief)

**Invariant 1 — one effect, empty deps.** No `useEffect` was added to the boot path. Gate 0 is a plain gate function the machine calls in sequence from `start()` (exactly like Gates 1–4), not a reactive effect. The single boot-driving effect in `app/_layout.tsx` (empty deps → `start()`) is unchanged. `OfflineReconnectInit` does have a `useEffect` keyed on `[status]`, but this is the **already-blessed external-event-source carve-out** — identical in shape to `MaintenancePollInit`, shipped in `expo-boot-redesign` (decisions.md 2026-05-29). The effect reads `status` only to start/stop the subscription; it never *advances the machine reactively* — advancement happens only via the explicit `reEnter()` call triggered by an **external** NetInfo change event, never by the status change itself.

**Invariant 2 — machine writes status; view reads status.** Gate 0 writes `'offline'` through the `toOffline()` transition helper (same mechanism as every other `toX`). The view (`_layout.tsx`) reads `status` to render the offline overlay. `OfflineReconnectInit` reads `status` only to gate its subscription lifecycle and **never writes status** — its sole machine interaction is `reEnter()`, the explicit entry point. Status does not cross from view back into the machine as an input.

**Invariant 3 — re-entry destroys nothing.** Reconnect calls `reEnter()` → `start()`, which sets `status: 'booting'` and runs Gate 0 again. `start()` clears no slot (base site, language, config, codes, translations, checksums all survive) — this is the exact non-destructive path maintenance-clear already uses, and the existing Invariant-3 tests (slot/identity preservation across re-entry) still pass unchanged. Gate 0 adds no clearing of its own.

## Reconnect re-entry mechanism (how it mirrors maintenance-clear)

`MaintenancePollInit` polls `checkIfMaintenance` on a 5s interval while `status === 'maintenance'` and calls `reEnter()` on clear, because maintenance has no client-observable signal. `OfflineReconnectInit` follows the same shape — a component whose `[status]`-gated effect attaches an external listener while in the blocked state and calls `reEnter()` at the recovery edge — but subscribes to `NetInfo.addEventListener` instead of polling, since connectivity *does* emit a change event. Both are external event sources calling the machine's existing `reEnter()`; neither writes status; both are non-destructive. The reconnect handler was extracted as the pure function `handleConnectivityChange` so it is unit-testable in the node test env, mirroring the extracted `pollMaintenanceOnce`.

## Translation-copy decision (the chicken-and-egg)

**Decision: minimal hardcoded Serbian fallback — the brief's one justified exception. No backend-seed dependency is asserted, because a backend-seeded key cannot be consumed at the `'offline'` point by construction.**

Reasoning (traced, not assumed): the offline screen renders only when Gate 0 sets `'offline'`. Gate 0 runs at every `start()`/`reEnter()` **before Gate 4**, which is the gate that initializes the i18n singleton. So at the `'offline'` point i18n is **not initialized** *and* the device has **no network**. I also confirmed the offline status is unreachable from `ready` (the only re-entry triggers are the maintenance poll, which runs only in `maintenance`, and the new offline listener, which runs only in `offline`), so there is no path where the screen renders with i18n already initialized from a prior pass. Therefore no backend-fetched ERRORS key can resolve there — offline is exactly when the fetch fails. Wiring a placeholder key that never resolves would be dead code (Part 4a), so I used a hardcoded fallback instead, matching this file's existing precedent (`INTRO_FALLBACK`, also Serbian — the app's `fallbackLng`). The two lines live in `OFFLINE_FALLBACK` in `BaseSiteSelector.tsx`, clearly commented as the justified exception. See "For Mastermind" for the recommended permanent home if localized offline copy is ever wanted.

## Tests

- Ran: `npm test` (vitest) → **13 files, 207 passed, 0 failed** (was 198; +6 Gate 0 tests in `bootStore.test.ts`, +3 in `OfflineReconnectInit.test.ts`).
- Ran: `npx tsc --noEmit` → clean (exit 0).
- Ran: `npm run lint` → **0 errors, 71 warnings** (all pre-existing baseline; my new files add no warnings — the two `import/first` lines flagged on `bootStore.test.ts` are the file's pre-existing mock-then-import pattern, line numbers shifted by the added mock).
- Ran: `npx expo-doctor` → 17/18 checks pass. The one failing check is **pre-existing Expo patch-version drift** (11 Expo-owned packages: `expo`, `expo-router`, `expo-crypto`, etc. — slightly behind the latest SDK 54 patch metadata). **netinfo is not among them** and matches the SDK-pinned version. Out of scope for this brief (fixing it means `expo install --check` upgrading 11 unrelated packages; not authorized). See "For Mastermind."
- New tests added: `OfflineReconnectInit.test.ts` (reconnect callback); Gate 0 block in `bootStore.test.ts` (offline→toOffline / online→advance / unknown→advance / throw→advance / start()-runs-Gate-0-first / toOffline()).

## Cleanup performed

- none needed. (No commented-out code, no debug logging added, no dead code introduced; `console.warn` was not added — Gate 0's failure path is silent-and-proceed by design.)

## Config-file impact

- conventions.md: no change.
- decisions.md: no change owed by this brief (the feature-close `decisions.md` entry is the chat's close-out, not this brief).
- state.md: **no edit drafted by this brief.** The DoD's "close the Risk Watch entry (service layer swallows errors)" and the Φ4 Risk Watch flip are explicitly the *chat* close-out per the brief's Config-file-impact note, not this brief. No Expo backlog row is removed by this brief (offline detection is part of the Φ4 foundation feature, not a standalone backlog row). Flagged for awareness only.
- issues.md: no change. (The "should mobile poll maintenance / read it from the edge worker" question is the existing 2026-05-29 entry and explicitly out of scope.)

## Obsoleted by this session

- nothing. (Gate 0 is additive ahead of Gate 1; no prior code became dead. The maintenance gate's behavior is untouched beyond letting the offline check run first.)

## Conventions check

- Part 4 (cleanliness): confirmed.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one flagged in "For Mastermind" (the pre-existing expo-doctor patch drift); nothing else newly observed in touched files.
- Part 6 (translations): one finding flagged — offline copy uses a justified hardcoded fallback (chicken-and-egg), no new mobile-authored namespace key, recommended ERRORS keys surfaced for Mastermind.
- Other parts touched: Part 9 (architectural defaults) — no new mobile-specific route; Gate 0 is client-only connectivity. Part 11 (trust boundaries) — N/A, no trust decision involved.

## Known gaps / TODOs

- none. (On-device airplane-mode verification is the Ψ chat's job per the brief; this brief ships the code.)
- Note: netinfo is a native module. It autolinks (no config plugin needed) but requires a native dev-client rebuild to function on device — that rebuild/verification is the Ψ chat, consistent with the brief's "out of scope: runtime/on-device verification."

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - **Added (earned complexity):** (1) `runConnectivityGate` / `'offline'` status / `toOffline` — the brief's core mechanism, one new gate matching the existing Gate 1–4 shape. (2) `OfflineReconnectInit` — earns its place as the reconnect event source; mirrors the shipped `MaintenancePollInit` precedent exactly rather than inventing a parallel pattern. (3) `isOffline` prop on `BaseSiteSelector` — reuses the existing full-screen boot-message surface instead of duplicating the intro layout; one prop + a `messageOnly` boolean.
  - **Considered and rejected:** (a) a brand-new `OfflineScreen` component — rejected; would duplicate `BaseSiteSelector`'s layout and "invent a new visual language" the brief forbids. (b) wrapping `NetInfo.fetch()` in `withGateTimeout` — rejected; it is a fast local native read and a probe hiccup must not escalate to maintenance. (c) a placeholder backend-seeded ERRORS key for the offline copy — rejected as dead code (it can never resolve at the offline point; see below). (d) detecting offline from the `ready` state at runtime — rejected; out of scope (Gate 0 runs at boot only, matching the spec's offline-at-cold-start framing).
  - **Simplified or removed:** nothing in this category.

- **Translation finding (Part 6 / brief §5.4, Step 5).** The offline screen uses a minimal hardcoded Serbian fallback because a backend-seeded key **cannot** resolve at the `'offline'` point (i18n is uninitialized before Gate 4, and there is no network to fetch ERRORS — offline is exactly when that fetch fails; the status is also unreachable from `ready`). This is the justified exception the brief anticipated. **If you want localized offline copy**, the correct home is **not** a backend-fetched ERRORS key (it will never be reached offline) but an **app-bundled i18n resource** loaded synchronously at module-eval — a small design decision for you, out of scope here. If you nonetheless want backend ERRORS keys reserved for symmetry, recommended pair: `offline.label.1` / `offline.label.2` in the `ERRORS` namespace (mirroring the maintenance `maintenance.label.1/2` shape). My recommendation: keep the hardcoded fallback; it is the only thing that can render at the offline point.

- **Adjacent observation (Part 4b).** `npx expo-doctor` reports one failing check: 11 Expo-owned packages (`expo` ~54.0.35 vs 54.0.33, `expo-router`, `expo-crypto`, `expo-auth-session`, `expo-dev-client`, `expo-file-system`, `expo-font`, `expo-image-picker`, `expo-linking`, `expo-localization`, `expo-notifications`) are a patch version behind the latest SDK 54 metadata. **Pre-existing** — my install added only netinfo (verified via git diff); these mismatches are independent dependency drift. File path: `package.json`. Severity: low (patch-level, no functional impact; `npx expo install --check` would resolve them in a dedicated chore). I did not fix this because it is out of scope and would touch 11 unrelated dependencies without brief authorization. Flagging so the expo-doctor "1 check failed" is not mistaken for a netinfo problem.

- **Nothing else flagged.**
