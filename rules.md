# Promotion Simulation — Rules & Calculation Logic

> Single source of truth for every value computed by the Promotion Simulation screen.
> For each metric: **(1) which dataset(s) it reads, (2) the joins used, (3) the exact calculation/derivation**.
>
> Cross-references:
> - UI / spec → `promotion_simulation.md`
> - Backend reference implementation → `backend/screens/step07_promotion_simulation.py`
> - ML artifact → `model_artifacts/promo_regression_daily_artifacts.pkl`
> - Source datasets → `data/*.xlsx` (described in `dataset_description.md`)

---

## 0. Notation & shared symbols

| Symbol | Meaning |
| --- | --- |
| `D` | Scenario date (from the UI date picker; clamped to `≥ 2026-04-13`). |
| `W4` | Last 4 calendar weeks ending at `max(transactions.order_date)`. |
| `discount` | `discount_depth / 100`, where `discount_depth ∈ [0, 80]`. |
| `base_price` | `product_master.base_price` for the selected product. |
| `cost_price` | `product_master.cost_price` (fallback to `transactions.cost_price` if `> 0`). |
| `scope` | The filtered `transactions` slice (last 4 weeks ∩ selected product / channel / segment). |
| `weights[s]` | Segment weight vector when `All Segments` is selected. |
| `clip(x, lo, hi)` | `max(lo, min(hi, x))`. |
| `safe_div(a, b)` | `a/b` if `|b| ≥ 1e-9`, else `0`. |

---

## 1. Input normalisation rules

### 1.1 `scenario_date`
- **Source:** UI date input (`ps9-date`) → `scenario.date`.
- **Rule:**
  ```
  parsed = pd.to_datetime(value, errors="coerce")
  if NaT or parsed < 2026-04-13:
      parsed = 2026-04-13
  scenario_date = parsed.normalize().strftime("%Y-%m-%d")
  ```

### 1.2 `discount`
- **Source:** UI slider `ps9-discount-depth`.
- **Rule:** clipped to `[0, 80]` (Clearance) or `[0, 30]` (% Off / First Time / Bundle implied).
  ```
  discount_depth = clip(value, 0, 80)
  discount       = discount_depth / 100
  ```

### 1.3 `channel` normalisation (lookup table)
- **Source:** UI multi-select; matched to `transactions.channel` and `elasticity.channel`.
- **Rule:** case-insensitive key match against:
  ```
  "amazon"           → "Amazon"
  "dtc" | "website" → "DTC"
  "nordstrom"       → "Nordstrom"
  "bloomingdales"   → "Bloomingdales"
  ```

### 1.4 `segment` normalisation
- **Source:** UI multi-select; matched to `customer_master.segment_name`.
- **Rule:** map raw labels to canonical:
  ```
  "fitsensitivebuyers" | "fitsensitive" → "Fit-Sensitive"
  "deal hunters"                        → "Deal Hunters"
  "premium loyalists"                   → "Premium Loyalists"
  "new customers"                       → "New Customers"
  ""                                    → "All Segments"
  ```

### 1.5 `discount_pct` normalisation in `transactions`
- **Source:** `transactions.discount_pct` may be stored as `15` or `0.15`.
- **Rule:**
  ```
  if |x| > 1: x = x / 100
  ```

---

## 2. Scope construction (last 4 weeks)

### 2.1 `scope` (the baseline transaction slice)
- **Datasets:** `transactions`, `product_master`, `customer_master`.
- **Joins (in this order):**
  ```
  transactions
      LEFT JOIN product_master   ON transactions.product_id  = product_master.product_id
      LEFT JOIN customer_master  ON transactions.customer_id = customer_master.customer_id
  ```
- **Filter:**
  ```
  scope =
      max_date = max(order_date) over transactions
      min_date = max_date − 4 weeks
      WHERE order_date BETWEEN min_date AND max_date
        AND product_id ∈ selected_products
        AND channel    ∈ selected_channels
        AND (segment   ∈ selected_segments  OR  selected_segments = "All Segments")
  ```
- **Optional filter (Additional Conditions → "Exclude existing promo products"):**
  ```
  AND promo_flag = 0       # transactions.promo_id is null/blank
  ```

