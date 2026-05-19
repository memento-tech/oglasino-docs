# QA Preparation

The canonical spec for the QA Preparation feature. Reflects the route inventory and structure audit from 2026-05-14 plus the schema decisions logged in `decisions.md` (2026-05-14, "QA Preparation: schema and structure decisions"). This spec describes what the feature will be after the engineering and content work — not the current state of the `/design` page.

## Goal

One in-app QA reference page. A QA tester opens it, finds a topic by browsing the header or searching, and reads a structured account of a page, feature, or flow: what it is, what the controls are, how to use it, what correct behavior looks like, where it breaks, and what to test. Topics cross-reference each other so a tester reading a page topic can jump to that page's feature topics.

The page exists at `/[locale]/design` in `oglasino-web`. It is content-driven from a TypeScript data module. It may be visible in production — its content only describes behavior already reachable through the portal, so there is nothing to gate.

## Scope

### In scope

- A redesigned `QaTopic` schema (defined below).
- A redesigned `/[locale]/design` page that renders topics against that schema.
- Topic content authored for the in-scope pages and their main features (the page list below).

### Out of scope

- Prod-exclusion mechanism. Decided against — see `decisions.md`.
- Backend changes. This feature is web + docs only. If content authoring needs a backend fact verified, that is escalated as a separate question, not folded in.
- Mobile. The QA page is web-only.
- Deep-linking topics by URL hash. The audited page scrolls by `id` without touching the URL; that stays. Revisit post-launch if QA wants shareable topic links.

## Page list

From the `oglasino-web` route inventory (2026-05-14), trimmed by Igor. Each row marked `in` becomes a `page` topic. Feature and flow topics are not enumerated here — they are discovered during content authoring, when a page brief surfaces a main feature worth its own topic.

### Portal pages

| Route                                         | Source                               | Scope |
| --------------------------------------------- | ------------------------------------ | ----- |
| `/[locale]` (home)                            | `(portal)/(public)/page.tsx`         | in    |
| `/[locale]/about`                             | `(portal)/(public)/about`            | in    |
| `/[locale]/blog/free-zone`                    | `(portal)/(public)/blog/free-zone`   | in    |
| `/[locale]/catalog/[[...slugs]]`              | `(portal)/(public)/catalog`          | in    |
| `/[locale]/pricing`                           | `(portal)/(public)/pricing`          | in    |
| `/[locale]/privacy`                           | `(portal)/(public)/privacy`          | in    |
| `/[locale]/product/[productId]/[productName]` | `(portal)/(public)/product`          | in    |
| `/[locale]/terms`                             | `(portal)/(public)/terms`            | in    |
| `/[locale]/user/[userId]`                     | `(portal)/(public)/user`             | in    |
| `/[locale]/favorites`                         | `(portal)/(protected)/favorites`     | in    |
| `/[locale]/messages`                          | `(portal)/(protected)/messages`      | in    |
| `/[locale]/notifications`                     | `(portal)/(protected)/notifications` | in    |

### Owner (dashboard) pages

All in scope. Some owner sub-pages are intentionally unfinished — those get a thin topic: `overview` filled, other sections empty. A thin topic is valid; the schema allows every section but `overview` to be absent.

| Route                                  | Source                       | Scope |
| -------------------------------------- | ---------------------------- | ----- |
| `/[locale]/owner`                      | `owner/page.tsx`             | in    |
| `/[locale]/owner/account-verification` | `owner/account-verification` | in    |
| `/[locale]/owner/analytics`            | `owner/analytics`            | in    |
| `/[locale]/owner/balance`              | `owner/balance`              | in    |
| `/[locale]/owner/follows`              | `owner/follows`              | in    |
| `/[locale]/owner/not-ready`            | `owner/not-ready`            | in    |
| `/[locale]/owner/products`             | `owner/products`             | in    |
| `/[locale]/owner/products/[productId]` | `owner/products/[productId]` | in    |
| `/[locale]/owner/reviews`              | `owner/reviews`              | in    |
| `/[locale]/owner/user`                 | `owner/user`                 | in    |

### Out of scope

| Route                               | Reason                           |
| ----------------------------------- | -------------------------------- |
| `/[locale]/admin` and 15 sub-routes | Admin surface — not in this pass |
| `/[locale]/wants`                   | Out                              |
| `/[locale]/icons`                   | Dev icon viewer                  |
| `/[locale]/test/notifications`      | Dev/test page                    |
| `/[locale]/design`                  | The QA page itself               |

Total: 22 `page` topics. Feature and flow topics are added during content authoring.

## Schema

The content lives in `app/[locale]/design/topics.ts` as a TypeScript module exporting the types below and a `qaTopics: QaTopic[]` const. Statically imported into the page, bundled at build time. The type is the schema — content is authored against it with typecheck enforcement.

