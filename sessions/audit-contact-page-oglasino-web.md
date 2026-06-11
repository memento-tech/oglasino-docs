# Audit — Contact/Support Feature (oglasino-web)

Read-only audit. No code changed. Every citation below was verified with both
`view` (Read) and `rg`/`sed`. Branch: `dev`. Date: 2026-06-11.

There is **no existing `/contact` route, page, component, or service** in the
repo (`find app -ipath '*contact*'` → empty; no `contact` route refs in
`app/`/`src/`). This is greenfield. The patterns below are what a contact page
should mirror.

---

## 1. Static / legal / content page pattern

Public content pages live under
`app/[locale]/(portal)/(public)/<slug>/page.tsx`. Two distinct shells exist:

**A. Markdown-backed legal pages** (`privacy`, `terms`):
- `app/[locale]/(portal)/(public)/privacy/page.tsx:17-26` and
  `terms/page.tsx:17-26`.
- Server component. Resolves language via
  `getTenantLocale(await getRoutingLocale())?.oglasinoLocale ?? 'sr'`
  (`privacy/page.tsx:18`) and renders `<MarkdownViewer url={legalDocUrl(...)} />`.
- Not the right template for contact — contact is interactive, not static prose.

**B. Component-composed content pages** (`about`, `pricing`):
- `about/page.tsx:21-108`, `pricing/page.tsx:22-111`.
- Server component, `export default async function`. Pulls multiple translators
  via `getTranslations(TranslationNamespaceEnum.X)`
  (`about/page.tsx:22-23`, `pricing/page.tsx:23-24`), gets locale via
  `getRoutingLocale()` (`about/page.tsx:24`).
- Renders interactive client components inside the server page:
  `pricing/page.tsx:1` imports `SupportButton` (a `'use client'` button,
  `src/components/client/buttons/SupportButton.tsx:8`); `about/page.tsx:3`
  imports `AboutRegisterButton`.

**Metadata generator pattern** (identical across all four):
- `page.tsx` exports `generateMetadata(): Promise<Metadata>` which loads the
  `METADATA` translator + locale and delegates to a per-page generator in
  `src/metadata/` (`about/page.tsx:14-19`, `pricing/page.tsx:15-20`,
  `privacy/page.tsx:10-15`).
- The generator (e.g. `src/metadata/generateAboutMetadata.ts:6-52`) calls
  `getBasicMetada(t)`, builds `title`/`description` from
  `t('page.about.title')` / `t('page.about.description')`, sets a `pagePath`,
  builds `alternates.canonical` + `languages` via
  `buildAlternateLanguages(locale, 'base-site-scoped', ...)` and OG locales via
  `buildOgLocales(locale, 'base-site-scoped')`.
- Pages that need structured data also render `<JsonLd data={generate...StructuredData(...)} />`
  (`about/page.tsx:28`, `pricing/page.tsx:29`). Optional for contact.

**Locale wiring:** `getRoutingLocale()` (server) from
`@/src/i18n/getRoutingLocale`; the `[locale]` route segment + the `(public)`
group give the page its URL. `(public)` means SessionGuard never redirects it —
it can be opened cold/unauthenticated (documented on `forgot-password/page.tsx:15-18`).

**Best template:** **the `about` page** (`about/page.tsx`). It is the closest
match — a server `(public)` page with `generateMetadata` → `src/metadata/`
generator, multiple `getTranslations` namespaces, optional `JsonLd`, that
composes one or more `'use client'` interactive components. A contact page is
"static content shell + one interactive client form," exactly about's shape.
The actual form should be a separate `'use client'` component the server page
renders (mirroring how `about` renders `AboutRegisterButton` and `pricing`
renders `SupportButton`).

---

## 2. Footer structure

**File:** `src/components/server/layout/Footer.tsx` (server component,
`async function Footer`, `:12`). Rendered once by the public layout:
`app/[locale]/(portal)/(public)/layout.tsx:1,11`.

**Column structure** — three columns in a flex row (`Footer.tsx:24`):
1. **Category** column (`:25-39`) — heading `tFooter('category.title')`, links
   built from `baseSite.catalog.categories`.
2. **Portal / company** column (`:40-63`) — heading `tFooter('portal.title')`,
   iterates `companyNavigations` (`src/lib/data/companyNavigations.tsx:5-29`).
