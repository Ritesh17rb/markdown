# Screen 7 ML Model Documentation

## 1. Purpose

The `decision_model.pkl` artifact is the machine learning model used for **Screen 7: Promotion Simulation / Promotion Decision Engine**.

Its job is to predict the likely business outcome of a promotion scenario before execution.

The model predicts these 5 outputs directly:

| Output | Meaning |
|---|---|
| `units_sold` | Expected units sold for the scenario |
| `revenue` | Expected revenue for the scenario |
| `return_rate` | Expected return rate |
| `conversion_rate` | Expected conversion rate |
| `aov` | Expected average order value |

These predictions are then used by Screen 7 to calculate:

| Derived business metric | Formula |
|---|---|
| `baseline_margin` | `baseline_revenue - (cost_price * baseline_units)` |
| `new_margin` | `new_revenue - (cost_price * new_units)` |
| `incremental_units` | `new_units - baseline_units` |
| `incremental_revenue` | `new_revenue - baseline_revenue` |
| `incremental_margin` | `new_margin - baseline_margin` |
| `promo_cost` | `baseline_units * base_price * discount` |
| `roi` | `incremental_margin / promo_cost` |
| `margin_rate` | `new_margin / new_revenue` |

## 2. Final Artifact

| Item | Value |
|---|---|
| Model artifact | `model_artifacts/decision_model.pkl` |
| Metrics file | `.generated_data/decision_model_metrics.json` |
| Training frame | `.generated_data/decision_training_frame.csv` |
| Validation report | `.generated_data/decision_model_validation_report.json` |
| Model type label | `TargetWiseTreeEnsemble` |
| Training grain | `daily_product_channel_segment` |

## 3. Dataset Sources Used

The trainer now prefers the curated `.generated_data/*.csv` files first and only falls back to raw `data/*.xlsx` if needed.

This was important because the raw `transactions.xlsx` file contained duplicated rows that distorted training labels.

### Source datasets

| Dataset | Rows used | Purpose |
|---|---:|---|
| `transactions_52_weeks.csv` | 8000 | historical transaction facts, pricing, promo flags, targets |
| `product_master.csv` | 25 | product/category/base price/cost |
| `customer_master.csv` | 2000 | segment, price sensitivity, promo affinity, return risk, AOV |
| `inventory.csv` | 1300 | stock, inbound stock, sell-through, weeks of cover |
| `promotion_master.csv` | 40 | promotion calendar and promo archetypes |
| `demand_signals.csv` | 780 | sentiment, engagement, influencer signal |
| `weather_report.csv` | 395 | historical weather features for training |
| `event_report.csv` | 8 | event flags |
| `holiday_report.csv` | 8 | holiday flags |

## 4. Training Dataset Construction

The training table is built at:

- `date x product_id x channel x segment`

This is the effective grain:

| Grain component | Description |
|---|---|
| `date` | daily grain |
| `product_id` | product-level behavior |
| `channel` | DTC / Amazon / Nordstrom / Bloomingdales |
| `segment` | customer segment context |

### Target construction

| Target | How it is built |
|---|---|
| `units_sold` | sum of `units` |
| `revenue` | sum of `net_price` |
| `return_rate` | `returned_units / units_sold`, clipped to `[0, 1]` |
| `conversion_rate` | `unique_customers / eligible_customers`, clipped to `[0, 1]` |
| `aov` | `revenue / order_count` |

### Important data handling rules

| Rule | Why it exists |
|---|---|
| Curated generated CSVs are preferred | raw transaction xlsx had duplicate rows |
| transactions are deduplicated defensively | prevents inflated labels |
| channels are normalized | keeps feature values consistent |
| segments are normalized | avoids segment-name fragmentation |
| discount values are normalized to fractional rates | keeps `%` representation consistent |
| promo rows are resolved conservatively | transaction promo IDs and promotion master promo IDs do not share the same namespace |

### Promotion handling

Historical promo rows are not blindly mapped to `promotion_master` by `promo_id`, because the transaction promo IDs and promotion calendar promo IDs do not align directly in this repo.

So the training frame uses conservative promo resolution:

| Case | Promo label used |
|---|---|
| no promo | `NONE` |
| observed promo row with discount | `OBSERVED_PRICE_PROMO` |
| observed promo row without price discount | `OBSERVED_PROMO` |

In the current trained frame, the main promo classes are:

| Promo type | Approx. rows |
|---|---:|
| `NONE` | 5553 |
| `OBSERVED_PRICE_PROMO` | 2383 |

## 5. Feature Architecture

The trained model uses **56 features** total.

### Categorical features

| Categorical feature |
|---|
| `product_id` |
| `product_name` |
| `category` |
| `channel` |
| `segment` |
| `promo_type` |
| `weather_condition` |

### Numeric features

| Numeric feature |
|---|
| `base_price` |
| `gross_price` |
| `discount_pct` |
| `cost_price` |
| `promo_flag` |
| `current_stock` |
| `inbound_stock` |
| `weeks_of_cover` |
| `sell_through_rate` |
| `stockout_flag` |
| `overstock_flag` |
| `inventory_pressure` |
| `inventory_age_days` |
| `price_sensitivity_score` |
| `promo_affinity` |
| `return_risk_score` |
| `customer_aov` |
| `channel_margin` |
| `channel_conversion_baseline` |
| `temperature_c` |
| `rain_chance_pct` |
| `precip_mm` |
| `humidity_pct` |
| `wind_kph` |
| `uv_index` |
| `holiday_flag` |
| `event_flag` |
| `event_count` |
| `social_trend_score` |
| `sentiment_score` |
| `engagement_rate` |
| `influencer_share` |
| `day_of_week` |
| `week_of_year` |
| `month` |
| `is_weekend` |
| `baseline_units_7d` |
| `baseline_units_28d` |
| `baseline_revenue_7d` |
| `baseline_revenue_28d` |
| `baseline_return_rate_28d` |
| `baseline_aov_28d` |
| `baseline_orders_28d` |
| `baseline_unique_customers_28d` |
| `baseline_discount_28d` |
| `discount_x_inventory` |
| `discount_x_segment` |
| `weather_x_channel` |
| `trend_x_category` |

### Feature groups

| Feature group | Included signals |
|---|---|
| Product & price | product, category, base price, gross price, cost |
| Promotion | promo type, promo flag, discount depth |
| Inventory | stock, inbound stock, cover, pressure, stockout/overstock |
| Customer | segment, price sensitivity, promo affinity, return risk, customer AOV |
| Channel | channel, margin baseline, conversion baseline |
| Weather | condition, rain, temperature, humidity, wind, UV |
| Calendar | holiday flag, event flag, event count, weekend |
| Social | sentiment, engagement, influencer activity, social trend |
| Time | day of week, week number, month |
| Historical baselines | recent units, revenue, return rate, AOV, orders, customer counts |
| Interaction features | `discount_x_inventory`, `discount_x_segment`, `weather_x_channel`, `trend_x_category` |

## 6. ML Model Architecture

The implementation is a **target-wise ensemble pipeline**, not one single shared estimator.

### High-level architecture

| Layer | Implementation |
|---|---|
| Preprocessing | `ColumnTransformer` |
| Numeric preprocessing | median imputation |
| Categorical preprocessing | most-frequent imputation + one-hot encoding |
| Multi-target wrapper | custom `TargetWiseRegressor` |
| Final artifact container | `PromoDecisionArtifacts` |

### Per-target estimators

| Target | Estimator | Trees | Max depth | Min samples leaf |
|---|---|---:|---:|---:|
| `units_sold` | `ExtraTreesRegressor` | 300 | 16 | 2 |
| `revenue` | `ExtraTreesRegressor` | 300 | 16 | 2 |
| `return_rate` | `RandomForestRegressor` | 300 | 16 | 2 |
| `conversion_rate` | `RandomForestRegressor` | 300 | 16 | 2 |
| `aov` | `ExtraTreesRegressor` | 300 | 16 | 2 |

### Target transforms

| Target set | Training transform | Prediction inverse |
|---|---|---|
| `units_sold`, `revenue`, `aov` | `log1p(...)` | `expm1(...)` |
| `return_rate`, `conversion_rate` | logit transform | sigmoid |

