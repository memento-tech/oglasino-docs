# Session summary

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-06-12
**Task:** Mobile: first-run base-site / region-city selection crashes, then the selection window never reappears — AUDIT FIRST, do not fix blind.

## Outcome

**Audit only — no code changed.** The crash throw-site could not be pinned to the
first-run setup path, and the brief's central "partial persist" premise is
contradicted by the code. Per the brief's explicit STOP branch ("IF the cause is
NOT clear after the trace → STOP, write the trace findings, return to Mastermind")
and Igor's in-session decision, I am returning the trace rather than fixing blind.

## The flow (as it actually is)

"Add a product" entry points call `openProductCreateGate(user, openDialog)` →
`productCreateGateTarget(user)` (`src/lib/utils/productCreateGate.ts:24`). A user
missing setup is routed to `UserBasicDataSelectorDialog` (base-site choice → region/
city choice). On submit that dialog persists, refreshes the user, and — on the
create-gate path (`shouldOpenDialog: true`) — auto-advances into the create wizard
(`AddUpdateProductDialog`). Single-slot dialog store (`useDialogStore`), so
`openDialog(NEW_PRODUCT_DIALOG)` *replaces* the setup dialog; no stacking.

There is **no separate "base site + region/city" picker inside the create wizard**.
In the wizard, region/city is **display-only**, sourced from `user.regionAndCity`
and guarded (`BasicInfoProductDialog.tsx:266`). The "selection window" in the
symptom is the **setup gate dialog** (`UserBasicDataSelectorDialog`), shown the
first time an un-set-up user tries to add a product.

## Trace findings (the three the brief asked for)

### Persist point — `UserBasicDataSelectorDialog.tsx:84-115`
The only write is `assignUserLocation(selectedRegionAndCity)` →
`POST /secure/user/update/region-city`, sending `{region, city}` only (backend
derives base site — `userService.ts:48`, web parity). It is gated behind
`canSubmit = !!selectedBaseSite && !!selectedRegionAndCity` (line 117), and
`onUpdate` early-returns if either is missing (line 85). `CitySelector.handleSelect`
always sets `region` and `city` **together** (`CitySelector.tsx:105`). After the
write, `syncUserToBackend` refreshes the authoritative user and `setUser` stores it;
then `openDialog(NEW_PRODUCT_DIALOG)`.

**Therefore there is no partial-write path from this dialog** — the persist only
fires on a complete (base-site + region + city) selection. The brief's "selection
persisted before completion" half has nothing to fix on the client.

### Gate condition — `productCreateGate.ts:25`
```ts
if (!user.baseSite || !user.regionAndCity) { /* show setup dialog */ }
```
`RegionAndCityDTO = { region: KeyLabelPair | null, city: KeyLabelPair | null }`
(`src/lib/types/catalog/RegionAndCityDTO.ts`). The gate checks that the
`regionAndCity` **object is present**, NOT that `.region` and `.city` are both
non-null. So if the backend ever returns a non-null `regionAndCity` whose `region`
or `city` is null (a partial/empty object), the gate **passes** and the setup picker
is **skipped on an incomplete setup**. This is exactly the "gate checks presence,
not completeness" failure mode, and it matches the symptom "base site + region
appear already selected, picker never reshows."
- **Caveat:** whether the backend ever emits a partial `regionAndCity` object (vs.
  `null`) is unverified — it lives in `oglasino-backend`, out of this repo. The gate
  logic is loose regardless, but the symptom only manifests if the backend returns a
  partial object. This is the open question for Mastermind / a backend check.

### Crash site — NOT pinned on the first-run path
The region/city derefs in the setup path are all guarded:
`UserBasicDataSelectorDialog`, `CitySelector` (selectedRegionAndCity always
optional-chained), and `BasicInfoProductDialog.tsx:266` (`userRegionAndCity?.region
&& userRegionAndCity.city &&`). I could not reproduce or localize a definite throw
in the setup → create auto-advance from static reading.

