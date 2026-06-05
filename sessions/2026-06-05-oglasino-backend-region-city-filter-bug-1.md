# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-06-05
**Task:** READ-ONLY backend audit — diagnose why region/city product filters do not work
(broken identically on web + mobile). Write findings to `.agent/audit-region-city-filter-bug.md`,
answering Q1–Q6 with file:line evidence. No fixes.

## Implemented

- Traced the full backend region/city path: data-serving endpoint
  (`GET /api/public/baseSite/regions/{code}`) and all apply endpoints (public/dashboard/admin
  `POST …/product search`) → facade → service → `buildQuery` → `RegionAndCityQueryGenerator`.
- Verified the predicate is correctly built and added (filter+bool+should+minShouldMatch=1 on ES
  fields `regionId`/`cityId`), the ES doc fields exist + are populated from entity IDs, the handler
  is registered in the autowired `QueryGenerator` list, and regions/cities are seeded + linked to
  base sites.
- Empirically confirmed the one cosmetic divergence (`FieldValue.of(Object)` vs sibling
  `.value(Long)`) is harmless: compiled both against the project's `elasticsearch-java 9.2.6` jar
  and serialized them — byte-identical (`{"term":{"regionId":{"value":5}}}`).
- Verdict: backend is correct end to end; the only silent-drop path is
  `selectedRegionAndCityValues == null` (`RegionAndCityQueryGenerator.java:19`). Most-likely root
  cause is the wire contract (clients not sending `selectedRegionAndCityValues.{regionIds,cityIds}`)
  — a seam to confirm against the web/expo audits — or a stale ES index. Not decidable from backend
  code alone.

## Files touched

- `.agent/audit-region-city-filter-bug.md` (new, deliverable) — no source code changed (read-only).

## Tests

- Ran: none of the project suite (read-only audit; no code changed).
- Ad-hoc: compiled + ran a 1-file ES-client serialization check in `/tmp` (off-tree, deleted-able)
  to prove `FieldValue.of(Object)` ≡ `value(Long)`. Result: identical term JSON.

## Cleanup performed

- none needed (read-only; the only artifact is the audit file. The `/tmp/fvcheck` scratch dir is
  outside the repo).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change (this is a Phase-2 audit feeding seam analysis; no new out-of-scope issue
  authored. The three Part 4b notes below are audit findings for Mastermind to triage, not yet
  issues.md entries — flagged, not drafted, per read-only scope.)

## Obsoleted by this session

- nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code added; audit file only.
- Part 4a (simplicity): N/A — read-only, no code introduced (see "For Mastermind").
- Part 4b (adjacent observations): three flagged below + in the audit.
- Part 6 (translations): N/A this session.
- Other parts touched: Part 10 (feature lifecycle) — this is a Phase-2 audit; Part 11 (trust
  boundaries) — confirmed: filter IDs are entity FKs the server validates against its own index,
  and `DASHBOARD_SEARCH` already ignores client `ownerId` (`DefaultProductsFilterQueryBuilder
  :104-112`); no trust-boundary issue in the region/city path.

## Known gaps / TODOs

- The root cause cannot be confirmed backend-side: it requires (a) the web + expo request-body
  audits to verify the wire contract, and (b) an inspection of the live `products` index
  mapping/contents. Both are outside this repo and this read-only scope.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit; no code).
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **Primary finding / next step:** Backend region/city filter is provably correct (predicate +
  mapping + handler + data + serialized query all verified). The symptom's only backend cause is
  `selectedRegionAndCityValues` arriving null. **Recommend Phase-3 seam check:** diff the web +
  expo region/city filter request bodies against the exact contract
  `{"productsFilter":{"selectedRegionAndCityValues":{"regionIds":[id],"cityIds":[id]}}}` (entity
  IDs, not `RegionAndCityDTO` objects, not `labelKey`s). Note: `RegionAndCityDTO` is the user-profile
  shape, NOT the filter shape — a likely client confusion point. Also verify the live ES index has
  `regionId`/`cityId` populated (reindex if stale).

- **Part 4b flags (read-only — not fixed, for triage):**
  - `RegionAndCityQueryGenerator.java:41` — term value via `FieldValue.of(Object)` (JsonData-wrapped)
    vs siblings' typed `.value(Long)`. Proven byte-identical output. **Severity: low (cosmetic).**
  - `DocumentProductConverter.java:87,89` — unguarded `getCity()/getRegion()` deref though
    `Product.region`/`city` are nullable (`Product.java:46-52`) → NPE aborts indexing for a
    region/city-less product. **Severity: medium** (could fail a reindex batch, but not the silent
    symptom under audit).
  - `ProductIndexer.indexOne:361-367` — omits `Hibernate.initialize(region/city)`; benign only
    because the method is `@Transactional`. **Severity: low.**
  - Zero automated test coverage of the region/city filter path (no test references the generator or
    the wire keys). **Severity: low/medium** — a contract test would catch exactly this class of
    drift.

- **Config-file dependency check (closure gate):** none required — no edit to conventions.md,
  decisions.md, state.md, or issues.md is implied by this read-only audit. The Part 4b flags are
  surfaced here for Mastermind to decide whether any become issues.md entries.
