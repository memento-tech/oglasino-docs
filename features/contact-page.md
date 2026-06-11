# Feature — Contact / Support Page

**Status:** Implementation-complete across backend + web + mobile (code-complete on `dev`, 2026-06-11) — pending Igor's end-to-end live verification
**Owner:** Igor (Mastermind orchestration)
**Driver:** iOS App Store review requires an in-app support/contact surface.
**Repos touched:** oglasino-backend, oglasino-web, oglasino-expo
**Date:** 2026-06-11

---

## 1. Goal

Ship a Contact/Support page on web and mobile. Logged-in and anonymous users can reach support. Web carries a real submit form; mobile surfaces the support addresses as tappable + selectable text (no form). Footer links to the page on both platforms. The page visibly shows support@oglasino.com and privacy@oglasino.com.

## 2. Scope / non-goals

IN:

- Web: `/contact` page (content shell + a submit form) — composes a client form component, styled consistently with the existing about/pricing pages.
- Web: `POST /api/public/contact` endpoint that emails the support inbox.
- Mobile: `/contact` screen showing the two support addresses as tappable mailto links AND as selectable plain text. No form, no backend call.
- Footer link to `/contact` on both platforms.

OUT (v1):

- No DB persistence of contact requests (email-only; see Decision 1).
- No topic/category routing in the form (always support@; see Decision 2).
- No mobile submit form (mailto only; see Decision 3).
- No new translation namespace (reuse existing; see Decision 6).
- No content-moderation rules on the message beyond structural length checks.

## 3. Locked decisions (with rejected alternatives)

### D1 — Destination: email only, no DB table

Submitted form requests are emailed to the monitored support inbox. No `contact_request` table in v1.

- Rejected — reuse `suggestion` table: wrong shape (varchar(255), no email column, no subject, nullable userId; would pollute the admin moderation surface with support PII). Backend audit decisive.
- Rejected — dedicated `contact_request` table: premature. The email stack covers v1 with no new primitive. A persisted record can be added later without rework (reuse the suggestion server-derived-userId pattern then).

### D2 — privacy@ is display-only; form always sends to support@

The form has a single destination: the support inbox. privacy@ is shown in the page copy as a visible/tappable address for users who want the privacy channel directly; the form does not route to it.

- Rejected — topic dropdown routing to different inboxes: added form complexity; privacy@ has no backend config today. Clean later addition if wanted.

### D3 — Mobile: mailto + selectable text, no form

Mobile shows support@ and privacy@ as tappable `Linking.openURL('mailto:…')` rows AND as selectable plain text. No form, no backend dependency.

- Rationale: clears Apple's "reachable support" bar with the native pattern; zero backend dependency; zero new namespace.
- Hard requirement: addresses MUST render as selectable text, not be reachable only behind the tap. `mailto:` opens a composer only if a mail client is configured; on a bare simulator (no Mail account) `Linking.openURL('mailto:…')` can reject. The text fallback is the correct degradation.
- Rejected — full mobile form: with email-only destination (D1) there is nothing to track that web doesn't already cover; strictly more work for a screen reviewers glance at once.

### D4 — Web form error model: per-field, page-shaped

The web form surfaces per-field errors (email vs message fail independently). Reuse the generic `{field,code,translationKey}` parsing (`parseProductValidationErrors` — generic despite the name). Page-level shape follows `forgot-password/page.tsx` (public-page form + status line), NOT a dialog.

- Rejected — ReportDialog single-error model: collapses two independent fields into one message area; worse UX here.

### D5 — Footer placement: company/portal column

Web: add to the company/portal column (`companyNavigations.tsx`), positioned AFTER `terms.label` and BEFORE the manage-cookies button (consent button stays last). Mobile: matching placement — add to the company column for parity.

- Rationale: page-family consistency with about/pricing/privacy/terms; keeps web/mobile parallel.
- (Mobile note: `helpNavigations.tsx` was the alternative home; company column chosen for cross-platform parity.)

### D6 — Translations: reuse existing namespaces, no new namespace

No CONTACT_PAGE / SUPPORT_PAGE namespace. Strings split across existing, already-fetched namespaces:

- Page heading + body copy → COMMON
- Footer link label (`contact.label`) → PAGING
- Button label(s) → BUTTONS
- Validation / error display → VALIDATION / ERRORS
- Rationale: mobile cannot invent a namespace; this keeps web and mobile drawing from identical keys and avoids a backend enum addition.
- Rejected — CONTACT_PAGE namespace: web-parity argument is real but forces a backend enum + seed and breaks web/mobile key symmetry for no rendering benefit on a small page.

### D7 — Styling: consistent with existing pages

Web: mirror the about/pricing page structure and visual treatment — server `(public)` page with `generateMetadata`, composing a `'use client'` form component. No bespoke styling; match the existing content-page look.

Mobile: clone the privacy/terms screen shell (ScrollView → BackToHomeButton → body → Footer). Match existing content-screen styling.

## 4. Trust boundary (non-negotiable)

The submit endpoint serves BOTH authenticated and anonymous callers. Identity is derived from the SERVER's view of authentication, never from the request body.

- Request carries a valid token (`getCurrentUserId()` present): attribute to that userId; derive the user's email server-side from the DB (userId → User.email). IGNORE any client-supplied email on this path. Do not echo a client email into reply-to/from for a logged-in user — that would let them spoof attribution.
- No token (`getCurrentUserId()` empty): genuinely anonymous. The form email is the only source; use it as reply-to/contact CONTENT (not identity). Validate format (`@Email` + `@NotBlank`). No account exists to spoof.
- The mode switch is the presence/absence of a server-verified principal ONLY. Do NOT accept a client `anonymous` flag or a client `userId`. Mirrors the existing `/api/public/suggestion` endpoint, which never reads identity from the body.

Email of a logged-in user is a DB read (User.email via userId), NOT an auth-context field — OglasinoAuthentication and AuthenticatedUserDTO do not carry email.

## 5. Backend specification (oglasino-backend)

Endpoint:

- `POST /api/public/contact` (permitAll; FirebaseAuthFilter opportunistically populates the principal — no extra wiring, same as `/api/public/suggestion`).

Request DTO:

- New ContactRequestDTO: `email` (`@Email` `@NotBlank`), `message` (`@NotBlank` `@Size(min=50, max=2000)` — see §8). The message minimum is now SERVER-enforced (it was client-side only before); a too-short message yields a distinct code `CONTACT_MESSAGE_TOO_SHORT` → `validation.contact.message.too_short` (VALIDATION). Other contact codes/keys unchanged.
- HARD RULE: the controller MUST annotate the body `@Valid`. The existing SuggestionController omits `@Valid` (SuggestionController.java:24), so its DTO constraints are dead and a malformed body 500s instead of 400s. Do NOT copy that omission. This is a spec invariant, not a preference.

Handler logic:

- Resolve identity: `currentUserService.getCurrentUserId()`.
  - Present → load User, use `User.getEmail()` as the sender/reply-to; ignore DTO.email.
  - Empty → use DTO.email as the sender/reply-to (validated content).
- Build a branded support email and send to the support inbox.

Email send:

- Reuse the existing stack: assembler modeled on `TransactionalEmailSender` (service/impl/TransactionalEmailSender.java) → inner HTML + plain-text fallback → `EmailLayout.wrap(...)` → `EmailService.sendHtml(to, subject, html, text)`.
- Recipient (`to:`): the existing `oglasino.email.reply-to` value — already `support@oglasino.com` in every env via `OGLASINO_EMAIL_REPLY_TO`. No new property or env var added in v1. The contact send destination and the outbound Reply-To header are the same support inbox, so they reuse one config (now live — see Brief 2).
- Subject + body copy localized via TranslationService backend translations, same as the transactional senders.

Abuse controls (BOTH layers, they stack):

1. URL/IP bucket: add `/api/public/contact` to RateLimitFilter.categorize(). Prefer a dedicated tighter category over PUBLIC_WRITE (20/min) — an anonymous contact form is a spam target and the IP-keyed fallback is the only key for anonymous callers. Spec a tighter cap (see §8).
2. Per-email cooldown: copy the AuthController 60s + 5/day Redis pattern (resendVerification / requestPasswordReset). Key on the effective email (DB-derived when logged in, form email when anonymous). On a real SMTP send failure, release both limits so a transient error doesn't lock the user out (existing pattern already does this).

Duplicate suppression (courtesy guard — distinct from the abuse controls above):

