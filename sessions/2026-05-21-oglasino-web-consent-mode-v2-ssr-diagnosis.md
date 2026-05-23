# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-21
**Task:** Brief 9-pre — Diagnose the SSR Suspense failure on `/rs-sr` and `/rs-sr/owner/*` (admin works, `/` works). The dev-server log shows `TypeError: controller[kState].transformAlgorithm is not a function` with `digest: '3992848766'` and the client shows "Switched to client rendering". Find the cause; do NOT fix.

## Brief vs reality

Read the brief end-to-end, the consent-mode-v2 spec, the eight prior consent-mode-v2 session summaries in `.agent/`, the consent-audit (`.agent/2026-05-20-oglasino-web-consent-audit-1.md`), and every file the brief named. One substantive finding surfaced *while* working the diagnosis that materially shifts the brief's working hypothesis. I am calling it out here rather than burying it in "For Mastermind" because the brief's prescribed bisect at §11 leans on it.

1. **The "ConsentBanner is common to portal+owner, absent from admin" framing is structurally true at the *import* level but not at the *SSR-render* level.**
   - Brief says (Suspect): "Common mount across `(portal)/layout.tsx` and `owner/layout.tsx`, absent from `admin/layout.tsx`: **`ConsentBanner`**".
   - Code says:
     - `app/[locale]/(portal)/layout.tsx:20` mounts `<ConsentBanner />` directly, not gated by any client guard. During SSR, this client component renders (returns `null` because `open=false`) inside the SSR HTML stream.
     - `app/[locale]/owner/layout.tsx:44` mounts `<ConsentBanner />` **inside `<SessionGuard isAdminRoute={false}>`** (line 28). `SessionGuard` is `'use client'` and returns `<LoadingOverlay />` while its `loading` state is `true` — the initial value. **During SSR, `SessionGuard`'s children — including `<ConsentBanner />` — are not rendered to the SSR HTML at all.** Children are passed as the `children` prop, serialized for client hydration, but the initial-render branch (`if (loading) return <LoadingOverlay />`) does not enter them.
     - `app/[locale]/admin/layout.tsx:28` uses the same `<SessionGuard isAdminRoute={true}>` wrapping pattern. Admin's children also do not render during SSR.
   - Why this matters: if ConsentBanner's *SSR render* were the throw site, owner and admin should behave identically (both gated behind `SessionGuard` → `LoadingOverlay`), but the brief reports owner fails and admin succeeds. So a single uniform explanation that pins everything on ConsentBanner's SSR pass cannot fit owner. Either (a) the cause is the *import / module-graph* effect of ConsentBanner being referenced in owner's tree (the `'use client'` reference entry is created and resolved by the framework even when the parent doesn't render the child), or (b) the cause is something else in the owner layout that is also in the portal layout but not in admin.
   - I have not started any fix; this finding sharpens the bisect at §11.

2. **`readOgConsent`'s SSR guard is intact** (per brief §7 — verify brief 7 didn't strip it). `src/lib/consent/cookie.ts:11-13, 19-31` carry the `hasDocument()` guard verbatim: `function hasDocument(): boolean { return typeof document !== 'undefined'; }` and `if (!hasDocument()) return null;` is the first statement of `readOgConsent`. Brief 7's strip removed `migrateLegacyConsent` and the legacy fallbacks; the SSR guard on `readOgConsent` was preserved.

3. **`readConsentForSsr` still uses the Next.js `cookies()` API** (per brief §8). `src/lib/consent/ssr.ts:1, 8, 42-50` imports `cookies` from `next/headers` (type-only — `import type { cookies } from 'next/headers'`), derives the cookie-store type from the imported function (`type CookieStore = Awaited<ReturnType<typeof cookies>>`), and reads the value with `cookieStore.get(OG_CONSENT_COOKIE_NAME)?.value`. No client-side `getCookie` reference. Brief 7's simplification left the SSR helper sound.

## Implemented

None — this is a diagnostic session. Working tree unchanged from session start. No bisect edits applied (see "Tests" — bisect requires a reproducible failure, which I could not achieve in this environment; see §B below).