The one **live** instance of the known crash family is **off this path**:
`ProductUserDetails.tsx:122-125` — guards on `regionAndCity?.region` but then derefs
`regionAndCity.city.labelKey` unguarded (line 125). Crashes if `city` is null while
`region` is set. That is the product-detail surface, not the add-product flow — but
it is the same `regionAndCity.<x>` of-undefined family as the resolved 2026-05-31
update-product crash, and could be a sibling symptom if the backend is emitting
partial `regionAndCity` objects.

## Brief vs reality

1. **No partial-persist path on the client**
   - Brief says: "the first selection attempt half-completed (persisted SOMETHING)
     before crashing" → fix half (a) "only persist a COMPLETE selection."
   - Code says: `assignUserLocation` is the sole write, gated behind
     `canSubmit` (both base site and a region+city pair) and an early-return guard;
     `CitySelector` sets region+city atomically (`UserBasicDataSelectorDialog.tsx:85,117`,
     `CitySelector.tsx:105`).
   - Why this matters: half (a) of the proposed fix has nothing to change — the
     client already persists only complete selections. If a partial selection is
     reaching storage, it is a backend behavior, not a client write path.
   - Recommended resolution: drop client half (a); investigate whether the backend
     persists/returns a partial `regionAndCity`.

2. **Gate checks object-presence, not completeness (real, but symptom depends on backend)**
   - Brief says: gate half (b) "gate checks presence-not-completeness."
   - Code says: `productCreateGate.ts:25` — `!user.regionAndCity` is true only when
     the whole object is null/undefined; a `{region:null, city:null}` (or one-null)
     object passes the gate.
   - Why this matters: this is a genuine looseness and the most likely "picker never
     reshows" mechanism — but it only fires if the backend returns a partial object.
   - Recommended resolution (deferred — see "For Mastermind"): tighten to
     `!user.baseSite || !user.regionAndCity?.region || !user.regionAndCity?.city`.
     Held back this session because the crash is unpinned and the backend shape is
     unverified ("do not fix blind").

3. **Crash site not in the first-run setup path**
   - Brief says: "Region/city selection has a known crash family — check whether a
     region or city object is dereferenced before it exists."
   - Code says: the setup/CitySelector/create-wizard region/city derefs are guarded;
     the one live unguarded deref is `ProductUserDetails.tsx:125`, on the product-
     detail surface, not the add-product flow.
   - Why this matters: fixing a guard on the setup path blind would not address the
     reported crash; the actual throw needs the on-device stack trace to localize.
   - Recommended resolution: obtain the device crash stack trace.

## Files touched

- none (audit only)

## Tests

- Ran: none (no code changed)
- Result: N/A
- New tests added: none

## Cleanup performed

- none needed (no code changed)

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: **1 new entry drafted** for Docs/QA (see "For Mastermind"). Captures
  the gate presence-vs-completeness looseness + the `ProductUserDetails.tsx:125`
  unguarded deref, both tied to the unpinned first-run crash. Drafted, not applied —
  Docs/QA is the sole writer.

## Obsoleted by this session

- nothing (no code changed)

## Conventions check

- Part 4 (cleanliness): N/A — no code changed.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one flagged (`ProductUserDetails.tsx:125`).
- Part 6 (translations): N/A this session.
- Other parts touched: Part 7 (error contract) — confirmed: the persist path uses
  the standard service/throw contract; not modified.

## Known gaps / TODOs

- Crash throw-site for the first-run setup → create auto-advance is unpinned. Needs
  the on-device stack trace.
- Backend `regionAndCity` shape for a setup-incomplete user (null vs partial object)
  is unverified and is the hinge of the gate-skip symptom.
- (no TODO/FIXME added to code.)

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — audit only, no code.
  - Considered and rejected: tightening `productCreateGate.ts:25` to a completeness
    check, and guarding `ProductUserDetails.tsx:125`. Both are small and defensible,
    but rejected for THIS session because the crash is unpinned and the backend shape
    is unverified — the brief is "do not fix blind," and a blind guard on the setup
    path would not address the actual reported crash.
  - Simplified or removed: nothing.

