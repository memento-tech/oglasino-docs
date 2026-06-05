# Audit — oglasino-backend — Deep Linking (Universal/App Links)

**Repo:** oglasino-backend
**Branch:** `dev` (unchanged — read-only)
**Type:** READ-ONLY audit. No code changes, no config edits, no git ops, no deploys. `psql` not needed.
**Date:** 2026-06-04

Backend's only plausible involvement was (a) whether any `/.well-known/*` or static-asset
path routes here, and (b) the Android release-cert SHA-256 fingerprint. Read the actual source;
cross-checked every surprising hit with `grep`. Findings below.

---

## Q1 — Does any backend route handle `/.well-known/*`?

**No — and if such a request ever reached the backend, it would be `403`, not served.**

- `SecurityConfig.java:66-100` is the full authorize-requests block. Its explicit matchers are:
  `OPTIONS /**` (permit), `/actuator/health/**` (permit), `/error` (permit), `/health` (permit),
  `/actuator/prometheus` + `/actuator/info` (deny), `/api/public/**` + `/api/auth/**` + `/internal/**`
  (permit), `/api/secure/**` (authenticated), then `anyRequest().authenticated()` (`:99-100`).
- `/.well-known/**` is **not** in any matcher. It therefore falls through to the H1 default-deny
  `anyRequest().authenticated()` line. An unauthenticated `/.well-known/apple-app-site-association`
  or `/.well-known/assetlinks.json` request would be rejected **403** (no token → `FirebaseAuthFilter`
  leaves the context unauthenticated → default-deny).
- **No controller maps `/.well-known`.** Grep for `well-known` / `app-site-association` / `assetlinks`
  across `src/main` returns zero mapping hits (the only `well-known` string anywhere is this audit's
  own filename). There is no root or catch-all mapping either: grep for
  `@RequestMapping("/" | "/**" | "")` in `src/main` returns nothing.

**Conclusion:** the backend neither serves nor permits `/.well-known/*`. It is the wrong origin for
the association files, and the security posture already (correctly) denies them by default.

## Q2 — Does backend serve any static assets at all?

**No.**

- No `WebMvcConfigurer` implementer in `src/main` (grep: zero hits) → no `addResourceHandler` /
  `ResourceHandlerRegistry` customization.
- No `src/main/resources/static/` or `src/main/resources/public/` directory exists with any files.
  The only resource directories are seed/data dirs: `db/migration`, `data/*`, `catalogJSON`,
  `dataJSON`, `es-config` — none of which Spring serves over HTTP.
- No `spring.web.resources.*` / `spring.mvc.static-path-pattern` config in any of
  `application-dev.yaml` / `application-stage.yaml` / `application-prod.yaml`. (The `web:` keys present
  in those files are app-custom config — `web.base-url` for email links, `web.revalidate.*` for the
  Next.js revalidate webhook — not Spring static-resource config.)

**Conclusion:** there is no mechanism by which the backend could serve a flat, extensionless JSON file.
Not relevant to this feature.

## Q3 — Release-cert fingerprint

**Backend has no role in producing the Android signing-cert SHA-256 fingerprint** — it comes from EAS
credentials (per the brief, and confirmed by state.md's Expo cloud-setup row: fingerprints come from
the development-tier keystore and are registered in Firebase, not produced by backend).

The only `keystore` / `signing-cert` / `fingerprint` hits in backend are **none** — those literal terms
return zero results. All `SHA-256` hits are unrelated: email/userId audit hashing
(`DefaultUserAuditService`), the per-chat push collapse-key
(`DefaultMessageNotificationService`), and catalog version checksums (`VersionChecksumService`).
No cert/fingerprint handling exists.

## Q4 — Any existing deep-link / app-association awareness

**No app-association awareness.** Grep for `well-known`, `app-site-association`, `assetlinks`,
`applinks`, `universal link`, `app link` → zero hits in `src/main` (none anywhere except this file).

The term "deep-link" **does** appear, but exclusively in the **in-app push-notification navigation**
sense — the relative app route a notification tap opens — not Universal/App Links:

- `DefaultMessageNotificationService.java:39` — "Deep-link target for a MESSAGE push tap"
  (a locale-unprefixed in-app path).
- `DefaultAdminReportFacade.java:113` — INFO-card "deep-link" comment (no navigate path).
- Test javadoc in `DefaultAdminReviewServiceTest` / `DefaultUserFacadeTest` — push-navigation paths
  (`/owner/products`, profile route) after the owner-route flatten.

These are the notification feature's client-navigation targets (relative paths like
`/owner/products?productId=`), which the OS-level Universal/App Link layer is orthogonal to. None of
them produce, serve, or reference an app-association file.

---

## Verdict

**Backend is NOT in scope for the Deep Linking feature.** No `/.well-known/*` routing (it would 403
under default-deny), no static-asset serving, no Android cert-fingerprint handling, no app-association
awareness. The association files belong at the edge (Cloudflare worker) or web; the Android SHA-256
fingerprint belongs to EAS credentials. There is no single backend touchpoint. Result: **N/A**, as the
brief expected.

## For Mastermind

Nothing flagged. One worth-noting (not a defect): the H1 default-deny already makes the backend
*safe by construction* for this feature — were the association files ever (mis)pointed at the backend
origin, the request would 403 rather than leak or mis-serve. So the audit's "N/A" is the strong kind:
not "backend happens not to handle it," but "backend would actively refuse it." No action needed.

## Conventions check

- Part 4 (cleanliness): N/A — read-only audit, no code touched.
- Part 11 (trust boundaries): confirmed in passing — the default-deny posture (`SecurityConfig.java:99`)
  is what makes the `/.well-known/*` answer a 403 rather than a silent public serve.
