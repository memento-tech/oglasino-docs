# Cloudflare DNS — oglasino.com zone

## Production records

| Record | Type | Target | Proxy | Notes |
|---|---|---|---|---|
| oglasino.com | A | 46.101.166.120 | Proxied | apex; routes through oglasino-router-prod Worker, forwards to Vercel |
| www.oglasino.com | CNAME | oglasino.com | Proxied | redirected 301 to apex by Worker |
| api.oglasino.com | A | 46.101.166.120 | Proxied | routes through oglasino-router-prod Worker, forwards to api-origin |
| api-origin.oglasino.com | A | 46.101.166.120 | DNS only | gray cloud — Worker fetches this directly to reach droplet |
| cdn.oglasino.com | (Worker custom domain) | oglasino-images-prod | Proxied | image CDN |

## Stage records

| Record | Type | Target | Proxy | Notes |
|---|---|---|---|---|
| api-stage.oglasino.com | A | 192.0.2.1 (placeholder) | Proxied | Worker route intercepts before reaching IP; routes through oglasino-router-stage Worker |
| api-origin-stage.oglasino.com | A | 142.93.106.90 | DNS only | gray cloud — Worker fetches this to reach stage droplet |
| cdn-stage.oglasino.com | (Worker custom domain) | oglasino-images-stage | Proxied | stage image CDN |
| stage.oglasino.com | _NOT YET CONFIGURED_ | _TBD_ | _TBD_ | deferred to Phase 1E (needs Vercel stage URL) |

## Worker custom domains vs. Worker routes

When you "Add custom domain" to a Worker in Cloudflare, it:
1. Creates a DNS record (CNAME for non-apex, A for apex) automatically
2. Binds the Worker to that hostname

When you "Add route" instead, you provide a pattern like `host.com/*`
and need to maintain the DNS record yourself.

We use Custom Domains where possible (cleaner). Routes only when the
hostname needs more flexibility than custom domains support.

## Notes

- The placeholder IP `192.0.2.1` for `api-stage.oglasino.com` is not
  arbitrary — it's a TEST-NET-1 IP from RFC 5737 reserved for
  documentation. Cloudflare requires SOME IP for an A record but the
  Worker route intercepts before traffic reaches it.
- All stage subdomains are proxied (orange cloud) except `api-origin-stage`
  which is gray cloud to allow Worker → droplet direct fetch without
  proxy recursion.
