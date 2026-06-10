# Oglasino vs. Kupujem Prodajem — Strategic Product Analysis

**Prepared:** 2026-06-10
**For:** Igor, founder of Oglasino (pre-launch, Serbia + Montenegro)
**Scope:** Post-launch feature strategy, investor differentiation, unexplored directions
**Method:** All claims about the current market verified by web research where possible; every factual and pricing claim tagged VERIFIED / LIKELY / GUESS. Sources listed at the end as [S1]–[S14].

**What research changed before writing:** KP **has** integrated courier scheduling with COD (shipping alone is not differentiation). KP **confirms it has no payment system** (escrow is). Vinted **does not operate in Serbia** (the playbook is unoccupied). A new Serbian consumer-protection law lands **August 1, 2026** and applies to Oglasino at launch.

Goal tags used throughout: **G1** = decide what to build after launch · **G2** = investor differentiation · **G3** = directions not previously considered.

---

## 0. Verified market picture (basis for everything below)

**KP.** Launched 2008, operated by Quable B.V. (Netherlands); as of August 2023, 5.5M+ active listings and 3M+ registered users; one of the most visited sites in Serbia [S4] — VERIFIED but the user counts are dated. KP has integrated courier scheduling: sellers book pickup in-product, choose who pays, set COD (otkup) with a bank account for payout, and both parties track shipments under Moj KP > Pošiljke; example pricing 340 RSD for a 2 kg parcel, +180 RSD for COD [S1] — VERIFIED. KP states it still has no integrated payment system for listed items, by card or any other method, and warns users that anyone claiming a "new KP payment service" is a scammer [S2] — VERIFIED. KP's own safety guidance tells users never to share card data in KP Poruke and never to open unknown links [S3] — i.e., the phishing-scam problem is real enough that the incumbent's mitigation is a blog post. The pasted cenovnik (dated 07.10.2025) [S14] is treated as VERIFIED as of that date; the live page was behind a maintenance screen when re-checked on 2026-06-10.

**Regulatory (new, important).** A new Consumer Protection Law (ZZP) and amended Trade Law take effect August 1, 2026. Platforms must explain ranking criteria (why one listing is first and another tenth), must clearly mark seller status (trader vs. private person), and fake reviews — including displaying reviews whose authenticity the platform hasn't established — become an offense with fines of 300,000–2,000,000 RSD for legal entities. The trader definition extends to physical persons who sell continuously and in an organized way, giving market inspection a basis to act against de facto traders even on classifieds. Registered traders must show real-time prices and publish them on the National Open Data Portal [S5] — VERIFIED (published 09.06.2026). This applies to **Oglasino** at launch, not just KP.

**Market size.** NBS data: Serbian citizens made 108M+ online purchases in 2025; ~237.4 billion RSD (~€2B) in dinar transactions plus over €1B in euro transactions [S5] — VERIFIED.

**Cautionary tale.** Sasomange launched May 2020 backed by the largest online media group in Serbia (3.5M daily visits across its portals) [S8] and hit ~100,000 registered users and ~500,000 listings within six months [S7] — VERIFIED. A forum thread reports "Sasomange.rs prestaje sa radom 02. juna" [S9] — the shutdown and its year could **not** be confirmed from a primary source; status UNCERTAIN. What's certain: six years of media-conglomerate backing did not displace KP. Features and marketing budget alone didn't move liquidity.

**Vinted gap.** Vinted is currently not available in Serbia [S12] — VERIFIED (a local clone, Vinto.rs, exists precisely because of this, and notably Vinto does no payment intermediation — buyers and sellers settle directly [S12]). Meanwhile Vinted is Europe's leading second-hand marketplace (€1.1B revenue, 2025) and operates Vinted Pay (in-app payments) and Vinted Go (logistics) as core businesses [S13] — VERIFIED. The transactional-marketplace playbook is unoccupied in this market.

**Montenegro.** Patuljak.me self-describes as the largest platform in Montenegro for buying, selling, renting and exchanging [S10] — the claim exists (VERIFIED), its actual scale is a GUESS; the rest of the ME market looks fragmented (small niche sites). Per MUP data from October 2025, ~100k foreigners legally reside in Montenegro: ~24,538 Serbian citizens, 21,153 Russians, 13,396 Turks [S11] — VERIFIED. On a population of ~620k, Russians alone are >3% of residents. The Russian locale is a real asset there. KP appears Serbia-only (site available in Serbian [S4]) — no evidence of an ME portal: LIKELY no ME presence.

