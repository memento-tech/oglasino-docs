# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-06-05
**Task:** Inventory dead/unused code (read-only); serve as the mobile-cleanup inventory. Tier each item, confirm the W2 `shown` notification field, check unused type members, and flag prebuild-vs-JS-bundle per item. Write findings to `.agent/audit-deadcode-expo.md`. No fixes.

## Implemented

- Read-only inventory only — **no code changed**. Produced `.agent/audit-deadcode-expo.md`.
- **Q1 confirmed:** `shown` on `FirebaseNotification` (`src/lib/client/firebaseNotifications.ts:45`) is declared, never written (only `seen` is), never read/rendered (`grep .shown/markNotificationAsShown/nonShown` → 0). Tier 1, safe to remove, next JS bundle.
- Independently re-verified every claim in the earlier same-day `audit-mobile-cleanup-inventory.md` against the code and **confirmed** it, then **extended** it with findings that pass missed.
- Tiered all findings per the brief (Tier 1 safe / Tier 2 has-a-catch / Tier 3 kept-Facebook) and flagged prebuild-vs-next-bundle per item: everything is next-bundle except the `expo-auth-session` dependency drop.

### New findings beyond the prior audit

- **3 dead dynamic-icon files** — `PressureRangeFilterIcon`, `OtherIcon`, `WeightFilterIcon` under `src/components/icons/dynamic/`. They are absent from the auto-generated barrel `index.ts` (138 icon files vs 135 exports) so `DynamicIcon`'s `Icons[name]` lookup can never resolve them. **Tier 2**, because icon names are backend-data-driven (`filter.iconId`/`cat.iconId`) — needs a cross-repo check of the catalog/filter seed before deletion (if the backend emits these names, the absence-from-barrel is a latent rendering bug, not deadness).
- **2 member-level dead wire fields** — `banReason` (`AuthUserDTO.ts:19`) and `deletionStatus` (`:20`) have 0 client reads. **Tier 2** (likely backend-emitted wire contract; confirm before removing).
- **`UserOverviewDTO` cascade orphan** — alive only via two already-dead type files (`ReportDTO`, `UserDetailsDTO`); delete it in the same pass or it becomes a fresh orphan.
- **Methodology correction:** the prior audit's "no barrel re-exports exist" is false — `icons/dynamic/index.ts` is a named-re-export barrel feeding `DynamicIcon` dynamically; zero-direct-import ≠ dead for the ~135 icons in it.

## Files touched

- `.agent/audit-deadcode-expo.md` (new, inventory output)
- `.agent/2026-06-05-oglasino-expo-deadcode-expo-1.md` (this summary)
- `.agent/last-session.md` (exact copy)
- **No source files touched** (read-only brief).

## Tests

- Not run — read-only audit, no code change. (Lint signal was used only as evidence: `no-unused-vars` warnings corroborate the dead locals; lint still passes.)

## Cleanup performed

- None needed (read-only inventory).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change — but see "For Mastermind": the `shown`-field close-out (issues.md 2026-06-02, "the only live action is removing the dead `shown` member … deferred to a dedicated expo structural-sweep session") and the `expo-auth-session` orphan are both now itemized here; the eventual fix session, not this audit, will warrant the issues.md status update.

## Obsoleted by this session

- This audit supersedes `.agent/audit-mobile-cleanup-inventory.md` as the superset (same scope, plus the 3 dynamic icons, the 2 wire-field members, the `UserOverviewDTO` cascade, and the barrel methodology correction). The prior file is not deleted — deleting another session's `.agent/` artifact is not mine to do; flagged for Mastermind to retire or merge.

## Conventions check

- Part 4 (cleanliness): N/A — read-only, no code authored, nothing to clean.
- Part 4a (simplicity): N/A this session (read-only) — no abstraction added/considered/removed. Stated explicitly per brief.
- Part 4b (adjacent observations): two flagged in "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: Part 8 (architectural defaults) — confirmed the icon registry is data-driven across the web↔mobile sync seam, which is what makes the 3 icons a cross-repo question rather than a local deletion.

## Known gaps / TODOs

- The 3 dynamic-icon files and the 2 `AuthUserDTO` wire fields are Tier 2 pending a **backend cross-repo check** I cannot perform from this repo (whether the catalog/filter seed emits those `iconId`s; whether firebase-sync emits `banReason`/`deletionStatus`). Surfaced, not resolved.
- No TODO/FIXME added.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code.
  - Considered and rejected: nothing — no code authored.
  - Simplified or removed: nothing — read-only.
- **Duplicate-work flag (medium).** This brief re-ran the inventory that `oglasino-expo-mobile-cleanup-inventory-1` (same date, same branch) already produced as `audit-mobile-cleanup-inventory.md`. I confirmed + extended rather than blindly duplicating; `audit-deadcode-expo.md` is the superset. Decide whether to retire/merge the older file (I did not delete it).
- **Adjacent observation 1 (low/medium) — undeclared direct dependency.** `expo-web-browser` is imported by the live `src/lib/services/authService.ts:4` (+ test) but is **not** a direct entry in `package.json` — it resolves transitively. Same shape as the backend Guava finding (issues.md 2026-06-04). If the transitive provider ever drops it, the real Google-login flow breaks silently. Out of scope; not fixed. Candidate for `issues.md`.
- **Adjacent observation 2 (low) — stale pattern-reference comments.** `themeStorage.ts:6-7` and `consentStorage.ts:7` reference `authStorage` / `userPreferenceStorage` by name; if those dead modules are deleted (Tier 1 #3/#4), the comments name non-existent modules. Reword in the same fix pass.
- **Sequencing recommendation for the eventual fix brief:** one JS-only session lands all Tier 1 items + the 3 icons (if the backend check clears them) + the `useGoogleLogin.ts` file delete; the `expo-auth-session` dep drop rides the next owed prebuild alongside the already-logged `expo-tracking-transparency` removal (issues.md 2026-06-02).
- **Config-file impact:** none required this session (read-only). No drafted config text. The `shown`-member removal and `expo-auth-session` orphan are tracked in issues.md already; their status flips belong to the fix session, not this audit.
