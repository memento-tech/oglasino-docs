# Audit — QA Preparation: Route Inventory + QA Page Structure

**Repo:** oglasino-web
**Branch:** feature/qa-preparation
**Date:** 2026-05-14
**Type:** Read-only audit.

---

## Task 1 — Route inventory

App Router root: `app/`. Locale prefix `[locale]` (next-intl) wraps every user-facing page. Route groups in parentheses (`(portal)`, `(public)`, `(protected)`) do not appear in URLs. Routes below list the URL path on the left and the source file on the right.

### User-facing pages

#### Root

- `/` → `app/page.tsx`
- `/[locale]` → `app/[locale]/(portal)/(public)/page.tsx`

#### Portal — public

- `/[locale]/about` → `app/[locale]/(portal)/(public)/about/page.tsx`
- `/[locale]/blog/free-zone` → `app/[locale]/(portal)/(public)/blog/free-zone/page.tsx`
- `/[locale]/catalog/[[...slugs]]` → `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx`
- `/[locale]/pricing` → `app/[locale]/(portal)/(public)/pricing/page.tsx`
- `/[locale]/privacy` → `app/[locale]/(portal)/(public)/privacy/page.tsx`
- `/[locale]/product/[productId]/[productName]` → `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx`
- `/[locale]/terms` → `app/[locale]/(portal)/(public)/terms/page.tsx`
- `/[locale]/user/[userId]` → `app/[locale]/(portal)/(public)/user/[userId]/page.tsx`

#### Portal — protected

- `/[locale]/favorites` → `app/[locale]/(portal)/(protected)/favorites/page.tsx`
- `/[locale]/messages` → `app/[locale]/(portal)/(protected)/messages/page.tsx`
- `/[locale]/notifications` → `app/[locale]/(portal)/(protected)/notifications/page.tsx`

#### Owner (dashboard)

- `/[locale]/owner` → `app/[locale]/owner/page.tsx`
- `/[locale]/owner/account-verification` → `app/[locale]/owner/account-verification/page.tsx`
- `/[locale]/owner/analytics` → `app/[locale]/owner/analytics/page.tsx`
- `/[locale]/owner/balance` → `app/[locale]/owner/balance/page.tsx`
- `/[locale]/owner/follows` → `app/[locale]/owner/follows/page.tsx`
- `/[locale]/owner/not-ready` → `app/[locale]/owner/not-ready/page.tsx`
- `/[locale]/owner/products` → `app/[locale]/owner/products/page.tsx`
- `/[locale]/owner/products/[productId]` → `app/[locale]/owner/products/[productId]/page.tsx`
- `/[locale]/owner/reviews` → `app/[locale]/owner/reviews/page.tsx`
- `/[locale]/owner/user` → `app/[locale]/owner/user/page.tsx`

#### Admin

- `/[locale]/admin` → `app/[locale]/admin/page.tsx`
- `/[locale]/admin/cache` → `app/[locale]/admin/cache/page.tsx`
- `/[locale]/admin/chats/[userId]` → `app/[locale]/admin/chats/[userId]/page.tsx`
- `/[locale]/admin/chats/messages/[userId]/[chatId]` → `app/[locale]/admin/chats/messages/[userId]/[chatId]/page.tsx`
- `/[locale]/admin/config` → `app/[locale]/admin/config/page.tsx`
- `/[locale]/admin/es` → `app/[locale]/admin/es/page.tsx`
- `/[locale]/admin/products` → `app/[locale]/admin/products/page.tsx`
- `/[locale]/admin/products/[userId]` → `app/[locale]/admin/products/[userId]/page.tsx`
- `/[locale]/admin/products/product/[productId]` → `app/[locale]/admin/products/product/[productId]/page.tsx`
- `/[locale]/admin/reports` → `app/[locale]/admin/reports/page.tsx`
- `/[locale]/admin/reviews` → `app/[locale]/admin/reviews/page.tsx`
- `/[locale]/admin/statistics` → `app/[locale]/admin/statistics/page.tsx`
- `/[locale]/admin/suggestions` → `app/[locale]/admin/suggestions/page.tsx`
- `/[locale]/admin/translations` → `app/[locale]/admin/translations/page.tsx`
- `/[locale]/admin/users` → `app/[locale]/admin/users/page.tsx`
- `/[locale]/admin/users/[userId]` → `app/[locale]/admin/users/[userId]/page.tsx`

#### Other top-level

- `/[locale]/wants` → `app/[locale]/wants/page.tsx`

