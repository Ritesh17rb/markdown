# Promotion Simulation — UI + Calculation Spec

> **Purpose of this document.** Describe (1) the exact UI/icons that must be rebuilt to match the 6 reference screenshots in `promotion_simulation/`, and (2) the rules, datasets, columns, and step-by-step formulas behind every input, button, and metric on the screen.
>
> Once approved, this is the contract that drives the actual implementation in `frontend/js/steps/step10-promotion-simulation.js` and `backend/screens/step07_promotion_simulation.py`.
>
> Reference images in `promotion_simulation/`:
> - `Image_20260424_101547_109.png` → **Promotion Type 1: % Discount** (default view)
> - `Image_20260424_101547_140.png` → **Promotion Type 2: Bundle (Mix & Match)**
> - `Image_20260424_101547_167.png` → **Promotion Type 3: First Time Offer**
> - `Image_20260424_101547_195.png` → **Promotion Type 4: Clearance Markdown**
> - `Image_20260424_101547_223.png` → **Promotion Type 5: Free Shipping**
> - `Image_20260424_101547_254.png` → **Scenario Output (post-Run Simulation)**

---

## 1. Top-level Page Frame (shared by all 6 screens)

### 1.1 Header strip
- **Title (left):** `Promotion Simulation`
- **Subtitle (left):** `Simulate, compare and optimize promotions before you launch`
- **Stepper (center):** 7 numbered nodes shown horizontally with connector lines.
  1. `Data Foundation` — icon: `bi-database`
  2. `Business Drivers` — icon: `bi-bar-chart-line`
  3. `Growth Strategy` — icon: `bi-graph-up-arrow`
  4. `Customer Strategy` — icon: `bi-people`
  5. `Past Performance` — icon: `bi-clock-history`
  6. `Promotion Studio` — icon: `bi-magic` (this is the **active/highlighted** node — purple pill background)
  7. `Scenarios` — icon: `bi-collection`
- **Top-right button:** `Saved Scenarios` (outline button, icon: `bi-bookmark`).

### 1.2 Two-column body layout
| Column | Content |
| --- | --- |
| **Left (≈ 75% width)** | `Scenario Builder` card → `Offer Details` card → `Timing` card → `Additional Conditions / Scenario Notes` card → action bar (Reset · Save Scenario · Run Simulation). |
| **Right (≈ 25% width)** | Three stacked info cards: `AI Guidance`, `Key Impact Drivers`, `What's Next?`, plus a small `Data Window` card at the bottom with a `Change Window` link. |

### 1.3 Right-panel cards (content varies by promotion type — see §3)

| Card | Static elements |
| --- | --- |
| **AI Guidance** | Header icon `bi-lightbulb`, body = a 1–2 line tip generated for the currently selected promotion type. |
| **Key Impact Drivers** | Header icon `bi-bar-chart`, body = bulleted list (3 bullets). |
| **What's Next?** | Header icon `bi-flag`, body text: *"Run the simulation to see expected impact on revenue, margin, ROI and returns."* |
| **Data Window** | Header `Data Window`, body = baseline window dates (e.g. *"Apr 12, 2026 — Mar 27, 2026"*), link `Change Window`. |

The Data Window dates are derived by taking `max(transactions.order_date)` and subtracting **4 weeks** to get the baseline start.

---

## 2. Scenario Builder (top card — common section across all 5 promotion types)

### 2.1 Header row
- **Section title:** `Scenario Builder`
- **Sub-title:** `Configure your promotion scenario`

### 2.2 First row of inputs
| Field | Control | Default | Storage / source |
| --- | --- | --- | --- |
| **Scenario Name** *(Optional)* | text input | empty (placeholder = type-specific, e.g. `Q2 Leggings Promotion - 20% Off`) | Local state only — persisted only on `Save Scenario`. |
| **Objective** | text input | `Maximize Revenue with Margin > 30%` | Free-text business intent (drives optimizer weights when set to `revenue` vs `profit`; numeric ≥ XX% becomes the `min_margin_rate` constraint). |
| **Scenario Compare** | toggle pill | `Baseline vs Scenario` | When ON, the output screen shows the dual-bar comparison (image 6). |

### 2.3 Promotion Type tabs (icon + label)
A horizontal pill-tab strip with **5 mutually-exclusive options**. Selecting a tab swaps the **Offer Details** card body (§3).

