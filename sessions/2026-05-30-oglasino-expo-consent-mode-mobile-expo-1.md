# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-30
**Task:** Read-only Phase-2 audit — confirm consent-field cleanup targets + current line numbers (Part C) and discover the consent-UI build-surface (Part G) against the post-Φ1–Φ4 `new-expo-dev` tree.

> Full findings (C1–C8, G1–G6, Z, with every grep and file:line) are in the deliverable: **`.agent/audit-consent-mode-mobile-expo.md`**. This summary is the Part-5 record; the two For-Mastermind deliverables are reproduced below in full.

## Implemented

- Read-only audit only — no source touched. Fanned out 10 parallel read-only auditors over `new-expo-dev`, then re-verified the two load-bearing findings (Φ4 grenade, B2 double-set) by direct file read.
- Re-confirmed all C cleanup targets against current line numbers; two prior claims no longer hold (Φ4 grenade defused; B18 `var` gone). Everything else survives the foundation work.
- Mapped the G build-surface: AsyncStorage/consent-store patterns, analytics-init mount point, candidate UI mounts, privacy/terms screens, ATT absence, and the i18n/COOKIES-namespace situation.

## Files touched

- None (read-only). Two `.agent/` artifacts written: `audit-consent-mode-mobile-expo.md` (deliverable), this summary + `last-session.md`.

## Tests

- Not run — read-only audit, no code change. (`npm run lint` / `tsc` / `npm test` N/A this session.)

## Cleanup performed

- none needed (read-only audit; no code written).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change (8 adjacent observations surfaced in "For Mastermind" for Mastermind to triage; this session does not draft `issues.md` entries)

## Obsoleted by this session

- nothing (read-only audit). Note: the audit *identifies* dead code for a future C/Ω chat to delete (`src/lib/types/cookie/ConsentData.ts` + `GlobalCookie.ts` closed dead chain; web-only `src/lib/client/firebaseAnalytics.ts`; unused `src/lib/store/userPreferenceStorage.ts`) but deletes nothing.

## Conventions check

