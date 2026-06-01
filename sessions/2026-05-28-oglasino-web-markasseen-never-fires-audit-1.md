# Session summary

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-28
**Task:** Audit brief — markAsSeen never fires / views always 0 (NEW, surfaced 2026-05-28). READ-ONLY.

## Implemented

- Read-only audit of the `/public/product/seen/<id>` write path in `oglasino-web`: the call site in the product page RSC render, the `markAsSeen` helper, the `FETCH_BACKEND_API` wrapper, and a repo-wide search for any other invokers. Confirmed the call is un-awaited in an RSC render and that it is the only invocation site. Sized three fix shapes and noted the owner-exclusion / auth implication of each. No code changes.

## Files touched

- (none)

## Files read

- `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx` (the gate + call, lines 91–96; full file 165 lines)
- `src/lib/service/nextCalls/productService.ts` (the `markAsSeen` helper, full file 12 lines)
- `src/lib/config/fetchApi.ts` (the `FETCH_BACKEND_API` request implementation, full file 99 lines — confirms base URL from `process.env.NEXT_PUBLIC_API_URL`, the `await cookies()` hop on non-`skipAuth` calls, and the swallow-to-`{status: 0, errorCode: 'connection.timeout'}` outer try/catch)
- `src/lib/service/nextCalls/userService.ts` (current shape of `getUserForId` — note: working-tree differs from the brief's assumption, see §1 and "For Mastermind")
- `src/lib/types/user/UserInfoDTO.ts` (confirms `iamActive: boolean` field on the owner DTO)
- `src/components/client/initializers/ProductViewTracker.tsx` (full file — confirms analytics-only, does NOT call `markAsSeen`)
- `src/lib/utils/serviceLog.ts` (`logServiceError` — confirms the catch in `markAsSeen` only logs, does not re-throw)
- Prior audit `.agent/2026-05-28-oglasino-web-views-not-displaying-audit-1.md` (the views-display audit; the present audit complements it on the write side)
- Prior audit `.agent/2026-05-28-oglasino-web-skipauth-footprint-audit-1.md` (the `skipAuth` footprint table — referenced for the auth posture of `markAsSeen`)
- Repo-wide grep: `markAsSeen`, `/product/seen`, `product.seen` — only two matches across `*.ts`/`*.tsx`: the definition (`src/lib/service/nextCalls/productService.ts:6`) and the import + single call site (`app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:10,95`)
- `git diff HEAD` on the three core files (verifies what's currently on disk vs HEAD)
- `package.json` (versions: `next ^16.2.6`, `react ^19.2.6`)

## Audit

### §1 — The gate is reachable; the call is un-awaited

**`app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:91–96` (verbatim):**

```tsx
const owner: UserInfoDTO | null = await getUserForId(productDetails.ownerId);
if (!owner) notFound();

if (!owner.iamActive) {
  markAsSeen(productDetails.id);
}
```

- **Path to the gate is open in the steady-state case.** The earlier short-circuits — `if (!baseSite) notFound()` (line 60), `redirect(...)` on baseSite mismatch (line 79–81), the "not found" JSX return when `productDetails` is missing (line 84–89), and `if (!owner) notFound()` (line 92) — all fire only on misconfiguration / 404 / wrong-tenant cases. For any product page that visibly renders (which is the symptom scope: "views always 0 on every product"), the render reaches line 94 with a non-null `owner`.
- **The gate `!owner.iamActive` is satisfied for non-owner viewers**, which is the population that should be counted:
  - Owner viewing own product → `iamActive = true` → gate false → no call. Correct.
  - Authenticated non-owner viewer → `iamActive = false` → gate true → call attempted.
  - Anonymous viewer → no Bearer token attached → backend defaults `iamActive = false` for the no-identity case (per the brief's restatement of the backend audit) → gate true → call attempted.
- **The call at line 95 is NOT awaited.** `markAsSeen` is declared `export const markAsSeen = async (productId: number) => {...}` (productService.ts:6) — i.e. it returns a `Promise<void>`. The call site invokes it as a bare statement without `await`. The render function `ProductPage` continues immediately and returns the JSX tree without waiting for the Promise to settle.

### §2 — `markAsSeen` itself: no early returns, no env guard, but adds two awaits before the fetch lands on the wire

**`src/lib/service/nextCalls/productService.ts` (full file, 12 lines, verbatim):**

```ts
'use server';

import { FETCH_BACKEND_API } from '@/src/lib/config/fetchApi';
import { logServiceError } from '@/src/lib/utils/serviceLog';

export const markAsSeen = async (productId: number) => {
  try {
    await FETCH_BACKEND_API.get('/public/product/seen/' + productId);
  } catch (err) {
    logServiceError('product.next.markAsSeen', err);
  }
};
```

- No env guard, no feature flag, no `if (!productId) return`, no conditional skip. The endpoint string is `'/public/product/seen/' + productId` — a straight concatenation, no encoding hazards (numeric id). Base URL resolution lives in the helper.
- `FETCH_BACKEND_API.get` enters `request<T>(endpoint, options)` in `src/lib/config/fetchApi.ts:29–80`. Trace:
  - Line 3: `const BASE_URL = process.env.NEXT_PUBLIC_API_URL as string;` — read at module load. If the env var is missing, `BASE_URL` is `undefined` and the literal `"undefined/public/product/seen/<id>"` would be passed to `fetch`. `fetch` would throw `TypeError: Invalid URL`, caught by the inner try/catch at line 47, which returns `{ status: 0, success: false, data: {}, errorCode: 'connection.timeout' }` — and `markAsSeen`'s outer try awaits a *resolved* promise (no throw), so `markAsSeen`'s catch never fires either. `logServiceError` is therefore not called and there is no trace of the failure in logs. **A missing or misnamed `NEXT_PUBLIC_API_URL` would produce exactly the reported symptom and is invisible at the call site.** Worth verifying as a sanity check, but the same env var is used by every other `FETCH_BACKEND_API` call that *is* working (product details, base site, getUserForId, etc.), so this is unlikely in production — flagged only because the failure mode would be silent.
  - Lines 36–45: `if (!skipAuth && typeof window === 'undefined') { const cookieStore = await cookies(); ... }`. `markAsSeen` does NOT pass `skipAuth: true`, so this branch runs and `await cookies()` is the first await inside the chain. Only AFTER it resolves is line 48's `await fetch(\`${BASE_URL}${endpoint}\`, ...)` reached, and only after THAT request is dispatched on the TCP socket are the bytes on the wire.
  - The chain therefore contains **two awaits** (`cookies()` then `fetch()`) BEFORE any network IO has been initiated for the seen-write — and this entire chain hangs off an un-awaited outer Promise from the RSC render.
- The outer try/catch in `markAsSeen` only catches synchronous throws from inside the function body. Because `FETCH_BACKEND_API.get` *swallows* its own errors inside `request()`'s inner try/catch (line 47–79) and returns a resolved `{ status: 0, ... }` envelope on failure, `markAsSeen`'s catch effectively never runs — failure paths come back as a normal resolution. Net: any failure of the seen-write (env, network, abort, backend 5xx) produces NO log line on the web side.
- Endpoint string and base-URL resolution are otherwise correct. Same pattern as every other working `/public/...` call in the repo.

### §3 — The only invocation site is the un-awaited RSC call

Repo-wide grep of `*.ts`/`*.tsx` for `markAsSeen` and `/product/seen`:

```
app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:10  import { markAsSeen } from '@/src/lib/service/nextCalls/productService';
app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:95      markAsSeen(productDetails.id);
src/lib/service/nextCalls/productService.ts:6   export const markAsSeen = async (productId: number) => {
src/lib/service/nextCalls/productService.ts:8       await FETCH_BACKEND_API.get('/public/product/seen/' + productId);
```

- Three references: one import, one call, the definition (with the endpoint string). **No client-side invoker. No `useEffect` calling it. No analytics-side dispatcher. No server-action form action. No `after()` scheduler.** The un-awaited RSC render call at page.tsx:95 is the entire web-side write surface for `/public/product/seen/<id>`.
- **`ProductViewTracker.tsx` does NOT call `markAsSeen`.** Confirmed by reading the full file (62 lines). It is analytics-only: it fires `track('product_view', {...})` once per mount after `useAuthResolved()` resolves, with no reference to `markAsSeen` or the seen endpoint. It returns `null`. The brief's framing — "is there any `ProductViewTracker` or client effect that was SUPPOSED to call it but doesn't?" — answers cleanly: there isn't one, never was.

### §4 — Why an un-awaited fetch in an RSC render does not reliably reach the backend

This is the prime hypothesis. Walking through it explicitly:

**The Next 15 / React 19 model.** A Server Component's render function is itself an async function. React invokes it, awaits its return, and serializes the result into the RSC payload. When the render function returns its JSX tree, React considers that Server Component's work done. The request lifecycle — including the async-context store that backs `cookies()` and `headers()` — is sized to the request, and Next ends it when the response is fully sent.

**An un-awaited Promise started during render is orphaned with respect to the render.** The render does not include it in its set of awaited dependencies. React/Next have no record that there is pending work tied to the request. There is no documented guarantee that the runtime will keep the request-scoped async context alive, or keep the AbortSignal un-aborted, until that orphan Promise settles.

**Specific to this code path:** the orphan Promise's first awaited operation is `await cookies()` (inside `FETCH_BACKEND_API.request`). `cookies()` in Next 15 is an async API that reads from the per-request AsyncLocalStorage cookie store. If the request lifecycle is torn down before that microtask is scheduled, the chain stops there — `fetch` is never called, no bytes hit the wire. Even if `cookies()` resolves, the subsequent `await fetch(...)` may run against an AbortSignal that Next has already aborted at response-flush time, which undici interprets as a cancellation before the request is dispatched.

**This is exactly the failure mode Next 15+ added `after()` to solve.** Next's own documentation for `next/server`'s `after` describes it as the supported way to schedule side effects to run after the response is sent — explicitly because un-awaited side effects started during render are not guaranteed to complete. The very existence of `after()` is documentary evidence that the fire-and-forget pattern is unsupported / unreliable.

**Empirically consistent with the symptom.** The user reports view count = 0 on every product. With this code path:
- The fetch sometimes runs (when the orphan happens to get scheduled before teardown), sometimes does not. In production under load — pooled connections, varied response sizes, varied scheduler pressure — the "does not" case dominates.
- The path produces no log line on failure (see §2). So a 0% success rate looks identical to a 100% success rate from the web's log surface — there is nothing to falsify the assumption that the call is firing.
- The prior backend check (per the brief) confirms: backend chain is fully working **if** `/public/product/seen/<id>` arrives. The bytes are not arriving — and the un-awaited orphan is the structural reason they aren't.

**Could it be something else?** Three alternatives were considered:
- **Missing `NEXT_PUBLIC_API_URL`.** Would produce silent zero increments, BUT would also break every other `FETCH_BACKEND_API` call, including the working `getPortalProductDetails` and `getUserForId` that render the page. Rejected — the page renders, so the env var is set.
- **`/public/product/seen/<id>` returning silently but not persisting.** Already ruled out by the backend chain check (per the brief's framing).
- **The gate suppressing the call for all viewers.** Rejected in §1 — the gate allows the call for non-owner authenticated viewers and for anonymous viewers (the population that should be counted). The owner-only suppression is correct behaviour.

### §5 — Verdict

**Single most likely reason `/public/product/seen/<id>` never reaches the backend, file:line:**

> `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:95` — the un-awaited `markAsSeen(productDetails.id)` call inside the RSC render. The orphan Promise's first hop is `await cookies()` inside `FETCH_BACKEND_API.request` (`src/lib/config/fetchApi.ts:39`); the render function returns before that microtask completes, and Next 15 / React 19 do not guarantee that an un-awaited side effect tied to the request scope flushes before the request lifecycle ends. The fetch is never dispatched. Every failure path is also silent (see §2 — `request()` swallows errors into a resolved envelope, so `markAsSeen`'s catch never fires and `logServiceError` is never called). Net: 0% delivery, 0 log lines.

**Sized fix shapes — three candidates, each with the auth/owner-exclusion implication called out (per backend Q3 in the brief — owner exclusion needs auth on the call).**

#### Fix shape A — `await` the call in the RSC render

Smallest diff:

```ts
if (!owner.iamActive) {
  await markAsSeen(productDetails.id);
}
```

- **Cost:** the product page render now blocks on a backend round-trip for every non-owner / anonymous viewer. The seen endpoint is presumably fast (single counter increment) but still adds RTT to every product page render. On the same backend host this is small; over the public internet it shifts product-page TTFB.
- **Auth posture:** unchanged from today. `markAsSeen` calls `FETCH_BACKEND_API` without `skipAuth`, so cookies + Bearer (when present) are forwarded. Backend can identify the viewer and perform server-side owner exclusion if it chooses. The current web-side `!owner.iamActive` gate also still excludes owner views before the call goes out. Owner exclusion: covered by gate + backend.
- **Awkwardness flagged by the brief:** an owner-gated `await` is a slightly odd shape — for owner views the render skips the await, for non-owner views it blocks on the increment. Functional, but not elegant. The render-latency cost only applies to the gated branch.
- **Why this works:** awaited means React's render set includes the call; Next will not end the request until it settles. Reliable.

#### Fix shape B — `after()` from `next/server` (Next 15+ native fire-and-forget)

Next 16 is on `package.json` (`"next": "^16.2.6"`), so `after()` is available stably.

```ts
import { after } from 'next/server';
...
if (!owner.iamActive) {
  after(() => markAsSeen(productDetails.id));
}
```

- **Cost:** zero render latency. The callback runs after the response is sent, within an extended request scope that Next keeps alive specifically for this purpose. Cookies and headers remain available inside `after()`.
- **Auth posture:** identical to A. The `markAsSeen` chain still calls `FETCH_BACKEND_API` without `skipAuth`; cookies + Bearer are still attached. Backend can do owner exclusion if it wants. Web-side gate still suppresses owner self-views before the call is scheduled. Owner exclusion: covered by gate + backend.
- **Why this works:** `after()` is the documented Next 15+ API for exactly this case — side-effect work that must run reliably but should not block the response. The whole reason Next added the API is the unreliability described in §4.
- **Risk:** none specific to this code path. `after()` is the lowest-risk reliable fix and most closely matches the current intent ("fire-and-forget, after the user has the page").

#### Fix shape C — Move the increment to a client-side effect (a `ProductViewTracker`-style hook fires `/public/product/seen/<id>` once per mount via the client axios `BACKEND_API`)

The shape would mirror `ProductViewTracker`: a `'use client'` component that calls the seen endpoint once per mount after `useAuthResolved` resolves, then returns `null`. The gate moves from the RSC to the client effect.

- **Auth posture changes — and this matters for owner exclusion.** The client-side axios instance `BACKEND_API` (`src/lib/config/api.ts`) attaches `Authorization: Bearer <token>` from `auth.currentUser` via its request interceptor, and uses `withCredentials: true` so the browser sends cookies. So the call DOES carry auth — but only after Firebase auth has resolved on the client. Two consequences:
  - For an authenticated viewer (owner or not), the backend sees the viewer's identity authoritatively. The backend can owner-exclude directly. The web-side `!owner.iamActive` gate becomes redundant and is candidate for removal.
  - For an anonymous viewer, no Bearer attaches (no `auth.currentUser`). Backend gets a true no-identity request. Owner exclusion is moot for that population.
- **This is the most structurally aligned with conventions Part 11 (server is the trust boundary).** The owner-exclusion decision moves entirely to the backend, which already has the authoritative identity from the Bearer token. The web no longer needs to consult `owner.iamActive` from a separately-fetched DTO to decide whether to fire the increment.
- **Cost:** introduces a new client component (or extends `ProductViewTracker`). Adds a once-per-mount client request that wasn't there before. Subject to client-side caveats: ad-blockers, no-JS visitors, bot crawlers that don't run JS, all of which would no longer be counted. For a view counter on a public marketplace, that is arguably *more* correct (bot views shouldn't count anyway) but worth flagging.
- **Race with `useAuthResolved`:** the existing `ProductViewTracker` already handles this — gate the fire on `if (!resolved) return;` and use a `useRef` for the once-per-mount guard. This pattern would carry over.
- **Where it would live:** either fold into `ProductViewTracker` (it already takes `productId` and `ownerId` and resolves identity, so the seen-write is structurally adjacent to the analytics fire) or create a sibling `ProductSeenTracker`. Folding into the existing tracker is simpler — one mount, one auth-resolve, two side effects.

**Recommendation across the three shapes:** B is the smallest, lowest-risk fix that preserves current auth posture and matches the existing intent ("fire-and-forget after response"). A is acceptable but adds render latency. C is the most structurally clean (moves the trust decision to the backend, removes the gate, aligns with Part 11), but it is the largest change and shifts the population that gets counted (bots / no-JS out, which is arguably correct). Mastermind decides — the brief is read-only and explicitly does not ask for a decision.

## Cleanup performed

- none needed (read-only audit, no code changes)

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change. This audit's verdict is draft material for Mastermind to triage into the existing or a new `issues.md` entry covering the views-counter bug (the views-display audit `2026-05-28-oglasino-web-views-not-displaying-audit-1.md` is its sibling on the read side). Per Part 3 / Part 5 closure gate: nothing is drafted here for Docs/QA to apply directly; the next-action is Mastermind triage of the two-audit pair, not a Docs/QA edit.

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): confirmed — read-only audit, no commented code, no unused imports, no console.log, no TODO/FIXME added.
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): flagged in "For Mastermind"
- Part 6 (translations): N/A this session
- Part 7 (error contract): N/A this session (no validation surface touched)
- Part 11 (trust boundaries): touched in §5 / Fix C — moving the seen-write to the client uses the client's authenticated identity authoritatively, removing the web's reliance on `owner.iamActive` from a separately-fetched DTO. Flagged, not enacted.
- Other parts touched: none

## Known gaps / TODOs

- **Whether the orphan Promise EVER reaches the backend** is not 100% determinable from web code alone. The structural argument in §4 is strong (un-awaited side effects in RSC render are explicitly unsupported by Next 15+; `after()` exists for exactly this case), but a definitive test would be: (a) add a temporary `await` to the call and observe whether the count starts incrementing reliably in stage, OR (b) instrument the backend `/public/product/seen/<id>` handler to count hits and compare to the rate of product page renders. Both are out of scope here.
- **The direction of the backend `iamActive` default for the no-identity case** is taken from the brief's restatement of the backend audit, not verified web-side. Out of scope per the brief; both directions yield "gate allows the call for the population that should be counted" in the current working tree (skipAuth removed — see "For Mastermind" / Brief vs reality).
- **`NEXT_PUBLIC_API_URL` is implicitly trusted to be set** in the §2 analysis (because other working calls share it). Not formally verified in this audit. If the count actually became non-zero in stage after applying any of the fix shapes, the env var is by definition fine.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit; no code added).
  - Considered and rejected: nothing (no implementation decisions made).
  - Simplified or removed: nothing (no code removed).

- **Single named root-cause verdict (required by brief Definition of Done):** the un-awaited `markAsSeen(productDetails.id)` at `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:95` is the structural reason `/public/product/seen/<id>` never reaches the backend. Next 15 / React 19 do not guarantee that fire-and-forget side effects started during an RSC render flush before the request lifecycle ends — `await cookies()` inside `FETCH_BACKEND_API.request` (`src/lib/config/fetchApi.ts:39`) is the first hop and is exposed to teardown. Three sized fix shapes in §5; recommendation: **Fix B (`after()` from `next/server`)** — smallest, lowest-risk, preserves current auth posture and matches existing intent. Fix A (`await`) is acceptable but adds render latency on the gated branch. Fix C (client-side `ProductSeenTracker`) is the most structurally aligned with Part 11 but is the largest change and shifts the counted population.

- **Brief vs reality — `getUserForId` already had `skipAuth: true` removed in the working tree.** The brief says: "with the CURRENT `skipAuth: true` on `getUserForId`, the backend check says `iamActive` defaults to `false` for the no-identity case." That statement is true of HEAD (`6f03de5`), but `git diff HEAD -- src/lib/service/nextCalls/userService.ts` shows `skipAuth: true` is REMOVED on disk and a new comment is added: "Auth is forwarded so the backend can populate viewer-dependent fields (`isFollowingCurrent`, `iamActive`) authoritatively." The product page diff in the same working tree also added a `notFound()` guard on null owner and widened the type to `UserInfoDTO | null`. These uncommitted changes appear to be a separate in-flight fix for issue #8 (skipAuth footprint). Did not change the verdict — for the population that should be counted (non-owner authenticated viewers and anonymous viewers) the gate still allows the call regardless of which `skipAuth` posture is in force. Flagged because the brief reads as if it was written against HEAD and a chunk of related uncommitted work exists on disk that Mastermind may not have current visibility into. If those edits land, the views-display audit's "skipauth #8" framing needs revisiting too — both audits' "skipAuth-induced iamActive default" hedges become moot once #8 is actually fixed.

- **Adjacent observation — `request()` swallows all errors into a resolved `{ status: 0, errorCode: 'connection.timeout' }` envelope (`src/lib/config/fetchApi.ts:72–79`); `markAsSeen` therefore never catches and never logs (severity: medium).** Combined with the present bug, this means a 0% delivery rate produces 0 web log lines — there is nothing in the operational surface to flag that the call is silently failing. Independent of the chosen fix shape, consider either (a) propagating non-2xx as a thrown error from `request()` for callers that explicitly opt in, or (b) having `markAsSeen` inspect the returned envelope's `success`/`status` and log on failure. Out of scope, but worth flagging because the silent-failure mode is what allowed this bug to persist undetected — every other audit on this surface has called the same shape out (`skipauth-footprint` audit also notes the swallow; this is the third time the shape has been adjacent-flagged).

- **Adjacent observation — `markAsSeen` does not pass `skipAuth: true` despite hitting a `/public/*` endpoint (severity: low, re-flagged from `views-not-displaying-audit-1` and `skipauth-footprint-audit-1`).** Re-flagging only because it is structurally relevant to the fix-shape choice: under Fix A or Fix B, whether `markAsSeen` carries auth determines whether the backend can owner-exclude server-side or whether the web-side gate must remain. Today the gate is the truth source. If Mastermind decides on Fix C (client-side), the question is settled — auth attaches client-side via the axios interceptor. If Fix A or Fix B is chosen, decide whether `markAsSeen` should pass `skipAuth: true` (saves the `await cookies()` hop, makes the call cacheable by URL, but the backend then cannot identify the viewer for any future owner-exclusion logic). Two-way decision; either is defensible.

- **Adjacent observation — `markAsSeen` would be a single fetch call per product-page render and is currently dispatched without `cache: 'no-store'` or `next: { revalidate: ... }` (severity: low).** Next 15's default for `fetch` in dynamic routes is uncached (`auto no store`). So this is fine today. But IF the fix moves to `skipAuth: true` (per the previous bullet), the call could fall into the Data Cache and be deduped across requests by URL — which for an increment endpoint is exactly the wrong behaviour. If Mastermind picks Fix A or B AND wants to add `skipAuth: true` for the cookie-hop saving, also add `cache: 'no-store'` (or pass `next: { revalidate: 0 }`) to defeat the Data Cache. Trivial to get wrong; worth being explicit.

- **Process note — this audit and the prior views-display audit are siblings.** The views-display audit (`2026-05-28-oglasino-web-views-not-displaying-audit-1.md`) named the render-guard `<= 0` at `NumberOfViews.tsx:20` as the proximate cause of the visible "absent" symptom; it explicitly hedged that the upstream cause could be "never incremented" (scenario 2a) OR "incremented but `/seen` doesn't persist what `/views` reads" (scenario 2b). The present audit converges on scenario 2a and adds the structural mechanism (un-awaited RSC side effect). Combined with the brief's restatement of the backend audit ("backend view-counter chain fully working IF `/seen/<id>` is actually called"), the case for 2a is now well-supported and 2b can be parked. The two web fix shapes are now: (i) restore reliable `/seen` delivery (this audit's Fix A/B/C) AND (ii) relax the `<= 0` display guard so a real 0 renders visibly (the views-display audit's recommendation). Both are needed — only fixing (i) leaves new products silently invisible until their first view; only fixing (ii) displays "0" everywhere but doesn't repair the underlying counter.

- (nothing else flagged)
