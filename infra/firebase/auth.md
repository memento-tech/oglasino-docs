# Firebase Authentication

## Configured providers (both projects)

| Provider | Status | Notes |
|---|---|---|
| Email/Password | enabled | passwordless (email link) NOT enabled |
| Google | enabled | OAuth client auto-created in underlying GCP project |

## OAuth consent screens

Each Google sign-in provider creates an OAuth 2.0 client in the
underlying Google Cloud project. Default consent screen settings:
- Project public-facing name: oglasino-stage / oglasino-prod
- Project support email: oglasino@gmail.com

To customize the OAuth consent screen (logo, app domain, privacy
policy URL) — Firebase Console → Authentication → Sign-in method →
Google → click the provider → "View in Google Cloud Console". Defer
until pre-launch polish.

## Authorized domains

See `firebase/projects.md` for per-project domain lists.

When adding new domains:
- Firebase Console → Authentication → Settings tab → Authorized domains

## Future providers

Not configured today, may add later:
- Apple Sign-In (required by App Store if Google sign-in is offered —
  Phase 1D / 3E followup)
- Anonymous auth (if needed for browse-without-login flows)
