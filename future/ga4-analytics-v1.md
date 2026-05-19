# Brief — GA4 discovery audit

**Repo:** `oglasino-web`
**Branch:** stay on whatever branch is currently checked out — this is read-only
**Session slug:** `ga4-discovery` — `<n>` is `1` if no prior `*-ga4-discovery-*.md` exists in your `.agent/`, otherwise increment per conventions Part 5
**Mode:** read-only. No code changes. No commits. No staged edits.

---

## Framing

This is a discovery audit for Google Analytics 4 instrumentation. Like the SEO discovery audits before it, this is not a Phase 2 feature audit — analytics is a cross-cutting capability, not a feature with a single contract. There is no `features/<slug>.md` spec coming after this directly; the output feeds a feature spec drafted in Mastermind that defines what to build. Treat the structural rules of an audit as binding (read-only, code-as-ground-truth, written output) but the output shape is a findings list, not a feature inventory.

Scope is **web only**. Mobile (`oglasino-expo`) gets its own analytics conversation in a future Mastermind chat.

---

## Task

Audit `oglasino-web` for the current state of analytics instrumentation, consent infrastructure, and the surfaces where the planned v1 GA4 events would need to fire. Code is ground truth. Do not read prior session summaries about analytics. Do not read `oglasino-docs/issues.md`, `oglasino-docs/decisions.md`, or `oglasino-docs/state.md`.

---

## Decisions already locked

So the audit can speak to specifics rather than abstractions:

- **GA4 only.** Not Mixpanel, not PostHog, not Segment.
- **Web only in v1.** No mobile considerations.
- **Client-side via `gtag.js`.** Not server-side, not GTM.
- **Consent Mode v2.** GA fires from page load with consent state; modeled data when users decline.
- **v1 event set, 12 events:** `sign_up`, `login`, `product_view`, `product_create_started`, `product_create_completed`, `contact_seller_clicked`, `message_sent`, `search`, `view_search_results`, `select_promotion`, `exception`, `form_submit_failed`. Plus automatic `page_view`.

The audit doesn't validate these decisions — it inventories what exists and what would have to change to deliver them.

---

## Scope — what to inventory

For each item: what the code does today, where (file + line), and whether it's `present` / `partial` / `absent` / `n/a`. No fixes.

### 1. Existing analytics SDKs and tracking code

Search the repo for any analytics SDK references. Specifically look for:

- `gtag`, `ga(`, `dataLayer`, `googletagmanager`, `google-analytics.com`, `analytics.google.com` — GA4 / GA / GTM
- `mixpanel`, `posthog`, `amplitude`, `segment`, `heap` — competing tools
- Any `<script>` tag injection in `app/layout.tsx`, `app/[locale]/layout.tsx`, `src/components/server/` for tracking pixels
- Any `fetch` to analytics endpoints
- Any `next/script` components loading external trackers
- `next.config.ts` for any analytics-related script configuration

Report: what exists, where, what state it's in (active / commented out / referenced but unused).

### 2. The cookie consent banner

`state.md` references a "Manage cookie preferences" footer link as a pre-launch action item, and the Privacy Policy mentions cookies. The audit needs to find what's actually implemented:

- Is there a cookie banner component? Where?
- What does it cover — strictly necessary cookies only, or categories (analytics, marketing, preferences)?
- Does it write a consent decision anywhere readable by other code (a cookie, localStorage, a store)?
- Is there a "Manage cookie preferences" UI surface that lets users change their decision after the fact?
- Does the banner integrate with anything (Cookiebot, OneTrust, custom)?
- What language coverage exists (all four languages or only Serbian)?

The Consent Mode v2 implementation depends entirely on this. If a banner exists, GA4 reads from it. If it doesn't, the foundation has to be built first.

### 3. Identity — who is the user

GA4 event firing benefits from knowing a stable user identifier (the `user_id` parameter) when the user is signed in. Inventory what's available:

