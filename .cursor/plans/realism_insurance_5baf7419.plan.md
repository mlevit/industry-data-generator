---
name: Realism Insurance
overview: "Implement Insurance-specific realism improvements: cap claim amounts, add occurrence/report dates, fix business days, replace hash() adjuster IDs, differentiate Life vs P&C, add coverage_items and catastrophe_events tables."
todos:
  - id: ins-cap-claim
    content: Cap claim_amount at coverage_amount minus deductible
    status: pending
  - id: ins-dates
    content: Add occurrence_date and report_date; enforce policy period validity
    status: pending
  - id: ins-biz-days
    content: Fix days_to_resolve to use business days instead of calendar days
    status: pending
  - id: ins-adjuster
    content: Replace hash()-based adjuster IDs with pre-generated deterministic surrogate keys
    status: pending
  - id: ins-life
    content: Differentiate Life insurance (mortality premium, term, beneficiary) from P&C
    status: pending
  - id: ins-coverage
    content: Add coverage_items table with per-peril limits and deductibles
    status: pending
  - id: ins-cat
    content: Add catastrophe_events table and link correlated property claims
    status: pending
isProject: false
---

# Realism Improvements: Insurance

Target file: [Insurance Dataset Generator.ipynb](Insurance Dataset Generator.ipynb)

- **Cell 4** (index 4): Data generation — `people` tuples, `policyholders` loop, `claims` loop
- **Cell 5** (index 5): Column comments and PK/FK constraints
- **Cell 6** (index 6): `%sql COMMENT ON SCHEMA` description

Key variables: `CATALOG_SCHEMA`, `NUM_PEOPLE` (15000), `policy_id_counter`, `policyholders`, `ph_lookup`, `person_policies`, `claim_pool`, `policy_claim_count`, `claims`, `states_config`, `region_agents`, `policy_types` (Auto/Home/Life/Health/Renters), `policy_params`, `deductible_options`, `NOW = date(2026, 3, 21)`

---

## TODO 1: Cap `claim_amount` at `coverage_amount` or model sublimits

Currently `claim_amount` is drawn independently and can exceed the policy's `coverage_amount`.

### Implementation (cell 4)

In the claims loop, after drawing `claim_amount` from `claim_amount_params`, clamp:

```python
deductible = ph["deductible"]
max_payable = ph["coverage_amount"] - deductible
claim_amount = min(claim_amount, ph["coverage_amount"])
approved_amount = min(approved_amount, max_payable) if approved_amount else 0
```

Find the existing `claim_amount` assignment (approx. line 230-240 of cell 4) and insert the clamp immediately after.

### Verification

- Assert `claim_amount <= coverage_amount` for every claim.
- Assert `approved_amount <= coverage_amount - deductible` for approved claims.

---

## TODO 2: Distinguish policy period, report date, and occurrence date

Currently claims have only `claim_date`. Real claims have three temporal anchors.

### Implementation (cell 4)

Replace or augment the single `claim_ts` with three dates:

```python
# Policy period
policy_start = ph["policy_start_date"]
policy_end = policy_start + timedelta(days=365 * ph.get("tenure_years", 1))

# Occurrence must be within active policy period
occurrence_date = policy_start + timedelta(days=random.randint(0, (min(policy_end, NOW) - policy_start).days))
report_delay = random.choices([0,1,2,3,5,7,14,30], weights=[30,20,15,10,8,7,5,5])[0]
report_date = occurrence_date + timedelta(days=report_delay)

# Only generate claims for policies that were active at occurrence time
if ph["policy_status"] in ("Lapsed", "Cancelled") and occurrence_date > ph.get("termination_date", NOW):
    continue  # skip this claim
```

Add `occurrence_date` and `report_date` columns to each claim record. Keep `claim_date` as an alias for `report_date` for backward compatibility, or rename entirely.

### Cell 5/6

- Add comments for `occurrence_date` and `report_date`
- Update schema comment to mention the three-date model

---

## TODO 3: Fix `days_to_resolve` to use business days

Documented as business days but generated as calendar days via `int(random.lognormvariate(...))`.

### Implementation (cell 4)

Add a helper function near the top:

```python
def calendar_to_business_days(calendar_days):
    """Convert calendar days to approximate business days (exclude weekends)."""
    full_weeks = calendar_days // 7
    remainder = calendar_days % 7
    return full_weeks * 5 + min(remainder, 5)

def business_to_calendar_days(business_days):
    """Convert business days back to calendar days for date arithmetic."""
    full_weeks = business_days // 5
    remainder = business_days % 5
    return full_weeks * 7 + remainder
```

In the claims loop, generate `days_to_resolve` as business days:

```python
raw_calendar = int(random.lognormvariate(3.0, 0.8))
days_to_resolve = calendar_to_business_days(raw_calendar)
resolved_date = report_date + timedelta(days=business_to_calendar_days(days_to_resolve))
```

### Verification

- Confirm `days_to_resolve` values are ~71% of the raw calendar day draws (5/7 ratio).

---

## TODO 4: Replace `hash()`-based adjuster IDs with deterministic surrogate keys

Python `hash()` is non-deterministic across processes unless `PYTHONHASHSEED` is set.

