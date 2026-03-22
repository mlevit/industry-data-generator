---
name: Realism Cross-Cutting
overview: "Implement cross-cutting realism improvements that apply to all 8 industry notebooks: time-series patterns, heavy-tail sampling, SCD history tables, running balances, and probability capping."
todos:
  - id: cc-time-series
    content: Add time-series realism helpers and replace uniform timestamps in all 8 notebooks
    status: pending
  - id: cc-pareto
    content: Implement heavy-tail Pareto entity sampling in all 8 notebooks
    status: pending
  - id: cc-scd
    content: Add slowly changing dimension history tables to all 8 notebooks
    status: pending
  - id: cc-ledger
    content: Add running balance / ledger reconciliation to Finance and Gaming
    status: pending
  - id: cc-cap-prob
    content: Cap fraud/risk probability at 1.0 in Finance and Insurance
    status: pending
isProject: false
---

# Realism Improvements: Cross-Cutting

Changes apply to **all 8 notebooks** in cell 4 (data generation), cell 5 (comments/constraints), and cell 6 (schema comment).

---

## TODO 1: Add time-series realism

Replace uniform random timestamps with seasonal, weekly, and hourly patterns.

### Shared helper functions (add near top of cell 4 in each notebook)

```python
HOUR_WEIGHTS = [0.5,0.3,0.2,0.2,0.3,0.5,1.0,2.0,3.0,3.5,3.5,3.0,2.5,3.0,3.5,3.5,3.0,2.5,2.0,1.5,1.2,1.0,0.8,0.6]
DOW_WEIGHTS  = [1.2, 1.3, 1.3, 1.2, 1.1, 0.7, 0.5]  # Mon-Sun
MONTH_SEASONALITY = [0.85,0.80,0.95,1.00,1.05,1.10,1.05,1.00,1.05,1.10,1.20,1.30]

def realistic_timestamp(base_year=2024, span_days=730):
    day_offset = random.randint(0, span_days)
    d = date(base_year, 1, 1) + timedelta(days=day_offset)
    month_factor = MONTH_SEASONALITY[d.month - 1]
    dow_factor = DOW_WEIGHTS[d.weekday()]
    if random.random() > month_factor * dow_factor / 1.69:
        return realistic_timestamp(base_year, span_days)  # rejection sampling
    hour = random.choices(range(24), weights=HOUR_WEIGHTS)[0]
    minute = random.randint(0, 59)
    return datetime(d.year, d.month, d.day, hour, minute)
```

### Where to apply (per notebook, cell 4 event loops)

| Notebook      | Current pattern                                                                                       | Variable to replace                            |
| ------------- | ----------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| Finance       | `datetime(2024,1,1) + timedelta(days=random.randint(0,730), hours=..., minutes=...)`                  | `txn_ts`                                       |
| Gaming        | `datetime(2024,1,1) + timedelta(days=random.randint(0,730), hours=..., minutes=...)`                  | `wager_ts`                                     |
| Health        | `datetime(2024,1,1) + timedelta(days=random.randint(0,730), hours=random.randint(7,19), minutes=...)` | `visit_ts` (keep ER/urgent-care hour override) |
| Insurance     | `datetime(2024,1,1) + timedelta(days=random.randint(0,730), hours=..., minutes=...)`                  | `claim_ts`                                     |
| Manufacturing | `date(2024,1,1) + timedelta(days=random.randint(0,730))`                                              | `start_dt` (date only, skip hour weights)      |
| Retail        | `date(2024,1,1) + timedelta(days=random.randint(0,730))`                                              | `order_date` (date only, skip hour weights)    |
| Telecom       | `datetime(2024,6,1) + timedelta(days=random.randint(0,600), hours=..., minutes=...)`                  | `record_ts`                                    |
| Utilities     | `datetime(2024,1,1) + timedelta(days=random.randint(0,730), hours=random.randint(0,23))`              | `reading_ts`                                   |

Each notebook may need industry-specific seasonality tweaks (e.g., Retail December boost already exists via `holiday_boost`; merge with `MONTH_SEASONALITY`).

### Verification

- Print a histogram of event counts by month and by hour-of-day to confirm non-uniform distribution.
- Ensure total event count remains within the `NUM_EVENT_RECORDS` target.

---

## TODO 2: Use heavy-tail (Pareto) entity sampling

Replace uniform entity ID selection with Pareto-weighted sampling so a small fraction of entities generates most activity.

### Implementation pattern

```python
import numpy as np

def pareto_weights(n, alpha=1.5):
    """Generate Pareto-distributed weights for n entities."""
    raw = np.random.default_rng(42).pareto(alpha, n) + 1
    return raw / raw.sum()

entity_weights = pareto_weights(NUM_ENTITIES)
```

### Where to apply (per notebook, cell 4 event loops)