### 2.2 `order_denominator`
- **Calculation:** `unique(scope.order_id)` — used in promo-dependency, return-rate, and AOV.

### 2.3 `weights[s]` (segment weighting)
- **When:** UI `segment == "All Segments"`.
- **Calculation:**
  ```
  counts[s] = number of rows in scope where segment = s
  total     = Σ counts[s]
  weights[s] = counts[s] / total          (if total > 0)
  ```
- **Fallback:** if `total = 0`, distribute uniformly across the 4 known segments
  (`Deal Hunters`, `Premium Loyalists`, `Fit-Sensitive`, `New Customers`).

---

## 3. Baseline metrics (rendered in the "Baseline" KPI card)

### 3.1 `baseline_units_seg` (per segment)
- **Datasets:** ML artifact `promo_regression_daily_artifacts.pkl` + `product_master`.
- **Steps:**
  1. Pick the most recent base row from `artifacts.scored_frame` that matches
     `(product_id, category, channel, segment, ≤ D)` — with cascading fallbacks
     `(product_id, channel, segment) → (product_id, channel) → (product_id, segment) → (product_id) → (category, channel, segment) → (category) → ()`.
  2. Build the feature row by overriding:
     ```
     gross_price  = base_price
     discount_pct = 0
     promo_flag   = 0
     date         = D
     trend        = base_row.trend + days_since(base_row.date, D)
     inventory    = base_row.inventory  (default 0.85)
     social_buzz  = base_row.social_buzz (default 1.0)
     units        = max(base_row.units, 1.0)
     plus the context features (day_of_week, is_weekend, month, week_of_year, is_holiday, days_to_holiday)
     ```
  3. `baseline_revenue_pred = trained_model.predict(features)` → clamped `≥ 0`.
  4. `baseline_units_seg    = safe_div(baseline_revenue_pred, base_price)`.

### 3.2 `baseline_units` (rolled up)
- **Calculation:**
  ```
  baseline_units = Σ_s  baseline_units_seg × weights[s]
  baseline_units = max(baseline_units, 0)
  ```

### 3.3 `baseline_revenue`
- **Calculation:**
  ```
  baseline_revenue = baseline_units × base_price
  ```

### 3.4 `baseline_margin`
- **Datasets:** `product_master.cost_price`.
- **Calculation:**
  ```
  baseline_margin = (base_price − cost_price) × baseline_units
  ```

### 3.5 `baseline_return_rate`
- **Datasets:** `returns ⨝ scope` on `order_id`.
- **Calculation:**
  ```
  return_orders        = unique(returns.order_id ∩ scope.order_id)
  baseline_return_rate = safe_div(count(return_orders), order_denominator)
  ```

### 3.6 `baseline_aov` (used by Bundle / Free Shipping)
- **Source:** `scope`.
- **Calculation:**
  ```
  basket_value_per_order = scope GROUP BY order_id → SUM(net_price)
  baseline_aov           = mean(basket_value_per_order)
  ```

### 3.7 `basket_size_avg` (Bundle)
- **Calculation:**
  ```
  basket_size_per_order = scope GROUP BY order_id → COUNT(*)
  basket_size_avg       = mean(basket_size_per_order)
  ```

### 3.8 `cheapest_avg` (Bundle "cheapest item is free")
- **Calculation:**
  ```
  cheapest_per_order = scope GROUP BY order_id → MIN(net_price)
  cheapest_avg       = mean(cheapest_per_order)
  ```

---

## 4. Predicted (under-promo) units & revenue

### 4.1 `predicted_units_seg`
- **Datasets:** ML artifact + `product_master`.
- **Calculation:**
  ```
  promo_features                = baseline_features
  promo_features.gross_price    = base_price × (1 − discount)
  promo_features.discount_pct   = discount
  promo_features.promo_flag     = 1
  promo_revenue_pred            = trained_model.predict(promo_features)   (clamped ≥ 0)
  promo_price                   = max(base_price × (1 − discount), 1e-9)
  predicted_units_seg           = safe_div(promo_revenue_pred, promo_price)
  ```

