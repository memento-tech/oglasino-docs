# Audit — Accent region/city URL hydrate bug (oglasino-web)

**Repo:** oglasino-web · **Branch:** dev · **Mode:** READ-ONLY (no code changes)
**Date:** 2026-06-05
**Scope:** Confirm and finalize the fix shape for accented region/city names failing to hydrate
from the URL (`/rs-sr?regions=nis` does not select Niš; `?regions=beograd` does select Beograd).
**Predecessor audit:** `.agent/audit-topfilter-hydrate-bug.md` (traced the bug to the region/city
hydrate block).

---

## TL;DR

The bug is **still present** in current code at `FilterManager.tsx:185-194`, unchanged by the
recent region/city SEND reshape. The hydrate maps URL tokens through the **lossy** `fromQueryParam`
and compares the result to the **raw** `t(labelKey)`. `fromQueryParam` restores spaces, dash tokens,
and capitalization but **never restores accents**, so `nis → "Nis"` never equals the label `"Niš"`.
ASCII names (`beograd → "Beograd" === "Beograd"`) match; accented names (`nis`, `cacak`) do not.

The SEND path (`FilterManager.tsx:270-282`), the `f_<key>` option hydrate
(`FilterManager.tsx:121`), and the request-builder `getRegionAndCitiesData`
(`filtersHelper.ts:223-244`) all already use the **correct** strategy: compare the raw URL token to
`toQueryParam(t(labelKey))`. The fix aligns the one broken block to that existing strategy — it is
**consistent, not novel**. There is exactly one broken site; no other place matches region/city by a
lossy token.

---

## Q1 — Re-confirm against CURRENT code: bug still present post-send-fix

The region/city HYDRATE block, verbatim with current line numbers (`FilterManager.tsx`):

```ts
184	    // regions/cities
185	    const regionValues = params.get('regions')?.split(',').map(fromQueryParam) || [];
186	    const cityValues = params.get('cities')?.split(',').map(fromQueryParam) || [];
187	
188	    setRegionCityValues({
189	      regions: baseSite.regions?.filter((r) => regionValues.includes(t(r.labelKey))) || [],
190	      cities:
191	        baseSite.regions
192	          ?.flatMap((r) => r.cities || [])
193	          .filter((c) => cityValues.includes(t(c.labelKey))) || [],
194	    });
```

**Confirmed:** still `fromQueryParam` on the token side (185-186) and **raw** `t(r.labelKey)` /
`t(c.labelKey)` on the label side (189, 193). The recent SEND reshape did not touch this block. Bug
is live.

**Why it fails (mechanism), with the helpers as written (`filtersHelper.ts:14-36`):**

- `toQueryParam('Niš')`: NFD-normalize strips the combining caron → `"Nis"`, lowercase → **`"nis"`**.
  So the SEND path emits the token `nis` for Niš.
- On hydrate, `fromQueryParam('nis')` → dashes→spaces (none), `_d_`→`-` (none), capitalize first
  letters → **`"Nis"`**. Accent is **not** restored (there is no information to restore it from).
- Compare `regionValues.includes(t(r.labelKey))` → does `["Nis"]` include `"Niš"`? **No.** Niš is
  dropped from the hydrated selection.
- For Beograd: `fromQueryParam('beograd')` → `"Beograd"`, and `t(labelKey) === "Beograd"` → **match.**
  This is why ASCII regions hydrate and accented ones silently don't.

---

## Q2 — SEND path and `f_<key>` option hydrate both use the correct strategy

**SEND path** (`FilterManager.tsx:270-282`) — compares nothing; it emits `toQueryParam(t(label))`:

```ts
270	    if (selectedRegionsAndCities.regions.length) {
271	      params.set(
272	        'regions',
273	        selectedRegionsAndCities.regions.map((r) => toQueryParam(t(r.labelKey))).join(',')
274	      );
275	    }
276	
277	    if (selectedRegionsAndCities.cities.length) {
278	      params.set(
279	        'cities',
280	        selectedRegionsAndCities.cities.map((c) => toQueryParam(t(c.labelKey))).join(',')
281	      );
282	    }
```

**`f_<key>` option hydrate** (`FilterManager.tsx:121`) — compares the **raw** token to
`toQueryParam(t(label))`:

```ts
120	          filter.options.forEach((opt) => {
121	            if (optionValues.includes(toQueryParam(t(opt.labelKey)))) {
```

**Request-builder `getRegionAndCitiesData`** (`filtersHelper.ts:223-244`) — same correct strategy,
and it is the function that actually builds the region/city IDs sent to the backend:

```ts
232	  const regions =
233	    baseSite.regions?.filter((r) => regionValues.includes(toQueryParam(t(r.labelKey)))) || [];
234	
235	  const cities =
236	    baseSite.regions
237	      ?.flatMap((r) => r.cities || [])
238	      .filter((c) => cityValues.includes(toQueryParam(t(c.labelKey)))) || [];
```

