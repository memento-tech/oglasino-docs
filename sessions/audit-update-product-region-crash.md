# Audit (read-only) — Update-product screen crash: `regionAndCity` undefined

**Repo:** oglasino-expo
**Branch:** new-expo-dev (read-only; no code changed, nothing staged)
**Date:** 2026-05-31
**issues.md ref:** 2026-05-31 — "Mobile: update-product screen crashes — regionAndCity undefined on productDetails"

Reported stack (on-device, not previously code-verified):
`TypeError: Cannot read property 'region' of undefined`
at `UpdateProductScreen` → `app/owner/dashboard/products/[productId].tsx:333`,
propagating `dashboard/_layout.tsx:11 → owner/_layout.tsx:42 → app/_layout.tsx:100`.

**Stack is confirmed against the code.** Line 333 reads `productDetails.regionAndCity.region.labelKey` raw; when `regionAndCity` is `undefined`, reading `.region` throws exactly the reported message.

---

## 1. Where `productDetails` comes from / loading gate

- State declared `undefined`: `[productDetails, setProductDetails] = useState<NewProductRequestDTO>()` — `app/owner/dashboard/products/[productId].tsx:49`.
- Source: `useEffect` (`:109-135`) calls `getDashboardProductDetails(parseInt(productId))` at `:112`; on success sets the **whole** object once at `:120` (`setProductDetails(data)`). Re-seeded after a successful save by `reseedFromServer` (`:177-187`, `setProductDetails(data)` at `:181`).
- Loading gate: `[loading, setLoading] = useState(true)` (`:52`); screen early-returns `null` while loading (`:267-269`). Render additionally gated on `{productDetails && (…)}` (`:291`).
- The region/city block (`:331-343`) is inside the `productDetails &&` block but is **not** gated on `regionAndCity` being present.

## 2. Crash site — raw dereference, no optional chaining

```
333  {t(productDetails.regionAndCity.region.labelKey)}/
334  {t(productDetails.regionAndCity.city.labelKey)}
```

Both dereference `regionAndCity` raw (no `?.`, no guard). The whole object is passed safely to `CitySelector` at `:337`. The only raw dereferences are `:333-334`.

## 3. Type of `productDetails` / `regionAndCity`

- `productDetails: NewProductRequestDTO` (`:49`).
- `NewProductRequestDTO.regionAndCity: RegionAndCityDTO | undefined;` — **declared optional** — `src/lib/types/product/NewProductRequestDTO.ts:16`.
- `RegionAndCityDTO = { region: KeyLabelPair | null; city: KeyLabelPair | null }` — `src/lib/types/catalog/RegionAndCityDTO.ts:6-9`. Both the object and its `region`/`city` members are nullable in the type.

## 4. Network source / response shape

- `getDashboardProductDetails` → `GET /secure/products?productId=${productId}` — `src/lib/services/productsSearchService.ts:36-38`, `:13-30`. Returns `res.data` (untyped) cast to `Promise<UpdateProductRequestDTO>`.
- `UpdateProductRequestDTO extends NewProductRequestDTO` — `src/lib/types/product/UpdateProductRequestDTO.ts:5` — inherits `regionAndCity: RegionAndCityDTO | undefined`.
- No runtime shape validation between `res.data` and state; the response is set straight into `productDetails`. The DTO models `regionAndCity` as possibly-absent.

## 5. Verdict — (a), with nuance

**(a): `regionAndCity` is absent (or null) in the response payload for the crashing product — a data-shape exposure, NOT a render-before-hydration race.**

Evidence:
- `productDetails` is set exactly once with the complete server object (`:120`); nothing sets a partial object (`handleChange` only merges user edits, `:137-139`).
- Render is double-gated: `loading===false` (`:267-269`) and `productDetails` truthy (`:291`). There is no window where the block renders against a half-hydrated object.
- Therefore at `:333` `productDetails` is the full response; `regionAndCity` being undefined means the GET `/secure/products` payload lacked it (or sent it null). The `| undefined` type confirms absence is a modeled possibility the render fails to handle.

(b) is ruled out: no partial/streamed hydration path exists.

> Cross-repo caveat: *why* the backend omits `regionAndCity` for this product (legitimate for some product states vs. a backend regression) is a Backend question — see "For Mastermind". Mobile evidence only shows the field was absent post-load and the render is unguarded.

## 6. Adjacent exposures (same render, same failure mode)

- **Same lines, deeper:** `region`/`city` are `KeyLabelPair | null` (RegionAndCityDTO:7-8). Even a present `regionAndCity` with a null `region` or `city` crashes at `.labelKey` ("Cannot read property 'labelKey' of null"). `:334` (`.city`) has identical exposure to `:333` (`.region`).
- **`imageKeys` — separate raw deref:** `productDetails.imageKeys.map(...)` at `:296`, where `imageKeys?: string[]` is optional (`NewProductRequestDTO.ts:18`). Earlier line in the same render; would throw the same way if `imageKeys` is undefined (fetch guards it at `:123`, render does not — and `:296` would crash *before* `:333` if both absent).
- `currency` (`:375-376`) is typed required (`currency: CurrencyDTO`); lower risk and gated under the `topCategory && !freeZone` branch (`:345`).

---

## For Mastermind

1. **Render bug (mobile, in-scope):** `[productId].tsx:333-334` dereferences `productDetails.regionAndCity.region.labelKey` / `.city.labelKey` raw, against a field typed `RegionAndCityDTO | undefined` whose `region`/`city` are themselves `| null`. Three nullable hops, zero guards. This is the direct cause of the reported crash. (No fix proposed per brief — flagging only.)
2. **Backend question (cross-repo):** is `regionAndCity` legitimately absent/null on a valid product's `GET /secure/products` details response, or is its omission a backend regression? Mobile can't answer from this repo. The fix shape on mobile depends on the answer: if legitimately optional, the render must degrade gracefully (hide/placeholder the region line); if it should always be present, this is also a backend contract bug worth a Backend issue.
3. **Web parity:** worth confirming whether the web update-product screen guards this same field (it consumes the same endpoint). If web guards it and mobile doesn't, this is a straightforward mobile-only render hardening.
4. **Adjacent latent crash:** `imageKeys.map` at `:296` is the same unguarded-optional pattern and a candidate to harden in the same pass.

## Config-file impact

No edit to any of the four config files. This is a read-only audit; it adopts/retires no `state.md` Expo-backlog row. The follow-up belongs in `issues.md` (already tracked there as the source line) — Docs/QA owns any edit. No closure-gate dependency surfaced.
