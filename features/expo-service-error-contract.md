# Mobile Service Layer + Error Contract Foundation

**Slug:** `expo-service-error-contract`
**Status:** shipped (code) / verifying (on-device airplane-mode check deferred to the Ψ chat)
**Repo:** oglasino-expo only
**Branch:** new-expo-dev
**Program:** Expo structural foundation, last of the four foundation chats. Blocks chat A (mobile validation rebuild).

## 1. Goal

The mobile service layer throws away backend errors. Roughly 14 of 16 services catch an error, log it, and return a sentinel (`null` / `false` / `[]` / `0`), discarding the structured `{errors:[{field, code, translationKey}]}` body. The app cannot show "name is too short" — it shows nothing or a generic failure.

This chat makes every service surface that structured error, adds one shared helper to turn an error into screen-state, fixes two reportService bugs in passing, and makes airplane mode show an offline screen instead of the maintenance screen.

This is foundation work. It delivers the plumbing the feature chats consume. It does **not** rebuild any screen's error display — that is chat A and the other feature chats.

## 2. Scope — three pieces

1. **Error contract (the core).** Every service surfaces the backend's structured error instead of swallowing it. One shared helper turns a parsed error into per-field screen-state.
2. **reportService fixes.** The dead boolean-inversion bug, and the review-report wire pointing the review id at the wrong field.
3. **Offline detection.** NetInfo installed; a connectivity check in front of the maintenance gate; an offline screen distinct from the maintenance screen.

## 3. Error contract — the plumbing

### 3.1 The wire shape
The backend error shape is Part 7: `{ "errors": [{ "field": "name", "code": "NAME_BANNED_WORDS", "translationKey": "..." }] }`. `field` is `null` for object-level errors. First error per field wins on the client.

Mobile keys on `code`. The recent backend error-code split (codes moved between four enums) left the wire `code` value byte-identical — only the enum home and the `translationKey` string changed. So the split is invisible to mobile; no code-string remapping is needed. Confirm `isErrorWithCode` (`src/lib/utils/isErrorWithCode.ts`) still reads `errors[0].code` correctly and is unaffected.

### 3.2 The shape is not uniform — confirm before building
The audit found one service (`imageTokensService.ts`) reads a *different* error shape: singular `data.error.code`, not the `{errors:[...]}` array. Before building the shared helper, the engineer confirms which endpoints return the array shape `{errors:[...]}` and which (if any) return the singular shape. The helper is built around the array shape that the product / report / user / validation endpoints actually return. Services on a different shape are noted, not force-fitted.