### Not user-facing (API routes, dev/QA pages)

- `/api/auth/token` (route handler) → `app/api/auth/token/route.ts`
- `/api/revalidate` (route handler) → `app/api/revalidate/route.ts`
- `/[locale]/design` (the QA page — confirmed by Igor) → `app/[locale]/design/page.tsx`
- `/[locale]/icons` (likely dev icon viewer) → `app/[locale]/icons/page.tsx`
- `/[locale]/test/notifications` (path indicates dev/test page) → `app/[locale]/test/notifications/page.tsx`

---

## Task 2 — QA page structure audit

**Path confirmed by Igor:** `app/[locale]/design/page.tsx`. Route: `/[locale]/design`.

### Page layout

The page is a single client component (`'use client'`) named `QaPortalDocumentationPage`. Two structural regions: a fixed header and a long-scroll main column.

**Header** (`<header>`, fixed, `top-0 z-50`):

- Title block: H1 "Oglasino QA Documentation" + a one-line subtitle.
- A single text `<input>` ("Search topics...") sitting in the same row as the title at `lg` and above.
- A topic-chip nav row beneath. Renders one `<button>` per topic showing `topic.shortLabel`, in a `flex flex-wrap` container. The container has a collapsed `max-h-[44px]` (≈ one row of chips) and an expanded `max-h-[500px]`.
- A "Show all / Collapse" toggle button to the right of the chip row, conditionally rendered only when the chip container overflows (`scrollHeight > clientHeight + 2`, checked on mount and on window `resize`).
- The whole header hides on scroll-down past 80px and reappears on scroll-up. Implemented via a window scroll listener + `requestAnimationFrame` (`page.tsx:54-80`).

**Main content** (`<main>`, scrollable column):

- Maps `filteredTopics` to `<section>` elements, one per topic.
- Each section sets `id={topic.id}` so the header chips can anchor-scroll to it. `scroll-mt-28` keeps the section title clear of the fixed header.
- Section structure: H2 (`topic.title`), a small `route` line, a description paragraph, an optional image carousel, a 2-column grid of four `<Info>` cards (Goals, Constraints, QA Checklist, Risks), and an optional related-links footer.
- `<Info>` is a tiny local component rendering a titled bullet list from `string[]` (`page.tsx:204-218`).
- The image carousel reuses an existing component: `ProductImageCarousel` imported from `@/src/components/client/ProductImageCarusel`, fed `imageKeys={topic.images}`. Rendered only when `topic.images.length > 0`. Today no topic has images, so the carousel never renders.
- Related links render as anchor-tag chips. `target="_blank"` is added only when `href.startsWith('http')`.

**Search bar behavior** (`page.tsx:29-35`):

- Client-side filter. No debounce.
- Match: `(t.title + t.description).toLowerCase().includes(search.toLowerCase())` — case-insensitive substring against the concatenation of `title` and `description`. No tokenization, no fuzzy match, no field-targeted match. `shortLabel`, `goals`, `constraints`, `qaChecklist`, `knownRisks`, and `relatedLinks` are **not** part of the search corpus.
- Empty/whitespace-only search short-circuits to the full list (`if (!search.trim()) return qaTopics`).
- The filter applies to both header chips and main content sections — they read the same `filteredTopics` array — so chips and sections always stay in sync.

**In-page scroll** (`page.tsx:9-14`):

- `scrollToSection(id)` does `document.getElementById(id).scrollIntoView({ behavior: 'smooth', block: 'start' })`. No URL hash update, no history entry.

### Data source

- File: `app/[locale]/design/topics.ts`.
- Format: a TypeScript module exporting a `QaTopic` type and a `qaTopics: QaTopic[]` const literal.
- Loading: static import at the top of `page.tsx` (`import { qaTopics } from './topics'`). Bundled at build time. No runtime fetch, no JSON file, no dynamic import.
- The brief refers to "content JSON" — there is no JSON in the implementation. Content is authored directly in TypeScript and ships as part of the page bundle. See "For Mastermind" below.

### Schema

The `QaTopic` type (`topics.ts:1-19`) in full:

```ts
export type QaTopic = {
  id: string;
  title: string;
  shortLabel: string;
  route: string;
  description: string;

  goals: string[];
  constraints: string[];
  qaChecklist: string[];
  knownRisks: string[];

  relatedLinks: {
    label: string;
    href: string;
  }[];

  images: [];
};
```

Field-by-field:

- **`id`** (`string`) — unique key per topic. Used three ways: as the React `key` for nav chips and sections, as the DOM `id` attribute on the section, and as the scroll-target argument to `scrollToSection`. Format is free-form kebab-case in practice (`home-page`, `category-page`, `user-profile-topic`, `portal-config-topic`); no validation.
- **`title`** (`string`) — full title shown as the section H2.
- **`shortLabel`** (`string`) — short label shown on the header nav chip. Independent from `title`; usually shorter (`"Home"` vs `"Home Page"`).
- **`route`** (`string`) — a free-form display string shown in small text under the title. **Not** a Next.js href, **not** clickable. Values in the existing entries mix conventions: real-looking paths (`/`, `/about-us`, `/pricing`), parameterized paths (`/category/:id`), a popup marker (`popup:user-profile`), and a function-style path (`/function/suggest-category`).
- **`description`** (`string`) — paragraph shown under the section header.
- **`goals`** (`string[]`) — items rendered as a bulleted list in the "Goals" `<Info>` card.
- **`constraints`** (`string[]`) — items rendered in the "Constraints" `<Info>` card.
- **`qaChecklist`** (`string[]`) — items rendered in the "QA Checklist" `<Info>` card.
- **`knownRisks`** (`string[]`) — items rendered in the "Risks" `<Info>` card (note the type field name uses "knownRisks" but the heading shown is "Risks").
- **`relatedLinks`** (`{ label: string; href: string }[]`) — anchor-tag chips at the bottom of the section. Section is omitted when the array is empty. External-target detection is `href.startsWith('http')`.
- **`images`** (`[]`) — typed as an empty tuple. Every existing entry sets `images: []`. Passed to `ProductImageCarousel` as `imageKeys` when non-empty. Because the type is literally `[]`, adding image keys would require widening the type (e.g. `string[]`). See "For Mastermind."

A representative entry (`topics.ts:22-69`):

```ts
{
  id: 'home-page',
  title: 'Home Page',
  shortLabel: 'Home',
  route: '/',
  description: '...',
  goals:        ['Provide randomized product discovery on load and refresh', ...],
  constraints:  ['Product listing must support randomization without duplication bugs', ...],
  qaChecklist:  ['Verify random product refresh behavior', ...],
  knownRisks:   ['Incorrect randomization logic causing repeated products', ...],
  relatedLinks: [
    { label: 'Production', href: 'https://oglasino.com' },
    { label: 'Staging',    href: 'https://stage.oglasino.com' },
  ],
  images: [],
}
```

Nesting: none beyond `relatedLinks` (an array of `{label, href}` records). All four bullet-list fields are flat `string[]`. There are no sections-within-a-topic, no sub-features, no sub-flows. A topic is a single-level record.

### Topic model

- **Keying:** by `id` (string). The same `id` serves as React key, DOM id, and scroll target. There is no separate "slug" or numeric key.
- **Header → content mapping:** the same `qaTopics` array is mapped twice. The header maps it into `<button>` chips that call `scrollToSection(topic.id)`. The main column maps it into `<section id={topic.id}>...</section>` blocks. The mapping is implicit through the shared `id` value; nothing else ties the two together.
- **Topic types:** there is no `type` field, no enum, no discriminated union. A "topic" is uniformly one record shape. The `route` string is the only place a topic communicates whether it represents a page (`/about-us`), a popup (`popup:user-profile`), a function (`/function/suggest-category`), or a parameterized route (`/category/:id`) — and that distinction is by string convention only, not enforced.

### Prod exclusion

**None.** The page has no guard.

- The page file is plain `'use client'` with a default-exported component; no env check, no `notFound()`, no role check.
- No project-level `middleware.ts` exists in this repo (only `node_modules/...` and `.next/...` build artifacts match a middleware filename).
- `next.config.*` (not edited, not opened — out of scope) has no `qa`/`design` reference.
- No build-time flag wraps the route, and there is no parent layout for `/[locale]/design/` (it inherits the `[locale]` layout only).

Net effect: visiting `/{locale}/design` in any environment renders this page to anyone who reaches the URL. If "not shipped to prod" is a hard requirement, a mechanism still needs to be picked and added — see "For Mastermind."

### Seams

