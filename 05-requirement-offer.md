# Module: Requirement & Offer

## Entity: Requirement (buyer-posted demand, primarily bulk/B2B)

| Field | Type | Notes |
|---|---|---|
| id | UUID | PK |
| buyer_id | UUID | FK -> BulkBuyer profile (Consumers may also post, but low priority for MVP) |
| crop | enum (controlled vocabulary) | same vocabulary as Listing.crop |
| variety | string, nullable | |
| target_quantity | decimal | total quantity buyer wants |
| unit | enum [kg, quintal, dozen, piece] | |
| indicative_price_per_unit | decimal, nullable | non-binding guide price, NOT an offer buyers must honor |
| delivery_location_lat, delivery_location_lng | decimal | |
| needed_by_date | date | hard deadline for fulfillment |
| status | enum [open, partially_fulfilled, fulfilled, expired, cancelled] | |
| fulfilled_quantity | decimal | sum of accepted Offer.offered_qty, denormalized for fast reads |
| created_at | timestamp | |

## Entity: Offer (farmer/FPO response to a Requirement)

| Field | Type | Notes |
|---|---|---|
| id | UUID | PK |
| requirement_id | UUID | FK -> Requirement |
| seller_id | UUID | FK -> Farmer or FPO profile |
| offered_quantity | decimal | may be less than requirement.target_quantity |
| offered_price_per_unit | decimal | binding once accepted |
| status | enum [pending, accepted, rejected, withdrawn, expired] | |
| resulting_order_id | UUID, nullable | FK -> Order, set only when status = accepted |
| source_listing_id | UUID, nullable | FK -> Listing; if set, offer draws down that listing's available_qty on acceptance. If null, offer is a self-declared commitment with no system-verified stock backing (see R2a) |
| created_at | timestamp | |

## Rules

### R1 — Requirement broadcast
```
ON requirement.create:
  requirement.status = "open"
  matched_sellers = find_sellers(crop=requirement.crop, region=requirement.delivery_location, radius_km=50)
  notify(matched_sellers, "new_requirement", requirement)
```
Farmers do NOT need a pre-existing active Listing to respond — Requirement fulfillment is independent of Listing.

### R2 — Offer submission
```
ON offer.create(requirement_id, seller_id, offered_qty, offered_price, source_listing_id?):
  IF requirement.status NOT IN ["open", "partially_fulfilled"]: REJECT (409, "requirement_closed")
  IF source_listing_id IS NOT NULL:
    IF listing.available_qty < offered_qty: REJECT (409, "insufficient_stock")
  -- shared function, NOT duplicated logic -- see 00-trust-score.md / shared-utils for check_underpricing()
  warn_if_underpriced(seller, requirement.crop, requirement.delivery_location, offered_price)
  offer.status = "pending"
  offer.source_listing_id = source_listing_id
  notify(requirement.buyer_id, "new_offer_received")
```

### R2a — Capacity verification and no-listing risk flag
```
IF offer.source_listing_id IS SET:
  -- backed by verified stock; no additional risk flag
  order.stock_backed = true
ELSE:
  -- self-declared commitment, no system-verified stock
  order.stock_backed = false
  IF seller.trust_score.offer_fulfillment_rate < threshold OR seller.completed_order_count < 10:
    surface_risk_badge(offer, "unverified_capacity")  -- shown to buyer before they accept
```

### R3 — Offer acceptance (partial fulfillment supported)
```
ON buyer_accept_offer(offer):
  IF offer.status != "pending": REJECT (409, "offer_not_available")
  offer.status = "accepted"
  order = create_order(
    listing_id = NULL,               -- Requirement-originated orders have no source Listing
    requirement_id = offer.requirement_id,
    offer_id = offer.id,
    buyer_id = offer.requirement.buyer_id,
    seller_id = offer.seller_id,
    quantity = offer.offered_qty,
    price_per_unit_locked = offer.offered_price_per_unit,
    order_type = "bulk"              -- Requirement/Offer flow is bulk-only for MVP
  )
  IF offer.source_listing_id IS NOT NULL:
    listing.available_qty -= offer.offered_qty
  offer.resulting_order_id = order.id
  requirement.fulfilled_quantity += offer.offered_qty
  -- platform commission applies identically to Offer-originated orders as Listing-originated
  -- orders -- see 05-payment-escrow.md; this is a transaction fee, not a listing-usage fee
  IF requirement.fulfilled_quantity >= requirement.target_quantity:
    requirement.status = "fulfilled"
    auto_reject_remaining_pending_offers(requirement)
  ELSE:
    requirement.status = "partially_fulfilled"
  -- order now enters standard Order state machine at "placed" -- see 04-ordering.md
  -- NOTE: unlike listing-originated orders, no separate stock-reservation check against
  -- a Listing.available_qty is needed here; farmer's own capacity check happened at Offer time
```

### R4 — Offer rejection / withdrawal
```
ON buyer_reject_offer(offer): offer.status = "rejected"; notify(seller)
ON seller_withdraw_offer(offer) WHERE offer.status == "pending": offer.status = "withdrawn"
```

### R5 — Requirement expiry (scheduled job, runs daily)
```
FOR requirement WHERE status IN ["open", "partially_fulfilled"] AND now() > needed_by_date:
  requirement.status = "expired"
  auto_reject_remaining_pending_offers(requirement)
  notify(buyer, "requirement_expired", unfulfilled_qty = target_quantity - fulfilled_quantity)
```

### R6 — Multiple offers, multiple sellers, one requirement
No cap on number of offers per requirement or number of accepted offers — a 2000kg requirement can be fulfilled by any combination of accepted offers summing toward target_quantity. Each accepted offer becomes an independent Order with its own price, buyer notification, and downstream payment/logistics flow. There is no single "requirement price" — only per-offer prices.

## Order entity — required additions

```
Order.listing_id: now NULLABLE (was previously required)
Order.requirement_id: UUID, nullable, FK -> Requirement
Order.offer_id: UUID, nullable, FK -> Offer
-- exactly one of (listing_id) or (requirement_id + offer_id) must be set, never both, never neither
```

## API Endpoints

| Method | Path | Body | Response |
|---|---|---|---|
| POST | /requirements | crop, target_quantity, unit, indicative_price?, delivery_location, needed_by_date | requirement object, 201 |
| GET | /requirements?crop=&region= | — (farmer-facing feed) | requirement[] with status=open/partially_fulfilled |
| POST | /requirements/:id/offers | offered_quantity, offered_price_per_unit | offer object, 201 |
| POST | /offers/:id/accept | — (buyer auth) | order object |
| POST | /offers/:id/reject | — (buyer auth) | offer object |
| POST | /offers/:id/withdraw | — (seller auth) | offer object |

## Validation

- `target_quantity > 0`, `needed_by_date > today`
- `offered_quantity > 0`, `offered_quantity` may exceed remaining unfulfilled amount (buyer decides how much to accept — no hard cap enforced at submission, only warn)
- Buyer must be `verified` bulk buyer status to post a Requirement (Consumers restricted from this module in MVP)
- Requirement.indicative_price is advisory only — never used to auto-reject or auto-accept an Offer

## Explicitly out of scope for MVP

- Automated best-offer ranking/recommendation to buyer (buyer manually reviews and accepts offers)
- Partial-offer auto-splitting (a farmer offers a fixed quantity; system does not auto-negotiate a smaller amount)
- Reverse-auction / bidding-war mechanics between competing offers
