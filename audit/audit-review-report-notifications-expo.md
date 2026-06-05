# Audit — /owner/reviews screen + review/report notification navigation

**Repo:** `oglasino-expo` · **Branch:** `new-expo-dev` · **Date:** 2026-06-02
**Type:** Read-only audit. No code changed. Code is ground truth.

**Scope (as briefed):** ONLY `/owner/reviews` (post route-flatten) + review/report notification
navigation. Explicitly NOT audited: the flicker fix, token plumbing, MESSAGE removal.

---

## TL;DR

- The reviews screen exists post-flatten at `app/owner/reviews.tsx`, route path **`/owner/reviews`**.
- It has **GIVEN / RECEIVED** tabs, but the active tab is **client/local `useState` only — NOT
  URL-addressable**. A notification can therefore land on `/owner/reviews` but **cannot target a
  specific tab**; it always opens on the default tab (**GIVEN**).
- There is **no review-specific notification handling**. A review notification rides the **generic
  `NAVIGATION` category** with `data.navigate`. End-to-end it resolves to `router.push(data.navigate)`
  with **no locale handling**.
- **No user-facing reports screen exists** (confirmed). The only "report" surfaces are the
  report-*submission* `ReportButton`/`ReportDialog` on a received-review card. Reports remain admin-only.
- **In-app render guard gap:** the `NAVIGATION` action button is gated **only on `categoryId`, not on
  `data?.navigate`**. A `NAVIGATION` notification with no `navigate` renders a live button that calls
  `router.push(undefined)`. (The push-tap handler *does* guard; the in-app screen does not.)
- Enum confirmed: `NotificationCategoryId` = `MESSAGE | SAVED_PRODUCT | PRODUCT_EXPIRATION |
  PRODUCT_EXPIRED | NAVIGATION | INFO`. The iOS category registration still uses the literal
  **`'NAVIGATE'`** (mismatch with the handled `'NAVIGATION'`) and registers **no `INFO`** category.

---

## 1. THE `/owner/reviews` SCREEN (post-flatten)

### Existence + exact route path
- File: **`app/owner/reviews.tsx`**, default export `ReviewsPage` (`reviews.tsx:9`).
- Route segment layout: `app/owner/` is a **top-level segment** (not wrapped in a route group).
  Confirmed app tree: `app/_layout.tsx` → `app/owner/_layout.tsx` (a `<Stack>`), and the only other
  top-level group is `(portal)`. So the route path is exactly **`/owner/reviews`**.
- Cross-checked against `src/lib/navigation/dashboardNavigations.tsx:51-52` which lists this screen as
  `url: '/owner/reviews'`. The flatten moved it from `app/owner/dashboard/reviews.tsx` →
  `app/owner/reviews.tsx` (matches the git status rename `R app/owner/dashboard/reviews.tsx ->
  app/owner/reviews.tsx`).

### GIVEN vs RECEIVED tabs — exact mechanism
**Two tabs, selected by local component state only. NOT a route param, NOT a query param.**

Verbatim (`reviews.tsx:10`, `:16-30`):
```tsx
const [activeTab, setActiveTab] = useState<OwnerReviewTabType>(OwnerReviewTabType.GIVEN);
...
<Pressable onPress={() => setActiveTab(OwnerReviewTabType.GIVEN)} ...>
  <Text>{tDash('reviews.given.label')}</Text>
</Pressable>
<Pressable onPress={() => setActiveTab(OwnerReviewTabType.RECEIVED)} ...>
  <Text>{tDash('reviews.received.label')}</Text>
</Pressable>
...
<OwnerReviewList type={activeTab} />
```
- `OwnerReviewTabType` (`src/lib/types/review/OwnerReviewTabType.ts`) = `{ GIVEN = 'GIVEN', RECEIVED =
  'RECEIVED' }`.
- Default tab on mount is **`GIVEN`**.
- The screen reads **no** `useLocalSearchParams` / `useGlobalSearchParams` and takes no route param.
  Tab state is pure `useState`, lost on unmount, not reflected in the URL.
- `OwnerReviewList` (`src/components/dashboard/components/OwnerReviewList.tsx`) takes `type` as a prop,
  resets and refetches via `getMyReviews(page, type)` on every `type` change, and renders
  `GivenReviewCard` vs `ReceivedReviewCard` accordingly.

**Implication for notifications:** a notification can deep-link to `/owner/reviews` **only**. There is
**no URL shape** like `/owner/reviews?tab=received` or `/owner/reviews/received` — `router.push` to any
such path would still mount `ReviewsPage`, ignore the unrecognized param, and default to **GIVEN**. A
"you received a review" notification therefore cannot land the user on the RECEIVED tab.

