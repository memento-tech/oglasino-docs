# Audit — Mobile cleanup inventory (oglasino-expo)

**Repo:** oglasino-expo · **Branch:** new-expo-dev · **Mode:** READ-ONLY (inventory, no fixes) · **Date:** 2026-06-05

**Goal:** definitive list of dead/leftover code so a fix brief can be written. Evidence is `grep`/path-import analysis over `src/` + `app/` (the two source roots per `tsconfig.json`; `@/*` aliases resolve into `src/`). No barrel re-exports exist in the repo (`grep "export *"` → none), and the only dynamic `require()`s are asset images, so a zero-importer result for a module is conclusive — nothing reaches it via a re-export or a dynamic path.

---

## Q1 — The known item: `shown` member on the expo notification type

**CONFIRMED DEAD. Safe to remove (no consumer, no writer).**

- **Declaration:** `src/lib/client/firebaseNotifications.ts:45` — `shown: boolean;` on `interface FirebaseNotification` (lines 40–50).
- **Never read / never rendered:** repo-wide grep for property access finds nothing:
  - `grep -rn "\.shown\|markNotificationAsShown\|nonShown" src` → **no matches** (exit 1).
  - The only `shown` hits anywhere in `src/` are the substring inside English code comments ("…two copy lines are shown", "empty picker is never shown", etc.) — none are the notification field. (`grep -rn "shown" src` → 7 hits, all comments + the line-45 declaration.)
- **Never written:** the only writer in the file is `markNotificationsAsSeen` (line 96), which writes `{ seen: true }`. No code writes `shown`. The field is *populated only* by the Firestore `...doc.data()` spread (lines 63–66, 88–91), i.e. it carries whatever the backend doc holds, but nothing in the app touches it.
- **Backend cross-check (from issues.md 2026-06-02 close-out, lines 344–353):** the backend writes `seen`, never `shown` (`DefaultNotificationsService` writes no `shown` key); the web type never declared `shown`. Mobile's only former consumer (the `nonShown` filter) and writer (`markNotificationAsShown`) were already removed in the notifications-feature E2 session. The line-45 member is the last residue.
- **Safe to remove?** **YES.** It is a TS-only interface member. Removing it cannot affect runtime: the spread still copies any field Firestore returns; nothing references the typed property. No consumer exists on mobile, web, or backend.

---

## Q2 — Other genuinely-removable orphans

Every item below was verified to have **zero importers** across `src/` + `app/` by import-path grep (`/<basename>['"]`), cross-checked by symbol grep. All are **pure TS/TSX → ship on the next Metro bundle, no native rebuild** (see Q4 for the single exception). Excludes the Facebook scaffolding (see the explicit exclusion note at the end of Q2).

### 2a. Dead modules referenced nowhere (whole files)

