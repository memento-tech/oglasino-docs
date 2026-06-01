# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-06-01
**Task:** AUDIT (read-only) — Re-confirm 3 consent spec-vs-code claims against current oglasino-web (issues.md ref: 2026-05-30 — "Web consent spec-vs-code drifts (docs)")

READ-ONLY audit. No code changed, nothing staged. Findings only.

---

## Findings

> **Refactor note that governs all three claims:** the 2026-05-30 audit (and the spec it
> fed) referenced a component `src/components/analytics/ConsentMode.tsx`. **That file no
> longer exists.** Neither does `src/lib/consent/consentSignals.ts`, `consentCookie.ts`,
> or `consentState.ts`. The consent implementation now lives in `src/lib/consent/*`
> (`cookie.ts`, `ssr.ts`, `sideEffects.ts`, `gating.ts`, `types.ts`) and
> `src/components/client/consent/*` (`ConsentBanner.tsx`, `consentDecisions.ts`,
> `ManageCookiesFooterButton.tsx`), with the store at `src/lib/store/useConsentStore.ts`.
> The *behaviors* the 2026-05-30 audit described are unchanged; only the file layout moved.
> (Verified: `cat`/`ls` on the old paths return "No such file or directory".)

### CLAIM 1 — the gtag consent 'update' snippet

**CURRENT TRUTH:** the runtime `gtag('consent','update',...)` call passes **only the four
Google signals — no `preference` key.**

- The single runtime update call lives in `src/lib/consent/sideEffects.ts:35-40`:

  ```ts
  gtag?.('consent', 'update', {
    analytics_storage: next.analytics_storage,
    ad_storage: next.ad_storage,
    ad_user_data: next.ad_user_data,
    ad_personalization: next.ad_personalization,
  });
  ```

  The local `GtagFn` type (`sideEffects.ts:8-17`) structurally permits only those four
  signal keys — a `preference` key would not type-check.
- A repo-wide grep for `gtag(...'consent', 'update'...)` returns exactly **one runtime
  site** (`src/lib/consent/sideEffects.ts:35`) plus one test assertion
  (`src/lib/consent/sideEffects.test.ts:46`, which asserts the same four-key object). There
  is no second update call anywhere.
- `preference` exists in the codebase **only** as a persisted consent field, never in the
  gtag update payload: `src/lib/consent/types.ts:10` (`preference: ConsentSignal;`) and the
  synchronous read `src/lib/consent/gating.ts:13`
  (`useConsentStore.getState().consent?.preference === 'granted'`).
- The SSR `gtag('consent', 'default', {...})` snippet is at `app/layout.tsx:49` — that is
  the `'default'` snippet, explicitly out of scope for this claim.

**Verdict:** the 2026-05-30 audit description **STILL HOLDS.** The update object lists only
`analytics_storage`, `ad_storage`, `ad_user_data`, `ad_personalization`. No `preference`.
(The spec's claim that it lists `preference` is wrong, as the 2026-05-30 audit said.)

### CLAIM 2 — the legacy migration shim

**CURRENT TRUTH:** there is **no** `globalCookie.cookieConsent → og_consent` migration shim.

- Repo-wide grep across `src/` and `app/` for `cookieConsent` and `migrateLegacyConsent`:
  **zero hits.** No code reads a legacy `cookieConsent` shape, client-side or SSR.
- The consent cookie is `og_consent`: `src/lib/consent/cookie.ts:3`
  (`export const OG_CONSENT_COOKIE_NAME = 'og_consent';`). It is read only as `og_consent`,
  with no legacy fallback — client read `readOgConsent` (`cookie.ts:19-31`, matches only
  `OG_CONSENT_COOKIE_NAME`), SSR read `readConsentForSsr` (`ssr.ts:42-50`,
  `cookieStore.get(OG_CONSENT_COOKIE_NAME)?.value`). Store hydration reads the same
  (`useConsentStore.ts:31`, `const existing = readOgConsent();`).
- The only migration-adjacent code is **not** the described shim:
  - `src/components/client/initializers/AppInit.tsx:28-45` self-heals the **separate**
    `globalCookie` (the UI-state cookie holding lang/theme/card-size — `GLOBAL_COOKIE_NAME =
    'globalCookie'`, `src/lib/service/oglasinoCookies.ts:5`): if preference consent is not
    granted but `globalCookie` still carries data, it calls `clearGlobalCookie()`. This is a
    self-heal of an unrelated cookie, not a `cookieConsent → og_consent` migration.
  - `src/components/client/consent/ConsentBanner.tsx:48` contains a **comment** reading
    "Hydration completes (cookie read + optional legacy migration)." No `cookieConsent` /
    legacy-shape read backs that comment in the consent code (the `hydrate()` it refers to is
    `useConsentStore.ts:30-33`, which only calls `readOgConsent()`) — it is a loose/vestigial
    comment (flagged below as a Part 4b observation; not fixed — out of read-only scope).

**Verdict:** the 2026-05-30 audit description **STILL HOLDS.** The documented
`globalCookie.cookieConsent → og_consent` migration shim does not exist in current code.

### CLAIM 3 — the cookie-settings route

**CURRENT TRUTH:** the dedicated cookie-settings page is at **`/owner/cookies`**, not `/owner/user`.

- Page file: `app/[locale]/owner/cookies/page.tsx` — the page renders the three consent
  toggles (necessary disabled, preference, analytics; lines 33-73), wired to `applyConsent`.
  App Router route `/[locale]/owner/cookies` → `/owner/cookies`.
- Owner-sidebar navigation entry that targets it: `src/lib/data/sectionNavigation.ts:78-79`
  (`labelKey: 'owner.account.cookies.label'`, `url: '/owner/cookies'`).
- `/owner/user` is a **different** page — the account/user settings page
  (`app/[locale]/owner/user/page.tsx`), linked at `src/lib/data/sectionNavigation.ts:58-59`
  (`url: '/owner/user'`). Distinct route, distinct purpose.
- (Aside: the footer "manage cookies" control, `ManageCookiesFooterButton.tsx`, does not
  navigate — it flips `requestReopen()` to re-open the banner. The dedicated toggle *page*
  is the sidebar `/owner/cookies` route above.)

**Verdict:** the 2026-05-30 audit description **STILL HOLDS.** The real path is
`/owner/cookies`; the spec's `/owner/user` (reported in two places) is wrong.

---

## Summary table

| Claim | Spec says | Current code | 2026-05-30 audit |
|-------|-----------|--------------|------------------|
| 1 — gtag update keys | lists `preference` | four Google signals only, no `preference` (`sideEffects.ts:35-40`) | still holds |
| 2 — legacy migration shim | shim exists | no `cookieConsent`/migration code anywhere | still holds |
| 3 — cookie-settings route | `/owner/user` (×2) | `/owner/cookies` (`sectionNavigation.ts:78-79`) | still holds |

All three 2026-05-30 audit descriptions are re-confirmed against current `oglasino-web` code.

## Files touched

- None (read-only audit).

## Tests

- Not run — read-only audit, no code changed.

## Cleanup performed

- none needed (read-only).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change.
- issues.md: no change by me. **Note for Docs/QA:** this audit re-confirms the open
  2026-05-30 "Web consent spec-vs-code drifts (docs)" entry; the spec correction
  (`features/consent-mode-v2.md`) it points to remains owed. Draft pointer in "For
  Mastermind" below. I did not edit issues.md or the spec.

## Obsoleted by this session

- Nothing in code. The audit confirms `features/consent-mode-v2.md` is stale on three
  points (claims 1–3) and references files that no longer exist (`ConsentMode.tsx` etc.) —
  but the spec lives in `oglasino-docs/` and is Docs/QA's to correct, not mine.

## Conventions check

- Part 4 (cleanliness): confirmed — no code changed.
- Part 4a (simplicity): N/A — read-only; see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): two observations flagged in "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: none.

