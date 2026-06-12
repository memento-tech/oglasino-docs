# Audit — Mobile products empty-state(s)

**Repo:** oglasino-expo
**Branch:** dev
**Type:** Phase-2 audit, read-only
**Scope:** Brief Part 2 — what the "no products" display actually renders. Per Igor's
in-session note, this covers **both** the public products surfaces (home + catalog) **and**
the owner/dashboard products page — not only the public one.

> Context: these empty states were just built in session
> `oglasino-expo-empty-states-1` (2026-06-12). This audit documents the resulting current
> behavior for the web-vs-mobile comparison Mastermind is running.

---

## How the empty branch is reached (shared mechanism)

All four surfaces render through the same component chain:

`<screen>` → `FilteredProductList` (`src/components/product/FilteredProductList.tsx`)
→ `ProductList` (`src/components/product/ProductList.tsx`).

`ProductList` renders the screen-supplied `NoProductsComponent` inside its
**`ListHeaderComponent`**, gated only by:

```tsx
{(!products || products.length === 0) && NoProductsComponent && (...render...)}
```

(`ProductList.tsx:353–355`). There is **no loading/`refreshing` guard** on this — see the
adjacent observation at the end. `ProductList` returns nothing until `bootStatus === 'ready'`
(`ProductList.tsx:328`). The total-count line (`tCommon('total.number.of.products')`) shows
only once `totalNumberOfProducts >= 0`.

Each screen decides "genuinely empty vs filtered-empty" itself, via an inline
`anyFilterActive` selector over its filter store, and passes the right copy as
`NoProdctsComponent` (note the prop is spelled `NoProdctsComponent` on `FilteredProductList`,
`NoProductsComponent` on `ProductList`).

---

## A. Owner / dashboard products — `app/owner/products/index.tsx`

Filter store: `useDashboardFilterStore`. Card: `DashboardProductCard`. `applyRandom={false}`,
`addFooter={false}`.

`anyFilterActive` is true if **any** of: `selectedFilters`, `selectedProductStates`,
`selectedModerationStates` non-empty; `selectedOrder !== undefined`;
`selectedPriceRange.free`; price `from`/`to` non-empty; `searchText` non-empty; regions or
cities selected. (The dashboard set additionally counts `productStates` + `moderationStates`,
which the public set does not.)

| Branch | Renders | Keys (namespace) | Button |
| --- | --- | --- | --- |
| **Genuinely empty** (no filters) | bold title + body | `empty.products.title` + `empty.products.body` (**DASHBOARD_PAGES**) | `AddProductButton` |
| **Filtered/search empty** | single centered line | `products.filters.empty.list` (**DASHBOARD_PAGES**) | `AddProductButton` |

`AddProductButton` shows in **both** branches. Container:
`min-h-60 items-center justify-center gap-2 px-4`.

---

## B. Home feed (public) — `app/(portal)/(public)/index.tsx`

Filter store: `usePortalFilterStore`. Card: `PortalProductCard`. `applyRandom={true}`.

**Only one branch — no filtered-empty distinction.** Home always renders:

| Branch | Renders | Keys (namespace) | Button |
| --- | --- | --- | --- |
| Empty (always, even with filters/search active) | bold title + body | `empty.home.title` + `empty.home.body` (**COMMON**) | `AddProductButton` |

Container: `min-h-60 items-center justify-center gap-2 px-4`. Home has a
`FloatingFiltersButton` and a search field, so a filtered/searched-to-zero home still shows
the "be the first" incentive copy (see inconsistency note below).

---

## C. Catalog / category (public) — `app/(portal)/(public)/catalog/[...categories].tsx`

Filter store: `usePortalFilterStore`. Card: `PortalProductCard`. `applyRandom={true}`.
Header: `BackButton`.

`anyFilterActive` (public set): `selectedFilters` non-empty, `selectedOrder !== undefined`,
price `from`/`to` non-empty, regions or cities selected. (Does **not** count
`searchText`, `selectedPriceRange.free`, product/moderation states.)

