# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-06-04
**Task:** ES read-path verification + mapping/settings audit (read-only) — independently confirm prior performance-audit source claims with raw grep/cat evidence (Part 1) and audit the ES mapping/settings layer (Part 2).

## Implemented

- Read-only audit only — no source, config, or live-ES mutations. Produced `.agent/audit-es-performance.md`.
- **Part 1 (5 claims, all confirmed):** no `withSourceFilter`/`FetchSourceFilter` anywhere (full `_source` shipped on every hit); `descriptionTranslations` is in `_source` but never read by `ProductOverviewConverter` (read only by `TextQueryGenerator` as a search dependency); `withTrackTotalHits(true)` at exactly lines 45/72/88 of `DefaultProductsFilterQueryBuilder`; autocomplete routes through the same `getProductsFor`→`buildQuery` path as full listing then maps to a 3-field DTO; all four external clients (reCAPTCHA `new RestTemplate()`, SMTP yamls, OpenAI `HttpClients.createDefault()`, R2 `UrlConnectionHttpClient.builder().build()`) have no timeout.
- **Part 2 (live local ES 9.0.1):** pulled `_mapping`, `_settings` (incl. `include_defaults`), `_cat/indices`, `_cat/shards`, `_alias`. Key decision fact: `descriptionTranslations.translation` is `search_as_you_type` (analyzed) — excluding it from a `_source` filter is **free for search** (search reads the inverted index, not `_source`). Settings: 1 shard, 1 replica, refresh 1s, `max_result_window` default 10000, slow-log **OFF**.
- Surfaced 5 adjacent ES-layer observations (see For Mastermind).

## Files touched

- `.agent/audit-es-performance.md` (new, audit deliverable) (+~230 / -0)
- `.agent/2026-06-04-oglasino-backend-es-performance-1.md` (this summary)
- `.agent/last-session.md` (copy of this summary)

No source/config/test files modified.

## Tests

- Ran: none. Read-only audit; no code change, so no test run is warranted (no touched module).
- Result: n/a
- New tests added: none

## Cleanup performed

- none needed (no code changed)

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change required by me. (Note for Mastermind: the brief's premise "indices are empty everywhere pre-launch" is false on the local box — 1083 seed docs / 172.9 MB — but this is local seed data, not a prod/state fact; flagged below, not drafted as a state edit.)
- issues.md: no change authored. 4 net-new ES-layer findings are flagged in "For Mastermind" for Mastermind to triage into issues.md if it chooses (I do not write the config files).

## Obsoleted by this session

- nothing. (This audit re-verifies and, in places, corrects the prior exploratory performance audit's framing, but it does not obsolete any code or doc artifact.)

## Conventions check

- Part 4 (cleanliness): confirmed — no code touched; deliverable is a single new `.agent/` audit file referenced by the brief.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): 5 ES-layer observations flagged in "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: Part 11 (trust boundaries) — incidentally confirmed sound: the sort-field whitelist (`OrderType` catalog, `DefaultProductsFilterQueryBuilder.java:159`) and the DASHBOARD_SEARCH owner-from-auth guard (lines 104-112) are present and reading from the server side. No new trust-boundary concern surfaced.

## Known gaps / TODOs

- I did **not** verify whether `ProductDetailsDTO` (single-id detail path) reads `descriptionTranslations`. The source-filter "free" conclusion is asserted for the **listing/autocomplete** builders (`buildQuery`) only; before any future write-brief adds a source filter to the single-id builders it must confirm the detail DTO. Stated explicitly in the audit §2.1.
- Absolute ES query latency not measured — deferred by design (empty/unrepresentative indices pre-launch). On record in the audit's closing line.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code or abstractions introduced.
  - Considered and rejected: nothing — no implementation decisions in scope.
  - Simplified or removed: nothing — read-only.

- **No brief-vs-reality blocker.** All 5 Part 1 claims confirmed exactly as stated (including the line numbers 45/72/88). One framing correction, not a blocker: the brief states indices are "empty everywhere pre-launch" — the local product index holds **1083 seed docs / 172.9 MB** (audit §2.3). Local seed data, not prod; the profiling-deferral conclusion is unaffected, but the empty-index assumption is locally false and is what makes `store.size` numbers (and the size-bloat findings below) observable at all.

- **Net-new ES-layer findings (Part 4b) — candidates for issues.md (I did not fix; out of scope, read-only):**
  1. **(medium)** `search_as_you_type` on `descriptionTranslations.translation` is over-specified — the only query against it (`TextQueryGenerator.java:107`) is a plain `.match()` at boost 0.2, using none of the shingle/prefix subfields `search_as_you_type` exists to provide. A plain `text` field would serve it identically far cheaper. Shared mapping at `reference/TranslationReference.java:9-13`. Likely a primary driver of ~163 KB/doc.
  2. **(low)** `description_autocomplete_index/_search` analyzers + `description_autocomplete_tokenizer` in `es-config/elastic-analyzer.json` are **dead config** — `TranslationReference.translation` (name *and* description) is hardcoded to the `name_autocomplete_*` analyzers. Also a correctness smell: descriptions are analyzed with name's edge_ngram min_gram 1, not the min_gram 3 the unused `description_autocomplete_filter` implies.
  3. **(medium)** `include_in_parent: true` on every nested field appears **unused** — all searching goes through `QueryBuilders.nested().path(...)` (`TextQueryGenerator`, `SpecFilterQueryGenerator.java:50-72`); no query reads the flattened parent copies, and converters read `_source` (unaffected by `include_in_parent`). Pure index bloat. Removing it needs a prior check for any flattened-field consumer (sort/agg/non-nested term).
  4. **(low/medium)** `number_of_replicas: 1` on a single-node cluster → permanently-yellow index, unallocatable replica. Harmless locally; if it ships to a single-node prod ES the prod product index sits yellow forever. Confirm prod ES node count; if single-node, `number_of_replicas: 0` for that env.

- **Already-tracked, not re-litigated:** a fifth no-timeout HTTP client (`DefaultCloudflareKvService.java:23`) exists beyond the brief's four; it is already in issues.md (2026-06-03). Noted in the audit, no new entry needed.

- **Suggested next step:** the source-filter opportunity (drop `descriptionTranslations` — and possibly the whole filter footprint — from the listing/autocomplete `buildQuery` `_source`) plus findings #1 and #3 are the highest-value, lowest-risk ES read-path wins, but each is a *write* with a test pass and (for #1/#3) a reindex. They belong in a post-audit write-brief, not bundled here. Latency profiling stays deferred to post-launch per the brief.
