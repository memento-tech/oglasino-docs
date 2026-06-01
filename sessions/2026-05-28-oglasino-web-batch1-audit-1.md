# Session summary

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-28
**Task:** Audit brief — Batch 1: two-redirect chain + malformed-429 wizard block

## Implemented

- Read-only audit of two well-described `issues.md` entries against current code. No edits.
- Confirmed both mechanisms still hold; both line numbers in the entries are still accurate.
- Identified the smallest cookie-aware single-redirect shape for Part 1.
- Sized the two fix options (client guard vs leave-to-backend) for Part 2 and recommended one.

## Files touched

- (none — read-only audit)

## Tests

- Ran: (none — read-only audit)
- Result: N/A
- New tests added: none

## Cleanup performed

- none needed

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change (this audit is the prep for a fix brief; neither entry's status flips here)

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): N/A (read-only)
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): one observation flagged in "For Mastermind"
- Part 6 (translations): N/A this session
- Other parts touched: none

## Known gaps / TODOs

- none

---

# Audit

## Part 1 — Two-redirect chain (issues.md 2026-05-22)

### Confirmation that the entry still describes the code

`proxy.ts` is the middleware file (Next.js 16 naming, `proxy.ts` only — no `middleware.ts` in the tree). The two branches the entry names are both present and structurally identical to how the entry describes them.

**Invalid-locale fallback branch:** `oglasino-web/proxy.ts:41-46`

```ts
if (ALLOWED_LANGS_PER_TENANT[tenant] && !ALLOWED_LANGS_PER_TENANT[tenant].has(urlLang)) {
  const fallbackLang = [...ALLOWED_LANGS_PER_TENANT[tenant]][0] as Lang;
  const redirectUrl = new URL(request.url);
  redirectUrl.pathname = getLocalizedPath(pathname, fallbackLang);
  return NextResponse.redirect(redirectUrl);
}
```

**Cookie-wins branch:** `oglasino-web/proxy.ts:48-65`

```ts
const raw = request.cookies.get(GLOBAL_COOKIE_NAME)?.value;
if (raw) {
  try {
    const parsed = JSON.parse(decodeURIComponent(raw)) as { lang?: Lang };
    const cookieLang = parsed.lang;
    if (
      cookieLang &&
      cookieLang !== urlLang &&
      ALLOWED_LANGS_PER_TENANT[tenant]?.has(cookieLang)
    ) {
      const redirectUrl = new URL(request.url);
      redirectUrl.pathname = getLocalizedPath(pathname, cookieLang);
      return NextResponse.redirect(redirectUrl);
    }
  } catch {
    // Malformed cookie - fall through to next-intl, URL wins.
  }
}
```

`ALLOWED_LANGS_PER_TENANT` is derived once at module init from `routing.locales` in `src/i18n/routing.ts` (currently produces `{rs: {sr,en,ru}, rsmoto: {sr,en,ru}, me: {sr,cnr,en,ru}}` — set iteration order is insertion order, so the "tenant default" the invalid-locale branch returns is whatever locale appears first per tenant in `routing.locales`, i.e. `sr` for `rs` and `rsmoto`, and `sr` for `me`).

### Hop trace for the entry's input

Input: request to `/rs-cnr/...`, `cookie.lang='ru'`. Tenant `rs` allows `{sr,en,ru}`.

**Hop 1.** `proxy.ts:33` matches `LOCALE_PREFIX` (`tenant='rs'`, `urlLang='cnr'`). Line 41 fires (`'cnr'` not in `{sr,en,ru}` for `rs`). `fallbackLang` = first allowed = `'sr'`. Returns `308`/`307` redirect to `/rs-sr/...`. Cookie is **not consulted** on this hop.

**Hop 2.** Browser follows the redirect. New request to `/rs-sr/...`. Line 41's check is false (`'sr'` is in `{sr,en,ru}`). Falls through to line 48. Cookie parses, `cookieLang='ru'`, `'ru' !== 'sr'`, `'ru'` is in `{sr,en,ru}` for `rs`. Lines 58-60 fire. Redirect to `/rs-ru/...`.

**Hop 3.** Browser follows. Request to `/rs-ru/...`. Line 41 false (`'ru'` valid). Line 48 reads cookie, `cookieLang='ru'`, `'ru' === urlLang`, branch does nothing. Falls through to `defaultMiddleware`. Render.

