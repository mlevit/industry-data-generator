---
name: Realism Telecom
overview: "Implement Telecom-specific realism improvements: fix call_result and data_usage for non-voice records, usage-dependent churn model, add plans dimension, network_incidents table, and support_tickets table."
todos:
  - id: tel-call-result
    content: Set call_result to NULL for Data/SMS/MMS record types
    status: pending
  - id: tel-data-usage
    content: Set data_usage_mb to 0 for SMS/MMS record types
    status: pending
  - id: tel-churn-model
    content: Make churn probability depend on drop rate, overage, and quality metrics
    status: pending
  - id: tel-plans-dim
    content: Add plans dimension table with effective dates, features, pricing, and SCD rows
    status: pending
  - id: tel-incidents
    content: Add network_incidents table with tower outages, duration, and root cause
    status: pending
  - id: tel-tickets
    content: Add support_tickets table with issue type, priority, resolution time, CSAT
    status: pending
isProject: false
---

# Realism Improvements: Telecom

Target file: [Telecom Dataset Generator.ipynb](Telecom Dataset Generator.ipynb)

- **Cell 4** (index 4): Data generation — `subscribers` entity loop, `call_records` event loop
- **Cell 5** (index 5): Column comments and PK/FK constraints
- **Cell 6** (index 6): `%sql COMMENT ON SCHEMA` description

Key variables: `CATALOG_SCHEMA`, `NUM_SUBSCRIBERS`, `NUM_EVENT_RECORDS`, `sub_lookup`, `plan_profiles` (charges, limits, contract/device weights, 5g prob), `plans`, `device_types`, `contract_types`, `statuses`, `region_towers`, `record_types`, `call_results`, `roaming_countries`, `plan_record_weights`, `records`, `NOW = datetime(2026, 3, 21)`

---

## TODO 1: Set `call_result` to NULL for non-voice records

Currently all record types (Voice, Data, SMS, MMS, Voicemail) get a `call_result` from `call_results` list (Completed, Dropped, Failed, Busy, No Answer). This is meaningless for Data/SMS/MMS.

### Implementation (cell 4)

In the call records loop, after determining `record_type`, make `call_result` conditional:

```python
if record_type in ("Voice", "Voicemail"):
    call_result = random.choices(call_results, weights=[70, 10, 8, 7, 5])[0]
else:
    call_result = None
```

Replace the existing unconditional `call_result` assignment. Find it in the record generation block (approx. line 150-160 of cell 4).

### Verification

- Filter records where `record_type` in (Data, SMS, MMS) — all should have `call_result = None`.
- Voice/Voicemail records should still have a distributed `call_result`.

---

## TODO 2: Set `data_usage_mb` to 0 for SMS/MMS

Currently SMS/MMS records get a small exponential noise value (up to ~5 MB) instead of 0 or null.

### Implementation (cell 4)

In the call records loop, make `data_usage_mb` conditional on record type:

```python
if record_type == "Data":
    data_usage_mb = round(random.expovariate(1 / 250), 2)  # existing heavy usage
elif record_type == "Voice":
    data_usage_mb = round(random.expovariate(1 / 5), 2)  # small VoIP component
elif record_type == "Voicemail":
    data_usage_mb = round(random.uniform(0.1, 2.0), 2)
else:
    data_usage_mb = 0  # SMS and MMS have no data usage
```

Replace the existing `data_usage_mb` assignment that applies to all types.

### Verification

- All SMS/MMS records should have `data_usage_mb = 0`.
- Data records should retain their exponential distribution.

---

## TODO 3: Make churn probability depend on usage and quality metrics

Currently churn depends only on plan tier and tenure. It should incorporate experienced service quality.

### Implementation (cell 4)

This requires a two-pass approach since quality metrics come from call records.

**Pass 1:** Generate all call records as currently done (without churn filtering).

**Pass 2:** After generating all records, compute per-subscriber quality metrics:

```python
from collections import defaultdict

sub_metrics = defaultdict(lambda: {"total_records": 0, "dropped": 0, "data_total": 0.0, "overage_count": 0})
for rec in records:
    sid = rec["subscriber_id"]
    sub_metrics[sid]["total_records"] += 1
    if rec.get("call_result") == "Dropped":
        sub_metrics[sid]["dropped"] += 1
    sub_metrics[sid]["data_total"] += rec.get("data_usage_mb", 0)

# Recompute churn for each subscriber
for sub in subscribers:
    sid = sub["subscriber_id"]
    m = sub_metrics[sid]

    # Base churn from existing plan/tenure logic
    base_churn = sub.get("_base_churn_prob", 0.1)

    # Quality adjustments
    drop_rate = m["dropped"] / max(m["total_records"], 1)
    churn_prob = base_churn
    if drop_rate > 0.05: churn_prob += 0.10   # high drop rate
    if drop_rate > 0.10: churn_prob += 0.15   # very high drop rate
    if m["overage_count"] > 3: churn_prob += 0.08

    churn_prob = min(churn_prob, 0.95)
    sub["churned"] = random.random() < churn_prob
    if sub["churned"]:
        sub["churn_date"] = sub["activation_date"] + timedelta(days=random.randint(90, 730))
```

**Pass 3:** Filter out post-churn records:

```python
records = [r for r in records if not (sub_lookup[r["subscriber_id"]].get("churned")
           and r["record_date"] > sub_lookup[r["subscriber_id"]]["churn_date"])]
```

### Note

This restructures the current flow which computes churn during entity generation. Save the base churn probability as `_base_churn_prob` during entity generation, then adjust after records exist.

### Verification