- **Two things are independently true and worth deciding on:**
  1. **Gate looseness (half b — real).** `productCreateGate.ts:25` gates on the
     `regionAndCity` object's presence, not on `.region`/`.city` being set. One-line
     fix available: `!user.baseSite || !user.regionAndCity?.region ||
     !user.regionAndCity?.city`. But it only changes behavior if the backend emits a
     partial `regionAndCity` — **needs a backend confirmation** (does
     `/auth/firebase-sync` ever return `regionAndCity` with one or both of
     region/city null for a half-set-up user?). If yes, this is the "picker never
     reshows" root cause and the fix is correct + web-parity-sensible. If the backend
     only ever returns `null` until both are set, the gate is harmless as-is and the
     symptom has a different cause.
  2. **Client half (a) is a non-issue.** The client already persists only complete
     selections (`assignUserLocation` behind `canSubmit` + early-return; CitySelector
     sets region+city atomically). No client change is warranted for "persist only a
     complete selection." If partial state exists in storage, it originates backend-
     side.

- **The crash itself is unlocalized.** I recommend getting the on-device stack trace
  (the 2026-05-31 update-product `regionAndCity` crash was localized only from Igor's
  pasted trace). Two leading candidates to check against the trace: (i) the setup →
  create auto-advance in `AddUpdateProductDialog` when the user's setup base site
  differs from the boot `selectedBaseSite`; (ii) the `ProductUserDetails.tsx:125`
  family if the home/detail surface renders a partially-set user.

- **Adjacent observation (Part 4b):**
  - One-line: `ProductUserDetails.tsx:125` derefs `regionAndCity.city.labelKey`
    while only `regionAndCity?.region` was checked on line 122 — crashes if `city`
    is null but `region` is set.
  - File: `src/components/product/ProductUserDetails.tsx:122-125`
  - Severity: high (user-facing crash, same family as the resolved 2026-05-31 bug).
  - I did not fix this because it is out of scope (off the first-run setup path) and
    the brief is audit-first / do-not-fix-blind.

- **Drafted `issues.md` entry (for Docs/QA to apply — target: new dated entry near
  the top of `issues.md`):**

  ```markdown
  ## 2026-06-12 — Mobile: product-create setup gate skips on an incomplete
  regionAndCity (presence-not-completeness) + unguarded ProductUserDetails deref

  **Repo:** `oglasino-expo` · **Severity:** high (user-facing: setup picker skipped,
  potential crash) · **Status:** open — audited, not fixed (audit-first session
  `oglasino-expo-first-run-setup-crash-1`)

  **Detail:** First-run report (Igor, on device): selecting base site + region/city
  crashes; afterwards the setup picker never reappears and base site + region look
  already selected. Trace (no code changed):
  - `productCreateGate.ts:25` gates the setup dialog on `!user.baseSite ||
    !user.regionAndCity` — i.e. the `regionAndCity` OBJECT being present, not on
    `.region` and `.city` both being non-null. `RegionAndCityDTO` is
    `{region: KeyLabelPair|null, city: KeyLabelPair|null}`. So a partial/empty
    `regionAndCity` object passes the gate and the picker is skipped on an incomplete
    setup. Fix candidate: `!user.baseSite || !user.regionAndCity?.region ||
    !user.regionAndCity?.city`. **Blocked on a backend confirmation** of whether
    `/auth/firebase-sync` ever returns a partial `regionAndCity` (vs `null`).
  - The client persist (`UserBasicDataSelectorDialog` → `assignUserLocation`) is
    already gated on a complete selection, so the brief's "persist before completion"
    premise does not hold on the client; any partial stored state is backend-side.
  - The first-run crash throw-site was NOT pinned from static reading (setup-path
    region/city derefs are guarded). Needs the on-device stack trace.
  - Live instance of the known `regionAndCity.<x>`-of-undefined crash family found
    OFF the first-run path: `ProductUserDetails.tsx:125` derefs
    `regionAndCity.city.labelKey` while only `regionAndCity?.region` is checked
    (line 122) — crashes if `city` is null but `region` is set.

  **Next:** (1) confirm backend `regionAndCity` shape for a half-set-up user;
  (2) capture the device crash stack trace to localize the throw; (3) then apply the
  gate-completeness tightening and guard `ProductUserDetails.tsx:125`.
  ```
