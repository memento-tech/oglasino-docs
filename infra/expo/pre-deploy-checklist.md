# Expo Pre-Deploy Checklist

Things that must be done before each tier of the mobile app can be shipped to its intended audience. Reference for whoever is preparing a deploy, separate from the state spec at [cloud-setup.md](cloud-setup.md).

This is a forward-looking checklist, not a record of completed work. Items are grouped by which deploy gate they unblock. Mark complete by amending this file when each item lands.

## Before any preview build can be installed by testers

The preview tier is internally-distributed (Ad Hoc on iOS, raw APK on Android). Testers need to be able to install the build on their devices and have the app authenticate against the stage Firebase project.

### iOS-specific
- [ ] Register each tester's iPhone UDID via `eas device:create`. iOS Ad Hoc distribution only allows installation on devices included in the provisioning profile.
- [ ] Verify the preview iOS build's provisioning profile includes all registered tester UDIDs. Rebuild if a tester was added after the most recent build.

### Android-specific
- [ ] First preview Android build will generate a separate keystore from the development keystore. Extract the SHA-1 and SHA-256 fingerprints from the new keystore (via `eas credentials → Download existing keystore → keytool -list -v`).
- [ ] Register both fingerprints against the `com.oglasino.preview` Firebase App in the `oglasino-stage-49abb` project (Firebase Console → Project Settings → Your apps → Add fingerprint).
- [ ] Re-download `google-services.json` for `com.oglasino.preview` after fingerprint registration and replace the existing `google-services.preview.json` in the repo.
- [ ] Trigger a fresh preview Android build to bake the updated google-services file into the APK.
- [ ] Back up the preview keystore + password to `oglasino-private/01-secrets/oglasino-infra/oglasino-expo-android-preview-keystore.jks`.

### Both platforms
- [ ] Confirm `.env.preview` is fully populated. No empty values for any required key. Verify with `grep -c "=$" .env.preview` returns 0.
- [ ] Confirm the stage backend at `https://api-stage.oglasino.com/api` is reachable and authenticated requests work end-to-end.

## Before any production build can be installed by anyone

The production tier ships to App Store and Play Store. Higher bar than preview.

### iOS-specific

- [ ] **Apple Developer Program enrollment is active** (renewed annually, $99/year). Confirm at https://developer.apple.com/account.

- [ ] **Apple Distribution Certificate** exists. EAS generates and manages this automatically on first production build.

- [ ] **App Store Connect listing exists** for `com.oglasino`. Create at https://appstoreconnect.apple.com → My Apps → "+" button. Required for TestFlight or App Store submission.

- [ ] **Production app icon, screenshots, description, keywords** prepared in App Store Connect listing. Apple rejects submissions missing any of these.

- [ ] **APNs Key** (Apple Push Notification service) generated and uploaded to Firebase Cloud Messaging in `oglasino-prod-7e5db` project. Required for push notifications on iOS in production.
  - Apple Developer → Certificates, Identifiers & Profiles → Keys → "+" → enable Apple Push Notifications service (APNs)
  - Download the `.p8` file (one-time download)
  - Note the Key ID and Team ID
  - Firebase Console → `oglasino-prod-7e5db` → Project Settings → Cloud Messaging → Apple app configuration → Upload APN auth key
  - Back up the `.p8` file to `oglasino-private` immediately — Apple does not allow re-download

- [ ] **TestFlight setup** if using TestFlight for beta distribution: create internal testing group, add testers by email, configure beta app information.

### Android-specific

- [ ] **Google Play Console** has a project for `com.oglasino`. Create at https://play.google.com/console.

- [ ] **Production app icon, screenshots, store listing, content rating, target audience, privacy policy URL** all completed in Google Play Console. Play rejects submissions missing any of these.

- [ ] **First production Android build generates a separate keystore from development and preview keystores.** Extract SHA-1 and SHA-256 fingerprints.

- [ ] **Register both fingerprints against the `com.oglasino` Firebase App in the `oglasino-prod-7e5db` project.**

