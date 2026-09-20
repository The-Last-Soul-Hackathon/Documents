# Module: AI Demand Forecasting

## Overview
Python/FastAPI service. Weekly batch retrain, not real-time. Cold-start uses Agmarknet historical data as proxy; mature phase blends in platform order history.

## Entity: Forecast
See 01-data-model.md. Key fields: id, crop, region, predicted_qty, confidence [low, medium, high], week_of, model_version.

## Pipeline

```
STEP 1 - Data ingestion (weekly batch job):
  platform_orders = aggregate ORDER WHERE status = "payment_captured"
    GROUP BY crop, region, week
  agmarknet_data = fetch_agmarknet_arrivals(crop, region, historical_weeks=52)
  festival_calendar = load_static_reference("in_festival_calendar.json")
  weather_data = fetch_weather_api(region, historical_weeks=52)

STEP 2 - Feature engineering:
  IF platform_orders.week_count >= 12:
    training_source = "platform"  -- enough native history
    confidence_ceiling = "high"
  ELIF platform_orders.week_count >= 4:
    training_source = "blended"  -- mix platform + agmarknet proxy
    confidence_ceiling = "medium"
  ELSE:
    training_source = "agmarknet_proxy_only"
    confidence_ceiling = "low"

STEP 3 - Model:
  model = Prophet(
    holidays = festival_calendar,
    regressors = [weather_data]
  )
  model.fit(training_source data, per crop-region pair)

STEP 4 - Prediction:
  forecast = model.predict(horizon_days=14)
  FOR each crop-region pair:
    save Forecast(
      predicted_qty = forecast.yhat,
      confidence = min(model_internal_confidence, confidence_ceiling),
      week_of = next_week,
      model_version = current_version
    )

STEP 5 - Insufficient data fallback:
  IF agmarknet_data is also unavailable for this crop/region (rare crop, no coverage):
    DO NOT fabricate a number
    save Forecast(predicted_qty = NULL, confidence = "unavailable")
    surface as "insufficient data for forecast" in UI, not a fake number

STEP 6 - Regional data sparsity fallback:
  IF district-level data_points < minimum_threshold (e.g., 20 weeks):
    fall back to state-level aggregated model
    surface note: "district-level data insufficient, showing state-level trend"
```

## R1 — Actionable surfacing on Listing creation
```
ON farmer opens create-listing form for crop=X, region=Y:
  forecast = get_latest_forecast(crop=X, region=Y)
  IF forecast.predicted_qty IS NOT NULL:
    show_suggested_quantity_field(forecast.predicted_qty, forecast.confidence)
  -- non-binding suggestion only, does not alter farmer's actual input
```

## R2 — Explainability
```
ON forecast displayed to user:
  decompose_prophet_components(model, forecast)
    -> trend_contribution, seasonality_contribution, holiday_contribution
  show top 1-2 contributing factors, e.g. "Diwali next week: +20% typical demand"
  -- never show a bare number with zero context
```

## R3 — Model performance monitoring (admin-facing)
```
WEEKLY:
  FOR each crop-region pair with a forecast from 1+ weeks ago:
    actual_qty = sum(ORDER.quantity WHERE crop, region, week matches)
    error_pct = abs(actual_qty - forecast.predicted_qty) / actual_qty
    log_to_admin_dashboard(crop, region, error_pct)
  -- if error_pct consistently high for a crop-region pair, flag for model review
  -- (not auto-retrained differently -- human review triggers investigation)
```

## API Endpoints

| Method | Path | Body | Response |
|---|---|---|---|
| GET | /ai/forecast?crop=&region= | — | { predicted_qty, confidence, week_of, top_factors[] } |
| GET | /ai/forecast/accuracy?crop=&region= | — (admin only) | { historical error_pct per week } |
| POST | /ai/forecast/retrain | — (internal scheduled trigger, not user-facing) | 202 |

## Explicitly out of scope for MVP
- Real-time/live forecast updates (weekly batch only)
- Planting-decision-horizon forecasting (long-range, season-ahead prediction) — current scope is short-horizon (1-2 weeks), feeding listing/logistics decisions only, not planting decisions
- Automatic model retraining triggered by detected drift (manual review only, per R3)