---

## 2. HOW A REVIEW NOTIFICATION IS ROUTED ON TAP

There is **no review-specific category or branch** anywhere. A review notification must travel as the
**generic `NAVIGATION`** category carrying `data.navigate`.

### Push-tap path (`PushNotificationsInit.tsx`)
- Foreground/background tap → `addNotificationResponseReceivedListener` → `handleResponse(response,
  false)` (`:131-137`).
- Cold-start tap → `getLastNotificationResponse()` → `handleResponse(response, true)` (`:139-149`),
  with the `LAST_HANDLED_RESPONSE_KEY = 'LHNR'` dedup (`:118-123`).
- `handleResponse` reads `content.categoryIdentifier` + `content.data` and calls `handleNavigation`
  (`:125-128`).
- `handleNavigation` `NAVIGATION` branch, verbatim (`:89-93`):
  ```tsx
  case 'NAVIGATION':
    if (data?.navigate) {
      router.push(data.navigate as RelativePathString);
    }
    break;
  ```
  → **guarded** on `data?.navigate`; if present, `router.push(data.navigate)` raw. **No locale
  handling, no transform.** A payload `data.navigate = "/owner/reviews"` pushes exactly `/owner/reviews`.

### In-app notifications screen path (`app/(portal)/(secured)/notifications.tsx`)
- `NAVIGATION` action button, verbatim (`:87-93`):
  ```tsx
  {item.categoryId === NotificationCategoryId.NAVIGATION && (
    <View>
      <Button variant="outline" onPress={() => router.push(item.data.navigate)}>
        <Text>{tNotifications('notifications.see.notification')}</Text>
      </Button>
    </View>
  )}
  ```
  → on tap, `router.push(item.data.navigate)` raw. **No locale handling.** A `data.navigate =
  "/owner/reviews"` pushes exactly `/owner/reviews`.

### End-to-end for `data.navigate = "/owner/reviews..."`
- Both paths call `router.push("/owner/reviews")` (in-app) / `router.push(data.navigate)` (push-tap).
- Lands on `ReviewsPage`, default **GIVEN** tab. Any tab hint appended to the path/query is inert
  (§1). No locale prefix is applied on mobile (expo-router routes here are not locale-segmented).

**Generic vs review-specific:** **generic `NAVIGATION` handling only.** There is no `REVIEW` enum
member, no review branch in `handleNavigation`, and no review branch in the in-app `renderItem`.

---

## 3. REPORTS — NO user-facing reports screen (confirmed)

- `find app src -iname "*report*"` returns only report-*submission* and type plumbing:
  `ReportButton.tsx`, `ReportDialog.tsx`, `reportService.ts`, `reportOptionTranslations.ts`, and the
  `src/lib/types/report/*` + `ReviewReportOption.ts` DTOs/enums.
- The only mount of a report surface in the owner/user-facing tree is **`ReportButton`** inside
  `src/components/dashboard/components/ReceivedReviewCard.tsx:1,17` — i.e. *file a report about a
  received review*, opening `ReportDialog`. That is report **creation**, not a reports list/inbox.
- No route file under `app/` named or routing to "reports"; `grep -rin "report" app/` matches nothing
  beyond the submission components. **No reports screen exists** in the owner/user app.

**Conclusion:** matches Igor's "reports are admin-only." No expo-side reports screen to deep-link to; a
report notification has nowhere review-distinct to land and would, like reviews, ride generic
`NAVIGATION`/`data.navigate` (or fall to the `default` no-op if sent under an unhandled category).

---

## 4. INFORMATIONAL (no-navigate) RENDER BEHAVIOR

### In-app screen: what renders for NAVIGATION-without-navigate / INFO
- **`categoryId = NAVIGATION` but no `data.navigate`:** the button **IS still rendered** — the guard is
  on `categoryId` only, **not** on `data?.navigate` (`notifications.tsx:87`). `onPress` then runs
  `router.push(item.data.navigate)` → `router.push(undefined)`. This is a **dead / error button**:
  it shows the "see notification" CTA but navigates to `undefined`. **No guard.**
  - (For contrast, the push-tap path *does* guard: `if (data?.navigate)` at `PushNotificationsInit.tsx:90`.
    The in-app screen lacks the equivalent guard.)
  - The same unguarded pattern applies to `SAVED_PRODUCT` (`notifications.tsx:77-85`): button renders on
    `categoryId` alone and `onPress` reads `item.data.productId` unguarded.
