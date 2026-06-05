# Audit — Expo release readiness: admin removal scope

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-05-23
**Mode:** read-only — no code changes

---

## Section 1 — Entry points

### The button

The "go to admin" button is rendered in `src/components/user/UserMenu.tsx:132-138`. It is conditionally rendered via `{isAdmin && ...}` where `isAdmin` is local state set by `isAdminUser().then(setIsAdmin)` on mount (line 53-55). The button navigates to `/admin` via `router.push('/admin')` (line 135).

There is no other entry point into the admin surface from non-admin UI.

### The routes

12 route files under `app/admin/`:

| File path | Purpose |
|---|---|
| `app/admin/_layout.tsx` | Admin layout wrapper — validates admin via `isAdminUser()`, sets portal scope to `'admin'`, mounts `TopBar` with `useAdminFilterStore` and `AdminSidebar` |
| `app/admin/index.tsx` | Admin global product list — uses `useAdminFilterStore` and `getAdminProducts` |
| `app/admin/products/[userId].tsx` | Per-user product list — uses `useAdminFilterStore` and `getAdminProducts` |
| `app/admin/products/product/[productId].tsx` | Admin product detail view |
| `app/admin/users/index.tsx` | Admin users management list |
| `app/admin/users/[userId].tsx` | Admin user detail view — uses `EnableDisableButton` |
| `app/admin/reports.tsx` | Admin reports page |
| `app/admin/reviews.tsx` | Admin reviews page |
| `app/admin/statistics.tsx` | Admin statistics dashboard |
| `app/admin/suggestions.tsx` | Admin suggestions page |
| `app/admin/chats/[userId].tsx` | Admin chat list for a user |
| `app/admin/chats/messages/[...messagesData].tsx` | Admin chat messages view |

### Deep links / URL schemes

No admin deep-link or URL-scheme configuration exists. `app.config.ts` has no admin references. The admin surface is only reachable via in-app navigation from the `UserMenu` button.

### Navigation

`AdminSidebar` (`src/components/admin/AdminSidebar.tsx`) renders the admin navigation using `adminNavigations` from `src/lib/navigation/adminNavigations.tsx`. Six sidebar entries: Products (`/admin`), Users (`/admin/users`), Suggestions (`/admin/suggestions`), Reviews (`/admin/reviews`), Reports (`/admin/reports`), Statistics (`/admin/statistics`).

The sidebar is mounted in `app/admin/_layout.tsx:49` only — no presence in any non-admin layout.

---

## Section 2 — Admin-only code inventory

Every file that exists solely to serve the admin surface. "Imported from (non-admin)" column flags coupling investigated in Section 3.

### Route files (12 files)

| File path | What it does | Imported from (non-admin) |
|---|---|---|
| `app/admin/_layout.tsx` | Layout wrapper, admin gate, search bar, sidebar | none |
| `app/admin/index.tsx` | Admin global product list | none |
| `app/admin/products/[userId].tsx` | Per-user product list | none |
| `app/admin/products/product/[productId].tsx` | Product detail view | none |
| `app/admin/users/index.tsx` | Users management list | none |
| `app/admin/users/[userId].tsx` | User detail view with enable/disable | none |
| `app/admin/reports.tsx` | Reports management | none |
| `app/admin/reviews.tsx` | Reviews management | none |
| `app/admin/statistics.tsx` | Statistics dashboard | none |
| `app/admin/suggestions.tsx` | Suggestions management | none |
| `app/admin/chats/[userId].tsx` | Chat list for a user | none |
| `app/admin/chats/messages/[...messagesData].tsx` | Chat messages view | none |

### Component files (21 files)