## Diagnostic chain — what I observed, step by step

### A. File-level audit (brief §1–§8)

**A1. `src/components/client/consent/ConsentBanner.tsx` (brief §1).** `'use client'` at line 1. All store-imperative calls live inside the single mount-only `useEffect` (lines 37-76): the `useConsentStore.subscribe(...)` registration at line 47, the `useConsentStore.getState()` read at line 66, the conditional `state.hydrate()` call at line 68, the `useConsentStore.getState().clearReopen()` at line 59. **No module-body or component-body top-level call to `document`, `window`, `localStorage`, `cookies`, or `useConsentStore.subscribe` / `getState()` / `hydrate`.** Imports pulled in: `Button`, `Drawer`, `DrawerContent`, `Label`, `Switch`, `applyConsent`, `ConsentData`, `useConsentStore`, `TranslationNamespaceEnum`, `X` (lucide), `useTranslations` (next-intl), `useEffect`, `useState`, and the four `consentDecisions` builders. The component returns `null` (line 95: `if (!open) return null`) on the SSR pass because the initial state is `open=false` — no JSX from `<Drawer>` reaches the SSR HTML.

**A2. `(portal)/layout.tsx` (brief §2 — portal half).** Renders `<RemoveHash />`, `<Header />`, `<PortalMain>{children}</PortalMain>`, `<ToTheTopButton />`, `<MobileFooterNavigation />`, `<ConsentBanner />` as a flat fragment. No `SessionGuard`. ConsentBanner renders during SSR (returns `null`).

**A3. `owner/layout.tsx` (brief §2 — owner half).** Renders `<SessionGuard isAdminRoute={false}>` wrapping `<SidebarProvider>` wrapping `<SelectableFilterManagerWrapper portalScope="dashboard" />`, `<DashboardSidebar />`, `<SidebarInset>{<TopNavigation /><main>{children}</main>}</SidebarInset>`, `<ConsentBanner />`. **The `<ConsentBanner />` sits inside `<SessionGuard>` (line 44).** Per A6 below, this means ConsentBanner does not render during SSR on owner routes.

**A4. `admin/layout.tsx` (brief §3).** Renders `<SessionGuard isAdminRoute={true}>` wrapping `<SelectableFilterManagerWrapper portalScope="admin" />`, two layout `<div>`s, `<SidebarProvider>` wrapping `<AdminSidebar />`, `<SidebarInset>{<TopNavigation /><div>{<ScrollToTop />{children}}</div>}</SidebarInset>`. **No `ConsentBanner` mount anywhere.** Both `generateMetadata` and the render body shape mirror owner's (different translation namespaces, different sidebar component, no portal-side ConsentBanner / DashboardSidebar). Behind the `SessionGuard` gate, the admin tree's children also do not render during SSR — same as owner.

**A5. `app/[locale]/layout.tsx` (brief §4).** Server component. `setRequestLocale(locale)`, then two `await getBaseSiteServer()` / `getAllBaseSitesOverviews()` calls. Renders `<NextIntlClientProvider>`, `<NavigationProgressBar />`, `<AuthInit />`, `<QuickRecommendButton />`, `<BaseSiteInit ... />`, `<TooltipProvider>` wrapping `<AppInit />`, `<Toaster .../>`, `<DrawerDialogManager />`, `{children}`. Calls `notFound()` if locale invalid or `baseSite` is `null`. This layout is identical for `/rs-sr/admin` (works) and `/rs-sr/owner/*` (fails), so it cannot in itself be the SSR-throw site — admin would fail too.

**A6. `app/layout.tsx` (brief §5).** `await headers()`, `await cookies()`, then `sanitizeForSnippet(readConsentForSsr(cookieStore))` and the inline `<script dangerouslySetInnerHTML={{ __html: consentDefaultSnippet }} />` in `<head>`. Brief 2's snippet shape is unchanged. `/` works (the brief says so), confirming this root layout is fine in isolation. The cookie-read path here is sound: `readConsentForSsr` is type-only `import type { cookies }` (no runtime client-side cookie read), and the snippet's four interpolated signals are sanitized to the `'granted' | 'denied'` union before reaching the template string.

