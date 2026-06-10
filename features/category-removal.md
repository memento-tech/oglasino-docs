# Category Removal

**Status:** implemented on `dev`, validated via clean DB reseed + smoke test (2026-06-10)
**Repos:** oglasino-backend (catalog JSON + translations). Web/mobile greps parked (see Open items).
**Scope:** catalog structure only — removes live-animal and weapon categories and replaces the
three pet-accessory buckets with a curated supply taxonomy. Does NOT add content-moderation
rules; preventing these goods from being listed under remaining categories is a separate lever,
out of scope.

## Open items

- **RU strings.** The RU values for the 9 leaves are Latin-script transliteration drafts;
  Igor's translator finalizes them. (`cat_accessories`/`dog_accessories` reuse pre-existing
  labelKeys — only their RU values were updated.)
- **Web/mobile greps.** Key/route greps for the removed + new routes across `oglasino-web`
  and `oglasino-expo` are parked ("later").

## Goal

Remove weapon and live-animal categories from the catalog, and replace the three filtered
pet-accessory buckets with a curated, filterless supply taxonomy under `pets`.

## Decisions (Igor, 2026-06-10)

- **weapons** — `airguns`, `firearms` removed; `collectible_weaponry` removed.
- **militaria** — KEPT. Its filter and options stay. (Re-partitions the audit — see Caution.)
- **ammunition / alcohol / tobacco** — no category exists in the catalog (audit confirmed:
  only false-positive substrings and appliance/drinkware/BBQ filter options). Backend no-op.
- **live animals** — removed: `hunting_dogs`, and under `pets` the species leaves `cats`,
  `dogs`, `fish`, `small_animals`, `terrarium_animals`. `pets.other` KEPT.
- **pet accessories** — the three filtered accessory buckets (`cat_accessories`,
  `dog_accessories`, `fish_accessories`) are removed as filtered buckets. They are replaced by
  a **curated, filterless supply taxonomy** under `pets` (NOT a 1:1 option-to-category
  promotion — that was built, smoke-tested, and rejected as too granular). `cat_accessories`
  and `dog_accessories` survive as curated leaves reusing their existing labelKeys;
  `fish_accessories` is dropped (no replacement leaf).
- **alcohol-adjacent filter options** (`wine_coolers`, `wine_glasses`, `beer_glasses`,
  `smoker`) — KEPT (a wine glass is not alcohol; a BBQ smoker is not tobacco).

All cut/transform nodes live under root `hobbies`. Only
`catalogJSON/categories/hobbies.json` is edited; `catalog-{rs,me,rsmoto}.json` untouched
(roots, top filters, free zone unchanged). `hunting_fishing` and `collectibles` parents stay;
only the named children change.

## The pets supply taxonomy (as built)

The live-animal leaves and the three filtered accessory buckets are removed. In their place,
`pets` carries 9 curated, terminal leaves — no subcategories, no filters (filters may be added
later). `pets.other` is kept, repositioned last (position 100). The set is deliberately
asymmetric: dog-only transport cages (cat cages a possible future add), no small-animal
supplies, and no fish/bird/terrarium toys — a curated launch set, not a gap.

| pos | key (under category.hobbies.pets.) | route | EN | RS / CNR | RU (draft) |
|-----|------------------------------------|-------|----|---------:|-----------:|
| 1 | cat_toys | /cat_toys | Cat Toys | Igračke za mačke | Igrushki dlya koshek |
| 2 | cat_accessories | /cat_accessories | Cat Accessories | Oprema za mačke | Prinadlezhnosti dlya koshek |
| 3 | dog_toys | /dog_toys | Dog Toys | Igračke za pse | Igrushki dlya sobak |
| 4 | dog_accessories | /dog_accessories | Dog Accessories | Oprema za pse | Prinadlezhnosti dlya sobak |
| 5 | dog_transport_cages | /dog_transport_cages | Dog Transport Cages | Transportni boksevi za pse | Perenoski dlya sobak |
| 6 | aquarium_equipment | /aquarium_equipment | Aquarium Equipment | Oprema za akvarijume | Oborudovanie dlya akvariumov |
| 7 | terrarium_equipment | /terrarium_equipment | Terrarium Equipment | Oprema za terarijume | Oborudovanie dlya terrariumov |
| 8 | bird_cages | /bird_cages | Bird Cages | Kavezi za ptice | Kletki dlya ptits |
| 9 | bird_accessories | /bird_accessories | Bird Accessories | Oprema za ptice | Prinadlezhnosti dlya ptits |
| 100 | other | /other | Other | Ostalo | (existing) |

Labels are freshly authored (qualified) strings, NOT relocated from the old option values. RS
and CNR share the Serbian string. RU is a Latin-script transliteration draft pending Igor's
translator. `cat_accessories`/`dog_accessories` reuse pre-existing labelKeys (their rows were
kept; only RU values updated); all other leaf labels are new.

## SHARED keys — NEVER delete

Used by hundreds of kept categories. Deleting any breaks kept labels:

- `category.other` (67 kept categories)
- `filter.type` (every `.type` filter catalog-wide)
- `filter.options.other` (471 uses across 9 root files — down from 480; the 9 removed `.type`
  filters each carried an `other` option)

`pets.other` is KEPT (repositioned last, position 100) and legitimately uses `category.other`
and `filter.type` — its node and rows stay; those SHARED labels are never deleted.

## Caution — militaria staying re-partitions the audit

The audit's "all filters/options deletable" assumed all four groups died. militaria is KEPT,
so:

- `filter.hobbies.collectibles.militaria.type` and its option rows are KEPT.
- `collectible_weaponry.type` and its option rows are removed.
- Any option shared between militaria (kept) and collectible_weaponry (cut) is now PROTECTED.
  The audit pinned `equipment → militaria` and `blades → collectible_weaponry` but did not
  fully resolve `decorative_weapons`, `replica_weapons`, `medals`, `uniforms`, `insignia`.

Before deleting ANY filter-option row or its translation, the engineer re-derives option
membership against the then-current hobbies.json and deletes only options that no kept
category (militaria included) references. This re-run is mandatory — the standing
tool-reliability hazard means the audit snapshot is not a delete authority on its own.

## Catalog mechanics (from audit)

- Catalog is hand-authored JSON → DB at boot via `CatalogManager` (upsert-only, NEVER
  removes). Editing JSON alone does not remove a category from a populated DB.
- Each cut category = 2 `catalog_category_assignment` rows (base sites rs=1000 AND me=2000);
  rsmoto has no hobbies.
- No `ON DELETE CASCADE` on any catalog FK — DB deletes are FK-ordered (audit §6) if/when a
  DB delete is scripted. Pre-prod, a clean reseed from the edited JSON is the simpler path.
- Labels live only in `0002-data-translations-*.sql` (categories/filters) and
  `0003-data-filter-option-translations-*.sql` (options), per locale (EN/RS/RU/CNR).

## As built

The feature shipped as a single recut (catalog JSON + the 8 translation seeds in one pass)
after an initial 20-flat-leaf implementation was reverted following smoke test. Translation
deletions were militaria-aware and sweep-gated (a key is deleted only if referenced by zero
nodes across all catalog JSON post-recut). Net translation change: +9 category labels and
−10 category labels in 0002 (×4 locales); −42 filter options in 0003 (×4 locales).

Validated 2026-06-10 via clean DB reseed + smoke test: the new 9-leaf shape renders under
`pets`, no kept node lost, militaria intact, and the three SHARED keys preserved. The 9 leaves
resolve a label in all four locales (RU as transliteration drafts pending the translator).
