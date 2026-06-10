# Session summary — oglasino-web — legal-document fact verification (Q8)

- **Date:** 2026-06-10
- **Repo:** oglasino-web
- **Slug:** legal-doc-audit  **Order:** 1
- **Branch:** dev
- **Session type:** READ-ONLY AUDIT (no code changes)

## Brief vs reality
Not applicable in the usual sense — this *was* a fact-finding audit, not an implementation brief. The "brief vs reality" gaps found are the audit findings themselves (see Findings). No code was written.

One process tension worth flagging: the brief says "Do not modify, create, or delete any file. Answers only," while CLAUDE.md mandates writing this summary to `.agent/`. I read the brief's intent as "do not touch the audited codebase," and treat the `.agent/` session log as the agent's own out-of-band record (not part of the audited surface), so I wrote it. If Igor wants a strict zero-file-write audit next time, say so and I will deliver answers in chat only.

## Task
Verify nine Privacy-Policy factual claims (Q8 a–i) against the actual web code: GA4 loading model, Consent Mode signals, reCAPTCHA scope, banner version re-display, cookie-preferences page + links, web push, password path, region pinning, cookie lifetimes. Dual-source every claim (direct Read + grep).

## Findings (verdicts)
- **(a) GA4 loading model — VERIFIED-FALSE (advanced, not basic).** Script loads upfront gated only by `NEXT_PUBLIC_GA4_MEASUREMENT_ID` (`app/layout.tsx:17,70-81`) with `gtag('consent','default',…)` default-denied (`layout.tsx:49-55`). Advanced mode → cookieless pings on decline, so PP §7 "no analytics data is collected" is false (cookie half is true).
- **(b) Consent signals — VERIFIED-TRUE.** Default `layout.tsx:49-55`; update `sideEffects.ts:35-40`; `ad_*` hard-denied as literals (`types.ts:12-14`), enforced by all builders (`consentDecisions.ts`), `isValidConsent` (`types.ts:45-47`), and `sanitizeForSnippet` (`ssr.ts:31-37`). No enabling path.
- **(c) reCAPTCHA — PARTIAL.** v2 invisible (`ReCaptchaWrapper.tsx:30`, `react-google-recaptcha@^3.1.0`). Present: registration (`RegisterDialog.tsx:222`), listing creation (`MetaDataProductDialog.tsx:52` via `CreateNewProductDialog.tsx:171`), plus review/suggest-category/report. **Login has none** (`LogInDialog.tsx`, grep=0). Doc's login claim false; three surfaces undisclosed.
- **(d) Banner version re-display — NOT FOUND.** Banner opens only on absent cookie (`ConsentBanner.tsx:51`) or footer reopen flag (`:57`). `version:1` is a schema literal (`types.ts:16,48`), not a policy/terms stamp. No version-comparison logic. §12 promise unimplemented.
- **(e) Cookies page + links — PARTIAL.** Page exists (`app/[locale]/owner/cookies/page.tsx`). Account nav links it (`sectionNavigation.ts:78-81` → `DashboardSidebar.tsx:29` → owner layout). Footer does NOT link the page — `ManageCookiesFooterButton.tsx:18` reopens the banner.
- **(f) Web push — VERIFIED-TRUE.** `devicePush.ts:23-26` `getToken` + VAPID + SW registration; `requestPermission` (`:36`); `attachPushTokenToBackend` (`:46`); wired `UseTokenRefresh.tsx:103`; SW at `public/firebase-messaging-sw.js`. PP §2.11 correct; §2.15 "app-only" contradicted.
- **(g) Password path — VERIFIED-TRUE.** Client→Firebase SDK only: `authService.ts:169,179`, `passwordResetService.ts:76`. Backend gets email-only reset request (`passwordResetService.ts:36`) and ID-token sync (`authService.ts:134`). No password in any api route/server action. PP §2.1 accurate.
- **(h) Region pinning — VERIFIED-TRUE.** `vercel.json:4` `"regions": ["fra1"]`.
- **(i) Lifetimes — VERIFIED.** `og_consent` Max-Age = 31,536,000s/365d (`cookie.ts:5,37`); SameSite=Lax, Secure on HTTPS. No `_ga` config anywhere (grep cookie_expires/flags/domain → none); `_ga` = GA default ~2y, set only on analytics grant. Banner writes only `og_consent`.

Adjacent findings delivered in chat (7 items): advanced-mode wording, login reCAPTCHA gap, undisclosed reCAPTCHA surfaces, footer-not-a-link, no version re-prompt, web push is live, `_ga` lifetime undisclosed.

## Obsoleted by this session
Nothing — read-only audit, no code touched.

## Cleanup performed
None needed — no files modified.

## Conventions check
- Part 4 (cleanliness): no code changed; nothing to clean.
- Part 4a (simplicity): N/A — no implementation.
- Part 4b (adjacent observations): captured as the 7 adjacent findings above.
- Verification discipline (brief): every claim dual-sourced (direct Read + plain grep). Noted that `rg` highlight output is corrupted in this environment and was not relied upon.
- Hard rules: no commit/push/merge/branch change; no deploys; no edits to other repos or the four config files; no `next.config` edits. Honored.

## Config-file impact
No edit required to `conventions.md`, `decisions.md`, `state.md`, or `issues.md` by the audit itself. However, the audit surfaces seven factual gaps between the Privacy Policy text and the web code (see Findings/Adjacent). These are **legal-doc** corrections, not web-repo config changes — they belong to whoever owns the PP/Terms content, routed via Igol/Mastermind. No `issues.md` entry is strictly mandated by repo convention, but items (c) login reCAPTCHA gap and (d) missing version re-prompt are latent functionality gaps Mastermind may want logged.

## For Mastermind
The Privacy Policy currently asserts several things the web code does not back:
1. PP §7 "no analytics data is collected" on decline — code runs **advanced** Consent Mode (cookieless pings still fire). Either soften the PP wording or switch to basic mode (load gtag only after analytics grant).
2. PP login-reCAPTCHA claim — login dialog has no reCAPTCHA. Add it, or drop login from the doc.
3. PP §7/§12 version-change re-prompt — no mechanism exists. Implement a policy-version stamp in `og_consent` + re-show, or remove the promise.
4. PP §7/§9 "footer links to the cookie page" — footer reopens the banner instead. Decide intended behavior.
5. PP §2.15 "push is app-only" — web push is live; correct the doc.
6. PP §7 cookie-lifetime disclosure — add `og_consent` (365d) and `_ga` (GA default ~2y, unconfigured).
7. Undisclosed reCAPTCHA on reviews/suggest-category/report — add to the doc's processing disclosures if required.
