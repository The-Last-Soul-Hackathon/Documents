# 🍅 AgriQ — Perishability-Aware Routing

## 1. Objective

AgriQ should not optimize agricultural transportation only by **distance** or **cost**.

The logistics engine must also consider **how quickly the produce loses freshness**.

### Normal routing

```text
Find the cheapest / shortest feasible route
```
In this need improvement etc
### AgriQ routing

```text
Find the lowest-cost feasible route
while ensuring perishable produce can arrive
within its usable freshness window.
```

The implementation should therefore combine:

```text
Crop Characteristics
        +
Shipment Age
        +
Remaining Shelf Life
        +
Route Transit Time
        +
Temperature Conditions
        ↓
Freshness Risk
        ↓
Route Feasibility
        ↓
Cost + Time + Freshness Optimization
```

---

# 2. Core Design

The system has **two stages**:

### Stage 1 — Freshness calculation

Calculate how risky it is for a shipment to arrive using a specific route.

### Stage 2 — Route optimization

Use the freshness result together with logistics cost and travel time to select the route.

```text
Candidate Route A ──┐
Candidate Route B ──┼──> Freshness Engine ──> Feasible / Reject
Candidate Route C ──┘             │
                                  ↓
                         Cost + Time + Risk
                                  ↓
                         Recommended Route
```

---

# 3. Important Principle

Do **not** use distance directly as freshness risk.

Distance is only useful because it affects:

```text
distance → estimated transit time → freshness consumption
```

The main freshness variables are:

* Harvest time
* Current time
* Shelf life
* Transit time
* Temperature
* Crop sensitivity

---

# 4. Crop Profile

Crop-specific values must be stored separately from the scoring code.

## Example configuration

```python
CROP_PROFILES = {
    "tomato": {
        "shelf_life_hours": 168,
        "sensitivity_multiplier": 1.10,
        "temperature_min_c": 10,
        "temperature_max_c": 20
    },

    "potato": {
        "shelf_life_hours": 2160,
        "sensitivity_multiplier": 0.85,
        "temperature_min_c": 7,
        "temperature_max_c": 10
    },

    "leafy_vegetables": {
        "shelf_life_hours": 72,
        "sensitivity_multiplier": 1.20,
        "temperature_min_c": 0,
        "temperature_max_c": 5
    }
}
```

> **Note:** These values are prototype configuration values. They should eventually be replaced with crop/variety-specific post-harvest data.

---

# 5. Shipment Data Model

Each shipment should contain:

```json
{
  "shipment_id": "SHP-1024",
  "crop": "tomato",
  "quantity_kg": 2000,
  "harvested_at": "2026-09-20T04:00:00Z",
  "origin": {
    "lat": 26.9124,
    "lng": 75.7873
  },
  "destination": {
    "lat": 28.6139,
    "lng": 77.2090
  },
  "temperature_c": 24
}
```

The routing engine provides route-specific values:

```json
{
  "distance_km": 265,
  "transit_time_hours": 5.8
}
```

---

# 6. Step 1 — Calculate Shipment Age

```text
age_hours = current_time - harvested_at
```

Example:

```text
Harvested:       04:00
Current time:    16:00

Age = 12 hours
```

Python:

```python
age_hours = (
    current_time - harvested_at
).total_seconds() / 3600
```

---

# 7. Step 2 — Calculate Remaining Freshness Life

```text
remaining_life_hours =
    shelf_life_hours - age_hours
```

Example:

```text
Shelf life = 168 hours
Age        = 120 hours

Remaining life = 48 hours
```

### Edge case

If:

```text
remaining_life_hours <= 0
```

the shipment is already beyond the configured freshness window.

Return:

```text
freshness_risk = 100
priority = CRITICAL
```

and mark routes as infeasible unless the application explicitly allows delivery of already-degraded produce.

---

# 8. Step 3 — Calculate Arrival Consumption

For every candidate route:

```text
arrival_consumption =
    (age_hours + transit_time_hours)
    / shelf_life_hours
```

This represents the percentage of the configured freshness window consumed when the shipment arrives.

Example:

```text
Age = 120 hours
Transit = 8 hours
Shelf life = 168 hours

arrival_consumption =
    (120 + 8) / 168
    = 0.7619
```

So approximately:

```text
76.2% of the freshness window
will be consumed at arrival.
```

---

