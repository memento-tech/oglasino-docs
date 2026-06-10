# Audit — Legal localization (oglasino-expo)

**Type:** Pre-feature audit, READ-ONLY. No code changed, nothing installed, no prebuild/eas run, nothing committed.
**Date:** 2026-06-10
**Branch:** `dev` (`git branch --show-current` → `dev`). The brief names no branch.
**Spec note:** there is no `oglasino-docs/features/legal-localization.md` yet — this audit precedes the spec. The feature is described only by the brief: `sr`/`cnr` → `.sr.md`, everything else (`en`/`ru`) → `.en.md`, for both Privacy Policy and Terms of Use; the `.en.md` files fetched today are unchanged, and a `.sr.md`-selection branch is added.

Every file:line and present/absent claim below is backed by BOTH a `view` (Read) and an independent `rg`. Where a string is claimed absent, the `rg` command and its empty result are shown. `view` and `rg` agreed on every citation.

---

## 1 — The two in-app legal screens (URL-building lines)

There are exactly two `MarkdownViewer` call sites in the app. `rg -n "MarkdownViewer" app/ src/` returns only `src/components/MarkdownViewer.tsx` (the component) and the two screens below — no third legal surface.

### Privacy Policy screen

- **File:** `app/(portal)/(public)/privacy.tsx`
- The URL is a hardcoded inline string literal passed as the `url` prop (lines 10–14):

```tsx
      <MarkdownViewer
        url={
          'https://raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/privacy-policy.en.md'
        }
      />
```

Exact URL string (`privacy.tsx:12`):
`https://raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/privacy-policy.en.md`

### Terms of Use screen

- **File:** `app/(portal)/(public)/terms.tsx`
- The URL is a hardcoded inline string literal passed as the `url` prop (lines 10–14):

```tsx
      <MarkdownViewer
        url={
          'https://raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/terms-of-use.en.md'
        }
      />
```

Exact URL string (`terms.tsx:12`):
`https://raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/terms-of-use.en.md`

**Note for the implementer:** the URL is fully inlined per-screen — there is no shared constant or URL-builder helper. `rg -n "githubusercontent" app/ src/` returns only these two screen lines. So the `.sr.md`-selection branch must be added in each screen (or a new helper introduced), and neither screen currently reads the active language at all (see §2 — no language import in either file).

---

## 2 — Active-language source available to these screens

### How the current display language is read on mobile

The active display language lives in **`useBootStore`**'s `language` slot, typed `LanguageDTO | null` (`src/lib/store/bootStore.ts:93`):

```ts
  language: LanguageDTO | null;
```

`LanguageDTO` is a **bare-code shape**, not compound (`src/lib/types/catalog/LanguageDTO.ts:1–4`):

```ts
export type LanguageDTO = {
  code: string;
  active: boolean;
};
```

The canonical accessor pattern used everywhere is a Zustand selector returning the DTO, then `.code` for the bare code string. Confirmed live call sites:

- `src/components/product/ProductList.tsx:51` — `const selectedLanguage = useBootStore((s) => s.language);` then `selectedLanguage?.code` (`ProductList.tsx:165`).
- `src/components/dialog/dialogs/PortalConfigDialog.tsx:51` — `const selectedLanguage = useBootStore((s) => s.language);` then `selectedLanguage.code` (`PortalConfigDialog.tsx:128`).
- `src/components/product/ShareProductButton.tsx:27` — `const language = useBootStore((s) => s.language);` then `language.code` (`ShareProductButton.tsx:34`).
- `src/lib/config/api.ts:46` — `if (language) config.headers.set('X-Lang', language.code);`

So the value the legal screens would read is **`useBootStore((s) => s.language)?.code`**, a bare lowercase code string like `'sr'` / `'cnr'` / `'en'` / `'ru'` — NOT a compound `rs-sr` locale. (The compound `{base-site}-{language}` form exists only in deep-link routing segments, e.g. `parseCompoundLocale` in `src/lib/navigation/deepLinkLocale.ts:28`; it is not the language slot.)

**Caveat for the implementer:** the slot is nullable (`LanguageDTO | null`). The two screens live under `(portal)/(public)/`, which only mounts after boot reaches `ready` (Gate 4 has resolved + registered the active language), so `language` should be non-null when these screens render — but the branch should still tolerate `null`/`undefined` defensively (the existing readers above use `?.code` or guard on `language`). A null/unknown code should fall to the `.en.md` default per the brief's "everything else → `.en.md`".

### Is `cnr` a real distinct value the language slot can hold?

**Yes — `cnr` is a real, distinct language code and is never collapsed to `sr` upstream.** The slot holds whatever `code` the backend emits in the base-site's `allowedLanguages` / `defaultLanguage`; nothing in the boot path rewrites `cnr` → `sr`. Evidence:

- `src/lib/navigation/deepLinkLocale.ts:43–44` (comment in `isLocaleValid`):
  > `rs-cnr` is a near-miss — `rs` exists and `cnr` is a real language, but `cnr` is `me`-only, so the pair is invalid.
- `src/lib/navigation/deepLinkLocale.ts:21` documents `me-cnr` as a real 3-letter language locale; `src/lib/navigation/redirectSystemPath.ts:9,31` likewise treat `me-cnr` as a valid compound segment.
- The language slot is populated purely from backend DTO fields — `storedSite.defaultLanguage` / `allowedLanguages.find((l) => l.code === …)` (`bootStore.ts:345–347`, `452–454`, `476`) and persisted/restored verbatim via `setStoredLanguage`/`getStoredLanguage`. No normalization step touches `code`.
- `rg -n "cnr" src/` over the whole `src/` tree shows `cnr` only ever referenced as a first-class distinct language; there is no `cnr → sr` mapping anywhere. The only `'sr'` literals in code are `fallbackLng: 'sr'` (`bootStore.ts:699`) and a hardcoded INTRO-namespace prefetch `fetchNamespace(TranslationNamespace.INTRO, 'sr')` (`BaseSiteSelector.tsx:65`) — neither collapses `cnr`.