### Sample weighting

During training, row weights are increased for:

| Weighted condition | Why |
|---|---|
| promo rows | promotion behavior matters more for Screen 7 |
| holiday rows | seasonal context matters |
| event rows | event uplift/risk matters |
| higher discount rows | stronger discount behavior should influence fit more |

## 7. Training / Testing Setup

| Setting | Value |
|---|---|
| Train/test split method | time-based split |
| Train portion | oldest 80% |
| Test portion | newest 20% |
| Training window | `2025-04-14` to `2026-04-13` |
| Train rows | `6348` |
| Test rows | `1588` |

This is important because the model should simulate future-like scenarios instead of learning from random shuffled rows.

## 8. Model Outputs

For each scenario row sent into the model, the output is:

| Output field | Type |
|---|---|
| `units_sold` | float |
| `revenue` | float |
| `return_rate` | float in `[0, 1]` |
| `conversion_rate` | float in `[0, 1]` |
| `aov` | float |

The prediction path used by Screen 7 is:

1. Build a **baseline feature row**
2. Build a **promo feature row**
3. Predict both rows with `decision_model.pkl`
4. Compute incremental business metrics from the two predictions
5. Apply business rules for ROI / margin / returns / recommendation

## 9. Training Results

### Offline metrics

| Target | MAE | R2 |
|---|---:|---:|
| `units_sold` | 0.0155 | -0.0311 |
| `revenue` | 1.1147 | 0.9097 |
| `return_rate` | 0.2265 | -0.2148 |
| `conversion_rate` | 0.0004 | 0.9938 |
| `aov` | 0.1673 | 0.9992 |

### Validation summary

| Check | Status |
|---|---|
| arithmetic consistency | passed |
| behavioral sanity | not fully passed |

### Honest interpretation

| Target quality | Assessment |
|---|---|
| `revenue` | strong |
| `conversion_rate` | strong |
| `aov` | very strong |
| `units_sold` | weak |
| `return_rate` | weak |

This means the current artifact is useful for:

- revenue estimation
- AOV estimation
- conversion estimation

But it is still weaker for:

- precise unit movement
- precise return-rate behavior

## 10. Current Architecture Flow

```text
Curated repo datasets
    ->
load_repo_decision_source_frames()
    ->
build_decision_training_frame()
    ->
feature engineering + target construction
    ->
ColumnTransformer preprocessing
    ->
TargetWiseRegressor
    ->
per-target tree ensemble models
    ->
PromoDecisionArtifacts
    ->
decision_model.pkl
    ->
Screen 7 baseline and promo prediction
    ->
business metric calculation
    ->
ROI / margin / returns / recommendation
```

## 11. Recommended Charts / Tables For Review

For testing, demo, and stakeholder review, these are the best charts to show:

| Chart / table | Purpose |
|---|---|
| R2 by target | show target-wise fit quality |
| MAE by target | show target-wise error magnitude |
| Actual vs predicted scatter plot | show calibration |
| Revenue distribution | show revenue scale and spread |
| Conversion distribution | show conversion behavior |
| Return-rate distribution | show return-rate sparsity/problem |
| Promo vs non-promo comparison | show promotional response |
| Discount sweep chart | show discount sensitivity |
| Real example comparison table | show actual vs predicted values row-by-row |
| Screen 7 scenario summary table | show baseline vs new revenue/margin/ROI |

## 12. Final Summary

| Item | Summary |
|---|---|
| Model used | target-wise tree ensemble |
| Main estimators | `ExtraTreesRegressor` + `RandomForestRegressor` |
| Inputs | 56 engineered features from product, customer, inventory, promo, weather, event, holiday, social, time, and historical baselines |
| Outputs | `units_sold`, `revenue`, `return_rate`, `conversion_rate`, `aov` |
| Training rows | 6348 |
| Test rows | 1588 |
| Strongest targets | `revenue`, `conversion_rate`, `aov` |
| Weakest targets | `units_sold`, `return_rate` |
| Deployment artifact | `model_artifacts/decision_model.pkl` |