# 9. Step 4 — Safety Buffer

Transportation has uncertainty:

* Traffic
* Loading delays
* Unloading delays
* Vehicle delays
* Route changes
* Temperature fluctuations

Use a configurable safety buffer.

```python
SAFETY_BUFFER = 0.15
```

This means the shipment should ideally arrive before:

```text
safe_limit = 1.0 - SAFETY_BUFFER
           = 0.85
```

### Interpretation

```text
arrival_consumption <= 0.85
    → within safe planning range

arrival_consumption > 0.85
    → high freshness pressure

arrival_consumption > 1.00
    → outside configured freshness window
```

---

# 10. Step 5 — Base Freshness Risk

Convert arrival consumption into a 0–100 risk score.

Use a piecewise function rather than a completely linear formula.

```python
def calculate_base_risk(arrival_consumption: float) -> float:

    if arrival_consumption <= 0:
        return 0.0

    if arrival_consumption <= 0.50:
        return arrival_consumption * 40

    if arrival_consumption <= 0.85:
        return 20 + (
            (arrival_consumption - 0.50) / 0.35
        ) * 40

    if arrival_consumption <= 1.00:
        return 60 + (
            (arrival_consumption - 0.85) / 0.15
        ) * 30

    return 100.0
```

### Approximate interpretation

| Arrival consumption | Base risk |
| ------------------: | --------: |
|                  0% |         0 |
|                 25% |        10 |
|                 50% |        20 |
|               67.5% |        40 |
|                 85% |        60 |
|               92.5% |        75 |
|                100% |        90 |
|               >100% |       100 |

---

# 11. Step 6 — Temperature Stress

Temperature should act as a **multiplier**, not simply another independent score.

The crop profile defines:

```text
temperature_min_c
temperature_max_c
```

Then calculate:

```text
Temperature condition
        ↓
Stress multiplier
```

Use a simple MVP model:

| Condition                | Multiplier |
| ------------------------ | ---------: |
| Within recommended range |     `1.00` |
| Slightly outside         |     `1.10` |
| Moderately outside       |     `1.25` |
| Severely outside         |     `1.50` |

Example function:

```python
def temperature_multiplier(
    temperature_c: float,
    min_c: float,
    max_c: float
) -> float:

    if min_c <= temperature_c <= max_c:
        return 1.00

    distance_from_range = min(
        abs(temperature_c - min_c),
        abs(temperature_c - max_c)
    )

    if distance_from_range <= 5:
        return 1.10

    if distance_from_range <= 10:
        return 1.25

    return 1.50
```

This is intentionally simple for the MVP.

Later, this can be replaced with a crop-specific deterioration model.

---

# 12. Step 7 — Crop Sensitivity

Each crop profile contains:

```text
sensitivity_multiplier
```

Example:

```text
Leafy vegetables → 1.20
Tomato            → 1.10
Banana            → 1.05
Potato            → 0.85
Onion             → 0.80
```

The value represents how aggressively the base risk should be adjusted for the prototype.

---

# 13. Step 8 — Final Freshness Risk

Calculate:

```text
adjusted_risk =
    base_risk
    × temperature_multiplier
    × sensitivity_multiplier
```

Then cap it:

```python
freshness_risk = min(adjusted_risk, 100.0)
```

Example:

```text
Base risk               = 55
Temperature multiplier  = 1.25
Crop sensitivity        = 1.10

Final risk =
55 × 1.25 × 1.10
= 75.625

Freshness Risk = 76
```

---

# 14. Priority Classification

Convert the final risk to a delivery priority.

```python
def get_priority(risk: float) -> str:

    if risk <= 30:
        return "NORMAL"

    if risk <= 60:
        return "MODERATE"

    if risk <= 80:
        return "HIGH"

    return "CRITICAL"
```

### Priority table

|     Risk | Priority    |
| -------: | ----------- |
|   `0–30` | 🟢 NORMAL   |
|  `31–60` | 🟡 MODERATE |
|  `61–80` | 🟠 HIGH     |
| `81–100` | 🔴 CRITICAL |

---

# 15. Route Feasibility

This is extremely important.

Do not rely only on the weighted optimization score.

If a route is expected to arrive after the configured freshness window:

```text
arrival_consumption > 1.0
```

mark the route:

```text
INFEASIBLE
```

Example:

```python
if arrival_consumption > 1.0:
    route.feasible = False
    route.rejection_reason = "Exceeds freshness window"
```

---

# 16. Route Evaluation

For every candidate route, calculate:

```text
Route
│
├── Distance
├── Transit Time
├── Transport Cost
├── Arrival Consumption
├── Freshness Risk
├── Priority
└── Feasible?
```

Example:

```json
{
  "route_id": "R2",
  "distance_km": 265,
  "transit_time_hours": 5.8,
  "transport_cost": 8000,
  "arrival_consumption": 0.76,
  "freshness_risk": 64,
  "priority": "HIGH",
  "feasible": true
}
```

---

# 17. Route Optimization

AgriQ should consider:

```text
Transport Cost
+
Transit Time
+
Freshness Risk
```

However, the weights should change based on perishability.

## Normal shipment

```text
Cost      = 60%
Time      = 20%
Freshness = 20%
```

## Moderate shipment

```text
Cost      = 45%
Time      = 25%
Freshness = 30%
```

## High-risk shipment

```text
Cost      = 25%
Time      = 15%
Freshness = 60%
```

## Critical shipment

```text
Freshness becomes the dominant factor.
```

For critical loads, prioritize feasible routes with the lowest freshness risk before considering cost.

---

# 18. Normalize Route Values

Cost, time and freshness risk have different units.

Do **not** directly calculate:

```text
₹8000 + 5.8 hours + 64 risk
```

Normalize each value to `0–100`.

For a set of candidate routes:

```python
def normalize(value, minimum, maximum):

    if maximum == minimum:
        return 0.0

    return (
        (value - minimum)
        / (maximum - minimum)
    ) * 100
```

For cost and transit time:

```text
Lower = better
```

Therefore:

```python
cost_score = normalize(
    route.cost,
    min_cost,
    max_cost
)

time_score = normalize(
    route.transit_time,
    min_time,
    max_time
)
```

Freshness risk already ranges from `0–100`.

Because lower values are better, the optimization score should also be minimized.

---

# 19. Final Route Score

Example:

```python
route_score = (
    COST_WEIGHT * cost_score
    + TIME_WEIGHT * time_score
    + FRESHNESS_WEIGHT * freshness_risk
)
```

Example for a high-risk shipment:

```python
route_score = (
    0.25 * cost_score
    + 0.15 * time_score
    + 0.60 * freshness_risk
)
```

Choose the feasible route with the **lowest route score**.

---

# 20. Example

Suppose:

```text
Shipment:
Tomato
Age = 120 hours
Shelf life = 168 hours
Temperature = 24°C
```

Candidate routes:

| Route |   Cost | ETA |
| ----- | -----: | --: |
| A     | ₹7,000 |  8h |
| B     | ₹8,000 |  5h |
| C     | ₹6,500 | 10h |

Freshness engine:

| Route |    Arrival Consumption |   Risk | Feasible |
| ----- | ---------------------: | -----: | -------- |
| A     |  `(120+8)/168 = 0.762` |    ~64 | ✅        |
| B     |  `(120+5)/168 = 0.744` |  lower | ✅        |
| C     | `(120+10)/168 = 0.774` | higher | ✅        |

The cheapest route is not automatically selected.

The optimizer evaluates:

```text
Cost
+
Transit time
+
Freshness risk
```

and returns the route with the lowest total score.

---

# 21. Critical Shipment Example

Suppose:

```text
Age = 165 hours
Shelf life = 168 hours
```

Routes:

```text
Route A → 8 hours
Route B → 2 hours
Route C → 5 hours
```

Then:

```text
A:
(165 + 8) / 168 = 1.03
→ INFEASIBLE

B:
(165 + 2) / 168 = 0.994
→ Feasible but critical

C:
(165 + 5) / 168 = 1.01
→ INFEASIBLE
```

The optimizer should reject A and C.

```text
Only Route B remains feasible.
```

This is stronger than simply saying:

> "Route B has a better score."

The system is enforcing a **freshness constraint**.

---

# 22. API Design

## `POST /freshness`

Calculate freshness risk for one shipment/route.

### Request

```json
{
  "crop": "tomato",
  "harvested_at": "2026-09-20T04:00:00Z",
  "current_time": "2026-09-20T16:00:00Z",
  "temperature_c": 24,
  "transit_time_hours": 5.8
}
```

### Response

