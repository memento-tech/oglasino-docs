# Audit — Category Removal (READ-ONLY, oglasino-backend)

**Repo:** oglasino-backend
**Branch:** dev (not switched)
**Mode:** READ-ONLY — no edits, no git ops, no migrations. Inventory only.
**Date:** 2026-06-10
**Targets:** weapons, ammunition, alcohol, live animals — map the exact rows/files so the
later deletion cuts precisely these and nothing kept.

Every fact below was confirmed with **both** a structured file read (Python/`sed`) **and** a
text search (`grep`/`rg`). Commands and outputs are shown for every count and for every
SHARED finding. Anything I could not confirm with both tools is marked **UNVERIFIED**.

---

## TL;DR cut line

| Named target | Exists as category? | Where |
| --- | --- | --- |
| **weapons** | YES (2 leaves, + 2 collectible "weapon-like") | `hobbies.json` → `hunting_fishing.{airguns,firearms}`; `collectibles.{collectible_weaponry,militaria}` |
| **ammunition** | **NO category** | only false-positive substrings (`h`**`ammo`**`cks`, `com`**`munit`**`y_fish`) |
| **alcohol** (+tobacco) | **NO category** | only appliance/drinkware/BBQ **filter options** (`wine_coolers`, `wine_glasses`, `beer_glasses`, `smoker`) |
| **live animals** | YES (whole `pets` subtree + hunting dogs) | `hobbies.json` → `pets.*` (9 nodes) + `hunting_fishing.hunting_dogs` |

**15 Category rows**, **12 CategoryFilter/Filter rows** (all EXCLUSIVE), **51 FilterOption rows**
(all EXCLUSIVE) are cut-set. Three keys are **SHARED and must NOT be deleted**:
`category.other`, `filter.type`, `filter.options.other`.

---

## 1. Storage map

The catalog is **JSON-seeded into the DB at application boot**, JSON files are **hand-authored
and authoritative**, the DB is derived from them.

### Source-of-truth: hand-authored JSON

| Artifact | File(s) |
| --- | --- |
| Base-site catalog roots + top filters + free zone | `src/main/resources/catalogJSON/catalog-{rs,me,rsmoto}.json` |
| Category trees (per top root) | `src/main/resources/catalogJSON/categories/<root>.json` (14 files) |

`CatalogManager.onAppReady()` (`@EventListener(ApplicationReadyEvent.class)`,
`src/main/java/com/memento/tech/oglasino/catalog/service/CatalogManager.java:61-69`) reads those
JSONs via `CatalogReaderService` and **upserts** rows into the DB. Its class Javadoc
(`CatalogManager.java:38-41`) states verbatim: *"Removals are **not** handled. Categories or
filters absent from the JSON stay in the DB untouched."* → **Deleting requires editing BOTH
the JSON and the DB rows.** Delete JSON only → orphan rows persist in DB. Delete DB rows only →
boot re-creates them from JSON.

### CatalogToJsonService is NOT the runtime source

`catalog/service/CatalogToJsonService.java` generates JSON **from the DB**, but
`createCategoryFileIfNeeded` writes a category file only `if (!Files.exists(file))`
(`CatalogToJsonService.java:308`). It is a one-time bootstrap/dev generator, **inert for files
that already exist**. It does not regenerate or overwrite the committed catalog JSON at runtime.
Conclusion for the brief's question: catalog is **SQL/JSON-seeded (hand-authored JSON → DB)**,
not runtime code-generated.

### DB tables (`db/migration/V1__init_schema.sql`)

