# Per-category Authored SEO Descriptions (Option C)

## Scope

Brief 5i seeded six per-base-site templates with `{categoryName}` and `{parentCategoryName}` placeholders. Option C in the feature discussion was per-category authored descriptions — a unique sentence per category in the catalog tree, not template-generated.

## Why deferred

Requires authoring hundreds of unique descriptions × 4 locales. High content-authoring cost. Template-based descriptions already shipped and are the "good enough" baseline.

## Prerequisites

- Content team commitment to author per-category descriptions
- Decision on which categories get authored descriptions first (high-traffic categories as priority)

## Estimated work split

- Backend seed work for N rows × 4 locales (where N is the number of categories with authored descriptions)
- Web brief to select authored vs template at render time (fallback to template when no authored description exists)
- Effort depends entirely on N

## References

- SEO foundation feature spec: [features/seo-foundation.md](../features/seo-foundation.md) §9.16
- SEO brief 5i session summary (template descriptions implementation)
