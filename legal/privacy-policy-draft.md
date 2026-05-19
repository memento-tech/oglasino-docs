# Privacy Policy — Oglasino

> **PRE-LAWYER DRAFT — NOT LEGAL ADVICE.**
> This document is a structured first draft prepared for review by a qualified lawyer before publication. It reflects the platform owner's intent and the structured intake captured in the legal-drafts chat. It does not constitute a binding legal commitment until reviewed, finalized, and published. Inline `[LAWYER REVIEW: ...]` notes mark clauses where a lawyer's judgment is specifically required.

**Last updated:** [LAWYER REVIEW: set to publication date when finalized.]

---

## 1. Who we are

Oglasino is an online classifieds platform for buying, selling, and exchanging new and used goods. The platform operates in two country portals — Serbia (rs) and Montenegro (me) — and is also accessible to visitors from other countries, including the European Union. Oglasino does not handle or mediate payments between users; all transactions are arranged and completed directly between buyers and sellers.

The data controller for Oglasino is:

**Igor Stojanović**
Bulevar 12. Februar 32, 18000 Niš, Serbia
privacy@oglasino.com

Oglasino is currently operated as a sole proprietorship under the name above. The operator intends to incorporate a company in the future once the platform demonstrates sustained growth. When that happens, this Privacy Policy will be updated to reflect the new legal entity and you will be notified through the platform.

