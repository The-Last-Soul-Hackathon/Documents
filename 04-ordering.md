# Module: Ordering

## Entity: Order

| Field | Type | Notes |
|---|---|---|
| id | UUID | PK |
| listing_id | UUID | FK -> Listing |
| buyer_id | UUID | FK -> Account (via Consumer or BulkBuyer profile) |
| buyer_type | enum [consumer, bulk_buyer] | denormalized for query convenience |
| quantity | decimal | final settled quantity may differ from this if hub-reweigh applies |
| unit | enum [kg, quintal, dozen, piece] | copied from listing at order time |
| order_type | enum [retail, bulk] | computed: retail if quantity < 50 kg-equivalent, else bulk |
| collection_route | enum [direct_from_farm, via_hub] | default via_hub if listing.hub_id is set, else direct_from_farm |
| price_per_unit_locked | decimal | copied from listing.price_per_unit at order creation time; immutable after creation |
| surplus_rescue | boolean | true if order placed against a listing in surplus_rescue state |
| delivery_fee_estimate | decimal, nullable | retail only; reconciled after batching |
| delivery_fee_final | decimal, nullable | set once batch/route computed (retail) or negotiated (bulk) |
| delivery_address_id | UUID | FK -> Address |
| status | enum | see State Machine |
| placed_at | timestamp | |
| farmer_response_deadline | timestamp | placed_at + response window (see Rule 2) |
| farmer_responded_at | timestamp, nullable | |
| payment_id | UUID, nullable | FK -> Payment, set on farmer_accepted |
| shipment_ids | UUID[] | FK -> Shipment, one-to-many (order can split across vehicles) |

## Entity: Inquiry (bulk pre-order negotiation)

| Field | Type | Notes |
|---|---|---|
| id | UUID | PK |
| listing_id | UUID | FK -> Listing |
| bulk_buyer_id | UUID | FK -> BulkBuyer profile |
| proposed_quantity | decimal | |
| proposed_price_per_unit | decimal | |
| status | enum [sent, countered, agreed, rejected, withdrawn] | |
| counter_quantity | decimal, nullable | farmer's counter |
| counter_price_per_unit | decimal, nullable | farmer's counter |
| resulting_order_id | UUID, nullable | FK -> Order, set only when status = agreed and Order created |

## Rules

### R1 — Order type classification
```
IF quantity_in_kg_equivalent < 50: order_type = retail
ELSE: order_type = bulk
```
Bulk orders from a NEW buyer with no prior inquiry MUST first go through Inquiry flow (R6) unless buyer explicitly opts to place a direct fixed-price bulk order at listed price.

### R2 — Stock reservation (on placement, not payment)
```
ON order.create:
  IF listing.available_qty < order.quantity: REJECT (409, "insufficient_stock")
  listing.available_qty -= order.quantity
  order.status = "placed"
  order.price_per_unit_locked = listing.price_per_unit
  order.farmer_response_deadline = now() + response_window(listing.perishability_class)

response_window(perishability_class):
  highly_perishable -> 1 hour
  perishable        -> 2 hours
  semi_durable      -> 4 hours
  durable           -> 4 hours
```
Exception: if `order.surplus_rescue == true`, skip response window entirely -> order.status = "farmer_accepted" immediately (pre-accepted, see R7).

### R3 — No-response timeout (scheduled job, runs every 5 min)
```
FOR order WHERE status = "placed" AND now() > farmer_response_deadline:
  order.status = "no_response"
  listing.available_qty += order.quantity
  notify(buyer, "seller_no_response")
  increment farmer.trust_signals.no_response_count
```

### R4 — Farmer accept/reject
```
ON farmer_accept(order):
  IF now() > order.farmer_response_deadline: REJECT (410, "window_expired")
  order.status = "farmer_accepted"
  order.farmer_responded_at = now()
  payment = create_payment_authorization(order)   -- see 05-payment-escrow.md
  order.payment_id = payment.id
  trigger logistics_assignment(order)              -- see 06-logistics.md

ON farmer_reject(order):
  order.status = "rejected"
  listing.available_qty += order.quantity
  notify(buyer, "seller_rejected")
```

### R5 — Buyer cancellation
```
ON buyer_cancel(order):
  IF order.status == "placed":
    order.status = "cancelled" (buyer-initiated, no charge existed)
    listing.available_qty += order.quantity
  ELIF order.status == "farmer_accepted" AND shipment not yet dispatched:
    void_authorization(order.payment_id)
    order.status = "refunded"
    listing.available_qty += order.quantity
  ELIF shipment already dispatched:
    REJECT (409, "cannot_cancel_in_transit") -- route to dispute flow instead
```

