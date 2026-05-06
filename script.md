# Promotion Simulation Engine: Exact Logic, Model Build, and Friday Talk Track

## 1. Executive Summary

The current promotion simulation stack is a **decision-model-based engine** that predicts **5 outputs directly** for a scenario:

- `units_sold`
- `revenue`
- `return_rate`
- `conversion_rate`
- `aov`

The core model is trained in [promo_decision_model.py](/C:/Users/admin/work/avantika/spanx/backend/models/promo_decision_model.py:531) and persisted as [decision_model.pkl](/C:/Users/admin/work/avantika/spanx/model_artifacts/decision_model.pkl). The main simulation/optimizer logic lives in [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:1), and the server routes for the UI point there from [server.py](/C:/Users/admin/work/avantika/spanx/backend/server.py:203).

The most important practical point for Friday:

- The **optimizer endpoint** uses the proper promotion-type dispatch (`%_OFF`, `BUNDLE`, `FIRST_TIME`, `CLEARANCE`, `FREE_SHIPPING`).
- The main **simulate payload path currently routes through the `%_OFF` simulator**, even when another promo type is selected, because `simulate_scenario_selection()` calls `simulate_scenario_bundle()`, and that is currently just an alias to `simulate_percent_off()`. See [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:2244) and [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:2922).

So if Ritesh asks, “What is the engine doing today?”, the accurate answer is:

1. The model itself is a multi-target tree ensemble.
2. The optimizer understands type-specific promo logic.
3. The main simulate screen has wiring issues, so it often behaves like `%_OFF` even for other types.

---

## 2. Which Code Path Is Actually Live

### UI/API routing

- `POST /api/screens/step10-promotion-simulation/simulate` -> `build_assistant_simulation_response()` in [server.py](/C:/Users/admin/work/avantika/spanx/backend/server.py:203) and [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:4214)
- `POST /api/screens/step10-promotion-simulation/optimize` -> `build_assistant_optimizer_response()` in [server.py](/C:/Users/admin/work/avantika/spanx/backend/server.py:217) and [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:4224)

### Important current behavior

The simulate path is:

`build_assistant_simulation_response()`  
-> `build_backend_payload()`  
-> `simulate_scenario_selection()`  
-> `simulate_scenario_bundle()`  
-> `simulate_percent_off()`

That is visible here:

- `simulate_scenario_bundle()` returns `simulate_percent_off(...)`: [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:2244)
- `simulate_scenario_selection()` uses `simulate_scenario_bundle(...)`: [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:2960)

By contrast, the optimizer path uses:

`run_promotion_optimizer()`  
-> `simulate_scenario()`  
-> dispatch by `promotion_type`

See [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:2908) and [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:3767).

---

## 3. How We Built the Model

### Training data used

The repo training loader prefers curated generated CSVs before raw Excel because the raw transaction export can contain duplicates. See [train_promo_decision_model_from_repo.py](/C:/Users/admin/work/avantika/spanx/backend/models/train_promo_decision_model_from_repo.py:40).

Primary inputs:

- transactions
- product master
- customer master
- inventory
- promotion master
- demand signals
- weather report
- event report
- holiday report

### Training grain

The model is trained at:

- **daily x product x channel x segment**

This is set in the saved artifact metadata and in the returned training artifact.

### Training-frame construction

The training frame is built in [promo_decision_model.py](/C:/Users/admin/work/avantika/spanx/backend/models/promo_decision_model.py:531). The code:

1. Cleans and deduplicates transactions.
2. Resolves promo metadata from observed promo IDs and promotion master.
3. Merges product, customer, inventory, social, weather, event, and holiday context.
4. Aggregates to daily `product_id/product_name/category/channel/segment/promo_type/promo_flag`.
5. Creates targets and rolling baseline features.

### Targets

The model predicts:

- `units_sold`
- `revenue`
- `return_rate`
- `conversion_rate`
- `aov`

See [promo_decision_model.py](/C:/Users/admin/work/avantika/spanx/backend/models/promo_decision_model.py:17).

### Features

The saved artifact currently uses **56 features**. Major groups:

- Price and discount: `base_price`, `gross_price`, `discount_pct`, `cost_price`, `promo_flag`
- Inventory: `current_stock`, `inbound_stock`, `weeks_of_cover`, `sell_through_rate`, `stockout_flag`, `overstock_flag`, `inventory_pressure`, `inventory_age_days`
- Customer: `price_sensitivity_score`, `promo_affinity`, `return_risk_score`, `customer_aov`
- Channel/context: `channel_margin`, `channel_conversion_baseline`
- Weather: `temperature_c`, `rain_chance_pct`, `precip_mm`, `humidity_pct`, `wind_kph`, `uv_index`, `weather_condition`
- Calendar/signals: `holiday_flag`, `event_flag`, `event_count`, `social_trend_score`, `sentiment_score`, `engagement_rate`, `influencer_share`
- Time: `day_of_week`, `week_of_year`, `month`, `is_weekend`
- Rolling baselines: `baseline_units_7d`, `baseline_units_28d`, `baseline_revenue_7d`, `baseline_revenue_28d`, `baseline_return_rate_28d`, `baseline_aov_28d`, `baseline_orders_28d`, `baseline_unique_customers_28d`, `baseline_discount_28d`
- Interaction terms: `discount_x_inventory`, `discount_x_segment`, `weather_x_channel`, `trend_x_category`
- Categoricals: `product_id`, `product_name`, `category`, `channel`, `segment`, `promo_type`, `weather_condition`

### Model architecture

The trained artifact is a **TargetWiseTreeEnsemble**:

- `ExtraTreesRegressor` for `units_sold`, `revenue`, `aov`
- `RandomForestRegressor` for `return_rate`, `conversion_rate`

See [promo_decision_model.py](/C:/Users/admin/work/avantika/spanx/backend/models/promo_decision_model.py:107).

The model also transforms targets before training:

- `log1p` for `units_sold`, `revenue`, `aov`
- `logit` for `return_rate`, `conversion_rate`

Predictions are inverse-transformed at inference time in [promo_decision_model.py](/C:/Users/admin/work/avantika/spanx/backend/models/promo_decision_model.py:889).

### Train/test split and weighting

- Split is **time ordered**, not random.
- Sample weights emphasize:
  - promo rows
  - holiday rows
  - event rows
  - higher discount rows

See [promo_decision_model.py](/C:/Users/admin/work/avantika/spanx/backend/models/promo_decision_model.py:788).

### Current saved model metrics

From [.generated_data/decision_model_metrics.json](/C:/Users/admin/work/avantika/spanx/.generated_data/decision_model_metrics.json:1):

- Train rows: `6348`
- Test rows: `1588`
- Training window: `2025-04-14` to `2026-04-13`
- Revenue `R²`: `0.9097`
- Conversion rate `R²`: `0.9938`
- AOV `R²`: `0.9992`
- Units sold `R²`: `-0.0311`
- Return rate `R²`: `-0.2148`

So the model is currently **stronger on revenue / conversion / AOV** than on **units / return rate**.

---

## 4. Exact Runtime Logic

### 4.1 Scenario normalization

The request is normalized in [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:1144).

Important behaviors:

- `promotion_type` defaults to `%_OFF`
- default `discount_depth` is `20`
- default margin floor is `30%`
- default max return rate is `25%`

### 4.2 Important normalization bug

Two places use `or` on numeric discount depth:

- [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:1179)
- [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:1240)

That means **`discount_depth = 0` is treated as false and replaced with the default**, effectively `20`.  
So a “0%” scenario is not preserved correctly in the current request merge path.

### 4.3 Percent-off simulation logic

The active `%_OFF` simulation is in [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:1995).

For each segment in scope:

1. Build a **baseline feature row** with:
   - `promo_type = NONE`
   - `discount_pct = 0`
   - `promo_flag = 0`
2. Build a **promo feature row** with:
   - selected `promo_type`
   - `discount_pct = discount_depth / 100`
   - `promo_flag = 1`
   - `gross_price = base_price * (1 - discount_pct)`
3. Score both rows with `predict_promo_decision()`
4. Weight segment predictions and aggregate them

The business math is then:

- `baseline_margin = baseline_revenue - cost_price * baseline_units`
- `new_margin = new_revenue - cost_price * new_units`
- `incremental_units = new_units - baseline_units`
- `incremental_revenue = new_revenue - baseline_revenue`
- `incremental_margin = new_margin - baseline_margin`
- `promo_cost = baseline_units * base_price * discount`
- `roi = incremental_margin / promo_cost`

Decision cutoff:

- `ROI > 1` -> `SCALE`
- `0 <= ROI <= 1` -> `OPTIMIZE`
- `ROI < 0` -> `STOP`

See [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:2889).

### 4.4 Incrementality / cannibalization logic

Implemented in [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:994).

Definitions:

- `promo_dependency = promo_orders / total_orders_in_scope`
- `elasticity_factor = min(abs(weighted_elasticity), 2.0)`
- `cannibalization_rate = clip(0.2 + 0.3*promo_dependency + 0.2*(1 - elasticity_factor/2), 0.1, 0.6)`
- `uplift_units = new_units - baseline_units`
- `true_incremental = uplift_units * (1 - cannibalization_rate)`
- `cannibalized = uplift_units * cannibalization_rate`
- `pull_forward = uplift_units * 0.10`