```json
{
  "crop": "tomato",
  "age_hours": 12,
  "shelf_life_hours": 168,
  "remaining_life_hours": 156,
  "arrival_consumption": 0.106,
  "base_risk": 8.4,
  "temperature_multiplier": 1.25,
  "sensitivity_multiplier": 1.10,
  "freshness_risk": 11.6,
  "priority": "NORMAL",
  "feasible": true
}
```

---

# 23. `POST /optimize`

This should be the main logistics endpoint.

### Request

```json
{
  "shipment": {
    "shipment_id": "SHP-1024",
    "crop": "tomato",
    "quantity_kg": 2000,
    "harvested_at": "2026-09-20T04:00:00Z",
    "temperature_c": 24
  },

  "routes": [
    {
      "route_id": "R1",
      "distance_km": 265,
      "transit_time_hours": 8,
      "transport_cost": 7000
    },
    {
      "route_id": "R2",
      "distance_km": 280,
      "transit_time_hours": 5,
      "transport_cost": 8000
    }
  ]
}
```

### Response

```json
{
  "recommended_route": "R2",

  "reason": {
    "freshness_risk": 58,
    "priority": "MODERATE",
    "transport_cost": 8000,
    "transit_time_hours": 5
  },

  "alternatives": [
    {
      "route_id": "R1",
      "freshness_risk": 65,
      "transport_cost": 7000,
      "transit_time_hours": 8,
      "feasible": true
    }
  ]
}
```

---

# 24. Suggested Backend Structure

```text
logistics-service/
│
├── main.py
├── models.py
│
├── freshness/
│   ├── __init__.py
│   ├── crop_profiles.py
│   ├── calculator.py
│   └── temperature.py
│
├── routing/
│   ├── __init__.py
│   ├── routing.py
│   └── optimizer.py
│
├── profit/
│   └── profit.py
│
└── tests/
    ├── test_freshness.py
    ├── test_temperature.py
    └── test_optimizer.py
```

---

# 25. Recommended Python Responsibility

## `crop_profiles.py`

Store:

```text
shelf_life_hours
sensitivity_multiplier
temperature range
```

## `temperature.py`

Calculate:

```text
temperature_multiplier
```

## `calculator.py`

Calculate:

```text
age
remaining life
arrival consumption
base risk
final freshness risk
priority
feasibility
```

## `optimizer.py`

Calculate:

```text
candidate route evaluation
normalization
dynamic weights
route score
best feasible route
```

---

# 26. Freshness Calculator Pseudocode

```python
def calculate_freshness(
    crop,
    harvested_at,
    current_time,
    transit_time_hours,
    temperature_c
):

    profile = CROP_PROFILES[crop]

    shelf_life = profile["shelf_life_hours"]

    age_hours = (
        current_time - harvested_at
    ).total_seconds() / 3600

    remaining_life = shelf_life - age_hours

    if remaining_life <= 0:
        return {
            "freshness_risk": 100,
            "priority": "CRITICAL",
            "feasible": False
        }

    arrival_consumption = (
        age_hours + transit_time_hours
    ) / shelf_life

    base_risk = calculate_base_risk(
        arrival_consumption
    )

    temp_multiplier = temperature_multiplier(
        temperature_c,
        profile["temperature_min_c"],
        profile["temperature_max_c"]
    )

    sensitivity = profile[
        "sensitivity_multiplier"
    ]

    final_risk = min(
        base_risk
        * temp_multiplier
        * sensitivity,
        100
    )

    feasible = arrival_consumption <= 1.0

    return {
        "age_hours": age_hours,
        "remaining_life_hours": remaining_life,
        "arrival_consumption": arrival_consumption,
        "freshness_risk": final_risk,
        "priority": get_priority(final_risk),
        "feasible": feasible
    }
```

---

# 27. Optimizer Pseudocode

```python
def optimize_routes(shipment, candidate_routes):

    evaluated_routes = []

    for route in candidate_routes:

        freshness = calculate_freshness(
            crop=shipment.crop,
            harvested_at=shipment.harvested_at,
            current_time=shipment.current_time,
            transit_time_hours=route.transit_time_hours,
            temperature_c=shipment.temperature_c
        )

        if not freshness["feasible"]:
            continue

        evaluated_routes.append({
            "route": route,
            "freshness": freshness
        })

    if not evaluated_routes:
        return {
            "status": "NO_FEASIBLE_ROUTE"
        }

    # Normalize cost/time here

    # Select weights based on freshness risk

    # Calculate:
    #
    # route_score =
    #   cost_weight * cost_score +
    #   time_weight * time_score +
    #   freshness_weight * freshness_risk

    # Return minimum score

    return best_route
```