- Part 4 (cleanliness): N/A this session — no code written. Adjacent cleanliness items (`console.error` `user.tsx:74`, hardcoded `"DODAJ ZA BASE SITE"` `:242`, dead types/files) flagged in "For Mastermind."
- Part 4a (simplicity): N/A — no code written (structured evidence below, all "nothing").
- Part 4b (adjacent observations): 8 observations flagged with file:line + severity + "out of scope / read-only."
- Part 6 (translations): COOKIES-namespace usage mapped; 5 candidate-deleted keys + the surviving notifications/email/promo keys located; the phone-calling labels use DASHBOARD_PAGES not COOKIES.
- Part 7 (error contract): the save-throw path is silent to the user (observation #1) — relevant to C.
- Part 11 (trust boundaries): checked — no trust decision changes (Section Z). C removes a backend-ignored field; G's consent is device-local, client-side analytics gate only.

## Known gaps / TODOs

- The backend half of B4 (which of the 5 COOKIES keys are actually deleted from seed SQL) is a separate backend audit — this audit reports only the mobile references, which is the requested scope.
- The "remove vs wire" decision for `allowNotifications` is Mastermind's; the audit reports the current truth (declared + rendered, never seeded/compared/sent; DTO field exists).

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code written.
  - Considered and rejected: nothing — no code written.
  - Simplified or removed: nothing — no code written.

### Stale-audit delta summary (#1 deliverable)

Prior audit = `.agent/audit-expo-readiness-consent-mode.md`, 2026-05-23, on `dev`. Status on `new-expo-dev`:

| Item | Prior claim | Status now | Current file:line |
|---|---|---|---|
| `allowPreferenceCookies` inventory | 6 occ / 3 files | HOLDS (files same; 8 occ — line-split only) | `AuthUserDTO.ts:13`, `UpdateUserDTO.ts:12`, `user.tsx:46,67,97,176,286,287` |
| Φ4 grenade (updateUser no try/catch) | `:163`, no try/catch, throws uncaught | **DOES NOT HOLD — defused** | try/catch `user.tsx:166-186`, call `:167`, catch re-throws `:185` |
| B2 `allowPromoEmails` init bug | `:70` overwrites w/ phone-calling | HOLDS | `user.tsx:69-70` |
| B3 `allowNotifications` dead toggle | declared/rendered/never seeded/sent | HOLDS (all 4) | state `:47`, render `:299-303` |
| B4 five deleted COOKIES keys (mobile refs) | `254/255/257/267/268` | HOLDS (shifted) | `user.tsx:266,267,269,279,280`; hook `:35` |
| Other COOKIES keys (notif/email/promo) | "likely exist", `281-283/296-298/307-309` | HOLDS (confirmed referenced) | `user.tsx:293-295,308-310,319-321` |
| Dead types ConsentData/GlobalCookie | closed dead chain, deletable | HOLDS | `src/lib/types/cookie/ConsentData.ts`, `GlobalCookie.ts` |
| B17 `"DODAJ ZA BASE SITE"` | `:230`, placeholder | HOLDS (rendered `<Text>` label) | `user.tsx:242` |
| B16 `console.*` in user.tsx | `console.error` `:74` | HOLDS (one) | `user.tsx:74` |
| B18 `var` in user.tsx | `var toastMessage` `:178` | **DOES NOT HOLD — gone** (now `let`) | `user.tsx:188`; no `var` in file |

**Net:** every C target survives Φ1–Φ4 **except two the foundation already fixed** — the Φ4 grenade (try/catch now wraps the save) and B18 (`var → let`). C's real scope: remove `allowPreferenceCookies` from both DTOs + `user.tsx`; decide remove-vs-wire on `allowNotifications`; fix/remove the B2 double-set; remove the `ConsentData`/`GlobalCookie` dead chain; translate/remove the B17 placeholder; remove the B16 `console.error`. C8 confirms C is **removals only** — none of E's `disabled`/`state`/`deletionStatus` or F's `wasRegister` exist on either DTO yet.

### G build-surface summary (for the Phase 4 spec)

- **AsyncStorage pattern:** house style = dedicated per-concern module + own key constant + direct `AsyncStorage` calls. Generic `userPreferenceStorage` wrapper exists but is **unused** (zero callers) — not the convention.
- **Consent-store pattern:** only Zustand-persist precedent is the auth store (`authStore.ts:256-263`: `persist` + `createJSONStorage(() => AsyncStorage)` + `partialize` + `onRehydrateStorage`). `useConsentStore` can mirror this or use the simpler per-concern-module template.
- **Analytics-init mount (F-facing):** a new child of `AppInit` (`src/components/init/AppInit.tsx:21-29`), mounting only at `bootStatus === 'ready'` (`app/_layout.tsx:77`). Dead `firebaseAnalytics.ts` (web SDK) cannot be reused.
- **Candidate UI mounts:** first-run = `intro-picker` boot state → `BaseSiteSelector` (`app/_layout.tsx:121`); settings = post-cleanup `user.tsx` or a new `app/owner/dashboard/privacy.tsx` (no slot today); nav = footer `companyNavigations.tsx` (by `/privacy`) and/or `owner.account.sub.label` sidebar group in `dashboardNavigations.tsx`. Policy links `/privacy` + `/terms` (network-fetched markdown).
- **i18n/namespace:** COOKIES already fetched (boot loops the full 20-member enum; `COOKIES` at `src/i18n/types.ts:27`; loop at `bootStore.ts:451`). New strings under COOKIES → **zero mobile change**; a new namespace → one-line enum add. Backend must emit a checksum for the chosen namespace.
- **ATT:** entirely absent (no package, no code) — stays chat F.

### Part 4b adjacent observations (file:line, severity, out-of-scope)

1. **Save-throw path silent to user** — `user.tsx:181-186` + handler `:348`; catch re-throws on a fire-and-forget `onPress`, no error toast on throw. Severity **medium**. Relevant to C. Not fixed (read-only).
2. **B2 double-set `setAllowPromoEmails`** — `user.tsx:69-70`. Severity **medium**. In C's scope.
3. **`allowNotifications` dead toggle** — `user.tsx:47,299-303`. Severity **low**. Remove-vs-wire for C.
4. **B16 `console.error`** — `user.tsx:74`. Severity **low**. In C's scope.
5. **B17 untranslated `"DODAJ ZA BASE SITE"`** — `user.tsx:242`. Severity **low**. In C's scope.
6. **Dead web-only `firebaseAnalytics.ts`** — `src/lib/client/firebaseAnalytics.ts`. Severity **low**. C/Ω-adjacent.
7. **Dead `ConsentData`/`GlobalCookie` types** — `src/lib/types/cookie/`. Severity **low**. In C's scope.
8. **Unused `userPreferenceStorage` wrapper** — `src/lib/store/userPreferenceStorage.ts`. Severity **low**. Adopt for consent store or it's dead. For the G spec / Ω.

### Config-file impact: none (read-only audit). No implicit config-file dependency left unstated.
