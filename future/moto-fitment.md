# F4 Deep Dive — Serbia Moto: Vehicle-Aware Parts Marketplace

**Prepared:** 2026-06-10 · **Companion to:** oglasino-strategic-analysis.md and oglasino-f3-ai-listing-studio.md
**Context:** Second build after F3 (AI Listing Studio). Escrow/payments (F1) is explicitly deferred by founder decision — nothing in this plan touches payment rails. F4 was acquisition #1 and cold-start #2 in the master analysis, and is the designated first 90-day battleground.

**Escrow-deferral implications (brief):** the acquisition strategy is unaffected (all top picks were free by design); near-term revenue realism shifts to lines needing no payment infrastructure — affiliate monetization and B2B subscriptions paid by predračun/bank transfer; F8 (dropshipping) becomes doubly non-viable. Cheap optionality to keep: when talking to couriers about anything else, ask which payment institutions they see doing marketplace work.

---

## 1. The reframe that makes it buildable

The naive version is "license or build a motorcycle fitment database." Don't. Licensed data (TecDoc-class) is car-centric, expensive (thousands €/yr — GUESS), and weak on moto. Pre-building universal fitment is a multi-year data project.

The buildable version: **structured fitment tagging on listings + vehicle-scoped search.** Sellers already know what their part came off ("skidano sa CBR600RR iz 2005") — today they bury that knowledge in free text where search can't use it. Give them a constrained picker instead of a sentence, and the fitment database **accumulates as a byproduct of listings**. The moat isn't bought; the marketplace secretes it. That's also why KP can't shortcut-copy it: the data only exists where the listings happen.

Scope sharpening from existing taxonomy work: the *attribute* axis for moto gear and parts (type, material, sizes) is largely covered by the May filter-coverage work on `moto_equipment_parts`. F4 adds the orthogonal **vehicle axis** — and that axis only matters for *parts*. Helmets need sizes, not fitment. F4 = parts, concretely.

## 2. Data model

Three small tables and two ES fields:

**`moto_vehicles`** — the master list: make, model, year_from, year_to, generation/engine where it disambiguates (MT-07 2014–2020 vs 2021+ is a real fitment boundary). Scope: the ~100–150 models that dominate the regional market. Bootstrap: AI-drafted first pass from what's actually listed on regional sites (browsed, not scraped), then founder verification — days of work, not weeks. Admin CRUD with an append-friendly correction log; community corrections are the feature working, not a failure.

**Listing fitment** — multi-select of vehicle IDs + a `universal_fit` flag. No free-text fitment ever (free text kills filtering). When a seller's model isn't listed: "predloži model" → admin queue → master list grows exactly where demand is. Plus an optional **OEM part number** field, normalized (strip spaces/dashes), indexed — buyers genuinely search by OEM number, and supporting it costs one keyword field.

**`user_vehicles` ("Moja garaža")** — user saves their bikes once; every moto-parts search gets a one-tap "Za moj motor" chip. Multiple bikes supported. The retention hook: a saved garage is a reason to come back that KP structurally can't offer.

## 3. Search behavior — one cold-start-critical detail

