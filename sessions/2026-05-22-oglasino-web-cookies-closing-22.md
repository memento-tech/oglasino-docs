# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-22
**Task:** Brief 3j-investigate — language change broken when consent declined (read-only investigation)

## Confirmed task

Trace why a same-tenant language click in `PortalConfigDialog` results in "dialog closes, no navigation" when preference consent is declined, while the same click works when consent is granted.

## Cause

The cause is **not** inside `PortalConfigDialog.navigate()`. The call site behaves correctly under declined consent: the cookie write is skipped (intended) and `router.push(target)` is unconditionally reached with the right URL. The breakage happens one layer down, in `proxy.ts`'s cookie-wins-over-URL redirect (`proxy.ts:48-65`), which is **not consent-aware**.

Trace under declined consent (cookie carries a stale `lang: 'sr'` from a prior consent-granted session — or from before the consent system was introduced in commit `51df7e9`):

1. User on `/rs-sr/...` clicks EN.
2. `PortalConfigDialog.tsx:165` → `navigate('rs', 'en')`.
3. `PortalConfigDialog.tsx:84` — `isSameTenant = true`.
4. `PortalConfigDialog.tsx:89-91` — `isSameTenant && isPreferenceConsentGranted()` is `false` → **cookie write skipped**. Cookie still has `lang: 'sr'`.
5. `PortalConfigDialog.tsx:96-98` — `target = getLocalizedPath('/rs-sr/...', 'en') + queryString = '/rs-en/...'`.
6. `PortalConfigDialog.tsx:100` — `router.push('/rs-en/...')` fires.
7. `PortalConfigDialog.tsx:102` — `onClose()` runs synchronously, dialog closes.
8. Next.js App Router issues an RSC fetch for `/rs-en/...`; `proxy.ts` runs.
9. `proxy.ts:41` — `ALLOWED_LANGS_PER_TENANT['rs'].has('en')` is true, invalid-lang branch skipped.
10. `proxy.ts:48-65` — reads `globalCookie`. `cookieLang = 'sr'`, `urlLang = 'en'`, `cookieLang !== urlLang`, `ALLOWED_LANGS_PER_TENANT['rs']?.has('sr')` is true → **307 redirect to `getLocalizedPath('/rs-en/...', 'sr') = '/rs-sr/...'`**.
11. Client router follows the 307 back to `/rs-sr/...` — the URL the user started on.
12. End state: URL unchanged, dialog already closed by step 7.

That is exactly the reported symptom. The proxy reverts the user's URL change because the cookie says "lang=sr" and the proxy treats the cookie as authoritative — but the consent gate has prevented the cookie from being updated to match the user's new choice. The two layers' invariants are inconsistent: the cookie-write layer respects consent; the cookie-read-and-redirect layer does not.

Under consent **granted**, the cookie is updated to `lang: 'en'` at step 4 before step 6 fires. By the time `proxy.ts` reads the cookie at step 10, `cookieLang === urlLang`, the redirect is skipped, and navigation completes. That is why scenarios 1 and 3 of Brief 3j's smoke pass.

### Brief's diagnostic checklist — each answered

