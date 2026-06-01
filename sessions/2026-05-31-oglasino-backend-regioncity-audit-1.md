# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-31
**Task:** AUDIT (read-only) — region/city: is it missing, dropped, or a DTO name mismatch? (issues.md 2026-05-31 — "Mobile: update-product screen crashes — regionAndCity undefined on productDetails")

READ-ONLY audit. No code changed, nothing staged. Findings only.

---

## Findings

### Q1 — EXACT WIRE SHAPE

**Endpoint:** `GET /api/secure/products?productId=<id>` (dashboard product-details).

- **Controller:** `DashboardProductController.getProductDetails`
  `controller/DashboardProductController.java:53-67`
  ```java
  @GetMapping
  public ResponseEntity<UpdateProductRequestDTO> getProductDetails(
      @RequestParam @NotNull Long productId) {
  ```
  Class mapping: `@RequestMapping("/api/secure/products")` (`:43`). No method-level path → resolves to `GET /api/secure/products`.

- **Service path:** `productFacade.getProductForUpdate(productId)` then `productFacade.toUpdateData(product)` (`:56`, `:66`).
  - `DefaultProductFacade.getProductForUpdate` → `productService.getProductForId(productId)` (`facade/impl/DefaultProductFacade.java:39-42`).
  - `DefaultProductFacade.toUpdateData` → `modelMapper.map(product, UpdateProductRequestDTO.class)` (`facade/impl/DefaultProductFacade.java:44-47`).
  - `DefaultProductService.getProductForId` → `productRepository.findById(productId)` (`service/impl/DefaultProductService.java:591-594`).

- **Response DTO:** `UpdateProductRequestDTO` (`dto/UpdateProductRequestDTO.java`).

**VERBATIM region/city fields on the response DTO: THERE ARE NONE.**

The full field set of `UpdateProductRequestDTO` (`:22-39`) is:
```java
private Long id;
private String name;
private String description;
private BigDecimal price;
private CurrencyDTO currency;
private List<SelectedFilterDTO> filters;
private Set<String> imageKeys;
```
There is no `regionAndCity` field. There is no `region` field. There is no `city` field. There is no `location`/`regionCity` field. The omission is deliberate and documented in the class Javadoc (`:13-16`):
> "Mutable fields for a product update. Categories, region/city, and baseSite are immutable post-creation and are therefore absent from this DTO — they are read from the persisted entity."

**Jackson naming:** no global naming strategy. No `spring.jackson.property-naming-strategy` in `application-dev.yaml` / `application-prod.yaml`; no `@JsonNaming`, no `PropertyNamingStrategies`, no custom `ObjectMapper`/`Jackson2ObjectMapperBuilder` bean, and no `@JsonProperty` on any region/city DTO field. Fields serialize as-is, camelCase. So the absence of `regionAndCity` on the wire is not a rename — the field genuinely does not exist on this endpoint's payload.

**Name comparison (where the names DO exist, for reference):** the backend has a `RegionAndCityDTO` (`dto/RegionAndCityDTO.java:5-25`) whose members are **character-for-character identical** to the mobile shape:
```java
@NotNull private RegionDTO region;
@NotNull private CityDTO city;
```
and both `RegionDTO` and `CityDTO` carry `@NotBlank private String labelKey;` (`dto/RegionDTO.java:9`, `dto/CityDTO.java:9`). So `regionAndCity.region.labelKey` / `regionAndCity.city.labelKey` is a real, exactly-matching backend shape — **but `RegionAndCityDTO` is wired into the user context, not into `UpdateProductRequestDTO`.** The product update-details endpoint never references it.

### Q2 — POPULATION

The mapping is a blanket `modelMapper.map(product, UpdateProductRequestDTO.class)` (`facade/impl/DefaultProductFacade.java:46`). It cannot populate region/city because the **target DTO has no region/city properties to populate**. The source `Product` entity does carry the data (`entity/Product.java:46-52`):
```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn
private Region region;

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn
private City city;
```
…but with no matching destination field on `UpdateProductRequestDTO`, ModelMapper drops it. So region/city is **always** absent from this endpoint's response, regardless of entity state.

### Q3 — SCHEMA + WRITE

- **Schema (nullable):** `db/migration/V1__init_schema.sql` product table (`:342-364`) declares `region_id bigint` and `city_id bigint` with **no NOT NULL** — both columns are nullable at the DB level. FKs to `region(id)` / `city(id)` exist (`:1492-1496`, `:1580-1584`). `Region.labelKey` / `City.labelKey` are `nullable = false` (`entity/Region.java:19-20`, `entity/City.java:20-21`; schema `region.label_key`, `city.label_key` NOT NULL).
- **Write path (create):** `DefaultProductService.createProduct` (`service/impl/DefaultProductService.java:93-112`) sets region/city **from the authenticated user**, never from the request body, and guards against null first:
  ```java
  if (Objects.isNull(user.getBaseSite())
      || Objects.isNull(user.getRegion())
      || Objects.isNull(user.getCity())) {
    throw new ProductValidationException(UserErrorCode.USER_SETUP_INCOMPLETE);
  }
  ...
  product.setRegion(user.getRegion());
  product.setCity(user.getCity());
  ```
  So a product created through this path **always** has non-null region and city (the guard rejects users without them; values are trust-boundary-correct — derived from the user, not the client).
