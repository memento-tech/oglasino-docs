# Intro Page Cluster Splitting Evaluation

## Scope

Intro page (`/`) currently emits an all-locales hreflang cluster (7 entries post-BCP-47-collision, with rsmoto winning the rs slots). Brief 2b kept this as the one exception to per-base-site clustering. Post-launch SEO evidence may motivate splitting intro into three independent clusters too.

## Why deferred

Intro is the locale-selector landing page; the cluster's semantic purpose differs from content pages. Splitting requires re-thinking how x-default behaves on a non-content page.

## Prerequisites

- Search Console data showing whether the current shape produces ranking issues for Serbian-language general-goods searchers (rs.oglasino vs rsmoto.oglasino)
- Sufficient post-launch traffic data to evaluate

## Estimated work split

- 1 audit session + 1 web brief if changes warranted
- Only proceed if Search Console data shows a problem

## References

- SEO foundation feature spec: [features/seo-foundation.md](../features/seo-foundation.md) §6.4
- SEO brief 2b session summary (BCP-47 collision analysis)
- SEO brief 5h session summary (per-cluster x-default)
