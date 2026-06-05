# Audit — Dead/unused code inventory (oglasino-expo)

**Repo:** oglasino-expo · **Branch:** new-expo-dev · **Mode:** READ-ONLY (inventory, no fixes) · **Date:** 2026-06-05

**Goal:** inventory dead/unused code; doubles as the mobile-cleanup inventory. Every candidate carries `file:line`, what it is, grep/evidence, a tier, and a **prebuild-vs-next-JS-bundle** flag (sequencing).

---

## Relationship to the earlier same-day audit (read first)

A prior session today already produced `.agent/audit-mobile-cleanup-inventory.md` (session `2026-06-05-oglasino-expo-mobile-cleanup-inventory-1.md`), covering essentially this same ground. **This audit independently re-verified every claim against the code** (the prior file was not trusted blind) and **confirms it**, with three substantive additions and one methodology correction the prior pass missed:

1. **Three dead dynamic-icon files** (`PressureRangeFilterIcon`, `OtherIcon`, `WeightFilterIcon`) the prior audit did not list — **Tier 2**, with a cross-repo verification need (see §Tier 2).
2. **Two member-level dead fields** on the live `AuthUserDTO` (`banReason`, `deletionStatus`) — **Tier 2** wire fields. The prior audit did member-level analysis for `shown` only.
3. **`UserOverviewDTO` is a cascade orphan** — alive only because two *already-dead* type files import it.
4. **Methodology correction:** the prior audit asserted "No barrel re-exports exist in the repo." That is **false** — `src/components/icons/dynamic/index.ts` is a named-re-export barrel (`export { default as X }`, which their `export *` grep missed). It is consumed by `DynamicIcon` via a runtime `Icons[name]` namespace lookup, so **zero-direct-import ≠ dead** for the ~135 icons in that barrel. This blind spot is why the 3 dead icons were missed and is corrected here. (It does not invalidate any other prior conclusion — no other listed item is an icon or lives in a barrel.)

→ **For Mastermind:** this audit and `audit-mobile-cleanup-inventory.md` are near-duplicates. Treat THIS file as the superset.

---

## Methodology & evidence base

- Two source roots per `tsconfig.json`: `src/` and `app/`. `@/*` resolves into `src/`.
- **Whole-file deadness** = zero import-path referents (`grep -rnE "(import|from).*['\"].*/<basename>['\"]"`), cross-checked by symbol grep. Validated by a full orphan sweep over `src/components`, `src/lib/services`, `src/lib/hooks`, `src/lib/storage`, `src/lib/client`.
- **Dynamic-reachability check:** the only internal barrel is `src/components/icons/dynamic/index.ts`, consumed by `src/components/DynamicIcon.tsx:4,21` via `import * as Icons` + `Icons[name as keyof typeof Icons]`. Every other `import * as` in `src/` is a third-party library namespace (no internal dynamic component resolution). So a file is dynamically reachable **iff** it is exported from that one barrel.
- **`app/` is excluded from whole-file deadness:** every file under `app/` is an expo-router route registered by file path, not by import — a zero-importer result there is not evidence of deadness.
- No `export *` barrels exist (`grep -rln "export \*" src` → none). The only dynamic `require()`s in the repo are asset images.

---

## Q1 (must-check) — the `shown` member on the notification type (W2 leftover)

**CONFIRMED declared + dead. Tier 1. Safe to remove. Ships on next JS bundle (no prebuild).**

- **Declaration:** `src/lib/client/firebaseNotifications.ts:45` — `shown: boolean;` on `interface FirebaseNotification` (lines 40–50).
- **Never read / never rendered:** `grep -rn "\.shown\|markNotificationAsShown\|nonShown" src app` → **0 matches** (exit 1). The only `shown` substrings anywhere in `src/` are inside English code comments (`grep -rn "shown" src` → 7 hits: 6 comments + the line-45 declaration).
- **Never written:** the file's only Firestore writer is `markNotificationsAsSeen` (`firebaseNotifications.ts:96,99`), which writes `{ seen: true }`. No code writes `shown`. The field is populated only by the `...doc.data()` spread (`:65, :90`) — it carries whatever the doc holds, but nothing in the app touches it.
- **Backend cross-check (issues.md 2026-06-02 close-out, the `shown`-field bullet):** the backend writes `seen`, never `shown` (`DefaultNotificationsService` writes no `shown` key); the web type never declared `shown`. Mobile's only former consumer (`nonShown` filter) and writer (`markNotificationAsShown`) were removed in the notifications-feature E2 session. Line 45 is the last residue — and removing it is explicitly the "only live action" left from that close-out, deferred to a structural-sweep session (this one).
- **Safe to remove?** **YES.** A TS-only interface member. Runtime is unaffected — the spread still copies any field Firestore returns; nothing reads the typed property.
- **Prebuild?** No. Pure TS — next Metro bundle.