(Note `regionValues`/`cityValues` here are the raw split tokens — **no** `fromQueryParam` map; line
228-230.) The selected-filter option matcher in the same file does the same
(`filtersHelper.ts:161`).

**Confirmed:** the correct strategy is "compare raw URL token to `toQueryParam(t(label))`". It is used
in 4 places. The region/city UI hydrate at 185-194 is the **only** outlier using the lossy
`fromQueryParam`-vs-raw-`t()` comparison. Aligning it is consistent with the rest of the file.

> **Consequence worth noting:** because `getRegionAndCitiesData` (the request builder) is already
> correct, a direct deep-link like `/rs-sr?regions=nis` likely **filters the product results
> correctly** (backend receives Niš's region id) while the **filter chip/selector UI shows nothing
> selected** — the visible symptom Igor confirmed. The two paths disagree today; the fix makes the UI
> agree with the request builder.

---

## Q3 — Exact minimal edit

Change the region/city hydrate to match the request-builder strategy: drop the `fromQueryParam` map,
keep the raw tokens, and wrap the label in `toQueryParam`.

**Before** (`FilterManager.tsx:185-194`):

```ts
    const regionValues = params.get('regions')?.split(',').map(fromQueryParam) || [];
    const cityValues = params.get('cities')?.split(',').map(fromQueryParam) || [];

    setRegionCityValues({
      regions: baseSite.regions?.filter((r) => regionValues.includes(t(r.labelKey))) || [],
      cities:
        baseSite.regions
          ?.flatMap((r) => r.cities || [])
          .filter((c) => cityValues.includes(t(c.labelKey))) || [],
    });
```

**After:**

```ts
    const regionValues = params.get('regions')?.split(',') || [];
    const cityValues = params.get('cities')?.split(',') || [];

    setRegionCityValues({
      regions:
        baseSite.regions?.filter((r) => regionValues.includes(toQueryParam(t(r.labelKey)))) || [],
      cities:
        baseSite.regions
          ?.flatMap((r) => r.cities || [])
          .filter((c) => cityValues.includes(toQueryParam(t(c.labelKey)))) || [],
    });
```

Net change: remove `.map(fromQueryParam)` from both token lines (185-186); wrap both label reads in
`toQueryParam(...)` (189, 193). No other lines touched. `toQueryParam` is already imported
(`FilterManager.tsx:13`).

**Match verification after the change** (token side is the raw URL value; label side is
`toQueryParam(t(label))`):

| URL token | `toQueryParam(t(label))` | Match? |
|-----------|--------------------------|--------|
| `beograd` (Beograd, ASCII) | `toQueryParam("Beograd")` = `beograd` | ✅ yes — still matches |
| `nis` (Niš) | `toQueryParam("Niš")` = `nis` | ✅ now matches |
| `cacak` (Čačak) | `toQueryParam("Čačak")` = `cacak` | ✅ now matches |
| `novi-sad` (Novi Sad, multi-word) | `toQueryParam("Novi Sad")` = `novi-sad` | ✅ matches |

By construction the hydrate token equals the SEND token for every name, because both sides now run the
label through the **same** `toQueryParam`. ASCII names keep matching; accented names now match.

**`fromQueryParam` import:** after this edit, `FilterManager.tsx` still uses `fromQueryParam` at line
97 (`search_text`), so the import stays. (See Q4 — that remaining use is a different, lower-severity
concern, not part of this fix.) No unused-import cleanup is triggered by this edit.

---

## Q4 — Every `fromQueryParam` / `toQueryParam` site, classified

Repo-wide grep (`src/`, `app/`). Region/city-token matching is the bug class; everything else is
classified for completeness.

### `fromQueryParam` (4 call-site groups)

| Site | Purpose | Verdict |
|------|---------|---------|
| `FilterManager.tsx:185-186` | **region/city token → label match (hydrate)** | ❌ **INCORRECT — the bug.** Fixed by Q3. The only region/city match-by-token site that is wrong. |
| `FilterManager.tsx:97` | `search_text` token → search-box **display** value | ⚠️ Lossy but **not a match-failure** (no label set to match against). Reconstructs a display string from a token that already lost accents/case on the SEND side (`toQueryParam`, line 214). An accented search like "Čačak" shows as "Cacak" in the box. **Out of scope** for this fix — it cannot be fixed by a `toQueryParam` swap (nothing to compare to); it is inherent round-trip lossiness. Flagged as an adjacent observation, low severity. |
| `UserFilters.tsx:43,44,50,51` | admin `displayName` / `email` token → filter-input **display** value | ⚠️ Same class as `search_text`: free-text display reconstruction, **not** region/city matching. Note: this panel's region/city is matched by **id** (`UserFilters.tsx:30-34`, `params.regionId`/`cityId`), so it is **immune** to the accent bug. Out of scope. |
| `filtersHelper.ts:25` | the `fromQueryParam` **definition** | n/a |

### `toQueryParam` (all correct — the send / match-key strategy)

| Site | Purpose | Verdict |
|------|---------|---------|
| `FilterManager.tsx:121` | `f_<key>` option hydrate match | ✅ correct (raw token vs `toQueryParam(t(label))`) |
| `FilterManager.tsx:214,224,273,280` | SYNC-to-URL (search_text, options, regions, cities) | ✅ correct (send) |
| `filtersHelper.ts:161` | request-builder option match | ✅ correct |
| `filtersHelper.ts:233,238` | request-builder region/city match | ✅ correct — the reference the fix aligns to |
| `FiltersPanel.tsx:61` | admin generic value → URL | ✅ correct (send) |
| `SearchInput.tsx:141` | search term → URL | ✅ correct (send) |
| `filtersHelper.ts:14` | the `toQueryParam` **definition** | n/a |

**Conclusion for Q4:** there is **exactly one** region/city match-by-token site that uses the lossy
pattern — `FilterManager.tsx:185-194`. No other location reads a region or city from a URL token and
matches it against a label. The remaining two `fromQueryParam` reads (`search_text`, admin
`displayName`/`email`) are free-text display reconstruction — a separate, lower-severity lossiness
that the Q3 edit neither touches nor needs to.

---

## Q5 — Cheap test that locks the parity

**Recommended file:** `src/lib/utils/filtersHelper.test.ts` (vitest; project already runs
`vitest run` via `npm test`, config at `vitest.config.ts`, colocated `*.test.ts` is the in-repo
pattern — e.g. `parseProductValidationErrors.test.ts`). None exists for `filtersHelper` yet.

The exact "parity between hydrate matching and send matching" is awkward to assert directly because
the hydrate predicate is **inlined** in a `useEffect` in `FilterManager.tsx` (not an exported pure
function), while the send/request side lives in the exported helpers. Two options:

**Option A (cheapest, zero refactor — recommended).** Test the pure helpers, which both sides share.
The bug is fully characterized by `toQueryParam`/`fromQueryParam` behavior on an accented fixture, so
a 6-line pure test locks it with no React, no mocks:

```ts
import { describe, expect, it } from 'vitest';
import { toQueryParam, fromQueryParam } from './filtersHelper';

describe('region/city query-param accent handling', () => {
  // SEND and (fixed) HYDRATE both key on toQueryParam(label); this is the parity contract.
  it('keys accented labels to an accent-stripped token on both send and match', () => {
    expect(toQueryParam('Niš')).toBe('nis');
    expect(toQueryParam('Čačak')).toBe('cacak');
    expect(toQueryParam('Beograd')).toBe('beograd');
    expect(toQueryParam('Novi Sad')).toBe('novi-sad');
  });

  // Documents WHY the old hydrate failed: fromQueryParam cannot restore the accent,
  // so the token never equals the raw label.
  it('fromQueryParam cannot reconstruct the accented label (the old-hydrate failure)', () => {
    expect(fromQueryParam('nis')).not.toBe('Niš'); // was compared to raw t(label) === "Niš"
    expect(fromQueryParam('beograd')).toBe('Beograd'); // ASCII happened to round-trip — masked the bug
  });
});
```

This locks the contract that the fix relies on (token === `toQueryParam(label)`), proves the ASCII
case still works, and documents the accented regression so a future `fromQueryParam` re-introduction
fails the suite.

**Option B (truer parity, small refactor — optional, flag-only).** Extract the now-identical match
predicate into one exported helper, e.g.
`matchesRegionCityToken(tokens: string[], label: string) => tokens.includes(toQueryParam(label))`,
use it in **both** `FilterManager.tsx` (hydrate) and `filtersHelper.ts`
(`getRegionAndCitiesData`), and unit-test that helper with the accented fixture. This gives a single
seam exercised by both code paths (genuine hydrate↔send parity) and removes the duplicated predicate.
It earns its place (two real callers + dedupe), but it is **more than the minimal Q3 edit** and is a
separate decision for Mastermind — not required to fix the bug. If chosen, the test would assert the
helper returns the region for `['nis']`/`'Niš'` and `['beograd']`/`'Beograd'`.

**Recommendation:** ship Option A with the Q3 fix (cheap, no surface change). Treat Option B as an
optional follow-up if the duplication is considered worth removing.

---

## Closure

- **Read-only:** no code changed; Part 4a N/A per the brief.
- **Single fix site confirmed:** `FilterManager.tsx:185-194`. The fix aligns it to the strategy
  already used in 4 other places — consistent, not novel.
- **No config-file dependency.** Nothing here requires an edit to `conventions.md`, `decisions.md`,
  `state.md`, or `issues.md`. (The bug is already implicitly covered by the prior
  `audit-topfilter-hydrate-bug.md` lineage; whether to log a new `issues.md` entry is Mastermind's
  call — not drafted here.)
- **Adjacent observation (low severity, out of scope):** `search_text` hydrate
  (`FilterManager.tsx:97`) and admin `displayName`/`email` (`UserFilters.tsx`) reconstruct **display**
  strings from accent-lossy tokens via `fromQueryParam`; an accented search term or username renders
  de-accented in its input box. Not a match-failure and not fixable by a `toQueryParam` swap. Flagged
  only; no fix proposed.