**A7. `src/lib/store/useConsentStore.ts` (brief §6).** Module body is **pure**: `import { create }`, `import { readOgConsent, writeOgConsent }`, `import { ConsentData }`, then a single `export const useConsentStore = create<ConsentStore>()(...)` call with an inline factory. **No top-level `subscribe` registration, no module-load `getCookie`, no module-load `hydrate()` call.** The store's `hydrate` action (line 29-32) calls `readOgConsent()` and `set({ consent: existing, hydrated: true })`. `readOgConsent` is SSR-safe (A8). `hydrate` is invoked from exactly two sites: `ConsentBanner.tsx:68` inside a `useEffect` (client only), and `app/[locale]/owner/cookies/page.tsx:21` inside a `useEffect` (client only). The store never fires during SSR.

**A8. `src/lib/consent/cookie.ts` (brief §7).** `hasDocument()` guard intact at line 11-13. `readOgConsent` (line 19) early-returns `null` if `!hasDocument()` (line 20). `writeOgConsent` (line 33) early-returns if `!hasDocument()` (line 34). `deleteOgConsent` same shape. **Brief 7 did not strip the SSR guard.** The `console.warn` on the non-Secure dev branch (line 40-42) is module-scoped only via the `warnedAboutInsecure` latch — does not fire during SSR because `writeOgConsent` itself early-returns before reaching the latch on the server.

**A9. `src/lib/consent/ssr.ts` (brief §8).** Type-only `import type { cookies } from 'next/headers'` (line 1). `readConsentForSsr(cookieStore)` (line 42) takes a `CookieStore` parameter (the awaited `cookies()` return shape — `type CookieStore = Awaited<ReturnType<typeof cookies>>`). Function body reads `cookieStore.get(OG_CONSENT_COOKIE_NAME)?.value`, `safeJsonParse`s, validates via `isValidConsent`, falls back to `DENIED_DEFAULTS`. **Cannot throw** — every code path returns a `ConsentData`. `sanitizeForSnippet` (line 24) walks the four signals against the `'granted' | 'denied'` union, logs `console.error` and returns `DENIED_DEFAULTS` on off-union values. **Cannot throw** for the same reason.

**A10. Transitive import — `vaul` (brief §1, "Any import that pulls in a UI primitive (Drawer, Dialog, Portal). Note the import paths").** `src/components/shadcn/ui/drawer.tsx:4` imports `Drawer as DrawerPrimitive` from `vaul` (v1.1.2 — `node_modules/vaul/package.json`). Audit of `node_modules/vaul/dist/index.mjs` for top-level browser globals:
- `__insertCSS` (line 2) is guarded by `if (!code || typeof document == 'undefined') return` (line 3).
- `const visualViewport = typeof document !== 'undefined' && window.visualViewport;` (line 103) is guarded.
- `navigator.userAgent` references (lines 65 et al) all sit inside function bodies, not at module scope.
- All `document.documentElement`, `window.pageXOffset`, `window.getComputedStyle` references sit inside function bodies.

**Vaul's module-level evaluation is SSR-safe.** Importing `Drawer` from `vaul` during SSR does not throw.

### B. Reproduction (brief §9 + §10)

Ran the dev server fresh with `NODE_OPTIONS='--stack-trace-limit=200' npm run dev`. Next.js 16.2.6 + Turbopack, port 3000. Node v22.22.2. `Ready in 279ms`.

Probed the four route shapes the brief named, three times each (warmup + two more) to catch any HMR-state-dependent failure:

| Route | Status | Bytes | Time | Repeat | Notes |
| --- | --- | --- | --- | --- | --- |
| `/` | 200 | 32,934 | 0.28s | clean | no errors logged |
| `/rs-sr` | 200 | 1,690,479 | 1.86s → 0.39s → 0.39s | clean | three hits, no errors |
| `/rs-sr/owner/user` | 200 | 1,543,043 | 0.56s → 0.17s | clean | no errors |
| `/rs-sr/admin` | 200 | 1,539,844 | 0.53s | clean | as expected |
| `/rs-sr/about` | 200 | 1,680,534 | 0.64s | clean | portal/public sibling |
| `/rs-sr/wants` | 200 | 1,554,746 | 0.52s | clean | locale-protected portal route |

