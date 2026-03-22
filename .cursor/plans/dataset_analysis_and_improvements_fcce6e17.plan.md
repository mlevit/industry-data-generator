---
name: Dataset analysis and improvements
overview: A comprehensive analysis of all 8 industry datasets covering limitations, data quality issues, and high-value tables that could be added to each schema.
todos: []
isProject: false
---

# Industry Dataset Analysis: Limitations and Recommended Additions

## Cross-Cutting Limitations (all 8 notebooks)

These issues apply to every notebook and should be addressed systematically:

- **No time-series realism** -- all event timestamps are i.i.d. uniform over a 2-year window. No seasonality, weekly patterns, business hours, or event clustering.
- **Uniform entity sampling** -- event rows pick entity IDs uniformly (or near-uniform), so every account/customer/player gets roughly equal activity. Real data has heavy-tail (Pareto) distributions.
- **No temporal consistency** -- events can occur before the entity existed (e.g., transactions before account opened, wagers before player registered, orders before customer signup, call records after churn date).
- **Denormalized dimensions** -- merchants, products, physicians, agents, towers, and operators exist only as free-text strings on fact rows, not as proper dimension tables with FKs.
- **No slowly changing dimensions** -- entity attributes are static snapshots. No history of plan changes, status transitions, address moves, or tier upgrades.

---

## 1. Finance

**Current tables:** `accounts` (~2K), `transactions` (100K-500K)

### Limitations

- **No running balance / ledger** -- `accounts.balance` is static; transactions do not update or reconcile it.
- **No customer entity** -- one name per account; no joint accounts, household, or customer-level aggregation.
- **Fraud flag oversells** -- cell 5 comments mention "velocity" checks, but code only uses amount and channel thresholds. No sliding-window or sequence-based detection.
- **No FX rates** -- 7 currencies present but no exchange rate table, so cross-currency analytics are impossible.
- **Fraud probability can exceed 1.0** -- stacking bonuses for large Wire Purchases can push `fraud_prob` above 1 (functionally OK but semantically wrong).

### Recommended New Tables

- `**customers` -- separate from accounts; enables 1:N account ownership, household grouping, and customer-level analytics (CLV, churn).
- `**merchants` -- dimension with MCC code, category, location; enables merchant-level fraud and spend analytics.
- `**daily_account_balances` -- time-series snapshot; critical for liquidity, overdraft, and trend analysis.
- `**fraud_cases` -- investigation outcomes, resolution, chargeback amounts; enables supervised ML training instead of just a boolean flag.

---

## 2. Gaming

**Current tables:** `players` (~5K), `wager_records` (100K-500K)

### Limitations

- **Wagers before registration** -- `wager_date` starts 2024-01-01 but `registration_date` can be as late as mid-2025 (2020-01-01 + 2000 days). Many players have wagers dated before they signed up.
- **Self-excluded players still wager** -- no filter prevents wager generation after `self_exclusion_date`.
- **VIP tier uncorrelated with spend** -- `vip_tier` comes from an independent `activity_score`, not from actual `lifetime_deposits` or wager volume.
- `**session_duration_minutes` per wager, not per session -- value is redrawn for every wager row, but `session_id` groups multiple wagers. Real sessions have one duration.
- `**preferred_game` independent of actual play -- a player's preferred game has no relationship to their `game_type` distribution on wagers.
- **Voided wagers still count in financial totals** -- `player_wager_totals` includes voided/cashed-out bets.
- **RG cumulative loss uses insertion order, not time order** -- flags are based on running totals during generation, not chronological wager sequence.

### Recommended New Tables

- `**payment_transactions` -- deposits, withdrawals, chargebacks, payment methods; currently only lifetime aggregates exist on the player row.
- `**sessions` -- one row per `session_id` with start/end time, actual duration, device, game mix; fixes the duration-per-wager problem.
- `**games` -- dimension with RTP, variance class, supplier, regulatory classification; enables game-level analytics.
- `**promotions` / `bonuses` -- campaign ID, terms, wagering requirements; currently `bonus_type` is a random nullable string with no campaign context.
- `**rg_interventions` -- actual actions taken (limit changes, cooling-off, exclusions) linked to trigger events; much richer than a boolean flag.

---

## 3. Health

**Current tables:** `patients` (~5K), `medical_records` (100K-500K)

### Limitations

- **No provider dimension** -- 50 physicians are generated as Faker strings with no specialty, NPI, or facility affiliation. Visit physician is independent of `primary_physician`.
- **Diagnosis loosely tied to conditions** -- ICD codes are drawn from weighted categories, not from the patient's specific chronic conditions. A diabetic patient could get a respiratory diagnosis with no diabetes code.
- **No payer/plan detail** -- `insurance_type` is a label (Private/Medicare/etc.) with no plan ID, network, deductible, or OOP maximum. `insurance_coverage_pct` is random Beta, unrelated to insurance type.
- **Visit hours only 07:00-19:00** -- underrepresents night ER and urgent care.
- **No procedures, lab results, or vitals at visit** -- `lab_work_ordered` is boolean only; no CPT codes, test names, or result values.
- **Cost model is simplistic** -- `out_of_pocket = cost * (1 - coverage%)` with no deductible, copay, or annual OOP max logic.
- **No longitudinal episodes** -- no readmission chains, follow-up links, or episode groupers despite `follow_up_required` flag.

