# Audit — bug-batch-web (read-only)

**Repo:** `oglasino-web` · **Branch:** `dev` · **Date:** 2026-06-04
**Mode:** READ-ONLY. No code changed. The strict-mode measurement (Q2) was taken via a `tsc --strict` CLI override, so `tsconfig.json` was never edited (confirmed clean: `git status --porcelain tsconfig.json` empty).

---

## Q1 (W7) — `isAllowedPath` anti-pattern: realistic fix size

### What the code actually is

`FilterManager.tsx` (`src/components/client/initializers/FilterManager.tsx`) is the filter engine: a render-null component with two effects — **HYDRATE** (URL → store, once per path, lines 77–225) and **SYNC** (store → URL, lines 230–355). It is parameterised by which store to drive (`useFilterStore` prop).

`isAllowedPath()` (lines 54–72) is a hard-coded string allowlist:

```
exact: '/'  and  '/owner/products'
startsWith: '/admin/products'
startsWith: '/catalog'
```

It is called as a guard at the top of both effects (lines 79, 232) so the engine no-ops on every other route.

**Why the allowlist exists:** `FilterManager` is mounted **broadly at the layout level**, not per page. It rides into the tree through `SelectableFilterManagerWrapper` (picks the store by `portalScope`), which is mounted in **three layouts**:

- `app/[locale]/(portal)/(public)/layout.tsx:11` → `portalScope="portal"` (wraps the whole public portal: home, catalog, product, user, about, …)
- `app/[locale]/owner/layout.tsx:30` → `portalScope="owner"` (wraps all `/owner/*`)
- `app/[locale]/admin/layout.tsx:29` → `portalScope="admin"` (wraps all `/admin/*`)

Because the mount point is a broad layout, the allowlist is the mechanism that narrows the engine back down to the specific list/search surfaces. That is the anti-pattern: mount-everywhere-then-string-filter.

### The decisive finding — the opt-in surface already exists

The filter **UI** components are **already mounted per-page**, on exactly the routes the allowlist permits. `SelectableFiltersWrapper` + `SelectableSelectedFiltersDisplayWrapper` appear in:

| Allowed path (isAllowedPath) | Page file that already renders the filter UI |
| --- | --- |
| `/` (exact) | `app/[locale]/(portal)/(public)/page.tsx:63,65` |
| `/catalog/*` (startsWith) | `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx:184,193` |
| `/owner/products` (exact) | `app/[locale]/owner/products/page.tsx:44,46` |
| `/admin/products` (startsWith) | `app/[locale]/admin/products/page.tsx:42,44` |
| `/admin/products/[userId]` (startsWith) | `app/[locale]/admin/products/[userId]/page.tsx:47,49` |

The allowlist set is **identical** to the set of pages that already host the filter UI. Routes the allowlist rejects (`/owner/products/[productId]`, `/admin/products/product/[productId]`, product/user/about pages) are exactly the pages that do **not** render the filter UI. The string allowlist is therefore redundant with an already-established page-level mounting pattern — it is duplicating, in string form, a decision the page tree already encodes structurally.

### The fix shape (per-route opt-in)

1. Delete `isAllowedPath()` and its two guard call-sites from `FilterManager.tsx` (the engine becomes "if I'm mounted, I sync").
2. Remove `<SelectableFilterManagerWrapper>` from the **3 layouts**.
3. Add `<SelectableFilterManagerWrapper portalScope=…>` to the **5 page files** above (alongside the filter UI already there). `SelectableFilterManagerWrapper.tsx` itself is unchanged.

### Files touched / call-site count

- **Edited:** `FilterManager.tsx` (delete ~19-line function + 2 internal guards).
- **Mounts removed:** 3 (the three layouts).
- **Mounts added:** 5 (the five page files).
- **Unchanged:** `SelectableFilterManagerWrapper.tsx`.
- **Total files touched: ~9.** Net call-site churn: −3 layout mounts, +5 page mounts, −1 function (−2 guards).

### Risk / hidden dependencies

- **Mount lifecycle changes from persistent to per-page.** At layout level the engine never unmounts while you stay in the layout; the `lastHydratedPathRef` (line 31) + `hydrated` store flag suppress re-hydrate loops. Page-level mounting means navigating between surfaces unmounts/remounts the engine — but the Zustand stores are module-level and persist, and a fresh mount re-hydrates from the current URL (the intended behaviour). Within the catalog catch-all (`/catalog/x` → `/catalog/y`) the page component does **not** remount; the existing `[baseSite, pathname]` effect dependency handles the param change exactly as today. So behaviour is preserved, but this is the one thing a smoke must confirm.
- **Shared portal store across `/` and `/catalog`.** Both use `usePortalFilterStore`; navigating between them remounts the engine but keeps store state, then re-hydrates from the new URL. Correct, but worth an eyeball.
- **Header search box is global** (`Header.tsx:25`, `SelectableSearchInputWrapper portalScope="portal"`) and is *not* gated by `isAllowedPath` today either — it writes `searchText` to the store on any portal page and the engine only syncs it where allowed. Page-level mounting reproduces this identically (engine absent off-allowlist ⇒ no sync), so no change in behaviour.

### Verdict

