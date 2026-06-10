# Audit — legal-localization (oglasino-web)

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-06-10
**Type:** Read-only audit (Phase 2). No code changed.
**Scope:** Privacy Policy + Terms of Use pages, the GitHub raw markdown URLs they
build, locale resolution available to those server components, the fetch
caching wrapper, and the trust boundary.

Every file:line claim below was verified with both `view` (Read) and `rg`.

---

## 1. The two legal page files and their hardcoded URLs

Both pages are async **server components**. Neither uses the locale in the page
body today — the URL is a hardcoded English string literal. The locale is
resolved only inside `generateMetadata`, not inside the page render.

### `app/[locale]/(portal)/(public)/privacy/page.tsx`

The URL is built as a bare string literal passed to `MarkdownViewer`
(lines 22–26):

```tsx
        <MarkdownViewer
          url={
            'https://raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/privacy-policy.en.md'
          }
        />
```

The full hardcoded URL string (`page.tsx:24`):

```
https://raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/privacy-policy.en.md
```

There is a standing comment above the component (lines 15–17) pointing at the
swap task:

```tsx
// Hardcoded English: per-locale legal content is pending lawyer review.
// See issues.md 2026-05-27 — "Per-locale legal markdown content (privacy + terms)"
// for the swap-to-locale-aware task.
```

### `app/[locale]/(portal)/(public)/terms/page.tsx`

Identical shape (lines 22–26):

```tsx
        <MarkdownViewer
          url={
            'https://raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/terms-of-use.en.md'
          }
        />
```

The full hardcoded URL string (`page.tsx:24`):

```
https://raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/terms-of-use.en.md
```

Same standing comment block (lines 15–17).

**`rg` cross-check** — the two raw-githubusercontent URLs in `app/` are exactly these two and no others:

```
app/[locale]/(portal)/(public)/terms/page.tsx:24:   ...terms-of-use.en.md'
app/[locale]/(portal)/(public)/privacy/page.tsx:24: ...privacy-policy.en.md'
```

---

## 2. Locale / language resolution available in these server components

### 2a. Is `getRoutingLocale()` / `getLocale()` callable here?

**Yes.** `getRoutingLocale()` is already imported and called in both files
today — inside `generateMetadata`, not inside the page render.

Import (`privacy/page.tsx:2`, `terms/page.tsx:2`, identical):

```tsx
import { getRoutingLocale } from '@/src/i18n/getRoutingLocale';
```

How the locale is currently obtained (`privacy/page.tsx:8-13`):

```tsx
export async function generateMetadata(): Promise<Metadata> {
  const t = await getTranslations(TranslationNamespaceEnum.METADATA);
  const locale = await getRoutingLocale();

  return generatePrivacyPageMetadata(t, locale);
}
```

`terms/page.tsx:8-13` is identical except it calls `generateTermsPageMetadata`.

The page render functions themselves (`PrivacyPage`, `TermsPage`) do **not**
obtain a locale at all — they only return JSX with the hardcoded URL.

`getRoutingLocale` is an async server helper that reads the
`x-next-intl-locale` request header (set by next-intl middleware from the URL
segment) and returns the **compound routing locale**, falling back to
`routing.defaultLocale` if the header is missing/invalid
(`src/i18n/getRoutingLocale.ts:9-15`):

```tsx
export async function getRoutingLocale(): Promise<string> {
  const h = await headers();
  const locale = h.get('x-next-intl-locale');
  return locale && routing.locales.includes(locale as (typeof routing.locales)[number])
    ? locale
    : routing.defaultLocale;
}
```

Because it relies on `headers()` and the next-intl request context, it is
callable anywhere in these async server components — including the page body, if
the swap moves it there. (`getLocale()` from `next-intl/server` is also
callable, but note its file-level comment in `getRoutingLocale.ts:5-8` says
`getLocale()` returns the **bare formatter locale** after the `request.ts`
change, whereas `getRoutingLocale()` returns the **compound** locale that
`getTenantLocale` parsing needs.)