- [ ] **Re-download `google-services.json` for prod after fingerprint registration**, replace `google-services.prod.json` in the repo, rebuild production.

- [ ] **Back up the production keystore + password** to `oglasino-private`. Treat this keystore as critical — losing it makes Play Store updates impossible.

- [ ] **Play App Signing** enrollment. Google Play manages the production signing key on Google's side. Opt in during first production upload. After enrollment, the keystore used to sign the AAB becomes the "upload key" and the actual app signing key lives at Google.

- [ ] **FCM Service Account credentials** uploaded for server-side push send. Firebase Console → Project Settings → Service accounts → Generate new private key. Used by backend to send pushes via FCM HTTP v1 API.

### Both platforms

- [ ] **`.env.production` fully populated** with prod Firebase + prod backend + prod CDN + prod GA4 values. Verify with `grep -c "=$" .env.production` returns 0.

- [ ] **Production backend at `https://oglasino.com/api`** confirmed live and ready to serve production traffic.

- [ ] **Privacy Policy and Terms of Use** finalized (lawyer review complete) and the URLs added to App Store Connect and Google Play Console listings.

- [ ] **Production GA4 property** (`G-GNKB4WBNC0`) has all custom dimensions, conversion events, and internal-traffic filters configured. Verify against the stage property's runbook.

- [ ] **Search Console verification** of `oglasino.com` and link to production GA4 property. Recommended timing: 1-2 days before production deploy.

- [ ] **Production Firebase rules** deployed and tested against the `oglasino-prod-7e5db` Firestore database. The `oglasino-firestore-rules` repo handles this.

## Before EAS Workflows actually fire

The workflow YAMLs in `.eas/workflows/preview.yml` and `production.yml` exist but have never been triggered. The first push to `preview` or `main` branch fires them.

- [ ] **Test preview workflow:** push a small change to `preview` branch. Confirm both Android and iOS builds trigger automatically in EAS dashboard. Confirm the path filter works (a doc-only push should NOT trigger; a code push should).
- [ ] **Test production workflow:** similar test on `main` branch, with extra caution since this triggers production builds. Consider an empty commit or a non-functional change for the first trigger.
- [ ] **Handle workflow failures:** if a workflow run fails, debug from the EAS dashboard logs. Common failure modes: missing env vars, malformed YAML, credential issues.

## Cross-cutting

- [ ] **Backup all keystores** (development, preview, production) to `oglasino-private`. EAS Cloud is the canonical store; local backup is recovery insurance against EAS account loss or accidental deletion.
- [ ] **Document the Apple ID + 2FA flow** for whoever else may run iOS builds (currently single-developer, but future-proof).
- [ ] **Confirm Apple Developer Program auto-renewal** is enabled. Expiration mid-launch is catastrophic.
- [ ] **Confirm Firebase project billing** is set up if either project exceeds the free tier (Spark plan limits vs Blaze plan).

## After production launches

Things you cannot do before launch but must do shortly after:

- [ ] **Mark conversion events** in production GA4 (`G-GNKB4WBNC0`): `sign_up`, `product_create_completed`, `contact_seller_clicked`, `message_sent`. GA4 only allows marking after at least one event of each type has been received.
- [ ] **Connect Search Console to production GA4** via Admin → Product links → Search Console links. Only useful once Search Console has organic-search data, typically 1-2 weeks after launch.
- [ ] **Monitor first Sunday's messaging cleanup cron** (per `oglasino-backend` `state.md` Risk Watch). Tracked separately in backend deploy.
- [ ] **Verify push delivery end-to-end** on both platforms with real user accounts (assuming APNs key + FCM service account are both uploaded).

## What this checklist does NOT cover

- Feature work (auth flows, image pipeline, messaging UI, etc.)
- Backend deploy gates (those live in backend-specific docs)
- Web deploy gates (those live in web-specific docs)
- Per-feature platform-adoption work (e.g., porting backend features to mobile)
- Native-module-specific concerns (camera permissions strings, location permissions copy, etc.) — these are tied to feature work, not infrastructure
