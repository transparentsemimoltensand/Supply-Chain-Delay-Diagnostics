# Supply Chain Delay Diagnostics — Data Preparation

Python/Pandas pipeline that cleans a 100K-row synthetic supply chain dataset and produces a Power BI–ready star schema (2 fact tables + 5 dimensions).

## Dataset

- **Source:** [Supply Chain Operation & Delivery Performance 100K](https://www.kaggle.com/datasets/mobeenfatimah/supply-chain-operation-and-delivery-performance-100k)
- **Size:** 100,000 shipment records × 106 columns (2022–2026)
- **Nature:** Synthetic data simulating global logistics operations. Findings are dataset-scoped, not real-world causal claims.

## Objective

Prepare a clean, query-ready model to diagnose **which carriers, suppliers, routes, and shipment characteristics drive delivery delays**, and quantify the **cost impact**.

## Pipeline

### 1. Load & type cleaning
- Parse `shipment_date` to datetime.
- Convert Yes/No flags to 1/0: `delivery_delayed`, `delivery_successful`, `customs_required`.

### 2. Derived features
- `vehicle_age_bucket`: equal-width bands (1–3, 4–6, 7–9, 10–11 yrs) chosen after confirming `vehicle_age` is uniformly distributed 1–11 with no natural breakpoints.

### 3. Dimension stability audit
Verified that dimension attributes are functionally dependent on their entity key before assigning them to dimensions:

| Entity | Attribute | Unique values per key |
|---|---|---|
| Carrier | `carrier_rating`, `carrier_reliability_score` | 1 |
| Route | `route_type`, `route_quality`, `road_condition` | 1 |
| Supplier | `supplier_country`, `supplier_rating`, `supplier_lead_time_days` | 1 |
| Product | `product_category`, `product_subcategory`, `order_priority` | 1 |

All passed → dimensions are correctly normalized.

### 4. Leaky column audit
Correlation of numeric columns against `delay_hours` identified outcome-derived ("leaky") fields:

| Column | r with `delay_hours` | Action |
|---|---|---|
| `delay_days` | 0.998 | Descriptive only (duplicate of outcome) |
| `delivery_delayed` | 0.882 | Outcome flag — not a driver |
| `delivery_attempts` | 0.822 | Descriptive only |
| `delivery_performance_score` | −0.844 | Excluded from model (leaky) |
| `supply_chain_risk_score` | 0.653 | Excluded (semi-leaky composite) |

Legitimate drivers retained: `traffic_index` (r=0.397), `supplier_historical_delay_rate` (r=0.214), `historical_delay_rate` (r=0.203), `warehouse_utilization` (r=0.160).

### 5. Cost decomposition validation
Verified the six component costs (`fuel`, `transportation`, `warehouse`, `handling`, `customs`, `insurance`) sum exactly to `total_logistics_cost` across all 100,000 rows (max floating-point deviation ~1.5e-11). Full decomposition confirmed.

## Output Schema

| Table | Grain | Rows | Cols |
|---|---|---|---|
| `Dim_Date` | 1 row per calendar day | 1,641 | 8 |
| `Dim_Carrier` | 1 row per carrier | 120 | 4 |
| `Dim_Supplier` | 1 row per supplier | 550 | 5 |
| `Dim_Route` | 1 row per route | 60 | 5 |
| `Dim_Product` | 1 row per category × subcategory × priority | 160 | 4 |
| `Fact_Shipment` | 1 row per shipment | 100,000 | 22 |
| `Fact_Cost` | 1 row per shipment × cost type | 600,000 | 8 |

**Relationships:** All dimensions → both facts (one-to-many, single direction). 10 relationships total.

`Fact_Cost` uses a composite grain (`shipment_id` + `cost_type`); `shipment_id` is **not** unique there. Count shipments from `Fact_Shipment` using `COUNT`, or from `Fact_Cost` using `DISTINCTCOUNT`.

## Design Decisions

- **One fact per grain, not one wide table.** Shipment-level measures (delay, cost, quantity) live in `Fact_Shipment`. Cost breakdown lives in `Fact_Cost` in long format so `cost_type` is a filterable dimension rather than six hardcoded columns.
- **Static vs. varying attributes.** Attributes verified as stable per entity (e.g., `carrier_rating`) live in dimensions; attributes that vary per shipment (e.g., `weather_condition`, `transport_mode`, `vehicle_age_bucket`) live in `Fact_Shipment`.
- **Static ratings are cross-sectional, not time-series.** The dataset provides one static profile per carrier/supplier/route (no SCD history), so trend analysis is performed on fact measures only.
- **Leaky columns excluded from driver analysis** but retained for descriptive visuals where appropriate.

## Files
pbi_supply_chain/
├── Dim_Date.csv
├── Dim_Carrier.csv
├── Dim_Supplier.csv
├── Dim_Route.csv
├── Dim_Product.csv
├── Fact_Shipment.csv
└── Fact_Cost.csv


## How to Reproduce

1. Download the dataset from Kaggle.
2. Run `build_schema.py` (adjust the input `path` at the top).
3. Output CSVs are written to `pbi_supply_chain/` and zipped as `pbi_supply_chain_tables.zip`.
4. Import all 7 CSVs into Power BI Desktop → build relationships in Model view.

## Next Phase

- DAX measures: `Delay Rate`, `Avg Delay Hours (delayed only)`, `Total Logistics Cost`, `Cost % of Total`, `YoY Delay Rate Change`.
- Dashboard pages: Delay Overview → Carrier/Supplier Ranking → Route & Environment → Cost Breakdown → Weather × Transport Mode Sensitivity.
- Optional: LLM-powered Q&A integration.
