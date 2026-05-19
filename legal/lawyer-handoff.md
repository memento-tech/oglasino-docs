# Lawyer Handoff Package — Oglasino

> **Purpose.** This document accompanies two pre-lawyer drafts:
>
> 1. `privacy-policy-draft.md`
> 2. `terms-of-use-draft.md`
>
> It is intended for the qualified lawyer who will review and finalize these documents before publication. It gives you the context, the key decisions already made, and the specific questions where your professional judgment is required.
>
> The drafts have been structured to minimize your review time. Every clause where your judgment is specifically required is marked with `[LAWYER REVIEW: ...]` inline. This handoff document consolidates and groups those review items so you can see the scope of decisions at a glance.

**Prepared:** [SET DATE WHEN HANDOFF IS SENT]
**Operator contact:** Igor Stojanović — privacy@oglasino.com

---

## 1. Platform summary (one page)

**What Oglasino is.** Oglasino is an online classifieds platform for buying, selling, and exchanging new and used goods. The platform operates in two country portals — Serbia (`rs`) and Montenegro (`me`) — and is accessible from other countries, including EU member states.

**Business model.** Free at launch. No paid features, no subscriptions, no transactions processed by the platform. Users arrange and complete transactions directly with each other; Oglasino is the venue, not a party to any transaction. Paid features are planned for post-launch but are not part of this review.

**Operator.** Igor Stojanović, individual sole proprietor (not yet incorporated). Address: Bulevar 12. Februar 32, 18000 Niš, Serbia. The operator plans to incorporate a company after launch when the platform shows sustained growth. The Terms of Use contains an Assignment clause to facilitate that transition without re-papering user agreements.

**Users.** Individual consumers (18+). Users register with email + password or Google sign-in. Authentication is handled by Firebase Authentication. The operator confirms no current presence of business users; users are individual consumers transacting peer-to-peer.

**Languages.** Serbian, English, Russian, and Montenegrin. Drafts are produced in English first; Serbian is designated as the controlling language for both documents.

**Technology stack and processors:**

| Processor | Role | Location |
|---|---|---|
| Google Firebase Authentication | User authentication | Google EU region |
| Google Firestore | Real-time database (messages, notifications) | Google EU region |
| Cloudflare R2 | Object storage (images) | Default global; operator setting to EU pre-launch |
| Cloudflare | CDN, edge caching | Global edge network |
| OpenAI (API) | Automatic translation of user content | United States |
| DigitalOcean | Backend hosting | Frankfurt, Germany (fra1) |
| Vercel | Web frontend hosting (with SSR) | Frankfurt, Germany (fra1) |
| Google reCAPTCHA | Bot detection on forms | United States |

**No email service, no SMS, no analytics, no advertising tools, no payment processor, no error-tracking service, no third-party logging service at launch.** All planned for post-launch.

**Key platform features:**

- User accounts with optional phone number and bio
- Listings with title, description, price, category, photos (auto-translated into all four languages via OpenAI)
- Free Zone for items given away at no cost
- Direct messaging between users (Firestore-backed)
- Reviews (bidirectional buyer/seller, 1–5 stars, manual admin approval)
- Reports against users or listings (admin-handled)
- Cookie consent (two-category: necessary + preferences)
- Account deletion (7-day soft-delete with reporting window, then hard delete; hashed audit retained 30 days general / 12 months banned-user)

---

## 2. Key decisions already made

These choices were made by the operator during intake and are reflected in the drafts. We note them here so you can confirm or adjust quickly.

### 2.1 Privacy and data-protection choices

- **Lawful bases** for each processing activity are listed in Section 3 of the Privacy Policy. Most processing relies on **contract** (core platform functionality), with **legitimate interest** for security, moderation, and audit, and **consent** for cookies and marketing-style communications.
- **Retention periods:**
  - Active accounts: data retained while account is active
  - IP addresses in rate-limit cache: ~30 minutes
  - Application logs: 90 days
  - Deletion audit (hashed): 30 days
  - Banned-user audit (hashed email): 12 months