- The endpoint rejects an EXACT duplicate — the same lowercased effective email (DB-derived when logged in, form email when anonymous) AND character-for-character identical message text — re-submitted within a short Redis TTL window. "Exact" means identical text; a single changed character is a new message. Returns a 4xx (business-rule 422 per the error contract below) with code `CONTACT_DUPLICATE` → `contact.duplicate` (ERRORS), which the web surfaces as a "you've already sent this message" notice. This guards against accidental double-sends; the per-email cooldown and daily cap remain the actual spam controls.

Error contract:

- `{errors:[{field,code,translationKey}]}` envelope (GlobalExceptionHandler). 400 for Jakarta constraint violations (requires `@Valid`), 422 for any business rule, 429 from RateLimitFilter / the per-email cooldown.

Translation seed (Backend owns the SQL seed):

- Keys: PAGING `contact.label`; COMMON page heading + body keys; BUTTONS submit label; VALIDATION `validation.contact.message.too_short`; ERRORS `contact.duplicate`. (Other exact key names enumerated in the engineering brief; namespaces fixed per D6.)
- ALL LOCALES, not EN only. Every contact key MUST be seeded in every locale the translation system supports — seeding EN alone would leave the SR/CNR/RU portals rendering raw keys or English fallbacks on a launch-blocking page.
- Locale coverage rule: match the EXACT locale set of the existing sibling keys. `contact.label` mirrors the coverage of `about.label` / `terms.label` / `pricing.label` in PAGING; the COMMON/BUTTONS contact keys mirror the coverage of existing keys in those namespaces. Read the current seed to get the canonical locale list and reproduce it 1:1 — do not invent a locale the existing keys lack, and do not omit one they carry. (Do not hardcode a locale list from memory; derive it from the existing seed.)
- Serbian and Montenegrin copy uses MASCULINE grammatical forms (standing operator preference).
- The string VALUES (the actual translations) are drafted as an all-locale translation table at backend-engineering-brief time and reviewed before seeding — they are not invented by the seeding engineer mid-implementation.

## 6. Web specification (oglasino-web)

Page:

- New `/contact` route: `app/[locale]/(portal)/(public)/contact/page.tsx`, server component on the about/pricing shell (D7). Exports `generateMetadata` delegating to a NEW `src/metadata/generateContactMetadata.ts` modeled on `generateAboutMetadata.ts`. (public) group → openable cold/unauthenticated.

Form component (`'use client'`):

- Anti-flash gate: `useAuthResolved()` — render the email field (or form) placeholder until resolved === true, so a logged-in user never flashes empty/unlocked then fills.
- Auth + email: `useAuthStore((s) => s.user)`; `user?.email`.
- Prepopulate-and-lock: when `user` non-null, seed email `value` from `user.email` and pass `disabled` to the existing `Input` (src/components/server/Input.tsx supports disabled + controlled value). Anonymous → empty + editable.
- Client validation: NEW `contactSchema` (Zod, structural only) modeled on productSchemas.ts style (named constants + z.object). email `.email().max(254)`, message `.min(…).max(…)` (see §8).
- Service: NEW service in `src/lib/service/reactCalls/` on the productService pattern, POST to `/api/public/contact` via BACKEND_API.
- Error handling: reuse `parseProductValidationErrors` (`{field,code,translationKey}`) for per-field display (D4). Page-level shape mirrors forgot-password/page.tsx.
  - Interceptor quirk (api.ts:69): catch reads `err.data.errors` and `err.status` — NOT `err.response.data.errors`. Mirror the existing services.
- Visible addresses: render support@oglasino.com and privacy@oglasino.com in the page copy (privacy@ display-only per D2).

Footer:

- Add `{kind:'link', labelKey:'contact.label', route:'/contact'}` to `companyNavigations.tsx`, AFTER `terms.label`, BEFORE the manage-cookies button entry (D5). Label renders through PAGING (tPaging).

## 7. Mobile specification (oglasino-expo)

Screen:

- New `app/(portal)/(public)/contact.tsx`, cloned from privacy.tsx (D7): ScrollView → BackToHomeButton → body → Footer. File-based routing auto-registers `/contact` (no Stack.Screen entry needed).
- Body: support@ and privacy@ as
  - tappable rows: `Linking.openURL('mailto:support@oglasino.com')` (import `Linking` from 'react-native', same call shape as CallUserButton.tsx:65 tel:)
  - AND selectable plain text (D3 hard requirement — degrade gracefully when no mail client is configured).
