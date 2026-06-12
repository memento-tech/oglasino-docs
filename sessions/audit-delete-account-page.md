# Audit — Mobile content-page pattern (for a "how to delete your account" page)

**Repo:** oglasino-expo
**Branch:** dev
**Type:** Phase-2 audit, read-only
**Scope:** Brief Part 1 — does a blog/content page exist on mobile, and how would a
new one (the "how to delete your account" article) be added.

> Note: this audits the **informational content page** the brief wants to add. The
> actual destructive *delete-account flow* already exists on mobile — `owner/user.tsx`
> "Danger Zone" (lines ~353–373) opens `DELETE_ACCOUNT_CONFIRMATION_DIALOG`. The new
> page is a how-to article, not the delete action itself.

---

## 1. Does a blog / long-form content page exist on mobile? — YES

The free-zone blog post exists and is the closest precedent to a "blog article" page.

- **Route:** `/blog/free-zone`
- **File:** `app/(portal)/(public)/blog/free-zone.tsx` (component `WhatIsFreeZone`)

Routing is **file-system based** (expo-router). The file's location under
`app/(portal)/(public)/` *is* the route — no screen registration anywhere. The public
group's layout is just `<Stack screenOptions={{ headerShown: false }} />`
(`app/(portal)/(public)/_layout.tsx`), so every content page renders chrome-less and
supplies its own back affordance.

Deep links resolve for free: `redirectSystemPath` (`src/lib/navigation/redirectSystemPath.ts`)
strips a leading locale segment by **shape**, not an allowlist, so a new
`/blog/<anything>` route is reachable from a web canonical link with **no edit** to the
deep-link layer. (Confirmed by `redirectSystemPath.test.ts:37` for `/blog/free-zone`.)

Other content pages, all under `app/(portal)/(public)/`:

| Page | Route | File | Content source |
| --- | --- | --- | --- |
| Free zone (blog) | `/blog/free-zone` | `blog/free-zone.tsx` | **native components + translations** |
| About | `/about` | `about.tsx` | **native components + translations + bundled images** |
| Pricing | `/pricing` | `pricing.tsx` | native components + translations |
| Contact | `/contact` | `contact.tsx` | native components + translations |
| Privacy | `/privacy` | `privacy.tsx` | **remote markdown** via `MarkdownViewer` |
| Terms | `/terms` | `terms.tsx` | **remote markdown** via `MarkdownViewer` |

So there are **two distinct content-page patterns**. Pick one for the delete-account page.

---

## 2. Content source — the two patterns

### Pattern A — native components + backend translation keys (free-zone / about / contact)

Body is authored **in the component** as JSX, with every user-visible string pulled from
a backend-seeded translation namespace. Example (`blog/free-zone.tsx`):

```tsx
const tFreeZone = useTranslations(TranslationNamespace.FREE_ZONE_PAGE);
...
<Text className="text-3xl font-semibold text-logo">{tFreeZone('header')}</Text>
<Text className="text-base">{tFreeZone('subtitle')}</Text>
```

Repeated sections are driven by arrays + `.map`, indexing into keyed translations
(`tFreeZone('how.one')`, `tFreeZone(\`importance.values.${i}.title\`)`).

- **Best when** the page has structure (cards, steps, CTAs, icons, buttons) and needs
  interactive elements (the free-zone page has two "create product" CTAs gated through
  `openProductCreateGate`).
- **Localization is automatic** via the translation layer (see §4).

### Pattern B — remote markdown via `MarkdownViewer` (privacy / terms)

Body is **fetched at runtime** from an external repo and rendered as markdown. The whole
screen is ~12 lines (`privacy.tsx`):

```tsx
const lang = useBootStore((s) => s.language)?.code;
return (
  <ScrollView className="flex-1">
    <BackToHomeButton />
    <MarkdownViewer url={legalDocUrl('privacy', lang)} />
    <Footer />
  </ScrollView>
);
```

- `MarkdownViewer` (`src/components/MarkdownViewer.tsx`): `fetch(url)` → `res.text()` →
  renders with `react-native-markdown-display`'s `<Markdown>`. Shows a `Skeleton`
  placeholder while loading, an error line (`tErrors('markdown.fild.load')`) on failure.
  Markdown styling (headings, links, code, blockquote, list) is a local `markdownStyles`
  object; body text color flips on `colorScheme`.