- **Write path (update):** `UpdateProductRequestDTO` carries no region/city, so the update path never reads, writes, or nulls them. They are immutable post-create, as the Javadoc states.

Net: the schema permits null, but the only write path guards null and sources the values server-side from the user. Region/city should be present on virtually every persisted product.

### Q4 — VERDICT

**(c) CONTRACT MISMATCH — and stronger than a rename: the field is entirely ABSENT from the response contract by design.** The dashboard details endpoint returns `UpdateProductRequestDTO`, which deliberately omits region/city ("immutable post-creation"); the backend never serializes `regionAndCity` (nor `region`/`city`) on `GET /api/secure/products?productId=<id>`, so the mobile read `productDetails.regionAndCity` is `undefined` and `.region.labelKey` throws. Not (a) — the write path guards null and sources region/city from the user, so the data exists on the entity. Not (b) — there is no field to "drop"; the destination DTO has no region/city property at all. The exact names the mobile reads (`regionAndCity` → `region`/`city` → `labelKey`) DO exist verbatim in the backend's `RegionAndCityDTO`/`RegionDTO`/`CityDTO`, but that DTO is not wired into this endpoint.

---

## Implemented

- Nothing. Read-only audit; the findings above are the deliverable.

## Files touched

- None (read-only). Files **read** for evidence: `DashboardProductController.java`, `UpdateProductRequestDTO.java`, `DefaultProductFacade.java`, `DefaultProductService.java`, `Product.java`, `Region.java`, `City.java`, `RegionAndCityDTO.java`, `RegionDTO.java`, `CityDTO.java`, `NewProductRequestDTO.java`, `NewProductResponseDTO.java`, `V1__init_schema.sql`, `application-dev.yaml`, `application-prod.yaml`.

## Tests

- Ran: none (read-only audit; no code change).

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change.
- issues.md: no change authored by me (Docs/QA is sole writer). This audit answers the open 2026-05-31 "Mobile: update-product screen crashes — regionAndCity undefined" entry; see "For Mastermind" for the recommended status note. No config-file edit required from this session beyond that optional annotation.

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code written.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one flagged in "For Mastermind".
- Part 6 (translations): N/A this session.
- Part 7 (error contract): touched — noted the `createProduct` null-region/city path raises `USER_SETUP_INCOMPLETE`; consistent with the codes-only contract. No issue.
- Part 11 (trust boundaries): confirmed — region/city are derived server-side from the authenticated user, never from client input. No violation.

## Known gaps / TODOs

- I did not read the mobile (`oglasino-expo`) code — cross-repo, forbidden, and not needed. The mobile-side assertion (that `[productId].tsx` reads `productDetails.regionAndCity.region.labelKey`) is taken from the issues.md entry / pasted stack trace, not verified by me. My audit establishes the backend half: this endpoint does not send `regionAndCity`.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — no code written.
  - Considered and rejected: nothing — read-only audit.
  - Simplified or removed: nothing.

- **Verdict in one line (for the issues.md entry, Docs/QA to apply if you want it recorded):** "(c) contract mismatch — backend `GET /api/secure/products?productId=<id>` returns `UpdateProductRequestDTO`, which deliberately omits region/city (immutable post-create); the field mobile reads (`regionAndCity`) is never sent, so it is `undefined`. Data exists on the entity; write path guards null and sources region/city from the user."

- **The fix is a product/design decision, not a pure bug — flagging for routing.** The mismatch is real but the resolution is a choice between three shapes, and it is cross-repo:
  1. **Backend adds region/city to the response.** Add a `RegionAndCityDTO regionAndCity` field to `UpdateProductRequestDTO` (or a separate read DTO) and map it from the entity. The exact names already match the mobile shape, so this is small backend work — BUT it contradicts the DTO's stated contract ("region/city … absent from this DTO"), so it needs an explicit decision that the *update-details read* should carry immutable display fields even though they are not editable. Cleaner option may be a dedicated read DTO rather than overloading the request/response DTO.
  2. **Mobile reads region/city from elsewhere** (e.g. the authenticated user, which is where the backend sources them on create) and stops expecting them on `productDetails`. Web-parity check needed — see below.
  3. **Mobile guards/omits the display** if region/city is not part of the update surface at all.
  Picking among these is Mastermind's call; I'm only establishing that the backend currently sends nothing here.

- **Adjacent observation (Part 4b) — web likely has the same gap, severity low/medium, NOT fixed (cross-repo, out of scope):** `UpdateProductRequestDTO` is the shared dashboard-details contract; if `oglasino-web`'s update-product screen also expects region/city on this endpoint it would hit the same absent-field condition (web would render undefined/blank rather than crash, since JS optional chaining differs). Worth a web check when this is scoped. I did not read web code. File: `dto/UpdateProductRequestDTO.java`. "I did not fix this because it is out of scope (cross-repo)."

- **Adjacent observation (Part 4b) — low:** the create flow rejects a user with null region/city via `USER_SETUP_INCOMPLETE` (`DefaultProductService.java:99-103`), so the nullable `region_id`/`city_id` columns are effectively never null for created products. If a future direct-write path (admin import, data migration) bypasses `createProduct`, the nullable columns + the absent guard would allow null region/city, which any future region/city-on-read DTO must then tolerate. Not actionable today; noted so a future "add region/city to the read DTO" brief defends against null. "I did not fix this because there is nothing to fix today."

- No drafted config-file text beyond the optional one-line verdict for the issues.md entry above. No unstated config-file dependency.
