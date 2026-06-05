# Audit — Web Output-Encoding of Backend-Supplied Strings

**Repo:** oglasino-web · **Branch:** dev · **Mode:** READ-ONLY (no code changes)
**Feature:** backend-security-hardening (Phase-2 audit)
**Question settled:** does the web client encode backend-supplied strings at render, so the
backend can drop its input HTML-sanitization filter and rely on render-time output encoding?

**Tool-reliability note (state.md Risk Watch):** every `file:line` below was both `grep`-found
and `view`-read; the two agreed in every case. No disagreement surfaced.

---

## Part 1 — Raw-HTML injection surfaces

Repo-wide grep for `dangerouslySetInnerHTML` returns exactly three call sites; a fourth raw-HTML
surface exists via `rehype-raw` inside `MarkdownViewer` (no `dangerouslySetInnerHTML`, but it
renders raw HTML, so it is included here).

### 1a. `src/components/server/seo/JsonLd.tsx:14` — JSON-LD `<script>` — **(a) BACKEND/USER-SUPPLIED — THE SINK**

```tsx
<script
  key={i}
  type="application/ld+json"
  // eslint-disable-next-line react/no-danger
  dangerouslySetInnerHTML={{ __html: JSON.stringify(item) }}
/>
```

`JsonLd` is rendered on seven pages. Three feed it **raw backend/user content**:

- **Product page** (`app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:126`,
  data from `generateProductPageStructuredData`):
  `src/metadata/generateProductPageStructuredData.ts:61-62`
  ```ts
  name: product.name,
  description: product.description || product.name,
  ```
  and `:116` `name: product.name` again in the BreadcrumbList. `product.name` / `product.description`
  are free-text user-authored ad fields.
- **User page** (`app/[locale]/(portal)/(public)/user/[userId]/page.tsx:93`, from
  `generateUserPageStructuredData`): `src/metadata/generateUserPageStructuredData.ts:13,19,46`
  — `displayName = user.displayName` injected as `Person.name` and the breadcrumb name.
- **Catalog page** (`app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx:172`, from
  `generateCatalogPageStructuredData`): `src/metadata/generateCatalogPageStructuredData.ts:44`
  — `name: product.title` per item in the `ItemList`.

(The home/intro/about/pricing/free-zone JsonLd usages are fed only from translation keys —
trusted — but the three above carry raw user content.)

**Why this is a sink:** `JSON.stringify` escapes `"`, `\`, and control characters, but it does
**not** escape `<`, `>`, or `/`. When the resulting string is placed inside a `<script>` element
via `dangerouslySetInnerHTML`, the HTML parser still terminates the element at the first literal
`</script>` in the data. A product name or description of:

```
</script><script>alert(document.cookie)</script>
```

is serialized verbatim by `JSON.stringify`, breaks out of the JSON-LD `<script>` element, and
the trailing `<script>…</script>` executes as a real script — stored XSS, served on a public,
unauthenticated page. There is no `.replace(/</g, '\\u003c')` (or equivalent) escaping the
serialized JSON for the HTML script context.

**This is the one sink whose safety currently depends on the backend HTML-encoding `product.name`
/ `product.description` / `displayName` / `product.title` on input.** If the backend stops
encoding `<` → `&lt;` on these fields, this becomes directly exploitable. It must be fixed
web-side (escape the serialized JSON for the script context, or switch the structured data to a
non-`<script>`-injection delivery) before or together with the backend filter removal.

### 1b. `app/layout.tsx:68` — consent default `<script>` — **(b) trusted (cookie-derived, sanitized)**

```tsx
<script dangerouslySetInnerHTML={{ __html: consentDefaultSnippet }} />
```

`consentDefaultSnippet` (`app/layout.tsx:48-56`) interpolates only `consent.*` fields produced by
`sanitizeForSnippet(readConsentForSsr(cookieStore))`. Cookie-derived, not backend-supplied, and
passed through an explicit sanitizer. Not the backend's concern.

### 1c. `app/layout.tsx:69` — theme `<script>` — **(b) trusted (cookie-derived, allowlisted)**

```tsx
<script dangerouslySetInnerHTML={{ __html: themeSnippet }} />
```

`generateThemeSnippet` (`src/lib/theme/ssr.ts:10-15`) interpolates `theme` only after
`sanitizeTheme` allowlists it to `'light' | 'dark' | 'system'` (else `undefined`). Cookie-derived,
allowlisted, not backend-supplied. Not the backend's concern.

### 1d. `src/components/server/MarkdownViewer.tsx:27` — `rehype-raw`, **no sanitizer** — **(b) trusted source, but fragile; INDEPENDENT of the backend question**

```tsx
<ReactMarkdown remarkPlugins={[remarkGfm]} rehypePlugins={[rehypeRaw]}>
  {content}