### 4.2 `elasticity_val` (per segment)
- **Datasets:** `elasticity.xlsx`.
- **Joins:** filter only — no SQL join.
  ```
  scoped = elasticity[
      channel  = scenario.channel
      AND category = scenario.category
      AND segment  = segment_name
  ]
  if scoped has rows for product_id: scoped = scoped[product_id = product_id]
  ```
- **Calculation:**
  ```
  elasticity_val = mean(scoped.elasticity)
  fallback (no rows): elasticity_val = -1.0
  ```

### 4.3 `elasticity_boost`
- **Calculation (% Off / First Time / Bundle):**
  ```
  elasticity_boost = 1 + |elasticity_val| × discount × 0.30
  ```
- **Calculation (Clearance — steeper):**
  ```
  elasticity_boost = 1 + min(|elasticity_val|, 3.0) × discount × 0.60
  ```

### 4.4 `new_units_seg`
- **Calculation:**
  ```
  new_units_seg = predicted_units_seg × elasticity_boost
  ```

### 4.5 `new_units` (rolled up)
- **Calculation:**
  ```
  new_units = max( Σ_s new_units_seg × weights[s], 0 )
  ```

### 4.6 `weighted_elasticity`
- **Calculation:**
  ```
  weighted_elasticity = Σ_s elasticity_val[s] × weights[s]
  ```

---

## 5. Scenario output metrics (Scenario card)

### 5.1 `new_price`
```
new_price = base_price × (1 − discount)
```

### 5.2 `new_revenue`
```
new_revenue = new_units × new_price
```

### 5.3 `new_margin`
```
new_margin = (new_price − cost_price) × new_units
```

### 5.4 `incremental_units`
```
incremental_units = new_units − baseline_units
```

### 5.5 `incremental_revenue`
```
incremental_revenue = new_revenue − baseline_revenue
```

### 5.6 `incremental_margin`
```
incremental_margin = new_margin − baseline_margin
```

### 5.7 `promo_cost`
```
promo_cost = baseline_units × base_price × discount
```
**Interpretation:** the discount $ amount we "give up" on the assumed-baseline volume.

### 5.8 `ROI`
```
ROI = safe_div(incremental_margin, promo_cost)
```

### 5.9 `margin_rate`
```
margin_rate = safe_div(new_margin, new_revenue)
```

### 5.10 `new_units_change_pct`, `new_revenue_change_pct`, `new_margin_change_pct`
```
new_units_change_pct   = safe_div(incremental_units,   baseline_units)
new_revenue_change_pct = safe_div(incremental_revenue, baseline_revenue)
new_margin_change_pct  = safe_div(incremental_margin,  baseline_margin)
```

### 5.11 `decision`
```
decision =
    "SCALE"    if ROI > 1
    "OPTIMIZE" if 0 ≤ ROI ≤ 1
    "STOP"     if ROI < 0
```
*(Optimizer-side rule in §10 uses a stricter `OPTIMIZE if 0.3 ≤ ROI ≤ 1`.)*

---

## 6. Incrementality vs Cannibalization

### 6.1 `promo_dependency`
- **Datasets:** `scope`.
- **Calculation:**
  ```
  promo_orders     = unique(scope.order_id WHERE promo_flag = 1)
  promo_dependency = safe_div(count(promo_orders), order_denominator)
  ```

### 6.2 `cannibalization_rate`
```
e_factor             = min(|weighted_elasticity|, 2.0)
cannibalization_rate = clip(0.20 + 0.30 × promo_dependency
                                 + 0.20 × (1 − e_factor / 2),
                            0.10, 0.60)
```

### 6.3 `true_incremental`, `cannibalized`, `pull_forward`
```
uplift_units     = incremental_units
true_incremental = uplift_units × (1 − cannibalization_rate)
cannibalized     = uplift_units × cannibalization_rate
pull_forward     = uplift_units × 0.10
```

### 6.4 Percent breakdown (for the stacked bar)
```
total = true_incremental + cannibalized + pull_forward
true_incremental_pct = safe_div(true_incremental, total)
cannibalized_pct     = safe_div(cannibalized,     total)
pull_forward_pct     = safe_div(pull_forward,     total)
```

---

## 7. Returns impact

