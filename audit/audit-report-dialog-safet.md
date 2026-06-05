# Audit — ReportDialog unguarded `tErrors()` and the `safeT` pattern

**Repo:** oglasino-web · **Branch:** dev · **Date:** 2026-06-01 · **Mode:** read-only (findings only, no code changed)
**issues.md ref:** 2026-05-31 — "Web: `ReportDialog` renders error via unguarded `tErrors()` with no missing-key fallback"

---

## Headline (read this first)

Three things the brief asked, answered up front:

1. **`safeT` exists, but its contract does not actually work in production for the namespaced translators this app uses.** It detects "missing" by `result === key`. With a *namespaced* translator (`useTranslations('ERRORS')`), next-intl returns `"ERRORS.<key>"` on a miss — **not** the bare key — so `result === key` is always `false` and `safeT` never returns `null`. Its inline-English fallback is effectively dead code at runtime. The unit test only passes because its mock echoes the bare key, which real next-intl does not do. **This means "just apply `safeT` at :115" would NOT fix the gap** — on a missing key it would render `ERRORS.<key>` rather than degrade gracefully. (Q2, Q5)

2. **`:115` is the only unguarded *dynamic* site, and the REVIEW path is literally the same line — not a separate site.** One fix covers USER, PRODUCT, and REVIEW. No split pattern is created by guarding it. (Q3, Q4)

3. **next-intl does NOT throw on a missing key in this project.** No `onError`/`getMessageFallback` is configured anywhere, so the defaults apply: it calls `console.error` and renders the namespace-prefixed key string `"ERRORS.<key>"`. The "throws" half of the issue-entry premise is false; the real degraded state is a developer-looking string on screen plus a console error. (Q6)

A clean, idiomatic guard *does* exist and works: next-intl's own **`t.has(key)`** (verified `true` for present, `false` for missing). That, not `safeT`'s `=== key` check, is the primitive a real fix should use.

---

## Q1 — THE SITE

`src/components/popups/dialogs/ReportDialog.tsx:114-115`:

```tsx
} else if (result.errorTranslationKey) {
  setErrorMessage(tErrors(result.errorTranslationKey));
}
```

`tErrors` is the ERRORS-namespace translator (`ReportDialog.tsx:60` — `useTranslations(TranslationNamespaceEnum.ERRORS)`).

How `result.errorTranslationKey` reaches it: `result` is the resolved value of `sendReport(...)` (`ReportDialog.tsx:101`). `sendReport` (`src/lib/service/reactCalls/reportService.ts:55-89`) reads the wire `{field, code, translationKey}` errors, finds the first whose `code` is in the allow-list `REPORT_ERROR_CODES` **and** carries a `translationKey` (`reportService.ts:75-77`), and returns `{ success:false, alreadyReported:false, errorTranslationKey: reportError.translationKey }` (`reportService.ts:78-83`). So the string handed to `tErrors(...)` at :115 is a **backend-supplied** translation key.

**No missing-key fallback at this call today** — confirmed. It is a bare `tErrors(<dynamic key>)`. The only safety net is two layers up: if the backend code is *not* in the allow-list or carries no `translationKey`, `sendReport` returns no `errorTranslationKey` and the dialog falls to the generic `tErrors('report.send.fail')` at :117. But once a code *is* allow-listed and its `translationKey` is not seeded web-side, :115 renders it unguarded.

---

## Q2 — `safeT`

`src/lib/images/errorMapping.ts:110-113` (exported):

```ts
export function safeT(t: Translator, key: string, params?: Record<string, unknown>): string | null {
  const result = t(key, params);
  return result === key ? null : result;
}
```

`Translator` is a local type alias (`errorMapping.ts:102`): `(key: string, params?: Record<string, unknown>) => string`.

- **Signature:** takes the translator function `t`, a `key`, optional `params`. Returns `string | null`. It does **not** take a fallback string — the caller supplies the fallback via `safeT(...) ?? fallback`.
- **Documented intent (errorMapping.ts:104-109):** on a missing key, return `null` so the caller can use inline English, "because next-intl returns the key itself when a translation is missing."
- **Actual runtime behavior — the contract is broken for namespaced translators.** next-intl's missing-key fallback for a *namespaced* translator is **not** the bare key; it is `"<NAMESPACE>.<key>"` (see Q6, empirically verified). So `result === key` is `false` on a miss, and `safeT` returns the non-null string `"ERRORS.image.invalid"` / `"INPUT.image.processing.x"` instead of `null`. The inline-English fallback path it was written to enable **never executes in production**.
- **Why the test misses it:** `errorMapping.test.ts:80-83` mocks the translator as `(key) => key` ("simulates next-intl missing-key fallback"). That simulation is inaccurate for namespaced translators — real next-intl prepends the namespace. The test is green; the behavior is not what the test asserts.