- `legalDocUrl` (`src/lib/utils/legalDocUrl.ts`) builds a **raw GitHub** URL:
  `https://raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/<stem>.<token>.md`.
  `STEMS = { privacy: 'privacy-policy', terms: 'terms-of-use' }`.
- **Best when** the copy is long-form prose, lawyer-/editor-authored, and wants to live
  outside the app bundle so it can change without an app release.

---

## 3. Images

### Pattern A (native)

Local **bundled** assets via `require(...)`, rendered with `expo-image`'s `Image`:

```tsx
import { Image } from 'expo-image';
const missionImages = [ require('../../../assets/images/ourMission1.png'), ... ];
<Image source={missionImages[i]} contentFit="contain" style={{ width: 200, height: 200 }} />
```

- **Where assets live:** `assets/images/` (e.g. `ourMission1.png`, `oglasino-hero2.jpg`),
  `assets/avatar/` (avatars). Path is **relative from the screen file** —
  `../../../assets/...` from a page under `app/(portal)/(public)/`. From
  `app/(portal)/(public)/blog/` it would be `../../../../assets/...` (one level deeper).
- **Sizing:** explicit `style={{ width, height }}` or nativewind classes
  (`className="h-20 w-20 rounded-full"`), plus `contentFit` (`cover`/`contain`).
- Free-zone itself uses **no raster images** — it uses SVG icon components from
  `@/components/icons` (e.g. `ReuseIcon`, `GivingBackBigIcon`), often wrapped in
  `ThemeSupportiveIcon` for theme-aware tinting, sized via a `size` prop.

### Pattern B (markdown)

Images would be **remote URLs embedded in the markdown** (`![alt](https://…)`), rendered
by `react-native-markdown-display`'s image rule. No bundled assets; nothing to size in the
app — the markdown controls it. (The current legal docs contain no images.)

---

## 4. Translation / localization (SR / CNR / EN / RU)

### Pattern A

`useTranslations(TranslationNamespace.X)` returns a `t(key)` reading from backend-seeded
keys for the **active language**, fetched at boot. Switching language re-renders with new
strings; no per-locale files in the repo. Namespaces are a **fixed enum**
(`src/i18n/types.ts`) mirroring the backend — agents **do not invent** namespaces
(conventions Part 6). Existing page namespaces: `ABOUT_PAGE`, `FREE_ZONE_PAGE`,
`PRICING_PAGE`, `DASHBOARD_PAGES`, `MESSAGES_PAGE`.

> **Decision the build brief must resolve:** there is **no** `BLOG_PAGE` / `HELP_PAGE`
> namespace. A delete-account *guide* page has nowhere obvious to put its keys. Options:
> (a) reuse `DASHBOARD_PAGES` (the existing `delete.account.*` keys already live there —
> see `owner/user.tsx`); or (b) backend adds a new namespace (not a mobile change; needs
> Igor + backend, and an enum addition here). Recommend (a) unless web wants a dedicated
> namespace.

### Pattern B

Localized by **file selection**, not the translation layer. `legalDocUrl(doc, lang)`:
`lang === 'en' || lang === 'ru'` → English file; **everything else** (`sr`, `cnr`/`me`,
undefined, unknown) → Serbian file. So SR is the default and CNR aliases to SR. The doc
must be authored as `<stem>.en.md` + `<stem>.sr.md` in the external
`memento-tech/oglasino-platform` repo (RU readers get the English file — there is no RU
markdown file). `MarkdownViewer` is keyed on its `url` prop, so a language switch
re-fetches automatically.

> For a delete-account markdown page you would: author `<stem>.en.md` / `<stem>.sr.md` in
> the external platform repo **and** add the stem to `STEMS` in `legalDocUrl.ts` (or write
> a sibling URL helper). That external-repo authoring is **outside this repo**.

---

## 5. Styling / layout — what a content page inherits vs styles itself

A content page is **not** wrapped by any shared screen layout beyond the chrome-less
`<Stack>`. Each page assembles the same envelope by hand:

```tsx
<ScrollView className="flex-1">        {/* or w-full */}
  <BackToHomeButton />                 {/* top-right "← back home", routes to '/' */}
  ...content in a centered View...     {/* className="mx-auto w-[90%]" or max-w-2xl */}
  <Footer />                           {/* shared site footer */}
</ScrollView>
```

