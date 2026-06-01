# Session summary

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-28
**Task:** View-count increment — deep web audit (READ-ONLY). Determine why `/public/product/seen/<id>` does not arrive at backend despite the prior `after()` + log-on-failure fix.

## Implemented

- Read-only runtime audit of the `/public/product/seen/<id>` write path. Three pieces of temporary diagnostic instrumentation added and removed in the same session: (1) a `[VIEW-AUDIT] page render` log block at `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx` immediately before the gate, dumping `owner.iamActive` + `owner.state` from `getUserForId` AND from the embedded `productDetails.owner`; (2) a `[VIEW-AUDIT] scheduling after(markAsSeen)` line + a `[VIEW-AUDIT] after() callback firing` line inside the `after()` callback; (3) `[VIEW-AUDIT] markAsSeen ENTERED` / `RESPONSE` / `THREW` lines at the entry, success, and catch arms of `markAsSeen`. Dev server (`next dev -p 3000`) driven through three viewer cases. All temp lines reverted before session end; `git diff` against HEAD on the two touched files matches the prior-fix baseline exactly (no fresh changes from this session).
- Definitive runtime evidence captured per the four-point spec in the brief. Hypothesis (b) confirmed, (c) disproven, (d) ruled out.

## Files touched

- (none — temporary instrumentation added and reverted in-session; on-disk diff is identical to the `markasseen-after-fix-1` session's baseline)

## Files instrumented (and reverted)

- `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx` — 2 temp `console.log` blocks (`[VIEW-AUDIT] page render` + `[VIEW-AUDIT] scheduling/firing`)
- `src/lib/service/nextCalls/productService.ts` — 3 temp `console.log` lines at entry / response / catch arms of `markAsSeen`

## Files read

- `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx` — the gate at line 95
- `src/lib/service/nextCalls/productService.ts` — `markAsSeen`
- `src/lib/service/nextCalls/userService.ts` — `getUserForId` (post-#8: `skipAuth: true` REMOVED, auth forwarded, `revalidate: 300`)
- `src/lib/service/nextCalls/productsSearchService.ts` — `getProductDetails`, portal branch (post-#8: `skipAuth: true` REMOVED, `revalidate: 60`)
- `src/lib/config/fetchApi.ts` — the `request()` implementation; the `await cookies()` hop at line 39 (the throw site)
- `src/lib/types/user/UserInfoDTO.ts` and `src/lib/types/product/ProductDetailsDTO.ts` — to confirm `owner.iamActive` shape and the embedded `productDetails.owner: UserInfoDTO`
- `src/components/client/UserDetails.tsx:55-61` — the CLIENT-SIDE interpretation of `iamActive` ("viewer is this user")
- Prior audits in `oglasino-docs/sessions/`: `2026-05-28-oglasino-web-markasseen-never-fires-audit-1.md`, `2026-05-28-oglasino-web-skipauth-footprint-audit-1.md`, `2026-05-28-oglasino-web-markasseen-after-fix-1.md`
- `git diff HEAD` on the five core files to confirm what is on disk vs HEAD (the `after()` fix and the `skipAuth: true` removals on `getUserForId` + `getProductDetails` portal branch are all on disk on `stage`).

## Audit — per the brief's four-point spec

### 1. The gate's runtime input (`owner.iamActive`)

Diagnostic instrumentation logged `owner` from BOTH sources (the separate `getUserForId(productDetails.ownerId)` fetch AND `productDetails.owner` embedded in the `getPortalProductDetails` response) on every render. Product 8567, owner userId 8452, was used as the smoke target.

| Viewer case | `getUserForId` `owner.iamActive` | `getUserForId` `owner.state` | `productDetails.owner` embedded | Gate `!owner.iamActive` |
|---|---|---|---|---|
| (a) Logged-out (incognito) | **`false`** | `'ACTIVE'` | `null` (field absent from response) | passes — `markAsSeen` SHOULD fire |
| (b) Logged-in non-owner | **`false`** | `'ACTIVE'` | `null` (field absent) | passes — `markAsSeen` SHOULD fire |
| (c) Logged-in owner (userId 8452) | **`true`** | `'ACTIVE'` | `null` (field absent) | **skips correctly** — owner self-view |

**Where `owner` came from:** the live gate input is `getUserForId`'s result, not the embedded `productDetails.owner`. The embedded field came back `null` across all three cases — i.e. the backend's `/public/products?productId=<id>` response shape does NOT carry an `owner` object for the portal/public branch on this build (worth re-checking the DTO contract; flagged in adjacent observations). The gate's actual source is therefore the dedicated `getUserForId` fetch.

**Verdict on point 1.** Post-#8 (with `skipAuth: true` removed from `getUserForId`), `iamActive` resolves AUTHORITATIVELY against the forwarded viewer identity. `false` for non-owner / no-identity viewers; `true` only for the actual owner. The gate behaves exactly as intended at runtime. **Hypothesis (c) DISPROVEN** — `iamActive` is not falsely `true` for normal viewers; the gate is not incorrectly suppressing the call.

### 2. `after()` callback delivery, and whether the fetch fires inside it

For cases (a) and (b) — where the gate passes — runtime evidence is:

```
[VIEW-AUDIT] scheduling after(markAsSeen) for productId= 8567
 GET /rs-sr/product/8567/... 200 in 727ms
[VIEW-AUDIT] after() callback firing for productId= 8567
[VIEW-AUDIT] markAsSeen ENTERED for productId= 8567 { baseUrlSet: true, baseUrlPreview: 'http://localhost:8080/api' }
[VIEW-AUDIT] markAsSeen THREW for productId= 8567 Error: Route /[locale]/product/[productId]/[productName] used `cookies()` inside `after()`. This is not supported. If you need this data inside an `after()` callback, use `cookies()` outside of the callback.
    at request (src/lib/config/fetchApi.ts:39:38)
    at Object.get (src/lib/config/fetchApi.ts:91:5)
```

Three observations stack:

1. The `after()` callback DOES execute (Next 16.2.6's `after` is wired correctly; the response flushes first, the callback runs second).
2. `markAsSeen` IS entered. The `[VIEW-AUDIT] markAsSeen ENTERED` line fires AFTER the GET response, confirming `after()` lifecycle.
3. The fetch chain THROWS at `src/lib/config/fetchApi.ts:39` — the `await cookies()` call inside `request()` — because **Next 15+ explicitly disallows `cookies()` (and other dynamic data APIs) inside `after()` callbacks.** The error message links to https://nextjs.org/docs/canary/app/api-reference/functions/after.

The throw is caught by `markAsSeen`'s outer `try/catch` and routed to `logServiceError`. The post-`markasseen-after-fix-1` "log on failure" check (`if (!response.success) logServiceWarn`) does NOT fire on this path because the failure mode is a throw, not a returned-but-unsuccessful envelope. `logServiceError` IS invoked, so a structured log line should exist; the prior session's "no web-side log line" smoke result is either an environment-specific observation (different log surface) or simply was missed in noise — the runtime evidence here is that the catch arm runs and logs.

**Verdict on point 2.** The callback delivery is intact; the fetch INSIDE it is structurally blocked by Next's runtime guard against `cookies()` in `after()`. **Hypothesis (b) CONFIRMED.** The bytes never hit the wire because the `await cookies()` opcode at `fetchApi.ts:39` throws before `fetch()` is reached.

### 3. Owner-resolution correctness post-#8, and is `iamActive` the right field to gate on

`getUserForId` post-#8 (`skipAuth: true` removed) calls `/public/user?id=<ownerId>` with the viewer's `Cookie:` + `Authorization: Bearer …` attached. The backend computes `iamActive` against the authenticated principal (`true` iff viewer.id == requested.id) and returns it on `UserInfoDTO`. Runtime evidence above confirms this works as designed across the three viewer cases.

**Is `iamActive` the right field to gate on for a view-count increment?** Yes for the current intent (suppressing owner self-views), but the choice is structurally redundant with the gate semantics:

- The client component `src/components/client/UserDetails.tsx:55-61` recomputes `iamActive` locally as `user.id === userDetails.id` (CLIENT truth), implying the canonical interpretation is "viewer is this user."
- The server-side `iamActive` on `UserInfoDTO` is, per the runtime check, populated with the same semantic ("viewer is this user"). Field name and field meaning agree.
- The product page uses it on the SERVER for the gate (`!owner.iamActive` ≡ "viewer is not the owner").

A clearer alternative (out of scope for this audit): the server already has the viewer's identity in `OglasinoAuthentication` and could expose a more specific server-side helper (e.g. `getCurrentUserId()` / `isViewerOwnerOf(productDetails)`) so the gate doesn't depend on shipping `iamActive` across a DTO that is also rendered by the UI. Today's coupling works but conflates "viewer-is-this-profile" (a relational fact) with a field carried on the displayed profile.

**Verdict on point 3.** Post-#8 `owner.iamActive` is correct and authoritative for the gate. Naming + co-location is slightly confusing but does not cause the present bug. Flag as a low-severity adjacent observation; do not refactor in this session.

### 4. Base-URL / fetch resolution

`baseUrlSet: true` and `baseUrlPreview: 'http://localhost:8080/api'` logged on every `markAsSeen` entry. Sibling `FETCH_BACKEND_API.get` calls on the same page (`getUserForId`, `getPortalProductDetails`) succeed against the same base URL — the page renders. Base URL is correctly resolved.

**Verdict on point 4.** **Hypothesis (d) RULED OUT.** The base URL is not the problem.

### Best single explanation, ranked against the four hypotheses

**Single named root cause:** the `markAsSeen` → `FETCH_BACKEND_API.get` → `request()` chain calls `await cookies()` at `src/lib/config/fetchApi.ts:39` inside an `after()` callback. **Next 15+ explicitly rejects `cookies()` (and other dynamic-rendering APIs) inside `after()` callbacks at runtime, throwing the error captured verbatim above.** The throw is caught by `markAsSeen`'s outer `try/catch` and routed to `logServiceError`, so `/public/product/seen/<id>` is never dispatched. This is the structural reason the previous `after()` fix failed manual smoke despite the gate, callback delivery, and base URL all being correct.

**Ranking against the four brief hypotheses:**

| Hypothesis | Status | Note |
|---|---|---|
| (a) `after()` callback never executes | refuted | `[VIEW-AUDIT] after() callback firing` log proves the callback runs |
| (b) callback executes but fetch never fires | **confirmed — root cause** | The fetch chain throws on `await cookies()` before `fetch()` is reached |
| (c) gate skips correctly-but-fatally because `iamActive=true` for normal viewers | refuted | `iamActive=false` for cases (a) and (b); only `true` for case (c) — gate is correct |
| (d) base-URL / fetch resolution wrong for this call | refuted | `baseUrlPreview: 'http://localhost:8080/api'` matches sibling working calls |

The bug is hypothesis (b), with the specific mechanism being Next's runtime guard against `cookies()` inside `after()` — not the orphan-Promise-flush issue the prior audit theorized.

## Tests

- Ran: `npx eslint <touched paths>`, `npx tsc --noEmit`, `npm test`
- Result: eslint clean (no output), tsc clean (no output), vitest `247 passed (247)` across 22 files. Post-revert tree green.
- New tests added: none — read-only audit.

## Cleanup performed

- Removed all `[VIEW-AUDIT]` `console.log` instrumentation from `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx` and `src/lib/service/nextCalls/productService.ts`. Confirmed with `grep -rn "VIEW-AUDIT" src/ app/` (no matches) and `git diff` (touched-file diff vs HEAD is identical to the `markasseen-after-fix-1` baseline — no fresh changes from this session).
- Stopped the background `npm run dev` process.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change. The bug remains OPEN in `state.md`'s Risk Watch (view-count increment); the entry should be updated by Docs/QA only after Mastermind triages this audit's verdict and a follow-up fix lands. No draft text required from this session.
- issues.md: no change. The existing 2026-05-28 view-count entry stands; this audit's verdict is the next-action input for Mastermind, not new issue text. The previously-drafted entry from `markasseen-after-fix-1`'s "For Mastermind" is now stale on its root-cause framing (it credited `after()` as the fix; this audit shows the `after()` fix is blocked by a separate runtime constraint). Mastermind to decide whether to amend the existing entry or supersede it when the next fix brief lands.

## Obsoleted by this session

- The prior audit's (`markasseen-never-fires-audit-1`) §4 "un-awaited orphan Promise" mechanism explanation is partially obsoleted. Yes, the orphan-Promise pattern at HEAD (`markAsSeen(productDetails.id)` un-awaited in RSC render) is unreliable for the documented Next 15+ reason. But the on-disk `after()` fix DOES correctly schedule the callback (proven at runtime); the fetch then fails for a DIFFERENT reason — `cookies()` is disallowed inside `after()`. The prior audit's recommendation "Fix B (`after()`)" remains the right SHAPE; it just needs to be combined with a `skipAuth: true` (or equivalent — see "For Mastermind") on `markAsSeen` so the `cookies()` hop is bypassed.
- Nothing else obsoleted; the audit pair's other findings (display guard `< 0` already fixed in `NumberOfViews`; silent-failure swallow in `FETCH_BACKEND_API.request()` still an adjacent concern) remain valid.

## Conventions check

- Part 4 (cleanliness): confirmed — all temp `console.log` instrumentation reverted; `grep -rn "VIEW-AUDIT" src/ app/` returns no matches; on-disk diff matches the prior-fix baseline exactly. No commented-out code, no unused imports, no `TODO`/`FIXME` added. `npx eslint`, `npx tsc --noEmit`, `npm test` all green.
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): flagged in "For Mastermind"
- Part 6 (translations): N/A this session — no translation keys touched.
- Part 7 (error contract): N/A this session — the `/public/product/seen/<id>` endpoint is not a validation surface.
- Part 11 (trust boundaries): touched only insofar as the gate's `iamActive` is server-derived (verified) and the `markAsSeen` call itself is a view-counter side-effect that the backend can owner-exclude or not, independently of web's gate. No new trust-boundary concern surfaced.
- Other parts touched: none

## Known gaps / TODOs

- The runtime evidence was captured in dev (`next dev -p 3000`, Turbopack). Stage / prod (`next build` + `next start`) may have a different runtime guard implementation for `cookies()` inside `after()` (the error message is identical wording in Next 16.2.6's source either way; the docs link in the runtime error is the canonical one). The audit's verdict is robust to this — Next's documented constraint is the same in dev and prod — but the EMPIRICAL evidence is dev-only. If the next fix needs prod-level confirmation, the same instrumentation pattern can be re-applied on stage; out of scope here.
- The prior session's claim "no web-side log line" was not reproduced or refuted within this audit's scope. Runtime evidence shows `markAsSeen`'s catch IS reached (proven by the `[VIEW-AUDIT] markAsSeen THREW` line) and `logServiceError` IS called from the catch — so a structured log SHOULD exist on the same surface where `logServiceError` writes. Possible explanations (not verified here): the prior smoke environment looked at a different log destination, or the noisy server log buried the line. Worth flagging only because it slightly weakens the prior session's diagnostic — `logServiceError` is in fact firing on every failed `markAsSeen`. Reconciliation is Mastermind's call.
- `productDetails.owner` was observed `null` across all three viewer cases on this build. The TypeScript declaration at `src/lib/types/product/ProductDetailsDTO.ts:6` declares it as a required `UserInfoDTO`. This is a type/wire-shape mismatch and is an adjacent observation flagged in "For Mastermind" — it does NOT affect the present bug because the gate uses the separate `getUserForId` fetch, but the type and the wire disagree.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit; only temp diagnostic `console.log` lines, all removed before session end).
  - Considered and rejected: nothing — no design decisions, no introduced abstractions.
  - Simplified or removed: nothing — no code removed from the working tree this session.

- **Single named root-cause verdict (required by brief Definition of Done):** the `markAsSeen` → `FETCH_BACKEND_API.get` → `request()` chain calls `await cookies()` at `src/lib/config/fetchApi.ts:39` inside the `after()` callback scheduled at `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:96`. Next 15+ explicitly rejects `cookies()` inside `after()` callbacks at runtime, throwing the error verbatim:

  > Error: Route /[locale]/product/[productId]/[productName] used `cookies()` inside `after()`. This is not supported. If you need this data inside an `after()` callback, use `cookies()` outside of the callback.

  The throw is caught by `markAsSeen`'s outer `try/catch` and routed to `logServiceError`. The `/public/product/seen/<id>` fetch never lands on the wire. **Hypothesis (b) from the brief; (c) DISPROVEN by runtime evidence (`iamActive=false` for non-owner viewers, `true` only for the actual owner); (d) RULED OUT (base URL correct).**

- **Fix-shape implications (not a fix, just framing for the next brief).** There are at least three structurally valid ways to unblock `markAsSeen` inside `after()`. All preserve the current `!owner.iamActive` gate (owner exclusion stays correct):

  1. **Pass `skipAuth: true` on the `markAsSeen` call.** Simplest. Skips the `await cookies()` branch entirely; the `/public/product/seen/<id>` endpoint doesn't need viewer identity because the gate already excluded the owner client-side. **The prior fix brief explicitly forbade adding `skipAuth: true`** ("would break server-side owner-exclusion identification"). That posture should be re-examined in light of this audit — the gate already does owner exclusion before scheduling the callback, so the backend does NOT need to identify the viewer to do owner exclusion on `/seen`. Backend can keep its existing no-auth behavior on `/seen`.
  2. **Read `cookies()` outside `after()` and pass them through.** Read the cookie store and Bearer token in the RSC render (before `after()`), then construct the `markAsSeen` request manually inside the callback with those values pre-captured. More invasive (would need a new `markAsSeenWithAuth(productId, cookieHeader, bearer)` shape, or threading `headers` through `FETCH_BACKEND_API.get`'s options bag — the existing `headers` field can already accept a `Cookie:` header verbatim). Preserves auth-forwarded posture.
  3. **Move the increment to a client-side effect (Fix C from `markasseen-never-fires-audit-1`).** Largest change. Client-side fetch carries auth via the axios interceptor; the gate moves to backend-only (no longer needs the web-side `!owner.iamActive` check); counted population shifts (bot / no-JS viewers no longer count).

  **Recommendation framing only:** (1) is the smallest and matches the endpoint's nominal contract (`/public/*`, fire-and-forget). The decision to enable it should be paired with an explicit acknowledgement that owner exclusion is enforced by the gate, not by the backend reading the Bearer on `/seen`.

- **Brief vs reality — nothing material.** The brief's "the prior fix wrapped the call in `after()` from `next/server`" is exactly what's on disk. The brief's hypothesis enumeration is exhaustive and the runtime evidence cleanly ranks them. No re-scoping needed.

- **Adjacent observation — `productDetails.owner` arrives `null` for the portal branch (severity: low-medium).** `src/lib/types/product/ProductDetailsDTO.ts:6` declares `owner: UserInfoDTO` (required, non-null). Runtime evidence on product 8567 across three viewer cases shows the field is absent (`null`) in the response. Possible causes: backend's `/public/products?productId=<id>` does not populate `owner` (intentional or oversight), OR the field is conditional on some flag not set in this smoke environment. The product page does NOT rely on this field today — it does a separate `getUserForId(productDetails.ownerId)` fetch — but the type and the wire disagree, which is a Part-7-flavoured contract mismatch (DTO claim vs actual response). The downstream `<UserDetails userDetails={owner} ... />` would be the consumer if anyone tried to use the embedded owner instead of the separate fetch. Worth flagging to Mastermind for backend audit; not a fix for this session.

- **Adjacent observation — re-flagged from prior audits (no new severity).** `FETCH_BACKEND_API.request()` swallows network/abort errors into a resolved `{ status: 0, errorCode: 'connection.timeout' }` envelope at `src/lib/config/fetchApi.ts:72-79`. This audit's catch arm IS reached because Next throws SYNCHRONOUSLY on `cookies()` inside `after()` (the throw happens at line 39, before the outer try at line 47 wraps `fetch`). So the swallow does not apply here. But the broader silent-failure shape across `/public/*` callers remains unchanged and was flagged in three prior audits (`views-not-displaying-audit-1`, `markasseen-never-fires-audit-1`, `skipauth-footprint-audit-1`, `markasseen-after-fix-1`). Not a new flag.

- **Adjacent observation — the `iamActive` field name is semantically confusing (severity: very low).** The field name reads naturally as "this profile is active" (a property of the displayed user), but the actual semantic is "viewer is this user" (a relational fact between viewer and displayed user). The client-side recomputation in `UserDetails.tsx:55-61` agrees with the relational semantic, and the server (per runtime evidence) populates it the same way. Field-name confusion only; no functional bug. Worth flagging if/when the next refactor touches `UserInfoDTO` — a rename to `viewerIsThisUser` or `isCurrentViewer` would be clearer. Not a fix for this session.

- **Process note — this audit converges the bug trio.** Three audits to date have looked at the views-counter symptom from three angles: (1) `views-not-displaying-audit-1` (display side, fixed in `NumberOfViews`); (2) `markasseen-never-fires-audit-1` (write side, recommended Fix B — `after()`); (3) the present audit (deep audit after Fix B failed smoke). The present verdict shows Fix B was the right SHAPE but is structurally blocked by Next's `cookies()`-inside-`after()` guard. The next-fix shape is one of the three above; the audit is settled and the next move is a fix brief.

- (nothing else flagged)