Two redirects (hops 1→2 and 2→3) before render. Each hop alone is correct. The waste is that hop 1 throws away the cookie information that hop 2 then re-reads.

### Smallest single-redirect fix

The invalid-locale fallback at `proxy.ts:41-46` is cookie-blind: it always picks the first allowed lang regardless of the user's cookie preference. Making it cookie-aware (prefer cookie's lang if the tenant supports it, else fall back to the first allowed lang) collapses the chain to one hop.

Two viable shapes; the second is cleaner:

**Shape A — inline cookie peek in the invalid-locale branch.** Copy the cookie-parse logic from lines 48-65 into the invalid-locale branch and pick the cookie's lang if it's allowed for the tenant. Minimal diff but duplicates the JSON-parse/try-catch logic.

**Shape B — hoist the cookie parse to a single read at the top of the locale-handling block.** Compute `cookieLangIfValidForTenant` once, then:
- Invalid-locale branch uses `cookieLangIfValidForTenant ?? [...allowed][0]` as the redirect target.
- Cookie-wins branch only fires when `cookieLangIfValidForTenant && cookieLangIfValidForTenant !== urlLang`.

Shape B is shorter overall, removes the duplicated try/catch, and reads top-to-bottom as one decision tree.

After the fix, the entry's input collapses to:

- Hop 1. `/rs-cnr/...` matches, `urlLang='cnr'` invalid, cookie peek yields `'ru'` (valid for `rs`), redirect to `/rs-ru/...`.
- Hop 2. `/rs-ru/...` valid, cookie matches URL, render.

One redirect.

### Regression check on the cookie-absent / cookie-invalid case

The fix must preserve today's behavior when no cookie is present or the cookie's lang isn't valid for the tenant. Today's behavior is: fall back to the first allowed lang of that tenant. Both shapes preserve this (Shape A by inheriting the existing fallback line; Shape B by `?? [...allowed][0]`).

No tests exercise `proxy.ts` (no `proxy.test.*` in the tree). So there is no test surface to update or regress. Manual smoke would need to cover four cases on a tenant with at least two allowed langs:

1. URL lang invalid + no cookie → redirects to tenant default (no change).
2. URL lang invalid + cookie lang valid for tenant → redirects directly to cookie lang (the fix).
3. URL lang invalid + cookie lang not valid for tenant (e.g. malformed cookie or `lang='zz'`) → redirects to tenant default (no change).
4. URL lang valid + cookie lang differs from URL but is valid → cookie-wins branch fires (no change).

No other call sites consume the invalid-locale fallback — the constant `ALLOWED_LANGS_PER_TENANT` is private to `proxy.ts` and `getLocalizedPath` is consumed by `proxy.ts` only on these two redirect lines plus a few non-proxy callers that aren't affected by this change.

### Smallest fix — summary

File: `oglasino-web/proxy.ts`. Lines: `41-46` (invalid-locale branch) and `48-65` (cookie-wins branch). Smallest change: hoist the cookie parse once at the top of the `if (match)` block (Shape B), then make the invalid-locale fallback prefer the cookie lang when valid. ~10-line edit, no new helpers, no API surface change, no test surface to update.

---

## Part 2 — Malformed 429 blocks create wizard (issues.md 2026-05-14)

### `ensureSystemErrorKey` confirmation

`grep -rn "ensureSystemErrorKey" oglasino-web/` returns zero matches. The helper is gone, as the entry says. Nothing replaced it on the client side. The contract-trust posture is implemented at `parseProductValidationErrors` (`src/lib/utils/parseProductValidationErrors.ts:31-45`): the parser maps each `errors[]` entry by its `field` (or `SYSTEM_ERROR_KEY = '__system'` when `field === null`) and writes `err.translationKey` as-is.

### 429 handling path in `productService.ts`

Three call sites read a 429:

