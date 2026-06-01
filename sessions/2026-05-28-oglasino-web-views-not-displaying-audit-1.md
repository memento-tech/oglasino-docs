# Session summary

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-28
**Task:** Audit brief — product-page view count not displaying (NEW, surfaced in bug chat 2026-05-28). READ-ONLY.

## Implemented

- Read-only audit of the full product-page view-count chain in `oglasino-web`: display component, display fetch, mount site, increment call, increment gate. No code changes. Output to the two named summary files per brief.

## Files touched

- (none)

## Files read

- `src/components/client/NumberOfViews.tsx` (display component — full file, 32 lines)
- `src/lib/service/reactCalls/productService.ts` (display fetch `getNumberOfViews`, lines 347–361; also confirmed `BACKEND_API` is the axios client used)
- `src/lib/service/nextCalls/productService.ts` (increment call `markAsSeen`, full file, 12 lines)
- `src/lib/config/api.ts` (the axios `BACKEND_API` — interceptors, 404 normalisation at line 44–49)
- `src/lib/config/fetchApi.ts` (`FETCH_BACKEND_API` — `skipAuth` mechanism that backs the §2 hypothesis)
- `src/components/server/ProductDetails.tsx` (mount site at line 77, inside the favorites block)
- `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx` (product page render, increment gate at lines 93–95)
- `src/components/client/initializers/ProductViewTracker.tsx` (confirmed unrelated — analytics-only, returns null, doesn't touch the view counter)
- `src/lib/service/nextCalls/userService.ts` (`getUserForId` — confirms `skipAuth: true`, 5-min revalidate, the source of the defaulted `owner.iamActive`)
- `src/lib/types/user/UserInfoDTO.ts` (confirms `iamActive: boolean` is on the owner DTO)
- `src/lib/types/product/ProductDetailsDTO.ts`, `src/lib/types/product/ProductOverviewDTO.ts` (confirms there is **no** `numberOfViews` field on either DTO — only `numberOfFavorites` is embedded; the count must be fetched separately)
- Prior audit `.agent/2026-05-28-oglasino-web-skipauth-footprint-audit-1.md` (Section 2 row 6, Section 3 route 2 — the existing read on `getUserForId`'s skipAuth defaulting of `iamActive`)
- `oglasino-docs/issues.md` 2026-05-14 entry "NumberOfViews fires a redundant duplicate fetch" (the 2026-05-16 fix introduced the current guard alongside the dep fix)
- Pending working-tree diff for the five chain files via `git diff HEAD` (confirms no chain-relevant changes are pending; the diff in `reactCalls/productService.ts` is rate-limit handling unrelated to views)

## Audit

### §0 — Establishing the symptom

The visible symptom is **Absent**, not zero-or-blank, not flash-then-gone. Mechanism, verbatim from `NumberOfViews.tsx` lines 10–22:

```tsx
export default function NumberOfViews({ productId }: { productId: number }) {
  const tCommon = useTranslations(TranslationNamespaceEnum.COMMON);
  const [numberOfViews, setNumberOfViews] = useState(0);                       // line 12

  useEffect(() => {                                                            // line 14
    getNumberOfViews(productId)
      .then((res) => setNumberOfViews(res))
      .catch(() => setNumberOfViews(0));                                       // line 17
  }, [productId]);                                                             // line 18

  if (numberOfViews <= 0) {                                                    // line 20
    return null;
  }
```

- Initial state is `0` (line 12). On every first render (SSR and the initial CSR pass), the guard at line 20 evaluates `0 <= 0` → `true` → returns `null`. The element is absent from the DOM before the fetch resolves.
- The render gate uses `<= 0`, not `< 0`. So a real-but-zero count remains absent; only `numberOfViews > 0` causes the element to render.
- The `.catch(() => setNumberOfViews(0))` (line 17) explicitly funnels every fetch failure (404, 5xx, network, axios rejection — see §1 below for the axios behaviour) into state `0`, which the guard then hides. There is no error UI surface.
- Because the initial state already trips the guard, there is **no flash window**: the element is never present pre-fetch, so it cannot "flash then disappear." Brief's prior `issues.md` 2026-05-14 entry confirms the guard was added on 2026-05-16 with the explicit purpose "to hide the placeholder flash" — it does that, and additionally hides every 0-resolution case.

Net symptom: the views element never appears in the DOM unless `getNumberOfViews` resolves to a strictly-positive number. For "all users" on the affected product(s), it resolves to either `0` (success or failure both write `0` through the same guard) or a falsy value that satisfies `<= 0`.

### §1 — Display side

**Component:** `src/components/client/NumberOfViews.tsx` (32 lines, single component, no children, no providers, no auth gate).

- Mount: in `src/components/server/ProductDetails.tsx:77`, inside the favorites block: `<NumberOfViews productId={productDetails.id} />`. Unconditional mount — no owner gate, no hydration flag, no breakpoint guard, no Zustand subscription. `ProductDetails` is itself rendered unconditionally from the product page (`app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:160`), with no preview-mode wrapper since the product detail page is not in preview mode (the preview path is a separate render path; here `isPreview` defaults to `false`).
- Render gate state (verbatim, line 20): `if (numberOfViews <= 0) return null;` — **confirmed present, unchanged since the 2026-05-16 fix.** Matches the brief's prime-suspect description exactly. The guard collapses three distinct backend/network conditions into one indistinguishable absent state: (i) endpoint returned `0`; (ii) endpoint returned a falsy/non-number that coerces to `<= 0`; (iii) endpoint errored and the catch wrote `0`.
- Owner-gating / hydration flag: **none.** Unlike `ProductFunctions` and `UserDetails` (the two files `issues.md` notes for the "owner-view hydration flash" pattern on 2026-05-14), `NumberOfViews` does not consult `useAuthResolved`, does not subscribe to `useAuthStore`, and does not differ by viewer. Every viewer sees the same render path.
- Effect dependency: `[productId]` — single fetch per mount per productId, no re-fetch loop. This is the post-2026-05-16 shape.

**Failure-to-zero swallow:** explicit, in two places. Line 17 catches client-side rejection and writes `0`. The fetch helper at `src/lib/service/reactCalls/productService.ts:357–360` catches its own rejection and `return 0`. The 2xx-but-out-of-range branch at line 355–356 also returns `0`. Net: every non-200-with-positive-number response path resolves the promise to `0`. A backend failure is therefore visually identical to a real zero-views product.

### §2 — Increment side

**Call:** `src/lib/service/nextCalls/productService.ts:6-12`:

```ts
export const markAsSeen = async (productId: number) => {
  try {
    await FETCH_BACKEND_API.get('/public/product/seen/' + productId);
  } catch (err) {
    logServiceError('product.next.markAsSeen', err);
  }
};
```

- Wrapper: `FETCH_BACKEND_API` (the RSC-side `fetch` helper at `src/lib/config/fetchApi.ts`), not the axios `BACKEND_API`. So this call is server-side.
- **No `skipAuth: true`.** Per `fetchApi.ts:38-45`, that means the helper calls `await cookies()` and forwards any cookies + `Authorization: Bearer <token>` to the backend. Two consequences: (a) the call carries the viewer's identity to the backend (relevant if backend has any per-viewer logic on `/seen/<id>`), and (b) the call dynamizes the surrounding route — this is the mechanism the prior skipauth-footprint audit (§3, route 2) relied on to declare the product page "already dynamic."
- **No `next.revalidate`, no `cache: 'no-store'`.** Next 15 defaults `fetch` to uncached, so this fires once per server render.
- Fire-and-forget: result is discarded, errors are caught and logged, no return value reaches the page.

**Gate:** `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:93-95`:

```ts
const owner: UserInfoDTO = await getUserForId(productDetails.ownerId);

if (!owner.iamActive) {
  markAsSeen(productDetails.id);
}
```

- `owner.iamActive` comes from `getUserForId(productDetails.ownerId)`. Per `src/lib/service/nextCalls/userService.ts:13-20`, that call uses `skipAuth: true` — this is the known #8 issue. With `skipAuth: true`, the backend has no viewer identity, so `iamActive` (a viewer-dependent field — see `UserInfoDTO.ts:13`) is computed against no viewer and gets defaulted by the backend.
- **Direction of the default is unverified web-side** — depends on backend code, which is out of scope per the brief. Two sub-scenarios:
  - If backend defaults `iamActive = false` for no-viewer requests: `!false = true` → `markAsSeen` fires on every render for every visitor, including the actual owner viewing their own product (over-counts; would over-increment, not under-increment).
  - If backend defaults `iamActive = true` for no-viewer requests: `!true = false` → `markAsSeen` **never fires for any visitor** → no counts ever incremented → fetched count is always 0 → guard hides element. This sub-scenario produces the exact reported symptom across all visitors and all products.
- The skipauth-footprint audit inferred "almost certainly false" (its §3 footnote: "The exact backend default for `iamActive` when no viewer identity is attached is inferred (almost certainly `false`) but not verified by reading backend code"). That inference reasons from web-visible behaviour ("the product page seems to act dynamic for everyone," which requires `!owner.iamActive` to be true for everyone). But that same reasoning produces a contradiction in the present audit: if `!owner.iamActive` is true for everyone, then `markAsSeen` is firing for everyone, and the count should not be uniformly zero — unless `/public/product/seen/<id>` doesn't actually persist anything backend-side. So either:
  - **scenario 2a**: backend defaults `iamActive=true` (inverting the skipauth audit's inference) — then `markAsSeen` is gated off entirely and counts never accrue; or
  - **scenario 2b**: backend defaults `iamActive=false`, `markAsSeen` does fire, but the backend `/seen/<id>` endpoint doesn't persist a counter that `/views/<id>` reads (different counter, dead endpoint, or no-op handler).
- Both scenarios are backend-resolvable, both produce the exact reported symptom, and both flow through the same web-side render-gate multiplier in §1.

### §3 — The endpoint contract

- Increment endpoint (write): `GET /public/product/seen/<productId>`. Called from `markAsSeen` (RSC, via `FETCH_BACKEND_API`, cookies + Bearer forwarded, no `skipAuth`).
- Display endpoint (read): `GET /public/product/views/<productId>`. Called from `getNumberOfViews` (`src/lib/service/reactCalls/productService.ts:347-361`) on the client via the axios `BACKEND_API`. The axios response interceptor at `src/lib/config/api.ts:44-49` normalises both "no response" and `404` into `Promise.reject({ data: { errorCode: 'not.found' }, status: 404 })`. That rejection lands in `getNumberOfViews`'s catch (line 357), which logs and returns `0`. So a missing-endpoint backend would silently produce the absent symptom on every product page render.
- **Whether `/seen` and `/views` read/write the same counter is not determinable from `oglasino-web`.** No DTO field, no comment, no shared client type ties them. The display fetch reads `res.data` as a raw number — there's no `{ count: ... }` shape, just a bare `number`. If the backend stores views on the product row and `/views/<id>` reads from there while `/seen/<id>` writes to a separate counter (or no-ops), the two endpoints would silently disagree. Backend audit required to confirm both endpoints touch the same field.
- No field on `ProductDetailsDTO` carries the view count. The DTO embeds `numberOfFavorites: number` (line 11) but not `numberOfViews`. So the dedicated client-side fetch is the only display source — no SSR-seeded fallback exists.

### §4 — Which link is broken

The visible symptom (**Absent** for all users on all products) requires two things to be true simultaneously:

1. The backing value `getNumberOfViews` resolves to is `<= 0` (either `0`, `null`, or a rejection that the catch coerces to `0`).
2. The render guard at `NumberOfViews.tsx:20` collapses any such value into `return null`.

Item 2 is **confirmed present in code** and is the single multiplier that converts every flavour of upstream failure into invisible absence. The guard is mechanically responsible for the "absent for all users" symptom. Removing or weakening the guard would not by itself surface a useful count — but it would immediately distinguish "fetched 0" from "fetched 5" from "fetch failed," which is currently impossible from the user's perspective.

Item 1 has two plausible upstream causes, both backend-resolvable, both consistent with the reported symptom and indistinguishable from each other web-side:

- **(a) never incremented** — most likely. The increment gate `!owner.iamActive` (page.tsx:93) reads from a field that `getUserForId`'s `skipAuth: true` (issue #8) renders non-authoritative. If backend defaults `iamActive=true` for no-viewer requests, `markAsSeen` is suppressed for every visitor, counts never accrue, fetch returns 0, guard hides. This is the most economical hypothesis: one backend default flipped vs. expectation, plus the existing #8 footprint, explains the symptom across all users and all products. The prior skipauth-footprint audit's hedge ("almost certainly false") was inferred from a separate symptom (route dynamism) and is in tension with the present symptom — present audit re-evaluates and treats either direction as live.
- **(d) fetched-and-rendered-but-always-0 because of an error-swallow or backend gap** — also live. If `/public/product/views/<id>` returns 404 (endpoint not implemented), or returns 0 because the `/seen/<id>` writer doesn't actually persist, the catch-and-return-0 paths in `getNumberOfViews` (lines 357-360) silently zero the value and the guard hides.

**Single most likely root cause:** the render guard at `src/components/client/NumberOfViews.tsx:20` (`if (numberOfViews <= 0) return null`) is the **proximate** cause of the visible "absent for all users" symptom. It converts every upstream failure mode — whether (a) never-incremented, (b) increment-but-broken-read, (c) fetched-but-rendered-as-null/undefined that coerces to `<= 0`, or (d) genuine zero on a new product — into the same indistinguishable absence. **The smallest fix shape that surfaces the underlying cause** is to weaken the guard so that 0 renders visibly:

```tsx
if (numberOfViews < 0) return null;   // was: <= 0
```

(Or alternatively: drop the guard entirely and accept "0 views" as a valid render.) After this change, the user-visible behaviour distinguishes "0 views" from "no views element at all" and makes the underlying issue diagnosable without a backend round-trip. Then a follow-up backend audit can determine whether (a) or (d) is the actual upstream cause and apply the structural fix there.

**Backend involvement suspected — yes.** The full fix requires either (a) ensuring `markAsSeen` actually fires for non-owner visitors regardless of the skipauth-induced `iamActive` default — which means either flipping the gate's semantics (e.g., gating on the authenticated viewer's id from the auth store, not on the owner's `iamActive`), or fixing #8 so `iamActive` is authoritative — or (d) verifying that `/public/product/seen/<id>` actually persists a counter that `/public/product/views/<id>` reads. Backend code is out of scope per this brief.

## Cleanup performed

- none needed (read-only audit, no code changes)

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change. This audit's findings are draft material for Mastermind to triage into a new `issues.md` entry (the bug is NEW per the brief and not yet in `issues.md`). The drafter for that entry is the bug chat or Mastermind, not this engineer agent. Per Part 3 / conventions Part 5 closure gate: nothing is drafted here for Docs/QA to apply directly; the next-action is a Mastermind triage, not a Docs/QA edit.

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): confirmed — read-only audit, no commented code, no unused imports, no console.log, no TODO/FIXME added.
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): flagged in "For Mastermind"
- Part 6 (translations): N/A this session
- Part 7 (error contract): N/A this session (no validation surface touched)
- Part 11 (trust boundaries): N/A — view counter is not a moderation, authorisation, or state-transition decision input.
- Other parts touched: none

## Known gaps / TODOs

- **The direction of the backend `iamActive` default for no-viewer (skipAuth) requests is unverified web-side.** Both directions produce the same web-visible symptom in this audit (because the render-gate multiplier hides the difference), but the structural fix path differs. Resolving this requires reading backend code (out of scope per the brief).
- **Whether `/public/product/seen/<id>` and `/public/product/views/<id>` touch the same counter is not determinable from oglasino-web.** No shared DTO, no shared client type, no comment ties them. Confirming requires backend.
- **Whether any historical product has a non-zero view count** is unverified. If `markAsSeen` has been gated off since the skipauth-induced `iamActive` default landed, every product created since would have 0 views forever; older products may carry pre-bug counts. A DB query would distinguish — out of scope.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit; no code added).
  - Considered and rejected: nothing (no implementation decisions made).
  - Simplified or removed: nothing (no code removed).