| # | Tab label | Icon (`bi-…`) | Sub-label under icon |
| --- | --- | --- | --- |
| 1 | `% Discount` | `bi-percent` | `% Off Discount` |
| 2 | `Bundle` | `bi-box-seam` | `Mix & Match` |
| 3 | `First time Offer` | `bi-star` | `New customers` |
| 4 | `Clearance Markdown` | `bi-tag` | `Stock Clearance` |
| 5 | `Free Shipping` | `bi-truck` | `Shipping Incentive` |

### 2.4 Second row — scope filters
| Field | Control | Source dataset | Column → label |
| --- | --- | --- | --- |
| **Products** | multi-select with chips | `product_master.xlsx` | `product_name`; chips show product, with category in faint text |
| **Categories** | multi-select with chips | `product_master.xlsx` | unique `category` |
| **Channels** | multi-select with chips | `transactions.xlsx` | unique normalized `channel` (DTC, Amazon, Nordstrom, Bloomingdales) |
| **Customer Segments** | multi-select with chips | `customer_master.xlsx` | unique `segment_name` (Deal Hunters, Premium Loyalists, Fit-Sensitive, New Customers) + `All Segments` |
| **Value Seekers** chip | hardcoded shortcut chip alongside Customer Segments | derived (any segment with median net_price below category median) | adds to Customer Segments list |

### 2.5 Offer Details card (varies by promotion type — see §3)
This is the card where the **type-specific fields** live. The header reads `Offer Details — <Promotion Type Name>`.

### 2.6 Timing card
| Field | Control | Source / formula |
| --- | --- | --- |
| **Date Range** | two date inputs (start / end) | min = `MIN_SCENARIO_DATE = 2026-04-13`. End auto-fills to start + duration. |
| **Duration** | dropdown | options: 1 / 2 / 4 / 6 / 8 / 12 weeks |
| **Day Targeting** *(Optional)* | toggle pills `M T W T F S S` | each day toggles a 0/1 flag passed to `build_context_features` (combined with `is_weekend`) |

### 2.7 Additional Conditions card
| Field | Control | Behaviour |
| --- | --- | --- |
| **Exclude existing promo products** | checkbox | If ON: filter out rows where `transactions.promo_flag == 1` from baseline scope. |
| **Maintain margin >** | number + `%` suffix | Sets `constraints.min_margin_rate = value/100`. |
| **Scenario Notes** *(Optional)* | textarea | Appears in saved scenario detail and on output screen. |

### 2.8 Action bar (bottom of left column)
- **Reset** (outline) — resets all inputs to defaults.
- **Save Scenario** (outline) — persists the scenario JSON to `localStorage` under `spanx:promotionSimulationPreset` plus the saved-scenarios list.
- **Run Simulation** (primary, purple) — POSTs the request to backend and renders the output screen (§4).

---

## 3. Offer Details — per promotion type

> **Common rule:** every Offer Details card has three columns:
> - **Left:** the type-specific input(s) (slider / threshold / discount depth).
> - **Middle:** an `Applies To` radio group.
> - **Right:** an `Additional Settings` panel with type-specific toggles.

### 3.1 % Discount (Image 109)

**Card title:** `Offer Details — % Discount`

#### 3.1.1 Inputs
| Field | Control | Bounds | Storage |
| --- | --- | --- | --- |
| `Discount Depth` | slider + `%` input box | 0 – 50 % (step 1) | `scenario.discount_depth` |
| `Applies To` | radio group: `Selected Products` / `All Products in Selected Categories` | radio (one selected) | `scenario.applies_to` |
| `Stack with ongoing offers` | checkbox | bool | `scenario.stack_offers` |

#### 3.1.2 Datasets used
| Dataset | Columns consumed |
| --- | --- |
| `transactions.xlsx` | `order_id, order_date, product_id, customer_id, channel, gross_price, net_price, cost_price, discount_pct, promo_id, promo_flag, is_returned` |
| `product_master.xlsx` | `product_id, product_name, category, base_price, cost_price` |
| `customer_master.xlsx` | `customer_id, segment_name` |
| `elasticity.xlsx` | `product_id, category, channel, segment, elasticity` |
| `returns.xlsx` | `order_id, product_id` |
| `inventory.xlsx` | `Product, Category, current_stock, weeks_of_cover, week_start` |
| ML artifact | `model_artifacts/promo_regression_daily_artifacts.pkl` |