Then with `Cookie: globalCookie=…cookieConsent…` (a stale legacy globalCookie payload that brief 7 stripped support for):

| Route | Status | Bytes | Time | Notes |
| --- | --- | --- | --- | --- |
| `/rs-sr` | 200 | 1,690,479 | 4.10s | unchanged; stale legacy cookie does not destabilize SSR |

Then with `RSC: 1` header (simulating the App Router client-navigation prefetch shape):

| Route | Status | Bytes | Time | Notes |
| --- | --- | --- | --- | --- |
| `/rs-sr` | 200 | 1,459,757 | 121s | slow (Turbopack first-touch on the RSC variant) but successful, no `transformAlgorithm` in the server log |

Grep on the rendered HTML for known SSR-streaming-failure markers (`"errorDigest"`, `"digest"`, `formatProdErrorMessage`, `"\u`, `transformAlgorithm`, `kState`, error boundary digests) returned **zero hits** on every captured body. The body contains the expected RSC instruction stream (`self.__next_f.push`), the consent snippet (`gtag('consent', 'default', { ..., analytics_storage: 'denied', wait_for_update: 500 })` and `window.__og_consent_loaded = true`), one `ConsentBanner` client-reference marker, and the rendered portal/public page tree. The `digest: '3992848766'` from Igor's log does not appear.

**I cannot reproduce Igor's failure in this environment.** Dev server is fresh (no HMR state), no backend running (some routes have `MISSING_MESSAGE` fallbacks for keys that depend on the backend translation seeds, plus `connection.timeout` for product/backend calls — both pre-existing and noted in prior session probes; neither produces a `transformAlgorithm` error).

### C. Why my reproduction does not match Igor's — three plausible explanations

The error message Igor sees — `TypeError: controller[kState].transformAlgorithm is not a function` — is from Node's internal WHATWG TransformStream, surfaced when React/Next's streaming SSR pipeline tries to abort a stream that has already transitioned out of its valid state. The error itself is the *abort cleanup*, not the underlying SSR throw. The `ignore-listed frames` text Igor reports tells us Next.js's framework boundary is masking the real frame. So:

- **C1.** The underlying throw must be inside an SSR-rendered tree that includes portal-vs-owner-vs-admin difference. Per A2/A3/A4 above, the only tree that actually *renders* during portal SSR (and includes ConsentBanner) is `(portal)/layout.tsx` — owner's children are gated behind `SessionGuard`'s `LoadingOverlay` and admin's children likewise. So `/rs-sr` and `/rs-sr/owner/*` failing with the same digest is suspicious: if they had the same underlying throw, it would have to be in a layout *above* the `SessionGuard` for owner — i.e., `(portal)/layout.tsx` is not in owner's tree; both routes share `app/layout.tsx`, `app/[locale]/layout.tsx`, and the route-group-specific layout. For owner the route-group-specific layout *is* `owner/layout.tsx` — whose body, *up until* the `SessionGuard` gate, is what renders. That body is `<SessionGuard ...><SidebarProvider>...<ConsentBanner /></SidebarProvider></SessionGuard>` — the JSX itself contains the `<ConsentBanner />` reference even though it doesn't render during SSR. **Module resolution for ConsentBanner happens at compile/bundle time, not at SSR render time** — so importing ConsentBanner is not the cause of an SSR-render-time throw.

- **C2.** The two routes (`/rs-sr` and `/rs-sr/owner/*`) may be failing for *separate* underlying reasons that both surface as the same `transformAlgorithm` cleanup error, because the cleanup error is the symptom, not the cause. `/rs-sr` has the most surface area to throw inside SSR (full portal/public tree, backend product-fetch, Header, Footer including `<ManageCookiesFooterButton />`, the JSON.parse on `globalCookie` cookie at `(portal)/(public)/page.tsx:43`). `/rs-sr/owner/*` only renders its `SessionGuard → LoadingOverlay` shell during SSR, but the layout *also* includes `generateMetadata` work (`getTranslations + getBaseSiteServer`) and the `<SelectableFilterManagerWrapper>` / `<DashboardSidebar>` / `<TopNavigation>` references — any of those could be the throw site.

