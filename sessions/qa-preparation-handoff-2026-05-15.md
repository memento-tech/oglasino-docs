# QA Preparation — Handoff Note (Chat 1 → Chat 2)

**Feature:** QA Preparation. Spec: `oglasino-docs/features/qa-preparation.md`.
**As of:** 2026-05-15. Chat 1 closes here; Chat 2 picks up from this note plus the committed files.

## What QA Preparation is

One in-app QA reference page at `/[locale]/design` in `oglasino-web`, content-driven from a TypeScript module (`app/[locale]/design/topics.ts`). Each `QaTopic` covers a page, feature, flow, or other QA-relevant thing. The schema, the renderer, and the page are built and approved. The work now is authoring topic content.

## Status: DONE

- **Web rebuild** — schema + renderer rebuilt against the spec. Approved.
- **`imageKey` rename** — `QaImage.url` → `imageKey` across type, renderer, spec. Approved.
- **12 portal page topics authored and approved**, all in `topics.ts`:
  home-page, catalog-page, pricing-page, about-page, free-zone-page, privacy-page, terms-page, messages-page, product-page, user-page, notifications-page, favorites-page.
- `topics.ts` `qaTopics` array holds 12 entries. No example topics remain (removed during the Catalog session).

## Status: REMAINING

- **10 owner-dashboard page topics** — not started. Routes:
  `/[locale]/owner`, `/owner/account-verification`, `/owner/analytics`, `/owner/balance`, `/owner/follows`, `/owner/not-ready`, `/owner/products`, `/owner/products/[productId]`, `/owner/reviews`, `/owner/user`.
  Per Igor: all 10 are in scope; several are intentionally unfinished and get thin topics (overview + maybe one section). `/owner/user` is the dashboard *account* page — explicitly distinct from the public `user-page` already authored. Likely runs as one or two batches.
- **Portal feature/flow topics** — not started. Author "as many as context allows" after the dashboard pages. The candidate list (below) needs triage first — not all candidates necessarily warrant standalone topics.

## Candidate feature/flow topics (need triage before briefing)

Spotted across the page-authoring sessions, none created:
- `filters` — feature. The filter sidebar; shared by home, catalog, owner dashboard, admin.
- `product-list-and-pagination` — feature. The shared product grid + pagination.
- `global-header-search` — feature. The header search input.
- `category-breadcrumbs` — feature. The breadcrumb chain on catalog and product pages.
- `catalog-menu-navigation` — flow. How a user reaches `/catalog/<slug>` from anywhere.
- `start-message-flow` — flow. Entry points on product page, user page, messages kebab.
- `follow-flow` — flow. Follow/unfollow across user page, product cards, messages.
- `favorite-a-product-flow` — flow. Favorite from product detail and product cards, lands on /favorites.
- `report-flow` — flow. Product-report and user-report dialogs.
- `cross-baseSite-redirect-flow` — flow. Foreign-tenant URL redirect. May not warrant a standalone topic.

Triage note: some of these may fold into each other or into page topics. Chat 2 should decide scope, not author all ten blindly.

## Standing process (carry into Chat 2)

- **Two-stop docs sessions.** Author reads code, drafts structural sections, proposes 2-3 options for QA-judgment sections. Igor decides. Author finalizes. Logged in `decisions.md` (2026-05-14, "two-stop propose-decide session").
- **Cross-repo write.** Docs/QA writes topic entries directly into `oglasino-web/app/[locale]/design/topics.ts` — explicit per-brief authorization under conventions Part 3, scoped to that one file.
- **Paste-back discipline.** Igor pastes only the new topic entry/entries + the session summary, not the whole `topics.ts`. The summary's diff line confirms the existing topics are untouched.
- **`issues.md` is a source.** Each docs brief tells the author to read `issues.md` for entries touching the page and fold relevant ones into pitfalls/checklist.
- **Bugs found while reading code go to `issues.md`**, not into topics. Every page session so far has surfaced 1-3. They are not QA Preparation work; this chat does not fix them.
- **Session numbering:** next Docs/QA session for this slug is `-8`. Use the real date (`2026-05-15` or later), not `2026-05-14`.

## Open threads carried forward

- **home-page `optionsControls` clarity fix** — the home topic doesn't say its advanced-filter set is "top-level only" vs catalog's category-level set. One-line fix, fold into a future docs brief touching that topic. (Surfaced in the Catalog session.)
- **Candidate-topic triage** — see above. First task of the feature/flow phase.
- **`issues.md` line-number backfill** — the two catalog-slug entries (committed early) say "exact locations not pinned"; the Catalog session later pinned them. Optional cleanup: add the line numbers to those two entries. Non-blocking.

## How Chat 2 opens

Igor opens a fresh Mastermind chat, pastes `conventions.md`, `mastermind-bootstrap.md`, `state.md`, `decisions.md`, and this handoff note. Names the feature: QA Preparation, continuing. The spec and `topics.ts` are the ground truth; this note is the orientation.