[LAWYER REVIEW: Controller identity — operator is currently an individual (sole proprietor), not a registered company. Confirm this is acceptable for launch and that the planned transition to a company structure can be handled as a Privacy Policy update at the time of incorporation, rather than requiring the document to be drafted against a not-yet-existing entity. Also confirm the home address as the controller's contact address is acceptable or whether a registered correspondence address should be obtained before launch.]

---

## 2. The data we collect and why

This section describes every category of personal data Oglasino collects, why we collect it, and the legal basis we rely on under the General Data Protection Regulation (GDPR) and Serbian Law on Personal Data Protection.

### 2.1 Account data

When you create an account on Oglasino, we collect and store:

- **Email address** — required for account creation, login, and account-related notifications.
- **Authentication identifier (Firebase UID)** — an internal identifier issued by our authentication provider, Firebase Authentication. This identifier links your account in our database to your authentication record.
- **Display name** — for accounts created with email, this is initially derived from the part of your email address before the "@" symbol; for accounts created through Google sign-in, it is taken from your Google profile. You can change your display name at any time in your profile settings.
- **Sign-in provider** — whether you registered with email or with Google.
- **Email verification status** — whether your email has been verified.
- **Password** — your password is never sent to or stored by Oglasino. It is handled exclusively by Firebase Authentication, our authentication provider. Oglasino has no access to it.
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

Your location is set at the level of your account, not per listing. All your listings are associated with the same region and city as your account; the platform does not support listing goods located in a different region or country than the seller.

**Lawful basis:** performance of a contract (Article 6(1)(b)) — Oglasino is a geographically scoped local classifieds platform; the listings feature cannot function without your location.

### 2.4 Listing data

When you create a listing, we collect and store:

- **Title and description** — written by you, in any of the platform's supported languages. We automatically translate these into the platform's other supported languages (see Section 2.8).
- **Price and currency** — Serbian dinar (RSD) and euro (EUR) are supported on the Serbian portal; euro (EUR) is supported on the Montenegrin portal.
- **"Free" indicator** — whether the listing is for an item you are giving away at no cost (used in the platform's Free Zone).
- **Category and sub-category** — chosen from a fixed taxonomy.
- **Category-specific attributes** — structured fields that vary by category (for example, size, color, year for relevant categories).
- **Photos** — images you upload, stored on Cloudflare R2 (see Section 4 on third parties).
- **Listing status** — whether the listing is active, banned, etc.
- **Counters** — anonymous view count and favorite count for each listing. These counters track totals only; we do not record which individual users viewed your listing.
- **Edit count** — how many times you have edited the listing.

**Lawful basis:** performance of a contract (Article 6(1)(b)) — listings are core platform functionality.

### 2.5 Messages and chat attachments

The platform includes a private messaging feature between users. We store:

- **Messages** — sender, recipient, message content, timestamp, read status. Messages are stored on Google Firestore (see Section 4).
- **Image attachments** — you may attach images to messages. These are stored on Cloudflare R2.
- **Optional product reference** — when you start a conversation with another user by clicking "Send message" on a listing's page, the first message you send in that conversation includes a reference to that listing.
- **Notifications** — when another user sends you a message, follows you, or interacts with your profile, we store a notification on Firestore. Notifications contain only a generic indication (for example, "you have a new message from John Doe"); they do not contain message content previews.

**Access to messages.** The platform's administrators may access message contents to investigate reports of misconduct or violations of our Terms of Use. Outside of moderation work, your messages are visible only to you and the other participant in the conversation. This moderation capability is necessary for the safety of our users and is the basis on which we allow reporting of messages.

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

### 2.7 Favorites

When you save a listing to your favorites, we store a record linking your account to that listing. This is visible only to you.

**Lawful basis:** performance of a contract (Article 6(1)(b)).

### 2.8 Automatic translation of your content

The platform supports four languages: Serbian, English, Russian, and Montenegrin. When you write a listing title, listing description, profile bio, or review comment in one language, we send your text to **OpenAI** (an external service based in the United States) for automatic translation into the platform's other supported languages. The translated versions are stored alongside your original text so that users on the platform can see your content in their preferred language.

OpenAI's API terms commit that data sent through their API is not used to train their models. Oglasino itself does not train any AI model on your content (see Section 6 for our broader commitments on data use).

This processing involves transferring your text outside the European Economic Area to the United States. See Section 5 on international transfers.

**Lawful basis:** performance of a contract (Article 6(1)(b)) — multilingual content is a core feature of the platform.

[LAWYER REVIEW: Auto-translation lawful basis — drafted as contract, on the reasoning that multilingual support is a service feature the user signs up for when joining a multi-language platform. Alternative reasoning is legitimate interest. Either is defensible. Confirm preferred basis.]

### 2.9 Reports

Any user may report another user or a listing for violating our Terms of Use. When a report is filed, we store:

- **Who filed the report** — the reporter's account.
- **Who or what is being reported** — the target user or listing.
- **Report category** — one of: fraud, poor service, rules violation, violence or harassment, inappropriate content, misleading or fake listing, portal rules violation, technical issue, other.
- **Description** — free text from the reporter.
- **Resolution status** — whether an administrator has reviewed the report, and any resolution notes.

Currently, reports are reviewed manually by an administrator. In the future, we plan to introduce automated handling of certain report categories; this Privacy Policy will be updated when that change is made.

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

- Whether you allow non-essential preference cookies.
- Whether you allow in-app notifications.
- Whether you allow transactional emails about your account.
- Whether you allow promotional or marketing emails.
- Whether you allow other users to call your phone number.

Note that even if you turn off "transactional emails," we may still send you emails that are essential to your use of the platform (for example, confirming that your account is scheduled for deletion). Truly optional and promotional emails are gated by the "promotional emails" toggle.

**Lawful basis:** consent (Article 6(1)(a)) for each preference. You can withdraw consent at any time by changing the toggle in your settings; withdrawal does not affect the lawfulness of processing before withdrawal.

### 2.12 Technical data

To keep the platform functioning, secure, and abuse-resistant, we process:

- **IP address** — temporarily logged in our Redis cache for rate-limiting and abuse prevention. IP records are tied to your account when you are logged in. We retain these records for approximately 30 minutes.
- **Application logs** — operational logs of activity on the platform, used for debugging, security monitoring, and incident response.
- **Cookies** — see Section 7.

**Lawful basis:** legitimate interest (Article 6(1)(f)) — security, abuse prevention, and platform reliability.

### 2.13 Account deletion records

When you delete your account, we retain a limited audit record after deletion. See Section 8 (Data retention) for full details.

---

## 3. How we use your data — summary

To make Section 2 easier to navigate, here is a summary of every purpose for which we use your data and the legal basis for each. Detailed disclosures are in Section 2.

| Purpose | Lawful basis |
|---|---|
| Creating and operating your account | Contract |
| Displaying your profile to other users | Contract |
| Publishing your listings to other users | Contract |
| Delivering messages between users | Contract |
| Displaying reviews about you and from you | Contract |
| Saving your favorite listings | Contract |
| Automatically translating your content into other supported languages | Contract |
| Sending you the communications you have opted into | Consent |
| Setting non-essential cookies in your browser | Consent |
| Preventing fraud, abuse, and spam | Legitimate interest |
| Moderating content and enforcing platform rules | Legitimate interest |
| Maintaining the platform's security and reliability | Legitimate interest |
| Investigating reports filed by users | Legitimate interest |
| Retaining a hashed audit record of account deletions | Legitimate interest |

---

## 4. Third parties that process your data

We use a number of external service providers to operate Oglasino. Each of them processes some of your data on our behalf. We never sell your data to anyone (see Section 6 on our specific commitments).

| Provider | Role | What they process | Location |
|---|---|---|---|
| **Google Firebase Authentication** | Authentication provider | Your email, password (held only by Firebase), Google sign-in tokens | EU region (Google Ireland Ltd as EEA counterpart) [LAWYER REVIEW: confirm exact Firebase project region after operator verifies] |
| **Google Firestore** | Real-time database for messages and notifications | Message contents, notifications, sender display names | EU region (same Firebase project) |
| **Cloudflare R2** | Object storage for images | Profile pictures, listing photos, chat attachments, review photos | [LAWYER REVIEW: operator has stated intent to set R2 bucket jurisdiction to EU before launch. If not set, R2 data may be globally distributed. Confirm before publishing.] |
| **Cloudflare** | Content delivery network and edge caching | Cached static content and images | Global edge network |
| **OpenAI** | Automatic translation of user-written content | Listing titles and descriptions, profile bios, review comments | United States |
| **DigitalOcean** | Backend application hosting | All data processed by our backend services | Frankfurt, Germany (fra1 region) |
| **Vercel** | Web frontend hosting (including server-side rendering) | Page requests, session cookies, request bodies | Frankfurt, Germany (fra1 region) |
| **Google reCAPTCHA** | Bot detection on forms | IP address, browser information, mouse and timing data, cookies set by Google | United States |

[LAWYER REVIEW: reCAPTCHA — confirm whether reCAPTCHA's data collection (IP address, mouse movement, Google session cookies including for non-Google-account visitors) is acceptable under legitimate interest, or whether the cookie consent banner should include a separate category requiring explicit consent before reCAPTCHA loads. Industry practice varies; some EU regulators have required explicit consent for reCAPTCHA in past decisions.]

[LAWYER REVIEW: Auto-translation processor — currently OpenAI. The operator may change translation providers in the future. This Privacy Policy names OpenAI specifically; any change will require a Privacy Policy update.]

---

## 5. International transfers

Most of our processing takes place inside the European Economic Area (EEA), which has the data-protection standards required by GDPR.

Two of our processors are based in the United States: **OpenAI** (for content translation) and **Google reCAPTCHA** (for bot detection). When your data is sent to these processors, it leaves the EEA.

We rely on the following safeguards for these transfers:

- **OpenAI** — we transfer data under the European Commission's Standard Contractual Clauses, and OpenAI's adherence to the EU-US Data Privacy Framework (where applicable). [LAWYER REVIEW: confirm OpenAI's current DPF certification status at publication time and confirm the operator has accepted OpenAI's data-processing addendum (DPA) on their OpenAI account. If DPA not yet accepted, this is a launch-blocker.]
- **Google reCAPTCHA** — we rely on Google's adherence to the EU-US Data Privacy Framework and Google's Standard Contractual Clauses.

You can request more information about these safeguards by contacting us at privacy@oglasino.com.

---

## 6. What we will never do with your data

In addition to the obligations we owe you under GDPR, we make two explicit commitments about your data:

**We do not sell your data.** We never have, and we never will. We do not sell user data to advertisers, data brokers, or any third party. We share data with processors only as described in Section 4, and only to the extent necessary to operate the platform.

**We do not train AI on your data.** Oglasino does not use your content — your listings, your messages, your reviews, your profile — to train any AI or machine-learning model. This commitment is specifically about what we do with your data. Separately, our translation provider OpenAI commits in their API terms that data sent through their API is not used to train their models; that is a commitment OpenAI makes about their own use of the data we send them.

If we ever change either of these commitments, we will tell you clearly before the change takes effect, and we will give you the opportunity to delete your account before any new processing begins.

---

## 7. Cookies and similar technologies

The Oglasino website uses cookies and similar browser storage mechanisms (such as localStorage and IndexedDB). We use these for two purposes:

**Strictly necessary** — required for the website to function. These include your authentication session (managed by Firebase Authentication, stored in localStorage and IndexedDB), security tokens (for example, CSRF protection), the cookie consent state itself, and a small number of similar technical cookies.

**Preferences** — used to remember your choices, such as your preferred language, your selected country portal (Serbia or Montenegro), and similar settings.

We do **not** currently use analytics cookies, advertising cookies, or tracking cookies of any kind. If we add analytics in the future (for example, Google Analytics or Vercel Analytics), we will update this Privacy Policy, update our consent banner to add an analytics category, and request your consent before loading any analytics scripts. The current consent banner does not offer this option because we do not currently set any cookies that require it beyond preferences.

This list of cookies may be updated over time as the platform evolves; we will keep this section current.

**Cookie consent.** When you first visit Oglasino, you are asked to accept or decline preference cookies. Strictly necessary cookies cannot be declined because the site cannot function without them. You can change your cookie preferences at any time by clicking the "Manage cookie preferences" link in the website footer, or from your account settings page if you are logged in. When we make a material change to this Privacy Policy or our Terms of Use, the consent banner reappears so that you can review and re-confirm your choices.

[LAWYER REVIEW: Cookie consent banner — confirm that the two-category model (strictly necessary + preferences) plus a footer "Manage cookie preferences" link satisfies your interpretation of GDPR/ePrivacy requirements. The operator has confirmed plans to add the footer link before launch.]

**Third-party scripts.** One third-party script on our site requires specific disclosure: **Google reCAPTCHA**. reCAPTCHA loads on pages with forms (registration, login, listing creation) to prevent automated abuse. It sets its own cookies (including the `_GRECAPTCHA` cookie) and collects information including your IP address, browser environment, mouse movements, and the contents of any Google session cookies you already have. We rely on reCAPTCHA's processing as a legitimate interest in fraud prevention. For more information, see Google's own privacy disclosures at https://policies.google.com/privacy.

---

## 8. How long we keep your data

The default rule is simple: while your account is active, we keep your data for as long as it is needed to provide the service. When you delete your account, we delete your data — with a few specific, time-limited exceptions described below.

### 8.1 While your account is active

Your account data, profile data, listings, listing images, messages, chat attachments, reviews, and favorites are retained for as long as your account is active. Listings do not currently expire automatically; you can delete individual listings at any time from your account.

In the future, we plan to add automatic deletion of listings and chat messages that have been inactive for an extended period. When we introduce this, we will update this Privacy Policy.

### 8.2 IP addresses and operational logs

- **IP addresses in our rate-limiting cache** are retained for approximately 30 minutes.
- **Application logs** are retained for **90 days**, after which they are automatically purged.

### 8.3 When you delete your account

Account deletion on Oglasino is a two-stage process designed to give you a meaningful change-of-mind window and, equally important, to give other users a fair last chance to report you if they believe you have acted in bad faith on the platform.

**Stage 1 — Soft deletion (7 days).** When you request account deletion from your settings, your account immediately enters a "scheduled for deletion" state:

- **Your profile remains publicly visible** and is marked with a "Scheduled for deletion" status badge so that other users can see you have requested deletion. We deliberately keep your profile visible during this window, rather than hiding it, so that users who have interacted with you still have the opportunity to file a report if they have a legitimate concern about your conduct.
- **Your phone number is hidden** during this window, so that you are not contacted while your deletion is pending.
- **Your listings are hidden** from public view.
- **Reviews about you remain visible** so that the context for any potential reports is preserved.
- **You cannot send or receive messages** during the grace period. Existing messages remain accessible to the other participants in your conversations for reporting purposes, with your sender display name changed to "Deleted User," but no new messages can be sent to or from your account.
- **Reports against you remain open.** Any user who interacted with you can still file a report during the grace period.
- You are signed out.

During these 7 days, you can restore your account at any time simply by signing back in. We do this so that you do not lose your data because of an accidental deletion, a moment of frustration, or a change of mind. If you sign in during the grace period, your profile, listings, phone-number visibility, and messaging are all restored.

**Stage 2 — Hard deletion (after 7 days).** On the 8th day, your account is permanently deleted, subject to the postponement exceptions described in Section 8.4. Hard deletion means:

- Your account record, listings, listing images, chat messages, chat attachments, notifications, favorites, and reviews about you are permanently erased from our databases.
- Your profile picture and all images you have uploaded are deleted from our storage.
- Your authentication record at Firebase is deleted.
- Reviews you have left about other users are kept, with your name replaced by "Deleted User" — these reviews reflect the experience of the user you reviewed, and erasing them would erase part of their record. We disclose this to you here so that, when you leave a review, you know that it survives your account deletion (anonymized).

After hard deletion, the only data we retain is a limited audit record described next.

### 8.4 When deletion may be postponed

In limited circumstances, we may delay hard deletion of your account beyond the standard 7-day grace period. This applies only in two cases:

- **Legal investigations.** When a law-enforcement authority (such as the police or a court) asks us to preserve your data in connection with an ongoing investigation, we are legally required to comply and we will postpone deletion for as long as the legal request requires.
- **Internal abuse investigations.** When we have reason to believe your account has been used for serious abuse of the platform — for example, fraud against multiple users — and we are actively investigating, we may postpone deletion to preserve evidence relevant to the investigation.

In both cases, if your deletion is postponed, we will tell you that the deletion has been delayed and, to the extent we are legally permitted to do so, why. You will continue to be signed out and unable to use the account during the postponement; the only effect is that hard deletion is paused. Once the legal or internal investigation concludes, hard deletion proceeds.

**Lawful basis for postponement:** legal obligation (Article 6(1)(c)) for law-enforcement preservation requests, and legitimate interest (Article 6(1)(f)) for internal abuse investigations.

[LAWYER REVIEW: Disclosure of admin postponement — confirm that the transparent model (telling the user their deletion is delayed and why) is preferred over the opaque model (telling the user the deletion is "in progress" without detail). The transparent model respects user rights more strongly but may compromise investigation integrity in some cases. The operator has chosen the transparent model.]

### 8.5 Audit records retained after deletion

After hard deletion, we retain the following limited records:

- **A general deletion audit record** — a hashed version of your user identifier and email (using SHA-256, a one-way hash that cannot be reversed to recover your original email), along with the date your account was deleted and the reason. This is kept for **30 days** after deletion, then automatically purged.
- **Banned-user audit record (only if your account was banned at the time of deletion)** — a hashed version of your email and the reason for the ban. This is used to prevent banned users from immediately re-registering with the same email address. Retained for **12 months**, then automatically purged. [LAWYER REVIEW: 12 months for banned-user retention — confirm proportionality to the fraud-prevention purpose. Industry practice ranges from 6 to 24 months.]

These records contain no personally identifiable information that can be recovered — only one-way hashes. After the retention period expires, even the hash is deleted.

### 8.6 Reports filed by or about users

Currently, reports about users or listings are retained indefinitely so that administrators can detect patterns of repeat abuse. We plan to introduce automatic purging of resolved reports after a fixed period; when we do, we will update this Privacy Policy.

---

## 9. Your rights

Under GDPR and Serbian data-protection law, you have the following rights about your personal data. We support each of them.

**Right of access.** You can request a copy of the personal data we hold about you. Email privacy@oglasino.com with your request. We will respond within one month, as required by GDPR.

**Right to rectification.** You can correct most of your personal data directly in your profile settings. For data you cannot self-edit (such as moderation-related information), email privacy@oglasino.com and we will correct any inaccuracies.

**Right to erasure ("right to be forgotten").** You can delete your account at any time from your account settings; see Section 8.3 for what this involves. If you are unable to use the self-service deletion flow — for example, because your account has been disabled — contact support@oglasino.com to request deletion and we will respond within 30 days as required by GDPR. For all other privacy-related deletion enquiries, email privacy@oglasino.com.

**Right to restriction of processing.** You can ask us to limit how we process your personal data — for example, while you contest a moderation decision. Email privacy@oglasino.com with your request and the reason.

**Right to data portability.** You can request a copy of the personal data you have provided to us, in a structured, commonly used, machine-readable format (such as JSON). This right applies to data you provided directly to us (your profile, listings, messages, reviews); it does not extend to data we derived about you (such as your rating or audit records). Email privacy@oglasino.com to request a portable export. [LAWYER REVIEW: Portability — confirm draft language correctly captures the GDPR scope (data provided by the user, not derived data).]

**Right to object.** You can object to our processing of your data when we rely on legitimate interest. You can object to receiving promotional emails by turning off the corresponding toggle in your settings, and we will always honor that. For other objections, email privacy@oglasino.com and we will assess your objection.

**Right to withdraw consent.** Where we rely on your consent (preference cookies, communication preferences), you can withdraw it at any time:
- Cookie preferences: through the "Manage cookie preferences" link in the website footer or your account settings.
- Communication preferences: through the toggles on your account settings page.

Withdrawing consent does not affect the lawfulness of any processing we did before withdrawal.

**Right to lodge a complaint with a supervisory authority.** If you believe we have not handled your personal data lawfully, you have the right to complain to a data-protection authority. In Serbia, the relevant authority is the Commissioner for Information of Public Importance and Personal Data Protection (Poverenik za informacije od javnog značaja i zaštitu podataka o ličnosti) — https://www.poverenik.rs/. If you are in the EU, you can also complain to the data-protection authority in your country of residence.

**How we respond to requests.** We aim to respond to all data-subject requests within one month of receiving them. In limited cases involving particularly complex requests, GDPR allows us to extend this by up to two further months, in which case we will tell you within the first month.

[LAWYER REVIEW: 30-day response window for email-based requests — the operator has confirmed that the dedicated privacy@oglasino.com mailbox is not yet operational and will be set up before launch as a critical pre-launch action item. The Privacy Policy commits to a one-month response window from publication. The lawyer should confirm this commitment is operationally supported before publication and may wish to review what an actual reasonable response time will look like in practice given the platform's solo operation.]

**Automated decision-making.** Oglasino does not currently make significant decisions about you using purely automated processes. Content validation runs automatically when you create or edit a listing (for example, checking for prohibited content), but all enforcement decisions — banning a listing, disabling an account, resolving a report — are made by a human administrator. If we introduce automated decision-making in the future, we will update this Privacy Policy.

---

## 10. Security

We take reasonable technical and organizational measures to protect your personal data against unauthorized access, alteration, disclosure, and loss. These include:

- Encryption of data in transit (HTTPS for all connections between your browser and our services).
- Encryption of data at rest by our infrastructure providers (Firebase, Firestore, Cloudflare R2, Postgres on DigitalOcean).
- Authentication and authorization controls limiting access to user data.
- Logging and monitoring of administrative access.
- Hashing of stored email addresses and identifiers in our deletion audit records (so that even our retained audit data cannot be reversed to identify deleted users).

No system is perfectly secure, and we cannot guarantee absolute security. If we learn of a personal data breach that creates a risk to your rights, we will notify the relevant supervisory authority and, where required by GDPR, notify you directly.

---

## 11. Children

Oglasino is intended for users aged **18 and over**. We do not knowingly collect personal data from anyone under 18. If you are under 18, please do not register or use the platform. If you believe a minor has registered, please contact privacy@oglasino.com and we will take prompt action to remove the account.

[LAWYER REVIEW: 18+ age restriction — confirm whether Serbian or Montenegrin law requires age verification beyond a self-declared checkbox at registration. Many EU marketplaces use self-declaration; some regulators expect stronger measures.]

---

## 12. Changes to this Privacy Policy

We may update this Privacy Policy from time to time — for example, when we add features, change processors, or in response to changes in the law. When we make a material change, we will:

- Update the "Last updated" date at the top of this document.
- Show a notice on the platform.
- Re-display the cookie consent banner so that you can review your cookie preferences against any changes.

Continued use of the platform after a change means you accept the updated Privacy Policy. If you do not accept the changes, you can delete your account (see Section 8.3).

---

## 13. Contact

For any question about this Privacy Policy or about how Oglasino processes your data, contact us at:

**privacy@oglasino.com**

For all other (non-privacy) questions, please use the platform's general support channels.

---

*End of draft.*
