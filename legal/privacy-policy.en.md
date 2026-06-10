# Privacy Policy — Oglasino

**Last updated:** 10.06.2026.

---

## 1. Who we are

Oglasino is an online classifieds platform for buying, selling, and exchanging new and used goods. The platform operates in two country portals — Serbia (rs) and Montenegro (me) — and is also accessible to visitors from other countries, including the European Union. Oglasino does not handle or mediate payments between users; all transactions are arranged and completed directly between buyers and sellers.

You can use Oglasino through our website and through our native mobile apps for iOS and Android. This Privacy Policy applies to all of them. The mobile apps offer the same features as the website and handle your data in the same way, except where the device makes a difference — app permissions (such as the camera) and on-device storage — which we describe explicitly in Sections 2.16 and 7.

The data controller for Oglasino is:

**Igor Stojanović**
Bulevar 12. februar 32, 18000 Niš, Serbia
privacy@oglasino.com

Oglasino is currently operated as a sole proprietorship under the name above. The operator intends to incorporate a company in the future once the platform demonstrates sustained growth. When that happens, this Privacy Policy will be updated to reflect the new legal entity and you will be notified through the platform.

[LAWYER REVIEW: The platform is offered to EU visitors while the controller is established outside the EU/EEA. Assess whether an EU representative under GDPR Art. 27 must be appointed; if so, appoint one and add the representative's identity and contact details here.]

This Privacy Policy is published in Serbian and English. In case of any conflict between language versions, the Serbian version is authoritative. [LAWYER REVIEW: confirm that the authoritative-language approach for the Privacy Policy should match the Terms of Use (§17).]

---

## 2. The data we collect and why

This section describes every category of personal data Oglasino collects, why we collect it, and the legal basis we rely on under the General Data Protection Regulation (GDPR), the Serbian Law on Personal Data Protection, and Montenegrin data-protection law.

### 2.1 Account data

When you create an account on Oglasino, we collect and store:

- **Email address** — required for account creation, login, and account-related notifications.
- **Authentication identifier (Firebase UID)** — an internal identifier issued by our authentication provider, Firebase Authentication. This identifier links your account in our database to your authentication record.
- **Display name** — a name shown on your public profile and listings. You choose it when you register with an email address; for accounts created through Google sign-in, it is taken from your Google profile. (If a display name is ever unavailable at registration, the part of your email address before the "@" symbol may be used as a fallback.) You can change your display name at any time in your profile settings.
- **Sign-in provider** — whether you registered with email or with Google.
- **Email verification status** — whether your email has been verified.
- **Password** — your password is never sent to or stored by Oglasino. It is handled exclusively by Firebase Authentication, our authentication provider, directly from your device or browser. Oglasino has no access to it.
- **Account creation and last update timestamps.**

**Lawful basis:** performance of a contract (GDPR Article 6(1)(b)) — we cannot provide an account-based service without this data.

### 2.2 Profile data

In addition to account data, your profile may contain:

- **Phone number** — optional. You may provide one in your profile settings.
- **Profile picture (avatar)** — if you sign in with Google and your Google account has a profile picture, we make a copy of that picture and store it on our infrastructure (Cloudflare R2). If you do not have a Google picture, the platform displays an automatically generated initial-based avatar (your display name's first letter on a colored background) — this generated avatar is not a stored image and contains no separate personal data. You can upload, replace, or remove your profile picture at any time in your settings.
- **Personal description (bio)** — optional free-text you may add about yourself. If you write your bio in one language, we automatically translate it into the platform's other supported languages so that other users can read it in their language (see Section 2.8 on auto-translation).
- **Account rating** — a numeric rating derived from reviews you have received from other users.
- **Verification status** — an internal flag indicating whether your account has been verified by the platform.

**Lawful basis:** performance of a contract (Article 6(1)(b)) — profile data is part of the account-based service you have signed up for.

### 2.3 Location data

To create or browse listings, the platform needs to know which country portal you use and your local area. We collect:

- **Country portal (Serbia or Montenegro)** — chosen when you create your first listing.
- **Region** — chosen when you create your first listing.
- **City** — chosen when you create your first listing.

Your location is set at the level of your account, not per listing. All your listings are associated with the same region and city as your account; the platform does not support listing goods located in a different region or country than the seller. We do not use your device's GPS or any other automatic geolocation — including in our mobile apps; your area comes only from the portal, region, and city you select. (Photos you upload may themselves contain location metadata added by your camera — see Section 2.4.)

**Lawful basis:** performance of a contract (Article 6(1)(b)) — Oglasino is a geographically scoped local classifieds platform; the listings feature cannot function without your location.

### 2.4 Listing data

When you create a listing, we collect and store:

- **Title and description** — written by you, in any of the platform's supported languages. We automatically translate these into the platform's other supported languages (see Section 2.8).
- **Price and currency** — Serbian dinar (RSD) and euro (EUR) are supported on the Serbian portal; euro (EUR) is supported on the Montenegrin portal.
- **"Free" indicator** — whether the listing is for an item you are giving away at no cost (used in the platform's Free Zone).
- **Category and sub-category** — chosen from a fixed taxonomy.
- **Category-specific attributes** — structured fields that vary by category (for example, size, color, year for relevant categories).
- **Photos** — images you upload, stored on Cloudflare R2 (see Section 4 on third parties). Photos can contain hidden metadata added by your camera — for example, the time the photo was taken and, if enabled on your device, the location. We do not currently remove this metadata from uploaded images, so check your device's camera settings if you do not want it included in photos you publish.
- **Listing status** — whether the listing is active, banned, etc.
- **Counters** — view count and favorite count for each listing. The stored counters are anonymous totals. To count each view only once, we keep a short-lived technical record combining the listing with your device identifier or IP address for up to 12 hours, after which it is automatically deleted; we do not build a history of which listings you have viewed. Separately, if you have consented to analytics, your own browsing activity is part of the analytics described in Section 2.14.
- **Edit count** — how many times you have edited the listing.

**Lawful basis:** performance of a contract (Article 6(1)(b)) — listings are core platform functionality.

### 2.5 Messages and chat attachments

The platform includes a private messaging feature between users. We store:

- **Messages** — sender, recipient, message content, timestamp, read status. Messages are stored on Google Firestore (see Section 4).
- **Image attachments** — you may attach images to messages. These are stored on Cloudflare R2 and are accessible only through signed, time-limited links.
- **Optional product reference** — when you start a conversation with another user by clicking "Send message" on a listing's page, the first message you send in that conversation includes a reference to that listing.
- **Notifications** — we notify you about relevant activity, and what the notification contains depends on the event:
  - **Someone favorites your listing** — we store an in-app notification on Firestore and send a push notification. It names the listing but does **not** identify the user who favorited it.
  - **Someone follows you** — we store an in-app notification and send a push notification that includes the follower's display name and a link to their profile.
  - **An administrator acts on your account or content** — we store an in-app notification and send a push notification describing the action; it contains no other user's personal data.
  - **You receive a chat message** — we send a **push notification only**; no in-app notification record is stored. So that you can read the message from your device's notification screen, as is standard for messaging, this notification shows the sender's display name and the message text.

**Access to messages.** The platform's administrators may access message contents when investigating reports of misconduct or violations of our Terms of Use — for example, a harassment report about another user. This access happens through our administrative tools, not through any general visibility: outside of moderation work, your messages are visible only to you and the other participant in the conversation. For what happens to messages when an account is deleted, see Section 8.3.

**Lawful basis:** performance of a contract (Article 6(1)(b)) for storing and delivering messages; legitimate interest (Article 6(1)(f)) for administrator access to messages for moderation and safety purposes.

### 2.6 Reviews

The platform allows buyers and sellers to leave reviews about each other after an interaction. We store, for each review:

- **Reviewer** — the user who wrote the review.
- **Target user** — the user the review is about.
- **Product reference** — if the review relates to a specific listing.
- **Rating** — a number from 1 to 5.
- **Comment** — free text. We automatically translate review comments into the platform's other supported languages (see Section 2.8).
- **Images** — optional photos attached to the review, stored on Cloudflare R2.
- **Verification status** — whether the reviewer had a legitimate platform interaction with the target user related to a product.
- **Approval status** — whether the review has been approved by a platform administrator for public display. All reviews require administrator approval before they become visible to other users.

**Lawful basis:** performance of a contract (Article 6(1)(b)) — reviews are part of the platform's trust system.

### 2.7 Favorites and follows

When you save a listing to your favorites, we store a record linking your account to that listing. Your list of favorites is visible only to you. The listing's owner is notified that the listing was favorited, but that notification does not identify you (see Section 2.5).

**Follows.** You can follow other users. When you follow someone, we store a record linking your account to theirs, and the followed user receives a notification that includes your display name (see Section 2.5). The list of accounts you follow is visible only to you; we do not show other users your followers or who you follow. Follow records are deleted when either account is permanently deleted.

**Lawful basis:** performance of a contract (Article 6(1)(b)).

### 2.8 Automatic translation of your content

The platform supports four languages: Serbian, English, Russian, and Montenegrin. When you write a listing title, listing description, profile bio, or review comment in one language, we send your text to **OpenAI** (an external service based in the United States) for automatic translation into the platform's other supported languages. The translated versions are stored alongside your original text so that users on the platform can see your content in their preferred language.

These requests contain only the text to be translated, wrapped in our translation instructions — they do not include your name, email address, account identifier, or any other metadata about you, and we ask OpenAI not to store the request content. Chat messages are never sent to OpenAI.

**AI description suggestion.** When creating a listing, you can optionally ask the platform to suggest a description. If you use this feature, the listing name you typed is sent to OpenAI to generate the draft, under the same conditions described above.

OpenAI's API terms commit that data sent through their API is not used to train their models. Oglasino itself does not train any AI model on your content (see Section 6 for our broader commitments on data use).

This processing involves transferring your text outside the European Economic Area to the United States. See Section 5 on international transfers.

**Lawful basis:** performance of a contract (Article 6(1)(b)) — multilingual content is a core feature of the platform.

### 2.9 Reports

Any user may report another user, a listing, or a review for violating our Terms of Use. When a report is filed, we store:

- **Who filed the report** — the reporter's account.
- **Who or what is being reported** — the target user, listing, or review.
- **Report category** — one of: fraud, poor service, rules violation, violence or harassment, inappropriate content, misleading or fake listing, portal rules violation, technical issue, other.
- **Description** — free text from the reporter.
- **Resolution status** — whether an administrator has reviewed the report, and any resolution notes.

Currently, reports are reviewed manually by an administrator. In the future, we plan to introduce automated handling of certain report categories; this Privacy Policy will be updated when that change is made. For how long reports are kept, see Section 8.6.

**Lawful basis:** legitimate interest (Article 6(1)(f)) — platform safety, abuse prevention, and policy enforcement.

### 2.10 Moderation and platform safety data

To keep the platform safe, we maintain the following data about each user:

- **Moderation status** of your listings — whether each of your listings is approved or banned.
- **Number of penalties** issued against your account.
- **Disabled status** — whether your account has been disabled by an administrator.
- **Internal verification status** — whether your account has been internally verified.

**Lawful basis:** legitimate interest (Article 6(1)(f)) — platform safety and policy enforcement.

### 2.11 Communication preferences

Your account stores your communication preferences, which you can change at any time in your settings:

- Whether you allow transactional emails about your account.
- Whether you allow promotional or marketing emails.
- Whether you allow other users to call your phone number.

Cookie and analytics choices are not stored as account settings: on the website you make them through the cookie consent banner (stored in your browser), and in the mobile app through the app's analytics setting (stored on your device). See Section 7. Notifications are controlled by your device's or browser's notification permissions rather than by an account setting (see Sections 2.5 and 2.15): turning notifications off in your operating system or browser stops them, and signing out detaches your device from push notifications.

Note that even if you turn off "transactional emails," we may still send you emails that are essential to your use of the platform (for example, confirming that your account is scheduled for deletion). Truly optional and promotional emails are gated by the "promotional emails" toggle.

**Lawful basis:** consent (Article 6(1)(a)) for each preference. You can withdraw consent at any time by changing the toggle in your settings; withdrawal does not affect the lawfulness of processing before withdrawal.

[LAWYER REVIEW: Emails that are essential to the service (for example, the deletion confirmation) are sent regardless of the "transactional emails" toggle. Consider whether contract (Art. 6(1)(b)) or legitimate interest is the accurate basis for essential service emails, with consent governing only the optional categories.]

### 2.12 Technical data

To keep the platform functioning, secure, and abuse-resistant, we process:

- **IP address** — in two short-lived technical records: a rate-limiting record in our Redis cache, retained for approximately 30 minutes, and the view-deduplication record described in Section 2.4, retained for up to 12 hours. IP records are tied to your account when you are logged in.
- **Application logs** — operational logs of activity on the platform, used for debugging, security monitoring, and incident response. Each logged request includes technical metadata such as your IP address and, when you are logged in, your account identifier. See Section 8.2 for retention.
- **Cookies** — see Section 7.

**Lawful basis:** legitimate interest (Article 6(1)(f)) — security, abuse prevention, and platform reliability.

### 2.13 Account deletion records

When you delete your account, we retain a limited audit record after deletion. See Section 8 (Data retention) for full details.

### 2.14 Analytics data

To understand how Oglasino is used and to improve it, we collect analytics data using Google Analytics 4 (GA4) on both the website and the mobile app — but **only if you consent**. When enabled, GA4 collects pseudonymous usage information such as pages or screens viewed, searches and filters used, listings viewed, and similar interaction events, together with a pseudonymous analytics identifier (on the web, an analytics cookie identifier; in the mobile app, a Firebase app-instance identifier). If you are logged in, we associate these events with your account's user identifier so that usage can be understood across your sessions. Our analytics is first-party only: we do not use it for advertising, ad targeting, or cross-site or cross-app tracking, and Google's advertising signals are disabled. How you give or withhold consent differs by platform: on the website, through the cookie banner; in the mobile app, through a device-level choice you make when you first open the app and can change anytime in the app's settings (see Section 7 for both). Analytics is off by default; we do not enable analytics collection unless you have allowed it. Analytics data is retained in Google Analytics for [2/14] months, after which it is automatically deleted.

**Lawful basis:** consent (Article 6(1)(a)). You can withdraw consent at any time — on the website through the cookie banner or the cookie-preferences page, and in the mobile app through the app's analytics setting; withdrawal does not affect the lawfulness of processing before withdrawal.

### 2.15 Push notification tokens

If you allow notifications — in the Oglasino mobile app, or in your web browser on our website — your device or browser is issued a **push token**: an identifier that lets us send push notifications to that specific device or browser. We store your push token in our database and associate it with your account so that we can deliver notifications to you (for example, new-message alerts; see Section 2.5). In the mobile app, your push token is detached from your account when you sign out, and is replaced or removed as your device or its permissions change. All stored push tokens are deleted when your account is permanently deleted. Push notifications are delivered through the providers listed in Section 4: in the mobile app via the Expo Push Service (with Apple's and Google's underlying push services), and in the browser via Firebase Cloud Messaging.

**Lawful basis:** performance of a contract (Article 6(1)(b)) — delivering notifications is part of the messaging and account service you sign up for. You can stop push notifications at any time by turning them off in your device or browser settings or by signing out.

### 2.16 Using the mobile app — device permissions and device data

Our iOS and Android apps ask your permission to use certain device features, and only when a feature needs them:

- **Camera and photo library** — when you take or choose a photo to add to a listing, your profile picture, a message, or a review. The app accesses the camera or photo library only at the moment you choose to add a photo; the photos you then select are handled as described in Sections 2.2, 2.4, 2.5, and 2.6.
- **Notifications** — to send you push notifications. This results in a push token for your device (see Section 2.15).

You can grant or revoke these permissions at any time in your device's settings.

To be clear about what the mobile app does **not** do: it does **not** access your device's location (GPS) — the region and city used on the platform are the account-level values you choose yourself (see Section 2.3), not your device location; it does **not** read your contacts; it does **not** collect an advertising identifier (such as Apple's IDFA or Android's advertising ID); it does **not** use Apple's App Tracking Transparency; and it does **not** track you across other apps or websites.

<!-- RESOLVED IN CODE 2026-06-10 — see §2.14 expo note. One nuance for the Play Console data-safety form: the merged Android manifest still *declares* ACCESS_ADSERVICES_AD_ID / ACCESS_ADSERVICES_ATTRIBUTION (transitive from play-services-measurement) while collection is disabled; this sentence about collection stays true, but answer the store form as declared-but-not-collected. -->

The app also stores some information on your device; see Section 7.

---

## 3. How we use your data — summary

To make Section 2 easier to navigate, here is a summary of every purpose for which we use your data and the legal basis for each. Detailed disclosures are in Section 2.

| Purpose                                                                                       | Lawful basis        |
| --------------------------------------------------------------------------------------------- | ------------------- |
| Creating and operating your account                                                           | Contract            |
| Displaying your profile to other users                                                        | Contract            |
| Publishing your listings to other users                                                       | Contract            |
| Delivering messages between users                                                             | Contract            |
| Displaying reviews about you and from you                                                     | Contract            |
| Saving your favorite listings and the accounts you follow                                     | Contract            |
| Automatically translating your content into other supported languages                         | Contract            |
| Sending you the communications you have opted into                                            | Consent             |
| Setting non-essential cookies in your browser                                                 | Consent             |
| Measuring and improving the platform with analytics (only if you consent)                     | Consent             |
| Delivering push notifications to your device or browser                                       | Contract            |
| Preventing fraud, abuse, and spam                                                             | Legitimate interest |
| Moderating content and enforcing platform rules                                               | Legitimate interest |
| Maintaining the platform's security and reliability                                           | Legitimate interest |
| Investigating reports filed by users                                                          | Legitimate interest |
| Retaining a hashed audit record of account deletions and bans                                 | Legitimate interest |
| Complying with legal obligations (for example, preserving data at the request of authorities) | Legal obligation    |

---

## 4. Third parties that process your data

We use a number of external service providers to operate Oglasino. Each of them processes some of your data on our behalf. We never sell your data to anyone (see Section 6 on our specific commitments).

| Provider                                                                                              | Role                                                                                           | What they process                                                                                                                                                                                                                                                                                                                                                     | Location                                                                               |
| ----------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| **Google Firebase Authentication**                                                                    | Authentication provider                                                                        | Your email, password (held only by Firebase), Google sign-in tokens                                                                                                                                                                                                                                                                                                   | Google global infrastructure; processing may include the United States (see Section 5) |
| **Google Firestore**                                                                                  | Real-time database for messages and notifications                                              | Message contents, notifications, sender display names                                                                                                                                                                                                                                                                                                                 | Frankfurt, Germany (europe-west3 region)                                               |
| **Cloudflare R2**                                                                                     | Object storage for images                                                                      | Profile pictures, listing photos, chat attachments, review photos                                                                                                                                                                                                                                                                                                     | Eastern Europe (EEUR)                                                                  |
| **Cloudflare**                                                                                        | Network edge, content delivery network, and image-access service                               | All traffic to our website and API passes through Cloudflare's edge, which terminates encrypted connections and routes requests to our hosting; Cloudflare caches static content and images, and runs the service that checks signed access tokens when you upload images or view private images, processing your account identifier and IP address in technical logs | Global edge network                                                                    |
| **OpenAI**                                                                                            | Automatic translation of user-written content; optional listing-description suggestions        | Listing titles and descriptions, profile bios, review comments (text only — see Section 2.8)                                                                                                                                                                                                                                                                          | United States                                                                          |
| **DigitalOcean**                                                                                      | Backend application hosting (including our self-hosted cache and search services)              | All data processed by our backend services                                                                                                                                                                                                                                                                                                                            | Frankfurt, Germany (fra1 region)                                                       |
| **Vercel**                                                                                            | Web frontend hosting (including server-side rendering)                                         | Page requests, session cookies, request bodies                                                                                                                                                                                                                                                                                                                        | Frankfurt, Germany (fra1 region)                                                       |
| **Google reCAPTCHA**                                                                                  | Bot detection on forms                                                                         | IP address, browser information, mouse and timing data, cookies set by Google                                                                                                                                                                                                                                                                                         | United States                                                                          |
| **Google Analytics 4 (Google LLC)**                                                                   | Website and mobile-app usage analytics                                                         | Pseudonymous usage events and analytics identifiers (the `_ga` cookie on the web; a Firebase app-instance identifier in the mobile app); collected only if you consent to analytics                                                                                                                                                                                   | United States                                                                          |
| **Brevo**                                                                                             | Sending account and platform emails (e.g. email verification, password reset, account notices) | Your email address and the content of those emails                                                                                                                                                                                                                                                                                                                    | European Union                                                                         |
| **Expo Push Service, with Apple Push Notification service (APNs) and Firebase Cloud Messaging (FCM)** | Delivering push notifications (mobile app via Expo with APNs/FCM; web browser via FCM)         | Your device or browser push token and the notification content (including, for chat-message notifications, the sender's display name and message text)                                                                                                                                                                                                                | United States (Expo; Apple; Google)                                                    |

**Disclosure to authorities.** In addition to the processors above, we may disclose personal data to courts, law-enforcement, or other public authorities when we receive a legally binding request, and to professional advisers (such as lawyers) where necessary to establish, exercise, or defend legal claims. [LAWYER REVIEW: confirm the scope and wording of this authority-disclosure clause.]

---

## 5. International transfers

Most of our processing takes place inside the European Economic Area (EEA), which has the data-protection standards required by GDPR.

Several of our processors are based in, or may process data in, the United States: **OpenAI** (for content translation), **Google reCAPTCHA** (for bot detection), **Google Analytics** (only if you consent), **Google Firebase Authentication** (operated on Google's global infrastructure), and the providers that deliver push notifications (**Expo**, and the underlying **Apple** and **Google** push services, including Firebase Cloud Messaging for the browser). When your data is sent to these processors, it may leave the EEA.

We rely on the following safeguards for these transfers:

- **OpenAI** — we transfer data under the European Commission's Standard Contractual Clauses, and OpenAI's adherence to the EU-US Data Privacy Framework (where applicable). The controller (Oglasino) uses OpenAI API services under the OpenAI Services Agreement. The OpenAI Data Processing Addendum is incorporated into that agreement and applies to use of the services.
- **Google reCAPTCHA** — we rely on Google's adherence to the EU-US Data Privacy Framework and Google's Standard Contractual Clauses.
- **Google Analytics** — for users who consent to analytics, we rely on Google's adherence to the EU-US Data Privacy Framework and Google's Standard Contractual Clauses.
- **Firebase Authentication** — Google operates Firebase Authentication on global infrastructure, and processing may include the United States. We rely on Google's adherence to the EU-US Data Privacy Framework and the Standard Contractual Clauses incorporated in Google's Firebase/Google Cloud data-processing terms.
- **Push notification delivery** — push notifications are routed through Expo (mobile app) and Firebase Cloud Messaging (browser), with Apple's (APNs) and Google's (FCM) services underneath. For Expo, we rely on a data-processing agreement with Expo incorporating the European Commission's Standard Contractual Clauses; for Apple and Google, we rely on their adherence to the EU-US Data Privacy Framework and Standard Contractual Clauses. [LAWYER REVIEW: confirm that a DPA/SCCs with Expo is actually in force before publication — currently unverified (open item, 2026-06-10).]

You can request more information about these safeguards by contacting us at privacy@oglasino.com.

---

## 6. What we will never do with your data

In addition to the obligations we owe you under GDPR, we make two explicit commitments about your data:

**We do not sell your data.** We never have, and we never will. We do not sell user data to advertisers, data brokers, or any third party. We share data with processors only as described in Section 4, and only to the extent necessary to operate the platform.

**We do not train AI on your data.** Oglasino does not use your content — your listings, your messages, your reviews, your profile — to train any AI or machine-learning model. This commitment is specifically about what we do with your data. Separately, our translation provider OpenAI commits in their API terms that data sent through their API is not used to train their models; that is a commitment OpenAI makes about their own use of the data we send them.

If we ever change either of these commitments, we will tell you clearly before the change takes effect, and we will give you the opportunity to delete your account before any new processing begins.

---

## 7. Cookies and similar technologies

The Oglasino website uses cookies and similar browser storage mechanisms (such as localStorage and IndexedDB); our mobile apps use similar on-device storage instead of cookies. This section covers both — the website first, then the mobile app. On the website, we use cookies and similar storage for three purposes:

**Strictly necessary** — required for the website to function. These include your authentication session (managed by Firebase Authentication, stored in localStorage and IndexedDB), security tokens (for example, CSRF protection), the cookie consent state itself, and a small number of similar technical cookies.

**Preferences** — used to remember your choices, such as your preferred language, your selected country portal (Serbia or Montenegro), and similar settings.

**Analytics** — used to measure how the website is used so that we can understand and improve it. We use Google Analytics 4 (GA4). Analytics cookies (for example, `_ga` and `_gid`) and analytics collection are set **only if you consent** to the analytics category; if you decline, no analytics cookies are set and no analytics data is collected. We use Google Consent Mode v2: of Google's four consent signals, only analytics storage can be enabled by your choice — the three advertising-related signals (`ad_storage`, `ad_user_data`, `ad_personalization`) are permanently denied, because we do not use Google's advertising features. Our analytics is first-party only; we do not use it for advertising, ad targeting, or cross-site tracking.

<!-- RESOLVED IN CODE 2026-06-10 (oglasino-web, basic Consent Mode) — see §2.14 note; pending Igor's browser checklist. -->

**Cookie lifetimes.** The `og_consent` consent cookie is stored for 12 months. Google's analytics cookies follow Google's defaults (`_ga` up to 2 years, `_gid` about 24 hours) and are set only after you consent. The `_GRECAPTCHA` cookie is set and managed by Google.

This list of cookies may be updated over time as the platform evolves; we will keep this section current.

**Cookie consent.** When you first visit Oglasino, a consent banner lets you **Accept all**, **Reject all**, or **Customize** your choices by category. Strictly necessary cookies cannot be declined because the site cannot function without them; the preferences and analytics categories are optional and stay off until you allow them. Your choice is recorded in a small cookie stored in your own browser (named `og_consent`) — because it is stored in your browser rather than on your account, the choice applies to that browser. You can change your cookie preferences at any time on our cookie-preferences page at `/owner/cookies` (linked from your account settings page), or by reopening the consent banner using the "Manage cookies" control in the website footer.

**Third-party scripts.** One third-party script on our site requires specific disclosure: **Google reCAPTCHA**. reCAPTCHA loads on pages with forms — registration, listing creation, review submission, category suggestions, and report forms — to prevent automated abuse. It sets its own cookies (including the `_GRECAPTCHA` cookie) and collects information including your IP address, browser environment, mouse movements, and the contents of any Google session cookies you already have. We rely on reCAPTCHA's processing as a legitimate interest in fraud prevention. For more information, see Google's own privacy disclosures at https://policies.google.com/privacy.

**Mobile app — on-device storage.** The mobile apps do not use cookies. Instead, they store information on your device to make the app work, including: your sign-in session (held by Firebase Authentication); your selected portal, language, theme, and display preferences; your analytics choice (see below); cached translations and catalog data; timing markers used to avoid re-asking for permissions or re-showing the same notification; and a cached copy of images the app has displayed. Most of this is functional or preference data. Your sign-in session and the stored copy of your account profile are cleared when you sign out; a small number of non-identifying functional items — for example, a flag recording that your account was already set up on the device, and the app's image cache — may remain on the device after sign-out until your operating system reclaims them. You can clear everything by signing out and deleting the app, or through your device's app-storage controls.

**Mobile app — analytics consent.** Analytics in the mobile app works differently from the website: there is no cookie banner and there are no cookies. Instead, when you first open the app we ask whether to allow analytics, and analytics is **off by default** — we do not enable analytics collection unless you allow it. You can change your choice at any time in the app's settings. The mobile app's analytics is the same first-party GA4 described above, reporting into the same analytics property using a Firebase app-instance identifier; it does **not** use Apple's App Tracking Transparency and does **not** collect an advertising identifier.

<!-- RESOLVED IN CODE 2026-06-10 (oglasino-expo) — see §2.14 note; pending EAS on-device smoke. -->

---

## 8. How long we keep your data

The default rule is simple: while your account is active, we keep your data for as long as it is needed to provide the service. When you delete your account, we delete your data — with a few specific, time-limited exceptions described below.

### 8.1 While your account is active

Your account data, profile data, listings, listing images, messages, chat attachments, reviews, favorites, and follow relationships are retained for as long as your account is active. Listings do not currently expire automatically; you can delete individual listings at any time from your account.

In the future, we plan to add automatic deletion of listings and chat messages that have been inactive for an extended period. When we introduce this, we will update this Privacy Policy.

### 8.2 IP addresses and operational logs

- **IP addresses in short-lived technical records** are retained for approximately 30 minutes (rate limiting) and up to 12 hours (view deduplication) — see Sections 2.4 and 2.12.
- **Application logs**, which include IP addresses and account identifiers (see Section 2.12), are retained for **90 days**, after which they are purged.
<!-- RESOLVED IN REPO 2026-06-10 (oglasino-backend infra): json-file size caps on all services + infra/prune-container-logs.sh (daily cron deletes the backend container's rotated segments older than 88 days) + infra/LOG-RETENTION.md. The 90-day claim goes live only after Igor's host apply: ship files, `up -d --force-recreate` affected containers, install the root cron. Do not publish before the host apply. -->

### 8.3 When you delete your account

Account deletion on Oglasino is a two-stage process designed to give you a meaningful change-of-mind window and, equally important, to give other users a fair last chance to report you if they believe you have acted in bad faith on the platform.

**Stage 1 — Soft deletion (7 days).** When you request account deletion from your settings, your account immediately enters a "scheduled for deletion" state:

- **Your profile remains publicly visible** and is marked with a "Scheduled for deletion" status badge so that other users can see you have requested deletion. We deliberately keep your profile visible during this window, rather than hiding it, so that users who have interacted with you still have the opportunity to file a report if they have a legitimate concern about your conduct.
- **Your phone number is hidden** during this window, so that you are not contacted while your deletion is pending.
- **Your listings are hidden** from public view.
- **Reviews about you remain visible** so that the context for any potential reports is preserved.
- **You cannot send or receive messages** during the grace period. Existing conversations remain accessible to the other participants for reporting purposes, but no new messages can be sent to or from your account.
- **Reports against you remain open.** Any user who interacted with you can still file a report during the grace period.
- You are signed out.

During these 7 days, you can restore your account at any time simply by signing back in. We do this so that you do not lose your data because of an accidental deletion, a moment of frustration, or a change of mind. If you sign in during the grace period, your profile, listings, phone-number visibility, and messaging are all restored.

**Stage 2 — Hard deletion (after 7 days).** On the 8th day, your account is permanently deleted, subject to the postponement exceptions described in Section 8.4. Hard deletion means:

- Your account record, listings, listing images, favorites, follow relationships, stored push tokens, and in-app notifications are permanently erased from our databases and storage.
- Your profile picture is deleted from our storage.
- Your authentication record at Firebase is deleted.
- **Chat messages are anonymized rather than erased.** Your conversations are part of the other participant's message history too, so the message content remains available to them — but your identity is removed: your account's references in those conversations are replaced with an anonymous marker, shown to other users as "Deleted User." The same applies to images you attached to messages. [LAWYER REVIEW: confirm that retaining anonymized message content after an erasure request is the intended and defensible position (Art. 17(1) balanced against the other participant's interest in their own correspondence), or whether message content should be deleted as well.]
- **Reviews you have left about other users:** approved reviews are kept, with your name replaced by "Deleted User" — they reflect the experience of the user you reviewed, and erasing them would erase part of their record. We disclose this here so that, when you leave a review, you know an approved review survives your account deletion (anonymized). Reviews of yours that were still awaiting approval, or were declined, are deleted. Reviews about you are deleted.
- **Residual copies.** Copies of deleted content can persist for a limited time in technical systems that refresh on their own schedule — for example, edge caches of images and entries in our search index — until those systems expire or re-synchronize.

After hard deletion, the only data we retain are the limited audit records described in Section 8.5 and, if your account was banned, the anonymized report records described in Section 8.6.

### 8.4 When deletion may be postponed

In limited circumstances, we may delay hard deletion of your account beyond the standard 7-day grace period. This applies only in two cases:

- **Legal investigations.** When a law-enforcement authority (such as the police or a court) asks us to preserve your data in connection with an ongoing investigation, we are legally required to comply and we will postpone deletion for as long as the legal request requires.
- **Internal abuse investigations.** When we have reason to believe your account has been used for serious abuse of the platform — for example, fraud against multiple users — and we are actively investigating, we may postpone deletion to preserve evidence relevant to the investigation.

In both cases, if your deletion is postponed, we will tell you that the deletion has been delayed and, to the extent we are legally permitted to do so, why. You will continue to be signed out and unable to use the account during the postponement; the only effect is that hard deletion is paused. Once the legal or internal investigation concludes, hard deletion proceeds.

**Lawful basis for postponement:** legal obligation (Article 6(1)(c)) for law-enforcement preservation requests, and legitimate interest (Article 6(1)(f)) for internal abuse investigations.

### 8.5 Audit records retained after deletion or ban

- **A general deletion audit record** — a hashed version of your user identifier and email (using SHA-256, a one-way hash that cannot be reversed to recover your original email), along with the date your account was deleted and the reason. This is kept for **30 days** after deletion, then automatically purged.
- **Banned-user audit record** — if your account is banned, we record a hashed version of your email and the reason for the ban at the moment the ban is issued. This record is used to prevent banned users from re-registering with the same email address, and is retained for **12 months** from the ban, then automatically purged. It remains in place if the banned account is later deleted.

These records contain no readable personal data — only one-way hashes. A hash cannot be reversed to reveal your email address, although it can be compared against a known email address (that comparison is exactly how the re-registration check works). After the retention period expires, even the hash is deleted. [LAWYER REVIEW: confirm the treatment of hashed identifiers as pseudonymized personal data and the adequacy of this disclosure.]

### 8.6 Reports filed by or about users

While the accounts involved are active, reports are retained so that administrators can detect patterns of repeat abuse; we currently keep them without a fixed time limit, and we plan to introduce automatic purging of resolved reports after a set period. What happens to reports when an account involved in them is permanently deleted depends on whether that account was banned:

- If the deleted account was **not banned**, reports filed by it and about it are deleted together with the account.
- If the deleted account was **banned**, the reports are kept — with all account references removed, linked only to the anonymized ban record from Section 8.5 — and are deleted together with that record, at the latest 12 months after the ban.

### 8.7 If your account is banned

If we permanently ban your account (see the Terms of Use, Section 10.2), the ban takes effect immediately: your public profile and phone number stop being shown, your listings are hidden, and your sign-in is disabled. The ban does not by itself trigger the deletion process — we retain the account's data for as long as it is needed to document the violation, handle any appeal, protect other users, and establish, exercise, or defend legal claims. You can request erasure of a banned account at any time (see Section 9), subject to the postponement cases in Section 8.4. A banned-user audit record (Section 8.5) is created when the ban is issued and is retained for 12 months.

**Lawful basis:** legitimate interest (Article 6(1)(f)) — platform safety and the establishment, exercise, or defense of legal claims. [LAWYER REVIEW: confirm the basis and proportionality of retaining banned-account data, and whether a fixed maximum retention period should be stated.]

---

## 9. Your rights

Under GDPR and Serbian and Montenegrin data-protection law, you have the following rights about your personal data. We support each of them.

**Right of access.** You can request a copy of the personal data we hold about you. Email privacy@oglasino.com with your request. We will respond within one month, as required by GDPR.

**Right to rectification.** You can correct most of your personal data directly in your profile settings. For data you cannot self-edit (such as moderation-related information), email privacy@oglasino.com and we will correct any inaccuracies.

**Right to erasure ("right to be forgotten").** You can delete your account at any time from your account settings; see Section 8.3 for what this involves. If you are unable to use the self-service deletion flow — for example, because your account has been disabled — contact support@oglasino.com to request deletion and we will respond within one month, as required by GDPR. For all other privacy-related deletion enquiries, email privacy@oglasino.com.

**Right to restriction of processing.** You can ask us to limit how we process your personal data — for example, while you contest a moderation decision. Email privacy@oglasino.com with your request and the reason.

**Right to data portability.** You can request a copy of the personal data you have provided to us, in a structured, commonly used, machine-readable format (such as JSON). This right applies to data you provided directly to us (your profile, listings, messages, reviews); it does not extend to data we derived about you (such as your rating or audit records). Email privacy@oglasino.com to request a portable export.

**Right to object.** You can object to our processing of your data when we rely on legitimate interest. You can object to receiving promotional emails by turning off the corresponding toggle in your settings, and we will always honor that. For other objections, email privacy@oglasino.com and we will assess your objection.

**Right to withdraw consent.** Where we rely on your consent (cookies and analytics, communication preferences), you can withdraw it at any time:

- Cookie and analytics preferences on the website: on the cookie-preferences page at `/owner/cookies` (linked from your account settings), or by reopening the consent banner from the "Manage cookies" control in the website footer.
- Analytics in the mobile app: through the analytics setting in the app.
- Communication preferences: through the toggles on your account settings page.

Withdrawing consent does not affect the lawfulness of any processing we did before withdrawal.

**Right to lodge a complaint with a supervisory authority.** If you believe we have not handled your personal data lawfully, you have the right to complain to a data-protection authority. In Serbia, the relevant authority is the Commissioner for Information of Public Importance and Personal Data Protection (Poverenik za informacije od javnog značaja i zaštitu podataka o ličnosti) — https://www.poverenik.rs/. In Montenegro, the relevant authority is the Agency for Personal Data Protection and Free Access to Information (Agencija za zaštitu ličnih podataka i slobodan pristup informacijama) — https://www.azlp.me/. If you are in the EU, you can also complain to the data-protection authority in your country of residence.

**How we respond to requests.** We aim to respond to all data-subject requests within one month of receiving them. In limited cases involving particularly complex requests, GDPR allows us to extend this by up to two further months, in which case we will tell you within the first month.

**Automated decision-making.** Oglasino does not make significant decisions about you using purely automated processes, with one narrow exception described below. Content validation runs automatically when you create or edit a listing (for example, checking for prohibited content), but it can only reject the submission with an error message — enforcement decisions such as banning a listing, disabling an account, or resolving a report are made by a human administrator. The one exception: when an account's sign-in record no longer exists at our authentication provider (an "orphaned" account) and that account has multiple unresolved reports against it, our cleanup process removes the orphaned account and automatically records the re-registration block described in Section 8.5, without an individual human decision. [LAWYER REVIEW: assess this path against GDPR Art. 22 — an automated decision with a 12-month re-registration effect — and whether human review or an objection route should be added.] If we introduce further automated decision-making in the future, we will update this Privacy Policy.

---

## 10. Security

We take reasonable technical and organizational measures to protect your personal data against unauthorized access, alteration, disclosure, and loss. These include:

- Encryption of data in transit (HTTPS for all connections between your browser and our services).
- Encryption of data at rest by our infrastructure providers (Firebase, Firestore, Cloudflare R2, Postgres on DigitalOcean).
- Authentication and authorization controls limiting access to user data, including signed, time-limited access to private images.
- Logging of administrative access and administrative actions.
- Hashing of stored email addresses and identifiers in our deletion and ban audit records.

No system is perfectly secure, and we cannot guarantee absolute security. If we learn of a personal data breach that creates a risk to your rights, we will notify the relevant supervisory authority and, where required by GDPR, notify you directly.

---

## 11. Children

Oglasino is intended for users aged **18 and over**. We do not knowingly collect personal data from anyone under 18. If you are under 18, please do not register or use the platform. If you believe a minor has registered, please contact privacy@oglasino.com and we will take prompt action to remove the account.

---

## 12. Changes to this Privacy Policy

We may update this Privacy Policy from time to time — for example, when we add features, change processors, or in response to changes in the law. When we make a material change, we will:

- Update the "Last updated" date at the top of this document.
- Show a notice on the platform.
- Re-display the cookie consent banner so that you can review your cookie preferences against any changes.
- For changes that materially affect your rights or how we process your data, ask for your explicit acceptance before you continue using the platform. [LAWYER REVIEW: align acceptance mechanics with Terms §16.]

Continued use of the platform after a change takes effect means you accept the updated Privacy Policy, except where we have asked for your explicit acceptance. If you do not accept the changes, you can delete your account before they take effect (see Section 8.3).

---

## 13. Contact

For any question about this Privacy Policy or about how Oglasino processes your data, contact us at:

**privacy@oglasino.com**

For all other (non-privacy) questions, please use the platform's general support channels.

---

_End of draft._
