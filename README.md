# Industry Data Generator

Synthetically generated multi-industry dataset catalog for [Databricks Unity Catalog](https://docs.databricks.com/en/data-governance/unity-catalog/index.html) with **realistic statistical distributions** (log-normal, beta, gamma) and **Faker-generated PII**. Each industry schema contains one entity table and one high-volume event table (100K–500K rows). All date columns use native Date/Timestamp types.

Designed for analytics, BI, and ML use cases where realistic-looking data is needed without exposing real customer information.

## Industries

| Schema            | Entity Table    | Rows | Event Table         | Rows      | Key Features                                                                             |
| ----------------- | --------------- | ---- | ------------------- | --------- | ---------------------------------------------------------------------------------------- |
| **finance**       | `accounts`      | ~2K  | `transactions`      | 100K–500K | 7 account types, 7 currencies, 40+ merchants, pattern-based fraud flags                  |
| **gaming**        | `players`       | ~5K  | `wager_records`     | 100K–500K | Activity-earned VIP tiers, session tracking, behavior-driven responsible gambling flags  |
| **health**        | `patients`      | ~5K  | `medical_records`   | 100K–500K | Comorbidity modeling (up to 3 conditions), ~35 ICD-10 codes, 50 physicians               |
| **insurance**     | `policyholders` | ~22K | `claims`            | 100K–500K | Multi-policy (15K people), all 50 US states + DC, pattern-based fraud, subrogation       |
| **manufacturing** | `equipment`     | ~3K  | `production_orders` | 100K–500K | Maintenance tracking, energy consumption, scrap cost, yield variance (70–105%)           |
| **retail**        | `customers`     | ~5K  | `orders`            | 100K–500K | 60+ products across 7 categories, demographics, product-level return rates               |
| **telecom**       | `subscribers`   | ~5K  | `call_records`      | 100K–500K | Temporal churn dates, network quality metrics (latency/signal), region-correlated towers |
| **utilities**     | `meters`        | ~5K  | `usage_records`     | 100K–500K | Temperature-driven usage, grid zones, spatially correlated outages                       |

## Prerequisites

- [Databricks](https://www.databricks.com/) workspace with Unity Catalog enabled
- A cluster with permissions to create and drop catalogs
- Python package: `faker` (installed automatically via `%pip install faker` in each notebook)

## Quick Start

1. Import all notebooks into your Databricks workspace.
2. Open `[MASTER] Sample Dataset Generator.ipynb`.
3. Set the `catalog` widget parameter (default: `industry_sample_data`).
4. Click **Run All**.

The master notebook drops and recreates the target catalog, then runs each industry notebook sequentially. A full run takes approximately 15–20 minutes depending on cluster size.

## Project Structure

```
├── [MASTER] Sample Dataset Generator.ipynb    # Orchestrator — runs all industry notebooks
├── Finance Dataset Generator.ipynb            # Banking accounts & transactions
├── Gaming Dataset Generator.ipynb             # Player profiles & wager records
├── Health Dataset Generator.ipynb             # Patients & medical records
├── Insurance Dataset Generator.ipynb          # Policyholders & claims
├── Manufacturing Dataset Generator.ipynb      # Equipment & production orders
├── Retail Dataset Generator.ipynb             # Customers & orders
├── Telecom Dataset Generator.ipynb            # Subscribers & call records
└── Utilities Dataset Generator.ipynb          # Meters & usage records
```

## How It Works

Each industry notebook follows the same pattern:

1. **`%pip install faker`** and restart the Python interpreter.
2. **Read the `catalog` widget** passed from the master notebook.
3. **Create the schema** (e.g., `CREATE SCHEMA IF NOT EXISTS <catalog>.finance`).
4. **Generate entity data** — builds a list of `Row` objects with Faker-generated names, emails, addresses and statistically distributed numeric fields, then writes to a managed Delta table.
5. **Generate event data** — produces 100K–500K rows of transactional records linked to entity IDs, with realistic distributions and domain-specific logic (fraud patterns, seasonal curves, temperature correlations, etc.).
6. **Apply metadata** — adds column-level `COMMENT`s, a schema-level description, and a `RemoveAfter` governance tag.

### Data Quality Highlights

- **Referential integrity** — event records always reference valid entity IDs.
- **Realistic distributions** — amounts, durations, and quantities use log-normal, beta, and gamma distributions rather than uniform random values.
- **Domain-specific logic** — fraud flags follow suspicious patterns, churn dates only appear on churned subscribers, utility usage correlates with temperature, insurance policyholders can hold multiple policies, and gaming deposits/withdrawals reconcile with wager activity.
- **Temporal consistency** — dates are ordered logically (e.g., account opened before first transaction).

## Configuration

| Parameter | Default                | Description                                                   |
| --------- | ---------------------- | ------------------------------------------------------------- |
| `catalog` | `industry_sample_data` | Target Unity Catalog name. Dropped and recreated on each run. |

## Adding a New Industry

1. Create a new notebook following the same pattern as the existing ones (entity table + event table).
2. Accept the `catalog` widget parameter via `dbutils.widgets.get('catalog')`.
3. Add a `dbutils.notebook.run("<Your New Notebook>", ...)` call in the master notebook.

## Output

All tables are written as **managed Delta tables** in Unity Catalog. After a successful run, the catalog structure looks like:

```
<catalog>/
├── finance/
│   ├── accounts
│   └── transactions
├── gaming/
│   ├── players
│   └── wager_records
├── health/
│   ├── patients
│   └── medical_records
├── insurance/
│   ├── policyholders
│   └── claims
├── manufacturing/
│   ├── equipment
│   └── production_orders
├── retail/
│   ├── customers
│   └── orders
├── telecom/
│   ├── subscribers
│   └── call_records
└── utilities/
    ├── meters
    └── usage_records
```

## Realism Improvements

Known limitations and future enhancements to make the synthetic data more representative of real-world patterns.

### Cross-Cutting

- [ ] Add time-series realism — seasonality, day-of-week patterns, business hours, and event clustering instead of uniform random timestamps
- [ ] Use heavy-tail (Pareto) entity sampling so a small fraction of entities generate most activity, matching real-world distributions
- [ ] Introduce slowly changing dimensions — history of plan changes, status transitions, address moves, and tier upgrades over time
- [ ] Add running balance / ledger reconciliation where applicable (finance accounts, gaming deposits)
- [ ] Cap fraud probability at 1.0 to avoid semantically invalid values when multiple risk bonuses stack

### Finance

- [ ] Add `daily_account_balances` table — time-series snapshots for liquidity, overdraft, and trend analysis
- [ ] Add `fraud_cases` table — investigation outcomes, resolution status, and chargeback amounts for supervised ML
- [ ] Add FX rate table for cross-currency analytics (7 currencies exist but no exchange rates)
- [ ] Implement velocity-based fraud detection (sliding-window over recent transactions per account) — currently only amount/channel thresholds

### Gaming

- [ ] Correlate VIP tier with actual spend — currently derived from an independent activity score
- [ ] Align `preferred_game` with actual `game_type` distribution on wagers
- [ ] Exclude voided/cashed-out wagers from `player_wager_totals` reconciliation
- [ ] Order wagers chronologically before computing cumulative loss for responsible gaming flags
- [ ] Add `promotions` / `bonuses` table with campaign IDs, wagering requirements, and terms

### Health

- [ ] Extend visit hours past 19:00 for night ER and urgent care
- [ ] Add CPT/HCPCS procedure codes alongside ICD-10 diagnoses
- [ ] Model deductible, copay, and annual out-of-pocket maximum instead of flat `cost * (1 - coverage%)`
- [ ] Add `prescriptions` table — medication, dose, days supply, refill count (currently a single nullable string per visit)
- [ ] Add `payers` / `plans` dimension — plan ID, network, deductible, effective dates
- [ ] Link follow-up visits to prior encounters for longitudinal episode modeling

### Insurance

- [ ] Cap `claim_amount` at `coverage_amount` or model sublimits
- [ ] Distinguish policy period, report date, and occurrence date for claims on non-active policies
- [ ] Fix `days_to_resolve` to use business days (documented as such, but generated as calendar days)
- [ ] Replace `hash()`-based adjuster IDs with deterministic surrogate keys
- [ ] Differentiate life insurance products (term vs whole, mortality tables, beneficiaries) from P&C
- [ ] Add `coverage_items` table — per-peril limits and deductibles (comp vs collision, dwelling vs personal property)
- [ ] Add `catastrophe_events` table for correlated property losses by region

### Manufacturing

- [ ] Correlate defect rate with equipment age, efficiency, and maintenance state
- [ ] Enforce temporal consistency — order `start_date` must be after equipment `install_date`
- [ ] Anchor `next_maintenance_date` relative to `NOW` so it is always in the future for operational equipment
- [ ] Use equipment-type-specific capacity instead of a single 450 min/day constant
- [ ] Add `quality_inspections` table — sample-based results, inspector, pass/fail, root cause
- [ ] Add `shift_calendar` table — plant, date, shift, crew, planned hours for capacity utilization
- [ ] Add `inventory` / `wip_snapshots` for raw material consumption and throughput modeling

### Retail

- [ ] Fix geography — Boise appears under both Pacific Northwest and Mountain West
- [ ] Model refund amounts on returned orders (currently `total_amount` stays positive)
- [ ] Add `promotions` / `coupon_redemptions` table — campaign ID, promo code, discount type, dates
- [ ] Add `dim_store` table — store ID, location, type, square footage for omnichannel analytics
- [ ] Add customer–product affinity so repeat purchases reflect preferences

### Telecom

- [ ] Set `call_result` to NULL or a type-appropriate value for non-voice records (currently always "Completed")
- [ ] Set `data_usage_mb` to 0 or NULL for SMS/MMS instead of small exponential noise
- [ ] Make churn probability depend on experienced latency, dropped calls, and overage charges — not just plan tier and tenure
- [ ] Add `plans` dimension with effective dates, features, and pricing for plan migration analysis
- [ ] Add `network_incidents` table — outage events, duration, affected towers, root cause
- [ ] Add `support_tickets` table — issue type, resolution time, CSAT score

### Utilities

- [ ] Implement time-of-use pricing logic — vary rate by hour-of-day for TOU rate plans
- [ ] Make outages reduce usage — currently `had_outage` is independent of `usage_amount`
- [ ] Model net metering credits for solar export instead of `cost = abs(usage) * rate`
- [ ] Define explicit interval length (daily vs monthly) for usage records
- [ ] Correlate `peak_demand_kw` with interval usage instead of independent log-normal draws
- [ ] Add `outage_events` table — start/end timestamps, cause, affected zone, SAIDI/SAIFI metrics
- [ ] Add `billing_periods` / `invoices` table — monthly bills, payment status, arrears
- [ ] Add `weather_observations` table — station-level hourly data to replace per-reading synthetic temperature

## License

This project is provided as-is for demonstration and testing purposes. All generated data is synthetic and contains no real personal information.