3. **Help** column (`:64-78`) — heading `tFooter('help.title')`, iterates
   `helpNavigations` (`src/lib/data/helpNavigations.tsx:1-7`).

Below the columns: app-store badges (`:80-87`) and copyright (`:88`).

**The "Cookies" entry — how it's wired:**
- `companyNavigations` is a discriminated union:
  `{kind:'link', labelKey, route}` or `{kind:'button', id:'manage-cookies'}`
  (`companyNavigations.tsx:1-3`). The cookies entry is the `'button'` variant
  (`companyNavigations.tsx:26-29`, last in the array).
- In the render loop, `if (navigation.kind === 'button')` renders
  `<ManageCookiesFooterButton />` instead of a `<Link>`
  (`Footer.tsx:44-51`).
- `ManageCookiesFooterButton`
  (`src/components/client/consent/ManageCookiesFooterButton.tsx:12-23`) is a
  `'use client'` `<button>` whose `onClick` calls
  `useConsentStore.getState().requestReopen()` (`:18`) — that's the reopen of
  the consent banner. Its label is `t('footer.link.label')` from the **COOKIES**
  namespace (`:13`).

**Where a "Contact" → `/contact` link sits:**
A plain navigation link is the `{kind:'link', labelKey, route}` shape, rendered
as `<Link href={navigation.route}>{tPaging(navigation.labelKey)}</Link>`
(`Footer.tsx:52-60`). Two defensible homes — this is a Mastermind decision:

- **Company/portal column** (`companyNavigations.tsx`): holds the sibling
  static content pages (`about.label`→`/about`, `pricing.label`→`/pricing`,
  `privacy.label`→`/privacy`, `terms.label`→`/terms`, then the manage-cookies
  button). Adding `{kind:'link', labelKey:'contact.label', route:'/contact'}`
  here keeps the contact page in the same family as about/pricing. **Position:**
  insert it among the `'link'` entries — recommended *after* `terms.label` and
  **before** the `manage-cookies` button entry, so the consent button stays
  last (it is the only `'button'` and reads naturally as the trailing item).
- **Help column** (`helpNavigations.tsx`): currently only `blog.free.zone.label`.
  Semantically "Contact / support" is help. Adding
  `{labelKey:'contact.label', route:'/contact'}` here is also clean
  (`helpNavigations` is a plain `{labelKey, route}[]`, no union).

Recommendation: **company/portal column, before manage-cookies** — page-family
consistency with about/pricing/privacy/terms, and that column already groups
"about the company / legal / settings" entries. (Flagged for Mastermind to
confirm; both options use the same translation namespace, so the key contract
is identical either way.)

**Translation namespace for the label:** both `companyNavigations` and
`helpNavigations` link labels are rendered through **`tPaging`** =
`TranslationNamespaceEnum.PAGING` (`Footer.tsx:15` declares
`tPaging = getTranslations(TranslationNamespaceEnum.PAGING)`; `:58` and `:72`
render `tPaging(navigation.labelKey)`). The existing keys are `about.label`,
`pricing.label`, `privacy.label`, `terms.label`, `blog.free.zone.label`. So a
**"Contact" footer label belongs in the `PAGING` namespace**, key shape
`contact.label`. (Column *headings* — `category.title`/`portal.title`/`help.title`
— are in the **FOOTER** namespace, `Footer.tsx:14`; no new heading is needed
since Contact reuses an existing column.) The cookies button is the exception:
its label is in COOKIES, not PAGING, because it's the `button` variant.

> Per CLAUDE.md, adding the `contact.label` key (and any form keys) to the SQL
> seed is the Backend engineer's job — web only identifies the missing keys.

---

## 3. Auth state on a public page (logged-in? + email, client-side)

**Reading login state + email:** `useAuthStore`
(`src/lib/store/useAuthStore.ts`, a Zustand store). The signed-in backend user
is `state.user: AuthUserDTO | null` (`useAuthStore.ts:71`). `user === null`
means signed out. The email is **`user.email`** —
`AuthUserDTO.email: string` (`src/lib/types/user/AuthUserDTO.ts:8`). Client
usage: `const user = useAuthStore((s) => s.user)` then `user?.email`.