The routing locales are all **compound** `tenant-lang` strings
(`src/i18n/routing.ts:5-16`):

```
rs-sr, rs-en, rs-ru, rsmoto-sr, rsmoto-en, rsmoto-ru, me-sr, me-cnr, me-en, me-ru
```

`defaultLocale` is `rs-sr`. `cnr` appears only in `me-cnr`.

### 2b. Is there a `getTenantLocale(locale)` helper yielding a bare language code?

**Yes — it exists and is the right tool.**
`src/translations/lib/util/getTenantLocale.ts`.

Signature (`getTenantLocale.ts:25`):

```tsx
export function getTenantLocale(locale: string): TenantLocale | undefined
```

Return type (`getTenantLocale.ts:1-5`):

```tsx
export type TenantLocale = {
  tenant: string;       // e.g. 'me'  (segment before the dash)
  locale: string;       // BCP-47-ish intl locale, e.g. 'cnr-ME' (via mapToIntlLocale)
  oglasinoLocale: string; // BARE language code, e.g. 'cnr' (segment after the dash)
};
```

Body (`getTenantLocale.ts:25-33`):

```tsx
export function getTenantLocale(locale: string): TenantLocale | undefined {
  if (!locale) return undefined;
  const [tenant, lang] = locale.split('-');
  const intlLocale = mapToIntlLocale(locale);
  return { tenant, locale: intlLocale, oglasinoLocale: lang };
}
```

**The bare language code the brief asks for is the `oglasinoLocale` field** — it
is literally `locale.split('-')[1]`. For each routing locale this yields:

| routing locale | `oglasinoLocale` (bare lang) |
| -------------- | ---------------------------- |
| `rs-sr` / `rsmoto-sr` / `me-sr` | `sr` |
| `rs-en` / `rsmoto-en` / `me-en` | `en` |
| `rs-ru` / `rsmoto-ru` / `me-ru` | `ru` |
| `me-cnr` | `cnr` |

So `getTenantLocale(getRoutingLocale-result).oglasinoLocale ∈ {sr, en, ru, cnr}`
— exactly the bare-language set in the brief. The branch the feature wants
(`sr`/`cnr` → `.sr.md`, everything else → `.en.md`) maps cleanly onto this
field.

This is the established pattern across the codebase — the same destructure
`const { ..., oglasinoLocale: lang } = getTenantLocale(locale)` appears in
`src/lib/config/api.ts:97`, `src/i18n/request.ts:14`,
`src/lib/service/nextCalls/userService.ts:9`,
`src/lib/service/nextCalls/productsSearchService.ts:32,74`,
`src/lib/service/nextCalls/metadataSnapshotService.ts:45`, and others — so the
legal-page swap would conform to existing convention, not introduce a new one.

**One caveat (type-safety):** the return type is `TenantLocale | undefined`. The
swap snippet drafted in `issues.md` (2026-05-27) destructures the result
directly — `const { oglasinoLocale: lang } = getTenantLocale(locale)` — which is
a TS error under the project's strictness because the result can be `undefined`.
In practice `getRoutingLocale()` only ever returns a value in `routing.locales`
(or the default), so `getTenantLocale` will never actually return `undefined`
here — but the implementer should either narrow it (`const tl =
getTenantLocale(locale); const lang = tl?.oglasinoLocale ?? 'en';`) or the
`tsc --noEmit` gate will fail. Flagged for the Phase-5 brief, not fixed here
(read-only audit).

---

## 3. Do the two `.en.md` URLs match the target files on `refs/heads/main`?

**Yes — exact match.** The two hardcoded URLs use the `refs/heads/main` path and
the filenames `privacy-policy.en.md` / `terms-of-use.en.md`:

```
https://raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/privacy-policy.en.md
https://raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/terms-of-use.en.md
```

- Org/repo: `memento-tech/oglasino-platform` ✓
- Ref path: `refs/heads/main` ✓
- Filenames: `privacy-policy.en.md`, `terms-of-use.en.md` ✓