- **No data sale, no AI training on user content** — these are explicit user-facing commitments in both documents.
- **International transfers:** OpenAI and reCAPTCHA (both US). Relying on EU-US Data Privacy Framework and Standard Contractual Clauses.
- **Children:** Platform is 18+. Self-declared age at registration.
- **Auto-translation:** All user content (listings, profile bios, review comments) is automatically translated via OpenAI's API. Disclosed in the Privacy Policy.

### 2.2 Terms of Use choices

- **Governing law:** Serbian law.
- **Jurisdiction:** Exclusive Niš courts, with explicit EU consumer carve-out preserving home-country rights.
- **Mandatory ODR** (EU consumer dispute resolution) link included. No mandatory mediation or arbitration.
- **Termination grounds:** discretionary, proportionate to violation. Notice for minor violations; immediate action for serious violations (fraud, harassment, illegal content).
- **Appeals:** by email, operator's decision is final.
- **Banned-user re-registration prohibited for 12 months** via hashed-email check.
- **Content license:** non-exclusive, royalty-free, limited to platform operation. Explicit exclusions: no sublicensing to third parties (beyond the listed processors), no marketing use of user content, no AI training, no resale.
- **Modifications to Terms:** notice + explicit acceptance for material changes; notice only for non-material changes.
- **Controlling language:** Serbian.

### 2.3 Specific content rules

The Terms permit, with conditions, three categories that warrant your specific review:

- **Hunting weapons and ammunition** for lawful hunting purposes. Operator's intent is to allow this category while excluding combat-grade weapons, weapons in criminal contexts, and any items requiring permits the seller does not hold. See `[LAWYER REVIEW]` flag in Section 6.1 of the Terms.
- **Tobacco and alcohol** for adult users. Permitted.
- **Home-bred and agricultural animals** (dogs, fish, lizards, chickens). Animals restricted under Serbian/Montenegrin law or CITES are prohibited. See `[LAWYER REVIEW]` flag in Section 6.1 of the Terms.

---

## 3. Lawyer review items — consolidated list

Every `[LAWYER REVIEW: ...]` flag from both documents, grouped by theme. Each item is the lawyer's call.

### 3.1 Operator identity and entity transition

- **Privacy Policy Section 1** — Operator is currently an individual sole proprietor with intent to incorporate a company later. Confirm this is acceptable for launch. Confirm the home address as the controller's contact address is acceptable or whether a registered correspondence address should be obtained pre-launch.
- **Terms of Use Section 2** — Same operator-identity flag. Confirm the planned company transition can be handled via Terms update + Assignment clause without requiring re-acceptance by all users.

### 3.2 Lawful basis and processing

- **Privacy Policy Section 2.8** — Auto-translation lawful basis drafted as **contract**, on the reasoning that multilingual support is a platform feature the user signs up for. Alternative reasoning is **legitimate interest**. Confirm preferred basis.

### 3.3 Cookies, consent, and tracking

- **Privacy Policy Section 7** — Cookie banner uses a two-category model (strictly necessary + preferences). Confirm this satisfies your interpretation of GDPR/ePrivacy in Serbia and Montenegro. Operator will add a "Manage cookie preferences" footer link before launch.
- **Privacy Policy Section 4 (third-party table) + Section 7** — **reCAPTCHA**: confirm whether reCAPTCHA's data collection (IP, mouse movement, Google session cookies including for non-Google-account visitors) is acceptable under legitimate interest, or whether the consent banner needs a third category requiring explicit consent before reCAPTCHA loads. Industry practice varies; some EU regulators have required explicit consent in past decisions.

### 3.4 International transfers

- **Privacy Policy Section 5** — **OpenAI**: confirm current DPF certification status at publication time. Confirm the operator has accepted OpenAI's data-processing addendum (DPA) on their OpenAI account. **If DPA not yet accepted, this is a launch-blocker.**

### 3.5 Retention periods

- **Privacy Policy Section 8.4 — Banned-user retention (12 months)** — confirm proportionality to the fraud-prevention purpose. Industry practice ranges 6–24 months; 12 months is the operator's choice.

### 3.6 Deletion mechanics and grace period

- **Privacy Policy Section 8.4 — Admin postponement of deletion (transparent model)** — confirm that telling the user their deletion is delayed and why is preferred over an opaque "in progress" model. The transparent model respects user rights more strongly but may compromise investigation integrity in some cases. The operator has chosen the transparent model.