**The hydration-flash gate:** `useAuthResolved()`
(`src/lib/hooks/useAuthResolved.ts:13-27`). It subscribes to Firebase
`onAuthStateChanged` and returns `true` only once Firebase has reported auth
state **and**, if a Firebase user exists, the store's backend user has synced
(`return firebaseReady && (!firebaseUser || storeUser !== null)`, `:26`). Its
own doc comment (`:8-12`) states the exact concern from the brief: *"Auth-gated
UI should render `null` (or a placeholder) until this is true, otherwise
signed-in users flash the signed-out variant on first paint."* It is already
used by `HeaderNavButtons` and `MobileFooterNavigation`.

**Correct gate for "prepopulate + lock email for logged-in users":** the form
must be a `'use client'` component that:
1. calls `const resolved = useAuthResolved()` and
   `const user = useAuthStore((s) => s.user)`;
2. renders nothing/placeholder for the email field (or the whole form) until
   `resolved === true`, so the field never flashes empty-unlocked then fills;
3. once resolved: if `user` is non-null, seed the email state from `user.email`
   and pass `disabled` to lock it; otherwise leave it empty + editable.

The existing `Input` component (`src/components/server/Input.tsx`) supports both
needs directly: it accepts `disabled` (renders `cursor-not-allowed text-gray-400`,
`Input.tsx:48,92,103` region) and a controlled `value`/`onChange`
(`Input.tsx:6-20`). So "lock" = pass `disabled={!!user}` and seed `value` from
`user.email`. **Hook to use: `useAuthResolved` for the gate, `useAuthStore`
selector for the email.**

---

## 4. Form + submit pattern (POST to backend, surface errors)

Two candidate models. They differ in error granularity — pick by how many
per-field errors the contact form needs.

**(a) Single-message dialog form — `ReportDialog`**
(`src/components/popups/dialogs/ReportDialog.tsx`):
- `'use client'`, `useState` per field
  (`option`/`reportDescription`/`errorMessage`/`success`/`loading`, `:52-56`),
  client structural checks first (`:64-82`), then `await sendReport({...})`
  (`:91-98`), then branches on the result (`:100-108`).
- **It surfaces only ONE error string** (`errorMessage`,
  rendered `:157-164`). The service maps a backend code to a single
  `errorTranslationKey` and the dialog shows it — there is **no per-field**
  inline error rendering here. Good model for a *simple* contact form (one
  message area), not for true per-field `{field,code,translationKey}` display.

**(b) True per-field `{field,code,translationKey}` model — product create/edit**
(`src/lib/service/reactCalls/productService.ts` +
`src/lib/utils/parseProductValidationErrors.ts`):
- `createNewProduct` POSTs to `/secure/products/create`
  (`productService.ts:152-175`) and on a contract-shaped error body returns
  `{type:'validation', errors: ParsedProductValidationErrors}`
  (`:178,194`).
- `parseProductValidationErrors` (`parseProductValidationErrors.ts:31-45`) turns
  the backend `{errors:[{field,code,translationKey}]}` envelope into two views:
  `byField: Record<field, translationKey>` (display, first-error-per-field wins)
  and `list: {field,code,translationKey}[]` (analytics). `field:null`
  object-level errors collapse under `SYSTEM_ERROR_KEY = '__system'`
  (`:6,38`). Consumed by the form dialogs
  (`BasicInfoProductDialog`, `UploadedProductDialog`,
  `app/[locale]/owner/products/[productId]/page.tsx`) which read
  `errors.byField['<field>']` to show inline per-field messages.

**Cleanest model for the contact form:** if the contact form needs per-field
errors (e.g. `email` invalid vs `message` too long surfaced separately), model
the **service** on `productService` + reuse the **`parseProductValidationErrors`
contract** (it is generic over `{field,code,translationKey}` despite the
"Product" name). If one combined error area suffices, `ReportDialog` +
`reportService` is the lighter model. Recommendation noted in the session
summary.

**Service-call layer (`reactCalls/`) + error-envelope parsing:**
- Services live in `src/lib/service/reactCalls/*Service.ts` and POST via the
  shared axios instance `BACKEND_API` from `@/src/lib/config/api`
  (`reportService.ts:1,57`; `productService.ts:1,171`;
  `passwordResetService.ts:1,36`).
- **Critical envelope quirk:** `BACKEND_API`'s response interceptor rejects with
  the **unwrapped response**, not the raw `AxiosError` (`api.ts:69`
  `return Promise.reject(error.response)`). So in catch blocks the status is
  `err.status` (top-level) and errors are at `err.data.errors` — **not**
  `err.response.data.errors`. Services tolerate both shapes:
  `reportService.ts:46-53` (`readWireErrors`), `isErrorWithCode.ts:5-15`,
  `parseProductValidationErrors`'s callers (`productService.ts:194` reads
  `err.status`/`err.data`). A contact service must read `err.data.errors` /
  `err.status`, mirroring these.