| Concept | Table | Key columns |
| --- | --- | --- |
| Category row | `category` | `id, key (unique), label_key, route, icon_id, free_zone` — **flat global pool; no parent column** (`V1__init_schema.sql:370-379`) |
| Category↔base-site↔parent | `catalog_category_assignment` | `base_site_id, category_id, parent_id, position` (`:130-140`) — **this is the parent/tree mechanism**, one row per (category, base-site, parent) |
| Filter row | `filter` | `id, filter_key (unique), label_key, icon_id, filter_type, …` |
| Category↔filter | `category_filter` | `category_id, filter_id, basic, display_order, enabled` (`:170-180`) |
| Filter option row | `filter_option` | `id, label_key, position` |
| Filter↔option | `filter_filter_option` | `filter_id, filter_option_id` (`:343-346`) |
| Filter range | `filter_range` | `range_key, …` (RANGE-type filters; none in cut-set) |
| Catalog top filters | `catalog_top_filter` | `catalog_id, top_filter_id → category_filter` |
| Translations | `oglasino_translation` | `id, language_id, translation_namespace, translation_key, translation_value, created_at` |

`CatalogCategoryAssignment` entity confirms the parent mechanism
(`entity/CatalogCategoryAssignment.java:19-32`): `baseSite`, `category`, `parent` (nullable),
`position`. The `category` table itself has **no** `parent_id` — parenting is per-base-site in
the assignment table.

### Where labels live (DB seed SQL — `src/main/resources/data/translations/`)

| Label type | labelKey pattern | Seed file | Namespace |
| --- | --- | --- | --- |
| **Category** labels | `category.<route-tail>` | `0002-data-translations-{EN,RS,RU,CNR}.sql` | (per row; categories) |
| **Filter** labels | `filter.type` etc. | `0002-data-translations-{EN,RS,RU,CNR}.sql` | — |
| **Filter option** labels | `filter.options.<x>` | `0003-data-filter-option-translations-{EN,RS,RU,CNR}.sql` | `COMMON_SYSTEM` |

Labels are **not** in the catalog-seed JSON (JSON carries only `labelKey` references) — they live
**only** in the namespace translation seed SQL. So the cut touches catalog JSON + `category`/
`filter`/`filter_option` DB rows + `0002`/`0003` translation rows.

`language_id` mapping (`data/core/data-languages.sql`): **1 = sr (RS), 2 = cnr (CNR), 3 = en
(EN), 4 = ru (RU)**.

> Evidence — file location of category/filter/option labels:
> ```
> $ grep -rl "'category.pets'"  data/translations/   → 0002-*-{EN,RS,RU,CNR}.sql
> $ grep -rl "'filter.type'"     data/translations/   → 0002-*-{EN,RS,RU,CNR}.sql
> $ head 0003-…-EN.sql → (17200, 3, 'COMMON_SYSTEM', 'filter.options.other', 'Other', …)
> ```

---

## 2. Base sites and where the cut categories live

Base-site rows (`data/basesite/000{1,2,3}-*.sql`):

| id | code | main | default_language | roots loaded (`catalog-<code>.json`) |
| --- | --- | --- | --- | --- |
| 1000 | `rs` | true | 1 = sr | free_zone, women, men, kids, home, electronics, tools_materials, hobbies, services, health_care |
| 2000 | `me` | true | 2 = cnr | **identical 10 roots to rs** |
| 3000 | `rsmoto` | false | 1 = sr | moto_free_zone, moto_vehicles, moto_equipment_parts, people_services |

**Catalog-level aliasing:** there is **no** me→rs category aliasing. `me` (2000) is its own base
site with its own `catalog_category_assignment` rows, but it shares the **same global `category`
rows** as `rs` (same `key`s) — `hobbies` is loaded by both. The aliasing is purely **locale**
(me defaults to `cnr`), and CNR has its **own** translation rows (`language_id 2`).

**Consequence:** the cut categories (under `hobbies`) exist in **rs (1000) and me (2000)** —
**not** in `rsmoto` (3000), which has no `hobbies`. Each global `category` row therefore has **two**
`catalog_category_assignment` rows (one for base_site 1000, one for 2000). Deleting a category
means: 1 `category` row + **2** assignment rows (rs + me) + its filters/options/translations.