### 7.1 `return_sensitivity`
- **Rule:** category-driven constant.
  ```
  return_sensitivity = 0.20 if category == "Bras" else 0.10
  ```

### 7.2 `new_return_rate`
```
new_return_rate = min( baseline_return_rate + discount × return_sensitivity, 0.60 )
```

### 7.3 `return_rate_change`
```
return_rate_change = new_return_rate − baseline_return_rate
```

### 7.4 `incremental_returns_units`
- **Calculation (used by output KPI tile):**
  ```
  incremental_returns_units = new_units × new_return_rate
                            − baseline_units × baseline_return_rate
  ```

---

## 8. Inventory impact

### 8.1 `inventory_snapshot`
- **Datasets:** `inventory.xlsx`.
- **Joins (lookup, not SQL):**
  ```
  inventory  filtered to  norm_key(Product) = norm_key(scenario.product)
  AND        week_start    = MAX(week_start)
  ```

### 8.2 `current_inventory`
```
current_inventory = SUM(inventory_snapshot.current_stock)     # 0 if no rows
```

### 8.3 `weekly_sales`
```
weekly_sales = baseline_units / 4
```

### 8.4 `weeks_of_cover_before`
- **Rule:** prefer the dataset's stored value, else compute from stock.
  ```
  if inventory_snapshot.weeks_of_cover.notna().any():
      weeks_of_cover_before = mean(inventory_snapshot.weeks_of_cover)
  else:
      weeks_of_cover_before = safe_div(current_inventory, weekly_sales)
  ```

### 8.5 `remaining_inventory`
```
remaining_inventory = current_inventory − new_units
```

### 8.6 `weeks_of_cover_post`
```
weeks_of_cover_post = safe_div(max(remaining_inventory, 0), weekly_sales)
```

### 8.7 `weeks_of_cover_change`
```
weeks_of_cover_change = weeks_of_cover_post − weeks_of_cover_before
```

### 8.8 `inventory_reduction_pct`
```
inventory_reduction_pct =
    safe_div(current_inventory − max(remaining_inventory, 0), current_inventory) × 100
```

### 8.9 `inventory_insight` (string)
```
if inventory_reduction_pct ≥ 20:
    "Inventory will reduce by X% (good for clearance)."
elif inventory_reduction_pct ≥ 0:
    "Inventory will reduce by X%."
else:
    "Inventory is expected to increase by X%."
```

---

## 9. Type-specific overrides

### 9.1 % Discount (default — uses §3–§8 unchanged)

### 9.2 Bundle — Mix & Match
| Value | Rule |
| --- | --- |
| `aov_lift_factor` | constant `1 + 0.10` |
| `basket_size_lift` | constant `1 + 0.15` |
| `new_basket_size_avg` | `basket_size_avg × basket_size_lift` |
| `new_basket_value_avg` | `basket_value_avg × aov_lift_factor` |
| If `cheapest_item_is_free`: | `new_basket_value_avg -= cheapest_avg` |
| `discount_per_unit` | `(basket_value_avg − new_basket_value_avg) / new_basket_size_avg` |
| `effective_discount` | `discount_per_unit / base_price` (replaces `discount` in §6 & §7) |
| `new_unit_price` | `base_price − discount_per_unit` |
| `new_units` | `new_basket_size_avg × baseline_orders` |
| `new_revenue` | `baseline_orders × new_basket_value_avg` |
| `new_margin` | `(new_unit_price − cost_price) × new_units` |
| `qualifying_share` (if `min_cart_value` set) | `count(orders WHERE basket_value ≥ min_cart_value) / total_orders` |
| Final scaling | `incremental_* ×= qualifying_share` |