- No form, no backend call, no auth read needed for v1.

Footer:

- Add the contact row to the company column data (companyNavigations.tsx) for parity with web (D5). Label key `contact.label` in PAGING (already seeded by backend; mobile fetches PAGING in the boot loop).

Strings:

- Drawn from existing COMMON / PAGING / BUTTONS (D6). No new namespace, no i18n/types.ts addition.

## 8. Open numeric parameters (set in engineering briefs)

- message length bounds — SET: message `@Size(min=50, max=2000)` (minimum is server-enforced; was client-side only). email max 254.
- contact endpoint rate-limit category cap (tighter than PUBLIC_WRITE 20/min) — SET: dedicated `RateLimitCategory.CONTACT` = 5/min.
- per-email cooldown window + daily cap — SET: mirror of the auth pattern (60s gap, 5/day).

## 9. Cross-repo dependencies / pre-launch checklist

- support@oglasino.com and privacy@oglasino.com mailboxes: NOW LIVE AND ACTIVE. decisions.md must be corrected (it records them as not-yet-operational — see the Docs/QA correction brief). The form-send destination and the visible mailto links both depend on these being live; they are.
- No new operator provisioning step for the contact destination: it reuses `OGLASINO_EMAIL_REPLY_TO`, already set in stage + prod. Destination and reply-to are the same inbox (support@) in v1, so they are coupled; splitting out a dedicated `oglasino.email.contact-to` property is a clean future change if they ever need to diverge.

## 10. Invariants (do not violate)