So: `safeT` exists, is exported, but on a missing key it returns `"<NAMESPACE>.<key>"`, **not** `null` and **not** a generic fallback — and that is the opposite of its stated contract.

---

## Q3 — Every translation call in `ReportDialog.tsx`

| Line | Call | Key source | Guarded? |
|------|------|-----------|----------|
| :75  | `tValidation('report.missing.option')` | static literal | raw |
| :83  | `tValidation('report.missing.description')` | static literal | raw |
| :87  | `tValidation('report.short.description')` | static literal | raw |
| :98  | `tValidation('suspicion')` | static literal | raw |
| :113 | `tErrors('report.one.per.day')` | static literal | raw |
| **:115** | **`tErrors(result.errorTranslationKey)`** | **dynamic, backend-supplied** | **raw ← THE SITE** |
| :117 | `tErrors('report.send.fail')` | static literal | raw |
| :120 | `tErrors('report.send.fail')` | static literal | raw |
| :126 | `tErrors('report.send.fail')` | static literal | raw |
| :143 | `tInputs('report.reason')` | static literal | raw |
| :153 | `tDialog(getReportOptionTranslation(opt))` | derived from a fixed `Record<ReportOption,string>` (`option.1`–`option.10`, `reportOptionTranslations.ts:3-17`) | raw |
| :161 | `tInputs('report.description.label')` | static literal | raw |
| :173 | `tDialog('report.success')` | static literal | raw |
| :175 | `tDialog('notice.period')` | static literal | raw |
| :179 | `tDialog('button.cancel.label')` | static literal | raw |
| :187 | `tButtons('report.label')` | static literal | raw |

**Every call is raw — none use `safeT` or `t.has`.** But the risk classes differ:

- **:115 is the only call whose key is not known at compile time** — it comes off the wire. This is the genuine missing-key exposure.
- **:153 is derived but bounded** — keys come from an exhaustive in-repo `Record<ReportOption, string>`, not the backend. Same low-risk class as the static keys.
- **All others are compile-time literals.** A missing one is a seed bug caught the first time the dialog renders, not a wire-contract surprise.

Practical read: guarding only :115 does **not** create a split pattern in this file in any meaningful sense — the other calls are a different (static/bounded, deterministic) risk class. There is no second *dynamic backend-key* render in the file that would be left unguarded.

---

## Q4 — The REVIEW path

**Same file, same line — not a separate site.** `ReportDialog` is the single shared report dialog for all three report types (mounted via `ReportButton` from `ReceivedReviewCard.tsx` for REVIEW, and from `ProductFunctions.tsx` / `UserDetails.tsx` for PRODUCT/USER). The REVIEW-specific backend codes `REPORTED_REVIEW_ID_REQUIRED` and `REPORTED_REVIEW_NOT_FOUND` are members of the same `REPORT_ERROR_CODES` allow-list (`reportService.ts:32, :36`), so they resolve to the same `errorTranslationKey` field and render through the same `tErrors(result.errorTranslationKey)` at `ReportDialog.tsx:115`.

So the issue-entry phrasing ("the existing REVIEW path used the same unguarded render") is accurate, but to be precise: it is **the identical line**, reached by all report types. **One guard at :115 covers USER, PRODUCT, and REVIEW.** There is no parallel REVIEW render site that would need its own guard.

---

## Q5 — Existing `safeT` adoption

`safeT` is used in exactly one module — `src/lib/images/errorMapping.ts` (the image pipeline) — at:

- `buildUploadErrorTitle` — :128, :130, :132
- `stageLabel` — :217, :222
- `processingMessage` — :235, :243

**Established call shape:** `safeT(tNamespaced, key, params?)`, then the caller falls back with `?? <inline English>`:

```ts
// errorMapping.ts:135
return translated ?? englishFallback(err);

// errorMapping.ts:217-225
const localized = safeT(tInputs, key);
if (localized !== null) return localized;
const generic = safeT(tInputs, `${STAGE_KEY_PREFIX}default`);
if (generic !== null) return generic;
return englishStageLabel(stage);
```

Callers pass **no** fallback string into `safeT` itself; they pass `params` (e.g. `{ retryAfterSec }`, `{ filename, code }`) and handle the fallback at the call site with `??` / `if (x !== null)`.

**Caveat that matters for the fix:** every one of these adoptions is called with a *namespaced* translator (`tErrors` = ERRORS, `tInputs` = INPUT). Per Q2/Q6, `safeT` never returns `null` for those in production, so the `?? englishFallback` / `if (localized !== null)` branches are dead at runtime. Matching "the established usage shape" would therefore propagate a guard that does not actually guard. This is a pre-existing latent issue in the image pipeline, flagged below for Mastermind.

---

## Q6 — Missing-key behavior of next-intl in this project

**No global missing-key handler is configured.** `src/i18n/request.ts` returns only `{ locale, messages }` (no `onError`, no `getMessageFallback`). `NextIntlClientProvider` at `app/[locale]/layout.tsx:41` is passed only `locale`. A repo-wide grep for `getMessageFallback` / `onError` (next-intl sense) across `app/` and `src/` found nothing. So next-intl uses its built-in defaults.

**The defaults (from `node_modules/use-intl/dist/.../initializeConfig-*.js`):**

```js
function defaultGetMessageFallback(props) { return joinPath(props.namespace, props.key); }
function defaultOnError(error) { console.error(error); }   // does NOT throw
```

**Empirically verified** with this project's installed `use-intl` (`createTranslator({ namespace: 'ERRORS', messages, onError })`):

```
present key      -> "Sending failed"
MISSING key      -> "ERRORS.report.banned.words"     // namespace-prefixed, NOT the bare key
missing===passed -> false                            // so safeT's `result === key` never fires
t.has(present)   -> true
t.has(missing)   -> false
```

So, for this project:

- **next-intl does not throw on a missing key.** It calls `console.error` (default `onError`) and returns the fallback string.
- **The fallback string is namespace-prefixed:** `"ERRORS.report.<code>"`, not the raw bare key, and not blank.
- The user-visible degraded state at `:115` is therefore a developer-looking string (`ERRORS.report.<code>`) in the dialog's error line, plus a console error — **not** a thrown render error and **not** a crash.

The issue entry's "throws / renders the raw key" is half-right at most: no throw; and what renders is the *namespace-prefixed* key, not the raw bare key.

**Clean guard available:** next-intl exposes `t.has(key)` (verified above). A real fix at :115 should branch on `tErrors.has(result.errorTranslationKey)` and fall back to `tErrors('report.send.fail')` otherwise — this uses the library's own existence check rather than `safeT`'s broken `=== key` comparison.

---

## Summary table

| Q | Answer |
|---|--------|
| 1 | `:114-115` renders `tErrors(result.errorTranslationKey)` (backend-supplied key from `sendReport`); no missing-key guard. |
| 2 | `safeT` exists & is exported (`errorMapping.ts:110`); signature `(t, key, params?) → string \| null`; **intends** to return `null` on miss, but its `result === key` test never matches a namespaced translator's real fallback (`NAMESPACE.key`), so it returns the namespaced string instead — contract broken in production. |
| 3 | 16 translation calls; all raw; **`:115` is the only dynamic backend-keyed one**; the rest are static literals or a bounded local map (`:153`). No split-pattern risk from guarding :115 alone. |
| 4 | REVIEW uses **the same line** (`:115`) via the shared dialog + the two REVIEW codes in `REPORT_ERROR_CODES`. One fix covers all three report types. |
| 5 | `safeT` used only in `errorMapping.ts` (image pipeline); shape is `safeT(tNs, key, params?) ?? inlineEnglish`; no fallback string passed into `safeT`. All adoptions use namespaced translators → same latent breakage as Q2. |
| 6 | No `onError`/`getMessageFallback` configured → defaults: `console.error` (no throw) + `joinPath(namespace, key)`. Missing key renders `"ERRORS.<key>"`. `t.has(key)` is the working existence check. |
</content>
</invoke>