### 9.3 First Time Offer — New Customers
| Value | Rule |
| --- | --- |
| `first_order_date_per_cust` | `transactions GROUP BY customer_id → MIN(order_date)` |
| `new_customers` | `customers WHERE first_order_date ≥ D − duration` |
| `scope_new` | `transactions[customer_id ∈ new_customers]` |
| `baseline_new_customers` | `count(new_customers in last 4 weeks)` |
| `baseline_aov_new` | `mean(net_price per first order)` |
| `baseline_cac` | `marketing_spend / baseline_new_customers`  *(constant input)* |
| `conv_lift` | `1 + |elasticity[New Customers]| × discount × 0.50` |
| `new_new_customers` | `baseline_new_customers × conv_lift` |
| `new_aov_new` | `baseline_aov_new × (1 − discount)` |
| `new_revenue_new` | `new_new_customers × new_aov_new` |
| `ltv_uplift_factor` | constant `1.15` |
| `ltv_per_customer` | `baseline_aov_new × repeat_rate × avg_orders_per_year × ltv_uplift_factor` |
| `incremental_ltv` | `(new_new_customers − baseline_new_customers) × ltv_per_customer` |
| `promo_cost` | `new_new_customers × baseline_aov_new × discount` |
| `ROI` | `(incremental_margin + incremental_ltv × discount_factor) / promo_cost` |
| Min-order-value gate | scale `new_new_customers` by historical share of orders ≥ `min_order_value` |

### 9.4 Clearance Markdown
| Value | Rule |
| --- | --- |
| Candidate set | `inventory[Product ∈ selected, weeks_of_cover > inventory_trigger_weeks, week_start = MAX(week_start)]` |
| `clearance_lift_factor` | `1 + min(|elasticity_val|, 3.0) × discount × 0.60` |
| `new_units_seg` | `predicted_units_seg × clearance_lift_factor` |
| `sell_through_pct` | `new_units / current_stock` |
| `weeks_of_cover_post` | `max(current_stock − new_units, 0) / weekly_sales` |
| Margin floor | if `margin_rate < 0`: `decision = "STOP"` |
| Decision | `"SCALE"` if `sell_through_pct ≥ 0.50 AND ROI ≥ 0.3` else `"OPTIMIZE"` |
| Final-sale toggle | If ON, `return_sensitivity = 0` (returns frozen at `baseline_return_rate`) |

### 9.5 Free Shipping
| Value | Rule |
| --- | --- |
| `qualifying_share` | `count(orders WHERE basket_value ≥ threshold) / total_orders` |
| `conv_lift_factor` | `1 + 0.10 + 0.15 × indicator(threshold > 0)` |
| `new_orders` | `baseline_orders × (1 + qualifying_share × (conv_lift_factor − 1))` |
| `aov_lift_factor` | `1 + 0.05` |
| `new_aov` | `baseline_aov × aov_lift_factor` |
| `new_revenue` | `new_orders × new_aov` |
| `avg_shipping_cost_per_order` | `8.95` for DTC, `0` for marketplaces |
| `shipping_cost_total` | `qualifying_orders × avg_shipping_cost_per_order` |
| `incremental_revenue` | `new_revenue − baseline_revenue` |
| `incremental_margin` | `(new_revenue × baseline_margin_rate) − shipping_cost_total` |
| `ROI` | `incremental_margin / shipping_cost_total` |
| `return_rate_change` | `+0.005` (constant 0.5pp) |

---

## 10. Optimizer & Recommendation rules (Run Variations)

### 10.1 Scenario generator
```
scenarios = cartesian_product(
    selected_products,
    selected_channels,
    selected_segments,
    selected_promo_types,
    [10, 15, 20, 25, 30]            # discount sweep
)
```

### 10.2 Constraint engine
```
valid    = scenarios WHERE margin_rate  ≥ constraints.min_margin_rate
                       AND return_rate ≤ constraints.max_return_rate
rejected = scenarios − valid

violation_reason[row] = {
    "margin_fail": row.margin_rate  < constraints.min_margin_rate,
    "return_fail": row.return_rate > constraints.max_return_rate
}
```

### 10.3 Score (per scenario, objective-aware)
```
quality_score     = safe_div(true_incremental, incremental_revenue + 1e-6)
return_penalty    = return_rate
inventory_penalty = weeks_of_cover_post

if objective == "profit":
    score = 0.35 × incremental_margin
          + 0.25 × ROI
          + 0.15 × quality_score
          + 0.15 × incremental_revenue
          − 0.05 × return_penalty
          − 0.05 × inventory_penalty
else:                                       # objective == "revenue"
    score = 0.40 × incremental_revenue
          + 0.20 × incremental_margin
          + 0.15 × ROI
          + 0.15 × quality_score
          − 0.05 × return_penalty
          − 0.05 × inventory_penalty
```