The brief states the `.sr.md` siblings (`privacy-policy.sr.md`,
`terms-of-use.sr.md`) now exist on that same main branch and that the English
files are unchanged. Note: this audit verifies the **URLs as written in the
web code**; it cannot reach across to the `oglasino-platform` repo (not a sibling
of this working tree) to confirm the remote files' existence — that confirmation
is owed by whoever can see `oglasino-platform`. The web-side claim (the code's
English URLs are exactly these two, on `refs/heads/main`) is confirmed.

The language-varying swap is purely a substitution of the `en` token in the
filename — `privacy-policy.${lang}.md` / `terms-of-use.${lang}.md` — with
everything else (org, repo, ref path) unchanged.

---

## 4. Caching / revalidation wrapper on these fetches

The fetch lives in the shared `MarkdownViewer` server component, not in the
pages themselves: `src/components/server/MarkdownViewer.tsx` (imported by both
pages as `@/components/server/MarkdownViewer`, which resolves to
`src/components/...` via the `"@/*": ["./*", "./src/*"]` tsconfig alias).

The fetch and its options (`MarkdownViewer.tsx:13-16`):

```tsx
    const res = await fetch(url, {
      cache: 'force-cache',
      next: { revalidate: 3600 },
    });
```

- `cache: 'force-cache'` + `next: { revalidate: 3600 }` — the response is cached
  in Next's Data Cache and revalidated at most once per hour (3600 s).
- **No cache tags** (`next.tags`) are set, so there is no tag-based invalidation
  to keep in sync.
- The cache key is the **full URL**. Because the URL is the only argument, a
  language-varying URL (`...privacy-policy.sr.md` vs `...privacy-policy.en.md`)
  becomes a **distinct cache entry automatically** — each language gets its own
  independently-revalidated 1-hour-cached copy. No code change to the caching is
  needed for the swap; the existing per-URL keying already does the right thing.
- On any fetch failure (`!res.ok` or a thrown error) the component swallows it
  and renders the translated `markdown.field.load` error string
  (`MarkdownViewer.tsx:18-23`) — so a missing `.sr.md` would degrade to the
  error message, not a crash. Worth knowing for the swap: if a `.sr.md` file is
  ever absent, sr/cnr users see the load-error string rather than falling back
  to English. The fallback-to-English behavior the feature wants must be encoded
  in the URL-building ternary (compute `effectiveLang` before the fetch), not in
  `MarkdownViewer`.

No other caching/revalidation wrapper (no `unstable_cache`, no route segment
`export const revalidate`, no `fetchCache`) applies on these pages — verified by
reading both page files top-to-bottom; neither sets any route-segment cache
config.

---

## 5. Trust boundary

**None — confirmed.** The language code on these pages is display-only. It is
used in exactly two places: (a) `generateMetadata` passes the compound locale to
the metadata generators (page title/description/OG — display), and (b) after the
swap it would select which markdown filename to fetch (display). It is **not**
read into any authentication, authorization, moderation, or state-transition
decision; the pages are public static content with no request DTO, no auth call,
and no server-side mutation. A user manipulating the locale segment can at most
change which language of public legal text they see.

---

## Summary for Mastermind (seam-relevant facts)

1. **The swap is a one-line URL change per page**, gated on the bare language
   from `getTenantLocale(getRoutingLocale()).oglasinoLocale`, mapping
   `sr`/`cnr` → `sr`, everything else (`en`/`ru`) → `en`. This matches the
   `issues.md` 2026-05-27 drafted swap shape and the existing codebase pattern.
2. **`getTenantLocale` returns `TenantLocale | undefined`** — the drafted
   destructure in `issues.md` would fail `tsc --noEmit` as written; the Phase-5
   brief should narrow the result (`?.oglasinoLocale ?? 'en'`).
3. **Caching needs no change** — per-URL keying already gives each language its
   own revalidated cache entry; there are no cache tags to update.
4. **Fallback must live in the URL ternary, not the fetch** — `MarkdownViewer`
   renders an error string (not English) on a 404, so the `en` fallback for
   `ru`/missing-`sr` has to be computed before the fetch.
5. **No trust boundary** — language is display-only on these pages.