**Implication for the brief's mapping:** the brief's grouping `sr`/`cnr` → `.sr.md` is correct and necessary precisely because `cnr` arrives as its own code — a naive `=== 'sr'` check would wrongly send Montenegrin (`cnr`) users to `.en.md`. The branch must test membership in `{'sr','cnr'}` (or equivalent), exactly as the brief states.

---

## 3 — Do the two `.en.md` URLs in code match `privacy-policy.en.md` / `terms-of-use.en.md` on `refs/heads/main`?

**Yes — exact match, character-for-character.** Both URLs use host `raw.githubusercontent.com`, repo path `memento-tech/oglasino-platform`, ref path `refs/heads/main`, and the expected filenames:

- Privacy (`privacy.tsx:12`):
  `https://raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/privacy-policy.en.md`
  → filename segment = `privacy-policy.en.md` on `refs/heads/main`. ✅
- Terms (`terms.tsx:12`):
  `https://raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/terms-of-use.en.md`
  → filename segment = `terms-of-use.en.md` on `refs/heads/main`. ✅

This is a confirmation of the **code strings** (verified by both Read and `rg -n "githubusercontent"`). I did NOT fetch the live GitHub repo to confirm the files physically exist (read-only audit; `memento-tech/oglasino-platform` is likely a private repo and a live fetch is out of scope). The implementer should confirm `privacy-policy.sr.md` and `terms-of-use.sr.md` actually exist on `refs/heads/main` before shipping the `.sr.md` branch — the new URLs would be the same strings with `.en.md` → `.sr.md`.

---

## 4 — Do these screens fetch at render, or is there a caching layer?

**Fetch at render, on mount, with NO caching layer.** `src/components/MarkdownViewer.tsx:23–37`:

```ts
  useEffect(() => {
    async function load() {
      try {
        const res = await fetch(url);
        if (!res.ok) throw new Error();
        setContent(await res.text());
      } catch {
        setError(true);
      } finally {
        setTimeout(() => setPreparing(false), 100);
      }
    }

    load();
  }, [url]);
```

- It is a plain `fetch(url)` inside a `useEffect` keyed on `[url]` — it runs every time the component mounts (i.e. every time the user opens Privacy/Terms) and re-runs if the `url` prop changes.
- **No caching of any kind:** `rg -n "AsyncStorage|cache|Cache"` over `MarkdownViewer.tsx`, `privacy.tsx`, and `terms.tsx` returns nothing. No AsyncStorage, no checksum/freshness gating (unlike the i18n namespace path in `bootStore`), no in-memory memo, no HTTP cache header handling. Whatever the platform/RN networking layer caches by default is the only caching; the app adds none.

**Implication for the `.sr.md` branch:** because the effect is keyed on `[url]`, building the URL from the active language is sufficient — switching language (which changes the resolved `url` prop) will re-trigger the fetch automatically. No cache invalidation work is needed.

---

## 5 — Trust boundary

None. The content is public, static markdown fetched anonymously from a GitHub raw URL with no auth header, no credentials, no user-supplied input in the URL, and no parsing of secrets — it is rendered read-only via `react-native-markdown-display`; the only language-derived choice is selecting between two fixed, hardcoded URL filenames, so there is no injection or privilege surface to guard.

---

## Verification log

- `rg -n "MarkdownViewer" app/ src/` → only the component def + the two screens.
- `rg -n "githubusercontent" app/ src/` → only `privacy.tsx:12` and `terms.tsx:12`.
- `rg -n "AsyncStorage|cache|Cache" src/components/MarkdownViewer.tsx app/(portal)/(public)/privacy.tsx app/(portal)/(public)/terms.tsx` → no output (no caching).
- `rg -n "cnr|'sr'|\"sr\"" src/` → `cnr` appears only as a distinct language in deep-link/locale code; `'sr'` only as `fallbackLng` and the INTRO prefetch.
- Read: `app/(portal)/(public)/privacy.tsx`, `app/(portal)/(public)/terms.tsx`, `src/components/MarkdownViewer.tsx`, `src/lib/store/bootStore.ts`, `src/lib/types/catalog/LanguageDTO.ts`, `src/lib/navigation/deepLinkLocale.ts`.

---

## Brief vs reality

No correctness conflicts. The brief's model matches the code on every checkable point:

- The two screens, their inline hardcoded `.en.md` URLs, and the `refs/heads/main` path are exactly as the brief describes.
- The active language is a bare code (`'sr'`/`'cnr'`/`'en'`/`'ru'`) read via `useBootStore((s) => s.language)?.code`, and `cnr` is genuinely distinct — so the brief's `sr`/`cnr` grouping is both correct and necessary.

Two non-blocking notes for the implementer (not challenges — implement the brief as written):

1. **Neither screen reads the language today.** Both `privacy.tsx` and `terms.tsx` are static and import nothing from `bootStore`. The `.sr.md` branch is new wiring in each screen (or a shared helper); there is no existing language-aware URL builder to extend.
2. **`language` is nullable.** Guard the branch so a null/unknown code falls to the `.en.md` default, consistent with the brief's "everything else → `.en.md`" and the existing `?.code`-guarded readers.
