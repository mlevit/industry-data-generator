---
name: Realism Utilities
overview: "Implement Utilities-specific realism improvements: time-of-use pricing, outage-reduced usage, net metering credits, explicit interval length, correlated peak demand, add outage_events, billing/invoices, and weather_observations tables."
todos:
  - id: util-tou
    content: Implement time-of-use pricing with peak/shoulder/off-peak rates by hour and plan type
    status: pending
  - id: util-outage-usage
    content: Make outages reduce usage proportional to outage duration
    status: pending
  - id: util-net-meter
    content: Model net metering credits for negative (solar export) usage instead of abs()
    status: pending
  - id: util-interval
    content: Add explicit interval_type column (daily vs monthly) to usage records
    status: pending
  - id: util-peak-demand
    content: Derive peak_demand_kw from usage amount instead of independent log-normal
    status: pending
  - id: util-outage-tbl
    content: Add outage_events table with start/end, cause, zone, SAIDI/SAIFI metrics
    status: pending
  - id: util-invoices
    content: Add billing_invoices table with monthly charges, payment status, arrears
    status: pending
  - id: util-weather
    content: Add weather_observations table with station-level hourly temperature/humidity/wind
    status: pending
isProject: false
---

# Realism Improvements: Utilities

Target file: [Utilities Dataset Generator.ipynb](Utilities Dataset Generator.ipynb)

- **Cell 4** (index 4): Data generation — `customers` dicts with nested `services`, `meters` loop, `usage_records` event loop
- **Cell 5** (index 5): Column comments and PK/FK constraints
- **Cell 6** (index 6): `%sql COMMENT ON SCHEMA` description

Key variables: `CATALOG_SCHEMA`, `NUM_CUSTOMERS`, `NUM_EVENT_RECORDS`, `customers` (list of dicts with `services`), `meters` (from nested cust/svc loop), `meter_lookup`, `meter_ids`, `service_types` (Electric/Gas/Water), `meter_statuses`, `regions`, `cities`, `region_grid_zones`, `service_rate_plans` (per service type), `outage_events` (keyed by `(grid_zone, month)`), `temp_baselines` (by region/month), `records`, `NOW = datetime.now()`

**Note:** Unlike other notebooks, this one uses `NOW = datetime.now()` instead of a fixed date. Consider changing to `NOW = datetime(2026, 3, 21)` for reproducibility as a prerequisite fix.

---

## TODO 1: Implement time-of-use pricing logic

Currently `rate_per_unit` is random per reading, not tied to `rate_plan` or hour-of-day. Two meters on the same TOU plan can have wildly different rates.

### Implementation (cell 4)

Add a TOU schedule and base rates near the top:

```python
base_rates_by_plan = {
    "Electric": {
        "Flat Rate": {"rate": 0.12},
        "Time-of-Use": {"peak": 0.22, "shoulder": 0.14, "off_peak": 0.08},
        "Tiered": {"tier1_rate": 0.10, "tier1_limit": 500, "tier2_rate": 0.15, "tier2_limit": 1000, "tier3_rate": 0.22},
        "EV Rate": {"peak": 0.25, "shoulder": 0.15, "off_peak": 0.06},
    },
    "Gas": {
        "Flat Rate": {"rate": 1.05},
        "Seasonal": {"summer": 0.85, "winter": 1.35},
        "Budget Billing": {"rate": 1.10},
    },
    "Water": {
        "Flat Rate": {"rate": 0.005},
        "Conservation": {"tier1_rate": 0.004, "tier1_limit": 500, "tier2_rate": 0.008, "tier2_limit": 1000, "tier3_rate": 0.015},
        "Irrigation": {"rate": 0.006},
    },
}

tou_hours = {
    "peak": range(16, 21),       # 4 PM - 9 PM
    "shoulder": list(range(7, 16)) + list(range(21, 23)),  # 7 AM - 4 PM, 9 PM - 11 PM
    "off_peak": list(range(0, 7)) + [23],  # 11 PM - 7 AM
}

def get_tou_period(hour):
    if hour in tou_hours["peak"]: return "peak"
    if hour in tou_hours["shoulder"]: return "shoulder"
    return "off_peak"

def get_rate(service_type, rate_plan, hour=None, usage=None, month=None):
    plans = base_rates_by_plan.get(service_type, {})
    plan = plans.get(rate_plan, {"rate": 0.10})

    if "peak" in plan and hour is not None:
        period = get_tou_period(hour)
        return plan[period]
    elif "tier1_rate" in plan and usage is not None:
        if usage <= plan["tier1_limit"]: return plan["tier1_rate"]
        if usage <= plan["tier2_limit"]: return plan["tier2_rate"]
        return plan["tier3_rate"]
    elif "summer" in plan and month is not None:
        return plan["summer"] if 5 <= month <= 9 else plan["winter"]
    else:
        return plan.get("rate", 0.10)
```

In the usage records loop, replace the random `rate_per_unit` with:

```python
hour = reading_ts.hour
month = reading_ts.month
rate_per_unit = get_rate(meter["service_type"], meter["rate_plan"], hour=hour, usage=usage_amount, month=month)
```

### Verification

- All readings for the same meter/plan should have consistent rates (or predictable TOU variation).
- TOU plan readings at peak hours (4-9 PM) should have the highest rate.

---

## TODO 2: Make outages reduce usage

Currently `had_outage` and `outage_duration_minutes` are independent of `usage_amount`.

### Implementation (cell 4)

In the usage records loop, after determining `had_outage` and `outage_duration_minutes`, adjust usage:

```python
if had_outage and outage_duration_minutes > 0:
    # Reduce usage proportional to outage duration (assume 24-hour reading interval)
    outage_fraction = min(outage_duration_minutes / (24 * 60), 1.0)
    usage_amount *= (1.0 - outage_fraction)
    usage_amount = max(0, round(usage_amount, 2))
```

Insert this immediately after the existing `usage_amount` calculation and before the `cost` calculation.

### Verification

- Readings with long outages (e.g., 12+ hours) should have significantly reduced usage.
- A 24-hour outage should produce near-zero usage.
- Readings with no outage should be unaffected.

---

## TODO 3: Model net metering credits for solar export

Currently `cost = abs(usage) * rate` eliminates the negative usage (export) signal.

### Implementation (cell 4)

In the usage records loop, replace the cost calculation:

```python
# Old: cost = round(abs(usage_amount) * rate_per_unit, 2)
# New:
if usage_amount < 0:
    # Net metering: export credit at a reduced rate
    net_metering_rate = rate_per_unit * 0.75  # credit at 75% of retail rate
    cost = round(usage_amount * net_metering_rate, 2)  # negative cost = credit
    net_metering_credit = round(abs(cost), 2)
else:
    cost = round(usage_amount * rate_per_unit, 2)
    net_metering_credit = 0.0
```

Add `net_metering_credit` column to the usage record dict.

### Cell 5/6

- Add comment for `net_metering_credit`: "Credit amount for solar/DER export under net metering; 0 for consumption-only readings"
- Update schema comment

### Verification

- Negative-usage readings should have negative cost and positive `net_metering_credit`.
- Positive-usage readings should have `net_metering_credit = 0`.

---

## TODO 4: Define explicit interval length

Currently ambiguous whether readings are daily or monthly. Usage caps (300 kWh electric, 1500 gal water) suggest daily, but this is undocumented.

### Implementation (cell 4)

Add `interval_type` column. Assign based on meter type:

```python
# Smart meters report daily, traditional meters report monthly
if meter.get("is_smart_meter", False):
    interval_type = "daily"
    # Keep existing usage caps for daily
else:
    interval_type = "monthly"
    # Scale up usage for monthly readings
    usage_amount *= 30  # approximate monthly from daily generation logic
```

Add `interval_type` to each record dict.

Alternatively, if all readings should be the same interval, pick one (daily) and document it:

```python
interval_type = "daily"
```

### Cell 5/6

- Add comment for `interval_type`: "Reading interval — 'daily' for smart meters, 'monthly' for traditional"
- Update schema comment

---

## TODO 5: Correlate `peak_demand_kw` with interval usage

Currently drawn from an independent log-normal distribution.

### Implementation (cell 4)

Replace the independent draw with a derivation from usage:

```python
# Old: peak_demand_kw = round(random.lognormvariate(1.5, 0.5), 2) if service_type == "Electric" else None
# New:
if service_type == "Electric":
    # Peak demand derived from usage and interval
    hours_in_interval = 24 if interval_type == "daily" else 720
    avg_demand = usage_amount / max(hours_in_interval, 1)
    # Peak is 1.5-3x average demand
    demand_factor = random.uniform(1.5, 3.0)
    peak_demand_kw = round(avg_demand * demand_factor, 2)
else:
    peak_demand_kw = None
```

### Verification

- `peak_demand_kw` should be positively correlated with `usage_amount` for electric meters.
- Scatter plot of usage vs peak demand should show clear positive trend.

---

## TODO 6: Add `outage_events` table

Proper event table replacing the per-reading boolean with structured outage records.

### Implementation (cell 4)

Generate outage events before the usage records loop (replaces or supplements the existing `outage_events` dict keyed by `(grid_zone, month)`):

```python
outage_causes = ["Storm", "Equipment Failure", "Planned Maintenance", "Vegetation",
                 "Vehicle Accident", "Animal Contact", "Overload", "Unknown"]

outage_event_records = []
outage_id = 0
for grid_zone in set(region_grid_zones.values()) if isinstance(region_grid_zones, dict) else region_grid_zones:
    num_outages = random.randint(2, 15)  # per zone over 2 years
    for _ in range(num_outages):
        outage_id += 1
        start = datetime(2024, 1, 1) + timedelta(
            days=random.randint(0, 730),
            hours=random.randint(0, 23),
            minutes=random.randint(0, 59)
        )
        duration_minutes = int(random.expovariate(1 / 180))  # mean 3 hours
        end = start + timedelta(minutes=duration_minutes)
        cause = random.choices(outage_causes, weights=[25, 20, 15, 10, 8, 7, 10, 5])[0]
        affected = random.randint(10, 2000)
        saidi = round(duration_minutes * affected / 5000, 2)  # simplified
        saifi = round(affected / 5000, 4)
        outage_event_records.append({
            "outage_id": outage_id,
            "grid_zone": grid_zone,
            "start_time": start,
            "end_time": end,
            "duration_minutes": duration_minutes,
            "cause": cause,
            "affected_meters_count": affected,
            "saidi_minutes": saidi,
            "saifi_count": saifi,
        })
```