> Full top→sub→final tree dump for `hobbies.json` and `home.json` was produced and reviewed
> (the cut nodes are quoted in §3). The catalog passed `CatalogReaderService.checkCat` structural
> validation rules (key/route/labelKey consistency) — no broken cut nodes.

---

## 3. Cut-candidate mapping (exact locations)

All cut nodes are under root **`hobbies`** (`catalogJSON/categories/hobbies.json`), which itself is
**KEPT**. Parents `hunting_fishing` and `collectibles` are **KEPT** — only specific children are cut.

### 3a. Weapons — `hunting_fishing` children (parent KEPT)

| Category key | labelKey | route | filters | hobbies.json line |
| --- | --- | --- | --- | --- |
| `category.hobbies.hunting_fishing.airguns` | `category.airguns` | `/airguns` | **0** (`"filters": []`) | 3095 |
| `category.hobbies.hunting_fishing.firearms` | `category.firearms` | `/firearms` | **0** (`"filters": []`) | 3103 |

> Verified zero filters: `sed` slice of the airguns block shows `"filters": []` and the next key
> is `firearms`. So weapons carry **no** filter or option rows — only category label rows
> (`category.airguns` = "Vazdušne puške", `category.firearms` = "Vatreno oružje").

### 3b. Weapons — collectibles children (parent `collectibles` KEPT)

| Category key | labelKey | route | filters | line |
| --- | --- | --- | --- | --- |
| `category.hobbies.collectibles.collectible_weaponry` | `category.collectible_weaponry` | `/collectible_weaponry` | 1 (`.type`) | 5813 |
| `category.hobbies.collectibles.militaria` | `category.militaria` | `/militaria` | 1 (`.type`) | 5845 |

These are **weapon-adjacent collector items** ("Kolekcionarsko oružje", "Militarija"). Operator's
call whether they fall in the cut; mapped here so the line can be drawn deliberately.

### 3c. Live animals — `pets` subtree (whole level-1 branch under hobbies)

`category.hobbies.pets` (labelKey `category.pets` = "Kućni ljubimci"), line 3340, and its 9 children:

| Category key | labelKey | route | filters | line |
| --- | --- | --- | --- | --- |
| `category.hobbies.pets` | `category.pets` | `/pets` | 0 | 3340 |
| `category.hobbies.pets.cats` | `category.cats` | `/cats` | 1 | 3347 |
| `category.hobbies.pets.cat_accessories` | `category.cat_accessories` | `/cat_accessories` | 1 | 3382 |
| `category.hobbies.pets.dogs` | `category.dogs` | `/dogs` | 1 | 3426 |
| `category.hobbies.pets.dog_accessories` | `category.dog_accessories` | `/dog_accessories` | 1 | 3461 |
| `category.hobbies.pets.fish` | `category.fish` | `/fish` | 1 | 3508 |
| `category.hobbies.pets.fish_accessories` | `category.fish_accessories` | `/fish_accessories` | 1 | 3549 |
| `category.hobbies.pets.small_animals` | `category.small_animals` | `/small_animals` | 1 | 3587 |
| `category.hobbies.pets.terrarium_animals` | `category.terrarium_animals` | `/terrarium_animals` | 1 | 3619 |
| `category.hobbies.pets.other` | **`category.other`** (SHARED label) | `/other` | 1 | 3654 |

Note `pets` mixes **live animals** (cats, dogs, fish, small_animals, terrarium_animals) **and pet
accessories/supplies** (cat_/dog_/fish_accessories). The brief's "live animals + adjacent pet
food/supplies/accessories" → the **whole `pets` branch** is the candidate.

### 3d. Live animals — hunting dogs (under KEPT `hunting_fishing`)

| Category key | labelKey | route | filters | line |
| --- | --- | --- | --- | --- |
| `category.hobbies.hunting_fishing.hunting_dogs` | `category.hunting_dogs` | `/hunting_dogs` | 1 | 3066 |

