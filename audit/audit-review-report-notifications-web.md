# Audit — Review/Report Notification Routing (web)

**Repo:** oglasino-web
**Date:** 2026-06-02
**Type:** Read-only audit. No code changes.
**Scope:** Brief `audit-review-report-notifications` — ONLY the `/owner/reviews` route and how
review/report notifications are navigated. Token plumbing and the SW locale work (settled in
W1/W2) are NOT re-audited; W2's strip+reprefix SW behavior is taken as given and only confirmed
where it bears on review/report routing.

Method: code is ground truth. Every claim is anchored to `file:line`. Where the true source of a
behavior lives in another repo (backend), it is marked **[needs cross-repo verification]**.

---

## 1. The `/owner/reviews` route

### Route exists — file path
- **Route:** `/owner/reviews`
- **File:** `app/[locale]/owner/reviews/page.tsx` (default export `ReviewsPage`, `'use client'`).
- Renders `OwnerReviewList` (`src/components/owner/reviews/OwnerReviewList.tsx`) inside shadcn
  `<Tabs>`. Note: this route is **not** under a `(protected)` group — it sits directly under
  `app/[locale]/owner/`. (Auth gating for `/owner/*` is out of this brief's scope.)

### GIVEN vs RECEIVED tabs — exact mechanism
Yes, the page has two tabs, GIVEN and RECEIVED. The tab values come from an enum:

```ts
// src/lib/types/review/OwnerReviewTabType.ts
export enum OwnerReviewTabType {
  GIVEN = 'GIVEN',
  RECEIVED = 'RECEIVED',
}
```

The active tab is **client state only** — a `useState`, defaulted to `GIVEN`:

```tsx
// app/[locale]/owner/reviews/page.tsx:13
const [activeTab, setActiveTab] = useState<OwnerReviewTabType>(OwnerReviewTabType.GIVEN);
...
// :17-20  controlled shadcn Tabs
<Tabs
  value={activeTab}
  onValueChange={(v) => setActiveTab(v as OwnerReviewTabType)}
  className="w-full">
```

- There is **no query-param read** and **no path segment** for the tab. Grep for
  `searchParams` / `useSearchParams` in both `app/[locale]/owner/reviews/` and
  `src/components/owner/reviews/` returns nothing (exit 1, no matches). The only `tab` symbol on
  the page is the local `activeTab`/`setActiveTab` state.
- The tab is selected purely by the controlled `<Tabs value={activeTab}>` + `onValueChange`
  user click. It is **not URL-addressable**.

### Consequence for deep-linking
**A notification can only deep-link to `/owner/reviews` — it cannot select a specific tab.**
The page always mounts on the `GIVEN` tab (the `useState` default). A `data.navigate` of
`/owner/reviews?tab=received` (or `/owner/reviews/received`) would **not** select the RECEIVED
tab: the query string is carried through navigation (see §2) but nothing on the page reads it,
and there is no `/owner/reviews/[tab]` route segment. The user lands on GIVEN regardless and must
click the RECEIVED tab manually.

If the feature needs a review notification to land on a specific tab, that is a **code change to
make the tab URL-addressable** (read a `?tab=` searchParam into the initial `activeTab`, or add a
path segment) — it does not exist today.

---

## 2. How a review notification is routed on click

### It is generic NAVIGATION handling — no review-specific code
There is **no review-specific or report-specific branch** anywhere in the notification handlers.
Grep of `src/notifications/`, `public/firebase-messaging-sw.js`, and
`public/firebase-messaging-sw.template.js` for `review`/`report` returns nothing. A review
notification reaches the user as a generic `categoryId: 'NAVIGATION'` document/payload carrying
`data.navigate` (e.g. `"/owner/reviews"`). **[needs cross-repo verification — backend is the
source of `categoryId` and `data.navigate`; web has no visibility into what it actually emits.]**

### End-to-end for `data.navigate = "/owner/reviews"` across the three handlers

All three handlers strip a leading routing-locale segment, then a locale is (re-)added. The
backend contract (per the W2 SW comment, `firebase-messaging-sw.template.js:28-30`) is that
`data.navigate` is **locale-unprefixed**; the strip is the defensive same-path normalizer.

| Handler | File:line | What it does with `data.navigate` | Result |
|---|---|---|---|
| **In-app list tap** | `notificationActions.ts:22-28` | `router.push(stripRoutingLocale(data.navigate))` via the **next-intl wrapped router** (the page passes `useRouter()` from `next/navigation`, see note) | wrapped router re-adds current locale → `/{locale}/owner/reviews` |
| **Foreground push tap** | `ForegroundPushInit.tsx:44-49` | `url = stripRoutingLocale(data.navigate)`; `router.push(url)` via `@/src/i18n/navigation` wrapped router | `/{locale}/owner/reviews` |
| **Background push tap (SW)** | `firebase-messaging-sw.js` (generated) / `firebase-messaging-sw.template.js:93-110` | `path = stripRoutingLocale(data.navigate)`, then re-add the current locale read from an open window (`currentLocaleFromClients`), else open unprefixed | `/{locale}/owner/reviews`, or `/owner/reviews` if no window open (edge resolves default locale) |

Net: a review notification clicked from any of the three surfaces navigates to
`/{locale}/owner/reviews` and lands on the **GIVEN** tab (§1). The generated SW
(`public/firebase-messaging-sw.js`) matches the template — the W2 strip+reprefix code is present
(`stripRoutingLocale` fn + `currentLocaleFromClients` + the NAVIGATION case all confirmed in the
generated file).

Note on the in-app router: `notifications/page.tsx:8,14` imports `useRouter` from
`next/navigation` (the raw Next router) and passes it into `resolveNotificationAction`. The
in-app handler's comment and the `stripRoutingLocale` call assume a locale-re-prefixing wrapped
router. With the raw `next/navigation` router, `router.push('/owner/reviews')` pushes an
**unprefixed** path; the proxy/middleware then resolves the locale. This is a pre-existing W2
behavior on the in-app path (out of this brief's scope to fix), but worth flagging because it
means the in-app tap and the foreground/SW taps don't take an identical code path to the locale
prefix. It does **not** change the destination route (`/owner/reviews`) or the tab outcome.

### `stripRoutingLocale` preserves query strings
`stripRoutingLocale` (`src/lib/utils/stripRoutingLocale.ts`) only strips a leading segment that
is in `routing.locales`; it slices from the first slash onward, so any `?tab=...` query string on
`data.navigate` **survives** the strip and the navigation. But — per §1 — the page ignores it.
So even if the backend emitted `/owner/reviews?tab=received`, the query would arrive at the page
and do nothing.

---

## 3. Reports — no user-facing reports surface

Confirmed: **there is no user-facing (non-admin) reports route.**

- The only `report` route in the app is `app/[locale]/admin/reports/page.tsx` — under the
  `admin/` route group (server component, reads `searchParams`, renders `ReportsFilter` +
  `ReportsTable`).
- `find app -ipath '*report*' -type f` returns only the admin page. There is **no**
  `app/[locale]/owner/reports`, no `(portal)` reports page, nothing outside `admin/`.
- Grep for `owner/reports` across `src` and `app` returns nothing.

**Conclusion:** Igor's statement holds — reports are admin-only. There is no user-facing
destination a report-resolve notification could deep-link to. **The report-resolve notification
for users must be informational (no `data.navigate`).** If it were sent as `NAVIGATION` pointing
at an admin route, a normal user clicking it would hit an admin page they cannot use (and which
the admin layout/guard would presumably block).

---

## 4. Category render behavior for an informational (no-navigate) notification

### The in-app list has NO button — the whole card is the click target
There is no per-notification action button. The entire card is clickable, gated on whether an
action resolved (`notifications/page.tsx:45-51`):

```tsx
const action = resolveNotificationAction(n, router);

return (
  <div
    key={n.id}
    onClick={action}
    className={`${action && 'cursor-pointer hover:scale-101'} flex w-full flex-col rounded-lg border ${leftBorderColor} border-l-4 p-4 pt-1 transition hover:opacity-90 sm:border-l-12`}>
```

- If `action` is `undefined`: `onClick={undefined}` (no-op) and the `cursor-pointer hover:scale-101`
  classes are **not** applied. The card renders as a **clean, non-interactive informational card**
  (still has the hover:opacity-90 transition, which is on the static class — minor, see below).
- If `action` is a function: the card is clickable and shows the pointer cursor + hover scale.

### The guard depends entirely on what `resolveNotificationAction` returns

```ts
// src/notifications/lib/notificationActions.ts:10-34
switch (categoryId) {
  case 'SAVED_PRODUCT':
    return () => { const productId = data?.productId; if (!productId) return; router.push(...); };
  case 'NAVIGATION':
    return () => { const navigateTo = data?.navigate; if (!navigateTo) return; router.push(stripRoutingLocale(navigateTo)); };
  case 'PRODUCT_EXPIRATION':
  case 'PRODUCT_EXPIRED':
  default:
    return undefined;
}
```

This is the **key finding** for the "drop the navigate field" decision:

- **An unknown category, or `PRODUCT_EXPIRATION` / `PRODUCT_EXPIRED`, returns `undefined`** → clean
  button-less, non-interactive informational card. ✅
- **`NAVIGATION` returns a function _unconditionally_** — the `if (!navigateTo) return;` guard is
  **inside** the returned closure, not at resolution time. So a `NAVIGATION` notification with
  **no `data.navigate`** still resolves to a truthy function. The card therefore renders as
  **interactive** (`cursor-pointer hover:scale-101` applied, pointer cursor shown) but clicking it
  is a **no-op** (the inner guard returns early). It is a **dead-looking clickable card**, not a
  clean informational card, and not an error/crash.

### Answer to the decision being made
**Dropping the `navigate` field while keeping `categoryId: 'NAVIGATION'` does NOT cleanly yield a
button-less informational card on web.** It yields a card that looks clickable but does nothing.

To get a clean informational card on web with no interaction, the notification must carry a
`categoryId` that resolves to `undefined` in `resolveNotificationAction` — i.e. **not**
`NAVIGATION` and not `SAVED_PRODUCT`. Any value the switch doesn't case (including a new `INFO`)
falls to `default` → `undefined` → clean card. So the cleanest contract for an informational
report-resolve notification is a non-NAVIGATION category (e.g. an `INFO` category, or reuse one of
the expiration-style no-action categories), **not** `NAVIGATION`-minus-navigate.

If the intent is specifically to keep `NAVIGATION` but render no button when `navigate` is absent,
that requires a **one-line code change** to `resolveNotificationAction` (hoist the
`if (!navigateTo) return undefined;` guard out of the closure so the resolver itself returns
`undefined`). That change does not exist today. (Flagged for Mastermind — out of scope to apply in
this read-only audit.)

### NotificationCategoryId union — verbatim
```ts
// src/notifications/types/NotificationCategoryId.ts
export type NotificationCategoryId =
  | 'MESSAGE'
  | 'SAVED_PRODUCT'
  | 'PRODUCT_EXPIRATION'
  | 'PRODUCT_EXPIRED'
  | 'NAVIGATION';
```

**It does NOT include `INFO`.** If the backend sends `categoryId: 'INFO'`, it is not a member of
the TS union (a type-level mismatch only — there is no runtime Zod/schema validation of incoming
docs, per the prior `audit-notifications.md` §2/§5), and at runtime it would hit the `default`
branch of `resolveNotificationAction` → `undefined` → clean non-interactive card. So an `INFO`
informational notification would render cleanly on web **today** even though the union doesn't
list it — but the union would need `'INFO'` added for type correctness if the feature adopts it.

For completeness, `NotificationType` (the visual style axis, separate from category) is verbatim:
`'NORMAL' | 'SUCCESS' | 'DANGER' | 'WARNING'` (`types/NotificationType.ts`). An informational card
would presumably be `NORMAL` (blue left border, `notifications/page.tsx:41-42`) or `SUCCESS`.

---

## Summary table

| Question | Answer |
|---|---|
| `/owner/reviews` route exists? | Yes — `app/[locale]/owner/reviews/page.tsx` |
| GIVEN/RECEIVED tabs? | Yes — shadcn `<Tabs>`, `OwnerReviewTabType` enum |
| Tab URL-addressable? | **No** — `useState`, default `GIVEN`; no query/path/searchParams |
| Deep-link granularity | `/owner/reviews` only; always lands on GIVEN tab |
| Review notification routing | Generic `NAVIGATION` → strip locale → re-add locale → `/{locale}/owner/reviews` (all 3 handlers) |
| Review-specific handling? | None — generic NAVIGATION only |
| User-facing reports route? | **None** — `admin/reports` only |
| Report-resolve for users | Must be informational (no `navigate`) |
| Drop `navigate`, keep `NAVIGATION`? | **Does NOT** yield clean card — renders dead clickable card (guard is inside the closure) |
| Clean informational card | Use a non-NAVIGATION category (default → `undefined`) |
| Union includes `INFO`? | **No** — 5 members, no `INFO` |

---

## Adjacent observations (Part 4b — not in scope to fix here)

- **`NAVIGATION`-without-navigate renders a dead clickable card** (§4). `resolveNotificationAction`
  (`src/notifications/lib/notificationActions.ts:22-28`) returns a truthy closure for `NAVIGATION`
  even when `data.navigate` is absent; the card shows pointer/hover affordance but click is a
  no-op. Severity: **medium** (misleading affordance; bites directly if the report-resolve
  notification is modeled as `NAVIGATION`-minus-navigate). One-line fix available (hoist the guard
  to return `undefined`). Did not fix — out of scope (read-only audit).
- **Tab not URL-addressable** (§1). If the product wants review notifications to land on RECEIVED,
  the page needs a `?tab=` searchParam read or a path segment. Severity: **low/medium** (feature
  gap, not a bug). Did not fix — out of scope.
- **In-app tap uses the raw `next/navigation` router, not the wrapped next-intl router** (§2 note).
  `notifications/page.tsx:8` imports `useRouter` from `next/navigation`, but
  `notificationActions.ts` strips the locale as if a re-prefixing router will re-add it. Same
  destination route, but a different locale-resolution path than the foreground/SW handlers (which
  use the wrapped router). Severity: **low** (pre-existing W2 behavior; works via edge resolution).
  Did not fix — out of scope; flagging in case W2's intent was uniform wrapped-router usage.

---

## What I did not do
- Did not re-audit token plumbing or the SW locale mechanics beyond confirming the W2
  strip+reprefix code is present and bears on review routing (brief said W1/W2 are settled).
- Did not verify backend behavior — what the backend actually emits for `categoryId` /
  `data.navigate` on a review or report-resolve notification is **[needs cross-repo verification]**.
- No code changes.
