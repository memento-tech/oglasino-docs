# Session — 2026-06 bug-batch audit (read-only scoping)

**Repo:** oglasino-expo · **Branch:** new-expo-dev (no switch, no commit, no push)
**Date:** 2026-06-01 · **Slug:** bug-batch (order 2) · **Type:** read-only audit — **zero code changes**

## Task (one sentence)

Locate, shell-confirm, and size-classify eight reported expo UI/UX bug-batch items, writing the findings to `.agent/audit-expo-2026-06-bug-batch.md` — no code touched anywhere.

## Outcome

Deliverable written: **`.agent/audit-expo-2026-06-bug-batch.md`** (one section per item, in brief order, plus an adjacent-observations section and a summary table). All eight items located and verdicted. Every asserted file:line confirmed with `cat -n` / `grep -n`, not Read.

| # | Verdict | Key location | Size |
| --- | --- | --- | --- |
| 1 Facebook button | real — 2 provider buttons | `LoginOptionsDialog.tsx:57–63` (+icon, store action, service stub, test mock dead on removal) | small (teardown) |
| 2 cold-start → notifications | real; **not** tab config — boot push redirect | `PushNotificationsInit.tsx:99–100` `default → /notifications`, fired by stale `getLastNotificationResponse()` `:119` | small–structural |
| 3 toggle no feedback | refetch IS wired (`onRefreshCurrent`); gap = backend read-after-write lag | `DashboardProductFunctionsDialog.tsx:66/86` → `ProductList.tsx:261/176–232` | small (optimistic) |
| 4 search placeholder = full path | real; scope independent of placeholder text | `SearchInput.tsx:123` (text) vs `:89–94` (scope IDs) | trivial |
| 5 ProductCard raw enum | real; no existing keys (dropdown also raw) | `ProductCard.tsx:84,:88`; `FiltersDialog.tsx:198` | small (+keys) |
| 6 Preview reads `imageKeys` | real, guarded, no crash | `PreviewProductDialog.tsx:220` (should read `imagesData`) | small |
| 7 unused `getTranslation` | dead, zero callers | `PreviewProductDialog.tsx:64–68` | trivial |
| 8 config return type | real, type-only; caller `?? {}`-safe | `configurationService.tsx:6` vs `:15/:18`; `bootStore.ts:203–204` | trivial |

## Notable findings worth Mastermind's attention

- **Item 2 is not an initial-route fix.** There is no `initialRouteName`/`unstable_settings` anywhere; the tab navigator's natural first route is already home (`(portal)/_layout.tsx:27`). Notifications wins because boot-mounted `PushNotificationsInit` calls `router.push('/notifications')` via the `default` switch branch off a non-cleared `getLastNotificationResponse()`. Forcing home in tab config would do nothing.
- **Item 3 has the refresh already wired.** `onRequestProductRefresh` → `onRefreshCurrent` refetches all loaded pages after a toggle. If feedback is still delayed, the cause is list-endpoint read-after-write lag (the code comment at `ProductList.tsx:211` even acknowledges backend-consistency dependence). Cleanest client fix = optimistic row update. A single-product owner fetch exists (`getDashboardProductDetails`, `productsSearchService.ts:36`) but returns `UpdateProductRequestDTO`, not the list-row DTO, so a single-row local refetch needs shape adaptation.
- **Item 5 needs new keys.** No translation key set exists for `ProductState`/`ModerationState` on the client; the dashboard state dropdown also renders the raw enum (`FiltersDialog.tsx:198`). Translations are backend-seeded, so new keys must be seeded backend-side — nothing to reuse locally.

## Brief vs reality

None blocking. Two reframings (documented in the audit, not contradictions of the brief):

1. **Item 2 framing** — Brief asks whether forcing home is a one-line initial-route change. Code says: it isn't an initial-route issue at all; it's the notification boot-redirect's `default` branch + a stale last-response. Recommended resolution captured in the audit (dedupe the handled response / narrow the `default` push).
2. **Item 3 framing** — Brief frames it as "no refresh." Code says: a full loaded-pages refetch is already wired to the toggle; the residual symptom, if real, is backend freshness. Optimistic update is the client-only path; single-row refetch would need DTO adaptation. Both forks surfaced.