---

# 28. Initial MVP Scope

### Implement now

```text
✅ Crop profile configuration
✅ Harvest time
✅ Shelf life
✅ Transit time
✅ Temperature input
✅ Freshness Risk Score
✅ Priority classification
✅ Freshness feasibility constraint
✅ Cost + time + freshness optimization
✅ OR-Tools integration
✅ `/freshness` endpoint
✅ `/optimize` endpoint
✅ Unit tests
```

### Simulate for demo

```text
🌡️ Temperature
🌦️ Weather conditions
🚚 Vehicle availability
```

### Add later

```text
🔮 ML-based shelf-life prediction
🌡️ IoT temperature sensors
📍 Real-time GPS
🌦️ Weather API integration
📦 Packaging/storage-condition effects
📊 Historical spoilage learning
```

---

# 29. Testing Requirements

At minimum, create tests for:

### Fresh shipment

```text
Age = 5%
Transit = short

Expected:
Risk = low
Priority = NORMAL
```

### Nearly expired shipment

```text
Age = 90%
Transit = short

Expected:
Risk = high/critical
```

### Expired shipment

```text
Age > shelf life

Expected:
Risk = 100
Feasible = false
```

### Fast vs cheap route

```text
Cheap route → long ETA
Expensive route → short ETA

Expected:
High-risk shipment should heavily prefer
the faster feasible route.
```

### Impossible route

```text
Arrival time > freshness window

Expected:
Route rejected.
```

### Temperature stress

```text
Same shipment
Same route

Normal temperature → lower risk
Poor temperature → higher risk
```

---

# 30. Important Implementation Rules

### Rule 1

**Never let a cheaper route automatically win.**

Freshness feasibility must be checked first.

### Rule 2

**Never hard-code crop logic inside the optimizer.**

Keep crop profiles configurable.

### Rule 3

**Calculate freshness per candidate route.**

The same shipment can have different freshness risk on different routes.

### Rule 4

**Keep the score explainable.**

The API should be able to show:

```text
Risk = 76
because:

Age                → high
Transit             → moderate
Temperature         → elevated
Crop sensitivity    → high
```

### Rule 5

**Use the rule-based model for the MVP.**

Do not introduce ML merely to call it AI.

Build the deterministic engine first and later replace individual components with learned models when historical shipment/spoilage data becomes available.

---

# 31. AgriQ Architecture

```text
                 MARKET / DEMAND DATA
                         │
                         ↓
                  Demand Forecast
                         │
                         ↓
                   Buyer Orders
                         │
                         ↓
                    Shipment
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ↓                ↓                ↓
      Crop          Harvest Age       Temperature
     Profile              │                │
        │                 │                │
        └─────────────────┼────────────────┘
                          ↓
                 Freshness Engine
                          │
                          ↓
                 Freshness Risk
                          │
                          ↓
              ┌───────────┴───────────┐
              ↓                       ↓
       Freshness OK            Freshness Too High
              │                       │
              ↓                       ↓
       Route Evaluation              Reject
              │
              ↓
     Cost + Time + Freshness
              │
              ↓
       OR-Tools Optimizer
              │
              ↓
       Recommended Route
              │
              ↓
       Delivery / Settlement
```

---

# 32. Product Goal

The feature should ultimately answer:

> **“Given this produce, its current freshness, and the available transport options, which feasible route gives the best economic outcome without exposing the shipment to unacceptable freshness risk?”**

That is the core of **AgriQ Perishability-Aware Routing**.

---

# 33. Future ML Upgrade

The initial model is deterministic.

Later:

```text
Historical Shipment Data
        ↓
Actual Delivery Outcomes
        ↓
Spoilage / Quality Outcome
        ↓
ML Model
        ↓
Predicted Remaining Shelf Life
        ↓
Improved Freshness Risk
        ↓
Improved Route Optimization
```

The architecture should therefore keep:

```text
Freshness Engine
```

separate from:

```text
Route Optimizer
```

so the rule-based model can eventually be replaced by an ML model without rewriting the entire logistics system.
