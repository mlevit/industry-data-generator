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

## License

This project is provided as-is for demonstration and testing purposes. All generated data is synthetic and contains no real personal information.