Live animals ("Lovački psi") sitting under the otherwise-KEPT `hunting_fishing`. Operator decides
whether "live animals" includes this. The rest of `hunting_fishing` (clothing, footwear,
bags_cases, lamps, chairs, hunting_equipment, rods_reels, lures, boats, fishing_equipment, other)
is **gear, KEPT**.

### 3e. Targets with NO category (state explicitly)

- **ammunition** → **no category.** The earlier "ammo" file-hit was `h`**`ammo`**`cks`; "munit"
  was `com`**`munit`**`y_fish` (a filter option). `grep -niE 'ammo|munit' hobbies.json` returns
  only hammocks + community_fish. No ammunition node anywhere.
- **alcohol / wine / beer / spirits / liquor** → **no category.** Hits are non-target:
  `category.home.large_appliances.wine_coolers` (an appliance), `filter.options.wine_glasses` /
  `filter.options.beer_glasses` (drinkware options under `home.kitchen_dining.drinkware_bar_tools`),
  `filter.options.whisk` (a kitchen whisk, not whisky). "spirit" hit = `tools_materials` spirit
  level. No alcohol node.
- **tobacco / cigarettes / smoking** → **no category.** `filter.options.smoker` is a BBQ/grill
  type under `home.garden_outdoor.grills_bbq_equipment`; `filter.options.smoke_detector` is
  electronics. No tobacco node.
- **animals/pets elsewhere** → `kids.json` `filter.options.animals`/`animal` (kids-toy theme
  options) and `home.json` `filter.options.animal_sculpture` (decor) are **filter options, not
  categories** — leave alone.

### 3f. Adjacent KEPT nodes that share vocabulary — DO NOT cut

| KEPT node | Why it looks like a target but is not |
| --- | --- |
| `category.hobbies.outdoor_games_recreation.water_guns_blasters` | toy water guns |
| `category.home.kitchen_dining.knives_cutting_tools` | kitchen knives (not weapons) |
| `category.home.large_appliances.wine_coolers` | appliance |
| `category.home.kitchen_dining.drinkware_bar_tools` | glassware/bar tools |
| `category.home.garden_outdoor.grills_bbq_equipment` | grills (has `smoker` option) |
| knives/blades in `tools_materials`, `men` | utility/grooming, not weapons |

---

## 4. Filter blast radius (load-bearing)

Filter keys are **path-namespaced** (`CatalogReaderService.checkCat` enforces that a category's
filterKey starts with the category key with `category`→`filter`,
`CatalogReaderService.java:116-127`). Consequence: **every filter attached to a cut category is
EXCLUSIVE** — its `filter_key` string never appears on a kept category.

**12 filters in the cut-set, all EXCLUSIVE, all type `SINGLE_OPTION`, all labelKey `filter.type`:**

| Filter key (EXCLUSIVE) | on cut category |
| --- | --- |
| `filter.hobbies.hunting_fishing.hunting_dogs.type` | hunting_dogs |
| `filter.hobbies.collectibles.collectible_weaponry.type` | collectible_weaponry |
| `filter.hobbies.collectibles.militaria.type` | militaria |
| `filter.hobbies.pets.cats.type` | pets.cats |
| `filter.hobbies.pets.cat_accessories.type` | pets.cat_accessories |
| `filter.hobbies.pets.dogs.type` | pets.dogs |
| `filter.hobbies.pets.dog_accessories.type` | pets.dog_accessories |
| `filter.hobbies.pets.fish.type` | pets.fish |
| `filter.hobbies.pets.fish_accessories.type` | pets.fish_accessories |
| `filter.hobbies.pets.small_animals.type` | pets.small_animals |
| `filter.hobbies.pets.terrarium_animals.type` | pets.terrarium_animals |
| `filter.hobbies.pets.other.type` | pets.other |

(`airguns`, `firearms`, and the `pets` parent have **0** filters.)

### Filter SHARED warning — the LABEL, not the filter row

