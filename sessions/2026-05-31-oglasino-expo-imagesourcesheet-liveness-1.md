# Session — ImageSourceSheet hardcoded strings → translation keys (B15 mobile)

**Repo:** oglasino-expo
**Branch:** new-expo-dev (no switch, no commit, no push)
**Date:** 2026-05-31
**Slug:** imagesourcesheet-liveness (order 1)
**Type:** implementation — one file, three string swaps. Mobile-only.

---

## Task (one sentence)

Swap the three hardcoded Serbian strings in `src/components/ImageSourceSheet.tsx` for `t(...)` calls against the now-seeded keys (DIALOG + BUTTONS namespaces) and remove the `//TODO TRANSLATIONS ADD` marker.

---

## What I did

`src/components/ImageSourceSheet.tsx`:

1. Added two translator hooks at the top of the component, matching the repo pattern in `ImagesImport.tsx`:
   - `const tDialog = useTranslations(TranslationNamespace.DIALOG);`
   - `const tButtons = useTranslations(TranslationNamespace.BUTTONS);`
   - New imports: `TranslationNamespace` from `@/i18n/types`, `useTranslations` from `@/i18n/useTranslations`.
2. Replaced the three inline strings with the seeded keys (exactly as specified — no invented/renamed keys):

   | Old inline string | New call | Line |
   | --- | --- | --- |
   | `Izaberite izvor slike` | `tDialog('image.source.title')` | sheet title |
   | `Kamera` | `tButtons('image.source.camera')` | camera button |
   | `Galerija` | `tButtons('image.source.gallery')` | gallery button |

3. Removed the `//TODO TRANSLATIONS ADD` marker (previously line 13).

No other files touched.

---

## Pre-edit verification (per brief)

- File matched the brief: title at line 23, `Kamera` at 34, `Galerija` at 42, TODO at 13. No discrepancy → proceeded.
- `TranslationNamespace` enum (`src/i18n/types.ts`) contains **both** `DIALOG` (line 17) and `BUTTONS` (line 15).
- Boot fetch set: `src/lib/store/bootStore.ts:451` and `:541` iterate `Object.values(TranslationNamespace)`, so every enum member — including DIALOG and BUTTONS — is fetched at boot. Confirmed in `bootStore.test.ts:1084` (`expect(opts.ns).toEqual(Object.values(TranslationNamespace))`).

---

## Verification

- `npx tsc --noEmit` — clean (exit 0).
- `npm run lint` — 80 problems (0 errors, 80 warnings). Matches the 80-warning baseline; no new warnings/errors introduced (the touched file produces none).
- `npm test` (vitest) — 24 files passed, **325 tests passed**. Suite green.
- On-device / non-EN locale rendering NOT claimed — that's the Ψ smoke per the test-cases doc.

---

## Brief vs reality

Nothing. The brief matched the code exactly (line numbers, strings, TODO marker, both namespaces present in the enum and in the boot fetch). Implemented as written.

---

## Cleanup performed

- Removed the `//TODO TRANSLATIONS ADD` marker (the work it tracked is now done).
- No commented-out old strings left behind.
- No unused imports (both new imports are used; no imports became dead).
- No `console.log` / debug logging.
- No new `TODO`/`FIXME`.

---

## Obsoleted by this session

The `//TODO TRANSLATIONS ADD` marker and the three hardcoded `sr` strings documented in `audit-imagesourcesheet-liveness.md` §4 are now resolved. The audit's §5c recommendation (three new backend-seeded keys) has been fulfilled by the backend seed and adopted here.

---

## Conventions check

- **Part 4 (cleanliness):** no leftover TODO, no unused imports, no commented-out code, no debug logging. tsc/lint/test all run on the touched path.
- **Part 4a (simplicity):** two hooks for two genuinely-different namespaces (DIALOG title, BUTTONS labels) — not over-engineering; this is the established repo pattern (`ImagesImport.tsx` uses three namespace hooks the same way).
- **Part 4b (adjacent observations):** none.
- **Stack reminders:** keys are backend-seeded and resolved via `useTranslations`; no local catalog edits, no invented keys.

---

## Config-file impact

No edit needed to any of the four config files. This is an implementation session adopting already-seeded keys for a single live component; it does not adopt or retire a `state.md` Expo-backlog row by itself (no closure gate dependency surfaced). If Mastermind tracked this swap as a backlog row, retiring it is Docs/QA's edit to make — I draft nothing into the config files.

---

## For Mastermind

- **Three strings swapped** in `src/components/ImageSourceSheet.tsx`: title → `tDialog('image.source.title')`, camera → `tButtons('image.source.camera')`, gallery → `tButtons('image.source.gallery')`.
- **`//TODO TRANSLATIONS ADD` removed.**
- **Both namespaces confirmed in the boot fetch set:** DIALOG and BUTTONS are members of `TranslationNamespace` and `bootStore.ts` fetches every enum member at boot.
- **No keys invented or renamed** — all three match the backend seed exactly as listed in the brief.
- tsc clean, lint at 80-warning baseline (no new), 325 tests green.
- On-device / non-EN rendering is left for Ψ.
