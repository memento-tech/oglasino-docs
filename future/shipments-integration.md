# Shipments Integration — Specification

**Prepared:** 2026-06-10 · **Companion to:** oglasino-strategic-analysis.md, oglasino-f2-trust-stack.md
**Constraint honored throughout:** Oglasino never touches money (escrow/payments deferred by founder decision).
**Verification tags:** VERIFIED / LIKELY / GUESS as in the master analysis. Sources at the end.

---

## 1. Scope, constraints, and the shaping decision

**Goal:** structured courier shipping for C2C deals — booking, COD, tracking, notifications — Serbia first, Montenegro second, RS↔ME corridor third.

**The money model (KP-proven, platform-bypassing).** KP's documented flow [S1]: seller schedules the courier in-product, chooses who pays delivery, sets the COD amount and the bank account to which the courier pays out; delivery is paid in cash directly to the courier (at pickup if sender pays, at delivery if recipient pays); COD payout to the seller averages 2–3 working days; both parties get a chat confirmation and track on a dedicated page; package limits 20 kg and 60×80×100 cm — VERIFIED. Three money flows, zero through the platform:

1. **Delivery fee:** cash, user → courier driver.
2. **COD (otkupnina):** buyer → courier → seller's bank account, paid out by the courier.
3. **Platform fee:** none at launch. Only escrow-free monetization later: **courier partner rebate** per shipment, invoiced B2B monthly (program existence LIKELY; amounts GUESS). Never a consumer charge.

**Non-goals:** payments/escrow, same-day gig couriers, freight/oversize, own insurance products.

## 2. Market map (researched 2026-06-10)

**Serbia — Partner #1 candidates:**
- **D Express** — national, ~1,200 employees, 750+ vehicles, today-for-tomorrow, same-day in larger cities [S2] (VERIFIED scale); dedicated eCommerce program (VERIFIED page); partner API for contracted clients — LIKELY (widely integrated; tracking supported by aggregators like AfterShip — VERIFIED). Default strongest candidate.
- **City Express** — urban specialization, precise delivery time windows, modern app/payments [S2]; developer forum reports e-commerce integrations and an officially documented REST API [S3] — LIKELY most developer-friendly; urban-only coverage is a real C2C limitation.
- **BEX** — competes on very low prices [S2]; API status GUESS.
- **AKS** — national, public price list; **no COD on international shipments** [S4] — VERIFIED (matters for Phase 3).
- **Post Express (Pošta Srbije)** — best rural coverage; WebExpress code-based senders' flow without printers — LIKELY; API depth GUESS.

**Montenegro:** **Pošta Crne Gore Post Express** — pickup at sender's address, online addressing, electronic tracking, COD collected at delivery, 24h delivery within ME, volume discounts 5% (200–500/mo) and 10% (500+/mo) [S5] — VERIFIED. Pricing anchor: dedicated online-order tariff, next-day, **€2.00 up to 5 kg** [S6] — VERIFIED. Small private couriers exist (PostExpress.me, BB Post Express — LIKELY local, no APIs assumed).

**Cross-border RS↔ME:** **PostPak** — COD parcel exchange among regional postal operators (Pošta Srbije, Pošta CG, BH posts, HP Mostar): parcels ≤30 kg, COD value ≤€999, payout to sender via international postal money order (PostCash), barcode tracking [S7] — VERIFIED. The corridor rail exists; it lacks a product wrapper.

**Build-vs-buy:** open multi-carrier frameworks (Karrio-class) have no Serbian adapters — the adapter would be written regardless → keep a thin own adapter layer in Spring Boot. Decision-record material.

## 3. Phasing