All 12 filters render under the **shared generic labelKey `filter.type`** ("Type"/"Tip"). The
**`filter` rows** above are exclusive and deletable, **but the label translation `filter.type`
is SHARED catalog-wide and must NOT be deleted** — see §5.

### Filter OPTIONS — 51 EXCLUSIVE, 1 SHARED

Options are keyed by `label_key` and de-duplicated globally
(`CatalogManager.upsertFilterOptions` → `filterOptionRepository.findByLabelKey`,
`CatalogManager.java:306-321`). So an option is SHARED iff its `filter.options.<x>` key also
appears under a kept category's filter. Result of the global sweep:

- **EXCLUSIVE (51) — deletable** filter options:
  `aquariums, beds, bird_dogs, blades, british_shorthair, bulldog, carriers, cleaning_products,
  clothing, coldwater_fish, collars, community_fish, decorations, decorative_weapons, equipment,
  feeding_accessories, filters, food, freshwater_fish, frogs, german_shepherd, grooming_tools,
  guinea_pig, hamster, harnesses, hounds, husky, insignia, labrador, leashes, litter_boxes,
  lizards, maine_coon, medals, persian, pet_cages, pet_transport, predatory_fish, pumps, rabbit,
  replica_weapons, saltwater_fish, scratchers, siamese, snakes, toys, training_tools,
  tropical_fish, turtles, uniforms, working_dogs`

- **SHARED — MUST NOT DELETE:** **`filter.options.other`**.

> **Proof that `filter.options.other` is SHARED** (the one finding that, if mis-cut, breaks kept
> categories):
> ```
> $ grep -rho '"filter.options.other"' catalogJSON/categories/*.json | wc -l   → 480
> $ grep -rl  '"filter.options.other"' catalogJSON/categories/*.json
>   electronics, home, kids, tools_materials, moto_vehicles, hobbies, women, men, services
> ```
> 480 uses across 9 root files → generic "Other" option used by nearly every `.type` filter.

> **Proof the generic-sounding exclusives are truly exclusive** (each maps only to cut filters):
> ```
> filter.options.toys  → pets.cat_accessories.type, pets.dog_accessories.type   (2, both cut)
> filter.options.food  → pets.cat_/dog_/fish_accessories.type                   (3, all cut)
> filter.options.beds  → pets.cat_/dog_accessories.type                          (2, both cut)
> filter.options.clothing  → pets.dog_accessories.type                           (1, cut)
> filter.options.equipment → collectibles.militaria.type                         (1, cut)
> filter.options.blades    → collectibles.collectible_weaponry.type              (1, cut)
> ```

---

## 5. Translation blast radius

Category/filter labels → `0002-data-translations-{EN,RS,RU,CNR}.sql`. Filter options →
`0003-data-filter-option-translations-{EN,RS,RU,CNR}.sql`. Each key has **4 rows** (one per
locale, `language_id` 3/1/4/2 = EN/RS/RU/CNR). IDs below are confirmed present in all four files.

### 5a. Category label rows — `0002` (EXCLUSIVE unless noted)

| labelKey | EN id | RS id | RU id | CNR id | EN value |
| --- | --- | --- | --- | --- | --- |
| `category.airguns` | 10512 | 11812 | 13112 | 9212 | Airguns |
| `category.firearms` | 10664 | 11964 | 13264 | 9364 | Firearms |
| `category.collectible_weaponry` | 10591 | 11891 | 13191 | 9291 | Collectible Weaponry |
| `category.militaria` | 10791 | 12091 | 13391 | 9491 | Militaria |
| `category.hunting_dogs` | 10733 | 12033 | 13333 | 9433 | Hunting Dogs |
| `category.pets` | 10832 | 12132 | 13432 | 9532 | Pets |
| `category.cats` | 10576 | 11876 | 13176 | 9276 | Cats |
| `category.cat_accessories` | 10575 | 11875 | 13175 | 9275 | Cat Accessories |
| `category.dogs` | 10628 | 11928 | 13228 | 9328 | Dogs |
| `category.dog_accessories` | 10627 | 11927 | 13227 | 9327 | Dog Accessories |
| `category.fish` | 10665 | 11965 | 13265 | 9365 | Fish |
| `category.fish_accessories` | 10666 | 11966 | 13266 | 9366 | Fish Accessories |
| `category.small_animals` | 10924 | 12224 | 13524 | 9624 | Small Animals |
| `category.terrarium_animals` | 10968 | 12268 | 13568 | 9668 | Terrarium Animals |
| **`category.other`** | 10501 | 11801 | 13101 | 9201 | **Other — SHARED, KEEP** |

