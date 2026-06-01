# Per-locale Privacy and Terms Markdown Content

## Scope

Privacy and terms pages serve English markdown for all locales. Audit Part 7.9 flagged this. All locales (SR, EN, RU, CNR) render the same English content fetched from GitHub.

## Why deferred

Content-authoring work (legal translations) needed before code can ship. Tracked since feature start as out of scope for the SEO foundation.

## Prerequisites

- Localized markdown files authored for RS, RU, CNR
- Web wiring to select the right file per locale (parameterize the GitHub raw URL or host locale-specific files)

## Estimated work split

- Depends on translation timeline (external to engineering)
- Web work is ~2 hours once content exists (parameterize fetch URL by locale, add fallback logic)

## References

- SEO foundation feature spec: [features/seo-foundation.md](../features/seo-foundation.md)
- `issues.md` 2026-05-14 entry "Privacy and Terms render English markdown across all locales"
- Audit Part 7.9