**Phase 0 — structured shipping, no contract (1–1.5 weeks; ship with launch).** Generates demand data = negotiation leverage ("X booking intents/week").
- `shipping_available` boolean on listings → ES field → "Sa dostavom" filter chip.
- **Address card** in chat (§6): structured, private, replaces free-text address dictation.
- Courier picker: static list, maintained price tables, deep links; seller arranges pickup.
- **Tracking-number field** on the conversation → deep links to courier public tracking (allow-listed in the trust shield).
- Mutual "poslato / primljeno" self-reporting → feeds trust-stack mutual confirmation until API-verified delivery exists.

**Phase 1 — single courier, full API, Serbia (4–6 weeks build; contract negotiation in parallel — likely the long pole).** Sections 4–9.

**Phase 2 — Montenegro (2–3 weeks + negotiation).** Pošta CG Post Express business account with volume discounts; API availability unknown (GUESS) — fallback: their online-addressing portal in a semi-automated flow (pre-filled data, manual submission).

**Phase 3 — RS↔ME corridor (≈1 week as a guided flow).** PostPak wrapped as a *guided* product: platform generates address card, customs content description, step-by-step instructions, tracking link; seller walks the parcel into a post office. Cross-border stays semi-manual until volume argues otherwise. Recipient-side customs/de-minimis: GUESS — resolve with the posts during Phase 2 talks.

## 4. Functional spec — flows

**Actors:** seller (sender), buyer (recipient), courier (external), admin.

**Entry points:** (a) persistent "Dostava" action in the conversation toolbar; (b) after buyer shares an address card, seller gets primary CTA "Zakaži kurira"; (c) listing badge/filter "Sa dostavom".

**Happy path:**
1. Price agreed in chat.
2. Buyer taps "Pošalji adresu" (or seller requests) → structured address card in conversation.
3. Seller booking form: pickup address (saved/prefilled, editable); recipient (auto-filled from card); package — weight presets S/M/L/XL mapped to kg brackets + custom kg, optional dimensions; contents description (courier requirement; feeds prohibited-items gate); COD toggle + amount (prefilled with agreed price, editable); **seller payout account** (tekući račun, mod-97 validated, encrypted, reusable); who pays delivery (pošiljalac/primalac); pickup date window; courier note.
4. Pre-submit interstitial (expectation-setters): delivery paid **in cash to the courier**; enter real weight — the courier reweighs and folds extra cost into the delivery price [S1]; **if the buyer refuses the COD parcel, return shipping is charged to the sender** (prevents the worst support category).
5. Submit → adapter creates shipment → **system message in the Firestore conversation** with tracking code + timeline link; pushes per notification matrix.
6. Statuses via webhook/polling → timeline → `DELIVERED` → **confirmed_transaction (trust-stack interlock), review prompts.**

**Unhappy paths:** delivery attempt failed (retry loop, buyer nudged); **REFUSED** (seller notified with return info; internal buyer COD-refusal counter increments — trust-stack L3 signal); RETURNED; cancellation (until `PICKED_UP`, subject to courier cutoff); damage/loss (status → claim guidance; claim runs seller↔courier; platform supplies the paper trail); address correction pre-pickup only.

## 5. Status model

Internal enum: `DRAFT → REQUESTED → PICKUP_SCHEDULED → PICKED_UP → IN_TRANSIT → OUT_FOR_DELIVERY → DELIVERED | DELIVERY_FAILED (→ retry) | REFUSED → RETURNING → RETURNED | CANCELLED | CLAIM`.

Per-courier codes map to the enum via a **data-driven mapping table** (DB, not code). Unknown incoming code → `IN_TRANSIT` + log + weekly alert for mapping maintenance. Every transition appends to `shipment_events` (immutable, raw payload retained).

**Notification matrix** (push + mirrored localized system chat message; one source of truth):
- Seller: REQUESTED-confirm, PICKED_UP, DELIVERED (+ "isplata otkupnine obično 2–3 radna dana" per partner SLA), REFUSED/RETURNED.
- Buyer: PICKUP_SCHEDULED ("paket ti je na putu"), OUT_FOR_DELIVERY, DELIVERED (+ review prompt).

