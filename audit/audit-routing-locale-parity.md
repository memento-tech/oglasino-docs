# Audit — oglasino-expo: routing locale resolution & web-compatibility (read-only)

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-30
**Mode:** READ-ONLY. Nothing changed.

**Question:** How does mobile resolve the current locale, and will it produce a
web-compatible `${locale}` segment (web emits `rs-sr` via `useRoutingLocale()`)
for the prefixed/shared product URL?

---

## TL;DR verdict

- **Mobile's locale source of truth** is the boot state machine `useBootStore`
  (`src/lib/store/bootStore.ts`), which holds base site and language as **two
  separate DTO slots** — `selectedBaseSite: BaseSiteDTO` and `language:
  LanguageDTO`. There is **no compound `rs-sr` value stored anywhere.**
- The web-format locale **can be composed exactly** as
  `` `${selectedBaseSite.code}-${language.code}` `` → `rs-sr`. Same backend, same
  DTOs, same codes → 1:1 with web's 10 routing locales. **No casing / separator /
  ordering mismatch.**
- **Neither of the two product-URL call sites uses that source today**, and one
  has a live bug:
  - `UploadedProductDialog.tsx` emits `https://www.oglasino.rs/product/{id}/{slug}`
    — **no locale segment, wrong domain** (`www.oglasino.rs`, not web's
    `oglasino.com`).
  - `ShareProductButton.tsx` emits
    `https://www.oglasino.com/[object Object]/product/{id}/{RAW name}` — it reads
    `useLocale()` from **`@react-navigation/native`, which returns `{ direction }`
    (LTR/RTL), not a locale**. The object stringifies to `[object Object]`.
- On web, the locale segment is **required**; an invalid/missing one **404s**.
  The current share URL (`[object Object]`, no hyphen) **404s on web today.**

---

## Item 1 — Mobile's locale / routing source

### The source of truth: `useBootStore`

`src/lib/store/bootStore.ts:80-117` — the boot state machine store. Relevant slots:

```ts
selectedBaseSite: BaseSiteDTO | null;   // :85
language: LanguageDTO | null;           // :86
```

- `BaseSiteDTO.code: string` — `src/lib/types/catalog/BaseSiteDTO.ts`. Concrete
  values are the backend base-site codes: `rs`, `rsmoto`, `me`.
- `LanguageDTO` — `src/lib/types/catalog/LanguageDTO.ts`:
  ```ts
  export type LanguageDTO = { code: string; active: boolean };
  ```
  `code` values: `sr`, `en`, `ru`, `cnr`.

These two slots are populated by **Gate 3** (`runBaseSiteGate`,
`bootStore.ts:269-295`) from AsyncStorage on a returning user, or by
`pickBaseSite` (`:307-333`) on first run. The store is the canonical wire source:

```ts
// src/lib/config/api.ts:44-46  (axios request interceptor)
const { selectedBaseSite, language } = useBootStore.getState();
if (selectedBaseSite) config.headers.set('X-Base-Site', selectedBaseSite.code);
if (language) config.headers.set('X-Lang', language.code);
```

So `selectedBaseSite.code` / `language.code` are exactly the codes the backend
keys everything on. **No compound locale string exists in the store or anywhere
else in the app.**

### Exact value format (quoted)

From `bootStore.test.ts`:
```ts
const langSr: LanguageDTO = { code: 'sr', active: true };   // :226
const site = buildSiteDTO();  // code 'rs', defaultLanguage langSr ('sr')  // :1021
useBootStore.getState().pickBaseSite('rs');                 // :1008
```
→ base site `code: 'rs'`, language `code: 'sr'`. **Separate `rs` and `sr`, never a
joined `rs-sr`.**

### Availability at the two PREFIXED-URL call sites — YES (synchronous)

Both call sites mount **inside the portal**, and the portal mounts **only when
`bootStatus ∈ {ready, updating}`** (Stack-level mount gate; decisions.md
2026-05-29 amendment #6 / issues.md 2026-05-28 "Always-mounted `<Stack>`…"
*fixed* entry). Gate 3 fills both slots before the machine ever leaves
`booting`, so by the time either component renders, `selectedBaseSite` and
`language` are **non-null and readable synchronously** — identically to how
`api.ts:44` reads them: `useBootStore.getState()` or the
`useBootStore((s) => …)` hook.

### …but neither call site reads that source today

