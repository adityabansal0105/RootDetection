# 🅿️ Smart Parking Dynamic Pricing System

![Python](https://img.shields.io/badge/Python-3.8%2B-3776AB?style=flat&logo=python&logoColor=white)
![Pandas](https://img.shields.io/badge/Pandas-1.3%2B-150458?style=flat&logo=pandas&logoColor=white)
![NumPy](https://img.shields.io/badge/NumPy-1.21%2B-013243?style=flat&logo=numpy&logoColor=white)
![Bokeh](https://img.shields.io/badge/Bokeh-2.4%2B-grey?style=flat)
![Jupyter](https://img.shields.io/badge/Jupyter-Notebook-F37626?style=flat&logo=jupyter&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green?style=flat)

---

## Executive Summary

A real-time dynamic pricing engine for urban parking lots. The system ingests historical/streaming occupancy telemetry, computes a multi-factor demand score per lot per timestamp, derives an optimised parking price, and applies geospatial competitive adjustment against neighbouring lots within a configurable radius. Output is streamed as interactive Bokeh time-series plots rendered directly inside Jupyter.

**Dataset:** 18,368 records · 14 parking lots · Birmingham, UK · Oct–Nov 2016  
**Price Bounds:** £5 (floor) – £20 (ceiling)  
**Competitive Radius:** 0.5 km (configurable)

---

## Tech Stack

| Layer | Library | Purpose |
|---|---|---|
| Core language | Python 3.8+ | All computation |
| Data wrangling | Pandas, NumPy | CSV ingestion, vectorised ops |
| Geospatial | Geopy (`geodesic`) | Haversine distance between lots |
| Visualisation | Bokeh 2.4+ | Interactive streaming plots in Jupyter |
| Runtime | Jupyter Notebook | Notebook-based demo & dev environment |
| Optional streaming | Pathway | Real-time event pipeline (install only) |

---

## Features

- **Multi-factor demand scoring** — occupancy ratio, queue length, traffic condition, special-day flag, and vehicle-type coefficient combined into a single scalar.
- **Baseline + demand-based dual pricing** — separates initial occupancy-driven price from demand-adjusted price, allowing independent tuning.
- **Geospatial competitive adjustment** — identifies all lots within `radius_km` using geodesic distance; nudges price ±£1 toward the neighbourhood average.
- **Per-lot streaming visualisation** — Bokeh `ColumnDataSource.stream()` updates live plots without re-rendering the full figure.
- **Configurable weight system** — all demand weights, vehicle multipliers, and sensitivity factors are top-level constants.
- **Price guardrails** — hard floor £5, hard ceiling £20 enforced at every computation stage.

---

## Architecture

```
dataset.csv
    │
    ▼
┌─────────────────────────┐
│  Preprocessing          │  Merge LastUpdatedDate + LastUpdatedTime → Timestamp
│                         │  Sort by (Timestamp, SystemCodeNumber)
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Demand Computation     │  compute_demand(row) → scalar
│                         │  Components: occupancy, queue, traffic, special_day, vehicle
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Pricing Engine         │  baseline_pricing()  →  raw price
│                         │  demand_based_price() → demand-adjusted price
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Competitive Adjustment │  get_nearby_lots(radius_km=0.5)
│                         │  competitive_adjustment(my_price, competitor_df)
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Bokeh Streaming Layer  │  ColumnDataSource.stream() per lot
│                         │  push_notebook() → live Jupyter update
└─────────────────────────┘
```

---

## Dataset Schema

| Column | Type | Description |
|---|---|---|
| `ID` | int | Row identifier |
| `SystemCodeNumber` | str | Unique lot ID (e.g. `BHMBCCMKT01`) |
| `Capacity` | int | Maximum parking spaces |
| `Latitude` | float | Lot GPS latitude |
| `Longitude` | float | Lot GPS longitude |
| `Occupancy` | int | Current occupied spaces |
| `VehicleType` | str | `car` / `bike` / `truck` / `cycle` |
| `TrafficConditionNearby` | str | `low` / `average` / `high` |
| `QueueLength` | int | Vehicles waiting to enter |
| `IsSpecialDay` | int | `1` = public holiday / event |
| `LastUpdatedDate` | str | `DD-MM-YYYY` |
| `LastUpdatedTime` | str | `HH:MM:SS` |

---

## Installation & Environment Setup

### Prerequisites

- Python 3.8+
- pip
- Jupyter Notebook or JupyterLab

### 1. Clone / copy project files

```bash
git clone <your-repo-url>
cd smart-parking-dynamic-pricing
```

### 2. Create and activate a virtual environment

```bash
python3 -m venv .venv
source .venv/bin/activate        # Linux / macOS
.venv\Scripts\activate           # Windows
```

### 3. Install dependencies

```bash
pip install pathway bokeh geopy --quiet
pip install numpy pandas jupyter
```

Or use the pinned requirements file if present:

```bash
pip install -r requirements.txt
```

### 4. Launch Jupyter

```bash
jupyter notebook final_hackathon.ipynb
```

> **Note:** Bokeh's `push_notebook()` requires the notebook to be served via `jupyter notebook` (classic). JupyterLab requires the `jupyter_bokeh` extension.

### Suggested `requirements.txt`

```
numpy>=1.21
pandas>=1.3
geopy>=2.3
bokeh>=2.4
pathway
jupyter
```

---

## Configuration Reference

All tunable parameters are defined as module-level constants at the top of the pricing section in `final_hackathon.ipynb`:

| Parameter | Default | Description |
|---|---|---|
| `default_price` | `10` | Baseline price (£) applied at cold-start |
| `alpha` | `5` | Occupancy sensitivity multiplier for baseline pricing |
| `sensitivity` | `0.5` | Demand-score sensitivity for demand-based pricing |
| `weight_occupancy` | `0.6` | Demand weight: occupancy ratio |
| `weight_queue` | `0.3` | Demand weight: queue length |
| `weight_traffic` | `0.4` | Demand weight: traffic condition (applied negatively) |
| `weight_special_day` | `1.0` | Demand weight: special day flag |
| `radius_km` | `0.5` | Competitive radius for nearby lot detection (km) |
| **Vehicle multipliers** | | |
| `car` | `1.0` | Baseline vehicle impact |
| `bike` | `0.5` | Reduced demand contribution |
| `truck` | `1.5` | Elevated demand contribution |
| **Traffic mapping** | | |
| `Low` | `0` | Low traffic → negative demand modifier |
| `Average` | `1` | Medium modifier |
| `High` | `2` | High modifier |

---

## Usage Guide

### Step 1 — Load and preprocess data

```python
import pandas as pd

df = pd.read_csv('dataset.csv')
df['Timestamp'] = pd.to_datetime(
    df['LastUpdatedDate'] + ' ' + df['LastUpdatedTime'],
    format='%d-%m-%Y %H:%M:%S'
)
df = df.sort_values(['Timestamp', 'SystemCodeNumber'])
```

### Step 2 — Compute demand score for a single row

```python
def compute_demand(lot_data):
    occupancy_ratio = lot_data['Occupancy'] / lot_data['Capacity']
    occupancy_component  = weight_occupancy * occupancy_ratio
    queue_component      = weight_queue * lot_data['QueueLength']
    traffic_val          = traffic_map.get(str(lot_data['TrafficConditionNearby']).capitalize(), 1)
    traffic_component    = -weight_traffic * traffic_val        # high traffic → lower demand
    special_day          = weight_special_day * int(lot_data['IsSpecialDay'])
    vehicle_component    = vehicle_impact.get(lot_data['VehicleType'].lower(), 1)

    return (occupancy_component + queue_component +
            traffic_component + special_day + vehicle_component)
```

### Step 3 — Derive price from demand

```python
def demand_based_price(demand_val):
    normalized = (demand_val - 1) / 9          # normalise to [0, 1] approx
    adjusted   = default_price * (1 + sensitivity * normalized)
    return max(5, min(20, round(adjusted, 2)))  # enforce guardrails
```

### Step 4 — Baseline price (occupancy-only fallback)

```python
def baseline_pricing(current_occupancy, max_capacity, previous_price=default_price):
    if max_capacity == 0:
        return default_price
    raw_price = previous_price + alpha * (current_occupancy / max_capacity)
    return max(5, min(20, round(raw_price, 2)))
```

### Step 5 — Competitive adjustment

```python
from geopy.distance import geodesic

def get_nearby_lots(all_lots_df, this_lot, radius_km=0.5):
    nearby = []
    for _, lot in all_lots_df.iterrows():
        if lot['SystemCodeNumber'] == this_lot['SystemCodeNumber']:
            continue
        dist = geodesic(
            (this_lot['Latitude'], this_lot['Longitude']),
            (lot['Latitude'], lot['Longitude'])
        ).km
        if dist <= radius_km:
            nearby.append(lot)
    return pd.DataFrame(nearby)

def competitive_adjustment(my_price, other_lot_prices):
    if len(other_lot_prices) == 0:
        return my_price
    avg = other_lot_prices['price'].mean()
    if my_price > avg:
        return max(5, my_price - 1)
    elif my_price < avg:
        return min(20, my_price + 1)
    return my_price
```

### Step 6 — Full streaming simulation loop

```python
price_memory = {}

for time_point in sorted(df['Timestamp'].unique()):
    snapshot = df[df['Timestamp'] == time_point].copy()

    for idx, row in snapshot.iterrows():
        lot_id    = row['SystemCodeNumber']
        demand    = compute_demand(row)
        price     = demand_based_price(demand)

        competitors = get_nearby_lots(snapshot, row)
        if not competitors.empty:
            competitors['price'] = competitors.apply(
                lambda r: demand_based_price(compute_demand(r)), axis=1
            )
            avg_comp_price = competitors['price'].mean()
            price = competitive_adjustment(price, competitors)
        else:
            avg_comp_price = price

        price_memory[lot_id] = price

        # Stream to Bokeh plot
        sources[lot_id].stream({
            'x': [time_point],
            'self_price': [price],
            'comp_price': [avg_comp_price]
        }, rollover=100)

    push_notebook(handle=handle)
```

---

## Function Reference

| Function | Signature | Returns | Description |
|---|---|---|---|
| `baseline_pricing` | `(occupancy, capacity, prev_price)` | `float` | Occupancy-driven price with alpha scaling |
| `compute_demand` | `(lot_data: Series)` | `float` | Weighted demand score across 5 factors |
| `demand_based_price` | `(demand_val: float)` | `float` | Normalised price in [5, 20] from demand score |
| `get_nearby_lots` | `(all_lots_df, this_lot, radius_km)` | `DataFrame` | Competitor lots within radius |
| `competitive_adjustment` | `(my_price, competitor_df)` | `float` | ±£1 nudge toward competitor average |

---

## Project Structure

```
smart-parking-dynamic-pricing/
├── dataset.csv              # 18,368-row parking telemetry (14 lots, Birmingham)
├── final_hackathon.ipynb    # Main notebook: all logic, config, and visualisations
└── README.md                # This file
```

---

## Extending the System

| Extension | Implementation hint |
|---|---|
| Weather integration | Add `weight_weather` constant; map weather codes → demand modifier |
| Historical trend | Compute rolling 7-day avg price per lot; use as `default_price` seed |
| Additional vehicle types | Add keys to `vehicle_impact` dict (`ev`, `van`, etc.) |
| Larger radius | Increase `radius_km`; consider caching geodesic calls for performance |
| Real-time pipeline | Route `dataset.csv` reads through Pathway's `pw.io.csv.read()` |
| REST API | Wrap `compute_demand` + `demand_based_price` in a FastAPI endpoint |

---

## Contributing

1. Fork the repository.
2. Create a feature branch: `git checkout -b feature/my-change`
3. Commit with a descriptive message: `git commit -m "feat: add weather factor"`
4. Open a pull request against `main`.

Please keep functions pure and add a docstring for any new pricing factor.

---

## License

MIT License — see [LICENSE](LICENSE) for full terms.