### 3.3 The shared helper
One helper (location and exact signature the engineer's call, stated in the brief). It takes a caught error, extracts the `{errors:[...]}` array if present, and returns a typed result a screen can consume — at minimum: the list of `{field, code, translationKey}` entries, and a by-field lookup (first-error-per-field-wins). It does not render anything and does not touch the translation system — turning a `code`/`translationKey` into displayed text is the screen's job, in the feature chats.

The helper is the single abstraction this chat introduces. It earns its place: ~14 services need the same parse, and the feature chats all consume the same shape (Part 4a).

### 3.4 Services surface, not swallow
Each service that today catches → logs → returns a sentinel changes to surface the parsed error to its caller (throw the typed error, or return it — the engineer picks one consistent pattern and applies it across all services; mixing patterns is the smell). `logServiceError` stays where it fits the existing logging strategy. The sentinel-only path is what's being removed.

No screen wiring. Services hand the structured error up; what screens do with it is out of scope here.

## 4. reportService fixes

`reportService.ts` is one of the services getting the §3 treatment. Two extra fixes land in the same file:

### 4.1 Boolean inversion (dead, fix anyway)
`reportService.ts:16,24` — the `error` field is `true` on success (`res.status !== 406 || (2xx)` is true for any 2xx). No consumer reads it today, so no runtime impact, but it's a trap for the next consumer. Fix to `error: !(res.status >= 200 && res.status < 300)` or equivalent.

### 4.2 Review-report wire
The dashboard review-report cards send the review id in `reportedProductId`. The backend now expects a dedicated `reportedReviewId` field. Fix the wire:
- Add `reportedReviewId` to the mobile report request type / `ReportButton` / `ReportDialog`, threaded the same way `reportedUserId` / `reportedProductId` are.
- `ReceivedReviewCard.tsx` and `GivenReviewCard.tsx` mount the report with the review id in `reportedReviewId`, not `reportedProductId`.
- The audit noted `reportedUserId` is omitted though `ReportDialog` types it required — reconcile: either send it where the contract needs it or correct the type. Engineer's call against the current backend contract, stated in the brief.

### 4.3 Error text — confirm, do not add
The two review error codes (`REPORTED_REVIEW_ID_REQUIRED`, `REPORTED_REVIEW_NOT_FOUND`) were seeded backend-side across all locales when review-reports shipped, and mobile pulls translations from the backend. The engineer **confirms these keys resolve on mobile**. If they resolve: nothing to add. If a key is missing: flag it back to Mastermind as a one-line finding — do not author translation rows in this chat.

### 4.4 What is NOT in scope
- No report surface on public reviews (`ProductReview.tsx`). The audit confirmed none exists; dashboard-only is the intended state. Not a gap.
- This absorbs the planned `oglasino-expo-review-reports` mobile chat — with the wire fixed here and no public surface wanted, that chat has nothing left. Docs/QA: note it absorbed so no empty chat is opened later.

## 5. Offline detection

### 5.1 The problem
No network-state library is present. Maintenance is decided at `bootStore.ts` Gate 1 (`GET /public/maintenance/active`, 5s timeout, unreachable → maintenance). In airplane mode, Gate 1 times out and the user lands on the maintenance screen — "we're under maintenance" when the truth is "you're offline."

### 5.2 The fix
- Install `@react-native-community/netinfo`.
- Add a connectivity check **before** the maintenance gate. Insertion point per the audit: `bootStore.ts:120`, ahead of `runMaintenanceGate()`. A new pre-maintenance step (a "Gate 0") → a new `'offline'` `BootStatus` → a new overlay branch in `app/_layout.tsx`.
- When the device has no connectivity, show an **offline screen**: mirrors the maintenance screen's look (the `BaseSiteSelector` `isMaintenance` branch is the reference), copy says the app needs a connection to work.
- When connectivity returns, re-enter boot the same way maintenance-clear does — non-destructive re-entry per the boot-redesign invariants (re-entry clears nothing; the existing `start()` path is reused).

### 5.3 Honoring the boot machine's invariants
The offline check is subject to the three boot-redesign invariants: one mount effect with empty deps, the machine writes status / the view reads it, re-entry destroys no progress. The offline check is a gate like the others, not a reactive effect. The engineer reads the boot-redesign spec before touching `bootStore.ts` and states in the summary how the offline path honors each invariant.

### 5.4 Offline copy needs translation keys
The offline screen's text needs keys in the ERRORS namespace. Mobile translation seeds come from the backend. If new keys are required for the offline copy, that is a backend seed — flag it as a dependency; do not hand-author mobile-only strings. The engineer surfaces the exact key(s) needed; Mastermind decides whether to seed backend-side or reuse an existing key.

### 5.5 What is NOT in scope
The "should mobile poll the backend for maintenance at all, or read it from the edge worker" question is a separate cross-repo (router + mobile) decision, already an open `issues.md` entry (2026-05-29). Φ4 keeps the backend maintenance poll. It only adds an offline screen in front of it. Do not touch the maintenance gate's existing behavior beyond letting the offline check run first.

## 6. Trust boundaries (Part 11)

None affected. Mobile sends all report ids (`reportedUserId`, `reportedProductId`, `reportedReviewId`) as-is; the backend validates them server-side (confirmed clean in the report trust-boundary fix, 2026-05-28). Mobile makes no local trust decision off these ids. Surfacing structured errors is read-only consumption of a server response — no decision input changes.

## 7. Brief order

All in `oglasino-expo`, sequential, each its own session:

1. **Error-contract plumbing** — confirm the wire shape(s), build the shared helper, convert the services to surface instead of swallow. Includes the reportService §4.1 boolean fix (it's one of the services).
2. **Review-report wire** — `reportedReviewId` field + threading, the two dashboard card mounts, the `reportedUserId` reconciliation, the §4.3 key-resolution confirmation. Separate from brief 1 so the error-plumbing change and the report-wire change are reviewable independently.
3. **Offline detection** — NetInfo install, Gate 0, `'offline'` status, offline overlay, re-entry on reconnect, invariant compliance, the §5.4 key surfacing.

Briefs 2 and 3 are independent of each other; both depend on nothing from brief 1 except living in the same branch. Order 1 → 2 → 3 is the default; 2 and 3 can swap if convenient.

## 8. Definition of done

- Every service surfaces structured backend errors per Part 7, keyed on `code` against the current four-enum taxonomy; field-level errors are reachable by a caller via the shared helper.
- One shared error-parsing helper exists; no screen wiring added.
- `reportService` boolean inversion fixed; review-report wire points the review id at `reportedReviewId`; the two review error keys confirmed to resolve on mobile (or a missing-key flag raised).
- NetInfo integrated; airplane-mode cold start shows the offline screen, not the maintenance screen; reconnect re-enters boot non-destructively.
- The boot-redesign's three invariants hold across the offline-gate addition.
- Closes the `state.md` Risk Watch entry: "Mobile service layer silently swallows backend validation errors."
- Runtime verification of the offline behavior is deferred to the Ψ chat (airplane-mode check) — Φ4 ships the code, Ψ verifies on-device.
- `npx tsc --noEmit`, `npm run lint`, `npm test`, and `npx expo-doctor` (where relevant) green per conventions Part 4.

## 9. After this chat

The Expo structural foundation program is done (all four foundation chats shipped). The A–I feature queue opens against a structurally correct foundation — chat A (mobile validation rebuild) first, since it was blocked on this chat's service layer.

## Factual vs inferred

- **Factual** (from the 2026-05-29 audit, verified first-hand against `new-expo-dev`): the ~14-service swallow pattern; `imageTokensService` reading the singular shape; `isErrorWithCode` reading `errors[0].code`; the reportService boolean bug at `:16,24`; review-reporting ~35% wired with the id in `reportedProductId` and no `reportedReviewId` field; no public-review report surface; netinfo absent; maintenance at `bootStore.ts` Gate 1; airplane-mode → maintenance screen; insertion point at `bootStore.ts:120`.
- **Inferred** (Mastermind framing, confirmed by Igor): foundation scope = plumbing + one helper, no screen wiring; review-report wire fixed here rather than in its own chat, with the public-review surface explicitly dropped and the `review-reports` mobile chat absorbed; offline screen mirrors the maintenance screen's look; the edge-worker maintenance reframing stays a separate `issues.md` item.

## Session log

- **2026-05-29 — Phase 2 audit** ([`sessions/2026-05-29-oglasino-expo-service-error-contract-1.md`](../sessions/2026-05-29-oglasino-expo-service-error-contract-1.md); deliverable [`sessions/audit-expo-service-error-contract.md`](../sessions/audit-expo-service-error-contract.md)): read-only audit mapped the post-`expo-boot-redesign` service layer against the four-enum error-code split and the review-report contract. Confirmed ~14 services swallow-to-sentinel; `imageTokensService` reads the singular shape; `isErrorWithCode` reads `errors[0].code` only; reportService boolean bug; review-reporting ~35% wired with the id in `reportedProductId`; no public-review report surface; netinfo absent; maintenance at Gate 1; insertion point `bootStore.ts:120`.
- **2026-05-29 — Brief 1 (§3 error plumbing)** ([`sessions/2026-05-29-oglasino-expo-service-error-contract-2.md`](../sessions/2026-05-29-oglasino-expo-service-error-contract-2.md)): shared `parseServiceError` helper at `src/lib/utils/parseServiceError.ts` returning `{ errors, byField }`; ~14 services converted to surface-via-throw (one consistent pattern). ~10 callers now throw, three with no catch (`app/owner/dashboard/user.tsx:163`, `BasicInfoProductDialog.tsx:82`, `FollowUserButton.tsx:43`) — the designed seam for chat A. 194 tests pass; tsc/lint green.
- **2026-05-29 — Brief 2 (§4 review-report wire)** ([`sessions/2026-05-29-oglasino-expo-service-error-contract-3.md`](../sessions/2026-05-29-oglasino-expo-service-error-contract-3.md)): `reportedReviewId` threaded button → dialog → body; `ReceivedReviewCard` + `GivenReviewCard` re-pointed to `reportedReviewId`; `reportService` boolean inversion fixed; `ReportDialog` target-id types made optional; `reportedUserId` not sent on REVIEW reports (server-derived); the two REVIEW error keys confirmed resolving on mobile. 198 tests pass; tsc/lint green.
- **2026-05-29 — Brief 3 (§5 offline detection)** ([`sessions/2026-05-29-oglasino-expo-service-error-contract-4.md`](../sessions/2026-05-29-oglasino-expo-service-error-contract-4.md)): `@react-native-community/netinfo` 11.4.1 (SDK-54-pinned) installed; connectivity Gate 0 ahead of the maintenance gate in `bootStore`; new `'offline'` `BootStatus` + offline screen (reuses `BaseSiteSelector`); `OfflineReconnectInit` reconnect re-entry; three boot-redesign invariants honored; hardcoded Serbian offline fallback (a backend key cannot resolve before i18n init). Tests/tsc/lint green.

Status: `shipped (code)`. On-device airplane-mode verification deferred to the Ψ chat, gated on the pending iOS+Android rebuild that bundles the new netinfo native module. See [decisions.md](../decisions.md) 2026-05-29 Φ4 entry.
