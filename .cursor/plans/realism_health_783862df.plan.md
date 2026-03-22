---
name: Realism Health
overview: "Implement Health-specific realism improvements: extend visit hours, add CPT codes, model deductible/copay/OOP, add prescriptions table, add payers/plans dimension, and link follow-up visits."
todos:
  - id: hlt-hours
    content: Extend visit hours past 19:00 for ER (0-23) and urgent care (7-21)
    status: pending
  - id: hlt-cpt
    content: Add CPT/HCPCS procedure codes mapped to diagnosis categories
    status: pending
  - id: hlt-deductible
    content: Model deductible, copay, coinsurance, and annual OOP max per insurance type
    status: pending
  - id: hlt-prescriptions
    content: Add prescriptions table with dose, frequency, days supply, refill count
    status: pending
  - id: hlt-payers
    content: Add payers/plans dimension table and link patients and visits to plan_id
    status: pending
  - id: hlt-followup
    content: Link follow-up visits to prior encounters via prior_visit_id
    status: pending
isProject: false
---

# Realism Improvements: Health

Target file: [Health Dataset Generator.ipynb](Health Dataset Generator.ipynb)

- **Cell 4** (index 4): Data generation — `patients` entity loop, `medical_records` event loop
- **Cell 5** (index 5): Column comments and PK/FK constraints
- **Cell 6** (index 6): `%sql COMMENT ON SCHEMA` description

Key variables: `CATALOG_SCHEMA`, `NUM_PATIENTS`, `NUM_EVENT_RECORDS`, `patient_lookup`, `patient_pool` (weighted), `visit_types`, `diag_dept_med`, `insurance_types`, `physicians`, `records`, `NOW = date(2026, 3, 21)`

---

## TODO 1: Extend visit hours past 19:00 for night ER and urgent care

Currently all visits use `random.randint(7, 19)` for the hour component.

### Implementation (cell 4)

Replace the single hour assignment with visit-type-conditional logic in the medical records loop:

```python
if visit_type == "Emergency":
    hour = random.choices(range(24), weights=[
        1.5,1.2,1.0,0.8,0.8,1.0,1.2,1.5,2.0,2.0,2.0,2.0,
        2.0,2.0,2.0,2.0,2.5,2.5,3.0,3.0,2.5,2.5,2.0,1.8
    ])[0]
elif visit_type == "Urgent Care":
    hour = random.randint(7, 21)
else:
    hour = random.randint(7, 19)
```

This replaces the current `hours=random.randint(7, 19)` in the `visit_ts` assignment.

### Verification

- Filter ER visits and confirm hours span 0-23 with higher evening weight.
- Filter routine visits and confirm hours remain 7-19.

---

## TODO 2: Add CPT/HCPCS procedure codes alongside ICD-10 diagnoses

Currently visits only have ICD-10 diagnosis codes with no procedure coding.

### Implementation (cell 4)

Add a `procedure_codes` mapping near the existing `diag_dept_med` dict:

```python
category_procedures = {
    "Cardiovascular": [("99213","Office visit, est. patient"), ("93000","ECG"), ("93306","Echocardiogram")],
    "Respiratory": [("99213","Office visit, est. patient"), ("94010","Spirometry"), ("71046","Chest X-ray")],
    "Musculoskeletal": [("99213","Office visit, est. patient"), ("97110","Therapeutic exercises"), ("73030","Shoulder X-ray")],
    "Neurological": [("99214","Office visit, detailed"), ("95819","EEG"), ("70553","Brain MRI")],
    "Gastrointestinal": [("99213","Office visit, est. patient"), ("43239","Upper endoscopy"), ("74177","CT abdomen")],
    "Endocrine": [("99214","Office visit, detailed"), ("80061","Lipid panel"), ("83036","HbA1c")],
    "Mental Health": [("90834","Psychotherapy 45min"), ("90837","Psychotherapy 60min"), ("96127","Brief emotional assessment")],
    "Dermatological": [("99213","Office visit, est. patient"), ("11102","Skin biopsy"), ("17000","Cryotherapy")],
    "Renal/Urological": [("99214","Office visit, detailed"), ("81001","Urinalysis"), ("74178","CT abdomen/pelvis")],
    "General/Preventive": [("99395","Preventive visit 18-39"), ("99396","Preventive visit 40-64"), ("36415","Venipuncture")],
}
```

In the visit loop, after determining the diagnosis category, select 1-2 procedure codes:

```python
procedures = category_procedures.get(diag_category, category_procedures["General/Preventive"])
selected = random.sample(procedures, k=min(random.randint(1,2), len(procedures)))
procedure_code = selected[0][0]
procedure_description = selected[0][1]
```

Add `procedure_code` and `procedure_description` columns to each record.

### Cell 5/6

- Add comments for the 2 new columns in `apply_comments`
- Update schema comment to mention procedure codes

---

## TODO 3: Model deductible, copay, and annual out-of-pocket maximum

Replace `out_of_pocket = cost * (1 - coverage_pct)` with realistic insurance math.

### Implementation (cell 4)

Define per-insurance-type parameters near the top:

```python
insurance_params = {
    "Private":   {"deductible": 1500, "copay": 30, "oop_max": 8000, "coinsurance": 0.20},
    "Medicare":  {"deductible": 240,  "copay": 20, "oop_max": 7550, "coinsurance": 0.20},
    "Medicaid":  {"deductible": 0,    "copay": 5,  "oop_max": 2000, "coinsurance": 0.05},
    "VA":        {"deductible": 0,    "copay": 15, "oop_max": 3000, "coinsurance": 0.10},
    "Uninsured": {"deductible": 0,    "copay": 0,  "oop_max": 999999, "coinsurance": 1.00},
}
```