| Notebook      | Current sampling                            | Replace with                                                                     |
| ------------- | ------------------------------------------- | -------------------------------------------------------------------------------- |
| Finance       | `acct_id = random.randint(1, NUM_ACCOUNTS)` | `acct_id = random.choices(range(1, NUM_ACCOUNTS+1), weights=account_weights)[0]` |
| Gaming        | `pid = random.randint(1, NUM_PLAYERS)`      | `pid = random.choices(range(1, NUM_PLAYERS+1), weights=player_weights)[0]`       |
| Health        | `pid = random.choice(patient_pool)`         | Already weighted via `patient_pool`; make weights more heavy-tailed              |
| Insurance     | `pol_id = random.choice(claim_pool)`        | Already weighted via `claim_pool`; make weights more heavy-tailed                |
| Manufacturing | `eid = random.randint(1, NUM_EQUIPMENT)`    | `eid = random.choices(range(1, NUM_EQUIPMENT+1), weights=equip_weights)[0]`      |
| Retail        | `cust_id = random.choice(customer_pool)`    | Already weighted via `customer_pool`; make weights more heavy-tailed             |
| Telecom       | `sid = random.randint(1, NUM_SUBSCRIBERS)`  | `sid = random.choices(range(1, NUM_SUBSCRIBERS+1), weights=sub_weights)[0]`      |
| Utilities     | `meter_id = random.choice(meter_ids)`       | `meter_id = random.choices(meter_ids, weights=meter_weights)[0]`                 |

### Verification

- Compute entity activity distribution: top 10% of entities should generate ~50-60% of events.
- Print Gini coefficient or percentile breakdown.

---

## TODO 3: Introduce slowly changing dimensions

Add `_history` tables that record attribute transitions over time for key entity attributes.

### Pattern (per notebook)

For each entity table, identify 1-2 mutable attributes and generate a change history:

| Notebook      | Entity          | Mutable attributes                     | History table               |
| ------------- | --------------- | -------------------------------------- | --------------------------- |
| Finance       | `accounts`      | `status`, `balance`                    | `account_status_history`    |
| Gaming        | `players`       | `vip_tier`, `account_status`           | `player_status_history`     |
| Health        | `patients`      | `insurance_type`, `primary_physician`  | `patient_insurance_history` |
| Insurance     | `policyholders` | `policy_status`, `coverage_amount`     | `policy_status_history`     |
| Manufacturing | `equipment`     | `status`, `efficiency_rating`          | `equipment_status_history`  |
| Retail        | `customers`     | `membership_tier`, `preferred_channel` | `customer_tier_history`     |
| Telecom       | `subscribers`   | `plan`, `status`                       | `subscriber_plan_history`   |
| Utilities     | `meters`        | `meter_status`, `rate_plan`            | `meter_plan_history`        |

### Implementation steps

1. After generating entities in cell 4, loop through each entity and generate 0-3 change events:

```python
   for entity in entities:
       num_changes = random.choices([0,1,2,3], weights=[40,35,20,5])[0]
       for _ in range(num_changes):
           change_date = entity.created_date + timedelta(days=random.randint(30, 730))
           old_value = current_attr
           new_value = random.choice(possible_values - {old_value})
           history_rows.append(...)


```

1. Write history table with: `history_id`, `entity_id`, `attribute_name`, `old_value`, `new_value`, `effective_date`, `changed_by`
2. Add PK/FK constraints in cell 5
3. Update schema comment in cell 6

---

## TODO 4: Add running balance / ledger reconciliation

Applies to **Finance** and **Gaming** only.

### Finance

In cell 4, after generating all transactions:

```python
from collections import defaultdict
account_running = defaultdict(float)
for txn in sorted(transactions, key=lambda t: t.transaction_date):
    delta = txn.amount if txn.transaction_type == "Credit" else -txn.amount
    account_running[txn.account_id] += delta

# Update account balance to match ledger
for acct in accounts:
    acct.balance = round(account_running[acct.account_id] + acct.initial_balance, 2)
```

### Gaming

After generating all wagers, compute `lifetime_deposits` and `lifetime_withdrawals` from actual `payment_transactions` (if that table exists) or from wager totals, and set the player's running balance accordingly.

### Verification

- For Finance: `sum(credits) - sum(debits) + initial_balance == final_balance` for every account.
- For Gaming: `lifetime_deposits - lifetime_withdrawals == current_balance` for every player.

---

## TODO 5: Cap fraud probability at 1.0

### Finance (cell 4)

The current code stacks bonuses:

```python
fraud_prob = 0.005
if amount > 2000: fraud_prob += 0.02
if amount > 5000: fraud_prob += 0.03
if cat == "Purchase" and channel == "Wire": fraud_prob += 0.04
if cat == "Withdrawal" and amount > 3000: fraud_prob += 0.02
```

While this currently peaks at ~0.115, add a safety clamp after all bonuses:

```python
fraud_prob = min(fraud_prob, 1.0)
fraud_suspected = random.random() < fraud_prob
```

### Insurance (cell 4)

Check for similar `fraud_prob` or `fraud_indicator` stacking in the claims loop. Apply the same `min(..., 1.0)` pattern.

### Verification

- Assert no `fraud_prob > 1.0` values exist after generation.
