# F2 Deep Dive — Trust Stack

**Prepared:** 2026-06-10 · **Companion to:** oglasino-strategic-analysis.md, oglasino-f3-ai-listing-studio.md, oglasino-f4-moto-fitment.md
**Status in sequence:** Phase 1 is launch hygiene (ships before/with launch, in parallel with production push, not after F3/F4). Phases 2–3 interleave with F3/F4 builds.

## 0. Reframe

The original F2 was three pieces; at spec level it is **six components on one shared philosophy**: make the scam economically pointless on Oglasino, and make every trust claim verifiable by architecture. KP's trust model is reputation-by-volume plus blog-post education — their official mitigation for card-phishing in KP Poruke is telling users to be careful and not open unknown links. Oglasino's can be enforcement in the message path, buildable because a server-side analyzer chain and a dedicated Firestore-rules repo already exist.

Framing that elevates the whole stack: the dominant Serbian scam (fake "courier payment link") works because **no official transaction path exists** on classifieds — every payment/shipping arrangement happens in free-text chat, so a fake link looks plausible. Once official courier booking (shipping integration) exists, the shield gets a killer line: *"Oglasino ima zvanično zakazivanje kurira u aplikaciji — sve van toga je prevara."* Trust stack and shipping integration reinforce each other; build them as one program.

## 1. Threat model