- **Content vs reality.** The `route` strings on existing entries (e.g. `/category/:id`) are free-form and do not need to match the actual App Router. The real category surface in this codebase is `/[locale]/catalog/[[...slugs]]` (Task 1). Any future content pass that wants `route` to be a real Next.js path will need to keep it in sync manually — there is no validation.
- **Image carousel coupling.** The page reuses `ProductImageCarousel` from the product surface and passes `imageKeys={topic.images}`. Whatever shape that component expects for `imageKeys` becomes part of the QA-content authoring contract once images are added. The QA page does not own the carousel.
- **Cross-repo facts.** When the JSON-vs-TS question is decided and the content is filled in, the descriptions / constraints / risks fields will assert things about backend behavior, validation rules, error codes, and translation keys. Keeping that content correct will require facts owned by the Backend Engineer and the Docs/QA agent — but this is a content concern, not a structural one, and out of scope for this audit.
- **No URL hashing.** `scrollToSection` does not update `window.location.hash` and the page does not read a hash on mount. Topics are not deep-linkable today.

---

## Implemented

- Walked the App Router and produced a flat route list grouped by surface (root, portal public/protected, owner, admin, other top-level, plus a separate non-user-facing list for API handlers and dev/QA pages).
- Audited `app/[locale]/design/page.tsx` and `app/[locale]/design/topics.ts` for structure, schema, data source, topic model, prod exclusion, and seams.
- No code changes; output is this audit file plus its Part 5 duplicates.

## Files touched

- `.agent/audit-qa-preparation.md` (created)
- `.agent/2026-05-14-oglasino-web-qa-preparation-1.md` (created, duplicate)
- `.agent/last-session.md` (overwritten with the same content)

## Tests

- Not run. Read-only audit, no code paths touched.

## Cleanup performed

- none needed (read-only audit; no code changes)

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): N/A this session (no code touched)
- Part 4a (simplicity) / Part 4b (adjacent observations): adjacent observations flagged in "For Mastermind" below
- Part 6 (translations): N/A this session
- Other parts touched: Part 5 (session summary structure) — confirmed; audit duplicated to `.agent/last-session.md` and to `.agent/2026-05-14-oglasino-web-qa-preparation-1.md` per the naming rule.

## Known gaps / TODOs

- none

## For Mastermind

1. **The content is TypeScript, not JSON.** The brief asked for "where the content JSON lives," "exact shape of the JSON," and called the page "content-driven from JSON." The implementation is a TS module: `app/[locale]/design/topics.ts` exports a `QaTopic` type and a `qaTopics: QaTopic[]` const, statically imported into the page and bundled at build time. Functionally the type acts as the schema, but if the intent is "content separate from code, easier to author without a TS toolchain, hot-swappable without a rebuild," then a JSON file + runtime load is a different shape and would need a deliberate switch. Flagging because the brief's phrasing suggests JSON may have been assumed — not the same as confirmed.

2. **No prod-exclusion mechanism exists.** The brief frames QA as "not shipped to prod." Today there is no middleware guard, no env check, no build flag, no role check — `/{locale}/design` is reachable in every environment. Mechanism options (env-gated 404, `NEXT_PUBLIC_*` build flag with a `notFound()` early return, middleware rewrite, admin role guard) need a Mastermind decision before any engineer adds one. No fix attempted in this read-only audit.

3. **`QaTopic.images` is typed as `[]` (empty tuple).** Severity: low. Every existing entry conforms because every entry is literally `images: []`. The first attempt to add an image key (`images: ['some-key']`) will fail typecheck. Easy fix when content authoring starts — widen to `string[]` — but worth noting before someone hits it cold.

4. **Stray file `app/[locale]/design/t.txt` (24 lines).** Severity: low. Looks like a scratch file in the QA page folder. Not imported anywhere. Not deleted in this session (read-only audit; file is unrelated to the audit's scope). Flagging per Part 4b.

5. **`route` field is a free-form display string, not a Next.js href.** Severity: low. Values across existing entries mix three conventions (real paths, parameterized paths, `popup:`-prefixed marker, `/function/...` paths). Whatever convention Mastermind decides for the canonical content pass should be documented, and ideally the field's role ("display only" vs "real anchor") should be settled — otherwise the second author will pick a different convention.

6. **Search corpus is `title + description` only.** Severity: low. If future content authoring expects search to find topics by keyword text inside `goals`/`constraints`/`qaChecklist`/`knownRisks`, the filter (`page.tsx:32-34`) will need broadening. Today, searching for a word that only appears in a checklist item won't match.

7. **Brief vs reality on base branch (carried over from the held-position version of this file).** The brief's opening line says "create a new branch `feature/qa-preparation` off `main`" but its "Hard rules" section says "no branch changes beyond creating `feature/qa-preparation` off `dev`." Inconsistent. Did not affect this session (Igor pre-created the branch). Flagging for any future brief that reuses the template.