- **C3.** Environment difference. The most concrete branch is that Igor's dev server has been running through edits (briefs 1-7 + 8b + B all landed sequentially) and accumulated Turbopack / HMR state from each. The `transformAlgorithm` error class is *specifically* documented to surface when a streaming React renderer's abort path interacts with a stale module reference — a known footgun when a `'use client'` module is hot-replaced while a server-streaming render is in flight. A fresh dev server (mine, this session) starts clean. A dev server with several hot-reloads of `ConsentBanner.tsx` (and the surrounding tree) under its belt could be carrying a corrupted client-reference. The "constantly reloads" symptom Igor reports matches this branch: a `Switched to client rendering` recovery + dev-server HMR ping → browser reload → SSR fails again → recovery again → reload → loop.

C1+C2+C3 are not mutually exclusive. C3 explains the env divergence; C1 explains why the "ConsentBanner is the SSR-render-time culprit" framing doesn't quite fit owner; C2 leaves open that the actual throw may not even be in ConsentBanner code.

### D. Bisect prerequisites (brief §11)

The brief's prescribed bisect — comment out `<ConsentBanner />` in `(portal)/layout.tsx` and re-test — requires a *reproducible failure to bisect against*. I do not have one in this environment (B above). Two options to make the bisect viable:

1. **Igor reruns the diagnostic on his machine after first deleting `.next/` and restarting the dev server.** This forces a fresh Turbopack cache and a clean module graph. If the failure goes away after that, branch C3 is confirmed and no code fix is needed — only a runbook note. If the failure persists, run the bisect against the reproducer.
2. **If C3 turns out not to be the cause, the bisect itself is one-shot:** comment out `<ConsentBanner />` in `(portal)/layout.tsx:20` *and* in `owner/layout.tsx:44`, hard-reload `/rs-sr` and `/rs-sr/owner/user`. Branches:
   - Both clear → bug is in ConsentBanner's module-graph (one of its transitive client imports, most likely vaul under a specific dev-server state; A10 above rules out module-level browser-globals as a cold-import throw, but doesn't rule out a hot-replace / cache-stale interaction).
   - Only `/rs-sr` clears → bug is split between portal (ConsentBanner) and owner (something else in `owner/layout.tsx`'s pre-`SessionGuard` body — most likely the `generateMetadata` chain, or a transitive `'use client'` module inside `<SidebarProvider>` / `<DashboardSidebar>` / `<TopNavigation>` / `<SelectableFilterManagerWrapper>`).
   - Neither clears → ConsentBanner is not the cause; bisect upward (next: comment the consent snippet emission in `app/layout.tsx:58` and re-test).

## Files touched

- none. Diagnostic-only session per brief hard rules. Working tree restored — no bisect edits applied because there was no reproducible failure to bisect against in this environment (see §B and §D).

## Tests

- Did NOT run `tsc --noEmit` / `npm test` / `npm run lint` — diagnostic-only session, no code change. Brief 9 (the docs close-out) and the eventual fix brief will exercise the gate.
- Ran the dev server (`NODE_OPTIONS='--stack-trace-limit=200' npm run dev`) and curl-probed eight route shapes (six routes, two with extra header variants). All returned HTTP 200 with no `transformAlgorithm` entry in the dev-server log. Stopped the dev server before writing the summary.

## Cleanup performed

- none needed. No code touched. The temporary curl output files (`/tmp/curl-*.html`, `/tmp/devlog.txt`, `/tmp/balance.html`, `/tmp/rsc-rssr.html`) are in `/tmp` and will be reaped by the OS. No working-tree change.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change (this session's findings are diagnostic, not bug-tracking — they belong in the next-brief input, not in `issues.md`)

**Closure gate confirmed** — no implicit config-file dependency. No drafted config-file text. The fix brief Mastermind will draft next is a working-tree code change; if its outcome requires a Risk Watch entry on `state.md` (e.g., "consent feature briefly broke /rs-sr SSR on dev, root-caused to X") that's the fix brief's call.

## Obsoleted by this session

- nothing. Diagnostic-only. No code obsoleted.

## Conventions check

- Part 4 (cleanliness): N/A — no code edited.
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): two observations flagged in "For Mastermind." Both low severity, both pre-existing or hypothesis-grade.
- Part 6 (translations): N/A — no translation surface touched.
- Other parts touched:
  - Part 3 (no cross-repo edits): confirmed. No file outside `oglasino-web` read for write.
  - Part 11 (trust boundaries): N/A — no auth/moderation/state-transition surface touched.
  - Brief's own hard rules: confirmed.
    - Branch is `dev`. No commits. ✓
    - No fixes applied. Diagnostic only. ✓
    - No edits outside `oglasino-web`. ✓
    - No deletions, no new files. ✓
    - Working tree restored (untouched). ✓

## Known gaps / TODOs

- **Reproduction.** The dev-server `transformAlgorithm` error does not surface in this environment. I cannot complete brief §11's bisect without a reproducer. Branch C3 (Turbopack/HMR-cache state on Igor's running dev server) is my leading hypothesis for the env divergence; rm-`.next` + fresh restart on Igor's machine is the cheapest first test.
- **Underlying throw not identified.** Per C1/C2, the `transformAlgorithm` error is the abort cleanup, not the root throw. The root throw's frame is hidden by Next.js's `ignore-listed frames` framework filter. The brief mentioned `NODE_OPTIONS="--stack-trace-limit=100"` — I ran with `200` but with no error in my reproduction to attach the stack to, the technique is moot for this session. On Igor's reproducing machine, the same flag may surface a non-framework frame.
- **ConsentBanner-is-the-culprit confidence.** Lowered from "strong working hypothesis" (the brief's framing) to "structurally circumstantial, but inconsistent with the owner-vs-admin symmetry under `SessionGuard`." See §A3/A4 + C1. A bisect (per §D) is the only thing that can move this from circumstantial to confirmed.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - **Added (earned complexity):** nothing. Diagnostic-only session, zero code added.
  - **Considered and rejected:**
    - **Running the bisect (commenting out `<ConsentBanner />` in `(portal)/layout.tsx`) anyway, despite no reproducer.** Rejected because (a) the brief's hard rule "If you uncomment something to bisect, restore the working tree before writing the session summary" treats bisect as a means to a diagnostic end, not an end in itself, and (b) without a reproducible failure, a passing `/rs-sr` after the comment is uninformative (it was already passing for me before the comment), so the bisect cannot generate signal. The bisect remains the right move once a reproducer exists — see §D for the prescription Igor should run on his machine.
    - **Adding a small `console.log` at the top of `ConsentBanner` and re-probing.** Rejected — no reproducer means no log to read. Same logic as above.
    - **Drafting the fix brief speculatively assuming ConsentBanner is the cause.** Rejected per the brief's "If you find yourself wanting to 'just fix it quickly' — stop. Document the cause and stop. Mastermind needs to draft the fix brief." The brief explicitly forbids speculative fix-brief drafting; "Proposed fix shape sketched in 'For Mastermind' if confident. Otherwise note 'fix requires Mastermind input on whether to gate / restructure / re-add legacy guard.'" I am in the "otherwise" branch — see "Proposed next-step" below.
  - **Simplified or removed:** nothing.

- **Proposed next-step (informational, not a fix brief):**

  Mastermind should have Igor run the following sequence on his reproducing machine, in order. Each step is independent; stop at the first one that clears the failure.

  1. **`rm -rf .next` + restart dev server.** Forces a clean Turbopack cache. If `/rs-sr` clears, branch C3 is confirmed: this was Turbopack/HMR-state corruption from edits-while-running, not a real code defect. The "fix" is a runbook entry, not a code change.
  2. **`rm -rf .next node_modules/.cache && npm install && npm run dev`.** Wider cache clean. Same signal interpretation.
  3. **Comment out `<ConsentBanner />` in `(portal)/layout.tsx:20`.** Reload `/rs-sr`. If it clears, the cause is ConsentBanner or one of its imports under the specific dev-state Igor is in. The first thing to try then is replacing `Drawer`/`DrawerContent` with an inline conditional render (still returns null until open) to isolate vaul. If `/rs-sr` still fails with ConsentBanner commented, the cause is elsewhere in the portal layout — `Header`, `PortalMain`, `MobileFooterNavigation`, or the `(portal)/(public)/page.tsx` body (the backend product-fetch and the `JSON.parse(cookieValue)` on a potentially malformed legacy `globalCookie`).
  4. **If §3 narrows to "elsewhere in the portal layout," repeat the comment-out on `(portal)/(public)/page.tsx`'s `JSON.parse(cookieValue)` line** — if Igor's browser still has a stale `globalCookie` with the `cookieConsent` field (which brief 7 stripped support for but didn't expire the cookie itself), and that value's shape now diverges in a way that triggers the parse path's downstream consumer, the throw would be `(public)/page.tsx:43`. This is speculative but cheap to check.
  5. **For `/rs-sr/owner/*`, comment out `<ConsentBanner />` in `owner/layout.tsx:44` and `<SelectableFilterManagerWrapper>` at line 30.** Same shape-of-bisect; the owner SSR tree above `SessionGuard` is thin, so a few targeted comments triangulate quickly.

  Capture the dev-server log at each step. The first step that produces a non-`transformAlgorithm` line (a real stack trace with a non-ignore-listed frame) identifies the throw site. That's the fix brief's input.

  Once the throw site is identified, the fix shape is:
  - **If C3 (cache state):** no code change. Document the recovery runbook in a fix brief that drafts a `state.md` Risk Watch entry on dev-server cache hygiene for the consent feature's HMR window. Resolve and remove the entry once the feature ships.
  - **If ConsentBanner-internal:** the most plausible candidates are (a) the `useConsentStore.subscribe` registration inside `useEffect` (move to `useSyncExternalStore` for SSR symmetry), or (b) the conditional `state.hydrate()` synchronous call inside `useEffect` (defer to the next tick). Both are minor. But this is only worth doing once the bisect points there.
  - **If portal-layout-elsewhere:** the fix shape is whatever the specific throw demands (e.g., guarding `JSON.parse` against malformed legacy `globalCookie`).

- **Adjacent observations (Part 4b) flagged for Mastermind triage:**

  - **`(portal)/(public)/page.tsx:42-44` parses `globalCookie` without a try/catch.** Severity: low–medium. If a user has a stale `globalCookie` cookie whose value is non-JSON (a corruption edge case), `JSON.parse` throws and the SSR pass aborts. Brief 7 stripped the *type* field `cookieConsent` from `GlobalCookie` but cookies in users' browsers may still carry the legacy field — that case is still valid JSON, so this concern is only on the actually-malformed branch, not the legacy-shape branch. Not introduced by any consent-mode-v2 brief; pre-existing. If the bisect (above, step 4) implicates this site, the fix is a try/catch + `{ portalCardSize: 'small' }` fallback. Out of scope for the diagnostic brief.

  - **`useConsentStore.subscribe` registration inside `ConsentBanner`'s `useEffect` (`ConsentBanner.tsx:47`) is the correct shape per React 19's external-state pattern but is fragile under HMR.** Severity: low. The pattern is documented in the file's own comment (lines 33-36 explain the choice). Under a fresh dev server it's fine. Under HMR that hot-replaces ConsentBanner.tsx while a server-streaming render is in flight, the subscribe callback may hold a closure over the previous render's `setState` setters, which on the next dispatch would try to update a stale React fiber. This is well-trodden React-DevTools-warning territory, not a defect. Migrating to `useSyncExternalStore` would make the pattern HMR-robust but is feature-sized refactor work, not bug-fix scope. Flagging in case the bisect lands here.

- **No drafted text for `conventions.md`, `decisions.md`, `state.md`, or `issues.md`.** Closure gate satisfied. Diagnostic-only session.