| File path | What it does | Imported from (non-admin) |
|---|---|---|
| `src/components/admin/AdminSessionGuard.tsx` | Guards admin routes via `isAdminUser()` check | none |
| `src/components/admin/AdminSidebar.tsx` | Admin navigation sidebar | none |
| `src/components/admin/BackButton.tsx` | Back navigation button | none |
| `src/components/admin/FilterInput.tsx` | Filter text input | none |
| `src/components/admin/FilterToggle.tsx` | Filter toggle | none |
| `src/components/admin/FiltersPanel.tsx` | Panel for filter controls | none |
| `src/components/admin/PaginatedTable.tsx` | Generic paginated table | none |
| `src/components/admin/chat/Chat.tsx` | Chat display | none |
| `src/components/admin/chat/ChatsListing.tsx` | Chat listing | none |
| `src/components/admin/chat/MessagesList.tsx` | Messages list | none |
| `src/components/admin/products/AdminProductCard.tsx` | Admin product card | none |
| `src/components/admin/reports/ReportFilters.tsx` | Report filters | none |
| `src/components/admin/reports/ReportsTable.tsx` | Reports table | none |
| `src/components/admin/reviews.tsx/ReviewFilters.tsx` | Review filters | none |
| `src/components/admin/reviews.tsx/ReviewsTable.tsx` | Reviews table | none |
| `src/components/admin/stats/StatsPaginatedTable.tsx` | Stats paginated table | none |
| `src/components/admin/suggestions/SuggestionFilter.tsx` | Suggestion filter | none |
| `src/components/admin/suggestions/SuggestionsTable.tsx` | Suggestions table | none |
| `src/components/admin/users/EnableDisableButton.tsx` | User enable/disable button | none |
| `src/components/admin/users/EnableDisableIcon.tsx` | User enable/disable icon | none |
| `src/components/admin/users/UserFilters.tsx` | User filters | none |
| `src/components/admin/users/UsersTable.tsx` | Users table | none |

Note: the `reviews.tsx/` directory name contains a `.tsx` extension, which is unusual. It is a directory, not a file. This is a pre-existing naming quirk.

### Service files (7 admin-only files)

| File path | What it does | Imported from (non-admin) |
|---|---|---|
| `src/lib/services/admin/adminService.ts` | `isAdminUser()` — calls `GET /secure/admin` | `src/components/user/UserMenu.tsx` (coupling, Section 3) |
| `src/lib/services/admin/usersService.ts` | `getReactAdminUser`, `getReactAdminUsers`, `disableUser`, `enableUser` | none |
| `src/lib/services/admin/chatsService.ts` | `getReactAdminUserChats`, `getReactAdminChatMessages` | none |
| `src/lib/services/admin/reviewService.ts` | `getReactAdminReviews`, `approveDisapproveReview` | none |
| `src/lib/services/admin/reportsService.ts` | `getReactAdminReports`, `resolveReport` | none |
| `src/lib/services/admin/statsService.ts` | `getAdminStats()` | none |
| `src/lib/services/admin/suggestionsService.ts` | `getReactAdminSuggestions` | none |

### Type files (7 files)

| File path | What it does | Imported from (non-admin) |
|---|---|---|
| `src/lib/types/chat/admin/ChatData.ts` | Admin chat data type | none |
| `src/lib/types/chat/admin/ChatMessage.ts` | Admin chat message type | none |
| `src/lib/types/chat/admin/ChatMessagesResponse.ts` | Admin chat messages response type | none |
| `src/lib/types/chat/admin/ChatUser.ts` | Admin chat user type | none |
| `src/lib/types/chat/admin/ChatsDataResponse.ts` | Admin chats data response type | none |
| `src/lib/types/chat/admin/MessageGroup.ts` | Admin message group type | none |
| `src/lib/types/review/AdminReviewDTO.ts` | Admin review DTO type | none |

### Navigation config (1 file)

| File path | What it does | Imported from (non-admin) |
|---|---|---|
| `src/lib/navigation/adminNavigations.tsx` | Defines 6 admin nav entries using `NavItem` type | none |

### Tests

No admin-specific test files exist anywhere in the repo.

**Total admin-only files: 48**

---

## Section 3 — Coupling check

Every non-admin file that references admin code, or shared element used by both admin and non-admin code.