| # | File | What it is | Evidence | Safe to remove? |
|---|------|-----------|----------|-----------------|
| 1 | `src/lib/hooks/useGoogleLogin.ts` | Abandoned `expo-auth-session` Google-login hook. Carries a placeholder `iosClientId: 'YOUR_IOS_CLIENT_ID'` and a hard-coded `androidClientId`. | Imported nowhere (`grep -rn useGoogleLogin` → only its own definition). The **real** Google login is `loginWithGoogleFirebase` in `src/lib/services/authService.ts:199` via the native `@react-native-google-signin/google-signin`. | **YES.** Superseded scaffolding. **Cascade note (Q4):** this is the *only* user of `expo-auth-session` (`grep -rln expo-auth-session src` → just this file), so deleting it orphans the `expo-auth-session` dependency. The file delete is JS-only; the dep removal is a prebuild item. `expo-web-browser` (also imported here) must **stay** — it is used by the real `authService.ts`. |
| 2 | `src/lib/services/catalogService.ts` | A `getCatalog()` HTTP service (18 lines). | `grep -rn "getCatalog\b"` → only its own definition + its own log strings. Catalog actually arrives embedded in `BaseSiteDTO` via `baseSitesService`/`bootStore`; nothing calls `getCatalog`. | **YES.** |
| 3 | `src/lib/storage/authStorage.ts` | Custom `access_token` AsyncStorage store (`setAuthToken`/`getAuthToken`/`clearAuthStorage`). | `grep -rn "setAuthToken\|getAuthToken\|clearAuthStorage\|access_token"` → only this file. The two other "authStorage" mentions are *comments* in `consentStorage.ts`/`themeStorage.ts` ("…direct AsyncStorage, like authStorage…"). Firebase manages its own token persistence; this vestige is unused. | **YES.** (Minor: removing it leaves two harmless pattern-reference comments mentioning the name — cosmetic, see Q2 note.) |
| 4 | `src/lib/store/userPreferenceStorage.ts` | Generic prefixed AsyncStorage wrapper (`userPreferenceStorage`, key prefix `@user_pref:`). | `grep -rn userPreferenceStorage` → its own definition + two *comments* in `themeStorage.ts`/`consentStorage.ts` ("NOT the generic userPreferenceStorage"). Storage was split into per-key modules (`themeStorage`, `consentStorage`); this generic wrapper has no caller. | **YES.** (Same harmless-stale-comment note as #3.) |

### 2b. Dead components referenced nowhere (whole files)

| # | File | What it is | Evidence | Safe to remove? |
|---|------|-----------|----------|-----------------|
| 5 | `src/components/product/ProductBreadcrumb.tsx` | A product breadcrumb component (`export default ProductBreadcrumb`). | Imported nowhere (`grep -rn ProductBreadcrumb src app` → only its own definition). | **YES.** Note: also carries a dead local (see 2d #13). |
| 6 | `src/components/FilterSelector.tsx` | A filter-option selector component. | Zero importers. | **YES.** |
| 7 | `src/components/user/UserAvatar.tsx` | An avatar component (`size`/`imageUrl`/`focused`). | Zero importers. | **YES.** |
| 8 | `src/components/aboutPage/RegisterButton.tsx` | About-page register button. | Zero importers. | **YES.** |
| 9 | `src/components/dialog/components/ADialogTemplate.tsx` | A boilerplate dialog **template** (`export default function Dialog`, 14 lines) — scaffolding stub. | Zero importers. | **YES.** |

### 2c. Dead type declarations referenced nowhere (whole files)

All verified zero-importer. Two are **byte-identical duplicates** of a live type that lives elsewhere (the canonical copy stays):

| # | File | Note | Safe to remove? |
|---|------|------|-----------------|
| 10 | `src/lib/types/ui/CategoriesFromPath.ts` | **Dead duplicate.** Canonical `CategoriesFromPath` is `src/lib/utils/utils.ts:71` (byte-identical); every consumer imports from `@/lib/utils/utils`. | **YES** (delete the `types/ui` copy; keep `utils.ts`). |
| 11 | `src/lib/types/KeyLabelPair.ts` | **Dead duplicate.** Canonical `KeyLabelPair` is declared in `src/lib/types/catalog/RegionAndCityDTO.ts:1` (byte-identical) and consumed there. | **YES** (delete the standalone copy; keep the one in `RegionAndCityDTO.ts`). |
| 12 | `src/lib/types/StatsDTO.ts` | No importer. | YES |
| 13 | `src/lib/types/ui/CardSelectionOption.ts` | No importer. | YES |
| 14 | `src/lib/types/ui/CardStyling.ts` | No importer (`CardStylings`/`CardStyling`). | YES |
| 15 | `src/lib/types/configuration/ConfigurationDTO.ts` | No importer. | YES |
| 16 | `src/lib/types/configuration/BasicMetadata.ts` | No importer (`SocialLinks`/`BasicMetadata`). | YES |
| 17 | `src/lib/types/suggestion/SuggestionDTO.ts` | No importer. | YES — but see Q2 future-scaffolding note. |
| 18 | `src/lib/types/suggestion/SuggestionsFilterRequestDTO.ts` | No importer. | YES — see note. |
| 19 | `src/lib/types/user/UserDetailsDTO.ts` | No importer (`extends UserOverviewDTO`). | YES — see note. |
| 20 | `src/lib/types/product/PreValidateProductRequestDTO.ts` | No importer. | YES — see note. |
| 21 | `src/lib/types/review/ReviewFilterRequestDTO.ts` | No importer. | YES — see note. |
| 22 | `src/lib/types/report/ReportFilterRequestDTO.ts` | No importer. | YES — see note. |
| 23 | `src/lib/types/report/ReportDTO.ts` | No importer. | YES — see note. |
| 24 | `src/lib/types/cookie/UserPreference.ts` | No importer. (Sole content of the `types/cookie/` dir.) | YES |
| 25 | `src/lib/types/filter/UsersFiltersRequest.ts` | No importer. | YES — see note. |
| 26 | `src/lib/types/filter/FilterCategoriesProps.ts` | No importer. | YES |

> **Future-scaffolding judgment (for the fix-brief author).** Items 17–23, 25 are backend-DTO-mirror "request/details" types (`Suggestion*`, `*FilterRequestDTO`, `ReportDTO`, `UserDetailsDTO`, `PreValidateProductRequestDTO`, `UsersFiltersRequest`). They are genuinely dead **now** (zero importers), but a few sit in feature areas still queued for mobile adoption (the A–I queue in `expo-release-readiness`). Removing them is technically safe; if the fix brief wants to be conservative, confirm with Mastermind that none are deliberate pre-staging for an imminent A–I brief before deleting. This is *not* an "I'm-unsure-it's-dead" caveat (it is provably unreferenced) — it is a "delete-now vs. re-add-later" call for Igor.

### 2d. Dead locals / unused bindings inside live files (linter-confirmed)

These are not whole-file removals; they are dead declarations inside files that are otherwise alive. Surfaced by `npx eslint src` (warnings, lint still passes):

| # | Location | What | Safe to remove? |
|---|----------|------|-----------------|
| 13 | `src/components/product/ProductBreadcrumb.tsx:11` | `const t = useTranslations(TranslationNamespace.COMMON_SYSTEM);` — `t` never used (`no-unused-vars`). Removing it also orphans the `useTranslations` + `TranslationNamespace` imports (lines 1–2) — both then deletable. *Moot if the whole file is removed per 2b #5.* | YES |
| 14 | `src/components/dialog/dialogs/PortalConfigDialog.tsx:45–46` | `const router = useRouter();` and `const segments = useSegments();` — both unused (`no-unused-vars`). Removing them orphans `useRouter, useSegments` from the `expo-router` import (line 15) — deletable. | YES |
| 15 | `src/components/dialog/dialogs/InfoDialog.tsx:47` | `catch (e)` binding unused. Cosmetic — convert to optional-catch `catch {`. | YES (low value) |
| 16 | `src/lib/services/appVersionConfigService.tsx:19` | `catch (error)` binding unused. Same optional-catch nit. | YES (low value) |
| 17 | `src/lib/hooks/useGoogleLogin.ts:10` | `request` unused in the `[request, response, promptAsync]` destructure. *Moot — the whole file is dead (2a #1).* | n/a (file dies) |

> **Excluded by brief — Facebook sign-in scaffolding (E2 / wontfix, issues.md 2026-06-01).** I explicitly did **not** include, and recommend the fix brief continues to retain: `src/components/icons/FacebookIcon.tsx` (orphan, surfaced in the same scan), `authStore.ts` `loginWithFacebook` (type + impl) and its `loginWithFacebookFirebase` import, `authService.ts:215` `loginWithFacebookFirebase` stub + its commented FB-SDK block, `authStore.test.ts` FB mock, and the `DeleteAccountConfirmationDialog` `'facebook.com'` provider branch. These are intentionally retained per the 2026-06-04 wontfix; they are out of scope for this cleanup.

> **Stale-comment follow-on (low).** If 2a #3/#4 (`authStorage`, `userPreferenceStorage`) are deleted, the pattern-reference comments in `src/lib/storage/themeStorage.ts:6-7` and `src/lib/storage/consentStorage.ts:7` ("…like authStorage…", "NOT the generic userPreferenceStorage") name modules that no longer exist. Harmless and still historically sensible, but the fix brief may want to reword them in the same pass.

---

## Q3 — Looks dead but NOT confidently removable (do NOT brief for removal)

- **`src/components/basic/CategorySelector.tsx:97` and `:105`** — `next.has(id) ? next.delete(id) : next.add(id);`. ESLint flags these as `no-unused-expressions`, which *looks* like dead code. **It is not** — these are ternaries evaluated for their `Set` mutation side-effects (toggle add/remove) and are functionally load-bearing. Do **not** brief removal; at most a stylistic rewrite to `if/else`, which is out of scope for a dead-code sweep.
- **`expo-web-browser` import in `useGoogleLogin.ts`** — when that dead file is removed, do **not** also drop the `expo-web-browser` dependency: it is still used by the live `src/lib/services/authService.ts`. Only `expo-auth-session` becomes orphaned (Q4).
- **Whole-file orphans under `app/`** — none flagged: every file under `app/` is an expo-router route registered by file path (not by import), so a "no importer" result there is *not* evidence of deadness. All Q2 candidates are under `src/`, where import-reachability is the correct test. (Stated so the fix-brief author does not extend the same grep logic into `app/`.)

---

## Q4 — Removal that requires a prebuild / native rebuild (sequencing)

- **All Q1 + Q2 items are pure JS/TS.** Deleting the dead `.ts`/`.tsx` files, the `shown` member, and the dead locals changes only the JS bundle — they take effect on the next Metro bundle with **no prebuild and no native rebuild**. None of them are referenced in `app.json`/native config (top-level config grep clean; they are components/types/services).
- **Single native-rebuild caveat — `expo-auth-session` dependency.** Deleting `useGoogleLogin.ts` (Q2 #1) makes `expo-auth-session` (`package.json:39`, `~7.0.10`) an unused dependency. Removing the *dependency* is `npm uninstall` + autolinking regeneration → a **prebuild/native rebuild** item, and editing native config / running a prebuild is outside a normal mobile **code** session's hard rules (no `app.json`/native edits, no prebuild without an explicit brief). So sequence it like the already-logged `expo-tracking-transparency` decision (issues.md 2026-06-02, "remove at next prebuild"): **delete the file in the JS cleanup session; drop the `expo-auth-session` dep on the next owed prebuild.** (`expo-auth-session` has no entry of its own in `ios/Podfile.lock` — it is JS over `expo-web-browser`/`expo-crypto` — so the rebuild is about clean autolinking, not a pod removal.)
- **Net:** the cleanup can ship in one JS-only code session for everything except the `expo-auth-session` dep drop, which rides the next prebuild.

---

## Summary counts

- **Q1:** 1 confirmed-dead type member (`shown`).
- **Q2:** 4 dead modules, 5 dead components, 17 dead type files (2 of them exact duplicates of live types), 4 dead locals/unused-bindings inside live files. Facebook scaffolding explicitly excluded.
- **Q3:** 2 false-positive patterns to NOT brief (`CategorySelector` side-effect ternaries; `expo-web-browser` keep-import), plus the `app/`-routes caveat.
- **Q4:** everything is JS-only/next-bundle except the optional `expo-auth-session` dep drop, which is a prebuild item.