- **Does `navigate(tenant, lang)` get called?** Yes. `PortalConfigDialog.tsx:164-166` wires the language button's `onClick` to `navigate(baseSite.code, lang.code as Lang)`. There is no consent-aware short-circuit on the button itself (no `disabled` prop, no guarded handler). The button fires identically in both consent states.
- **What does `navigate()` do under declined consent?** Walked line-by-line above (steps 3-7). `isSameTenant` test fires (84), consent-gated cookie write block (89-91) is skipped, `target` computed (96-98), `router.push(target)` (100), `onClose()` (102). No throw, no early return.
- **Is `router.push(target)` reached?** Yes — unconditionally. The consent guard only gates the cookie write block (89-91), not the navigation.
- **Could the wrapped `useRouter` from `@/src/i18n/navigation` have consent-aware behavior?** Not applicable here. `PortalConfigDialog.tsx:20` imports `useRouter` from **`next/navigation`** (the plain Next.js router), not from `@/src/i18n/navigation`. Audited the wrapped router anyway (`src/i18n/navigation-client.tsx:59-86`) — zero consent references. It only prefixes hrefs with `routingLocale`.
- **Could `getLocalizedPath()` return the current URL unchanged?** No. `src/i18n/getLocalizedPath.ts:7-15` swaps the `-lang` portion of the leading `<tenant>-<lang>` segment. From `/rs-sr/...` with `newLang='en'` it returns `/rs-en/...` — different string. `router.push` is not a no-op.
- **Could `onClose()` be running before `router.push()`?** No. JavaScript execution order is sequential: `router.push(target)` (line 100) runs before `onClose()` (line 102). `router.push` is synchronous (queues the navigation and returns immediately), so the navigation is initiated before the dialog closes. `onClose` does not abort an in-flight router navigation — and even if it did, that wouldn't explain why the consent-granted case works through the same code path.

### Possible diagnoses listed in the brief — verdicts

- **"The consent gate was placed wrong — gates something that shouldn't be gated."** No. The gate at `PortalConfigDialog.tsx:89` correctly gates only the persistence side effect (cookie write). The navigation is correctly outside the gate.
- **"`router.push()` is being called but immediately overridden by the dialog's close + the wrapped router's interaction with consent state."** No on the wrapped router (not used here). `router.push` IS being called and IS being immediately overridden — by `proxy.ts`'s cookie-wins 307, not by `onClose()`.
- **"A throw or early return somewhere swallows the navigation when consent isn't granted."** No throw, no early return inside `navigate()`. The swallow happens server-side in the middleware redirect.
- **"The button itself has a consent-aware disabled state somewhere we haven't audited."** No — confirmed by reading `PortalConfigDialog.tsx:160-169`.

### Why this didn't surface in Brief 3j's smoke

Brief 3j's manual scenarios (per the session summary on disk) covered: cookie write on same-tenant language change with consent granted; consent-gate enforcement (write skipped when declined — but did not check post-navigation URL); same-tenant gate enforcement on cross-tenant click; "cookie-wins middleware redirect unaffected" tested with consent granted (cookie and URL agree, no redirect needed). The combination "consent declined AND `lang` already in cookie AND user clicks a different language" — the bug's full precondition — was not in the smoke list. Brief 3j is not the introducer; the bug predates it.

### Where the bug actually originated

Brief 3j is **not** the introducer despite the brief listing it as the suspect. Brief 3j inlined the cookie write from `useLanguageStore` to `PortalConfigDialog`, but the consent gate already existed in `useLanguageStore.setLang` (per `audit-language-routing.md` Area 6: "writes the same to the global cookie via `updateGlobalCookie('lang', lang)` (consent-gated, wrapped in try/catch)"). The pre-3j and post-3j shapes both consent-gate the cookie write and both unconditionally call `router.push(target)`.

The true introducer is commit `51df7e9 Added cookies V2 and some other fixes` (May 21), which (per `git log -- proxy.ts src/lib/service/oglasinoCookies.ts src/lib/consent/`) brought the consent system in alongside the cookie-wins proxy logic. Before that commit, cookie writes were unconditional, so a `lang` cookie always matched the user's last URL click — no mismatch was possible. After that commit, the gate-on-write but no-gate-on-read asymmetry exists. Brief 3j is incidental.

### Affected population

The bug only manifests when the cookie has a `lang` field that the user can no longer overwrite. Today (post-consent-system), the only writer of `globalCookie.lang` is `PortalConfigDialog.tsx:90`, which is consent-gated. So the affected users are:

1. **Users who had consent granted, clicked a language, then declined or revoked consent.** Cookie persists; user is stuck on the cookie's language and cannot override via the UI.
2. **Users with cookies predating the consent system.** Before commit `51df7e9` the cookie write was unconditional, so anyone who clicked a language pre-consent-system has a `lang` cookie that they may not be able to update post-consent-system without re-granting.
3. **A user with no `globalCookie` and no `lang` field would not hit this** — `proxy.ts:54` short-circuits on `cookieLang &&`. So a truly fresh visitor with declined consent navigates fine. The brief's "or no consent set at all" caveat only reproduces the bug if such a visitor happens to have a stale `lang` cookie from before they made the consent decision (or from a stale cookie that survives consent withdrawal).

## Files touched

None. Read-only investigation.

## Tests

None run. Read-only investigation.

## Cleanup performed

None needed — no code changes.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change for the investigation itself. A fix brief (separate session) would likely append a decision entry on whether cookie-wins should respect consent, or some equivalent — that is Mastermind's call, not mine.
- state.md: no change for the investigation.
- issues.md: this bug is the live "Item 4" of `features/cookies-closing.md` per the brief; if items there get retired to `issues.md` or vice versa, Docs/QA decides. No edit drafted in this session.

## Obsoleted by this session

Nothing.

## Conventions check

- Part 3 (hard rules): respected — no commits, no pushes, no cross-repo edits, no four-config-file writes, no `next.config.js` edits, no new docs in `docs/`. Investigation only.
- Part 4 (cleanliness): N/A — no code changes.
- Part 4a (simplicity): N/A — no code changes.
- Part 4b (adjacent observations): see "For Mastermind."
- Part 5 (session summary): this file + `.agent/last-session.md` copy below.
- Part 6 (translations): N/A.
- Part 7 (HTTP error contract): N/A.
- Part 11 (trust boundaries): the cookie-wins redirect is a display-preference routing decision, not a trust-boundary surface — relevant context but not in scope for the bug's diagnosis.

## Known gaps / TODOs

None for the investigation. The brief says "Don't fix yet. Find the cause. ... Do not propose a fix. Surface the cause; Mastermind drafts the fix brief." — followed.

## For Mastermind

- **Correction to an earlier framing.** An initial draft of this summary called `PortalConfigDialog.navigate()` "internally consistent" and put the locus solely at `proxy.ts`. Igor pushed back on line 89 specifically and he is right: line 89 is the proximate cause of the no-op viewed from the dialog side, and the proxy's cookie-wins read is the proximate cause viewed from the routing side. They are the same asymmetry. The fix can live at either end. I am leaving both ends on the table.

- **The upstream tension is in the spec itself.** `features/cookies-closing.md`:
  - Line 392: "`globalCookie.lang` writes happen on PortalConfigDialog language change, **under preference consent gate**."
  - Line 394: "`proxy.ts` reads `globalCookie.lang` and redirects when cookie language differs from URL language and target tenant allows the cookie's language." — **no consent term on the read side.**
  This is a spec asymmetry, not just an implementation slip. Resolving the bug requires resolving the spec. Until the spec settles, all four options below remain live design choices.

