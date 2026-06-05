# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-06-01
**Task:** Audit (read-only): determine the exact JSON web POSTs to `/public/product/search` for the
region/city portion of the body — outer key, inner shape, and where (if anywhere) the UI's selected
region/city objects are mapped to the posted shape.

## Implemented

- Read-only audit. No code changed. Traced the region/city filter from server-side hydration
  (`filterHydrationSSR` → `parseFiltersFromQueryParams` → `getRegionAndCitiesData`) through to the actual
  POST body in both the `nextCalls` (SSR/first-page) and `reactCalls` (client-pagination) search services.
- Conclusion: web POSTs full objects under outer key **`selectedRegionsAndCities`** with inner arrays
  **`regions`** / **`cities`** (objects carrying `id` + `labelKey`; regions additionally carry a nested
  full `cities[]`). It does **not** send `selectedRegionAndCityValues`, and it does **not** send
  `regionIds`/`cityIds` number arrays.
- There is **no object→ID mapping step** anywhere on the web path. The hydrated objects are serialized
  as-is; the reactCalls variant only shallow-spreads `{ ...filtersData }` and sends the same shape.
- The web TS type and the web wire body **agree**; both **disagree** with the backend read's claim of
  `selectedRegionAndCityValues.{regionIds,cityIds}`. That cross-repo divergence is the finding; resolving
  it is backend-side and outside this web audit.
- Wrote `.agent/audit-region-city-wire-web.md` answering the three points with file:line.

## Files touched

- `.agent/audit-region-city-wire-web.md` (new audit artifact; +/- N/A)
- `.agent/2026-06-01-oglasino-web-catalog-filters-2.md` (this summary)
- `.agent/last-session.md` (exact copy of this summary)
- No source files touched. No git writes.

## Tests

- Ran: none. Read-only audit; no code changed, so `npm run lint` / `npx tsc --noEmit` / `npm test` were
  not required for any touched source path (none exist).
- Result: N/A
- New tests added: none

## Cleanup performed

- none needed (no code changed)

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change — but see "For Mastermind": a cross-repo region/city wire-shape divergence was
  confirmed (web sends `selectedRegionsAndCities` objects; backend read claimed
  `selectedRegionAndCityValues.{regionIds,cityIds}`). This is a Phase-3 seam-analysis input for Mastermind,
  not an `issues.md` entry I am authoring. No config-file edit is required by this audit itself.

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): confirmed — no code added, nothing to clean.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one observation flagged in "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: Part 11 (trust boundaries) — noted that `RegionDTO`/`CityDTO` ids on the wire are
  client-supplied foreign keys the server must validate against its own data; not an issue, just the
  relevant boundary. Part 10 (feature lifecycle) — this is a Phase-2 read-only audit feeding Phase-3.

## Known gaps / TODOs

- The audit answers only the **web** side (what web puts on the wire), per the brief's scope. How the
  backend actually consumes that shape — and whether the backend read was of a renamed/different DTO — is
  not resolved here and is explicitly a backend/Mastermind question.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **The seam finding (this is the answer to the brief's contradiction):** Web's region/city filter type
  AND its serialized POST body agree with each other — both use outer key `selectedRegionsAndCities` with
  inner `regions: RegionDTO[]` / `cities: CityDTO[]` (full objects: `id` + `labelKey`, regions also carry
  nested `cities[]`). There is no rename/reshape between web's type and web's wire. So the contradiction is
  **not** "web type says X but web wire sends Y"; it is **web (objects under `selectedRegionsAndCities`)
  vs. the backend read (`selectedRegionAndCityValues` with `regionIds`/`cityIds` `List<Long>`)**. Since web
  filtering works in production, the live backend must be binding the object shape under
  `selectedRegionsAndCities` (reading `.id` off the objects), or the backend read was of a different DTO.
  Recommend a targeted backend re-check of the DTO that `/public/product/search` actually binds before the
  spec freezes this field's shape.

- **Part 4b adjacent observation (low):** `RegionDTO` carried in
  `selectedRegionsAndCities.regions[]` ships its full nested `cities[]` array on every search POST
  (`src/lib/types/catalog/RegionDTO.ts`; built in `filtersHelper.ts:223-244`). If the backend only needs
  region/city ids, web is sending materially more JSON than required (every city of every selected region,
  with labelKeys). File: `src/lib/utils/filtersHelper.ts:223-244`. Severity low (works today; payload
  bloat, not a bug). I did not change this — out of scope for a read-only audit, and trimming the wire
  shape is a contract decision that belongs to the spec/seam analysis, not this session.

- No drafted config-file text. No config-file edit is required by this audit.
