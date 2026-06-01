# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-06-01
**Task:** AUDIT (read-only) — ReportDialog unguarded `tErrors()` and the `safeT` pattern (issues.md 2026-05-31)

## Implemented

- Read-only audit, no code changed. Full findings in `.agent/audit-report-dialog-safet.md` (answers Q1–Q6 with `file:line` citations and an empirical next-intl probe).
- Confirmed `ReportDialog.tsx:114-115` renders `tErrors(result.errorTranslationKey)` — a backend-supplied key from `sendReport` (`reportService.ts:78-83`) — with no missing-key guard.
- **Key finding the brief did not anticipate:** `safeT` (`errorMapping.ts:110-113`) detects "missing" via `result === key`, but next-intl's missing-key fallback for a *namespaced* translator is `"<NAMESPACE>.<key>"`, not the bare key. So `safeT` never returns `null` in production for the namespaced translators it's used with — its inline-English fallback is dead at runtime, and "just apply `safeT` at :115" would not actually fix the gap. Verified empirically against the installed `use-intl`.
- Confirmed next-intl does **not** throw on a missing key here (no `onError`/`getMessageFallback` configured anywhere) — it `console.error`s and renders `"ERRORS.<key>"`. The issue entry's "throws" premise is false.
- Confirmed the REVIEW path is the *same line* (`:115`) via the shared dialog + REVIEW codes in `REPORT_ERROR_CODES` — one guard covers USER/PRODUCT/REVIEW. Identified `t.has(key)` as the working existence-check primitive (verified) the real fix should use instead of `safeT`'s `=== key`.

## Files touched

- None (read-only audit). New artifact: `.agent/audit-report-dialog-safet.md` (findings doc); this summary + `last-session.md`.

## Tests

- Ran: none against the app (read-only). Ran one throwaway probe with the installed `use-intl` `createTranslator` to confirm namespaced missing-key return value and `t.has()` behavior; probe file removed.
- Result: confirmed `missing key → "ERRORS.<key>"`, `result === key → false`, `t.has(missing) → false`, `t.has(present) → true`.
- New tests added: none.

## Cleanup performed

- Removed the throwaway `use-intl` probe script (`_intltest.mjs`) after running it. Otherwise none needed — no code changed.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change.
- issues.md: no change authored by me (read-only). **Drafted amendments** for the existing 2026-05-31 ReportDialog entry are in "For Mastermind" below for Docs/QA to apply — the entry's "throws" premise and its implied "just use `safeT`" remedy are both contradicted by this audit.

## Obsoleted by this session

- Nothing deleted. The audit surfaces that `safeT`'s missing-key path and the image-pipeline `?? englishFallback` branches are effectively dead at runtime, but I did not change them (out of scope, flagged for Mastermind).

## Conventions check

- Part 4 (cleanliness): confirmed — no code changed; probe script removed.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): flagged in "For Mastermind" (the `safeT` runtime breakage + the inaccurate test mock).
- Part 6 (translations): N/A this session (no keys added/changed). The audit concerns translation-key *rendering*, not the seed.
- Other parts touched: Part 7 (error contract) — confirmed the `{field, code, translationKey}` wire shape is what `sendReport` consumes (`reportService.ts:41-53, :75-83`).

## Known gaps / TODOs

- I did not propose or write a fix (brief said findings only). The recommended primitive (`tErrors.has(key)`) is noted in the findings doc as evidence, not as an applied change.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code added.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing (the throwaway probe script doesn't count as repo code).

- **(high) `safeT`'s missing-key contract does not work in production for namespaced translators.** `src/lib/images/errorMapping.ts:110-113`. `result === key` never matches next-intl's real namespaced fallback (`"<NAMESPACE>.<key>"`), so `safeT` returns the namespaced string instead of `null`, and every `?? englishFallback` / `if (localized !== null)` branch in `errorMapping.ts` (`buildUploadErrorTitle`, `stageLabel`, `processingMessage`) is dead at runtime. Severity high because it's a user-facing graceful-degradation net that silently doesn't degrade — on a genuinely missing image key, users would see `INPUT.image.processing.x` / `ERRORS.image.x` rather than the inline English the code intends. I did not fix this because it is out of scope for a read-only audit. Correct primitive is next-intl's `t.has(key)`.

- **(medium) The `safeT` unit test asserts a behavior real next-intl does not produce.** `src/lib/images/errorMapping.test.ts:80-83` mocks the translator as `(key) => key`, but a namespaced next-intl translator returns `"<NAMESPACE>.<key>"`. The test is green while the production guard is broken — a future reader will trust a guard that doesn't hold. Out of scope; flagged.

- **Drafted issues.md amendment (for Docs/QA to apply to the existing 2026-05-31 ReportDialog entry):** append a clarification note —
  > **Audit 2026-06-01 (`oglasino-web`, `.agent/audit-report-dialog-safet.md`):** Two premises in this entry need correcting. (1) next-intl does **not** throw on a missing key in this project — no `onError`/`getMessageFallback` is configured, so it `console.error`s and renders the *namespace-prefixed* key `"ERRORS.<code>"` (not the raw bare key, not a throw). (2) The implied remedy "use the `safeT` pattern from `errorMapping.ts`" would not work: `safeT` detects missing via `result === key`, but next-intl's namespaced fallback is `"<NAMESPACE>.<key>"`, so `safeT` never returns `null` for these translators — confirmed empirically. `:115` is the only dynamic backend-keyed render and the REVIEW path is the same line, so one guard covers all three report types. The correct guard primitive is next-intl's own `tErrors.has(key)`. Separately, this exposes that `safeT`'s missing-key path and the image-pipeline fallbacks that depend on it are dead at runtime (logged as adjacent observations above) — worth its own triage.

- **Suggested next step:** if a fix brief follows, scope it to (a) guard `:115` with `tErrors.has(result.errorTranslationKey) ? tErrors(...) : tErrors('report.send.fail')`, and (b) decide separately whether to repair `safeT` (switch to `t.has`) given the image-pipeline net is also affected — that's a larger, cross-cutting change than the one-line ReportDialog guard and probably its own brief.

- **Config-file closure:** the only config-file dependency is the drafted issues.md amendment above (queued for Docs/QA). No other config-file edit is required by this session.
</content>