Inherited / shared building blocks:
- **`BackToHomeButton`** (`src/components/BackToHomeButton.tsx`) — the only "header"; pushes `/`.
- **`Footer`** (`src/components/navigation/Footer.tsx`) — shared footer with category +
  company + help link columns (see §6).
- **`Text`** (`@/components/basic/text`) — theme-aware text primitive; **use this, not RN `Text`**.
- **`ScrollToTop`** (`about.tsx` uses it with a `scrollRef`) — optional.
- Typography/spacing are **nativewind classes**, picked per-page (no shared typography
  scale component): `text-3xl font-semibold`, `text-base`, `gap-4`, `bg-hero`, `text-logo`,
  `shadow-card`, `max-w-2xl`, etc.
- `headerShown:false` is inherited from the layout, so the page owns its top spacing
  (free-zone uses `pt-14`).

The page **styles itself** for everything content-specific (cards, CTAs, hero blocks,
icon sizing). There is no opinionated content-page template component to extend.

---

## 6. Navigation / linking — where a new page is reached from

The **Footer** is the link hub, rendered on essentially every screen (it's the
`ListFooterComponent` of every product list and sits at the bottom of every content page).
It renders two link arrays:

- `companyNavigations` (`src/lib/navigation/companyNavigations.tsx`) — about, pricing,
  privacy, terms, contact.
- `helpNavigations` (`src/lib/navigation/helpNavigations.tsx`) — currently just
  `{ labelKey: 'blog.free.zone.label', route: '/blog/free-zone' }`.

Both are `{ labelKey, route }` arrays; Footer maps them to `<Pressable onPress={() =>
router.push(route)}>` and resolves the label with **`tPaging` = `PAGING` namespace**. So a
new footer link needs a `PAGING` label key (e.g. `delete.account.guide.label`) — that's a
**backend translation add**, not a repo string.

Other plausible entry points:
- **`owner/user.tsx` "Danger Zone"** — natural place to link the how-to next to the actual
  delete button (`router.push('/blog/delete-account')` or similar).
- Anywhere the existing dialog flow lives, as a "learn more" link.

---

## Recipe — how to add the delete-account content page (for the build brief)

Assuming **Pattern A** (native + translations), placed under `/blog/` to match free-zone:

1. **Create** `app/(portal)/(public)/blog/delete-account.tsx` (route auto-becomes
   `/blog/delete-account`; deep links work with no other change). Envelope:
   `<ScrollView>` → `<BackToHomeButton />` → centered content `View` → `<Footer />`.
2. **Strings:** `useTranslations(...)` against a chosen namespace (recommend
   `DASHBOARD_PAGES`, reusing the existing `delete.account.*` family — **needs a namespace
   decision; see §4**). All copy is backend-seeded keys, no in-repo strings. Localization
   is then automatic across SR/CNR/EN/RU.
3. **Images (if any):** bundle in `assets/images/`, import `Image` from `expo-image`,
   `require('../../../../assets/images/<name>.png')` (note the **four** `../` from
   `app/(portal)/(public)/blog/`), size with explicit `style` or classes. Or use existing
   SVG icon components from `@/components/icons`. (If long-form prose is preferred instead,
   use **Pattern B**: author `<stem>.en.md`/`<stem>.sr.md` in the external platform repo
   and extend `STEMS` in `legalDocUrl.ts`.)
4. **Typography/layout:** nativewind classes matching free-zone/about (`Text` primitive,
   `max-w-2xl`/`w-[90%]`, section `gap`/card patterns). No template to extend.
5. **Link it:** add `{ labelKey: 'delete.account.guide.label', route: '/blog/delete-account' }`
   to `helpNavigations.tsx` (label key seeded in **`PAGING`** by backend). Optionally also
   link from `owner/user.tsx` Danger Zone.

**Cross-repo / non-mobile prerequisites the build brief must line up:**
- New translation keys seeded by backend (page body keys + the `PAGING` footer-label key),
  in whichever namespace is chosen.
- If Pattern B: the `.en.md` / `.sr.md` files authored in `memento-tech/oglasino-platform`.
- A namespace decision (reuse `DASHBOARD_PAGES` vs new namespace).