#### 3.1.3 Calculation pipeline (step-by-step)

Let the inputs be `discount = discount_depth / 100`, scenario_date `D`, and scope (`product_id, channel, segment`).

**Step 1 — Build the baseline scope (last 4 weeks).**
```
scope = transactions[
    order_date BETWEEN max(date) - 4 weeks AND max(date)
    AND product_id ∈ selected_products
    AND channel    ∈ selected_channels
    AND segment    ∈ selected_segments
]
```

**Step 2 — Segment weights (when `All Segments` selected).**
```
weights[s] = count(scope where segment = s) / count(scope)
```

**Step 3 — Baseline units & revenue (per segment, weighted).**
For each `(segment, weight)`:
1. `base_row = pick_base_row(artifacts.scored_frame, product_id, category, channel, segment, D)`
2. `baseline_features = base_feature_row(base_row, gross_price = base_price, discount_pct = 0, promo_flag = 0)`
3. `baseline_revenue_seg = trained_model.predict(baseline_features)`
4. `baseline_units_seg   = baseline_revenue_seg / base_price`

```
baseline_units = Σ baseline_units_seg × weights[s]
baseline_revenue = baseline_units × base_price
baseline_margin  = (base_price - cost_price) × baseline_units
```

**Step 4 — Predicted units under promo.**
```
promo_features            = baseline_features
promo_features.gross_price = base_price × (1 - discount)
promo_features.discount_pct = discount
promo_features.promo_flag   = 1
promo_revenue_pred  = trained_model.predict(promo_features)
predicted_units_seg = promo_revenue_pred / max(base_price × (1 - discount), 1e-9)
```

**Step 5 — Apply elasticity boost.**
```
elasticity_val   = mean(elasticity[product_id, category, channel, segment]) → fallback -1.0
elasticity_boost = 1 + |elasticity_val| × discount × 0.3
new_units_seg    = predicted_units_seg × elasticity_boost
```

**Step 6 — Aggregate scenario metrics.**
```
new_units    = Σ new_units_seg × weights[s]
new_price    = base_price × (1 - discount)
new_revenue  = new_units × new_price
new_margin   = (new_price - cost_price) × new_units

incremental_units   = new_units   - baseline_units
incremental_revenue = new_revenue - baseline_revenue
incremental_margin  = new_margin  - baseline_margin
promo_cost          = baseline_units × base_price × discount
ROI                 = incremental_margin / promo_cost
margin_rate         = new_margin / new_revenue
```

**Step 7 — Cannibalization split.**
```
promo_dependency = unique_orders(scope where promo_flag=1) / unique_orders(scope)
e_factor         = min(|weighted_elasticity|, 2)
cannib_rate      = clip(0.2 + 0.3·promo_dependency + 0.2·(1 - e_factor/2), 0.1, 0.6)
true_incremental = incremental_units × (1 - cannib_rate)
cannibalized     = incremental_units × cannib_rate
pull_forward     = incremental_units × 0.10
```

**Step 8 — Returns.**
```
return_sensitivity = 0.2 if category == "Bras" else 0.1
baseline_return_rate = unique_orders(returns ⨝ scope.order_id) / unique_orders(scope)
new_return_rate      = min(baseline_return_rate + discount × return_sensitivity, 0.60)
```

**Step 9 — Inventory / Weeks of Cover.**
```
weekly_sales        = baseline_units / 4
weeks_of_cover_pre  = inventory.weeks_of_cover OR (current_stock / weekly_sales)
weeks_of_cover_post = max(current_stock - new_units, 0) / weekly_sales
```

**Step 10 — Decision.**
```
SCALE     if ROI > 1
OPTIMIZE  if 0.3 ≤ ROI ≤ 1
STOP      if ROI < 0.3
```

#### 3.1.4 Right-panel content (this type)
- **AI Guidance:** *"20% off discounts on Leggings for Deal Hunters could improve revenue uplift while protecting margin."*
- **Key Impact Drivers:**
  - High price elasticity in selected segment
  - Strong cost demand for leggings
  - Competitor promotions ending soon
- **What's Next?:** Run the simulation to see expected impact on revenue, margin, ROI and returns.

---

### 3.2 Bundle — Mix & Match (Image 140)

**Card title:** `Offer Details — Bundle — Mix & Match`