**Call site A — create-success dialog.**
`src/components/dialog/dialogs/product-creation/UploadedProductDialog.tsx:109`:
```ts
setProductUrl(getNormalizedProductUrl(result.id, result.name, true));
```
`getNormalizedProductUrl` (`src/lib/utils/utils.ts:109-115`):
```ts
return `${withPrefix ? 'https://www.oglasino.rs' : ''}/product/${productId}/${normalizeProductName(productName)}`;
```
→ `https://www.oglasino.rs/product/{id}/{slug}`. **No locale segment. Domain is
`www.oglasino.rs`, not web's `oglasino.com`.** The helper takes no `locale`
argument at all (web's helper takes a 4th `locale?` arg — see Item 2).

**Call site B — share button.**
`src/components/product/ShareProductButton.tsx` — **does NOT call
`getNormalizedProductUrl`.** It builds the URL inline:
```ts
import { useLocale } from '@react-navigation/native';   // :1
const locale = useLocale();                             // :23
...
message: `${title}\nhttps://www.oglasino.com/${locale}/product/${productId}/${productName}`,  // :29
```

### Live bug in call site B (`useLocale`)

`useLocale()` from `@react-navigation/native` **v7.1.31** returns the text
**direction**, not a locale:
```ts
// node_modules/@react-navigation/native/lib/.../useLocale.{d.ts,js}
export function useLocale(): { direction: LocaleDirection } { ... }
```
The app never sets a direction on the navigation container (only `ThemeProvider`
is wired — `app/_layout.tsx:75`), so `direction` defaults to `'ltr'`. The object
interpolated into the template string yields the literal string
**`[object Object]`**. Current runtime output:

```
https://www.oglasino.com/[object Object]/product/{id}/{RAW productName}
```

Two extra defects on this same line: the domain is `www.oglasino.com` (web uses
bare `oglasino.com`), and `productName` is passed **raw** — it is never run
through `normalizeProductName`, so spaces / diacritics / `/` survive in the slug.

> Other callers of `getNormalizedProductUrl` (`SearchInput.tsx`,
> `PortalProductCard.tsx`, `ExtraProductCard.tsx`, `ProductReview.tsx`,
> `PushNotificationsInit.tsx`, and `UploadedProductDialog.tsx:110`) all use the
> **relative** form (`withPrefix=false`) for in-app `router.push` — locale is
> irrelevant for those, and they are out of scope for the prefixed-URL parity.

---

## Item 2 — Compare to web's format

### Web's value

`oglasino-web/src/i18n/useRoutingLocale.ts`:
```ts
export function useRoutingLocale(): string {
  const params = useParams<{ locale: string }>();
  return params.locale;   // the [locale] URL segment, e.g. 'rs-sr'
}
```
Valid values (`oglasino-web/src/i18n/routing.ts`):
```
rs-sr, rs-en, rs-ru, rsmoto-sr, rsmoto-en, rsmoto-ru, me-sr, me-cnr, me-en, me-ru
```
Format: **`{baseSite}-{language}`**, lowercase, single hyphen, baseSite first.
Web's prefixed helper (per the web audit `audit-create-update-flow.md:462-465`):
```ts
getNormalizedProductUrl(productId, productName, withPrefix=false, locale?)
return `${withPrefix ? `https://oglasino.com/${locale}` : ''}/product/${productId}/${normalizeProductName(productName)}`;
// create passes useRoutingLocale(), e.g. rs-sr → https://oglasino.com/rs-sr/product/42/opel-astra
```

### Does mobile already have a same-format value? — NO

Mobile stores base site and language **separately**. There is no compound locale
string anywhere in the app.

### Can it be composed from the separate fields? — YES, exactly

```ts
const { selectedBaseSite, language } = useBootStore.getState();
const routingLocale = `${selectedBaseSite.code}-${language.code}`; // → 'rs-sr'
```
- Fields: `selectedBaseSite.code` + `language.code`.
- Separator: `-`.
- Order: baseSite, then language.

This is provably 1:1 with web's 10 locales because **both clients consume the
same backend** base-site/language DTOs. Web itself builds its locale list the
same way — `oglasino-web/proxy.ts:12-18` ("Static mirror of
baseSite.allowedLanguages derived from routing.locales"), and web's `sitemap.ts`
prefixes paths with `${bs.code}-${lang.code}`. Every valid backend
`(baseSite, allowedLanguage)` pair composes to exactly one of the 10 routing
locales.

### Mismatch risk

| Dimension | Risk | Notes |
|-----------|------|-------|
| Casing | none | both lowercase |
| Separator | none | both `-` |
| Ordering | none | both `{baseSite}-{language}` |
| Locale coverage | none (format) | composition lands on one of web's 10 |
| `me-cnr` (Montenegrin) | none | `me-cnr` is a first-class web routing locale; compose from `me` + `cnr` directly. Conventions Part 9's me/cnr→sr aliasing is a **formatter** concern, NOT the routing locale — do not alias when composing the URL segment. |

One **non-format** caveat for implementation (not a mismatch, a coverage note):
the composed value is only as complete as mobile's resolved base site. A device
that only ever resolved the `rs` base site will only ever produce `rs-*`; it
cannot fabricate `rsmoto-*`/`me-*`. That's correct behavior (the URL reflects the
site the user is actually on) and matches web, where the segment is the user's
current locale — just flagging that there is no "all locales" enumeration on the
client, nor is one needed for this feature.

---

## Item 3 — How web's product route consumes the locale segment

### Required? — YES. Invalid/missing → 404.

The product page lives under `app/[locale]/…`. The locale layout hard-gates:
```ts
// oglasino-web/app/[locale]/layout.tsx:26-27
if (!hasLocale(routing.locales, locale)) {
  notFound();   // 404
}
```

### Valid values

The 10 routing locales listed in Item 2.

### Wrong-locale behavior (the proxy is the authority)

`oglasino-web/proxy.ts`:
```ts
const LOCALE_PREFIX = /^\/([^/-]+)-([^/]+)(\/.*)?$/;   // :10  — requires {tenant}-{lang}
```
- **Valid shape, invalid tenant-lang combo** (e.g. `/rs-cnr/…`): proxy
  **redirects** to the cookie's lang, else the tenant's first allowed lang
  (`:56-64`), then serves.
- **Valid locale ≠ viewer's cookie locale**: proxy **redirects** to the cookie
  locale (`:67-70`), then serves. So a copied `rs-sr` link may be redirected to
  e.g. `rs-en` for a viewer whose cookie is `en` — **the product still loads**,
  just in the viewer's language.
- **Malformed / no hyphen** (e.g. `[object Object]`, bare `rs`, bare `sr`): the
  regex misses → falls through → `hasLocale` fails → **`notFound()` → 404**.

### Consequence for the current mobile URLs

- ShareProductButton's `…/[object Object]/product/…` — segment has **no hyphen**
  → **404 on web today.**
- UploadedProductDialog's `https://www.oglasino.rs/product/…` — **no `[locale]`
  segment** (and wrong domain). The path `/product/…` would treat `product` as
  the `[locale]` segment → not a valid locale → **404** (independent of the
  `.rs` domain issue).

### What a copied URL's locale segment MUST contain to resolve

A valid `{baseSite}-{language}` routing locale — i.e. the composed
`${selectedBaseSite.code}-${language.code}` value. Any of the 10 is fine; the
viewer's cookie may redirect the language, but the correct product resolves.

### UNDETERMINED

- **`www.oglasino.com` vs apex `oglasino.com`.** Web's canonical uses bare
  `oglasino.com` (no `www`). Whether a `www.` host 301-redirects to the apex
  before the Next app runs is a **Cloudflare-router / DNS** concern, not visible
  in the web app repo. Mark UNDETERMINED — verify on the router side. (Mobile
  should match web's bare `oglasino.com` regardless, to avoid depending on a
  www→apex redirect.)

---

## Recommendations for the parity brief (informational — no change made here)

1. **One helper, web-shaped.** Give mobile's `getNormalizedProductUrl` the same
   4-arg signature as web (`…, withPrefix, locale?`) emitting
   `https://oglasino.com/${locale}/product/${id}/${normalizeProductName(name)}`
   — bare `oglasino.com`, locale segment, normalized slug.
2. **Compose the locale at the call sites** from
   `useBootStore` → `` `${selectedBaseSite.code}-${language.code}` ``.
3. **Fix ShareProductButton** to use the helper (remove the
   `@react-navigation/native` `useLocale()` misuse and the raw-name bug); it is
   currently emitting `[object Object]` and an un-normalized slug.
4. Decide whether the dialog's `www.oglasino.rs` and the share's
   `www.oglasino.com` both move to bare `oglasino.com` (recommend yes, to match
   web canonical and dodge the UNDETERMINED www→apex question).

---

## Config-file impact

- None (read-only audit). No `conventions.md` / `decisions.md` / `state.md` /
  `issues.md` edits required from this audit.