- Where is the authenticated user available in client components? (`useAuthStore`, per the conventions and prior audits.)
- Is there a stable, non-PII user identifier available — the backend's `userId`, the Firebase UID, a hashed email? (PII identifiers like raw email cannot be sent to GA4.)
- For unauthenticated users, what identifier (if any) persists across sessions?
- Where in the layout tree is `useAuthStore` populated, and at what point in the render is a stable user identifier available?

Per the SEO audits, `useAuthResolved` is the existing hook gating auth-dependent rendering. Note how GA4 should hook into this resolution timing — events fired before the user is resolved would be missing `user_id`.

### 4. Page-view firing surface — App Router + locales

Next.js App Router has specific patterns for client-side route changes that don't trigger a full page load. The audit needs to inventory:

- How are route changes handled? Pure server-rendered navigation, client-side via next-intl `<Link>`, or both?
- Is there a global client component at `app/layout.tsx` or `app/[locale]/layout.tsx` that could host a route-change listener?
- What `usePathname` / `useSearchParams` usage exists today? Any hooks already listening to route changes?
- What does a "page view" mean for the catalog page with searchParams (filters)? — does a filter change count as a new page view or the same one? (Architectural decision, but the audit needs to know how the catalog routes today.)

### 5. Event-firing surfaces, per the 12-event set

For each of the v1 events, identify the file(s) and line(s) where the firing call would naturally live. The audit isn't writing code — it's identifying where the instrumentation hooks land.

- **`sign_up`** — where does registration completion fire today? Backend success response handler, redirect to home, or some other completion signal?
- **`login`** — where does Firebase auth signal a successful login? Hook into `useAuthStore`'s set-user flow, or directly into the Firebase callback?
- **`product_view`** — product detail page's server component render or client component mount?
- **`product_create_started`** — entry to step 1 of the create-product wizard. File path for the wizard's first step.
- **`product_create_completed`** — successful submission of step 4 of the wizard. File path.
- **`contact_seller_clicked`** — the call/message/favorite-then-message buttons on the product page. Inventory all surfaces (per the QA-Preparation work, there are buttons in `ProductFunctions.tsx` plus the messages-page entry buttons).
- **`message_sent`** — the first-message-in-conversation send path. File path. Note: per a prior audit, the chat store's `sendMessage` has a silent text-only-send failure path (issues.md 2026-05-14) — instrumentation should fire on success only.
- **`search`** — the global header search submit. Per the QA-Preparation `global-header-search` work, there's a known issue that Enter-key submission doesn't work today (issues.md 2026-05-16). Inventory the submit handler regardless of that bug.
- **`view_search_results`** — catalog page render when arriving via search vs via category navigation. How to distinguish?
- **`select_promotion`** — promoted-product cards. Inventory what exists — Free Zone surface, "popular products" carousels, anywhere a promoted item is rendered distinctly from organic results.
- **`exception`** — global error boundary. Does one exist? Where do client-side errors land today?
- **`form_submit_failed`** — every form submission path that surfaces server-side validation errors. Inventory the existing error-rendering surfaces.

For each event: file + line + a one-line note on whether the surface is straightforward (a single handler) or fragmented (multiple components, would need a shared utility).

### 6. PII surface analysis

GA4 forbids sending PII (raw email, phone number, raw address, payment data) in event parameters. The v1 events shouldn't carry PII by design, but the audit should flag any event parameter that *would* be tempting to send but would violate the rule. Specifically:

