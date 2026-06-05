# Session summary

**Repo:** oglasino-backend
**Branch:** dev (not switched)
**Date:** 2026-06-04
**Task:** Read-only investigation — Igor has 47 products but the local `products` ES index reports 1083 docs / 172.9 MB. Find out what the 1083 docs actually are. Do not clean anything up — just explain the gap.

## Implemented

- Read-only investigation only. No code, no ES mutations, no DB writes. Deliverable is `.agent/audit-es-doccount.md`.
- **Verdict: no product-count discrepancy.** The index holds exactly 47 products = 47 root documents. The "1083" is Lucene's `docs.count`, which counts every nested object as its own hidden Lucene document. The prior audit (`audit-es-performance.md`) misread `_cat/indices` `docs.count` as a product/document count.
- Proved it three ways: `_count` = 47, `hits.total` = 47, and a `cardinality` agg on the product `id` = 47 distinct ids. Postgres `SELECT count(*) FROM product` = 47 (44 ACTIVE / 2 DELETED / 1 INACTIVE), and the ES `productState` breakdown mirrors the DB exactly.
- Accounted for all 1083 Lucene docs exactly via nested-path aggregations: `47 root + 188 nameTranslations + 188 descriptionTranslations + 47 baseSite + 205 filterReferences + 205 filterReferences.filter + 203 filterReferences.options = 1083`. `ProductDocument` has 4 `FieldType.Nested` fields and `filterReferences` nests recursively (filter + options), which is the 23× inflation.
- Confirmed the indexer writes one id-keyed doc per product (`ProductIndexer.java:341-344, 373-377`) with no fan-out, and that every write path (admin reindex, per-product `indexOne`, non-prod boot `reindexAll`) is an upsert or full-rebuild that cannot accumulate docs. Only one concrete `products_*` index exists (no orphans), `docs.deleted = 0`.
- Sampled docs are real listings (RTX 3070, Bosch jigsaw, sneakers, cargo bike), not seed/test junk.

## Files touched

- `.agent/audit-es-doccount.md` (new, +~180) — the deliverable
- `.agent/2026-06-04-oglasino-backend-es-doccount-1.md` (new) — this summary
- `.agent/last-session.md` (overwritten) — exact copy of this summary

No source files modified.

## Tests

- Ran: none — read-only investigation, no code change.
- Result: N/A
- New tests added: none

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change — but the prior-audit "163 KB/doc" / "1083 seed docs" framing referenced in the ES performance work is corrected by this audit (see "For Mastermind"); flagged, not edited.
- issues.md: no change — two adjacent observations flagged for Mastermind to route as potential issue entries (see "For Mastermind"); I did not author them per the no-write rule.

## Obsoleted by this session

- The interpretation in `.agent/audit-es-performance.md` (lines 220-260) that the index holds "1083 docs" / "1083 seeded local docs" and that "172.9 MB / 1083 ≈ 163 KB/doc" is the per-document size. The count is a Lucene root+nested figure; per *product* it is 47 docs and ≈3.78 MB/product. That prior audit's size *root-cause* analysis (search_as_you_type subfields + include_in_parent duplication) remains valid and is reinforced. I did not edit that file (it is a peer audit record, not mine to rewrite); the correction is documented in the new audit and flagged here.

## Conventions check

- Part 4 (cleanliness): confirmed — no code touched, no debug artifacts.
- Part 4a (simplicity): N/A — no code added.
- Part 4b (adjacent observations): two adjacent observations surfaced and flagged (DELETED/INACTIVE products are indexed; ~3.78 MB/product store size). Flagged, not acted on, per read-only scope.
- Part 6 (translations): N/A this session.
- Other parts touched: hard rules on read-only `psql` (used `docker exec` against the local container, read-only SELECTs only) and no-branch-switch (stayed on `dev`) observed.

## Known gaps / TODOs

- The read path's `productState` filtering was not verified (out of scope) — flagged below since DELETED/INACTIVE products are present in the index.
- The ~3.78 MB/product index size is not investigated here (it is a known item from the prior performance audit); flagged, not pursued.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing
  - Considered and rejected: nothing
  - Simplified or removed: nothing
- **Headline:** the 23× "discrepancy" is not real. 47 products = 47 ES documents = 47 distinct ids = 47 DB rows. `1083` is the Lucene root+nested doc count, fully accounted for: `47 + 188 + 188 + 47 + 205 + 205 + 203`. Nothing to clean.
- **Correction to prior audit / state framing (for Docs/QA, no edit made by me):** anywhere the ES performance work or `state.md` characterizes the local index as "1083 docs / 1083 seed docs / 163 KB per doc," the accurate framing is "47 product documents (1083 Lucene root+nested docs); ≈3.78 MB per product." The prior audit's *size driver* analysis (search_as_you_type subfields, include_in_parent duplication) is correct and is amplified by the nested-doc multiplier. Suggest Docs/QA reconcile the two audits when archiving.
- **Two adjacent observations to consider as issues.md entries (flag only — I did not author them):**
  1. DELETED (2) and INACTIVE (1) products are present in the `products` index (ES mirrors the DB row-for-row; `findIdsForIndexing` filters no state — `ProductRepository.java:45`). This is fine *iff* the read path filters `productState=ACTIVE` at query time. Worth a quick read-only confirmation of `DefaultProductsFilterQueryBuilder` to rule out non-ACTIVE listings appearing in search.
  2. Index store size is ≈3.78 MB/product (47 products → 172.9 MB), driven by the nested fan-out × per-field `search_as_you_type` subfields × `include_in_parent` term duplication. Already flagged in `audit-es-performance.md` as a write-brief candidate; this audit quantifies the per-product cost.
- Nothing else flagged.