#### 3.2.1 Inputs
| Field | Control | Notes |
| --- | --- | --- |
| `Bundle Type` | dropdown | `Mix & Match` (default), `Buy X Get Y` |
| `Applies To` | radio | `Selected Products` / `All Products in Selected Categories` / `All Products` |
| `Minimum Cart Value` *(Optional)* | number ($) | Sets `scenario.min_cart_value` |
| `Cheapest item is free` | checkbox | If ON, applies BOGO style: free unit = `min(line_items.net_price)` |

#### 3.2.2 Datasets used
Same as §3.1.2 plus:
- **Bundle simulation requires the basket structure**, so we group `transactions` by `order_id` to compute `basket_size`, `basket_value`, and `cheapest_unit_price`.

#### 3.2.3 Calculation pipeline

**Step 1 — Baseline basket KPIs (last 4 weeks).**
```
baskets = transactions GROUP BY order_id
basket_size_avg   = mean(count(items per order))
basket_value_avg  = mean(sum(net_price per order))
cheapest_avg      = mean(min(net_price per order))
```

**Step 2 — Bundle uplift assumption.**
For Bundle (Mix & Match) we estimate AOV lift instead of unit lift:
```
aov_lift_factor = 1 + 0.10                # default 10% AOV uplift
new_basket_size_avg  = basket_size_avg × (1 + 0.15)
new_basket_value_avg = basket_value_avg × aov_lift_factor

If cheapest_item_is_free:
    discount_value_per_basket = cheapest_avg
    new_basket_value_avg     -= discount_value_per_basket
```

**Step 3 — Translate to revenue/margin (per segment, weighted).**
```
baseline_units   = baseline_units_per_basket × baseline_orders
new_units        = new_basket_size_avg × baseline_orders
baseline_revenue = baseline_orders × basket_value_avg
new_revenue      = baseline_orders × new_basket_value_avg

baseline_margin  = (base_price - cost_price) × baseline_units
discount_per_unit = (basket_value_avg - new_basket_value_avg) / new_basket_size_avg
new_unit_price   = base_price - discount_per_unit
new_margin       = (new_unit_price - cost_price) × new_units
```

**Step 4 — Cannibalization, returns, inventory.**
Same formulas as §3.1 (steps 7–9) but `discount` is replaced by `effective_discount = discount_per_unit / base_price`.

**Step 5 — Min cart value gate.**
If `min_cart_value` is set, only baskets where `basket_value ≥ min_cart_value` qualify; recalculate `qualifying_share = qualifying_orders / total_orders` and multiply the incremental metrics by `qualifying_share`.

#### 3.2.4 Right-panel content
- **AI Guidance:** *"Bundle promo with MoM and incremental revenue for Deal Hunters in Q2 — 20% discounts perform best while protecting margin."*
- **Key Impact Drivers:**
  - Higher AOV and basket size
  - Increased customer engagement
  - Lower revenue dependency on double-digit % off
- **What's Next?:** Run the simulation to see expected impact on revenue, margin, ROI and returns.

---

### 3.3 First Time Offer — New Customers (Image 167)

**Card title:** `Offer Details — First Time Offer`

#### 3.3.1 Inputs
| Field | Control | Notes |
| --- | --- | --- |
| `Discount Depth for New Customers` | slider + % box | 0 – 30 % |
| `Applies To` | radio | `Selected Products` / `Products in Selected Categories` |
| `Minimum Order Value` *(Optional)* | number ($) | Constraint on order eligibility |
| `New customers only` | checkbox (on by default) | Filters scope to first-purchase customers |
| `Use auto-applied` | checkbox | UX flag for redemption mechanics; no calc impact |

#### 3.3.2 Datasets used
Same as §3.1 plus customer-level join:
- `customer_master.xlsx` to flag `is_new_customer` (no orders before `D - 12 months`).
- `transactions.xlsx` filtered to `customer_id WHERE first_order_date IN scenario_window`.

#### 3.3.3 Calculation pipeline

**Step 1 — Define the new-customer scope.**
```
first_order_date_per_cust = transactions GROUP BY customer_id → MIN(order_date)
new_customers = customers WHERE first_order_date ≥ D - duration
scope_new = transactions[customer_id ∈ new_customers]
```