- **`categoryId = INFO`:** there is **no render branch** for `INFO`. The card renders title +
  description + timestamp + colored left border, and **no action button** (`renderItem` only emits a
  button for `SAVED_PRODUCT` and `NAVIGATION`). So `INFO` (and any non-`SAVED_PRODUCT`/non-`NAVIGATION`
  category) renders cleanly as an informational card with no dead button.

So the dead-button risk is specific to **`NAVIGATION` (and `SAVED_PRODUCT`) notifications that arrive
without their expected `data` sub-field** — not to `INFO`.

### `NotificationCategoryId` enum — verbatim (`firebaseNotifications.ts:23-30`)
```ts
export enum NotificationCategoryId {
  MESSAGE = 'MESSAGE',
  SAVED_PRODUCT = 'SAVED_PRODUCT',
  PRODUCT_EXPIRATION = 'PRODUCT_EXPIRATION',
  PRODUCT_EXPIRED = 'PRODUCT_EXPIRED',
  NAVIGATION = 'NAVIGATION',
  INFO = 'INFO',
}
```
- `INFO` is present (confirms the prior audit). No `REVIEW`, no `REPORT`, no `FOLLOW` member.

### iOS category registration — current state (`PushNotificationsInit.tsx:22-44`)
- Registered categories: **`'SAVED_PRODUCT'`** (`:22`), **`'NAVIGATE'`** (`:30`), **`'MESSAGE'`**
  (`:38`). Plus the Android `'default'` channel (`:46-49`).
- **NAVIGATE-vs-NAVIGATION mismatch CONFIRMED, still present.** The iOS category literal is `'NAVIGATE'`
  (`:30`), but the enum, the `handleNavigation` switch (`:89`), and the in-app render (`notifications.tsx:87`)
  all use `'NAVIGATION'`. A push whose `categoryIdentifier = 'NAVIGATION'` (to be handled) will **not**
  match the registered `'NAVIGATE'` iOS category, so its OS-level action button is not the one
  registered; and a push sent as `'NAVIGATE'` falls to `handleNavigation`'s `default` (no-op).
- **No `INFO` iOS category** is registered (consistent with `INFO` carrying no action button anyway).
  No category registered for `PRODUCT_EXPIRATION` / `PRODUCT_EXPIRED` either.

---

## Brief vs reality

No blocking contradiction with the brief — the brief framed this as discovery and the code confirms its
premises. Two facts worth surfacing for the standardization work:

1. **Review tab is not URL-addressable**
   - Brief asks whether a tab is URL-addressable so a notification can target it.
   - Code says: `app/owner/reviews.tsx:10` — tab is `useState`, no route/query param read anywhere in
     the screen.
   - Why it matters: standardizing `data.navigate` to `/owner/reviews?tab=received` (or similar) will
     **silently default to GIVEN** — the deep link cannot reach RECEIVED without a screen change to read
     a param. If "received-review" notifications should land on RECEIVED, that is a (separate, out-of-scope)
     screen change.
   - Recommended resolution: standardize the path to `/owner/reviews` only, OR open a follow-up to make
     the tab read an initial value from a route/query param.

2. **In-app NAVIGATION/SAVED_PRODUCT buttons are unguarded on `data`**
   - Brief asks if the action button guards on `data?.navigate`.
   - Code says: `notifications.tsx:87` gates the button on `categoryId` only; `onPress` uses
     `item.data.navigate` unguarded (same for `SAVED_PRODUCT` `productId` at `:77-85`).
   - Why it matters: a standardized NAVIGATION notification missing `data.navigate` shows a live button
     that pushes `undefined`. The push-tap path already guards; the in-app screen should too.
   - Recommended resolution: add `&& item.data?.navigate` (resp. `&& item.data?.productId`) to the
     render guards. (Out of scope for this read-only audit — flagged for the standardization brief.)

---

## Config-file impact

Nothing. This is a read-only audit; it adopts no feature and removes no Expo backlog row. No edit to
`state.md`, `decisions.md`, `conventions.md`, or `issues.md` is implied by this session. The
NAVIGATE/NAVIGATION mismatch and the unguarded in-app buttons are already-known/now-confirmed and belong
to the upcoming standardization brief, not to a config-file change here.

## Cleanup performed

None needed — no code touched.

## Obsoleted by this session

Nothing.

## Conventions check

- Part 4 (cleanliness): N/A — no code changed.
- Part 4a (simplicity), Part 4b (adjacent observations): the two flagged items (non-URL-addressable tab;
  unguarded in-app action buttons) are adjacent observations recorded above for the standardization brief;
  not fixed here per the read-only scope.
- Part 11 (trust boundary): not in scope; see the prior `audit-notifications.md` push-token note.