**Tax frame for sellers.** Occasional sale of personal used items requires no registration; frequent, organized resale — especially goods acquired for resale — can be treated as a business activity requiring registration and taxes, and unregistered online sellers operate illegally [S6] — VERIFIED. This matters for the dropshipping idea (see §6).

---

## 1–2. Eight strategic features

---

### F1. Safe Deal — in-app payment with buyer protection (escrow) **[G1 later, G2 core]**

**What it does / flow.** Buyer taps "Kupi uz zaštitu" on a listing. Pays by card (later IPS QR) into an account held by a licensed payment partner — not by Oglasino. Seller gets a pre-paid shipping label (same couriers KP-style flows already use). Buyer confirms receipt or money auto-releases 48–72h after delivery confirmation. Disputes: photos in-app, platform arbitrates, partner refunds or releases.

**Problem solved, for whom.** Both sides. Buyer: advance-payment fraud and the dominant Serbian scam vector — fake buyers pushing phishing "payment links," which KP itself can only warn about [S3]. Seller: COD refusal losses (buyer ghosts the courier, seller eats return shipping). It also unlocks remote purchases that COD culture handles badly: higher-value items, ME↔RS, diaspora buying for family.

**Why KP doesn't have it.** Confirmed they don't [S2]. LIKELY reasons: 17 profitable years monetizing visibility, not transactions; payments mean licensing exposure, dispute ops, fraud loss — a different company. Their owner is a small Dutch holding, not an OLX-style group with shared payment infrastructure (LIKELY). Structural reluctance, not oversight.

**Complexity (solo + AI): very large.** Code is the easy half. Risks: (1) finding an NBS-licensed payment institution or bank willing to run marketplace escrow for a zero-revenue startup — the gating item; (2) dispute operations are a human job that lands on the founder; (3) fraud (stolen cards, triangulation) hits immediately; (4) ME is a second currency (EUR) and second regulator.

**KP copy path.** Partner + build + ops: 12–24 months if they decide to (GUESS). Their decision lag is the real moat window.

**Adoption risk.** High. Cash/COD culture; severe fee aversion; and the cold-start paradox — people must trust a no-name platform with money precisely when it has no reputation. Mitigation: launch only after organic transactions exist; seed it free (absorb the PSP fee) for the first N transactions.

**Monetization.** Per-use buyer fee. Anchor: Vinted's buyer-protection fee, ~5% + ~€0.70 fixed (LIKELY — varies by market). Suggested: **3–6% + €0.30–0.70 (35–80 RSD)** — GUESS, anchored to Vinted and to courier COD fees (~180 RSD COD handling [S1] — VERIFIED), so "protection for roughly what COD costs" is the pitch.

**Legal.** Holding client funds = payment institution under Serbia's Law on Payment Services → must partner, never touch the funds. AML/KYC sits with the partner. New ZZP (Aug 2026): trader vs. private seller marking mandatory; B2C purchases through the rails pull consumer-rights questions toward the platform — keep scope C2C first. DSA: accessible to EU users; baseline obligations (contact point, notice-and-action, T&C transparency) apply if targeting the EU — keep it light by not targeting.

**Confidence: medium.** Biggest assumption: Serbian buyers will pay a Vinted-like fee to escape COD risk. Untested in Serbia — both the risk and the opportunity.

---

### F2. Trust stack — chat scam-shield + transaction-verified reviews + optional verified-ID badge **[G1 launch, G2]**

**What it does / flow.** Three pieces. (a) **Scam-shield in chat:** the existing analyzer chain (already screening listings) runs on Firestore messages in real time — block/flag external payment links, known phishing domains, "the courier will send you a link to receive money" patterns; interstitial warning when a counterpart asks to move off-platform. (b) **Verified-deal reviews:** reviews only after a confirmed transaction — QR handshake at in-person meetups, or a tracking number for shipped deals. (c) **Optional ID verification** via a KYC vendor (iDenfy/Veriff-class, €0.50–1.50/check — GUESS) for a "Verifikovan" badge.

**Problem solved, for whom.** Both. The #1 reported consumer pain: the Republic Union of Consumers says most citizens seek help over online-purchase fraud — goods that never arrive, sellers who vanish, scammers using fake addresses and data [S5].

**Why KP doesn't have it.** Their review base is legacy and unverifiable; retrofitting breaks millions of existing reviews (LIKELY). Real-time chat intervention: no public evidence; their published mitigation is education (GUESS that they lack active blocking). The new law works **for** Oglasino: displaying reviews without establishing authenticity is now a finable offense [S5] — this architecture is compliant by design; theirs needs surgery.