**Step 2 — Baseline acquisition KPIs.**
```
baseline_new_customers = count(new_customers in last 4 weeks)
baseline_aov_new       = mean(net_price per first order)
baseline_cac           = (assumed) marketing_spend / baseline_new_customers
```

**Step 3 — Predicted new customers under promo.**
Use a conversion-uplift heuristic (the segment's first-purchase elasticity, taken from `elasticity` for `segment = New Customers`):
```
conv_lift = 1 + |elasticity[New Customers]| × discount × 0.5     # higher coefficient than %-Off
new_new_customers     = baseline_new_customers × conv_lift
new_aov_new           = baseline_aov_new × (1 - discount)
new_revenue_new       = new_new_customers × new_aov_new
```

**Step 4 — LTV & ROI.**
```
ltv_uplift_factor = 1.15                                          # +15% expected LTV from acquired-via-promo cohort
ltv_per_customer  = baseline_aov_new × repeat_rate × avg_orders_per_year × ltv_uplift_factor
incremental_ltv   = (new_new_customers - baseline_new_customers) × ltv_per_customer
promo_cost        = new_new_customers × baseline_aov_new × discount
ROI               = (incremental_margin + incremental_ltv × discount_factor) / promo_cost
```

**Step 5 — Min order value gate.** If set, scale `new_new_customers` by the historical share of orders ≥ `min_order_value`.

#### 3.3.4 Right-panel content
- **AI Guidance:** *"First time Offer with new customer acquisition and long term LTV — 15% discounts perform best while protecting margin."*
- **Key Impact Drivers:**
  - Higher conversion rates
  - Increased customer lifetime value
- **What's Next?:** Run the simulation to see expected impact on revenue, margin, ROI and returns.

---

### 3.4 Clearance Markdown (Image 195)

**Card title:** `Offer Details — Clearance Markdown`

#### 3.4.1 Inputs
| Field | Control | Notes |
| --- | --- | --- |
| `Discount Depth` | slider + % box | 0 – 80 % (deeper than %-Off) |
| `Applies To` | radio | `Selected Products` / `Products in Selected Categories` / `All Products` |
| `Inventory Trigger` *(Optional)* | dropdown | `Weeks of Cover Greater Than` |
| `Weeks of Cover Greater Than` | number (weeks) | Only items with `weeks_of_cover > X` are eligible |

#### 3.4.2 Datasets used
Same as §3.1 — but **`inventory.xlsx` is the gating dataset** here (not optional).

#### 3.4.3 Calculation pipeline

**Step 1 — Build clearance candidate list.**
```
candidates = inventory[
    Product ∈ selected_products
    AND weeks_of_cover > inventory_trigger_weeks
    AND week_start = MAX(week_start)
]
```

**Step 2 — Per-product simulation (loop over candidates).** Same %-Off pipeline (§3.1 steps 3-6) BUT with category-specific demand-curve adjustment:
```
clearance_lift_factor = 1 + min(|elasticity_val|, 3.0) × discount × 0.6     # steeper than 0.3
new_units_seg        = predicted_units_seg × clearance_lift_factor
```

**Step 3 — Sell-through and inventory drawdown.**
```
sell_through_pct  = new_units / current_stock
remaining_inv     = max(current_stock - new_units, 0)
weeks_of_cover_post = remaining_inv / weekly_sales
inventory_reduction_pct = (current_stock - remaining_inv) / current_stock × 100
```

**Step 4 — Margin floor & decision.**
```
margin_rate = new_margin / new_revenue
if margin_rate < 0:
    decision = "STOP"
else:
    decision = SCALE if sell_through_pct >= 0.50 and ROI >= 0.3 else OPTIMIZE
```

**Step 5 — Returns.** Returns stay at base sensitivity (0.1) since clearance is final-sale by default; optional checkbox `Final Sale` can zero-out the discount return penalty.

#### 3.4.4 Right-panel content
- **AI Guidance:** *"Clearance markdown helps liquidate excess inventory, lift margin from the slow-moving of even and lower demand styles."*
- **Key Impact Drivers:**
  - Improved cash flow
  - Higher margin impact
  - Risk of training customers to wait for discounts
- **What's Next?:** Run the simulation to see expected impact on revenue, margin, ROI and returns.

---

### 3.5 Free Shipping (Image 223)

**Card title:** `Offer Details — Free Shipping`

#### 3.5.1 Inputs
| Field | Control | Notes |
| --- | --- | --- |
| `Threshold — Minimum Order Value` | number ($) | Only baskets ≥ this value qualify |
| `Applies To` | radio | `Selected Products` / `Products in Selected Categories` |
| `Applicable Channels` | multi-select | DTC / Amazon / Nordstrom / Bloomingdales |
| `Stack with other offers` | checkbox | If ON, allow co-occurrence with %-Off scenarios |

#### 3.5.2 Datasets used
Same as §3.1 plus:
- An assumed `avg_shipping_cost_per_order` constant (e.g. `$8.95` for DTC, `0` for marketplaces).

#### 3.5.3 Calculation pipeline

**Step 1 — Baseline metrics.**
```
baseline_aov   = mean(basket_value)
baseline_orders = count(orders in scope)
qualifying_share = orders[basket_value ≥ threshold] / baseline_orders
```

**Step 2 — Conversion lift (industry calibration).**
```
conv_lift_factor   = 1 + 0.10 + (0.15 × indicator(threshold > 0))   # +10% baseline, +25% if threshold gates
new_orders         = baseline_orders × (1 + qualifying_share × (conv_lift_factor - 1))
aov_lift_factor    = 1 + 0.05                                       # +5% AOV from threshold push
new_aov            = baseline_aov × aov_lift_factor
new_revenue        = new_orders × new_aov
```

**Step 3 — Margin impact (shipping is the only cost).**
```
shipping_cost_total      = qualifying_orders × avg_shipping_cost_per_order
incremental_revenue      = new_revenue - baseline_revenue
incremental_margin       = (new_revenue × baseline_margin_rate) - shipping_cost_total
ROI                      = incremental_margin / shipping_cost_total
```

**Step 4 — Returns & inventory.**
```
return_rate_change   = +0.5 % (threshold orders carry slightly higher return risk)
weeks_of_cover_post  = same formula as §3.1 step 9 with new_units = new_orders × basket_size_avg
```

**Step 5 — Decision.** Same SCALE / OPTIMIZE / STOP rule as §3.1 step 10.

#### 3.5.4 Right-panel content
- **AI Guidance:** *"Free Shipping boosts conversion rates by 10–25%. Ideal for Deal Hunters and Value Seekers."*
- **Key Impact Drivers:**
  - Higher conversion rate
  - Increased average order value
  - Lower bracket dependency
- **What's Next?:** Run the simulation to see expected impact on revenue, margin, ROI and returns.

---

## 4. Scenario Output (Image 254)

### 4.1 Header strip
- Page title `Promotion Simulation` (unchanged) + stepper (Promotion Studio still highlighted — the user is still in step 6).
- Top-right: `Saved Scenarios` button.

### 4.2 Output card — top metrics row
A horizontal row with **4 KPI tiles** + 2 action buttons.

| Tile | Big value | Delta vs Baseline | Source field |
| --- | --- | --- | --- |
| **Incremental Revenue** | `$2.4M` | `+11.7%` (green) | `incremental_revenue`, `new_revenue_change_pct` |
| **Incremental Margin** | `$1.12M` | `+5.4%` (green) | `incremental_margin`, `new_margin_change_pct` |
| **Incremental Units** | `4.6k` | `+2.6%` (green) | `incremental_units`, `new_units_change_pct` |
| **Incremental Returns** | `128.7K` | `-1.3%` (red = good for returns) | `return_rate_change × baseline_orders` |

**Right-side buttons:** `Edit Scenario` (outline), `Run New Scenario` (primary).

### 4.3 Impact Overview chart (left, ≈ 60% width)
A grouped bar chart with **Baseline (light gray) vs Scenario (purple)** for these 6 categories:
1. Revenue
2. Gross Margin
3. Units
4. AOV
5. Conversion Rate
6. Repeat Rate

Each pair shows a `%` delta label between bars. Source: same fields as §3.1 steps 6, plus AOV = `revenue / orders` and conversion / repeat from `transactions` aggregations.

### 4.4 Scenario Scorecard (right of Impact Overview)
A square card showing:
- **Center gauge:** integer score `89` with label `Excellent`.
- **Composite formula:**
```
score = 100 × (
    0.40 × clip(incremental_revenue / target_revenue, 0, 1.25) +
    0.20 × clip(incremental_margin  / target_margin , 0, 1.25) +
    0.15 × clip(ROI / 1.0           , 0, 1.25) +
    0.10 × clip(true_incremental_pct, 0, 1.0) +
    0.10 × (1 - clip(return_rate / max_return_rate, 0, 1)) +
    0.05 × (1 if margin_rate ≥ min_margin_rate else 0)
)
```
- **Sub-rows (right side of the card):**
  - `Revenue Impact`
  - `Margin Impact`
  - `Customer Reach`
  - `Retention Risk`
  - `Inventory Feasibility`

Each sub-row shows a small bar (0–100) with the same proportional weighting as above.

### 4.5 Customer Segment Impact (Revenue Lift %)
A horizontal bar chart of revenue lift by `segment`:
```
lift_seg = (new_revenue_seg - baseline_revenue_seg) / baseline_revenue_seg
```

### 4.6 Category Impact (Revenue Lift %)
A donut/bar chart with **headline `$2.4M`** in the middle. Slices = top categories from `product_master.category` aggregation.

### 4.7 Channel Impact (Revenue Lift %)
A pie chart broken down by `channel`. Each slice has its `% of incremental revenue` and absolute $ tooltip.

### 4.8 Assumptions & Constraints panel (bottom-left)
- Bullet list summarising the active scenario:
  - `20% off discount applied for 2 weeks`
  - `Top return rate increases by 5%`
  - `Exclude existing promos`
- Sourced from the form state at submission time.

### 4.9 Scenario Notes panel (bottom-center)
Free text from the Additional Conditions card.

### 4.10 Right side info column
| Card | Content |
| --- | --- |
| **Hi Key Takeaways** | Generated 2-line tldr from the recommendation engine (e.g. *"Increase revenue by 18.2% with a margin trade-off below 1.5%. Q2 Leggings Promotion appears most effective and may be high-impact feasible for execution."*). Includes a `View All Recommendations` button. |
| **Scenario Comparison** | Two row mini-cards: `Baseline` vs `Best Scenario`. Below, a `Compare More Scenarios` button. |
| **Read More / Recommendation** | One-line LLM blurb (e.g. *"Looks good Deal Hunters because this is for limited offer."*). |
| **Run Variations** | Primary purple button — re-runs the optimizer with discount sweep `[10, 15, 20, 25, 30]` and returns the top 5 scenarios. |

### 4.11 Footer action bar
- `Download Report` (outline, icon `bi-download`) — exports a PDF/Excel.
- `Save Scenario` (outline) — same as §2.8.
- `View Scenario` (primary) — re-opens the editable Scenario Builder pre-filled with this scenario's inputs.

---

## 5. Optimizer & Recommendation Engine (used by `Run Variations`)

This wires together the math from §3 with the constraint-aware engine from `promotion_simulation_upgrade.md`.

### 5.1 Scenario generator
```
scenarios = product([selected_products, selected_channels, selected_segments,
                     selected_promos, [10, 15, 20, 25, 30]])
```

### 5.2 Per-scenario simulation
Calls the §3.x pipeline matching the promotion type; returns the same fields used by the output screen (`incremental_revenue, incremental_margin, ROI, return_rate, margin_rate, weeks_of_cover_post, true_incremental_pct, decision`).

### 5.3 Constraint engine
```
valid = scenarios[
    margin_rate ≥ constraints.min_margin_rate
    AND return_rate ≤ constraints.max_return_rate
]
rejected = scenarios \ valid          # tagged with violation_reason
```

### 5.4 Scoring (objective-aware)
```
quality_score    = true_incremental / (incremental_revenue + 1e-6)
return_penalty   = return_rate
inventory_penalty = weeks_of_cover_post

if objective == "profit":
    score = 0.35·incremental_margin + 0.25·ROI + 0.15·quality_score
          + 0.15·incremental_revenue - 0.05·return_penalty - 0.05·inventory_penalty
else:                                   # objective == "revenue"
    score = 0.40·incremental_revenue + 0.20·incremental_margin + 0.15·ROI
          + 0.15·quality_score - 0.05·return_penalty - 0.05·inventory_penalty
```

### 5.5 Output payload (for the right-side cards)
```json
{
  "best": {…top scenario…},
  "top_5": [scenario, …],
  "rejected_count": N,
  "explanation": {
    "why_selected": ["Meets margin constraint (0.46)", "Return rate 0.18 within limit",
                     "High incremental revenue ($2.4M)", "ROI = 1.42"],
    "why_rejected": ["12 scenarios rejected due to constraint violations"]
  },
  "recommendation": {
    "action": "Run %_OFF on Q2 Leggings at 20%",
    "channel": "DTC",
    "segment": "Deal Hunters",
    "impact": {"revenue": 2400000, "margin": 1120000, "ROI": 1.42, "return_rate": 0.18},
    "confidence": "HIGH"
  }
}
```

---

## 6. Datasets — quick reference

| File | Used by | Required columns |
| --- | --- | --- |
| `data/transactions.xlsx` | All 5 types (baseline, ROI, returns, channel, segment) | `order_id, order_date, product_id, customer_id, channel, gross_price, net_price, cost_price, discount_pct, promo_id, is_returned` |
| `data/product_master.xlsx` | All types | `product_id, product_name, category, base_price, cost_price` |
| `data/customer_master.xlsx` | All types (segments) and First Time Offer (new vs existing) | `customer_id, segment_name` |
| `data/elasticity.xlsx` | All types (uplift) | `product_id, category, channel, segment, elasticity` |
| `data/returns.xlsx` | All types (return rate) | `order_id, product_id` |
| `data/inventory.xlsx` | All types; **gating** for Clearance | `Product, Category, current_stock, weeks_of_cover, week_start` |
| `data/promotion_master.xlsx` | (Optional) historical promo performance | `promo_id, promo_type, discount_pct, start_date, end_date, channel, target_segment` |
| `data/competitor_pricing.xlsx` | AI Guidance "competitor promotions ending soon" insight | `product_id, week, channel, our_price, competitor_price, price_gap_pct` |
| `data/demand_signals.xlsx` | AI Guidance "strong demand for leggings" insight | `date, category, social_buzz_index, search_trend, demand_index` |
| `data/channel_migration.xlsx` | Channel Impact card (output screen) | `from_channel, to_channel, segment, migration_pct, product_category` |
| `model_artifacts/promo_regression_daily_artifacts.pkl` | Predicted revenue under any promo | trained `PromoRegressionArtifacts` |

---

## 7. State, persistence, and APIs

### 7.1 Frontend state
Stored under `state` in `step10-promotion-simulation.js`:
```
{
  promotionType: '%_OFF' | 'BUNDLE' | 'FIRST_TIME' | 'CLEARANCE' | 'FREE_SHIPPING',
  scenarioName, objective, scenarioCompare,
  products: [], categories: [], channels: [], segments: [],
  offerDetails: { … type-specific … },
  timing: { startDate, endDate, durationWeeks, dayTargeting: 'MTWTFSS' bitmap },
  conditions: { excludeExistingPromo, minMarginPct, scenarioNotes },
  output: { … last simulation payload … },
  savedScenarios: [ … ]
}
```

### 7.2 Backend route
`GET /api/screens/step10-promotion-simulation/bootstrap`
- Query params accept any subset of the state above (URL-encoded).
- Response includes `simulation_backend.{scenario_builder, baseline, scenario_output, incrementality_vs_cannibalization, inventory_returns_impact, ai_recommendation}` matching the output card schema.

### 7.3 Saved scenarios
- Stored in `localStorage` under `spanx:promotionSimulationPreset` (single most-recent) and `spanx:savedPromotionScenarios` (array).
- The `Saved Scenarios` button opens a side-panel listing names + dates + a `Load` action.

---

## 8. Open assumptions to confirm

These are calibration constants used above; we should confirm before we wire them in:
- Bundle AOV uplift (`+10%`) and basket-size uplift (`+15%`).
- First Time Offer LTV uplift factor (`+15%`).
- Free Shipping conversion lift (`+10%` baseline, `+25%` when threshold gates) and `aov_lift = +5%`.
- Clearance demand coefficient (`0.6` vs `0.3` for %-Off).
- DTC shipping cost (`$8.95`).
- Score weights in §5.4 (0.40/0.20/0.15/0.15/0.05/0.05).

---

**Approval gate:** once you confirm this spec is correct, the implementation will (1) refactor `step10-promotion-simulation.js` to render the new 5-tab UI, (2) extend `step07_promotion_simulation.py` with type-specific simulators (Bundle, First Time, Clearance, Free Shipping), and (3) add the optimizer + scorecard endpoint described in §4.4 and §5.
