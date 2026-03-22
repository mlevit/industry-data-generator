---
name: Realism Retail
overview: "Implement Retail-specific realism improvements: fix Boise geography, model refund amounts, add promotions/coupon_redemptions table, add dim_store table, and implement customer-product affinity."
todos:
  - id: ret-geo
    content: Fix Boise duplicate — remove from Pacific Northwest, keep in Mountain West
    status: pending
  - id: ret-refund
    content: Model refund_amount and refund_date on returned orders
    status: pending
  - id: ret-promos
    content: Add promotions table with campaign/discount/promo_code and link to orders
    status: pending
  - id: ret-store
    content: Add dim_store table with store_id/type/sqft and link In-Store orders
    status: pending
  - id: ret-affinity
    content: Implement customer-product affinity with 2-3 preferred categories per customer
    status: pending
isProject: false
---

# Realism Improvements: Retail

Target file: [Retail Dataset Generator.ipynb](Retail Dataset Generator.ipynb)

- **Cell 4** (index 4): Data generation — `customers` entity loop, `orders` event loop
- **Cell 5** (index 5): Column comments and PK/FK constraints
- **Cell 6** (index 6): `%sql COMMENT ON SCHEMA` description

Key variables: `CATALOG_SCHEMA`, `NUM_CUSTOMERS`, `NUM_EVENT_RECORDS`, `customer_lookup`, `customer_pool` (weighted), `regions`, `cities` (dict mapping region -> city list), `tiers`, `product_catalog`, `category_price_params`, `channel_payment_weights`, `payment_methods`, `tier_discount_weights`, `tier_activity`, `product_return_rate`, `return_reasons`, `orders`, `NOW = datetime(2026, 3, 21)`

---

## TODO 1: Fix geography — Boise appears under both Pacific Northwest and Mountain West

### Implementation (cell 4)

Locate the `cities` dict near the top of cell 4. It currently has `"Boise"` in both the Pacific Northwest and Mountain West lists. Remove it from Pacific Northwest:

```python
# Find and fix in cities dict:
# "Pacific Northwest": ["Seattle", "Portland", "Boise", "Spokane", ...]
# "Mountain West": ["Denver", "Salt Lake City", "Boise", ...]
#
# Remove "Boise" from Pacific Northwest only:
# "Pacific Northwest": ["Seattle", "Portland", "Spokane", ...]
```

Use `StrReplace` on the notebook cell to remove the duplicate entry.

### Verification

- Grep the cell for "Boise" — should appear exactly once, under Mountain West.

---

## TODO 2: Model refund amounts on returned orders

Currently returned orders keep a positive `total_amount` with no refund modeled.

### Implementation (cell 4)

In the orders loop, after determining `order_status`, add refund logic for returned orders:

```python
refund_amount = None
refund_date = None
if order_status == "Returned":
    is_full_refund = random.random() < 0.7  # 70% full refunds
    if is_full_refund:
        refund_amount = round(total_amount, 2)
    else:
        restocking_pct = random.uniform(0.10, 0.25)
        refund_amount = round(total_amount * (1 - restocking_pct), 2)
    days_to_refund = random.randint(3, 21)
    refund_date = order_date + timedelta(days=random.randint(5, 30) + days_to_refund)
```

Add `refund_amount` and `refund_date` columns to each order record (set to `None` for non-returned orders).

### Cell 5/6

- Add comments for `refund_amount`: "Refund issued for returned orders; partial refunds reflect restocking fees"
- Add comment for `refund_date`: "Date the refund was processed"
- Update schema comment to mention refund modeling

### Verification

- All returned orders should have `refund_amount > 0` and `refund_amount <= total_amount`.
- Non-returned orders should have `refund_amount = None`.

---

## TODO 3: Add `promotions`/`coupon_redemptions` table

Campaign-level promotion data with discount types and marketing attribution.

### Implementation (cell 4)

Generate promotions before the orders loop:

```python
discount_types = ["Percentage", "Fixed Amount", "BOGO", "Free Shipping"]
promo_channels = ["Email", "Social Media", "In-Store", "App Push", "Direct Mail"]

NUM_PROMOTIONS = 40
promotions = []
for i in range(1, NUM_PROMOTIONS + 1):
    start = date(2024, 1, 1) + timedelta(days=random.randint(0, 650))
    duration = random.randint(3, 45)
    disc_type = random.choice(discount_types)
    promotions.append({
        "promo_id": i,
        "campaign_name": f"{random.choice(['Summer','Winter','Spring','Fall','Flash','Loyalty','Clearance','Holiday'])} Sale {i}",
        "discount_type": disc_type,
        "discount_value": random.choice([5,10,15,20,25,30]) if disc_type == "Percentage" else round(random.uniform(5, 50), 2) if disc_type == "Fixed Amount" else 0,
        "promo_code": f"PROMO{i:03d}",
        "start_date": start,
        "end_date": start + timedelta(days=duration),
        "min_order_value": random.choice([0, 25, 50, 75, 100]),
        "channel": random.choice(promo_channels),
    })
```

In the orders loop, assign ~20% of orders a promo_id from active promotions whose `min_order_value` is met:

```python
active_promos = [p for p in promotions
                 if p["start_date"] <= order_date <= p["end_date"]
                 and total_amount >= p["min_order_value"]]
promo_id = random.choice(active_promos)["promo_id"] if active_promos and random.random() < 0.20 else None
```

Write `{CATALOG_SCHEMA}.promotions`.

### Cell 5 additions

- PK: `promo_id`
- FK on orders: `promo_id -> promotions(promo_id)` (add `promo_id` column to orders)

### Cell 6 update

Add: `promotions` (~40 rows, PK: promo_id). Update orders description to include FK: promo_id -> promotions.

---

## TODO 4: Add `dim_store` table

Store dimension for omnichannel and geographic analytics.

### Implementation (cell 4)

Generate stores before the orders loop:

```python
store_types = ["Flagship", "Standard", "Outlet", "Pop-up"]
store_type_weights = [10, 55, 25, 10]

stores = []
store_id = 0
for region, city_list in cities.items():
    for city in city_list:
        num_stores = random.randint(1, 3)  # 1-3 stores per city
        for _ in range(num_stores):
            store_id += 1
            stype = random.choices(store_types, weights=store_type_weights)[0]
            sqft = {
                "Flagship": random.randint(15000, 40000),
                "Standard": random.randint(5000, 15000),
                "Outlet": random.randint(3000, 10000),
                "Pop-up": random.randint(500, 2000),
            }[stype]
            stores.append({
                "store_id": store_id,
                "store_name": f"{city} {stype} #{store_id}",
                "region": region,
                "city": city,
                "store_type": stype,
                "square_footage": sqft,
                "opened_date": date(2015, 1, 1) + timedelta(days=random.randint(0, 3650)),
            })
```

In the orders loop, for In-Store orders, assign a `store_id` from stores matching the customer's region:

```python
region_stores = [s["store_id"] for s in stores if s["region"] == cust["region"]]
store_id = random.choice(region_stores) if channel == "In-Store" and region_stores else None
```

Write `{CATALOG_SCHEMA}.dim_store`.

### Cell 5 additions

- PK: `store_id`
- FK on orders: `store_id -> dim_store(store_id)` (add `store_id` column to orders)

---

## TODO 5: Add customer-product affinity

Repeat purchases should reflect customer preferences rather than uniform random product selection.

### Implementation (cell 4)

After generating customers, assign each customer 2-3 preferred categories:

```python
all_categories = list(product_catalog.keys())

customer_preferences = {}
for cust in customers:
    num_prefs = random.randint(2, 3)
    preferred = random.sample(all_categories, k=num_prefs)
    customer_preferences[cust["customer_id"]] = preferred
```

In the orders loop, when selecting a product category, weight toward the customer's preferences:

```python
prefs = customer_preferences.get(cust_id, [])
# Build weights: preferred categories get 3x weight
cat_weights = []
for cat in all_categories:
    base_weight = 1.0
    if cat in prefs:
        base_weight = 3.0
    # Combine with existing seasonal weights if present
    cat_weights.append(base_weight)

category = random.choices(all_categories, weights=cat_weights)[0]
```

This replaces or augments the existing category selection logic. Merge with any existing seasonal weights (e.g., `holiday_boost`, `fitness_boost`, `beauty_boost`) by multiplying the affinity weight with the seasonal multiplier.

### Verification

- For each customer, their top 2-3 purchased categories should align with their assigned preferences.
- Compute category concentration per customer; should be higher than uniform random.