### 3.7 User rights and response time

- **Privacy Policy Section 9 — 30-day response window for email-based access/portability/rectification requests** — operator has confirmed that privacy@oglasino.com is not yet operational and will be set up before launch as a critical pre-launch action item. The Privacy Policy commits to one-month response per GDPR Article 12. **Confirm the operator has a credible plan to actually meet this commitment** before publication.
- **Privacy Policy Section 9 — Right to data portability** — confirm draft language correctly captures GDPR's scope: data the user *provided* (profile, listings, messages, reviews), not derived data (rating, moderation flags, audit records).

### 3.8 Children

- **Privacy Policy Section 11 + Terms Section 4** — 18+ age restriction. Confirm whether Serbian or Montenegrin law requires age verification beyond a self-declared checkbox. Many EU marketplaces use self-declaration; some regulators expect stronger measures.

### 3.9 Content categories — Terms of Use

- **Terms of Use Section 6.1 — Hunting weapons listings** — confirm Serbian and Montenegrin legal requirements for online weapons advertising. Some jurisdictions require platform operators to take specific measures (age-gating, registered-user-only viewing, license-verification) for weapons listings. If required, the Terms must reflect this and the platform may need a corresponding technical feature.
- **Terms of Use Section 6.1 — Animal listings** — confirm CITES and Serbian/Montenegrin requirements for online animal sales. Microchipping, vaccination, and breeder-registration rules apply to some species; some species are prohibited entirely. Confirm what additional Terms language is needed to manage liability exposure.

### 3.10 Disclaimers and liability

- **Terms of Use Section 12 — As-is disclaimer** — confirm specific wording and the EU consumer-protection adjustments. EU consumer law limits the extent to which a service provider can disclaim warranties against consumers (e.g., implied warranties of merchantability cannot always be disclaimed). Tune accordingly.
- **Terms of Use Section 13 — Liability cap** — recommended approach is to exclude liability to the maximum extent permitted by law, with specific monetary caps set by you, consistent with EU and Serbian consumer-protection law. Confirm carve-outs (death, personal injury, gross negligence, intentional misconduct) cannot be excluded.

### 3.11 Jurisdiction and language

- **Terms of Use Section 15 — Jurisdiction** — exclusive Niš jurisdiction with EU consumer carve-out. Confirm wording correctly preserves EU consumer rights without inviting jurisdiction shopping by sophisticated parties claiming consumer status.
- **Terms of Use Section 17 — Controlling language** — Serbian designated as authoritative. Drafts are in English first; Serbian translation follows. Confirm Serbian as controlling is consistent with operator being a Serbian resident operating under Serbian law, and that this designation will hold up if challenged by a user who only read the English version.

### 3.12 Modifications

- **Terms of Use Section 16 — Modification mechanism** — confirm whether the proposed model (notice + explicit acceptance for material changes; notice only for non-material changes) is consistent with EU consumer-contract rules on unilateral modification of ongoing service contracts. Some Member States require explicit opt-in for any material change.

---

## 4. Pre-launch action items the operator commits to

Independent of your legal review, the operator has committed to completing the following before publication. We list them here so you can confirm none of these are missing or insufficient:

1. **Set up and monitor privacy@oglasino.com.** Required for GDPR right-of-access, rectification, restriction, portability, and objection handling. Mailbox does not yet exist. Auto-responder confirming receipt and 30-day commitment will be configured.
2. **Set up and monitor support@oglasino.com.** Required for general support, complaints, and appeals. Mailbox does not yet exist.
3. **Configure Cloudflare R2 bucket jurisdiction to EU.** Currently default-global; operator will set EU jurisdiction before launch.
4. **Accept OpenAI's data-processing addendum (DPA).** Must be accepted on the operator's OpenAI account at platform.openai.com before any user data is sent to OpenAI's API in production.
5. **Add "Manage cookie preferences" footer link** to enable consent withdrawal for non-logged-in users. Currently the cookie banner is one-time-only with a per-user-settings re-open path; this misses non-logged-in users.

---

## 5. Specific questions for the lawyer