### 10.4 Best & top-5 selection
```
ranked = valid SORT BY score DESC
best   = ranked[0]
top5   = ranked[0:5]
```

### 10.5 Constraint penalty (soft-mode alternative — optional)
```
penalty = 0
penalty += max(0, constraints.min_margin_rate − margin_rate) × 100
penalty += max(0, return_rate − constraints.max_return_rate) × 100
score_soft = score − penalty
```

### 10.6 Confidence label
```
confidence = "HIGH"   if best.ROI > 1
           = "MEDIUM" otherwise
```

### 10.7 Recommendation payload
```
recommendation = {
    "action":  "Run {promo_type} on {product} at {discount}%",
    "channel": best.channel,
    "segment": best.segment,
    "impact":  { "revenue":   round(best.incremental_revenue, 0),
                 "margin":    round(best.incremental_margin, 0),
                 "ROI":       round(best.ROI, 2),
                 "return_rate": round(best.return_rate, 2) },
    "confidence": confidence
}
```

### 10.8 Explainability
```
why_selected = [
    "Meets margin constraint ({margin_rate})",
    "Return rate within limit ({return_rate})",
    "High incremental revenue (${incremental_revenue})",
    "ROI = {ROI}"
]
why_rejected = [
    "{N} scenarios rejected due to constraint violations"
]   if rejected non-empty
```

---

## 11. Output-screen aggregations (Scenario Output card)

### 11.1 Top KPI tile values
| Tile | Calculation |
| --- | --- |
| Incremental Revenue $ | `incremental_revenue` (5.5) |
| Incremental Revenue % | `new_revenue_change_pct` (5.10) |
| Incremental Margin $ | `incremental_margin` (5.6) |
| Incremental Margin % | `new_margin_change_pct` (5.10) |
| Incremental Units | `incremental_units` (5.4) |
| Incremental Units % | `new_units_change_pct` (5.10) |
| Incremental Returns | `incremental_returns_units` (7.4) |
| Incremental Returns % | `return_rate_change × 100` (7.3) |

### 11.2 Impact Overview chart pairs (Baseline vs Scenario)
| Bar pair | Baseline | Scenario |
| --- | --- | --- |
| Revenue | `baseline_revenue` | `new_revenue` |
| Gross Margin | `baseline_margin` | `new_margin` |
| Units | `baseline_units` | `new_units` |
| AOV | `baseline_aov` | `new_revenue / new_orders` |
| Conversion Rate | `count(scope.order_id) / count(unique scope.customer_id sessions)` *(approx via transactions)* | scaled by `conv_lift_factor` |
| Repeat Rate | `count(customers with ≥ 2 orders in W4) / count(customers in W4)` | `× ltv_uplift_factor` (First Time only, else unchanged) |

### 11.3 Customer Segment Impact
- **Datasets:** `scope`, joined to ML predictions per segment.
- **Calculation per segment `s`:**
  ```
  baseline_revenue_seg = baseline_units_seg × base_price
  new_revenue_seg      = new_units_seg × new_price
  lift_seg             = safe_div(new_revenue_seg − baseline_revenue_seg, baseline_revenue_seg)
  ```

### 11.4 Category Impact
- **Datasets:** `scope`, `product_master`.
- **Joins:** `scope ⨝ product_master ON product_id`.
- **Calculation per category `c`:**
  ```
  baseline_revenue_c = SUM(net_price WHERE category = c)  over scope
  new_revenue_c      = baseline_revenue_c × (1 + new_revenue_change_pct)
  lift_c             = safe_div(new_revenue_c − baseline_revenue_c, baseline_revenue_c)
  ```
- The donut headline value:
  ```
  donut_total = SUM_c (new_revenue_c − baseline_revenue_c)  =  incremental_revenue
  ```

### 11.5 Channel Impact
- **Datasets:** `scope`.
- **Calculation per channel `ch`:**
  ```
  baseline_revenue_ch = SUM(net_price WHERE channel = ch)  over scope
  share_ch            = baseline_revenue_ch / SUM(baseline_revenue_ch)
  incremental_ch      = incremental_revenue × share_ch
  lift_ch             = safe_div(incremental_ch, baseline_revenue_ch)
  ```