**Complexity: small–medium.** Moderation pipeline exists; chat-shield is wiring, not research. Verified reviews are a state machine. ID is a vendor integration. Main risk: false positives annoying legitimate users; warn-first, block-second.

**KP copy path.** Chat filtering: 3–6 months. Verified reviews: structurally painful — 6–12+ months and a migration mess (GUESS).

**Adoption risk.** Low for the shield (invisible until it saves someone). Medium for ID (friction; keep optional, never gate listing). Deeper risk: trust features don't *pull* users who haven't been burned yet — they retain and convert, they don't acquire alone.

**Monetization: free.** Brand, compliance, retention. Don't charge for safety.

**Legal.** ID data is sensitive under ZZPL/GDPR — vendor holds documents; store only a boolean + timestamp. Verified-review mechanics described in T&C (ZZP transparency).

**Confidence: high.** Biggest assumption: "the marketplace that blocks scammers" is a message ordinary users notice. Fraud-complaint data says yes; whether it changes platform choice is untested.

---

### F3. AI Listing Studio + 10-minute migration **[G1 launch]**

**What it does / flow.** Seller takes 2–4 photos. Vision model returns: category (mapped to the deep taxonomy), title, Serbian description, structured attributes (per-category filters), suggested price range (phase 2, once own data exists). Seller confirms/edits, publishes. Second piece: **migration assistant** — seller pastes the text of existing listings (or uploads screenshots); AI parses them into Oglasino listings in bulk. Deliberately paste/screenshot-based, **not** scraping (see Legal).

**Problem solved, for whom.** Sellers. Listing creation is the single biggest seller cost, and a deep taxonomy — a buyer-side advantage — is a seller-side burden. AI fills the structure for free. Migration directly attacks switching cost, the actual reason sellers stay on KP.

**Why KP doesn't have it.** LIKELY they don't yet (no evidence found; GUESS). Incumbent inertia plus an old codebase. Note: the most copyable feature on this list.

**Complexity: medium.** Risks: per-listing inference cost (order of €0.005–0.03/listing with current vision models — LIKELY); Serbian-language quality (good with frontier models; verify with an eval set); miscategorization polluting the taxonomy — confidence threshold + human-confirm step.

**KP copy path.** 3–6 months once they care. The edge is temporary; at cold start, temporary is exactly what's needed.

**Adoption risk.** Low. Worst case sellers ignore the draft. Real risk is quality: a bad auto-description teaches sellers to distrust it after one use.

**Monetization.** Free for individuals (acquisition weapon — contrast with KP charging from 99 RSD to post in paid-visibility groups [S14], VERIFIED as of 10/2025). Later: bulk/API tier for business sellers, €5–15/mo (600–1,800 RSD) — GUESS, anchored to KP Izlog at 1,470 RSD/3mo (VERIFIED [S14]).

**Legal.** Migration: server-side scraping of KP violates their ToS and risks database-rights claims — don't. User-pasted content of the user's own listings is far more defensible (their text, their photos) but worth one lawyer-hour before launch. AI-generated descriptions: platform liability as with any listing — run them through the analyzer chain like everything else.

**Confidence: high.** Biggest assumption: listing friction is a top-3 seller complaint about KP. Strongly LIKELY, unverified.

---

### F4. Serbia Moto depth — fitment, compatibility, history **[G1, G2 wedge story]**

**What it does / flow.** Turn Serbia Moto from "category subsite" into the only place in the region where motorcycle parts search actually works. Buyer enters marka/model/godina (or saves "Moja garaža" bikes) → sees only compatible parts. Sellers listing a part get AI-assisted fitment suggestions from the part number/photos (OEM cross-reference). Add gear-specific structure (helmet sizes, ECE 22.06 rating, boot/glove sizing) and, for whole bikes, a paid history check via a provider like carVertical (operates regionally — LIKELY; affiliate terms GUESS).

**Problem solved, for whom.** Both, in a passionate niche. On a generalist classifieds, parts search is keyword roulette: wrong-fitment purchases, hours of forum cross-checking. KP's taxonomy can't answer "does this fit a 2014 MT-07" (LIKELY — generalist taxonomy).

**Why KP doesn't have it.** Fitment is a data problem, not a feature checkbox; generalists never do it because it doesn't generalize. Misaligned with their model, not oversight.

**Complexity: medium–large — the hard part is data, not code.** Options: license TecDoc-class data (car-centric, thousands of €/yr — GUESS, weak moto coverage); or build a curated compatibility table for the top ~50–100 bikes registered in Serbia, AI-extracted from OEM part catalogs and verified by founder/community. Start curated. Risks: data accuracy (a wrong fitment claim is worse than none); moto fitment data is genuinely sparse.

