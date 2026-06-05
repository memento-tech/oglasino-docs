# Session — review/report notification audit (read-only)

**Repo:** `oglasino-expo` · **Branch:** `new-expo-dev` · **Date:** 2026-06-02
**Slug:** `review-report-notifications` · **Order:** 1
**Type:** Read-only audit per `.agent/brief.md`. No code changed.

## Task (one sentence)
Audit the post-flatten `/owner/reviews` screen shape and how a review/report notification routes on
expo, and confirm there is no user-facing reports screen — findings written to
`.agent/audit-review-report-notifications.md`.

## Deliverable
- `.agent/audit-review-report-notifications.md` — the full audit (route path, verbatim tab mechanism,
  verbatim render-guard logic, enum + iOS category state, reports-screen confirmation).

## Key findings (summary — see audit for detail + file:line)
1. Screen at `app/owner/reviews.tsx`, route path **`/owner/reviews`** (owner is a top-level segment).
2. GIVEN/RECEIVED tabs are **local `useState` only — not URL-addressable**. A notification can land on
   `/owner/reviews` but not on a specific tab; defaults to **GIVEN**.
3. **No review-specific notification handling** — review notifications ride generic `NAVIGATION` +
   `data.navigate`; both push-tap and in-app resolve to `router.push(data.navigate)` with **no locale
   handling**.
4. **No user-facing reports screen** — only report-*submission* `ReportButton`/`ReportDialog` on the
   received-review card. Reports remain admin-only (matches Igor).
5. **In-app render-guard gap:** `NAVIGATION` (and `SAVED_PRODUCT`) action buttons gate on `categoryId`
   only, not on `data?.navigate`/`data?.productId` → a missing-`data` notification renders a button that
   pushes `undefined`. Push-tap path is guarded; in-app screen is not.
6. Enum `NotificationCategoryId` confirmed verbatim (incl. `INFO`); iOS category registration still uses
   literal **`'NAVIGATE'`** (mismatch with handled `'NAVIGATION'`), no `INFO` category registered.

## Brief vs reality
No blocking contradiction. Two facts surfaced for the standardization brief (both detailed in the audit):
non-URL-addressable review tab; unguarded in-app NAVIGATION/SAVED_PRODUCT buttons. Both out of scope to
fix in this read-only audit.

## For Mastermind
- If "received-review" notifications should land on the RECEIVED tab, the path `/owner/reviews` alone
  cannot achieve it — needs a follow-up screen change to read an initial tab from a route/query param.
  Recommend standardizing `data.navigate` to plain `/owner/reviews` for now.
- Consider a small follow-up (separate brief) to add `&& item.data?.navigate` / `&& item.data?.productId`
  guards to the in-app notification action buttons, mirroring the push-tap guards.

## Cleanup performed
None needed — no code touched.

## Obsoleted by this session
Nothing.

## Conventions check
- Part 4 (cleanliness): N/A — no code changed.
- Part 4a / 4b: adjacent observations (non-URL tab; unguarded buttons) recorded for the standardization
  brief, not fixed here per read-only scope.
- Hard rules honored: no commits/pushes, no cross-repo edits, no config-file writes, no app.json/eas edits.

## Config-file impact
Nothing. Read-only audit; adopts no feature, removes no Expo backlog row. No edit to `state.md`,
`decisions.md`, `conventions.md`, or `issues.md` is implied.