- Subscribers with high drop rates should have higher churn rates.
- No records should exist after a subscriber's churn date.

---

## TODO 4: Add `plans` dimension with effective dates and features

Currently plan info is embedded in `plan_profiles` dict and denormalized onto subscriber rows.

### Implementation (cell 4)

Generate a plans dimension table before the subscribers loop:

```python
plan_records = []
plan_id = 0
for plan_name, profile in plan_profiles.items():
    # Current version of the plan
    plan_id += 1
    plan_records.append({
        "plan_id": plan_id,
        "plan_name": plan_name,
        "monthly_price": profile["monthly_charge"],
        "data_cap_gb": profile.get("data_limit_gb", None),
        "voice_minutes": profile.get("voice_limit", None),
        "sms_limit": profile.get("sms_limit", None),
        "contract_type": random.choice(contract_types),
        "includes_5g": profile.get("5g_prob", 0) > 0.5,
        "effective_date": date(2023, 1, 1),
        "end_date": None,
    })
    # Historical version (price was higher, less data)
    plan_id += 1
    plan_records.append({
        "plan_id": plan_id,
        "plan_name": plan_name,
        "monthly_price": round(profile["monthly_charge"] * 1.1, 2),
        "data_cap_gb": max(1, (profile.get("data_limit_gb", 10) - 5)) if profile.get("data_limit_gb") else None,
        "voice_minutes": profile.get("voice_limit", None),
        "sms_limit": profile.get("sms_limit", None),
        "contract_type": random.choice(contract_types),
        "includes_5g": False,
        "effective_date": date(2021, 1, 1),
        "end_date": date(2022, 12, 31),
    })

plan_name_to_current_id = {p["plan_name"]: p["plan_id"] for p in plan_records if p["end_date"] is None}
```

In the subscribers loop, assign `plan_id = plan_name_to_current_id[sub_plan]`. Add `plan_id` column to subscribers.

Write `{CATALOG_SCHEMA}.plans`.

### Cell 5 additions

- PK: `plan_id`
- FK on subscribers: `plan_id -> plans(plan_id)`

---

## TODO 5: Add `network_incidents` table

Outage events with duration, affected towers, and root cause.

### Implementation (cell 4)

Generate incidents after the towers are established in `region_towers`:

```python
incident_types = ["Outage", "Degradation", "Planned Maintenance"]
root_causes = ["Hardware Failure", "Power Outage", "Fiber Cut", "Software Bug",
               "Weather Damage", "Capacity Overload", "Planned Upgrade"]

all_towers = [(region, tower) for region, towers in region_towers.items() for tower in towers]

incidents = []
NUM_INCIDENTS = 200
for i in range(1, NUM_INCIDENTS + 1):
    region, tower_id = random.choice(all_towers)
    start_time = datetime(2024, 6, 1) + timedelta(
        days=random.randint(0, 600),
        hours=random.randint(0, 23),
        minutes=random.randint(0, 59)
    )
    duration_hours = random.expovariate(1 / 4)  # mean 4 hours
    end_time = start_time + timedelta(hours=duration_hours)
    inc_type = random.choices(incident_types, weights=[40, 35, 25])[0]
    incidents.append({
        "incident_id": i,
        "tower_id": tower_id,
        "region": region,
        "start_time": start_time,
        "end_time": end_time,
        "duration_hours": round(duration_hours, 2),
        "incident_type": inc_type,
        "root_cause": random.choice(root_causes),
        "affected_subscribers_est": random.randint(50, 5000),
    })
```

Write `{CATALOG_SCHEMA}.network_incidents`.

### Cell 5 additions

- PK: `incident_id`
- FK: `tower_id -> towers(tower_id)`

---

## TODO 6: Add `support_tickets` table

Customer service interactions with issue type, resolution, and CSAT.

### Implementation (cell 4)

Generate support tickets linked to subscribers:

```python
issue_types = ["Billing", "Network", "Device", "Account", "Plan Change", "Roaming", "Coverage"]
priorities = ["Low", "Medium", "High", "Critical"]
priority_weights = [30, 40, 20, 10]

tickets = []
NUM_TICKETS = random.randint(10000, 30000)
for i in range(1, NUM_TICKETS + 1):
    sid = random.randint(1, NUM_SUBSCRIBERS)
    sub = sub_lookup[sid]
    created = datetime(2024, 6, 1) + timedelta(
        days=random.randint(0, 600),
        hours=random.randint(8, 20),
        minutes=random.randint(0, 59)
    )
    priority = random.choices(priorities, weights=priority_weights)[0]
    resolution_hours = {
        "Low": random.expovariate(1 / 48),
        "Medium": random.expovariate(1 / 24),
        "High": random.expovariate(1 / 8),
        "Critical": random.expovariate(1 / 2),
    }[priority]
    resolved_date = created + timedelta(hours=resolution_hours)
    # CSAT inversely related to resolution time
    base_csat = max(1, min(5, int(5 - resolution_hours / 24)))
    csat = max(1, min(5, base_csat + random.randint(-1, 1)))
    tickets.append({
        "ticket_id": i,
        "subscriber_id": sid,
        "created_date": created,
        "issue_type": random.choice(issue_types),
        "priority": priority,
        "resolution_time_hours": round(resolution_hours, 2),
        "csat_score": csat,
        "resolved_date": resolved_date,
    })
```

Write `{CATALOG_SCHEMA}.support_tickets`.

### Cell 5 additions

- PK: `ticket_id`
- FK: `subscriber_id -> subscribers(subscriber_id)`

### Cell 6 update

Add all three new tables to schema comment with row counts, PKs, and FKs.