In the usage records loop, when `had_outage=True`, link to the nearest matching outage event:

```python
zone = meter["grid_zone"]
month = reading_ts.month
matching = [o for o in outage_event_records if o["grid_zone"] == zone and o["start_time"].month == month]
outage_event_id = random.choice(matching)["outage_id"] if matching else None
```

Write `{CATALOG_SCHEMA}.outage_events`.

### Cell 5 additions

- PK: `outage_id`
- FK on usage_records: `outage_event_id -> outage_events(outage_id)` (add `outage_event_id` column)

---

## TODO 7: Add `billing_periods`/`invoices` table

Monthly bills, payment status, and arrears for revenue and collections analytics.

### Implementation (cell 4)

After generating usage records, aggregate into monthly invoices per customer:

```python
from collections import defaultdict

monthly_charges = defaultdict(lambda: defaultdict(float))  # {customer_id: {(year,month): total}}
for rec in records:
    meter = meter_lookup[rec["meter_id"]]
    cust_id = meter["customer_id"]
    ym = (rec["reading_date"].year, rec["reading_date"].month)
    monthly_charges[cust_id][ym] += rec["cost"]

payment_statuses = ["Paid", "Overdue", "Partial", "Pending"]
payment_weights = [70, 10, 8, 12]

invoices = []
inv_id = 0
for cust_id, months in monthly_charges.items():
    arrears = 0.0
    for (year, month), total in sorted(months.items()):
        inv_id += 1
        status = random.choices(payment_statuses, weights=payment_weights)[0]
        if status == "Overdue":
            arrears += total
            payment_date = None
        elif status == "Partial":
            paid = round(total * random.uniform(0.3, 0.8), 2)
            arrears += (total - paid)
            payment_date = date(year, month, 1) + timedelta(days=random.randint(15, 45))
        else:
            payment_date = date(year, month, 1) + timedelta(days=random.randint(10, 35))
        invoices.append({
            "invoice_id": inv_id,
            "customer_id": cust_id,
            "billing_period_start": date(year, month, 1),
            "billing_period_end": date(year, month, 28),  # simplified
            "total_charges": round(total, 2),
            "payment_status": status,
            "payment_date": payment_date,
            "arrears_amount": round(arrears, 2),
        })
```

Write `{CATALOG_SCHEMA}.billing_invoices`.

### Cell 5 additions

- PK: `invoice_id`
- FK: `customer_id -> customers(customer_id)`

---

## TODO 8: Add `weather_observations` table

Station-level hourly weather data to replace per-reading synthetic temperature.

### Implementation (cell 4)

Generate weather stations and hourly observations:

```python
weather_stations = []
station_id = 0
for region in regions:
    station_id += 1
    weather_stations.append({"station_id": station_id, "region": region, "station_name": f"{region} WX-{station_id}"})

region_station = {ws["region"]: ws["station_id"] for ws in weather_stations}

weather_obs = []
obs_id = 0
for station in weather_stations:
    region = station["region"]
    for day_offset in range(730):
        obs_date = date(2024, 1, 1) + timedelta(days=day_offset)
        month = obs_date.month
        base_temp = temp_baselines[region][month]  # reuse existing temp_baselines
        for hour in range(0, 24, 3):  # every 3 hours to limit row count
            obs_id += 1
            temp = round(base_temp + random.gauss(0, 5) + (5 if 12 <= hour <= 16 else -3), 1)
            weather_obs.append({
                "observation_id": obs_id,
                "station_id": station["station_id"],
                "observation_time": datetime(obs_date.year, obs_date.month, obs_date.day, hour),
                "temperature_f": temp,
                "humidity_pct": round(random.uniform(20, 95), 1),
                "wind_speed_mph": round(random.expovariate(1 / 8), 1),
                "precipitation_in": round(max(0, random.gauss(0, 0.1)), 2),
            })
```

In the usage records loop, replace the per-reading synthetic temperature with a lookup:

```python
station = region_station[meter["region"]]
# Find nearest weather observation
obs_hour = (reading_ts.hour // 3) * 3
# Use the weather table as the source of truth for temperature
```

Write `{CATALOG_SCHEMA}.weather_observations`. Expect ~730 days x 8 obs/day x N stations rows.

### Cell 5 additions

- PK: `observation_id`
- FK: `station_id` (no formal FK target unless weather_stations is also a table)

### Cell 6 update

Add all three new tables to schema comment with row counts, PKs, and FKs.