## Tool-reliability note

`rg`/`grep` **content** output mangled certain literal words this session (e.g. "Facebook"→"n", "placeholder"→"n", "FreeZone…"→"ng…"). Line numbers and file paths from grep were reliable; literal content was always re-confirmed via `cat -n` (accurate). No fabricated-Read incident — I used `cat -n`/`grep -n` throughout per the brief, never asserting from Read. Flagging the grep-content mangling as a new flavor for the standing tool-output-reliability watch (decisions.md log).

## Concurrent working-tree activity (flag for Igor)

The working tree changed **during** this read-only session — files not in the session-start snapshot became modified with timestamps inside my session window (`package.json`/`package-lock.json` 18:03, `app/(portal)/_layout.tsx` 18:09), plus new analytics artifacts (`src/components/analytics/`, `src/lib/analytics/`, `src/components/init/AnalyticsInit.tsx`, `src/lib/store/authStore.test.ts`), a deletion (`src/lib/client/firebaseAnalytics.ts`), and edits to `AppInit.tsx`/`authStore.ts`/`authService.test.ts`. **I made none of these** — my session was `cat`/`grep`/`find` reads plus the two `.agent/` writes only. This is consistent with concurrent Google-Analytics-v1 mobile-adoption work in the same checkout. Surfaced because: (a) it means another agent/process shares this tree, and (b) it shifted one of my citations — `AppInit.tsx` `PushNotificationsInit` moved 26→28 after an `AnalyticsInit` insert; the audit was corrected to `:28`. All other audited files were either unaffected or read at their current on-disk state. **No code conflict with this audit (read-only).**

## Cleanup performed

None needed — read-only audit, no code modified.

## Obsoleted by this session

Nothing. No code obsoleted (no edits). The audit is input for a future implementation brief.

## Conventions check

- **Part 4 (cleanliness):** N/A — no code written. No TODO/FIXME/console.log added. (Flagged, did not fix: pre-existing commented-out FB SDK block at `authService.ts:216–240` — audit Adjacent #3.)
- **Part 4a (simplicity):** audit recommends the simplest viable fix per item (e.g. optimistic update over a new endpoint for #3; annotation-only change for #8).
- **Part 4b (adjacent observations):** four logged in the audit (DeleteAccount FB branch, dashboard raw-enum dropdown, commented FB SDK block, `ProductReviewDialog` `imageKeys: []`) — all marked out of scope, none fixed.
- **DoD lint/tsc/test:** not run — no touched code paths this session (read-only). Standing `new-expo-dev` lint baseline remains 84 warnings per the Risk Watch note; unchanged by this session.

## Config-file impact

No `state.md` / `issues.md` / `conventions.md` / `decisions.md` writes (I do not write these). For Docs/QA to consider (drafts, not applied):

1. **Optional `issues.md` rows** for the eight audited items if Mastermind wants them tracked before an implementation brief — but the audit file itself is the durable record, so this is optional.
2. **`decisions.md` tool-reliability log:** add the grep/rg **content-mangling** observation (distinct from the prior fabricated-Read and brief-swap flavors) to the standing tool-output-reliability watch.

No Expo-backlog row is adopted or removed by this session (audit only, nothing shipped). **Closure gate:** no implicit config-file dependency beyond the optional drafts above — stated explicitly.

## For Mastermind

- Audit ready at `.agent/audit-expo-2026-06-bug-batch.md`. Recommended implementation grouping when a brief is written: trivial cluster (#4, #7, #8) as one quick pass; #5 + Adjacent #2 together (shared enum-key work, needs backend-seeded keys — coordinate with Backend/Docs on key names); #6 standalone small; #1 as a full FB teardown (sweeps Adjacent #1 + #3); #2 as the one genuinely structural item (notification boot-redirect dedupe — decide the fork before coding).
- Pre-req call-outs: #5 needs new backend-seeded translation keys (cross-repo); #3 fork (optimistic vs refetch) and #2 fork (dedupe vs narrow default) are owner decisions.