### Recommended New Tables

- `**providers` -- NPI-style ID, specialty, facility, region; enables referral network and provider performance analytics.
- `**diagnoses` (bridge table) -- patient_id, ICD code, onset date, resolved date; separates billing diagnosis from problem list.
- `**prescriptions` -- medication, dose, days supply, refill count; currently meds are a single nullable string per visit.
- `**lab_results` -- test name, value, unit, reference range, abnormal flag; enables clinical outcome modeling.
- `**payers` / `plans` -- plan ID, network, deductible, effective dates; enables utilization and cost modeling by plan type.

---

## 4. Insurance

**Current tables:** `policyholders` (~22K policies from 15K people), `claims` (100K-500K)

### Limitations

- **No separate `persons` table** -- demographics are denormalized on every policy row. `policyholder_id` exists but has no FK target.
- `**has_bundled_discount` is cosmetic -- the flag is set for multi-policy holders, but the premium calculation does not actually apply a discount.
- **No claim-to-coverage validation** -- `claim_amount` can exceed `coverage_amount` with no sublimit or cap logic.
- **Non-active policies still generate claims** -- lapsed/cancelled/expired policies appear in the claims pool (low weight). There is no "policy period" or "report date vs occurrence date" distinction.
- `**days_to_resolve` documentation says "business days" but code draws calendar days from a log-normal.
- **Adjuster ID uses `hash()`** -- Python's `hash()` is non-deterministic across processes (unless `PYTHONHASHSEED` is set), making adjuster IDs unstable.
- **Life insurance treated identically to P&C** -- same premium/claim structure; no term vs whole, no mortality tables, no beneficiary.
- **Cell 4 comment says "5K people"** but code uses `NUM_PEOPLE = 15000` -- stale comment.

### Recommended New Tables

- `**persons` -- demographics once, with `policyholder_id` as PK; enables true customer-level analytics across policies.
- `**coverage_items` -- limits, deductibles, and perils per policy (comp vs collision for Auto, dwelling vs personal property for Home); enables coverage adequacy analysis.
- `**claim_payments` -- multiple partial payments, loss adjustment expenses, recovery receipts with dates; currently everything is a single-row snapshot.
- `**agents` -- dimension with office, region, book size; currently free-text on policy rows.
- `**catastrophe_events` -- date, region, type, severity; enables correlated property loss modeling.

---

## 5. Manufacturing

**Current tables:** `equipment` (~3K), `production_orders` (100K-500K)

### Limitations

- **Decommissioned/idle equipment gets orders at the same rate** -- equipment `status` is completely ignored when sampling `equipment_id` for orders.
- **No temporal consistency** -- orders can reference equipment whose `install_date` is after the order's `start_date`.
- **Defect rate independent of equipment condition** -- `defect_quantity` does not correlate with equipment age, efficiency, or maintenance state.
- `**maintenance_count` is not tied to `last_maintenance_date` spacing -- count is derived from age with noise, not from any actual event sequence.
- `**next_maintenance_date` not anchored to NOW -- can be in the past or far future without clear meaning.
- **Single capacity constant (450 min/day)** -- all equipment types and plants share one production capacity assumption.

### Recommended New Tables

- `**dim_product` -- SKU, BOM, category, target cycle time, unit cost; currently products are strings with a separate cost dict in code.
- `**maintenance_events` -- date, type (preventive/corrective), cost, duration, parts; replaces the disconnected `maintenance_count` integer.
- `**quality_inspections` -- sample-based results, inspector, pass/fail, root cause; richer than a single `defect_quantity` integer.
- `**shift_calendar` -- plant, date, shift, crew, planned hours; enables capacity utilization analytics.
- `**inventory` / `wip_snapshots` -- raw material consumption and work-in-progress tracking for throughput models.

---

## 6. Retail

**Current tables:** `customers` (~5K), `orders` (100K-500K)

### Limitations

- **One line per order (no baskets)** -- each order has exactly one product. No multi-item carts, bundle analysis, or market basket analytics.
- **No product dimension table** -- product name and category are duplicated on every order row.
- **No returns pipeline** -- `return_reason` is a string on the order row. No return date, refund amount, partial return, or restocking fee.
- **Geography is inconsistent** -- Boise appears under both "Pacific Northwest" and "Mountain West" in the cities dict.
- `**age` vs `date_of_birth` off by ~1 day -- DOB anchored to `2026-03-20`, age computed from `2026-03-21` (`NOW`).
- **No promotions or coupons** -- only tier-based discount percentages; no campaign IDs, promo codes, or marketing attribution.
- **Returned orders keep full `total_amount`** -- no negative adjustment, credit, or refund modeled.
- **No store or fulfillment dimension** -- `channel` is In-Store/Mobile/Website but no store ID, warehouse, or BOPIS tracking.