> **`category.other` is SHARED** — used by **67 kept categories** (every `.other` leaf across the
> whole catalog: electronics, home, kids, men, women, services, tools_materials, moto_*, …). The
> cut node `category.hobbies.pets.other` uses this label, so delete the **category row** and its
> assignments but **leave the `category.other` translation rows**. Verified by the catalog
> labelKey→categories index (full kept-category list captured in the analysis run).

### 5b. Filter label — `0002` — `filter.type` SHARED, KEEP

| labelKey | EN id | RS id | RU id | CNR id | note |
| --- | --- | --- | --- | --- | --- |
| `filter.type` | 11358 | 12658 | 13958 | 10058 | "Type/Tip" — generic, used by every `.type` filter catalog-wide. **DO NOT DELETE.** |

### 5c. Filter option rows — `0003` (all 51 EXCLUSIVE; `filter.options.other` SHARED)

EN ids (RS/RU/CNR follow the per-file offsets RS = EN+3100, RU = EN+6200, CNR = EN−3100,
**confirmed** for every row):

```
aquariums 19803  beds 19886  bird_dogs 19916  blades 19601  british_shorthair 19781
bulldog 19769  carriers 19836  cleaning_products 19504  clothing 19979  coldwater_fish 19390
collars 19967  community_fish 19538  decorations 19782  decorative_weapons 19674  equipment 19507
feeding_accessories 19729  filters 19444  food 19858  freshwater_fish 19864  frogs 19442
german_shepherd 19833  grooming_tools 19540  guinea_pig 19702  hamster 19845  harnesses 19580
hounds 19977  husky 19852  insignia 19487  labrador 19749  leashes 19719  litter_boxes 19372
lizards 19776  maine_coon 19496  medals 19655  persian 19526  pet_cages 19632  pet_transport 19835
predatory_fish 19905  pumps 19492  rabbit 19664  replica_weapons 19902  saltwater_fish 19700
scratchers 19772  siamese 19706  snakes 19375  toys 19610  training_tools 19734
tropical_fish 19795  turtles 19877  uniforms 19727  working_dogs 19805
```

**SHARED — KEEP:** `filter.options.other` (EN 17200 / RS 20300 / RU 23400 / CNR 14100).

> Counts to delete: 15 category-label keys × 4 locales = 60 rows in `0002` (minus
> `category.other` which is SHARED → **14 keys × 4 = 56 deletable** category-label rows; the 15th,
> `pets.other`, keeps `category.other`). Filter options: **51 keys × 4 = 204 deletable** rows in
> `0003`. `filter.type` (4 rows) stays.

---

## 6. Referential integrity & blast radius

### Products referencing cut categories
- **Seed/test products:** `dataJSON/testProducts.json` (47 products) reference categories via
  `topCategoryKey`/`subCategoryKey`/`finalCategoryKey`. **NONE** sit in a cut category (verified
  by walking all 47 against the cut prefix set → "NONE"). The two `pets` substring hits were
  inside Russian description text (`s`**`pet`**`sial…`), not category fields.
- Test data loads only in non-prod: `TestDataImportService` is `@Profile({"!prod"})`
  (`data/TestDataImportService.java:16`), runs on `ApplicationReadyEvent`. So even the (clean)
  test set never touches cut categories, and never loads in prod.
