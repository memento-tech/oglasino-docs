# Cloudflare Access Rules & Bot Protection

## Stage-specific protection

### HTTP header noindex

The `oglasino-router-stage` Worker adds the following header to all
responses (both API and frontend, when DNS exists):

```
X-Robots-Tag: noindex, nofollow, noarchive, nosnippet
```

This tells search engines not to index, follow, archive, or snippet
stage content. Implementation: see `oglasino-router/src/index.ts`,
the `forwardToOrigin` function, parameterized by `ENVIRONMENT === "stage"`.

### robots.txt

Deferred to Phase 3D — implemented in `oglasino-web` Next.js routing
based on `NEXT_PUBLIC_ENVIRONMENT === "stage"`. Will serve:

```
User-agent: *
Disallow: /
```

For stage frontend at stage.oglasino.com.

For stage API at api-stage.oglasino.com, robots.txt is unnecessary
(it's an API, not crawled). The X-Robots-Tag header above suffices.

## Future considerations (not configured today)

- Cloudflare Bot Fight Mode — could be enabled per-zone for additional
  protection. Not enabled today (free, but adds an extra header
  inspection layer that may rate-limit legitimate traffic).
- Cloudflare Access (Zero Trust) — could be added to lock stage
  behind email-based access. Not needed for v1; stage is publicly
  accessible by design (search engine indexing prevented as above).