### Implementation (cell 4)

Pre-generate adjusters and assign by region:

```python
NUM_ADJUSTERS_PER_REGION = 5
adjusters = {}
adj_id = 0
for region in region_agents.keys():
    adjusters[region] = []
    for _ in range(NUM_ADJUSTERS_PER_REGION):
        adj_id += 1
        adjusters[region].append(f"ADJ-{adj_id:05d}")
```

In the claims loop, replace the `hash()`-based assignment:

```python
# Old: adjuster_id = f"ADJ-{abs(hash(region)) % 100:03d}"
# New:
adjuster_id = random.choice(adjusters[ph["region"]])
```

### Verification

- Adjuster IDs should be stable across runs with the same seed.
- Each region should have exactly `NUM_ADJUSTERS_PER_REGION` distinct adjusters.

---

## TODO 5: Differentiate Life insurance from P&C

Currently all policy types share the same premium/claim structure.

### Implementation (cell 4)

In the policyholders loop, when `policy_type == "Life"`, apply different logic:

```python
if assigned_type == "Life":
    # Mortality-based premium: increases with age
    base_premium = 200
    age_factor = 1.0 + (age - 30) * 0.03 if age > 30 else 1.0
    smoker_factor = 1.8 if random.random() < 0.2 else 1.0
    annual_premium = round(base_premium * age_factor * smoker_factor, 2)

    term_years = random.choice([10, 15, 20, 25, 30])
    coverage_amount = random.choice([100000, 250000, 500000, 750000, 1000000])
    cash_value = round(coverage_amount * random.uniform(0, 0.3), 2) if term_years > 15 else 0
    beneficiary_name = fake.name()
```

Add columns to the policyholder record: `term_years` (null for P&C), `cash_value` (null for P&C), `beneficiary_name` (null for P&C), `is_smoker` (null for P&C).

In the claims loop, Life insurance claims represent death benefits — set `claim_type = "Death Benefit"` and `claim_amount = coverage_amount`.

### Cell 5/6

- Add comments for the 4 new columns
- Update schema comment to note Life vs P&C distinction

---

## TODO 6: Add `coverage_items` table

Per-peril limits and deductibles for each policy.

### Implementation (cell 4)

Define peril mappings by policy type:

```python
perils_by_type = {
    "Auto": [("Collision", 0.7), ("Comprehensive", 0.5), ("Liability", 1.0), ("Uninsured Motorist", 0.4)],
    "Home": [("Dwelling", 1.0), ("Personal Property", 0.8), ("Liability", 0.6), ("Loss of Use", 0.3)],
    "Renters": [("Personal Property", 1.0), ("Liability", 0.6), ("Loss of Use", 0.3)],
    "Health": [("Inpatient", 1.0), ("Outpatient", 1.0), ("Prescription", 0.8), ("Mental Health", 0.5)],
    "Life": [],  # Life has no sub-perils
}
```

After generating policyholders, generate coverage items:

```python
coverage_items = []
ci_id = 0
for ph in policyholders:
    perils = perils_by_type.get(ph["policy_type"], [])
    for peril_name, inclusion_prob in perils:
        if random.random() < inclusion_prob:
            ci_id += 1
            limit = round(ph["coverage_amount"] * random.uniform(0.3, 1.0), 2)
            ded = random.choice([250, 500, 1000, 2000, 5000])
            coverage_items.append({
                "coverage_item_id": ci_id,
                "policy_id": ph["policy_id"],
                "peril": peril_name,
                "limit_amount": limit,
                "deductible_amount": ded,
            })
```

Write as `{CATALOG_SCHEMA}.coverage_items`.

### Cell 5 additions

- PK: `coverage_item_id`
- FK: `policy_id -> policyholders(policy_id)`

---

## TODO 7: Add `catastrophe_events` table

Correlated property losses by region for Home/Renters claims.

### Implementation (cell 4)

Generate catastrophe events:

```python
cat_types = ["Hurricane", "Earthquake", "Wildfire", "Tornado", "Flood", "Hailstorm", "Winter Storm"]
cat_events = []
for i in range(1, 11):  # 10 events over 2-year window
    event_date = date(2024, 1, 1) + timedelta(days=random.randint(0, 730))
    region = random.choice(state_names)
    cat_events.append({
        "cat_event_id": i,
        "event_date": event_date,
        "event_type": random.choice(cat_types),
        "region": region,
        "severity_score": round(random.uniform(1, 10), 1),
        "estimated_insured_loss": round(random.uniform(1_000_000, 500_000_000), 2),
    })
```

In the claims loop for Home/Renters policies, with ~10% probability link the claim to a matching cat event:

```python
matching_cats = [c for c in cat_events
                 if c["region"] == ph["state"]
                 and abs((claim_date - c["event_date"]).days) < 30]
cat_event_id = random.choice(matching_cats)["cat_event_id"] if matching_cats and random.random() < 0.10 else None
```

Write as `{CATALOG_SCHEMA}.catastrophe_events`.

### Cell 5 additions

- PK: `cat_event_id`
- FK on claims: `cat_event_id -> catastrophe_events(cat_event_id)`