| # | Shared element | File path | Used by (admin) | Used by (non-admin) | Disposition |
|---|---|---|---|---|---|
| 1 | `isAdminUser` import | `src/components/user/UserMenu.tsx:3,39,53-55,132-138` | Admin dashboard button render gated by `isAdmin` state | Same file, all other menu items | **Split** — remove `isAdminUser` import, `isAdmin` state, the `useEffect` checking admin status, and the conditional `{isAdmin && ...}` block (6 lines of JSX + 4 lines of state/effect) |
| 2 | `PortalScope` type | `src/lib/types/ui/PortalScope.ts:1` | `'admin'` literal in the union type | `'dashboard'` and `'portal'` literals | **Split** — remove `'admin'` from the union, yielding `'dashboard' \| 'portal'` |
| 3 | `usePortalScope` store | `src/lib/store/usePortalScope.ts` | Admin layout calls `setPortalScope('admin')` | Dashboard/portal layouts set their scopes | **Move** — no file edit needed; once admin callers are deleted, the store has no admin reference |
| 4 | `useCardSizeStore` | `src/lib/store/useCardSizeStore.ts:9,22,31` | `adminSize` property; `case 'admin'` branch in `getSizeForPortalScope` | `portalSize`, `dashboardSize`, `hydrate`, `setSize` | **Split** — delete `adminSize` property (line 9), its default (line 22), and the `case 'admin'` branch (line 31). The `hydrate()` and `setSize()` methods already only persist `dashboardSize` and `portalSize`; `adminSize` is never persisted (lines 42-44, 67-68). No data migration needed. |
| 5 | `useAdminFilterStore` | `src/lib/store/useFilterStore.ts:205` | `useAdminFilterStore` export | `usePortalFilterStore`, `useDashboardFilterStore` exports | **Split** — delete the `useAdminFilterStore` export line. `createFilterStore` and the other two exports stay. |
| 6 | `getAdminProducts`, `getAdminProductDetails`, `getAdminAutocompleteSuggestions` | `src/lib/services/productsSearchService.ts:47-48,98-101,155-159` | Three admin-specific wrapper functions + `'admin'` branch in three `uriMap` objects (lines 17, 58, 113) | `getPortal*` and `getDashboard*` wrappers | **Split** — delete the three `getAdmin*` functions and the `admin:` entries from the three `uriMap` objects |
| 7 | `getReactAdminSuggestions` | `src/lib/services/suggestionsService.ts:7-30` | `getReactAdminSuggestions` function (calls `/secure/admin/suggestion`) | `suggestCategory` function (calls `/public/suggestion/category`) | **Split** — delete `getReactAdminSuggestions` from this file; `suggestCategory` stays. Note: a duplicate may exist at `src/lib/services/admin/suggestionsService.ts` — if so, that entire file deletes. |
| 8 | `FiltersDialog` admin branches | `src/components/dialog/dialogs/FiltersDialog.tsx:176,188` | Line 188: `currentPortalScope === 'admin'` gates ModerationState filter. Line 176: `currentPortalScope === 'admin' \|\| currentPortalScope === 'dashboard'` gates ProductState filter | ProductState filter for dashboard; all other filter UI | **Split** — line 188: delete the entire `{currentPortalScope === 'admin' && ...}` block (ModerationState filter, lines 188-198). Line 176: simplify to `currentPortalScope === 'dashboard'`. |
| 9 | `ProductCard` — `adminSize` dep | `src/components/product/ProductCard.tsx:26,35` | `adminSize` destructured from `useCardSizeStore`; included in `useEffect` deps | All card rendering logic | **Split** — remove `adminSize` from destructuring (line 26) and from deps array (line 35) after `useCardSizeStore` is cleaned |
| 10 | `SearchInput` — admin route map | `src/components/SearchInput.tsx:195` | `admin: '/admin'` in `toRouteMap` | `portal: '/'` and `dashboard: '/owner/dashboard/products'` entries | **Split** — delete the `admin: '/admin'` line. TypeScript will enforce completeness if `PortalScope` union is narrowed (coupling #2). |
| 11 | `DialogId` — admin entries | `src/components/dialog/dialogRegistry.ts:16-19` | 4 admin dialog IDs (`ADMIN_REVIEW_OVERVIEW_DIALOG`, `ADMIN_DISAPPROVE_REVIEW_DIALOG`, `ADMIN_REPORT_OVERVIEW_DIALOG`, `ADMIN_REPORT_RESOLVE_DIALOG`) | All non-admin dialog IDs | **Split** — delete lines 16-19 |
| 12 | `TranslationNamespace.ADMIN_PAGES` | `src/i18n/types.ts:44` | Used by 14 admin components for translation keys | `src/components/filters/SelectFilter.tsx:35` uses it for `filter.all.label` | **Split** — see coupling #13 |
| 13 | `SelectFilter` namespace reference | `src/components/filters/SelectFilter.tsx:35,44,100` | Not an admin component itself, but uses `ADMIN_PAGES` namespace for `filter.all.label` key | Used in `FiltersDialog` for dashboard `ProductState` filter (non-admin) | **Split** — change namespace from `ADMIN_PAGES` to `DASHBOARD_PAGES` or `COMMON_SYSTEM`. Requires verifying the `filter.all.label` key exists in the target namespace (backend translation seed check). After the namespace change, `ADMIN_PAGES` can be removed from the `TranslationNamespace` enum. |

---

## Role-gating mechanism

Admin access is gated by a single function: `isAdminUser()` in `src/lib/services/admin/adminService.ts`. It calls `GET /secure/admin` on the backend, which returns a boolean. Two consumers use this gate:

1. **Route-level gate:** `app/admin/_layout.tsx:24-31` — calls `isAdminUser()` on mount; if false, redirects to `/`. All admin routes are children of this layout, so this is the sole route-level gate.

2. **Menu-level gate:** `src/components/user/UserMenu.tsx:53-55` — calls `isAdminUser()` on mount; the admin dashboard button is conditionally rendered based on the result.

`AdminSessionGuard.tsx` exists as a component-level guard (`src/components/admin/AdminSessionGuard.tsx`) but appears to be unused — the layout-level gate in `_layout.tsx` handles the same function. This is either dead code or a belt-and-suspenders guard; either way, it deletes with the admin surface.

There is no client-side role stored in any Zustand store or auth state. The admin check is always a fresh backend call.

---

## Removal complexity estimate

**Estimated effort: 1 session (low-medium complexity)**

### Pure deletions (mechanical, ~75% of the work)

- 12 route files under `app/admin/` — delete entire directory
- 21 component files under `src/components/admin/` — delete entire directory
- 7 service files under `src/lib/services/admin/` — delete entire directory
- 7 type files: `src/lib/types/chat/admin/` directory + `AdminReviewDTO.ts`
- 1 navigation config: `src/lib/navigation/adminNavigations.tsx`

**Total: 48 files deleted outright**. No admin-only file is imported by non-admin code (except `adminService.ts` → `UserMenu.tsx`, which is a coupling edit not a deletion blocker).

### Coupling edits (surgical, ~25% of the work)

13 coupling cases across 11 files. Every edit is a deletion of an `if`/`case`/property/export/enum-entry — no structural redesign needed. The largest edit is `UserMenu.tsx` (~10 lines removed). Most are 1-3 line changes.

One coupling case (#13, `SelectFilter.tsx` namespace) requires checking whether `filter.all.label` exists in a non-admin translation namespace. If it doesn't, a backend translation seed is needed (one key, four locales — trivial but cross-repo).

### Post-deletion verification

- `npx tsc --noEmit` — TypeScript will catch any dangling imports
- `npm run lint` — ESLint will catch unused imports
- `npm test` — no admin tests exist, so no test changes expected
- Manual check: `UserMenu` renders without the admin button for non-admin users (no behavior change for the release build)

### Risk: zero

No non-admin functionality depends on any admin code. The admin surface is a fully self-contained feature with well-isolated coupling points. Removal cannot break any non-admin flow.

---

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit)
  - Considered and rejected: nothing
  - Simplified or removed: nothing

- **Adjacent observation (Part 4b):** `oglasino-dev-firebase-adminsdk-fbsvc-002e6b2f58.json` — a Firebase Admin SDK service account key file — exists at the repo root. This is NOT part of the admin UI surface and should NOT be deleted in admin removal. However, it appears to be a credential file committed to the repo. Severity: medium. I did not investigate further because it is outside this audit's scope (the general project health audit covers credential management).

- **Adjacent observation (Part 4b):** `src/components/admin/reviews.tsx/` — a directory named with a `.tsx` extension. Unusual naming; the file explorer may confuse it with a file. Severity: low. Moot after admin removal (entire directory deletes).

- **Adjacent observation (Part 4b):** `SelectFilter.tsx` uses the `ADMIN_PAGES` translation namespace for a key (`filter.all.label`) that is semantically generic. The key predates this audit. After admin removal, the namespace reference must change, which requires a backend translation seed. Not blocked by admin removal — the removal brief should include this one-key seed in its scope. Severity: low.

- **Config-file impact:** none. This audit produces no code changes and no config-file edits.

- **Expo backlog:** no change. Admin removal is not a feature adoption — it's a pre-release cleanup. No row to add or remove.