## Known gaps / TODOs

- None. All three findings are confirmed first-hand from direct file reads
  (`sideEffects.ts`, `cookie.ts`, `ssr.ts`, `useConsentStore.ts`, `AppInit.tsx`,
  `ConsentBanner.tsx`, `sectionNavigation.ts`, `app/[locale]/owner/cookies/page.tsx`),
  cross-checked with repo-wide grep. (Mid-session the tool layer dropped output
  intermittently — see "For Mastermind" — but all reads ultimately returned and every
  quote above is verbatim from disk.)

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **Audit result:** all three 2026-05-30 consent claims re-confirm against current code
  (details above). The drift is entirely on the docs side (`features/consent-mode-v2.md`),
  not the code.

- **Drafted issues.md / spec pointer (for Docs/QA, I did not apply it):** the open
  2026-05-30 "Web consent spec-vs-code drifts (docs)" issue can be actioned — the spec
  `features/consent-mode-v2.md` needs three corrections, all re-confirmed today:
  1. The client `gtag('consent','update',...)` snippet lists only the four Google signals
     (`analytics_storage`, `ad_storage`, `ad_user_data`, `ad_personalization`) — remove the
     `preference` key from the documented snippet. (code: `src/lib/consent/sideEffects.ts:35-40`)
  2. The `globalCookie.cookieConsent → og_consent` migration shim was never built — remove
     it from the spec. (No `cookieConsent`/migration code exists; cookie name is `og_consent`
     at `src/lib/consent/cookie.ts:3`.)
  3. The cookie-settings route is `/owner/cookies` (two places say `/owner/user`) — correct
     both. (code: `app/[locale]/owner/cookies/page.tsx`; nav `src/lib/data/sectionNavigation.ts:79`)
  4. **New, beyond the 2026-05-30 list:** if the spec names `src/components/analytics/ConsentMode.tsx`
     (or `consentSignals.ts` / `consentCookie.ts` / `consentState.ts`) as the implementation
     site, those files no longer exist — the consent code is now under `src/lib/consent/*`
     and `src/components/client/consent/*`. Worth updating any file pointers in the spec.

- **Part 4b adjacent observations (not fixed — read-only audit):**
  1. **Loose comment** — `src/components/client/consent/ConsentBanner.tsx:48` says
     "(cookie read + optional legacy migration)", but no legacy-cookie migration code backs
     it; the referenced `hydrate()` (`useConsentStore.ts:30-33`) only calls `readOgConsent()`.
     Severity: low (could mislead a future reader into thinking a migration exists). I did
     not fix this — out of read-only scope.
  2. **Stale spec file pointers** — `features/consent-mode-v2.md` (oglasino-docs, not my
     repo) references `ConsentMode.tsx` and sibling files that were deleted in a refactor.
     Severity: low. Cross-repo / docs-owned; flagged for the Docs/QA correction pass above.

- **Process note (harness):** this session hit a persistent tool-output delivery problem —
  `Read`/`Bash`/`Agent` calls frequently returned empty and outputs arrived batched several
  turns later; the `Read` tool also briefly served phantom content for files that do not
  exist on disk (a 30-line `ConsentMode.tsx` that `cat`/`ls` confirm is absent). All reads
  ultimately returned and every finding was cross-checked against grep that did return; the
  quotes above are verbatim from disk. Flagging so the empties in the transcript aren't
  mistaken for missing evidence.

- Closure gate: no config-file edit was made by me; the only config-file dependency
  (the consent spec correction) is drafted above for Docs/QA, not applied. Nothing else flagged.