- **What `proxy.ts:48-65` does and why it lacks a consent check.** This is the dialog-side complement to line 89 and the place Igor explicitly asked to inspect. Walkthrough:
  - `proxy.ts:48` — `const raw = request.cookies.get(GLOBAL_COOKIE_NAME)?.value;` reads `globalCookie` from the incoming HTTP request. Edge-runtime read; no `document.cookie`. Same cookie that `PortalConfigDialog.tsx:90` writes to.
  - `proxy.ts:49` — if the cookie is absent (truly fresh visitor, never written), the whole block is skipped and the proxy falls through to `defaultMiddleware` at line 68. URL wins.
  - `proxy.ts:50-52` — parse the JSON-encoded cookie and extract `lang`. Body looks like `{"portalCardSize":"small","lang":"sr"}`; `cookieLang` may be `undefined` if `lang` was never written.
  - `proxy.ts:53-56` — the cookie-wins predicate. Three terms: `cookieLang &&` (cookie carries a lang), `cookieLang !== urlLang` (URL disagrees with cookie), `ALLOWED_LANGS_PER_TENANT[tenant]?.has(cookieLang)` (target tenant supports the cookie's lang — sanity guard against e.g. `/me-cnr/...` redirecting to `/me-sr/...` when `me` doesn't allow `sr`).
  - `proxy.ts:58-60` — when all three terms hold, build a 307 redirect to the same path with the language swapped to the cookie's value via `getLocalizedPath`. Return immediately; `defaultMiddleware` does not run.
  - `proxy.ts:62-64` — malformed-cookie `catch` falls through. URL wins on parse error.

  **Intent:** "the user's persisted language preference wins over what URL they happened to land on." Use cases the design protects:
  - Address-bar entry of a bare path that next-intl would otherwise resolve to the default locale.
  - Unlocalized links that resolve through next-intl's default-locale fallback.
  - SSR/initial-page-load consistency with the user's last-chosen language.

  **Why no consent check on the read?** Because the design was written on the implicit assumption that **the write-side gate at `PortalConfigDialog.tsx:89` is the sole consent boundary** — "if there's a `lang` in the cookie, the user must have granted consent to write it, so the read can trust it." That assumption holds at write time but fails at read time in two realistic populations:
  1. **Granted → wrote → revoked.** User accepts preference cookies, clicks a language (cookie gets `lang: 'sr'`), later reopens the banner and switches to declined. The cookie persists — `useConsentStore.setConsent` at `src/lib/store/useConsentStore.ts:34-37` only touches the `og_consent` cookie, not `globalCookie`. So `cookieLang` is still set and authoritative for the proxy, but the user can no longer overwrite it from the dialog because line 89 blocks writes.
  2. **Cookies predating the consent system.** Before commit `51df7e9` (May 21, 2026, "Added cookies V2 and some other fixes"), the write was unconditional — any user who clicked a language back then has a `lang` in their cookie that was never tied to an explicit consent decision in the current sense. The write gate is non-retroactive.

  In both cases, the proxy honors a cookie value the user cannot control. They get stuck on the cookie's language until they re-grant consent.

  **Consent Mode v2 framing.** Beyond the asymmetry argument, the cookie-wins effect itself — using stored data to alter the rendered output — is arguably the kind of behavior preference-consent is supposed to gate, not just the act of writing. Under that reading, even option 1 (drop the write gate, treat `lang` as functional) is debatable: the user is still being routed by a stored value, which the GDPR-style "preference cookie" framing would want gated. The cleanest split is functional-cookie status (no consent involved on either side) vs preference-cookie status (consent gates both sides). The current split — preference on write, no gate on read — is the unstable middle that produces this bug.

