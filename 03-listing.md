# Module: Listing

## Entity: Listing

| Field | Type | Notes |
|---|---|---|
| id | UUID | PK |
| seller_id | UUID | FK -> Farmer or FPO profile |
| seller_type | enum [farmer, fpo] | denormalized for query convenience |
| crop | enum (controlled vocabulary) | must join against Agmarknet crop keys — no free text |
| variety | string, nullable | defaults to "standard" if omitted |
| quantity_total | decimal | as originally listed |
| available_qty | decimal | decrements on reservation (R2 in Ordering), restored on cancel/refund |
| unit | enum [kg, quintal, dozen, piece] | |
| price_per_unit | decimal | editable while status = active (see R4) |
| location_lat, location_lng | decimal | auto-filled from seller profile, editable |
| harvest_date | date | must be <= today + 60 days |
| perishability_class | enum [highly_perishable, perishable, semi_durable, durable] | required; drives R3 (expiry), R5 (surplus rescue), and farmer response window in Ordering |
| min_order_quantity | decimal, nullable | |
| quality_grade | enum [A, B, C], nullable | self-declared |
| photo_urls | string[] | at least 1 required |
| reference_price | decimal | system-computed from Agmarknet, read-only, refreshed daily |
| hub_id | UUID, nullable | FK -> Hub; if set, collection_route on resulting orders defaults to via_hub |
| status | enum | see State Machine |
| surplus_rescue_discount_pct | decimal, nullable | set when status = surplus_rescue |
| created_at, updated_at | timestamp | |

## Rules

### R1 — Underpricing warning (non-blocking)
```
ON listing.create OR listing.update(price_per_unit):
  ref = get_agmarknet_reference(listing.crop, listing.location)
  IF ref IS NOT NULL AND listing.price_per_unit < ref * 0.7:
    warn(seller, "price_below_reference", ref)
  -- non-blocking: listing still saves regardless of warning
```

### R2 — New Seller badge (computed at render time, not stored)
```
FUNCTION is_new_seller(seller_id):
  RETURN count(Order WHERE seller_id = seller_id AND status = "payment_captured") < 10
```
Never derive this from trust_score. Trust score and order-count are separate signals (see 00-trust-score.md).

### R3 — Listing expiry (scheduled job, runs hourly)
```
expiry_window(perishability_class):
  highly_perishable -> 24 hours from creation if unsold
  perishable        -> 5 days
  semi_durable      -> 21 days
  durable           -> 90 days

FOR listing WHERE status = "active" AND now() > created_at + expiry_window(perishability_class):
  IF available_qty == quantity_total:  -- fully unsold
    listing.status = "expired"
  ELSE:
    -- partially sold, let existing reservations complete, block new orders
    listing.status = "expired"
    -- available_qty remains as-is; no new orders accepted against this id
```

### R4 — Price/quantity edit while Active
```
ON listing.update(price_per_unit OR quantity_total) WHERE status = "active":
  ALLOW update
  -- does NOT retroactively affect orders already placed:
  -- Order.price_per_unit_locked was copied at order creation time (see 04-ordering.md R2)
  -- so this update only affects FUTURE orders against this listing
```

### R5 — Surplus Rescue trigger (scheduled job, runs hourly)
```
FOR listing WHERE status = "active" AND available_qty > 0
  AND time_remaining_in_expiry_window(listing) <= 0.25 * expiry_window(listing.perishability_class):
    listing.status = "surplus_rescue"
    discount_pct = compute_discount(time_remaining_ratio)  -- e.g. 15% at 25% remaining, up to 30% near-zero
    listing.surplus_rescue_discount_pct = discount_pct
    effective_price = listing.price_per_unit * (1 - discount_pct)
    push_notification(nearby_buyers(listing.location, radius_km=15), listing, effective_price)
```
Orders placed against a `surplus_rescue` listing use `effective_price`, not `price_per_unit`, and skip the farmer response window entirely (see 04-ordering.md R7).

### R6 — Withdrawal
```
ON farmer_withdraw(listing):
  pending_orders = Order WHERE listing_id = listing.id AND status = "placed"
  IF pending_orders is not empty:
    FOR order IN pending_orders:
      order.status = "cancelled"
      notify(order.buyer_id, "listing_withdrawn")
      -- no charge existed yet (order.status was "placed", pre-payment) -- no refund flow needed
  listing.status = "withdrawn"
```

### R7 — Stock restoration on post-delivery void (cross-module, triggered from Payment/Dispute)
```
ON order.status -> "refunded" OR fully-voided dispute resolution:
  IF listing.status == "active":
    listing.available_qty += order.quantity
  ELSE:
    -- listing expired/withdrawn since order was placed
    log_inventory_writeoff(listing.id, order.quantity)
```

## State Machine

```
draft --(publish)--> active --(available_qty reaches 0)--> sold_out
active --(expiry window elapsed, unsold)--> expired
active --(farmer withdraws)--> withdrawn
active --(nearing expiry, still available)--> surplus_rescue --(sold out or expiry)--> sold_out | expired
```
`sold_out` and `expired` listings are retained in records (not deleted) for seller history, ratings, and forecast training data.

## API Endpoints

| Method | Path | Body | Response |
|---|---|---|---|
| POST | /listings | crop, variety, quantity_total, unit, price_per_unit, location, harvest_date, perishability_class, photos | listing object, 201 |
| PATCH | /listings/:id | price_per_unit?, quantity_total? | listing object (only if status=active) |
| POST | /listings/:id/withdraw | — | listing object |
| GET | /listings?crop=&location=&status=active | — | listing[] with reference_price attached |

## Validation

- `quantity_total > 0`, `price_per_unit > 0`
- `harvest_date <= today + 60 days`
- `photo_urls.length >= 1`
- `crop` must exist in controlled vocabulary table
- seller must be `verified` (not `pending`) — checked against Farmer/FPO profile status

## Explicitly out of scope for MVP

- Automated AI-based quality grading from photos (manual self-declaration only)
- Dynamic real-time Agmarknet price refresh (daily batch refresh is sufficient)
- Per-listing custom expiry override (fixed perishability-class buckets only)