---

## TIER 1 — safe to delete (declared, zero readers, not dynamic, not native config)

All ship on the **next JS bundle, no prebuild** unless flagged. None are referenced in `app.config.ts` / native config (grep clean).

### 1a. The `shown` member — see Q1 above.

### 1b. Dead modules (whole files)

| # | File | What it is | Evidence (verified this session) | Prebuild? |
|---|------|-----------|----------------------------------|-----------|
| 1 | `src/lib/hooks/useGoogleLogin.ts` | Abandoned `expo-auth-session` Google-login hook. Carries placeholder `iosClientId: 'YOUR_IOS_CLIENT_ID'` (`:12`) and a hard-coded `androidClientId`. The **real** Google login is `loginWithGoogleFirebase` (`src/lib/services/authService.ts:199`) via native `@react-native-google-signin/google-signin`. | `grep -rn useGoogleLogin src app` → only its own def (`:9`). Orphan-sweep confirms. | **File: No.** **Dep: Yes** — see Tier 2 / §Sequencing: this is the sole `expo-auth-session` importer, so the file delete is JS-only but the dep drop is a prebuild item. |
| 2 | `src/lib/services/catalogService.ts` | A `getCatalog()` HTTP service (18 lines). Catalog actually arrives embedded in `BaseSiteDTO` via `baseSitesService`/`bootStore`. | `grep -rn "getCatalog" src app` → only its own def (`:5`) + its own log strings (`:12,:15`). | No |
| 3 | `src/lib/storage/authStorage.ts` | Custom `access_token` AsyncStorage store (`setAuthToken`/`getAuthToken`/`clearAuthStorage`). Firebase manages its own token persistence. | `grep -rn "setAuthToken\|getAuthToken\|clearAuthStorage\|access_token" src app` → only this file (`:3,:5,:9,:13`). The two other "authStorage" mentions are *comments* in `consentStorage.ts:7` / `themeStorage.ts` (pattern references). | No |
| 4 | `src/lib/store/userPreferenceStorage.ts` | Generic prefixed AsyncStorage wrapper (key prefix `@user_pref:`). Storage was split into per-key modules (`themeStorage`, `consentStorage`). | `grep -rn userPreferenceStorage src app` → its own def + two *comments* in `themeStorage.ts:7` / `consentStorage.ts:7` ("NOT the generic userPreferenceStorage"). | No |

> **Stale-comment follow-on (low):** deleting #3/#4 leaves the pattern-reference comments in `themeStorage.ts:6-7` and `consentStorage.ts:7` naming modules that no longer exist. Harmless; a fix brief may reword them in the same pass.

### 1c. Dead components (whole files)