- **The fix shape is a design choice with at least four live options** — I am not picking one, but listing them so the fix brief can pick from a known set:
  1. **Drop the consent term at `PortalConfigDialog.tsx:89`.** Treat `globalCookie.lang` as a functional/routing cookie rather than a preference cookie. The cookie's only consumer is `proxy.ts`'s URL-routing redirect; arguably that is strictly-necessary-for-the-service behavior, not a "remember my preferences" cookie in the Consent Mode v2 sense. With the gate gone, every same-tenant language click writes the cookie, and the proxy's cookie-wins matches the URL the user just clicked. Trade-off: changes the cookie's consent category — would need a corresponding update in `features/cookies-closing.md` (line 392 and Item 4's section 5/B1-B) and in any consent-banner copy that promises the user `lang` is gated. Decisions.md likely needs an entry on the recategorization.
  2. **Consent-gate the cookie read at the proxy.** Read `og_consent` from `request.cookies` inside `proxy.ts` and skip the cookie-wins branch when preference consent is not granted. Symmetric to option 1 but at the read end — keeps `lang` categorized as a preference cookie. Trade-off: the cookie's `lang` value is now ignored under declined consent even for users who legitimately had it set under granted-then-withdrawn consent — meaning consent withdrawal silently invalidates the user's persisted language preference on every page load. May or may not be desired; depends on what "consent declined" is supposed to mean for already-persisted data. Implementation note: `src/lib/consent/cookie.ts:readOgConsent` is client-only (`document.cookie`); the edge runtime would need a `request.cookies.get('og_consent')`-based variant.
  3. **Stop writing to `globalCookie.lang` entirely; derive the user's language from the URL alone.** The cookie's `lang` field is only used by the proxy to redirect URL→lang. If we trust the URL as the single source of truth for language, the cookie field becomes unnecessary and the cookie-wins branch (`proxy.ts:48-65`) can be deleted. Collapses the bug at the root by removing one of the two competing sources of truth. Trade-off: the "cookie wins" guarantee (user lands on the right language when they type a bare path or follow a no-locale link) is lost. May or may not be acceptable depending on the feature spec's intent for cookie-wins. Spec impact larger than options 1 or 2.
  4. **Clear `globalCookie.lang` when consent is declined or withdrawn.** Add a cleanup pass on consent decline/withdraw that removes the `lang` field from the cookie (or the whole `globalCookie`). With no `lang` in the cookie, the cookie-wins branch short-circuits naturally on `cookieLang &&`. Trade-off: needs to be wired into the consent-decline path, and any other persistence in `globalCookie` (`portalCardSize`, etc.) needs the same treatment if the broader consent-revocation contract requires erasure. Doesn't help users who had no cookie deletion at the consent-decline moment (e.g. pre-consent-system cookies still carry `lang`); a one-off migration would be needed.

- **Code loci.** Both ends are short edits:
  - Option 1: remove `&& isPreferenceConsentGranted()` from `src/components/popups/dialogs/PortalConfigDialog.tsx:89`, drop the now-unused `isPreferenceConsentGranted` import. ~2 lines.
  - Option 2: add an `og_consent` parse + predicate term to `proxy.ts:48-65`. ~5-10 lines including the parser.
  - Option 3: delete `proxy.ts:48-65` plus the lang write site at `PortalConfigDialog.tsx:89-91` plus the `lang` field on the `GlobalCookie` type plus all readers (`readGlobalCookieForSsr`, etc.). Largest edit.
  - Option 4: hook into `useConsentStore.setConsent` when transitioning to denied, or add an effect in the consent banner. Medium edit, plus the migration question.

- **Naming caveat.** I called this an interaction between "consent-gated cookie write" and "consent-unaware cookie-wins redirect." The cookie-wins redirect predates Brief 3j and was likely a deliberate choice in commit `51df7e9`. I am not claiming it was wrong — only that the two halves disagree on whether consent gates the lang-persistence pathway, and that disagreement is the bug.

- **Adjacent observation (Part 4b)**: `useCardSizeStore.setSize` (`src/lib/store/useCardSize.ts:37`) is consent-gated for the cookie write, and `SyncCardSizeFromCookie` (`src/components/client/SyncCardSizeFromCookie.tsx:12-25`) is gated on the `preferenceGranted` selector before doing anything. So card-size persistence is consistent within the consent-aware pattern. The `lang` write site is the only persistence pathway that shares its consent gate with a downstream non-consent-aware consumer (`proxy.ts` cookie-wins). Worth flagging as a single-instance asymmetry rather than a systemic one.

- **Adjacent observation (Part 4b)**: `proxy.ts:50` calls `JSON.parse(decodeURIComponent(raw))` and wraps it in `try/catch` (`proxy.ts:62`). That's correct defensive code for a request-cookie parse — out-of-process input. Mentioning so any fix brief that adds a second cookie parse for `og_consent` knows to follow the same shape.

- **No drafted config-file text.** The investigation's purpose is to surface the cause; the resulting fix's config-file impact will be drafted in the fix session, not here.