### Recommended New Tables

- `**order_lines` -- replaces single-product orders with multi-line baskets; enables basket analysis, cross-sell, and AOV metrics.
- `**dim_product` -- SKU, category, cost, margin, supplier, return policy; normalizes the denormalized product data.
- `**returns` -- return date, refund amount, return reason, condition, restocking; enables return rate analytics and fraud detection.
- `**promotions` / `coupon_redemptions` -- campaign ID, discount type, channel, dates; enables marketing ROI and attribution.
- `**dim_store` -- store ID, location, type, square footage; enables omnichannel and geographic analytics.

---

## 7. Telecom

**Current tables:** `subscribers` (~5K), `call_records` (100K-500K)

### Limitations

- **Churned subscribers still generate call records** -- no filter on `record_date` vs `churn_date`, producing post-churn usage.
- `**monthly_charge` is perfectly deterministic -- each plan has one fixed price with no taxes, promos, equipment installments, or discounts.
- `**call_result = "Completed"` for non-voice records -- SMS, Data, MMS, and Voicemail always show "Completed", which is meaningless for those record types.
- `**data_usage_mb` on non-data records -- SMS/MMS get a tiny exponential noise value (up to 5 MB) instead of 0 or null.
- **No true tower dimension** -- 25 towers per region are generated as random strings with no lat/long, capacity, technology, or sector.
- **Churn model ignores usage and quality** -- churn probability depends only on plan tier and tenure, not on experienced latency, dropped calls, or overage charges.
- **No billing or payment** -- ARPU, collections, payment failures, and revenue recognition are impossible.

### Recommended New Tables

- `**plans` -- dimension with effective dates, features, prices, data caps; enables plan migration and SCD analysis.
- `**towers` / `sites` -- location, technology (3G/4G/5G), capacity, sector; enables network quality and coverage analytics.
- `**billing_invoices` / `payments` -- monthly charges, usage-based fees, payment method, payment status; enables ARPU, churn-from-billing, and collections analytics.
- `**network_incidents` -- outage events, duration, affected towers, root cause; enables service quality correlation with churn.
- `**support_tickets` -- issue type, resolution, CSAT; enables service experience modeling.

---

## 8. Utilities

**Current tables:** `meters` (~5K), `usage_records` (100K-500K)

### Limitations

- **No standalone `customers` table** -- customer data is denormalized on every meter row. `customer_id` exists but is not a PK/FK target.
- `**rate_per_unit` is random, not tied to `rate_plan` -- two meters on the same "Time-of-Use" plan can have wildly different rates per reading. Pricing analytics are meaningless.
- **No time-of-use logic despite TOU plan names** -- `reading_date` has an hour component but usage generation ignores hour-of-day entirely.
- **Outages do not affect usage** -- `had_outage` and `outage_duration_minutes` are independent of `usage_amount`. A 12-hour outage produces the same usage as no outage.
- **Solar cost is always positive** -- `cost = abs(usage) * rate` eliminates the negative usage (export/generation) signal, so net metering credits are not modeled.
- **Interval length is ambiguous** -- electric usage capped at 300 kWh, water at 1500 gallons, but whether these are daily, monthly, or arbitrary intervals is undefined.
- `**NOW = datetime.now()` -- unlike other notebooks which use a fixed date, this makes install-age calculations non-reproducible.
- `**peak_demand_kw` uncorrelated with usage -- drawn from an independent log-normal instead of being derived from interval load.

### Recommended New Tables

- `**customers` -- PK `customer_id`, name, address, premise type, square footage; normalizes the denormalized data on meters.
- `**rate_schedules` / `tariffs` -- rate plan, effective dates, fixed charges, demand charges, TOU periods, tiered rates; makes pricing analytics meaningful.
- `**outage_events` -- start/end timestamps, cause, affected zone, SAIDI/SAIFI metrics; replaces the per-reading boolean with a proper event model.
- `**billing_periods` / `invoices` -- monthly bills, payment status, arrears; enables revenue and collections analytics.
- `**weather_observations` -- station, hourly temperature/humidity/wind; replaces the synthetic per-reading temperature with a proper weather dimension.

---

## Priority Summary: Highest-Impact Additions per Industry

| Industry      | #1 New Table         | #2 New Table           | #1 Fix                                          |
| ------------- | -------------------- | ---------------------- | ----------------------------------------------- |
| Finance       | `customers`          | `merchants`            | Temporal: txn date >= opened_date               |
| Gaming        | `sessions`           | `payment_transactions` | Temporal: wager date >= registration_date       |
| Health        | `providers`          | `lab_results`          | Diagnosis-condition alignment                   |
| Insurance     | `persons`            | `claim_payments`       | Bundled discount in premium calc                |
| Manufacturing | `maintenance_events` | `dim_product`          | Skip decommissioned equipment in order sampling |
| Retail        | `order_lines`        | `dim_product`          | Multi-item baskets                              |
| Telecom       | `billing_invoices`   | `towers`               | No call records after churn_date                |
| Utilities     | `customers`          | `rate_schedules`       | Rate tied to rate_plan, not random              |
