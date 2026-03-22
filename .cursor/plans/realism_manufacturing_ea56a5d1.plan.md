---
name: Realism Manufacturing
overview: "Implement Manufacturing-specific realism improvements: correlate defects with equipment condition, enforce temporal consistency, anchor maintenance dates, type-specific capacity, add quality_inspections, shift_calendar, and inventory tables."
todos:
  - id: mfg-defect-corr
    content: Correlate defect rate with equipment age, efficiency, and maintenance state
    status: pending
  - id: mfg-temporal
    content: Enforce order start_date >= equipment install_date
    status: pending
  - id: mfg-maint-anchor
    content: Anchor next_maintenance_date to always be in the future for operational equipment
    status: pending
  - id: mfg-capacity
    content: Replace single 450 min/day constant with equipment-type-specific capacity dict
    status: pending
  - id: mfg-inspections
    content: Add quality_inspections table with sample-based results per order
    status: pending
  - id: mfg-shifts
    content: Add shift_calendar table with plant/date/shift/crew/hours
    status: pending
  - id: mfg-inventory
    content: Add inventory/wip_snapshots table with weekly product-level stock levels
    status: pending
isProject: false
---

# Realism Improvements: Manufacturing

Target file: [Manufacturing Dataset Generator.ipynb](Manufacturing Dataset Generator.ipynb)

- **Cell 4** (index 4): Data generation — `equipment` entity loop, `production_orders` event loop
- **Cell 5** (index 5): Column comments and PK/FK constraints
- **Cell 6** (index 6): `%sql COMMENT ON SCHEMA` description

Key variables: `CATALOG_SCHEMA`, `NUM_EQUIPMENT`, `NUM_EVENT_RECORDS`, `equip_lookup`, `equip_types`, `manufacturers`, `locations`, `annual_hours_by_type`, `energy_per_hour`, `product_names`, `equip_product_affinity`, `order_statuses`, `priorities`, `defect_types_list`, `product_defect_weights`, `product_unit_cost`, `NOW = datetime(2026, 3, 21)`

Current capacity constant: `450` min/day used across all equipment types.

---

## TODO 1: Correlate defect rate with equipment age, efficiency, and maintenance state

Currently `defect_quantity` is drawn independently of equipment condition.

### Implementation (cell 4)

In the production orders loop, after selecting `eid` and retrieving `equip_info`, compute a condition-based defect multiplier:

```python
age_years = equip_info["age_years"]
efficiency = equip_info["efficiency_rating"]
last_maint = equip_info.get("last_maintenance_date")
days_since_maint = (start_dt - last_maint).days if last_maint else 999

# Condition-based defect multiplier
age_factor = 1.0 + max(0, age_years - 5) * 0.08       # +8% per year beyond 5
efficiency_factor = max(0.5, 2.0 - efficiency / 50.0)   # lower efficiency -> more defects
maint_factor = 1.0 + max(0, days_since_maint - 90) * 0.005  # +0.5% per day overdue
defect_multiplier = age_factor * efficiency_factor * maint_factor
```

Apply the multiplier to the existing defect probability or quantity:

```python
base_defect_rate = product_defect_weights.get(product, 0.02)
adjusted_rate = min(base_defect_rate * defect_multiplier, 0.25)  # cap at 25%
defect_quantity = int(actual_quantity * adjusted_rate)
```

### Verification

- Older, less efficient equipment with overdue maintenance should produce more defects.
- Plot defect rate vs equipment age to confirm positive correlation.

---

## TODO 2: Enforce temporal consistency — order start_date after equipment install_date

Currently orders can reference equipment whose `install_date` is after the order's `start_date`.

### Implementation (cell 4)

In the production orders loop, after selecting `eid` and retrieving `equip_info`:

```python
equip_install = equip_info["install_date"]
order_window_start = date(2024, 1, 1)

# Ensure order starts after equipment was installed
earliest_start = max(order_window_start, equip_install)
latest_start = date(2026, 1, 1)

if earliest_start >= latest_start:
    continue  # skip — equipment installed too recently for this window

start_dt = earliest_start + timedelta(days=random.randint(0, (latest_start - earliest_start).days))
```

This replaces the current `start_dt = date(2024,1,1) + timedelta(days=random.randint(0,730))`.

### Verification

- Assert `start_dt >= equip_info["install_date"]` for every order.

---

## TODO 3: Anchor `next_maintenance_date` relative to NOW

Currently `next_maintenance_date` can be in the past for operational equipment.

### Implementation (cell 4)

In the equipment entity loop, after setting status and maintenance fields:

```python
if status == "Operational":
    maint_interval = random.randint(30, 180)  # days between maintenance
    next_maintenance_date = NOW.date() + timedelta(days=random.randint(1, maint_interval))
elif status == "Under Maintenance":
    next_maintenance_date = NOW.date() + timedelta(days=random.randint(1, 7))
else:
    next_maintenance_date = None  # Decommissioned/Idle — no scheduled maintenance
```

This replaces any existing `next_maintenance_date` logic that generates arbitrary dates.

### Verification

- All operational equipment should have `next_maintenance_date > NOW`.
- Decommissioned equipment should have `next_maintenance_date = None`.

---

## TODO 4: Use equipment-type-specific capacity instead of single 450 min/day

