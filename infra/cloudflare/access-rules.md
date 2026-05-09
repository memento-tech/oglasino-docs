# Cloudflare Access & Bot Rules

## Bot / SEO protection on stage subdomains

Stage must not be indexed by search engines or scraped by bots. The
`oglasino-router-stage` Worker injects:

- `X-Robots-Tag: noindex, nofollow` on every response
- A `/robots.txt` returning `User-agent: *\nDisallow: /`

This is implemented during Phase 1C.5 of
[`../master-plan.md`](../master-plan.md).

## Cloudflare Access policies

None configured yet. If/when added (e.g., to gate stage behind a
team-only login), document the policy name, the application it
protects, and the identity provider here.