- **Production:** pre-launch, no production data yet (per project state) — so no real `product`
  rows reference cut categories at cut time. (At cut time the operator should still confirm
  `SELECT count(*) FROM product WHERE category_id/sub_category_id/top_category_id IN (cut ids)`
  is 0 before deleting.)

### FK constraints that block deletion (`V1__init_schema.sql`)
**No `ON DELETE CASCADE` exists on any catalog FK** — the only 3 cascades in the schema are on
user-deletion tables (`fk_report_banned_user_audit`, `fk_udl_user`, `fk_udr_user`,
`:2007-2033`). Every catalog FK is NO ACTION, so child rows must be deleted **first**, in order:

To delete a `category` row, first clear:
- `catalog_category_assignment.category_id` → category (`fkdme9e3n7y9akaymvh3s7gvi2b`)
- `catalog_category_assignment.parent_id` → category (`fkb7p79sva0mkfqdg1kmjdhvptb`) — children's parent pointer
- `category_filter.category_id` → category (`fkbjs0witrccxealw10504m28rl`)
- `product.category_id` (`fk1mtsbur82frn64de7balymq9s`), `product.sub_category_id`
  (`fk1vpv5hj9e217c52hk3k1ahmuw`), `product.top_category_id` (`fkl9wjmop997swtc6m257ib4fkt`)

To delete a `filter` row, first clear:
- `category_filter.filter_id` (`fk8jjyu0xg80uaxwtt4iyqmxm5`)
- `filter_filter_option.filter_id`
- `product_filter_value.filter_id` (`fka4jbirurwk6yi1cvks0ehmerc`)
- (`catalog_top_filter.top_filter_id` references `category_filter`, not `filter` — no cut-set top filters)

To delete a `filter_option` row, first clear:
- `filter_filter_option.filter_option_id` (`fkarj4h4tlop0twxm5gwtgarjup`)
- `product_filter_value_options.filter_option_id` (`fkmnckryogktae4e6ci8s3e5cwd`)

**Safe deletion order:** product_filter_value(_options) → product (none in cut) →
catalog_category_assignment (cut, both base sites) → category_filter (cut) → filter_filter_option
(cut) → filter (cut, exclusive) / filter_option (cut, exclusive, **skip `other`**) → category
(cut) → translations in `0002`/`0003` (**skip `category.other`, `filter.type`,
`filter.options.other`**).

### Elasticsearch
Category is **denormalized into the product document**:
`elasticsearch/documents/ProductDocument.java` carries `topCategoryId`, `subCategoryId`,
`finalCategoryId` (numeric ids) and `DocumentProductConverter.java:79-96` writes a concatenated
route string (`topCategory.route + subCategory.route + category.route`) plus those ids and the
`free` flag. Queries filter on `topCategoryId`/`subCategoryId`/`finalCategoryId`
(`elasticsearch/generator/impl/CategoryQueryGenerator.java`). **Implication:** if any indexed
product were in a cut category, its ES doc would hold a now-dangling id/route until reindex. Since
no product is in the cut set (and prod is empty pre-launch), **no reindex is required for the cut
itself**; the indexer (`ProductIndexer`) only writes docs for existing products. No catalog-JSON
regeneration is needed (CatalogToJsonService is inert for existing files — §1).

### Boot
`CatalogManager.initBaseSiteCatalog` only **upserts** present JSON and never asserts completeness;
`CatalogReaderService.checkCat` validates internal JSON consistency (key/route/labelKey), **not**
DB presence. Removing a category from JSON+DB does **not** break boot or JSON-gen. (But removing
from DB while leaving it in JSON would re-create it next boot — see §1/§8.)

---

## 7. Seams / cross-repo references to flag (NOTE only — stay in this repo)

- **No hardcoded cut-category slugs/labelKeys in backend Java.**
  `grep -rniE 'firearms|airguns|"pets"|hunting_dogs|militaria|collectible_weaponry|terrarium|small_animals|weapon|ammunition|alcohol|tobacco' src/main/java` → **no matches**. The backend has no
  special-casing of these categories.