**KP copy path.** 12+ months *if* they decide a niche justifies it; niche economics say they won't (GUESS). Most defensible feature on this list — the moat is accumulated data + community trust, not code.

**Adoption risk.** Medium. The niche must (a) be underserved enough to move, (b) be reachable — moto FB groups and forums are tight and reachable, which cuts both ways: one bad data incident travels fast.

**Monetization.** Per-use history checks: retail €15–25 (1,800–3,000 RSD), affiliate or markup — GUESS anchored to carVertical single-report pricing (LIKELY ~€20–30). Later: moto shops/dealers subscription €10–30/mo (1,200–3,500 RSD) — GUESS anchored to KP Izlog. Fitment search itself: free, always.

**Legal.** Light. OEM part-number cross-references as facts are generally usable; don't copy proprietary catalog databases wholesale. History-check provider handles its own data licensing.

**Confidence: medium-high.** Biggest assumption: Serbian/regional moto buyers experience fitment pain strongly enough to switch platforms for it. Cheapest assumption on this list to validate before building — ask 20 people in moto groups.

---

### F5. Montenegro + Russian-speaker wedge **[G2 "own a country" story, G3]**

**What it does / flow.** Make Oglasino the default in a market KP ignores. Product work: EUR-native ME experience (ME uses the euro); full-fidelity Russian locale including notifications (RU exists — finish the last mile); ME courier/COD integration (Pošta Crne Gore + whichever private couriers do COD — landscape GUESS, needs a week of calls); RS↔ME cross-border flow with honest customs/VAT guidance for parcels (de-minimis thresholds — GUESS, verify with carriers).

**Problem solved, for whom.** Both, geographically. ME has a self-proclaimed leader in Patuljak.me [S10] and a long tail of tiny sites — fragmented supply, no modern app experience (LIKELY). And a concentrated, underserved community: 21,153 Russian residents per MUP (Oct 2025) [S11], integrating via language similarity but plainly preferring Russian UX — which no local classifieds offers (LIKELY) and Oglasino already built.

**Why KP doesn't have it.** A ~620k-person market is rounding error for them. Rational neglect (LIKELY).

**Complexity: small–medium** (mostly operational/marketing, not engineering). Risk: the real competitor in ME may be Facebook groups, not Patuljak — then the fight is against habit, not a product.

**KP copy path.** They could enter ME in 6–12 months if the market is proven (GUESS). Counter: by then Oglasino is the local default with local couriers wired — small markets reward whoever shows up properly first.

**Adoption risk.** Medium. Small market = small absolute numbers; if FB groups own ME C2C, even a better product moves slowly.

**Monetization.** Standard toolkit later; low ARPU. Value is strategic: "default marketplace of a whole country" is an investor sentence, a liquidity lab, and an RS↔ME corridor nobody else serves.