- For `sign_up` — would anything in the registration completion payload be PII?
- For `contact_seller_clicked` — would the seller's identity be PII?
- For `message_sent` — would the message recipient's identity be PII?
- For `product_create_completed` — would any product field be PII? (Note: a product description may legitimately contain a seller's phone number that they've chosen to publish — that's user-published data, not PII in the GA4 sense, but flag for clarity.)

This isn't about catching violations — it's about scoping which parameters can safely accompany each event.

### 7. Performance and SSR considerations

GA4's `gtag.js` is a third-party script. The audit should inventory:

- Where would the GA4 script tag be injected? `app/layout.tsx`'s `<head>` is the natural home.
- What `next/script` strategy fits? `afterInteractive`, `lazyOnload`, or `beforeInteractive`?
- Is there existing third-party script loading to compare against (Firebase SDK, reCAPTCHA, anything else)?
- Per the SEO audit's performance section, the layout tree already has heavy client initialization. Adding GA4 contributes another async script — note any constraints.

### 8. Environment separation

GA4 needs separate measurement IDs for stage and prod, or events from stage pollute production analytics. Inventory:

- How does the repo distinguish stage from prod today? (`process.env.NODE_ENV`, custom env var, Vercel environment, something else?)
- Where would the GA4 measurement ID live as a config value? Env var, hardcoded constant, translation table, backend-served config?
- Are there existing precedents for environment-aware config values in `oglasino-web`?

### 9. Debug / observability infrastructure

GA4 has a Debug View in its admin UI that requires the `debug_mode: true` parameter on events. The audit should note:

- Is there a debug-mode flag in the repo today (any context, not GA-specific)?
- Where would a developer toggle GA4 debug mode locally without polluting prod data?

### 10. Cross-repo seams

GA4 v1 is web-only, but the events touch backend-driven flows. Identify seams:

- `sign_up` and `login` fire after backend acknowledges success. The web layer holds these signals via Firebase auth + `useAuthStore`. Document the flow.
- `product_create_completed` fires after backend returns success on the create-product POST. Document where the success signal lands in web.
- `contact_seller_clicked` happens entirely in web (no backend roundtrip). Document.
- `message_sent` fires after Firestore write succeeds (per the chat-store implementation noted in prior issues). Document.

For each: what backend behavior or response shape the analytics instrumentation depends on. No backend changes are expected — this is documenting the assumption so the feature spec captures it.

---

## Output

Write to **two** files per conventions Part 5:

1. `.agent/yyyy-mm-dd-oglasino-web-ga4-discovery-<n>.md`
2. `.agent/last-session.md` (exact copy)

Use the session summary template from conventions Part 5. Replace the "Implemented" section with **"Findings"** organized by the 10 scope items above. For each finding:

- One-line description
- File path + line(s) where relevant
- Status: `present` / `partial` / `absent` / `n/a`
- Severity for the planned v1 build: `blocking` (v1 cannot ship without addressing this), `friction` (v1 can ship but the gap costs significant engineer time), `nice-to-have` (v1 ships fine; gap is for v2 or beyond)
- One-line note on why

Other template sections:

- **Files touched:** `none` (read-only)
- **Tests:** `n/a`
- **Cleanup performed:** `none needed`
- **Obsoleted by this session:** `nothing`
- **Config-file impact:** `no change` for all four files
- **Part 4a evidence:** all three categories are `nothing` for a read-only audit
- **For Mastermind:** any cross-repo seams (per scope item 10) and any architectural questions the audit surfaces that need a Mastermind decision before the feature spec is drafted

---

## Hard rules

- Read-only. No edits, no commits, no staged changes.
- Do not read prior session summaries. Do not read `oglasino-docs/issues.md`, `oglasino-docs/decisions.md`, or `oglasino-docs/state.md`.
- You may read `oglasino-docs/meta/conventions.md` for stack reference and `oglasino-docs/features/` only for spec contracts on routes you're auditing.
- If a finding requires assuming backend behavior, write the assumption explicitly as a seam in "For Mastermind." Do not fabricate.
- Stay in `oglasino-web/`. No cross-repo file reads except `oglasino-docs/` per above.
- Standard engineer-agent hard rules from conventions Part 3 apply.

---

## Definition of done

The findings document covers all 10 scope items. Every finding has file + line, status, severity, and a one-line note. Cross-repo seams are listed. Session summary template is filled out per conventions Part 5.