**Small / contained — not a feature-sized refactor.** The issue suspected feature-sized; the code does not bear that out, because the per-page opt-in surface (the filter UI mounts) already exists on precisely the allowlisted routes. The work is ~9 files of mechanical mount-relocation plus deleting one function. The real cost is **verification, not code**: a manual smoke of all five surfaces plus the language-switch / base-site-switch interactions (the engine's re-hydrate triggers), because the change alters mount lifecycle. Recommend it as a `chore/` or small `fix/` brief with a defined smoke checklist, not a feature.

---

## Q2 (W9) — `tsconfig` strict: error count under `strict: true`

Measured with `npx tsc --noEmit --strict` (CLI override; `tsconfig.json` not edited). Baseline `npx tsc --noEmit` (strict off, as committed) = **0 errors**.

- **TOTAL errors under full `strict`: 214**, across **72 files**.

### Breakdown by TS code (raw)

| Count | Code | Meaning |
| --- | --- | --- |
| 73 | TS2322 | Type X not assignable to type Y (63 of the 73 cite `null`/`undefined`) |
| 47 | TS18048 | 'X' is possibly 'undefined' |
| 37 | TS2345 | Argument not assignable to parameter (28 of the 37 cite `null`/`undefined`) |
| 33 | TS2339 | Property does not exist on type (stricter union narrowing) |
| 14 | TS18047 | 'X' is possibly 'null' |
| 4 | TS2538 | Type 'undefined' cannot be used as an index type |
| 2 | TS7006 | Parameter implicitly has 'any' type |
| 1 | TS2531 | Object is possibly 'null' |
| 1 | TS18046 | 'X' is of type 'unknown' |
| 1 | TS2488 | Type must have a `[Symbol.iterator]` |
| 1 | TS2366 | Function lacks ending return statement |

### Breakdown by category (rolled up)

- **Null/undefined safety (`strictNullChecks` family): ~157 of 214 (~73%).** = TS18048 (47) + TS18047 (14) + TS2531 (1) + TS2538 (4) + the 63 null/undefined-citing TS2322 + the 28 null/undefined-citing TS2345. This is the overwhelming majority; turning on `strictNullChecks` alone would account for nearly three-quarters of the total.
- **Genuine type mismatches (not null-driven): ~19.** = 10 remaining TS2322 + 9 remaining TS2345.
- **Property-does-not-exist (TS2339): 33.** Stricter narrowing of union/optional shapes.
- **Implicit-any / unknown: 3.** = TS7006 (2) + TS18046 (1).
- **Misc: 2.** = TS2488 (1) + TS2366 (1).

### Concentration

Top offenders (errors per file): `PreviewProductDialog.tsx` (11), `CreateNewProductDialog.tsx` (11), `owner/products/[productId]/page.tsx` (10), `admin/translations/TranslationsFilter.tsx` (9), `admin/products/product/[productId]/page.tsx` (9), `productService.ts` (8). The 214 errors are spread over 72 files, so there is a long tail.

Numbers only, per the brief. No remediation attempted.

---

## Q3 (W2) — notification `shown` field: declared on the web type? unrendered?

### Finding: `shown` is NOT declared on the web notification type at all.

The web notification shape is `AppNotification` (`src/notifications/types/AppNotification.ts`). Its full member list is:

```
id, title, description, seen, createdAt, type, categoryId, data?
```

There is **no `shown` member**. `git log -p -S"shown" -- src/notifications/types/AppNotification.ts` returns nothing — the field has never existed in this file's history.

A repo-wide grep for `shown` finds only:
- `notificationManager.ts` — a local `const shownIds = new Set<string>()` toast-dedup set (lines 10, 49, 51, 68). This is an unrelated in-memory dedup keyed on `notification.id`; it is **not** the backend `shown` wire field.
- prose occurrences ("…button is shown…") in comments and `app/[locale]/design/topics.ts`.

A grep for `shown` as a type member (`shown\s*[?:]`) across `src` and `app` returns **zero** matches.

### Brief vs reality

The brief's Q3 premise — "Confirm `shown` is declared on the web notification type and is currently unrendered" — does **not hold web-side**. The premise traces to the 2026-06-02 notifications carry-forward entry in `issues.md` ("…remains on the type as a documented backend-written wire field"), but that bullet describes the **backend/mobile** wire shape. On web, `shown` is neither declared nor referenced anywhere. The same entry's note that it "was already unrendered on web" is true but understated: it is not merely unrendered, it is **absent from the type**.

### Is it safe to remove the type member if the backend confirms it's dead?

**Web-side there is nothing to remove** — `AppNotification` already omits `shown`, no component renders it, no code reads it. The `issues.md` action item "drop it from … both clients' types" is therefore **already satisfied for web** (and was, before this audit). Whatever cleanup remains is purely on the **backend wire contract** and the **mobile** type — out of this repo. If the backend stops emitting `shown`, web requires **zero changes**: the Firestore notification doc carrying an extra unread field is silently ignored by `AppNotification`'s structural typing.

**Verdict:** Safe — trivially, because web carries no dependency on `shown` in the first place. The only live work is backend-side (stop writing it) and, if it exists there, the mobile type. Recommend the `issues.md` bullet be amended to reflect that the web half is a no-op.