Ranked by expected frequency/damage for a Serbian classifieds (ranks LIKELY; pattern #1 confirmed real by KP's own published warnings about payment links, card data in messages, and fake "KP payment services"):

1. **Fake buyer → phishing payout link.** "Kupujem odmah, kurirska služba će vam isplatiti novac, evo link da primite uplatu" → lookalike courier/post/bank page → seller enters card + 3D code → drained. Variants: link sent as image/screenshot to evade text filters; QR code image; shortened URLs.
2. **Off-platform pull.** "Javi se na Viber/WhatsApp" within first messages — precursor to most scams (exits the moderated channel).
3. **Advance-payment fake seller.** Demands uplata unapred / Western Union / crypto, never ships.
4. **Bait listings.** Cloned photos from real listings, too-good price; harvests victims into flows 1/3.
5. **COD games.** Buyer-side: fake "carina/doplata" links while awaiting a parcel. Seller-side: ghosting COD parcels (cost abuse).
6. **Account takeover.** Phished reputable accounts reused to run flows 1–4.
7. **Review fraud.** Reciprocal fake-positive rings; legal exposure under the new ZZP: displaying reviews without establishing authenticity is a finable offense (300,000–2,000,000 RSD for legal entities).

## 2. Component A — Chat Scam-Shield (threats 1, 2, 3, 5)

### Detection layers, in execution order
- **L1 — URL policy (synchronous, cheap).** Extract URLs (incl. obfuscated: spaced domains, Cyrillic/punycode lookalikes, shorteners). Policy: allow-list (own domains; courier tracking domains once shipping exists) → known-bad blocklist (curated + community-reported, Redis-cached) → heuristic risk (shortener, raw IP, lookalike distance to courier/bank/post brands, domain age via RDAP later) → default: deliver-with-warning for established accounts; hold-for-review for accounts younger than N days with zero completed deals.
- **L2 — Pattern rules (synchronous).** Serbian phrase pack: "kurirska služba će (vam|ti) isplatiti", "primite uplatu", "unesite (podatke|broj) kartice", "potvrdite prijem novca", card/IBAN regexes, Viber/WhatsApp pulls in first 3 messages. Implemented as a **chat profile of the existing analyzer chain** (which already detects links/contact info in listings). Phrase pack is data (DB/config), not code — scammers iterate weekly; response must be a config push, not a deploy.
- **L3 — Behavioral velocity.** Redis counters: new conversations/hour, identical-first-message hash fan-out, link-message rate per account-day, account age × activity shape. Trips step-up actions (hold-all-links for that account; admin queue entry).
- **L4 — LLM conversation classifier (async, Phase 3).** Only when L1–L3 raise risk: ship the conversation window to a cheap text model with a fixed label schema (payout-link scam / advance-fee / off-platform pull / benign). Cost lands only on suspicious traffic (a few % of conversations).
- **L5 — Image OCR (async, Phase 3).** OCR chat images; extracted text re-enters L1/L2. Closes the screenshot-evasion hole, which appears within weeks of L1 working.

### Enforcement ladder
log-only → **warn recipient inline** ("⚠️ Ova poruka sadrži link koji liči na prevaru. Kurirske službe nikada ne 'isplaćuju' novac preko linkova.") → block delivery with sender notice → freeze conversation + admin queue → suspend (existing 12-month re-registration ban as terminal step).
Principles: **warn the victim even when delivering** (the interstitial at the moment of risk is the highest-value education available); **never reveal which rule fired** to the sender — explanatory errors train scammers.

### Architecture — key design decision
Chat is Firestore-backed; the dedicated rules repo suggests direct client writes. Two viable patterns:
- *Pattern 1 — backend relay:* messages route through the Spring Boot API → analyzer synchronously → write to Firestore. True pre-delivery blocking, one moderation codepath; cost: chat latency/availability moves onto the backend; bigger architectural change.
- *Pattern 2 — pending-visible (recommended if clients write directly today):* Firestore rules require new messages created with `moderation: "pending"`; recipient queries only `moderation == "approved"`; sender sees own message immediately; a trigger/worker runs L1–L3 and flips to `approved`/`blocked`/`flagged` in ~100–500 ms. Pre-delivery scanning with no perceptible UX change; the enforcement state machine lives in data and is enforceable by the rules repo.

Either way, **detection logic stays in the backend service** (one analyzer, two profiles: listings + chat). Firestore rules enforce *state visibility*; the backend decides *state*.

### False positives
"Ovo nije prevara" feedback on every warning → admin review → allow-list/pattern fix; per-conversation relaxation after a confirmed deal between the pair; FP appeal rate is a first-class metric (<2% of warnings — GUESS target).

## 3. Component B — Transaction-verified reviews (threat 7 + ZZP compliance)

Reviews + approve/disapprove moderation already exist. The new part is the **verification gate**: a review can only be created against a `confirmed_transaction` record. Three confirmation paths, in trust order:

1. **Shipment-verified (strongest):** shipment entity reaching `DELIVERED` auto-creates the confirmed transaction. Zero user effort, collusion-resistant. Formal dependency on the shipping integration.
2. **QR handshake (in-person):** "Potvrdi kupoprodaju" in conversation → short-lived signed QR (listing ID + party IDs + nonce, 5-min TTL) → other party scans → both gain review rights; listing offers "označi kao prodato". Offline fallback: 6-digit code.
3. **Mutual confirmation (weakest, still gated):** both parties independently tap "obavljeno". Collusion-possible → lower internal trust weight in anti-gaming scoring, same public badge (no user-facing tiers).

**Decision to make now, with zero legacy:** reviews are verified-only from day one. Public framing: every review on Oglasino says "iz potvrđene kupoprodaje" because there is no other kind. Also the clean answer to the ZZP authenticity requirement — compliance by architecture, one paragraph in T&C.

**Anti-gaming:** one review per pair per listing; pair-frequency caps; reciprocal-positive cluster checks (Phase 3; plain SQL first); new-account review weight ramps with age; device/IP clustering reused from L3.

**Cold-start tradeoff:** gating lowers early review volume. Compensate with *computed* signals needing no reviews: response rate, median response time, completed-deal count, account age — on the trust card. Unfakeable by friends, free to compute.

## 4. Component C — Identity tiers (threats 1, 3, 6 + ZZP seller-status duty)

- **Tier 0 — phone verification, mandatory before first listing or first outbound message.** SMS OTP. The phone number is the practical identity layer regionally; per-number cost makes throwaway-account farming expensive. Kills more spam/scam volume than anything else here (LIKELY). Cost ~€0.01–0.05/SMS (LIKELY); rate-limit + voice fallback.
- **Tier 1 — verified person (optional badge).** KYC vendor (iDenfy/Veriff/Sumsub-class; RS/ME document support — LIKELY; €0.50–1.50/check — GUESS). Vendor holds documents; store only `verified: true` + vendor ref + timestamp. Incentives: badge, higher limits, faster moderation lane. Never required to browse/buy.
- **Tier 2 — verified business ("Firma").** Validate matični broj/PIB against APR (RS) and CRPS (ME). Doubles as the machinery for the new legal duty to clearly mark seller status (trader vs. private person) — required by August 1 regardless; building it as verification yields compliance + trust signal for one effort.

## 5. Component D — Listing-level trust signals (threat 4)

**Trust card** on every listing: account age, phone/ID/business badges, completed-deal count, response rate, verified-review summary. **Duplicate-photo detection:** perceptual hash (pHash) of uploads against own corpus — catches cloned bait listings cheaply; external reverse-image search Phase 3+. **Price-anomaly chip:** with ~months of category data, far-below-median listings get a buyer caution chip ("cena znatno ispod proseka — proverite pažljivo") + moderation entry; pre-data, a static heuristic for high-fraud categories (phones, consoles, bikes).

## 6. Component E — Account security (threat 6)

New-device login notification (push/email — infra exists); session list + revoke; password-breach check via k-anonymity HIBP API (free); optional TOTP 2FA Phase 3. Rationale: badges and reviews are stealable capital; the stack is incomplete if accounts are soft.

## 7. Component F — Education + trust ops

One-time first-chat interstitial (three rules, ten seconds); category-aware tips (vehicles: "prepis kod MUP-a"; high-value: public place); Safety Center page doubling as DSA/ZZP transparency posture. Admin: **trust queue** in the existing panel — flagged conversations with context, one-tap actions (dismiss/warn/block/ban), rule-fire and FP analytics, blocklist/phrase-pack CRUD so the response loop is config, not deploys.

## 8. Phasing, effort (solo + AI), cut order

- **Phase 1 — launch hygiene (2–3 weeks):** L1+L2 in pending-visible pipeline, recipient warnings, report button, mandatory phone verification, trust card v1 (computed signals), first-chat interstitial, trust queue v1. Ships before/with launch.
- **Phase 2 (3–4 weeks):** QR + mutual deal confirmation, verified-review gate live, business verification (APR/CRPS) → ZZP trader marking, L3 velocity, pHash duplicates.
- **Phase 3 (3–4 weeks, post-traction):** KYC Tier 1, L4 LLM classifier, L5 OCR, data-driven price anomaly, 2FA, review-graph analytics.
- **Lands with shipping integration:** shipment-verified confirmations.

Cut order under pressure: Phase 3 entirely → pHash → business verification (returns before **August 1** as a legal item) → QR (mutual-confirm carries reviews briefly). **Never cut:** L1/L2 + warnings + phone verification — that trio is the brand.

## 9. Metrics

Scam reports per 1,000 conversations (trend down); warning→proceeded-anyway rate; blocked-message FP appeal rate <2% (GUESS); median report→action time <4h (GUESS); % new users phone-verified (=100% by design); % of completed deals leaving a verified review ≥30% (GUESS); Tier-1 badge adoption ≥3–5% of active sellers in 6 months, else park the KYC vendor (GUESS). The stack as a whole has no kill criterion — hygiene and law — but each Phase-3 piece individually justifies itself or gets parked.

## 10. Legal

ZZPL/GDPR: phone numbers and verification artifacts are personal data (lawful basis: contract performance + legitimate interest for anti-fraud — document it); KYC vendor as processor, minimal local storage; message scanning disclosed in Privacy Policy and T&C ("automatske provere radi bezbednosti") — add to the open privacy-doc workstream alongside new processors (SMS provider, KYC vendor). ZZP: review-verification mechanism described in T&C; trader marking live by August 1. Moderation liability: T&C reserve the right to filter/block; log enforcement reasons internally, keep them generic externally. The report button doubles as notice-and-action if EU/DSA posture ever matters.

## 11. Repo mapping

**oglasino-backend** (~55%): analyzer chat-profile, URL/pattern/velocity services, moderation verdict API, confirmed-transaction + review-gate domain, phone OTP, APR/CRPS lookups, trust-queue endpoints, phrase-pack/blocklist CRUD.
**oglasino-firestore-rules**: the pending-visible message state machine — rules enforce visibility states, never detection logic.
**oglasino-web / oglasino-expo**: warning banners, report flow, QR confirm, badges, trust card, interstitials, safety center.
**oglasino-router**: optional edge rate limiting on OTP endpoints.
**oglasino-docs**: threat model as a living document + decision records (e.g., *reviews verified-only from day one; rejected: open reviews with a verified tier — legacy pollution and ZZP exposure*).