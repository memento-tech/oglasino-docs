# Active Submission to Google (Sitemap Auto-ping, IndexNow, Search Console API)

## Scope

Today the site relies on Google's natural crawl to discover new products. Post-launch, active submission via sitemap auto-ping on product creation, IndexNow for Bing, and Search Console API for monitoring would accelerate indexation.

## Why deferred

Separate concern from foundation correctness. Needs its own feature intake with backend integration — a product-creation hook that triggers submission is cross-cutting and touches the product lifecycle.

## Prerequisites

- Google Search Console verified property (operator task)
- IndexNow API key (operator task)
- Backend product-creation hook to trigger submissions (engineering)
- Decision on which events trigger submission (create only? update? price change?)

## Estimated work split

- Full-feature intake — not a single-brief item
- Backend: product lifecycle hook + HTTP client to ping sitemap / IndexNow endpoint
- Web: Search Console API integration for monitoring (optional, could be operator-only via the Console UI)
- Router: possibly exempt IndexNow verification file from rewrites

## References

- SEO foundation feature spec: [features/seo-foundation.md](../features/seo-foundation.md) §11 post-launch backlog
- Google Search Console documentation
- IndexNow protocol specification