Beyond the consolidated `[LAWYER REVIEW]` flags above, the operator would like your judgment on a small number of broader strategic questions. These are not embedded in the drafts because they are about scope rather than wording.

1. **Is there anything in the platform description (Section 1 of this handoff) that should change how the documents are framed?** For example, does the lack of payment-processing and Oglasino's pure-venue role qualify it for any specific regulatory regime in Serbia or Montenegro that would alter Terms language?

2. **Operator is launching as an individual sole proprietor.** Is there anything in either document that you would draft differently given that the operator is *personally* liable for everything (no corporate veil)? For example, more aggressive liability-exclusion language, more restrictive content-moderation language, or specific insurance recommendations.

3. **The 12-month banned-user retention** is on the high end of industry practice. The operator chose this length because the Balkans classifieds market includes repeat-fraud patterns where a banned user trying to return after 6 months is plausible. If 12 months is hard to defend, what is the maximum defensible retention period?

4. **Hunting weapons listings.** The operator wants to support this category. What is the minimum compliance burden (Terms language + platform features) that allows this category to be live at launch? If full compliance is heavy, is there a path to launch without weapons and add them in a follow-up release once compliance is met?

5. **Animal listings.** Same question. The operator wants to support common pets and agricultural animals. What is the minimum compliance burden?

6. **GDPR Article 30 record of processing activities (ROPA).** As a sole proprietor processing personal data systematically, the operator likely needs to maintain a written ROPA, separate from the Privacy Policy. Is this required for an operator of this scale? If yes, can you draft one as part of the engagement or recommend a template?

7. **GDPR Article 37 — DPO requirement.** Is a Data Protection Officer required for an operation of this scale? Most sole-proprietor platforms do not appoint one. Confirm.

8. **Mandatory cookie banner before any non-essential script loads.** Is the current implementation (which presents the banner on first visit but allows the page to render in the background) compliant under ePrivacy in Serbia and Montenegro? Or does the operator need a hard block before any non-essential cookie or script is set?

9. **Anything missing.** What clauses, disclosures, or features are missing from these drafts that you consider essential and that we have not surfaced?

---

## 6. Out of scope for this review

For completeness, we list what was deliberately not included in this review so you can confirm none of it should have been:

- **Translation into Serbian, English, and Russian.** Drafts are English-only. Translation is a separate workstream.
- **Mobile-app-specific terms.** Oglasino has a mobile app in development but it is not yet ready for launch. App-store terms (Apple, Google Play) and any in-app-purchase mechanics are not covered.
- **GDPR data subject request forms or templates.** Not required by GDPR but useful operationally. Out of scope unless the lawyer recommends including them.
- **Data Processing Agreements with sub-processors.** OpenAI, Cloudflare, DigitalOcean, Vercel, Firebase — each has their own DPA the operator will need to accept. Logging the operator's commitment to accept all DPAs before launch (item 4 of Section 4 above covers OpenAI specifically).
- **Insurance.** Professional liability insurance, cyber-liability insurance — not addressed in these documents. Worth a separate conversation with you if you have recommendations.

---

## 7. Engagement scope (suggested)

The operator is a sole proprietor with limited resources. We propose the following scope for the lawyer engagement, in priority order. The lawyer can recommend a different structure.

**Minimum scope (essential for launch):**

- Review and finalize the Privacy Policy and Terms of Use against Serbian and Montenegrin law plus GDPR.
- Resolve all `[LAWYER REVIEW]` flags and any additional items the lawyer identifies.
- Sign off on the final versions for publication.

**Recommended additions:**

- Brief review of the operator's pre-launch action items (Section 4) to confirm sufficiency.
- Advice on whether ROPA, DPO, or other compliance artifacts are required at this scale.
- Recommendation on insurance.

**Out of initial scope but planned:**

- Translation review (once Serbian and Russian translations are produced).
- Mobile-app terms.
- Any updates triggered by post-launch features (paid subscriptions, analytics activation, etc.).

---

## 8. Final note

Both drafts contain a pre-lawyer notice block at the top. Before publication, that block must be removed and replaced with the final "Last updated" date. The operator will make those final edits after your sign-off.

If you have any questions during your review, please contact the operator at privacy@oglasino.com.

Thank you for the review.

---

*End of handoff package.*
