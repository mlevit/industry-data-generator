---
name: Realism Finance
overview: "Implement Finance-specific realism improvements: daily_account_balances table, fraud_cases table, FX rate table, and velocity-based fraud detection."
todos:
  - id: fin-daily-bal
    content: Add daily_account_balances table with running balance per account per day
    status: pending
  - id: fin-fraud-cases
    content: Add fraud_cases table with investigation outcomes for fraud-flagged transactions
    status: pending
  - id: fin-fx-rates
    content: Add fx_rates table with daily exchange rates for all 7 currencies
    status: pending
  - id: fin-velocity
    content: Implement velocity-based fraud detection using sliding window in transaction loop
    status: pending
isProject: false
---

# Realism Improvements: Finance

Target file: [Finance Dataset Generator.ipynb](Finance Dataset Generator.ipynb)

- **Cell 4** (index 4): Data generation — main Python cell with `accounts` and `transactions` loops
- **Cell 5** (index 5): Column comments and PK/FK constraints
- **Cell 6** (index 6): `%sql COMMENT ON SCHEMA` description

Key variables: `CATALOG_SCHEMA`, `NUM_ACCOUNTS`, `NUM_EVENT_RECORDS`, `account_lookup`, `accounts`, `transactions`, `currencies` (USD/EUR/GBP/JPY/AUD/CAD/CHF), `currency_weights`

---

## TODO 1: Add `daily_account_balances` table

Time-series snapshots for liquidity, overdraft, and trend analysis.

### Implementation (cell 4)

After the transactions loop and before writing DataFrames, aggregate transactions into daily balances:

```python
from collections import defaultdict

daily_movements = defaultdict(lambda: defaultdict(lambda: {"credits": 0.0, "debits": 0.0}))
for txn in transactions:
    day_key = txn["transaction_date"].date()
    acct_key = txn["account_id"]
    if txn["transaction_type"] == "Credit":
        daily_movements[acct_key][day_key]["credits"] += txn["amount"]
    else:
        daily_movements[acct_key][day_key]["debits"] += txn["amount"]

daily_balances = []
bal_id = 0
for acct in accounts:
    acct_id = acct["account_id"]
    running = round(random.uniform(500, 50000), 2)  # initial balance
    movements = sorted(daily_movements.get(acct_id, {}).items())
    for day, mv in movements:
        bal_id += 1
        opening = running
        credits = round(mv["credits"], 2)
        debits = round(mv["debits"], 2)
        running = round(opening + credits - debits, 2)
        daily_balances.append({
            "balance_id": bal_id, "account_id": acct_id,
            "balance_date": day, "opening_balance": opening,
            "credits": credits, "debits": debits, "closing_balance": running
        })
```

Write as `{CATALOG_SCHEMA}.daily_account_balances`.

### Cell 5 additions

- `apply_comments` for all 7 columns
- `ALTER TABLE ... ALTER COLUMN balance_id SET NOT NULL`
- `ADD CONSTRAINT pk_daily_account_balances PRIMARY KEY (balance_id)`
- `ADD CONSTRAINT fk_daily_account_balances_account_id FOREIGN KEY (account_id) REFERENCES {CATALOG_SCHEMA}.accounts(account_id)`

### Cell 6 update

Add to schema comment: `daily_account_balances` (~X rows, PK: balance_id, FK: account_id -> accounts)

---

## TODO 2: Add `fraud_cases` table

Investigation outcomes, resolution status, and chargeback amounts for supervised ML.

### Implementation (cell 4)

After transaction generation, iterate over fraud-flagged transactions:

```python
investigation_statuses = ["Open", "Under Review", "Closed"]
resolutions = ["Confirmed Fraud", "False Positive", "Chargeback", "Account Frozen"]
fraud_cases = []
case_id = 0
for txn in transactions:
    if txn.get("fraud_suspected"):
        case_id += 1
        reported = txn["transaction_date"] + timedelta(days=random.randint(0, 7))
        status = random.choices(investigation_statuses, weights=[15, 25, 60])[0]
        resolution = random.choice(resolutions) if status == "Closed" else None
        resolved_date = reported + timedelta(days=random.randint(3, 90)) if status == "Closed" else None
        chargeback = round(txn["amount"] * random.uniform(0.5, 1.0), 2) if resolution == "Chargeback" else 0.0
        fraud_cases.append({
            "case_id": case_id, "transaction_id": txn["transaction_id"],
            "account_id": txn["account_id"], "reported_date": reported,
            "investigation_status": status, "resolution": resolution,
            "chargeback_amount": chargeback, "resolved_date": resolved_date
        })
```

Write as `{CATALOG_SCHEMA}.fraud_cases`.

### Cell 5 additions

- `apply_comments` for all 8 columns
- PK: `case_id`
- FK: `transaction_id -> transactions`, `account_id -> accounts`

---

## TODO 3: Add FX rate table

Daily exchange rates for cross-currency analytics across the 7 existing currencies.

### Implementation (cell 4)

```python
base_rates = {
    ("USD","EUR"): 0.92, ("USD","GBP"): 0.79, ("USD","JPY"): 149.5,
    ("USD","AUD"): 1.53, ("USD","CAD"): 1.36, ("USD","CHF"): 0.88
}

fx_rates = []
rate_id = 0
for day_offset in range(730):
    rate_date = date(2024, 1, 1) + timedelta(days=day_offset)
    for (base, quote), mid in base_rates.items():
        rate_id += 1
        drift = mid * random.gauss(0, 0.002)  # daily random walk
        base_rates[(base, quote)] = mid + drift
        fx_rates.append({
            "rate_id": rate_id, "base_currency": base,
            "quote_currency": quote, "rate_date": rate_date,
            "exchange_rate": round(mid + drift, 6)
        })
```

Write as `{CATALOG_SCHEMA}.fx_rates`. ~4,380 rows (730 days x 6 pairs).

### Cell 5/6

- PK: `rate_id`
- No FK (reference/dimension table)
- Schema comment: `fx_rates` (~4K rows, PK: rate_id)

---

## TODO 4: Implement velocity-based fraud detection

Sliding-window fraud scoring over recent transactions per account.

### Implementation (cell 4)

In the transactions loop, maintain a deque-based sliding window:

```python
from collections import deque

recent_txns = defaultdict(lambda: deque(maxlen=20))

# Inside the transaction loop, after computing amount:
window = recent_txns[acct_id]
# Velocity checks
txns_last_hour = sum(1 for t in window if (txn_ts - t[0]).total_seconds() < 3600)
sum_last_day = sum(t[1] for t in window if (txn_ts - t[0]).total_seconds() < 86400)
if txns_last_hour >= 5: fraud_prob += 0.05
if sum_last_day > 10000: fraud_prob += 0.03
window.append((txn_ts, amount))
```

This augments the existing amount/channel-based fraud logic (around line 140 of cell 4) with temporal velocity checks.

### Verification

- Count fraud-flagged transactions; should increase slightly with velocity checks.
- Spot-check that high-frequency accounts get more fraud flags.