Notes:

- `pull_forward` uses a fixed `10%` rate.
- If `uplift_units` is negative, these components also go negative.

### 4.5 Inventory / returns logic

Implemented in [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:1026).

Logic:

- baseline period = `4 weeks`
- `weekly_sales = baseline_units / 4`
- `new_weekly_sales = new_units / 4`
- consistent inventory is anchored to `weeks_of_cover_before * weekly_sales`
- `remaining_inventory = max(consistent_inventory - new_units, 0)`
- `weeks_of_cover_after = remaining_inventory / max(new_weekly_sales, weekly_sales)`
- `inventory_reduction_pct = (consistent_inventory - remaining_inventory) / consistent_inventory`

### 4.6 Scorecard logic

Implemented in [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:3254).

Inputs:

- incremental revenue
- incremental margin
- ROI
- true incrementality %
- return rate
- margin floor pass/fail

Weights:

- Revenue impact: `40%`
- Margin impact: `20%`
- ROI: `15%`
- Quality / true incrementality: `10%`
- Returns: `10%`
- Margin floor pass: `5%`

The score is normalized to a **0-100** label:

- `>= 85` -> `Excellent`
- `>= 70` -> `Strong`
- `>= 50` -> `Acceptable`
- else -> `Risky`

---

## 5. Promotion-Type Logic That Exists in Code

These functions exist and are used by the optimizer path via `simulate_scenario()`.

### `%_OFF`

File: [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:1995)

- Uses the decision model directly
- Baseline vs promo row scoring
- ROI based on discount cost

### `BUNDLE`

File: [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:2272)

Key constants:

- `BUNDLE_AOV_LIFT = 10%`
- `BUNDLE_BASKET_LIFT = 15%`

Logic:

- compute historical basket size and basket value
- increase basket size by `15%`
- increase basket value by `10%`
- optional `cheapest_free` subtracts cheapest item average
- derive `discount_per_unit`
- derive `effective_discount`
- compute new units/revenue/margin from basket economics

### `FIRST_TIME`

File: [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:2436)

Key constants:

- `FIRST_TIME_CONV_COEF = 0.50`
- `FIRST_TIME_LTV_LIFT = 15%`
- `FIRST_TIME_REPEAT_RATE = 35%`
- `FIRST_TIME_AVG_ORDERS_PER_YEAR = 3`
- `FIRST_TIME_LTV_DISCOUNT_FACTOR = 0.50`

Logic:

- identify new customers from first-order dates
- use elasticity to lift conversion:
  - `conv_lift = 1 + abs(elasticity) * discount * 0.50`
- compute new customers acquired
- reduce AOV by the discount
- compute incremental LTV
- ROI includes:
  - current-period margin lift
  - plus `50%` of incremental LTV

### `CLEARANCE`

File: [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:2598)

Key constants:

- `ELASTICITY_COEFF_CLEARANCE = 0.60`
- default `inventory_trigger_weeks = 8`

Logic:

- identify inventory above trigger
- use stronger elasticity-based lift:
  - `clearance_lift = 1 + min(abs(elasticity), 3.0) * discount * 0.60`
- compute sell-through %
- special decision rule:
  - if `margin_rate < 0` -> `STOP`
  - else if `sell_through_pct >= 50%` and `ROI >= 0.30` -> `SCALE`
  - else -> `OPTIMIZE`

### `FREE_SHIPPING`

File: [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:2740)

Key constants:

- `FREE_SHIP_BASE_LIFT = 10%`
- `FREE_SHIP_THRESHOLD_LIFT = 15%`
- `FREE_SHIP_AOV_LIFT = 5%`
- `FREE_SHIP_COST_DTC = 8.95`
- `FREE_SHIP_RETURN_RATE_CHANGE = 0.005`

Logic:

- compute qualifying share of orders above threshold
- conversion lift:
  - `1 + 10% + 15% if threshold > 0`
- AOV lift:
  - `+5%`
- shipping cost only charged for DTC
- ROI:
  - `incremental_margin / shipping_cost_total`

---

## 6. Optimizer and Recommendation Logic

### Optimizer

Implemented in [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:3767).

Flow:

1. Generate grid over product x channel x segment x promo_type x discount
2. Simulate each cell
3. Apply hard constraints:
   - `margin_rate >= min_margin_rate`
   - `return_rate <= max_return_rate`
4. Rank surviving scenarios

### Recommendation ranking

Implemented in [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:3914).

Guardrails:

- `ROI > 0.3`
- `true_incremental > 0`
- `return_rate < 0.5`
- `decision != STOP`

Important current issue:

In `compute_recommendations()`, the code simulates only the first promo type (`promo_values[0]`, effectively `%_OFF`) and then just rewrites `row["promo_type"] = promo` for the other promo labels. See:

- [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:3985)
- [step07_promotion_simulation.py](/C:/Users/admin/work/avantika/spanx/backend/screens/step07_promotion_simulation.py:4000)

So the recommendation table is **not truly promo-type-specific today**.

---

## 7. Important Caveats You Should Explicitly Say

These are worth saying directly to Ritesh because they are true in the code today.

1. The **decision model is real and trained**, but its quality is uneven.
   Revenue, conversion, and AOV are strong; units and return-rate prediction are weak.

2. The **main simulate screen is not fully honoring promo-type selection**.
   The wrapper currently routes through `%_OFF` logic.

3. A **`0% discount` input is not preserved correctly** in the request merge path; it falls back to default `20%`.

4. The **recommendation table duplicates `%_OFF` results across promo labels** instead of re-simulating each type.

5. The validation report already says:
   “Arithmetic consistency passed, but behavioral/model-quality checks need improvement.”
   See [.generated_data/decision_model_validation_report.json](/C:/Users/admin/work/avantika/spanx/.generated_data/decision_model_validation_report.json:1).

---

## 8. Concrete Examples

### Example A: Current `%_OFF` simulation result

Scenario:

- Product: `SPANXshape Everyday Leggings`
- Channel: `Amazon`
- Segment: `New Customers`
- Promo: `15% off`
- Date: `2026-05-04`

Observed output from the active simulate path:

- Baseline revenue: `68.29`
- New revenue: `60.76`
- Revenue change: `-11.0%`
- Baseline margin: `24.63`
- New margin: `20.27`
- ROI: `-0.32`
- Decision: `STOP`

Interpretation:

- The model predicts the promo reduces revenue and margin.
- Because ROI is negative, the engine classifies it as `STOP`.

### Example B: Type-specific `FIRST_TIME` logic via simulator dispatch

Same leggings example, `FIRST_TIME`, `15%`, `Amazon`, `New Customers`, `2026-05-04`.

Output from the actual `simulate_first_time()` logic:

- Baseline revenue: `93.60`
- New revenue: `96.63`
- Baseline margin: `42.25`
- New margin: `49.27`
- Incremental LTV: `24.24`
- ROI: `1.12`
- Decision: `SCALE`

Interpretation:

- This promo type behaves differently because the simulator adds **new-customer conversion lift + LTV economics**.

### Example C: Type-specific `FREE_SHIPPING` logic via simulator dispatch

Scenario:

- Product: `SPANXshape Everyday Leggings`
- Channel: `DTC`
- Segment: `Deal Hunters`
- Free shipping threshold: `$75`
- Date: `2026-05-04`

Output from `simulate_free_shipping()`:

- Baseline revenue: `215.16`
- New revenue: `282.40`
- New margin: `146.86`
- Shipping cost total: `17.90`
- ROI: `7.20`
- Decision: `SCALE`

Interpretation:

- For DTC, free shipping can score well because the logic lifts orders and AOV, then subtracts shipping cost explicitly.

---

## 9. Suggested Friday Talk Track

You can say this almost verbatim:

“Let me explain how the promotion simulation engine is built today. The underlying model is a daily product-channel-segment decision model trained on transactions, products, customers, inventory, promotions, demand signals, weather, events, and holidays. It predicts five outputs directly: units sold, revenue, return rate, conversion rate, and AOV.

At runtime, the engine builds a baseline scenario and a promo scenario for the same product/channel/segment/date, scores both with the model, and then calculates incremental revenue, incremental margin, promo cost, ROI, incrementality, cannibalization, pull-forward, returns impact, and inventory impact. The main decision rule is simple: ROI above 1 is SCALE, ROI between 0 and 1 is OPTIMIZE, and ROI below 0 is STOP.

For percent-off scenarios, that flow is what is actively driving the screen today. We also have separate code paths for bundle, first-time, clearance, and free-shipping logic. Those type-specific formulas are correctly used by the optimizer path, but the main simulate payload currently has a wiring issue and often falls back to percent-off logic even when another promo type is selected.

So the short version is: the model and business-math engine are real, the optimizer path is closer to the intended final design, but the simulate screen still has a few implementation gaps we should close, especially around promo-type routing, zero-discount handling, and recommendation-table logic.” 

---

## 10. If They Ask “Can We Trust It?”

Best honest answer:

- Trust it more for **revenue direction, conversion, and AOV**.
- Be more cautious on **units and return-rate precision**.
- Treat current outputs as a **decision-support system**, not a fully calibrated finance-approved forecast.
- The math is internally consistent, but the current wiring and model-quality gaps should be fixed before calling it a final production-grade promotion engine.