```ts
export type QaTopicType = "page" | "feature" | "flow" | "other";

export type QaImage = {
  name: string; // kebab-case descriptive filename, e.g. 'home-page-filters-open.png'
  description: string; // what the screenshot should show — written for whoever shoots it
  imageKey?: string; // R2 object key, not a URL — filled in after the screenshot is uploaded
};

export type QaRelatedTopic = {
  topicId: string; // the `id` of another QaTopic — drives in-page scroll to that topic
  label?: string; // optional display override; defaults to the target topic's title
};

export type QaRelatedLink = {
  label: string;
  href: string; // external URL
};

export type QaTopic = {
  id: string;
  type: QaTopicType;
  title: string;
  shortLabel: string;
  route: string; // free-form display string, not a clickable href

  overview: string; // prose — what this page/feature/flow is

  // The five optional content sections. An empty or absent
  // array means the section does not render.
  optionsControls?: string[];
  howToUse?: string[];
  whatToExpect?: string[];
  pitfalls?: string[];
  qaChecklist?: string[];

  images?: QaImage[];
  relatedTopics?: QaRelatedTopic[];
  relatedLinks?: QaRelatedLink[];
};
```

Notes on the schema:

- **`overview` is required.** Every topic has at least an id, type, title, labels, route, and an overview. Everything else is optional. A thin topic — `overview` only — is valid and is the expected shape for intentionally-unfinished pages.
- **`type`** drives nothing structural today beyond letting the header group or filter chips by type. It is recorded so the page can use it without a schema change later.
- **The content sections** are `overview` (required prose) plus five optional `string[]` arrays: `optionsControls`, `howToUse`, `whatToExpect`, `pitfalls`, `qaChecklist`. Every optional section that is empty or absent does not render — same rule the audited page already applied to `relatedLinks`, extended to every section.
- **`relatedTopics`** is the cross-reference mechanism. It holds the `id`s of other topics. Clicking one scrolls to that topic on the same page — the QA flow: read the home page topic, click through to the filters feature topic. Separate from `relatedLinks`, which is external URLs only.
- **`images`** is an array of `QaImage`. The content author writes `name` and `description`. Igor shoots the screenshot, names it to match, uploads to R2, fills `imageKey`. The carousel renders an image only once `imageKey` is set — entries with `name`/`description` but no `imageKey` are pending and do not render.
- **`route`** stays a free-form display string. It is shown in small text under the title. It is not a clickable href and does not need to match the App Router. Content authors should still write it to match the real route where one exists.

## Page behavior

The redesigned page keeps the audited page's UI shape. Changes are in what it renders, not how it looks.

- **Layout:** fixed header + long-scroll main column. Unchanged.
- **Header:** title block, search input, topic-chip nav row, show-all/collapse toggle. Unchanged in structure. The chip nav may group or label chips by `type` — engineer's call on whether that lands in this brief or a follow-up.
- **Search:** the corpus broadens. Search matches against `title`, `overview`, and all five content-section arrays (`optionsControls`, `howToUse`, `whatToExpect`, `pitfalls`, `qaChecklist`). Case-insensitive substring, as today. The audited page searched only `title + description` — that is the half-broken behavior this fixes.
- **Topic section:** renders `title` (H2), `route` (small line), `overview` (prose), then the optional content sections as titled bullet-list cards, then images, then related topics, then related links. Any section whose data is empty or absent does not render.
- **Content cards:** the content maps onto cards in the same visual style as the audited `<Info>` cards. The card set changes from Goals/Constraints/QA Checklist/Risks to Overview / Options & Controls / How to use / What to expect / Pitfalls / QA checklist. The visual treatment stays close to what exists.
- **Related topics:** render as chips. Clicking scrolls to the target topic by `id` — reuses the existing `scrollToSection` behavior. No URL hash change.
- **Related links:** render as anchor chips, external target when `href` starts with `http`. Unchanged from audited behavior.
- **Images:** render via the existing `ProductImageCarousel`, fed the `imageKey` values of `QaImage` entries that have an `imageKey`. Carousel renders only when at least one image has an `imageKey`.

## Cleanup folded into the build

The route audit flagged a stray scratch file: `app/[locale]/design/t.txt`, 24 lines, unreferenced. The web build brief deletes it.

The audited `topics.ts` example content is discarded. The build brief replaces the schema and may leave one or two example entries conforming to the new schema as a reference for content authors — but the audited example content does not survive.

## Execution plan

Phase 5 runs in this order:

1. **Web brief — rebuild the page and schema.** One session. The engineer replaces the `QaTopic` schema in `topics.ts` with the schema above, rebuilds `page.tsx` to render it, broadens the search corpus, deletes `t.txt`, and leaves one or two example topics conforming to the new schema. Definition of done includes lint, typecheck, and tests per conventions Part 4. No content authoring — example entries only.
2. **Docs/QA briefs — author topic content, page by page.** After the schema exists. One brief per page or per batch of pages — granularity decided after the web brief lands. Each brief gives Docs/QA the topic(s) to write, the schema to write against, and the facts available. Docs/QA writes entries into `topics.ts`. Page-by-page — the loop lives here.
3. **Review.** Each session — web and docs — comes back through Mastermind for verdict before the next brief runs.

Content authoring needs source facts. Docs/QA does not invent behavior. Where a topic needs a fact about how a page actually works, that fact comes from Igor, from the existing audit, or from a targeted question — not from guessing. Briefs in step 2 name what facts the author has and what to do when a fact is missing.