Strict filtering at low liquidity makes the site look dead: filter on, three results, user leaves. Default behavior: **fitted results first, then a visible "ostali oglasi u kategoriji (bez podataka o kompatibilnosti)" section** — relevance ranking, not exclusion. Strict mode as an explicit toggle once inventory justifies it. The same layout gracefully handles listings created before the feature (which won't carry fitment tags).

ES side: `fitment_ids` (keyword array), `universal_fit` (bool), `oem_numbers` (keyword + normalizer) are **new fields → mapping update, no reindex** — they only populate going forward, which is exactly right. Tiny keyword fields, nowhere near the analyzed-text machinery that previously inflated the index.

## 4. Seller side — where F3 and F4 are one feature

The fitment picker lives in the listing flow, which post-F3 means **the confirm screen**. Phase 1: when category ∈ moto parts, the picker appears with autocomplete over the master list. Phase 2: the vision model suggests fitment candidates from photos and any visible part number — *suggests only*. Hallucinated fitment is this feature's worst failure mode (a wrong category embarrasses; a wrong fitment costs a buyer money and gets screenshotted into the exact FB groups being courted). AI fitment is always constrained-vocabulary suggestions behind a human confirm, never auto-published.

Quality ops: a "Prijavi pogrešnu kompatibilnost" button on every fitted listing → existing moderation queue → fix + notify. Target fitment-report rate <1–2% (GUESS). Fast, visible corrections in a tight community are themselves marketing.

## 5. Alerts — a contained pilot of F6 (demand matching)

Garage + "obavesti me kad se pojavi deo za moj motor" = a saved query. Mechanically: ES **percolator** — register the query (category subtree + fitment terms + optional price ceiling), percolate each new listing at index time, fire the existing localized push. This pilots the F6 demand-matching concept in the scope where it's most valuable and least empty-feeling. Strong alert open-rates = evidence for generalizing F6 later; weak = a cheap lesson. Phase 2 work. Note: percolator registration is a mapping consideration to bundle with the already-queued reindex, same bundling logic as the F3 embeddings note.

## 6. The campaign half (50% of this feature is not code)

- **Shops first.** Hand-onboard 5–10 moto parts shops (Novi Sad/Belgrade radius — physically walkable). Inventory arrives via F7-style feed or concierge-listed through the F3 studio in bulk. Shops are double value: they pre-fill fitment-tagged supply, *and* they know fitment cold, anchoring data quality.
- **Community second.** Moto FB groups and forums are the channel, and they ban drive-by promo — the play is genuine participation plus the one demo that sells itself: "sačuvaj svoj motor, dobiješ push kad se pojavi deo koji mu odgovara." A sentence nobody can say about KP.
- **Season timing.** Parts/maintenance demand concentrates in pre-season prep, roughly Feb–Apr, with an autumn wrenching bump (GUESS, directionally safe). Natural arc from June 2026: build Q3, soft-launch into autumn wrenching season with shops seeded, full community push before spring 2027.

## 7. Monetization — fully escrow-free

Fitment search: **free forever** (the moat-builder; charging for it strangles the data flywheel). Later, two lines:

- **Bike history checks** on whole-bike listings via affiliate (carVertical-class providers; retail ~€15–25 / 1,800–3,000 RSD — GUESS). Affiliate = zero payment infrastructure on the platform side — the cleanest possible revenue under the no-payments constraint.
- **Shop subscriptions** once traffic justifies them: €10–30/mo (1,200–3,500 RSD — GUESS, anchored to KP Izlog), invoiced by predračun/bank transfer like every other small-business service in Serbia.

Mark affiliate links as such (advertising-law hygiene — LIKELY requirement, trivial to satisfy).

## 8. Scope ladder and effort (solo + AI)

- **Phase 1 (3–5 weeks):** master list + admin CRUD, fitment fields + picker in listing flow, "Za moj motor" filter with fitted-first/others-below layout, garage, report button, instrumentation. Web first.
- **Phase 2 (2–3 weeks):** garage alerts via percolator, OEM-number search, AI fitment suggestions in the F3 confirm screen, Expo parity.
- **Phase 3 (1–2 weeks):** history-check affiliate on bike listings, OEM cross-reference table seeded from accumulated data.
- **Deferred indefinitely:** VIN decode — nice demo, no early value.

Cut order under pressure: Phase 3, then alerts, then AI suggestions. Non-negotiable core: master list + picker + garage filter — that alone is the differentiated product.

Build order vs F3: F3 Phase 1 → F4 Phase 1 (the picker extends the confirm screen just built) → F3 Phase 2 / F4 Phase 2 interleaved by what seeding demands.

## 9. Metrics and kill criteria

- **Supply:** active moto-parts listings (the number that *is* the wedge); % of new moto-parts listings carrying fitment tags (target >70% — lower means the picker UX is failing).
- **Demand:** % of moto-parts searches using garage/fitment filtering (≥25–30% working, <10% failing); listing→contact rate in moto vs site average; garages created; alert open rate.
- **Quality:** fitment-report rate <1–2%.
- **Kill:** after ~3 months of *active* seeding (shops in, groups engaged), <300–500 active parts listings and <10% filter usage means the niche isn't biting (all thresholds GUESS) → fold Moto back into the general site, keep the master list and fitment schema, stop spending founder-weeks. The data survives a retreat; the moat just builds slower.

## 10. Legal — light, three notes

Fitment facts (part X fits bike Y) aren't protectable and are fine to state; bulk-copying proprietary OEM/dealer catalog *databases* is not — build from sellers, shops, and manufacturer-published aftermarket fitment lists, not from scraping parts retailers. Garage data is personal data under ZZPL (reveals vehicle ownership) — ordinary handling; mention in the privacy doc's data inventory while that workstream is open. Affiliate links disclosed.

## 11. Repo mapping

**oglasino-backend:** ~60% — tables, fitment endpoints, ES mapping additions, percolator service, admin CRUD + moderation-queue wiring, affiliate redirect with disclosure. **oglasino-web:** garage UI, filter chips + two-tier results layout, picker, report flow. **oglasino-expo:** Phase 2 parity. **oglasino-docs:** the master list's source-of-truth location plus the decision record — *fitment is seller-declared against a constrained vocabulary; rejected alternative: licensed fitment DB (cost, moto coverage)*. **oglasino-router:** nothing.

## 12. Sequencing honesty

Post-launch work, and Phase 1 sits behind F3 Phase 1 on the dependency chain. The zero-code work doable this month: the 20-question validation in moto groups ("kad si poslednji put kupovao deo, koliko ti je trebalo da budeš siguran da odgovara?") and a first AI-drafted cut of the vehicle master list, reviewable on a coffee break.