1. Contact controller body annotated `@Valid`. (Suggestion controller's omission is a bug, not a model.)
2. Identity derived server-side from the principal only; client email ignored when a principal is present.
3. Mobile addresses selectable as text, not tap-only.
4. No new translation namespace.
5. No DB persistence in v1.
6. Mode switch = server-verified principal presence; never a client flag.
7. Every contact translation key seeded in ALL supported locales (coverage matched to existing sibling keys), never EN-only.

## 11. Session log

- **2026-06-11 · backend · [contact-page-1](../sessions/2026-06-11-oglasino-backend-contact-page-1.md) (audit, read-only):** Phase-2 audit of the email send path, suggestion table, auth/identity, abuse controls, error contract, and trust boundary. Confirmed the email port (`EmailService.sendHtml` + `EmailLayout.wrap`) and the both-modes `/api/public/` + opportunistic-`FirebaseAuthFilter` pattern already cover the feature; the `suggestion` table should NOT host contact rows (reuse its server-derived-userId _pattern_, not the table). Flagged a live gap: `SuggestionController` omits `@Valid`. Audit deliverable archived at [`audit-contact-page-oglasino-backend.md`](../sessions/audit-contact-page-oglasino-backend.md).
- **2026-06-11 · backend · [contact-page-2](../sessions/2026-06-11-oglasino-backend-contact-page-2.md) (build):** `POST /api/public/contact` built on `dev` — `@Valid` `ContactRequestDTO`, server-derived identity (client `email` ignored when a principal is present), `ContactEmailSender` sending to the support inbox via the existing `oglasino.email.reply-to` (no new property), two stacked abuse controls (URL/IP `RateLimitCategory.CONTACT` = 5/min + per-email 60s gap + 5/day Redis cooldown), and a 12-key × 4-locale (EN/RS/CNR/RU) translation seed. New `ContactErrorCode` (VALIDATION-namespace `@Valid` 400 family); send-failure surfaces as 502 `CONTACT_SEND_FAILED`. 15/15 targeted unit tests green; full `@SpringBootTest` integration suite not run (needs local Postgres/Redis/ES) — seed validated structurally. Three brief-vs-reality items resolved (no generic email-code family exists → minted `ContactErrorCode`; destination reuses reply-to; support email frame fixed-English).
- **2026-06-11 · backend · [contact-page-3](../sessions/2026-06-11-oglasino-backend-contact-page-3.md) (follow-on):** Two changes — (1) server-side message minimum 50 via a second repeated `@Size(min=50)`, distinct 400 `CONTACT_MESSAGE_TOO_SHORT` → `validation.contact.message.too_short`; (2) exact-duplicate notice — a Redis marker on (lowercased effective email + SHA-256 of the exact message), 6h TTL, stamped only after a successful send and checked AHEAD of the per-email throttle so a duplicate never consumes a cooldown/daily slot, returned as 422 `CONTACT_DUPLICATE` → `contact.duplicate` (ERRORS, hand-built like `CONTACT_SEND_FAILED`). Throttle byte-for-byte unchanged. 2 new keys × 4 locales seeded. 28 tests green; full integration suite not run. See [decisions.md](../decisions.md) 2026-06-11.
- **2026-06-11 · backend · [contact-page-4](../sessions/2026-06-11-oglasino-backend-contact-page-4.md) (METADATA seed fix):** Seeded the two missing METADATA keys `page.contact.title` + `page.contact.description` across all four locales (EN/RS/CNR/RU), mirroring the `page.about.*` / `page.pricing.*` pattern — fixes the MISSING_MESSAGE the web `/contact` page threw on SR and every non-EN locale.
- **2026-06-11 · backend · [contact-page-5](../sessions/2026-06-11-oglasino-backend-contact-page-5.md) (key-set verify):** Verified the full contact validation key set; found all five keys seeded UN-prefixed (`email.empty`, `email.bad`, `contact.message.empty/too_short/too_long`) and not the `validation.`-prefixed strings the `ContactErrorCode` constants emitted — root-caused the systematic mismatch (0 of 5 messages resolving), feeding the page-6 fix.
- **2026-06-11 · backend · [contact-page-6](../sessions/2026-06-11-oglasino-backend-contact-page-6.md) (ContactErrorCode prefix fix, Option B):** Code-only — dropped the non-existent `validation.` prefix from all five `ContactErrorCode` `translationKey` constants so they match the seeded VALIDATION-namespace keys. Resolves the mismatch; no seed change.
- **2026-06-11 · web · [contact-page-1](../sessions/2026-06-11-oglasino-web-contact-page-1.md) (audit, read-only):** Audited the contact patterns — static page shell, footer column, auth/email state, form/submit, validation parsing — with file:line citations. Audit deliverable archived at [`audit-contact-page-oglasino-web.md`](../sessions/audit-contact-page-oglasino-web.md).
- **2026-06-11 · web · [contact-page-2](../sessions/2026-06-11-oglasino-web-contact-page-2.md) (build):** Built the `/contact` slice — a `(public)` server page on the about/pricing shell with `generateContactMetadata`, a `'use client'` `ContactForm` (anti-flash gate, prepopulate-and-lock email for logged-in users, Zod structural validation), a `contactService` POSTing to `/api/public/contact` via `BACKEND_API` with per-field/form-level error mapping, the footer link, and tests.
- **2026-06-11 · web · [contact-page-3](../sessions/2026-06-11-oglasino-web-contact-page-3.md) (white-box fix):** Removed the visually intrusive placeholder white box on `/contact` first paint (the `useAuthResolved` anti-flash gate's placeholder); the gate itself is correct and stays.
- **2026-06-11 · web · [contact-page-4](../sessions/2026-06-11-oglasino-web-contact-page-4.md) (key-resolution verify):** Confirmed the contact validation keys resolve after the backend un-prefixed fix (`email.empty`, `email.bad`, `contact.message.empty/too_short/too_long` in VALIDATION); web side verified correct, no change needed.
- **2026-06-11 · mobile · [contact-page-1](../sessions/2026-06-11-oglasino-expo-contact-page-1.md) (audit, read-only):** Audited the mobile contact surface — content-screen pattern, mailto/external-link mechanism, auth/email read, form+`parseServiceError` template, translation namespace — with file:line citations. Audit deliverable archived at [`audit-contact-page-oglasino-expo.md`](../sessions/audit-contact-page-oglasino-expo.md).
- **2026-06-11 · mobile · [contact-page-2](../sessions/2026-06-11-oglasino-expo-contact-page-2.md) (build):** Built the static Contact/Support screen (no form, no API) — support@ and privacy@oglasino.com as tappable `mailto` rows AND selectable plain text, plus a footer link, cloned from the privacy/terms shell per §7 and D3/D5/D6/D7.
- **2026-06-11 · mobile · [contact-page-3](../sessions/2026-06-11-oglasino-expo-contact-page-3.md) (label-colour fix):** Fixed the two address-row labels (`contact.page.support.label` / `contact.page.privacy.label`) — dropped the non-theme colour, styled italic + `text-primary`, verified legibility in the real theme.