### 11.6 Scenario Scorecard composite
```
target_revenue = constraints.target_revenue   (default = 1.10 × baseline_revenue)
target_margin  = constraints.target_margin    (default = 1.05 × baseline_margin)
max_return_rate = constraints.max_return_rate (default = 0.25)

revenue_term     = clip(incremental_revenue / target_revenue,    0, 1.25)
margin_term      = clip(incremental_margin  / target_margin ,    0, 1.25)
roi_term         = clip(ROI / 1.0,                                0, 1.25)
quality_term     = clip(true_incremental_pct,                     0, 1.00)
returns_term     = 1 − clip(return_rate / max_return_rate,        0, 1.00)
margin_floor     = 1 if margin_rate ≥ constraints.min_margin_rate else 0

score = 100 × ( 0.40 × revenue_term
               + 0.20 × margin_term
               + 0.15 × roi_term
               + 0.10 × quality_term
               + 0.10 × returns_term
               + 0.05 × margin_floor )

label =
    "Excellent"   if score ≥ 85
    "Strong"      if 70 ≤ score < 85
    "Acceptable"  if 50 ≤ score < 70
    "Risky"       if score < 50
```

### 11.7 Scorecard sub-bars (each 0–100, weighted slice of `score`)
```
Revenue Impact        = 100 × revenue_term  / 1.25
Margin Impact         = 100 × margin_term   / 1.25
Customer Reach        = 100 × clip(new_units / target_units, 0, 1)
Retention Risk        = 100 × returns_term
Inventory Feasibility = 100 × clip(weeks_of_cover_post / weeks_of_cover_target, 0, 1)
```

---

## 12. Context features (used by the ML model)

| Feature | Rule |
| --- | --- |
| `day_of_week` | `D.weekday()` (0=Mon … 6=Sun) |
| `is_weekend` | `1 if D.weekday() ∈ {5,6} else 0` |
| `month` | `D.month` |
| `week_of_year` | `D.isocalendar().week` |
| `is_holiday` | `1 if D.MM-DD ∈ {11-28, 12-25, 10-24} else 0` |
| `days_to_holiday` | `min |D − holiday|` over the 3 holidays in the same year |

`day_targeting` (UI MTWTFSS bitmap) — when ON for any subset, the simulation runs once per active weekday and averages the predicted units, otherwise uses `D` as-is.

---

## 13. Caches & invalidation

- **Frame cache:** `_frame_cache[path] = (mtime, size, df)`. Re-read if `(mtime, size)` differs.
- **Pickle cache:** same shape for the ML artifact.
- **Payload cache key:** `tuple(_path_signature(p) for p in [artifact, transactions, product_master, customer_master, elasticity, returns, inventory])` + scenario inputs.
- **Recommendation cache:** keyed on the same signature + `objective`.
- **Cache purge trigger:** any change to source data files, or Reset button on the UI.

---

## 14. Validation & error handling

| Condition | Rule |
| --- | --- |
| Product not found in `product_master` | raise `ValueError("Product not found: {name}")` → UI shows red toast. |
| No model base row (all fallbacks empty) | raise `ValueError("No model base row available …")` → UI shows red toast. |
| `weekly_sales = 0` | `weeks_of_cover_post = 0` (avoids div-by-zero). |
| `current_inventory = 0` | `inventory_reduction_pct = 0`. |
| `baseline_revenue = 0` | `new_revenue_change_pct = 0`. |
| `promo_cost = 0` (`discount = 0`) | `ROI = 0` (avoids div-by-zero). |
| `baseline_units = 0` | `cannibalization_rate = 0.10` (lower bound). |

---

## 15. Constants registry (single place to tune)