- **Single named root-cause verdict (required by brief Definition of Done):** the render guard at `src/components/client/NumberOfViews.tsx:20` (`if (numberOfViews <= 0) return null`) is the proximate cause of the visible "absent for all users" symptom. It is the multiplier that converts any upstream zero (real-zero, fetch-failure-to-zero, or never-incremented-to-zero) into indistinguishable invisibility. Smallest fix shape: change `<= 0` to `< 0` (one-character edit) so "0 views" renders visibly. Backend-side root cause (one of: skipauth-induced `iamActive` default suppressing `markAsSeen`, or `/seen` not persisting what `/views` reads) is suspected but out of scope per the brief — flag for a backend audit follow-up.

- **Overlap with skipAuth #8 (recorded, not folded):** the increment gate's reliance on `owner.iamActive` from a `skipAuth: true` fetch is structurally identical to the prior skipauth-footprint audit's row 6. The two audits read in the same direction — `iamActive` is non-authoritative — and the chains overlap at this gate. The bug discussed here would persist even after a #8 fix that re-attaches auth to `getUserForId`, **if** the backend gap (scenario 2b: `/seen` doesn't increment what `/views` reads) is the real upstream cause; conversely, a #8 fix would resolve the bug for authenticated owner-viewers correctly (they'd be excluded from the count) and for authenticated non-owner viewers correctly (they'd be counted). Anonymous viewers remain ambiguous and depend on backend default. Recording the overlap so the two fix tracks are not collapsed into one.

