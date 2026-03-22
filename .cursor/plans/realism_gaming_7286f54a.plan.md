---
name: Realism Gaming
overview: "Implement Gaming-specific realism improvements: correlate VIP tier with spend, align preferred_game, fix voided wager totals, chronological RG flags, and add promotions/bonuses table."
todos:
  - id: gam-vip
    content: Correlate VIP tier with actual lifetime wagered using percentile thresholds
    status: pending
  - id: gam-pref-game
    content: Derive preferred_game from actual game_type distribution per player
    status: pending
  - id: gam-void-excl
    content: Exclude voided/cashed-out wagers from player_wager_totals
    status: pending
  - id: gam-rg-chrono
    content: Sort wagers chronologically before computing cumulative loss for RG flags
    status: pending
  - id: gam-promotions
    content: Add promotions/bonuses table with campaign IDs and wagering requirements
    status: pending
isProject: false
---

# Realism Improvements: Gaming

Target file: [Gaming Dataset Generator.ipynb](Gaming Dataset Generator.ipynb)

- **Cell 4** (index 4): Data generation — `players` entity loop, `wager_records` event loop, reconciliation into `reconciled_players`
- **Cell 5** (index 5): Column comments and PK/FK constraints
- **Cell 6** (index 6): `%sql COMMENT ON SCHEMA` description

Key variables: `CATALOG_SCHEMA`, `NUM_PLAYERS`, `NUM_EVENT_RECORDS`, `players`, `records` (wagers), `player_wager_totals` (dict keyed by player_id -> `[wagered, payout]`), `game_catalog`, `game_types`, `vip_tiers`, `reconciled_players`, `NOW = datetime(2026, 3, 21)`

---

## TODO 1: Correlate VIP tier with actual spend

Currently `vip_tier` is derived from an independent `activity_score` random draw during entity generation. It should reflect actual wagering volume.

### Implementation (cell 4)

Move VIP assignment from the players entity loop to the reconciliation phase (after all wagers are generated and `player_wager_totals` is populated):

```python
# After wager generation, compute VIP from actual lifetime wagered
all_wagered = sorted([player_wager_totals.get(pid, [0,0])[0] for pid in range(1, NUM_PLAYERS+1)])
thresholds = {
    "Diamond": np.percentile(all_wagered, 99),
    "Platinum": np.percentile(all_wagered, 95),
    "Gold": np.percentile(all_wagered, 85),
    "Silver": np.percentile(all_wagered, 70),
}

def derive_vip(total_wagered):
    if total_wagered >= thresholds["Diamond"]: return "Diamond"
    if total_wagered >= thresholds["Platinum"]: return "Platinum"
    if total_wagered >= thresholds["Gold"]: return "Gold"
    if total_wagered >= thresholds["Silver"]: return "Silver"
    return "Bronze"
```

In the `reconciled_players` loop, replace the existing `vip_tier` assignment with `derive_vip(player_wager_totals[pid][0])`.

Remove the `activity_score`-based VIP logic from the initial `players` entity loop (keep a placeholder tier that gets overwritten during reconciliation).

### Verification

- Cross-tab VIP tier vs lifetime_wagered quartiles; Diamond players should be in the top percentile.

---

## TODO 2: Align preferred_game with actual wager distribution

Currently `preferred_game` is randomly assigned from `preferred_games` list independent of actual play.

### Implementation (cell 4)

After wager generation, compute each player's most-played game type:

```python
from collections import Counter

player_game_counts = defaultdict(Counter)
for rec in records:
    player_game_counts[rec["player_id"]][rec["game_type"]] += 1

def get_preferred_game(pid):
    counts = player_game_counts.get(pid)
    if not counts:
        return random.choice(preferred_games)
    top_type = counts.most_common(1)[0][0]
    type_games = game_catalog[top_type]["games"]
    return random.choice(type_games)
```

In the `reconciled_players` loop, replace the existing `preferred_game` value with `get_preferred_game(pid)`.

---

## TODO 3: Exclude voided/cashed-out wagers from financial totals

Currently `player_wager_totals` includes all wagers regardless of status.

### Implementation (cell 4)

In the wager loop where `player_wager_totals` is accumulated (approx. after wager status determination), add a guard:

```python
# Only count settled wagers in financial totals
if wager_status not in ("Voided", "Cashed Out"):
    player_wager_totals[pid][0] += wager_amount
    player_wager_totals[pid][1] += payout
```

Move the existing unconditional accumulation inside this guard.

### Verification

- Sum of `lifetime_wagered` across all players should be less than before (voided/cashed-out excluded).
- Cross-check: count voided wagers and verify their amounts are excluded.

---

## TODO 4: Chronological cumulative loss for RG flags

Currently responsible gaming flags use insertion-order running totals, not time-ordered.

### Implementation (cell 4)

After all wagers are generated, sort by timestamp before computing RG flags:

```python
# Sort wagers chronologically per player
from collections import defaultdict
player_wagers = defaultdict(list)
for rec in records:
    player_wagers[rec["player_id"]].append(rec)

for pid, wagers in player_wagers.items():
    wagers.sort(key=lambda w: w["wager_date"])
    cumulative_loss = 0
    for w in wagers:
        cumulative_loss += w["wager_amount"] - w["payout"]
        # Apply RG flag logic based on chronological cumulative_loss
        w["rg_flag"] = cumulative_loss > RG_THRESHOLD or w["wager_amount"] > SINGLE_BET_THRESHOLD
```

This replaces the current in-loop RG flag computation. The `RG_THRESHOLD` and `SINGLE_BET_THRESHOLD` values should match the existing thresholds in the code.

---

## TODO 5: Add `promotions`/`bonuses` table

Campaign-level promotion data with wagering requirements.

### Implementation (cell 4)

Generate promotions before the wager loop:

```python
bonus_types = ["Deposit Match", "Free Bet", "Cashback", "Loyalty Reward", "Refer-a-Friend"]
campaign_names = ["Welcome Bonus", "Weekend Special", "High Roller", "VIP Exclusive",
                  "Holiday Promo", "March Madness", "Summer Slam", "Anniversary"]

NUM_PROMOTIONS = 50
promotions = []
for i in range(1, NUM_PROMOTIONS + 1):
    start = date(2024, 1, 1) + timedelta(days=random.randint(0, 600))
    duration = random.randint(3, 30)
    promotions.append({
        "promo_id": i,
        "campaign_name": f"{random.choice(campaign_names)} {i}",
        "bonus_type": random.choice(bonus_types),
        "wagering_requirement": random.choice([1, 5, 10, 15, 20, 25, 30]),
        "min_deposit": random.choice([10, 20, 25, 50, 100]),
        "max_bonus_amount": round(random.uniform(10, 500), 2),
        "start_date": start,
        "end_date": start + timedelta(days=duration),
    })
```

In the wager loop, assign ~15% of wagers a `promo_id` from active promotions (where `wager_date` falls within `start_date`/`end_date`):

```python
active_promos = [p for p in promotions if p["start_date"] <= wager_date <= p["end_date"]]
promo_id = random.choice(active_promos)["promo_id"] if active_promos and random.random() < 0.15 else None
```

Write `{CATALOG_SCHEMA}.promotions`.

### Cell 5 additions

- `apply_comments` for all 7 promotion columns
- PK: `promo_id`
- FK on wager_records: `promo_id -> promotions(promo_id)` (add `promo_id` column to wager_records)

### Cell 6 update

Add: `promotions` (~50 rows, PK: promo_id). Update wager_records description to include FK: promo_id -> promotions.