| Branch | Renders | Keys (namespace) | Button |
| --- | --- | --- | --- |
| **Genuinely empty** (no filters) | "not found for <category>" line + incentive line | `navigation.search.not.found` `{value: categoryLabel}` (**HEADER**) + `empty.category.incentive` (**COMMON**) | `AddProductButton` |
| **Filtered empty** | single centered line | `products.filters.empty.list` (**DASHBOARD_PAGES**) | none |

`AddProductButton` shows **only in the genuinely-empty branch**, not the filtered branch.
`categoryLabel` is the deepest selected category's `labelKey` resolved via `COMMON_SYSTEM`.
Container: `flex min-h-60 items-center justify-center gap-2 px-4`.

---

## Button behavior — `AddProductButton` (`src/components/product/AddProductButton.tsx`)

The same shared, **auth-aware** CTA in every branch that shows a button.

- Label: `add.new.product.label` (**BUTTONS**) — "Create Product".
- No-op until `authStore._hasHydrated`.
- **Logged out:** `openDialog(LOGIN_OPTIONS_DIALOG, { dialogDescription:
  tDialog('login.options.create.product.description') })`.
- **Logged in:** `openProductCreateGate(user, openDialog)` — same gate the BottomBar FAB and
  dashboard sidebar use (setup-dialog vs create-dialog routing).
- It is a plain labeled `<Button>` (text only, no icon), `className="mt-4"`.

---

## Summary table (every no-products branch on mobile)

| Surface | Scenario | Copy keys | Button? |
| --- | --- | --- | --- |
| Dashboard | empty, no filters | `empty.products.title`+`empty.products.body` (DASHBOARD_PAGES) | yes |
| Dashboard | filtered empty | `products.filters.empty.list` (DASHBOARD_PAGES) | yes |
| Home | empty (any case) | `empty.home.title`+`empty.home.body` (COMMON) | yes |
| Catalog | empty, no filters | `navigation.search.not.found`(HEADER)+`empty.category.incentive`(COMMON) | yes |
| Catalog | filtered empty | `products.filters.empty.list` (DASHBOARD_PAGES) | no |

---

## What differs from web (for Mastermind's comparison) — flags, not fixes

1. **Web had not adopted these keys at empty-states build time.** The prior session
   (`empty-states-1`, "For Mastermind") recorded that on `oglasino-web` `dev` these keys
   are not consumed anywhere — web's dashboard still rendered the older
   `products.empty.list` / `products.filters.empty.list`, and web home/catalog had no such
   empty state. Mobile built against the **backend-seeded** keys (the stable contract), not
   against a shipped web reference. Web parallel adoption may still be pending. **Severity:
   low.**
2. **Home has no filtered-empty branch.** Catalog and dashboard distinguish
   genuinely-empty vs filtered-empty; home always shows the "be the first" incentive even
   when a search/filter is what emptied it. If web's home distinguishes the two, this is a
   copy mismatch. **Severity: low–medium** (could show misleading "be the first" copy to a
   user who just over-filtered). Out of scope — flagged only.
3. **Catalog filtered-empty has no button; dashboard filtered-empty does.** Intra-mobile
   inconsistency in whether the CTA shows in the filtered branch. Worth confirming against
   web's per-branch button presence. **Severity: low.**
4. **Catalog's filtered-empty copy uses a DASHBOARD_PAGES key**
   (`products.filters.empty.list`) on a public page. Functionally fine (mobile fetches that
   namespace), but namespace-semantically odd for a public surface. **Severity: low.**
5. **No loading guard on the empty state** (see below) — could differ from web if web
   gates its empty state behind a loading/`isFetching` check.

### Adjacent observation (conventions Part 4b)

- **No loading/`refreshing` guard before showing the empty state.** `ProductList.tsx:353`
  renders `NoProductsComponent` whenever `products.length === 0`, including the window
  between mount (`products` initialized to `[]`, `bootStatus === 'ready'`) and the first
  page resolving. The empty-state copy + CTA can **flash** before real products arrive.
  File: `src/components/product/ProductList.tsx:353–355`. **Severity: medium** (user-visible
  flicker on every cold list load). Did not fix — out of audit scope.