## 6. Address subsystem (deceptively the hardest part)

Serbian courier APIs commonly require **their own registry IDs for town/municipality/street**; free text gets rejected or mis-routed.
- `syncReferenceData()` per adapter: pull courier registries into local tables on schedule; Redis-cache lookups.
- Entry UX: city autocomplete → street autocomplete (courier registry) → number/floor free; graceful free-text fallback with "kurir može tražiti pojašnjenje" notice (cutting normalization raises failed-pickup rates — cut last).
- **Address card** entity: name, phone, normalized address, note. Shared **only on explicit user action**; rendered as a card to the counterpart; masked in admin and logs; encrypted at rest.

## 7. Backend design (Spring Boot)

- Module `shipping/`: entities `Shipment`, `ShipmentEvent`, `AddressCard`, `PayoutAccountRef`; one **`CourierAdapter` interface**: `createShipment`, `cancel`, `fetchStatuses(batch)` or `handleWebhook(payload)`, `quoteRate (optional)`, `syncReferenceData`. Partner #1 = first implementation; Pošta CG and PostPak-guided are additional adapters or a manual-mode adapter.
- **Reliability:** idempotency key on create (`conversation_id + listing_id + attempt`); retries with backoff; circuit breaker around courier calls (courier outage must not queue-bomb the backend); **outbox pattern** for status events → notification fan-out (a push failure never loses a state change).
- **Polling fallback** if no webhooks: `@Scheduled` over non-terminal shipments, 15–30 min cadence, tighter near expected delivery, jittered batches.
- **Ops alerting — reuse existing Telegram/email alert infra:** REQUESTED unconfirmed >24h; PICKED_UP without movement >5 days; unknown-status spikes; webhook signature failures. Admin shipments view: search by tracking/user, force-refresh, audited manual status override, claim notes.

## 8. Security, privacy, legal

- **Data classes:** addresses, phones, and the seller's bank account number (the most sensitive new object). App-layer AES-GCM or pgcrypto at rest; masked rendering (`***-***-1234`); excluded from logs; **retention:** purge address + account fields N days (e.g., 90 — GUESS, set with lawyer) after terminal status; keep anonymized skeleton for stats.
- **Privacy policy:** courier becomes a **recipient/processor** — add to the processor list in the open privacy-doc workstream; DPA with the courier (negotiation item).
- **Platform role:** booking agent. T&C: Oglasino ne pruža poštanske usluge; carriage contract arises seller↔courier (licensed postal operator); damage/loss claims to the courier; platform supplies records. LIKELY outside postal-licensing scope — one lawyer-hour against the Zakon o poštanskim uslugama.
- **ZZP:** pure C2C is clean. F7 business sellers carry trader obligations — registered sellers shipping COD must include an invoice to the buyer in the parcel, with courier collecting to the business account [S8] — surface as guidance on business accounts.
- **Prohibited items:** per-courier list (dangerous goods, weapons, cash, perishables [S4]) → pre-booking checkbox gate + auto-block for already-restricted categories.

## 9. Trust-stack interlock

`DELIVERED` ⇒ `confirmed_transaction` ⇒ verified review (strongest path). Courier tracking domains enter the chat shield allow-list; everything else claiming to be a courier link gets the warning. Shield copy becomes literally true the day Phase 0 ships: *"Zakazivanje kurira postoji samo u aplikaciji — svaki 'kurirski link' u porukama je prevara."* COD-refusal counter feeds trust-stack L3 as a buyer-risk signal.

## 10. Courier negotiation checklist (first-call agenda)

