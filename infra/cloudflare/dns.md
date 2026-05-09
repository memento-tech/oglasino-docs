# Cloudflare DNS

| Record | Type | Target | Proxied? | Notes |
|---|---|---|---|---|
| oglasino.com | A/CNAME | Vercel | yes | prod web |
| www.oglasino.com | CNAME | oglasino.com | yes | prod web |
| api.oglasino.com | A | <prod droplet IP> | yes | prod backend |
| cdn.oglasino.com | CNAME | <Worker route> | yes | prod images |
| stage.oglasino.com | A/CNAME | Vercel | yes | stage web (Phase 1C.2) |
| api-stage.oglasino.com | A | <stage droplet IP> | yes | stage backend (Phase 1C.3) |
| cdn-stage.oglasino.com | CNAME | <Worker route> | yes | stage images |
| api-origin... | TBD | TBD | TBD | Igor to explain api-origin pattern |

Nameservers are delegated from Namecheap to Cloudflare — see
[`../namecheap/domains.md`](../namecheap/domains.md).
