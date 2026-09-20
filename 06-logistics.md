# Module: Logistics Assignment & Route Optimization

## Overview
Five chained functions: batch trigger -> vehicle matching -> bin-packing -> route sequencing (VRP solve) -> dispatch/tracking. Implemented as a Python/FastAPI service, called by the Node.js backend via REST.

## Entity: Vehicle
See 01-data-model.md. Key fields: id, logistics_id, type, capacity_kg, status [available, busy, offline], current_location_lat/lng.

## Entity: Shipment
| Field | Type | Notes |
|---|---|---|
| id | UUID PK | |
| order_ids | UUID[] | one shipment may carry multiple orders (batched retail); one order may span multiple shipments (large bulk order split across vehicles) |
| vehicle_id | UUID FK -> VEHICLE | |
| route_sequence | JSON | ordered list of {stop_type: pickup\|drop, order_id, lat, lng, eta} |
| pickup_status | enum [pending, in_progress, completed] | |
| delivery_status | enum [pending, in_progress, completed] | |
| pickup_photo_url | string, nullable | required before pickup_status = completed |
| delivery_scan_at | timestamp, nullable | this is the "driver-delivered" event — distinct from buyer's "Received" confirmation in Order |

## R1 — Batch window trigger (scheduled job)
```
TRIGGER CONDITIONS (whichever fires first):
  - every 2 hours (fixed schedule), OR
  - a zone accumulates enough pending orders to fill >=1 full vehicle capacity early, OR
  - any order in the zone has perishability_class = "highly_perishable" AND
    time_to_deadline < 50% of remaining window  -- early trigger override

FOR each geographic zone (15km radius cluster of pickup locations):
  batch_orders = ORDER WHERE status = "farmer_accepted"
    AND shipment_id IS NULL
    AND pickup_location WITHIN zone
  IF batch_orders is empty: SKIP
  CALL vehicle_matching(zone, batch_orders)
```

## R2 — Vehicle/driver pool matching
```
FUNCTION vehicle_matching(zone, batch_orders):
  candidates = VEHICLE WHERE status = "available"
    AND logistics_profile.status = "active"  -- excludes suspended
    AND service_area OVERLAPS zone
  IF candidates is empty:
    retry_with_expanded_radius(zone, 30km)  -- fallback, do not fail silently
    IF still empty: flag_unassigned_batch(zone, batch_orders); notify(admin)
    RETURN
  CALL bin_packing(batch_orders, candidates)
```

## R3 — Bin-packing (capacity assignment)
```
FUNCTION bin_packing(orders, vehicles):
  -- greedy heuristic sufficient at demo scale (<50 orders/batch):
  sort orders by weight_kg DESC
  FOR order IN orders:
    assign to vehicle with (capacity_kg - current_load) >= order.weight_kg,
      preferring least-utilized vehicle
    IF no vehicle fits (single order exceeds any vehicle capacity):
      SPLIT order into multiple shipments across >=2 vehicles (Order.shipment_ids becomes 1:many)
  RETURN vehicle_load_groups
```

## R4 — Route sequencing (VRP solve via OR-Tools)
```
FOR each vehicle_load_group:
  stops = [pickup locations] + [drop locations] for assigned orders
  distance_matrix = compute via maps API
  solve using OR-Tools CVRP solver (capacity already satisfied by R3;
    this stage optimizes sequencing for shortest total distance/time)
  HARD CONSTRAINT CHECK:
    IF projected_arrival_time(any highly_perishable order's drop) > order.deadline:
      flag_order_for_priority_dedicated_dispatch(order)  -- pulled out of batch, handled separately
  route_sequence = solver_output
  create SHIPMENT(vehicle_id, order_ids, route_sequence)
  FOR order IN assigned orders: order.status stays "farmer_accepted" until pickup_status flips
```

## R5 — Dispatch + tracking
```
push_route_to_driver(shipment)  -- via app if built, else SMS/WhatsApp with stop list for MVP

ON driver_arrives_at_pickup(shipment, order):
  REQUIRE pickup_photo_url before allowing status update  -- dispute evidence baseline
  IF order.collection_route == "via_hub":
    trigger hub_weigh_event(order, actual_weighed_qty)  -- see 04-ordering.md R8
  shipment.pickup_status = "in_progress" -> "completed" per stop
  order.status = "picked_up"

ON driver_arrives_at_drop(shipment, order):
  REQUIRE delivery_scan (photo or digital signature)
  shipment.delivery_status updates per stop
  shipment.delivery_scan_at = now()
  order.status = "delivered"
  trigger notify(buyer, "delivery_notification")  -- starts 48h auto-confirm countdown (04-ordering.md)
  trigger driver_payout_eligible(order)  -- independent of buyer confirmation, see 05-payment-escrow.md
```

## R6 — Driver cancels/goes offline mid-assignment
```
ON driver_unavailable(shipment) WHERE pickup_status != "completed" for some stops:
  orphaned_orders = orders in shipment with pickup_status = "pending"
  remove orphaned_orders from this shipment
  re-run R2+R3+R4 for ONLY orphaned_orders (not the full original batch)
```

## API Endpoints (internal, Node -> Python service)

| Method | Path | Body | Response |
|---|---|---|---|
| POST | /ai/logistics/batch-solve | zone_id, order_ids[], vehicle_candidates[] | { assignments: [{vehicle_id, order_ids, route_sequence}] } |
| POST | /ai/logistics/reoptimize | shipment_id, orphaned_order_ids[] | updated assignment |

## API Endpoints (driver-facing, via Node backend)

| Method | Path | Body | Response |
|---|---|---|---|
| GET | /driver/shipments/active | — | shipment[] with route_sequence |
| POST | /driver/shipments/:id/pickup | order_id, photo_url | shipment object |
| POST | /driver/shipments/:id/deliver | order_id, scan_evidence | shipment object |

## Explicitly out of scope for MVP
- Full VRPTW (hard per-stop time windows) — current version uses a post-hoc deadline CHECK that pulls perishable orders out of batch rather than solving time windows natively
- Live GPS vehicle tracking — manual available/busy status toggle for MVP; real-time location is a stated future integration
- True optimal bin-packing (NP-hard) — greedy heuristic accepted at demo scale
