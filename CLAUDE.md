# CLAUDE.md

## Project Overview

Industry Data Generator produces synthetic, multi-industry datasets loaded into **Databricks Unity Catalog** as managed Delta tables. It generates realistic-looking data (log-normal/beta/gamma distributions, Faker PII) for analytics, BI, and ML without real customer information.

## Architecture

This is a **Databricks notebook project**, not a conventional Python package. There is no `src/`, `tests/`, `setup.py`, or `requirements.txt`.

- **Master notebook** (`[MASTER] Sample Dataset Generator.ipynb`) orchestrates everything — drops/recreates the catalog, then calls each industry notebook via `dbutils.notebook.run()`.
- **Nine industry notebooks** each generate one schema with two tables (entity + event).
- All notebooks are Databricks-format Jupyter notebooks (`.ipynb`) with `%sql` magic cells and `dbutils` calls. They are **not runnable** as vanilla local Jupyter notebooks.

### Industry Schemas

| Schema | Entity Table | Event Table |
|---|---|---|
| finance | accounts | transactions |
| gaming | players | wager_records |
| health | patients | medical_records |
| insurance | policyholders | claims |
| manufacturing | equipment | production_orders |
| retail | customers | orders |
| telecom | subscribers | call_records |
| utilities | meters | usage_records |
| construction | projects | work_orders |

## Notebook Pipeline Pattern

Every industry notebook follows the same sequence:

1. `%pip install faker --quiet` → restart Python interpreter
2. Read `catalog` widget from `dbutils.widgets.get('catalog')`
3. SQL: `CREATE SCHEMA IF NOT EXISTS <catalog>.<industry>`
4. Python: generate entity rows as `Row` objects → `spark.createDataFrame()` → `.write.saveAsTable()`
5. Python: generate 100K–500K event rows linked to entity IDs → save as Delta table
6. SQL: add column `COMMENT`s, `PRIMARY KEY` / `FOREIGN KEY` constraints, schema description, `RemoveAfter` tag

## Key Technologies

- **PySpark** — all data generation uses `spark.createDataFrame()` and `saveAsTable()`
- **Faker** — names, emails, addresses, phone numbers (installed per-notebook via `%pip`)
- **Python `random` + `math`** — statistical distributions (log-normal, beta, gamma, weighted choices)
- **Databricks Unity Catalog** — catalog/schema/table DDL, column comments, governance tags
- **Databricks Asset Bundles** — `databricks.yml` defines the bundle for deployment

## Running

There are no local build/test commands. The project runs entirely on Databricks:

1. Import notebooks into a Databricks workspace (Unity Catalog enabled)
2. Open `[MASTER] Sample Dataset Generator.ipynb`
3. Set the `catalog` widget (default: `industry_sample_data`)
4. Run All (~15–20 minutes)

Individual industry notebooks can be run standalone if the catalog already exists.

## Conventions

### Naming
- Notebooks: `<Industry> Dataset Generator.ipynb`, master prefixed with `[MASTER]`
- Variables: `CATALOG` from widget, `SCHEMA` = industry name, `CATALOG_SCHEMA = f"{CATALOG}.{SCHEMA}"`
- SQL uses `IDENTIFIER(:catalog || '.<schema>')` for catalog-qualified references

### Code Style
- Python indent: 2 spaces in notebooks
- Seeded RNG for reproducibility: `Faker.seed(42)`, `random.seed(42)`
- Domain values defined as lists with weighted `random.choices()`
- Small helper functions (e.g., `clamp`) defined inline in notebook cells
- Databricks cell titles used via notebook metadata (`showTitle`, `title`)

### Data Generation
- Event records always reference valid entity IDs (referential integrity)
- Dates are temporally consistent (e.g., account opened before first transaction)
- Domain-specific logic: fraud patterns, seasonal effects, temperature correlations, churn mechanics, construction phase sequencing, weather delays

## Configuration

| Parameter | Default | Description |
|---|---|---|
| `catalog` | `industry_sample_data` | Target Unity Catalog. Dropped and recreated on each run. |

The `RemoveAfter` tag is set on both the catalog (in master) and each schema (in industry notebooks).

## Adding a New Industry

1. Copy an existing industry notebook and follow the same pattern (entity table + event table)
2. Accept the `catalog` widget via `dbutils.widgets.get('catalog')`
3. Add a `dbutils.notebook.run("<New Notebook Name>", ...)` call in the master notebook
4. Update `README.md` with the new industry's schema, tables, and features

## File Layout

```
[MASTER] Sample Dataset Generator.ipynb   # Orchestrator
Finance Dataset Generator.ipynb
Gaming Dataset Generator.ipynb
Health Dataset Generator.ipynb
Insurance Dataset Generator.ipynb
Manufacturing Dataset Generator.ipynb
Retail Dataset Generator.ipynb
Telecom Dataset Generator.ipynb
Utilities Dataset Generator.ipynb
Construction Dataset Generator.ipynb
databricks.yml                             # Asset bundle config
README.md
LICENSE                                    # MIT
```

## Important Gotchas

- Notebooks use `%sql` magic and `dbutils` — these only work in a Databricks runtime, not locally
- The master notebook **drops the entire catalog** (`DROP CATALOG ... CASCADE`) on every run
- `%pip install` is followed by `%restart_python` which resets the Python interpreter state
- There are no automated tests — validation is manual via the generated tables
- The `databricks.yml` workspace host points to a specific demo environment
