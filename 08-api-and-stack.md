# Module: API Conventions & Tech Stack

## Tech Stack (locked)

| Layer | Choice | Reasoning |
|---|---|---|
| Main backend | Node.js + Express (monolith) | Handles auth, listing, ordering, payment orchestration, logistics coordination |
| AI service | Python + FastAPI (separate service) | Forecasting (Prophet) and route optimization (OR-Tools) — libraries not well-supported in Node |
| Database | Supabase | Relational integrity needed for stock reservation, escrow state, FK constraints across many entities |
| Payment | Gateway with authorize/capture support (e.g., Razorpay) | Implements escrow *behavior* without requiring a PA/PG license |
| Maps/routing | Any maps API with distance-matrix support (e.g., Google Maps Distance Matrix, OSRM) | Feeds OR-Tools VRP solver |
| Reference data | Agmarknet API/CSV | Mandi price reference, demand-forecast cold-start proxy |
| Weather | Any free-tier weather API (e.g., OpenWeatherMap) | Forecast regressor |
| Auth | JWT (access + refresh tokens) | Standard for Node backend |

## Inter-service communication
Node backend calls Python FastAPI service via plain REST/HTTP (JSON over POST). No message queue, no service mesh — deliberate scope-down for hackathon timeline. Endpoints:
- `POST /ai/logistics/batch-solve`
- `POST /ai/logistics/reoptimize`
- `GET /ai/forecast`

## API conventions (applies to all endpoints across all modules)

- Base path: `/api/v1/`
- Auth: `Authorization: Bearer <JWT>` header on all endpoints except `/auth/*` and public `GET /listings`
- Response envelope:
```json
{ "data": { ... }, "error": null }
```
on error:
```json
{ "data": null, "error": { "code": "insufficient_stock", "message": "..." } }
```
- Standard HTTP status codes: 200 (success), 201 (created), 400 (validation), 401 (auth), 403 (forbidden — e.g., pending account attempting write action), 404, 409 (conflict — e.g., duplicate registration, stock race), 410 (gone — e.g., response window expired), 500
- Pagination: `?page=&limit=` on all list endpoints, default limit=20, max=100
- Timestamps: ISO 8601 UTC throughout
- Money fields: always decimal with 2 places, currency assumed INR (no multi-currency for MVP)

## Cross-module dependency order (build sequence for an AI agent)

1. `01-data-model.md` — create schema/migrations first, everything else depends on this
2. `00-trust-score.md` — shared utility functions and TrustProfile table
3. `02-registration.md` — auth + all profile types
4. `03-listing.md`
5. `05-requirement-offer.md` — depends on Listing (source_listing_id) and Order shape
6. `04-ordering.md` — depends on Listing, Requirement/Offer, TrustProfile
7. `05-payment-escrow.md` — depends on Order
8. `06-logistics.md` — depends on Order, Vehicle
9. `07-forecasting.md` — depends on Order history (can be stubbed with Agmarknet-only data initially, since platform order history won't exist until orders start flowing)

## Environment/config (not secrets — structure only)

```
DATABASE_URL=postgresql://...
JWT_SECRET=...
PAYMENT_GATEWAY_KEY=...
MAPS_API_KEY=...
WEATHER_API_KEY=...
AGMARKNET_API_ENDPOINT=...
AI_SERVICE_URL=http://localhost:8001  -- FastAPI service base URL
```

## Testing expectations (minimum for MVP demo)
- Unit tests for: stock reservation race condition (R2 in Ordering), trust score composite calculation, underpricing warning threshold
- Integration test for: full order lifecycle happy path (place -> accept -> pay -> deliver -> confirm -> capture)
- Integration test for: Offer-originated order path (Requirement -> Offer -> accept -> Order)