- `createNewProduct` (step 4 final submit): `src/lib/service/reactCalls/productService.ts:127-173`. 429 is in `PARSEABLE_ERROR_STATUSES` (line 41) and routed through `parseProductValidationErrors` on both the resolved-non-2xx branch (lines 151-154) and the thrown-with-status branch (lines 168-170). If the body isn't a `ProductErrorResponse` shape (the `isProductErrorResponse` typeguard at lines 43-49 checks for `Array.isArray(body.errors)`), the call falls through to `{type: 'error'}`.
- `updateProductData` (mirrors create): `src/lib/service/reactCalls/productService.ts:191-232`. Same shape, same fall-through.
- `preValidateProductBasics` (step 1 Next button gate): `src/lib/service/reactCalls/productService.ts:234-275`. Explicit 429 branches at lines 253-255 (resolved) and 270-272 (thrown). Both require `isProductErrorResponse(data)`; otherwise `{type: 'error'}`.

### Two distinct "malformed 429" shapes

The entry compresses two failure modes that the code handles differently:

**Mode α — body lacks an `errors[]` array entirely** (e.g. empty body, plain text, `{message: 'rate limited'}`).
`isProductErrorResponse` returns `false`. Service returns `{type: 'error'}`. In `BasicInfoProductDialog.onNextInternal` (lines 144-150), the `'error'` branch is `notify.warning` + `onNextStep()` — the user **advances** with a toast warning. **This case is not the one the entry describes** — the user is not blocked.

**Mode β — body has `errors[]` but no entry with `field: null + translationKey`** (e.g. `{errors: []}`, or `{errors: [{field: 'name', code: 'X', translationKey: 'product.name.x'}]}`, or `{errors: [{field: null, code: 'X'}]}` with `translationKey` missing).
`isProductErrorResponse` returns `true`. Service returns `{type: 'validation', errors: parseProductValidationErrors(body)}`. `result.errors.byField[SYSTEM_ERROR_KEY]` is undefined. In `BasicInfoProductDialog.onNextInternal` (lines 119-142):

- Line 120 `setProductErrors(result.errors.byField)` populates field-keyed errors (or clears them if `errors: []`).
- Lines 121-127 fire `form_submit_failed` analytics per entry.
- Line 128 `if (result.errors.byField[SYSTEM_ERROR_KEY])` is **false** → no `isRateLimited` flag set, no timer started.
- Line 141 `return;` — `onNextStep()` is **not** called.

User result: the function returned without advancing AND without setting `isRateLimited`. The Next button (lines 305-312) is `disabled={preValidating || isRateLimited}` — neither is true after the early return, so the button is technically clickable. But clicking again hits the same 429, same no-op, same silent return. From the user's perspective the wizard is **stuck on step 1 with no inline rate-limit reason** — the entry's "Next button blocked" is functionally accurate even though the button isn't literally disabled.

If the malformed body happens to populate a recognizable field key (e.g. `field: 'name'`), the user sees a misleading per-field error message until the next keystroke. If the body is `{errors: []}`, the user sees nothing.

### Consuming components and gate

The wizard step 1 gate lives in `src/components/popups/components/BasicInfoProductDialog.tsx`:

- Pre-validate result handling: lines 109-151 (`onNextInternal`).
- Rate-limit timer / `isRateLimited` state: lines 70-77, 128-140.
- System-error inline rendering: lines 292-296.
- Next button disabled state: lines 305-312.

The step 4 final-submit path (`UploadedProductDialog.tsx:52-118`) handles the same shape differently — it always lands in either the success or failure state and renders the field-keyed errors as a bulleted list (`UploadedProductDialog.tsx:189-208`). On a Mode-β malformed 429 at step 4, the user would see either an empty error list with the generic failure header (if `errors: []`) or the misleading field-keyed entries (if non-null `field` values present). Step 4 is less affected because the wizard ends in a failure UX either way, but the inline rate-limit message is still missing.

### Two fix options

**Option A — thin client-side guard (reintroduce something like `ensureSystemErrorKey`, or add a 429-specific fallback).**

Shape: in `productService.ts`, when the response status is 429, if the parsed validation result has no `SYSTEM_ERROR_KEY` entry, synthesize one with the canonical translation key `product.system.rate_limited` (the key already exists in the backend seed and is in the existing tests at `productService.test.ts:110, 120, 122`).

Cheapest surface: a small post-parse normalizer on the 429 branch only, in the three 429-handling call sites in `productService.ts` (`createNewProduct` lines 151-154 and 168-170, `updateProductData` lines 208-211 and 227-229, `preValidateProductBasics` lines 253-255 and 270-272). Alternatively, fold it into `parseProductValidationErrors` parameterized by status (cleaner but touches a shared util).

