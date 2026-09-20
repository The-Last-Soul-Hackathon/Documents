# Module: Trust Score (shared model)

## Purpose
A single composite trust score exists per Account-role (Farmer, FPO, Logistics Partner, Buyer), built from independently-tracked signals. Signals must NOT be conflated with each other or with the "New Seller" badge — each answers a different question.

## Entity: TrustProfile

| Field | Type | Notes |
|---|---|---|
| id | UUID | PK |
| profile_id | UUID | FK -> Farmer/FPO/LogisticsPartner/BulkBuyer profile (one per role, not per Account) |
| completed_order_count | int | used ONLY for "New Seller" badge threshold (<10), never for trust_score directly |
| no_response_rate | decimal [0-1] | rolling rate of orders where farmer/seller missed response window (Ordering R3) |
| dispute_liability_rate | decimal [0-1] | rolling rate of disputes where this party was found liable (Payment/Dispute module) |
| offer_fulfillment_rate | decimal [0-1] | rolling rate of accepted Offers that converted into a completed order vs abandoned (Requirement/Offer module) |
| trust_score | int [0-100] | composite, computed nightly by scheduled job, NOT real-time |
| updated_at | timestamp | |

## Composite scoring (scheduled job, nightly)

```
FUNCTION recompute_trust_score(profile):
  base = 50  -- neutral starting point for any profile with insufficient history
  IF profile.completed_order_count < 3:
    profile.trust_score = base  -- not enough data, stay neutral, do not penalize/reward yet
    RETURN

  penalty = (profile.no_response_rate * 15)
          + (profile.dispute_liability_rate * 20)
          + ((1 - profile.offer_fulfillment_rate) * 15)   -- only weighted if profile has ever submitted offers
  reward  = min(profile.completed_order_count, 50) * 0.6   -- caps contribution from raw volume

  profile.trust_score = clamp(base - penalty + reward, 0, 100)
```

## Shared function: check_underpricing (used by Listing R1 and Requirement/Offer R2 — single implementation, not duplicated)

```
FUNCTION warn_if_underpriced(seller, crop, region, price):
  ref = get_agmarknet_reference(crop, region)
  IF ref IS NOT NULL AND price < ref * 0.7:
    RETURN warning("price_below_reference", ref)
  RETURN null  -- non-blocking always; never rejects the listing/offer
```

## Consumption rules (how other modules must use this, not reinvent it)

- **New Seller badge** (Listing FR-5): `completed_order_count < 10` — NEVER read `trust_score` for this check.
- **Risk badge on Offers without verified stock** (Requirement/Offer R2a): reads `offer_fulfillment_rate` and `completed_order_count`, not raw `trust_score` alone — a low-volume-but-otherwise-fine seller shouldn't be flagged purely on a neutral trust_score.
- **Logistics Partner suspension** (Logistics module): uses the same `TrustProfile` shape, but `dispute_liability_rate` here reflects delivery-caused disputes specifically (see Payment/Dispute R1 liability-attribution logic), not seller-side disputes.
- **Buyer-side trust** (Ordering R3's no-show tracking): Consumers/Bulk Buyers also get a TrustProfile; `no_response_rate` here is repurposed to mean "reserved stock and never triggered farmer response" is not applicable to buyers — buyers get a separate signal: `order_no_show_rate` (order placed, farmer accepted, buyer never completes payment path / ghosts). Do not merge this into the same field as seller no-response; they are different actors' failure modes and must stay in clearly named separate fields even though conceptually similar.

## Explicitly out of scope for MVP

- Real-time trust score updates (nightly batch is sufficient; no action should block on a live recompute)
- Cross-role trust portability (a Farmer's trust score does not transfer if the same Account also registers as a Logistics Partner — each role's TrustProfile is independent)