**Legal.** ME is not EU; separate consumer/data regimes (ME's data law is GDPR-aligned — LIKELY). EUR payments simplify any future Safe Deal there. Extend the privacy-doc diligence to ME specifics.

**Confidence: medium.** Biggest assumption: ME demand isn't already fully absorbed by Facebook groups. Unverified; one week of community immersion answers it.

---

### F6. Demand board done right — structured "Tražim" with auto-matching **[G1, G3 — underrated cold-start tool]**

**What it does / flow.** Buyer posts a structured want: category + filters + price ceiling + location ("Tražim: Yamaha MT-07 2014–2017, do 4.500 €, Vojvodina"). Every new listing is matched against open demands — Elasticsearch percolator queries make this nearly free on this stack — and matching sellers get an instant push: "Neko traži tačno ovo što prodaješ." Buyer gets pinged when supply appears.

**Problem solved, for whom.** Both. Buyers stop refreshing searches; sellers get warm leads the minute they list. KP LIKELY supports wanted ads (the name literally says "kupujem") but as unstructured posts that rot in a feed, not standing matched queries (structure claim: GUESS).

**Why KP doesn't have it (structured).** Demand ads don't monetize under a visibility-promotion model — nobody pays to promote "I want to buy X." Misaligned incentives, not difficulty.

**Complexity: small–medium.** Percolator + push already exist. Risk: empty-marketplace embarrassment — a demand post with zero matches forever. Mitigate with expectation-setting ("we'll notify you") rather than "0 results."

**KP copy path.** 3–6 months technically; incentive misalignment says they won't bother (GUESS).

**Adoption risk.** Medium-high at zero traffic: demands with no supply and vice versa. But note the asymmetry — demand posts are *content the founder can act on*: go find the item from shops/groups. A cold-start tool, not just a feature.

**Monetization.** Free for individuals. Later: businesses pay per qualified lead (€0.30–1 / 35–120 RSD — GUESS, anchored to classifieds lead-gen norms) or subscribe to demand streams in their category.

**Legal.** Trivial. Push consent under ZZPL — machinery exists.

**Confidence: medium.** Biggest assumption: enough buyers will articulate structured demand rather than passively browse.

---

### F7. Local shop catalog feeds (B2C-light) — the honest version of the dropshipping instinct **[G3, G2]**

**What it does / flow.** Registered local retailers (moto shops first) connect an XML/CSV product feed. Their inventory appears as listings, marked **"Prodavac: firma"** with stock auto-sync, deduplication, and the moderation chain on top. Transactions still happen their way (their webshop, their fiscal receipt) — Oglasino is the discovery layer.

**Problem solved, for whom.** Buyers get breadth on day one; the platform gets supply without inventory — likely the real itch behind the dropshipping idea. Sellers (shops) get a free distribution channel.

**Why KP doesn't have it (fully).** KP monetizes business sellers through Obnavljač/Izlog manual flows (VERIFIED [S14]); automated feed ingestion at small-shop scale is unglamorous plumbing they haven't prioritized (GUESS — they may have something private; unverified).

**Complexity: medium.** Feed parsing is boring; risks are taxonomy mapping at scale (AI pipeline helps), stale stock, listing-quality dilution. Cap feed listings per category so C2C isn't drowned.

**KP copy path.** 3–6 months; they have the business relationships. The edge: Oglasino can offer it free as a seeding weapon; for KP it cannibalizes paid products.

**Adoption risk.** Shops are slow and skeptical with zero-traffic platforms; expect to onboard the first ten by hand, in person.

**Monetization.** Free during seeding; later €15–50/mo per shop (1,800–6,000 RSD) — GUESS, anchored between KP Izlog (≈490 RSD/mo at the 12-month tier — VERIFIED math from [S14]) and standalone webshop costs.

**Legal — the new law bites helpfully.** Platforms must clearly mark seller status, and registered traders must keep prices current in real time and publish them on the National Open Data Portal under their own account [S5] — VERIFIED. Feed-synced listings make *both* obligations trivially true for the shops. Compliance becomes a sellable benefit. Liability stays with the shop (whoever issues the fiscal receipt bears responsibility for the purchase [S5] — VERIFIED principle).

**Confidence: medium-high.** Biggest assumption: shops will bother integrating with a zero-traffic platform — hence start in the moto niche where personal onboarding is possible.

---

### F8. Dropshipping system — requested idea, full treatment **[recommendation in §6]**

**What it would do.** Sellers list goods they don't hold; on order, goods ship from a third-party supplier (typically AliExpress-class) to the buyer; the platform orchestrates listing sync, order relay, tracking.

**Problem it solves.** For "sellers": income without inventory. For buyers: essentially nothing — the same item exists cheaper at the source. That asymmetry is the core flaw.

**Why KP doesn't have it.** It's a different business with negative synergy for a trust-based C2C brand (LIKELY their reasoning).

**Complexity: very large.** Supplier APIs, order-state machines across unreliable third parties, refund flows without controlling payments (dropshipping barely works without integrated payments — it would force F1 first), customs tracking, support volume.

**KP copy path.** Irrelevant — they wouldn't want to.

**Adoption risk: severe, and structural.** Buyer side: Temu and AliExpress serve Serbia directly with brutal pricing (LIKELY — widely used; unverified count). Trust side: dropship listings are exactly the inventory that generates "goods not as described / never arrived" complaints — the top consumer-complaint category [S5] — and would poison the trust positioning of F1/F2.

**Monetization.** Theoretically the highest take-rate on this list (commission on every order). Expected value after adoption probability and support cost: poor. No price suggested; any number would be a GUESS stacked on a GUESS.

**Legal: heaviest on this list.** Dropshippers are traders, full stop: selling goods acquired for resale triggers business registration; without registration it's illegal trade, and consumers have no one to claim against [S6] — VERIFIED. The new law's extended trader definition explicitly gives inspectors the basis to pursue physical persons who de facto trade [S5] — VERIFIED. Add import VAT/customs per parcel, product-safety liability, distance-selling withdrawal rights. The infrastructure's primary users would be, by default, non-compliant — with inspections arriving August 1.

**Confidence in this assessment: high.** Biggest assumption: buyer demand for dropshipped goods on a classifieds is near zero given Temu — if Temu's local penetration is overestimated, the case weakens slightly, not structurally.

---

## 3. Two rankings

**By user acquisition (first 6–12 months — what makes someone try Oglasino or leave KP):**

1. **F4 Moto depth** — gives one reachable community a concrete reason KP cannot match
2. **F3 AI listing + migration** — directly attacks switching cost; multiplies every other acquisition effort
3. **F2 Trust stack** — the marketable identity ("platforma koja blokira prevarante")
4. **F5 Montenegro/RU wedge** — high acquisition power, but only inside a small pond
5. **F6 Demand board** — modest pull; real value is operational (see §5)
6. **F1 Safe Deal** — eventually a major buyer magnet; at launch nobody transacts yet, so it pulls no one
7. **F7 Shop feeds** — supply infrastructure, not demand acquisition
8. **F8 Dropshipping** — neutral-to-negative for acquisition

**By revenue per adopted user:**

1. **F1 Safe Deal** — take rate on GMV; the only item that scales with transaction value
2. **F8 Dropshipping** — high conditional take rate, heavily discounted by everything in its writeup
3. **F7 Shop feeds** — recurring B2B subscriptions, the most reliable early revenue
4. **F4 Moto** — per-use checks + shop subscriptions
5. **F5 Montenegro** — standard monetization, small ARPU
6. **F3 AI listing** — freemium upsell only
7. **F6 Demand board** — per-lead, later
8. **F2 Trust stack** — ~zero direct revenue, by design

**Where and why they disagree.** Almost perfectly inverted, and the inversion is the strategy: everything that acquires is free and friction-reducing; everything that earns is transactional infrastructure presupposing liquidity. KP monetizes *attention* (promotion, renewal, visibility — the cenovnik is entirely that). The structural play available to Oglasino is monetizing *transactions* — which only exists once the free layer has created transactions to monetize. Sequencing, not choosing: acquisition features are the spend, F1/F7/F4-monetization is the return, 12–18 months apart.

---

## 4. Build plans — top 3 (F4 acquisition #1, F3 acquisition #2, F1 revenue #1)

**F4 Moto depth.**
MVP (4–8 weeks): curated fitment for the top ~50 bikes on Serbian roads × parts categories; "Moja garaža" (save bikes, one-tap filter); AI-suggested fitment at listing time with seller confirmation. Full: OEM cross-reference search, VIN decode, history-check integration, community fitment corrections. Cut first under pressure: VIN decode, then history checks — the curated table + garage filter IS the product. Working signal: ≥25–30% of moto-parts searches use the fitment filter; listing→contact rate in moto beats site average; fitment-correction reports stay low. Failing / kill: after ~3 months of active seeding (shops onboarded, groups engaged), if active moto-parts listings are still <300–500 (GUESS threshold) and filter usage <10%, the niche isn't biting — fold Moto back into the general site and stop investing. The data table survives either way.

**F3 AI Listing Studio.**
MVP (3–5 weeks): photos → category + title + description + attributes, confirm-screen, Serbian only, analyzer chain as safety net. Full: price suggestions (only once own comps exist — don't fake it earlier), bulk mode, paste/screenshot migration, RU/EN. Cut first: price suggestion (most likely to be wrong, most remembered when wrong), then migration. Working signal: ≥40% of new listings start from an AI draft; draft-to-publish edit distance small and falling; listing completion rate up vs. manual. Failing: <20–25% adoption after 6–8 weeks → a UX problem first (fix entry point) before a value problem; kill only if adoption stays flat after one redesign cycle.

**F1 Safe Deal.**
Pre-req gate, not a date: do not start until the platform organically clears ~50–100 transactions/week (GUESS threshold) — below that, payments integration for an empty room. MVP (8–14 weeks of build + partner negotiation, which may take longer than the build): one courier, card-only, partner-held funds, 48–72h auto-release, manual dispute queue, Serbia only, hard caps (e.g., ≤€300/order) to bound fraud. Full: IPS QR (NBS instant payments — VERIFIED these exist and are widely used; cost advantage over cards LIKELY), multiple couriers, ME/EUR, instant payouts, seller protection. Cut first: ME, then IPS, then instant payouts. Working signal: ≥15–20% attach rate on eligible (shippable, in-range) transactions within 3 months of visibility; dispute rate <3–4%; fraud loss <0.5% of protected GMV (all GUESS thresholds, directionally sane). Failing / park: <10% attach at a ≤5% fee after 3 months → park, keep the partner relationship warm, retry at higher liquidity. Payments parked ≠ payments killed.

---

## 5. Cold start, explicitly

Liquidity truth first: **no feature solves cold start.** Features change the slope; distribution changes the intercept.

**Work at zero/low liquidity (value is per-user, not network-shaped):**
- **F3** — helps the very first seller
- **F2** — protects the very first conversation
- **F7** — manufactures supply without needing users
- **F6** — inverts the problem: a demand post with no matches is an *errand for the founder* (go source it from shops/groups — concierge fulfillment is a legitimate pre-liquidity tactic)
- **F5** — doesn't reduce the liquidity requirement, it shrinks the pond until the requirement is reachable

**Need liquidity to mean anything:** F1 (needs transactions), F8 (needs buyers), F4 (needs *niche* liquidity — but a niche is the only liquidity a solo founder can plausibly manufacture: a few hundred active moto listings is a winnable number; a few hundred thousand general listings is not).

**Top 3 re-ranked under the cold-start constraint:**
1. **F3 AI listing + migration** — supply is the scarce side of every young marketplace; this is the supply pump
2. **F4 Moto** — the wedge designed for cold start: bounded community, reachable channels (FB groups, forums, shops to visit in person), seasonal spikes, plus F7-style shop feeds to pre-fill shelves
3. **F2 Trust stack** — full value from interaction #1, and the brand sentence for every group post

Difference vs. §3: F4 and F3 swap (supply mechanics beat positioning when shelves are empty), and F1 drops out of the top entirely — the *second act*, not the opener.

Distribution note: the channels are unusually concrete — moto FB groups and forums for F4; Russian-speaking Telegram communities in Podgorica/Budva/Bar for F5 (that demographic lives on Telegram — LIKELY); hand-onboarded shops for F7. Budget founder-hours for this like a feature, because it is the feature.

---

## 6. Disagreements

**Dropshipping is wrong for Oglasino — not "hard," wrong.** Four independent reasons, any one sufficient: (1) buyers gain nothing — the same goods exist cheaper at the source, and Temu/AliExpress already serve the market (LIKELY); (2) it manufactures exactly the complaint category regulators and consumers care most about, on a platform whose only viable differentiation is trust; (3) the legal posture is the worst possible for a pre-launch solo founder — an extended trader definition with inspection powers landing August 1 [S5] plus registration/fiscalization obligations the default users would violate [S6]; (4) it fails cold-start logic — dropship supply without buyer traffic is a warehouse with no door. The *instinct* underneath — supply without inventory — is correct. F7 is that instinct with local, registered, fiscally-compliant shops bearing their own liability. Build F7. Don't build F8. If a B2C marketplace play ever makes sense, it's post-traction and looks like Ananas — a different company.

**The "differentiate on features" framing is half wrong.** Evidence: Sasomange reached 100k users and 500k listings in six months [S7] with the largest online media group in the country pushing it daily [S8] — and KP today is undented. Features are copyable; marketing buys visits, not habits. Defensible at this scale: (a) a niche community that considers the platform *theirs* (F4), (b) a geography the incumbent rationally ignores (F5), (c) accumulated structured data (fitment tables, verified-transaction graphs), (d) eventually transaction rails with real switching costs (F1). Investor articulation (goal 2) should not be "KP with better features." It should be: *"The transactional marketplace for the ex-YU second-hand economy — Vinted's playbook in a region Vinted skipped — entered through wedges the incumbent won't contest: the moto vertical and Montenegro."* Vinted built a €1.1B-revenue business on exactly the payments + logistics layer [S13] that KP demonstrably lacks [S2]. A fundable sentence; a feature list isn't.

**Unsolicited third disagreement:** launching two countries, three locales, and a vertical subsite simultaneously, alone, dilutes the only scarce resource (founder weeks) across three cold starts. Pick the first 90-day battleground — recommendation: Serbia Moto, with ME as battleground #2 once Moto shows a pulse — and let the rest exist in maintenance mode. Infrastructure for all three can stay; founder attention can't split three ways.

---

## 7. Caveats — guesses, missing data, user questions

**Everything tagged GUESS, collected:** KP's lack of real-time chat-link blocking; KP's demand-ad structure; KP's copy timelines (all); Patuljak.me's actual size; the ME courier/COD landscape; ME parcel customs thresholds; TecDoc-class licensing costs and moto coverage; carVertical's Serbian availability, pricing and affiliate terms; Vinted's exact fee structure (LIKELY-grade); Temu's Serbian penetration; KYC per-check pricing; every kill-threshold number in §4; every proposed price range; Telegram as the dominant RU-émigré channel. UNCERTAIN: Sasomange's current operating status. Note: the KP cenovnik is October 2025 and the live page was behind a maintenance screen on 2026-06-10 — re-pull before quoting prices to investors.

**Data to obtain before committing months or money:**
- KP's moto-parts category: listing counts, turnover, and 10–20 real search sessions watching someone fail to find a part (validates the F4 bet for the cost of coffee)
- Courier COD refusal rates in Serbia (ask D Express/AKS/BEX sales reps — they know; sizes the F1 seller-side pain)
- Which NBS-licensed payment institutions or banks will discuss marketplace escrow for a pre-revenue platform — names and terms (gates F1 harder than any code)
- ME second-hand volume actually flowing through FB groups vs. Patuljak (one week of lurking)
- A 50-person willingness-to-pay probe on buyer protection at 3% vs 5% vs "never"

**The 3 questions most worth asking real Serbian/Montenegrin users:**
1. *"Ispričaj mi poslednju kupoprodaju preko KP koja je propala ili zamalo propala — šta se tačno desilo, i šta si posle promenio u načinu kako kupuješ/prodaješ?"* — surfaces the true ranking of pains (scam fear vs. ghosting vs. COD refusals vs. fees).
2. To moto people: *"Kad si poslednji put kupovao deo — gde si prvo tražio, koliko ti je trebalo da budeš siguran da odgovara tvom modelu, i da li si nekad kupio pogrešan?"* — directly validates or kills F4 before a line of code.
3. In Montenegro (Serbian and Russian versions): *"Gde danas prodaješ/kupuješ polovne stvari — i šta te kod toga najviše nervira?"* — reveals whether the ME opponent is Patuljak, Facebook, or nobody, which determines whether F5 is a wedge or a mirage.

**Closing honesty:** the largest unknown isn't any feature — it's whether *trust* or *price/liquidity* dominates platform choice for Serbian users. Every strategic bet above leans toward trust. If user interviews come back "I'll tolerate scam risk for the bigger selection," re-weight toward F3 + F7 (supply mechanics) and away from F1/F2. The interviews cost a week. The wrong bet costs a year.

---

## Sources

- **[S1]** KP blog — "Zakazivanje kurira": https://blog.kupujemprodajem.com/info/zakazivanje-kurira/
- **[S2]** KP help center — "Da li KupujemProdajem ima svoj sistem za plaćanje?": https://blog.kupujemprodajem.com/helpcentar/bezbedna-kupoprodaja/da-li-kupujemprodajem-ima-svoj-sistem-za-placanje/
- **[S3]** KP blog — "Kako da bezbedno kupujete i prodajete preko KP": https://blog.kupujemprodajem.com/kako-da/kako-da-bezbedno-kupujete-i-prodajete-preko-kp/
- **[S4]** Wikipedia — KupujemProdajem: https://en.wikipedia.org/wiki/KupujemProdajem
- **[S5]** Nova Ekonomija (09.06.2026) — "Uvode se oštrija pravila za onlajn platforme poput Ananasa, Limunda i KP": https://novaekonomija.rs/ostalo/uvode-se-ostrija-pravila-za-onlajn-platforme-poput-ananasa-limunda-i-kp-inspekcija-i-kod-gradjana-prodavaca
- **[S6]** Biznis.rs (05/2025) — "Da li se plaća porez na prodaju polovne robe preko interneta?": https://biznis.rs/preduzetnik/porezi/da-li-se-placa-porez-na-prodaju-polovne-robe-preko-interneta/
- **[S7]** Netokracija (11/2020) — Sasomange 100k users / 500k listings: https://www.netokracija.rs/sasomange-oglasnik-176495
- **[S8]** Mondo (05/2020) — Sasomange launch: https://mondo.rs/Info/Ekonomija/a1320895/Sasomange-online-kupoprodaja-sasomange.rs.html
- **[S9]** Benchmark forum — Sasomange shutdown report (unconfirmed): https://forum.benchmark.rs/threads/sasomange-rs-by-adria-media-group.463839/
- **[S10]** Patuljak.me — self-description: http://patuljak.me/
- **[S11]** Investitor.me (17.10.2025) — foreigners in Montenegro, MUP data: https://investitor.me/2025/10/17/skoro-100-000-stranaca-zivi-u-crnoj-gori-najvise-iz-srbije-rusije-i-turske-broj-rusa-opao-za-gotovo-6-000/
- **[S12]** Vinto.rs — "Vinted Srbija" availability page: https://vinto.rs/vinted-srbija
- **[S13]** Wikipedia — Vinted: https://en.wikipedia.org/wiki/Vinted
- **[S14]** KP cenovnik, user-provided copy dated 07.10.2025 (live page unreachable on 2026-06-10): https://www.kupujemprodajem.com/cenovnik