Blast radius:
- Affects only the 429 path. 400/422/403/500 still demand a contract-shaped body.
- One translation key (`product.system.rate_limited`) is the universal fallback. No new keys needed.
- Hides backend bugs: if backend ever emits a malformed 429, the test suite will pass and QA may never notice. (This is the reason it was removed in web session 4.)
- Re-introduces a piece of dead code that was deliberately deleted — visible as a regression in `decisions.md`/session history.

**Option B — leave to backend, no client-side change.**

Shape: file a backend bug when a malformed 429 is first observed in QA or prod; fix at the source. Document the current behavior (user silently stuck) as the accepted UX cost of contract discipline. This is what the `issues.md` entry's "the fix belongs at the backend/transport source" line already says.

Blast radius:
- Zero code change today.
- Pre-launch the backend 429 emission is well-tested and the contract is unified through `RateLimitFilter` (decisions.md / state.md confirm 2026-05-13 backend session 1). No known production malformed-429 incident.
- Risk: if a malformed 429 does ship (e.g. a future infrastructure layer — Cloudflare router worker, edge rate limiter, reverse proxy — emits a 429 that doesn't pass through the unified Spring `RateLimitFilter`), the user-visible failure mode is silent. The edge router worker is a real candidate here per `meta/conventions.md` Part 8 ("The Cloudflare router worker is the edge boundary").

### Recommendation

**Option B — leave to backend — with one caveat.** The web session 4 rationale (trust the contract, don't paper over backend bugs) was a deliberate engineering decision logged in `decisions.md`, not an accident. The `RateLimitFilter` produces the unified shape today and there is no known live incident. Re-introducing a synth in this single path would set a precedent that "we synthesize entries when the wire shape is off," which corrodes the contract discipline that made the validation feature ship cleanly.

The caveat: if a malformed 429 is observed once from a non-backend emitter (most plausibly the Cloudflare router worker on a hot path), Option A becomes the right call **for the worker-originated path only**, not as a general escape hatch. At that point the cleanest shape is to scope the synth narrowly: status 429 + empty/malformed body → synth `product.system.rate_limited` under `SYSTEM_ERROR_KEY` — and to document the gap so a future engineer reads the synth as "this exists because the worker's rate-limit response doesn't pass through `RateLimitFilter`," not as "we're defensive everywhere."

If Mastermind decides Option A is the immediate call regardless: the shape is a ~5-line post-parse normalizer on the 429 branches in `productService.ts`. Lower blast radius and less surface than going into `parseProductValidationErrors`.

---

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit).
  - Considered and rejected: nothing (no code work).
  - Simplified or removed: nothing (no code work).

- **Part 1 fix-brief shape.** When Mastermind writes the Part 1 fix brief, Shape B (hoist the cookie parse once and reuse it for both branches) is what I'd recommend instructing the engineer to use. Shape A is acceptable but duplicates the try/catch. Both produce identical user-visible behavior.

- **Part 2 recommendation is Option B with the caveat in the audit.** The brief asked me to "report both options with blast radius — don't decide" earlier, and "ends with a recommendation with reasoning" in the definition of done. I've recommended Option B (leave-to-backend) with the caveat that if a malformed 429 is ever observed from a non-Spring emitter (router worker most plausibly), Option A becomes correct narrowly scoped to that path. Mastermind to confirm.

- **Adjacent observation (Part 4b, low severity).** `parseProductValidationErrors` (`src/lib/utils/parseProductValidationErrors.ts:38-43`) writes `err.translationKey` into `byField` without checking it's a non-empty string. If backend ever emits a malformed entry like `{field: null, code: 'RATE_LIMITED'}` (no `translationKey`), `byField[SYSTEM_ERROR_KEY]` becomes `undefined`. Today's consumers all do truthy checks (`if (productErrors[SYSTEM_ERROR_KEY])`), so the visible effect is "nothing renders" — not a crash. Worth flagging because it's the exact code path that would intersect with Option A above if it's ever chosen, and because the parser silently accepting partial entries is the seam where the contract discipline weakens. I did not fix this because it is out of scope.

- **No config-file impact this session.** No drafts pending for `conventions.md`, `decisions.md`, `state.md`, or `issues.md`. The two `issues.md` entries this audit covers will need status updates when their fix briefs land — those drafts belong to the engineer sessions that implement the fixes, not to this audit.