| # | File | What it is | Evidence | Prebuild? |
|---|------|-----------|----------|-----------|
| 5 | `src/components/product/ProductBreadcrumb.tsx` | Product breadcrumb component (`export default ProductBreadcrumb`). Also carries a dead local (see 1e #16). | `grep -rn ProductBreadcrumb src app` → only its own def. Orphan-sweep confirms. | No |
| 6 | `src/components/FilterSelector.tsx` | A filter-option selector component. (NOTE: distinct from the **live** `*FilterSelector` family — `RangeFilterSelector`, `MultiOptionFilterSelector`, `SingleOptionFilterSelector`, `DateFilterSelector` — which a naïve substring grep falsely associates with it.) | No bare-name `import FilterSelector from` and no `components/FilterSelector'` path import anywhere (`grep -rnE "import +FilterSelector +from"` → none). Orphan-sweep confirms. | No |
| 7 | `src/components/user/UserAvatar.tsx` | Avatar component (`size`/`imageUrl`/`focused`). | Zero importers (orphan-sweep). | No |
| 8 | `src/components/aboutPage/RegisterButton.tsx` | About-page register button. | Zero importers (orphan-sweep). | No |
| 9 | `src/components/dialog/components/ADialogTemplate.tsx` | Boilerplate dialog **template** stub (`export default function Dialog`, 14 lines). | Zero importers (orphan-sweep). | No |

### 1d. Dead type files (whole files)

All verified zero-importer this session via per-name import grep. Two are **byte-identical duplicates** of a live type (the canonical copy stays).

| # | File | Note | Prebuild? |
|---|------|------|-----------|
| 10 | `src/lib/types/ui/CategoriesFromPath.ts` | **Dead duplicate.** Canonical `CategoriesFromPath` is `src/lib/utils/utils.ts:71`; all 4 consumers import from `@/lib/utils/utils` (`ProductDetails.tsx:4`, `FiltersDialog.tsx:18`, `resolveDeepLinkFilters.ts:8`, `app/(portal)/(public)/product/[...productData].tsx:19`). Delete the `types/ui` copy; keep `utils.ts`. | No |
| 11 | `src/lib/types/KeyLabelPair.ts` | **Dead duplicate.** Canonical `KeyLabelPair` is declared and consumed in `src/lib/types/catalog/RegionAndCityDTO.ts:1` (`region`/`city` fields, `:7-8`). Nothing imports the standalone copy. | No |
| 12 | `src/lib/types/StatsDTO.ts` | Zero importers. | No |
| 13 | `src/lib/types/ui/CardSelectionOption.ts` | Zero importers. | No |
| 14 | `src/lib/types/ui/CardStyling.ts` | Zero importers. | No |
| 15 | `src/lib/types/configuration/ConfigurationDTO.ts` | Zero importers. | No |
| 16 | `src/lib/types/configuration/BasicMetadata.ts` | Zero importers. | No |
| 17 | `src/lib/types/cookie/UserPreference.ts` | Zero importers. Sole content of `types/cookie/`. | No |
| 18 | `src/lib/types/filter/FilterCategoriesProps.ts` | Zero importers. | No |
| 19 | `src/lib/types/suggestion/SuggestionDTO.ts` | Zero importers. **See future-scaffolding note.** | No |
| 20 | `src/lib/types/suggestion/SuggestionsFilterRequestDTO.ts` | Zero importers. **See note.** | No |
| 21 | `src/lib/types/user/UserDetailsDTO.ts` | Zero importers (`extends UserOverviewDTO`). **See note** + cascade below. | No |
| 22 | `src/lib/types/product/PreValidateProductRequestDTO.ts` | Zero importers. **See note.** | No |
| 23 | `src/lib/types/review/ReviewFilterRequestDTO.ts` | Zero importers. **See note.** | No |
| 24 | `src/lib/types/report/ReportFilterRequestDTO.ts` | Zero importers. **See note.** | No |
| 25 | `src/lib/types/report/ReportDTO.ts` | Zero importers (imports `UserOverviewDTO`). **See note** + cascade below. | No |
| 26 | `src/lib/types/filter/UsersFiltersRequest.ts` | Zero importers. **See note.** | No |

**Cascade orphan — `src/lib/types/user/UserOverviewDTO.ts`:** still imported, but ONLY by `ReportDTO.ts:1` (#25) and `UserDetailsDTO.ts:1` (#21) — **both themselves dead** (above). Its sole non-dead reachability is therefore nil. Once #21 and #25 are deleted, `UserOverviewDTO.ts` becomes a third dead type file. **Delete it in the same pass as #21/#25**, or it survives as a fresh orphan. (Listed here, not double-counted in the table.)

> **Future-scaffolding judgment (delete-now vs re-add-later — Igor/Mastermind call, NOT a deadness caveat).** Items 19–26 (the `Suggestion*`, `*FilterRequestDTO`, `ReportDTO`, `UserDetailsDTO`, `PreValidateProductRequestDTO`, `UsersFiltersRequest` backend-DTO mirrors) are provably dead **now** (zero importers), but several sit in feature areas still queued for mobile adoption (the A–I queue in `expo-release-readiness`). Removing them is technically safe; the only question is whether any is deliberate pre-staging for an imminent A–I brief. If a fix brief wants to be conservative, confirm with Mastermind before deleting these eight. The other type files (10–18) and the two duplicates are not feature-staging — delete freely.

### 1e. Dead locals / unused bindings inside live files (eslint-confirmed `no-unused-vars`)

Not whole-file removals — dead declarations inside otherwise-live files. All JS-only (next bundle).

| # | Location | What | Note |
|---|----------|------|------|
| 16 | `src/components/product/ProductBreadcrumb.tsx:11` | `const t = useTranslations(TranslationNamespace.COMMON_SYSTEM);` — `t` unused; removing it orphans the `useTranslations` + `TranslationNamespace` imports too. | **Moot if file removed per 1c #5.** |
| 17 | `src/components/dialog/dialogs/PortalConfigDialog.tsx:45-46` | `const router = useRouter();` and `const segments = useSegments();` — both unused; removing them orphans `useRouter, useSegments` from the `expo-router` import (`:15`). | File is otherwise live — local removal only. |
| 18 | `src/components/dialog/dialogs/InfoDialog.tsx:47` | `catch (e)` binding unused — convert to optional-catch `catch {`. | Cosmetic. |
| 19 | `src/lib/services/appVersionConfigService.tsx:19` | `catch (error)` binding unused — same optional-catch nit. | Cosmetic. |

---

## TIER 2 — looks dead, has a catch (do NOT delete without resolving the catch)

### 2a. Three dynamic-icon files absent from the registry (data-driven; cross-repo verify) — NEW

- **Files:** `src/components/icons/dynamic/PressureRangeFilterIcon.tsx`, `.../OtherIcon.tsx`, `.../WeightFilterIcon.tsx`.
- **Why they LOOK dead:** zero direct importers (orphan-sweep), and — critically — **absent from the dynamic barrel** `src/components/icons/dynamic/index.ts`. Count proof: **138 icon `.tsx` files vs 135 `export { default … }` lines** (138 − 135 = exactly these 3). Since `DynamicIcon` resolves icons only via `Icons[name]` over that barrel's namespace (`DynamicIcon.tsx:4,21`), an icon not in the barrel can never render — `Icons["OtherIcon"]` → `undefined` → `DynamicIcon` returns `null`.
- **The catch (why NOT Tier 1):** the icon `name` is **backend data**, not a literal — every call site passes `filter.iconId` / `cat.iconId` (`CategorySelector.tsx:139,169`; `DateFilter.tsx:61`; `RangeFilter.tsx:60`; `SimpleCollapsibleFilter.tsx:41`; `CategoryNavigation.tsx:85`; `CategoryNavigationDropdown.tsx:56`; `ProductSpec.tsx:47`). The icon set is a **web-synced, data-driven registry** (`index.ts` header: "AUTO-GENERATED ICON EXPORTS"; the icons arrived via commit `e807dbb` "RN Sync Implementation from WEB"). So whether these three names are reachable depends on the **backend catalog/filter seed `iconId` values**, which this repo cannot see.
  - If the backend **never** seeds `iconId ∈ {PressureRangeFilterIcon, OtherIcon, WeightFilterIcon}` → the three `.tsx` files are genuinely dead → safe to delete (Tier 1 in effect).
  - If the backend **does** seed any of them → the files are needed and their absence from the barrel is a **latent rendering bug** (icon silently missing on mobile), NOT deadness — the correct action is regenerating/extending the barrel to include them, not deleting the files.
- **Action before any deletion:** confirm against the backend catalog/filter seed (and/or web's icon barrel for parity) whether these three `iconId` names are emitted. **Do not delete blind.** Note `OtherIcon` in particular is a plausible generic fallback name.
- **Prebuild?** No (pure TSX either way). Note `generate-icons.sh` in `scripts/` is unrelated — it generates the PNG app/notification logos, not this React barrel.

### 2b. `banReason` and `deletionStatus` on `AuthUserDTO` — member-level, wire fields — NEW

- **`src/lib/types/user/AuthUserDTO.ts:19` `banReason: string | null;`** and **`:20` `deletionStatus: 'ACTIVE' | 'PENDING_DELETION';`**.
- **Why they look dead:** `grep -rn "banReason" src app | grep -v types/` → **0 client reads**; `grep -rn "deletionStatus"` (excluding the type def) → **0 client reads**. Nothing on mobile accesses either property. (For contrast, the sibling `scheduledDeletionAt` on the same DTO **is** read — 5 sites — and ban state is surfaced via `UserInfoDTO.state`, not `AuthUserDTO.banReason`.)
- **The catch:** `AuthUserDTO` is the `/auth/firebase-sync` response — a **wire DTO**. Unlike `shown` (which the backend was confirmed to never emit), `banReason`/`deletionStatus` are plausibly **emitted by the backend** and declared here as the documented contract. Removing the TS members is runtime-safe (the response spread still carries the data), but it narrows a declared wire contract.
- **Action:** Tier 2 — before removing, confirm the backend `AuthUserDTO`/firebase-sync payload and decide whether to keep these as documented-but-unread wire fields. Cross-repo; out of scope for a JS-only sweep.
- **Prebuild?** No.

### 2c. `expo-auth-session` dependency drop (the only prebuild item)

- Deleting `useGoogleLogin.ts` (Tier 1 #1) makes **`expo-auth-session` (`package.json:39`, `~7.0.10`) an unused dependency** — `grep -rln "expo-auth-session" src app` → that file only.
- Removing the **dependency** is `npm uninstall` + autolinking regeneration → a **prebuild / native rebuild**, which is outside a normal mobile *code* session's hard rules (no `app.json`/native edits, no prebuild without an explicit brief).
- **Sequence it like the logged `expo-tracking-transparency` decision** (issues.md 2026-06-02, "remove at next prebuild"): delete `useGoogleLogin.ts` in the JS cleanup session; drop the `expo-auth-session` dep on the next owed prebuild. (`expo-auth-session` has no `ios/Podfile.lock` entry of its own — it rides `expo-web-browser`/`expo-crypto` — so this is about clean autolinking, not a pod removal.)
- **Do NOT also drop `expo-web-browser`:** it is still imported by the **live** `src/lib/services/authService.ts:4` (and `authService.test.ts`). Only `expo-auth-session` is orphaned.

---

## TIER 3 — intentionally kept (recognized and EXCLUDED)

**Facebook sign-in scaffolding (E2 / wontfix — issues.md 2026-06-01 "Orphaned Facebook sign-in scaffolding" + 2026-06-04 wontfix).** Surfaced by the orphan sweep (`FacebookIcon.tsx` flagged) and explicitly **excluded** per brief. Igor retains it for future Facebook-login reuse. Seen-and-kept inventory:

- `src/components/icons/FacebookIcon.tsx` — orphan (no consumer).
- `authStore.ts` `loginWithFacebook` (type `:62` + impl `:143`) and its `loginWithFacebookFirebase` import (`:9`).
- `authService.ts:215` `loginWithFacebookFirebase` stub over its commented-out FB-SDK block (`:216–240`) — the commented block is a standing Part 4 violation, but retained-by-decision.
- `authStore.test.ts:34` stale `loginWithFacebookFirebase` mock.
- `DeleteAccountConfirmationDialog` `'facebook.com'` provider branch (structurally unreachable).

No other code is marked retained by a comment or by `decisions.md`.

---

## Q3-equivalent — false positives to NOT brief for removal

- **`src/components/basic/CategorySelector.tsx:97,105`** — `next.has(id) ? next.delete(id) : next.add(id);`. ESLint flags `no-unused-expressions`, which *looks* dead. It is **not** — these ternaries are evaluated for their `Set` mutation side-effects (toggle add/remove) and are load-bearing. At most a stylistic `if/else` rewrite, out of scope for a dead-code sweep.
- **`expo-web-browser`** — keep (live in `authService.ts`); see Tier 2c.
- **All `app/` files** — registered by expo-router file path, not by import. A zero-importer result there is not deadness. Every removal candidate in this audit is under `src/`.
- **The ~135 dynamic icons IN the barrel** — zero *direct* import, but reachable via `DynamicIcon`'s `Icons[name]` runtime lookup. Not dead. (Only the 3 NOT in the barrel are suspect — Tier 2a.)

---

## Per-item prebuild vs next-JS-bundle (sequencing summary)

- **Everything in this audit ships on the next Metro JS bundle with NO prebuild / native rebuild** — the `shown` member, all dead modules/components/type files, all dead locals, and (if confirmed dead) the 3 dynamic-icon files. None are referenced in `app.config.ts`/native config (grep clean).
- **The single prebuild item is the `expo-auth-session` DEPENDENCY drop** (Tier 2c), which rides the next owed prebuild — the `useGoogleLogin.ts` *file* delete itself is JS-only.
- **Net:** one JS-only code session can land the entire cleanup; only the `expo-auth-session` dep removal waits for the next prebuild.

---

## Summary counts

- **Q1 / Tier 1 member:** 1 confirmed-dead type member — `shown` on `FirebaseNotification`.
- **Tier 1 whole files:** 4 modules + 5 components + 17 type files (2 exact duplicates; +1 cascade orphan `UserOverviewDTO` once its two dead importers go) + 4 dead locals.
- **Tier 2 (catch):** 3 dynamic-icon files (data-driven; cross-repo verify), 2 member-level wire fields on `AuthUserDTO` (`banReason`, `deletionStatus`), 1 dependency drop (`expo-auth-session`, prebuild).
- **Tier 3 (excluded, kept):** Facebook scaffolding (6 sites).
- **Prebuild:** only the `expo-auth-session` dep drop. All else is next-bundle.
- **Methodology correction over the prior same-day audit:** the icon barrel + the 3 dead icons it hides.