Track per-patient annual accumulator:

```python
patient_annual_spend = defaultdict(lambda: defaultdict(float))  # {pid: {year: accumulated}}
```

In the visit loop, replace the current cost calculation:

```python
params = insurance_params[patient["insurance_type"]]
year = visit_date.year
accumulated = patient_annual_spend[pid][year]
remaining_deductible = max(0, params["deductible"] - accumulated)
after_deductible = max(0, visit_cost - remaining_deductible)
coinsurance_share = after_deductible * params["coinsurance"]
patient_cost = remaining_deductible + params["copay"] + coinsurance_share
oop_remaining = max(0, params["oop_max"] - accumulated)
out_of_pocket = min(patient_cost, oop_remaining)
patient_annual_spend[pid][year] += out_of_pocket
```

### Verification

- No patient's annual out-of-pocket should exceed their insurance type's `oop_max`.
- Uninsured patients should pay full cost.

---

## TODO 4: Add `prescriptions` table

Medications with dose, frequency, days supply, and refill count — currently a single nullable string per visit.

### Implementation (cell 4)

After the medical records loop, generate prescriptions for visits that have a medication:

```python
medications_detail = {
    "Lisinopril": {"dose": "10mg", "frequency": "Once daily", "days_supply": 30, "refills": 3},
    "Metformin": {"dose": "500mg", "frequency": "Twice daily", "days_supply": 30, "refills": 5},
    "Atorvastatin": {"dose": "20mg", "frequency": "Once daily", "days_supply": 90, "refills": 3},
    # ... extend for all medications in diag_dept_med
}

prescriptions = []
rx_id = 0
for rec in records:
    if rec.get("medication"):
        rx_id += 1
        med = rec["medication"]
        detail = medications_detail.get(med, {"dose": "10mg", "frequency": "Once daily", "days_supply": 30, "refills": 2})
        prescriptions.append({
            "prescription_id": rx_id,
            "visit_id": rec["visit_id"],
            "patient_id": rec["patient_id"],
            "medication_name": med,
            "dosage": detail["dose"],
            "frequency": detail["frequency"],
            "days_supply": detail["days_supply"],
            "refill_count": random.randint(0, detail["refills"]),
            "prescribed_date": rec["visit_date"],
        })
```

Write as `{CATALOG_SCHEMA}.prescriptions`.

### Cell 5 additions

- `apply_comments` for all 9 columns
- PK: `prescription_id`
- FK: `visit_id -> medical_records(visit_id)`, `patient_id -> patients(patient_id)`

---

## TODO 5: Add `payers`/`plans` dimension

Plan ID, network, deductible, effective dates — replaces flat `insurance_type` string.

### Implementation (cell 4)

Generate payers and plans before the patients loop:

```python
payer_names = {
    "Private": ["Aetna", "Blue Cross", "Cigna", "UnitedHealth", "Humana"],
    "Medicare": ["Medicare Part A", "Medicare Part B", "Medicare Advantage"],
    "Medicaid": ["State Medicaid"],
    "VA": ["VA Health"],
}
network_types = ["PPO", "HMO", "EPO", "POS"]

plans = []
plan_id = 0
for ins_type, payers in payer_names.items():
    for payer in payers:
        for network in random.sample(network_types, k=random.randint(1,3)):
            plan_id += 1
            params = insurance_params[ins_type]
            plans.append({
                "plan_id": plan_id, "payer_name": payer,
                "plan_name": f"{payer} {network}",
                "insurance_type": ins_type, "network_type": network,
                "deductible": params["deductible"],
                "oop_max": params["oop_max"],
                "effective_date": date(2023, 1, 1),
                "termination_date": None,
            })
```

In the patients loop, assign each patient a `plan_id` from plans matching their `insurance_type`. Add `plan_id` column to `medical_records` as well.

Write as `{CATALOG_SCHEMA}.payers_plans`.

### Cell 5 additions

- PK: `plan_id`
- FK on patients: `plan_id -> payers_plans(plan_id)`
- FK on medical_records: `plan_id -> payers_plans(plan_id)`

---

## TODO 6: Link follow-up visits to prior encounters

When `follow_up_required=True`, generate a linked follow-up visit.

### Implementation (cell 4)

Add a `prior_visit_id` column (default `None`) to all records. After the main medical records loop, generate follow-ups:

```python
follow_ups = []
follow_up_id = len(records)
for rec in records:
    if rec.get("follow_up_required") and random.random() < 0.7:  # 70% compliance
        follow_up_id += 1
        follow_date = rec["visit_date"] + timedelta(days=random.randint(7, 30))
        follow_ups.append({
            "visit_id": follow_up_id,
            "patient_id": rec["patient_id"],
            "visit_type": "Follow-up",
            "visit_date": follow_date,
            "prior_visit_id": rec["visit_id"],
            # ... copy relevant fields, lighter cost
        })
records.extend(follow_ups)
```

Add `prior_visit_id` column to the DataFrame schema. Self-referencing FK: `prior_visit_id -> medical_records(visit_id)`.

### Verification

- Count follow-up visits; should be ~70% of records where `follow_up_required=True`.
- All `prior_visit_id` values should reference valid `visit_id` values.