</ReactMarkdown>
```

`rehype-raw` re-parses raw HTML embedded in the markdown and renders it as real DOM. There is
**no** `rehype-sanitize` in the chain (`package.json` carries `rehype-raw` but no
`rehype-sanitize`/`dompurify`). So whatever HTML the markdown source contains is rendered live.

**But the source is operator-controlled and never touches the backend.** Both callers fetch fixed
GitHub raw URLs in the platform's own repo:

- `app/[locale]/(portal)/(public)/privacy/page.tsx:23-25` →
  `…/memento-tech/oglasino-platform/refs/heads/main/privacy-policy.en.md`
- `app/[locale]/(portal)/(public)/terms/page.tsx:23-25` →
  `…/memento-tech/oglasino-platform/refs/heads/main/terms-of-use.en.md`

No request data, DB content, or backend-emitted string flows here. Removing the backend input
sanitization has **zero** effect on this surface. It is flagged only as a standing fragility
(unsanitized `rehype-raw` would become an XSS sink the day the markdown source ever switches to
user/DB-authored content) — out of scope for the backend question, noted in passing per the brief.

### 1e. Auto-linkify (messaging) — **(a) backend/user content, but SAFE — escapes the body**

`src/messages/components/Message.tsx:24-41` renders the message body. The body is tokenized by
`tokenizeMessageText` (`src/messages/utils/linkify.ts:19-40`) and rendered as:

```tsx
token.type === 'link' ? (
  <a key={tokenIndex} href={token.href} target="_blank" rel="noopener noreferrer" …>
    {token.value}
  </a>
) : (
  <span key={tokenIndex}>{token.value}</span>
)
```

Every text run goes through JSX `{token.value}` text interpolation (escaped). The only constructed
markup is the `<a>` wrapper; its `href` is `m.url` from `linkify-it`, whose schema is narrowed in
`linkify.ts:10-13` (`mailto:`, `ftp:`, `//` removed; `fuzzyEmail` off; `javascript:`/`data:`/
`vbscript:` are not in the default schema and were not added). There is **no** raw-HTML
interpolation of the message body — it cannot carry arbitrary markup. Confirms the decisions.md
2026-05-20 "schemes restricted by construction" note. **Safe.**

---

## Part 2 — Default rendering of backend strings (the baseline)

React JSX `{value}` text interpolation HTML-escapes by default. Spot-checks of backend/user
content on the normal (non-`<script>`) render path:

- **Product name** — `src/components/server/ProductDetails.tsx:68`:
  `<h1 className="mb-2 text-xl">{productDetails.name}</h1>` — JSX text, escaped.
- **Product description** — `src/components/server/ProductDetails.tsx:91-95` passes
  `text={productDetails.description}` to `InlineCollapsibleText`, which renders
  `<p>{text}</p>` (`src/components/client/CollapsibleText.tsx:44`) — JSX text, escaped. No
  `dangerouslySetInnerHTML` in that component.
- **User displayName** — `src/components/owner/follows/UserCard.tsx:51`:
  `<p>{userInfo.displayName}</p>`; also `src/components/owner/client/NavUser.tsx:23`:
  `<span …>{user?.displayName}</span>` — JSX text, escaped.
- **Message body** — covered in 1e: escaped text + constructed-only `<a>`.

Baseline holds: the default render path escapes backend strings. "Rely on output encoding" is true
*for the normal DOM-text path*. It is the `<script>`-context path (JSON-LD) where the default JSX
escaping does not apply, because the bytes are injected via `dangerouslySetInnerHTML`, not as a
text node.

---

## Part 3 — Verdict

**Not safe as-is.** The normal render path (JSX text) escapes by default and the messaging
linkifier escapes the body, so for ordinary on-page text the "trust the client to encode"
assumption holds. But there is exactly one sink that injects backend/user content as **raw
`<script>` content**: `JsonLd` (`src/components/server/seo/JsonLd.tsx:14`) serializes structured
data with `JSON.stringify` into a `<script type="application/ld+json">`, and `JSON.stringify` does
not escape `</script>`/`<`/`>`. On the product, user, and catalog pages that script carries raw
`product.name` / `product.description` / `user.displayName` / `product.title`. The safety of these
pages against XSS **currently depends on the backend HTML-encoding `<` in those fields on input** —
i.e. it depends on exactly the filter the hardening feature wants to remove. Remove the filter
without fixing this sink, and a product name/description containing `</script><script>…` becomes
stored XSS on a public page. The fix is small and web-side (escape the serialized JSON for the
script context — e.g. replace `<` with `<` before injection — or stop delivering structured
data via a raw `<script>`); it must land before or together with the backend removal. The
`MarkdownViewer` `rehype-raw`/no-sanitize surface is fragile but **independent** of the backend
question (operator-controlled GitHub source, never touches the backend) and does not block the
removal.

---

**Bottom line:** `NOT SAFE — see sink JsonLd (`src/components/server/seo/JsonLd.tsx:14`): backend
product name/description, user displayName, and catalog product titles are injected into a
`<script type="application/ld+json">` via `JSON.stringify`, which does not escape `</script>`. Fix
this web-side before/with removing the backend input HTML-encoding.`