| Constant | Value | Used in |
| --- | --- | --- |
| `MIN_SCENARIO_DATE` | `2026-04-13` | §1.1 |
| `BASELINE_WEEKS` | 4 | §2.1 |
| `ELASTICITY_FALLBACK` | `-1.0` | §4.2 |
| `ELASTICITY_COEFF_DEFAULT` | `0.30` | §4.3 |
| `ELASTICITY_COEFF_CLEARANCE` | `0.60` | §4.3, §9.4 |
| `RETURN_SENSITIVITY_BRAS` | `0.20` | §7.1 |
| `RETURN_SENSITIVITY_DEFAULT` | `0.10` | §7.1 |
| `RETURN_RATE_CAP` | `0.60` | §7.2 |
| `CANNIB_BASE` | `0.20` | §6.2 |
| `CANNIB_PROMO_DEP_COEF` | `0.30` | §6.2 |
| `CANNIB_ELASTICITY_COEF` | `0.20` | §6.2 |
| `CANNIB_LOWER`, `CANNIB_UPPER` | `0.10`, `0.60` | §6.2 |
| `PULL_FORWARD_RATE` | `0.10` | §6.3 |
| `BUNDLE_AOV_LIFT` | `0.10` | §9.2 |
| `BUNDLE_BASKET_LIFT` | `0.15` | §9.2 |
| `FIRST_TIME_CONV_COEF` | `0.50` | §9.3 |
| `FIRST_TIME_LTV_LIFT` | `0.15` | §9.3 |
| `FREE_SHIP_BASE_LIFT` | `0.10` | §9.5 |
| `FREE_SHIP_THRESHOLD_LIFT` | `0.15` | §9.5 |
| `FREE_SHIP_AOV_LIFT` | `0.05` | §9.5 |
| `FREE_SHIP_COST_DTC` | `8.95` | §9.5 |
| `DECISION_SCALE_THRESHOLD` | `1.0` | §5.11 |
| `DECISION_OPTIMIZE_FLOOR` | `0.0` (UI) / `0.3` (optimizer) | §5.11 / §10 |
| `HOLIDAY_MM_DD` | `{11-28, 12-25, 10-24}` | §12 |
| Scoring weights (revenue mode) | `[0.40, 0.20, 0.15, 0.15, 0.05, 0.05]` | §10.3 |
| Scoring weights (profit mode) | `[0.35, 0.25, 0.15, 0.15, 0.05, 0.05]` | §10.3 |
| Scorecard composite weights | `[0.40, 0.20, 0.15, 0.10, 0.10, 0.05]` | §11.6 |
## Implementation alignment note (April 30, 2026)

This file is now aligned to the current implementation, not only the original design intent.

### What is aligned
- The 5 promotion types are implemented in `backend/screens/step07_promotion_simulation.py`: `%_OFF`, `BUNDLE`, `FIRST_TIME`, `CLEARANCE`, `FREE_SHIPPING`.
- The main simulation constants in `PROMO_CONSTANTS` match this document's registry.
- The optimizer, scorecard, assumptions, recommendations, and output payload structure are implemented.
- The Screen 7 builder sends the richer scenario payload described here instead of the older legacy `offer_*` request shape.

### What is not an exact 1:1 match to the original spec
- The UI is close to the screenshots but not a perfect pixel-faithful rebuild. The active implementation is the rebuilt `ps10-*` screen.
- The app prefers `promo_regression_daily_artifacts.pkl`, but due local pandas/pickle compatibility issues it may fall back to `promo_regression_artifacts.pkl` at runtime.
- `Customer Segment Impact`, `Category Impact`, and `Channel Impact` are implemented as scope-level grouped aggregations using the scenario's overall lift, not fully independent ML re-predictions per displayed row.
- The original `Value Seekers` shortcut from the mockup/spec is not treated as a hardcoded guaranteed segment anymore. The UI now only suggests a real selectable segment from the loaded options.
- `Download Report` is implemented as a client-side HTML export, not a backend-generated report pipeline.

### Current UI coupling rules actually implemented
- If the user selects a category first, the product dropdown is filtered to products in that category.
- If the user selects a product first, its category is auto-added to the category selection.
- If categories are narrowed after product selection, products outside those categories are removed from the current selection.

### Conclusion
- The implementation is substantially aligned with the rules and formulas.
- It is not exact in every detail of the original screenshot/spec contract.
- Where implementation and earlier spec diverged, this file should describe the implemented behavior as the source of truth.