- The interceptor also special-cases network/timeout → `{data:{errorCode:'connection.timeout'},status:0}`
  (`api.ts:38-42`), 404 → `not.found` (`:44-49`), and 403 `USER_BANNED`
  (`:51-67`). A contact form hitting a non-secure endpoint still passes through
  these.
- A non-dialog public-page form that POSTs by email only and renders one
  status message is `forgot-password/page.tsx:20-104` — closest structural
  precedent for a **public-page** (not dialog) form: `useState` fields,
  `await requestPasswordReset(email)` (`:69`), `applyResult` switch on a typed
  result union (`:47-64`), `MessageText` for the status line. The contact form's
  page-level shape should mirror this; its richer error handling can come from
  model (b).

---

## 5. Validation (client-side Zod, structural only)

**The only Zod schema file in the repo:** `src/lib/validators/productSchemas.ts`
(confirmed: `ls src/lib/validators/` → `productSchemas.ts`,
`productValidator.ts`, plus `src/metadata/schemaOrgMappings.ts` which is
unrelated schema.org mapping, not Zod).

**Pattern** (`productSchemas.ts:1-28`):
- `import { z } from 'zod'` (`:1`), explicit comment that schemas are
  **structural-only** — shape and length, content moderation is server-side
  (`:3-4`).
- Named numeric constants (`NAME_MIN`, `DESCRIPTION_MAX`, etc., `:6-10`).
- Field schemas built with `z.string().min(...).max(...)` (`:12-13`); optional
  fields use `.optional().refine(...)` with a `{ message: 'invalid' }` code
  (`:14-17`); composed into an object schema via `z.object({...})` (`:21-27`).
- Used by `src/lib/validators/productValidator.ts` (the runner), tested by
  `productValidator.test.ts`.

**What a contact form's Zod schema would look like** (structural only; matches
the existing style — constants + `z.object`):

```ts
import { z } from 'zod';

const EMAIL_MAX = 254;
const MESSAGE_MIN = 10;
const MESSAGE_MAX = 2000;

const emailSchema = z.string().email().max(EMAIL_MAX);
const messageSchema = z.string().min(MESSAGE_MIN).max(MESSAGE_MAX);

export const contactSchema = z.object({
  email: emailSchema,
  message: messageSchema,
});
```

Notes:
- `z.string().email()` is the structural email check (format only). Server
  remains the single source of truth for content moderation (spam, links,
  banned words) per CLAUDE.md + the `productSchemas.ts:3-4` comment.
- Length bounds (`MESSAGE_MIN`/`MESSAGE_MAX`) mirror the description bounds in
  `productSchemas.ts`. Exact numbers are Mastermind's/the feature spec's call;
  the *pattern* (named constants + `z.object`) is what to match.
- For a logged-in user with a locked, prepopulated email, the email field is
  trusted-from-`user.email` but the schema should still validate it
  defensively.

---

## Summary of named targets

| Need | Use |
| --- | --- |
| Page shell template | `about/page.tsx` (server `(public)` page composing a client component) |
| Metadata | `generateMetadata` → new `src/metadata/generateContactMetadata.ts` modeled on `generateAboutMetadata.ts` |
| Footer link | add `{kind:'link', labelKey:'contact.label', route:'/contact'}` to `companyNavigations.tsx`, before the manage-cookies button (alt: `helpNavigations.tsx`) |
| Footer label namespace | `PAGING` (`contact.label`) — Backend seeds the key |
| Logged-in? + email | `useAuthStore((s)=>s.user)` → `user.email` (`AuthUserDTO.email`) |
| Anti-flash gate | `useAuthResolved()` |
| Lock the email field | `Input` with `disabled` + seeded `value` |
| Form/submit + per-field errors | service on `productService` pattern, reuse `parseProductValidationErrors` `{field,code,translationKey}`; or `ReportDialog`/`reportService` for single-message |
| Error envelope | `err.data.errors` + `err.status` (interceptor unwraps — `api.ts:69`) |
| Zod | new `contactSchema` modeled on `productSchemas.ts` (structural only) |