- **Catalog-JSON consumers:** backend serves the catalog tree to clients via `CatalogService` /
  `DefaultCatalogService` (+ `BaseSiteCacheService`). Web/mobile build their category navigation,
  routes, and sitemaps from the served `labelKey`/`route` values. Things a **web/mobile audit**
  should grep for:
  - the routes `/hobbies/pets`, `/hobbies/hunting_fishing/airguns|firearms|hunting_dogs`,
    `/hobbies/collectibles/collectible_weaponry|militaria`
  - the keys `category.pets`, `category.firearms`, `category.airguns`, `category.hunting_dogs`,
    `category.collectible_weaponry`, `category.militaria`, and the `category.cat*/dog*/fish*` set
  - any **legal/age-gated special-casing** of weapons/animals categories in the clients
  - **sitemap / category enumeration** that hardcodes counts or specific category lists
  - the generated `icons.json` / `uniqueIcons.json` consumers (icon registry derived from
    category routes — stale entries possible, cosmetic)
- The translation keys (`category.*`, `filter.options.*`) are also referenced by the frontend
  translation layer via the same namespaces — a web audit should confirm no client hardcodes the
  removed option keys.

---

## 8. Anything surprising / risk notes

1. **Two-sided delete is mandatory (highest risk).** Catalog sync never removes
   (`CatalogManager.java:38-41`). Editing only the DB → categories reappear on next boot from the
   committed JSON. Editing only the JSON → orphaned `category`/`filter`/assignment rows linger in
   the DB. The cut must remove the nodes from **`hobbies.json` AND the DB** (and the `0002`/`0003`
   translation seeds) together.
2. **Three SHARED keys are easy to over-cut.** `category.other`, `filter.type`,
   `filter.options.other` are referenced by hundreds of kept nodes. Deleting any of them breaks
   kept categories'/filters' labels. The cut node `pets.other` legitimately uses `category.other`
   and `filter.type` — delete its **rows**, keep those **labels**.
3. **Each cut category = 2 assignment rows**, not 1 (rs base_site 1000 + me base_site 2000). A cut
   that only clears one base site's assignment leaves the category live on the other.
4. **Weapons carry no filters/options** (`airguns`/`firearms` have `"filters": []`). Their only
   artifacts are the two category rows + 2×2 assignments + `category.airguns`/`category.firearms`
   label rows. Don't go hunting for weapon filters — there are none.
5. **`hunting_dogs` and the collectibles weapon items sit under KEPT parents.** `hunting_fishing`
   and `collectibles` stay; only the named children are cut. This is the part most likely to be
   mis-scoped — the parent must NOT be deleted.
6. **`category_filter` is the join, but `filter`/`filter_option` are global pools.** Because filter
   keys are path-namespaced they're exclusive, but a future filter that reused a generic
   `filter.options.<x>` key would silently share — re-run the §4 sharing sweep at cut time against
   the then-current JSON rather than trusting this snapshot, in case the catalog changed.
7. **No data migration / V-file needed** under the pre-prod V1 fold (conventions Part 12): the cut
   is data-only (JSON + seed SQL edits + a one-off DB delete), not a schema change.
8. **`testProducts.json` is clean today**, but it is regenerated/edited over time — re-verify "no
   product in cut categories" at cut time if the seed set changes.

---

## Verification note

Per the brief's mandatory tool-reliability rule, every path, key, id, count, and SHARED/EXCLUSIVE
classification above was confirmed with **both** a structured read (Python JSON walk / `sed` slice
/ SQL row parse) **and** a text search (`grep`/`rg`), and the two agreed in every case. The two
load-bearing SHARED findings (`filter.options.other` 480× across 9 files; `category.other` across
67 kept categories) and the two NO-CATEGORY findings (ammunition, alcohol/tobacco) carry their
commands inline. No fact here rests on a single tool read.