1. **C2C pickups at varying third-party addresses under one platform account** — not warehouse-only? (make-or-break; KP proves the model exists in-market)
2. COD payout: SLA in days, method, payout to **natural persons'** accounts confirmed.
3. API: docs, REST/SOAP, test environment, webhooks vs polling, reference-data requirements (town/street IDs).
4. Labels: driver brings printed label vs sender prints vs code-on-package.
5. Reweigh policy and billing of differences.
6. Return-to-sender pricing and trigger rules for refused COD.
7. Declared-value insurance options.
8. Prohibited list.
9. Cancellation cutoffs.
10. Max weight/dimensions (reference: KP partner runs 20 kg, 60×80×100 cm [S1]).
11. DPA willingness.
12. **Partner rebate program** (the only monetization path here).
13. Rate card + rate API vs static table.

Run two negotiations in parallel (sensible pair: D Express for coverage + City Express for API quality) — competition improves every answer.

## 11. Metrics, effort, cut order

**Metrics:** booking conversion (address card → booked); shipments/week (headline); delivery success rate (DELIVERED/booked, >90% — GUESS); refusal rate (<8–10% — GUESS, calibrate); pickup latency; status-staleness incidents; support tickets per 100 shipments; % of verified reviews originating from DELIVERED.

**Effort (solo + AI):** Phase 0: 1–1.5 wks. Phase 1: 4–6 wks build (adapter 1.5–2, domain + state machine 1, UI 1–1.5, ops 0.5–1) + contract lead 2–6 wks parallel (GUESS). Phase 2: 2–3 wks. Phase 3 guided: ~1 wk.

**Cut order:** Phase 3 → Phase 2 → rate display (static table suffices) → address autocomplete (free-text fallback; accept temporarily higher failed pickups). **Never cut:** status polling + notifications, address-card privacy, the refusal-cost interstitial.

## 12. Repo mapping

**oglasino-backend** (~65%): `shipping/` module, `CourierAdapter` + first implementation, reference-data sync, poller/outbox, encryption, admin endpoints, alert hooks.
**oglasino-web / oglasino-expo:** address card, booking form, timeline, Moje pošiljke, listing `shipping_available` toggle + filter.
**oglasino-firestore-rules:** system-message type for shipment events (write-restricted to backend).
**oglasino-router:** nothing required.
**oglasino-docs:** courier negotiation checklist as a living doc; adapter decision record (*own thin adapter layer; rejected: open-source multi-carrier framework — no Serbian adapters, foreign runtime*); status-mapping table ownership.

---

## Sources

- **[S1]** KP blog — "Zakazivanje kurira" (flow, COD payout 2–3 days, cash-to-courier, 20 kg / 60×80×100 cm, reweigh warning): https://blog.kupujemprodajem.com/info/zakazivanje-kurira/
- **[S2]** SerbiaPlaces — overview of Serbian couriers (D Express scale; City Express urban focus/time windows; BEX pricing): https://serbiaplaces.com/najbolje-kurirske-sluzbe-u-srbiji/
- **[S3]** Benchmark forum — City Express B2B documented REST API mention: https://forum.benchmark.rs/threads/kurirske-slu%C5%BEbe.181575/page-228
- **[S4]** AKS — FAQ (no international COD; prohibited items): https://www.aks.rs/najcesca-pitanja/
- **[S5]** Pošta Crne Gore — Post Express (address pickup, online addressing, tracking, COD, 24h, volume discounts): https://www.postacg.me/usluge/express-usluge/crna-gora/
- **[S6]** Pošta Crne Gore — Dostava onlajn porudžbi (€2.00 ≤5 kg, next-day): https://www.postacg.me/usluge/express-usluge/dostava-onlajn-porudzbi/
- **[S7]** Pošta Crne Gore — PostPak (regional cross-border COD ≤30 kg, ≤€999, PostCash payout, tracking): https://www.postacg.me/usluge/postpak/
- **[S8]** Biznis.rs — tax/invoice rules for online sellers (registered sellers must include invoice; courier collects to business account): https://biznis.rs/preduzetnik/porezi/da-li-se-placa-porez-na-prodaju-polovne-robe-preko-interneta/