- **Adjacent observation — `markAsSeen` lacks `skipAuth: true` (severity: low, already flagged in prior audit).** The prior skipauth-footprint audit's "Adjacent observation — `markAsSeen` should arguably pass `skipAuth: true`" applies here. The endpoint is `/public/...`, the call is fire-and-forget, and forwarding cookies + Bearer to a public endpoint is structurally noisy. Re-flagging because this audit confirms `markAsSeen` is on the read path that produces the bug — any future re-evaluation of the view-counter chain should treat the `skipAuth` posture of `markAsSeen` as part of the same surface as `getUserForId`'s posture.

- **Adjacent observation — the increment gate's semantics are inverted vs. the cleanest reading (severity: low).** The intent of `if (!owner.iamActive) markAsSeen(...)` is presumably "don't increment views when the owner is viewing their own product." But the gate reads from the **owner's** profile DTO and asks "is the looked-up profile the actively-signed-in user." Two equivalent fixes that are more robust to the #8 class of bug: (i) read the viewer's id from `useAuthStore` on the client and POST a `viewerUserId` (or omit when anonymous) so the backend can decide; (ii) drop the gate entirely and let the backend decide based on its authenticated identity (which would require `markAsSeen` to attach auth — already the case since it lacks `skipAuth: true`). Either restructures the trust boundary to the server (conventions Part 11) and removes the web-side gate's exposure to `iamActive` defaulting. I did not propose a specific fix because the cleanest shape depends on backend behaviour that's out of scope.

- **Adjacent observation — `getNumberOfViews` returns `res.data` as a bare `number` with no type assertion (severity: very low).** `src/lib/service/reactCalls/productService.ts:352` returns `res.data` without checking that the response body is actually a number. If the backend ever returns a `{ count: N }` shape or a string, `numberOfViews` would carry a non-number value that compares with `<= 0` unpredictably (`'0' <= 0 → true`; `{ count: 5 } <= 0 → false` and the JSX would render `[object Object]`). Not the cause of the present bug (the catch path covers most failure modes) but worth a defensive type-check at the boundary if the file is touched.

- **Process note — the skipauth-footprint audit's inference about `iamActive`'s default direction is in tension with the present symptom.** That audit reasoned from observed dynamism on the product page ("`markAsSeen` fires on every render → cookies() is read → route is dynamic") to infer `iamActive` defaults false. The present bug ("views absent for all users") is most parsimoniously explained by `iamActive` defaulting true (so `markAsSeen` never fires). Both can be reconciled if `markAsSeen` does fire but `/seen` is a no-op or writes to a counter `/views` doesn't read — that requires backend confirmation. Carrying both possibilities forward without collapsing.

- (nothing else flagged)
