# Module: Data Model (consolidated)

This file is the single source of truth for entity shapes referenced across all other spec files. Individual module specs (03-listing.md, 04-ordering.md, 05-requirement-offer.md, etc.) may repeat relevant fields for readability, but this file wins in case of conflict.

## ACCOUNT
| Field | Type |
|---|---|
| id | UUID PK |
| phone | string, unique login credential (NOT unique identity key — one phone may link multiple role-profiles) |
| password_hash | string |
| created_at | timestamp |

## Role Profiles (each FK -> ACCOUNT.id)

**FPO_PROFILE**: id, account_id, name, reg_number (CIN/coop ID, format-validated only for MVP), address, member_count, bank_upi_id, jurisdiction, status [pending, active, rejected], trust_profile_id

**FARMER_PROFILE**: id, account_id, fpo_id (nullable), name, aadhaar_hash (never raw), gps_location, bank_upi_id, status [pending, active, rejected], trust_profile_id

**CONSUMER_PROFILE**: id, account_id, name, status [active] (no pending state — light KYC), trust_profile_id

**BULKBUYER_PROFILE**: id, account_id, business_name, gstin (format-validated), payment_terms [upfront_only for MVP], status [pending, active, rejected], trust_profile_id

**LOGISTICS_PROFILE**: id, account_id, type [individual_driver, transport_company], license_number, vehicle_ids[], service_area, status [pending, active, suspended, rejected], trust_profile_id

## LISTING
See 03-listing.md for full field list and rules. Key fields: id, seller_id, seller_type, crop, unit, quantity_total, available_qty, price_per_unit, perishability_class, hub_id (nullable), status, surplus_rescue_discount_pct.

## INQUIRY (single-listing bulk negotiation)
id, listing_id FK, bulk_buyer_id FK, proposed_quantity, proposed_price_per_unit, counter_quantity, counter_price_per_unit, status [sent, countered, agreed, rejected, withdrawn], resulting_order_id (nullable FK -> ORDER)

## REQUIREMENT (broadcast demand)
See 05-requirement-offer.md. Key fields: id, buyer_id FK, crop, target_quantity, fulfilled_quantity, indicative_price_per_unit, needed_by_date, status.

## OFFER (response to a Requirement)
See 05-requirement-offer.md. Key fields: id, requirement_id FK, seller_id FK, offered_quantity, offered_price_per_unit, source_listing_id (nullable FK -> LISTING), status, resulting_order_id (nullable FK -> ORDER).

## ORDER
| Field | Type | Notes |
|---|---|---|
| id | UUID PK |
| listing_id | UUID, nullable FK -> LISTING | set for Listing-originated and Inquiry-originated orders |
| requirement_id | UUID, nullable FK -> REQUIREMENT | set for Offer-originated orders |
| offer_id | UUID, nullable FK -> OFFER | set for Offer-originated orders |
| **constraint** | | exactly one of (listing_id) or (requirement_id + offer_id) must be set — never both, never neither |
| buyer_id | UUID FK -> CONSUMER_PROFILE or BULKBUYER_PROFILE | |
| seller_id | UUID FK -> FARMER_PROFILE or FPO_PROFILE | denormalized for query convenience on Offer-originated orders |
| quantity | decimal | may be adjusted post hub-reweigh (Ordering R8) |
| unit | enum [kg, quintal, dozen, piece] | |
| order_type | enum [retail, bulk] | |
| collection_route | enum [direct_from_farm, via_hub] | |
| price_per_unit_locked | decimal | |
| surplus_rescue | boolean | |
| stock_backed | boolean | false only possible for Offer-originated orders with no source_listing_id |
| delivery_fee_estimate, delivery_fee_final | decimal, nullable | |
| status | enum, see 04-ordering.md state machine | |
| farmer_response_deadline | timestamp | |
| payment_id | UUID, nullable FK -> PAYMENT | |
| shipment_ids | UUID[] FK -> SHIPMENT | one-to-many |

## SHIPMENT
id, order_id FK, vehicle_id FK, pickup_status, delivery_status, route_sequence, pickup_photo_url, delivery_scan_at

## VEHICLE
id, logistics_id FK -> LOGISTICS_PROFILE, type, capacity_kg, status [available, busy, offline]

## PAYMENT
id, order_id FK (1:1), base_amount, commission (5% of base_amount, buyer-side), delivery_fee, total_buyer_amount, escrow_status [none, authorized, captured, voided, partial_captured], captured_at, driver_payout_status, driver_payout_at

## DISPUTE
id, order_id FK, category [quantity_shortfall, quality_spoilage, wrong_item, not_delivered], evidence_url, liability [farmer, logistics_partner, unresolved], resolution [partial_capture, full_void, contested_upheld], resolved_at

## FORECAST
id, crop, region, predicted_qty, confidence [low, medium, high], week_of, model_version

## TrustProfile
See 00-trust-score.md. One per role-profile (Farmer, FPO, LogisticsPartner, BulkBuyer, Consumer), not per Account.

## Hub
id, name, location_lat, location_lng, fpo_id (nullable, if hub is FPO-operated), service_radius_km, status [active, inactive]

## Cross-entity constraints (must be enforced at the DB or application-transaction level, not just documented)

1. `ORDER`: exactly-one-of constraint on (listing_id) vs (requirement_id + offer_id) — enforce via CHECK constraint or application-layer validation on every create.
2. `LISTING.available_qty` decrements must happen inside the same transaction as `ORDER` creation (Ordering R2) or `OFFER` acceptance (Requirement/Offer R3) to prevent race conditions — use row-level locking (`SELECT ... FOR UPDATE`) on the Listing row during this check.
3. `PAYMENT.escrow_status` transitions (`authorized` -> `captured`) must be idempotent — a retried capture call (e.g., from a retried webhook) must not double-charge.
4. `TrustProfile` recompute job must run after all of the day's order/dispute/offer status changes are finalized, not concurrently with them, to avoid computing against a half-updated day's data.