Currently all equipment shares one `450` min/day production capacity constant.

### Implementation (cell 4)

Add a capacity dict near the top of cell 4 (alongside `equip_types`):

```python
capacity_minutes_per_day = {
    "CNC Mill": 420,
    "Assembly Robot": 480,
    "Injection Molder": 450,
    "Laser Cutter": 400,
    "3D Printer": 360,
    "Welding Station": 440,
    "Paint Booth": 390,
    "Press Brake": 430,
    "Conveyor System": 500,
    "Packaging Machine": 470,
}
```

In the production orders loop, replace the hardcoded `450`:

```python
# Old: production_days = max(1, int(math.ceil(planned_qty * cycle_time / 450)))
# New:
daily_capacity = capacity_minutes_per_day.get(equip_info["equipment_type"], 450)
production_days = max(1, int(math.ceil(planned_qty * cycle_time / daily_capacity)))
```

Ensure the keys in `capacity_minutes_per_day` match the values in `equip_types`.

### Verification

- Equipment types with lower capacity (e.g., 3D Printer at 360) should have longer production durations for the same order quantity.

---

## TODO 5: Add `quality_inspections` table

Sample-based inspection results linked to production orders.

### Implementation (cell 4)

After the production orders loop, generate inspections:

```python
inspectors = [fake.name() for _ in range(15)]
root_causes = ["Material Defect", "Machine Calibration", "Operator Error", "Design Flaw",
               "Environmental", "Tool Wear", "Process Drift", None]

inspections = []
insp_id = 0
for order in orders:
    if order["order_status"] in ("Completed", "In Progress") and random.random() < 0.6:
        insp_id += 1
        sample_size = random.randint(5, max(5, order["actual_quantity"] // 10))
        defects_found = int(sample_size * random.uniform(0, order.get("defect_rate", 0.05)))
        pass_fail = "Pass" if defects_found <= sample_size * 0.02 else "Fail"
        inspections.append({
            "inspection_id": insp_id,
            "order_id": order["order_id"],
            "equipment_id": order["equipment_id"],
            "inspector_name": random.choice(inspectors),
            "inspection_date": order["end_date"] + timedelta(days=random.randint(0, 2)) if order["end_date"] else order["start_date"],
            "sample_size": sample_size,
            "defects_found": defects_found,
            "pass_fail": pass_fail,
            "root_cause": random.choice(root_causes) if pass_fail == "Fail" else None,
        })
```

Write as `{CATALOG_SCHEMA}.quality_inspections`.

### Cell 5 additions

- PK: `inspection_id`
- FK: `order_id -> production_orders(order_id)`, `equipment_id -> equipment(equipment_id)`

---

## TODO 6: Add `shift_calendar` table

Plant shift records for capacity utilization analytics.

### Implementation (cell 4)

Generate shift records covering the 2-year event window:

```python
shift_names = ["Day", "Night", "Swing"]
shift_hours = {"Day": 8, "Night": 8, "Swing": 8}
crews = [f"Crew-{c}" for c in "ABCDEF"]

shift_calendar = []
shift_id = 0
for loc in locations:
    for day_offset in range(730):
        shift_date = date(2024, 1, 1) + timedelta(days=day_offset)
        is_weekend = shift_date.weekday() >= 5
        active_shifts = ["Day"] if is_weekend else shift_names
        for shift_name in active_shifts:
            shift_id += 1
            planned = shift_hours[shift_name]
            actual = round(planned * random.uniform(0.85, 1.05), 1)
            shift_calendar.append({
                "shift_id": shift_id,
                "plant_location": loc,
                "shift_date": shift_date,
                "shift_name": shift_name,
                "crew_id": random.choice(crews),
                "planned_hours": planned,
                "actual_hours": actual,
            })
```

Write as `{CATALOG_SCHEMA}.shift_calendar`. Expect ~730 days x N locations x ~2.4 shifts/day rows.

### Cell 5 additions

- PK: `shift_id`
- No FK (reference/dimension table)

---

## TODO 7: Add `inventory`/`wip_snapshots` table

Periodic raw material and work-in-progress snapshots for throughput modeling.

### Implementation (cell 4)

Generate weekly snapshots:

```python
inventory_snapshots = []
snap_id = 0
for product in product_names:
    base_raw = random.randint(500, 5000)
    base_wip = random.randint(100, 1000)
    base_fg = random.randint(200, 2000)
    reorder_point = int(base_raw * 0.3)
    for week in range(104):  # 2 years of weekly snapshots
        snap_id += 1
        snap_date = date(2024, 1, 1) + timedelta(weeks=week)
        drift = random.gauss(0, 0.1)
        inventory_snapshots.append({
            "snapshot_id": snap_id,
            "product_name": product,
            "snapshot_date": snap_date,
            "raw_material_qty": max(0, int(base_raw * (1 + drift))),
            "wip_qty": max(0, int(base_wip * (1 + drift * 1.5))),
            "finished_goods_qty": max(0, int(base_fg * (1 - drift))),
            "reorder_point": reorder_point,
        })
```

Write as `{CATALOG_SCHEMA}.inventory_snapshots`.

### Cell 5 additions

- PK: `snapshot_id`
- No FK (aggregated reference data)