### R6 — Bulk negotiation (Inquiry)
```
ON inquiry.create(bulk_buyer, listing, proposed_qty, proposed_price):
  inquiry.status = "sent"
  notify(farmer, "new_inquiry")

ON farmer_counter(inquiry, qty, price):
  inquiry.status = "countered"
  inquiry.counter_quantity = qty
  inquiry.counter_price_per_unit = price
  notify(bulk_buyer, "counter_received")

ON buyer_accept_counter(inquiry) OR farmer_accept_inquiry(inquiry):
  inquiry.status = "agreed"
  order = create_order(
    listing_id = inquiry.listing_id,
    buyer_id = inquiry.bulk_buyer_id,
    quantity = inquiry.counter_quantity OR inquiry.proposed_quantity,
    price_per_unit_locked = inquiry.counter_price_per_unit OR inquiry.proposed_price_per_unit
  )
  inquiry.resulting_order_id = order.id
  -- order now follows standard R2 onward, but skip R2's price copy step
  -- since price_per_unit_locked is already set from negotiation
```

### R7 — Surplus Rescue pre-acceptance
```
IF listing.state == "surplus_rescue":
  order.surplus_rescue = true
  order.status = "farmer_accepted"   -- skip placed/response-window entirely
  payment = create_payment_authorization(order)
  order.payment_id = payment.id
  trigger logistics_assignment(order)
```

### R8 — Hub reweigh adjustment (collection_route = via_hub only)
```
ON hub_weigh_event(order, actual_weighed_qty):
  diff_pct = abs(actual_weighed_qty - order.quantity) / order.quantity
  IF diff_pct <= 0.03:
    order.quantity = actual_weighed_qty  -- silent auto-adjust
  ELIF diff_pct <= 0.15:
    order.quantity = actual_weighed_qty
    flag_for_review(order, "quantity_discrepancy")
  ELSE:
    HOLD order, escalate to admin, do not auto-adjust
  -- downstream payment capture always uses final order.quantity, not original
```

### R9 — Delivery fee
```
IF order_type == "retail":
  order.delivery_fee_estimate = zone_heuristic_fee(listing.location, order.quantity)
  -- finalized at batch time (06-logistics.md), difference credited to buyer if lower,
  -- absorbed by platform if higher -- NEVER re-charge buyer above estimate
IF order_type == "bulk":
  delivery cost is embedded in price_per_unit_locked (negotiated or listed as delivered price)
  order.delivery_fee_final = 0  -- already priced in
```

### R10 — Post-capture output (required, not optional)
```
ON payment.status = "captured":
  order.status = "payment_captured"
  emit_earnings_summary(order)   -- see 05-payment-escrow.md for computation
```

## State Machine

```
placed --(farmer accept)--> farmer_accepted --(pickup scheduled)--> picked_up
  --> in_transit --> delivered --(buyer confirm OR 48h auto)--> received
  --> payment_captured

placed --(no response / timeout)--> no_response  [terminal, stock released]
placed --(farmer reject)--> rejected             [terminal, stock released]
placed --(buyer cancel)--> cancelled             [terminal, stock released]
farmer_accepted --(buyer cancel, pre-dispatch)--> refunded [terminal, auth voided]
delivered --(dispute raised)--> disputed --(resolved)--> [payment_captured (partial) | refunded | payment_captured (contested-resolved)]

surplus_rescue listings skip: placed -> directly enter farmer_accepted
```

## API Endpoints (contract only — implement per 08-api-contracts.md conventions)

| Method | Path | Body | Response |
|---|---|---|---|
| POST | /orders | listing_id, quantity, delivery_address_id | order object, 201 |
| POST | /orders/:id/accept | — (farmer auth) | order object |
| POST | /orders/:id/reject | reason | order object |
| POST | /orders/:id/cancel | — (buyer auth) | order object |
| POST | /orders/:id/confirm-received | — (buyer auth) | order object |
| POST | /inquiries | listing_id, proposed_quantity, proposed_price_per_unit | inquiry object, 201 |
| POST | /inquiries/:id/counter | quantity, price_per_unit | inquiry object |
| POST | /inquiries/:id/accept | — | order object (converted) |

## Validation (must be enforced server-side, not just client-side)

- `quantity > 0`
- `quantity >= listing.min_order_quantity` if set
- `quantity <= listing.available_qty` at creation time (re-check inside a DB transaction / row lock to prevent race conditions — two simultaneous orders must not both pass this check against stale data)
- `buyer.status == "active"` (bulk buyer must be `verified`, not `pending`)
- `listing.status == "active"` OR `listing.status == "surplus_rescue"`

## Explicitly out of scope for MVP (do not implement, note as future work)

- Credit/deferred payment terms for bulk buyers (upfront only)
- Full VRPTW-aware response-window calculation (current version uses static perishability-class buckets, not live route data)
- Automated fraud detection on Inquiry spam (basic rate-limiting only)
